# 0.8 更新日志

> 原文：[`docs.sqlalchemy.org/en/20/changelog/changelog_08.html`](https://docs.sqlalchemy.org/en/20/changelog/changelog_08.html)

## 0.8.7

发布日期：2014 年 7 月 22 日

### orm

+   **[orm] [bug]**

    修复了子查询的急加载中的一个错误，即在与多态加载一起使用时，跨多态子类边界的长链急加载会失败，无法在链中找到子类链接，导致在`AliasedClass`上出现缺失属性名称的错误。

    参考：[#3055](https://www.sqlalchemy.org/trac/ticket/3055)

+   **[orm] [bug]**

    修复了 ORM 中的一个错误，即`class_mapper()`函数会掩盖应该在映射器配置期间由于用户错误引发的 AttributeErrors 或 KeyErrors。对于属性/键错误的捕获已经更具体，不包括配置步骤。

    参考：[#3047](https://www.sqlalchemy.org/trac/ticket/3047)

### sql

+   **[sql] [bug]**

    修复了`Enum`和其他`SchemaType`子类的错误，即直接将类型与`MetaData`关联会导致在发出事件（如创建事件）时 Hang 住`MetaData`。

    参考：[#3124](https://www.sqlalchemy.org/trac/ticket/3124)

+   **[sql] [bug]**

    修复了自定义操作符加上`TypeEngine.with_variant()`系统中的一个错误，在使用变体时与`TypeDecorator`结合使用时，当使用比较运算符时会出现 MRO 错误。

    参考：[#3102](https://www.sqlalchemy.org/trac/ticket/3102)

+   **[sql] [bug]**

    修复了 INSERT..FROM SELECT 构造中的一个错误，即从 UNION 选择时，会将联合包装在一个匿名（例如未标记的）子查询中。

    参考：[#3044](https://www.sqlalchemy.org/trac/ticket/3044)

+   **[sql] [bug]**

    修复了`Table.update()`和`Table.delete()`在应用空的`and_()`或`or_()`或其他空表达式时产生空的 WHERE 子句的错误。现在与`select()`一致。

    参考：[#3045](https://www.sqlalchemy.org/trac/ticket/3045)

### postgresql

+   **[postgresql] [bug]**

    向 PG `HSTORE` 类型添加了`hashable=False`标志，这是为了允许 ORM 在请求混合列/实体列表中的 ORM 映射的 HSTORE 列时跳过尝试“哈希”它。补丁由 Gunnlaugur Þór Briem 提供。

    参考：[#3053](https://www.sqlalchemy.org/trac/ticket/3053)

+   **[postgresql] [bug]**

    添加了一个新的“disconnect”消息“connection has been closed unexpectedly”。这似乎与较新版本的 SSL 有关。感谢 Antti Haapala 的拉取请求。

### mysql

+   **[mysql] [bug]**

    MySQL 错误 2014“commands out of sync”似乎是在现代 MySQL-Python 版本中作为 ProgrammingError 而不是 OperationalError 引发的；现在在 OperationalError 和 ProgrammingError 中都检查了所有测试“is disconnect”的 MySQL 错误代码。

    参考：[#3101](https://www.sqlalchemy.org/trac/ticket/3101)

+   **[mysql] [bug]**

    修复了在索引的`mysql_length`参数上添加列名时需要具有相同引号引用才能被识别的错误。修复使引号变为可选，但也为那些使用此解决方法的人提供了旧的行为以实现向后兼容。

    参考：[#3085](https://www.sqlalchemy.org/trac/ticket/3085)

+   **[mysql] [bug]**

    添加了对反映包含 KEY_BLOCK_SIZE 的索引的表的支持，使用等号。感谢 Sean McGivern 的拉取请求。

### mssql

+   **[mssql] [bug]**

    将语句编码添加到“SET IDENTITY_INSERT”语句中，当在 IDENTITY 列中插入显式 INSERT 时，以支持在不支持 unicode 语句的驱动程序（如 pyodbc + unix + py2k）上使用非 ascii 表标识符。

+   **[mssql] [bug]**

    在 SQL Server pyodbc 方言中，修复了`description_encoding`方言参数的实现，当未明确设置时，会导致在包含其他编码名称的结果集中无法正确解析 cursor.description。这个参数在未来不应该再需要。

    参考：[#3091](https://www.sqlalchemy.org/trac/ticket/3091)

### misc

+   **[bug] [declarative]**

    当访问`__mapper_args__`字典时，从声明性 mixin 或抽象类中复制，以便声明性本身对此字典所做的修改不会与其他映射冲突。该字典在`version_id_col`和`polymorphic_on`参数方面进行修改，用本地类/表正式映射到的列替换其中的列。

    参考：[#3062](https://www.sqlalchemy.org/trac/ticket/3062)

+   **[bug] [ext]**

    修复了可变扩展中的错误，即`MutableDict`未报告`setdefault()`字典操作的更改事件。

    参考：[#3051](https://www.sqlalchemy.org/trac/ticket/3051), [#3093](https://www.sqlalchemy.org/trac/ticket/3093)

+   **[bug] [ext]**

    修复了`MutableDict.setdefault()`未返回现有值或新值的错误（此错误未在任何 0.8 版本中发布）。感谢 Thomas Hervé的拉取请求。

    参考：[#3051](https://www.sqlalchemy.org/trac/ticket/3051), [#3093](https://www.sqlalchemy.org/trac/ticket/3093)

## 0.8.6

发布日期：2014 年 3 月 28 日

### 通用

+   **[general] [bug]**

    调整了`setup.py`文件，以支持将来可能从 setuptools 中删除`setuptools.Feature`扩展。如果不存在此关键字，设置仍将成功使用 setuptools 而不是退回到 distutils。现在还可以通过设置 DISABLE_SQLALCHEMY_CEXT 环境变量来禁用 C 扩展构建。无论 setuptools 是否可用，此变量都有效。

    参考：[#2986](https://www.sqlalchemy.org/trac/ticket/2986)

### orm

+   **[orm] [bug]**

    修复了 ORM 中的错误，即更改对象的主键，然后将其标记为 DELETE 将无法针对 DELETE 定位到正确的行。

    参考：[#3006](https://www.sqlalchemy.org/trac/ticket/3006)

+   **[orm] [bug]**

    修复了从 0.8.3 中的回归，导致`Query.exists()`在只有一个`Query.select_from()`条目但没有其他实体的查询上无法工作的问题。

    参考：[#2995](https://www.sqlalchemy.org/trac/ticket/2995)

+   **[orm] [bug]**

    改进了一个错误消息，如果对非可选择的查询（例如`literal_column()`）进行查询，然后尝试使用`Query.join()`使“左”侧被确定为`None`，然后失败。现在明确检测到这种情况。

+   **[orm] [bug]**

    从 `sqlalchemy.orm.interfaces.__all__` 中删除了陈旧的名称，并使用当前名称进行刷新，以便再次从此模块进行 `import *`。

    参考：[#2975](https://www.sqlalchemy.org/trac/ticket/2975)

### sql

+   **[sql] [bug]**

    修复了在 `tuple_()` 构造中的 bug，在这里，实质上第一个 SQL 表达式的“类型”会被应用为比较元组值的“比较类型”；在某些情况下，这会导致不适当的“类型转换”发生，例如当元组中混合了字符串和二进制值时，错误地将目标值转换为二进制，即使左侧的类型并非如此。`tuple_()` 现在期望其值列表中存在异构类型。

    参考：[#2977](https://www.sqlalchemy.org/trac/ticket/2977)

### postgresql

+   **[postgresql] [feature]**

    启用了对 psycopg2 DBAPI 的“合理的多行计数”检查，因为这似乎是在 psycopg2 2.0.9 中支持的。

+   **[postgresql] [bug]**

    修复了由于版本 0.8.5 / 0.9.3 的兼容性增强引起的回归，其中仅针对 8.1、8.2 系列的 PostgreSQL 版本的索引反射再次中断，涉及到一直问题多多的 int2vector 类型。尽管 int2vector 从 8.1 开始支持数组操作，但显然只能从 8.3 开始支持 CAST 到 varchar。

    参考：[#3000](https://www.sqlalchemy.org/trac/ticket/3000)

### misc

+   **[bug] [ext]**

    修复了可变扩展中的 bug 以及 `flag_modified()` 中的 bug，在这里，如果属性已被重新分配给自身，则更改事件将不会传播。

    参考：[#2997](https://www.sqlalchemy.org/trac/ticket/2997)

## 0.8.5

发布日期：2014 年 2 月 19 日

### orm

+   **[orm] [bug]**

    修复了 `Query.get()` 在查询已存在条件的查询时无法始终引发 `InvalidRequestError` 的错误的 bug，当给定的标识已经存在于标识映射中时。

    参考：[#2951](https://www.sqlalchemy.org/trac/ticket/2951)

+   **[orm] [bug]**

    修复了当将迭代器对象传递给 `class_mapper()` 或类似函数时出现的错误消息，其中错误会在字符串格式化时无法呈现的 bug。来自 Kyle Stark 的 Pullreq。

+   **[orm] [bug]**

    对`subqueryload()`策略进行了调整，确保查询在加载过程开始后运行；这样，subqueryload 优先于其他加载器运行，这可能是由于其他贪婪/无加载情况在错误的时间命中相同属性。

    参考：[#2887](https://www.sqlalchemy.org/trac/ticket/2887)

+   **[orm] [bug]**

    修复了一个 bug，当从表继承到基表的选择/别名时使用联接表继承时，PK 列也不是同名时，持久性系统将无法在 INSERT 时将主键值从基表复制到继承表。

    参考：[#2885](https://www.sqlalchemy.org/trac/ticket/2885)

+   **[orm] [bug]**

    当传递的列/属性（名称）不解析为列或映射属性（例如错误的元组）时，`composite()`将引发一个信息性错误消息；先前引发了一个未绑定的本地错误。

    参考：[#2889](https://www.sqlalchemy.org/trac/ticket/2889)

### engine

+   **[engine] [bug] [pool]**

    修复了由[#2880](https://www.sqlalchemy.org/trac/ticket/2880)引起的关键回归问题，其中新的并发能力从池中返回连接意味着“first_connect”事件现在也不再同步，因此在即使是最小并发情况下也会导致方言配置错误。

    参考：[#2880](https://www.sqlalchemy.org/trac/ticket/2880)，[#2964](https://www.sqlalchemy.org/trac/ticket/2964)

### sql

+   **[sql] [bug]**

    修复了一个 bug，即使用空列表或元组调用`Insert.values()`将引发 IndexError。现在会生成一个空的插入构造，就像使用空字典的情况一样。

    参考：[#2944](https://www.sqlalchemy.org/trac/ticket/2944)

+   **[sql] [bug]**

    修复了一个 bug，即如果错误地传递了包含`__getitem__()`方法的列表达式的比较器的列表达式，例如使用`ARRAY`类型的列，`ColumnOperators.in_()`将进入无限循环。

    参考：[#2957](https://www.sqlalchemy.org/trac/ticket/2957)

+   **[sql] [bug]**

    修复了一个问题，即具有 Sequence 的主键列，但该列不是“自动增量”列，要么因为它有外键约束，要么设置了`autoincrement=False`，在没有主键值的情况下尝试在不支持序列的后端上插入时，会尝试触发 Sequence。这将发生在像 SQLite、MySQL 这样的非序列后端上。

    参考：[#2896](https://www.sqlalchemy.org/trac/ticket/2896)

+   **[sql] [bug]**

    修复了`Insert.from_select()`方法的错误，其中给定名称的顺序在生成 INSERT 语句时不会被考虑，因此与给定 SELECT 语句中的列名不匹配。还指出`Insert.from_select()`暗示不能使用 Python 端的插入默认值，因为该语句没有 VALUES 子句。

    参考：[#2895](https://www.sqlalchemy.org/trac/ticket/2895)

+   **[sql] [enhancement]**

    当编译语句中存在一个未赋值的`BindParameter`时引发的异常现在在错误消息中包含绑定参数的键名。

### postgresql

+   **[postgresql] [bug]**

    添加了一个额外的消息到 psycopg2 断开检测，“无法发送数据到服务器”，这与现有的“无法从服务器接收数据”相辅相成，并已被用户观察到。

    参考：[#2936](https://www.sqlalchemy.org/trac/ticket/2936)

+   **[postgresql] [bug]**

    > 对于非常古老的（8.1 版本之前）PostgreSQL 版本以及潜在的其他 PG 引擎（假设 Redshift 将版本报告为<8.1），已改进对 PostgreSQL 反射行为的支持，查询“索引”和“主键”依赖于检查所谓的“int2vector”数据类型，该数据类型在 8.1 之前拒绝强制转换为数组，导致查询中使用的“ANY()”运算符失败。通过广泛的搜索找到了非常 hacky 但被 PG 核心开发人员推荐的查询，用于在使用 PG 版本<8.1 时使用，因此现在在这些版本上可以正常工作索引和主键约束反射。

+   **[postgresql] [bug]**

    修订了这个非常古老的问题，其中 PostgreSQL 的“获取主键”反射查询已更新以考虑已重命名的主键约束；新查询在非常古老的 PostgreSQL 版本（如版本 7）上失败，因此在检测到 server_version_info < (8, 0)的情况下，在这些情况下恢复旧查询。

    参考：[#2291](https://www.sqlalchemy.org/trac/ticket/2291)

### mysql

+   **[mysql] [feature]**

    添加了新的 MySQL 特定的`DATETIME`，其中包括分数秒支持；还向`TIMESTAMP`添加了分数秒支持。尽管 DBAPI 支持有限，但 MySQL Connector/Python 已知支持分数秒。补丁由 Geert JM Vanderkelen 提供。

    参考：[#2941](https://www.sqlalchemy.org/trac/ticket/2941)

+   **[mysql] [bug]**

    添加了对 `PARTITION BY` 和 `PARTITIONS` MySQL 表关键字的支持，指定为 `mysql_partition_by='value'` 和 `mysql_partitions='value'` 给 `Table`. 感谢 Marcus McCurdy 提交的 Pullreq。

    参考：[#2966](https://www.sqlalchemy.org/trac/ticket/2966)

+   **[mysql] [错误]**

    修复了阻止基于 MySQLdb 的方言（例如 pymysql）在 Py3K 中工作的错误，其中一个对“连接字符集”的检查会由于 Py3K 的更严格的值比较规则而失败。在这种情况下，调用并没有考虑到数据库版本，因为服务器版本在那时仍然是 None，因此该方法总体上已经简化为依赖于 connection.character_set_name()。

    参考：[#2933](https://www.sqlalchemy.org/trac/ticket/2933)

+   **[mysql] [错误]**

    在 cymysql 方言中添加了一些缺失的方法，包括 _get_server_version_info() 和 _detect_charset()。感谢 Hajime Nakagami 提交的 Pullreq。

### sqlite

+   **[sqlite] [错误]**

    恢复了在将唯一约束反射回到 0.8 时被遗漏的更改，其中包含 SQLite 的 `UniqueConstraint`，如果列名中包含保留关键字，则会失败。感谢 Roman Podolyaka 提交的 Pullreq。

### mssql

+   **[mssql] [错误] [firebird]**

    使用 `Float` 类型的 “asdecimal” 标志现在将与 Firebird 以及 mssql+pyodbc 方言一起工作；以前的十进制转换未发生。

+   **[mssql] [错误] [pymssql]**

    将 “Net-Lib error during Connection reset by peer” 消息添加到 pymssql 方言中检查的消息列表中，以检查“disconnect”。感谢 John Anderson。

### 杂项

+   **[错误] [py3k]**

    修复了 Py3K 中的错误，其中缺少的导入将导致在呈现绑定参数时“literal binary”模式无法导入“util.binary_type”。0.9 处理方式不同。感谢 Andreas Zeidler 提交的 Pullreq。

+   **[错误] [firebird]**

    firebird 方言将引用以下划线开头的标识符。感谢 Treeve Jelbert。

    参考：[#2897](https://www.sqlalchemy.org/trac/ticket/2897)

+   **[错误] [firebird]**

    修复了 Firebird 索引反射中的错误，其中索引中的列没有正确排序；它们现在按照 RDB$FIELD_POSITION 的顺序排序。

+   **[错误] [declarative]**

    当将无法解析为类或映射器的字符串参数发送到 `relationship()` 时，错误消息已校正为与接收到非字符串参数时相同的方式，该方式指示了具有配置错误的关系的名称。

    参考：[#2888](https://www.sqlalchemy.org/trac/ticket/2888)

## 0.8.4

发布日期：2013 年 12 月 8 日

### orm

+   **[orm] [错误]**

    修复了由 [#2818](https://www.sqlalchemy.org/trac/ticket/2818) 引入的回归，其中生成的 EXISTS 查询会为具有两个同名列的语句产生“columns being replaced”警告，因为内部的 SELECT 没有设置 use_labels。

    参考文献：[#2818](https://www.sqlalchemy.org/trac/ticket/2818)

### engine

+   **[engine] [bug]**

    在 `connect()` 上引发错误的 DBAPI（如 `TypeError`、`NotImplementedError` 等）如果不是 dbapi.Error 的子类，则会以原样传播异常。先前，针对 `connect()` 例程的特定错误处理既会不适当地通过方言的 `Dialect.is_disconnect()` 例程运行异常，也会将其包装在一个 `sqlalchemy.exc.DBAPIError` 中。现在它会像在执行过程中一样保持不变地传播。

    参考文献：[#2881](https://www.sqlalchemy.org/trac/ticket/2881)

+   **[engine] [bug] [pool]**

    `QueuePool` 已经改进，不再在现有连接尝试阻塞时阻止新的连接尝试。先前，新连接的生成在监视溢出的块内串行化；现在，溢出计数器在连接过程本身之外的自己的临界区内进行修改。

    参考文献：[#2880](https://www.sqlalchemy.org/trac/ticket/2880)

+   **[engine] [bug] [pool]**

    对等待池化连接可用性的逻辑进行了微小调整，对于未指定超时的连接池，它将每隔半秒中断一次等待，以检查所谓的“abort”标志，这允许等待者在整个连接池被转储的情况下中断；通常情况下，等待者应该由于 notify_all() 而中断，但在极少数情况下可能会错过这个 notify_all()。这是从 0.8.0 版本首次引入的逻辑的扩展，该问题只在压力测试中偶尔观察到。

    参考文献：[#2522](https://www.sqlalchemy.org/trac/ticket/2522)

+   **[engine] [bug]**

    修复了一个 bug，当在 `Connection.execute()` 内引发一个预先的 DBAPI `StatementError` 时，SQL 语句会被错误地 ASCII 编码，导致非 ASCII 语句的编码错误。现在字符串化仍然保持在 Python unicode 中，从而避免了编码错误。

    参考文献：[#2871](https://www.sqlalchemy.org/trac/ticket/2871)

### sql

+   **[sql] [feature]**

    添加了对“唯一约束”反射的支持，通过 `Inspector.get_unique_constraints()` 方法。感谢 Roman Podolyaka 提供的补丁。

    参考：[#1443](https://www.sqlalchemy.org/trac/ticket/1443)

### postgresql

+   **[postgresql] [bug]**

    修复了一个 bug，当使用 pypostgresql 适配器时，索引反射会错误地解释 indkey 值，该适配器将这些值作为列表返回，而不是 psycopg2 返回的字符串类型。

    参考：[#2855](https://www.sqlalchemy.org/trac/ticket/2855)

### mssql

+   **[mssql] [bug]**

    修复了 0.8.0 中引入的 bug，在 MSSQL 中，如果索引在备用模式中，则 `DROP INDEX` 语句会错误地渲染; schemaname/tablename 会被颠倒。格式也已经被修改以匹配当前的 MSSQL 文档。感谢 Derek Harland。

### oracle

+   **[oracle] [bug]**

    将 ORA-02396 “最大空闲时间”错误代码添加到了 cx_oracle 的“断开连接”代码列表中。

    参考：[#2864](https://www.sqlalchemy.org/trac/ticket/2864)

+   **[oracle] [bug]**

    修复了一个 bug，在 Oracle 中，给定没有长度的 `VARCHAR` 类型（例如，用于 `CAST` 或类似操作）会错误地渲染为 `None CHAR` 或类似内容。

    参考：[#2870](https://www.sqlalchemy.org/trac/ticket/2870)

### 杂项

+   **[bug] [ext]**

    修复了一个 bug，该 bug 阻止了 `serializer` 扩展在包含非 ASCII 字符的表或列名中正常工作。

    参考：[#2869](https://www.sqlalchemy.org/trac/ticket/2869)

## 0.8.3

发布日期：2013 年 10 月 26 日

### orm

+   **[orm] [feature]**

    添加了新选项到`relationship()` `distinct_target_key`。这使得子查询的贪婪加载策略对内部的 SELECT 子查询应用 DISTINCT，在这种关系对应的内部查询生成重复行的情况下有所帮助（目前还没有一个通用的解决方案来解决子查询贪婪加载中的重复行问题，然而，当内部子查询之外的连接产生重复行时）。当标志设置为`True`时，DISTINCT 无条件渲染，当设置为`None`时，如果内部关系的目标列不构成完整的主键，则渲染 DISTINCT。该选项在 0.8 版本中默认为 False（例如，在所有情况下默认关闭），在 0.9 版本中为 None（例如，默认情况下自动化）。感谢 Alexander Koval 对此的帮助。

    另见

    Subquery Eager Loading will apply DISTINCT to the innermost SELECT for some queries

    参考：[#2836](https://www.sqlalchemy.org/trac/ticket/2836)

+   **[orm] [bug]**

    修复了列表仪器化无法正确表示`[0:0]`的切片的错误，特别是在使用关联代理时可能会发生。由于 Python 集合的某些怪癖，该问题在 Python 3 中更有可能发生，而不是在 Python 2 中。

    此更改也被**回溯**到：0.7.11

    参考：[#2807](https://www.sqlalchemy.org/trac/ticket/2807)

+   **[orm] [bug]**

    修复了在与父`Table`关联之前使用类似`remote()`或`foreign()`的注释在关联之前，可能会导致父表由于注释执行的固有复制操作而未在联接中呈现的问题。

    参考：[#2813](https://www.sqlalchemy.org/trac/ticket/2813)

+   **[orm] [bug]**

    修复了`Query.exists()`在没有任何 WHERE 条件的情况下无法正常工作的错误。感谢 Vladimir Magamedov。

    参考：[#2818](https://www.sqlalchemy.org/trac/ticket/2818)

+   **[orm] [bug]**

    从 0.9 版本中借鉴了一个变化，即在多态继承加载中使用的映射层次结构的迭代是有序的，这使得为多态查询生成的 SELECT 语句具有确定性呈现，进而有助于缓存方案，这些方案在 SQL 字符串本身上进行缓存。

    参考：[#2779](https://www.sqlalchemy.org/trac/ticket/2779)

+   **[orm] [bug]**

    修复了 ORM 用于迭代映射层次结构的有序序列实现中的潜在问题；在 Jython 解释器下，该实现未被排序，尽管 cPython 和 PyPy 保持了排序。

    参考：[#2794](https://www.sqlalchemy.org/trac/ticket/2794)

+   **[orm] [bug]**

    修复了 ORM 级事件注册中“原始”或“传播”标志在某些“未映射的基类”配置中可能被错误配置的潜在问题。

    参考：[#2786](https://www.sqlalchemy.org/trac/ticket/2786)

+   **[orm] [bug]**

    关于使用`defer()`选项加载映射实体时的性能修复。在加载时将每个对象的延迟调用函数应用到实例的函数开销显著高于仅从行加载数据的函数开销（请注意，`defer()`旨在减少 DB/network 开销，而不一定是函数调用次数）；现在在所有情况下，函数调用开销都小于从列加载数据的开销。每次从 N（结果中的总延迟值）加载时，还会减少创建的“懒惰可调用”对象的数量到 1（延迟列的总数）。

    参考：[#2778](https://www.sqlalchemy.org/trac/ticket/2778)

+   **[ORM] [错误]**

    修复了属性历史函数在使用`make_transient()`函数将对象从“持久”移动到“挂起”时会失败的 bug，特别是涉及基于集合的反向引用的操作。

    参考：[#2773](https://www.sqlalchemy.org/trac/ticket/2773)

### ORM 声明式

+   **[ORM] [声明式] [特性]**

    添加了一个方便的类装饰器`as_declarative()`，它是`declarative_base()`的包装器，允许使用一种巧妙的类装饰器方法应用现有的基类。

### 示例

+   **[示例] [特性]**

    改进了`examples/generic_associations`中的示例，包括`discriminator_on_association.py`使用单表继承来处理“鉴别器”。还添加了一个真正的“通用外键”示例，它类似于其他流行框架，使用开放的整数指向任何其他表，放弃了传统的引用完整性。虽然我们不推荐这种模式，但信息想要自由。

+   **[示例] [错误]**

    在版本控制示例中创建的历史表中添加了“autoincrement=False”，因为这个表在任何情况下都不应该具有自增属性，感谢 Patrick Schmid。

### 引擎

+   **[引擎] [特性]**

    `Engine`的`URL`的`repr()`现在会使用星号隐藏密码。感谢 Gunnlaugur Þór Briem。

    参考：[#2821](https://www.sqlalchemy.org/trac/ticket/2821)

+   **[引擎] [错误]**

    `make_url()`函数现在使用的正则表达式解析 ipv6 地址，例如用方括号括起来。

    此更改也**回溯**到：0.7.11

    参考：[#2851](https://www.sqlalchemy.org/trac/ticket/2851)

+   **[引擎] [错误] [Oracle]**

    如果重新创建`Engine`时，不会第二次调用 Dialect.initialize()，因为出现了断开连接错误。这修复了 Oracle 8 方言中的一个特定问题，但通常来说，Dialect.initialize()阶段应该只执行一次。

    参考：[#2776](https://www.sqlalchemy.org/trac/ticket/2776)

+   **[引擎] [错误] [池]**

    修复了`QueuePool`在现有池化连接在无效或重置事件后未能重新连接时会丢失正确的已检出计数的错误。

    参考：[#2772](https://www.sqlalchemy.org/trac/ticket/2772)

### SQL

+   **[SQL] [特性]**

    添加了`insert()`构造的新方法`Insert.from_select()`。给定列的列表和可选择项，呈现`INSERT INTO (table) (columns) SELECT ..`。

    参考：[#722](https://www.sqlalchemy.org/trac/ticket/722)

+   **[sql] [feature]**

    `update()`、`insert()`和`delete()`构造现在将解释 ORM 实体作为要操作的目标表，例如：

    ```py
    from sqlalchemy import insert, update, delete

    ins = insert(SomeMappedClass).values(x=5)

    del_ = delete(SomeMappedClass).where(SomeMappedClass.id == 5)

    upd = update(SomeMappedClass).where(SomeMappedClass.id == 5).values(name="ed")
    ```

+   **[sql] [bug]**

    修复了自 0.7.9 以来的回归，其中如果 CTE 在多个 FROM 子句中被引用，则其名称可能无法正确引用。

    此更改也**回溯**到：0.7.11

    参考：[#2801](https://www.sqlalchemy.org/trac/ticket/2801)

+   **[sql] [bug] [cte]**

    修复了公共表达式系统中的错误，如果 CTE 仅用作`alias()`构造，则不会使用 WITH 关键字呈现。

    此更改也**回溯**到：0.7.11

    参考：[#2783](https://www.sqlalchemy.org/trac/ticket/2783)

+   **[sql] [bug]**

    修复了`CheckConstraint` DDL 中的错误，其中来自`Column`对象的“quote”标志不会传播。

    此更改也**回溯**到：0.7.11

    参考：[#2784](https://www.sqlalchemy.org/trac/ticket/2784)

+   **[sql] [bug]**

    修复了`type_coerce()`无法正确解释具有`__clause_element__()`方法的 ORM 元素的错误。

    参考：[#2849](https://www.sqlalchemy.org/trac/ticket/2849)

+   **[sql] [bug]**

    `Enum`和`Boolean`类型现在在生成“非本机”类型的 CHECK 约束时绕过任何自定义（例如 TypeDecorator）类型。这样，自定义类型不会参与 CHECK 中的表达式，因为此表达式针对“impl”值而不是“decorated”值。

    参考：[#2842](https://www.sqlalchemy.org/trac/ticket/2842)

+   **[sql] [bug]**

    `Index`上的`.unique`标志可能会在从未指定`unique`（默认为`None`）的`Column`生成时产生`None`。该标志现在将始终为`True`或`False`。

    参考：[#2825](https://www.sqlalchemy.org/trac/ticket/2825)

+   **[sql] [bug]**

    修复了默认编译器以及 postgresql、mysql 和 mssql 的 bug，以确保任何字面 SQL 表达式值在 CREATE INDEX 语句中直接呈现为字面值，而不是作为绑定参数。这也改变了其他 DDL 的呈现方案，如约束。

    参考：[#2742](https://www.sqlalchemy.org/trac/ticket/2742)

+   **[sql] [bug]**

    在其 FROM 子句中使自身引用的`select()`，通常通过就地突变，将引发信息性错误消息，而不会导致递归溢出。

    参考：[#2815](https://www.sqlalchemy.org/trac/ticket/2815)

+   **[sql] [bug]**

    `ForeignKey`上的非工作“schema”参数已被弃用；引发警告。在 0.9 中移除。

    参考：[#2831](https://www.sqlalchemy.org/trac/ticket/2831)

+   **[sql] [bug]**

    修复了使用`column_reflect`事件更改传入`Column`的`.key`会阻止主键约束、索引和外键约束被正确反映的 bug。

    参考：[#2811](https://www.sqlalchemy.org/trac/ticket/2811)

+   **[sql] [bug]**

    在 0.8 中添加的`ColumnOperators.notin_()`运算符现在正确地生成了对空集合使用时“IN”返回的表达式的否定。

+   **[sql] [bug] [postgresql]**

    修复了表达式系统依赖于在`select()`构造上引用`.c`集合时一些表达式的`str()`形式，但由于元素依赖于特定于方言的编译构造，特别是与 PostgreSQL `ARRAY`元素一起使用的`__getitem__()`运算符，因此`str()`形式不可用。该修复还添加了一个新的异常类`UnsupportedCompilationError`，在编译器被要求编译它不知道如何处理的内容时引发。

    参考：[#2780](https://www.sqlalchemy.org/trac/ticket/2780)

### postgresql

+   **[postgresql] [bug]**

    从列的服务器默认值反射中删除了 128 个字符的截断；这段代码最初来自 PG 系统视图，用于截断字符串以便阅读。

    参考：[#2844](https://www.sqlalchemy.org/trac/ticket/2844)

+   **[postgresql] [bug]**

    括号将应用于在 CREATE INDEX 语句的列列表中呈现的复合 SQL 表达式。

    参考：[#2742](https://www.sqlalchemy.org/trac/ticket/2742)

+   **[postgresql] [bug]**

    修复了一个 bug，即具有在“PostgreSQL”或“EnterpriseDB”之前的前缀的 PostgreSQL 版本字符串将无法解析。感谢 Scott Schaefer。

    参考：[#2819](https://www.sqlalchemy.org/trac/ticket/2819)

### mysql

+   **[mysql] [bug]**

    MySQL 版本 5.5、5.6 的保留字更新，感谢 Hanno Schlichting。

    此更改也**回溯**到：0.7.11

    参考：[#2791](https://www.sqlalchemy.org/trac/ticket/2791)

+   **[mysql] [bug]**

    [#2721](https://www.sqlalchemy.org/trac/ticket/2721)中的更改是，`ForeignKeyConstraint`的`deferrable`关键字在 MySQL 后端上被静默忽略，将在 0.9 版本中恢复；此关键字现在将再次渲染，在 MySQL 上引发错误，因为它不被理解 - 相同的行为也将适用于`initially`关键字。在 0.8 中，这些关键字将继续被忽略，但会发出警告。此外，`match`关键字现在在 0.9 上引发`CompileError`，在 0.8 上发出警告；这个关键字不仅被 MySQL 静默忽略，还会破坏 ON UPDATE/ON DELETE 选项。

    要使用在 MySQL 上不渲染或以不同方式渲染的`ForeignKeyConstraint`，请使用自定义编译选项。文档中已添加了此用法示例，请参见 MySQL / MariaDB Foreign Keys。

    参考：[#2721](https://www.sqlalchemy.org/trac/ticket/2721)���[#2839](https://www.sqlalchemy.org/trac/ticket/2839)

+   **[mysql] [bug]**

    MySQL-connector 方言现在允许在 create_engine 查询字符串中覆盖在连接中设置的默认值，包括“buffered”和“raise_on_warnings”。

    参考：[#2515](https://www.sqlalchemy.org/trac/ticket/2515)

### sqlite

+   **[sqlite] [bug]**

    新增的 SQLite DATETIME 参数 storage_format 和 regexp 显然没有完全正确实现；虽然参数被接受，但实际上它们没有任何效果；这个问题已经修复。

    参考：[#2781](https://www.sqlalchemy.org/trac/ticket/2781)

### oracle

+   **[oracle] [bug]**

    修复了一个 bug，即使用同义词进行 Oracle 表反射时，如果同义词和表位于不同的远程模式中，则会失败。修复补丁由 Kyle Derr 提供。

    参考：[#2853](https://www.sqlalchemy.org/trac/ticket/2853)

### 杂项

+   **[feature]**

    为`Column`添加了一个新标志`system=True`，将列标记为“系统”列，这些列将由数据库自动添加（例如 PostgreSQL 的`oid`或`xmin`）。该列将在`CREATE TABLE`语句中被省略，但仍可用于查询。此外，`CreateColumn`构造可以应用于自定义编译规则，允许跳过列，通过生成返回`None`的规则。

## 0.8.2

发布日期：2013 年 7 月 3 日

### orm

+   **[orm] [feature]**

    添加了一个新方法`Query.select_entity_from()`，将在 0.9 版本中取代`Query.select_from()`的部分功能。在 0.8 版本中，这两个方法执行相同的功能，因此可以适当地将代码迁移到使用`Query.select_entity_from()`方法。详细信息请参阅 0.9 迁移指南。

    参考：[#2736](https://www.sqlalchemy.org/trac/ticket/2736)

+   **[orm] [bug]**

    当尝试刷新已分配多态鉴别器为无效值的继承类对象时，会发出警告。

    参考：[#2750](https://www.sqlalchemy.org/trac/ticket/2750)

+   **[orm] [bug]**

    修复了多个加入继承实体针对相同基类相互连接时，在连接字符串超过两个实体时，基表上的列不会独立跟踪的多态 SQL 生成中的错误。

    参考：[#2759](https://www.sqlalchemy.org/trac/ticket/2759)

+   **[orm] [bug]**

    修复了将复合属性发送到`Query.order_by()`会产生一些数据库不接受的括号表达式的错误。

    参考：[#2754](https://www.sqlalchemy.org/trac/ticket/2754)

+   **[orm] [bug]**

    修复了复合属性与`aliased()`函数之间的交互。以前，在应用别名时，复合属性在比较操作中无法正常工作。

    参考：[#2755](https://www.sqlalchemy.org/trac/ticket/2755)

+   **[orm] [bug] [ext]**

    修复了`MutableDict`在调用`clear()`时未报告更改事件的错误。

    参考：[#2730](https://www.sqlalchemy.org/trac/ticket/2730)

+   **[orm] [bug]**

    修复了由 [#2682](https://www.sqlalchemy.org/trac/ticket/2682) 引起的回归问题，即 `Query.update()` 和 `Query.delete()` 调用的评估会触发不支持的 `True` 和 `False` 符号，因为现在使用了 `IS`。

    参考：[#2737](https://www.sqlalchemy.org/trac/ticket/2737)

+   **[orm] [bug]**

    由此票据引起的 0.7 版本的回归问题已修复，这使得自引用的贪婪连接的递归溢出检查变得太宽松，忽略了一个特定情况，即子类配置了 lazy=”joined” 或 “subquery”，并且加载是针对基类的“with_polymorphic”。

    参考：[#2481](https://www.sqlalchemy.org/trac/ticket/2481)

+   **[orm] [bug]**

    修复了从 0.7 版本中引入的回归问题，`Session.begin_nested()` 的 contextmanager 功能在发生 flush 错误时未能正确回滚事务，而是引发了自己的异常，同时保留了仍处于待回滚状态的会话。

    参考：[#2718](https://www.sqlalchemy.org/trac/ticket/2718)

### orm declarative

+   **[orm] [declarative] [feature]**

    现在可以在 `order_by`、`primaryjoin` 或类似情况下使用字符串参数引用 ORM 描述符，例如混合属性，以及与列绑定的属性一样在 `relationship()` 中使用。

    参考：[#2761](https://www.sqlalchemy.org/trac/ticket/2761)

### examples

+   **[examples] [bug]**

    修复了“版本控制”配方中的问题，其中当存在反向引用时，一个多对一引用可能会为目标生成一个无意义的版本，即使它没有被更改。由 Matt Chisholm 提供的补丁。

+   **[examples] [bug]**

    修复了狗堆示例中的一个小错误，即 SQL 缓存键的生成未像 `Query` 通常所做的那样对语句应用去重标签。

### engine

+   **[engine] [bug]**

    修复了各种 `Pool` 实现中 `reset_on_return` 参数未在重新生成池时传播的错误。由 Eevee 提供。

+   **[engine] [bug] [sybase]**

    修复了在某些情况下无法检测到正确的 kwargs 被发送到 `create_engine()` 的例程失败的错误，例如在 Sybase 方言中。

    参考：[#2732](https://www.sqlalchemy.org/trac/ticket/2732)

### sql

+   **[sql] [feature]**

    为`TypeDecorator`提供了一个名为`TypeDecorator.coerce_to_is_types`的新属性，以便更容易控制使用`==`或`!=`与`None`和布尔类型进行比较时如何生成`IS`表达式，或者带有绑定参数的普通相等表达式。

    参考：[#2734](https://www.sqlalchemy.org/trac/ticket/2734)，[#2744](https://www.sqlalchemy.org/trac/ticket/2744)

+   **[sql] [bug]**

    多个修复针对`Select`构造的相关行为，首次引入于 0.8.0 版本：

    +   为满足 FROM 条目应向外部 SELECT 进行相关的用例，该 SELECT 包含另一个 SELECT，而后者又包含此 SELECT，现在当通过`Select.correlate()`建立明确相关性时，相关性现在可以跨多个级别工作，前提是目标 SELECT 在由 WHERE/ORDER BY/columns 子句包含的链中的某处，而不仅仅是嵌套的 FROM 子句。这使得`Select.correlate()`再次更加兼容 0.7 版本，同时仍保持新的“智能”相关性。

    +   当未使用明确相关性时，“隐式”相关性将其行为限制在仅限于直接封闭的 SELECT 中，以最大限度地提高与 0.7 应用程序的兼容性，并且在这种情况下还防止跨嵌套 FROM 的相关性，保持与 0.8.0/0.8.1 的兼容性。

    +   `Select.correlate_except()`方法未在所有情况下阻止给定的 FROM 子句进行相关，并且还会导致 FROM 子句被不正确地完全省略（更像是 0.7 会做的），这已经修复。

    +   调用 select.correlate_except(None)将使所有 FROM 子句进入相关性，正如预期的那样。

    参考：[#2668](https://www.sqlalchemy.org/trac/ticket/2668)，[#2746](https://www.sqlalchemy.org/trac/ticket/2746)

+   **[sql] [bug]**

    修复了一个错误，即将表“A”的 select()与多个外键路径连接到表“B”，连接到表“B”时，如果直接将表“A”连接到“B”会报告“模糊连接条件”错误，而不是生成具有多个条件的连接条件。

    参考：[#2738](https://www.sqlalchemy.org/trac/ticket/2738)

+   **[sql] [bug] [reflection]**

    修复了在远程模式和本地模式下同时使用`MetaData.reflect()` 可能会在两个模式都有相同名称的表时产生错误结果的错误。

    参考：[#2728](https://www.sqlalchemy.org/trac/ticket/2728)

+   **[sql] [bug]**

    从基础`ColumnOperators` 类中删除了“未实现”的`__iter__()`调用，虽然这是在 0.8.0 中引入的，以防止在自定义运算符上实现`__getitem__()`方法并在该对象上错误调用`list()`时出现无限、内存增长的循环，但它导致列元素报告它们实际上是可迭代类型，然后在尝试迭代时抛出错误。在这里没有真正的方法同时拥有两边，所以我们坚持使用 Python 最佳实践。在自定义运算符上实现`__getitem__()`时要小心！

    参考：[#2726](https://www.sqlalchemy.org/trac/ticket/2726)

+   **[sql] [bug] [mssql]**

    从这个票据中导致的回归导致不支持的关键字“true”被呈现，添加逻辑将其转换为 SQL 服务器的 1/0。

    参考：[#2682](https://www.sqlalchemy.org/trac/ticket/2682)

### postgresql

+   **[postgresql] [feature]**

    添加了对 PostgreSQL 9.2 范围类型的支持。目前，不提供类型转换，因此目前直接使用字符串或 psycopg2 2.5 范围扩展类型。补丁由 Chris Withers 提供。

+   **[postgresql] [feature]**

    在使用 psycopg2 DBAPI 时，添加了对“AUTOCOMMIT”隔离的支持。该关键字可通过`isolation_level`执行选项使用。补丁由 Roman Podolyaka 提供。

    参考：[#2072](https://www.sqlalchemy.org/trac/ticket/2072)

+   **[postgresql] [bug]**

    在 PostgreSQL 方言上，`extract()` 的行为已经简化，不再向给定表达式注入硬编码的`::timestamp`或类似转换，因为这会干扰诸如时区感知日期时间之类的类型，但在现代版本的 psycopg2 中似乎也不是必要的。

    参考：[#2740](https://www.sqlalchemy.org/trac/ticket/2740)

+   **[postgresql] [bug]**

    修复了 HSTORE 类型中包含反斜杠引号的键/值在使用“非本地”（即非-psycopg2）方式转换 HSTORE 数据时无法正确转义的错误。补丁由 Ryan Kelly 提供。

    参考：[#2766](https://www.sqlalchemy.org/trac/ticket/2766)

+   **[postgresql] [bug]**

    修复了多列 PostgreSQL 索引中列的顺序会反映错误顺序的错误。由 Roman Podolyaka 提供。

    参考：[#2767](https://www.sqlalchemy.org/trac/ticket/2767)

+   **[postgresql] [bug]**

    修复了 HSTORE 类型以正确对 unicode 进行编码/解码。这始终是开启的，因为 hstore 是一种文本类型，并且与使用 Python 3 时 psycopg2 的行为相匹配。感谢 Dmitry Mugtasimov。

    参考：[#2735](https://www.sqlalchemy.org/trac/ticket/2735)

### mysql

+   **[mysql] [feature]**

    可以将用于`Index`的`mysql_length`参数作为列名/长度的字典传递，用于复合索引。非常感谢 Roman Podolyaka 提供的补丁。

    参考：[#2704](https://www.sqlalchemy.org/trac/ticket/2704)

+   **[mysql] [bug]**

    修复了使用多表 UPDATE 时的错误，其中一个辅助表是带有自己绑定参数的 SELECT，当使用 MySQL 的特殊语法时，绑定参数的位置与语句本身相反。

    参考：[#2768](https://www.sqlalchemy.org/trac/ticket/2768)

+   **[mysql] [bug]**

    在`mysql+gaerdbms`方言中添加了另一个条件，以检测所谓的“开发”模式，在这种模式下，我们应该使用`rdbms_mysqldb` DBAPI。感谢 Brett Slatkin 提供补丁。

    参考：[#2715](https://www.sqlalchemy.org/trac/ticket/2715)

+   **[mysql] [bug]**

    在`ForeignKey`和`ForeignKeyConstraint`上的`deferrable`关键字参数将不会在 MySQL 方言上呈现`DEFERRABLE`关键字。长期以来，我们一直保留了这个设置，因为不可推迟的外键会与可推迟的外键表现得截然不同，但是某些环境只是在 MySQL 上禁用了 FKs，所以我们在这里将不那么坚持己见。

    参考：[#2721](https://www.sqlalchemy.org/trac/ticket/2721)

+   **[mysql] [bug]**

    更新了 mysqlconnector 方言，以根据异常中发送的明显字符串消息来检查断开连接；针对 mysqlconnector 1.0.9 进行了测试。

### sqlite

+   **[sqlite] [bug]**

    将`sqlalchemy.types.BIGINT`添加到可以由 SQLite 方言反射的类型名称列表中；感谢 Russell Stuart。

    参考：[#2764](https://www.sqlalchemy.org/trac/ticket/2764)

### mssql

+   **[mssql] [bug]**

    在查询 SQL Server 2000 上的信息模式时，删除了在 0.8.1 版中添加的 CAST 调用，以帮助解决驱动程序问题，显然这不兼容于 2000。对于 SQL Server 2005 及更高版本，保留了 CAST。

    参考：[#2747](https://www.sqlalchemy.org/trac/ticket/2747)

### 杂项

+   **[feature] [firebird]**

    将新标志`retaining=True`添加到 kinterbasdb 和 fdb 方言中。这控制了发送到 DBAPI 连接的`commit()`和`rollback()`方法的`retaining`标志的值。由于历史问题，这个标志在 0.8.2 中默认为`True`，但是在 0.9.0b1 中，这个标志默认为`False`。

    参考：[#2763](https://www.sqlalchemy.org/trac/ticket/2763)

+   **[bug] [firebird]**

    修复了在反射 Firebird 类型 LONG 和 INT64 时的类型查找，现在 LONG 被视为 INTEGER，INT64 被视为 BIGINT，除非类型具有“精度”，否则将被视为 NUMERIC。补丁由 Russell Stuart 提供。

    参考：[#2757](https://www.sqlalchemy.org/trac/ticket/2757)

+   **[bug] [ext]**

    修复了一个 bug，即如果使用函数而不是类来设置复合类型，则当可变扩展尝试检查该列是否为`MutableComposite`时，可变扩展会出错（实际上不是）。感谢 asldevi。

+   **[requirements]**

    现在运行单元测试套件需要 Python 的[mock](https://pypi.org/project/mock)库。虽然作为 Python 3.3 的一部分已经包含在标准库中，但之前的 Python 安装需要安装此库才能运行单元测试或使用`sqlalchemy.testing`包来支持外部方言。

## 0.8.1

发布日期：2013 年 4 月 27 日

### orm

+   **[orm] [feature]**

    在 Query 中添加了一个方便的方法，将查询转换为`EXISTS (SELECT 1 FROM ... WHERE ...)`的子查询形式。

    参考：[#2673](https://www.sqlalchemy.org/trac/ticket/2673)

+   **[orm] [bug]**

    修复了一个查询的 bug 形式：`query(SubClass).options(subqueryload(Baseclass.attrname))`，其中`SubClass`是`BaseClass`的一个连接继承，将无法在属性加载时应用`JOIN`，从而产生笛卡尔积。填充的结果仍然往往是正确的，因为额外的行只是被忽略，所以这个问题可能会作为性能下降存在于其他方面正常工作的应用程序中。

    此更改也已**回溯**至：0.7.11

    参考：[#2699](https://www.sqlalchemy.org/trac/ticket/2699)

+   **[orm] [bug]**

    修复了一个工作单元中的 bug，即如果两个表之间没有设置外键约束，那么一个继承子类可能会在父表之前插入“子”表的行。

    此更改也已**回溯**至：0.7.11

    参考：[#2689](https://www.sqlalchemy.org/trac/ticket/2689)

+   **[orm] [bug]**

    修复了`sqlalchemy.ext.serializer`扩展的问题，包括从 pickler 传递的“id”被转换为字符串以防止在 Py3K 上解析字节，以及`relationship()`和`orm.join()`构造现在正确序列化。

    参考：[#2698](https://www.sqlalchemy.org/trac/ticket/2698)

+   **[orm] [bug]**

    对 query.join()的内部工作机制进行了重大改进，简化了如何进行连接的决策过程。现在新的测试用例通过了，例如从已经涉及继承的复杂连接系列的中间进行多个连接。从深度嵌套的子查询结构进行连接仍然很复杂，不是没有注意事项，但通过这些改进，边缘情况希望被推到更远的边缘。

    参考：[#2714](https://www.sqlalchemy.org/trac/ticket/2714)

+   **[orm] [bug]**

    为 ORM 映射对象的反序列化过程添加了条件，以便在对象被序列化时丢失对对象的引用时，我们不会错误地尝试设置 _sa_instance_state - 修复了 NoneType 错误。

+   **[orm] [bug]**

    修复了一个 bug，即当 uselist=False 的多对多关系尝试删除关联行并且标量属性设置为 None 时，会引发错误。这是由于[#2229](https://www.sqlalchemy.org/trac/ticket/2229)的更改引入的回归。

    参考：[#2710](https://www.sqlalchemy.org/trac/ticket/2710)

+   **[orm] [bug]**

    改进了实例管理在 Session 中创建强引用时的行为；如果对象处于瞬时状态或进入分离状态，将不再创建内部引用循环 - 只有当对象附加到 Session 时才会创建强引用，并在对象分离时删除。即使不建议这样做，这使得对象具有 __del__()方法更加安全，因为具有反向引用的关系也会产生循环。当映射具有 __del__()方法的类时，会添加警告。

    参考：[#2708](https://www.sqlalchemy.org/trac/ticket/2708)

+   **[orm] [bug]**

    修复了一个 bug，即当刷新一个继承映射类时，ORM 会运行错误类型的查询，其中超类映射到非 Table 对象，如自定义 join()或 select()，运行一个假设映射到单独 Table-per-class 层次结构的查询。

    参考：[#2697](https://www.sqlalchemy.org/trac/ticket/2697)

+   **[orm] [bug]**

    修复了在对象初始化之前，mapper 属性构造中的 __repr__()方法无法正常工作的问题，因此最近的 Sphinx 版本可以读取它们。

### orm 声明式

+   **[orm] [declarative] [bug]**

    修复了关于`has_inherited_table()`的间接回归问题，因为它考虑了当前类的`__table__`，所以在调用时敏感。这也是 0.7 的行为，但在 0.7 中，事情往往会在`__mapper_args__()`等事件中“解决”。`has_inherited_table()`现在只考虑超类，因此无论何时调用它，都应该返回关于当前类的相同答案（显然假设超类的状态）。

    参考：[#2656](https://www.sqlalchemy.org/trac/ticket/2656)

### 例子

+   **[examples] [bug]**

    修复了缓存示例中长期存在的一个 bug，即在计算缓存键时，limit/offset 参数值不会被考虑。 _key_from_query() 函数已经简化，直接从最终编译的语句中工作，以便获取完整的语句以及完全处理过的参数列表。

### sql

+   **[sql] [feature]**

    放宽了传递给 Table()的特定于方言的参数名称的检查；由于我们希望支持外部方言，并且希望支持未安装某个特定方言的参数，因此现在只检查参数的格式，而不再在 sqlalchemy.dialects 中查找该方言。

+   **[sql] [bug] [mysql]**

    完全实现了 IS 和 IS NOT 运算符与 True/False 常量相关。例如，`col.is_(True)`表达式现在将在目标平台上呈现`col IS true`，而不是将 True/False 常量转换为整数绑定参数。这使得`is_()`运算符在给定 True/False 常量时可以在 MySQL 上工作。

    参考：[#2682](https://www.sqlalchemy.org/trac/ticket/2682)

+   **[sql] [bug]**

    对于使用 apply_labels()时 select()对象生成带标签列的方式进行了重大修复；这种模式生成一个 SELECT，其中每列都标记为<tablename>_<columnname>，以消除多表选择的列名冲突。修复的地方是，如果两个标签与表名组合时发生冲突，即“foo.bar_id”和“foo_bar.id”，则将对其中一个重复项应用匿名别名。这允许 ORM 独立处理两个列；以前，0.7 在某些情况下会静默地为“重复项”发出第二个 SELECT，并且在 0.8 中会发出模棱两可的列错误。应用于 select()的.c.集合的“键”也将被去重，因此对于指定了 use_labels 的任何 select()，将不再为“被替换的列”警告发出，尽管重复键将被赋予一个通常不友好的匿名标签。

    参考：[#2702](https://www.sqlalchemy.org/trac/ticket/2702)

+   **[sql] [bug]**

    修复了一个 bug，即在连接对象已经关闭后，如果错误在断开连接时检测到，则会引发属性错误。

    参考资料：[#2691](https://www.sqlalchemy.org/trac/ticket/2691)

+   **[sql] [bug]**

    重新设计了在重新引发之前发出 rollback()的内部异常引发，以便在进入 rollback 之前保留 sys.exc_info()的堆栈跟踪。这样，在使用可能在回滚函数返回之前切换上下文的协程框架时，堆栈跟踪将得以保留。

    参考资料：[#2703](https://www.sqlalchemy.org/trac/ticket/2703)

+   **[sql] [bug] [postgresql]**

    现在，当在 Python 3 上运行时，_Binary 基本类型通过可调用的 bytes()将值转换为字节;特别是，带有 Python 3.3 的 psycopg2 2.5 现在似乎返回“memoryview”类型，因此在返回之前将其转换为字节。

+   **[sql] [bug]**

    对 Connection 自动失效处理进行了改进。如果发生非断开连接错误，但在错误处理中导致延迟断开连接错误（在 MySQL 中发生），则会检测到断开连接条件。现在，即使处于无效状态，Connection 也可以关闭，这意味着在下次使用时它将引发“closed”，此外，“close with result”功能也将在错误处理例程中的 autorollback 失败时以及无论条件是否为断开连接都会起作用。

    参考资料：[#2695](https://www.sqlalchemy.org/trac/ticket/2695)

+   **[sql] [bug]**

    修复了一个 bug，即 DBAPI 可能会对 cursor.lastrowid 返回“0”，并且在与`ResultProxy.inserted_primary_key`结合使用时将无法正常工作。

### postgresql

+   **[postgresql] [bug]**

    将对 psycopg2/libpq 的“断开连接”检查扩展为在完整异常层次结构中检查所有各种“断开连接”消息。具体来说，“意外关闭连接”的消息现在已经至少在三种不同的异常类型中看到。由 Eli Collins 提供。

    参考资料：[#2712](https://www.sqlalchemy.org/trac/ticket/2712)

+   **[postgresql] [bug]**

    PostgreSQL ARRAY 类型的运算符支持输入类型为集合、生成器等，即使未指定维度，也会将给定的可迭代对象无条件转换为集合。

    参考资料：[#2681](https://www.sqlalchemy.org/trac/ticket/2681)

+   **[postgresql] [bug]**

    将 HSTORE 类型添加到 postgresql 类型名称中，以便可以反射该类型。

    参考资料：[#2680](https://www.sqlalchemy.org/trac/ticket/2680)

### mysql

+   **[mysql] [bug]**

    修复了支持最新的 cymysql DBAPI 所需的问题，由 Hajime Nakagami 提供。

+   **[mysql] [bug]**

    对 Python 3 上 pymysql 方言的操作进行了改进，包括一些重要的解码/字节步骤。由于驱动程序问题，BLOB 类型仍然存在问题。由 Ben Trofatter 提供。

    参考资料：[#2663](https://www.sqlalchemy.org/trac/ticket/2663)

+   **[mysql] [bug]**

    更新了一个正则表达式，以正确提取 google app engine v1.7.5 及更新版本的错误代码。由 Dan Ring 提供。

### mssql

+   **[mssql] [bug]**

    作为为 pyodbc+ mssql 所需的一系列修复的一部分，已向所有信息模式查询的绑定参数的表名和模式名添加了 CAST 到 NVARCHAR(max)，以避免将 NVARCHAR 与 NTEXT 进行比较的问题，在某些情况下，例如 FreeTDS（仅限 0.91？）加上传递的 unicode 绑定参数。该问题似乎特定于 SQL Server 信息模式表，并且解决方法对于那些问题本来就不存在的情况是无害的。

    参考：[#2355](https://www.sqlalchemy.org/trac/ticket/2355)

+   **[mssql] [bug]**

    为 pymssql 方言添加了对额外“disconnect”消息的支持。感谢 John Anderson。

+   **[mssql] [bug]**

    修复了关于“binary”类型和 pymssql 的 Py3K bug。感谢 Marc Abramowitz。

    参考：[#2683](https://www.sqlalchemy.org/trac/ticket/2683)

## 0.8.0

发布日期：2013 年 3 月 9 日

注意

0.8.0 中存在一些新的行为变化，而在 0.8.0b2 中不存在。它们在迁移文档中如下所示：

+   将“pending”对象视为“孤立”对象的考虑更加激进

+   create_all()和 drop_all()现在将尊重空列表

+   相关性现在始终是特定于上下文的

### orm

+   **[orm] [feature]**

    添加了有意义的`QueryableAttribute.info`属性，它代理到直接存在的`Column`对象的`.info`属性，否则代理到`MapperProperty`。完整的行为已记录并通过测试确保保持稳定。

    参考：[#2675](https://www.sqlalchemy.org/trac/ticket/2675)

+   **[orm] [feature]**

    在`relationship()`构造已经构建后，可以设置/更改“cascade”属性。这不是正常使用的模式，但我们喜欢在教程中更改设置以进行演示。

+   **[orm] [feature]**

    添加了新的辅助函数`was_deleted()`，如果给定对象是`Session.delete()`操作的主题，则返回 True。

    参考：[#2658](https://www.sqlalchemy.org/trac/ticket/2658)

+   **[orm] [feature]**

    扩展了运行时检查 API 系统，以便检索与 ORM 或其扩展相关的所有 Python 描述符。 这满足了能够检查所有 `QueryableAttribute` 描述符的常见请求，以及扩展类型，例如 `hybrid_property` 和 `AssociationProxy`。 请参阅 `Mapper.all_orm_descriptors`。

+   **[orm] [bug]**

    在映射器配置期间改进了对现有反向引用名称冲突的检查；现在将在超类和子类上测试名称冲突，除了当前映射器之外，因为这些冲突会造成同样的问题。 这对于 0.8 是新的，但请参阅下面对于将在 0.7.11 中触发的警告。

    参考：[#2674](https://www.sqlalchemy.org/trac/ticket/2674)

+   **[orm] [bug]**

    当检测到“反向引用循环”时，会发出清晰的错误消息，即当属性事件触发两个其他属性之间的双向赋值时。 当一个对象的类型错误地被赋值时，此条件可能会发生，但是当属性被错误地配置为反向引用到现有的反向引用对时，也会发生。 也出现在 0.7.11 中。

    参考：[#2674](https://www.sqlalchemy.org/trac/ticket/2674)

+   **[orm] [bug]**

    如果将 MapperProperty 分配给替换现有属性的映射器，则会发出警告，如果问题属性不是基于纯列的属性。 替换关系属性很少（或者说从来没有？）是预期的，通常是指映射器配置错误。 也出现在 0.7.11 中。

    参考：[#2674](https://www.sqlalchemy.org/trac/ticket/2674)

+   **[orm] [bug]**

    如果事件处理程序在 after_commit() 处理程序中尝试在会话中发出 SQL，而在此期间没有可行的事务，则会发出清晰的错误消息。

    参考：[#2662](https://www.sqlalchemy.org/trac/ticket/2662)

+   **[orm] [bug]**

    在级联自然主键更新过程中检测到主键更改将会成功，即使该键是复合键，且仅有部分属性发生了变化。

    参考：[#2665](https://www.sqlalchemy.org/trac/ticket/2665)

+   **[orm] [bug]**

    从会话中删除的对象在事务提交后将完全取消与该会话的关联，即 `object_session()` 函数将返回 None。

    参考：[#2658](https://www.sqlalchemy.org/trac/ticket/2658)

+   **[orm] [bug]**

    修复了一个 bug，`Query.yield_per()` 会错误地设置执行选项，从而破坏了后续使用 `Query.execution_options()` 方法的情况。感谢 Ryan Kelly。

    参考：[#2661](https://www.sqlalchemy.org/trac/ticket/2661)

+   **[orm] [bug]**

    修复了对 `between()` 操作符的考虑，以便它正确地与新的关系本地/远程系统配合使用。

    参考：[#1768](https://www.sqlalchemy.org/trac/ticket/1768)

+   **[orm] [bug]**

    将待处理对象视为“孤立对象”的考虑已经修改，以更接近持久对象的行为，即一旦对象与其任何启用了孤立模式的父对象解除关联，该对象就会从`Session`中清除。之前，只有在对象从所有启用了孤立模式的父对象中解除关联时，待处理对象才会被清除。新标志 `legacy_is_orphan` 被添加到`Mapper`中，以恢复传统行为。

    详细讨论此更改的变更说明和示例案例，请参阅将“待处理”对象视为“孤立”对象的考虑变得更为激进。

    参考：[#2655](https://www.sqlalchemy.org/trac/ticket/2655)

+   **[orm] [bug]**

    修复了（很可能从未被使用过的）“@collection.link”集合方法，该方法在每次将集合与映射对象关联或取消关联时触发 - 该装饰器未经测试或功能上不正常。装饰器方法现在命名为 `collection.linker()`，尽管名称“link”仍然保留了向后兼容性。感谢 Luca Wehrstedt。

    参考：[#2653](https://www.sqlalchemy.org/trac/ticket/2653)

+   **[orm] [bug]**

    对生成自定义受监控集合系统进行了一些修复，主要是现在使用@collection 装饰器将遵循给定类的 __mro__，应用特定集合方法的子类的版本的逻辑。以前，在对现有受监控类进行子类化时，例如 `MappedCollection`，无法预测自定义方法是否会正确解析。

    参考：[#2654](https://www.sqlalchemy.org/trac/ticket/2654)

+   **[orm] [bug]**

    修复了可能发生的内存泄漏问题，如果创建了任意数量的`sessionmaker`对象。当 sessionmaker 创建的匿名子类被取消引用时，由于事件包中仍然存在类级别的引用，该子类将无法被垃圾回收。此问题也适用于任何与事件调度程序一起使用临时子类的自定义系统。也适用于 0.7.10 版本。

    参考：[#2650](https://www.sqlalchemy.org/trac/ticket/2650)

+   **[orm] [错误]**

    `Query.merge_result()`现在可以从外连接加载行，其中实体可能为`None`而不会引发错误。也��用于 0.7.10 版本。

    参考：[#2640](https://www.sqlalchemy.org/trac/ticket/2640)

+   **[orm] [错误]**

    修复了`relationship()`上“动态”加载器的问题，包括在自动刷新被禁用时，反向引用将正常工作，历史事件在同一对象多次添加/移除的情况下更加准确。

    参考：[#2637](https://www.sqlalchemy.org/trac/ticket/2637)

+   **[orm] [已移除]**

    已删除了使用与集合相关联的`__instrumentation__`数据结构生成自定义集合的未记录（希望未被使用）系统，因为这是一个复杂且未经测试的功能，与装饰器方法基本重复。还对 orm.collections 模块进行了其他内部简化。

### 示例

+   **[示例] [错误]**

    修复了示例/dogpile_caching 示例中的回归，这是由于[#2614](https://www.sqlalchemy.org/trac/ticket/2614)的更改引起的。

### sql

+   **[sql] [功能]**

    为`Enum`及其基本`SchemaType`添加了一个新参数`inherit_schema`。当设置为`True`时，该类型将设置其`schema`属性为其关联的`Table`的`schema`。在进行`Table.tometadata()`操作时也会发生这种情况；当`Table.tometadata()`发生时，无论何种情况下都会复制`SchemaType`，如果`inherit_schema=True`，则该类型将采用传递给该方法的新模式名称。在与 PostgreSQL 后端一起使用时，`schema`非常重要，因为该类型会导致`CREATE TYPE`语句。

    参考：[#2657](https://www.sqlalchemy.org/trac/ticket/2657)

+   **[sql] [功能]**

    `Index` 现在支持任意的 SQL 表达式和/或函数，除了直接列。常见的修饰符包括使用`somecolumn.desc()`来创建降序索引和`func.lower(somecolumn)`来创建不区分大小写的索引，具体取决于目标后端的功能。

    参考：[#695](https://www.sqlalchemy.org/trac/ticket/695)

+   **[sql] [bug]**

    改进了 SELECT 关联的行为，使得`Select.correlate()` 和 `Select.correlate_except()` 方法，以及它们的 ORM 类似方法，在 FROM 子句仅在输出为合法 SQL 时才被修改；也就是说，如果关联的 SELECT 未在 WHERE、columns 或 HAVING 子句的上下文中使用，则 FROM 子句保持不变。这两种方法现在只指定默认的“自动关联”条件，而不是绝对的 FROM 列表。

    参考：[#2668](https://www.sqlalchemy.org/trac/ticket/2668)

+   **[sql] [bug]**

    修复了关于列注释的一个 bug，特别是可能影响到新的`remote()`和`local()`注释函数的一些用法，当列在后续表达式中使用时，注释可能会丢失。

    参考：[#1768](https://www.sqlalchemy.org/trac/ticket/1768), [#2660](https://www.sqlalchemy.org/trac/ticket/2660)

+   **[sql] [bug]**

    `ColumnOperators.in_()` 操作符现在将`None`的值强制转换为`null()`。

    参考：[#2496](https://www.sqlalchemy.org/trac/ticket/2496)

+   **[sql] [bug]**

    修复了一个 bug，当`Table.tometadata()`中的`Column`既有外键又有列的替代“.key”名称时会失败。也适用于 0.7.10 版本。

    参考：[#2643](https://www.sqlalchemy.org/trac/ticket/2643)

+   **[sql] [bug]**

    如果在不支持 RETURNING 的方言上尝试编译 insert().returning()，将引发一个信息性的 CompileError。

    参考：[#2629](https://www.sqlalchemy.org/trac/ticket/2629)

+   **[sql] [bug]**

    调整了编译器用于识别需要传递的 INSERT/UPDATE 绑定参数的“REQUIRED”符号，使得在编写自定义绑定处理代码时更���易识别。

    参考：[#2648](https://www.sqlalchemy.org/trac/ticket/2648)

### 模式

+   **[schema] [bug]**

    `MetaData.create_all()` 和 `MetaData.drop_all()` 现在将接受一个空列表作为指示，不创建/删除任何项，而不是忽略该集合。

    参考：[#2664](https://www.sqlalchemy.org/trac/ticket/2664)

### postgresql

+   **[postgresql] [feature]**

    添加了对 PostgreSQL 传统 SUBSTRING 函数语法的支持，当使用常规`func.substring()`时，呈现为“SUBSTRING(x FROM y FOR z)”。感谢 Gunnlaugur Þór Briem。

    此更改也**回溯**到：0.7.11

    参考：[#2676](https://www.sqlalchemy.org/trac/ticket/2676)

+   **[postgresql] [feature]**

    添加了`Comparator.any()`和`Comparator.all()`方法，以及独立的表达式构造。非常感谢 Audrius Kažukauskas 在这里的出色工作。

+   **[postgresql] [bug]**

    修复了`array()`构造中的错误，其中在`insert()`构造中使用它会产生关于`self_group()`方法中参数问题的错误。

### mysql

+   **[mysql] [feature]**

    添加了 CyMySQL 的新方言，感谢 Hajime Nakagami。

+   **[mysql] [feature]**

    GAE 方言现在接受 URL 中的用户名/密码参数，感谢 Owen Nelson。

+   **[mysql] [bug] [gae]**

    在`gaerdbms`方言中添加了一个条件导入，尝试导入 rdbms_apiproxy vs. rdbms_googleapi 以在开发和生产平台上工作。现在也支持`instance`属性。感谢 Sean Lynch。也在 0.7.10 中。

    参考：[#2649](https://www.sqlalchemy.org/trac/ticket/2649)

+   **[mysql] [bug]**

    如果无法从异常抛出中提取错误代码，GAE 方言不会在无匹配时失败；感谢 Owen Nelson。

### mssql

+   **[mssql] [feature]**

    添加了`mssql_include`和`mssql_clustered`选项到`Index`，分别呈现`INCLUDE`和`CLUSTERED`关键字。感谢 Derek Harland。

+   **[mssql] [feature]**

    现在支持对非主键列的 IDENTITY 列的 DDL，通过在任何整数列上建立一个`Sequence`构造。感谢 Derek Harland。

    参考：[#2644](https://www.sqlalchemy.org/trac/ticket/2644)

+   **[mssql] [bug]**

    在 mssql 信息模式中添加了一个 py3K 条件，围绕不必要的.decode()调用，修复了 Py3K 中的反射问题。也在 0.7.10 中。

    参考：[#2638](https://www.sqlalchemy.org/trac/ticket/2638)

+   **[mssql] [bug]**

    修复了字符类型 CHAR、NCHAR 等的“collation”参数停止工作的回归，因为“collation”现在由基本字符串类型支持。MSSQL 方言中的 TEXT、NCHAR、CHAR、VARCHAR 类型现在是基本类型的同义词。

### oracle

+   **[oracle] [bug]**

    cx_oracle 方言将不再通过 `encode()` 运行绑定参数名称，因为这在 Python 3 上无效，并且阻止了 Python 3 上语句的正确运行。现在只有在 `supports_unicode_binds` 为 False 时才进行编码，而当至少使用版本 5 的 cx_oracle 时，这并不适用于 cx_oracle。

### tests

+   **[tests] [bug]**

    修复了在一些 Linux 平台上无法正常工作的 test_execute 中的“logging”导入。也在 0.7.11 中。

    参考：[#2669](https://www.sqlalchemy.org/trac/ticket/2669)

## 0.8.0b2

发布日期：2012 年 12 月 14 日

### orm

+   **[orm] [feature]**

    添加了 `KeyedTuple._asdict()` 和 `KeyedTuple._fields` 到 `KeyedTuple` 类，以提供与 Python 标准库 `collections.namedtuple()` 一定程度的兼容性。

    参考：[#2601](https://www.sqlalchemy.org/trac/ticket/2601)

+   **[orm] [feature]**

    允许在定义关系的主要和次要连接时使用同义词。

+   **[orm] [bug]**

    `Query.select_from()` 方法现在可以与`aliased()`构造一起使用，而不会干扰被选择的实体。基本上，像这样的语句：

    ```py
    ua = aliased(User)
    session.query(User.name).select_from(ua).join(User, User.name > ua.name)
    ```

    将保持 SELECT 的列子句来自未别名化的“user”，如指定的；select_from 只发生在 FROM 子句中：

    ```py
    SELECT  users.name  AS  users_name  FROM  users  AS  users_1
    JOIN  users  ON  users.name  <  users_1.name
    ```

    请注意，这种行为与原始、较旧的 `Query.select_from()` 的用例形成对比，即在不同的可选择性方面重新陈述映射实体：

    ```py
    session.query(User.name).select_from(user_table.select().where(user_table.c.id > 5))
    ```

    产生：

    ```py
    SELECT  anon_1.name  AS  anon_1_name  FROM  (SELECT  users.id  AS  id,
    users.name  AS  name  FROM  users  WHERE  users.id  >  :id_1)  AS  anon_1
    ```

    后一种用例的“别名”行为妨碍了前一种用例。该方法现在明确地将类似`select()`或`alias()`的 SQL 表达式与类似`aliased()`构造的映射实体分开考虑。

    参考：[#2635](https://www.sqlalchemy.org/trac/ticket/2635)

+   **[orm] [bug]**

    `MutableComposite`类型不允许使用`MutableBase.coerce()`方法，尽管代码似乎表明了这一意图，所以现在可以使用，并添加了一个简短的示例。作为副作用，此事件处理程序的机制已更改，以便新的`MutableComposite`类型不再添加每种类型的全局事件处理程序。也适用于 0.7.10。

    参考：[#2624](https://www.sqlalchemy.org/trac/ticket/2624)

+   **[orm] [错误]**

    第二次对别名/内部路径机制进行改进，现在允许两个子类具有相同名称的不同关系，在同时使用子查询或连接式贪婪加载时支持全多态加载。

    参考：[#2614](https://www.sqlalchemy.org/trac/ticket/2614)

+   **[orm] [错误]**

    修复了一个 bug，即在特定的 with_polymorphic 加载中进行多跳子查询加载会产生 KeyError 的问题。利用了与[#2614](https://www.sqlalchemy.org/trac/ticket/2614)相同的内部路径改进。

    参考：[#2617](https://www.sqlalchemy.org/trac/ticket/2617)

+   **[orm] [错误]**

    修复了查询.update()在“fetch”同步策略匹配的对象不在本地时会产生错误的回归。感谢 Scott Torborg。

    参考：[#2602](https://www.sqlalchemy.org/trac/ticket/2602)

### orm 扩展

+   **[orm] [扩展] [功能]**

    `sqlalchemy.ext.mutable`扩展现在包括示例`MutableDict`类作为扩展的一部分。

### engine

+   **[engine] [功能]**

    `Connection.connect()`和`Connection.contextual_connect()`方法现在返回一个“branched”版本，以便在返回的连接上调用`Connection.close()`方法而不影响原始连接。在使用`Engine`和`Connection`对象作为上下文管理器时提供对称性：

    ```py
    with conn.connect() as c:  # leaves the Connection open
        c.execute("...")

    with engine.connect() as c:  # closes the Connection
        c.execute("...")
    ```

+   **[engine] [错误]**

    修复了 `MetaData.reflect()` 方法，正确使用给定的 `Connection`，如果有的话，而不是从该连接的 `Engine` 中打开第二个连接。

    此更改也已**回溯**至：0.7.10

    参考：[#2604](https://www.sqlalchemy.org/trac/ticket/2604)

+   **[引擎]**

    `MetaData` 的“reflect=True”参数已被弃用。请使用 `MetaData.reflect()` 方法。

### sql

+   **[sql] [feature]**

    `Insert` 构造现在支持多值插入，即类似于“INSERT INTO table VALUES (…), (…), …” 的插入方式。支持 PostgreSQL、SQLite 和 MySQL。特别感谢 Idan Kamara 在这方面的工作。

    另请参阅

    Insert 的多值支持

    参考：[#2623](https://www.sqlalchemy.org/trac/ticket/2623)

+   **[sql] [bug]**

    修复了在使用 server_onupdate=<FetchedValue|DefaultClause> 但没有传递“for_update=True”标志时，会将默认对象应用于 server_default，覆盖原有内容的 bug。这种用法不应该需要显式的 for_update=True 参数（特别是因为文档中显示了一个没有使用该参数的示例），因此现在在内部使用给定默认对象的副本，如果标志未设置为对应该参数的值。

    此更改也已**回溯**至：0.7.10

    参考：[#2631](https://www.sqlalchemy.org/trac/ticket/2631)

+   **[sql] [bug]**

    修复了由 [#2410](https://www.sqlalchemy.org/trac/ticket/2410) 引起的回归，即 `CheckConstraint` 在 `Table.tometadata()` 操作期间会将自身应用回原始表，因为它会解析父表的 SQL 表达式。现在该操作会将给定的表达式复制以对应新表。

    参考：[#2633](https://www.sqlalchemy.org/trac/ticket/2633)

+   **[sql] [bug]**

    修复了在方言上使用 label_length 小于实际列标识符大小的 bug，会导致在 SELECT 语句中无法正确渲染列。

    参考：[#2610](https://www.sqlalchemy.org/trac/ticket/2610)

+   **[sql] [bug]**

    `DECIMAL` 类型现在在渲染 DDL 时遵守“precision”和“scale”参数。

    参考：[#2618](https://www.sqlalchemy.org/trac/ticket/2618)

+   **[sql] [bug]**

    对二进制表达式的“布尔”（即`__nonzero__`）评估进行了调整，即`x1 == x2`，使得`BinaryExpression`在某些情况下的“自动分组”不会妨碍此比较。 以前，像这样的表达式：

    ```py
    expr1 = mycolumn > 2
    bool(expr1 == expr1)
    ```

    尽管这是一个身份比较，但会评估为`False`，因为`mycolumn > 2`会在放入`BinaryExpression`之前被“分组”，从而改变其身份。 `BinaryExpression`现在跟踪传入的“原始”对象。 此外，`__nonzero__`方法现在仅在运算符为`==`或`!=`时返回 - 所有其他情况都会引发`TypeError`。

    参考：[#2621](https://www.sqlalchemy.org/trac/ticket/2621)

+   **[sql] [bug]**

    修复了一个问题，即无意中在`ColumnElement`上调用 list()会进入无限循环，如果实现了`ColumnOperators.__getitem__()`。 通过`__iter__()`发出新的 NotImplementedError。

+   **[sql] [bug]**

    修复了`type_coerce()`中的一个 bug，即如果语句被用作另一个语句内部的子查询，以及其他类似情况，可能会丢失类型信息。 其中之一是当 Oracle/mssql 方言应用 limit/offset 包装时，会导致类型信息丢失。

    参考：[#2603](https://www.sqlalchemy.org/trac/ticket/2603)

+   **[sql] [bug]**

    修复了一个 bug，即在生成可选择的列的“代理”时，未使用列的“.key”��� 这可能在 0.7 中没有发生，因为 0.7 在更广泛的情况下不尊重“.key”。

    参考：[#2597](https://www.sqlalchemy.org/trac/ticket/2597)

### postgresql

+   **[postgresql] [feature]**

    `HSTORE`现在在 PostgreSQL 方言中可用。 如果可用，还将使用 psycopg2 的扩展。 感谢 Audrius Kažukauskas。

    参考：[#2606](https://www.sqlalchemy.org/trac/ticket/2606)

### sqlite

+   **[sqlite] [bug]**

    对此与 SQLite 相关的问题进行了更多调整，该问题在 0.7.9 中发布，以拦截反映外键时的传统 SQLite 引号字符。 除了拦截双引号外，现在还拦截其他引号字符，如括号、反引号和单引号。

    此更改也**回溯**到：0.7.10

    参考：[#2568](https://www.sqlalchemy.org/trac/ticket/2568)

### mssql

+   **[mssql] [功能]**

    支持反射“主键约束”的“名称”，感谢戴夫·摩尔。

    参考：[#2600](https://www.sqlalchemy.org/trac/ticket/2600)

+   **[mssql] [错误]**

    修复了在使用 Column 的“key”与拥有表的“schema”结合时，由于 MSSQL 方言的“schema 渲染”逻辑未考虑 .key 而导致无法定位结果行的错误。

    此更改也**回溯**到：0.7.10

### oracle

+   **[oracle] [错误]**

    修复了在访问引用到 DBLINK 远程数据库的同义词时，Oracle 的表反射问题；虽然该语法在 Oracle 方言中已存在一段时间，但直到现在还未经过测试。该语法已针对链接到自身的示例数据库进行了测试，但在查询远程数据库的表信息时仍存在一些不确定性。目前，从 user_db_links 中使用的“用户名”值用于匹配“所有者”。

    参考：[#2619](https://www.sqlalchemy.org/trac/ticket/2619)

+   **[oracle] [错误]**

    Oracle 的 LONG 类型，虽然是一个无界文本类型，但在返回结果行时似乎不使用 cx_Oracle.LOB 类型，因此方言已修复以排除 LONG 从应用 cx_Oracle.LOB 过滤。也在 0.7.10 中。

    参考：[#2620](https://www.sqlalchemy.org/trac/ticket/2620)

+   **[oracle] [错误]**

    修复了在与 cx_Oracle 结合使用 `.prepare()` 时，如果返回值为 `False`，则不会调用 `connection.commit()`，从而避免“无事务”错误。已经证明 SQLAlchemy 和 cx_oracle 可以以一种基本方式工作，但受到驱动程序的注意事项的影响；请查看文档以��取详细信息。也在 0.7.10 中。

    参考：[#2611](https://www.sqlalchemy.org/trac/ticket/2611)

### 杂项

+   **[功能] [sybase]**

    Sybase 方言现在添加了反射支持。非常感谢本·特罗法特为开发和测试所做的所有工作。

    参考：[#1753](https://www.sqlalchemy.org/trac/ticket/1753)

+   **[功能] [池]**

    `Pool` 现在将记录所有 connection.close() 操作，包括对无效连接、分离连接和超出池容量的连接的关闭。

+   **[功能] [池]**

    `Pool` 现在咨询 `Dialect` 关于连接应如何“自动回滚”以及关闭的功能。这使得方言对事务范围有更多控制，因此我们将更好地实现对 pysqlite 和 cx_oracle 可能需要的事务解决方法。

    参考：[#2611](https://www.sqlalchemy.org/trac/ticket/2611)

+   **[特性] [池]**

    添加了新的 `PoolEvents.reset()` 钩子来捕获连接自动回滚前的事件，返回到池中。与 `ConnectionEvents.rollback()` 一起，这允许拦截所有回滚事件。

+   **[错误] [firebird]**

    在实验性的“firebird+fdb”方言中添加了对“fdb”的丢失导入。

    参考：[#2622](https://www.sqlalchemy.org/trac/ticket/2622)

+   **[informix]**

    已删除一些关于 informix 事务处理的垃圾，包括跳过调用 commit()/rollback() 以及在 begin() 上的一些硬编码隔离级别假设的特性。对该方言的状态理解不深，因为我们没有任何与之相关的用户，也没有任何访问 Informix 数据库的权限。如果有人有权访问 Informix 并愿意帮助测试该方言，请告诉我们。

## 0.8.0b1

发布日期：2012 年 10 月 30 日

### 通用

+   **[通用] [移除]**

    完全移除了“sqlalchemy.exceptions”作为“sqlalchemy.exc”的同义词。

    参考：[#2433](https://www.sqlalchemy.org/trac/ticket/2433)

+   **[通用]**

    SQLAlchemy 0.8 现在针对 Python 2.5 及以上版本。不再支持 Python 2.4。

### orm

+   **[orm] [特性]**

    relationship() 的主要重写允许加入条件，其中包括指向自身的列在复合外键内。添加了一个新的非常专业化的 primaryjoin 条件的 API，允许在需要时在表达式内联中放置注释函数 remote() 和 foreign() 来处理基于 SQL 函数、CAST 等的条件。之前使用半私有 _local_remote_pairs 方法的配方可以升级到这种新方法。

    另见

    重写的 _orm.relationship() 机制

    参考：[#1401](https://www.sqlalchemy.org/trac/ticket/1401)

+   **[orm] [特性]**

    新的独立函数 with_polymorphic() 提供了 query.with_polymorphic() 功能的独立形式。它可以应用于查询中的任何实体，包括作为“of_type()”修改器的目标进行连接。

    参考：[#2333](https://www.sqlalchemy.org/trac/ticket/2333)

+   **[orm] [特性]**

    属性上的 of_type() 结构现在也接受别名化的类构造以及 with_polymorphic 结构，并且可以与 query.join()、any()、has() 以及也包括 eager loaders subqueryload()、joinedload()、contains_eager()

    参考：[#1106](https://www.sqlalchemy.org/trac/ticket/1106)、[#2438](https://www.sqlalchemy.org/trac/ticket/2438)

+   **[orm] [特性]**

    对映射类的事件监听进行改进，允许指定未映射类用于实例和映射器事件。当传递 propagate=True 标志时，建立的事件将自动设置在该类的子类上，如果最终映射了该类，则事件将为该类本身设置。

    参考：[#2585](https://www.sqlalchemy.org/trac/ticket/2585)

+   **[orm] [feature]**

    “延迟声明式反射”系统已经移入声明式扩展本身，使用新的 DeferredReflection 类。该类现在已经针对单表和联合表继承用例进行了测试。

    参考：[#2485](https://www.sqlalchemy.org/trac/ticket/2485)

+   **[orm] [feature]**

    添加了新的核心函数“inspect()”，它作为对映射器、对象等进行内省的通用入口。Mapper 和 InstanceState 对象已经增强了公共 API，允许检查映射属性，包括针对列绑定或关系绑定属性的过滤器，检查当前对象状态，属性历史记录等。

    参考：[#2208](https://www.sqlalchemy.org/trac/ticket/2208)

+   **[orm] [feature]**

    在 session.begin_nested() 中调用 rollback() 现在只会使那些在该事务范围内具有净变化的对象过期，即在 flush 时被标记为脏或被修改的对象。这允许 begin_nested() 的典型用例，即修改一小部分对象，保留未在子事务中修改的较大对象集的数据。

    参考：[#2452](https://www.sqlalchemy.org/trac/ticket/2452)

+   **[orm] [feature]**

    添加了实用功能 Session.enable_relationship_loading()，取代了 relationship.load_on_pending。然而，应避免使用这两个功能。

    参考：[#2372](https://www.sqlalchemy.org/trac/ticket/2372)

+   **[orm] [feature]**

    添加了对 column_property()、relationship()、composite() 的 .info 字典参数支持。所有 MapperProperty 类都具有可用的自动创建的 .info 字典。

+   **[orm] [feature]**

    从映射集合中添加/移除 None 现在会生成属性事件。以前，在某些情况下，None 追加会被忽略。相关内容。

    参考：[#2229](https://www.sqlalchemy.org/trac/ticket/2229)

+   **[orm] [feature]**

    映射集合中存在 None 现在在 flush 期间会引发错误。以前，集合中的 None 值会被静默忽略。

    参考：[#2229](https://www.sqlalchemy.org/trac/ticket/2229)

+   **[orm] [feature]**

    现在更宽松地支持查询的 update() 方法所更新的表。现在更好地支持普通的 Table 对象，还可以使用 joined-inheritance 子类来进行 update(); 子类表将是更新的目标，如果在 WHERE 子句中引用了父表，则编译器将调用 UPDATE..FROM 语法（如果方言允许）来满足 WHERE 子句。如果在“values”字典中以对象指定列，则还支持 MySQL 的多表更新功能。PG 的 DELETE..USING 在核心中还不可用。

+   **[orm] [功能]**

    新的会话事件 after_transaction_create 和 after_transaction_end 允许跟踪新的 SessionTransaction 对象。如果检查了对象，可以确定会话何时首次处于活动状态以及何时停用。

+   **[orm] [功能]**

    查询现在可以加载包含非可哈希类型的实体/标量混合“元组”行，方法是在使用的相应 TypeEngine 对象上设置标志“hashable=False”。返回不可哈希类型（通常是列表）的自定义类型可以将此标志设置为 False。

    参考资料：[#2592](https://www.sqlalchemy.org/trac/ticket/2592)

+   **[orm] [功能]**

    查询现在默认情况下“自动关联”，方式与 select() 一样。以前，在另一个查询中使用的查询作为子查询时，需要显式调用 correlate() 方法，以便将内部的表与外部关联起来。像往常一样，correlate(None) 禁用关联。

    参考资料：[#2179](https://www.sqlalchemy.org/trac/ticket/2179)

+   **[orm] [功能]**

    现在在对象在 Session.add()、Session.merge() 等中建立在 Session.new 或 Session.identity_map 中后，会发出 after_attach 事件，以便在调用事件时这些集合中表示对象。添加了 before_attach 事件以适应需要在预附加对象中进行自动刷新的用例。

    参考资料：[#2464](https://www.sqlalchemy.org/trac/ticket/2464)

+   **[orm] [功能]**

    当在 flush 的“execute”部分使用不支持的方法时，会产生警告。这些方法包括熟悉的 add()、delete() 等，以及在 mapper 级别的 flush 事件中调用的集合和相关对象操作，如 after_insert()、after_update() 等。长期以来，明确记录了当在 flush 计划的执行中操纵 Session 时，SQLAlchemy 不能保证结果，但是用户仍然在这样做，所以现在有了一个警告。也许将来 Session 将被增强以支持在 flush 中执行这些操作，但目前不能保证结果。

+   **[orm] [功能]**

    ORM 实体可以传递到核心的 select() 构造中，以及传递到 select() 的 select_from()、correlate() 和 correlate_except() 方法中，它们将被解包为可选择的对象。

    参考资料：[#2245](https://www.sqlalchemy.org/trac/ticket/2245)

+   **[orm] [功能]**

    一些支持根据映射属性自动渲染关系连接条件，使用核心 SQL 构造。例如 select([SomeClass]).where(SomeClass.somerelationship) 将渲染 SELECT from “someclass” 并将 “somerelationship”的 primaryjoin 作为 WHERE 子句。这改变了在核心 SQL 上下文中使用“SomeClass.somerelationship”时的先前含义；先前，它将“解析”为父可选择的内容，这通常不是有用的。在 query.filter() 中也起作用。相关链接。

    参考文献：[#2245](https://www.sqlalchemy.org/trac/ticket/2245)

+   **[orm] [feature]**

    声明性基础（declarative_base()）中类的注册表现在是 WeakValueDictionary。因此，“Base”的子类如果被解除引用，将会被垃圾收集，*如果它们不被任何其他映射器/超类映射器引用*。参见本票据的下一个说明。

    参考文献：[#2526](https://www.sqlalchemy.org/trac/ticket/2526)

+   **[orm] [feature]**

    单继承声明性子类之间的列冲突，无论是否使用混合类，都可以使用文档中描述的新的@declared_attr 用法来解决。

    参考文献：[#2472](https://www.sqlalchemy.org/trac/ticket/2472)

+   **[orm] [feature]**

    declared_attr 现在可以在非混合类上使用，尽管这通常仅对单继承子类列冲突解决有用。

    参考文献：[#2472](https://www.sqlalchemy.org/trac/ticket/2472)

+   **[orm] [feature]**

    declared_attr 现在可以与不是 Column 或 MapperProperty 的属性一起使用；包括任何用户定义的值以及关联代理对象。

    参考文献：[#2517](https://www.sqlalchemy.org/trac/ticket/2517)

+   **[orm] [feature]**

    *非常有限*地支持继承映射器在类本身被解引用时被 GC 回收。映射器不能有自己的表（即只有单个表继承），而没有多态属性的情况。这允许了这样一种用例：创建一个声明性映射类的临时子类，在没有自己的表或映射指令的情况下，通过单元测试的解引用进行垃圾收集。

    参考文献：[#2526](https://www.sqlalchemy.org/trac/ticket/2526)

+   **[orm] [feature]**

    声明性现在维护一个按字符串名称和完整模块限定名称注册的类的注册表。现在可以基于关系（relationship()）中的模块限定字符串查找具有相同名称的多个类。简单类名查找，当有多个类共享相同名称时，现在会引发一个信息性的错误消息。

    参考文献：[#2338](https://www.sqlalchemy.org/trac/ticket/2338)

+   **[orm] [feature]**

    现在可以提供绑定到类的属性，这些属性可以覆盖任何非 ORM 类型的列，而不仅仅是描述符。

    参考文献：[#2535](https://www.sqlalchemy.org/trac/ticket/2535)

+   **[orm] [feature]**

    添加了 with_labels 和 reduce_columns 关键字参数到 Query.subquery()，以提供两种产生具有唯一命名列的查询的替代策略。

    参考：[#1729](https://www.sqlalchemy.org/trac/ticket/1729)

+   **[orm] [feature]**

    当对一个已经由于过期/属性刷新/集合替换而不再与父类关联的受监控集合进行附加或移除操作时，会发出警告。

    参考：[#2476](https://www.sqlalchemy.org/trac/ticket/2476)

+   **[orm] [bug]**

    如果两个表通过联合继承相关，并且 FK 依赖关系不是继承条件的一部分，则 ORM 在刷新期间会进行额外的努力来确定这种 FK 依赖关系不重要，从而节省用户使用 use_alter 指令的必要性。

    参考：[#2527](https://www.sqlalchemy.org/trac/ticket/2527)

+   **[orm] [bug]**

    现在，仅对被分配为监听器的类的后代类触发 instrumentation events class_instrument()、class_uninstrument()和 attribute_instrument()。以前，无论传递了什么“目标”参数，事件监听器都会被分配为在所有情况下监听所有类。

    参考：[#2590](https://www.sqlalchemy.org/trac/ticket/2590)

+   **[orm] [bug]**

    在发送多级子类的任意顺序或中间类缺失的情况下，with_polymorphic()会按正确顺序生成 JOIN，并且具有正确的继承表。

    参考：[#1900](https://www.sqlalchemy.org/trac/ticket/1900)

+   **[orm] [bug]**

    改进了处理共享共同基类的子类实体链的 joined/subquery eager loading，没有提供特定的“join depth”。在检测到“循环”之前，将单独链接到每个子类映射器，而不是考虑基类为“循环”的源。

    参考：[#2481](https://www.sqlalchemy.org/trac/ticket/2481)

+   **[orm] [bug]**

    Session.is_modified()上的“passive”标志不再起作用。在所有情况下，is_modified()只查看本地内存中的修改标志，不会发出任何 SQL 或调用加载器可调用/初始化程序。

    参考：[#2320](https://www.sqlalchemy.org/trac/ticket/2320)

+   **[orm] [bug]**

    当在没有设置 single-parent=True 的情况下使用 delete-orphan 级联时，发出的警告现在是一个错误。在任何情况下，ORM 在此警告后将无法正常工作。

    参考：[#2405](https://www.sqlalchemy.org/trac/ticket/2405)

+   **[orm] [bug]**

    在 flush 事件中发出的延迟加载，如 before_flush()、before_update()等，现在将像在非事件代码中一样运行，考虑到在延迟加载查询中使用的 PK/FK 值。以前，会建立特殊标志，导致延迟加载基于在 flush 中调用时父 PK/FK 值的“先前”值加载相关项目；现在，以这种方式加载的信号局限于工作单元实际需要以这种方式加载的地方。请注意，UOW 有时会在调用 before_update()事件之前加载这些集合，因此“passive_updates”的使用与否可能会影响在 flush 事件中访问时集合是否表示“旧”或“新”数据，根据延迟加载的发出时间。这种更改在极小的情况下是不兼容的，用户事件代码依赖于旧行为。

    参考：[#2350](https://www.sqlalchemy.org/trac/ticket/2350)

+   **[orm] [错误]**

    继续关于由于事件监听器导致 flush 后的额外状态；从属性角度标记为“脏”的任何状态，通常通过在 after_insert()、after_update()等中设置列属性事件，现在在所有情况下都将“历史”标志重置，而不仅仅是那些在 flush 中的实例。这样做的效果是，这种“脏”状态在 flush 后不会传递，并且不会导致 UPDATE 语句。会发出相应的警告；可以使用 set_committed_state()方法在对象上分配属性而不产生历史事件。

    参考：[#2566](https://www.sqlalchemy.org/trac/ticket/2566), [#2582](https://www.sqlalchemy.org/trac/ticket/2582)

+   **[orm] [错误]**

    修复了在混合类中逐渐出现的@declared_attr 列和直接定义列之间的断开连接。在这两种情况下，列将被应用于声明类的表，但不会应用于连接继承子类的表。以前，直接定义的列会被放在基表和子表上，这通常不是所期望的。

    参考：[#2565](https://www.sqlalchemy.org/trac/ticket/2565)

+   **[orm] [错误]**

    当父类本身映射到 join()或 select()语句时，声明可以将在单表继承子类上声明的列传播到父类的表，直接或通过连接继承，而不仅仅是一个表。

    参考：[#2549](https://www.sqlalchemy.org/trac/ticket/2549)

+   **[orm] [错误]**

    当 uselist=False 与“dynamic”加载器结合时会发出错误。这在 0.7.9 中是一个警告。

+   **[orm] [已移除]**

    ORM 的传统“可变”系统已移除，包括 MutableType 类以及 PickleType 和 postgresql.ARRAY 上的 mutable=True 标志。ORM 使用 sqlalchemy.ext.mutable 扩展来检测原地变异，该扩展是在 0.7 版本中引入的。MutableType 和相关结构的移除从 SQLAlchemy 的内部删除了大量复杂性。这种方法的性能表现不佳，因为在使用时会扫描 Session 的全部内容。

    参考：[#2442](https://www.sqlalchemy.org/trac/ticket/2442)

+   **[ORM] [已移除]**

    已弃用的标识符已移除：

    +   allow_null_pks mapper() 参数（使用 allow_partial_pks）

    +   _get_col_to_prop() 映射方法（使用 get_property_by_column()）

    +   dont_load 参数已移除至 Session.merge()（使用 load=True）

    +   sqlalchemy.orm.shard 模块（使用 sqlalchemy.ext.horizontal_shard）

+   **[ORM] [移动]**

    InstrumentationManager 接口和整个相关的交替类实现系统现已移动到 sqlalchemy.ext.instrumentation。这是一个很少使用的系统，它增加了类仪器化的复杂性和开销。新的架构允许它保持未使用状态，直到实际导入 InstrumentationManager 时，此时它将被引导到核心中。

### 示例

+   **[示例]**

    Beaker 缓存示例已转换为使用 [dogpile.cache](https://dogpilecache.readthedocs.io/)。这是一个由 Beaker 缓存内部的相同创建者编写的新缓存库，代表了一个大大改进、简化和现代化的缓存系统。

    另请参阅

    狗窝缓存

    参考：[#2589](https://www.sqlalchemy.org/trac/ticket/2589)

### 引擎

+   **[引擎] [功能]**

    现在，连接事件侦听器可以与单个 Connection 对象关联，而不仅仅是 Engine 对象。

    参考：[#2511](https://www.sqlalchemy.org/trac/ticket/2511)

+   **[引擎] [功能]**

    before_cursor_execute 事件触发所谓的“_cursor_execute”事件，这些事件通常是特殊情况下执行的主键绑定序列和不使用 RETURNING 时调用的默认生成 SQL 短语。

    参考：[#2459](https://www.sqlalchemy.org/trac/ticket/2459)

+   **[引擎] [功能]**

    测试套件使用的库已经稍作调整，使其再次成为 SQLAlchemy 的一部分。此外，新的测试套件现在在新的 sqlalchemy.testing.suite 包中。这是一个正在开发中的系统，希望为外部方言提供一个通用的测试套件。在 SQLAlchemy 外部维护的方言可以使用新的测试装置作为其自己测试的框架，并且将免费获得一个“兼容性”方言测试套件，其中包括一个改进的“要求”系统，其中可以为测试启用或禁用特定功能和特性。

+   **[引擎] [功能]**

    添加了一种新的在进程中注册新方言的系统，而无需使用入口点。请参阅“注册新方言”文档。

    参考：[#2462](https://www.sqlalchemy.org/trac/ticket/2462)

+   **[engine] [feature]**

    如果在 bindparam()中未显式传递“value”或“callable”参数，则“required”标志默认设置为 True。这将导致语句执行检查参数是否存在于最终绑定参数集合中，而不是隐式分配为 None。

    参考：[#2556](https://www.sqlalchemy.org/trac/ticket/2556)

+   **[engine] [feature]**

    对“dialect”API 进行了各种 API 调整，以更好地支持高度专业化的系统，如 Akiban 数据库，包括更多的钩子以允许执行上下文访问类型处理器。

+   **[engine] [feature]**

    Inspector.get_primary_keys()已被弃用；请使用 Inspector.get_pk_constraint()。感谢 Diana Clarke。

    参考：[#2422](https://www.sqlalchemy.org/trac/ticket/2422)

+   **[engine] [feature]**

    添加了一个名为“utils”的新 C 扩展模块，用于在有时间实现时提供额外的函数加速。

+   **[engine] [bug]**

    Inspector.get_table_names()的 order_by=”foreign_key”功能现在按照依赖关系表先排序，以与 util.sort_tables 和 metadata.sorted_tables 保持一致。

+   **[engine] [bug]**

    修复了一个 bug，即如果数据库重新启动影响了多个连接，每个连接都会单独调用池的新处理，尽管只需要一个处理。

    参考：[#2522](https://www.sqlalchemy.org/trac/ticket/2522)

+   **[engine] [bug]**

    select().apply_labels()的.c.属性上列的名称现在基于<tablename>_<colkey>而不是<tablename>_<colname>，对于那些具有明确定义的.key 的列。

    参考：[#2397](https://www.sqlalchemy.org/trac/ticket/2397)

+   **[engine] [bug]**

    当 Table 上的 autoload_replace 标志为 False 时，将跳过任何反射的外键约束，这些约束引用已声明的列，假设在 Python 中声明的列将接管指定在 Python 中的 ForeignKey 或 ForeignKeyConstraint 声明的任务。

+   **[engine] [bug]**

    ResultProxy 方法 inserted_primary_key、last_updated_params()、last_inserted_params()、postfetch_cols()、prefetch_cols()都断言给定的语句是一个已编译的构造，并且是一个适当的 insert()或 update()语句，否则会引发 InvalidRequestError。

    参考：[#2498](https://www.sqlalchemy.org/trac/ticket/2498)

+   **[engine]**

    ResultProxy.last_inserted_ids 已被移除，替换为 inserted_primary_key。

### sql

+   **[sql] [feature]**

    向`Engine`添加了一个新方法`Engine.execution_options()`。此方法的工作方式与`Connection.execution_options()`类似，它创建一个指向新选项集的父对象的副本。该方法可用于构建每个引擎共享相同底层连接池的分片方案。该方法已针对 ORM 中的水平分片配方进行了测试。

    另见

    `Engine.execution_options()`

+   **[sql] [feature]**

    核心操作系统的重大改进，允许在类型级别重新定义现有操作符以及添加新操作符。可以从现有类型创建新类型，这些新类型添加或重新定义了导出到列表达式的操作，类似于 ORM 允许的比较器工厂。新的架构将此功能移至核心，以便在所有情况下都可以一致使用，并且使用现有的类型传播行为进行清晰传播。

    参考：[#2547](https://www.sqlalchemy.org/trac/ticket/2547)

+   **[sql] [feature]**

    为了补充，类型现在可以提供“绑定表达式”和“列表达式”，允许在每列或每绑定级别的语句中进行 SQL 表达式的编译时注入。这适用于类型需要在 SQL 级别而不是在 Python 级别增强绑定和结果行为的用例。允许透明加密/解密、使用 PostGIS 函数等方案。

    参考：[#1534](https://www.sqlalchemy.org/trac/ticket/1534), [#2547](https://www.sqlalchemy.org/trac/ticket/2547)

+   **[sql] [feature]**

    核心操作系统现在包括 getitem 操作符，即 Python 中的括号操作符。首先，这用于为 PostgreSQL ARRAY 类型提供索引和切片行为，并且还提供了一个钩子，用于终端用户定义自定义 __getitem__ 方案，这些方案可以应用于类型级别以及 ORM 级别的自定义操作符方案。还支持 lshift（<<）和 rshift（>>）作为可选操作符。

    请注意，此更改的效果是，ORM 与 synonym()或其他“描述符包装”方案一起使用的基于描述符的 __getitem__ 方案将需要开始使用自定义比较器以维护此行为。

+   **[sql] [feature]**

    修订了用于确定用户定义运算符的运算符优先级的规则，即使用 `op()` 方法授予的运算符。以前，在所有情况下都应用最小的优先级，现在默认优先级为零，低于所有运算符，除了“逗号”（例如，在 `func` 调用的参数列表中使用）和“AS”，并且还可以通过 `op()` 方法上的“precedence”参数进行自定义。

    参考：[#2537](https://www.sqlalchemy.org/trac/ticket/2537)

+   **[sql] [特性]**

    对所有 String 类型添加了“collation”参数。当存在时，呈现为 COLLATE <collation>。这是为了支持现在多个数据库（包括 MySQL、SQLite 和 PostgreSQL）支持的 COLLATE 关键字。

    参考：[#2276](https://www.sqlalchemy.org/trac/ticket/2276)

+   **[sql] [特性]**

    现在可以通过将 operators.custom_op() 与 UnaryExpression() 结合使用来使用自定义一元运算符。

+   **[sql] [特性]**

    增强了 GenericFunction 和 func.*，允许通过类名自动在 func.* 命名空间中使用用户定义的 GenericFunction 子类，可选择使用包名，以及具有在 func.* 中的标识名不同于渲染名的功能。

+   **[sql] [特性]**

    现在，cast() 和 extract() 结构也将通过 func.* 访问器生成，因为用户自然会尝试从 func.* 访问这些名称，即使返回的对象不是 FunctionElement，也应该做出预期的操作。

    参考：[#2562](https://www.sqlalchemy.org/trac/ticket/2562)

+   **[sql] [特性]**

    现在可以使用新的 inspect() 服务获取 Inspector 对象的实例。

    参考：[#2208](https://www.sqlalchemy.org/trac/ticket/2208)

+   **[sql] [特性]**

    现在 column_reflect 事件接受 Inspector 对象作为第一个参数，位于“table”之前。使用这个非常新的事件的 0.7 版���的代码将需要修改以添加“inspector”对象作为第一个参数。

    参考：[#2418](https://www.sqlalchemy.org/trac/ticket/2418)

+   **[sql] [特性]**

    现在结果集中列的定位行为默认区分大小写。多年来，SQLAlchemy 会对这些值进行不区分大小写的转换，可能是为了缓解像 Oracle 和 Firebird 这样的方言早期大小写敏感性问题。这些问题在更现代的版本中已经更清晰地解决，因此在标识符上调用 lower() 的性能损失已被移除。可以通过在 create_engine() 上设置“case_insensitive=False”来重新启用不区分大小写的比较。

    参考：[#2423](https://www.sqlalchemy.org/trac/ticket/2423)

+   **[sql] [特性]**

    当在 insert.values() 或 update.values() 中存在不在目标表中的键时，现在“未使用的列名”警告已更改为异常。

    参考：[#2415](https://www.sqlalchemy.org/trac/ticket/2415)

+   **[sql] [特性]**

    为 ForeignKey、ForeignKeyConstraint 添加了“MATCH”子句，感谢 Ryan Kelly。

    参考：[#2502](https://www.sqlalchemy.org/trac/ticket/2502)

+   **[sql] [feature]**

    添加了对从表的别名进行的 DELETE 和 UPDATE 的支持，这些表可能在查询中的其他地方与自身关联，由 Ryan Kelly 提供。

    参考：[#2507](https://www.sqlalchemy.org/trac/ticket/2507)

+   **[sql] [feature]**

    select() 现在具有 correlate_except() 方法，自动关联除传递的其他所有可选择项。

+   **[sql] [feature]**

    prefix_with() 方法现在可在每个 select()、insert()、update()、delete() 上使用，具有相同的 API，接受多个前缀调用，以及“方言名称”，以便将前缀限制为一种方言。

    参考：[#2431](https://www.sqlalchemy.org/trac/ticket/2431)

+   **[sql] [feature]**

    将 reduce_columns() 方法添加到 select() 构造中，使用 util.reduce_columns 实用函数内联替换列以删除等效列。reduce_columns() 还添加了“with_only_synonyms”，以限制只减少具有相同名称的列。移除了已弃用的 fold_equivalents() 功能。

    参考：[#1729](https://www.sqlalchemy.org/trac/ticket/1729)

+   **[sql] [feature]**

    重新设计了 startswith()、endswith()、contains() 运算符，以更好地处理否定（NOT LIKE），并在编译时将它们组装起来，以便其生成的 SQL 可以被修改，比如在 Firebird STARTING WITH 的情况下。

    参考：[#2470](https://www.sqlalchemy.org/trac/ticket/2470)

+   **[sql] [feature]**

    向渲染 CREATE TABLE 的系统添加了一个钩子，通过针对新的 schema.CreateColumn 构造一个 @compiles 函数，为每个列提供访问渲染的功能。

    参考：[#2463](https://www.sqlalchemy.org/trac/ticket/2463)

+   **[sql] [feature]**

    “标量”选择现在具有 WHERE 方法，以帮助进行生成性构建。此外，在 SS “关联”列的方法上进行了轻微调整；新方法不再将含义应用于所选择的基础表列。这改进了一些相当微妙的情况，而且原有的逻辑似乎没有任何目的。

+   **[sql] [feature]**

    当首次使用构造为引用多个远程表的 ForeignKeyConstraint() 时，将会明确引发错误。

    参考：[#2455](https://www.sqlalchemy.org/trac/ticket/2455)

+   **[sql] [feature]**

    将 `ColumnOperators.notin_()`、`ColumnOperators.notlike()`、`ColumnOperators.notilike()` 添加到 `ColumnOperators` 中。

    参考：[#2580](https://www.sqlalchemy.org/trac/ticket/2580)

+   **[sql] [change]**

    Text() 类型呈现给定的长度，如果指定了长度。

+   **[sql] [changed]**

    expression.sql 中的大多数类不再以下划线开头，即 Label、SelectBase、Generative、CompareMixin。_BindParamClause 也被重命名为 BindParameter。这些类的旧下划线名称将在可预见的未来保持可用。

+   **[sql] [bug]**

    修复了将关键字参数传递给 `Compiler.process()` 时，不会将这些参数传播到 SELECT 语句的 columns 子句中的列表达式的 bug。特别是在使用依赖于特殊标志的自定义编译方案时，这会出现问题。

    参考：[#2593](https://www.sqlalchemy.org/trac/ticket/2593)

+   **[sql] [bug] [orm]**

    `select()` 的自相关特性，以及由此导致的 `Query` 的自相关特性，对于直接在封闭的 SELECT 的 FROM 列表中呈现的 SELECT 语句不会生效。在 SQL 中，自相关仅适用于诸如 WHERE、ORDER BY、columns 子句中的列表达式。

    参考：[#2595](https://www.sqlalchemy.org/trac/ticket/2595)

+   **[sql] [bug]**

    对列优先级进行了微调，将 “concat” 和 “match” 操作符移到了与 “is”、“like” 等操作符相同的位置；这有助于在与 “IS” 结合使用时进行括号渲染。

    参考：[#2564](https://www.sqlalchemy.org/trac/ticket/2564)

+   **[sql] [bug]**

    对使用标签将列表达式应用于选择语句的操作进行了调整，无论是否有其他修改构造，都不再将该表达式“定位”到底层 Column；这会影响依赖于 Column 定位以检索结果的 ORM 操作。也就是说，像 query(User.id, User.id.label('foo')) 这样的查询现在将分别跟踪每个 “User.id” 表达式的值，而不是将它们混合在一起。预计不会影响任何用户；但是，如果使用 select() 结合 query.from_statement() 并尝试加载完全组合的 ORM 实体，则可能会出现问题，因为 select() 将不再将带有任意 .label() 名称的 Column 对象定位到该实体映射的 Column 对象。

    参考：[#2591](https://www.sqlalchemy.org/trac/ticket/2591)

+   **[sql] [bug]**

    修复了将 Column 的 “default” 参数解释为可调用对象时不将 ExecutionContext 传递给关键字参数的问题。

    参考：[#2520](https://www.sqlalchemy.org/trac/ticket/2520)

+   **[sql] [bug]**

    所有的 UniqueConstraint、ForeignKeyConstraint、CheckConstraint 和 PrimaryKeyConstraint 在直接引用绑定到表的 Column 对象（即不仅仅是字符串列名）并且只引用一个表时，都会自动附加到它们的父表上。在 0.8 之前，这种行为发生在 UniqueConstraint 和 PrimaryKeyConstraint 上，但不是 ForeignKeyConstraint 或 CheckConstraint。

    参考：[#2410](https://www.sqlalchemy.org/trac/ticket/2410)

+   **[sql] [错误]**

    TypeDecorator 现在默认包含一个基于 “impl” 类型的通用 repr()。对于那些指定了自定义 __init__ 方法的 TypeDecorator 类来说，这是一个行为变化；如果这些类型需要 __repr__() 提供忠实的构造函数表示，则需要重新定义 __repr__()。

    参考：[#2594](https://www.sqlalchemy.org/trac/ticket/2594)

+   **[sql] [错误]**

    `column.label(None)` 现在生成一个匿名标签，而不是返回列对象本身，与 `label(column, None)` 的行为一致。

    参考：[#2168](https://www.sqlalchemy.org/trac/ticket/2168)

+   **[sql] [已移除]**

    在 `create_engine()` 以及 `String` 上的长期弃用且无效的 `assert_unicode` 标志已被移除。

### postgresql

+   **[postgresql] [功能]**

    postgresql.ARRAY 现在具有可选的 “dimension” 参数，将为数组分配特定数量的维度，这将在 DDL 中呈现为 ARRAY[][]…，还改善了绑定/结果处理的性能。

    参考：[#2441](https://www.sqlalchemy.org/trac/ticket/2441)

+   **[postgresql] [功能]**

    postgresql.ARRAY 现在支持索引和切片。Python 的 [] 操作符在所有类型为 ARRAY 的 SQL 表达式上都可用；可以传递整数或简单切片。这些切片也可以在 UPDATE 语句的 SET 子句中的赋值方面使用，方法是将它们传递给 Update.values()；有关示例，请参阅文档。

+   **[postgresql] [功能]**

    添加了新的 “数组字面量” 构造 postgresql.array()。基本上是一个呈现为 ARRAY[1,2,3] 的 “元组”。

+   **[postgresql] [功能]**

    添加了对 PostgreSQL ONLY 关键字的支持，该关键字可以在 SELECT、UPDATE 或 DELETE 语句中对应一个表出现。该短语是使用 with_hint() 建立的。感谢 Ryan Kelly

    参考：[#2506](https://www.sqlalchemy.org/trac/ticket/2506)

+   **[postgresql] [功能]**

    PostgreSQL 方言的 “ischema_names” 字典是 “非官方” 可自定义的。这意味着，可以将新类型（例如 PostGIS 类型）添加到该字典中，并且 PG 类型反射代码应该能够处理具有可变数量参数的简单类型。这里的功能性是 “非官方的” ，有三个原因：

    1.  这不是一个 “官方” API。理想情况下，一个 “官方” API 应该允许在方言或全局级别以一种通用方式添加自定义类型处理可调用对象。

    1.  这仅针对 PG 方言实现，特别是因为 PG 对自定义类型有广泛支持，与其他数据库后端不同。真正的 API 将在默认方言级别实现。

    1.  此处的反射代码仅针对简单类型进行了测试，可能在更复杂类型上存在问题。

    补丁由Éric Lemoine 提供。

### mysql

+   **[mysql] [feature]**

    将 TIME 类型添加到 mysql 方言中，接受“fst”参数，这是最近 MySQL 版本的新“分数秒”指定符。数据类型将解释从驱动程序接收的微秒部分，但请注意，目前大多数/所有 MySQL DBAPI 不支持返回此值。

    参考：[#2534](https://www.sqlalchemy.org/trac/ticket/2534)

+   **[mysql] [bug]**

    方言在首次连接时不再发出昂贵的服务器排序查询，以及服务器大小写查询。这些功能仍然作为半私有功能可用。

    参考：[#2404](https://www.sqlalchemy.org/trac/ticket/2404)

### sqlite

+   **[sqlite] [feature]**

    SQLite 的日期和时间类型已经进行了全面改进，以支持更开放的输入和输出格式，使用基于名称的格式字符串和正则表达式。新参数“microseconds”还提供了省略时间戳中的“微秒”部分的选项。感谢 Nathan Wright 在此方面的工作和测试。

    参考：[#2363](https://www.sqlalchemy.org/trac/ticket/2363)

+   **[sqlite]**

    将`NCHAR`、`NVARCHAR`添加到 SQLite 方言的已识别类型名称列表中以供反射使用。SQLite 返回给定类型的名称作为返回的名称。

    参考：[rc3addcc9ffad](https://www.sqlalchemy.org/trac/changeset/c3addcc9ffad)

### mssql

+   **[mssql] [feature]**

    SQL Server 方言可以给出数据库限定的模式名称，即“schema='mydatabase.dbo'”；反射操作将检测到这一点，将模式分割在“.”之间以单独获取所有者，并在反射目标内部发出“USE mydatabase”语句；然后恢复从 DB_NAME()返回的现有数据库。

+   **[mssql] [feature]**

    更新了对 mxodbc 驱动程序的支持；建议使用 mxodbc 3.2.1 以获得完全兼容性。

+   **[mssql] [bug]**

    移除了旧版本行为，即通过==与标量 SELECT 进行列比较会强制转换为 SQL 服务器方言的 IN。这是隐式行为，在其他情况下会失败，因此被移除。依赖此行为的代码需要修改为显式使用 column.in_(select)。

    参考：[#2277](https://www.sqlalchemy.org/trac/ticket/2277)

### oracle

+   **[oracle] [feature]**

    可以通过将排除的字符串 DBAPI 类型名称列表发送到 exclude_setinputsizes 方���参数来自定义不包括在 setinputsizes()集中的列的类型。此列表以前是固定的。该列表现在默认为 STRING、UNICODE，移除了 CLOB、NCLOB。 

    参考：[#2561](https://www.sqlalchemy.org/trac/ticket/2561)

+   **[oracle] [bug]**

    当生成同名绑定参数到 bindparam()对象时，现在会从具有 quote=True 的 Column 中传递引用信息，就像在生成的 INSERT 和 UPDATE 语句中一样，以便完全支持未知的保留名称。

    参考：[#2437](https://www.sqlalchemy.org/trac/ticket/2437)

+   **[oracle] [bug]**

    在 Oracle 中，CreateIndex 构造现在将索引的名称模式限定为父表的名称。以前，此名称被省略，显然会在默认模式中创建索引，而不是表的模式。

### 杂项

+   **[feature] [access]**

    MS Access 方言已移至 Bitbucket 上的自己的项目，利用了新的 SQLAlchemy 方言兼容性套件。该方言仍然处于非常初步的阶段，可能还没有准备好供一般使用，但现在具有*极其*基本的功能。[`bitbucket.org/zzzeek/sqlalchemy-access`](https://bitbucket.org/zzzeek/sqlalchemy-access)

+   **[feature] [firebird]**

    “startswith()”运算符渲染为“STARTING WITH”，“~startswith()”渲染为“NOT STARTING WITH”，使用 FB 更有效的运算符。

    参考：[#2470](https://www.sqlalchemy.org/trac/ticket/2470)

+   **[feature] [firebird]**

    添加了一个用于 fdb 驱动程序的实验性方言，但由于无法构建 fdb 软件包，因此未经测试。

    参考：[#2504](https://www.sqlalchemy.org/trac/ticket/2504)

+   **[bug] [firebird]**

    当尝试发出没有长度的 VARCHAR 时，会引发 CompileError，与 MySQL 的方式相同。

    参考：[#2505](https://www.sqlalchemy.org/trac/ticket/2505)

+   **[bug] [firebird]**

    Firebird 现在使用严格的“ansi 绑定规则”，以便绑定参数不会在语句的列子句中呈现-它们会直接呈现。

+   **[bug] [firebird]**

    在使用 DateTime 类型与 Firebird 时，支持将 datetime 作为 date 传递；其他方言也支持此功能。

+   **[moved] [maxdb]**

    MaxDB 方言已经多年没有功能了，现在移至一个待定的 bitbucket 项目中，[`bitbucket.org/zzzeek/sqlalchemy-maxdb`](https://bitbucket.org/zzzeek/sqlalchemy-maxdb)。

## 0.8.7

发布日期：2014 年 7 月 22 日

### orm

+   **[orm] [bug]**

    修复了子查询急加载中的错误，其中在多态子类边界上的长链急加载与多态加载一起会无法找到子类链接，导致在`AliasedClass`上出现缺少属性名称的错误。

    参考：[#3055](https://www.sqlalchemy.org/trac/ticket/3055)

+   **[orm] [bug]**

    修复了 ORM bug，其中 `class_mapper()` 函数会掩盖应该在 mapper 配置期间由于用户错误而引发的 AttributeErrors 或 KeyErrors。对于属性/键错误的捕获已更具体，以不包括配置步骤。

    参考：[#3047](https://www.sqlalchemy.org/trac/ticket/3047)

### sql

+   **[sql] [bug]**

    修复了在 `Enum` 和其他 `SchemaType` 子类中直接将类型与 `MetaData` 关联时会导致在发出事件（如创建事件）时挂起的 bug。

    参考：[#3124](https://www.sqlalchemy.org/trac/ticket/3124)

+   **[sql] [bug]**

    修复了自定义操作符加法 `TypeEngine.with_variant()` 系统中的一个错误，当与变体一起使用 `TypeDecorator` 时，使用比较运算符会导致 MRO 错误。

    参考：[#3102](https://www.sqlalchemy.org/trac/ticket/3102)

+   **[sql] [bug]**

    修复了在 INSERT..FROM SELECT 结构中的一个 bug，其中从 UNION 中选择会将 union 包装在一个匿名的子查询中。

    参考：[#3044](https://www.sqlalchemy.org/trac/ticket/3044)

+   **[sql] [bug]**

    修复了当应用空的 `and_()` 或 `or_()` 或其他空白表达式时，`Table.update()` 和 `Table.delete()` 会产生空的 WHERE 子句的 bug。现在这与 `select()` 的行为一致了。

    参考：[#3045](https://www.sqlalchemy.org/trac/ticket/3045)

### postgresql

+   **[postgresql] [bug]**

    将 `hashable=False` 标志添加到 PG `HSTORE` 类型中，这是为了允许 ORM 在请求混合列/实体列表中的 ORM 映射的 HSTORE 列时跳过尝试“散列”ORM 映射的 HSTORE 列所需的。修补程序由 Gunnlaugur Þór Briem 提供。

    参考：[#3053](https://www.sqlalchemy.org/trac/ticket/3053)

+   **[postgresql] [bug]**

    添加了一个新的“断开连接”消息“连接已意外关闭”。这似乎与较新版本的 SSL 有关。感谢 Antti Haapala 提供的拉取请求。

### mysql

+   **[mysql] [bug]**

    MySQL 错误 2014“命令不同步”似乎是在现代 MySQL-Python 版本中作为 ProgrammingError 而不是 OperationalError 引发的；所有经过测试的 MySQL 错误代码“is disconnect”现在都在 OperationalError 和 ProgrammingError 中进行检查。

    参考：[#3101](https://www.sqlalchemy.org/trac/ticket/3101)

+   **[mysql] [bug]**

    修复了一个 bug，即在索引的`mysql_length`参数中添加的列名需要具有相同的引号才能被识别。修复使引号变为可选，但也为那些使用解决方法的人提供了旧的行为以实现向后兼容。

    参考：[#3085](https://www.sqlalchemy.org/trac/ticket/3085)

+   **[mysql] [bug]**

    添加了对包含 KEY_BLOCK_SIZE 的索引使用等号的表进行反射的支持。感谢 Sean McGivern 提供的拉取请求。

### mssql

+   **[mssql] [bug]**

    将语句编码添加到“SET IDENTITY_INSERT”语句中，当在 IDENTITY 列中插入显式 INSERT 时进行操作，以支持在不支持 unicode 语句的驱动程序（如 pyodbc + unix + py2k）上使用非 ascii 表标识符。

+   **[mssql] [bug]**

    在 SQL Server pyodbc 方言中，修复了`description_encoding`方言参数的实现，当未明确设置时，会导致在包含其他编码名称的结果集中无法正确解析 cursor.description。今后不应该需要此参数。

    参考：[#3091](https://www.sqlalchemy.org/trac/ticket/3091)

### misc

+   **[bug] [declarative]**

    当访问`__mapper_args__`字典时，它是从声明性 mixin 或抽象类中复制的，以便声明性本身对该字典所做的修改不会与其他映射发生冲突。该字典在`version_id_col`和`polymorphic_on`参数方面进行修改，用本地类/表正式映射到的列替换其中的列。

    参考：[#3062](https://www.sqlalchemy.org/trac/ticket/3062)

+   **[bug] [ext]**

    修复了可变扩展中的 bug，即`MutableDict`未为`setdefault()`字典操作报告更改事件。

    参考：[#3051](https://www.sqlalchemy.org/trac/ticket/3051), [#3093](https://www.sqlalchemy.org/trac/ticket/3093)

+   **[bug] [ext]**

    修复了`MutableDict.setdefault()`未返回现有值或新值的 bug（此 bug 未在任何 0.8 版本中发布）。感谢 Thomas Hervé提供的拉取请求。

    参考：[#3051](https://www.sqlalchemy.org/trac/ticket/3051), [#3093](https://www.sqlalchemy.org/trac/ticket/3093)

### orm

+   **[orm] [bug]**

    修复了子查询贪婪加载中的错误，在多态子类边界上长链贪婪加载与多态加载结合使用时，会无法找到链中的子类链接，会在一个`AliasedClass`上出错，缺少属性名称。

    参考：[#3055](https://www.sqlalchemy.org/trac/ticket/3055)

+   **[orm] [bug]**

    修复了 ORM 中的错误，`class_mapper()`函数会掩盖应该在映射器配置期间由于用户错误而引发的 AttributeErrors 或 KeyErrors。对于属性/键错误的捕获已经更具体，不包括配置步骤。

    参考：[#3047](https://www.sqlalchemy.org/trac/ticket/3047)

### sql

+   **[sql] [bug]**

    修复了`Enum`和其他`SchemaType`子类中的错误，在直接将类型与`MetaData`关联时，当在`MetaData`上触发事件（如创建事件）时会导致 hang。

    参考：[#3124](https://www.sqlalchemy.org/trac/ticket/3124)

+   **[sql] [bug]**

    修复了自定义操作符加上`TypeEngine.with_variant()`系统中的错误，其中使用`TypeDecorator`与 variant 一起时，在使用比较运算符时会出现 MRO 错误。

    参考：[#3102](https://www.sqlalchemy.org/trac/ticket/3102)

+   **[sql] [bug]**

    修复了在 INSERT..FROM SELECT 结构中的错误，其中从 UNION 中选择会将联合包装在一个匿名（例如未标记）子查询中。

    参考：[#3044](https://www.sqlalchemy.org/trac/ticket/3044)

+   **[sql] [bug]**

    修复了当应用空的`and_()`或`or_()`或其他空白表达式时，`Table.update()`和`Table.delete()`会生成空的 WHERE 子句的错误。现在，这与`select()`的行为一致。

    参考：[#3045](https://www.sqlalchemy.org/trac/ticket/3045)

### postgresql

+   **[postgresql] [bug]**

    向 PG `HSTORE`类型添加了`hashable=False`标志，这是为了允许 ORM 在请求混合列/实体列表中的 ORM 映射的 HSTORE 列时跳过尝试“哈希”它。补丁由 Gunnlaugur Þór Briem 提供。

    参考：[#3053](https://www.sqlalchemy.org/trac/ticket/3053)

+   **[postgresql] [bug]**

    添加了新的“断开连接”消息“连接意外关闭”。这似乎与较新版本的 SSL 有关。感谢 Antti Haapala 的拉取请求。

### mysql

+   **[mysql] [bug]**

    MySQL 错误 2014“commands out of sync”似乎在现代 MySQL-Python 版本中被提升为 ProgrammingError，而不是 OperationalError；现在所有被测试为“is disconnect”的 MySQL 错误代码都在 OperationalError 和 ProgrammingError 中进行检查。

    参考：[#3101](https://www.sqlalchemy.org/trac/ticket/3101)

+   **[mysql] [bug]**

    修复了在索引的`mysql_length`参数上添加列名时，需要对带引号的名称使用相同的引号才能被识别的错误。修复使引号变为可选，但也为那些使用此解决方法的人提供了旧的行为以实现向后兼容。

    参考：[#3085](https://www.sqlalchemy.org/trac/ticket/3085)

+   **[mysql] [bug]**

    添加了对包含 KEY_BLOCK_SIZE 的索引使用等号进行反射表的支持。感谢 Sean McGivern 的拉取请求。

### mssql

+   **[mssql] [bug]**

    在“SET IDENTITY_INSERT”语句中添加了语句编码，当在 IDENTITY 列中插入显式 INSERT 时，以支持在不支持 unicode 语句的驱动程序（如 pyodbc + unix + py2k）上操作非 ascii 表标识符。

+   **[mssql] [bug]**

    在 SQL Server pyodbc 方言中，修复了`description_encoding`方言参数的实现，当未显式设置时，会导致在包含以其他编码命名的名称的结果集中，无法正确解析 cursor.description。这个参数在未来不应该需要。

    参考：[#3091](https://www.sqlalchemy.org/trac/ticket/3091)

### 杂项

+   **[bug] [declarative]**

    当访问`__mapper_args__`字典时，该字典是从声明性 mixin 或抽象类中复制的，因此声明性本身对该字典所做的修改不会与其他映射冲突。该字典在`version_id_col`和`polymorphic_on`参数方面进行修改，用本地类/表正式映射到的列替换其中的列。

    参考：[#3062](https://www.sqlalchemy.org/trac/ticket/3062)

+   **[bug] [ext]**

    修复了可变扩展中的错误，即`MutableDict`未��告`setdefault()`字典操作的更改事件。

    参考：[#3051](https://www.sqlalchemy.org/trac/ticket/3051)，[#3093](https://www.sqlalchemy.org/trac/ticket/3093)

+   **[bug] [ext]**

    修复了`MutableDict.setdefault()`方法未返回现有值或新值的错误（此错误未在任何 0.8 版本中发布）。感谢 Thomas Hervé提供的拉取请求。

    参考：[#3051](https://www.sqlalchemy.org/trac/ticket/3051)，[#3093](https://www.sqlalchemy.org/trac/ticket/3093)

## 0.8.6

发布日期：2014 年 3 月 28 日

### 一般

+   **[general] [bug]**

    调整了`setup.py`文件，以支持可能将来从 setuptools 中删除`setuptools.Feature`扩展。如果不存在此关键字，设置仍将成功使用 setuptools 而不是退回到 distutils。现在还可以通过设置 DISABLE_SQLALCHEMY_CEXT 环境变量来禁用 C 扩展构建。无论 setuptools 是否可用，此变量都有效。

    参考：[#2986](https://www.sqlalchemy.org/trac/ticket/2986)

### orm

+   **[orm] [bug]**

    修复了 ORM 中的错误，即更改对象的主键，然后将其标记为 DELETE 会导致无法针对 DELETE 操作正确定位行的错误。

    参考：[#3006](https://www.sqlalchemy.org/trac/ticket/3006)

+   **[orm] [bug]**

    由于[#2818](https://www.sqlalchemy.org/trac/ticket/2818)导致 0.8.3 中的回归，`Query.exists()`在只有一个`Query.select_from()`条目但没有其他实体的查询上无法工作。

    参考：[#2995](https://www.sqlalchemy.org/trac/ticket/2995)

+   **[orm] [bug]**

    改进了一个错误消息，该错误消息会在对非可选择对象（例如`literal_column()`）进行查询（`query()`）后，尝试使用`Query.join()`时出现，使得“左”侧被确定为`None`然后失败。现在明确检测到这种情况。

+   **[orm] [bug]**

    从`sqlalchemy.orm.interfaces.__all__`中删除了过时的名称，并更新为当前名称，以便再次从该模块进行`import *`操作。

    参考：[#2975](https://www.sqlalchemy.org/trac/ticket/2975)

### sql

+   **[sql] [bug]**

    修复了 `tuple_()` 构造中的错误，其中基本上第一个 SQL 表达式的“类型”将被应用为比较元组值的“比较类型”；在某些情况下，这会导致不合适的“类型强制转换”发生，例如当一个元组具有字符串和二进制值的混合时，错误地将目标值强制转换为二进制，即使左侧的值并非如此。`tuple_()` 现在预期其值列表中有异构类型。

    参考：[#2977](https://www.sqlalchemy.org/trac/ticket/2977)

### postgresql

+   **[postgresql] [feature]**

    启用了 psycopg2 DBAPI 的“合理的多行计数”检查，因为从 psycopg2 2.0.9 开始似乎支持了这个功能。

+   **[postgresql] [bug]**

    由于版本 0.8.5 / 0.9.3 的兼容性增强引起的固定回归，再次导致仅针对 8.1、8.2 系列的 PostgreSQL 版本的索引反射再次中断，围绕着一直存在问题的 int2vector 类型。虽然 int2vector 支持从 8.1 开始的数组操作，但显然，从 8.3 开始只支持 CAST 到 varchar。

    参考：[#3000](https://www.sqlalchemy.org/trac/ticket/3000)

### 杂项

+   **[bug] [ext]**

    修复了可变扩展和 `flag_modified()` 中的错误，如果属性已重新分配给自身，则更改事件将不会传播。

    参考：[#2997](https://www.sqlalchemy.org/trac/ticket/2997)

### 一般

+   **[general] [bug]**

    调整了 `setup.py` 文件，以支持可能将来从 setuptools 中移除 `setuptools.Feature` 扩展。如果没有这个关键字，设置仍然会成功使用 setuptools 而不是回退到 distutils。现在也可以通过设置 DISABLE_SQLALCHEMY_CEXT 环境变量来禁用 C 扩展构建。此变量无论 setuptools 是否可用都有效。

    参考：[#2986](https://www.sqlalchemy.org/trac/ticket/2986)

### orm

+   **[orm] [bug]**

    修复了 ORM 中的错误，在更改对象的主键后，将其标记为 DELETE 将无法针对正确的行进行 DELETE。

    参考：[#3006](https://www.sqlalchemy.org/trac/ticket/3006)

+   **[orm] [bug]**

    从 0.8.3 的回归中修复了问题，这是由于 [#2818](https://www.sqlalchemy.org/trac/ticket/2818) 导致的，`Query.exists()` 在只有一个 `Query.select_from()` 条目但没有其他实体的查询上无法工作。

    参考：[#2995](https://www.sqlalchemy.org/trac/ticket/2995)

+   **[orm] [bug]**

    改进了一个错误消息，该消息会在对非可选择对象（例如`literal_column()`）进行查询（`query()`）后，尝试使用`Query.join()`时出现，导致“左”侧被确定为`None`然后失败。现在明确检测到这种情况。

+   **[orm] [bug]**

    从`sqlalchemy.orm.interfaces.__all__`中删除了过时的名称，并使用当前名称进行刷新，以便再次从该模块进行`import *`操作。

    参考：[#2975](https://www.sqlalchemy.org/trac/ticket/2975)

### sql

+   **[sql] [bug]**

    修复了`tuple_()`构造中的 bug，其中基本上第一个 SQL 表达式的“类型”将被应用为与比较的元组值的“比较类型”；在某些情况下，这会导致不适当的“类型强制转换”发生，例如当一个元组具有混合的 String 和 Binary 值时，错误��将目标值强制转换为 Binary，即使左侧并不是这样。`tuple_()`现在期望其值列表中存在异构类型。

    参考：[#2977](https://www.sqlalchemy.org/trac/ticket/2977)

### postgresql

+   **[postgresql] [feature]**

    为 psycopg2 DBAPI 启用了“合理的多行计数”检查，因为似乎从 psycopg2 2.0.9 开始支持这一功能。

+   **[postgresql] [bug]**

    修复了由版本 0.8.5 / 0.9.3 的兼容性增强引起的回归，其中针对仅适用于 8.1、8.2 系列的 PostgreSQL 版本的索引反射再次中断，围绕着一直存在问题的 int2vector 类型。虽然 int2vector 从 8.1 开始支持数组操作，但显然只有从 8.3 开始才支持将其转换为 varchar。

    参考：[#3000](https://www.sqlalchemy.org/trac/ticket/3000)

### 杂项

+   **[bug] [ext]**

    修复了可变扩展中的错误以及`flag_modified()`中的 bug，如果属性已重新分配给自身，则更改事件将不会传播。

    参考：[#2997](https://www.sqlalchemy.org/trac/ticket/2997)

## 0.8.5

发布日期：2014 年 2 月 19 日

### orm

+   **[orm] [bug]**

    修复了`Query.get()`中的 bug，当在具有现有条件的查询上调用时，给定的标识已经存在于标识映射中时，它将无法一致地引发`InvalidRequestError`。

    参考：[#2951](https://www.sqlalchemy.org/trac/ticket/2951)

+   **[orm] [bug]**

    修复了当向`class_mapper()`或类似函数传递迭代器对象时，错误消息无法正确呈现的错误。Kyle Stark 提供的 Pullreq。

+   **[orm] [bug]**

    对`subqueryload()`策略的调整，确保查询在加载过程开始后运行；这样子查询加载就优先于其他可能在错误的时间由于其他急切/不加载情况而命中同一属性的加载器。

    引用：[#2887](https://www.sqlalchemy.org/trac/ticket/2887)

+   **[orm] [bug]**

    修复了当从基表继承到一个选择/别名的联接表继承时，PK 列也不具有相同名称时的错误，此时持久化系统会在插入时失败，无法将主键值从基表复制到继承表。

    引用：[#2885](https://www.sqlalchemy.org/trac/ticket/2885)

+   **[orm] [bug]**

    当传递的列/属性（名称）不能解析为列或映射属性（例如错误的元组）时，`composite()`将引发一个信息性错误消息；以前会引发一个未绑定的本地错误。

    引用：[#2889](https://www.sqlalchemy.org/trac/ticket/2889)

### 引擎

+   **[engine] [bug] [pool]**

    修复了由[#2880](https://www.sqlalchemy.org/trac/ticket/2880)引起的关键性回归，其中新的并发能力从池中返回连接意味着“first_connect”事件现在也不再同步，因此在即使是最小并发情况下也会导致方言配置错误。

    引用：[#2880](https://www.sqlalchemy.org/trac/ticket/2880), [#2964](https://www.sqlalchemy.org/trac/ticket/2964)

### SQL

+   **[sql] [bug]**

    修复了使用空列表或元组调用`Insert.values()`会引发 IndexError 的错误。现在它会产生一个空的插入构造，就像空字典的情况一样。

    引用：[#2944](https://www.sqlalchemy.org/trac/ticket/2944)

+   **[sql] [bug]**

    修复了当`ColumnOperators.in_()`被错误地传递包含`__getitem__()`方法的列表达式时进入无限循环的错误，例如使用`ARRAY`类型的列。

    引用：[#2957](https://www.sqlalchemy.org/trac/ticket/2957)

+   **[sql] [bug]**

    修复了一个问题，即具有 Sequence 的主键列，但该列不是“自动增量”列，要么因为有外键约束，要么设置了 `autoincrement=False`，在没有主键值的 INSERT 中尝试触发 Sequence，对于不支持序列的后端（如 SQLite、MySQL）会发生这种情况。

    参考：[#2896](https://www.sqlalchemy.org/trac/ticket/2896)

+   **[sql] [bug]**

    修复了 `Insert.from_select()` 方法的 bug，其中给定名称的顺序在生成 INSERT 语句时不会被考虑，因此与给定 SELECT 语句中的列名不匹配。还注意到 `Insert.from_select()` 暗示不能使用 Python 端的插入默认值，因为语句没有 VALUES 子句。

    参考：[#2895](https://www.sqlalchemy.org/trac/ticket/2895)

+   **[sql] [enhancement]**

    当编译语句中存在一个未设置值的 `BindParameter` 时，引发的异常现在在错误消息中包含绑定参数的键名。

### postgresql

+   **[postgresql] [bug]**

    对 psycopg2 断开连接检测添加了额外的消息，“无法向服务器发送数据”，这与现有的“无法从服务器接收数据”相辅相成，并已被用户观察到。

    参考：[#2936](https://www.sqlalchemy.org/trac/ticket/2936)

+   **[postgresql] [bug]**

    > 改进了对非常古老（8.1 之前）版本的 PostgreSQL 反射行为的支持，以及潜在的其他 PG 引擎，如 Redshift（假设 Redshift 报告版本为 < 8.1）。用于“索引”和“主键”的查询依赖于检查所谓的“int2vector”数据类型，该数据类型在 8.1 之前拒绝强制转换为数组，导致查询中使用的“ANY()”运算符失败。通过广泛的搜索，找到了非常巧妙但被 PG 核心开发人员推荐使用的查询，用于在使用 PG 版本 < 8.1 时，现在索引和主键约束反射可以在这些版本上工作。

+   **[postgresql] [bug]**

    修复了一个非常古老的问题，即 PostgreSQL 的“获取主键”反射查询已更新以考虑已重命名的主键约束；新的查询在 PostgreSQL 的非常古老版本（如版本 7）上失败，因此在检测到 server_version_info < (8, 0) 的情况下，在这些情况下恢复了旧查询。

    参考：[#2291](https://www.sqlalchemy.org/trac/ticket/2291)

### mysql

+   **[mysql] [feature]**

    添加了新的 MySQL 特定`DATETIME`，其中包括小数秒支持；还向`TIMESTAMP`添加了小数秒支持。DBAPI 支持有限，尽管 MySQL Connector/Python 已知支持小数秒。Patch 由 Geert JM Vanderkelen 提供。

    参考：[#2941](https://www.sqlalchemy.org/trac/ticket/2941)

+   **[mysql] [bug]**

    添加了对`PARTITION BY`和`PARTITIONS` MySQL 表关键字的支持，指定为`mysql_partition_by='value'`和`mysql_partitions='value'`以用于`Table`。拉取请求由 Marcus McCurdy 提供。

    参考：[#2966](https://www.sqlalchemy.org/trac/ticket/2966)

+   **[mysql] [bug]**

    修复了阻止基于 MySQLdb 的方言（例如 pymysql）在 Py3K 中工作的错误，其中对“connection charset”的检查会由于 Py3K 的更严格的值比较规则而失败。在任何情况下，该调用都没有考虑数据库版本，因为在那时服务器版本仍然为 None，因此该方法总体上已简化为依赖于 connection.character_set_name()。

    参考：[#2933](https://www.sqlalchemy.org/trac/ticket/2933)

+   **[mysql] [bug]**

    一些缺失的方法已添加到 cymysql 方言中，包括 _get_server_version_info()和 _detect_charset()。拉取请求由 Hajime Nakagami 提供。

### sqlite

+   **[sqlite] [bug]**

    恢复了在将唯一约束反射回 0.8 时遗漏的更改，其中包含 SQLite 列名称中的保留关键字将导致失败。拉取请求由 Roman Podolyaka 提供。

### mssql

+   **[mssql] [bug] [firebird]**

    与`Float`类型一起使用的“asdecimal”标志现在在 Firebird 和 mssql+pyodbc 方言中也可以工作；以前的十进制转换未发生。

+   **[mssql] [bug] [pymssql]**

    将“Net-Lib error during Connection reset by peer”消息添加到检查“pymssql”方言中的“disconnect”消息列表中。由 John Anderson 提供。

### 杂项

+   **[bug] [py3k]**

    修复了 Py3K 错误，其中缺少导入会导致在呈现绑定参数时无法导入“util.binary_type”的“literal binary”模式失败。0.9 处理方式不同。拉取请求由 Andreas Zeidler 提供。

+   **[bug] [firebird]**

    firebird 方言将引用以下划线开头的标识符。由 Treeve Jelbert 提供。

    参考：[#2897](https://www.sqlalchemy.org/trac/ticket/2897)

+   **[bug] [firebird]**

    修复了 Firebird 索引反射中的错误，其中索引中的列未正确排序；它们现在按照 RDB$FIELD_POSITION 的顺序排序。

+   **[bug] [declarative]**

    当将字符串参数发送给`relationship()`时，如果无法解析为类或映射器，则错误消息已更正为与接收非字符串参数时相同的方式，该方式指示了配置错误的关系名称。

    参考：[#2888](https://www.sqlalchemy.org/trac/ticket/2888)

### orm

+   **[orm] [bug]**

    修复了`Query.get()`在查询中存在条件时无法一致引发`InvalidRequestError`的 bug，��给定的标识已经存在于标识映射中时。

    参考：[#2951](https://www.sqlalchemy.org/trac/ticket/2951)

+   **[orm] [bug]**

    当将迭代器对象传递给`class_mapper()`或类似方法时，修复了错误消息无法在字符串格式化时呈现的问题。感谢 Kyle Stark 提交的 Pullreq。

+   **[orm] [bug]**

    对`subqueryload()`策略进行了调整，确保查询在加载过程开始后运行；这样，subqueryload 优先于其他加载器运行，这些加载器可能由于其他错误的时机导致了错误的贪婪加载。

    参考：[#2887](https://www.sqlalchemy.org/trac/ticket/2887)

+   **[orm] [bug]**

    修复了从表继承到基表的连接表继承时的 bug，其中主键列也不具有相同名称；持久性系统在插入时无法将主键值从基表复制到继承表中。

    参考：[#2885](https://www.sqlalchemy.org/trac/ticket/2885)

+   **[orm] [bug]**

    当传递的列/属性（名称）无法解析为列或映射属性（例如错误的元组）时，`composite()`将引发一个信息性错误消息；之前会引发未绑定的本地错误。

    参考：[#2889](https://www.sqlalchemy.org/trac/ticket/2889)

### 引擎

+   **[engine] [bug] [pool]**

    修复了由[#2880](https://www.sqlalchemy.org/trac/ticket/2880)引起的关键回归，其中新的并发能力从池中返回连接意味着“first_connect”事件现在也不再同步，从而导致在即使是最小并发情况下也会出现方言配置错误。

    参考：[#2880](https://www.sqlalchemy.org/trac/ticket/2880)，[#2964](https://www.sqlalchemy.org/trac/ticket/2964)

### sql

+   **[sql] [bug]**

    修复了调用带有空列表或元组的`Insert.values()`会引发 IndexError 的 bug。现在它会产生一个空的插入构造，就像使用空字典一样。

    参考：[#2944](https://www.sqlalchemy.org/trac/ticket/2944)

+   **[sql] [bug]**

    修复了`ColumnOperators.in_()`会进入无限循环的 bug，如果错误地传递了一个包含`__getitem__()`方法的列表达式的比较器，比如使用`ARRAY`类型的列。

    参考：[#2957](https://www.sqlalchemy.org/trac/ticket/2957)

+   **[sql] [bug]**

    修复了一个问题，即主键列上有一个 Sequence，但该列不是“自动增量”列，可能是因为它有外键约束或设置了`autoincrement=False`，在不支持序列的后端上，当出现一个缺少主键值的 INSERT 时，会尝试触发 Sequence。这将发生在像 SQLite、MySQL 这样的非序列后端上。

    参考：[#2896](https://www.sqlalchemy.org/trac/ticket/2896)

+   **[sql] [bug]**

    修复了`Insert.from_select()`方法的 bug，其中给定名称的顺序在生成 INSERT 语句时不会被考虑，因此与给定 SELECT 语句中的列名不匹配。还指出`Insert.from_select()`暗示不能使用 Python 端的插入默认值，因为该语句没有 VALUES 子句。

    参考：[#2895](https://www.sqlalchemy.org/trac/ticket/2895)

+   **[sql] [enhancement]**

    当编译语句中存在一个未赋值的`BindParameter`时，引发的异常现在在错误消息中包含绑定参数的键名。

### postgresql

+   **[postgresql] [bug]**

    添加了一个额外的消息到 psycopg2 的断开连接检测中，“无法发送数据到服务器”，这与现有的“无法从服务器接收数据”相辅相成，并已被用户观察到。

    参考：[#2936](https://www.sqlalchemy.org/trac/ticket/2936)

+   **[postgresql] [bug]**

    > 改进了对非常古老（8.1 之前）版本的 PostgreSQL 以及其他可能的 PG 引擎（如 Redshift，假设 Redshift 将版本报告为<8.1）的 PostgreSQL 反射行为的支持。关于“索引”和“主键”的查询依赖于检查所谓的“int2vector”数据类型，该数据类型在 8.1 之前拒绝强制转换为数组，导致查询中使用的“ANY()”运算符失败。通过广泛的搜索，找到了非常 hacky 但由 PG 核心开发人员推荐使用的查询，用于在使用 PG 版本<8.1 时使用，因此现在在这些版本上可以正常工作索引和主键约束反射。

+   **[postgresql] [错误]**

    修改了这个非常古老的问题，其中 PostgreSQL 的“获取主键”反射查询已更新以考虑已重命名的主键约束；新的查询在非常古老的 PostgreSQL 版本（如版本 7）上失败，因此在检测到 server_version_info < (8, 0)的情况下，在这些情况下恢复旧查询。

    参考：[#2291](https://www.sqlalchemy.org/trac/ticket/2291)

### mysql

+   **[mysql] [功能]**

    添加了新的 MySQL 特定的`DATETIME`，其中包括分数秒支持；还向`TIMESTAMP`添加了分数秒支持。DBAPI 支持有限，尽管 MySQL Connector/Python 已知支持分数秒。Patch 由 Geert JM Vanderkelen 提供。

    参考：[#2941](https://www.sqlalchemy.org/trac/ticket/2941)

+   **[mysql] [错误]**

    添加了对`PARTITION BY`和`PARTITIONS` MySQL 表关键字的支持，指定为`mysql_partition_by='value'`和`mysql_partitions='value'`到`Table`。感谢 Marcus McCurdy 的 Pull 请求。

    参考：[#2966](https://www.sqlalchemy.org/trac/ticket/2966)

+   **[mysql] [错误]**

    修复了阻止基于 MySQLdb 的方言（例如 pymysql）在 Py3K 中工作的错误，其中“连接字符集”检查将由于 Py3K 更严格的值比较规则而失败。在任何情况下，该调用都没有考虑数据库���本，因为服务器版本在那时仍然为 None，因此该方法已简化为依赖于 connection.character_set_name()。

    参考：[#2933](https://www.sqlalchemy.org/trac/ticket/2933)

+   **[mysql] [错误]**

    在 cymysql 方言中添加了一些缺失的方法，包括 _get_server_version_info()和 _detect_charset()。感谢 Hajime Nakagami 的 Pullreq。

### sqlite

+   **[sqlite] [错误]**

    恢复了在将唯一约束反射回 0.8 时遗漏的更改，其中在列名称中包含保留关键字的情况下，使用 SQLite 的`UniqueConstraint`将失败。感谢 Roman Podolyaka 的 Pull 请求。

### mssql

+   **[mssql] [bug] [firebird]**

    与`Float`类型一起使用的“asdecimal”标志现在也适用于 Firebird 以及 mssql+pyodbc 方言；以前未进行十进制转换。

+   **[mssql] [bug] [pymssql]**

    将“Net-Lib error during Connection reset by peer”消息添加到在 pymssql 方言中检查“disconnect”消息列表中。感谢 John Anderson。

### 杂项

+   **[bug] [py3k]**

    修复了 Py3K 中的错误，其中缺少导入将导致在呈现绑定参数时“literal binary”模式无法导入“util.binary_type”。0.9 处理方式不同。感谢 Andreas Zeidler 的拉取请求。

+   **[bug] [firebird]**

    火鸟方言将引用以下划线开头的标识符。感谢 Treeve Jelbert。

    参考：[#2897](https://www.sqlalchemy.org/trac/ticket/2897)

+   **[bug] [firebird]**

    修复了 Firebird 索引反射中列未正确排序的错误；现在它们按照 RDB$FIELD_POSITION 的顺序排序。

+   **[bug] [declarative]**

    当发送给`relationship()`的字符串参数无法解析为类或映射器时，错误消息已更正，与接收非字符串参数时的工作方式相同，指示配置错误的关系名称。

    参考：[#2888](https://www.sqlalchemy.org/trac/ticket/2888)

## 0.8.4

发布日期：2013 年 12 月 8 日

### orm

+   **[orm] [bug]**

    修复了由[#2818](https://www.sqlalchemy.org/trac/ticket/2818)引入的回归，生成的 EXISTS 查询会为具有两个同名列的语句产生“正在替换列”警告，因为内部 SELECT 没有设置 use_labels。

    参考：[#2818](https://www.sqlalchemy.org/trac/ticket/2818)

### 引擎

+   **[engine] [bug]**

    在`connect()`上引发错误的 DBAPI 不是 dbapi.Error 的子类（例如`TypeError`，`NotImplementedError`等）将不会经过 dialect 的`Dialect.is_disconnect()`例程特定的错误处理，也不会将其包装在`sqlalchemy.exc.DBAPIError`中。现在将以与执行过程中发生的方式传播未更改的异常。

    参考：[#2881](https://www.sqlalchemy.org/trac/ticket/2881)

+   **[engine] [bug] [pool]**

    `QueuePool`已经改进，不会在现有连接尝试阻塞时阻止新的连接尝试。以前，新连接的生成在监视溢出的块内被串行化；现在，溢出计数器在连接过程本身之外的自己的关键部分中被改变。 

    参考：[#2880](https://www.sqlalchemy.org/trac/ticket/2880)

+   **[engine] [bug] [pool]**

    对等待池化连接可用的逻辑进行了轻微调整，对于未指定超时的连接池，每隔半秒就会中断等待，以检查所谓的“中止”标志，这允许等待者在整个连接池被释放时中断；通常情况下，等待者应该由于 notify_all()而中断，但在极少数情况下可能会错过这个 notify_all()。这是在 0.8.0 中首次引入的逻辑的扩展，该问题只在压力测试中偶尔观察到。

    参考：[#2522](https://www.sqlalchemy.org/trac/ticket/2522)

+   **[engine] [bug]**

    修复了当在`Connection.execute()`中引发预-DBAPI `StatementError`时，SQL 语句会被错误地 ASCII 编码的错误，导致非 ASCII 语句的编码错误。现在字符串化保持在 Python unicode 中，从而避免编码错误。

    参考：[#2871](https://www.sqlalchemy.org/trac/ticket/2871)

### sql

+   **[sql] [功能]**

    通过`Inspector.get_unique_constraints()`方法，增加了对“唯一约束”反射的支持。感谢 Roman Podolyaka 的补丁。

    参考：[#1443](https://www.sqlalchemy.org/trac/ticket/1443)

### postgresql

+   **[postgresql] [bug]**

    修复了在使用 pypostgresql 适配器时，索引反射会错误解释 indkey 值的 bug，该适配器将这些值作为列表返回，而不是 psycopg2 返回的字符串类型。

    参考：[#2855](https://www.sqlalchemy.org/trac/ticket/2855)

### mssql

+   **[mssql] [bug]**

    修复了在 0.8.0 版本中引入的错误，即在 MSSQL 中，如果索引位于替代模式中，则`DROP INDEX`语句会显示错误；模式名/表名会被颠倒。格式也已经修订，以匹配当前的 MSSQL 文档。感谢 Derek Harland。

### oracle

+   **[oracle] [bug]**

    将“最大空闲时间”错误代码 ORA-02396 添加到了与 cx_oracle 一起的“断开连接”代码列表中。

    参考：[#2864](https://www.sqlalchemy.org/trac/ticket/2864)

+   **[oracle] [bug]**

    修复了一个 bug，其中 Oracle `VARCHAR`类型在没有长度的情况下（例如用于`CAST`或类似情况）会错误地呈现`None CHAR`或类似情况。

    参考：[#2870](https://www.sqlalchemy.org/trac/ticket/2870)

### 杂项

+   **[bug] [ext]**

    修复了一个 bug，该 bug 导致`serializer`扩展无法正确处理包含非 ASCII 字符的表或列名。

    参考：[#2869](https://www.sqlalchemy.org/trac/ticket/2869)

### orm

+   **[orm] [bug]**

    修复了由[#2818](https://www.sqlalchemy.org/trac/ticket/2818)引入的回归，生成的 EXISTS 查询会为具有两个同名列的语句产生“正在替换列”警告，因为内部 SELECT 不会设置 use_labels。

    参考：[#2818](https://www.sqlalchemy.org/trac/ticket/2818)

### engine

+   **[engine] [bug]**

    一个在`connect()`上引发错误的 DBAPI，如果不是`dbapi.Error`的子类（例如`TypeError`，`NotImplementedError`等），将不会改变异常。以前，`connect()`例程特定的错误处理既不恰当地通过方言的`Dialect.is_disconnect()`例程运行异常，也会将其包装在`sqlalchemy.exc.DBAPIError`中。现在，它将以与执行过程中发生的方式相同的方式传播。

    参考：[#2881](https://www.sqlalchemy.org/trac/ticket/2881)

+   **[engine] [bug] [pool]**

    `QueuePool`已经改进，当现有连接尝试阻塞时，不会阻止新的连接尝试。以前，新连接的生成在监视溢出的块内串行化；现在，溢出计数器在连接过程本身之外的自己的关键部分中进行了修改。

    参考：[#2880](https://www.sqlalchemy.org/trac/ticket/2880)

+   **[engine] [bug] [pool]**

    对等待可用连接的逻辑进行了轻微调整，对于未指定超时的连接池，每隔半秒就会中断等待以检查所谓的“中止”标志，这允许等待者在整个连接池被丢弃的情况下中断；通常，等待者应该由于 notify_all()而中断，但在极少数情况下可能会错过这个 notify_all()。这是在 0.8.0 中首次引入的逻辑的扩展，该问题只在压力测试中偶尔观察到。

    参考：[#2522](https://www.sqlalchemy.org/trac/ticket/2522)

+   **[engine] [bug]**

    修复了一个 bug，在`Connection.execute()`中引发预先 DBAPI `StatementError`时，SQL 语句会被错误地 ASCII 编码，导致非 ASCII 语句的编码错误。现在字符串化保持在 Python unicode 中，从而避免编码错误。

    参考：[#2871](https://www.sqlalchemy.org/trac/ticket/2871)

### SQL

+   **[sql] [feature]**

    通过`Inspector.get_unique_constraints()`方法，增加了对“唯一约束”反射的支持。感谢 Roman Podolyaka 的补丁。

    参考：[#1443](https://www.sqlalchemy.org/trac/ticket/1443)

### PostgreSQL

+   **[postgresql] [bug]**

    修复了一个 bug，其中在使用 pypostgresql 适配器时，索引反射会错误地解释 indkey 值，该适配器将这些值作为列表返回，而不是 psycopg2 的字符串返回类型。

    参考：[#2855](https://www.sqlalchemy.org/trac/ticket/2855)

### MSSQL

+   **[mssql] [bug]**

    修复了在 0.8.0 中引入的 bug，其中 MSSQL 中索引的`DROP INDEX`语句会在索引位于备用模式时错误呈现；模式名/表名会被颠倒。格式也已经修订，以匹配当前的 MSSQL 文档。感谢 Derek Harland。

### Oracle

+   **[oracle] [bug]**

    将 ORA-02396“最大空闲时间”错误代码添加到与 cx_oracle 一起的“断开连接”代码列表中。

    参考：[#2864](https://www.sqlalchemy.org/trac/ticket/2864)

+   **[oracle] [bug]**

    修复了一个 bug，其中没有长度的 Oracle `VARCHAR`类型（例如用于`CAST`或类似操作）会错误地呈现为`None CHAR`或类似情况。

    参考：[#2870](https://www.sqlalchemy.org/trac/ticket/2870)

### 杂项

+   **[bug] [ext]**

    修复了一个 bug，该 bug 导致`serializer`扩展无法正确处理包含非 ASCII 字符的表格或列名。

    参考：[#2869](https://www.sqlalchemy.org/trac/ticket/2869)

## 0.8.3

发布日期：2013 年 10 月 26 日

### ORM

+   **[orm] [feature]**

    添加了新选项`relationship()` `distinct_target_key`。这使得子查询急加载策略可以对最内部的 SELECT 子查询应用 DISTINCT，以帮助解决由最内部查询生成重复行的情况（尚无关于子查询急加载中重复行问题的通用解决方案，但是当最内部子查询之外的连接产生重复行时）。当标志设置为`True`时，DISTINCT 被无条件渲染，当设置为`None`时，如果最内部关系目标列不包括完整主键，则渲染 DISTINCT。该选项在 0.8 中默认为 False（例如，在所有情况下默认关闭），在 0.9 中为 None（例如，默认情况下自动）。感谢 Alexander Koval 对此的帮助。

    另请参阅

    子查询急加载将对某些查询的最内部 SELECT 应用 DISTINCT

    参考：[#2836](https://www.sqlalchemy.org/trac/ticket/2836)

+   **[orm] [bug]**

    修复了列表插入操作未能正确表示`[0:0]`的切片设置，特别是在使用`insert(0, item)`与关联代理时可能发生。由于 Python 集合中的一些怪癖，这个问题在 Python 3 中比在 Python 2 中更有可能发生。

    此更改也**回溯**到：0.7.11

    参考：[#2807](https://www.sqlalchemy.org/trac/ticket/2807)

+   **[orm] [bug]**

    修复了在使用类似`remote()`或`foreign()`在与父`Table`关联之前的`Column`上可能会产生与父表在连接中未呈现相关的问题的注释时可能会产生的问题，这是由于注释执行的固有复制操作。

    参考：[#2813](https://www.sqlalchemy.org/trac/ticket/2813)

+   **[orm] [bug]**

    修复了`Query.exists()`在没有任何 WHERE 条件的情况下无法正常工作的错误。感谢 Vladimir Magamedov。

    参考：[#2818](https://www.sqlalchemy.org/trac/ticket/2818)

+   **[orm] [bug]**

    从 0.9 中回溯了一个更改，其中用于多态继承加载的映射器层次结构的迭代被排序，这允许为多态查询生成的 SELECT 语句具有确定性渲染，从而有助于缓存方案，该方案基于 SQL 字符串本身进行缓存。

    参考：[#2779](https://www.sqlalchemy.org/trac/ticket/2779)

+   **[orm] [bug]**

    修复了 ORM 用于迭代映射器层次结构的有序序列实现中的潜在问题；在 Jython 解释器下，这个实现没有排序，尽管 cPython 和 PyPy 保持了顺序。

    参考：[#2794](https://www.sqlalchemy.org/trac/ticket/2794)

+   **[orm] [bug]**

    修复了 ORM 级别事件注册中的 bug，其中“原始”或“传播”标志在某些“未映射基类”配置中可能被错误配置。

    参考：[#2786](https://www.sqlalchemy.org/trac/ticket/2786)

+   **[orm] [bug]**

    与加载映射实体时使用`defer()`选项相关的性能修复。在加载时将每个对象的延迟可调用应用于实例的函数开销明显高于仅从行加载数据的开销（请注意，`defer()`旨在减少 DB/网络开销，而不一定是函数调用次数）；在所有情况下，函数调用开销现在小于从列加载数据的开销。每次加载从 N（结果中的总延迟值）到 1（延迟列的总数）的“延迟可调用”对象数量也减少了。

    参考：[#2778](https://www.sqlalchemy.org/trac/ticket/2778)

+   **[orm] [bug]**

    修复了一个 bug，当使用`make_transient()`函数将对象从“持久”移动到“挂起”时，属性历史函数会失败，用于涉及基于集合的反向引用的操作。

    参考：[#2773](https://www.sqlalchemy.org/trac/ticket/2773)

### orm 声明式

+   **[orm] [声明式] [功能]**

    添加了一个方便的类装饰器`as_declarative()`，是`declarative_base()`的包装器，允许使用一种巧妙的类装饰器方法应用现有的基类。

### 例子

+   **[例子] [功能]**

    改进了`examples/generic_associations`中的例子，包括`discriminator_on_association.py`利用单表继承来处理“鉴别器”的工作。还添加了一个真正的“通用外键”示例，它与其他流行框架类似，使用开放式整数指向任何其他表，放弃了传统的引用完整性。虽然我们不推荐这种模式，但信息想要自由。

+   **[例子] [bug]**

    在版本示例中创建的历史表中添加了“autoincrement=False”，因为这个表在任何情况下都不应该有自增，感谢 Patrick Schmid。

### 引擎

+   **[引擎] [功能]**

    对于`Engine`的`URL`的`repr()`现在将使用星号隐藏密码。感谢 Gunnlaugur Þór Briem。

    参考：[#2821](https://www.sqlalchemy.org/trac/ticket/2821)

+   **[engine] [bug]**

    `make_url()`函数使用的正则表达式现在解析 ipv6 地址，例如用方括号括起来。

    此更改也被**回溯**到：0.7.11

    参考：[#2851](https://www.sqlalchemy.org/trac/ticket/2851)

+   **[engine] [bug] [oracle]**

    如果重新创建`Engine`，则不会第二次调用 Dialect.initialize()，因为出现了断开连接错误。这修复了 Oracle 8 方言中的一个特定问题，但通常情况下，dialect.initialize()阶段应该只执行一次。

    参考：[#2776](https://www.sqlalchemy.org/trac/ticket/2776)

+   **[engine] [bug] [pool]**

    修复了`QueuePool`在现有池化连接在无效或重新生成事件后未能重新连接时会丢失正确的已检出计数的 bug。

    参考：[#2772](https://www.sqlalchemy.org/trac/ticket/2772)

### sql

+   **[sql] [feature]**

    在`insert()`构造中添加了新方法`Insert.from_select()`。给定列的列表和可选择的内容，渲染`INSERT INTO (table) (columns) SELECT ..`。

    参考：[#722](https://www.sqlalchemy.org/trac/ticket/722)

+   **[sql] [feature]**

    `update()`、`insert()`和`delete()`构造现在将解释 ORM 实体作为要操作的目标表，例如：

    ```py
    from sqlalchemy import insert, update, delete

    ins = insert(SomeMappedClass).values(x=5)

    del_ = delete(SomeMappedClass).where(SomeMappedClass.id == 5)

    upd = update(SomeMappedClass).where(SomeMappedClass.id == 5).values(name="ed")
    ```

+   **[sql] [bug]**

    修复了自 0.7.9 以来的回归，即如果在多个 FROM 子句中引用 CTE，则可能无法正确引用 CTE 的名称。

    此更改也被**回溯**到：0.7.11

    参考：[#2801](https://www.sqlalchemy.org/trac/ticket/2801)

+   **[sql] [bug] [cte]**

    修复了通用表达式系统中的一个 bug，如果 CTE 仅用作`alias()`构造，则不会使用 WITH 关键字进行渲染。

    此更改也被**回溯**到：0.7.11

    参考：[#2783](https://www.sqlalchemy.org/trac/ticket/2783)

+   **[sql] [bug]**

    修复了`CheckConstraint` DDL 中的一个 bug，其中来自`Column`对象的“quote”标志不会被传播。

    这个更改也被**回溯**到：0.7.11

    参考：[#2784](https://www.sqlalchemy.org/trac/ticket/2784)

+   **[sql] [bug]**

    修复了`type_coerce()`无法正确解释具有`__clause_element__()`方法的 ORM 元素的 bug。

    参考：[#2849](https://www.sqlalchemy.org/trac/ticket/2849)

+   **[sql] [bug]**

    当生成“非本地”类型的 CHECK 约束时，`Enum`和`Boolean`类型现在会绕过任何自定义（例如 TypeDecorator）类型的使用。这样，自定义类型不会参与 CHECK 中的表达式，因为这个表达式是针对“impl”值而不是“decorated”值的。

    参考：[#2842](https://www.sqlalchemy.org/trac/ticket/2842)

+   **[sql] [bug]**

    在`Index`上的`.unique`标志可能会产生`None`，如果它是从未指定`unique`（默认为`None`）的`Column`生成的。该标志现在将始终是`True`或`False`。

    参考：[#2825](https://www.sqlalchemy.org/trac/ticket/2825)

+   **[sql] [bug]**

    修复了默认编译器以及 postgresql、mysql 和 mssql 的 bug，以确保任何字面 SQL 表达式值在 CREATE INDEX 语句中直接呈现为字面值，而不是作为绑定参数。这也改变了其他 DDL 的呈现方案，如约束。

    参考：[#2742](https://www.sqlalchemy.org/trac/ticket/2742)

+   **[sql] [bug]**

    当`select()`在其 FROM 子句中引用自身时，通常通过就地突变，将引发一个信息性错误消息，而不是导致递归溢出。

    参考：[#2815](https://www.sqlalchemy.org/trac/ticket/2815)

+   **[sql] [bug]**

    不起作用的“schema”参数在`ForeignKey`上已被弃用；会发出警告。在 0.9 中移除。

    参考：[#2831](https://www.sqlalchemy.org/trac/ticket/2831)

+   **[sql] [bug]**

    修复了使用`column_reflect`事件来更改传入`Column`的`.key`会阻止正确反映主键约束、索引和外键约束的 bug。

    参考：[#2811](https://www.sqlalchemy.org/trac/ticket/2811)

+   **[sql] [bug]**

    `ColumnOperators.notin_()` 运算符在 0.8 版本中添加，现在正确地生成了针对空集合使用时“IN”表达式的否定结果。

+   **[sql] [bug] [postgresql]**

    修复了表达式系统依赖于`select()`构造中的`.c`集合上的一些表达式的`str()`形式的错误，但由于元素依赖于特定于方言的编译构造，特别是与 PostgreSQL `ARRAY` 元素一起使用的 `__getitem__()` 运算符，因此`str()`形式不可用。 修复还添加了一个新的异常类`UnsupportedCompilationError`，在编译器被要求编译它不知道如何处理的内容时引发该异常。

    参考：[#2780](https://www.sqlalchemy.org/trac/ticket/2780)

### postgresql

+   **[postgresql] [bug]**

    移除了对列的服务器默认值反射的 128 字符截断；这段代码最初来自 PG 系统视图，用于截断字符串以便阅读。

    参考：[#2844](https://www.sqlalchemy.org/trac/ticket/2844)

+   **[postgresql] [bug]**

    括号将应用于在 CREATE INDEX 语句的列列表中呈现的复合 SQL 表达式。

    参考：[#2742](https://www.sqlalchemy.org/trac/ticket/2742)

+   **[postgresql] [bug]**

    修复了 PostgreSQL 版本字符串中在“PostgreSQL”或“EnterpriseDB”之前有前缀的情况无法解析的错误。感谢 Scott Schaefer。

    参考：[#2819](https://www.sqlalchemy.org/trac/ticket/2819)

### mysql

+   **[mysql] [bug]**

    更新了 MySQL 5.5、5.6 版本的保留字，感谢 Hanno Schlichting。

    此更改也被**回溯**到：0.7.11

    参考：[#2791](https://www.sqlalchemy.org/trac/ticket/2791)

+   **[mysql] [bug]**

    在[#2721](https://www.sqlalchemy.org/trac/ticket/2721)中的更改，即 MySQL 后端对 `ForeignKeyConstraint` 的 `deferrable` 关键字被静默忽略，将在 0.9 版本中被撤销；这个关键字现在将再次呈现，在 MySQL 上引发错误，因为它不被理解 - 相同的行为也将适用于 `initially` 关键字。 在 0.8 版本中，这些关键字将继续被忽略，但会发出警告。此外，`match` 关键字现在在 0.9 上引发 `CompileError`，在 0.8 上发出警告；这个关键字不仅被 MySQL 静默忽略，还会破坏 ON UPDATE/ON DELETE 选项。

    要使用不在 MySQL 上呈现或在 MySQL 上呈现不同的 `ForeignKeyConstraint`，请使用自定义编译选项。已将此用法示例添加到文档中，请参阅 MySQL / MariaDB 外键。

    参考：[#2721](https://www.sqlalchemy.org/trac/ticket/2721), [#2839](https://www.sqlalchemy.org/trac/ticket/2839)

+   **[mysql] [bug]**

    MySQL-connector 方言现在允许在 create_engine 查询字符串中使用选项来覆盖在连接中设置的默认值，包括“buffered”和“raise_on_warnings”。

    参考：[#2515](https://www.sqlalchemy.org/trac/ticket/2515)

### sqlite

+   **[sqlite] [bug]**

    新添加的 SQLite DATETIME 参数 storage_format 和 regexp 显然没有完全正确实现；虽然接受了参数，但实际上它们没有任何效果；这个问题已经修复。

    参考：[#2781](https://www.sqlalchemy.org/trac/ticket/2781)

### oracle

+   **[oracle] [bug]**

    修复了使用同义词进行 Oracle 表反射时，如果同义词和表位于不同的远程模式中，则会失败的 bug。修复补丁由 Kyle Derr 提供。

    参考：[#2853](https://www.sqlalchemy.org/trac/ticket/2853)

### 杂项

+   **[feature]**

    向`Column`添加了一个新标志`system=True`，将列标记为“系统”列，该列将自动由数据库（如 PostgreSQL 的 `oid` 或 `xmin`）生成。 该列将在`CREATE TABLE`语句中被省略，但仍可用于查询。此外，`CreateColumn`构造可以应用于自定义编译规则，允许跳过列，通过生成返回`None`的规则。

### orm

+   **[orm] [feature]**

    向 `relationship()`添加了新选项 `distinct_target_key`。这使得子查询预加载策略可以对最内部的 SELECT 子查询应用 DISTINCT，以帮助解决由该关系对应的最内部查询生成重复行的情况（目前还没有解决子查询预加载中重复行的一般解决方案，但是当最内部子查询之外的连接产生重复行时）。当标志设置为`True`时，DISTINCT 无条件呈现，当设置为`None`时，如果最内部关系目标列不包括完整主键，则呈现 DISTINCT。该选项在 0.8 版本中默认为 False（例如，在所有情况下默认关闭），在 0.9 版本中默认为 None（例如，默认情��下自动）。感谢 Alexander Koval 对此的帮助。

    另请参阅

    子查询预加载将对某些查询的最内部 SELECT 应用 DISTINCT

    参考：[#2836](https://www.sqlalchemy.org/trac/ticket/2836)

+   **[orm] [bug]**

    修复了列表插入操作`insert(0, item)`在使用关联代理时可能出现的`[0:0]`切片表示错误的 bug。由于 Python 集合中的一些怪癖，这个问题在 Python 3 中比在 Python 2 中更容易出现。

    这个改动也被**回溯**到了：0.7.11

    参考：[#2807](https://www.sqlalchemy.org/trac/ticket/2807)

+   **[orm] [bug]**

    修复了在使用`Column`之前对父级`Table`进行关联时，使用`remote()`或`foreign()`等注释可能导致父表在连接中未渲染的 bug，这是由于注释执行的固有复制操作所致。

    参考：[#2813](https://www.sqlalchemy.org/trac/ticket/2813)

+   **[orm] [bug]**

    修复了`Query.exists()`在没有任何 WHERE 条件的情况下无法正常工作的 bug。感谢 Vladimir Magamedov。

    参考：[#2818](https://www.sqlalchemy.org/trac/ticket/2818)

+   **[orm] [bug]**

    从 0.9 版本中回溯了一个改动，该改动使得在多态继承加载中使用的映射器层次结构的迭代是有序的，这允许为多态查询生成的 SELECT 语句具有确定性渲染，从而有助于缓存方案在 SQL 字符串本身上进行缓存。

    参考：[#2779](https://www.sqlalchemy.org/trac/ticket/2779)

+   **[orm] [bug]**

    修复了 ORM 用于迭代映射器层次结构的有序序列实现中可能存在的问题；在 Jython 解释器下，这个实现没有保持顺序，尽管 cPython 和 PyPy 保持了顺序。

    参考：[#2794](https://www.sqlalchemy.org/trac/ticket/2794)

+   **[orm] [bug]**

    修复了 ORM 级别事件注册中“raw”或“propagate”标志在一些“未映射基类”配置中可能被错误配置的 bug。

    参考：[#2786](https://www.sqlalchemy.org/trac/ticket/2786)

+   **[orm] [bug]**

    与加载映射实体时使用`defer()`选项相关的性能修复。在加载时将每个对象的延迟可调用应用于实例的函数开销明显高于仅从行加载数据的开销（请注意，`defer()`旨在减少 DB/网络开销，而不一定是函数调用次数）；在所有情况下，函数调用开销现在都小于从列加载数据的开销。每次加载从 N（结果中的总延迟值）到 1（延迟列的总数）的“延迟可调用”对象数量也减少了。

    参考：[#2778](https://www.sqlalchemy.org/trac/ticket/2778)

+   **[orm] [错误]**

    修复了一个 bug，当使用`make_transient()`函数将对象从“持久”移动到“挂起”时，涉及基于集合的反向引用的操作会导致属性历史函数失败。

    参考：[#2773](https://www.sqlalchemy.org/trac/ticket/2773)

### orm 声明式

+   **[orm] [声明式] [特性]**

    添加了一个方便的类装饰器`as_declarative()`，它是`declarative_base()`的包装器，允许使用一种巧妙的类装饰器方法应用现有的基类。

### 示例

+   **[示例] [特性]**

    改进了`examples/generic_associations`中的示例，包括`discriminator_on_association.py`使用单表继承来处理“鉴别器”。还添加了一个真正的“通用外键”示例，它类似于其他流行框架，使用开放的整数指向任何其他表，放弃传统的引用完整性。虽然我们不推荐这种模式，但信息想要自由。

+   **[示例] [错误]**

    在版本控制示例中创建的历史表中添加了“autoincrement=False”，因为这个表在任何情况下都不应该具有自增属性，感谢 Patrick Schmid。

### 引擎

+   **[引擎] [特性]**

    对`Engine`的`URL`的`repr()`现在将使用星号隐藏密码。感谢 Gunnlaugur Þór Briem。

    参考：[#2821](https://www.sqlalchemy.org/trac/ticket/2821)

+   **[引擎] [错误]**

    `make_url()`函数现在使用的正则表达式可以解析 ipv6 地址，例如用方括号括起来。

    此更改也**回溯**到：0.7.11

    参考：[#2851](https://www.sqlalchemy.org/trac/ticket/2851)

+   **[引擎] [错误] [oracle]**

    如果重新创建了一个`Engine`，则不会再次调用 Dialect.initialize()，这是因为出现了断开连接的错误。这修复了 Oracle 8 方言中的一个特定问题，但通常情况下，dialect.initialize() 阶段应该只执行一次。

    参考：[#2776](https://www.sqlalchemy.org/trac/ticket/2776)

+   **[engine] [bug] [pool]**

    修复了如果现有的池化连接在失效或重新生成事件后无法重新连接，则`QueuePool` 会丢失正确的检出计数的错误。

    参考：[#2772](https://www.sqlalchemy.org/trac/ticket/2772)

### sql

+   **[sql] [feature]**

    添加了一个新方法到 `insert()` 构造中，`Insert.from_select()`。给定一个列列表和一个可选择项，渲染 `INSERT INTO (table) (columns) SELECT ..`。

    参考：[#722](https://www.sqlalchemy.org/trac/ticket/722)

+   **[sql] [feature]**

    `update()`、`insert()` 和 `delete()` 构造现在会将 ORM 实体解释为要操作的目标表，例如：

    ```py
    from sqlalchemy import insert, update, delete

    ins = insert(SomeMappedClass).values(x=5)

    del_ = delete(SomeMappedClass).where(SomeMappedClass.id == 5)

    upd = update(SomeMappedClass).where(SomeMappedClass.id == 5).values(name="ed")
    ```

+   **[sql] [bug]**

    修复了回归问题，自 0.7.9 以来，如果 CTE 在多个 FROM 子句中被引用，则可能无法正确引用其名称。

    这个改变也被**反向移植**到了：0.7.11

    参考：[#2801](https://www.sqlalchemy.org/trac/ticket/2801)

+   **[sql] [bug] [cte]**

    修复了通用表达式系统中的 bug，如果 CTE 仅用作 `alias()` 构造，则不会使用 WITH 关键字进行呈现。

    这个改变也被**反向移植**到了：0.7.11

    参考：[#2783](https://www.sqlalchemy.org/trac/ticket/2783)

+   **[sql] [bug]**

    修复了 `CheckConstraint` DDL 中的 bug，在这个 bug 中，来自 `Column` 对象的 “quote” 标志不会传播。

    这个改变也被**反向移植**到了：0.7.11

    参考：[#2784](https://www.sqlalchemy.org/trac/ticket/2784)

+   **[sql] [bug]**

    修复了 `type_coerce()` 无法正确解释具有 `__clause_element__()` 方法的 ORM 元素的 bug。

    参考：[#2849](https://www.sqlalchemy.org/trac/ticket/2849)

+   **[sql] [bug]**

    当为“非本地”类型生成 CHECK 约束时，`Enum` 和 `Boolean` 类型现在会跳过任何自定义（例如 TypeDecorator）类型。这样，自定义类型不会参与 CHECK 中的表达式，因为此表达式针对的是“impl”值而不是“decorated”值。

    参考：[#2842](https://www.sqlalchemy.org/trac/ticket/2842)

+   **[sql] [bug]**

    如果来自未指定 `unique` 的 `Column` （其中默认为 `None`）生成了 `Index` 的 `.unique` 标志可能会产生 `None`。现在该标志将始终为 `True` 或 `False`。

    参考：[#2825](https://www.sqlalchemy.org/trac/ticket/2825)

+   **[sql] [bug]**

    修复了在默认编译器中的 bug，以及 postgresql、mysql 和 mssql 的 bug，以确保在 CREATE INDEX 语句中直接将任何文本 SQL 表达式值呈现为文字，而不是绑定参数。这也改变了其他 DDL 的呈现方案，如约束。

    参考：[#2742](https://www.sqlalchemy.org/trac/ticket/2742)

+   **[sql] [bug]**

    在其 FROM 子句中引用自身的 `select()`，通常通过原地修改，现在会引发一个信息性错误消息，而不会导致递归溢出。

    参考：[#2815](https://www.sqlalchemy.org/trac/ticket/2815)

+   **[sql] [bug]**

    在 `ForeignKey` 上使用的不起作用的“schema”参数已被弃用；会引发警告。在 0.9 中已移除。

    参考：[#2831](https://www.sqlalchemy.org/trac/ticket/2831)

+   **[sql] [bug]**

    修复了使用 `column_reflect` 事件来更改传入 `Column` 的 `.key` 会阻止正确反映主键约束、索引和外键约束的 bug。

    参考：[#2811](https://www.sqlalchemy.org/trac/ticket/2811)

+   **[sql] [bug]**

    `ColumnOperators.notin_()` 操作符在 0.8 中添加，现在正确地生成了空集合时“IN”返回的表达式的否定。

+   **[sql] [bug] [postgresql]**

    修复了一个 bug，表达式系统在引用`select()`构造中的`.c`集合时依赖于一些表达式的`str()`形式，但是由于元素依赖于特定于方言的编译构造，特别是与 PostgreSQL `ARRAY`元素一起使用的`__getitem__()`运算符，因此`str()`形式不可用。修复还添加了一个新的异常类`UnsupportedCompilationError`，在编译器被要求编译它不知道如何处理的内容时引发该异常。

    参考：[#2780](https://www.sqlalchemy.org/trac/ticket/2780)

### postgresql

+   **[postgresql] [bug]**

    删除了对列的服务器默认值反射的 128 字符截断；此代码最初来自 PG 系统视图，用于为可读性截断字符串。

    参考：[#2844](https://www.sqlalchemy.org/trac/ticket/2844)

+   **[postgresql] [bug]**

    括号将被应用于在 CREATE INDEX 语句的列列表中呈现的复合 SQL 表达式。

    参考：[#2742](https://www.sqlalchemy.org/trac/ticket/2742)

+   **[postgresql] [bug]**

    修复了一个 bug，其中具有在“PostgreSQL”或“EnterpriseDB”之前的前缀的 PostgreSQL 版本字符串无法解析。感谢 Scott Schaefer。

    参考：[#2819](https://www.sqlalchemy.org/trac/ticket/2819)

### mysql

+   **[mysql] [bug]**

    更新了 MySQL 保留字的版本 5.5、5.6，感谢 Hanno Schlichting。

    此更改也**回溯**到：0.7.11

    参考：[#2791](https://www.sqlalchemy.org/trac/ticket/2791)

+   **[mysql] [bug]**

    在 0.9 版本中，将撤销[#2721](https://www.sqlalchemy.org/trac/ticket/2721)中的更改，即在 MySQL 后端上`ForeignKeyConstraint`的`deferrable`关键字将被静默忽略，现在将再次呈现此关键字，在 MySQL 上引发错误，因为它不被理解 - 相同的行为也将适用于`initially`关键字。在 0.8 版本中，这些关键字将继续被忽略，但会发出警告。此外，在 0.9 版本中，`match`关键字现在会引发一个`CompileError`，在 0.8 版本中会发出警告；这个关键字不仅被 MySQL 静默忽略，而且会破坏 ON UPDATE/ON DELETE 选项。

    要使用一个不在 MySQL 上呈现或呈现方式不同的`ForeignKeyConstraint`，请使用自定义编译选项。文档中已添加了此用法示例，请参见 MySQL / MariaDB 外键。

    参考：[#2721](https://www.sqlalchemy.org/trac/ticket/2721)，[#2839](https://www.sqlalchemy.org/trac/ticket/2839)

+   **[mysql] [bug]**

    MySQL-connector 方言现在允许在 create_engine 查询字符串中使用选项来覆盖连接中设置的默认值，包括“buffered”和“raise_on_warnings”。

    参考：[#2515](https://www.sqlalchemy.org/trac/ticket/2515)

### sqlite

+   **[sqlite] [bug]**

    新增的 SQLite DATETIME 参数 storage_format 和 regexp 显然没有完全正确实现；虽然参数被接受了，但实际上它们没有任何效果；这个问题已经修复。

    参考：[#2781](https://www.sqlalchemy.org/trac/ticket/2781)

### oracle

+   **[oracle] [bug]**

    修复了一个 bug，Oracle 表反射使用同义词会失败，如果同义词和表位于不同的远程模式中。修复补丁由 Kyle Derr 提供。

    参考：[#2853](https://www.sqlalchemy.org/trac/ticket/2853)

### 杂项

+   **[feature]**

    添加了一个新的标志 `system=True` 到 `Column` 中，将列标记为“系统”列，该列将由数据库自动添加（例如 PostgreSQL 的 `oid` 或 `xmin`）。该列将从 `CREATE TABLE` 语句中省略，但在其他情况下可供查询使用。此外，`CreateColumn` 构造可以应用于自定义编译规则，允许跳过列，通过生成返回 `None` 的规则。

## 0.8.2

发布日期：2013 年 7 月 3 日

### orm

+   **[orm] [feature]**

    添加了一个新的方法 `Query.select_entity_from()`，将在 0.9 版本中部分取代 `Query.select_from()` 的功能。在 0.8 版本中，这两个方法执行相同的功能，因此代码可以相应地迁移到使用 `Query.select_entity_from()` 方法。详细信息请参阅 0.9 迁移指南。

    参考：[#2736](https://www.sqlalchemy.org/trac/ticket/2736)

+   **[orm] [bug]**

    当尝试刷新一个继承类的对象时，如果多态鉴别器已经被分配了对于该类无效的值，则会发出警告。

    参考：[#2750](https://www.sqlalchemy.org/trac/ticket/2750)

+   **[orm] [bug]**

    修复了多态 SQL 生成中的 bug，在多个针对同一基类的连接继承实体之间连接到彼此时，如果连接字符串长度超过两个实体，则不会独立跟踪基表上的列。

    参考：[#2759](https://www.sqlalchemy.org/trac/ticket/2759)

+   **[orm] [bug]**

    修复了一个 bug，在将复合属性发送到 `Query.order_by()` 中时，会产生一种括号表达式，某些数据库不接受。

    参考：[#2754](https://www.sqlalchemy.org/trac/ticket/2754)

+   **[orm] [bug]**

    修复了复合属性与`aliased()`函数之间的交互。以前，在应用别名时，复合属性在比较操作中不会正确工作。

    参考：[#2755](https://www.sqlalchemy.org/trac/ticket/2755)

+   **[orm] [bug] [ext]**

    修复了`MutableDict`在调用`clear()`时未报告更改事件的错误。

    参考：[#2730](https://www.sqlalchemy.org/trac/ticket/2730)

+   **[orm] [bug]**

    修复了由[#2682](https://www.sqlalchemy.org/trac/ticket/2682)引起的回归，即由于使用`IS`而出现的不受支持的`True`和`False`符号导致`Query.update()`和`Query.delete()`调用的评估失败。

    参考：[#2737](https://www.sqlalchemy.org/trac/ticket/2737)

+   **[orm] [bug]**

    修复了由此票引起的 0.7 版本的回归，使得在自引用的急切连接中检查递归溢出的过程过于宽松，错过了一个特定情况，即子类配置了 lazy=”joined”或“subquery”，并且加载是针对基类的“with_polymorphic”。

    参考：[#2481](https://www.sqlalchemy.org/trac/ticket/2481)

+   **[orm] [bug]**

    修复了从 0.7 版本引起的回归，即`Session.begin_nested()`的上下文管理器功能在发生刷新错误时未能正确回滚事务，而是引发自己的异常，同时使会话仍处于待回滚状态。

    参考：[#2718](https://www.sqlalchemy.org/trac/ticket/2718)

### orm 声明式

+   **[orm] [declarative] [feature]**

    现在可以通过名称引用 ORM 描述符，如混合属性，在与`order_by`、`primaryjoin`或类似的字符串参数一起在`relationship()`中使用，除了列绑定属性。

    参考：[#2761](https://www.sqlalchemy.org/trac/ticket/2761)

### 示例

+   **[examples] [bug]**

    修复了“版本控制”配方中的一个问题，即当存在反向引用时，一个多对一引用可能会为目标产生一个无意义的版本，即使它没有更改。修补程序由 Matt Chisholm 提供。

+   **[examples] [bug]**

    修复了狗窝示例中的一个小错误，即生成 SQL 缓存键时未像`Query`通常做的那样对语句应用去重标签。

### engine

+   **[engine] [bug]**

    修复了一个 bug，即各种`Pool` 实现的`reset_on_return`参数在重新生成池时不会传播。感谢 Eevee。

+   **[engine] [bug] [sybase]**

    修复了一个 bug，即检测正确的 kwargs 被发送到`create_engine()`的例程在某些情况下会失败，比如在 Sybase 方言中。

    参考：[#2732](https://www.sqlalchemy.org/trac/ticket/2732)

### sql

+   **[sql] [feature]**

    为`TypeDecorator`提供了一个名为`TypeDecorator.coerce_to_is_types`的新属性，以便更容易控制使用`==`或`!=`与`None`和布尔类型进行比较时如何产生`IS`表达式，或者带有绑定参数的普通相等���达式。

    参考：[#2734](https://www.sqlalchemy.org/trac/ticket/2734), [#2744](https://www.sqlalchemy.org/trac/ticket/2744)

+   **[sql] [bug]**

    对`Select` 构造中引入的相关联行为进行了多次修复，这是从 0.8.0 版本开始的：

    +   为了满足 FROM 条目应该向外部相关联到包含另一个 SELECT 的 SELECT，然后再包含这个 SELECT 的用例，当通过`Select.correlate()`建立明确的相关联时，现在在多个级别上进行相关联，只要目标 SELECT 在链中的某处被包含在一个 WHERE/ORDER BY/columns 子句中，而不仅仅是嵌套的 FROM 子句。这使得`Select.correlate()`的行为再次更加兼容 0.7，同时仍然保持新的“智能”相关联。

    +   当未使用明确的相关联时，“隐式”相关联将其行为限制在仅仅是直接包含的 SELECT 中，以最大程度地提高与 0.7 应用程序的兼容性，并且在这种情况下还防止了跨嵌套 FROM 的相关联，保持了与 0.8.0/0.8.1 的兼容性。

    +   `Select.correlate_except()` 方法在所有情况下都未能阻止给定的 FROM 子句进行相关联，并且还会导致 FROM 子句被错误地完全省略（更像是 0.7 版本的行为），这个问题已经修复。

    +   调用 select.correlate_except(None)将使所有 FROM 子句进入相关联，正如预期的那样。

    参考：[#2668](https://www.sqlalchemy.org/trac/ticket/2668), [#2746](https://www.sqlalchemy.org/trac/ticket/2746)

+   **[sql] [bug]**

    修复了一个 bug，即将一个表“A”的 select() 与多个外键路径连接到表“B”，到表“B”，如果直接将表“A”连接到“B”会报告“模糊的连接条件”错误，但连接条件会包含多个条件。

    参考：[#2738](https://www.sqlalchemy.org/trac/ticket/2738)

+   **[sql] [bug] [reflection]**

    修复了一个 bug，即在远程模式下使用 `MetaData.reflect()` 跨越一个本地模式和一个远程模式的模式时，如果两个模式都有相同名称的表，则可能产生错误的结果。

    参考：[#2728](https://www.sqlalchemy.org/trac/ticket/2728)

+   **[sql] [bug]**

    从基础 `ColumnOperators` 类中移除了“未实现”的`__iter__()`调用，虽然这是在 0.8.0 中引入的，以防止在自定义运算符上实现`__getitem__()`方法并在该对象上错误调用`list()`时出现无限、内存增长的循环，但这会导致列元素报告它们实际上是可迭代类型，然后在尝试迭代时抛出错误。在这里没有真正的办法同时拥有两边，所以我们坚持使用 Python 的最佳实践。在自定义运算符上实现`__getitem__()`时要小心！

    参考：[#2726](https://www.sqlalchemy.org/trac/ticket/2726)

+   **[sql] [bug] [mssql]**

    由于这个问题导致不支持的关键字“true”被呈现，添加逻辑将其转换为 SQL Server 的 1/0。

    参考：[#2682](https://www.sqlalchemy.org/trac/ticket/2682)

### postgresql

+   **[postgresql] [feature]**

    添加了对 PostgreSQL 9.2 范围类型的支持。目前，不提供类型转换，因此目前直接使用字符串或 psycopg2 2.5 范围扩展类型。补丁由 Chris Withers 提供。

+   **[postgresql] [feature]**

    在使用 psycopg2 DBAPI 时，添加了对“AUTOCOMMIT”隔离的支持。该关键字可通过`isolation_level`执行选项使用。补丁由 Roman Podolyaka 提供。

    参考：[#2072](https://www.sqlalchemy.org/trac/ticket/2072)

+   **[postgresql] [bug]**

    在 PostgreSQL 方言上，`extract()` 的行为已简化，不再向给定表达式注入硬编码的`::timestamp`或类似转换，因为这会干扰诸如时区感知日期时间之类的类型，但在现代版本的 psycopg2 中似乎也不是必要的。

    参考：[#2740](https://www.sqlalchemy.org/trac/ticket/2740)

+   **[postgresql] [bug]**

    修复了 HSTORE 类型中包含反斜杠引号的键/值在使用“非本机”（即非-psycopg2）手段翻译 HSTORE 数据时无法正确转义的错误。感谢 Ryan Kelly 提供的补丁。

    参考：[#2766](https://www.sqlalchemy.org/trac/ticket/2766)

+   **[postgresql] [bug]**

    修复了多列 PostgreSQL 索引中列的顺序反映错误的 bug。感谢 Roman Podolyaka。

    参考：[#2767](https://www.sqlalchemy.org/trac/ticket/2767)

+   **[postgresql] [bug]**

    修复了 HSTORE 类型以正确编码/解码 Unicode。这始终开启，因为 hstore 是一种文本类型，并且在使用 Python 3 时与 psycopg2 的行为匹配。感谢 Dmitry Mugtasimov。

    参考：[#2735](https://www.sqlalchemy.org/trac/ticket/2735)

### mysql

+   **[mysql] [功能]**

    用于 `Index` 的 `mysql_length` 参数现在可以作为列名/长度字典传递，用于复合索引。非常感�� Roman Podolyaka 提供的补丁。

    参考：[#2704](https://www.sqlalchemy.org/trac/ticket/2704)

+   **[mysql] [bug]**

    修复了在使用多表 UPDATE 时出现的 bug，其中一个补充表是带有自己绑定参数的 SELECT，当使用 MySQL 的特殊语法时，绑定参数的位置与语句本身相反。

    参考：[#2768](https://www.sqlalchemy.org/trac/ticket/2768)

+   **[mysql] [bug]**

    添加了另一个条件到 `mysql+gaerdbms` 方言中，以检测所谓的“开发”模式，在这种模式下，我们应该使用 `rdbms_mysqldb` DBAPI。感谢 Brett Slatkin 提供的补丁。

    参考：[#2715](https://www.sqlalchemy.org/trac/ticket/2715)

+   **[mysql] [bug]**

    在 `ForeignKey` 和 `ForeignKeyConstraint` 上的 `deferrable` 关键字参数不会在 MySQL 方言上呈现 `DEFERRABLE` 关键字。很长一段时间以来，我们一直保留这个设置，因为非延迟外键与延迟外键的行为非常不同，但某些环境只是在 MySQL 上禁用 FKs，所以我们在这里会少些主观意见。

    参考：[#2721](https://www.sqlalchemy.org/trac/ticket/2721)

+   **[mysql] [bug]**

    更新了 mysqlconnector 方言以根据异常中发送的明显字符串消息检查断开连接；已针对 mysqlconnector 1.0.9 进行测试。

### sqlite

+   **[sqlite] [bug]**

    将 `sqlalchemy.types.BIGINT` 添加到可以由 SQLite 方言反射的类型名称列表中；感谢 Russell Stuart。

    参考：[#2764](https://www.sqlalchemy.org/trac/ticket/2764)

### mssql

+   **[mssql] [bug]**

    在查询 SQL Server 2000 上的信息模式时，移除了在 0.8.1 版本中添加的 CAST 调用，以帮助处理驱动程序问题，显然在 2000 年不兼容。CAST 仍然适用于 SQL Server 2005 及更高版本。

    参考：[#2747](https://www.sqlalchemy.org/trac/ticket/2747)

### misc

+   **[feature] [firebird]**

    为 kinterbasdb 和 fdb 方言添加了新标志 `retaining=True`。这控制发送到 DBAPI 连接的 `commit()` 和 `rollback()` 方法的 `retaining` 标志的值。由于历史原因，在 0.8.2 版本中，此标志默认为 `True`，但在 0.9.0b1 版本中，此标志默认为 `False`。

    参考：[#2763](https://www.sqlalchemy.org/trac/ticket/2763)

+   **[bug] [firebird]**

    在反射 Firebird 类型 LONG 和 INT64 时，类型查找已修复，使得 LONG 被视为 INTEGER，INT64 被视为 BIGINT，除非该类型具有“精度”，在这种情况下，它将被视为 NUMERIC。补丁由 Russell Stuart 提供。

    参考：[#2757](https://www.sqlalchemy.org/trac/ticket/2757)

+   **[bug] [ext]**

    修复了一个错误，即如果使用函数而不是类设置复合类型，则当尝试检查该列是否为 `MutableComposite` 时，可变扩展会出错（它不是）。感谢 asldevi。

+   **[requirements]**

    现在运行单元测试套件需要 Python 的 [mock](https://pypi.org/project/mock) 库。虽然作为 Python 3.3 的一部分，但之前的 Python 安装需要安装此库才能运行单元测试或使用 `sqlalchemy.testing` 包进行外部方言。

### orm

+   **[orm] [feature]**

    添加了一个新方法 `Query.select_entity_from()`，将在 0.9 版本中取代 `Query.select_from()` 的部分功能。在 0.8 版本中，这两个方法执行相同的功能，因此可以将代码迁移到适当使用 `Query.select_entity_from()` 方法。详细信息请参阅 0.9 迁移指南。

    参考：[#2736](https://www.sqlalchemy.org/trac/ticket/2736)

+   **[orm] [bug]**

    当尝试刷新一个继承类的对象时，如果多态鉴别器被分配给了该类无效的值，将会发出警告。

    参考：[#2750](https://www.sqlalchemy.org/trac/ticket/2750)

+   **[orm] [bug]**

    修复了多态 SQL 生成中的错误，如果多个继承实体针对同一基类相互连接，且连接字符串超过两个实体，则基表上的列将无法独立跟踪彼此。

    参考：[#2759](https://www.sqlalchemy.org/trac/ticket/2759)

+   **[orm] [bug]**

    修复了将复合属性发送到`Query.order_by()`会产生一种某些数据库不接受的带括号的表达式的错误。

    参考：[#2754](https://www.sqlalchemy.org/trac/ticket/2754)

+   **[orm] [错误]**

    修复了复合属性与`aliased()`函数之间的交互。 以前，在应用别名时，复合属性在比较操作中无法正常工作。

    参考：[#2755](https://www.sqlalchemy.org/trac/ticket/2755)

+   **[orm] [错误] [扩展]**

    修复了当调用`clear()`时`MutableDict`未报告更改事件的错误。

    参考：[#2730](https://www.sqlalchemy.org/trac/ticket/2730)

+   **[orm] [错误]**

    修复了由[#2682](https://www.sqlalchemy.org/trac/ticket/2682)引起的回归，即`Query.update()`和`Query.delete()`调用的评估会遇到由于使用`IS`而出现的不支持的`True`和`False`符号。

    参考：[#2737](https://www.sqlalchemy.org/trac/ticket/2737)

+   **[orm] [错误]**

    修复了由此票引起的从 0.7 版本中的一个回归，该回归使得在自引用的急切连接中递归溢出检查过于宽松，错过了一个特定情况，即子类配置了 lazy=”joined”或“subquery”，并且加载是针对基类的“with_polymorphic”。

    参考：[#2481](https://www.sqlalchemy.org/trac/ticket/2481)

+   **[orm] [错误]** 

    修复了从 0.7 版本中的一个回归，即`Session.begin_nested()`的上下文管理器功能在发生刷新错误时无法正确回滚事务，而是引发自己的异常，同时保留会话仍在等待回滚。

    参考：[#2718](https://www.sqlalchemy.org/trac/ticket/2718)

### orm 声明

+   **[orm] [声明] [功能]**

    现在可以通过名称引用 ORM 描述符，例如混合属性，在与`relationship()`中使用的字符串参数中，用于`order_by`，`primaryjoin`或类似操作，除了列绑定属性。

    参考：[#2761](https://www.sqlalchemy.org/trac/ticket/2761)

### 示例

+   **[示例] [错误]**

    修复了“版本控制”配方中的一个问题，即当存在反向引用时，一个多对一引用可能会为目标产生一个无意义的版本，即使它没有被更改。 补丁由 Matt Chisholm 提供。

+   **[示例] [错误]**

    修复了狗窝示例中的一个小 bug，在生成 SQL 缓存键时，没有像 `Query` 通常做的那样将去重标签应用到语句上的问题。

### engine

+   **[engine] [bug]**

    修复了一个 bug，在重新生成池时，各种 `Pool` 实现中的 `reset_on_return` 参数不会传播的问题。感谢 Eevee。

+   **[engine] [bug] [sybase]**

    修复了一个 bug，在某些情况下（例如 Sybase 方言），用于检测传递给 `create_engine()` 的正确 kwargs 的例程会失败的问题。

    参考：[#2732](https://www.sqlalchemy.org/trac/ticket/2732)

### sql

+   **[sql] [feature]**

    为 `TypeDecorator` 提供了一个新属性，名为 `TypeDecorator.coerce_to_is_types`，使得更容易控制使用 `==` 或 `!=` 与 `None` 和布尔类型进行比较时产生 `IS` 表达式或带有绑定参数的普通等式表达式的方式。

    参考：[#2734](https://www.sqlalchemy.org/trac/ticket/2734), [#2744](https://www.sqlalchemy.org/trac/ticket/2744)

+   **[sql] [bug]**

    对 `Select` 构造的相关行为进行了多次修复，这些修复首次出现在 0.8.0 中：

    +   为了满足应该将 FROM 条目向外相关到封装另一个 SELECT 的 SELECT，然后再封装此 SELECT 的用例，现在当通过 `Select.correlate()` 建立了显式相关性时，相关性会在多个级别上进行操作，前提是目标选择在由 WHERE/ORDER BY/columns 子句包含的链中的某个位置，而不仅仅是嵌套 FROM 子句。这使得 `Select.correlate()` 再次表现得更加与 0.7 兼容，同时仍保持新的“智能”相关性。

    +   当未使用显式相关性时，通常的“隐式”相关性限制其行为仅限于直接封闭的 SELECT，以最大限度地提高与 0.7 应用的兼容性，并且在这种情况下还防止了跨嵌套 FROM 的相关性，从而保持了与 0.8.0/0.8.1 的兼容性。

    +   `Select.correlate_except()`方法未在所有情况下阻止给定的 FROM 子句进行关联，并且还会导致 FROM 子句被错误地完全省略（更像是 0.7 的行为），这已经修复。

    +   调用 select.correlate_except(None)将使所有 FROM 子句进入关联，这是预期的行为。

    参考：[#2668](https://www.sqlalchemy.org/trac/ticket/2668), [#2746](https://www.sqlalchemy.org/trac/ticket/2746)

+   **[sql] [bug]**

    修复了一个 bug，即将一个表“A”的 select()与到表“B”的多个外键路径连接时，连接到表“B”会失败，而不会产生“模糊的连接条件”错误，如果直接将表“A”连接到“B”会报告错误；相反，它会产生具有多个条件的连接条件。

    参考：[#2738](https://www.sqlalchemy.org/trac/ticket/2738)

+   **[sql] [bug] [reflection]**

    修复了一个 bug，即在远程模式下使用`MetaData.reflect()`跨越一个本地模式和一个远程模式时，如果两个模式都有相同名称的表，则可能产生错误的结果。

    参考：[#2728](https://www.sqlalchemy.org/trac/ticket/2728)

+   **[sql] [bug]**

    从基础`ColumnOperators`类中删除了“未实现”的`__iter__()`调用，尽管这在 0.8.0 中引入是为了防止在自定义运算符上实现`__getitem__()`方法并在该对象上错误调用`list()`时出现无限、内存增长的循环，但这导致列元素报告它们实际上是可迭代类型，然后在尝试迭代时抛出错误。在这里没有真正的办法同时实现两者，因此我们坚持使用 Python 最佳实践。在自定义运算符上实现`__getitem__()`时要小心！

    参考：[#2726](https://www.sqlalchemy.org/trac/ticket/2726)

+   **[sql] [bug] [mssql]**

    从这个票据中导致不支持的关键字“true”被呈现，添加逻辑将其转换为 SQL 服务器的 1/0。

    参考：[#2682](https://www.sqlalchemy.org/trac/ticket/2682)

### postgresql

+   **[postgresql] [feature]**

    添加了对 PostgreSQL 9.2 范围类型的支持。目前，不提供类型转换，因此目前直接使用字符串或 psycopg2 2.5 范围扩展类型。补丁由 Chris Withers 提供。

+   **[postgresql] [feature]**

    当使用 psycopg2 DBAPI 时，添加了对“AUTOCOMMIT”隔离的支持。该关键字可通过`isolation_level`执行选项使用。补丁由 Roman Podolyaka 提供。

    参考：[#2072](https://www.sqlalchemy.org/trac/ticket/2072)

+   **[postgresql] [bug]**

    在 PostgreSQL 方言上，`extract()` 的行为已经简化，不再将硬编码的 `::timestamp` 或类似的转换注入到给定的表达式中，因为这会干扰到像时区感知的日期时间这样的类型，但是对于现代版本的 psycopg2 来说，这似乎完全不必要。

    参考：[#2740](https://www.sqlalchemy.org/trac/ticket/2740)

+   **[postgresql] [bug]**

    修复了 HSTORE 类型中的一个错误，即包含反斜杠引号的键/值在使用“非本地”（即非-psycopg2）方式转换 HSTORE 数据时无法正确转义。补丁由 Ryan Kelly 提供。

    参考：[#2766](https://www.sqlalchemy.org/trac/ticket/2766)

+   **[postgresql] [bug]**

    修复了在多列 PostgreSQL 索引中列的顺序反映错误顺序的错误。感谢 Roman Podolyaka。

    参考：[#2767](https://www.sqlalchemy.org/trac/ticket/2767)

+   **[postgresql] [bug]**

    修复了 HSTORE 类型以正确地对 Unicode 进行编码/解码。这总是开启的，因为 hstore 是一种文本类型，并且与使用 Python 3 时 psycopg2 的行为匹配。感谢 Dmitry Mugtasimov。

    参考：[#2735](https://www.sqlalchemy.org/trac/ticket/2735)

### mysql

+   **[mysql] [feature]**

    在 `Index` 中使用的 `mysql_length` 参数现在可以作为列名/长度的字典传递，用于复合索引。非常感谢 Roman Podolyaka 提供的补丁。

    参考：[#2704](https://www.sqlalchemy.org/trac/ticket/2704)

+   **[mysql] [bug]**

    修复了在使用多表 UPDATE 时出现的一个错误，其中一个辅助表是带有自己绑定参数的 SELECT，当使用 MySQL 的特殊语法时，绑定参数的位置与语句本身相反。

    参考：[#2768](https://www.sqlalchemy.org/trac/ticket/2768)

+   **[mysql] [bug]**

    在 `mysql+gaerdbms` 方言中添加了另一个条件来检测所谓的“开发”模式，在这种情况下，我们应该使用 `rdbms_mysqldb` DBAPI。补丁由 Brett Slatkin 提供。

    参考：[#2715](https://www.sqlalchemy.org/trac/ticket/2715)

+   **[mysql] [bug]**

    在 MySQL 方言上，`ForeignKey` 和 `ForeignKeyConstraint` 上的 `deferrable` 关键字参数将不会渲染 `DEFERRABLE` 关键字。很长一段时间以来，我们一直保留这个功能，因为非延迟外键的行为与延迟外键的行为有很大不同，但是一些环境只是在 MySQL 上禁用了 FK，所以我们在这里的态度会更加开放。

    参考：[#2721](https://www.sqlalchemy.org/trac/ticket/2721)

+   **[mysql] [bug]**

    更新了 mysqlconnector 方言，根据异常中发送的明显字符串消息检查断开连接；已针对 mysqlconnector 1.0.9 进行了测试。

### sqlite

+   **[sqlite] [bug]**

    将`sqlalchemy.types.BIGINT`添加到可以由 SQLite 方言反射的类型名称列表中；感谢 Russell Stuart。

    参考：[#2764](https://www.sqlalchemy.org/trac/ticket/2764)

### mssql

+   **[mssql] [bug]**

    在 SQL Server 2000 上查询信息模式时，删除了在 0.8.1 中添加的 CAST 调用，以帮助处理驱动程序问题，显然在 2000 上不兼容。CAST 保留在 SQL Server 2005 及更高版本中。

    参考：[#2747](https://www.sqlalchemy.org/trac/ticket/2747)

### 杂项

+   **[feature] [firebird]**

    向 kinterbasdb 和 fdb 方言添加了新标志`retaining=True`。这控制了发送到 DBAPI 连接的`commit()`和`rollback()`方法的`retaining`标志的值。由于历史原因，此标志在 0.8.2 中默认为`True`，但在 0.9.0b1 中此标志默认为`False`。

    参考：[#2763](https://www.sqlalchemy.org/trac/ticket/2763)

+   **[bug] [firebird]**

    修复了在反射 Firebird 类型 LONG 和 INT64 时的类型查找，使得 LONG 被视为 INTEGER，INT64 被视为 BIGINT，���非类型具有“精度”，在这种情况下，它被视为 NUMERIC。补丁感谢 Russell Stuart。

    参考：[#2757](https://www.sqlalchemy.org/trac/ticket/2757)

+   **[bug] [ext]**

    修复了一个 bug，当使用函数而不是类设置复合类型时，当尝试检查该列是否为`MutableComposite`时，可变扩展会出错（它不是）。感谢 asldevi。

+   **[requirements]**

    现在需要 Python 的[mock](https://pypi.org/project/mock)库才能运行单元测试套件。虽然作为 Python 3.3 的一部分已包含在标准库中，但以前的 Python 安装需要安装此库才能运行单元测试或使用`sqlalchemy.testing`包进行外部方言。

## 0.8.1

发布日期：2013 年 4 月 27 日

### orm

+   **[orm] [feature]**

    添加了一个方便的方法到 Query，将查询转换为形式为`EXISTS (SELECT 1 FROM ... WHERE ...)`的 EXISTS 子查询。

    参考：[#2673](https://www.sqlalchemy.org/trac/ticket/2673)

+   **[orm] [bug]**

    修复了一个 bug，当查询形式为：`query(SubClass).options(subqueryload(Baseclass.attrname))`，其中`SubClass`是`BaseClass`的联接继承时，会导致在属性加载时未能应用子查询中的`JOIN`，从而产生笛卡尔积。填充的结果仍然倾向于是正确的，因为额外的行只是被忽略，所以这个问题可能会在其他方面正常工作的应用程序中表现为性能下降。

    此更改也**回溯**到：0.7.11

    参考：[#2699](https://www.sqlalchemy.org/trac/ticket/2699)

+   **[orm] [bug]**

    修复了工作单元中的一个 bug，即如果两个表之间没有设置 ForeignKey 约束，一个 joined-inheritance 子类可能会在父表之前插入“sub”表的行。

    此更改也被**回溯**到：0.7.11

    参考：[#2689](https://www.sqlalchemy.org/trac/ticket/2689)

+   **[orm] [bug]**

    修复了`sqlalchemy.ext.serializer`扩展的问题，包括从 pickler 传递的“id”被转换为字符串以防止在 Py3K 上解析字节，以及`relationship()`和`orm.join()`构造现在被正确序列化。

    参考：[#2698](https://www.sqlalchemy.org/trac/ticket/2698)

+   **[orm] [bug]**

    对 query.join()的内部工作方式进行了显著改进，使得如何进行连接的决策变得极为简化。现在新的测试用例通过了，例如从已经复杂的涉及继承的连接系列的中间延伸多个连接。从深度嵌套的子查询结构进行连接仍然很复杂且不是没有注意事项的，但通过这些改进，边缘情况希望被推到更远的边缘。

    参考：[#2714](https://www.sqlalchemy.org/trac/ticket/2714)

+   **[orm] [bug]**

    添加了一个条件到 ORM 映射对象的反序列化过程中，这样如果对象被 pickled 时丢失了对对象的引用，我们就不会错误地尝试设置 _sa_instance_state - 修复了一个 NoneType 错误。

+   **[orm] [bug]**

    修复了 uselist=False 的多对多关系无法删除关联行并在标量属性设置为 None 时引发错误的 bug。这是由于为[#2229](https://www.sqlalchemy.org/trac/ticket/2229)引入的变更导致的回归。

    参考：[#2710](https://www.sqlalchemy.org/trac/ticket/2710)

+   **[orm] [bug]**

    改进了实例管理关于在 Session 中创建强引用的行为；如果对象处于瞬态状态或进入分离状态，对象将不再创建内部引用循环 - 只有当对象附加到 Session 时才创建强引用，并在对象分离时删除。这使得对象具有 __del__()方法更安全，尽管这并不被推荐，因为具有反向引用的关系也会产生循环。当映射了一个具有 __del__()方法的类时，会添加一个警告。

    参考：[#2708](https://www.sqlalchemy.org/trac/ticket/2708)

+   **[orm] [bug]**

    修复了一个 bug，当刷新一个继承映射类时，ORM 会运行错误类型的查询，其中超类被映射到非 Table 对象，比如自定义 join()或 select()，运行一个假设映射到单独 Table-per-class 的层次结构的查询。

    参考：[#2697](https://www.sqlalchemy.org/trac/ticket/2697)

+   **[orm] [bug]**

    修复了在对象初始化之前就能正常工作的 mapper 属性构造函数的 __repr__()，以便最近的 Sphinx 版本可以读取它们。

### orm 声明式

+   **[orm] [declarative] [bug]**

    修复了间接回归有关`has_inherited_table()`的问题，因为它考虑了当前类的`__table__`，所以对调用时敏感。这也是 0.7 的行为，但在 0.7 中，事情往往会在`__mapper_args__()`等事件中“解决”。`has_inherited_table()`现在只考虑超类，因此无论何时调用它，都应返回关于当前类的相同答案（显然假设超类的状态）。

    参考：[#2656](https://www.sqlalchemy.org/trac/ticket/2656)

### 示例

+   **[examples] [bug]**

    修复了缓存示例中长期存在的一个 bug，其中在计算缓存键时不会考虑 limit/offset 参数值。_key_from_query()函数已简化，直接从最终编译的语句中工作，以便获取完整的语句以及完全处理的参数列表。

### sql

+   **[sql] [feature]**

    放宽了传递给 Table()的特定于方言的参数名称的检查；因为我们希望支持外部方言，也希望支持未安装某个特定方言的参数，所以现在只检查参数的格式，而不再在 sqlalchemy.dialects 中查找该方言。

+   **[sql] [bug] [mysql]**

    完全实现了 IS 和 IS NOT 运算符与 True/False 常量相关。像`col.is_(True)`这样的表达式现在将在目标平台上呈现`col IS true`，而不是将 True/False 常量转换为整数绑定参数。这允许`is_()`运算符在给定 True/False 常量时在 MySQL 上工作。

    参考：[#2682](https://www.sqlalchemy.org/trac/ticket/2682)

+   **[sql] [bug]**

    对 select()对象在使用 apply_labels()时生成带标签列的方式进行了重大修复；这种模式生成一个 SELECT，其中每列都标记为<tablename>_<columnname>，以消除多个表选择的列名冲突。修复的是，如果两个标签与表名组合时发生冲突，即“foo.bar_id”和“foo_bar.id”，则将对其中一个重复项应用匿名别名。这允许 ORM 独立处理两个列；以前，0.7 在某些情况下会默默地为“重复项”发出第二个 SELECT，并且在 0.8 中会发出模棱两可的列错误。应用于 select()的.c.集合的“键”也将被去重，因此对于指定了 use_labels 的任何 select()，将不再为“被替换的列”警告发出，尽管重复键将被赋予一个通常不友好的匿名标签。

    参考：[#2702](https://www.sqlalchemy.org/trac/ticket/2702)

+   **[sql] [bug]**

    修复了一个 bug，即在连接对象已经关闭后，如果错误在断开连接检测时被引发，会引发属性错误。

    参考：[#2691](https://www.sqlalchemy.org/trac/ticket/2691)

+   **[sql] [bug]**

    重新设计了在重新引发之前发出 rollback()的内部异常引发，以便在进入 rollback 之前保留来自 sys.exc_info()的堆栈跟踪。这样，在使用协程框架时，即使在 rollback 函数返回之前切换上下文，也能保留堆栈跟踪。

    参考：[#2703](https://www.sqlalchemy.org/trac/ticket/2703)

+   **[sql] [bug] [postgresql]**

    _Binary 基本类型现在在 Python 3 上通过 bytes()可调用函数转换值；特别是 psycopg2 2.5 与 Python 3.3 似乎现在返回“memoryview”类型，因此在返回之前将其转换为 bytes。

+   **[sql] [bug]**

    改进了 Connection 自动失效处理。如果发生非断开连接错误，但在错误处理中导致延迟断开连接错误（在 MySQL 中发生），则会检测到断开连接条件。Connection 现在也可以在无效状态下关闭，这意味着在下一次使用时会引发“closed”，此外，“close with result”功能将在错误处理例程中的自动回滚失败时工作，无论条件是断开连接还是其他情况。

    参考：[#2695](https://www.sqlalchemy.org/trac/ticket/2695)

+   **[sql] [bug]**

    修复了一个 bug，即 DBAPI 可能会返回“0”作为 cursor.lastrowid，但与`ResultProxy.inserted_primary_key`结合使用时无法正常工作。

### postgresql

+   **[postgresql] [bug]**

    打开了使用 psycopg2/libpq 检查“断开连接”的检查，以检查完整异常层次结构中的所有各种“断开连接”消息。具体来说，“意外关闭连接”消息现在至少在三种不同的异常类型中被看到。感谢 Eli Collins。

    参考：[#2712](https://www.sqlalchemy.org/trac/ticket/2712)

+   **[postgresql] [bug]**

    PostgreSQL ARRAY 类型的运算符支持输入类型为集合、生成器等，即使未指定维度，也会将给定的可迭代对象无条件地转换为集合。

    参考：[#2681](https://www.sqlalchemy.org/trac/ticket/2681)

+   **[postgresql] [bug]**

    添加了缺失的 HSTORE 类型到 postgresql 类型名称中，���便可以反射该类型。

    参考：[#2680](https://www.sqlalchemy.org/trac/ticket/2680)

### mysql

+   **[mysql] [bug]**

    修复以支持最新的 cymysql DBAPI，感谢 Hajime Nakagami。

+   **[mysql] [bug]**

    改进了 Python 3 上 pymysql 方言的操作，包括一些重要的解码/字节步骤。由于驱动程序问题，BLOB 类型仍然存在问题。感谢 Ben Trofatter。

    参考：[#2663](https://www.sqlalchemy.org/trac/ticket/2663)

+   **[mysql] [bug]**

    更新了一个正则表达式以正确提取 Google App Engine v1.7.5 及更新版本的错误代码。感谢丹·林格。

### mssql

+   **[mssql] [bug]**

    作为解决 pyodbc + mssql 所需的一系列修复的一部分，现在在所有信息模式查询的绑定参数中为表名和模式名添加了 CAST 到 NVARCHAR(max)，以避免 NVARCHAR 与 NTEXT 的比较问题，这似乎在某些情况下被 ODBC 驱动程序拒绝，例如 FreeTDS (0.91 only?) 加上传递的 Unicode 绑定参数。这个问题似乎特定于 SQL Server 信息模式表，并且这个解决方法对于那些首次不存在问题的情况是无害的。

    参考：[#2355](https://www.sqlalchemy.org/trac/ticket/2355)

+   **[mssql] [bug]**

    添加了对于 pymssql 方言的额外“断开连接”消息的支持。感谢约翰·安德森。

+   **[mssql] [bug]**

    修复了关于“二进制”类型和 pymssql 的 Py3K bug。感谢马克·阿布拉莫维茨。

    参考：[#2683](https://www.sqlalchemy.org/trac/ticket/2683)

### orm

+   **[orm] [feature]**

    添加了一个方便的方法到 Query，将查询转换为 EXISTS 子查询，形式为 `EXISTS (SELECT 1 FROM ... WHERE ...)`。

    参考：[#2673](https://www.sqlalchemy.org/trac/ticket/2673)

+   **[orm] [bug]**

    修复了一个 bug，当查询形式为：`query(SubClass).options(subqueryload(Baseclass.attrname))`，其中 `SubClass` 是 `BaseClass` 的 joined inh 时，会导致属性加载中的子查询内部未能应用 `JOIN`，从而产生笛卡尔积。生成的结果仍然倾向于是正确的，因为额外的行被忽略掉了，所以这个问题可能存在于其他方面正常工作的应用程序中的性能下降。

    这个更改也 **回溯** 到：0.7.11

    参考：[#2699](https://www.sqlalchemy.org/trac/ticket/2699)

+   **[orm] [bug]**

    修复了一个单元操作中的 bug，即当一个 joined-inheritance 子类可以在父表之前插入“子”表的行时，如果两个表之间没有设置外键约束。

    这个更改也 **回溯** 到：0.7.11

    参考：[#2689](https://www.sqlalchemy.org/trac/ticket/2689)

+   **[orm] [bug]**

    对于 `sqlalchemy.ext.serializer` 扩展进行了修复，包括将来自 pickler 的“id”转换为字符串，以防止在 Py3K 上解析字节，以及现在正确地序列化 `relationship()` 和 `orm.join()` 结构。

    参考：[#2698](https://www.sqlalchemy.org/trac/ticket/2698)

+   **[orm] [bug]**

    对于 query.join() 的内部工作进行了显著改进，决定如何连接的过程已经大大简化。新的测试案例现在通过了，例如从已经复杂的继承关系和多个连接扩展的中间部分进行连接。从深度嵌套的子查询结构进行连接仍然很复杂，并且不是没有注意事项，但是通过这些改进，边缘情况希望能够进一步推迟。

    参考：[#2714](https://www.sqlalchemy.org/trac/ticket/2714)

+   **[orm] [bug]**

    在 ORM 映射对象的反序列化过程中添加了一个条件，这样如果对象在序列化时丢失引用，我们就不会错误地尝试设置 _sa_instance_state——修复了 NoneType 错误。

+   **[orm] [bug]**

    修复了当多对多关系中 uselist=False 时删除关联行失败的 bug，如果标量属性设置为 None，则会引发错误。这是由于对 [#2229](https://www.sqlalchemy.org/trac/ticket/2229) 的更改引入的退化。

    参考：[#2710](https://www.sqlalchemy.org/trac/ticket/2710)

+   **[orm] [bug]**

    改进了关于 Session 中强引用创建的实例管理行为；如果对象处于临时状态或进入脱离状态，则不再创建内部引用循环——只有在对象附加到会话时才创建强引用，并在对象脱离时移除。即使不推荐这样做，这样做会使得对象拥有 __del__() 方法更安全，因为具有反向引用的关系也会产生循环。当映射了具有 __del__() 方法的类时，添加了警告。

    参考：[#2708](https://www.sqlalchemy.org/trac/ticket/2708)

+   **[orm] [bug]**

    修复了一个 bug，在刷新继承映射类时 ORM 会运行错误类型的查询，其中超类映射到非 Table 对象，如自定义 join() 或 select()，运行一个假设映射到单独 Table-per-class 的层次结构的查询。

    参考：[#2697](https://www.sqlalchemy.org/trac/ticket/2697)

+   **[orm] [bug]**

    修复了对映射器属性构造函数的 __repr__() 方法，在对象初始化之前就可以工作，这样最近的 Sphinx 版本就可以读取它们了。

### ORM 声明式

+   **[orm] [declarative] [bug]**

    修复了关于 `has_inherited_table()` 的间接退化，因为它考虑当前类的 `__table__`，所以对其调用时敏感。这也是 0.7 版本的行为，但在 0.7 版本中，事情往往会在 `__mapper_args__()` 等事件中“解决”。`has_inherited_table()` 现在只考虑超类，因此无论何时调用它，都应该返回相同的关于当前类的答案（显然假设超类的状态相同）。

    参考：[#2656](https://www.sqlalchemy.org/trac/ticket/2656)

### 示例

+   **[examples] [bug]**

    修复了缓存示例中的一个长期存在的 bug，其中 limit/offset 参数值在计算缓存键时不会被考虑。_key_from_query() 函数已简化为直接从最终编译的语句中工作，以便获取完整的语句以及完全处理过的参数列表。

### sql

+   **[sql] [feature]**

    放宽了传递给 Table() 的特定于方言的参数名称的检查；由于我们希望支持外部方言，并且还希望支持未安装某个特定方言的参数，因此现在仅检查参数的格式，而不是在 sqlalchemy.dialects 中查找该方言。

+   **[sql] [bug] [mysql]**

    完全实现了对 True/False 常量的 IS 和 IS NOT 运算符。现在像 `col.is_(True)` 这样的表达式将在目标平台上呈现为 `col IS true`，而不是将 True/False 常量转换为整数绑定参数。这使得 `is_()` 运算符在给定 True/False 常量时能够在 MySQL 上工作。

    参考：[#2682](https://www.sqlalchemy.org/trac/ticket/2682)

+   **[sql] [bug]**

    当 apply_labels() 被使用时，修复了 select() 对象生成标记列的方式；此模式生成一个 SELECT，其中每个列都被标记为 <tablename>_<columnname>，以消除多表选择中的列名冲突。修复的问题是，如果两个标签与表名组合时发生冲突，即“foo.bar_id” 和 “foo_bar.id”，则将对其中一个 dupes 应用匿名别名。这允许 ORM 独立处理两个列；在此之前，0.7 在某些情况下会静默地为“duped”列发出第二个 SELECT，并且在 0.8 中会发出一个模糊的列错误。对 select() 的 .c. 集合应用的 “keys” 也将被去重，因此对于指定了 use_labels 的任何 select()，将不再发出“正在替换的列”警告，尽管 dupes 键将被赋予一个不通常用户友好的匿名标签。

    参考：[#2702](https://www.sqlalchemy.org/trac/ticket/2702)

+   **[sql] [bug]**

    修复了在错误断开连接时，如果错误是在 Connection 对象已经关闭之后引发的，会引发属性错误的错误。

    参考：[#2691](https://www.sqlalchemy.org/trac/ticket/2691)

+   **[sql] [bug]**

    重新设计了在重新引发之前发出回滚() 的内部异常引发，以便从 sys.exc_info() 中保留堆栈跟踪，然后进入回滚。这样，当使用可能在回滚函数返回之前切换上下文的协程框架时，将保留堆栈跟踪。

    参考：[#2703](https://www.sqlalchemy.org/trac/ticket/2703)

+   **[sql] [bug] [postgresql]**

    当在 Python 3 上运行时，_Binary 基础类型现在通过 bytes() 可调用进行值转换；特别是 psycopg2 2.5 与 Python 3.3 似乎现在返回“memoryview”类型，因此在返回之前将其转换为字节。

+   **[sql] [bug]**

    改进了 Connection 自动失效处理。如果发生非断开连接错误，但在错误处理中导致延迟断开连接错误（在 MySQL 中发生），则会检测到断开连接条件。Connection 现在也可以在无效状态下关闭，这意味着在下一次使用时会引发“closed”，此外，“close with result”功能将在错误处理例程中的自动回滚失败时工作，无论条件是断开连接还是其他情况。

    参考：[#2695](https://www.sqlalchemy.org/trac/ticket/2695)

+   **[sql] [bug]**

    修复了一个 bug，即一个可以为 cursor.lastrowid 返回“0”的 DBAPI 与 `ResultProxy.inserted_primary_key` 结合使用时将无法正确运行。

### postgresql

+   **[postgresql] [bug]**

    打开了对 psycopg2/libpq 中“断开连接”检查的检查，以检查完整异常层次结构中的所有不同“断开连接”消息。具体来说，“意外关闭连接”消息现在至少在三种不同的异常类型中被看到。由 Eli Collins 提供。

    参考：[#2712](https://www.sqlalchemy.org/trac/ticket/2712)

+   **[postgresql] [bug]**

    PostgreSQL ARRAY 类型的操作符支持集合、生成器等输入类型，即使未指定维度，也会无条件地将给定的可迭代对象转换为集合。

    参考：[#2681](https://www.sqlalchemy.org/trac/ticket/2681)

+   **[postgresql] [bug]**

    添加了缺失的 HSTORE 类型到 postgresql 类型名称中，以便可以反射该类型。

    参考：[#2680](https://www.sqlalchemy.org/trac/ticket/2680)

### mysql

+   **[mysql] [bug]**

    修复以支持最新的 cymysql DBAPI，由 Hajime Nakagami 提供。

+   **[mysql] [bug]**

    对 Python 3 上 pymysql 方言的操作进行了改进，包括一些重要的解码/字节步骤。由 Ben Trofatter 提供。问题仍然存在于 BLOB 类型，由于驱动程序问题。

    参考：[#2663](https://www.sqlalchemy.org/trac/ticket/2663)

+   **[mysql] [bug]**

    更新了一个正则表达式，以正确提取谷歌应用引擎 v1.7.5 及更新版本的错误代码。感谢 Dan Ring。

### mssql

+   **[mssql] [bug]**

    作为一系列修复的一部分，需要为 pyodbc+ mssql 添加 CAST 到 NVARCHAR(max)，以避免在所有信息模式查询的表名和模式名的绑定参数中出现将 NVARCHAR 与 NTEXT 进行比较的问题，这似乎在某些情况下被 ODBC 驱动程序拒绝，例如 FreeTDS（仅限 0.91？）加上传递的 Unicode 绑定参数。该问题似乎特定于 SQL Server 信息模式表，并且解决方法对于那些问题本来就不存在的情况是无害的。

    参考：[#2355](https://www.sqlalchemy.org/trac/ticket/2355)

+   **[mssql] [bug]**

    为 pymssql 方言添加了对额外“断开连接”消息的支持��由 John Anderson 提供。

+   **[mssql] [bug]**

    修复了关于“binary”类型和 pymssql 的 Py3K bug。由 Marc Abramowitz 提供。

    参考：[#2683](https://www.sqlalchemy.org/trac/ticket/2683)

## 0.8.0

发布日期：2013 年 3 月 9 日

注意

0.8.0 中存在一些新的行为变化，而在 0.8.0b2 中不存在。它们在迁移文档中如下所示：

+   将“待定”对象视为“孤立”对象的考虑更加积极

+   create_all() 和 drop_all() 现在将尊重空列表

+   相关性现在始终是上下文特定的

### orm

+   **[orm] [feature]**

    添加了有意义的`QueryableAttribute.info`属性，它代理到直接存在的`Column`对象的`.info`属性，否则代理到`MapperProperty`。完整的行为已记录并通过测试确保稳定。

    参考：[#2675](https://www.sqlalchemy.org/trac/ticket/2675)

+   **[orm] [feature]**

    可以在`relationship()`构造已经创建后设置/更改“cascade”属性。这不是正常使用的模式，但我们喜欢在教程中为了演示目的更改设置。

+   **[orm] [feature]**

    添加了新的辅助函数`was_deleted()`，如果给定对象是`Session.delete()`操作的主题，则返回 True。

    参考：[#2658](https://www.sqlalchemy.org/trac/ticket/2658)

+   **[orm] [feature]**

    扩展了运行时检查 API 系统，以便检索与 ORM 或其扩展相关的所有 Python 描述符。这满足了能够检查所有`QueryableAttribute`描述符的常见请求，以及扩展类型，如`hybrid_property`和`AssociationProxy`。请参阅`Mapper.all_orm_descriptors`。

+   **[orm] [bug]**

    在映射器配置期间改进了对现有反向引用名称冲突的检查；现在将在超类和子类上测试名称冲突，除了当前映射器之外，因为这些冲突同样会导致问题。这是 0.8 的新功能，但请参见下面关于 0.7.11 中也会触发的警告。

    参考：[#2674](https://www.sqlalchemy.org/trac/ticket/2674)

+   **[orm] [bug]**

    改进了在检测到“反向引用循环”时发出的错误消息，即当属性事件触发两个其他属性之间的双向赋值时，没有结束。这种情况不仅发生在分配错误类型的对象时，还发生在属性被错误配置为反向引用到现有反向引用对时。也适用于 0.7.11。

    参考：[#2674](https://www.sqlalchemy.org/trac/ticket/2674)

+   **[orm] [bug]**

    当将 MapperProperty 分配给替换现有属性的映射器时，如果涉及的属性不是简单的基于列的属性，则会发出警告。替换关系属性很少（甚至从未？）是预期的，通常指的是映射器配置错误。也适用于 0.7.11。

    参考：[#2674](https://www.sqlalchemy.org/trac/ticket/2674)

+   **[orm] [bug]**

    如果事件处理程序在 after_commit()处理程序中尝试在没有进行中的有效事务的情况下在会话中发出 SQL，则会发出清晰的错误消息。

    参考：[#2662](https://www.sqlalchemy.org/trac/ticket/2662)

+   **[orm] [bug]**

    在级联自然主键更新过程中检测到主键更改将成功，即使该键是复合键，只有部分属性发生更改。

    参考：[#2665](https://www.sqlalchemy.org/trac/ticket/2665)

+   **[orm] [bug]**

    从会话中删除的对象在事务提交后将完全与该会话解除关联，也就是`object_session()`函数将返回 None。

    参考：[#2658](https://www.sqlalchemy.org/trac/ticket/2658)

+   **[orm] [bug]**

    修复了`Query.yield_per()`错误设置执行选项的错误，从而破坏了后续使用`Query.execution_options()`方法。感谢 Ryan Kelly。

    参考：[#2661](https://www.sqlalchemy.org/trac/ticket/2661)

+   **[orm] [bug]**

    修复了`between()`运算符的考虑，使其与新的关系本地/远程系统正确工作。

    参考：[#1768](https://www.sqlalchemy.org/trac/ticket/1768)

+   **[orm] [bug]**

    将待定对象视为“孤立”对象的考虑已经修改，以更接近持久对象的行为，即一旦与任何启用孤立的父对象解除关联，该对象就会从`Session`中清除。以前，只有在从所有启用孤立的父对象中解除关联时，待定对象才会被清除。新增了`legacy_is_orphan`标志到`Mapper`，它重新建立了传统行为。

    详细讨论此更改的变更说明和示例案例，请参阅将“待定”对象视为“孤立”对象的考虑更加激进。

    参考：[#2655](https://www.sqlalchemy.org/trac/ticket/2655)

+   **[orm] [bug]**

    修复了（很可能从未使用过的）“@collection.link”集合方法，每次将集合与映射对象关联或解除关联时都会触发-装饰器未经测试或功能性。装饰器方法现在命名为`collection.linker()`，尽管名称“link”仍保留以确保向后兼容性。感谢 Luca Wehrstedt。

    参考：[#2653](https://www.sqlalchemy.org/trac/ticket/2653)

+   **[orm] [bug]**

    对生成自定义受控集合系统进行了一些修复，主要是现在@collection 装饰器的使用将遵循给定类的 __mro__，应用特定集合方法的子类最底层类的逻辑。以前，在对现有受控类进行子类化时，例如`MappedCollection`，无法预测自定义方法是否会正确解析。

    参考：[#2654](https://www.sqlalchemy.org/trac/ticket/2654)

+   **[orm] [bug]**

    修复了潜在的内存泄漏问题，可能会在创建任意数量的`sessionmaker`对象时发生。当 sessionmaker 创建的匿名子类被解除引用时，由于事件包中仍存在类级别的引用，该子类不会被垃圾回收。这个问题也适用于任何自定义系统，该系统与事件调度程序一起使用临时子类。也适用于 0.7.10 版本。

    参考：[#2650](https://www.sqlalchemy.org/trac/ticket/2650)

+   **[orm] [bug]**

    `Query.merge_result()`现在可以从外连接中加载可能为`None`的实体行而不会抛出错误。也适用于 0.7.10 版本。

    参考：[#2640](https://www.sqlalchemy.org/trac/ticket/2640)

+   **[orm] [bug]**

    对`relationship()`上的“dynamic”加载器进行修复，包括当禁用自动刷新时，反向引用将正常工作，历史事件在多次添加/删除相同对象的情况下更加准确。

    参考：[#2637](https://www.sqlalchemy.org/trac/ticket/2637)

+   **[orm] [removed]**

    已删除了使用与集合关联的`__instrumentation__`数据结构来生成自定义集合的未记录（希望未使用）系统，因为这是一个复杂且未经测试的功能，与装饰器方法基本重复。还对 orm.collections 模块进行了其他内部简化。

### examples

+   **[examples] [bug]**

    修复了示例/dogpile_caching 示例中的回归问题，这是由于[#2614](https://www.sqlalchemy.org/trac/ticket/2614)中的更改引起的。

### sql

+   **[sql] [feature]**

    向`Enum`及其基本`SchemaType`添加了一个新参数`inherit_schema`。当设置为`True`时，该类型将把其`schema`属性设置为其关联的`Table`的模式。这也会在进行`Table.tometadata()`操作时发生；在所有情况下，当`Table.tometadata()`发生时，`SchemaType`现在都会被复制，如果`inherit_schema=True`，则该类型将采用传递给该方法的新模式名称。在与 PostgreSQL 后端一起使用时，`schema`非常重要，因为该类型会导致`CREATE TYPE`语句。

    参考：[#2657](https://www.sqlalchemy.org/trac/ticket/2657)

+   **[sql] [feature]**

    `Index` 现在支持任意的 SQL 表达式和/或函数，除了直接列。常见的修饰符包括使用 `somecolumn.desc()` 来创建降序索引，以及 `func.lower(somecolumn)` 来创建不区分大小写的索引，具体取决于目标后端的功能。

    参考：[#695](https://www.sqlalchemy.org/trac/ticket/695)

+   **[sql] [bug]**

    改进了 SELECT 关联的行为，使得 `Select.correlate()` 和 `Select.correlate_except()` 方法，以及它们的 ORM 类似方法，在 FROM 子句仅在输出为合法 SQL 时才被修改；也就是说，如果关联的 SELECT 在 WHERE、columns 或 HAVING 子句的上下文中未被用于封闭的 SELECT 中，FROM 子句将保持不变。这两种方法现在只指定默认“自动关联”的条件，而不是绝对的 FROM 列表。

    参考：[#2668](https://www.sqlalchemy.org/trac/ticket/2668)

+   **[sql] [bug]**

    修复了关于列注释的 bug，特别是可能影响到新的 `remote()` 和 `local()` 注释函数的某些用法，当列在后续表达式中使用时，注释可能会丢失。

    参考：[#1768](https://www.sqlalchemy.org/trac/ticket/1768), [#2660](https://www.sqlalchemy.org/trac/ticket/2660)

+   **[sql] [bug]**

    `ColumnOperators.in_()` 操作符现在会将`None`值强制转换为`null()`。

    参考：[#2496](https://www.sqlalchemy.org/trac/ticket/2496)

+   **[sql] [bug]**

    修复了一个 bug，当一个 `Column` 同时具有外键和列的替代“.key”名称时，`Table.tometadata()` 会失败。也适用于 0.7.10 版本。

    参考：[#2643](https://www.sqlalchemy.org/trac/ticket/2643)

+   **[sql] [bug]**

    在不支持 RETURNING 的方言上尝试编译时，insert().returning() 会引发一个信息性的 CompileError。

    参考：[#2629](https://www.sqlalchemy.org/trac/ticket/2629)

+   **[sql] [bug]**

    调整了编译器用于识别需要传递的 INSERT/UPDATE 绑定参数的“REQUIRED”符号，以便在编写自定义绑定处理代码时更容易识别。

    参考：[#2648](https://www.sqlalchemy.org/trac/ticket/2648)

### schema

+   **[schema] [bug]**

    `MetaData.create_all()` 和 `MetaData.drop_all()` 现在将接受一个空列表作为不创建/删除任何项的指令，而不是忽略该集合。

    参考：[#2664](https://www.sqlalchemy.org/trac/ticket/2664)

### postgresql

+   **[postgresql] [feature]**

    增加了对 PostgreSQL 传统 SUBSTRING 函数语法的支持，当使用常规的 `func.substring()` 时，呈现为“SUBSTRING(x FROM y FOR z)”。感谢 Gunnlaugur Þór Briem。

    这个更改也被**回溯**到：0.7.11

    参考：[#2676](https://www.sqlalchemy.org/trac/ticket/2676)

+   **[postgresql] [feature]**

    添加了 `Comparator.any()` 和 `Comparator.all()` 方法，以及独立的表达式构造。非常感谢 Audrius Kažukauskas 在这里的出色工作。

+   **[postgresql] [bug]**

    修复了在 `array()` 构造中的 bug，当在 `insert()` 构造中使用它时，会产生关于 `self_group()` 方法中参数问题的错误。

### mysql

+   **[mysql] [feature]**

    添加了 CyMySQL 的新方言，感谢 Hajime Nakagami。

+   **[mysql] [feature]**

    GAE 方言现在接受 URL 中的用户名/密码参数，感谢 Owen Nelson。

+   **[mysql] [bug] [gae]**

    在 `gaerdbms` 方言中添加了一个条件导入，尝试导入 rdbms_apiproxy 和 rdbms_googleapi 以在开发和生产平台上工作。现在也支持 `instance` 属性。感谢 Sean Lynch。也在 0.7.10 版本中。

    参考：[#2649](https://www.sqlalchemy.org/trac/ticket/2649)

+   **[mysql] [bug]**

    如果无法从异常中提取错误代码，GAE 方言将不会在无匹配时失败；感谢 Owen Nelson。

### mssql

+   **[mssql] [feature]**

    在 `Index` 中添加了 `mssql_include` 和 `mssql_clustered` 选项，分别呈现 `INCLUDE` 和 `CLUSTERED` 关键字。感谢 Derek Harland。

+   **[mssql] [feature]**

    现在支持在非主键列上建立 `Sequence` 构造来支持 IDENTITY 列的 DDL。感谢 Derek Harland。

    参考：[#2644](https://www.sqlalchemy.org/trac/ticket/2644)

+   **[mssql] [bug]**

    在 mssql 信息模式中添加了一个 py3K 条件，解决了 Py3K 中反射的问题。也在 0.7.10 版本中。

    参考：[#2638](https://www.sqlalchemy.org/trac/ticket/2638)

+   **[mssql] [bug]**

    修复了一个回归问题，即字符类型 CHAR、NCHAR 等的“collation”参数停止工作，因为“collation”现在受到基本字符串类型的支持。MSSQL 方言中的 TEXT、NCHAR、CHAR、VARCHAR 类型现在是基本类型的同义词。

### oracle

+   **[oracle] [bug]**

    cx_oracle 方言将不再通过`encode()`运行绑定参数名称，因为这在 Python 3 上无效，并且阻止了 Python 3 上语句的正确功能。现在只有在`supports_unicode_binds`为 False 时才进行编码，当至少使用 cx_oracle 的版本 5 时，这不是 cx_oracle 的情况。

### 测试

+   **[测试] [错误]**

    修复了在一些 Linux 平台上无法在 test_execute 中工作的“logging”导入。也在 0.7.11 中。

    参考：[#2669](https://www.sqlalchemy.org/trac/ticket/2669)

### orm

+   **[orm] [功能]**

    添加了有意义的`QueryableAttribute.info`属性，它代理到直接存在的`Column`对象的`.info`属性，否则代理到`MapperProperty`。完整的行为已记录并通过测试确保保持稳定。

    参考：[#2675](https://www.sqlalchemy.org/trac/ticket/2675)

+   **[orm] [功能]**

    可以在已经构建的`relationship()`构造函数上设置/更改“cascade”属性。这不是正常使用的模式，但我们喜欢在教程中更改设置以进行演示。

+   **[orm] [功能]**

    添加了新的辅助函数`was_deleted()`，如果给定对象是`Session.delete()`操作的主题，则返回 True。

    参考：[#2658](https://www.sqlalchemy.org/trac/ticket/2658)

+   **[orm] [功能]**

    扩展了运行时检查 API 系统，以便检索与 ORM 或其扩展相关的所有 Python 描述符。这满足了能够检查所有`QueryableAttribute`描述符的常见请求，以及扩展类型，如`hybrid_property`和`AssociationProxy`。请参阅`Mapper.all_orm_descriptors`。

+   **[orm] [错误]**

    改进了在映射器配置期间检查现有反向引用名称冲突的方法；现在将在超类和子类上测试名称冲突，除了当前映射器外，因为这些冲突会造成同样的问题。这是 0.8 版本的新功能，但请参见下面关于 0.7.11 版本也会触发的警告。

    参考：[#2674](https://www.sqlalchemy.org/trac/ticket/2674)

+   **[orm] [bug]**

    当检测到“反向引用循环”时，改进了发出的错误消息，即当属性事件触发两个其他属性之间的双向赋值而没有结束时。这种情况不仅会在分配错误类型的对象时发生，还会在属性被错误配置为反向引用到现有反向引用对时发生。也适用于 0.7.11 版本。

    参考：[#2674](https://www.sqlalchemy.org/trac/ticket/2674)

+   **[orm] [bug]**

    当将 MapperProperty 分配给替换现有属性的映射器时，如果涉及的属性不是简单的基于列的属性，则会发出警告。替换关系属性很少（甚至从未？）是预期的，通常指的是映射器错误配置。也适用于 0.7.11 版本。

    参考：[#2674](https://www.sqlalchemy.org/trac/ticket/2674)

+   **[orm] [bug]**

    如果事件处理程序尝试在没有进行中的有效事务的情况下在`after_commit()`处理程序中发出 SQL，则会发出清晰的错误消息。

    参考：[#2662](https://www.sqlalchemy.org/trac/ticket/2662)

+   **[orm] [bug]**

    在级联自然主键更新过程中检测主键更改将成功，即使该主键是复合的，只有一些属性发生了变化。

    参考：[#2665](https://www.sqlalchemy.org/trac/ticket/2665)

+   **[orm] [bug]**

    从会话中删除的对象在事务提交后将完全与该会话解除关联，即`object_session()`函数将返回 None。

    参考：[#2658](https://www.sqlalchemy.org/trac/ticket/2658)

+   **[orm] [bug]**

    修复了`Query.yield_per()`设置执行选项不正确的错误，从而破坏了后续使用`Query.execution_options()`方法的情况。感谢 Ryan Kelly。

    参考：[#2661](https://www.sqlalchemy.org/trac/ticket/2661)

+   **[orm] [bug]**

    修复了`between()`运算符的考虑，使其与新的关系本地/远程系统正确工作。

    参考：[#1768](https://www.sqlalchemy.org/trac/ticket/1768)

+   **[orm] [bug]**

    将待处理对象视为“孤儿”的考虑已修改，以更接近持久对象的行为，即一旦它从任何启用孤儿的父级对象中取消关联，该对象就会从`Session`中删除。以前，只有当待处理对象从所有启用孤儿的父级对象中取消关联时，才会将其删除。在 `Mapper`中添加了新标志`legacy_is_orphan`，它重新建立了旧版本的行为。

    有关此更改的详细讨论，请参阅将“待处理”对象视为“孤儿”的考虑变得更加激进的更改说明和示例情况。

    参考：[#2655](https://www.sqlalchemy.org/trac/ticket/2655)

+   **[orm] [bug]**

    修复了（很可能从未使用过的）“@collection.link”集合方法，该方法在每次将集合与映射对象关联或取消关联时触发 - 未测试或功能性不强。装饰器方法现在命名为`collection.linker()`，尽管名称“link”仍保留以保持向后兼容性。感谢 Luca Wehrstedt。

    参考：[#2653](https://www.sqlalchemy.org/trac/ticket/2653)

+   **[orm] [bug]**

    对生成自定义检测集合系统进行了一些修复，主要是现在使用 @collection 装饰器将遵循给定类的 __mro__，应用于特定集合方法的子类版本的逻辑。以前，在对现有的检测类进行子类化时，例如`MappedCollection`，无法预测自定义方法是否会正确解析。

    参考：[#2654](https://www.sqlalchemy.org/trac/ticket/2654)

+   **[orm] [bug]**

    修复了可能会发生的内存泄漏问题，如果创建了任意数量的 `sessionmaker`对象，则会发生。由于事件包中保留了来自类级别的引用，当取消引用 sessionmaker 创建的匿名子类时，它不会被垃圾收集。此问题还适用于与事件调度程序结合使用临时子类的任何自定义系统。也适用于 0.7.10 版本。

    参考：[#2650](https://www.sqlalchemy.org/trac/ticket/2650)

+   **[orm] [bug]**

    `Query.merge_result()` 现在可以从外连接加载行，其中实体可能为`None`而不会抛出错误。也适用于 0.7.10 版本。

    参考：[#2640](https://www.sqlalchemy.org/trac/ticket/2640)

+   **[orm] [bug]**

    对`relationship()`上的“dynamic”加载器进行修复，包括当禁用自动刷新时，反向引用将正常工作，历史事件在同一对象多次添加/删除的情况下更加准确。

    参考：[#2637](https://www.sqlalchemy.org/trac/ticket/2637)

+   **[orm] [removed]**

    已删除了使用与集合关联的`__instrumentation__`数据结构来生成自定义集合的未记录（希望未使用）系统，因为这是一个复杂且未经测试的功能，与装饰器方法基本重复。还对 orm.collections 模块进行了其他内部简化。

### 示例

+   **[examples] [bug]**

    修复了示例/dogpile_caching 示例中的回归，这是由于[#2614](https://www.sqlalchemy.org/trac/ticket/2614)的更改引起的。

### sql

+   **[sql] [feature]**

    向`Enum`及其基类`SchemaType`添加了一个新参数`inherit_schema`。当设置为`True`时，该类型将设置其与之关联的`Table`的`schema`属性。这也会在进行`Table.tometadata()`操作时发生；在`Table.tometadata()`发生时，`SchemaType`现在在所有情况下都会被复制，并且如果`inherit_schema=True`，该类型将采用传递给该方法的新模式名称。在与 PostgreSQL 后端一起使用时，`schema`非常重要，因为该类型会导致`CREATE TYPE`语句。

    参考：[#2657](https://www.sqlalchemy.org/trac/ticket/2657)

+   **[sql] [feature]**

    `Index`现在支持任意的 SQL 表达式和/或函数，除了直接列。常见的修饰符包括使用`somecolumn.desc()`来创建降序索引和`func.lower(somecolumn)`来创建不区分大小写的索引，具体取决于目标后端的功能。

    参考：[#695](https://www.sqlalchemy.org/trac/ticket/695)

+   **[sql] [bug]**

    改进了 SELECT 关联的行为，使得 `Select.correlate()` 和 `Select.correlate_except()` 方法，以及它们的 ORM 类似方法，在上下文中仍将保留“自动关联”的行为，即仅在输出合法的 SQL 时修改 FROM 子句；也就是说，如果关联的 SELECT 在 WHERE、columns 或 HAVING 子句的上下文中未被用于封闭的 SELECT 中，FROM 子句将保持不变。这两种方法现在只指定默认“自动关联”的条件，而不是绝对的 FROM 列表。

    参考：[#2668](https://www.sqlalchemy.org/trac/ticket/2668)

+   **[sql] [bug]**

    修复了关于列注释的 bug，特别是可能影响新的 `remote()` 和 `local()` 注释函数的某些用法，其中在列在后续表达式中使用时可能会丢失注释。

    参考：[#1768](https://www.sqlalchemy.org/trac/ticket/1768), [#2660](https://www.sqlalchemy.org/trac/ticket/2660)

+   **[sql] [bug]**

    `ColumnOperators.in_()` 运算符现在会将 `None` 的值强制转换为 `null()`。

    参考：[#2496](https://www.sqlalchemy.org/trac/ticket/2496)

+   **[sql] [bug]**

    修复了一个 bug，关于 `Table.tometadata()` 如果一个 `Column` 同时具有外键和列的替代“.key”名称，将会失败。也适用于 0.7.10 版本。

    参考：[#2643](https://www.sqlalchemy.org/trac/ticket/2643)

+   **[sql] [bug]**

    `insert().returning()` 在不支持 RETURNING 的方言上尝试编译时会引发一个信息丰富的 CompileError。

    参考：[#2629](https://www.sqlalchemy.org/trac/ticket/2629)

+   **[sql] [bug]**

    调整了编译器用于识别需要传递的 INSERT/UPDATE 绑定参数的“REQUIRED”符号，使其在编写自定义绑定处理代码时更容易识别。

    参考：[#2648](https://www.sqlalchemy.org/trac/ticket/2648)

### schema

+   **[schema] [bug]**

    `MetaData.create_all()` 和 `MetaData.drop_all()` 现在将空列表作为不创建/删除任何项的指令，而不是忽略该集合。

    参考：[#2664](https://www.sqlalchemy.org/trac/ticket/2664)

### postgresql

+   **[postgresql] [功能]**

    添加了对 PostgreSQL 传统 SUBSTRING 函数语法的支持，当使用常规的 `func.substring()` 时，会呈现为“SUBSTRING(x FROM y FOR z)”。感谢 Gunnlaugur Þór Briem。

    这个更改也被**回溯**到：0.7.11

    参考：[#2676](https://www.sqlalchemy.org/trac/ticket/2676)

+   **[postgresql] [功能]**

    添加了 `Comparator.any()` 和 `Comparator.all()` 方法，以及独立的表达式构造。非常感谢 Audrius Kažukauskas 在这里的出色工作。

+   **[postgresql] [错误]**

    修复了在 `array()` 构造中的 bug，该 bug 在 `insert()` 构造中使用时会产生关于 `self_group()` 方法中参数问题的错误。

### mysql

+   **[mysql] [功能]**

    新增了 CyMySQL 的方言，感谢 Hajime Nakagami。

+   **[mysql] [功能]**

    GAE 方言现在接受 URL 中的用户名/密码参数，感谢 Owen Nelson。

+   **[mysql] [错误] [gae]**

    在 `gaerdbms` 方言中添加了一个条件导入，尝试导入 rdbms_apiproxy 而不是 rdbms_googleapi 以在开发和生产平台上工作。现在也支持 `instance` 属性。感谢 Sean Lynch。也在 0.7.10 版本中。

    参考：[#2649](https://www.sqlalchemy.org/trac/ticket/2649)

+   **[mysql] [错误]**

    如果 GAE 方言无法从异常抛出中提取错误代码，则不会因为 None 匹配而失败；感谢 Owen Nelson。

### mssql

+   **[mssql] [功能]**

    在`Index`中添加了 `mssql_include` 和 `mssql_clustered` 选项，分别呈现 `INCLUDE` 和 `CLUSTERED` 关键字。感谢 Derek Harland。

+   **[mssql] [功能]**

    现在支持在非主键列上为自增列设置 DDL，方法是在任何整数列上建立一个`Sequence`构造。感谢 Derek Harland。

    参考：[#2644](https://www.sqlalchemy.org/trac/ticket/2644)

+   **[mssql] [错误]**

    在 mssql 信息模式中添加了一个 py3K 条件，修复了 Py3K 中的反射问题。也在 0.7.10 版本中。

    参考：[#2638](https://www.sqlalchemy.org/trac/ticket/2638)

+   **[mssql] [错误]**

    修复了字符类型 CHAR、NCHAR 等的“collation”参数停止工作的回归问题，因为“collation”现在由基本字符串类型支持��MSSQL 方言中的 TEXT、NCHAR、CHAR、VARCHAR 类型现在是基本类型的同义词。

### oracle

+   **[oracle] [错误]**

    cx_oracle 方言将不再通过`encode()`运行绑定参数名称，因为这在 Python 3 上是无效的，并且阻止了 Python 3 上语句的正确功能。现在只有在`supports_unicode_binds`为 False 时才进行编码，当至少使用 cx_oracle 的版本 5 时，这种情况并不适用于 cx_oracle。

### 测试

+   **[测试] [错误]**

    修复了在一些 Linux 平台上无法在 test_execute 中工作的“logging”导入。也在 0.7.11 中。

    参考：[#2669](https://www.sqlalchemy.org/trac/ticket/2669)

## 0.8.0b2

发布日期：2012 年 12 月 14 日

### orm

+   **[orm] [功能]**

    添加了`KeyedTuple._asdict()`和`KeyedTuple._fields`到`KeyedTuple`类，以提供与 Python 标准库`collections.namedtuple()`一定程度的兼容性。

    参考：[#2601](https://www.sqlalchemy.org/trac/ticket/2601)

+   **[orm] [功能]**

    允许在定义关系的主要和次要连接时使用同义词。

+   **[orm] [错误]**

    `Query.select_from()`方法现在可以与`aliased()`构造一起使用，而不会干扰被选择的实体。基本上，像这样的语句：

    ```py
    ua = aliased(User)
    session.query(User.name).select_from(ua).join(User, User.name > ua.name)
    ```

    将保持 SELECT 中的列子句作为未命名的“user”指定；select_from 仅在 FROM 子句中发生：

    ```py
    SELECT  users.name  AS  users_name  FROM  users  AS  users_1
    JOIN  users  ON  users.name  <  users_1.name
    ```

    请注意，这种行为与原始、较旧的`Query.select_from()`用例形成对比，即在不同的可选择性方面重新表述映射实体：

    ```py
    session.query(User.name).select_from(user_table.select().where(user_table.c.id > 5))
    ```

    产生：

    ```py
    SELECT  anon_1.name  AS  anon_1_name  FROM  (SELECT  users.id  AS  id,
    users.name  AS  name  FROM  users  WHERE  users.id  >  :id_1)  AS  anon_1
    ```

    正是后一种用例的“别名”行为妨碍了前一种用例。该方法现在明确将 SQL 表达式（如`select()`或`alias()`）与映射实体（如`aliased()`构造）分开考虑。

    参考：[#2635](https://www.sqlalchemy.org/trac/ticket/2635)

+   **[orm] [错误]**

    `MutableComposite` 类型不允许使用 `MutableBase.coerce()` 方法，即使代码似乎表明了这个意图，所以现在可以使用，并添加了一个简要示例。作为副作用，此事件处理程序的机制已更改，以使新的 `MutableComposite` 类型不再添加每类型的全局事件处理程序。还在 0.7.10 版本中。

    参考：[#2624](https://www.sqlalchemy.org/trac/ticket/2624)

+   **[orm] [bug]**

    对别名/内部路径机制的第二次彻底改进现在允许两个子类具有相同名称的不同关系，当使用完整的多态加载时同时支持子查询或联接的快速加载。

    参考文献：[#2614](https://www.sqlalchemy.org/trac/ticket/2614)

+   **[orm] [bug]**

    修复了一个错误，其中特定的 with_polymorphic 加载中的多跳子查询加载会产生 KeyError。利用了与 [#2614](https://www.sqlalchemy.org/trac/ticket/2614) 相同的内部路径改进。

    参考：[#2617](https://www.sqlalchemy.org/trac/ticket/2617)

+   **[orm] [bug]**

    修复了一个错误，当对象通过“fetch”同步策略匹配但本地不存在时，查询.update() 会产生错误。由 Scott Torborg 提供。

    参考：[#2602](https://www.sqlalchemy.org/trac/ticket/2602)

### orm 扩展

+   **[orm] [extensions] [feature]**

    `sqlalchemy.ext.mutable` 扩展现在包括作为扩展的一部分的示例 `MutableDict` 类。

### 引擎

+   **[engine] [feature]**

    `Connection.connect()` 和 `Connection.contextual_connect()` 方法现在返回一个“branched”版本，以便在返回的连接上调用 `Connection.close()` 方法而不影响原始连接。在使用 `Engine` 和 `Connection` 对象作为上下文管理器时允许对称性：

    ```py
    with conn.connect() as c:  # leaves the Connection open
        c.execute("...")

    with engine.connect() as c:  # closes the Connection
        c.execute("...")
    ```

+   **[engine] [bug]**

    修复了 `MetaData.reflect()` 方法，正确使用给定的 `Connection`，如果给定的话，而不是从该连接的 `Engine` 中打开第二个连接。

    此更改也已**回溯**至：0.7.10

    参考：[#2604](https://www.sqlalchemy.org/trac/ticket/2604)

+   **[engine]**

    `MetaData` 的“reflect=True”参数已被弃用。请使用 `MetaData.reflect()` ��法。

### sql

+   **[sql] [feature]**

    `Insert` 构造现在支持多值插入，即类似于“INSERT INTO table VALUES (…), (…), …”。受 PostgreSQL、SQLite 和 MySQL 支持。特别感谢 Idan Kamara 在这方面的工作。

    另请参阅

    Insert 的多值支持

    参考：[#2623](https://www.sqlalchemy.org/trac/ticket/2623)

+   **[sql] [bug]**

    修复了一个 bug，即在不传递“for_update=True”标志的情况下使用 server_onupdate=<FetchedValue|DefaultClause> 会将默认对象应用于 server_default，覆盖原有内容。这种用法不应该需要显式的 for_update=True 参数（尤其是因为文档中显示了一个没有使用该参数的示例），因此现在在内部使用给定默认对象的副本来安排，如果标志没有设置为对应该参数的值。

    此更改也已**回溯**至：0.7.10

    参考：[#2631](https://www.sqlalchemy.org/trac/ticket/2631)

+   **[sql] [bug]**

    修复了由 [#2410](https://www.sqlalchemy.org/trac/ticket/2410) 引起的回归，即在 `Table.tometadata()` 操作期间，`CheckConstraint` 会将自身应用回原始表，因为它会解析父表的 SQL 表达式。现在该操作会将给定表达式复制以对应新表。

    参考：[#2633](https://www.sqlalchemy.org/trac/ticket/2633)

+   **[sql] [bug]**

    修复了在方言上使用的 label_length 小于实际列标识符大小时，在 SELECT 语句中无法正确渲染列的 bug。

    参考：[#2610](https://www.sqlalchemy.org/trac/ticket/2610)

+   **[sql] [bug]**

    `DECIMAL` 类型现在在渲染 DDL 时遵守“precision”和“scale”参数。

    参考：[#2618](https://www.sqlalchemy.org/trac/ticket/2618)

+   **[sql] [bug]**

    对二进制表达式的“boolean”（即`__nonzero__`）评估进行了调整，例如`x1 == x2`，以确保`BinaryExpression`在某些情况下的“自动分组”不会妨碍此比较。之前，像下面这样的表达式：

    ```py
    expr1 = mycolumn > 2
    bool(expr1 == expr1)
    ```

    即使这是一个身份比较，也会评估为`False`，因为`mycolumn > 2`将在被放入`BinaryExpression`之前被“分组”，从而改变其身份。`BinaryExpression`现在跟踪传递的“原始”对象。此外，`__nonzero__`方法现在仅在运算符为`==`或`!=`时返回 - 其他所有情况均引发`TypeError`。

    参考：[#2621](https://www.sqlalchemy.org/trac/ticket/2621)

+   **[sql] [bug]**

    修复了一个陷阱，即无意中对`ColumnElement`调用`list()`会陷入无限循环，如果实现了`ColumnOperators.__getitem__()`。现在通过`__iter__()`发出新的 NotImplementedError。

+   **[sql] [bug]**

    修复了 type_coerce()中的错误，即如果语句被用作另一语句中的子查询，以及其他类似情况，可能会丢失类型信息。在其他情况下，当 Oracle/mssql 方言应用限制/偏移包装时，会导致类型信息丢失。

    参考：[#2603](https://www.sqlalchemy.org/trac/ticket/2603)

+   **[sql] [bug]**

    修复了一个错误，即在生成可选择的列的“代理”时未使用 Column 的“.key”。在 0.7 版本中可能不会发生这种情况，因为 0.7 版本在更广泛的情况下不会尊重“.key”。

    参考：[#2597](https://www.sqlalchemy.org/trac/ticket/2597)

### postgresql

+   **[postgresql] [feature]**

    `HSTORE`现在在 PostgreSQL 方言中可用。如果可用，还将使用 psycopg2 的扩展。感谢 Audrius Kažukauskas。

    参考：[#2606](https://www.sqlalchemy.org/trac/ticket/2606)

### sqlite

+   **[sqlite] [bug]**

    对此与 SQLite 相关的问题进行了更多调整，该问题在 0.7.9 中发布，以拦截反射外键时的遗留 SQLite 引号字符。除了拦截双引号之外，还拦截其他引号字符，例如方括号、反引号和单引号。

    此更改还**回溯**至：0.7.10

    参考：[#2568](https://www.sqlalchemy.org/trac/ticket/2568)

### mssql

+   **[mssql] [功能]**

    添加了对主键约束“name”的反射支持，感谢 Dave Moore。

    参考：[#2600](https://www.sqlalchemy.org/trac/ticket/2600)

+   **[mssql] [错误]**

    修复了在使用 Column 的“key”与拥有表的“schema”结合使用时，由于 MSSQL 方言的“schema 渲染”逻辑未考虑.key 而导致无法定位结果行的错误。

    此更改也已**回溯**至：0.7.10

### oracle

+   **[oracle] [错误]**

    修复了在访问引用到 DBLINK 远程数据库的同义词时，Oracle 的表反射问题；虽然 Oracle 方言中的语法已经存在一段时间，但直到现在才进行了测试。该语法已经针对链接到自身的示例数据库进行了测试，但在查询远程数据库的表信息时，仍存在一些不确定性。目前，从 user_db_links 中使用的“username”值用于匹配“owner”。

    参考：[#2619](https://www.sqlalchemy.org/trac/ticket/2619)

+   **[oracle] [错误]**

    Oracle 的 LONG 类型，虽然是一个无界文本类型，但在返回结果行时似乎不使用 cx_Oracle.LOB 类型，因此方言已被修复，排除了对 LONG 应用 cx_Oracle.LOB 过滤。同样适用于 0.7.10。

    参考：[#2620](https://www.sqlalchemy.org/trac/ticket/2620)

+   **[oracle] [错误]**

    修复了与 cx_Oracle 一起使用`.prepare()`时，如果返回值为`False`，则不会调用`connection.commit()`，从而避免“无事务”错误。已经展示了在 SQLAlchemy 和 cx_oracle 中以基本方式工作的两阶段事务，但受到驱动程序观察到的注意事项的影响；请查看文档获取详细信息。同样适用于 0.7.10。

    参考：[#2611](https://www.sqlalchemy.org/trac/ticket/2611)

### 杂项

+   **[功能] [sybase]**

    已为 Sybase 方言添加了反射支持。非常感谢 Ben Trofatter 为开发和测试所做的所有工作。

    参考：[#1753](https://www.sqlalchemy.org/trac/ticket/1753)

+   **[功能] [池]**

    `Pool`现在将记录所有 connection.close()操作，包括对无效连接、分离连接和超出池容量的连接的关闭。

+   **[功能] [池]**

    `Pool`现在咨询`Dialect`有关连接应如何“自动回滚”以及关闭的功能。这使得方言对事务范围有更多控制，因此我们将能够更好地实现对 pysqlite 和 cx_oracle 可能需要的事务处理的绕过。

    参考：[#2611](https://www.sqlalchemy.org/trac/ticket/2611)

+   **[feature] [pool]**

    添加了新的`PoolEvents.reset()`钩子，以在连接自动回滚之前捕获事件，返回到池中。与`ConnectionEvents.rollback()`一起，这允许拦截所有回滚事件。

+   **[bug] [firebird]**

    为实验性的“firebird+fdb”方言添加了“fdb”的缺失导入。

    参考：[#2622](https://www.sqlalchemy.org/trac/ticket/2622)

+   **[informix]**

    删除了有关 Informix 事务处理的一些杂项，包括一个会跳过调用 commit()/rollback()的功能，以及在 begin()中的一些硬编码隔离级别假设。由于我们没有任何用户使用它，也没有访问 Informix 数据库的权限，因此对这个方言的状态了解不多。如果有人能够访问 Informix 并愿意帮助测试这个方言，请告诉我们。

### orm

+   **[orm] [feature]**

    添加了`KeyedTuple._asdict()`和`KeyedTuple._fields`到`KeyedTuple`类，以提供与 Python 标准库`collections.namedtuple()`一定程度的兼容性。

    参考：[#2601](https://www.sqlalchemy.org/trac/ticket/2601)

+   **[orm] [feature]**

    允许在定义关系的主要和次要连接时使用同义词。

+   **[orm] [bug]**

    现在可以使用`aliased()`构造来与`Query.select_from()`方法一起使用，而不会干扰被选择的实体。基本上，像这样的语句：

    ```py
    ua = aliased(User)
    session.query(User.name).select_from(ua).join(User, User.name > ua.name)
    ```

    将保留 SELECT 的列子句作为来自未别名化的“user”，如指定的；select_from 仅在 FROM 子句中发生：

    ```py
    SELECT  users.name  AS  users_name  FROM  users  AS  users_1
    JOIN  users  ON  users.name  <  users_1.name
    ```

    请注意，这种行为与`Query.select_from()`的原始、较旧的用例形成对比，即在不同的可选择性方面重新表述映射实体：

    ```py
    session.query(User.name).select_from(user_table.select().where(user_table.c.id > 5))
    ```

    这将产生：

    ```py
    SELECT  anon_1.name  AS  anon_1_name  FROM  (SELECT  users.id  AS  id,
    users.name  AS  name  FROM  users  WHERE  users.id  >  :id_1)  AS  anon_1
    ```

    后一种用例的“别名”行为妨碍了前一种用例。该方法现在明确将类似`select()`或`alias()`的 SQL 表达式与像`aliased()`构造一样的映射实体分开考虑。

    参考：[#2635](https://www.sqlalchemy.org/trac/ticket/2635)

+   **[orm] [bug]**

    `MutableComposite` 类型不允许使用 `MutableBase.coerce()` 方法，尽管代码似乎表明了这一意图，所以现在可以使用，并添加了一个简短示例。作为副作用，此事件处理程序的机制已更改，以便新的 `MutableComposite` 类型不再添加每种类型的全局事件处理程序。也在 0.7.10 版本中。

    参考：[#2624](https://www.sqlalchemy.org/trac/ticket/2624)

+   **[orm] [bug]**

    第二次对别名/内部路径机制进行改进，现在允许两个子类具有相同名称的不同关系，在同时使用子查询或连接式加载时支持全多态加载时。当使用全多态加载时，同时在两个子类上使用子查询或连接式加载时支持全多态加载。

    参考：[#2614](https://www.sqlalchemy.org/trac/ticket/2614)

+   **[orm] [bug]**

    修复了在特定的 with_polymorphic 加载中多跳子查询加载会产生 KeyError 的错误。利用了与 [#2614](https://www.sqlalchemy.org/trac/ticket/2614) 相同的内部路径改进。

    参考：[#2617](https://www.sqlalchemy.org/trac/ticket/2617)

+   **[orm] [bug]**

    修复了查询.update()在“fetch”同步策略匹配的对象不在本地时会产生错误的回归。感谢 Scott Torborg。

    参考：[#2602](https://www.sqlalchemy.org/trac/ticket/2602)

### orm 扩展

+   **[orm] [extensions] [feature]**

    `sqlalchemy.ext.mutable` 扩展现在包括示例 `MutableDict` 类作为扩展的一部分。

### engine

+   **[engine] [feature]**

    `Connection.connect()` 和 `Connection.contextual_connect()` 方法现在返回一个“分支”版本，以便可以在返回的连接上调用 `Connection.close()` 方法而不影响原始连接。在使用`Engine` 和 `Connection` 对象作为上下文管理器时提供对称性：

    ```py
    with conn.connect() as c:  # leaves the Connection open
        c.execute("...")

    with engine.connect() as c:  # closes the Connection
        c.execute("...")
    ```

+   **[engine] [bug]**

    修复了`MetaData.reflect()`以正确使用给定的`Connection`，如果给定的话，而不是从该连接的`Engine`中打开第二个连接。

    此更改也被**回溯**到：0.7.10

    参考：[#2604](https://www.sqlalchemy.org/trac/ticket/2604)

+   **[engine]**

    对`MetaData`的“reflect=True”参数已被弃用。请使用`MetaData.reflect()`方法。

### sql

+   **[sql] [feature]**

    `Insert`构造现在支持多值插入，即，一个类似“INSERT INTO table VALUES (…), (…), …”的 INSERT。由 PostgreSQL、SQLite 和 MySQL 支持。非常感谢 Idan Kamara 在这方面的工作。

    另请参见

    插入的多值支持

    参考：[#2623](https://www.sqlalchemy.org/trac/ticket/2623)

+   **[sql] [bug]**

    修复了使用 server_onupdate=<FetchedValue|DefaultClause>而没有传递“for_update=True”标志的 bug，会将默认对象应用于 server_default，覆盖原有内容。明确的 for_update=True 参数在这种用法中不应该是必需的（特别是因为文档显示了一个没有使用它的示例），因此现在在内部使用给定默认对象的副本，如果标志未设置为对应该参数的内容。

    此更改也被**回溯**到：0.7.10

    参考：[#2631](https://www.sqlalchemy.org/trac/ticket/2631)

+   **[sql] [bug]**

    修复了由[#2410](https://www.sqlalchemy.org/trac/ticket/2410)引起的回归，其中`CheckConstraint`在`Table.tometadata()`操作期间会将自身应用回原始表，因为它会解析父表的 SQL 表达式。该操作现在将给定的表达式复制以对应新表。

    参考：[#2633](https://www.sqlalchemy.org/trac/ticket/2633)

+   **[sql] [bug]**

    修复了在 dialect 上使用 label_length 小于实际列标识符大小的 bug，会导致在 SELECT 语句中无法正确渲染列。

    参考：[#2610](https://www.sqlalchemy.org/trac/ticket/2610)

+   **[sql] [bug]**

    当渲染 DDL 时，`DECIMAL`类型现在遵守“precision”和“scale”参数。

    参考：[#2618](https://www.sqlalchemy.org/trac/ticket/2618)

+   **[sql] [bug]**

    对二进制表达式（即 `__nonzero__`）的“布尔”评估进行了调整，即 `x1 == x2`，以确保 `BinaryExpression` 在某些情况下的“自动分组”不会干扰此比较。先前，像这样的表达式：

    ```py
    expr1 = mycolumn > 2
    bool(expr1 == expr1)
    ```

    将被评估为 `False`，即使这是一个恒等比较，因为在被放入 `BinaryExpression` 之前，`mycolumn > 2` 将被“分组”，从而改变其标识。`BinaryExpression` 现在跟踪传入的“原始”对象。此外，`__nonzero__` 方法现在仅在操作符为 `==` 或 `!=` 时返回 - 所有其他操作符都会引发 `TypeError`。

    参考：[#2621](https://www.sqlalchemy.org/trac/ticket/2621)

+   **[sql] [bug]**

    修复了一个问题，即无意中对 `ColumnElement` 调用 list() 会进入无限循环，如果实现了 `ColumnOperators.__getitem__()`。现在通过 `__iter__()` 发出一个新的 NotImplementedError。

+   **[sql] [bug]**

    修复了 type_coerce() 中的错误，其中如果语句被用作另一个语句内部的子查询，以及其他类似情况，则可能会丢失类型信息。在 Oracle/mssql 方言应用 limit/offset 包装时，会导致类型信息丢失，这是其中之一。

    参考：[#2603](https://www.sqlalchemy.org/trac/ticket/2603)

+   **[sql] [bug]**

    修复了一个错误，即在生成可选择的列的“代理”时，未使用列的“.key”。这可能在 0.7 中没有发生，因为在更广泛的范围内 0.7 不尊重“.key”。

    参考：[#2597](https://www.sqlalchemy.org/trac/ticket/2597)

### postgresql

+   **[postgresql] [feature]**

    在 PostgreSQL 方言中现在可以使用 `HSTORE`。如果可用，还将使用 psycopg2 的扩展。感谢 Audrius Kažukauskas。

    参考：[#2606](https://www.sqlalchemy.org/trac/ticket/2606)

### sqlite

+   **[sqlite] [bug]**

    对于这个与 SQLite 相关的问题进行了更多调整，该问题在 0.7.9 中发布，以拦截反射外键时的传统 SQLite 引号字符。除了拦截双引号外，现在还拦截其他引号字符，如方括号、反引号和单引号。

    这个更改也被**回溯**到：0.7.10

    参考：[#2568](https://www.sqlalchemy.org/trac/ticket/2568)

### mssql

+   **[mssql] [功能]**

    添加了对主键约束“name”的反射支持，感谢 Dave Moore。

    参考：[#2600](https://www.sqlalchemy.org/trac/ticket/2600)

+   **[mssql] [错误]**

    修复了在将“key”与 Column 结合使用时与拥有表的“schema”失败的 bug，原因是 MSSQL 方言的“schema 渲染”逻辑未考虑 .key。

    此更改也已**回溯**到：0.7.10

### oracle

+   **[oracle] [错误]**

    修复了在访问引用到 DBLINK 远程数据库的同义词时的 Oracle 表反射问题；虽然该语法在 Oracle 方言中已经存在一段时间，但直到现在还没有进行过测试。该语法已针对连接到自身的样本数据库进行了测试，但是仍然存在一些不确定因素，即在查询远程数据库以获取表信息时应使用什么作为“owner”的值。当前，来自 user_db_links 的“username” 值用于匹配“owner”。

    参考：[#2619](https://www.sqlalchemy.org/trac/ticket/2619)

+   **[oracle] [错误]**

    Oracle LONG 类型，虽然是无界文本类型，但在返回结果行时似乎没有使用 cx_Oracle.LOB 类型，因此修复了该方言以排除 LONG 从应用 cx_Oracle.LOB 过滤。同样适用于 0.7.10。

    参考：[#2620](https://www.sqlalchemy.org/trac/ticket/2620)

+   **[oracle] [错误]**

    修复了与 cx_Oracle 结合使用 `.prepare()` 的用法，因此返回值为 `False` 将导致不调用 `connection.commit()`，从而避免“无事务”错误。已经证明 SQLAlchemy 和 cx_oracle 可以以一种基本方式使用两阶段事务，但是受到驱动程序观察到的警告的影响；有关详细信息，请参阅文档。同样适用于 0.7.10。

    参考：[#2611](https://www.sqlalchemy.org/trac/ticket/2611)

### 杂项

+   **[功能] [sybase]**

    已向 Sybase 方言添加了反射支持。非常感谢 Ben Trofatter 在开发和测试中所做的所有工作。

    参考：[#1753](https://www.sqlalchemy.org/trac/ticket/1753)

+   **[功能] [池]**

    `Pool` 现在将平等记录所有连接关闭（`connection.close()`）操作，包括对失效连接、脱离连接和超出池容量的连接的关闭。

+   **[功能] [池]**

    `Pool` 现在会咨询 `Dialect` 关于连接应该如何“自动回滚”以及关闭的功能。这使得方言更加掌控事务范围，因此我们将更好地能够实现对 pysqlite 和 cx_oracle 等可能需要的事务性回避的控制。

    参考：[#2611](https://www.sqlalchemy.org/trac/ticket/2611)

+   **[功能] [池]**

    添加了新的`PoolEvents.reset()`钩子，用于在连接自动回滚之前捕获事件，返回到池中。与`ConnectionEvents.rollback()`一起，这允许拦截所有回滚事件。

+   **[错误] [firebird]**

    为实验性的“firebird+fdb”方言添加了“fdb”的缺失导入。

    参考：[#2622](https://www.sqlalchemy.org/trac/ticket/2622)

+   **[informix]**

    已删除有关 informix 事务处理的一些不必要内容，包括一个跳过调用 commit()/rollback()以及在 begin()上的一些硬编码隔离级别假设的功能。由于我们没有任何用户使用它，也没有访问 Informix 数据库的权限，因此对这种方言的状态了解不多。如果有人有权访问 Informix 并愿意帮助测试这种方言，请告诉我们。

## 0.8.0b1

发布日期：2012 年 10 月 30 日

### 通用

+   **[通用] [已移除]**

    “sqlalchemy.exceptions”作为“sqlalchemy.exc”的同义词已完全移除。

    参考：[#2433](https://www.sqlalchemy.org/trac/ticket/2433)

+   **[通用]**

    SQLAlchemy 0.8 现在支持 Python 2.5 及以上版本。不再支持 Python 2.4。

### orm

+   **[orm] [功能]**

    relationship()内部的重大重写现在允许包含指向自身的列在复合外键中的连接条件。添加了一个新的 API，用于非常专业的 primaryjoin 条件，允许基于 SQL 函数、CAST 等的条件通过在表达式中必要时内联放置注释函数 remote()和 foreign()来处理。以前使用半私有的 _local_remote_pairs 方法的配方可以升级到这种新方法。

    另请参阅

    重写的 _orm.relationship()机制

    参考：[#1401](https://www.sqlalchemy.org/trac/ticket/1401)

+   **[orm] [功能]**

    新的独立函数 with_polymorphic()提供了 query.with_polymorphic()的功能，以独立形式提供。它可以应用于查询中的任何实体，包括作为“of_type()”修饰符的连接目标。

    参考：[#2333](https://www.sqlalchemy.org/trac/ticket/2333)

+   **[orm] [功能]**

    现在属性上的 of_type()构造接受别名为 aliased()的类构造以及 with_polymorphic 构造，并且与 query.join()、any()、has()以及 eager loaders subqueryload()、joinedload()、contains_eager()一起工作。

    参考：[#1106](https://www.sqlalchemy.org/trac/ticket/1106)，[#2438](https://www.sqlalchemy.org/trac/ticket/2438)

+   **[orm] [功能]**

    对映射类的事件监听进行了改进，允许指定未映射类用于实例和映射器事件。当传递 propagate=True 标志时，已建立的事件将自动设置在该类的子类上，并且在最终映射时将为该类本身设置事件。

    参考：[#2585](https://www.sqlalchemy.org/trac/ticket/2585)

+   **[orm] [feature]**

    “延迟声明反射”系统已经移入声明式扩展本身，使用新的 DeferredReflection 类。该类现在已经针对单表和联合表继承用例进行了测试。

    参考：[#2485](https://www.sqlalchemy.org/trac/ticket/2485)

+   **[orm] [feature]**

    添加了新的核心功能“inspect()”，它作为对映射器、对象等进行内省的通用入口。映射器和 InstanceState 对象已经增强了公共 API，允许检查映射属性，包括针对列绑定或关系绑定属性的过滤器，检查当前对象状态，属性历史记录等。

    参考：[#2208](https://www.sqlalchemy.org/trac/ticket/2208)

+   **[orm] [feature]**

    在 session.begin_nested()中调用 rollback()现在只会使那些在该事务范围内具有净变化的对象过期，即在刷新时被修改或被修改的对象。这允许 begin_nested()的典型用例，即修改一小部分对象，保留未在子事务中修改的更大范围对象的数据。

    参考：[#2452](https://www.sqlalchemy.org/trac/ticket/2452)

+   **[orm] [feature]**

    添加了实用功能 Session.enable_relationship_loading()，取代了 relationship.load_on_pending。然而，应该避免使用这两个功能。

    参考：[#2372](https://www.sqlalchemy.org/trac/ticket/2372)

+   **[orm] [feature]**

    添加了对.column_property()、relationship()、composite()的.info 字典参数的支持。所有 MapperProperty 类都具有可用的自动创建的.info 字典。

+   **[orm] [feature]**

    从映射集合中添加/移除 None 现在会生成属性事件。以前，在某些情况下，会忽略 None 的追加。相关。

    参考：[#2229](https://www.sqlalchemy.org/trac/ticket/2229)

+   **[orm] [feature]**

    映射集合中存在 None 现在在刷新时会引发错误。以前，集合中的 None 值会被静默忽略。

    参考：[#2229](https://www.sqlalchemy.org/trac/ticket/2229)

+   **[orm] [feature]**

    Query.update()方法现在对被更新的表更加宽松。现在更好地支持普通的 Table 对象，并且可以使用附加的继承子类进行 update(); 子类表将成为更新的目标，如果父表在 WHERE 子句中被引用，编译器将调用 UPDATE..FROM 语法，以满足 WHERE 子句。如果在“values”字典中通过对象指定列，还支持 MySQL 的多表更新功能。PG 的 DELETE..USING 在 Core 中还不可用。

+   **[orm] [feature]**

    新的 session 事件 after_transaction_create 和 after_transaction_end 允许跟踪新的 SessionTransaction 对象。如果检查了对象，可以用于确定会话何时首次激活和何时停用。

+   **[orm] [feature]**

    Query 现在可以加载包含不可哈希类型的实体/标量混合“元组”行，方法是在使用的相应 TypeEngine 对象上设置“hashable=False”标志。返回不可哈希类型（通常是列表）的自定义类型可以将此标志设置为 False。

    参考：[#2592](https://www.sqlalchemy.org/trac/ticket/2592)

+   **[orm] [feature]**

    现在 Query 默认情况下会“自动关联”，与 select()相同。以前，在另一个查询中使用的 Query 需要显式调用 correlate()方法，以便将内部的表与外部关联起来。如常，correlate(None)会禁用关联。

    参考：[#2179](https://www.sqlalchemy.org/trac/ticket/2179)

+   **[orm] [feature]**

    在 Session.add()、Session.merge()等操作后，现在会触发 after_attach 事件，以确保对象在 Session.new 或 Session.identity_map 中建立后，当调用事件时，对象会在这些集合中表示。添加了 before_attach 事件以适应需要使用预附加对象的用例。

    参考：[#2464](https://www.sqlalchemy.org/trac/ticket/2464)

+   **[orm] [feature]**

    当在 flush 的“execute”部分中使用不受支持的方法时，Session 将产生警告。这些熟悉的方法包括 add()、delete()等，以及在 mapper 级别 flush 事件中调用的集合和相关对象操作，如 after_insert()、after_update()等。长期以来，明确记录了当 Session 在执行 flush 计划时被操纵时，SQLAlchemy 无法保证结果，但用户仍在这样做，所以现在有了警告。也许将来 Session 会增强以支持在 flush 内部执行这些操作，但目前无法保证结果。

+   **[orm] [feature]**

    ORM 实体可以传递给核心 select()构造，以及传递给 select()的 select_from()、correlate()和 correlate_except()方法，它们将被解包为可选择的。

    参考：[#2245](https://www.sqlalchemy.org/trac/ticket/2245)

+   **[orm] [feature]**

    为基于映射属性的关系连接条件提供自动渲染支持，使用核心 SQL 构造。例如，select([SomeClass]).where(SomeClass.somerelationship) 将渲染出从“someclass”选择，并使用“somerelationship”的主连接作为 WHERE 子句。这改变了在核心 SQL 上下文中使用“SomeClass.somerelationship”的先前含义；以前，它会“解析”为父可选择项，这通常不太有用。也适用于 query.filter()。相关内容。

    参考：[#2245](https://www.sqlalchemy.org/trac/ticket/2245)

+   **[orm] [功能]**

    在 declarative_base() 中的类注册现在是 WeakValueDictionary。因此，“Base”的子类如果没有被任何其他映射器/超类映射器引用，将被垃圾回收。请查看此票证的下一个注释。

    参考：[#2526](https://www.sqlalchemy.org/trac/ticket/2526)

+   **[orm] [功能]**

    可以使用文档中描述的新的 @declared_attr 用法解决单继承声明子类之间的列冲突，无论是否使用混合类。

    参考：[#2472](https://www.sqlalchemy.org/trac/ticket/2472)

+   **[orm] [功能]**

    declared_attr 现在可以用于非混合类，尽管这通常只对单继承子类列冲突解决有用。

    参考：[#2472](https://www.sqlalchemy.org/trac/ticket/2472)

+   **[orm] [功能]**

    declared_attr 现在可以用于不是 Column 或 MapperProperty 的属性；包括任何用户定义的值以及关联代理对象。

    参考：[#2517](https://www.sqlalchemy.org/trac/ticket/2517)

+   **[orm] [功能]**

    *非常有限*的支持，当类本身被解除引用时，继承映射器可以被垃圾回收。映射器不能有自己的表（即仅支持单表继承），没有放置多态属性。这允许用例创建一个临时的声明映射类的子类，在被单元测试解除引用时可以被垃圾回收。

    参考：[#2526](https://www.sqlalchemy.org/trac/ticket/2526)

+   **[orm] [功能]**

    Declarative 现在通过字符串名称和完整模块限定名称维护类的注册表。现在可以基于关系() 中的模块限定字符串查找具有相同名称的多个类。当多个类共享相同名称时，简单类名查找现在会引发信息性错误消息。

    参考：[#2338](https://www.sqlalchemy.org/trac/ticket/2338)

+   **[orm] [功能]**

    现在可以提供类绑定属性，覆盖任何非 ORM 类型的列，而不仅仅是描述符。

    参考：[#2535](https://www.sqlalchemy.org/trac/ticket/2535)

+   **[orm] [功能]**

    在 Query.subquery() 中添加了 with_labels 和 reduce_columns 关键字参数，提供两种生成具有唯一命名列的查询的替代策略。

    参考：[#1729](https://www.sqlalchemy.org/trac/ticket/1729)

+   **[orm] [feature]**

    当对一个仪器化集合的引用由于过期/属性刷新/集合替换而不再与父类关联，但现在分离的集合接收到附加或移除操作时，会发出警告。

    参考：[#2476](https://www.sqlalchemy.org/trac/ticket/2476)

+   **[orm] [bug]**

    在刷新时，如果两个表之间存在外键依赖关系，并且这些表通过连接继承相关联，并且外键依赖关系不是 inherit_condition 的一部分，则 ORM 将进行额外的努力来确定这种依赖关系不重要，从而为用户节省了 use_alter 指令。

    参考：[#2527](https://www.sqlalchemy.org/trac/ticket/2527)

+   **[orm] [bug]**

    现在，仅对 listen() 分配的类的后代类触发 instrumentation 事件 class_instrument()、class_uninstrument() 和 attribute_instrument()。以前，无论传递了什么“目标”参数，事件侦听器都会被分配为在所有情况下监听所有类。

    参考：[#2590](https://www.sqlalchemy.org/trac/ticket/2590)

+   **[orm] [bug]**

    在将多级子类以任意顺序或中间类缺失的情况下发送给 with_polymorphic() 时，会按正确顺序生成 JOIN，并在正确的继承表中生成 JOIN。

    参考：[#1900](https://www.sqlalchemy.org/trac/ticket/1900)

+   **[orm] [bug]**

    对共享共同基类的子类实体链进行了改进，处理连接/子查询的急切加载，没有提供特定的“连接深度”。在检测到“循环”之前，将单独链出每个子类映射器，而不是将基类视为“循环”的源。

    参考：[#2481](https://www.sqlalchemy.org/trac/ticket/2481)

+   **[orm] [bug]**

    “被动”标志在 Session.is_modified() 上不再起作用。在所有情况下，is_modified() 只查看本地内存中修改的标志，不会发出任何 SQL 或调用加载器可调用/初始化程序。

    参考：[#2320](https://www.sqlalchemy.org/trac/ticket/2320)

+   **[orm] [bug]**

    当使用 delete-orphan 级联时，如果 one-to-many 或 many-to-many 没有设置 single-parent=True，则发出的警告现在是一个错误。在任何情况下，ORM 在此警告后将无法正常运行。

    参考：[#2405](https://www.sqlalchemy.org/trac/ticket/2405)

+   **[orm] [bug]**

    在 flush 事件中发出的延迟加载，如 before_flush()、before_update()等，现在将像在非事件代码中一样运行，关于在延迟发出的查询中使用的 PK/FK 值的考虑。以前，会建立特殊标志，导致延迟加载基于在刷新时调用时父 PK/FK 值的“先前”值加载相关项目；现在，以这种方式加载的信号现在局限于工作单元实际需要以这种方式加载的地方。请注意，UOW 有时会在调用 before_update()事件之前加载这些集合，因此“passive_updates”的使用与否可能会影响在刷新事件中访问时集合是否表示“旧”或“新”数据，根据延迟加载何时发出。这种变化在极小的机会上是不兼容的，用户事件代码依赖于旧行为。

    参考：[#2350](https://www.sqlalchemy.org/trac/ticket/2350)

+   **[orm] [bug]**

    继续关于由于事件监听器导致刷新后的额外状态；任何从属性角度标记为“脏”的状态，通常通过 after_insert()、after_update()等中的列属性设置事件，将在所有情况下重置“历史”标志，而不仅仅是那些参与刷新的实例。这样做的效果是，这种“脏”状态在刷新后不会传递，并且不会导致 UPDATE 语句。会发出一个警告；可以使用 set_committed_state()方法在对象上分配属性而不产生历史事件。

    参考：[#2566](https://www.sqlalchemy.org/trac/ticket/2566), [#2582](https://www.sqlalchemy.org/trac/ticket/2582)

+   **[orm] [bug]**

    修复了在@declared_attr Column 和直接定义的 Column 之间逐渐演变的断开。在这两种情况下，Column 将被应用于声明类的表，但不会应用于联合继承子类的表。以前，直接定义的 Column 会被放置在基表和子表上，这通常不是所期望的。

    参考：[#2565](https://www.sqlalchemy.org/trac/ticket/2565)

+   **[orm] [bug]**

    现在，声明式可以将在单表继承子类上声明的列传播到父类的表，当父类本身被映射到一个 join()或 select()语句时，直接或通过联合继承，而不仅仅是一个 Table。

    参考：[#2549](https://www.sqlalchemy.org/trac/ticket/2549)

+   **[orm] [bug]**

    当 uselist=False 与“dynamic”加载器结合时会发出错误。这在 0.7.9 中是一个警告。

+   **[orm] [removed]**

    ORM 的传统“可变”系统，包括 MutableType 类以及 PickleType 和 postgresql.ARRAY 上的 mutable=True 标志已被移除。ORM 使用在 0.7 版本中引入的 sqlalchemy.ext.mutable 扩展来检测原地变异。移除 MutableType 和相关结构从 SQLAlchemy 的内部移除了大量复杂性。这种方法的性能表现不佳，因为在使用时会导致对 Session 的全部内容进行扫描。

    参考：[#2442](https://www.sqlalchemy.org/trac/ticket/2442)

+   **[orm] [removed]**

    已移除的弃用标识符：

    +   allow_null_pks mapper() 参数���使用 allow_partial_pks）

    +   _get_col_to_prop() 映射器方法（使用 get_property_by_column()）

    +   Session.merge() 的 dont_load 参数（使用 load=True）

    +   sqlalchemy.orm.shard 模块（使用 sqlalchemy.ext.horizontal_shard）

+   **[orm] [moved]**

    InstrumentationManager 接口和整个相关的替代类实现系统现在已经移动到 sqlalchemy.ext.instrumentation。这是一个很少使用的系统，会给类的仪器化机制增加显著的复杂性和开销。新的架构允许它保持未使用状态，直到实际导入 InstrumentationManager 时，它才会被引导到核心部分。

### examples

+   **[examples]**

    Beaker 缓存示例已转换为使用 [dogpile.cache](https://dogpilecache.readthedocs.io/)。这是一个由 Beaker 缓存内部的相同创建者编写的新缓存库，代表了一个大幅改进、简化和现代化的缓存系统。

    另请参阅

    Dogpile Caching

    参考：[#2589](https://www.sqlalchemy.org/trac/ticket/2589)

### engine

+   **[engine] [feature]**

    现在连接事件监听器可以与单独的 Connection 对象关联，而不仅仅是 Engine 对象。

    参考：[#2511](https://www.sqlalchemy.org/trac/ticket/2511)

+   **[engine] [feature]**

    before_cursor_execute 事件会触发所谓的“_cursor_execute”事件，这些事件通常是主键绑定序列和在 INSERT 时未使用 RETURNING 时调用的默认生成 SQL 短语的特殊执行。

    参考：[#2459](https://www.sqlalchemy.org/trac/ticket/2459)

+   **[engine] [feature]**

    测试套件使用的库已经稍微移动了一下，以便它们再次成为 SQLAlchemy 安装的一部分。此外，新的测试套件现在位于新的 sqlalchemy.testing.suite 包中。这是一个正在开发中的系统，希望为外部方言提供一个通用的测试套件。在 SQLAlchemy 之外维护的方言可以使用新的测试装置作为其自己测试的框架，并将免费获得一个“兼容性”方言专注的测试套件，包括一个改进的“要求”系统，其中可以为测试启用或禁用特定功能和特性。

+   **[engine] [feature]**

    添加了一个新的系统，可以在不使用入口点的情况下在进程中注册新的方言。请参阅“注册新方言”的文档。

    参考：[#2462](https://www.sqlalchemy.org/trac/ticket/2462)

+   **[engine] [feature]**

    如果没有明确传递“value”或“callable”参数，则默认将“required”标志设置为 True，在 bindparam() 上。这将导致语句执行检查参数是否存在于最终的绑定参数集合中，而不是隐式地赋值为 None。

    参考：[#2556](https://www.sqlalchemy.org/trac/ticket/2556)

+   **[engine] [feature]**

    对“方言” API 进行了各种 API 调整，以更好地支持高度专业化的系统，如 Akiban 数据库，包括更多的钩子，允许执行上下文访问类型处理器。

+   **[engine] [feature]**

    Inspector.get_primary_keys() 已弃用；使用 Inspector.get_pk_constraint()。谢谢 Diana Clarke。

    参考：[#2422](https://www.sqlalchemy.org/trac/ticket/2422)

+   **[engine] [feature]**

    添加了一个新的 C 扩展模块“utils”，用于在有时间实现时进行额外的函数加速。

+   **[engine] [bug]**

    Inspector.get_table_names() 的 order_by=”foreign_key” 功能现在将表按照依赖者优先排序，以与 util.sort_tables 和 metadata.sorted_tables 保持一致。

+   **[engine] [bug]**

    修复了一个 bug，即如果数据库重新启动影响了多个连接，则每个连接都会单独调用池的新释放，即使只需要一个释放。

    参考：[#2522](https://www.sqlalchemy.org/trac/ticket/2522)

+   **[engine] [bug]**

    select().apply_labels() 上的 .c. 属性的列名称现在基于 <tablename>_<colkey> 而不是 <tablename>_<colname>，对于具有明确定义的 .key 的列。

    参考：[#2397](https://www.sqlalchemy.org/trac/ticket/2397)

+   **[engine] [bug]**

    autoload_replace 标志在 Table 上，当为 False 时，将导致引用已声明列的反射外键约束被跳过，假设 Python 中声明的列将接管指定 Python 中的 ForeignKey 或 ForeignKeyConstraint 声明的任务。

+   **[engine] [bug]**

    ResultProxy 方法 inserted_primary_key、last_updated_params()、last_inserted_params()、postfetch_cols()、prefetch_cols() 都会断言给定的语句是一个已编译的构造，并且是一个适当的 insert() 或 update() 语句，否则会引发 InvalidRequestError。

    参考：[#2498](https://www.sqlalchemy.org/trac/ticket/2498)

+   **[engine]**

    ResultProxy.last_inserted_ids 被移除，替换为 inserted_primary_key。

### sql

+   **[sql] [feature]**

    添加了一个新的方法`Engine.execution_options()`到`Engine`。这个方法的工作方式类似于`Connection.execution_options()`，因为它创建了一个引用新选项集的父对象的副本。该方法可用于构建每个引擎共享相同基础连接池的分片方案。该方法已在 ORM 中针对水平分片配方进行了测试。

    另见

    `Engine.execution_options()`

+   **[sql] [feature]**

    核心中的运算符系统进行了重大改组，允许重新定义现有运算符以及在类型级别添加新运算符。新类型可以从现有类型创建，这些类型增加或重新定义了导出到列表达式的操作，类似于 ORM 允许 comparator_factory。新的架构将此功能移入核心，以便在所有情况下一致可用，并使用现有类型传播行为进行干净传播。

    参考：[#2547](https://www.sqlalchemy.org/trac/ticket/2547)

+   **[sql] [feature]**

    为了补充，类型现在可以提供“绑定表达式”和“列表达式”，允许在列或绑定级别上将 SQL 表达式注入到语句中。这适用于类型需要在 SQL 级别增强绑定和结果行为的用例，而不是在 Python 级别。允许类似透明加密/解密、使用 PostGIS 函数等方案。

    参考：[#1534](https://www.sqlalchemy.org/trac/ticket/1534), [#2547](https://www.sqlalchemy.org/trac/ticket/2547)

+   **[sql] [feature]**

    核心运算符系统现在包括 getitem 运算符，即 Python 中的括号运算符。首先用于为 PostgreSQL ARRAY 类型提供索引和切片行为，并且还提供了一个用于端用户定义自定义 __getitem__ 方案的钩子，这些方案可以应用于类型级别以及 ORM 级别的自定义操作符方案中。lshift（<<）和 rshift（>>）也支持作为可选运算符。

    注意，这个变化的效果是，ORM 与 synonym()或其他“装饰器封装”方案一起使用的基于描述符的 __getitem__ 方案将需要开始使用自定义比较器以保持这种行为。

+   **[sql] [feature]**

    修订了用于确定用户定义运算符的运算符优先级的规则，即使用`op()`方法授予的运算符。以前，在所有情况下都应用最小的优先级，现在默认优先级为零，低于所有运算符，除了“逗号”（例如，在`func`调用的参数列表中使用）和“AS”，并且还可以通过`op()`方法��的“precedence”参数进行自定义。

    参考：[#2537](https://www.sqlalchemy.org/trac/ticket/2537)

+   **[sql] [feature]**

    对所有 String 类型添加了“collation”参数。当存在时，呈现为 COLLATE <collation>。这是为了支持现在多个数据库（包括 MySQL、SQLite 和 PostgreSQL）支持的 COLLATE 关键字。

    参考：[#2276](https://www.sqlalchemy.org/trac/ticket/2276)

+   **[sql] [feature]**

    现在可以通过将 operators.custom_op()与 UnaryExpression()结合来使用自定义的一元运算符。

+   **[sql] [feature]**

    增强了 GenericFunction 和 func.*，允许通过类名自动在 func.*命名空间中使用用户定义的 GenericFunction 子类，可选地使用包名，以及具有与 func.*中标识名称不同的渲染名称的功能。

+   **[sql] [feature]**

    现在 cast()和 extract()构造也将通过 func.*访问器生成，因为用户自然会尝试从 func.*访问这些名称，即使返回的对象不是 FunctionElement 也应该符合预期。

    参考：[#2562](https://www.sqlalchemy.org/trac/ticket/2562)

+   **[sql] [feature]**

    现在可以使用新的 inspect()服务获取 Inspector 对象的实例。

    参考：[#2208](https://www.sqlalchemy.org/trac/ticket/2208)

+   **[sql] [feature]**

    column_reflect 事件现在接受 Inspector 对象作为第一个参数，位于“table”之前。使用这个非常新的事件的 0.7 版本的代码将需要修改，以添加“inspector”对象作为第一个参数。

    参考：[#2418](https://www.sqlalchemy.org/trac/ticket/2418)

+   **[sql] [feature]**

    现在结果集中列的定位行为默认区分大小写。SQLAlchemy 多年来会对这些值进行不区分大小写的转换，可能是为了缓解像 Oracle 和 Firebird 这样的方言早期大小写敏感性问题。这些问题在更现代的版本中已经更清晰地解决，因此在标识符上调用 lower()的性能损失已经消除。可以通过在 create_engine()上设置“case_insensitive=False”来重新启用不区分大小写的比较。

    参考：[#2423](https://www.sqlalchemy.org/trac/ticket/2423)

+   **[sql] [feature]**

    当在 insert.values()或 update.values()中存在不在目标表中的键时，发出的“未使用的列名”警告现在是一个异常。

    参考：[#2415](https://www.sqlalchemy.org/trac/ticket/2415)

+   **[sql] [feature]**

    向 ForeignKey、ForeignKeyConstraint 添加了“MATCH”子句，感谢 Ryan Kelly。

    参考：[#2502](https://www.sqlalchemy.org/trac/ticket/2502)

+   **[sql] [feature]**

    添加了对从表的别名进行 DELETE 和 UPDATE 的支持，这在查询中可能与其他地方的自身相关联，由 Ryan Kelly 提供。

    ��考：[#2507](https://www.sqlalchemy.org/trac/ticket/2507)

+   **[sql] [feature]**

    select()具有 correlate_except()方法，自动关联除传递的所有 selectables 之外的所有 selectables。

+   **[sql] [feature]**

    prefix_with()方法现在在每个 select()、insert()、update()、delete()上都可用，具有相同的 API，接受多个前缀调用，以及“方言名称”，以便将前缀限制为一种方言。

    参考：[#2431](https://www.sqlalchemy.org/trac/ticket/2431)

+   **[sql] [feature]**

    在 select()构造中添加了 reduce_columns()方法，使用 util.reduce_columns 实用程序函数内联替换列以删除等效列。reduce_columns()还添加了“with_only_synonyms”以限制仅对具有相同名称的列进行减少。已删除不推荐使用的 fold_equivalents()功能。

    参考：[#1729](https://www.sqlalchemy.org/trac/ticket/1729)

+   **[sql] [feature]**

    重新设计了 startswith()、endswith()、contains()运算符，以更好地处理否定（NOT LIKE），并且在编译时组装它们，以便它们的渲染 SQL 可以被修改，例如在 Firebird STARTING WITH 的情况下。

    参考：[#2470](https://www.sqlalchemy.org/trac/ticket/2470)

+   **[sql] [feature]**

    添加了一个钩子到渲染 CREATE TABLE 系统，通过针对新的 schema.CreateColumn 构造函数构造@compiles 函数，为每个列提供访问渲染的系统。

    参考：[#2463](https://www.sqlalchemy.org/trac/ticket/2463)

+   **[sql] [feature]**

    “标量”选择现在具有 WHERE 方法以帮助生成构建。此外，关于 SS“关联”列的方式略有调整；新方法不再将意义应用于所选的基础 Table 列。这改进了一些相当晦涩的情况，而且似乎之前的逻辑没有任何目的。

+   **[sql] [feature]**

    当首次使用构造为引用多个远程表的 ForeignKeyConstraint()时，将引发显式错误。

    参考：[#2455](https://www.sqlalchemy.org/trac/ticket/2455)

+   **[sql] [feature]**

    向`ColumnOperators.notin_()`、`ColumnOperators.notlike()`、`ColumnOperators.notilike()`添加了`ColumnOperators`。

    参考：[#2580](https://www.sqlalchemy.org/trac/ticket/2580)

+   **[sql] [change]**

    如果指定了长度，Text()类型将呈现给定的长度。

+   **[sql] [changed]**

    expression.sql 中的大多数类不再以下划线开头，即 Label、SelectBase、Generative、CompareMixin。_BindParamClause 也更名为 BindParameter。这些类的旧下划线名称将在可预见的未来保持可用作为同义词。

+   **[sql] [bug]**

    修复了传递给`Compiler.process()`的关键字参数不会传播到 SELECT 语句的列子句中存在的列表达式的 bug。特别是当自定义编译方案依赖于特殊标志时，这种情况会出现。

    参考：[#2593](https://www.sqlalchemy.org/trac/ticket/2593)

+   **[sql] [bug] [orm]**

    `select()`的自动相关特性，以及由此引发的`Query`的特性，不会对直接在包含 SELECT 的 FROM 列表中呈现的 SELECT 语句产生影响。 SQL 中的相关性仅适用于诸如 WHERE、ORDER BY、列子句中的列表达式。

    参考：[#2595](https://www.sqlalchemy.org/trac/ticket/2595)

+   **[sql] [bug]**

    调整了列优先级，将“concat”和“match”运算符移动到与“is”、“like”等运算符相同的位置；这有助于在与“IS”结合使用时进行括号渲染。 

    参考：[#2564](https://www.sqlalchemy.org/trac/ticket/2564)

+   **[sql] [bug]**

    将列表达式应用于带有标签的选择语句，无论是否有其他修改构造，都不再将该表达式“定位”到底层列；这会影响依赖于列定位以检索结果的 ORM 操作。也就是说，像 query(User.id, User.id.label(‘foo’))这样的查询现在将分别跟踪每个“User.id”表达式的值，而不是将它们混合在一起。预计不会影响任何用户；但是，如果使用 select()与 query.from_statement()结合使用并尝试加载完全组合的 ORM 实体的用法可能不会按预期运行，如果 select()命名的列对象具有任意的.label()名称，因为这些将不再定位到由该实体映射的列对象。

    参考：[#2591](https://www.sqlalchemy.org/trac/ticket/2591)

+   **[sql] [bug]**

    修复了将列“default”参数解释为可调用项，以便不将 ExecutionContext 传递给关键字参数的问题。

    参考：[#2520](https://www.sqlalchemy.org/trac/ticket/2520)

+   **[sql] [bug]**

    所有的 UniqueConstraint、ForeignKeyConstraint、CheckConstraint 和 PrimaryKeyConstraint 在直接引用绑定到表的 Column 对象时（即不仅仅是字符串列名），并且只引用一个 Table 时，将自动附加到它们的父表上。在 0.8 之前，这种行为仅适用于 UniqueConstraint 和 PrimaryKeyConstraint，而不适用于 ForeignKeyConstraint 或 CheckConstraint。

    参考：[#2410](https://www.sqlalchemy.org/trac/ticket/2410)

+   **[sql] [错误]**

    TypeDecorator 现在包括一个通用的 repr()，默认情况下以“impl”类型为基础工作。对于那些指定了自定义 __init__ 方法的 TypeDecorator 类来说，这是一个行为变化；那些类型如果需要 __repr__()提供一个忠实的构造函数表示，就需要重新定义 __repr__()。

    参考：[#2594](https://www.sqlalchemy.org/trac/ticket/2594)

+   **[sql] [错误]**

    column.label(None)现在生成一个匿名标签，而不是返回列对象本身，与 label(column, None)的行为一致。

    参考：[#2168](https://www.sqlalchemy.org/trac/ticket/2168)

+   **[sql] [已移除]**

    在`create_engine()`以及`String`上长期弃用且无效的`assert_unicode`标志已被移除。

### postgresql

+   **[postgresql] [功能]**

    postgresql.ARRAY 具有可选的“维度”参数，将为数组分配特定数量的维度，这将在 DDL 中呈现为 ARRAY[][]…，还提高了绑定/结果处理的性能。

    参考：[#2441](https://www.sqlalchemy.org/trac/ticket/2441)

+   **[postgresql] [功能]**

    postgresql.ARRAY 现在支持索引和切片。Python []运算符可用于所有类型为 ARRAY 的 SQL 表达式；可以传递整数或简单切片。这些切片也可以在 UPDATE 语句的 SET 子句的赋值侧使用，通过将它们传递给 Update.values()；查看文档以获取示例。

+   **[postgresql] [功能]**

    添加了新的“数组字面量”构造函数 postgresql.array()。基本上是一个以 ARRAY[1,2,3]形式呈现的“元组”。

+   **[postgresql] [功能]**

    添加了对 PostgreSQL ONLY 关键字的支持，该关键字可以出现在 SELECT、UPDATE 或 DELETE 语句中对应的表中。该短语是使用 with_hint()建立的。感谢 Ryan Kelly

    参考：[#2506](https://www.sqlalchemy.org/trac/ticket/2506)

+   **[postgresql] [功能]**

    PostgreSQL 方言的“ischema_names”字典是“非官方”可定制的。意思是，新类型如 PostGIS 类型可以添加到这个字典中，PG 类型反射代码应该能够处理带有可变参数数量的简单类型。这里的功能之所以“非官方”，有三个原因：

    1.  这不是一个“官方”API。理想情况下，“官方”API 应该允许在方言或全局级别以一种通用方式调用自定义类型处理可调用对象。

    1.  这仅针对 PG 方言实现，特别是因为 PG 对自定义类型有广泛支持，而其他数据库后端不是这样。真正的 API 将在默认方言级别实现。

    1.  这里的反射代码仅针对简单类型进行了测试，可能在更复杂类型上存在问题。

    补丁由Éric Lemoine 提供。

### mysql

+   **[mysql] [feature]**

    将 TIME 类型添加到 mysql 方言，接受“fst”参数，这是最近 MySQL 版本的新“分数秒”指定符。数据类型将解释从驱动程序接收的微秒部分，但请注意，目前大多数/所有 MySQL DBAPI 都不支持返回此值。

    参考：[#2534](https://www.sqlalchemy.org/trac/ticket/2534)

+   **[mysql] [bug]**

    方言在第一次连接时不再发出昂贵的服务器排序查询，以及服务器大小写。这些功能仍然作为半私有功能可用。

    参考：[#2404](https://www.sqlalchemy.org/trac/ticket/2404)

### sqlite

+   **[sqlite] [feature]**

    SQLite 的日期和时间类型已经进行了改进，以支持更开放的输入和输出格式，使用基于名称的格式字符串和正则表达式。新的参数“microseconds”还提供了省略时间戳“微秒”部分的选项。感谢 Nathan Wright 对此工作和测试的贡献。

    参考：[#2363](https://www.sqlalchemy.org/trac/ticket/2363)

+   **[sqlite]**

    将`NCHAR`、`NVARCHAR`添加到 SQLite 方言的已识别类型名称列表以供反射使用。SQLite 返回给定类型的名称作为返回的名称。

    参考：[rc3addcc9ffad](https://www.sqlalchemy.org/trac/changeset/c3addcc9ffad)

### mssql

+   **[mssql] [feature]**

    SQL Server 方言可以给出数据库限定的模式名称，即“schema='mydatabase.dbo'”；反射操作将检测到这一点，将模式分割在“.”之间以单独获取所有者，并在反映“dbo”所有者内的目标之前发出“USE mydatabase”语句；然后恢复从 DB_NAME()返回的现有数据库。

+   **[mssql] [feature]**

    更新了对 mxodbc 驱动程序的支持；建议使用 mxodbc 3.2.1 以获得完全兼容性。

+   **[mssql] [bug]**

    移除了旧行为，即通过==将列与标量 SELECT 进行比较会强制转换为 SQL 服务器方言的 IN。这是隐式行为，在其他情况下会失败，因此被移除。依赖于此行为的代码需要修改为显式使用 column.in_(select)。

    参考：[#2277](https://www.sqlalchemy.org/trac/ticket/2277)

### oracle

+   **[oracle] [feature]**

    从 setinputsizes()集中排除的列的类型可以通过将字符串 DBAPI 类型名称列表发送到 exclude_setinputsizes 方言参数来自定义，该列表以前是固定的。该列表现在默认为 STRING、UNICODE，移除了 CLOB、NCLOB。

    参考：[#2561](https://www.sqlalchemy.org/trac/ticket/2561)

+   **[oracle] [bug]**

    当使用 quote=True 的 Column 生成同名的绑定参数时，将引用信息传递给 bindparam() 对象，就像在生成的 INSERT 和 UPDATE 语句中一样，以便完全支持未知的保留名称。

    参考：[#2437](https://www.sqlalchemy.org/trac/ticket/2437)

+   **[oracle] [bug]**

    Oracle 中的 CreateIndex 结构现在将索引的名称模式限定为父表的名称。以前，此名称被省略，导致索引在默认模式中创建，而不是表的模式。

### misc

+   **[feature] [access]**

    MS Access 方言已移至 Bitbucket 上的自己的项目中，利用了新的 SQLAlchemy 方言兼容性套件。该方言仍处于非常粗糙的状态，可能尚未准备好供一般使用，但现在已经具有极其基础的功能。[`bitbucket.org/zzzeek/sqlalchemy-access`](https://bitbucket.org/zzzeek/sqlalchemy-access)

+   **[feature] [firebird]**

    “startswith()” 运算符显示为 “STARTING WITH”，“~startswith()” 显示为 “NOT STARTING WITH”，使用 FB 的更有效的运算符。

    参考：[#2470](https://www.sqlalchemy.org/trac/ticket/2470)

+   **[feature] [firebird]**

    为 fdb 驱动程序添加了一个实验性方言，但由于无法构建 fdb 包，因此尚未经过测试。

    参考：[#2504](https://www.sqlalchemy.org/trac/ticket/2504)

+   **[bug] [firebird]**

    当尝试发出没有长度的 VARCHAR 时，引发 CompileError，与 MySQL 相同。

    参考：[#2505](https://www.sqlalchemy.org/trac/ticket/2505)

+   **[bug] [firebird]**

    Firebird 现在使用严格的“ansi 绑定规则”，以便绑定参数不会在语句的列子句中显示 - 它们会按照字面意思显示。

+   **[bug] [firebird]**

    当使用 DateTime 类型与 Firebird 时，支持将 datetime 作为 date 传递；其他方言也支持此功能。

+   **[moved] [maxdb]**

    MaxDB 方言，多年来一直无法使用，已移到待定的 Bitbucket 项目中，[`bitbucket.org/zzzeek/sqlalchemy-maxdb`](https://bitbucket.org/zzzeek/sqlalchemy-maxdb)。

### general

+   **[general] [removed]**

    “sqlalchemy.exceptions” 作为 “sqlalchemy.exc” 的同义词已完全移除。

    参考：[#2433](https://www.sqlalchemy.org/trac/ticket/2433)

+   **[general]**

    SQLAlchemy 0.8 现在面向 Python 2.5 及以上版本。不再支持 Python 2.4。

### orm

+   **[orm] [feature]**

    关系（relationship()）内部进行了重大重写，现在允许包含指向自身的列在复合外键内的连接条件。添加了一个用于非常专业化的 primaryjoin 条件的新 API，允许根据需要在表达式内联中放置注释函数 remote() 和 foreign() 处理基于 SQL 函数、CAST 等的条件。以前使用半私有的 _local_remote_pairs 方法的方案可以升级到这种新方法。

    另请参阅

    重写 _orm.relationship() 机制

    参考：[#1401](https://www.sqlalchemy.org/trac/ticket/1401)

+   **[orm] [feature]**

    新的独立函数 with_polymorphic() 提供了 query.with_polymorphic() 的功能，以独立形式提供。它可以应用于查询中的任何实体，包括作为联接的目标，在其中取代 “of_type()” 修改器。

    参考：[#2333](https://www.sqlalchemy.org/trac/ticket/2333)

+   **[orm] [feature]**

    在属性上的 of_type() 结构现在接受别名化（aliased()）类构造以及多态（with_polymorphic）构造，并且与 query.join()、any()、has() 以及同时也适用于 eager loaders 子查询加载（subqueryload()）、连接加载（joinedload()）、包含加载（contains_eager()）。

    参考：[#1106](https://www.sqlalchemy.org/trac/ticket/1106)，[#2438](https://www.sqlalchemy.org/trac/ticket/2438)

+   **[orm] [feature]**

    对于映射类的事件监听的改进允许指定未映射的类用于实例和映射器事件。当传递 propagate=True 标志时，建立的事件将自动设置在该类的子类上，当最终映射时，事件将为该类本身设置。

    参考：[#2585](https://www.sqlalchemy.org/trac/ticket/2585)

+   **[orm] [feature]**

    “延迟声明式反射”系统已经移动到声明式扩展本身中，使用新的 DeferredReflection 类。此类现在已经针对单表和联合表继承用例进行了测试。

    参考：[#2485](https://www.sqlalchemy.org/trac/ticket/2485)

+   **[orm] [feature]**

    添加了新的核心函数“inspect()”，它作为对映射器、对象、其他内容进行内省的通用网关。Mapper 和 InstanceState 对象已经增强，提供了公共 API，允许检查映射的属性，包括针对列绑定或关系绑定属性的过滤器，检查当前对象状态，属性历史记录等。

    参考：[#2208](https://www.sqlalchemy.org/trac/ticket/2208)

+   **[orm] [feature]**

    在 session.begin_nested() 中调用 rollback() 现在只会使那些在该事务范围内具有净更改的对象失效，即在刷新时脏对象或修改的对象。这允许 begin_nested() 的典型用例，即修改一小部分对象，保留不在子事务中修改的较大对象集合中的数据。

    参考：[#2452](https://www.sqlalchemy.org/trac/ticket/2452)

+   **[orm] [feature]**

    添加了实用功能 Session.enable_relationship_loading()，取代了 relationship.load_on_pending。然而，应该避免使用这两个功能。

    参考：[#2372](https://www.sqlalchemy.org/trac/ticket/2372)

+   **[orm] [feature]**

    增加了对 column_property()、relationship()、composite() 的 .info 字典参数的支持。所有 MapperProperty 类都可以在整体上使用自动创建的 .info 字典。

+   **[orm] [feature]**

    向/从映射集合中添加/移除 None 现在会生成属性事件。以前，在某些情况下，None 追加会被忽略。相关内容。

    参考：[#2229](https://www.sqlalchemy.org/trac/ticket/2229)

+   **[orm] [feature]**

    映射集合中存在 None 现在在刷新时会引发错误。以前，集合中的 None 值会被静默忽略。

    参考：[#2229](https://www.sqlalchemy.org/trac/ticket/2229)

+   **[orm] [feature]**

    Query.update() 方法现在对更新的表更加宽松。现在更好地支持普通的 Table 对象，并且可以使用一个加入继承的子类与 update() 一起使用；子类表将成为更新的目标，如果父表在 WHERE 子句中被引用，编译器将调用 UPDATE..FROM 语法来满足 WHERE 子句。如果在“values”字典中通过对象指定列，还支持 MySQL 的多表更新功能。PG 的 DELETE..USING 在 Core 中还不可用。

+   **[orm] [feature]**

    新的 session 事件 after_transaction_create 和 after_transaction_end 允许跟踪新的 SessionTransaction 对象。如果检查对象，则可以确定会话何时首次激活以及何时停用。

+   **[orm] [feature]**

    现在 Query 可以通过在使用的相应 TypeEngine 对象上设置标志“hashable=False”来加载包含不可哈希类型的实体/标量混合“元组”行。返回不可哈希类型（通常是列表）的自定义类型可以将此标志设置为 False。

    参考：[#2592](https://www.sqlalchemy.org/trac/ticket/2592)

+   **[orm] [feature]**

    现在 Query 默认会像 select() 一样“自动关联”。以前，在另一个查询中使用的 Query 需要显式调用 correlate() 方法才能将内部的表与外部关联起来。如常，correlate(None) 可以禁用关联。

    参考：[#2179](https://www.sqlalchemy.org/trac/ticket/2179)

+   **[orm] [feature]**

    现在在对象在 Session.add()、Session.merge() 等方法中建立在 Session.new 或 Session.identity_map 中后，会发出 after_attach 事件，以便在调用事件时这些集合中表示对象。添加了 before_attach 事件以适应需要在预附加对象时进行自动刷新的用例。

    参考：[#2464](https://www.sqlalchemy.org/trac/ticket/2464)

+   **[orm] [feature]**

    当在 flush 的“execute”部分中使用不受支持的方法时，会产生警告。这些是熟悉的方法 add()、delete() 等，以及在 mapper 级别 flush 事件中调用的集合和相关对象操作，如 after_insert()、after_update() 等。长期以来，已经明确记录了当 Session 在执行 flush 计划时被操作时，SQLAlchemy 无法保证结果，但用户仍在这样做，所以现在有了警告。也许将来 Session 将被增强以支持在 flush 中执行这些操作，但目前无法保证结果。

+   **[orm] [feature]**

    ORM 实体可以传递给核心 select() 构造，以及传递给 select() 的 select_from()、correlate() 和 correlate_except() 方法，它们将被解包为可选择项。

    参考：[#2245](https://www.sqlalchemy.org/trac/ticket/2245)

+   **[orm] [feature]**

    一些支持根据映射属性自动渲染关系连接条件的功能，使用核心 SQL 构造。例如，select([SomeClass]).where(SomeClass.somerelationship) 将从“someclass”中选择，并使用“somerelationship”的主连接作为 WHERE 子句。这改变了在核心 SQL 上下文中使用“SomeClass.somerelationship”之前的含义；以前，它会“解析”为父可选择项，这通常不太有用。也适用于 query.filter()。相关链接。

    参考：[#2245](https://www.sqlalchemy.org/trac/ticket/2245)

+   **[orm] [feature]**

    在 declarative_base() 中的类注册表现在是 WeakValueDictionary。因此，“Base”的子类如果没有被其他映射器/超类映射器引用，将被垃圾回收。查看此票证的下一个注释。

    参考：[#2526](https://www.sqlalchemy.org/trac/ticket/2526)

+   **[orm] [feature]**

    单继承声明子类之间的列冲突，无论是否使用混合类，都可以使用文档中描述的新的 @declared_attr 用法解决。

    参考：[#2472](https://www.sqlalchemy.org/trac/ticket/2472)

+   **[orm] [feature]**

    declared_attr 现在可以用于非混合类，尽管这通常只对单继承子类列冲突解决有用。

    参考：[#2472](https://www.sqlalchemy.org/trac/ticket/2472)

+   **[orm] [feature]**

    declared_attr 现在可以与不是 Column 或 MapperProperty 的属性一起使用；包括任何用户定义的值以及关联代理对象。

    参考：[#2517](https://www.sqlalchemy.org/trac/ticket/2517)

+   **[orm] [feature]**

    *非常有限*的支持，用于继承映射器在类本身被解除引用时进行垃圾回收。映射器不能拥有自己的表（即只有单个表继承），没有多态属性。这允许创建声明性映射类的临时子类的用例，在被单元测试解除引用时进行垃圾回收。

    参考：[#2526](https://www.sqlalchemy.org/trac/ticket/2526)

+   **[orm] [feature]**

    Declarative 现在通过字符串名称以及完整的模块限定名称维护类的注册表。现在可以根据模块限定的字符串在 relationship() 中查找具有相同名称的多个类。当多个类共享相同名称时，简单的类名称查找现在会引发一条信息性错误消息。

    参考：[#2338](https://www.sqlalchemy.org/trac/ticket/2338)

+   **[orm] [feature]**

    现在可以提供类绑定属性，这些属性会覆盖任何非 ORM 类型的列，而不仅仅是描述符。

    参考：[#2535](https://www.sqlalchemy.org/trac/ticket/2535)

+   **[orm] [feature]**

    向 Query.subquery() 添加了 with_labels 和 reduce_columns 关键字参数，以提供两种用于生成具有唯一命名列的查询的替代策略。

    参考：[#1729](https://www.sqlalchemy.org/trac/ticket/1729)

+   **[orm] [feature]**

    当对一个工具化集合的引用不再与父类相关联（因为到期/属性刷新/集合替换），但是接收到现在已分离的集合上的 append 或 remove 操作时，会发出警告。

    参考：[#2476](https://www.sqlalchemy.org/trac/ticket/2476)

+   **[orm] [bug]**

    如果两个表通过连接继承相关，并且 FK 依赖关系不是 inherit_condition 的一部分，则 ORM 在 flush 期间将执行额外的工作来确定 FK 依赖关系不重要，这样用户就不需要使用 use_alter 指令。

    参考：[#2527](https://www.sqlalchemy.org/trac/ticket/2527)

+   **[orm] [bug]**

    现在，instrumentation 事件 class_instrument()、class_uninstrument() 和 attribute_instrument() 仅对分配给 listen() 的类的后代类触发。以前，无论传递了什么“target”参数，事件监听器都会被分配为在所有情况下监听所有类。

    参考：[#2590](https://www.sqlalchemy.org/trac/ticket/2590)

+   **[orm] [bug]**

    在 with_polymorphic() 中，如果以任意顺序发送多级子类或者中间类缺失，则会正确按顺序生成 JOIN 并正确生成继承表。

    参考：[#1900](https://www.sqlalchemy.org/trac/ticket/1900)

+   **[orm] [bug]**

    在处理共享共同基类的子类实体链的 joined/subquery eager loading 中进行了改进，没有提供特定的“连接深度”。在检测到“循环”之前，会逐个链到每个子类映射器，而不是将基类视为“循环”的源。 

    参考：[#2481](https://www.sqlalchemy.org/trac/ticket/2481)

+   **[orm] [bug]**

    现在，Session.is_modified() 上的“passive”标志不再起作用。在所有情况下，is_modified() 只查看本地内存中的修改标志，并且不会发出任何 SQL 或调用加载器可调用/初始化器。

    参考：[#2320](https://www.sqlalchemy.org/trac/ticket/2320)

+   **[orm] [bug]**

    当使用 delete-orphan 级联与 one-to-many 或 many-to-many 且没有设置 single-parent=True 时发出的警告现在是一个错误。在任何情况下，ORM 在此警告后将无法正常运行。

    参考：[#2405](https://www.sqlalchemy.org/trac/ticket/2405)

+   **[orm] [bug]**

    在 flush 事件中发出的惰性加载（如 before_flush()、before_update() 等）现在将按照非事件代码中的方式运行，考虑到在惰性发出的查询中使用的主键/外键值。之前，特殊标志将被建立，这些标志将导致惰性加载基于父主键/外键值的“先前”值，特别是在 flush 中调用时；现在，以这种方式加载的信号现在被局限于工作单元实际需要以这种方式加载的位置。请注意，工作单元有时会在调用 before_update() 事件之前加载这些集合，因此“passive_updates”的使用与否可能会影响在 flush 事件中访问时集合是否表示“旧”或“新”数据，具体取决于惰性加载的时间。该更改在极小的可能性下与旧行为相关的用户事件代码依赖不兼容。

    参考：[#2350](https://www.sqlalchemy.org/trac/ticket/2350)

+   **[orm] [bug]**

    关于 flush 后额外状态的持续讨论，由于事件侦听器，任何从属性角度标记为“脏”的状态，通常是通过在 after_insert()、after_update() 等中设置列属性事件来完成的，现在在所有情况下都将重置“历史”标志，而不仅仅是那些在 flush 中的实例。这样做的效果是，在 flush 后，这种“脏”状态不会继续存在，并且不会产生 UPDATE 语句。发出了相应的警告；set_committed_state() 方法可用于在不产生历史事件的情况下分配对象上的属性。

    参考：[#2566](https://www.sqlalchemy.org/trac/ticket/2566)，[#2582](https://www.sqlalchemy.org/trac/ticket/2582)

+   **[orm] [bug]**

    修复了在 @declared_attr Column 和直接定义的 Column 之间逐渐发展的断开连接。在这两种情况下，Column 将应用于声明类的表，但不适用于联合继承子类的表。之前，直接定义的 Column 将被放置在基表和子表上，这通常不是所期望的。

    参考：[#2565](https://www.sqlalchemy.org/trac/ticket/2565)

+   **[orm] [bug]**

    当父类本身被映射到 join() 或 select() 语句时，Declarative 现在可以将在单表继承子类上声明的列传播到父类的表上，而不仅仅是通过连接继承直接或间接映射到表上。

    参考：[#2549](https://www.sqlalchemy.org/trac/ticket/2549)

+   **[orm] [bug]**

    当 uselist=False 与“dynamic”加载器结合时会发出错误。这在 0.7.9 中是一个警告。

+   **[orm] [removed]**

    ORM 的遗留“可变”系统，包括 MutableType 类以及 PickleType 和 postgresql.ARRAY 上的 mutable=True 标志已被移除。ORM 使用 sqlalchemy.ext.mutable 扩展检测原地突变，该扩展在 0.7 中引入。移除 MutableType 和相关构造从 SQLAlchemy 的内部删除了大量复杂性。由于在使用时会扫描 Session 的全部内容，这种方法性能较差。

    参考：[#2442](https://www.sqlalchemy.org/trac/ticket/2442)

+   **[orm] [removed]**

    已移除的已弃用标识符：

    +   allow_null_pks mapper() 参数（使用 allow_partial_pks）

    +   _get_col_to_prop() mapper 方法（使用 get_property_by_column()）

    +   Session.merge() 的 dont_load 参数（使用 load=True）

    +   sqlalchemy.orm.shard 模块（使用 sqlalchemy.ext.horizontal_shard）

+   **[orm] [moved]**

    InstrumentationManager 接口及整个相关的替代类实现系统现已移动到 sqlalchemy.ext.instrumentation。这是一个很少使用的系统，它给类的实现机制增加了显著的复杂性和开销。新的架构允许直到实际导入 InstrumentationManager 时才使用它，此时它被引导到核心中。

### 例子

+   **[例子]**

    Beaker 缓存示例已转换为使用 [dogpile.cache](https://dogpilecache.readthedocs.io/)。这是一个由 Beaker 的缓存内部创建者编写的新的缓存库，代表了一个大大改进的、简化的、现代化的缓存系统。

    参见

    Dogpile 缓存

    参考：[#2589](https://www.sqlalchemy.org/trac/ticket/2589)

### engine

+   **[engine] [feature]**

    现在可以将连接事件监听器与单个连接对象关联，而不仅仅是与 Engine 对象关联。

    参考：[#2511](https://www.sqlalchemy.org/trac/ticket/2511)

+   **[engine] [feature]**

    before_cursor_execute 事件会触发所谓的“_cursor_execute”事件，这些事件通常是主键绑定序列和默认生成的 SQL 语句的特殊执行情况，当使用 INSERT 时未使用 RETURNING 时单独调用。

    参考：[#2459](https://www.sqlalchemy.org/trac/ticket/2459)

+   **[engine] [feature]**

    测试套件使用的库已经重新安排，以便它们再次成为 SQLAlchemy 安装的一部分。此外，新的测试套件现在位于新的 sqlalchemy.testing.suite 包中。这是一个正在开发中的系统，希望为外部方言提供一个通用的测试套件。在 SQLAlchemy 之外维护的方言可以使用新的测试装置作为其自己测试的框架，并将免费获得一个针对方言的“兼容性”测试套件，包括一个改进的“要求”系统，其中可以为测试启用或禁用特定功能和功能。

+   **[engine] [feature]**

    添加了一种新的在进程中注册新方言而不使用入口点的系统。请参阅“注册新方言”文档。

    参考：[#2462](https://www.sqlalchemy.org/trac/ticket/2462)

+   **[engine] [feature]**

    默认情况下，“required”标志在 bindparam()上设置为 True，如果“value”或“callable”参数未显式传递，则会导致语句执行检查参数是否存在于最终绑定参数集合中，而不是隐式分配 None。

    参考：[#2556](https://www.sqlalchemy.org/trac/ticket/2556)

+   **[engine] [feature]**

    对“dialect” API 进行了各种 API 调整，以更好地支持高度专业化的系统，如 Akiban 数据库，包括更多的钩子以允许执行上下文访问类型处理器。

+   **[engine] [feature]**

    Inspector.get_primary_keys()已弃用；请使用 Inspector.get_pk_constraint()。感谢 Diana Clarke。

    参考：[#2422](https://www.sqlalchemy.org/trac/ticket/2422)

+   **[engine] [feature]**

    添加了一个新的 C 扩展模块“utils”，用于在有时间实现时提供额外的功能加速。

+   **[engine] [bug]**

    Inspector.get_table_names()的 order_by=”foreign_key”功能现在首先按 dependee 排序表，以与 util.sort_tables 和 metadata.sorted_tables 保持一致。

+   **[engine] [bug]**

    修复了一个错误，即如果数据库重新启动影响多个连接，则每个连接将单独调用池的新处理，即使只需要一个处理。

    参考：[#2522](https://www.sqlalchemy.org/trac/ticket/2522)

+   **[engine] [bug]**

    select().apply_labels()上的.c.属性的列名称现在基于<tablename>_<colkey>而不是<tablename>_<colname>，对于那些具有明确定义的.key 的列。

    参考：[#2397](https://www.sqlalchemy.org/trac/ticket/2397)

+   **[engine] [bug]**

    当 Table 上的 autoload_replace 标志为 False 时，将跳过引用已声明列的反射外键约束，假设在 Python 中声明的列将接管指定 Python ForeignKey 或 ForeignKeyConstraint 声明的任务。

+   **[engine] [bug]**

    ResultProxy 方法 inserted_primary_key，last_updated_params()，last_inserted_params()，postfetch_cols()，prefetch_cols()都断言给定的语句是一个已编译的构造，并且是一个适当的 insert()或 update()语句，否则引发 InvalidRequestError。

    参考：[#2498](https://www.sqlalchemy.org/trac/ticket/2498)

+   **[engine]**

    ResultProxy.last_inserted_ids 已被移除，替换为 inserted_primary_key。

### sql

+   **[sql] [feature]**

    添加了一个新方法`Engine.execution_options()`到`Engine`。该方法类似于`Connection.execution_options()`，它创建一个指向新选项集的父对象的副本。该方法可用于构建分片方案，其中每个引擎共享相同的底层连接池。该方法已针对 ORM 中的水平分片配方进行了测试。

    另请参阅

    `Engine.execution_options()`

+   **[sql] [feature]**

    在 Core 中对操作符系统进行了重大改组，允许在类型级别重新定义现有操作符以及添加新操作符。可以从现有类型创建新类型，这些新类型添加或重新定义导出到列表达式的操作，类似于 ORM 如何允许 comparator_factory。新架构将此功能移入 Core，以便在所有情况下一致可用，并且使用现有类型传播行为进行干净传播。

    参考：[#2547](https://www.sqlalchemy.org/trac/ticket/2547)

+   **[sql] [feature]**

    为了补充，现在类型可以提供“绑定表达式”和“列表达式”，允许在每列或每绑定级别的语句中在编译时注入 SQL 表达式。这适用于需要在 SQL 级别增强绑定和结果行为的类型的用例，而不是在 Python 级别。允许方案如透明加密/解密，使用 PostGIS 函数等。

    参考：[#1534](https://www.sqlalchemy.org/trac/ticket/1534), [#2547](https://www.sqlalchemy.org/trac/ticket/2547)

+   **[sql] [feature]**

    核心操作符系统现在包括 getitem 操作符，即 Python 中的方括号操作符。首先用于为 PostgreSQL ARRAY 类型提供索引和切片行为，并且还提供了一个钩子，用于最终用户定义自定义 __getitem__ 方案，这些方案可以应用于类型级别以及 ORM 级别的自定义操作符方案。lshift (<<)和 rshift (>>)也作为可选操作符受支持。

    请注意，此更改会导致 ORM 与 synonym()或其他“包装描述符”的方案结合使用的基于描述符的 __getitem__ 方案需要开始使用自定义比较器以保持此行为。

+   **[sql] [feature]**

    修改了用于确定用户定义运算符（即使用`op()`方法授予的运算符）的运算符优先级的规则。以前，在所有情况下都应用最小优先级，现在默认优先级为零，低于所有运算符，除了“逗号”（例如，在`func`调用的参数列表中使用）和“AS”，还可以通过`op()`方法上的“precedence”参数进行自定义。

    参考：[#2537](https://www.sqlalchemy.org/trac/ticket/2537)

+   **[sql] [feature]**

    向所有 String 类型添加了“collation”参数。当存在时，呈现为 COLLATE <collation>。这是为了支持现在多个数据库支持的 COLLATE 关键字，包括 MySQL、SQLite 和 PostgreSQL。

    参考：[#2276](https://www.sqlalchemy.org/trac/ticket/2276)

+   **[sql] [feature]**

    现在可以通过将 operators.custom_op()与 UnaryExpression()结合使用来使用自定义一元运算符。

+   **[sql] [feature]**

    加强了 GenericFunction 和 func.*，允许用户定义的 GenericFunction 子类通过类名自动在 func.*命名空间中可用，可选择使用包名，以及具有在 func.*中渲染名称与标识名称不同的功能。

+   **[sql] [feature]**

    现在 cast()和 extract()构造也将通过 func.*访问器生成，因为用户自然会尝试从 func.*访问这些名称，他们可能会按照预期的方式执行，即使返回的对象不是 FunctionElement。

    参考：[#2562](https://www.sqlalchemy.org/trac/ticket/2562)

+   **[sql] [feature]**

    现��可以使用新的 inspect()服务获取 Inspector 对象的实例，作为

    参考：[#2208](https://www.sqlalchemy.org/trac/ticket/2208)

+   **[sql] [feature]**

    现在 column_reflect 事件接受 Inspector 对象作为第一个参数，位于“table”之前。使用这个非常新的事件的 0.7 版本的代码将需要修改以添加“inspector”对象作为第一个参数。

    参考：[#2418](https://www.sqlalchemy.org/trac/ticket/2418)

+   **[sql] [feature]**

    现在结果集中的列定位行为默认区分大小写。多年来，SQLAlchemy 会对这些值进行不区分大小写的转换，可能是为了缓解像 Oracle 和 Firebird 这样的方言早期大小写敏感性问题。这些问题在更现代的版本中已经更清晰地解决，因此在标识符上调用 lower()的性能损失已经消除。可以通过在 create_engine()上设置“case_insensitive=False”来重新启用不区分大小写的比较。

    参考：[#2423](https://www.sqlalchemy.org/trac/ticket/2423)

+   **[sql] [feature]**

    当在 insert.values()或 update.values()中存在键不在目标表中时，现在发出的“未使用的列名”警告已经变成异常。

    参考：[#2415](https://www.sqlalchemy.org/trac/ticket/2415)

+   **[sql] [feature]**

    在 ForeignKey、ForeignKeyConstraint 中添加了“MATCH”子句，感谢 Ryan Kelly。

    参考：[#2502](https://www.sqlalchemy.org/trac/ticket/2502)

+   **[sql] [feature]**

    增加了对从表的别名进行 DELETE 和 UPDATE 的支持，这在查询中可能与其他地方的表相关联，感谢 Ryan Kelly。

    参考：[#2507](https://www.sqlalchemy.org/trac/ticket/2507)

+   **[sql] [feature]**

    select()现在具有 correlate_except()方法，自动关联除传递的所有可选择项之外的所有可选择项。

+   **[sql] [feature]**

    prefix_with()方法现在在 select()、insert()、update()、delete()的每个上都可用，具有相同的 API，接受多个前缀调用，以及“方言名称”，以便前缀可以限制为一种方言。

    参考：[#2431](https://www.sqlalchemy.org/trac/ticket/2431)

+   **[sql] [feature]**

    在 select()构造中添加了 reduce_columns()方法，使用 util.reduce_columns 实用程序函数替换内联列以删除等效列。reduce_columns()还添加了“with_only_synonyms”以限制仅对具有相同名称的列进行减少。已弃用的 fold_equivalents()功能已被移除。

    参考：[#1729](https://www.sqlalchemy.org/trac/ticket/1729)

+   **[sql] [feature]**

    重新设计了 startswith()、endswith()、contains()运算符，以更好地处理否定（NOT LIKE），并在编译时组装它们，以便它们的渲染 SQL 可以被修改，例如在 Firebird STARTING WITH 的情况下。

    参考：[#2470](https://www.sqlalchemy.org/trac/ticket/2470)

+   **[sql] [feature]**

    添加了一个钩子到渲染 CREATE TABLE 的系统中，通过针对新的 schema.CreateColumn 构造一个@compiles 函数，为每个列提供单独的渲染访问。

    参考：[#2463](https://www.sqlalchemy.org/trac/ticket/2463)

+   **[sql] [feature]**

    “标量”选择现在具有 WHERE 方法，以帮助生成构建。此外，关于如何 SS“关联”列的轻微调整；新方法不再将所选的基础表列赋予任何含义。这改进了一些相当奇特的情况，而那里的逻辑似乎没有任何目的。

+   **[sql] [feature]**

    当首次使用构造为引用多个远程表的 ForeignKeyConstraint()时，将引发显式错误。

    参考：[#2455](https://www.sqlalchemy.org/trac/ticket/2455)

+   **[sql] [feature]**

    添加了`ColumnOperators.notin_()`，`ColumnOperators.notlike()`，`ColumnOperators.notilike()` 到 `ColumnOperators`.

    参考：[#2580](https://www.sqlalchemy.org/trac/ticket/2580)

+   **[sql] [change]**

    如果指定了长度，Text() 类型会呈现给定的长度。

+   **[sql] [changed]**

    expression.sql 中的大多数类不再以下划线开头，即 Label、SelectBase、Generative、CompareMixin。_BindParamClause 也被重命名为 BindParameter。这些类的旧下划线名称将在可预见的未来保持可用作为同义词。

+   **[sql] [bug]**

    修复了传递给 `Compiler.process()` 的关键字参数不会传播到 SELECT 语句的 columns 子句中的列表达式的 bug。特别是当由依赖于特殊标志的自定义编译方案使用时，会出现这种情况。

    参考：[#2593](https://www.sqlalchemy.org/trac/ticket/2593)

+   **[sql] [bug] [orm]**

    `select()` 的自动相关特性，以及由此引发的 `Query` 的特性，将不会对直接在包含 SELECT 的 FROM 列表中呈现的 SELECT 语句产生影响。 SQL 中的相关性仅适用于诸如 WHERE、ORDER BY、columns 子句中的列表达式。

    参考：[#2595](https://www.sqlalchemy.org/trac/ticket/2595)

+   **[sql] [bug]**

    调整了列优先级，将“concat”和“match”运算符移动到与“is”、“like”等运算符相同的位置；当与“IS”一起使用时，这有助于括号渲染。

    参考：[#2564](https://www.sqlalchemy.org/trac/ticket/2564)

+   **[sql] [bug]**

    将列表达式应用于带有或不带有其他修改构造的标签的 select 语句，将不再将该表达式“定位”到底层列；这会影响依赖于列定位以检索结果的 ORM 操作。也就是说，像 query(User.id, User.id.label(‘foo’))这样的查询现在将分别跟踪每个“User.id”表达式的值，而不是将它们合并在一起。不过，不希望任何用户受到影响；但是，如果 select()与 query.from_statement()结合使用，并尝试加载完全组合的 ORM 实体，而 select()命名了具有任意.label()名称的 Column 对象，那么可能无法按预期方式运行，因为这些对象将不再定位到该实体映射的 Column 对象。

    参考：[#2591](https://www.sqlalchemy.org/trac/ticket/2591)

+   **[sql] [bug]**

    修复了将“default”参数解释为可调用项时，不将 ExecutionContext 传递给关键字参数的问题。

    参考：[#2520](https://www.sqlalchemy.org/trac/ticket/2520)

+   **[sql] [bug]**

    当它们直接引用绑定到 Table 的列对象（即不仅是字符串列名）并且引用一个且仅一个 Table 时，UniqueConstraint、ForeignKeyConstraint、CheckConstraint 和 PrimaryKeyConstraint 现在会自动附加到它们的父表上。在 0.8 之前，此行为仅适用于 UniqueConstraint 和 PrimaryKeyConstraint，而不适用于 ForeignKeyConstraint 或 CheckConstraint。

    参考：[#2410](https://www.sqlalchemy.org/trac/ticket/2410)

+   **[sql] [bug]**

    TypeDecorator 现在默认包含一个基于“impl”类型的通用 repr()。对于指定了自定义 __init__ 方法的那些 TypeDecorator 类，这是一种行为变更；这些类型将需要重新定义 __repr__()，如果需要 __repr__()提供一个忠实的构造函数表示。

    参考：[#2594](https://www.sqlalchemy.org/trac/ticket/2594)

+   **[sql] [bug]**

    column.label(None)现在生成一个匿名标签，而不是返回列对象本身，这与 label(column, None)的行为一致。

    参考：[#2168](https://www.sqlalchemy.org/trac/ticket/2168)

+   **[sql] [removed]**

    已删除了长期弃用且不起作用的`create_engine()`和`String`上的`assert_unicode`标志。

### postgresql

+   **[postgresql] [feature]**

    postgresql.ARRAY 具有一个可选的“dimension”参数，将为数组分配一个特定数量的维度，这将以 DDL 形式渲染为 ARRAY[][]…，还提高了绑定/结果处理的性能。

    参考：[#2441](https://www.sqlalchemy.org/trac/ticket/2441)

+   **[postgresql] [feature]**

    postgresql.ARRAY 现在支持索引和切片。Python [] 运算符可用于所有类型为 ARRAY 的 SQL 表达式；可以传递整数或简单切片。这些切片也可以在 UPDATE 语句的 SET 子句中的赋值方面使用，通过将它们传递给 Update.values()；请参阅文档以获取示例。

+   **[postgresql] [feature]**

    增加了新的“数组文字”构造 postgresql.array()。基本上是一个渲染为 ARRAY[1,2,3] 的“元组”。

+   **[postgresql] [feature]**

    增加了对 PostgreSQL ONLY 关键字的支持，该关键字可以在 SELECT、UPDATE 或 DELETE 语句中对应于表。该短语是使用 with_hint() 建立的。感谢 Ryan Kelly

    参考：[#2506](https://www.sqlalchemy.org/trac/ticket/2506)

+   **[postgresql] [feature]**

    PostgreSQL 方言的 “ischema_names” 字典是“非官方”可定制的。这意味着，诸如 PostGIS 类型之类的新类型可以添加到此字典中，并且 PG 类型反射代码应该能够处理具有不同数量参数的简单类型。这里的功能性是“非官方”的，原因有三：

    1.  这不是一个“官方”API。理想情况下，“官方”API 应该以一种通用的方式允许在方言或全局级别调用自定义类型处理可调用函数。

    1.  这仅在 PG 方言中实现，特别是因为 PG 对自定义类型有广泛支持，而其他数据库后端则不然。一个真正的 API 应该在默认方言级别实现。

    1.  这里的反射代码仅针对简单类型进行了测试，可能存在与更复杂类型相关的问题。

    补丁由 Éric Lemoine 提供。

### mysql

+   **[mysql] [feature]**

    在 mysql 方言中添加了 TIME 类型，接受“fst”参数，这是最近 MySQL 版本的新“分秒”指定符。数据类型将解释从驱动程序接收的微秒部分，但请注意，目前大多数/所有 MySQL DBAPI 不支持返回此值。

    参考：[#2534](https://www.sqlalchemy.org/trac/ticket/2534)

+   **[mysql] [bug]**

    方言不再在首次连接时发出昂贵的服务器排序查询，以及服务器大小写查询。这些函数仍然作为半私有函数可用。

    参考：[#2404](https://www.sqlalchemy.org/trac/ticket/2404)

### sqlite

+   **[sqlite] [feature]**

    SQLite 的日期和时间类型已经进行了改进，以支持更开放的输入和输出格式，使用基于名称的格式字符串和正则表达式。一个新的参数“microseconds”还提供了省略时间戳的“微秒”部分的选项。感谢 Nathan Wright 在这方面的工作和测试。

    参考：[#2363](https://www.sqlalchemy.org/trac/ticket/2363)

+   **[sqlite]**

    将 `NCHAR`、`NVARCHAR` 添加到 SQLite 方言识别的类型名称列表中以供反射使用。SQLite 返回给定类型的名称作为返回的名称。

    参考：[rc3addcc9ffad](https://www.sqlalchemy.org/trac/changeset/c3addcc9ffad)

### mssql

+   **[mssql] [功能]**

    SQL Server 方言可以给出数据库限定的模式名称，即“schema='mydatabase.dbo'”；反射操作将检测到这一点，将模式分割在“.”之间以单独获取所有者，并在反射目标内部发出“USE mydatabase”语句；然后恢复从 DB_NAME()返回的现有数据库。

+   **[mssql] [功能]**

    更新了对 mxodbc 驱动程序的支持；建议使用 mxodbc 3.2.1 以实现完全兼容。

+   **[mssql] [错误]**

    删除了旧行为，即通过==将列与标量 SELECT 进行比较时，会将其强制转换为 SQL 服务器方言中的 IN。这是隐式行为，在其他情况下会失败，因此已删除。依赖于此行为的代码需要修改为显式使用 column.in_(select)。

    参考：[#2277](https://www.sqlalchemy.org/trac/ticket/2277)

### oracle

+   **[oracle] [功能]**

    可以通过将字符串 DBAPI 类型名称列表发送到 exclude_setinputsizes 方言参数来自定义从 setinputsizes()集中排除的列的类型。此列表以前是固定的。该列表现在默认为 STRING，UNICODE，从列表中删除了 CLOB，NCLOB。

    参考：[#2561](https://www.sqlalchemy.org/trac/ticket/2561)

+   **[oracle] [错误]**

    当生成与 Column 具有 quote=True 的同名绑定参数时，现在会将引用信息传递给 bindparam()对象，就像在生成的 INSERT 和 UPDATE 语句中一样，以便完全支持未知的保留名称。

    参考：[#2437](https://www.sqlalchemy.org/trac/ticket/2437)

+   **[oracle] [错误]**

    Oracle 中的 CreateIndex 构造现在将索引的名称模式限定为父表的名称。以前，此名称被省略，显然会在默认模式中创建索引，而不是表的模式。

### 杂项

+   **[功能] [access]**

    MS Access 方言已移至 Bitbucket 上的自己的项目，利用了新的 SQLAlchemy 方言兼容性套件。该方言仍处于非常初步的形式，可能尚未准备好供一般使用，但现在具有*极其*基本的功能。[`bitbucket.org/zzzeek/sqlalchemy-access`](https://bitbucket.org/zzzeek/sqlalchemy-access)

+   **[功能] [firebird]**

    “startswith()”运算符呈现为“STARTING WITH”，“~startswith()”呈现为“NOT STARTING WITH”，使用了 FB 更高效的运算符。

    参考：[#2470](https://www.sqlalchemy.org/trac/ticket/2470)

+   **[功能] [firebird]**

    添加了一个用于 fdb 驱动程序的实验性方言，但由于无法构建 fdb 包，因此未经测试。

    参考：[#2504](https://www.sqlalchemy.org/trac/ticket/2504)

+   **[错误] [firebird]**

    当尝试发出没有长度的 VARCHAR 时，会引发 CompileError，与 MySQL 相同。

    参考：[#2505](https://www.sqlalchemy.org/trac/ticket/2505)

+   **[错误] [firebird]**

    Firebird 现在使用严格的“ansi 绑定规则”，这样绑定的参数不会在语句的列子句中呈现 - 它们会被字面呈现。

+   **[错误] [firebird]**

    当使用 DateTime 类型与 Firebird 时，支持将 datetime 作为 date 传递；其他方言也支持这一点。

+   **[移动] [maxdb]**

    MaxDB 方言，已经多年没有功能了，被移出到一个待定的 bitbucket 项目，[`bitbucket.org/zzzeek/sqlalchemy-maxdb`](https://bitbucket.org/zzzeek/sqlalchemy-maxdb)。
