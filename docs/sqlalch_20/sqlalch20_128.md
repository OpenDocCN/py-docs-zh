# 1.3 更新日志

> 原文：[`docs.sqlalchemy.org/en/20/changelog/changelog_13.html`](https://docs.sqlalchemy.org/en/20/changelog/changelog_13.html)

## 1.3.25

无发布日期

### orm

+   **[orm] [错误]**

    修复了在与主键列名与属性名不同的映射的主键跟踪失败的情况下，使用持久对象时 `Session.bulk_save_objects()` 中的问题。

    参考：[#6392](https://www.sqlalchemy.org/trac/ticket/6392)

### 模式

+   **[模式] [错误]**

    当未至少传递 `Table.name` 和 `Table.metadata` 参数位置时，`Table` 对象现在会引发一个信息性错误消息。 以前，如果这些作为关键字参数传递，对象将无声地无法正确初始化。

    参考：[#6135](https://www.sqlalchemy.org/trac/ticket/6135)

### postgresql

+   **[postgresql] [错误] [回归]**

    修复了由 [#6023](https://www.sqlalchemy.org/trac/ticket/6023) 引起的回归，当使用 psycopg2 时，应用于 `ARRAY` 中的元素的 PostgreSQL 强制转换运算符在数据类型也嵌入在 `Variant` 适配器的实例中时，将无法使用正确的类型。

    此外，修复了在使用 `Variant(ARRAY(some_schema_type))` 时发出正确的 CREATE TYPE 的支持。

    参考：[#6182](https://www.sqlalchemy.org/trac/ticket/6182)

### mysql

+   **[mysql] [错误] [mariadb]**

    修复以适应 MariaDB 10.6 系列的变化，包括 mariadb-connector Python 驱动程序（仅支持 SQLAlchemy 1.4）和 mysqlclient DBAPI 自动使用的本机 10.6 客户端库中的不兼容更改（适用于 1.3 和 1.4）。 当编码状态为“utf8”时，这些客户端库现在报告“utf8mb3”编码符号，导致 MySQL 方言中的查找和编码错误，该方言不期望此符号。 更新了 MySQL 基础库以适应报告此 utf8mb3 符号以及测试套件。 感谢 Georg Richter 的支持。

    参考：[#7115](https://www.sqlalchemy.org/trac/ticket/7115), [#7136](https://www.sqlalchemy.org/trac/ticket/7136)

### sqlite

+   **[sqlite] [错误]**

    添加有关通过 url 传递给 pysqlcipher 的与加密相关的 pragma 的说明。

    参考：[#6589](https://www.sqlalchemy.org/trac/ticket/6589)

## 1.3.24

发布日期：2021 年 3 月 30 日

### orm

+   **[orm] [错误]**

    移除了一个非常古老的警告，指出 passive_deletes 不适用于多对一关系。虽然在许多情况下，将此参数放在多对一关系上可能不是预期的操作，但在某些情况下，可能希望在此类关系之后禁止删除级联。

    参考：[#5983](https://www.sqlalchemy.org/trac/ticket/5983)

+   **[ORM] [错误]**

    修复了一个问题，即如果其中一个表具有不相关的、无法解析的外键约束，那么连接两个表的过程可能会失败，这将在连接过程中引发`NoReferenceError`，尽管可以绕过此异常以允许连接完成。在处理中测试异常重要性的逻辑会对构造做出可能失败的假设。

    参考：[#5952](https://www.sqlalchemy.org/trac/ticket/5952)

+   **[ORM] [错误]**

    修复了一个问题，即当父对象已加载并且随后被后续查询覆盖时，`MutableComposite`构造可能处于无效状态，因为复合属性的刷新处理程序会用新对象替换对象，而这些对象未受可变扩展处理。

    参考：[#6001](https://www.sqlalchemy.org/trac/ticket/6001)

### 引擎

+   **[引擎] [错误]**

    修复了“schema_translate_map”功能在直接执行`DefaultGenerator`对象（如序列）的情况未被考虑的错误，包括在禁用 implicit_returning 时“预执行”它们以生成主键值的情况。

    参考：[#5929](https://www.sqlalchemy.org/trac/ticket/5929)

### 模式

+   **[模式] [错误]**

    修复了首次引入的错误，可能是由[#2892](https://www.sqlalchemy.org/trac/ticket/2892)、[#2919](https://www.sqlalchemy.org/trac/ticket/2919)和[#3832](https://www.sqlalchemy.org/trac/ticket/3832)的某种组合引起的，其中`TypeDecorator`的附加事件会对“impl”类重复，如果“impl”也是`SchemaType`。实际情况是，任何`TypeDecorator`与`Enum`或`Boolean`相对时，当设置`create_constraint=True`标志时，将获得重复的`CheckConstraint`。

    参考：[#6152](https://www.sqlalchemy.org/trac/ticket/6152)

+   **[模式] [错误] [sqlite]**

    修复了由`Boolean`或`Enum`生成的 CHECK 约束在第一次编译后无法正确呈现命名约定的问题，这是由于约束名称中的状态意外更改导致的。此问题首次出现在 0.9 版本中，修复了问题＃3067，修复了当时采取的方法，似乎比实际需要的更复杂。

    参考：[#6007](https://www.sqlalchemy.org/trac/ticket/6007)

+   **[模式] [错误]**

    修复/实现了支持使用列名/键等作为约定的主键约束命名约定的功能。特别是，这包括`Table`自动关联的`PrimaryKeyConstraint`对象将在向表添加新的主键`Column`对象后更新其名称，然后更新约束。现在已经考虑了与此约束构造过程相关的内部故障模式，包括没有列存在、没有名称存在或存在空名称。

    参考：[#5919](https://www.sqlalchemy.org/trac/ticket/5919)

+   **[模式] [错误]**

    调整了在删除多个表时发出 DROP 语句的逻辑，使得所有 `Sequence` 对象在所有表之后被删除，即使给定的 `Sequence` 只与一个 `Table` 对象相关，而不直接与整体的 `MetaData` 对象相关。这种用例支持同一个 `Sequence` 同时关联多个 `Table`。

    参考：[#6071](https://www.sqlalchemy.org/trac/ticket/6071)

### postgresql

+   **[postgresql] [bug]**

    修复了在某些条件下使用 `aggregate_order_by` 会返回 ARRAY(NullType) 的问题，干扰了结果对象正确返回数据的能力。

    参考：[#5989](https://www.sqlalchemy.org/trac/ticket/5989)

+   **[postgresql] [bug] [reflection]**

    修复了在 PostgreSQL 反射中出现的问题，其中表达“NOT NULL”的列将取代相应域的可空性。

    参考：[#6161](https://www.sqlalchemy.org/trac/ticket/6161)

+   **[postgresql] [bug] [types]**

    调整了 psycopg2 方言以为包含 ARRAY 元素的绑定参数发出显式的 PostgreSQL 风格转换。这允许各种数据类型在数组中正确运行。asyncpg 方言已在最终语句中生成了这些内部转换。这还包括对数组切片更新的支持以及 PostgreSQL 特定的 `ARRAY.contains()` 方法。

    参考：[#6023](https://www.sqlalchemy.org/trac/ticket/6023)

### mssql

+   **[mssql] [bug] [reflection]**

    修复了关于旧版 SQL Server 2005 版本的 SQL Server 反射问题，调用 sp_columns 时如果没有加上 EXEC 关键字，将无法正确进行。这种方法在当前的 1.4 系列中不再使用。

    参考：[#5921](https://www.sqlalchemy.org/trac/ticket/5921)

## 1.3.23

发布日期：2021 年 2 月 1 日

### sql

+   **[sql] [bug]**

    修复了在`TypeDecorator`类型上使用`TypeEngine.with_variant()`方法时的错误，由于`TypeDecorator`中的规则尝试检查`TypeDecorator`实例链，未考虑到正在使用的特定方言映射。

    参考：[#5816](https://www.sqlalchemy.org/trac/ticket/5816)

### postgresql

+   **[postgresql] [bug]**

    仅适用于 SQLAlchemy 1.3，setup.py 将 pg8000 固定在低于 1.16.6 版本。版本 1.16.6 及以上受 SQLAlchemy 1.4 支持。感谢 Giuseppe Lumia 的拉取请求。

    参考：[#5645](https://www.sqlalchemy.org/trac/ticket/5645)

+   **[postgresql] [bug]**

    修复了在与使用了临时列表达式的 PostgreSQL `ExcludeConstraint`一起使用`Table.to_metadata()`（在 1.3 中称为`Table.tometadata()`）时未能正确复制的问题。

    参考：[#5850](https://www.sqlalchemy.org/trac/ticket/5850)

### mysql

+   **[mysql] [usecase]**

    现在在 MySQL >= (8, 0, 17)和 MariaDb >= (10, 4, 5)中支持转换为`FLOAT`。

    参考：[#5808](https://www.sqlalchemy.org/trac/ticket/5808)

+   **[mysql] [bug] [reflection]**

    修复了 MySQL 服务器默认反射对带有否定符号的数值失败的错误。

    参考：[#5860](https://www.sqlalchemy.org/trac/ticket/5860)

+   **[mysql] [bug]**

    修复了 MySQL 方言中长期存在的 bug，其中 255 的最大标识符长度对于所有类型的约束名称都太长，不仅仅是索引，所有这些都有 64 的大小限制。由于元数据命名约定可能在此区域创建过长的名称，因此将限制应用于 DDL 编译器内的标识符生成器。

    参考：[#5898](https://www.sqlalchemy.org/trac/ticket/5898)

+   **[mysql] [bug]**

    修复了由于 PyMySQL 1.0 发布而产生的弃用警告，包括“db”和“passwd”参数现已替换为“database”和“password”的弃用警告。

    参考：[#5821](https://www.sqlalchemy.org/trac/ticket/5821)

+   **[mysql] [bug]**

    修复了 SQLAlchemy 1.3.20 中由于[#5462](https://www.sqlalchemy.org/trac/ticket/5462)修复引起的回归问题，该修复为 MySQL 功能表达式在索引中添加了双括号，这是后端所需的，这无意中扩展到包括任意`text()`表达式以及 Alembic 的内部文本组件，这些对于 Alembic 来说是必需的，用于不暗示双括号的任意索引表达式。检查已经缩小，只包括直接的二元/一元/功能表达式。

    参考：[#5800](https://www.sqlalchemy.org/trac/ticket/5800)

### oracle

+   **[oracle] [bug]**

    修复了 SQLAlchemy 1.3.11 中由[#4894](https://www.sqlalchemy.org/trac/ticket/4894)引入的 Oracle 方言中的回归问题，其中在 UPDATE 的 RETURNING 中使用 SQL 表达式会导致编译失败，因为在任意 SQL 表达式不是列时会检查“server_default”。

    参考：[#5813](https://www.sqlalchemy.org/trac/ticket/5813)

+   **[oracle] [bug]**

    修复了 Oracle 方言中的一个 bug，即通过`Insert.returning()`检索 CLOB/BLOB 列时会失败，因为在返回时需要读取 LOB 值；此外，修复了在 Python 2 下通过 RETURNING 检索 Unicode 值的支持。

    参考：[#5812](https://www.sqlalchemy.org/trac/ticket/5812)

### 杂项

+   **[bug] [ext]**

    修复了一个问题，即在尝试为可选择的`.c`集合生成“key”时有时会调用字符串化，如果列是使用`sqlalchemy.ext.compiler`扩展创建的未标记的自定义 SQL 构造，并且没有提供默认编译形式，则会失败；虽然这似乎是一个不寻常的情况，但在一些 ORM 场景中可能会调用它，例如在与连接的急加载一起在“order by”中使用表达式时。问题在于缺乏默认编译器函数会引发`CompileError`而不是`UnsupportedCompilationError`。

    参考：[#5836](https://www.sqlalchemy.org/trac/ticket/5836)

## 1.3.22

发布日期：2020 年 12 月 18 日

### oracle

+   **[oracle] [bug]**

    修复了由于[#5755](https://www.sqlalchemy.org/trac/ticket/5755)引起的回归，该问题实现了 Oracle 的隔离级别支持。据报道，许多 Oracle 帐户实际上没有权限查询`v$transaction`视图，因此当在数据库连接时失败时，此功能已被更改为优雅地回退，其中方言将假定“READ COMMITTED”是默认的隔离级别，就像在 SQLAlchemy 1.3.21 之前的情况一样。然而，现在必须明确使用`Connection.get_isolation_level()`方法引发异常，因为具有此限制的 Oracle 数据库明确禁止用户读取当前隔离级别。

    参考：[#5784](https://www.sqlalchemy.org/trac/ticket/5784)

## 1.3.21

发布日期：2020 年 12 月 17 日

### orm

+   **[orm] [bug]**

    为传递给`relationship.secondary`的映射类或字符串映射类名称的情况添加了全面检查和信息丰富的错误消息。这是一个极为常见的错误，值得提供清晰的消息。

    此外，添加了一个新规则到类注册解析中，关于`relationship.secondary`参数，如果映射类及其表的字符串名称相同，则在解析此参数时将优先考虑`Table`。在所有其他情况下，如果类和表共享相同的名称，则仍然优先考虑类。

    参考：[#5774](https://www.sqlalchemy.org/trac/ticket/5774)

+   **[orm] [bug]**

    修复了在`Query.update()`中的错误，当`_ormsession.Session`中的对象已过期时，当它们被“evaluate”同步策略刷新时，会不必要地单独进行 SELECT 查询。

    参考：[#5664](https://www.sqlalchemy.org/trac/ticket/5664)

+   **[orm] [bug]**

    修复了涉及 ORM 事件的`restore_load_context`选项的错误，例如`InstanceEvents.load()`，使得标志不会传递到在首次建立事件处理程序之后映射的子类。

    参考：[#5737](https://www.sqlalchemy.org/trac/ticket/5737)

### sql

+   **[sql] [bug]**

    如果多次调用 `Insert.returning()` 或类似的 `returning()` 方法，则会发出警告，因为目前不支持累加操作。版本 1.4 将支持此操作。此外，`Insert.returning()` 和 `ValuesBase.return_defaults()` 方法的任何组合现在都会引发错误，因为这些方法是互斥的；之前的操作会静默失败。

    参考：[#5691](https://www.sqlalchemy.org/trac/ticket/5691)

+   **[sql] [bug]**

    修复了结构性编译器问题，某些构造（如 MySQL / PostgreSQL 的“on conflict / on duplicate key”）依赖于 `Compiler` 对象的状态被固定为其作为顶级语句的状态，这在这些语句从不同上下文中分支出来时会失败，例如与 SQL 语句链接的 DDL 构造。

    参考：[#5656](https://www.sqlalchemy.org/trac/ticket/5656)

### postgresql

+   **[postgresql] [usecase]**

    向 `ExcludeConstraint` 对象添加了新参数 `ExcludeConstraint.ops`，以支持此约束的运算符类规范。感谢 Alon Menczer 的拉取请求。

    参考：[#5604](https://www.sqlalchemy.org/trac/ticket/5604)

+   **[postgresql] [bug] [mysql]**

    修复了在 PostgreSQL 方言中引入的 1.3.2 版本的回归，也在 1.3.18 版本中复制到了 MySQL 方言的特性中，当使用非 `Table` 构造（如 `text()`）作为 `Select.with_for_update.of` 的参数时，在 PostgreSQL 或 MySQL 编译器中未能正确处理。 

    参考：[#5729](https://www.sqlalchemy.org/trac/ticket/5729)

### mysql

+   **[mysql] [bug] [reflection]**

    修复了仅包含值中有小数点的 MariaDB 上的服务器默认反射问题，导致反射表缺乏任何服务器默认值。

    参考：[#5744](https://www.sqlalchemy.org/trac/ticket/5744)

+   **[mysql] [sql]**

    向 MySQL 方言的 `RESERVED_WORDS` 列表中添加了缺失的关键字：`action`、`level`、`mode`、`status`、`text`、`time`。感谢 Oscar Batori 的拉取请求。

    参考：[#5696](https://www.sqlalchemy.org/trac/ticket/5696)

### sqlite

+   **[sqlite] [usecase]**

    添加了`sqlite_with_rowid=False`方言关键字，以便创建`CREATE TABLE … WITHOUT ROWID`表。感谢 Sean Anderson 的补丁。

    参考：[#5685](https://www.sqlalchemy.org/trac/ticket/5685)

### mssql

+   **[mssql] [bug]**

    修复了当同时指定`mssql-include`和`mssql_where`时，CREATE INDEX 语句渲染不正确的错误。感谢@Adiorz 的拉取请求。

    参考：[#5751](https://www.sqlalchemy.org/trac/ticket/5751)

+   **[mssql] [bug]**

    将 SQL Server 代码“01000”添加到断开连接代码列表中。

    参考：[#5646](https://www.sqlalchemy.org/trac/ticket/5646)

+   **[mssql] [reflection] [sqlite]**

    修复了复合主键列未按正确顺序报告的问题。感谢@fulpm。

    参考：[#5661](https://www.sqlalchemy.org/trac/ticket/5661)

### oracle

+   **[oracle] [usecase]**

    实现了对 Oracle 数据库的 SERIALIZABLE 隔离级别的支持，以及对`Connection.get_isolation_level()`的真正实现。

    另请参见

    事务隔离级别 / 自动提交

    参考：[#5755](https://www.sqlalchemy.org/trac/ticket/5755)

## 1.3.20

发布日期：2020 年 10 月 12 日

### orm

+   **[orm] [bug]**

    如果`Query.join()`的目标参数设置为未映射对象，则现在会引发带有更多详细信息的`ArgumentError`。在此更改之前，会引发一个较少详细的`AttributeError`。感谢 Ramon Williams 的拉取请求。

    参考：[#4428](https://www.sqlalchemy.org/trac/ticket/4428)

+   **[orm] [bug]**

    修复了针对实际上不是映射属性的字符串属性名称（例如普通 Python 描述符）使用加载器选项会引发一个不具信息性的 AttributeError 的问题；现在会引发一个描述性错误。

    参考：[#4589](https://www.sqlalchemy.org/trac/ticket/4589)

### engine

+   **[engine] [bug]**

    修复了将非字符串对象发送给`SQLAlchemyError`或其子类（在某些第三方方言中会发生）时无法正确字符串化的问题。感谢 Andrzej Bartosiński 的拉取请求。

    参考：[#5599](https://www.sqlalchemy.org/trac/ticket/5599)

+   **[engine] [bug]**

    修复了未在 sqlalchemy.exc 模块内使用 SQLAlchemy 标准延迟导入系统的函数级别导入。

    参考：[#5632](https://www.sqlalchemy.org/trac/ticket/5632)

### sql

+   **[sql] [bug]**

    修复了针对`Over`构造执行`pickle.dumps()`操作会产生递归溢出的问题。

    参考：[#5644](https://www.sqlalchemy.org/trac/ticket/5644)

+   **[sql] [bug]**

    修复了一个 bug，在这种情况下，当一个 `column()` 被同时添加到多个 `table()` 时，错误没有被引发。这对 `Column` 和 `Table` 对象正确引发。当发生这种情况时，现在会引发一个 `ArgumentError`。

    参考：[#5618](https://www.sqlalchemy.org/trac/ticket/5618)

### postgresql

+   **[postgresql] [usecase]**

    psycopg2 方言现在支持将主机/端口组合传递给查询字符串，以支持 PostgreSQL 多主机连接。感谢 Ramon Williams 的拉取请求。

    另请参阅

    指定多个备用主机

    参考：[#4392](https://www.sqlalchemy.org/trac/ticket/4392)

+   **[postgresql] [bug]**

    调整了 `Comparator.any()` 和 `Comparator.all()` 方法，实现了直接的“NOT”操作来进行否定，而不是否定比较运算符。

    参考：[#5518](https://www.sqlalchemy.org/trac/ticket/5518)

+   **[postgresql] [bug]**

    修复了 `ENUM` 类型在进行 CREATE TYPE 或 DROP TYPE 时不会查阅模式转换映射的问题。此外，修复了一个问题，即如果在单个 DDL 序列中多次遇到相同的枚举，则“check”查询将重复运行，而不是依赖于缓存值。

    参考：[#5520](https://www.sqlalchemy.org/trac/ticket/5520)

### mysql

+   **[mysql] [usecase]**

    调整了 MySQL 方言，以正确地将函数索引表达式括在括号中，这是 MySQL 8 所接受的。感谢 Ramon Williams 的拉取请求。

    参考：[#5462](https://www.sqlalchemy.org/trac/ticket/5462)

+   **[mysql] [change]**

    添加了新的 MySQL 保留字：`cube`，`lateral` 分别在 MySQL 8.0.1 和 8.0.14 中添加；这表明如果作为表或列标识符名称使用，这些术语将被引用。

    参考：[#5539](https://www.sqlalchemy.org/trac/ticket/5539)

+   **[mysql] [bug]**

    使用 `with_for_update()` 时使用的 “skip_locked” 关键字在 MariaDB 后端上使用时会发出警告，然后将被忽略。这是一个已弃用的行为，在 SQLAlchemy 1.4 中将引发异常，因为请求“skip locked”的应用程序正在寻找一个在这些后端上不可用的非阻塞操作。

    参考文献：[#5568](https://www.sqlalchemy.org/trac/ticket/5568)

+   **[mysql] [bug]**

    修复了针对使用 MySQL 多表格式的 JOIN 的 UPDATE 语句的错误，如果语句没有 WHERE 子句，则会失败地包括目标表的表前缀，因为在这一特定点只有 WHERE 子句被扫描以检测“多表更新”。现在，如果是 JOIN，则还会扫描目标，以获取最左边的表作为主表，以及其他条目作为其他 FROM 条目。

    参考文献：[#5617](https://www.sqlalchemy.org/trac/ticket/5617)

### mssql

+   **[mssql] [bug]**

    修复了一个问题，其中对于 Azure DW 的 SQLAlchemy 连接 URI 使用 `authentication=ActiveDirectoryIntegrated`（而没有用户名+密码）时，构造的 ODBC 连接字符串的方式不被 Azure DW 实例接受。

    参考文献：[#5592](https://www.sqlalchemy.org/trac/ticket/5592)

### 测试

+   **[tests] [bug]**

    修复了与 Pytest 6.x 运行时测试套件不兼容的问题。

    参考文献：[#5635](https://www.sqlalchemy.org/trac/ticket/5635)

### 杂项

+   **[bug] [pool]**

    修复了当调用 `Engine.dispose()` 时，以下池参数未被传播到新创建的池的问题：`pre_ping`，`use_lifo`。此外，现在还将 `recycle` 和 `reset_on_return` 参数传播给了 `AssertionPool` 类。

    参考文献：[#5582](https://www.sqlalchemy.org/trac/ticket/5582)

+   **[bug] [associationproxy] [ext]**

    在尝试将关联代理元素用作要从中选择或在 SQL 函数中使用的普通列表达式时，现在会引发一个信息性错误；目前不支持此用例。

    参考文献：[#5541](https://www.sqlalchemy.org/trac/ticket/5541), [#5542](https://www.sqlalchemy.org/trac/ticket/5542)

## 1.3.19

发布日期：2020 年 8 月 17 日

### orm

+   **[orm] [usecase]**

    调整了 `Mapper.all_orm_descriptors()` 访问器的工作方式，以一种确定性的方式表示属性，假定使用 Python 3.6 或更高版本，该版本根据声明属性的方式维护类属性的排序顺序。然而，这种排序不能保证在所有情况下都与属性的声明顺序匹配；请参阅方法文档以获取确切的方案。

    参考文献：[#5494](https://www.sqlalchemy.org/trac/ticket/5494)

### orm 声明

+   **[orm] [declarative] [usecase]**

    当使用`AbstractConcreteBase`和`ConcreteBase`类时，虚拟列的名称现在可以自定义，以允许具有实际命名为`type`的列的模型。拉请求由 Jesse-Bakker 提供。

    参考：[#5513](https://www.sqlalchemy.org/trac/ticket/5513)

### sql

+   **[sql] [bug]**

    修复了“ORDER BY”子句呈现标签名称而不是完整表达式的问题，这对于 SQL Server 特别重要，在某些情况下，如果表达式被括在括号中，则不会发生。此案例已添加到测试支持。此更改还调整了 ORM 查询中“在存在 DISTINCT 时自动添加 ORDER BY 列”的行为，在 1.4 中已弃用，以更准确地检测已经存在的列表达式。

    参考：[#5470](https://www.sqlalchemy.org/trac/ticket/5470)

+   **[sql] [bug] [datatypes]**

    `LookupError` 消息现在将向用户提供通过`Enum`约束的列可能的四个可能值。超过 11 个字符的值将被截断并替换为省略号。拉请求由 Ramon Williams 提供。

    参考：[#4733](https://www.sqlalchemy.org/trac/ticket/4733)

+   **[sql] [bug]**

    修复了`Connection.execution_options.schema_translate_map`功能在使用`Column.server_default`参数中使用`Sequence.next_value()`函数时不起作用，并且创建表 DDL 被发出时。

    参考：[#5500](https://www.sqlalchemy.org/trac/ticket/5500)

### postgresql

+   **[postgresql] [bug]**

    修复了各种 RANGE 比较运算符的返回类型本身将是相同的 RANGE 类型而不是 BOOLEAN 的问题，这将导致在使用定义了结果处理行为的`TypeDecorator`时产生不良结果。拉请求由 Jim Bosch 提供。

    参考：[#5476](https://www.sqlalchemy.org/trac/ticket/5476)

### mysql

+   **[mysql] [usecase]**

    MySQL 方言将为没有 FROM 子句但有 WHERE 子句的 SELECT 语句渲染 FROM DUAL。 这允许像“SELECT 1 WHERE EXISTS (subquery)”这样的查询以及其他用例。

    参考资料：[#5481](https://www.sqlalchemy.org/trac/ticket/5481)

+   **[mysql] [错误]**

    修复了 CREATE TABLE 语句未正确指定 COLLATE 关键字的问题。

    参考资料：[#5411](https://www.sqlalchemy.org/trac/ticket/5411)

+   **[mysql] [错误]**

    将 MariaDB 代码 1927 添加到“断开连接”代码列表中，因为最近的 MariaDB 版本显然在停止数据库服务器时使用此代码。

    参考资料：[#5493](https://www.sqlalchemy.org/trac/ticket/5493)

### sqlite

+   **[sqlite] [错误] [mssql] [反射]**

    对所有包含的方言进行了扫描，以确保在查询系统表时正确转义包含单引号或双引号的名称，对所有`Inspector`方法接受对象名称作为参数（例如表名、视图名等）的方法进行修复（例如表名、视图名等）。 SQLite 和 MSSQL 存在两个修复的引号问题。

    参考资料：[#5456](https://www.sqlalchemy.org/trac/ticket/5456)

### mssql

+   **[mssql] [错误] [sql]**

    修复了 mssql 方言错误地转义包含‘]’字符的对象名称的错误。

    参考资料：[#5467](https://www.sqlalchemy.org/trac/ticket/5467)

### 杂项

+   **[用例] [py3k]**

    在`DeclarativeMeta.__init__()`方法中添加了一个`**kw`参数。这允许类支持[**PEP 487**](https://peps.python.org/pep-0487/)元类钩子`__init_subclass__`。合并请求由 Ewen Gillies 提供。

    参考资料：[##5357](https://www.sqlalchemy.org/trac/ticket/#5357)

## 1.3.18

发布日期：2020 年 6 月 25 日

### orm

+   **[orm] [用例]**

    当在查询中使用`Query.filter_by()`时，若第一个实体不是映射类，则改进错误消息。

    参考资料：[#5326](https://www.sqlalchemy.org/trac/ticket/5326)

+   **[orm] [用例]**

    在`query_expression()`构造中添加了一个新参数`query_expression.default_expr`，如果未使用`with_expression()`选项，则该参数将自动应用于查询。合并请求由 Haoyu Sun 提供。

    参考资料：[#5198](https://www.sqlalchemy.org/trac/ticket/5198)

### 示例

+   **[示例] [变更]**

    为 examples.performance 套件添加了新选项`--raw`，该选项将原始性能测试转储给任意数量的性能分析可视化工具使用。由于在这一点上很难构建 runsnake，因此删除了“runsnake”选项；

### 引擎

+   **[引擎] [错误]**

    进一步细化了在 [#5326](https://www.sqlalchemy.org/trac/ticket/5326) 中修复的“reset”代理的修复，现在在未正确调用时发出警告并纠正其行为。已识别并修复了额外的情况，其中会发出此警告。

    参考：[#5326](https://www.sqlalchemy.org/trac/ticket/5326)

+   **[engine] [bug]**

    修复了 `URL` 对象中的问题，其中对对象进行字符串化不会对特殊字符进行 URL 编码，从而阻止 URL 重新被消费为真实 URL。感谢 Miguel Grinberg 提交的拉取请求。

    参考：[#5341](https://www.sqlalchemy.org/trac/ticket/5341)

### sql

+   **[sql] [usecase]**

    `table()` 构造中添加了一个“.schema”参数，允许临时表达式也包含模式名称。感谢 Dylan Modesitt 提交的拉取请求。

    参考：[#5309](https://www.sqlalchemy.org/trac/ticket/5309)

+   **[sql] [change] [sybase]**

    为 sybase 方言添加了 `.offset` 支持。感谢 Alan D. Snow 提交的拉取请求。

    参考：[#5294](https://www.sqlalchemy.org/trac/ticket/5294)

+   **[sql] [bug]**

    在 `type_coerce` 元素中正确应用 `self_group`。

    在表达式中使用时，类型强制元素未正确应用分组规则。

    参考：[#5344](https://www.sqlalchemy.org/trac/ticket/5344)

+   **[sql] [bug]**

    在调用语句的 `str()` 时，将 `Select.with_hint()` 输出到生成的通用 SQL 字符串中。先前，此子句会被省略，假设其是方言特定的。提示文本在括号内呈现，以指示此类提示在后端之间的呈现方式有所不同。

    参考：[#5353](https://www.sqlalchemy.org/trac/ticket/5353)

+   **[sql] [schema]**

    引入 `IdentityOptions` 来存储序列和身份列的常见参数。

    参考：[#5324](https://www.sqlalchemy.org/trac/ticket/5324)

### schema

+   **[schema] [bug]**

    修复了使用 `tometadata()` 复制数据库对象（例如，`Table`）时省略了 `dialect_options` 的问题。

    参考：[#5276](https://www.sqlalchemy.org/trac/ticket/5276)

### mysql

+   **[mysql] [usecase]**

    为 mysql 实现了行级锁支持。感谢 Quentin Somerville 提交的拉取请求。

    参考：[#4860](https://www.sqlalchemy.org/trac/ticket/4860)

### sqlite

+   **[sqlite] [usecase]**

    SQLite 3.31 添加了对计算列的支持。此更改在针对 SQLite 时启用了对其的支持。

    参考：[#5297](https://www.sqlalchemy.org/trac/ticket/5297)

+   **[sqlite] [bug]**

    将“exists”添加到 SQLite 的保留字列表中，以便在用作标签或列名时将此单词引用。感谢 Thodoris Sotiropoulos 的拉取请求。

    参考：[#5395](https://www.sqlalchemy.org/trac/ticket/5395)

### mssql

+   **[mssql] [change]**

    将`supports_sane_rowcount_returning = False`的要求从`PyODBCConnector`级别移至`MSDialect_pyodbc`，因为在某些情况下 pyodbc 可以正常工作。

    参考：[#5321](https://www.sqlalchemy.org/trac/ticket/5321)

+   **[mssql] [bug]**

    优化了 SQL Server 方言用于解释包含许多点的多部分模式名称的逻辑，如果名称没有使用括号或引号，则实际上不会丢失任何点，并且还支持包含多个独立括号部分的“dbname”标记。

    参考：[#5364](https://www.sqlalchemy.org/trac/ticket/5364)，[#5366](https://www.sqlalchemy.org/trac/ticket/5366)

+   **[mssql] [bug] [pyodbc]**

    修复了 pyodbc 连接器中的问题，当使用完全空的 URL 时，会发出关于 pyodbc“drivername”的警告。在生成非连接的方言对象或使用“creator”参数创建引擎时，空 URL 是正常的。现在只有在缺少驱动程序名称但其他参数仍然存在时才会发出警告。

    参考：[#5346](https://www.sqlalchemy.org/trac/ticket/5346)

+   **[mssql] [bug]**

    修复了在为 pyodbc DBAPI 组装 ODBC 连接字符串时的问题。包含分号和/或大括号“{}”的标记未被正确转义，导致 ODBC 驱动程序错误解释连接字符串属性。

    参考：[#5373](https://www.sqlalchemy.org/trac/ticket/5373)

+   **[mssql] [bug]**

    修复了一个问题，其中`datetime.time`参数被转换为`datetime.datetime`，导致与实际`TIME`列进行`>=`比较时不兼容。

    参考：[#5339](https://www.sqlalchemy.org/trac/ticket/5339)

+   **[mssql] [bug]**

    修复了 SQL Server pyodbc 方言中的`is_disconnect`函数在异常消息中包含与 SQL Server ODBC 错误代码匹配的子字符串时错误报告断开状态的问题。

    参考：[#5359](https://www.sqlalchemy.org/trac/ticket/5359)

### oracle

+   **[oracle] [bug] [reflection]**

    修复了 Oracle 方言中的一个问题，其中包含完整主键列集的索引会被误认为是主键索引本身，即使存在多个也会被省略。检查已经被细化，以将主键约束的名称与索引名称本身进行比较，而不是根据索引中存在的列来猜测。

    参考：[#5421](https://www.sqlalchemy.org/trac/ticket/5421)

## 1.3.17

发布日期：2020 年 5 月 13 日

### orm

+   **[orm] [usecase]**

    添加了一个访问器`Comparator.expressions`，它提供对在多列`ColumnProperty`属性下映射的列组的访问。

    参考：[#5262](https://www.sqlalchemy.org/trac/ticket/5262)

+   **[orm] [usecase]**

    引入了`relationship.sync_backref`标志，用于控制是否添加会改变 Python 属性的同步事件。这取代了先前的更改[#5149](https://www.sqlalchemy.org/trac/ticket/5149)，该更改警告说`viewonly=True`关系的 back_populates 或 backref 配置的目标将被禁止。

    参考：[#5237](https://www.sqlalchemy.org/trac/ticket/5237)

+   **[orm] [bug]**

    修复了一个 bug，当在一个已经具有与请求的相同的基于子查询的 with_polymorphic 设置的映射器上通过`RelationshipComparator.of_type()`使用`with_polymorphic()`作为连接的目标时，ON 子句在连接中不会正确别名化。

    参考：[#5288](https://www.sqlalchemy.org/trac/ticket/5288)

+   **[orm] [bug]**

    修复了加载器选项（如 selectinload()）与烘焙查询系统交互的问题，使得如果加载器选项本身具有当前不兼容缓存的元素（如 with_polymorphic()对象），则不应该发生查询的缓存。在这些情况下，烘焙加载器有时无法完全使自身失效，导致急切加载被忽略。

    参考：[#5303](https://www.sqlalchemy.org/trac/ticket/5303)

+   **[orm] [bug]**

    修改了内部的“identity set”实现，这是一个根据它们的 id()而不是它们的哈希值对对象进行哈希的集合，以便实际上不调用对象的`__hash__()`方法，这些对象通常是用户映射的对象。一些方法在实现的副作用中调用了这个方法。

    参考：[#5304](https://www.sqlalchemy.org/trac/ticket/5304)

+   **[orm] [bug]**

    当尝试对一个不是实际映射实例的 ORM 多对一比较时，会引发一个信息性错误消息。不支持诸如与标量子查询的比较；使用`Comparator.has()`更好地实现了与子查询的广义比较。

    参考：[#5269](https://www.sqlalchemy.org/trac/ticket/5269)

### engine

+   **[engine] [bug]**

    修复了一个相当关键的问题，即在仍处于未回滚状态时，DBAPI 连接可能会被返回到连接池。负责回滚连接的重置代理可能会在事务“关闭”而未回滚或提交的情况下被损坏，在使用 ORM 会话并以涉及保存点的某种模式发出.close()时可能会发生这种情况。修复确保重置代理始终处于活动状态。

    参考：[#5326](https://www.sqlalchemy.org/trac/ticket/5326)

### schema

+   **[schema] [bug]**

    修复了一个问题，即一个被延迟与表关联的`Index`，例如当它包含一个尚未与任何`Table`关联的`Column`时，如果还包含非表导向表达式，则无法正确附加。 

    参考：[#5298](https://www.sqlalchemy.org/trac/ticket/5298)

+   **[schema] [bug]**

    当使用`MetaData.sorted_tables`属性以及`sort_tables()`函数时，如果给定的表由于外键约束之间存在循环依赖而无法正确排序时，会发出警告。在这种情况下，这些函数将不再按照外键对涉及的表进行排序，并将发出警告。不属于循环的其他表仍将按依赖顺序返回。以前，当检测到循环时，排序表例程会返回一个集合，该集合在检测到循环时会无条件省略所有外键，并且不会发出警告。

    参考：[#5316](https://www.sqlalchemy.org/trac/ticket/5316)

+   **[schema]**

    在`Column`的`__repr__`方法中添加`comment`属性。

    参考：[#4138](https://www.sqlalchemy.org/trac/ticket/4138)

### postgresql

+   **[postgresql] [usecase]**

    在 PostgreSQL 中添加了对`Enum`的列或类型`ARRAY`、`JSON`或`JSONB`的支持。在这些用例中以前需要使用一种解决方法。

    参考：[#5265](https://www.sqlalchemy.org/trac/ticket/5265)

+   **[postgresql] [usecase]**

    当添加一个配置了`Enum.native_enum`为`False`的`Enum`类型的`ARRAY`列的表时，如果`Enum.create_constraint`未设置为`False`，则引发明确的`CompileError`

    参考：[#5266](https://www.sqlalchemy.org/trac/ticket/5266)

### mssql

+   **[mssql] [错误] [反射]**

    修复了在使用旧版 TDS 版本 4.2 时，在 MSSQL 中反射计算列引入的回归。方言将尝试检测首次连接的协议版本，并在无法检测到时运行在兼容模式下。

    参考：[#5255](https://www.sqlalchemy.org/trac/ticket/5255)

+   **[mssql] [错误] [反射]**

    修复了在使用不支持 `concat` 函数的 SQL Server 版本 2012 之前的版本时，在 MSSQL 中反射计算列引入的回归。

    参考：[#5271](https://www.sqlalchemy.org/trac/ticket/5271)

### oracle

+   **[oracle] [性能] [错误]**

    更改了获取 CLOB 和 BLOB 对象的实现方式，使用了 cx_Oracle 的本机实现，该实现将 CLOB/BLOB 对象与其他结果列一起获取，而不是执行单独的获取操作。如往常一样，可以通过将 auto_convert_lobs 设置为 False 来禁用此功能。

    作为此更改的一部分，给定空字符串的 CLOB 现在在 INSERT 时返回 None，在 SELECT 时也返回 None，这与 Oracle 中 VARCHAR 的行为现在保持一致。

    参考：[#5314](https://www.sqlalchemy.org/trac/ticket/5314)

+   **[oracle] [错误]**

    对于 LOB 和数字数据类型，对 cx_oracle 方言如何设置每列输出类型处理程序进行了一些修改，以适应即将到来的 cx_Oracle 8 中的潜在更改。

    参考：[#5246](https://www.sqlalchemy.org/trac/ticket/5246)

### 杂项

+   **[变更] [firebird]**

    调整了对 `firebird://` URI 的方言加载，如果已安装外部的 sqlalchemy-firebird 方言，则使用该方言，否则回退到（现在已弃用的）内部 Firebird 方言。

    参考：[#5278](https://www.sqlalchemy.org/trac/ticket/5278)

## 1.3.16

发布日期：2020 年 4 月 7 日

### orm

+   **[orm] [性能]**

    修改了 subqueryload 和 selectinload 使用的查询，不再按照父实体的主键排序；这种排序是为了允许行按照它们进入的顺序直接复制到列表中，以便在 Python 端进行最小级别的排序。然而，这些 ORDER BY 子句可能会对查询的性能产生负面影响，因为在许多情况下，这些列是从子查询派生的，或者以其他方式不是实际的主键列，以至于 SQL 规划器无法使用索引。Python 端排序使用本机 itertools.group_by() 来整理传入的行，并已经修改为允许使用 list.extend() 将多个行组合到一起，这仍然可以实现相对快速的 Python 端性能。对于包含显式 order_by 参数的关系，仍将存在一个 ORDER BY，但这是两种加载方式的查询中唯一添加的 ORDER BY。

    参考：[#5162](https://www.sqlalchemy.org/trac/ticket/5162)

+   **[orm] [bug]**

    修复了`selectinload()`加载选项中的一个 bug，当两个或更多的加载器代表不同关系，并且具有相同的字符串键名称，从一个包含多个子类映射器的`with_polymorphic()`构造中引用时，将无法单独调用每个 subqueryload，而是使用单个基于字符串的插槽，这将阻止其他加载器被调用。

    参考：[#5228](https://www.sqlalchemy.org/trac/ticket/5228)

+   **[orm] [bug]**

    修复了一个问题，即 lazyload 使用会话本地的“get”针对目标多对一关系，当具有正确主键的对象存在时，但它是同级类的实例时，不会正确返回 None，就像懒加载器实际为该行发出加载时的情况一样。

    参考：[#5210](https://www.sqlalchemy.org/trac/ticket/5210)

### orm 声明式

+   **[orm] [declarative] [bug]**

    当使用声明式 API 时，`relationship()` 函数接受的第一个位置参数作为字符串参数不再使用 Python 的 `eval()` 函数进行解释；相反，名称是点分隔的，并且名称直接在名称解析字典中查找，而不将值视为 Python 表达式。然而，将字符串参数传递给其他必须接受 Python 表达式的 `relationship()` 参数仍将使用 `eval()`；文档已经澄清以确保没有歧义。

    另请参阅

    评估关系参数 - 字符串评估的详细信息

    参考：[#5238](https://www.sqlalchemy.org/trac/ticket/5238)

### sql

+   **[sql] [类型]**

    添加了在使用字符串方言进行调试时，能够对`DateTime`、`Date`或`Time`进行文字编译的能力。此更改不会影响保留其当前行为的真实方言实现。

    参考：[#5052](https://www.sqlalchemy.org/trac/ticket/5052)

### 模式

+   **[模式] [反射]**

    添加了对“计算”列的反射支持，这些列现在作为`Inspector.get_columns()`返回的结构的一部分返回。在反映完整的`Table`对象时，计算列将使用`Computed`构造表示。

    参考：[#5063](https://www.sqlalchemy.org/trac/ticket/5063)

### postgresql

+   **[postgresql] [错误]**

    修复了“覆盖”索引的问题，例如具有 INCLUDE 子句的索引，将被反映为包含所有 INCLUDE 子句中的列的常规列。如果检测到这些额外列，现在会发出警告，指示它们当前被忽略。请注意，“覆盖”索引的全面支持是[#4458](https://www.sqlalchemy.org/trac/ticket/4458)的一部分。拉取请求由 Marat Sharafutdinov 提供。

    参考：[#5205](https://www.sqlalchemy.org/trac/ticket/5205)

### mysql

+   **[mysql] [错误]**

    修复了在连接到伪 MySQL 数据库（例如由 ProxySQL 提供的数据库）时 MySQL 方言中的问题，当它返回没有行时，对隔离级别的事先检查不会阻止方言继续连接。会发出警告，指出无法检测到隔离级别。

    参考：[#5239](https://www.sqlalchemy.org/trac/ticket/5239)

### sqlite

+   **[sqlite] [用例]**

    当使用 pysqlite 时，为 SQLite 实现了 AUTOCOMMIT 隔离级别。

    参考：[#5164](https://www.sqlalchemy.org/trac/ticket/5164)

### mssql

+   **[mssql] [用例] [mysql] [oracle]**

    为 SQL Server、MySQL 和 Oracle 添加了对`ColumnOperators.is_distinct_from()`和`ColumnOperators.isnot_distinct_from()`的支持。

    参考：[#5137](https://www.sqlalchemy.org/trac/ticket/5137)

### oracle

+   **[oracle] [用例]**

    当使用 cx_Oracle 时，为 Oracle 实现了 AUTOCOMMIT 隔离级别。还为 Oracle 添加了一个固定的默认隔离级别为 READ COMMITTED。

    参考：[#5200](https://www.sqlalchemy.org/trac/ticket/5200)

+   **[oracle] [错误] [反射]**

    修复了由于[#5146](https://www.sqlalchemy.org/trac/ticket/5146)的修复导致的回归/不正确的修复，其中 Oracle 方言从“all_tab_comments”视图读取表注释，但未能适应请求的表的当前所有者，导致如果多个具有相同名称的表存在于多个模式中，则读取错误的注释。

    参考：[#5146](https://www.sqlalchemy.org/trac/ticket/5146)

### 测试

+   **[测试] [错误]**

    修复了阻止测试套件在最近发布的 py.test 5.4.0 上运行的问题。

    参考：[#5201](https://www.sqlalchemy.org/trac/ticket/5201)

### 杂项

+   **[枚举] [类型]**

    `Enum`类型现在支持参数`Enum.length`来指定创建 VARCHAR 列时的长度，当使用非本机枚举时，通过将`Enum.native_enum`设置为`False`。

    参考：[#5183](https://www.sqlalchemy.org/trac/ticket/5183)

+   **[安装程序]**

    确保“pyproject.toml”文件不包含在构建中，因为此文件的存在表示 pip 应使用 pep-517 安装过程。由于当前工具/发行版似乎不太支持这种操作模式，在 SQLAlchemy 安装范围内通过省略文件来避免这些问题。

    参考：[#5207](https://www.sqlalchemy.org/trac/ticket/5207)

## 1.3.15

发布日期：2020 年 3 月 11 日

### orm

+   **[orm] [错误]**

    当`Query.join()`无法定位左侧时，调整了错误消息，指出`Query.select_from()`方法是解决问题的最佳方式。此外，在 1.3 系列中，从传递给`Query`的给定列实体确定 FROM 子句时，使用确定性排序，以便每次确定相同的表达式。

    参考：[#5194](https://www.sqlalchemy.org/trac/ticket/5194)

+   **[orm] [错误]**

    由于[#4849](https://www.sqlalchemy.org/trac/ticket/4849)导致的 1.3.14 中的回归错误已修复，当发生刷新错误时，sys.exc_info()调用未能正确调用。已为此异常情况添加了测试覆盖。

    参考：[#5196](https://www.sqlalchemy.org/trac/ticket/5196)

## 1.3.14

发布日期：2020 年 3 月 10 日

### 通用

+   **[通用] [错误] [py3k]**

    对于大多数或全部在内部异常捕获中引发的内部异常，都明确添加了一个“cause”，以避免误导性的堆栈跟踪，这些堆栈跟踪表明在处理异常时出现了错误。虽然最好抑制内部捕获的异常，就像`__suppress_context__`属性所做的那样，但目前似乎还没有一种方法可以在不抑制包含的用户构造上下文的情况下做到这一点，因此目前它将内部捕获的异常暴露为原因，以便维护有关错误上下文的全部信息。

    参考文献：[#4849](https://www.sqlalchemy.org/trac/ticket/4849)

### orm

+   **[orm] [usecase]**

    添加了一个新标志`InstanceEvents.restore_load_context`和`SessionEvents.restore_load_context`，适用于`InstanceEvents.load()`、`InstanceEvents.refresh()`和`SessionEvents.loaded_as_persistent()`事件，当设置时，将在调用事件挂钩后恢复对象的“加载上下文”。这确保对象保持在已经进行的加载操作的“加载器上下文”中，而不是由于在事件中可能发生的刷新操作而将对象转移到新的加载上下文。当出现这种情况时，现在会发出警告，建议使用标志来解决这种情况。该标志是“选择加入”的，因此不会引入对现有应用程序的风险。

    此更改还为会话生命周期事件添加了对`raw=True`标志的支持。

    参考文献：[#5129](https://www.sqlalchemy.org/trac/ticket/5129)

+   **[orm] [bug]**

    1.3.13 中由[#5056](https://www.sqlalchemy.org/trac/ticket/5056)引起的固定回归，其中 ORM 路径注册表系统的重构使路径不再可以与空元组进行比较，这可能发生在一种特定类型的连接式及加载路径中。已解决“空元组”用例，以便在所有情况下将路径注册表与路径注册表进行比较；`PathRegistry`对象本身现在实现了`__eq__()`和`__ne__()`方法，这些方法将在所有等式比较中发生，并继续在预料之外的情况下成功进行，即比较非`PathRegistry`对象时，同时发出警告，指出不应将该对象作为比较的主题。

    参考文献：[#5110](https://www.sqlalchemy.org/trac/ticket/5110)

+   **[orm] [bug]**

    将一个关系设置为 viewonly=True，同时它也是 back_populates 或 backref 配置的目标，现在会发出警告，并最终被禁止。back_populates 具体指的是对属性或集合的改变，当属性受到 viewonly=True 的影响时，这是不允许的。viewonly 属性不受持久性行为的影响，这意味着当它在本地被改变时，它将不会反映出正确的结果。

    参考：[#5149](https://www.sqlalchemy.org/trac/ticket/5149)

+   **[orm] [bug]**

    修复了与[#5080](https://www.sqlalchemy.org/trac/ticket/5080)相同区域的另一个回归，这是通过[#4468](https://www.sqlalchemy.org/trac/ticket/4468)在 1.3.0b3 中引入的，在`with_polymorphic()`创建跨一个关系的选项时，该关系进入对该 with_polymorphic 的基类的关系，然后进一步进入常规映射关系将失败，因为基类组件不会以可被加载器策略定位的方式添加到加载路径中。在[#5080](https://www.sqlalchemy.org/trac/ticket/5080)中应用的更改已经进一步完善，以适应这种情况。

    参考：[#5121](https://www.sqlalchemy.org/trac/ticket/5121)

### engine

+   **[engine] [bug]**

    扩展了执行语句时游标/连接清理的范围，包括当结果对象构造失败时，或者 after_cursor_execute()事件引发错误时，或者 autocommit/autoclose 失败时。这允许在失败时清理 DBAPI 游标，并且对于无连接执行，允许关闭连接并将其返回到连接池，之前等待垃圾回收触发连接池返回。

    参考：[#5182](https://www.sqlalchemy.org/trac/ticket/5182)

### sql

+   **[sql] [bug] [postgresql]**

    修复了 CTE（Common Table Expression）的一个问题，该问题出现在 INSERT/UPDATE/DELETE 语句中同时使用 RETURNING，并且随后无法直接从中进行 SELECT，因为编译器的内部状态会尝试将外部 SELECT 当作一个 DELETE 语句来处理，并访问不存在的状态。

    参考：[#5181](https://www.sqlalchemy.org/trac/ticket/5181)

### postgresql

+   **[postgresql] [bug]**

    修复了“schema_translate_map”功能无法与 PostgreSQL 原生枚举类型（即`Enum`, `ENUM`)一起使用的问题。虽然“CREATE TYPE”语句会被正确地生成带有正确模式的枚举，但在引用枚举的 CREATE TABLE 语句中，模式不会被呈现出来。

    参考：[#5158](https://www.sqlalchemy.org/trac/ticket/5158)

+   **[postgresql] [bug] [reflection]**

    修复了一个 bug，在该 bug 中，PostgreSQL 反射 CHECK 约束将无法解析约束，如果 SQL 文本包含换行符，则正则表达式已调整以适应此情况。拉取请求由 Eric Borczuk 提供。

    参考：[#5170](https://www.sqlalchemy.org/trac/ticket/5170)

### mysql

+   **[mysql] [错误]**

    修复了在 MySQL `Insert.on_duplicate_key_update()` 构造中使用 SQL 函数或其他组合表达式作为列参数时，不正确渲染列本身周围的 `VALUES` 关键字的问题。

    参考：[#5173](https://www.sqlalchemy.org/trac/ticket/5173)

### mssql

+   **[mssql] [错误]**

    修复了 `DATETIMEOFFSET` 类型不适应 `None` 值的问题，这是为了解决此类型首次引入的一系列修复的一部分，首次引入于 [#4983](https://www.sqlalchemy.org/trac/ticket/4983)、[#5045](https://www.sqlalchemy.org/trac/ticket/5045)。此外，添加了支持通过此类型传递后端特定的日期格式字符串的支持，这通常允许在大多数其他 DBAPI 上的日期/时间类型上使用。

    参考：[#5132](https://www.sqlalchemy.org/trac/ticket/5132)

### oracle

+   **[oracle] [错误]**

    修复了一个反射错误，在该错误中，只能为实际由用户拥有但不是由用户拥有的表获取表注释，而不是对用户可见但由其他人拥有的表。由 Dave Hirschfeld 提供的拉取请求。

    参考：[#5146](https://www.sqlalchemy.org/trac/ticket/5146)

### 其他

+   **[用例] [扩展]**

    向 `MutableList.sort()` 函数添加了关键字参数，以便可以提供键函数和“reverse”关键字参数。

    参考：[#5114](https://www.sqlalchemy.org/trac/ticket/5114)

+   **[性能] [错误]**

    对测试系统的内部更改进行了修订，该更改是由 [#5085](https://www.sqlalchemy.org/trac/ticket/5085) 导致的，该更改是无条件加载每个方言的与测试相关的模块，一旦使用了该方言，就会拉入 SQLAlchemy 的测试框架以及 ORM 到模块导入空间。这只会对初始启动时间和内存产生适度的影响，但最好这些附加模块不要对纯 Core 使用产生反向依赖。

    参考：[#5180](https://www.sqlalchemy.org/trac/ticket/5180)

+   **[错误] [安装]**

    将 `inspect.formatannotation` 函数放置在 `sqlalchemy.util.compat` 内，该函数在 vendored 版本的 `inspect.formatargspec` 中需要，该函数未在 cPython 中记录，并且不能保证在将来的 Python 版本中可用。

    参考：[#5138](https://www.sqlalchemy.org/trac/ticket/5138)

## 1.3.13

发布日期：2020 年 1 月 22 日

### orm

+   **[orm] [performance]**

    通过基于映射关系构建连接的系统中发现了性能问题。子句适配系统将用于大多数连接表达式，包括在常见情况下不需要适配的情况。已经对发生适配的条件进行了细化，以便平均非别名连接沿着简单关系使用约 70% 的函数调用。

+   **[orm] [bug] [engine]**

    添加了测试支持，并修复了在短暂对象中创建的大量不必要的引用循环，主要集中在 ORM 查询领域。非常感谢 Carson Ip 在此方面的帮助。

    参考：[#5050](https://www.sqlalchemy.org/trac/ticket/5050), [#5056](https://www.sqlalchemy.org/trac/ticket/5056), [#5071](https://www.sqlalchemy.org/trac/ticket/5071)

+   **[orm] [bug]**

    修复了在 1.3.0b3 版本中引入的加载器选项中的回归问题，该问题通过 [#4468](https://www.sqlalchemy.org/trac/ticket/4468) 引入，其中使用 `PropComparator.of_type()` 创建一个针对前一个关系引用的实体的继承子类的别名实体的加载器选项将无法生成匹配路径。另请参见在此相同版本中修复的 [#5082](https://www.sqlalchemy.org/trac/ticket/5082)，其中涉及类似类型的问题。

    参考：[#5107](https://www.sqlalchemy.org/trac/ticket/5107)

+   **[orm] [bug]**

    修复了在 1.3.0b3 版本中引入的连接式预加载中的回归问题，该问题通过 [#4468](https://www.sqlalchemy.org/trac/ticket/4468) 引入，其中通过 `RelationshipProperty.of_type()` 创建跨越 `with_polymorphic()` 到多态子类的连接选项，然后沿着常规映射关系进一步失败，因为多态子类不会以可以被加载策略定位的方式将自身添加到加载路径中。已进行微调以解决此场景。

    参考：[#5082](https://www.sqlalchemy.org/trac/ticket/5082)

+   **[orm] [bug]**

    修复了在删除使用“version_id”功能的对象时，在 ORM 刷新过程中出现的一个警告，该警告未被测试覆盖。通常情况下，此警告是无法触及的，除非使用的方言将“supports_sane_rowcount”标志设置为 False，这在大多数情况下并不是通常情况，但对于某些 MySQL 配置以及较旧的 Firebird 驱动程序以及可能的一些第三方方言可能是可能的。

    参考：[#5068](https://www.sqlalchemy.org/trac/ticket/5068)

+   **[orm] [bug]**

    修复了一个 bug，即在使用连接的急加载时，当针对查询使用`Query.group_by()`时，不会正确将查询包装在子查询中。当使用任何种类的结果限制方法时，例如 DISTINCT、LIMIT、OFFSET，连接的急加载会将行限制的查询嵌入到子查询中，以便不影响集合结果。出于某种原因，GROUP BY 的存在从未包含在此标准中，即使它具有与使用 DISTINCT 相同的效果。此外，该 bug 将阻止对大多数数据库平台的连接急加载查询使用 GROUP BY，这些数据库平台禁止在查询中存在非聚合、非分组的列，因为连接的急加载的附加列不会被数据库接受。

    参考：[#5065](https://www.sqlalchemy.org/trac/ticket/5065)

### 引擎

+   **[引擎] [错误]**

    修复了一个问题，即在使用具有绑定值处理器的数据类型与“扩展 IN”参数一起使用时，`Compiled` 对象上的值处理器集合会发生变异；特别是，这意味着在使用语句缓存和/或烘焙查询时，同一 compiled._bind_processors 集合会同时发生变异。由于这些处理器对于给定的绑定参数命名空间每次都是相同的函数，因此这个问题实际上没有任何负面影响，但是，`Compiled` 对象的执行绝不应该导致其状态发生任何更改，尤其是考虑到它们旨在在完全构造后是线程安全和可重复使用的。

    参考：[#5048](https://www.sqlalchemy.org/trac/ticket/5048)

### sql

+   **[sql] [用例]**

    使用`GenericFunction`创建的函数现在可以通过将`quoted_name`构造分配给对象的.name 元素来指定函数的名称是否应该带引号或不带引号。在 1.3.4 之前，从不对函数名称应用引号，并且在[#4467](https://www.sqlalchemy.org/trac/ticket/4467)中引入了一些引号，但没有任何方法来强制对混合大小写名称进行引用。此外，当作为名称使用`quoted_name`构造时，将正确在函数注册表中注册其小写名称，以便名称继续通过`func.`注册表可用。

    另请参阅

    `GenericFunction`

    参考：[#5079](https://www.sqlalchemy.org/trac/ticket/5079)

### postgresql

+   **[postgresql] [usecase]**

    为`CTE`构造添加了前缀支持，以支持 Postgresql 12 中的“MATERIALIZED”和“NOT MATERIALIZED”短语。感谢 Marat Sharafutdinov 的拉取请求。

    另请参阅

    `HasCTE.cte()`

    参考：[#5040](https://www.sqlalchemy.org/trac/ticket/5040)

+   **[postgresql] [bug]**

    修复了 PostgreSQL 方言无法解析反射的 CHECK 约束的问题，该约束是一个布尔值函数（而不是布尔值表达式）。

    参考：[#5039](https://www.sqlalchemy.org/trac/ticket/5039)

### mssql

+   **[mssql] [bug]**

    修复了一个问题，即将时区感知的`datetime`值转换为字符串以用作`DATETIMEOFFSET`列的参数值时，省略了小数秒。

    参考：[#5045](https://www.sqlalchemy.org/trac/ticket/5045)

### 测试

+   **[tests] [bug]**

    修复了一些在 Windows 上由于 SQLite 文件锁定问题而导致的测试失败，以及连接池相关测试中的一些时间问题；感谢 Federico Caselli 的拉取请求。

    参考：[#4946](https://www.sqlalchemy.org/trac/ticket/4946)

+   **[tests] [postgresql]**

    通过测试 max_prepared_transactions 是否设置为大于 0 的值，改进了对 PostgreSQL 数据库的两阶段事务需求的检测。感谢 Federico Caselli 的拉取请求。

    参考：[#5057](https://www.sqlalchemy.org/trac/ticket/5057)

### 杂项

+   **[bug] [ext]**

    修复了 sqlalchemy.ext.serializer 中的一个 bug，即如果一个唯一的`BindParameter`对象同时存在于映射本身和查询的过滤条件中，那么它可能会与自身冲突，因为一侧将用于非反序列化版本，另一侧将用于反序列化版本。在`BindParameter`中添加了类似于其“clone”方法的逻辑，该方法在反序列化时将参数名称唯一化，以避免与原始版本冲突。

    参考：[#5086](https://www.sqlalchemy.org/trac/ticket/5086)

## 1.3.12

发布日期：2019 年 12 月 16 日

### orm

+   **[orm] [bug]**

    修复了涉及`lazy="raise"`策略的问题，其中 ORM 删除对象会对具有配置为`lazy="raise"`的简单“use-get”样式的多对一关系引发异常。这与 1.3 中引入的更改不一致，作为[#4353](https://www.sqlalchemy.org/trac/ticket/4353)的一部分，其中确定了不期望发出 SQL 的历史操作应绕过`lazy="raise"`检查，并且实际上将其视为此情况下的`lazy="raise_on_sql"`。修复调整了延迟加载器策略，以便在指示延迟加载器不应发出 SQL 的情况下，如果对象不存在，则不会引发异常。

    参考：[#4997](https://www.sqlalchemy.org/trac/ticket/4997)

+   **[orm] [bug]**

    修复了 1.3.0 中与[#4351](https://www.sqlalchemy.org/trac/ticket/4351)中的关联代理重构相关的回归，该回归阻止了`composite()`属性在引用它们的关联代理方面的工作。

    参考：[#5000](https://www.sqlalchemy.org/trac/ticket/5000)

+   **[orm] [bug]**

    在设置`viewonly=True`的同时在`relationship()`上设置与持久性相关的标志现在会发出常规警告，因为这些标志对于`viewonly=True`关系没有意义。特别是，“cascade”设置有自己的警告，根据各个值生成，例如“delete, delete-orphan”，不应适用于`viewonly`关系。但请注意，在“cascade”情况下，这些设置仍然错误地生效，即使关系设置为“viewonly”。在 1.4 中，将禁止在`viewonly=True`关系上设置所有与持久性相关的级联设置，以解决此问题。

    参考：[#4993](https://www.sqlalchemy.org/trac/ticket/4993)

+   **[orm] [bug] [py3k]**

    修复了将集合分配给自身作为切片时出现的问题，导致变异操作失败，因为它首先会意外地擦除分配的集合。由于不改变内容的赋值不应生成事件，因此该操作现在是一个空操作。请注意，此修复仅适用于 Python 3；在 Python 2 中，此情况下不会调用`__setitem__`钩子；而是使用`__setslice__`，它会在所有情况下逐个重新创建列表项。

    参考：[#4990](https://www.sqlalchemy.org/trac/ticket/4990)

+   **[orm] [bug]**

    修复了一个问题，即如果在 Core 引擎/连接级别失败了事务的“begin”，例如由于网络错误或某些事务配方导致数据库被锁定，那么在从连接池获取该连接并立即返回它的 ORM `Session` 上下文中，ORM `Session` 将不会关闭连接，尽管该连接未存储在该 `Session` 的状态中。这将导致连接被垃圾收集中的连接池弱引用处理程序清除，这是一种不理想的代码路径，在某些特殊配置中可能会在标准错误中发出错误。

    参考：[#5034](https://www.sqlalchemy.org/trac/ticket/5034)

### sql

+   **[sql] [bug]**

    修复了一个错误，即传递给 `select()` 的 “distinct” 关键字不会像 `select.distinct()` 那样将字符串值视为 “标签引用”，而是无条件地引发异常。这个关键字参数和其他传递给 `select()` 的参数最终将在 SQLAlchemy 2.0 中被弃用。

    参考：[#5028](https://www.sqlalchemy.org/trac/ticket/5028)

+   **[sql] [bug]**

    更改了“无法解析标签引用”异常的文本，以包括其他类型的标签强制转换，即在 PostgreSQL 方言下，“DISTINCT” 也属于此类别。

### sqlite

+   **[sqlite] [bug]**

    修复了解决 SQLite 将 “numeric” 亲和性分配给 JSON 数据类型的行为问题，首次描述于 Support for SQLite JSON Added，该行为将标量数值 JSON 值返回为数字而不是可以进行 JSON 反序列化的字符串。SQLite 特定的 JSON 反序列化器现在对于这种情况会优雅地降级为异常，并且对于单个数值的情况绕过反序列化，因为从 JSON 视角来看，它们已经被反序列化了。

    参考：[#5014](https://www.sqlalchemy.org/trac/ticket/5014)

### mssql

+   **[mssql] [bug]**

    通过添加 PyODBC 级别的结果处理程序修复了对 PyODBC 上 `DATETIMEOFFSET` 数据类型的支持，因为它不包括对此数据类型的本机支持。这包括使用 Python 3 中的“timezone” tzinfo 子类来设置时区，在 Python 2 中使用 sqlalchemy.util 中的“timezone”的最小回退。

    参考：[#4983](https://www.sqlalchemy.org/trac/ticket/4983)

## 1.3.11

发布日期：2019 年 11 月 11 日

### orm

+   **[orm] [usecase]**

    添加了访问器 `Query.is_single_entity()` 到 `Query`，该访问器将指示此 `Query` 返回的结果是一个 ORM 实体列表，还是实体或列表达式的元组。SQLAlchemy 希望在未来的版本中改进单个实体/元组的行为，以便行为在前期就是明确的，但是当前行为下此属性应该有所帮助。感谢 Patrick Hayes 提供的拉取请求。

    参考：[#4934](https://www.sqlalchemy.org/trac/ticket/4934)

+   **[orm] [bug]**

    `relationship.omit_join` 标志并非意在手动设置为 True，当发生此情况时将会发出警告。`omit_join` 优化会被自动检测到，`omit_join` 标志仅用于在假设优化可能干扰正确结果的情况下禁用优化，但在现代版本中尚未观察到这种情况。在非默认主连接条件正在使用时，将标志设置为 True 可能会导致 selectin 加载功能无法正确工作。

    参考：[#4954](https://www.sqlalchemy.org/trac/ticket/4954)

+   **[orm] [bug]**

    如果将主键值传递给 `Query.get()`，并且所有主键列位置都为 None，则会发出警告。以前，传递单个 None 会在元组之外引发 `TypeError`，传递复合 None（None 值的元组）会悄悄通过。现在的修复将单个 None 强制转换为元组，以便与其他 None 条件一致处理。感谢 Lev Izraelit 对此的帮助。

    参考：[#4915](https://www.sqlalchemy.org/trac/ticket/4915)

+   **[orm] [bug]**

    `BakedQuery` 不会缓存通过 `QueryEvents.before_compile()` 事件修改的查询，因此可能会对每次运行生效的编译钩子应用临时修改。特别是对于修改延迟加载和急加载中使用的查询的事件非常有帮助，比如“select in”加载。为了重新启用通过此事件修改的查询的缓存，添加了一个新标志 `bake_ok`；详细信息请参见使用 before_compile 事件。

    提供一种新形式的 SQL 缓存的长期计划应该更全面地解决这种问题。

    参考：[#4947](https://www.sqlalchemy.org/trac/ticket/4947)

+   **[orm] [bug]**

    修复了 ORM 中的 bug，其中引用某种方式引用本地主表的可选择的“secondary”表在生成关系相关的连接条件时，无论是通过`Query.join()`还是通过`joinedload()`，都会对连接条件的两侧应用别名。现在排除了“本地”一侧。

    参考：[#4974](https://www.sqlalchemy.org/trac/ticket/4974)

### engine

+   **[engine] [bug]**

    修复了一个 bug，在日志记录和错误报告中使用的参数 repr 需要额外的上下文以区分单个语句的参数列表和参数列表的列表，因为“列表的列表”结构也可能表示第一个参数本身是一个列表的单个参数列表，例如用于数组参数。引擎/连接现在传入一个额外的布尔值，指示参数应如何考虑。唯一期望数组作为参数的 SQLAlchemy 后端是使用 pyformat 参数的 psycopg2，因此这个问题并不太明显，但随着其他使用位置参数的驱动程序获得更多功能，支持这一点变得重要。它还消除了参数 repr 函数根据传递的参数结构猜测的需要。

    参考：[#4902](https://www.sqlalchemy.org/trac/ticket/4902)

+   **[engine] [bug] [postgresql]**

    修复了`Inspector`中的 bug，其中缓存键生成没有考虑以元组形式传递的参数，例如为了返回 PostgreSQL 方言的视图名称样式的元组。这将导致检查器对更具体的一组条件进行了过于一般化的缓存。逻辑已经调整为包含缓存中的每个关键字元素，因为每个参数都应适用于缓存，否则应该绕过方言的缓存装饰器。

    参考：[#4955](https://www.sqlalchemy.org/trac/ticket/4955)

### sql

+   **[sql] [usecase]**

    对`JSON`类型的表达式添加了新的访问器，以允许特定数据类型的访问和比较，包括字符串、整数、数字、布尔元素。这修订了在比较值时将其转换为字符串的文档化方法，而是在 PostgreSQL、SQlite、MySQL 方言中添加了特定功能，以可靠地在所有情况下提供这些基本类型。

    另请参见

    `JSON`

    `Comparator.as_string()`

    `Comparator.as_boolean()`

    `Comparator.as_float()`

    `Comparator.as_integer()`

    参考：[#4276](https://www.sqlalchemy.org/trac/ticket/4276)

+   **[sql] [用例]**

    `text()` 构造现在支持“unique”绑定参数，这将在编译时动态使自己唯一，从而允许多个具有相同绑定参数名称的 `text()` 构造组合在一起。

    参考：[#4933](https://www.sqlalchemy.org/trac/ticket/4933)

+   **[sql] [bug] [py3k]**

    将 `quoted_name` 构造的 `repr()` 更改为在 Python 3 下使用常规字符串 repr()，而不是通过“backslashreplace”转义，这可能会误导。

    参考：[#4931](https://www.sqlalchemy.org/trac/ticket/4931)

### 模式

+   **[模式] [用例]**

    增加了对“计算列”（computed columns）的 DDL 支持；这些是针对具有服务器计算值的列的 DDL 列规范，无论是在 SELECT 时（称为“虚拟”）还是在它们被 INSERT 或 UPDATE 时（称为“存储”）都有。支持已建立在 Postgresql、MySQL、Oracle SQL Server 和 Firebird 上。感谢 Federico Caselli 在这方面的大量工作。

    另请参阅

    计算列 (GENERATED ALWAYS AS)

    参考：[#4894](https://www.sqlalchemy.org/trac/ticket/4894)

+   **[模式] [bug]**

    修复了一个 bug，即一个表的列标签与普通列名重叠，例如“foo.id AS foo_id”与“foo.foo_id”，会在此重叠被检测之前过早生成`._label`属性，因为在列上使用`index=True`或`unique=True`标志与默认命名约定`"column_0_label"`结合使用。然后，当稍后使用`._label`生成绑定参数名称时，特别是在 ORM 生成 UPDATE 语句的 WHERE 子句时，会导致失败。通过使用用于 DDL 生成的替代`._label`访问器来修复此问题，该访问器不会影响`Column`的状态。该访问器还绕过了键去重步骤，因为对于 DDL 来说这是不必要的，命名现在在 DDL 中一致地是`"<tablename>_<columnname>"`，在用于 DDL 时不会有任何后续的数字符号。

    参考：[#4911](https://www.sqlalchemy.org/trac/ticket/4911)

### mysql

+   **[mysql] [bug]**

    添加了从基本 pymysql.Error 类中解释的“连接被终止”消息，以便检测到关闭的连接，根据报告，此消息通过 pymysql.InternalError() 对象到达，这表明 pymysql 未正确处理它。

    参考：[#4945](https://www.sqlalchemy.org/trac/ticket/4945)

### mssql

+   **[mssql] [bug]**

    修复了 MSSQL 方言中的问题，即在 SELECT 中基于表达式的 OFFSET 值会被拒绝，尽管方言可以在 ROW NUMBER 导向的 LIMIT/OFFSET 结构中呈现此表达式。

    参考：[#4973](https://www.sqlalchemy.org/trac/ticket/4973)

+   **[mssql] [bug]**

    修复了`Engine.table_names()`方法中的问题，该方法会将方言的默认模式名称反馈给方言级别的表函数，在 SQL Server 的情况下，会将其解释为 mssql 方言视图中的点标记模式名称，这会导致在数据库用户名实际上包含点的情况下该方法失败。在 1.3 版本中，此方法仍然被`MetaData.reflect()`函数使用，因此是一个重要的代码路径。在当前主开发分支 1.4 中，这个问题不存在，因为`MetaData.reflect()`没有使用这个方法，也不会显式传递默认模式名称。尽管如此，修复仍然通过将方言返回的默认服务器名称值用 quoted_name() 包装来防止在任何情况下被解释为点标记名称。

    参考：[#4923](https://www.sqlalchemy.org/trac/ticket/4923)

### oracle

+   **[oracle] [usecase]**

    为 cx_Oracle 方言添加了方言级别标志 `encoding_errors`，可以作为 `create_engine()` 的一部分指定。在 Python 2 下，这将传递给 SQLAlchemy 的 unicode 解码转换器，而在 Python 3 下，这将传递给 cx_Oracle 的 `cursor.var()` 对象作为 `encodingErrors` 参数，用于目标数据库中存在无法获取的破损编码的非常罕见情况，除非放宽错误处理。该值最终是传递给 `decode()` 的 Python “编码错误”参数之一。

    参考：[#4799](https://www.sqlalchemy.org/trac/ticket/4799)

+   **[oracle] [bug] [firebird]**

    修改了 Oracle 和 Firebird 方言的“名称规范化”方法，将这些方言的大写作为不区分大小写的约定转换为 SQLAlchemy 中的小写作为不区分大小写的约定，以便不自动将 `quoted_name` 构造应用于在大写或小写转换下匹配自身的名称，这对许多非欧洲字符来说是一种情况。在元数据结构中使用的所有名称都会转换为 `quoted_name` 对象；这里的更改只会影响一些检查函数的输出。

    参考：[#4931](https://www.sqlalchemy.org/trac/ticket/4931)

+   **[oracle] [bug]**

    当使用绑定参数时，`NCHAR` 数据类型现在将绑定到 `cx_Oracle.FIXED_NCHAR` DBAPI 数据绑定，从而提供针对可变长度字符串的正确比较行为。以前，`NCHAR` 数据类型会绑定到 `cx_oracle.NCHAR`，这不是固定长度；`CHAR` 数据类型已经绑定到 `cx_Oracle.FIXED_CHAR`，因此现在一致的是`NCHAR` 绑定到 `cx_Oracle.FIXED_NCHAR`。

    ��考：[#4913](https://www.sqlalchemy.org/trac/ticket/4913)

### 测试

+   **[tests] [bug]**

    修复了在新版本 SQLite（3.30 或更高版本）中可能出现的测试失败，这是由于它们添加了空值排序语法以及对聚合函数的新限制。感谢 Nils Philippsen 提交的拉取请求。

    参考：[#4920](https://www.sqlalchemy.org/trac/ticket/4920)

### 杂项

+   **[bug] [installation] [windows]**

    添加了一个解决方案，用于解决在 Windows 安装中观察到的 setuptools 相关故障，当未安装 MSVC 构建依赖项时，setuptools 没有正确报告构建错误，因此不允许优雅地降级为非 C 扩展构建。

    参考：[#4967](https://www.sqlalchemy.org/trac/ticket/4967)

+   **[bug] [firebird]**

    向 Firebird 断开检测添加了额外的“断开”消息“写入数据到连接时出错”。拉取请求由 lukens 提供。

    参考：[#4903](https://www.sqlalchemy.org/trac/ticket/4903)

## 1.3.10

发布日期：2019 年 10 月 9 日

### mssql

+   **[mssql] [bug]**

    修复了 SQL Server 方言中新“max_identifier_length”功能的错误，其中 mssql 方言已经具有此标志，但实现未正确适应新的初始化挂钩。

    参考：[#4857](https://www.sqlalchemy.org/trac/ticket/4857)

### oracle

+   **[oracle] [bug]**

    修复了 Oracle 方言中的回归，该方言在 Oracle 服务器 12.2 及更高版本上无意中使用了 128 个字符的最大标识符长度，尽管 1.3 系列的其余部分的规定是该值保持在 30，直到 SQLAlchemy 1.4 版本。还修复了检索“兼容性”版本的问题，并删除了当“v$parameter”视图不可访问时发出的警告，因为这导致用户困惑。

    参考：[#4857](https://www.sqlalchemy.org/trac/ticket/4857), [#4898](https://www.sqlalchemy.org/trac/ticket/4898)

## 1.3.9

发布日期：2019 年 10 月 4 日

### orm

+   **[orm] [bug]**

    修复了由[#4775](https://www.sqlalchemy.org/trac/ticket/4775)（在版本 1.3.6 中发布）引起的 selectinload 加载策略中的回归，其中 None 的多对一属性将不再被加载器填充。虽然由于延迟加载器在获取时填充 None，通常不会注意到这一点，但如果对象被分离，将导致分离实例错误。

    参考：[#4872](https://www.sqlalchemy.org/trac/ticket/4872)

+   **[orm] [bug]**

    将普通字符串表达式传递给`Session.query()`已被弃用，因为所有字符串强制转换在[#4481](https://www.sqlalchemy.org/trac/ticket/4481)中已被移除，而这个应该已经包含在内。可以使用`literal_column()`函数生成文本列表达式。

    参考：[#4873](https://www.sqlalchemy.org/trac/ticket/4873)

+   **[orm] [bug]**

    在`Session`可能会在`SessionEvents.after_flush()`挂钩中发生的加载操作中，隐式地将一个对象与具有相同主键的另一个对象交换出标识映射，从而分离旧对象，这可能是发生的结果。警告旨在通知用户某些特殊条件导致此情况发生，并且先前的对象可能不处于预期状态。

    参考：[#4890](https://www.sqlalchemy.org/trac/ticket/4890)

### 引擎

+   **[引擎] [用例]**

    添加了新的`create_engine()`参数`create_engine.max_identifier_length`。这将覆盖方言编码的“最大标识符长度”，以适应最近更改了此长度的数据库，而 SQLAlchemy 方言尚未调整以检测该版本。此参数与现有的`create_engine.label_length`参数交互，因为它为匿名生成的标签建立了最大（和默认）值。此外，方言系统中已添加了关于最大标识符长度的连接后检测。此功能首先由 Oracle 方言使用。

    另请参阅

    最大标识符长度 - 在 Oracle 方言文档中

    参考：[#4857](https://www.sqlalchemy.org/trac/ticket/4857)

### sql

+   **[sql] [用例]**

    添加了一个明确的错误消息，用于当传递给`Table`的对象不是`SchemaItem`对象时，而不是解析为属性错误。

    参考：[#4847](https://www.sqlalchemy.org/trac/ticket/4847)

+   **[sql] [错误]**

    在绑定参数中干扰“pyformat”或“named”格式的字符，即`%，（，）`和空格字符，以及一些通常不希望出现的字符，会在使用匿名名称的`bindparam()`中被提前剥离。这些匿名名称通常是从一个包含这些字符的命名列自动生成的，不使用`.key`，以便它们既不干扰 SQLAlchemy 编译器对字符串格式化的使用，也不干扰参数的驱动程序级解析，这两者在修复之前可能会被演示。此更改仅适用于内部生成和使用的匿名参数名称，不适用于最终用户定义的名称，因此该更改不应对任何现有代码产生影响。特别适用于 psycopg2 驱动程序，它否则不会引用特殊参数名称，但也会剥离前导下划线以适应 Oracle（但尚未剥离前导数字，因为某些匿名参数当前完全基于数字/下划线）；无论如何，Oracle 继续引用包含特殊字符的参数名称。

    参考：[#4837](https://www.sqlalchemy.org/trac/ticket/4837)

### sqlite

+   **[sqlite] [用例]**

    增加了对 sqlite“URI”连接的支持，允许在查询字符串中传递特定于 sqlite 的标志，例如 Python sqlite3 驱动程序支持的“只读”。

    另请参阅

    URI 连接

    参考：[#4863](https://www.sqlalchemy.org/trac/ticket/4863)

### mssql

+   **[mssql] [错误]**

    在用于反射的`Table`中使用 SQL Server 多部分模式名称时，对模式名称应用了标识符引用，同时对`Inspector`方法（如`Inspector.get_table_names()`）也进行了处理；这适用于数据库名称中的特殊字符或空格。此外，如果当前数据库与传递的目标所有者数据库名称匹配，则不会发出“use”语句。

    参考：[#4883](https://www.sqlalchemy.org/trac/ticket/4883)

### oracle

+   **[oracle] [用例]**

    现在 Oracle 方言在使用 Oracle 版本 12.2 或更高版本时会发出警告，如果未设置`create_engine.max_identifier_length`参数。在这种特定情况下，默认版本为 Oracle 服务器配置中设置的“兼容性”版本，而不是实际服务器版本。在 1.4 版本中，12.2 或更高版本的默认 max_identifier_length 将移至 128 个字符。为了保持向前兼容性，应用程序应将`create_engine.max_identifier_length`设置为 30，以保持相同的长度行为，或者设置为 128 以测试即将到来的行为。这个长度决定了生成的约束名称如何被截断，例如`CREATE CONSTRAINT`和`DROP CONSTRAINT`语句，这意味着新长度可能会导致与使用旧长度生成的名称不匹配，影响数据库迁移。

    另请参阅

    最大标识符长度 - 在 Oracle 方言文档中

    参考：[#4857](https://www.sqlalchemy.org/trac/ticket/4857)

+   **[oracle] [错误]**

    当使用 SQLAlchemy 的`Date`、`DateTime`或`Time`数据类型时，恢复了将 cx_Oracle.DATETIME 添加到 setinputsizes() 调用中的功能，因为一些复杂查询需要它存在。这在 1.2 系列中由于任意原因被移除。

    参考：[#4886](https://www.sqlalchemy.org/trac/ticket/4886)

### 测试

+   **[测试] [错误]**

    修复了在 1.3.8 中发布的单元测试回归，该回归会导致 Oracle、SQL Server 和其他非本地 ENUM 平台失败，原因是作为[#4285](https://www.sqlalchemy.org/trac/ticket/4285)的一部分添加了新的枚举测试，枚举在工作单元中创建了重复的名称约束。

    参考：[#4285](https://www.sqlalchemy.org/trac/ticket/4285)

## 1.3.8

发布日期：2019 年 8 月 27 日

### ORM

+   **[ORM] [用例]**

    添加了对使用 Python pep-435 枚举对象作为 ORM 映射的主键列值的`Enum`数据类型的支持。由于这些值本身不可排序，而 ORM 对主键要求可排序，因此在类型系统中添加了一个新的`TypeEngine.sort_key_function`属性，允许任何 SQL 类型为其类型的 Python 对象实现排序，工作单元会查询此排序。`Enum`类型然后使用给定枚举的数据库值定义这一点。通过将可调用对象传递给`Enum.sort_key_function`参数，还可以重新定义排序方案。拉取请求由 Nicolas Caniart 提供。

    参考：[#4285](https://www.sqlalchemy.org/trac/ticket/4285)

+   **[ORM] [错误]**

    修复了由于内部上下文字典中的映射器/关系状态而导致`Load`对象不可 pickle 的 bug。现在，这些对象通过与加载器选项系统中其他元素类似的技术转换为可 pickle。

    参考：[#4823](https://www.sqlalchemy.org/trac/ticket/4823)

### 引擎

+   **[引擎] [特性]**

    添加了新参数`create_engine.hide_parameters`，当设置为 True 时，将导致 SQL 参数不再被记录，也不会在`StatementError`对象的字符串表示中呈现。

    参考：[#4815](https://www.sqlalchemy.org/trac/ticket/4815)

+   **[engine] [bug]**

    修复了一个问题，即如果 dialect“initialize”过程在首次连接时遇到意外异常，初始化过程将无法完成，然后在后续连接尝试中不再尝试，使得 dialect 处于未初始化或部分初始化状态，需要根据对实时连接的检查来建立参数的范围。事件系统中的“invoke once”逻辑已经重新设计，以适应这种情况，使用新的私有 API 功能建立一个“exec once”钩子，该钩子将继续允许初始化程序在后续连接中触发，直到完成而不引发异常。这不会影响事件系统中现有的`once=True`标志的行为。

    参考：[#4807](https://www.sqlalchemy.org/trac/ticket/4807)

### postgresql

+   **[postgresql] [usecase]**

    增加了对包含特殊 PostgreSQL 限定词“NOT VALID”的 CHECK 约束的反射支持，这些限定词可能存在于已添加到现有表中的 CHECK 约束中，指示这些约束不适用于表中现有数据。由`Inspector.get_check_constraints()`返回的 CHECK 约束的 PostgreSQL 字典可能包含一个额外的条目`dialect_options`，其中将包含一个条目`"not_valid": True`，如果检测到此符号。感谢 Bill Finn 的拉取请求。

    参考：[#4824](https://www.sqlalchemy.org/trac/ticket/4824)

+   **[postgresql] [bug]**

    修订了对于 1.3.7 版本中添加的对 psycopg2“execute_values()”功能的支持的方法，该支持是为了[#4623](https://www.sqlalchemy.org/trac/ticket/4623)而添加的。该方法依赖于一个正则表达式，该表达式无法匹配更复杂的 INSERT 语句，例如涉及子查询的语句。新方法完全匹配作为 VALUES 子句呈现的字符串。

    参考：[#4623](https://www.sqlalchemy.org/trac/ticket/4623)

+   **[postgresql] [bug]**

    修复了一个错误，即当针对一个`array`对象使用 Postgresql 运算符（如`Comparator.contains()`和`Comparator.contained_by()`）时，对于非整数值，由于一个错误的断言语句，这些运算符无法正确运行。

    参考：[#4822](https://www.sqlalchemy.org/trac/ticket/4822)

### sqlite

+   **[sqlite] [bug] [reflection]**

    修复了设置为仅通过表名而不是列名引用父表的 FOREIGN KEY 不会正确反映“referred columns”的 bug，因为如果没有显式给出，SQLite 的 PRAGMA 不会报告这些列。出于某种原因，这是硬编码为假定本地列的名称，这对某些情况可能有效，但不正确。新方法反映了被引用表的主键，并使用约束列列表作为被引用列列表，如果远程列不直接在反映的 pragma 中存在。

    参考：[#4810](https://www.sqlalchemy.org/trac/ticket/4810)

## 1.3.7

发布日期：2019 年 8 月 14 日

### orm

+   **[orm] [bug]**

    修复了由新的多对一逻辑的 selectinload 引起的回归问题，其中一个基于主键连接条件而不是真实外键的条件会导致如果父对象上给定键值的相关对象不存在，则会导致 KeyError。

    参考：[#4777](https://www.sqlalchemy.org/trac/ticket/4777)

+   **[orm] [bug]**

    修复了在与应用了基于表达式的“offset”的查询一起使用`Query.first()`或切片表达式时会引发 TypeError 的 bug，由于针对“offset”的“or”条件不希望它是 SQL 表达式而不是整数或 None。

    参考：[#4803](https://www.sqlalchemy.org/trac/ticket/4803)

### sql

+   **[sql] [bug]**

    修复了`Index`对象中包含一些无法解析为特定列的混合函数表达式，与基于字符串的列名结合使用时，会导致初始化内部状态失败，从而在 DDL 编译过程中失败的问题。

    参考：[#4778](https://www.sqlalchemy.org/trac/ticket/4778)

+   **[sql] [bug]**

    修复了`TypeEngine.column_expression()`方法不会应用于 UNION 或其他`_selectable.CompoundSelect`内部的后续 SELECT 语句的 bug，即使 SELECT 语句在语句的最顶层呈现。新逻辑现在区分了呈现列表达式（对于列表中的所有 SELECT 都需要）与收集结果行的返回数据类型（仅对第一个 SELECT 需要）。

    参考：[#4787](https://www.sqlalchemy.org/trac/ticket/4787)

+   **[sql] [bug]**

    修复了内部克隆 SELECT 结构可能导致关键错误的问题，如果 SELECT 的副本更改其状态，使其列列表发生更改。这在一些 ORM 场景中被观察到，可能是 1.3 及以上版本独有的问题，因此部分是回归修复。

    参考：[#4780](https://www.sqlalchemy.org/trac/ticket/4780)

### postgresql

+   **[postgresql] [usecase]**

    为 psycopg2 方言添加了新的方言标志，`executemany_mode`，它取代了之前的实验性`use_batch_mode`标志。`executemany_mode`支持 psycopg2 提供的“执行批处理”和“执行值”函数，后者用于编译的`insert()`构造。感谢 Yuval Dinari 提供的拉取请求。

    另请参阅

    Psycopg2 快速执行助手

    参考：[#4623](https://www.sqlalchemy.org/trac/ticket/4623)

### mysql

+   **[mysql] [用例]**

    将 ARRAY 和 MEMBER 添加到 MySQL 保留字列表中，因为 MySQL 8.0 现在已将这些设为保留字。

    参考：[#4783](https://www.sqlalchemy.org/trac/ticket/4783)

+   **[mysql] [错误]**

    当 MySQL 方言的 charset 给定给 MySQL 驱动程序时，连接开始时将发出“SET NAMES”以平息 MySQL 8.0 中观察到的一个明显行为，即在 UNION 包含与形式为 CAST(NULL AS CHAR(..))的列联接的字符串列时引发排序规则错误，这正是 SQLAlchemy 的 polymorphic_union 函数所做的。该问题似乎至少影响了 PyMySQL 一年，但是最近出现了 mysqlclient 1.4.4，基于这个 DBAPI 创建连接的方式发生了变化。由于该指令的存在影响了三个不同的 MySQL charset 设置，每个设置根据其是否存在具有复杂的影响，因此 SQLAlchemy 现在将在新连接上发出该指令，以确保正确的行为。

    参考：[#4804](https://www.sqlalchemy.org/trac/ticket/4804)

+   **[mysql] [错误]**

    添加了另一个修复上游 MySQL 8 问题的方法，其中对于外键约束反射中的大小写敏感的表名错误地报告，这是首次添加到[#4344](https://www.sqlalchemy.org/trac/ticket/4344)的修复的扩展，它影响了大小写敏感的列名。新问题发生在 MySQL 8.0.17，因此 88718 修复的一般逻辑仍然存在。

    另请参阅

    [`bugs.mysql.com/bug.php?id=96365`](https://bugs.mysql.com/bug.php?id=96365) - 上游错误

    参考：[#4751](https://www.sqlalchemy.org/trac/ticket/4751)

### sqlite

+   **[sqlite] [错误]**

    支持 json 的方言应该在 create_engine()级别接受参数`json_serializer`和`json_deserializer`，然而 SQLite 方言称它们为`_json_serializer`和`_json_deserilalizer`。已更正名称，旧名称在更改警告下被接受，并且这些参数现在被文档化为`create_engine.json_serializer`和`create_engine.json_deserializer`。

    参考：[#4798](https://www.sqlalchemy.org/trac/ticket/4798)

+   **[sqlite] [错误]**

    修复了在 SQLite 方言中使用“PRAGMA table_info”时的 bug，这意味着反射功能用于检测表存在性、表列列表和外键列表时，默认会在任何附加数据库中的任何表上运行，当未给出模式名称且表在基本模式中不存在时。修复明确地对“main”模式运行 PRAGMA，然后如果“main”返回没有行，则对“temp”模式运行，以保持“无模式”命名空间中的表和临时表的行为，附加表仅在“模式”命名空间中。

    参考：[#4793](https://www.sqlalchemy.org/trac/ticket/4793)

### mssql

+   **[mssql] [usecase]**

    为 SQL Server 新增了`try_cast()`构造，发出“TRY_CAST”语法。感谢 Leonel Atencio 的 Pull 请求。

    参考：[#4782](https://www.sqlalchemy.org/trac/ticket/4782)

### 杂项

+   **[bug] [events]**

    修复了事件系统中的问题，其中使用`once=True`标志与动态生成的监听器函数会导致未来事件的事件注册失败，如果这些监听器函数在使用后被垃圾回收，因为假设监听函数是强引用的。现在“once”包装被修改为持久地强引用内部函数，并更新了文档，使用“once”不意味着自动注销监听器函数。

    参考：[#4794](https://www.sqlalchemy.org/trac/ticket/4794)

## 1.3.6

发布日期：2019 年 7 月 21 日

### orm

+   **[orm] [feature]**

    新增了新的加载器选项方法`Load.options()`，允许按层次结构构建加载器选项，这样就可以在不需要多次调用`defaultload()`的情况下，将许多子选项应用于特定路径。感谢 Alessio Bogon 提出的想法。

    参考：[#4736](https://www.sqlalchemy.org/trac/ticket/4736)

+   **[orm] [performance]**

    应用于选择加载中的优化[#4340](https://www.sqlalchemy.org/trac/ticket/4340)，其中不需要 JOIN 即可急切加载相关项目的优化现在也适用于多对一关系，因此只查询相关表以进行简单的连接条件。在这种情况下，基于父对象上的外键列的值查询相关项目；如果这些列被延迟加载或其他方式未加载到集合中的任何父对象上，则加载器将退回到 JOIN 方法。

    参考：[#4775](https://www.sqlalchemy.org/trac/ticket/4775)

+   **[orm] [bug]**

    修复了由[#4365](https://www.sqlalchemy.org/trac/ticket/4365)引起的回归，其中从实体到自身的连接不使用别名时不再引发信息性错误消息，而是在断言失败时失败。已恢复信息性错误条件。

    参考：[#4773](https://www.sqlalchemy.org/trac/ticket/4773)

+   **[orm] [bug]**

    修复了一个问题，即`_ORMJoin.join()`方法，这是一个未内部使用的 ORM 级方法，它公开了通常是内部过程的 `Query.join()` 方法，未正确传播`full`和`outerjoin`关键字参数。拉取请求由 Denis Kataev 提供。

    参考：[#4713](https://www.sqlalchemy.org/trac/ticket/4713)

+   **[orm] [bug]**

    修复了一个 bug，即指定了`uselist=True`的多对一关系在主键更改期间未能正确更新的问题，其中相关列需要更改。

    参考：[#4772](https://www.sqlalchemy.org/trac/ticket/4772)

+   **[orm] [bug]**

    修复了一个 bug，即检测与“动态”关系一对一或多对一使用的错误配置的错误，如果关系配置为`uselist=True`，则会失败。当前的修复方案是发出警告，而不是引发异常，因为这样做会导致向后不兼容，但在将来的版本中，它将引发异常。

    参考：[#4772](https://www.sqlalchemy.org/trac/ticket/4772)

+   **[orm] [bug]**

    修复了一个 bug，即对一个尚不存在的映射属性创建同义词时会引发递归错误，例如当它引用配置映射器之前的 backref 时会出现，当尝试在其上测试最终不存在的属性时会引发递归错误（例如当类通过 Sphinx autodoc 运行时会发生），因为同义词的未配置状态会将其置于未找到属性的循环中。

    参考：[#4767](https://www.sqlalchemy.org/trac/ticket/4767)

### engine

+   **[engine] [bug]**

    修复了一个 bug，即使用反射函数（例如`MetaData.reflect()`）与已应用执行选项的`Engine`对象将失败，因为生成的`OptionEngine`代理对象未包括在反射例程中使用的`.engine`属性。

    参考：[#4754](https://www.sqlalchemy.org/trac/ticket/4754)

### sql

+   **[sql] [bug]**

    调整了 `Enum` 的初始化，以最小化调用给定 PEP-435 枚举对象的 `.__members__` 属性的频率，以适应这种属性的调用成本很高的情况，这是一些常用的第三方枚举库的情况。

    参考：[#4758](https://www.sqlalchemy.org/trac/ticket/4758)

+   **[sql] [bug] [postgresql]**

    修复了`array_agg`构造与`FunctionElement.filter()`结合时，与数组索引操作符结合时无法产生正确的运算符优先级的问题。

    参考：[#4760](https://www.sqlalchemy.org/trac/ticket/4760)

+   **[sql] [错误]**

    修复了一个不太可能的问题，即对于联合和其他`_selectable.CompoundSelect`对象的“对应列”例程可能在某些重叠列情况下返回错误的列，从而在使用集合操作时可能影响一些 ORM 操作，如果底层的`select()`构造先前在其他类似类型的例程中使用过，则由于未清除缓存值而可能受��影响。

    参考：[#4747](https://www.sqlalchemy.org/trac/ticket/4747)

### postgresql

+   **[postgresql] [用例]**

    增加了对 PostgreSQL 分区表上索引的反射支持，这是从 PostgreSQL 11 版本开始添加的功能。

    参考：[#4771](https://www.sqlalchemy.org/trac/ticket/4771)

+   **[postgresql] [用例]**

    通过将`array`对象嵌套在另一个对象中，为多维 Postgresql 数组文字提供支持。多维数组类型会被自动检测。

    另请参阅

    `array`

    参考：[#4756](https://www.sqlalchemy.org/trac/ticket/4756)

### mysql

+   **[mysql] [错误]**

    修复了当`nullable=True`时，为`TIMESTAMP`数据类型渲染“NULL”特殊逻辑无法工作的错误，如果列的数据类型是`TypeDecorator`或`Variant`。现在的逻辑确保它向下展开到原始的`TIMESTAMP`，以便在请求时正确呈现此特殊情况的 NULL 关键字。

    参考：[#4743](https://www.sqlalchemy.org/trac/ticket/4743)

+   **[mysql] [错误]**

    增强了 MySQL/MariaDB 版本字符串解析，以适应奇异的 MariaDB 版本字符串，其中“MariaDB”一词嵌入在其他字母数字字符中，例如“MariaDBV1”。这种检测对于正确适应已分为 MySQL 和 MariaDB 的 API 功能至关重要，例如“transaction_isolation”系统变量。

    参考：[#4624](https://www.sqlalchemy.org/trac/ticket/4624)

### sqlite

+   **[sqlite] [usecase]**

    为 SQLite 添加了对复合（元组）IN 运算符的支持，通过在此后端渲染 VALUES 关键字。由于其他后端（如 DB2）已知使用相同的语法，因此在基本编译器中使用一个方言级标志`tuple_in_values`启用了该语法。该更改还包括对 SQLite 中使用“in_()”在元组值和空集之间时的“空 IN 元组”表达式的支持。

    参考：[#4766](https://www.sqlalchemy.org/trac/ticket/4766)

### mssql

+   **[mssql] [bug]**

    确保用于反映索引和视图定义的查询将字符串参数明确转换为 NVARCHAR，因为许多 SQL Server 驱动程序经常将字符串值，特别是具有非 ASCII 字符或较大字符串值的值，视为 TEXT，这通常与 SQL Server 的信息模式表中的 VARCHAR 字符不正确地进行比较。这些 CAST 操作已经在反射查询中对 SQL Server `information_schema.`表进行了处理，但在针对`sys.`表的另外三个查询中缺失。

    参考：[#4745](https://www.sqlalchemy.org/trac/ticket/4745)

## 1.3.5

发布日期：2019 年 6 月 17 日

### orm

+   **[orm] [bug]**

    修复了一系列关于超过两层深度的联合表继承的相关错误，以及对主键值的修改，其中这些主键列也以外键关系相互链接，这在联合表继承中是很典型的。在三级继承层次结构中的中间表现在只有在主键值发生变化且`passive_updates=False`（例如，外键约束不被执行）时才会进行更新；而在之前会被跳过；同样，在`passive_updates=True`（例如，ON UPDATE CASCADE 生效）的情况下，第三级表将不会收到更新语句，这与之前的情况不同，因为 CASCADE 已经修改了它。在一个相关问题中，与联合继承层次结构中的中间表的主键相关联的关系在父对象的主键被修改时也将正确地更新其外键列，即使该父对象是链接父类的子类，而在之前这些类不会被计算。

    参考：[#4723](https://www.sqlalchemy.org/trac/ticket/4723)

+   **[orm] [bug]**

    修复了`Mapper.all_orm_descriptors`访问器返回一个条目给`Mapper`本身在声明式`__mapper__`键下的 bug，当这不是一个描述符时。现在，所有`InspectionAttr`对象上存在的`.is_attribute`标志现在被查询，这也已经被修改为对于关联代理为`True`，因为对于这个对象错误地设置为 False。

    参考：[#4729](https://www.sqlalchemy.org/trac/ticket/4729)

+   **[orm] [bug]**

    修复了`Query.join()`中的回归，其中`aliased=True`标志不会正确应用到过滤条件的情况，如果之前对同一实体进行了连接。这是因为适配器的顺序放错了。顺序已经被颠倒，以便最近的`aliased=True`调用的适配器优先，就像在 1.2 版本和更早版本中一样。这破坏了“elementtree”示例等其他内容。

    参考：[#4704](https://www.sqlalchemy.org/trac/ticket/4704)

+   **[orm] [bug] [py3k]**

    用完全从 Python 3.3 中供应的版本替换了`getfullargspec()`的 Python 兼容性例程。最初，Python 在 Python 3.8 alpha 版本中对该函数发出了弃用警告。虽然这一变化被撤销了，但观察到 Python 3 对`getfullargspec()`的实现在 3.4 系列中重写为`Signature`后慢了一个数量级。虽然 Python 计划改进这种情况，但目前 SQLAlchemy 项目使用了一个简单的替代方案以避免任何未来问题。

    参考：[#4674](https://www.sqlalchemy.org/trac/ticket/4674)

+   **[orm] [bug]**

    重新设计了`AliasedClass`使用的属性机制，不再依赖于在包装类的 MRO 上调用`__getattribute__`，而是在包装类上使用 getattr()正常解析属性，然后解包/适应。这允许在映射类上使用更广泛的属性样式，包括特殊的`__getattr__()`方案；但这也使代码更简单、更具弹性。

    参考：[#4694](https://www.sqlalchemy.org/trac/ticket/4694)

### sql

+   **[sql] [bug]**

    解决了由于使用 `literal_column`()` 构造而导致的一系列引号问题。当这个构造通过子查询进行“代理”并通过与其文本匹配的标签引用时，即使使用 `quoted_name` 构造设置了 `Label` 的字符串，标签也不会应用引号规则。不对 `Label` 的文本应用引号是一个 bug，因为这个文本严格来说是一个 SQL 标识符名称而不是一个 SQL 表达式，而且字符串不应该已经嵌入引号，与 `literal_column()` 不同，后者可能会被应用到。为了帮助手动引号方案，维护了一个非标记的 `literal_column()` 在子查询的外部传播的现有行为，尽管不清楚是否可以为这样的构造生成有效的 SQL。

    参考：[#4730](https://www.sqlalchemy.org/trac/ticket/4730)

### postgresql

+   **[postgresql] [usecase]**

    在为 PostgreSQL 反射索引时，添加了对列排序标志的支持，包括 ASC、DESC、NULLSFIRST、NULLSLAST。同时也将这一功能添加到了通用的反射系统中，可以在未来的版本中应用于其他方言。感谢 Eli Collins 的 Pull request。

    参考：[#4717](https://www.sqlalchemy.org/trac/ticket/4717)

+   **[postgresql] [bug]**

    修复了 PostgreSQL 方言无法正确反映没有成员的 ENUM 数据类型的 bug，调用 `get_enums()` 返回一个带有 `None` 的列表，并在反射具有此类数据类型的列时引发 TypeError。现在检查返回一个空列表。

    参考：[#4701](https://www.sqlalchemy.org/trac/ticket/4701)

### mysql

+   **[mysql] [bug]**

    修复了 MySQL ON DUPLICATE KEY UPDATE 无法将列设置为 NULL 值的 bug。感谢 Lukáš Banič 的 Pull request。

    参考：[#4715](https://www.sqlalchemy.org/trac/ticket/4715)

## 1.3.4

发布日期：2019 年 5 月 27 日

### orm

+   **[orm] [bug]**

    修复了当事件监听器通过 `AttributeEvents.propagate` 标志向子类传播时，`AttributeEvents.active_history` 标志未设置的问题。这个 bug 在整个 `AttributeEvents` 系统的使用期间都存在。

    参考：[#4695](https://www.sqlalchemy.org/trac/ticket/4695)

+   **[orm] [bug]**

    修复了新关联代理系统仍未代理混合属性的回归，当它们使用 `@hybrid_property.expression` 装饰器返回替代 SQL 表达式，或者当混合属性返回任意 `PropComparator` 时，会在表达式级别进行代理。这涉及进一步泛化用于检测在 `QueryableAttribute` 级别代理对象类型的启发式方法，以更好地检测描述符最终服务于映射类还是列表达式。

    参考：[#4690](https://www.sqlalchemy.org/trac/ticket/4690)

+   **[orm] [bug]**

    应用了映射器“配置互斥锁”来防止在动态模块导入方案仍在为相关类配置映射器的过程中使用映射器时可能发生的竞争。这并不能防止所有可能的竞争条件，比如如果并发导入尚未遇到相关类，但它在 SQLAlchemy 声明过程中尽可能多地防范了可能发生的情况。

    参考：[#4686](https://www.sqlalchemy.org/trac/ticket/4686)

+   **[orm] [bug]**

    现在对将瞬态对象与 `Session.merge()` 合并到会话中时，如果该对象在 `Session` 中已经是瞬态的情况发出警告。这会对通常会导致对象被双重插入的情况发出警告。

    参考：[#4647](https://www.sqlalchemy.org/trac/ticket/4647)

+   **[orm] [bug]**

    修复了在与映射实例中处于未获取状态且持久化为 NULL 的属性进行比较时，新关系 m2o 比较逻辑首次引入的回归，当属性没有显式默认值时，在持久化设置中访问时需要默认为 NULL。 

    参考：[#4676](https://www.sqlalchemy.org/trac/ticket/4676)

### engine

+   **[engine] [bug] [postgresql]**

    将在方言初始化期间发生的“回滚”移动到额外的方言特定初始化步骤之后，特别是 psycopg2 方言的初始化步骤，这些步骤会在第一个新连接上无意中保留事务状态，这可能会干扰一些需要确保没有启动事务的 psycopg2 特定 API。感谢 Matthew Wilkes 的拉取请求。

    参考：[#4663](https://www.sqlalchemy.org/trac/ticket/4663)

### sql

+   **[sql] [bug]**

    修复了`GenericFunction`类意外注册自己为命名函数的问题。感谢 Adrien Berchet 提供的拉取请求。

    参考：[#4653](https://www.sqlalchemy.org/trac/ticket/4653)

+   **[sql] [bug]**

    修复了布尔列的双重否定不会重置“NOT”运算符的问题。

    参考：[#4618](https://www.sqlalchemy.org/trac/ticket/4618)

+   **[sql] [bug]**

    `GenericFunction`命名空间正在迁移，以便函数名称以不区分大小写的方式查找，因为 SQL 函数不会因区分大小写的差异而发生冲突，用户定义的函数或存储过程也不会发生这种情况。现在使用`GenericFunction`声明的函数查找使用不区分大小写的方案，但支持一个弃用案例，允许存在两个或更多具有不同大小写的相同名称的`GenericFunction`对象，这将导致对该特定名称进行区分大小写的查找，同时在函数注��时发出警告。感谢 Adrien Berchet 在这个复杂功能上的大量工作。

    参考：[#4569](https://www.sqlalchemy.org/trac/ticket/4569)

### postgresql

+   **[postgresql] [bug] [orm]**

    修复了“匹配行数”警告即使 dialect 报告“supports_sane_multi_rowcount=False”也会发出的问题，例如 psycogp2 使用`use_batch_mode=True`等情况。

    参考：[#4661](https://www.sqlalchemy.org/trac/ticket/4661)

### mysql

+   **[mysql] [bug]**

    增加了对 DROP CHECK 约束的支持，MySQL 8.0.16 需要删除 CHECK 约束；MariaDB 支持普通的 DROP CONSTRAINT。该逻辑通过检查 MariaDB 版本字符串来区分这两种语法的区别。Alembic 迁移已经解决了这个问题，通过在 MySQL / MariaDB CHECK 约束上实现自己的 DROP，但这个改变直接在 Core 中实现，以便供一般使用。感谢 Hannes Hansen 提供的拉取请求。

    参考：[#4650](https://www.sqlalchemy.org/trac/ticket/4650)

### mssql

+   **[mssql] [feature]**

    增加了对 SQL Server 过滤索引的支持，通过`mssql_where`参数，其工作方式类似于 PostgreSQL 方言中的`postgresql_where`索引函数。

    参见

    过滤索引

    参考：[#4657](https://www.sqlalchemy.org/trac/ticket/4657)

+   **[mssql] [bug]**

    为 pymssql 添加了错误代码 20047 到“is_disconnect”。感谢 Jon Schuff 提供的拉取请求。

    参考文献：[#4680](https://www.sqlalchemy.org/trac/ticket/4680)

### misc

+   **[misc] [bug]**

    从 MANIFEST.in 中移除了多余的“sqla_nose.py”符号，该符号会创建一个不希望出现的警告消息。

    参考文献：[#4625](https://www.sqlalchemy.org/trac/ticket/4625)

## 1.3.3

发布日期：2019 年 4 月 15 日

### orm

+   **[orm] [bug]**

    修复了 1.3 版本中新的“模糊 FROMs”查询逻辑中的回归问题，该问题在 Query.join() handles ambiguity in deciding the “left” side more explicitly 中引入，当一个`Query`显式地将一个实体放在 FROM 子句中，并且还使用`Query.join()`将其与之连接时，如果该实体在额外的连接中使用，稍后将会导致“模糊 FROM”错误，因为该实体在`Query`的“from”列表中出现两次。修复此问题的方法是将独立实体合并到它已经是一部分的连接中，这与在渲染 SELECT 语句时最终发生的方式相同。

    参考文献：[#4584](https://www.sqlalchemy.org/trac/ticket/4584)

+   **[orm] [bug]**

    调整了`Query.filter_by()`方法，不再对多个条件内部调用`and()`，而是将其作为一系列条件传递给`Query.filter()`，而不是一个单一条件。这样做允许`Query.filter_by()`推迟到`Query.filter()`处理可变数量的子句，包括列表为空的情况。在这种情况下，`Query`对象将不会有`.whereclause`，这使得后续的“无 whereclause”方法如`Query.select_from()`表现一致。

    参考文献：[#4606](https://www.sqlalchemy.org/trac/ticket/4606)

### postgresql

+   **[postgresql] [bug]**

    修复了从版本 1.3.2 发布以来由 [#4562](https://www.sqlalchemy.org/trac/ticket/4562) 引起的回归问题，其中一个仅包含查询字符串而没有主机名的 URL，例如用于指定包含连接信息的服务文件，将不再正确传播到 psycopg2。[#4562](https://www.sqlalchemy.org/trac/ticket/4562) 中的更改已经调整以进一步适应 psycopg2 的确切要求，即如果存在任何连接参数，则不再需要“dsn”参数，因此在这种情况下，仅传递查询字符串参数。

    参考：[#4601](https://www.sqlalchemy.org/trac/ticket/4601)

### mssql

+   **[mssql] [bug]**

    修复了 SQL Server 方言中的问题，如果 ORDER BY 表达式中存在一个绑定参数，最终在 SQL Server 版本的语句中不会被渲染，那么这些参数仍然会成为执行参数的一部分，导致 DBAPI 级别的错误。感谢 Matt Lewellyn 提交的拉取请求。

    参考：[#4587](https://www.sqlalchemy.org/trac/ticket/4587)

### 杂项

+   **[bug] [pool]**

    修复了由于弃用 `Pool` 的 `use_threadlocal` 标志导致的行为回归，`SingletonThreadPool` 不再使用此选项，这会导致在事务的上下文中多次使用相同的 `Engine` 连接或隐式执行时发生“回滚”逻辑，从而取消事务。虽然这不是推荐的引擎和连接工作方式，但当使用 `SingletonThreadPool` 时，事务应该保持打开状态，无论在同一线程中对相同引擎做了什么。`use_threadlocal` 标志仍然被弃用，但 `SingletonThreadPool` 现在实现了自己版本的相同逻辑。

    参考：[#4585](https://www.sqlalchemy.org/trac/ticket/4585)

+   **[bug] [ext]**

    修复了一个 bug，即在`MutableList`上使用`copy.copy()`或`copy.deepcopy()`会导致列表中的项目重复，这是由于 Python 的 pickle 和 copy 在处理列表时使用`__getstate__()`和`__setstate__()`存在不一致性。为了解决这个问题，必须向`MutableList`添加一个`__reduce_ex__`方法。为了与基于`__getstate__()`的现有 pickle 保持向后兼容性，`__setstate__()`方法也保留了；测试套件断言，对旧版本类进行的 pickle 仍然可以被 pickle 模块反序列化。

    参考：[#4603](https://www.sqlalchemy.org/trac/ticket/4603)

## 1.3.2

发布日期：April 2, 2019

### orm

+   **[orm] [bug] [ext]**

    恢复了对纯 Python 描述符（例如`@property`对象）的实例级支持，与关联代理一起使用时，如果代理对象根本不在 ORM 范围内，则被归类为“模糊”，但直接被代理。对于类级别访问，基本的类级别`__get__()`现在直接返回`AmbiguousAssociationProxyInstance`，而不是引发异常，这是返回可能的最接近以前返回`AssociationProxy`本身的行为的近似。还改进了这些对象的字符串表示，以更具描述性地反映当前状态。

    参考：[#4573](https://www.sqlalchemy.org/trac/ticket/4573), [#4574](https://www.sqlalchemy.org/trac/ticket/4574)

+   **[orm] [bug]**

    修复了一个 bug，即在使用`with_polymorphic()`或其他别名构造时，当别名目标被用作在`column_property()`内部的子查询的`Select.correlate_except()`目标时，适配不正确。这需要修复子句适配机制，以正确处理出现在“correlate except”列表中的可选择项，类似于出现在“correlate”列表中的可选择项的方式。这实际上是一个持续了很长时间的相当基本的 bug，但很难遇到它。

    参考：[#4537](https://www.sqlalchemy.org/trac/ticket/4537)

+   **[orm] [bug]**

    修复了一个新的错误消息，当尝试将关系选项链接到一个未使用`PropComparator.of_type()`的 AliasedClass 时，应该引发错误消息，而不是引发`AttributeError`。请注意，在 1.3 中，不再允许从普通映射器关系创建选项路径到`AliasedClass`而不使用`PropComparator.of_type()`。

    参考：[#4566](https://www.sqlalchemy.org/trac/ticket/4566)

### sql

+   **[sql] [bug] [documentation]**

    多亏了 TypeEngine methods bind_expression, column_expression work with Variant, type-specific types，我们不再需要依赖直接子类化特定方言类型的方法，`TypeDecorator`现在可以处理所有情况。此外，上述更改使得直接子类化基本 SQLAlchemy 类型的行为稍微不太可能按预期工作，这可能会产生误导。文档已更新，使用`TypeDecorator`来处理这些示例，包括 PostgreSQL 的“ArrayOfEnum”示例数据类型，并且直接支持“直接子类化类型”的功能已被移除。

    参考：[#4580](https://www.sqlalchemy.org/trac/ticket/4580)

### postgresql

+   **[postgresql] [feature]**

    为 psycopg2 方言添加了支持无参数连接 URL 的功能，这意味着 URL 可以作为`"postgresql+psycopg2://"`传递给`create_engine()`，而不需要额外的参数来指示传递给 libpq 的空 DSN，这表示连接到“localhost”而不提供用户名、密码或数据库。感谢 Julian Mehnle 提供的拉取请求。

    参考：[#4562](https://www.sqlalchemy.org/trac/ticket/4562)

+   **[postgresql] [bug]**

    修改了`Select.with_for_update.of`参数，如果传递了一个 join 或其他组合的可选择对象，则将从中过滤出各个`Table`对象，允许将 join()对象传递给参数，这在使用 ORM 时通常会发生。感谢 Raymond Lu 提供的拉取请求。

    参考：[#4550](https://www.sqlalchemy.org/trac/ticket/4550)

## 1.3.1

发布日期：2019 年 3 月 9 日

### orm

+   **[orm] [bug] [ext]**

    修复了一个回归问题，即关联代理链接到同义词将不再起作用，无论是在实例级别还是在类级别。

    参考资料：[#4522](https://www.sqlalchemy.org/trac/ticket/4522)

### mssql

+   **[mssql] [错误]**

    在将隔离级别更改为 SNAPSHOT 后发出了 commit()，因为 pyodbc 和 pymssql 都会打开一个隐式事务，该事务会阻止后续 SQL 在当前事务中发出。

    此更改还**回溯**到：1.2.19

    参考资料：[#4536](https://www.sqlalchemy.org/trac/ticket/4536)

+   **[mssql] [错误]**

    修复了 SQL Server 反射中的回归问题，原因是[#4393](https://www.sqlalchemy.org/trac/ticket/4393)中从`Float`数据类型中删除了开放式`**kw`导致反射此类型失败，因为传递了“scale”参数。

    参考资料：[#4525](https://www.sqlalchemy.org/trac/ticket/4525)

## 1.3.0

发布日期：2019 年 3 月 4 日

### orm

+   **[orm] [功能]**

    `Query.get()`方法现在可以接受一个属性键和值的字典作为指示要加载的主键值的手段；对于复合主键特别有用。拉取请求由 Sanjana S. 提供。

    参考资料：[#4316](https://www.sqlalchemy.org/trac/ticket/4316)

+   **[orm] [功能]**

    现在可以将 SQL 表达式分配给 ORM 刷新的主键属性，方法与普通属性描述的方式相同，如将 SQL 插入/更新表达式嵌入到刷新中，其中表达式将被评估，然后使用 RETURNING 返回给 ORM，或者在 pysqlite 的情况下，使用 cursor.lastrowid 属性工作。需要支持 RETURNING 的数据库（例如 Postgresql、Oracle、SQL Server）或 pysqlite。

    参考资料：[#3133](https://www.sqlalchemy.org/trac/ticket/3133)

### 引擎

+   **[引擎] [功能]**

    更改了在`StatementError`字符串化时的格式。现在，每个错误细节都会被分成多个新行，而不是在单行上间隔开。此外，SQL 表示现在会将 SQL 语句字符串化，而不是使用`repr()`，因此换行符会按原样呈现。拉取请求由 Nate Clark 提供。

    另请参阅

    更改了 StatementError 的格式（换行和 %s）

    参考资料：[#4500](https://www.sqlalchemy.org/trac/ticket/4500)

### sql

+   **[sql] [错误]**

    `Alias`类及其相关子类`CTE`、`Lateral`和`TableSample`已经重新设计，用户不再能直接构造这些对象。这些构造需要使用独立的构造函数或与可选择绑定的方法来实例化新对象。

    参考：[#4509](https://www.sqlalchemy.org/trac/ticket/4509)

### 模式

+   **[模式] [特性]**

    添加了新参数`Table.resolve_fks`和`MetaData.reflect.resolve_fks`，当设置为 False 时，将禁用在`ForeignKey`对象中遇到的相关表的自动反射，这既可以减少省略表的 SQL 开销，也可以避免由于数据库特定原因无法反射的表。即使两个`Table`对象存在于同一个`MetaData`集合中，这两个表的反射仍然可以相互引用，即使这两个表的反射是分开进行的。

    参考：[#4517](https://www.sqlalchemy.org/trac/ticket/4517)

## 1.3.0b3

发布日期：2019 年 2 月 8 日

### orm

+   **[orm] [错误]**

    改进了`with_polymorphic()`与加载器选项一起使用的行为，特别是通配符操作以及`load_only()`。多态对象将更准确地定位，以便实体上的列级选项能够正确生效。这个问题是在[#4468](https://www.sqlalchemy.org/trac/ticket/4468)中修复的同类问题的延续。

    参考：[#4469](https://www.sqlalchemy.org/trac/ticket/4469)

### orm 声明式

+   **[orm] [声明式] [错误]**

    添加了一些帮助异常，当基于`AbstractConcreteBase`、`DeferredReflection`或`AutoMap`的映射在映射准备好使用之前使用时，将调用这些异常，这些异常包含关于类的描述性信息，而不是掉入其他信息较少的故障模式。

    引用：[#4470](https://www.sqlalchemy.org/trac/ticket/4470)

### sql

+   **[sql] [bug]**

    完全删除了直接作为`select()`或`Query`对象组件传递的字符串的行为，自动强制将其转换为`text()`构造；现在发出的警告是一个`ArgumentError`或在`order_by()` / `group_by()`的情况下是`CompileError`。自版本 1.0 以来一直发出警告，但其存在仍然引发了对此行为可能被误用的担忧。

    请注意，order_by() / group_by()的公共 CVE 已发布，此提交已解决：CVE-2019-7164 CVE-2019-7548

    另请参阅

    删除字符串 SQL 片段强制转换为 text()

    引用：[#4481](https://www.sqlalchemy.org/trac/ticket/4481)

+   **[sql] [bug]**

    在编译时将引用应用于`Function`名称，如果它们包含非法字符，例如空格或标点符号，则通常但不一定由`sqlalchemy.sql.expression.func`构造生成的名称。但是，名称与以前一样是不区分大小写的，这意味着如果名称包含大写字母或混合大小写字符，仅此并不会触发引用。目前为了向后兼容性，不区分大小写仍然保持不变。

    引用：[#4467](https://www.sqlalchemy.org/trac/ticket/4467)

+   **[sql] [bug]**

    添加了对接受为纯字符串的关键 DDL 短语的“SQL 短语验证”，包括`ForeignKeyConstraint.on_delete`、`ForeignKeyConstraint.on_update`、`ExcludeConstraint.using`、`ForeignKeyConstraint.initially`等，用于期望一系列 SQL 关键字的地方。任何非空格字符表明该短语需要引用的情况将引发`CompileError`。此更改与作为[#4481](https://www.sqlalchemy.org/trac/ticket/4481)一部分提交的一系列更改相关。

    参考：[#4481](https://www.sqlalchemy.org/trac/ticket/4481)

### postgresql

+   **[postgresql] [错误]**

    修复了使用大写名称作为索引类型（例如 GIST、BTREE 等）或 EXCLUDE 约束时将其视为需要引用的标识符，而不是按原样呈现的问题。新行为将这些类型转换为小写，并确保它们只包含有效的 SQL 字符。

    参考：[#4473](https://www.sqlalchemy.org/trac/ticket/4473)

### 测试

+   **[测试] [更改]**

    测试系统已删除对 Nose 的支持，Nose 已多年未维护，并在 Python 3 下产生警告。测试套件目前标准化为 Pytest。感谢 Parth Shandilya 的拉取请求。

    参考：[#4460](https://www.sqlalchemy.org/trac/ticket/4460)

### 杂项

+   **[错误] [扩展]**

    当使用关联代理与集合或字典时，实现了更全面的赋值操作（例如“批量替换”）。修复了创建多余代理对象以替换旧对象的问题，这会导致过多的事件和 SQL，并且在唯一约束的情况下会导致刷新失败。

    另请参阅

    为集合、字典实现了批量替换与 AssociationProxy

    参考：[#2642](https://www.sqlalchemy.org/trac/ticket/2642)

## 1.3.0b2

发布日期：2019 年 1 月 25 日

### 通用

+   **[通用] [更改]**

    在整个库中进行了大规模的更改，确保所有已被标记为弃用或遗留的对象、参数和行为在调用时都会发出`DeprecationWarning`警告。由于 Python 3 解释器现在默认显示弃用警告，以及基于 tox 和 pytest 等工具的现代测试套件通常会显示弃用警告，这一变化应该使得更容易注意到哪些 API 功能已经过时。这一变化的主要原因是，长期以来已被弃用但仍然在实际应用中使用的功能最终可以在不久的将来被移除；其中最大的例子是自版本 0.7 以来就已被弃用但仍然存在于库中的`SessionExtension`和`MapperExtension`类，以及一些其他的预事件扩展钩子。另一个是，还将弃用几个长期存在的重要行为，包括线程本地引擎策略、convert_unicode 标志和非主要映射器。

    另请参阅

    对所有已弃用元素发出弃用警告；添加新的弃用

    参考：[#4393](https://www.sqlalchemy.org/trac/ticket/4393)

### orm

+   **[orm] [功能]**

    实现了一个新功能，即`AliasedClass`构造现在可以用作`relationship()`的目标。这样就不再需要“非主要映射器”的概念，因为`AliasedClass`更容易配置，并自动继承映射类的所有关系，同时保留加载器选项正常工作的能力。

    另请参阅

    与 AliasedClass 的关系取代了非主要映射器的需求

    参考：[#4423](https://www.sqlalchemy.org/trac/ticket/4423)

+   **[orm] [功能]**

    添加了新的`MapperEvents.before_mapper_configured()`事件。该事件与其他“配置”阶段的映射器事件相辅相成，提供了一个每个`Mapper`在其配置步骤之前接收的事件，并且还可以用于阻止或延迟特定`Mapper`对象的配置，使用新的返回值`interfaces.EXT_SKIP`。请参阅文档链接以获取示例。

    另请参阅

    `MapperEvents.before_mapper_configured()`

    参考：[#4397](https://www.sqlalchemy.org/trac/ticket/4397)

+   **[orm] [change]**

    添加了一个新函数`close_all_sessions()`，它接管了`Session.close_all()`方法的任务，后者现在已被弃用，因为这会让人误解为一个类方法。感谢 Augustin Trancart 的拉取请求。

    参考：[#4412](https://www.sqlalchemy.org/trac/ticket/4412)

+   **[orm] [bug]**

    修复了长期存在的问题，即重复的集合成员会导致反向引用在删除其中一个重复项时删除成员与其父对象之间的关联，就像在一条语句中交换两个对象的副作用一样。

    另请参见

    在删除操作期间，对于集合重复项的多对一反向引用检查

    参考：[#1103](https://www.sqlalchemy.org/trac/ticket/1103)

+   **[orm] [bug]**

    扩展了首次作为[#3287](https://www.sqlalchemy.org/trac/ticket/3287)的一部分进行的修复，其中针对使用通配符的子类的加载器选项将扩展自身以包括将通配符应用于超类属性的情况，例如在表达式中`Load(SomeSubClass).load_only('foo')`。`SomeSubClass`的父类的列也将被排除，就像使用未绑定选项`load_only('foo')`一样。

    参考：[#4373](https://www.sqlalchemy.org/trac/ticket/4373)

+   **[orm] [bug]**

    改进了 ORM 在加载器选项遍历领域发出的错误消息。这包括早期检测到不匹配的加载器策略以及更清晰地解释为什么这些策略不匹配。

    参考：[#4433](https://www.sqlalchemy.org/trac/ticket/4433)

+   **[orm] [bug]**

    在`collection.remove()`方法中，现在在移除项目之前调用“remove”集合事件，这与大多数其他形式的集合项目移除行为一致（例如`__delitem__`、`__setitem__`下的替换）。对于`pop()`方法，删除事件仍然在操作之后触发。

+   **[orm] [bug] [engine]**

    为 Core 和 ORM 添加了执行选项的访问器，通过`Query.get_execution_options()`、`Connection.get_execution_options()`、`Engine.get_execution_options()`和`Executable.get_execution_options()`。PR 由 Daniel Lister 提供。

    参考：[#4464](https://www.sqlalchemy.org/trac/ticket/4464)

+   **[orm] [bug]**

    修复了与[#3423](https://www.sqlalchemy.org/trac/ticket/3423)相关的关联代理问题，导致使用自定义`PropComparator`对象与混合属性（例如在`dictlike-polymorphic`示例中演示的属性）在关联代理中无法正常工作。在[#3423](https://www.sqlalchemy.org/trac/ticket/3423)中添加的严格性已经放宽，并添加了额外的逻辑以适应关联代理链接到自定义混合的情况。

    参考：[#4446](https://www.sqlalchemy.org/trac/ticket/4446)

+   **[orm] [bug]**

    实现了`.get_history()`方法，这也意味着对于`synonym()`属性也有`AttributeState.history`的可用性。以前，尝试通过同义词访问属性历史会引发`AttributeError`。

    参考：[#3777](https://www.sqlalchemy.org/trac/ticket/3777)

### orm 声明式

+   **[bug] [orm declarative]**

    在`ColumnProperty`中添加了一个`__clause_element__()`方法，可以使在声明映射类中使用未完全声明的列或延迟属性在约束或其他基于列的场景中稍微更友好，尽管这仍无法在开放式表达式中工作；如果收到`TypeError`，建议调用`ColumnProperty.expression`属性。

    参考：[#4372](https://www.sqlalchemy.org/trac/ticket/4372)

### engine

+   **[engine] [feature]**

    添加了公共访问器`QueuePool.timeout()`，返回`QueuePool`对象的配置超时时间。感谢 Irina Delamare 的拉取请求。

    参考：[#3689](https://www.sqlalchemy.org/trac/ticket/3689)

+   **[engine] [change]**

    自大约版本 0.2 以来一直是 SQLAlchemy 的传统特性的“threadlocal” 引擎策略现在已被弃用，以及 `Pool` 的 `Pool.threadlocal` 参数，在大多数现代用例中没有任何效果。

    另请参阅

    “threadlocal” 引擎策略已弃用

    参考：[#4393](https://www.sqlalchemy.org/trac/ticket/4393)

### sql

+   **[sql] [feature]**

    修改了 `AnsiFunction` 类，这是类似 `CURRENT_TIMESTAMP` 这样的常见 SQL 函数的基类，以接受位置参数，就像常规的临时函数一样。这适用于许多特定后端上这些函数接受参数（如“分数秒”精度等）的情况���如果函数使用参数创建，它会呈现括号和参数。如果没有参数，则编译器生成非括号形式。

    参考：[#4386](https://www.sqlalchemy.org/trac/ticket/4386)

+   **[sql] [change]**

    `create_engine.convert_unicode` 和 `String.convert_unicode` 参数已被弃用。这些参数是在大多数 Python DBAPIs 几乎不支持 Python Unicode 对象时构建的，而 SQLAlchemy 需要以高效的方式在整个系统中在 Unicode 和字节字符串之间传递数据和 SQL 字符串的非常复杂的任务。由于 Python 3，DBAPIs 被迫适应了 Unicode-aware APIs，今天所有由 SQLAlchemy 支持的 DBAPIs 都原生支持 Unicode，包括在 Python 2 上，这使得这个长期存在且非常复杂的功能最终被（大部分）移除。当然，在一些 Python 2 的边缘情况下，SQLAlchemy 仍然需要处理 Unicode，但这些都是自动处理的；在现代使用中，用户不应该需要与这些标志进行交互。

    另请参阅

    已弃用 convert_unicode 参数

    参考：[#4393](https://www.sqlalchemy.org/trac/ticket/4393)

### mssql

+   **[mssql] [bug]**

    `Unicode` 和 `UnicodeText` 数据类型的 `literal_processor` 现在在文本字符串表达式前面渲染一个 `N` 字符，这是 SQL Server 要求在 SQL 表达式中呈现 Unicode 字符值时的必需操作。

    参考：[#4442](https://www.sqlalchemy.org/trac/ticket/4442)

### 杂项

+   **[bug] [ext]**

    修复了 1.3.0b1 中由[#3423](https://www.sqlalchemy.org/trac/ticket/3423)引起的回归，其中访问仅存在于多态子类上的属性的关联代理对象会引发`AttributeError`，尽管实际被访问的实例是该子类的实例。

    参考：[#4401](https://www.sqlalchemy.org/trac/ticket/4401)

## 1.3.0b1

发布日期：2018 年 11 月 16 日

### orm

+   **[orm] [feature]**

    添加了新功能`Query.only_return_tuples()`。导致`Query`对象无条件返回键入的元组对象，即使查询针对单个实体。感谢 Eric Atkin 的贡献。

    此更改也已**回溯**至：1.2.5

+   **[orm] [feature]**

    在`Session.bulk_save_objects()`方法中添加了新标志`Session.bulk_save_objects.preserve_order`，默认值为 True。当设置为 False 时，给定的映射将按对象类型分组为插入和更新，以便更好地将常见操作批量处理在一起。感谢 Alessandro Cucci 的贡献。

+   **[orm] [feature]**

    “selectin”加载策略现在在简单的一对多加载情况下省略了 JOIN，而是仅从相关表中加载，依靠相关表的外键列来匹配父表中的主键。可以通过将`relationship.omit_join`标志设置为 False 来禁用此优化。非常感谢 Jayson Reis 的努力。

    另请参阅

    selectin loading no longer uses JOIN for simple one-to-many

    参考：[#4340](https://www.sqlalchemy.org/trac/ticket/4340)

+   **[orm] [feature]**

    在`InstanceState`类中添加了`.info`字典，该类是从调用映射对象的`inspect()`方法返回的对象。

    另请参阅

    info dictionary added to InstanceState

    参考：[#4257](https://www.sqlalchemy.org/trac/ticket/4257)

+   **[orm] [bug]**

    修复了在与`Query.join()`以及`Query.select_entity_from()`结合使用`Lateral`构造时，不会将子句适应到连接的右侧的错误。 “lateral”引入了连接右侧可以相关的用例。以前，未考虑适应此子句。请注意，在 1.2 版本中，由于[#4304](https://www.sqlalchemy.org/trac/ticket/4304)，由`Query.subquery()`引入的可选择性仍未适应；可选择性需要由`select()`函数生成以成为“lateral”连接的右侧。

    此更改也**回溯**到：1.2.12

    参考：[#4334](https://www.sqlalchemy.org/trac/ticket/4334)

+   **[orm] [bug]**

    修复了关于`passive_deletes=”all”`的问题，即对象的外键属性在从其父集合中移除后仍保持其值。以前，工作单元会将其设置为`NULL`，即使`passive_deletes`指示不应修改它。

    另请参阅

    `passive_deletes=’all’`将保留从集合中移除的对象的 FK 不变

    参考：[#3844](https://www.sqlalchemy.org/trac/ticket/3844)

+   **[orm] [bug]**

    改进了与关系绑定的多对一对象表达式的行为，使得在相关对象上检索列值现在对于对象从其父`Session`中分离，即使属性已过期也是弹性的。在`InstanceState`中使用了新功能来记忆特定列属性在其过期之前的最后已知值，以便在对象被分离和过期时表达式仍然可以评估。使用现代属性状态功能改进了错误条件，以根据需要生成更具体的消息。

    另请参阅

    改进了多对一查询表达式的行为

    参考：[#4359](https://www.sqlalchemy.org/trac/ticket/4359)

+   **[orm] [bug] [mysql] [postgresql]**

    现在 ORM 在某些情况下会在子查询中加倍使用“FOR UPDATE”子句，与联接式急加载一起使用，因为观察到 MySQL 不会锁定子查询中的行。这意味着查询会带有两个 FOR UPDATE 子句；请注意，在某些后端（如 Oracle）上，由于不必要，子查询中的 FOR UPDATE 子句会被静默忽略。此外，在主要与 PostgreSQL 一起使用的“OF”子句的情况下，仅在使用时才在内部子查询中呈现 FOR UPDATE，以便可选择地将可选择的对象定位到 SELECT 语句中的表。

    另请参阅

    FOR UPDATE 子句在联接急加载子查询中以及外部呈现

    参考：[#4246](https://www.sqlalchemy.org/trac/ticket/4246)

+   **[orm] [bug]**

    重构 `Query.join()` 以进一步澄清结构化联接的各个组件。此重构增加了 `Query.join()` 的能力，以确定在 FROM 列表中有多个元素或查询针对多个实体时，联接的最适当“左”侧。如果有多个 FROM/entity 匹配，将引发错误，要求指定 ON 子句以解决模棱两可。特别是针对我们在 [#4363](https://www.sqlalchemy.org/trac/ticket/4363) 中看到的回归，但也具有一般用途。现在 `Query.join()` 中的代码路径更易于遵循，并且在操作的较早阶段更具体地决定错误情况。

    另请参阅

    Query.join() 更明确地处理决定“左”侧的模棱两可

    参考：[#4365](https://www.sqlalchemy.org/trac/ticket/4365)

+   **[orm] [bug]**

    修复了`Query`中的一个长期存在的问题，即标量子查询（例如由`Query.exists()`、`Query.as_scalar()`和其他从`Query.statement`派生的查询生成的子查询）在被用于需要实体适配的新`Query`时，例如当查询被转换为联合查询或 from_self()等时，将无法正确适配。此更改从由`Query.statement`访问器生成的`select()`对象中移除了“无适配”注释。

    参考���[#4304](https://www.sqlalchemy.org/trac/ticket/4304)

+   **[orm] [bug]**

    在 Python 3 下，在 ORM 刷新期间，当主键值在 Python 中不可排序时（例如没有`__lt__()`方法的`Enum`），会重新引发一个信息性异常；通常在这种情况下，Python 3 会引发`TypeError`。刷新过程在 Python 中按主键对持久对象进行排序，因此这些值必须是可排序的。

    参考：[#4232](https://www.sqlalchemy.org/trac/ticket/4232)

+   **[orm] [bug]**

    移除了`MappedCollection`类使用的集合转换器。此转换器仅用于断言传入的字典键与其对应对象的键匹配，并且仅在批量设置操作期间使用。该转换器可能会干扰自定义验证器或想要进一步转换传入值的`AttributeEvents.bulk_replace()`监听器。当传入的键与值不匹配时，此转换器将引发的`TypeError`已被移除；在批量赋值期间，传入值将被键入其生成的键，而不是显式存在于字典中的键。

    总的来说，@converter 已被作为[#3896](https://www.sqlalchemy.org/trac/ticket/3896)的一部分添加的`AttributeEvents.bulk_replace()`事件处理程序所取代。

    参考：[#3604](https://www.sqlalchemy.org/trac/ticket/3604)

+   **[orm] [bug]**

    添加了新行为，当检索到多对一关系的“旧”值时，延迟加载时会跳过由于`lazy="raise"`或分离会话错误而引发的异常。

    另请参见

    多对一替换不会对“raiseload”或“old”对象引发异常

    参考：[#4353](https://www.sqlalchemy.org/trac/ticket/4353)

+   **[orm] [bug]**

    在 ORM 中长期存在的一个疏漏，即一个多对一关系的 `__delete__` 方法是非功能性的，例如对于 `del a.b` 这样的操作。现在已实现，并且等效于将属性设置为 `None`。

    另请参阅

    实现了“del”用于 ORM 属性

    参考：[#4354](https://www.sqlalchemy.org/trac/ticket/4354)

### orm 声明性

+   **[orm] [声明性] [bug]**

    修复了当声明性不会更新 `Mapper` 的状态，例如在映射器属性集已经被调用并且被备忘录化后添加或删除其他属性时的 bug。另外，如果从当前已映射的类中删除了完全映射的属性（例如列、关系等），则现在会引发 `NotImplementedError`，因为如果属性已被删除，则映射器将无法正确运行。

    参考：[#4133](https://www.sqlalchemy.org/trac/ticket/4133)

### engine

+   **[engine] [特性]**

    添加了对 `QueuePool` 的新的“后进先出”模式，通常通过将标志 `create_engine.pool_use_lifo` 设置为 True 来启用。“后进先出”模式意味着刚刚检入的连接将首先再次检出，允许在池仅部分利用时从服务器端清理多余的连接。拉请求由 Taem Park 提供。

    另请参阅

    队列池的新的后进先出策略

### sql

+   **[sql] [特性]**

    重构了 `SQLCompiler` 来公开一个类似于 `SQLCompiler.order_by_clause()` 和 `SQLCompiler.limit_clause()` 方法的 `SQLCompiler.group_by_clause()` 方法，这些方法可以被方言覆盖以自定义 GROUP BY 如何呈现。拉请求由 Samuel Chou 提供。

    此更改也被**回溯**到：1.2.13

+   **[sql] [特性]**

    在“字符串 SQL”系统中添加了 `Sequence`，当在没有方言的情况下字符串化包含“序列下一个值”表达式的语句时，将渲染一个有意义的字符串表达式（`"<next sequence value: my_sequence>"`），而不是引发编译错误。

    参考：[#4144](https://www.sqlalchemy.org/trac/ticket/4144)

+   **[sql] [特性]**

    添加了新的命名约定标记`column_0N_name`，`column_0_N_name`等，这些标记将为序列中特定约束引用的所有列的名称/键/标签提供渲染。为了适应这种命名约定的长度，SQL 编译器的自动截断功能现在也适用于约束名称，这将为约束创建一个缩短的、确定性生成的名称，该名称将适用于目标后端而不超过该后端的字符限制。

    此更改还修复了另外两个问题。一个是`column_0_key`标记尽管已记录，但却不可用，另一个是如果这两个值不同，`referred_column_0_name`标记会无意中渲染`.key`而不是`.name`的列。

    另请参阅

    新的多列命名约定标记，长名称截断

    参考：[#3989](https://www.sqlalchemy.org/trac/ticket/3989)

+   **[sql] [功能]**

    为“expanding IN”绑定参数功能添加了新逻辑，如果给定的列表为空，将生成特定于不同后端的特殊“空集”表达式，从而允许 IN 表达式完全动态，包括空 IN 表达式。

    另请参阅

    扩展 IN 功能现在支持空列表

    参考：[#4271](https://www.sqlalchemy.org/trac/ticket/4271)

+   **[sql] [功能]**

    现在支持 Python 内置的`dir()`用于 SQLAlchemy 的“properties”对象，例如 Core 列集合（例如`.c`）、`mapper.attrs`等。也允许 iPython 自动完成工作。感谢 Uwe Korn 的拉取请求。

+   **[sql] [功能]**

    添加了新功能`FunctionElement.as_comparison()`，允许 SQL 函数充当可以在 ORM 内部工作的二进制比较操作。

    另请参阅

    SQL 函数的二进制比较解释

    参考：[#3831](https://www.sqlalchemy.org/trac/ticket/3831)

+   **[sql] [错误]**

    添加了基于“like”的运算符作为“比较”运算符，包括`ColumnOperators.startswith()` `ColumnOperators.endswith()` `ColumnOperators.ilike()` `ColumnOperators.notilike()` 等等，以便所有这些运算符都可以成为 ORM“primaryjoin”条件的基础。

    参考：[#4302](https://www.sqlalchemy.org/trac/ticket/4302)

+   **[sql] [错误]**

    修复了`TypeEngine.bind_expression()`和`TypeEngine.column_expression()`方法的问题，这些方法在目标类型是`Variant`或其他`TypeDecorator`的一部分时无法正常工作。另外，SQL 编译器现在在渲染这些方法时调用方言级别的实现，以便方言现在可以为内置类型提供 SQL 级别的处理。

    另请参见

    TypeEngine 方法 bind_expression, column_expression 适用于 Variant，类型特定类型

    参考：[#3981](https://www.sqlalchemy.org/trac/ticket/3981)

### postgresql

+   **[postgresql] [特性]**

    增加了新的 PG 类型`REGCLASS`，用于将表名转换为 OID 值。感谢 Sebastian Bank 的拉取请求。

    此更改也被**回溯**到了：1.2.7

    参考：[#4160](https://www.sqlalchemy.org/trac/ticket/4160)

+   **[postgresql] [特性]**

    增加了对 PostgreSQL 分区表的基本反射支持，例如，在返回表信息的反射查询中添加了 relkind='p'。

    另请参见

    为 PostgreSQL 分区表添加了基本的反射支持

    参考：[#4237](https://www.sqlalchemy.org/trac/ticket/4237)

### mysql

+   **[mysql] [特性]**

    在 MySQL 中增加了对 CREATE FULLTEXT INDEX 的“WITH PARSER”语法的支持，使用`mysql_with_parser`关键字参数。还支持了反射，以适应 MySQL 的特殊注释格式，用于报告此选项。另外，“FULLTEXT”和“SPATIAL”索引前缀现在也反映到`mysql_prefix`索引选项中。

    参考：[#4219](https://www.sqlalchemy.org/trac/ticket/4219)

+   **[mysql] [特性]**

    增加了对 MySQL 中 ON DUPLICATE KEY UPDATE 语句中参数的排序支持，因为 MySQL UPDATE 子句中的参数顺序是有意义的，类似于参数有序更新中描述的方式。感谢 Maxim Bublis 的拉取请求。

    另请参见

    ON DUPLICATE KEY UPDATE 中参数排序的控制

+   **[mysql] [特性]**

    连接池的“预连接”特性现在在 mysqlclient、PyMySQL 和 mysql-connector-python 的情况下使用 DBAPI 连接的`ping()`方法。感谢 Maxim Bublis 的拉取请求。

    另请参见

    预先使用协议级 ping 进行预先 ping

### sqlite

+   **[sqlite] [feature]**

    通过新的 SQLite 实现支持 SQLite 的 JSON 功能，使用了`JSON`，`JSON`。该类型的名称为 `JSON`，遵循了 SQLite 文档中的示例。感谢 Ilja Everilä 提供的拉取请求。

    另请参阅

    添加对 SQLite JSON 的支持

    参考：[#3850](https://www.sqlalchemy.org/trac/ticket/3850)

+   **[sqlite] [feature]**

    实现了 SQLite 中 `ON CONFLICT` 语句在 DDL 级别的理解，例如用于主键、唯一键和 CHECK 约束以及在 `Column` 上指定的用于满足内联主键和 NOT NULL。感谢 Denis Kataev 提供的拉取请求。

    另请参阅

    添加了对 SQLite 中 ON CONFLICT 的约束的支持

    参考：[#4360](https://www.sqlalchemy.org/trac/ticket/4360)

### mssql

+   **[mssql] [feature]**

    在 SQL Server pyodbc 方言中添加了 `fast_executemany=True` 参数，启用了 pyodbc 的新性能特性，当使用 Microsoft ODBC 驱动程序时可以使用相同的名称。

    另请参阅

    添加对 pyodbc fast_executemany 的支持

    参考：[#4158](https://www.sqlalchemy.org/trac/ticket/4158)

+   **[mssql] [bug]**

    废弃了在 SQL Server 中使用 `Sequence` 来影响 IDENTITY 值的“开始”和“增量”的用法，而是使用新参数 `mssql_identity_start` 和 `mssql_identity_increment` 直接设置这些参数。在未来的版本中，`Sequence` 将用于生成真正的 `CREATE SEQUENCE` DDL 与 SQL Server。

    另请参阅

    新参数以影响 IDENTITY 的开始和增量，使用 Sequence 废弃

    参考：[#4362](https://www.sqlalchemy.org/trac/ticket/4362)

### oracle

+   **[oracle] [feature]**

    添加了一个新的事件，目前仅由 cx_Oracle 方言使用，`DialectEvents.setiputsizes()`。该事件将一个 `BindParameter` 对象的字典传递给特定于 DBAPI 的类型对象，这些对象在转换为参数名称后将传递给 cx_Oracle 的 `cursor.setinputsizes()` 方法。这允许查看 setinputsizes 过程以及修改传递给此方法的数据类型的行为。

    另请参阅

    使用 setinputsizes 实现对 cx_Oracle 数据绑定性能的精细控制

    此更改也 **回溯到**：1.2.9

    参考：[#4290](https://www.sqlalchemy.org/trac/ticket/4290)

+   **[oracle] [bug]**

    更新了可以发送到 cx_Oracle DBAPI 的参数，允许所有当前参数以及尚未添加的未来参数。此外，删除了在 1.2 版本中已弃用的未使用参数，并且现在我们将“threaded”默认设置为 False。

    另请参阅

    cx_Oracle 连接参数现代化，已弃用的参数已移除

    参考：[#4369](https://www.sqlalchemy.org/trac/ticket/4369)

+   **[oracle] [bug]**

    Oracle 方言将不再使用 NCHAR/NCLOB 数据类型来表示通用的 Unicode 字符串或 clob 字段，除非在`create_engine()`中传递了标志`use_nchar_for_unicode=True` - 这包括 CREATE TABLE 行为以及绑定参数的`setinputsizes()`。在读取方面，在 Python 2 下已经添加了 CHAR/VARCHAR/CLOB 结果行的自动 Unicode 转换，以匹配 Python 3 下 cx_Oracle 的行为。为了减轻 Python 2 下的性能损失，SQLAlchemy 在 Python 2 下使用非常高效（当构建了 C 扩展时）的本地 Unicode 处理程序。

    另请参阅

    通用 Unicode 取消强调的国家字符数据类型，通过选项重新启用

    参考：[#4242](https://www.sqlalchemy.org/trac/ticket/4242)

### 杂项

+   **[feature] [ext]**

    添加了新属性`Query.lazy_loaded_from`，其中填充了一个使用此`Query`来延迟加载关系的`InstanceState`。这样做的理由是它作为水平分片功能的提示，以便使用状态的标识令牌作为查询中要使用的默认标识令牌在 id_chooser()中使用。

    此更改也被**回溯**到：1.2.9

    参考：[#4243](https://www.sqlalchemy.org/trac/ticket/4243)

+   **[feature] [ext]**

    添加了新功能`BakedQuery.to_query()`，允许以干净的方式在另一个`BakedQuery`内使用一个`BakedQuery`作为子查询，而无需显式引用`Session`。

    参考文献：[#4318](https://www.sqlalchemy.org/trac/ticket/4318)

+   **[功能] [扩展]**

    当目标属性是普通列时，`AssociationProxy`现在具有标准列比较操作，例如`ColumnOperators.like()`和`ColumnOperators.startswith()` - 与目标表连接的 EXISTS 表达式通常呈现，但列表达式然后在 EXISTS 的 WHERE 条件中使用。请注意，这会更改关联代理上的`.contains()`方法的行为，使其在基于列的属性上使用`ColumnOperators.contains()`。

    另请参阅

    关联代理现在为基于列的目标提供标准列操作符

    参考文献：[#4351](https://www.sqlalchemy.org/trac/ticket/4351)

+   **[功能] [扩展]**

    在水平分片扩展中，对`ShardedQuery`类添加了对批量`Query.update()`和`Query.delete()`的支持。这还为批量更新/删除方法`Query._execute_crud()`添加了额外的扩展钩子。

    另请参阅

    水平分片扩展支持批量更新和删除方法

    参考文献：[#4196](https://www.sqlalchemy.org/trac/ticket/4196)

+   **[错误] [扩展]**

    重新设计了`AssociationProxy`，以在单独的对象中存储特定于父类的状态，这样一个`AssociationProxy`可以为多个父类提供服务，这是继承所固有的，而且不会有任何模糊性。添加了一个新方法`AssociationProxy.for_class()`，允许检查特定于类的状态。

    另请参阅

    关联代理在每个类上存储特定于类的状态

    参考：[#3423](https://www.sqlalchemy.org/trac/ticket/3423)

+   **[错误] [扩展]**

    关联代理集合的长期行为已经改变，现在代理将保持对父对象的强引用，只要代理集合本身也在内存中，就可以消除“陈旧的关联代理”错误。这一变更是基于实验性质进行的，以查看是否会出现任何导致副作用的用例。

    另请参阅

    新功能和改进 - 核心

    参考：[#4268](https://www.sqlalchemy.org/trac/ticket/4268)

+   **[错误] [扩展]**

    修复了关联代理与标量对象的解除关联的多个问题。现在`del`可以正常工作，另外还添加了一个新标志`AssociationProxy.cascade_scalar_deletes`，当设置为 True 时，表示将标量属性设置为`None`或通过`del`删除也会将源关联设置为`None`。

    另请参阅

    关联代理有新的 cascade_scalar_deletes 标志

    参考：[#4308](https://www.sqlalchemy.org/trac/ticket/4308)

## 1.3.25

无发布日期

### orm

+   **[orm] [错误]**

    修复了在`Session.bulk_save_objects()`与持久对象一起使用时的问题，其中会无法跟踪主键映射的主键，其中主键列名与属性名不同。

    参考：[#6392](https://www.sqlalchemy.org/trac/ticket/6392)

### 模式

+   **[模式] [错误]**

    如果未在至少通过参数传递`Table.name`和`Table.metadata`实例化`Table`对象，则会引发一个信息性错误消息。以前，如果这些是作为关键字参数传递的，则该对象将无声地初始化失败。

    参考：[#6135](https://www.sqlalchemy.org/trac/ticket/6135)

### postgresql

+   **[postgresql] [错误] [回归]**

    修复了由[#6023](https://www.sqlalchemy.org/trac/ticket/6023)引起的回归，当使用 psycopg2 时，PostgreSQL cast 运算符应用于`ARRAY`中的元素时，如果数据类型也嵌入到`Variant`适配器的实例中，则会失败使用正确的类型。

    此外，修复了在使用`Variant(ARRAY(some_schema_type))`时发出正确的 CREATE TYPE 的支持。

    参考：[#6182](https://www.sqlalchemy.org/trac/ticket/6182)

### mysql

+   **[mysql] [错误] [mariadb]**

    修复了 MariaDB 10.6 系列的问题，包括 mariadb-connector Python 驱动程序（仅在 SQLAlchemy 1.4 上受支持）以及自动使用的本地 10.6 客户端库中的向后不兼容更改，mysqlclient DBAPI（适用于 1.3 和 1.4）。当编码状态为“utf8”时，这些客户端库现在会报告“utf8mb3”编码符号，导致 MySQL 方言中的查找和编码错误，该方言不期望此符号。更新了 MySQL 基础库以适应报告此 utf8mb3 符号以及测试套件的更改。感谢 Georg Richter 的支持。

    参考：[#7115](https://www.sqlalchemy.org/trac/ticket/7115), [#7136](https://www.sqlalchemy.org/trac/ticket/7136)

### sqlite

+   **[sqlite] [错误]**

    添加有关传递给 pysqlcipher 的 url 的与加密相关的 pragma 的注意事项。

    参考：[#6589](https://www.sqlalchemy.org/trac/ticket/6589)

### orm

+   **[orm] [错误]**

    修复了在与持久化对象一起使用时的`Session.bulk_save_objects()`中的问题，这些对象将无法跟踪列名与属性名不同的映射的主键。

    参考：[#6392](https://www.sqlalchemy.org/trac/ticket/6392)

### 模式

+   **[模式] [错误]**

    如果`Table`对象在实例化时没有传递至少`Table.name`和`Table.metadata`参数，则现在会引发一个信息性错误消息。以前，如果这些参数作为关键字参数传递，对象将悄悄地无法正确初始化。

    参考：[#6135](https://www.sqlalchemy.org/trac/ticket/6135)

### postgresql

+   **[postgresql] [bug] [regression]**

    修复了由[#6023](https://www.sqlalchemy.org/trac/ticket/6023)引起的回归，当使用 psycopg2 时，将 PostgreSQL cast 运算符应用于`ARRAY`中的元素时，如果数据类型也嵌入在`Variant`适配器的实例中，将无法使用正确的类型。

    此外，修复了在使用`Variant(ARRAY(some_schema_type))`时发出正确的 CREATE TYPE 的支持。

    参考：[#6182](https://www.sqlalchemy.org/trac/ticket/6182)

### mysql

+   **[mysql] [bug] [mariadb]**

    修复了适应 MariaDB 10.6 系列的问题，包括 mariadb-connector Python 驱动程序（仅支持 SQLAlchemy 1.4）和自动使用的 mysqlclient DBAPI 中的本机 10.6 客户端库中的不兼容更改。当编码状态为“utf8”时，这些客户端库现在报告“utf8mb3”编码符号，导致 MySQL 方言中的查找和编码错误，该方言不期望此符号。更新了 MySQL 基本库以适应此 utf8mb3 符号的报告以及测试套件。感谢 Georg Richter 的支持。

    参考：[#7115](https://www.sqlalchemy.org/trac/ticket/7115), [#7136](https://www.sqlalchemy.org/trac/ticket/7136)

### sqlite

+   **[sqlite] [bug]**

    在传递给 pysqlcipher 的 url 中添加有关加密相关的 pragma 的注释。

    参考：[#6589](https://www.sqlalchemy.org/trac/ticket/6589)

## 1.3.24

发布日期：2021 年 3 月 30 日

### orm

+   **[orm] [bug]**

    删除了非常古老的警告，指出 passive_deletes 不适用于多对一关系。虽然在许多情况下，在多对一关系上放置此参数可能不是预期的操作，但在某些情况下，可能希望在此类关系之后禁止删除级联。

    参考：[#5983](https://www.sqlalchemy.org/trac/ticket/5983)

+   **[orm] [bug]**

    修复了连接两个表的过程可能失败的问题，如果其中一个表具有无关的、无法解决的外键约束，这将在连接过程中引发 `NoReferenceError`，尽管可以绕过此异常以允许连接完成。在处理中测试异常重要性的逻辑会对构造做出可能失败的假设。

    参考：[#5952](https://www.sqlalchemy.org/trac/ticket/5952)

+   **[orm] [bug]**

    修复了当父对象已经加载，并且后续查询覆盖时，`MutableComposite` 构造可能处于无效状态的问题，这是由于复合属性的刷新处理程序用新对象替换对象而不受可变扩展处理。

    参考：[#6001](https://www.sqlalchemy.org/trac/ticket/6001)

### engine

+   **[engine] [bug]**

    修复了“schema_translate_map”功能在直接执行 `DefaultGenerator` 对象（例如序列）时未被考虑的问题，其中包括当禁用 implicit_returning 时“预执行”它们以生成主键值的情况。 

    参考：[#5929](https://www.sqlalchemy.org/trac/ticket/5929)

### schema

+   **[schema] [bug]**

    修复了首次引入的错误，即 [#2892](https://www.sqlalchemy.org/trac/ticket/2892)、[#2919](https://www.sqlalchemy.org/trac/ticket/2919) 和 [#3832](https://www.sqlalchemy.org/trac/ticket/3832) 的某种组合，其中对于 `TypeDecorator` 的附加事件会与“impl”类重复，如果“impl”也是 `SchemaType`。真实情况是，任何 `TypeDecorator` 与 `Enum` 或 `Boolean` 相结合时，当设置 `create_constraint=True` 标志时，会获得重复的 `CheckConstraint`。

    参考：[#6152](https://www.sqlalchemy.org/trac/ticket/6152)

+   **[schema] [bug] [sqlite]**

    修复了由`Boolean`或`Enum`生成的 CHECK 约束在第一次编译后无法正确呈现命名约定的问题，这是由于约束名称中的状态意外更改导致的。此问题首次出现在修复问题＃3067 时引入的 0.9 版本中，修复了当时采取的方法，该方法似乎比实际需要的更复杂。

    参考：[#6007](https://www.sqlalchemy.org/trac/ticket/6007)

+   **[schema] [bug]**

    修复/实现了支持使用列名/键等作为约定一部分的主键约束命名约定。特别是，这包括`Table`自动关联的`PrimaryKeyConstraint`对象将在向表添加新的主键`Column`对象然后添加到约束时更新其名称。现在已经适应了与此约束构建过程相关的内部故障模式，包括没有列存在，没有名称存在或存在空名称。

    参考：[#5919](https://www.sqlalchemy.org/trac/ticket/5919)

+   **[schema] [bug]**

    调整了为`Sequence`对象发出 DROP 语句的逻辑，使得所有`Sequence`对象在所有表之后被删除，即使给定的`Sequence`仅与`Table`对象相关，而不直接与整体`MetaData`对象相关。该用例支持同一`Sequence`同时与多个`Table`相关联。

    参考：[#6071](https://www.sqlalchemy.org/trac/ticket/6071)

### postgresql

+   **[postgresql] [bug]**

    修复了在某些条件下使用`aggregate_order_by`会返回 ARRAY(NullType)的问题，干扰了结果对象正确返回数据的能力。

    参考：[#5989](https://www.sqlalchemy.org/trac/ticket/5989)

+   **[postgresql] [bug] [reflection]**

    修复了 PostgreSQL 反射中的问题，表达“NOT NULL”的列将取代对应域的可空性。

    参考：[#6161](https://www.sqlalchemy.org/trac/ticket/6161)

+   **[postgresql] [bug] [types]**

    调整了 psycopg2 方言，使其对包含 ARRAY 元素的绑定参数发出明确的 PostgreSQL 样式转换。这允许在数组中正确使用全部范围的数据类型。asyncpg 方言已在最终语句中生成了这些内部转换。这还包括对数组切片更新的支持以及 PostgreSQL 特有的 `ARRAY.contains()` 方法。

    参考：[#6023](https://www.sqlalchemy.org/trac/ticket/6023)

### mssql

+   **[mssql] [bug] [reflection]**

    修复了有关 SQL Server 反射的问题，针对较旧的 SQL Server 2005 版本，调用 sp_columns 如果不加 EXEC 关键字则无法正确执行。这种方法在当前的 1.4 系列中不再使用。

    参考：[#5921](https://www.sqlalchemy.org/trac/ticket/5921)

### orm

+   **[orm] [bug]**

    删除了一个非常古老的警告，指出 passive_deletes 不适用于多对一关系。虽然在许多情况下在多对一关系上放置此参数可能不是预期的行为，但有一些用例希望在这样的关系之后不允许删除级联。

    参考：[#5983](https://www.sqlalchemy.org/trac/ticket/5983)

+   **[orm] [bug]**

    修复了如果两个表中的一个表具有无关、不可解析的外键约束，则连接两个表的过程可能会失败的问题。此外，如果在连接过程中引发了 `NoReferenceError`，尽管可以绕过此错误完成连接，但测试过程中测试异常的逻辑将对结构进行假设，这些假设将失败。

    参考：[#5952](https://www.sqlalchemy.org/trac/ticket/5952)

+   **[orm] [bug]**

    修复了当父对象已加载，然后被后续查询覆盖时，`MutableComposite` 结构可能处于无效状态的问题，因为复合属性的刷新处理程序会用不受可变扩展控制的新对象替换对象。

    参考：[#6001](https://www.sqlalchemy.org/trac/ticket/6001)

### 引擎

+   **[engine] [bug]**

    修复了一个 bug，在直接执行 `DefaultGenerator` 对象（比如序列）时，“schema_translate_map” 特性未被考虑进去，其中包括当隐式返回被禁用时“预执行”以生成主键值的情况。

    参考：[#5929](https://www.sqlalchemy.org/trac/ticket/5929)

### 模式

+   **[模式] [错误]**

    修复了首次引入的错误，可能是由[#2892](https://www.sqlalchemy.org/trac/ticket/2892)、[#2919](https://www.sqlalchemy.org/trac/ticket/2919)和[#3832](https://www.sqlalchemy.org/trac/ticket/3832)的某种组合引起的，其中`TypeDecorator`的附加事件会对“impl”类重复，如果“impl”也是`SchemaType`。实际情况是，任何`TypeDecorator`与`Enum`或`Boolean`相对时，当设置`create_constraint=True`标志时，将会获得一个重复的`CheckConstraint`。

    参考：[#6152](https://www.sqlalchemy.org/trac/ticket/6152)

+   **[模式] [错误] [sqlite]**

    修复了由`Boolean`或`Enum`生成的 CHECK 约束在第一次编译后无法正确呈现命名约定的问题，这是由于约束名称中的状态意外更改导致的。此问题首次在 0.9 中引入，修复了当时为解决问题＃3067 而采取的方法，该方法似乎比实际需要的更复杂。

    参考：[#6007](https://www.sqlalchemy.org/trac/ticket/6007)

+   **[模式] [错误]**

    修复/实现了支持使用列名/键等作为约定的主键约束命名规范。特别是，这包括`Table`自动关联的`PrimaryKeyConstraint`对象将在向表添加新的主键`Column`对象后更新其名称，然后更新约束。现在已经适应了与此约束构建过程相关的内部故障模式，包括没有列存在、没有名称存在或空名称存在。

    参考：[#5919](https://www.sqlalchemy.org/trac/ticket/5919)

+   **[模式] [错误]**

    调整了发出`Sequence`对象的 DROP 语句的逻辑，以便在删除多个表时，所有`Sequence`对象都在所有表之后被删除，即使给定的`Sequence`仅与一个`Table`对象相关联，而不直接与整体`MetaData`对象相关联。该用例支持同一`Sequence`同时与多个`Table`相关联。

    参考：[#6071](https://www.sqlalchemy.org/trac/ticket/6071)

### postgresql

+   **[postgresql] [bug]**

    修复了在某些条件下使用`aggregate_order_by`会返回 ARRAY(NullType)的问题，干扰了结果对象正确返回数据的能力。

    参考：[#5989](https://www.sqlalchemy.org/trac/ticket/5989)

+   **[postgresql] [bug] [reflection]**

    修复了在 PostgreSQL 反射中，表达“NOT NULL”的列将取代相应域的可空性的问题。

    参考：[#6161](https://www.sqlalchemy.org/trac/ticket/6161)

+   **[postgresql] [bug] [types]**

    调整了 psycopg2 方言以发出包含 ARRAY 元素的绑定参数的显式 PostgreSQL 风格转换。这允许各种数据类型在数组中正确运行。asyncpg 方言已在最终语句中生成了这些内部转换。这还包括对数组切片更新以及 PostgreSQL 特定的`ARRAY.contains()`方法的支持。

    参考：[#6023](https://www.sqlalchemy.org/trac/ticket/6023)

### mssql

+   **[mssql] [bug] [reflection]**

    修复了有关旧版 SQL Server 2005 版本的 SQL Server 反射的问题，调用 sp_columns 时，如果没有使用 EXEC 关键字作为前缀，则无法正确进行。这种方法在当前的 1.4 系列中没有使用。

    参考：[#5921](https://www.sqlalchemy.org/trac/ticket/5921)

## 1.3.23

发布日期：2021 年 2 月 1 日

### sql

+   **[sql] [bug]**

    修复了在 `TypeDecorator` 类型上使用 `TypeEngine.with_variant()` 方法时未考虑到正在使用的特定方言映射的 bug，这是由于 `TypeDecorator` 中的规则尝试检查 `TypeDecorator` 实例链而导致的。

    参考：[#5816](https://www.sqlalchemy.org/trac/ticket/5816)

### postgresql

+   **[postgresql] [bug]**

    仅适用于 SQLAlchemy 1.3，setup.py 将 pg8000 固定在低于 1.16.6 的版本。版本 1.16.6 及以上由 SQLAlchemy 1.4 支持。感谢 Giuseppe Lumia 的拉取请求。

    参考：[#5645](https://www.sqlalchemy.org/trac/ticket/5645)

+   **[postgresql] [bug]**

    修复了在使用`Table.to_metadata()`（在 1.3 中称为`Table.tometadata()`）与 PostgreSQL 的 `ExcludeConstraint` 结合使用时，使用临时列表达式的情况下无法正确复制的问题。

    参考：[#5850](https://www.sqlalchemy.org/trac/ticket/5850)

### mysql

+   **[mysql] [usecase]**

    在 MySQL >= (8, 0, 17) 和 MariaDb >= (10, 4, 5) 中现在支持转换为 `FLOAT`。

    参考：[#5808](https://www.sqlalchemy.org/trac/ticket/5808)

+   **[mysql] [bug] [reflection]**

    修复了 MySQL 服务器默认反射对带有否定符号的数值失败的 bug。

    参考：[#5860](https://www.sqlalchemy.org/trac/ticket/5860)

+   **[mysql] [bug]**

    修复了 MySQL 方言中长期存在的 bug，其中标识符长度的最大限制为 255 对于所有类型的约束名称都太长，不仅仅是索引，所有这些都有一个大小限制为 64。由于元数据命名约定可能在此区域创建过长的名称，因此将限制应用于 DDL 编译器内的标识符生成器。

    参考：[#5898](https://www.sqlalchemy.org/trac/ticket/5898)

+   **[mysql] [bug]**

    修复了由于 PyMySQL 1.0 发布而引起的弃用警告，包括“db”和“passwd”参数的弃用警告，现在替换为“database”和“password”。

    参考：[#5821](https://www.sqlalchemy.org/trac/ticket/5821)

+   **[mysql] [bug]**

    修复了 SQLAlchemy 1.3.20 中由于修复[#5462](https://www.sqlalchemy.org/trac/ticket/5462)而引起的 Oracle 方言中的回归，该修复为 MySQL 功能表达式在索引中添加双括号，后端所需，这无意中扩展到包括任意`text()`表达式以及 Alembic 的内部文本组件，这些对于 Alembic 来说是需要的，用于不暗示双括号的任意索引表达式。检查已经缩小，仅包括直接的二元/一元/功能表达式。

    参考：[#5800](https://www.sqlalchemy.org/trac/ticket/5800)

### oracle

+   **[oracle] [bug]**

    修复了 SQLAlchemy 1.3.11 中由[#4894](https://www.sqlalchemy.org/trac/ticket/4894)引入的 Oracle 方言中的回归，其中在 UPDATE 的 RETURNING 中使用 SQL 表达式会因为在任意 SQL 表达式不是列时检查“server_default”而无法编译。

    参考：[#5813](https://www.sqlalchemy.org/trac/ticket/5813)

+   **[oracle] [bug]**

    修复了 Oracle 方言中的 bug，通过`Insert.returning()`检索 CLOB/BLOB 列会失败，因为当返回时需要读取 LOB 值；此外，修复了在 Python 2 下通过 RETURNING 检索 Unicode 值的支持。

    参考：[#5812](https://www.sqlalchemy.org/trac/ticket/5812)

### misc

+   **[bug] [ext]**

    修复了在尝试为可选择的`.c`集合生成“key”时有时会调用的字符串化失败的问题，如果列是使用`sqlalchemy.ext.compiler`扩展的未标记的自定义 SQL 构造，并且没有提供默认编译形式；虽然这似乎是一个不寻常的情况，但在一些 ORM 场景中可能会调用它，比如当表达式与连接的急加载一起在“order by”中使用时。问题在于缺乏默认编译器函数会引发`CompileError`而不是`UnsupportedCompilationError`。

    参考：[#5836](https://www.sqlalchemy.org/trac/ticket/5836)

### sql

+   **[sql] [bug]**

    修复了在`TypeDecorator`类型上使用`TypeEngine.with_variant()`方法时存在的 bug，由于`TypeDecorator`中的一条规则，它试图检查`TypeDecorator`实例链，而未考虑到正在使用的特定方言映射。

    参考：[#5816](https://www.sqlalchemy.org/trac/ticket/5816)

### postgresql

+   **[postgresql] [bug]**

    仅适用于 SQLAlchemy 1.3，setup.py 将 pg8000 固定在低于 1.16.6 的版本。版本 1.16.6 及以上受 SQLAlchemy 1.4 支持。感谢 Giuseppe Lumia 的拉取请求。

    参考：[#5645](https://www.sqlalchemy.org/trac/ticket/5645)

+   **[postgresql] [bug]**

    修复了在与使用临时列表达式的 PostgreSQL `ExcludeConstraint`一起使用`Table.to_metadata()`（在 1.3 中称为`Table.tometadata()`）时无法正确复制的问题。

    参考：[#5850](https://www.sqlalchemy.org/trac/ticket/5850)

### mysql

+   **[mysql] [usecase]**

    现在 MySQL >= (8, 0, 17)和 MariaDb >= (10, 4, 5)中支持转换为`FLOAT`。

    参考：[#5808](https://www.sqlalchemy.org/trac/ticket/5808)

+   **[mysql] [bug] [reflection]**

    修复了 MySQL 服务器默认反射对带有否定符号的数值失败的 bug。

    参考：[#5860](https://www.sqlalchemy.org/trac/ticket/5860)

+   **[mysql] [bug]**

    修复了 MySQL 方言中存在已久的 bug，其中 255 的最大标识符长度对于所有类型的约束名称都太长，而不仅仅是索引，所有这些都有 64 的大小限制。由于元数据命名约定可能在此区域创建过长的名称，因此将限制应用于 DDL 编译器内的标识符生成器。

    参考：[#5898](https://www.sqlalchemy.org/trac/ticket/5898)

+   **[mysql] [bug]**

    修复了由于 PyMySQL 1.0 发布而引起的弃用警告，包括“db”和“passwd”参数的弃用警告，现已替换为“database”和“password”。

    参考：[#5821](https://www.sqlalchemy.org/trac/ticket/5821)

+   **[mysql] [bug]**

    由于修复了[#5462](https://www.sqlalchemy.org/trac/ticket/5462)导致的 SQLAlchemy 1.3.20 中的回归问题，该修复为 MySQL 功能表达式在索引中添加了双括号，后端需要这样做，这无意中扩展到了包括任意`text()`表达式以及 Alembic 的内部文本组件，这些对于 Alembic 来说是必需的，用于不暗示双括号的任意索引表达式。检查已经缩小，只包括直接的二元/一元/功能表达式。

    引用：[#5800](https://www.sqlalchemy.org/trac/ticket/5800)

### oracle

+   **[oracle] [bug]**

    修复了在 SQLAlchemy 1.3.11 中由[#4894](https://www.sqlalchemy.org/trac/ticket/4894)引入的 Oracle 方言中的回归问题，其中在 UPDATE 的 RETURNING 中使用 SQL 表达式会因为在任意 SQL 表达式不是列时检查“server_default”而无法编译。

    引用：[#5813](https://www.sqlalchemy.org/trac/ticket/5813)

+   **[oracle] [bug]**

    修复了 Oracle 方言中的一个错误，即通过`Insert.returning()`检索 CLOB/BLOB 列会失败，因为当返回时需要读取 LOB 值；此外，修复了在 Python 2 下通过 RETURNING 检索 Unicode 值的支持。

    引用：[#5812](https://www.sqlalchemy.org/trac/ticket/5812)

### 杂项

+   **[bug] [ext]**

    修复了在尝试为可选择的`.c`集合生成“key”时有时会调用字符串化的问题，如果列是使用`sqlalchemy.ext.compiler`扩展的未标记的自定义 SQL 构造，并且没有提供默认编译形式，则会失败；虽然这似乎是一个不寻常的情况，但在一些 ORM 场景中可能会调用它，比如在“order by”中与联接的急加载一起使用表达式时。问题在于缺少默认编译器函数会引发`CompileError`而不是`UnsupportedCompilationError`。

    引用：[#5836](https://www.sqlalchemy.org/trac/ticket/5836)

## 1.3.22

发布日期：2020 年 12 月 18 日

### oracle

+   **[oracle] [bug]**

    修复了由于[#5755](https://www.sqlalchemy.org/trac/ticket/5755)引起的回归，该回归实现了对 Oracle 的隔离级别支持。据报道，许多 Oracle 帐户实际上没有权限查询`v$transaction`视图，因此，当在数据库连接失败时，此功能已被修改为优雅地回退，其中方言将假定“READ COMMITTED”是默认的隔离级别，就像在 SQLAlchemy 1.3.21 之前的情况一样。但是，现在必须明确使用`Connection.get_isolation_level()`方法引发异常，因为具有此限制的 Oracle 数据库明确禁止用户读取当前隔离级别。

    参考：[#5784](https://www.sqlalchemy.org/trac/ticket/5784)

### oracle

+   **[oracle] [bug]**

    修复了由于[#5755](https://www.sqlalchemy.org/trac/ticket/5755)引起的回归，该回归实现了对 Oracle 的隔离级别支持。据报道，许多 Oracle 帐户实际上没有权限查询`v$transaction`视图，因此，当在数据库连接失败时，此功能已被修改为优雅地回退，其中方言将假定“READ COMMITTED”是默认的隔离级别，就像在 SQLAlchemy 1.3.21 之前的情况一样。但是，现在必须明确使用`Connection.get_isolation_level()`方法引发异常，因为具有此限制的 Oracle 数据库明确禁止用户读取当前隔离级别。

    参考：[#5784](https://www.sqlalchemy.org/trac/ticket/5784)

## 1.3.21

发布日期：2020 年 12 月 17 日

### orm

+   **[orm] [bug]**

    对于传递给`relationship.secondary`的映射类或字符串映射类名的情况，添加了全面的检查和信息丰富的错误消息。这是一个极其常见的错误，需要清晰的消息。

    另外，针对`relationship.secondary`参数，添加了一个新的规则到类注册表解析中，如果一个映射类及其表的字符串名称相同，则优先考虑`Table`解析该参数。在所有其他情况下，如果类和表共享相同的名称，则仍然优先考虑类。

    参考：[#5774](https://www.sqlalchemy.org/trac/ticket/5774)

+   **[orm] [bug]**

    修复了`Query.update()`中的一个 bug，即当`_ormsession.Session`中的对象已经过期时，通过“evaluate”同步策略刷新时会不必要地单独进行 SELECT 操作。

    参考：[#5664](https://www.sqlalchemy.org/trac/ticket/5664)

+   **[orm] [bug]**

    修复了涉及 ORM 事件的`restore_load_context`选项（例如`InstanceEvents.load()`）的 bug，使得标志不会传递给在首次建立事件处理程序之后映射的子类。

    参考：[#5737](https://www.sqlalchemy.org/trac/ticket/5737)

### sql

+   **[sql] [bug]**

    如果多次调用类似`Insert.returning()`的`returning()`方法，会发出警告，因为目前还不支持累加操作。版本 1.4 将支持此功能。此外，任何组合`Insert.returning()`和`ValuesBase.return_defaults()`方法现在会引发错误，因为这些方法是互斥的；之前的操作会悄无声息地失败。

    参考：[#5691](https://www.sqlalchemy.org/trac/ticket/5691)

+   **[sql] [bug]**

    修复了结构编译器问题，其中一些构造（如 MySQL / PostgreSQL 的“on conflict / on duplicate key”）依赖于`Compiler`对象的状态固定为它们的语句作为顶层语句，这在这些语句从不同上下文分支出来的情况下会失败，比如与 SQL 语句相关联的 DDL 构造。

    参考：[#5656](https://www.sqlalchemy.org/trac/ticket/5656)

### postgresql

+   **[postgresql] [用例]**

    为`ExcludeConstraint`对象添加了新参数`ExcludeConstraint.ops`，以支持此约束的操作符类规范。感谢 Alon Menczer 的拉取请求。

    参考：[#5604](https://www.sqlalchemy.org/trac/ticket/5604)

+   **[postgresql] [bug] [mysql]**

    修复了在 1.3.2 中引入的 PostgreSQL 方言的回归，也复制到了 1.3.18 中 MySQL 方言的功能，其中使用非 `Table` 构造（如`text()`）作为`Select.with_for_update.of`参数时，将无法在 PostgreSQL 或 MySQL 编译器中正确处理。

    参考：[#5729](https://www.sqlalchemy.org/trac/ticket/5729)

### mysql

+   **[mysql] [bug] [reflection]**

    修复了在仅包含值中包含小数点的 MariaDB 上反映服务器默认值时出现问题的问题，导致反映的表缺少任何服务器默认值。

    参考：[#5744](https://www.sqlalchemy.org/trac/ticket/5744)

+   **[mysql] [sql]**

    将缺失的关键字添加到 MySQL 方言的`RESERVED_WORDS`列表中：`action`，`level`，`mode`，`status`，`text`，`time`。感谢 Oscar Batori 的拉取请求。

    参考：[#5696](https://www.sqlalchemy.org/trac/ticket/5696)

### sqlite

+   **[sqlite] [usecase]**

    添加了`sqlite_with_rowid=False`方言关键字，以启用创建`CREATE TABLE … WITHOUT ROWID`表。感谢 Sean Anderson 的补丁。

    参考：[#5685](https://www.sqlalchemy.org/trac/ticket/5685)

### mssql

+   **[mssql] [bug]**

    修复了当同时指定`mssql-include`和`mssql_where`时，CREATE INDEX 语句呈现不正确的错误。感谢@Adiorz 的拉取请求。

    参考：[#5751](https://www.sqlalchemy.org/trac/ticket/5751)

+   **[mssql] [bug]**

    将 SQL Server 代码“01000”添加到断开连接代码列表中。

    参考：[#5646](https://www.sqlalchemy.org/trac/ticket/5646)

+   **[mssql] [reflection] [sqlite]**

    修复了复合主键列未按正确顺序报告的问题。感谢@fulpm 的补丁。

    参考：[#5661](https://www.sqlalchemy.org/trac/ticket/5661)

### oracle

+   **[oracle] [usecase]**

    实现了对 Oracle 数据库的 SERIALIZABLE 隔离级别的支持，以及对`Connection.get_isolation_level()`的真实实现。

    另请参阅

    事务隔离级别 / 自动提交

    参考：[#5755](https://www.sqlalchemy.org/trac/ticket/5755)

### orm

+   **[orm] [bug]**

    对于将映射类或字符串映射类名称传递给`relationship.secondary`的情况，添加了全面的检查和信息丰富的错误消息。这是一个极其常见的错误，需要清晰的消息。

    另外，增加了一个新规则到类注册解析中，关于 `relationship.secondary` 参数，如果映射类及其表的字符串名称相同，则在解析此参数时将优先选择 `Table`。在所有其他情况下，如果类和表共享相同的名称，则继续优先选择类。

    参考：[#5774](https://www.sqlalchemy.org/trac/ticket/5774)

+   **[orm] [bug]**

    修复了 `Query.update()` 中的错误，当 `_ormsession.Session` 中的对象已过期时，会在刷新时不必要地单独进行 SELECT 查询。

    参考：[#5664](https://www.sqlalchemy.org/trac/ticket/5664)

+   **[orm] [bug]**

    修复了关于 ORM 事件的 `restore_load_context` 选项（如 `InstanceEvents.load()`）的错误，使得标志不会被传递给在事件处理程序首次建立之后映射的子类。

    参考：[#5737](https://www.sqlalchemy.org/trac/ticket/5737)

### sql

+   **[sql] [bug]**

    当 `Insert.returning()` 等返回方法被多次调用时，会发出警告，因为目前不支持累加操作。版本 1.4 将支持此功能。此外，任何 `Insert.returning()` 和 `ValuesBase.return_defaults()` 方法的组合现在都会引发错误，因为这些方法是互斥的；以前这个操作会悄悄失败。

    参考：[#5691](https://www.sqlalchemy.org/trac/ticket/5691)

+   **[sql] [bug]**

    修复了一些结构编译器问题，例如 MySQL / PostgreSQL 的 “on conflict / on duplicate key” 会依赖于 `Compiler` 对象的状态被固定为其语句的顶级语句，这在这些语句从不同上下文分支出来时会失败，比如与 SQL 语句链接到 DDL 构造。

    参考：[#5656](https://www.sqlalchemy.org/trac/ticket/5656)

### postgresql

+   **[postgresql] [usecase]**

    向 `ExcludeConstraint` 对象添加了新参数 `ExcludeConstraint.ops`，以支持此约束的操作符类规范。感谢 Alon Menczer 的拉取请求。

    参考：[#5604](https://www.sqlalchemy.org/trac/ticket/5604)

+   **[postgresql] [bug] [mysql]**

    修复了在 1.3.2 中引入的针对 PostgreSQL 方言的回归，也在 1.3.18 中复制到了 MySQL 方言的功能中，其中使用非 `Table` 构造（如 `text()`）作为 `Select.with_for_update.of` 参数将无法在 PostgreSQL 或 MySQL 编译器中正确处理。

    参考：[#5729](https://www.sqlalchemy.org/trac/ticket/5729)

### mysql

+   **[mysql] [bug] [reflection]**

    修复了在 MariaDB 上反射仅包含值中的小数点的服务器默认值时无法正确反映的问题，导致反映的表缺乏任何服务器默认值。

    参考：[#5744](https://www.sqlalchemy.org/trac/ticket/5744)

+   **[mysql] [sql]**

    将缺少的关键字添加到 MySQL 方言的 `RESERVED_WORDS` 列表中：`action`、`level`、`mode`、`status`、`text`、`time`。感谢 Oscar Batori 的拉取请求。

    参考：[#5696](https://www.sqlalchemy.org/trac/ticket/5696)

### sqlite

+   **[sqlite] [usecase]**

    添加了 `sqlite_with_rowid=False` 方言关键字，以启用创建表为 `CREATE TABLE … WITHOUT ROWID`。感谢 Sean Anderson 的补丁。

    参考：[#5685](https://www.sqlalchemy.org/trac/ticket/5685)

### mssql

+   **[mssql] [bug]**

    修复了当同时指定 `mssql-include` 和 `mssql_where` 时，CREATE INDEX 语句呈现不正确的 bug。感谢 @Adiorz 的拉取请求。

    参考：[#5751](https://www.sqlalchemy.org/trac/ticket/5751)

+   **[mssql] [bug]**

    将 SQL Server 代码“01000”添加到断开连接代码列表中。

    参考：[#5646](https://www.sqlalchemy.org/trac/ticket/5646)

+   **[mssql] [reflection] [sqlite]**

    修复了复合主键列未按正确顺序报告的问题。感谢 @fulpm 的补丁。

    参考：[#5661](https://www.sqlalchemy.org/trac/ticket/5661)

### oracle

+   **[oracle] [usecase]**

    实现了对 Oracle 数据库的 SERIALIZABLE 隔离级别的支持，以及对 `Connection.get_isolation_level()` 的真正实现。

    另请参见

    事务隔离级别 / 自动提交

    参考：[#5755](https://www.sqlalchemy.org/trac/ticket/5755)

## 1.3.20

发布日期：2020 年 10 月 12 日

### orm

+   **[orm] [bug]**

    如果`Query.join()`的目标参数设置为未映射对象，则现在会引发带有更多详细信息的`ArgumentError`。在此更改之前，会引发一个不太详细的`AttributeError`。感谢 Ramon Williams 提交的拉取请求。

    参考：[#4428](https://www.sqlalchemy.org/trac/ticket/4428)

+   **[orm] [bug]**

    修复了针对实际上不是映射属性的字符串属性名称（例如普通的 Python 描述符）使用加载器选项会引发一个不具信息性的 AttributeError 的问题；现在会引发一个描述性错误。

    参考：[#4589](https://www.sqlalchemy.org/trac/ticket/4589)

### 引擎

+   **[engine] [bug]**

    修复了将非字符串对象发送到`SQLAlchemyError`或其子类时（某些第三方方言会出现此情况），无法正确字符串化的问题。感谢 Andrzej Bartosiński 提交的拉取请求。

    参考：[#5599](https://www.sqlalchemy.org/trac/ticket/5599)

+   **[engine] [bug]**

    修复了未在 sqlalchemy.exc 模块内使用 SQLAlchemy 的标准延迟导入系统的函数级别导入。

    参考：[#5632](https://www.sqlalchemy.org/trac/ticket/5632)

### sql

+   **[sql] [bug]**

    修复了针对`Over`构造执行`pickle.dumps()`操作会产生递归溢出的问题。

    参考：[#5644](https://www.sqlalchemy.org/trac/ticket/5644)

+   **[sql] [bug]**

    修复了在将`column()`添加到多个`table()`时未引发错误的错误。对于`Column`和`Table`对象，现在正确引发错误。当发生这种情况时，现在会引发`ArgumentError`。

    参考：[#5618](https://www.sqlalchemy.org/trac/ticket/5618)

### postgresql

+   **[postgresql] [usecase]**

    psycopg2 方言现在支持通过将主机/端口组合传递给查询字符串来支持 PostgreSQL 多主机连接。感谢 Ramon Williams 提交的拉取请求。

    另请参阅

    指定多个备用主机

    参考：[#4392](https://www.sqlalchemy.org/trac/ticket/4392)

+   **[postgresql] [bug]**

    调整了`Comparator.any()`和`Comparator.all()`方法，以实现直接的“NOT”操作进行否定，而不是否定比较运算符。

    参考：[#5518](https://www.sqlalchemy.org/trac/ticket/5518)

+   **[postgresql] [bug]**

    修复了`ENUM`类型在测试期间在发出 CREATE TYPE 或 DROP TYPE 时不会查看模式转换映射是否存在该类型的问题。此外，修复了一个问题，即如果在单个 DDL 序列中多次遇到相同的枚举，则“check”查询将重复运行，而不是依赖于缓存值。

    参考：[#5520](https://www.sqlalchemy.org/trac/ticket/5520)

### mysql

+   **[mysql] [用例]**

    调整了 MySQL 方言，以正确地将函数索引表达式括在括号中，这是 MySQL 8 所接受的。感谢 Ramon Williams 的拉取请求。

    参考：[#5462](https://www.sqlalchemy.org/trac/ticket/5462)

+   **[mysql] [更改]**

    添加了新的 MySQL 保留字：`cube`，`lateral`分别在 MySQL 8.0.1 和 8.0.14 中添加；这表示如果作为表或列标识符名称使用这些术语，它们将被引用。

    参考：[#5539](https://www.sqlalchemy.org/trac/ticket/5539)

+   **[mysql] [bug]**

    使用`with_for_update()`中的“skip_locked”关键字在 MariaDB 后端上使用时会发出警告，然后将被忽略。这是一种已弃用的行为，在 SQLAlchemy 1.4 中将会引发错误，因为请求“skip locked”的应用程序正在寻找一个在这些后端上不可用的非阻塞操作。

    参考：[#5568](https://www.sqlalchemy.org/trac/ticket/5568)

+   **[mysql] [bug]**

    修复了针对使用 MySQL 多表格式的 JOIN 的 UPDATE 语句的一个 bug，如果语句没有 WHERE 子句，那么目标表的表前缀将不会被包括在内，因为在那个特定点只有 WHERE 子句被扫描以检测“多表更新”。现在，如果目标是 JOIN，则也会被扫描，以获取最左边的表作为主表和其他条目作为额外的 FROM 条目。

    参考：[#5617](https://www.sqlalchemy.org/trac/ticket/5617)

### mssql

+   **[mssql] [bug]**

    修复了一个问题，即使用`authentication=ActiveDirectoryIntegrated`的 Azure DW 的 SQLAlchemy 连接 URI（没有用户名+密码）未以 Azure DW 实例可接受的方式构建 ODBC 连接字符串。

    参考：[#5592](https://www.sqlalchemy.org/trac/ticket/5592)

### 测试

+   **[测试] [bug]**

    修复了针对 Pytest 6.x 运行时测试套件中的不兼容性。

    参考：[#5635](https://www.sqlalchemy.org/trac/ticket/5635)

### misc

+   **[bug] [池]**

    修复了在调用`Engine.dispose()`时，以下池参数未传播到新创建的池的问题：`pre_ping`，`use_lifo`。此外，`recycle`和`reset_on_return`参数现在也传播到`AssertionPool`类。

    参考：[#5582](https://www.sqlalchemy.org/trac/ticket/5582)

+   **[bug] [associationproxy] [ext]**

    当尝试将一个关联代理元素用作要从中选择或在 SQL 函数中使用的普通列表达式时，现在会引发一个信息丰富的错误；目前不支持这种用例。

    参考：[#5541](https://www.sqlalchemy.org/trac/ticket/5541), [#5542](https://www.sqlalchemy.org/trac/ticket/5542)

### orm

+   **[orm] [bug]**

    如果`Query.join()`的目标参数设置为未映射对象，则现在会引发带有更多详细信息的`ArgumentError`。在此更改之前，会引发一个不太详细的`AttributeError`。感谢 Ramon Williams 提供的拉取请求。

    参考：[#4428](https://www.sqlalchemy.org/trac/ticket/4428)

+   **[orm] [bug]**

    修复了针对实际上不是映射属性的字符串属性名称（例如普通的 Python 描述符）使用加载器选项会引发一个不具描述性的 AttributeError 的问题；现在会引发一个描述性错误。

    参考：[#4589](https://www.sqlalchemy.org/trac/ticket/4589)

### 引擎

+   **[engine] [bug]**

    修复了将非字符串对象发送到`SQLAlchemyError`或其子类时（某些第三方方言会发生这种情况），无法正确字符串化的问题。感谢 Andrzej Bartosiński 提供的拉取请求。

    参考：[#5599](https://www.sqlalchemy.org/trac/ticket/5599)

+   **[engine] [bug]**

    修复了未在 sqlalchemy.exc 模块内使用 SQLAlchemy 标准的延迟导入系统的函数级别导入。

    参考：[#5632](https://www.sqlalchemy.org/trac/ticket/5632)

### sql

+   **[sql] [bug]**

    修复了针对`Over`构造的`pickle.dumps()`操作会导致递归溢出的问题。

    参考：[#5644](https://www.sqlalchemy.org/trac/ticket/5644)

+   **[sql] [bug]**

    修复了一个 bug，在此 bug 中，当一个 `column()` 被同时添加到多个 `table()` 时，错误未被引发。现在对于 `Column` 和 `Table` 对象，此错误已正确引发。当这种情况发生时，现在会引发 `ArgumentError`。

    参考：[#5618](https://www.sqlalchemy.org/trac/ticket/5618)

### postgresql

+   **[postgresql] [usecase]**

    psycopg2 方言现在支持 PostgreSQL 多主机连接，通过将主机/端口组合传递给查询字符串。拉取请求由拉蒙·威廉姆斯提供。

    另请参阅

    指定多个备用主机

    参考：[#4392](https://www.sqlalchemy.org/trac/ticket/4392)

+   **[postgresql] [bug]**

    调整了 `Comparator.any()` 和 `Comparator.all()` 方法，以实现直接的“NOT”操作来进行否定，而不是对比运算符的否定。

    参考：[#5518](https://www.sqlalchemy.org/trac/ticket/5518)

+   **[postgresql] [bug]**

    修复了一个问题，即当 `ENUM` 类型在发出 CREATE TYPE 或 DROP TYPE 期间不会查看模式转换映射时，问题仍然存在。另外，修复了一个问题，即如果在单个 DDL 序列中多次遇到相同的枚举，则“check”查询将重复运行，而不是依赖于缓存的值。

    参考：[#5520](https://www.sqlalchemy.org/trac/ticket/5520)

### mysql

+   **[mysql] [usecase]**

    调整了 MySQL 方言，以正确地将功能性索引表达式括在括号中，这符合 MySQL 8 的规范。拉取请求由拉蒙·威廉姆斯提供。

    参考：[#5462](https://www.sqlalchemy.org/trac/ticket/5462)

+   **[mysql] [change]**

    添加了新的 MySQL 保留字：`cube`、`lateral` 分别在 MySQL 8.0.1 和 8.0.14 中添加；这表示如果作为表或列标识符名称使用这些术语，它们将被引用。

    参考：[#5539](https://www.sqlalchemy.org/trac/ticket/5539)

+   **[mysql] [bug]**

    当在 MariaDB 后端使用 `with_for_update()` 中的 “skip_locked” 关键字时，将发出警告，然后将被忽略。这是一种已弃用的行为，在 SQLAlchemy 1.4 中将会引发错误，因为请求“跳过锁定”表示的是一种不可用于这些后端的非阻塞操作。

    参考：[#5568](https://www.sqlalchemy.org/trac/ticket/5568)

+   **[mysql] [bug]**

    修复了一个 bug，即针对使用 MySQL 多表格式的 JOIN 进行 UPDATE 语句时，如果语句没有 WHERE 子句，则目标表的表前缀将不会被包括，因为只有 WHERE 子句被扫描以检测“多表更新”在那个特定点。现在，如果目标是 JOIN，则也会被扫描，以获取最左边的表作为主表和其他条目作为额外的 FROM 条目。

    参考：[#5617](https://www.sqlalchemy.org/trac/ticket/5617)

### mssql

+   **[mssql] [bug]**

    修复了一个问题，即使用`authentication=ActiveDirectoryIntegrated`（没有用户名+密码）的 Azure DW 的 SQLAlchemy 连接 URI 未构建出可接受的 ODBC 连接字符串，无法被 Azure DW 实例接受。

    参考：[#5592](https://www.sqlalchemy.org/trac/ticket/5592)

### 测试

+   **[tests] [bug]**

    修复了在针对 Pytest 6.x 运行时测试套件中的不兼容性。

    参考：[#5635](https://www.sqlalchemy.org/trac/ticket/5635)

### 杂项

+   **[bug] [pool]**

    修复了当调用`Engine.dispose()`时，以下池参数未传播到新创建的池的问题：`pre_ping`，`use_lifo`。此外，`recycle`和`reset_on_return`参数现在也传播到`AssertionPool`类。

    参考：[#5582](https://www.sqlalchemy.org/trac/ticket/5582)

+   **[bug] [associationproxy] [ext]**

    当尝试将关联代理元素用作要从中选择或在 SQL 函数中使用的普通列表达式时，现在会引发一个信息性错误；目前不支持这种用例。

    参考：[#5541](https://www.sqlalchemy.org/trac/ticket/5541), [#5542](https://www.sqlalchemy.org/trac/ticket/5542)

## 1.3.19

发布日期：2020 年 8 月 17 日

### orm

+   **[orm] [usecase]**

    调整了`Mapper.all_orm_descriptors()`访问器的工作方式，以一种确定性的方式表示属性，假设使用 Python 3.6 或更高版本，它根据属性声明的方式维护类属性的排序顺序。然而，这种排序并不保证在所有情况下都与属性的声明顺序匹配；请参考方法文档以获取确切的方案。

    参考：[#5494](https://www.sqlalchemy.org/trac/ticket/5494)

### orm 声明

+   **[orm] [declarative] [usecase]**

    当使用`AbstractConcreteBase`和`ConcreteBase`类时，现在可以自定义虚拟列的名称，以允许具有实际命名为`type`的列的模型。拉取请求由 Jesse-Bakker 提供。

    参考：[#5513](https://www.sqlalchemy.org/trac/ticket/5513)

### sql

+   **[sql] [bug]**

    修复了一个问题，即“ORDER BY”子句渲染标签名称而不是完整表达式的问题，在某些情况下，如果表达式被括在括号分组中，将无法发生。此案例已添加到测试支持。此更改还调整了 ORM 查询中“在 DISTINCT 存在时自动添加 ORDER BY 列”的行为，1.4 中已弃用，以更准确地检测已存在的列表达式。

    参考：[#5470](https://www.sqlalchemy.org/trac/ticket/5470)

+   **[sql] [bug] [datatypes]**

    `LookupError`消息现在将向用户提供通过`Enum`约束的列的最多四个可能值。超过 11 个字符的值将被截断并替换为省略号。拉取请求由 Ramon Williams 提供。

    参考：[#4733](https://www.sqlalchemy.org/trac/ticket/4733)

+   **[sql] [bug]**

    修复了当使用`Connection.execution_options.schema_translate_map`功能时，当使用`Sequence`的`Column.server_default`参数中的`Sequence.next_value()`函数，并发出创建表 DDL 时，`Connection.execution_options.schema_translate_map`功能将不起作用的问题。

    参考：[#5500](https://www.sqlalchemy.org/trac/ticket/5500)

### postgresql

+   **[postgresql] [bug]**

    修复了各种 RANGE 比较运算符的返回类型本身将是相同的 RANGE 类型而不是 BOOLEAN 的问题，这将导致在使用定义了结果处理行为的`TypeDecorator`的情况下产生不良结果。拉取请求由 Jim Bosch 提供。

    参考：[#5476](https://www.sqlalchemy.org/trac/ticket/5476)

### mysql

+   **[mysql] [usecase]**

    MySQL 方言将为没有 FROM 子句但有 WHERE 子句的 SELECT 语句呈现 FROM DUAL。这允许像“SELECT 1 WHERE EXISTS (subquery)”这样的查询以及其他用例。

    参考：[#5481](https://www.sqlalchemy.org/trac/ticket/5481)

+   **[mysql] [bug]**

    修复了一个问题，即 CREATE TABLE 语句未正确指定 COLLATE 关键字。

    参考：[#5411](https://www.sqlalchemy.org/trac/ticket/5411)

+   **[mysql] [bug]**

    将 MariaDB 代码 1927 添加到“断开连接”代码列表中，因为最近的 MariaDB 版本显然在数据库服务器停止时使用此代码。

    参考：[#5493](https://www.sqlalchemy.org/trac/ticket/5493)

### sqlite

+   **[sqlite] [bug] [mssql] [reflection]**

    对所有包含的方言进行了一次扫描，以确保包含单引号或双引号的名称在查询系统表时得到正确转义，对于所有接受对象名称作为参数的`Inspector`方法（例如表名、视图名等）。修复了 SQLite 和 MSSQL 中存在的两个引号问题。

    参考：[#5456](https://www.sqlalchemy.org/trac/ticket/5456)

### mssql

+   **[mssql] [bug] [sql]**

    修复了一个 bug，即 mssql 方言错误地转义包含‘]’字符的对象名称。

    参考：[#5467](https://www.sqlalchemy.org/trac/ticket/5467)

### misc

+   **[usecase] [py3k]**

    向`DeclarativeMeta.__init__()`方法添加了一个`**kw`参数。这允许类支持[**PEP 487**](https://peps.python.org/pep-0487/)元类钩子`__init_subclass__`。感谢 Ewen Gillies 的拉取请求。

    参考：[##5357](https://www.sqlalchemy.org/trac/ticket/#5357)

### orm

+   **[orm] [usecase]**

    调整了`Mapper.all_orm_descriptors()`访问器的工作方式，以一种确定性的方式表示属性，假设使用 Python 3.6 或更高版本，该版本基于声明的顺序维护类属性的排序顺序。但是，这种排序不能保证在所有情况下与属性的声明顺序匹配；请参阅方法文档以获取确切的方案。

    参考：[#5494](https://www.sqlalchemy.org/trac/ticket/5494)

### orm declarative

+   **[orm] [declarative] [usecase]**

    当使用`AbstractConcreteBase`和`ConcreteBase`类时，现在可以自定义虚拟列的名称，以允许具有实际命名为`type`的列的模型。感谢 Jesse-Bakker 的拉取请求。

    参考：[#5513](https://www.sqlalchemy.org/trac/ticket/5513)

### sql

+   **[sql] [bug]**

    修复了一个问题，即“ORDER BY”子句渲染标签名称而不是完整表达式，在某些情况下，如果表达式被括在括号分组中，将无法发生。此案例已添加到测试支持中。此更改还调整了 ORM 查询的“在 DISTINCT 存在时自动添加 ORDER BY 列”的行为，在 1.4 中已弃用，以更准确地检测已存在的列表达式。

    参考：[#5470](https://www.sqlalchemy.org/trac/ticket/5470)

+   **[sql] [bug] [datatypes]**

    `LookupError`消息现在将向用户提供通过`Enum`约束的列的最多四个可能值。长度超过 11 个字符的值将被截断并替换为省略号。拉取请求由 Ramon Williams 提供。

    参考：[#4733](https://www.sqlalchemy.org/trac/ticket/4733)

+   **[sql] [bug]**

    修复了`Connection.execution_options.schema_translate_map`功能在使用`Sequence`的`Column.server_default`参数中的`Sequence.next_value()`函数时不起作用，并且创建表 DDL 被发出。

    参考：[#5500](https://www.sqlalchemy.org/trac/ticket/5500)

### postgresql

+   **[postgresql] [bug]**

    修复了各种 RANGE 比较运算符的返回类型本身将是相同的 RANGE 类型而不是 BOOLEAN 的问题，这将导致在使用定义了结果处理行为的`TypeDecorator`时产生不良��果。拉取请求由 Jim Bosch 提供。

    参考：[#5476](https://www.sqlalchemy.org/trac/ticket/5476)

### mysql

+   **[mysql] [usecase]**

    MySQL 方言将为没有 FROM 子句但有 WHERE 子句的 SELECT 语句呈现 FROM DUAL。这允许像“SELECT 1 WHERE EXISTS (subquery)”这样的查询以及其他用例。

    参考：[#5481](https://www.sqlalchemy.org/trac/ticket/5481)

+   **[mysql] [bug]**

    修复了 CREATE TABLE 语句未正确指定 COLLATE 关键字的问题。

    参考：[#5411](https://www.sqlalchemy.org/trac/ticket/5411)

+   **[mysql] [bug]**

    将 MariaDB 代码 1927 添加到“断开连接”代码列表中，因为最近的 MariaDB 版本显然在数据库服务器停止时使用此代码。

    参考：[#5493](https://www.sqlalchemy.org/trac/ticket/5493)

### sqlite

+   **[sqlite] [错误] [mssql] [反射]**

    对所有包含的方言进行了一次扫描，以确保在查询系统表时，包含单引号或双引号的名称在所有接受对象名称作为参数的`Inspector`方法中得到正确转义（例如表名、视图名等）。修复了 SQLite 和 MSSQL 中存在的两个引号问题。

    参考：[#5456](https://www.sqlalchemy.org/trac/ticket/5456)

### mssql

+   **[mssql] [错误] [sql]**

    修复了 mssql 方言错误地转义包含‘]’字符的对象名称的错误。

    参考：[#5467](https://www.sqlalchemy.org/trac/ticket/5467)

### 杂项

+   **[用例] [py3k]**

    向`DeclarativeMeta.__init__()`方法添加了一个`**kw`参数。这允许类支持[**PEP 487**](https://peps.python.org/pep-0487/)元类钩子`__init_subclass__`。感谢 Ewen Gillies 的拉取请求。

    参考：[##5357](https://www.sqlalchemy.org/trac/ticket/#5357)

## 1.3.18

发布日期：2020 年 6 月 25 日

### orm

+   **[orm] [用例]**

    在查询中使用`Query.filter_by()`时，如果第一个实体不是映射类，则改进错误消息。

    参考：[#5326](https://www.sqlalchemy.org/trac/ticket/5326)

+   **[orm] [用例]**

    向`query_expression()`构造添加了一个新参数`query_expression.default_expr`，如果未使用`with_expression()`选项，则将自动应用于查询。感谢 Haoyu Sun 的拉取请求。

    参考：[#5198](https://www.sqlalchemy.org/trac/ticket/5198)

### 示例

+   **[示例] [更改]**

    在 examples.performance 套件中添加了新选项`--raw`，它将为任意数量的性能分析可视化工具提供原始配置文件测试。删除了“runsnake”选项，因为在这一点上很难构建 runsnake；

### 引擎

+   **[引擎] [错误]**

    进一步完善了对[#5326](https://www.sqlalchemy.org/trac/ticket/5326)中修复的“reset”代理的修复，现在在未正确调用时会发出警告并纠正行为。已经确定并修复了发出此警告的其他情况。

    参考：[#5326](https://www.sqlalchemy.org/trac/ticket/5326)

+   **[引擎] [错误]**

    修复了`URL`对象中的问题，其中对对象进行字符串化不会对特殊字符进行 URL 编码，导致 URL 无法重新使用为真实 URL。感谢 Miguel Grinberg 的拉取请求。

    参考：[#5341](https://www.sqlalchemy.org/trac/ticket/5341)

### sql

+   **[sql] [使用案例]**

    在 `table()` 构造中添加了一个“.schema”参数，允许临时表达式也包括模式名称。拉取请求由 Dylan Modesitt 提供。

    参考：[#5309](https://www.sqlalchemy.org/trac/ticket/5309)

+   **[sql] [更改] [sybase]**

    为 sybase 方言添加了 `.offset` 支持。拉取请求由 Alan D. Snow 提供。

    参考：[#5294](https://www.sqlalchemy.org/trac/ticket/5294)

+   **[sql] [错误]**

    正确应用 `self_group` 在 `type_coerce` 元素中。

    在表达式中使用时，类型转换元素未正确应用分组规则。

    参考：[#5344](https://www.sqlalchemy.org/trac/ticket/5344)

+   **[sql] [错误]**

    将 `Select.with_hint()` 输出添加到在语句上调用 `str()` 时生成的通用 SQL 字符串中。以前，此子句将被省略，假定它是方言特定的。提示文本以括号表示，以指示此类提示的渲染在后端之间有所不同。

    参考：[#5353](https://www.sqlalchemy.org/trac/ticket/5353)

+   **[sql] [模式]**

    引入 `IdentityOptions` 来存储序列和标识列的常见参数。

    参考：[#5324](https://www.sqlalchemy.org/trac/ticket/5324)

### 模式

+   **[模式] [错误]**

    修复了在使用 `tometadata()` 复制数据库对象（例如 `Table`）时省略 `dialect_options` 的问题。

    参考：[#5276](https://www.sqlalchemy.org/trac/ticket/5276)

### mysql

+   **[mysql] [使用案例]**

    为 mysql 实现了行级锁定支持。拉取请求由 Quentin Somerville 提供。

    参考：[#4860](https://www.sqlalchemy.org/trac/ticket/4860)

### sqlite

+   **[sqlite] [使用案例]**

    SQLite 3.31 添加了对计算列的支持。此更改在针对 SQLite 时启用了它们在 SQLAlchemy 中的支持。

    参考：[#5297](https://www.sqlalchemy.org/trac/ticket/5297)

+   **[sqlite] [错误]**

    将“exists”添加到 SQLite 的保留字列表中，以便在将其用作标签或列名时将其引用。拉取请求由 Thodoris Sotiropoulos 提供。

    参考：[#5395](https://www.sqlalchemy.org/trac/ticket/5395)

### mssql

+   **[mssql] [更改]**

    将 `supports_sane_rowcount_returning = False` 的要求从 `PyODBCConnector` 级别移到 `MSDialect_pyodbc`，因为在某些情况下 pyodbc 不正确工作。

    参考：[#5321](https://www.sqlalchemy.org/trac/ticket/5321)

+   **[mssql] [错误]**

    优化了 SQL Server 方言用于解释包含许多点的多部分模式名称的逻辑，以便在名称不使用括号或引号时不会实际丢失任何点，并且还支持一个包含多个部分的“dbname”标记，包括它可能具有多个、独立括号的部分。

    参考：[#5364](https://www.sqlalchemy.org/trac/ticket/5364), [#5366](https://www.sqlalchemy.org/trac/ticket/5366)

+   **[mssql] [错误] [pyodbc]**

    修复了 pyodbc 连接器中的一个问题，即在使用完全空的 URL 时会发出关于 pyodbc“drivername”的警告。当生成非连接的方言对象或在使用“creator”参数创建引擎时，空 URL 是正常的。现在只有在缺少驱动程序名称但其他参数仍然存在时才会发出警告。

    参考：[#5346](https://www.sqlalchemy.org/trac/ticket/5346)

+   **[mssql] [错误]**

    修复了为 pyodbc DBAPI 组装 ODBC 连接字符串时的问题。包含分号和/或大括号“{}”的标记没有被正确转义，导致 ODBC 驱动程序错误解释连接字符串属性。

    参考：[#5373](https://www.sqlalchemy.org/trac/ticket/5373)

+   **[mssql] [错误]**

    修复了一个问题，其中`datetime.time`参数被转换为`datetime.datetime`，导致它们与实际的`TIME`列进行比较时不兼容。

    参考：[#5339](https://www.sqlalchemy.org/trac/ticket/5339)

+   **[mssql] [错误]**

    修复了 SQL Server pyodbc 方言中`is_disconnect`函数在异常消息中包含与 SQL Server ODBC 错误代码匹配的子字符串时错误报告断开状态的问题。

    参考：[#5359](https://www.sqlalchemy.org/trac/ticket/5359)

### oracle

+   **[oracle] [错误] [反射]**

    修复了 Oracle 方言中的一个错误，其中包含完整主键列集的索引会被误认为是主键索引本身，即使存在多个也会被省略。检查已经被优化，以比较主键约束的名称与索引名称本身，而不是根据索引中存在的列来猜测。

    参考：[#5421](https://www.sqlalchemy.org/trac/ticket/5421)

### orm

+   **[orm] [用例]**

    在查询中使用`Query.filter_by()`时，如果第一个实体不是映射类，则改进错误消息。

    参考：[#5326](https://www.sqlalchemy.org/trac/ticket/5326)

+   **[orm] [用例]**

    为 `query_expression()` 构造添加了一个新参数 `query_expression.default_expr`，如果未使用 `with_expression()` 选项，则会自动应用于查询。感谢 Haoyu Sun 的拉取请求。

    参考：[#5198](https://www.sqlalchemy.org/trac/ticket/5198)

### examples

+   **[examples] [change]**

    为 examples.performance 套件添加了新选项 `--raw`，该选项将原始配置文件测试转储以供任意数量的性能分析可视化工具使用。删除了“runsnake”选项，因为此时 runsnake 非常难以构建；

### engine

+   **[engine] [bug]**

    进一步完善了在 [#5326](https://www.sqlalchemy.org/trac/ticket/5326) 中修复的“reset”代理的修复程序，现在在未正确调用时会发出警告并纠正其行为。已经确定并修复了额外的情景，在这些情况下会发出此警告。

    参考：[#5326](https://www.sqlalchemy.org/trac/ticket/5326)

+   **[engine] [bug]**

    `URL` 对象中的问题已经修复，该对象的字符串化不会对特殊字符进行 URL 编码，导致 URL 无法作为真实的 URL 重新使用。感谢 Miguel Grinberg 的拉取请求。

    参考：[#5341](https://www.sqlalchemy.org/trac/ticket/5341)

### sql

+   **[sql] [usecase]**

    向 `table()` 构造添加了一个“.schema”参数，允许临时表达式也包括模式名称。感谢 Dylan Modesitt 的拉取请求。

    参考：[#5309](https://www.sqlalchemy.org/trac/ticket/5309)

+   **[sql] [change] [sybase]**

    向 sybase 方言添加了 `.offset` 支持。感谢 Alan D. Snow 的拉取请求。

    参考：[#5294](https://www.sqlalchemy.org/trac/ticket/5294)

+   **[sql] [bug]**

    正确应用 self_group 在 type_coerce 元素中。

    当在表达式中使用时，类型强制元素未正确应用分组规则

    参考：[#5344](https://www.sqlalchemy.org/trac/ticket/5344)

+   **[sql] [bug]**

    在调用语句的`str()`时，现已将 `Select.with_hint()` 的输出添加到产生的通用 SQL 字符串中。先前，该子句会被省略，因为假设它是方言特定的。提示文本显示在括号内，以指示此类提示的呈现在后端之间变化。

    参考：[#5353](https://www.sqlalchemy.org/trac/ticket/5353)

+   **[sql] [schema]**

    引入 `IdentityOptions` 以存储序列和标识列的常见参数。

    参考：[#5324](https://www.sqlalchemy.org/trac/ticket/5324)

### 模式

+   **[模式] [错误]**

    修复了在使用 `tometadata()` 复制数据库对象（例如 `Table`）时省略 `dialect_options` 的问题。

    参考：[#5276](https://www.sqlalchemy.org/trac/ticket/5276)

### mysql

+   **[mysql] [用例]**

    为 mysql 实现了行级锁定支持。感谢 Quentin Somerville 的拉取请求。

    参考：[#4860](https://www.sqlalchemy.org/trac/ticket/4860)

### sqlite

+   **[sqlite] [用例]**

    SQLite 3.31 添加了对计算列的支持。此更改在针对 SQLite 时启用了 SQLAlchemy 对其的支持。

    参考：[#5297](https://www.sqlalchemy.org/trac/ticket/5297)

+   **[sqlite] [错误]**

    将“exists”添加到 SQLite 的保留字列表中，以便在用作标签或列名时将此单词引用。感谢 Thodoris Sotiropoulos 的拉取请求。

    参考：[#5395](https://www.sqlalchemy.org/trac/ticket/5395)

### mssql

+   **[mssql] [更改]**

    将 `supports_sane_rowcount_returning = False` 的要求从 `PyODBCConnector` 级别移至 `MSDialect_pyodbc`，因为在某些情况下 pyodbc 可以正常工作。

    参考：[#5321](https://www.sqlalchemy.org/trac/ticket/5321)

+   **[mssql] [错误]**

    优化了 SQL Server 方言用于解释包含许多点的多部分模式名称的逻辑，以便在名称未使用括号或引号时实际上不会丢失任何点，并且还支持具有许多部分的“dbname”标记，包括可能具有多个、独立括号的部分。

    参考：[#5364](https://www.sqlalchemy.org/trac/ticket/5364), [#5366](https://www.sqlalchemy.org/trac/ticket/5366)

+   **[mssql] [错误] [pyodbc]**

    修复了 pyodbc 连接器中的问题，当使用完全空的 URL 时会发出关于 pyodbc “drivername”的警告。在生成非连接的方言对象或在使用“creator”参数创建 create_engine() 时，空 URL 是正常的。现在只有在缺少驱动程序名称但其他参数仍然存在时才会发出警告。

    参考：[#5346](https://www.sqlalchemy.org/trac/ticket/5346)

+   **[mssql] [错误]**

    修复了为 pyodbc DBAPI 组装 ODBC 连接字符串的问题。包含分号和/或大括号“{}”的标记未能正确转义，导致 ODBC 驱动程序错误解释连接字符串属性。

    参考：[#5373](https://www.sqlalchemy.org/trac/ticket/5373)

+   **[mssql] [错误]**

    修复了将 `datetime.time` 参数转换为 `datetime.datetime` 的问题，使其与实际 `TIME` 列进行 `>=` 等比较时不兼容。

    参考：[#5339](https://www.sqlalchemy.org/trac/ticket/5339)

+   **[mssql] [错误]**

    修复了 SQL Server pyodbc 方言中的一个问题，即当异常消息中包含与 SQL Server ODBC 错误代码匹配的子字符串时，`is_disconnect`函数会错误地报告断开状态。

    参考：[#5359](https://www.sqlalchemy.org/trac/ticket/5359)

### oracle

+   **[oracle] [bug] [reflection]**

    修复了 Oracle 方言中的一个 bug，即包含完整主键列集的索引会被误认为是主键索引本身，即使存在多个索引也会被省略。检查已经被细化为根据主键约束的名称与索引名称本身进行比较，而不是基于索引中存在的列来猜测。

    参考：[#5421](https://www.sqlalchemy.org/trac/ticket/5421)

## 1.3.17

发布日期：2020 年 5 月 13 日

### orm

+   **[orm] [usecase]**

    添加了一个访问器`Comparator.expressions`，它提供对在多列`ColumnProperty`属性下映射的列组的访问。

    参考：[#5262](https://www.sqlalchemy.org/trac/ticket/5262)

+   **[orm] [usecase]**

    引入了`relationship.sync_backref`标志，用于控制在关系中添加用于同步修改 Python 属性的事件。这取代了先前的更改[#5149](https://www.sqlalchemy.org/trac/ticket/5149)，该更改警告称`viewonly=True`关系的 back_populates 或 backref 配置的目标将被禁止。

    参考：[#5237](https://www.sqlalchemy.org/trac/ticket/5237)

+   **[orm] [bug]**

    修复了一个 bug，即在已经具有与请求的相同的基于子查询的 with_polymorphic 设置的映射器上，将`with_polymorphic()`用作通过`RelationshipComparator.of_type()`进行连接的目标时，ON 子句在连接中不会正确别名化的问题。

    参考：[#5288](https://www.sqlalchemy.org/trac/ticket/5288)

+   **[orm] [bug]**

    修复了一个在加载器选项（如 selectinload()）与烘焙查询系统交互的领域中的问题，即如果加载器选项本身具有当前不兼容缓存的元素（如 with_polymorphic()对象），则不应发生查询的缓存。在这些一些情况下，烘焙加载器有时无法完全使自身失效，导致错过了急切加载。

    参考：[#5303](https://www.sqlalchemy.org/trac/ticket/5303)

+   **[orm] [bug]**

    修改了内部的“identity set”实现，该集合根据其 id()而不是其哈希值对对象进行哈希处理，以避免实际调用对象的`__hash__()`方法，这些对象通常是用户映射的对象。一些方法调用了这个方法作为实现的副作用。

    参考：[#5304](https://www.sqlalchemy.org/trac/ticket/5304)

+   **[orm] [bug]**

    当尝试对不是实际映射实例的对象进行 ORM 多对一比较时，会引发一个信息性错误消息。不支持与标量子查询等比较；使用`Comparator.has()`更好地实现了与子查询的广义比较。

    参考：[#5269](https://www.sqlalchemy.org/trac/ticket/5269)

### 引擎

+   **[engine] [bug]**

    修复了一个相当关键的问题，即在 DBAPI 连接仍处于未回滚状态时可能将其返回到连接池。负责回滚连接的重置代理可能在事务“关闭”而未回滚或提交的情况下出现损坏，在使用 ORM 会话并以涉及保存点的特定模式发出 .close() 时可能发生这种情况。修复确保重置代理始终处于活动状态。

    参考：[#5326](https://www.sqlalchemy.org/trac/ticket/5326)

### 模式

+   **[schema] [bug]**

    修复了一个问题，即延迟与表关联的`Index`，例如当它包含一个尚未与任何`Table`关联的`Column`时，如果还包含非表导向表达式，则无法正确附加。

    参考：[#5298](https://www.sqlalchemy.org/trac/ticket/5298)

+   **[schema] [bug]**

    当使用`MetaData.sorted_tables`属性以及`sort_tables()`函数时，如果由于外键约束之间存在循环依赖关系而无法正确排序给定的表，则会发出警告。在这种情况下，这些函数将不再按照外键对涉及的表进行排序，并将发出警告。不属于循环的其他表仍将按依赖顺序返回。以前，当检测到循环时，排序表例程将返回一个集合，该集合在检测到循环时将无条件省略所有外键，并且不会发出警告。

    参考：[#5316](https://www.sqlalchemy.org/trac/ticket/5316)

+   **[schema]**

    在 `Column` 的 `__repr__` 方法中添加了 `comment` 属性。

    参考：[#4138](https://www.sqlalchemy.org/trac/ticket/4138)

### postgresql

+   **[postgresql] [用例]**

    在 PostgreSQL 中添加了对 `ARRAY`、`Enum`、`JSON` 或 `JSONB` 类型的列的支持。以前在这些情况下需要使用一种解决方法。

    参考：[#5265](https://www.sqlalchemy.org/trac/ticket/5265)

+   **[postgresql] [用例]**

    在将配置为 `Enum.native_enum` 设置为 `False` 且 `Enum.create_constraint` 未设置为 `False` 时，向表添加具有 `ARRAY` 类型的列时引发明确的 `CompileError`

    参考：[#5266](https://www.sqlalchemy.org/trac/ticket/5266)

### mssql

+   **[mssql] [bug] [reflection]**

    修复了在使用旧版 TDS 版本 4.2 时，在 MSSQL 中反射计算列引入的回归。如果无法检测到协议版本，方言将尝试检测首次连接的协议版本并在兼容模式下运行。

    参考：[#5255](https://www.sqlalchemy.org/trac/ticket/5255)

+   **[mssql] [bug] [reflection]**

    修复了在使用不支持 `concat` 函数的 SQL Server 版本 2012 之前的 MSSQL 中反射计算列引入的回归。

    参考：[#5271](https://www.sqlalchemy.org/trac/ticket/5271)

### oracle

+   **[oracle] [性能] [bug]**

    更改了获取 CLOB 和 BLOB 对象的实现方式，使用了 cx_Oracle 的原生实现，将 CLOB/BLOB 对象与其他结果列一起获取，而不是执行单独的获取操作。如往常一样，可以通过将 auto_convert_lobs 设置为 False 来禁用此功能。

    作为此更改的一部分，给定空字符串的 CLOB 现在在 INSERT 时返回 None，在 SELECT 时与 Oracle 上的 VARCHAR 一致。

    参考：[#5314](https://www.sqlalchemy.org/trac/ticket/5314)

+   **[oracle] [bug]**

    对于 cx_oracle 方言如何为 LOB 和数字数据类型设置每列输出类型处理程序进行了一些修改，以适应可能在 cx_Oracle 8 中出现的潜在更改。

    参考：[#5246](https://www.sqlalchemy.org/trac/ticket/5246)

### 杂项

+   **[change] [firebird]**

    调整了`firebird://` URI 的方言加载，如果已安装外部的 sqlalchemy-firebird 方言，则使用它，否则回退到（现在已弃用的）内部 Firebird 方言。

    参考：[#5278](https://www.sqlalchemy.org/trac/ticket/5278)

### orm

+   **[orm] [用例]**

    添加了一个访问器`Comparator.expressions`，它提供了对多列`ColumnProperty`属性下映射的列组的访问。

    参考：[#5262](https://www.sqlalchemy.org/trac/ticket/5262)

+   **[orm] [用例]**

    引入了`relationship.sync_backref`标志，用于控制是否添加会改变 Python 属性的同步事件。这取代了先前的更改[#5149](https://www.sqlalchemy.org/trac/ticket/5149)，该更改警告说`viewonly=True`关系的目标 back_populates 或 backref 配置将被禁止。

    参考：[#5237](https://www.sqlalchemy.org/trac/ticket/5237)

+   **[orm] [错误]**

    修复了一个 bug，即在已经具有与请求的等效的基于子查询的 with_polymorphic 设置的映射器上，通过`RelationshipComparator.of_type()`将`with_polymorphic()`用作连接的目标时，不会正确别名连接中的 ON 子句。

    参考：[#5288](https://www.sqlalchemy.org/trac/ticket/5288)

+   **[orm] [错误]**

    修复了加载器选项（如 selectinload()）与烘焙查询系统交互的问题，即如果加载器选项本身包含当前不兼容缓存的元素（如 with_polymorphic()对象），则不应该发生查询的缓存。在这些情况下，烘焙加载器有时无法完全使自身失效，导致错过了急切加载。

    参考：[#5303](https://www.sqlalchemy.org/trac/ticket/5303)

+   **[orm] [错误]**

    修改了内部的“identity set”实现，它是一个根据对象的 id()而不是它们的哈希值进行哈希的集合，不再实际调用对象的`__hash__()`方法，这些对象通常是用户映射的对象。一些方法在实现的副作用中调用了这个方法。

    参考：[#5304](https://www.sqlalchemy.org/trac/ticket/5304)

+   **[orm] [错误]**

    当尝试对一个不是实际映射实例的对象进行 ORM 多对一比较时，会引发一个信息丰富的错误消息。不支持与标量子查询之类的比较；使用子查询进行广义比较更好地通过`Comparator.has()`实现。

    参考：[#5269](https://www.sqlalchemy.org/trac/ticket/5269)

### 引擎

+   **[引擎] [错误]**

    修复了一个相当关键的问题，即在仍处于未回滚状态时将 DBAPI 连接返回到连接池的情况。负责回滚连接的重置代理可能在事务“关闭”而没有被回滚或提交时被破坏，这在使用 ORM 会话并以某种模式发出.close()时可能发生。修复确保重置代理始终处于活动状态。

    参考：[#5326](https://www.sqlalchemy.org/trac/ticket/5326)

### 模式

+   **[模式] [错误]**

    修复了一个问题，即当一个被延迟关联到表的`Index`，例如当它包含一个尚未与任何`Table`关联的`Column`时，如果它还包含一个非表导向的表达式，则无法正确附加。

    参考：[#5298](https://www.sqlalchemy.org/trac/ticket/5298)

+   **[模式] [错误]**

    当使用`MetaData.sorted_tables`属性以及`sort_tables()`函数时，如果由于外键约束之间存在循环依赖而无法正确排序给定的表，则会发出警告。在这种情况下，函数将不再按外键对涉及的表进行排序，并将发出警告。其他不属于循环的表仍将按依赖顺序返回。以前，当检测到循环时，排序表例程将返回一个集合，该集合在检测到循环时将无条件省略所有外键，并且不会发出警告。

    参考：[#5316](https://www.sqlalchemy.org/trac/ticket/5316)

+   **[模式]**

    在`Column`的`__repr__`方法中添加`comment`属性。

    参考：[#4138](https://www.sqlalchemy.org/trac/ticket/4138)

### postgresql

+   **[postgresql] [用例]**

    在 PostgreSQL 中添加了对 `Enum`、`JSON` 或 `JSONB` 类型的列��类型 `ARRAY` 的支持。在这些用例中以前需要使用一种解决方法。

    参考：[#5265](https://www.sqlalchemy.org/trac/ticket/5265)

+   **[postgresql] [用例]**

    在添加具有配置为 `Enum.native_enum` 设置为 `False` 的 `Enum` 类型的列的表时，引发明确的 `CompileError`，当 `Enum.create_constraint` 未设置为 `False` 时

    参考：[#5266](https://www.sqlalchemy.org/trac/ticket/5266)

### mssql

+   **[mssql] [错误] [反射]**

    修复了在使用传统 TDS 版本 4.2 时，通过 MSSQL 计算列的反射引入的回归。方言将尝试检测首次连接的协议版本，如果无法检测到，则运行在兼容模式下。

    参考：[#5255](https://www.sqlalchemy.org/trac/ticket/5255)

+   **[mssql] [错误] [反射]**

    修复了在使用不支持 `concat` 函数的 SQL Server 版本（2012 年之前）时，通过 MSSQL 计算列的反射引入的回归。

    参考：[#5271](https://www.sqlalchemy.org/trac/ticket/5271)

### oracle

+   **[oracle] [性能] [错误]**

    更改了获取 CLOB 和 BLOB 对象的实现，以使用 cx_Oracle 的本机实现，该实现将 CLOB/BLOB 对象与其他结果列一起获取，而不是执行单独的获取。一如既往，可以通过将 auto_convert_lobs 设置为 False 来禁用此功能。

    作为此更改的一部分，给定空字符串的 CLOB 现在在 INSERT 时返回 None 在 SELECT 时，这与 Oracle 上的 VARCHAR 的行为现在保持一致。

    参考：[#5314](https://www.sqlalchemy.org/trac/ticket/5314)

+   **[oracle] [错误]**

    对 cx_oracle 方言如何为 LOB 和数字数据类型设置每列输出类型处理程序进行了一些修改，以调整为 cx_Oracle 8 中可能出现的潜在更改。

    参考：[#5246](https://www.sqlalchemy.org/trac/ticket/5246)

### 杂项

+   **[变更] [firebird]**

    调整了 `firebird://` URI 的方言加载，以便在安装了外部 sqlalchemy-firebird 方言时使用它，否则退回到（现在已弃用的）内部 Firebird 方言。

    参考：[#5278](https://www.sqlalchemy.org/trac/ticket/5278)

## 1.3.16

发布日期：2020 年 4 月 7 日

### orm

+   **[orm] [performance]**

    修改了子查询加载和选择加载使用的查询，不再按父实体的主键排序；这种排序是为了允许行按照它们进入的顺序直接复制到列表中，最小程度地进行 Python 端排序。然而，这些 ORDER BY 子句可能会对查询的性能产生负面影响，因为在许多情况下，这些列是从子查询派生的，或者以其他方式不是实际的主键列，以至于 SQL 规划器无法使用索引。Python 端排序使用本机 itertools.group_by()来整理传入的行，并已修改为允许将多个行组合到一起使用 list.extend()，这仍然可以实现相对快速的 Python 端性能。对于包含显式 order_by 参数的关系，仍将存在一个 ORDER BY，但这是查询中唯一添加的 ORDER BY，用于两种加载方式。

    参考：[#5162](https://www.sqlalchemy.org/trac/ticket/5162)

+   **[orm] [bug]**

    修复了`selectinload()`加载选项中的错误，其中代表具有相同字符串键名称的不同关系的两个或更多加载器，从单个具有多个子类映射器的`with_polymorphic()`构造中引用时，将无法单独调用每个子查询加载器，而是使用单个基于字符串的插槽，这将阻止其他加载器被调用。

    参考：[#5228](https://www.sqlalchemy.org/trac/ticket/5228)

+   **[orm] [bug]**

    修复了一个问题，即使用会话本地“get”对目标多对一关系进行惰性加载时，存在具有正确主键的对象，但它是同类的实例时，不会正确返回 None，就像惰性加载器实际发出该行的加载时一样。

    参考：[#5210](https://www.sqlalchemy.org/trac/ticket/5210)

### orm 声明式

+   **[orm] [declarative] [bug]**

    当使用声明式 API 时，`relationship()`函数接受的第一个位置参数作为字符串参数不再使用 Python 的`eval()`函数进行解释；相反，名称是点分隔的，并且名称直接在名称解析字典中查找，而不将值视为 Python 表达式。然而，将字符串参数传递给其他必须接受 Python 表达式的`relationship()`参数仍将使用`eval()`；文档已经澄清，以确保没有任何歧义。

    参见

    评估关系参数 - 字符串评估的详细信息

    参考：[#5238](https://www.sqlalchemy.org/trac/ticket/5238)

### sql

+   **[sql] [类型]**

    在使用字符串方言进行调试时，添加了将`DateTime`、`Date`或`Time`文字编译的能力。此更改不影响保留其当前行为的真实方言实现。

    参考：[#5052](https://www.sqlalchemy.org/trac/ticket/5052)

### 模式

+   **[模式] [反射]**

    添加了对“computed”列的反射支持，这些列现在作为`Inspector.get_columns()`返回的结构的一部分返回。在反射完整的`Table`对象时，计算列将使用`Computed`构造表示。

    参考：[#5063](https://www.sqlalchemy.org/trac/ticket/5063)

### postgresql

+   **[postgresql] [错误]**

    修复了“covering”索引的问题，例如具有 INCLUDE 子句的索引，将被反映为包含所有 INCLUDE 子句中的列的常规列。如果检测到这些额外列，则现在会发出警告，指示它们当前被忽略。请注意，“covering”索引的全面支持是[#4458](https://www.sqlalchemy.org/trac/ticket/4458)的一部分。感谢 Marat Sharafutdinov 的拉取请求。

    参考：[#5205](https://www.sqlalchemy.org/trac/ticket/5205)

### mysql

+   **[mysql] [错误]**

    修复了在连接到伪 MySQL 数据库（例如��ProxySQL 提供的数据库）时 MySQL 方言中的问题，当它返回没有行时对隔离级别进行的事先检查不会阻止方言继续连接。会发出警告，指示无法检测到隔离级别。

    参考：[#5239](https://www.sqlalchemy.org/trac/ticket/5239)

### sqlite

+   **[sqlite] [用例]**

    在使用 pysqlite 时为 SQLite 实现了 AUTOCOMMIT 隔离级别。

    参考：[#5164](https://www.sqlalchemy.org/trac/ticket/5164)

### mssql

+   **[mssql] [用例] [mysql] [oracle]**

    为 SQL Server、MySQL 和 Oracle 添加了对`ColumnOperators.is_distinct_from()`和`ColumnOperators.isnot_distinct_from()`的支持。

    参考：[#5137](https://www.sqlalchemy.org/trac/ticket/5137)

### oracle

+   **[oracle] [用例]**

    在使用 cx_Oracle 时为 Oracle 实现了 AUTOCOMMIT 隔离级别。此外，为 Oracle 添加了一个固定的默认隔离级别为 READ COMMITTED。

    参考：[#5200](https://www.sqlalchemy.org/trac/ticket/5200)

+   **[Oracle] [错误] [反射]**

    修复了由于修复[#5146](https://www.sqlalchemy.org/trac/ticket/5146)引起的回归/不正确修复，其中 Oracle 方言从“all_tab_comments”视图读取表注释，但未考虑到请求的表的当前所有者，导致如果多个具有相同名称的表存在于多个模式中，则读取错误的注释。

    参考：[#5146](https://www.sqlalchemy.org/trac/ticket/5146)

### 测试

+   **[测试] [错误]**

    修复了一个问题，该问题导致测试套件无法在最近发布的 py.test 5.4.0 上运行。

    参考：[#5201](https://www.sqlalchemy.org/trac/ticket/5201)

### 杂项

+   **[枚举] [类型]**

    `Enum`类型现在支持参数`Enum.length`，用于指定在使用非本机枚举时创建 VARCHAR 列的长度，方法是将`Enum.native_enum`设置为`False`

    参考：[#5183](https://www.sqlalchemy.org/trac/ticket/5183)

+   **[安装程序]**

    确保“pyproject.toml”文件不包含在构建中，因为此文件的存在会告诉 pip 应该使用 pep-517 安装过程。由于当前工具/发行版似乎不太支持这种操作模式，在 SQLAlchemy 安装范围内通过省略该文件来避免这些问题。

    参考：[#5207](https://www.sqlalchemy.org/trac/ticket/5207)

### orm

+   **[orm] [性能]**

    修改了子查询加载和选择加载使用的查询，不再按父实体的主键排序；这种排序是为了允许行按照它们进入的顺序直接复制到列表中，以最小的 Python 端排序级别。然而，这些 ORDER BY 子句可能会对查询的性能产生负面影响，因为在许多情况下，这些列是从子查询派生的或者以其他方式不是实际的主键列，以至于 SQL 规划器无法使用索引。Python 端排序使用本机 itertools.group_by()来整理传入的行，并已修改为允许使用 list.extend()将多个行组合到一起，这仍然可以实现相对快速的 Python 端性能。对于包含显式 order_by 参数的关系，仍将存在一个 ORDER BY，但这是唯一将添加到两种加载方式的查询中的 ORDER BY。

    参考：[#5162](https://www.sqlalchemy.org/trac/ticket/5162)

+   **[orm] [错误]**

    修复了 `selectinload()` 加载选项中的错误，其中代表同一字符串键名称的不同关系的两个或多个加载器，从单个具有多个子类映射器的 `with_polymorphic()` 构造引用时，会失败地单独调用每个子查询加载，而是使用单个基于字符串的插槽，这会阻止其他加载器被调用。

    参考：[#5228](https://www.sqlalchemy.org/trac/ticket/5228)

+   **[orm] [错误]**

    修复了一个问题，即在对目标多对一关系使用会话本地“get”时，存在具有正确主键的对象，但它是同级类的实例时，懒加载不会像懒加载实际发出该行的加载时那样正确返回 None。

    参考：[#5210](https://www.sqlalchemy.org/trac/ticket/5210)

### orm 声明式

+   **[orm] [声明式] [错误]**

    当使用声明式 API 时，`relationship()` 函数的第一个位置参数接受的字符串参数不再使用 Python 的 `eval()` 函数进行解释；相反，名称是点分隔的，名称直接在名称解析字典中查找，而不将值视为 Python 表达式。然而，将字符串参数传递给其他必须接受 Python 表达式的 `relationship()` 参数仍然会使用 `eval()`；文档已经明确说明了这一点，以确保这一使用没有歧义。

    另请参阅

    关系参数的评估 - 字符串评估的详细信息

    参考：[#5238](https://www.sqlalchemy.org/trac/ticket/5238)

### sql

+   **[sql] [类型]**

    添加对在使用字符串方言进行调试时字面编译 `DateTime`、`Date` 或 `Time` 的功能。此更改不影响保留其当前行为的实际方言实现。

    参考：[#5052](https://www.sqlalchemy.org/trac/ticket/5052)

### 模式

+   **[模式] [反射]**

    增加了对“计算”列的反射支持，现在这些列作为`Inspector.get_columns()` 返回的结构的一部分返回。当反映完整的 `Table` 对象时，计算列将使用 `Computed` 构造表示。

    参考：[#5063](https://www.sqlalchemy.org/trac/ticket/5063)

### postgresql

+   **[postgresql] [bug]**

    修复了“覆盖”索引的问题，例如具有 INCLUDE 子句的索引，将被反映为包括所有 INCLUDE 子句中的列作为常规列。如果检测到这些附加列，则现在会发出警告，指示当前被忽略。请注意，对“覆盖”索引的完全支持是[#4458](https://www.sqlalchemy.org/trac/ticket/4458)的一部分。拉请求由 Marat Sharafutdinov 提供。

    参考：[#5205](https://www.sqlalchemy.org/trac/ticket/5205)

### mysql

+   **[mysql] [bug]**

    修复了连接到伪 MySQL 数据库时 MySQL 方言中的问题，例如由 ProxySQL 提供的数据库，当它返回零行时，隔离级别的前置检查将不会阻止方言继续连接。发出警告，指示无法检测到隔离级别。

    参考：[#5239](https://www.sqlalchemy.org/trac/ticket/5239)

### sqlite

+   **[sqlite] [usecase]**

    在使用 pysqlite 时，实现了 SQLite 的 AUTOCOMMIT 隔离级别。

    参考：[#5164](https://www.sqlalchemy.org/trac/ticket/5164)

### mssql

+   **[mssql] [usecase] [mysql] [oracle]**

    添加了对`ColumnOperators.is_distinct_from()`和`ColumnOperators.isnot_distinct_from()`在 SQL Server、MySQL 和 Oracle 中的支持。

    参考：[#5137](https://www.sqlalchemy.org/trac/ticket/5137)

### oracle

+   **[oracle] [usecase]**

    在使用 cx_Oracle 时，实现了 Oracle 的 AUTOCOMMIT 隔离级别。还为 Oracle 添加了固定的默认隔离级别为 READ COMMITTED。

    参考：[#5200](https://www.sqlalchemy.org/trac/ticket/5200)

+   **[oracle] [bug] [reflection]**

    由于修复[#5146](https://www.sqlalchemy.org/trac/ticket/5146)而导致的回归/不正确的修复问题，其中 Oracle 方言从“all_tab_comments”视图中读取表注释，但未能适应请求的当前表的所有者，导致如果多个模式中存在同名表，则会读取错误的注释。

    参考：[#5146](https://www.sqlalchemy.org/trac/ticket/5146)

### 测试

+   **[tests] [bug]**

    修复了一个问题，该问题阻止了最近发布的 py.test 5.4.0 运行测试套件。

    参考：[#5201](https://www.sqlalchemy.org/trac/ticket/5201)

### 杂项

+   **[enum] [types]**

    `Enum` 类型现在支持参数`Enum.length`来指定创建 VARCHAR 列时的长度，当使用非本地枚举时，通过将`Enum.native_enum`设置为`False`。

    参考：[#5183](https://www.sqlalchemy.org/trac/ticket/5183)

+   **[installer]**

    确保“pyproject.toml”文件不包含在构建中，因为此文件的存在指示 pip 应使用 pep-517 安装过程。由于当前工具/发行版似乎不太支持这种操作模式，因此通过在 SQLAlchemy 安装范围内省略该文件来避免这些问题。

    参考：[#5207](https://www.sqlalchemy.org/trac/ticket/5207)

## 1.3.15

发布日期：2020 年 3 月 11 日

### orm

+   **[orm] [bug]**

    调整了`Query.join()`发出的错误消息，当无法找到左侧时，`Query.select_from()`方法是解决问题的最佳方法。此外，在 1.3 系列中，使用确定性排序确定从传递给`Query`的给定列实体确定 FROM 子句时，以便每次确定相同的表达式。

    参考：[#5194](https://www.sqlalchemy.org/trac/ticket/5194)

+   **[orm] [bug]**

    由于[#4849](https://www.sqlalchemy.org/trac/ticket/4849)导致的 1.3.14 版本中的回归问题已修复，当发生刷新错误时，sys.exc_info() 调用未能正确调用。已为此异常情况添加了测试覆盖。

    参考：[#5196](https://www.sqlalchemy.org/trac/ticket/5196)

### orm

+   **[orm] [bug]**

    调整了`Query.join()`发出的错误消息，当无法找到左侧时，`Query.select_from()`方法是解决问题的最佳方法。此外，在 1.3 系列中，使用确定性排序确定从传递给`Query`的给定列实体确定 FROM 子句时，以便每次确定相同的表达式。

    参考：[#5194](https://www.sqlalchemy.org/trac/ticket/5194)

+   **[orm] [bug]**

    由于[#4849](https://www.sqlalchemy.org/trac/ticket/4849)导致的 1.3.14 版本中的回归问题已修复，当发生刷新错误时，sys.exc_info() 调用未能正确调用。已为此异常情况添加了测试覆盖。

    参考：[#5196](https://www.sqlalchemy.org/trac/ticket/5196)

## 1.3.14

发布日期：2020 年 3 月 10 日

### 通用

+   **[通用] [错误] [py3k]**

    对大多数或所有从内部异常捕获中引发的内部异常应用了明确的“原因”，以避免误导性的堆栈跟踪，表明异常处理中存在错误。虽然最好是通过`__suppress_context__`属性来抑制内部捕获的异常，但目前似乎还没有办法在不抑制包含的用户构造上下文的情况下做到这一点，因此暴露了内部捕获的异常作为原因，以便保持有关错误上下文的完整信息。

    参考：[#4849](https://www.sqlalchemy.org/trac/ticket/4849)

### orm

+   **[orm] [用例]**

    添加了一个新标志`InstanceEvents.restore_load_context`和`SessionEvents.restore_load_context`，适用于`InstanceEvents.load()`、`InstanceEvents.refresh()`和`SessionEvents.loaded_as_persistent()`事件，设置后将在调用事件钩子后恢复对象的“加载上下文”。这确保对象仍然处于已经进行的加载操作的“加载上下文”中，而不是由于事件中可能发生的刷新操作而将对象转移到新的加载上下文。当出现这种情况时，现在会发出警告，建议使用该标志解决此情况。该标志是“选择加入”的，因此不会对现有应用程序引入风险。

    此更改还为会话生命周期事件添加了对`raw=True`标志的支持。

    参考：[#5129](https://www.sqlalchemy.org/trac/ticket/5129)

+   **[orm] [错误]**

    修复了 1.3.13 版本中由[#5056](https://www.sqlalchemy.org/trac/ticket/5056)引起的回归，其中 ORM 路径注册表系统的重构使得路径不再能与空元组进行比较，这可能发生在特定类型的连接式贪婪加载路径中。已解决“空元组”用例，使得路径注册表在所有情况下与路径注册表进行比较；`PathRegistry`对象本身现在实现了`__eq__()`和`__ne__()`方法，这些方法将用于所有相等比较，并继续在未预期的情况下成功比较非`PathRegistry`对象时发出警告，指出此对象不应是比较的主体。

    参考：[#5110](https://www.sqlalchemy.org/trac/ticket/5110)

+   **[orm] [bug]**

    将一个设定为 viewonly=True 的关系同时设置为 back_populates 或 backref 配置的目标现在会发出警告，并最终被禁止。back_populates 特指对属性或集合的变异，当属性受到 viewonly=True 时是不允许的。viewonly 属性不受持久性行为的影响，这意味着当它在本地变异时不会反映正确的结果。

    参考：[#5149](https://www.sqlalchemy.org/trac/ticket/5149)

+   **[orm] [bug]**

    修复了与[#5080](https://www.sqlalchemy.org/trac/ticket/5080)中引入的 1.3.0b3 版本中相同区域的另一个回归问题，通过[#4468](https://www.sqlalchemy.org/trac/ticket/4468)。在这种情况下，通过`with_polymorphic()`创建跨越到与该 with_polymorphic 的基类关系的连接选项，然后进一步到常规映射关系会失败，因为基类组件不会以可以被加载策略定位的方式添加到加载路径中。在[#5080](https://www.sqlalchemy.org/trac/ticket/5080)中应用的更改已经进一步完善，以适应这种情况。

    参考：[#5121](https://www.sqlalchemy.org/trac/ticket/5121)

### engine

+   **[engine] [bug]**

    执行语句时扩展了游标/连接清理的范围，包括当结果对象无法构建时，或者 after_cursor_execute()事件引发错误，或者 autocommit/autoclose 失败。这允许在失败时清理 DBAPI 游标，并且对于无连接执行，允许关闭连接并将其返回到连接池，之前等待直到垃圾回收触发池返回。

    参考：[#5182](https://www.sqlalchemy.org/trac/ticket/5182)

### sql

+   **[sql] [bug] [postgresql]**

    修复了一个 bug，即一个 INSERT/UPDATE/DELETE 的 CTE 也使用 RETURNING，然后无法直接从中选择，因为编译器的内部状态会尝试将外部 SELECT 视为 DELETE 语句本身并访问不存在的状态。

    参考：[#5181](https://www.sqlalchemy.org/trac/ticket/5181)

### postgresql

+   **[postgresql] [bug]**

    修复了“schema_translate_map”功能与 PostgreSQL 本机枚举类型（即`Enum`, `ENUM`)不起作用的问题，即在发出“CREATE TYPE”语句时，枚举被引用的地方的 CREATE TABLE 语句中不会呈现正确的模式。

    参考：[#5158](https://www.sqlalchemy.org/trac/ticket/5158)

+   **[postgresql] [bug] [reflection]**

    修复了 PostgreSQL 反射 CHECK 约束时无法解析约束的 bug，如果 SQL 文本包含换行符，则会失败。正则表达式已经调整以适应这种情况。感谢 Eric Borczuk 提供的拉取请求。

    参考：[#5170](https://www.sqlalchemy.org/trac/ticket/5170)

### mysql

+   **[mysql] [bug]**

    修复了 MySQL `Insert.on_duplicate_key_update()` 构造中的问题，其中对于列参数使用 SQL 函数或其他组合表达式将不正确地渲染围绕列本身的`VALUES`关键字。

    参考：[#5173](https://www.sqlalchemy.org/trac/ticket/5173)

### mssql

+   **[mssql] [bug]**

    修复了`DATETIMEOFFSET`类型无法容纳`None`值的问题，该问题是为了修复此类型而首次引入的一系列修复之一，首次引入于[#4983](https://www.sqlalchemy.org/trac/ticket/4983)、[#5045](https://www.sqlalchemy.org/trac/ticket/5045)。此外，还增加了通过此类型传递后端特定日期格式字符串的支持，这通常允许在大多数其他 DBAPI 上的日期/时间类型上使用。

    参考：[#5132](https://www.sqlalchemy.org/trac/ticket/5132)

### oracle

+   **[oracle] [bug]**

    修复了一个反射 bug，表注释只能检索由用户拥有的表，而不能检索对用户可见但由其他人拥有的表。感谢 Dave Hirschfeld 提供的拉取请求。

    参考：[#5146](https://www.sqlalchemy.org/trac/ticket/5146)

### 杂项

+   **[用例] [扩展]**

    为`MutableList.sort()`函数添加了关键字参数，以便提供键函数以及“reverse”关键字参数。

    参考：[#5114](https://www.sqlalchemy.org/trac/ticket/5114)

+   **[性能] [bug]**

    修订了作为[#5085](https://www.sqlalchemy.org/trac/ticket/5085)结果添加到测试系统的内部更改，即在使用该方言时会无条件加载每个方言的与测试相关的模块，将 SQLAlchemy 的测试框架以及 ORM 引入模块导入空间。这只会对初始启动时间和内存产生一定影响，但最好这些附加模块不会反向依赖于直接的 Core 使用。

    参考：[#5180](https://www.sqlalchemy.org/trac/ticket/5180)

+   **[bug] [安装]**

    将`inspect.formatannotation`函数放入`sqlalchemy.util.compat`中，这是`inspect.formatargspec`的版本化版本所需的。该函数在 cPython 中没有文档，并且不能保证在未来的 Python 版本中可用。

    参考：[#5138](https://www.sqlalchemy.org/trac/ticket/5138)

### 一般

+   **[通用] [错误] [py3k]**

    对大多数，如果不是所有内部引发的异常应用了明确的“cause”，这些异常是从内部异常捕获中引发的，以避免误导性的堆栈跟踪，表明异常处理中存在错误。虽然最好是抑制内部捕获的异常，就像`__suppress_context__`属性那样，但目前似乎还没有一种方法可以在不抑制封闭的用户构造上下文的情况下做到这一点，因此它将内部捕获的异常公开为原因，以便保持有关错误上下文的完整信息。

    参考：[#4849](https://www.sqlalchemy.org/trac/ticket/4849)

### ORM

+   **[ORM] [用例]**

    添加了一个新标志`InstanceEvents.restore_load_context`和`SessionEvents.restore_load_context`，适用于`InstanceEvents.load()`、`InstanceEvents.refresh()`和`SessionEvents.loaded_as_persistent()`事件，设置后将在事件钩子调用后恢复对象的“加载上下文”。这确保对象仍然保持在已经进行的加载操作的“加载上下文”中，而不是由于刷新操作可能发生在事件中而将对象转移到新的加载上下文。当出现这种情况时，现在会发出警告，建议使用该标志解决此情况。该标志是“选择加入”的，因此不会对现有应用程序引入风险。

    此更改还为会话生命周期事件添加了对`raw=True`标志的支持。

    参考：[#5129](https://www.sqlalchemy.org/trac/ticket/5129)

+   **[ORM] [错误]**

    修复了 1.3.13 版本中由[#5056](https://www.sqlalchemy.org/trac/ticket/5056)引起的回归，其中对 ORM 路径注册系统进行的重构使得路径不再能与空元组进行比较，这可能发生在特定类型的连接式急加载路径中。已解决“空元组”使用情况，使得路径注册在所有情况下与路径注册进行比较；`PathRegistry`对象本身现在实现了`__eq__()`和`__ne__()`方法，这些方法将在所有相等比较中生效，并继续成功处理未预期的情况，即比较非`PathRegistry`对象时，同时发出警告，指出该对象不应是比较的主体。

    参考：[#5110](https://www.sqlalchemy.org/trac/ticket/5110)

+   **[ORM] [错误]**

    将设置一个关系为 viewonly=True，而该关系也是 back_populates 或 backref 配置的目标时会发出警告，并最终不再支持。back_populates 特指属性或集合的变化，当属性设为 viewonly=True 时不允许变化。viewonly 属性不受持久性行为的影响，这意味着当它在本地变化时不会反映出正确的结果。

    参考：[#5149](https://www.sqlalchemy.org/trac/ticket/5149)

+   **[orm] [bug]**

    修复了与 [#5080](https://www.sqlalchemy.org/trac/ticket/5080) 相同区域的另一个回归问题，在 1.3.0b3 中通过 [#4468](https://www.sqlalchemy.org/trac/ticket/4468) 引入，通过`with_polymorphic()`创建跨越关系到基类的 joined option，然后进一步到常规映射关系会失败，因为基类组件不会以可以被加载策略定位的方式添加到加载路径中。在 [#5080](https://www.sqlalchemy.org/trac/ticket/5080) 中应用的更改已进一步精细化，以适应此场景。

    参考：[#5121](https://www.sqlalchemy.org/trac/ticket/5121)

### 引擎

+   **[engine] [bug]**

    扩展了在执行语句时游标/连接清理的范围，包括当无法构造结果对象时，或者 after_cursor_execute() 事件引发错误，或者自动提交 / 自动关闭失败时。这允许在失败时清理 DBAPI 游标，并且在无连接执行时允许连接关闭并返回到连接池，之前它等待垃圾回收触发连接池返回。

    参考：[#5182](https://www.sqlalchemy.org/trac/ticket/5182)

### sql

+   **[sql] [bug] [postgresql]**

    修复了一个问题，在 INSERT/UPDATE/DELETE 的 CTE 中同时使用 RETURNING 时，无法直接从中 SELECT 的 bug。因为编译器的内部状态会尝试将外部 SELECT 当作 DELETE 语句本身并访问不存在的状态。

    参考：[#5181](https://www.sqlalchemy.org/trac/ticket/5181)

### postgresql

+   **[postgresql] [bug]**

    修复了一个问题，即“schema_translate_map”功能无法与 PostgreSQL 原生枚举类型（即 `Enum`, `ENUM`）一起工作，因为虽然“CREATE TYPE”语句会以正确的模式发出，但模式不会在引用枚举时在 CREATE TABLE 语句中呈现。

    参考：[#5158](https://www.sqlalchemy.org/trac/ticket/5158)

+   **[postgresql] [bug] [reflection]**

    修复了 PostgreSQL 反射 CHECK 约束的 bug，如果 SQL 文本包含换行符，则解析约束会失败。正则表达式已经调整以适应这种情况。感谢 Eric Borczuk 提交的拉取请求。

    参考：[#5170](https://www.sqlalchemy.org/trac/ticket/5170)

### mysql

+   **[mysql] [bug]**

    修复了 MySQL `Insert.on_duplicate_key_update()` 构造中的问题，其中对于列参数使用 SQL 函数或其他组合表达式时，不会正确渲染围绕列本身的 `VALUES` 关键字。

    参考：[#5173](https://www.sqlalchemy.org/trac/ticket/5173)

### mssql

+   **[mssql] [bug]**

    修复了 `DATETIMEOFFSET` 类型无法容纳 `None` 值的问题，这是为了修复此类型的一系列问题而引入的，首次在 [#4983](https://www.sqlalchemy.org/trac/ticket/4983) 中引入，[#5045](https://www.sqlalchemy.org/trac/ticket/5045)。此外，还增加了通过此类型传递后端特定日期格式字符串的支持，这通常允许在大多数其他 DBAPI 上的日期/时间类型上使用。

    参考：[#5132](https://www.sqlalchemy.org/trac/ticket/5132)

### oracle

+   **[oracle] [bug]**

    修复了一个反射 bug，表注释只能检索由用户拥有的表，而不能检索对用户可见但由其他人拥有的表。感谢 Dave Hirschfeld 提交的拉取请求。

    参考：[#5146](https://www.sqlalchemy.org/trac/ticket/5146)

### misc

+   **[usecase] [ext]**

    为 `MutableList.sort()` 函数添加了关键字参数，以便提供键函数以及“reverse”关键字参数。

    参考：[#5114](https://www.sqlalchemy.org/trac/ticket/5114)

+   **[performance] [bug]**

    修订了由于 [#5085](https://www.sqlalchemy.org/trac/ticket/5085) 导致的测试系统的内部更改，其中对于每个方言的测试相关模块将在使用该方言时无条件加载，将 SQLAlchemy 的测试框架���及 ORM 引入模块导入空间。这只会对初始启动时间和内存产生一定影响，但最好这些额外模块不会反向依赖于直接的 Core 使用。

    参考：[#5180](https://www.sqlalchemy.org/trac/ticket/5180)

+   **[bug] [installation]**

    在 `sqlalchemy.util.compat` 中添加了 `inspect.formatannotation` 函数的供应版本，这是 `inspect.formatargspec` 的供应版本所需的。该函数在 cPython 中没有文档，并且不能保证在未来的 Python 版本中可用。

    参考：[#5138](https://www.sqlalchemy.org/trac/ticket/5138)

## 1.3.13

发布日期：2020 年 1 月 22 日

### orm

+   **[orm] [performance]**

    发现了一个性能问题，即基于映射关系构建连接的系统。子句适配系统将用于大多数连接表达式，包括在常见情况下不需要适配的情况。已经对发生适配的条件进行了优化，以便在简单关系中的平均非别名连接中，没有“secondary”表使用约 70%的函数调用。

+   **[orm] [bug] [engine]**

    添加了测试支持，并修复了在 ORM 查询领域中为短暂对象创建的大量不必要的引用循环，这主要是在 ORM 查询领域。非常感谢 Carson Ip 的帮助。

    参考：[#5050](https://www.sqlalchemy.org/trac/ticket/5050)，[#5056](https://www.sqlalchemy.org/trac/ticket/5056)，[#5071](https://www.sqlalchemy.org/trac/ticket/5071)

+   **[orm] [bug]**

    修复了在 1.3.0b3 版本中引入的加载器选项中的回归问题，通过 [#4468](https://www.sqlalchemy.org/trac/ticket/4468)，其中使用`PropComparator.of_type()`创建加载器选项，针对的是一个继承自前一个关系引用的实体的别名实体，将无法生成匹配路径。另请参见在此相同版本中修复的 [#5082](https://www.sqlalchemy.org/trac/ticket/5082)，涉及类似类型的问题。

    参考：[#5107](https://www.sqlalchemy.org/trac/ticket/5107)

+   **[orm] [bug]**

    修复了在 1.3.0b3 版本中引入的连接式贪婪加载中的回归问题，通过 [#4468](https://www.sqlalchemy.org/trac/ticket/4468)，在使用`RelationshipProperty.of_type()`创建跨越`with_polymorphic()`到一个多态子类的连接选项，然后沿着常规映射关系进一步失败的情况，因为多态子类不会以可以被加载策略定位的方式将自身添加到加载路径中。已进行调整以解决此场景。

    参考：[#5082](https://www.sqlalchemy.org/trac/ticket/5082)

+   **[orm] [bug]**

    修复了在 ORM 刷新过程中的警告，当删除使用“version_id”功能的对象时，该警告未被测试覆盖。通常情况下，此警告是无法触及的，除非使用的方言将“supports_sane_rowcount”标志设置为 False，这在一些 MySQL 配置以及较旧的 Firebird 驱动程序以及可能一些第三方方言中可能发生。

    参考：[#5068](https://www.sqlalchemy.org/trac/ticket/5068)

+   **[orm] [bug]**

    修复了使用连接式懒加载时，当针对查询使用`Query.group_by()`时，查询未正确包装在子查询中的错误。当使用任何形式的结果限制方法时，例如 DISTINCT、LIMIT、OFFSET，连接式懒加载将行限制查询嵌入到子查询中，以便不影响集合结果。由于某种原因，GROUP BY 的存在从未包含在这个标准中，即使它具有与使用 DISTINCT 相似的效果。此外，该 bug 将阻止在大多数数据库平台上使用 GROUP BY 作为连接式懒加载查询的所有部分，因为这些平台禁止在查询中存在非聚合、非分组的列，而连接式懒加载的附加列将不被数据库接受。

    参考：[#5065](https://www.sqlalchemy.org/trac/ticket/5065)

### 引擎

+   **[engine] [bug]**

    当使用具有绑定值处理器的数据类型（例如“扩展 IN”参数）时，修复了在`Compiled`对象上收集的值处理器集合会发生变异的问题；特别地，这意味着当使用语句缓存和/或烘焙查询时，同一 compiled._bind_processors 集合将同时发生变异。由于这些处理器对于给定的绑定参数命名空间每次都是相同的函数，因此这个问题实际上没有负面影响，然而，`Compiled`对象的执行绝不能导致其状态发生任何更改，特别是考虑到它们被设计为一旦完全构建就是线程安全和可重复使用的。

    参考：[#5048](https://www.sqlalchemy.org/trac/ticket/5048)

### sql

+   **[sql] [usecase]**

    使用`GenericFunction`创建的函数现在可以通过将`quoted_name`结构分配给对象的.name 元素来指定函数名称是否应该用引号括起来。在 1.3.4 之前，从不将引号应用于函数名称，并且在[#4467](https://www.sqlalchemy.org/trac/ticket/4467)中引入了一些引号，但没有一种方法来强制对一个混合大小写名称进行引用。此外，当用作名称时，`quoted_name`结构将正确地在函数注册表中注册其小写名称，以便该名称继续通过`func.`注册表可用。

    另请参阅

    `GenericFunction`

    参考：[#5079](https://www.sqlalchemy.org/trac/ticket/5079)

### postgresql

+   **[postgresql] [usecase]**

    添加了对`CTE`构造的前缀支持，以支持 Postgresql 12 的“MATERIALIZED”和“NOT MATERIALIZED”短语。感谢 Marat Sharafutdinov 的拉取请求。

    另请参阅

    `HasCTE.cte()`

    参考：[#5040](https://www.sqlalchemy.org/trac/ticket/5040)

+   **[postgresql] [bug]**

    修复了 PostgreSQL 方言无法解析反射的 CHECK 约束的问题，该约束是一个布尔值函数（而不是布尔值表达式）。

    参考：[#5039](https://www.sqlalchemy.org/trac/ticket/5039)

### mssql

+   **[mssql] [bug]**

    修复了将时区感知的`datetime`值转换为字符串以用作`DATETIMEOFFSET`列的参数值时，省略了小数秒的问题。

    参考：[#5045](https://www.sqlalchemy.org/trac/ticket/5045)

### tests

+   **[tests] [bug]**

    修复了一些测试失败的问题，这些问题可能是由于 SQLite 文件锁定问题在 Windows 上发生，以及连接池相关测试中的一些时间问题；感谢 Federico Caselli 的拉取请求。

    参考：[#4946](https://www.sqlalchemy.org/trac/ticket/4946)

+   **[tests] [postgresql]**

    通过测试确保了 PostgreSQL 数据库对两阶段事务需求的改进，测试 max_prepared_transactions 是否设置为大于 0 的值。感谢 Federico Caselli 的拉取请求。

    参考：[#5057](https://www.sqlalchemy.org/trac/ticket/5057)

### misc

+   **[bug] [ext]**

    修复了 sqlalchemy.ext.serializer 中的错误，其中一个唯一的`BindParameter`对象可能会与自身冲突，如果它同时存在于映射本身和查询的过滤条件中，其中一侧将用于与未反序列化版本对比，另一侧将使用反序列化版本。在`BindParameter`中添加了类似于其“clone”方法的逻辑，该方法在反序列化时将参数名称唯一化，以避免与原始名称冲突。

    参考：[#5086](https://www.sqlalchemy.org/trac/ticket/5086)

### orm

+   **[orm] [performance]**

    通过系统发现了一个性能问题，即基于映射关系构建连接的问题。子句适配系统将用于大多数连接表达式，包括在常见情况下不需要适配的情况。已经对发生适配的条件进行了优化，以便平均非别名连接沿着简单关系使用约 70%的函数调用。

+   **[orm] [bug] [engine]**

    添加了测试支持，并修复了在 ORM 查询中为短暂对象创建的大量不必要的引用循环。非常感谢 Carson Ip 在此方面的帮助。

    参考：[#5050](https://www.sqlalchemy.org/trac/ticket/5050)，[#5056](https://www.sqlalchemy.org/trac/ticket/5056)，[#5071](https://www.sqlalchemy.org/trac/ticket/5071)

+   **[orm] [bug]**

    修复了在 1.3.0b3 中引入的加载器选项回归，通过[#4468](https://www.sqlalchemy.org/trac/ticket/4468)，使用`PropComparator.of_type()`创建一个针对前一个关系引用的实体的继承子类的别名实体的加载器选项将无法生成匹配路径。另请参见[#5082](https://www.sqlalchemy.org/trac/ticket/5082)，在此相同版本中修复了类似类型的问题。

    参考：[#5107](https://www.sqlalchemy.org/trac/ticket/5107)

+   **[orm] [bug]**

    修复了在 1.3.0b3 中引入的连接急加载中的回归，通过[#4468](https://www.sqlalchemy.org/trac/ticket/4468)，使用`RelationshipProperty.of_type()`创建跨`with_polymorphic()`到多态子类的连接选项，然后沿着常规映射关系进一步失败，因为多态子类不会以可以被加载策略定位的方式将自身添加到加载路径中。已进行微调以解决此场景。

    参考：[#5082](https://www.sqlalchemy.org/trac/ticket/5082)

+   **[orm] [bug]**

    修复了在 ORM 刷新过程中的警告，当删除使用“version_id”功能的对象时，该警告未被测试覆盖。通常情况下，除非使用的方言将“supports_sane_rowcount”标志设置为 False，否则无法到达此警告，尽管对于一些 MySQL 配置以及较旧的 Firebird 驱动程序以及可能的一些第三方方言来说，这是可能的。

    参考：[#5068](https://www.sqlalchemy.org/trac/ticket/5068)

+   **[orm] [bug]**

    修复了一个错误，即使用连接的急加载时，当针对查询使用`Query.group_by()`时，查询不会正确地包装在子查询中。当使用任何种类的结果限制方法时，例如 DISTINCT、LIMIT、OFFSET，连接的急加载将行限制的查询嵌入到子查询中，以便不影响集合结果。出于某种原因，GROUP BY 的存在从未包含在此标准中，即使它具有与使用 DISTINCT 相同的效果。此外，该错误将阻止对大多数数据库平台的连接急加载查询使用 GROUP BY，这些平台禁止在查询中存在非聚合、非分组的列，因为连接的急加载的附加列不会被数据库接受。

    参考：[#5065](https://www.sqlalchemy.org/trac/ticket/5065)

### engine

+   **[engine] [bug]**

    修复了一个问题，当使用具有绑定值处理器的数据类型与“展开 IN”参数一起使用时，`Compiled`对象上的值处理器集合会发生变异；特别是，这意味着当使用语句缓存和/或烘焙查询时，同一 compiled._bind_processors 集合会同时发生变异。由于这些处理器对于给定的绑定参数命名空间每次都是相同的函数，因此这个问题实际上没有任何负面影响，但是，`Compiled`对象的执行绝不应该导致其状态发生任何更改，特别是考虑到它们旨在在完全构造后是线程安全和可重复使用的。

    参考：[#5048](https://www.sqlalchemy.org/trac/ticket/5048)

### sql

+   **[sql] [usecase]**

    使用`GenericFunction`创建的函数现在可以通过将`quoted_name`构造分配给对象的.name 元素来指定函数的名称是否应该带引号或不带引号。在 1.3.4 之前，函数名称从不应用引号，[#4467](https://www.sqlalchemy.org/trac/ticket/4467)中引入了一些引号，但没有一种方法可以强制对混合大小写名称进行引号。此外，当作为名称使用`quoted_name`构造时，将正确在函数注册表中注册其小写名称，以便名称继续通过`func.`注册表可用。

    另请参阅

    `GenericFunction`

    参考：[#5079](https://www.sqlalchemy.org/trac/ticket/5079)

### postgresql

+   **[postgresql] [usecase]**

    为 `CTE` 构造添加了前缀支持，以支持 Postgresql 12 的“MATERIALIZED”和“NOT MATERIALIZED”短语。感谢 Marat Sharafutdinov 提供的拉取请求。

    参见

    `HasCTE.cte()`

    参考：[#5040](https://www.sqlalchemy.org/trac/ticket/5040)

+   **[postgresql] [bug]**

    修复了 PostgreSQL 方言无法解析反射的 CHECK 约束的问题，该约束是布尔值函数（而不是布尔值表达式）。  

    参考：[#5039](https://www.sqlalchemy.org/trac/ticket/5039)

### mssql

+   **[mssql] [bug]**

    修复了将时区感知的 `datetime` 值转换为字符串以用作 `DATETIMEOFFSET` 列的参数值时，省略了小数秒的问题。

    参考：[#5045](https://www.sqlalchemy.org/trac/ticket/5045)

### 测试

+   **[tests] [bug]**

    修复了一些在 Windows 上由于 SQLite 文件锁定问题导致的测试失败，以及连接池相关测试中的一些时间问题；感谢 Federico Caselli 提供的拉取请求。

    参考：[#4946](https://www.sqlalchemy.org/trac/ticket/4946)

+   **[tests] [postgresql]**

    通过测试检测到 PostgreSQL 数据库的两阶段事务要求的改进，测试 max_prepared_transactions 设置为大于 0 的值。感谢 Federico Caselli 提供的拉取请求。

    参考：[#5057](https://www.sqlalchemy.org/trac/ticket/5057)

### 杂项

+   **[bug] [ext]**

    修复了 sqlalchemy.ext.serializer 中的错误，其中唯一的 `BindParameter` 对象可能会与自身冲突，如果它存在于映射本身以及查询的筛选条件中，则一侧将用于与非反序列化版本一起使用，另一侧将使用序列化版本。类似于其 “clone” 方法的逻辑被添加到 `BindParameter` 中，该方法将在反序列化时使参数名称唯一化，以避免与其原始名称冲突。

    参考：[#5086](https://www.sqlalchemy.org/trac/ticket/5086)

## 1.3.12

发布日期：2019 年 12 月 16 日

### orm

+   **[orm] [bug]**

    修复了涉及`lazy="raise"`策略的问题，其中 ORM 删除对象会为一个简单的“use-get”样式的多对一关系引发异常，该关系配置了 lazy=”raise”。这与 1.3 版本中作为[#4353](https://www.sqlalchemy.org/trac/ticket/4353)的一部分引入的更改不一致，其中已经确定了一个不期望发出 SQL 的历史操作应绕过`lazy="raise"`检查，并且对于这种情况实际上将其视为`lazy="raise_on_sql"`。修复调整了懒加载器策略，以便在懒加载指示不应在对象不存在时发出 SQL 时不引发异常。

    参考：[#4997](https://www.sqlalchemy.org/trac/ticket/4997)

+   **[orm] [bug]**

    修复了 1.3.0 中引入的与关联代理重构相关的回归问题，该问题阻止了`composite()`属性在引用它们的关联代理方面的工作。

    参考：[#5000](https://www.sqlalchemy.org/trac/ticket/5000)

+   **[orm] [bug]**

    在设置`relationship()`上的持久性相关标志时，同时设置 viewonly=True 将会发出常规警告，因为这些标志对于 viewonly=True 关系没有意义。特别是，“cascade”设置有自己的警告，根据各个值生成，例如“delete, delete-orphan”，这些值不应适用于 viewonly 关系。然而需要注意的是，在“cascade”情况下，这些设置仍然错误地生效，即使关系设置为“viewonly”。在 1.4 版本中，将禁止在 viewonly=True 关系上设置所有与持久性相关的级联设置，以解决此问题。

    参考：[#4993](https://www.sqlalchemy.org/trac/ticket/4993)

+   **[orm] [bug] [py3k]**

    修复了将集合分配给自身作为切片时出现的问题，变异操作会失败，因为它首先无意中擦除了分配的集合。由于不改变内容的赋值不应生成事件，因此该操作现在是一个无操作。需要注意的是，修复仅适用于 Python 3；在 Python 2 中，在这种情况下不会调用`__setitem__`钩子；而是使用`__setslice__`，它会逐个项目地重新创建列表项。

    参考：[#4990](https://www.sqlalchemy.org/trac/ticket/4990)

+   **[orm] [bug]**

    修复了一个问题，即如果 Core 引擎/连接级别的“begin”失败，例如由于网络错误或数据库由于某些事务配方而被锁定，`Session`在从连接池获取该连接并立即返回它的上下文中，ORM `Session`尽管该连接未存储在该`Session`的状态中，但不会关闭连接。这将导致连接被连接池弱引用处理程序在垃圾回收中清除，这是一个不受欢迎的代码路径，在某些特殊配置中可能会在标准错误中发出错误。

    参考：[#5034](https://www.sqlalchemy.org/trac/ticket/5034)

### sql

+   **[sql] [bug]**

    修复了将“distinct”关键字传递给`select()`时不会像`select.distinct()`一样将字符串值视为“标签引用”的错误，而是会无条件引发异常。这些关键字参数和传递给`select()`的其他参数最终将在 SQLAlchemy 2.0 中被弃用。

    参考：[#5028](https://www.sqlalchemy.org/trac/ticket/5028)

+   **[sql] [bug]**

    将“无法解析标签引用”异常的文本更改为包括其他类型的标签强制转换，即“DISTINCT”也属于 PostgreSQL 方言下的此类别。

### sqlite

+   **[sqlite] [bug]**

    修复了解决 SQLite 将“numeric”亲和性分配给 JSON 数据类型的行为问题，首次描述在添加对 SQLite JSON 的支持中，该行为将标量数值 JSON 值返回为数字而不是可以进行 JSON 反序列化的字符串。SQLite 特定的 JSON 反序列化器现在对这种情况进行了优雅降级处理，作为异常并且对于单个数值值绕过反序列化，因为从 JSON 的角度来看，它们已经被反序列化。

    参考：[#5014](https://www.sqlalchemy.org/trac/ticket/5014)

### mssql

+   **[mssql] [bug]**

    通过在 PyODBC 中添加 PyODBC 级别的结果处理程序修复了对`DATETIMEOFFSET`数据类型的支持，因为它不包含对此数据类型的原生支持。这包括使用 Python 3 的“timezone” tzinfo 子类来设置时区，在 Python 2 中使用了 SQLAlchemy.util 中“timezone”的最小回退。

    参考：[#4983](https://www.sqlalchemy.org/trac/ticket/4983)

### orm

+   **[orm] [bug]**

    修复了涉及`lazy="raise"`策略的问题，其中对一个对象进行 ORM 删除操作会在一个简单的“使用获取”风格的多对一关系上引发异常，而该关系配置了`lazy="raise"`。这与在 1.3 版本中引入的更改不一致，作为[#4353](https://www.sqlalchemy.org/trac/ticket/4353)的一部分，已经确定不希望发出 SQL 的历史操作应绕过`lazy="raise"`检查，而是在这种情况下有效地将其视为`lazy="raise_on_sql"`。修复调整了懒加载器策略，以便在懒加载指示不应在对象不存在时发出 SQL 的情况下不引发异常。

    参考：[#4997](https://www.sqlalchemy.org/trac/ticket/4997)

+   **[orm] [bug]**

    修复了 1.3.0 中与[#4351](https://www.sqlalchemy.org/trac/ticket/4351)中的关联代理重构相关的回归，该回归阻止了`composite()`属性在引用它们的关联代理方面的工作。

    参考：[#5000](https://www.sqlalchemy.org/trac/ticket/5000)

+   **[orm] [bug]**

    在设置`relationship()`上的与持久性相关的标志的同时设置`viewonly=True`现在会发出常规警告，因为这些标志对于`viewonly=True`关系没有意义。特别是，“cascade”设置有自己的警告，根据各个值生成，例如“delete, delete-orphan”，不应适用于`viewonly`关系。但请注意，在“cascade”情况下，这些设置仍然错误地生效，即使关系设置为“viewonly”。在 1.4 版本中，将禁止在`viewonly=True`关系上设置所有与持久性相关的级联设置，以解决此问题。

    参考：[#4993](https://www.sqlalchemy.org/trac/ticket/4993)

+   **[orm] [bug] [py3k]**

    修复了将集合分配给自身作为切片时出现的问题，由于首先无意中擦除了分配的集合，因此变异操作将失败。由于不改变内容的赋值不应生成事件，因此该操作现在是一个无操作。请注意，修复仅适用于 Python 3；在 Python 2 中，这种情况下不会调用`__setitem__`钩子；而是使用`__setslice__`，它会逐个项目地重新创建列表项。

    参考：[#4990](https://www.sqlalchemy.org/trac/ticket/4990)

+   **[orm] [bug]**

    修复了一个问题，即如果事务的“begin”在 Core 引擎/连接级别失败，比如由于网络错误或数据库被锁定导致某些事务配方失败，在`Session`的上下文中从连接池获取该连接然后立即返回它，ORM `Session`尽管该连接未存储在该`Session`的状态中，但不会关闭该连接。这将导致连接被连接池中的弱引用处理程序在垃圾回收中清除，这是一个不受欢迎的代码路径，在某些特殊配置中可能会在标准错误中发出错误。

    参考：[#5034](https://www.sqlalchemy.org/trac/ticket/5034)

### sql

+   **[sql] [bug]**

    修复了将“distinct”关键字传递给`select()`时不会像`select.distinct()`那样将字符串值视为“标签引用”的错误，而是会无条件引发异常。这个关键字参数和其他传递给`select()`的参数最终将在 SQLAlchemy 2.0 中被弃用。

    参考：[#5028](https://www.sqlalchemy.org/trac/ticket/5028)

+   **[sql] [bug]**

    更改了“无法解析标签引用”异常的文本，包括其他类型的标签强制转换，即“DISTINCT”也属于 PostgreSQL 方言下的这一类别。

### sqlite

+   **[sqlite] [bug]**

    修复了解决 SQLite 将“numeric”亲和性分配给 JSON 数据类型的行为的问题，首次描述在添加对 SQLite JSON 的支持中，它将标量数值 JSON 值作为数字返回，而不是作为可以进行 JSON 反序列化的字符串。现在，SQLite 特定的 JSON 反序列化器对于这种情况会优雅地降级为异常，并且对于单个数值值，从 JSON 的角度来看，它们已经被反序列化。

    参考：[#5014](https://www.sqlalchemy.org/trac/ticket/5014)

### mssql

+   **[mssql] [bug]**

    修复了对 PyODBC 上的`DATETIMEOFFSET`数据类型的支持，通过添加 PyODBC 级别的结果处理程序，因为它不包含对这种数据类型的原生支持。这包括在 Python 3 中使用“timezone” tzinfo 子类来设置时区，而在 Python 2 中则利用了 SQLAlchemy.util 中“timezone”的最小回退。

    参考：[#4983](https://www.sqlalchemy.org/trac/ticket/4983)

## 1.3.11

发布日期：2019 年 11 月 11 日

### orm

+   **[orm] [usecase]**

    添加了访问器 `Query.is_single_entity()` 到 `Query`，该访问器将指示此 `Query` 返回的结果是 ORM 实体列表，还是实体或列表达式的元组。SQLAlchemy 希望在未来的版本中改进单个实体/元组的行为，以便行为在前期就是明确的，但此属性应有助于当前行为。感谢 Patrick Hayes 提供的拉取请求。

    引用：[#4934](https://www.sqlalchemy.org/trac/ticket/4934)

+   **[orm] [bug]**

    `relationship.omit_join` 标志并非意在手动设置为 True，当发生此情况时将会发出警告。`omit_join` 优化会被自动检测到，`omit_join` 标志仅用于在假设优化可能干扰正确结果的情况下禁用优化，但在现代版本的此功能中尚未观察到这种情况。当未自动检测到该标志时将其设置为 True 可能导致在使用非默认主连接条件时 `selectin` 加载功能无法正常工作。

    引用：[#4954](https://www.sqlalchemy.org/trac/ticket/4954)

+   **[orm] [bug]**

    如果将主键值传递给 `Query.get()`，并且所有主键列位置都为 None，则会发出警告。以前，传递单个 None 会在元组之外引发 `TypeError`，传递复合 None（包含 None 值的元组）会静默通过。现在的修复将单个 None 强制转换为元组，以便与其他 None 条件一致处理。感谢 Lev Izraelit 对此的帮助。

    引用：[#4915](https://www.sqlalchemy.org/trac/ticket/4915)

+   **[orm] [bug]**

    `BakedQuery` 不会缓存通过 `QueryEvents.before_compile()` 事件修改的查询，因此可能对查询应用临时修改的编译钩子将在每次运行时生效。特别是对于修改在惰性加载和急切加载中使用的查询的事件非常有帮助，比如“select in” 加载。为了重新启用通过此事件修改的查询的缓存，添加了一个新标志 `bake_ok`；请参阅使用 before_compile 事件了解详情。

    提供一种新形式的 SQL 缓存的长期计划应更全��地解决这种问题。

    参考：[#4947](https://www.sqlalchemy.org/trac/ticket/4947)

+   **[orm] [bug]**

    修复了 ORM 中的一个 bug，其中一个“secondary”表引用了一个可选择的表，该表在某种程度上会引用本地主表，当通过关系相关的连接生成关系相关的连接时，无论是通过`Query.join()`还是通过`joinedload()`，都会将连接条件的两侧都应用别名。现在“local”一侧被排除在外。

    参考：[#4974](https://www.sqlalchemy.org/trac/ticket/4974)

### engine

+   **[engine] [bug]**

    修复了一个 bug，其中在日志记录和错误报告中使用的参数 repr 需要额外的上下文以区分单个语句的参数列表和参数列表的列表，因为“列表的列表”结构也可能表示第一个参数本身是一个列表，比如对于数组参数。引擎/连接现在传入一个额外的布尔值，指示参数应如何考虑。唯一期望数组作为参数的 SQLAlchemy 后端是使用 pyformat 参数的 psycopg2，因此这个问题并不太明显，但随着其他使用位置参数的驱动程序获得更多功能，支持这一点变得重要。它还消除了参数 repr 函数根据传递的参数结构猜测的需要。

    参考：[#4902](https://www.sqlalchemy.org/trac/ticket/4902)

+   **[engine] [bug] [postgresql]**

    修复了`Inspector`中的 bug，其中缓存键生成没有考虑以元组形式传递的参数，比如要返回给 PostgreSQL 方言的视图名称样式元组。这会导致检查器对更具体的一组条件进行了过于普遍的缓存。逻辑已经调整，以包含缓存中的每个关键字元素，因为每个参数都应适用于缓存，否则方言应该绕过缓存装饰器。

    参考：[#4955](https://www.sqlalchemy.org/trac/ticket/4955)

### sql

+   **[sql] [usecase]**

    为`JSON`类型的表达式添加了新的访问器，以允许特定数据类型的访问和比较，包括字符串、整数、数字、布尔元素。这修改了将值转换为字符串进行比较的文档化方法，而是在 PostgreSQL、SQlite、MySQL 方言中添加了特定功能，以可靠地在所有情况下提供这些基本类型。

    另请参阅

    `JSON`

    `Comparator.as_string()`

    `Comparator.as_boolean()`

    `Comparator.as_float()`

    `Comparator.as_integer()`

    参考：[#4276](https://www.sqlalchemy.org/trac/ticket/4276)

+   **[sql] [usecase]**

    `text()` 构造现在支持“unique”绑定参数，这将在编译时动态地使自己唯一，从而允许多个具有相同绑定参数名称的`text()`构造组合在一起。

    参考：[#4933](https://www.sqlalchemy.org/trac/ticket/4933)

+   **[sql] [bug] [py3k]**

    更改了`quoted_name`构造的`repr()`，在 Python 3 下使用常规字符串 repr()，而不是通过“backslashreplace”转义，这可能会产生误导。

    参考：[#4931](https://www.sqlalchemy.org/trac/ticket/4931)

### 模式

+   **[schema] [usecase]**

    增加了对“计算列”DDL 的支持；这些是 DDL 列规范，用于具有服务器计算值的列，无论是在 SELECT 时（称为“虚拟”）还是在它们被 INSERT 或 UPDATE 时（称为“存储”）。支持已建立在 Postgresql、MySQL、Oracle SQL Server 和 Firebird 上。感谢 Federico Caselli 在这方面的大量工作。

    另请参阅

    计算列（GENERATED ALWAYS AS）

    参考：[#4894](https://www.sqlalchemy.org/trac/ticket/4894)

+   **[schema] [bug]**

    修复了一个 bug，即一个表的列标签与普通列名重叠，例如“foo.id AS foo_id”与“foo.foo_id”，将在检测到此重叠之前生成`._label`属性，因为在列上使用`index=True`或`unique=True`标志与默认命名约定`"column_0_label"`一起。然后，当稍后使用`._label`生成绑定参数名称时，特别是 ORM 在为 UPDATE 语句生成 WHERE 子句时使用的那些参数时，将导致失败。通过使用一个不影响`Column`状态的 DDL 生成的替代`._label`访问器来修复此问题。该访问器还绕过了关键去重步骤，因为对于 DDL 是不必要的，命名现在在 DDL 中一致地是`"<tablename>_<columnname>"`，在 DDL 中使用时没有任何后续的数字符号。

    参考：[#4911](https://www.sqlalchemy.org/trac/ticket/4911)

### mysql

+   **[mysql] [bug]**

    添加了从基本 pymysql.Error 类解释的“连接被终止”消息，以便检测关闭的连接，根据报告，此消息通过一个指示 pymysql 未正确处理的 pymysql.InternalError()对象到达。

    参考：[#4945](https://www.sqlalchemy.org/trac/ticket/4945)

### mssql

+   **[mssql] [bug]**

    修复了 MSSQL 方言中的问题，其中 SELECT 中的基于表达式的 OFFSET 值将被拒绝，尽管方言可以在 ROW NUMBER 定向的 LIMIT/OFFSET 结构内呈现此表达式。

    参考：[#4973](https://www.sqlalchemy.org/trac/ticket/4973)

+   **[mssql] [bug]**

    修复了`Engine.table_names()`方法中的问题，该方法会将方言的默认模式名称反馈到方言级别的表函数中，在 SQL Server 的情况下，它会将其解释为由 mssql 方言查看的点令牌化模式名称，这将导致在数据库用户名实际上包含点的情况下该方法失败。在 1.3 中，此方法仍然被`MetaData.reflect()`函数使用，因此是一个重要的代码路径。在 1.4 中，这是当前主开发分支，这个问题不存在，因为`MetaData.reflect()`既不使用这个方法，也不显式传递默认模式名称。尽管如此，修复仍然通过将其包装在 quoted_name()中防止方言返回的默认服务器名称值在任何情况下被解释为点令牌化名称。

    参考：[#4923](https://www.sqlalchemy.org/trac/ticket/4923)

### oracle

+   **[oracle] [usecase]**

    在 cx_Oracle 方言中添加了方言级别的标志 `encoding_errors`，可以作为 `create_engine()` 的一部分指定。在 Python 2 下，将其传递给 SQLAlchemy 的 Unicode 解码转换器，在 Python 3 下，将其传递给 cx_Oracle 的 `cursor.var()` 对象作为 `encodingErrors` 参数，以处理目标数据库中存在破损编码的非常罕见情况，除非放宽错误处理，否则无法获取。该值最终是传递给 `decode()` 的 Python “编码错误”参数之一。

    参考：[#4799](https://www.sqlalchemy.org/trac/ticket/4799)

+   **[oracle] [bug] [firebird]**

    修改了 Oracle 和 Firebird 方言的 “名称规范化” 方法，将这些方言的大写作为不区分大小写的约定转换为 SQLAlchemy 中的小写作为不区分大小写的约定，以不自动将符合大写或小写转换的名称应用于 `quoted_name` 构造，正如许多非欧洲字符所示。在元数据结构中使用的所有名称都转换为 `quoted_name` 对象；此处的更改只会影响一些检查函数的输出。

    参考：[#4931](https://www.sqlalchemy.org/trac/ticket/4931)

+   **[oracle] [bug]**

    当在绑定参数中使用时，`NCHAR` 数据类型现在将绑定到 `cx_Oracle.FIXED_NCHAR` 的 DBAPI 数据绑定，这为变长字符串提供了正确的比较行为。以前，`NCHAR` 数据类型将绑定到非固定长度的 `cx_oracle.NCHAR`，而 `CHAR` 数据类型已经绑定到 `cx_Oracle.FIXED_CHAR`，因此现在一致性是 `NCHAR` 绑定到 `cx_Oracle.FIXED_NCHAR`。

    参考：[#4913](https://www.sqlalchemy.org/trac/ticket/4913)

### 测试

+   **[tests] [bug]**

    修复了在新版本 SQLite（3.30 或更高版本）中会出现的测试失败，因为它们增加了空值排序语法以及对聚合函数的新限制。感谢 Nils Philippsen 的拉取请求。

    参考：[#4920](https://www.sqlalchemy.org/trac/ticket/4920)

### 杂项

+   **[bug] [installation] [windows]**

    添加了一个解决方案，用于解决在 Windows 安装中观察到的 setuptools 相关失败，其中 setuptools 在未正确安装 MSVC 构建依赖项时未正确报告构建错误，因此不允许优雅地降级为非 C 扩展构建。

    参考：[#4967](https://www.sqlalchemy.org/trac/ticket/4967)

+   **[bug] [firebird]**

    向 Firebird 断开连接检测添加了额外的“断开连接”消息“写入数据到连接时出错”。感谢 lukens 提供的拉取请求。

    参考：[#4903](https://www.sqlalchemy.org/trac/ticket/4903)

### orm

+   **[orm] [usecase]**

    在`Query`中添加了访问器`Query.is_single_entity()`，该访问器将指示此`Query`返回的结果是 ORM 实体列表还是实体或列表达式的元组。SQLAlchemy 希望在未来的版本中改进单个实体/元组的行为，以便行为在一开始就是明确的，但是这个属性应该有助于当前的行为。感谢 Patrick Hayes 提供的拉取请求。

    参考：[#4934](https://www.sqlalchemy.org/trac/ticket/4934)

+   **[orm] [bug]**

    `relationship.omit_join`标志并不打算手动设置为 True，当发生这种情况时将会发出警告。`omit_join`优化会被自动检测到，`omit_join`标志只打算在假设优化可能干扰正确结果的情况下禁用优化，但在这个特性的现代版本中并没有观察到这种情况。当非默认主要连接条件在使用时，将`omit_join`标志设置为 True 可能会导致 selectin 加载特性无法正确工作。

    参考：[#4954](https://www.sqlalchemy.org/trac/ticket/4954)

+   **[orm] [bug]**

    如果向`Query.get()`传递的主键值在所有主键列位置上都是 None，则会发出警告。以前，传递单个 None 会引发`TypeError`，传递复合 None（None 值的元组）会静默通过。现在的修复方法是将单个 None 强制转换为元组，以便与其他 None 条件一致处理。感谢 Lev Izraelit 的帮助。

    参考：[#4915](https://www.sqlalchemy.org/trac/ticket/4915)

+   **[orm] [bug]**

    `BakedQuery`不会缓存通过`QueryEvents.before_compile()`事件修改的查询，因此可能会对查询应用临时修改的编译钩子将在每次运行时生效。特别是对于修改用于延迟加载和急加载的查询的事件非常有帮助，比如“select in”加载。为了重新启用通过此事件修改的查询的缓存，添加了一个新标志`bake_ok`；详细信息请参见使用 before_compile 事件。

    提供一种新形式的 SQL 缓存的长期计划应该更全面地解决这种问题。

    参考：[#4947](https://www.sqlalchemy.org/trac/ticket/4947)

+   **[orm] [bug]**

    修复了一个 ORM bug，其中引用了一个与本地主表在某种方式上引用的可选择的“secondary”表，在生成关系相关的连接条件时，无论是通过`Query.join()`还是通过`joinedload()`，都会对连接条件的两侧应用别名。现在排除了“本地”一侧。

    参考：[#4974](https://www.sqlalchemy.org/trac/ticket/4974)

### engine

+   **[engine] [bug]**

    修复了一个 bug，在日志记录和错误报告中使用的参数 repr 需要额外的上下文来区分单个语句的参数列表和参数列表的列表，因为“列表的列表”结构也可能表示第一个参数本身是一个列表的单个参数列表，比如数组参数。引擎/连接现在传入一个额外的布尔值，指示参数应如何考虑。唯一期望数组作为参数的 SQLAlchemy 后端是使用 pyformat 参数的 psycopg2，因此这个问题并不太明显，但随着其他使用位置的驱动程序获得更多功能，支持这一点很重要。它还消除了参数 repr 函数根据传递的参数结构猜测的需要。

    参考：[#4902](https://www.sqlalchemy.org/trac/ticket/4902)

+   **[engine] [bug] [postgresql]**

    修复了`Inspector`中的错误，其中缓存键生成未考虑以元组形式传递的参数，例如要为 PostgreSQL 方言返回的视图名称样式元组。这将导致检查器对更具体的条件进行缓存过于普遍。逻辑已调整为在缓存中包含每个关键字元素，因为每个参数都应适用于缓存，否则应通过方言绕过缓存装饰器。

    参考：[#4955](https://www.sqlalchemy.org/trac/ticket/4955)

### sql

+   **[sql] [用例]**

    为`JSON`类型的表达式添加了新的访问器，以允许特定数据类型的访问和比较，包括字符串、整数、数字、布尔元素。这修改了将值转换为字符串进行比较的文档方法，而是在 PostgreSQL、SQlite、MySQL 方言中添加了特定功能，以可靠地在所有情况下提供这些基本类型。

    参见

    `JSON`

    `Comparator.as_string()`

    `Comparator.as_boolean()`

    `Comparator.as_float()`

    `Comparator.as_integer()`

    参考：[#4276](https://www.sqlalchemy.org/trac/ticket/4276)

+   **[sql] [用例]**

    `text()`构造现在支持“unique”绑定参数，这将在编译时动态使它们唯一，从而允许多个具有相同绑定参数名称的`text()`构造组合在一起。

    参考：[#4933](https://www.sqlalchemy.org/trac/ticket/4933)

+   **[sql] [bug] [py3k]**

    更改了`quoted_name`构造的`repr()`，在 Python 3 下使用常规字符串`repr()`，而不是通过“backslashreplace”转义，这可能会产生误导。

    参考：[#4931](https://www.sqlalchemy.org/trac/ticket/4931)

### 模式

+   **[模式] [用例]**

    为“计算列”添加了 DDL 支持；这些是用于具有服务器计算值的列的 DDL 列规范，无论是在 SELECT 时（称为“虚拟”）还是在它们被 INSERT 或 UPDATE 时（称为“存储”）。已为 Postgresql、MySQL、Oracle SQL Server 和 Firebird 建立了支持。感谢 Federico Caselli 在这方面的大量工作。

    另请参阅

    计算列（GENERATED ALWAYS AS）

    参考：[#4894](https://www.sqlalchemy.org/trac/ticket/4894)

+   **[schema] [bug]**

    修复了一个 bug，即表中的列标签与普通列名重叠，例如“foo.id AS foo_id”与“foo.foo_id”，将在能够检测到此重叠之前为列生成`._label`属性，因为在列上使用`index=True`或`unique=True`标志与默认命名约定`"column_0_label"`相结合。然后，当稍后使用`._label`生成绑定参数名称时，特别是在 ORM 生成 UPDATE 语句的 WHERE 子句时使用的那些参数时，将导致失败。通过使用一个不影响`Column`状态的 DDL 生成的替代`._label`访问器来修复此问题。该访问器还绕过了键去重步骤，因为对于 DDL 是不必要的，当在 DDL 中使用时，命名现在始终是`"<tablename>_<columnname>"`，没有任何后续的数字符号。

    参考：[#4911](https://www.sqlalchemy.org/trac/ticket/4911)

### mysql

+   **[mysql] [bug]**

    添加了从基本 pymysql.Error 类解释的“连接已关闭”消息，以便检测到关闭的连接，根据报告，此消息通过一个指示 pymysql 未正确处理的 pymysql.InternalError()对象到达。

    参考：[#4945](https://www.sqlalchemy.org/trac/ticket/4945)

### mssql

+   **[mssql] [bug]**

    修复了 MSSQL 方言中的问题，在 SELECT 中基于表达式的 OFFSET 值将被拒绝，即使方言可以在 ROW NUMBER 定向的 LIMIT/OFFSET 结构内呈现此表达式。

    参考：[#4973](https://www.sqlalchemy.org/trac/ticket/4973)

+   **[mssql] [bug]**

    修复了 `Engine.table_names()` 方法中的问题，该方法会将方言的默认模式名称反馈给方言级别的表函数，在 SQL Server 的情况下，它会将其解释为 mssql 方言视图中的点标记模式名称，这将导致在数据库用户名实际上包含点的情况下该方法失败。在 1.3 版本中，此方法仍然被 `MetaData.reflect()` 函数使用，因此是一个重要的代码路径。在当前主开发分支 1.4 中，这个问题不存在，因为 `MetaData.reflect()` 没有使用这个方法，也不会显式传递默认模式名称。尽管如此，修复仍然防止方言返回的默认服务器名称值在任何情况下被解释为点标记名称，通过将其包装在 quoted_name() 中。

    参考：[#4923](https://www.sqlalchemy.org/trac/ticket/4923)

### oracle

+   **[oracle] [usecase]**

    在 cx_Oracle 方言中添加了方言级别标志 `encoding_errors`，可以作为 `create_engine()` 的一部分指定。在 Python 2 下，这将传递给 SQLAlchemy 的 unicode 解码转换器，而在 Python 3 下，这将传递给 cx_Oracle 的 `cursor.var()` 对象作为 `encodingErrors` 参数，用于处理目标数据库中存在破损编码的非常罕见情况，除非放宽错误处理，否则无法获取。该值最终是传递给 `decode()` 的 Python “编码错误” 参数之一。

    参考：[#4799](https://www.sqlalchemy.org/trac/ticket/4799)

+   **[oracle] [bug] [firebird]**

    修改了 Oracle 和 Firebird 方言的“名称规范化”方法，将这些方言的大写作为不区分大小写的约定转换为 SQLAlchemy 的小写作为不区分大小写的约定，以避免自动将 `quoted_name` 构造应用于在大写或小写转换下匹配自身的名称，就像许多非欧洲字符一样。在元数据结构中使用的所有名称都会在任何情况下转换为 `quoted_name` 对象；这里的更改只会影响一些检查函数的输出。

    参考：[#4931](https://www.sqlalchemy.org/trac/ticket/4931)

+   **[oracle] [bug]**

    当在绑定参数中使用时，`NCHAR`数据类型现在将绑定到`cx_Oracle.FIXED_NCHAR` DBAPI 数据绑定，从而提供与可变长度字符串的正确比较行为。以前，`NCHAR`数据类型会绑定到`cx_oracle.NCHAR`，这不是固定长度；`CHAR`数据类型已经绑定到`cx_Oracle.FIXED_CHAR`，因此现在一致的是`NCHAR`绑定到`cx_Oracle.FIXED_NCHAR`。

    参考：[#4913](https://www.sqlalchemy.org/trac/ticket/4913)

### 测试

+   **[tests] [bug]**

    修复了在新的 SQLite 版本（3.30 或更高版本）中会发生的测试失败，这是由于它们添加了空值排序语法以及对聚合函数的新限制。感谢 Nils Philippsen 的拉取请求。

    参考：[#4920](https://www.sqlalchemy.org/trac/ticket/4920)

### 杂项

+   **[bug] [installation] [windows]**

    添加了一个解决方案，用于解决在 Windows 安装中观察到的与 setuptools 相关的故障，其中 setuptools 在未安装 MSVC 构建依赖项时未正确报告构建错误，因此不允许优雅地降级为非 C 扩展构建。

    参考：[#4967](https://www.sqlalchemy.org/trac/ticket/4967)

+   **[bug] [firebird]**

    添加了额外的“disconnect”消息“Error writing data to the connection”以用于 Firebird 断开检测。感谢 lukens 的拉取请求。

    参考：[#4903](https://www.sqlalchemy.org/trac/ticket/4903)

## 1.3.10

发布日期：2019 年 10 月 9 日

### mssql

+   **[mssql] [bug]**

    修复了 SQL Server 方言中的一个 bug，该 bug 涉及新的“max_identifier_length”功能，其中 mssql 方言已经具有此标志，但实现未正确适应新的初始化挂钩。

    参考：[#4857](https://www.sqlalchemy.org/trac/ticket/4857)

### oracle

+   **[oracle] [bug]**

    修复了 Oracle 方言中的一个回归问题，该问题在 Oracle 服务器 12.2 及更高版本上无意中使用了 128 个字符的最大标识符长度，尽管 1.3 系列的其余部分的规定是此值保持在 30，直到 SQLAlchemy 1.4 版本。还修复了关于“compatibility”版本检索的问题，并删除了当“v$parameter”视图不可访问时发出的警告，因为这导致用户困惑。

    参考：[#4857](https://www.sqlalchemy.org/trac/ticket/4857)，[#4898](https://www.sqlalchemy.org/trac/ticket/4898)

### mssql

+   **[mssql] [bug]**

    修复了 SQL Server 方言中的一个 bug，该 bug 涉及新的“max_identifier_length”功能，其中 mssql 方言已经具有此标志，但实现未正确适应新的初始化挂钩。

    参考：[#4857](https://www.sqlalchemy.org/trac/ticket/4857)

### oracle

+   **[oracle] [bug]**

    修复了 Oracle 方言中的回归问题，该问题在 Oracle 服务器 12.2 及更高版本中无意中使用了 128 个字符的最大标识符长度，尽管 1.3 系列的其余部分的规定是该值保持在 30，直到 SQLAlchemy 1.4 版本。还修复了检索“兼容性”版本的问题，并删除了当“v$parameter”视图不可访问时发出的警告，因为这导致用户混淆。

    参考：[#4857](https://www.sqlalchemy.org/trac/ticket/4857), [#4898](https://www.sqlalchemy.org/trac/ticket/4898)

## 1.3.9

发布日期：2019 年 10 月 4 日

### orm

+   **[orm] [bug]**

    修复了由[#4775](https://www.sqlalchemy.org/trac/ticket/4775)引起的 selectinload 加载策略中的回归问题（在版本 1.3.6 中发布），其中 None 的多对一属性将不再由加载器填充。虽然这通常不会被注意到，因为懒加载器在获取时会填充 None，但如果对象被分离，这将导致分离实例错误。

    参考：[#4872](https://www.sqlalchemy.org/trac/ticket/4872)

+   **[orm] [bug]**

    将纯字符串表达式传递给`Session.query()`已被弃用，因为所有字符串强制转换在[#4481](https://www.sqlalchemy.org/trac/ticket/4481)中已被移除，而这个应该已经包含在内。可以使用`literal_column()`函数生成文本列表达式。

    参考：[#4873](https://www.sqlalchemy.org/trac/ticket/4873)

+   **[orm] [bug]**

    在`Session`可能会在`SessionEvents.after_flush()`钩子中发生的加载操作中，隐式地将一个对象与具有相同主键的另一个对象交换出标识映射，从而分离旧对象，这可能是一个观察到的结果。警告旨在通知用户，某些特殊条件导致此情况发生，并且先前的对象可能不处于预期状态。

    参考：[#4890](https://www.sqlalchemy.org/trac/ticket/4890)

### 引擎

+   **[engine] [usecase]**

    添加了新的 `create_engine()` 参数 `create_engine.max_identifier_length`。这将覆盖方言编码的“最大标识符长度”，以适应最近更改了此长度的数据库，而 SQLAlchemy 方言尚未调整以检测该版本。该参数与现有的 `create_engine.label_length` 参数交互，它建立了匿名生成标签的最大（和默认）值。此外，已将最大标识符长度的后连接检测添加到方言系统中。此功能首先由 Oracle 方言使用。

    请参见

    最大标识符长度 - Oracle 方言文档中

    参考：[#4857](https://www.sqlalchemy.org/trac/ticket/4857)

### sql

+   **[sql] [用例]**

    当传递给 `Table` 的对象不是 `SchemaItem` 对象时，为此情况添加了显式错误消息，而不是解析为属性错误。

    参考：[#4847](https://www.sqlalchemy.org/trac/ticket/4847)

+   **[sql] [错误]**

    对于使用匿名化名称的 `bindparam()`，这些字符会在早期被剥离，匿名化名称通常是从一个使用这些字符的命名列自动生成的，它不使用 `.key`，这样它们既不会干扰 SQLAlchemy 编译器对字符串格式化的使用，也不会干扰参数的驱动级解析，这两者在修复之前都可能被演示出来。该更改仅适用于内部生成和消耗的匿名化参数名称，并不适用于终端用户定义的名称，因此该更改不应影响任何现有代码。特别适用于不以引号引用特殊参数名称的 psycopg2 驱动程序，但也会剥离前导下划线以适应 Oracle（但尚未剥离前导数字，因为一些匿名参数当前完全基于数字/下划线）；在任何情况下，Oracle 继续引用包含特殊字符的参数名称。

    参考：[#4837](https://www.sqlalchemy.org/trac/ticket/4837)

### sqlite

+   **[sqlite] [用例]**

    添加了对 sqlite “URI” 连接的支持，允许在查询字符串中传递 sqlite 特定标志，例如对于支持此功能的 Python sqlite3 驱动程序，“只读”。

    另请参阅

    URI 连接

    参考：[#4863](https://www.sqlalchemy.org/trac/ticket/4863)

### mssql

+   **[mssql] [bug]**

    将标识符引用添加到了模式名称上，在反映时使用了 “use” 语句，当使用 `Table` 中使用了 SQL Server 的多部分模式名称时，以及对 `Inspector` 方法（如 `Inspector.get_table_names()`）；这适用于数据库名称中的特殊字符或空格。此外，如果当前数据库与传递的目标所有者数据库名称匹配，则不会发出 “use” 语句。

    参考：[#4883](https://www.sqlalchemy.org/trac/ticket/4883)

### oracle

+   **[oracle] [用例]**

    Oracle 方言现在在使用 Oracle 版本 12.2 或更高版本时会发出警告，且未设置 `create_engine.max_identifier_length` 参数。在这种特定情况下，默认版本为 Oracle 服务器配置中设置的“兼容性”版本，而不是实际服务器版本。在版本 1.4 中，对于 12.2 或更高版本，默认的 max_identifier_length 将移动到 128 个字符。为了保持向前兼容性，应用程序应将 `create_engine.max_identifier_length` 设置为 30 以保持相同的长度行为，或者设置为 128 以测试即将到来的行为。此长度决定了诸如 `CREATE CONSTRAINT` 和 `DROP CONSTRAINT` 之类的语句中生成的约束名称的截断方式，这意味着新长度可能会导致与使用旧长度生成的名称不匹配，影响数据库迁移。

    另请参阅

    最大标识符长度 - 在 Oracle 方言文档中

    参考：[#4857](https://www.sqlalchemy.org/trac/ticket/4857)

+   **[oracle] [bug]**

    当使用 SQLAlchemy 的`Date`、`DateTime`或`Time`数据类型时，恢复了将 cx_Oracle.DATETIME 添加到 setinputsizes()调用中的操作，因为一些复杂查询需要它存在。这在 1.2 系列中由于任意原因被移除。

    参考：[#4886](https://www.sqlalchemy.org/trac/ticket/4886)

### tests

+   **[tests] [bug]**

    修复了在 1.3.8 版本中发布的单元测试回归，该回归会导致 Oracle、SQL Server 和其他非本地 ENUM 平台失败，因为作为[#4285](https://www.sqlalchemy.org/trac/ticket/4285)枚举可排序性的一部分添加了新的枚举测试；创建的枚举会在名称上重复约束。

    参考：[#4285](https://www.sqlalchemy.org/trac/ticket/4285)

### orm

+   **[orm] [bug]**

    修复了由[#4775](https://www.sqlalchemy.org/trac/ticket/4775)（在 1.3.6 版本中发布）引起的 selectinload 加载策略中的回归，其中 None 的多对一属性将不再被加载器填充。虽然这通常不会被注意到，因为懒加载器在获取时会填充 None，但如果对象被分离，它将导致一个分离的实例错误。

    参考：[#4872](https://www.sqlalchemy.org/trac/ticket/4872)

+   **[orm] [bug]**

    将纯字符串表达式传递给`Session.query()`已被弃用，因为所有字符串强制转换在[#4481](https://www.sqlalchemy.org/trac/ticket/4481)中被移除，而这个应该被包含在内。可以使用`literal_column()`函数生成文本列表达式。

    参考：[#4873](https://www.sqlalchemy.org/trac/ticket/4873)

+   **[orm] [bug]**

    当`Session`可能会在`SessionEvents.after_flush()`钩子中发生的加载操作中，隐式地将一个对象与另一个具有相同主键的对象交换出标识映射，从而分离旧对象时，会发出警告。该警告旨在通知用户某些特殊条件导致此情况发生，并且先前的对象可能不处于预期状态。

    参考：[#4890](https://www.sqlalchemy.org/trac/ticket/4890)

### engine

+   **[engine] [usecase]**

    添加了新的`create_engine()`参数`create_engine.max_identifier_length`。这将覆盖方言编码的“最大标识符长度”，以适应最近更改了此长度但 SQLAlchemy 方言尚未调整以检测该版本的数据库。此参数与现有的`create_engine.label_length`参数交互，它建立了匿名生成标签的最大（和默认）值。此外，方言系统中已添加了关于最大标识符长度的连接后检测。此功能首次由 Oracle 方言使用。

    另请参阅

    最大标识符长度 - 在 Oracle 方言文档中

    参考：[#4857](https://www.sqlalchemy.org/trac/ticket/4857)

### sql

+   **[sql] [用例]**

    添加了一个明确的错误消息，用于处理传递给`Table`的对象不是`SchemaItem`对象的情况，而不是解析为属性错误。

    参考：[#4847](https://www.sqlalchemy.org/trac/ticket/4847)

+   **[sql] [错误]**

    在绑定参数中干扰“pyformat”或“named”格式的字符，即`%， (, )`和空格字符，以及一些其他通常不希望的字符，会在使用匿名化名称的`bindparam()`时提前剥离，这通常是从一个包含这些字符的命名列自动生成的，该列本身不包含`.key`，以便它们既不干扰 SQLAlchemy 编译器对字符串格式化的使用，也不干扰参数的驱动程序级解析，这两者在修复之前可能会被演示。此更改仅适用于内部生成和使用的匿名化参数名称，而不适用于最终用户定义的名称，因此该更改不应对任何现有代码产生影响。特别适用于不引用特殊参数名称的 psycopg2 驱动程序，但也会剥离前导下划线以适应 Oracle（但尚未剥离前导数字，因为某些匿名参数当前完全基于数字/下划线）；无论如何，Oracle 继续引用包含特殊字符的参数名称。

    参考：[#4837](https://www.sqlalchemy.org/trac/ticket/4837)

### sqlite

+   **[sqlite] [用例]**

    增加了对 sqlite “URI” 连接的支持，允许在查询字符串中传递特定于 sqlite 的标志，例如对于支持此功能的 Python sqlite3 驱动程序，“只读”。

    另请参阅

    URI 连接

    参考：[#4863](https://www.sqlalchemy.org/trac/ticket/4863)

### mssql

+   **[mssql] [错误]**

    在被反射的 `Table` 中使用 SQL Server 多部分模式名称时，将标识符引用添加到“use”语句中，以及用于 `Inspector` 方法，例如 `Inspector.get_table_names()`；这样可以容纳数据库名称中的特殊字符或空格。此外，如果当前数据库与传递的目标所有者数据库名称匹配，则不会发出“use”语句。

    参考：[#4883](https://www.sqlalchemy.org/trac/ticket/4883)

### oracle

+   **[oracle] [用例]**

    Oracle 方言现在在使用 Oracle 版本 12.2 或更高版本时会发出警告，并且未设置 `create_engine.max_identifier_length` 参数。在这种特定情况下，默认版本为 Oracle 服务器配置中设置的“兼容性”版本，而不是实际服务器版本。在版本 1.4 中，12.2 或更高版本的默认 max_identifier_length 将移至 128 个字符。为了保持向前兼容性，应用程序应将 `create_engine.max_identifier_length` 设置为 30，以保持相同的长度行为，或者设置为 128 以测试即将到来的行为。此长度决定了生成的约束名称在诸如 `CREATE CONSTRAINT` 和 `DROP CONSTRAINT` 等语句中被截断的方式，这意味着新长度可能会导致与使用旧长度生成的名称不匹配，影响数据库迁移。

    另请参阅

    最大标识符长度 - 在 Oracle 方言文档中

    参考：[#4857](https://www.sqlalchemy.org/trac/ticket/4857)

+   **[oracle] [错误]**

    恢复了在使用 SQLAlchemy 的`Date`、`DateTime`或`Time`数据类型时将 cx_Oracle.DATETIME 添加到 setinputsizes()调用中的功能，因为一些复杂查询需要这个功能。这在 1.2 系列中由于任意原因被移除。

    参考：[#4886](https://www.sqlalchemy.org/trac/ticket/4886)

### 测试

+   **[测试] [bug]**

    修复了 1.3.8 版本中发布的单元测试回归，该回归会导致 Oracle、SQL Server 和其他非本地 ENUM 平台失败，因为作为[#4285](https://www.sqlalchemy.org/trac/ticket/4285)枚举可排序性的一部分添加了新的枚举测试；这些枚举创建了重复的名称约束。

    参考：[#4285](https://www.sqlalchemy.org/trac/ticket/4285)

## 1.3.8

发布日期：2019 年 8 月 27 日

### ORM

+   **[ORM] [用例]**

    增加了对使用 Python pep-435 枚举对象作为 ORM 映射的主键列值的`Enum`数据类型的支持。由于这些值本身并不可排序，而 ORM 对主键要求可排序，因此在类型系统中添加了一个新的`TypeEngine.sort_key_function`属性，允许任何 SQL 类型实现其类型的 Python 对象排序，这将被工作单元所使用。`Enum`类型通过使用给定枚举的数据库值来定义这一点。排序方案也可以通过将可调用对象传递给`Enum.sort_key_function`参数来重新定义。感谢 Nicolas Caniart 的拉取请求。

    参考：[#4285](https://www.sqlalchemy.org/trac/ticket/4285)

+   **[ORM] [bug]**

    修复了一个 bug，`Load`对象由于内部上下文字典中的映射器/关系状态而无法被 pickle 化。现在，这些对象已经通过与加载器选项系统中其他元素类似的技术转换为可 pickle 化。

    参考：[#4823](https://www.sqlalchemy.org/trac/ticket/4823)

### 引擎

+   **[引擎] [特性]**

    添加了新参数`create_engine.hide_parameters`，当设置为 True 时，将导致不再记录 SQL 参数，也不会在`StatementError`对象的字符串表示中呈现。

    参考：[#4815](https://www.sqlalchemy.org/trac/ticket/4815)

+   **[engine] [bug]**

    修复了一个问题，即如果在首次连接时发生的“初始化”过程遇到意外异常，初始化过程将无法完成，然后在后续连接尝试中不再尝试，导致方言处于未初始化或部分初始化状态，在需要根据对现有连接的检查来建立参数的范围内。事件系统中的“仅调用一次”逻辑已经重新设计，以适应这种情况，使用新的私有 API 功能建立了一个“仅执行一次”钩子，将继续允许初始化程序在后续连接中触发，直到完成而不引发异常。这不会影响事件系统中现有`once=True`标志的行为。

    参考：[#4807](https://www.sqlalchemy.org/trac/ticket/4807)

### postgresql

+   **[postgresql] [usecase]**

    增加了对包含特殊 PostgreSQL 修饰符“NOT VALID”的 CHECK 约束的反射支持，该修饰符可能存在于已添加到现有表中的 CHECK 约束中，指示它们不应用于表中的现有数据。由`Inspector.get_check_constraints()`返回的 CHECK 约束的 PostgreSQL 字典可能包含一个额外的条目`dialect_options`，其中将包含一个条目`"not_valid": True`，如果检测到此符号。拉取请求由 Bill Finn 提供。

    参考：[#4824](https://www.sqlalchemy.org/trac/ticket/4824)

+   **[postgresql] [bug]**

    修订了刚刚添加的对 psycopg2“execute_values()”功能的支持的方法，该功能在 1.3.7 中添加了对[#4623](https://www.sqlalchemy.org/trac/ticket/4623)的支持。该方法依赖于一个正则表达式，该正则表达式无法匹配更复杂的 INSERT 语句，例如涉及子查询的语句。新方法完全匹配作为 VALUES 子句呈现的字符串。

    参考：[#4623](https://www.sqlalchemy.org/trac/ticket/4623)

+   **[postgresql] [bug]**

    修复了一个错误，即 Postgresql 运算符（例如`Comparator.contains()`和`Comparator.contained_by()`）在针对`array`对象使用非整数值时无法正确运行，这是由于错误的断言语句。

    参考：[#4822](https://www.sqlalchemy.org/trac/ticket/4822)

### sqlite

+   **[sqlite] [bug] [reflection]**

    修复了一个 bug，其中一个外键被设置为仅通过表名而不是列名引用父表时，由于 SQLite 的 PRAGMA 没有报告这些列（如果它们没有被明确给出），因此“引用列”不会正确反映。由于某种原因，这是硬编码为假定本地列的名称，这对某些情况可能有效，但是不正确。新方法反映了被引用表的主键，并将约束列列表用作被引用列列表，如果远程列在反映的 PRAGMA 中不存在。

    参考：[#4810](https://www.sqlalchemy.org/trac/ticket/4810)

### orm

+   **[orm] [usecase]**

    增加了对使用 Python pep-435 枚举对象作为 ORM 映射的主键列值的`Enum`数据类型的支持。由于这些值本身不可排序，这是 ORM 对主键所需的，因此在类型系统中添加了一个新的`TypeEngine.sort_key_function`属性，允许任何 SQL 类型实现其类型的 Python 对象的排序，该排序由工作单元查询。然后，`Enum`类型使用给定枚举的数据库值定义这一点。通过将可调用对象传递给`Enum.sort_key_function`参数，还可以重新定义排序方案。感谢 Nicolas Caniart 的拉取请求。

    参考：[#4285](https://www.sqlalchemy.org/trac/ticket/4285)

+   **[orm] [bug]**

    修复了一个 bug，其中`Load`对象由于内部上下文字典中的映射器/关系状态而无法被 pickle 化。现在，这些对象通过类似于加载器选项系统中其他元素长期以来可序列化的技术进行转换为可 pickle 化。

    参考：[#4823](https://www.sqlalchemy.org/trac/ticket/4823)

### engine

+   **[engine] [feature]**

    增加了新参数`create_engine.hide_parameters`，当设置为 True 时，将导致不再记录 SQL 参数，也不会在`StatementError`对象的字符串表示中呈现。

    参考：[#4815](https://www.sqlalchemy.org/trac/ticket/4815)

+   **[engine] [bug]**

    修复了一个问题，即如果在首次连接时发生“初始化”过程的方言遇到意外异常，初始化过程将无法完成，然后在后续连接尝试中不再尝试，导致方言处于未初始化或部分初始化状态，在需要根据实时连接检查来建立参数的范围内。事件系统中的“仅调用一次”逻辑已经重新设计，以适应这种情况，使用新的私有 API 功能建立了一个“仅执行一次”钩子，将继续允许初始化程序在后续连接中触发，直到完成而不引发异常。这不会影响事件系统中现有的`once=True`标志的行为。

    参考：[#4807](https://www.sqlalchemy.org/trac/ticket/4807)

### postgresql

+   **[postgresql] [usecase]**

    增加了对包含特殊的 PostgreSQL 限定词“NOT VALID”的 CHECK 约束的反射支持，该限定词可能存在于已向现有表添加的 CHECK 约束中，指示不将其应用于表中现有数据。由`Inspector.get_check_constraints()`返回的 CHECK 约束的 PostgreSQL 字典可能包含一个额外的条目`dialect_options`，其中将包含一个条目`"not_valid": True`，如果检测到此符号。拉取请求由 Bill Finn 提供。

    参考：[#4824](https://www.sqlalchemy.org/trac/ticket/4824)

+   **[postgresql] [bug]**

    修订了对于在 1.3.7 中添加的对 psycopg2“execute_values()”功能的支持的方法，该功能添加在[#4623](https://www.sqlalchemy.org/trac/ticket/4623)中。该方法依赖于一个正则表达式，该正则表达式无法匹配更复杂的 INSERT 语句，例如涉及子查询的语句。新方法完全匹配作为 VALUES 子句呈现的字符串。

    参考：[#4623](https://www.sqlalchemy.org/trac/ticket/4623)

+   **[postgresql] [bug]**

    修复了一个错误，即 Postgresql 运算符（例如`Comparator.contains()`和`Comparator.contained_by()`）在针对`array`对象使用时，对非整数值的功能无法正确运行，这是由于错误的断言语句。

    参考：[#4822](https://www.sqlalchemy.org/trac/ticket/4822)

### sqlite

+   **[sqlite] [bug] [reflection]**

    修复了一个 bug，即设置为仅通过表名而不是列名引用父表的 FOREIGN KEY 不会正确地反映为设置“referred columns”，因为如果没有显式给出，SQLite 的 PRAGMA 不会报告这些列。出于某种原因，这被硬编码为假设本地列的名称，这对某些情况可能有效，但不正确。新方法反映了被引用表的主键，并使用约束列列表作为被引用列列表，如果远程列不直接在反映的 PRAGMA 中存在。

    参考：[#4810](https://www.sqlalchemy.org/trac/ticket/4810)

## 1.3.7

发布日期：2019 年 8 月 14 日

### orm

+   **[orm] [bug]**

    修复了由新的 many-to-one 逻辑的 selectinload 引起的回归，其中一个基于非真实外键的 primaryjoin 条件会导致如果父对象上给定键值的相关对象不存在，则会引发 KeyError。

    参考：[#4777](https://www.sqlalchemy.org/trac/ticket/4777)

+   **[orm] [bug]**

    修复了一个 bug，即在与应用了基于表达式的“offset”的查询一起使用`Query.first()`或切片表达式时，会引发 TypeError，因为“offset”的“or”条件不希望它是 SQL 表达式而不是整数或 None。

    参考：[#4803](https://www.sqlalchemy.org/trac/ticket/4803)

### sql

+   **[sql] [bug]**

    修复了一个问题，即包含一些无法解析为特定列的函数表达式的`Index`对象，与基于字符串的列名混合，将无法正确初始化其内部状态，导致 DDL 编译期间失败。

    参考：[#4778](https://www.sqlalchemy.org/trac/ticket/4778)

+   **[sql] [bug]**

    修复了一个 bug，即`TypeEngine.column_expression()`方法在 UNION 或其他`_selectable.CompoundSelect`内的后续 SELECT 语句中不会被应用，尽管 SELECT 语句在语句的最顶层被渲染。新逻辑现在区分了渲染列表达式（对于列表中的所有 SELECT 都需要）与收集结果行的返回数据类型，后者仅对第一个 SELECT 需要。

    参考：[#4787](https://www.sqlalchemy.org/trac/ticket/4787)

+   **[sql] [bug]**

    修复了内部克隆 SELECT 结构可能导致键错误的问题，如果 SELECT 的副本更改其状态，使其列列表发生变化。这在一些 ORM 场景中观察到，可能是 1.3 及以上版本独有的问题，因此部分是一个回归修复。

    参考：[#4780](https://www.sqlalchemy.org/trac/ticket/4780)

### postgresql

+   **[postgresql] [usecase]**

    为 psycopg2 方言添加了新的方言标志 `executemany_mode`，它取代了之前的实验性 `use_batch_mode` 标志。`executemany_mode` 支持 psycopg2 提供的“execute batch”和“execute values”函数，后者用于编译的 `insert()` 构造。感谢 Yuval Dinari 的拉取请求。

    另请参阅

    Psycopg2 快速执行助手

    参考：[#4623](https://www.sqlalchemy.org/trac/ticket/4623)

### mysql

+   **[mysql] [usecase]**

    将 ARRAY 和 MEMBER 添加到 MySQL 保留字列表中，因为 MySQL 8.0 现在将它们视为保留字。

    参考：[#4783](https://www.sqlalchemy.org/trac/ticket/4783)

+   **[mysql] [bug]**

    当 MySQL 方言给出字符集时，MySQL 方言将在连接开始时发出“SET NAMES”以满足 MySQL 8.0 中观察到的一个明显行为，当 UNION 包含字符串列与形式为 CAST(NULL AS CHAR(..)) 的列联合时会引发一个排序错误，这是 SQLAlchemy 的 polymorphic_union 函数所做的。这个问题似乎至少影响了 PyMySQL 一年，然而最近在基于 mysqlclient 1.4.4 的更改中出现。由于这个指令的存在影响了三个不同的 MySQL 字符集设置，每个设置根据其存在的情况有复杂的影响，因此 SQLAlchemy 现在会在新连接上发出该指令以确保正确的行为。

    参考：[#4804](https://www.sqlalchemy.org/trac/ticket/4804)

+   **[mysql] [bug]**

    为上游 MySQL 8 问题添加了另一个修复，其中一个区分大小写的表名在外键约束反射中被错误报告，这是对首次为 [#4344](https://www.sqlalchemy.org/trac/ticket/4344) 添加的修复的扩展，影响一个区分大小写的列名。新问题发生在 MySQL 8.0.17 中，因此 88718 修复的一般逻辑仍然有效。

    另请参阅

    [`bugs.mysql.com/bug.php?id=96365`](https://bugs.mysql.com/bug.php?id=96365) - 上游 bug

    参考：[#4751](https://www.sqlalchemy.org/trac/ticket/4751)

### sqlite

+   **[sqlite] [bug]**

    支持 json 的方言应该在 create_engine() 级别接受参数 `json_serializer` 和 `json_deserializer`，然而 SQLite 方言将它们称为 `_json_serializer` 和 `_json_deserilalizer`。这些名称已经被更正，旧名称会有更改警告，这些参数现在被记录为 `create_engine.json_serializer` 和 `create_engine.json_deserializer`。

    参考：[#4798](https://www.sqlalchemy.org/trac/ticket/4798)

+   **[sqlite] [bug]**

    修复了在 SQLite 方言中使用“PRAGMA table_info”时引发的 bug，这意味着检测表存在性、列出表列和表外键列表的反射功能会默认为任何附加数据库中的任何表，当未给出模式名称并且表不存在于基本模式中时。修复显式地对“main”模式运行 PRAGMA，然后如果“main”返回零行，则对“temp”模式运行 PRAGMA，以维护“无模式”命名空间中的表和临时表的行为，仅在“模式”命名空间中附加表。

    参考：[#4793](https://www.sqlalchemy.org/trac/ticket/4793)

### mssql

+   **[mssql] [usecase]**

    添加了适用于 SQL Server 的新 `try_cast()` 构造，使用“TRY_CAST” 语法。感谢 Leonel Atencio 的 Pull 请求。

    参考：[#4782](https://www.sqlalchemy.org/trac/ticket/4782)

### misc

+   **[bug] [events]**

    修复了事件系统中的问题，即在使用 `once=True` 标志与动态生成的监听器函数时，如果这些监听器函数在使用后被垃圾收集，未来事件的事件注册将失败，因为假设监听函数被强引用。现在，“once”包装已修改为持久性地强引用内部函数，并且文档已更新，使用“once”不意味着自动取消注册监听器函数。

    参考：[#4794](https://www.sqlalchemy.org/trac/ticket/4794)

### orm

+   **[orm] [bug]**

    修复了由于新的一对多逻辑的 selectinload 导致的回归，其中基于真实外键的主要联接条件会导致 KeyError，如果父对象的给定键值上不存在相关对象，则会出现此问题。

    参考：[#4777](https://www.sqlalchemy.org/trac/ticket/4777)

+   **[orm] [bug]**

    修复了一个 bug，即在与应用了基于表达式的“偏移量”的查询结合使用时，使用 `Query.first()` 或切片表达式将引发 TypeError，因为“偏移量”被视为 SQL 表达式而不是整数或 None。

    参考：[#4803](https://www.sqlalchemy.org/trac/ticket/4803)

### sql

+   **[sql] [bug]**

    修复了一个问题，即包含一组功能表达式且无法解析为特定列的 `Index` 对象，与基于字符串的列名组合使用时，将无法正确初始化其内部状态，导致在 DDL 编译过程中失败。

    参考：[#4778](https://www.sqlalchemy.org/trac/ticket/4778)

+   **[sql] [bug]**

    修复了一个 bug，即 `TypeEngine.column_expression()` 方法不会应用于 UNION 或其他 `_selectable.CompoundSelect` 中的后续 SELECT 语句，即使 SELECT 语句在语句的最顶层渲染。现在的新逻辑区分了渲染列表达式（对列表中的所有 SELECT 都需要）与收集结果行的返回数据类型（仅对第一个 SELECT 需要）。

    参考文献：[#4787](https://www.sqlalchemy.org/trac/ticket/4787)

+   **[sql] [bug]**

    修复了内部克隆 SELECT 结构可能导致键错误的问题，如果 SELECT 的副本改变了其状态，使得其列列表发生了变化。这在一些 ORM 场景中被观察到，可能是独特于 1.3 及以上版本的，因此部分是一个回归修复。

    参考文献：[#4780](https://www.sqlalchemy.org/trac/ticket/4780)

### postgresql

+   **[postgresql] [usecase]**

    为 psycopg2 方言添加了新的方言标志 `executemany_mode`，它取代了先前的实验性 `use_batch_mode` 标志。`executemany_mode` 支持 psycopg2 提供的“execute batch”和“execute values”函数，后者用于编译的 `insert()` 构造。感谢 Yuval Dinari 提交的拉取请求。

    另请参阅

    Psycopg2 快速执行助手

    参考文献：[#4623](https://www.sqlalchemy.org/trac/ticket/4623)

### mysql

+   **[mysql] [usecase]**

    将 ARRAY 和 MEMBER 添加到 MySQL 保留字列表中，因为 MySQL 8.0 现在已将这些保留字。

    参考文献：[#4783](https://www.sqlalchemy.org/trac/ticket/4783)

+   **[mysql] [bug]**

    当 MySQL 方言给出字符集时，MySQL 方言将在连接开始时发出“SET NAMES”，以满足 MySQL 8.0 中观察到的明显行为，当一个 UNION 包含字符串列与形式为 CAST(NULL AS CHAR(..)) 的列时，会引发排序错误，这是 SQLAlchemy 的 polymorphic_union 函数所做的。该问题似乎已经影响了 PyMySQL 至少一年，然而最近出现了 mysqlclient 1.4.4，基于这个 DBAPI 如何创建连接的变化。由于该指令的存在影响了三个不同的 MySQL 字符集设置，每个设置都根据其存在的方式产生复杂的影响，因此 SQLAlchemy 现在将在新连接上发出该指令以确保正确的行为。

    参考文献：[#4804](https://www.sqlalchemy.org/trac/ticket/4804)

+   **[mysql] [bug]**

    为上游 MySQL 8 问题添加了另一个修复，其中对于大小写敏感的表名在外键约束反射中报告不正确，这是首次为 [#4344](https://www.sqlalchemy.org/trac/ticket/4344) 添加的修复的扩展，影响到大小写敏感的列名。新问题发生在 MySQL 8.0.17 中，因此 88718 修复的一般逻辑仍然有效。

    另请参阅

    [`bugs.mysql.com/bug.php?id=96365`](https://bugs.mysql.com/bug.php?id=96365) - 上游 bug

    参考：[#4751](https://www.sqlalchemy.org/trac/ticket/4751)

### sqlite

+   **[sqlite] [bug]**

    支持 json 的方言应在 create_engine() 级别接受参数 `json_serializer` 和 `json_deserializer`，然而 SQLite 方言将它们称为 `_json_serializer` 和 `_json_deserilalizer`。这些名称已经更正，旧名称将带有更改警告接受，并且这些参数现在被记录为 `create_engine.json_serializer` 和 `create_engine.json_deserializer`。

    参考：[#4798](https://www.sqlalchemy.org/trac/ticket/4798)

+   **[sqlite] [bug]**

    修复了在 SQLite 方言中使用 “PRAGMA table_info” 导致反射功能默认为任何附加数据库中的任何表，当未给出模式名称且表在基本模式中不存在时，用于检测表存在性、表列列表和外键列表的反射功能。修复明确地为 ‘main’ 模式运行 PRAGMA，然后如果 ‘main’ 返回没有行，则运行 ‘temp’ 模式，以保持“无模式”命名空间中的表 + 临时表的行为，仅在“模式”命名空间中附加表。

    参考：[#4793](https://www.sqlalchemy.org/trac/ticket/4793)

### mssql

+   **[mssql] [usecase]**

    为 SQL Server 添加了新的 `try_cast()` 构造，它发出“TRY_CAST”语法。感谢 Leonel Atencio 的拉取请求。

    参考：[#4782](https://www.sqlalchemy.org/trac/ticket/4782)

### 杂项

+   **[bug] [events]**

    修复了事件系统中的问题，其中使用 `once=True` 标志与动态生成的监听器函数会导致未来事件的事件注册失败，如果这些监听器函数在使用后被垃圾回收，因为假设监听函数被强引用。现在，“once” 包装已被修改为持久性地强引用内部函数，并且更新了文档，使用 “once” 不意味着自动注销监听器函数。

    参考：[#4794](https://www.sqlalchemy.org/trac/ticket/4794)

## 1.3.6

发布日期：2019 年 7 月 21 日

### orm

+   **[orm] [feature]**

    添加了新的加载器选项方法`Load.options()`，允许按层次结构构建加载器选项，因此可以在不需要多次调用`defaultload()`的情况下，将许多子选项应用于特定路径。感谢 Alessio Bogon 提出的想法。

    参考：[#4736](https://www.sqlalchemy.org/trac/ticket/4736)

+   **[orm] [performance]**

    应用于选择加载中的优化，在[#4340](https://www.sqlalchemy.org/trac/ticket/4340)中，不需要 JOIN 即可急切加载相关项目，现在也应用于多对一关系，因此仅查询相关表以进行简单的连接条件。在这种情况下，相关项目基于父级的外键列的值进行查询；如果这些列被延迟加载或其他方式未加载到集合中的任何父对象上，则加载器将退回到 JOIN 方法。

    参考：[#4775](https://www.sqlalchemy.org/trac/ticket/4775)

+   **[orm] [bug]**

    由[#4365](https://www.sqlalchemy.org/trac/ticket/4365)引起的回归错误已修复，即从一个实体到自身的连接没有使用别名时不再引发信息性错误消息，而是在断言失败时失败。已恢复信息性错误条件。

    参考：[#4773](https://www.sqlalchemy.org/trac/ticket/4773)

+   **[orm] [bug]**

    修复了一个问题，即`_ORMJoin.join()`方法，这是一个未内部使用的 ORM 级方法，公开了通常是内部过程的`Query.join()`，未正确传播`full`和`outerjoin`关键字参数。感谢 Denis Kataev 提供的拉取请求。

    参考：[#4713](https://www.sqlalchemy.org/trac/ticket/4713)

+   **[orm] [bug]**

    修复了一个错误，即指定`uselist=True`的多对一关系在主键更改时无法正确更新的问题，其中相关列需要更改。

    参考：[#4772](https://www.sqlalchemy.org/trac/ticket/4772)

+   **[orm] [bug]**

    修复了一个错误，即检测到与“动态”关系一起使用的多对一或一对一关系的用法，这是一个无效的配置，如果关系配置为`uselist=True`，则不会引发错误。当前修复是发出警告，而不是引发错误，因为否则将是向后不兼容的，但在将来的版本中将引发错误。

    参考：[#4772](https://www.sqlalchemy.org/trac/ticket/4772)

+   **[orm] [bug]**

    修复了一个 bug，即针对尚不存在的映射属性创建的同义词（例如在映射器配置之前引用 backref 时）在尝试测试最终不存在的属性时会引发递归错误（例如当类通过 Sphinx autodoc 运行时），因为同义词的未配置状态会将其置于找不到属性的循环中。

    参考：[#4767](https://www.sqlalchemy.org/trac/ticket/4767)

### engine

+   **[engine] [bug]**

    修复了一个 bug，即使用反射函数（如`MetaData.reflect()`）与应用了执行选项的`Engine`对象会失败的情况，因为生成的`OptionEngine`代理对象未包含在反射例程中使用的`.engine`属性。

    参考：[#4754](https://www.sqlalchemy.org/trac/ticket/4754)

### sql

+   **[sql] [bug]**

    调整了对`Enum`的初始化，以减少调用给定 PEP-435 枚举对象的`.__members__`属性的频率，以适应某些流行的第三方枚举库中这一属性调用昂贵的情况。

    参考：[#4758](https://www.sqlalchemy.org/trac/ticket/4758)

+   **[sql] [bug] [postgresql]**

    修复了一个问题，即`array_agg`构造与`FunctionElement.filter()`结合使用时，与数组索引操作符结合时未能产生正确的运算符优先级。

    参考：[#4760](https://www.sqlalchemy.org/trac/ticket/4760)

+   **[sql] [bug]**

    修复了一个不太可能的问题，即对于联合和其他`_selectable.CompoundSelect`对象的“对应列”例程在某些重叠列情况下可能返回错误的列，从而在使用集合操作时可能影响一些 ORM 操作，如果底层的`select()`构造之前在其他类似例程中使用过，则由于未清除缓存值而可能出现问题。

    参考：[#4747](https://www.sqlalchemy.org/trac/ticket/4747)

### postgresql

+   **[postgresql] [usecase]**

    增加了对 PostgreSQL 分区表上索引的反射支持，这在 PostgreSQL 11 版本中添加。

    参考：[#4771](https://www.sqlalchemy.org/trac/ticket/4771)

+   **[postgresql] [usecase]**

    添加了对多维 Postgresql 数组文字的支持，通过将 `array` 对象嵌套在另一个对象中。多维数组类型会被自动检测。

    另请参阅

    `array`

    参考：[#4756](https://www.sqlalchemy.org/trac/ticket/4756)

### mysql

+   **[mysql] [bug]**

    修复了一个 bug，在 `nullable=True` 时渲染 `TIMESTAMP` 数据类型时，特殊逻辑将无法工作，如果列的数据类型是 `TypeDecorator` 或 `Variant`。现在的逻辑确保将其解包到原始的 `TIMESTAMP`，以便在请求时正确渲染此特殊情况的 NULL 关键字。

    参考：[#4743](https://www.sqlalchemy.org/trac/ticket/4743)

+   **[mysql] [bug]**

    增强了对 MySQL/MariaDB 版本字符串的解析，以适应复杂的 MariaDB 版本字符串，其中 “MariaDB” 一词嵌入在其他字母数字字符中，例如 “MariaDBV1”。这种检测对于正确适应在 MySQL 和 MariaDB 之间拆分的 API 功能至关重要，例如 “transaction_isolation” 系统变量。

    参考：[#4624](https://www.sqlalchemy.org/trac/ticket/4624)

### sqlite

+   **[sqlite] [usecase]**

    添加了对 SQLite 的复合（元组）IN 操作符的支持，通过为此后端渲染 VALUES 关键字。由于其他后端（如 DB2）已知使用相同的语法，因此使用方言级别的标志 `tuple_in_values` 在基础编译器中启用了此语法。此更改还包括对 SQLite 的 “empty IN tuple” 表达式的支持，当使用 “in_()`” 在元组值和空集之间时。

    参考：[#4766](https://www.sqlalchemy.org/trac/ticket/4766)

### mssql

+   **[mssql] [bug]**

    确保用于反映索引和视图定义的查询将字符串参数明确转换为 NVARCHAR，因为许多 SQL Server 驱动程序经常将字符串值，特别是具有非 ASCII 字符或较大字符串值的值，视为 TEXT，这通常不会与 SQL Server 的信息模式表中的 VARCHAR 字符正确比较。原因之一。这些 CAST 操作已经在反射查询针对 SQL Server 的 `information_schema.` 表时发生，但缺少了另外三个针对 `sys.` 表的查询。

    参考：[#4745](https://www.sqlalchemy.org/trac/ticket/4745)

### orm

+   **[orm] [feature]**

    添加了新的加载器选项方法`Load.options()`，允许按层次结构构建加载器选项，因此可以在不需要多次调用`defaultload()`的情况下，将许多子选项应用于特定路径。感谢 Alessio Bogon 提出的想法。

    参考：[#4736](https://www.sqlalchemy.org/trac/ticket/4736)

+   **[orm] [performance]**

    在[#4340](https://www.sqlalchemy.org/trac/ticket/4340)中应用的选择加载优化，即在不需要 JOIN 即可急切加载相关项目时，现在也应用于多对一关系，因此只查询相关表以进行简单的连接条件。在这种情况下，相关项目是基于父对象上的外键列的值进行查询；如果这些列被延迟加载或其他方式未加载到集合中的任何父对象上，则加载器将退回到 JOIN 方法。

    参考：[#4775](https://www.sqlalchemy.org/trac/ticket/4775)

+   **[orm] [bug]**

    修复了由[#4365](https://www.sqlalchemy.org/trac/ticket/4365)引起的回归，即从一个实体到自身的连接没有使用别名时不再引发信息性错误消息，而是在断言失败。已恢复信息性错误条件。

    参考：[#4773](https://www.sqlalchemy.org/trac/ticket/4773)

+   **[orm] [bug]**

    修复了一个问题，即`_ORMJoin.join()`方法，这是一个未在内部使用的 ORM 级方法，公开了通常是内部过程的`Query.join()`，未正确传播`full`和`outerjoin`关键字参数。感谢 Denis Kataev 提供的拉取请求。

    参考：[#4713](https://www.sqlalchemy.org/trac/ticket/4713)

+   **[orm] [bug]**

    修复了一个 bug，即指定了`uselist=True`的多对一关系在主键更改时无法正确更新，需要更改相关列。

    参考：[#4772](https://www.sqlalchemy.org/trac/ticket/4772)

+   **[orm] [bug]**

    修复了一个 bug，即检测到“动态”关系与`uselist=True`配置的多对一或一对一使用是无效配置时，如果关系配置为`uselist=True`，则不会引发错误。当前的修复是发出警告，而不是引发错误，因为否则将是向后不兼容的，但在将来的版本中将引发错误。

    参考：[#4772](https://www.sqlalchemy.org/trac/ticket/4772)

+   **[orm] [bug]**

    修复了一个 bug，即针对尚不存在的映射属性创建的同义词，例如在它引用 backref 之前，当尝试测试其上的属性时会引发递归错误，最终这些属性不存在（当类通过 Sphinx autodoc 运行时会发生），因为同义词的未配置状态会将其置于找不到属性的循环中。

    参考：[#4767](https://www.sqlalchemy.org/trac/ticket/4767)

### engine

+   **[engine] [bug]**

    修复了一个 bug，即使用反射函数（如 `MetaData.reflect()`）与应用了执行选项的 `Engine` 对象一起使用时会失败，因为生成的 `OptionEngine` 代理对象未包含在反射例程中使用的 `.engine` 属性。

    参考：[#4754](https://www.sqlalchemy.org/trac/ticket/4754)

### sql

+   **[sql] [bug]**

    调整了 `Enum` 的初始化，以最大程度地减少调用给定 PEP-435 枚举对象的 `.__members__` 属性的频率，以适应某些流行的第三方枚举库中调用此属性的昂贵情况。

    参考：[#4758](https://www.sqlalchemy.org/trac/ticket/4758)

+   **[sql] [bug] [postgresql]**

    修复了一个问题，即 `array_agg` 结构与 `FunctionElement.filter()` 结合使用时，与数组索引运算符结合时不会产生正确的运算符优先级。

    参考：[#4760](https://www.sqlalchemy.org/trac/ticket/4760)

+   **[sql] [bug]**

    修复了一个不太可能的问题，即对于联合和其他 `_selectable.CompoundSelect` 对象的“对应列”例程在某些重叠列情况下可能返回错误的列，从而在使用集合操作时可能影响一些 ORM 操作，如果之前在其他类似的例程中使用了底层的 `select()` 构造，由于缓存值未被清除。

    参考：[#4747](https://www.sqlalchemy.org/trac/ticket/4747)

### postgresql

+   **[postgresql] [usecase]**

    增加了对 PostgreSQL 分区表上索引的反射支持，这在 PostgreSQL 11 版本中添加。

    参考：[#4771](https://www.sqlalchemy.org/trac/ticket/4771)

+   **[postgresql] [usecase]**

    通过将 `array` 对象嵌套在另一个对象中，增加了对多维 Postgresql 数组文字的支持。多维数组类型会自动检测���

    另请参阅

    `array`

    参考：[#4756](https://www.sqlalchemy.org/trac/ticket/4756)

### mysql

+   **[mysql] [bug]**

    修复了一个 bug，即当列的数据类型为 `TypeDecorator` 或 `Variant` 时，为 `nullable=True` 渲染“NULL”特殊逻辑不起作用。现在的逻辑确保将其解包到原始的 `TIMESTAMP`，以便在请求时正确呈现此特殊情况的 NULL 关键字。

    参考：[#4743](https://www.sqlalchemy.org/trac/ticket/4743)

+   **[mysql] [bug]**

    增强了对 MySQL/MariaDB 版本字符串的解析，以适应奇异的 MariaDB 版本字符串，其中“MariaDB” 一词嵌入在其他字母数字字符中，如“MariaDBV1”。这种检测对于正确适应已在 MySQL 和 MariaDB 之间拆分的 API 功能至关重要，例如“transaction_isolation” 系统变量。

    参考：[#4624](https://www.sqlalchemy.org/trac/ticket/4624)

### sqlite

+   **[sqlite] [usecase]**

    增加了对 SQLite 的复合（元组）IN 操作符的支持，通过在此后端渲染 VALUES 关键字来实现。由于其他后端（如 DB2）已知使用相同的语法，因此在基本编译器中使用方言级标志 `tuple_in_values` 启用了该语法。该更改还包括在使用“in_()”时支持 SQLite 的“空 IN 元组”表达式，介于元组值和空集之间。

    参考：[#4766](https://www.sqlalchemy.org/trac/ticket/4766)

### mssql

+   **[mssql] [bug]**

    确保用于反射索引和视图定义的查询将字符串参数明确转换为 NVARCHAR，因为许多 SQL Server 驱动程序经常将字符串值（特别是具有非 ASCII 字符或较大字符串值的值）视为 TEXT，这些值通常无法与 SQL Server 的信息模式表中的 VARCHAR 字符正确比较。这些 CAST 操作已经在反射查询针对 SQL Server `information_schema.` 表时发生，但缺少了针对针对 `sys.` 表的另外三个查询。

    参考：[#4745](https://www.sqlalchemy.org/trac/ticket/4745)

## 1.3.5

发布日期：2019 年 6 月 17 日

### orm

+   **[orm] [bug]**

    修复了一系列关于连接表继承超过两级深度的相关错误，与修改主键值结合使用，其中这些主键列也以外键关系链接在一起，这是连接表继承的典型情况。在三级继承层次结构中，中间表现在将获得其 UPDATE，仅在主键值更改且 passive_updates=False（例如，不执行外键约束）时，而以前会被跳过；类似地，如果 passive_updates=True（例如，ON UPDATE CASCADE 生效），则第三级表将不会收到 UPDATE 语句，这与早期的情况相同，因为 CASCADE 已经修改了它。在相关问题中，与连接继承层次结构中间表的主键相关联的关系，在父对象的主键被修改时也将正确地更新其外键列，即使该父对象是链接父类的子类，而在此之前，这些类将不会被计算。

    参考：[#4723](https://www.sqlalchemy.org/trac/ticket/4723)

+   **[orm] [bug]**

    修复了`Mapper.all_orm_descriptors`访问器的错误，当此项不是描述符时，它会在声明性`__mapper__`键下返回一个条目。现在将考虑到所有`InspectionAttr`对象上存在的`.is_attribute`标志，这也已被修改为对于关联代理是`True`，因为对于此对象错误地设置为 False。

    参考：[#4729](https://www.sqlalchemy.org/trac/ticket/4729)

+   **[orm] [bug]**

    修复了`Query.join()`中的回归问题，其中`aliased=True`标志未能正确地应用于过滤条件的子句适配，如果之前对同一实体进行了连接。这是因为适配器放置顺序错误。已经将顺序反转，以使最近的`aliased=True`调用的适配器优先，就像 1.2 及更早版本一样。这破坏了“elementtree”示例等内容。

    参考：[#4704](https://www.sqlalchemy.org/trac/ticket/4704)

+   **[orm] [bug] [py3k]**

    用 Python 3.3 中的完全供应版本替换了 `getfullargspec()` 的 Python 兼容性例程。最初，Python 在 Python 3.8 alpha 版本中为此函数发出了弃用警告。尽管已回滚此更改，但观察到 Python 3 实现的 `getfullargspec()` 在 3.4 系列中被重写为 `Signature` 后变得慢了一个数量级。虽然 Python 打算改进这种情况，但 SQLAlchemy 项目目前使用简单的替代方案来避免任何未来问题。

    参考：[#4674](https://www.sqlalchemy.org/trac/ticket/4674)

+   **[orm] [bug]**

    重新设计了 `AliasedClass` 使用的属性机制，不再依赖于在包装类的 MRO 上调用 `__getattribute__`，而是在包装类上正常解析属性使用 getattr()，然后解包/适应它。这允许在映射类上使用更广泛的属性样式，包括特殊的 `__getattr__()` 方案；但它也使代码更简单、更具弹性。

    参考：[#4694](https://www.sqlalchemy.org/trac/ticket/4694)

### sql

+   **[sql] [bug]**

    解决了因使用 `literal_column`()` 构造而导致的一系列引号问题。当这个构造通过子查询“代理”并由其文本匹配的标签引用时，即使字符串在 `Label` 中是使用 `quoted_name`` 构造设置的，该标签也不会应用引号规则。不将引号应用于 `Label` 的文本是一个错误，因为该文本严格地说是 SQL 标识符名称，而不是 SQL 表达式，该字符串不应该已经嵌入引号，不像 `literal_column()` 可能会应用于。维护了一个未标记的 `literal_column()` 在子查询外部传播的现有行为，以帮助手动引号方案，尽管目前尚不清楚是否可以为这种情况生成有效的 SQL。

    参考：[#4730](https://www.sqlalchemy.org/trac/ticket/4730)

### postgresql

+   **[postgresql] [usecase]**

    在为 PostgreSQL 反射索引时添加了列排序标志的支持，包括 ASC、DESC、NULLSFIRST、NULLSLAST。还将此功能添加到了通用的反射系统中，在未来的发布中可以应用于其他方言。拉取请求由 Eli Collins 提供。

    参考：[#4717](https://www.sqlalchemy.org/trac/ticket/4717)

+   **[postgresql] [bug]**

    修复了 PostgreSQL 方言无法正确反映没有成员的 ENUM 数据类型的错误，对于`get_enums()`调用返回一个带有`None`的列表，并在反映具有此类数据类型的列时引发 TypeError。现在检查返回一个空列表。

    参考：[#4701](https://www.sqlalchemy.org/trac/ticket/4701)

### mysql

+   **[mysql] [bug]**

    修复了 MySQL ON DUPLICATE KEY UPDATE 无法将列设置为值 NULL 的错误。感谢 Lukáš Banič的拉取请求。

    参考：[#4715](https://www.sqlalchemy.org/trac/ticket/4715)

### orm

+   **[orm] [bug]**

    修复了关于联接表继承超过两层深度的一系列相关错误，与修改主键值相结合，其中这些主键列也以外键关系相互链接，这在联接表继承中是典型的。在三级继承层次结构中的中间表现在将获得其 UPDATE，仅当主键值发生变化且 passive_updates=False（例如，外键约束未被执行）时；而以前会被跳过；类似地，当 passive_updates=True（例如，ON UPDATE CASCADE 生效）时，第三级表将不会收到 UPDATE 语句，这与以前的情况相同，因为 CASCADE 已经修改了它。在一个相关问题中，与联接继承层次结构的中间表的主键相关联的关系也将在父对象的主键被修改时正确地更新其外键列，即使该父对象是链接父类的子类，而以前这些类不会被计算。

    参考：[#4723](https://www.sqlalchemy.org/trac/ticket/4723)

+   **[orm] [bug]**

    修复了`Mapper.all_orm_descriptors`访问器会在声明性`__mapper__`键下返回一个关于`Mapper`本身的条目的错误，当这不是一个描述符时。现在将查询所有`InspectionAttr`对象上存在的`.is_attribute`标志，这也已被修改为对于关联代理为`True`，因为对于此对象错误地设置为 False。

    参考：[#4729](https://www.sqlalchemy.org/trac/ticket/4729)

+   **[orm] [bug]**

    修复了`Query.join()`中的回归问题，其中`aliased=True`标志未能正确应用于过滤条件的子句适配，如果之前对相同实体进行了联接。这是因为适配器的顺序放置错误。现已将顺序反转，以便最近的`aliased=True`调用的适配器优先级较高，就像在 1.2 及之前的情况一样。这破坏了“elementtree”示例等其他内容。

    参考：[#4704](https://www.sqlalchemy.org/trac/ticket/4704)

+   **[orm] [bug] [py3k]**

    用 Python 3.3 完全供应的版本替换了`getfullargspec()`的 Python 兼容性例程。最初，Python 在 Python 3.8 alpha 版本中对此函数发出了弃用警告。尽管此更改已被撤销，但观察到自 Python 3.4 系列以来，用于`getfullargspec()`的 Python 3 实现慢了一个数量级，因为它是针对`Signature`重写的。虽然 Python 计划改进这种情况，但 SQLAlchemy 项目目前正在使用简单的替换来避免任何未来的问题。

    参考：[#4674](https://www.sqlalchemy.org/trac/ticket/4674)

+   **[orm] [bug]**

    重构了`AliasedClass`使用的属性机制，不再依赖于在包装类的 MRO 上调用`__getattribute__`，而是在包装类上使用 getattr()正常解析属性，然后解包/适配。这允许映射类上的更广泛的属性样式，包括特殊的`__getattr__()`方案；但它也使代码在一般情况下更简单和更具弹性。

    参考：[#4694](https://www.sqlalchemy.org/trac/ticket/4694)

### sql

+   **[sql] [bug]**

    解决了由于使用`literal_column`()`构造引起的一系列引号问题。当此构造通过子查询“代理”并由与其文本匹配的标签引用时，即使在使用`quoted_name``构造设置`Label`的字符串时，标签也不会应用引号规则。不对`Label`的文本应用引号是一个 bug，因为此文本严格来说是 SQL 标识符名称，而不是 SQL 表达式，该字符串不应该已经包含引号，不像可能应用于的`literal_column()`。保留了非标记的`literal_column()`在子查询外部传播的现有行为，以帮助手动引号方案，尽管目前尚不清楚是否可以为这种构造生成有效的 SQL。

    参考：[#4730](https://www.sqlalchemy.org/trac/ticket/4730)

### postgresql

+   **[postgresql] [usecase]**

    在反映 PostgreSQL 索引时，添加了对列排序标志的支持，包括 ASC、DESC、NULLSFIRST、NULLSLAST。还将此功能添加到反射系统中，以便在将来的版本中应用于其他方言。感谢 Eli Collins 提供的拉取请求。

    参考：[#4717](https://www.sqlalchemy.org/trac/ticket/4717)

+   **[postgresql] [bug]**

    修复了一个 bug，即 PostgreSQL 方言无法正确反映没有成员的 ENUM 数据类型，对于`get_enums()`调用返回一个带有`None`的列表，并在反映具有此数据类型的列时引发 TypeError。现在检查返回一个空列表。

    参考：[#4701](https://www.sqlalchemy.org/trac/ticket/4701)

### mysql

+   **[mysql] [bug]**

    修复了一个 bug，即 MySQL 的 ON DUPLICATE KEY UPDATE 无法将列设置为 NULL 值。感谢 Lukáš Banič提供的拉取请求。

    参考：[#4715](https://www.sqlalchemy.org/trac/ticket/4715)

## 1.3.4

发布日期：2019 年 5 月 27 日

### orm

+   **[orm] [bug]**

    修复了`AttributeEvents.active_history`标志未设置为通过`AttributeEvents.propagate`标志传播到子类的事件侦听器的问题。这个 bug 存在于整个`AttributeEvents`系统的整个时间段。

    参考：[#4695](https://www.sqlalchemy.org/trac/ticket/4695)

+   **[orm] [bug]**

    修复了新的关联代理系统仍然不代理混合属性的回归，当它们使用`@hybrid_property.expression`装饰器返回另一个 SQL 表达式，或者当混合返回一个任意的`PropComparator`时，涉及到进一步通用化的启发式，用于检测在`QueryableAttribute`级别检测被代理对象的类型，以更好地检测描述符最终是否服务于映射类或列表达式。

    引用：[#4690](https://www.sqlalchemy.org/trac/ticket/4690)

+   **[orm] [bug]**

    对声明性类映射过程应用了“配置互斥体”映射器，以防止在动态模块导入方案仍在为相关类配置映射器的过程中使用映射器时可能发生的竞态。这并不能防止所有可能的竞态条件，比如并发导入尚未遇到相关类的情况，但它可以在 SQLAlchemy 声明性过程中尽可能地防范。

    引用：[#4686](https://www.sqlalchemy.org/trac/ticket/4686)

+   **[orm] [bug]**

    在将一个瞬态对象与`Session.merge()`一起合并到`Session`时，现在会发出警告，当对象在`Session`中已经是瞬态时。这种情况下会发出警告，表示对象通常会被双重插入。

    引用：[#4647](https://www.sqlalchemy.org/trac/ticket/4647)

+   **[orm] [bug]**

    修复了在新的关系 m2o 比较逻辑中的回归，该逻辑最初在 Improvement to the behavior of many-to-one query expressions 中引入，当与持久化为 NULL 并且在映射实例中处于未获取状态的属性进行比较时。由于该属性没有显式默认值，在持久化设置中访问时需要默认为 NULL。

    引用：[#4676](https://www.sqlalchemy.org/trac/ticket/4676)

### engine

+   **[engine] [bug] [postgresql]**

    将在方言初始化期间发生的“回滚”移动到在执行其他方言特定初始化步骤之后发生，特别是 psycopg2 方言的步骤，该步骤无意中会在第一个新连接上留下事务状态，这可能会干扰某些需要确保没有启动事务的 psycopg2 特定 API。感谢 Matthew Wilkes 提供的拉取请求。

    引用：[#4663](https://www.sqlalchemy.org/trac/ticket/4663)

### sql

+   **[sql] [bug]**

    修复了`GenericFunction`类意外地注册自身为命名函数的问题。感谢 Adrien Berchet 提供的拉取请求。

    参考：[#4653](https://www.sqlalchemy.org/trac/ticket/4653)

+   **[sql] [bug]**

    修复了布尔列的双重否定不会重置“NOT”运算符的问题。

    参考：[#4618](https://www.sqlalchemy.org/trac/ticket/4618)

+   **[sql] [bug]**

    `GenericFunction`命名空间正在迁移，以便以不区分大小写的方式查找函数名称，因为 SQL 函数在区分大小写的差异上不会发生冲突，用户定义的函数或存储过程也不会发生这种情况。现在使用`GenericFunction`声明的函数查找采用不区分大小写的方案，但支持一个弃用案例，允许存在两个或更多具有不同大小写的相同名称的`GenericFunction`对象，这将导致对该特定名称进行区分大小写查找，同时在函数注册时发出警告。感谢 Adrien Berchet 在这个复杂功能上的大量工作。

    参考：[#4569](https://www.sqlalchemy.org/trac/ticket/4569)

### postgresql

+   **[postgresql] [bug] [orm]**

    修复了一个问题，即“匹配的行数”警告即使方言报告“supports_sane_multi_rowcount=False”也会发出，例如对于 psycogp2 使用`use_batch_mode=True`等情况。

    参考：[#4661](https://www.sqlalchemy.org/trac/ticket/4661)

### mysql

+   **[mysql] [bug]**

    添加了对 DROP CHECK 约束的支持，MySQL 8.0.16 需要此约束才能删除 CHECK 约束；MariaDB 支持普通的 DROP CONSTRAINT。该逻辑通过检查服务器版本字符串是否存在 MariaDB 来区分这两种语法。Alembic 迁移已经解决了这个问题，通过在 MySQL / MariaDB CHECK 约束上实现自己的 DROP，但这个改变直接在 Core 中实现，以便普遍使用。感谢 Hannes Hansen 提供的拉取请求。

    参考：[#4650](https://www.sqlalchemy.org/trac/ticket/4650)

### mssql

+   **[mssql] [feature]**

    添加了对 SQL Server 过滤索引的支持，通过`mssql_where`参数实现，其工作方式类似于 PostgreSQL 方言中的`postgresql_where`索引函数。

    另请参阅

    过滤索引

    参考：[#4657](https://www.sqlalchemy.org/trac/ticket/4657)

+   **[mssql] [bug]**

    为 pymssql 添加了错误代码 20047 到“is_disconnect”。感谢 Jon Schuff 提供的拉取请求。

    引用：[#4680](https://www.sqlalchemy.org/trac/ticket/4680)

### 杂项

+   **[杂项] [bug]**

    从 MANIFEST.in 中删除了错误的“sqla_nose.py”符号，这会创建一个不希望的警告消息。

    引用：[#4625](https://www.sqlalchemy.org/trac/ticket/4625)

### orm

+   **[orm] [bug]**

    修复了`AttributeEvents.active_history`标志未设置为通过`AttributeEvents.propagate`标志传播到子类的事件侦听器的问题。这个错误存在于整个`AttributeEvents`系统的整个时间段。

    引用：[#4695](https://www.sqlalchemy.org/trac/ticket/4695)

+   **[orm] [bug]**

    修复了新的关联代理系统仍未代理混合属性的回归，当它们使用`@hybrid_property.expression`装饰器返回替代的 SQL 表达式，或者当混合属性在表达式级别返回任意的`PropComparator`时。这涉及进一步泛化用于检测在`QueryableAttribute`级别代理的对象类型的启发式方法，以更好地检测描述符最终是为映射类还是列表达式服务。

    引用：[#4690](https://www.sqlalchemy.org/trac/ticket/4690)

+   **[orm] [bug]**

    对声明类映射过程应用了映射器“配置互斥锁”，以防止在动态模块导入方案仍在为相关类配置映射器的过程中发生的竞争。这并不防范所有可能的竞争条件，比如如果并发导入尚未遇到相关类，但它在 SQLAlchemy 声明过程中尽可能地防范了尽可能多的情况。

    引用：[#4686](https://www.sqlalchemy.org/trac/ticket/4686)

+   **[orm] [bug]**

    现在对将瞬态对象与`Session.merge()`合并到`Session`中已经是瞬态的对象的情况发出警告。这为通常会导致对象被双重插入的情况发出警告。

    引用：[#4647](https://www.sqlalchemy.org/trac/ticket/4647)

+   **[orm] [bug]**

    修复了在新关系 m2o 比较逻辑中引入的回归问题，该逻辑首次出现在改进了一对多查询表达式的行为时，当与在映射实例中处于未获取状态的持久化为 NULL 的属性进行比较时。由于属性没有明确的默认值，在持久化设置中访问时需要默认为 NULL。

    参考：[#4676](https://www.sqlalchemy.org/trac/ticket/4676)

### 引擎

+   **[engine] [bug] [postgresql]**

    将在方言初始化期间发生的“回滚”移动到额外的方言特定初始化步骤之后，特别是 psycopg2 方言的初始化步骤，该步骤会在第一个新连接上无意中保留事务状态，这可能会干扰一些需要不启动事务的 psycopg2 特定 API。感谢 Matthew Wilkes 提供的拉取请求。

    参考：[#4663](https://www.sqlalchemy.org/trac/ticket/4663)

### sql

+   **[sql] [bug]**

    修复了`GenericFunction`类无意中注册自身为命名函数的问题。感谢 Adrien Berchet 提供的拉取请求。

    参考：[#4653](https://www.sqlalchemy.org/trac/ticket/4653)

+   **[sql] [bug]**

    修复了布尔列的双重否定不会重置“NOT”运算符的问题。

    参考：[#4618](https://www.sqlalchemy.org/trac/ticket/4618)

+   **[sql] [bug]**

    `GenericFunction`命名空间正在迁移，以便以不区分大小写的方式查找函数名称，因为 SQL 函数不会因区分大小写而发生冲突，用户定义的函数或存储过程也不会发生这种情况。现在使用不区分大小写的方案查找使用`GenericFunction`声明的函数，但支持一个弃用案例，允许存在两个或更多具有不同大小写的相同名称的`GenericFunction`对象，这将导致对该特定名称进行区分大小写查找，同时在函数注册时发出警告。感谢 Adrien Berchet 在这个复杂功能上的大量工作。

    参考：[#4569](https://www.sqlalchemy.org/trac/ticket/4569)

### postgresql

+   **[postgresql] [bug] [orm]**

    修复了一个问题，即“匹配的行数”警告即使方言报告“supports_sane_multi_rowcount=False”也会发出，例如对于 psycogp2 与`use_batch_mode=True`和其他情况。

    参考：[#4661](https://www.sqlalchemy.org/trac/ticket/4661)

### mysql

+   **[mysql] [bug]**

    添加了对 DROP CHECK 约束的支持，MySQL 8.0.16 要求删除 CHECK 约束；MariaDB 支持普通的 DROP CONSTRAINT。该逻辑通过检查服务器版本字符串是否存在 MariaDB 来区分这两种语法。Alembic 迁移已经通过实现自己的 MySQL / MariaDB CHECK 约束删除来解决了这个问题，但是这个改变直接在 Core 中实现了这一功能，以便供一般使用。感谢 Hannes Hansen 的拉取请求。

    参考：[#4650](https://www.sqlalchemy.org/trac/ticket/4650)

### mssql

+   **[mssql] [功能]**

    添加了对 SQL Server 过滤索引的支持，通过 `mssql_where` 参数实现，其工作方式类似于 PostgreSQL 方言中的 `postgresql_where` 索引函数。

    参见

    过滤索引

    参考：[#4657](https://www.sqlalchemy.org/trac/ticket/4657)

+   **[mssql] [错误]**

    为 pymssql 添加了错误代码 20047 到“is_disconnect”。感谢 Jon Schuff 的拉取请求。

    参考：[#4680](https://www.sqlalchemy.org/trac/ticket/4680)

### 杂项

+   **[杂项] [错误]**

    从 MANIFEST.in 中删除了错误的“sqla_nose.py”符号，这会导致不必要的警告消息。

    参考：[#4625](https://www.sqlalchemy.org/trac/ticket/4625)

## 1.3.3

发布日期：2019 年 4 月 15 日

### orm

+   **[orm] [错误]**

    修复了 1.3 版本中新的“模糊 FROMs”查询逻辑中的回归问题，该问题是在 Query.join() 处理在决定“左”侧时的模糊性更加明确 中引入的，其中一个 `Query` 明确将一个实体放在 FROM 子句中，并使用 `Query.join()` 进行连接，如果该实体在额外的连接中使用，那么稍后将会导致“模糊 FROM”错误，因为该实体在 `Query` 的“from”列表中出现两次。修复此模糊性的方法是将独立实体合并到已经存在的连接中，就像在渲染 SELECT 语句时最终发生的那样。

    参考：[#4584](https://www.sqlalchemy.org/trac/ticket/4584)

+   **[orm] [错误]**

    调整了 `Query.filter_by()` 方法，不再在多个条件内部调用 `and()`，而是将其作为一系列条件传递给 `Query.filter()`，而不是单个条件。这允许 `Query.filter_by()` 推迟到 `Query.filter()` 处理可变数量的子句，包括列表为空的情况。在这种情况下，`Query` 对象将不会有 `.whereclause`，这允许后续的“无 whereclause”方法如 `Query.select_from()` 保持一致的行为。

    参考：[#4606](https://www.sqlalchemy.org/trac/ticket/4606)

### postgresql

+   **[postgresql] [bug]**

    从 1.3.2 版本中的回归修复引起的问题，该问题是由 [#4562](https://www.sqlalchemy.org/trac/ticket/4562) 引起的，其中包含了仅包含查询字符串而不包含主机名的 URL，例如用于指定包含连接信息的服务文件时，将不再正确地传播到 psycopg2。对 [#4562](https://www.sqlalchemy.org/trac/ticket/4562) 中的更改已经调整以进一步适应 psycopg2 的确切要求，即如果存在任何连接参数，那么“dsn”参数不再是必需的，因此在这种情况下，只传递查询字符串参数。

    参考：[#4601](https://www.sqlalchemy.org/trac/ticket/4601)

### mssql

+   **[mssql] [bug]**

    修复了 SQL Server 方言中的问题，即如果 ORDER BY 表达式中存在一个绑定参数，最终在 SQL Server 版本的语句中不会呈现，那么这些参数仍然会成为执行参数的一部分，导致 DBAPI 级别的错误。感谢 Matt Lewellyn 的拉取请求。

    参考：[#4587](https://www.sqlalchemy.org/trac/ticket/4587)

### 杂项

+   **[bug] [pool]**

    由于弃用了`Pool`的`use_threadlocal`标志，导致行为回归修复，`SingletonThreadPool`不再使用此选项，这会导致在事务的上下文中多次使用相同的`Engine`连接或隐式执行时发生“返回时回滚”逻辑，从而取消事务。虽然这不是与引擎和连接一起工作的推荐方式，但这仍然是一个令人困惑的行为变化，因为当使用`SingletonThreadPool`时，无论在同一线程中对相同引擎做了什么，事务都应该保持打开状态。`use_threadlocal`标志仍然被弃用，但`SingletonThreadPool`现在实现了自己版本的相同逻辑。

    参考：[#4585](https://www.sqlalchemy.org/trac/ticket/4585)

+   **[bug] [ext]**

    修复了在`MutableList`上使用`copy.copy()`或`copy.deepcopy()`会导致列表内的项目被复制的错误，这是由于 Python pickle 和 copy 在处理列表时如何使用`__getstate__()`和`__setstate__()`存在不一致性。为了解决这个问题，必须向`MutableList`添加一个`__reduce_ex__`方法。为了与基于`__getstate__()`的现有 pickle 保持向后兼容性，`__setstate__()`方法也保留；测试套件断言，对旧版本类进行的 pickle 仍然可以被 pickle 模块反序列化。

    参考：[#4603](https://www.sqlalchemy.org/trac/ticket/4603)

### orm

+   **[orm] [bug]**

    修复了新“模糊 FROMs”查询逻辑中的 1.3 回归，该逻辑是在 Query.join()更明确地处理决定“左”侧的模糊性中引入的，其中一个`Query`在 FROM 子句中明确放置一个实体，并使用`Query.join()`进行连接，如果该实体在额外的连接中使用，那么稍后将导致“模糊 FROM”错误，因为该实体在`Query`的“from”列表中出现两次。 该修复通过将独立实体折叠到已经是一部分的连接中来解决这种模糊性，就像在渲染 SELECT 语句时最终发生的那样。

    参考：[#4584](https://www.sqlalchemy.org/trac/ticket/4584)

+   **[orm] [bug]**

    调整了`Query.filter_by()`方法，不再在多个条件内部调用`and()`，而是将其作为一系列条件传递给`Query.filter()`，而不是单个条件。 这允许`Query.filter_by()`推迟到`Query.filter()`对变量数量的子句的处理，包括列表为空的情况。 在这种情况下，`Query`对象将不具有`.whereclause`，这允许随后的“无 whereclause”方法（如`Query.select_from()`）保持一致的行为。

    参考：[#4606](https://www.sqlalchemy.org/trac/ticket/4606)

### postgresql

+   **[postgresql] [bug]**

    修复了从版本 1.3.2 发布引起的回归，原因是[#4562](https://www.sqlalchemy.org/trac/ticket/4562)，其中包含仅查询字符串而没有主机名的 URL，例如用于指定包含连接信息的服务文件，将不再正确传播到 psycopg2。 [#4562](https://www.sqlalchemy.org/trac/ticket/4562)中的更改已经调整以进一步适应 psycopg2 的确切要求，即如果有任何连接参数，那么“dsn”参数将不再是必需的，因此在这种情况下，仅传递查询字符串参数。

    参考：[#4601](https://www.sqlalchemy.org/trac/ticket/4601)

### mssql

+   **[mssql] [bug]**

    修复了 SQL Server 方言中的问题，如果 ORDER BY 表达式中存在一个绑定参数，最终在 SQL Server 版本的语句中不会呈现，那么这些参数仍然会成为执行参数的一部分，导致 DBAPI 级别的错误。感谢 Matt Lewellyn 的拉取请求。

    参考：[#4587](https://www.sqlalchemy.org/trac/ticket/4587)

### misc

+   **[bug] [pool]**

    修复了行为回归问题，因为取消了对`Pool`的`use_threadlocal`标志，`SingletonThreadPool`不再使用此选项，导致在事务的上下文中多次使用相同的`Engine`连接或隐式执行时发生“回滚返回”逻辑，从而取消事务。虽然这不是推荐的引擎和连接工作方式，但当使用`SingletonThreadPool`时，事务应该保持打开状态，无论在同一线程中对相同引擎做了什么。`use_threadlocal`标志仍然被弃用，但`SingletonThreadPool`现在实现了自己版本的相同逻辑。

    参考：[#4585](https://www.sqlalchemy.org/trac/ticket/4585)

+   **[bug] [ext]**

    修复了在对`MutableList`使用`copy.copy()`或`copy.deepcopy()`时出现的 bug，由于 Python 的 pickle 和 copy 在处理列表时使用`__getstate__()`和`__setstate__()`存在不一致性，导致列表内的项目被复制。为了解决这个问题，必须向`MutableList`添加一个`__reduce_ex__`方法。为了与基于`__getstate__()`的现有 pickle 兼容，`__setstate__()`方法也保留了；测试套件断言，对旧版本类进行的 pickle 仍然可以被 pickle 模块反序列化。

    参考：[#4603](https://www.sqlalchemy.org/trac/ticket/4603)

## 1.3.2

发布日期：2019 年 4 月 2 日

### orm

+   **[orm] [bug] [ext]**

    恢复了对纯 Python 描述符（例如`@property`对象）的实例级支持，与关联代理一起使用时，如果代理对象根本不在 ORM 范围内，则被归类为“模糊”，但直接被代理。对于类级别访问，基本类级别``__get__()``现在直接返回`AmbiguousAssociationProxyInstance`，而不是引发其异常，这是返回可能的最接近以前返回`AssociationProxy`本身的行为的近似值。还改进了这些对象的字符串表示，以更具描述性地反映当前状态。

    参考：[#4573](https://www.sqlalchemy.org/trac/ticket/4573), [#4574](https://www.sqlalchemy.org/trac/ticket/4574)

+   **[orm] [bug]**

    修复了一个 bug，当使用`with_polymorphic()`或其他别名构造时，当别名目标被用作子查询中的`column_property()`的`Select.correlate_except()`目标时，适配不正确。这需要修复子句适配机制，以正确处理出现在“除了关联”列表中的可选择项，类似于出现在“关联”列表中的可选择项的方式。这实际上是一个相当基本的 bug，已经存在很长时间，但很难遇到。

    参考：[#4537](https://www.sqlalchemy.org/trac/ticket/4537)

+   **[orm] [bug]**

    修复了一个回归问题，当尝试将关系选项链接到一个未使用`PropComparator.of_type()`的 AliasedClass 时，本应引发一个新的错误消息，而实际上会引发`AttributeError`。请注意，在 1.3 版本中，从普通映射关系到`AliasedClass`创建选项路径而不使用`PropComparator.of_type()`已不再有效。

    参考：[#4566](https://www.sqlalchemy.org/trac/ticket/4566)

### sql

+   **[sql] [bug] [documentation]**

    多亏了 TypeEngine methods bind_expression, column_expression work with Variant, type-specific types，我们不再需要依赖直接子类化特定方言类型的配方，`TypeDecorator`现在可以处理所有情况。此外，上述更改使得直接子类化基本 SQLAlchemy 类型的直接子类工作的可能性稍微降低，这可能会产生误导。文档已更新，以在这些示例中使用`TypeDecorator`，包括 PostgreSQL 的“ArrayOfEnum”示例数据类型，并且已删除了对“直接子类化类型”的直接支持。

    参考：[#4580](https://www.sqlalchemy.org/trac/ticket/4580)

### postgresql

+   **[postgresql] [功能]**

    为 psycopg2 方言添加了对无参数连接 URL 的支持，这意味着可以将 URL 作为`"postgresql+psycopg2://"`传递给`create_engine()`，而不需要额外的参数来指示传递给 libpq 的空 DSN，这表示连接到“localhost”而不提供用户名、密码或数据库。感谢 Julian Mehnle 提交的拉取请求。

    参考：[#4562](https://www.sqlalchemy.org/trac/ticket/4562)

+   **[postgresql] [错误]**

    修改了`Select.with_for_update.of`参数，如果传递了联接或其他组合可选择的对象，则将从中过滤出各个`Table`对象，允许将 join()对象传递给参数，就像在使用 ORM 时正常情况下使用联接表继承一样。感谢 Raymond Lu 提交的拉取请求。

    参考：[#4550](https://www.sqlalchemy.org/trac/ticket/4550)

### orm

+   **[orm] [错误] [扩展]**

    恢复了对纯 Python 描述符（例如`@property`对象）的实例级支持，与关联代理一起使用，如果代理对象根本不在 ORM 范围内，则被归类为“模糊”，但直接进行代理。对于类级别访问，基本类级别``__get__()``现在直接返回`AmbiguousAssociationProxyInstance`，而不是引发其异常，这是返回可能的最接近以前返回`AssociationProxy`本身的行为的近似值。还改进了这些对象的字符串表��，以更具描述性地反映当前状态。

    参考：[#4573](https://www.sqlalchemy.org/trac/ticket/4573)，[#4574](https://www.sqlalchemy.org/trac/ticket/4574)

+   **[orm] [错误]**

    修复了一个错误，当使用`with_polymorphic()`或其他别名构造时，当别名目标被用作子查询中的`Select.correlate_except()`目标时，适配不会正确进行。这需要修复子句适配机制，以正确处理出现在“除外关联”列表中的可选择项，类似于出现在“关联”列表中的可选择项的方式。这实际上是一个相当基本的长期存在的 bug，但很难遇到它。

    参考：[#4537](https://www.sqlalchemy.org/trac/ticket/4537)

+   **[orm] [bug]**

    修复了一个回归，当尝试将一个关系选项链接到一个未使用`PropComparator.of_type()`的别名类（AliasedClass）时，本应该引发一个新的错误消息，而实际上会引发一个`AttributeError`。请注意，在 1.3 版本中，不再允许从普通的映射关系到一个`AliasedClass`创建选项路径，而不使用`PropComparator.of_type()`。

    参考：[#4566](https://www.sqlalchemy.org/trac/ticket/4566)

### sql

+   **[sql] [bug] [documentation]**

    由于 TypeEngine methods bind_expression, column_expression work with Variant, type-specific types 的改动，我们不再需要依赖直接子类化特定于方言的类型的食谱，`TypeDecorator`现在可以处理所有情况。此外，上述更改使得直接基于 SQLAlchemy 类型的子类的预期工作几率略微降低，这可能会误导。文档已更新为在这些示例中使用`TypeDecorator`，包括 PostgreSQL 的“ArrayOfEnum”示例数据类型，并且直接支持“直接子类化类型”的已被移除。

    参考：[#4580](https://www.sqlalchemy.org/trac/ticket/4580)

### postgresql

+   **[postgresql] [feature]**

    为 psycopg2 方言添加了无参数连接 URL 的支持，这意味着可以将 URL 作为`"postgresql+psycopg2://"`传递给`create_engine()`，而不需要额外的参数来指示传递给 libpq 的空 DSN，这表示连接到“localhost”而不提供用户名、密码或数据库。感谢 Julian Mehnle 的拉取请求。

    参考：[#4562](https://www.sqlalchemy.org/trac/ticket/4562)

+   **[postgresql] [bug]**

    修改了`Select.with_for_update.of`参数，以便如果传递了连接或其他组合可选择项，则将从中过滤出各个`Table`对象，允许将 join()对象传递给参数，就像在使用 ORM 时通常发生的那样。感谢 Raymond Lu 的拉取请求。

    参考：[#4550](https://www.sqlalchemy.org/trac/ticket/4550)

## 1.3.1

发布日期：2019 年 3 月 9 日

### orm

+   **[orm] [bug] [ext]**

    修复了关联代理链接到同义词时不再工作的回归，无论是在实例级别还是在类级别。

    参考：[#4522](https://www.sqlalchemy.org/trac/ticket/4522)

### mssql

+   **[mssql] [bug]**

    在将隔离级别更改为 SNAPSHOT 后会发出一个 commit()，因为 pyodbc 和 pymssql 都会打开一个隐式事务，这会阻止当前事务中发出后续的 SQL。

    此更改也**回溯**到：1.2.19

    参考：[#4536](https://www.sqlalchemy.org/trac/ticket/4536)

+   **[mssql] [bug]**

    修复了 SQL Server 反射中的回归，原因是[#4393](https://www.sqlalchemy.org/trac/ticket/4393)中从`Float`数据类型中删除了开放式`**kw`，导致此类型的反射失败，因为传递了一个“scale”参数。

    参考：[#4525](https://www.sqlalchemy.org/trac/ticket/4525)

### orm

+   **[orm] [bug] [ext]**

    修复了关联代理链接到同义词时不再工作的回归，无论是在实例级别还是在类级别。

    参考：[#4522](https://www.sqlalchemy.org/trac/ticket/4522)

### mssql

+   **[mssql] [bug]**

    在将隔离级别更改为 SNAPSHOT 后会发出一个 commit()，因为 pyodbc 和 pymssql 都会打开一个隐式事务，这会阻止当前事务中发出后续的 SQL。

    此更改也**回溯**到：1.2.19

    参考：[#4536](https://www.sqlalchemy.org/trac/ticket/4536)

+   **[mssql] [bug]**

    修复了 SQL Server 反射中的回归，原因是[#4393](https://www.sqlalchemy.org/trac/ticket/4393)中从`Float`数据类型中删除了开放式`**kw`，导致此类型的反射失败，因为传递了一个“scale”参数。

    参考：[#4525](https://www.sqlalchemy.org/trac/ticket/4525)

## 1.3.0

发布日期：2019 年 3 月 4 日

### orm

+   **[orm] [特性]**

    `Query.get()` 方法现在可以接受一个属性键和值的字典，作为指示要加载的主键值的手段；特别适用于复合主键。感谢 Sanjana S 提交的拉取请求。

    参考：[#4316](https://www.sqlalchemy.org/trac/ticket/4316)

+   **[orm] [特性]**

    现在可以将 SQL 表达式分配给 ORM 刷新中的主键属性，方式与普通属性描述的方式相同，如将 SQL 插入/更新表达式嵌入到刷新中，其中表达式将被评估，然后使用 RETURNING 返回给 ORM，或者在 pysqlite 的情况下，使用 cursor.lastrowid 属性工作。需要支持 RETURNING 的数据库（例如 Postgresql、Oracle、SQL Server）或 pysqlite。

    参考：[#3133](https://www.sqlalchemy.org/trac/ticket/3133)

### engine

+   **[engine] [特性]**

    修订了当字符串化时 `StatementError` 的格式。每个错误细节都分布在多个新行上，而不是在单行上间隔开。此外，SQL 表示现在将 SQL 语句字符串化，而不是使用 `repr()`，因此换行符会按原样呈现。感谢 Nate Clark 提交的拉取请求。

    另请参阅

    更改 StatementError 格式（换行和 %s）

    参考：[#4500](https://www.sqlalchemy.org/trac/ticket/4500)

### sql

+   **[sql] [错误]**

    `Alias` 类及相关子类 `CTE`、`Lateral` 和 `TableSample` 已经重新设计，用户不再能直接构造这些对象。这些构造需要使用独立的构造函数或可选择绑定的方法来实例化新对象。

    参考：[#4509](https://www.sqlalchemy.org/trac/ticket/4509)

### schema

+   **[schema] [特性]**

    添加了新参数`Table.resolve_fks`和`MetaData.reflect.resolve_fks`，当设置为 False 时，将禁用遇到的`ForeignKey`对象的自动反射，这既可以减少省略表的 SQL 开销，也可以避免由于数据库特定原因无法反射的表。同一`MetaData`集合中存在的两个`Table`对象仍然可以相互引用，即使两个表的反射是分开进行的。

    参考：[#4517](https://www.sqlalchemy.org/trac/ticket/4517)

### ORM

+   **[ORM] [特性]**

    `Query.get()`方法现在可以接受一个属性键和值的字典作为指示要加载的主键值的手段；对于复合主键特别有用。感谢 Sanjana S.的拉取请求。

    参考：[#4316](https://www.sqlalchemy.org/trac/ticket/4316)

+   **[ORM] [特性]**

    现在可以将 SQL 表达式分配给 ORM 刷新中的主键属性，方式与普通属性描述的方式相同，如将 SQL 插入/更新表达式嵌入到刷新中，其中表达式将被评估，然后使用 RETURNING 返回给 ORM，或者在 pysqlite 的情况下，使用 cursor.lastrowid 属性工作。需要支持 RETURNING 的数据库（例如 Postgresql、Oracle、SQL Server）或 pysqlite。

    参考：[#3133](https://www.sqlalchemy.org/trac/ticket/3133)

### 引擎

+   **[引擎] [特性]**

    修订了在字符串化时的`StatementError`格式。每个错误细节现在分布在多个新行上，而不是在单行上间隔开。此外，SQL 表示现在将 SQL 语句字符串化，而不是使用`repr()`，因此换行符将按原样呈现。感谢 Nate Clark 的拉取请求。

    另请参阅

    更改了 StatementError 的格式（换行和%s）

    参考：[#4500](https://www.sqlalchemy.org/trac/ticket/4500)

### SQL

+   **[SQL] [错误]**

    `Alias`类及其相关子类`CTE`、`Lateral`和`TableSample`已经重新设计，用户不再可以直接构造这些对象。这些构造要求使用独立的构造函数或可选择绑定方法来实例化新对象。

    参考：[#4509](https://www.sqlalchemy.org/trac/ticket/4509)

### 模式

+   **[schema] [feature]**

    添加了新参数`Table.resolve_fks`和`MetaData.reflect.resolve_fks`，当设置为 False 时，将禁用在`ForeignKey`对象中遇到的相关表的自动反射，这既可以减少省略表的 SQL 开销，也可以避免由于数据库特定原因无法反射的表。同一`MetaData`集合中存在的两个`Table`对象仍然可以相互引用，即使两个表的反射是分开进行的。

    参考：[#4517](https://www.sqlalchemy.org/trac/ticket/4517)

## 1.3.0b3

发布日期：2019 年 2 月 8 日

### orm

+   **[orm] [bug]**

    改进了`with_polymorphic()`与加载器选项一起使用的行为，特别是通配符操作以及`load_only()`。多态对象将更准确地被定位，以便实体上的列级选项能够正确生效。该问题是在[#4468](https://www.sqlalchemy.org/trac/ticket/4468)中修复的相同类型问题的延续。

    参考：[#4469](https://www.sqlalchemy.org/trac/ticket/4469)

### orm 声明

+   **[orm] [declarative] [bug]**

    添加了一些辅助异常，当基于`AbstractConcreteBase`、`DeferredReflection`或`AutoMap`的映射在映射准备好使用之前被使用时，这些异常包含有关类的描述性信息，而不是陷入其他信息较少的故障模式中。

    参考：[#4470](https://www.sqlalchemy.org/trac/ticket/4470)

### sql

+   **[sql] [bug]**

    完全删除了直接作为`select()`或`Query`对象组件传递的字符串被自动强制转换为`text()`构造的行为；现在发出的警告现在是一个 ArgumentError 或在 order_by() / group_by()的情况下是 CompileError。自 1.0 版本以来一直发出警告，但其存在继续引起对此行为潜在误用的担忧。

    请注意，已发布了关于 order_by() / group_by()的公共 CVE，这些 CVE 由此提交解决：CVE-2019-7164 CVE-2019-7548

    另请参阅

    完全删除将字符串 SQL 片段强制转换为 text()

    参考：[#4481](https://www.sqlalchemy.org/trac/ticket/4481)

+   **[sql] [bug]**

    引用应用于`Function`名称，这些名称通常但不一定是从`sqlalchemy.sql.expression.func`构造生成的，在编译时如果它们包含非法字符，比如空格或标点符号。这些名称仍然被视为不区分大小写，这意味着如果名称包含大写字母或混合大小写字符，仅此并不会触发引用。目前为了向后兼容性而保持不区分大小写。

    参考：[#4467](https://www.sqlalchemy.org/trac/ticket/4467)

+   **[sql] [bug]**

    添加了对被接受为纯字符串的关键 DDL 短语的“SQL 短语验证”，包括 `ForeignKeyConstraint.on_delete`、`ForeignKeyConstraint.on_update`、`ExcludeConstraint.using`、`ForeignKeyConstraint.initially` 等，用于预期仅有一系列 SQL 关键字的区域。任何非空格字符都暗示该短语需要引用，则会引发 `CompileError`。此更改与作为 [#4481](https://www.sqlalchemy.org/trac/ticket/4481) 一部分提交的一系列更改相关。

    参考：[#4481](https://www.sqlalchemy.org/trac/ticket/4481)

### postgresql

+   **[postgresql] [bug]**

    修复了使用大写名称作为索引类型（例如 GIST、BTREE 等）或 EXCLUDE 约束时将其视为要引用的标识符的问题，而不是直接呈现。新行为将这些类型转换为小写，并确保它们只包含有效的 SQL 字符。

    参考：[#4473](https://www.sqlalchemy.org/trac/ticket/4473)

### 测试

+   **[tests] [change]**

    测试系统已移除对多年未维护且在 Python 3 下产生警告的 Nose 的支持。测试套件目前标准化为 Pytest。感谢 Parth Shandilya 提交的拉取请求。

    参考：[#4460](https://www.sqlalchemy.org/trac/ticket/4460)

### 其他

+   **[bug] [ext]**

    当使用关联代理与集合或字典时，实现了更全面的赋值操作（例如“批量替换”）。修复了创建冗余代理对象以替换旧对象的问题，这导致了过多的事件和 SQL，在唯一约束的情况下将导致刷新失败。

    参见

    使用关联代理对集合、字典实现批量替换

    参考：[#2642](https://www.sqlalchemy.org/trac/ticket/2642)

### orm

+   **[orm] [bug]**

    改进了 `with_polymorphic()` 与加载器选项一起的行为，特别是通配符操作以及 `load_only()`。多态对象将更准确地被定位，以便实体的列级选项能够正确生效。该问题是在 [#4468](https://www.sqlalchemy.org/trac/ticket/4468) 中修复的相同类型的问题的延续。

    参考：[#4469](https://www.sqlalchemy.org/trac/ticket/4469)

### orm 声明式

+   **[orm] [declarative] [bug]**

    添加了一些辅助异常，当基于`AbstractConcreteBase`、`DeferredReflection`或`AutoMap`的映射在映射准备好使用之前被使用时，这些异常会被触发，其中包含有关类的描述性信息，而不是陷入其他不够信息丰富的失败模式中。

    参考：[#4470](https://www.sqlalchemy.org/trac/ticket/4470)

### sql

+   **[sql] [bug]**

    完全移除了直接作为`select()`或`Query`对象的组件传递的字符串被自动强制转换为`text()`构造的行为；自版本 1.0 以来已发出警告，但其存在继续引发对此行为潜在误用的担忧。

    请注意，已发布了关于 order_by() / group_by()的公共 CVE，这些 CVE 已通过此提交解决：CVE-2019-7164 CVE-2019-7548

    另请参阅

    将字符串 SQL 片段强制转换为 text()已完全移除

    参考：[#4481](https://www.sqlalchemy.org/trac/ticket/4481)

+   **[sql] [bug]**

    引用被应用于`Function`名称，这些名称通常但不一定是从`sqlalchemy.sql.expression.func`构造生成的，在编译时如果它们包含非法字符，比如空格或标点符号。这些名称仍然被视为不区分大小写，这意味着如果名称包含大写字母或混合大小写字符，仅此并不会触发引用。目前为了向后兼容性，大小写不敏感性仍然被保留。

    参考：[#4467](https://www.sqlalchemy.org/trac/ticket/4467)

+   **[sql] [bug]**

    添加了对被接受为纯字符串的关键 DDL 短语“SQL 短语验证”的支持，包括`ForeignKeyConstraint.on_delete`，`ForeignKeyConstraint.on_update`，`ExcludeConstraint.using`，`ForeignKeyConstraint.initially`等，用于期望一系列 SQL 关键字的地方。任何非空格字符表明该短语需要引号的情况将引发`CompileError`。此更改与提交的一系列更改相关，作为[#4481](https://www.sqlalchemy.org/trac/ticket/4481)的一部分。

    参考：[#4481](https://www.sqlalchemy.org/trac/ticket/4481)

### postgresql

+   **[postgresql] [bug]**

    修复了使用大写名称作为索引类型（例如 GIST、BTREE 等）或 EXCLUDE 约束时将其视为需要引用的标识符的问题，而不是按原样呈现。新行为将这些类型转换为小写，并确保它们只包含有效的 SQL 字符。

    参考：[#4473](https://www.sqlalchemy.org/trac/ticket/4473)

### 测试

+   **[tests] [change]**

    测试系统已移除对 Nose 的支持，Nose 已多年未维护，并在 Python 3 下产生警告。测试套件目前标准化为 Pytest。感谢 Parth Shandilya 的拉取请求。

    参考：[#4460](https://www.sqlalchemy.org/trac/ticket/4460)

### 杂项

+   **[bug] [ext]**

    当使用关联代理与集合或字典时，实现了更全面的赋值操作（例如“批量替换”）。修复了创建多余代理对象以替换旧对象的问题，这会导致事件和 SQL 过多，并且在唯一约束的情况下会导致刷新失败。

    另请参阅

    为 AssociationProxy 实现了集合、字典的批量替换

    参考：[#2642](https://www.sqlalchemy.org/trac/ticket/2642)

## 1.3.0b2

发布日期：2019 年 1 月 25 日

### 一般

+   **[general] [change]**

    在整个库中进行了大规模的更改，确保所有被标记为已弃用或遗留的对象、参数和行为在调用时都会发出`DeprecationWarning`警告。由于 Python 3 解释器现在默认显示弃用警告，以及基于像 tox 和 pytest 这样的现代测试套件 tend to 显示弃用警告，这个更改应该使得更容易注意到哪些 API 功能已经过时。这个更改的一个主要原因是，长期被弃用的功能，尽管仍然在实际应用中使用，但最终将在不久的将来被移除；其中最大的例子是自版本 0.7 以来就已被弃用但仍然存在于库中的`SessionExtension`和`MapperExtension`类，以及少数其他的预事件扩展钩子。另一个是，还将弃用几个长期存在的行为，包括线程本地引擎策略、convert_unicode 标志和非主映射器。

    参见

    对所有已弃用元素发出弃用警告；添加新的弃用

    参考：[#4393](https://www.sqlalchemy.org/trac/ticket/4393)

### orm

+   **[orm] [feature]**

    实现了一个新功能，可以将`AliasedClass`构造用作`relationship()`的目标。这样就不再需要“非主映射器”的概念，因为`AliasedClass`更容易配置，并自动继承了映射类的所有关系，同时保留了加载器选项正常工作的能力。

    参见

    关系到 AliasedClass 取代了非主映射器的需要

    参考：[#4423](https://www.sqlalchemy.org/trac/ticket/4423)

+   **[orm] [feature]**

    添加了新的`MapperEvents.before_mapper_configured()`事件。这个事件与其他“配置”阶段的映射器事件相辅相成，接收每个`Mapper`在其配置步骤之前的事件，并且可以用于阻止或延迟特定`Mapper`对象的配置，使用新的返回值`interfaces.EXT_SKIP`。请参考文档链接获取示例。

    参见

    `MapperEvents.before_mapper_configured()`

    参考：[#4397](https://www.sqlalchemy.org/trac/ticket/4397)

+   **[orm] [change]**

    添加了一个新函数`close_all_sessions()`，它接管了`Session.close_all()`方法的任务，后者现已被弃用，因为这会让人误解为类方法。感谢 Augustin Trancart 提供的拉取请求。

    参考：[#4412](https://www.sqlalchemy.org/trac/ticket/4412)

+   **[orm] [bug]**

    修复了长期存在的问题，即重复的集合成员会导致反向引用在删除其中一个重复项时删除成员与其父对象之间的关联，就像在一条语句中交换两个对象的副作用一样。

    另请参阅

    在删除操作期间检查多对一反向引用的集合重复项

    参考：[#1103](https://www.sqlalchemy.org/trac/ticket/1103)

+   **[orm] [bug]**

    将首次作为[#3287](https://www.sqlalchemy.org/trac/ticket/3287)的一部分进行的修复扩展，其中针对使用通配符的子类的加载器选项将扩展自身以包括将通配符应用于超类属性的“绑定”加载器选项，例如在表达式中`Load(SomeSubClass).load_only('foo')`。`SomeSubClass`的父类的列也将被排除，就像使用未绑定选项`load_only('foo')`一样。

    参考：[#4373](https://www.sqlalchemy.org/trac/ticket/4373)

+   **[orm] [bug]**

    在 ORM 领域改进了由加载器选项遍历引发的错误消息。这包括早期检测到不匹配的加载器策略，以及更清晰地解释为什么这些策略不匹配。

    参考：[#4433](https://www.sqlalchemy.org/trac/ticket/4433)

+   **[orm] [bug]**

    在`collection.remove()`方法中，现在在删除项目之前调用“remove”集合事件，这与大多数其他形式的集合项目删除行为一致（例如`__delitem__`、`__setitem__`下的替换）。对于`pop()`方法，删除事件仍然在操作之后触发。

+   **[orm] [bug] [engine]**

    为 Core 和 ORM 添加了执行选项的访问器，通过 `Query.get_execution_options()`、`Connection.get_execution_options()`、`Engine.get_execution_options()` 和 `Executable.get_execution_options()`。感谢 Daniel Lister 提交的 PR。

    参考：[#4464](https://www.sqlalchemy.org/trac/ticket/4464)

+   **[orm] [bug]**

    由于 [#3423](https://www.sqlalchemy.org/trac/ticket/3423) 导致关联代理中存在问题，该问题导致自定义 `PropComparator` 对象与混合属性（例如在 `dictlike-polymorphic` 示例中演示的对象）在关联代理中无法正常工作。在 [#3423](https://www.sqlalchemy.org/trac/ticket/3423) 中添加的严格性已经放宽，并添加了额外的逻辑以适应关联代理链接到自定义混合的情况。

    参考：[#4446](https://www.sqlalchemy.org/trac/ticket/4446)

+   **[orm] [bug]**

    实现了 `.get_history()` 方法，这也意味着 `synonym()` 属性的可用性，以前，尝试通过同义词访问属性历史会引发 `AttributeError`。

    参考：[#3777](https://www.sqlalchemy.org/trac/ticket/3777)

### orm 声明式

+   **[bug] [orm declarative]**

    为 `ColumnProperty` 添加了 `__clause_element__()` 方法，该方法可以在声明式映射类中更友好地使用未完全声明的列或延迟属性，当它在类声明中用于约束或其他基于列的场景时，虽然这仍无法在开放式表达式中工作；如果收到 `TypeError`，建议调用 `ColumnProperty.expression` 属性。

    参考：[#4372](https://www.sqlalchemy.org/trac/ticket/4372)

### engine

+   **[engine] [feature]**

    添加了公共访问器 `QueuePool.timeout()`，用于返回 `QueuePool` 对象的配置超时时间。感谢 Irina Delamare 提交的拉取请求。

    参考文献：[#3689](https://www.sqlalchemy.org/trac/ticket/3689)

+   **[engine] [change]**

    自 SQLAlchemy 大约版本 0.2 起就是遗留功能的“threadlocal”引擎策略现已弃用，以及 `Pool` 的 `Pool.threadlocal` 参数，在大多数现代用例中没有效果。

    请参阅

    “threadlocal” 引擎策略已弃用

    参考文献：[#4393](https://www.sqlalchemy.org/trac/ticket/4393)

### sql

+   **[sql] [feature]**

    修改了 `AnsiFunction` 类，这是像 `CURRENT_TIMESTAMP` 这样的常见 SQL 函数的基类，以接受像常规即席函数一样的位置参数。这适用于特定后端上许多这些函数接受“分数秒”精度等参数的情况。如果函数是带参数创建的，则呈现括号和参数。如果没有参数，则编译器生成非括号形式。

    参考文献：[#4386](https://www.sqlalchemy.org/trac/ticket/4386)

+   **[sql] [change]**

    `create_engine.convert_unicode` 和 `String.convert_unicode` 参数已经弃用。这些参数是在大多数 Python DBAPI 几乎不支持 Python Unicode 对象时构建的，而 SQLAlchemy 需要以高性能的方式在整个系统中处理 Unicode 和字节串之间的数据和 SQL 字符串的复杂任务。多亏了 Python 3，DBAPI 被迫适应了 Unicode 感知的 API，今天由 SQLAlchemy 支持的所有 DBAPI 都原生支持 Unicode，包括在 Python 2 上，这样就可以最终（大部分）删除这个长期存在且非常复杂的功能。当然，在一些 Python 2 的边缘情况下，SQLAlchemy 仍然需要处理 Unicode，但这些情况都是自动处理的；在现代使用中，用户不应该与这些标志进行交互。

    请参阅

    弃用 convert_unicode 参数

    参考文献：[#4393](https://www.sqlalchemy.org/trac/ticket/4393)

### mssql

+   **[mssql] [bug]**

    `Unicode` 和 `UnicodeText` 数据类型的 `literal_processor` 现在在 SQL 表达式中呈现一个 `N` 字符，这是 SQL Server 要求的用于呈现 SQL 表达式中的 Unicode 字符串值的方式。

    参考文献：[#4442](https://www.sqlalchemy.org/trac/ticket/4442)

### 杂项

+   **[bug] [ext]**

    修复了 1.3.0b1 中由[#3423](https://www.sqlalchemy.org/trac/ticket/3423)引起的回归，其中访问仅存在于多态子类上的属性的关联代理对象会引发`AttributeError`，尽管实际被访问的实例是该子类的实例。

    参考：[#4401](https://www.sqlalchemy.org/trac/ticket/4401)

### 通用

+   **[general] [change]**

    在整个库中进行了大规模的更改，确保所有被标记为弃用或遗留的对象、参数和行为在调用时现在会发出`DeprecationWarning`警告。由于 Python 3 解释器现在默认显示弃用警告，以及基于像 tox 和 pytest 这样的现代测试套件 tend to 显示弃用警告，这个更改应该使得更容易注意到哪些 API 功能已经过时。这个更改的一个主要原因是，长期弃用的功能，尽管仍然在实际应用中使用，但最终还是会在不久的将来被移除；其中最大的例子是`SessionExtension`和`MapperExtension`类以及一些其他自版本 0.7 以来就已被弃用但仍然存在于库中的预事件扩展钩子。另一个是，还将弃用几个长期存在的行为，包括线程本地引擎策略、convert_unicode 标志和非主映射器。

    另请参阅

    对所有弃用元素发出弃用警告；添加新的弃用

    参考：[#4393](https://www.sqlalchemy.org/trac/ticket/4393)

### orm

+   **[orm] [feature]**

    实现了一个新功能，使得`AliasedClass`构造现在可以作为`relationship()`的目标使用。这样就不再需要“非主映射器”的概念，因为`AliasedClass`更容易配置，并且自动继承了映射类的所有关系，同时保留了加载器选项正常工作的能力。

    另请参阅

    AliasedClass 替代非主映射器的关系

    参考：[#4423](https://www.sqlalchemy.org/trac/ticket/4423)

+   **[orm] [feature]**

    添加了新的`MapperEvents.before_mapper_configured()`事件。该事件与其他“配置”阶段的映射器事件相辅相成，接收每个`Mapper`在其配置步骤之前的事件，并且还可以用于阻止或延迟特定`Mapper`对象的配置，使用新的返回值`interfaces.EXT_SKIP`。请参阅文档链接以获取示例。

    参见

    `MapperEvents.before_mapper_configured()`

    参考：[#4397](https://www.sqlalchemy.org/trac/ticket/4397)

+   **[orm] [change]**

    添加了一个新函数`close_all_sessions()`，它接管了`Session.close_all()`方法的任务，该方法现已被弃用，因为这会让人误解为一个类方法。感谢 Augustin Trancart 的拉取请求。

    参考：[#4412](https://www.sqlalchemy.org/trac/ticket/4412)

+   **[orm] [bug]**

    修复了长期存在的问题，即重复集合成员会导致反向引用在删除其中一个重复项时删除成员与其父对象之间的关联，这是在一条语句中交换两个对象的副作用。

    参见

    删除操作期间的一对多反向引用检查集合重复项

    参考：[#1103](https://www.sqlalchemy.org/trac/ticket/1103)

+   **[orm] [bug]**

    扩展了首次作为[#3287](https://www.sqlalchemy.org/trac/ticket/3287)的一部分进行的修复，其中针对使用通配符的子类进行的加载器选项将扩展到包括对超类属性应用通配符的情况，以及“绑定”加载器选项，例如在表达式中`Load(SomeSubClass).load_only('foo')`。`SomeSubClass`的父类的列也将被排除，就像使用未绑定选项`load_only('foo')`一样。

    参考：[#4373](https://www.sqlalchemy.org/trac/ticket/4373)

+   **[orm] [bug]**

    改进了 ORM 在加载器选项遍历领域发出的错误消息。这包括对不匹配的加载器策略的早期检测，以及更清晰地解释为什么这些策略不匹配。

    参考：[#4433](https://www.sqlalchemy.org/trac/ticket/4433)

+   **[orm] [bug]**

    在`collection.remove()`方法中，现在在删除项目之前调用集合的“remove”事件，这与大多数其他形式的集合项目删除行为一致（例如`__delitem__`、`__setitem__`下的替换）。对于`pop()`方法，删除事件仍然在操作之后触发。

+   **[orm] [bug] [engine]**

    通过`Query.get_execution_options()`、`Connection.get_execution_options()`、`Engine.get_execution_options()`和`Executable.get_execution_options()`为 Core 和 ORM 添加了执行选项的访问器。PR 由 Daniel Lister 提供。

    参考：[#4464](https://www.sqlalchemy.org/trac/ticket/4464)

+   **[orm] [bug]**

    修复了与[#3423](https://www.sqlalchemy.org/trac/ticket/3423)相关的关联代理中的问题，该问题导致使用自定义`PropComparator`对象与混合属性（例如在`dictlike-polymorphic`示例中演示的属性）在关联代理中无法正常工作。在[#3423](https://www.sqlalchemy.org/trac/ticket/3423)中添加的严格性已经放宽，并且添加了额外的逻辑以适应关联到自定义混合的关联代理。

    参考：[#4446](https://www.sqlalchemy.org/trac/ticket/4446)

+   **[orm] [bug]**

    实现了`.get_history()`方法，这也意味着对于`synonym()`属性的`AttributeState.history`的可用性。以前，尝试通过同义词访问属性历史会引发`AttributeError`。

    参考：[#3777](https://www.sqlalchemy.org/trac/ticket/3777)

### orm 声明式

+   **[bug] [orm 声明式]**

    为`ColumnProperty`添加了一个`__clause_element__()`方法，当在声明映射类中的约束或其他基于列的场景中使用未完全声明的列或延迟属性时，这可以使其在类声明中稍微更友好，尽管这仍然无法在开放式表达式中工作；如果收到`TypeError`，请优先调用`ColumnProperty.expression`属性。

    参考：[#4372](https://www.sqlalchemy.org/trac/ticket/4372)

### engine

+   **[engine] [功能]**

    添加了公共访问器`QueuePool.timeout()`，返回`QueuePool`对象的配置超时时间。感谢 Irina Delamare 的拉取请求。

    参考：[#3689](https://www.sqlalchemy.org/trac/ticket/3689)

+   **[engine] [更改]**

    “threadlocal”引擎策略自 SQLAlchemy 大约 0.2 版本以来一直是一个传统功能，现已被弃用，以及`Pool`的`Pool.threadlocal`参数在大多数现代用例中没有效果。

    另请参阅

    “threadlocal”引擎策略已弃用

    参考：[#4393](https://www.sqlalchemy.org/trac/ticket/4393)

### sql

+   **[sql] [功能]**

    修改了`AnsiFunction`类，这是常见 SQL 函数的基础，如`CURRENT_TIMESTAMP`，以接受位置参数，就像常规的临时函数一样。这样可以适应许多特定后端的这些函数接受参数，例如“分数秒”精度等情况。如果函数带有参数创建，它会呈现括号和参数。如果没有参数，则编译器生成非括号形式。

    参考：[#4386](https://www.sqlalchemy.org/trac/ticket/4386)

+   **[sql] [更改]**

    `create_engine.convert_unicode`和`String.convert_unicode`参数已被弃用。这些参数是在大多数 Python DBAPI 几乎不支持 Python Unicode 对象时构建的，而 SQLAlchemy 需要以高效的方式在 Unicode 和字节字符串之间传递数据和 SQL 字符串。由于 Python 3，DBAPI 被迫适应了支持 Unicode 的 API，今天 SQLAlchemy 支持的所有 DBAPI 都原生支持 Unicode，包括在 Python 2 上，允许这个长期存在且非常复杂的功能最终被（大部分）移除。当然，在一些 Python 2 的边缘情况下，SQLAlchemy 仍然需要处理 Unicode，但这些都是自动处理的；在现代用法中，用户不应该需要与这些标志进行交互。

    另请参阅

    convert_unicode 参数已弃用

    参考：[#4393](https://www.sqlalchemy.org/trac/ticket/4393)

### mssql

+   **[mssql] [错误]**

    The `literal_processor` for the `Unicode` and `UnicodeText` datatypes now render an `N` character in front of the literal string expression as required by SQL Server for Unicode string values rendered in SQL expressions.

    References: [#4442](https://www.sqlalchemy.org/trac/ticket/4442)

### misc

+   **[bug] [ext]**

    Fixed a regression in 1.3.0b1 caused by [#3423](https://www.sqlalchemy.org/trac/ticket/3423) where association proxy objects that access an attribute that’s only present on a polymorphic subclass would raise an `AttributeError` even though the actual instance being accessed was an instance of that subclass.

    References: [#4401](https://www.sqlalchemy.org/trac/ticket/4401)

## 1.3.0b1

Released: November 16, 2018

### orm

+   **[orm] [feature]**

    Added new feature `Query.only_return_tuples()`. Causes the `Query` object to return keyed tuple objects unconditionally even if the query is against a single entity. Pull request courtesy Eric Atkin.

    This change is also **backported** to: 1.2.5

+   **[orm] [feature]**

    Added new flag `Session.bulk_save_objects.preserve_order` to the `Session.bulk_save_objects()` method, which defaults to True. When set to False, the given mappings will be grouped into inserts and updates per each object type, to allow for greater opportunities to batch common operations together. Pull request courtesy Alessandro Cucci.

+   **[orm] [feature]**

    The “selectin” loader strategy now omits the JOIN in the case of a simple one-to-many load, where it instead relies loads only from the related table, relying upon the foreign key columns of the related table in order to match up to primary keys in the parent table. This optimization can be disabled by setting the `relationship.omit_join` flag to False. Many thanks to Jayson Reis for the efforts on this.

    See also

    selectin loading no longer uses JOIN for simple one-to-many

    References: [#4340](https://www.sqlalchemy.org/trac/ticket/4340)

+   **[orm] [feature]**

    Added `.info` dictionary to the `InstanceState` class, the object that comes from calling `inspect()` on a mapped object.

    See also

    info dictionary added to InstanceState

    参考：[#4257](https://www.sqlalchemy.org/trac/ticket/4257)

+   **[orm] [bug]**

    修复了在与`Query.join()`以及`Query.select_entity_from()`结合使用`Lateral`构造时，右侧的 join 不会应用子句适应的 bug。 “lateral”引入了 join 右侧可关联的用例。以前，未考虑适应此子句。请注意，在 1.2 版本中，由`Query.subquery()`引入的可选择项仍未适应，因为[#4304](https://www.sqlalchemy.org/trac/ticket/4304)；可选择项需要由`select()`函数生成，以成为“lateral” join 的右侧。

    此更改也**回溯**到：1.2.12

    参考：[#4334](https://www.sqlalchemy.org/trac/ticket/4334)

+   **[orm] [bug]**

    修复了关于`passive_deletes=”all”`的问题，即对象的外键属性在从其父集合中移除后仍保持其值。以前，工作单元会将其设置为 NULL，尽管`passive_deletes`指示不应修改它。

    另请参见

    `passive_deletes=’all’`将使从集合中移除的对象的 FK 保持不变

    参考：[#3844](https://www.sqlalchemy.org/trac/ticket/3844)

+   **[orm] [bug]**

    改进了与关系绑定的多对一对象表达式的行为，使得在相关对象上检索列值现在对于对象从其父`Session`分离后仍具有弹性，即使属性已过期。在`InstanceState`内部使用了新功能来记忆特定列属性在其过期之前的最后已知值，以便在对象分离和过期同时发生时表达式仍然可以评估。使用现代属性状态功能改进了错误条件，以根据需要生成更具体的消息。

    另请参见

    改进了多对一查询表达式的行为

    参考：[#4359](https://www.sqlalchemy.org/trac/ticket/4359)

+   **[orm] [bug] [mysql] [postgresql]**

    在某些情况下，ORM 现在会在子查询中对“FOR UPDATE”子句进行加倍渲染，因为观察到 MySQL 不会锁定子查询的行。这意味着查询将呈现两个 FOR UPDATE 子句；请注意，在某些后端（例如 Oracle）上，由于不必要，子查询上的 FOR UPDATE 子句会被静默忽略。此外，在主要与 PostgreSQL 一起使用的“OF”子句的情况下，仅在使用此子句时，才会在内部子查询中呈现 FOR UPDATE，以便可选择地将可选择的目标定位到 SELECT 语句中的表。

    另请参阅

    FOR UPDATE 子句在连接的预加载子查询中以及外部进行渲染

    参考：[#4246](https://www.sqlalchemy.org/trac/ticket/4246)

+   **[orm] [bug]**

    对 `Query.join()` 进行了重构，进一步澄清了结构化连接的各个组成部分。此重构增加了对于 `Query.join()` 的功能，当 FROM 列表中存在多个元素或查询涉及多个实体时，它可以确定连接的最适当的“左”侧。如果有多个 FROM/实体匹配，将引发错误，要求指定 ON 子句以解决歧义。特别是针对我们在 [#4363](https://www.sqlalchemy.org/trac/ticket/4363) 中看到的回归，但也具有一般用途。 `Query.join()` 中的代码路径现在更容易理解，并且错误情况更具体地在操作的较早阶段决定。

    另请参阅

    Query.join() 在更明确地决定“左”侧的歧义方面进行了处理

    参考资料：[#4365](https://www.sqlalchemy.org/trac/ticket/4365)

+   **[orm] [bug]**

    修复了 `Query` 中的一个长期存在的问题，即标量子查询（例如由 `Query.exists()`、`Query.as_scalar()` 以及其他从 `Query.statement` 派生的查询生成的）在被用于需要实体适配的新 `Query` 中（例如当查询被转换为 union 或 from_self() 等时）不会被正确适配。此更改从由 `Query.statement` 访问器生成的 `select()` 对象中移除了“无适配”注释。

    参考：[#4304](https://www.sqlalchemy.org/trac/ticket/4304)

+   **[orm] [bug]**

    在 Python 中，在 ORM 刷新期间，当主键值不可排序时，会引发一个信息性异常，比如一个没有`__lt__()`方法的`Enum`；通常情况下，Python 3 会在这种情况下引发一个`TypeError`。在 Python 中，刷新过程按照主键对持久化对象进行排序，因此这些值必须是可排序的。

    参考：[#4232](https://www.sqlalchemy.org/trac/ticket/4232)

+   **[orm] [bug]**

    移除了`MappedCollection`类使用的集合转换器。此转换器仅用于断言传入的字典键与其对应对象的键匹配，并且仅在批量设置操作期间使用。该转换器可能会干扰自定义验证器或想要进一步转换传入值的 `AttributeEvents.bulk_replace()` 监听器。当传入的键与值不匹配时，此转换器将引发的`TypeError`被移除；在批量赋值期间，传入的值将被键入其生成的键，而不是显式存在于字典中的键。

    总的来说，@converter 被 `AttributeEvents.bulk_replace()` 事件处理程序所取代，这是作为 [#3896](https://www.sqlalchemy.org/trac/ticket/3896) 的一部分添加的。

    参考：[#3604](https://www.sqlalchemy.org/trac/ticket/3604)

+   **[orm] [bug]**

    添加了一种新的行为，当检索到多对一的“old”值时，会触发延迟加载，这样就可以跳过由于`lazy="raise"`或分离会话错误而引发的异常。

    另请参见

    多对一替换不会对“raiseload”或“old”对象引发异常

    参考：[#4353](https://www.sqlalchemy.org/trac/ticket/4353)

+   **[orm] [bug]**

    ORM 中长期存在的一个疏忽是，对于一对多关系的`__delete__`方法是无效的，例如对于`del a.b`这样的操作。现在已经实现了这一功能，并且等同于将属性设置为`None`。

    另请参阅

    为 ORM 属性实现了“del”操作

    参考：[#4354](https://www.sqlalchemy.org/trac/ticket/4354)

### orm declarative

+   **[orm] [declarative] [bug]**

    修复了声明式在调用并记忆化映射属性集合后，当添加或删除其他属性后不会更新`Mapper`状态的 bug。此外，如果从当前已映射的类中删除了完全映射的属性（例如列、关系等），则现在会引发`NotImplementedError`，因为如果删除了属性，则映射器将无法正确运行。

    参考：[#4133](https://www.sqlalchemy.org/trac/ticket/4133)

### engine

+   **[engine] [feature]**

    在`QueuePool`中添加了新的“lifo”模式，通常通过将标志`create_engine.pool_use_lifo`设置为 True 来启用。 “lifo”模式意味着刚刚检入的相同连接将首先被再次检出，允许在池仅部分利用时从服务器端清理多余的连接。感谢 Taem Park 提交的拉取请求。

    另请参阅

    队列池的新后进先出策略

### sql

+   **[sql] [feature]**

    重构了`SQLCompiler`以公开类似于`SQLCompiler.order_by_clause()`和`SQLCompiler.limit_clause()`方法的`SQLCompiler.group_by_clause()`方法，可以被方言重写以自定义 GROUP BY 的渲染方式。感谢 Samuel Chou 提交的拉取请求。

    此更改也**回溯**到：1.2.13

+   **[sql] [feature]**

    在“字符串 SQL”系统中添加了`Sequence`，当在没有方言的情况下将包含“序列下一个值”表达式的语句字符串化时，会生成一个有意义的字符串表达式（`"<next sequence value: my_sequence>"`），而不是引发编译错误。

    参考：[#4144](https://www.sqlalchemy.org/trac/ticket/4144)

+   **[sql] [feature]**

    添加了新的命名约定标记`column_0N_name`、`column_0_N_name`等，将为特定约束中引用的所有列的名称/键/标签生成名称。为了适应这种命名约定的长度，SQL 编译器的自动截断功能现在也适用于约束名称，这将为约束创建一个缩短的、确定性生成的名称，该名称将适用于目标后端而不会超过该后端的字符限制。

    此更改还修复了另外两个问题。一个是`column_0_key`标记尽管已经记录在案，但却无法使用，另一个是如果这两个值不同，`referred_column_0_name`标记会错误地呈现`.key`而不是`.name`。

    另请参阅

    新的多列命名约定标记，长名称截断

    参考：[#3989](https://www.sqlalchemy.org/trac/ticket/3989)

+   **[sql] [feature]**

    对“expanding IN”绑定参数功能添加了新逻辑，如果给定的列表为空，将生成一个特殊的“空集”表达式，该表达式针对不同的后端是特定的，从而允许 IN 表达式完全动态化，包括空的 IN 表达式。

    另请参阅

    扩展 IN 功能现在支持空列表

    参考：[#4271](https://www.sqlalchemy.org/trac/ticket/4271)

+   **[sql] [feature]**

    Python 内置函数`dir()`现在支持 SQLAlchemy 的“properties”对象，比如核心列集合（例如`.c`）、`mapper.attrs`等。这样可以使 iPython 的自动补全功能正常工作。感谢 Uwe Korn 提供的拉取请求。

+   **[sql] [feature]**

    添加了新功能`FunctionElement.as_comparison()`，允许 SQL 函数充当可以在 ORM 中使用的二进制比较操作。

    另请参阅

    SQL 函数的二进制比较解释

    参考：[#3831](https://www.sqlalchemy.org/trac/ticket/3831)

+   **[sql] [bug]**

    添加了基于“like”操作符的“比较”操作符，包括`ColumnOperators.startswith()` `ColumnOperators.endswith()` `ColumnOperators.ilike()` `ColumnOperators.notilike()` 等，以便所有这些操作符都可以成为 ORM“primaryjoin”条件的基础。

    参考：[#4302](https://www.sqlalchemy.org/trac/ticket/4302)

+   **[sql] [bug]**

    修复了 `TypeEngine.bind_expression()` 和 `TypeEngine.column_expression()` 方法的问题，这些方法如果目标类型是 `Variant` 或其他 `TypeDecorator` 的目标类型，则不起作用。此外，当渲染这些方法时，SQL 编译器现在调用方言级别的实现，以便方言现在可以为内置类型提供 SQL 级别的处理。

    另请参阅

    TypeEngine 方法 bind_expression、column_expression 与 Variant、类型特定类型一起工作

    参考：[#3981](https://www.sqlalchemy.org/trac/ticket/3981)

### postgresql

+   **[postgresql] [feature]**

    添加了新的 PG 类型 `REGCLASS`，可帮助将表名转换为 OID 值。感谢 Sebastian Bank 的拉取请求。

    此更改也 **回溯** 至：1.2.7

    参考：[#4160](https://www.sqlalchemy.org/trac/ticket/4160)

+   **[postgresql] [feature]**

    添加了对 PostgreSQL 分区表的基本支持，例如，在返回表信息的反射查询中添加了 relkind=’p’。

    另请参阅

    为 PostgreSQL 分区表添加了基本的反射支持

    参考：[#4237](https://www.sqlalchemy.org/trac/ticket/4237)

### mysql

+   **[mysql] [feature]**

    支持在 MySQL 中使用“WITH PARSER”语法的 CREATE FULLTEXT INDEX，使用 `mysql_with_parser` 关键字参数。同时还支持反射，以适应 MySQL 的特殊注释格式，用于报告此选项。此外，“FULLTEXT”和“SPATIAL”索引前缀现在也反映到 `mysql_prefix` 索引选项中。

    参考：[#4219](https://www.sqlalchemy.org/trac/ticket/4219)

+   **[mysql] [feature]**

    添加了对 MySQL 中 ON DUPLICATE KEY UPDATE 语句中参数的支持，以使参数在 MySQL UPDATE 子句中的顺序有意义，类似于 Parameter Ordered Updates 中描述的方式。感谢 Maxim Bublis 的拉取请求。

    另请参阅

    在 ON DUPLICATE KEY UPDATE 中控制参数的顺序

+   **[mysql] [feature]**

    连接池的“pre-ping”功能现在在 mysqlclient、PyMySQL 和 mysql-connector-python 中使用 DBAPI 连接的 `ping()` 方法。感谢 Maxim Bublis 的拉取请求。

    另请参阅

    使用协议级 ping 现在用于预先 ping

### sqlite

+   **[sqlite] [feature]**

    通过新的 SQLite 实现为 `JSON`、`JSON` 增加对 SQLite 的 json 功能的支持。类型的名称是 `JSON`，遵循 SQLite 自己文档中找到的示例。感谢 Ilja Everilä 提交的拉取请求。

    亦参见

    增加对 SQLite JSON 的支持

    参考资料：[#3850](https://www.sqlalchemy.org/trac/ticket/3850)

+   **[sqlite] [feature]**

    将 SQLite 的 `ON CONFLICT` 子句实现为 DDL 理解的方式，例如用于主键、唯一约束和 CHECK 约束以及在 `Column` 上指定以满足内联主键和 NOT NULL。感谢 Denis Kataev 提交的拉取请求。

    亦参见

    增加对约束中的 SQLite ON CONFLICT 的支持

    参考资料：[#4360](https://www.sqlalchemy.org/trac/ticket/4360)

### mssql

+   **[mssql] [feature]**

    为 SQL Server pyodbc 方言添加了 `fast_executemany=True` 参数，当使用 Microsoft ODBC 驱动程序时，这使得可以使用 pyodbc 的同名新性能特性。

    亦参见

    增加对 pyodbc fast_executemany 的支持

    参考资料：[#4158](https://www.sqlalchemy.org/trac/ticket/4158)

+   **[mssql] [bug]**

    已弃用与 SQL Server 中的 `Sequence` 以影响“start”和“increment”值的使用，而更倾向于使用新参数 `mssql_identity_start` 和 `mssql_identity_increment` 直接设置这些参数。`Sequence` 将在未来的版本中用于生成真实的 `CREATE SEQUENCE` DDL 与 SQL Server。

    亦参见

    新增参数以影响 IDENTITY 的起始和增量，已弃用 Sequence 的使用

    参考资料：[#4362](https://www.sqlalchemy.org/trac/ticket/4362)

### oracle

+   **[oracle] [feature]**

    添加了一个新事件，目前仅由 cx_Oracle 方言使用，即 `DialectEvents.setiputsizes()`。该事件将一组 `BindParameter` 对象传递给 DBAPI 特定类型的对象，这些对象在转换为参数名称后将传递给 cx_Oracle `cursor.setinputsizes()` 方法。这允许查看 setinputsizes 过程，并能够修改传递给该方法的数据类型的行为。

    亦参见

    使用 `setinputsizes` 实现对 cx_Oracle 数据绑定性能的精细控制

    此更改也 **回溯到**：1.2.9

    参考：[#4290](https://www.sqlalchemy.org/trac/ticket/4290)

+   **[oracle] [错误]**

    更新了可以发送到 cx_Oracle DBAPI 的参数，既允许所有当前参数，也允许未来尚未添加的参数。此外，删除了在版本 1.2 中已弃用的未使用参数，并且我们现在将“threaded”默认为 False。

    另请参阅

    cx_Oracle 连接参数现代化，已弃用的参数已移除

    参考：[#4369](https://www.sqlalchemy.org/trac/ticket/4369)

+   **[oracle] [错误]**

    Oracle 方言将不再使用 NCHAR/NCLOB 数据类型来表示通用的 Unicode 字符串或 clob 字段，除非传递了标志`use_nchar_for_unicode=True`给`create_engine()` - 这包括 CREATE TABLE 行为以及绑定参数的`setinputsizes()`。在读取方面，在 Python 2 下已添加了 CHAR/VARCHAR/CLOB 结果行的自动 Unicode 转换，以匹配 Python 3 下 cx_Oracle 的行为。为了减轻 Python 2 下的性能损失，SQLAlchemy 在 Python 2 下使用非常高效（当构建了 C 扩展时）的本机 Unicode 处理程序。

    另请参阅

    通用 Unicode 取消强调，通过选项重新启用国家字符数据类型

    参考：[#4242](https://www.sqlalchemy.org/trac/ticket/4242)

### 杂项

+   **[功能] [扩展]**

    添加了新属性`Query.lazy_loaded_from`，其中填充了使用此`Query`来延迟加载关系的`InstanceState`。这样做的原因是它作为水平分片功能的提示，以便使用状态的标识令牌作为查询中用于 id_chooser()的默认标识令牌。

    此更改也**回溯**到：1.2.9

    参考：[#4243](https://www.sqlalchemy.org/trac/ticket/4243)

+   **[功能] [扩展]**

    添加了新功能`BakedQuery.to_query()`，允许在另一个`BakedQuery`内部清晰地使用一个`BakedQuery`作为子查询，而无需显式地引用`Session`。

    参考：[#4318](https://www.sqlalchemy.org/trac/ticket/4318)

+   **[feature] [ext]**

    当目标属性是一个普通列时，`AssociationProxy`现在具有标准的列比较操作，例如`ColumnOperators.like()`和`ColumnOperators.startswith()` - 连接到目标表的 EXISTS 表达式通常会呈现，但列表达式然后在 EXISTS 的 WHERE 条件中使用。请注意，当在基于列的属性上使用时，这改变了关联代理的`.contains()`方法的行为，使其使用`ColumnOperators.contains()`。

    参见

    AssociationProxy 现在为基于列的目标提供标准的列操作符

    参考：[#4351](https://www.sqlalchemy.org/trac/ticket/4351)

+   **[feature] [ext]**

    添加了对水平分片扩展内的`ShardedQuery`类的批量`Query.update()`和`Query.delete()`的支持。这还为批量更新/删除方法`Query._execute_crud()`添加了一个额外的扩展钩子。

    参见

    水平分片扩展支持批量更新和删除方法

    参考：[#4196](https://www.sqlalchemy.org/trac/ticket/4196)

+   **[bug] [ext]**

    重新设计了`AssociationProxy`以存储特定于父类的状态在一个单独的对象中，这样一个`AssociationProxy`可以为多个父类提供服务，就像继承一样，而不会在其返回的状态中产生任何歧义。添加了一个新方法`AssociationProxy.for_class()`，允许检查特定于类的状态。

    另请参阅

    关联代理在每个类的基础上存储特定于类的状态。

    参考：[#3423](https://www.sqlalchemy.org/trac/ticket/3423)

+   **[错误] [扩展]**

    关联代理集合长期维持对父对象的弱引用的行为已被恢复；现在，只要代理集合本身也在内存中，代理将始终对父对象保持强引用，消除了“过时的关联代理”错误。此更改正在试验性地进行，以查看是否会出现任何导致副作用的用例。

    另请参阅

    新功能和改进 - 核心

    参考：[#4268](https://www.sqlalchemy.org/trac/ticket/4268)

+   **[错误] [扩展]**

    修复了关联代理与标量对象解除关联的多个问题。现在`del`可以正常工作，并且还添加了一个新标志`AssociationProxy.cascade_scalar_deletes`，当设置为 True 时，表示将标量属性设置为`None`或通过`del`删除也会将源关联设置为`None`。

    另请参阅

    关联代理新增 cascade_scalar_deletes 标志

    参考：[#4308](https://www.sqlalchemy.org/trac/ticket/4308)

### orm

+   **[orm] [功能]**

    添加了新功能`Query.only_return_tuples()`。导致`Query`对象无条件返回键控元组对象，即使查询针对单个实体也是如此。感谢 Eric Atkin 提供的拉取请求。

    此更改也被**回溯**到：1.2.5

+   **[orm] [功能]**

    向`Session.bulk_save_objects()`方法添加了新标志`Session.bulk_save_objects.preserve_order`，默认值为 True。当设置为 False 时，给定的映射将按对象类型分组为插入和更新，以便更好地将常见操作批量处理在一起。感谢 Alessandro Cucci 的贡献。

+   **[orm] [feature]**

    “selectin”加载策略现在在简单的一对多加载情况下省略了 JOIN，而是仅从相关表中加载，依靠相关表的外键列来匹配父表中的主键。通过将`relationship.omit_join`标志设置为 False，可以禁用此优化。非常感谢 Jayson Reis 的努力。

    另请参阅

    selectin loading no longer uses JOIN for simple one-to-many

    参考：[#4340](https://www.sqlalchemy.org/trac/ticket/4340)

+   **[orm] [feature]**

    向`InstanceState`类添加了`.info`字典，该对象是从调用映射对象的`inspect()`方法返回的对象。

    另请参阅

    info dictionary added to InstanceState

    参考：[#4257](https://www.sqlalchemy.org/trac/ticket/4257)

+   **[orm] [bug]**

    修复了一个 bug，即在使用`Lateral`构造与`Query.join()`以及`Query.select_entity_from()`结合使用时，不会将子句适配应用到连接的右侧。 “lateral”引入了连接右侧可关联的用例。先前，未考虑适配此子句。请注意，在 1.2 版本中，由`Query.subquery()`引入的可选择项仍未适配，原因是[#4304](https://www.sqlalchemy.org/trac/ticket/4304)；可选择项需要由`select()`函数生成，以成为“lateral”连接的右侧。

    此更改也已**回溯**至：1.2.12

    参考：[#4334](https://www.sqlalchemy.org/trac/ticket/4334)

+   **[orm] [bug]**

    修复了关于 `passive_deletes="all"` 的问题，在此情况下，对象的外键属性在从其父集合中移除后仍保持其值。此前，工作单元会将其设置为 NULL，尽管 `passive_deletes` 指示不应修改它。

    另请参阅

    `passive_deletes='all'` 将使从集合中删除的对象的 FK 保持不变

    参考资料：[#3844](https://www.sqlalchemy.org/trac/ticket/3844)

+   **[orm] [bug]**

    改进了与关系绑定的多对一对象表达式的行为，以使对相关对象的列值的检索现在对对象被从其父 `Session` 中分离后也是弹性的，即使属性已过期。在对象过期之前，新功能在 `InstanceState` 中用于备忘特定列属性的最后已知值，以便在对象同时被分离和过期时表达式仍然可以评估。错误条件也通过使用现代属性状态功能改进，以根据需要生成更具体的消息。

    另请参阅

    改进了多对一查询表达式的行为

    参考资料：[#4359](https://www.sqlalchemy.org/trac/ticket/4359)

+   **[orm] [bug] [mysql] [postgresql]**

    ORM 现在在某些情况下在与连接式急加载同时呈现的子查询中将“FOR UPDATE”子句加倍，因为观察到 MySQL 不会锁定子查询中的行。这意味着查询呈现了两个 FOR UPDATE 子句；请注意，在某些后端，如 Oracle，在子查询上使用 FOR UPDATE 子句时会被静默忽略，因为它们是不必要的。此外，在主要与 PostgreSQL 一起使用的 “OF” 子句的情况下，仅当使用此子句时，在内部子查询上才呈现 FOR UPDATE，以便可选择的可针对 SELECT 语句中的表。

    另请参阅

    FOR UPDATE 子句在连接式急加载子查询中被呈现以及外部

    参考资料：[#4246](https://www.sqlalchemy.org/trac/ticket/4246)

+   **[orm] [bug]**

    重新设计了 `Query.join()` 以进一步澄清连接结构的各个组件。此重构添加了当 FROM 列表中有多个元素或查询针对多个实体时，`Query.join()` 确定最合适的“左”连接端的能力。如果有多个 FROM/entity 匹配，则会引发错误，要求指定 ON 子句以解决歧义。特别是针对我们在 [#4363](https://www.sqlalchemy.org/trac/ticket/4363) 中看到的回归，但也具有一般用途。现在 `Query.join()` 中的代码路径更易于跟踪，并且在操作的较早阶段更具体地决定了错误情况。

    另请参阅

    Query.join() 更明确地处理在决定“左”侧时的模棱两可性

    参考资料：[#4365](https://www.sqlalchemy.org/trac/ticket/4365)

+   **[orm] [bug]**

    修复了在 `Query` 中长期存在的问题，其中标量子查询（例如由 `Query.exists()`、`Query.as_scalar()` 和其他从 `Query.statement` 派生的查询生成的）在新的 `Query` 中需要实体适配时不会被正确适配，例如当查询转换为联合查询或 from_self() 时。该更改从 `Query.statement` 访问器生成的 `select()` 对象中删除了“无适配”注释。

    参考资料：[#4304](https://www.sqlalchemy.org/trac/ticket/4304)

+   **[orm] [bug]**

    在 Python 下进行 ORM 刷新时，如果主键值在 Python 中不可排序，例如 `Enum` 没有 `__lt__()` 方法，则会重新引发一个具有信息性的异常；通常情况下，在这种情况下 Python 3 会引发一个 `TypeError`。在 Python 中，刷新过程按主键对持久化对象进行排序，因此值必须可排序。

    参考资料：[#4232](https://www.sqlalchemy.org/trac/ticket/4232)

+   **[orm] [bug]**

    移除了`MappedCollection`类使用的集合转换器。该转换器仅用于断言传入的字典键与其对应对象的键匹配，并且仅在批量设置操作期间使用。该转换器可能会干扰自定义验证器或`AttributeEvents.bulk_replace()`监听器，后者希望进一步转换传入值。当传入的键与值不匹配时，该转换器引发的`TypeError`已被移除；在批量赋值期间，传入值将被键入其生成的键，而不是显式存在于字典中的键。

    总体而言，@converter 已被作为[#3896](https://www.sqlalchemy.org/trac/ticket/3896)的一部分添加的`AttributeEvents.bulk_replace()`事件处理程序所取代。

    参考：[#3604](https://www.sqlalchemy.org/trac/ticket/3604)

+   **[ORM] [错误]**

    添加了新的行为，当检索一对多关系的“旧”值时进行延迟加载，以便跳过由于`lazy="raise"`或分离会话错误而引发的异常。

    参见

    “一对多”替换不会为“raiseload”或“old”对象引发异常

    参考：[#4353](https://www.sqlalchemy.org/trac/ticket/4353)

+   **[ORM] [错误]**

    在 ORM 中存在已久的疏忽，对于一对多关系的`__delete__`方法是无效的，例如对于`del a.b`这样的操作。现在已实现此功能，并且等效于将属性设置为`None`。

    参见

    为 ORM 属性实现“del”

    参考：[#4354](https://www.sqlalchemy.org/trac/ticket/4354)

### ORM 声明式

+   **[ORM] [声明式] [错误]**

    修复了声明式在添加或删除额外属性后未更新`Mapper`状态的错误，当映射器属性集已被调用和记忆化后。此外，如果当前映射的类从完全映射的属性（例如列、关系等）中删除了属性，则现在会引发`NotImplementedError`，因为如果删除了属性，则映射器将无法正确运行。

    参考：[#4133](https://www.sqlalchemy.org/trac/ticket/4133)

### 引擎

+   **[引擎] [特性]**

    将新的“lifo”模式添加到`QueuePool`中，通常通过将标志`create_engine.pool_use_lifo`设置为 True 来启用。 “lifo”模式意味着刚刚检查的相同连接将首先再次检出，允许在池仅部分利用时从服务器端清理多余的连接。拉取请求由 Taem Park 提供。

    另请参阅

    QueuePool 的新后进先出策略

### sql

+   **[sql] [feature]**

    重构`SQLCompiler`以公开类似于`SQLCompiler.order_by_clause()`和`SQLCompiler.limit_clause()`方法的`SQLCompiler.group_by_clause()`方法，可以被方言重写以自定义 GROUP BY 的渲染方式。拉取请求由 Samuel Chou 提供。

    此更改也**回溯**到：1.2.13

+   **[sql] [feature]**

    将`Sequence`添加到“字符串 SQL”系统中，当在没有方言的情况下将包含“序列下一个值”表达式的语句字符串化时，它将生成一个有意义的字符串表达式(`"<next sequence value: my_sequence>"`)，而不是引发编译错误。

    参考：[#4144](https://www.sqlalchemy.org/trac/ticket/4144)

+   **[sql] [feature]**

    添加了新的命名约定标记`column_0N_name`，`column_0_N_name`等，这将为序列中特定约束引用的所有列的名称/键/标签生成名称。为了适应这种命名约定的长度，SQL 编译器的自动截断功能现在也适用于约束名称，这将为约束创建一个缩短的、确定性生成的名称，该名称将适用于目标后端而不超过该后端的字符限制。

    此更改还修复了另外两个问题。一个是`column_0_key`标记尽管已记录，但却不可用，另一个是如果这两个值不同，则`referred_column_0_name`标记会无意中渲染`.key`而不是`.name`的列。

    另请参阅

    新的多列命名约定标记，长名称截断

    参考：[#3989](https://www.sqlalchemy.org/trac/ticket/3989)

+   **[sql] [feature]**

    在“扩展 IN”绑定参数特性中添加了新逻辑，如果给定列表为空，则生成特定于不同后端的特殊“空集合”表达式，从而允许 IN 表达式完全动态化，包括空 IN 表达式。

    参见

    扩展 IN 特性现在支持空列表

    参考：[#4271](https://www.sqlalchemy.org/trac/ticket/4271)

+   **[sql] [feature]**

    现在支持 Python 内置函数 `dir()` 用于 SQLAlchemy “properties” 对象，比如 Core 列集合（例如 `.c`）、`mapper.attrs` 等。还允许 iPython 自动补全正常工作。感谢 Uwe Korn 提交的拉取请求。

+   **[sql] [feature]**

    添加了新特性 `FunctionElement.as_comparison()`，它允许 SQL 函数作为二进制比较操作，在 ORM 中起作用。

    参见

    SQL 函数的二进制比较解释

    参考：[#3831](https://www.sqlalchemy.org/trac/ticket/3831)

+   **[sql] [bug]**

    添加了基于“like”的操作符作为“比较”操作符，包括 `ColumnOperators.startswith()`、`ColumnOperators.endswith()`、`ColumnOperators.ilike()`、`ColumnOperators.notilike()` 等，因此所有这些操作符都可以成为 ORM “primaryjoin” 条件的基础。

    参考：[#4302](https://www.sqlalchemy.org/trac/ticket/4302)

+   **[sql] [bug]**

    修复了 `TypeEngine.bind_expression()` 和 `TypeEngine.column_expression()` 方法的问题，如果目标类型是 `Variant` 或其他 `TypeDecorator` 的一部分，则这些方法将无法工作。另外，SQL 编译器现在在呈现这些方法时调用方言级别的实现，因此方言现在可以为内置类型提供 SQL 级别的处理。

    参见

    TypeEngine 方法 bind_expression、column_expression 可与 Variant、类型特定类型一起使用

    引用：[#3981](https://www.sqlalchemy.org/trac/ticket/3981)

### postgresql

+   **[postgresql] [feature]**

    添加了新的 PG 类型`REGCLASS`，它帮助将表名转换为 OID 值。感谢 Sebastian Bank 的拉取请求。

    此更改还**被回溯到**：1.2.7

    引用：[#4160](https://www.sqlalchemy.org/trac/ticket/4160)

+   **[postgresql] [feature]**

    添加了对 PostgreSQL 分区表的反射基础支持，例如，将 relkind=’p’添加到返回表信息的反射查询中。

    另请参阅

    为 PostgreSQL 分区表添加了基本反射支持

    引用：[#4237](https://www.sqlalchemy.org/trac/ticket/4237)

### mysql

+   **[mysql] [feature]**

    在 MySQL 中添加了对 CREATE FULLTEXT INDEX 的“WITH PARSER”语法的支持，使用`mysql_with_parser`关键字参数。还支持反射，以适应 MySQL 关于此选项报告的特殊注释格式。此外，“FULLTEXT”和“SPATIAL”索引前缀现在被反射回`mysql_prefix`索引选项中。

    引用：[#4219](https://www.sqlalchemy.org/trac/ticket/4219)

+   **[mysql] [feature]**

    添加了对 MySQL 上 ON DUPLICATE KEY UPDATE 语句中参数的排序支持，因为 MySQL UPDATE 子句中的参数顺序是重要的，类似于 Parameter Ordered Updates 中描述的方式。感谢 Maxim Bublis 的拉取请求。

    另请参阅

    控制 ON DUPLICATE KEY UPDATE 中的参数顺序

+   **[mysql] [feature]**

    在 mysqlclient、PyMySQL 和 mysql-connector-python 的情况下，连接池的“pre-ping”功能现在使用 DBAPI 连接的`ping()`方法。感谢 Maxim Bublis 的拉取请求。

    另请参阅

    现在使用协议级别的 ping 进行预先 ping

### sqlite

+   **[sqlite] [feature]**

    通过新的 SQLite 实现为`JSON`添加了对 SQLite 的 json 功能的支持，`JSON`。类型使用的名称是`JSON`，遵循 SQLite 自己文档中找到的示例。感谢 Ilja Everilä的拉取请求。

    另请参阅

    添加了对 SQLite JSON 的支持

    引用：[#3850](https://www.sqlalchemy.org/trac/ticket/3850)

+   **[sqlite] [feature]**

    将 SQLite 的`ON CONFLICT`子句实现为在 DDL 级别理解，例如，对于主键、唯一性和 CHECK 约束，以及在`Column`上指定以满足内联主键和 NOT NULL。感谢 Denis Kataev 的拉取请求。

    另请参阅

    支持在约束中使用 SQLite ON CONFLICT

    参考：[#4360](https://www.sqlalchemy.org/trac/ticket/4360)

### mssql

+   **[mssql] [feature]**

    向 SQL Server pyodbc 方言添加了`fast_executemany=True`参数，该参数启用了使用 Microsoft ODBC 驱动程序时使用 pyodbc 的同名新性能功能。

    另请参阅

    支持 pyodbc fast_executemany

    参考：[#4158](https://www.sqlalchemy.org/trac/ticket/4158)

+   **[mssql] [bug]**

    弃用了在 SQL Server 中使用`Sequence`来影响“start”和“increment”值的做法，而是使用新参数`mssql_identity_start`和`mssql_identity_increment`直接设置这些参数。在未来的版本中，`Sequence`将用于生成真正的`CREATE SEQUENCE`DDL 与 SQL Server。

    另请参阅

    影响 IDENTITY 起始和增量的新参数，已弃用 Sequence 的使用

    参考：[#4362](https://www.sqlalchemy.org/trac/ticket/4362)

### oracle

+   **[oracle] [feature]**

    添加了一个新事件，目前仅由 cx_Oracle 方言使用，`DialectEvents.setiputsizes()`。该事件传递了一个`BindParameter`对象的字典，这些对象将被传递给 DBAPI 特定类型的对象，然后转换为参数名称，传递给 cx_Oracle `cursor.setinputsizes()`方法。这既允许查看 setinputsizes 过程，也允许更改传递给此方法的数据类型的行为。

    另请参阅

    通过 setinputsizes 对 cx_Oracle 数据绑定性能进行细粒度控制

    此更改也**回溯**到：1.2.9

    参考：[#4290](https://www.sqlalchemy.org/trac/ticket/4290)

+   **[oracle] [bug]**

    更新了可以发送到 cx_Oracle DBAPI 的参数，既允许所有当前参数，也允许尚未添加的未来参数。此外，删除了在 1.2 版本中已弃用的未使用参数，并且我们现在将“threaded”默认为 False。

    另请参阅

    cx_Oracle 连接参数现代化，已弃用的参数已移除

    参考：[#4369](https://www.sqlalchemy.org/trac/ticket/4369)

+   **[oracle] [bug]**

    Oracle 方言将不再使用 NCHAR/NCLOB 数据类型来表示通用的 Unicode 字符串或与`Unicode`和`UnicodeText`一起使用的 clob 字段，除非传递了标志`use_nchar_for_unicode=True`给`create_engine()` - 这包括 CREATE TABLE 行为以及绑定参数的`setinputsizes()`。在读取方面，在 Python 2 下已经添加了对 CHAR/VARCHAR/CLOB 结果行的自动 Unicode 转换，以匹配 Python 3 下 cx_Oracle 的行为。为了减轻 Python 2 下的性能损失，SQLAlchemy 在 Python 2 下使用非常高效（当构建了 C 扩展时）的本机 Unicode 处理程序。

    另请参阅

    通用 Unicode 取消强调的国家字符数据类型，通过选项重新启用

    参考：[#4242](https://www.sqlalchemy.org/trac/ticket/4242)

### 杂项

+   **[功能] [扩展]**

    添加了新属性`Query.lazy_loaded_from`，其中填充了正在使用此`Query`来延迟加载关系的`InstanceState`。这样做的原因是它作为水平分片功能的提示，以便可以使用状态的标识令牌作为查询中用于 id_chooser()的默认标识令牌。

    此更改也**回溯**到：1.2.9

    参考：[#4243](https://www.sqlalchemy.org/trac/ticket/4243)

+   **[功能] [扩展]**

    添加了新功能`BakedQuery.to_query()`，允许在另一个`BakedQuery`内部以子查询的干净方式使用一个`BakedQuery`，而无需明确引用`Session`。

    参考：[#4318](https://www.sqlalchemy.org/trac/ticket/4318)

+   **[功能] [扩展]**

    `AssociationProxy` 现在具有标准的列比较操作，例如 `ColumnOperators.like()` 和 `ColumnOperators.startswith()`，当目标属性是普通列时 - 与目标表连接的 EXISTS 表达式通常呈现，但列表达式然后在 EXISTS 的 WHERE 条件中使用。请注意，这会改变关联代理上的 `.contains()` 方法的行为，使其在基于列的属性上使用 `ColumnOperators.contains()`。

    另请参阅

    关联代理现在为基于列的目标提供标准列操作符

    参考：[#4351](https://www.sqlalchemy.org/trac/ticket/4351)

+   **[特性] [扩展]**

    为水平分片扩展中的 `ShardedQuery` 类添加了对批量 `Query.update()` 和 `Query.delete()` 的支持。这还为批量更新/删除方法 `Query._execute_crud()` 添加了一个额外的扩展挂钩。

    另请参阅

    水平分片扩展支持批量更新和删除方法

    参考：[#4196](https://www.sqlalchemy.org/trac/ticket/4196)

+   **[错误] [扩展]**

    重新设计 `AssociationProxy`，以将特定于父类的状态存储在单独的对象中，以便单个 `AssociationProxy` 可以为多个父类提供服务，这是继承固有的，而不会有任何模糊性。添加了一个新方法 `AssociationProxy.for_class()` 以允许检查特定于类的状态。

    另请参阅

    关联代理在每个类的基础上存储特定于类的状态

    参考：[#3423](https://www.sqlalchemy.org/trac/ticket/3423)

+   **[错误] [扩展]**

    关于关联代理集合仅保持对父对象的弱引用的长期行为已恢复；现在，只要代理集合本身也在内存中，代理将始终对父对象保持强引用，从而消除了“过期的关联代理”错误。此更改正在试验性地进行，以查看是否会出现任何引起副作用的用例。

    另请参阅

    核心新特性和改进

    参考资料：[#4268](https://www.sqlalchemy.org/trac/ticket/4268)

+   **[bug] [ext]**

    修复了关于解除标量对象与关联代理的多个问题。现在`del`能正常工作，并且还添加了一个新标志`AssociationProxy.cascade_scalar_deletes`，当设置为 True 时，表示将标量属性设置为`None`或通过`del`删除时也会将源关联设置为`None`。

    另请参阅

    关联代理有新的 cascade_scalar_deletes 标志

    参考资料：[#4308](https://www.sqlalchemy.org/trac/ticket/4308)
