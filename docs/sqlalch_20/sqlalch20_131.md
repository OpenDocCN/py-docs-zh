# 1.0 变更日志

> 原文：[`docs.sqlalchemy.org/en/20/changelog/changelog_10.html`](https://docs.sqlalchemy.org/en/20/changelog/changelog_10.html)

## 1.0.19

发布日期：2017 年 8 月 3 日

### oracle

+   **[oracle] [performance] [bug] [py2k]**

    修复了由于修复了[#3937](https://www.sqlalchemy.org/trac/ticket/3937)引起的性能回退，其中 cx_Oracle 在 5.3 版本中从其命名空间中删除了`.UNICODE`符号，这被解释为 cx_Oracle 的“WITH_UNICODE”模式被无条件地打开，从而在 SQLAlchemy 端调用函数将所有字符串无条件地转换为 unicode 并导致性能影响。实际上，根据 cx_Oracle 的作者，自 5.1 起，“WITH_UNICODE”模式已经被完全删除，因此不再需要昂贵的 unicode 转换函数，并且如果在 Python 2 下检测到 cx_Oracle 5.1 或更高版本，则会禁用它们。还恢复了在[#3937](https://www.sqlalchemy.org/trac/ticket/3937)中删除的有关“WITH_UNICODE”模式的警告。

    引用：[#4035](https://www.sqlalchemy.org/trac/ticket/4035)

## 1.0.18

发布日期：2017 年 7 月 24 日

### oracle

+   **[oracle] [bug]**

    通过发现 cx_Oracle 5.3 现在似乎在构建中硬编码了此标志，从而暴露了 cx_Oracle 的 WITH_UNICODE 模式的修复；使用此模式的内部方法未使用正确的签名。

    引用：[#3937](https://www.sqlalchemy.org/trac/ticket/3937)

### tests

+   **[tests] [bug] [py3k]**

    修复了与 Python 3.6.2 更改不兼容的测试装置中的问题，涉及到上下文管理器。

    引用：[#4034](https://www.sqlalchemy.org/trac/ticket/4034)

## 1.0.17

发布日期：2017 年 1 月 17 日

### orm

+   **[orm] [bug]**

    修复了针对多态继承也在使用的情况下对多个实体进行连接贪婪加载时会抛出“'NoneType'对象没有'isa'属性”的错误的错误。该问题是由于修复[#3611](https://www.sqlalchemy.org/trac/ticket/3611)引入的。

    引用：[#3884](https://www.sqlalchemy.org/trac/ticket/3884)

### misc

+   **[bug] [py3k]**

    修复了与没有'r'修饰符的转义字符串相关的 Python 3.6 DeprecationWarnings，并为 Python 3.6 添加了测试覆盖范围。

    引用：[#3886](https://www.sqlalchemy.org/trac/ticket/3886)

## 1.0.16

发布日期：2016 年 11 月 15 日

### orm

+   **[orm] [bug]**

    修复了`Session.bulk_update_mappings()`中一个备用命名的主键属性无法正确跟踪到 UPDATE 语句的错误。

    引用：[#3849](https://www.sqlalchemy.org/trac/ticket/3849)

+   **[orm] [bug]**

    修复了对多态加载映射器的连接贪婪加载将失败的错误，其中多态性设置为未映射表达式（如 CASE 表达式）的情况。

    引用：[#3800](https://www.sqlalchemy.org/trac/ticket/3800)

+   **[orm] [bug]**

    修复了一个 bug，当通过`Session.bind_mapper()`、`Session.bind_table()`或构造函数发送给 Session 的无效绑定时，引发的 ArgumentError 未能正确引发。

    参考：[#3798](https://www.sqlalchemy.org/trac/ticket/3798)

+   **[orm] [错误]**

    修复了`Session.bulk_save()`中的 bug，其中 UPDATE 与实现版本 id 计数器的映射结合使用时无法正常工作。

    参考：[#3781](https://www.sqlalchemy.org/trac/ticket/3781)

+   **[orm] [错误]**

    修复了当首次调用 mapper 属性或其他 ORM 构造添加到 mapper/class 后，`Mapper.attrs`、`Mapper.all_orm_descriptors`和其他派生属性无法刷新的 bug。

    参考：[#3778](https://www.sqlalchemy.org/trac/ticket/3778)

### mssql

+   **[mssql] [错误]**

    更改了用于获取“默认模式名称”的查询，从查询数据库主体表的查询到使用“schema_name()”函数，因为已报告在 Azure Data Warehouse 版本上前一系统不可用的问题。希望这将最终在所有 SQL Server 版本和认证样式上正常工作。

    参考：[#3810](https://www.sqlalchemy.org/trac/ticket/3810)

+   **[mssql] [错误]**

    更新了 pyodbc 的服务器版本信息方案，使用 SQL Server SERVERPROPERTY()，而不是依赖于 pyodbc.SQL_DBMS_VER，后者在 FreeTDS 中仍然不可靠。

    参考：[#3814](https://www.sqlalchemy.org/trac/ticket/3814)

+   **[mssql] [错误]**

    将错误代码 20017“服务器意外的 EOF”添加到导致连接池重置的断开异常列表中。感谢 Ken Robbins 的拉取请求。

    参考：[#3791](https://www.sqlalchemy.org/trac/ticket/3791)

+   **[mssql] [错误]**

    修复了 pyodbc 方言中的 bug（以及大部分不起作用的 adodbapi 方言中的 bug），其中密码或用户名字段中存在的分号可能被解释为另一个标记的分隔符；当分号存在时，现在对值进行引用。

    参考：[#3762](https://www.sqlalchemy.org/trac/ticket/3762)

### 杂项

+   **[错误] [orm.declarative]**

    修复了设置单表继承子类的 bug，该子类包括额外列，会破坏映射表的外键集合，从而干扰关系的初始化。

    参考：[#3797](https://www.sqlalchemy.org/trac/ticket/3797)

## 1.0.15

发布日期：2016 年 9 月 1 日

### orm

+   **[orm] [错误]**

    修复了子查询加载中的错误，其中对“of_type()”对象的子查询加载链接到第二个普通映射类的子查询加载，或者几个“of_type()”属性的较长链，将无法正确链接连接。

    参考：[#3773](https://www.sqlalchemy.org/trac/ticket/3773), [#3774](https://www.sqlalchemy.org/trac/ticket/3774)

### sql

+   **[sql] [错误]**

    修复了`Table`中的错误，其中内部方法`_reset_exported()`会破坏对象的状态。此方法用于可选择对象，并在某些情况下由 ORM 调用；错误的映射配置可能导致 ORM 在`Table`对象上调用此方法。

    参考：[#3755](https://www.sqlalchemy.org/trac/ticket/3755)

### mysql

+   **[mysql] [错误]**

    增加了对解析 MySQL/Connector 中布尔值和整数参数的支持，这些参数位于 URL 查询字符串中：connection_timeout, connect_timeout, pool_size, get_warnings, raise_on_warnings, raw, consume_results, ssl_verify_cert, force_ipv6, pool_reset_session, compress, allow_local_infile, use_pure。

    参考：[#3787](https://www.sqlalchemy.org/trac/ticket/3787)

### 杂项

+   **[错误] [扩展]**

    修复了`sqlalchemy.ext.baked`中的错误，其中由于变量作用域问题，在涉及多个子查询加载器时，解开子查询加载器查询的失败。感谢 Mark Hahnenberg 提交的拉取请求。

    参考：[#3743](https://www.sqlalchemy.org/trac/ticket/3743)

## 1.0.14

发布日期：2016 年 7 月 6 日

### 示例

+   **[示例] [错误]**

    修复了在 examples/vertical/dictlike-polymorphic.py 示例中发生的回归，导致无法运行。

    参考：[#3704](https://www.sqlalchemy.org/trac/ticket/3704)

### 引擎

+   **[引擎] [错误] [postgresql]**

    修复了跨模式外键反射与`MetaData.schema`参数结合使用时的错误，其中在“默认”模式中存在的引用表将失败，因为没有办法指示具有“空白”模式的`Table`。特殊符号`sqlalchemy.schema.BLANK_SCHEMA`已添加为`Table.schema`和`Sequence.schema`的可用值，指示应强制模式名称为`None`，即���指定了`MetaData.schema`。

    参考：[#3716](https://www.sqlalchemy.org/trac/ticket/3716)

### sql

+   **[sql] [错误]**

    修复了 SQL 数学否定运算符中的问题，表达式的类型不再是原始表达式的数值类型。这会导致确定结果集行为的类型问题。

    参考：[#3735](https://www.sqlalchemy.org/trac/ticket/3735)

+   **[sql] [bug]**

    修复了一个 bug，即由于 1.0 系列向`__slots__`的过渡，导致 sqlalchemy.util.Properties 的`__getstate__` / `__setstate__`方法无法正常工作。该问题可能影响一些第三方应用程序。感谢 Pieter Mulder 提交的拉取请求。

    参考：[#3728](https://www.sqlalchemy.org/trac/ticket/3728)

+   **[sql] [bug]**

    `FromClause.count()`将在 1.1 版本��被弃用。该函数使用表中的任意列，不可靠；对于 Core 使用，应优先使用`func.count()`。

    参考：[#3724](https://www.sqlalchemy.org/trac/ticket/3724)

+   **[sql] [bug]**

    修复了`CTE`结构中的一个 bug，当使用 union 时会导致它无法正确克隆，这在递归 CTE 中很常见。不正确的克隆会导致在各种 ORM 上下文中使用 CTE 时出现错误，比如`column_property()`。

    参考：[#3722](https://www.sqlalchemy.org/trac/ticket/3722)

+   **[sql] [bug]**

    修复了一个 bug，即`Table.tometadata()`会为每个具有`unique=True`参数的`Column`对象创建重复的`UniqueConstraint`。

    参考：[#3721](https://www.sqlalchemy.org/trac/ticket/3721)

### postgresql

+   **[postgresql] [bug]**

    修复了一个 bug，即 PostgreSQL 方言未深入检查`TypeDecorator`和`Variant`类型，以确定是否应该渲染 SMALLSERIAL 或 BIGSERIAL 而不是 SERIAL。

    参考：[#3739](https://www.sqlalchemy.org/trac/ticket/3739)

### oracle

+   **[oracle] [bug]**

    修复了`Select.with_for_update.of`中的一个 bug，在 Oracle 的“rownum”方法中，LIMIT/OFFSET 无法适应“OF”子句中的表达式，这些表达式必须在顶层引用子查询中的表达式。如果需要，这些表达式现在将添加到子查询中。

    参考：[#3741](https://www.sqlalchemy.org/trac/ticket/3741)

## 1.0.13

发布日期：2016 年 5 月 16 日

### orm

+   **[orm] [bug]**

    修复了`Query.update()`和`Query.delete()`中“evaluate”策略中的 bug，该 bug 无法适应具有“callable”值的绑定参数，当通过关系沿着 many-to-one 等式表达式进行过滤时会发生这种情况。

    参考：[#3700](https://www.sqlalchemy.org/trac/ticket/3700)

+   **[orm] [bug]**

    修复了一个 bug，即在使用深层类继承层次结构与多个映射器配置步骤同时使用时，用于反向引用的事件侦听器可能会被错误地应用多次。

    参考：[#3710](https://www.sqlalchemy.org/trac/ticket/3710)

+   **[orm] [bug]**

    修复了一个 bug，即将`text()`构造传递给`Query.group_by()`方法会引发错误，而不是将对象解释为 SQL 片段。

    参考：[#3706](https://www.sqlalchemy.org/trac/ticket/3706)

+   **[orm] [bug]**

    匿名标签应用于传递给`column_property()`的`func`构造，因此如果同一属性被引用两次作为列表达式，则名称将被去重，从而避免“模糊列”错误。以前，需要应用`.label(None)`才能去除匿名化的名称。

    参考：[#3663](https://www.sqlalchemy.org/trac/ticket/3663)

+   **[orm] [bug]**

    修复了在 1.0 系列中出现的 ORM 加载中的回归，其中对于缺少的预期列引发的异常将错误地是`NoneType`错误，而不是预期的`NoSuchColumnError`。

    参考：[#3658](https://www.sqlalchemy.org/trac/ticket/3658)

### 示例

+   **[examples] [bug]**

    将“有向图”示例更改为不再将节点的整数标识符视为重要；“更高”/“更低”引用现在允许双向的相互边。

    参考：[#3698](https://www.sqlalchemy.org/trac/ticket/3698)

### sql

+   **[sql] [bug]**

    修复了在使用`Engine`时，当`case_sensitive=False`时，结果集无法正确处理结果集中的重复列名的错误，导致在 1.0 版本中执行语句时出错，并阻止 1.1 版本中“模糊列”异常的功能。

    参考：[#3690](https://www.sqlalchemy.org/trac/ticket/3690)

+   **[sql] [bug]**

    修复了 EXISTS 表达式的否定在结果中未正确类型化为布尔值的 bug，并且在 SELECT 列表中也无法匿名别名，就像对非否定的 EXISTS 结构一样。

    引用：[#3682](https://www.sqlalchemy.org/trac/ticket/3682)

+   **[sql] [bug]**

    修复了当 `Insert.values()` 调用时使用参数映射列表而不是单个参数映射时，“未使用的列名”异常未能被引发的 bug。感谢 Athena Yao 提供的拉取请求。

    引用：[#3666](https://www.sqlalchemy.org/trac/ticket/3666)

### postgresql

+   **[postgresql] [bug]**

    添加了对错误字符串“SSL error: decryption failed or bad record mac”的断开连接检测支持。感谢 Iuri de Silvio 提供的拉取请求。

    引用：[#3715](https://www.sqlalchemy.org/trac/ticket/3715)

### mssql

+   **[mssql] [bug]**

    修复了在 SQL Server 中用于 OFFSET 查询的 ROW_NUMBER OVER 子句会不适当地替换语句中与语句的 ORDER BY 条件重叠的普通列的 bug。

    引用：[#3711](https://www.sqlalchemy.org/trac/ticket/3711)

+   **[mssql] [bug] [oracle]**

    修复了 1.0 系列中出现的回归问题，该问题会导致 Oracle 和 SQL Server 方言在将 SELECT 包装在子查询中以提供 LIMIT/OFFSET 行为时不正确地计算结果集列，原始的 SELECT 语句引用了同一列多次，例如一个列和该列的一个标签。这个问题与 [#3658](https://www.sqlalchemy.org/trac/ticket/3658) 相关，在发生错误时，它也会导致一个 `NoneType` 错误，而不是报告无法定位列。

    引用：[#3657](https://www.sqlalchemy.org/trac/ticket/3657)

### oracle

+   **[oracle] [bug]**

    修复了 cx_Oracle 连接过程中的一个 bug，当用户、密码或 dsn 中有一个为空时，会导致 TypeError。这阻止了对 Oracle 数据库的外部认证，并阻止连接到默认的 dsn。现在，连接字符串 oracle:// 将使用操作系统用户名登录到默认的 dsn，相当于使用 sqlplus 进行连接时使用 ‘/’。

    引用：[#3705](https://www.sqlalchemy.org/trac/ticket/3705)

+   **[oracle] [bug]**

    修复了主要由 Oracle 使用的结果代理中的一个 bug，当二进制和其他 LOB 类型参与时，如果使用了查询/语句缓存，类型级别的结果处理器，特别是二进制类型本身所需的处理器以及任何其他处理器，在第一次运行语句后会丢失，因为它被从缓存的结果元数据中删除了。

    引用：[#3699](https://www.sqlalchemy.org/trac/ticket/3699)

### 杂项

+   **[bug] [py3k]**

    修复了“to_list”转换中的错误，其中单个字节对象将转换为单个字符的列表。这将影响到使用`Query.get()`方法的主键是字节对象的情况。

    参考：[#3660](https://www.sqlalchemy.org/trac/ticket/3660)

## 1.0.12

发布日期：2016 年 2 月 15 日

### orm

+   **[orm] [bug]**

    在`Session.merge()`中修复了一个错误，其中具有复合主键的对象，其中一些但不是所有 PK 字段的值，将发出一个 SELECT 语句，将内部 NEVER_SET 符号泄漏到查询中，而不是检测到此对象没有可搜索的主键，并且不应发出 SELECT。

    参考：[#3647](https://www.sqlalchemy.org/trac/ticket/3647)

+   **[orm] [bug]**

    自 0.9 以来的固定回归，0.9 风格的加载器选项系统未能适应单个查询中的多个`undefer_group()`加载器选项。现在，多个`undefer_group()`选项将被考虑，即使是对同一实体也是如此。

    参考：[#3623](https://www.sqlalchemy.org/trac/ticket/3623)

### engine

+   **[engine] [bug] [mysql]**

    重新审视[#2696](https://www.sqlalchemy.org/trac/ticket/2696)，该问题在 1.0.10 版首次发布，试图通过在已失败的事务回滚时发出警告来解决 Python 2 缺乏异常上下文报告的问题；当尝试回滚已经失败的事务时，此问题继续在 MySQL 后端发生，与意外丢失的保存点相结合，然后在尝试回滚时引发“没有这样的保存点”错误，模糊了原始条件。

    此方法已推广到 Core“安全重新引发”功能，该功能在任何出现事务因试图提交而产生错误而回滚的地方在 ORM 和 Core 中都会发生，包括由`Session`和`Connection`提供的上下文管理器，并且对“RELEASE SAVEPOINT”等操作也进行了处理。以前，此修复仅适用于 ORM 刷新/提交过程中的特定路径；现在它也适用于所有事务上下文管理器。

    参考：[#2696](https://www.sqlalchemy.org/trac/ticket/2696)

### sql

+   **[sql] [bug]**

    修复了在将`insert()`、`update()`或`delete()`构造编译为字符串 SQL 时，“literal_binds”标志未传播的问题。感谢 Tim Tate 的拉取请求。

    参考：[#3643](https://www.sqlalchemy.org/trac/ticket/3643)

+   **[sql] [bug]**

    修复了在意外使用 Python `__contains__` 覆盖与列表达式（例如通过 `'x' in col` 使用）会导致 ARRAY 类型无限循环的问题，因为 Python 将此推迟到 `__getitem__` 访问，而此类型永远不会引发。总体上，所有对 `__contains__` 的使用现在都会引发 NotImplementedError。

    参考：[#3642](https://www.sqlalchemy.org/trac/ticket/3642)

+   **[sql] [bug]**

    修复了在 0.9 系列左右出现的`Table`元数据构造中的错误，向一个反序列化的`Table`添加列会导致无法正确建立‘c’集合中的`Column`，从而导致诸如 ORM 配置等领域的问题。这可能会影响到 `extend_existing` 等用例。

    参考：[#3632](https://www.sqlalchemy.org/trac/ticket/3632)

### postgresql

+   **[postgresql] [bug]**

    修复了`text()`构造中的错误，其中双冒号表达式无法正确转义，例如 `some\:\:expr`，这在渲染 PostgreSQL 风格的 CAST 表达式时最常见。

    参考：[#3644](https://www.sqlalchemy.org/trac/ticket/3644)

### mssql

+   **[mssql] [bug]**

    修复了在 MSSQL 上针对日期时间值使用`extract()`函数的语法；关键字周围的引号被移除。感谢 Guillaume Doumenc 的拉取请求。

    参考：[#3624](https://www.sqlalchemy.org/trac/ticket/3624)

+   **[mssql] [bug] [firebird]**

    修复了 1.0 版本中的回归问题，即对通过纯文本或通过`text()`构造发出的 UPDATE 或 DELETE 语句的 cursor.rowcount 的急切获取不再调用，影响那些在关闭游标后擦除 cursor.rowcount 的驱动程序，例如 SQL Server ODBC 和 Firebird 驱动程序。

    参考：[#3622](https://www.sqlalchemy.org/trac/ticket/3622)

### oracle

+   **[oracle] [bug] [jython]**

    修复了 Jython Oracle 编译器中的一个小问题，涉及“RETURNING”的呈现，这允许这个当前不受支持/未经测试的方言在 1.0 系列中基本工作。感谢 Carlos Rivas 的拉取请求。

    参考：[#3621](https://www.sqlalchemy.org/trac/ticket/3621)

### 杂项

+   **[bug] [py3k]**

    修复了一些异常重新引发场景会将异常附加到自身作为“cause”的错误；虽然 Python 3 解释器可以接受这种情况，但在 iPython 中可能会导致无限循环。

    参考：[#3625](https://www.sqlalchemy.org/trac/ticket/3625)

## 1.0.11

发布日期：2015 年 12 月 22 日

### orm

+   **[orm] [bug]**

    修复了 1.0.10 中由于修复[#3593](https://www.sqlalchemy.org/trac/ticket/3593)而引起的回归，其中为 poly_subclass->class->poly_baseclass 连接添加的多态 joinedload 检查将对 class->poly_subclass->class 的情况失败。

    参考：[#3611](https://www.sqlalchemy.org/trac/ticket/3611)

+   **[orm] [bug]**

    修复了`Session.bulk_update_mappings()`等相关方法在使用时不会增加版本 id 计数器的错误。这里的体验仍然有点粗糙，因为给定字典中仍需要原始版本 id，并且目前没有关于此的清晰错误报告。

    参考：[#3610](https://www.sqlalchemy.org/trac/ticket/3610)

+   **[orm] [bug]**

    对`Mapper.eager_defaults`标志进行了重大修复，该标志在多个 UPDATE 语句要发出的情况下不会被正确执行，无论是作为刷新的一部分还是作为批量更新操作的一部分。此外，在更新语句中不必要地发出 RETURNING。

    参考：[#3609](https://www.sqlalchemy.org/trac/ticket/3609)

+   **[orm] [bug]**

    修复了使用`Query.select_from()`方法会导致后续调用`Query.with_parent()`方法失败的错误。

    参考：[#3606](https://www.sqlalchemy.org/trac/ticket/3606)

### sql

+   **[sql] [bug]**

    修复了`Update.return_defaults()`中的错误，该错误会导致所有未包含在 SET 子句中（例如主键列）的插入默认列被渲染到 RETURNING 中，尽管这是一个 UPDATE。

    参考：[#3609](https://www.sqlalchemy.org/trac/ticket/3609)

### mysql

+   **[mysql] [bug]**

    调整了用于解析 MySQL 视图的正则表达式，不再假设反射视图源中存在“ALGORITHM”关键字，因为一些用户报告在某些 Amazon RDS 环境中不存在该关键字。

    参考：[#3613](https://www.sqlalchemy.org/trac/ticket/3613)

+   **[mysql] [bug]**

    为 MySQL 5.7 的 MySQL 方言添加了新的保留字，包括‘generated’、‘optimizer_costs’、‘stored’、‘virtual’。感谢 Hanno Schlichting 的 Pull 请求。

### 杂项

+   **[bug] [ext]**

    进一步修复了[#3605](https://www.sqlalchemy.org/trac/ticket/3605)，在`MutableDict`上的 pop 方法，其中未包含“default”参数。

    参考：[#3605](https://www.sqlalchemy.org/trac/ticket/3605)

+   **[bug] [ext]**

    修复了烘焙加载器系统中的 bug，该系统范围的 monkeypatch 用于设置烘焙懒加载器会干扰依赖于懒加载作为后备的其他加载器策略，例如连接和子查询急加载器，在映射器配置时导致`IndexError`异常。

    参考：[#3612](https://www.sqlalchemy.org/trac/ticket/3612)

## 1.0.10

发布日期：2015 年 12 月 11 日

### orm

+   **[orm] [bug]**

    修复了一个问题，即在一对多关系上的 post_update 在属性设置为 None 且之前未加载时会失败发出 UPDATE 的情况。

    参考：[#3599](https://www.sqlalchemy.org/trac/ticket/3599)

+   **[orm] [bug]**

    修复了一个 bug，实际上是在版本 0.8.0 和 0.8.1 之间发生的回归，由于[#2714](https://www.sqlalchemy.org/trac/ticket/2714)。当“with_polymorphic”也被使用时，需要通过子类绑定关系进行连接的连接急加载会无法从正确的实体进行连接。

    参考：[#3593](https://www.sqlalchemy.org/trac/ticket/3593)

+   **[orm] [bug]**

    修复了 joinedload bug，该 bug 会在以下情况下发生：a. 查询包含强制子查询的 limit/offset 条件 b. 关系使用“secondary” c. 关系的 primaryjoin 引用的列不是主键的一部分，或者是一个不同属性名称下的 joined-inheritance 子类表的 PK 列 d. 查询推迟了在 primaryjoin 中出现的列，通常通过不包含在 load_only()中；必要的列不会出现在子查询中，从而产生无效的 SQL。

    参考：[#3592](https://www.sqlalchemy.org/trac/ticket/3592)

+   **[orm] [bug]**

    在一些 MySQL SAVEPOINT 案例中观察到的一种罕见情况，当`Session.rollback()`在`Session.flush()`操作的范围内失败并引发异常时，会阻止在 flush 期间发出的原始数据库异常被观察到，但仅在 Py2K 上，因为 Py2K 不支持异常链接；在 Py3K 上，原始异常会被链接。作为一种解决方法，在这种特定情况下会发出警告，至少显示原始数据库错误的字符串消息，然后我们继续引发导致回滚的异常。

    参考：[#2696](https://www.sqlalchemy.org/trac/ticket/2696)

### orm 声明

+   **[orm] [declarative] [bug]**

    修复了一个 bug，在 Py2K 中，unicode 文字不能作为声明中使用`relationship()`上的`backref()`的类或其他参数的字符串名称。感谢 Nils Philippsen 的拉取请求。

### sql

+   **[sql] [feature]**

    增加了对 UPDATE 语句中参数顺序化 SET 子句的支持。通过将`update.preserve_parameter_order`标志传递给核心`Update`构造，或者将其添加到 ORM 级别的`Query.update.update_args`字典中，同时将参数本身作为 2 元组列表传递。感谢 Gorka Eguileor 的实现和测试。

    另请参见

    参数顺序化更新

+   **[sql] [bug]**

    修复了`Insert.from_select()`构造中的问题，当目标`Table`具有 Python 端默认值时，`Select`构造的`._raw_columns`集合会在编译`Insert`构造时被就地修改。`Select`构造在编译后会独立存在，错误的列会在编译`Insert`后出现，`Insert`语句本身在第二次编译尝试时会因重复的绑定参数而失败。

    参考：[#3603](https://www.sqlalchemy.org/trac/ticket/3603)

+   **[sql] [bug] [postgresql]**

    修复了一个 bug，当使用 CREATE TABLE 创建一个没有列但有约束（如 CHECK 约束）的表时，会在定义中出现错误的逗号；这种情况可能发生在具有自己没有列的 PostgreSQL INHERITS 表中。

    参考：[#3598](https://www.sqlalchemy.org/trac/ticket/3598)

### postgresql

+   **[postgresql] [bug]**

    修复了一个问题，当“FOR UPDATE OF” PostgreSQL 特定的 SELECT 修饰符引用的表具有模式限定符时，如果省略模式名称，PG 会失败。感谢 Diana Clarke 的拉取请求。

    参考：[#3573](https://www.sqlalchemy.org/trac/ticket/3573)

+   **[postgresql] [bug]**

    修复了一个 bug，当将某些类型的 SQL 表达式传递给`ExcludeConstraint`的“where”子句时，无法正确接受。感谢 aisch 的拉取请求。

+   **[postgresql] [bug]**

    修复了`INTERVAL`的`.python_type`属性，使其像`python_type`一样返回`datetime.timedelta`，而不是引发`NotImplementedError`。

    参考：[#3571](https://www.sqlalchemy.org/trac/ticket/3571)

### mysql

+   **[mysql] [bug]**

    修复了 MySQL 反射中的错误，其中`DATETIME`、`TIMESTAMP`和`TIME`类型的“小数部分”会被错误地放入未被 MySQL 使用的`timezone`属性中���而不是`fsp`属性。

    参考：[#3602](https://www.sqlalchemy.org/trac/ticket/3602)

### mssql

+   **[mssql] [bug]**

    将“20006: 写入服务器失败”错误添加到 pymssql 驱动程序的断开错误列表中，因为观察到这会使连接无法使用。

    参考：[#3585](https://www.sqlalchemy.org/trac/ticket/3585)

+   **[mssql] [bug]**

    现在，如果 SQL 服务器从 DATE 或 TIME 列返回无效的日期或时间格式，将引发描述性的 ValueError，而不是出现 NoneType 错误。感谢 Ed Avis 的贡献。

+   **[mssql] [bug]**

    修复了对于 MSSQL 类型 DATETIME2、TIME 和 DATETIMEOFFSET 的 DDL 生成中，精度为“零”时不会生成精度字段的问题。感谢 Jacobo de Vera 的贡献。

### 测试

+   **[tests] [change]**

    ORM 和 Core 教程一直以 doctest 格式存在，现在在 Python 2 和 Python 3 中都在正常的单元测试套件中执行。

### 杂项

+   **[bug] [ext]**

    为`MutableDict`类添加了对`dict.pop()`和`dict.popitem()`方法的支持。

    参考：[#3605](https://www.sqlalchemy.org/trac/ticket/3605)

+   **[bug] [py3k]**

    对内部`getargspec()`调用进行更新，一些与 py36 相关的 fixture 更新，以及对两个迭代器进行修改，使其“返回”而不是引发 StopIteration，以便在 Py3.5、Py3.6 上通过测试而不出现错误或警告，感谢 Jacob MacDonald、Luri de Silvio 和 Phil Jones 的贡献。

+   **[bug] [ext]**

    修复了烘焙查询中的问题，其中`.get()`方法，无论是直接使用还是在惰性加载中使用，都没有将映射器的“获取子句”视为缓存键的一部分，导致如果子句重新生成，则绑定参数不匹配。这个子句会被映射器动态缓存，但在高并发场景下，当首次访问时可能会生成多次。

    参考：[#3597](https://www.sqlalchemy.org/trac/ticket/3597)

## 1.0.9

发布日期：2015 年 10 月 20 日

### orm

+   **[orm] [feature]**

    添加了新方法`Query.one_or_none()`；与`Query.one()`相同，但如果未找到行，则返回 None。感谢 esiegerman 的贡献。

+   **[orm] [bug] [postgresql]**

    修复了 1.0 版本中的回归，即在 ORM 中使用“executemany”进行 UPDATE 语句的新功能（例如 UPDATE statements are now batched with executemany() in a flush）在 PostgreSQL 和其他 RETURNING 后端上会出现问题，当使用服务器端版本生成方案时，由于服务器端值是通过 RETURNING 检索的，而在使用 executemany 时不支持。

    参考：[#3556](https://www.sqlalchemy.org/trac/ticket/3556)

+   **[orm] [bug]**

    修复了在内部日志记录中对某些类型的内部列加载器选项进行字符串化时可能出现的罕见 TypeError。

    参考：[#3539](https://www.sqlalchemy.org/trac/ticket/3539)

+   **[orm] [bug]**

    修复了`Session.bulk_save_objects()`中的错误，其中具有某种“更新时获取”值的映射列，且在给定对象中不存在本地时，会导致操作中的 AttributeError。

    参考：[#3525](https://www.sqlalchemy.org/trac/ticket/3525)

+   **[orm] [bug]**

    修复了 1.0 版本中的回归，即“noload”加载策略在一对多关系中无法正常工作的问题。加载器使用 API 将“None”放入字典中，但实际上不再写入值；这是[#3061](https://www.sqlalchemy.org/trac/ticket/3061)的副作用。

    参考：[#3510](https://www.sqlalchemy.org/trac/ticket/3510)

### examples

+   **[examples] [bug]**

    修复了“history_meta”示例中的两个问题，其中历史跟踪可能遇到空历史，以及键入替代属性名称的列无法正确跟踪的问题。修复由 Alex Fraser 提供。

### sql

+   **[sql] [bug]**

    修复了 1.0 版本中发布的默认处理器对多值插入语句的回归，[#3288](https://www.sqlalchemy.org/trac/ticket/3288)，其中默认保存列的列类型不会传播到编译后的语句中，在使用默认值时，导致绑定级别类型处理程序不被调用。

    参考：[#3520](https://www.sqlalchemy.org/trac/ticket/3520)

### postgresql

+   **[postgresql] [bug]**

    对 1.0.6 中发布的反映存储选项和[#3455](https://www.sqlalchemy.org/trac/ticket/3455)的 PostgreSQL 新功能进行了调整，以禁用 PostgreSQL 版本< 8.2 的功能，其中未提供`reloptions`列；这允许 Amazon Redshift 再次正常工作，因为它基于 8.0.x 版本的 PostgreSQL。修复由 Pete Hollobon 提供。

### oracle

+   **[oracle] [bug] [py3k]**

    修复了 cx_Oracle 版本 5.2 的支持，该版本在 Python 3 下触发了 SQLAlchemy 的版本检测，并且意外地未使用正确的 Unicode 模式进行 Python 3。这会导致问题，例如绑定变量被误解为 NULL，以及行被静默地未返回。

    此更改也**回溯**到：0.7.0b1

    参考：[#3491](https://www.sqlalchemy.org/trac/ticket/3491)

+   **[oracle] [bug]**

    修复了 Oracle 方言中的一个错误，即对使用引号强制转换为全小写的表和其他符号的反射在反射查询中无法正确识别。现在，对于在“名称规范化”过程中被强制转换为全小写的传入符号名称，将应用`quoted_name`构造。

    参考：[#3548](https://www.sqlalchemy.org/trac/ticket/3548)

### 杂项

+   **[feature] [ext]**

    在`AssociationProxy`构造函数中添加了`AssociationProxy.info`参数，以适应在[#2971](https://www.sqlalchemy.org/trac/ticket/2971)中添加的`AssociationProxy.info`访问器。这是因为`AssociationProxy`是显式构造的，不像通过装饰器语法隐式构造的混合体。

    参考：[#3551](https://www.sqlalchemy.org/trac/ticket/3551)

+   **[bug] [sybase]**

    修复了关于 Sybase 反射的两个问题，允许没有主键的表被反射，同时确保涉及外键检测的 SQL 语句被预先获取，以避免嵌套查询时出现驱动程序问题。此处修复由 Eugene Zapolsky 提供；请注意，我们目前无法测试 Sybase 以本地验证这些更改。

    参考：[#3508](https://www.sqlalchemy.org/trac/ticket/3508)，[#3509](https://www.sqlalchemy.org/trac/ticket/3509)

## 1.0.8

发布日期：2015 年 7 月 22 日

### engine

+   **[engine] [bug]**

    修复了关键问题，即在池“checkout”事件处理程序可能针对未调用“connect”事件处理程序的陈旧连接进行调用的情况下，当池尝试在被使无效后重新连接并失败时，陈旧连接将保留并在随后的尝试中使用。这个问题在 1.0.2 之后的 1.0 系列中影响更大，因为它还向事件处理程序提供了一个空白的`.info`字典；在 1.0.2 之前，`.info`字典仍然是先前的字典。

    此更改也**回溯**至：0.7.0b1

    参考：[#3497](https://www.sqlalchemy.org/trac/ticket/3497)

### sqlite

+   **[sqlite] [bug]**

    修复了 SQLite 方言中的一个错误，即包含非字母字符（如点或空格）的唯一约束的反射不会反映其名称。

    此更改也**回溯**至：0.9.10

    参考：[#3495](https://www.sqlalchemy.org/trac/ticket/3495)

### 杂项

+   **[misc] [bug]**

    修复了一个问题，即 utils 中的特定基类未实现 `__slots__`，因此该类的所有子类也未实现，这使得使用 `__slots__` 没有意义。除了在 IronPython 上可能会出现问题外，其他地方都没有问题，因为 IronPython 显然不兼容 cPython 的 `__slots__` 行为。

    参考：[#3494](https://www.sqlalchemy.org/trac/ticket/3494)

## 1.0.7

发布日期：2015 年 7 月 20 日

### orm

+   **[orm] [bug]**

    修复了 1.0 版本中的一个回归问题，即覆盖 `__eq__()` 以返回非布尔类型对象的值对象，在工作单元更新操作期间会被测试为 `bool()`，而在 0.9 版本中，`__eq__()` 的返回值会与“is True”进行测试，以防止出现此问题。

    参考：[#3469](https://www.sqlalchemy.org/trac/ticket/3469)

+   **[orm] [bug]**

    修复了 1.0 版本中的一个回归问题，即“延迟”属性在“优化继承加载”中无法正确填充的情况，这是在使用联接表继承时发出的特殊 SELECT 语句，用于填充已过期或未加载的属性，而不加载基本表。这与 SQLA 1.0 不再猜测加载延迟列的事实有关，必须明确指示。

    参考：[#3468](https://www.sqlalchemy.org/trac/ticket/3468)

+   **[orm] [bug]**

    修复了 1.0 版本中的一个回归问题，即在 `aliased()` 对象上的同义词映射属性的“父实体”将解析为原始映射器，而不是其 `aliased()` 版本，从而导致依赖于此属性的 `Query` 出现问题（例如，在构造函数中只给出了代表性属性，以确定查询的正确 FROM 子句）。

    参考：[#3466](https://www.sqlalchemy.org/trac/ticket/3466)

### orm declarative

+   **[orm] [declarative] [bug]**

    修复了 `AbstractConcreteBase` 扩展中的错误，即在 ABC 基类上设置了一个具有不同属性名和列名的列，最终基类上不会正确映射该列。在 0.9 版本上失败是静默的，而在 1.0 版本上会引发 ArgumentError，因此在 1.0 版本之前可能不会注意到。

    参考：[#3480](https://www.sqlalchemy.org/trac/ticket/3480)

### engine

+   **[engine] [bug]**

    修复了 ORM `Query`对象中使用的`ResultProxy`上的新方法（作为[#3175](https://www.sqlalchemy.org/trac/ticket/3175)的性能增强的一部分）在驱动程序（通常是 MySQL）无法正确生成 cursor.description 的情况下不会引发“此结果不返回行”异常的回归；而是会引发针对 NoneType 的 AttributeError。 

    参考：[#3481](https://www.sqlalchemy.org/trac/ticket/3481)

+   **[engine] [bug]**

    修复了`ResultProxy.keys()`返回未调整的内部符号名称的回归，这些符号名称是我们在没有标签的 SQL 函数和类似情况下生成的“foo_1”类型的标签的副作用。这是作为#918 的性能增强的一部分实施的。

    参考：[#3483](https://www.sqlalchemy.org/trac/ticket/3483)

### sql

+   **[sql] [feature]**

    添加了一个`ColumnElement.cast()`方法，其执行与独立的`cast()`函数相同的目的。感谢 Sebastian Bank 的拉取请求。

    参考：[#3459](https://www.sqlalchemy.org/trac/ticket/3459)

+   **[sql] [bug]**

    修复了在与`and_()`或`or_()`结合使用文字`True`或`False`常量时，会因为 AttributeError 而失败的错误。

    参考：[#3490](https://www.sqlalchemy.org/trac/ticket/3490)

+   **[sql] [bug]**

    修复了自定义`FunctionElement`或其他列元素的子类错误地将‘None’或任何其他无效对象声明为`.type`属性时，会报告此异常而不是递归溢出的潜在问题。

    参考：[#3485](https://www.sqlalchemy.org/trac/ticket/3485)

+   **[sql] [bug]**

    修复了模数 SQL 运算符由于缺少`__rmod__`方法而无法反向工作的错误。感谢 dan-gittik 的拉取请求。

### 模式

+   **[schema] [feature]**

    增加了对 CREATE SEQUENCE 的 MINVALUE、MAXVALUE、NO MINVALUE、NO MAXVALUE 和 CYCLE 参数的支持，这些参数受 PostgreSQL 和 Oracle 支持。感谢 jakeogh 的拉取请求。

## 1.0.6

发布日期：2015 年 6 月 25 日

### orm

+   **[orm] [bug]**

    修复了 1.0 系列中的一个重大回归问题，即 version_id_counter 功能会导致对象的版本计数器在对象的行没有净变化时被增加，而是通过关系（例如通常是一对多）与其关联或取消关联的对象，导致更新语句更新对象的版本计数器而不更新其他内容。在相对较新的“服务器端”和/或“程序化/条件化”版本计数器功能被使用的用例中（例如将 version_id_generator 设置为 False），这个错误可能导致发出没有有效 SET 子句的 UPDATE。

    参考：[#3465](https://www.sqlalchemy.org/trac/ticket/3465)

+   **[orm] [bug]**

    修复了 1.0 版本中的一个回归问题，即增强的单继承连接的行为[#3222](https://www.sqlalchemy.org/trac/ticket/3222)不适当地发生在沿着显式连接条件进行 JOIN 时，其中单继承子类不使用任何鉴别器，导致额外的“AND NULL”子句。

    参考：[#3462](https://www.sqlalchemy.org/trac/ticket/3462)

+   **[orm] [bug]**

    修复了新的`Session.bulk_update_mappings()`功能中的错误，其中用于定位行的 WHERE 子句中使用的主键列也会包含在 SET 子句中，将它们的值不必要地设置为它们自己。感谢 Patrick Hayes 的拉取请求。

    参考：[#3451](https://www.sqlalchemy.org/trac/ticket/3451)

+   **[orm] [bug]**

    修复了一个意外使用回归问题，即自定义`Comparator`对象使用`__clause_element__()`方法并返回一个 ORM 映射的`InstrumentedAttribute`对象而不是显式的`ColumnElement`时，当作为表达式传递给`Session.query()`时无法正确处理。0.9 版本的逻辑恰好成功，因此现在支持这种用例。

    参考：[#3448](https://www.sqlalchemy.org/trac/ticket/3448)

### sql

+   **[sql] [bug]**

    修复了一个 bug，即应用于`Label`对象的子句适应在所有情况下都会失败，这样任何使用`Label.self_group()`的 SQL 操作都会使用原始未适应的表达式。其中一个影响是 ORM `aliased()`构造将无法完全适应由`column_property`映射的属性，因此在某些类型的 SQL 比较中，未别名化的表可能会泄漏出来。

    参考：[#3445](https://www.sqlalchemy.org/trac/ticket/3445)

### postgresql

+   **[postgresql] [feature]**

    增加了对在 CREATE INDEX 下使用存储参数的支持，使用了一个新的关键字参数`postgresql_with`。还增加了反射支持，以支持`postgresql_with`标志和`postgresql_using`标志，这些标志现在将设置在被反射的`Index`对象上，并且在`Inspector.get_indexes()`的结果中还存在一个新的“dialect_options”字典中。感谢 Pete Hollobon 的 Pull 请求。

    另请参阅

    索引存储参数

    参考：[#3455](https://www.sqlalchemy.org/trac/ticket/3455)

+   **[postgresql] [feature]**

    增加了新的执行选项`max_row_buffer`，当使用`stream_results`选项时，由 psycopg2 方言解释，它设置了可以分配的行缓冲区的大小限制。这个值也是基于发送给`Query.yield_per()`的整数值提供的。感谢 mcclurem 的 Pull 请求。

+   **[postgresql] [bug] [pypy]**

    重新修复了在 1.0.5 中首次发布的问题，以再次修复 psycopg2cffi 对 JSONB 支持，因为他们突然在 2.7.1 版本中切换到了对 JSONB 类型的无条件解码。版本检测现在指定 2.7.1 是我们应该期望 DBAPI 为我们进行 json 编码的地方。

    参考：[#3439](https://www.sqlalchemy.org/trac/ticket/3439)

+   **[postgresql] [bug]**

    修复了`ExcludeConstraint`构造，以支持其他对象（如`Index`）现在支持的常见功能，即列表达式可以指定为任意的 SQL 表达式，如`cast`或`text`。

    参考：[#3454](https://www.sqlalchemy.org/trac/ticket/3454)

### mssql

+   **[mssql] [bug]**

    修复了在使用`VARBINARY`类型与插入 NULL + pyodbc 时出现的问题；pyodbc 需要传递一个特殊对象以保留 NULL。由于由于[#3039](https://www.sqlalchemy.org/trac/ticket/3039)，`VARBINARY`类型现在通常是`LargeBinary`的默认值，因此这个问题在 1.0 中部分是一个退化。pymssql 驱动程序似乎不受影响。

    参考：[#3464](https://www.sqlalchemy.org/trac/ticket/3464)

### 杂项

+   **[bug] [documentation]**

    修复了内部的“记忆化”方法类型，不再使用 Python 描述符；修复了这些方法的可检查性，包括对 Sphinx 文档的支持。

    参考：[#2077](https://www.sqlalchemy.org/trac/ticket/2077)

## 1.0.5

发布日期：2015 年 6 月 7 日

### orm

+   **[orm] [feature]**

    添加了新的事件`InstanceEvents.refresh_flush()`，在刷新过程中通过 RETURNING 或 Python-side 默认值获取的 INSERT 或 UPDATE 级别默认值时调用。这是为了提供一个钩子，因为由于[#3167](https://www.sqlalchemy.org/trac/ticket/3167)的结果，属性和验证事件不再在刷新过程中调用。

    参考：[#3427](https://www.sqlalchemy.org/trac/ticket/3427)

+   **[orm] [bug]**

    当`Query`返回行时使用的“轻量级命名元组”未正确实现`__slots__`，以至于仍然有一个`__dict__`。这个问题已经解决，但在极不可能的情况下，如果有人给返回的元组赋值，那将不再起作用。

    参考：[#3420](https://www.sqlalchemy.org/trac/ticket/3420)

### engine

+   **[engine] [feature]**

    添加了新的引擎事件`ConnectionEvents.engine_disposed()`。在调用`Engine.dispose()`方法之后调用。

+   **[engine] [feature]**

    调整引擎插件钩子，使得当使用方言插件时，`URL.get_dialect()`方法将继续返回最终的`Dialect`对象，而不需要调用者知道`Dialect.get_dialect_cls()`方法。

    参考：[#3379](https://www.sqlalchemy.org/trac/ticket/3379)

+   **[engine] [bug]**

    修复了已知布尔值在`engine_from_config()`中使用时未被正确解析的 bug；这些包括`pool_threadlocal`和 psycopg2 参数`use_native_unicode`。

    参考：[#3435](https://www.sqlalchemy.org/trac/ticket/3435)

+   **[engine] [bug]**

    增加了对行为不端的 DBAPI 的支持，该 DBAPI 将 pep-249 异常名称链接到完全不同名称的异常类，从而阻止 SQLAlchemy 自身的异常包装适当地包装错误。正在使用的 SQLAlchemy 方言需要实现一个新的访问器`DefaultDialect.dbapi_exception_translation_map`来支持此功能；现在已为 py-postgresql 方言实现了这一功能。

    参考：[#3421](https://www.sqlalchemy.org/trac/ticket/3421)

+   **[engine] [bug]**

    修复了一个 bug，当使用池检出事件处理程序并在处理程序本身中进行连接尝试并失败时，拥有连接记录直到连接错误的堆栈跟踪被释放之前不会被释放。对于仅使用单个连接的测试池的情况，这意味着池将完全被检出，直到该堆栈跟踪被释放。这主要影响非常特定的调试场景，不太可能在任何生产应用程序中引起注意。修复方法是在重新引发捕获的异常之前显式检入记录。

    参考：[#3419](https://www.sqlalchemy.org/trac/ticket/3419)

### sql

+   **[sql] [feature]**

    官方支持了`Insert.from_select()`内部 SELECT 中使用的 CTE。此行为直到 0.9.9 之前都是偶然发生的，当时由于与 [#3248](https://www.sqlalchemy.org/trac/ticket/3248) 相关的其他更改，它不再起作用。请注意，这是在 INSERT 之后、SELECT 之前渲染 WITH 子句的方式；在 INSERT、UPDATE、DELETE 的顶层渲染 CTE 的完整功能是针对以后的版本发布的新功能。

    此更改也已**回溯到**：0.9.10

    参考：[#3418](https://www.sqlalchemy.org/trac/ticket/3418)

### postgresql

+   **[postgresql] [bug] [pypy]**

    修复了与 pypy psycopg2cffi 方言相关的一些打字和测试问题，特别是当前的 2.7.0 版本不支持 JSONB 类型。对于 psycopg2 特性的版本检测已调整为适用于特定的 psycopg2cffi 子版本。此外，已启用了对 psycopg2cffi 下的完整系列 psycopg2 特性的测试覆盖。

    参考：[#3439](https://www.sqlalchemy.org/trac/ticket/3439)

### mssql

+   **[mssql] [bug]**

    向 MSSQL 方言添加了一个新的方言标志 `legacy_schema_aliasing`，当设置为 False 时将禁用非常古老且已过时的行为，即编译器尝试将所有模式限定的表名转换为别名，以解决 SQL Server 无法在所有情况下解析多部分标识符名称的旧问题，以使更复杂的语句能够正确工作，包括使用提示的语句以及嵌入相关 SELECT 语句的 CRUD 语句。与其继续修复该特性以使其能够与更复杂的语句一起工作，不如直接禁用它，因为对于任何现代 SQL Server 版本，它应该不再需要。该标志在 1.0.x 系列中默认为 True，保持当前版本系列的行为不变。在 1.1 系列中，它将默认为 False。对于 1.0 系列，当未显式设置为任一值时，在语句中首次使用模式限定的表时会发出警告，建议将该标志设置为 False 以适用于所有现代 SQL Server 版本。

    另请参阅

    传统架构模式

    参考：[#3424](https://www.sqlalchemy.org/trac/ticket/3424)，[#3430](https://www.sqlalchemy.org/trac/ticket/3430)

### 杂项

+   **[功能] [扩展]**

    添加了对`*args`传递给烘焙查询初始可调用的支持，方式与`BakedQuery.add_criteria()`和`BakedQuery.with_criteria()`方法支持`*args`的方式相同。初始 PR 由 Naoki INADA 提供。

+   **[功能] [扩展]**

    添加了一个新的半公共方法`MutableBase` `MutableBase._get_listen_keys()`。在拦截`InstanceEvents.refresh()`或`InstanceEvents.refresh_flush()`事件时，需要重写此方法，以便在`MutableBase`子类需要事件传播到与可变类型关联的键之外的属性键时。目前的示例是使用`MutableComposite`的复合体。

    参考：[#3427](https://www.sqlalchemy.org/trac/ticket/3427)

+   **[错误] [扩展]**

    修复了`sqlalchemy.ext.mutable`扩展中的回归问题，这是由于对[#3167](https://www.sqlalchemy.org/trac/ticket/3167)的错误修复导致的，其中属性和验证事件不再在刷新过程中调用。在列级别的 Python 端默认值负责生成 INSERT 或 UPDATE 的新值，或者在“eager defaults”模式下从 RETURNING 子句中获取值的情况下，可变扩展依赖于此行为。当填充新值时，新值不会受到任何事件的影响，可变扩展无法建立适当的强制转换或历史监听。添加了一个新事件`InstanceEvents.refresh_flush()`，可变扩展现在使用这个事件来处理这种情况。

    参考：[#3427](https://www.sqlalchemy.org/trac/ticket/3427)

## 1.0.4

发布日期：2015 年 5 月 7 日

### orm

+   **[orm] [错误]**

    修复了一个意外使用回归问题，即在关系的主连接涉及与不可哈希类型（如 HSTORE）的比较的奇怪情况下，由于语句参数上的哈希导向检查，在 1.0 中修改为使用哈希，并在[#3368](https://www.sqlalchemy.org/trac/ticket/3368)中修改为在比“挂起加载”更常见的情况下发生。现在会事先检查值是否具有`__hash__`属性。

    参考：[#3416](https://www.sqlalchemy.org/trac/ticket/3416)

+   **[ORM] [错误]**

    放宽了一个断言，该断言是作为[#3347](https://www.sqlalchemy.org/trac/ticket/3347)的一部分添加的，以防止在使用`innerjoin=True`在连接的急切加载中拼接内部连接时出现未知条件；如果一些连接使用“secondary”表，则需要进一步展开连接以通过。

    参考：[#3347](https://www.sqlalchemy.org/trac/ticket/3347), [#3412](https://www.sqlalchemy.org/trac/ticket/3412)

+   **[ORM] [错误]**

    修复/添加了更多表达式的测试，这些表达式被报告为在新的‘entity’键值添加到`Query.column_descriptions`中失败，重新设计了发现“from”子句的逻辑，以适应来自别名类的列，以及在这些情况下报告“aliased”标志的正确值。

    参考：[#3320](https://www.sqlalchemy.org/trac/ticket/3320), [#3409](https://www.sqlalchemy.org/trac/ticket/3409)

### 模式

+   **[模式] [错误]**

    修复了在增强的约束附加逻辑中引入的错误，该错误在罕见情况下，约束同时引用`Column`对象和字符串列名称时，将跳过自动附加到列附加逻辑；对于约束在这种情况下自动附加，必须提前将所有列组装到目标表上。在迁移文档中添加了关于原始功能以及此更改的新部分。

    另请参阅

    引用未附加列的约束可以在其引用的列附加时自动附加到表格

    参考：[#3411](https://www.sqlalchemy.org/trac/ticket/3411)

### 测试

+   **[测试] [错误] [pypy]**

    修复了一个导入问题，导致“pypy setup.py test”无法正常工作。

    此更改也**回溯**到：0.9.10

    参考：[#3406](https://www.sqlalchemy.org/trac/ticket/3406)

### 杂项

+   **[错误] [扩展]**

    修复了使用扩展属性仪器系统时的错误，当使用`class_mapper()`调用无效输入（也恰好不是弱引用）时，不会引发正确的异常。

    此更改也**回溯**到：0.9.10

    参考：[#3408](https://www.sqlalchemy.org/trac/ticket/3408)

## 1.0.3

发布日期：2015 年 4 月 30 日

### orm

+   **[orm] [bug] [pypy]**

    由于[#3349](https://www.sqlalchemy.org/trac/ticket/3349)导致的发布之前的回归问题，即在`Query.update()`或`Query.delete()`上检查查询状态时，使用`is`将空元组与自身进行比较，在 PyPy 上失败，导致在这种情况下产生`True`；这将在 0.9 中错误地发出警告，并在 1.0 中引发异常。

    参考：[#3405](https://www.sqlalchemy.org/trac/ticket/3405)

+   **[orm] [bug]**

    修复了在发布之前从 0.9.10 版本开始的回归问题，即将`entity`添加到`Query.column_descriptions`访问器时，如果目标实体是从核心可选择对象（如`Table`或`CTE`对象）生成的，则会失败。

    参考：[#3320](https://www.sqlalchemy.org/trac/ticket/3320), [#3403](https://www.sqlalchemy.org/trac/ticket/3403)

+   **[orm] [bug]**

    修复了在刷新过程中的回归问题，当属性设置为 UPDATE 的 SQL 表达式时，与属性的先前值进行比较时，如果 SQL 表达式产生的 SQL 比较不是`==`或`!=`，则会引发异常“此子句的布尔值未定义”。修复确保工作单元不会以这种方式解释 SQL 表达式。

    参考：[#3402](https://www.sqlalchemy.org/trac/ticket/3402)

+   **[orm] [bug]**

    修复了由于[#2992](https://www.sqlalchemy.org/trac/ticket/2992)导致的意外使用回归问题，其中在与连接的急加载一起放置到`Query.order_by()`子句中的文本元素会被添加到内部查询的列子句中，以一种被假定为表绑定列名的方式，这种情况下，连接的急加载需要将查询包装在子查询中以适应限制/偏移量。

    最初，这里的行为是有意的，例如`query(User).order_by('name').limit(1)`这样的查询将按`user.name`排序，即使查询被连接式急加载修改为在子查询中，因为`'name'`将被解释为一个符号，应该在 FROM 子句中定位，此处为`User.name`，然后将其复制到列子句中以确保它在 ORDER BY 中存在。然而，该功能未能预料到`order_by("name")`指的是本地列子句中已经存在的特定标签名称，而不是绑定到 FROM 子句中的名称。

    此外，该功能在像`order_by("name desc")`这样的已弃用情况下也会失败，尽管它会发出警告，指出应该在这里使用`text()`（请注意，该问题不会影响显式使用`text()`的情况），但仍会产生与以前不同的查询，其中“name desc”表达式被不当地复制到列子句中。解决方案是，该功能的“连接式急加载”方面将在增强内部列子句时跳过这些所谓的“标签引用”表达式，就好像它们已经是`text()`构造一样。

    参考：[#3392](https://www.sqlalchemy.org/trac/ticket/3392)

+   **[orm] [bug]**

    修复了关于`MapperEvents.instrument_class()`事件的回归，其中其调用被移动到类管理器对类进行仪器化之后，这与事件文档明确说明的相反。切换的理由是由于 Declarative 在将类映射为新的`@declared_attr`功能描述的目的之前设置了完整的“仪器管理器”，但也为了与经典使用`Mapper`的一致性而进行了更改。然而，SQLSoup 依赖于在任何经典映射下的任何仪器化之前发生的仪器化事件。在经典和声明性映射的情况下，行为被恢复，后者通过简单的记忆化实现，而不使用类管理器。

    参考：[#3388](https://www.sqlalchemy.org/trac/ticket/3388)

+   **[orm] [bug]**

    在新的`QueryEvents.before_compile()`事件中修复了问题，在事件中对要加载的实体集合进行更改后，这些更改会反映在 SQL 中，但在加载过程中不会反映出来。

    参考：[#3387](https://www.sqlalchemy.org/trac/ticket/3387)

### 引擎

+   **[引擎] [功能]**

    新功能已添加以支持具有高级功能的引擎/池插件。在检出连接包装器的连接池级别以及`_ConnectionRecord`中添加了一个新的“软失效”功能。这类似于现代池失效，因为连接不会被主动关闭，但只有在下次检出时才会被回收；这本质上是该功能的每个连接版本。一个新的事件`PoolEvents.soft_invalidate()`被添加以补充它。

    还添加了新的标志`ExceptionContext.invalidate_pool_on_disconnect`。允许`ConnectionEvents.handle_error()`中的错误处理程序维护“断开连接”条件，但在事件中以特定方式调用单个连接的失效处理。

    参考：[#3379](https://www.sqlalchemy.org/trac/ticket/3379)

+   **[引擎] [功能]**

    添加了新的事件`do_connect`，它允许拦截/替换调用`Dialect.connect()`钩子以创建 DBAPI 连接的时机。还添加了方言插件钩子`Dialect.get_dialect_cls()`和`Dialect.engine_created()`，它们允许外部插件使用入口点向现有方言添加事件。

    参考：[#3355](https://www.sqlalchemy.org/trac/ticket/3355)

### sql

+   **[sql] [功能]**

    添加了一个占位符方法`TypeEngine.compare_against_backend()`，它现在在 Alembic 迁移中被消耗为 0.7.6\. 用户定义的类型可以实现此方法以协助比较来自数据库的类型与反射的类型。

+   **[sql] [错误]**

    修复了 SQL 中长标签的截断可能会产生与未截断的另一个标签重叠的 bug；这是因为截断的长度阈值大于截断后剩余标签的部分。现在这两个值已经调整为相同；`label_length - 6`。这里的效果是，较短的列标签在以前不会被截断的情况下现在将被“截断”。

    参考：[#3396](https://www.sqlalchemy.org/trac/ticket/3396)

+   **[sql] [bug]**

    由于 [#3282](https://www.sqlalchemy.org/trac/ticket/3282) 导致的回归已修复，此处关于 `tables` 集合作为关键字参数传递给 `DDLEvents.before_create()`、`DDLEvents.after_create()`、`DDLEvents.before_drop()` 和 `DDLEvents.after_drop()` 事件的 `tables` 集合不再是一个表的列表，而是一个包含第二个条目的元组列表，其中包含要添加或删除的外键。由于 `tables` 集合，虽然被文档化为不一定稳定，但已经被依赖，因此这个变化被认为是一个回归。此外，在某些情况下，“drop” 对于这个集合将是一个迭代器，如果过早迭代将导致操作失败。现在，这个集合在所有情况下都是一个表对象的列表，并且现在已经为该集合的格式添加了测试覆盖。

    参考：[#3391](https://www.sqlalchemy.org/trac/ticket/3391)

### 其他

+   **[bug] [ext]**

    修复了关联代理中的一个 bug，在关系-标量非对象属性比较上执行 `any()`/`has()` 操作会失败，例如 `filter(Parent.some_collection_to_attribute.any(Child.attr == 'foo'))`。

    参考：[#3397](https://www.sqlalchemy.org/trac/ticket/3397)

## 1.0.2

发布日期：2015 年 4 月 24 日

### orm 声明性

+   **[orm] [declarative] [bug]**

    修复了声明性 `__declare_first__` 和 `__declare_last__` 访问器的意外使用回归，这些访问器将不再在声明基类的超类上调用。

    参考：[#3383](https://www.sqlalchemy.org/trac/ticket/3383)

### sql

+   **[sql] [bug]**

    修复了一个在 1.0.0b4 中错误修复的回归问题（因此成为两个回归问题）；报告称 SELECT 语句将对标签名称进行 GROUP BY 并失败，被误解为某些后端（如 SQL Server）根本不应该在简单标签名称上发出 ORDER BY 或 GROUP BY；事实上，我们忘记了 0.9 版本已经为所有后端在简单标签名称上发出 ORDER BY，如 Label constructs can now render as their name alone in an ORDER BY 中所述，即使 1.0 版本包括对此逻辑的重写作为[#2992](https://www.sqlalchemy.org/trac/ticket/2992)的一部分。至于对简单标签进行 GROUP BY，即使 PostgreSQL 也有情况会引发错误，尽管应该明显可以看出要分组的标签，因此清楚地表明 GROUP BY 不应该自动以这种方式呈现。

    在 1.0.2 版本中，当传递一个`Label`构造到 SQL Server、Firebird 等数据库时，简单标签名称将再次发出 ORDER BY。此外，当传递一个`Label`构造时，任何后端都不会只对简单标签名称发出 GROUP BY。

    参考：[#3338](https://www.sqlalchemy.org/trac/ticket/3338), [#3385](https://www.sqlalchemy.org/trac/ticket/3385)

## 1.0.1

发布日期：2015 年 4 月 23 日

### orm

+   **[orm] [bug]**

    修复了一个问题，即形式为`query(B).filter(B.a != A(id=7))`的查询将渲染出`NEVER_SET`符号，当给定一个瞬态对象时。对于持久对象，它将始终使用持久化的数据库值而不是当前设置的值。假设自动刷新已打开，对于持久值，这通常对于持久值不会明显，因为任何待处理的更改都将首先被刷新。然而，这与用于非否定比较的逻辑不一致，`query(B).filter(B.a == A(id=7))`，它使用当前值并且还允许与瞬态对象进行比较。比较现在使用当前值而不是数据库持久化的值。

    与本次发布中由[#3061](https://www.sqlalchemy.org/trac/ticket/3061)引起的其他`NEVER_SET`问题不同，这个特定问题至少从 0.8 版本开始存在，可能更早，但是在修复相关的`NEVER_SET`问题时才被发现。

    参见

    “包含或等于的否定”关系比较将使用属性的当前值，而不是数据库值

    参考：[#3374](https://www.sqlalchemy.org/trac/ticket/3374)

+   **[orm] [bug]**

    修复了由[#3061](https://www.sqlalchemy.org/trac/ticket/3061)引起的意外使用回归问题，其中 NEVER_SET 符号可能会泄漏到关系导向查询中，包括`filter()`和`with_parent()`查询。在所有情况下都返回`None`符号，但��许多这些查询从未得到正确支持，并且在不使用 IS 运算符的情况下产生与 NULL 的比较。因此，对于当前不提供`IS NULL`的关系查询子集，还添加了警告。

    另请参阅

    比较具有 None 值的对象与关系时发出的警告

    参考：[#3371](https://www.sqlalchemy.org/trac/ticket/3371)

+   **[orm] [bug]**

    修复了由[#3061](https://www.sqlalchemy.org/trac/ticket/3061)引起的回归问题，其中 NEVER_SET 符号可能会泄漏到延迟加载查询中，在挂起对象刷新后。这通常会发生在不使用简单“get”策略的多对一关系中。好消息是，这个修复提高了效率，因为我们现在可以在检测到参数中存在 NEVER_SET 符号时完全跳过 SELECT 语句；在[#3061](https://www.sqlalchemy.org/trac/ticket/3061)之前，我们无法判断这里的 None 是否被设置。

    参考：[#3368](https://www.sqlalchemy.org/trac/ticket/3368)

### engine

+   **[engine] [bug]**

    将字符串值`"none"`添加到`Pool.reset_on_return`参数中，作为`None`的同义词，以便所有设置都可以使用字符串值，允许像`engine_from_config()`这样的实用程序可以无问题地使用。

    这个更改也被**回溯**到：0.9.10

    参考：[#3375](https://www.sqlalchemy.org/trac/ticket/3375)

### sql

+   **[sql] [bug]**

    修复了直接的 SELECT EXISTS 查询无法将正确的布尔结果类型分配给结果映射的问题，而是会从查询中泄漏列类型到结果映射中。这个问题在 0.9 版本及之前版本中也存在，但在那些版本中影响较小。在 1.0 版本中，由于[#918](https://www.sqlalchemy.org/trac/ticket/918)，这成为一个回归问题，因为我们现在依赖于结果映射非常准确，否则我们可能会将结果类型处理器分配给错误的列。在所有版本中，这个问题还会导致简单的 EXISTS 不适用布尔类型处理程序，导致后端没有原生布尔值而是简单的 1/0 值而不是 True/False。修复包括 EXISTS 列参数将像其他列表达式一样匿名标记；类似的修复也针对纯布尔表达式如`not_(True())`实现。

    参考：[#3372](https://www.sqlalchemy.org/trac/ticket/3372)

### sqlite

+   **[sqlite] [bug]**

    由于 [#3282](https://www.sqlalchemy.org/trac/ticket/3282) 导致的一个回归，由于我们在创建/删除模式时尝试假设 ALTER 可用，因此在 SQLite 的情况下，我们简单地说根本不用担心外键，因为在创建和删除表时 ALTER 不可用。这意味着在 SQLite 的情况下基本上跳过了表的排序，对于绝大多数 SQLite 使用情况，这并不是问题。

    然而，对于在 SQLite 上执行包含数据的表的 DROP 操作并且启用了引用完整性的用户，他们会遇到错误，因为在具有数据的表的 DROP 操作中，依赖排序确实很重要（SQLite 仍然可以让您创建对不存在表的外键并删除引用存在表的表，只要没有引用数据）。

    为了在仍允许 SQLite DROP 操作保持排序的同时保持 [#3282](https://www.sqlalchemy.org/trac/ticket/3282) 的新功能，我们现在考虑完整的外键进行排序，如果遇到无法解决的循环，*那么*我们放弃尝试对表进行排序；我们会发出警告并使用未排序的列表。如果一个环境需要有序的 DROP 操作 *并且* 存在外键循环，那么警告指出他们需要将 `use_alter` 标志恢复到他们的 `ForeignKey` 和 `ForeignKeyConstraint` 对象中，以便仅跳过这些对象的依赖排序。

    另请参阅

    ForeignKeyConstraint 上的 use_alter 标志（通常）不再需要 - 包含有关 SQLite 的更新说明。

    引用：[#3378](https://www.sqlalchemy.org/trac/ticket/3378)

### misc

+   **[bug] [firebird]**

    由于 [#3034](https://www.sqlalchemy.org/trac/ticket/3034) 导致的一个回归，Firebird 方言未能正确解释 limit/offset 子句。感谢 effem-git 提交的拉取请求。

    引用：[#3380](https://www.sqlalchemy.org/trac/ticket/3380)

+   **[bug] [firebird]**

    修复了在使用 Firebird 时“literal_binds”模式与 limit/offset 结合时的支持，因此当选择此模式时，值再次以内联方式呈现。与 [#3034](https://www.sqlalchemy.org/trac/ticket/3034) 相关。

    引用：[#3381](https://www.sqlalchemy.org/trac/ticket/3381)

## 1.0.0

发布日期：2015 年 4 月 16 日

### orm

+   **[orm] [feature]**

    添加了新参数`Query.update.update_args`，允许传递诸如`mysql_limit`之类的关键字参数到底层的`Update`构造。感谢 Amir Sadoughi 的拉取请求。

+   **[orm] [bug]**

    在处理多次对同一目标进行`Query.join()`时发现了一个不一致性；它只在关系连接的情况下隐式去重，由于[#3233](https://www.sqlalchemy.org/trac/ticket/3233)，在 1.0 中对同一表进行两次连接的行为与 0.9 不同，不再错误地别名。为了帮助记录这一变化，迁移说明中关于[#3233](https://www.sqlalchemy.org/trac/ticket/3233)的措辞已经泛化，并在多次调用`Query.join()`时添加了警告，针对相同目标关系连接。

    参考：[#3367](https://www.sqlalchemy.org/trac/ticket/3367)

+   **[orm] [bug]**

    在确定半自引用关系的远程端时，对关系的启发式进行了小的改进（例如，两个连接的继承子类相互引用），非简单的连接条件将考虑到父实体，并且可以减少使用`remote()`注释的需要；这可以恢复一些在 0.9.4 之前可能在没有注释的情况下工作的情况，通过[#2948](https://www.sqlalchemy.org/trac/ticket/2948)。

    参考：[#3364](https://www.sqlalchemy.org/trac/ticket/3364)

### sql

+   **[sql] [feature]**

    用于对`Table`对象进行排序的拓扑排序，并通过`MetaData.sorted_tables`集合提供，现在将产生一个**确定性**的排序；也就是说，给定一组具有特定名称和依赖关系的表，每次都会产生相同的排序。这有助于比较 DDL 脚本和其他用例。表按名称排序发送到拓扑排序，拓扑排序本身将以有序的方式处理传入的数据。感谢 Sebastian Bank 的拉取请求。

    参见

    MetaData.sorted_tables accessor is “deterministic”

    参考：[#3084](https://www.sqlalchemy.org/trac/ticket/3084)

+   **[sql] [bug]**

    修复了一个问题，即使用命名约定的 `MetaData` 对象不会正确地与 pickle 一起工作。该属性被跳过，导致如果使用了 unpickled 的 `MetaData` 对象来基于其他表，就会出现不一致和失败。

    此更改也已**回溯**到：0.9.10

    参考：[#3362](https://www.sqlalchemy.org/trac/ticket/3362)

### postgresql

+   **[postgresql] [bug]**

    修复了一个长期存在的 bug，即在使用 psycopg2 方言与非 ASCII 值和 `native_enum=False` 结合使用时，`Enum` 类型无法正确解码返回结果。这源自于 PG `ENUM` 类型过去是一个独立的类型，没有“非本机”选项。

    此更改也已**回溯**到：0.9.10

    参考：[#3354](https://www.sqlalchemy.org/trac/ticket/3354)

### mssql

+   **[mssql] [bug]**

    修复了一个回归，即“最后插入的 id”机制在 MSSQL 上的 INSERT 中失败，其中主键值在执行之前出现在插入参数中，以及在来自 SELECT 的 INSERT 中，会将目标列声明为列对象，而不是字符串键的情况下，将无法存储正确的值。

    参考：[#3360](https://www.sqlalchemy.org/trac/ticket/3360)

+   **[mssql] [bug]**

    现在使用 pymssql 中现有的 `Binary` 构造函数，而不是进行补丁。感谢 Ramiro Morales 提供的拉取请求。

### tests

+   **[tests] [bug]**

    修复了测试运行时使用的路径；对于 sqla_nose.py 和 py.test，再次在 sys.path 的开头插入“./lib”前缀，但仅当 sys.flags.no_user_site 没有设置时；这使其的行为就像 Python 默认将“.”放在当前路径中一样。对于 tox，我们现在设置了 PYTHONNOUSERSITE 标志。

    参考：[#3356](https://www.sqlalchemy.org/trac/ticket/3356)

## 1.0.0b5

发布日期：2015 年 4 月 3 日

### orm

+   **[orm] [bug]**

    修复了一个错误，在多个嵌套 `Session.begin_nested()` 操作内的状态跟踪会失败，如果在内部保存点内更新了对象，则该对象的“dirty”标志不会传播，因此如果回滚外部保存点，则对象将不会成为已过期状态的一部分，因此将恢复为其数据库状态。

    此更改也已**回溯**到：0.9.10

    参考：[#3352](https://www.sqlalchemy.org/trac/ticket/3352)

+   **[orm] [bug]**

    在使用 `Query.update()` 或 `Query.delete()` 方法时，`Query` 不支持连接、子选择或特殊的 FROM 子句；如果已经调用了类似 `Query.join()` 或 `Query.select_from()` 的方法，则不会默默地忽略这些字段，而是会引发错误。在 0.9.10 中，这只会发出警告。

    参考：[#3349](https://www.sqlalchemy.org/trac/ticket/3349)

+   **[orm] [bug]**

    在会话的提交阶段，添加了对使用弱字典的 list() 调用，如果没有它，如果垃圾收集与进程交互，可能会导致“迭代过程中字典大小发生变化”的错误。此更改由 #3139 引入。

+   **[orm] [bug]**

    修复了与“嵌套”内连接贪婪加载相关的错误，该错误在 0.9 中也存在，但由于 [#3008](https://www.sqlalchemy.org/trac/ticket/3008) 的原因，在 1.0 中更像是退化，默认情况下“嵌套”被启用，这样一个跨越共同祖先的兄弟路径的连接贪婪加载使用 innerjoin=True 将正确地将每个“innerjoin”兄弟插入连接的适当部分，当一系列内部/外部连接混合在一起时。

    参考：[#3347](https://www.sqlalchemy.org/trac/ticket/3347)

### sql

+   **[sql] [bug]**

    对于非 Unicode 类型的 unicode 类型发出的警告已经放宽，警告适用于甚至不是字符串值的值，比如整数；之前，1.0 的更新警告系统使用字符串格式化操作，这将引发内部 TypeError。虽然理想情况下，这些情况应该完全引发错误，但某些后端如 SQLite 和 MySQL 确实接受它们，并且可能被遗留代码使用，更不用说如果目标后端的 unicode 转换关闭，它们将始终通过。

    参考：[#3346](https://www.sqlalchemy.org/trac/ticket/3346)

### postgresql

+   **[postgresql] [bug]**

    修复了更新 PG 索引反射的 bug，这是 [#3184](https://www.sqlalchemy.org/trac/ticket/3184) 的结果，在 PostgreSQL 版本 8.4 及更早版本上将导致索引操作失败。在使用旧版本的 PostgreSQL 时，这些增强功能现已禁用。

    参考：[#3343](https://www.sqlalchemy.org/trac/ticket/3343)

## 1.0.0b4

Released: March 29, 2015

### sql

+   **[sql] [bug]**

    修复了新“标签解析”功能中的错误[#2992](https://www.sqlalchemy.org/trac/ticket/2992)，其中一个匿名标签，然后再次用名称标记，将无法通过文本标签定位。 当在查询中为映射的`column_property()`给定一个显式标签时，会自然发生这种情况。

    参考：[#3340](https://www.sqlalchemy.org/trac/ticket/3340)

+   **[sql] [bug]**

    修复了新“标签解析”功能中的错误[#2992](https://www.sqlalchemy.org/trac/ticket/2992)，其中在语句的 order_by()或 group_by()中放置的字符串标签会更优先地放在 FROM 子句内找到的名称上，而不是更本地可用的列子句内的名称。

    参考：[#3335](https://www.sqlalchemy.org/trac/ticket/3335)

### 模式

+   **[schema] [feature]**

    诸如`UniqueConstraint`和`CheckConstraint`等约束的“自动附加”功能已进一步增强，以便当约束与非绑定到表的`Column`对象相关联时，约束将与列本身设置事件侦听器，以便约束在与表关联的同时自动附加列。 这特别有助于一些声明性的边缘情况，但也是一般用途���

    另请参阅

    引用未附加列的约束可以在其引用的列附加时自动附加到表

    参考：[#3341](https://www.sqlalchemy.org/trac/ticket/3341)

### mysql

+   **[mysql] [bug] [pymysql]**

    修复了 PyMySQL 在使用“executemany”操作时对 Unicode 的支持。 SQLAlchemy 现在将语句和绑定参数都作为 Unicode 对象传递，因为 PyMySQL 通常在内部使用字符串插值来生成最终语句，并且在 executemany 的情况下仅在最终语句上执行“encode”步骤。

    此更改也**回溯**到：0.9.10

    参考：[#3337](https://www.sqlalchemy.org/trac/ticket/3337)

### mssql

+   **[mssql] [bug] [firebird] [oracle] [sybase]**

    关闭了 MSSQL、Oracle 方言上的“简单排序”标志；这是根据[#2992](https://www.sqlalchemy.org/trac/ticket/2992)的要求，导致 order by 或 group by 表达式也在列子句中时被复制为标签，即使作为表达式对象引用。 MSSQL 现在的行为是默认情况下复制整个表达式，因为 MSSQL 在这些特别是在 GROUP BY 表达式中可能会挑剔。 为 Firebird 和 Sybase 方言也防御性地关闭了该标志。

    注意

    此解决方案是错误的，请查看版本 1.0.2 以重新制定此解决方案。

    参考：[#3338](https://www.sqlalchemy.org/trac/ticket/3338)

## 1.0.0b3

发布日期：2015 年 3 月 20 日

### mysql

+   **[mysql] [错误]**

    修复了无意中注释掉的问题＃2771 的提交。

    参考：[#2771](https://www.sqlalchemy.org/trac/ticket/2771)

## 1.0.0b2

发布日期：2015 年 3 月 20 日

### ORM

+   **[ORM] [错误]**

    修复了从 pullreq github:137 中的意外使用回归，其中 Py2K unicode 文字（例如`u""`）将不被`relationship.cascade`选项接受。拉取请求由 Julien Castets 提供。

    参考：[#3327](https://www.sqlalchemy.org/trac/ticket/3327)

### ORM 声明

+   **[ORM] [声明] [更改]**

    放宽了对`@declared_attr`对象添加的一些限制，以防止它们在声明过程之外被调用；这与#3150 的增强相关，允许`@declared_attr`返回一个根据当前类缓存的值，因为它正在被配置。已删除异常引发，并更改了行为，以便在声明过程之外，由`@declared_attr`装饰的函数每次都像常规`@property`一样被调用，而不使用任何缓存，因为在此阶段没有可用的缓存。

    参考：[#3331](https://www.sqlalchemy.org/trac/ticket/3331)

### 引擎

+   **[引擎] [错误]**

    “自动关闭”`ResultProxy`现在是“软”关闭。也就是说，在使用提取方法耗尽所有行后，DBAPI 游标会像以前一样被释放，对象可以安全丢弃，但是提取方法仍然可以继续调用，它们将返回结果结束对象（对于 fetchone 为 None，对于 fetchmany 和 fetchall 为空列表）。只有显式调用`ResultProxy.close()`时，这些方法才会引发“结果已关闭”错误。

    另请参阅

    ResultProxy“自动关闭”现在是“软”关闭

    参考：[#3329](https://www.sqlalchemy.org/trac/ticket/3329), [#3330](https://www.sqlalchemy.org/trac/ticket/3330)

### mysql

+   **[mysql] [错误] [py3k]**

    修复了 Py3K 上未正确使用`ord()`函数的`BIT`类型。拉取请求由 David Marin 提供。

    此更改也**回溯**到：0.9.10

    参考：[#3333](https://www.sqlalchemy.org/trac/ticket/3333)

+   **[mysql] [错误]**

    修复了完全支持使用 MySQL 方言的`'utf8mb4'` MySQL 特定字符集的问题，特别是 MySQL-Python 和 PyMySQL。此外，报告更不寻常字符集（如‘koi8u’或‘eucjpms’）的 MySQL 数据库也将正常工作。拉取请求由 Thomas Grainger 提供。

    参考：[#2771](https://www.sqlalchemy.org/trac/ticket/2771)

## 1.0.0b1

发布日期：2015 年 3 月 13 日

版本 1.0.0b1 是 1.0 系列的第一个版本。这里描述的许多变化也存在于 0.9 甚至 0.8 系列中。有关特定于 1.0 且着重于兼容性问题的更改，请参阅 SQLAlchemy 1.0 的新功能是什么？。

### 常规

+   **[常规] [功能]**

    通过对许多内部对象更加显著地使用`__slots__`，改善了结构性内存使用。这种优化特别针对具有大量表和列的大型应用程序的基本内存大小，并且大大减少了许多高容量对象的内存大小，包括事件监听内部、比较器对象以及 ORM 属性和加载器策略系统的部分。

    另请参见

    结构性内存使用显著改进

+   **[常规] [错误]**

    对于所有派生为“公共工厂”符号的 SQL 和 ORM 函数，现在都设置了`__module__`属性，这应有助于文档工具能够报告目标模块。

    参考：[#3218](https://www.sqlalchemy.org/trac/ticket/3218)

### ORM

+   **[ORM] [功能]**

    向由`Query.column_descriptions`返回的字典添加了一个新条目`"entity"`。这指的是由表达式引用的主 ORM 映射类或别名类。与现有条目`"type"`相比，它始终是一个映射实体，即使从列表达式中提取，或者如果给定表达式是纯核心表达式，则为 None。另请参见[#3403](https://www.sqlalchemy.org/trac/ticket/3403)，修复了此功能中的一个回归，该回归未在 0.9.10 中发布，但在 1.0 版本中发布了。

    此更改还**回溯**到：0.9.10

    参考：[#3320](https://www.sqlalchemy.org/trac/ticket/3320)

+   **[ORM] [功能]**

    新增了新参数`Session.connection.execution_options`，可用于在事务开始之前首次检查连接时设置`Connection`上的执行选项。这用于在事务开始之前设置连接的选项，如隔离级别等。

    另请参见

    设置事务隔离级别/DBAPI AUTOCOMMIT - 新的文档部分，详细说明使用会话设置事务隔离的最佳实践。

    此更改还**回溯**到：0.9.9

    参考：[#3296](https://www.sqlalchemy.org/trac/ticket/3296)

+   **[ORM] [功能]**

    添加了新的方法`Session.invalidate()`，功能类似于`Session.close()`，除了还调用所有连接的`Connection.invalidate()`，确保它们不会返回到连接池。在某些情况下很有用，例如处理 gevent 超时时，进一步使用连接是不安全的，即使是用于回滚。

    这个更改也被**回溯**到：0.9.9

+   **[orm] [功能]**

    “primaryjoin”模型已经进一步扩展，允许一个严格从单个列到自身的连接条件，通过某种 SQL 函数或表达式进行转换。这有点实验性，但第一个概念验证是一个“材料化路径”连接条件，其中一个路径字符串与自身使用“like”进行比较。`ColumnOperators.like()` 操作符也已添加到可在 primaryjoin 条件中使用的有效操作符列表中。

    这个更改也被**回溯**到：0.9.5

    参考：[#3029](https://www.sqlalchemy.org/trac/ticket/3029)

+   **[orm] [功能]**

    添加了新的实用函数`make_transient_to_detached()`，可用于制造行为就像从会话加载然后分离的对象。不存在的属性被标记为过期，并且对象可以添加到一个会话中，它将表现得像一个持久对象一样。

    这个更改也被**回溯**到：0.9.5

    参考：[#3017](https://www.sqlalchemy.org/trac/ticket/3017)

+   **[orm] [功能]**

    添加了一个新的事件套件`QueryEvents`。`QueryEvents.before_compile()` 事件允许创建函数，在构建 SELECT 语句之前对`Query` 对象进行额外修改。希望通过引入一个新的检查系统，使这个事件更加有用，该系统将允许对`Query` 对象进行详细的自动修改。

    另请参阅

    `QueryEvents`

    参考：[#3317](https://www.sqlalchemy.org/trac/ticket/3317)

+   **[orm] [功能]**

    当使用具有 LIMIT、OFFSET 或 DISTINCT 的一对多查询的连接预加载与一对一关系一起使用时，即一对多与`relationship.uselist`设置为 False 的关系，将禁用发生的子查询包装。这将在这些情况下产生更有效的查询。

    另请参阅

    子查询不再应用于 uselist=False 的连接预加载

    参考：[#3249](https://www.sqlalchemy.org/trac/ticket/3249)

+   **[orm] [功能]**

    映射状态内部已经重新设计，以允许将与对象的“过期”相关的调用次数减少 50%，例如`Session.commit()`的“自动过期”功能以及`Session.expire_all()`中发生的“清理”步骤，当对象状态被垃圾回收时。

    参考：[#3307](https://www.sqlalchemy.org/trac/ticket/3307)

+   **[orm] [功能]**

    当在同一层次结构中为两个不同的映射器分配相同的多态标识时，会发出警告。这通常是用户错误，意味着在加载时无法正确区分这两种不同的映射类型。感谢 Sebastian Bank 的 Pull 请求。

    参考：[#3262](https://www.sqlalchemy.org/trac/ticket/3262)

+   **[orm] [功能]**

    一个新系列的`Session`方法已经被创建，这些方法直接提供了钩子，可以直接进入工作单元的发出 INSERT 和 UPDATE 语句的设施。当正确使用时，这个面向专家的系统可以允许使用 ORM 映射来生成批量插入和更新语句，分组成 executemany 组，使语句的执行速度可以与直接使用 Core 相媲美。

    另请参阅

    批量操作

    参考：[#3100](https://www.sqlalchemy.org/trac/ticket/3100)

+   **[orm] [功能]**

    添加了一个参数`Query.join.isouter`，它与调用`Query.outerjoin()`是同义的；这个标志旨在提供一个更一致的接口，与 Core `FromClause.join()`相比。感谢 Jonathan Vanasco 的 Pull 请求。

    参考：[#3217](https://www.sqlalchemy.org/trac/ticket/3217)

+   **[orm] [功能]**

    添加了新的事件处理程序`AttributeEvents.init_collection()`和`AttributeEvents.dispose_collection()`，用于跟踪集合何时首次与实例关联以及何时被替换。这些处理程序取代了`collection.linker()`注释。通过事件适配器仍支持旧钩子。

+   **[orm] [功能]**

    当`Query.yield_per()`与会发生子查询急加载或使用集合的连接急加载的映射或选项一起使用时，`Query`将引发异常。这些加载策略目前与 yield_per 不兼容，因此通过引发此错误，该方法更安全。可以使用`lazyload('*')`选项或`Query.enable_eagerloads()`来禁用急加载。

    另请参阅

    使用 yield_per 明确禁止 Joined/Subquery 急加载

+   **[orm] [功能]**

    `Query`对象使用的`KeyedTuple`的新实现在获取大量面向列的行时提供了显著的速度改进。

    另请参阅

    新的 KeyedTuple 实现速度显著提高

    参考：[#3176](https://www.sqlalchemy.org/trac/ticket/3176)

+   **[orm] [功能]**

    当内连接的急加载链接到外连接的急加载时，`joinedload.innerjoin`以及`relationship.innerjoin`的行为现在是使用“嵌套”内连接，即右嵌套，作为默认行为。

    另请参阅

    右内连接嵌套现在是使用 innerjoin=True 的 joinedload 的默认设置

    参考：[#3008](https://www.sqlalchemy.org/trac/ticket/3008)

+   **[orm] [功能]**

    现在可以在 ORM 刷新中将 UPDATE 语句批处理为更高效的 executemany()调用，类似于可以批处理 INSERT 语句；这将在刷新中被调用，以便后续对相同映射和表的 UPDATE 语句涉及相同列在 VALUES 子句中，没有嵌入 SET 级别的 SQL 表达式，并且映射的版本要求与后端方言能够为 executemany 操作返回正确的行数兼容。

+   **[orm] [feature]**

    构造函数中已添加`info`参数给`SynonymProperty`和`ComparableProperty`。

    参考：[#2963](https://www.sqlalchemy.org/trac/ticket/2963)

+   **[orm] [feature]**

    `InspectionAttr.info`集合现在已移至`InspectionAttr`，除了在所有`MapperProperty`对象上可用外，现在还可在混合属性、关联代理上通过`Mapper.all_orm_descriptors`访问。

    参考：[#2971](https://www.sqlalchemy.org/trac/ticket/2971)

+   **[orm] [change]**

    标记为延迟加载的映射属性如果没有明确取消延迟加载，即使它们的列以某种方式出现在结果集中，也将保持“延迟加载”。这是一个性能增强，ORM 加载不再在获取结果集时花费时间搜索每个延迟加载列。但是，对于一直依赖于此的应用程序，现在应该使用显式的`undefer()`或类似选项。

+   **[orm] [changed]**

    自定义`Bundle`类的`create_row_processor()`方法传递给`proc()`可调用对象现在只接受单个“row”参数。

    另请参阅

    在使用自定义行加载器时，新 Bundle 功能的 API 更改

+   **[orm] [changed]**

    已移除弃用的事件钩子：`populate_instance`、`create_instance`、`translate_row`、`append_result`

    另请参阅

    已移除弃用的 ORM 事件钩子

+   **[orm] [bug]**

    修复了子查询急加载中的错误，当跨多态子类边界的长链急加载与多态加载一起使用时，会无法定位链中的子类链接，在`AliasedClass`上出现缺少属性名称的错误。

    此更改也已**回溯**至：0.9.5、0.8.7

    参考：[#3055](https://www.sqlalchemy.org/trac/ticket/3055)

+   **[orm] [bug]**

    修复了 ORM 中的一个 bug，即`class_mapper()`函数在映射器配置期间掩盖了应该由于用户错误而引发的 AttributeErrors 或 KeyErrors。对于属性/关键错误的捕获已经更加具体，以排除配置步骤。

    此更改也已**回溯**至：0.9.5, 0.8.7

    参考：[#3047](https://www.sqlalchemy.org/trac/ticket/3047)

+   **[orm] [bug]**

    修复了 ORM 对象比较中的错误，其中多对一的`!= None`比较将失败，如果源是一个别名类，或者如果查询需要对表达式应用特殊别名处理，原因是别名连接或多态查询；还修复了以下情况的错误：如果比较多对一与对象状态，则如果查询需要对别名连接或多态查询应用特殊别名，则会失败。

    此更改也已**回溯**至：0.9.9

    参考：[#3310](https://www.sqlalchemy.org/trac/ticket/3310)

+   **[orm] [bug]**

    修复了在`after_rollback()`处理程序为一个`Session`不正确地在处理程序中添加状态，并且尝试警告和删除此状态的任务（由[#2389](https://www.sqlalchemy.org/trac/ticket/2389)建立）的情况下，内部断言会失败的错误。

    此更改也已**回溯**至：0.9.9

    参考：[#3309](https://www.sqlalchemy.org/trac/ticket/3309)

+   **[orm] [bug]**

    修复了当`Query.join()`调用具有未知 kw 参数时引发 TypeError 的错误，由于格式错误，会引发自己的 TypeError。感谢 Malthe Borch 提供的拉取请求。

    此更改也已**回溯**至：0.9.9

+   **[orm] [bug]**

    修复了惰性加载 SQL 构造中的错误，其中复杂的 primaryjoin 多次引用了相同的“本地”列，以“指向自身的列”样式的自引用连接将不在所有情况下进行替换。确定替换的逻辑已经重新设计为更加开放式。

    此更改也已**回溯**至：0.9.9

    参考：[#3300](https://www.sqlalchemy.org/trac/ticket/3300)

+   **[orm] [bug]**

    “通配符”加载器选项，特别是由`load_only()`选项设置的选项，以覆盖未明确提及的所有属性，现在考虑给定实体的超类，如果该实体使用继承映射进行映射，则超类中的属性名称也将从加载中省略。此外，多态鉴别器列无条件地包含在列表中，就像主键列一样，因此即使设置了 load_only()，子类型的多态加载仍将正常工作。

    此更改也**回溯**到：0.9.9

    参考：[#3287](https://www.sqlalchemy.org/trac/ticket/3287)

+   **[orm] [bug] [pypy]**

    修复了一个错误，即如果在`Query`开始获取结果之前抛出异常，特别是当无法形成行处理器时，游标将保持打开状态，结果仍在等待中，实际上不会关闭。这通常只在像 PyPy 这样的解释器上出现问题，其中游标不会立即被 GC 回收，并且在某些情况下可能导致事务/锁定打开时间超过所需时间。

    此更改也**回溯**到：0.9.9

    参考：[#3285](https://www.sqlalchemy.org/trac/ticket/3285)

+   **[orm] [bug]**

    修复了一个泄漏问题，该问题会在不支持且极不推荐的情况下多次替换固定映射类上的关系时发生，这些关系指向任意增长的目标映射器数量。在替换旧关系时会发出警告，但如果映射已用于查询，则旧关系仍将在某些注册表中被引用。

    此更改也**回溯**到：0.9.9

    参考：[#3251](https://www.sqlalchemy.org/trac/ticket/3251)

+   **[orm] [bug] [sqlite]**

    修复了关于表达式变异的错误，当使用`Query`从 SQLite 中选择多个匿名列实体进行查询时，可能会表现为“无法找到列”错误，这是由 SQLite 方言使用的“联接重写”功能的副作用。

    此更改也**回溯**到：0.9.9

    参考：[#3241](https://www.sqlalchemy.org/trac/ticket/3241)

+   **[orm] [bug]**

    修复了一个错误，即当使用`of_type()`连接到单一继承子类时，`Query.join()`和`Query.outerjoin()`的 ON 子句不会在 ON 子句中呈现“单表条件”，如果设置了`from_joinpoint=True`标志。

    此更改也**回溯**到：0.9.9

    参考：[#3232](https://www.sqlalchemy.org/trac/ticket/3232)

+   **[orm] [bug] [engine]**

    修复了通常影响与[#3199](https://www.sqlalchemy.org/trac/ticket/3199)相同类事件的错误，当使用`named=True`参数时。一些事件将无法注册，而其他事件将无法正确调用事件参数，通常在事件以某种其他方式“包装”以适应时。已重新排列“命名”机制，以不干扰内部包装函数期望的参数签名。

    这个更改也被**回溯**到：0.9.8

    参考：[#3197](https://www.sqlalchemy.org/trac/ticket/3197)

+   **[orm] [bug]**

    修复了影响许多类事件的错误，特别是 ORM 事件，但也包括引擎事件，其中“去重复”冗余调用`listen()`的通常逻辑会失败，对于那些监听器函数被包装的事件，registry.py 中会触发一个断言。现在，这个断言已经整合到去重复检查中，另外还有一个更简单的方法来全面检查去重复。

    这个更改也被**回溯**到：0.9.8

    参考：[#3199](https://www.sqlalchemy.org/trac/ticket/3199)

+   **[orm] [bug]**

    修复了当复杂的自引用主连接包含函数时会发出警告的问题，同时指定了 remote_side；警告会建议设置“remote side”。现在只有在 remote_side 不存在时才会发出警告。

    这个更改也被**回溯**到：0.9.8

    参考：[#3194](https://www.sqlalchemy.org/trac/ticket/3194)

+   **[orm] [bug] [eagerloading]**

    修复了在 0.9.4 中发布的[#2976](https://www.sqlalchemy.org/trac/ticket/2976)引起的回归，其中沿着一系列连接的急加载链传播“外连接”会错误地将兄弟连接路径上的“内连接”也转换为外连接，当只有后代路径应该接收“外连接”传播时；另外，修复了相关问题，即“嵌套”连接传播会不恰当地发生在两个兄弟连接路径之间。

    这个更改也被**回溯**到：0.9.7

    参考：[#3131](https://www.sqlalchemy.org/trac/ticket/3131)

+   **[orm] [bug]**

    由于[#2736](https://www.sqlalchemy.org/trac/ticket/2736)导致的从 0.9.0 回归，`Query.select_from()`方法不再正确设置`Query`对象的“from entity”，因此后续的`Query.filter_by()`或`Query.join()`调用将无法在按字符串名称搜索属性时检查适当的“from”实体。

    此更改也**回溯**到：0.9.7

    参考：[#2736](https://www.sqlalchemy.org/trac/ticket/2736), [#3083](https://www.sqlalchemy.org/trac/ticket/3083)

+   **[orm] [bug]**

    修复了在保存点块内持久化、删除或主键更改的项目在外部事务回滚后不参与恢复到其先前状态（不在会话中、在会话中、先前 PK）的错误。

    此更改也**回溯**到：0.9.7

    参考：[#3108](https://www.sqlalchemy.org/trac/ticket/3108)

+   **[orm] [bug]**

    修复了子查询急切加载与`with_polymorphic()`一起使用时的错误，子查询加载中实体和列的定位相对于此类型的实体和其他实体更加准确。

    此更改也**回溯**到：0.9.7

    参考：[#3106](https://www.sqlalchemy.org/trac/ticket/3106)

+   **[orm] [bug]**

    针对继承映射器隐式组合其基于列的属性与父级属性的情况，已添加额外的检查，其中这些列通常不一定共享相同的值。这是通过[#1892](https://www.sqlalchemy.org/trac/ticket/1892)添加的现有检查的扩展；然而，这个新检查只发出警告，而不是异常，以允许可能依赖现有行为的应用程序。

    另请参见

    我收到关于“在属性 Y 下隐式组合列 X”警告或错误

    此更改也**回溯**到：0.9.5

    参考：[#3042](https://www.sqlalchemy.org/trac/ticket/3042)

+   **[orm] [bug]**

    修改了`load_only()`的行为，使得主键列始终添加到“未延迟加载”列的列表中；否则，ORM 无法加载行的标识。显然，可以延迟映射的主键，ORM 将失败，这一点没有改变。但是，由于 load_only 本质上是说“除了 X 之外都延迟加载”，因此 PK 列不参与此延迟加载更为关键。

    此更改也**回溯**到：0.9.5

    参考：[#3080](https://www.sqlalchemy.org/trac/ticket/3080)

+   **[orm] [bug]**

    修复了在所谓的“行切换”场景中出现的一些边缘情况，其中 INSERT/DELETE 可以转换为 UPDATE。在这种情况下，将一个多对一关系设置为 None，或在某些情况下将标量属性设置为 None，可能不会被检测为值的净变化，因此 UPDATE 不会重置前一行上的内容。这是由于属性历史的一些尚未解决的副作用导致的，这些副作用在隐式假定 None 对于先前未设置属性实际上不是“变化”。另请参见[#3061](https://www.sqlalchemy.org/trac/ticket/3061)。

    注意

    此更改已在 0.9.6 版本中**撤销**。完整修复将在 SQLAlchemy 的 1.0 版本中实现。

    此更改也已**回溯**至：0.9.5

    参考：[#3060](https://www.sqlalchemy.org/trac/ticket/3060)

+   **[orm] [bug]**

    与[#3060](https://www.sqlalchemy.org/trac/ticket/3060)相关，对工作单元进行了调整，以便对于要删除的自引用对象图的相关多对一对象的加载略微更加积极；如果未设置`passive_deletes`，则相关对象的加载将有助于确定正确的删除顺序。

    此更改也已**回溯**至：0.9.5

+   **[orm] [bug]**

    修复了 SQLite 联接重写中匿名列名由于重复而无法在子查询中正确重写的错误。这将影响带有任何类型的子查询 + 联接的 SELECT 查询。

    此更改也已**回溯**至：0.9.5

    参考：[#3057](https://www.sqlalchemy.org/trac/ticket/3057)

+   **[orm] [bug] [sql]**

    修复了对[#2804](https://www.sqlalchemy.org/trac/ticket/2804)中新增的布尔强制转换的修复，其中“where”和“having”的新规则不会对`select()`构造函数的“whereclause”和“having” kw 参数产生影响，这也是`Query`所使用的，因此在 ORM 中也无法正常工作。

    此更改也已**回溯**至：0.9.5

    参考：[#3013](https://www.sqlalchemy.org/trac/ticket/3013)

+   **[orm] [bug]**

    修复了会话附加错误“对象已附加到会话 X”未能阻止该对象在错误引发后继续执行的情况下，也附加到新会话的错误。 

    参考：[#3301](https://www.sqlalchemy.org/trac/ticket/3301)

+   **[orm] [bug]**

    当调用`Query.count()`、`Query.update()`、`Query.delete()`以及针对映射列、`column_property`对象以及从映射列派生的 SQL 函数和表达式的查询时，`Query`的主要`Mapper`现在会传递给`Session.get_bind()`方法。这样，依赖于自定义`Session.get_bind()`方案或“绑定”元数据的会话可以在所有相关情况下工作。

    另请参阅

    在所有相关查询情况下，Session.get_bind() 将接收 Mapper

    参考：[#1326](https://www.sqlalchemy.org/trac/ticket/1326)、[#3227](https://www.sqlalchemy.org/trac/ticket/3227)、[#3242](https://www.sqlalchemy.org/trac/ticket/3242)

+   **[orm] [bug]**

    与`joinedload()`和`contains_eager()`等加载器指令一起，`PropComparator.of_type()`修饰符已经改进，以便在遇到两个相同基本类型/路径的`PropComparator.of_type()`修饰符时，它们将被合并为单个“多态”实体，而不是用类型 A 的实体替换类型 B 的实体。例如，`A.b.of_type(BSub1)->BSub1.c` 的 joinedload 与 `A.b.of_type(BSub2)->BSub2.c` 的 joinedload 将创建一个单个 joinedload，即 `A.b.of_type((BSub1, BSub2)) -> BSub1.c, BSub2.c`，而不需要在查询中显式使用 `with_polymorphic`。

    另请参阅

    多态子类型的急加载 - 包含一个更新的示例，说明了新的格式。

    参考：[#3256](https://www.sqlalchemy.org/trac/ticket/3256)

+   **[orm] [bug]**

    修复了当由 `CascadeOptions` 参数使用时 `copy.deepcopy()` 调用的支持，这种情况发生在 `relationship()` 使用 `copy.deepcopy()` 时（不是官方支持的用例）。请求已经由 duesenfranz 提出。

+   **[orm] [bug]**

    修复了一个 bug，即当对象经历了刷新但未提交的删除操作时，`Session.expunge()` 不会完全分离给定的对象。这也会影响到像 `make_transient()` 这样的相关操作。

    另请参阅

    session.expunge() 现在会完全分离已删除的对象

    参考文献：[#3139](https://www.sqlalchemy.org/trac/ticket/3139)

+   **[orm] [bug]**

    如果多个关系最终将填充与另一个冲突的外键列，则会发出警告，其中关系试图从不同的源列复制值。这种情况发生在将具有重叠列的复合外键映射到每个引用列都不同的关系时。新的文档部分演示了示例以及如何通过在每个关系基础上明确指定“foreign”列来克服此问题。

    另请参阅

    重叠的外键

    参考文献：[#3230](https://www.sqlalchemy.org/trac/ticket/3230)

+   **[orm] [bug]**

    `Query.update()` 方法现在将给定值字典中的字符串键名转换为正在更新的映射类的映射属性名称。以前，字符串名称直接被接受并传递给核心更新语句，没有任何方法解析到映射实体。支持使用 `Query.update()` 的主题属性的同义词和混合属性也得到了支持。

    另请参阅

    query.update() 现在将字符串名称解析为映射属性名称

    参考文献：[#3228](https://www.sqlalchemy.org/trac/ticket/3228)

+   **[orm] [bug]**

    对 `Session` 用于定位“绑定”（例如要使用的引擎）的机制进行了改进，这些引擎可以与混合类、具体子类以及更广泛的表元数据（如联接继承表）关联。

    另请参阅

    Session.get_bind() 处理更广泛的继承场景

    参考：[#3035](https://www.sqlalchemy.org/trac/ticket/3035)

+   **[orm] [bug]**

    修复了单表继承中的一个 bug，其中包含同一个单一继承实体超过一次的连接链（通常应该引发错误），在某些情况下，取决于从哪里连接，可能会隐式别名第二个单一继承实体的情况，生成一个“有效”的查询。但由于在单表继承的情况下并不打算进行这种隐式别名，它并没有真正“有效”，而且非常具有误导性，因为它并不总是出现。

    另请参阅

    处理重复连接目标的更改和修复

    参考：[#3233](https://www.sqlalchemy.org/trac/ticket/3233)

+   **[orm] [bug]**

    当使用 `Query.join()`、`Query.outerjoin()` 或独立的 `join()` / `outerjoin()` 函数连接到单一继承子类时，ON 子句现在将包括“单表条件”，即使 ON 子句是手动编写的；它现在使用 AND 将条件添加到 ON 子句中，方式与使用 relationship 或类似方式连接到单表目标时相同。

    这有点介于功能和 bug 之间。

    另请参阅

    单表继承条件无条件地添加到所有 ON 子句中

    参考：[#3222](https://www.sqlalchemy.org/trac/ticket/3222)

+   **[orm] [bug]**

    对表达式标签的行为进行了重大改进，特别是在与具有自定义 SQL 表达式的 ColumnProperty 结构一起使用时，以及与首次引入的“order by labels”逻辑结合使用时。修复包括，现在 `order_by(Entity.some_col_prop)` 将即使 Entity 经过别名处理，也会使用“order by label”规则；多次使用别名渲染相同的列属性（例如 `query(Entity.some_prop, entity_alias.some_prop)`) 将为实体的每次出现标记一个不同的标签，并且此外，“order by label”规则将适用于两者（例如 `order_by(Entity.some_prop, entity_alias.some_prop)`）。在 0.9 版本中可能阻止“order by label”逻辑工作的其他问题，特别是标签的状态可能会发生变化，以至于“order by label”会停止工作，具体取决于如何调用，已经修复。

    另请参阅

    ColumnProperty 结构在别名、order_by 方面工作得更好

    参考：[#3148](https://www.sqlalchemy.org/trac/ticket/3148), [#3188](https://www.sqlalchemy.org/trac/ticket/3188)

+   **[orm] [bug]**

    更改了使用 `Query.from_self()` 或其常用用户 `Query.count()` 时应用“单继承条件”的方法。现在，在内部子查询中指示将行限制为具有特定类型的标准，而不是在外部子查询中指示，因此即使“类型”列不在列子句中可用，我们也可以在“内部”查询中对其进行过滤。

    另见

    在使用 from_self()、count() 时更改为单表继承条件

    参考：[#3177](https://www.sqlalchemy.org/trac/ticket/3177)

+   **[orm] [bug]**

    对延迟加载的机制进行了微小调整，以使其在非常罕见的情况下几乎不会干扰 joinload()；在这种情况下，对象在加载其属性时会引用自身，这可能会导致加载程序之间的混淆。目前不完全支持“对象指向自身”的用例，但此修复还删除了一些开销，因此目前是测试的一部分。

    参考：[#3145](https://www.sqlalchemy.org/trac/ticket/3145)

+   **[orm] [bug]**

    已删除“复活” ORM 事件。自从在 0.8 版本中删除了旧的“可变属性”系统以来，此事件挂钩已经没有作用。

    参考：[#3171](https://www.sqlalchemy.org/trac/ticket/3171)

+   **[orm] [bug]**

    修复了一个 bug，在该 bug 中，“set” 事件或带有 `@validates` 的列在刷新过程中会触发事件，当这些列是“获取和填充”操作的目标时，例如自动增量主键、Python 端默认值或通过 RETURNING “急切地”获取的服务器端默认值。

    参考：[#3167](https://www.sqlalchemy.org/trac/ticket/3167)

+   **[orm] [bug] [py3k]**

    从 `Session.identity_map` 中暴露的 `IdentityMap` 现在在 Py3K 中为 `items()` 和 `values()` 返回列表。此处的早期移植到 Py3K 将这些返回迭代器，但它们在技术上应该是“可迭代视图”。暂时，列表是可以的。

+   **[orm] [bug]**

    用于 query.update()/delete() 的“评估器”在多表更新时不起作用，需要设置为 synchronize_session=False 或 synchronize_session=’fetch’；现在会引发异常，并显示更改同步设置的消息。这是从 0.9.7 版本开始发出的警告升级为异常。

    参考：[#3117](https://www.sqlalchemy.org/trac/ticket/3117)

+   **[orm] [enhancement]**

    调整属性机制，关于何时通过首次访问隐式初始化值为 None；此操作，通常导致属性的填充，不再这样做；返回 None 值，但底层属性不接收设置事件。这与集合的工作方式一致，并允许属性机制更一致地行为；特别是，获取没有值的属性不会压制事件，如果实际上将值设置为 None，则应该继续进行。

    另请参阅

    属性事件和其他操作更改，涉及没有预先存在值的属性

    在编译时选项基础上，绑定参数作为字符串内联呈现。此功能的工作由 Dobes Vandermeer 提供。

    > 另请参阅
    > 
    > 选择/查询 LIMIT/OFFSET 可以指定为任意 SQL 表达式。

    参考：[#3061](https://www.sqlalchemy.org/trac/ticket/3061)

### orm 声明性

+   **[orm] [declarative] [feature]**

    `declared_attr`结构在与声明性结合使用时具有新的改进行为和特性。装饰的函数现在在调用时将有权访问本地混合上存在的最终列副本，并且对于每个映射的类，将确切地调用一次，返回的结果被缓存。还添加了一个新的修饰符`declared_attr.cascading`。

    另请参阅

    改进声明性混合，@declared_attr 和相关功能

    参考：[#3150](https://www.sqlalchemy.org/trac/ticket/3150)

+   **[orm] [declarative] [bug]**

    修复了在与声明`__abstract__`的子类一起使用`AbstractConcreteBase`时，“'NoneType'对象没有属性'concrete'”错误。

    此更改也**被回溯到**：0.9.8

    参考：[#3185](https://www.sqlalchemy.org/trac/ticket/3185)

+   **[orm] [declarative] [bug]**

    修复了在声明性继承层次结构的中间使用`__abstract__`混合时，属性和配置无法从基类正确传播到继承类的错误。

    参考：[#3219](https://www.sqlalchemy.org/trac/ticket/3219), [#3240](https://www.sqlalchemy.org/trac/ticket/3240)

+   **[orm] [declarative] [bug]**

    使用`declared_attr`在`AbstractConcreteBase`基类上设置的关系现在将自动配置在抽象基础映射上，除了像往常一样在后代具体类上设置。

    另请参阅

    对声明性混合、@declared_attr 和相关功能的改进

    参考：[#2670](https://www.sqlalchemy.org/trac/ticket/2670)

### 示例

+   **[examples] [feature]**

    添加了一个新示例，演示了使用最新的关系特性的物化路径。示例由 Jack Zhou 提供。

    此更改也**回溯**到：0.9.5

+   **[examples] [feature]**

    一个新的示例套件，专门提供了对 SQLAlchemy ORM 和 Core 以及 DBAPI 性能的详细研究，从多个角度进行。该套件在一个容器中运行，通过控制台输出以及通过 RunSnake 工具图形显示提供内置的性能分析显示。

    另请参阅

    性能

+   **[examples] [bug]**

    更新了带有历史表的版本控制示例，使映射列重新映射以匹配列名以及列分组；特别是，这允许在同名列继承场景中明确分组的列在历史映射中以相同方式映射，避免了 0.9 系列中关于此模式的警告，并允许属性键的相同视图。

    此更改也**回溯**到：0.9.9

+   **[examples] [bug]**

    修复了示例/generic_associations/discriminator_on_association.py 中的一个错误，其中 AddressAssociation 的子类未被映射为“单表继承”，导致在尝试进一步使用映射时出现问题。

    此更改也**回溯**到：0.9.9

### engine

+   **[engine] [feature]**

    添加了用于查看事务隔离级别的新用户空间访问器；`Connection.get_isolation_level()`，`Connection.default_isolation_level`。

    此更改也**回溯**到：0.9.9

+   **[engine] [feature]**

    添加了新事件`ConnectionEvents.handle_error()`，这是对`ConnectionEvents.dbapi_error()`更全面和全面的替代。

    此更改也**回溯**到：0.9.7

    参考：[#3076](https://www.sqlalchemy.org/trac/ticket/3076)

+   **[engine] [feature]**

    可以发出一种新的警告样式，该样式将“过滤”掉最多 N 次出现的参数化字符串。这允许参数化警告引用其参数，以便在固定次数内传递，直到允许 Python 警告过滤器将其消除，并防止内存在 Python 的警告注册表中无限增长。

    另请参见

    Session.get_bind() 处理更广泛的继承场景

    参考：[#3178](https://www.sqlalchemy.org/trac/ticket/3178)

+   **[engine] [bug]**

    修复了`Connection`和池中的错误，当使用`Connection.execution_options()`时，如果使用了`isolation_level`参数，则`Connection.invalidate()`方法或由于数据库断开连接而使连接无效时会失败；重置隔离级别的“finalizer”将在不再打开的连接上调用。

    此更改也**回溯**至：0.9.9

    参考：[#3302](https://www.sqlalchemy.org/trac/ticket/3302)

+   **[engine] [bug]**

    如果在进行`Transaction`时使用`isolation_level`参数与`Connection.execution_options()`一起使用，则会发出警告；DBAPIs 和/或 SQLAlchemy 方言（如 psycopg2、MySQLdb）可能会隐式回滚或提交事务，或者在下一个事务中不更改设置，因此这永远不安全。

    此更改也**回溯**至：0.9.9

    参考：[#3296](https://www.sqlalchemy.org/trac/ticket/3296)

+   **[engine] [bug]**

    通过 `create_engine.execution_options` 或 `Engine.update_execution_options()` 传递给 `Engine` 的执行选项不会传递给用于在“首次连接”事件中初始化方言的特殊 `Connection`；方言通常会在此阶段执行自己的查询，并且不应在此处应用任何当前可用的选项。特别是，“autocommit” 选项导致在此初始连接中尝试自动提交，这将由于 `Connection` 的非标准状态而导致 AttributeError 失败。

    此更改也已**回溯**至：0.9.8

    参考：[#3200](https://www.sqlalchemy.org/trac/ticket/3200)

+   **[引擎] [错误]**

    用于确定受 INSERT 或 UPDATE 影响的列的字符串键现在在它们对“编译缓存”缓存键的贡献时进行排序。这些键以前是没有确定性顺序的，这意味着相同的语句可能会根据等效键被多次缓存，这样做既消耗内存又影响性能。

    此更改也已**回溯**至：0.9.8

    参考：[#3165](https://www.sqlalchemy.org/trac/ticket/3165)

+   **[引擎] [错误]**

    修复了一个错误，该错误可能会在引擎首次连接并执行其初始检查时发生 DBAPI 异常，并且异常不是断开连接异常，但是当我们尝试关闭游标时，游标引发错误。在这种情况下，由于我们试图通过连接池记录游标关闭异常并失败，因为我们试图以这种非常特定的方式访问池的记录器，因此真正的异常将被压制。

    此更改也已**回溯**至：0.9.5

    参考：[#3063](https://www.sqlalchemy.org/trac/ticket/3063)

+   **[引擎] [错误]**

    修复了一些“双重失效”情况，其中可能会检测到连接失效，而这些连接失效可能会在已经是关键部分的情况下发生，例如连接关闭；最终，这些条件是由 [#2907](https://www.sqlalchemy.org/trac/ticket/2907) 中的更改引起的，该更改在“返回时重置”功能中调用 Connection/Transaction 以处理它，在那里可能会被“断开连接检测”捕获。但是，可能最近对 [#2985](https://www.sqlalchemy.org/trac/ticket/2985) 的更改使这更有可能被视为“连接失效”操作更快，因为在 0.9.4 上更容易复现该问题而不是在 0.9.3 上。

    现在在可能发生无效操作的任何部分都添加了检查，以阻止在无效连接上进一步禁止的操作。这包括引擎级别和池级别的两个修复。虽然这个问题在高度并发的 gevent 情况下被观察到，但理论上在任何连接关闭操作中发生断开连接的情况下都可能发生。

    这个更改也被**回溯**到：0.9.5

    参考：[#3043](https://www.sqlalchemy.org/trac/ticket/3043)

+   **[engine] [bug]**

    引擎级别的错误处理和包装程序现在将在所有引擎连接使用情况下生效，包括当用户自定义连接例程通过`create_engine.creator`参数使用时，以及当`Connection`在重新验证时遇到连接错误时。

    另请参阅

    DBAPI 异常包装和 handle_error()事件改进

    参考：[#3266](https://www.sqlalchemy.org/trac/ticket/3266)

+   **[engine] [bug]**

    现在，在事件监听器被运行时同时移除（或添加）事件监听器，无论是从监听器内部还是从并发线程中，都会引发 RuntimeError，因为现在使用的集合是`collections.deque()`的实例，不支持在迭代时进行更改。以前使用的是普通的 Python 列表，其中在事件本身内部移除将导致静默失败。

    参考：[#3163](https://www.sqlalchemy.org/trac/ticket/3163)

### sql

+   **[sql] [feature]**

    在`Index`的合同中稍微放宽了一点，现在可以将`text()`表达式指定为目标；如果要手动将索引添加到表中，则索引不再需要存在绑定列，可以通过内联声明或通过`Table.append_constraint()`添加到表中。

    这个更改也被**回溯**到：0.9.5

    参考：[#3028](https://www.sqlalchemy.org/trac/ticket/3028)

+   **[sql] [feature]**

    添加了新标志`between.symmetric`，当设置为 True 时，会呈现“BETWEEN SYMMETRIC”。还添加了一个新的否定运算符“notbetween_op”，现在允许像`~col.between(x, y)`这样的表达式呈现为“col NOT BETWEEN x AND y”，而不是带括号的 NOT 字符串。

    这个更改也被**回溯**到：0.9.5

    参考：[#2990](https://www.sqlalchemy.org/trac/ticket/2990)

+   **[sql] [feature]**

    现在 SQL 编译器生成预期列的映射，使它们按位置与接收到的结果集匹配，而不是按名称。最初，这被视为处理列返回具有难以预测名称的情况的一种方式，尽管在现代使用中，通过匿名标记已经克服了这个问题。在这个版本中，该方法基本上通过减少每个结果的函数调用次数几十次，或者对于更大的结果列集合可能更多。如果编译的列集与接收到的列存在大小上的差异，该方法仍会退化为旧方法的现代版本，因此在这些列表可能不对齐的部分或完全文本编译场景中没有问题。

    参考：[#918](https://www.sqlalchemy.org/trac/ticket/918)

+   **[sql] [功能]**

    在`DefaultClause`中的字面值，当使用`Column.server_default`参数时被调用，现在将使用“内联”编译器进行呈现，以便它们按原样呈现，而不是作为绑定参数。

    另请参阅

    列服务器默认值现在呈现字面值

    参考：[#3087](https://www.sqlalchemy.org/trac/ticket/3087)

+   **[sql] [功能]**

    当传递给 SQL 表达式单元的对象无法解释为 SQL 片段时，将报告表达式的类型；感谢 Ryan P. Kelly 提交的拉取请求。

+   **[sql] [功能]**

    向`Table.tometadata()`方法添加了一个新参数`Table.tometadata.name`。类似于`Table.tometadata.schema`，此参数使新复制的`Table`采用新名称而不是现有名称。这样做的一个有趣功能是可以将`Table`对象复制到具有新名称的*相同* `MetaData` 目标中。感谢 n.d. parker 提交的拉取请求。

+   **[sql] [功能]**

    异常消息已经稍微改进。如果为 None，则不显示 SQL 语句和参数，减少与与语句无关的错误消息的混淆。显示了 DBAPI 级别异常的完整模块和类名，清楚地表明这是一个包装的 DBAPI 异常。语句和参数本身被限定在括号内的部分，以更好地将它们与错误消息和彼此隔离开来。

    参考：[#3172](https://www.sqlalchemy.org/trac/ticket/3172)

+   **[sql] [feature]**

    如果未另行指定，则 `Insert.from_select()` 现在包括 Python 和 SQL 表达式默认值；解除了非服务器列默认值不包��在 INSERT FROM SELECT 中的限制，并且这些表达式被渲染为常量插入到 SELECT 语句中。

    另请参阅

    INSERT FROM SELECT 现在包括 Python 和 SQL 表达式默认值

+   **[sql] [feature]**

    当反射一个 `Table` 对象时，现在包括 `UniqueConstraint` 构造，适用于这些数据库。为了准确地实现这一点，MySQL 和 PostgreSQL 现在包含了在反射表、索引和约束时纠正索引和唯一约束重复的功能。对于 MySQL，实际上没有独立于“唯一索引”的“唯一约束”概念，因此对于这个后端，反射的 `Table` 中仍然不包含 `UniqueConstraint`。对于 PostgreSQL，用于检测 `pg_index` 中的索引的查询已经改进，以检查 `pg_constraint` 中的相同构造，并且隐式构建的唯一索引不包括在反射的 `Table` 中。

    在这两种情况下，`Inspector.get_indexes()` 和 `Inspector.get_unique_constraints()` 方法分别返回这两个构造，但在 PostgreSQL 的情况下包括一个新的标记 `duplicates_constraint`，在 MySQL 的情况下包括一个 `duplicates_index` 标记以指示检测到此条件时。感谢 Johannes Erdfelt 的拉取请求。

    另请参阅

    唯一约束现在是表反射过程的一部分

    参考：[#3184](https://www.sqlalchemy.org/trac/ticket/3184)

+   **[sql] [feature]**

    添加了新方法`Select.with_statement_hint()`和 ORM 方法`Query.with_statement_hint()`，以支持不特定于表的语句级提示。

    参考：[#3206](https://www.sqlalchemy.org/trac/ticket/3206)

+   **[sql] [feature]**

    `info`参数已添加为所有模式构造函数的构造参数，包括`MetaData`、`Index`、`ForeignKey`、`ForeignKeyConstraint`、`UniqueConstraint`、`PrimaryKeyConstraint`、`CheckConstraint`。

    参考：[#2963](https://www.sqlalchemy.org/trac/ticket/2963)

+   **[sql] [feature]**

    `Table.autoload_with`标志现在意味着`Table.autoload`应该为`True`。感谢 Malik Diarra 的拉取请求。

    参考：[#3027](https://www.sqlalchemy.org/trac/ticket/3027)

+   **[sql] [feature]**

    `Select.limit()`和`Select.offset()`方法现在接受任何 SQL 表达式作为参数，除了整数值。通常用于允许传递绑定参数，稍后可以用值替换，从而允许在 Python 端缓存 SQL 查询。这里的实现完全向后兼容现有的第三方方言，但是那些实现特殊 LIMIT/OFFSET 系统的方言将需要修改以利用新功能。Limit 和 offset 还支持“literal_binds”模式，

    参考：[#3034](https://www.sqlalchemy.org/trac/ticket/3034)

+   **[sql] [changed]**

    `column()` 和 `table()` 构造现在可以从“from sqlalchemy”命名空间导入，就像其他所有 Core 构造一样。

+   **[sql] [changed]**

    当传递给大多数 `select()` 的构建器方法以及 `Query` 时，将字符串隐式转换为 `text()` 构造现在会发出警告。文本转换仍然正常进行。唯一不发出警告的方法是“标签引用”方法，如 order_by()、group_by()；这些函数现在在编译时将尝试将单个字符串参数解析为可选择的列或标签表达式；如果找不到，则表达式仍然呈现，但您会再次收到警告。这里的理由是，从字符串到文本的隐式转换如今更加意外，当用户将原始字符串传递给 Core/ORM 时，最好发送更多方向以指示应采取什么方向。Core/ORM 教程已更新，更深入地介绍了如何处理文本。

    请参阅

    将完整的 SQL 片段强制转换为 text() 时发出的警告

    参考：[#2992](https://www.sqlalchemy.org/trac/ticket/2992)

+   **[sql] [bug]**

    在 `Enum` 和其他 `SchemaType` 子类中修复了一个 bug，直接将该类型与 `MetaData` 关联会导致在 `MetaData` 上发出事件（如创建事件）时挂起。

    此更改也**被反向移植**至：0.9.7、0.8.7

    参考：[#3124](https://www.sqlalchemy.org/trac/ticket/3124)

+   **[sql] [bug]**

    修复了在自定义运算符加上 `TypeEngine.with_variant()` 系统中的一个 bug，使用 `TypeDecorator` 与 variant 时，当使用比较运算符时会失败并出现 MRO 错误。

    此更改也**被反向移植**至：0.9.7、0.8.7

    参考：[#3102](https://www.sqlalchemy.org/trac/ticket/3102)

+   **[sql] [bug]**

    修复了在从 UNION 选择时，INSERT..FROM SELECT 构造中的错误，会将 UNION 包装在一个匿名（例如未标记）子查询中。

    此更改也被**回溯**到：0.9.5, 0.8.7

    参考：[#3044](https://www.sqlalchemy.org/trac/ticket/3044)

+   **[sql] [bug]**

    修复了当应用空的`and_()`或`or_()`或其他空表达式时，`Table.update()`和`Table.delete()`会生成一个空的 WHERE 子句的错误。现在与`select()`的行为一致。

    此更改也被**回溯**到：0.9.5, 0.8.7

    参考：[#3045](https://www.sqlalchemy.org/trac/ticket/3045)

+   **[sql] [bug]**

    在`Enum`的`__repr__()`输出中添加了`native_enum`标志，当与 Alembic autogenerate 一起使用时，这一点非常重要。感谢 Dimitris Theodorou 的拉取请求。

    此更改也被**回溯**到：0.9.9

+   **[sql] [bug]**

    修复了使用实现了也是`TypeDecorator`的类型的`TypeDecorator`会在针对使用此类型的对象使用任何类型的 SQL 比较表达式时，导致 Python 的“无法创建一致的方法解析顺序（MRO）”错误的错误。

    此更改也被**回溯**到：0.9.9

    参考：[#3278](https://www.sqlalchemy.org/trac/ticket/3278)

+   **[sql] [bug]**

    修复了在 INSERT 中嵌入的 SELECT 中的列，无论是通过值子句还是作为“from select”，都会在具有相同名称的两个语句的列类型中污染由 RETURNING 子句产生的结果集中使用的列类型，导致在检索返回行时可能出现错误或错误适应。

    此更改也被**回溯**到：0.9.9

    参考：[#3248](https://www.sqlalchemy.org/trac/ticket/3248)

+   **[sql] [bug]**

    修复了 sql 包中的许多 SQL 元素无法成功执行`__repr__()`的错误，由于缺少`description`属性，然后会调用递归溢出，当内部 AttributeError 再���调用`__repr__()`时。

    此更改也被**回溯**到：0.9.8

    参考：[#3195](https://www.sqlalchemy.org/trac/ticket/3195)

+   **[sql] [bug]**

    调整表/索引反射，如果索引报告一个在表中找不到的列，则会发出警告并跳过该列。这可能发生在一些特殊的系统列情况下，如在 Oracle 中观察到的情况。

    此更改也**回溯**到：0.9.8

    参考：[#3180](https://www.sqlalchemy.org/trac/ticket/3180)

+   **[sql] [bug]**

    修复了 CTE 中的错误，其中当一个 CTE 引用另一个别名 CTE 时，`literal_binds` 编译器参数不会始终正确传播。

    此更改也**回溯**到：0.9.8

    参考：[#3154](https://www.sqlalchemy.org/trac/ticket/3154)

+   **[sql] [bug]**

    修复了 0.9.7 中由[#3067](https://www.sqlalchemy.org/trac/ticket/3067)引起的回归问题，与一个错误命名的单元测试一起，使得所谓的“模式”类型如`Boolean`和`Enum`无法再被序列化。

    此更改也**回溯**到：0.9.8

    参考：[#3067](https://www.sqlalchemy.org/trac/ticket/3067)，[#3144](https://www.sqlalchemy.org/trac/ticket/3144)

+   **[sql] [bug]**

    修复了命名约定功能中的错误，其中使用包含`constraint_name`的检查约定将强制所有`Boolean`和`Enum`类型也需要名称，因为这些隐式创建约束，即使最终目标后端不需要生成约束，例如 PostgreSQL。这些特定约束的命名约定机制已经重新组织，使得命名确定在 DDL 编译时完成，而不是在约束/表构建时完成。

    此更改也**回溯**到：0.9.7

    参考：[#3067](https://www.sqlalchemy.org/trac/ticket/3067)

+   **[sql] [bug]**

    修复了通用表达式中的错误，其中当 CTE 以某种方式嵌套时，位置绑定参数可能以错误的最终顺序表示。

    此更改也**回溯**到：0.9.7

    参考：[#3090](https://www.sqlalchemy.org/trac/ticket/3090)

+   **[sql] [bug]**

    修复了多值`Insert`构造中的错误，当给定字面 SQL 表达式的第一个值后，将无法检查后续值条目。

    此更改也**回溯**到：0.9.7

    参考：[#3069](https://www.sqlalchemy.org/trac/ticket/3069)

+   **[sql] [bug]**

    为 Python 版本 < 2.6.5 的 dialect_kwargs 迭代添加了一个“str()”步骤，解决了“无 unicode 关键字参数”错误，因为这些参数在某些反射过程中作为关键字参数传递。

    此更改也**回溯**到：0.9.7

    参考：[#3123](https://www.sqlalchemy.org/trac/ticket/3123)

+   **[sql] [bug]**

    `TypeEngine.with_variant()`方法现在将接受一个类型类作为参数，该参数在内部转换为一个实例，使用其他构造（如`Column`）长期建立的相同约定。

    此更改也**回溯**到：0.9.7

    参考：[#3122](https://www.sqlalchemy.org/trac/ticket/3122)

+   **[sql] [bug]**

    当在表的显式`PrimaryKeyConstraint`中引用该`Column`时，`Column.nullable`标志会隐式设置为`False`。此行为现在与当`Column`本身的`Column.primary_key`标志设置为`True`时的行为相匹配，这意味着它们是完全等效的情况。

    此更改也**回溯**到：0.9.5

    参考：[#3023](https://www.sqlalchemy.org/trac/ticket/3023)

+   **[sql] [bug]**

    修复了在自定义`Comparator`实现中无法重写`Operators.__and__()`、`Operators.__or__()`和`Operators.__invert__()`运算符重载方法的错误。

    此更改也**回溯**到：0.9.5

    参考：[#3012](https://www.sqlalchemy.org/trac/ticket/3012)

+   **[sql] [bug]**

    修复了新的`DialectKWArgs.argument_for()`方法中的错误，其中为以前未包含任何特殊参数的构造添加参数将失败。

    此更改也**回溯**到：0.9.5

    参考：[#3024](https://www.sqlalchemy.org/trac/ticket/3024)

+   **[sql] [bug]**

    修复了在 0.9 版本中引入的回归，即新的“ORDER BY <labelname>”功能从[#1068](https://www.sqlalchemy.org/trac/ticket/1068)中不会将标签名称在 ORDER BY 中呈现时应用引用规则。

    此更改也**回溯**到：0.9.5

    参考：[#1068](https://www.sqlalchemy.org/trac/ticket/1068), [#3020](https://www.sqlalchemy.org/trac/ticket/3020)

+   **[sql] [bug]**

    恢复了`Function`的导入到`sqlalchemy.sql.expression`导入命名空间，该导入在 0.9 开始时被移除。

    此更改也已**回溯**至：0.9.5

+   **[sql] [bug]**

    `Insert.values()`的多值版本已修复，以更有用地与具有 Python 端默认值和/或函数以及服务器端默认值的表一起使用。该功能现在将与使用“位置”参数的方言一起工作；Python 可调用程序也将像“executemany”样式调用一样为每一行单独调用；服务器端默认列将不再隐式接收明确为第一行指定的值，而是拒绝在没有明确值的情况下调用。

    另请参阅

    Python-side defaults invoked for each row individually when using a multivalued insert

    参考：[#3288](https://www.sqlalchemy.org/trac/ticket/3288)

+   **[sql] [bug]**

    修复了`Table.tometadata()`方法中的错误，其中与`Boolean`或`Enum`类型对象关联的`CheckConstraint`会在目标表中重复。复制过程现在将此约束对象的生成跟踪为类型对象的本地对象。

    参考：[#3260](https://www.sqlalchemy.org/trac/ticket/3260)

+   **[sql] [bug]**

    `ForeignKeyConstraint.columns`集合的行为契约已经变得一致；这个属性现在像所有其他约束一样是一个`ColumnCollection`，并且在约束与`Table`关联时初始化。

    另请参阅

    ForeignKeyConstraint.columns 现在是 ColumnCollection

    参考：[#3243](https://www.sqlalchemy.org/trac/ticket/3243)

+   **[sql] [bug]**

    `Column.key`属性现在用作表达式内匿名绑定参数名称的源，以匹配将此值作为键渲染在 INSERT 或 UPDATE 语句中的现有用法。这允许`Column.key`被用作“替代”字符串，以解决不太适合作为绑定参数名称的难以翻译的列名。请注意，paramstyle 在任何情况下都可以在`create_engine()`上进行配置，并且今天大多数 DBAPI 都支持命名和位置风格。

    参考文献：[#3245](https://www.sqlalchemy.org/trac/ticket/3245)

+   **[sql] [bug]**

    修复了`PoolEvents.reset.dbapi_connection`参数在传递给此事件时的名称；特别是这会影响到此事件的“命名”参数风格的使用。感谢 Jason Goldberger 提供的拉取请求。

+   **[sql] [bug]**

    撤销了 0.9 版本中所做的更改，即“常量”`null()`、`true()`和`false()`的“单例”特性已被恢复。这些函数返回“单例”对象的效果是，不论词法使用如何，不同的实例都将被视为相同，特别是会影响到 SELECT 语句的 columns 子句的渲染。

    参见

    null()、false()和 true()常量不再是单例

    参考文献：[#3170](https://www.sqlalchemy.org/trac/ticket/3170)

+   **[sql] [bug] [engine]**

    修复了一个 bug，在“分支”连接（当您调用`Connection.connect()`时获得的类型）不会与父连接共享失效状态的情况。分支连接的架构稍作调整，以使分支连接对于所有失效状态和操作都延迟至父连接。

    参考文献：[#3215](https://www.sqlalchemy.org/trac/ticket/3215)

+   **[sql] [bug] [engine]**

    修复了一个 bug，在“分支”连接（当您调用`Connection.connect()`时获得的类型）不会与父连接共享事务状态的情况。分支连接的架构稍作调整，以使分支连接对于所有事务状态和操作都延迟至父连接。

    参考文献：[#3190](https://www.sqlalchemy.org/trac/ticket/3190)

+   **[sql] [bug]**

    使用`Insert.from_select()`现在隐含着对`insert()`的`inline=True`设置。这有助于修复一个 bug，即在支持的后端上，INSERT…FROM SELECT 结构将会意外地被编译为“隐式返回”，这会导致以下问题：在插入零行的情况下（因为隐式返回期望一行），会导致断裂；在插入多行的情况下（例如多行中的第一行），会导致任意返回数据。对于具有多个参数集的 INSERT..VALUES，也对此语句应用了类似的更改；对于此语句，隐式 RETURNING 也不再被发出。由于这两个构造处理可变数量的行，因此`ResultProxy.inserted_primary_key` 访问器不适用。以前，有一个文档说明注意，在某些数据库不支持返回并且因此不能执行“隐式”返回的情况下，可能更喜欢在 INSERT..FROM SELECT 中使用`inline=True`，但在任何情况下，INSERT…FROM SELECT 都不需要隐式返回。如果需要插入的数据，应该使用常规的显式`Insert.returning()` 来返回可变数量的结果行。

    参考：[#3169](https://www.sqlalchemy.org/trac/ticket/3169)

+   **[SQL] [增强]**

    自定义实现`GenericTypeCompiler`的方言现在可以这样构造，使得访问方法在接收到所属表达式对象的指示时也能被调用。在大多数情况下，任何接受关键字参数（例如 `**kw`）的访问方法都将接收到一个关键字参数 `type_expression`，指向包含该类型的表达式对象。对于 DDL 中的列，方言的编译器类可能需要修改其 `get_column_specification()` 方法以支持此功能。如果`UserDefinedType.get_col_spec()`在其参数签名中提供了`**kw`，那么它也会收到 `type_expression`。

    参考：[#3074](https://www.sqlalchemy.org/trac/ticket/3074)

### 模式

+   **[模式] [功能]**

    `MetaData.create_all()` 和 `MetaData.drop_all()` 的 DDL 生成系统已增强，大多数情况下可以自动处理相互依赖的外键约束；`ForeignKeyConstraint.use_alter` 标志的需求大大减少。该系统还适用于事先未命名的约束；仅在 DROP 的情况下，循环中涉及的约束之一需要名称。

    另请参阅

    ForeignKeyConstraint 上的 use_alter 标志（通常）不再需要

    参考：[#3282](https://www.sqlalchemy.org/trac/ticket/3282)

+   **[schema] [feature]**

    添加了一个新的访问器 `Table.foreign_key_constraints`，以补充 `Table.foreign_keys` 集合，以及 `ForeignKeyConstraint.referred_table`。

+   **[schema] [bug]**

    `CheckConstraint` 构造现在支持包含令牌 `%(column_0_name)s` 的命名约定；约束表达式将被扫描以获取列。此外，不包括 `%(constraint_name)s` 令牌的检查约束的命名约定现在也适用于 `SchemaType` 生成的约束，例如 `Boolean` 和 `Enum` 的约束；这在 0.9.7 中因 [#3067](https://www.sqlalchemy.org/trac/ticket/3067) 而停止工作。

    另请参阅

    命名 CHECK 约束

    为布尔值、枚举和其他模式类型配置命名

    参考：[#3067](https://www.sqlalchemy.org/trac/ticket/3067)，[#3299](https://www.sqlalchemy.org/trac/ticket/3299)

### postgresql

+   **[postgresql] [feature]**

    添加了对 PostgreSQL 索引的 `CONCURRENTLY` 关键字的支持，使用 `postgresql_concurrently` 建立。由 Iuri de Silvio 提交的拉取请求。

    另请参阅

    使用 CONCURRENTLY 的索引

    此更改也**已回溯**至：0.9.9

+   **[postgresql] [feature] [pg8000]**

    添加了对 pg8000 驱动程序的“合理多行计数”支持，这主要适用于在 ORM 中使用版本控制时。该功能基于使用的 pg8000 1.9.14 或更高版本进行版本检测。拉取请求由 Tony Locke 提供。

    此更改也**被回溯**至：0.9.8

+   **[postgresql] [feature]**

    向 `ColumnOperators.match()` 操作符添加了 kw 参数 `postgresql_regconfig`，允许指定“reg config”参数传递给发出的 `to_tsquery()` 函数。拉取请求由 Jonathan Vanasco 提供。

    此更改也**被回溯**至：0.9.7

    参考：[#3078](https://www.sqlalchemy.org/trac/ticket/3078)

+   **[postgresql] [feature]**

    通过 `JSONB` 增加了对 PostgreSQL JSONB 的支持。拉取请求由 Damian Dimmich 提供。

    此更改也**被回溯**至：0.9.7

+   **[postgresql] [feature]**

    当使用 pg8000 DBAPI 时，增加了对 AUTOCOMMIT 隔离级别的支持。拉取请求由 Tony Locke 提供。

    此更改也**被回溯**至：0.9.5

+   **[postgresql] [feature]**

    向 PostgreSQL `ARRAY` 类型添加了一个新标志 `ARRAY.zero_indexes`。当设置为 `True` 时，将在传递到数据库之前将所有数组索引值加一，以更好地在 Python 风格的零基索引和 PostgreSQL 以一为基的索引之间进行互操作。拉取请求由 Alexey Terentev 提供。

    此更改也**被回溯**至：0.9.5

    参考：[#2785](https://www.sqlalchemy.org/trac/ticket/2785)

+   **[postgresql] [feature]**

    PG8000 方言现在支持 `create_engine.encoding` 参数，通过设置连接上的客户端编码，然后被 pg8000 拦截。拉取请求由 Tony Locke 提供。

+   **[postgresql] [feature]**

    增加了对 PG8000 的原生 JSONB 功能的支持。拉取请求由 Tony Locke 提供。

+   **[postgresql] [feature] [pypy]**

    在 pypy 上增加了对 psycopg2cffi DBAPI 的支持。拉取请求由 shauns 提供。

    参见

    `sqlalchemy.dialects.postgresql.psycopg2cffi`

    参考：[#3052](https://www.sqlalchemy.org/trac/ticket/3052)

+   **[postgresql] [feature]**

    增加对 FILTER 关键字在聚合函数中的支持，由 PostgreSQL 9.4 支持。拉取请求由 Ilja Everilä 提供。

    参见

    PostgreSQL FILTER 关键字

+   **[postgresql] [feature]**

    已添加对物化视图和外部表的反射支持，以及对`Inspector.get_view_names()`中物化视图的支持，并在 PostgreSQL 版本的`Inspector`上提供了一个新方法`PGInspector.get_foreign_table_names()`。感谢 Rodrigo Menezes 的拉取请求。

    另请参见

    PostgreSQL 方言反映物化视图，外部表

    参考：[#2891](https://www.sqlalchemy.org/trac/ticket/2891)

+   **[postgresql] [feature]**

    在通过`Table`构造渲染 DDL 时，添加了对 PG 表选项 TABLESPACE, ON COMMIT, WITH(OUT) OIDS 和 INHERITS 的支持。感谢 malikdiarra 的拉取请求。

    另请参见

    PostgreSQL 表选项

    参考：[#2051](https://www.sqlalchemy.org/trac/ticket/2051)

+   **[postgresql] [feature]**

    添加了新方法`PGInspector.get_enums()`，在使用 PostgreSQL 检查器时将提供 ENUM 类型列表。感谢 Ilya Pekelny 的拉取请求���

+   **[postgresql] [bug]**

    向 PG `HSTORE` 类型添加了`hashable=False`标志，这是为了允许 ORM 在请求混合列/实体列表中的 ORM 映射的 HSTORE 列时跳过“哈希”操作。补丁由 Gunnlaugur Þór Briem 提供。

    这个更改也被**回溯**到：0.9.5, 0.8.7

    参考：[#3053](https://www.sqlalchemy.org/trac/ticket/3053)

+   **[postgresql] [bug]**

    添加了一个新的“断开连接”消息“连接意外关闭”。这似乎与较新版本的 SSL 有关。感谢 Antti Haapala 的拉取请求。

    这个更改也被**回溯**到：0.9.5, 0.8.7

+   **[postgresql] [bug]**

    修复了在使用 psycopg2 时与 ARRAY 类型一起支持 PostgreSQL UUID 类型的问题。psycopg2 方言现在使用 psycopg2.extras.register_uuid()钩子，以便始终将 UUID 值作为 UUID()对象传递到/从 DBAPI。`UUID.as_uuid`标志仍然受到尊重，但是对于 psycopg2，当禁用此标志时，我们需要将返回的 UUID 对象转换回字符串。

    这个更改也被**回溯**到：0.9.9

    参考：[#2940](https://www.sqlalchemy.org/trac/ticket/2940)

+   **[postgresql] [bug]**

    在使用 psycopg2 2.5.4 或更高版本时，添加了对`postgresql.JSONB`数据类型的支持，该版本具有 JSONB 数据的本机转换，因此必须禁用 SQLAlchemy 的转换器；此外，还使用了新添加的 psycopg2 扩展`extras.register_default_jsonb`来建立通过`json_deserializer`参数传递给方言的 JSON 反序列化器。还修复了实际上未循环传递 JSONB 类型而不是 JSON 类型的 PostgreSQL 集成测试。拉取请求由 Mateusz Susik 提供。

    此更改也被**回溯**到：0.9.9

+   **[postgresql] [bug]**

    修复了在使用旧版本的 psycopg2 < 2.4.3 注册 HSTORE 类型时使用“array_oid”标志的问题，该版本不支持此标志，以及在 psycopg2 版本 < 2.5 上使用本地 json 序列化器钩子“register_default_json”与用户定义的`json_deserializer`时的问题，该版本不包括本地 json。

    此更改也被**回溯**到：0.9.9

+   **[postgresql] [bug]**

    修复了 PostgreSQL 方言无法在`Index`中呈现与表绑定列不直接对应的表达式的错误；通常当`text()`构造是索引中的表达式之一时；或者如果其中一个或多个是这样的表达式，则可能会误解表达式列表。

    此更改也被**回溯**到：0.9.9

    参考：[#3174](https://www.sqlalchemy.org/trac/ticket/3174)

+   **[postgresql] [bug]**

    重新审视了首次在 0.9.5 中修补的问题，显然 psycopg2 的`.closed`访问器并不像我们假设的那样可靠，因此我们已经添加了对异常消息“SSL SYSCALL error: Bad file descriptor”和“SSL SYSCALL error: EOF detected”进行显式检查以检测断开连接的情况。我们将继续将 psycopg2 的 connection.closed 作为首要检查。

    此更改也被**回溯**到：0.9.8

    参考：[#3021](https://www.sqlalchemy.org/trac/ticket/3021)

+   **[postgresql] [bug]**

    修复了 PostgreSQL JSON 类型无法持久化或以其他方式呈现 SQL NULL 列值而不是 JSON 编码的`'null'`的错误。为支持此情况，更改如下：

    +   现在可以指定值`null()`，这将始终导致语句中的 NULL 值。

    +   新增了一个参数`JSON.none_as_null`，当为 True 时表示 Python 的`None`值应该被持久化为 SQL NULL，而不是 JSON 编码的`'null'`。

    还为除 psycopg2 之外的其他 DBAPI 修复了将 NULL 检索为 None 的问题，即 pg8000。

    此更改也被**回溯**到：0.9.8

    参考：[#3159](https://www.sqlalchemy.org/trac/ticket/3159)

+   **[postgresql] [bug]**

    用于 DBAPI 错误的异常包装系统现在可以适应非标准的 DBAPI 异常，例如 psycopg2 的 TransactionRollbackError。这些异常现在将使用`sqlalchemy.exc`中最接近的可用子类引发，对于 TransactionRollbackError，是`sqlalchemy.exc.OperationalError`。

    此更改也被**回溯**到：0.9.8

    参考：[#3075](https://www.sqlalchemy.org/trac/ticket/3075)

+   **[postgresql] [bug]**

    修复了`array`对象中的错误，其中与普通的 Python 列表进行比较会导致使用不正确的数组构造函数。感谢 Andrew 的拉取请求。

    此更改也被**回溯**到：0.9.8

    参考：[#3141](https://www.sqlalchemy.org/trac/ticket/3141)

+   **[postgresql] [bug]**

    为函数添加了支持的`FunctionElement.alias()`方法，例如`func`构造。先前，此方法的行为是未定义的。当前行为模仿了 0.9.4 之前的行为，即将函数转换为具有给定别名的单列 FROM 子句，其中列本身是匿名命名的。

    此更改也被**回溯**到：0.9.8

    参考：[#3137](https://www.sqlalchemy.org/trac/ticket/3137)

+   **[postgresql] [bug] [pg8000]**

    通过新的 pg8000 隔离级别功能引入的 0.9.5 中的错误已被修复，其中引擎级别的隔离级别参数在连接时会引发错误。

    此更改也被**回溯**到：0.9.7

    参考：[#3134](https://www.sqlalchemy.org/trac/ticket/3134)

+   **[postgresql] [bug]**

    现在在确定异常是否为“断开连接”错误时，将查询 psycopg2 的`.closed`访问器；理想情况下，这应该消除对异常消息的任何其他检查来检测断开连接的需要，但我们将保留这些现有消息作为备用。这应该能够处理新的情况，如“SSL EOF”条件。感谢 Dirk Mueller 的拉取请求。

    此更改也被**回溯**到：0.9.5

    参考：[#3021](https://www.sqlalchemy.org/trac/ticket/3021)

+   **[postgresql] [bug]**

    当调用普通 `table.drop()` 时，PostgreSQL `ENUM` 类型将发出 DROP TYPE 指令，假设对象未直接与 `MetaData` 对象关联。为了适应多个表之间共享的枚举类型的用例，该类型应直接与 `MetaData` 对象关联；在这种情况下，该类型将仅在元数据级别创建，或者如果直接创建。一般来说，已经对 PostgreSQL 枚举类型的创建/删除规则进行了高度改写。

    另请参阅

    ENUM 类型创建/删除规则的彻底改革

    参考：[#3319](https://www.sqlalchemy.org/trac/ticket/3319)

+   **[postgresql] [bug]**

    `PGDialect.has_table()` 方法现在将查询 `pg_catalog.pg_table_is_visible(c.oid)`，而不是在模式名称为 None 时测试精确的模式匹配；这样做是为了使该方法还能显示临时表的存在。请注意，这是一种行为变更，因为 PostgreSQL 允许非临时表静默地覆盖同名的现有临时表，所以这改变了 `checkfirst` 在这种不寻常情况下的行为。

    另请参阅

    PostgreSQL has_table() 现在适用于临时表

    参考：[#3264](https://www.sqlalchemy.org/trac/ticket/3264)

+   **[postgresql] [enhancement]**

    为 PostgreSQL 方言添加了新类型 `OID`。虽然“oid”通常是 PG 中的一个私有类型，在现代版本中不会公开，但在某些 PG 使用场景中，如大型对象支持，这些类型可能会被公开，以及在一些用户报告的模式反射使用场景中。

    此更改也被 **backported** 至：0.9.5

    参考：[#3002](https://www.sqlalchemy.org/trac/ticket/3002)

### mysql

+   **[mysql] [feature]**

    MySQL 方言现在在所有情况下都使用 NULL / NOT NULL 渲染 TIMESTAMP，因此启用了带有 `explicit_defaults_for_timestamp` 标志的 MySQL 5.6.6 将允许 TIMESTAMP 在 `nullable=False` 时继续正常工作。现有应用程序不受影响，因为 SQLAlchemy 总是对于 `nullable=True` 的 TIMESTAMP 列发出 NULL。

    另请参阅

    MySQL TIMESTAMP 类型现在在所有情况下渲染 NULL / NOT NULL

    TIMESTAMP 列和 NULL

    参考：[#3155](https://www.sqlalchemy.org/trac/ticket/3155)

+   **[mysql] [feature]**

    将“supports_unicode_statements”标志更新为 True，适用于 Python 2 下的 MySQLdb 和 Pymysql。这指的是 SQL 语句本身，而不是参数，影响使用非 ASCII 字符的表和列名等问题。这两个驱动程序在现代版本中似乎都支持 Python 2 Unicode 对象而没有问题。

    参考：[#3121](https://www.sqlalchemy.org/trac/ticket/3121)

+   **[mysql] [change]**

    `gaerdbms` 方言不再必要，并发出弃用警告。Google 现在建议直接使用 MySQLdb 方言。

    此更改也**回溯**到：0.9.9

    参考：[#3275](https://www.sqlalchemy.org/trac/ticket/3275)

+   **[mysql] [bug]**

    MySQL 错误 2014“命令不同步”似乎在现代 MySQL-Python 版本中引发 ProgrammingError 而不是 OperationalError；现在在 OperationalError 和 ProgrammingError 中都检查了所有测试“is disconnect”的 MySQL 错误代码。

    此更改也**回溯**到：0.9.7, 0.8.7

    参考：[#3101](https://www.sqlalchemy.org/trac/ticket/3101)

+   **[mysql] [bug]**

    修复了在索引的`mysql_length`参数上添加列名时需要具有相同引号才能被识别的错误。修复使引号变为可选，但也为那些使用此解决方法的人提供了旧的行为以实现向后兼容。

    此更改也**回溯**到：0.9.5, 0.8.7

    参考：[#3085](https://www.sqlalchemy.org/trac/ticket/3085)

+   **[mysql] [bug]**

    添加了支持通过等号包含 KEY_BLOCK_SIZE 的索引来反射表的功能。拉取请求由 Sean McGivern 提供。

    此更改也**回溯**到：0.9.5, 0.8.7

+   **[mysql] [bug]**

    在 MySQLdb 方言周围添加了一个版本检查，用于检查‘utf8_bin’校对，因为这在 MySQL 服务器<5.0 上失败。

    此更改也**回溯**到：0.9.9

    参考：[#3274](https://www.sqlalchemy.org/trac/ticket/3274)

+   **[mysql] [bug] [mysqlconnector]**

    从版本 2.0 开始，Mysqlconnector 可能作为 Python 3 合并的副作用，现在不再期望百分号（例如用作模运算符等）被加倍，即使使用“pyformat”绑定参数格式（Mysqlconnector 未记录此更改）。方言现在在检测模运算符应该呈现为`%%`还是`%`时检查 py2k 和 mysqlconnector 小于版本 2.0。

    此更改也**回溯**到：0.9.8

+   **[mysql] [bug] [mysqlconnector]**

    Unicode SQL 现在传递给 MySQLconnector 版本 2.0 及以上；对于 Py2k 和 MySQL < 2.0，字符串被编码。

    此更改也**回溯**到：0.9.8

+   **[mysql] [bug]**

    MySQL 方言现在支持在构造为`TypeDecorator`对象的类型上进行 CAST。

+   **[mysql] [bug]**

    当在 MySQL 方言上使用`cast()`时，如果 MySQL 不支持 CAST 的类型，则会发出警告；MySQL 仅支持对部分数据类型进行 CAST。长期以来，SQLAlchemy 在 MySQL 的情况下对不支持的类型忽略了 CAST。虽然我们现在不想改变这一点，但我们会发出警告以显示已经发生的情况。当使用旧版本的 MySQL（< 4）不支持 CAST 时，也会发出警告，在这种情况下也会被跳过。

    参考：[#3237](https://www.sqlalchemy.org/trac/ticket/3237)

+   **[mysql] [bug]**

    `SET`类型已经进行了改进，不再假定空字符串或具有单个空字符串值的集合实际上是具有单个空字符串的集合；相反，默认情况下将其视为空集。为了处理实际希望将空值`''`包含为合法值的`SET`的持久性，添加了一种新的按位操作模式，通过`SET.retrieve_as_bitwise`标志启用，将使用其位标志位置持久化和检索值。还修复了对于未原生转换 Unicode 的驱动程序配置的 Unicode 值的存储和检索。

    另请参阅

    MySQL SET 类型进行了改进，支持空集，Unicode，空值处理

    参考：[#3283](https://www.sqlalchemy.org/trac/ticket/3283)

+   **[mysql] [bug]**

    `ColumnOperators.match()`操作现在处理方式已更改，不再严格假定返回类型为布尔值；现在返回一个名为`MatchType`的`Boolean`子类。当在 Python 表达式中使用时，该类型仍会产生布尔行为，但方言可以在结果时间覆盖其行为。在 MySQL 的情况下，虽然 MATCH 操作通常在表达式中的布尔上下文中使用，但如果实际查询匹配表达式的值，则会返回一个浮点值；此值与 SQLAlchemy 的基于 C 的布尔处理器不兼容，因此 MySQL 的结果集行为现在遐照`Float`类型。还添加了一个新的操作对象`notmatch_op`，以更好地允许方言定义匹配操作的否定。

    另请参阅

    match()运算符现在返回与 MySQL 的浮点返回值兼容的 MatchType

    参考：[#3263](https://www.sqlalchemy.org/trac/ticket/3263)

+   **[mysql] [bug]**

    MySQL 布尔符号“true”、“false”再次有效。0.9 中的更改[#2682](https://www.sqlalchemy.org/trac/ticket/2682)禁止了 MySQL 方言在“IS”/“IS NOT”上下文中使用“true”和“false”符号，但 MySQL 支持此语法，即使它没有布尔类型。MySQL 仍然是“非本地布尔”，但`true()`和`false()`符号再次生成关键字“true”和“false”，因此像`column.is_(true())`这样的表达式在 MySQL 上再次有效。

    另请参阅

    MySQL 布尔符号“true”、“false”再次有效

    参考：[#3186](https://www.sqlalchemy.org/trac/ticket/3186)

+   **[mysql] [bug]**

    MySQL 方言现在将禁用`ConnectionEvents.handle_error()`事件，用于检测表是否存在的内部语句不会触发该事件。这是通过使用执行选项`skip_user_error_events`来实现的，该选项在该执行范围内禁用处理错误事件。这样，重写异常的用户代码不需要担心 MySQL 方言或其他偶尔需要捕获 SQLAlchemy 特定异常的方言。

+   **[mysql] [bug]**

    将“raise_on_warnings”的默认值更改为 False，以用于 MySQLconnector。由于某种原因，此值设置为 True。不幸的是，“buffered”标志必须保持为 True，因为 MySQLconnector 不允许关闭游标，除非所有结果都完全获取。

    参考：[#2515](https://www.sqlalchemy.org/trac/ticket/2515)

### sqlite

+   **[sqlite] [feature]**

    增加了对 SQLite 上部分索引（例如带有 WHERE 子句）的支持。感谢 Kai Groner 的拉取请求。

    另请参阅

    部分索引

    此更改也**回溯**到：0.9.9

+   **[sqlite] [feature]**

    为 SQLCipher 后端添加了一个新的 SQLite 后端。该后端使用 pysqlcipher Python 驱动程序提供加密的 SQLite 数据库，该驱动程序与 pysqlite 驱动程序非常相似。

    另请参阅

    `pysqlcipher`

    此更改也**回溯**到：0.9.9

+   **[sqlite] [bug]**

    当从附加的数据库文件使用 UNION 进行选择时，pysqlite 驱动程序将列名在 cursor.description 中报告为 'dbname.tablename.colname'，而不是通常对于 UNION 的 'tablename.colname'（请注意，对于两者都应该只是 'colname'，但我们对此进行了处理）。此处的列翻译逻辑已经调整为检索最右边的标记，而不是第二个标记，因此在两种情况下都有效。解决方法由 Tony Roberts 提供。

    此更改也被**回溯**至：0.9.8

    参考：[#3211](https://www.sqlalchemy.org/trac/ticket/3211)

+   **[sqlite] [bug]**

    修复了 SQLite 加入重写问题，其中作为标量子查询嵌入的子查询（例如在 IN 中）会从包含查询中接收不适当的替换，如果相同的表在子查询中存在并且在包含查询中也存在，例如在连接继承场景中。

    此更改也被**回溯**至：0.9.7

    参考：[#3130](https://www.sqlalchemy.org/trac/ticket/3130)

+   **[sqlite] [bug]**

    现在在 SQLite 上完全反映了 UNIQUE 和 FOREIGN KEY 约束，无论是否有名称。以前，外键名称被忽略，未命名的唯一约束被跳过。感谢 Jon Nelson 协助解决此问题。

    参考：[#3244](https://www.sqlalchemy.org/trac/ticket/3244), [#3261](https://www.sqlalchemy.org/trac/ticket/3261)

+   **[sqlite] [bug]**

    当使用 `DATE`、`TIME` 或 `DATETIME` 类型的 SQLite 方言，并给定一个只呈现数字的 `storage_format`，将在 DDL 中将类型呈现为 `DATE_CHAR`、`TIME_CHAR` 和 `DATETIME_CHAR`，以便尽管值中缺少字母字符，列仍会提供“文本亲和性”。通常情况下，这是不需要的，因为默认存储格式中的文本值已经暗示了文本。

    另请参阅

    日期和时间类型

    参考：[#3257](https://www.sqlalchemy.org/trac/ticket/3257)

+   **[sqlite] [bug]**

    SQLite 现在支持从临时表反射唯一约束；以前，这将导致 TypeError。拉取请求由 Johannes Erdfelt 提供。

    另请参阅

    SQLite/Oracle 有不同的临时表/视图名称报告方法 - 关于 SQLite 临时表和视图反射的更改。

    参考：[#3203](https://www.sqlalchemy.org/trac/ticket/3203)

+   **[sqlite] [bug]**

    添加了`Inspector.get_temp_table_names()`和`Inspector.get_temp_view_names()`；目前，只有 SQLite 和 Oracle 方言支持这些方法。临时表和视图名称的返回已从 SQLite 和 Oracle 版本的`Inspector.get_table_names()`和`Inspector.get_view_names()`中**移除**；其他数据库后端不支持此信息（如 MySQL），操作范围也不同，因为表可以是会话本地的，通常不支持远程模式中的表。

    另请参阅

    SQLite/Oracle 有不同的方法用于临时表/视图名称报告

    参考：[#3204](https://www.sqlalchemy.org/trac/ticket/3204)

### mssql

+   **[mssql] [功能]**

    为 SQL Server 2008 启用了“多值插入”。感谢 Albert Cervin 提供的拉取请求。还扩展了“IDENTITY INSERT”模式的检查，以包括当标识键出现在语句的 VALUEs 子句中时。

    这一变更也被**回溯**到：0.9.7

+   **[mssql] [功能]**

    SQL Server 2012 现在推荐对于大文本/二进制类型使用 VARCHAR(max)、NVARCHAR(max)、VARBINARY(max)。MSSQL 方言现在会根据版本检测以及新的`deprecate_large_types`标志来尊重这一点。

    另请参阅

    大文本/二进制类型弃用

    参考：[#3039](https://www.sqlalchemy.org/trac/ticket/3039)

+   **[mssql] [更改]**

    当使用 pyodbc 时，基于主机名的 SQL Server 连接格式将不再指定默认的“驱动程序名称”，如果缺少此项将会发出警告。SQL Server 的最佳驱动程序名称经常变化且因平台而异，因此基于主机名的连接需要指定此项。优先使用 DSN-based 连接。

    另请参阅

    PyODBC 驱动程序名称在基于主机名的 SQL Server 连接中是必需的

    参考：[#3182](https://www.sqlalchemy.org/trac/ticket/3182)

+   **[mssql] [错误]**

    在“SET IDENTITY_INSERT”语句中添加了语句编码，当在 IDENTITY 列中插入显式 INSERT 时，以支持在不支持 Unicode 语句的驱动程序（如 pyodbc + unix + py2k）上使用非 ASCII 表标识符。

    这一变更也被**回溯**到：0.9.7, 0.8.7

+   **[mssql] [错误]**

    在 SQL Server pyodbc 方言中，修复了 `description_encoding` 方言参数的实现，当未明确设置时，会导致在包含其他编码名称的结果集中，无法正确解析 cursor.description。未来不应该需要此参数。

    此更改也被 **回溯** 到：0.9.7, 0.8.7

    参考：[#3091](https://www.sqlalchemy.org/trac/ticket/3091)

+   **[mssql] [bug]**

    修复了 pymssql 方言中版本字符串检测与 Microsoft SQL Azure 配合使用的问题，将 “SQL Server” 更改为 “SQL Azure”。

    此更改也被 **回溯** 到：0.9.8

    参考：[#3151](https://www.sqlalchemy.org/trac/ticket/3151)

+   **[mssql] [bug]**

    修订了用于确定当前默认模式名称的查询，使用 `database_principal_id()` 函数与 `sys.database_principals` 视图结合使用，以便我们可以独立于正在进行的登录类型（例如 SQL Server，Windows 等）确定默认模式。

    此更改也被 **回溯** 到：0.9.5

    参考：[#3025](https://www.sqlalchemy.org/trac/ticket/3025)

### oracle

+   **[oracle] [feature]**

    增加了对 cx_oracle 连接到特定服务名称的支持，而不是 tns 名称，通过在 URL 中传递 `?service_name=<name>`。Pull request 由 Sławomir Ehlert 提供。

+   **[oracle] [feature]**

    新的 Oracle DDL 功能用于表和索引：COMPRESS，BITMAP。Patch 由 Gabor Gombas 提供。

+   **[oracle] [feature]**

    增加了对 Oracle 下 CTE 的支持。这包括对别名语法的一些调整，以及一个新的 CTE 功能 `CTE.suffix_with()`，用于向 CTE 添加特殊的 Oracle 特定指令。

    另请参阅

    改进了 Oracle 中 CTE 的支持

    参考：[#3220](https://www.sqlalchemy.org/trac/ticket/3220)

+   **[oracle] [feature]**

    增加了对 Oracle 表选项 ON COMMIT 的支持。

+   **[oracle] [bug]**

    修复了 Oracle 方言中长期存在的 bug，即以数字开头的绑定参数名称不会被引用，因为 Oracle 不喜欢绑定参数名称中有数字。

    此更改也被 **回溯** 到：0.9.8

    参考：[#2138](https://www.sqlalchemy.org/trac/ticket/2138)

+   **[oracle] [bug] [tests]**

    修复了 Oracle 方言测试套件中的一个 bug，在一个测试中，假定 ‘username’ 在数据库 URL 中，即使这可能不是情况。

    此更改也被 **回溯** 到：0.9.7

    参考：[#3128](https://www.sqlalchemy.org/trac/ticket/3128)

+   **[oracle] [bug]**

    在 `Select.with_hint()` 方法中，当使用 `%(name)s` token 引用别名时，别名将被正确引用。之前，Oracle 后端尚未实现此引用。

### 测试

+   **[tests] [bug]**

    修复了一个 bug，即 “python setup.py test” 没有适当调用 distutils，测试套件结束时会发出错误。

    此更改还**反向移植**到：0.9.7

+   **[tests] [bug] [py3k]**

    修正了一些关于`imp`模块和 Python 3.3 或更高版本在运行测试时的弃用警告。感谢 Matt Chisholm 提供的拉取请求。

    此更改还**反向移植**到：0.9.5

    参考：[#2830](https://www.sqlalchemy.org/trac/ticket/2830)

### 杂项

+   **[feature] [ext]**

    添加了一个新的扩展套件`sqlalchemy.ext.baked`。这个简单但不寻常的系统可以大大节省 Python 在构建和处理 orm `Query`对象时的开销，从查询构建到渲染字符串 SQL 语句。

    另请参阅

    烘焙查询

    参考：[#3054](https://www.sqlalchemy.org/trac/ticket/3054)

+   **[feature] [ext]**

    `sqlalchemy.ext.automap`扩展现在会自动在检测到包含一个或多个非空列的外键的一对多关系/反向引用上设置`cascade="all, delete-orphan"`。在这种情况下，此参数存在于传递给`generate_relationship()`的关键字中，并仍然可以被覆盖。此外，如果`ForeignKeyConstraint`为非空列指定了`ondelete="CASCADE"`或为可空列指定了`ondelete="SET NULL"`，则还将在关系中添加参数`passive_deletes=True`。请注意，��非所有后端都支持 ondelete 的反射，但支持反射的后端包括 PostgreSQL 和 MySQL。

    参考：[#3210](https://www.sqlalchemy.org/trac/ticket/3210)

+   **[bug] [declarative]**

    当访问`__mapper_args__`字典时，它是从声明性 mixin 或抽象类中复制的，因此声明性本身对此字典所做的修改不会与其他映射冲突。该字典在`version_id_col`和`polymorphic_on`参数方面进行修改，用本地类/表正式映射到的列替换其中的列。

    此更改还**反向移植**到：0.9.5, 0.8.7

    参考：[#3062](https://www.sqlalchemy.org/trac/ticket/3062)

+   **[bug] [ext]**

    修复了可变扩展中的错误，其中`MutableDict`未报告`setdefault()`字典操作的更改事件。

    此更改还**反向移植**到：0.9.5, 0.8.7

    参考：[#3051](https://www.sqlalchemy.org/trac/ticket/3051), [#3093](https://www.sqlalchemy.org/trac/ticket/3093)

+   **[bug] [ext]**

    修复了 `MutableDict.setdefault()` 未能返回现有值或新值的 bug（此 bug 未在任何 0.8 版本中发布）。感谢 Thomas Hervé 提交的拉取请求。

    此更改也被 **回溯** 至：0.9.5，0.8.7

    参考：[#3051](https://www.sqlalchemy.org/trac/ticket/3051)，[#3093](https://www.sqlalchemy.org/trac/ticket/3093)

+   **[bug] [ext] [py3k]**

    修复了在 Py3K 下协会代理列表类无法正确解释切片的 bug。感谢 Gilles Dartiguelongue 提交的拉取请求。

    此更改也被 **回溯** 至：0.9.9

+   **[bug] [declarative]**

    修复了一种在某些奇特的最终用户设置中观察到的不太可能的竞争条件，在这种情况下，在声明时尝试检查“重复类名”的尝试会遇到一种不完全清理的弱引用，与其他被移除的类相关联；此处的检查现在确保弱引用在进一步调用之前仍然引用一个对象。

    此更改也被 **回溯** 至：0.9.8

    参考：[#3208](https://www.sqlalchemy.org/trac/ticket/3208)

+   **[bug] [ext]**

    修复了在集合替换事件期间项的顺序会被打乱的排序列表中的 bug，如果 `reorder_on_append` 标志设置为 True，则修复将确保排序列表仅影响显式与对象关联的列表。

    此更改也被 **回溯** 至：0.9.8

    参考：[#3191](https://www.sqlalchemy.org/trac/ticket/3191)

+   **[bug] [ext]**

    修复了 `MutableDict` 未能实现 `update()` 字典方法的 bug，因此未能捕获更改。感谢 Matt Chisholm 提交的拉取请求。

    此更改也被 **回溯** 至：0.9.8

+   **[bug] [ext]**

    修复了自定义子类 `MutableDict` 不会在“强制”操作中显示，并且会返回一个普通的 `MutableDict` 的 bug。感谢 Matt Chisholm 提交的拉取请求。

    此更改也被 **回溯** 至：0.9.8

+   **[bug] [pool]**

    修复了连接池日志记录中的 bug，在该 bug 中，“连接已检出”调试日志记录消息如果使用 `logging.setLevel()` 设置日志记录，而不是使用 `echo_pool` 标志，则不会发出。已添加测试以断言此日志记录。这是在 0.9.0 中引入的回归。

    此更改也被 **回溯** 至：0.9.8

    参考：[#3168](https://www.sqlalchemy.org/trac/ticket/3168)

+   **[bug] [declarative]**

    修复了在声明 `__abstract__` 标志未被区分为实际上是值 `False` 时的 bug。`__abstract__` 标志需要在被测试的级别上实际上计算为 True 值。

    这个更改也被**回溯**到：0.9.7

    参考：[#3097](https://www.sqlalchemy.org/trac/ticket/3097)

+   **[bug] [测试套件]**

    在公共测试套件中，从不太受支持的`Text`更改为使用`String(40)`在`StringTest.test_literal_backslashes`中。感谢 Jan 提交的 Pullreq。

    这个更改也被**回溯**到：0.9.5

+   **[已移除]**

    Drizzle 方言已从核心中移除；它现在作为[sqlalchemy-drizzle](https://bitbucket.org/zzzeek/sqlalchemy-drizzle)提供，这是一个独立的第三方方言。该方言仍然几乎完全基于 SQLAlchemy 中存在的 MySQL 方言。

    另请参阅

    Drizzle 方言现在是外部方言

## 1.0.19

发布日期：2017 年 8 月 3 日

### oracle

+   **[oracle] [性能] [bug] [py2k]**

    修复了由于修复[#3937](https://www.sqlalchemy.org/trac/ticket/3937)引起的性能回归问题，其中 cx_Oracle 版本 5.3 删除了其命名空间中的`.UNICODE`符号，这被解释为 cx_Oracle 的“WITH_UNICODE”模式被无条件地打开，从而在 SQLAlchemy 一侧调用函数，无条件地将所有字符串转换为 unicode 并导致性能影响。实际上，根据 cx_Oracle 的作者，自 5.1 版本起，“WITH_UNICODE”模式已完全移除，因此昂贵的 unicode 转换函数不再必要，如果在 Python 2 下检测到 cx_Oracle 5.1 或更高版本，则会禁用这些函数。已恢复在[#3937](https://www.sqlalchemy.org/trac/ticket/3937)中删除的“WITH_UNICODE”模式警告。

    参考：[#4035](https://www.sqlalchemy.org/trac/ticket/4035)

### oracle

+   **[oracle] [性能] [bug] [py2k]**

    修复了由于修复[#3937](https://www.sqlalchemy.org/trac/ticket/3937)引起的性能回归问题，其中 cx_Oracle 版本 5.3 删除了其命名空间中的`.UNICODE`符号，这被解释为 cx_Oracle 的“WITH_UNICODE”模式被无条件地打开，从而在 SQLAlchemy 一侧调用函数，无条件地将所有字符串转换为 unicode 并导致性能影响。实际上，根据 cx_Oracle 的作者，自 5.1 版本起，“WITH_UNICODE”模式已完全移除，因此昂贵的 unicode 转换函数不再必要，如果在 Python 2 下检测到 cx_Oracle 5.1 或更高版本，则会禁用这些函数。已恢复在[#3937](https://www.sqlalchemy.org/trac/ticket/3937)中删除的“WITH_UNICODE”模式警告。

    参考：[#4035](https://www.sqlalchemy.org/trac/ticket/4035)

## 1.0.18

发布日期：2017 年 7 月 24 日

### oracle

+   **[oracle] [bug]**

    修复了由于 cx_Oracle 5.3 现在似乎在构建中硬编码此标志而暴露出来的 cx_Oracle 的 WITH_UNICODE 模式问题；使用此模式的内部方法未使用正确的签名。

    参考：[#3937](https://www.sqlalchemy.org/trac/ticket/3937)

### 测试

+   **[测试] [bug] [py3k]**

    修复了与 Python 3.6.2 中关于上下文管理器的更改不兼容的测试固件中的问题。

    参考：[#4034](https://www.sqlalchemy.org/trac/ticket/4034)

### oracle

+   **[oracle] [bug]**

    修复了 cx_Oracle 的 WITH_UNICODE 模式，这是由于 cx_Oracle 5.3 现在似乎在构建中硬编码了此标志；一个使用此模式的内部方法未使用正确的签名。

    参考：[#3937](https://www.sqlalchemy.org/trac/ticket/3937)

### tests

+   **[tests] [bug] [py3k]**

    修复了与 Python 3.6.2 中关于上下文管理器的更改不兼容的测试固件中的问题。

    参考：[#4034](https://www.sqlalchemy.org/trac/ticket/4034)

## 1.0.17

发布日期：2017 年 1 月 17 日

### orm

+   **[orm] [bug]**

    修复了在多个实体上进行连接式急加载时涉及多态继承时会抛出“‘NoneType’ object has no attribute ‘isa’”错误的 bug。此问题是由于修复[#3611](https://www.sqlalchemy.org/trac/ticket/3611)引入的。

    参考：[#3884](https://www.sqlalchemy.org/trac/ticket/3884)

### misc

+   **[bug] [py3k]**

    修复了与未使用‘r’修饰符的转义字符串相关的 Python 3.6 DeprecationWarnings，并为 Python 3.6 添加了测试覆盖率。

    参考：[#3886](https://www.sqlalchemy.org/trac/ticket/3886)

### orm

+   **[orm] [bug]**

    修复了在多个实体上进行连接式急加载时涉及多态继承时会抛出“‘NoneType’ object has no attribute ‘isa’”错误的 bug。此问题是由于修复[#3611](https://www.sqlalchemy.org/trac/ticket/3611)引入的。

    参考：[#3884](https://www.sqlalchemy.org/trac/ticket/3884)

### misc

+   **[bug] [py3k]**

    修复了与未使用‘r’修饰符的转义字符串相关的 Python 3.6 DeprecationWarnings，并为 Python 3.6 添加了测试覆盖率。

    参考：[#3886](https://www.sqlalchemy.org/trac/ticket/3886)

## 1.0.16

发布日期：2016 年 11 月 15 日

### orm

+   **[orm] [bug]**

    修复了 `Session.bulk_update_mappings()` 中的 bug，其中备用命名的主键属性无法正确跟踪到 UPDATE 语句中。

    参考：[#3849](https://www.sqlalchemy.org/trac/ticket/3849)

+   **[orm] [bug]**

    修复了连接式急加载在多态加载映射器失败的 bug，其中 polymorphic_on 设置为未映射表达式（如 CASE 表达式）。

    参考：[#3800](https://www.sqlalchemy.org/trac/ticket/3800)

+   **[orm] [bug]**

    修复了当通过 `Session.bind_mapper()`、`Session.bind_table()` 或构造函数发送到 Session 的��效绑定时引发的 ArgumentError 未能正确引发的问题。

    参考：[#3798](https://www.sqlalchemy.org/trac/ticket/3798)

+   **[orm] [bug]**

    修复了`Session.bulk_save()`中的一个 BUG，其中一个实现了版本 id 计数器的映射与 UPDATE 结合使用时功能不正常。

    参考：[#3781](https://www.sqlalchemy.org/trac/ticket/3781)

+   **[orm] [bug]**

    修复了当映射器属性或其他 ORM 结构在首次调用这些访问器之后被添加到映射器/类后，`Mapper.attrs`、`Mapper.all_orm_descriptors` 和其他派生属性无法刷新的 BUG。

    参考：[#3778](https://www.sqlalchemy.org/trac/ticket/3778)

### mssql

+   **[mssql] [bug]**

    将用于获取“默认模式名称”的查询更改为使用“schema_name()”函数，而不是查询数据库主体表，因为已经报告了前者在 Azure 数据仓库版上不可用的问题。希望这样能够在所有 SQL Server 版本和认证样式中最终都能正常工作。

    参考：[#3810](https://www.sqlalchemy.org/trac/ticket/3810)

+   **[mssql] [bug]**

    将 pyodbc 的服务器版本信息方案更新为使用 SQL Server 的 SERVERPROPERTY()，而不是依赖于 pyodbc.SQL_DBMS_VER，后者特别是在使用 FreeTDS 时仍然不可靠。

    参考：[#3814](https://www.sqlalchemy.org/trac/ticket/3814)

+   **[mssql] [bug]**

    将错误代码 20017 “服务器意外 EOF” 添加到导致连接池重置的断开异常列表中。感谢 Ken Robbins 的拉取请求。

    参考：[#3791](https://www.sqlalchemy.org/trac/ticket/3791)

+   **[mssql] [bug]**

    修复了 pyodbc 方言（以及几乎不工作的 adodbapi 方言）中的一个 BUG，即密码或用户名字段中存在分号时，分号可能被解释为另一个令牌的分隔符；现在在存在分号时值将被引用。

    参考：[#3762](https://www.sqlalchemy.org/trac/ticket/3762)

### misc

+   **[bug] [orm.declarative]**

    修复了设置单表继承子类的一个 BUG，其中包含额外列的联接表子类将损坏映射表的外键集合，从而干扰关系的初始化。

    参考：[#3797](https://www.sqlalchemy.org/trac/ticket/3797)

### orm

+   **[orm] [bug]**

    修复了`Session.bulk_update_mappings()`中的一个 BUG，其中一个具有替代命名的主键属性无法正确跟踪到 UPDATE 语句中。

    参考：[#3849](https://www.sqlalchemy.org/trac/ticket/3849)

+   **[orm] [bug]**

    修复了对多态加载的映射器进行连接式快速加载时的 BUG，其中多态加载器被设置为未映射表达式，例如 CASE 表达式。

    参考：[#3800](https://www.sqlalchemy.org/trac/ticket/3800)

+   **[orm] [bug]**

    修复了一个错误，即当通过`Session.bind_mapper()`、`Session.bind_table()`或构造函数发送到 Session 的无效绑定时，为无法正确引发的 ArgumentError。

    参考：[#3798](https://www.sqlalchemy.org/trac/ticket/3798)

+   **[orm] [bug]**

    修复了`Session.bulk_save()`中的错误，其中 UPDATE 与实现版本 id 计数器的映射结合使用时无法正确运行。

    参考：[#3781](https://www.sqlalchemy.org/trac/ticket/3781)

+   **[orm] [bug]**

    修复了一个错误，即当在首次调用这些访问器后向映射器/类添加映射器属性或其他 ORM 构造时，`Mapper.attrs`、`Mapper.all_orm_descriptors`和其他派生属性将无法刷新。

    参考：[#3778](https://www.sqlalchemy.org/trac/ticket/3778)

### mssql

+   **[mssql] [bug]**

    更改了用于获取“默认模式名称”的查询，从查询数据库主体表到使用“schema_name()”函数，因为有报告称前一系统在 Azure Data Warehouse 版本上不可用。希望这将最终在所有 SQL Server 版本和认证样式上都起作用。

    参考：[#3810](https://www.sqlalchemy.org/trac/ticket/3810)

+   **[mssql] [bug]**

    更新了 pyodbc 的服务器版本信息方案，使用 SQL Server SERVERPROPERTY()，而不是依赖于 pyodbc.SQL_DBMS_VER，后者在 FreeTDS 中仍然不可靠。

    参考：[#3814](https://www.sqlalchemy.org/trac/ticket/3814)

+   **[mssql] [bug]**

    将错误代码 20017“服务器意外的 EOF”添加到导致连接池重置的断开异常列表中。感谢 Ken Robbins 的拉取请求。

    参考：[#3791](https://www.sqlalchemy.org/trac/ticket/3791)

+   **[mssql] [bug]**

    修复了 pyodbc 方言中的错误（以及大部分不起作用的 adodbapi 方言），其中密码或用户名字段中存在的分号可能被解释为另一个令牌的分隔符；当分号存在时，现在对值进行引用。

    参考：[#3762](https://www.sqlalchemy.org/trac/ticket/3762)

### 杂项

+   **[bug] [orm.declarative]**

    修复了设置单表继承子类的错误，该子类包括额外列，会破坏映射表的外键集合，从而干扰关系的初始化。

    参考：[#3797](https://www.sqlalchemy.org/trac/ticket/3797)

## 1.0.15

发布日期：2016 年 9 月 1 日

### orm

+   **[orm] [bug]**

    修复了子查询急加载中的错误，其中“of_type()”对象的子查询加载链接到第二个普通映射类的子查询加载，或者多个“of_type()”属性的较长链将无法正确链接连接。

    参考：[#3773](https://www.sqlalchemy.org/trac/ticket/3773)，[#3774](https://www.sqlalchemy.org/trac/ticket/3774)

### SQL

+   **[SQL] [bug]**

    修复了`Table`中的错误，其中内部方法`_reset_exported()`会破坏对象的状态。该方法旨在用于可选择的对象，并在某些情况下被 ORM 调用；错误的映射配置可能导致 ORM 在`Table`对象上调用此方法。

    参考：[#3755](https://www.sqlalchemy.org/trac/ticket/3755)

### MySQL

+   **[MySQL] [bug]**

    增加了对在 URL 查询字符串中解析 MySQL/Connector 布尔值和整数参数的支持：connection_timeout, connect_timeout, pool_size, get_warnings, raise_on_warnings, raw, consume_results, ssl_verify_cert, force_ipv6, pool_reset_session, compress, allow_local_infile, use_pure。

    参考：[#3787](https://www.sqlalchemy.org/trac/ticket/3787)

### 杂项

+   **[bug] [ext]**

    修复了`sqlalchemy.ext.baked`中的错误，其中由于变量作用域问题，解除子查询急加载器查询时会失败，涉及多个子查询加载器。感谢 Mark Hahnenberg 的拉取请求。

    参考：[#3743](https://www.sqlalchemy.org/trac/ticket/3743)

### ORM

+   **[ORM] [bug]**

    修复了子查询急加载中的错误，其中“of_type()”对象的子查询加载链接到第二个普通映射类的子查询加载，或者多个“of_type()”属性的较长链将无法正确链接连接。

    参考：[#3773](https://www.sqlalchemy.org/trac/ticket/3773)，[#3774](https://www.sqlalchemy.org/trac/ticket/3774)

### SQL

+   **[SQL] [bug]**

    修复了`Table`中的错误，其中内部方法`_reset_exported()`会破坏对象的状态。该方法旨在用于可选择的对象，并在某些情况下被 ORM 调用；错误的映射配置可能导致 ORM 在`Table`对象上调用此方法。

    参考：[#3755](https://www.sqlalchemy.org/trac/ticket/3755)

### MySQL

+   **[MySQL] [bug]**

    增加了对在 URL 查询字符串中解析 MySQL/Connector 布尔值和整数参数的支持：connection_timeout, connect_timeout, pool_size, get_warnings, raise_on_warnings, raw, consume_results, ssl_verify_cert, force_ipv6, pool_reset_session, compress, allow_local_infile, use_pure。

    参考：[#3787](https://www.sqlalchemy.org/trac/ticket/3787)

### 杂项

+   **[bug] [ext]**

    修复了`sqlalchemy.ext.baked`中的 bug，其中由于变量作用域问题，在涉及多个子查询加载程序时，解除子查询加载程序查询会失败。感谢 Mark Hahnenberg 的拉取请求。

    参考：[#3743](https://www.sqlalchemy.org/trac/ticket/3743)

## 1.0.14

发布日期：2016 年 7 月 6 日

### 示例

+   **[examples] [bug]**

    修复了在 examples/vertical/dictlike-polymorphic.py 示例中发生的回归，导致无法运行的问题。

    参考：[#3704](https://www.sqlalchemy.org/trac/ticket/3704)

### 引擎

+   **[engine] [bug] [postgresql]**

    修复了与`MetaData.schema`参数一起在跨模式外键反射中的 bug，在这种情况下，存在于“default”模式中的引用表将失败，因为没有办法指示一个具有“空白”模式的`Table`。特殊符号`sqlalchemy.schema.BLANK_SCHEMA`已添加为`Table.schema`和`Sequence.schema`的可用值，表示应强制模式名称为`None`，即使指定了`MetaData.schema`。

    参考：[#3716](https://www.sqlalchemy.org/trac/ticket/3716)

### sql

+   **[sql] [bug]**

    修复了 SQL 数学否定运算符中的问题，其中表达式的类型将不再是原始类型的数值类型。这将导致确定结果集行为的问题。

    参考：[#3735](https://www.sqlalchemy.org/trac/ticket/3735)

+   **[sql] [bug]**

    修复了在 1.0 系列过渡到`__slots__`时，导致 sqlalchemy.util.Properties 的`__getstate__` / `__setstate__`方法无法正常工作的 bug。该问题可能会影响一些第三方应用程序。感谢 Pieter Mulder 的拉取请求。

    参考：[#3728](https://www.sqlalchemy.org/trac/ticket/3728)

+   **[sql] [bug]**

    `FromClause.count()`将在 1.1 中被弃用。此函数使用表中的任意列，并不可靠；对于核心使用，应优先使用`func.count()`。

    参考：[#3724](https://www.sqlalchemy.org/trac/ticket/3724)

+   **[sql] [bug]**

    修复了`CTE`结构中的 bug，当使用联合时会导致它无法正确克隆，这在递归 CTE 中很常见。不正确的克隆会在各种 ORM 上下文中使用 CTE 时引发错误，例如`column_property()`的上下文。

    参考：[#3722](https://www.sqlalchemy.org/trac/ticket/3722)

+   **[sql] [bug]**

    修复了`Table.tometadata()`会为每个具有`unique=True`参数的`Column`对象创建重复的`UniqueConstraint`的错误。

    参考：[#3721](https://www.sqlalchemy.org/trac/ticket/3721)

### postgresql

+   **[postgresql] [bug]**

    修复了`TypeDecorator`和`Variant`类型未被 PostgreSQL 方言深度检查以确定是否需要呈现 SMALLSERIAL 或 BIGSERIAL 而不是 SERIAL 的错误。

    参考：[#3739](https://www.sqlalchemy.org/trac/ticket/3739)

### oracle

+   **[oracle] [bug]**

    修复了`Select.with_for_update.of`中的错误，其中 Oracle 的“rownum”方法在处理 LIMIT/OFFSET 时未能考虑“OF”子句内的表达式，这些表达式必须在引用子查询内的表达式的最高级别陈述。如果需要，这些表达式现在将添加到子查询中。

    参考：[#3741](https://www.sqlalchemy.org/trac/ticket/3741)

### 示例

+   **[examples] [bug]**

    修复了在示例/vertical/dictlike-polymorphic.py 示例中发生的回归，导致无法运行的问题。

    参考：[#3704](https://www.sqlalchemy.org/trac/ticket/3704)

### 引擎

+   **[engine] [bug] [postgresql]**

    修复了跨模式外键反射中的错误，与`MetaData.schema`参数一起使用时，引用的表存在于“default”模式中，将会失败，因为没有办法指示一个具有“空白”模式的`Table`。特殊符号`sqlalchemy.schema.BLANK_SCHEMA`已添加为`Table.schema`和`Sequence.schema`的可用值，指示模式名称应强制为`None`，即使指定了`MetaData.schema`。

    参考：[#3716](https://www.sqlalchemy.org/trac/ticket/3716)

### sql

+   **[sql] [bug]**

    修复了 SQL 数学否定运算符中的问题，其中表达式的类型将不再是原始表达式的数值类型。这会导致确定结果集行为的类型问题。

    参考：[#3735](https://www.sqlalchemy.org/trac/ticket/3735)

+   **[sql] [bug]**

    修复了由于 1.0 系列向`__slots__`过渡而导致的 sqlalchemy.util.Properties 的`__getstate__` / `__setstate__`方法不起作用的 bug。该问题可能影响某些第三方应用程序。感谢 Pieter Mulder 的拉取请求。

    参考：[#3728](https://www.sqlalchemy.org/trac/ticket/3728)

+   **[sql] [bug]**

    `FromClause.count()`在 1.1 版本中将被弃用。该函数使用表中的任意列，不可靠；对于核心使用，应优先使用`func.count()`。

    参考：[#3724](https://www.sqlalchemy.org/trac/ticket/3724)

+   **[sql] [bug]**

    修复了在使用联合时`CTE`结构未能正确克隆的 bug，这在递归 CTE 中很常见。不正确的克隆会导致在各种 ORM 上下文中使用 CTE 时出错，例如`column_property()`。

    参考：[#3722](https://www.sqlalchemy.org/trac/ticket/3722)

+   **[sql] [bug]**

    修复了`Table.tometadata()`会为每个具有`unique=True`参数的`Column`对象创建重复的`UniqueConstraint`的 bug。

    参考：[#3721](https://www.sqlalchemy.org/trac/ticket/3721)

### postgresql

+   **[postgresql] [bug]**

    修复了一个 bug，即 PostgreSQL 方言未能深入检查`TypeDecorator`和`Variant`类型，以确定是否应该渲染 SMALLSERIAL 或 BIGSERIAL 而不是 SERIAL。

    参考：[#3739](https://www.sqlalchemy.org/trac/ticket/3739)

### oracle

+   **[oracle] [bug]**

    修复了`Select.with_for_update.of`中的 bug，在这里 Oracle 的“rownum”方法对 LIMIT/OFFSET 无法适应“OF”子句内的表达式，这些表达式必须在最顶层引用子查询内的表达式。如果需要，这些表达式现在将添加到子查询中。

    参考：[#3741](https://www.sqlalchemy.org/trac/ticket/3741)

## 1.0.13

发布日期：2016 年 5 月 16 日

### orm

+   **[orm] [bug]**

    修复了`Query.update()`和`Query.delete()`中“evaluate”策略中的一个 bug，即无法适应具有“可调用”值的绑定参数，这种情况发生在沿着关系进行多对一相等表达式过滤时。

    参考：[#3700](https://www.sqlalchemy.org/trac/ticket/3700)

+   **[ORM] [错误]**

    修复了一个 bug，即在使用深层类继承层次结构与多个映射器配置步骤同时使用时，用于反向引用的事件侦听器可能会被错误地应用多次。

    参考：[#3710](https://www.sqlalchemy.org/trac/ticket/3710)

+   **[ORM] [错误]**

    修复了一个 bug，即将`text()`构造传递给`Query.group_by()`方法会引发错误，而不是将对象解释为 SQL 片段。

    参考：[#3706](https://www.sqlalchemy.org/trac/ticket/3706)

+   **[ORM] [错误]**

    匿名标记应用于传递给`column_property()`的`func`构造，因此如果同一属性被引用两次作为列表达式，则名称将被去重，从而避免“模糊列”错误。以前，需要应用`.label(None)`以便取消匿名化名称。

    参考：[#3663](https://www.sqlalchemy.org/trac/ticket/3663)

+   **[ORM] [错误]**

    修复了 1.0 系列中 ORM 加载中出现的回归问题，即对于缺少的预期列引发的异常错误会错误地是`NoneType`错误，而不是预期的`NoSuchColumnError`。

    参考：[#3658](https://www.sqlalchemy.org/trac/ticket/3658)

### 示例

+   **[示例] [错误]**

    将“有向图”示例更改为不再将节点的整数标识符视为重要；“更高”/“更低”引用现在允许在两个方向上存在相互边。

    参考：[#3698](https://www.sqlalchemy.org/trac/ticket/3698)

### SQL

+   **[SQL] [错误]**

    修复了一个 bug，即在与`Engine`一起使用`case_sensitive=False`时，结果集将无法正确适应结果集中的重复列名，导致在 1.0 中执行语句时出错，并阻止 1.1 中“模糊列”异常的功能。

    参考：[#3690](https://www.sqlalchemy.org/trac/ticket/3690)

+   **[SQL] [错误]**

    修复了一个 bug，即对于一个 EXISTS 表达式的否定在结果中未正确类型化为布尔值，并且在 SELECT 列表中也未能匿名别名化，就像对于一个非否定的 EXISTS 结构一样。

    参考：[#3682](https://www.sqlalchemy.org/trac/ticket/3682)

+   **[sql] [bug]**

    修复了一个 bug，即在调用`Insert.values()`时，如果传入的是参数映射列表而不是单个参数映射，则“未消耗的列名”异常不会被触发。感谢 Athena Yao 的拉取请求。

    参考：[#3666](https://www.sqlalchemy.org/trac/ticket/3666)

### postgresql

+   **[postgresql] [bug]**

    为错误字符串“SSL error: decryption failed or bad record mac”添加了断开检测支持。感谢 Iuri de Silvio 的拉取请求。

    参考：[#3715](https://www.sqlalchemy.org/trac/ticket/3715)

### mssql

+   **[mssql] [bug]**

    修复了在 SQL Server 中应用于 OFFSET 选择的 ROW_NUMBER OVER 子句会不当地用本地语句中与语句的 ORDER BY 条件中使用的标签名称重叠的普通列替换的 bug。

    参考：[#3711](https://www.sqlalchemy.org/trac/ticket/3711)

+   **[mssql] [bug] [oracle]**

    修复了 1.0 系列中出现的回归问题，该问题会导致 Oracle 和 SQL Server 方言在这些方言将 SELECT 包装在子查询中以提供 LIMIT/OFFSET 行为时，不正确地计算结果集列，原始 SELECT 语句多次引用相同列时，例如一个列和该列的标签。这个问题与[#3658](https://www.sqlalchemy.org/trac/ticket/3658)有关，当错误发生时，它还会导致`NoneType`错误，而不是报告无法定位列。

    参考：[#3657](https://www.sqlalchemy.org/trac/ticket/3657)

### oracle

+   **[oracle] [bug]**

    修复了 cx_Oracle 连接过程中的一个 bug，当用户、密码或 dsn 为空时会导致 TypeError。这阻止了对 Oracle 数据库的外部身份验证，并阻止了连接到默认 dsn。现在，oracle://连接字符串使用操作系统用户名登录到默认 dsn，相当于使用 sqlplus 连接时使用‘/’。

    参考：[#3705](https://www.sqlalchemy.org/trac/ticket/3705)

+   **[oracle] [bug]**

    修复了主要由 Oracle 使用的结果代理中的一个 bug，当二进制和其他 LOB 类型参与时，当使用查询/语句缓存时，类型级别的结果处理器，特别是二进制类型本身所需的处理器以及任何其他处理器，在第一次运行语句后会丢失，因为它从缓存的结果元数据中被移除。

    参考：[#3699](https://www.sqlalchemy.org/trac/ticket/3699)

### 杂项

+   **[bug] [py3k]**

    修复了“to_list”转换中的 bug，其中单个字节对象将被转换为单个字符列表。这将影响使用主键为字节对象的`Query.get()`方法等其他事项。 

    参考：[#3660](https://www.sqlalchemy.org/trac/ticket/3660)

### orm

+   **[orm] [bug]**

    修复了在`Query.update()`和`Query.delete()`的“evaluate”策略中的 bug，该 bug 无法适应具有“callable”值的绑定参数，当通过关系沿着多对一等式表达式进行过滤时会发生这种情况。

    参考：[#3700](https://www.sqlalchemy.org/trac/ticket/3700)

+   **[orm] [bug]**

    修复了一个 bug，当在深层类继承层次结构中与多个映射器配置步骤一起使用时，用于反向引用的事件侦听器可能会被错误地应用多次。

    参考：[#3710](https://www.sqlalchemy.org/trac/ticket/3710)

+   **[orm] [bug]**

    修复了一个 bug，通过将`text()`构造传递给`Query.group_by()`方法会引发错误，而不是将对象解释为 SQL 片段。

    参考：[#3706](https://www.sqlalchemy.org/trac/ticket/3706)

+   **[orm] [bug]**

    匿名标记应用于传递给`column_property()`的`func`构造，因此如果同一属性被引用两次作为列表达式，则名称将被去重，从而避免“模糊列”错误。以前，需要应用`.label(None)`以便取消匿名化名称。

    参考：[#3663](https://www.sqlalchemy.org/trac/ticket/3663)

+   **[orm] [bug]**

    修复了在 ORM 加载中出现的 1.0 系列中的回归，其中对于缺少的预期列引发的异常将错误地是一个`NoneType`错误，而不是预期的`NoSuchColumnError`。

    参考：[#3658](https://www.sqlalchemy.org/trac/ticket/3658)

### 例子

+   **[例子] [bug]**

    将“有向图”示例更改为不再将节点的整数标识符视为重要；“更高”/“更低”引用现在允许双向的相互边。

    参考：[#3698](https://www.sqlalchemy.org/trac/ticket/3698)

### sql

+   **[sql] [bug]**

    修复了当使用 `case_sensitive=False` 与 `Engine` 时，结果集未能正确适应结果集中的重复列名，导致在 1.0 中执行语句时出错，并阻止了在 1.1 中“模糊列”异常的发生。

    引用：[#3690](https://www.sqlalchemy.org/trac/ticket/3690)

+   **[sql] [bug]**

    修复了对 EXISTS 表达式的否定不会正确地被类型化为布尔值，并且在 SELECT 列表中也不会被匿名别名化的 bug，与非否定的 EXISTS 构造一样。

    引用：[#3682](https://www.sqlalchemy.org/trac/ticket/3682)

+   **[sql] [bug]**

    修复了在 `Insert.values()` 被调用时，当传递了参数映射的列表而不是单个参数映射时，“未使用的列名”异常未能被触发的 bug。感谢 Athena Yao 提交的 Pull 请求。

    引用：[#3666](https://www.sqlalchemy.org/trac/ticket/3666)

### postgresql

+   **[postgresql] [bug]**

    添加了对错误字符串 “SSL error: decryption failed or bad record mac” 的断开检测支持。感谢 Iuri de Silvio 提交的 Pull 请求。

    引用：[#3715](https://www.sqlalchemy.org/trac/ticket/3715)

### mssql

+   **[mssql] [bug]**

    修复了在 SQL Server 中对于 OFFSET 查询应用的 ROW_NUMBER OVER 子句会不正确地替换语句中与语句的 ORDER BY 条件重叠的纯列的 bug。

    引用：[#3711](https://www.sqlalchemy.org/trac/ticket/3711)

+   **[mssql] [bug] [oracle]**

    修复了在 1.0 系列中出现的回归，该回归会导致 Oracle 和 SQL Server 方言在将 SELECT 包装在子查询中以提供 LIMIT/OFFSET 行为时，当原始 SELECT 语句多次引用同一列时（例如列和该列的标签）时，不正确地计算结果集列。此问题与 [#3658](https://www.sqlalchemy.org/trac/ticket/3658) 相关，因为当错误发生时，它也会导致 `NoneType` 错误，而不是报告无法定位列。

    引用：[#3657](https://www.sqlalchemy.org/trac/ticket/3657)

### oracle

+   **[oracle] [bug]**

    修复了 cx_Oracle 连接过程中出现的 bug，当用户、密码或 dsn 中有任一为空时会导致 TypeError。这阻止了对 Oracle 数据库的外部认证，并阻止了连接到默认 dsn。现在 connect 字符串 oracle:// 将使用操作系统用户名登录到默认 dsn，相当于使用 sqlplus 进行连接时使用 ‘/’。

    引用：[#3705](https://www.sqlalchemy.org/trac/ticket/3705)

+   **[oracle] [bug]**

    修复了主要由 Oracle 使用的结果代理中的一个错误，当涉及二进制和其他 LOB 类型时，当使用查询/语句缓存时，类型级别的结果处理器，特别是二进制类型本身所需的处理器以及任何其他处理器，在第一次运行语句后会丢失，因为它从缓存的结果元数据中被移除。

    引用：[#3699](https://www.sqlalchemy.org/trac/ticket/3699)

### 杂项

+   **[bug] [py3k]**

    修复了“to_list”转换中的错误，其中单个字节对象将被转换为单个字符列表。这将影响使用`Query.get()`方法获取主键为字节对象的对象等其他情况。

    引用：[#3660](https://www.sqlalchemy.org/trac/ticket/3660)

## 1.0.12

发布日期：2016 年 2 月 15 日

### orm

+   **[orm] [bug]**

    修复了`Session.merge()`中的错误，其中具有复合主键的对象具有某些但不是所有 PK 字段的值会发出 SELECT 语句，将内部 NEVER_SET 符号泄漏到查询中，而不是检测到此对象没有可搜索的主键并且不应发出 SELECT。

    引用：[#3647](https://www.sqlalchemy.org/trac/ticket/3647)

+   **[orm] [bug]**

    自 0.9 版本以来的固定回归，0.9 风格的加载器选项系统未能适应单个查询中多个`undefer_group()`加载器选项。现在将考虑多个`undefer_group()`选项，即使针对同一实体也是如此。

    引用：[#3623](https://www.sqlalchemy.org/trac/ticket/3623)

### 引擎

+   **[engine] [bug] [mysql]**

    重新审视[#2696](https://www.sqlalchemy.org/trac/ticket/2696)，首次发布于 1.0.10，试图通过发出警告来解决 Python 2 缺乏异常上下文报告的问题，该警告是由于尝试回滚已失败的事务时被第二个异常中断而引发的；在 MySQL 后端与意外丢失的保存点一起发生时，此问题仍会发生，然后在尝试回滚时会导致“没有这样的保存点”错误，遮蔽了原始条件是什么。

    此方法已泛化为 Core 中的“安全重新引发”函数，该函数在 ORM 和 Core 中发生，任何处于尝试提交时出现错误而回滚事务的地方，包括 `Session` 和 `Connection` 提供的上下文管理器，并在诸如“RELEASE SAVEPOINT”失败等操作中进行。之前，此修复仅针对 ORM flush/commit 过程中的特定路径；现在也针对所有事务上下文管理器执行。

    参考：[#2696](https://www.sqlalchemy.org/trac/ticket/2696)

### sql

+   **[sql] [bug]**

    修复了在将 `insert()`、`update()` 或 `delete()` 构造编译为字符串 SQL 时未传播“literal_binds”标志的问题。感谢 Tim Tate 提供的拉取请求。

    参考：[#3643](https://www.sqlalchemy.org/trac/ticket/3643)

+   **[sql] [bug]**

    修复了无意中使用 Python 的 `__contains__` 覆盖与列表达式（例如使用 `'x' in col`）会导致 ARRAY 类型无限循环的问题，因为 Python 将此推迟到永不为此类型引发异常的 `__getitem__` 访问。总的来说，现在所有对 `__contains__` 的使用都会引发 NotImplementedError。

    参考：[#3642](https://www.sqlalchemy.org/trac/ticket/3642)

+   **[sql] [bug]**

    修复了在 0.9 系列左右出现的 `Table` 元数据构造中的错误，其中向反序列化的 `Table` 添加列时会无法正确建立 `Column` 在‘c’集合中，导致在诸如 ORM 配置等领域出现问题。这可能影响 `extend_existing` 等用例。

    参考：[#3632](https://www.sqlalchemy.org/trac/ticket/3632)

### postgresql

+   **[postgresql] [bug]**

    修复了在 `text()` 构造中的错误，其中双冒号表达式未正确转义，例如 `some\:\:expr`，这在呈现 PostgreSQL 样式的 CAST 表达式时最常需要。

    参考：[#3644](https://www.sqlalchemy.org/trac/ticket/3644)

### mssql

+   **[mssql] [bug]**

    修复了在对 MSSQL 上的日期时间值使用 `extract()` 函数时的语法问题；关键字周围的引号已被移除。感谢 Guillaume Doumenc 提供的拉取请求。

    参考：[#3624](https://www.sqlalchemy.org/trac/ticket/3624)

+   **[mssql] [bug] [firebird]**

    修复了 1.0 版中的一个回归，其中通过纯文本或通过 `text()` 构造发出的 UPDATE 或 DELETE 语句的即时提取不再调用 cursor.rowcount，影响那些一旦关闭游标就擦除 cursor.rowcount 的驱动程序，例如 SQL Server ODBC 和 Firebird 驱动程序。

    参考：[#3622](https://www.sqlalchemy.org/trac/ticket/3622)

### oracle

+   **[oracle] [bug] [jython]**

    修复了 Jython Oracle 编译器中的一个小问题，涉及到 “RETURNING” 的渲染，该问题允许当前不受支持/未经测试的方言与 1.0 系列基本工作。感谢 Carlos Rivas 提供的拉取请求。

    参考：[#3621](https://www.sqlalchemy.org/trac/ticket/3621)

### 杂项

+   **[bug] [py3k]**

    修复了一些异常重新引发场景会将异常附加到自身作为“原因”的错误；虽然 Python 3 解释器可以接受这样做，但它可能会在 iPython 中导致无限循环。

    参考：[#3625](https://www.sqlalchemy.org/trac/ticket/3625)

### orm

+   **[orm] [bug]**

    修复了 `Session.merge()` 中的一个错误，在此错误中，具有复合主键的对象对于一些但不是所有 PK 字段的值会发出一个 SELECT 语句，泄露内部的 NEVER_SET 符号到查询中，而不是检测到此对象没有可搜索的主键，并且不应发出任何 SELECT。

    参考：[#3647](https://www.sqlalchemy.org/trac/ticket/3647)

+   **[orm] [bug]**

    修复了自 0.9 版以来的一个回归，其中 0.9 风格的加载器选项系统未能适应单个查询中的多个 `undefer_group()` 加载器选项。现在将考虑多个 `undefer_group()` 选项，即使针对相同的实体也是如此。

    参考：[#3623](https://www.sqlalchemy.org/trac/ticket/3623)

### engine

+   **[engine] [bug] [mysql]**

    重新访问 [#2696](https://www.sqlalchemy.org/trac/ticket/2696)，首次发布于 1.0.10 版本中，该版本尝试通过发出警告来解决 Python 2 缺乏异常上下文报告的问题，该异常被第二个异常中断时尝试回滚已经失败的事务；这个问题继续出现在 MySQL 后端与意外丢失的保存点一起使用时，然后当尝试回滚时会导致“没有这样的保存点”错误，遮蔽了原始条件是什么。

    该方法已被泛化为 Core 中的“安全重新引发”函数，该函数在 ORM 和 Core 中的任何事务因尝试提交时发生错误而被回滚时发生，包括`Session`和`Connection`提供的上下文管理器，并在诸如“RELEASE SAVEPOINT”失败等操作中发生。以前，修复仅适用于 ORM 刷新/提交过程中的特定路径；现在它也适用于所有事务上下文管理器。

    参考：[#2696](https://www.sqlalchemy.org/trac/ticket/2696)

### sql

+   **[sql] [bug]**

    修复了在将`insert()`、`update()`或`delete()`构造编译为字符串 SQL 时，“literal_binds”标志未传播的问题。感谢 Tim Tate 的拉取请求。

    参考：[#3643](https://www.sqlalchemy.org/trac/ticket/3643)

+   **[sql] [bug]**

    修复了在意外使用 Python `__contains__` 覆盖与列表达式（例如通过使用 `'x' in col`）会导致 ARRAY 类型无限循环的问题，因为 Python 将此延迟到`__getitem__`访问，而此类型永远不会引发。总体而言，现在所有对`__contains__`的使用都会引发 NotImplementedError。

    参考：[#3642](https://www.sqlalchemy.org/trac/ticket/3642)

+   **[sql] [bug]**

    修复了在 0.9 系列左右出现的`Table`元数据构造中的错误，向`Table`添加列时，反序列化的`Table`未能正确建立‘c’集合中的`Column`，导致 ORM 配置等领域出现问题。这可能会影响`extend_existing`等用例。

    参考：[#3632](https://www.sqlalchemy.org/trac/ticket/3632)

### postgresql

+   **[postgresql] [bug]**

    修复了`text()`构造中的错误，双冒号表达式无法正确转义，例如`some\:\:expr`，这在渲染 PostgreSQL 风格的 CAST 表达式时最常见。

    参考：[#3644](https://www.sqlalchemy.org/trac/ticket/3644)

### mssql

+   **[mssql] [bug]**

    修复了在 MSSQL 上针对日期时间值使用`extract()`函数的语法；关键字周围的引号被移除。感谢 Guillaume Doumenc 的拉取请求。

    参考：[#3624](https://www.sqlalchemy.org/trac/ticket/3624)

+   **[mssql] [bug] [firebird]**

    修复了 1.0 版本中的一个回归问题，即对于通过纯文本或通过`text()`构造发出的 UPDATE 或 DELETE 语句，不再调用 cursor.rowcount 的急切获取，影响那些在关闭游标后擦除 cursor.rowcount 的驱动程序，如 SQL Server ODBC 和 Firebird 驱动程序。

    参考：[#3622](https://www.sqlalchemy.org/trac/ticket/3622)

### oracle

+   **[oracle] [bug] [jython]**

    修复了 Jython Oracle 编译器中的一个小问题，涉及“RETURNING”的呈现，这使得当前不受支持/未经测试的方言可以在 1.0 系列中基本工作。感谢 Carlos Rivas 的拉取请求。

    参考：[#3621](https://www.sqlalchemy.org/trac/ticket/3621)

### 杂项

+   **[bug] [py3k]**

    修复了一些异常重新引发场景会将异常附加到自身作为“原因”的 bug；虽然 Python 3 解释器可以接受这种情况，但在 iPython 中可能会导致无限循环。

    参考：[#3625](https://www.sqlalchemy.org/trac/ticket/3625)

## 1.0.11

发布日期：2015 年 12 月 22 日

### orm

+   **[orm] [bug]**

    由于对 [#3593](https://www.sqlalchemy.org/trac/ticket/3593) 的修复导致 1.0.10 中引入的回归 bug，对于从 poly_subclass->class->poly_baseclass 连接到 polymorphic joinedload 的检查会在 class->poly_subclass->class 的情况下失败。

    参考：[#3611](https://www.sqlalchemy.org/trac/ticket/3611)

+   **[orm] [bug]**

    修复了`Session.bulk_update_mappings()`及相关功能在使用时未增加版本 id 计数器的 bug。这里的体验仍然有些粗糙，因为给定字典中需要原始版本 id，并且目前还没有清晰的错误报告。

    参考：[#3610](https://www.sqlalchemy.org/trac/ticket/3610)

+   **[orm] [bug]**

    对`Mapper.eager_defaults`标志进行了重大修复，该标志在多个 UPDATE 语句需要被发出时没有被正确地遵守，无论是作为 flush 的一部分还是作为批量更新操作。此外，在更新语句中不必要地发出了 RETURNING。

    参考：[#3609](https://www.sqlalchemy.org/trac/ticket/3609)

+   **[orm] [bug]**

    修复了使用`Query.select_from()`方法会导致后续调用`Query.with_parent()`方法失败的 bug。

    参考：[#3606](https://www.sqlalchemy.org/trac/ticket/3606)

### sql

+   **[sql] [bug]**

    修复了`Update.return_defaults()`中的错误，该错误会导致所有未包含在 SET 子句中（例如主键列）的插入默认保持列被渲染到 RETURNING 中，尽管这是一个 UPDATE 操作。

    引用：[#3609](https://www.sqlalchemy.org/trac/ticket/3609)

### mysql

+   **[mysql] [bug]**

    调整了用于解析 MySQL 视图的正则表达式，不再假定反射视图源中存在“ALGORITHM”关键字，因为一些用户报告在某些 Amazon RDS 环境中不存在该关键字。

    引用：[#3613](https://www.sqlalchemy.org/trac/ticket/3613)

+   **[mysql] [bug]**

    向 MySQL 5.7 的 MySQL 方言添加了新的保留字，包括‘generated’、‘optimizer_costs’、‘stored’、‘virtual’。感谢 Hanno Schlichting 的拉取请求。

### 杂项

+   **[bug] [ext]**

    进一步修复了[#3605](https://www.sqlalchemy.org/trac/ticket/3605)，在`MutableDict`上的 pop 方法，其中“default”参数未包含。

    引用：[#3605](https://www.sqlalchemy.org/trac/ticket/3605)

+   **[bug] [ext]**

    修复了烘焙加载器系统中的错误，该系统的全局猴子补丁用于设置烘焙懒加载器，会干扰依赖延迟加载作为后备的其他加载器策略，例如连接和子查询急加载器，在映射器配置时导致`IndexError`异常。

    引用：[#3612](https://www.sqlalchemy.org/trac/ticket/3612)

### orm

+   **[orm] [bug]**

    修复了 1.0.10 中由于[#3593](https://www.sqlalchemy.org/trac/ticket/3593)修复引起的回归，其中为 poly_subclass->class->poly_baseclass 连接添加了多态 joinedload 的检查会导致在 class->poly_subclass->class 的情况下失败。

    引用：[#3611](https://www.sqlalchemy.org/trac/ticket/3611)

+   **[orm] [bug]**

    修复了`Session.bulk_update_mappings()`等在使用时不会增加版本 id 计数器的错误。这里的体验仍然有点粗糙，因为给定字典中仍需要原始版本 id，并且尚无干净的错误报告。

    引用：[#3610](https://www.sqlalchemy.org/trac/ticket/3610)

+   **[orm] [bug]**

    对`Mapper.eager_defaults`标志进行了重大修复，该标志在需要发出多个 UPDATE 语句时（作为刷新或批量更新操作的一部分）不会被正确执行。此外，在更新语句中不必要地发出 RETURNING。

    引用：[#3609](https://www.sqlalchemy.org/trac/ticket/3609)

+   **[orm] [bug]**

    修复了使用 `Query.select_from()` 方法会导致随后调用 `Query.with_parent()` 方法失败的 bug。

    参考：[#3606](https://www.sqlalchemy.org/trac/ticket/3606)

### sql

+   **[sql] [bug]**

    修复了 `Update.return_defaults()` 中的错误，该错误会导致所有插入默认列（否则不包含在 SET 子句中的列，例如主键列）被渲染到 RETURNING 中，尽管这是一个 UPDATE。

    参考：[#3609](https://www.sqlalchemy.org/trac/ticket/3609)

### mysql

+   **[mysql] [bug]**

    调整了用于解析 MySQL 视图的正则表达式，以便我们不再假设反映视图源中存在 “ALGORITHM” 关键字，因为一些用户报告称在某些 Amazon RDS 环境中未出现。

    参考：[#3613](https://www.sqlalchemy.org/trac/ticket/3613)

+   **[mysql] [bug]**

    将新的保留字添加到 MySQL 5.7 的 MySQL 方言中，包括 ‘generated’、‘optimizer_costs’、‘stored’、‘virtual’。Pull request 由 Hanno Schlichting 提供。

### misc

+   **[bug] [ext]**

    对 [#3605](https://www.sqlalchemy.org/trac/ticket/3605) 进行了进一步修复，即对 `MutableDict` 的 pop 方法，在其中没有包含 “default” 参数。

    参考：[#3605](https://www.sqlalchemy.org/trac/ticket/3605)

+   **[bug] [ext]**

    修复了烘焙加载器系统中的错误，该系统的系统范围 monkeypatch 用于设置烘焙延迟加载器会干扰依赖惰性加载作为后备的其他加载器策略，例如 joined 和子查询急切加载器，导致在映射器配置时出现 `IndexError` 异常。

    参考：[#3612](https://www.sqlalchemy.org/trac/ticket/3612)

## 1.0.10

发布日期：2015 年 12 月 11 日

### orm

+   **[orm] [bug]**

    修复了一个问题，即在一对多关系的 post_update 中，如果属性设置为 None 而且之前没有加载，则会导致 UPDATE 未发出。

    参考：[#3599](https://www.sqlalchemy.org/trac/ticket/3599)

+   **[orm] [bug]**

    修复了一个 bug，实际上是在版本 0.8.0 和 0.8.1 之间发生的回归，由 [#2714](https://www.sqlalchemy.org/trac/ticket/2714) 引起。当同时使用“with_polymorphic”时，需要通过子类绑定关系进行连接的连接急切加载会无法从正确的实体进行连接。

    参考：[#3593](https://www.sqlalchemy.org/trac/ticket/3593)

+   **[orm] [bug]**

    修复了在以下情况下会发生的 joinedload 错误：a. 查询包含强制子查询的 limit/offset 条件 b. 关系使用“secondary” c. 关系的 primaryjoin 引用的列既不是主键的一部分，也不是主键列在不同属性名称下的联合继承子类表中 d. 查询推迟了主要连接中存在的列，通常通过不包含在 load_only()中; 必要的列不会出现在子查询中，并产生无效的 SQL。

    参考：[#3592](https://www.sqlalchemy.org/trac/ticket/3592)

+   **[orm] [bug]**

    在`Session.rollback()`在引发异常的`Session.flush()`操作范围内失败时出现的一种罕见情况，这种情况已在某些 MySQL SAVEPOINT 情况下观察到，阻止了在 flush 期间发出的原始数据库异常被观察到，但仅在 Py2K 上，因为 Py2K 不支持异常链接; 在 Py3K 上，原始异常被链接。作为解决方法，在这种特定情况下发出警告，显示至少在继续引发回滚异常之前原始数据库错误的字符串消息。

    参考：[#2696](https://www.sqlalchemy.org/trac/ticket/2696)

### orm 声明

+   **[orm] [declarative] [bug]**

    修复了在 Py2K 中，unicode 文字不能作为声明中的类或其他参数的字符串名称被接受的错误，该错误发生在使用`backref()`在`relationship()`上时。感谢 Nils Philippsen 的拉取请求。

### sql

+   **[sql] [feature]**

    在 UPDATE 语句中添加了对参数顺序化 SET 子句的支持。通过将`update.preserve_parameter_order`标志传递给核心`Update`构造，或者将其添加到 ORM 级别的`Query.update.update_args`字典中，同时将参数本身作为 2 元组列表传递。感谢 Gorka Eguileor 的实现和测试。

    参见

    参数顺序化更新

+   **[sql] [bug]**

    修复了在 `Insert.from_select()` 结构中的问题，当目标 `Table` 具有 Python 端默认值时，`Select` 结构的 `._raw_columns` 集合会在编译 `Insert` 结构时被就地修改。`Select` 结构在编译后会独立存在，其中包含编译 `Insert` 后存在的错误列，并且由于重复的绑定参数，`Insert` 语句本身在第二次编译尝试时会失败。

    参考：[#3603](https://www.sqlalchemy.org/trac/ticket/3603)

+   **[sql] [bug] [postgresql]**

    修复了使用 CREATE TABLE 创建一个没有列的表，但具有约束（如 CHECK 约束）的 bug，会在定义中产生一个错误的逗号；这种情况可能发生在具有自己没有列的 PostgreSQL INHERITS 表中。

    参考：[#3598](https://www.sqlalchemy.org/trac/ticket/3598)

### postgresql

+   **[postgresql] [bug]**

    修复了“FOR UPDATE OF” PostgreSQL 特定的 SELECT 修饰符在所引用的表具有模式限定符时会失败的问题；PG 需要省略模式名称。感谢 Diana Clarke 的拉取请求。

    参考：[#3573](https://www.sqlalchemy.org/trac/ticket/3573)

+   **[postgresql] [bug]**

    修复了将一些 SQL 表达式传递给 `ExcludeConstraint` 的“where”子句时无法正确接受的 bug。感谢 aisch 的拉取请求。

+   **[postgresql] [bug]**

    修复了 `INTERVAL` 的 `.python_type` 属性以与 `python_type` 相同的方式返回 `datetime.timedelta`，而不是引发 `NotImplementedError`。

    参考：[#3571](https://www.sqlalchemy.org/trac/ticket/3571)

### mysql

+   **[mysql] [bug]**

    修复了在 MySQL 反射中的错误，其中 `DATETIME`、`TIMESTAMP` 和 `TIME` 类型的“小数部分”会被错误地放置到未使用的 `timezone` 属性中，而不是 `fsp` 属性中。

    参考：[#3602](https://www.sqlalchemy.org/trac/ticket/3602)

### mssql

+   **[mssql] [错误]**

    将 “20006: 向服务器写入失败” 错误添加到 pymssql 驱动程序的断开连接错误列表中，因为观察到这可能导致连接无法使用。

    参考：[#3585](https://www.sqlalchemy.org/trac/ticket/3585)

+   **[mssql] [错误]**

    现在，如果 SQL 服务器从 DATE 或 TIME 列返回无效的日期或时间格式，则会引发描述性 ValueError，而不是出现 NoneType 错误。感谢 Ed Avis 提交的拉取请求。

+   **[mssql] [错误]**

    修复了对 MSSQL 类型 DATETIME2、TIME 和 DATETIMEOFFSET 生成的 DDL 的问题，如果精度为“零”，则不会生成精度字段。感谢 Jacobo de Vera 提交的拉取请求。

### 测试

+   **[测试] [更改]**

    ORM 和 Core 教程一直以来都是以 doctest 格式存在的，现在在 Python 2 和 Python 3 中都在正常的单元测试套件中执行。

### 杂项

+   **[错误] [扩展]**

    向 `MutableDict` 类添加了对 `dict.pop()` 和 `dict.popitem()` 方法的支持。

    参考：[#3605](https://www.sqlalchemy.org/trac/ticket/3605)

+   **[错误] [py3k]**

    更新了内部的 getargspec() 调用，一些与 py36 相关的装置更新，以及对两个迭代器的更改，以“返回”而不是引发 StopIteration，以允许测试在 Py3.5、Py3.6 上通过而不产生错误或警告，拉取请求由 Jacob MacDonald、Luri de Silvio 和 Phil Jones 提交。

+   **[错误] [扩展]**

    修复了烘焙查询中的一个问题，其中 `.get()` 方法，在直接使用或在延迟加载中使用时，未将映射器的“获取子句”视为缓存键的一部分，导致如果子句重新生成，则绑定参数不匹配。这个子句会被映射器动态缓存，但在高并发场景下，可能在首次访问时生成多次。

    参考：[#3597](https://www.sqlalchemy.org/trac/ticket/3597)

### orm

+   **[orm] [错误]**

    修复了在多对一关系上的 post_update 失败的问题，在这种情况下，如果属性设置为 None 并且以前未加载，则不会发出 UPDATE。

    参考：[#3599](https://www.sqlalchemy.org/trac/ticket/3599)

+   **[orm] [错误]**

    修复了一个实际上是在版本 0.8.0 和 0.8.1 之间发生的回归错误，原因是[#2714](https://www.sqlalchemy.org/trac/ticket/2714)。当“with_polymorphic”也被使用时，需要跨子类绑定关系进行连接的连接急切加载的情况将无法从正确的实体进行连接。

    参考：[#3593](https://www.sqlalchemy.org/trac/ticket/3593)

+   **[orm] [bug]**

    修复了一个 joinedload bug，当 a.查询包含强制子查询的 limit/offset 条件时 b.关系使用“secondary” c.关系的 primaryjoin 引用的列既不是主键的一部分，也不是主键列在一个不同属性名称下的联合继承子类表中 d.查询推迟了在 primaryjoin 中存在的列，通常通过不包括在 load_only()中; 必要的列不会出现在子查询中，并产生无效的 SQL。

    参考：[#3592](https://www.sqlalchemy.org/trac/ticket/3592)

+   **[orm] [bug]**

    一个罕见的情况发生在`Session.rollback()`在引发异常的`Session.flush()`操作范围内失败时，如在一些 MySQL SAVEPOINT 情况下观察到的，阻止了在 flush 期间发出的原始数据库异常被观察到，但仅在 Py2K 上，因为 Py2K 不支持异常链接; 在 Py3K 上，原始异常被链接。作为一种解决方法，在这种特定情况下发出一个警告，至少显示原始数据库错误的字符串消息，然后我们继续引发 rollback-originating 异常。

    参考：[#2696](https://www.sqlalchemy.org/trac/ticket/2696)

### orm 声明式

+   **[orm] [declarative] [bug]**

    修复了一个 bug，在 Py2K 中，unicode 文字不会被接受为声明式中使用`backref()`在`relationship()`中的类或其他参数的字符串名称。感谢 Nils Philippsen 的拉取请求。

### sql

+   **[sql] [feature]**

    增加了对 UPDATE 语句中参数顺序 SET 子句的支持。通过将 `update.preserve_parameter_order` 标志传递给核心 `Update` 构造，或者将其添加到 ORM 级别的 `Query.update.update_args` 字典中，同时将参数本身作为 2 元组列表传递。感谢 Gorka Eguileor 的实现和测试。

    另请参阅

    参数顺序更新

+   **[sql] [bug]**

    修复了在 `Insert.from_select()` 构造中存在的问题，其中在编译 `Insert` 构造时，当目标 `Table` 具有 Python 端默认值时，`Select` 构造的 `._raw_columns` 集合会被就地修改。在编译 `Insert` 构造后，`Select` 构造将独立编译，并且在编译 `Insert` 构造后，`Insert` 语句本身将由于重复的绑定参数而在第二次编译尝试时失败。

    参考：[#3603](https://www.sqlalchemy.org/trac/ticket/3603)

+   **[sql] [bug] [postgresql]**

    修复了使用不带列的表创建 CREATE TABLE，但具有约束（如 CHECK 约束）会在定义中出现错误逗号的 bug；这种情况可能发生在具有自己没有列的 PostgreSQL INHERITS 表中。

    参考：[#3598](https://www.sqlalchemy.org/trac/ticket/3598)

### postgresql

+   **[postgresql] [bug]**

    修复了“FOR UPDATE OF” PostgreSQL 特定的 SELECT 修饰符如果所指表具有模式限定符则会失败的问题；PG 需要省略模式名称。感谢 Diana Clarke 提交的拉取请求。

    参考：[#3573](https://www.sqlalchemy.org/trac/ticket/3573)

+   **[postgresql] [bug]**

    修复了将某些类型的 SQL 表达式传递给 `ExcludeConstraint` 的“where”子句时无法正确接受的 bug。感谢 aisch 提交的拉取请求。

+   **[postgresql] [bug]**

    修复了 `INTERVAL` 的 `.python_type` 属性，使其像 `python_type` 一样返回 `datetime.timedelta`，而不是引发 `NotImplementedError`。

    参考：[#3571](https://www.sqlalchemy.org/trac/ticket/3571)

### mysql

+   **[mysql] [bug]**

    修复了 MySQL 反射中的错误，其中`DATETIME`、`TIMESTAMP` 和 `TIME` 类型的“小数部分”会错误地放置到未被 MySQL 使用的 `timezone` 属性中，而不是 `fsp` 属性中。

    参考：[#3602](https://www.sqlalchemy.org/trac/ticket/3602)

### mssql

+   **[mssql] [bug]**

    将错误“20006: 无法向服务器写入”添加到了 pymssql 驱动程序的断开连接错误列表中，因为观察到这会使连接无法使用。

    参考：[#3585](https://www.sqlalchemy.org/trac/ticket/3585)

+   **[mssql] [bug]**

    如果 SQL 服务器从 DATE 或 TIME 列返回无效的日期或时间格式，现在会引发描述性的 ValueError，而不是以 NoneType 错误失败。感谢 Ed Avis 提供的拉取请求。

+   **[mssql] [bug]**

    修复了对具有“零”精度的 MSSQL 类型 DATETIME2、TIME 和 DATETIMEOFFSET 生成的 DDL 不会生成精度字段的问题。感谢 Jacobo de Vera 提供的拉取请求。

### 测试

+   **[tests] [change]**

    ORM 和 Core 教程一直以 doctest 格式存在，现在在 Python 2 和 Python 3 中都在正常的单元测试套件中进行测试。

### 杂项

+   **[bug] [ext]**

    将对 `MutableDict` 类添加了对 `dict.pop()` 和 `dict.popitem()` 方法的支持。

    参考：[#3605](https://www.sqlalchemy.org/trac/ticket/3605)

+   **[bug] [py3k]**

    更新了内部的 getargspec() 调用，一些与 py36 相关的 fixture 更新，以及对两个迭代器的更改，使其“返回”而不是引发 StopIteration，以便在 Py3.5、Py3.6 上无错误或警告地通过测试，拉取请求由 Jacob MacDonald、Luri de Silvio 和 Phil Jones 提供。

+   **[bug] [ext]**

    修复了在烘焙查询中的问题，其中 `.get()` 方法（直接使用或在惰性加载中使用）未将映射器的“获取子句”视为缓存键的一部分，导致如果子句重新生成则导致绑定参数不匹配。这个子句在并发场景下会被映射器动态地缓存，但在高度并发的场景下，当首次访问时可能会生成多次。

    参考：[#3597](https://www.sqlalchemy.org/trac/ticket/3597)

## 1.0.9

发布日期：2015 年 10 月 20 日

### orm

+   **[orm] [feature]**

    添加了新方法 `Query.one_or_none()`；与 `Query.one()` 相同，但如果未找到行则返回 None。感谢 esiegerman 提交的拉取请求。

+   **[orm] [bug] [postgresql]**

    修复了 1.0 版本中的回归问题，即在 ORM 中使用“executemany”进行 UPDATE 语句的新功能（例如 UPDATE 语句现在在 flush 中使用 executemany() 进行批处理）在 PostgreSQL 和其他支持 RETURNING 的后端上会出现问题，因为服务器端值是通过 RETURNING 检索的，而在 executemany 中不支持。

    参考：[#3556](https://www.sqlalchemy.org/trac/ticket/3556)

+   **[orm] [bug]**

    修复了在内部日志中对某些类型的内部列加载器选项进行字符串化时可能出现的罕见 TypeError。

    参考：[#3539](https://www.sqlalchemy.org/trac/ticket/3539)

+   **[orm] [bug]**

    修复了 `Session.bulk_save_objects()` 中的错误，其中具有某种“更新时获取”值的映射列，且在给定对象中不存在时，会导致操作中出现 AttributeError。

    参考：[#3525](https://www.sqlalchemy.org/trac/ticket/3525)

+   **[orm] [bug]**

    修复了 1.0 版本中“noload”加载策略在对多对一关系无法正常工作的回归问题。加载器使用了一个 API 将“None”放入字典中，但实际上不再写入值；这是 [#3061](https://www.sqlalchemy.org/trac/ticket/3061) 的一个副作用。

    参考：[#3510](https://www.sqlalchemy.org/trac/ticket/3510)

### 示例

+   **[examples] [bug]**

    修复了“history_meta”示例中的两个问题，其中历史跟踪可能遇到空历史，并且键入替代属性名称的列无法正确跟踪的问题。修复由 Alex Fraser 提供。

### sql

+   **[sql] [bug]**

    修复了 1.0 版本中发布的多值插入语句的默认处理器回归问题，[#3288](https://www.sqlalchemy.org/trac/ticket/3288)，在使用默认时默认列类型不会传播到编译后的语句中，导致绑定级别类型处理程序不会被调用。

    参考：[#3520](https://www.sqlalchemy.org/trac/ticket/3520)

### postgresql

+   **[postgresql] [bug]**

    对 1.0.6 中发布的反映存储选项和 USING 的新 PostgreSQL 功能的调整，禁用了 PostgreSQL 版本 < 8.2 的功能，其中未提供 `reloptions` 列；这使得基于 8.0.x 版本的 PostgreSQL 的 Amazon Redshift 再次可以正常工作。修复由 Pete Hollobon 提供。

### oracle

+   **[oracle] [bug] [py3k]**

    修复了 cx_Oracle 版本 5.2 的支持，该版本在 Python 3 下触发了 SQLAlchemy 的版本检测，并无意中未使用正确的 Unicode 模式。这会导致问题，例如绑定变量被误解释为 NULL，以及行被静默地未返回。

    此更改也已**回溯**至：0.7.0b1

    参考：[#3491](https://www.sqlalchemy.org/trac/ticket/3491)

+   **[oracle] [bug]**

    修复了 Oracle 方言中的一个 bug，即反射带有引号强制转换为全小写的表和其他符号的情况在反射查询中无法正确识别。现在，`quoted_name` 构造已应用于检测在“名称规范化”过程中被强制转换为全小写的传入符号名称。

    参考：[#3548](https://www.sqlalchemy.org/trac/ticket/3548)

### misc

+   **[功能] [扩展]**

    向 `AssociationProxy` 构造函数添加了 `AssociationProxy.info` 参数，以适应在 [#2971](https://www.sqlalchemy.org/trac/ticket/2971) 中添加的 `AssociationProxy.info` 访问器。这是可能的，因为 `AssociationProxy` 是显式构造的，不像通过装饰器语法隐式构造的混合体。

    参考：[#3551](https://www.sqlalchemy.org/trac/ticket/3551)

+   **[bug] [sybase]**

    修复了关于 Sybase 反射的两个问题，允许没有主键的表被反射，同时确保涉及外键检测的 SQL 语句被预先获取，以避免嵌套查询时出现驱动程序问题。此处的修复由 Eugene Zapolsky 提供；请注意，我们目前无法测试 Sybase 以本地验��这些更改。

    参考：[#3508](https://www.sqlalchemy.org/trac/ticket/3508)，[#3509](https://www.sqlalchemy.org/trac/ticket/3509)

### orm

+   **[orm] [功能]**

    添加了新方法 `Query.one_or_none()`；与 `Query.one()` 相同，但如果未找到行，则返回 None。感谢 esiegerman 提交的拉取请求。

+   **[orm] [bug] [postgresql]**

    修复了 1.0 版本中使用“executemany”在 ORM 中进行 UPDATE 语句的新功能（例如现在在 flush 中使用 executemany() 批处理 UPDATE 语句）在 PostgreSQL 和其他支持 RETURNING 的后端上会出现问题，当使用服务器端版本生成方案时，由于服务器端值是通过 RETURNING 检索的，而在使用 executemany 时不支持。

    参考：[#3556](https://www.sqlalchemy.org/trac/ticket/3556)

+   **[orm] [错误]**

    修复了在内部日志中对某些类型的内部列加载器选项进行字符串化时可能出现的罕见 TypeError。

    参考：[#3539](https://www.sqlalchemy.org/trac/ticket/3539)

+   **[orm] [错误]**

    修复了 `Session.bulk_save_objects()` 中的 bug，其中一个映射列具有某种“��新时获取”值，并且在给定对象中不存在本地时，会导致操作中的 AttributeError。 

    参考：[#3525](https://www.sqlalchemy.org/trac/ticket/3525)

+   **[orm] [错误]**

    修复了 1.0 版本中“noload”加载策略在一对多关系中无法正常工作的回归问题。加载器使用一个 API 将“None”放入字典中，这实际上不再写入值；这是 [#3061](https://www.sqlalchemy.org/trac/ticket/3061) 的一个副作用。

    参考：[#3510](https://www.sqlalchemy.org/trac/ticket/3510)

### 例子

+   **[例子] [错误]**

    修复了“history_meta”示例中的两个问题，其中历史跟踪可能遇到空历史，以及一个键入到替代属性名称的列无法正确跟踪的问题。修复由 Alex Fraser 提供。

### sql

+   **[sql] [错误]**

    修复了 1.0 版本中默认处理多值插入语句的回归问题，[#3288](https://www.sqlalchemy.org/trac/ticket/3288)，在默认保持列的情况下，列类型不会传播到编译后的语句中，导致绑定级别的类型处理程序不会被调用。

    参考：[#3520](https://www.sqlalchemy.org/trac/ticket/3520)

### postgresql

+   **[postgresql] [错误]**

    对于 1.0.6 版本中发布的 [#3455](https://www.sqlalchemy.org/trac/ticket/3455) 的新 PostgreSQL 特性进行调整，以禁用 PostgreSQL 版本 < 8.2 的功能，其中未提供 `reloptions` 列；这允许 Amazon Redshift 再次正常工作，因为它基于 PostgreSQL 的 8.0.x 版本。修复由 Pete Hollobon 提供。

### oracle

+   **[oracle] [错误] [py3k]**

    修复了 cx_Oracle 版本 5.2 的支持，该版本在 Python 3 下触发了 SQLAlchemy 的版本检测，并无意中未使用正确的 Unicode 模式进行 Python 3。这会导致问题，例如绑定变量被误解释为 NULL，行被静默地未返回。

    此更改也被**回溯**到：0.7.0b1

    参考：[#3491](https://www.sqlalchemy.org/trac/ticket/3491)

+   **[oracle] [错误]**

    修复了 Oracle 方言中的一个错误，即反射带有引号名称以强制全部小写的表和其他符号时，在反射查询中将无法正确识别。现在，`quoted_name`构造应用于在“名称规范化”过程中检测为强制全部小写的传入符号名称。

    参考：[#3548](https://www.sqlalchemy.org/trac/ticket/3548)

### 杂项

+   **[功能] [扩展]**

    在`AssociationProxy`构造函数中添加了`AssociationProxy.info`参数，以适应[#2971](https://www.sqlalchemy.org/trac/ticket/2971)中添加的`AssociationProxy.info`访问器。这是因为`AssociationProxy`是显式构造的，不像通过装饰器语法隐式构造的混合体。

    参考：[#3551](https://www.sqlalchemy.org/trac/ticket/3551)

+   **[错误] [Sybase]**

    修复了关于 Sybase 反射的两个问题，允许没有主键的表被反射，同时确保涉及外键检测的 SQL 语句被预先获取，以避免嵌套查询时出现驱动程序问题。这里的修复由 Eugene Zapolsky 提供；请注意，我们目前无法测试 Sybase 以本地验证这些更改。

    参考：[#3508](https://www.sqlalchemy.org/trac/ticket/3508), [#3509](https://www.sqlalchemy.org/trac/ticket/3509)

## 1.0.8

发布日期：2015 年 7 月 22 日

### 引擎

+   **[引擎] [错误]**

    修复了一个严重问题，即池“checkout”事件处理程序可能针对未调用“connect”事件处理程序的陈旧连接进行调用，在池尝试重新连接并失败后；陈旧连接将保留并在随后的尝试中使用。这个问题在 1.0 系列中的影响更大，1.0.2 之后，因为它还向事件处理程序提供了一个空白的`.info`字典；在 1.0.2 之前，`.info`字典仍然是先前的字典。

    此更改也**回溯**至：0.7.0b1

    参考：[#3497](https://www.sqlalchemy.org/trac/ticket/3497)

### sqlite

+   **[sqlite] [错误]**

    修复了 SQLite 方言中的一个错误，即反射包含非字母字符（如点或空格）的唯一约束的名称时，将不会反映其名称。

    此更改也**回溯**至：0.9.10

    参考：[#3495](https://www.sqlalchemy.org/trac/ticket/3495)

### misc

+   **[misc] [bug]**

    修复了一个问题，即 utils 中的特定基类没有实现`__slots__`，因此该类的所有子类也没有实现，从而使得使用`__slots__`没有意义。这个问题只在 IronPython 上引起问题，显然它不兼容 cPython 的`__slots__`行为。

    参考：[#3494](https://www.sqlalchemy.org/trac/ticket/3494)

### engine

+   **[engine] [bug]**

    修复了一个关键问题，即池中的“checkout”事件处理程序可能针对一个陈旧的连接进行调用，而“connect”事件处理程序尚未被调用，在池尝试重新连接并失败后；陈旧的连接将保留并在随后的尝试中使用。这个问题在 1.0.2 之后的 1.0 系列中影响更大，因为它还向事件处理程序提供了一个空白的`.info`字典；在 1.0.2 之前，`.info`字典仍然是先前的字典。

    此更改也**回溯**到：0.7.0b1

    参考：[#3497](https://www.sqlalchemy.org/trac/ticket/3497)

### sqlite

+   **[sqlite] [bug]**

    修复了 SQLite 方言中的一个错误，即反射包含非字母字符（如点或空格）的唯一约束的名称时，不会反映其名称。

    此更改也**回溯**到：0.9.10

    参考：[#3495](https://www.sqlalchemy.org/trac/ticket/3495)

### misc

+   **[misc] [bug]**

    修复了一个问题，即 utils 中的特定基类没有实现`__slots__`，因此该类的所有子类也没有实现，从而使得使用`__slots__`没有意义。这个问题只在 IronPython 上引起问题，显然它不兼容 cPython 的`__slots__`行为。

    参考：[#3494](https://www.sqlalchemy.org/trac/ticket/3494)

## 1.0.7

发布日期：2015 年 7 月 20 日

### orm

+   **[orm] [bug]**

    修复了 1.0 中的回归问题，即覆盖`__eq__()`以返回一个非布尔类型对象的值对象，在工作单元更新操作期间进行`bool()`测试，而在 0.9 中，`__eq__()`的返回值被测试为“is True”以防止这种情况发生。

    参考：[#3469](https://www.sqlalchemy.org/trac/ticket/3469)

+   **[orm] [bug]**

    修复了 1.0 中的回归问题，即如果在“优化继承加载”中加载了“延迟”属性，则不会正确填充，这是在使用联接表继承填充过期或未加载属性的特殊 SELECT 中发出的情况下，用于填充联接表而不加载基表。这与 SQLA 1.0 不再猜测加载延迟列的事实有关，必须明确指示。

    参考：[#3468](https://www.sqlalchemy.org/trac/ticket/3468)

+   **[orm] [bug]**

    修复了 1.0 版本中的问题，即在`aliased()`对象上的同义词映射属性的“父实体”将解析为原始映射器，而不是其`aliased()`版本，从而导致依赖于此属性的`Query`出现问题（例如，在构造函数中给出的唯一代表性属性）无法确定查询的正确 FROM 子句。

    参考：[#3466](https://www.sqlalchemy.org/trac/ticket/3466)

### orm declarative

+   **[orm] [declarative] [bug]**

    在`AbstractConcreteBase`扩展中修复了一个 bug，即在 ABC 基类上设置一个具有不同属性名与列名的列时，该列将无法正确映射到最终的基类上。在 0.9 版本上会默默失败，而在 1.0 版本上会引发一个 ArgumentError，因此在 1.0 版本之前可能不会被注意到。

    参考：[#3480](https://www.sqlalchemy.org/trac/ticket/3480)

### engine

+   **[engine] [bug]**

    修复了一个回归问题，即 ORM `Query`对象使用的`ResultProxy`上的新方法（作为[#3175](https://www.sqlalchemy.org/trac/ticket/3175)性能增强的一部分）在驱动程序（通常是 MySQL）无法正确生成 cursor.description 时不会引发“此结果不返回行”异常；而是会引发一个针对 NoneType 的 AttributeError。

    参考：[#3481](https://www.sqlalchemy.org/trac/ticket/3481)

+   **[engine] [bug]**

    修复了一个问题，即`ResultProxy.keys()`会返回未调整的内部符号名称，用于“匿名”标签，这些标签是我们在没有标签的 SQL 函数和类似情况下生成的“foo_1”类型的标签。这是作为#918 的性能增强的副作用实现的。

    参考：[#3483](https://www.sqlalchemy.org/trac/ticket/3483)

### sql

+   **[sql] [feature]**

    添加了一个`ColumnElement.cast()`方法，其作用与独立的`cast()`函数相同。感谢 Sebastian Bank 的 Pull 请求。

    参考：[#3459](https://www.sqlalchemy.org/trac/ticket/3459)

+   **[sql] [bug]**

    修复了一个 bug，即在与`and_()`或`or_()`结合使用时，对字面值`True`或`False`的强制转换会导致 AttributeError。

    参考：[#3490](https://www.sqlalchemy.org/trac/ticket/3490)

+   **[sql] [bug]**

    修复了一个潜在问题，即自定义的`FunctionElement`子类或其他列元素错误地将“None”或任何其他无效对象声明为`.type`属性时，会报告此异常而不是递归溢出。

    参考：[#3485](https://www.sqlalchemy.org/trac/ticket/3485)

+   **[sql] [bug]**

    修复了取模 SQL 运算符无法反向工作的 bug，因为缺少了 `__rmod__` 方法。感谢 dan-gittik 提交的拉取请求。

### 模式

+   **[schema] [feature]**

    增加了对 PostgreSQL 和 Oracle 支持的 CREATE SEQUENCE 的 MINVALUE、MAXVALUE、NO MINVALUE、NO MAXVALUE 和 CYCLE 参数。感谢 jakeogh 提交的拉取请求。

### orm

+   **[orm] [bug]**

    修复了 1.0 版本中的一个问题，即值对象覆盖 `__eq__()` 以返回一个非布尔值对象，例如一些 geoalchemy 类型和 numpy 类型，在工作单元更新操作期间会被测试为 `bool()`，而在 0.9 版本中，`__eq__()` 的返回值被测试为“is True”以防止这种情况发生。

    参考：[#3469](https://www.sqlalchemy.org/trac/ticket/3469)

+   **[orm] [bug]**

    修复了 1.0 版本中的一个问题，即“延迟”属性在“优化继承加载”中未正确填充，这是在联接表继承的情况下发出的特殊 SELECT，用于填充过期或未加载的属性，而不加载基表。这与 SQLA 1.0 不再猜测加载延迟列有关，必须明确指定。

    参考��[#3468](https://www.sqlalchemy.org/trac/ticket/3468)

+   **[orm] [bug]**

    修复了 1.0 版本中的一个问题，即在 `aliased()` 对象上的同义映射属性的“父实体”将解析为原始映射器，而不是其 `aliased()` 版本，从而导致依赖于此属性的 `Query` 出现问题（例如，在构造函数中给出的唯一代表性属性）无法确定查询的正确 FROM 子句。

    参考：[#3466](https://www.sqlalchemy.org/trac/ticket/3466)

### orm 声明

+   **[orm] [declarative] [bug]**

    修复了在`AbstractConcreteBase`扩展中的一个 bug，即在 ABC 基类上设置的列，如果属性名与列名不同，则不会正确映射到最终基类上。在 0.9 版本上失败是静默的，而在 1.0 版本上会引发一个 ArgumentError，因此在 1.0 版本之前可能不会被注意到。

    引用：[#3480](https://www.sqlalchemy.org/trac/ticket/3480)

### engine

+   **[engine] [bug]**

    修复了用于 ORM `Query` 对象的新方法（作为 [#3175](https://www.sqlalchemy.org/trac/ticket/3175) 性能增强的一部分）的回归错误，在驱动程序（通常是 MySQL）无法正确生成 cursor.description 时，不会引发“此结果不返回行”异常；而是会引发 AttributeError。反而会引发针对 NoneType 的 AttributeError。

    引用：[#3481](https://www.sqlalchemy.org/trac/ticket/3481)

+   **[engine] [bug]**

    修复了`ResultProxy.keys()`返回未调整的内部符号名称的回归错误，这些名称是我们在没有标签的 SQL 函数和类似情况下生成的“foo_1”类型的标签。这是作为 #918 的一部分实施的性能增强的副作用。

    引用：[#3483](https://www.sqlalchemy.org/trac/ticket/3483)

### sql

+   **[sql] [feature]**

    添加了一个 `ColumnElement.cast()` 方法，其作用与独立的 `cast()` 函数相同。此功能感谢 Sebastian Bank 提交的拉取请求。

    引用：[#3459](https://www.sqlalchemy.org/trac/ticket/3459)

+   **[sql] [bug]**

    修复了将字面常量`True`或`False`与`and_()`或`or_()`结合使用时会导致 AttributeError 的错误。

    引用：[#3490](https://www.sqlalchemy.org/trac/ticket/3490)

+   **[sql] [bug]**

    修复了自定义 `FunctionElement` 或其他列元素的子类错误地将“None”或任何其他无效对象陈述为`.type`属性时报告此异常而不是递归溢出的潜在问题。

    引用：[#3485](https://www.sqlalchemy.org/trac/ticket/3485)

+   **[sql] [bug]**

    修复了取模 SQL 运算符无法反转的错误，原因是缺少了`__rmod__`方法。此修复感谢 dan-gittik 提交的拉取请求。

### schema

+   **[schema] [feature]**

    添加了对 CREATE SEQUENCE 的 MINVALUE、MAXVALUE、NO MINVALUE、NO MAXVALUE 和 CYCLE 参数的支持，这些参数由 PostgreSQL 和 Oracle 支持。此功能感谢 jakeogh 提交的拉取请求。

## 1.0.6

发布日期：2015 年 6 月 25 日

### orm

+   **[orm] [bug]**

    修复了 1.0 系列中的一个重大回归问题，即当 version_id_counter 功能导致对象的版本计数器在对象的行没有净变化时被增加，而是通过关系（例如通常是一对多）与之相关联或取消关联时，会导致一个 UPDATE 语句更新对象的版本计数器而不做其他任何操作。在相对较新的“服务器端”和/或“程序化/条件化”版本计数器功能被使用的用例中（例如将 version_id_generator 设置为 False），这个错误可能导致发出一个没有有效 SET 子句的 UPDATE。

    参考：[#3465](https://www.sqlalchemy.org/trac/ticket/3465)

+   **[orm] [bug]**

    修复了 1.0 版本中的一个问题，即对于不使用任何鉴别器的单继承子类的显式连接条件进行 JOIN 时，单继承连接的增强行为[#3222](https://www.sqlalchemy.org/trac/ticket/3222)不适当地发生，导致额外的“AND NULL”子句。

    参考：[#3462](https://www.sqlalchemy.org/trac/ticket/3462)

+   **[orm] [bug]**

    修复了新的`Session.bulk_update_mappings()`功能中的一个 bug，其中用于定位行的 WHERE 子句中使用的主键列也会包含在 SET 子句中，将它们的值不必要地设置为它们自己。感谢 Patrick Hayes 的拉取请求。

    参考：[#3451](https://www.sqlalchemy.org/trac/ticket/3451)

+   **[orm] [bug]**

    修复了一个意外使用回归问题，即自定义`Comparator`对象使用`__clause_element__()`方法并返回一个 ORM 映射的`InstrumentedAttribute`对象而不是显式地`ColumnElement`时，当作为表达式传递给`Session.query()`时将无法正确处理。0.9 版本的逻辑恰好成功处理了这个问题，因此现在支持这种用例。

    参考：[#3448](https://www.sqlalchemy.org/trac/ticket/3448)

### sql

+   **[sql] [bug]**

    修复了一个 bug，其中应用于`Label`对象的子句适应在所有情况下都会失败，以至于任何使用`Label.self_group()`的 SQL 操作都会使用原始未适应的表达式。这将导致 ORM `aliased()`构造无法完全适应由`column_property`映射的属性，因此在某些类型的 SQL 比较中，未别名化的表可能泄漏出来。

    参考：[#3445](https://www.sqlalchemy.org/trac/ticket/3445)

### postgresql

+   **[postgresql] [feature]**

    添加了对在 CREATE INDEX 下使用存储参数的支持，使用新的关键字参数`postgresql_with`。还添加了反射支持，以支持`postgresql_with`标志以及`postgresql_using`标志，这些标志现在将设置在反射的`Index`对象上，并且还存在于`Inspector.get_indexes()`结果中的新“dialect_options”字典中。感谢 Pete Hollobon 的拉取请求。

    另请参阅

    索引存储参数

    参考：[#3455](https://www.sqlalchemy.org/trac/ticket/3455)

+   **[postgresql] [feature]**

    添加了新的执行选项`max_row_buffer`，当使用`stream_results`选项时，由 psycopg2 方言解释，设置了可分配的行缓冲区大小限制。该值还是基于发送给`Query.yield_per()`的整数值提供的。感谢 mcclurem 的拉取请求。

+   **[postgresql] [bug] [pypy]**

    重新修复了在 1.0.5 版本中首次发布的问题，以再次修复 psycopg2cffi 对 JSONB 支持，因为他们在 2.7.1 版本中突然切换到对 JSONB 类型的无条件解码。版本检测现在指定 2.7.1 作为我们应该期望 DBAPI 为我们进行 json 编码的地方。

    参考：[#3439](https://www.sqlalchemy.org/trac/ticket/3439)

+   **[postgresql] [bug]**

    修复了`ExcludeConstraint`构造以支持其他对象（如`Index`）现在支持的常见功能，即列表达式可以被指定为任意 SQL 表达式，如`cast`或`text`。

    引用：[#3454](https://www.sqlalchemy.org/trac/ticket/3454)

### mssql

+   **[mssql] [bug]**

    修复了在使用 `VARBINARY` 类型与 NULL + pyodbc 的 INSERT 时出现的问题；pyodbc 需要传递一个特殊对象才能持久化 NULL。由于 [#3039](https://www.sqlalchemy.org/trac/ticket/3039) 的原因，`VARBINARY` 类型现在通常是 `LargeBinary` 的默认类型之一，因此该问题在 1.0 版本中部分是一个回归。pymssql 驱动程序似乎不受影响。

    引用：[#3464](https://www.sqlalchemy.org/trac/ticket/3464)

### 杂项

+   **[bug] [documentation]**

    修复了一个内部“记忆化”方法类型的例程，使得不再使用 Python 描述符；修复了这些方法的可检查性，包括对 Sphinx 文档的支持。

    引用：[#2077](https://www.sqlalchemy.org/trac/ticket/2077)

### orm

+   **[orm] [bug]**

    修复了 1.0 系列中的一个重大回归，即当版本 _id_counter 功能导致对象的版本计数器在对象的行没有净变化时被递增时，而是通过关系与之相关联（例如通常是多对一关系）或与之解除关联，导致更新语句仅更新对象的版本计数器而不更新其他内容。在相对较新的“服务器端”和/或“程序化/条件性”版本计数器功能（例如将 version_id_generator 设置为 False）的使用案例中，该错误可能导致发出无有效 SET 子句的更新。

    引用：[#3465](https://www.sqlalchemy.org/trac/ticket/3465)

+   **[orm] [bug]**

    修复了 1.0 版本中的一个回归，即对于一个没有使用任何鉴别器的单继承子类，通过显式连接条件进行 JOIN 时，增强的单继承连接行为不适当地发生，导致额外的“AND NULL”子句。

    引用：[#3462](https://www.sqlalchemy.org/trac/ticket/3462)

+   **[orm] [bug]**

    修复了新的`Session.bulk_update_mappings()`功能中的一个 bug，即在用于定位行的 WHERE 子句中使用的主键列也会包含在 SET 子句中，将它们的值不必要地设置为它们自己。感谢 Patrick Hayes 的拉取请求。

    参考：[#3451](https://www.sqlalchemy.org/trac/ticket/3451)

+   **[orm] [bug]**

    修复了一个意外使用回归，即自定义`Comparator`对象使用了`__clause_element__()`方法并返回一个 ORM 映射的`InstrumentedAttribute`对象而不是显式地`ColumnElement`时，当作为表达式传递给`Session.query()`时无法正确处理。0.9 版本中的逻辑恰好成功，因此现在支持这种用例。

    参考：[#3448](https://www.sqlalchemy.org/trac/ticket/3448)

### sql

+   **[sql] [bug]**

    修复了一个 bug，即应用于`Label`对象的子句适应在某些情况下无法容纳带标签的 SQL 表达式，因此任何使用`Label.self_group()`的 SQL 操作都将使用原始未适应的表达式。其中一个影响是，ORM `aliased()` 构造不会完全适应由`column_property`映射的属性，因此在某些类型的 SQL 比较中，未别名化的表可能泄漏出来。

    参考：[#3445](https://www.sqlalchemy.org/trac/ticket/3445)

### postgresql

+   **[postgresql] [feature]**

    添加了对在 CREATE INDEX 下使用存储参数的支持，使用新的关键字参数 `postgresql_with`。还添加了反射支持，以支持 `postgresql_with` 标志和 `postgresql_using` 标志，这些标志现在将设置在反射的`Index`对象上，并在`Inspector.get_indexes()`的结果中以新的“dialect_options”字典的形式呈现。感谢 Pete Hollobon 的拉取请求。

    另请参见

    索引存储参数

    参考文献：[#3455](https://www.sqlalchemy.org/trac/ticket/3455)

+   **[postgresql] [feature]**

    添加了新的执行选项`max_row_buffer`，当使用`stream_results`选项时，由 psycopg2 方言解释，该选项设置可以分配的行缓冲区的大小限制。这个值也是根据发送给`Query.yield_per()`的整数值提供的。感谢 mcclurem 的拉取请求。

+   **[postgresql] [bug] [pypy]**

    重新修复了在 1.0.5 中首次发布的此问题，以再次修复 psycopg2cffi 的 JSONB 支持，因为他们突然在版本 2.7.1 中无条件地切换了对 JSONB 类型的解码。版本检测现在指定 2.7.1 作为我们应该期望 DBAPI 为我们进行 json 编码的位置。

    参考文献：[#3439](https://www.sqlalchemy.org/trac/ticket/3439)

+   **[postgresql] [bug]**

    修复了`ExcludeConstraint`结构，以支持其他对象（如`Index`）现在支持的常见功能，即列表达式可以指定为任意的 SQL 表达式，如`cast`或`text`。

    参考文献：[#3454](https://www.sqlalchemy.org/trac/ticket/3454)

### mssql

+   **[mssql] [bug]**

    修复了在使用`VARBINARY`类型与插入 NULL + pyodbc 时出现的问题；pyodbc 需要传递一个特殊对象才能持久化 NULL。由于[#3039](https://www.sqlalchemy.org/trac/ticket/3039)，`VARBINARY`类型现在通常是`LargeBinary`的默认值，所以这个问题在 1.0 中部分是一个退化。pymssql 驱动程序似乎不受影响。

    参考文献：[#3464](https://www.sqlalchemy.org/trac/ticket/3464)

### misc

+   **[bug] [documentation]**

    修复了内部的“记忆化”方法类型，不再使用 Python 描述符；修复了这些方法的可检查性，包括对 Sphinx 文档的支持。

    参考文献：[#2077](https://www.sqlalchemy.org/trac/ticket/2077)

## 1.0.5

发布日期：2015 年 6 月 7 日

### orm

+   **[orm] [feature]**

    添加了新的事件`InstanceEvents.refresh_flush()`，在刷新过程中调用 INSERT 或 UPDATE 级别的默认值通过 RETURNING 或 Python 端的默认值调用时触发。这是为了提供一个钩子，不再像[#3167](https://www.sqlalchemy.org/trac/ticket/3167)那样在刷新过程中不再调用属性和验证事件。

    参考：[#3427](https://www.sqlalchemy.org/trac/ticket/3427)

+   **[orm] [bug]**

    当`Query`返回行时使用的“轻量级命名元组”未正确实现`__slots__`，以至于它仍然有一个`__dict__`。这个问题已经解决，但是在极端情况下，如果有人给返回的元组分配值，那么将不再起作用。

    参考：[#3420](https://www.sqlalchemy.org/trac/ticket/3420)

### engine

+   **[engine] [feature]**

    添加了新的引擎事件`ConnectionEvents.engine_disposed()`。在调用`Engine.dispose()`方法后调用。

+   **[engine] [feature]**

    对引擎插件钩子进行调整，使得当使用方言插件时，`URL.get_dialect()`方法将继续返回最终的`Dialect`对象，而不需要调用者知道`Dialect.get_dialect_cls()`方法。

    参考：[#3379](https://www.sqlalchemy.org/trac/ticket/3379)

+   **[engine] [bug]**

    修复已知布尔值在`engine_from_config()`中使用时未被正确解析的错误；这些包括`pool_threadlocal`和 psycopg2 参数`use_native_unicode`。

    参考：[#3435](https://www.sqlalchemy.org/trac/ticket/3435)

+   **[engine] [bug]**

    添加了对表现不当的 DBAPI 情况的支持，该情况下 pep-249 异常名称与完全不同名称的异常类相关联，阻止 SQLAlchemy 自己的异常包装正确包装错误。使用的 SQLAlchemy 方言需要实现一个新的访问器`DefaultDialect.dbapi_exception_translation_map`来支持此功能；现在为 py-postgresql 方言实现了此功能。

    参考：[#3421](https://www.sqlalchemy.org/trac/ticket/3421)

+   **[engine] [bug]**

    修复了一个 bug，涉及当使用池检出事件处理程序并在处理程序本身中进行连接尝试但失败时，拥有连接记录直到连接错误本身的堆栈跟踪被释放才会被释放的情况。对于仅使用单个连接的测试池的情况，这意味着池将被完全检出，直到该堆栈跟踪被释放。这主要影响非常具体的调试场景，并且在任何生产应用程序中都不太可能引起注意。修复方法是在重新引发捕获的异常之前显式检入记录。

    参考：[#3419](https://www.sqlalchemy.org/trac/ticket/3419)

### sql

+   **[sql] [feature]**

    官方添加了对 `Insert.from_select()` 中的 SELECT 使用的 CTE 的支持。此行为直到 0.9.9 时偶然有效，当时由于与 [#3248](https://www.sqlalchemy.org/trac/ticket/3248) 的不相关更改而不再有效。请注意，这是在 INSERT 之后 SELECT 之前呈现 WITH 子句的方式；在 INSERT、UPDATE、DELETE 的顶层呈现 CTE 的全部功能是针对稍后发布的新功能。

    此更改还被**反向移植**至：0.9.10

    参考：[#3418](https://www.sqlalchemy.org/trac/ticket/3418)

### postgresql

+   **[postgresql] [bug] [pypy]**

    修复了与 pypy psycopg2cffi 方言相关的一些打字和测试问题，特别是当前的 2.7.0 版本不支持 JSONB 类型。对于 psycopg2 功能的版本检测已调整为 psycopg2cffi 的特定子版本。此外，已启用了对 psycopg2cffi 下所有 psycopg2 功能系列的测试覆盖。

    参考：[#3439](https://www.sqlalchemy.org/trac/ticket/3439)

### mssql

+   **[mssql] [bug]**

    向 MSSQL 方言添加了一个新的方言标志`legacy_schema_aliasing`，当设置为 False 时，将禁用非常古老和过时的行为，即编译器试图将所有模式限定的表名转换为别名，以解决旧问题和不再可定位的问题，其中 SQL Server 无法在所有情况下解析多部分标识符名称。此行为阻止更复杂的语句正确工作，包括使用提示的语句，以及嵌入相关 SELECT 语句的 CRUD 语句。与其继续修复功能以使其与更复杂的语句一起工作，不如将其禁用，因为对于任何现代 SQL Server 版本都不应再需要它。对于 1.0.x 系列，该标志默认为 True，保持当前行为不变。在 1.1 系列中，默认为 False。对于 1.0 系列，当未显式设置为任何值时，将在语句中首次使用模式限定的表时发出警告，建议为所有现代 SQL Server 版本设置该标志为 False。

    请参见

    遗留模式模式

    参考：[#3424](https://www.sqlalchemy.org/trac/ticket/3424), [#3430](https://www.sqlalchemy.org/trac/ticket/3430)

### 杂项

+   **[功能] [扩展]**

    支持将`*args`传递给烘焙查询的初始可调用对象，方式与`BakedQuery.add_criteria()`和`BakedQuery.with_criteria()`方法支持`*args`的方式相同。初始 PR 由 Naoki INADA 提供。

+   **[功能] [扩展]**

    向`MutableBase`添加了一个新的半公共方法`MutableBase._get_listen_keys()`。在需要时重写此方法，如果`MutableBase`子类需要使事件传播到与可变类型关联的键以外的属性键时，当拦截`InstanceEvents.refresh()`或`InstanceEvents.refresh_flush()`事件时。目前的示例是使用`MutableComposite`的复合体。

    参考：[#3427](https://www.sqlalchemy.org/trac/ticket/3427)

+   **[错误] [扩展]**

    由于对[#3167](https://www.sqlalchemy.org/trac/ticket/3167)的错误修复，`sqlalchemy.ext.mutable`扩展中出现了回归，其中属性和验证事件不再在刷新过程中调用。在列级别的 Python 端默认值负责生成 INSERT 或 UPDATE 的新值，或者在“eager defaults”模式下从 RETURNING 子句中获取值时，可变扩展依赖于此行为。当填充新值时，不会触发任何事件，可变扩展无法建立正确的强制转换或历史监听。添加了一个新事件`InstanceEvents.refresh_flush()`，可变扩展现在在这种情况下使用。

    参考：[#3427](https://www.sqlalchemy.org/trac/ticket/3427)

### ORM

+   **[ORM] [特性]**

    添加了新事件`InstanceEvents.refresh_flush()`，在刷新过程中调用 INSERT 或 UPDATE 级别的默认值通过 RETURNING 或 Python 端默认值获取时调用。这是为了提供一个钩子，因为由于[#3167](https://www.sqlalchemy.org/trac/ticket/3167)，属性和验证事件不再在刷新过程中调用。

    参考：[#3427](https://www.sqlalchemy.org/trac/ticket/3427)

+   **[ORM] [错误]**

    当`Query`返回行时使用的“轻量级命名元组”未正确实现`__slots__`，导致仍然有一个`__dict__`。这个问题已经解决，但在极不可能的情况下，如果有人给返回的元组赋值，那将不再起作用。

    参考：[#3420](https://www.sqlalchemy.org/trac/ticket/3420)

### 引擎

+   **[引擎] [特性]**

    添加了新的引擎事件`ConnectionEvents.engine_disposed()`。在调用`Engine.dispose()`方法后调用。

+   **[引擎] [特性]**

    调整引擎插件钩子，使得`URL.get_dialect()`方法在使用方言插件时仍将返回最终的`Dialect`对象，而不需要调用者知道`Dialect.get_dialect_cls()`方法。

    参考：[#3379](https://www.sqlalchemy.org/trac/ticket/3379)

+   **[引擎] [错误]**

    修复了已知布尔值在 `engine_from_config()` 中未被正确解析的 bug；这些包括 `pool_threadlocal` 和 psycopg2 参数 `use_native_unicode`。

    参考：[#3435](https://www.sqlalchemy.org/trac/ticket/3435)

+   **[引擎] [错误]**

    增加了对行为不端的 DBAPI 的支持，该 DBAPI 将 pep-249 异常名称链接到完全不同名称的异常类，从而阻止 SQLAlchemy 自己的异常包装适当地包装错误。使用的 SQLAlchemy 方言需要实现一个新的访问器 `DefaultDialect.dbapi_exception_translation_map` 来支持此功能；现在已为 py-postgresql 方言实现了这一功能。

    参考：[#3421](https://www.sqlalchemy.org/trac/ticket/3421)

+   **[引擎] [错误]**

    修复了一个 bug，涉及当使用池检出事件处理程序并在处理程序中进行连接尝试并失败时，拥有连接记录直到连接错误本身的堆栈跟踪被释放之前不会被释放的情况。对于仅使用单个连接的测试池的情况，这意味着池将完全被检出，直到该堆栈跟踪被释放。这主要影响非常特定的调试场景，不太可能在任何生产应用程序中引起注意。修复方法是在重新引发捕获的异常之前显式检入记录。

    参考：[#3419](https://www.sqlalchemy.org/trac/ticket/3419)

### sql

+   **[sql] [功能]**

    为 `Insert.from_select()` 中的 SELECT 中使用的 CTE 添加了官方支持。此行为在 0.9.9 之前意外工作，当时由于与 [#3248](https://www.sqlalchemy.org/trac/ticket/3248) 的不相关更改而不再工作。请注意，这是在 INSERT 之后、SELECT 之前呈现 WITH 子句；在后续版本中，将针对新功能发布顶层 INSERT、UPDATE、DELETE 中呈现 CTE 的完整功能。

    此更改也已**回溯**至：0.9.10

    参考：[#3418](https://www.sqlalchemy.org/trac/ticket/3418)

### postgresql

+   **[postgresql] [错误] [pypy]**

    修复了与 pypy psycopg2cffi 方言相关的一些打字和测试问题，特别是当前的 2.7.0 版本不支持 JSONB 类型。对于 psycopg2 功能的版本检测已调整为适用于 psycopg2cffi 的特定子版本。此外，已启用了对 psycopg2cffi 下所有 psycopg2 功能的测试覆盖。

    参考：[#3439](https://www.sqlalchemy.org/trac/ticket/3439)

### mssql

+   **[mssql] [bug]**

    添加了一个新的方言标志到 MSSQL 方言`legacy_schema_aliasing`，当设置为 False 时，将禁用一个非常古老和过时的行为，即编译器尝试将所有模式限定的表名转换为别名，以解决 SQL 服务器在所有情况下无法解析多部分标识符名称的旧问题。该行为阻止更复杂的语句正确工作，包括使用提示的语句，以及嵌入相关 SELECT 语句的 CRUD 语句。与其继续修复该功能以使其与更复杂的语句一起工作，不如将其禁用，因为对于任何现代 SQL 服务器版本，它应该不再需要。该标志在 1.0.x 系列中默认为 True，保持当前行为不变。在 1.1 系列中，它将默认为 False。对于 1.0 系列，如果未显式设置为任何值，则在语句中首次使用模式限定表时会发出警告，建议为所有现代 SQL Server 版本将该标志设置为 False。

    参见

    Legacy Schema Mode

    参考：[#3424](https://www.sqlalchemy.org/trac/ticket/3424), [#3430](https://www.sqlalchemy.org/trac/ticket/3430)

### 杂项

+   **[feature] [ext]**

    添加了对`*args`的支持，以便像`BakedQuery.add_criteria()`和`BakedQuery.with_criteria()`方法一样，将`*args`传递给烘焙查询的初始可调用函数。初始 PR 由 Naoki INADA 提供。

+   **[feature] [ext]**

    添加了一个新的半公开方法到`MutableBase` `MutableBase._get_listen_keys()`。在拦截`InstanceEvents.refresh()`或`InstanceEvents.refresh_flush()`事件时，需要重写此方法，以便在`MutableBase`子类需要事件传播到与可变类型关联的键之外的属性键时。目前的示例是使用`MutableComposite`的复合类型。

    参考：[#3427](https://www.sqlalchemy.org/trac/ticket/3427)

+   **[bug] [ext]**

    修复了`sqlalchemy.ext.mutable`扩展中的回归，这是由于对[#3167](https://www.sqlalchemy.org/trac/ticket/3167)的错误修复导致的，其中属性和验证事件不再在刷新过程中调用。可变扩展依赖于这种行为，即在列级 Python 端默认值负责生成 INSERT 或 UPDATE 的新值时，或者当从“eager defaults”模式的 RETURNING 子句中获取值时。当填充新值时，不会触发任何事件，可变扩展无法建立正确的强制转换或历史监听。添加了一个新事件`InstanceEvents.refresh_flush()`，可变扩展现在在这种情况下使用。

    参考：[#3427](https://www.sqlalchemy.org/trac/ticket/3427)

## 1.0.4

发布日期：2015 年 5 月 7 日

### orm

+   **[orm] [错误]**

    修复了一个意外使用回归，即如果关系的 primaryjoin 涉及与不可哈希类型（如 HSTORE）的比较，由于在语句参数上进行了基于哈希的检查，导致懒加载失败，这是在 1.0 中由于[#3061](https://www.sqlalchemy.org/trac/ticket/3061)而修改为使用哈希，并在[#3368](https://www.sqlalchemy.org/trac/ticket/3368)中修改为在比“load on pending”更常见的情况下发生。现在会事先检查值是否具有`__hash__`属性。

    参考：[#3416](https://www.sqlalchemy.org/trac/ticket/3416)

+   **[orm] [错误]**

    自[#3347](https://www.sqlalchemy.org/trac/ticket/3347)添加的断言进行了自由化，以防止在使用`innerjoin=True`将内连接拼接在一起时出现未知条件；如果一些连接使用了“secondary”表，则需要进一步展开连接以通过断言。

    参考：[#3347](https://www.sqlalchemy.org/trac/ticket/3347), [#3412](https://www.sqlalchemy.org/trac/ticket/3412)

+   **[orm] [错误]**

    修复/添加了更多表达式的测试，这些表达式被报告为在新添加到`Query.column_descriptions`的‘entity’键值后失败，重新设计了发现“from”子句的逻辑，以适应来自别名类的列，以及在这些情况下报告“aliased”标志的正确值。

    参考：[#3320](https://www.sqlalchemy.org/trac/ticket/3320), [#3409](https://www.sqlalchemy.org/trac/ticket/3409)

### 模式

+   **[模式] [错误]**

    修复了在[#3341](https://www.sqlalchemy.org/trac/ticket/3341)中引入的增强约束附加逻辑中的错误，这种不寻常的情况是，约束同时引用`Column`对象和字符串列名称的混合，自动附加到列时逻辑将被跳过；对于在这种情况下自动附加约束，必须事先将所有列组装到目标表上。还添加了一个关于原始特性以及这个更改的迁移文档的新部分。

    另见

    当其引用的列附加到表时，约束引用未附加列可以自动附加到表中

    参考资料：[#3411](https://www.sqlalchemy.org/trac/ticket/3411)

### 测试

+   **[tests] [bug] [pypy]**

    修复了一个导入问题，导致“pypy setup.py test”无法正常工作。

    此更改也**回溯**到：0.9.10

    参考资料：[#3406](https://www.sqlalchemy.org/trac/ticket/3406)

### 其他

+   **[bug] [ext]**

    修复了使用扩展属性仪表化系统时的错误，当调用`class_mapper()`时，如果输入无效且无法弱引用，例如整数，则不会引发正确的异常。

    此更改也**回溯**到：0.9.10

    参考资料：[#3408](https://www.sqlalchemy.org/trac/ticket/3408)

### orm

+   **[orm] [bug]**

    修复了一个意外的回归问题，在这种奇怪的情况下，关系的主要连接涉及与非可散列类型（如 HSTORE）的比较，懒惰加载将失败，因为语句参数上的哈希定向检查在 1.0 中修改为使用哈希化，并在[#3368](https://www.sqlalchemy.org/trac/ticket/3368)中进行修改以发生在比“加载挂起”更常见的情况下。现在会先检查值是否具有`__hash__`属性。

    参考资料：[#3416](https://www.sqlalchemy.org/trac/ticket/3416)

+   **[orm] [bug]**

    放宽了一个断言，作为[#3347](https://www.sqlalchemy.org/trac/ticket/3347)的一部分添加的，以保护在连接到`innerjoin=True`的连接的连接中拼接内连接时的未知条件；如果一些连接使用“次要”表，则需要进一步展开连接以通过。

    参考资料：[#3347](https://www.sqlalchemy.org/trac/ticket/3347), [#3412](https://www.sqlalchemy.org/trac/ticket/3412)

+   **[orm] [bug]**

    修复/添加了更多被报告为在新添加到`Query.column_descriptions`的‘entity’键值后失败的表达式的测试，重新调整了发现“from”子句的逻辑，以适应来自别名类的列，以及在这些情况下报告“aliased”标志的正确值。

    参考：[#3320](https://www.sqlalchemy.org/trac/ticket/3320), [#3409](https://www.sqlalchemy.org/trac/ticket/3409)

### 模式

+   **[schema] [bug]**

    修复了在[#3341](https://www.sqlalchemy.org/trac/ticket/3341)中引入的增强约束附加逻辑中的错误，其中在一个约束引用同时引用`Column`对象和字符串列名的罕见情况下，自动附加到列时的逻辑将被跳过；在这种情况下，要使约束自动附加，所有列必须事先组装到目标表上。在迁移文档中添加了一个关于原始功能以及此更改的新部分。

    参见

    引用未附加的列的约束可以在其引用的列附加时自动附加到表格

    参考：[#3411](https://www.sqlalchemy.org/trac/ticket/3411)

### 测试

+   **[tests] [bug] [pypy]**

    修复了一个导入问题，导致“pypy setup.py test”无法正常工作。

    此更改也**回溯**到：0.9.10

    参考：[#3406](https://www.sqlalchemy.org/trac/ticket/3406)

### 杂项

+   **[bug] [ext]**

    修复了使用扩展属性检测系统时的错误，当使用`class_mapper()`调用无效输入时，也不是弱引用时，不会引发正确的异常，例如整数。

    此更改也**回溯**到：0.9.10

    参考：[#3408](https://www.sqlalchemy.org/trac/ticket/3408)

## 1.0.3

发布日期：2015 年 4 月 30 日

### orm

+   **[orm] [bug] [pypy]**

    从 0.9.10 发布之前修复了由于[#3349](https://www.sqlalchemy.org/trac/ticket/3349)引起的回归，其中在`Query.update()`或`Query.delete()`上检查查询状态时，将空元组与自身使用`is`进行比较，这在 PyPy 上失败，导致在这种情况下产生`True`；这将在 0.9 中错误地发出警告，并在 1.0 中引发异常。

    参考：[#3405](https://www.sqlalchemy.org/trac/ticket/3405)

+   **[orm] [bug]**

    修复了在发布之前从 0.9.10 开始的回归，新添加的将 `entity` 添加到 `Query.column_descriptions` 访问器会失败，如果目标实体是从诸如 `Table` 或 `CTE` 对象之类的核心可选择对象生成的。

    参考：[#3320](https://www.sqlalchemy.org/trac/ticket/3320)，[#3403](https://www.sqlalchemy.org/trac/ticket/3403)

+   **[orm] [bug]**

    在刷新过程中修复了一个回归，当属性设置为更新的 SQL 表达式，并且与属性的先前值进行比较的 SQL 表达式会产生 `==` 或 `!=` 之外的 SQL 比较时，异常 “此子句的布尔值未定义” 会引发。修复确保工作单元不会以这种方式解释 SQL 表达式。 

    参考：[#3402](https://www.sqlalchemy.org/trac/ticket/3402)

+   **[orm] [bug]**

    由于 [#2992](https://www.sqlalchemy.org/trac/ticket/2992)，修复了由于在联合预加载与 `Query.order_by()` 子句中放置文本元素而导致的意外使用回归，这样会将这些元素添加到内部查询的列子句中，以一种被假定为表绑定列名的方式，这在联合预加载需要将查询包装在子查询中以适应限制/偏移量的情况下会发生。

    最初，这里的行为是有意的，即像 `query(User).order_by('name').limit(1)` 这样的查询会按 `user.name` 排序，即使查询被联合预加载修改为在子查询中，因为 `'name'` 会被解释为一个符号，要在 FROM 子句中找到，这种情况下为 `User.name`，然后将其复制到列子句中以确保其存在以供 ORDER BY 使用。但是，该功能未能预见到 `order_by("name")` 指的是本地列子句中已经存在的特定标签名称，而不是绑定到 FROM 子句中的名称的情况。

    此外，该功能还对已弃用的情况（如`order_by("name desc")`）失败，尽管它会发出警告，建议在此处使用`text()`（请注意，该问题不会影响显式使用`text()`的情况），但仍会生成与以前不同的查询，其中“name desc”表达式不当地复制到列子句中。 解决方案是，该功能的“连接式急加载”方面将在增强内部列子句时跳过这些所谓的“标签引用”表达式，就好像它们已经是`text()`构造一样。

    参考：[#3392](https://www.sqlalchemy.org/trac/ticket/3392)

+   **[orm] [bug]**

    修复了关于`MapperEvents.instrument_class()`事件的回归，其中其调用被移动到类管理器对类进行仪器化之后，这与事件文档明确说明的相反。 切换的理由是由于 Declarative 在将类映射为新的`@declared_attr`功能描述的目的之前设置了完整的“仪器管理器”，但也针对经典使用`Mapper`的一致性进行了更改。 然而，SQLSoup 依赖于在任何经典映射下的仪器化之前发生的仪器化事件。 在经典和声明性映射的情况下，该行为被恢复，后者通过简单的备忘录实现，而不使用类管理器。

    参考：[#3388](https://www.sqlalchemy.org/trac/ticket/3388)

+   **[orm] [bug]**

    在新的`QueryEvents.before_compile()`事件中修复了一个问题，即在事件中对要加载的实体集合进行更改会反映在 SQL 中，但在加载过程中不会反映。

    参考：[#3387](https://www.sqlalchemy.org/trac/ticket/3387)

### engine

+   **[engine] [feature]**

    添加了支持引擎/池插件具有高级功能的新功能。在检出的连接包装器级别以及`_ConnectionRecord`的连接池中添加了一个新的“软失效”功能。这类似于现代池失效，因为连接不会被主动关闭，但仅在下次检出时被重用；这本质上是该功能的每个连接版本。添加了一个新的事件`PoolEvents.soft_invalidate()`来补充它。

    还添加了新的标志`ExceptionContext.invalidate_pool_on_disconnect`。允许`ConnectionEvents.handle_error()`中的错误处理程序维护“断开”条件，但在事件中以特定方式调用单个连接的无效。

    参考资料：[#3379](https://www.sqlalchemy.org/trac/ticket/3379)

+   **[engine] [feature]**

    添加了新事件`do_connect`，允许拦截/替换调用`Dialect.connect()`挂钩以创建 DBAPI 连接时。还添加了方言插件钩子`Dialect.get_dialect_cls()`和`Dialect.engine_created()`，它允许外部插件使用入口点向现有方言添加事件。

    参考：[#3355](https://www.sqlalchemy.org/trac/ticket/3355)

### sql

+   **[sql] [feature]**

    添加了一个占位符方法`TypeEngine.compare_against_backend()`，从 0.7.6 版本开始被 Alembic 迁移所使用。用户定义的类型可以实现此方法，以帮助比较数据库中反射的类型与另一个类型之间的比较。

+   **[sql] [bug]**

    修复了 SQL 中长标签截断可能产生与未截断的另一个标签重叠的错误；这是因为截断的长度阈值大于截断后保留的标签部分。现在，这两个值已经被设置为相同；标签长度 - 6。这里的效果是较短的列标签将在以前不会被截断的地方“截断”。

    参考：[#3396](https://www.sqlalchemy.org/trac/ticket/3396)

+   **[sql] [bug]**

    由于[#3282](https://www.sqlalchemy.org/trac/ticket/3282)导致的回归修复，将`tables`集合作为关键字参数传递给`DDLEvents.before_create()`、`DDLEvents.after_create()`、`DDLEvents.before_drop()`和`DDLEvents.after_drop()`事件时，不再是表的列表，而是包含第二个条目的元组列表，其中包含要添加或删除的外键。由于`tables`集合，虽然文档中说明不一定稳定，但已经被依赖，这种更改被认为是一个回归。此外，在某些情况下，“drop”操作会失败，因为此集合��能是一个迭代器，如果过早迭代会导致操作失败。现在，在所有情况下，该集合都是表对象的列表，并且现在已添加了对该集合格式的测试覆盖。

    参考：[#3391](https://www.sqlalchemy.org/trac/ticket/3391)

### 杂项

+   **[bug] [ext]**

    修复了在关联代理中的一个错误，即在关系->标量非对象属性比较上执行 any()/has()时会失败，例如`filter(Parent.some_collection_to_attribute.any(Child.attr == 'foo'))`

    参考：[#3397](https://www.sqlalchemy.org/trac/ticket/3397)

### orm

+   **[orm] [bug] [pypy]**

    由于[#3349](https://www.sqlalchemy.org/trac/ticket/3349)，在发布之前从 0.9.10 版本开始的回归中，`Query.update()`或`Query.delete()`上的查询状态检查将空元组与自身使用`is`进行比较，这在 PyPy 上会失败，导致在这种情况下产生`True`；这将在 0.9 中错误地发出警告，并在 1.0 中引发异常。

    参考：[#3405](https://www.sqlalchemy.org/trac/ticket/3405)

+   **[orm] [bug]**

    修复了在发布之前从 0.9.10 版本开始的回归，其中将`entity`添加到`Query.column_descriptions`访问器时，如果目标实体是从核心可选择对象（如`Table`或`CTE`对象）生成的，将会失败。

    参考：[#3320](https://www.sqlalchemy.org/trac/ticket/3320), [#3403](https://www.sqlalchemy.org/trac/ticket/3403)

+   **[orm] [bug]**

    修复了在刷新过程中的回归，当属性设置为 UPDATE 的 SQL 表达式时，与属性的先前值进行比较的 SQL 表达式会产生一个不是`==`或`!=`的 SQL 比较时，将引发异常“此子句的布尔值未定义”。修复确保工作单元不会以这种方式解释 SQL 表达式。

    引用：[#3402](https://www.sqlalchemy.org/trac/ticket/3402)

+   **[orm] [bug]**

    修复了由于[#2992](https://www.sqlalchemy.org/trac/ticket/2992)导致的意外使用回归，其中将文本元素放置到`Query.order_by()`子句中，与连接的急加载一起使用时，这些元素将被添加到内部查询的列子句中，以一种被假定为表绑定列名的方式，即在连接的急加载需要将查询包装在子查询中以适应限制/偏移量的情况下。

    最初，这里的行为是有意的，例如`query(User).order_by('name').limit(1)`这样的查询将按照`user.name`排序，即使查询被连接的急加载修改为在子查询中，因为`'name'`将被解释为一个符号，应该位于 FROM 子句中，在这种情况下是`User.name`，然后将其复制到列子句中以确保它存在于 ORDER BY 中。然而，该功能未能预料到`order_by("name")`指的是本地列子句中已经存在的特定标签名称，而不是绑定到 FROM 子句中的名称的情况。

    此外，该功能还对已弃用的情况失败，例如`order_by("name desc")`，虽然它会发出警告，应该在这里使用`text()`（请注意，该问题不影响显式使用`text()`的情况），但仍会产生与以前不同的查询，其中“name desc”表达式被不适当地复制到列子句中。解决方案是，该功能的“连接的急加载”方面将跳过这些所谓的“标签引用”表达式，当增强内部列子句时，就像它们已经是`text()`构造一样。

    引用：[#3392](https://www.sqlalchemy.org/trac/ticket/3392)

+   **[orm] [bug]**

    修复了关于`MapperEvents.instrument_class()`事件的回归，其中其调用被移动到类管理器对类进行仪器化之后，这与事件文档明确说明的相反。切换的理由是由于声明性在将类映射为新的`@declared_attr`功能描述的目的之前设置了完整的“仪器管理器”，但也针对经典使用`Mapper`的一致性进行了更改。然而，SQLSoup 依赖于在任何经典映射下的仪器化之前发生的仪器化事件。在经典和声明性映射的情况下，行为被恢复，后者通过使用简单的记忆化而不使用类管理器来实现。

    参考：[#3388](https://www.sqlalchemy.org/trac/ticket/3388)

+   **[ORM] [错误]**

    修复了新`QueryEvents.before_compile()`事件中对在事件中加载的实体集合进行更改的问题，这些更改会在 SQL 中呈现，但在加载过程中不会反映出来。

    参考：[#3387](https://www.sqlalchemy.org/trac/ticket/3387)

### 引擎

+   **[引擎] [特性]**

    添加了支持引擎/池插件具有高级功能的新功能。在已检出连接包装器以及`_ConnectionRecord`级别的连接池中添加了新的“软无效化”功能。这类似于现代池无效化，即连接不会被主动关闭，而只会在下次检出时被回收；这本质上是该功能的每个连接版本。添加了一个新事件`PoolEvents.soft_invalidate()`来补充它。

    还添加了新标志`ExceptionContext.invalidate_pool_on_disconnect`。允许`ConnectionEvents.handle_error()`中的错误处理程序维护“断开”条件，但在事件中以特定方式处理对各个连接的无效化调用。

    参考：[#3379](https://www.sqlalchemy.org/trac/ticket/3379)

+   **[引擎] [特性]**

    添加了新的事件`do_connect`，允许拦截/替换`Dialect.connect()` 被调用以创建 DBAPI 连接的钩子。还添加了方言插件钩子`Dialect.get_dialect_cls()` 和 `Dialect.engine_created()` ，允许外部插件使用入口点向现有方言添加事件。

    参考：[#3355](https://www.sqlalchemy.org/trac/ticket/3355)

### sql

+   **[sql] [feature]**

    添加了一个占位方法`TypeEngine.compare_against_backend()`，自 0.7.6 版开始由 Alembic 迁移使用。用户定义的类型可以实现此方法，以协助比较类型与从数据库反射的类型之间的差异。

+   **[sql] [bug]**

    修复了 SQL 中长标签截断可能产生重叠另一个未截断标签的错误。这是因为截断的长度阈值大于截断后剩余标签的部分。现在这两个值已经变成相同的；label_length - 6。这里的效果是，较短的列标签将在以前不会被截断的地方“截断”。

    参考：[#3396](https://www.sqlalchemy.org/trac/ticket/3396)

+   **[sql] [bug]**

    由于 [#3282](https://www.sqlalchemy.org/trac/ticket/3282) 导致的回归错误已修复，其中将 `tables` 集合作为关键字参数传递给`DDLEvents.before_create()`、`DDLEvents.after_create()`、`DDLEvents.before_drop()` 和 `DDLEvents.after_drop()` 事件时，不再是表的列表，而是包含第二个条目的元组列表，其中包含要添加或删除的外键。由于 `tables` 集合，虽然文档中声明不一定稳定，但已被依赖，这种变化被认为是一个回归。此外，对于“drop”，在某些情况下，此集合将是一个迭代器，如果过早迭代将导致操作失败。现在，在所有情况下，集合都是表对象的列表，并且现在已添加了此集合格式的测试覆盖。

    参考：[#3391](https://www.sqlalchemy.org/trac/ticket/3391)

### 杂项

+   **[bug] [ext]**

    修复了关于关联代理中的 bug，其中对关系->标量非对象属性比较的 any()/has()会失败，例如`filter(Parent.some_collection_to_attribute.any(Child.attr == 'foo'))`

    参考：[#3397](https://www.sqlalchemy.org/trac/ticket/3397)

## 1.0.2

发布日期：2015 年 4 月 24 日

### orm declarative

+   **[orm] [declarative] [bug]**

    修复了关于声明性`__declare_first__`和`__declare_last__`访问器的意外使用回归，其中这些访问器将不再在声明性基类的超类上调用。

    参考：[#3383](https://www.sqlalchemy.org/trac/ticket/3383)

### sql

+   **[sql] [bug]**

    修复了一个回归，该回归在 1.0.0b4 中被错误地修复（因此成为两个回归）；报告称 SELECT 语句将对标签名称进行 GROUP BY 并失败，被误解为某些后端（如 SQL Server）根本不应该在简单标签名称上发出 ORDER BY 或 GROUP BY；事实上，我们忘记了 0.9 版本已经对所有后端的简单标签名称进行 ORDER BY，如 Label constructs can now render as their name alone in an ORDER BY 中所述，即使 1.0 包括对此逻辑的重写作为[#2992](https://www.sqlalchemy.org/trac/ticket/2992)的一部分。关于对简单标签进行 GROUP BY，即使是 PostgreSQL 也有可能出现错误，即使应该明显要对其进行分组，因此很明显不应该自动以这种方式呈现 GROUP BY。

    在 1.0.2 版本中，SQL Server、Firebird 等再次在传递`Label`构造时对简单标签名称发出 ORDER BY，该构造也存在于列子句中。此外，当传递`Label`构造时，任何后端都不会仅对简单标签名称发出 GROUP BY。

    参考：[#3338](https://www.sqlalchemy.org/trac/ticket/3338), [#3385](https://www.sqlalchemy.org/trac/ticket/3385)

### orm declarative

+   **[orm] [declarative] [bug]**

    修复了关于声明性`__declare_first__`和`__declare_last__`访问器的意外使用回归，其中这些访问器将不再在声明性基类的超类上调用。

    参考：[#3383](https://www.sqlalchemy.org/trac/ticket/3383)

### sql

+   **[sql] [bug]**

    修复了一个在 1.0.0b4 中错误修复的回归（因此成为两个回归）；报告称 SELECT 语句将按标签名称进行 GROUP BY 并失败被误解为某些后端（如 SQL Server）根本不应该在简单标签名称上发出 ORDER BY 或 GROUP BY；事实上，我们忘记了 0.9 版本已经在所有后端上为简单标签名称发出 ORDER BY，如标签构造现在可以仅按其名称在 ORDER BY 中呈现中所述，尽管 1.0 版本包括对此逻辑的重写作为[#2992](https://www.sqlalchemy.org/trac/ticket/2992)的一部分。至于针对简单标签发出 GROUP BY，即使 PostgreSQL 也有可能会引发错误，尽管应该明显可以分组的标签，因此很明显不应该自动以这种方式呈现 GROUP BY。

    在 1.0.2 中，SQL Server、Firebird 和其他后端将再次在传递到`Label`构造的简单标签名称时发出 ORDER BY。此外，当传递到`Label`构造时，任何后端都不会仅针对简单标签名称发出 GROUP BY。

    参考：[#3338](https://www.sqlalchemy.org/trac/ticket/3338)，[#3385](https://www.sqlalchemy.org/trac/ticket/3385)

## 1.0.1

发布日期：2015 年 4 月 23 日

### orm

+   **[orm] [bug]**

    修复了一个问题，即形式为`query(B).filter(B.a != A(id=7))`的查询将在给定瞬态对象时呈现`NEVER_SET`符号。对于持久对象，它将始终使用持久化的数据库值而不是当前设置的值。假设自动刷新已打开，对于持久值，这通常对于持久值不会明显，因为任何待处理的更改都将首先刷新。但是，这与非否定比较`query(B).filter(B.a == A(id=7))`使用当前值的逻辑不一致，并且还允许与瞬态对象进行比较。比较现在使用当前值而不是数据库持久化的值。

    与此版本中由[#3061](https://www.sqlalchemy.org/trac/ticket/3061)引起的其他`NEVER_SET`问题不同，这个特定问题至少从 0.8 版本开始存在，可能更早，但是作为修复相关的`NEVER_SET`问题的结果被发现。

    另请参见

    “否定包含或等于”关系比较将使用属性的当前值，而不是数据库的值。

    参考：[#3374](https://www.sqlalchemy.org/trac/ticket/3374)

+   **[orm] [bug]**

    修复了由[#3061](https://www.sqlalchemy.org/trac/ticket/3061)引起的意外使用回归，其中 NEVER_SET 符号可能会泄漏到基于关系的查询中，包括`filter()`和`with_parent()`查询。在所有情况下都返回`None`符号，但是许多这些查询从未得到正确支持，并且在没有使用 IS 运算符的情况下产生对 NULL 的比较。出于这个原因，还向那些当前不提供`IS NULL`的关系查询子集添加了警告。

    参见

    比较对象与 None 值时发出的警告到关系

    参考：[#3371](https://www.sqlalchemy.org/trac/ticket/3371)

+   **[ORM] [错误]**

    修复了由[#3061](https://www.sqlalchemy.org/trac/ticket/3061)引起的回归，其中 NEVER_SET 符号可能会泄漏到延迟加载查询中，在挂起对象的刷新后发生。这通常会发生在不使用简单“get”策略的多对一关系中。好消息是，该修复改善了与 0.9 相比的效率，因为我们现在可以在检测到参数中存在 NEVER_SET 符号时完全跳过 SELECT 语句；在[#3061](https://www.sqlalchemy.org/trac/ticket/3061)之前，我们无法确定这里的 None 是否已设置。

    参考：[#3368](https://www.sqlalchemy.org/trac/ticket/3368)

### 引擎

+   **[引擎] [错误]**

    将字符串值`"none"`添加到`Pool.reset_on_return`参数接受的字符串值之一，作为`None`的同义词，以便所有设置都可以使用字符串值，从而允许像`engine_from_config()`这样的工具可以无问题地使用。

    这个改变也被**回溯**到了：0.9.10

    参考：[#3375](https://www.sqlalchemy.org/trac/ticket/3375)

### sql

+   **[SQL] [错误]**

    修复了直接 SELECT EXISTS 查询无法将正确的布尔结果类型分配给结果映射的问题，而是将查询中的列类型泄漏到结果映射中。这个问题在 0.9 版本和更早版本中也存在，但在那些版本中的影响较小。在 1.0 中，由于[#918](https://www.sqlalchemy.org/trac/ticket/918)，这成为了一个回归，因为我们现在依赖于结果映射非常准确，否则我们可能会将结果类型处理器分配给错误的列。在所有版本中，这个问题还会导致简单的 EXISTS 不应用布尔类型处理程序，导致在没有本地布尔值的后端中使用简单的 1/0 值而不是 True/False。修复包括 EXISTS 列参数将像其他列表达式一样匿名标记；类似的修复也针对纯布尔表达式如`not_(True())`实现。

    参考：[#3372](https://www.sqlalchemy.org/trac/ticket/3372)

### sqlite

+   **[sqlite] [错误]**

    修复了由于[#3282](https://www.sqlalchemy.org/trac/ticket/3282)导致的回归，由于我们在创建/删除模式时尝试假设 ALTER 可用，因此在 SQLite 的情况下，我们简单地说根本不用担心外键，因为 ALTER 不可用，当创建和删除表时，这意味着在 SQLite 的情况下基本上跳过了表的排序，对于绝大多数 SQLite 用例，这不是问题。

    然而，对于在 SQLite 上执行包含数据的表的 DROP 操作并且打开了引用完整性的用户，然后会遇到错误，因为在 DROP 时依赖排序确实很重要，当这些表包含数据时（SQLite 仍然可以让您创建对不存在表的外键并删除引用具有启用约束的现有表，只要没有引用数据）。

    为了在仍允许 SQLite DROP 操作保持排序的同时保持[#3282](https://www.sqlalchemy.org/trac/ticket/3282)的新功能，我们现在考虑完整的 FK 进行排序，如果遇到无法解决的循环，那么我们才放弃尝试对表进行排序；我们发出警告并使用未排序列表。如果环境需要有序的 DROP 操作并且存在外键循环，那么警告指出他们需要将`use_alter`标志恢复到他们的`ForeignKey`和`ForeignKeyConstraint`对象中，以便仅跳过这些对象的依赖排序。

    另请参阅

    ForeignKeyConstraint 上的 use_alter 标志（通常）不再需要 - 包含有关 SQLite 的更新说明。

    参考：[#3378](https://www.sqlalchemy.org/trac/ticket/3378)

### 杂项

+   **[错误] [firebird]**

    修复了由于[#3034](https://www.sqlalchemy.org/trac/ticket/3034)导致的回归，Firebird 方言未正确解释 limit/offset 子句。感谢 effem-git 的拉取请求。

    参考：[#3380](https://www.sqlalchemy.org/trac/ticket/3380)

+   **[错误] [firebird]**

    修复了在使用 Firebird 时使用 limit/offset 时“literal_binds”模式的支持，因此当选择此模式时，值再次以内联方式呈现。相关于[#3034](https://www.sqlalchemy.org/trac/ticket/3034)。

    参考：[#3381](https://www.sqlalchemy.org/trac/ticket/3381)

### orm

+   **[orm] [错误]**

    修复了一种形式的查询`query(B).filter(B.a != A(id=7))`在给定临时对象时会生成`NEVER_SET`符号的问题。对于持久化对象，它总是使用持久化的数据库值而不是当前设置的值。假设自动刷新已打开，这通常对于持久值不会明显，因为任何待处理的更改都会被首先刷新。但是，这与非否定比较使用的逻辑不一致，`query(B).filter(B.a == A(id=7))`，它确实使用当前值，并且还允许与临时对象进行比较。比较现在使用当前值而不是数据库持久值。

    与此版本中由[#3061](https://www.sqlalchemy.org/trac/ticket/3061)引起的回归修复的其他`NEVER_SET`问题不同，这个特定问题至少存在于 0.8 版本，并且可能更早，但是它是在修复相关的`NEVER_SET`问题时被发现的。

    另请参阅

    “否定包含或等于”关系比较将使用属性的当前值，而不是数据库的值

    引用：[#3374](https://www.sqlalchemy.org/trac/ticket/3374)

+   **[orm] [bug]**

    修复了由[#3061](https://www.sqlalchemy.org/trac/ticket/3061)引起的意外使用回归，其中 NEVER_SET 符号可能会渗漏到基于关系的查询中，包括`filter()`和`with_parent()`查询。在所有情况下都返回`None`符号，但是其中许多查询在任何情况下都从未正确支持，并且产生对 NULL 的比较而不使用 IS 运算符。因此，还在那些当前不提供`IS NULL`的关系查询子集中添加了警告。

    另请参阅

    当将对象与空值比较时发出警告

    引用：[#3371](https://www.sqlalchemy.org/trac/ticket/3371)

+   **[orm] [bug]**

    修复了由[#3061](https://www.sqlalchemy.org/trac/ticket/3061)引起的回归，其中 NEVER_SET 符号可能会渗漏到延迟加载查询中，在挂起对象刷新后发生。这通常会发生在不使用简单的“get”策略的一对多关系中。好消息是，该修复提高了效率，因为我们现在可以在检测到参数中存在 NEVER_SET 符号时完全跳过 SELECT 语句；在[#3061](https://www.sqlalchemy.org/trac/ticket/3061)之前，我们无法判断这里的 None 是否已设置。

    引用：[#3368](https://www.sqlalchemy.org/trac/ticket/3368)

### engine

+   **[engine] [bug]**

    将字符串值`"none"`添加到`Pool.reset_on_return`参数接受的值中，作为`None`的同义词，这样字符串值就可以用于所有设置，允许像`engine_from_config()`这样的实用程序无问题地可用。

    此更改也被**回移**到：0.9.10

    参考：[#3375](https://www.sqlalchemy.org/trac/ticket/3375)

### sql

+   **[sql] [bug]**

    修复了直接的 SELECT EXISTS 查询无法将 Boolean 的正确结果类型分配给结果映射的问题，而是会将查询内部的列类型泄漏到结果映射中。这个问题在 0.9 及更早版本中也存在，但在那些版本中影响较小。在 1.0 中，由于[#918](https://www.sqlalchemy.org/trac/ticket/918)，这成为了一个回归，因为我们现在依赖于结果映射非常准确，否则我们可能会将结果类型处理器分配给错误的列。在所有版本中，这个问题还有一个影响，即简单的 EXISTS 不会应用 Boolean 类型处理程序，导致没有本地布尔值的后端的简单的 1/0 值而不是 True/False。修复包括 EXISTS 列参数将被匿名标记，就像其他列表达式一样；对纯布尔表达式（如 `not_(True())`）也实现了类似的修复。

    参考：[#3372](https://www.sqlalchemy.org/trac/ticket/3372)

### sqlite

+   **[sqlite] [bug]**

    修复了由于[#3282](https://www.sqlalchemy.org/trac/ticket/3282)导致的一个回归，由于我们尝试假设在创建/删除模式时 ALTER 可用，因此在 SQLite 的情况下，我们简单地说根本不必担心外键，因为在创建和删除表时 ALTER 不可用。这意味着在 SQLite 的情况下基本上跳过了表的排序，在绝大多数 SQLite 使用情况下，这不是一个问题。

    但是，对于启用了强制约束的带有数据的表在 SQLite 上执行 DROP 的用户将会遇到错误，因为在 DROP 时依赖排序确实很重要，当这些表具有数据时（SQLite 仍然乐意让您创建对不存在表的外键并删除引用已存在表的具有启用约束的表，只要没有引用数据）。

    为了保持 [#3282](https://www.sqlalchemy.org/trac/ticket/3282) 的新功能，同时仍允许 SQLite 的 DROP 操作保持排序，我们现在考虑完整的外键进行排序，如果遇到无法解决的循环，*那么*我们放弃尝试对表进行排序；我们会发出警告并使用未排序的列表。如果一个环境既需要有序的 DROP 操作 *又* 存在外键循环，那么警告指出他们需要将 `ForeignKey` 和 `ForeignKeyConstraint` 对象的 `use_alter` 标志恢复，以便只有这些对象会被排除在依赖排序之外。

    另请参阅

    ForeignKeyConstraint 上的 use_alter 标志（通常）不再需要 - 包含有关 SQLite 的更新说明。

    参考：[#3378](https://www.sqlalchemy.org/trac/ticket/3378)

### 杂项

+   **[bug] [firebird]**

    修复了由于 [#3034](https://www.sqlalchemy.org/trac/ticket/3034) 导致的回归，导致 Firebird 方言未正确解释 limit/offset 子句。感谢 effem-git 提交的拉取请求。

    参考：[#3380](https://www.sqlalchemy.org/trac/ticket/3380)

+   **[bug] [firebird]**

    修复了在使用 Firebird 时在“literal_binds”模式下使用 limit/offset 时的支持，因此当选择此模式时，值再次以内联方式呈现。与 [#3034](https://www.sqlalchemy.org/trac/ticket/3034) 相关。

    参考：[#3381](https://www.sqlalchemy.org/trac/ticket/3381)

## 1.0.0

发布日期：2015 年 4 月 16 日

### orm

+   **[orm] [feature]**

    添加了新参数 `Query.update.update_args`，允许传递 kw 参数，如 `mysql_limit` 到底层的 `Update` 构造。感谢 Amir Sadoughi 提交的拉取请求。

+   **[orm] [bug]**

    发现了处理多次对相同目标进行 `Query.join()` 时的不一致性；它仅在关系连接的情况下隐式去重，并且由于 [#3233](https://www.sqlalchemy.org/trac/ticket/3233)，在 1.0 中，对同一张表进行两次连接的行为与 0.9 不同，不再错误地别名。为了帮助记录这一变化，迁移说明中关于 [#3233](https://www.sqlalchemy.org/trac/ticket/3233) 的措辞已经泛化，并且在多次针对相同目标关系调用 `Query.join()` 时添加了警告。

    参考：[#3367](https://www.sqlalchemy.org/trac/ticket/3367)

+   **[orm] [bug]**

    在确定半自引用关系的远程端时，对关系的启发式进行了小改进（例如，两个连接的继承子类相互引用），非简单的连接条件会考虑父实体并可以减少使用`remote()`注释的需要；这可以恢复一些在 0.9.4 之前可能在没有注释的情况下工作的情况，参见[#2948](https://www.sqlalchemy.org/trac/ticket/2948)。

    参考：[#3364](https://www.sqlalchemy.org/trac/ticket/3364)

### sql

+   **[sql] [feature]**

    用于对`Table`对象进行排序的拓扑排序，并通过`MetaData.sorted_tables`集合提供，现在将产生**确定性**的排序；也就是说，给定一组具有特定名称和依赖关系的表，每次都会产生相同的排序。这有助于比较 DDL 脚本和其他用例。表按名称发送到拓扑排序，拓扑排序本身将以有序的方式处理传入的数据。感谢 Sebastian Bank 的拉取请求。

    另请参阅

    MetaData.sorted_tables 访问器是“确定性”的

    参考：[#3084](https://www.sqlalchemy.org/trac/ticket/3084)

+   **[sql] [bug]**

    修复了一个问题，即使用命名约定的`MetaData`对象在 pickle 时无法正常工作。属性被跳过，导致不一致性和失败，如果反序列化的`MetaData`对象用于基于其他表的附加表。

    此更改也**回溯**到：0.9.10

    参考：[#3362](https://www.sqlalchemy.org/trac/ticket/3362)

### postgresql

+   **[postgresql] [bug]**

    修复了一个长期存在的 bug，即在 psycopg2 方言中与非 ASCII 值和`native_enum=False`一起使用`Enum`类型时，无法正确解码返回结果。这源自于 PG `ENUM` 类型曾经是一个独立的类型，没有“非本地”选项。

    此更改也**回溯**到：0.9.10

    参考：[#3354](https://www.sqlalchemy.org/trac/ticket/3354)

### mssql

+   **[mssql] [bug]**

    修复了一个回归问题，即“最后插入的 id”机制在 MSSQL 上执行 INSERT 时会失败，如果主键值在执行之前的插入参数中存在，以及在从 SELECT 进行的 INSERT 中，目标列被声明为列对象而不是字符串键时，会存储不正确的值。

    参考：[#3360](https://www.sqlalchemy.org/trac/ticket/3360)

+   **[mssql] [bug]**

    现在使用 pymssql 中的`Binary`构造函数，而不是打补丁。感谢 Ramiro Morales 的拉取请求。

### 测试

+   **[测试] [错误]**

    修复了测试运行时使用的路径；对于 sqla_nose.py 和 py.test，如果未设置 sys.flags.no_user_site，则再次在 sys.path 的开头插入“./lib”前缀；这使其的行为就像 Python 默认情况下将“.”放在当前路径一样。对于 tox，我们现在设置了 PYTHONNOUSERSITE 标志。

    参考：[#3356](https://www.sqlalchemy.org/trac/ticket/3356)

### ORM

+   **[ORM] [功能]**

    添加了新参数`Query.update.update_args`，允许传递诸如`mysql_limit`之类的关键字参数到底层的`Update`构造。感谢 Amir Sadoughi 的拉取请求。

+   **[ORM] [错误]**

    发现处理多次对同一目标进行`Query.join()`时存在不一致性；它仅在关系连接的情况下隐式去重，并且由于[#3233](https://www.sqlalchemy.org/trac/ticket/3233)，在 1.0 中两次连接到同一表的行为与 0.9 不同，不再错误地别名。为了帮助记录这一变化，在迁移说明中关于[#3233](https://www.sqlalchemy.org/trac/ticket/3233)的措辞已经被概括，并且在多次针对同一目标关系调用`Query.join()`时添加了警告。

    参考：[#3367](https://www.sqlalchemy.org/trac/ticket/3367)

+   **[ORM] [错误]**

    在确定半自引用关系的远程端时，改进了关系的启发式算法，考虑到父实体并且可以减少使用`remote()`注释的需要；这可以恢复一些在 0.9.4 之前可能在没有注释的情况下工作的情况，通过[#2948](https://www.sqlalchemy.org/trac/ticket/2948)。

    参考：[#3364](https://www.sqlalchemy.org/trac/ticket/3364)

### SQL

+   **[SQL] [功能]**

    用于对 `Table` 对象进行排序的拓扑排序，并通过 `MetaData.sorted_tables` 集合提供，现在将产生一个**确定性**排序；也就是说，给定一组具有特定名称和依赖关系的表，每次都会产生相同的顺序。这有助于比较 DDL 脚本和其他用例。表按名称发送到拓扑排序，拓扑排序本身将以有序的方式处理传入的数据。感谢 Sebastian Bank 的拉取请求。

    另请参阅

    MetaData.sorted_tables accessor is “deterministic”

    参考：[#3084](https://www.sqlalchemy.org/trac/ticket/3084)

+   **[sql] [bug]**

    修复了使用命名约定的 `MetaData` 对象无法正确与 pickle 配合使用的问题。属性被跳过，导致如果从未拾取的 `MetaData` 对象派生其他表，则会出现不一致和失败。

    此更改也被**回溯**到：0.9.10

    参考：[#3362](https://www.sqlalchemy.org/trac/ticket/3362)

### postgresql

+   **[postgresql] [bug]**

    修复了一个长期存在的 bug，即在与 psycopg2 方言一起使用时，`Enum` 类型与非 ASCII 值和 `native_enum=False` 结合使用时无法正确解码返回结果。这源于 PG `ENUM` 类型曾经是一个独立的类型，没有“非本地”选项。

    此更改也被**回溯**到：0.9.10

    参考：[#3354](https://www.sqlalchemy.org/trac/ticket/3354)

### mssql

+   **[mssql] [bug]**

    修复了一个回归，即“最后插入的 id”机制在 MSSQL 上在插入时未能存储正确的值，当主键值在执行之前出现在插入参数中时，以及在从 SELECT 插入时，目标列被声明为列对象而不是字符串键的情况下。

    参考：[#3360](https://www.sqlalchemy.org/trac/ticket/3360)

+   **[mssql] [bug]**

    现在使用 pymssql 中现有的 `Binary` 构造函数，而不是打补丁。感谢 Ramiro Morales 的拉取请求。

### tests

+   **[tests] [bug]**

    修复了测试运行时使用的路径；对于 sqla_nose.py 和 py.test，再次在 sys.path 的开头插入“./lib”前缀，但仅当 sys.flags.no_user_site 未设置时；这使其表现得就像 Python 默认将“.”放在当前路径中一样。对于 tox，我们现在设置了 PYTHONNOUSERSITE 标志。

    参考：[#3356](https://www.sqlalchemy.org/trac/ticket/3356)

## 1.0.0b5

发布日期：2015 年 4 月 3 日

### orm

+   **[orm] [bug]**

    修复了一个 bug，在多个嵌套的`Session.begin_nested()`操作中，状态跟踪在内部保存点中更新的对象的“dirty”标志未能传播，因此，如果回滚封闭保存点，则对象将不会成为过期状态的一部分，因此将恢复为其数据库状态。

    此更改也**回溯**到：0.9.10

    参考：[#3352](https://www.sqlalchemy.org/trac/ticket/3352)

+   **[orm] [bug]**

    当使用`Query.update()`或`Query.delete()`方法时，`Query`不支持连接、子选择或特殊的 FROM 子句；如果调用了像`Query.join()`或`Query.select_from()`这样的方法，而不是在静默中忽略这些字段，将引发错误。在 0.9.10 中，这只会发出警告。

    参考：[#3349](https://www.sqlalchemy.org/trac/ticket/3349)

+   **[orm] [bug]**

    在会话的提交阶段中，对一个弱字典进行了 list()调用，如果没有它，可能会在进程中与垃圾回收交互时引发“dictionary changed size during iter”错误。此更改由＃3139 引入。

+   **[orm] [bug]**

    修复了与“嵌套”内连接贪婪加载相关的 bug，在 0.9 中也存在，但在 1.0 中更像是一个回归，因为[#3008](https://www.sqlalchemy.org/trac/ticket/3008)默认打开了“嵌套”，这样，一个跨越共同祖先的兄弟路径的连接贪婪加载将正确地将每个“innerjoin”兄弟拼接到连接的适当部分，当一系列内部/外部连接混合在一起时。 

    参考：[#3347](https://www.sqlalchemy.org/trac/ticket/3347)

### sql

+   **[sql] [bug]**

    对于非 Unicode 类型的 Unicode 类型发出的警告已经放宽，以便对不是字符串值的值发出警告，例如整数；以前，1.0 的更新警告系统使用了字符串格式化操作，这将引发内部 TypeError。虽然这些情况理想情况下应该完全引发错误，但一些后端，如 SQLite 和 MySQL 确实接受它们，并且可能被传统代码使用，更不用说如果目标后端关闭了 Unicode 转换，它们将始终通过。

    参考：[#3346](https://www.sqlalchemy.org/trac/ticket/3346)

### postgresql

+   **[postgresql] [bug]**

    修复了一个 bug，其中更新的 PG 索引反映结果 [#3184](https://www.sqlalchemy.org/trac/ticket/3184) 会导致 PostgreSQL 版本 8.4 及更早版本的索引操作失败。当使用旧版本的 PostgreSQL 时，现在已禁用了这些增强功能。

    参考：[#3343](https://www.sqlalchemy.org/trac/ticket/3343)

### orm

+   **[orm] [bug]**

    修复了一个 bug，其中在多个嵌套的 `Session.begin_nested()` 操作内的状态跟踪将无法传播对象的“脏”标志，该对象已在内部保存点中更新，因此，如果回滚了封闭的保存点，则该对象将不会成为已过期状态的一部分，因此将恢复到其数据库状态。

    此更改也被**回溯**到：0.9.10

    参考：[#3352](https://www.sqlalchemy.org/trac/ticket/3352)

+   **[orm] [bug]**

    `Query` 在使用 `Query.update()` 或 `Query.delete()` 方法时不支持连接、子查询或特殊的 FROM 子句；如果已调用 `Query.join()` 或 `Query.select_from()` 等方法，而不是在这些字段上静默地忽略这些字段，则会引发错误。在 0.9.10 中，这只会发出警告。

    参考：[#3349](https://www.sqlalchemy.org/trac/ticket/3349)

+   **[orm] [bug]**

    在会话的提交阶段中的弱字典周围添加了一个 list() 调用，如果垃圾收集与进程交互，而没有它，则可能会导致“迭代期间字典大小已更改”错误。该更改由 #3139 引入。

+   **[orm] [bug]**

    修复了与“嵌套”内连接急切加载相关的 bug，该 bug 在 0.9 中也存在，但由于 [#3008](https://www.sqlalchemy.org/trac/ticket/3008) 而在 1.0 中更是一种退化，默认情况下启用“嵌套”，因此，当使用 innerjoin=True 的连接急切加载从共同祖先跨越兄弟路径时，将每个“innerjoin”兄弟正确地拼接到连接的适当部分中，当一系列的内/外连接混合在一起时。

    参考：[#3347](https://www.sqlalchemy.org/trac/ticket/3347)

### sql

+   **[sql] [bug]**

    对于非 Unicode 类型的 unicode 类型发出的警告已放宽，以警告不是字符串值的值，例如整数；以前，1.0 的更新警告系统使用了字符串格式化操作，这会引发内部 TypeError。虽然这些情况理想情况下应完全引发，但一些后端（如 SQLite 和 MySQL）确实接受它们，并且可能在遗留代码中使用，更不用说如果目标后端关闭了 unicode 转换，它们将始终通过。 

    参考：[#3346](https://www.sqlalchemy.org/trac/ticket/3346)

### postgresql

+   **[postgresql] [bug]**

    修复了一个问题，即更新了 [#3184](https://www.sqlalchemy.org/trac/ticket/3184) 导致的 PG 索引反射会导致在 PostgreSQL 8.4 及更早版本上索引操作失败的 bug。当使用较旧版本的 PostgreSQL 时，现在已禁用了这些增强功能。

    参考：[#3343](https://www.sqlalchemy.org/trac/ticket/3343)

## 1.0.0b4

发布日期：2015 年 3 月 29 日

### sql

+   **[sql] [bug]**

    修复了 [#2992](https://www.sqlalchemy.org/trac/ticket/2992) 中新的“标签解析”功能中的 bug，其中一个匿名标签，然后再次用名称标记时，无法通过文本标签定位。当在查询中为映射的 `column_property()` 显式地指定一个标签时，自然会发生这种情况。

    参考：[#3340](https://www.sqlalchemy.org/trac/ticket/3340)

+   **[sql] [bug]**

    修复了 [#2992](https://www.sqlalchemy.org/trac/ticket/2992) 中新的“标签解析”功能的 bug，其中在语句的 order_by() 或 group_by() 中放置的字符串标签会更优先于在 FROM 子句内找到的名称，而不是更本地可用的列子句内的名称。

    参考：[#3335](https://www.sqlalchemy.org/trac/ticket/3335)

### 架构

+   **[schema] [feature]**

    对约束的“自动附加”功能（如 `UniqueConstraint` 和 `CheckConstraint`）进行了进一步增强，使得当约束与非绑定到表的 `Column` 对象相关联时，约束会为列本身设置事件监听器，以便在将列与表相关联时约束自动附加。这在声明式中特别有用，但也是普遍适用的。

    另请参阅

    当引用未连接的列时，约束可以在其引用的列附加到表时自动附加

    参考：[#3341](https://www.sqlalchemy.org/trac/ticket/3341)

### mysql

+   **[mysql] [bug] [pymysql]**

    修复了 PyMySQL 在使用“executemany”操作时对 Unicode 支持的问题，当使用 Unicode 参数时，SQLAlchemy 现在将语句和绑定参数都作为 Unicode 对象传递，因为 PyMySQL 通常在内部使用字符串插值来生成最终语句，并且在 executemany 情况下仅对最终语句执行“encode”步骤。

    这个更改也被**回溯**到：0.9.10

    参考：[#3337](https://www.sqlalchemy.org/trac/ticket/3337)

### mssql

+   **[mssql] [bug] [firebird] [oracle] [sybase]**

    关闭了 MSSQL、Oracle 方言上的“简单排序”标志；这是根据[#2992](https://www.sqlalchemy.org/trac/ticket/2992)的规定，导致在表达式中有一个在列子句中也被引用的 order by 或 group by 时，即使被引用为表达式对象，也会被复制为标签。对于 MSSQL，现在的行为是默认情况下复制整个表达式，因为 MSSQL 在这些情况下可能会对 GROUP BY 表达式特别挑剔。该标志也被防御性地关闭了 Firebird 和 Sybase 方言。

    注意

    这个解决方案是错误的，请查看版本 1.0.2 以重新制定这个解决方案。

    参考：[#3338](https://www.sqlalchemy.org/trac/ticket/3338)

### sql

+   **[sql] [bug]**

    修复了[#2992](https://www.sqlalchemy.org/trac/ticket/2992)中新“标签解析”功能中的错误，其中一个匿名标签，然后再次使用名称标记，将无法通过文本标签定位。当在查询中为映射的`column_property()`给定一个显式标签时，这种情况自然发生。

    参考：[#3340](https://www.sqlalchemy.org/trac/ticket/3340)

+   **[sql] [bug]**

    修复了[#2992](https://www.sqlalchemy.org/trac/ticket/2992)中新“标签解析”功能中的错误，其中在语句的 order_by()或 group_by()中放置的字符串标签会更优先考虑在 FROM 子句中找到的名称，而不是在列子句中更容易找到的名称。

    参考：[#3335](https://www.sqlalchemy.org/trac/ticket/3335)

### schema

+   **[schema] [feature]**

    对于`UniqueConstraint`和`CheckConstraint`等约束的“自动附加”功能已进一步增强，以便当约束与非绑定到表的`Column`对象相关联时，约束将与列本身设置事件侦听器，以便约束在与表关联的同时自动附加。这特别有助于一些边缘情况下的声明式，但也是一般用途。

    另请参阅

    当引用的列被附加时，引用未附加的列的约束可以自动附加到表

    参考：[#3341](https://www.sqlalchemy.org/trac/ticket/3341)

### mysql

+   **[mysql] [bug] [pymysql]**

    修复了 PyMySQL 在使用带有 unicode 参数的“executemany”操作时的 unicode 支持。现在 SQLAlchemy 现在将语句和绑定参数都作为 unicode 对象传递，因为 PyMySQL 通常在内部使用字符串插值来生成最终语句，并且在 executemany 的情况下仅在最终语句上执行“encode”步骤。

    这个更改也被**回溯**到：0.9.10

    参考：[#3337](https://www.sqlalchemy.org/trac/ticket/3337)

### mssql

+   **[mssql] [bug] [firebird] [oracle] [sybase]**

    在 MSSQL、Oracle 方言上关闭了“简单排序”标志；这个标志根据[#2992](https://www.sqlalchemy.org/trac/ticket/2992)导致在表达式也在列子句中的 order by 或 group by 被复制为标签，即使被引用为表达式对象。对于 MSSQL，现在的行为是默认情况下复制整个表达式，因为 MSSQL 在 GROUP BY 表达式中可能会挑剔。该标志也被防御性地关闭了 Firebird 和 Sybase 方言。

    注意

    这个解决方案是不正确的，请查看版本 1.0.2 进行重新处理。

    参考：[#3338](https://www.sqlalchemy.org/trac/ticket/3338)

## 1.0.0b3

发布日期：2015 年 3 月 20 日

### mysql

+   **[mysql] [bug]**

    修复了无意中被注释掉的问题 #2771 的提交。

    参考：[#2771](https://www.sqlalchemy.org/trac/ticket/2771)

### mysql

+   **[mysql] [bug]**

    修复了无意中被注释掉的问题 #2771 的提交。

    参考：[#2771](https://www.sqlalchemy.org/trac/ticket/2771)

## 1.0.0b2

发布日期：2015 年 3 月 20 日

### orm

+   **[orm] [bug]**

    修复了从 pullreq github:137 中出现的意外使用回归，其中 Py2K unicode 文字（例如`u""`）不会被`relationship.cascade`选项接受。感谢 Julien Castets 提交的拉取请求。

    参考：[#3327](https://www.sqlalchemy.org/trac/ticket/3327)

### orm declarative

+   **[orm] [declarative] [change]**

    放宽了对 `@declared_attr` 对象的一些限制，使得它们可以在声明过程之外被调用；这与 #3150 的增强相关，允许 `@declared_attr` 返回一个基于当前类的缓存值，因为它正在被配置。异常抛出已被移除，行为已更改，使得在声明过程之外，由 `@declared_attr` 装饰的函数每次都像普通的 `@property` 一样被调用，而不使用任何缓存，因为在这个阶段没有可用的缓存。

    参考：[#3331](https://www.sqlalchemy.org/trac/ticket/3331)

### engine

+   **[engine] [bug]**

    `ResultProxy`的“自动关闭”现在是“软”关闭。也就是说，在使用提取方法耗尽所有行后，DBAPI 游标会像以前一样被释放，对象可以安全丢弃，但是提取方法仍然可以被调用，它们将返回一个结果结束对象（对于 fetchone 为 None，对于 fetchmany 和 fetchall 为空列表）。只有显式调用`ResultProxy.close()`，这些方法才会引发“结果已关闭”错误。

    另请参阅

    `ResultProxy`“自动关闭”现在是“软”关闭

    参考：[#3329](https://www.sqlalchemy.org/trac/ticket/3329)，[#3330](https://www.sqlalchemy.org/trac/ticket/3330)

### mysql

+   **[mysql] [bug] [py3k]**

    修复了 Py3K 上未正确使用`ord()`函数的`BIT`类型。感谢 David Marin 的拉取请求。

    此更改也**回溯**到：0.9.10

    参考：[#3333](https://www.sqlalchemy.org/trac/ticket/3333)

+   **[mysql] [bug]**

    修复了完全支持使用`'utf8mb4'` MySQL 特定字符集与 MySQL 方言，特别是 MySQL-Python 和 PyMySQL。此外，报告更不寻常字符集如‘koi8u’或‘eucjpms’的 MySQL 数据库也将正常工作。感谢 Thomas Grainger 的拉取请求。

    参考：[#2771](https://www.sqlalchemy.org/trac/ticket/2771)

### orm

+   **[orm] [bug]**

    修复了从 pullreq github:137 中的意外使用回归，其中 Py2K unicode 文字（例如`u""`）将不被`relationship.cascade`选项接受。感谢 Julien Castets 的拉取请求。

    参考：[#3327](https://www.sqlalchemy.org/trac/ticket/3327)

### orm declarative

+   **[orm] [declarative] [change]**

    放宽了对`@declared_attr`对象添加的一些限制，使其可以在声明过程之外被调用；这与增强功能#3150 有关，该功能允许`@declared_attr`返回一个根据当前类缓存的值，因为它正在配置。已删除异常抛出，并更改了行为，使得在声明过程之外，由`@declared_attr`装饰的函数每次都像常规的`@property`一样被调用，而不使用任何缓存，因为在这个阶段没有可用的缓存。

    参考：[#3331](https://www.sqlalchemy.org/trac/ticket/3331)

### 引擎

+   **[engine] [bug]**

    `ResultProxy`的“自动关闭”现在是“软”关闭。也就是说，在使用提取方法耗尽所有行后，DBAPI 游标会像以前一样被释放，对象可以安全丢弃，但是提取方法仍然可以被调用，它们将返回一个结果结束对象（对于 fetchone 为 None，对于 fetchmany 和 fetchall 为空列表）。只有显式调用`ResultProxy.close()`，这些方法才会引发“结果已关闭”错误。

    另请参阅

    ResultProxy “自动关闭”现在是“软”关闭

    参考：[#3329](https://www.sqlalchemy.org/trac/ticket/3329), [#3330](https://www.sqlalchemy.org/trac/ticket/3330)

### mysql

+   **[mysql] [错误] [py3k]**

    修复了在 Py3K 上未正确使用 `ord()` 函数的 `BIT` 类型。感谢 David Marin 提交的拉取请求。

    此更改也被**回溯**到：0.9.10

    参考：[#3333](https://www.sqlalchemy.org/trac/ticket/3333)

+   **[mysql] [错误]**

    修复了完全支持使用 `'utf8mb4'` 这种 MySQL 特定字符集的 MySQL 方言，特别是 MySQL-Python 和 PyMySQL。此外，报告更不寻常字符集的 MySQL 数据库，如 ‘koi8u’ 或 ‘eucjpms’ 也将正常工作。感谢 Thomas Grainger 提交的拉取请求。

    参考：[#2771](https://www.sqlalchemy.org/trac/ticket/2771)

## 1.0.0b1

发布日期：2015 年 3 月 13 日

版本 1.0.0b1 是 1.0 系列的第一个发布版本。这里描述的许多更改也存在于 0.9 系列，有时也存在于 0.8 系列。对于特定于 1.0 且强调兼容性问题的更改，请参阅 SQLAlchemy 1.0 中的新功能是什么？。

### 通用

+   **[通用] [功能]**

    通过更多地使用 `__slots__` 来改进结构内存使用，特别针对具有大量表和列的大型应用程序的基本内存大小进行了优化，并大大减少了各种高容量对象的内存大小，包括事件监听内部、比较器对象以及 ORM 属性和加载器策略系统的部分。

    另请参阅

    结构内存使用方面的显著改进

+   **[通用] [错误]**

    现在为所有那些作为“公共工厂”符号派生的 SQL 和 ORM 函数设置了 `__module__` 属性，这应有助于文档工具能够报告目标模块。

    参考：[#3218](https://www.sqlalchemy.org/trac/ticket/3218)

### orm

+   **[orm] [功能]**

    向 `Query.column_descriptions` 返回的字典中添加了一个新条目 `"entity"`。这指的是由表达式引用的主 ORM 映射类或别名类。与现有的 `"type"` 条目相比，它将始终是一个映射实体，即使从列表达式中提取，或者如果给定表达式是纯核心表达式，则为 None。另请参阅修复了此功能中的回归的 [#3403](https://www.sqlalchemy.org/trac/ticket/3403)，该功能未在 0.9.10 中发布，但在 1.0 版本中发布。

    此更改也被**回溯**到：0.9.10

    参考：[#3320](https://www.sqlalchemy.org/trac/ticket/3320)

+   **[orm] [功能]**

    添加了新参数`Session.connection.execution_options`，可用于在首次检出连接之前设置`Connection`上的执行选项，事务开始之前。这用于在事务开始之前在连接上设置隔离级别等选项。

    另请参阅

    设置事务隔离级别 / DBAPI AUTOCOMMIT - 新文档部分详细介绍了使用会话设置事务隔离的最佳实践。

    此更改也**回溯**到：0.9.9

    参考：[#3296](https://www.sqlalchemy.org/trac/ticket/3296)

+   **[orm] [功能]**

    添加了新方法`Session.invalidate()`，功能类似于`Session.close()`，但还调用所有连接的`Connection.invalidate()`，确保它们不会返回到连接池。在处理 gevent 超时等情况时非常有用，此时进一步使用连接是不安全的，即使是用于回滚。

    此更改也**回溯**到：0.9.9

+   **[orm] [功能]**

    “primaryjoin”模型已经进一步扩展，允许一个严格从单个列到自身的连接条件，通过某种 SQL 函数或表达式进行转换。这有点实验性，但第一个概念验证是一个“材料化路径”连接条件，其中一个路径字符串与自身使用“like”进行比较。`ColumnOperators.like()`操作符也已添加到可在 primaryjoin 条件中使用的有效操作符列表中。

    此更改也**回溯**到：0.9.5

    参考：[#3029](https://www.sqlalchemy.org/trac/ticket/3029)

+   **[orm] [功能]**

    添加了新的实用函数`make_transient_to_detached()`，可用于制造行为就像从会话加载然后分离的对象。不存在的属性被标记为过期，并且对象可以添加到一个会话中，它将表现得像一个持久对象。

    此更改也**回溯**到：0.9.5

    参考：[#3017](https://www.sqlalchemy.org/trac/ticket/3017)

+   **[orm] [功能]**

    添加了一个新的事件套件`QueryEvents`。`QueryEvents.before_compile()`事件允许创建函数，在构建 SELECT 语句之前对`Query`对象进行额外修改。希望通过引入一个新的检查系统，使这个事件变得更加有用，该系统将允许对`Query`对象进行详细的自动修改。

    另请参阅

    `QueryEvents`

    参考：[#3317](https://www.sqlalchemy.org/trac/ticket/3317)

+   **[orm] [功能]**

    当使用连接预加载与具有 LIMIT、OFFSET 或 DISTINCT 的一对多查询时，如果一对一关系也被禁用，即一对多关系中`relationship.uselist`设置为 False，将禁用子查询包装。这将在这些情况下产生更有效的查询。

    另请参阅

    子查询不再适用于 uselist=False 的连接预加载

    参考：[#3249](https://www.sqlalchemy.org/trac/ticket/3249)

+   **[orm] [功能]**

    映射状态内部已经重新设计，以允许将对象的“过期”（如`Session.commit()`和`Session.expire_all()`中的“自动过期”功能）以及在对象状态被垃圾回收时发生的“清理”步骤的调用次数减少 50%。

    参考：[#3307](https://www.sqlalchemy.org/trac/ticket/3307)

+   **[orm] [功能]**

    当将相同的多态标识分配给同一层次结构中的两个不同的映射器时，会发出警告。这通常是用户错误，意味着在加载时无法正确区分两种不同的映射类型。感谢 Sebastian Bank 的拉取请求。

    参考：[#3262](https://www.sqlalchemy.org/trac/ticket/3262)

+   **[orm] [功能]**

    创建了一系列新的`Session`方法，直接提供钩子到工作单元的功能，用于发出 INSERT 和 UPDATE 语句。当正确使用时，这种面向专家的系统可以允许使用 ORM 映射生成批量插入和更新语句，批量分组执行语句，使语句的速度与直接使用 Core 相媲美。

    另请参阅

    批量操作

    参考：[#3100](https://www.sqlalchemy.org/trac/ticket/3100)

+   **[orm] [feature]**

    添加了一个参数`Query.join.isouter`，它与调用`Query.outerjoin()`是同义的；此标志旨在提供与 Core `FromClause.join()`更一致的接口。感谢 Jonathan Vanasco 提供的拉取请求。

    参考：[#3217](https://www.sqlalchemy.org/trac/ticket/3217)

+   **[orm] [feature]**

    添加了新的事件处理程序`AttributeEvents.init_collection()`和`AttributeEvents.dispose_collection()`，用于跟踪集合首次与实例关联以及替换集合的情况。这些处理程序取代了`collection.linker()`注释。旧的钩子通过事件适配器仍然受支持。

+   **[orm] [feature]**

    当`Query`与映射或选项一起使用`Query.yield_per()`时，将引发异常，其中子查询急加载或与集合一起的连接急加载将发生。这些加载策略目前与 yield_per 不兼容，因此通过引发此错误，该方法更安全。可以使用`lazyload('*')`选项或`Query.enable_eagerloads()`来禁用急加载。

    另请参阅

    使用 yield_per 明确禁止连接/子查询急加载

+   **[orm] [feature]**

    `Query`对象使用的`KeyedTuple`的新实现在获取大量面向列的行时提供了显著的速度改进。

    另请参阅

    新的 KeyedTuple 实现速度显著提升

    参考：[#3176](https://www.sqlalchemy.org/trac/ticket/3176)

+   **[orm] [feature]**

    当内连接的 eager load 链接到外连接的 eager load 时，`joinedload.innerjoin` 和 `relationship.innerjoin` 的行为现在默认使用“嵌套”内连接，即右嵌套。

    另请参阅

    右内连接嵌套现在是 joinedload 的默认设置，innerjoin=True

    参考：[#3008](https://www.sqlalchemy.org/trac/ticket/3008)

+   **[orm] [功能]**

    现在可以将 UPDATE 语句批量处理到更高效的 executemany() 调用中，类似于 INSERT 语句的批量处理；这将在 flush 中被调用，前提是后续的 UPDATE 语句针对相同的映射和表涉及相同的列在 VALUES 子句中，没有嵌入 SET 级别的 SQL 表达式，并且映射的版本要求与后端方言能够返回 executemany 操作的正确行数的兼容性。

+   **[orm] [功能]**

    `info` 参数已添加到 `SynonymProperty` 和 `ComparableProperty` 的构造函数中。

    参考：[#2963](https://www.sqlalchemy.org/trac/ticket/2963)

+   **[orm] [功能]**

    `InspectionAttr.info` 集合现在移动到 `InspectionAttr`，除了在所有 `MapperProperty` 对象上可用外，现在还可以在混合属性、关联代理上使用，通过 `Mapper.all_orm_descriptors` 访问。

    参考：[#2971](https://www.sqlalchemy.org/trac/ticket/2971)

+   **[orm] [变更]**

    如果映射属性标记为延迟加载而没有明确取消延迟加载，则即使它们的列以某种方式出现在结果集中，它们现在仍将保持“延迟加载”。这是一个性能增强，因为 ORM 加载不再花费时间搜索每个延迟加载列，当结果集被获取时。然而，对于一直依赖于此的应用程序，现在应该使用显式的 `undefer()` 或类似选项。

+   **[orm] [更改]**

    传递给自定义 `Bundle` 类的 `create_row_processor()` 方法的 `proc()` 可调用现在只接受一个“row”参数。

    另请参阅

    当使用自定义行加载程序时，新 Bundle 功能的 API 更改

+   **[orm] [更改]**

    已弃用的事件钩子已移除：`populate_instance`、`create_instance`、`translate_row`、`append_result`

    另请参阅

    已弃用的 ORM 事件钩子已移除

+   **[orm] [bug]**

    修复了子查询急加载中的 bug，其中在跨多态子类边界的一长串急加载中，与多态加载一起会无法找到链中的子类链接，导致在 `AliasedClass` 上出现缺少属性名称的错误。

    此更改也已**回溯**到：0.9.5、0.8.7

    参考：[#3055](https://www.sqlalchemy.org/trac/ticket/3055)

+   **[orm] [bug]**

    修复了 ORM 中的 bug，`class_mapper()` 函数会掩盖应在映射器配置期间引发的 AttributeErrors 或 KeyErrors，因为用户错误。捕获属性/键错误的方式更具体，不包括配置步骤。

    此更改也已**回溯**到：0.9.5、0.8.7

    参考：[#3047](https://www.sqlalchemy.org/trac/ticket/3047)

+   **[orm] [bug]**

    修复了 ORM 对象比较中的 bug，其中对于多对一的 `!= None` 比较，如果源是一个别名类，或者查询需要对表达式应用特殊别名，因为别名连接或多态查询，比较多对一与对象状态会失败；还修复了比较多对一与对象状态时的 bug，如果查询需要应用特殊别名，因为别名连接或多态查询。

    此更改也已**回溯**到：0.9.9

    参考：[#3310](https://www.sqlalchemy.org/trac/ticket/3310)

+   **[orm] [bug]**

    修复了一个 bug，在 `after_rollback()` 处理程序为 `Session` 错误地在处理程序内向该 `Session` 添加状态时，内部断言会失败，并且尝试警告和移除此状态的任务（由 [#2389](https://www.sqlalchemy.org/trac/ticket/2389) 确定）会继续进行。

    此更改也已**回溯**到：0.9.9

    参考：[#3309](https://www.sqlalchemy.org/trac/ticket/3309)

+   **[orm] [bug]**

    修复了一个 bug，当调用 `Query.join()` 时，带有未知关键字参数会由于格式错误而引发自己的 TypeError 错误。感谢 Malthe Borch 提交的拉取请求。

    此更改也已**回溯**到：0.9.9

+   **[orm] [bug]**

    修复了懒加载 SQL 构造中的 bug，其中一个复杂的 primaryjoin 在“指向自身的列”的自引用连接风格中多次引用相同的“本地”列时，在所有情况下都不会被替换。这里重新设计了确定替换的逻辑，使其更加开放。

    此更改也**回溯**到：0.9.9

    参考：[#3300](https://www.sqlalchemy.org/trac/ticket/3300)

+   **[orm] [bug]**

    “通配符”加载器选项，特别是由`load_only()`选项设置的选项，以覆盖未明确提及的所有属性，现在考虑到给定实体的超类，如果该实体使用继承映射进行映射，则超类中的属性名称也将从加载中省略。此外，多态鉴别器列无条件地包含在列表中，就像主键列一样，因此即使设置了 load_only()，子类型的多态加载仍将正常工作。

    此更改也**回溯**到：0.9.9

    参考：[#3287](https://www.sqlalchemy.org/trac/ticket/3287)

+   **[orm] [bug] [pypy]**

    修复了一个错误，即如果在`Query`在获取结果之前抛出异常，特别是当无法形成行处理器时，游标将保持打开状态，结果仍在等待中，实际上并未关闭。这通常只在像 PyPy 这样的解释器上出现问题，其中游标不会立即被 GC 回收，并且在某些情况下可能导致事务/锁定打开时间过长。

    此更改也**回溯**到：0.9.9

    参考：[#3285](https://www.sqlalchemy.org/trac/ticket/3285)

+   **[orm] [bug]**

    修复了一个泄漏问题，该问题会在不支持且极不推荐的情况下多次替换固定映射类上的关系时发生，涉及任意增长数量的目标映射器。在替换旧关系时会发出警告，但如果映射已用于查询，则旧关系仍将在某些注册表中被引用。

    此更改也**回溯**到：0.9.9

    参考：[#3251](https://www.sqlalchemy.org/trac/ticket/3251)

+   **[orm] [bug] [sqlite]**

    修复了关于表达式变异的错误，当使用`Query`从多个匿名列实体中选择时，在针对 SQLite 进行查询时，可能会表现为“找不到列”错误，这是 SQLite 方言使用的“联接重写”功能的副作用。

    此更改也**回溯**到：0.9.9

    参考：[#3241](https://www.sqlalchemy.org/trac/ticket/3241)

+   **[orm] [bug]**

    修复了一个错误，即对于使用`of_type()`连接到单继承子类的`Query.join()`和`Query.outerjoin()`的 ON 子句，如果设置了`from_joinpoint=True`标志，则不会在 ON 子句中呈现“单表条件”。

    此更改也已**回溯**至：0.9.9

    参考：[#3232](https://www.sqlalchemy.org/trac/ticket/3232)

+   **[orm] [bug] [engine]**

    修复了一个 bug，影响了与[#3199](https://www.sqlalchemy.org/trac/ticket/3199)相同类别的事件，当使用`named=True`参数时会出现问题。一些事件将无法注册，而其他事件将无法正确调用事件参数，通常在事件被“包装”以适应其他方式时。已重新排列“named”机制，以不干扰内部包装函数所期望的参数签名。

    此更改也已**回溯**至：0.9.8

    参考：[#3197](https://www.sqlalchemy.org/trac/ticket/3197)

+   **[orm] [bug]**

    修复了一个 bug，影响了许多类别的事件，特别是 ORM 事件，但也包括引擎事件，其中“去重复”冗余调用`listen()`的通常逻辑失败，对于那些监听函数被包装的事件。在 registry.py 中会触发一个断言。现在，这个断言已经整合到去重复检查中，额外的好处是更简单地检查整体的去重复。

    此更改也已**回溯**至：0.9.8

    参考：[#3199](https://www.sqlalchemy.org/trac/ticket/3199)

+   **[orm] [bug]**

    修复了一个警告，当一个复杂的自引用的主连接包含函数时，同时指定了`remote_side`时会发出警告；警告会建议设置“remote side”。现在只有在`remote_side`不存在时才会发出警告。

    此更改也已**回溯**至：0.9.8

    参考：[#3194](https://www.sqlalchemy.org/trac/ticket/3194)

+   **[orm] [bug] [eagerloading]**

    修复了一个由[#2976](https://www.sqlalchemy.org/trac/ticket/2976)引起的回归，发布于 0.9.4，其中沿着一系列连接的连接贪婪加载的“外连接”传播会错误地将一个“内连接”沿着兄弟连接路径转换为外连接，当只有后代路径应该接收“外连接”传播时；此外，修复了相关问题，即“嵌套”连接传播会不适当地发生在两个兄弟连接路径之间。

    此更改也已**回溯**至：0.9.7

    参考：[#3131](https://www.sqlalchemy.org/trac/ticket/3131)

+   **[orm] [bug]**

    由于[#2736](https://www.sqlalchemy.org/trac/ticket/2736)导致从 0.9.0 中的一个回归，`Query.select_from()`方法不再正确设置`Query`对象的“from entity”，因此随后的`Query.filter_by()`或`Query.join()`调用将无法在按字符串名称搜索属性时检查适当的“from”实体。

    此更改也**回溯**到：0.9.7

    参考：[#2736](https://www.sqlalchemy.org/trac/ticket/2736)，[#3083](https://www.sqlalchemy.org/trac/ticket/3083)

+   **[orm] [bug]**

    修复了在保存点块内持久化、删除或主键更改的项目在外部事务回滚后不参与恢复到其先前状态（不在会话中、在会话中、先前 PK）的错误。

    此更改也**回溯**到：0.9.7

    参考：[#3108](https://www.sqlalchemy.org/trac/ticket/3108)

+   **[orm] [bug]**

    修复了与`with_polymorphic()`结合使用的子查询急加载中的错误，在子查询加载中对实体和列的定位相对于此类型的实体和其他实体更加准确。

    此更改也**回溯**到：0.9.7

    参考：[#3106](https://www.sqlalchemy.org/trac/ticket/3106)

+   **[orm] [bug]**

    已添加额外的检查，用于在继承映射器隐式组合其基于列的属性与父级的属性之一时，其中这些列通常不一定共享相同的值。这是通过[#1892](https://www.sqlalchemy.org/trac/ticket/1892)添加的现有检查的扩展；但是，这个新检查只发出警告，而不是异常，以允许依赖于现有行为的应用程序。

    另请参阅

    我收到关于“隐式组合列 X 在属性 Y 下”的警告或错误

    此更改也**回溯**到：0.9.5

    参考：[#3042](https://www.sqlalchemy.org/trac/ticket/3042)

+   **[orm] [bug]**

    修改了`load_only()`的行为，使得主键列始终被添加到“未延迟加载”列的列表中；否则，ORM 无法加载行的标识。显然，可以推迟映射的主键，ORM 将失败，这一点没有改变。但是，由于 load_only 本质上是在说“除了 X 之外都推迟”，因此 PK 列不参与此推迟更为关键。

    此更改也**回溯**到：0.9.5

    参考：[#3080](https://www.sqlalchemy.org/trac/ticket/3080)

+   **[orm] [bug]**

    修复了在所谓的“行切换”场景中出现的一些边缘情况，其中 INSERT/DELETE 可以转换为 UPDATE。在这种情况下，将多对一关系设置为 None，或在某些情况下将标量属性设置为 None，可能不会被检测为值的净变化，因此 UPDATE 不会重置前一行上的内容。这是由于属性历史记录的一些尚未解决的副作用导致的，隐式地假定 None 对于先前未设置的属性实际上不是一个“变化”。另请参见[#3061](https://www.sqlalchemy.org/trac/ticket/3061)。

    注意

    此更改已在 0.9.6 中**撤销**。完整的修复将在 SQLAlchemy 的 1.0 版本中实现。

    此更改也**回溯**到：0.9.5

    参考：[#3060](https://www.sqlalchemy.org/trac/ticket/3060)

+   **[orm] [bug]**

    与[#3060](https://www.sqlalchemy.org/trac/ticket/3060)相关，对工作单元进行了调整，以便在要删除的自引用对象图中，加载相关的多对一对象更加积极，以帮助确定删除顺序，如果未设置`passive_deletes`。

    此更改也**回溯**到：0.9.5

+   **[orm] [bug]**

    修复了 SQLite 连接重写中的错误，由于重复导致的匿名列名不会在子查询中正确重写。这会影响带有任何类型的子查询 + 连接的 SELECT 查询。

    此更改也**回溯**到：0.9.5

    参考：[#3057](https://www.sqlalchemy.org/trac/ticket/3057)

+   **[orm] [bug] [sql]**

    修复了在[#2804](https://www.sqlalchemy.org/trac/ticket/2804)中对新增增强的布尔强制转换规则的修复，其中“where”和“having”的新规则不会对`select()`构造函数的“whereclause”和“having”关键字参数生效，而`Query`也使用了这个构造函数，因此在 ORM 中也无法正常工作。

    此更改也**回溯**到：0.9.5

    参考：[#3013](https://www.sqlalchemy.org/trac/ticket/3013)

+   **[orm] [bug]**

    修复了会话附加错误“对象已附加到会话 X”无法阻止对象在错误引发后继续执行的情况下也附加到新会话的错误。

    参考：[#3301](https://www.sqlalchemy.org/trac/ticket/3301)

+   **[orm] [bug]**

    当调用`Query.count()`、`Query.update()`、`Query.delete()`以及针对映射列、`column_property`对象以及从映射列派生的 SQL 函数和表达式的查询时，现在`Query`的主要`Mapper`将传递给`Session.get_bind()`方法。这使得依赖于自定义`Session.get_bind()`方案或“绑定”元数据的会话在所有相关情况下都能正常工作。

    另请参阅

    Session.get_bind()将在所有相关的查询情况下接收到 Mapper

    参考：[#1326](https://www.sqlalchemy.org/trac/ticket/1326), [#3227](https://www.sqlalchemy.org/trac/ticket/3227), [#3242](https://www.sqlalchemy.org/trac/ticket/3242)

+   **[orm] [bug]**

    `PropComparator.of_type()`修饰符已经与加载指令（如`joinedload()`和`contains_eager()`）一起改进，以便如果遇到两个相同基本类型/路径的`PropComparator.of_type()`修饰符，它们将被合并为单个“多态”实体，而不是用类型 B 的实体替换类型 A 的实体。例如，`A.b.of_type(BSub1)->BSub1.c`的 joinedload 与`A.b.of_type(BSub2)->BSub2.c`的 joinedload 将创建一个单个的 joinedload `A.b.of_type((BSub1, BSub2)) -> BSub1.c, BSub2.c`，而不需要在查询中显式使用`with_polymorphic`。

    另请参阅

    多态子类型的急加载 - 包含一个更新的示例，展示了新的格式。

    参考：[#3256](https://www.sqlalchemy.org/trac/ticket/3256)

+   **[orm] [bug]**

    修复了当`CascadeOptions`参数使用`copy.deepcopy()`调用时的支持，这种情况发生在`relationship()`与`copy.deepcopy()`一起使用时（这不是官方支持的用例）。感谢 duesenfranz 的拉取请求。

+   **[orm] [bug]**

    修复了`Session.expunge()`在对象经历了已刷新但未提交的删除操作后，无法完全分离给定对象的错误。这也会影响到`make_transient()`等相关操作。

    另请参阅

    session.expunge() 现在会完全分离已删除的对象

    参考：[#3139](https://www.sqlalchemy.org/trac/ticket/3139)

+   **[orm] [bug]**

    在多个关系最终将填充与另一个冲突的外键列的情况下，会发出警告，其中关系试图从不同源列复制值。这发生在将具有重叠列的复合外键映射到每个引用列不同的关系的情况下。一个新的文档部分说明了示例以及如何通过在每个关系上明确指定“外键”列来克服该问题。

    另请参阅

    重叠的外键

    参考：[#3230](https://www.sqlalchemy.org/trac/ticket/3230)

+   **[orm] [bug]**

    `Query.update()` 方法现在会将给定值字典中的字符串键名转换为正在更新的映射类的映射属性名称。以前，字符串名称直接被接受并传递给核心更新语句，没有任何方法可以解析到映射实体。`Query.update()` 的主题属性也支持同义词和混合属性。

    另请参阅

    query.update() 现在将字符串名称解析为映射的属性名称

    参考：[#3228](https://www.sqlalchemy.org/trac/ticket/3228)

+   **[orm] [bug]**

    改进了`Session`用于定位“绑定”（例如要使用的引擎）的机制，这些引擎可以与混合类、具体子类以及更广泛的表元数据（如联合继承表）关联。

    另请参阅

    Session.get_bind() 处理更广泛的继承场景

    参考：[#3035](https://www.sqlalchemy.org/trac/ticket/3035)

+   **[orm] [bug]**

    修复了单表继承中的一个 bug，其中包含了多次包含相同单一继承实体的连接链（通常应该引发错误）的情况，可能会在某些情况下，取决于从何处连接，隐式别名第二个单一继承实体的情况，生成一个“有效”的查询。但由于这种隐式别名并不是单表继承的意图，它并没有完全“有效”，并且非常具有误导性，因为它并不总是出现。

    另请参阅

    处理重复连接目标中的更改和修复

    参考：[#3233](https://www.sqlalchemy.org/trac/ticket/3233)

+   **[orm] [bug]**

    当使用`Query.join()`、`Query.outerjoin()`或独立的`join()` / `outerjoin()`函数连接到单一继承子类时，生成的 ON 子句现在将包括“单表条件”，即使 ON 子句是手动编写的；它现在使用 AND 将条件添加到 ON 子句中，方式与使用 relationship 或类似方法连接到单表目标时相同。

    这在功能和 bug 之间有些模糊。

    另请参阅

    单表继承条件已无条件地添加到所有 ON 子句中

    参考：[#3222](https://www.sqlalchemy.org/trac/ticket/3222)

+   **[orm] [bug]**

    对表达式标签的行为进行了重大改进，特别是在与具有自定义 SQL 表达式的 ColumnProperty 构造一起使用时，以及与首次在 0.9 中引入的“按标签排序”逻辑结合使用时。修复包括，现在`order_by(Entity.some_col_prop)`将即使 Entity 经过别名处理（通过继承渲染或通过使用`aliased()`构造）也将使用“按标签排序”规则；使用别名多次渲染相同的列属性（例如`query(Entity.some_prop, entity_alias.some_prop)`)将为实体的每次出现标记一个不同的标签，并且此外，“按标签排序”规则将适用于两者（例如`order_by(Entity.some_prop, entity_alias.some_prop)`）。在 0.9 中可能阻止“按标签排序”逻辑工作的其他问题，尤其是标签的状态可能会发生变化，以至于“按标签排序”会根据调用方式停止工作的问题已经修复。

    另请参阅

    ColumnProperty 构造与别名、order_by 更好地配合

    参考：[#3148](https://www.sqlalchemy.org/trac/ticket/3148), [#3188](https://www.sqlalchemy.org/trac/ticket/3188)

+   **[orm] [bug]**

    改变了使用`Query.from_self()`或其常见用户`Query.count()`时应用“单一继承标准”的方法。现在，将限制行数为特定类型的标准指示在内部子查询中，而不是外部子查询中，因此即使“type”列不在列子句中可用，我们也可以在“内部”查询中对其进行过滤。

    另请参阅

    使用 from_self()、count()时单表继承标准的更改

    参考：[#3177](https://www.sqlalchemy.org/trac/ticket/3177)

+   **[orm] [bug]**

    对懒加载机制进行了小的调整，以减少在极为罕见的情况下干扰`joinload()`的可能性，即对象指向自身；在这种情况下，对象在加载其属性时会引用自身，这可能会导致加载器之间的混乱。“对象指向自身”的用例并未得到完全支持，但修复也减少了一些开销，因此目前是测试的一部分。

    参考：[#3145](https://www.sqlalchemy.org/trac/ticket/3145)

+   **[orm] [bug]**

    已删除“复活”ORM 事件。自从 0.8 版本中删除了旧的“可变属性”系统后，此事件挂钩就没有任何目的了。

    参考：[#3171](https://www.sqlalchemy.org/trac/ticket/3171)

+   **[orm] [bug]**

    修复了一个 bug，当属性“set”事件或带有`@validates`的列在刷新过程中触发事件时，当这些列是“获取和填充”操作的目标时，例如自增主键、Python 端默认值或通过 RETURNING“急切”获取的服务器端默认值。

    参考：[#3167](https://www.sqlalchemy.org/trac/ticket/3167)

+   **[orm] [bug] [py3k]**

    从`Session.identity_map`暴露的`IdentityMap`现在在 Py3K 中为`items()`和`values()`返回列表。早期移植到 Py3K 时，这些返回迭代器，但从技术上讲应该是“可迭代视图”..目前，列表是可以的。

+   **[orm] [bug]**

    查询.update()/delete()的“评估器”不适用于多表更新，并且需要设置为 synchronize_session=False 或 synchronize_session=’fetch’；现在会引发异常，并显示更改同步设置的消息。这是从 0.9.7 开始发出的警告升级。

    参考：[#3117](https://www.sqlalchemy.org/trac/ticket/3117)

+   **[orm] [enhancement]**

    调整属性机制，关于何时通过首次访问隐式初始化值为 None；这一操作，一直导致属性的填充，但现在不再这样；返回 None 值，但底层属性不接收设置事件。这与集合的工作方式一致，并允许属性机制更一致地行为；特别是，获取一个没有值的属性不会压制事件，如果值实际上被设置为 None，则事件应该继续。

    请参见

    关于没有预先存在值的属性事件和其他操作的更改

    其中绑定参数根据编译时选项作为字符串内联呈现。此功能的工作由 Dobes Vandermeer 提供。

    > 请参见
    > 
    > Select/Query LIMIT / OFFSET 可以指定为任意 SQL 表达式。

    参考：[#3061](https://www.sqlalchemy.org/trac/ticket/3061)

### orm declarative

+   **[orm] [declarative] [feature]**

    `declared_attr` 构造在与 declarative 结合使用时具有新的改进行为和特性。装饰的函数在调用时将有权访问本地 mixin 上存在的最终列副本，并且将确保为每个映射类仅调用一次，返回的结果将被缓存。还添加了一个新的修饰符 `declared_attr.cascading`。

    请参见

    对 declarative mixins、@declared_attr 和相关特性的改进

    参考：[#3150](https://www.sqlalchemy.org/trac/ticket/3150)

+   **[orm] [declarative] [bug]**

    修复了在使用 `AbstractConcreteBase` 与声明 `__abstract__` 的子类时出现的“‘NoneType’ 对象没有 ‘concrete’ 属性”错误。

    此更改也**回溯**到：0.9.8

    参考：[#3185](https://www.sqlalchemy.org/trac/ticket/3185)

+   **[orm] [declarative] [bug]**

    修复了在声明继承层次结构中间使用 `__abstract__` mixin 会阻止属性和配置正确从基类传播到继承类的错误。

    参考：[#3219](https://www.sqlalchemy.org/trac/ticket/3219), [#3240](https://www.sqlalchemy.org/trac/ticket/3240)

+   **[orm] [declarative] [bug]**

    使用`declared_attr`在`AbstractConcreteBase`基类上建立的关系现在将自动配置在抽象基类映射上，除了像往常一样在后代具体类上设置。

    另请参阅

    改进声明性混合，@declared_attr 和相关功能

    参考：[#2670](https://www.sqlalchemy.org/trac/ticket/2670)

### 示例

+   **[示例] [特性]**

    添加了一个使用最新关系特性说明材料化路径的新示例。示例由 Jack Zhou 提供。

    这个更改也被**回溯**到：0.9.5

+   **[示例] [特性]**

    一个新的示例套件，专门提供了对 SQLAlchemy ORM 和 Core 以及 DBAPI 性能的详细研究，从多个角度进行。该套件在一个容器内运行，通过控制台输出以及通过 RunSnake 工具图形化显示内置的性能分析。

    另请参阅

    性能

+   **[示例] [错误]**

    更新了带有历史表的版本控制示例，使映射列重新映射以匹配列名以及列的分组；特别是，这允许在同名列连接继承场景中明确分组的列在历史映射中以相同方式映射，避免了 0.9 系列中关于此模式的警告，并允许属性键的相同视图。

    这个更改也被**回溯**到：0.9.9

+   **[示例] [错误]**

    修复了示例/generic_associations/discriminator_on_association.py 示例中的一个 bug，其中 AddressAssociation 的子类没有被映射为“单表继承”，导致在尝试进一步使用映射时出现问题。

    这个更改也被**回溯**到：0.9.9

### 引擎

+   **[引擎] [特性]**

    添加了用于查看事务隔离级别的新用户空间访问器；`Connection.get_isolation_level()`, `Connection.default_isolation_level`.

    这个更改也被**回溯**到：0.9.9

+   **[引擎] [特性]**

    添加了新的事件`ConnectionEvents.handle_error()`，这是`ConnectionEvents.dbapi_error()`的更全面和全面的替代品。

    这个更改也被**回溯**到：0.9.7

    参考：[#3076](https://www.sqlalchemy.org/trac/ticket/3076)

+   **[引擎] [特性]**

    可以发出新的警告样式，这些警告样式将“过滤”掉参数化字符串的最多 N 次出现。这允许参数化警告可以引用它们的参数，直到允许 Python 警告过滤器将它们压制，并防止 Python 的警告注册表内存无限增长。

    另请参阅

    Session.get_bind() 处理更多种类的继承情况

    参考：[#3178](https://www.sqlalchemy.org/trac/ticket/3178)

+   **[引擎] [错误]**

    修复了 `Connection` 和池中的错误，当使用 `Connection.execution_options()` 时，`Connection.invalidate()` 方法或由于数据库断开而使连接无效时，如果 `isolation_level` 参数已被使用；将在不再打开的连接上调用重置隔离级别的“finalizer”。

    此更改也已**回溯**至：0.9.9

    参考：[#3302](https://www.sqlalchemy.org/trac/ticket/3302)

+   **[引擎] [错误]**

    如果在进行 `Transaction` 时使用了 `isolation_level` 参数，将发出警告；DBAPI 和/或 SQLAlchemy 方言（如 psycopg2、MySQLdb）可能会隐式回滚或提交事务，或者在下一次事务中不更改设置，因此这绝不安全。

    此更改也已**回溯**至：0.9.9

    参考：[#3296](https://www.sqlalchemy.org/trac/ticket/3296)

+   **[引擎] [错误]**

    通过`create_engine.execution_options`或`Engine.update_execution_options()`传递给`Engine`的执行选项不会传递给用于在“第一次连接”事件中初始化方言的特殊`Connection`；方言通常会在此阶段执行自己的查询，并且当前可用的选项不应该应用于此处。特别是，“autocommit”选项导致在这种初始连接中尝试自动提交，这将由于`Connection`的非标准状态而导致 AttributeError。

    这个更改也被**回溯**到：0.9.8

    参考：[#3200](https://www.sqlalchemy.org/trac/ticket/3200)

+   **[引擎] [错误]**

    用于确定 INSERT 或 UPDATE 受影响列的字符串键现在在它们对“编译缓存”缓存键的贡献时进行排序。在以前，这些键没有确定性地排序，这意味着相同的语句可能会根据等效键多次被缓存，这既会在内存和性能方面造成损失。

    这个更改也被**回溯**到：0.9.8

    参考：[#3165](https://www.sqlalchemy.org/trac/ticket/3165)

+   **[引擎] [错误]**

    修复了一个 bug，当引擎首次连接并进行初始检查时发生 DBAPI 异常，并且异常不是断开连接异常，但是当我们尝试关闭光标时光标引发错误时，会发生 bug。在这种情况下，由于我们试图通过连接池记录光标关闭异常并失败，因为我们试图以不适合这种非常特定情况的方式访问池的记录器，真正的异常将被扼杀。

    这个更改也被**回溯**到：0.9.5

    参考：[#3063](https://www.sqlalchemy.org/trac/ticket/3063)

+   **[引擎] [错误]**

    修复了一些检测到的“双重失效”情况，其中连接失效可能发生在已经处于关键部分的情况下，比如连接关闭(); 最终，这些条件是由于[#2907](https://www.sqlalchemy.org/trac/ticket/2907)中的更改引起的，因为“返回时重置”功能调用 Connection/Transaction 来处理它，其中可能会被捕获“断开连接检测”。然而，最近在[#2985](https://www.sqlalchemy.org/trac/ticket/2985)中的更改可能使得这种情况更容易被视为“连接失效”操作更快，因为在 0.9.4 上更容易复现这个问题，而在 0.9.3 上不太容易。

    现在在发生任何可能发生 invalidate 的部分内添加了检查，以阻止对无效连接进行进一步的不允许的操作。这包括在引擎级别和池级别都有两个修复。虽然问题在高度并发的 gevent 情况下观察到，但理论上可以在任何发生连接关闭操作时发生断开连接的情况下发生。

    此更改也被**反向移植**至：0.9.5

    参考：[#3043](https://www.sqlalchemy.org/trac/ticket/3043)

+   **[引擎] [错误]**

    引擎级错误处理和包装例程现在将在所有引擎连接用例中生效，包括当用户自定义连接例程通过`create_engine.creator`参数使用时，以及当`Connection`在重新验证时遇到连接错误时。

    另请参阅

    DBAPI 异常包装和 handle_error()事件改进

    参考：[#3266](https://www.sqlalchemy.org/trac/ticket/3266)

+   **[引擎] [错误]**

    在事件监听器正在运行时删除（或添加）事件监听器，无论是从监听器内部还是从并发线程中，现在都会引发 RuntimeError，因为现在使用的集合是`collections.deque()`的实例，并且不支持在迭代时进行更改。以前，使用的是简单的 Python 列表，其中从事件内部删除将产生静默失败。

    参考：[#3163](https://www.sqlalchemy.org/trac/ticket/3163)

### sql

+   **[sql] [特性]**

    在一定程度上放宽了`Index`的约定，如果要手动将索引添加到表中，则可以将`text()`表达式指定为目标；如果索引要通过内联声明或通过`Table.append_constraint()`添加到表中，则索引不再需要存在表绑定列。

    此更改也被**反向移植**至：0.9.5

    参考：[#3028](https://www.sqlalchemy.org/trac/ticket/3028)

+   **[sql] [特性]**

    添加了新的标志`between.symmetric`，当设置为 True 时渲染“BETWEEN SYMMETRIC”。还添加了一个新的否定运算符“notbetween_op”，现在允许像`~col.between(x, y)`这样的表达式渲染为“col NOT BETWEEN x AND y”，而不是带括号的 NOT 字符串。

    此更改也被**反向移植**至：0.9.5

    参考：[#2990](https://www.sqlalchemy.org/trac/ticket/2990)

+   **[sql] [特性]**

    SQL 编译器现在生成预期列的映射，使其按位置与接收到的结果集匹配，而不是按名称。最初，这被视为一种处理返回具有难以预测名称的列的情况的方法，尽管在现代使用中，这个问题已经通过匿名标记得以解决。在这个版本中，该方法基本上通过减少每个结果的函数调用次数几十次，或者对于更大的结果列集合来说更多，来降低函数调用次数。如果编译的列集合与接收到的列存在大小上的任何差异，该方法仍然会退化为旧方法的现代版本，因此在部分或完全文本编译场景中，这些列表可能不会对齐时不会出现问题。

    参考：[#918](https://www.sqlalchemy.org/trac/ticket/918)

+   **[sql] [feature]**

    在 `DefaultClause` 中的字面值，当使用 `Column.server_default` 参数时调用时，现在将使用“内联”编译器进行呈现，以便它们按原样呈现，而不是作为绑定参数。

    另请参阅

    列服务器默认值现在呈现字面值

    参考：[#3087](https://www.sqlalchemy.org/trac/ticket/3087)

+   **[sql] [feature]**

    当传递给 SQL 表达式单元的对象无法解释为 SQL 片段时，报告表达式的类型；感谢 Ryan P. Kelly 提交的拉取请求。

+   **[sql] [feature]**

    向 `Table.tometadata()` 方法添加了一个新参数 `Table.tometadata.name`。与 `Table.tometadata.schema` 类似，此参数导致新复制的 `Table` 使用新名称而不是现有名称。这种功能的一个有趣之处在于，它可以将 `Table` 对象复制到*相同的* `MetaData` 目标中并使用新名称。感谢 n.d. parker 提交的拉取请求。

+   **[sql] [feature]**

    异常消息稍作调整。如果为 None，则不显示 SQL 语句和参数，减少与语句无关的错误消息的混淆。显示了 DBAPI 级别异常的完整模块和类名，明确表明这是一个包装的 DBAPI 异常。语句和参数本身被限定在括号内，以更好地将它们与错误消息和彼此隔离开来。

    参考：[#3172](https://www.sqlalchemy.org/trac/ticket/3172)

+   **[sql] [feature]**

    `Insert.from_select()` 现在包括 Python 和 SQL 表达式默认值，如果未指定；解除了非服务器列默认值不包括在 INSERT FROM SELECT 中的限制，并将这些表达式作为常量呈现到 SELECT 语句中。

    请参阅

    INSERT FROM SELECT 现在包括 Python 和 SQL 表达式默认值

+   **[sql] [feature]**

    当反射一个 `Table` 对象时，现在包括了 `UniqueConstraint` 构造，适用于这些数据库。为了以足够的准确性实现这一点，MySQL 和 PostgreSQL 现在包含了纠正索引和唯一约束重复的功能，当反射表、索引和约束时。对于 MySQL，实际上没有独立于“唯一索引”的“唯一约束”概念，因此对于这个后端，`UniqueConstraint` 仍然不会出现在反射的 `Table` 中。对于 PostgreSQL，用于检测 `pg_index` 中的索引的查询已经改进，以检查 `pg_constraint` 中的相同构造，并且隐式构建的唯一索引不包含在反射的 `Table` 中。

    在这两种情况下，`Inspector.get_indexes()` 和 `Inspector.get_unique_constraints()` 方法分别返回这两种构造，但在 PostgreSQL 的情况下包含一个新的标记 `duplicates_constraint`，在 MySQL 的情况下包含一个新的标记 `duplicates_index`，以指示检测到此条件时。感谢 Johannes Erdfelt 的拉取请求。

    请参阅

    UniqueConstraint 现在是表反射过程的一部分

    参考：[#3184](https://www.sqlalchemy.org/trac/ticket/3184)

+   **[sql] [feature]**

    添加了新方法`Select.with_statement_hint()`和 ORM 方法`Query.with_statement_hint()`，以支持不特定于表的语句级提示。

    参考：[#3206](https://www.sqlalchemy.org/trac/ticket/3206)

+   **[sql] [feature]**

    `info` 参数已添加为所有模式构造函数的构造参数，包括`MetaData`、`Index`、`ForeignKey`、`ForeignKeyConstraint`、`UniqueConstraint`、`PrimaryKeyConstraint`、`CheckConstraint`。

    参考：[#2963](https://www.sqlalchemy.org/trac/ticket/2963)

+   **[sql] [feature]**

    `Table.autoload_with`标志现在意味着`Table.autoload`应为`True`。感谢 Malik Diarra 的拉取请求。

    参考：[#3027](https://www.sqlalchemy.org/trac/ticket/3027)

+   **[sql] [feature]**

    `Select.limit()`和`Select.offset()`方法现在接受任何 SQL 表达式作为参数，而不仅仅是整数值。通常用于允许传递绑定参数，稍后可以用值替换，从而允许在 Python 端缓存 SQL 查询。这里的实现完全向后兼容现有的第三方方言，但那些实现特殊 LIMIT/OFFSET 系统的方言需要修改以利用新功能。Limit 和 offset 还支持“literal_binds”模式，

    参考：[#3034](https://www.sqlalchemy.org/trac/ticket/3034)

+   **[sql] [changed]**

    `column()`和`table()`构造现在可以从“from sqlalchemy”命名空间导入，就像每个其他 Core 构造一样。

+   **[sql] [changed]**

    当传递给`select()`的大多数构建器方法以及`Query`时，将字符串隐式转换为`text()`构造时，现在只发送纯字符串会发出警告。文本转换仍然正常进行。唯一不会发出警告的方法是“标签引用”方法，如 order_by()，group_by()；这些函数现在在编译时将尝试将单个字符串参数解析为可选择的列或标签表达式；如果找不到任何内容，则表达式仍然呈现，但您会再次收到警告。这里的理由是，从字符串到文本的隐式转换现在比以往更加意外，当用户将原始字符串传递给 Core / ORM 时，最好让用户提供更多方向。Core/ORM 教程已更新，以更深入地介绍文本的处理方式。

    参见

    将完整 SQL 片段强制转换为 text()时发出的警告

    参考：[#2992](https://www.sqlalchemy.org/trac/ticket/2992)

+   **[sql] [bug]**

    修复了`Enum`和其他`SchemaType`子类中的错误，直接将类型与`MetaData`关联会导致在`MetaData`上发出事件（如创建事件）时挂起。

    这个更改也被**回溯**到：0.9.7, 0.8.7

    参考：[#3124](https://www.sqlalchemy.org/trac/ticket/3124)

+   **[sql] [bug]**

    修复了自定义操作符加`TypeEngine.with_variant()`系统中的错误，当使用`TypeDecorator`与 variant 结合使用时，当使用比较运算符时会出现 MRO 错误。

    这个更改也被**回溯**到：0.9.7, 0.8.7

    参考：[#3102](https://www.sqlalchemy.org/trac/ticket/3102)

+   **[sql] [bug]**

    修复了在从 UNION 中选择时，INSERT..FROM SELECT 构造中的错误，会将 UNION 包装在一个匿名的子查询中。

    此更改也被**回溯**到：0.9.5, 0.8.7

    参考：[#3044](https://www.sqlalchemy.org/trac/ticket/3044)

+   **[sql] [bug]**

    修复了当应用空的`and_()`或`or_()`或其他空表达式时，`Table.update()`和`Table.delete()`会生成空的 WHERE 子句的错误。现在这与`select()`的行为一致。

    此更改也被**回溯**到：0.9.5, 0.8.7

    参考：[#3045](https://www.sqlalchemy.org/trac/ticket/3045)

+   **[sql] [bug]**

    在`Enum`的`__repr__()`输出中添加了`native_enum`标志，当与 Alembic autogenerate 一起使用时，这是非常重要的。感谢 Dimitris Theodorou 的拉取请求。

    此更改也被**回溯**到：0.9.9

+   **[sql] [bug]**

    修复了使用实现了也是`TypeDecorator`的类型的`TypeDecorator`会在针对使用此类型的对象使用任何类型的 SQL 比较表达式时，导致 Python 的“无法创建一致的方法解析顺序（MRO）”错误的错误。

    此更改也被**回溯**到：0.9.9

    参考：[#3278](https://www.sqlalchemy.org/trac/ticket/3278)

+   **[sql] [bug]**

    修复了在 INSERT 中嵌入 SELECT 时，通过值子句或作为“from select”时，来自两个语句的列共享相同名称时，污染由 RETURNING 子句产生的结果集中使用的列类型，导致在检索返回行时可能出现错误或错误适应。

    此更改也被**回溯**到：0.9.9

    参考：[#3248](https://www.sqlalchemy.org/trac/ticket/3248)

+   **[sql] [bug]**

    修复了 sql 包中的相当数量的 SQL 元素无法成功执行`__repr__()`的错误，由于缺少`description`属性，然后会在内部 AttributeError 再次调用`__repr__()`时引发递归溢出。

    此更改也被**回溯**到：0.9.8

    参考：[#3195](https://www.sqlalchemy.org/trac/ticket/3195)

+   **[sql] [bug]**

    调整表/索引反射，如果索引报告一个在表中找不到的列，则会发出警告并跳过该列。这可能发生在一些特殊的系统列情况下，如在 Oracle 中观察到的情况。

    此更改也**回溯**到：0.9.8

    参考：[#3180](https://www.sqlalchemy.org/trac/ticket/3180)

+   **[sql] [bug]**

    修复了 CTE 中的错误，当一个 CTE 引用语句中的另一个别名 CTE 时，`literal_binds`编译器参数不会始终正确传播。

    此更改也**回溯**到：0.9.8

    参考：[#3154](https://www.sqlalchemy.org/trac/ticket/3154)

+   **[sql] [bug]**

    修复了由[#3067](https://www.sqlalchemy.org/trac/ticket/3067)引起的 0.9.7 回归，与一个命名错误的单元测试一起导致所谓的“模式”类型如`Boolean`和`Enum`无法再被 pickle。

    此更改也**回溯**到：0.9.8

    参考：[#3067](https://www.sqlalchemy.org/trac/ticket/3067), [#3144](https://www.sqlalchemy.org/trac/ticket/3144)

+   **[sql] [bug]**

    修复了命名约定功能中的错误，其中使用包含`constraint_name`的检查约定会强制所有`Boolean`和`Enum`类型也需要名称，因为这些隐式创建约束，即使最终目标后端不需要生成约束，比如 PostgreSQL。这些特定约束的命名约定机制已经重新组织，使得命名确定在 DDL 编译时完成，而不是在约束/表构建时完成。

    此更改也**回溯**到：0.9.7

    参考：[#3067](https://www.sqlalchemy.org/trac/ticket/3067)

+   **[sql] [bug]**

    修复了公共表达式中的错误，其中当 CTE 以某种方式嵌套时，位置绑定参数可能以错误的最终顺序表示。

    此更改也**回溯**到：0.9.7

    参考：[#3090](https://www.sqlalchemy.org/trac/ticket/3090)

+   **[sql] [bug]**

    修复了多值`Insert`构造失败检查后续值条目的错误，超出第一个给定的字面 SQL 表达式。

    此更改也**回溯**到：0.9.7

    参考：[#3069](https://www.sqlalchemy.org/trac/ticket/3069)

+   **[sql] [bug]**

    为 Python 版本 < 2.6.5 的 dialect_kwargs 迭代添加了一个“str()”步骤，解决了“无 unicode 关键字参数”错误，因为这些参数在某些反射过程中作为关键字参数传递。

    此更改也**回溯**到：0.9.7

    参考：[#3123](https://www.sqlalchemy.org/trac/ticket/3123)

+   **[sql] [bug]**

    `TypeEngine.with_variant()`方法现在将接受一个类型类作为参数，该参数在内部转换为一个实例，使用其他构��（如`Column`）长期以来建立的相同约定。

    此更改也被**回溯**到：0.9.7

    参考：[#3122](https://www.sqlalchemy.org/trac/ticket/3122)

+   **[sql] [bug]**

    当表中的`Column`在显式的`PrimaryKeyConstraint`中被引用时，`Column.nullable`标志会被隐式设置为`False`。这种行为现在与`Column`本身的`Column.primary_key`标志设置为`True`时的行为相匹配，这是一个完全等效的情况。

    此更改也被**回溯**到：0.9.5

    参考：[#3023](https://www.sqlalchemy.org/trac/ticket/3023)

+   **[sql] [bug]**

    修复了`Operators.__and__()`、`Operators.__or__()`和`Operators.__invert__()`运算符重载方法无法在自定义`Comparator`实现中被覆盖的 bug。

    此更改也被**回溯**到：0.9.5

    参考：[#3012](https://www.sqlalchemy.org/trac/ticket/3012)

+   **[sql] [bug]**

    修复了新的`DialectKWArgs.argument_for()`方法中的一个 bug，之前未包含任何特殊参数的构造添加参数将失败的问题。

    此更改也被**回溯**到：0.9.5

    参考：[#3024](https://www.sqlalchemy.org/trac/ticket/3024)

+   **[sql] [bug]**

    修复了 0.9 版本中引入的回归问题，即从[#1068](https://www.sqlalchemy.org/trac/ticket/1068)中的新“ORDER BY <labelname>”功能不会将标签名称在 ORDER BY 中呈现时应用引用规则。

    此更改也被**回溯**到：0.9.5

    参考：[#1068](https://www.sqlalchemy.org/trac/ticket/1068), [#3020](https://www.sqlalchemy.org/trac/ticket/3020)

+   **[sql] [bug]**

    恢复了 `Function` 的导入到 `sqlalchemy.sql.expression` 导入命名空间，该导入在 0.9 初始时被移除。

    此更改也已**回溯**至：0.9.5

+   **[sql] [bug]**

    `Insert.values()` 的多值版本已修复，可以更有效地与具有 Python 端默认值和/或函数以及服务器端默认值的表一起使用。该功能现在可以与使用“位置”参数的方言一起工作；Python 可调用对象也将像“executemany”样式调用一样为每一行单独调用；服务器端默认列将不再隐式接收明确为第一行指定的值，而是拒绝在没有明确值的情况下调用。

    另请参阅

    使用多值插入时为每行单独调用 Python 端默认值

    参考：[#3288](https://www.sqlalchemy.org/trac/ticket/3288)

+   **[sql] [bug]**

    修复了 `Table.tometadata()` 方法中的 bug，其中与 `Boolean` 或 `Enum` 类型对象关联的 `CheckConstraint` 将在目标表中重复。现在的复制过程跟踪此约束对象的生成作为类型对象的本地对象。

    参考：[#3260](https://www.sqlalchemy.org/trac/ticket/3260)

+   **[sql] [bug]**

    `ForeignKeyConstraint.columns` 集合的行为契约已经一致化；此属性现在像所有其他约束一样是一个 `ColumnCollection`，并在约束与 `Table` 关联时初始化。

    另请参阅

    ForeignKeyConstraint.columns 现在是 ColumnCollection

    参考：[#3243](https://www.sqlalchemy.org/trac/ticket/3243)

+   **[sql] [bug]**

    `Column.key` 属性现在用作表达式中匿名绑定参数名称的来源，以匹配此值在 INSERT 或 UPDATE 语句中呈现时的现有用法。这允许 `Column.key` 用作“替代”字符串，以解决一个难以转换为绑定参数名称的困难列名。请注意，无论如何，paramstyle 在 `create_engine()` 中是可配置的，并且今天大多数 DBAPI 都支持命名和位置样式。

    参考：[#3245](https://www.sqlalchemy.org/trac/ticket/3245)

+   **[sql] [bug]**

    修复了传递给此事件的 `PoolEvents.reset.dbapi_connection` 参数的名称；特别是这会影响此事件的“命名”参数样式的使用。感谢 Jason Goldberger 提交的拉取请求。

+   **[sql] [bug]**

    恢复了在 0.9 版本中所做的更改，即“常量” `null()`、`true()` 和 `false()` 的“单例”特性已被还原。这些函数返回一个“单例”对象的效果是，不同的实例将被视为相同，而不考虑其词法使用，这特别会影响 SELECT 语句的列子句的渲染。

    另请参阅

    null()、false() 和 true() 常量不再是单例

    参考：[#3170](https://www.sqlalchemy.org/trac/ticket/3170)

+   **[sql] [bug] [engine]**

    修复了一个 bug，即“分支”连接（当您调用 `Connection.connect()` 时获得的连接）不会与父连接共享失效状态。分支连接的架构稍作调整，使分支连接在所有失效状态和操作上都遵从父连接。

    参考：[#3215](https://www.sqlalchemy.org/trac/ticket/3215)

+   **[sql] [bug] [engine]**

    修复了一个 bug，即“分支”连接（当您调用 `Connection.connect()` 时获得的连接）不会与父连接共享事务状态。分支连接的架构稍作调整，使分支连接在所有事务状态和操作上都遵从父连接。

    参考：[#3190](https://www.sqlalchemy.org/trac/ticket/3190)

+   **[sql] [bug]**

    使用`Insert.from_select()`现在隐含在`insert()`上使用`inline=True`。这有助于修复一个 bug，即在支持的后端上，INSERT…FROM SELECT 结构会被错误地编译为“隐式返回”，这会导致在插入零行的情况下出现故障（因为隐式返回期望一行），以及在插入多行的情况下出现任意返回数据（例如，只有许多行中的第一行）。类似的更改也适用于具有多个参数集的 INSERT..VALUES；对于此语句，隐式 RETURNING 也不再发出。由于这两个构造处理可变数量的行，因此`ResultProxy.inserted_primary_key`访问器不适用。以前，有一个文档注释，即对于一些数据库不支持返回并且因此无法执行“隐式”返回的情况，可能更喜欢在 INSERT..FROM SELECT 中使用`inline=True`，但无论如何，INSERT…FROM SELECT 都不需要隐式返回。如果需要返回插入的数据的可变数量的结果行，则应使用常规的显式`Insert.returning()`。

    参考：[#3169](https://www.sqlalchemy.org/trac/ticket/3169)

+   **[SQL] [增强]**

    实现了`GenericTypeCompiler`的自定义方言现在可以构造，以便访问方法接收所属表达式对象的指示，如果有的话。大多数情况下，接受关键字参数（例如，`**kw`）的访问方法将接收一个关键字参数`type_expression`，指的是类型所包含的表达式对象。对于 DDL 中的列，方言的编译器类可能需要修改其`get_column_specification()`方法以支持此功能。如果`UserDefinedType.get_col_spec()`在其参数签名中提供了`**kw`，则还将接收`type_expression`。

    参考：[#3074](https://www.sqlalchemy.org/trac/ticket/3074)

### 模式

+   **[模式] [特性]**

    `MetaData.create_all()` 和 `MetaData.drop_all()` 的 DDL 生成系统已经增强，以自动处理大多数情况下的互相依赖的外键约束；对于约束，极大地减少了对 `ForeignKeyConstraint.use_alter` 标志的需求。该系统还适用于没有预先给定名称的约束；只有在 DROP 情况下，才需要至少一个约束的名称与循环相关联。

    另请参阅

    ForeignKeyConstraint 上的 use_alter 标志（通常）不再需要

    参考：[#3282](https://www.sqlalchemy.org/trac/ticket/3282)

+   **[模式] [特性]**

    添加了一个新的访问器 `Table.foreign_key_constraints` 来补充 `Table.foreign_keys` 集合，以及 `ForeignKeyConstraint.referred_table`。

+   **[模式] [错误]**

    `CheckConstraint` 构造现在支持包含令牌 `%(column_0_name)s` 的命名约定；约束表达式会扫描列。此外，不包括 `%(constraint_name)s` 令牌的检查约束的命名约定现在也适用于 `SchemaType` 生成的约束，例如 `Boolean` 和 `Enum` 的约束；这在 0.9.7 中停止工作，原因是 [#3067](https://www.sqlalchemy.org/trac/ticket/3067)。

    另请参阅

    命名 CHECK 约束

    配置布尔值、枚举和其他模式类型的命名

    参考：[#3067](https://www.sqlalchemy.org/trac/ticket/3067)，[#3299](https://www.sqlalchemy.org/trac/ticket/3299)

### postgresql

+   **[postgresql] [特性]**

    添加了对使用 `postgresql_concurrently` 建立的 PostgreSQL 索引的 `CONCURRENTLY` 关键字的支持。拉取请求由 Iuri de Silvio 提供。

    另请参阅

    CONCURRENTLY 创建索引

    此更改也被 **回溯** 至：0.9.9

+   **[postgresql] [特性] [pg8000]**

    为 pg8000 驱动程序添加了“合理的多行计数”支持，主要适用于在 ORM 中使用版本控制时。该功能基于使用 pg8000 1.9.14 或更高版本进行版本检测。感谢 Tony Locke 的拉取请求。

    此更改也**回溯**到：0.9.8

+   **[postgresql] [feature]**

    在`ColumnOperators.match()`操作符中添加了 kw 参数`postgresql_regconfig`，允许指定“reg config”参数传递给`to_tsquery()`函数。感谢 Jonathan Vanasco 的拉取请求。

    此更改也**回溯**到：0.9.7

    参考：[#3078](https://www.sqlalchemy.org/trac/ticket/3078)

+   **[postgresql] [feature]**

    通过`JSONB`添加了对 PostgreSQL JSONB 的支持。感谢 Damian Dimmich 的拉取请求。

    此更改也**回溯**到：0.9.7

+   **[postgresql] [feature]**

    在使用 pg8000 DBAPI 时，添加了对“AUTOCOMMIT”隔离级别的支持。感谢 Tony Locke 的拉取请求。

    此更改也**回溯**到：0.9.5

+   **[postgresql] [feature]**

    为 PostgreSQL `ARRAY`类型添加了一个新标志`ARRAY.zero_indexes`。当设置为`True`时，将在传递给数据库之前将所有数组索引值加一，从而在 Python 风格的从零开始索引和 PostgreSQL 从一开始索引之间实现更好的互操作性。感谢 Alexey Terentev 的拉取请求。

    此更改也**回溯**到：0.9.5

    参考：[#2785](https://www.sqlalchemy.org/trac/ticket/2785)

+   **[postgresql] [feature]**

    PG8000 方言现在支持`create_engine.encoding`参数，通过在连接上设置客户端编码，然后被 pg8000 拦截。感谢 Tony Locke 的拉取请求。

+   **[postgresql] [feature]**

    支持 PG8000 的原生 JSONB 功能。感谢 Tony Locke 的拉取请求。

+   **[postgresql] [feature] [pypy]**

    支持在 pypy 上使用 psycopg2cffi DBAPI。感谢 shauns 的拉取请求。

    另请参阅

    `sqlalchemy.dialects.postgresql.psycopg2cffi`

    参考：[#3052](https://www.sqlalchemy.org/trac/ticket/3052)

+   **[postgresql] [feature]**

    添加了对聚合函数应用的 FILTER 关键字的支持，由 PostgreSQL 9.4 支持。感谢 Ilja Everilä的拉取请求。

    另请参阅

    PostgreSQL FILTER 关键字

+   **[postgresql] [feature]**

    添加了对物化视图和外部表的反射支持，以及对`Inspector.get_view_names()`中的物化视图的支持，以及在 PostgreSQL 版本的`Inspector`上可用的新方法`PGInspector.get_foreign_table_names()`。拉取请求由 Rodrigo Menezes 提供。

    另请参阅

    PostgreSQL 方言反映了物化视图、外部表

    参考：[#2891](https://www.sqlalchemy.org/trac/ticket/2891)

+   **[postgresql] [feature]**

    在通过`Table`构造渲染 DDL 时，添加了对 PG 表选项 TABLESPACE、ON COMMIT、WITH(OUT) OIDS 和 INHERITS 的支持。拉取请求由 malikdiarra 提供。

    另请参阅

    PostgreSQL 表选项

    参考：[#2051](https://www.sqlalchemy.org/trac/ticket/2051)

+   **[postgresql] [feature]**

    添加了新方法`PGInspector.get_enums()`，在使用 PostgreSQL 检查器时将提供 ENUM 类型列表。拉取请求由 Ilya Pekelny 提供。

+   **[postgresql] [bug]**

    在 PG `HSTORE` 类型中添加了`hashable=False`标志，这是为了允许 ORM 在请求混合列/实体列表中的 ORM 映射的 HSTORE 列时跳过“哈希”操作所需的。补丁由 Gunnlaugur Þór Briem 提供。

    此更改也被**回溯**到：0.9.5, 0.8.7

    参考：[#3053](https://www.sqlalchemy.org/trac/ticket/3053)

+   **[postgresql] [bug]**

    添加了一个新的“断开连接”消息“连接意外关闭”。这似乎与较新版本的 SSL 有关。拉取请求由 Antti Haapala 提供。

    此更改也被**回溯**到：0.9.5, 0.8.7

+   **[postgresql] [bug]**

    修复了在使用 psycopg2 时与 ARRAY 类型一起支持 PostgreSQL UUID 类型的问题。psycopg2 方言现在使用 psycopg2.extras.register_uuid()钩子，以便始终将 UUID 值作为 UUID()对象传递到/从 DBAPI。`UUID.as_uuid`标志仍然受到尊重，但是对于 psycopg2，当禁用此标志时，我们需要将返回的 UUID 对象转换回字符串。

    此更改也被**回溯**到：0.9.9

    参考：[#2940](https://www.sqlalchemy.org/trac/ticket/2940)

+   **[postgresql] [bug]**

    在使用 psycopg2 2.5.4 或更高版本时，添加了对`postgresql.JSONB`数据类型的支持，该版本具有原生的 JSONB 数据转换，因此必须禁用 SQLAlchemy 的转换器；此外，还使用了新添加的 psycopg2 扩展`extras.register_default_jsonb`来建立通过`json_deserializer`参数传递给方言的 JSON 反序列化器。还修复了 PostgreSQL 集成测试，这些测试实际上并没有往返传输 JSONB 类型，而是 JSON 类型。感谢 Mateusz Susik 的拉取请求。

    此更改也被**回溯**到：0.9.9

+   **[postgresql] [bug]**

    修复了在使用旧的 psycopg2 版本< 2.4.3 注册 HSTORE 类型时使用“array_oid”标志的问题，该版本不支持此标志，以及在 psycopg2 版本< 2.5 上使用本机 json 序列化器钩子“register_default_json”与用户定义的`json_deserializer`时的问��，该版本不包括本机 json。

    此更改也被**回溯**到：0.9.9

+   **[postgresql] [bug]**

    修复了 PostgreSQL 方言在`Index`中无法正确渲染表绑定列之外的表达式的错误；通常情况下，如果`text()`构造是索引中的表达式之一，或者如果其中一个或多个表达式是这样的构造，则可能会误解表达式列表。

    此更改也被**回溯**到：0.9.9

    参考：[#3174](https://www.sqlalchemy.org/trac/ticket/3174)

+   **[postgresql] [bug]**

    重新审视了首次在 0.9.5 中修补的此问题，显然 psycopg2 的`.closed`访问器并不像我们所认为的那样可靠，因此我们已经添加了一个显式检查异常消息“SSL SYSCALL error: Bad file descriptor”和“SSL SYSCALL error: EOF detected”以检测断开连接的情况。我们将继续将 psycopg2 的 connection.closed 作为第一次检查。

    此更改也被**回溯**到：0.9.8

    参考：[#3021](https://www.sqlalchemy.org/trac/ticket/3021)

+   **[postgresql] [bug]**

    修复了 PostgreSQL JSON 类型无法持久化或以其他方式渲染 SQL NULL 列值而不是 JSON 编码的`'null'`的错误。为支持此情况，更改如下：

    +   现在可以指定值`null()`，这将始终导致语句中的 NULL 值。

    +   添加了一个新参数`JSON.none_as_null`，当为 True 时表示 Python 的`None`值应该被持久化为 SQL NULL，而不是 JSON 编码的`'null'`。

    对于除了 psycopg2 之外的其他 DBAPI，如 pg8000，也修复了将 NULL 检索为 None 的问题。

    此更改也被**回溯**到：0.9.8

    参考：[#3159](https://www.sqlalchemy.org/trac/ticket/3159)

+   **[postgresql] [bug]**

    用于 DBAPI 错误的异常包装系统现在可以容纳非标准的 DBAPI 异常，例如 psycopg2 的 TransactionRollbackError。这些异常现在将使用`sqlalchemy.exc`中最接近的可用子类引发，在 TransactionRollbackError 的情况下，使用`sqlalchemy.exc.OperationalError`。

    此更改也**回溯**到：0.9.8

    参考：[#3075](https://www.sqlalchemy.org/trac/ticket/3075)

+   **[postgresql] [bug]**

    修复了`array`对象中的错误，其中与普通 Python 列表的比较将无法使用正确的数组构造函数。感谢 Andrew 的拉取请求。

    此更改也**回溯**到：0.9.8

    参考：[#3141](https://www.sqlalchemy.org/trac/ticket/3141)

+   **[postgresql] [bug]**

    为函数添加了支持的`FunctionElement.alias()`方法，例如`func`构造。先前，此方法的行为是未定义的。当前行为模仿了 0.9.4 之前的行为，即将函数转换为具有给定别名的单列 FROM 子句，其中列本身是匿名命名的。

    此更改也**回溯**到：0.9.8

    参考：[#3137](https://www.sqlalchemy.org/trac/ticket/3137)

+   **[postgresql] [bug] [pg8000]**

    修复了 0.9.5 中由新的 pg8000 隔离级别功能引入的错误，其中引擎级隔离级别参数在连接时会引发错误。

    此更改也**回溯**到：0.9.7

    参考：[#3134](https://www.sqlalchemy.org/trac/ticket/3134)

+   **[postgresql] [bug]**

    现在在确定异常是否为“断开连接”错误时，将咨询 psycopg2 的`.closed`访问器；理想情况下，这应该消除对异常消息的任何其他检查以检测断开连接的需要，但我们将保留这些现有消息作为备用。这应该能够处理新的情况，如“SSL EOF”条件。感谢 Dirk Mueller 的拉取请求。

    此更改也**回溯**到：0.9.5

    参考：[#3021](https://www.sqlalchemy.org/trac/ticket/3021)

+   **[postgresql] [bug]**

    PostgreSQL `ENUM` 类型在调用普通的 `table.drop()` 时将发出 DROP TYPE 指令，假设对象没有直接与 `MetaData` 对象关联。为了适应在多个表之间共享枚举类型的用例，该类型应直接与 `MetaData` 对象关联；在这种情况下，该类型仅在元数据级别创建，或者如果直接创建。一般来说，已经对 PostgreSQL 枚举类型的创建/删除规则进行了高度改进。

    另请参阅

    ENUM 类型创建/删除规则的全面改进

    参考：[#3319](https://www.sqlalchemy.org/trac/ticket/3319)

+   **[postgresql] [bug]**

    `PGDialect.has_table()` 方法现在将查询 `pg_catalog.pg_table_is_visible(c.oid)`，而不是在模式名称为 None 时测试精确的模式匹配；这样该方法也将显示临时表的存在。请注意，这是一项行为更改，因为 PostgreSQL 允许非临时表悄悄地覆盖同名的现有临时表，因此这会改变在这种不寻常情况下 `checkfirst` 的行为。

    另请参阅

    PostgreSQL 的 has_table() 现在适用于临时表

    参考：[#3264](https://www.sqlalchemy.org/trac/ticket/3264)

+   **[postgresql] [enhancement]**

    在 PostgreSQL 方言中添加了一个新类型 `OID`。虽然“oid”通常是 PG 中的私有类型，在现代版本中不会暴露，但在某些 PG 使用情况下（如大对象支持）可能会暴露这些类型，以及在一些用户报告的模式反射使用情况中。

    此更改也已**回溯**至：0.9.5

    参考：[#3002](https://www.sqlalchemy.org/trac/ticket/3002)

### mysql

+   **[mysql] [feature]**

    MySQL 方言现在在所有情况下都使用 NULL / NOT NULL 渲染 TIMESTAMP，因此启用了 `explicit_defaults_for_timestamp` 标志的 MySQL 5.6.6 将允许 TIMESTAMP 在 `nullable=False` 时继续按预期工作。现有应用程序不受影响，因为 SQLAlchemy 一直为 `nullable=True` 的 TIMESTAMP 列发出 NULL。

    另请参阅

    MySQL TIMESTAMP 类型现在在所有情况下都渲染 NULL / NOT NULL

    TIMESTAMP 列和 NULL

    参考：[#3155](https://www.sqlalchemy.org/trac/ticket/3155)

+   **[mysql] [feature]**

    将“supports_unicode_statements”标志更新为 True，适用于 Python 2 下的 MySQLdb 和 Pymysql。这指的是 SQL 语句本身，而不是参数，影响到使用非 ASCII 字符的表和列名等问题。这两个驱动程序在现代版本中似乎都支持 Python 2 Unicode 对象而没有问题。

    参考：[#3121](https://www.sqlalchemy.org/trac/ticket/3121)

+   **[mysql] [change]**

    `gaerdbms`方言不再必要，并发出弃用警告。Google 现在建议直接使用 MySQLdb 方言。

    这个更改也被**回溯**到：0.9.9

    参考：[#3275](https://www.sqlalchemy.org/trac/ticket/3275)

+   **[mysql] [bug]**

    MySQL 错误 2014“commands out of sync”似乎在现代 MySQL-Python 版本中被提升为 ProgrammingError，而不是 OperationalError；所有被测试为“is disconnect”的 MySQL 错误代码现在都在 OperationalError 和 ProgrammingError 中进行检查。

    这个更改也被**回溯**到：0.9.7, 0.8.7

    参考：[#3101](https://www.sqlalchemy.org/trac/ticket/3101)

+   **[mysql] [bug]**

    修复了一个 bug，即在索引的`mysql_length`参数上添加列名时，需要对带引号的名称使用相同的引号才能被识别。修复使引号变为可选，但也为那些使用这种解决方法的人提供了旧的行为以保持向后兼容。

    这个更改也被**回溯**到：0.9.5, 0.8.7

    参考：[#3085](https://www.sqlalchemy.org/trac/ticket/3085)

+   **[mysql] [bug]**

    添加了对包含 KEY_BLOCK_SIZE 的索引的表进行反射的支持，使用等号。感谢 Sean McGivern 的拉取请求。

    这个更改也被**回溯**到：0.9.5, 0.8.7

+   **[mysql] [bug]**

    在 MySQLdb 方言周围添加了一个版本检查，用于检查‘utf8_bin’排序规则，因为这在 MySQL 服务器<5.0 上失败。

    这个更改也被**回溯**到：0.9.9

    参考：[#3274](https://www.sqlalchemy.org/trac/ticket/3274)

+   **[mysql] [bug] [mysqlconnector]**

    Mysqlconnector 在 2.0 版本中，可能是由于 Python 3 合并的副作用，现在不再期望百分号（例如用作模运算符和其他用途）被加倍，即使使用“pyformat”绑定参数格式（这个更改没有被 Mysqlconnector 记录）。方言现在在检测模运算符应该呈现为`%%`还是`%`时，检查 py2k 和 mysqlconnector 小于 2.0 版本。

    这个更改也被**回溯**到：0.9.8

+   **[mysql] [bug] [mysqlconnector]**

    现在对于 MySQLconnector 2.0 及以上版本传递 Unicode SQL；对于 Py2k 和 MySQL < 2.0，字符串被编码。

    这个更改也被**回溯**到：0.9.8

+   **[mysql] [bug]**

    MySQL 方言现在支持在构造为`TypeDecorator`对象的类型上进行 CAST。

+   **[mysql] [bug]**

    当在 MySQL 方言上使用 `cast()` 转换不支持的类型时，会发出警告；MySQL 仅支持对部分数据类型进行转换。长期以来，SQLAlchemy 在 MySQL 的情况下仅省略了不支持类型的 CAST。虽然我们现在不希望改变这一点，但我们发出警告以显示已经发生的情况。当使用不支持 CAST 的旧版 MySQL（< 4）时，也会发出警告，在这种情况下也会被跳过。

    参考：[#3237](https://www.sqlalchemy.org/trac/ticket/3237)

+   **[mysql] [bug]**

    `SET` 类型已经重构，不再假设空字符串或包含单个空字符串值的集合实际上是包含单个空字符串的集合；相反，默认情况下将其视为空集。为了处理实际上希望包含空白值 `''` 作为合法值的 `SET` 的持久性，添加了一种新的位操作模式，该模式通过 `SET.retrieve_as_bitwise` 标志启用，该标志将使用它们的位标志位置明确地持久化和检索值。还修复了对于不本地转换 Unicode 的驱动程序配置的 Unicode 值的存储和检索。

    另请参阅

    MySQL SET 类型重构以支持空集合、Unicode、空值处理

    参考：[#3283](https://www.sqlalchemy.org/trac/ticket/3283)

+   **[mysql] [bug]**

    `ColumnOperators.match()` 运算符现在处理方式不再严格假定返回类型为布尔值；现在返回一个名为 `MatchType` 的 `Boolean` 子类。当在 Python 表达式中使用时，该类型仍然会产生布尔行为，但是方言可以在结果时间覆盖其行为。在 MySQL 的情况下，虽然 MATCH 运算符通常在表达式的布尔上下文中使用，但如果实际上查询匹配表达式的值，则返回一个浮点值；此值与 SQLAlchemy 的基于 C 的布尔处理器不兼容，因此 MySQL 的结果集行为现在遵循 `Float` 类型的行为。还添加了一个新的运算符对象 `notmatch_op`，以更好地允许方言定义匹配操作的否定。

    另请参阅

    match()运算符现在返回与 MySQL 的浮点返回值兼容的 MatchType

    参考：[#3263](https://www.sqlalchemy.org/trac/ticket/3263)

+   **[mysql] [bug]**

    MySQL 布尔符号“true”、“false”再次有效。0.9 版本中的[#2682](https://www.sqlalchemy.org/trac/ticket/2682)更改禁止了 MySQL 方言在“IS”/“IS NOT”的上下文中使用“true”和“false”符号，但 MySQL 支持这种语法，尽管它没有布尔类型。MySQL 仍然是“非本机布尔”，但`true()`和`false()`符号再次生成关键字“true”和“false”，因此像`column.is_(true())`这样的表达式在 MySQL 上再次有效。

    另请参阅

    MySQL 布尔符号“true”、“false”再次有效

    参考：[#3186](https://www.sqlalchemy.org/trac/ticket/3186)

+   **[mysql] [bug]**

    MySQL 方言现在将禁用`ConnectionEvents.handle_error()`事件，以便对于其内部用于检测表是否存在的语句，不会触发该事件。这是通过使用一个执行选项`skip_user_error_events`来实现的，该选项在该执行范围内禁用了处理错误事件。通过这种方式，重写异常的用户代码不需要担心 MySQL 方言或其他偶尔需要捕获 SQLAlchemy 特定异常的方言。

+   **[mysql] [bug]**

    将“raise_on_warnings”的默认值更改为 False 以适用于 MySQLconnector。由于某种原因，这被设置为 True。不幸的是，“buffered”标志必须保持为 True，因为 MySQLconnector 不允许关闭游标，除非所有结果都被完全获取。

    参考：[#2515](https://www.sqlalchemy.org/trac/ticket/2515)

### sqlite

+   **[sqlite] [feature]**

    添加了对 SQLite 上的部分索引（例如带有 WHERE 子句）的支持。感谢 Kai Groner 的拉取请求。

    另请参阅

    部分索引

    此更改也**回溯**到：0.9.9

+   **[sqlite] [feature]**

    为 SQLCipher 后端添加了一个新的 SQLite 后端。该后端使用 pysqlcipher Python 驱动程序提供加密的 SQLite 数据库，该驱动程序与 pysqlite 驱动程序非常相似。

    另请参阅

    `pysqlcipher`

    此更改也**回溯**到：0.9.9

+   **[sqlite] [bug]**

    当使用附加数据库文件从 UNION 中进行选择时，pysqlite 驱动程序将列名报告为 ‘dbname.tablename.colname’，而不是通常对于 UNION 的 ‘tablename.colname’（请注意，对于两者，应该只是 ‘colname’，但我们对此进行了处理）。此处的列翻译逻辑已经调整为检索最右边的标记，而不是第二个标记，因此在两种情况下都有效。感谢 Tony Roberts 提供的解决方法。

    此更改也被**回溯**到：0.9.8

    参考：[#3211](https://www.sqlalchemy.org/trac/ticket/3211)

+   **[sqlite] [bug]**

    修复了 SQLite 连接重写问题，其中一个作为标量子查询嵌入的子查询（例如在 IN 中）会从包含查询中接收不适当的替换，如果相同的表在子查询中存在，如在连接继承场景中。

    此更改也被**回溯**到：0.9.7

    参考：[#3130](https://www.sqlalchemy.org/trac/ticket/3130)

+   **[sqlite] [bug]**

    现在在 SQLite 上完全反映了唯一约束和外键约束，无论是否有名称。之前，外键名称被忽略，未命名的唯一约束被跳过。感谢 Jon Nelson 的帮助。

    参考：[#3244](https://www.sqlalchemy.org/trac/ticket/3244), [#3261](https://www.sqlalchemy.org/trac/ticket/3261)

+   **[sqlite] [bug]**

    当使用 `DATE`、`TIME` 或 `DATETIME` 类型的 SQLite 方言，并给定一个只呈现数字的 `storage_format`，将在 DDL 中将类型呈现为 `DATE_CHAR`、`TIME_CHAR` 和 `DATETIME_CHAR`，以便尽管值中缺少字母字符，列仍会提供“文本亲和性”。通常情况下，这是不需要的，因为默认存储格式中的文本值已经暗示了文本。

    另请参阅

    日期和时间类型

    参考：[#3257](https://www.sqlalchemy.org/trac/ticket/3257)

+   **[sqlite] [bug]**

    SQLite 现在支持从临时表反射唯一约束；之前，这会导致 TypeError 错误。感谢 Johannes Erdfelt 提交的拉取请求。

    另请参阅

    SQLite/Oracle 有不同的临时表/视图名称报告方法 - 有关 SQLite 临时表和视图反射的更改。

    参考：[#3203](https://www.sqlalchemy.org/trac/ticket/3203)

+   **[sqlite] [bug]**

    添加了`Inspector.get_temp_table_names()`和`Inspector.get_temp_view_names()`；目前，只有 SQLite 和 Oracle 方言支持这些方法。临时表和视图名称的返回已从 SQLite 和 Oracle 的版本`Inspector.get_table_names()`和`Inspector.get_view_names()`中**移除**；其他数据库后端不支持此信息（如 MySQL），操作范围也不同，因为表可以是会话本地的，通常不支持远程模式中的表。

    另请参阅

    SQLite/Oracle 有不同的临时表/视图名称报告方法

    参考：[#3204](https://www.sqlalchemy.org/trac/ticket/3204)

### mssql

+   **[mssql] [feature]**

    为 SQL Server 2008 启用了“多值插入”。感谢 Albert Cervin 提交的拉取请求。还扩展了“IDENTITY INSERT”模式的检查，以包括当标识键出现在语句的 VALUEs 子句中时。

    此更改还**反向移植**到：0.9.7

+   **[mssql] [feature]**

    SQL Server 2012 现在推荐对于大文本/二进制类型使用 VARCHAR(max), NVARCHAR(max), VARBINARY(max)。MSSQL 方言现在会根据版本检测以及新的`deprecate_large_types`标志来尊重这一点。

    另请参阅

    大文本/二进制类型弃用

    参考：[#3039](https://www.sqlalchemy.org/trac/ticket/3039)

+   **[mssql] [changed]**

    当使用 pyodbc 时，基于主机名的 SQL Server 连接格式将不再指定默认的“驱动程序名称”，如果缺少此项将发出警告。SQL Server 的最佳驱动程序名称经常更改，并且是每个平台的，因此基于主机名的连接需要指定此项。优先使用 DSN-based 连接。

    另请参阅

    基于主机名的 SQL Server 连接需要 PyODBC 驱动程序名称

    参考：[#3182](https://www.sqlalchemy.org/trac/ticket/3182)

+   **[mssql] [bug]**

    添加了对“SET IDENTITY_INSERT”语句的语句编码，当明确在 IDENTITY 列中插入时进行操作，以支持在不支持 Unicode 语句的驱动程序（如 pyodbc + unix + py2k）上使用非 ASCII 表标识符。

    此更改还**反向移植**到：0.9.7, 0.8.7

+   **[mssql] [bug]**

    在 SQL Server pyodbc 方言中，修复了`description_encoding`方言参数的实现，当未明确设置时，会导致无法正确解析包含其他编码名称的结果集中的 cursor.description。未来不应该需要此参数。

    此更改也已**回溯**至：0.9.7, 0.8.7

    参考：[#3091](https://www.sqlalchemy.org/trac/ticket/3091)

+   **[mssql] [bug]**

    修复了在 pymssql 方言中检测版本字符串以与 Microsoft SQL Azure 一起使用的问题，后者将“SQL Server”更改为“SQL Azure”。

    此更改也已**回溯**至：0.9.8

    参考：[#3151](https://www.sqlalchemy.org/trac/ticket/3151)

+   **[mssql] [bug]**

    修订了用于确定当前默认模式名称的查询，使用`database_principal_id()`函数与`sys.database_principals`视图结合使用，以便我们可以独立于正在进行的登录类型（例如，SQL Server，Windows 等）确定默认模式。。

    此更改也已**回溯**至：0.9.5

    参考：[#3025](https://www.sqlalchemy.org/trac/ticket/3025)

### oracle

+   **[oracle] [feature]**

    增加了对 cx_oracle 连接到特定服务名称的支持，而不是传递`?service_name=<name>`到 URL 的 tns 名称。感谢 Sławomir Ehlert 的拉取请求。

+   **[oracle] [feature]**

    为表和索引添加了新的 Oracle DDL 功能：COMPRESS，BITMAP。补丁由 Gabor Gombas 提供。

+   **[oracle] [feature]**

    增加了对 Oracle 下 CTE 的支持。这包括对别名语法的一些调整，以及一个新的 CTE 功能`CTE.suffix_with()`，用于向 CTE 添加特殊的 Oracle 特定指令。

    另请参阅

    改进了 Oracle 中 CTE 的支持

    参考：[#3220](https://www.sqlalchemy.org/trac/ticket/3220)

+   **[oracle] [feature]**

    增加了对 Oracle 表选项 ON COMMIT 的支持。

+   **[oracle] [bug]**

    修复了 Oracle 方言中长期存在的 bug，即以数字开头的绑定参数名称不会被引用，因为 Oracle 不喜欢绑定参数名称中的数字。

    此更改也已**回溯**至：0.9.8

    参考：[#2138](https://www.sqlalchemy.org/trac/ticket/2138)

+   **[oracle] [bug] [tests]**

    修复了 oracle 方言测试套件中的 bug，在一个测试中，假定‘username’在数据库 URL 中，即使这可能并非如此。

    此更改也已**回溯**至：0.9.7

    参考：[#3128](https://www.sqlalchemy.org/trac/ticket/3128)

+   **[oracle] [bug]**

    当在`Select.with_hint()`方法中使用`%(name)s`标记引用别名时，别名将被正确引用。之前，Oracle 后端尚未实现此引用。

### 测试

+   **[tests] [bug]**

    修复了“python setup.py test”未适当调用 distutils 的错误，错误将在测试套件结束时发出。

    此更改也**回溯**至：0.9.7

+   **[测试] [bug] [py3k]**

    修正了涉及`imp`模块和 Python 3.3 或更高版本的一些弃用警告，在运行测试时。感谢 Matt Chisholm 的拉取请求。

    此更改也**回溯**至：0.9.5

    参考：[#2830](https://www.sqlalchemy.org/trac/ticket/2830)

### 杂项

+   **[功能] [扩展]**

    添加了一个新的扩展套件`sqlalchemy.ext.baked`。这个简单但不寻常的系统允许在构建和处理 orm `Query`对象时大大节省 Python 开销，从查询构建到渲染字符串 SQL 语句。

    另请参见

    烘焙查询

    参考：[#3054](https://www.sqlalchemy.org/trac/ticket/3054)

+   **[功能] [扩展]**

    `sqlalchemy.ext.automap`扩展现在将自动在检测到包含一个或多个非空列的外键的一对多关系/backref 上设置`cascade="all, delete-orphan"`。在这种情况下，此参数存在于传递给`generate_relationship()`的关键字中，仍然可以被覆盖。此外，如果`ForeignKeyConstraint`为非空列指定了`ondelete="CASCADE"`或为可空列指定了`ondelete="SET NULL"`，则还会将参数`passive_deletes=True`添加到关系中。请注意，并非所有后端都支持 ondelete 的反射，但支持反射的后端包括 PostgreSQL 和 MySQL。

    参考：[#3210](https://www.sqlalchemy.org/trac/ticket/3210)

+   **[bug] [声明性]**

    当访问`__mapper_args__`字典时，会从声明性 mixin 或抽象类中复制，以便声明性本身对该字典所做的修改不会与其他映射发生冲突。该字典在`version_id_col`和`polymorphic_on`参数方面进行修改，用本地类/表正式映射的列替换其中的列。

    此更改也**回溯**至：0.9.5，0.8.7

    参考：[#3062](https://www.sqlalchemy.org/trac/ticket/3062)

+   **[bug] [扩展]**

    修复了可变扩展中的一个 bug，`MutableDict`未对`setdefault()`字典操作报告更改事件。

    此更改也**回溯**至：0.9.5，0.8.7

    参考：[#3051](https://www.sqlalchemy.org/trac/ticket/3051)，[#3093](https://www.sqlalchemy.org/trac/ticket/3093)

+   **[bug] [扩展]**

    修复了`MutableDict.setdefault()`没有返回现有值或新值的错误（这个 bug 没有在任何 0.8 版本中发布）。感谢 Thomas Hervé提供的拉取请求。

    这个更改也被**回溯**到：0.9.5, 0.8.7

    参考：[#3051](https://www.sqlalchemy.org/trac/ticket/3051), [#3093](https://www.sqlalchemy.org/trac/ticket/3093)

+   **[bug] [ext] [py3k]**

    修复了在 Py3K 下关联代理列表类无法正确解释切片的错误。感谢 Gilles Dartiguelongue 提供的拉取请求。

    这个更改也被**回溯**到：0.9.9

+   **[bug] [declarative]**

    修复了一种在一些奇特的最终用户设置中观察到的不太可能的竞争条件，其中在声明中尝试检查“重复类名”时会遇到与其他被移除的类相关的未完全清理的弱引用；这里的检查现在确保在进一步调用之前弱引用仍然引用一个对象。

    这个更改也被**回溯**到：0.9.8

    参考：[#3208](https://www.sqlalchemy.org/trac/ticket/3208)

+   **[bug] [ext]**

    修复了在排序列表中的一个错误，其中如果将 reorder_on_append 标志设置为 True，则在集合替换事件期间项目的顺序会被打乱。修复确保排序列表仅影响与对象明确关联的列表。

    这个更改也被**回溯**到：0.9.8

    参考：[#3191](https://www.sqlalchemy.org/trac/ticket/3191)

+   **[bug] [ext]**

    修复了`MutableDict`未实现`update()`字典方法的错误，因此未捕获更改。感谢 Matt Chisholm 提供的拉取请求。

    这个更改也被**回溯**到：0.9.8

+   **[bug] [ext]**

    修复了`MutableDict`的自定义子类在“强制”操作中不会显示，并且会返回一个普通的`MutableDict`的错误。感谢 Matt Chisholm 提供的拉取请求。

    这个更改也被**回溯**到：0.9.8

+   **[bug] [pool]**

    修复了连接池日志记录中的错误，其中“连接已检出”调试日志记录消息不会在使用`logging.setLevel()`设置日志记录而不是使用`echo_pool`标志时发出。已添加用于断言此日志记录的测试。这是在 0.9.0 中引入的一个回退。

    这个更改也被**回溯**到：0.9.8

    参考：[#3168](https://www.sqlalchemy.org/trac/ticket/3168)

+   **[bug] [declarative]**

    修复了当声明的`__abstract__`标志未被区分为实际上是值`False`时的错误。`__abstract__`标志需要在被测试的级别实际上评估为 True 值。

    此更改也被**回溯**到：0.9.7

    参考：[#3097](https://www.sqlalchemy.org/trac/ticket/3097)

+   **[错误] [测试套件]**

    在公共测试套件中，从不太受支持的`Text`更改为使用`String(40)`在`StringTest.test_literal_backslashes`中。Pullreq 由 Jan 提供。

    此更改也被**回溯**到：0.9.5

+   **[已移除]**

    Drizzle 方言已从核心中移除；现在作为[sqlalchemy-drizzle](https://bitbucket.org/zzzeek/sqlalchemy-drizzle)提供，这是一个独立的第三方方言。该方言几乎完全基于 SQLAlchemy 中存在的 MySQL 方言。

    另请参见

    Drizzle 方言现在是外部方言

### 通用

+   **[通用] [特性]**

    通过更多地使用`__slots__`来改进结构内存使用。这种优化特别针对具有大量表和列的大型应用程序的基本内存大小，并大大减少了各种高容量对象的内存大小，包括事件监听内部、比较器对象以及 ORM 属性和加载器策略系统的部分。

    另请参见

    结构内存使用的显着改进

+   **[通用] [错误]**

    对于所有那些作为“公共工厂”符号派生的 SQL 和 ORM 函数，现在设置了`__module__`属性，这应有助于文档工具能够报告目标模块。

    参考：[#3218](https://www.sqlalchemy.org/trac/ticket/3218)

### ORM

+   **[ORM] [特性]**

    在`Query.column_descriptions`返回的字典中添加了一个新条目`"entity"`。这指的是由表达式引用的主要 ORM 映射类或别名类。与现有的`"type"`条目相比，它将始终是一个映射实体，即使从列表达式中提取，或者如果给定表达式是一个纯核心表达式，则为 None。另请参见[#3403](https://www.sqlalchemy.org/trac/ticket/3403)，修复了此功能中的一个回归，该回归在 0.9.10 中未发布，但在 1.0 版本中发布。

    此更改也被**回溯**到：0.9.10

    参考：[#3320](https://www.sqlalchemy.org/trac/ticket/3320)

+   **[ORM] [特性]**

    添加了新参数`Session.connection.execution_options`，可用于在事务开始之前首次检出`Connection`时设置执行选项。这用于在事务开始之前在连接上设置选项，如隔离级别。

    另请参见

    设置事务隔离级别 / DBAPI AUTOCOMMIT - 新的文档章节详细介绍了使用会话设置事务隔离的最佳实践。

    这个更改也**回溯**到：0.9.9

    参考：[#3296](https://www.sqlalchemy.org/trac/ticket/3296)

+   **[orm] [feature]**

    添加了新的方法`Session.invalidate()`，功能类似于`Session.close()`，除了还调用所有连接上的`Connection.invalidate()`，以确保它们不会返回到连接池中。这在处理 gevent 超时等情况时非常有用，此时不安全继续使用连接，即使是用于回滚也是如此。

    这个更改也**回溯**到：0.9.9

+   **[orm] [feature]**

    “primaryjoin”模型被进一步扩展，允许一个严格从单个列到其自身的连接条件，通过某种 SQL 函数或表达式转换。这有点是实验性的，但第一个概念验证是一个“材料化路径”连接条件，其中一个路径字符串与自身使用“like”进行比较。`ColumnOperators.like()`操作符也已添加到在 primaryjoin 条件中使用的有效操作符列表中。

    这个更改也**回溯**到：0.9.5

    参考：[#3029](https://www.sqlalchemy.org/trac/ticket/3029)

+   **[orm] [feature]**

    添加了新的实用函数`make_transient_to_detached()`，可用于制造行为就像从会话加载然后分离的对象。不存在的属性标记为过期，对象可以添加到会话中，在那里它将表现得像持久对象一样。

    这个更改也**回溯**到：0.9.5

    参考：[#3017](https://www.sqlalchemy.org/trac/ticket/3017)

+   **[orm] [feature]**

    添加了一个新的事件套件`QueryEvents`。`QueryEvents.before_compile()`事件允许创建函数，在构建 SELECT 语句之前对`Query`对象进行额外修改。希望通过引入一个新的检查系统，使这个事件变得更加有用，该系统将允许对`Query`对象进行详细的自动修改。

    另请参阅

    `QueryEvents`

    参考：[#3317](https://www.sqlalchemy.org/trac/ticket/3317)

+   **[orm] [功能]**

    当使用一对多查询并且还包含 LIMIT、OFFSET 或 DISTINCT 时，与一对一关系一起使用连接预加载时发生的子查询包装已在一对多关系中禁用，即一对多关系中`relationship.uselist`设置为 False。这将在这些情况下产生更有效的查询。

    另请参阅

    不再应用子查询于 uselist=False 的连接预加载

    参考：[#3249](https://www.sqlalchemy.org/trac/ticket/3249)

+   **[orm] [功能]**

    映射状态内部已经重新设计，以允许在对象“过期”时（如`Session.commit()`和`Session.expire_all()`的“自动过期”功能中）以及在对象状态被垃圾回收时发生的“清理”步骤中减少 50%的调用次数。

    参考：[#3307](https://www.sqlalchemy.org/trac/ticket/3307)

+   **[orm] [功能]**

    当在同一层次结构中为两个不同的映射器分配相同的多态标识时，会发出警告。这通常是用户错误，意味着在加载时无法正确区分这两种不同的映射类型。感谢 Sebastian Bank 的 Pull 请求。

    参考：[#3262](https://www.sqlalchemy.org/trac/ticket/3262)

+   **[orm] [功能]**

    创建了一系列新的`Session`方法，直接提供钩子进入工作单元的插入和更新语句发射设施。当正确使用时，这个面向专家的系统可以允许 ORM 映射生成批量插入和更新语句，分批执行，使语句的执行速度���以与直接使用 Core 相媲美。

    另请参阅

    批量操作

    参考：[#3100](https://www.sqlalchemy.org/trac/ticket/3100)

+   **[orm] [功能]**

    添加了一个参数`Query.join.isouter`，它与调用`Query.outerjoin()`是同义的；此标志旨在提供与 Core `FromClause.join()`更一致的接口。感谢 Jonathan Vanasco 提供的拉取请求。

    参考：[#3217](https://www.sqlalchemy.org/trac/ticket/3217)

+   **[orm] [功能]**

    添加了新的事件处理程序`AttributeEvents.init_collection()`和`AttributeEvents.dispose_collection()`，用于跟踪何时首次将集合与实例关联以及何时替换集合。这些处理程序取代了`collection.linker()`注释。通过事件适配器仍支持旧钩子。

+   **[orm] [功能]**

    当在映射或选项中使用`Query.yield_per()`时，`Query`将引发异常。这些加载策略目前与`yield_per`不兼容，因此通过引发此错误，该方法更安全。可以使用`lazyload('*')`选项或`Query.enable_eagerloads()`来禁用急加载。

    另请参阅

    使用 yield_per 明确禁止连接/子查询急加载

+   **[orm] [功能]**

    由`Query`对象使用的`KeyedTuple`的新实现在获取大量基于列的行时提供了显著的速度改进。

    另请参阅

    新的 KeyedTuple 实现速度显著提升

    参考：[#3176](https://www.sqlalchemy.org/trac/ticket/3176)

+   **[orm] [功能]**

    `joinedload.innerjoin`以及`relationship.innerjoin`的行为现在是使用“嵌套”内连接，即右嵌套，作为外连接贪婪加载链接到内连接贪婪加载时的默认行为。

    另请参阅

    使用 innerjoin=True 的 joinedload 现在默认为右内连接嵌套

    参考：[#3008](https://www.sqlalchemy.org/trac/ticket/3008)

+   **[orm] [特性]**

    现在可以将 UPDATE 语句批处理到 ORM 刷新中，以更高效地执行 executemany()调用，类似于可以将 INSERT 语句批处理；这将在刷新中被调用，以便后续针对相同映射和表的 UPDATE 语句涉及 VALUES 子句中的相同列，没有嵌入 SET 级别的 SQL 表达式，并且映射的版本要求与后端方言能够返回 executemany 操作的正确行数的兼容性。

+   **[orm] [特性]**

    `info`参数已添加到`SynonymProperty`和`ComparableProperty`的构造函数中。

    参考：[#2963](https://www.sqlalchemy.org/trac/ticket/2963)

+   **[orm] [特性]**

    `InspectionAttr.info`集合现在已移至`InspectionAttr`，除了在所有`MapperProperty`对象上可用外，还可在混合属性、关联代理上使用，通过`Mapper.all_orm_descriptors`访问时也可用。

    参考：[#2971](https://www.sqlalchemy.org/trac/ticket/2971)

+   **[orm] [更改]**

    映射属性标记为延迟加载，没有明确取消延迟加载的情况下，即使它们的列以某种方式出现在结果集中，现在也将保持“延迟加载”。 这是一个性能增强，因为 ORM 加载不再花时间搜索每个延迟加载的列。 但是，对于一直依赖此功能的应用程序，现在应该使用明确的`undefer()`或类似选项。

+   **[orm] [更改]**

    传递给自定义`Bundle`类的`create_row_processor()`方法的`proc()`可调用对象现在只接受一个“row”参数。

    另请参阅

    在使用自定义行加载程序时，新 Bundle 功能的 API 更改

+   **[orm] [更改]**

    弃用的事件钩子已移除：`populate_instance`，`create_instance`，`translate_row`，`append_result`

    另请参阅

    已移除的弃用 ORM 事件钩子

+   **[orm] [bug]**

    修复了子查询急加载中的 bug，其中在跨多态子类边界的长链急加载与多态加载相结合时，无法定位链���的子类链接，导致在一个`AliasedClass`上缺少属性名称的错误。

    此更改还**回溯**到：0.9.5, 0.8.7

    参考：[#3055](https://www.sqlalchemy.org/trac/ticket/3055)

+   **[orm] [bug]**

    修复了 ORM 中的一个 bug，`class_mapper()`函数会掩盖应在映射器配置期间引发的 AttributeErrors 或 KeyErrors，这是由于用户错误导致的。对于属性/键错误的捕获已经更具体，不包括配置步骤。

    此更改还**回溯**到：0.9.5, 0.8.7

    参考：[#3047](https://www.sqlalchemy.org/trac/ticket/3047)

+   **[orm] [bug]**

    修复了 ORM 对象比较中的错误，其中对于多对一的`!= None`比较，如果源是一个别名类，或者如果查询需要对表达式应用特殊别名处理，由于别名连接或多态查询而失败；还修复了一种情况，即如果比较多对一与对象状态，如果查询需要应用特殊别名处理，由于别名连接或多态查询而失败。

    此更改还**回溯**到：0.9.9

    参考：[#3310](https://www.sqlalchemy.org/trac/ticket/3310)

+   **[orm] [bug]**

    修复了一个 bug，在这种情况下，当一个`after_rollback()`处理程序为一个`Session`错误地在处理程序内向该`Session`添加状态时，内部断言将失败，并且警告和删除此状态的任务（由[#2389](https://www.sqlalchemy.org/trac/ticket/2389)建立）尝试继续进行。

    此更改还**回溯**到：0.9.9

    参考：[#3309](https://www.sqlalchemy.org/trac/ticket/3309)

+   **[orm] [bug]**

    修复了一个 bug，当`Query.join()`调用带有未知 kw 参数时，由于格式错误导致其自身引发 TypeError 的 TypeError。感谢 Malthe Borch 的拉取请求。

    此更改还**回溯**到：0.9.9

+   **[orm] [bug]**

    修复了懒加载 SQL 构造中的 bug，其中一个复杂的 primaryjoin 在“指向自身的列”的自引用连接风格中多次引用相同的“本地”列时，在所有情况下都不会被替换。这里确定替换的逻辑已经重新制定为更加开放式。

    此更改还被**回溯**到：0.9.9

    参考文献：[#3300](https://www.sqlalchemy.org/trac/ticket/3300)

+   **[orm] [错误]**

    “通配符”加载器选项，特别是由`load_only()`选项设置的选项，以覆盖未明确提及的所有属性，现在考虑了给定实体的超类，如果该实体使用继承映射进行映射，则超类中的属性名称也将从加载中省略。此外，多态鉴别器列无条件地包含在列表中，就像主键列一样，以便即使设置了 load_only()，子类型的多态加载也能继续正常工作。

    此更改还被**回溯**到：0.9.9

    参考文献：[#3287](https://www.sqlalchemy.org/trac/ticket/3287)

+   **[orm] [错误] [pypy]**

    修复了一个错误，即如果在 `Query` 开始之前抛出异常，特别是当无法形成行处理器时，游标将保持打开状态并且不会实际关闭结果。这通常仅在像 PyPy 这样的解释器上是一个问题，其中游标不会立即被 GC 清除，在某些情况下可能导致事务/锁定的打开时间比预期的长。

    此更改还被**回溯**到：0.9.9

    参考文献：[#3285](https://www.sqlalchemy.org/trac/ticket/3285)

+   **[orm] [错误]**

    修复了一个泄漏，该泄漏将在不支持且极不推荐的用例中发生，即在固定映射类上多次替换关系，引用任意增长的目标映射器的数量。当旧关系被替换时会发出警告，但是如果映射已用于查询，则旧关系仍将在某些注册表中引用。

    此更改还被**回溯**到：0.9.9

    参考文献：[#3251](https://www.sqlalchemy.org/trac/ticket/3251)

+   **[orm] [错误] [sqlite]**

    修复了表达式变异的错误，当使用`Query`从多个匿名列实体中选择时，如果针对 SQLite 进行查询，作为“联接重写”功能的副作用的一部分，可能会导致“找不到列”的错误。

    此更改还被**回溯**到：0.9.9

    参考文献：[#3241](https://www.sqlalchemy.org/trac/ticket/3241)

+   **[orm] [错误]**

    修复了使用`of_type()`将 `Query.join()` 和 `Query.outerjoin()` 加入到单继承子类时的 ON 子句的错误，如果设置了`from_joinpoint=True`标志，则不会在 ON 子句中呈现“单表条件”。

    此更改也**回溯**到：0.9.9

    参考：[#3232](https://www.sqlalchemy.org/trac/ticket/3232)

+   **[orm] [bug] [engine]**

    修复了一个 bug，影响了与#3199__，当使用`named=True`参数时，一些事件将无法注册，而其他事件将无法正确调用事件参数，通常在事件被“包装”以适应其他方式时。 “命名”机制已重新排列，以不干扰内部包装函数所期望的参数签名。

    此更改也**回溯**到：0.9.8

    参考：[#3197](https://www.sqlalchemy.org/trac/ticket/3197)

+   **[orm] [bug]**

    修复了一个 bug，影响了许多事件类别，特别是 ORM 事件，但也包括引擎事件，其中“去重复”使用相同参数调用`listen()`的常规逻辑将失败，对于那些监听器函数被包装的事件。在 registry.py 中会触发一个断言。现在，此断言已集成到去重复检查中，并且还有一个更简单的方法来全面检查去重复。

    此更改也**回溯**到：0.9.8

    参考：[#3199](https://www.sqlalchemy.org/trac/ticket/3199)

+   **[orm] [bug]**

    修复了一个警告，当一个复杂的自引用 primaryjoin 包含函数时会发出警告，同时指定了 remote_side；警告会建议设置“remote side”。现在只有在 remote_side 不存在时才会发出警告。

    此更改也**回溯**到：0.9.8

    参考：[#3194](https://www.sqlalchemy.org/trac/ticket/3194)

+   **[orm] [bug] [eagerloading]**

    修复了一个由 0.9.4 中引起的回归问题，该问题由[#2976](https://www.sqlalchemy.org/trac/ticket/2976)引起，当沿着一系列连接的贪婪加载传播“外连接”时，会错误地将沿着兄弟连接路径的“内连接”转换为外连接，当只有后代路径应该接收“外连接”传播时；此外，还修复了“嵌套”连接传播在两个兄弟连接路径之间的不当发生的相关问题。

    此更改也**回溯**到：0.9.7

    参考：[#3131](https://www.sqlalchemy.org/trac/ticket/3131)

+   **[orm] [bug]**

    修复了从 0.9.0 中的回归，由于[#2736](https://www.sqlalchemy.org/trac/ticket/2736)导致`Query.select_from()`方法不再正确设置`Query`对象的“from 实体”，因此后续的`Query.filter_by()`或`Query.join()`调用将无法在按字符串名称搜索属性时检查适当的“from”实体。

    此更改也**回溯**至：0.9.7

    参考：[#2736](https://www.sqlalchemy.org/trac/ticket/2736), [#3083](https://www.sqlalchemy.org/trac/ticket/3083)

+   **[orm] [bug]**

    修复了在保存点块内持久化、删除或主键更改的项目在外部事务回滚后不参与恢复到其先前状态（不在会话中、在会话中、先前的主键）的错误。

    此更改也**回溯**至：0.9.7

    参考：[#3108](https://www.sqlalchemy.org/trac/ticket/3108)

+   **[orm] [bug]**

    修复了与`with_polymorphic()`一起使用的子查询急加载中的错误，子查询加载中对实体和列的定位相对于此类型的实体和其他实体更加准确。

    此更改也**回溯**至：0.9.7

    参考：[#3106](https://www.sqlalchemy.org/trac/ticket/3106)

+   **[orm] [bug]**

    针对继承映射器隐式组合其基于列的属性之一与父级属性之一的情况添加了额外的检查，其中这些列通常不一定共享相同的值。这是通过[#1892](https://www.sqlalchemy.org/trac/ticket/1892)添加的现有检查的扩展；然而，这个新检查只发出警告，而不是异常，以允许依赖于现有行为的应用程序。

    另请参阅

    我收到关于“隐式组合列 X 在属性 Y 下”的警告或错误

    此更改也**回溯**至：0.9.5

    参考：[#3042](https://www.sqlalchemy.org/trac/ticket/3042)

+   **[orm] [bug]**

    修改了`load_only()`的行为，使得主键列始终添加到“未推迟”列的列表中；否则，ORM 无法加载行的标识。显然，可以推迟映射的主键，ORM 将失败，这一点没有改变。但是，由于 load_only 基本上是说“推迟除 X 外的所有列”，因此更重要的是 PK 列不参与此推迟。

    此更改也被**回溯**到：0.9.5

    参考：[#3080](https://www.sqlalchemy.org/trac/ticket/3080)

+   **[orm] [bug]**

    修复了一些边缘情况，这些情况出现在所谓的“行切换”场景中，其中 INSERT/DELETE 可以转换为 UPDATE。在这种情况下，将一个多对一关系设置为 None，或者在某些情况下将标量属性设置为 None，可能不会被检测为值的净变化，因此 UPDATE 不会重置前一行的内容。这是由于属性历史的一些尚未解决的副作用，隐式地假定 None 对于先前未设置的属性并不真正是一个“变化”。另请参见 [#3061](https://www.sqlalchemy.org/trac/ticket/3061)。

    注意

    此更改已在 0.9.6 版本中**撤销**。完整的修复将在 SQLAlchemy 的 1.0 版本中实现。

    此更改也被**回溯**到：0.9.5

    参考：[#3060](https://www.sqlalchemy.org/trac/ticket/3060)

+   **[orm] [bug]**

    与 [#3060](https://www.sqlalchemy.org/trac/ticket/3060) 相关，对工作单元进行了调整，以便在要删除的自引用对象图中，加载相关的多对一对象更加积极；加载相关对象有助于确定删除顺序的正确顺序，如果未设置 passive_deletes。

    此更改也被**回溯**到：0.9.5

+   **[orm] [bug]**

    修复了 SQLite 连接重写中的一个 bug，由于重复导致的匿名列名不会在子查询中正确重写。这会影响带有任何类型子查询 + 连接的 SELECT 查询。

    此更改也被**回溯**到：0.9.5

    参考：[#3057](https://www.sqlalchemy.org/trac/ticket/3057)

+   **[orm] [bug] [sql]**

    修复了在 [#2804](https://www.sqlalchemy.org/trac/ticket/2804) 中新增的布尔强制转换的 bug，新的“where”和“having”规则不会对 `select()` 构造函数的“whereclause”和“having”关键字参数产生影响，这也是 `Query` 使用的内容，因此在 ORM 中也无法正常工作。

    此更改也被**回溯**到：0.9.5

    参考：[#3013](https://www.sqlalchemy.org/trac/ticket/3013)

+   **[orm] [bug]**

    修复了一个 bug，即会话附加错误“对象已附加到会话 X”将无法阻止对象在错误引发后继续执行的情况下也被附加到新会话中。

    参考：[#3301](https://www.sqlalchemy.org/trac/ticket/3301)

+   **[orm] [bug]**

    当调用`Query.count()`、`Query.update()`、`Query.delete()`以及针对映射列、`column_property`对象和从映射列派生的 SQL 函数和表达式的查询时，`Query`的主要`Mapper`现在会传递给`Session.get_bind()`方法。这样，依赖于自定义`Session.get_bind()`方案或“绑定”元数据的会话可以在所有相关情况下工作。

    请参阅

    在所有相关查询情况下，Session.get_bind()将接收 Mapper

    参考：[#1326](https://www.sqlalchemy.org/trac/ticket/1326), [#3227](https://www.sqlalchemy.org/trac/ticket/3227), [#3242](https://www.sqlalchemy.org/trac/ticket/3242)

+   **[orm] [bug]**

    与`joinedload()`和`contains_eager()`等加载器指令一起，`PropComparator.of_type()`修饰符已经改进，以便在遇到相同基本类型/路径的两个`PropComparator.of_type()`修饰符时，它们将被合并为单个“多态”实体，而不是用类型 B 的实体替换类型 A 的实体。例如，`A.b.of_type(BSub1)->BSub1.c`的 joinedload 与`A.b.of_type(BSub2)->BSub2.c`的 joinedload 将创建一个单个的 joinedload，即`A.b.of_type((BSub1, BSub2)) -> BSub1.c, BSub2.c`，而不需要在查询中显式使用`with_polymorphic`。

    请参阅

    多态子类型的急加载 - 包含一个更新的示例，展示了新的格式。

    参考：[#3256](https://www.sqlalchemy.org/trac/ticket/3256)

+   **[orm] [bug]**

    修复了在使用`CascadeOptions`参数时，`copy.deepcopy()`调用的支持，这种情况发生在`copy.deepcopy()`与`relationship()`一起使用时（这不是官方支持的用例）。感谢 duesenfranz 的拉取请求。

+   **[orm] [bug]**

    修复了`Session.expunge()`在对象经历了已刷新但未提交的删除操作时不会完全分离给定对象的错误。这也会影响到类似`make_transient()`的相关操作。

    另请参阅

    session.expunge() 现在会完全分离已删除的对象

    参考：[#3139](https://www.sqlalchemy.org/trac/ticket/3139)

+   **[orm] [bug]**

    在多个关系最终将填充与另一个冲突的外键列的情况下，会发出警告，其中关系试图从不同源列复制值。这发生在将具有重叠列的复合外键映射到每个引用列不同的关系的情况下。一个新的文档部分说明了示例以及如何通过在每个关系上明确指定“外键”列来克服该问题。

    另请参阅

    重叠的外键

    参考：[#3230](https://www.sqlalchemy.org/trac/ticket/3230)

+   **[orm] [bug]**

    `Query.update()`方法现在将给定值字典中的字符串键名转换为针对正在更新的映射类的映射属性名。以前，字符串名称直接被接受并传递给核心更新语句，没有任何方法可以解析到映射实体。`Query.update()`的主题属性也支持同义词和混合属性。

    另请参阅

    query.update() 现在会将字符串名称解析为映射属性名称

    参考：[#3228](https://www.sqlalchemy.org/trac/ticket/3228)

+   **[orm] [bug]**

    改进了`Session`用于定位“绑定”（例如要使用的引擎）的机制，这些引擎可以与混合类、具体子类以及更广泛的表元数据（如联合继承表）关联。

    另请参阅

    Session.get_bind() 处理更广泛的继承场景

    参考：[#3035](https://www.sqlalchemy.org/trac/ticket/3035)

+   **[orm] [bug]**

    修复了单表继承中的错误，其中包含同一单一继承实体超过一次的连接链（通常应该引发错误）可能在某些情况下，取决于从何处连接，隐式别名第二次单一继承实体的情况，生成一个“有效”的查询。但由于在单表继承的情况下并不打算进行这种隐式别名处理，因此它并没有完全“有效”，并且非常具有误导性，因为它并不总是出现。

    另请参阅

    处理重复连接目标的更改和修复

    参考：[#3233](https://www.sqlalchemy.org/trac/ticket/3233)

+   **[orm] [bug]**

    当使用`Query.join()`、`Query.outerjoin()`或独立的`join()` / `outerjoin()`函数连接到单继承子类时，ON 子句现在将包括“单表条件”在 ON 子句中，即使 ON 子句是手动编写的；它现在被添加到条件中使用 AND，就像如果使用关系或类似方法连接到单表目标时一样。

    这有点介于特性和错误之间。

    另请参阅

    单表继承条件已无条件地添加到所有 ON 子句中

    参考：[#3222](https://www.sqlalchemy.org/trac/ticket/3222)

+   **[orm] [bug]**

    对表达式标签的行为进行了重大改进，特别是在与具有自定义 SQL 表达式的 ColumnProperty 构造和与 0.9 中首次引入的“按标签排序”逻辑结合使用时。修复包括`order_by(Entity.some_col_prop)`现在将使用“按标签排序”规则，即使 Entity 已经经历了别名处理，无论是通过继承渲染还是通过使用`aliased()`构造；多次使用别名渲染相同列属性（例如`query(Entity.some_prop, entity_alias.some_prop)`)将为实体的每次出现使用不同的标签，此外，“按标签排序”规则将适用于两者（例如`order_by(Entity.some_prop, entity_alias.some_prop)`）。在 0.9 中可能阻止“按标签排序”逻辑工作的其他问题，尤其是标签的状态可能会发生变化，导致“按标签排序”停止工作，具体取决于如何调用的问题已经修复。

    另请参阅

    ColumnProperty 构造在别名、order_by 中表现更好

    参考：[#3148](https://www.sqlalchemy.org/trac/ticket/3148)，[#3188](https://www.sqlalchemy.org/trac/ticket/3188)

+   **[orm] [bug]**

    更改了使用`Query.from_self()`或其常见用户`Query.count()`时应用“单继承条件”的方法。现在，将限制行数的条件指示在内部子查询中，而不是外部子查询中，因此即使“type”列不在列子句中，我们也可以在“内部”查询中对其进行过滤。

    另请参阅

    使用 from_self()，count()时更改为单表继承条件

    参考：[#3177](https://www.sqlalchemy.org/trac/ticket/3177)

+   **[orm] [bug]**

    对延迟加载的机制进行了小调整，以减少在极少情况下干扰 joinload()的机会；在这种情况下，对象在加载其属性时指向自身，这可能导致加载器之间的混淆。“对象指向自身”的用例并未得到完全支持，但修复也减少了一些开销，因此目前是测试的一部分。

    参考：[#3145](https://www.sqlalchemy.org/trac/ticket/3145)

+   **[orm] [bug]**

    已删除“复活”ORM 事件。自从 0.8 版本中删除了旧的“可变属性”系统后，此事件挂钩已经没有意义。

    参考：[#3171](https://www.sqlalchemy.org/trac/ticket/3171)

+   **[orm] [bug]**

    修复了在刷新过程中触发属性“set”事件或具有`@validates`的列的错误，当这些列是“获取和填充”操作的目标时，例如自增主键，Python 端默认值或通过 RETURNING“急切”获取的服务器端默认值。

    参考：[#3167](https://www.sqlalchemy.org/trac/ticket/3167)

+   **[orm] [bug] [py3k]**

    从`Session.identity_map`暴露的`IdentityMap`现在在 Py3K 中为`items()`和`values()`返回列表。早期移植到 Py3K 时，这些返回迭代器，但从技术上讲应该是“可迭代视图”..目前，列表是可以的。

+   **[orm] [bug]**

    对于 query.update()/delete()的“评估器”不适用于多表更新，需要将其设置为 synchronize_session=False 或 synchronize_session=’fetch’；现在会引发异常，并提示更改同步设置。这是从 0.9.7 开始发出的警告升级。

    参考：[#3117](https://www.sqlalchemy.org/trac/ticket/3117)

+   **[orm] [enhancement]**

    调整了属性机制，关于何时通过首次访问将值隐式初始化为 None；这一操作，一直导致属性的填充，现在不再这样；返回 None 值，但底层属性不接收设置事件。这与集合的工作方式一致，并允许属性机制更一致地行为；特别是，获取一个没有值的属性不会压制应该在实际将值设置为 None 时进行的事件。

    另请参阅

    关于没有预先存在值的属性的属性事件和其他操作的更改

    其中绑定参数根据编译时选项作为字符串内联呈现。这一功能的工作由 Dobes Vandermeer 提供。

    > 另请参阅
    > 
    > 选择/查询 LIMIT / OFFSET 可以指定为任意 SQL 表达式。

    参考：[#3061](https://www.sqlalchemy.org/trac/ticket/3061)

### orm 声明性

+   **[orm] [declarative] [feature]**

    `declared_attr` 构造在与声明性结合时具有新的改进行为和特性。装饰的函数在调用时现在将可以访问到本地混合类上的最终列副本，并且对于每个映射类将仅被调用一次，返回的结果将被缓存。同时还添加了一个新的修饰符 `declared_attr.cascading`。

    另请参阅

    改进声明性混合类、@declared_attr 和相关特性

    参考：[#3150](https://www.sqlalchemy.org/trac/ticket/3150)

+   **[orm] [declarative] [bug]**

    修复了在使用 `AbstractConcreteBase` 与声明了 `__abstract__` 的子类时出现的“‘NoneType’ 对象没有 ‘concrete’ 属性”错误。

    此更改也被**回溯**到：0.9.8

    参考：[#3185](https://www.sqlalchemy.org/trac/ticket/3185)

+   **[orm] [declarative] [bug]**

    修复了在声明性继承层次结构中使用 `__abstract__` 混合类会阻止属性和配置正确从基类传播到继承类的 bug。

    参考：[#3219](https://www.sqlalchemy.org/trac/ticket/3219), [#3240](https://www.sqlalchemy.org/trac/ticket/3240)

+   **[orm] [declarative] [bug]**

    使用`declared_attr`在`AbstractConcreteBase`基类上设置的关系现在将自动配置在抽象基本映射上，除了像往常一样在后代具体类上设置。

    另请参阅

    改进声明性混合，@declared_attr 和相关功能

    参考：[#2670](https://www.sqlalchemy.org/trac/ticket/2670)

### 示例

+   **[示例] [特性]**

    添加了一个新示例，演示了使用最新的关系特性的物化路径。示例由 Jack Zhou 提供。

    此更改也**回溯**到：0.9.5

+   **[示例] [特性]**

    一个新的示例套件，致力于从多个角度详细研究 SQLAlchemy ORM 和 Core 以及 DBAPI 的性能。该套件在一个容器中运行，通过控制台输出以及通过 RunSnake 工具图形化显示内置的性能分析。

    另请参阅

    性能

+   **[示例] [错误]**

    更新了带有历史表的版本控制示例，使映射列重新映射以匹配列名以及列的分组；特别是，这允许在历史映射中以相同方式映射明确分组的列，避免了在 0.9 系列中关于此模式的警告，并允许属性键的相同视图。

    此更改也**回溯**到：0.9.9

+   **[示例] [错误]**

    修复了示例/generic_associations/discriminator_on_association.py 中的一个错误，在这个示例中，AddressAssociation 的子类没有被映射为“单表继承”，导致在尝试进一步使用映射时出现问题。

    此更改也**回溯**到：0.9.9

### 引擎

+   **[引擎] [特性]**

    添加了用于查看事务隔离级别的新用户空间访问器；`Connection.get_isolation_level()`, `Connection.default_isolation_level`.

    此更改也**回溯**到：0.9.9

+   **[引擎] [特性]**

    添加了新事件`ConnectionEvents.handle_error()`，这是对`ConnectionEvents.dbapi_error()`的更全面和全面的替代。

    此更改也**回溯**到：0.9.7

    参考：[#3076](https://www.sqlalchemy.org/trac/ticket/3076)

+   **[引擎] [特性]**

    可以发出一种新风格的警告，该警告将“过滤”掉参数化字符串的 N 次出现。这允许参数化警告引用其参数，直到 Python 警告过滤器将其消除，并防止内存在 Python 的警告注册表中无限增长。

    另请参阅

    Session.get_bind() 处理更广泛的继承场景

    参考：[#3178](https://www.sqlalchemy.org/trac/ticket/3178)

+   **[引擎] [错误]**

    修复了`Connection`和池中的错误，其中`Connection.invalidate()`方法，或由于数据库断开连接而导致的失效，如果`isolation_level`参数与`Connection.execution_options()`一起使用，则会失败；重置隔离级别的“终结器”将在不再打开的连接上调用。

    此更改也已**回溯**至：0.9.9

    参考：[#3302](https://www.sqlalchemy.org/trac/ticket/3302)

+   **[引擎] [错误]**

    如果在使用`Transaction`时使用`isolation_level`参数与`Connection.execution_options()`，则会发出警告；DBAPIs 和/或 SQLAlchemy 方言（如 psycopg2、MySQLdb）可能会隐式回滚或提交事务，或者在下一个事务中不更改设置，因此这永远不安全。

    此更改也已**回溯**至：0.9.9

    参考：[#3296](https://www.sqlalchemy.org/trac/ticket/3296)

+   **[引擎] [错误]**

    传递给`Engine`的执行选项，无论是通过`create_engine.execution_options`还是`Engine.update_execution_options()`，都不会传递给用于在“第一次连接”事件中初始化方言的特殊`Connection`；方言通常会在此阶段执行自己的查询，并且不应在此应用任何当前可用的选项。特别是，“autocommit”选项导致在此初始连接中尝试自动提交，这将由于`Connection`的非标准状态而导致 AttributeError。

    此更改也**回溯**到：0.9.8

    参考：[#3200](https://www.sqlalchemy.org/trac/ticket/3200)

+   **[引擎] [错误]**

    用于确定插入或更新所影响列的字符串键现在在对“编译缓存”缓存键做出贡献时进行排序。这些键以前没有确定性地排序，这意味着相同的语句可能会根据等效键多次缓存，这既在内存方面又在性能方面造成了损失。

    此更改也**回溯**到：0.9.8

    参考：[#3165](https://www.sqlalchemy.org/trac/ticket/3165)

+   **[引��] [错误]**

    修复了一个错误，即当引擎首次连接并进行初始检查时发生 DBAPI 异常，且异常不是断开异常，但是当我们尝试关闭光标时光标引发错误。在这种情况下，真正的异常将被扼杀，因为我们尝试通过连接池记录光标关闭异常并失败，因为我们试图以不适当的方式访问池的记录器，这在这种非常特定的情况下是不合适的。

    此更改也**回溯**到：0.9.5

    参考：[#3063](https://www.sqlalchemy.org/trac/ticket/3063)

+   **[引擎] [错误]**

    修复了一些检测到的“双重失效”情况，其中连接失效可能发生在已经处于关键部分（如 connection.close()）的情况下；最终，这些条件是由于[#2907](https://www.sqlalchemy.org/trac/ticket/2907)中的更改引起的，因为“返回时重置”功能调用 Connection/Transaction 以处理它，其中可能会捕获“断开检测”。然而，最近在[#2985](https://www.sqlalchemy.org/trac/ticket/2985)中的更改可能使这种情况更容易被视为“连接失效”操作更快，因为在 0.9.4 上更容易复现这个问题，而不是在 0.9.3 上。

    现在在可能发生无效的任何部分中添加检查以阻止在无效连接上进行进一步的不允许的操作。这包括引擎级别和池级别的两个修复。虽然这个问题在高度并发的 gevent 情况下被观察到，但理论上它可能发生在任何断开连接发生在连接关闭操作内的情况下。

    这个更改也被**回溯**到：0.9.5

    参考：[#3043](https://www.sqlalchemy.org/trac/ticket/3043)

+   **[engine] [错误]**

    引擎级别的错误处理和包装程序现在将在所有引擎连接使用情况下生效，包括当用户自定义连接程序通过`create_engine.creator`参数使用时，以及当`Connection`在重新验证时遇到连接错误时。

    另请参阅

    DBAPI 异常包装和 handle_error()事件改进

    参考：[#3266](https://www.sqlalchemy.org/trac/ticket/3266)

+   **[engine] [错误]**

    在事件运行时同时删除（或添加）事件监听器，无论是从监听器内部还是从并发线程中，现在会引发 RuntimeError，因为现在使用的集合是`collections.deque()`的实例，不支持在迭代时进行更改。以前使用的是普通的 Python 列表，其中从事件本身内部删除会产生静默失败。

    参考：[#3163](https://www.sqlalchemy.org/trac/ticket/3163)

### sql

+   **[sql] [功能]**

    在`Index`的合同中稍微放宽了一点，你可以指定一个`text()`表达式作为目标；如果要手动将索引添加到表中，则索引不再需要存在绑定表列，可以通过内联声明或通过`Table.append_constraint()`添加。

    这个更改也被**回溯**到：0.9.5

    参考：[#3028](https://www.sqlalchemy.org/trac/ticket/3028)

+   **[sql] [功能]**

    添加了新标志`between.symmetric`，当设置为 True 时呈现“BETWEEN SYMMETRIC”。还添加了一个新的否定运算符“notbetween_op”，现在允许像`~col.between(x, y)`这样的表达式呈现为“col NOT BETWEEN x AND y”，而不是一个带括号的 NOT 字符串。

    这个更改也被**回溯**到：0.9.5

    参考：[#2990](https://www.sqlalchemy.org/trac/ticket/2990)

+   **[sql] [功能]**

    现在 SQL 编译器生成预期列的映射，使其按位置匹配接收到的结果集，而不是按名称匹配。最初，这被视为一种处理列返回具有难以预测名称的情况的方法，尽管在现代使用中，这个问题已经通过匿名标记得以解决。在这个版本中，这种方法基本上通过减少每个结果的函数调用次数几十次，或者对于更大的结果列集合来说更多。如果编译的列集合与接收到的列存在大小上的任何差异，这种方法仍然会退化为旧方法的现代版本，因此在这些列表可能不对齐的部分或完全文本编译场景中没有问题。

    参考：[#918](https://www.sqlalchemy.org/trac/ticket/918)

+   **[sql] [feature]**

    在`DefaultClause`中的字面值，当使用`Column.server_default`参数时被调用，现在将使用“内联”编译器呈现，以便它们按原样呈现，而不是作为绑定参数。

    另请参阅

    列服务器默认值现在呈现字面值

    参考：[#3087](https://www.sqlalchemy.org/trac/ticket/3087)

+   **[sql] [feature]**

    当传递给 SQL 表达式单元的对象无法解释为 SQL 片段时，报告表达式的类型；感谢 Ryan P. Kelly 的拉取请求。

+   **[sql] [feature]**

    向`Table.tometadata()`方法添加了一个新参数`Table.tometadata.name`。类似于`Table.tometadata.schema`，这个参数使得新复制的`Table`采用新名称而不是现有名称。这个功能的一个有趣的能力是，可以将`Table`对象复制到*相同的* `MetaData` 目标上并赋予新名称。感谢 n.d. parker 的拉取请求。

+   **[sql] [feature]**

    异常消息稍作调整。如果为 None，则不显示 SQL 语句和参数，减少与语句无关的错误消息的混淆。显示了 DBAPI 级别异常的完整模块和类名，清楚地表明这是一个包装的 DBAPI 异常。语句和参数本身被限定在括号内，以更好地将它们与错误消息和彼此隔离。

    参考：[#3172](https://www.sqlalchemy.org/trac/ticket/3172)

+   **[sql] [feature]**

    `Insert.from_select()` 现在包括 Python 和 SQL 表达式默认值，如果未指定其他值；解除了非服务器列默认值不包含在 INSERT FROM SELECT 中的限制，并且这些表达式被渲染为常量插入到 SELECT 语句中。

    另请参阅

    INSERT FROM SELECT 现在包括 Python 和 SQL 表达式默认值

+   **[sql] [feature]**

    当反射一个 `Table` 对象时，现在会包含 `UniqueConstraint` 构造，适用于这种情况的数据库。为了以足够的准确性实现这一点，MySQL 和 PostgreSQL 现在包含了在反射表、索引和约束时纠正索引和唯一约束重复的功能。对于 MySQL，实际上并没有独立于“唯一索引”的“唯一约束”概念，因此对于这个后端，反射的 `Table` 中仍然不包含 `UniqueConstraint`。对于 PostgreSQL，用于检测 `pg_index` 中的索引的查询已经改进，以检查 `pg_constraint` 中的相同构造，并且在反射的 `Table` 中不包含隐式构造的唯一索引。

    在这两种情况下，`Inspector.get_indexes()` 和 `Inspector.get_unique_constraints()` 方法分别返回这两种构造，但在 PostgreSQL 的情况下包含一个新的标记 `duplicates_constraint`，在 MySQL 的情况下包含一个 `duplicates_index` 标记以指示检测到此条件时。感谢 Johannes Erdfelt 提交的拉取请求。

    另请参阅

    UniqueConstraint 现在已经成为表反射过程的一部分

    参考：[#3184](https://www.sqlalchemy.org/trac/ticket/3184)

+   **-   [sql] [功能]**

    添加了新方法 `Select.with_statement_hint()` 和 ORM 方法 `Query.with_statement_hint()` 以支持���特定于表的语句级提示。

    参考：[#3206](https://www.sqlalchemy.org/trac/ticket/3206)

+   **-   [sql] [功能]**

    `info` 参数已添加为所有模式构造的构造函数参数，包括 `MetaData`、`Index`、`ForeignKey`、`ForeignKeyConstraint`、`UniqueConstraint`、`PrimaryKeyConstraint`、`CheckConstraint`。

    参考：[#2963](https://www.sqlalchemy.org/trac/ticket/2963)

+   **-   [sql] [功能]**

    `Table.autoload_with` 标志现在意味着 `Table.autoload` 应为 `True`。感谢 Malik Diarra 的拉取请求。

    参考：[#3027](https://www.sqlalchemy.org/trac/ticket/3027)

+   **-   [sql] [功能]**

    `Select.limit()` 和 `Select.offset()` 方法现在接受任何 SQL 表达式作为参数，而不仅仅是整数值。通常用于允许传递绑定参数，稍后可以用值替换，从而允许在 Python 端缓存 SQL 查询。这里的实现完全向后兼容现有的第三方方言，但那些实现特殊 LIMIT/OFFSET 系统的方言需要修改以利用新功能。Limit 和 offset 还支持“literal_binds”模式，

    参考：[#3034](https://www.sqlalchemy.org/trac/ticket/3034)

+   **-   [sql] [更改]**

    `column()`和`table()`构造现在可以从“from sqlalchemy”命名空间导入，就像每个其他核心构造一样。

+   **[sql] [changed]**

    当传递给`select()`的大多数构建器方法以及`Query`时，将字符串隐式转换为`text()`构造时，现在会发出警告，只发送纯字符串。文本转换仍然正常进行。唯一不会发出警告的方法是“标签引用”方法，如 order_by()，group_by()；这些函数现在在编译时将尝试将单个字符串参数解析为可选择的列或标签表达式；如果找不到任何内容，则表达式仍会呈现，但您会再次收到警告。这里的理由是，从字符串到文本的隐式转换现在比以往更加意外，当用户将原始字符串传递给 Core / ORM 时，最好让用户提供更多方向以确定应采取什么方向。 Core / ORM 教程已更新，以更深入地介绍文本的处理。

    另请参阅

    将完整的 SQL 片段强制转换为 text() 时发出的警告

    参考：[#2992](https://www.sqlalchemy.org/trac/ticket/2992)

+   **[sql] [bug]**

    修复了`Enum`和其他`SchemaType`子类中的错误，直接将类型与`MetaData`关联会导致在`MetaData`上发出事件（如创建事件）时挂起。

    此更改也**回溯**到：0.9.7, 0.8.7

    参考：[#3124](https://www.sqlalchemy.org/trac/ticket/3124)

+   **[sql] [bug]**

    修复了在自定义操作符加法`TypeEngine.with_variant()`系统中的一个错误，当与变体一起使用`TypeDecorator`时，如果使用比较运算符，则会出现 MRO 错误。

    此更改也**回溯**到：0.9.7, 0.8.7

    参考：[#3102](https://www.sqlalchemy.org/trac/ticket/3102)

+   **[sql] [bug]**

    修复了在 INSERT..FROM SELECT 结构中的 bug，从 UNION 中选择会将联合包装在一个匿名（例如未标记）子查询中。

    此更改也被**回溯**到：0.9.5, 0.8.7

    参考：[#3044](https://www.sqlalchemy.org/trac/ticket/3044)

+   **[sql] [bug]**

    修复了一个 bug，`Table.update()`和`Table.delete()`在应用空的`and_()`或`or_()`或其他空表达式时会产生空的 WHERE 子句。现在与`select()`的行为一致。

    此更改也被**回溯**到：0.9.5, 0.8.7

    参考：[#3045](https://www.sqlalchemy.org/trac/ticket/3045)

+   **[sql] [bug]**

    在`Enum`的`__repr__()`输出中添加了`native_enum`标志，当与 Alembic autogenerate 一起使用时非常重要。感谢 Dimitris Theodorou 的拉取请求。

    此更改也被**回溯**到：0.9.9

+   **[sql] [bug]**

    修复了一个 bug，使用实现了也是`TypeDecorator`的类型会在针对使用此类型的对象使用任何类型的 SQL 比较表达式时失败，Python 会出现“无法创建一致的方法解析顺序（MRO）”错误。

    此更改也被**回溯**到：0.9.9

    参考：[#3278](https://www.sqlalchemy.org/trac/ticket/3278)

+   **[sql] [bug]**

    修复了一个问题，SELECT 嵌入到 INSERT 中，通过值子句或作为“from select”，当两个语句的列共享相同名称时，会污染由 RETURNING 子句产生的结果集中使用的列类型，导致在检索返回行时可能出现错误或误适应。

    此更改也被**回溯**到：0.9.9

    参考：[#3248](https://www.sqlalchemy.org/trac/ticket/3248)

+   **[sql] [bug]**

    修复了一个 bug，在 sql 包中的大量 SQL 元素无法成功执行`__repr__()`，因为缺少一个`description`属性，然后会在内部 AttributeError 重新调用`__repr__()`时触发递归溢出。

    此更改也被**回溯**到：0.9.8

    参考：[#3195](https://www.sqlalchemy.org/trac/ticket/3195)

+   **[sql] [bug]**

    调整了表/索引反射，如果索引报告一个在表中找不到的列，将发出警告并跳过该列。这可能发生在一些特殊的系统列情况下，如在 Oracle 中观察到的情况。

    此更改也**回溯**到：0.9.8

    参考：[#3180](https://www.sqlalchemy.org/trac/ticket/3180)

+   **[sql] [bug]**

    修复了 CTE 中的一个 bug，即当一个 CTE 引用另一个别名 CTE 时，`literal_binds`编译器参数不会始终正确传播的问题。

    此更改也**回溯**到：0.9.8

    参考：[#3154](https://www.sqlalchemy.org/trac/ticket/3154)

+   **[sql] [bug]**

    修复了由[#3067](https://www.sqlalchemy.org/trac/ticket/3067)引起的 0.9.7 回归，与一个命名错误的单元测试一起，导致所谓的“模式”类型如`Boolean`和`Enum`无法再被 pickle。

    此更改也**回溯**到：0.9.8

    参考：[#3067](https://www.sqlalchemy.org/trac/ticket/3067)，[#3144](https://www.sqlalchemy.org/trac/ticket/3144)

+   **[sql] [bug]**

    修复了命名约定功能中的 bug，其中使用包含`constraint_name`的检查约定将强制所有`Boolean`和`Enum`类型也需要名称，因为这些隐式创建约束，即使最终目标后端不需要生成约束，如 PostgreSQL。这些特定约束的命名约定机制已重新组织，使得命名确定在 DDL 编译时进行，而不是在约束/表构建时进行。

    此更改也**回溯**到：0.9.7

    参考：[#3067](https://www.sqlalchemy.org/trac/ticket/3067)

+   **[sql] [bug]**

    修复了通用表达式中的一个 bug，即当 CTE 在某些方式中嵌套时，位置绑定参数可能以错误的最终顺序表达的问题。

    此更改也**回溯**到：0.9.7

    参考：[#3090](https://www.sqlalchemy.org/trac/ticket/3090)

+   **[sql] [bug]**

    修复了多值`Insert`构造中的 bug，使其无法检查除第一个给定的字面 SQL 表达式值条目之外的后续值条目。

    此更改也**回溯**到：0.9.7

    参考：[#3069](https://www.sqlalchemy.org/trac/ticket/3069)

+   **[sql] [bug]**

    为 Python 版本< 2.6.5 的 dialect_kwargs 迭代添加了一个“str()”步骤，解决了“无 unicode 关键字参数”bug，因为这些参数在某些反射过程中作为关键字参数传递。

    此更改也**回溯**到：0.9.7

    参考：[#3123](https://www.sqlalchemy.org/trac/ticket/3123)

+   **[sql] [bug]**

    `TypeEngine.with_variant()` 方法现在将接受类型类作为参数，该参数将被内部转换为一个实例，使用已经被其他构造（如`Column`）长期建立的相同约定。

    此更改还被**回溯**到：0.9.7

    参考：[#3122](https://www.sqlalchemy.org/trac/ticket/3122)

+   **[sql] [bug]**

    当该表中的`Column`在显式`PrimaryKeyConstraint`中引用时，`Column.nullable`标志被隐式设置为`False`。现在，该行为与`Column`本身的`Column.primary_key`标志设置为`True`时相匹配，这被认为是一个完全等效的情况。

    此更改还被**回溯**到：0.9.5

    参考：[#3023](https://www.sqlalchemy.org/trac/ticket/3023)

+   **[sql] [bug]**

    修复了在自定义`Comparator`实现中无法重写`Operators.__and__()`、`Operators.__or__()`和`Operators.__invert__()`运算符重载方法的错误。

    此更改还被**回溯**到：0.9.5

    参考：[#3012](https://www.sqlalchemy.org/trac/ticket/3012)

+   **[sql] [bug]**

    修复了在新的`DialectKWArgs.argument_for()`方法中的错误，在这种方法中，添加一个对于以前未包含在任何特殊参数中的构造的参数将会失败。

    此更改还被**回溯**到：0.9.5

    参考：[#3024](https://www.sqlalchemy.org/trac/ticket/3024)

+   **[sql] [bug]**

    修复了在 0.9 版本中引入的回归，即从[#1068](https://www.sqlalchemy.org/trac/ticket/1068)中的新“ORDER BY <labelname>”功能不会将标签名称渲染到 ORDER BY 中应用引用规则的情况。

    此更改还被**回溯**到：0.9.5

    参考：[#1068](https://www.sqlalchemy.org/trac/ticket/1068), [#3020](https://www.sqlalchemy.org/trac/ticket/3020)

+   **[sql] [bug]**

    `Function` 的导入已经恢复到了 `sqlalchemy.sql.expression` 导入命名空间中，这是在 0.9 版本初期移除的。

    这个更改也被 **回溯** 到了：0.9.5

+   **[sql] [bug]**

    `Insert.values()` 的多值版本已修复，可以更有用地与具有 Python 端默认值和/或函数以及服务器端默认值的表一起使用。该功能现在可以在使用“位置参数”的方言中工作；一个 Python 可调用对象也将与“executemany”风格的调用一样，为每一行单独调用；服务器端默认列将不再隐式地接收显式指定给第一行的值，而是拒绝不带显式值的调用。

    另见

    在使用多值插入时，每行都会单独调用 Python 端默认值（Python-side defaults）。

    参考：[#3288](https://www.sqlalchemy.org/trac/ticket/3288)

+   **[sql] [bug]**

    修复了 `Table.tometadata()` 方法中的错误，其中与 `Boolean` 或 `Enum` 类型对象相关联的 `CheckConstraint` 将在目标表中重复。现在的复制过程将这个约束对象的生成跟踪为局部于类型对象。

    参考：[#3260](https://www.sqlalchemy.org/trac/ticket/3260)

+   **[sql] [bug]**

    `ForeignKeyConstraint.columns` 集合的行为契约已经一致化；此属性现在像所有其他约束一样是一个 `ColumnCollection`，并在约束与 `Table` 关联时初始化。

    另见

    `ForeignKeyConstraint.columns` 现在是一个 `ColumnCollection`。

    参考：[#3243](https://www.sqlalchemy.org/trac/ticket/3243)

+   **[sql] [bug]**

    `Column.key` 属性现在用作表达式中匿名绑定参数名称的来源，以匹配此值在 INSERT 或 UPDATE 语句中呈现时作为键的现有使用情况。这允许 `Column.key` 用作“替代”字符串，以解决一个不太适合作为绑定参数名称的困难列名。请注意，无论如何，`create_engine()` 上的 paramstyle 都是可配置的，并且今天的大多数 DBAPI 都支持命名和位置样式。

    参考：[#3245](https://www.sqlalchemy.org/trac/ticket/3245)

+   **[sql] [bug]**

    修复了 `PoolEvents.reset.dbapi_connection` 参数在传递给此事件时的名称；特别是这会影响到此事件的“命名”参数样式的使用。感谢 Jason Goldberger 的 Pull 请求。

+   **[sql] [bug]**

    撤销了在 0.9 中进行的更改，即“常量”`null()`、`true()` 和 `false()` 的“单例”性质已经恢复。这些返回“单例”对象的函数会导致不同的实例无论在语法上如何使用都会被视为相同，特别是会影响到 SELECT 语句的列子句的渲染。

    另见

    `null()`、`false()` 和 `true()` 常量不再是单例

    参考：[#3170](https://www.sqlalchemy.org/trac/ticket/3170)

+   **[sql] [bug] [engine]**

    修复了一个 bug，即“分支”连接（即调用 `Connection.connect()` 时获得的连接）未与父连接共享失效状态的问题。分支架构已经进行了微调，使得分支连接在所有失效状态和操作上都 defer 到父连接。

    参考：[#3215](https://www.sqlalchemy.org/trac/ticket/3215)

+   **[sql] [bug] [engine]**

    修复了一个 bug，即“分支”连接（即调用 `Connection.connect()` 时获得的连接）未与父连接共享事务状态的问题。分支架构已经进行了微调，使得分支连接在所有事务状态和操作上都 defer 到父连接。

    参考：[#3190](https://www.sqlalchemy.org/trac/ticket/3190)

+   **[sql] [bug]**

    现在在使用`Insert.from_select()`时，`insert()`会隐含`inline=True`。这有助于修复一个 bug，即在支持的后端上，INSERT…FROM SELECT 结构会被错误地编译为“隐式返回”，这会导致在插入零行的情况下出现故障（因为隐式返回期望一行），以及在插入多行的情况下出现任意返回数据（例如，多行中的第一行）。对于具有多个参数集的 INSERT..VALUES，也应用了类似的更改；隐式 RETURNING 也不再为此语句发出。由于这两个结构处理可变数量的行，因此`ResultProxy.inserted_primary_key`访问器不适用。以前有一个文档注释，指出在 INSERT..FROM SELECT 中可能更喜欢`inline=True`，因为一些数据库不支持返回，因此无法进行“隐式”返回，但无论如何，INSERT…FROM SELECT 都不需要隐式返回。如果需要返回插入的数据的可变数量的结果行，则应使用常规的显式`Insert.returning()`。

    参考：[#3169](https://www.sqlalchemy.org/trac/ticket/3169)

+   **[sql] [增强]**

    现在可以构建实现`GenericTypeCompiler`的自定义方言，以便访问方法接收所属表达式对象的指示，如果有的话。大多数情况下接受关键字参数（例如，`**kw`）的访问方法将接收一个关键字参数`type_expression`，指的是类型所包含的表达式对象。对于 DDL 中的列，方言的编译器类可能还需要修改其`get_column_specification()`方法以支持此功能。如果`UserDefinedType.get_col_spec()`在其参数签名中提供了`**kw`，则还将接收`type_expression`。 

    参考：[#3074](https://www.sqlalchemy.org/trac/ticket/3074)

### 模式

+   **[模式] [特性]**

    `MetaData.create_all()`和`MetaData.drop_all()`的 DDL 生成系统已经增强，大多数情况下可以自动处理相互依赖的外键约束的情况；对于`ForeignKeyConstraint.use_alter`标志的需求大大减少。该系统还适用于没有提前命名的约束；只有在 DROP 的情况下，循环中涉及的约束至少需要一个名称。

    参见

    ForeignKeyConstraint 上的 use_alter 标志（通常）不再需要

    参考：[#3282](https://www.sqlalchemy.org/trac/ticket/3282)

+   **[模式] [功能]**

    添加了一个新的访问器`Table.foreign_key_constraints`来补充`Table.foreign_keys`集合，以及`ForeignKeyConstraint.referred_table`。

+   **[模式] [错误]**

    `CheckConstraint`构造现在支持包含`%(column_0_name)s`标记的命名约定；约束表达式会扫描列。此外，不包含`%(constraint_name)s`标记的检查约束的命名约定现在也适用于由`SchemaType`生成的约束，例如`Boolean`和`Enum`的约束；这在 0.9.7 中停止工作是因为[#3067](https://www.sqlalchemy.org/trac/ticket/3067)。

    参见

    命名 CHECK 约束

    为布尔值、枚举和其他模式类型配置命名

    参考：[#3067](https://www.sqlalchemy.org/trac/ticket/3067), [#3299](https://www.sqlalchemy.org/trac/ticket/3299)

### postgresql

+   **[postgresql] [功能]**

    添加了对使用`postgresql_concurrently`建立的 PostgreSQL 索引的`CONCURRENTLY`关键字的支持。感谢 Iuri de Silvio 的拉取请求。

    参见

    使用 CONCURRENTLY 的索引

    此更改也**回溯**至：0.9.9

+   **[postgresql] [功能] [pg8000]**

    使用 pg8000 驱动程序添加了“sane multi row count”支持，主要适用于在 ORM 中使用版本控制时。该功能基于使用 pg8000 1.9.14 或更高版本进行版本检测。Pull request 由 Tony Locke 提供。

    此更改还**回溯**到：0.9.8

+   **[postgresql] [feature]**

    为`ColumnOperators.match()`操作符添加了 kw 参数`postgresql_regconfig`，允许指定“reg config”参数以发出`to_tsquery()`函数。Pull request 由 Jonathan Vanasco 提供。

    此更改还**回溯**到：0.9.7

    参考：[#3078](https://www.sqlalchemy.org/trac/ticket/3078)

+   **[postgresql] [feature]**

    通过`JSONB`添加对 PostgreSQL JSONB 的支持。Pull request 由 Damian Dimmich 提供。

    此更改还**回溯**到：0.9.7

+   **[postgresql] [feature]**

    在使用 pg8000 DBAPI 时，添加了对 AUTOCOMMIT 隔离级别的支持。Pull request 由 Tony Locke 提供。

    此更改还**回溯**到：0.9.5

+   **[postgresql] [feature]**

    为 PostgreSQL `ARRAY`类型添加了一个新标志`ARRAY.zero_indexes`。当设置为`True`时，将在传递到数据库之前将所有数组索引值加一，从而允许 Python 风格的零基索引与 PostgreSQL 基索引之间更好地互操作。Pull request 由 Alexey Terentev 提供。

    此更改还**回溯**到：0.9.5

    参考：[#2785](https://www.sqlalchemy.org/trac/ticket/2785)

+   **[postgresql] [feature]**

    PG8000 方言现在支持`create_engine.encoding`参数，通过在连接上设置客户端编码，然后被 pg8000 拦截。Pull request 由 Tony Locke 提供。

+   **[postgresql] [feature]**

    添加了对 PG8000 的原生 JSONB 功能的支持。Pull request 由 Tony Locke 提供。

+   **[postgresql] [feature] [pypy]**

    添加了对 pypy 上使用 psycopg2cffi DBAPI 的支持。Pull request 由 shauns 提供。

    另请参阅

    `sqlalchemy.dialects.postgresql.psycopg2cffi`

    参考：[#3052](https://www.sqlalchemy.org/trac/ticket/3052)

+   **[postgresql] [feature]**

    添加对聚合函数应用的 FILTER 关键字的支持，由 PostgreSQL 9.4 支持。Pull request 由 Ilja Everilä提供。

    另请参阅

    PostgreSQL FILTER 关键字

+   **[postgresql] [feature]**

    已添加对物化视图和外部表的反射支持，以及对`Inspector.get_view_names()`中的物化视图的支持，并在 PostgreSQL 版本的 `Inspector` 上提供了一个新方法 `PGInspector.get_foreign_table_names()`。感谢 Rodrigo Menezes 提交的拉取请求。

    另请参阅

    PostgreSQL 方言反映物化视图、外部表

    参考：[#2891](https://www.sqlalchemy.org/trac/ticket/2891)

+   **[postgresql] [feature]**

    在通过 `Table` 构造渲染 DDL 时，已添加对 PG 表选项 TABLESPACE、ON COMMIT、WITH(OUT) OIDS 和 INHERITS 的支持。感谢 malikdiarra 提交的拉取请求。

    另请参阅

    PostgreSQL 表选项

    参考：[#2051](https://www.sqlalchemy.org/trac/ticket/2051)

+   **[postgresql] [feature]**

    新增了`PGInspector.get_enums()`方法，当在 PostgreSQL 中使用 inspector 时，将提供 ENUM 类型的列表。感谢 Ilya Pekelny 提交的拉取请求。

+   **[postgresql] [bug]**

    向 PG `HSTORE` 类型添加了 `hashable=False` 标志，这是为了允许 ORM 在请求混合列/实体列表中的 ORM 映射的 HSTORE 列时跳过尝试“哈希”它。感谢 Gunnlaugur Þór Briem 提交的补丁。

    此更改也已**回溯**至：0.9.5, 0.8.7

    参考：[#3053](https://www.sqlalchemy.org/trac/ticket/3053)

+   **[postgresql] [bug]**

    新增了一个新的“断开连接”消息“连接意外关闭”。这似乎与较新版本的 SSL 有关。感谢 Antti Haapala 提交的拉取请求。

    此更改也已**回溯**至：0.9.5, 0.8.7

+   **[postgresql] [bug]**

    修复了在使用 psycopg2 时与 ARRAY 类型一起支持 PostgreSQL UUID 类型的问题。现在 psycopg2 方言现在使用 psycopg2.extras.register_uuid() 钩子，以便始终将 UUID 值传递到/从 DBAPI 作为 UUID() 对象。尽管仍然遵守 `UUID.as_uuid` 标志，但在禁用时，我们需要将返回的 UUID 对象转换回字符串。

    此更改也已**回溯**至：0.9.9

    参考：[#2940](https://www.sqlalchemy.org/trac/ticket/2940)

+   **[postgresql] [bug]**

    在使用 psycopg2 2.5.4 或更高版本时，添加了对 `postgresql.JSONB` 数据类型的支持，该版本具有 JSONB 数据的本机转换，因此必须禁用 SQLAlchemy 的转换器；此外，还使用了新添加的 psycopg2 扩展 `extras.register_default_jsonb` 来建立通过 `json_deserializer` 参数传递给方言的 JSON 反序列化器。还修复了 PostgreSQL 集成测试，这些测试实际上并没有循环传输 JSONB 类型，而不是 JSON 类型。拉取请求由 Mateusz Susik 提供。

    此更改也**回溯**到：0.9.9

+   **[postgresql] [bug]**

    修复了在使用旧版本 psycopg2 < 2.4.3 时注册 HSTORE 类型时使用 “array_oid” 标志的问题，该版本不支持此标志，以及在 psycopg2 版本 < 2.5 上使用本机 json 序列化器钩子 “register_default_json” 与用户定义的 `json_deserializer` 一起使用，该版本不包括本机 json。

    此更改也**回溯**到：0.9.9

+   **[postgresql] [bug]**

    修复了 PostgreSQL 方言无法渲染在 `Index` 中的表绑定列之外的表达式的 bug；通常当 `text()` 构造是索引中的表达式之一时；或者如果其中一个或多个是这样的表达式，则可能会误解表达式列表。

    此更改也**回溯**到：0.9.9

    参考：[#3174](https://www.sqlalchemy.org/trac/ticket/3174)

+   **[postgresql] [bug]**

    重新审视了首次在 0.9.5 中修补的问题，显然 psycopg2 的 `.closed` 访问器并不像我们假设的那样可靠，因此我们已经添加了一个明确检查异常消息“SSL SYSCALL error: Bad file descriptor”和“SSL SYSCALL error: EOF detected”以检测断开连接的情况。我们将继续将 psycopg2 的 connection.closed 作为首要检查。

    此更改也**回溯**到：0.9.8

    参考：[#3021](https://www.sqlalchemy.org/trac/ticket/3021)

+   **[postgresql] [bug]**

    修复了 PostgreSQL JSON 类型无法持久化或以 JSON 编码的 `'null'` 而不是 SQL NULL 列值进行渲染的 bug。为支持此情况，更改如下：

    +   可以指定值 `null()`，这将始终导致语句中的 NULL 值。

    +   添加了一个新参数 `JSON.none_as_null`，当为 True 时表示 Python 的 `None` 值应该被持久化为 SQL NULL，而不是 JSON 编码的 `'null'`。

    对于除 psycopg2 外的其他 DBAPI，检索 NULL 作为 None 也已修复，即 pg8000。

    此更改也**回溯**到：0.9.8

    参考：[#3159](https://www.sqlalchemy.org/trac/ticket/3159)

+   **[postgresql] [bug]**

    DBAPI 错误的异常包装系统现在可以适应非标准的 DBAPI 异常，例如 psycopg2 TransactionRollbackError。这些异常现在将使用`sqlalchemy.exc`中最接近的可用子类引发，在 TransactionRollbackError 的情况下，是`sqlalchemy.exc.OperationalError`。

    此更改也**回溯**到：0.9.8

    参考：[#3075](https://www.sqlalchemy.org/trac/ticket/3075)

+   **[postgresql] [bug]**

    修复了`array`对象中的错误，在与普通 Python 列表的比较中未能使用正确的数组构造函数。感谢 Andrew 提供的拉取请求。

    此更改也**回溯**到：0.9.8

    参考：[#3141](https://www.sqlalchemy.org/trac/ticket/3141)

+   **[postgresql] [bug]**

    为函数添加了一个支持的`FunctionElement.alias()`方法，例如`func`构造。以前，该方法的行为未定义。当前行为模仿了 0.9.4 之前的行为，即将函数转换为具有给定别名的单列 FROM 子句，其中列本身被匿名命名。

    此更改也**回溯**到：0.9.8

    参考：[#3137](https://www.sqlalchemy.org/trac/ticket/3137)

+   **[postgresql] [bug] [pg8000]**

    修复了 0.9.5 中由新的 pg8000 隔离级别功能引入的错误，其中引擎级别的隔离级别参数在连接时会引发错误。

    此更改也**回溯**到：0.9.7

    参考：[#3134](https://www.sqlalchemy.org/trac/ticket/3134)

+   **[postgresql] [bug]**

    现在，在确定异常是否为“断开”错误时，将查看 psycopg2`.closed`访问器；理想情况下，这应该消除任何其他检查异常消息以检测断开的需要，但是我们将保留这些现有消息作为后备。这应该能够处理新的情况，例如“SSL EOF”条件。感谢 Dirk Mueller 提供的拉取请求。

    此更改也**回溯**到：0.9.5

    参考：[#3021](https://www.sqlalchemy.org/trac/ticket/3021)

+   **[postgresql] [bug]**

    当调用普通的`table.drop()`时，PostgreSQL `ENUM` 类型将发出 DROP TYPE 指令，假设对象没有直接与`MetaData`对象关联。为了适应在多个表之间共享枚举类型的用例，该类型应直接与`MetaData`对象关联；在这种情况下，类型仅在元数据级别创建，或者如果直接创建。一般来说，已经对创建/删除 PostgreSQL 枚举类型的规则进行了大幅改进。

    另请参阅

    ENUM 类型创建/删除规则的全面改进

    参考：[#3319](https://www.sqlalchemy.org/trac/ticket/3319)

+   **[postgresql] [错误]**

    `PGDialect.has_table()` 方法现在将针对`pg_catalog.pg_table_is_visible(c.oid)`进行查询，而不是在模式名称为 None 时测试精确的模式匹配；这样该方法也将显示出临时表的存在。请注意，这是一项行为更改，因为 PostgreSQL 允许非临时表悄悄地覆盖同名的现有临时表，因此这会改变`checkfirst`在这种不寻常情况下的行为。

    另请参阅

    PostgreSQL has_table() 现在适用于临时表

    参考：[#3264](https://www.sqlalchemy.org/trac/ticket/3264)

+   **[postgresql] [增强]**

    向 PostgreSQL 方言添加了一个新类型`OID`。虽然“oid”通常是 PG 中的私有类型，在现代版本中不会公开，但在某些 PG 使用情况下（如大对象支持）可能会公开这些类型，以及在一些用户报告的模式反射使用情况中可能会公开这些类型。

    此更改也已**回溯**至：0.9.5

    参考：[#3002](https://www.sqlalchemy.org/trac/ticket/3002)

### mysql

+   **[mysql] [功能]**

    MySQL 方言现在在所有情况下都使用 NULL / NOT NULL 渲染 TIMESTAMP，因此启用了`explicit_defaults_for_timestamp`标志的 MySQL 5.6.6 将允许 TIMESTAMP 在`nullable=False`时继续按预期工作。现有应用程序不受影响，因为 SQLAlchemy 一直为`nullable=True`的 TIMESTAMP 列发出 NULL。

    另请参阅

    MySQL TIMESTAMP 类型现在在所有情况下都渲染为 NULL / NOT NULL

    TIMESTAMP 列和 NULL

    参考：[#3155](https://www.sqlalchemy.org/trac/ticket/3155)

+   **[mysql] [功能]**

    将 “supports_unicode_statements” 标志更新为 True，对于 Python 2 下的 MySQLdb 和 Pymysql。这是指 SQL 语句本身，而不是参数，影响到使用非 ASCII 字符的表和列名等问题。这些驱动程序在现代版本中似乎都支持 Python 2 Unicode 对象而没有问题。

    参考：[#3121](https://www.sqlalchemy.org/trac/ticket/3121)

+   **[mysql] [change]**

    `gaerdbms` 方言已不再必要，并发出了弃用警告。 Google 现在建议直接使用 MySQLdb 方言。

    这个变更也**被回溯**到：0.9.9

    参考：[#3275](https://www.sqlalchemy.org/trac/ticket/3275)

+   **[mysql] [bug]**

    MySQL 错误 2014 “命令不同步” 现在似乎被作为 ProgrammingError 而不是 OperationalError 引发，在现代 MySQL-Python 版本中所有测试过的 MySQL 错误代码 “is disconnect” 现在都在 OperationalError 和 ProgrammingError 中进行检查。

    这个变更也**被回溯**到：0.9.7, 0.8.7

    参考：[#3101](https://www.sqlalchemy.org/trac/ticket/3101)

+   **[mysql] [bug]**

    修复了一个 bug，在索引的 `mysql_length` 参数上添加的列名需要具有相同的引号才能被识别。修复使引号变成可选，但也为那些使用此解决方法的人提供了旧的行为，以保持向后兼容性。

    这个变更也**被回溯**到：0.9.5, 0.8.7

    参考：[#3085](https://www.sqlalchemy.org/trac/ticket/3085)

+   **[mysql] [bug]**

    添加了对包含 KEY_BLOCK_SIZE 的索引的反射表的支持，使用等号。拉取请求由肖恩·麦克吉文提供。

    这个变更也**被回溯**到：0.9.5, 0.8.7

+   **[mysql] [bug]**

    在 MySQLdb 方言周围添加了一个版本检查，用于检查 ‘utf8_bin’ 校对，因为这在 MySQL 服务器 < 5.0 上失败。

    这个变更也**被回溯**到：0.9.9

    参考：[#3274](https://www.sqlalchemy.org/trac/ticket/3274)

+   **[mysql] [bug] [mysqlconnector]**

    自 MySQLconnector 版本 2.0 起，可能作为 python 3 合并的副作用，现在不再期望百分号（例如作为模运算符和其他）加倍，即使使用 “pyformat” 绑定参数格式（这个变更未被 MySQLconnector 记录）。方言现在在检测模运算符应该呈现为 `%%` 还是 `%` 时，检查了 py2k 和 mysqlconnector 小于版本 2.0。

    这个变更也**被回溯**到：0.9.8

+   **[mysql] [bug] [mysqlconnector]**

    现在对于 MySQLconnector 版本 2.0 及以上版本，Unicode SQL 被传递；对于 Py2k 和 MySQL < 2.0，字符串被编码。

    这个变更也**被回溯**到：0.9.8

+   **[mysql] [bug]**

    MySQL 方言现在支持在构造为`TypeDecorator`对象的类型上进行 CAST。

+   **[mysql] [bug]**

    当在 MySQL 方言上使用 `cast()` 时，如果 MySQL 不支持 CAST 的类型，则会发出警告；MySQL 只支持对部分数据类型进行 CAST。长期以来，SQLAlchemy 在 MySQL 的情况下仅省略不支持的类型的 CAST。虽然我们现在不想更改这一点，但我们会发出警告以表明已经发生了这种情况。当在旧版本的 MySQL（< 4）中使用 CAST 时，MySQL 完全不支持 CAST 时，也会发出警告，此时也会跳过 CAST。

    参考：[#3237](https://www.sqlalchemy.org/trac/ticket/3237)

+   **[mysql] [bug]**

    `SET` 类型已进行了彻底改版，不再假设空字符串或只有一个空字符串值的集合实际上是只有一个空字符串的集合；相反，默认情况下，这被视为空集。为了处理实际上希望将空值 `''` 包含为合法值的 `SET` 的持久性，添加了一个新的按位运算模式，通过 `SET.retrieve_as_bitwise` 标志启用，这将以位标志的位置持久化和检索值，使其不含糊。修复了对未原生转换 Unicode 的驱动程序配置的 Unicode 值的存储和检索。

    另请参阅

    MySQL SET 类型进行了彻底改版，以支持空集、Unicode、空值处理

    参考：[#3283](https://www.sqlalchemy.org/trac/ticket/3283)

+   **[mysql] [bug]**

    `ColumnOperators.match()` 操作现在被处理，不再严格假定返回类型为布尔值；现在返回一个名为 `MatchType` 的 `Boolean` 子类。当在 Python 表达式中使用时，该类型仍将产生布尔行为，但方言可以在结果时重写其行为。在 MySQL 的情况下，虽然 MATCH 运算符通常在表达式中的布尔上下文中使用，但如果实际上查询匹配表达式的值，则会返回浮点值；此值与 SQLAlchemy 的基于 C 的布尔处理器不兼容，因此 MySQL 的结果集行为现在遵循 `Float` 类型的行为。还添加了一个新的运算符对象 `notmatch_op`，以更好地允许方言定义匹配操作的否定。

    另请参阅

    match() 运算符现在返回与 MySQL 浮点返回值兼容的不可知 MatchType

    参考：[#3263](https://www.sqlalchemy.org/trac/ticket/3263)

+   **[mysql] [bug]**

    MySQL 布尔符号“true”、“false”再次有效。0.9 中的更改[#2682](https://www.sqlalchemy.org/trac/ticket/2682) 禁止了 MySQL 方言在“IS”/“IS NOT”的上下文中使用“true”和“false”符号，但 MySQL 支持这种语法，即使它没有布尔类型。MySQL 仍然是“非本地布尔”，但 `true()` 和 `false()` 符号再次生成关键字“true”和“false”，因此像 `column.is_(true())` 这样的表达式在 MySQL 上再次有效。

    另请参阅

    MySQL 布尔符号“true”、“false”再次有效

    参考：[#3186](https://www.sqlalchemy.org/trac/ticket/3186)

+   **[mysql] [bug]**

    MySQL 方言现在将禁用 `ConnectionEvents.handle_error()` 事件，以防止其内部使用的语句触发该事件以检测表是否存在。这是通过使用一个执行选项 `skip_user_error_events` 来实现的，该选项在该执行范围内禁用了处理错误事件。这样，重写异常的用户代码不需要担心 MySQL 方言或其他偶尔需要捕获 SQLAlchemy 特定异常的方言。

+   **[mysql] [bug]**

    将 MySQLconnector 的 “raise_on_warnings” 默认值更改为 False。出于某种原因，这个值被设置为 True。不幸的是，“buffered” 标志必须保持为 True，因为 MySQLconnector 不允许关闭游标，除非所有结果都被完全获取。

    参考：[#2515](https://www.sqlalchemy.org/trac/ticket/2515)

### sqlite

+   **[sqlite] [feature]**

    在 SQLite 上增加了对部分索引（例如带有 WHERE 子句）的支持。感谢 Kai Groner 提交的拉取请求。

    另请参阅

    部分索引

    此更改也被**回溯**到：0.9.9

+   **[sqlite] [feature]**

    添加了一个新的 SQLite 后端用于 SQLCipher 后端。该后端使用 pysqlcipher Python 驱动程序提供加密的 SQLite 数据库，该驱动程序与 pysqlite 驱动程序非常相似。

    另请参阅

    `pysqlcipher`

    此更改也被**回溯**到：0.9.9

+   **[sqlite] [bug]**

    使用附加数据库文件从 UNION 中选择时，pysqlite 驱动程序在 cursor.description 中报告列名为'dbname.tablename.colname'，而不是' tablename.colname'，如其对于 UNION 通常做的（请注意，对于两者来说，它应该只是'colname'，但我们绕过了它）。这里的列转换逻辑已调整为检索最右边的标记，而不是第二个标记，因此在两种情况下都有效。Tony Roberts 提供了解决方法。

    此更改也**被回溯**至：0.9.8

    参考：[#3211](https://www.sqlalchemy.org/trac/ticket/3211)

+   **[sqlite] [bug]**

    修复了一个 SQLite 连接重写问题，在此问题中，嵌入为标量子查询的子查询（例如在 IN 中），如果相同的表在子查询中存在，并且在封闭查询中存在，例如在连接的继承场景中，将从封闭查询中接收不合适的替换。

    此更改也**被回溯**至：0.9.7

    参考：[#3130](https://www.sqlalchemy.org/trac/ticket/3130)

+   **[sqlite] [bug]**

    在 SQLite 中，唯一约束和外键约束现在完全反映在命名和未命名的情况下。以前，外键名称被忽略，未命名的唯一约束被跳过。感谢乔恩·尼尔森（Jon Nelson）的协助。

    参考：[#3244](https://www.sqlalchemy.org/trac/ticket/3244)，[#3261](https://www.sqlalchemy.org/trac/ticket/3261)

+   **[sqlite] [bug]**

    当使用 `DATE`、`TIME` 或 `DATETIME` 类型时，SQLite 方言，如果给出的 `storage_format` 仅呈现数字，则将在 DDL 中将类型呈现为 `DATE_CHAR`、`TIME_CHAR` 和 `DATETIME_CHAR`，以便尽管值中缺少字母字符，但列仍将传递“文本亲和性”。通常情况下，这是不需要的，因为默认存储格式中的文本值已经暗示了文本。

    另请参阅

    日期和时间类型

    参考：[#3257](https://www.sqlalchemy.org/trac/ticket/3257)

+   **[sqlite] [bug]**

    SQLite 现在支持从临时表中反射唯一约束；以前，这将导致 TypeError。拉取请求由 Johannes Erdfelt 提供。

    另请参阅

    SQLite/Oracle 有不同的临时表/视图名称报告方法 - 关于 SQLite 临时表和视图反射的更改。

    参考：[#3203](https://www.sqlalchemy.org/trac/ticket/3203)

+   **[sqlite] [bug]**

    添加了`Inspector.get_temp_table_names()`和`Inspector.get_temp_view_names()`；目前，只有 SQLite 和 Oracle 方言支持这些方法。临时表和视图名称的返回已从 SQLite 和 Oracle 版本的`Inspector.get_table_names()`和`Inspector.get_view_names()`中**移除**；其他数据库后端不支持此信息（如 MySQL），操作范围也不同，因为表可以是会话本地的，通常不支持远程模式中的表。 

    参见

    SQLite/Oracle 有用于报告临时表/视图名称的不同方法

    参考：[#3204](https://www.sqlalchemy.org/trac/ticket/3204)

### mssql

+   **[mssql] [功能]**

    为 SQL Server 2008 启用了“多值插入”。感谢 Albert Cervin 提供的拉取请求。还扩展了对“IDENTITY INSERT”模式的检查，以包括当标识键出现在语句的 VALUEs 子句中时。

    此更改也被**回溯**到：0.9.7

+   **[mssql] [功能]**

    SQL Server 2012 现在推荐对于大文本/二进制类型使用 VARCHAR(max)、NVARCHAR(max)、VARBINARY(max)。MSSQL 方言现在会根据版本检测以及新的`deprecate_large_types`标志来尊重这一点。

    参见

    大文本/二进制类型弃用

    参考：[#3039](https://www.sqlalchemy.org/trac/ticket/3039)

+   **[mssql] [更改]**

    当使用 pyodbc 时，SQL Server 的基于主机名的连接格式将不再指定默认的“驱动程序名称”，如果缺少此项将发出警告。SQL Server 的最佳驱动程序名称经常变化，并且是每个平台独立的，因此基于主机名的连接需要指定此项。优先使用基于 DSN 的连接。

    参见

    基于主机名的 SQL Server 连接需要 PyODBC 驱动程序名称

    参考：[#3182](https://www.sqlalchemy.org/trac/ticket/3182)

+   **[mssql] [错误]**

    在“SET IDENTITY_INSERT”语句中添加了语句编码，当明确地向标识列插入一个 INSERT 时，以支持在不支持 unicode 语句的驱动程序（如 pyodbc + unix + py2k）上使用非 ascii 表标识符。

    此更改也被**回溯**到：0.9.7, 0.8.7

+   **[mssql] [错误]**

    在 SQL Server pyodbc 方言中，修复了 `description_encoding` 方言参数的实现，当未显式设置时，会导致在包含其他编码名称的结果集中，cursor.description 无法正确解析的情况。未来不应该需要此参数。

    此更改也被**回溯**到：0.9.7, 0.8.7

    参考：[#3091](https://www.sqlalchemy.org/trac/ticket/3091)

+   **[mssql] [bug]**

    修复了 pymssql 方言中版本字符串检测的问题，以便与 Microsoft SQL Azure 一起工作，后者将“SQL Server”更改为“SQL Azure”。

    此更改也被**回溯**到：0.9.8

    参考：[#3151](https://www.sqlalchemy.org/trac/ticket/3151)

+   **[mssql] [bug]**

    修订了用于确定当前默认模式名称的查询，使用 `database_principal_id()` 函数与 `sys.database_principals` 视图结合使用，以便我们可以独立于正在进行的登录类型（例如，SQL Server、Windows 等）确定默认模式。

    此更改也被**回溯**到：0.9.5

    参考：[#3025](https://www.sqlalchemy.org/trac/ticket/3025)

### oracle

+   **[oracle] [feature]**

    增加了对 cx_oracle 连接到特定服务名称的支持，而不是 tns 名称，通过在 URL 中传递 `?service_name=<name>`。感谢 Sławomir Ehlert 提交的拉取请求。

+   **[oracle] [feature]**

    Oracle 表、索引的新 DDL 功能：COMPRESS、BITMAP。补丁由 Gabor Gombas 提供。

+   **[oracle] [feature]**

    增加了对 Oracle 下 CTE 的支持。这包括对别名语法的一些调整，以及一个新的 CTE 功能 `CTE.suffix_with()`，用于向 CTE 添加特殊的 Oracle 特定指令。

    另请参阅

    改进了 Oracle 中 CTE 的支持

    参考：[#3220](https://www.sqlalchemy.org/trac/ticket/3220)

+   **[oracle] [feature]**

    增加了对 Oracle 表选项 ON COMMIT 的支持。

+   **[oracle] [bug]**

    修复了 Oracle 方言中长期存在的 bug，即以数字开头的绑定参数名称不会被引用，因为 Oracle 不喜欢绑定参数名称中有数字。

    此更改也被**回溯**到：0.9.8

    参考：[#2138](https://www.sqlalchemy.org/trac/ticket/2138)

+   **[oracle] [bug] [tests]**

    修复了 Oracle 方言测试套件中的一个 bug，在一个测试中，假定‘username’在数据库 URL 中，尽管这可能并非如此。

    此更改也被**回溯**到：0.9.7

    参考：[#3128](https://www.sqlalchemy.org/trac/ticket/3128)

+   **[oracle] [bug]**

    当在 `Select.with_hint()` ��法中使用 `%(name)s` 标记引用别名时，别名将被正确引用。以前，Oracle 后端尚未实现此引用。

### 测试

+   **[tests] [bug]**

    修复了“python setup.py test”未正确调用 distutils 的 bug，并且错误将在测试套件结束时发出。

    此更改也被**回溯**到：0.9.7

+   **[测试] [错误] [py3k]**

    修正了一些涉及`imp`模块和 Python 3.3 或更高版本的弃用警告，在运行测试时。感谢 Matt Chisholm 提供的拉取请求。

    此更改也被**回溯**到：0.9.5

    参考：[#2830](https://www.sqlalchemy.org/trac/ticket/2830)

### 杂项

+   **[特性] [扩展]**

    添加了一个新的扩展套件`sqlalchemy.ext.baked`。这个简单但不寻常的系统可以大大节省 Python 在构建和处理 orm `Query`对象时的开销，从查询构建到渲染字符串 SQL 语句。

    另请参阅

    烘焙查询

    参考：[#3054](https://www.sqlalchemy.org/trac/ticket/3054)

+   **[特性] [扩展]**

    `sqlalchemy.ext.automap`扩展现在会自动在检测到包含一个或多个非空列的外键的一对多关系/反向引用上设置`cascade="all, delete-orphan"`。在这种情况下，此参数存在于传递给`generate_relationship()`的关键字中，仍然可以被覆盖。此外，如果`ForeignKeyConstraint`为非空或可空列指定了`ondelete="CASCADE"`或`ondelete="SET NULL"`，则还会将参数`passive_deletes=True`添加到关系中。请注意，并非所有后端都支持 ondelete 的反射，但支持反射的后端包括 PostgreSQL 和 MySQL。

    参考：[#3210](https://www.sqlalchemy.org/trac/ticket/3210)

+   **[错误] [声明式]**

    当访问时，`__mapper_args__`字典是从声明性 mixin 或抽象类复制过来的，因此声明性本身对此字典所做的修改不会与其他映射的冲突。关于`version_id_col`和`polymorphic_on`参数，字典被修改，用本地类/表正式映射的列替换其中的列。

    此更改也被**回溯**到：0.9.5, 0.8.7

    参考：[#3062](https://www.sqlalchemy.org/trac/ticket/3062)

+   **[错误] [扩展]**

    修复了可变扩展中的一个错误，`MutableDict`未为`setdefault()`字典操作报告更改事件。

    此更改也被**回溯**到：0.9.5, 0.8.7

    参考：[#3051](https://www.sqlalchemy.org/trac/ticket/3051), [#3093](https://www.sqlalchemy.org/trac/ticket/3093)

+   **[错误] [扩展]**

    修复了`MutableDict.setdefault()`未返回现有值或新值的错误（此错误未在任何 0.8 版本中发布）。感谢 Thomas Hervé提供的拉取请求。

    此更改也被**回溯**到：0.9.5, 0.8.7

    参考：[#3051](https://www.sqlalchemy.org/trac/ticket/3051), [#3093](https://www.sqlalchemy.org/trac/ticket/3093)

+   **[错误] [扩展] [py3k]**

    修复了关联代理列表类在 Py3K 下无法正确解释切片的错误。感谢 Gilles Dartiguelongue 提供的拉取请求。

    此更改也被**回溯**到：0.9.9

+   **[错误] [声明式]**

    修复了在一些奇特的最终用户设置中观察到的不太可能的竞争条件，其中在声明式中检查“重复类名”时会遇到与其他被移除的类相关的未完全清理的弱引用；此处的检查现在确保弱引用在进一步调用之前仍然引用一个对象。

    此更改也被**回溯**到：0.9.8

    参考：[#3208](https://www.sqlalchemy.org/trac/ticket/3208)

+   **[错误] [扩展]**

    修复了有序列表中的错误，其中在集合替换事件期间，如果将`reorder_on_append`标志设置为 True，则项目的顺序会被打乱。修复确保有序列表仅影响与对象明确关联的列表。

    此更改也被**回溯**到：0.9.8

    参考：[#3191](https://www.sqlalchemy.org/trac/ticket/3191)

+   **[错误] [扩展]**

    修复了`MutableDict`未实现`update()`字典方法，因此未捕获更改的错误。感谢 Matt Chisholm 提供的拉取请求。

    此更改也被**回溯**到：0.9.8

+   **[错误] [扩展]**

    修复了自定义`MutableDict`的子类在“强制”操作中不会显示，并且会返回一个普通的`MutableDict`的错误。感谢 Matt Chisholm 提供的拉取请求。

    此更改也被**回溯**到：0.9.8

+   **[错误] [池]**

    修复了连接池日志记录中的错误，其中“连接已检出”调试日志消息不会在使用`logging.setLevel()`设置日志记录时发出，而不是使用`echo_pool`标志。已添加用于断言此日志记录的测试。这是在 0.9.0 中引入的一个回归。

    此更改也被**回溯**到：0.9.8

    参考：[#3168](https://www.sqlalchemy.org/trac/ticket/3168)

+   **[错误] [声明式]**

    修复了当声明式`__abstract__`标志未被区分为实际值`False`时的错误。`__abstract__`标志需要在被测试的级别实际评估为 True。

    这个更改也被**回溯**到：0.9.7

    参考：[#3097](https://www.sqlalchemy.org/trac/ticket/3097)

+   **[错误] [测试套件]**

    在公共测试套件中，从不太受支持的`Text`更改为使用`String(40)`在`StringTest.test_literal_backslashes`中。Pullreq 由 Jan 提供。

    这个更改也被**回溯**到：0.9.5

+   **[已移除]**

    Drizzle 方言已从核心中移除；现在作为[sqlalchemy-drizzle](https://bitbucket.org/zzzeek/sqlalchemy-drizzle)提供，这是一个独立的第三方方言。该方言仍然几乎完全基于 SQLAlchemy 中存在的 MySQL 方言。

    另请参阅

    Drizzle 方言现在是外部方言
