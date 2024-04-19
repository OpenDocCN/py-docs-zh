# 1.4 更新日志

> 原文：[`docs.sqlalchemy.org/en/20/changelog/changelog_14.html`](https://docs.sqlalchemy.org/en/20/changelog/changelog_14.html)

本文详细介绍了 1.4 版本中进行的单个问题级别的更改。有关 1.4 中的新内容的叙述性概述，请参阅 SQLAlchemy 1.4 有什么新功能？。

## 1.4.53

无发布日期

## 1.4.52

发布日期：2024 年 3 月 4 日

### orm

+   **[orm] [bug]**

    修复了 ORM 中 `with_loader_criteria()` 不会应用到 `Select.join()` 的 bug，其中 ON 子句被给定为普通的 SQL 比较，而不是作为关系目标或类似的东西。

    这是在 2.0 版本中修复的同一问题的回溯，针对 2.0.22。

    参考：[#10365](https://www.sqlalchemy.org/trac/ticket/10365)

## 1.4.51

发布日期：2024 年 1 月 2 日

### orm

+   **[orm] [bug]**

    改进了首次在版本 0.9.8 中实施的修复项，该修复项最初在 [#3208](https://www.sqlalchemy.org/trac/ticket/3208) 中发布，其中声明性内部使用的类的注册表可能在个别映射类在同时进行垃圾回收而新的映射类正在构造时出现竞态条件的情况下，发生某些测试套件配置或动态类创建环境中可能发生的情况。除了已经添加的弱引用检查外，还首先复制正在迭代的项目列表，以避免“在迭代时更改列表”错误。拉取请求由 Yilei Yang 提供。

    参考：[#10782](https://www.sqlalchemy.org/trac/ticket/10782)

### asyncio

+   **[asyncio] [bug]**

    修复了异步连接池的关键问题，其中调用 `AsyncEngine.dispose()` 会产生一个新的连接池，但没有完全重新建立使用 asyncio 兼容互斥锁的情况，导致在使用并发功能如 `asyncio.gather()` 时，在 asyncio 上下文中使用普通的 `threading.Lock()` 导致死锁。

    参考：[#10813](https://www.sqlalchemy.org/trac/ticket/10813)

### mysql

+   **[mysql] [bug]**

    修复了在使用旧于 1.0 版本的 PyMySQL 的池预先 ping 时由修复票据 [#10492](https://www.sqlalchemy.org/trac/ticket/10492) 引入的回归问题。

    参考：[#10650](https://www.sqlalchemy.org/trac/ticket/10650)

## 1.4.50

发布日期：2023 年 10 月 29 日

### orm

+   **[orm] [bug]**

    修复了某些形式的 ORM “注解” 无法对使用 `Select.join()` 进行关系目标的子查询进行的问题。这些注解在特殊情况下使用子查询时使用，例如在 `PropComparator.and_()` 和其他 ORM 特定情况下。

    参考资料：[#10223](https://www.sqlalchemy.org/trac/ticket/10223)

### sql

+   **[sql] [错误]**

    修复了在某些情况下，使用 `literal_execute=True` 时多次使用相同的绑定参数会由于迭代问题导致渲染错误值的问题。

    参考资料：[#10142](https://www.sqlalchemy.org/trac/ticket/10142)

+   **[sql] [错误]**

    修复了对 `Column` 或其他 `ColumnElement` 的反序列化失败无法恢复正确的“比较器”对象的基本问题，该对象用于生成特定于类型对象的 SQL 表达式。

    参考资料：[#10213](https://www.sqlalchemy.org/trac/ticket/10213)

### 模式

+   **[模式] [错误]**

    修改了仅对 Oracle 后端有效的 `Identity.order` 参数的渲染，该参数是 `Sequence` 和 `Identity` 的一部分，不再对诸如 PostgreSQL 等其他后端有效。未来的版本将重新命名 `Identity.order`、`Sequence.order` 和 `Identity.on_null` 参数，使用 Oracle 特定的名称，废弃旧名称，这些参数仅适用于 Oracle。

    参考资料：[#10207](https://www.sqlalchemy.org/trac/ticket/10207)

### mysql

+   **[mysql] [用例]**

    更新了 aiomysql 方言，因为该方言似乎又得到了维护。将其重新添加到使用版本 0.2.0 进行 ci 测试。

+   **[mysql] [错误]**

    修复了 MySQL“预先 ping”例程中的新不兼容性，其中传递给`connection.ping()`的`False`参数，旨在禁用不需要的“自动重新连接”功能，在 MySQL 驱动程序和后端中已弃用，并且正在为某些版本的 MySQL 原生客户端驱动程序生成警告。对于 mysqlclient，它已被删除，而对于 PyMySQL 和基于 PyMySQL 的驱动程序，该参数将在某个时候被弃用和删除，因此 API 内省用于未来防范这些不同的删除阶段。

    参考：[#10492](https://www.sqlalchemy.org/trac/ticket/10492)

### mssql

+   **[mssql] [bug] [reflection]**

    修复了对具有大型身份起始值（超过 18 位数）的 bigint 列的身份列反射将失败的问题。

    参考：[#10504](https://www.sqlalchemy.org/trac/ticket/10504)

## 1.4.49

发布日期：2023 年 7 月 5 日

### platform

+   **[platform] [usecase]**

    改进兼容性以完全与 Python 3.12 兼容

### sql

+   **[sql] [bug]**

    修复了使用“flags”时`ColumnOperators.regexp_match()`无法生成“稳定”缓存密钥的问题，即缓存密钥每次都会更改，导致缓存污染。对于带有标志和实际替换表达式的`ColumnOperators.regexp_replace()`也存在相同的问题。现在，标志被表示为固定的修改器字符串，呈现为安全字符串，而不是绑定参数，并且替换表达式在“二进制”元素的主要部分内建立，以便生成适当的缓存密钥。

    请注意，作为此更改的一部分，`ColumnOperators.regexp_match.flags`和`ColumnOperators.regexp_replace.flags`已修改为仅呈现为文字字符串，而以前它们呈现为完整的 SQL 表达式，通常是绑定参数。这些参数应始终作为纯 Python 字符串传递，而不是作为 SQL 表达式构造；不应该预期在实践中使用 SQL 表达式构造用于此参数，因此这是一个不兼容的更改。

    此更改还修改了生成的表达式的内部结构，用于带有或不带有标志的`ColumnOperators.regexp_replace()`以及带有标志的`ColumnOperators.regexp_match()`。第三方方言可能已经实现了自己的正则表达式实现（在搜索中找不到这样的方言，所以预计影响较小），它们需要调整结构的遍历以适应。

    参考：[#10042](https://www.sqlalchemy.org/trac/ticket/10042)

+   **[sql] [bug]**

    修复了一个在主要是内部使用的`CacheKey`结构中的问题，其中`__ne__()`运算符没有被正确实现，导致比较`CacheKey`实例时得到荒谬的结果。

### extensions

+   **[extensions] [bug]**

    修复了与 mypy 1.4 一起使用的 mypy 插件中的问题。

## 1.4.48

发布日期：2023 年 4 月 30 日

### orm

+   **[orm] [bug]**

    修复了一个关键的缓存问题，其中`aliased()`和`hybrid_property()`表达式组合的组合将导致缓存键不匹配，从而导致缓存键保留了实际的`aliased()`对象，同时又不匹配等价构造的缓存键，填满了缓存。

    参考：[#9728](https://www.sqlalchemy.org/trac/ticket/9728)

+   **[orm] [bug]**

    修复了一个 bug，在各种 ORM 特定的 getter 函数（比如`ORMExecuteState.is_column_load`、`ORMExecuteState.is_relationship_load`、`ORMExecuteState.loader_strategy_path`等）中，如果 SQL 语句本身是“复合选择”（如 UNION），则会抛出`AttributeError`。

    参考：[#9634](https://www.sqlalchemy.org/trac/ticket/9634)

+   **[orm] [bug]**

    修复了当使用“关联到别名类”的功能并且在加载器中指定了一个递归的急加载器，比如 `lazy="selectinload"`，与另一个急加载器在相反的一侧结合使用时可能出现的无限循环。循环检查已修复以包括别名类关系。

    引用：[#9590](https://www.sqlalchemy.org/trac/ticket/9590)

## 1.4.47

发布日期：2023 年 3 月 18 日

### sql

+   **[sql] [bug]**

    修复了使用`Update.values()`方法中与列相同名称的`bindparam()`的错误/回归，以及`Insert.values()`方法中与列相同名称的`bindparam()`的错误/回归，仅在 2.0 版本中会在某些情况下静默地失败，不会遵守呈现参数的 SQL 表达式，而是用同名的新参数替换表达式并丢弃 SQL 表达式的任何其他元素，比如 SQL 函数等。具体情况是针对 ORM 实体而不是普通的`Table` 实例构建的语句，但如果语句在使用时被调用了一个`Session`或一个`Connection`，则会发生。

    `Update` 的一部分问题既出现在 2.0 版本中又出现在 1.4 版本中，并且被回溯到 1.4 版本。

    引用：[#9075](https://www.sqlalchemy.org/trac/ticket/9075)

+   **[sql] [bug]**

    修复了`CreateSchema`和`DropSchema`DDL 构造的字符串化，当没有方言时会导致 `AttributeError`。

    引用：[#7664](https://www.sqlalchemy.org/trac/ticket/7664)

+   **[sql] [bug]**

    修复了临界的 SQL 缓存问题，其中使用 `Operators.op()` 自定义运算符函数不会生成适当的缓存键，导致 SQL 缓存的效果降低。

    引用：[#9506](https://www.sqlalchemy.org/trac/ticket/9506)

### mypy

+   **[mypy] [bug]**

    调整了 mypy 插件，以适应在使用 SQLAlchemy 1.4 时可能进行的一些针对问题#236 sqlalchemy2-stubs 的更改。这些更改在 SQLAlchemy 2.0 内保持同步。这些更改还与旧版本的 sqlalchemy2-stubs 兼容。

+   **[mypy] [bug]**

    修复了 mypy 插件中的崩溃，该崩溃可能发生在 1.4 和 2.0 版本上，如果使用一个装饰器来引用一个表达式中的装饰器（例如`@Backend.mapper_registry.mapped`），该表达式具有两个以上的组件，则会发生崩溃。现在会忽略这种情况；使用插件时，装饰器表达式需要是两个组件（即`@reg.mapped`）。

    参考文献：[#9102](https://www.sqlalchemy.org/trac/ticket/9102)

### postgresql

+   **[postgresql] [bug]**

    添加了对 asyncpg 方言的支持，以在可用时返回`cursor.rowcount`值以用于 SELECT 语句。虽然这不是`cursor.rowcount`的典型用法，但是其他 PostgreSQL 方言通常提供此值。拉取请求由 Michael Gorven 提供。

    参考文献：[#9048](https://www.sqlalchemy.org/trac/ticket/9048)

### mysql

+   **[mysql] [usecase]**

    添加了对 MySQL 索引反射的支持，以正确反映以前被忽略的`mysql_length`字典。

    参考文献：[#9047](https://www.sqlalchemy.org/trac/ticket/9047)

### mssql

+   **[mssql] [bug]**

    修复了一个 bug，在此 bug 中，使用方括号给出的模式名称，但名称内没有点，例如`Table.schema`的参数，将不会在 SQL Server 方言的上下文中解释为解释为标记定界符的文档化行为，首次在#2626 中添加，当在反射操作中引用模式名称时。有关#2626 行为的最初假设是，只有在存在点时，方括号的特殊解释才是重要的，但是在实践中，由于这些不是常规或定界标识符中的有效字符，因此在所有 SQL 渲染操作中都不包括方括号作为标识符名称的一部分。拉取请求由 Shan 提供。

    参考文献：[#9133](https://www.sqlalchemy.org/trac/ticket/9133)

### oracle

+   **[oracle] [bug]**

    将`ROWID`添加到反射类型中，因为此类型可能在“CREATE TABLE”语句中使用。

    参考文献：[#5047](https://www.sqlalchemy.org/trac/ticket/5047)

## 1.4.46

发布日期：2023 年 1 月 3 日

### 一般

+   **[general] [change]**

    现在，当`SQLALCHEMY_WARN_20`环境变量未设置时，首次发出任何 SQLAlchemy 2.0 弃用警告时，将发出新的弃用“超级警告”。警告至多发出一次，然后设置一个布尔值以防止其再次发出。

    此弃用警告旨在通知未在其要求文件中设置适当约束的用户，阻止对意外 SQLAlchemy 2.0 升级的惊喜，并提醒 SQLAlchemy 2.0 升级流程已经可用，因为预计很快将发布第一个完整的 2.0 版本。可以通过设置环境变量 `SQLALCHEMY_SILENCE_UBER_WARNING` 为 `"1"` 来关闭弃用警告。

    另请参阅

    SQLAlchemy 2.0 - 主要迁移指南

    参考：[#8983](https://www.sqlalchemy.org/trac/ticket/8983)

+   **[general] [bug]**

    修复了基本兼容模块调用 `platform.architecture()` 来检测某些系统属性的问题，这导致针对一些情况下不可用的系统级 `file` 调用发生了过度广泛的系统调用，包括在某些安全环境配置中。

    参考：[#8995](https://www.sqlalchemy.org/trac/ticket/8995)

### orm

+   **[orm] [bug]**

    修复了用于 DML 语句（如 `Update` 和 `Delete`）的内部 SQL 遍历中的问题，该问题可能会导致与 ORM 更新/删除功能一起使用 lambda 语句时出现特定问题，以及其他潜在问题。

    参考：[#9033](https://www.sqlalchemy.org/trac/ticket/9033)

### engine

+   **[engine] [bug]**

    修复了连接池中长期存在的竞态条件，该条件可能在使用 eventlet/gevent 的猴子补丁方案以及使用 eventlet/gevent `Timeout` 条件时发生，并且连接池检出由于超时而中断时未能清理失败状态，导致底层连接记录以及有时数据库连接本身“泄漏”，使得连接池处于无效状态，其中某些条目无法访问。该问题首次在 SQLAlchemy 1.2 中被识别并修复，针对 [#4225](https://www.sqlalchemy.org/trac/ticket/4225)，然而，在该修复中检测到的故障模式未能适应于 `BaseException`，而不是 `Exception`，这导致无法捕获 eventlet/gevent `Timeout`。此外，还发现并加固了初始连接池连接中的一个块，使用了 `BaseException` -> “清理失败连接” 块，以适应此位置中的相同条件。非常感谢 Github 用户 @niklaus 在识别和描述此复杂问题方面的坚持努力。

    参考：[#8974](https://www.sqlalchemy.org/trac/ticket/8974)

### sql

+   **[sql] [bug]**

    添加了参数 `FunctionElement.column_valued.joins_implicitly`, 这在使用表值或列值函数时防止“笛卡尔积”警告时非常有用。此参数已经为 `FunctionElement.table_valued()` 在 [#7845](https://www.sqlalchemy.org/trac/ticket/7845) 中引入，但未能为 `FunctionElement.column_valued()` 添加。

    参考：[#9009](https://www.sqlalchemy.org/trac/ticket/9009)

+   **[sql] [错误]**

    修复了 SQL 编译失败的 bug（2.0 中的断言失败，1.4 中的 NoneType 错误），当使用的表达式的类型包括 `TypeEngine.bind_expression()`，在与 `literal_binds` 编译器参数一起使用时处于“扩展”（即“IN”）参数的上下文中时。

    参考：[#8989](https://www.sqlalchemy.org/trac/ticket/8989)

+   **[sql] [错误]**

    修复了 lambda SQL 功能中的问题，其中文字值的计算类型不会考虑“比较类型”的类型强制转换规则，导致 SQL 表达式（例如与`JSON`元素的比较等）缺乏类型信息。

    参考：[#9029](https://www.sqlalchemy.org/trac/ticket/9029)

### postgresql

+   **[postgresql] [用例]**

    添加了 PostgreSQL 类型 `MACADDR8`。Pull 请求由 Asim Farooq 提供。

    参考：[#8393](https://www.sqlalchemy.org/trac/ticket/8393)

+   **[postgresql] [错误]**

    修复了 PostgreSQL `Insert.on_conflict_do_update.constraint` 参数接受 `Index` 对象的 bug，然而，它不会将此索引展开为其各个索引表达式，而是将其名称渲染到 ON CONFLICT ON CONSTRAINT 子句中，这不被 PostgreSQL 接受；“约束名”形式只接受唯一或排除约束名。该参数仍然接受索引，但现在将其展开为其组成表达式以进行渲染。

    参考：[#9023](https://www.sqlalchemy.org/trac/ticket/9023)

### sqlite

+   **[sqlite] [错误]**

    修复了 1.4.45 中对 SQLite 部分索引的反射支持引起的回归问题，该问题是由于早期版本的 SQLite（可能是 3.8.9 之前的版本）中的`index_list` pragma 命令未返回当前预期的列数，导致在反射表和索引时引发异常。

    参考：[#8969](https://www.sqlalchemy.org/trac/ticket/8969)

### 测试

+   **[测试] [错误]**

    修复了 tox.ini 文件中的问题，其中 tox 4.0 系列对“passenv”的格式进行了更改，导致 tox 无法正确运行，特别是在 tox 4.0.6 中引发错误。

+   **[测试] [错误]**

    为第三方方言添加了新的排除规则，称为`unusual_column_name_characters`，可以关闭不支持带有点、斜杠或百分号等特殊字符的列名的第三方方言，即使名称已经正确引用。

    参考：[#9002](https://www.sqlalchemy.org/trac/ticket/9002)

## 1.4.45

发布日期：2022 年 12 月 10 日

### orm

+   **[orm] [错误]**

    修复了`Session.merge()`无法保留使用`relationship.viewonly`参数指示的关系属性的当前加载内容的错误，从而破坏了使用`Session.merge()`从缓存和其他类似技术中拉取完全加载的对象的策略。在相关变更中，修复了一个问题，即包含已加载的关系但在映射上仍配置为`lazy='raise'`的对象在传递给`Session.merge()`时会失败；在合并过程中暂停了对“raise”的检查，假定`Session.merge.load`参数保持其默认值`True`。

    总的来说，这是对 1.4 系列中引入的一项变更的行为调整，截至[#4994](https://www.sqlalchemy.org/trac/ticket/4994)，该变更将“merge”从默认应用于“viewonly”关系的级联集中移除。由于“viewonly”关系在任何情况下都不会持久化，允许它们的内容在“merge”期间传输不会影响目标对象的持久化行为。这使得`Session.merge()`能够正确地满足其中一个用例，即向`Session`中添加在其他地方加载的对象，通常是为了从缓存中恢复。

    参考：[#8862](https://www.sqlalchemy.org/trac/ticket/8862)

+   **[orm] [错误]**

    修复了`with_expression()`中的问题，在这种情况下，由从封闭 SELECT 引用的列组成的表达式在某些情境下不会正确渲染 SQL，即使表达式具有与使用`query_expression()`的属性匹配的标签名称，即使`query_expression()`没有默认表达式。目前，如果`query_expression()`确实有默认表达式，那个标签名称仍然用于该默认表达式，并且具有相同名称的额外标签将继续被忽略。总的来说，这种情况相当棘手，可能需要进一步调整。

    参考：[#8881](https://www.sqlalchemy.org/trac/ticket/8881)

### engine

+   **[engine] [错误]**

    修复了`Result.freeze()`方法无法用于使用`text()`或`Connection.exec_driver_sql()`的文本 SQL 的问题。

    参考：[#8963](https://www.sqlalchemy.org/trac/ticket/8963)

### sql

+   **[sql] [用例]**

    现在，在任何“文字绑定参数”渲染操作失败的情况下，会抛出一个信息性的重新引发，指示值本身和正在使用的数据类型，以帮助调试在语句中渲染文字参数时出现的��题。

    参考：[#8800](https://www.sqlalchemy.org/trac/ticket/8800)

+   **[sql] [错误]**

    修复了关于渲染绑定参数位置和有时身份的一系列问题，例如用于 SQLite、asyncpg、MySQL、Oracle 等的参数。一些编译形式无法正确维护参数的顺序，例如 PostgreSQL 的`regexp_replace()`函数，首次引入于[#4123](https://www.sqlalchemy.org/trac/ticket/4123)的`CTE`构造的“嵌套”功能，以及使用 Oracle 的`FunctionElement.column_valued()`方法形成的可选择表。

    参考：[#8827](https://www.sqlalchemy.org/trac/ticket/8827)

### asyncio

+   **[asyncio] [错误]**

    从 `AsyncResult` 中删除了无效的 `merge()` 方法。此方法从未起作用，并且错误地包含在 `AsyncResult` 中。

    参考：[#8952](https://www.sqlalchemy.org/trac/ticket/8952)

### postgresql

+   **[postgresql] [bug]**

    调整了 PostgreSQL 方言在从表中反射列时考虑列类型的方式，以适应可能从 PG `format_type()` 函数返回 NULL 的替代后端。

    参考：[#8748](https://www.sqlalchemy.org/trac/ticket/8748)

### sqlite

+   **[sqlite] [usecase]**

    添加了对 SQLite 后端反映“DEFERRABLE”和“INITIALLY”关键字的支持，这些关键字可能存在于外键构造中。感谢 Michael Gorven 的贡献。

    参考：[#8903](https://www.sqlalchemy.org/trac/ticket/8903)

+   **[sqlite] [usecase]**

    添加了对 SQLite 方言中包含在索引中的基于表达式的 WHERE 条件的反射支持，类似于 PostgreSQL 方言的方式。感谢 Tobias Pfeiffer 的贡献。

    参考：[#8804](https://www.sqlalchemy.org/trac/ticket/8804)

+   **[sqlite] [bug]**

    回溯了一个关于 SQLite 反射附加模式中唯一约束的修复，作为 [#4379](https://www.sqlalchemy.org/trac/ticket/4379) 的一小部分在 2.0 中发布。以前，附加模式中的唯一约束会被 SQLite 反射忽略。感谢 Michael Gorven 的贡献。

    参考：[#8866](https://www.sqlalchemy.org/trac/ticket/8866)

### oracle

+   **[oracle] [bug]**

    继续修复 Oracle 修复 [#8708](https://www.sqlalchemy.org/trac/ticket/8708)，在 1.4.43 中发布，其中以下划线开头的绑定参数名称（Oracle 不允许）仍未在所有情况下正确转义。

    参考：[#8708](https://www.sqlalchemy.org/trac/ticket/8708)

+   **[oracle] [bug]**

    修复了 Oracle 编译器中 `FunctionElement.column_valued()` 语法不正确的问题，未正确限定源表，导致名称 `COLUMN_VALUE` 不正确。

    参考：[#8945](https://www.sqlalchemy.org/trac/ticket/8945)

## 1.4.44

发布日期：2022 年 11 月 12 日

### sql

+   **[sql] [bug]**

    修复了缓存键生成中识别到的关键内存问题，对于使用大量带有子查询的 ORM 别名的非常大且复杂的 ORM 语句，缓存键生成可能会产生比语句本身大几个数量级的过大键。非常感谢 Rollo Konig Brock 在最终识别此问题方面的非常耐心和长期帮助。

    参考：[#8790](https://www.sqlalchemy.org/trac/ticket/8790)

### postgresql

+   **[postgresql] [bug] [mssql]**

    仅针对 PostgreSQL 和 SQL Server 方言，调整了编译器，以便在渲染 RETURNING 子句中的列表达式时，对于生成标签的 SQL 表达式元素建议使用“非匿名”标签，主要示例是 SQL 函数，可能作为列的类型的一部分发出，其中标签名称应默认与列的名称匹配。这恢复了在版本 1.4.21 中由于[#6718](https://www.sqlalchemy.org/trac/ticket/6718)，[#6710](https://www.sqlalchemy.org/trac/ticket/6710)引起的一个未明确定义的行为变更。Oracle 方言具有不同的 RETURNING 实现，并不受此问题影响。版本 2.0 对其他后端的广泛扩展的 RETURNING 支持进行了全面更改。

    参考：[#8770](https://www.sqlalchemy.org/trac/ticket/8770)

### oracle

+   **[oracle] [bug]**

    修复了 Oracle 方言中的问题，其中针对完整的`Table`对象一次性使用 `insert(some_table).values(...).returning(some_table)` 的 INSERT 语句会失败执行，引发异常。

### 测试

+   **[tests] [bug]**

    修复了测试套件中使用 `--disable-asyncio` 参数时的问题，该参数实际上未能运行 greenlet 测试，并且也未能阻止套件在整个运行过程中使用“包装” greenlet。设置此参数后，确保整个运行过程中不会发生任何 greenlet 或 asyncio 使用。

    参考：[#8793](https://www.sqlalchemy.org/trac/ticket/8793)

+   **[tests] [bug]**

    调整了测试套件，测试 Mypy 插件，以适应 Mypy 0.990 中如何处理消息输出的更改，这影响了确定是否应为特定文件打印注释和错误时 sys.path 的解释。该更改使测试套件中的文件在使用 mypy API 运行时不再产生消息，从而破坏了测试套件。

## 1.4.43

发布日期：2022 年 11 月 4 日

### orm

+   **[orm] [bug]**

    修复了连接的急切加载中的问题，在其中，当急切加载跨越三个映射器时，具有特定外部/内部连接的急切加载组合时，会发生断言失败。

    参考：[#8738](https://www.sqlalchemy.org/trac/ticket/8738)

+   **[orm] [bug]**

    修复了涉及 `Select` 构造的错误，其中 `Select.select_from()` 与 `Select.join()` 的组合，以及使用 `Select.join_from()` 时，如果查询的列子句未明确包括 JOIN 的左侧实体，则会导致 `with_loader_criteria()` 功能以及单表继承查询所需的 IN 条件不会呈现。现在正确的实体被传递给内部生成的 `Join` 对象，以便正确添加与左侧实体的条件。

    引用：[#8721](https://www.sqlalchemy.org/trac/ticket/8721)

+   **[orm] [bug]**

    当在特定“加载器路径”中添加到加载器选项时引发了一个信息性异常，例如在 `Load.options()` 中使用它时，现在会引发一个信息性异常。此使用不受支持，因为 `with_loader_criteria()` 仅用作顶级加载器选项。以前，会生成内部错误。

    引用：[#8711](https://www.sqlalchemy.org/trac/ticket/8711)

+   **[orm] [bug]**

    为 `Session.get()` 改进了“字典模式”，以便于在命名字典中指示指向主键属性名称的同义词名。

    引用：[#8753](https://www.sqlalchemy.org/trac/ticket/8753)

+   **[orm] [bug]**

    修复了继承映射器中“selectin_polymorphic”加载的问题，如果 `Mapper.polymorphic_on` 参数引用的 SQL 表达式不直接映射到类上，则此问题将无法正确工作。

    引用：[#8704](https://www.sqlalchemy.org/trac/ticket/8704)

+   **[orm] [bug]**

    修复了在使用`Query`对象作为迭代器时，如果在迭代过程中引发了用户定义的异常情况，则底层的 DBAPI 游标不会被关闭的问题，从而导致迭代器被 Python 解释器关闭。当使用`Query.yield_per()`创建服务器端游标时，这将导致通常与服务器端游标不同步相关的 MySQL 问题，并且由于无法直接访问`Result`对象，最终用户代码无法访问游标以关闭它。

    为了解决这个问题，在迭代器方法中应用了对`GeneratorExit`的捕获，当迭代器被中断时，将关闭结果对象，并且根据定义将被 Python 解释器关闭。

    作为 1.4 系列实施的这一变更的一部分，确保了所有`Result`实现都提供了`.close()`方法，包括`ScalarResult`、`MappingResult`。2.0 版本的这一变更还包括了用于与`Result`类一起使用的新上下文管理器模式。

    参考：[#8710](https://www.sqlalchemy.org/trac/ticket/8710)

### 引擎

+   **[engine] [bug] [regression]**

    修复了当`Connection`关闭并正在将其 DBAPI 连接返回到连接池时，在某些情况下不会调用`PoolEvents.reset()`事件钩子的问题。

    情况是当`Connection`在将连接返回到池的过程中已经发出了`.rollback()`时，然后会指示连接池放弃执行自己的“重置”以节省额外的方法调用。然而，这阻止了在此钩子中使用自定义池重置方案，因为这样的钩子定义上做的不仅仅是调用`.rollback()`，并且需要在所有情况下被调用。这是在 1.4 版本中出现的一个退化。

    对于版本 1.4，`PoolEvents.checkin()`仍然可以作为用于自定义“重置”实现的替代事件钩子。版本 2.0 将提供一个改进版本的`PoolEvents.reset()`，它被用于其他情况，例如终止 asyncio 连接，并且还传递了有关重置的上下文信息，以允许“自定义连接重置”方案以不同的方式响应不同的重置情况。

    参考：[#8717](https://www.sqlalchemy.org/trac/ticket/8717)

+   **[engine] [bug]**

    确保所有`Result`对象都包括`Result.close()`方法以及`Result.closed`属性，包括`ScalarResult`和`MappingResult`。

    参考：[#8710](https://www.sqlalchemy.org/trac/ticket/8710)

### sql

+   **[sql] [bug]**

    修复了一个问题，该问题阻止了`literal_column()`构造在`Select`构造上正常工作，以及在其他潜在的可能生成“匿名标签”的地方，如果字面表达式包含可能干扰格式字符串的字符，例如括号，由于“匿名标签”结构的实现细节。

    参考：[#8724](https://www.sqlalchemy.org/trac/ticket/8724)

### mssql

+   **[mssql] [bug]**

    修复了使用`Inspector.has_table()`针对带有 SQL Server 方言的临时表时会在某些 Azure 变体上失败的问题，因为不支持在这些服务器版本上的不必要的信息模式查询。来自 Mike Barry 的拉取请求。

    参考：[#8714](https://www.sqlalchemy.org/trac/ticket/8714)

+   **[mssql] [bug] [reflection]**

    修复了针对带有 SQL Server 方言的视图使用时的`Inspector.has_table()`的问题，由于 1.4 系列中去除了对 SQL Server 的此支持而导致错误返回`False`。此问题在使用不同反射架构的 2.0 系列中不存在。添加了测试支持以确保`has_table()`保持符合视图的规范。

    引用：[#8700](https://www.sqlalchemy.org/trac/ticket/8700)

### oracle

+   **[oracle] [bug]**

    修复了一个问题，即包含通常需要在 Oracle 中用引号引用的字符的绑定参数名称，包括从同名数据库列自动派生的参数名称，在使用 Oracle 方言的“expanding parameters”时不会被转义，从而导致执行错误。 Oracle 方言使用的绑定参数的通常“引用”不适用于“expanding parameters”架构，因此现在使用一系列特定于 Oracle 的字符/转义来转义。 

    引用：[#8708](https://www.sqlalchemy.org/trac/ticket/8708)

+   **[oracle] [bug]**

    修复了一个问题，即在第一次连接时查询`nls_session_parameters`视图以获取默认的小数点字符可能不可用，具体取决于 Oracle 连接模式，因此可能会引发错误。检测小数字符的方法已经简化为直接测试一个小数值，而不是读取系统视图，这在任何后端/驱动程序上都有效。

    引用：[#8744](https://www.sqlalchemy.org/trac/ticket/8744)

## 1.4.42

发布日期：2022 年 10 月 16 日

### orm

+   **[orm] [bug]**

    当传递给`Session.execute()`等方法时，`Session.execute.bind_arguments` 字典不再发生变化；相反，它会被复制到内部字典以进行状态更改。在其他方面，这修复了一个问题，即传递给`Session.get_bind()`方法的“clause”将错误地引用了用于“fetch”同步策略的`Select`构造，而实际发出的查询是`Delete`或`Update`。这会干扰“路由会话”的处理方法。

    引用：[#8614](https://www.sqlalchemy.org/trac/ticket/8614)

+   **[orm] [bug]**

    在 ORM 配置中当将映射类的本地列应用于任何不包含相同表列的引用类时，会发出警告，这时应用了显式的`remote()` 注解。理想情况下，这在某个时候会引发错误，因为从映射的角度来看这是不正确的。

    参考文献：[#7094](https://www.sqlalchemy.org/trac/ticket/7094)

+   **[orm] [bug]**

    当尝试配置一个在继承层次结构中的映射类时，其中映射器没有给定任何多态标识时会发出警告，但是却分配了一个多态鉴别器列。这样的类如果从不打算直接加载，则应该是抽象的。

    参考文献：[#7545](https://www.sqlalchemy.org/trac/ticket/7545)

+   **[orm] [bug] [regression]**

    在`contains_eager()`中修复了 1.4 版本的回归问题，在其中 `joinedload()` 的“包装成子查询”逻辑会意外地触发使用 `contains_eager()` 函数的类似语句（例如使用 `distinct()`、`limit()` 或 `offset()` 的语句），然后会导致使用一些 SQL 标签名称和别名的查询的次要问题。这种“包装”对于 `contains_eager()` 并不合适，因为它一直遵循的合同是用户定义的 SQL 语句未经修改，只是添加了适当的列以被提取。

    参考文献：[#8569](https://www.sqlalchemy.org/trac/ticket/8569)

+   **[orm] [bug] [regression]**

    修复了使用同步会话=’fetch’的 ORM update() 会因为在刷新对象时使用评估器来确定 SET 子句中的表达式的 Python 值而失败的回归问题；如果评估器针对非数值值（例如 PostgreSQL JSONB）使用数学运算符，那么评估不可用的条件将无法正确检测。评估器现在只对数值类型限制了数学突变运算符的使用，异常是“+”继续对字符串起作用。SQLAlchemy 2.0 可能会通过完全获取 SET 值而不是使用评估来进一步更改此设置。

    参考文献：[#8507](https://www.sqlalchemy.org/trac/ticket/8507)

### engine

+   **[engine] [bug]**

    修复了在 `select()` 构造的列子句中混合使用“*”和额外显式命名列表达式会导致结果列定位有时将标签名称或其他非重复名称视为模糊目标的问题。

    参考文献：[#8536](https://www.sqlalchemy.org/trac/ticket/8536)

### asyncio

+   **[asyncio] [bug]**

    改进了 `asyncio.shield()` 在上下文管理器中的实现，如在 [#8145](https://www.sqlalchemy.org/trac/ticket/8145) 中所添加的，使得“关闭”操作被封装在一个 `asyncio.Task` 中，然后在操作进行时被强引用。这符合 Python 文档中指出的任务否则不会被强引用的情况。

    参考文献：[#8516](https://www.sqlalchemy.org/trac/ticket/8516)

### postgresql

+   **[postgresql] [usecase]**

    `aggregate_order_by` 现在支持缓存生成。

    参考文献：[#8574](https://www.sqlalchemy.org/trac/ticket/8574)

### mysql

+   **[mysql] [bug]**

    调整了在测试视图时用于匹配“CREATE VIEW”的正则表达式，以更灵活地工作，不再需要中间的特殊关键字“ALGORITHM”，这原本是可选的，但却不能正确工作。此更改允许视图反射在诸如 StarRocks 这样的兼容 MySQL 的变体上更完整地工作。拉取请求由 John Bodley 提供。

    参考文献：[#8588](https://www.sqlalchemy.org/trac/ticket/8588)

### mssql

+   **[mssql] [bug] [regression]**

    修复了 SQL Server 隔离级别获取中的另一个回归问题（见 [#8231](https://www.sqlalchemy.org/trac/ticket/8231), [#8475](https://www.sqlalchemy.org/trac/ticket/8475)），这次是关于“Microsoft Dynamics CRM 数据库通过 Azure Active Directory”，显然完全缺少 `system_views` 视图。错误捕获已扩展，保证这种方法绝对不会失败，只要有数据库连接存在。

    参考文献：[#8525](https://www.sqlalchemy.org/trac/ticket/8525)

## 1.4.41

发布日期：2022 年 9 月 6 日

### orm

+   **[orm] [bug] [events]**

    修复了一个事件监听问题，即当事件监听器添加到一个超类时，如果创建了一个子类，那么子类自己的监听器就会丢失。一个实际的例子是在为 `Session` 类关联事件之后创建的 `sessionmaker` 类。

    参考文献：[#8467](https://www.sqlalchemy.org/trac/ticket/8467)

+   **[orm] [bug]**

    加强了`aliased()`和`with_polymorphic()`构造的缓存键策略。虽然很难（如果有的话）演示涉及实际语句被缓存的问题，但是这两个构造在其缓存键中未包含足够使它们在缓存时独特的内容，以使仅对别名构造进行缓存是准确的。

    参考：[#8401](https://www.sqlalchemy.org/trac/ticket/8401)

+   **[orm] [bug] [regression]**

    修复了在 1.4 系列中出现的回归问题，其中作为子查询放置的继承查询在该实体的封闭查询中会失败以正确渲染 JOIN。这个问题在 1.4.18 版本之前和之后的两种不同情况下表现出来（相关问题 [#6595](https://www.sqlalchemy.org/trac/ticket/6595)），在一种情况下会渲染两次 JOIN，而在另一种情况下会完全丢失 JOIN。为了解决这个问题，已经缩减了应用“多态加载”的条件，不再为简单的继承查询调用它。

    参考：[#8456](https://www.sqlalchemy.org/trac/ticket/8456)

+   **[orm] [bug]**

    修复了`sqlalchemy.ext.mutable`扩展中的问题，如果对象在通过 `Session.merge()` 合并时同时传递 `Session.merge.load` 为 False，则会丢失到父对象的集合链接。

    参考：[#8446](https://www.sqlalchemy.org/trac/ticket/8446)

+   **[orm] [bug]**

    修复了涉及`with_loader_criteria()`的问题，其中在 lambda 内使用的闭包变量作为绑定参数值在语句被缓存后不会正确传递到额外的关系加载器（如`selectinload()`和`lazyload()`），而是使用过时的原始缓存值。

    参考：[#8399](https://www.sqlalchemy.org/trac/ticket/8399)

### sql

+   **[sql] [bug]**

    修复了使用 `table()` 构造时，将字符串传递给 `table.schema` 参数时，未考虑“schema”字符串在生成缓存键时，导致如果使用了具有不同模式的多个同名 `table()` 构造，可能会导致缓存冲突的问题。

    参考：[#8441](https://www.sqlalchemy.org/trac/ticket/8441)

### asyncio

+   **[asyncio] [bug]**

    集成了对 asyncpg 的 `terminate()` 方法调用的支持，用于在连接池回收可能超时的连接、在垃圾回收未正常关闭连接时以及连接已失效时。这允许 asyncpg 放弃连接而无需等待可能导致长时间超时的响应。

    参考：[#8419](https://www.sqlalchemy.org/trac/ticket/8419)

### mssql

+   **[mssql] [bug] [regression]**

    修复了在 1.4.40 中发布的 [#8231](https://www.sqlalchemy.org/trac/ticket/8231) 修复���起的回归，当用户没有权限查询 `dm_exec_sessions` 或 `dm_pdw_nodes_exec_sessions` 系统视图以确定当前事务隔离级别时，连接会失败。

    参考：[#8475](https://www.sqlalchemy.org/trac/ticket/8475)

## 1.4.40

发布日期：2022 年 8 月 8 日

### orm

+   **[orm] [bug]**

    修复了在多次引用 CTE 与多态 SELECT 结合时可能导致构建多个相同 CTE 的“克隆体”，然后触发这两个 CTE 作为重复项的问题。为了解决这个问题，在发生这种情况时，这两个 CTE 会进行深度比较以确保它们是等价的，然后被视为等价。

    参考：[#8357](https://www.sqlalchemy.org/trac/ticket/8357)

+   **[orm] [bug]**

    通过传递唯一的 `SELECT *` 参数（通过字符串、`text()` 或 `literal_column()`）给 `select()` 构造，将被解释为核心级别的 SQL 语句，而不是 ORM 级别的语句。这样，当 `*` 扩展以匹配任意数量的列时，将返回结果中的所有列。ORM 级别的 `select()` 的解释需要提前知道所有 ORM 列的名称和类型，而当使用 `'*'` 时无法实现。

    如果在 ORM 语句中同时使用`'*`和其他表达式，会引发错误，因为 ORM 无法正确解释这种情况。

    参考：[#8235](https://www.sqlalchemy.org/trac/ticket/8235)

### orm 声明式

+   **[orm] [declarative] [bug]**

    修复了一个问题，即将一系列设置为抽象或混合声明类的类层次结构上的独立列声明在超类上，然后正确复制到`declared_attr`可调用的类上，该类希望在后代类上使用它们。

    参考：[#8190](https://www.sqlalchemy.org/trac/ticket/8190)

### 引擎

+   **[engine] [usecase]**

    在核心中为`Connection`实现了新的`Connection.execution_options.yield_per`执行选项，以模仿 ORM 中相同的 yield_per 选项。该选项同时设置`Connection.execution_options.stream_results`选项，并调用`Result.yield_per()`，以提供最常见的流式结果配置，也与 ORM 使用情况中的使用模式相一致。

    另请参阅

    使用服务器端游标（即流式结果） - 修订文档

+   **[engine] [bug]**

    修复了`Result`中的错误，当使用缓冲结果策略时，如果使用的方言不支持显式的“服务器端游标”设置，则不会使用`Connection.execution_options.stream_results`。这是一个错误，因为像 SQLite 和 Oracle 这样的 DBAPI 已经使用了非缓冲结果获取方案，仍然受益于部分结果获取的使用。在设置`Connection.execution_options.stream_results`的所有情况下现在都使用“缓冲”策略。

+   **[engine] [bug]**

    添加了 `FilterResult.yield_per()`，以便结果实现（如 `MappingResult`、`ScalarResult` 和 `AsyncResult`）可以访问此方法。

    参考：[#8199](https://www.sqlalchemy.org/trac/ticket/8199)

### sql

+   **[sql] [bug]**

    调整了字符串包含函数 `.contains()`、`.startswith()`、`.endswith()` 的 SQL 编译，强制使用字符串连接运算符，而不是依赖于加法运算符的重载，以便非标准的用法，例如字节字符串仍然生成字符串连接运算符。

    参考：[#8253](https://www.sqlalchemy.org/trac/ticket/8253)

### mypy

+   **[mypy] [bug]**

    修复了使用 lambda 作为 Column 默认值时 mypy 插件崩溃的问题。感谢 tchapi 提交的拉取请求。

    参考：[#8196](https://www.sqlalchemy.org/trac/ticket/8196)

### asyncio

+   **[asyncio] [bug]**

    在使用 `AsyncConnection` 或 `AsyncSession` 作为上下文管理器释放对象时，特别是在 `__aexit__()` 上下文管理器退出时，将 `asyncio.shield()` 添加到连接和会话释放过程中。 当上下文管理器完成时，这似乎有助于使用其他并发库（如 `anyio`、`uvloop`）时取消任务时正确释放连接池中的连接。

    参考：[#8145](https://www.sqlalchemy.org/trac/ticket/8145)

### postgresql

+   **[postgresql] [bug]**

    修复了 psycopg2 方言中的问题，在实现了 [#4392](https://www.sqlalchemy.org/trac/ticket/4392) 的“多主机”功能时，可以将多个 `host:port` 对作为查询字符串传递，例如 `?host=host1:port1&host=host2:port2&host=host3:port3`，但没有正确实现，因为它没有适当地传播“port”参数。 没有使用不同“port”的连接可能正常工作，而具有某些条目的“port”可能会不正确地传递该主机名。 现在已更正格式以正确传递主机/端口。

    作为这一变化的一部分，保持了另一种意外有效的多主机样式的支持，即以逗号分隔的`?host=h1,h2,h3&port=p1,p2,p3`。这种格式更符合 libpq 的查询字符串格式，而以前的格式受到 libpq URI 格式的不同方面的启发，但并不完全相同。

    如果两种样式混合在一起，会引发错误，因为这是模棱两可的。

    参考：[#4392](https://www.sqlalchemy.org/trac/ticket/4392)

### mssql

+   **[mssql] [bug]**

    修复了与 SQL Server pyodbc 方言一起使用 ORM 对象的新使用模式在 使用 INSERT、UPDATE 和 ON CONFLICT（即 upsert）返回 ORM 对象时无法正确工作的问题。

    参考：[#8210](https://www.sqlalchemy.org/trac/ticket/8210)

+   **[mssql] [bug]**

    修复了在 Azure Synapse Analytics 上 SQL Server 方言的当前隔离级别查询失败的问题，这是由于这个数据库在错误发生后处理事务回滚的方式。已修改初始查询，不再依赖于在检测适当的系统视图时捕获错误。此外，为了更好地支持这个数据库的非常特定的“回滚”行为，实现了新参数`ignore_no_transaction_on_rollback`，指示回滚应忽略 Azure Synapse 错误‘No corresponding transaction found. (111214)’，如果没有与 Python DBAPI 冲突的事务存在。

    初版补丁和宝贵的调试协助由@ww2406 提供。

    另见

    避免 Azure Synapse Analytics 上的事务相关异常

    参考：[#8231](https://www.sqlalchemy.org/trac/ticket/8231)

### 杂项

+   **[bug] [types]**

    修复了`TypeDecorator`在装饰`ARRAY`数据类型时，不会正确代理`__getitem__()`运算符的问题，没有明确的解决方法。

    参考：[#7249](https://www.sqlalchemy.org/trac/ticket/7249)

## 1.4.39

发布日期：2022 年 6 月 24 日

### orm

+   **[orm] [bug] [regression]**

    修复了由[#8133](https://www.sqlalchemy.org/trac/ticket/8133)引起的回归，其中可变属性的 pickle 格式被更改，而没有回退到识别旧格式，导致 SQLAlchemy 的就地升级无法再读取来自旧版本的 pickled 数据。现在增加了检查以及对旧格式的回退。

    参考：[#8133](https://www.sqlalchemy.org/trac/ticket/8133)

## 1.4.38

发布日期：2022 年 6 月 23 日

### orm

+   **[orm] [bug] [regression]**

    修复了由[#8064](https://www.sqlalchemy.org/trac/ticket/8064)引起的回归，其中对列对应性的特定检查过于宽松，导致一些 ORM 子查询的渲染不正确，例如那些使用`PropComparator.has()`或`PropComparator.any()`与使用传统别名功能的联接继承查询。

    参考：[#8162](https://www.sqlalchemy.org/trac/ticket/8162)

+   **[orm] [bug] [sql]**

    修复了在使用 ORM 执行语句时，`GenerativeSelect.fetch()`未被应用的问题。

    参考：[#8091](https://www.sqlalchemy.org/trac/ticket/8091)

+   **[orm] [bug]**

    修复了一个问题，即`with_loader_criteria()`选项无法被 pickle，这在与缓存方案一起传播到懒加载器时是必要的。目前，唯一支持的可 pickle 形式是将“where criteria”作为一个固定的模块级可调用函数传递，该函数生成一个 SQL 表达式。一个临时的“lambda”无法被 pickle，而一个 SQL 表达式对象通常不能直接完全 pickle。

    参考：[#8109](https://www.sqlalchemy.org/trac/ticket/8109)

### engine

+   **[engine] [bug]**

    修复了一个阻止诸如`Connection`等关键对象具有正确`__weakref__`属性的弃用警告类装饰器，导致像 Python 标准库`inspect.getmembers()`之类的操作失败。

    参考：[#8115](https://www.sqlalchemy.org/trac/ticket/8115)

### sql

+   **[sql] [bug]**

    修复了与`lambda_stmt()`相关的多个观察到的竞争条件，包括在多个同时线程中首次分析新的 Python 代码对象时出现的初始“dogpile”问题，这导致了性能问题以及一些内部状态的损坏。此外，修复了观察到的竞争条件，当在不同线程中编译或访问正在被克隆的表达式构造时可能发生，因为 Python 版本在 3.10 之前的版本中，由于记忆化属性在迭代时改变`__dict__`，特别是 lambda SQL 构造对此很敏感，因为它持久地保留一个单一的语句对象。迭代已经被改进为使用`dict.copy()`，无论是否有额外的迭代。

    参考：[#8098](https://www.sqlalchemy.org/trac/ticket/8098)

+   **[sql] [bug]**

    加强了`Cast`和其他“包装”列构造的机制，以更完全地保留被包装的`Label`构造，包括标签名称将在`Subquery`的`.c`集合中被保留。该标签已经能够在它被包装在内部的构造外正确地呈现 SQL。

    参考：[#8084](https://www.sqlalchemy.org/trac/ticket/8084)

+   **[sql] [错误]**

    调整了对[#8056](https://www.sqlalchemy.org/trac/ticket/8056)所做的修复，该修复调整了具有特殊字符的绑定参数名称的转义方式，以便在 SQL 编译步骤之后翻译转义名称，这破坏了 FAQ 中一个已发布的示例，该示例说明了如何将参数名称合并到编译后的 SQL 字符串的输出中。该更改恢复了来自`compiled.params`的转义名称，并向`SQLCompiler.construct_params()`添加了一个条件参数，命名为`escape_names`，默认为`True`，以默认情况下恢复旧行为。

    参考资料：[#8113](https://www.sqlalchemy.org/trac/ticket/8113)

### 模式

+   **[模式] [错误]**

    修复了涉及`Table.include_columns`和`Table.resolve_fks`参数的错误；这些很少使用的参数显然无法为引用外键约束的列工作。

    在第一种情况下，引用外键的未包含列仍然会尝试创建一个`ForeignKey`对象，在尝试解析外键约束的列时会产生错误；引用被跳过的列的外键约束现在与具有相同条件的`Index`和`UniqueConstraint`对象一样在表反射过程中被省略了。但是不会产生警告，因为我们可能希望在 2.0 中删除所有约束的 include_columns 警告。

    在后一种情况下，如果找不到与 FK 相关的表，则会在`resolve_fks=False`的情况下无法生成表别名或子查询；逻辑已被修复，因此如果未找到相关表，则`ForeignKey`对象仍会被代理到别名表或子查询（这些`ForeignKey`对象通常用于生成连接条件），但会发送一个标志，表明它是不可解析的。然后，别名表/子查询将正常工作，唯一的例外是它无法自动生成连接条件，因为缺少外键信息。这已经是这种外键约束的行为，该约束是使用非反射方法产生的，例如从不同的`MetaData`集合中连接`Table`对象。

    参考：[#8100](https://www.sqlalchemy.org/trac/ticket/8100), [#8101](https://www.sqlalchemy.org/trac/ticket/8101)

+   **[模式] [错误] [mssql]**

    修复了一个问题，当`Table`对象使用带有`Numeric`数据类型的 IDENTITY 列时，尝试调解“autoincrement”列时会产生错误，阻止了使用`Column.autoincrement`参数来构造`Column`的构造，并在尝试调用`Insert`构造时发出错误。

    参考：[#8111](https://www.sqlalchemy.org/trac/ticket/8111)

### 扩展

+   **[扩展] [错误]**

    修复了在`Mutable`中的错误，在其中对包含多个`Mutable`启用属性的映射实例进行 pickling 和 unpickling 时，将不会正确恢复状态。

    参考：[#8133](https://www.sqlalchemy.org/trac/ticket/8133)

## 1.4.37

发布日期：2022 年 5 月 31 日

### orm

+   **[orm] [错误]**

    修复了使用`column_property()`构造包含子查询的问题，该子查询针对已映射的列属性将不正确应用 ORM 编译行为，包括针对单表继承表达式添加的“IN”表达式将无法包含的问题。

    参考：[#8064](https://www.sqlalchemy.org/trac/ticket/8064)

+   **[orm] [bug]**

    修复了一个问题，在 ORM 结果中，在选择的列集更改时，例如使用 `Select.with_only_columns()` 时，会向返回的 `Row` 对象应用不正确的键名。

    参考：[#8001](https://www.sqlalchemy.org/trac/ticket/8001)

+   **[orm] [bug] [oracle] [postgresql]**

    修复了一个 bug，很可能是从 1.3 版本开始的回归，当使用需要绑定参数转义的列名时（更具体地说，当在使用 Oracle 时列名需要引用时，例如以下划线开头的列名，或在某些情况下使用某些 PostgreSQL 驱动程序时，当使用包含百分号的列名时），如果版本控制列本身具有此类名称，则 ORM 版本控制功能将无法正常工作，因为 ORM 假定存在某些绑定参数命名约定，这些约定通过引号受到干扰。此问题与 [#8053](https://www.sqlalchemy.org/trac/ticket/8053) 相关，并且基本上修改了修复此问题的方法，修改了最初为广义绑定参数名称引用创建初始实现的原始问题 [#5653](https://www.sqlalchemy.org/trac/ticket/5653)。

    参考：[#8056](https://www.sqlalchemy.org/trac/ticket/8056)

### 引擎

+   **[engine] [bug] [tests]**

    修复了在支持记录“stacklevel”日志时实施的问题，在 [#7612](https://www.sqlalchemy.org/trac/ticket/7612) 中，需要调整以使其与最近发布的 Python 3.11.0b1 版本一起使用，还修复了测试此功能的单元测试。

    参考：[#8019](https://www.sqlalchemy.org/trac/ticket/8019)

### SQL

+   **[sql] [bug] [postgresql] [sqlite]**

    修复了一个 bug，在 PostgreSQL 的 `Insert.on_conflict_do_update()` 方法和 SQLite 的 `Insert.on_conflict_do_update()` 方法中，当使用字典传递给 `Insert.on_conflict_do_update.set_` 时，如果通过其键名指定列时失败，以及如果直接使用 `Insert.excluded` 集合作为字典时也会失败，尤其是当列有单独的“.key”时。

    参考：[#8014](https://www.sqlalchemy.org/trac/ticket/8014)

+   **[sql] [bug]**

    当`Insert.from_select()`传递一个“复合选择”对象，如 UNION，但 INSERT 语句需要附加额外的列以支持来自表元数据的 Python 端或显式 SQL 默认值时，将引发一个信息性错误。在这种情况下，应传递复合对象的子查询。

    引用：[#8073](https://www.sqlalchemy.org/trac/ticket/8073)

+   **[sql] [bug]**

    修复了使用`bindparam()`没有明确给出数据或类型时，可能会在表达式中被强制转换为不正确类型的问题，例如当使用`Comparator.any()`和`Comparator.all()`时。

    引用：[#7979](https://www.sqlalchemy.org/trac/ticket/7979)

+   **[sql] [bug]**

    如果两个单独的`BindParameter`对象共享相同的名称，但一个在“扩展”上下文中使用（通常是 IN 表达式），另一个则不是，则会引发一个信息性错误；在这两种不同风格的用法中混合使用相同的名称不受支持，通常应在要在 IN 表达式之外接收列表值的参数上设置`expanding=True`参数（默认情况下设置`expanding`）。

    引用：[#8018](https://www.sqlalchemy.org/trac/ticket/8018)

### mysql

+   **[mysql] [bug]**

    进一步调整 MySQL PyODBC 方言，以允许完全的连接性，尽管在[#7871](https://www.sqlalchemy.org/trac/ticket/7871)中已经修复，但之前仍然无法正常工作。

    引用：[#7966](https://www.sqlalchemy.org/trac/ticket/7966)

+   **[mysql] [bug]**

    为 MySQL 错误 4031 添加了断开连接代码，该错误在 MySQL >= 8.0.24 中引入，表示连接空闲超时已超过。特别是，这修复了一个问题，即预先 ping 不能在超时连接上重新连接的问题。拉取请求由 valievkarim 提供。

    引用：[#8036](https://www.sqlalchemy.org/trac/ticket/8036)

### mssql

+   **[mssql] [bug]**

    修复了以“{”开头的密码导致登录失败的问题。

    引用：[#8062](https://www.sqlalchemy.org/trac/ticket/8062)

+   **[mssql] [bug] [reflection]**

    在使用 MSSQL 反射表列时明确指定排序规则，以防止“排序规则冲突”错误。

    引用：[#8035](https://www.sqlalchemy.org/trac/ticket/8035)

### oracle

+   **[oracle] [usecase]**

    为 Oracle 断开处理添加了两个新的错误代码，以支持 Oracle 发布的新“python-oracledb”驱动程序的早期测试。

    引用：[#8066](https://www.sqlalchemy.org/trac/ticket/8066)

+   **[oracle] [bug]**

    修复了 SQL 编译器问题，即如果绑定参数的名称被“转义”，则绑定参数的“绑定处理”函数不会正确应用于绑定值。具体来说，这适用于 Oracle 等情况，当`Column`的名称本身需要引号引用时，因此在 DML 语句中生成的绑定参数使用需要绑定处理的数据类型时，引号引用的名称将用于绑定参数。 

    参考：[#8053](https://www.sqlalchemy.org/trac/ticket/8053)

## 1.4.36

发布日期：2022 年 4 月 26 日

### orm

+   **[orm] [bug] [regression]**

    修复了回归问题，即在版本 1.4.33 中发布的为[#7861](https://www.sqlalchemy.org/trac/ticket/7861)所做的更改，使`Insert`构造部分被识别为 ORM 启用的语句，未正确传递正确的映射器/映射表状态给`Session`，导致绑定到使用`Session.binds`参数绑定到引擎和/或连接的`Session`的`Session.get_bind()`方法失败。

    参考：[#7936](https://www.sqlalchemy.org/trac/ticket/7936)

### orm 声明式

+   **[orm] [declarative] [bug]**

    修改了`DeclarativeMeta`元类，将`cls.__dict__`传递到声明扫描过程中以查找属性，而不是传递给类型的`__init__()`方法的单独字典。这允许用户定义的基类在`__init_subclass__()`中添加属性时按预期工作，因为`__init_subclass__()`只能影响`cls.__dict__`本身，而不能影响其他字典。从技术上讲，这是从 1.3 版本中使用`__dict__`的退化。

    参考：[#7900](https://www.sqlalchemy.org/trac/ticket/7900)

### 引擎

+   **[engine] [bug]**

    修复了 C 扩展中的内存泄漏问题，当在 Python 3 中调用`Row`的命名成员时，如果成员不存在，可能会发生内存泄漏；特别是在 NumPy 转换期间尝试调用`.__array__`等成员时，但问题围绕着`Row`对象抛出的任何`AttributeError`。这个问题不适用于已经过渡到 Cython 的 2.0 版本。非常感谢 Sebastian Berg 发现了这个问题。

    参考：[#7875](https://www.sqlalchemy.org/trac/ticket/7875)

+   **[引擎] [错误]**

    添加了一个关于`Result.columns()`方法中存在的 bug 的警告，当在与将返回单个 ORM 实体的`Result`一起传递 0 时，这表明`Result.columns()`方法的当前行为在这种情况下是错误的，因为`Result`对象将产生标量值而不是`Row`对象。这个问题将在 2.0 中修复，这将是一个向后不兼容的更改，对于依赖于当前错误行为的代码来说，这是一个向后不兼容的更改。想要接收标量值集合的代码应该使用`Result.scalars()`方法，它将返回一个新的`ScalarResult`对象，该对象产生非行标量对象。

    参考：[#7953](https://www.sqlalchemy.org/trac/ticket/7953)

### 模式

+   **[模式] [错误]**

    修复了一个 bug，当使用`referred_column_0`命名约定键设置外键约束时，如果外键约束是作为一个`ForeignKey`对象而不是一个明确的`ForeignKeyConstraint`对象来设置时，命名约定将无效。由于此更改使用了一些从版本 2.0 中回退的修复的特性，还修复了一个很可能已经存在多年的、不为人所知的特性，即一个`ForeignKey`对象可以仅通过表的名称而不使用列名来引用被引用的表，如果被引用列的名称与被引用列的名称相同的话。

    先前未对 `ForeignKey` 对象进行测试的 `referred_column_0` 命名约定键，仅对 `ForeignKeyConstraint` 进行了测试，此 bug 揭示了除非对所有 FK 约束使用 `ForeignKeyConstraint`，否则该功能从未正确工作。此 bug 可追溯到为 [#3989](https://www.sqlalchemy.org/trac/ticket/3989) 引入该功能。

    参考：[#7958](https://www.sqlalchemy.org/trac/ticket/7958)

### asyncio

+   **[asyncio] [错误修复]**

    修复了异步适配事件处理程序中 `contextvar.ContextVar` 对象的处理。先前，在非可等待代码中调用可等待项时，不会传播应用于 `ContextVar` 的值。

    参考：[#7937](https://www.sqlalchemy.org/trac/ticket/7937)

### postgresql

+   **[postgresql] [错误修复]**

    修复了在 PostgreSQL 上使用 `ARRAY` 数据类型与 `Enum` 结合时的 bug，其中使用 `.any()` 或 `.all()` 方法以 Python 枚举成员作为参数来渲染 SQL 的 ANY() 或 ALL()，会导致所有驱动程序的类型适配失败。

    参考：[#6515](https://www.sqlalchemy.org/trac/ticket/6515)

+   **[postgresql] [错误修复]**

    为 PostgreSQL `UUID` 类型对象实现了 `UUID.python_type` 属性。该属性将根据 `UUID.as_uuid` 参数设置返回 `str` 或 `uuid.UUID`。此前，该属性未实现。感谢 Alex Grönholm 的拉取请求。

    参考：[#7943](https://www.sqlalchemy.org/trac/ticket/7943)

+   **[postgresql] [错误修复]**

    修复了在使用 `create_engine.pool_pre_ping` 参数时，psycopg2 方言中的问题，该问题会导致用户配置的 `AUTOCOMMIT` 隔离级别被“ping”处理程序意外重置。

    参考：[#7930](https://www.sqlalchemy.org/trac/ticket/7930)

### mysql

+   **[mysql] [错误修复] [回归]**

    修复了未经测试的 MySQL PyODBC 方言中的一个回归，该回归是由于版本 1.4.32 中对 [#7518](https://www.sqlalchemy.org/trac/ticket/7518) 的修复引起的，导致在首次连接时错误地传播了一个参数，从而导致 `TypeError`。

    参考：[#7871](https://www.sqlalchemy.org/trac/ticket/7871)

### 测试

+   **[测试] [错误修复]**

    对于第三方方言，修复了对 `SimpleUpdateDeleteTest` 套件测试的一个缺失要求，该套件测试未检查目标方言上的“rowcount”函数是否有效。

    参考：[#7919](https://www.sqlalchemy.org/trac/ticket/7919)

## 1.4.35

发布日期：2022 年 4 月 6 日

### sql

+   **[sql] [bug]**

    修复了新实现的`FunctionElement.table_valued.joins_implicitly`功能中的错误，即参数不会自动从原始`TableValuedAlias`对象传播到调用`TableValuedAlias.render_derived()`或`TableValuedAlias.alias()`时生成的次要对象。

    另外还修复了`TableValuedAlias`中的这些问题：

    +   修复了可能发生的内存问题，即在连续对同一对象的副本调用`TableValuedAlias.render_derived()`时可能发生的问题（对于.alias()，我们目前仍然必须继续从前一个元素链接。不确定是否可以改进，但这是.alias()在其他地方的标准行为）

    +   修复了调用`TableValuedAlias.render_derived()`或`TableValuedAlias.alias()`时个别元素类型丢失的问题。

    参考：[#7890](https://www.sqlalchemy.org/trac/ticket/7890)

+   **[sql] [bug] [regression]**

    修复了由[#7823](https://www.sqlalchemy.org/trac/ticket/7823)引起的回归问题，影响了缓存系统，使得在 ORM 操作中“克隆”绑定参数的情况下，例如多态加载，在某些情况下可能不会获得正确的执行时值，导致渲染不正确的绑定值。

    参考：[#7903](https://www.sqlalchemy.org/trac/ticket/7903)

## 1.4.34

发布日期：2022 年 3 月 31 日

### orm

+   **[orm] [bug] [regression]**

    修复了由[#7861](https://www.sqlalchemy.org/trac/ticket/7861)引起的回归问题，即通过`Session.execute()`直接调用包含 ORM 实体的`Insert`构造会失败的问题。

    参考：[#7878](https://www.sqlalchemy.org/trac/ticket/7878)

### postgresql

+   **[postgresql] [bug]**

    为了修复[#6581](https://www.sqlalchemy.org/trac/ticket/6581)中的问题，缩减了对 psycopg2 的“executemany values”模式的禁用，不再适用于所有“ON CONFLICT”风格的 INSERT，以避免应用于“ON CONFLICT DO NOTHING”子句，该子句不包含任何参数，对于“executemany values”模式是安全的。“ON CONFLICT DO UPDATE”仍然被阻止使用“executemany values”，因为在 DO UPDATE 子句中可能有无法批量处理的额外参数（这是[#6581](https://www.sqlalchemy.org/trac/ticket/6581)修复的原始问题）。

    参考：[#7880](https://www.sqlalchemy.org/trac/ticket/7880)

## 1.4.33

发布日期：2022 年 3 月 31 日

### orm

+   **[orm] [usecase]**

    添加了`with_polymorphic.adapt_on_names`到`with_polymorphic()`函数，允许针对将仅根据列名适应到原始映射可选择项的替代可选择项进行多态加载（通常使用具体映射）。

    参考：[#7805](https://www.sqlalchemy.org/trac/ticket/7805)

+   **[orm] [usecase]**

    添加了新属性`UpdateBase.returning_column_descriptions`和`UpdateBase.entity_description`，允许检查作为`Insert`、`Update`或`Delete`构造的 ORM 属性和实体。对于仅限于 Core 的可选择项，现在还实现了`Select.column_descriptions`访问器。

    参考：[#7861](https://www.sqlalchemy.org/trac/ticket/7861)

+   **[orm] [performance] [bug]**

    ORM 在内存使用方面有所改进，删除了一组重要的中间表达式对象，这些对象通常在创建表达式对象的副本时存储。这些克隆对象已大大减少，将 ORM 映射中存储的总表达式对象数量减少约 30%。

    参考：[#7823](https://www.sqlalchemy.org/trac/ticket/7823)

+   **[orm] [bug] [regression]**

    修复了“动态”加载策略中的回归，其中`Query.filter_by()`方法在查询的关系中存在“次要”表且映射针对复杂内容（如“with polymorphic”）时，不会给出适当的实体进行过滤。

    参考：[#7868](https://www.sqlalchemy.org/trac/ticket/7868)

+   **[orm] [错误]**

    修复了`composite()`属性与连接表继承的`selectin_polymorphic()`加载策略不兼容的错误。

    参考：[#7801](https://www.sqlalchemy.org/trac/ticket/7801)

+   **[orm] [错误]**

    修复了`selectin_polymorphic()`加载选项在没有固定“polymorphic_on”列的连接继承映射器上无法工作的问题。此外，还增加了对此结构的更广泛用法模式的测试支持。

    参考：[#7799](https://www.sqlalchemy.org/trac/ticket/7799)

+   **[orm] [错误]**

    修复了`with_loader_criteria()`函数中的错误，其中加载条件不会应用于在父对象的刷新操作范围内调用的连接急加载。

    参考：[#7862](https://www.sqlalchemy.org/trac/ticket/7862)

+   **[orm] [错误]**

    修复了`Mapper`在映射到`UNION`时过于激进地减少用户定义的`Mapper.primary_key`参数的错误，其中对于某些 SELECT 条目，两列本质上是等效的，但在另一个条目中，它们不是，例如在递归 CTE 中。这里的逻辑已更改为接受给定的用户定义 PK，其中列将与映射的可选择相关联，但不再“减少”，因为这种启发式无法适应所有情况。

    参考：[#7842](https://www.sqlalchemy.org/trac/ticket/7842)

### 引擎

+   **[引擎] [用例]**

    添加了新参数`Engine.dispose.close`，默认为 True。当为 False 时，引擎处理不会触及旧池中的连接，只是丢弃池并替换它。这种用例是为了当原始池从父进程传输时，父进程可以继续使用这些连接。

    另请参阅

    使用连接池与多进程或 os.fork() - 修订文档

    参考：[#7815](https://www.sqlalchemy.org/trac/ticket/7815), [#7877](https://www.sqlalchemy.org/trac/ticket/7877)

+   **[引擎] [错误]**

    进一步澄清连接级别的日志记录，以指示当使用 AUTOCOMMIT 隔离级别时，BEGIN、ROLLBACK 和 COMMIT 日志消息实际上并不表示真实的事务；消息已扩展以包括 BEGIN 消息本身，并且消息还已被修复以适应直接使用 `Engine` 级别的 `create_engine.isolation_level` 参数时的情况。

    参考：[#7853](https://www.sqlalchemy.org/trac/ticket/7853)

### sql

+   **[sql] [用例]**

    添加了新参数 `FunctionElement.table_valued.joins_implicitly`, 用于 `FunctionElement.table_valued()` 结构。此参数指示提供的表值函数将自动与引用的表执行隐式连接。这实际上禁用了“from linting”功能，例如由于存在此参数而触发的“笛卡尔积”警告。可用于函数，如 `func.json_each()`。

    参考：[#7845](https://www.sqlalchemy.org/trac/ticket/7845)

+   **[sql] [错误]**

    `bindparam.literal_execute` 参数现在参与了 `bindparam()` 的缓存生成，因为它更改了编译器生成的 sql 字符串。以前使用了正确的绑定值，但是在相同查询的后续执行中会忽略 `literal_execute`。

    参考：[#7876](https://www.sqlalchemy.org/trac/ticket/7876)

+   **[sql] [错误] [回归]**

    修复了由 [#7760](https://www.sqlalchemy.org/trac/ticket/7760) 引起的回归，其中 `TextualSelect` 的新功能未在编译器中完全实现，导致与 CTE 和文本语句结合时出现“INSERT FROM SELECT”和“INSERT…ON CONFLICT”等复合 INSERT 结构的问题。

    参考：[#7798](https://www.sqlalchemy.org/trac/ticket/7798)

### 模式

+   **[模式] [用例]**

    添加了支持，使得传递给`Table.to_metadata()`的`Table.to_metadata.referred_schema_fn`可调用对象可以返回值`BLANK_SCHEMA`以指示参考外键应重置为 None。该函数还可以返回`RETAIN_SCHEMA`符号以指示“无更改”，这将与当前行为相同，当前行为也表示无更改。

    参考资料：[#7860](https://www.sqlalchemy.org/trac/ticket/7860)

### sqlite

+   **[sqlite] [bug] [reflection]**

    修复了在 SQLite 下 CHECK 约束的名称不会反映的错误，如果使用引号创建名称，则会出现这种情况，这种情况下名称使用混合大小写或特殊字符。

    参考资料：[#5463](https://www.sqlalchemy.org/trac/ticket/5463)

### mssql

+   **[mssql] [bug] [regression]**

    修复了由[#7160](https://www.sqlalchemy.org/trac/ticket/7160)引起的回归，当 FK 反射与低兼容性级别设置（兼容性级别 80：SQL Server 2000）一起导致“模糊列名”错误。修补程序由@Lin-Your 提供。

    参考资料：[#7812](https://www.sqlalchemy.org/trac/ticket/7812)

### 杂项

+   **[bug] [ext]**

    改进了`association_proxy()`构造尝试在类级别访问目标属性且此访问失败时引发的错误消息。这里的特定用例是当代理到不包括工作类级实现的混合属性时。

    参考资料：[#7827](https://www.sqlalchemy.org/trac/ticket/7827)

## 1.4.32

发布日期：2022 年 3 月 6 日

### orm

+   **[orm] [bug] [regression]**

    修复了 ORM 异常的回归，当 INSERT 默默失败未真正插入行（例如来自触发器）时，由于运行时提前引发的异常由于缺少主键值而不会到达，因此引发了一个不具信息的异常而不是正确的异常。对于 1.4 及以上版本，在此情况下添加了一个新的`FlushError`，它比 1.3 的以前的“空标识”异常更早地被引发，因为实际 INSERT 的行数不符合预期的情况在 1.4 中是一个更关键的情况，因为它阻止了多个对象的批处理正确工作。这与新提取的主键值被提取为 NULL 的情况不同，后者继续引发现有的“空标识”异常。

    参考：[#7594](https://www.sqlalchemy.org/trac/ticket/7594)

+   **[ORM] [错误]**

    修复了在`relationship()`中使用完全合格路径的类名的问题，尽管其中包含了不正确的路径标记名称，但不是第一个标记，但是失败不会引发具有信息的错误，而是在稍后的步骤中随机失败。

    参考：[#7697](https://www.sqlalchemy.org/trac/ticket/7697)

### 引擎

+   **[引擎] [错误]**

    调整了关键的 SQLAlchemy 组件的日志记录，包括`Engine`，`Connection`以建立适当的堆栈级别参数，这样当在自定义日志格式化程序中使用 Python 日志标记`funcName`和`lineno`时，将报告正确的信息，这在过滤日志输出时可能很有用；在 Python 3.8 及以上版本上支持。感谢 Markus Gerstel 的贡献的拉取请求。

    参考：[#7612](https://www.sqlalchemy.org/trac/ticket/7612)

### SQL

+   **[SQL] [错误]**

    修复了由于字符串格式错误而导致值为元组的错误消息失败的问题，包括对不支持的文字值和无效的布尔值的编译。

    参考：[#7721](https://www.sqlalchemy.org/trac/ticket/7721)

+   **[SQL] [错误] [MySQL]**

    修复了 MySQL `SET` 数据类型以及通用 `Enum` 数据类型中的问题，在这些类型的`__repr__()`方法不会在字符串输出中呈现所有可选参数，影响这些类型在 Alembic 自动生成中的使用。MySQL 的拉取请求由 Yuki Nishimine 提供。

    参考：[#7598](https://www.sqlalchemy.org/trac/ticket/7598), [#7720](https://www.sqlalchemy.org/trac/ticket/7720), [#7789](https://www.sqlalchemy.org/trac/ticket/7789)

+   **[SQL] [错误]**

    现在，如果指定了`Enum.length`参数而没有同时指定`Enum.native_enum`为 False，`Enum`数据类型会发出警告，因为在这种情况下参数将被静默忽略，尽管`Enum`数据类型仍会在没有原生 ENUM 数据类型的后端（如 SQLite）上渲染 VARCHAR DDL。这种行为在未来的发布中可能会更改，以便无论“native_enum”设置如何，“length”都会适用于所有非本机“enum”类型。

+   **[sql] [bug]**

    修复了当在`TextualSelect`实例上调用`HasCTE.add_cte()`方法时，SQL 编译器未进行适配的问题。此修复还将更多“SELECT”类似的编译器行为添加到`TextualSelect`中，包括可容纳 DML CTEs（如 UPDATE 和 INSERT）。

    参考：[#7760](https://www.sqlalchemy.org/trac/ticket/7760)

### asyncio

+   **[asyncio] [bug]**

    修复了在某些事件监听类别中未为异步引擎引发描述性错误消息的问题，应该是同步引擎实例。

+   **[asyncio] [bug]**

    修复了当使用不兼容同步风格`Result`对象的流式结果执行选项`Connection.execution_options.stream_results`时，`AsyncSession.execute()`方法未引发信息丰富的异常的问题。在这种情况下，现在会引发异常，就像在与`AsyncConnection.execute()`方法一起使用`Connection.execution_options.stream_results`选项时已引发异常一样。

    此外，为了提高与状态敏感的数据库驱动程序（如 asyncmy）的稳定性，当出现此错误条件时，游标现在会被关闭；以前在 asyncmy 方言中，连接会进入无效状态，服务器端结果仍未消耗。

    参考：[#7667](https://www.sqlalchemy.org/trac/ticket/7667)

### postgresql

+   **[postgresql] [usecase]**

    添加了对 PostgreSQL `NOT VALID` 短语的编译器支持，用于渲染 `CheckConstraint`、`ForeignKeyConstraint` 和 `ForeignKey` 架构构造的 DDL。感谢 Gilbert Gilb 的拉取请求。

    另请参阅

    PostgreSQL 约束选项

    参考：[#7600](https://www.sqlalchemy.org/trac/ticket/7600)

### mysql

+   **[mysql] [bug] [regression]**

    由 [#7518](https://www.sqlalchemy.org/trac/ticket/7518) 引起的回归问题，其中将语法“SHOW VARIABLES”更改为“SELECT @@”破坏了与早于 5.6 版本的 MySQL 版本（包括早期 5.0 发行版）的兼容性。虽然这些是非常旧的 MySQL 版本，但并未计划更改兼容性，因此已恢复了特定于版本的逻辑，以便在 MySQL 服务器版本 < 5.6 时回退到“SHOW VARIABLES”。

    参考：[#7518](https://www.sqlalchemy.org/trac/ticket/7518)

### mariadb

+   **[mariadb] [bug] [regression]**

    修复了 mariadbconnector 方言中的回归问题，截至 mariadb 连接器 1.0.10，DBAPI 不再预缓冲 cursor.lastrowid，导致在使用 ORM 插入对象时出错，同时导致 `CursorResult.inserted_primary_key` 属性不可用。该方言现在主动获取此值，以适用的情况。

    参考：[#7738](https://www.sqlalchemy.org/trac/ticket/7738)

### sqlite

+   **[sqlite] [usecase]**

    添加了对反映 SQLite 内联唯一约束的支持，其中列名使用 SQLite 的“转义引号” `[]` 或 ```py, which are discarded by the database when producing the column name.

    References: [#7736](https://www.sqlalchemy.org/trac/ticket/7736)

*   **[sqlite] [bug]**

    Fixed issue where SQLite unique constraint reflection would fail to detect a column-inline UNIQUE constraint where the column name had an underscore in its name.

    References: [#7736](https://www.sqlalchemy.org/trac/ticket/7736)

### oracle

*   **[oracle] [bug]**

    Fixed issue in Oracle dialect where using a column name that requires quoting when written as a bound parameter, such as `"_id"`, would not correctly track a Python generated default value due to the bound-parameter rewriting missing this value, causing an Oracle error to be raised.

    References: [#7676](https://www.sqlalchemy.org/trac/ticket/7676)

*   **[oracle] [bug] [regression]**

    Added support to parse “DPI” error codes from cx_Oracle exception objects such as `DPI-1080` and `DPI-1010`, both of which now indicate a disconnect scenario as of cx_Oracle 8.3.

    References: [#7748](https://www.sqlalchemy.org/trac/ticket/7748)

### tests

*   **[tests] [bug]**

    Improvements to the test suite’s integration with pytest such that the “warnings” plugin, if manually enabled, will not interfere with the test suite, such that third parties can enable the warnings plugin or make use of the `-W` parameter and SQLAlchemy’s test suite will continue to pass. Additionally, modernized the detection of the “pytest-xdist” plugin so that plugins can be globally disabled using PYTEST_DISABLE_PLUGIN_AUTOLOAD=1 without breaking the test suite if xdist were still installed. Warning filters that promote deprecation warnings to errors are now localized to SQLAlchemy-specific warnings, or within SQLAlchemy-specific sources for general Python deprecation warnings, so that non-SQLAlchemy deprecation warnings emitted from pytest plugins should also not impact the test suite.

    References: [#7599](https://www.sqlalchemy.org/trac/ticket/7599)

*   **[tests] [bug]**

    Made corrections to the default pytest configuration regarding how test discovery is configured, to fix issue where the test suite would not configure warnings correctly and also attempt to load example suites as tests, in the specific case where the SQLAlchemy checkout were located in an absolute path that had a super-directory named “test”.

    References: [#7045](https://www.sqlalchemy.org/trac/ticket/7045)

## 1.4.31

Released: January 20, 2022

### orm

*   **[orm] [bug]**

    Fixed issue in `Session.bulk_save_objects()` where the sorting that takes place when the `preserve_order` parameter is set to False would sort partially on `Mapper` objects, which is rejected in Python 3.11.

    References: [#7591](https://www.sqlalchemy.org/trac/ticket/7591)

### postgresql

*   **[postgresql] [bug] [regression]**

    Fixed regression where the change in [#7148](https://www.sqlalchemy.org/trac/ticket/7148) to repair ENUM handling in PostgreSQL broke the use case of an empty ARRAY of ENUM, preventing rows that contained an empty array from being handled correctly when fetching results.

    References: [#7590](https://www.sqlalchemy.org/trac/ticket/7590)

### mysql

*   **[mysql] [bug] [regression]**

    Fixed regression in asyncmy dialect caused by [#7567](https://www.sqlalchemy.org/trac/ticket/7567) where removal of the PyMySQL dependency broke binary columns, due to the asyncmy dialect not being properly included within CI tests.

    References: [#7593](https://www.sqlalchemy.org/trac/ticket/7593)

### mssql

*   **[mssql]**

    Added support for `FILESTREAM` when using `VARBINARY(max)` in MSSQL.

    See also

    `VARBINARY.filestream`

    References: [#7243](https://www.sqlalchemy.org/trac/ticket/7243)

## 1.4.30

Released: January 19, 2022

### orm

*   **[orm] [bug]**

    Fixed issue in joined-inheritance load of additional attributes functionality in deep multi-level inheritance where an intermediary table that contained no columns would not be included in the tables joined, instead linking those tables to their primary key identifiers. While this works fine, it nonetheless in 1.4 began producing the cartesian product compiler warning. The logic has been changed so that these intermediary tables are included regardless. While this does include additional tables in the query that are not technically necessary, this only occurs for the highly unusual case of deep 3+ level inheritance with intermediary tables that have no non primary key columns, potential performance impact is therefore expected to be negligible.

    References: [#7507](https://www.sqlalchemy.org/trac/ticket/7507)

*   **[orm] [bug]**

    Fixed issue where calling upon `registry.map_imperatively()` more than once for the same class would produce an unexpected error, rather than an informative error that the target class is already mapped. This behavior differed from that of the `mapper()` function which does report an informative message already.

    References: [#7579](https://www.sqlalchemy.org/trac/ticket/7579)

*   **[orm] [bug] [asyncio]**

    Added missing method `AsyncSession.invalidate()` to the `AsyncSession` class.

    References: [#7524](https://www.sqlalchemy.org/trac/ticket/7524)

*   **[orm] [bug] [regression]**

    Fixed regression which appeared in 1.4.23 which could cause loader options to be mis-handled in some cases, in particular when using joined table inheritance in combination with the `polymorphic_load="selectin"` option as well as relationship lazy loading, leading to a `TypeError`.

    References: [#7557](https://www.sqlalchemy.org/trac/ticket/7557)

*   **[orm] [bug] [regression]**

    Fixed ORM regression where calling the `aliased()` function against an existing `aliased()` construct would fail to produce correct SQL if the existing construct were against a fixed table. The fix allows that the original `aliased()` construct is disregarded if it were only against a table that’s now being replaced. It also allows for correct behavior when constructing a `aliased()` without a selectable argument against a `aliased()` that’s against a subuquery, to create an alias of that subquery (i.e. to change its name).

    The nesting behavior of `aliased()` remains in place for the case where the outer `aliased()` object is against a subquery which in turn refers to the inner `aliased()` object. This is a relatively new 1.4 feature that helps to suit use cases that were previously served by the deprecated `Query.from_self()` method.

    References: [#7576](https://www.sqlalchemy.org/trac/ticket/7576)

*   **[orm] [bug]**

    Fixed issue where `Select.correlate_except()` method, when passed either the `None` value or no arguments, would not correlate any elements when used in an ORM context (that is, passing ORM entities as FROM clauses), rather than causing all FROM elements to be considered as “correlated” in the same way which occurs when using Core-only constructs.

    References: [#7514](https://www.sqlalchemy.org/trac/ticket/7514)

*   **[orm] [bug] [regression]**

    Fixed regression from 1.3 where the “subqueryload” loader strategy would fail with a stack trace if used against a query that made use of `Query.from_statement()` or `Select.from_statement()`. As subqueryload requires modifying the original statement, it’s not compatible with the “from_statement” use case, especially for statements made against the `text()` construct. The behavior now is equivalent to that of 1.3 and previously, which is that the loader strategy silently degrades to not be used for such statements, typically falling back to using the lazyload strategy.

    References: [#7505](https://www.sqlalchemy.org/trac/ticket/7505)

### sql

*   **[sql] [bug] [postgresql]**

    Added additional rule to the system that determines `TypeEngine` implementations from Python literals to apply a second level of adjustment to the type, so that a Python datetime with or without tzinfo can set the `timezone=True` parameter on the returned `DateTime` object, as well as `Time`. This helps with some round-trip scenarios on type-sensitive PostgreSQL dialects such as asyncpg, psycopg3 (2.0 only).

    References: [#7537](https://www.sqlalchemy.org/trac/ticket/7537)

*   **[sql] [bug]**

    Added an informative error message when a method object is passed to a SQL construct. Previously, when such a callable were passed, as is a common typographical error when dealing with method-chained SQL constructs, they were interpreted as “lambda SQL” targets to be invoked at compilation time, which would lead to silent failures. As this feature was not intended to be used with methods, method objects are now rejected.

    References: [#7032](https://www.sqlalchemy.org/trac/ticket/7032)

### mypy

*   **[mypy] [bug]**

    Fixed Mypy crash when running id daemon mode caused by a missing attribute on an internal mypy `Var` instance.

    References: [#7321](https://www.sqlalchemy.org/trac/ticket/7321)

### asyncio

*   **[asyncio] [usecase]**

    Added new method `AdaptedConnection.run_async()` to the DBAPI connection interface used by asyncio drivers, which allows methods to be called against the underlying “driver” connection directly within a sync-style function where the `await` keyword can’t be used, such as within SQLAlchemy event handler functions. The method is analogous to the `AsyncConnection.run_sync()` method which translates async-style calls to sync-style. The method is useful for things like connection-pool on-connect handlers that need to invoke awaitable methods on the driver connection when it’s first created.

    See also

    Using awaitable-only driver methods in connection pool and other events

    References: [#7580](https://www.sqlalchemy.org/trac/ticket/7580)

### postgresql

*   **[postgresql] [usecase]**

    Added string rendering to the `UUID` datatype, so that stringifying a statement with “literal_binds” that uses this type will render an appropriate string value for the PostgreSQL backend. Pull request courtesy José Duarte.

    References: [#7561](https://www.sqlalchemy.org/trac/ticket/7561)

*   **[postgresql] [bug] [asyncpg]**

    Improved support for asyncpg handling of TIME WITH TIMEZONE, which was not fully implemented.

    References: [#7537](https://www.sqlalchemy.org/trac/ticket/7537)

*   **[postgresql] [bug] [mssql] [reflection]**

    Fixed reflection of covering indexes to report `include_columns` as part of the `dialect_options` entry in the reflected index dictionary, thereby enabling round trips from reflection->create to be complete. Included columns continue to also be present under the `include_columns` key for backwards compatibility.

    References: [#7382](https://www.sqlalchemy.org/trac/ticket/7382)

*   **[postgresql] [bug]**

    Fixed handling of array of enum values which require escape characters.

    References: [#7418](https://www.sqlalchemy.org/trac/ticket/7418)

### mysql

*   **[mysql] [change]**

    Replace `SHOW VARIABLES LIKE` statement with equivalent `SELECT @@variable` in MySQL and MariaDB dialect initialization. This should avoid mutex contention caused by `SHOW VARIABLES`, improving initialization performance.

    References: [#7518](https://www.sqlalchemy.org/trac/ticket/7518)

*   **[mysql] [bug]**

    Removed unnecessary dependency on PyMySQL from the asyncmy dialect. Pull request courtesy long2ice.

    References: [#7567](https://www.sqlalchemy.org/trac/ticket/7567)

## 1.4.29

Released: December 22, 2021

### orm

*   **[orm] [usecase]**

    Added `Session.get.execution_options` parameter which was previously missing from the `Session.get()` method.

    References: [#7410](https://www.sqlalchemy.org/trac/ticket/7410)

*   **[orm] [bug]**

    Fixed issue in new “loader criteria” method `PropComparator.and_()` where usage with a loader strategy like `selectinload()` against a column that was a member of the `.c.` collection of a subquery object, where the subquery would be dynamically added to the FROM clause of the statement, would be subject to stale parameter values within the subquery in the SQL statement cache, as the process used by the loader strategy to replace the parameters at execution time would fail to accommodate the subquery when received in this form.

    References: [#7489](https://www.sqlalchemy.org/trac/ticket/7489)

*   **[orm] [bug]**

    Fixed recursion overflow which could occur within ORM statement compilation when using either the `with_loader_criteria()` feature or the the `PropComparator.and_()` method within a loader strategy in conjunction with a subquery which referred to the same entity being altered by the criteria option, or loaded by the loader strategy. A check for coming across the same loader criteria option in a recursive fashion has been added to accommodate for this scenario.

    References: [#7491](https://www.sqlalchemy.org/trac/ticket/7491)

*   **[orm] [bug] [mypy]**

    Fixed issue where the `__class_getitem__()` method of the generated declarative base class by `as_declarative()` would lead to inaccessible class attributes such as `__table__`, for cases where a `Generic[T]` style typing declaration were used in the class hierarchy. This is in continuation from the basic addition of `__class_getitem__()` in [#7368](https://www.sqlalchemy.org/trac/ticket/7368). Pull request courtesy Kai Mueller.

    References: [#7368](https://www.sqlalchemy.org/trac/ticket/7368), [#7462](https://www.sqlalchemy.org/trac/ticket/7462)

*   **[orm] [bug] [regression]**

    Fixed caching-related issue where the use of a loader option of the form `lazyload(aliased(A).bs).joinedload(B.cs)` would fail to result in the joinedload being invoked for runs subsequent to the query being cached, due to a mismatch for the options / object path applied to the objects loaded for a query with a lead entity that used `aliased()`.

    References: [#7447](https://www.sqlalchemy.org/trac/ticket/7447)

### engine

*   **[engine] [bug]**

    Corrected the error message for the `AttributeError` that’s raised when attempting to write to an attribute on the `Row` class, which is immutable. The previous message claimed the column didn’t exist which is misleading.

    References: [#7432](https://www.sqlalchemy.org/trac/ticket/7432)

*   **[engine] [bug] [regression]**

    Fixed regression in the `make_url()` function used to parse URL strings where the query string parsing would go into a recursion overflow if a Python 2 `u''` string were used.

    References: [#7446](https://www.sqlalchemy.org/trac/ticket/7446)

### mypy

*   **[mypy] [bug]**

    Fixed mypy regression where the release of mypy 0.930 added additional internal checks to the format of “named types”, requiring that they be fully qualified and locatable. This broke the mypy plugin for SQLAlchemy, raising an assertion error, as there was use of symbols such as `__builtins__` and other un-locatable or unqualified names that previously had not raised any assertions.

    References: [#7496](https://www.sqlalchemy.org/trac/ticket/7496)

### asyncio

*   **[asyncio] [usecase]**

    Added `async_engine_config()` function to create an async engine from a configuration dict. This otherwise behaves the same as `engine_from_config()`.

    References: [#7301](https://www.sqlalchemy.org/trac/ticket/7301)

### mariadb

*   **[mariadb] [bug]**

    Corrected the error classes inspected for the “is_disconnect” check for the `mariadbconnector` dialect, which was failing for disconnects that occurred due to common MySQL/MariaDB error codes such as 2006; the DBAPI appears to currently use the `mariadb.InterfaceError` exception class for disconnect errors such as error code 2006, which has been added to the list of classes checked.

    References: [#7457](https://www.sqlalchemy.org/trac/ticket/7457)

### tests

*   **[tests] [bug] [regression]**

    Fixed a regression in the test suite where the test called `CompareAndCopyTest::test_all_present` would fail on some platforms due to additional testing artifacts being detected. Pull request courtesy Nils Philippsen.

    References: [#7450](https://www.sqlalchemy.org/trac/ticket/7450)

## 1.4.28

Released: December 9, 2021

### platform

*   **[platform] [bug]**

    Python 3.10 has deprecated “distutils” in favor of explicit use of “setuptools” in [**PEP 632**](https://peps.python.org/pep-0632/); SQLAlchemy’s setup.py has replaced imports accordingly. However, since setuptools itself only recently added the replacement symbols mentioned in pep-632 as of November of 2021 in version 59.0.1, `setup.py` still has fallback imports to distutils, as SQLAlchemy 1.4 does not have a hard setuptools versioning requirement at this time. SQLAlchemy 2.0 is expected to use a full [**PEP 517**](https://peps.python.org/pep-0517/) installation layout which will indicate appropriate setuptools versioning up front.

    References: [#7311](https://www.sqlalchemy.org/trac/ticket/7311)

### orm

*   **[orm] [bug] [ext]**

    Fixed issue where the internal cloning used by the `PropComparator.any()` method on a `relationship()` in the case where the related class also makes use of ORM polymorphic loading, would fail if a hybrid property on the related, polymorphic class were used within the criteria for the `any()` operation.

    References: [#7425](https://www.sqlalchemy.org/trac/ticket/7425)

*   **[orm] [bug] [mypy]**

    Fixed issue where the `as_declarative()` decorator and similar functions used to generate the declarative base class would not copy the `__class_getitem__()` method from a given superclass, which prevented the use of pep-484 generics in conjunction with the `Base` class. Pull request courtesy Kai Mueller.

    References: [#7368](https://www.sqlalchemy.org/trac/ticket/7368)

*   **[orm] [bug] [regression]**

    Fixed ORM regression where the new behavior of “eager loaders run on unexpire” added in [#1763](https://www.sqlalchemy.org/trac/ticket/1763) would lead to loader option errors being raised inappropriately for the case where a single `Query` or `Select` were used to load multiple kinds of entities, along with loader options that apply to just one of those kinds of entity like a `joinedload()`, and later the objects would be refreshed from expiration, where the loader options would attempt to be applied to the mismatched object type and then raise an exception. The check for this mismatch now bypasses raising an error for this case.

    References: [#7318](https://www.sqlalchemy.org/trac/ticket/7318)

*   **[orm] [bug]**

    User defined ORM options, such as those illustrated in the dogpile.caching example which subclass `UserDefinedOption`, by definition are handled on every statement execution and do not need to be considered as part of the cache key for the statement. Caching of the base `ExecutableOption` class has been modified so that it is no longer a `HasCacheKey` subclass directly, so that the presence of user defined option objects will not have the unwanted side effect of disabling statement caching. Only ORM specific loader and criteria options, which are all internal to SQLAlchemy, now participate within the caching system.

    References: [#7394](https://www.sqlalchemy.org/trac/ticket/7394)

*   **[orm] [bug]**

    Fixed issue where mappings that made use of `synonym()` and potentially other kinds of “proxy” attributes would not in all cases successfully generate a cache key for their SQL statements, leading to degraded performance for those statements.

    References: [#7394](https://www.sqlalchemy.org/trac/ticket/7394)

*   **[orm] [bug]**

    Fixed issue where a list mapped with `relationship()` would go into an endless loop if in-place added to itself, i.e. the `+=` operator were used, as well as if `.extend()` were given the same list.

    References: [#7389](https://www.sqlalchemy.org/trac/ticket/7389)

*   **[orm] [bug]**

    Fixed issue where if an exception occurred when the `Session` were to close the connection within the `Session.commit()` method, when using a context manager for `Session.begin()` , it would attempt a rollback which would not be possible as the `Session` was in between where the transaction is committed and the connection is then to be returned to the pool, raising the exception “this sessiontransaction is in the committed state”. This exception can occur mostly in an asyncio context where CancelledError can be raised.

    References: [#7388](https://www.sqlalchemy.org/trac/ticket/7388)

*   **[orm] [deprecated]**

    Deprecated an undocumented loader option syntax `".*"`, which appears to be no different than passing a single asterisk, and will emit a deprecation warning if used. This syntax may have been intended for something but there is currently no need for it.

    References: [#4390](https://www.sqlalchemy.org/trac/ticket/4390)

### engine

*   **[engine] [usecase]**

    Added support for `copy()` and `deepcopy()` to the `URL` class. Pull request courtesy Tom Ritchford.

    References: [#7400](https://www.sqlalchemy.org/trac/ticket/7400)

### sql

*   **[sql] [usecase]**

    ”Compound select” methods like `Select.union()`, `Select.intersect_all()` etc. now accept `*other` as an argument rather than `other` to allow for multiple additional SELECTs to be compounded with the parent statement at once. In particular, the change as applied to `CTE.union()` and `CTE.union_all()` now allow for a so-called “non-linear CTE” to be created with the `CTE` construct, whereas previously there was no way to have more than two CTE sub-elements in a UNION together while still correctly calling upon the CTE in recursive fashion. Pull request courtesy Eric Masseran.

    References: [#7259](https://www.sqlalchemy.org/trac/ticket/7259)

*   **[sql] [usecase]**

    Support multiple clause elements in the `Exists.where()` method, unifying the api with the one presented by a normal `select()` construct.

    References: [#7386](https://www.sqlalchemy.org/trac/ticket/7386)

*   **[sql] [bug] [regression]**

    Extended the `TypeDecorator.cache_ok` attribute and corresponding warning message if this flag is not defined, a behavior first established for `TypeDecorator` as part of [#6436](https://www.sqlalchemy.org/trac/ticket/6436), to also take place for `UserDefinedType`, by generalizing the flag and associated caching logic to a new common base for these two types, `ExternalType` to create `UserDefinedType.cache_ok`.

    The change means any current `UserDefinedType` will now cause SQL statement caching to no longer take place for statements which make use of the datatype, along with a warning being emitted, unless the class defines the `UserDefinedType.cache_ok` flag as True. If the datatype cannot form a deterministic, hashable cache key derived from its arguments, the attribute may be set to False which will continue to keep caching disabled but will suppress the warning. In particular, custom datatypes currently used in packages such as SQLAlchemy-utils will need to implement this flag. The issue was observed as a result of a SQLAlchemy-utils datatype that is not currently cacheable.

    See also

    `ExternalType.cache_ok`

    References: [#7319](https://www.sqlalchemy.org/trac/ticket/7319)

*   **[sql] [bug]**

    Custom SQL elements, third party dialects, custom or third party datatypes will all generate consistent warnings when they do not clearly opt in or out of SQL statement caching, which is achieved by setting the appropriate attributes on each type of class. The warning links to documentation sections which indicate the appropriate approach for each type of object in order for caching to be enabled.

    References: [#7394](https://www.sqlalchemy.org/trac/ticket/7394)

*   **[sql] [bug]**

    Fixed missing caching directives for a few lesser used classes in SQL Core which would cause `[no key]` to be logged for elements which made use of these.

    References: [#7394](https://www.sqlalchemy.org/trac/ticket/7394)

### mypy

*   **[mypy] [bug]**

    Fixed Mypy crash which would occur when using Mypy plugin against code which made use of `declared_attr` methods for non-mapped names like `__mapper_args__`, `__table_args__`, or other dunder names, as the plugin would try to interpret these as mapped attributes which would then be later mis-handled. As part of this change, the decorated function is still converted by the plugin into a generic assignment statement (e.g. `__mapper_args__: Any`) so that the argument signature can continue to be annotated in the same way one would for any other `@classmethod` without Mypy complaining about the wrong argument type for a method that isn’t explicitly `@classmethod`.

    References: [#7321](https://www.sqlalchemy.org/trac/ticket/7321)

### postgresql

*   **[postgresql] [bug]**

    Fixed missing caching directives for `hstore` and `array` constructs which would cause `[no key]` to be logged for these elements.

    References: [#7394](https://www.sqlalchemy.org/trac/ticket/7394)

### tests

*   **[tests] [bug]**

    Implemented support for the test suite to run correctly under Pytest 7. Previously, only Pytest 6.x was supported for Python 3, however the version was not pinned on the upper bound in tox.ini. Pytest is not pinned in tox.ini to be lower than version 8 so that SQLAlchemy versions released with the current codebase will be able to be tested under tox without changes to the environment. Much thanks to the Pytest developers for their help with this issue.

## 1.4.27

Released: November 11, 2021

### orm

*   **[orm] [bug]**

    Fixed bug in “relationship to aliased class” feature introduced at Relationship to Aliased Class where it was not possible to create a loader strategy option targeting an attribute on the target using the `aliased()` construct directly in a second loader option, such as `selectinload(A.aliased_bs).joinedload(aliased_b.cs)`, without explicitly qualifying using `PropComparator.of_type()` on the preceding element of the path. Additionally, targeting the non-aliased class directly would be accepted (inappropriately), but would silently fail, such as `selectinload(A.aliased_bs).joinedload(B.cs)`; this now raises an error referring to the typing mismatch.

    References: [#7224](https://www.sqlalchemy.org/trac/ticket/7224)

*   **[orm] [bug]**

    All `Result` objects will now consistently raise `ResourceClosedError` if they are used after a hard close, which includes the “hard close” that occurs after calling “single row or value” methods like `Result.first()` and `Result.scalar()`. This was already the behavior of the most common class of result objects returned for Core statement executions, i.e. those based on `CursorResult`, so this behavior is not new. However, the change has been extended to properly accommodate for the ORM “filtering” result objects returned when using 2.0 style ORM queries, which would previously behave in “soft closed” style of returning empty results, or wouldn’t actually “soft close” at all and would continue yielding from the underlying cursor.

    As part of this change, also added `Result.close()` to the base `Result` class and implemented it for the filtered result implementations that are used by the ORM, so that it is possible to call the `CursorResult.close()` method on the underlying `CursorResult` when the `yield_per` execution option is in use to close a server side cursor before remaining ORM results have been fetched. This was again already available for Core result sets but the change makes it available for 2.0 style ORM results as well.

    References: [#7274](https://www.sqlalchemy.org/trac/ticket/7274)

*   **[orm] [bug] [regression]**

    Fixed 1.4 regression where `Query.filter_by()` would not function correctly on a `Query` that was produced from `Query.union()`, `Query.from_self()` or similar.

    References: [#7239](https://www.sqlalchemy.org/trac/ticket/7239)

*   **[orm] [bug]**

    Fixed issue where deferred polymorphic loading of attributes from a joined-table inheritance subclass would fail to populate the attribute correctly if the `load_only()` option were used to originally exclude that attribute, in the case where the load_only were descending from a relationship loader option. The fix allows that other valid options such as `defer(..., raiseload=True)` etc. still function as expected.

    References: [#7304](https://www.sqlalchemy.org/trac/ticket/7304)

*   **[orm] [bug] [regression]**

    Fixed 1.4 regression where `Query.filter_by()` would not function correctly when `Query.join()` were joined to an entity which made use of `PropComparator.of_type()` to specify an aliased version of the target entity. The issue also applies to future style ORM queries constructed with `select()`.

    References: [#7244](https://www.sqlalchemy.org/trac/ticket/7244)

### engine

*   **[engine] [bug]**

    Fixed issue in future `Connection` object where the `Connection.execute()` method would not accept a non-dict mapping object, such as SQLAlchemy’s own `RowMapping` or other `abc.collections.Mapping` object as a parameter dictionary.

    References: [#7291](https://www.sqlalchemy.org/trac/ticket/7291)

*   **[engine] [bug] [regression]**

    Fixed regression where the `CursorResult.fetchmany()` method would fail to autoclose a server-side cursor (i.e. when `stream_results` or `yield_per` is in use, either Core or ORM oriented results) when the results were fully exhausted.

    References: [#7274](https://www.sqlalchemy.org/trac/ticket/7274)

*   **[engine] [bug]**

    Fixed issue in future `Engine` where calling upon `Engine.begin()` and entering the context manager would not close the connection if the actual BEGIN operation failed for some reason, such as an event handler raising an exception; this use case failed to be tested for the future version of the engine. Note that the “future” context managers which handle `begin()` blocks in Core and ORM don’t actually run the “BEGIN” operation until the context managers are actually entered. This is different from the legacy version which runs the “BEGIN” operation up front.

    References: [#7272](https://www.sqlalchemy.org/trac/ticket/7272)

### sql

*   **[sql] [usecase]**

    Added `TupleType` to the top level `sqlalchemy` import namespace.

*   **[sql] [bug] [regression]**

    Fixed regression where the row objects returned for ORM queries, which are now the normal `Row` objects, would not be interpreted by the `ColumnOperators.in_()` operator as tuple values to be broken out into individual bound parameters, and would instead pass them as single values to the driver leading to failures. The change to the “expanding IN” system now accommodates for the expression already being of type `TupleType` and treats values accordingly if so. In the uncommon case of using “tuple-in” with an untyped statement such as a textual statement with no typing information, a tuple value is detected for values that implement `collections.abc.Sequence`, but that are not `str` or `bytes`, as always when testing for `Sequence`.

    References: [#7292](https://www.sqlalchemy.org/trac/ticket/7292)

*   **[sql] [bug]**

    Fixed issue where using the feature of using a string label for ordering or grouping described at Ordering or Grouping by a Label would fail to function correctly if used on a `CTE` construct, when the CTE were embedded inside of an enclosing `Select` statement that itself was set up as a scalar subquery.

    References: [#7269](https://www.sqlalchemy.org/trac/ticket/7269)

*   **[sql] [bug] [regression]**

    Fixed regression where the `text()` construct would no longer be accepted as a target case in the “whens” list within a `case()` construct. The regression appears related to an attempt to guard against some forms of literal values that were considered to be ambiguous when passed here; however, there’s no reason the target cases shouldn’t be interpreted as open-ended SQL expressions just like anywhere else, and a literal string or tuple will be converted to a bound parameter as would be the case elsewhere.

    References: [#7287](https://www.sqlalchemy.org/trac/ticket/7287)

### schema

*   **[schema] [bug]**

    Fixed issue in `Table` where the `Table.implicit_returning` parameter would not be accommodated correctly when passed along with `Table.extend_existing` to augment an existing `Table`.

    References: [#7295](https://www.sqlalchemy.org/trac/ticket/7295)

### postgresql

*   **[postgresql] [usecase] [asyncpg]**

    Added overridable methods `PGDialect_asyncpg.setup_asyncpg_json_codec` and `PGDialect_asyncpg.setup_asyncpg_jsonb_codec` codec, which handle the required task of registering JSON/JSONB codecs for these datatypes when using asyncpg. The change is that methods are broken out as individual, overridable methods to support third party dialects that need to alter or disable how these particular codecs are set up.

    References: [#7284](https://www.sqlalchemy.org/trac/ticket/7284)

*   **[postgresql] [bug] [asyncpg]**

    Changed the asyncpg dialect to bind the `Float` type to the “float” PostgreSQL type instead of “numeric” so that the value `float(inf)` can be accommodated. Added test suite support for persistence of the “inf” value.

    References: [#7283](https://www.sqlalchemy.org/trac/ticket/7283)

*   **[postgresql] [pg8000]**

    Improve array handling when using PostgreSQL with the pg8000 dialect.

    References: [#7167](https://www.sqlalchemy.org/trac/ticket/7167)

### mysql

*   **[mysql] [bug] [mariadb]**

    Reorganized the list of reserved words into two separate lists, one for MySQL and one for MariaDB, so that these diverging sets of words can be managed more accurately; adjusted the MySQL/MariaDB dialect to switch among these lists based on either explicitly configured or server-version-detected “MySQL” or “MariaDB” backend. Added all current reserved words through MySQL 8 and current MariaDB versions including recently added keywords like “lead” . Pull request courtesy Kevin Kirsche.

    References: [#7167](https://www.sqlalchemy.org/trac/ticket/7167)

*   **[mysql] [bug]**

    Fixed issue in MySQL `Insert.on_duplicate_key_update()` which would render the wrong column name when an expression were used in a VALUES expression. Pull request courtesy Cristian Sabaila.

    References: [#7281](https://www.sqlalchemy.org/trac/ticket/7281)

### mssql

*   **[mssql] [bug]**

    Adjusted the compiler’s generation of “post compile” symbols including those used for “expanding IN” as well as for the “schema translate map” to not be based directly on plain bracketed strings with underscores, as this conflicts directly with SQL Server’s quoting format of also using brackets, which produces false matches when the compiler replaces “post compile” and “schema translate” symbols. The issue created easy to reproduce examples both with the `Inspector.get_schema_names()` method when used in conjunction with the `Connection.execution_options.schema_translate_map` feature, as well in the unlikely case that a symbol overlapping with the internal name “POSTCOMPILE” would be used with a feature like “expanding in”.

    References: [#7300](https://www.sqlalchemy.org/trac/ticket/7300)

## 1.4.26

Released: October 19, 2021

### orm

*   **[orm] [bug]**

    Improved the exception message generated when configuring a mapping with joined table inheritance where the two tables either have no foreign key relationships set up, or where they have multiple foreign key relationships set up. The message is now ORM specific and includes context that the `Mapper.inherit_condition` parameter may be needed particularly for the ambiguous foreign keys case.

*   **[orm] [bug]**

    Fixed issue with `with_loader_criteria()` feature where ON criteria would not be added to a JOIN for a query of the form `select(A).join(B)`, stating a target while making use of an implicit ON clause.

    References: [#7189](https://www.sqlalchemy.org/trac/ticket/7189)

*   **[orm] [bug]**

    Fixed bug where the ORM “plugin”, necessary for features such as `with_loader_criteria()` to work correctly, would not be applied to a `select()` which queried from an ORM column expression if it made use of the `ColumnElement.label()` modifier.

    References: [#7205](https://www.sqlalchemy.org/trac/ticket/7205)

*   **[orm] [bug]**

    Add missing methods added in [#6991](https://www.sqlalchemy.org/trac/ticket/6991) to `scoped_session` and `async_scoped_session()`.

    References: [#7103](https://www.sqlalchemy.org/trac/ticket/7103)

*   **[orm] [bug]**

    An extra layer of warning messages has been added to the functionality of `Query.join()` and the ORM version of `Select.join()`, where a few places where “automatic aliasing” continues to occur will now be called out as a pattern to avoid, mostly specific to the area of joined table inheritance where classes that share common base tables are being joined together without using explicit aliases. One case emits a legacy warning for a pattern that’s not recommended, the other case is fully deprecated.

    The automatic aliasing within ORM join() which occurs for overlapping mapped tables does not work consistently with all APIs such as `contains_eager()`, and rather than continue to try to make these use cases work everywhere, replacing with a more user-explicit pattern is clearer, less prone to bugs and simplifies SQLAlchemy’s internals further.

    The warnings include links to the errors.rst page where each pattern is demonstrated along with the recommended pattern to fix.

    See also

    An alias is being generated automatically for raw clauseelement

    An alias is being generated automatically due to overlapping tables

    References: [#6972](https://www.sqlalchemy.org/trac/ticket/6972), [#6974](https://www.sqlalchemy.org/trac/ticket/6974)

*   **[orm] [bug]**

    Fixed bug where iterating a `Result` from a `Session` after that `Session` were closed would partially attach objects to that session in an essentially invalid state. It now raises an exception with a link to new documentation if an **un-buffered** result is iterated from a `Session` that was closed or otherwise had the `Session.expunge_all()` method called after that `Result` was generated. The `prebuffer_rows` execution option, as is used automatically by the asyncio extension for client-side result sets, may be used to produce a `Result` where the ORM objects are prebuffered, and in this case iterating the result will produce a series of detached objects.

    See also

    Object cannot be converted to ‘persistent’ state, as this identity map is no longer valid.

    References: [#7128](https://www.sqlalchemy.org/trac/ticket/7128)

*   **[orm] [bug]**

    Related to [#7153](https://www.sqlalchemy.org/trac/ticket/7153), fixed an issue where result column lookups would fail for “adapted” SELECT statements that selected for “constant” value expressions most typically the NULL expression, as would occur in such places as joined eager loading in conjunction with limit/offset. This was overall a regression due to issue [#6259](https://www.sqlalchemy.org/trac/ticket/6259) which removed all “adaption” for constants like NULL, “true”, and “false” when rewriting expressions in a SQL statement, but this broke the case where the same adaption logic were used to resolve the constant to a labeled expression for the purposes of result set targeting.

    References: [#7154](https://www.sqlalchemy.org/trac/ticket/7154)

*   **[orm] [bug] [regression]**

    Fixed regression where ORM loaded objects could not be pickled in cases where loader options making use of `"*"` were used in certain combinations, such as combining the `joinedload()` loader strategy with `raiseload('*')` of sub-elements.

    References: [#7134](https://www.sqlalchemy.org/trac/ticket/7134)

*   **[orm] [bug] [regression]**

    Fixed regression where the use of a `hybrid_property` attribute or a mapped `composite()` attribute as a key passed to the `Update.values()` method for an ORM-enabled `Update` statement, as well as when using it via the legacy `Query.update()` method, would be processed for incoming ORM/hybrid/composite values within the compilation stage of the UPDATE statement, which meant that in those cases where caching occurred, subsequent invocations of the same statement would no longer receive the correct values. This would include not only hybrids that use the `hybrid_property.update_expression()` method, but any use of a plain hybrid attribute as well. For composites, the issue instead caused a non-repeatable cache key to be generated, which would break caching and could fill up the statement cache with repeated statements.

    The `Update` construct now handles the processing of key/value pairs passed to `Update.values()` and `Update.ordered_values()` up front when the construct is first generated, before the cache key has been generated so that the key/value pairs are processed each time, and so that the cache key is generated against the individual column/value pairs that will ultimately be used in the statement.

    References: [#7209](https://www.sqlalchemy.org/trac/ticket/7209)

*   **[orm]**

    Passing a `Query` object to `Session.execute()` is not the intended use of this object, and will now raise a deprecation warning.

    References: [#6284](https://www.sqlalchemy.org/trac/ticket/6284)

### examples

*   **[examples] [bug]**

    Repaired the examples in examples/versioned_rows to use SQLAlchemy 1.4 APIs correctly; these examples had been missed when API changes like removing “passive” from `Session.is_modified()` were made as well as the `SessionEvents.do_orm_execute()` event hook were added.

    References: [#7169](https://www.sqlalchemy.org/trac/ticket/7169)

### engine

*   **[engine] [bug]**

    Fixed issue where the deprecation warning for the `URL` constructor which indicates that the `URL.create()` method should be used would not emit if a full positional argument list of seven arguments were passed; additionally, validation of URL arguments will now occur if the constructor is called in this way, which was being skipped previously.

    References: [#7130](https://www.sqlalchemy.org/trac/ticket/7130)

*   **[engine] [bug] [postgresql]**

    The `Inspector.reflect_table()` method now supports reflecting tables that do not have user defined columns. This allows `MetaData.reflect()` to properly complete reflection on databases that contain such tables. Currently, only PostgreSQL is known to support such a construct among the common database backends.

    References: [#3247](https://www.sqlalchemy.org/trac/ticket/3247)

*   **[engine] [bug]**

    Implemented proper `__reduce__()` methods for all SQLAlchemy exception objects to ensure they all support clean round trips when pickling, as exception objects are often serialized for the purposes of various debugging tools.

    References: [#7077](https://www.sqlalchemy.org/trac/ticket/7077)

### sql

*   **[sql] [bug]**

    Fixed issue where SQL queries using the `FunctionElement.within_group()` construct could not be pickled, typically when using the `sqlalchemy.ext.serializer` extension but also for general generic pickling.

    References: [#6520](https://www.sqlalchemy.org/trac/ticket/6520)

*   **[sql] [bug]**

    Repaired issue in new `HasCTE.cte.nesting` parameter introduced with [#4123](https://www.sqlalchemy.org/trac/ticket/4123) where a recursive `CTE` using `HasCTE.cte.recursive` in typical conjunction with UNION would not compile correctly. Additionally makes some adjustments so that the `CTE` construct creates a correct cache key. Pull request courtesy Eric Masseran.

    References: [#4123](https://www.sqlalchemy.org/trac/ticket/4123)

*   **[sql] [bug]**

    Account for the `table.schema` parameter passed to the `table()` construct, such that it is taken into account when accessing the `TableClause.fullname` attribute.

    References: [#7061](https://www.sqlalchemy.org/trac/ticket/7061)

*   **[sql] [bug]**

    Fixed an inconsistency in the `ColumnOperators.any_()` / `ColumnOperators.all_()` functions / methods where the special behavior these functions have of “flipping” the expression such that the “ANY” / “ALL” expression is always on the right side would not function if the comparison were against the None value, that is, “column.any_() == None” should produce the same SQL expression as “null() == column.any_()”. Added more docs to clarify this as well, plus mentions that any_() / all_() generally supersede the ARRAY version “any()” / “all()”.

    References: [#7140](https://www.sqlalchemy.org/trac/ticket/7140)

*   **[sql] [bug] [regression]**

    Fixed issue where “expanding IN” would fail to function correctly with datatypes that use the `TypeEngine.bind_expression()` method, where the method would need to be applied to each element of the IN expression rather than the overall IN expression itself.

    References: [#7177](https://www.sqlalchemy.org/trac/ticket/7177)

*   **[sql] [bug]**

    Adjusted the “column disambiguation” logic that’s new in 1.4, where the same expression repeated gets an “extra anonymous” label, so that the logic more aggressively deduplicates those labels when the repeated element is the same Python expression object each time, as occurs in cases like when using “singleton” values like `null()`. This is based on the observation that at least some databases (e.g. MySQL, but not SQLite) will raise an error if the same label is repeated inside of a subquery.

    References: [#7153](https://www.sqlalchemy.org/trac/ticket/7153)

### mypy

*   **[mypy] [bug]**

    Fixed issue in mypy plugin to improve upon some issues detecting `Enum()` SQL types containing custom Python enumeration classes. Pull request courtesy Hiroshi Ogawa.

    References: [#6435](https://www.sqlalchemy.org/trac/ticket/6435)

### postgresql

*   **[postgresql] [bug]**

    Added a “disconnect” condition for the “SSL SYSCALL error: Bad address” error message as reported by psycopg2\. Pull request courtesy Zeke Brechtel.

    References: [#5387](https://www.sqlalchemy.org/trac/ticket/5387)

*   **[postgresql] [bug] [regression]**

    Fixed issue where IN expressions against a series of array elements, as can be done with PostgreSQL, would fail to function correctly due to multiple issues within the “expanding IN” feature of SQLAlchemy Core that was standardized in version 1.4\. The psycopg2 dialect now makes use of the `TypeEngine.bind_expression()` method with `ARRAY` to portably apply the correct casts to elements. The asyncpg dialect was not affected by this issue as it applies bind-level casts at the driver level rather than at the compiler level.

    References: [#7177](https://www.sqlalchemy.org/trac/ticket/7177)

### mysql

*   **[mysql] [bug] [mariadb]**

    Fixes to accommodate for the MariaDB 10.6 series, including backwards incompatible changes in both the mariadb-connector Python driver (supported on SQLAlchemy 1.4 only) as well as the native 10.6 client libraries that are used automatically by the mysqlclient DBAPI (applies to both 1.3 and 1.4). The “utf8mb3” encoding symbol is now reported by these client libraries when the encoding is stated as “utf8”, leading to lookup and encoding errors within the MySQL dialect that does not expect this symbol. Updates to both the MySQL base library to accommodate for this utf8mb3 symbol being reported as well as to the test suite. Thanks to Georg Richter for support.

    This change is also **backported** to: 1.3.25

    References: [#7115](https://www.sqlalchemy.org/trac/ticket/7115), [#7136](https://www.sqlalchemy.org/trac/ticket/7136)

*   **[mysql] [bug]**

    Fixed issue in MySQL `match()` construct where passing a clause expression such as `bindparam()` or other SQL expression for the “against” parameter would fail. Pull request courtesy Anton Kovalevich.

    References: [#7144](https://www.sqlalchemy.org/trac/ticket/7144)

*   **[mysql] [bug]**

    Fixed installation issue where the `sqlalchemy.dialects.mysql` module would not be importable if “greenlet” were not installed.

    References: [#7204](https://www.sqlalchemy.org/trac/ticket/7204)

### mssql

*   **[mssql] [usecase]**

    Added reflection support for SQL Server foreign key options, including “ON UPDATE” and “ON DELETE” values of “CASCADE” and “SET NULL”.

*   **[mssql] [bug]**

    Fixed issue with `Inspector.get_foreign_keys()` where foreign keys were omitted if they were established against a unique index instead of a unique constraint.

    References: [#7160](https://www.sqlalchemy.org/trac/ticket/7160)

*   **[mssql] [bug]**

    Fixed issue with `Inspector.has_table()` where it would return False if a local temp table with the same name from a different session happened to be returned first when querying tempdb. This is a continuation of [#6910](https://www.sqlalchemy.org/trac/ticket/6910) which accounted for the temp table existing only in the alternate session and not the current one.

    References: [#7168](https://www.sqlalchemy.org/trac/ticket/7168)

*   **[mssql] [bug] [regression]**

    Fixed bug in SQL Server `DATETIMEOFFSET` datatype where the ODBC implementation would not generate the correct DDL, for cases where the type were converted using the `dialect.type_descriptor()` method, the usage of which is illustrated in some documented examples for `TypeDecorator`, though not necessary for most datatypes. Regression was introduced by [#6366](https://www.sqlalchemy.org/trac/ticket/6366). As part of this change, the full list of SQL Server date types have been amended to return a “dialect impl” that generates the same DDL name as the supertype.

    References: [#7129](https://www.sqlalchemy.org/trac/ticket/7129)

## 1.4.25

Released: September 22, 2021

### platform

*   **[platform] [bug] [regression]**

    Fixed regression due to [#7024](https://www.sqlalchemy.org/trac/ticket/7024) where the reorganization of the “platform machine” names used by the `greenlet` dependency mis-spelled “aarch64” and additionally omitted uppercase “AMD64” as is needed for Windows machines. Pull request courtesy James Dow.

    References: [#7024](https://www.sqlalchemy.org/trac/ticket/7024)

## 1.4.24

Released: September 22, 2021

### platform

*   **[platform] [bug]**

    Further adjusted the “greenlet” package specifier in setup.cfg to use a long chain of “or” expressions, so that the comparison of `platform_machine` to a specific identifier matches only the complete string.

    References: [#7024](https://www.sqlalchemy.org/trac/ticket/7024)

### orm

*   **[orm] [usecase]**

    Added loader options to `Session.merge()` and `AsyncSession.merge()` via a new `Session.merge.options` parameter, which will apply the given loader options to the `get()` used internally by merge, allowing eager loading of relationships etc. to be applied when the merge process loads a new object. Pull request courtesy Daniel Stone.

    References: [#6955](https://www.sqlalchemy.org/trac/ticket/6955)

*   **[orm] [bug] [regression]**

    Fixed ORM issue where column expressions passed to `query()` or ORM-enabled `select()` would be deduplicated on the identity of the object, such as a phrase like `select(A.id, null(), null())` would produce only one “NULL” expression, which previously was not the case in 1.3\. However, the change also allows for ORM expressions to render as given as well, such as `select(A.data, A.data)` will produce a result row with two columns.

    References: [#6979](https://www.sqlalchemy.org/trac/ticket/6979)

*   **[orm] [bug] [regression]**

    Fixed issue in recently repaired `Query.with_entities()` method where the flag that determines automatic uniquing for legacy ORM `Query` objects only would be set to `True` inappropriately in cases where the `with_entities()` call would be setting the `Query` to return column-only rows, which are not uniqued.

    References: [#6924](https://www.sqlalchemy.org/trac/ticket/6924)

### engine

*   **[engine] [usecase] [asyncio]**

    Improve the interface used by adapted drivers, like the asyncio ones, to access the actual connection object returned by the driver.

    The `_ConnectionFairy` object has two new attributes:

    *   `_ConnectionFairy.dbapi_connection` always represents a DBAPI compatible object. For pep-249 drivers, this is the DBAPI connection as it always has been, previously accessed under the `.connection` attribute. For asyncio drivers that SQLAlchemy adapts into a pep-249 interface, the returned object will normally be a SQLAlchemy adaption object called `AdaptedConnection`.

    *   `_ConnectionFairy.driver_connection` always represents the actual connection object maintained by the third party pep-249 DBAPI or async driver in use. For standard pep-249 DBAPIs, this will always be the same object as that of the `dbapi_connection`. For an asyncio driver, it will be the underlying asyncio-only connection object.

    The `.connection` attribute remains available and is now a legacy alias of `.dbapi_connection`.

    See also

    How do I get at the raw DBAPI connection when using an Engine?

    References: [#6832](https://www.sqlalchemy.org/trac/ticket/6832)

*   **[engine] [usecase] [orm]**

    Added new methods `Session.scalars()`, `Connection.scalars()`, `AsyncSession.scalars()` and `AsyncSession.stream_scalars()`, which provide a short cut to the use case of receiving a row-oriented `Result` object and converting it to a `ScalarResult` object via the `Result.scalars()` method, to return a list of values rather than a list of rows. The new methods are analogous to the long existing `Session.scalar()` and `Connection.scalar()` methods used to return a single value from the first row only. Pull request courtesy Miguel Grinberg.

    References: [#6990](https://www.sqlalchemy.org/trac/ticket/6990)

*   **[engine] [bug] [regression]**

    Fixed issue where the ability of the `ConnectionEvents.before_execute()` method to alter the SQL statement object passed, returning the new object to be invoked, was inadvertently removed. This behavior has been restored.

    References: [#6913](https://www.sqlalchemy.org/trac/ticket/6913)

*   **[engine] [bug]**

    Ensure that `str()` is called on the an `URL.create.password` argument, allowing usage of objects that implement the `__str__()` method as password attributes. Also clarified that one such object is not appropriate to dynamically change the password for each database connection; the approaches at Generating dynamic authentication tokens should be used instead.

    References: [#6958](https://www.sqlalchemy.org/trac/ticket/6958)

*   **[engine] [bug]**

    Fixed issue in `URL` where validation of “drivername” would not appropriately respond to the `None` value where a string were expected.

    References: [#6983](https://www.sqlalchemy.org/trac/ticket/6983)

*   **[engine] [bug] [postgresql]**

    Fixed issue where an engine that had `create_engine.implicit_returning` set to False would fail to function when PostgreSQL’s “fast insertmany” feature were used in conjunction with a `Sequence`, as well as if any kind of “executemany” with “return_defaults()” were used in conjunction with a `Sequence`. Note that PostgreSQL “fast insertmany” uses “RETURNING” by definition, when the SQL statement is passed to the driver; overall, the `create_engine.implicit_returning` flag is legacy and has no real use in modern SQLAlchemy, and will be deprecated in a separate change.

    References: [#6963](https://www.sqlalchemy.org/trac/ticket/6963)

### sql

*   **[sql] [usecase]**

    Added new parameter `HasCTE.cte.nesting` to the `CTE` constructor and `HasCTE.cte()` method, which flags the CTE as one which should remain nested within an enclosing CTE, rather than being moved to the top level of the outermost SELECT. While in the vast majority of cases there is no difference in SQL functionality, users have identified various edge-cases where true nesting of CTE constructs is desirable. Much thanks to Eric Masseran for lots of work on this intricate feature.

    References: [#4123](https://www.sqlalchemy.org/trac/ticket/4123)

*   **[sql] [bug]**

    Implemented missing methods in `FunctionElement` which, while unused, would lead pylint to report them as unimplemented abstract methods.

    References: [#7052](https://www.sqlalchemy.org/trac/ticket/7052)

*   **[sql] [bug]**

    Fixed a two issues where combinations of `select()` and `join()` when adapted to form a copy of the element would not completely copy the state of all column objects associated with subqueries. A key problem this caused is that usage of the `ClauseElement.params()` method (which should probably be moved into a legacy category as it is inefficient and error prone) would leave copies of the old `BindParameter` objects around, leading to issues in correctly setting the parameters at execution time.

    References: [#7055](https://www.sqlalchemy.org/trac/ticket/7055)

*   **[sql] [bug]**

    Fixed issue related to new `HasCTE.add_cte()` feature where pairing two “INSERT..FROM SELECT” statements simultaneously would lose track of the two independent SELECT statements, leading to the wrong SQL.

    References: [#7036](https://www.sqlalchemy.org/trac/ticket/7036)

*   **[sql] [bug]**

    Fixed issue where using ORM column expressions as keys in the list of dictionaries passed to `Insert.values()` for “multi-valued insert” would not be processed correctly into the correct column expressions.

    References: [#7060](https://www.sqlalchemy.org/trac/ticket/7060)

### mypy

*   **[mypy] [bug]**

    Fixed issue where mypy plugin would crash when interpreting a `query_expression()` construct.

    References: [#6950](https://www.sqlalchemy.org/trac/ticket/6950)

*   **[mypy] [bug]**

    Fixed issue in mypy plugin where columns on a mixin would not be correctly interpreted if the mapped class relied upon a `__tablename__` routine that came from a superclass.

    References: [#6937](https://www.sqlalchemy.org/trac/ticket/6937)

### asyncio

*   **[asyncio] [feature] [mysql]**

    Added initial support for the `asyncmy` asyncio database driver for MySQL and MariaDB. This driver is very new, however appears to be the only current alternative to the `aiomysql` driver which currently appears to be unmaintained and is not working with current Python versions. Much thanks to long2ice for the pull request for this dialect.

    See also

    asyncmy

    References: [#6993](https://www.sqlalchemy.org/trac/ticket/6993)

*   **[asyncio] [usecase]**

    The `AsyncSession` now supports overriding which `Session` it uses as the proxied instance. A custom `Session` class can be passed using the `AsyncSession.sync_session_class` parameter or by subclassing the `AsyncSession` and specifying a custom `AsyncSession.sync_session_class`.

    References: [#6746](https://www.sqlalchemy.org/trac/ticket/6746)

*   **[asyncio] [bug]**

    Fixed a bug in `AsyncSession.execute()` and `AsyncSession.stream()` that required `execution_options` to be an instance of `immutabledict` when defined. It now correctly accepts any mapping.

    References: [#6943](https://www.sqlalchemy.org/trac/ticket/6943)

*   **[asyncio] [bug]**

    Added missing `**kw` arguments to the `AsyncSession.connection()` method.

*   **[asyncio] [bug]**

    Deprecate usage of `scoped_session` with asyncio drivers. When using Asyncio the `async_scoped_session` should be used instead.

    References: [#6746](https://www.sqlalchemy.org/trac/ticket/6746)

### postgresql

*   **[postgresql] [bug]**

    Qualify `version()` call to avoid shadowing issues if a different search path is configured by the user.

    References: [#6912](https://www.sqlalchemy.org/trac/ticket/6912)

*   **[postgresql] [bug]**

    The `ENUM` datatype is PostgreSQL-native and therefore should not be used with the `native_enum=False` flag. This flag is now ignored if passed to the `ENUM` datatype and a warning is emitted; previously the flag would cause the type object to fail to function correctly.

    References: [#6106](https://www.sqlalchemy.org/trac/ticket/6106)

### sqlite

*   **[sqlite] [bug]**

    Fixed bug where the error message for SQLite invalid isolation level on the pysqlite driver would fail to indicate that “AUTOCOMMIT” is one of the valid isolation levels.

### mssql

*   **[mssql] [bug] [reflection]**

    Fixed an issue where `sqlalchemy.engine.reflection.has_table()` returned `True` for local temporary tables that actually belonged to a different SQL Server session (connection). An extra check is now performed to ensure that the temp table detected is in fact owned by the current session.

    References: [#6910](https://www.sqlalchemy.org/trac/ticket/6910)

### oracle

*   **[oracle] [performance] [bug]**

    Added a CAST(VARCHAR2(128)) to the “table name”, “owner”, and other DDL-name parameters as used in reflection queries against Oracle system views such as ALL_TABLES, ALL_TAB_CONSTRAINTS, etc to better enable indexing to take place against these columns, as they previously would be implicitly handled as NVARCHAR2 due to Python’s use of Unicode for strings; these columns are documented in all Oracle versions as being VARCHAR2 with lengths varying from 30 to 128 characters depending on server version. Additionally, test support has been enabled for Unicode-named DDL structures against Oracle databases.

    References: [#4486](https://www.sqlalchemy.org/trac/ticket/4486)

## 1.4.23

Released: August 18, 2021

### general

*   **[general] [bug]**

    The setup requirements have been modified such `greenlet` is a default requirement only for those platforms that are well known for `greenlet` to be installable and for which there is already a pre-built binary on pypi; the current list is `x86_64 aarch64 ppc64le amd64 win32`. For other platforms, greenlet will not install by default, which should enable installation and test suite running of SQLAlchemy 1.4 on platforms that don’t support `greenlet`, excluding any asyncio features. In order to install with the `greenlet` dependency included on a machine architecture outside of the above list, the `[asyncio]` extra may be included by running `pip install sqlalchemy[asyncio]` which will then attempt to install `greenlet`.

    Additionally, the test suite has been repaired so that tests can complete fully when greenlet is not installed, with appropriate skips for asyncio-related tests.

    References: [#6136](https://www.sqlalchemy.org/trac/ticket/6136)

### orm

*   **[orm] [usecase]**

    Added new attribute `Select.columns_clause_froms` that will retrieve the FROM list implied by the columns clause of the `Select` statement. This differs from the old `Select.froms` collection in that it does not perform any ORM compilation steps, which necessarily deannotate the FROM elements and do things like compute joinedloads etc., which makes it not an appropriate candidate for the `Select.select_from()` method. Additionally adds a new parameter `Select.with_only_columns.maintain_column_froms` that transfers this collection to `Select.select_from()` before replacing the columns collection.

    In addition, the `Select.froms` is renamed to `Select.get_final_froms()`, to stress that this collection is not a simple accessor and is instead calculated given the full state of the object, which can be an expensive call when used in an ORM context.

    Additionally fixes a regression involving the `with_only_columns()` function to support applying criteria to column elements that were replaced with either `Select.with_only_columns()` or `Query.with_entities()` , which had broken as part of [#6503](https://www.sqlalchemy.org/trac/ticket/6503) released in 1.4.19.

    References: [#6808](https://www.sqlalchemy.org/trac/ticket/6808)

*   **[orm] [bug] [sql]**

    Fixed issue where a bound parameter object that was “cloned” would cause a name conflict in the compiler, if more than one clone of this parameter were used at the same time in a single statement. This could occur in particular with things like ORM single table inheritance queries that indicated the same “discriminator” value multiple times in one query.

    References: [#6824](https://www.sqlalchemy.org/trac/ticket/6824)

*   **[orm] [bug]**

    Fixed issue in loader strategies where the use of the `Load.options()` method, particularly when nesting multiple calls, would generate an overly long and more importantly non-deterministic cache key, leading to very large cache keys which were also not allowing efficient cache usage, both in terms of total memory used as well as number of entries used in the cache itself.

    References: [#6869](https://www.sqlalchemy.org/trac/ticket/6869)

*   **[orm] [bug]**

    Revised the means by which the `ORMExecuteState.user_defined_options` accessor receives `UserDefinedOption` and related option objects from the context, with particular emphasis on the “selectinload” on the loader strategy where this previously was not working; other strategies did not have this problem. The objects that are associated with the current query being executed, and not that of a query being cached, are now propagated unconditionally. This essentially separates them out from the “loader strategy” options which are explicitly associated with the compiled state of a query and need to be used in relation to the cached query.

    The effect of this fix is that a user-defined option, such as those used by the dogpile.caching example as well as for other recipes such as defining a “shard id” for the horizontal sharing extension, will be correctly propagated to eager and lazy loaders regardless of whether a cached query was ultimately invoked.

    References: [#6887](https://www.sqlalchemy.org/trac/ticket/6887)

*   **[orm] [bug]**

    Fixed issue where the unit of work would internally use a 2.0-deprecated SQL expression form, emitting a deprecation warning when SQLALCHEMY_WARN_20 were enabled.

    References: [#6812](https://www.sqlalchemy.org/trac/ticket/6812)

*   **[orm] [bug]**

    Fixed issue in `selectinload()` where use of the new `PropComparator.and_()` feature within options that were nested more than one level deep would fail to update bound parameter values that were in the nested criteria, as a side effect of SQL statement caching.

    References: [#6881](https://www.sqlalchemy.org/trac/ticket/6881)

*   **[orm] [bug]**

    Adjusted ORM loader internals to no longer use the “lambda caching” system that was added in 1.4, as well as repaired one location that was still using the previous “baked query” system for a query. The lambda caching system remains an effective way to reduce the overhead of building up queries that have relatively fixed usage patterns. In the case of loader strategies, the queries used are responsible for moving through lots of arbitrary options and criteria, which is both generated and sometimes consumed by end-user code, that make the lambda cache concept not any more efficient than not using it, at the cost of more complexity. In particular the problems noted by [#6881](https://www.sqlalchemy.org/trac/ticket/6881) and [#6887](https://www.sqlalchemy.org/trac/ticket/6887) are made are made considerably less complicated by removing this feature internally.

    References: [#6079](https://www.sqlalchemy.org/trac/ticket/6079), [#6889](https://www.sqlalchemy.org/trac/ticket/6889)

*   **[orm] [bug]**

    Fixed an issue where the `Bundle` construct would not create proper cache keys, leading to inefficient use of the query cache. This had some impact on the “selectinload” strategy and was identified as part of [#6889](https://www.sqlalchemy.org/trac/ticket/6889).

    References: [#6889](https://www.sqlalchemy.org/trac/ticket/6889)

### sql

*   **[sql] [bug]**

    Fix issue in `CTE` where new `HasCTE.add_cte()` method added in version 1.4.21 / [#6752](https://www.sqlalchemy.org/trac/ticket/6752) failed to function correctly for “compound select” structures such as `union()`, `union_all()`, `except()`, etc. Pull request courtesy Eric Masseran.

    References: [#6752](https://www.sqlalchemy.org/trac/ticket/6752)

*   **[sql] [bug]**

    Fixed an issue in the `CacheKey.to_offline_string()` method used by the dogpile.caching example where attempting to create a proper cache key from the special “lambda” query generated by the lazy loader would fail to include the parameter values, leading to an incorrect cache key.

    References: [#6858](https://www.sqlalchemy.org/trac/ticket/6858)

*   **[sql] [bug]**

    Adjusted the “from linter” warning feature to accommodate for a chain of joins more than one level deep where the ON clauses don’t explicitly match up the targets, such as an expression such as “ON TRUE”. This mode of use is intended to cancel the cartesian product warning simply by the fact that there’s a JOIN from “a to b”, which was not working for the case where the chain of joins had more than one element.

    References: [#6886](https://www.sqlalchemy.org/trac/ticket/6886)

*   **[sql] [bug]**

    Fixed issue in lambda caching system where an element of a query that produces no cache key, like a custom option or clause element, would still populate the expression in the “lambda cache” inappropriately.

### schema

*   **[schema] [enum]**

    Unify behaviour `Enum` in native and non-native implementations regarding the accepted values for an enum with aliased elements. When `Enum.omit_aliases` is `False` all values, alias included, are accepted as valid values. When `Enum.omit_aliases` is `True` only non aliased values are accepted as valid values.

    References: [#6146](https://www.sqlalchemy.org/trac/ticket/6146)

### mypy

*   **[mypy] [usecase]**

    Added support for SQLAlchemy classes to be defined in user code using “generic class” syntax as defined by `sqlalchemy2-stubs`, e.g. `Column[String]`, without the need for qualifying these constructs within a `TYPE_CHECKING` block by implementing the Python special method `__class_getitem__()`, which allows this syntax to pass without error at runtime.

    References: [#6759](https://www.sqlalchemy.org/trac/ticket/6759), [#6804](https://www.sqlalchemy.org/trac/ticket/6804)

### postgresql

*   **[postgresql] [bug]**

    Added the “is_comparison” flag to the PostgreSQL “overlaps”, “contained_by”, “contains” operators, so that they work in relevant ORM contexts as well as in conjunction with the “from linter” feature.

    References: [#6886](https://www.sqlalchemy.org/trac/ticket/6886)

### mssql

*   **[mssql] [bug] [sql]**

    Fixed issue where the `literal_binds` compiler flag, as used externally to render bound parameters inline, would fail to work when used with a certain class of parameters known as “literal_execute”, which covers things like LIMIT and OFFSET values for dialects where the drivers don’t allow a bound parameter, such as SQL Server’s “TOP” clause. The issue locally seemed to affect only the MSSQL dialect.

    References: [#6863](https://www.sqlalchemy.org/trac/ticket/6863)

### misc

*   **[bug] [ext]**

    Fixed issue where the horizontal sharding extension would not correctly accommodate for a plain textual SQL statement passed to `Session.execute()`.

    References: [#6816](https://www.sqlalchemy.org/trac/ticket/6816)

## 1.4.22

Released: July 21, 2021

### orm

*   **[orm] [bug]**

    Fixed issue in new `Table.table_valued()` method where the resulting `TableValuedColumn` construct would not respond correctly to alias adaptation as is used throughout the ORM, such as for eager loading, polymorphic loading, etc.

    References: [#6775](https://www.sqlalchemy.org/trac/ticket/6775)

*   **[orm] [bug]**

    Fixed issue where usage of the `Result.unique()` method with an ORM result that included column expressions with unhashable types, such as `JSON` or `ARRAY` using non-tuples would silently fall back to using the `id()` function, rather than raising an error. This now raises an error when the `Result.unique()` method is used in a 2.0 style ORM query. Additionally, hashability is assumed to be True for result values of unknown type, such as often happens when using SQL functions of unknown return type; if values are truly not hashable then the `hash()` itself will raise.

    For legacy ORM queries, since the legacy `Query` object uniquifies in all cases, the old rules remain in place, which is to use `id()` for result values of unknown type as this legacy uniquing is mostly for the purpose of uniquing ORM entities and not column values.

    References: [#6769](https://www.sqlalchemy.org/trac/ticket/6769)

*   **[orm] [bug]**

    Fixed an issue where clearing of mappers during things like test suite teardowns could cause a “dictionary changed size” warning during garbage collection, due to iteration of a weak-referencing dictionary. A `list()` has been applied to prevent concurrent GC from affecting this operation.

    References: [#6771](https://www.sqlalchemy.org/trac/ticket/6771)

*   **[orm] [bug] [regression]**

    Fixed critical caching issue where the ORM’s persistence feature using INSERT..RETURNING would cache an incorrect query when mixing the “bulk save” and standard “flush” forms of INSERT.

    References: [#6793](https://www.sqlalchemy.org/trac/ticket/6793)

### engine

*   **[engine] [bug]**

    Added some guards against `KeyError` in the event system to accommodate the case that the interpreter is shutting down at the same time `Engine.dispose()` is being called, which would cause stack trace warnings.

    References: [#6740](https://www.sqlalchemy.org/trac/ticket/6740)

### sql

*   **[sql] [bug]**

    Fixed issue where use of the `case.whens` parameter passing a dictionary positionally and not as a keyword argument would emit a 2.0 deprecation warning, referring to the deprecation of passing a list positionally. The dictionary format of “whens”, passed positionally, is still supported and was accidentally marked as deprecated.

    References: [#6786](https://www.sqlalchemy.org/trac/ticket/6786)

*   **[sql] [bug]**

    Fixed issue where type-specific bound parameter handlers would not be called upon in the case of using the `Insert.values()` method with the Python `None` value; in particular, this would be noticed when using the `JSON` datatype as well as related PostgreSQL specific types such as `JSONB` which would fail to encode the Python `None` value into JSON null, however the issue was generalized to any bound parameter handler in conjunction with this specific method of `Insert`.

    References: [#6770](https://www.sqlalchemy.org/trac/ticket/6770)

## 1.4.21

Released: July 14, 2021

### orm

*   **[orm] [usecase]**

    Modified the approach used for history tracking of scalar object relationships that are not many-to-one, i.e. one-to-one relationships that would otherwise be one-to-many. When replacing a one-to-one value, the “old” value that would be replaced is no longer loaded immediately, and is instead handled during the flush process. This eliminates an historically troublesome lazy load that otherwise often occurs when assigning to a one-to-one attribute, and is particularly troublesome when using “lazy=’raise’” as well as asyncio use cases.

    This change does cause a behavioral change within the `AttributeEvents.set()` event, which is nonetheless currently documented, which is that the event applied to such a one-to-one attribute will no longer receive the “old” parameter if it is unloaded and the `relationship.active_history` flag is not set. As is documented in `AttributeEvents.set()`, if the event handler needs to receive the “old” value when the event fires off, the active_history flag must be established either with the event listener or with the relationship. This is already the behavior with other kinds of attributes such as many-to-one and column value references.

    The change additionally will defer updating a backref on the “old” value in the less common case that the “old” value is locally present in the session, but isn’t loaded on the relationship in question, until the next flush occurs. If this causes an issue, again the normal `relationship.active_history` flag can be set to `True` on the relationship.

    References: [#6708](https://www.sqlalchemy.org/trac/ticket/6708)

*   **[orm] [bug] [regression]**

    Fixed regression caused in 1.4.19 due to [#6503](https://www.sqlalchemy.org/trac/ticket/6503) and related involving `Query.with_entities()` where the new structure used would be inappropriately transferred to an enclosing `Query` when making use of set operations such as `Query.union()`, causing the JOIN instructions within to be applied to the outside query as well.

    References: [#6698](https://www.sqlalchemy.org/trac/ticket/6698)

*   **[orm] [bug] [regression]**

    Fixed regression which appeared in version 1.4.3 due to [#6060](https://www.sqlalchemy.org/trac/ticket/6060) where rules that limit ORM adaptation of derived selectables interfered with other ORM-adaptation based cases, in this case when applying adaptations for a `with_polymorphic()` against a mapping which uses a `column_property()` which in turn makes use of a scalar select that includes a `aliased()` object of the mapped table.

    References: [#6762](https://www.sqlalchemy.org/trac/ticket/6762)

*   **[orm] [regression]**

    Fixed ORM regression where ad-hoc label names generated for hybrid properties and potentially other similar types of ORM-enabled expressions would usually be propagated outwards through subqueries, allowing the name to be retained in the final keys of the result set even when selecting from subqueries. Additional state is now tracked in this case that isn’t lost when a hybrid is selected out of a Core select / subquery.

    References: [#6718](https://www.sqlalchemy.org/trac/ticket/6718)

### sql

*   **[sql] [usecase]**

    Added new method `HasCTE.add_cte()` to each of the `select()`, `insert()`, `update()` and `delete()` constructs. This method will add the given `CTE` as an “independent” CTE of the statement, meaning it renders in the WITH clause above the statement unconditionally even if it is not otherwise referenced in the primary statement. This is a popular use case on the PostgreSQL database where a CTE is used for a DML statement that runs against database rows independently of the primary statement.

    References: [#6752](https://www.sqlalchemy.org/trac/ticket/6752)

*   **[sql] [bug]**

    Fixed issue in CTE constructs where a recursive CTE that referred to a SELECT that has duplicate column names, which are typically deduplicated using labeling logic in 1.4, would fail to refer to the deduplicated label name correctly within the WITH clause.

    References: [#6710](https://www.sqlalchemy.org/trac/ticket/6710)

*   **[sql] [bug] [regression]**

    Fixed regression where the `tablesample()` construct would fail to be executable when constructed given a floating-point sampling value not embedded within a SQL function.

    References: [#6735](https://www.sqlalchemy.org/trac/ticket/6735)

### postgresql

*   **[postgresql] [bug]**

    Fixed issue in `Insert.on_conflict_do_nothing()` and `Insert.on_conflict_do_update()` where the name of a unique constraint passed as the `constraint` parameter would not be properly truncated for length if it were based on a naming convention that generated a too-long name for the PostgreSQL max identifier length of 63 characters, in the same way which occurs within a CREATE TABLE statement.

    References: [#6755](https://www.sqlalchemy.org/trac/ticket/6755)

*   **[postgresql] [bug]**

    Fixed issue where the PostgreSQL `ENUM` datatype as embedded in the `ARRAY` datatype would fail to emit correctly in create/drop when the `schema_translate_map` feature were also in use. Additionally repairs a related issue where the same `schema_translate_map` feature would not work for the `ENUM` datatype in combination with a `CAST`, that’s also intrinsic to how the `ARRAY(ENUM)` combination works on the PostgreSQL dialect.

    References: [#6739](https://www.sqlalchemy.org/trac/ticket/6739)

*   **[postgresql] [bug]**

    Fixed issue in `Insert.on_conflict_do_nothing()` and `Insert.on_conflict_do_update()` where the name of a unique constraint passed as the `constraint` parameter would not be properly quoted if it contained characters which required quoting.

    References: [#6696](https://www.sqlalchemy.org/trac/ticket/6696)

### mssql

*   **[mssql] [bug] [regression]**

    Fixed regression where the special dotted-schema name handling for the SQL Server dialect would not function correctly if the dotted schema name were used within the `schema_translate_map` feature.

    References: [#6697](https://www.sqlalchemy.org/trac/ticket/6697)

## 1.4.20

Released: June 28, 2021

### orm

*   **[orm] [bug] [regression]**

    Fixed regression in ORM regarding an internal reconstitution step for the `with_polymorphic()` construct, when the user-facing object is garbage collected as the query is processed. The reconstitution was not ensuring the sub-entities for the “polymorphic” case were handled, leading to an `AttributeError`.

    References: [#6680](https://www.sqlalchemy.org/trac/ticket/6680)

*   **[orm] [bug] [regression]**

    Adjusted `Query.union()` and similar set operations to be correctly compatible with the new capabilities just added in [#6661](https://www.sqlalchemy.org/trac/ticket/6661), with SQLAlchemy 1.4.19, such that the SELECT statements rendered as elements of the UNION or other set operation will include directly mapped columns that are mapped as deferred; this both fixes a regression involving unions with multiple levels of nesting that would produce a column mismatch, and also allows the `undefer()` option to be used at the top level of such a `Query` without having to apply the option to each of the elements within the UNION.

    References: [#6678](https://www.sqlalchemy.org/trac/ticket/6678)

*   **[orm] [bug]**

    Adjusted the check in the mapper for a callable object that is used as a `@validates` validator function or a `@reconstructor` reconstruction function, to check for “callable” more liberally such as to accommodate objects based on fundamental attributes like `__func__` and `__call__`, rather than testing for `MethodType` / `FunctionType`, allowing things like cython functions to work properly. Pull request courtesy Miłosz Stypiński.

    References: [#6538](https://www.sqlalchemy.org/trac/ticket/6538)

### engine

*   **[engine] [bug]**

    Fixed an issue in the C extension for the `Row` class which could lead to a memory leak in the unlikely case of a `Row` object which referred to an ORM object that then was mutated to refer back to the `Row` itself, creating a cycle. The Python C APIs for tracking GC cycles has been added to the native `Row` implementation to accommodate for this case.

    References: [#5348](https://www.sqlalchemy.org/trac/ticket/5348)

*   **[engine] [bug]**

    Fixed old issue where a `select()` made against the token “*”, which then yielded exactly one column, would fail to correctly organize the `cursor.description` column name into the keys of the result object.

    References: [#6665](https://www.sqlalchemy.org/trac/ticket/6665)

### sql

*   **[sql] [usecase]**

    Add a impl parameter to `PickleType` constructor, allowing any arbitrary type to be used in place of the default implementation of `LargeBinary`. Pull request courtesy jason3gb.

    References: [#6646](https://www.sqlalchemy.org/trac/ticket/6646)

*   **[sql] [bug] [orm]**

    Fixed the class hierarchy for the `Sequence` and the more general `DefaultGenerator` base, as these are “executable” as statements they need to include `Executable` in their hierarchy, not just `StatementRole` as was applied arbitrarily to `Sequence` previously. The fix allows `Sequence` to work in all `.execute()` methods including with `Session.execute()` which was not working in the case that a `SessionEvents.do_orm_execute()` handler was also established.

    References: [#6668](https://www.sqlalchemy.org/trac/ticket/6668)

### schema

*   **[schema] [bug]**

    Fixed issue where passing `None` for the value of `Table.prefixes` would not store an empty list, but rather the constant `None`, which may be unexpected by third party dialects. The issue is revealed by a usage in recent versions of Alembic that are passing `None` for this value. Pull request courtesy Kai Mueller.

    References: [#6685](https://www.sqlalchemy.org/trac/ticket/6685)

### mysql

*   **[mysql] [usecase]**

    Made a small adjustment in the table reflection feature of the MySQL dialect to accommodate for alternate MySQL-oriented databases such as TiDB which include their own “comment” directives at the end of a constraint directive within “CREATE TABLE” where the format doesn’t have the additional space character after the comment, in this case the TiDB “clustered index” feature. Pull request courtesy Daniël van Eeden.

    References: [#6659](https://www.sqlalchemy.org/trac/ticket/6659)

### misc

*   **[bug] [ext] [regression]**

    Fixed regression in `sqlalchemy.ext.automap` extension such that the use case of creating an explicit mapped class to a table that is also the `relationship.secondary` element of a `relationship()` that automap will be generating would emit the “overlaps” warnings introduced in 1.4 and discussed at relationship X will copy column Q to column P, which conflicts with relationship(s): ‘Y’. While generating this case from automap is still subject to the same caveats mentioned in the ‘overlaps’ warning, since automap is primarily intended for more ad-hoc use cases, the condition triggering the warning is disabled when a many-to-many relationship with this specific pattern is generated.

    References: [#6679](https://www.sqlalchemy.org/trac/ticket/6679)

## 1.4.19

Released: June 22, 2021

### orm

*   **[orm] [bug] [regression]**

    Fixed further regressions in the same area as that of [#6052](https://www.sqlalchemy.org/trac/ticket/6052) where loader options as well as invocations of methods like `Query.join()` would fail if the left side of the statement for which the option/join depends upon were replaced by using the `Query.with_entities()` method, or when using 2.0 style queries when using the `Select.with_only_columns()` method. A new set of state has been added to the objects which tracks the “left” entities that the options / join were made against which is memoized when the lead entities are changed.

    References: [#6253](https://www.sqlalchemy.org/trac/ticket/6253), [#6503](https://www.sqlalchemy.org/trac/ticket/6503)

*   **[orm] [bug]**

    Refined the behavior of ORM subquery rendering with regards to deferred columns and column properties to be more compatible with that of 1.3 while also providing for 1.4’s newer features. As a subquery in 1.4 does not make use of loader options, including `undefer()`, a subquery that is against an ORM entity with deferred attributes will now render those deferred attributes that refer directly to mapped table columns, as these are needed in the outer SELECT if that outer SELECT makes use of these columns; however a deferred attribute that refers to a composed SQL expression as we normally do with `column_property()` will not be part of the subquery, as these can be selected explicitly if needed in the subquery. If the entity is being SELECTed from this subquery, the column expression can still render on “the outside” in terms of the derived subquery columns. This produces essentially the same behavior as when working with 1.3\. However in this case the fix has to also make sure that the `.selected_columns` collection of an ORM-enabled `select()` also follows these rules, which in particular allows recursive CTEs to render correctly in this scenario, which were previously failing to render correctly due to this issue.

    References: [#6661](https://www.sqlalchemy.org/trac/ticket/6661)

### sql

*   **[sql] [bug]**

    Fixed issue in CTE constructs mostly relevant to ORM use cases where a recursive CTE against “anonymous” labels such as those seen in ORM `column_property()` mappings would render in the `WITH RECURSIVE xyz(...)` section as their raw internal label and not a cleanly anonymized name.

    References: [#6663](https://www.sqlalchemy.org/trac/ticket/6663)

### mypy

*   **[mypy] [bug]**

    Fixed issue in mypy plugin where class info for a custom declarative base would not be handled correctly on a cached mypy pass, leading to an AssertionError being raised.

    References: [#6476](https://www.sqlalchemy.org/trac/ticket/6476)

### asyncio

*   **[asyncio] [usecase]**

    Implemented `async_scoped_session` to address some asyncio-related incompatibilities between `scoped_session` and `AsyncSession`, in which some methods (notably the `async_scoped_session.remove()` method) should be used with the `await` keyword.

    See also

    Using asyncio scoped session

    References: [#6583](https://www.sqlalchemy.org/trac/ticket/6583)

*   **[asyncio] [bug] [postgresql]**

    Fixed bug in asyncio implementation where the greenlet adaptation system failed to propagate `BaseException` subclasses, most notably including `asyncio.CancelledError`, to the exception handling logic used by the engine to invalidate and clean up the connection, thus preventing connections from being correctly disposed when a task was cancelled.

    References: [#6652](https://www.sqlalchemy.org/trac/ticket/6652)

### postgresql

*   **[postgresql] [bug] [oracle]**

    Fixed issue where the `INTERVAL` datatype on PostgreSQL and Oracle would produce an `AttributeError` when used in the context of a comparison operation against a `timedelta()` object. Pull request courtesy MajorDallas.

    References: [#6649](https://www.sqlalchemy.org/trac/ticket/6649)

*   **[postgresql] [bug]**

    Fixed issue where the pool “pre ping” feature would implicitly start a transaction, which would then interfere with custom transactional flags such as PostgreSQL’s “read only” mode when used with the psycopg2 driver.

    References: [#6621](https://www.sqlalchemy.org/trac/ticket/6621)

### mysql

*   **[mysql] [usecase]**

    Added new construct `match`, which provides for the full range of MySQL’s MATCH operator including multiple column support and modifiers. Pull request courtesy Anton Kovalevich.

    See also

    `match`

    References: [#6132](https://www.sqlalchemy.org/trac/ticket/6132)

### mssql

*   **[mssql] [change]**

    Made improvements to the server version regexp used by the pymssql dialect to prevent a regexp overflow in case of an invalid version string.

    References: [#6253](https://www.sqlalchemy.org/trac/ticket/6253), [#6503](https://www.sqlalchemy.org/trac/ticket/6503)

*   **[mssql] [bug]**

    Fixed bug where the “schema_translate_map” feature would fail to function correctly in conjunction with an INSERT into a table that has an IDENTITY column, where the value of the IDENTITY column were specified in the values of the INSERT thus triggering SQLAlchemy’s feature of setting IDENTITY INSERT to “on”; it’s in this directive where the schema translate map would fail to be honored.

    References: [#6658](https://www.sqlalchemy.org/trac/ticket/6658)

## 1.4.18

Released: June 10, 2021

### orm

*   **[orm] [performance] [bug] [regression]**

    Fixed regression involving how the ORM would resolve a given mapped column to a result row, where under cases such as joined eager loading, a slightly more expensive “fallback” could take place to set up this resolution due to some logic that was removed since 1.3\. The issue could also cause deprecation warnings involving column resolution to be emitted when using a 1.4 style query with joined eager loading.

    References: [#6596](https://www.sqlalchemy.org/trac/ticket/6596)

*   **[orm] [bug]**

    Clarified the current purpose of the `relationship.bake_queries` flag, which in 1.4 is to enable or disable “lambda caching” of statements within the “lazyload” and “selectinload” loader strategies; this is separate from the more foundational SQL query cache that is used for most statements. Additionally, the lazy loader no longer uses its own cache for many-to-one SQL queries, which was an implementation quirk that doesn’t exist for any other loader scenario. Finally, the “lru cache” warning that the lazyloader and selectinloader strategies could emit when handling a wide array of class/relationship combinations has been removed; based on analysis of some end-user cases, this warning doesn’t suggest any significant issue. While setting `bake_queries=False` for such a relationship will remove this cache from being used, there’s no particular performance gain in this case as using no caching vs. using a cache that needs to refresh often likely still wins out on the caching being used side.

    References: [#6072](https://www.sqlalchemy.org/trac/ticket/6072), [#6487](https://www.sqlalchemy.org/trac/ticket/6487)

*   **[orm] [bug] [regression]**

    Adjusted the means by which classes such as `scoped_session` and `AsyncSession` are generated from the base `Session` class, such that custom `Session` subclasses such as that used by Flask-SQLAlchemy don’t need to implement positional arguments when they call into the superclass method, and can continue using the same argument styles as in previous releases.

    References: [#6285](https://www.sqlalchemy.org/trac/ticket/6285)

*   **[orm] [bug] [regression]**

    Fixed issue where query production for joinedload against a complex left hand side involving joined-table inheritance could fail to produce a correct query, due to a clause adaption issue.

    References: [#6595](https://www.sqlalchemy.org/trac/ticket/6595)

*   **[orm] [bug]**

    Fixed issue in experimental “select ORM objects from INSERT/UPDATE” use case where an error was raised if the statement were against a single-table-inheritance subclass.

    References: [#6591](https://www.sqlalchemy.org/trac/ticket/6591)

*   **[orm] [bug]**

    The warning that’s emitted for `relationship()` when multiple relationships would overlap with each other as far as foreign key attributes written towards, now includes the specific “overlaps” argument to use for each warning in order to silence the warning without changing the mapping.

    References: [#6400](https://www.sqlalchemy.org/trac/ticket/6400)

### asyncio

*   **[asyncio] [usecase]**

    Implemented a new registry architecture that allows the `Async` version of an object, like `AsyncSession`, `AsyncConnection`, etc., to be locatable given the proxied “sync” object, i.e. `Session`, `Connection`. Previously, to the degree such lookup functions were used, an `Async` object would be re-created each time, which was less than ideal as the identity and state of the “async” object would not be preserved across calls.

    From there, new helper functions `async_object_session()`, `async_session()` as well as a new `InstanceState` attribute `InstanceState.async_session` have been added, which are used to retrieve the original `AsyncSession` associated with an ORM mapped object, a `Session` associated with an `AsyncSession`, and an `AsyncSession` associated with an `InstanceState`, respectively.

    This patch also implements new methods `AsyncSession.in_nested_transaction()`, `AsyncSession.get_transaction()`, `AsyncSession.get_nested_transaction()`.

    References: [#6319](https://www.sqlalchemy.org/trac/ticket/6319)

*   **[asyncio] [bug]**

    Fixed an issue that presented itself when using the `NullPool` or the `StaticPool` with an async engine. This mostly affected the aiosqlite dialect.

    References: [#6575](https://www.sqlalchemy.org/trac/ticket/6575)

*   **[asyncio] [bug]**

    Added `asyncio.exceptions.TimeoutError`, `asyncio.exceptions.CancelledError` as so-called “exit exceptions”, a class of exceptions that include things like `GreenletExit` and `KeyboardInterrupt`, which are considered to be events that warrant considering a DBAPI connection to be in an unusable state where it should be recycled.

    References: [#6592](https://www.sqlalchemy.org/trac/ticket/6592)

### postgresql

*   **[postgresql] [bug] [regression]**

    Fixed regression where using the PostgreSQL “INSERT..ON CONFLICT” structure would fail to work with the psycopg2 driver if it were used in an “executemany” context along with bound parameters in the “SET” clause, due to the implicit use of the psycopg2 fast execution helpers which are not appropriate for this style of INSERT statement; as these helpers are the default in 1.4 this is effectively a regression. Additional checks to exclude this kind of statement from that particular extension have been added.

    References: [#6581](https://www.sqlalchemy.org/trac/ticket/6581)

### sqlite

*   **[sqlite] [bug]**

    Add note regarding encryption-related pragmas for pysqlcipher passed in the url.

    This change is also **backported** to: 1.3.25

    References: [#6589](https://www.sqlalchemy.org/trac/ticket/6589)

*   **[sqlite] [bug] [regression]**

    The fix for pysqlcipher released in version 1.4.3 [#5848](https://www.sqlalchemy.org/trac/ticket/5848) was unfortunately non-working, in that the new `on_connect_url` hook was erroneously not receiving a `URL` object under normal usage of `create_engine()` and instead received a string that was unhandled; the test suite failed to fully set up the actual conditions under which this hook is called. This has been fixed.

    References: [#6586](https://www.sqlalchemy.org/trac/ticket/6586)

## 1.4.17

Released: May 29, 2021

### orm

*   **[orm] [bug] [regression]**

    Fixed regression caused by just-released performance fix mentioned in #6550 where a query.join() to a relationship could produce an AttributeError if the query were made against non-ORM structures only, a fairly unusual calling pattern.

    References: [#6558](https://www.sqlalchemy.org/trac/ticket/6558)

## 1.4.16

Released: May 28, 2021

### general

*   **[general] [bug]**

    Resolved various deprecation warnings which were appearing as of Python version 3.10.0b1.

    References: [#6540](https://www.sqlalchemy.org/trac/ticket/6540), [#6543](https://www.sqlalchemy.org/trac/ticket/6543)

### orm

*   **[orm] [bug]**

    Fixed issue when using `relationship.cascade_backrefs` parameter set to `False`, which per cascade_backrefs behavior deprecated for removal in 2.0 is set to become the standard behavior in SQLAlchemy 2.0, where adding the item to a collection that uniquifies, such as `set` or `dict` would fail to fire a cascade event if the object were already associated in that collection via the backref. This fix represents a fundamental change in the collection mechanics by introducing a new event state which can fire off for a collection mutation even if there is no net change on the collection; the action is now suited using a new event hook `AttributeEvents.append_wo_mutation()`.

    References: [#6471](https://www.sqlalchemy.org/trac/ticket/6471)

*   **[orm] [bug] [regression]**

    Fixed regression involving clause adaption of labeled ORM compound elements, such as single-table inheritance discriminator expressions with conditionals or CASE expressions, which could cause aliased expressions such as those used in ORM join / joinedload operations to not be adapted correctly, such as referring to the wrong table in the ON clause in a join.

    This change also improves a performance bump that was located within the process of invoking `Select.join()` given an ORM attribute as a target.

    References: [#6550](https://www.sqlalchemy.org/trac/ticket/6550)

*   **[orm] [bug] [regression]**

    Fixed regression where the full combination of joined inheritance, global with_polymorphic, self-referential relationship and joined loading would fail to be able to produce a query with the scope of lazy loads and object refresh operations that also attempted to render the joined loader.

    References: [#6495](https://www.sqlalchemy.org/trac/ticket/6495)

*   **[orm] [bug]**

    Enhanced the bind resolution rules for `Session.execute()` so that when a non-ORM statement such as an `insert()` construct nonetheless is built against ORM objects, to the greatest degree possible the ORM entity will be used to resolve the bind, such as for a `Session` that has a bind map set up on a common superclass without specific mappers or tables named in the map.

    References: [#6484](https://www.sqlalchemy.org/trac/ticket/6484)

### engine

*   **[engine] [bug]**

    Fixed issue where an `@` sign in the database portion of a URL would not be interpreted correctly if the URL also had a username:password section.

    References: [#6482](https://www.sqlalchemy.org/trac/ticket/6482)

*   **[engine] [bug]**

    Fixed a long-standing issue with `URL` where query parameters following the question mark would not be parsed correctly if the URL did not contain a database portion with a backslash.

    References: [#6329](https://www.sqlalchemy.org/trac/ticket/6329)

### sql

*   **[sql] [bug] [regression]**

    Fixed regression in dynamic loader strategy and `relationship()` overall where the `relationship.order_by` parameter were stored as a mutable list, which could then be mutated when combined with additional “order_by” methods used against the dynamic query object, causing the ORDER BY criteria to continue to grow repetitively.

    References: [#6549](https://www.sqlalchemy.org/trac/ticket/6549)

### mssql

*   **[mssql] [usecase]**

    Implemented support for a `CTE` construct to be used directly as the target of a `delete()` construct, i.e. “WITH … AS cte DELETE FROM cte”. This appears to be a useful feature of SQL Server.

    References: [#6464](https://www.sqlalchemy.org/trac/ticket/6464)

### misc

*   **[bug] [ext]**

    Fixed a deprecation warning that was emitted when using `automap_base()` without passing an existing `Base`.

    References: [#6529](https://www.sqlalchemy.org/trac/ticket/6529)

*   **[bug] [pep484]**

    Remove pep484 types from the code. Current effort is around the stub package, and having typing in two places makes thing worse, since the types in the SQLAlchemy source were usually outdated compared to the version in the stubs.

    References: [#6461](https://www.sqlalchemy.org/trac/ticket/6461)

*   **[bug] [ext] [regression]**

    Fixed regression in the `sqlalchemy.ext.instrumentation` extension that prevented instrumentation disposal from working completely. This fix includes both a 1.4 regression fix as well as a fix for a related issue that existed in 1.3 also. As part of this change, the `sqlalchemy.ext.instrumentation.InstrumentationManager` class now has a new method `unregister()`, which replaces the previous method `dispose()`, which was not called as of version 1.4.

    References: [#6390](https://www.sqlalchemy.org/trac/ticket/6390)

## 1.4.15

Released: May 11, 2021

### general

*   **[general] [feature]**

    A new approach has been applied to the warnings system in SQLAlchemy to accurately predict the appropriate stack level for each warning dynamically. This allows evaluating the source of SQLAlchemy-generated warnings and deprecation warnings to be more straightforward as the warning will indicate the source line within end-user code, rather than from an arbitrary level within SQLAlchemy’s own source code.

    References: [#6241](https://www.sqlalchemy.org/trac/ticket/6241)

### orm

*   **[orm] [bug] [regression]**

    Fixed additional regression caused by “eager loaders run on unexpire” feature [#1763](https://www.sqlalchemy.org/trac/ticket/1763) where the feature would run for a `contains_eager()` eagerload option in the case that the `contains_eager()` were chained to an additional eager loader option, which would then produce an incorrect query as the original query-bound join criteria were no longer present.

    References: [#6449](https://www.sqlalchemy.org/trac/ticket/6449)

*   **[orm] [bug]**

    Fixed issue in subquery loader strategy which prevented caching from working correctly. This would have been seen in the logs as a “generated” message instead of “cached” for all subqueryload SQL emitted, which by saturating the cache with new keys would degrade overall performance; it also would produce “LRU size alert” warnings.

    References: [#6459](https://www.sqlalchemy.org/trac/ticket/6459)

### sql

*   **[sql] [bug]**

    Adjusted the logic added as part of [#6397](https://www.sqlalchemy.org/trac/ticket/6397) in 1.4.12 so that internal mutation of the `BindParameter` object occurs within the clause construction phase as it did before, rather than in the compilation phase. In the latter case, the mutation still produced side effects against the incoming construct and additionally could potentially interfere with other internal mutation routines.

    References: [#6460](https://www.sqlalchemy.org/trac/ticket/6460)

### mysql

*   **[mysql] [bug] [documentation]**

    Added support for the `ssl_check_hostname=` parameter in mysql connection URIs and updated the mysql dialect documentation regarding secure connections. Original pull request courtesy of Jerry Zhao.

    References: [#5397](https://www.sqlalchemy.org/trac/ticket/5397)

## 1.4.14

Released: May 6, 2021

### orm

*   **[orm] [bug] [regression]**

    Fixed regression involving `lazy='dynamic'` loader in conjunction with a detached object. The previous behavior was that the dynamic loader upon calling methods like `.all()` returns empty lists for detached objects without error, this has been restored; however a warning is now emitted as this is not the correct result. Other dynamic loader scenarios correctly raise `DetachedInstanceError`.

    References: [#6426](https://www.sqlalchemy.org/trac/ticket/6426)

### engine

*   **[engine] [usecase] [orm]**

    Applied consistent behavior to the use case of calling `.commit()` or `.rollback()` inside of an existing `.begin()` context manager, with the addition of potentially emitting SQL within the block subsequent to the commit or rollback. This change continues upon the change first added in [#6155](https://www.sqlalchemy.org/trac/ticket/6155) where the use case of calling “rollback” inside of a `.begin()` contextmanager block was proposed:

    *   calling `.commit()` or `.rollback()` will now be allowed without error or warning within all scopes, including that of legacy and future `Engine`, ORM `Session`, asyncio `AsyncEngine`. Previously, the `Session` disallowed this.

    *   The remaining scope of the context manager is then closed; when the block ends, a check is emitted to see if the transaction was already ended, and if so the block returns without action.

    *   It will now raise **an error** if subsequent SQL of any kind is emitted within the block, **after** `.commit()` or `.rollback()` is called. The block should be closed as the state of the executable object would otherwise be undefined in this state.

    References: [#6288](https://www.sqlalchemy.org/trac/ticket/6288)

*   **[engine] [bug] [regression]**

    Established a deprecation path for calling upon the `CursorResult.keys()` method for a statement that returns no rows to provide support for legacy patterns used by the “records” package as well as any other non-migrated applications. Previously, this would raise `ResourceClosedException` unconditionally in the same way as it does when attempting to fetch rows. While this is the correct behavior going forward, the `LegacyCursorResult` object will now in this case return an empty list for `.keys()` as it did in 1.3, while also emitting a 2.0 deprecation warning. The `_cursor.CursorResult`, used when using a 2.0-style “future” engine, will continue to raise as it does now.

    References: [#6427](https://www.sqlalchemy.org/trac/ticket/6427)

### sql

*   **[sql] [bug] [regression]**

    Fixed regression caused by the “empty in” change just made in [#6397](https://www.sqlalchemy.org/trac/ticket/6397) 1.4.12 where the expression needs to be parenthesized for the “not in” use case, otherwise the condition will interfere with the other filtering criteria.

    References: [#6428](https://www.sqlalchemy.org/trac/ticket/6428)

*   **[sql] [bug] [regression]**

    The `TypeDecorator` class will now emit a warning when used in SQL compilation with caching unless the `.cache_ok` flag is set to `True` or `False`. A new class-level attribute `TypeDecorator.cache_ok` may be set which will be used as an indication that all the parameters passed to the object are safe to be used as a cache key if set to `True`, `False` means they are not.

    References: [#6436](https://www.sqlalchemy.org/trac/ticket/6436)

## 1.4.13

Released: May 3, 2021

### orm

*   **[orm] [bug] [regression]**

    Fixed regression in `selectinload` loader strategy that would cause it to cache its internal state incorrectly when handling relationships that join across more than one column, such as when using a composite foreign key. The invalid caching would then cause other unrelated loader operations to fail.

    References: [#6410](https://www.sqlalchemy.org/trac/ticket/6410)

*   **[orm] [bug] [regression]**

    Fixed regression where `Query.filter_by()` would not work if the lead entity were a SQL function or other expression derived from the primary entity in question, rather than a simple entity or column of that entity. Additionally, improved the behavior of `Select.filter_by()` overall to work with column expressions even in a non-ORM context.

    References: [#6414](https://www.sqlalchemy.org/trac/ticket/6414)

*   **[orm] [bug] [regression]**

    Fixed regression where using `selectinload()` and `subqueryload()` to load a two-level-deep path would lead to an attribute error.

    References: [#6419](https://www.sqlalchemy.org/trac/ticket/6419)

*   **[orm] [bug] [regression]**

    Fixed regression where using the `noload()` loader strategy in conjunction with a “dynamic” relationship would lead to an attribute error as the noload strategy would attempt to apply itself to the dynamic loader.

    References: [#6420](https://www.sqlalchemy.org/trac/ticket/6420)

### engine

*   **[engine] [bug] [regression]**

    Restored a legacy transactional behavior that was inadvertently removed from the `Connection` as it was never tested as a known use case in previous versions, where calling upon the `Connection.begin_nested()` method, when no transaction is present, does not create a SAVEPOINT at all and instead starts an outer transaction, returning a `RootTransaction` object instead of a `NestedTransaction` object. This `RootTransaction` then will emit a real COMMIT on the database connection when committed. Previously, the 2.0 style behavior was present in all cases that would autobegin a transaction but not commit it, which is a behavioral change.

    When using a 2.0 style connection object, the behavior is unchanged from previous 1.4 versions; calling `Connection.begin_nested()` will “autobegin” the outer transaction if not already present, and then as instructed emit a SAVEPOINT, returning the `NestedTransaction` object. The outer transaction is committed by calling upon `Connection.commit()`, as is “commit-as-you-go” style usage.

    In non-“future” mode, while the old behavior is restored, it also emits a 2.0 deprecation warning as this is a legacy behavior.

    References: [#6408](https://www.sqlalchemy.org/trac/ticket/6408)

### asyncio

*   **[asyncio] [bug] [regression]**

    Fixed a regression introduced by [#6337](https://www.sqlalchemy.org/trac/ticket/6337) that would create an `asyncio.Lock` which could be attached to the wrong loop when instantiating the async engine before any asyncio loop was started, leading to an asyncio error message when attempting to use the engine under certain circumstances.

    References: [#6409](https://www.sqlalchemy.org/trac/ticket/6409)

### postgresql

*   **[postgresql] [usecase]**

    Add support for server side cursors in the pg8000 dialect for PostgreSQL. This allows use of the `Connection.execution_options.stream_results` option.

    References: [#6198](https://www.sqlalchemy.org/trac/ticket/6198)

## 1.4.12

Released: April 29, 2021

### orm

*   **[orm] [bug]**

    Fixed issue in `Session.bulk_save_objects()` when used with persistent objects which would fail to track the primary key of mappings where the column name of the primary key were different than the attribute name.

    This change is also **backported** to: 1.3.25

    References: [#6392](https://www.sqlalchemy.org/trac/ticket/6392)

*   **[orm] [bug] [caching] [regression]**

    Fixed critical regression where bound parameter tracking as used in the SQL caching system could fail to track all parameters for the case where the same SQL expression containing a parameter were used in an ORM-related query using a feature such as class inheritance, which was then embedded in an enclosing expression which would make use of that same expression multiple times, such as a UNION. The ORM would individually copy the individual SELECT statements as part of compilation with class inheritance, which then embedded in the enclosing statement would fail to accommodate for all parameters. The logic that tracks this condition has been adjusted to work for multiple copies of a parameter.

    References: [#6391](https://www.sqlalchemy.org/trac/ticket/6391)

*   **[orm] [bug]**

    Fixed two distinct issues mostly affecting `hybrid_property`, which would come into play under common mis-configuration scenarios that were silently ignored in 1.3, and now failed in 1.4, where the “expression” implementation would return a non `ClauseElement` such as a boolean value. For both issues, 1.3’s behavior was to silently ignore the mis-configuration and ultimately attempt to interpret the value as a SQL expression, which would lead to an incorrect query.

    *   Fixed issue regarding interaction of the attribute system with hybrid_property, where if the `__clause_element__()` method of the attribute returned a non-`ClauseElement` object, an internal `AttributeError` would lead the attribute to return the `expression` function on the hybrid_property itself, as the attribute error was against the name `.expression` which would invoke the `__getattr__()` method as a fallback. This now raises explicitly. In 1.3 the non-`ClauseElement` was returned directly.

    *   Fixed issue in SQL argument coercions system where passing the wrong kind of object to methods that expect column expressions would fail if the object were altogether not a SQLAlchemy object, such as a Python function, in cases where the object were not just coerced into a bound value. Again 1.3 did not have a comprehensive argument coercion system so this case would also pass silently.

    References: [#6350](https://www.sqlalchemy.org/trac/ticket/6350)

*   **[orm] [bug]**

    Fixed issue where using a `Select` as a subquery in an ORM context would modify the `Select` in place to disable eagerloads on that object, which would then cause that same `Select` to not eagerload if it were then re-used in a top-level execution context.

    References: [#6378](https://www.sqlalchemy.org/trac/ticket/6378)

*   **[orm] [bug] [regression]**

    Fixed issue where the new autobegin behavior failed to “autobegin” in the case where an existing persistent object has an attribute change, which would then impact the behavior of `Session.rollback()` in that no snapshot was created to be rolled back. The “attribute modify” mechanics have been updated to ensure “autobegin”, which does not perform any database work, does occur when persistent attributes change in the same manner as when `Session.add()` is called. This is a regression as in 1.3, the rollback() method always had a transaction to roll back and would expire every time.

    References: [#6359](https://www.sqlalchemy.org/trac/ticket/6359), [#6360](https://www.sqlalchemy.org/trac/ticket/6360)

*   **[orm] [bug] [regression]**

    Fixed regression in ORM where using hybrid property to indicate an expression from a different entity would confuse the column-labeling logic in the ORM and attempt to derive the name of the hybrid from that other class, leading to an attribute error. The owning class of the hybrid attribute is now tracked along with the name.

    References: [#6386](https://www.sqlalchemy.org/trac/ticket/6386)

*   **[orm] [bug] [regression]**

    Fixed regression in hybrid_property where a hybrid against a SQL function would generate an `AttributeError` when attempting to generate an entry for the `.c` collection of a subquery in some cases; among other things this would impact its use in cases like that of `Query.count()`.

    References: [#6401](https://www.sqlalchemy.org/trac/ticket/6401)

*   **[orm] [bug] [dataclasses]**

    Adjusted the declarative scan for dataclasses so that the inheritance behavior of `declared_attr()` established on a mixin, when using the new form of having it inside of a `dataclasses.field()` construct and not actually a descriptor attribute on the class, correctly accommodates the case when the target class to be mapped is a subclass of an existing mapped class which has already mapped that `declared_attr()`, and therefore should not be re-applied to this class.

    References: [#6346](https://www.sqlalchemy.org/trac/ticket/6346)

*   **[orm] [bug]**

    Fixed an issue with the (deprecated in 1.4) `ForeignKeyConstraint.copy()` method that caused an error when invoked with the `schema` argument.

    References: [#6353](https://www.sqlalchemy.org/trac/ticket/6353)

### engine

*   **[engine] [bug]**

    Fixed issue where usage of an explicit `Sequence` would produce inconsistent “inline” behavior for an `Insert` construct that includes multiple values phrases; the first seq would be inline but subsequent ones would be “pre-execute”, leading to inconsistent sequence ordering. The sequence expressions are now fully inline.

    References: [#6361](https://www.sqlalchemy.org/trac/ticket/6361)

### sql

*   **[sql] [bug]**

    Revised the “EMPTY IN” expression to no longer rely upon using a subquery, as this was causing some compatibility and performance problems. The new approach for selected databases takes advantage of using a NULL-returning IN expression combined with the usual “1 != 1” or “1 = 1” expression appended by AND or OR. The expression is now the default for all backends other than SQLite, which still had some compatibility issues regarding tuple “IN” for older SQLite versions.

    Third party dialects can still override how the “empty set” expression renders by implementing a new compiler method `def visit_empty_set_op_expr(self, type_, expand_op)`, which takes precedence over the existing `def visit_empty_set_expr(self, element_types)` which remains in place.

    References: [#6258](https://www.sqlalchemy.org/trac/ticket/6258), [#6397](https://www.sqlalchemy.org/trac/ticket/6397)

*   **[sql] [bug] [regression]**

    Fixed regression where usage of the `text()` construct inside the columns clause of a `Select` construct, which is better handled by using a `literal_column()` construct, would nonetheless prevent constructs like `union()` from working correctly. Other use cases, such as constructing subuqeries, continue to work the same as in prior versions where the `text()` construct is silently omitted from the collection of exported columns. Also repairs similar use within the ORM.

    References: [#6343](https://www.sqlalchemy.org/trac/ticket/6343)

*   **[sql] [bug] [regression]**

    Fixed regression involving legacy methods such as `Select.append_column()` where internal assertions would fail.

    References: [#6261](https://www.sqlalchemy.org/trac/ticket/6261)

*   **[sql] [bug] [regression]**

    Fixed regression caused by [#5395](https://www.sqlalchemy.org/trac/ticket/5395) where tuning back the check for sequences in `select()` now caused failures when doing 2.0-style querying with a mapped class that also happens to have an `__iter__()` method. Tuned the check some more to accommodate this as well as some other interesting `__iter__()` scenarios.

    References: [#6300](https://www.sqlalchemy.org/trac/ticket/6300)

### schema

*   **[schema] [bug] [mariadb] [mysql] [oracle] [postgresql]**

    Ensure that the MySQL and MariaDB dialect ignore the `Identity` construct while rendering the `AUTO_INCREMENT` keyword in a create table.

    The Oracle and PostgreSQL compiler was updated to not render `Identity` if the database version does not support it (Oracle < 12 and PostgreSQL < 10). Previously it was rendered regardless of the database version.

    References: [#6338](https://www.sqlalchemy.org/trac/ticket/6338)

### postgresql

*   **[postgresql] [bug]**

    Fixed very old issue where the `Enum` datatype would not inherit the `MetaData.schema` parameter of a `MetaData` object when that object were passed to the `Enum` using `Enum.metadata`.

    References: [#6373](https://www.sqlalchemy.org/trac/ticket/6373)

### sqlite

*   **[sqlite] [usecase]**

    Default to using `SingletonThreadPool` for in-memory SQLite databases created using URI filenames. Previously the default pool used was the `NullPool` that precented sharing the same database between multiple engines.

    References: [#6379](https://www.sqlalchemy.org/trac/ticket/6379)

### mssql

*   **[mssql] [bug] [schema]**

    Add `TypeEngine.as_generic()` support for `sqlalchemy.dialects.mysql.BIT` columns, mapping them to `Boolean`.

    References: [#6345](https://www.sqlalchemy.org/trac/ticket/6345)

*   **[mssql] [bug] [regression]**

    Fixed regression caused by [#6306](https://www.sqlalchemy.org/trac/ticket/6306) which added support for `DateTime(timezone=True)`, where the previous behavior of the pyodbc driver of implicitly dropping the tzinfo from a timezone-aware date when INSERTing into a timezone-naive DATETIME column were lost, leading to a SQL Server error when inserting timezone-aware datetime objects into timezone-native database columns.

    References: [#6366](https://www.sqlalchemy.org/trac/ticket/6366)

## 1.4.11

Released: April 21, 2021

### orm declarative

*   **[orm] [declarative] [bug] [regression]**

    Fixed regression where recent changes to support Python dataclasses had the inadvertent effect that an ORM mapped class could not successfully override the `__new__()` method.

    References: [#6331](https://www.sqlalchemy.org/trac/ticket/6331)

### engine

*   **[engine] [bug] [regression]**

    Fixed critical regression caused by the change in [#5497](https://www.sqlalchemy.org/trac/ticket/5497) where the connection pool “init” phase no longer occurred within mutexed isolation, allowing other threads to proceed with the dialect uninitialized, which could then impact the compilation of SQL statements.

    References: [#6337](https://www.sqlalchemy.org/trac/ticket/6337)

## 1.4.10

Released: April 20, 2021

### orm

*   **[orm] [usecase]**

    Altered some of the behavior repaired in [#6232](https://www.sqlalchemy.org/trac/ticket/6232) where the `immediateload` loader strategy no longer goes into recursive loops; the modification is that an eager load (joinedload, selectinload, or subqueryload) from A->bs->B which then states `immediateload` for a simple manytoone B->a->A that’s in the identity map will populate the B->A, so that this attribute is back-populated when the collection of A/A.bs are loaded. This allows the objects to be functional when detached.

*   **[orm] [bug]**

    Fixed bug in new `with_loader_criteria()` feature where using a mixin class with `declared_attr()` on an attribute that were accessed inside the custom lambda would emit a warning regarding using an unmapped declared attr, when the lambda callable were first initialized. This warning is now prevented using special instrumentation for this lambda initialization step.

    References: [#6320](https://www.sqlalchemy.org/trac/ticket/6320)

*   **[orm] [bug] [regression]**

    Fixed additional regression caused by the “eagerloaders on refresh” feature added in [#1763](https://www.sqlalchemy.org/trac/ticket/1763) where the refresh operation historically would set `populate_existing`, which given the new feature now overwrites pending changes on eagerly loaded objects when autoflush is false. The populate_existing flag has been turned off for this case and a more specific method used to ensure the correct attributes refreshed.

    References: [#6326](https://www.sqlalchemy.org/trac/ticket/6326)

*   **[orm] [bug] [result]**

    Fixed an issue when using 2.0 style execution that prevented using `Result.scalar_one()` or `Result.scalar_one_or_none()` after calling `Result.unique()`, for the case where the ORM is returning a single-element row in any case.

    References: [#6299](https://www.sqlalchemy.org/trac/ticket/6299)

### sql

*   **[sql] [bug]**

    Fixed issue in SQL compiler where the bound parameters set up for a `Values` construct wouldn’t be positionally tracked correctly if inside of a `CTE`, affecting database drivers that support VALUES + ctes and use positional parameters such as SQL Server in particular as well as asyncpg. The fix also repairs support for compiler flags such as `literal_binds`.

    References: [#6327](https://www.sqlalchemy.org/trac/ticket/6327)

*   **[sql] [bug]**

    Repaired and solidified issues regarding custom functions and other arbitrary expression constructs which within SQLAlchemy’s column labeling mechanics would seek to use `str(obj)` to get a string representation to use as an anonymous column name in the `.c` collection of a subquery. This is a very legacy behavior that performs poorly and leads to lots of issues, so has been revised to no longer perform any compilation by establishing specific methods on `FunctionElement` to handle this case, as SQL functions are the only use case that it came into play. An effect of this behavior is that an unlabeled column expression with no derivable name will be given an arbitrary label starting with the prefix `"_no_label"` in the `.c` collection of a subquery; these were previously being represented either as the generic stringification of that expression, or as an internal symbol.

    References: [#6256](https://www.sqlalchemy.org/trac/ticket/6256)

### schema

*   **[schema] [bug]**

    Fixed issue where `next_value()` was not deriving its type from the corresponding `Sequence`, instead hardcoded to `Integer`. The specific numeric type is now used.

    References: [#6287](https://www.sqlalchemy.org/trac/ticket/6287)

### mypy

*   **[mypy] [bug]**

    Fixed issue where mypy plugin would not correctly interpret an explicit `Mapped` annotation in conjunction with a `relationship()` that refers to a class by string name; the correct annotation would be downgraded to a less specific one leading to typing errors.

    References: [#6255](https://www.sqlalchemy.org/trac/ticket/6255)

### mssql

*   **[mssql] [usecase]**

    The `DateTime.timezone` parameter when set to `True` will now make use of the `DATETIMEOFFSET` column type with SQL Server when used to emit DDL, rather than `DATETIME` where the flag was silently ignored.

    References: [#6306](https://www.sqlalchemy.org/trac/ticket/6306)

### misc

*   **[bug] [declarative] [regression]**

    Fixed `instrument_declarative()` that called a non existing registry method.

    References: [#6291](https://www.sqlalchemy.org/trac/ticket/6291)

## 1.4.9

Released: April 17, 2021

### orm

*   **[orm] [usecase]**

    Established support for `synoynm()` in conjunction with hybrid property, assocaitionproxy is set up completely, including that synonyms can be established linking to these constructs which work fully. This is a behavior that was semi-explicitly disallowed previously, however since it did not fail in every scenario, explicit support for assoc proxy and hybrids has been added.

    References: [#6267](https://www.sqlalchemy.org/trac/ticket/6267)

*   **[orm] [performance] [bug] [regression] [sql]**

    Fixed a critical performance issue where the traversal of a `select()` construct would traverse a repetitive product of the represented FROM clauses as they were each referenced by columns in the columns clause; for a series of nested subqueries with lots of columns this could cause a large delay and significant memory growth. This traversal is used by a wide variety of SQL and ORM functions, including by the ORM `Session` when it’s configured to have “table-per-bind”, which while this is not a common use case, it seems to be what Flask-SQLAlchemy is hardcoded as using, so the issue impacts Flask-SQLAlchemy users. The traversal has been repaired to uniqify on FROM clauses which was effectively what would happen implicitly with the pre-1.4 architecture.

    References: [#6304](https://www.sqlalchemy.org/trac/ticket/6304)

*   **[orm] [bug] [regression]**

    Fixed regression where an attribute that is mapped to a `synonym()` could not be used in column loader options such as `load_only()`.

    References: [#6272](https://www.sqlalchemy.org/trac/ticket/6272)

### sql

*   **[sql] [bug] [regression]**

    Fixed regression where an empty in statement on a tuple would result in an error when compiled with the option `literal_binds=True`.

    References: [#6290](https://www.sqlalchemy.org/trac/ticket/6290)

### postgresql

*   **[postgresql] [bug] [regression] [sql]**

    Fixed an argument error in the default and PostgreSQL compilers that would interfere with an UPDATE..FROM or DELETE..FROM..USING statement that was then SELECTed from as a CTE.

    References: [#6303](https://www.sqlalchemy.org/trac/ticket/6303)

## 1.4.8

Released: April 15, 2021

### orm

*   **[orm] [bug] [regression]**

    Fixed a cache leak involving the `with_expression()` loader option, where the given SQL expression would not be correctly considered as part of the cache key.

    Additionally, fixed regression involving the corresponding `query_expression()` feature. While the bug technically exists in 1.3 as well, it was not exposed until 1.4\. The “default expr” value of `null()` would be rendered when not needed, and additionally was also not adapted correctly when the ORM rewrites statements such as when using joined eager loading. The fix ensures “singleton” expressions like `NULL` and `true` aren’t “adapted” to refer to columns in ORM statements, and additionally ensures that a `query_expression()` with no default expression doesn’t render in the statement if a `with_expression()` isn’t used.

    References: [#6259](https://www.sqlalchemy.org/trac/ticket/6259)

*   **[orm] [bug]**

    Fixed issue in the new feature of `Session.refresh()` introduced by [#1763](https://www.sqlalchemy.org/trac/ticket/1763) where eagerly loaded relationships are also refreshed, where the `lazy="raise"` and `lazy="raise_on_sql"` loader strategies would interfere with the `immediateload()` loader strategy, thus breaking the feature for relationships that were loaded with `selectinload()`, `subqueryload()` as well.

    References: [#6252](https://www.sqlalchemy.org/trac/ticket/6252)

### engine

*   **[engine] [bug]**

    The `Dialect.has_table()` method now raises an informative exception if a non-Connection is passed to it, as this incorrect behavior seems to be common. This method is not intended for external use outside of a dialect. Please use the `Inspector.has_table()` method or for cross-compatibility with older SQLAlchemy versions, the `Engine.has_table()` method.

### sql

*   **[sql] [feature]**

    The tuple returned by `CursorResult.inserted_primary_key` is now a `Row` object with a named tuple interface on top of the existing tuple interface.

    References: [#3314](https://www.sqlalchemy.org/trac/ticket/3314)

*   **[sql] [bug] [regression]**

    Fixed regression where the `BindParameter` object would not properly render for an IN expression (i.e. using the “post compile” feature in 1.4) if the object were copied from either an internal cloning operation, or from a pickle operation, and the parameter name contained spaces or other special characters.

    References: [#6249](https://www.sqlalchemy.org/trac/ticket/6249)

*   **[sql] [bug] [regression] [sqlite]**

    Fixed regression where the introduction of the INSERT syntax “INSERT… VALUES (DEFAULT)” was not supported on some backends that do however support “INSERT..DEFAULT VALUES”, including SQLite. The two syntaxes are now each individually supported or non-supported for each dialect, for example MySQL supports “VALUES (DEFAULT)” but not “DEFAULT VALUES”. Support for Oracle has also been enabled.

    References: [#6254](https://www.sqlalchemy.org/trac/ticket/6254)

### mypy

*   **[mypy] [change]**

    Updated Mypy plugin to only use the public plugin interface of the semantic analyzer.

*   **[mypy] [bug]**

    Revised the fix for `OrderingList` from version 1.4.7 which was testing against the incorrect API.

    References: [#6205](https://www.sqlalchemy.org/trac/ticket/6205)

### asyncio

*   **[asyncio] [bug]**

    Fix typo that prevented setting the `bind` attribute of an `AsyncSession` to the correct value.

    References: [#6220](https://www.sqlalchemy.org/trac/ticket/6220)

### mssql

*   **[mssql] [bug] [regression]**

    Fixed an additional regression in the same area as that of [#6173](https://www.sqlalchemy.org/trac/ticket/6173), [#6184](https://www.sqlalchemy.org/trac/ticket/6184), where using a value of 0 for OFFSET in conjunction with LIMIT with SQL Server would create a statement using “TOP”, as was the behavior in 1.3, however due to caching would then fail to respond accordingly to other values of OFFSET. If the “0” wasn’t first, then it would be fine. For the fix, the “TOP” syntax is now only emitted if the OFFSET value is omitted entirely, that is, `Select.offset()` is not used. Note that this change now requires that if the “with_ties” or “percent” modifiers are used, the statement can’t specify an OFFSET of zero, it now needs to be omitted entirely.

    References: [#6265](https://www.sqlalchemy.org/trac/ticket/6265)

## 1.4.7

Released: April 9, 2021

### orm

*   **[orm] [bug] [regression]**

    Fixed regression where the `subqueryload()` loader strategy would fail to correctly accommodate sub-options, such as a `defer()` option on a column, if the “path” of the subqueryload were more than one level deep.

    References: [#6221](https://www.sqlalchemy.org/trac/ticket/6221)

*   **[orm] [bug] [regression]**

    Fixed regression where the `merge_frozen_result()` function relied upon by the dogpile.caching example was not included in tests and began failing due to incorrect internal arguments.

    References: [#6211](https://www.sqlalchemy.org/trac/ticket/6211)

*   **[orm] [bug] [regression]**

    Fixed critical regression where the `Session` could fail to “autobegin” a new transaction when a flush occurred without an existing transaction in place, implicitly placing the `Session` into legacy autocommit mode which commit the transaction. The `Session` now has a check that will prevent this condition from occurring, in addition to repairing the flush issue.

    Additionally, scaled back part of the change made as part of [#5226](https://www.sqlalchemy.org/trac/ticket/5226) which can run autoflush during an unexpire operation, to not actually do this in the case of a `Session` using legacy `Session.autocommit` mode, as this incurs a commit within a refresh operation.

    References: [#6233](https://www.sqlalchemy.org/trac/ticket/6233)

*   **[orm] [bug] [regression]**

    Fixed regression where the ORM compilation scheme would assume the function name of a hybrid property would be the same as the attribute name in such a way that an `AttributeError` would be raised, when it would attempt to determine the correct name for each element in a result tuple. A similar issue exists in 1.3 but only impacts the names of tuple rows. The fix here adds a check that the hybrid’s function name is actually present in the `__dict__` of the class or its superclasses before assigning this name; otherwise, the hybrid is considered to be “unnamed” and ORM result tuples will use the naming scheme of the underlying expression.

    References: [#6215](https://www.sqlalchemy.org/trac/ticket/6215)

*   **[orm] [bug] [regression]**

    Fixed critical regression caused by the new feature added as part of [#1763](https://www.sqlalchemy.org/trac/ticket/1763), eager loaders are invoked on unexpire operations. The new feature makes use of the “immediateload” eager loader strategy as a substitute for a collection loading strategy, which unlike the other “post-load” strategies was not accommodating for recursive invocations between mutually-dependent relationships, leading to recursion overflow errors.

    References: [#6232](https://www.sqlalchemy.org/trac/ticket/6232)

### engine

*   **[engine] [bug] [regression]**

    Fixed up the behavior of the `Row` object when dictionary access is used upon it, meaning converting to a dict via `dict(row)` or accessing members using strings or other objects i.e. `row["some_key"]` works as it would with a dictionary, rather than raising `TypeError` as would be the case with a tuple, whether or not the C extensions are in place. This was originally supposed to emit a 2.0 deprecation warning for the “non-future” case using `LegacyRow`, and was to raise `TypeError` for the “future” `Row` class. However, the C version of `Row` was failing to raise this `TypeError`, and to complicate matters, the `Session.execute()` method now returns `Row` in all cases to maintain consistency with the ORM result case, so users who didn’t have C extensions installed would see different behavior in this one case for existing pre-1.4 style code.

    Therefore, in order to soften the overall upgrade scheme as most users have not been exposed to the more strict behavior of `Row` up through 1.4.6, `LegacyRow` and `Row` both provide for string-key access as well as support for `dict(row)`, in all cases emitting the 2.0 deprecation warning when `SQLALCHEMY_WARN_20` is enabled. The `Row` object still uses tuple-like behavior for `__contains__`, which is probably the only noticeable behavioral change compared to `LegacyRow`, other than the removal of dictionary-style methods `values()` and `items()`.

    References: [#6218](https://www.sqlalchemy.org/trac/ticket/6218)

### sql

*   **[sql] [bug] [regression]**

    Enhanced the “expanding” feature used for `ColumnOperators.in_()` operations to infer the type of expression from the right hand list of elements, if the left hand side does not have any explicit type set up. This allows the expression to support stringification among other things. In 1.3, “expanding” was not automatically used for `ColumnOperators.in_()` expressions, so in that sense this change fixes a behavioral regression.

    References: [#6222](https://www.sqlalchemy.org/trac/ticket/6222)

*   **[sql] [bug]**

    Fixed the “stringify” compiler to support a basic stringification of a “multirow” INSERT statement, i.e. one with multiple tuples following the VALUES keyword.

### schema

*   **[schema] [bug] [regression]**

    Fixed regression where usage of a token in the `Connection.execution_options.schema_translate_map` dictionary which contained special characters such as braces would fail to be substituted properly. Use of square bracket characters `[]` is now explicitly disallowed as these are used as a delimiter character in the current implementation.

    References: [#6216](https://www.sqlalchemy.org/trac/ticket/6216)

### mypy

*   **[mypy] [bug]**

    Fixed issue in Mypy plugin where the plugin wasn’t inferring the correct type for columns of subclasses that don’t directly descend from `TypeEngine`, in particular that of `TypeDecorator` and `UserDefinedType`.

### tests

*   **[tests] [change]**

    Added a new flag to `DefaultDialect` called `supports_schemas`; third party dialects may set this flag to `False` to disable SQLAlchemy’s schema-level tests when running the test suite for a third party dialect.

## 1.4.6

Released: April 6, 2021

### orm

*   **[orm] [bug] [regression]**

    Fixed regression where a deprecated form of `Query.join()` were used, passing a series of entities to join from without any ON clause in a single `Query.join()` call, would fail to function correctly.

    References: [#6203](https://www.sqlalchemy.org/trac/ticket/6203)

*   **[orm] [bug] [regression]**

    Fixed critical regression where the `Query.yield_per()` method in the ORM would set up the internal `Result` to yield chunks at a time, however made use of the new `Result.unique()` method which uniques across the entire result. This would lead to lost rows since the ORM is using `id(obj)` as the uniquing function, which leads to repeated identifiers for new objects as already-seen objects are garbage collected. 1.3’s behavior here was to “unique” across each chunk, which does not actually produce “uniqued” results when results are yielded in chunks. As the `Query.yield_per()` method is already explicitly disallowed when joined eager loading is in place, which is the primary rationale for the “uniquing” feature, the “uniquing” feature is now turned off entirely when `Query.yield_per()` is used.

    This regression only applies to the legacy `Query` object; when using 2.0 style execution, “uniquing” is not automatically applied. To prevent the issue from arising from explicit use of `Result.unique()`, an error is now raised if rows are fetched from a “uniqued” ORM-level `Result` if any yield per API is also in use, as the purpose of `yield_per` is to allow for arbitrarily large numbers of rows, which cannot be uniqued in memory without growing the number of entries to fit the complete result size.

    References: [#6206](https://www.sqlalchemy.org/trac/ticket/6206)

### sql

*   **[sql] [bug] [mssql] [oracle] [regression]**

    Fixed further regressions in the same area as that of [#6173](https://www.sqlalchemy.org/trac/ticket/6173) released in 1.4.5, where a “postcompile” parameter, again most typically those used for LIMIT/OFFSET rendering in Oracle and SQL Server, would fail to be processed correctly if the same parameter rendered in multiple places in the statement.

    References: [#6202](https://www.sqlalchemy.org/trac/ticket/6202)

*   **[sql] [bug]**

    Executing a `Subquery` using `Connection.execute()` is deprecated and will emit a deprecation warning; this use case was an oversight that should have been removed from 1.4\. The operation will now execute the underlying `Select` object directly for backwards compatibility. Similarly, the `CTE` class is also not appropriate for execution. In 1.3, attempting to execute a CTE would result in an invalid “blank” SQL statement being executed; since this use case was not working it now raises `ObjectNotExecutableError`. Previously, 1.4 was attempting to execute the CTE as a statement however it was working only erratically.

    References: [#6204](https://www.sqlalchemy.org/trac/ticket/6204)

### schema

*   **[schema] [bug]**

    The `Table` object now raises an informative error message if it is instantiated without passing at least the `Table.name` and `Table.metadata` arguments positionally. Previously, if these were passed as keyword arguments, the object would silently fail to initialize correctly.

    This change is also **backported** to: 1.3.25

    References: [#6135](https://www.sqlalchemy.org/trac/ticket/6135)

### mypy

*   **[mypy] [bug]**

    Applied a series of refactorings and fixes to accommodate for Mypy “incremental” mode across multiple files, which previously was not taken into account. In this mode the Mypy plugin has to accommodate Python datatypes expressed in other files coming in with less information than they have on a direct run.

    Additionally, a new decorator `declarative_mixin()` is added, which is necessary for the Mypy plugin to be able to definifitely identify a Declarative mixin class that is otherwise not used inside a particular Python file.

    See also

    Using @declared_attr and Declarative Mixins

    References: [#6147](https://www.sqlalchemy.org/trac/ticket/6147)

*   **[mypy] [bug]**

    Fixed issue where the Mypy plugin would fail to interpret the “collection_class” of a relationship if it were a callable and not a class. Also improved type matching and error reporting for collection-oriented relationships.

    References: [#6205](https://www.sqlalchemy.org/trac/ticket/6205)

### asyncio

*   **[asyncio] [usecase] [postgresql]**

    Added accessors `.sqlstate` and synonym `.pgcode` to the `.orig` attribute of the SQLAlchemy exception class raised by the asyncpg DBAPI adapter, that is, the intermediary exception object that wraps on top of that raised by the asyncpg library itself, but below the level of the SQLAlchemy dialect.

    References: [#6199](https://www.sqlalchemy.org/trac/ticket/6199)

## 1.4.5

Released: April 2, 2021

### orm

*   **[orm] [bug] [regression]**

    Fixed regression where the `joinedload()` loader strategy would not successfully joinedload to a mapper that is mapper against a `CTE` construct.

    References: [#6172](https://www.sqlalchemy.org/trac/ticket/6172)

*   **[orm] [bug] [regression]**

    Scaled back the warning message added in [#5171](https://www.sqlalchemy.org/trac/ticket/5171) to not warn for overlapping columns in an inheritance scenario where a particular relationship is local to a subclass and therefore does not represent an overlap.

    References: [#6171](https://www.sqlalchemy.org/trac/ticket/6171)

### sql

*   **[sql] [bug] [postgresql]**

    Fixed bug in new `FunctionElement.render_derived()` feature where column names rendered out explicitly in the alias SQL would not have proper quoting applied for case sensitive names and other non-alphanumeric names.

    References: [#6183](https://www.sqlalchemy.org/trac/ticket/6183)

*   **[sql] [bug] [regression]**

    Fixed regression where use of the `Operators.in_()` method with a `Select` object against a non-table-bound column would produce an `AttributeError`, or more generally using a `ScalarSelect` that has no datatype in a binary expression would produce invalid state.

    References: [#6181](https://www.sqlalchemy.org/trac/ticket/6181)

*   **[sql] [bug]**

    Added a new flag to the `Dialect` class called `Dialect.supports_statement_cache`. This flag now needs to be present directly on a dialect class in order for SQLAlchemy’s query cache to take effect for that dialect. The rationale is based on discovered issues such as [#6173](https://www.sqlalchemy.org/trac/ticket/6173) revealing that dialects which hardcode literal values from the compiled statement, often the numerical parameters used for LIMIT / OFFSET, will not be compatible with caching until these dialects are revised to use the parameters present in the statement only. For third party dialects where this flag is not applied, the SQL logging will show the message “dialect does not support caching”, indicating the dialect should seek to apply this flag once they have verified that no per-statement literal values are being rendered within the compilation phase.

    See also

    Caching for Third Party Dialects

    References: [#6184](https://www.sqlalchemy.org/trac/ticket/6184)

### schema

*   **[schema] [bug]**

    Introduce a new parameter `Enum.omit_aliases` in `Enum` type allow filtering aliases when using a pep435 Enum. Previous versions of SQLAlchemy kept aliases in all cases, creating database enum type with additional states, meaning that they were treated as different values in the db. For backward compatibility this flag defaults to `False` in the 1.4 series, but will be switched to `True` in a future version. A deprecation warning is raise if this flag is not specified and the passed enum contains aliases.

    References: [#6146](https://www.sqlalchemy.org/trac/ticket/6146)

### mypy

*   **[mypy] [bug]**

    Fixed issue in mypy plugin where newly added support for `as_declarative()` needed to more fully add the `DeclarativeMeta` class to the mypy interpreter’s state so that it does not result in a name not found error; additionally improves how global names are setup for the plugin including the `Mapped` name.

    References: [#sqlalchemy/sqlalchemy2-stubs/#14](https://www.sqlalchemy.org/trac/ticket/sqlalchemy/sqlalchemy2-stubs/#14)

### asyncio

*   **[asyncio] [bug]**

    Fixed issue where the asyncio extension could not be loaded if running Python 3.6 with the backport library of `contextvars` installed.

    References: [#6166](https://www.sqlalchemy.org/trac/ticket/6166)

### postgresql

*   **[postgresql] [bug] [regression]**

    Fixed regression caused by [#6023](https://www.sqlalchemy.org/trac/ticket/6023) where the PostgreSQL cast operator applied to elements within an `ARRAY` when using psycopg2 would fail to use the correct type in the case that the datatype were also embedded within an instance of the `Variant` adapter.

    Additionally, repairs support for the correct CREATE TYPE to be emitted when using a `Variant(ARRAY(some_schema_type))`.

    This change is also **backported** to: 1.3.25

    References: [#6182](https://www.sqlalchemy.org/trac/ticket/6182)

*   **[postgresql] [bug]**

    Fixed typo in the fix for [#6099](https://www.sqlalchemy.org/trac/ticket/6099) released in 1.4.4 that completely prevented this change from working correctly, i.e. the error message did not match what was actually emitted by pg8000.

    References: [#6099](https://www.sqlalchemy.org/trac/ticket/6099)

*   **[postgresql] [bug]**

    Fixed issue where the PostgreSQL `PGInspector`, when generated against an `Engine`, would fail for `.get_enums()`, `.get_view_names()`, `.get_foreign_table_names()` and `.get_table_oid()` when used against a “future” style engine and not the connection directly.

    References: [#6170](https://www.sqlalchemy.org/trac/ticket/6170)

### mysql

*   **[mysql] [bug] [regression]**

    Fixed regression in the MySQL dialect where the reflection query used to detect if a table exists would fail on very old MySQL 5.0 and 5.1 versions.

    References: [#6163](https://www.sqlalchemy.org/trac/ticket/6163)

### mssql

*   **[mssql] [bug]**

    Fixed a regression in MSSQL 2012+ that prevented the order by clause to be rendered when `offset=0` is used in a subquery.

    References: [#6163](https://www.sqlalchemy.org/trac/ticket/6163)

### oracle

*   **[oracle] [bug] [regression]**

    Fixed critical regression where the Oracle compiler would not maintain the correct parameter values in the LIMIT/OFFSET for a select due to a caching issue.

    References: [#6173](https://www.sqlalchemy.org/trac/ticket/6173)

## 1.4.4

Released: March 30, 2021

### orm

*   **[orm] [bug]**

    Fixed critical issue in the new `PropComparator.and_()` feature where loader strategies that emit secondary SELECT statements such as `selectinload()` and `lazyload()` would fail to accommodate for bound parameters in the user-defined criteria in terms of the current statement being executed, as opposed to the cached statement, causing stale bound values to be used.

    This also adds a warning for the case where an object that uses `lazyload()` in conjunction with `PropComparator.and_()` is attempted to be serialized; the loader criteria cannot reliably be serialized and deserialized and eager loading should be used for this case.

    References: [#6139](https://www.sqlalchemy.org/trac/ticket/6139)

*   **[orm] [bug] [regression]**

    Fixed missing method `Session.get()` from the `ScopedSession` interface.

    References: [#6144](https://www.sqlalchemy.org/trac/ticket/6144)

### engine

*   **[engine] [usecase]**

    Modified the context manager used by `Transaction` so that an “already detached” warning is not emitted by the ending of the context manager itself, if the transaction were already manually rolled back inside the block. This applies to regular transactions, savepoint transactions, and legacy “marker” transactions. A warning is still emitted if the `.rollback()` method is called explicitly more than once.

    References: [#6155](https://www.sqlalchemy.org/trac/ticket/6155)

*   **[engine] [bug]**

    Repair wrong arguments to exception handling method in CursorResult.

    References: [#6138](https://www.sqlalchemy.org/trac/ticket/6138)

### postgresql

*   **[postgresql] [bug] [reflection]**

    Fixed issue in PostgreSQL reflection where a column expressing “NOT NULL” will supersede the nullability of a corresponding domain.

    This change is also **backported** to: 1.3.24

    References: [#6161](https://www.sqlalchemy.org/trac/ticket/6161)

*   **[postgresql] [bug]**

    Modified the `is_disconnect()` handler for the pg8000 dialect, which now accommodates for a new `InterfaceError` emitted by pg8000 1.19.0\. Pull request courtesy Hamdi Burak Usul.

    References: [#6099](https://www.sqlalchemy.org/trac/ticket/6099)

### misc

*   **[misc] [bug]**

    Adjusted the usage of the `importlib_metadata` library for loading setuptools entrypoints in order to accommodate for some deprecation changes.

## 1.4.3

Released: March 25, 2021

### orm

*   **[orm] [bug]**

    Fixed a bug where python 2.7.5 (default on CentOS 7) wasn’t able to import sqlalchemy, because on this version of Python `exec "statement"` and `exec("statement")` do not behave the same way. The compatibility `exec_()` function was used instead.

    References: [#6069](https://www.sqlalchemy.org/trac/ticket/6069)

*   **[orm] [bug]**

    Fixed bug where ORM queries using a correlated subquery in conjunction with `column_property()` would fail to correlate correctly to an enclosing subquery or to a CTE when `Select.correlate_except()` were used in the property to control correlation, in cases where the subquery contained the same selectables as ones within the correlated subquery that were intended to not be correlated.

    References: [#6060](https://www.sqlalchemy.org/trac/ticket/6060)

*   **[orm] [bug]**

    Fixed bug where combinations of the new “relationship with criteria” feature could fail in conjunction with features that make use of the new “lambda SQL” feature, including loader strategies such as selectinload and lazyload, for more complicated scenarios such as polymorphic loading.

    References: [#6131](https://www.sqlalchemy.org/trac/ticket/6131)

*   **[orm] [bug]**

    Repaired support so that the `ClauseElement.params()` method can work correctly with a `Select` object that includes joins across ORM relationship structures, which is a new feature in 1.4.

    References: [#6124](https://www.sqlalchemy.org/trac/ticket/6124)

*   **[orm] [bug]**

    Fixed issue where a “removed in 2.0” warning were generated internally by the relationship loader mechanics.

    References: [#6115](https://www.sqlalchemy.org/trac/ticket/6115)

### orm declarative

*   **[orm] [declarative] [bug] [regression]**

    Fixed regression where the `.metadata` attribute on a per class level would not be honored, breaking the use case of per-class-hierarchy `MetaData` for abstract declarative classes and mixins.

    See also

    metadata

    References: [#6128](https://www.sqlalchemy.org/trac/ticket/6128)

### engine

*   **[engine] [bug] [regression]**

    Restored the `ResultProxy` name back to the `sqlalchemy.engine` namespace. This name refers to the `LegacyCursorResult` object.

    References: [#6119](https://www.sqlalchemy.org/trac/ticket/6119)

### schema

*   **[schema] [bug]**

    Adjusted the logic that emits DROP statements for `Sequence` objects among the dropping of multiple tables, such that all `Sequence` objects are dropped after all tables, even if the given `Sequence` is related only to a `Table` object and not directly to the overall `MetaData` object. The use case supports the same `Sequence` being associated with more than one `Table` at a time.

    This change is also **backported** to: 1.3.24

    References: [#6071](https://www.sqlalchemy.org/trac/ticket/6071)

### mypy

*   **[mypy] [bug]**

    Added support for the Mypy extension to correctly interpret a declarative base class that’s generated using the `as_declarative()` function as well as the `registry.as_declarative_base()` method.

*   **[mypy] [bug]**

    Fixed bug in Mypy plugin where the Python type detection for the `Boolean` column type would produce an exception; additionally implemented support for `Enum`, including detection of a string-based enum vs. use of Python `enum.Enum`.

    References: [#6109](https://www.sqlalchemy.org/trac/ticket/6109)

### postgresql

*   **[postgresql] [bug] [types]**

    Adjusted the psycopg2 dialect to emit an explicit PostgreSQL-style cast for bound parameters that contain ARRAY elements. This allows the full range of datatypes to function correctly within arrays. The asyncpg dialect already generated these internal casts in the final statement. This also includes support for array slice updates as well as the PostgreSQL-specific `ARRAY.contains()` method.

    This change is also **backported** to: 1.3.24

    References: [#6023](https://www.sqlalchemy.org/trac/ticket/6023)

*   **[postgresql] [bug] [reflection]**

    Fixed reflection of identity columns in tables with mixed case names in PostgreSQL.

    References: [#6129](https://www.sqlalchemy.org/trac/ticket/6129)

### sqlite

*   **[sqlite] [feature] [asyncio]**

    Added support for the aiosqlite database driver for use with the SQLAlchemy asyncio extension.

    See also

    Aiosqlite

    References: [#5920](https://www.sqlalchemy.org/trac/ticket/5920)

*   **[sqlite] [bug] [regression]**

    Repaired the `pysqlcipher` dialect to connect correctly which had regressed in 1.4, and added test + CI support to maintain the driver in working condition. The dialect now imports the `sqlcipher3` module for Python 3 by default before falling back to `pysqlcipher3` which is documented as now being unmaintained.

    See also

    Pysqlcipher

    References: [#5848](https://www.sqlalchemy.org/trac/ticket/5848)

## 1.4.2

Released: March 19, 2021

### orm

*   **[orm] [usecase] [dataclasses]**

    Added support for the `declared_attr` object to work in the context of dataclass fields.

    See also

    Using Declarative Mixins with pre-existing dataclasses

    References: [#6100](https://www.sqlalchemy.org/trac/ticket/6100)

*   **[orm] [bug] [dataclasses]**

    Fixed issue in new ORM dataclasses functionality where dataclass fields on an abstract base or mixin that contained column or other mapping constructs would not be mapped if they also included a “default” key within the dataclasses.field() object.

    References: [#6093](https://www.sqlalchemy.org/trac/ticket/6093)

*   **[orm] [bug] [regression]**

    Fixed regression where the `Query.selectable` accessor, which is a synonym for `Query.__clause_element__()`, got removed, it’s now restored.

    References: [#6088](https://www.sqlalchemy.org/trac/ticket/6088)

*   **[orm] [bug] [regression]**

    Fixed regression where use of an unnamed SQL expression such as a SQL function would raise a column targeting error if the query itself were using joinedload for an entity and was also being wrapped in a subquery by the joinedload eager loading process.

    References: [#6086](https://www.sqlalchemy.org/trac/ticket/6086)

*   **[orm] [bug] [regression]**

    Fixed regression where the `Query.filter_by()` method would fail to locate the correct source entity if the `Query.join()` method had been used targeting an entity without any kind of ON clause.

    References: [#6092](https://www.sqlalchemy.org/trac/ticket/6092)

*   **[orm] [bug] [regression]**

    Fixed regression where the SQL compilation of a `Function` would not work correctly if the object had been “annotated”, which is an internal memoization process used mostly by the ORM. In particular it could affect ORM lazy loads which make greater use of this feature in 1.4.

    References: [#6095](https://www.sqlalchemy.org/trac/ticket/6095)

*   **[orm] [bug]**

    Fixed regression where the `ConcreteBase` would fail to map at all when a mapped column name overlapped with the discriminator column name, producing an assertion error. The use case here did not function correctly in 1.3 as the polymorphic union would produce a query that ignored the discriminator column entirely, while emitting duplicate column warnings. As 1.4’s architecture cannot easily reproduce this essentially broken behavior of 1.3 at the `select()` level right now, the use case now raises an informative error message instructing the user to use the `.ConcreteBase._concrete_discriminator_name` attribute to resolve the conflict. To assist with this configuration, `.ConcreteBase._concrete_discriminator_name` may be placed on the base class only where it will be automatically used by subclasses; previously this was not the case.

    References: [#6090](https://www.sqlalchemy.org/trac/ticket/6090)

### engine

*   **[engine] [bug] [regression]**

    Restored top level import for `sqlalchemy.engine.reflection`. This ensures that the base `Inspector` class is properly registered so that `inspect()` works for third party dialects that don’t otherwise import this package.

### sql

*   **[sql] [bug] [regression]**

    Fixed issue where using a `func` that includes dotted packagenames would fail to be cacheable by the SQL caching system due to a Python list of names that needed to be a tuple.

    References: [#6101](https://www.sqlalchemy.org/trac/ticket/6101)

*   **[sql] [bug] [regression]**

    Fixed regression in the `case()` construct, where the “dictionary” form of argument specification failed to work correctly if it were passed positionally, rather than as a “whens” keyword argument.

    References: [#6097](https://www.sqlalchemy.org/trac/ticket/6097)

### mypy

*   **[mypy] [bug]**

    Fixed issue in MyPy extension which crashed on detecting the type of a `Column` if the type were given with a module prefix like `sa.Integer()`.

    References: [#sqlalchemy/sqlalchemy2-stubs/2](https://www.sqlalchemy.org/trac/ticket/sqlalchemy/sqlalchemy2-stubs/2)

### postgresql

*   **[postgresql] [usecase]**

    Rename the column name used by a reflection query that used a reserved word in some postgresql compatible databases.

    References: [#6982](https://www.sqlalchemy.org/trac/ticket/6982)

## 1.4.1

Released: March 17, 2021

### orm

*   **[orm] [bug] [regression]**

    Fixed regression where producing a Core expression construct such as `select()` using ORM entities would eagerly configure the mappers, in an effort to maintain compatibility with the `Query` object which necessarily does this to support many backref-related legacy cases. However, core `select()` constructs are also used in mapper configurations and such, and to that degree this eager configuration is more of an inconvenience, so eager configure has been disabled for the `select()` and other Core constructs in the absence of ORM loading types of functions such as `Load`.

    The change maintains the behavior of `Query` so that backwards compatibility is maintained. However, when using a `select()` in conjunction with ORM entities, a “backref” that isn’t explicitly placed on one of the classes until mapper configure time won’t be available unless `configure_mappers()` or the newer `configure()` has been called elsewhere. Prefer using `relationship.back_populates` for more explicit relationship configuration which does not have the eager configure requirement.

    References: [#6066](https://www.sqlalchemy.org/trac/ticket/6066)

*   **[orm] [bug] [regression]**

    Fixed a critical regression in the relationship lazy loader where the SQL criteria used to fetch a related many-to-one object could go stale in relation to other memoized structures within the loader if the mapper had configuration changes, such as can occur when mappers are late configured or configured on demand, producing a comparison to None and returning no object. Huge thanks to Alan Hamlett for their help tracking this down late into the night.

    References: [#6055](https://www.sqlalchemy.org/trac/ticket/6055)

*   **[orm] [bug] [regression]**

    Fixed regression where the `Query.exists()` method would fail to create an expression if the entity list of the `Query` were an arbitrary SQL column expression.

    References: [#6076](https://www.sqlalchemy.org/trac/ticket/6076)

*   **[orm] [bug] [regression]**

    Fixed regression where calling upon `Query.count()` in conjunction with a loader option such as `joinedload()` would fail to ignore the loader option. This is a behavior that has always been very specific to the `Query.count()` method; an error is normally raised if a given `Query` has options that don’t apply to what it is returning.

    References: [#6052](https://www.sqlalchemy.org/trac/ticket/6052)

*   **[orm] [bug] [regression]**

    Fixed regression in `Session.identity_key()`, including that the method and related methods were not covered by any unit test as well as that the method contained a typo preventing it from functioning correctly.

    References: [#6067](https://www.sqlalchemy.org/trac/ticket/6067)

### orm declarative

*   **[orm] [declarative] [bug] [regression]**

    Fixed bug where user-mapped classes that contained an attribute named “registry” would cause conflicts with the new registry-based mapping system when using `DeclarativeMeta`. While the attribute remains something that can be set explicitly on a declarative base to be consumed by the metaclass, once located it is placed under a private class variable so it does not conflict with future subclasses that use the same name for other purposes.

    References: [#6054](https://www.sqlalchemy.org/trac/ticket/6054)

### engine

*   **[engine] [bug] [regression]**

    The Python `namedtuple()` has the behavior such that the names `count` and `index` will be served as tuple values if the named tuple includes those names; if they are absent, then their behavior as methods of `collections.abc.Sequence` is maintained. Therefore the `Row` and `LegacyRow` classes have been fixed so that they work in this same way, maintaining the expected behavior for database rows that have columns named “index” or “count”.

    References: [#6074](https://www.sqlalchemy.org/trac/ticket/6074)

### mssql

*   **[mssql] [bug] [regression]**

    Fixed regression where a new setinputsizes() API that’s available for pyodbc was enabled, which is apparently incompatible with pyodbc’s fast_executemany() mode in the absence of more accurate typing information, which as of yet is not fully implemented or tested. The pyodbc dialect and connector has been modified so that setinputsizes() is not used at all unless the parameter `use_setinputsizes` is passed to the dialect, e.g. via `create_engine()`, at which point its behavior can be customized using the `DialectEvents.do_setinputsizes()` hook.

    See also

    Setinputsizes Support

    References: [#6058](https://www.sqlalchemy.org/trac/ticket/6058)

### misc

*   **[bug] [regression]**

    Added back `items` and `values` to `ColumnCollection` class. The regression was introduced while adding support for duplicate columns in from clauses and selectable in ticket #4753.

    References: [#6068](https://www.sqlalchemy.org/trac/ticket/6068)

## 1.4.0

Released: March 15, 2021

### orm

*   **[orm] [bug]**

    Removed very old warning that states that passive_deletes is not intended for many-to-one relationships. While it is likely that in many cases placing this parameter on a many-to-one relationship is not what was intended, there are use cases where delete cascade may want to be disallowed following from such a relationship.

    This change is also **backported** to: 1.3.24

    References: [#5983](https://www.sqlalchemy.org/trac/ticket/5983)

*   **[orm] [bug]**

    Fixed issue where the process of joining two tables could fail if one of the tables had an unrelated, unresolvable foreign key constraint which would raise `NoReferenceError` within the join process, which nonetheless could be bypassed to allow the join to complete. The logic which tested the exception for significance within the process would make assumptions about the construct which would fail.

    This change is also **backported** to: 1.3.24

    References: [#5952](https://www.sqlalchemy.org/trac/ticket/5952)

*   **[orm] [bug]**

    Fixed issue where the `MutableComposite` construct could be placed into an invalid state when the parent object was already loaded, and then covered by a subsequent query, due to the composite properties’ refresh handler replacing the object with a new one not handled by the mutable extension.

    This change is also **backported** to: 1.3.24

    References: [#6001](https://www.sqlalchemy.org/trac/ticket/6001)

*   **[orm] [bug]**

    Fixed regression where the `relationship.query_class` parameter stopped being functional for “dynamic” relationships. The `AppenderQuery` remains dependent on the legacy `Query` class; users are encouraged to migrate from the use of “dynamic” relationships to using `with_parent()` instead.

    References: [#5981](https://www.sqlalchemy.org/trac/ticket/5981)

*   **[orm] [bug] [regression]**

    Fixed regression where `Query.join()` would produce no effect if the query itself as well as the join target were against a `Table` object, rather than a mapped class. This was part of a more systemic issue where the legacy ORM query compiler would not be correctly used from a `Query` if the statement produced had not ORM entities present within it.

    References: [#6003](https://www.sqlalchemy.org/trac/ticket/6003)

*   **[orm] [bug] [asyncio]**

    The API for `AsyncSession.delete()` is now an awaitable; this method cascades along relationships which must be loaded in a similar manner as the `AsyncSession.merge()` method.

    References: [#5998](https://www.sqlalchemy.org/trac/ticket/5998)

*   **[orm] [bug]**

    The unit of work process now turns off all “lazy=’raise’” behavior altogether when a flush is proceeding. While there are areas where the UOW is sometimes loading things that aren’t ultimately needed, the lazy=”raise” strategy is not helpful here as the user often does not have much control or visibility into the flush process.

    References: [#5984](https://www.sqlalchemy.org/trac/ticket/5984)

### engine

*   **[engine] [bug]**

    Fixed bug where the “schema_translate_map” feature failed to be taken into account for the use case of direct execution of `DefaultGenerator` objects such as sequences, which included the case where they were “pre-executed” in order to generate primary key values when implicit_returning was disabled.

    This change is also **backported** to: 1.3.24

    References: [#5929](https://www.sqlalchemy.org/trac/ticket/5929)

*   **[engine] [bug]**

    Improved engine logging to note ROLLBACK and COMMIT which is logged while the DBAPI driver is in AUTOCOMMIT mode. These ROLLBACK/COMMIT are library level and do not have any effect when AUTOCOMMIT is in effect, however it’s still worthwhile to log as these indicate where SQLAlchemy sees the “transaction” demarcation.

    References: [#6002](https://www.sqlalchemy.org/trac/ticket/6002)

*   **[engine] [bug] [regression]**

    Fixed a regression where the “reset agent” of the connection pool wasn’t really being utilized by the `Connection` when it were closed, and also leading to a double-rollback scenario that was somewhat wasteful. The newer architecture of the engine has been updated so that the connection pool “reset-on-return” logic will be skipped when the `Connection` explicitly closes out the transaction before returning the pool to the connection.

    References: [#6004](https://www.sqlalchemy.org/trac/ticket/6004)

### sql

*   **[sql] [change]**

    Altered the compilation for the `CTE` construct so that a string is returned representing the inner SELECT statement if the `CTE` is stringified directly, outside of the context of an enclosing SELECT; This is the same behavior of `FromClause.alias()` and `Select.subquery()`. Previously, a blank string would be returned as the CTE is normally placed above a SELECT after that SELECT has been generated, which is generally misleading when debugging.

*   **[sql] [bug]**

    Fixed bug where the “percent escaping” feature that occurs with dialects that use the “format” or “pyformat” bound parameter styles was not enabled for the `Operators.op()` and `custom_op` constructs, for custom operators that use percent signs. The percent sign will now be automatically doubled based on the paramstyle as necessary.

    References: [#6016](https://www.sqlalchemy.org/trac/ticket/6016)

*   **[sql] [bug] [regression]**

    Fixed regression where the “unsupported compilation error” for unknown datatypes would fail to raise correctly.

    References: [#5979](https://www.sqlalchemy.org/trac/ticket/5979)

*   **[sql] [bug] [regression]**

    Fixed regression where usage of the standalone `distinct()` used in the form of being directly SELECTed would fail to be locatable in the result set by column identity, which is how the ORM locates columns. While standalone `distinct()` is not oriented towards being directly SELECTed (use `select.distinct()` for a regular `SELECT DISTINCT..`) , it was usable to a limited extent in this way previously (but wouldn’t work in subqueries, for example). The column targeting for unary expressions such as “DISTINCT <col>” has been improved so that this case works again, and an additional improvement has been made so that usage of this form in a subquery at least generates valid SQL which was not the case previously.

    The change additionally enhances the ability to target elements in `row._mapping` based on SQL expression objects in ORM-enabled SELECT statements, including whether the statement was invoked by `connection.execute()` or `session.execute()`.

    References: [#6008](https://www.sqlalchemy.org/trac/ticket/6008)

### schema

*   **[schema] [bug] [sqlite]**

    Fixed issue where the CHECK constraint generated by `Boolean` or `Enum` would fail to render the naming convention correctly after the first compilation, due to an unintended change of state within the name given to the constraint. This issue was first introduced in 0.9 in the fix for issue #3067, and the fix revises the approach taken at that time which appears to have been more involved than what was needed.

    This change is also **backported** to: 1.3.24

    References: [#6007](https://www.sqlalchemy.org/trac/ticket/6007)

*   **[schema] [bug]**

    Repaired / implemented support for primary key constraint naming conventions that use column names/keys/etc as part of the convention. In particular, this includes that the `PrimaryKeyConstraint` object that’s automatically associated with a `Table` will update its name as new primary key `Column` objects are added to the table and then to the constraint. Internal failure modes related to this constraint construction process including no columns present, no name present or blank name present are now accommodated.

    This change is also **backported** to: 1.3.24

    References: [#5919](https://www.sqlalchemy.org/trac/ticket/5919)

*   **[schema] [bug]**

    Deprecated all schema-level `.copy()` methods and renamed to `_copy()`. These are not standard Python “copy()” methods as they typically rely upon being instantiated within particular contexts which are passed to the method as optional keyword arguments. The `Table.tometadata()` method is the public API that provides copying for `Table` objects.

    References: [#5953](https://www.sqlalchemy.org/trac/ticket/5953)

### mypy

*   **[mypy] [feature]**

    Rudimentary and experimental support for Mypy has been added in the form of a new plugin, which itself depends on new typing stubs for SQLAlchemy. The plugin allows declarative mappings in their standard form to both be compatible with Mypy as well as to provide typing support for mapped classes and instances.

    See also

    Mypy / Pep-484 Support for ORM Mappings

    References: [#4609](https://www.sqlalchemy.org/trac/ticket/4609)

### postgresql

*   **[postgresql] [usecase] [asyncio] [mysql]**

    Added an `asyncio.Lock()` within SQLAlchemy’s emulated DBAPI cursor, local to the connection, for the asyncpg and aiomysql dialects for the scope of the `cursor.execute()` and `cursor.executemany()` methods. The rationale is to prevent failures and corruption for the case where the connection is used in multiple awaitables at once.

    While this use case can also occur with threaded code and non-asyncio dialects, we anticipate this kind of use will be more common under asyncio, as the asyncio API is encouraging of such use. It’s definitely better to use a distinct connection per concurrent awaitable however as concurrency will not be achieved otherwise.

    For the asyncpg dialect, this is so that the space between the call to `prepare()` and `fetch()` is prevented from allowing concurrent executions on the connection from causing interface error exceptions, as well as preventing race conditions when starting a new transaction. Other PostgreSQL DBAPIs are threadsafe at the connection level so this intends to provide a similar behavior, outside the realm of server side cursors.

    For the aiomysql dialect, the mutex will provide safety such that the statement execution and the result set fetch, which are two distinct steps at the connection level, won’t get corrupted by concurrent executions on the same connection.

    References: [#5967](https://www.sqlalchemy.org/trac/ticket/5967)

*   **[postgresql] [bug]**

    Fixed issue where using `aggregate_order_by` would return ARRAY(NullType) under certain conditions, interfering with the ability of the result object to return data correctly.

    This change is also **backported** to: 1.3.24

    References: [#5989](https://www.sqlalchemy.org/trac/ticket/5989)

### mssql

*   **[mssql] [bug]**

    Fix a reflection error for MSSQL 2005 introduced by the reflection of filtered indexes.

    References: [#5919](https://www.sqlalchemy.org/trac/ticket/5919)

### misc

*   **[usecase] [ext]**

    Add new parameter `AutomapBase.prepare.reflection_options` to allow passing of `MetaData.reflect()` options like `only` or dialect-specific reflection options like `oracle_resolve_synonyms`.

    References: [#5942](https://www.sqlalchemy.org/trac/ticket/5942)

*   **[bug] [ext]**

    The `sqlalchemy.ext.mutable` extension now tracks the “parents” collection using the `InstanceState` associated with objects, rather than the object itself. The latter approach required that the object be hashable so that it can be inside of a `WeakKeyDictionary`, which goes against the behavioral contract of the ORM overall which is that ORM mapped objects do not need to provide any particular kind of `__hash__()` method and that unhashable objects are supported.

    References: [#6020](https://www.sqlalchemy.org/trac/ticket/6020)

## 1.4.0b3

Released: February 15, 2021

### orm

*   **[orm] [feature]**

    The ORM used in 2.0 style can now return ORM objects from the rows returned by an UPDATE..RETURNING or INSERT..RETURNING statement, by supplying the construct to `Select.from_statement()` in an ORM context.

    See also

    Using INSERT, UPDATE and ON CONFLICT (i.e. upsert) to return ORM Objects

*   **[orm] [bug]**

    Fixed issue in new 1.4/2.0 style ORM queries where a statement-level label style would not be preserved in the keys used by result rows; this has been applied to all combinations of Core/ORM columns / session vs. connection etc. so that the linkage from statement to result row is the same in all cases. As part of this change, the labeling of column expressions in rows has been improved to retain the original name of the ORM attribute even if used in a subquery.

    References: [#5933](https://www.sqlalchemy.org/trac/ticket/5933)

### engine

*   **[engine] [bug] [postgresql]**

    Continued with the improvement made as part of [#5653](https://www.sqlalchemy.org/trac/ticket/5653) to further support bound parameter names, including those generated against column names, for names that include colons, parenthesis, and question marks, as well as improved test support, so that bound parameter names even if they are auto-derived from column names should have no problem including for parenthesis in psycopg2’s “pyformat” style.

    As part of this change, the format used by the asyncpg DBAPI adapter (which is local to SQLAlchemy’s asyncpg dialect) has been changed from using “qmark” paramstyle to “format”, as there is a standard and internally supported SQL string escaping style for names that use percent signs with “format” style (i.e. to double percent signs), as opposed to names that use question marks with “qmark” style (where an escaping system is not defined by pep-249 or Python).

    See also

    psycopg2 dialect no longer has limitations regarding bound parameter names

    References: [#5941](https://www.sqlalchemy.org/trac/ticket/5941)

### sql

*   **[sql] [usecase] [postgresql] [sqlite]**

    Enhance `set_` keyword of `OnConflictDoUpdate` to accept a `ColumnCollection`, such as the `.c.` collection from a `Selectable`, or the `.excluded` contextual object.

    References: [#5939](https://www.sqlalchemy.org/trac/ticket/5939)

*   **[sql] [bug]**

    Fixed bug where the “cartesian product” assertion was not correctly accommodating for joins between tables that relied upon the use of LATERAL to connect from a subquery to another subquery in the enclosing context.

    References: [#5924](https://www.sqlalchemy.org/trac/ticket/5924)

*   **[sql] [bug]**

    Fixed 1.4 regression where the `Function.in_()` method was not covered by tests and failed to function properly in all cases.

    References: [#5934](https://www.sqlalchemy.org/trac/ticket/5934)

*   **[sql] [bug]**

    Fixed regression where use of an arbitrary iterable with the `select()` function was not working, outside of plain lists. The forwards/backwards compatibility logic here now checks for a wider range of incoming “iterable” types including that a `.c` collection from a selectable can be passed directly. Pull request compliments of Oliver Rice.

    References: [#5935](https://www.sqlalchemy.org/trac/ticket/5935)

## 1.4.0b2

Released: February 3, 2021

### general

*   **[general] [bug]**

    Fixed a SQLite source file that had non-ascii characters inside of its docstring without a source encoding, introduced within the “INSERT..ON CONFLICT” feature, which would cause failures under Python 2.

### platform

*   **[platform] [performance]**

    Adjusted some elements related to internal class production at import time which added significant latency to the time spent to import the library vs. that of 1.3\. The time is now about 20-30% slower than 1.3 instead of 200%.

    References: [#5681](https://www.sqlalchemy.org/trac/ticket/5681)

### orm

*   **[orm] [usecase]**

    Added `ORMExecuteState.bind_mapper` and `ORMExecuteState.all_mappers` accessors to `ORMExecuteState` event object, so that handlers can respond to the target mapper and/or mapped class or classes involved in an ORM statement execution.

*   **[orm] [usecase] [asyncio]**

    Added `AsyncSession.scalar()`, `AsyncSession.get()` as well as support for `sessionmaker.begin()` to work as an async context manager with `AsyncSession`. Also added `AsyncSession.in_transaction()` accessor.

    References: [#5796](https://www.sqlalchemy.org/trac/ticket/5796), [#5797](https://www.sqlalchemy.org/trac/ticket/5797), [#5802](https://www.sqlalchemy.org/trac/ticket/5802)

*   **[orm] [changed]**

    Mapper “configuration”, which occurs within the `configure_mappers()` function, is now organized to be on a per-registry basis. This allows for example the mappers within a certain declarative base to be configured, but not those of another base that is also present in memory. The goal is to provide a means of reducing application startup time by only running the “configure” process for sets of mappers that are needed. This also adds the `registry.configure()` method that will run configure for the mappers local in a particular registry only.

    References: [#5897](https://www.sqlalchemy.org/trac/ticket/5897)

*   **[orm] [bug]**

    Added a comprehensive check and an informative error message for the case where a mapped class, or a string mapped class name, is passed to `relationship.secondary`. This is an extremely common error which warrants a clear message.

    Additionally, added a new rule to the class registry resolution such that with regards to the `relationship.secondary` parameter, if a mapped class and its table are of the identical string name, the `Table` will be favored when resolving this parameter. In all other cases, the class continues to be favored if a class and table share the identical name.

    This change is also **backported** to: 1.3.21

    References: [#5774](https://www.sqlalchemy.org/trac/ticket/5774)

*   **[orm] [bug]**

    Fixed bug involving the `restore_load_context` option of ORM events such as `InstanceEvents.load()` such that the flag would not be carried along to subclasses which were mapped after the event handler were first established.

    This change is also **backported** to: 1.3.21

    References: [#5737](https://www.sqlalchemy.org/trac/ticket/5737)

*   **[orm] [bug] [regression]**

    Fixed issue in new `Session` similar to that of the `Connection` where the new “autobegin” logic could be tripped into a re-entrant (recursive) state if SQL were executed within the `SessionEvents.after_transaction_create()` event hook.

    References: [#5845](https://www.sqlalchemy.org/trac/ticket/5845)

*   **[orm] [bug] [unitofwork]**

    Improved the unit of work topological sorting system such that the toplogical sort is now deterministic based on the sorting of the input set, which itself is now sorted at the level of mappers, so that the same inputs of affected mappers should produce the same output every time, among mappers / tables that don’t have any dependency on each other. This further reduces the chance of deadlocks as can be observed in a flush that UPDATEs among multiple, unrelated tables such that row locks are generated.

    References: [#5735](https://www.sqlalchemy.org/trac/ticket/5735)

*   **[orm] [bug]**

    Fixed regression where the `Bundle.single_entity` flag would take effect for a `Bundle` even though it were not set. Additionally, this flag is legacy as it only makes sense for the `Query` object and not 2.0 style execution. a deprecation warning is emitted when used with new-style execution.

    References: [#5702](https://www.sqlalchemy.org/trac/ticket/5702)

*   **[orm] [bug]**

    Fixed regression where creating an `aliased` construct against a plain selectable and including a name would raise an assertionerror.

    References: [#5750](https://www.sqlalchemy.org/trac/ticket/5750)

*   **[orm] [bug]**

    Related to the fixes for the lambda criteria system within Core, within the ORM implemented a variety of fixes for the `with_loader_criteria()` feature as well as the `SessionEvents.do_orm_execute()` event handler that is often used in conjunction [ticket:5760]:

    *   fixed issue where `with_loader_criteria()` function would fail if the given entity or base included non-mapped mixins in its descending class hierarchy [ticket:5766]

    *   The `with_loader_criteria()` feature is now unconditionally disabled for the case of ORM “refresh” operations, including loads of deferred or expired column attributes as well as for explicit operations like `Session.refresh()`. These loads are necessarily based on primary key identity where additional WHERE criteria is never appropriate. [ticket:5762]

    *   Added new attribute `ORMExecuteState.is_column_load` to indicate that a `SessionEvents.do_orm_execute()` handler that a particular operation is a primary-key-directed column attribute load, where additional criteria should not be added. The `with_loader_criteria()` function as above ignores these in any case now. [ticket:5761]

    *   Fixed issue where the `ORMExecuteState.is_relationship_load` attribute would not be set correctly for many lazy loads as well as all selectinloads. The flag is essential in order to test if options should be added to statements or if they would already have been propagated via relationship loads. [ticket:5764]

    References: [#5760](https://www.sqlalchemy.org/trac/ticket/5760), [#5761](https://www.sqlalchemy.org/trac/ticket/5761), [#5762](https://www.sqlalchemy.org/trac/ticket/5762), [#5764](https://www.sqlalchemy.org/trac/ticket/5764), [#5766](https://www.sqlalchemy.org/trac/ticket/5766)

*   **[orm] [bug]**

    Fixed 1.4 regression where the use of `Query.having()` in conjunction with queries with internally adapted SQL elements (common in inheritance scenarios) would fail due to an incorrect function call. Pull request courtesy esoh.

    References: [#5781](https://www.sqlalchemy.org/trac/ticket/5781)

*   **[orm] [bug]**

    Fixed an issue where the API to create a custom executable SQL construct using the `sqlalchemy.ext.compiles` extension according to the documentation that’s been up for many years would no longer function if only `Executable, ClauseElement` were used as the base classes, additional classes were needed if wanting to use `Session.execute()`. This has been resolved so that those extra classes aren’t needed.

*   **[orm] [bug] [regression]**

    Fixed ORM unit of work regression where an errant “assert primary_key” statement interferes with primary key generation sequences that don’t actually consider the columns in the table to use a real primary key constraint, instead using `Mapper.primary_key` to establish certain columns as “primary”.

    References: [#5867](https://www.sqlalchemy.org/trac/ticket/5867)

### orm declarative

*   **[orm] [declarative] [feature]**

    Added an alternate resolution scheme to Declarative that will extract the SQLAlchemy column or mapped property from the “metadata” dictionary of a dataclasses.Field object. This allows full declarative mappings to be combined with dataclass fields.

    See also

    Mapping pre-existing dataclasses using Declarative-style fields

    References: [#5745](https://www.sqlalchemy.org/trac/ticket/5745)

### engine

*   **[engine] [feature]**

    Dialect-specific constructs such as `Insert.on_conflict_do_update()` can now stringify in-place without the need to specify an explicit dialect object. The constructs, when called upon for `str()`, `print()`, etc. now have internal direction to call upon their appropriate dialect rather than the “default”dialect which doesn’t know how to stringify these. The approach is also adapted to generic schema-level create/drop such as `AddConstraint`, which will adapt its stringify dialect to one indicated by the element within it, such as the `ExcludeConstraint` object.

*   **[engine] [feature]**

    Added new execution option `Connection.execution_options.logging_token`. This option will add an additional per-message token to log messages generated by the `Connection` as it executes statements. This token is not part of the logger name itself (that part can be affected using the existing `create_engine.logging_name` parameter), so is appropriate for ad-hoc connection use without the side effect of creating many new loggers. The option can be set at the level of `Connection` or `Engine`.

    See also

    Setting Per-Connection / Sub-Engine Tokens

    References: [#5911](https://www.sqlalchemy.org/trac/ticket/5911)

*   **[engine] [bug] [sqlite]**

    Fixed bug in the 2.0 “future” version of `Engine` where emitting SQL during the `EngineEvents.begin()` event hook would cause a re-entrant (recursive) condition due to autobegin, affecting among other things the recipe documented for SQLite to allow for savepoints and serializable isolation support.

    References: [#5845](https://www.sqlalchemy.org/trac/ticket/5845)

*   **[engine] [bug] [oracle] [postgresql]**

    Adjusted the “setinputsizes” logic relied upon by the cx_Oracle, asyncpg and pg8000 dialects to support a `TypeDecorator` that includes an override the `TypeDecorator.get_dbapi_type()` method.

*   **[engine] [bug]**

    Added the “future” keyword to the list of words that are known by the `engine_from_config()` function, so that the values “true” and “false” may be configured as “boolean” values when using a key such as `sqlalchemy.future = true` or `sqlalchemy.future = false`.

### sql

*   **[sql] [feature]**

    Implemented support for “table valued functions” along with additional syntaxes supported by PostgreSQL, one of the most commonly requested features. Table valued functions are SQL functions that return lists of values or rows, and are prevalent in PostgreSQL in the area of JSON functions, where the “table value” is commonly referred to as the “record” datatype. Table valued functions are also supported by Oracle and SQL Server.

    Features added include:

    *   the `FunctionElement.table_valued()` modifier that creates a table-like selectable object from a SQL function

    *   A `TableValuedAlias` construct that renders a SQL function as a named table

    *   Support for PostgreSQL’s special “derived column” syntax that includes column names and sometimes datatypes, such as for the `json_to_recordset` function, using the `TableValuedAlias.render_derived()` method.

    *   Support for PostgreSQL’s “WITH ORDINALITY” construct using the `FunctionElement.table_valued.with_ordinality` parameter

    *   Support for selection FROM a SQL function as column-valued scalar, a syntax supported by PostgreSQL and Oracle, via the `FunctionElement.column_valued()` method

    *   A way to SELECT a single column from a table-valued expression without using a FROM clause via the `FunctionElement.scalar_table_valued()` method.

    See also

    Table-Valued Functions - in the SQLAlchemy Unified Tutorial

    References: [#3566](https://www.sqlalchemy.org/trac/ticket/3566)

*   **[sql] [usecase]**

    Multiple calls to “returning”, e.g. `Insert.returning()`, may now be chained to add new columns to the RETURNING clause.

    References: [#5695](https://www.sqlalchemy.org/trac/ticket/5695)

*   **[sql] [usecase]**

    Added `Select.outerjoin_from()` method to complement `Select.join_from()`.

*   **[sql] [usecase]**

    Adjusted the “literal_binds” feature of `Compiler` to render NULL for a bound parameter that has `None` as the value, either explicitly passed or omitted. The previous error message “bind parameter without a renderable value” is removed, and a missing or `None` value will now render NULL in all cases. Previously, rendering of NULL was starting to happen for DML statements due to internal refactorings, but was not explicitly part of test coverage, which it now is.

    While no error is raised, when the context is within that of a column comparison, and the operator is not “IS”/”IS NOT”, a warning is emitted that this is not generally useful from a SQL perspective.

    References: [#5888](https://www.sqlalchemy.org/trac/ticket/5888)

*   **[sql] [bug]**

    Fixed issue in new `Select.join()` method where chaining from the current JOIN wasn’t looking at the right state, causing an expression like “FROM a JOIN b <onclause>, b JOIN c <onclause>” rather than “FROM a JOIN b <onclause> JOIN c <onclause>”.

    References: [#5858](https://www.sqlalchemy.org/trac/ticket/5858)

*   **[sql] [bug]**

    Deprecation warnings are emitted under “SQLALCHEMY_WARN_20” mode when passing a plain string to `Session.execute()`.

    References: [#5754](https://www.sqlalchemy.org/trac/ticket/5754)

*   **[sql] [bug] [orm]**

    A wide variety of fixes to the “lambda SQL” feature introduced at Using Lambdas to add significant speed gains to statement production have been implemented based on user feedback, with an emphasis on its use within the `with_loader_criteria()` feature where it is most prominently used [ticket:5760]:

    *   Fixed the issue where boolean True/False values, which were referred to in the closure variables of the lambda, would cause failures. [ticket:5763]

    *   Repaired a non-working detection for Python functions embedded in the lambda that produce bound values; this case is likely not supportable so raises an informative error, where the function should be invoked outside the lambda itself. New documentation has been added to further detail this behavior. [ticket:5770]

    *   The lambda system by default now rejects the use of non-SQL elements within the closure variables of the lambda entirely, where the error suggests the two options of either explicitly ignoring closure variables that are not SQL parameters, or specifying a specific set of values to be considered as part of the cache key based on hash value. This critically prevents the lambda system from assuming that arbitrary objects within the lambda’s closure are appropriate for caching while also refusing to ignore them by default, preventing the case where their state might not be constant and have an impact on the SQL construct produced. The error message is comprehensive and new documentation has been added to further detail this behavior. [ticket:5765]

    *   Fixed support for the edge case where an `in_()` expression against a list of SQL elements, such as `literal()` objects, would fail to be accommodated correctly. [ticket:5768]

    References: [#5760](https://www.sqlalchemy.org/trac/ticket/5760), [#5763](https://www.sqlalchemy.org/trac/ticket/5763), [#5765](https://www.sqlalchemy.org/trac/ticket/5765), [#5768](https://www.sqlalchemy.org/trac/ticket/5768), [#5770](https://www.sqlalchemy.org/trac/ticket/5770)

*   **[sql] [bug] [mysql] [postgresql] [sqlite]**

    An informative error message is now raised for a selected set of DML methods (currently all part of `Insert` constructs) if they are called a second time, which would implicitly cancel out the previous setting. The methods altered include: `on_conflict_do_update`, `on_conflict_do_nothing` (SQLite), `on_conflict_do_update`, `on_conflict_do_nothing` (PostgreSQL), `on_duplicate_key_update` (MySQL)

    References: [#5169](https://www.sqlalchemy.org/trac/ticket/5169)

*   **[sql] [bug]**

    Fixed issue in new `Values` construct where passing tuples of objects would fall back to per-value type detection rather than making use of the `Column` objects passed directly to `Values` that tells SQLAlchemy what the expected type is. This would lead to issues for objects such as enumerations and numpy strings that are not actually necessary since the expected type is given.

    References: [#5785](https://www.sqlalchemy.org/trac/ticket/5785)

*   **[sql] [bug]**

    Fixed issue where a `RemovedIn20Warning` would erroneously emit when the `.bind` attribute were accessed internally on objects, particularly when stringifying a SQL construct.

    References: [#5717](https://www.sqlalchemy.org/trac/ticket/5717)

*   **[sql] [bug]**

    Properly render `cycle=False` and `order=False` as `NO CYCLE` and `NO ORDER` in `Sequence` and `Identity` objects.

    References: [#5722](https://www.sqlalchemy.org/trac/ticket/5722)

*   **[sql]**

    Replace `Query.with_labels()` and `GenerativeSelect.apply_labels()` with explicit getters and setters `GenerativeSelect.get_label_style()` and `GenerativeSelect.set_label_style()` to accommodate the three supported label styles: `LABEL_STYLE_DISAMBIGUATE_ONLY`, `LABEL_STYLE_TABLENAME_PLUS_COL`, and `LABEL_STYLE_NONE`.

    In addition, for Core and “future style” ORM queries, `LABEL_STYLE_DISAMBIGUATE_ONLY` is now the default label style. This style differs from the existing “no labels” style in that labeling is applied in the case of column name conflicts; with `LABEL_STYLE_NONE`, a duplicate column name is not accessible via name in any case.

    For cases where labeling is significant, namely that the `.c` collection of a subquery is able to refer to all columns unambiguously, the behavior of `LABEL_STYLE_DISAMBIGUATE_ONLY` is now sufficient for all SQLAlchemy features across Core and ORM which involve this behavior. Result set rows since SQLAlchemy 1.0 are usually aligned with column constructs positionally.

    For legacy ORM queries using `Query`, the table-plus-column names labeling style applied by `LABEL_STYLE_TABLENAME_PLUS_COL` continues to be used so that existing test suites and logging facilities see no change in behavior by default.

    References: [#4757](https://www.sqlalchemy.org/trac/ticket/4757)

### schema

*   **[schema] [feature]**

    Added `TypeEngine.as_generic()` to map dialect-specific types, such as `sqlalchemy.dialects.mysql.INTEGER`, with the “best match” generic SQLAlchemy type, in this case `Integer`. Pull request courtesy Andrew Hannigan.

    See also

    Reflecting with Database-Agnostic Types - example usage

    References: [#5659](https://www.sqlalchemy.org/trac/ticket/5659)

*   **[schema] [usecase]**

    The `DDLEvents.column_reflect()` event may now be applied to a `MetaData` object where it will take effect for the `Table` objects local to that collection.

    See also

    `DDLEvents.column_reflect()`

    Automating Column Naming Schemes from Reflected Tables - in the ORM mapping documentation

    Intercepting Column Definitions - in the Automap documentation

    References: [#5712](https://www.sqlalchemy.org/trac/ticket/5712)

*   **[schema] [usecase]**

    Added parameters `CreateTable.if_not_exists`, `CreateIndex.if_not_exists`, `DropTable.if_exists` and `DropIndex.if_exists` to the `CreateTable`, `DropTable`, `CreateIndex` and `DropIndex` constructs which result in “IF NOT EXISTS” / “IF EXISTS” DDL being added to the CREATE/DROP. These phrases are not accepted by all databases and the operation will fail on a database that does not support it as there is no similarly compatible fallback within the scope of a single DDL statement. Pull request courtesy Ramon Williams.

    References: [#2843](https://www.sqlalchemy.org/trac/ticket/2843)

*   **[schema] [changed]**

    Altered the behavior of the `Identity` construct such that when applied to a `Column`, it will automatically imply that the value of `Column.nullable` should default to `False`, in a similar manner as when the `Column.primary_key` parameter is set to `True`. This matches the default behavior of all supporting databases where `IDENTITY` implies `NOT NULL`. The PostgreSQL backend is the only one that supports adding `NULL` to an `IDENTITY` column, which is here supported by passing a `True` value for the `Column.nullable` parameter at the same time.

    References: [#5775](https://www.sqlalchemy.org/trac/ticket/5775)

### asyncio

*   **[asyncio] [usecase]**

    The `AsyncEngine`, `AsyncConnection` and `AsyncTransaction` objects may be compared using Python `==` or `!=`, which will compare the two given objects based on the “sync” object they are proxying towards. This is useful as there are cases particularly for `AsyncTransaction` where multiple instances of `AsyncTransaction` can be proxying towards the same sync `Transaction`, and are actually equivalent. The `AsyncConnection.get_transaction()` method will currently return a new proxying `AsyncTransaction` each time as the `AsyncTransaction` is not otherwise statefully associated with its originating `AsyncConnection`.

*   **[asyncio] [bug]**

    Adjusted the greenlet integration, which provides support for Python asyncio in SQLAlchemy, to accommodate for the handling of Python `contextvars` (introduced in Python 3.7) for `greenlet` versions greater than 0.4.17. Greenlet version 0.4.17 added automatic handling of contextvars in a backwards-incompatible way; we’ve coordinated with the greenlet authors to add a preferred API for this in versions subsequent to 0.4.17 which is now supported by SQLAlchemy’s greenlet integration. For greenlet versions prior to 0.4.17 no behavioral change is needed, version 0.4.17 itself is blocked from the dependencies.

    References: [#5615](https://www.sqlalchemy.org/trac/ticket/5615)

*   **[asyncio] [bug]**

    Implemented “connection-binding” for `AsyncSession`, the ability to pass an `AsyncConnection` to create an `AsyncSession`. Previously, this use case was not implemented and would use the associated engine when the connection were passed. This fixes the issue where the “join a session to an external transaction” use case would not work correctly for the `AsyncSession`. Additionally, added methods `AsyncConnection.in_transaction()`, `AsyncConnection.in_nested_transaction()`, `AsyncConnection.get_transaction()`, `AsyncConnection.get_nested_transaction()` and `AsyncConnection.info` attribute.

    References: [#5811](https://www.sqlalchemy.org/trac/ticket/5811)

*   **[asyncio] [bug]**

    Fixed bug in asyncio connection pool where `asyncio.TimeoutError` would be raised rather than `TimeoutError`. Also repaired the `create_engine.pool_timeout` parameter set to zero when using the async engine, which previously would ignore the timeout and block rather than timing out immediately as is the behavior with regular `QueuePool`.

    References: [#5827](https://www.sqlalchemy.org/trac/ticket/5827)

*   **[asyncio] [bug] [pool]**

    When using an asyncio engine, the connection pool will now detach and discard a pooled connection that is was not explicitly closed/returned to the pool when its tracking object is garbage collected, emitting a warning that the connection was not properly closed. As this operation occurs during Python gc finalizers, it’s not safe to run any IO operations upon the connection including transaction rollback or connection close as this will often be outside of the event loop.

    The `AsyncAdaptedQueue` used by default on async dpapis should instantiate a queue only when it’s first used to avoid binding it to a possibly wrong event loop.

    References: [#5823](https://www.sqlalchemy.org/trac/ticket/5823)

*   **[asyncio]**

    The SQLAlchemy async mode now detects and raises an informative error when an non asyncio compatible DBAPI is used. Using a standard `DBAPI` with async SQLAlchemy will cause it to block like any sync call, interrupting the executing asyncio loop.

### postgresql

*   **[postgresql] [usecase]**

    Added new parameter `ExcludeConstraint.ops` to the `ExcludeConstraint` object, to support operator class specification with this constraint. Pull request courtesy Alon Menczer.

    This change is also **backported** to: 1.3.21

    References: [#5604](https://www.sqlalchemy.org/trac/ticket/5604)

*   **[postgresql] [usecase]**

    Added a read/write `.autocommit` attribute to the DBAPI-adaptation layer for the asyncpg dialect. This so that when working with DBAPI-specific schemes that need to use “autocommit” directly with the DBAPI connection, the same `.autocommit` attribute which works with both psycopg2 as well as pg8000 is available.

*   **[postgresql] [changed]**

    Fixed issue where the psycopg2 dialect would silently pass the `use_native_unicode=False` flag without actually having any effect under Python 3, as the psycopg2 DBAPI uses Unicode unconditionally under Python 3\. This usage now raises an `ArgumentError` when used under Python 3\. Added test support for Python 2.

*   **[postgresql] [performance]**

    Enhanced the performance of the asyncpg dialect by caching the asyncpg PreparedStatement objects on a per-connection basis. For a test case that makes use of the same statement on a set of pooled connections this appears to grant a 10-20% speed improvement. The cache size is adjustable and may also be disabled.

    See also

    Prepared Statement Cache

*   **[postgresql] [bug] [mysql]**

    Fixed regression introduced in 1.3.2 for the PostgreSQL dialect, also copied out to the MySQL dialect’s feature in 1.3.18, where usage of a non `Table` construct such as `text()` as the argument to `Select.with_for_update.of` would fail to be accommodated correctly within the PostgreSQL or MySQL compilers.

    This change is also **backported** to: 1.3.21

    References: [#5729](https://www.sqlalchemy.org/trac/ticket/5729)

*   **[postgresql] [bug]**

    Fixed a small regression where the query for “show standard_conforming_strings” upon initialization would be emitted even if the server version info were detected as less than version 8.2, previously it would only occur for server version 8.2 or greater. The query fails on Amazon Redshift which reports a PG server version older than this value.

    References: [#5698](https://www.sqlalchemy.org/trac/ticket/5698)

*   **[postgresql] [bug]**

    Established support for `Column` objects as well as ORM instrumented attributes as keys in the `set_` dictionary passed to the `Insert.on_conflict_do_update()` and `Insert.on_conflict_do_update()` methods, which match to the `Column` objects in the `.c` collection of the target `Table`. Previously, only string column names were expected; a column expression would be assumed to be an out-of-table expression that would render fully along with a warning.

    References: [#5722](https://www.sqlalchemy.org/trac/ticket/5722)

*   **[postgresql] [bug] [asyncio]**

    Fixed bug in asyncpg dialect where a failure during a “commit” or less likely a “rollback” should cancel the entire transaction; it’s no longer possible to emit rollback. Previously the connection would continue to await a rollback that could not succeed as asyncpg would reject it.

    References: [#5824](https://www.sqlalchemy.org/trac/ticket/5824)

### mysql

*   **[mysql] [feature]**

    Added support for the aiomysql driver when using the asyncio SQLAlchemy extension.

    See also

    aiomysql

    References: [#5747](https://www.sqlalchemy.org/trac/ticket/5747)

*   **[mysql] [bug] [reflection]**

    Fixed issue where reflecting a server default on MariaDB only that contained a decimal point in the value would fail to be reflected correctly, leading towards a reflected table that lacked any server default.

    This change is also **backported** to: 1.3.21

    References: [#5744](https://www.sqlalchemy.org/trac/ticket/5744)

### sqlite

*   **[sqlite] [usecase]**

    Implemented INSERT… ON CONFLICT clause for SQLite. Pull request courtesy Ramon Williams.

    See also

    INSERT…ON CONFLICT (Upsert)

    References: [#4010](https://www.sqlalchemy.org/trac/ticket/4010)

*   **[sqlite] [bug]**

    Use python `re.search()` instead of `re.match()` as the operation used by the `Column.regexp_match()` method when using sqlite. This matches the behavior of regular expressions on other databases as well as that of well-known SQLite plugins.

    References: [#5699](https://www.sqlalchemy.org/trac/ticket/5699)

### mssql

*   **[mssql] [bug] [datatypes] [mysql]**

    Decimal accuracy and behavior has been improved when extracting floating point and/or decimal values from JSON strings using the `Comparator.as_float()` method, when the numeric value inside of the JSON string has many significant digits; previously, MySQL backends would truncate values with many significant digits and SQL Server backends would raise an exception due to a DECIMAL cast with insufficient significant digits. Both backends now use a FLOAT-compatible approach that does not hardcode significant digits for floating point values. For precision numerics, a new method `Comparator.as_numeric()` has been added which accepts arguments for precision and scale, and will return values as Python `Decimal` objects with no floating point conversion assuming the DBAPI supports it (all but pysqlite).

    References: [#5788](https://www.sqlalchemy.org/trac/ticket/5788)

### oracle

*   **[oracle] [bug]**

    Fixed regression which occurred due to [#5755](https://www.sqlalchemy.org/trac/ticket/5755) which implemented isolation level support for Oracle. It has been reported that many Oracle accounts don’t actually have permission to query the `v$transaction` view so this feature has been altered to gracefully fallback when it fails upon database connect, where the dialect will assume “READ COMMITTED” is the default isolation level as was the case prior to SQLAlchemy 1.3.21. However, explicit use of the `Connection.get_isolation_level()` method must now necessarily raise an exception, as Oracle databases with this restriction explicitly disallow the user from reading the current isolation level.

    This change is also **backported** to: 1.3.22

    References: [#5784](https://www.sqlalchemy.org/trac/ticket/5784)

*   **[oracle] [bug]**

    Oracle two-phase transactions at a rudimentary level are now no longer deprecated. After receiving support from cx_Oracle devs we can provide for basic xid + begin/prepare support with some limitations, which will work more fully in an upcoming release of cx_Oracle. Two phase “recovery” is not currently supported.

    References: [#5884](https://www.sqlalchemy.org/trac/ticket/5884)

*   **[oracle] [bug]**

    The Oracle dialect now uses `select sys_context( 'userenv', 'current_schema' ) from dual` to get the default schema name, rather than `SELECT USER FROM DUAL`, to accommodate for changes to the session-local schema name under Oracle.

    References: [#5716](https://www.sqlalchemy.org/trac/ticket/5716)

### tests

*   **[tests] [usecase] [pool]**

    Improve documentation and add test for sub-second pool timeouts. Pull request courtesy Jordan Pittier.

    References: [#5582](https://www.sqlalchemy.org/trac/ticket/5582)

### misc

*   **[usecase] [pool]**

    The internal mechanics of the engine connection routine has been altered such that it’s now guaranteed that a user-defined event handler for the `PoolEvents.connect()` handler, when established using `insert=True`, will allow an event handler to run that is definitely invoked **before** any dialect-specific initialization starts up, most notably when it does things like detect default schema name. Previously, this would occur in most cases but not unconditionally. A new example is added to the schema documentation illustrating how to establish the “default schema name” within an on-connect event.

    References: [#5497](https://www.sqlalchemy.org/trac/ticket/5497), [#5708](https://www.sqlalchemy.org/trac/ticket/5708)

*   **[bug] [reflection]**

    Fixed bug where the now-deprecated `autoload` parameter was being called internally within the reflection routines when a related table were reflected.

    References: [#5684](https://www.sqlalchemy.org/trac/ticket/5684)

*   **[bug] [pool]**

    Fixed regression where a connection pool event specified with a keyword, most notably `insert=True`, would be lost when the event were set up. This would prevent startup events that need to fire before dialect-level events from working correctly.

    References: [#5708](https://www.sqlalchemy.org/trac/ticket/5708)

*   **[bug] [pool] [pypy]**

    Fixed issue where connection pool would not return connections to the pool or otherwise be finalized upon garbage collection under pypy if the checked out connection fell out of scope without being closed. This is a long standing issue due to pypy’s difference in GC behavior that does not call weakref finalizers if they are relative to another object that is also being garbage collected. A strong reference to the related record is now maintained so that the weakref has a strong-referenced “base” to trigger off of.

    References: [#5842](https://www.sqlalchemy.org/trac/ticket/5842)

## 1.4.0b1

Released: November 2, 2020

### general

*   **[general] [change]**

    ”python setup.py test” is no longer a test runner, as this is deprecated by Pypa. Please use “tox” with no arguments for a basic test run.

    References: [#4789](https://www.sqlalchemy.org/trac/ticket/4789)

*   **[general] [bug]**

    Refactored the internal conventions used to cross-import modules that have mutual dependencies between them, such that the inspected arguments of functions and methods are no longer modified. This allows tools like pylint, Pycharm, other code linters, as well as hypothetical pep-484 implementations added in the future to function correctly as they no longer see missing arguments to function calls. The new approach is also simpler and more performant.

    See also

    Repaired internal importing conventions such that code linters may work correctly

    References: [#4656](https://www.sqlalchemy.org/trac/ticket/4656), [#4689](https://www.sqlalchemy.org/trac/ticket/4689)

### platform

*   **[platform] [change]**

    The `importlib_metadata` library is used to scan for setuptools entrypoints rather than pkg_resources. as importlib_metadata is a small library that is included as of Python 3.8, the compatibility library is installed as a dependency for Python versions older than 3.8.

    References: [#5400](https://www.sqlalchemy.org/trac/ticket/5400)

*   **[platform] [change]**

    Installation has been modernized to use setup.cfg for most package metadata.

    References: [#5404](https://www.sqlalchemy.org/trac/ticket/5404)

*   **[platform] [removed]**

    Dropped support for python 3.4 and 3.5 that has reached EOL. SQLAlchemy 1.4 series requires python 2.7 or 3.6+.

    See also

    Python 3.6 is the minimum Python 3 version; Python 2.7 still supported

    References: [#5634](https://www.sqlalchemy.org/trac/ticket/5634)

*   **[platform] [removed]**

    Removed all dialect code related to support for Jython and zxJDBC. Jython has not been supported by SQLAlchemy for many years and it is not expected that the current zxJDBC code is at all functional; for the moment it just takes up space and adds confusion by showing up in documentation. At the moment, it appears that Jython has achieved Python 2.7 support in its releases but not Python 3\. If Jython were to be supported again, the form it should take is against the Python 3 version of Jython, and the various zxJDBC stubs for various backends should be implemented as a third party dialect.

    References: [#5094](https://www.sqlalchemy.org/trac/ticket/5094)

### orm

*   **[orm] [feature]**

    The ORM can now generate queries previously only available when using `Query` using the `select()` construct directly. A new system by which ORM “plugins” may establish themselves within a Core `Select` allow the majority of query building logic previously inside of `Query` to now take place within a compilation-level extension for `Select`. Similar changes have been made for the `Update` and `Delete` constructs as well. The constructs when invoked using `Session.execute()` now do ORM-related work within the method. For `Select`, the `Result` object returned now contains ORM-level entities and results.

    See also

    ORM Query is internally unified with select, update, delete; 2.0 style execution available

    References: [#5159](https://www.sqlalchemy.org/trac/ticket/5159)

*   **[orm] [feature]**

    Added the ability to add arbitrary criteria to the ON clause generated by a relationship attribute in a query, which applies to methods such as `Query.join()` as well as loader options like `joinedload()`. Additionally, a “global” version of the option allows limiting criteria to be applied to particular entities in a query globally.

    See also

    Adding Criteria to loader options

    Adding global WHERE / ON criteria

    `with_loader_criteria()`

    References: [#4472](https://www.sqlalchemy.org/trac/ticket/4472)

*   **[orm] [feature]**

    The ORM Declarative system is now unified into the ORM itself, with new import spaces under `sqlalchemy.orm` and new kinds of mappings. Support for decorator-based mappings without using a base class, support for classical style-mapper() calls that have access to the declarative class registry for relationships, and full integration of Declarative with 3rd party class attribute systems like `dataclasses` and `attrs` is now supported.

    See also

    Declarative is now integrated into the ORM with new features

    Python Dataclasses, attrs Supported w/ Declarative, Imperative Mappings

    References: [#5508](https://www.sqlalchemy.org/trac/ticket/5508)

*   **[orm] [feature]**

    Eager loaders, such as joined loading, SELECT IN loading, etc., when configured on a mapper or via query options will now be invoked during the refresh on an expired object; in the case of selectinload and subqueryload, since the additional load is for a single object only, the “immediateload” scheme is used in these cases which resembles the single-parent query emitted by lazy loading.

    See also

    Eager loaders emit during unexpire operations

    References: [#1763](https://www.sqlalchemy.org/trac/ticket/1763)

*   **[orm] [feature]**

    Added support for direct mapping of Python classes that are defined using the Python `dataclasses` decorator. Pull request courtesy Václav Klusák. The new feature integrates into new support at the Declarative level for systems such as `dataclasses` and `attrs`.

    See also

    Python Dataclasses, attrs Supported w/ Declarative, Imperative Mappings

    Declarative is now integrated into the ORM with new features

    References: [#5027](https://www.sqlalchemy.org/trac/ticket/5027)

*   **[orm] [feature]**

    Added “raiseload” feature for ORM mapped columns via `defer.raiseload` parameter on `defer()` and `deferred()`. This provides similar behavior for column-expression mapped attributes as the `raiseload()` option does for relationship mapped attributes. The change also includes some behavioral changes to deferred columns regarding expiration; see the migration notes for details.

    See also

    Raiseload for Columns

    References: [#4826](https://www.sqlalchemy.org/trac/ticket/4826)

*   **[orm] [usecase]**

    The evaluator that takes place within the ORM bulk update and delete for synchronize_session=”evaluate” now supports the IN and NOT IN operators. Tuple IN is also supported.

    References: [#1653](https://www.sqlalchemy.org/trac/ticket/1653)

*   **[orm] [usecase]**

    Enhanced logic that tracks if relationships will be conflicting with each other when they write to the same column to include simple cases of two relationships that should have a “backref” between them. This means that if two relationships are not viewonly, are not linked with back_populates and are not otherwise in an inheriting sibling/overriding arrangement, and will populate the same foreign key column, a warning is emitted at mapper configuration time warning that a conflict may arise. A new parameter `relationship.overlaps` is added to suit those very rare cases where such an overlapping persistence arrangement may be unavoidable.

    References: [#5171](https://www.sqlalchemy.org/trac/ticket/5171)

*   **[orm] [usecase]**

    The ORM bulk update and delete operations, historically available via the `Query.update()` and `Query.delete()` methods as well as via the `Update` and `Delete` constructs for 2.0 style execution, will now automatically accommodate for the additional WHERE criteria needed for a single-table inheritance discriminator in order to limit the statement to rows referring to the specific subtype requested. The new `with_loader_criteria()` construct is also supported for with bulk update/delete operations.

    References: [#3903](https://www.sqlalchemy.org/trac/ticket/3903), [#5018](https://www.sqlalchemy.org/trac/ticket/5018)

*   **[orm] [usecase]**

    Update `relationship.sync_backref` flag in a relationship to make it implicitly `False` in `viewonly=True` relationships, preventing synchronization events.

    See also

    Viewonly relationships don’t synchronize backrefs

    References: [#5237](https://www.sqlalchemy.org/trac/ticket/5237)

*   **[orm] [change]**

    The condition where a pending object being flushed with an identity that already exists in the identity map has been adjusted to emit a warning, rather than throw a `FlushError`. The rationale is so that the flush will proceed and raise a `IntegrityError` instead, in the same way as if the existing object were not present in the identity map already. This helps with schemes that are using the `IntegrityError` as a means of catching whether or not a row already exists in the table.

    See also

    The “New instance conflicts with existing identity” error is now a warning

    References: [#4662](https://www.sqlalchemy.org/trac/ticket/4662)

*   **[orm] [change] [sql]**

    A selection of Core and ORM query objects now perform much more of their Python computational tasks within the compile step, rather than at construction time. This is to support an upcoming caching model that will provide for caching of the compiled statement structure based on a cache key that is derived from the statement construct, which itself is expected to be newly constructed in Python code each time it is used. This means that the internal state of these objects may not be the same as it used to be, as well as that some but not all error raise scenarios for various kinds of argument validation will occur within the compilation / execution phase, rather than at statement construction time. See the migration notes linked below for complete details.

    See also

    Many Core and ORM statement objects now perform much of their construction and validation in the compile phase

*   **[orm] [change]**

    The automatic uniquing of rows on the client side is turned off for the new 2.0 style of ORM querying. This improves both clarity and performance. However, uniquing of rows on the client side is generally necessary when using joined eager loading for collections, as there will be duplicates of the primary entity for each element in the collection because a join was used. This uniquing must now be manually enabled and can be achieved using the new `Result.unique()` modifier. To avoid silent failure, the ORM explicitly requires the method be called when the result of an ORM query in 2.0 style makes use of joined load collections. The newer `selectinload()` strategy is likely preferable for eager loading of collections in any case.

    See also

    ORM Rows not uniquified by default

    References: [#4395](https://www.sqlalchemy.org/trac/ticket/4395)

*   **[orm] [change]**

    The ORM will now warn when asked to coerce a `select()` construct into a subquery implicitly. This occurs within places such as the `Query.select_entity_from()` and `Query.select_from()` methods as well as within the `with_polymorphic()` function. When a `SelectBase` (which is what’s produced by `select()`) or `Query` object is passed directly to these functions and others, the ORM is typically coercing them to be a subquery by calling the `SelectBase.alias()` method automatically (which is now superseded by the `SelectBase.subquery()` method). See the migration notes linked below for further details.

    See also

    A SELECT statement is no longer implicitly considered to be a FROM clause

    References: [#4617](https://www.sqlalchemy.org/trac/ticket/4617)

*   **[orm] [change]**

    The “KeyedTuple” class returned by `Query` is now replaced with the Core `Row` class, which behaves in the same way as KeyedTuple. In SQLAlchemy 2.0, both Core and ORM will return result rows using the same `Row` object. In the interim, Core uses a backwards-compatibility class `LegacyRow` that maintains the former mapping/tuple hybrid behavior used by “RowProxy”.

    See also

    The “KeyedTuple” object returned by Query is replaced by Row

    References: [#4710](https://www.sqlalchemy.org/trac/ticket/4710)

*   **[orm] [performance]**

    The bulk update and delete methods `Query.update()` and `Query.delete()`, as well as their 2.0-style counterparts, now make use of RETURNING when the “fetch” strategy is used in order to fetch the list of affected primary key identites, rather than emitting a separate SELECT, when the backend in use supports RETURNING. Additionally, the “fetch” strategy will in ordinary cases not expire the attributes that have been updated, and will instead apply the updated values directly in the same way that the “evaluate” strategy does, to avoid having to refresh the object. The “evaluate” strategy will also fall back to expiring attributes that were updated to a SQL expression that was unevaluable in Python.

    See also

    ORM Bulk Update and Delete use RETURNING for “fetch” strategy when available

*   **[orm] [performance] [postgresql]**

    Implemented support for the psycopg2 `execute_values()` extension within the ORM flush process via the enhancements to Core made in [#5401](https://www.sqlalchemy.org/trac/ticket/5401), so that this extension is used both as a strategy to batch INSERT statements together as well as that RETURNING may now be used among multiple parameter sets to retrieve primary key values back in batch. This allows nearly all INSERT statements emitted by the ORM on behalf of PostgreSQL to be submitted in batch and also via the `execute_values()` extension which benches at five times faster than plain executemany() for this particular backend.

    See also

    ORM Batch inserts with psycopg2 now batch statements with RETURNING in most cases

    References: [#5263](https://www.sqlalchemy.org/trac/ticket/5263)

*   **[orm] [bug]**

    A query that is against a mapped inheritance subclass which also uses `Query.select_entity_from()` or a similar technique in order to provide an existing subquery to SELECT from, will now raise an error if the given subquery returns entities that do not correspond to the given subclass, that is, they are sibling or superclasses in the same hierarchy. Previously, these would be returned without error. Additionally, if the inheritance mapping is a single-inheritance mapping, the given subquery must apply the appropriate filtering against the polymorphic discriminator column in order to avoid this error; previously, the `Query` would add this criteria to the outside query however this interferes with some kinds of query that return other kinds of entities as well.

    See also

    Stricter behavior when querying inheritance mappings using custom queries

    References: [#5122](https://www.sqlalchemy.org/trac/ticket/5122)

*   **[orm] [bug]**

    The internal attribute symbols NO_VALUE and NEVER_SET have been unified, as there was no meaningful difference between these two symbols, other than a few codepaths where they were differentiated in subtle and undocumented ways, these have been fixed.

    References: [#4696](https://www.sqlalchemy.org/trac/ticket/4696)

*   **[orm] [bug]**

    Fixed bug where a versioning column specified on a mapper against a `select()` construct where the version_id_col itself were against the underlying table would incur additional loads when accessed, even if the value were locally persisted by the flush. The actual fix is a result of the changes in [#4617](https://www.sqlalchemy.org/trac/ticket/4617), by fact that a `select()` object no longer has a `.c` attribute and therefore does not confuse the mapper into thinking there’s an unknown column value present.

    References: [#4194](https://www.sqlalchemy.org/trac/ticket/4194)

*   **[orm] [bug]**

    An `UnmappedInstanceError` is now raised for `InstrumentedAttribute` if an instance is an unmapped object. Prior to this an `AttributeError` was raised. Pull request courtesy Ramon Williams.

    References: [#3858](https://www.sqlalchemy.org/trac/ticket/3858)

*   **[orm] [bug]**

    The `Session` object no longer initiates a `SessionTransaction` object immediately upon construction or after the previous transaction is closed; instead, “autobegin” logic now initiates the new `SessionTransaction` on demand when it is next needed. Rationale includes to remove reference cycles from a `Session` that has been closed out, as well as to remove the overhead incurred by the creation of `SessionTransaction` objects that are often discarded immediately. This change affects the behavior of the `SessionEvents.after_transaction_create()` hook in that the event will be emitted when the `Session` first requires a `SessionTransaction` be present, rather than whenever the `Session` were created or the previous `SessionTransaction` were closed. Interactions with the `Engine` and the database itself remain unaffected.

    See also

    Session features new “autobegin” behavior

    References: [#5074](https://www.sqlalchemy.org/trac/ticket/5074)

*   **[orm] [bug]**

    Added new entity-targeting capabilities to the ORM query context help with the case where the `Session` is using a bind dictionary against mapped classes, rather than a single bind, and the `Query` is against a Core statement that was ultimately generated from a method such as `Query.subquery()`. First implemented using a deep search, the current approach leverages the unified `select()` construct to keep track of the first mapper that is part of the construct.

    References: [#4829](https://www.sqlalchemy.org/trac/ticket/4829)

*   **[orm] [bug] [inheritance]**

    An `ArgumentError` is now raised if both the `selectable` and `flat` parameters are set to True in `with_polymorphic()`. The selectable name is already aliased and applying flat=True overrides the selectable name with an anonymous name that would’ve previously caused the code to break. Pull request courtesy Ramon Williams.

    References: [#4212](https://www.sqlalchemy.org/trac/ticket/4212)

*   **[orm] [bug]**

    Fixed issue in polymorphic loading internals which would fall back to a more expensive, soon-to-be-deprecated form of result column lookup within certain unexpiration scenarios in conjunction with the use of “with_polymorphic”.

    References: [#4718](https://www.sqlalchemy.org/trac/ticket/4718)

*   **[orm] [bug]**

    An error is raised if any persistence-related “cascade” settings are made on a `relationship()` that also sets up viewonly=True. The “cascade” settings now default to non-persistence related settings only when viewonly is also set. This is the continuation from [#4993](https://www.sqlalchemy.org/trac/ticket/4993) where this setting was changed to emit a warning in 1.3.

    See also

    Persistence-related cascade operations disallowed with viewonly=True

    References: [#4994](https://www.sqlalchemy.org/trac/ticket/4994)

*   **[orm] [bug]**

    Improved declarative inheritance scanning to not get tripped up when the same base class appears multiple times in the base inheritance list.

    References: [#4699](https://www.sqlalchemy.org/trac/ticket/4699)

*   **[orm] [bug]**

    Fixed bug in ORM versioning feature where assignment of an explicit version_id for a counter configured against a mapped selectable where version_id_col is against the underlying table would fail if the previous value were expired; this was due to the fact that the mapped attribute would not be configured with active_history=True.

    References: [#4195](https://www.sqlalchemy.org/trac/ticket/4195)

*   **[orm] [bug]**

    An exception is now raised if the ORM loads a row for a polymorphic instance that has a primary key but the discriminator column is NULL, as discriminator columns should not be null.

    References: [#4836](https://www.sqlalchemy.org/trac/ticket/4836)

*   **[orm] [bug]**

    Accessing a collection-oriented attribute on a newly created object no longer mutates `__dict__`, but still returns an empty collection as has always been the case. This allows collection-oriented attributes to work consistently in comparison to scalar attributes which return `None`, but also don’t mutate `__dict__`. In order to accommodate for the collection being mutated, the same empty collection is returned each time once initially created, and when it is mutated (e.g. an item appended, added, etc.) it is then moved into `__dict__`. This removes the last of mutating side-effects on read-only attribute access within the ORM.

    See also

    Accessing an uninitialized collection attribute on a transient object no longer mutates __dict__

    References: [#4519](https://www.sqlalchemy.org/trac/ticket/4519)

*   **[orm] [bug]**

    The refresh of an expired object will now trigger an autoflush if the list of expired attributes include one or more attributes that were explicitly expired or refreshed using the `Session.expire()` or `Session.refresh()` methods. This is an attempt to find a middle ground between the normal unexpiry of attributes that can happen in many cases where autoflush is not desirable, vs. the case where attributes are being explicitly expired or refreshed and it is possible that these attributes depend upon other pending state within the session that needs to be flushed. The two methods now also gain a new flag `Session.expire.autoflush` and `Session.refresh.autoflush`, defaulting to True; when set to False, this will disable the autoflush that occurs on unexpire for these attributes.

    References: [#5226](https://www.sqlalchemy.org/trac/ticket/5226)

*   **[orm] [bug]**

    The behavior of the `relationship.cascade_backrefs` flag will be reversed in 2.0 and set to `False` unconditionally, such that backrefs don’t cascade save-update operations from a forwards-assignment to a backwards assignment. A 2.0 deprecation warning is emitted when the parameter is left at its default of `True` at the point at which such a cascade operation actually takes place. The new behavior can be established as always by setting the flag to `False` on a specific `relationship()`, or more generally can be set up across the board by setting the `Session.future` flag to True.

    See also

    cascade_backrefs behavior deprecated for removal in 2.0

    References: [#5150](https://www.sqlalchemy.org/trac/ticket/5150)

*   **[orm] [deprecated]**

    The “slice index” feature used by `Query` as well as by the dynamic relationship loader will no longer accept negative indexes in SQLAlchemy 2.0\. These operations do not work efficiently and load the entire collection in, which is both surprising and undesirable. These will warn in 1.4 unless the `Session.future` flag is set in which case they will raise IndexError.

    References: [#5606](https://www.sqlalchemy.org/trac/ticket/5606)

*   **[orm] [deprecated]**

    Calling the `Query.instances()` method without passing a `QueryContext` is deprecated. The original use case for this was that a `Query` could yield ORM objects when given only the entities to be selected as well as a DBAPI cursor object. However, for this to work correctly there is essential metadata that is passed from a SQLAlchemy `ResultProxy` that is derived from the mapped column expressions, which comes originally from the `QueryContext`. To retrieve ORM results from arbitrary SELECT statements, the `Query.from_statement()` method should be used.

    References: [#4719](https://www.sqlalchemy.org/trac/ticket/4719)

*   **[orm] [deprecated]**

    Using strings to represent relationship names in ORM operations such as `Query.join()`, as well as strings for all ORM attribute names in loader options like `selectinload()` is deprecated and will be removed in SQLAlchemy 2.0\. The class-bound attribute should be passed instead. This provides much better specificity to the given method, allows for modifiers such as `of_type()`, and reduces internal complexity.

    Additionally, the `aliased` and `from_joinpoint` parameters to `Query.join()` are also deprecated. The `aliased()` construct now provides for a great deal of flexibility and capability and should be used directly.

    See also

    ORM Query - Joining / loading on relationships uses attributes, not strings

    ORM Query - join(…, aliased=True), from_joinpoint removed

    References: [#4705](https://www.sqlalchemy.org/trac/ticket/4705), [#5202](https://www.sqlalchemy.org/trac/ticket/5202)

*   **[orm] [deprecated]**

    Deprecated logic in `Query.distinct()` that automatically adds columns in the ORDER BY clause to the columns clause; this will be removed in 2.0.

    See also

    Using DISTINCT with additional columns, but only select the entity

    References: [#5134](https://www.sqlalchemy.org/trac/ticket/5134)

*   **[orm] [deprecated]**

    Passing keyword arguments to methods such as `Session.execute()` to be passed into the `Session.get_bind()` method is deprecated; the new `Session.execute.bind_arguments` dictionary should be passed instead.

    References: [#5573](https://www.sqlalchemy.org/trac/ticket/5573)

*   **[orm] [deprecated]**

    The `eagerload()` and `relation()` were old aliases and are now deprecated. Use `joinedload()` and `relationship()` respectively.

    References: [#5192](https://www.sqlalchemy.org/trac/ticket/5192)

*   **[orm] [removed]**

    All long-deprecated “extension” classes have been removed, including MapperExtension, SessionExtension, PoolListener, ConnectionProxy, AttributeExtension. These classes have been deprecated since version 0.7 long superseded by the event listener system.

    References: [#4638](https://www.sqlalchemy.org/trac/ticket/4638)

*   **[orm] [removed]**

    Remove the deprecated loader options `joinedload_all`, `subqueryload_all`, `lazyload_all`, `selectinload_all`. The normal version with method chaining should be used in their place.

    References: [#4642](https://www.sqlalchemy.org/trac/ticket/4642)

*   **[orm] [removed]**

    Remove deprecated function `comparable_property`. Please refer to the `hybrid` extension. This also removes the function `comparable_using` in the declarative extension.

    Remove deprecated function `compile_mappers`. Please use `configure_mappers()`

    Remove deprecated method `collection.linker`. Please refer to the `AttributeEvents.init_collection()` and `AttributeEvents.dispose_collection()` event handlers.

    Remove deprecated method `Session.prune` and parameter `Session.weak_identity_map`. See the recipe at Session Referencing Behavior for an event-based approach to maintaining strong identity references. This change also removes the class `StrongInstanceDict`.

    Remove deprecated parameter `mapper.order_by`. Use `Query.order_by()` to determine the ordering of a result set.

    Remove deprecated parameter `Session._enable_transaction_accounting`.

    Remove deprecated parameter `Session.is_modified.passive`.

    References: [#4643](https://www.sqlalchemy.org/trac/ticket/4643)

### engine

*   **[engine] [feature]**

    Implemented an all-new `Result` object that replaces the previous `ResultProxy` object. As implemented in Core, the subclass `CursorResult` features a compatible calling interface with the previous `ResultProxy`, and additionally adds a great amount of new functionality that can be applied to Core result sets as well as ORM result sets, which are now integrated into the same model. `Result` includes features such as column selection and rearrangement, improved fetchmany patterns, uniquing, as well as a variety of implementations that can be used to create database results from in-memory structures as well.

    See also

    New Result object

    References: [#4395](https://www.sqlalchemy.org/trac/ticket/4395), [#4959](https://www.sqlalchemy.org/trac/ticket/4959), [#5087](https://www.sqlalchemy.org/trac/ticket/5087)

*   **[engine] [feature] [orm]**

    SQLAlchemy now includes support for Python asyncio within both Core and ORM, using the included asyncio extension. The extension makes use of the [greenlet](https://greenlet.readthedocs.io/en/latest/) library in order to adapt SQLAlchemy’s sync-oriented internals such that an asyncio interface that ultimately interacts with an asyncio database adapter is now feasible. The single driver supported at the moment is the asyncpg driver for PostgreSQL.

    See also

    Asynchronous IO Support for Core and ORM

    References: [#3414](https://www.sqlalchemy.org/trac/ticket/3414)

*   **[engine] [feature] [alchemy2]**

    Implemented the `create_engine.future` parameter which enables forwards compatibility with SQLAlchemy 2\. is used for forwards compatibility with SQLAlchemy 2\. This engine features always-transactional behavior with autobegin.

    See also

    SQLAlchemy 2.0 - Major Migration Guide

    References: [#4644](https://www.sqlalchemy.org/trac/ticket/4644)

*   **[engine] [feature] [pyodbc]**

    Reworked the “setinputsizes()” set of dialect hooks to be correctly extensible for any arbitrary DBAPI, by allowing dialects individual hooks that may invoke cursor.setinputsizes() in the appropriate style for that DBAPI. In particular this is intended to support pyodbc’s style of usage which is fundamentally different from that of cx_Oracle. Added support for pyodbc.

    References: [#5649](https://www.sqlalchemy.org/trac/ticket/5649)

*   **[engine] [feature]**

    Added new reflection method `Inspector.get_sequence_names()` which returns all the sequences defined and `Inspector.has_sequence()` to check if a particular sequence exits. Support for this method has been added to the backend that support `Sequence`: PostgreSQL, Oracle and MariaDB >= 10.3.

    References: [#2056](https://www.sqlalchemy.org/trac/ticket/2056)

*   **[engine] [feature]**

    The `Table.autoload_with` parameter now accepts an `Inspector` object directly, as well as any `Engine` or `Connection` as was the case before.

    References: [#4755](https://www.sqlalchemy.org/trac/ticket/4755)

*   **[engine] [change]**

    The `RowProxy` class is no longer a “proxy” object, and is instead directly populated with the post-processed contents of the DBAPI row tuple upon construction. Now named `Row`, the mechanics of how the Python-level value processors have been simplified, particularly as it impacts the format of the C code, so that a DBAPI row is processed into a result tuple up front. The object returned by the `ResultProxy` is now the `LegacyRow` subclass, which maintains mapping/tuple hybrid behavior, however the base `Row` class now behaves more fully like a named tuple.

    See also

    RowProxy is no longer a “proxy”; is now called Row and behaves like an enhanced named tuple

    References: [#4710](https://www.sqlalchemy.org/trac/ticket/4710)

*   **[engine] [performance]**

    The pool “pre-ping” feature has been refined to not invoke for a DBAPI connection that was just opened in the same checkout operation. pre ping only applies to a DBAPI connection that’s been checked into the pool and is being checked out again.

    References: [#4524](https://www.sqlalchemy.org/trac/ticket/4524)

*   **[engine] [performance] [change] [py3k]**

    Disabled the “unicode returns” check that runs on dialect startup when running under Python 3, which for many years has occurred in order to test the current DBAPI’s behavior for whether or not it returns Python Unicode or Py2K strings for the VARCHAR and NVARCHAR datatypes. The check still occurs by default under Python 2, however the mechanism to test the behavior will be removed in SQLAlchemy 2.0 when Python 2 support is also removed.

    This logic was very effective when it was needed, however now that Python 3 is standard, all DBAPIs are expected to return Python 3 strings for character datatypes. In the unlikely case that a third party DBAPI does not support this, the conversion logic within `String` is still available and the third party dialect may specify this in its upfront dialect flags by setting the dialect level flag `returns_unicode_strings` to one of `String.RETURNS_CONDITIONAL` or `String.RETURNS_BYTES`, both of which will enable Unicode conversion even under Python 3.

    References: [#5315](https://www.sqlalchemy.org/trac/ticket/5315)

*   **[engine] [bug]**

    Revised the `Connection.execution_options.schema_translate_map` feature such that the processing of the SQL statement to receive a specific schema name occurs within the execution phase of the statement, rather than at the compile phase. This is to support the statement being efficiently cached. Previously, the current schema being rendered into the statement for a particular run would be considered as part of the cache key itself, meaning that for a run against hundreds of schemas, there would be hundreds of cache keys, rendering the cache much less performant. The new behavior is that the rendering is done in a similar manner as the “post compile” rendering added in 1.4 as part of [#4645](https://www.sqlalchemy.org/trac/ticket/4645), [#4808](https://www.sqlalchemy.org/trac/ticket/4808).

    References: [#5004](https://www.sqlalchemy.org/trac/ticket/5004)

*   **[engine] [bug]**

    The `Connection` object will now not clear a rolled-back transaction until the outermost transaction is explicitly rolled back. This is essentially the same behavior that the ORM `Session` has had for a long time, where an explicit call to `.rollback()` on all enclosing transactions is required for the transaction to logically clear, even though the DBAPI-level transaction has already been rolled back. The new behavior helps with situations such as the “ORM rollback test suite” pattern where the test suite rolls the transaction back within the ORM scope, but the test harness which seeks to control the scope of the transaction externally does not expect a new transaction to start implicitly.

    See also

    Connection-level transactions can now be inactive based on subtransaction

    References: [#4712](https://www.sqlalchemy.org/trac/ticket/4712)

*   **[engine] [bug]**

    Adjusted the dialect initialization process such that the `Dialect.on_connect()` is not called a second time on the first connection. The hook is called first, then the `Dialect.initialize()` is called if that connection is the first for that dialect, then no more events are called. This eliminates the two calls to the “on_connect” function which can produce very difficult debugging situations.

    References: [#5497](https://www.sqlalchemy.org/trac/ticket/5497)

*   **[engine] [deprecated]**

    The `URL` object is now an immutable named tuple. To modify a URL object, use the `URL.set()` method to produce a new URL object.

    See also

    The URL object is now immutable - notes on migration

    References: [#5526](https://www.sqlalchemy.org/trac/ticket/5526)

*   **[engine] [deprecated]**

    The `MetaData.bind` argument as well as the overall concept of “bound metadata” is deprecated in SQLAlchemy 1.4 and will be removed in SQLAlchemy 2.0\. The parameter as well as related functions now emit a `RemovedIn20Warning` when SQLAlchemy 2.0 Deprecations Mode is in use.

    See also

    “Implicit” and “Connectionless” execution, “bound metadata” removed

    References: [#4634](https://www.sqlalchemy.org/trac/ticket/4634)

*   **[engine] [deprecated]**

    The `server_side_cursors` engine-wide parameter is deprecated and will be removed in a future release. For unbuffered cursors, the `Connection.execution_options.stream_results` execution option should be used on a per-execution basis.

*   **[engine] [deprecated]**

    The `Connection.connect()` method is deprecated as is the concept of “connection branching”, which copies a `Connection` into a new one that has a no-op “.close()” method. This pattern is oriented around the “connectionless execution” concept which is also being removed in 2.0.

    References: [#5131](https://www.sqlalchemy.org/trac/ticket/5131)

*   **[engine] [deprecated]**

    The `case_sensitive` flag on `create_engine()` is deprecated; this flag was part of the transition of the result row object to allow case sensitive column matching as the default, while providing backwards compatibility for the former matching method. All string access for a row should be assumed to be case sensitive just like any other Python mapping.

    References: [#4878](https://www.sqlalchemy.org/trac/ticket/4878)

*   **[engine] [deprecated]**

    ”Implicit autocommit”, which is the COMMIT that occurs when a DML or DDL statement is emitted on a connection, is deprecated and won’t be part of SQLAlchemy 2.0\. A 2.0-style warning is emitted when autocommit takes effect, so that the calling code may be adjusted to use an explicit transaction.

    As part of this change, DDL methods such as `MetaData.create_all()` when used against an `Engine` will run the operation in a BEGIN block if one is not started already.

    See also

    SQLAlchemy 2.0 Deprecations Mode

    References: [#4846](https://www.sqlalchemy.org/trac/ticket/4846)

*   **[engine] [deprecated]**

    Deprecated the behavior by which a `Column` can be used as the key in a result set row lookup, when that `Column` is not part of the SQL selectable that is being selected; that is, it is only matched on name. A deprecation warning is now emitted for this case. Various ORM use cases, such as those involving `text()` constructs, have been improved so that this fallback logic is avoided in most cases.

    References: [#4877](https://www.sqlalchemy.org/trac/ticket/4877)

*   **[engine] [deprecated]**

    Deprecated remaining engine-level introspection and utility methods including `Engine.run_callable()`, `Engine.transaction()`, `Engine.table_names()`, `Engine.has_table()`. The utility methods are superseded by modern context-manager patterns, and the table introspection tasks are suited by the `Inspector` object.

    References: [#4755](https://www.sqlalchemy.org/trac/ticket/4755)

*   **[engine] [removed]**

    Remove deprecated method `get_primary_keys` in the `Dialect` and `Inspector` classes. Please refer to the `Dialect.get_pk_constraint()` and `Inspector.get_primary_keys()` methods.

    Remove deprecated event `dbapi_error` and the method `ConnectionEvents.dbapi_error`. Please refer to the `ConnectionEvents.handle_error()` event. This change also removes the attributes `ExecutionContext.is_disconnect` and `ExecutionContext.exception`.

    References: [#4643](https://www.sqlalchemy.org/trac/ticket/4643)

*   **[engine] [removed]**

    The internal dialect method `Dialect.reflecttable` has been removed. A review of third party dialects has not found any making use of this method, as it was already documented as one that should not be used by external dialects. Additionally, the private `Engine._run_visitor` method is also removed.

    References: [#4755](https://www.sqlalchemy.org/trac/ticket/4755)

*   **[engine] [removed]**

    The long-deprecated `Inspector.get_table_names.order_by` parameter has been removed.

    References: [#4755](https://www.sqlalchemy.org/trac/ticket/4755)

*   **[engine] [renamed]**

    The `Inspector.reflecttable()` was renamed to `Inspector.reflect_table()`.

    References: [#5244](https://www.sqlalchemy.org/trac/ticket/5244)

### sql

*   **[sql] [feature]**

    Added “from linting” as a built-in feature to the SQL compiler. This allows the compiler to maintain graph of all the FROM clauses in a particular SELECT statement, linked by criteria in either the WHERE or in JOIN clauses that link these FROM clauses together. If any two FROM clauses have no path between them, a warning is emitted that the query may be producing a cartesian product. As the Core expression language as well as the ORM are built on an “implicit FROMs” model where a particular FROM clause is automatically added if any part of the query refers to it, it is easy for this to happen inadvertently and it is hoped that the new feature helps with this issue.

    See also

    Built-in FROM linting will warn for any potential cartesian products in a SELECT statement

    References: [#4737](https://www.sqlalchemy.org/trac/ticket/4737)

*   **[sql] [feature] [mssql] [oracle]**

    Added new “post compile parameters” feature. This feature allows a `bindparam()` construct to have its value rendered into the SQL string before being passed to the DBAPI driver, but after the compilation step, using the “literal render” feature of the compiler. The immediate rationale for this feature is to support LIMIT/OFFSET schemes that don’t work or perform well as bound parameters handled by the database driver, while still allowing for SQLAlchemy SQL constructs to be cacheable in their compiled form. The immediate targets for the new feature are the “TOP N” clause used by SQL Server (and Sybase) which does not support a bound parameter, as well as the “ROWNUM” and optional “FIRST_ROWS()” schemes used by the Oracle dialect, the former of which has been known to perform better without bound parameters and the latter of which does not support a bound parameter. The feature builds upon the mechanisms first developed to support “expanding” parameters for IN expressions. As part of this feature, the Oracle `use_binds_for_limits` feature is turned on unconditionally and this flag is now deprecated.

    See also

    New “post compile” bound parameters used for LIMIT/OFFSET in Oracle, SQL Server

    References: [#4808](https://www.sqlalchemy.org/trac/ticket/4808)

*   **[sql] [feature]**

    Add support for regular expression on supported backends. Two operations have been defined:

    *   `ColumnOperators.regexp_match()` implementing a regular expression match like function.

    *   `ColumnOperators.regexp_replace()` implementing a regular expression string replace function.

    Supported backends include SQLite, PostgreSQL, MySQL / MariaDB, and Oracle.

    See also

    Support for SQL Regular Expression operators

    References: [#1390](https://www.sqlalchemy.org/trac/ticket/1390)

*   **[sql] [feature]**

    The `select()` construct and related constructs now allow for duplication of column labels and columns themselves in the columns clause, mirroring exactly how column expressions were passed in. This allows the tuples returned by an executed result to match what was SELECTed for in the first place, which is how the ORM `Query` works, so this establishes better cross-compatibility between the two constructs. Additionally, it allows column-positioning-sensitive structures such as UNIONs (i.e. `_selectable.CompoundSelect`) to be more intuitively constructed in those cases where a particular column might appear in more than one place. To support this change, the `ColumnCollection` has been revised to support duplicate columns as well as to allow integer index access.

    See also

    SELECT objects and derived FROM clauses allow for duplicate columns and column labels

    References: [#4753](https://www.sqlalchemy.org/trac/ticket/4753)

*   **[sql] [feature]**

    Enhanced the disambiguating labels feature of the `select()` construct such that when a select statement is used in a subquery, repeated column names from different tables are now automatically labeled with a unique label name, without the need to use the full “apply_labels()” feature that combines tablename plus column name. The disambiguated labels are available as plain string keys in the .c collection of the subquery, and most importantly the feature allows an ORM `aliased()` construct against the combination of an entity and an arbitrary subquery to work correctly, targeting the correct columns despite same-named columns in the source tables, without the need for an “apply labels” warning.

    See also

    Selecting from the query itself as a subquery, e.g. “from_self()” - Illustrates the new disambiguation feature as part of a strategy to migrate away from the `Query.from_self()` method.

    References: [#5221](https://www.sqlalchemy.org/trac/ticket/5221)

*   **[sql] [feature]**

    The “expanding IN” feature, which generates IN expressions at query execution time which are based on the particular parameters associated with the statement execution, is now used for all IN expressions made against lists of literal values. This allows IN expressions to be fully cacheable independently of the list of values being passed, and also includes support for empty lists. For any scenario where the IN expression contains non-literal SQL expressions, the old behavior of pre-rendering for each position in the IN is maintained. The change also completes support for expanding IN with tuples, where previously type-specific bind processors weren’t taking effect.

    See also

    All IN expressions render parameters for each value in the list on the fly (e.g. expanding parameters)

    References: [#4645](https://www.sqlalchemy.org/trac/ticket/4645)

*   **[sql] [feature]**

    Along with the new transparent statement caching feature introduced as part of [#4369](https://www.sqlalchemy.org/trac/ticket/4369), a new feature intended to decrease the Python overhead of creating statements is added, allowing lambdas to be used when indicating arguments being passed to a statement object such as select(), Query(), update(), etc., as well as allowing the construction of full statements within lambdas in a similar manner as that of the “baked query” system. The rationale of using lambdas is adapted from that of the “baked query” approach which uses lambdas to encapsulate any amount of Python code into a callable that only needs to be called when the statement is first constructed into a string. The new feature however is more sophisticated in that Python literal values that would be passed as parameters are automatically extracted, so that there is no longer a need to use bindparam() objects with such queries. Use of the feature is optional and can be used to as small or as great a degree as is desired, while still allowing statements to be fully cacheable.

    See also

    Using Lambdas to add significant speed gains to statement production

    References: [#5380](https://www.sqlalchemy.org/trac/ticket/5380)

*   **[sql] [usecase]**

    The `Index.create()` and `Index.drop()` methods now have a parameter `Index.create.checkfirst`, in the same way as that of `Table` and `Sequence`, which when enabled will cause the operation to detect if the index exists (or not) before performing a create or drop operation.

    References: [#527](https://www.sqlalchemy.org/trac/ticket/527)

*   **[sql] [usecase]**

    The `true()` and `false()` operators may now be applied as the “onclause” of a `join()` on a backend that does not support “native boolean” expressions, e.g. Oracle or SQL Server, and the expression will render as “1=1” for true and “1=0” false. This is the behavior that was introduced many years ago in [#2804](https://www.sqlalchemy.org/trac/ticket/2804) for and/or expressions.

*   **[sql] [usecase]**

    Change the method `__str` of `ColumnCollection` to avoid confusing it with a python list of string.

    References: [#5191](https://www.sqlalchemy.org/trac/ticket/5191)

*   **[sql] [usecase]**

    Add support to `FETCH {FIRST | NEXT} [ count ] {ROW | ROWS} {ONLY | WITH TIES}` in the select for the supported backends, currently PostgreSQL, Oracle and MSSQL.

    References: [#5576](https://www.sqlalchemy.org/trac/ticket/5576)

*   **[sql] [usecase]**

    Additional logic has been added such that certain SQL expressions which typically wrap a single database column will use the name of that column as their “anonymous label” name within a SELECT statement, potentially making key-based lookups in result tuples more intuitive. The primary example of this is that of a CAST expression, e.g. `CAST(table.colname AS INTEGER)`, which will export its default name as “colname”, rather than the usual “anon_1” label, that is, `CAST(table.colname AS INTEGER) AS colname`. If the inner expression doesn’t have a name, then the previous “anonymous label” logic is used. When using SELECT statements that make use of `Select.apply_labels()`, such as those emitted by the ORM, the labeling logic will produce `<tablename>_<inner column name>` in the same was as if the column were named alone. The logic applies right now to the `cast()` and `type_coerce()` constructs as well as some single-element boolean expressions.

    See also

    Improved column labeling for simple column expressions using CAST or similar

    References: [#4449](https://www.sqlalchemy.org/trac/ticket/4449)

*   **[sql] [change]**

    The “clause coercion” system, which is SQLAlchemy Core’s system of receiving arguments and resolving them into `ClauseElement` structures in order to build up SQL expression objects, has been rewritten from a series of ad-hoc functions to a fully consistent class-based system. This change is internal and should have no impact on end users other than more specific error messages when the wrong kind of argument is passed to an expression object, however the change is part of a larger set of changes involving the role and behavior of `select()` objects.

    References: [#4617](https://www.sqlalchemy.org/trac/ticket/4617)

*   **[sql] [change]**

    Added a core `Values` object that enables a VALUES construct to be used in the FROM clause of an SQL statement for databases that support it (mainly PostgreSQL and SQL Server).

    References: [#4868](https://www.sqlalchemy.org/trac/ticket/4868)

*   **[sql] [change]**

    The `select()` construct is moving towards a new calling form that is `select(col1, col2, col3, ..)`, with all other keyword arguments removed, as these are all suited using generative methods. The single list of column or table arguments passed to `select()` is still accepted, however is no longer necessary if expressions are passed in a simple positional style. Other keyword arguments are disallowed when this form is used.

    See also

    select(), case() now accept positional expressions

    References: [#5284](https://www.sqlalchemy.org/trac/ticket/5284)

*   **[sql] [change]**

    As part of the SQLAlchemy 2.0 migration project, a conceptual change has been made to the role of the `SelectBase` class hierarchy, which is the root of all “SELECT” statement constructs, in that they no longer serve directly as FROM clauses, that is, they no longer subclass `FromClause`. For end users, the change mostly means that any placement of a `select()` construct in the FROM clause of another `select()` requires first that it be wrapped in a subquery first, which historically is through the use of the `SelectBase.alias()` method, and is now also available through the use of `SelectBase.subquery()`. This was usually a requirement in any case since several databases don’t accept unnamed SELECT subqueries in their FROM clause in any case.

    See also

    A SELECT statement is no longer implicitly considered to be a FROM clause

    References: [#4617](https://www.sqlalchemy.org/trac/ticket/4617)

*   **[sql] [change]**

    Added a new Core class `Subquery`, which takes the place of `Alias` when creating named subqueries against a `SelectBase` object. `Subquery` acts in the same way as `Alias` and is produced from the `SelectBase.subquery()` method; for ease of use and backwards compatibility, the `SelectBase.alias()` method is synonymous with this new method.

    See also

    A SELECT statement is no longer implicitly considered to be a FROM clause

    References: [#4617](https://www.sqlalchemy.org/trac/ticket/4617)

*   **[sql] [performance]**

    An all-encompassing reorganization and refactoring of Core and ORM internals now allows all Core and ORM statements within the areas of DQL (e.g. SELECTs) and DML (e.g. INSERT, UPDATE, DELETE) to allow their SQL compilation as well as the construction of result-fetching metadata to be fully cached in most cases. This effectively provides a transparent and generalized version of what the “Baked Query” extension has offered for the ORM in past versions. The new feature can calculate the cache key for any given SQL construction based on the string that it would ultimately produce for a given dialect, allowing functions that compose the equivalent select(), Query(), insert(), update() or delete() object each time to have that statement cached after it’s generated the first time.

    The feature is enabled transparently but includes some new programming paradigms that may be employed to make the caching even more efficient.

    See also

    Transparent SQL Compilation Caching added to All DQL, DML Statements in Core, ORM

    SQL Compilation Caching

    References: [#4639](https://www.sqlalchemy.org/trac/ticket/4639)

*   **[sql] [bug]**

    Fixed issue where when constructing constraints from ORM-bound columns, primarily `ForeignKey` objects but also `UniqueConstraint`, `CheckConstraint` and others, the ORM-level `InstrumentedAttribute` is discarded entirely, and all ORM-level annotations from the columns are removed; this is so that the constraints are still fully pickleable without the ORM-level entities being pulled in. These annotations are not necessary to be present at the schema/metadata level.

    References: [#5001](https://www.sqlalchemy.org/trac/ticket/5001)

*   **[sql] [bug]**

    Registered function names based on `GenericFunction` are now retrieved in a case-insensitive fashion in all cases, removing the deprecation logic from 1.3 which temporarily allowed multiple `GenericFunction` objects to exist with differing cases. A `GenericFunction` that replaces another on the same name whether or not it’s case sensitive emits a warning before replacing the object.

    References: [#4569](https://www.sqlalchemy.org/trac/ticket/4569), [#4649](https://www.sqlalchemy.org/trac/ticket/4649)

*   **[sql] [bug]**

    Creating an `and_()` or `or_()` construct with no arguments or empty `*args` will now emit a deprecation warning, as the SQL produced is a no-op (i.e. it renders as a blank string). This behavior is considered to be non-intuitive, so for empty or possibly empty `and_()` or `or_()` constructs, an appropriate default boolean should be included, such as `and_(True, *args)` or `or_(False, *args)`. As has been the case for many major versions of SQLAlchemy, these particular boolean values will not render if the `*args` portion is non-empty.

    References: [#5054](https://www.sqlalchemy.org/trac/ticket/5054)

*   **[sql] [bug]**

    Improved the `tuple_()` construct such that it behaves predictably when used in a columns-clause context. The SQL tuple is not supported as a “SELECT” columns clause element on most backends; on those that do (PostgreSQL, not surprisingly), the Python DBAPI does not have a “nested type” concept so there are still challenges in fetching rows for such an object. Use of `tuple_()` in a `select()` or `Query` will now raise a `CompileError` at the point at which the `tuple_()` object is seen as presenting itself for fetching rows (i.e., if the tuple is in the columns clause of a subquery, no error is raised). For ORM use,the `Bundle` object is an explicit directive that a series of columns should be returned as a sub-tuple per row and is suggested by the error message. Additionally ,the tuple will now render with parenthesis in all contexts. Previously, the parenthesization would not render in a columns context leading to non-defined behavior.

    References: [#5127](https://www.sqlalchemy.org/trac/ticket/5127)

*   **[sql] [bug] [postgresql]**

    Improved support for column names that contain percent signs in the string, including repaired issues involving anonymous labels that also embedded a column name with a percent sign in it, as well as re-established support for bound parameter names with percent signs embedded on the psycopg2 dialect, using a late-escaping process similar to that used by the cx_Oracle dialect.

    References: [#5653](https://www.sqlalchemy.org/trac/ticket/5653)

*   **[sql] [bug]**

    Custom functions that are created as subclasses of `FunctionElement` will now generate an “anonymous label” based on the “name” of the function just like any other `Function` object, e.g. `"SELECT myfunc() AS myfunc_1"`. While SELECT statements no longer require labels in order for the result proxy object to function, the ORM still targets columns in rows by using objects as mapping keys, which works more reliably when the column expressions have distinct names. In any case, the behavior is now made consistent between functions generated by `func` and those generated as custom `FunctionElement` objects.

    References: [#4887](https://www.sqlalchemy.org/trac/ticket/4887)

*   **[sql] [bug]**

    Reworked the `ClauseElement.compare()` methods in terms of a new visitor-based approach, and additionally added test coverage ensuring that all `ClauseElement` subclasses can be accurately compared against each other in terms of structure. Structural comparison capability is used to a small degree within the ORM currently, however it also may form the basis for new caching features.

    References: [#4336](https://www.sqlalchemy.org/trac/ticket/4336)

*   **[sql] [bug]**

    Deprecate usage of `DISTINCT ON` in dialect other than PostgreSQL. Deprecate old usage of string distinct in MySQL dialect

    References: [#4002](https://www.sqlalchemy.org/trac/ticket/4002)

*   **[sql] [bug]**

    The ORDER BY clause of a `_selectable.CompoundSelect`, e.g. UNION, EXCEPT, etc. will not render the table name associated with a given column when applying `CompoundSelect.order_by()` in terms of a `Table` - bound column. Most databases require that the names in the ORDER BY clause be expressed as label names only which are matched to names in the first SELECT statement. The change is related to [#4617](https://www.sqlalchemy.org/trac/ticket/4617) in that a previous workaround was to refer to the `.c` attribute of the `_selectable.CompoundSelect` in order to get at a column that has no table name. As the subquery is now named, this change allows both the workaround to continue to work, as well as allows table-bound columns as well as the `CompoundSelect.selected_columns` collections to be usable in the `CompoundSelect.order_by()` method.

    References: [#4617](https://www.sqlalchemy.org/trac/ticket/4617)

*   **[sql] [bug]**

    The `Join` construct no longer considers the “onclause” as a source of additional FROM objects to be omitted from the FROM list of an enclosing `Select` object as standalone FROM objects. This applies to an ON clause that includes a reference to another FROM object outside the JOIN; while this is usually not correct from a SQL perspective, it’s also incorrect for it to be omitted, and the behavioral change makes the `Select` / `Join` behave a bit more intuitively.

    References: [#4621](https://www.sqlalchemy.org/trac/ticket/4621)

*   **[sql] [deprecated]**

    The `Join.alias()` method is deprecated and will be removed in SQLAlchemy 2.0\. An explicit select + subquery, or aliasing of the inner tables, should be used instead.

    References: [#5010](https://www.sqlalchemy.org/trac/ticket/5010)

*   **[sql] [deprecated]**

    The `Table` class now raises a deprecation warning when columns with the same name are defined. To replace a column a new parameter `Table.append_column.replace_existing` was added to the `Table.append_column()` method.

    The `ColumnCollection.contains_column()` will now raises an error when called with a string, suggesting the caller to use `in` instead.

*   **[sql] [removed]**

    The “threadlocal” execution strategy, deprecated in 1.3, has been removed for 1.4, as well as the concept of “engine strategies” and the `Engine.contextual_connect` method. The “strategy=’mock’” keyword argument is still accepted for now with a deprecation warning; use `create_mock_engine()` instead for this use case.

    See also

    “threadlocal” engine strategy deprecated - from the 1.3 migration notes which discusses the rationale for deprecation.

    References: [#4632](https://www.sqlalchemy.org/trac/ticket/4632)

*   **[sql] [removed]**

    Removed the `sqlalchemy.sql.visitors.iterate_depthfirst` and `sqlalchemy.sql.visitors.traverse_depthfirst` functions. These functions were unused by any part of SQLAlchemy. The `iterate()` and `traverse()` functions are commonly used for these functions. Also removed unused options from the remaining functions including “column_collections”, “schema_visitor”.

*   **[sql] [removed]**

    Removed the concept of a bound engine from the `Compiler` object, and removed the `.execute()` and `.scalar()` methods from `Compiler`. These were essentially forgotten methods from over a decade ago and had no practical use, and it’s not appropriate for the `Compiler` object itself to be maintaining a reference to an `Engine`.

*   **[sql] [removed]**

    Remove deprecated methods `Compiled.compile`, `ClauseElement.__and__` and `ClauseElement.__or__` and attribute `Over.func`.

    Remove deprecated `FromClause.count` method. Please use the `count` function available from the `func` namespace.

    References: [#4643](https://www.sqlalchemy.org/trac/ticket/4643)

*   **[sql] [removed]**

    Remove deprecated parameters `text.bindparams` and `text.typemap`. Please refer to the `TextClause.bindparams()` and `TextClause.columns()` methods.

    Remove deprecated parameter `Table.useexisting`. Please use `Table.extend_existing`.

    References: [#4643](https://www.sqlalchemy.org/trac/ticket/4643)

*   **[sql] [renamed]**

    `Table` parameter `mustexist` has been renamed to `Table.must_exist` and will now warn when used.

*   **[sql] [renamed]**

    The `SelectBase.as_scalar()` and `Query.as_scalar()` methods have been renamed to `SelectBase.scalar_subquery()` and `Query.scalar_subquery()`, respectively. The old names continue to exist within 1.4 series with a deprecation warning. In addition, the implicit coercion of `SelectBase`, `Alias`, and other SELECT oriented objects into scalar subqueries when evaluated in a column context is also deprecated, and emits a warning that the `SelectBase.scalar_subquery()` method should be called explicitly. This warning will in a later major release become an error, however the message will always be clear when `SelectBase.scalar_subquery()` needs to be invoked. The latter part of the change is for clarity and to reduce the implicit decisionmaking by the query coercion system. The `Subquery.as_scalar()` method, which was previously `Alias.as_scalar`, is also deprecated; `.scalar_subquery()` should be invoked directly from ` `select()` object or `Query` object.

    This change is part of the larger change to convert `select()` objects to no longer be directly part of the “from clause” class hierarchy, which also includes an overhaul of the clause coercion system.

    References: [#4617](https://www.sqlalchemy.org/trac/ticket/4617)

*   **[sql] [renamed]**

    Several operators are renamed to achieve more consistent naming across SQLAlchemy.

    The operator changes are:

    *   `isfalse` is now `is_false`

    *   `isnot_distinct_from` is now `is_not_distinct_from`

    *   `istrue` is now `is_true`

    *   `notbetween` is now `not_between`

    *   `notcontains` is now `not_contains`

    *   `notendswith` is now `not_endswith`

    *   `notilike` is now `not_ilike`

    *   `notlike` is now `not_like`

    *   `notmatch` is now `not_match`

    *   `notstartswith` is now `not_startswith`

    *   `nullsfirst` is now `nulls_first`

    *   `nullslast` is now `nulls_last`

    *   `isnot` is now `is_not`

    *   `notin_` is now `not_in`

    Because these are core operators, the internal migration strategy for this change is to support legacy terms for an extended period of time – if not indefinitely – but update all documentation, tutorials, and internal usage to the new terms. The new terms are used to define the functions, and the legacy terms have been deprecated into aliases of the new terms.

    References: [#5429](https://www.sqlalchemy.org/trac/ticket/5429), [#5435](https://www.sqlalchemy.org/trac/ticket/5435)

*   **[sql] [postgresql]**

    Allow specifying the data type when creating a `Sequence` in PostgreSQL by using the parameter `Sequence.data_type`.

    References: [#5498](https://www.sqlalchemy.org/trac/ticket/5498)

*   **[sql] [reflection]**

    The “NO ACTION” keyword for foreign key “ON UPDATE” is now considered to be the default cascade for a foreign key on all supporting backends (SQlite, MySQL, PostgreSQL) and when detected is not included in the reflection dictionary; this is already the behavior for PostgreSQL and MySQL for all previous SQLAlchemy versions in any case. The “RESTRICT” keyword is positively stored when detected; PostgreSQL does report on this keyword, and MySQL as of version 8.0 does as well. On earlier MySQL versions, it is not reported by the database.

    References: [#4741](https://www.sqlalchemy.org/trac/ticket/4741)

*   **[sql] [reflection]**

    Added support for reflecting “identity” columns, which are now returned as part of the structure returned by `Inspector.get_columns()`. When reflecting full `Table` objects, identity columns will be represented using the `Identity` construct. Currently the supported backends are PostgreSQL >= 10, Oracle >= 12 and MSSQL (with different syntax and a subset of functionalities).

    References: [#5324](https://www.sqlalchemy.org/trac/ticket/5324), [#5527](https://www.sqlalchemy.org/trac/ticket/5527)

### schema

*   **[schema] [change]**

    The `Enum.create_constraint` and `Boolean.create_constraint` parameters now default to False, indicating when a so-called “non-native” version of these two datatypes is created, a CHECK constraint will not be generated by default. These CHECK constraints present schema-management maintenance complexities that should be opted in to, rather than being turned on by default.

    See also

    Enum and Boolean datatypes no longer default to “create constraint”

    References: [#5367](https://www.sqlalchemy.org/trac/ticket/5367)

*   **[schema] [bug]**

    Cleaned up the internal `str()` for datatypes so that all types produce a string representation without any dialect present, including that it works for third-party dialect types without that dialect being present. The string representation defaults to being the UPPERCASE name of that type with nothing else.

    References: [#4262](https://www.sqlalchemy.org/trac/ticket/4262)

*   **[schema] [removed]**

    Remove deprecated class `Binary`. Please use `LargeBinary`.

    References: [#4643](https://www.sqlalchemy.org/trac/ticket/4643)

*   **[schema] [renamed]**

    Renamed the `Table.tometadata()` method to `Table.to_metadata()`. The previous name remains with a deprecation warning.

    References: [#5413](https://www.sqlalchemy.org/trac/ticket/5413)

*   **[schema] [sql]**

    Added the `Identity` construct that can be used to configure identity columns rendered with GENERATED { ALWAYS | BY DEFAULT } AS IDENTITY. Currently the supported backends are PostgreSQL >= 10, Oracle >= 12 and MSSQL (with different syntax and a subset of functionalities).

    References: [#5324](https://www.sqlalchemy.org/trac/ticket/5324), [#5360](https://www.sqlalchemy.org/trac/ticket/5360), [#5362](https://www.sqlalchemy.org/trac/ticket/5362)

### extensions

*   **[extensions] [usecase]**

    Custom compiler constructs created using the `sqlalchemy.ext.compiled` extension will automatically add contextual information to the compiler when a custom construct is interpreted as an element in the columns clause of a SELECT statement, such that the custom element will be targetable as a key in result row mappings, which is the kind of targeting that the ORM uses in order to match column elements into result tuples.

    References: [#4887](https://www.sqlalchemy.org/trac/ticket/4887)

*   **[extensions] [change]**

    Added new parameter `AutomapBase.prepare.autoload_with` which supersedes `AutomapBase.prepare.reflect` and `AutomapBase.prepare.engine`.

    References: [#5142](https://www.sqlalchemy.org/trac/ticket/5142)

### postgresql

*   **[postgresql] [usecase]**

    Added support for PostgreSQL “readonly” and “deferrable” flags for all of psycopg2, asyncpg and pg8000 dialects. This takes advantage of a newly generalized version of the “isolation level” API to support other kinds of session attributes set via execution options that are reliably reset when connections are returned to the connection pool.

    See also

    Setting READ ONLY / DEFERRABLE

    References: [#5549](https://www.sqlalchemy.org/trac/ticket/5549)

*   **[postgresql] [usecase]**

    The maximum buffer size for the `BufferedRowResultProxy`, which is used by dialects such as PostgreSQL when `stream_results=True`, can now be set to a number greater than 1000 and the buffer will grow to that size. Previously, the buffer would not go beyond 1000 even if the value were set larger. The growth of the buffer is also now based on a simple multiplying factor currently set to 5\. Pull request courtesy Soumaya Mauthoor.

    References: [#4914](https://www.sqlalchemy.org/trac/ticket/4914)

*   **[postgresql] [change]**

    When using the psycopg2 dialect for PostgreSQL, psycopg2 minimum version is set at 2.7\. The psycopg2 dialect relies upon many features of psycopg2 released in the past few years, so to simplify the dialect, version 2.7, released in March, 2017 is now the minimum version required.

*   **[postgresql] [performance]**

    The psycopg2 dialect now defaults to using the very performant `execute_values()` psycopg2 extension for compiled INSERT statements, and also implements RETURNING support when this extension is used. This allows INSERT statements that even include an autoincremented SERIAL or IDENTITY value to run very fast while still being able to return the newly generated primary key values. The ORM will then integrate this new feature in a separate change.

    See also

    psycopg2 dialect features “execute_values” with RETURNING for INSERT statements by default - full list of changes regarding the `executemany_mode` parameter.

    References: [#5401](https://www.sqlalchemy.org/trac/ticket/5401)

*   **[postgresql] [bug]**

    The pg8000 dialect has been revised and modernized for the most recent version of the pg8000 driver for PostgreSQL. Pull request courtesy Tony Locke. Note that this necessarily pins pg8000 at 1.16.6 or greater, which no longer has Python 2 support. Python 2 users who require pg8000 should ensure their requirements are pinned at `SQLAlchemy<1.4`.

*   **[postgresql] [deprecated]**

    The pygresql and py-postgresql dialects are deprecated.

    References: [#5189](https://www.sqlalchemy.org/trac/ticket/5189)

*   **[postgresql] [removed]**

    Remove support for deprecated engine URLs of the form `postgres://`; this has emitted a warning for many years and projects should be using `postgresql://`.

    References: [#4643](https://www.sqlalchemy.org/trac/ticket/4643)

### mysql

*   **[mysql] [feature]**

    Added support for MariaDB Connector/Python to the mysql dialect. Original pull request courtesy Georg Richter.

    References: [#5459](https://www.sqlalchemy.org/trac/ticket/5459)

*   **[mysql] [usecase]**

    Added a new dialect token “mariadb” that may be used in place of “mysql” in the `create_engine()` URL. This will deliver a MariaDB dialect subclass of the MySQLDialect in use that forces the “is_mariadb” flag to True. The dialect will raise an error if a server version string that does not indicate MariaDB in use is received. This is useful for MariaDB-specific testing scenarios as well as to support applications that are hardcoding to MariaDB-only concepts. As MariaDB and MySQL featuresets and usage patterns continue to diverge, this pattern may become more prominent.

    References: [#5496](https://www.sqlalchemy.org/trac/ticket/5496)

*   **[mysql] [usecase]**

    Added support for use of the `Sequence` construct with MariaDB 10.3 and greater, as this is now supported by this database. The construct integrates with the `Table` object in the same way that it does for other databases like PostgreSQL and Oracle; if is present on the integer primary key “autoincrement” column, it is used to generate defaults. For backwards compatibility, to support a `Table` that has a `Sequence` on it to support sequence only databases like Oracle, while still not having the sequence fire off for MariaDB, the optional=True flag should be set, which indicates the sequence should only be used to generate the primary key if the target database offers no other option.

    See also

    Added Sequence support for MariaDB 10.3

    References: [#4976](https://www.sqlalchemy.org/trac/ticket/4976)

*   **[mysql] [bug]**

    The MySQL and MariaDB dialects now query from the information_schema.tables system view in order to determine if a particular table exists or not. Previously, the “DESCRIBE” command was used with an exception catch to detect non-existent, which would have the undesirable effect of emitting a ROLLBACK on the connection. There appeared to be legacy encoding issues which prevented the use of “SHOW TABLES”, for this, but as MySQL support is now at 5.0.2 or above due to [#4189](https://www.sqlalchemy.org/trac/ticket/4189), the information_schema tables are now available in all cases.

*   **[mysql] [bug]**

    The “skip_locked” keyword used with `with_for_update()` will render “SKIP LOCKED” on all MySQL backends, meaning it will fail for MySQL less than version 8 and on current MariaDB backends. This is because those backends do not support “SKIP LOCKED” or any equivalent, so this error should not be silently ignored. This is upgraded from a warning in the 1.3 series.

    References: [#5568](https://www.sqlalchemy.org/trac/ticket/5568)

*   **[mysql] [bug]**

    MySQL dialect’s server_version_info tuple is now all numeric. String tokens like “MariaDB” are no longer present so that numeric comparison works in all cases. The .is_mariadb flag on the dialect should be consulted for whether or not mariadb was detected. Additionally removed structures meant to support extremely old MySQL versions 3.x and 4.x; the minimum MySQL version supported is now version 5.0.2.

    References: [#4189](https://www.sqlalchemy.org/trac/ticket/4189)

*   **[mysql] [deprecated]**

    The OurSQL dialect is deprecated.

    References: [#5189](https://www.sqlalchemy.org/trac/ticket/5189)

*   **[mysql] [removed]**

    Remove deprecated dialect `mysql+gaerdbms` that has been deprecated since version 1.0\. Use the MySQLdb dialect directly.

    Remove deprecated parameter `quoting` from `ENUM` and `SET` in the `mysql` dialect. The values passed to the enum or the set are quoted by SQLAlchemy when needed automatically.

    References: [#4643](https://www.sqlalchemy.org/trac/ticket/4643)

### sqlite

*   **[sqlite] [change]**

    Dropped support for right-nested join rewriting to support old SQLite versions prior to 3.7.16, released in 2013\. It is expected that all modern Python versions among those now supported should all include much newer versions of SQLite.

    See also

    Removed “join rewriting” logic from SQLite dialect; updated imports

    References: [#4895](https://www.sqlalchemy.org/trac/ticket/4895)

### mssql

*   **[mssql] [feature] [sql]**

    Added support for the `JSON` datatype on the SQL Server dialect using the `JSON` implementation, which implements SQL Server’s JSON functionality against the `NVARCHAR(max)` datatype as per SQL Server documentation. Implementation courtesy Gord Thompson.

    References: [#4384](https://www.sqlalchemy.org/trac/ticket/4384)

*   **[mssql] [feature]**

    Added support for “CREATE SEQUENCE” and full `Sequence` support for Microsoft SQL Server. This removes the deprecated feature of using `Sequence` objects to manipulate IDENTITY characteristics which should now be performed using `mssql_identity_start` and `mssql_identity_increment` as documented at Auto Increment Behavior / IDENTITY Columns. The change includes a new parameter `Sequence.data_type` to accommodate SQL Server’s choice of datatype, which for that backend includes INTEGER, BIGINT, and DECIMAL(n, 0). The default starting value for SQL Server’s version of `Sequence` has been set at 1; this default is now emitted within the CREATE SEQUENCE DDL for all backends.

    See also

    Added Sequence support distinct from IDENTITY to SQL Server

    References: [#4235](https://www.sqlalchemy.org/trac/ticket/4235), [#4633](https://www.sqlalchemy.org/trac/ticket/4633)

*   **[mssql] [usecase] [postgresql] [reflection] [schema]**

    Improved support for covering indexes (with INCLUDE columns). Added the ability for postgresql to render CREATE INDEX statements with an INCLUDE clause from Core. Index reflection also report INCLUDE columns separately for both mssql and postgresql (11+).

    References: [#4458](https://www.sqlalchemy.org/trac/ticket/4458)

*   **[mssql] [usecase] [postgresql]**

    Added support for inspection / reflection of partial indexes / filtered indexes, i.e. those which use the `mssql_where` or `postgresql_where` parameters, with `Index`. The entry is both part of the dictionary returned by `Inspector.get_indexes()` as well as part of a reflected `Index` construct that was reflected. Pull request courtesy Ramon Williams.

    References: [#4966](https://www.sqlalchemy.org/trac/ticket/4966)

*   **[mssql] [usecase] [reflection]**

    Added support for reflection of temporary tables with the SQL Server dialect. Table names that are prefixed by a pound sign “#” are now introspected from the MSSQL “tempdb” system catalog.

    References: [#5506](https://www.sqlalchemy.org/trac/ticket/5506)

*   **[mssql] [change]**

    SQL Server OFFSET and FETCH keywords are now used for limit/offset, rather than using a window function, for SQL Server versions 11 and higher. TOP is still used for a query that features only LIMIT. Pull request courtesy Elkin.

    References: [#5084](https://www.sqlalchemy.org/trac/ticket/5084)

*   **[mssql] [bug] [schema]**

    Fixed an issue where `sqlalchemy.engine.reflection.has_table()` always returned `False` for temporary tables.

    References: [#5597](https://www.sqlalchemy.org/trac/ticket/5597)

*   **[mssql] [bug]**

    Fixed the base class of the `DATETIMEOFFSET` datatype to be based on the `DateTime` class hierarchy, as this is a datetime-holding datatype.

    References: [#4980](https://www.sqlalchemy.org/trac/ticket/4980)

*   **[mssql] [deprecated]**

    The adodbapi and mxODBC dialects are deprecated.

    References: [#5189](https://www.sqlalchemy.org/trac/ticket/5189)

*   **[mssql]**

    The mssql dialect will assume that at least MSSQL 2005 is used. There is no hard exception raised if a previous version is detected, but operations may fail for older versions.

*   **[mssql] [reflection]**

    As part of the support for reflecting `Identity` objects, the method `Inspector.get_columns()` no longer returns `mssql_identity_start` and `mssql_identity_increment` as part of the `dialect_options`. Use the information in the `identity` key instead.

    References: [#5527](https://www.sqlalchemy.org/trac/ticket/5527)

*   **[mssql] [engine]**

    Deprecated the `legacy_schema_aliasing` parameter to `sqlalchemy.create_engine()`. This is a long-outdated parameter that has defaulted to False since version 1.1.

    References: [#4809](https://www.sqlalchemy.org/trac/ticket/4809)

### oracle

*   **[oracle] [usecase]**

    The max_identifier_length for the Oracle dialect is now 128 characters by default, unless compatibility version less than 12.2 upon first connect, in which case the legacy length of 30 characters is used. This is a continuation of the issue as committed to the 1.3 series which adds max identifier length detection upon first connect as well as warns for the change in Oracle server.

    See also

    Max Identifier Lengths - in the Oracle dialect documentation

    References: [#4857](https://www.sqlalchemy.org/trac/ticket/4857)

*   **[oracle] [change]**

    The LIMIT / OFFSET scheme used in Oracle now makes use of named subqueries rather than unnamed subqueries when it transparently rewrites a SELECT statement to one that uses a subquery that includes ROWNUM. The change is part of a larger change where unnamed subqueries are no longer directly supported by Core, as well as to modernize the internal use of the select() construct within the Oracle dialect.

*   **[oracle] [bug]**

    Correctly render `Sequence` and `Identity` column options `nominvalue` and `nomaxvalue` as `NOMAXVALUE` and ``NOMINVALUE` on oracle database.

*   **[oracle] [bug]**

    The `INTERVAL` class of the Oracle dialect is now correctly a subclass of the abstract version of `Interval` as well as the correct “emulated” base class, which allows for correct behavior under both native and non-native modes; previously it was only based on `TypeEngine`.

    References: [#4971](https://www.sqlalchemy.org/trac/ticket/4971)

### misc

*   **[deprecated] [firebird]**

    The Firebird dialect is deprecated, as there is now a 3rd party dialect that supports this database.

    References: [#5189](https://www.sqlalchemy.org/trac/ticket/5189)

*   **[misc] [deprecated]**

    The Sybase dialect is deprecated.

    References: [#5189](https://www.sqlalchemy.org/trac/ticket/5189)

## 1.4.53

no release date

## 1.4.52

Released: March 4, 2024

### orm

*   **[orm] [bug]**

    Fixed bug where ORM `with_loader_criteria()` would not apply itself to a `Select.join()` where the ON clause were given as a plain SQL comparison, rather than as a relationship target or similar.

    This is a backport of the same issue fixed in version 2.0 for 2.0.22.

    References: [#10365](https://www.sqlalchemy.org/trac/ticket/10365)

### orm

*   **[orm] [bug]**

    Fixed bug where ORM `with_loader_criteria()` would not apply itself to a `Select.join()` where the ON clause were given as a plain SQL comparison, rather than as a relationship target or similar.

    This is a backport of the same issue fixed in version 2.0 for 2.0.22.

    References: [#10365](https://www.sqlalchemy.org/trac/ticket/10365)

## 1.4.51

Released: January 2, 2024

### orm

*   **[orm] [bug]**

    Improved a fix first implemented for [#3208](https://www.sqlalchemy.org/trac/ticket/3208) released in version 0.9.8, where the registry of classes used internally by declarative could be subject to a race condition in the case where individual mapped classes are being garbage collected at the same time while new mapped classes are being constructed, as can happen in some test suite configurations or dynamic class creation environments. In addition to the weakref check already added, the list of items being iterated is also copied first to avoid “list changed while iterating” errors. Pull request courtesy Yilei Yang.

    References: [#10782](https://www.sqlalchemy.org/trac/ticket/10782)

### asyncio

*   **[asyncio] [bug]**

    Fixed critical issue in asyncio version of the connection pool where calling `AsyncEngine.dispose()` would produce a new connection pool that did not fully re-establish the use of asyncio-compatible mutexes, leading to the use of a plain `threading.Lock()` which would then cause deadlocks in an asyncio context when using concurrency features like `asyncio.gather()`.

    References: [#10813](https://www.sqlalchemy.org/trac/ticket/10813)

### mysql

*   **[mysql] [bug]**

    Fixed regression introduced by the fix in ticket [#10492](https://www.sqlalchemy.org/trac/ticket/10492) when using pool pre-ping with PyMySQL version older than 1.0.

    References: [#10650](https://www.sqlalchemy.org/trac/ticket/10650)

### orm

*   **[orm] [bug]**

    Improved a fix first implemented for [#3208](https://www.sqlalchemy.org/trac/ticket/3208) released in version 0.9.8, where the registry of classes used internally by declarative could be subject to a race condition in the case where individual mapped classes are being garbage collected at the same time while new mapped classes are being constructed, as can happen in some test suite configurations or dynamic class creation environments. In addition to the weakref check already added, the list of items being iterated is also copied first to avoid “list changed while iterating” errors. Pull request courtesy Yilei Yang.

    References: [#10782](https://www.sqlalchemy.org/trac/ticket/10782)

### asyncio

*   **[asyncio] [bug]**

    Fixed critical issue in asyncio version of the connection pool where calling `AsyncEngine.dispose()` would produce a new connection pool that did not fully re-establish the use of asyncio-compatible mutexes, leading to the use of a plain `threading.Lock()` which would then cause deadlocks in an asyncio context when using concurrency features like `asyncio.gather()`.

    References: [#10813](https://www.sqlalchemy.org/trac/ticket/10813)

### mysql

*   **[mysql] [bug]**

    Fixed regression introduced by the fix in ticket [#10492](https://www.sqlalchemy.org/trac/ticket/10492) when using pool pre-ping with PyMySQL version older than 1.0.

    References: [#10650](https://www.sqlalchemy.org/trac/ticket/10650)

## 1.4.50

Released: October 29, 2023

### orm

*   **[orm] [bug]**

    Fixed fundamental issue which prevented some forms of ORM “annotations” from taking place for subqueries which made use of `Select.join()` against a relationship target. These annotations are used whenever a subquery is used in special situations such as within `PropComparator.and_()` and other ORM-specific scenarios.

    References: [#10223](https://www.sqlalchemy.org/trac/ticket/10223)

### sql

*   **[sql] [bug]**

    Fixed issue where using the same bound parameter more than once with `literal_execute=True` in some combinations with other literal rendering parameters would cause the wrong values to render due to an iteration issue.

    References: [#10142](https://www.sqlalchemy.org/trac/ticket/10142)

*   **[sql] [bug]**

    Fixed issue where unpickling of a `Column` or other `ColumnElement` would fail to restore the correct “comparator” object, which is used to generate SQL expressions specific to the type object.

    References: [#10213](https://www.sqlalchemy.org/trac/ticket/10213)

### schema

*   **[schema] [bug]**

    Modified the rendering of the Oracle only `Identity.order` parameter that’s part of both `Sequence` and `Identity` to only take place for the Oracle backend, and not other backends such as that of PostgreSQL. A future release will rename the `Identity.order`, `Sequence.order` and `Identity.on_null` parameters to Oracle-specific names, deprecating the old names, these parameters only apply to Oracle.

    References: [#10207](https://www.sqlalchemy.org/trac/ticket/10207)

### mysql

*   **[mysql] [usecase]**

    Updated aiomysql dialect since the dialect appears to be maintained again. Re-added to the ci testing using version 0.2.0.

*   **[mysql] [bug]**

    Repaired a new incompatibility in the MySQL “pre-ping” routine where the `False` argument passed to `connection.ping()`, which is intended to disable an unwanted “automatic reconnect” feature, is being deprecated in MySQL drivers and backends, and is producing warnings for some versions of MySQL’s native client drivers. It’s removed for mysqlclient, whereas for PyMySQL and drivers based on PyMySQL, the parameter will be deprecated and removed at some point, so API introspection is used to future proof against these various stages of removal.

    References: [#10492](https://www.sqlalchemy.org/trac/ticket/10492)

### mssql

*   **[mssql] [bug] [reflection]**

    Fixed issue where identity column reflection would fail for a bigint column with a large identity start value (more than 18 digits).

    References: [#10504](https://www.sqlalchemy.org/trac/ticket/10504)

### orm

*   **[orm] [bug]**

    Fixed fundamental issue which prevented some forms of ORM “annotations” from taking place for subqueries which made use of `Select.join()` against a relationship target. These annotations are used whenever a subquery is used in special situations such as within `PropComparator.and_()` and other ORM-specific scenarios.

    References: [#10223](https://www.sqlalchemy.org/trac/ticket/10223)

### sql

*   **[sql] [bug]**

    Fixed issue where using the same bound parameter more than once with `literal_execute=True` in some combinations with other literal rendering parameters would cause the wrong values to render due to an iteration issue.

    References: [#10142](https://www.sqlalchemy.org/trac/ticket/10142)

*   **[sql] [bug]**

    Fixed issue where unpickling of a `Column` or other `ColumnElement` would fail to restore the correct “comparator” object, which is used to generate SQL expressions specific to the type object.

    References: [#10213](https://www.sqlalchemy.org/trac/ticket/10213)

### schema

*   **[schema] [bug]**

    Modified the rendering of the Oracle only `Identity.order` parameter that’s part of both `Sequence` and `Identity` to only take place for the Oracle backend, and not other backends such as that of PostgreSQL. A future release will rename the `Identity.order`, `Sequence.order` and `Identity.on_null` parameters to Oracle-specific names, deprecating the old names, these parameters only apply to Oracle.

    References: [#10207](https://www.sqlalchemy.org/trac/ticket/10207)

### mysql

*   **[mysql] [usecase]**

    Updated aiomysql dialect since the dialect appears to be maintained again. Re-added to the ci testing using version 0.2.0.

*   **[mysql] [bug]**

    Repaired a new incompatibility in the MySQL “pre-ping” routine where the `False` argument passed to `connection.ping()`, which is intended to disable an unwanted “automatic reconnect” feature, is being deprecated in MySQL drivers and backends, and is producing warnings for some versions of MySQL’s native client drivers. It’s removed for mysqlclient, whereas for PyMySQL and drivers based on PyMySQL, the parameter will be deprecated and removed at some point, so API introspection is used to future proof against these various stages of removal.

    References: [#10492](https://www.sqlalchemy.org/trac/ticket/10492)

### mssql

*   **[mssql] [bug] [reflection]**

    Fixed issue where identity column reflection would fail for a bigint column with a large identity start value (more than 18 digits).

    References: [#10504](https://www.sqlalchemy.org/trac/ticket/10504)

## 1.4.49

Released: July 5, 2023

### platform

*   **[platform] [usecase]**

    Compatibility improvements to work fully with Python 3.12

### sql

*   **[sql] [bug]**

    Fixed issue where the `ColumnOperators.regexp_match()` when using “flags” would not produce a “stable” cache key, that is, the cache key would keep changing each time causing cache pollution. The same issue existed for `ColumnOperators.regexp_replace()` with both the flags and the actual replacement expression. The flags are now represented as fixed modifier strings rendered as safestrings rather than bound parameters, and the replacement expression is established within the primary portion of the “binary” element so that it generates an appropriate cache key.

    Note that as part of this change, the `ColumnOperators.regexp_match.flags` and `ColumnOperators.regexp_replace.flags` have been modified to render as literal strings only, whereas previously they were rendered as full SQL expressions, typically bound parameters. These parameters should always be passed as plain Python strings and not as SQL expression constructs; it’s not expected that SQL expression constructs were used in practice for this parameter, so this is a backwards-incompatible change.

    The change also modifies the internal structure of the expression generated, for `ColumnOperators.regexp_replace()` with or without flags, and for `ColumnOperators.regexp_match()` with flags. Third party dialects which may have implemented regexp implementations of their own (no such dialects could be located in a search, so impact is expected to be low) would need to adjust the traversal of the structure to accommodate.

    References: [#10042](https://www.sqlalchemy.org/trac/ticket/10042)

*   **[sql] [bug]**

    Fixed issue in mostly-internal `CacheKey` construct where the `__ne__()` operator were not properly implemented, leading to nonsensical results when comparing `CacheKey` instances to each other.

### extensions

*   **[extensions] [bug]**

    Fixed issue in mypy plugin for use with mypy 1.4.

### platform

*   **[platform] [usecase]**

    Compatibility improvements to work fully with Python 3.12

### sql

*   **[sql] [bug]**

    Fixed issue where the `ColumnOperators.regexp_match()` when using “flags” would not produce a “stable” cache key, that is, the cache key would keep changing each time causing cache pollution. The same issue existed for `ColumnOperators.regexp_replace()` with both the flags and the actual replacement expression. The flags are now represented as fixed modifier strings rendered as safestrings rather than bound parameters, and the replacement expression is established within the primary portion of the “binary” element so that it generates an appropriate cache key.

    Note that as part of this change, the `ColumnOperators.regexp_match.flags` and `ColumnOperators.regexp_replace.flags` have been modified to render as literal strings only, whereas previously they were rendered as full SQL expressions, typically bound parameters. These parameters should always be passed as plain Python strings and not as SQL expression constructs; it’s not expected that SQL expression constructs were used in practice for this parameter, so this is a backwards-incompatible change.

    The change also modifies the internal structure of the expression generated, for `ColumnOperators.regexp_replace()` with or without flags, and for `ColumnOperators.regexp_match()` with flags. Third party dialects which may have implemented regexp implementations of their own (no such dialects could be located in a search, so impact is expected to be low) would need to adjust the traversal of the structure to accommodate.

    References: [#10042](https://www.sqlalchemy.org/trac/ticket/10042)

*   **[sql] [bug]**

    Fixed issue in mostly-internal `CacheKey` construct where the `__ne__()` operator were not properly implemented, leading to nonsensical results when comparing `CacheKey` instances to each other.

### extensions

*   **[extensions] [bug]**

    Fixed issue in mypy plugin for use with mypy 1.4.

## 1.4.48

Released: April 30, 2023

### orm

*   **[orm] [bug]**

    Fixed critical caching issue where the combination of `aliased()` and `hybrid_property()` expression compositions would cause a cache key mismatch, leading to cache keys that held onto the actual `aliased()` object while also not matching that of equivalent constructs, filling up the cache.

    References: [#9728](https://www.sqlalchemy.org/trac/ticket/9728)

*   **[orm] [bug]**

    Fixed bug where various ORM-specific getters such as `ORMExecuteState.is_column_load`, `ORMExecuteState.is_relationship_load`, `ORMExecuteState.loader_strategy_path` etc. would throw an `AttributeError` if the SQL statement itself were a “compound select” such as a UNION.

    References: [#9634](https://www.sqlalchemy.org/trac/ticket/9634)

*   **[orm] [bug]**

    Fixed endless loop which could occur when using “relationship to aliased class” feature and also indicating a recursive eager loader such as `lazy="selectinload"` in the loader, in combination with another eager loader on the opposite side. The check for cycles has been fixed to include aliased class relationships.

    References: [#9590](https://www.sqlalchemy.org/trac/ticket/9590)

### orm

*   **[orm] [bug]**

    Fixed critical caching issue where the combination of `aliased()` and `hybrid_property()` expression compositions would cause a cache key mismatch, leading to cache keys that held onto the actual `aliased()` object while also not matching that of equivalent constructs, filling up the cache.

    References: [#9728](https://www.sqlalchemy.org/trac/ticket/9728)

*   **[orm] [bug]**

    Fixed bug where various ORM-specific getters such as `ORMExecuteState.is_column_load`, `ORMExecuteState.is_relationship_load`, `ORMExecuteState.loader_strategy_path` etc. would throw an `AttributeError` if the SQL statement itself were a “compound select” such as a UNION.

    References: [#9634](https://www.sqlalchemy.org/trac/ticket/9634)

*   **[orm] [bug]**

    Fixed endless loop which could occur when using “relationship to aliased class” feature and also indicating a recursive eager loader such as `lazy="selectinload"` in the loader, in combination with another eager loader on the opposite side. The check for cycles has been fixed to include aliased class relationships.

    References: [#9590](https://www.sqlalchemy.org/trac/ticket/9590)

## 1.4.47

Released: March 18, 2023

### sql

*   **[sql] [bug]**

    Fixed bug / regression where using `bindparam()` with the same name as a column in the `Update.values()` method of `Update`, as well as the `Insert.values()` method of `Insert` in 2.0 only, would in some cases silently fail to honor the SQL expression in which the parameter were presented, replacing the expression with a new parameter of the same name and discarding any other elements of the SQL expression, such as SQL functions, etc. The specific case would be statements that were constructed against ORM entities rather than plain `Table` instances, but would occur if the statement were invoked with a `Session` or a `Connection`.

    `Update` part of the issue was present in both 2.0 and 1.4 and is backported to 1.4.

    References: [#9075](https://www.sqlalchemy.org/trac/ticket/9075)

*   **[sql] [bug]**

    Fixed stringify for a the `CreateSchema` and `DropSchema` DDL constructs, which would fail with an `AttributeError` when stringified without a dialect.

    References: [#7664](https://www.sqlalchemy.org/trac/ticket/7664)

*   **[sql] [bug]**

    Fixed critical SQL caching issue where use of the `Operators.op()` custom operator function would not produce an appropriate cache key, leading to reduce the effectiveness of the SQL cache.

    References: [#9506](https://www.sqlalchemy.org/trac/ticket/9506)

### mypy

*   **[mypy] [bug]**

    Adjustments made to the mypy plugin to accommodate for some potential changes being made for issue #236 sqlalchemy2-stubs when using SQLAlchemy 1.4\. These changes are being kept in sync within SQLAlchemy 2.0. The changes are also backwards compatible with older versions of sqlalchemy2-stubs.

*   **[mypy] [bug]**

    Fixed crash in mypy plugin which could occur on both 1.4 and 2.0 versions if a decorator for the `mapped()` decorator were used that was referenced in an expression with more than two components (e.g. `@Backend.mapper_registry.mapped`). This scenario is now ignored; when using the plugin, the decorator expression needs to be two components (i.e. `@reg.mapped`).

    References: [#9102](https://www.sqlalchemy.org/trac/ticket/9102)

### postgresql

*   **[postgresql] [bug]**

    Added support to the asyncpg dialect to return the `cursor.rowcount` value for SELECT statements when available. While this is not a typical use for `cursor.rowcount`, the other PostgreSQL dialects generally provide this value. Pull request courtesy Michael Gorven.

    References: [#9048](https://www.sqlalchemy.org/trac/ticket/9048)

### mysql

*   **[mysql] [usecase]**

    Added support to MySQL index reflection to correctly reflect the `mysql_length` dictionary, which previously was being ignored.

    References: [#9047](https://www.sqlalchemy.org/trac/ticket/9047)

### mssql

*   **[mssql] [bug]**

    Fixed bug where a schema name given with brackets, but no dots inside the name, for parameters such as `Table.schema` would not be interpreted within the context of the SQL Server dialect’s documented behavior of interpreting explicit brackets as token delimiters, first added in 1.2 for #2626, when referring to the schema name in reflection operations. The original assumption for #2626’s behavior was that the special interpretation of brackets was only significant if dots were present, however in practice, the brackets are not included as part of the identifier name for all SQL rendering operations since these are not valid characters within regular or delimited identifiers. Pull request courtesy Shan.

    References: [#9133](https://www.sqlalchemy.org/trac/ticket/9133)

### oracle

*   **[oracle] [bug]**

    Added `ROWID` to reflected types as this type may be used in a “CREATE TABLE” statement.

    References: [#5047](https://www.sqlalchemy.org/trac/ticket/5047)

### sql

*   **[sql] [bug]**

    Fixed bug / regression where using `bindparam()` with the same name as a column in the `Update.values()` method of `Update`, as well as the `Insert.values()` method of `Insert` in 2.0 only, would in some cases silently fail to honor the SQL expression in which the parameter were presented, replacing the expression with a new parameter of the same name and discarding any other elements of the SQL expression, such as SQL functions, etc. The specific case would be statements that were constructed against ORM entities rather than plain `Table` instances, but would occur if the statement were invoked with a `Session` or a `Connection`.

    `Update` part of the issue was present in both 2.0 and 1.4 and is backported to 1.4.

    References: [#9075](https://www.sqlalchemy.org/trac/ticket/9075)

*   **[sql] [bug]**

    Fixed stringify for a the `CreateSchema` and `DropSchema` DDL constructs, which would fail with an `AttributeError` when stringified without a dialect.

    References: [#7664](https://www.sqlalchemy.org/trac/ticket/7664)

*   **[sql] [bug]**

    Fixed critical SQL caching issue where use of the `Operators.op()` custom operator function would not produce an appropriate cache key, leading to reduce the effectiveness of the SQL cache.

    References: [#9506](https://www.sqlalchemy.org/trac/ticket/9506)

### mypy

*   **[mypy] [bug]**

    Adjustments made to the mypy plugin to accommodate for some potential changes being made for issue #236 sqlalchemy2-stubs when using SQLAlchemy 1.4\. These changes are being kept in sync within SQLAlchemy 2.0. The changes are also backwards compatible with older versions of sqlalchemy2-stubs.

*   **[mypy] [bug]**

    Fixed crash in mypy plugin which could occur on both 1.4 and 2.0 versions if a decorator for the `mapped()` decorator were used that was referenced in an expression with more than two components (e.g. `@Backend.mapper_registry.mapped`). This scenario is now ignored; when using the plugin, the decorator expression needs to be two components (i.e. `@reg.mapped`).

    References: [#9102](https://www.sqlalchemy.org/trac/ticket/9102)

### postgresql

*   **[postgresql] [bug]**

    Added support to the asyncpg dialect to return the `cursor.rowcount` value for SELECT statements when available. While this is not a typical use for `cursor.rowcount`, the other PostgreSQL dialects generally provide this value. Pull request courtesy Michael Gorven.

    References: [#9048](https://www.sqlalchemy.org/trac/ticket/9048)

### mysql

*   **[mysql] [usecase]**

    Added support to MySQL index reflection to correctly reflect the `mysql_length` dictionary, which previously was being ignored.

    References: [#9047](https://www.sqlalchemy.org/trac/ticket/9047)

### mssql

*   **[mssql] [bug]**

    Fixed bug where a schema name given with brackets, but no dots inside the name, for parameters such as `Table.schema` would not be interpreted within the context of the SQL Server dialect’s documented behavior of interpreting explicit brackets as token delimiters, first added in 1.2 for #2626, when referring to the schema name in reflection operations. The original assumption for #2626’s behavior was that the special interpretation of brackets was only significant if dots were present, however in practice, the brackets are not included as part of the identifier name for all SQL rendering operations since these are not valid characters within regular or delimited identifiers. Pull request courtesy Shan.

    References: [#9133](https://www.sqlalchemy.org/trac/ticket/9133)

### oracle

*   **[oracle] [bug]**

    Added `ROWID` to reflected types as this type may be used in a “CREATE TABLE” statement.

    References: [#5047](https://www.sqlalchemy.org/trac/ticket/5047)

## 1.4.46

Released: January 3, 2023

### general

*   **[general] [change]**

    A new deprecation “uber warning” is now emitted at runtime the first time any SQLAlchemy 2.0 deprecation warning would normally be emitted, but the `SQLALCHEMY_WARN_20` environment variable is not set. The warning emits only once at most, before setting a boolean to prevent it from emitting a second time.

    This deprecation warning intends to notify users who may not have set an appropriate constraint in their requirements files to block against a surprise SQLAlchemy 2.0 upgrade and also alert that the SQLAlchemy 2.0 upgrade process is available, as the first full 2.0 release is expected very soon. The deprecation warning can be silenced by setting the environment variable `SQLALCHEMY_SILENCE_UBER_WARNING` to `"1"`.

    See also

    SQLAlchemy 2.0 - Major Migration Guide

    References: [#8983](https://www.sqlalchemy.org/trac/ticket/8983)

*   **[general] [bug]**

    Fixed regression where the base compat module was calling upon `platform.architecture()` in order to detect some system properties, which results in an over-broad system call against the system-level `file` call that is unavailable under some circumstances, including within some secure environment configurations.

    References: [#8995](https://www.sqlalchemy.org/trac/ticket/8995)

### orm

*   **[orm] [bug]**

    Fixed issue in the internal SQL traversal for DML statements like `Update` and `Delete` which would cause among other potential issues, a specific issue using lambda statements with the ORM update/delete feature.

    References: [#9033](https://www.sqlalchemy.org/trac/ticket/9033)

### engine

*   **[engine] [bug]**

    Fixed a long-standing race condition in the connection pool which could occur under eventlet/gevent monkeypatching schemes in conjunction with the use of eventlet/gevent `Timeout` conditions, where a connection pool checkout that’s interrupted due to the timeout would fail to clean up the failed state, causing the underlying connection record and sometimes the database connection itself to “leak”, leaving the pool in an invalid state with unreachable entries. This issue was first identified and fixed in SQLAlchemy 1.2 for [#4225](https://www.sqlalchemy.org/trac/ticket/4225), however the failure modes detected in that fix failed to accommodate for `BaseException`, rather than `Exception`, which prevented eventlet/gevent `Timeout` from being caught. In addition, a block within initial pool connect has also been identified and hardened with a `BaseException` -> “clean failed connect” block to accommodate for the same condition in this location. Big thanks to Github user @niklaus for their tenacious efforts in identifying and describing this intricate issue.

    References: [#8974](https://www.sqlalchemy.org/trac/ticket/8974)

### sql

*   **[sql] [bug]**

    Added parameter `FunctionElement.column_valued.joins_implicitly`, which is useful in preventing the “cartesian product” warning when making use of table-valued or column-valued functions. This parameter was already introduced for `FunctionElement.table_valued()` in [#7845](https://www.sqlalchemy.org/trac/ticket/7845), however it failed to be added for `FunctionElement.column_valued()` as well.

    References: [#9009](https://www.sqlalchemy.org/trac/ticket/9009)

*   **[sql] [bug]**

    Fixed bug where SQL compilation would fail (assertion fail in 2.0, NoneType error in 1.4) when using an expression whose type included `TypeEngine.bind_expression()`, in the context of an “expanding” (i.e. “IN”) parameter in conjunction with the `literal_binds` compiler parameter.

    References: [#8989](https://www.sqlalchemy.org/trac/ticket/8989)

*   **[sql] [bug]**

    Fixed issue in lambda SQL feature where the calculated type of a literal value would not take into account the type coercion rules of the “compared to type”, leading to a lack of typing information for SQL expressions, such as comparisons to `JSON` elements and similar.

    References: [#9029](https://www.sqlalchemy.org/trac/ticket/9029)

### postgresql

*   **[postgresql] [usecase]**

    Added the PostgreSQL type `MACADDR8`. Pull request courtesy of Asim Farooq.

    References: [#8393](https://www.sqlalchemy.org/trac/ticket/8393)

*   **[postgresql] [bug]**

    Fixed bug where the PostgreSQL `Insert.on_conflict_do_update.constraint` parameter would accept an `Index` object, however would not expand this index out into its individual index expressions, instead rendering its name in an ON CONFLICT ON CONSTRAINT clause, which is not accepted by PostgreSQL; the “constraint name” form only accepts unique or exclude constraint names. The parameter continues to accept the index but now expands it out into its component expressions for the render.

    References: [#9023](https://www.sqlalchemy.org/trac/ticket/9023)

### sqlite

*   **[sqlite] [bug]**

    Fixed regression caused by new support for reflection of partial indexes on SQLite added in 1.4.45 for [#8804](https://www.sqlalchemy.org/trac/ticket/8804), where the `index_list` pragma command in very old versions of SQLite (possibly prior to 3.8.9) does not return the current expected number of columns, leading to exceptions raised when reflecting tables and indexes.

    References: [#8969](https://www.sqlalchemy.org/trac/ticket/8969)

### tests

*   **[tests] [bug]**

    Fixed issue in tox.ini file where changes in the tox 4.0 series to the format of “passenv” caused tox to not function correctly, in particular raising an error as of tox 4.0.6.

*   **[tests] [bug]**

    Added new exclusion rule for third party dialects called `unusual_column_name_characters`, which can be “closed” for third party dialects that don’t support column names with unusual characters such as dots, slashes, or percent signs in them, even if the name is properly quoted.

    References: [#9002](https://www.sqlalchemy.org/trac/ticket/9002)

### general

*   **[general] [change]**

    A new deprecation “uber warning” is now emitted at runtime the first time any SQLAlchemy 2.0 deprecation warning would normally be emitted, but the `SQLALCHEMY_WARN_20` environment variable is not set. The warning emits only once at most, before setting a boolean to prevent it from emitting a second time.

    This deprecation warning intends to notify users who may not have set an appropriate constraint in their requirements files to block against a surprise SQLAlchemy 2.0 upgrade and also alert that the SQLAlchemy 2.0 upgrade process is available, as the first full 2.0 release is expected very soon. The deprecation warning can be silenced by setting the environment variable `SQLALCHEMY_SILENCE_UBER_WARNING` to `"1"`.

    See also

    SQLAlchemy 2.0 - Major Migration Guide

    References: [#8983](https://www.sqlalchemy.org/trac/ticket/8983)

*   **[general] [bug]**

    Fixed regression where the base compat module was calling upon `platform.architecture()` in order to detect some system properties, which results in an over-broad system call against the system-level `file` call that is unavailable under some circumstances, including within some secure environment configurations.

    References: [#8995](https://www.sqlalchemy.org/trac/ticket/8995)

### orm

*   **[orm] [bug]**

    Fixed issue in the internal SQL traversal for DML statements like `Update` and `Delete` which would cause among other potential issues, a specific issue using lambda statements with the ORM update/delete feature.

    References: [#9033](https://www.sqlalchemy.org/trac/ticket/9033)

### engine

*   **[engine] [bug]**

    Fixed a long-standing race condition in the connection pool which could occur under eventlet/gevent monkeypatching schemes in conjunction with the use of eventlet/gevent `Timeout` conditions, where a connection pool checkout that’s interrupted due to the timeout would fail to clean up the failed state, causing the underlying connection record and sometimes the database connection itself to “leak”, leaving the pool in an invalid state with unreachable entries. This issue was first identified and fixed in SQLAlchemy 1.2 for [#4225](https://www.sqlalchemy.org/trac/ticket/4225), however the failure modes detected in that fix failed to accommodate for `BaseException`, rather than `Exception`, which prevented eventlet/gevent `Timeout` from being caught. In addition, a block within initial pool connect has also been identified and hardened with a `BaseException` -> “clean failed connect” block to accommodate for the same condition in this location. Big thanks to Github user @niklaus for their tenacious efforts in identifying and describing this intricate issue.

    References: [#8974](https://www.sqlalchemy.org/trac/ticket/8974)

### sql

*   **[sql] [bug]**

    Added parameter `FunctionElement.column_valued.joins_implicitly`, which is useful in preventing the “cartesian product” warning when making use of table-valued or column-valued functions. This parameter was already introduced for `FunctionElement.table_valued()` in [#7845](https://www.sqlalchemy.org/trac/ticket/7845), however it failed to be added for `FunctionElement.column_valued()` as well.

    References: [#9009](https://www.sqlalchemy.org/trac/ticket/9009)

*   **[sql] [bug]**

    Fixed bug where SQL compilation would fail (assertion fail in 2.0, NoneType error in 1.4) when using an expression whose type included `TypeEngine.bind_expression()`, in the context of an “expanding” (i.e. “IN”) parameter in conjunction with the `literal_binds` compiler parameter.

    References: [#8989](https://www.sqlalchemy.org/trac/ticket/8989)

*   **[sql] [bug]**

    Fixed issue in lambda SQL feature where the calculated type of a literal value would not take into account the type coercion rules of the “compared to type”, leading to a lack of typing information for SQL expressions, such as comparisons to `JSON` elements and similar.

    References: [#9029](https://www.sqlalchemy.org/trac/ticket/9029)

### postgresql

*   **[postgresql] [usecase]**

    Added the PostgreSQL type `MACADDR8`. Pull request courtesy of Asim Farooq.

    References: [#8393](https://www.sqlalchemy.org/trac/ticket/8393)

*   **[postgresql] [bug]**

    Fixed bug where the PostgreSQL `Insert.on_conflict_do_update.constraint` parameter would accept an `Index` object, however would not expand this index out into its individual index expressions, instead rendering its name in an ON CONFLICT ON CONSTRAINT clause, which is not accepted by PostgreSQL; the “constraint name” form only accepts unique or exclude constraint names. The parameter continues to accept the index but now expands it out into its component expressions for the render.

    References: [#9023](https://www.sqlalchemy.org/trac/ticket/9023)

### sqlite

*   **[sqlite] [bug]**

    Fixed regression caused by new support for reflection of partial indexes on SQLite added in 1.4.45 for [#8804](https://www.sqlalchemy.org/trac/ticket/8804), where the `index_list` pragma command in very old versions of SQLite (possibly prior to 3.8.9) does not return the current expected number of columns, leading to exceptions raised when reflecting tables and indexes.

    References: [#8969](https://www.sqlalchemy.org/trac/ticket/8969)

### tests

*   **[tests] [bug]**

    Fixed issue in tox.ini file where changes in the tox 4.0 series to the format of “passenv” caused tox to not function correctly, in particular raising an error as of tox 4.0.6.

*   **[tests] [bug]**

    Added new exclusion rule for third party dialects called `unusual_column_name_characters`, which can be “closed” for third party dialects that don’t support column names with unusual characters such as dots, slashes, or percent signs in them, even if the name is properly quoted.

    References: [#9002](https://www.sqlalchemy.org/trac/ticket/9002)

## 1.4.45

Released: December 10, 2022

### orm

*   **[orm] [bug]**

    Fixed bug where `Session.merge()` would fail to preserve the current loaded contents of relationship attributes that were indicated with the `relationship.viewonly` parameter, thus defeating strategies that use `Session.merge()` to pull fully loaded objects from caches and other similar techniques. In a related change, fixed issue where an object that contains a loaded relationship that was nonetheless configured as `lazy='raise'` on the mapping would fail when passed to `Session.merge()`; checks for “raise” are now suspended within the merge process assuming the `Session.merge.load` parameter remains at its default of `True`.

    Overall, this is a behavioral adjustment to a change introduced in the 1.4 series as of [#4994](https://www.sqlalchemy.org/trac/ticket/4994), which took “merge” out of the set of cascades applied by default to “viewonly” relationships. As “viewonly” relationships aren’t persisted under any circumstances, allowing their contents to transfer during “merge” does not impact the persistence behavior of the target object. This allows `Session.merge()` to correctly suit one of its use cases, that of adding objects to a `Session` that were loaded elsewhere, often for the purposes of restoring from a cache.

    References: [#8862](https://www.sqlalchemy.org/trac/ticket/8862)

*   **[orm] [bug]**

    Fixed issues in `with_expression()` where expressions that were composed of columns that were referenced from the enclosing SELECT would not render correct SQL in some contexts, in the case where the expression had a label name that matched the attribute which used `query_expression()`, even when `query_expression()` had no default expression. For the moment, if the `query_expression()` does have a default expression, that label name is still used for that default, and an additional label with the same name will continue to be ignored. Overall, this case is pretty thorny so further adjustments might be warranted.

    References: [#8881](https://www.sqlalchemy.org/trac/ticket/8881)

### engine

*   **[engine] [bug]**

    Fixed issue where `Result.freeze()` method would not work for textual SQL using either `text()` or `Connection.exec_driver_sql()`.

    References: [#8963](https://www.sqlalchemy.org/trac/ticket/8963)

### sql

*   **[sql] [usecase]**

    An informative re-raise is now thrown in the case where any “literal bindparam” render operation fails, indicating the value itself and the datatype in use, to assist in debugging when literal params are being rendered in a statement.

    References: [#8800](https://www.sqlalchemy.org/trac/ticket/8800)

*   **[sql] [bug]**

    Fixed a series of issues regarding the position and sometimes the identity of rendered bound parameters, such as those used for SQLite, asyncpg, MySQL, Oracle and others. Some compiled forms would not maintain the order of parameters correctly, such as the PostgreSQL `regexp_replace()` function, the “nesting” feature of the `CTE` construct first introduced in [#4123](https://www.sqlalchemy.org/trac/ticket/4123), and selectable tables formed by using the `FunctionElement.column_valued()` method with Oracle.

    References: [#8827](https://www.sqlalchemy.org/trac/ticket/8827)

### asyncio

*   **[asyncio] [bug]**

    Removed non-functional `merge()` method from `AsyncResult`. This method has never worked and was included with `AsyncResult` in error.

    References: [#8952](https://www.sqlalchemy.org/trac/ticket/8952)

### postgresql

*   **[postgresql] [bug]**

    Made an adjustment to how the PostgreSQL dialect considers column types when it reflects columns from a table, to accommodate for alternative backends which may return NULL from the PG `format_type()` function.

    References: [#8748](https://www.sqlalchemy.org/trac/ticket/8748)

### sqlite

*   **[sqlite] [usecase]**

    Added support for the SQLite backend to reflect the “DEFERRABLE” and “INITIALLY” keywords which may be present on a foreign key construct. Pull request courtesy Michael Gorven.

    References: [#8903](https://www.sqlalchemy.org/trac/ticket/8903)

*   **[sqlite] [usecase]**

    Added support for reflection of expression-oriented WHERE criteria included in indexes on the SQLite dialect, in a manner similar to that of the PostgreSQL dialect. Pull request courtesy Tobias Pfeiffer.

    References: [#8804](https://www.sqlalchemy.org/trac/ticket/8804)

*   **[sqlite] [bug]**

    Backported a fix for SQLite reflection of unique constraints in attached schemas, released in 2.0 as a small part of [#4379](https://www.sqlalchemy.org/trac/ticket/4379). Previously, unique constraints in attached schemas would be ignored by SQLite reflection. Pull request courtesy Michael Gorven.

    References: [#8866](https://www.sqlalchemy.org/trac/ticket/8866)

### oracle

*   **[oracle] [bug]**

    Continued fixes for Oracle fix [#8708](https://www.sqlalchemy.org/trac/ticket/8708) released in 1.4.43 where bound parameter names that start with underscores, which are disallowed by Oracle, were still not being properly escaped in all circumstances.

    References: [#8708](https://www.sqlalchemy.org/trac/ticket/8708)

*   **[oracle] [bug]**

    Fixed issue in Oracle compiler where the syntax for `FunctionElement.column_valued()` was incorrect, rendering the name `COLUMN_VALUE` without qualifying the source table correctly.

    References: [#8945](https://www.sqlalchemy.org/trac/ticket/8945)

### orm

*   **[orm] [bug]**

    Fixed bug where `Session.merge()` would fail to preserve the current loaded contents of relationship attributes that were indicated with the `relationship.viewonly` parameter, thus defeating strategies that use `Session.merge()` to pull fully loaded objects from caches and other similar techniques. In a related change, fixed issue where an object that contains a loaded relationship that was nonetheless configured as `lazy='raise'` on the mapping would fail when passed to `Session.merge()`; checks for “raise” are now suspended within the merge process assuming the `Session.merge.load` parameter remains at its default of `True`.

    Overall, this is a behavioral adjustment to a change introduced in the 1.4 series as of [#4994](https://www.sqlalchemy.org/trac/ticket/4994), which took “merge” out of the set of cascades applied by default to “viewonly” relationships. As “viewonly” relationships aren’t persisted under any circumstances, allowing their contents to transfer during “merge” does not impact the persistence behavior of the target object. This allows `Session.merge()` to correctly suit one of its use cases, that of adding objects to a `Session` that were loaded elsewhere, often for the purposes of restoring from a cache.

    References: [#8862](https://www.sqlalchemy.org/trac/ticket/8862)

*   **[orm] [bug]**

    Fixed issues in `with_expression()` where expressions that were composed of columns that were referenced from the enclosing SELECT would not render correct SQL in some contexts, in the case where the expression had a label name that matched the attribute which used `query_expression()`, even when `query_expression()` had no default expression. For the moment, if the `query_expression()` does have a default expression, that label name is still used for that default, and an additional label with the same name will continue to be ignored. Overall, this case is pretty thorny so further adjustments might be warranted.

    References: [#8881](https://www.sqlalchemy.org/trac/ticket/8881)

### engine

*   **[engine] [bug]**

    Fixed issue where `Result.freeze()` method would not work for textual SQL using either `text()` or `Connection.exec_driver_sql()`.

    References: [#8963](https://www.sqlalchemy.org/trac/ticket/8963)

### sql

*   **[sql] [usecase]**

    An informative re-raise is now thrown in the case where any “literal bindparam” render operation fails, indicating the value itself and the datatype in use, to assist in debugging when literal params are being rendered in a statement.

    References: [#8800](https://www.sqlalchemy.org/trac/ticket/8800)

*   **[sql] [bug]**

    Fixed a series of issues regarding the position and sometimes the identity of rendered bound parameters, such as those used for SQLite, asyncpg, MySQL, Oracle and others. Some compiled forms would not maintain the order of parameters correctly, such as the PostgreSQL `regexp_replace()` function, the “nesting” feature of the `CTE` construct first introduced in [#4123](https://www.sqlalchemy.org/trac/ticket/4123), and selectable tables formed by using the `FunctionElement.column_valued()` method with Oracle.

    References: [#8827](https://www.sqlalchemy.org/trac/ticket/8827)

### asyncio

*   **[asyncio] [bug]**

    Removed non-functional `merge()` method from `AsyncResult`. This method has never worked and was included with `AsyncResult` in error.

    References: [#8952](https://www.sqlalchemy.org/trac/ticket/8952)

### postgresql

*   **[postgresql] [bug]**

    Made an adjustment to how the PostgreSQL dialect considers column types when it reflects columns from a table, to accommodate for alternative backends which may return NULL from the PG `format_type()` function.

    References: [#8748](https://www.sqlalchemy.org/trac/ticket/8748)

### sqlite

*   **[sqlite] [usecase]**

    Added support for the SQLite backend to reflect the “DEFERRABLE” and “INITIALLY” keywords which may be present on a foreign key construct. Pull request courtesy Michael Gorven.

    References: [#8903](https://www.sqlalchemy.org/trac/ticket/8903)

*   **[sqlite] [usecase]**

    Added support for reflection of expression-oriented WHERE criteria included in indexes on the SQLite dialect, in a manner similar to that of the PostgreSQL dialect. Pull request courtesy Tobias Pfeiffer.

    References: [#8804](https://www.sqlalchemy.org/trac/ticket/8804)

*   **[sqlite] [bug]**

    Backported a fix for SQLite reflection of unique constraints in attached schemas, released in 2.0 as a small part of [#4379](https://www.sqlalchemy.org/trac/ticket/4379). Previously, unique constraints in attached schemas would be ignored by SQLite reflection. Pull request courtesy Michael Gorven.

    References: [#8866](https://www.sqlalchemy.org/trac/ticket/8866)

### oracle

*   **[oracle] [bug]**

    Continued fixes for Oracle fix [#8708](https://www.sqlalchemy.org/trac/ticket/8708) released in 1.4.43 where bound parameter names that start with underscores, which are disallowed by Oracle, were still not being properly escaped in all circumstances.

    References: [#8708](https://www.sqlalchemy.org/trac/ticket/8708)

*   **[oracle] [bug]**

    Fixed issue in Oracle compiler where the syntax for `FunctionElement.column_valued()` was incorrect, rendering the name `COLUMN_VALUE` without qualifying the source table correctly.

    References: [#8945](https://www.sqlalchemy.org/trac/ticket/8945)

## 1.4.44

Released: November 12, 2022

### sql

*   **[sql] [bug]**

    Fixed critical memory issue identified in cache key generation, where for very large and complex ORM statements that make use of lots of ORM aliases with subqueries, cache key generation could produce excessively large keys that were orders of magnitude bigger than the statement itself. Much thanks to Rollo Konig Brock for their very patient, long term help in finally identifying this issue.

    References: [#8790](https://www.sqlalchemy.org/trac/ticket/8790)

### postgresql

*   **[postgresql] [bug] [mssql]**

    For the PostgreSQL and SQL Server dialects only, adjusted the compiler so that when rendering column expressions in the RETURNING clause, the “non anon” label that’s used in SELECT statements is suggested for SQL expression elements that generate a label; the primary example is a SQL function that may be emitting as part of the column’s type, where the label name should match the column’s name by default. This restores a not-well defined behavior that had changed in version 1.4.21 due to [#6718](https://www.sqlalchemy.org/trac/ticket/6718), [#6710](https://www.sqlalchemy.org/trac/ticket/6710). The Oracle dialect has a different RETURNING implementation and was not affected by this issue. Version 2.0 features an across the board change for its widely expanded support of RETURNING on other backends.

    References: [#8770](https://www.sqlalchemy.org/trac/ticket/8770)

### oracle

*   **[oracle] [bug]**

    Fixed issue in the Oracle dialect where an INSERT statement that used `insert(some_table).values(...).returning(some_table)` against a full `Table` object at once would fail to execute, raising an exception.

### tests

*   **[tests] [bug]**

    Fixed issue where the `--disable-asyncio` parameter to the test suite would fail to not actually run greenlet tests and would also not prevent the suite from using a “wrapping” greenlet for the whole suite. This parameter now ensures that no greenlet or asyncio use will occur within the entire run when set.

    References: [#8793](https://www.sqlalchemy.org/trac/ticket/8793)

*   **[tests] [bug]**

    Adjusted the test suite which tests the Mypy plugin to accommodate for changes in Mypy 0.990 regarding how it handles message output, which affect how sys.path is interpreted when determining if notes and errors should be printed for particular files. The change broke the test suite as the files within the test directory itself no longer produced messaging when run under the mypy API.

### sql

*   **[sql] [bug]**

    Fixed critical memory issue identified in cache key generation, where for very large and complex ORM statements that make use of lots of ORM aliases with subqueries, cache key generation could produce excessively large keys that were orders of magnitude bigger than the statement itself. Much thanks to Rollo Konig Brock for their very patient, long term help in finally identifying this issue.

    References: [#8790](https://www.sqlalchemy.org/trac/ticket/8790)

### postgresql

*   **[postgresql] [bug] [mssql]**

    For the PostgreSQL and SQL Server dialects only, adjusted the compiler so that when rendering column expressions in the RETURNING clause, the “non anon” label that’s used in SELECT statements is suggested for SQL expression elements that generate a label; the primary example is a SQL function that may be emitting as part of the column’s type, where the label name should match the column’s name by default. This restores a not-well defined behavior that had changed in version 1.4.21 due to [#6718](https://www.sqlalchemy.org/trac/ticket/6718), [#6710](https://www.sqlalchemy.org/trac/ticket/6710). The Oracle dialect has a different RETURNING implementation and was not affected by this issue. Version 2.0 features an across the board change for its widely expanded support of RETURNING on other backends.

    References: [#8770](https://www.sqlalchemy.org/trac/ticket/8770)

### oracle

*   **[oracle] [bug]**

    Fixed issue in the Oracle dialect where an INSERT statement that used `insert(some_table).values(...).returning(some_table)` against a full `Table` object at once would fail to execute, raising an exception.

### tests

*   **[tests] [bug]**

    Fixed issue where the `--disable-asyncio` parameter to the test suite would fail to not actually run greenlet tests and would also not prevent the suite from using a “wrapping” greenlet for the whole suite. This parameter now ensures that no greenlet or asyncio use will occur within the entire run when set.

    References: [#8793](https://www.sqlalchemy.org/trac/ticket/8793)

*   **[tests] [bug]**

    Adjusted the test suite which tests the Mypy plugin to accommodate for changes in Mypy 0.990 regarding how it handles message output, which affect how sys.path is interpreted when determining if notes and errors should be printed for particular files. The change broke the test suite as the files within the test directory itself no longer produced messaging when run under the mypy API.

## 1.4.43

Released: November 4, 2022

### orm

*   **[orm] [bug]**

    Fixed issue in joined eager loading where an assertion fail would occur with a particular combination of outer/inner joined eager loads, when eager loading across three mappers where the middle mapper was an inherited subclass mapper.

    References: [#8738](https://www.sqlalchemy.org/trac/ticket/8738)

*   **[orm] [bug]**

    Fixed bug involving `Select` constructs, where combinations of `Select.select_from()` with `Select.join()`, as well as when using `Select.join_from()`, would cause the `with_loader_criteria()` feature as well as the IN criteria needed for single-table inheritance queries to not render, in cases where the columns clause of the query did not explicitly include the left-hand side entity of the JOIN. The correct entity is now transferred to the `Join` object that’s generated internally, so that the criteria against the left side entity is correctly added.

    References: [#8721](https://www.sqlalchemy.org/trac/ticket/8721)

*   **[orm] [bug]**

    An informative exception is now raised when the `with_loader_criteria()` option is used as a loader option added to a specific “loader path”, such as when using it within `Load.options()`. This use is not supported as `with_loader_criteria()` is only intended to be used as a top level loader option. Previously, an internal error would be generated.

    References: [#8711](https://www.sqlalchemy.org/trac/ticket/8711)

*   **[orm] [bug]**

    Improved “dictionary mode” for `Session.get()` so that synonym names which refer to primary key attribute names may be indicated in the named dictionary.

    References: [#8753](https://www.sqlalchemy.org/trac/ticket/8753)

*   **[orm] [bug]**

    Fixed issue where “selectin_polymorphic” loading for inheritance mappers would not function correctly if the `Mapper.polymorphic_on` parameter referred to a SQL expression that was not directly mapped on the class.

    References: [#8704](https://www.sqlalchemy.org/trac/ticket/8704)

*   **[orm] [bug]**

    Fixed issue where the underlying DBAPI cursor would not be closed when using the `Query` object as an iterator, if a user-defined exception case were raised within the iteration process, thereby causing the iterator to be closed by the Python interpreter. When using `Query.yield_per()` to create server-side cursors, this would lead to the usual MySQL-related issues with server side cursors out of sync, and without direct access to the `Result` object, end-user code could not access the cursor in order to close it.

    To resolve, a catch for `GeneratorExit` is applied within the iterator method, which will close the result object in those cases when the iterator were interrupted, and by definition will be closed by the Python interpreter.

    As part of this change as implemented for the 1.4 series, ensured that `.close()` methods are available on all `Result` implementations including `ScalarResult`, `MappingResult`. The 2.0 version of this change also includes new context manager patterns for use with `Result` classes.

    References: [#8710](https://www.sqlalchemy.org/trac/ticket/8710)

### engine

*   **[engine] [bug] [regression]**

    Fixed issue where the `PoolEvents.reset()` event hook would not be be called in all cases when a `Connection` were closed and was in the process of returning its DBAPI connection to the connection pool.

    The scenario was when the `Connection` had already emitted `.rollback()` on its DBAPI connection within the process of returning the connection to the pool, where it would then instruct the connection pool to forego doing its own “reset” to save on the additional method call. However, this prevented custom pool reset schemes from being used within this hook, as such hooks by definition are doing more than just calling `.rollback()`, and need to be invoked under all circumstances. This was a regression that appeared in version 1.4.

    For version 1.4, the `PoolEvents.checkin()` remains viable as an alternate event hook to use for custom “reset” implementations. Version 2.0 will feature an improved version of `PoolEvents.reset()` which is called for additional scenarios such as termination of asyncio connections, and is also passed contextual information about the reset, to allow for “custom connection reset” schemes which can respond to different reset scenarios in different ways.

    References: [#8717](https://www.sqlalchemy.org/trac/ticket/8717)

*   **[engine] [bug]**

    Ensured all `Result` objects include a `Result.close()` method as well as a `Result.closed` attribute, including on `ScalarResult` and `MappingResult`.

    References: [#8710](https://www.sqlalchemy.org/trac/ticket/8710)

### sql

*   **[sql] [bug]**

    Fixed issue which prevented the `literal_column()` construct from working properly within the context of a `Select` construct as well as other potential places where “anonymized labels” might be generated, if the literal expression contained characters which could interfere with format strings, such as open parenthesis, due to an implementation detail of the “anonymous label” structure.

    References: [#8724](https://www.sqlalchemy.org/trac/ticket/8724)

### mssql

*   **[mssql] [bug]**

    Fixed issue with `Inspector.has_table()`, which when used against a temporary table with the SQL Server dialect would fail on some Azure variants, due to an unnecessary information schema query that is not supported on those server versions. Pull request courtesy Mike Barry.

    References: [#8714](https://www.sqlalchemy.org/trac/ticket/8714)

*   **[mssql] [bug] [reflection]**

    Fixed issue with `Inspector.has_table()`, which when used against a view with the SQL Server dialect would erroneously return `False`, due to a regression in the 1.4 series which removed support for this on SQL Server. The issue is not present in the 2.0 series which uses a different reflection architecture. Test support is added to ensure `has_table()` remains working per spec re: views.

    References: [#8700](https://www.sqlalchemy.org/trac/ticket/8700)

### oracle

*   **[oracle] [bug]**

    Fixed issue where bound parameter names, including those automatically derived from similarly-named database columns, which contained characters that normally require quoting with Oracle would not be escaped when using “expanding parameters” with the Oracle dialect, causing execution errors. The usual “quoting” for bound parameters used by the Oracle dialect is not used with the “expanding parameters” architecture, so escaping for a large range of characters is used instead, now using a list of characters/escapes that are specific to Oracle.

    References: [#8708](https://www.sqlalchemy.org/trac/ticket/8708)

*   **[oracle] [bug]**

    Fixed issue where the `nls_session_parameters` view queried on first connect in order to get the default decimal point character may not be available depending on Oracle connection modes, and would therefore raise an error. The approach to detecting decimal char has been simplified to test a decimal value directly, instead of reading system views, which works on any backend / driver.

    References: [#8744](https://www.sqlalchemy.org/trac/ticket/8744)

### orm

*   **[orm] [bug]**

    Fixed issue in joined eager loading where an assertion fail would occur with a particular combination of outer/inner joined eager loads, when eager loading across three mappers where the middle mapper was an inherited subclass mapper.

    References: [#8738](https://www.sqlalchemy.org/trac/ticket/8738)

*   **[orm] [bug]**

    Fixed bug involving `Select` constructs, where combinations of `Select.select_from()` with `Select.join()`, as well as when using `Select.join_from()`, would cause the `with_loader_criteria()` feature as well as the IN criteria needed for single-table inheritance queries to not render, in cases where the columns clause of the query did not explicitly include the left-hand side entity of the JOIN. The correct entity is now transferred to the `Join` object that’s generated internally, so that the criteria against the left side entity is correctly added.

    References: [#8721](https://www.sqlalchemy.org/trac/ticket/8721)

*   **[orm] [bug]**

    An informative exception is now raised when the `with_loader_criteria()` option is used as a loader option added to a specific “loader path”, such as when using it within `Load.options()`. This use is not supported as `with_loader_criteria()` is only intended to be used as a top level loader option. Previously, an internal error would be generated.

    References: [#8711](https://www.sqlalchemy.org/trac/ticket/8711)

*   **[orm] [bug]**

    Improved “dictionary mode” for `Session.get()` so that synonym names which refer to primary key attribute names may be indicated in the named dictionary.

    References: [#8753](https://www.sqlalchemy.org/trac/ticket/8753)

*   **[orm] [bug]**

    Fixed issue where “selectin_polymorphic” loading for inheritance mappers would not function correctly if the `Mapper.polymorphic_on` parameter referred to a SQL expression that was not directly mapped on the class.

    References: [#8704](https://www.sqlalchemy.org/trac/ticket/8704)

*   **[orm] [bug]**

    Fixed issue where the underlying DBAPI cursor would not be closed when using the `Query` object as an iterator, if a user-defined exception case were raised within the iteration process, thereby causing the iterator to be closed by the Python interpreter. When using `Query.yield_per()` to create server-side cursors, this would lead to the usual MySQL-related issues with server side cursors out of sync, and without direct access to the `Result` object, end-user code could not access the cursor in order to close it.

    To resolve, a catch for `GeneratorExit` is applied within the iterator method, which will close the result object in those cases when the iterator were interrupted, and by definition will be closed by the Python interpreter.

    As part of this change as implemented for the 1.4 series, ensured that `.close()` methods are available on all `Result` implementations including `ScalarResult`, `MappingResult`. The 2.0 version of this change also includes new context manager patterns for use with `Result` classes.

    References: [#8710](https://www.sqlalchemy.org/trac/ticket/8710)

### engine

*   **[engine] [bug] [regression]**

    Fixed issue where the `PoolEvents.reset()` event hook would not be be called in all cases when a `Connection` were closed and was in the process of returning its DBAPI connection to the connection pool.

    The scenario was when the `Connection` had already emitted `.rollback()` on its DBAPI connection within the process of returning the connection to the pool, where it would then instruct the connection pool to forego doing its own “reset” to save on the additional method call. However, this prevented custom pool reset schemes from being used within this hook, as such hooks by definition are doing more than just calling `.rollback()`, and need to be invoked under all circumstances. This was a regression that appeared in version 1.4.

    For version 1.4, the `PoolEvents.checkin()` remains viable as an alternate event hook to use for custom “reset” implementations. Version 2.0 will feature an improved version of `PoolEvents.reset()` which is called for additional scenarios such as termination of asyncio connections, and is also passed contextual information about the reset, to allow for “custom connection reset” schemes which can respond to different reset scenarios in different ways.

    References: [#8717](https://www.sqlalchemy.org/trac/ticket/8717)

*   **[engine] [bug]**

    Ensured all `Result` objects include a `Result.close()` method as well as a `Result.closed` attribute, including on `ScalarResult` and `MappingResult`.

    References: [#8710](https://www.sqlalchemy.org/trac/ticket/8710)

### sql

*   **[sql] [bug]**

    Fixed issue which prevented the `literal_column()` construct from working properly within the context of a `Select` construct as well as other potential places where “anonymized labels” might be generated, if the literal expression contained characters which could interfere with format strings, such as open parenthesis, due to an implementation detail of the “anonymous label” structure.

    References: [#8724](https://www.sqlalchemy.org/trac/ticket/8724)

### mssql

*   **[mssql] [bug]**

    Fixed issue with `Inspector.has_table()`, which when used against a temporary table with the SQL Server dialect would fail on some Azure variants, due to an unnecessary information schema query that is not supported on those server versions. Pull request courtesy Mike Barry.

    References: [#8714](https://www.sqlalchemy.org/trac/ticket/8714)

*   **[mssql] [bug] [reflection]**

    Fixed issue with `Inspector.has_table()`, which when used against a view with the SQL Server dialect would erroneously return `False`, due to a regression in the 1.4 series which removed support for this on SQL Server. The issue is not present in the 2.0 series which uses a different reflection architecture. Test support is added to ensure `has_table()` remains working per spec re: views.

    References: [#8700](https://www.sqlalchemy.org/trac/ticket/8700)

### oracle

*   **[oracle] [bug]**

    Fixed issue where bound parameter names, including those automatically derived from similarly-named database columns, which contained characters that normally require quoting with Oracle would not be escaped when using “expanding parameters” with the Oracle dialect, causing execution errors. The usual “quoting” for bound parameters used by the Oracle dialect is not used with the “expanding parameters” architecture, so escaping for a large range of characters is used instead, now using a list of characters/escapes that are specific to Oracle.

    References: [#8708](https://www.sqlalchemy.org/trac/ticket/8708)

*   **[oracle] [bug]**

    Fixed issue where the `nls_session_parameters` view queried on first connect in order to get the default decimal point character may not be available depending on Oracle connection modes, and would therefore raise an error. The approach to detecting decimal char has been simplified to test a decimal value directly, instead of reading system views, which works on any backend / driver.

    References: [#8744](https://www.sqlalchemy.org/trac/ticket/8744)

## 1.4.42

Released: October 16, 2022

### orm

*   **[orm] [bug]**

    The `Session.execute.bind_arguments` dictionary is no longer mutated when passed to `Session.execute()` and similar; instead, it’s copied to an internal dictionary for state changes. Among other things, this fixes and issue where the “clause” passed to the `Session.get_bind()` method would be incorrectly referring to the `Select` construct used for the “fetch” synchronization strategy, when the actual query being emitted was a `Delete` or `Update`. This would interfere with recipes for “routing sessions”.

    References: [#8614](https://www.sqlalchemy.org/trac/ticket/8614)

*   **[orm] [bug]**

    A warning is emitted in ORM configurations when an explicit `remote()` annotation is applied to columns that are local to the immediate mapped class, when the referenced class does not include any of the same table columns. Ideally this would raise an error at some point as it’s not correct from a mapping point of view.

    References: [#7094](https://www.sqlalchemy.org/trac/ticket/7094)

*   **[orm] [bug]**

    A warning is emitted when attempting to configure a mapped class within an inheritance hierarchy where the mapper is not given any polymorphic identity, however there is a polymorphic discriminator column assigned. Such classes should be abstract if they never intend to load directly.

    References: [#7545](https://www.sqlalchemy.org/trac/ticket/7545)

*   **[orm] [bug] [regression]**

    Fixed regression for 1.4 in `contains_eager()` where the “wrap in subquery” logic of `joinedload()` would be inadvertently triggered for use of the `contains_eager()` function with similar statements (e.g. those that use `distinct()`, `limit()` or `offset()`), which would then lead to secondary issues with queries that used some combinations of SQL label names and aliasing. This “wrapping” is not appropriate for `contains_eager()` which has always had the contract that the user-defined SQL statement is unmodified with the exception of adding the appropriate columns to be fetched.

    References: [#8569](https://www.sqlalchemy.org/trac/ticket/8569)

*   **[orm] [bug] [regression]**

    Fixed regression where using ORM update() with synchronize_session=’fetch’ would fail due to the use of evaluators that are now used to determine the in-Python value for expressions in the SET clause when refreshing objects; if the evaluators make use of math operators against non-numeric values such as PostgreSQL JSONB, the non-evaluable condition would fail to be detected correctly. The evaluator now limits the use of math mutation operators to numeric types only, with the exception of “+” that continues to work for strings as well. SQLAlchemy 2.0 may alter this further by fetching the SET values completely rather than using evaluation.

    References: [#8507](https://www.sqlalchemy.org/trac/ticket/8507)

### engine

*   **[engine] [bug]**

    Fixed issue where mixing “*” with additional explicitly-named column expressions within the columns clause of a `select()` construct would cause result-column targeting to sometimes consider the label name or other non-repeated names to be an ambiguous target.

    References: [#8536](https://www.sqlalchemy.org/trac/ticket/8536)

### asyncio

*   **[asyncio] [bug]**

    Improved implementation of `asyncio.shield()` used in context managers as added in [#8145](https://www.sqlalchemy.org/trac/ticket/8145), such that the “close” operation is enclosed within an `asyncio.Task` which is then strongly referenced as the operation proceeds. This is per Python documentation indicating that the task is otherwise not strongly referenced.

    References: [#8516](https://www.sqlalchemy.org/trac/ticket/8516)

### postgresql

*   **[postgresql] [usecase]**

    `aggregate_order_by` now supports cache generation.

    References: [#8574](https://www.sqlalchemy.org/trac/ticket/8574)

### mysql

*   **[mysql] [bug]**

    Adjusted the regular expression used to match “CREATE VIEW” when testing for views to work more flexibly, no longer requiring the special keyword “ALGORITHM” in the middle, which was intended to be optional but was not working correctly. The change allows view reflection to work more completely on MySQL-compatible variants such as StarRocks. Pull request courtesy John Bodley.

    References: [#8588](https://www.sqlalchemy.org/trac/ticket/8588)

### mssql

*   **[mssql] [bug] [regression]**

    Fixed yet another regression in SQL Server isolation level fetch (see [#8231](https://www.sqlalchemy.org/trac/ticket/8231), [#8475](https://www.sqlalchemy.org/trac/ticket/8475)), this time with “Microsoft Dynamics CRM Database via Azure Active Directory”, which apparently lacks the `system_views` view entirely. Error catching has been extended that under no circumstances will this method ever fail, provided database connectivity is present.

    References: [#8525](https://www.sqlalchemy.org/trac/ticket/8525)

### orm

*   **[orm] [bug]**

    The `Session.execute.bind_arguments` dictionary is no longer mutated when passed to `Session.execute()` and similar; instead, it’s copied to an internal dictionary for state changes. Among other things, this fixes and issue where the “clause” passed to the `Session.get_bind()` method would be incorrectly referring to the `Select` construct used for the “fetch” synchronization strategy, when the actual query being emitted was a `Delete` or `Update`. This would interfere with recipes for “routing sessions”.

    References: [#8614](https://www.sqlalchemy.org/trac/ticket/8614)

*   **[orm] [bug]**

    A warning is emitted in ORM configurations when an explicit `remote()` annotation is applied to columns that are local to the immediate mapped class, when the referenced class does not include any of the same table columns. Ideally this would raise an error at some point as it’s not correct from a mapping point of view.

    References: [#7094](https://www.sqlalchemy.org/trac/ticket/7094)

*   **[orm] [bug]**

    A warning is emitted when attempting to configure a mapped class within an inheritance hierarchy where the mapper is not given any polymorphic identity, however there is a polymorphic discriminator column assigned. Such classes should be abstract if they never intend to load directly.

    References: [#7545](https://www.sqlalchemy.org/trac/ticket/7545)

*   **[orm] [bug] [regression]**

    Fixed regression for 1.4 in `contains_eager()` where the “wrap in subquery” logic of `joinedload()` would be inadvertently triggered for use of the `contains_eager()` function with similar statements (e.g. those that use `distinct()`, `limit()` or `offset()`), which would then lead to secondary issues with queries that used some combinations of SQL label names and aliasing. This “wrapping” is not appropriate for `contains_eager()` which has always had the contract that the user-defined SQL statement is unmodified with the exception of adding the appropriate columns to be fetched.

    References: [#8569](https://www.sqlalchemy.org/trac/ticket/8569)

*   **[orm] [bug] [regression]**

    Fixed regression where using ORM update() with synchronize_session=’fetch’ would fail due to the use of evaluators that are now used to determine the in-Python value for expressions in the SET clause when refreshing objects; if the evaluators make use of math operators against non-numeric values such as PostgreSQL JSONB, the non-evaluable condition would fail to be detected correctly. The evaluator now limits the use of math mutation operators to numeric types only, with the exception of “+” that continues to work for strings as well. SQLAlchemy 2.0 may alter this further by fetching the SET values completely rather than using evaluation.

    References: [#8507](https://www.sqlalchemy.org/trac/ticket/8507)

### engine

*   **[engine] [bug]**

    Fixed issue where mixing “*” with additional explicitly-named column expressions within the columns clause of a `select()` construct would cause result-column targeting to sometimes consider the label name or other non-repeated names to be an ambiguous target.

    References: [#8536](https://www.sqlalchemy.org/trac/ticket/8536)

### asyncio

*   **[asyncio] [bug]**

    Improved implementation of `asyncio.shield()` used in context managers as added in [#8145](https://www.sqlalchemy.org/trac/ticket/8145), such that the “close” operation is enclosed within an `asyncio.Task` which is then strongly referenced as the operation proceeds. This is per Python documentation indicating that the task is otherwise not strongly referenced.

    References: [#8516](https://www.sqlalchemy.org/trac/ticket/8516)

### postgresql

*   **[postgresql] [usecase]**

    `aggregate_order_by` now supports cache generation.

    References: [#8574](https://www.sqlalchemy.org/trac/ticket/8574)

### mysql

*   **[mysql] [bug]**

    Adjusted the regular expression used to match “CREATE VIEW” when testing for views to work more flexibly, no longer requiring the special keyword “ALGORITHM” in the middle, which was intended to be optional but was not working correctly. The change allows view reflection to work more completely on MySQL-compatible variants such as StarRocks. Pull request courtesy John Bodley.

    References: [#8588](https://www.sqlalchemy.org/trac/ticket/8588)

### mssql

*   **[mssql] [bug] [regression]**

    Fixed yet another regression in SQL Server isolation level fetch (see [#8231](https://www.sqlalchemy.org/trac/ticket/8231), [#8475](https://www.sqlalchemy.org/trac/ticket/8475)), this time with “Microsoft Dynamics CRM Database via Azure Active Directory”, which apparently lacks the `system_views` view entirely. Error catching has been extended that under no circumstances will this method ever fail, provided database connectivity is present.

    References: [#8525](https://www.sqlalchemy.org/trac/ticket/8525)

## 1.4.41

Released: September 6, 2022

### orm

*   **[orm] [bug] [events]**

    Fixed event listening issue where event listeners added to a superclass would be lost if a subclass were created which then had its own listeners associated. The practical example is that of the `sessionmaker` class created after events have been associated with the `Session` class.

    References: [#8467](https://www.sqlalchemy.org/trac/ticket/8467)

*   **[orm] [bug]**

    Hardened the cache key strategy for the `aliased()` and `with_polymorphic()` constructs. While no issue involving actual statements being cached can easily be demonstrated (if at all), these two constructs were not including enough of what makes them unique in their cache keys for caching on the aliased construct alone to be accurate.

    References: [#8401](https://www.sqlalchemy.org/trac/ticket/8401)

*   **[orm] [bug] [regression]**

    Fixed regression appearing in the 1.4 series where a joined-inheritance query placed as a subquery within an enclosing query for that same entity would fail to render the JOIN correctly for the inner query. The issue manifested in two different ways prior and subsequent to version 1.4.18 (related issue [#6595](https://www.sqlalchemy.org/trac/ticket/6595)), in one case rendering JOIN twice, in the other losing the JOIN entirely. To resolve, the conditions under which “polymorphic loading” are applied have been scaled back to not be invoked for simple joined inheritance queries.

    References: [#8456](https://www.sqlalchemy.org/trac/ticket/8456)

*   **[orm] [bug]**

    Fixed issue in `sqlalchemy.ext.mutable` extension where collection links to the parent object would be lost if the object were merged with `Session.merge()` while also passing `Session.merge.load` as False.

    References: [#8446](https://www.sqlalchemy.org/trac/ticket/8446)

*   **[orm] [bug]**

    Fixed issue involving `with_loader_criteria()` where a closure variable used as bound parameter value within the lambda would not carry forward correctly into additional relationship loaders such as `selectinload()` and `lazyload()` after the statement were cached, using the stale originally-cached value instead.

    References: [#8399](https://www.sqlalchemy.org/trac/ticket/8399)

### sql

*   **[sql] [bug]**

    Fixed issue where use of the `table()` construct, passing a string for the `table.schema` parameter, would fail to take the “schema” string into account when producing a cache key, thus leading to caching collisions if multiple, same-named `table()` constructs with different schemas were used.

    References: [#8441](https://www.sqlalchemy.org/trac/ticket/8441)

### asyncio

*   **[asyncio] [bug]**

    Integrated support for asyncpg’s `terminate()` method call for cases where the connection pool is recycling a possibly timed-out connection, where a connection is being garbage collected that wasn’t gracefully closed, as well as when the connection has been invalidated. This allows asyncpg to abandon the connection without waiting for a response that may incur long timeouts.

    References: [#8419](https://www.sqlalchemy.org/trac/ticket/8419)

### mssql

*   **[mssql] [bug] [regression]**

    Fixed regression caused by the fix for [#8231](https://www.sqlalchemy.org/trac/ticket/8231) released in 1.4.40 where connection would fail if the user did not have permission to query the `dm_exec_sessions` or `dm_pdw_nodes_exec_sessions` system views when trying to determine the current transaction isolation level.

    References: [#8475](https://www.sqlalchemy.org/trac/ticket/8475)

### orm

*   **[orm] [bug] [events]**

    Fixed event listening issue where event listeners added to a superclass would be lost if a subclass were created which then had its own listeners associated. The practical example is that of the `sessionmaker` class created after events have been associated with the `Session` class.

    References: [#8467](https://www.sqlalchemy.org/trac/ticket/8467)

*   **[orm] [bug]**

    Hardened the cache key strategy for the `aliased()` and `with_polymorphic()` constructs. While no issue involving actual statements being cached can easily be demonstrated (if at all), these two constructs were not including enough of what makes them unique in their cache keys for caching on the aliased construct alone to be accurate.

    References: [#8401](https://www.sqlalchemy.org/trac/ticket/8401)

*   **[orm] [bug] [regression]**

    Fixed regression appearing in the 1.4 series where a joined-inheritance query placed as a subquery within an enclosing query for that same entity would fail to render the JOIN correctly for the inner query. The issue manifested in two different ways prior and subsequent to version 1.4.18 (related issue [#6595](https://www.sqlalchemy.org/trac/ticket/6595)), in one case rendering JOIN twice, in the other losing the JOIN entirely. To resolve, the conditions under which “polymorphic loading” are applied have been scaled back to not be invoked for simple joined inheritance queries.

    References: [#8456](https://www.sqlalchemy.org/trac/ticket/8456)

*   **[orm] [bug]**

    Fixed issue in `sqlalchemy.ext.mutable` extension where collection links to the parent object would be lost if the object were merged with `Session.merge()` while also passing `Session.merge.load` as False.

    References: [#8446](https://www.sqlalchemy.org/trac/ticket/8446)

*   **[orm] [bug]**

    Fixed issue involving `with_loader_criteria()` where a closure variable used as bound parameter value within the lambda would not carry forward correctly into additional relationship loaders such as `selectinload()` and `lazyload()` after the statement were cached, using the stale originally-cached value instead.

    References: [#8399](https://www.sqlalchemy.org/trac/ticket/8399)

### sql

*   **[sql] [bug]**

    Fixed issue where use of the `table()` construct, passing a string for the `table.schema` parameter, would fail to take the “schema” string into account when producing a cache key, thus leading to caching collisions if multiple, same-named `table()` constructs with different schemas were used.

    References: [#8441](https://www.sqlalchemy.org/trac/ticket/8441)

### asyncio

*   **[asyncio] [bug]**

    Integrated support for asyncpg’s `terminate()` method call for cases where the connection pool is recycling a possibly timed-out connection, where a connection is being garbage collected that wasn’t gracefully closed, as well as when the connection has been invalidated. This allows asyncpg to abandon the connection without waiting for a response that may incur long timeouts.

    References: [#8419](https://www.sqlalchemy.org/trac/ticket/8419)

### mssql

*   **[mssql] [bug] [regression]**

    Fixed regression caused by the fix for [#8231](https://www.sqlalchemy.org/trac/ticket/8231) released in 1.4.40 where connection would fail if the user did not have permission to query the `dm_exec_sessions` or `dm_pdw_nodes_exec_sessions` system views when trying to determine the current transaction isolation level.

    References: [#8475](https://www.sqlalchemy.org/trac/ticket/8475)

## 1.4.40

Released: August 8, 2022

### orm

*   **[orm] [bug]**

    Fixed issue where referencing a CTE multiple times in conjunction with a polymorphic SELECT could result in multiple “clones” of the same CTE being constructed, which would then trigger these two CTEs as duplicates. To resolve, the two CTEs are deep-compared when this occurs to ensure that they are equivalent, then are treated as equivalent.

    References: [#8357](https://www.sqlalchemy.org/trac/ticket/8357)

*   **[orm] [bug]**

    A `select()` construct that is passed a sole ‘*’ argument for `SELECT *`, either via string, `text()`, or `literal_column()`, will be interpreted as a Core-level SQL statement rather than as an ORM level statement. This is so that the `*`, when expanded to match any number of columns, will result in all columns returned in the result. the ORM- level interpretation of `select()` needs to know the names and types of all ORM columns up front which can’t be achieved when `'*'` is used.

    If `'*` is used amongst other expressions simultaneously with an ORM statement, an error is raised as this can’t be interpreted correctly by the ORM.

    References: [#8235](https://www.sqlalchemy.org/trac/ticket/8235)

### orm declarative

*   **[orm] [declarative] [bug]**

    Fixed issue where a hierarchy of classes set up as an abstract or mixin declarative classes could not declare standalone columns on a superclass that would then be copied correctly to a `declared_attr` callable that wanted to make use of them on a descendant class.

    References: [#8190](https://www.sqlalchemy.org/trac/ticket/8190)

### engine

*   **[engine] [usecase]**

    Implemented new `Connection.execution_options.yield_per` execution option for `Connection` in Core, to mirror that of the same yield_per option available in the ORM. The option sets both the `Connection.execution_options.stream_results` option at the same time as invoking `Result.yield_per()`, to provide the most common streaming result configuration which also mirrors that of the ORM use case in its usage pattern.

    See also

    Using Server Side Cursors (a.k.a. stream results) - revised documentation

*   **[engine] [bug]**

    Fixed bug in `Result` where the usage of a buffered result strategy would not be used if the dialect in use did not support an explicit “server side cursor” setting, when using `Connection.execution_options.stream_results`. This is in error as DBAPIs such as that of SQLite and Oracle already use a non-buffered result fetching scheme, which still benefits from usage of partial result fetching. The “buffered” strategy is now used in all cases where `Connection.execution_options.stream_results` is set.

*   **[engine] [bug]**

    Added `FilterResult.yield_per()` so that result implementations such as `MappingResult`, `ScalarResult` and `AsyncResult` have access to this method.

    References: [#8199](https://www.sqlalchemy.org/trac/ticket/8199)

### sql

*   **[sql] [bug]**

    Adjusted the SQL compilation for string containment functions `.contains()`, `.startswith()`, `.endswith()` to force the use of the string concatenation operator, rather than relying upon the overload of the addition operator, so that non-standard use of these operators with for example bytestrings still produces string concatenation operators.

    References: [#8253](https://www.sqlalchemy.org/trac/ticket/8253)

### mypy

*   **[mypy] [bug]**

    Fixed a crash of the mypy plugin when using a lambda as a Column default. Pull request courtesy of tchapi.

    References: [#8196](https://www.sqlalchemy.org/trac/ticket/8196)

### asyncio

*   **[asyncio] [bug]**

    Added `asyncio.shield()` to the connection and session release process specifically within the `__aexit__()` context manager exit, when using `AsyncConnection` or `AsyncSession` as a context manager that releases the object when the context manager is complete. This appears to help with task cancellation when using alternate concurrency libraries such as `anyio`, `uvloop` that otherwise don’t provide an async context for the connection pool to release the connection properly during task cancellation.

    References: [#8145](https://www.sqlalchemy.org/trac/ticket/8145)

### postgresql

*   **[postgresql] [bug]**

    Fixed issue in psycopg2 dialect where the “multiple hosts” feature implemented for [#4392](https://www.sqlalchemy.org/trac/ticket/4392), where multiple `host:port` pairs could be passed in the query string as `?host=host1:port1&host=host2:port2&host=host3:port3` was not implemented correctly, as it did not propagate the “port” parameter appropriately. Connections that didn’t use a different “port” likely worked without issue, and connections that had “port” for some of the entries may have incorrectly passed on that hostname. The format is now corrected to pass hosts/ports appropriately.

    As part of this change, maintained support for another multihost style that worked unintentionally, which is comma-separated `?host=h1,h2,h3&port=p1,p2,p3`. This format is more consistent with libpq’s query-string format, whereas the previous format is inspired by a different aspect of libpq’s URI format but is not quite the same thing.

    If the two styles are mixed together, an error is raised as this is ambiguous.

    References: [#4392](https://www.sqlalchemy.org/trac/ticket/4392)

### mssql

*   **[mssql] [bug]**

    Fixed issues that prevented the new usage patterns for using DML with ORM objects presented at Using INSERT, UPDATE and ON CONFLICT (i.e. upsert) to return ORM Objects from working correctly with the SQL Server pyodbc dialect.

    References: [#8210](https://www.sqlalchemy.org/trac/ticket/8210)

*   **[mssql] [bug]**

    Fixed issue where the SQL Server dialect’s query for the current isolation level would fail on Azure Synapse Analytics, due to the way in which this database handles transaction rollbacks after an error has occurred. The initial query has been modified to no longer rely upon catching an error when attempting to detect the appropriate system view. Additionally, to better support this database’s very specific “rollback” behavior, implemented new parameter `ignore_no_transaction_on_rollback` indicating that a rollback should ignore Azure Synapse error ‘No corresponding transaction found. (111214)’, which is raised if no transaction is present in conflict with the Python DBAPI.

    Initial patch and valuable debugging assistance courtesy of @ww2406.

    See also

    Avoiding transaction-related exceptions on Azure Synapse Analytics

    References: [#8231](https://www.sqlalchemy.org/trac/ticket/8231)

### misc

*   **[bug] [types]**

    Fixed issue where `TypeDecorator` would not correctly proxy the `__getitem__()` operator when decorating the `ARRAY` datatype, without explicit workarounds.

    References: [#7249](https://www.sqlalchemy.org/trac/ticket/7249)

### orm

*   **[orm] [bug]**

    Fixed issue where referencing a CTE multiple times in conjunction with a polymorphic SELECT could result in multiple “clones” of the same CTE being constructed, which would then trigger these two CTEs as duplicates. To resolve, the two CTEs are deep-compared when this occurs to ensure that they are equivalent, then are treated as equivalent.

    References: [#8357](https://www.sqlalchemy.org/trac/ticket/8357)

*   **[orm] [bug]**

    A `select()` construct that is passed a sole ‘*’ argument for `SELECT *`, either via string, `text()`, or `literal_column()`, will be interpreted as a Core-level SQL statement rather than as an ORM level statement. This is so that the `*`, when expanded to match any number of columns, will result in all columns returned in the result. the ORM- level interpretation of `select()` needs to know the names and types of all ORM columns up front which can’t be achieved when `'*'` is used.

    If `'*` is used amongst other expressions simultaneously with an ORM statement, an error is raised as this can’t be interpreted correctly by the ORM.

    References: [#8235](https://www.sqlalchemy.org/trac/ticket/8235)

### orm declarative

*   **[orm] [declarative] [bug]**

    Fixed issue where a hierarchy of classes set up as an abstract or mixin declarative classes could not declare standalone columns on a superclass that would then be copied correctly to a `declared_attr` callable that wanted to make use of them on a descendant class.

    References: [#8190](https://www.sqlalchemy.org/trac/ticket/8190)

### engine

*   **[engine] [usecase]**

    Implemented new `Connection.execution_options.yield_per` execution option for `Connection` in Core, to mirror that of the same yield_per option available in the ORM. The option sets both the `Connection.execution_options.stream_results` option at the same time as invoking `Result.yield_per()`, to provide the most common streaming result configuration which also mirrors that of the ORM use case in its usage pattern.

    See also

    Using Server Side Cursors (a.k.a. stream results) - revised documentation

*   **[engine] [bug]**

    Fixed bug in `Result` where the usage of a buffered result strategy would not be used if the dialect in use did not support an explicit “server side cursor” setting, when using `Connection.execution_options.stream_results`. This is in error as DBAPIs such as that of SQLite and Oracle already use a non-buffered result fetching scheme, which still benefits from usage of partial result fetching. The “buffered” strategy is now used in all cases where `Connection.execution_options.stream_results` is set.

*   **[engine] [bug]**

    Added `FilterResult.yield_per()` so that result implementations such as `MappingResult`, `ScalarResult` and `AsyncResult` have access to this method.

    References: [#8199](https://www.sqlalchemy.org/trac/ticket/8199)

### sql

*   **[sql] [bug]**

    Adjusted the SQL compilation for string containment functions `.contains()`, `.startswith()`, `.endswith()` to force the use of the string concatenation operator, rather than relying upon the overload of the addition operator, so that non-standard use of these operators with for example bytestrings still produces string concatenation operators.

    References: [#8253](https://www.sqlalchemy.org/trac/ticket/8253)

### mypy

*   **[mypy] [bug]**

    Fixed a crash of the mypy plugin when using a lambda as a Column default. Pull request courtesy of tchapi.

    References: [#8196](https://www.sqlalchemy.org/trac/ticket/8196)

### asyncio

*   **[asyncio] [bug]**

    Added `asyncio.shield()` to the connection and session release process specifically within the `__aexit__()` context manager exit, when using `AsyncConnection` or `AsyncSession` as a context manager that releases the object when the context manager is complete. This appears to help with task cancellation when using alternate concurrency libraries such as `anyio`, `uvloop` that otherwise don’t provide an async context for the connection pool to release the connection properly during task cancellation.

    References: [#8145](https://www.sqlalchemy.org/trac/ticket/8145)

### postgresql

*   **[postgresql] [bug]**

    Fixed issue in psycopg2 dialect where the “multiple hosts” feature implemented for [#4392](https://www.sqlalchemy.org/trac/ticket/4392), where multiple `host:port` pairs could be passed in the query string as `?host=host1:port1&host=host2:port2&host=host3:port3` was not implemented correctly, as it did not propagate the “port” parameter appropriately. Connections that didn’t use a different “port” likely worked without issue, and connections that had “port” for some of the entries may have incorrectly passed on that hostname. The format is now corrected to pass hosts/ports appropriately.

    As part of this change, maintained support for another multihost style that worked unintentionally, which is comma-separated `?host=h1,h2,h3&port=p1,p2,p3`. This format is more consistent with libpq’s query-string format, whereas the previous format is inspired by a different aspect of libpq’s URI format but is not quite the same thing.

    If the two styles are mixed together, an error is raised as this is ambiguous.

    References: [#4392](https://www.sqlalchemy.org/trac/ticket/4392)

### mssql

*   **[mssql] [bug]**

    Fixed issues that prevented the new usage patterns for using DML with ORM objects presented at Using INSERT, UPDATE and ON CONFLICT (i.e. upsert) to return ORM Objects from working correctly with the SQL Server pyodbc dialect.

    References: [#8210](https://www.sqlalchemy.org/trac/ticket/8210)

*   **[mssql] [bug]**

    Fixed issue where the SQL Server dialect’s query for the current isolation level would fail on Azure Synapse Analytics, due to the way in which this database handles transaction rollbacks after an error has occurred. The initial query has been modified to no longer rely upon catching an error when attempting to detect the appropriate system view. Additionally, to better support this database’s very specific “rollback” behavior, implemented new parameter `ignore_no_transaction_on_rollback` indicating that a rollback should ignore Azure Synapse error ‘No corresponding transaction found. (111214)’, which is raised if no transaction is present in conflict with the Python DBAPI.

    Initial patch and valuable debugging assistance courtesy of @ww2406.

    See also

    Avoiding transaction-related exceptions on Azure Synapse Analytics

    References: [#8231](https://www.sqlalchemy.org/trac/ticket/8231)

### misc

*   **[bug] [types]**

    Fixed issue where `TypeDecorator` would not correctly proxy the `__getitem__()` operator when decorating the `ARRAY` datatype, without explicit workarounds.

    References: [#7249](https://www.sqlalchemy.org/trac/ticket/7249)

## 1.4.39

Released: June 24, 2022

### orm

*   **[orm] [bug] [regression]**

    Fixed regression caused by [#8133](https://www.sqlalchemy.org/trac/ticket/8133) where the pickle format for mutable attributes was changed, without a fallback to recognize the old format, causing in-place upgrades of SQLAlchemy to no longer be able to read pickled data from previous versions. A check plus a fallback for the old format is now in place.

    References: [#8133](https://www.sqlalchemy.org/trac/ticket/8133)

### orm

*   **[orm] [bug] [regression]**

    Fixed regression caused by [#8133](https://www.sqlalchemy.org/trac/ticket/8133) where the pickle format for mutable attributes was changed, without a fallback to recognize the old format, causing in-place upgrades of SQLAlchemy to no longer be able to read pickled data from previous versions. A check plus a fallback for the old format is now in place.

    References: [#8133](https://www.sqlalchemy.org/trac/ticket/8133)

## 1.4.38

Released: June 23, 2022

### orm

*   **[orm] [bug] [regression]**

    Fixed regression caused by [#8064](https://www.sqlalchemy.org/trac/ticket/8064) where a particular check for column correspondence was made too liberal, resulting in incorrect rendering for some ORM subqueries such as those using `PropComparator.has()` or `PropComparator.any()` in conjunction with joined-inheritance queries that also use legacy aliasing features.

    References: [#8162](https://www.sqlalchemy.org/trac/ticket/8162)

*   **[orm] [bug] [sql]**

    Fixed an issue where `GenerativeSelect.fetch()` would not be applied when executing a statement using the ORM.

    References: [#8091](https://www.sqlalchemy.org/trac/ticket/8091)

*   **[orm] [bug]**

    Fixed issue where a `with_loader_criteria()` option could not be pickled, as is necessary when it is carried along for propagation to lazy loaders in conjunction with a caching scheme. Currently, the only form that is supported as picklable is to pass the “where criteria” as a fixed module-level callable function that produces a SQL expression. An ad-hoc “lambda” can’t be pickled, and a SQL expression object is usually not fully picklable directly.

    References: [#8109](https://www.sqlalchemy.org/trac/ticket/8109)

### engine

*   **[engine] [bug]**

    Repaired a deprecation warning class decorator that was preventing key objects such as `Connection` from having a proper `__weakref__` attribute, causing operations like Python standard library `inspect.getmembers()` to fail.

    References: [#8115](https://www.sqlalchemy.org/trac/ticket/8115)

### sql

*   **[sql] [bug]**

    Fixed multiple observed race conditions related to `lambda_stmt()`, including an initial “dogpile” issue when a new Python code object is initially analyzed among multiple simultaneous threads which created both a performance issue as well as some internal corruption of state. Additionally repaired observed race condition which could occur when “cloning” an expression construct that is also in the process of being compiled or otherwise accessed in a different thread due to memoized attributes altering the `__dict__` while iterated, for Python versions prior to 3.10; in particular the lambda SQL construct is sensitive to this as it holds onto a single statement object persistently. The iteration has been refined to use `dict.copy()` with or without an additional iteration instead.

    References: [#8098](https://www.sqlalchemy.org/trac/ticket/8098)

*   **[sql] [bug]**

    Enhanced the mechanism of `Cast` and other “wrapping” column constructs to more fully preserve a wrapped `Label` construct, including that the label name will be preserved in the `.c` collection of a `Subquery`. The label was already able to render in the SQL correctly on the outside of the construct which it was wrapped inside.

    References: [#8084](https://www.sqlalchemy.org/trac/ticket/8084)

*   **[sql] [bug]**

    Adjusted the fix made for [#8056](https://www.sqlalchemy.org/trac/ticket/8056) which adjusted the escaping of bound parameter names with special characters such that the escaped names were translated after the SQL compilation step, which broke a published recipe on the FAQ illustrating how to merge parameter names into the string output of a compiled SQL string. The change restores the escaped names that come from `compiled.params` and adds a conditional parameter to `SQLCompiler.construct_params()` named `escape_names` that defaults to `True`, restoring the old behavior by default.

    References: [#8113](https://www.sqlalchemy.org/trac/ticket/8113)

### schema

*   **[schema] [bug]**

    Fixed bugs involving the `Table.include_columns` and the `Table.resolve_fks` parameters on `Table`; these little-used parameters were apparently not working for columns that refer to foreign key constraints.

    In the first case, not-included columns that refer to foreign keys would still attempt to create a `ForeignKey` object, producing errors when attempting to resolve the columns for the foreign key constraint within reflection; foreign key constraints that refer to skipped columns are now omitted from the table reflection process in the same way as occurs for `Index` and `UniqueConstraint` objects with the same conditions. No warning is produced however, as we likely want to remove the include_columns warnings for all constraints in 2.0.

    In the latter case, the production of table aliases or subqueries would fail on an FK related table not found despite the presence of `resolve_fks=False`; the logic has been repaired so that if a related table is not found, the `ForeignKey` object is still proxied to the aliased table or subquery (these `ForeignKey` objects are normally used in the production of join conditions), but it is sent with a flag that it’s not resolvable. The aliased table / subquery will then work normally, with the exception that it cannot be used to generate a join condition automatically, as the foreign key information is missing. This was already the behavior for such foreign key constraints produced using non-reflection methods, such as joining `Table` objects from different `MetaData` collections.

    References: [#8100](https://www.sqlalchemy.org/trac/ticket/8100), [#8101](https://www.sqlalchemy.org/trac/ticket/8101)

*   **[schema] [bug] [mssql]**

    Fixed issue where `Table` objects that made use of IDENTITY columns with a `Numeric` datatype would produce errors when attempting to reconcile the “autoincrement” column, preventing construction of the `Column` from using the `Column.autoincrement` parameter as well as emitting errors when attempting to invoke an `Insert` construct.

    References: [#8111](https://www.sqlalchemy.org/trac/ticket/8111)

### extensions

*   **[extensions] [bug]**

    Fixed bug in `Mutable` where pickling and unpickling of an ORM mapped instance would not correctly restore state for mappings that contained multiple `Mutable`-enabled attributes.

    References: [#8133](https://www.sqlalchemy.org/trac/ticket/8133)

### orm

*   **[orm] [bug] [regression]**

    Fixed regression caused by [#8064](https://www.sqlalchemy.org/trac/ticket/8064) where a particular check for column correspondence was made too liberal, resulting in incorrect rendering for some ORM subqueries such as those using `PropComparator.has()` or `PropComparator.any()` in conjunction with joined-inheritance queries that also use legacy aliasing features.

    References: [#8162](https://www.sqlalchemy.org/trac/ticket/8162)

*   **[orm] [bug] [sql]**

    Fixed an issue where `GenerativeSelect.fetch()` would not be applied when executing a statement using the ORM.

    References: [#8091](https://www.sqlalchemy.org/trac/ticket/8091)

*   **[orm] [bug]**

    Fixed issue where a `with_loader_criteria()` option could not be pickled, as is necessary when it is carried along for propagation to lazy loaders in conjunction with a caching scheme. Currently, the only form that is supported as picklable is to pass the “where criteria” as a fixed module-level callable function that produces a SQL expression. An ad-hoc “lambda” can’t be pickled, and a SQL expression object is usually not fully picklable directly.

    References: [#8109](https://www.sqlalchemy.org/trac/ticket/8109)

### engine

*   **[engine] [bug]**

    Repaired a deprecation warning class decorator that was preventing key objects such as `Connection` from having a proper `__weakref__` attribute, causing operations like Python standard library `inspect.getmembers()` to fail.

    References: [#8115](https://www.sqlalchemy.org/trac/ticket/8115)

### sql

*   **[sql] [bug]**

    Fixed multiple observed race conditions related to `lambda_stmt()`, including an initial “dogpile” issue when a new Python code object is initially analyzed among multiple simultaneous threads which created both a performance issue as well as some internal corruption of state. Additionally repaired observed race condition which could occur when “cloning” an expression construct that is also in the process of being compiled or otherwise accessed in a different thread due to memoized attributes altering the `__dict__` while iterated, for Python versions prior to 3.10; in particular the lambda SQL construct is sensitive to this as it holds onto a single statement object persistently. The iteration has been refined to use `dict.copy()` with or without an additional iteration instead.

    References: [#8098](https://www.sqlalchemy.org/trac/ticket/8098)

*   **[sql] [bug]**

    Enhanced the mechanism of `Cast` and other “wrapping” column constructs to more fully preserve a wrapped `Label` construct, including that the label name will be preserved in the `.c` collection of a `Subquery`. The label was already able to render in the SQL correctly on the outside of the construct which it was wrapped inside.

    References: [#8084](https://www.sqlalchemy.org/trac/ticket/8084)

*   **[sql] [bug]**

    Adjusted the fix made for [#8056](https://www.sqlalchemy.org/trac/ticket/8056) which adjusted the escaping of bound parameter names with special characters such that the escaped names were translated after the SQL compilation step, which broke a published recipe on the FAQ illustrating how to merge parameter names into the string output of a compiled SQL string. The change restores the escaped names that come from `compiled.params` and adds a conditional parameter to `SQLCompiler.construct_params()` named `escape_names` that defaults to `True`, restoring the old behavior by default.

    References: [#8113](https://www.sqlalchemy.org/trac/ticket/8113)

### schema

*   **[schema] [bug]**

    Fixed bugs involving the `Table.include_columns` and the `Table.resolve_fks` parameters on `Table`; these little-used parameters were apparently not working for columns that refer to foreign key constraints.

    In the first case, not-included columns that refer to foreign keys would still attempt to create a `ForeignKey` object, producing errors when attempting to resolve the columns for the foreign key constraint within reflection; foreign key constraints that refer to skipped columns are now omitted from the table reflection process in the same way as occurs for `Index` and `UniqueConstraint` objects with the same conditions. No warning is produced however, as we likely want to remove the include_columns warnings for all constraints in 2.0.

    In the latter case, the production of table aliases or subqueries would fail on an FK related table not found despite the presence of `resolve_fks=False`; the logic has been repaired so that if a related table is not found, the `ForeignKey` object is still proxied to the aliased table or subquery (these `ForeignKey` objects are normally used in the production of join conditions), but it is sent with a flag that it’s not resolvable. The aliased table / subquery will then work normally, with the exception that it cannot be used to generate a join condition automatically, as the foreign key information is missing. This was already the behavior for such foreign key constraints produced using non-reflection methods, such as joining `Table` objects from different `MetaData` collections.

    References: [#8100](https://www.sqlalchemy.org/trac/ticket/8100), [#8101](https://www.sqlalchemy.org/trac/ticket/8101)

*   **[schema] [bug] [mssql]**

    Fixed issue where `Table` objects that made use of IDENTITY columns with a `Numeric` datatype would produce errors when attempting to reconcile the “autoincrement” column, preventing construction of the `Column` from using the `Column.autoincrement` parameter as well as emitting errors when attempting to invoke an `Insert` construct.

    References: [#8111](https://www.sqlalchemy.org/trac/ticket/8111)

### extensions

*   **[extensions] [bug]**

    Fixed bug in `Mutable` where pickling and unpickling of an ORM mapped instance would not correctly restore state for mappings that contained multiple `Mutable`-enabled attributes.

    References: [#8133](https://www.sqlalchemy.org/trac/ticket/8133)

## 1.4.37

Released: May 31, 2022

### orm

*   **[orm] [bug]**

    Fixed issue where using a `column_property()` construct containing a subquery against an already-mapped column attribute would not correctly apply ORM-compilation behaviors to the subquery, including that the “IN” expression added for a single-table inherits expression would fail to be included.

    References: [#8064](https://www.sqlalchemy.org/trac/ticket/8064)

*   **[orm] [bug]**

    Fixed issue where ORM results would apply incorrect key names to the returned `Row` objects in the case where the set of columns to be selected were changed, such as when using `Select.with_only_columns()`.

    References: [#8001](https://www.sqlalchemy.org/trac/ticket/8001)

*   **[orm] [bug] [oracle] [postgresql]**

    Fixed bug, likely a regression from 1.3, where usage of column names that require bound parameter escaping, more concretely when using Oracle with column names that require quoting such as those that start with an underscore, or in less common cases with some PostgreSQL drivers when using column names that contain percent signs, would cause the ORM versioning feature to not work correctly if the versioning column itself had such a name, as the ORM assumes certain bound parameter naming conventions that were being interfered with via the quotes. This issue is related to [#8053](https://www.sqlalchemy.org/trac/ticket/8053) and essentially revises the approach towards fixing this, revising the original issue [#5653](https://www.sqlalchemy.org/trac/ticket/5653) that created the initial implementation for generalized bound-parameter name quoting.

    References: [#8056](https://www.sqlalchemy.org/trac/ticket/8056)

### engine

*   **[engine] [bug] [tests]**

    Fixed issue where support for logging “stacklevel” implemented in [#7612](https://www.sqlalchemy.org/trac/ticket/7612) required adjustment to work with recently released Python 3.11.0b1, also repairs the unit tests which tested this feature.

    References: [#8019](https://www.sqlalchemy.org/trac/ticket/8019)

### sql

*   **[sql] [bug] [postgresql] [sqlite]**

    Fixed bug where the PostgreSQL `Insert.on_conflict_do_update()` method and the SQLite `Insert.on_conflict_do_update()` method would both fail to correctly accommodate a column with a separate “.key” when specifying the column using its key name in the dictionary passed to `Insert.on_conflict_do_update.set_`, as well as if the `Insert.excluded` collection were used as the dictionary directly.

    References: [#8014](https://www.sqlalchemy.org/trac/ticket/8014)

*   **[sql] [bug]**

    An informative error is raised for the use case where `Insert.from_select()` is being passed a “compound select” object such as a UNION, yet the INSERT statement needs to append additional columns to support Python-side or explicit SQL defaults from the table metadata. In this case a subquery of the compound object should be passed.

    References: [#8073](https://www.sqlalchemy.org/trac/ticket/8073)

*   **[sql] [bug]**

    Fixed an issue where using `bindparam()` with no explicit data or type given could be coerced into the incorrect type when used in expressions such as when using `Comparator.any()` and `Comparator.all()`.

    References: [#7979](https://www.sqlalchemy.org/trac/ticket/7979)

*   **[sql] [bug]**

    An informative error is raised if two individual `BindParameter` objects share the same name, yet one is used within an “expanding” context (typically an IN expression) and the other is not; mixing the same name in these two different styles of usage is not supported and typically the `expanding=True` parameter should be set on the parameters that are to receive list values outside of IN expressions (where `expanding` is set by default).

    References: [#8018](https://www.sqlalchemy.org/trac/ticket/8018)

### mysql

*   **[mysql] [bug]**

    Further adjustments to the MySQL PyODBC dialect to allow for complete connectivity, which was previously still not working despite fixes in [#7871](https://www.sqlalchemy.org/trac/ticket/7871).

    References: [#7966](https://www.sqlalchemy.org/trac/ticket/7966)

*   **[mysql] [bug]**

    Added disconnect code for MySQL error 4031, introduced in MySQL >= 8.0.24, indicating connection idle timeout exceeded. In particular this repairs an issue where pre-ping could not reconnect on a timed-out connection. Pull request courtesy valievkarim.

    References: [#8036](https://www.sqlalchemy.org/trac/ticket/8036)

### mssql

*   **[mssql] [bug]**

    Fix issue where a password with a leading “{” would result in login failure.

    References: [#8062](https://www.sqlalchemy.org/trac/ticket/8062)

*   **[mssql] [bug] [reflection]**

    Explicitly specify the collation when reflecting table columns using MSSQL to prevent “collation conflict” errors.

    References: [#8035](https://www.sqlalchemy.org/trac/ticket/8035)

### oracle

*   **[oracle] [usecase]**

    Added two new error codes for Oracle disconnect handling to support early testing of the new “python-oracledb” driver released by Oracle.

    References: [#8066](https://www.sqlalchemy.org/trac/ticket/8066)

*   **[oracle] [bug]**

    Fixed SQL compiler issue where the “bind processing” function for a bound parameter would not be correctly applied to a bound value if the bound parameter’s name were “escaped”. Concretely, this applies, among other cases, to Oracle when a `Column` has a name that itself requires quoting, such that the quoting-required name is then used for the bound parameters generated within DML statements, and the datatype in use requires bind processing, such as the `Enum` datatype.

    References: [#8053](https://www.sqlalchemy.org/trac/ticket/8053)

### orm

*   **[orm] [bug]**

    Fixed issue where using a `column_property()` construct containing a subquery against an already-mapped column attribute would not correctly apply ORM-compilation behaviors to the subquery, including that the “IN” expression added for a single-table inherits expression would fail to be included.

    References: [#8064](https://www.sqlalchemy.org/trac/ticket/8064)

*   **[orm] [bug]**

    Fixed issue where ORM results would apply incorrect key names to the returned `Row` objects in the case where the set of columns to be selected were changed, such as when using `Select.with_only_columns()`.

    References: [#8001](https://www.sqlalchemy.org/trac/ticket/8001)

*   **[orm] [bug] [oracle] [postgresql]**

    Fixed bug, likely a regression from 1.3, where usage of column names that require bound parameter escaping, more concretely when using Oracle with column names that require quoting such as those that start with an underscore, or in less common cases with some PostgreSQL drivers when using column names that contain percent signs, would cause the ORM versioning feature to not work correctly if the versioning column itself had such a name, as the ORM assumes certain bound parameter naming conventions that were being interfered with via the quotes. This issue is related to [#8053](https://www.sqlalchemy.org/trac/ticket/8053) and essentially revises the approach towards fixing this, revising the original issue [#5653](https://www.sqlalchemy.org/trac/ticket/5653) that created the initial implementation for generalized bound-parameter name quoting.

    References: [#8056](https://www.sqlalchemy.org/trac/ticket/8056)

### engine

*   **[engine] [bug] [tests]**

    Fixed issue where support for logging “stacklevel” implemented in [#7612](https://www.sqlalchemy.org/trac/ticket/7612) required adjustment to work with recently released Python 3.11.0b1, also repairs the unit tests which tested this feature.

    References: [#8019](https://www.sqlalchemy.org/trac/ticket/8019)

### sql

*   **[sql] [bug] [postgresql] [sqlite]**

    Fixed bug where the PostgreSQL `Insert.on_conflict_do_update()` method and the SQLite `Insert.on_conflict_do_update()` method would both fail to correctly accommodate a column with a separate “.key” when specifying the column using its key name in the dictionary passed to `Insert.on_conflict_do_update.set_`, as well as if the `Insert.excluded` collection were used as the dictionary directly.

    References: [#8014](https://www.sqlalchemy.org/trac/ticket/8014)

*   **[sql] [bug]**

    An informative error is raised for the use case where `Insert.from_select()` is being passed a “compound select” object such as a UNION, yet the INSERT statement needs to append additional columns to support Python-side or explicit SQL defaults from the table metadata. In this case a subquery of the compound object should be passed.

    References: [#8073](https://www.sqlalchemy.org/trac/ticket/8073)

*   **[sql] [bug]**

    Fixed an issue where using `bindparam()` with no explicit data or type given could be coerced into the incorrect type when used in expressions such as when using `Comparator.any()` and `Comparator.all()`.

    References: [#7979](https://www.sqlalchemy.org/trac/ticket/7979)

*   **[sql] [bug]**

    An informative error is raised if two individual `BindParameter` objects share the same name, yet one is used within an “expanding” context (typically an IN expression) and the other is not; mixing the same name in these two different styles of usage is not supported and typically the `expanding=True` parameter should be set on the parameters that are to receive list values outside of IN expressions (where `expanding` is set by default).

    References: [#8018](https://www.sqlalchemy.org/trac/ticket/8018)

### mysql

*   **[mysql] [bug]**

    Further adjustments to the MySQL PyODBC dialect to allow for complete connectivity, which was previously still not working despite fixes in [#7871](https://www.sqlalchemy.org/trac/ticket/7871).

    References: [#7966](https://www.sqlalchemy.org/trac/ticket/7966)

*   **[mysql] [bug]**

    Added disconnect code for MySQL error 4031, introduced in MySQL >= 8.0.24, indicating connection idle timeout exceeded. In particular this repairs an issue where pre-ping could not reconnect on a timed-out connection. Pull request courtesy valievkarim.

    References: [#8036](https://www.sqlalchemy.org/trac/ticket/8036)

### mssql

*   **[mssql] [bug]**

    Fix issue where a password with a leading “{” would result in login failure.

    References: [#8062](https://www.sqlalchemy.org/trac/ticket/8062)

*   **[mssql] [bug] [reflection]**

    Explicitly specify the collation when reflecting table columns using MSSQL to prevent “collation conflict” errors.

    References: [#8035](https://www.sqlalchemy.org/trac/ticket/8035)

### oracle

*   **[oracle] [usecase]**

    Added two new error codes for Oracle disconnect handling to support early testing of the new “python-oracledb” driver released by Oracle.

    References: [#8066](https://www.sqlalchemy.org/trac/ticket/8066)

*   **[oracle] [bug]**

    Fixed SQL compiler issue where the “bind processing” function for a bound parameter would not be correctly applied to a bound value if the bound parameter’s name were “escaped”. Concretely, this applies, among other cases, to Oracle when a `Column` has a name that itself requires quoting, such that the quoting-required name is then used for the bound parameters generated within DML statements, and the datatype in use requires bind processing, such as the `Enum` datatype.

    References: [#8053](https://www.sqlalchemy.org/trac/ticket/8053)

## 1.4.36

Released: April 26, 2022

### orm

*   **[orm] [bug] [regression]**

    Fixed regression where the change made for [#7861](https://www.sqlalchemy.org/trac/ticket/7861), released in version 1.4.33, that brought the `Insert` construct to be partially recognized as an ORM-enabled statement did not properly transfer the correct mapper / mapped table state to the `Session`, causing the `Session.get_bind()` method to fail for a `Session` that was bound to engines and/or connections using the `Session.binds` parameter.

    References: [#7936](https://www.sqlalchemy.org/trac/ticket/7936)

### orm declarative

*   **[orm] [declarative] [bug]**

    Modified the `DeclarativeMeta` metaclass to pass `cls.__dict__` into the declarative scanning process to look for attributes, rather than the separate dictionary passed to the type’s `__init__()` method. This allows user-defined base classes that add attributes within an `__init_subclass__()` to work as expected, as `__init_subclass__()` can only affect the `cls.__dict__` itself and not the other dictionary. This is technically a regression from 1.3 where `__dict__` was being used.

    References: [#7900](https://www.sqlalchemy.org/trac/ticket/7900)

### engine

*   **[engine] [bug]**

    Fixed a memory leak in the C extensions which could occur when calling upon named members of `Row` when the member does not exist under Python 3; in particular this could occur during NumPy transformations when it attempts to call members such as `.__array__`, but the issue was surrounding any `AttributeError` thrown by the `Row` object. This issue does not apply to version 2.0 which has already transitioned to Cython. Thanks much to Sebastian Berg for identifying the problem.

    References: [#7875](https://www.sqlalchemy.org/trac/ticket/7875)

*   **[engine] [bug]**

    Added a warning regarding a bug which exists in the `Result.columns()` method when passing 0 for the index in conjunction with a `Result` that will return a single ORM entity, which indicates that the current behavior of `Result.columns()` is broken in this case as the `Result` object will yield scalar values and not `Row` objects. The issue will be fixed in 2.0, which would be a backwards-incompatible change for code that relies on the current broken behavior. Code which wants to receive a collection of scalar values should use the `Result.scalars()` method, which will return a new `ScalarResult` object that yields non-row scalar objects.

    References: [#7953](https://www.sqlalchemy.org/trac/ticket/7953)

### schema

*   **[schema] [bug]**

    Fixed bug where `ForeignKeyConstraint` naming conventions using the `referred_column_0` naming convention key would not work if the foreign key constraint were set up as a `ForeignKey` object rather than an explicit `ForeignKeyConstraint` object. As this change makes use of a backport of some fixes from version 2.0, an additional little-known feature that has likely been broken for many years is also fixed which is that a `ForeignKey` object may refer to a referred table by name of the table alone without using a column name, if the name of the referent column is the same as that of the referred column.

    The `referred_column_0` naming convention key was previously not tested with the `ForeignKey` object, only `ForeignKeyConstraint`, and this bug reveals that the feature has never worked correctly unless `ForeignKeyConstraint` is used for all FK constraints. This bug traces back to the original introduction of the feature introduced for [#3989](https://www.sqlalchemy.org/trac/ticket/3989).

    References: [#7958](https://www.sqlalchemy.org/trac/ticket/7958)

### asyncio

*   **[asyncio] [bug]**

    Repaired handling of `contextvar.ContextVar` objects inside of async adapted event handlers. Previously, values applied to a `ContextVar` would not be propagated in the specific case of calling upon awaitables inside of non-awaitable code.

    References: [#7937](https://www.sqlalchemy.org/trac/ticket/7937)

### postgresql

*   **[postgresql] [bug]**

    Fixed bug in `ARRAY` datatype in combination with `Enum` on PostgreSQL where using the `.any()` or `.all()` methods to render SQL ANY() or ALL(), given members of the Python enumeration as arguments, would produce a type adaptation failure on all drivers.

    References: [#6515](https://www.sqlalchemy.org/trac/ticket/6515)

*   **[postgresql] [bug]**

    Implemented `UUID.python_type` attribute for the PostgreSQL `UUID` type object. The attribute will return either `str` or `uuid.UUID` based on the `UUID.as_uuid` parameter setting. Previously, this attribute was unimplemented. Pull request courtesy Alex Grönholm.

    References: [#7943](https://www.sqlalchemy.org/trac/ticket/7943)

*   **[postgresql] [bug]**

    Fixed an issue in the psycopg2 dialect when using the `create_engine.pool_pre_ping` parameter which would cause user-configured `AUTOCOMMIT` isolation level to be inadvertently reset by the “ping” handler.

    References: [#7930](https://www.sqlalchemy.org/trac/ticket/7930)

### mysql

*   **[mysql] [bug] [regression]**

    Fixed a regression in the untested MySQL PyODBC dialect caused by the fix for [#7518](https://www.sqlalchemy.org/trac/ticket/7518) in version 1.4.32 where an argument was being propagated incorrectly upon first connect, leading to a `TypeError`.

    References: [#7871](https://www.sqlalchemy.org/trac/ticket/7871)

### tests

*   **[tests] [bug]**

    For third party dialects, repaired a missing requirement for the `SimpleUpdateDeleteTest` suite test which was not checking for a working “rowcount” function on the target dialect.

    References: [#7919](https://www.sqlalchemy.org/trac/ticket/7919)

### orm

*   **[orm] [bug] [regression]**

    Fixed regression where the change made for [#7861](https://www.sqlalchemy.org/trac/ticket/7861), released in version 1.4.33, that brought the `Insert` construct to be partially recognized as an ORM-enabled statement did not properly transfer the correct mapper / mapped table state to the `Session`, causing the `Session.get_bind()` method to fail for a `Session` that was bound to engines and/or connections using the `Session.binds` parameter.

    References: [#7936](https://www.sqlalchemy.org/trac/ticket/7936)

### orm declarative

*   **[orm] [declarative] [bug]**

    Modified the `DeclarativeMeta` metaclass to pass `cls.__dict__` into the declarative scanning process to look for attributes, rather than the separate dictionary passed to the type’s `__init__()` method. This allows user-defined base classes that add attributes within an `__init_subclass__()` to work as expected, as `__init_subclass__()` can only affect the `cls.__dict__` itself and not the other dictionary. This is technically a regression from 1.3 where `__dict__` was being used.

    References: [#7900](https://www.sqlalchemy.org/trac/ticket/7900)

### engine

*   **[engine] [bug]**

    Fixed a memory leak in the C extensions which could occur when calling upon named members of `Row` when the member does not exist under Python 3; in particular this could occur during NumPy transformations when it attempts to call members such as `.__array__`, but the issue was surrounding any `AttributeError` thrown by the `Row` object. This issue does not apply to version 2.0 which has already transitioned to Cython. Thanks much to Sebastian Berg for identifying the problem.

    References: [#7875](https://www.sqlalchemy.org/trac/ticket/7875)

*   **[engine] [bug]**

    Added a warning regarding a bug which exists in the `Result.columns()` method when passing 0 for the index in conjunction with a `Result` that will return a single ORM entity, which indicates that the current behavior of `Result.columns()` is broken in this case as the `Result` object will yield scalar values and not `Row` objects. The issue will be fixed in 2.0, which would be a backwards-incompatible change for code that relies on the current broken behavior. Code which wants to receive a collection of scalar values should use the `Result.scalars()` method, which will return a new `ScalarResult` object that yields non-row scalar objects.

    References: [#7953](https://www.sqlalchemy.org/trac/ticket/7953)

### schema

*   **[schema] [bug]**

    Fixed bug where `ForeignKeyConstraint` naming conventions using the `referred_column_0` naming convention key would not work if the foreign key constraint were set up as a `ForeignKey` object rather than an explicit `ForeignKeyConstraint` object. As this change makes use of a backport of some fixes from version 2.0, an additional little-known feature that has likely been broken for many years is also fixed which is that a `ForeignKey` object may refer to a referred table by name of the table alone without using a column name, if the name of the referent column is the same as that of the referred column.

    The `referred_column_0` naming convention key was previously not tested with the `ForeignKey` object, only `ForeignKeyConstraint`, and this bug reveals that the feature has never worked correctly unless `ForeignKeyConstraint` is used for all FK constraints. This bug traces back to the original introduction of the feature introduced for [#3989](https://www.sqlalchemy.org/trac/ticket/3989).

    References: [#7958](https://www.sqlalchemy.org/trac/ticket/7958)

### asyncio

*   **[asyncio] [bug]**

    Repaired handling of `contextvar.ContextVar` objects inside of async adapted event handlers. Previously, values applied to a `ContextVar` would not be propagated in the specific case of calling upon awaitables inside of non-awaitable code.

    References: [#7937](https://www.sqlalchemy.org/trac/ticket/7937)

### postgresql

*   **[postgresql] [bug]**

    Fixed bug in `ARRAY` datatype in combination with `Enum` on PostgreSQL where using the `.any()` or `.all()` methods to render SQL ANY() or ALL(), given members of the Python enumeration as arguments, would produce a type adaptation failure on all drivers.

    References: [#6515](https://www.sqlalchemy.org/trac/ticket/6515)

*   **[postgresql] [bug]**

    Implemented `UUID.python_type` attribute for the PostgreSQL `UUID` type object. The attribute will return either `str` or `uuid.UUID` based on the `UUID.as_uuid` parameter setting. Previously, this attribute was unimplemented. Pull request courtesy Alex Grönholm.

    References: [#7943](https://www.sqlalchemy.org/trac/ticket/7943)

*   **[postgresql] [bug]**

    Fixed an issue in the psycopg2 dialect when using the `create_engine.pool_pre_ping` parameter which would cause user-configured `AUTOCOMMIT` isolation level to be inadvertently reset by the “ping” handler.

    References: [#7930](https://www.sqlalchemy.org/trac/ticket/7930)

### mysql

*   **[mysql] [bug] [regression]**

    Fixed a regression in the untested MySQL PyODBC dialect caused by the fix for [#7518](https://www.sqlalchemy.org/trac/ticket/7518) in version 1.4.32 where an argument was being propagated incorrectly upon first connect, leading to a `TypeError`.

    References: [#7871](https://www.sqlalchemy.org/trac/ticket/7871)

### tests

*   **[tests] [bug]**

    For third party dialects, repaired a missing requirement for the `SimpleUpdateDeleteTest` suite test which was not checking for a working “rowcount” function on the target dialect.

    References: [#7919](https://www.sqlalchemy.org/trac/ticket/7919)

## 1.4.35

Released: April 6, 2022

### sql

*   **[sql] [bug]**

    Fixed bug in newly implemented `FunctionElement.table_valued.joins_implicitly` feature where the parameter would not automatically propagate from the original `TableValuedAlias` object to the secondary object produced when calling upon `TableValuedAlias.render_derived()` or `TableValuedAlias.alias()`.

    Additionally repaired these issues in `TableValuedAlias`:

    *   repaired a potential memory issue which could occur when repeatedly calling `TableValuedAlias.render_derived()` against successive copies of the same object (for .alias(), we currently have to still continue chaining from the previous element. not sure if this can be improved but this is standard behavior for .alias() elsewhere)

    *   repaired issue where the individual element types would be lost when calling upon `TableValuedAlias.render_derived()` or `TableValuedAlias.alias()`.

    References: [#7890](https://www.sqlalchemy.org/trac/ticket/7890)

*   **[sql] [bug] [regression]**

    Fixed regression caused by [#7823](https://www.sqlalchemy.org/trac/ticket/7823) which impacted the caching system, such that bound parameters that had been “cloned” within ORM operations, such as polymorphic loading, would in some cases not acquire their correct execution-time value leading to incorrect bind values being rendered.

    References: [#7903](https://www.sqlalchemy.org/trac/ticket/7903)

### sql

*   **[sql] [bug]**

    Fixed bug in newly implemented `FunctionElement.table_valued.joins_implicitly` feature where the parameter would not automatically propagate from the original `TableValuedAlias` object to the secondary object produced when calling upon `TableValuedAlias.render_derived()` or `TableValuedAlias.alias()`.

    Additionally repaired these issues in `TableValuedAlias`:

    *   repaired a potential memory issue which could occur when repeatedly calling `TableValuedAlias.render_derived()` against successive copies of the same object (for .alias(), we currently have to still continue chaining from the previous element. not sure if this can be improved but this is standard behavior for .alias() elsewhere)

    *   repaired issue where the individual element types would be lost when calling upon `TableValuedAlias.render_derived()` or `TableValuedAlias.alias()`.

    References: [#7890](https://www.sqlalchemy.org/trac/ticket/7890)

*   **[sql] [bug] [regression]**

    Fixed regression caused by [#7823](https://www.sqlalchemy.org/trac/ticket/7823) which impacted the caching system, such that bound parameters that had been “cloned” within ORM operations, such as polymorphic loading, would in some cases not acquire their correct execution-time value leading to incorrect bind values being rendered.

    References: [#7903](https://www.sqlalchemy.org/trac/ticket/7903)

## 1.4.34

Released: March 31, 2022

### orm

*   **[orm] [bug] [regression]**

    Fixed regression caused by [#7861](https://www.sqlalchemy.org/trac/ticket/7861) where invoking an `Insert` construct which contained ORM entities directly via `Session.execute()` would fail.

    References: [#7878](https://www.sqlalchemy.org/trac/ticket/7878)

### postgresql

*   **[postgresql] [bug]**

    Scaled back a fix made for [#6581](https://www.sqlalchemy.org/trac/ticket/6581) where “executemany values” mode for psycopg2 were disabled for all “ON CONFLICT” styles of INSERT, to not apply to the “ON CONFLICT DO NOTHING” clause, which does not include any parameters and is safe for “executemany values” mode. “ON CONFLICT DO UPDATE” is still blocked from “executemany values” as there may be additional parameters in the DO UPDATE clause that cannot be batched (which is the original issue fixed by [#6581](https://www.sqlalchemy.org/trac/ticket/6581)).

    References: [#7880](https://www.sqlalchemy.org/trac/ticket/7880)

### orm

*   **[orm] [bug] [regression]**

    Fixed regression caused by [#7861](https://www.sqlalchemy.org/trac/ticket/7861) where invoking an `Insert` construct which contained ORM entities directly via `Session.execute()` would fail.

    References: [#7878](https://www.sqlalchemy.org/trac/ticket/7878)

### postgresql

*   **[postgresql] [bug]**

    Scaled back a fix made for [#6581](https://www.sqlalchemy.org/trac/ticket/6581) where “executemany values” mode for psycopg2 were disabled for all “ON CONFLICT” styles of INSERT, to not apply to the “ON CONFLICT DO NOTHING” clause, which does not include any parameters and is safe for “executemany values” mode. “ON CONFLICT DO UPDATE” is still blocked from “executemany values” as there may be additional parameters in the DO UPDATE clause that cannot be batched (which is the original issue fixed by [#6581](https://www.sqlalchemy.org/trac/ticket/6581)).

    References: [#7880](https://www.sqlalchemy.org/trac/ticket/7880)

## 1.4.33

Released: March 31, 2022

### orm

*   **[orm] [usecase]**

    Added `with_polymorphic.adapt_on_names` to the `with_polymorphic()` function, which allows a polymorphic load (typically with concrete mapping) to be stated against an alternative selectable that will adapt to the original mapped selectable on column names alone.

    References: [#7805](https://www.sqlalchemy.org/trac/ticket/7805)

*   **[orm] [usecase]**

    Added new attributes `UpdateBase.returning_column_descriptions` and `UpdateBase.entity_description` to allow for inspection of ORM attributes and entities that are installed as part of an `Insert`, `Update`, or `Delete` construct. The `Select.column_descriptions` accessor is also now implemented for Core-only selectables.

    References: [#7861](https://www.sqlalchemy.org/trac/ticket/7861)

*   **[orm] [performance] [bug]**

    Improvements in memory usage by the ORM, removing a significant set of intermediary expression objects that are typically stored when a copy of an expression object is created. These clones have been greatly reduced, reducing the number of total expression objects stored in memory by ORM mappings by about 30%.

    References: [#7823](https://www.sqlalchemy.org/trac/ticket/7823)

*   **[orm] [bug] [regression]**

    Fixed regression in “dynamic” loader strategy where the `Query.filter_by()` method would not be given an appropriate entity to filter from, in the case where a “secondary” table were present in the relationship being queried and the mapping were against something complex such as a “with polymorphic”.

    References: [#7868](https://www.sqlalchemy.org/trac/ticket/7868)

*   **[orm] [bug]**

    Fixed bug where `composite()` attributes would not work in conjunction with the `selectin_polymorphic()` loader strategy for joined table inheritance.

    References: [#7801](https://www.sqlalchemy.org/trac/ticket/7801)

*   **[orm] [bug]**

    Fixed issue where the `selectin_polymorphic()` loader option would not work with joined inheritance mappers that don’t have a fixed “polymorphic_on” column. Additionally added test support for a wider variety of usage patterns with this construct.

    References: [#7799](https://www.sqlalchemy.org/trac/ticket/7799)

*   **[orm] [bug]**

    Fixed bug in `with_loader_criteria()` function where loader criteria would not be applied to a joined eager load that were invoked within the scope of a refresh operation for the parent object.

    References: [#7862](https://www.sqlalchemy.org/trac/ticket/7862)

*   **[orm] [bug]**

    Fixed issue where the `Mapper` would reduce a user-defined `Mapper.primary_key` argument too aggressively, in the case of mapping to a `UNION` where for some of the SELECT entries, two columns are essentially equivalent, but in another, they are not, such as in a recursive CTE. The logic here has been changed to accept a given user-defined PK as given, where columns will be related to the mapped selectable but no longer “reduced” as this heuristic can’t accommodate for all situations.

    References: [#7842](https://www.sqlalchemy.org/trac/ticket/7842)

### engine

*   **[engine] [usecase]**

    Added new parameter `Engine.dispose.close`, defaulting to True. When False, the engine disposal does not touch the connections in the old pool at all, simply dropping the pool and replacing it. This use case is so that when the original pool is transferred from a parent process, the parent process may continue to use those connections.

    See also

    Using Connection Pools with Multiprocessing or os.fork() - revised documentation

    References: [#7815](https://www.sqlalchemy.org/trac/ticket/7815), [#7877](https://www.sqlalchemy.org/trac/ticket/7877)

*   **[engine] [bug]**

    Further clarified connection-level logging to indicate the BEGIN, ROLLBACK and COMMIT log messages do not actually indicate a real transaction when the AUTOCOMMIT isolation level is in use; messaging has been extended to include the BEGIN message itself, and the messaging has also been fixed to accommodate when the `Engine` level `create_engine.isolation_level` parameter was used directly.

    References: [#7853](https://www.sqlalchemy.org/trac/ticket/7853)

### sql

*   **[sql] [usecase]**

    Added new parameter `FunctionElement.table_valued.joins_implicitly`, for the `FunctionElement.table_valued()` construct. This parameter indicates that the table-valued function provided will automatically perform an implicit join with the referenced table. This effectively disables the ‘from linting’ feature, such as the ‘cartesian product’ warning, from triggering due to the presence of this parameter. May be used for functions such as `func.json_each()`.

    References: [#7845](https://www.sqlalchemy.org/trac/ticket/7845)

*   **[sql] [bug]**

    The `bindparam.literal_execute` parameter now takes part of the cache generation of a `bindparam()`, since it changes the sql string generated by the compiler. Previously the correct bind values were used, but the `literal_execute` would be ignored on subsequent executions of the same query.

    References: [#7876](https://www.sqlalchemy.org/trac/ticket/7876)

*   **[sql] [bug] [regression]**

    Fixed regression caused by [#7760](https://www.sqlalchemy.org/trac/ticket/7760) where the new capabilities of `TextualSelect` were not fully implemented within the compiler properly, leading to issues with composed INSERT constructs such as “INSERT FROM SELECT” and “INSERT…ON CONFLICT” when combined with CTE and textual statements.

    References: [#7798](https://www.sqlalchemy.org/trac/ticket/7798)

### schema

*   **[schema] [usecase]**

    Added support so that the `Table.to_metadata.referred_schema_fn` callable passed to `Table.to_metadata()` may return the value `BLANK_SCHEMA` to indicate that the referenced foreign key should be reset to None. The `RETAIN_SCHEMA` symbol may also be returned from this function to indicate “no change”, which will behave the same as `None` currently does which also indicates no change.

    References: [#7860](https://www.sqlalchemy.org/trac/ticket/7860)

### sqlite

*   **[sqlite] [bug] [reflection]**

    Fixed bug where the name of CHECK constraints under SQLite would not be reflected if the name were created using quotes, as is the case when the name uses mixed case or special characters.

    References: [#5463](https://www.sqlalchemy.org/trac/ticket/5463)

### mssql

*   **[mssql] [bug] [regression]**

    Fixed regression caused by [#7160](https://www.sqlalchemy.org/trac/ticket/7160) where FK reflection in conjunction with a low compatibility level setting (compatibility level 80: SQL Server 2000) causes an “Ambiguous column name” error. Patch courtesy @Lin-Your.

    References: [#7812](https://www.sqlalchemy.org/trac/ticket/7812)

### misc

*   **[bug] [ext]**

    Improved the error message that’s raised for the case where the `association_proxy()` construct attempts to access a target attribute at the class level, and this access fails. The particular use case here is when proxying to a hybrid attribute that does not include a working class-level implementation.

    References: [#7827](https://www.sqlalchemy.org/trac/ticket/7827)

### orm

*   **[orm] [usecase]**

    Added `with_polymorphic.adapt_on_names` to the `with_polymorphic()` function, which allows a polymorphic load (typically with concrete mapping) to be stated against an alternative selectable that will adapt to the original mapped selectable on column names alone.

    References: [#7805](https://www.sqlalchemy.org/trac/ticket/7805)

*   **[orm] [usecase]**

    Added new attributes `UpdateBase.returning_column_descriptions` and `UpdateBase.entity_description` to allow for inspection of ORM attributes and entities that are installed as part of an `Insert`, `Update`, or `Delete` construct. The `Select.column_descriptions` accessor is also now implemented for Core-only selectables.

    References: [#7861](https://www.sqlalchemy.org/trac/ticket/7861)

*   **[orm] [performance] [bug]**

    Improvements in memory usage by the ORM, removing a significant set of intermediary expression objects that are typically stored when a copy of an expression object is created. These clones have been greatly reduced, reducing the number of total expression objects stored in memory by ORM mappings by about 30%.

    References: [#7823](https://www.sqlalchemy.org/trac/ticket/7823)

*   **[orm] [bug] [regression]**

    Fixed regression in “dynamic” loader strategy where the `Query.filter_by()` method would not be given an appropriate entity to filter from, in the case where a “secondary” table were present in the relationship being queried and the mapping were against something complex such as a “with polymorphic”.

    References: [#7868](https://www.sqlalchemy.org/trac/ticket/7868)

*   **[orm] [bug]**

    Fixed bug where `composite()` attributes would not work in conjunction with the `selectin_polymorphic()` loader strategy for joined table inheritance.

    References: [#7801](https://www.sqlalchemy.org/trac/ticket/7801)

*   **[orm] [bug]**

    Fixed issue where the `selectin_polymorphic()` loader option would not work with joined inheritance mappers that don’t have a fixed “polymorphic_on” column. Additionally added test support for a wider variety of usage patterns with this construct.

    References: [#7799](https://www.sqlalchemy.org/trac/ticket/7799)

*   **[orm] [bug]**

    Fixed bug in `with_loader_criteria()` function where loader criteria would not be applied to a joined eager load that were invoked within the scope of a refresh operation for the parent object.

    References: [#7862](https://www.sqlalchemy.org/trac/ticket/7862)

*   **[orm] [bug]**

    Fixed issue where the `Mapper` would reduce a user-defined `Mapper.primary_key` argument too aggressively, in the case of mapping to a `UNION` where for some of the SELECT entries, two columns are essentially equivalent, but in another, they are not, such as in a recursive CTE. The logic here has been changed to accept a given user-defined PK as given, where columns will be related to the mapped selectable but no longer “reduced” as this heuristic can’t accommodate for all situations.

    References: [#7842](https://www.sqlalchemy.org/trac/ticket/7842)

### engine

*   **[engine] [usecase]**

    Added new parameter `Engine.dispose.close`, defaulting to True. When False, the engine disposal does not touch the connections in the old pool at all, simply dropping the pool and replacing it. This use case is so that when the original pool is transferred from a parent process, the parent process may continue to use those connections.

    See also

    Using Connection Pools with Multiprocessing or os.fork() - revised documentation

    References: [#7815](https://www.sqlalchemy.org/trac/ticket/7815), [#7877](https://www.sqlalchemy.org/trac/ticket/7877)

*   **[engine] [bug]**

    Further clarified connection-level logging to indicate the BEGIN, ROLLBACK and COMMIT log messages do not actually indicate a real transaction when the AUTOCOMMIT isolation level is in use; messaging has been extended to include the BEGIN message itself, and the messaging has also been fixed to accommodate when the `Engine` level `create_engine.isolation_level` parameter was used directly.

    References: [#7853](https://www.sqlalchemy.org/trac/ticket/7853)

### sql

*   **[sql] [usecase]**

    Added new parameter `FunctionElement.table_valued.joins_implicitly`, for the `FunctionElement.table_valued()` construct. This parameter indicates that the table-valued function provided will automatically perform an implicit join with the referenced table. This effectively disables the ‘from linting’ feature, such as the ‘cartesian product’ warning, from triggering due to the presence of this parameter. May be used for functions such as `func.json_each()`.

    References: [#7845](https://www.sqlalchemy.org/trac/ticket/7845)

*   **[sql] [bug]**

    The `bindparam.literal_execute` parameter now takes part of the cache generation of a `bindparam()`, since it changes the sql string generated by the compiler. Previously the correct bind values were used, but the `literal_execute` would be ignored on subsequent executions of the same query.

    References: [#7876](https://www.sqlalchemy.org/trac/ticket/7876)

*   **[sql] [bug] [regression]**

    Fixed regression caused by [#7760](https://www.sqlalchemy.org/trac/ticket/7760) where the new capabilities of `TextualSelect` were not fully implemented within the compiler properly, leading to issues with composed INSERT constructs such as “INSERT FROM SELECT” and “INSERT…ON CONFLICT” when combined with CTE and textual statements.

    References: [#7798](https://www.sqlalchemy.org/trac/ticket/7798)

### schema

*   **[schema] [usecase]**

    Added support so that the `Table.to_metadata.referred_schema_fn` callable passed to `Table.to_metadata()` may return the value `BLANK_SCHEMA` to indicate that the referenced foreign key should be reset to None. The `RETAIN_SCHEMA` symbol may also be returned from this function to indicate “no change”, which will behave the same as `None` currently does which also indicates no change.

    References: [#7860](https://www.sqlalchemy.org/trac/ticket/7860)

### sqlite

*   **[sqlite] [bug] [reflection]**

    Fixed bug where the name of CHECK constraints under SQLite would not be reflected if the name were created using quotes, as is the case when the name uses mixed case or special characters.

    References: [#5463](https://www.sqlalchemy.org/trac/ticket/5463)

### mssql

*   **[mssql] [bug] [regression]**

    Fixed regression caused by [#7160](https://www.sqlalchemy.org/trac/ticket/7160) where FK reflection in conjunction with a low compatibility level setting (compatibility level 80: SQL Server 2000) causes an “Ambiguous column name” error. Patch courtesy @Lin-Your.

    References: [#7812](https://www.sqlalchemy.org/trac/ticket/7812)

### misc

*   **[bug] [ext]**

    Improved the error message that’s raised for the case where the `association_proxy()` construct attempts to access a target attribute at the class level, and this access fails. The particular use case here is when proxying to a hybrid attribute that does not include a working class-level implementation.

    References: [#7827](https://www.sqlalchemy.org/trac/ticket/7827)

## 1.4.32

Released: March 6, 2022

### orm

*   **[orm] [bug] [regression]**

    Fixed regression where the ORM exception that is to be raised when an INSERT silently fails to actually insert a row (such as from a trigger) would not be reached, due to a runtime exception raised ahead of time due to the missing primary key value, thus raising an uninformative exception rather than the correct one. For 1.4 and above, a new `FlushError` is added for this case that’s raised earlier than the previous “null identity” exception was for 1.3, as a situation where the number of rows actually INSERTed does not match what was expected is a more critical situation in 1.4 as it prevents batching of multiple objects from working correctly. This is separate from the case where a newly fetched primary key is fetched as NULL, which continues to raise the existing “null identity” exception.

    References: [#7594](https://www.sqlalchemy.org/trac/ticket/7594)

*   **[orm] [bug]**

    Fixed issue where using a fully qualified path for the classname in `relationship()` that nonetheless contained an incorrect name for path tokens that were not the first token, would fail to raise an informative error and would instead fail randomly at a later step.

    References: [#7697](https://www.sqlalchemy.org/trac/ticket/7697)

### engine

*   **[engine] [bug]**

    Adjusted the logging for key SQLAlchemy components including `Engine`, `Connection` to establish an appropriate stack level parameter, so that the Python logging tokens `funcName` and `lineno` when used in custom logging formatters will report the correct information, which can be useful when filtering log output; supported on Python 3.8 and above. Pull request courtesy Markus Gerstel.

    References: [#7612](https://www.sqlalchemy.org/trac/ticket/7612)

### sql

*   **[sql] [bug]**

    Fixed type-related error messages that would fail for values that were tuples, due to string formatting syntax, including compile of unsupported literal values and invalid boolean values.

    References: [#7721](https://www.sqlalchemy.org/trac/ticket/7721)

*   **[sql] [bug] [mysql]**

    Fixed issues in MySQL `SET` datatype as well as the generic `Enum` datatype where the `__repr__()` method would not render all optional parameters in the string output, impacting the use of these types in Alembic autogenerate. Pull request for MySQL courtesy Yuki Nishimine.

    References: [#7598](https://www.sqlalchemy.org/trac/ticket/7598), [#7720](https://www.sqlalchemy.org/trac/ticket/7720), [#7789](https://www.sqlalchemy.org/trac/ticket/7789)

*   **[sql] [bug]**

    The `Enum` datatype now emits a warning if the `Enum.length` argument is specified without also specifying `Enum.native_enum` as False, as the parameter is otherwise silently ignored in this case, despite the fact that the `Enum` datatype will still render VARCHAR DDL on backends that don’t have a native ENUM datatype such as SQLite. This behavior may change in a future release so that “length” is honored for all non-native “enum” types regardless of the “native_enum” setting.

*   **[sql] [bug]**

    Fixed issue where the `HasCTE.add_cte()` method as called upon a `TextualSelect` instance was not being accommodated by the SQL compiler. The fix additionally adds more “SELECT”-like compiler behavior to `TextualSelect` including that DML CTEs such as UPDATE and INSERT may be accommodated.

    References: [#7760](https://www.sqlalchemy.org/trac/ticket/7760)

### asyncio

*   **[asyncio] [bug]**

    Fixed issues where a descriptive error message was not raised for some classes of event listening with an async engine, which should instead be a sync engine instance.

*   **[asyncio] [bug]**

    Fixed issue where the `AsyncSession.execute()` method failed to raise an informative exception if the `Connection.execution_options.stream_results` execution option were used, which is incompatible with a sync-style `Result` object when using an asyncio calling style, as the operation to fetch more rows would need to be awaited. An exception is now raised in this scenario in the same way one was already raised when the `Connection.execution_options.stream_results` option would be used with the `AsyncConnection.execute()` method.

    Additionally, for improved stability with state-sensitive database drivers such as asyncmy, the cursor is now closed when this error condition is raised; previously with the asyncmy dialect, the connection would go into an invalid state with unconsumed server side results remaining.

    References: [#7667](https://www.sqlalchemy.org/trac/ticket/7667)

### postgresql

*   **[postgresql] [usecase]**

    Added compiler support for the PostgreSQL `NOT VALID` phrase when rendering DDL for the `CheckConstraint`, `ForeignKeyConstraint` and `ForeignKey` schema constructs. Pull request courtesy Gilbert Gilb’s.

    See also

    PostgreSQL Constraint Options

    References: [#7600](https://www.sqlalchemy.org/trac/ticket/7600)

### mysql

*   **[mysql] [bug] [regression]**

    Fixed regression caused by [#7518](https://www.sqlalchemy.org/trac/ticket/7518) where changing the syntax “SHOW VARIABLES” to “SELECT @@” broke compatibility with MySQL versions older than 5.6, including early 5.0 releases. While these are very old MySQL versions, a change in compatibility was not planned, so version-specific logic has been restored to fall back to “SHOW VARIABLES” for MySQL server versions < 5.6.

    References: [#7518](https://www.sqlalchemy.org/trac/ticket/7518)

### mariadb

*   **[mariadb] [bug] [regression]**

    Fixed regression in mariadbconnector dialect as of mariadb connector 1.0.10 where the DBAPI no longer pre-buffers cursor.lastrowid, leading to errors when inserting objects with the ORM as well as causing non-availability of the `CursorResult.inserted_primary_key` attribute. The dialect now fetches this value proactively for situations where it applies.

    References: [#7738](https://www.sqlalchemy.org/trac/ticket/7738)

### sqlite

*   **[sqlite] [usecase]**

    Added support for reflecting SQLite inline unique constraints where the column names are formatted with SQLite “escape quotes” `[]` or ``` 格式化，这些在生成列名时被数据库丢弃。

    参考：[#7736](https://www.sqlalchemy.org/trac/ticket/7736)

+   **[sqlite] [bug]**

    修复了 SQLite 唯一约束反射无法检测到列内联 UNIQUE 约束的问题，其中列名中带有下划线。

    参考：[#7736](https://www.sqlalchemy.org/trac/ticket/7736)

### oracle

+   **[oracle] [bug]**

    修复了 Oracle 方言中的问题，当使用一个作为绑定参数时需要引号的列名，例如`"_id"`，由于绑定参数重写缺少此值，导致无法正确跟踪 Python 生成的默认值，从而引发 Oracle 错误。

    参考：[#7676](https://www.sqlalchemy.org/trac/ticket/7676)

+   **[oracle] [错误] [回归]**

    添加了对从 cx_Oracle 异常对象中解析“DPI”错误代码的支持，例如`DPI-1080`和`DPI-1010`，这两个错误代码现在均表示断开连接的情况，从 cx_Oracle 8.3 开始。

    参考：[#7748](https://www.sqlalchemy.org/trac/ticket/7748)

### 测试

+   **[测试] [错误]**

    对测试套件与 pytest 的集成进行了改进，以使“warnings”插件（如果手动启用）不会干扰测试套件，从而第三方可以启用警告插件或使用`-W`参数，而 SQLAlchemy 的测试套件仍将通过。另外，现代化了“pytest-xdist”插件的检测，以便全局禁用插件，即使 xdist 仍然安装，也不会破坏测试套件。将促使将弃用警告升级为错误的警告过滤器本地化到 SQLAlchemy 特定的警告中，或在 SQLAlchemy 特定的源中用于一般的 Python 弃用警告，以便来自 pytest 插件的非 SQLAlchemy 弃用警告也不会影响测试套件。

    参考：[#7599](https://www.sqlalchemy.org/trac/ticket/7599)

+   **[测试] [错误]**

    对默认的 pytest 配置进行了修正，以修复测试发现配置的问题，解决了测试套件未正确配置警告和尝试将示例套件加载为测试的问题，具体情况是 SQLAlchemy 检出位于具有名为“test”的上级目录的绝对路径中。

    参考：[#7045](https://www.sqlalchemy.org/trac/ticket/7045)

### orm

+   **[orm] [错误] [回归]**

    修复了一个回归问题，即当插入操作未实际插入行（例如来自触发器）时，将引发 ORM 异常未能被触及，由于提前由于缺少主键值而引发的运行时异常，因此引发了一个无信息的异常而不是正确的异常。对于 1.4 及以上版本，为此情况添加了一个新的`FlushError`，它比 1.3 的先前“空标识”异常更早地被引发，因为实际插入的行数与预期的不匹配是 1.4 中更为关键的情况，因为它阻止了多个对象的批处理工作的正确进行。这与新获取的主键被获取为 NULL 的情况不同，后者继续引发现有的“空标识”异常。

    参考：[#7594](https://www.sqlalchemy.org/trac/ticket/7594)

+   **[orm] [错误]**

    修复了一个问题，即在`relationship()`中对类名使用了完全限定路径，但对于不是第一个标记的路径标记包含了不正确的名称时，不会引发一个详细的错误，而是会在后续步骤中随机失败。

    参考：[#7697](https://www.sqlalchemy.org/trac/ticket/7697)

### engine

+   **[engine] [bug]**

    为了调整关键的 SQLAlchemy 组件的日志记录，包括`Engine`、`Connection`等，以建立适当的堆栈级别参数，这样当在自定义日志格式化器中使用 Python 日志标记`funcName`和`lineno`时，将报告正确的信息，这在过滤日志输出时非常有用；支持 Python 3.8 及以上版本。此次贡献由 Markus Gerstel 提供。

    参考：[#7612](https://www.sqlalchemy.org/trac/ticket/7612)

### sql

+   **[sql] [bug]**

    修复了与类型相关的错误消息，这些错误消息对于元组值会失败，因为字符串格式化语法不支持，包括不支持的文字值的编译和无效的布尔值。

    参考：[#7721](https://www.sqlalchemy.org/trac/ticket/7721)

+   **[sql] [bug] [mysql]**

    修复了 MySQL `SET` 数据类型以及通用 `Enum` 数据类型中的问题，其中 `__repr__()` 方法不会在字符串输出中呈现所有可选参数，这影响了在 Alembic 自动生成中使用这些类型。此次 MySQL 贡献由 Yuki Nishimine 提供。

    参考：[#7598](https://www.sqlalchemy.org/trac/ticket/7598)，[#7720](https://www.sqlalchemy.org/trac/ticket/7720)，[#7789](https://www.sqlalchemy.org/trac/ticket/7789)

+   **[sql] [bug]**

    当指定了`Enum`数据类型的`Enum.length`参数而没有指定`Enum.native_enum`时，现在会发出警告，因为在这种情况下，参数会被静默忽略，尽管`Enum`数据类型仍将在没有本地 ENUM 数据类型（例如 SQLite）的后端上呈现 VARCHAR DDL。这种行为可能会在将来的版本中发生变化，以便对于所有非本地“enum”类型都尊重“length”设置，而不管“native_enum”设置如何。

+   **[sql] [bug]**

    修复了在调用`HasCTE.add_cte()`方法时，对`TextualSelect`实例未被 SQL 编译器处理的问题。此修复还向`TextualSelect`添加了更多类似于“SELECT”的编译器行为，包括可以容纳 DML CTE（如 UPDATE 和 INSERT）。

    参考：[#7760](https://www.sqlalchemy.org/trac/ticket/7760)

### asyncio

+   **[asyncio] [bug]**

    修复了在使用异步引擎进行事件监听时未引发描述性错误消息的问题，应该改为使用同步引擎实例。

+   **[asyncio] [bug]**

    修复了`AsyncSession.execute()`方法在使用`Connection.execution_options.stream_results`执行选项时未引发信息丰富的异常的问题，这与使用异步调用风格时的同步式`Result`对象不兼容，因为获取更多行的操作需要等待。在这种情况下，现在会引发异常，就像在使用`Connection.execution_options.stream_results`选项与`AsyncConnection.execute()`方法时已经引发异常一样。

    另外，为了改善与状态敏感的数据库驱动程序（如 asyncmy）的稳定性，当出现此错误条件时，游标现在会被关闭；以前在 asyncmy 方言中，连接会进入无效状态，服务器端结果仍然未消耗。

    参考：[#7667](https://www.sqlalchemy.org/trac/ticket/7667)

### postgresql

+   **[postgresql] [usecase]**

    添加了对 PostgreSQL `NOT VALID`短语的编译器支持，用于为`CheckConstraint`、`ForeignKeyConstraint`和`ForeignKey`模式构造渲染 DDL。拉取请求由 Gilbert Gilb 提供。

    另请参阅

    PostgreSQL 约束选项

    参考：[#7600](https://www.sqlalchemy.org/trac/ticket/7600)

### mysql

+   **[mysql] [bug] [regression]**

    修复了由[#7518](https://www.sqlalchemy.org/trac/ticket/7518)引起的回归，其中将语法“SHOW VARIABLES”更改为“SELECT @@”破坏了与 MySQL 5.6 之前的旧版本的兼容性，包括早期的 5.0 版本。虽然这些是非常旧的 MySQL 版本，但没有计划更改兼容性，因此已恢复了版本特定的逻辑，以便对 MySQL 服务器版本 < 5.6 进行“SHOW VARIABLES”回退。

    参考：[#7518](https://www.sqlalchemy.org/trac/ticket/7518)

### mariadb

+   **[mariadb] [bug] [regression]**

    mariadbconnector 方言中的回归问题已修复，从 mariadb connector 1.0.10 开始，DBAPI 不再预先缓冲 cursor.lastrowid，导致使用 ORM 插入对象时出错，以及导致`CursorResult.inserted_primary_key`属性不可用。方言现在为适用情况主动获取此值。

    参考：[#7738](https://www.sqlalchemy.org/trac/ticket/7738)

### sqlite

+   **[sqlite] [usecase]**

    增加了对反映 SQLite 内联唯一约束的支持，其中列名的格式为 SQLite 的“转义引号”`[]`或`` ` ``，这些引号在生成列名时被数据库丢弃。

    参考：[#7736](https://www.sqlalchemy.org/trac/ticket/7736)

+   **[sqlite] [bug]**

    修复了 SQLite 唯一约束反射无法检测到列内联的唯一约束的问题，其中列名中的下划线会导致失败。

    参考：[#7736](https://www.sqlalchemy.org/trac/ticket/7736)

### oracle

+   **[oracle] [bug]**

    修复了 Oracle 方言中的问题，当使用作为绑定参数写入时需要引号的列名，例如`"_id"`，由于绑定参数重写丢失了这个值，导致 Python 生成的默认值无法正确跟踪，从而引发了 Oracle 错误。

    参考：[#7676](https://www.sqlalchemy.org/trac/ticket/7676)

+   **[oracle] [bug] [regression]**

    增加了对从 cx_Oracle 异常对象（如 `DPI-1080` 和 `DPI-1010`）解析“DPI”错误代码的支持，这两个错误代码现在都表示 cx_Oracle 8.3 中的断开连接场景。

    参考：[#7748](https://www.sqlalchemy.org/trac/ticket/7748)

### tests

+   **[tests] [bug]**

    改进了测试套件与 pytest 的集成，使得如果手动启用了“warnings”插件，则不会干扰测试套件，这样第三方可以启用 warnings 插件或使用 `-W` 参数，并且 SQLAlchemy 的测试套件将继续通过。此外，现代化了“pytest-xdist”插件的检测，以便可以在全局禁用插件而不会破坏测试套件，即使 xdist 仍然安装着。现在将将将将将将促将的警告过滤器局限于 SQLAlchemy 特定的警告，或者在通用的 Python 警告中，用于一般的 Python 警告，以便来自 pytest 插件的非 SQLAlchemy 警告也不会影响测试套件。

    参考：[#7599](https://www.sqlalchemy.org/trac/ticket/7599)

+   **[tests] [bug]**

    更正了默认 pytest 配置，以修复测试套件配置警告的问题，并且在特定情况下，还会尝试将示例套件作为测试加载，具体情况是当 SQLAlchemy 检出位于具有名为“test”的上级目录的绝对路径中时。

    参考：[#7045](https://www.sqlalchemy.org/trac/ticket/7045)

## 1.4.31

发布日期：2022 年 1 月 20 日

### orm

+   **[orm] [bug]**

    修复了 `Session.bulk_save_objects()` 中的问题，在 `preserve_order` 参数设置为 False 时进行排序会部分地在 `Mapper` 对象上进行排序，这在 Python 3.11 中被拒绝。

    参考：[#7591](https://www.sqlalchemy.org/trac/ticket/7591)

### postgresql

+   **[postgresql] [bug] [regression]**

    修复了 [#7148](https://www.sqlalchemy.org/trac/ticket/7148) 中的变更，以修复 PostgreSQL 中 ENUM 处理的问题，该变更破坏了空 ENUM 数组的用例，导致包含空数组的行在获取结果时无法正确处理。

    参考：[#7590](https://www.sqlalchemy.org/trac/ticket/7590)

### mysql

+   **[mysql] [bug] [regression]**

    修复了 [#7567](https://www.sqlalchemy.org/trac/ticket/7567) 引起的 asyncmy 方言的回归，其中移除了 PyMySQL 依赖导致二进制列出现问题，因为 asyncmy 方言未正确包含在 CI 测试中。

    参考：[#7593](https://www.sqlalchemy.org/trac/ticket/7593)

### mssql

+   **[mssql]**

    在 MSSQL 中使用 `VARBINARY(max)` 时添加了对 `FILESTREAM` 的支持。

    另见

    `VARBINARY.filestream`

    参考：[#7243](https://www.sqlalchemy.org/trac/ticket/7243)

### orm

+   **[orm] [bug]**

    修复了 `Session.bulk_save_objects()` 中的问题，在 `preserve_order` 参数设置为 False 时进行排序会部分地在 `Mapper` 对象上进行排序，这在 Python 3.11 中被拒绝。

    参考：[#7591](https://www.sqlalchemy.org/trac/ticket/7591)

### postgresql

+   **[postgresql] [bug] [regression]**

    修复了在[#7148](https://www.sqlalchemy.org/trac/ticket/7148)中修复 PostgreSQL 中 ENUM 处理的变化导致空 ENUM 数组用例的回归，阻止包含空数组的行在获取结果时被正确处理。

    参考：[#7590](https://www.sqlalchemy.org/trac/ticket/7590)

### mysql

+   **[mysql] [bug] [regression]**

    修复了由[#7567](https://www.sqlalchemy.org/trac/ticket/7567)引起的 asyncmy 方言中的回归，其中删除 PyMySQL 依赖项导致二进制列出现问题，因为 asyncmy 方言未正确包含在 CI 测试中。

    参考：[#7593](https://www.sqlalchemy.org/trac/ticket/7593)

### mssql

+   **[mssql]**

    在 MSSQL 中使用`VARBINARY(max)`时添加了对`FILESTREAM`的支持。

    另请参阅

    `VARBINARY.filestream`

    参考：[#7243](https://www.sqlalchemy.org/trac/ticket/7243)

## 1.4.30

发布日期：2022 年 1 月 19 日

### orm

+   **[orm] [bug]**

    修复了在深层多级继承中 joined-inheritance 加载附加属性功能中的问题，在包含没有列的中间表的情况下，这些中间表不会被包含在连接的表中，而是将这些表链接到它们的主键标识符。虽然这样做没问题，但在 1.4 版本中开始产生笛卡尔积编译器警告。逻辑已更改，以便无论如何都包括这些中间表。虽然这样做会在查询中包含不是技术上必要的附加表，但这仅适用于具有没有非主键列的中间表的深层 3 级以上继承的极不寻常的情况，因此预计性能影响将可以忽略不计。

    参考：[#7507](https://www.sqlalchemy.org/trac/ticket/7507)

+   **[orm] [bug]**

    修复了多次调用`registry.map_imperatively()`会产生意外错误的问题，而不是提示目标类已经映射的信息错误。这种行为与`mapper()`函数不同，后者已经报告了相关信息。

    参考：[#7579](https://www.sqlalchemy.org/trac/ticket/7579)

+   **[orm] [bug] [asyncio]**

    向`AsyncSession.invalidate()`方法添加到`AsyncSession`类中。

    参考：[#7524](https://www.sqlalchemy.org/trac/ticket/7524)

+   **[orm] [bug] [regression]**

    修复了在 1.4.23 中出现的一个回归，该回归可能导致在某些情况下错误处理加载器选项，特别是在使用连接表继承与`polymorphic_load="selectin"`选项以及关系惰性加载时，可能导致`TypeError`。

    参考：[#7557](https://www.sqlalchemy.org/trac/ticket/7557)

+   **[orm] [bug] [regression]**

    修复了一个 ORM 回归问题，即针对现有的`aliased()`构造调用`aliased()`函数会导致如果现有构造针对一个固定表，则无法生成正确的 SQL。修复使得如果原始的`aliased()`构造仅针对将被替换的表，则可以忽略原始构造。它还允许在没有可选择参数的情况下构造一个`aliased()`，该构造是针对子查询的`aliased()`，以创建该子查询的别名（即更改其名称）。

    当外部`aliased()`对象针对一个反过来引用内部`aliased()`对象的子查询时，`aliased()`的嵌套行为仍然保持不变。这是一个相对较新的 1.4 功能，它有助于适应以前由已弃用的`Query.from_self()`方法提供的用例。

    参考：[#7576](https://www.sqlalchemy.org/trac/ticket/7576)

+   **[orm] [bug]**

    修复了一个问题，在 ORM 上下文中使用`Select.correlate_except()`方法时，当传递`None`值或没有参数时，不会关联任何元素，而不是导致所有 FROM 元素都被视为“关联”，这与仅使用 Core 构造时发生的情况相同。

    参考：[#7514](https://www.sqlalchemy.org/trac/ticket/7514)

+   **[orm] [bug] [regression]**

    修复了从 1.3 版本开始出现的问题，即“subqueryload”加载策略在针对使用`Query.from_statement()`或`Select.from_statement()`的查询时会导致堆栈跟踪失败的回归。由于 subqueryload 需要修改原始语句，因此与“from_statement”用例不兼容，特别是针对`text()`构造的语句。现在的行为与 1.3 版本及之前的版本相同，即加载策略会静默降级，不会用于这种语句，通常会回退到使用 lazyload 策略。

    参考：[#7505](https://www.sqlalchemy.org/trac/ticket/7505)

### sql

+   **[sql] [错误] [postgresql]**

    向系统添加了一个额外的规则，用于从 Python 文字到`TypeEngine`实现的类型，以对类型进行第二级调整，以便 Python 日期时间无论是否具有 tzinfo 都可以在返回的`DateTime`对象上设置`timezone=True`参数，以及`Time`。这有助于某些类型敏感的 PostgreSQL 方言的往返场景，如 asyncpg、psycopg3（仅限 2.0）。

    参考：[#7537](https://www.sqlalchemy.org/trac/ticket/7537)

+   **[sql] [错误]**

    当将方法对象传递给 SQL 构造时，添加了一个信息性错误消息。以前，当传递这样一个可调用对象时，这是一种常见的处理方法错误，当处理方法链接的 SQL 构造时，它们被解释为在编译时调用的“lambda SQL”目标，这将导致静默失败。由于此功能不打算与方法一起使用，因此现在拒绝使用方法对象。

    参考：[#7032](https://www.sqlalchemy.org/trac/ticket/7032)

### mypy

+   **[mypy] [错误]**

    修复了在运行 id 守护程序模式时由于内部 mypy `Var`实例上缺少属性而导致的 Mypy 崩溃。

    参考：[#7321](https://www.sqlalchemy.org/trac/ticket/7321)

### asyncio

+   **[asyncio] [用例]**

    向 DBAPI 连接接口添加了新方法 `AdaptedConnection.run_async()`，该接口由 asyncio 驱动程序使用，允许在不能使用 `await` 关键字的同步样式函数中直接调用底层“驱动程序”连接的方法，例如在 SQLAlchemy 事件处理程序函数中。该方法类似于 `AsyncConnection.run_sync()` 方法，它将异步式调用转换为同步式调用。该方法对于诸如连接池连接处理程序之类的需要在驱动程序连接首次创建时调用可等待方法的情况非常有用。

    亦参见

    在连接池和其他事件中使用仅可等待的驱动程序方法

    参考：[#7580](https://www.sqlalchemy.org/trac/ticket/7580)

### postgresql

+   **[postgresql] [用例]**

    添加了对 `UUID` 数据类型的字符串渲染，因此使用“literal_binds”来对这种类型进行字符串化的语句将为 PostgreSQL 后端渲染适当的字符串值。此拉取请求由 José Duarte 提供。

    参考：[#7561](https://www.sqlalchemy.org/trac/ticket/7561)

+   **[postgresql] [错误] [asyncpg]**

    对于时间带时区的 asyncpg 处理进行了改进，之前没有完全实现。

    参考：[#7537](https://www.sqlalchemy.org/trac/ticket/7537)

+   **[postgresql] [错误] [mssql] [反射]**

    修复了将覆盖索引反射为将 `include_columns` 报告为反射索引字典中的 `dialect_options` 条目的一部分，从而使得从反射->创建的往返完整。包含的列仍然在 `include_columns` 键下出现以保持向后兼容性。

    参考：[#7382](https://www.sqlalchemy.org/trac/ticket/7382)

+   **[postgresql] [错误]**

    修复了需要转义字符的枚举值数组的处理方式。

    参考：[#7418](https://www.sqlalchemy.org/trac/ticket/7418)

### mysql

+   **[mysql] [变更]**

    在 MySQL 和 MariaDB 方言初始化中，使用等效的 `SELECT @@variable` 替换了 `SHOW VARIABLES LIKE` 语句。这应该避免由 `SHOW VARIABLES` 引起的互斥体争用，从而改善初始化性能。

    参考：[#7518](https://www.sqlalchemy.org/trac/ticket/7518)

+   **[mysql] [错误]**

    从 asyncmy 方言中删除了对 PyMySQL 的不必要依赖。拉取请求由 long2ice 提供。

    参考：[#7567](https://www.sqlalchemy.org/trac/ticket/7567)

### orm

+   **[orm] [错误]**

    修复了在深层多级继承中的联接继承加载附加属性功能中的问题，其中一个不包含任何列的中介表不会包含在联接的表中，而是将这些表链接到它们的主键标识符。虽然这样做没问题，但在 1.4 中开始产生笛卡尔积编译器警告。逻辑已更改，以便无论如何都包括这些中介表。虽然这样做会在查询中包含不是技术上必要的附加表，但这仅适用于具有没有非主键列的中介表的深层 3 级以上继承的极不寻常的情况，因此预计性能影响将可以忽略不计。

    参考：[#7507](https://www.sqlalchemy.org/trac/ticket/7507)

+   **[orm] [bug]**

    修复了调用`registry.map_imperatively()`多次针对同一类时会产生意外错误的问题，而不是提示目标类已经映射的信息性错误。这种行为与`mapper()`函数不同，后者已经报告了信息性消息。

    参考：[#7579](https://www.sqlalchemy.org/trac/ticket/7579)

+   **[orm] [bug] [asyncio]**

    向`AsyncSession`类添加了缺失的方法`AsyncSession.invalidate()`。

    参考：[#7524](https://www.sqlalchemy.org/trac/ticket/7524)

+   **[orm] [bug] [regression]**

    修复了在 1.4.23 中出现的导致某些情况下无法正确处理加载器选项的回归问题，特别是在使用联接表继承与`polymorphic_load="selectin"`选项以及关系惰性加载时，可能导致`TypeError`。

    参考：[#7557](https://www.sqlalchemy.org/trac/ticket/7557)

+   **[orm] [bug] [regression]**

    修复了 ORM 回归，其中针对现有`aliased()`构造调用`aliased()`函数将无法生成正确的 SQL，如果现有构造针对的是一个固定表。修复允许忽略原始`aliased()`构造，如果它只针对一个现在被替换的表。它还允许在没有可选择参数的情况下构造一个针对子查询的`aliased()`时，创建该子查询的别名（即更改其名称）。

    `aliased()`的嵌套行为仍然适用于外部`aliased()`对象针对一个子查询的情况，该子查询反过来引用内部`aliased()`对象。这是一个相对较新的 1.4 功能，有助于适应以前由已弃用的`Query.from_self()`方法提供服务的用例。

    参考：[#7576](https://www.sqlalchemy.org/trac/ticket/7576)

+   **[orm] [bug]**

    修复了当`Select.correlate_except()`方法传递`None`值或没有参数时，在 ORM 上下文中使用时（即将 ORM 实体作为 FROM 子句传递时），不会关联任何元素的问题，而不是像在仅使用 Core 构造时发生的那样，导致所有 FROM 元素被视为“相关”。

    参考：[#7514](https://www.sqlalchemy.org/trac/ticket/7514)

+   **[orm] [bug] [regression]**

    修复了从 1.3 版本中的一个回归，即“subqueryload”加载策略如果用于针对使用 `Query.from_statement()` 或 `Select.from_statement()` 进行查询的查询，则会失败并显示堆栈跟踪。由于 subqueryload 需要修改原始语句，所以它与“from_statement”用例不兼容，特别是对于针对 `text()` 构造的语句。现在的行为等同于 1.3 版本和以前的版本，即加载策略会静默地降级，不会用于这种语句，通常会回退到使用 lazyload 策略。

    参考：[#7505](https://www.sqlalchemy.org/trac/ticket/7505)

### sql

+   **[sql] [bug] [postgresql]**

    向确定从 Python 字面量到 `TypeEngine` 实现的系统添加了一个额外的规则，以对类型进行第二级调整，以便 Python 的 datetime 带或不带 tzinfo 可以在返回的 `DateTime` 对象上设置 `timezone=True` 参数，以及 `Time`。这有助于一些类型敏感的 PostgreSQL 方言的往返场景，比如 asyncpg、psycopg3 (2.0 only)。

    参考：[#7537](https://www.sqlalchemy.org/trac/ticket/7537)

+   **[sql] [bug]**

    当一个方法对象传递给 SQL 构造时，添加了一个信息性的错误消息。以前，当传递这样一个可调用对象时，通常是在处理方法链式 SQL 构造时出现的常见打字错误，它们被解释为在编译时调用的“lambda SQL”目标，这会导致静默失败。由于这个特性不打算与方法一起使用，所以现在拒绝了方法对象。

    参考：[#7032](https://www.sqlalchemy.org/trac/ticket/7032)

### mypy

+   **[mypy] [bug]**

    修复了运行 id 守护进程模式时 Mypy 崩溃的问题，原因是内部 mypy `Var` 实例缺少属性。

    参考：[#7321](https://www.sqlalchemy.org/trac/ticket/7321)

### asyncio

+   **[asyncio] [usecase]**

    添加了新方法 `AdaptedConnection.run_async()` 到由 asyncio 驱动程序使用的 DBAPI 连接接口，该接口允许在无法使用 `await` 关键字的同步样式函数中直接对底层“驱动程序”连接调用方法，例如在 SQLAlchemy 事件处理程序函数内。 该方法类似于 `AsyncConnection.run_sync()` 方法，该方法将异步样式调用转换为同步样式。 该方法对于诸如连接池连接处理程序之类的需要在驱动程序连接首次创建时调用可等待方法的情况非常有用。

    另请参阅

    在连接池和其他事件中使用仅可等待的驱动程序方法

    参考：[#7580](https://www.sqlalchemy.org/trac/ticket/7580)

### postgresql

+   **[postgresql] [usecase]**

    向 `UUID` 数据类型添加了字符串渲染，以便在使用此类型的“literal_binds”语句的字符串化时，为 PostgreSQL 后端呈现适当的字符串值。 感谢 José Duarte 提交的拉取请求。

    参考：[#7561](https://www.sqlalchemy.org/trac/ticket/7561)

+   **[postgresql] [bug] [asyncpg]**

    改进了 asyncpg 处理 TIME WITH TIMEZONE 的支持，该支持尚未完全实现。

    参考：[#7537](https://www.sqlalchemy.org/trac/ticket/7537)

+   **[postgresql] [bug] [mssql] [reflection]**

    修复了覆盖索引的反射，以将 `include_columns` 作为反射索引字典中的 `dialect_options` 条目的一部分报告，从而使从反射->创建的往返完整。 包含的列继续出现在 `include_columns` 键下以确保向后兼容性。

    参考：[#7382](https://www.sqlalchemy.org/trac/ticket/7382)

+   **[postgresql] [bug]**

    修复了需要转义字符的枚举值数组的处理。

    参考：[#7418](https://www.sqlalchemy.org/trac/ticket/7418)

### mysql

+   **[mysql] [change]**

    在 MySQL 和 MariaDB 方言初始化中，用等效的 `SELECT @@variable` 替换了 `SHOW VARIABLES LIKE` 语句。 这应该避免由 `SHOW VARIABLES` 引起的互斥体争用，从而提高了初始化性能。

    参考：[#7518](https://www.sqlalchemy.org/trac/ticket/7518)

+   **[mysql] [bug]**

    从 asyncmy 方言中删除了对 PyMySQL 的不必要依赖。 感谢 long2ice 提交的拉取请求。

    参考：[#7567](https://www.sqlalchemy.org/trac/ticket/7567)

## 1.4.29

发布日期：2021 年 12 月 22 日

### orm

+   **[orm] [usecase]**

    添加了`Session.get.execution_options`参数，此参数之前在`Session.get()`方法中缺失。

    参考：[#7410](https://www.sqlalchemy.org/trac/ticket/7410)

+   **[orm] [bug]**

    修复了新的“加载条件”方法`PropComparator.and_()`中的问题，在与`selectinload()`这样的加载策略一起使用时，针对子查询对象的`.c.`集合的成员列，其中子查询将动态添加到语句的 FROM 子句中，将受到 SQL 语句缓存中子查询内部参数值过时的影响，因为加载策略在执行时用于替换参数的过程未能适应以这种形式接收到的子查询。

    参考：[#7489](https://www.sqlalchemy.org/trac/ticket/7489)

+   **[orm] [bug]**

    修复了在使用 ORM 语句编译时可能发生的递归溢出问题，当使用`with_loader_criteria()`特性或者在加载策略中使用`PropComparator.and_()`方法与引用被同一实体更改的条件选项或由加载策略加载的子查询一起使用时。已添加检查以适应此场景中的递归方式遇到相同的加载器条件选项。

    参考：[#7491](https://www.sqlalchemy.org/trac/ticket/7491)

+   **[orm] [bug] [mypy]**

    修复了`__class_getitem__()`方法在通过`as_declarative()`生成的声明基类中的问题，对于在类层次结构中使用`Generic[T]`样式类型声明的情况，会导致无法访问的类属性，例如`__table__`。这是继续自[#7368](https://www.sqlalchemy.org/trac/ticket/7368)中基本添加的`__class_getitem__()`。拉取请求由 Kai Mueller 提供。

    参考：[#7368](https://www.sqlalchemy.org/trac/ticket/7368), [#7462](https://www.sqlalchemy.org/trac/ticket/7462)

+   **[orm] [bug] [regression]**

    修复了缓存相关问题，其中形式为`lazyload(aliased(A).bs).joinedload(B.cs)`的加载器选项在查询被缓存后无法导致联接加载被调用，这是由于在用`aliased()`使用的引导实体的对象路径/选项与对象加载时的选项不匹配。

    参考：[#7447](https://www.sqlalchemy.org/trac/ticket/7447)

### engine

+   **[engine] [bug]**

    更正了当尝试向不可变的`Row`类的属性写入时引发的`AttributeError`的错误消息，先前的消息声称列不存在，这是误导性的。

    参考：[#7432](https://www.sqlalchemy.org/trac/ticket/7432)

+   **[engine] [bug] [regression]**

    修复了`make_url()`函数中的回归，在解析 URL 字符串时，如果使用了 Python 2 的`u''`字符串，查询字符串解析将进入递归溢出。

    参考：[#7446](https://www.sqlalchemy.org/trac/ticket/7446)

### mypy

+   **[mypy] [bug]**

    修复了 mypy 回归，其中 mypy 0.930 版本的发布为“命名类型”的格式增加了额外的内部检查，要求它们是完全限定和可定位的。这破坏了用于 SQLAlchemy 的 mypy 插件，引发了断言错误，因为以前对于`__builtins__`等符号的使用没有引发任何断言。

    参考：[#7496](https://www.sqlalchemy.org/trac/ticket/7496)

### asyncio

+   **[asyncio] [usecase]**

    添加了`async_engine_config()`函数，用于根据配置字典创建异步引擎。否则，其行为与`engine_from_config()`相同。

    参考：[#7301](https://www.sqlalchemy.org/trac/ticket/7301)

### mariadb

+   **[mariadb] [bug]**

    更正了对于`mariadbconnector`方言的“is_disconnect”检查所检查的错误类，这导致由于常见的 MySQL/MariaDB 错误代码（如 2006）导致的断开连接失败；目前 DBAPI 显然使用`mariadb.InterfaceError`异常类表示断开连接的错误，例如错误代码 2006，已将其添加到检查类列表中。

    参考：[#7457](https://www.sqlalchemy.org/trac/ticket/7457)

### tests

+   **[tests] [bug] [regression]**

    修复了测试套件中的一个回归，其中测试调用了`CompareAndCopyTest::test_all_present`，在某些平台上由于检测到额外的测试工件而失败。感谢 Nils Philippsen 提交的拉取请求。

    参考：[#7450](https://www.sqlalchemy.org/trac/ticket/7450)

### orm

+   **[orm] [usecase]**

    添加了`Session.get.execution_options`参数，此前该参数缺失于`Session.get()`方法中。

    参考：[#7410](https://www.sqlalchemy.org/trac/ticket/7410)

+   **[orm] [bug]**

    修复了新的“加载器条件”方法`PropComparator.and_()`中的问题，其中使用类似`selectinload()`的加载策略针对作为子查询对象的`.c.`集合的成员的列使用时，子查询将动态添加到语句的 FROM 子句中，将在 SQL 语句缓存中的子查询内使用过时的参数值，因为加载器策略用于在执行时替换参数的过程将无法适应以这种形式接收的子查询。

    参考文献：[#7489](https://www.sqlalchemy.org/trac/ticket/7489)

+   **[ORM] [错误]**

    修复了使用`with_loader_criteria()`功能或在加载器策略中使用`PropComparator.and_()`方法时，当子查询引用被相同的实体修改或由加载器策略加载时，可能在 ORM 语句编译中发生递归溢出的问题。 已添加了检查是否以递归方式遇到相同加载器条件选项以适应此场景。

    参考文献：[#7491](https://www.sqlalchemy.org/trac/ticket/7491)

+   **[ORM] [错误] [mypy]**

    修复了通过`as_declarative()`生成的声明基类的`__class_getitem__()`方法可能导致无法访问的类属性，例如`__table__`，对于在类层次结构中使用`Generic[T]`样式类型声明的情况。 这是从[#7368](https://www.sqlalchemy.org/trac/ticket/7368)的基本添加中继续的`__class_getitem__()`。 拉取请求由 Kai Mueller 提供。

    参考文献：[#7368](https://www.sqlalchemy.org/trac/ticket/7368), [#7462](https://www.sqlalchemy.org/trac/ticket/7462)

+   **[ORM] [错误] [回归]**

    修复了与缓存相关的问题，其中使用形式为`lazyload(aliased(A).bs).joinedload(B.cs)`的加载器选项将无法导致连接加载在查询被缓存后的运行中被调用，由于应用于使用`aliased()`的主体实体的选项/对象路径与为查询加载的对象不匹配。

    参考文献：[#7447](https://www.sqlalchemy.org/trac/ticket/7447)

### 引擎

+   **[引擎] [错误]**

    修正了在尝试写入不可变属性`Row`类时引发的`AttributeError`错误消息，以前的消息声称列不存在，这是误导性的。

    参考：[#7432](https://www.sqlalchemy.org/trac/ticket/7432)

+   **[engine] [bug] [regression]**

    修复了 `make_url()` 函数中的回归问题，该函数用于解析 URL 字符串，在使用 Python 2 的 `u''` 字符串时，查询字符串解析会导致递归溢出。

    参考：[#7446](https://www.sqlalchemy.org/trac/ticket/7446)

### mypy

+   **[mypy] [bug]**

    修复了 mypy 0.930 版本发布后的一个回归问题，该版本增加了对“命名类型”格式的额外内部检查，要求它们必须是完全合格且可定位的。这导致了 SQLAlchemy 的 mypy 插件出现问题，引发了一个断言错误，因为之前使用了诸如 `__builtins__` 等无法定位或未合格的符号，而这些符号之前并未引发任何断言。

    参考：[#7496](https://www.sqlalchemy.org/trac/ticket/7496)

### asyncio

+   **[asyncio] [usecase]**

    添加了 `async_engine_config()` 函数，用于从配置字典创建一个异步引擎。否则，此函数与 `engine_from_config()` 的行为相同。

    参考：[#7301](https://www.sqlalchemy.org/trac/ticket/7301)

### mariadb

+   **[mariadb] [bug]**

    修正了对 `mariadbconnector` 方言进行“is_disconnect”检查时所检查的错误类，此检查在由于常见的 MySQL/MariaDB 错误代码（例如 2006）而导致的断开连接时失败；目前 DBAPI 显然使用 `mariadb.InterfaceError` 异常类来表示诸如错误代码 2006 等断开连接的错误，这已经添加到了检查类列表中。

    参考：[#7457](https://www.sqlalchemy.org/trac/ticket/7457)

### tests

+   **[tests] [bug] [regression]**

    修复了测试套件中的一个回归问题，在测试调用 `CompareAndCopyTest::test_all_present` 时，由于检测到了额外的测试工件，导致在某些平台上测试失败。拉取请求由 Nils Philippsen 提供。

    参考：[#7450](https://www.sqlalchemy.org/trac/ticket/7450)

## 1.4.28

发布日期：2021 年 12 月 9 日

### platform

+   **[platform] [bug]**

    Python 3.10 已经弃用了“distutils”，而是推荐显式使用“setuptools” [**PEP 632**](https://peps.python.org/pep-0632/)；SQLAlchemy 的 setup.py 已相应地替换了导入。但是，由于 setuptools 自身直到 2021 年 11 月的版本 59.0.1 才最近添加了 pep-632 中提到的替代符号，因此 `setup.py` 仍然具有对 distutils 的后备导入，因为 SQLAlchemy 1.4 目前不具有对 setuptools 的硬版本要求。预计 SQLAlchemy 2.0 将使用完整的 [**PEP 517**](https://peps.python.org/pep-0517/) 安装布局，其将从一开始就指定适当的 setuptools 版本。

    参考：[#7311](https://www.sqlalchemy.org/trac/ticket/7311)

### orm

+   **[orm] [bug] [ext]**

    修复了在`relationship()`上使用`PropComparator.any()`方法时内部克隆的问题，在相关类也使用 ORM 多态加载的情况下，如果在`any()`操作的条件中使用了相关的多态类上的混合属性，则会失败。

    参考：[#7425](https://www.sqlalchemy.org/trac/ticket/7425)

+   **[orm] [bug] [mypy]**

    修复了`as_declarative()`装饰器和类似函数生成声明基类时不会从给定超类复制`__class_getitem__()`方法的问题，这阻止了与`Base`类结合使用 pep-484 泛型的使用。感谢 Kai Mueller 的拉取请求。

    参考：[#7368](https://www.sqlalchemy.org/trac/ticket/7368)

+   **[orm] [bug] [regression]**

    修复了 ORM 回归问题，即“急切加载器在未过期时运行”的新行为，添加在[#1763](https://www.sqlalchemy.org/trac/ticket/1763)中，会导致加载器选项错误被不当地引发，对于仅使用一个`Query`或`Select`来加载多种类型的实体，以及仅适用于其中一种实体的加载器选项，如`joinedload()`，然后对象将从过期中刷新，加载器选项将尝试应用于不匹配的对象类型，然后引发异常。现在，对于这种不匹配情况的检查现在绕过了引发错误。

    参考：[#7318](https://www.sqlalchemy.org/trac/ticket/7318)

+   **[orm] [bug]**

    用户定义的 ORM 选项，例如在 dogpile.caching 示例中展示的那些子类`UserDefinedOption`，根据定义在每个语句执行时处理，并且不需要考虑作为语句缓存键的一部分。基本`ExecutableOption`类的缓存已被修改，以便它不再直接是`HasCacheKey`的子类，因此用户定义的选项对象的存在将不会导致禁用语句缓存的不良副作用。现在，只有 ORM 特定的加载器和条件选项，这些选项都是 SQLAlchemy 内部的，才参与缓存系统。

    参考：[#7394](https://www.sqlalchemy.org/trac/ticket/7394)

+   **[orm] [bug]**

    修复了使用`synonym()`和可能其他类型的“代理”属性的映射在某些情况下无法成功为其 SQL 语句生成缓存键，导致这些语句性能下降的问题。

    参考：[#7394](https://www.sqlalchemy.org/trac/ticket/7394)

+   **[orm] [错误]**

    修复了使用`relationship()`映射的列表如果被就地添加到自身，即使用`+=`运算符，以及如果`.extend()`给定相同的列表，会进入无限循环的问题。

    参考：[#7389](https://www.sqlalchemy.org/trac/ticket/7389)

+   **[orm] [错误]**

    修复了在`Session.commit()`方法中关闭连接时发生异常的问题，当使用`Session.begin()`的上下文管理器时，会尝试回滚，但由于`Session`处于事务提交和连接返回到池之间的状态，无法回滚，引发异常“此会话事务处于已提交状态”。这种异常主要在 asyncio 上下文中可能会引发 CancelledError。

    参考：[#7388](https://www.sqlalchemy.org/trac/ticket/7388)

+   **[orm] [已弃用]**

    废弃了一个未记录的加载器选项语法`".*"`，似乎与传递单个星号没有区别，并且如果使用将发出弃用警告。这种语法可能曾经有过用途，但目前没有必要。

    参考：[#4390](https://www.sqlalchemy.org/trac/ticket/4390)

### engine

+   **[engine] [用例]**

    为`URL`类添加了对`copy()`和`deepcopy()`的支持。感谢 Tom Ritchford 的拉取请求。

    参考：[#7400](https://www.sqlalchemy.org/trac/ticket/7400)

### sql

+   **[sql] [用例]**

    "Compound select" 方法如 `Select.union()`、`Select.intersect_all()` 等现在接受 `*other` 作为参数，而不是 `other`，以允许一次性将多个额外的 SELECT 与父语句合并。特别是，对 `CTE.union()` 和 `CTE.union_all()` 的更改现在允许使用 `CTE` 构造创建所谓的“非线性 CTE”，而以前没有办法在正确地调用递归方式的同时将多个 CTE 子元素联合到一起。感谢 Eric Masseran 的拉取请求。

    参考：[#7259](https://www.sqlalchemy.org/trac/ticket/7259)

+   **[sql] [用例]**

    支持在 `Exists.where()` 方法中使用多个子句元素，将 API 与普通 `select()` 构造方法统一起来。

    参考：[#7386](https://www.sqlalchemy.org/trac/ticket/7386)

+   **[sql] [错误] [回归]**

    扩展了 `TypeDecorator.cache_ok` 属性和相应的警告消息，如果未定义此标志，则首次在 [#6436](https://www.sqlalchemy.org/trac/ticket/6436) 中为 `TypeDecorator` 设定的行为也适用于 `UserDefinedType`，通过将标志和相关缓存逻辑泛化到这两种类型的新通用基类 `ExternalType` 来创建 `UserDefinedType.cache_ok`。

    这一变更意味着任何当前的`UserDefinedType`都将导致不再对使用该数据类型的语句进行 SQL 语句缓存，并发出警告，除非该类定义了`UserDefinedType.cache_ok`标志为 True。如果数据类型无法从其参数派生出确定性的、可哈希的缓存键，则该属性可以设置为 False，这将继续保持缓存禁用但将抑制警告。特别是，目前在 SQLAlchemy-utils 等包中使用的自定义数据类型将需要实现此标志。此问题是由于一个目前无法缓存的 SQLAlchemy-utils 数据类型而观察到的。

    另请参阅

    `ExternalType.cache_ok`

    参考：[#7319](https://www.sqlalchemy.org/trac/ticket/7319)

+   **[sql] [bug]**

    自定义 SQL 元素、第三方方言、自定义或第三方数据类型在不清楚地选择 SQL 语句缓存时都将生成一致的警告，这可以通过在每种类型的类上设置适当的属性来实现。警告链接到文档部分，指示每种对象类型的适当方法以启用缓存。

    参考：[#7394](https://www.sqlalchemy.org/trac/ticket/7394)

+   **[sql] [bug]**

    修复了 SQL Core 中几个较少使用的类的缺失缓存指令，这将导致使用这些类的元素记录为 `[no key]`。  

    参考：[#7394](https://www.sqlalchemy.org/trac/ticket/7394)

### mypy

+   **[mypy] [bug]**

    修复了当使用 Mypy 插件针对使用了 `declared_attr` 方法的代码时，可能导致 Mypy 崩溃的问题，该插件会尝试将这些方法解释为映射属性，然后将它们后续地处理错误。作为这一变更的一部分，装饰的函数仍然被插件转换为通用的赋值语句（例如 `__mapper_args__: Any`），以便参数签名可以继续以与对任何其他 `@classmethod` 相同的方式进行注释，而不会让 Mypy 抱怨对于一个不明确是 `@classmethod` 的方法的错误参数类型。

    参考：[#7321](https://www.sqlalchemy.org/trac/ticket/7321)

### postgresql

+   **[postgresql] [bug]**

    修复了`hstore`和`array`构造缺少缓存指令的问题，这会导致这些元素被记录为`[no key]`。

    参考：[#7394](https://www.sqlalchemy.org/trac/ticket/7394)

### tests

+   **[tests] [bug]**

    实现了测试套件在 Pytest 7 下正确运行的支持。以前，只有 Pytest 6.x 支持 Python 3，然而版本未在 tox.ini 中固定上限。在 tox.ini 中未将 Pytest 固定为低于版本 8，以便在当前代码库中发布的 SQLAlchemy 版本在不更改环境的情况下可以在 tox 下进行测试。非常感谢 Pytest 开发人员在解决此问题时提供的帮助。

### platform

+   **[platform] [bug]**

    Python 3.10 已将“distutils”弃用，而推荐显式使用“setuptools”[**PEP 632**](https://peps.python.org/pep-0632/)；SQLAlchemy 的 setup.py 已相应替换了导入。然而，由于 setuptools 自身直到 2021 年 11 月的 59.0.1 版本才添加了 pep-632 中提到的替代符号，`setup.py` 仍具有对 distutils 的回退导入，因为此时 SQLAlchemy 1.4 没有对 setuptools 版本有硬性要求。预计 SQLAlchemy 2.0 将使用完整的[**PEP 517**](https://peps.python.org/pep-0517/)安装布局，这将提前指示适当的 setuptools 版本。

    参考：[#7311](https://www.sqlalchemy.org/trac/ticket/7311)

### orm

+   **[orm] [bug] [ext]**

    修复了在内部克隆中使用`PropComparator.any()`方法时出现的问题，在这种情况下，如果相关的类也使用了 ORM 多态加载，并且在`any()`操作的条件中使用了相关，多态类的混合属性，则会失败。

    参考：[#7425](https://www.sqlalchemy.org/trac/ticket/7425)

+   **[orm] [bug] [mypy]**

    修复了`as_declarative()`装饰器及类似函数生成声明基类时未复制给定超类的`__class_getitem__()`方法的问题，这导致了无法在`Base`类与 pep-484 泛型结合使用。感谢 Kai Mueller 提交的拉取请求。

    参考：[#7368](https://www.sqlalchemy.org/trac/ticket/7368)

+   **[orm] [bug] [regression]**

    修复了 ORM 回归，即“急加载器在未过期时运行”的新行为添加在[#1763](https://www.sqlalchemy.org/trac/ticket/1763)中，会导致加载器选项错误被不适当地引发的情况，即当单个`Query`或`Select`用于加载多种类型的实体时，以及仅适用于其中一种实体的加载器选项，如`joinedload()`，然后对象将从过期中刷新，加载器选项将尝试应用于不匹配的对象类型，然后引发异常。现在，对于这种情况，检查不再引发错误。

    参考：[#7318](https://www.sqlalchemy.org/trac/ticket/7318)

+   **[orm] [bug]**

    用户定义的 ORM 选项，例如在 dogpile.caching 示例中展示的那些子类化`UserDefinedOption`的选项，根据定义在每次语句执行时处理，不需要被视为语句的缓存键的一部分。已修改基本`ExecutableOption`类的缓存方式，使其不再直接是`HasCacheKey`的子类，因此用户定义的选项对象的存在不会导致禁用语句缓存的不良副作用。现在，只有 SQLAlchemy 内部的 ORM 特定加载器和条件选项参与缓存系统。

    参考：[#7394](https://www.sqlalchemy.org/trac/ticket/7394)

+   **[orm] [bug]**

    修复了使用`synonym()`和可能其他类型的“代理”属性的映射在某些情况下无法成功为其 SQL 语句生成缓存键，导致这些语句性能下降的问题。

    参考：[#7394](https://www.sqlalchemy.org/trac/ticket/7394)

+   **[orm] [bug]**

    修复了使用`relationship()`映射的列表如果被就地添加到自身，即使用`+=`运算符，以及如果`.extend()`给定相同的列表会进入无限循环的问题。

    参考：[#7389](https://www.sqlalchemy.org/trac/ticket/7389)

+   **[orm] [bug]**

    修复了一个问题，即如果在`Session`在`Session.commit()`方法中关闭连接时发生异常，在使用`Session.begin()`的上下文管理器时，它会尝试执行一个回滚，但这将是不可能的，因为`Session`在事务提交和然后将连接返回到池之间，在“此会话事务处于已提交状态”引发异常。这种异常主要在异步上下文中可能发生，在这种情况下可能会引发 CancelledError。

    参考：[#7388](https://www.sqlalchemy.org/trac/ticket/7388)

+   **[orm] [deprecated]**

    废弃了一个未记录的加载器选项语法`".*"`，它似乎与传递单个星号没有任何区别，并且如果使用将发出弃用警告。此语法可能是用于某些内容，但目前不需要。

    参考：[#4390](https://www.sqlalchemy.org/trac/ticket/4390)

### engine

+   **[engine] [usecase]**

    为`URL`类添加了对`copy()`和`deepcopy()`的支持。Tom Ritchford 提供的拉取请求。

    参考：[#7400](https://www.sqlalchemy.org/trac/ticket/7400)

### sql

+   **[sql] [usecase]**

    ”复合选择”方法，如`Select.union()`，`Select.intersect_all()` 等，现在接受`*other`作为参数，而不是`other`，以允许一次性将多个额外的 SELECT 与父语句合并。特别地，应用于`CTE.union()`和`CTE.union_all()`的更改现在允许使用`CTE`构造创建所谓的“非线性 CTE”，而以前没有办法在正确调用递归时一起将两个以上的 CTE 子元素联合起来。Eric Masseran 提供的拉取请求。

    参考：[#7259](https://www.sqlalchemy.org/trac/ticket/7259)

+   **[sql] [usecase]**

    在 `Exists.where()` 方法中支持多个子句元素，将 API 与正常的 `select()` 构造所呈现的 API 统一起来。

    参考：[#7386](https://www.sqlalchemy.org/trac/ticket/7386)

+   **[sql] [bug] [regression]**

    扩展了 `TypeDecorator.cache_ok` 属性和相应的警告消息，如果未定义此标志，首次为 `TypeDecorator` 建立的行为也会发生在 `UserDefinedType` 上，通过将标志和相关缓存逻辑泛化到这两种类型的新通用基类 `ExternalType` 来创建 `UserDefinedType.cache_ok`。

    此更改意味着任何当前的 `UserDefinedType` 现在将导致不再对使用该数据类型的语句进行 SQL 语句缓存，同时会发出警告，除非该类将 `UserDefinedType.cache_ok` 标志定义为 True。如果数据类型无法从其参数派生确定性、可哈希的缓存键，则该属性可以设置为 False，这将继续保持缓存禁用但会抑制警告。特别是，目前在 SQLAlchemy-utils 等包中使用的自定义数据类型将需要实现此标志。此问题是由于当前无法缓存的 SQLAlchemy-utils 数据类型而观察到的。

    另请参阅

    `ExternalType.cache_ok`

    参考：[#7319](https://www.sqlalchemy.org/trac/ticket/7319)

+   **[sql] [bug]**

    自定义 SQL 元素、第三方方言、自定义或第三方数据类型在未明确选择 SQL 语句缓存时都会生成一致的警告，这可以通过在每种类别的类上设置适当的属性来实现。警告链接到文档部分，指示每种对象的适当方法以启用缓存。

    参考：[#7394](https://www.sqlalchemy.org/trac/ticket/7394)

+   **[sql] [bug]**

    修复了 SQL Core 中一些较少使用的类缺少缓存指令的问题，这会导致使用这些元素的元素被记录为`[no key]`。

    参考：[#7394](https://www.sqlalchemy.org/trac/ticket/7394)

### mypy

+   **[mypy] [bug]**

    修复了使用 Mypy 插件针对使用`declared_attr`方法处理非映射名称（如`__mapper_args__`、`__table_args__`或其他 dunder 名称）的代码时可能导致 Mypy 崩溃的问题，因为插件会尝试将这些解释为映射属性，然后会被错误处理。作为这一变化的一部分，装饰的函数仍然被插件转换为通用的赋值语句（例如`__mapper_args__: Any`），以便参数签名可以继续以与任何其他`@classmethod`相同的方式进行注释，而 Mypy 不会抱怨对于一个不明确声明为`@classmethod`的方法的错误参数类型。

    参考：[#7321](https://www.sqlalchemy.org/trac/ticket/7321)

### postgresql

+   **[postgresql] [bug]**

    修复了对`hstore`和`array`构造缺少缓存指令的问题，这会导致这些元素被记录为`[no key]`。

    参考：[#7394](https://www.sqlalchemy.org/trac/ticket/7394)

### 测试

+   **[tests] [bug]**

    实现了使测试套件在 Pytest 7 下正确运行的支持。以前，只支持 Python 3 下的 Pytest 6.x，但是版本没有在 tox.ini 中固定上限。在 tox.ini 中没有将 Pytest 固定为低于版本 8，以便当前代码库发布的 SQLAlchemy 版本可以在 tox 下进行测试而无需更改环境。非常感谢 Pytest 开发人员对此问题的帮助。

## 1.4.27

发布日期：2021 年 11 月 11 日

### orm

+   **[orm] [bug]**

    修复了在关系到别名类功能中引入的 bug，在这里，不可能直接在第二个加载器选项中使用`aliased()`构造直接针对目标上的属性创建一个加载策略选项，例如`selectinload(A.aliased_bs).joinedload(aliased_b.cs)`，而不是在路径的前一个元素上明确使用`PropComparator.of_type()`进行修饰。此外，直接针对非别名类进行定位将被接受（不恰当），但会悄悄失败，例如`selectinload(A.aliased_bs).joinedload(B.cs)`；现在会引发引用类型不匹配的错误。

    参考：[#7224](https://www.sqlalchemy.org/trac/ticket/7224)

+   **[orm] [bug]**

    所有`Result`对象现在在硬关闭后一致地引发`ResourceClosedError`，包括在调用“单行或值”方法如`Result.first()`和`Result.scalar()`后发生的“硬关闭”。这已经是基于`CursorResult`的核心语句执行返回的最常见结果对象的行为，因此这种行为并非新鲜事。然而，此变更已扩展以正确适应使用 2.0 风格 ORM 查询时返回的 ORM“过滤”结果对象，以前这些对象会以“软关闭”方式返回空结果，或根本不会“软关闭”，并会继续从基础游标中产生结果。

    作为这一变更的一部分，还将`Result.close()`添加到基础`Result`类中，并为 ORM 使用的过滤结果实现实现了它，以便在使用`yield_per`执行选项关闭服务器端游标之前调用基础`CursorResult`上的`CursorResult.close()`方法，即在获取剩余的 ORM 结果之前关闭游标。这对于核心结果集已经可用，但此变更也使其适用于 2.0 风格的 ORM 结果。

    参考：[#7274](https://www.sqlalchemy.org/trac/ticket/7274)

+   **[orm] [bug] [regression]**

    修复了 1.4 版本中`Query.filter_by()`在从`Query.union()`、`Query.from_self()`或类似方法生成的`Query`上无法正确运行的回归问题。

    参考：[#7239](https://www.sqlalchemy.org/trac/ticket/7239)

+   **[orm] [bug]**

    修复了从连接表继承子类的延迟多态属性加载无法正确填充属性的问题，如果在初始排除该属性时使用了 `load_only()` 选项，则在排除该属性的情况下，如果 `load_only` 是从关系加载器选项下降的话。修复允许其他有效选项，例如 `defer(..., raiseload=True)` 等仍然按预期运行。

    参考：[#7304](https://www.sqlalchemy.org/trac/ticket/7304)

+   **[ORM] [错误] [回归]**

    修复了 1.4 版本的回归问题，其中 `Query.filter_by()` 在连接到使用 `PropComparator.of_type()` 指定目标实体的别名版本时，`Query.join()` 将不会正确运行的问题。该问题还适用于使用 `select()` 构造的未来风格 ORM 查询。

    参考：[#7244](https://www.sqlalchemy.org/trac/ticket/7244)

### 引擎

+   **[引擎] [错误]**

    修复了未来 `Connection` 对象中的问题，其中 `Connection.execute()` 方法将不接受非字典映射对象，例如 SQLAlchemy 自己的 `RowMapping` 或其他 `abc.collections.Mapping` 对象作为参数字典。

    参考：[#7291](https://www.sqlalchemy.org/trac/ticket/7291)

+   **[引擎] [错误] [回归]**

    修复了 `CursorResult.fetchmany()` 方法的回归问题，当结果被完全耗尽时（即当使用 `stream_results` 或 `yield_per` 时，无论是核心还是 ORM 导向的结果），将无法自动关闭服务器端游标。

    参考：[#7274](https://www.sqlalchemy.org/trac/ticket/7274)

+   **[引擎] [错误]**

    修复了未来`Engine`中的问题，即调用`Engine.begin()`并进入上下文管理器，如果实际的 BEGIN 操作由于某种原因（例如事件处理程序引发异常）而失败，则不会关闭连接；这种用例未被测试用于引擎的未来版本。请注意，“未来”上下文管理器在 Core 和 ORM 中处理`begin()`块时实际上不会运行“BEGIN”操作，直到进入上下文管理器。这与立即运行“BEGIN”操作的传统版本不同。

    参考：[#7272](https://www.sqlalchemy.org/trac/ticket/7272)

### sql

+   **[sql] [用例]**

    将`TupleType`添加到顶层`sqlalchemy`导入命名空间。

+   **[sql] [bug] [regression]**

    修复了 ORM 查询返回的行对象（现在是普通的`Row`对象）不会被`ColumnOperators.in_()`操作符解释为要分解为单独绑定参数的元组值，并且会将它们作为单个值传递给驱动程序导致失败的问题。对于已经是`TupleType`类型的表达式，对“扩展 IN”系统的更改现在会相应地处理值。在使用“tuple-in”与没有类型信息的文本语句（例如没有类型信息的文本语句）的罕见情况下，检测到实现`collections.abc.Sequence`的值的元组值，但不是`str`或`bytes`，总是在测试`Sequence`时。

    参考：[#7292](https://www.sqlalchemy.org/trac/ticket/7292)

+   **[sql] [bug]**

    修复了一个问题，即在描述按标签排序或分组时使用字符串标签进行排序或分组的功能会在将 CTE 嵌入到本身设置为标量子查询的封闭`Select`语句中时无法正确运行。

    参考：[#7269](https://www.sqlalchemy.org/trac/ticket/7269)

+   **[sql] [bug] [regression]**

    修复了`text()`构造在“case()`”构造的“whens”列表中不再被接受为目标案例的回归。该回归似乎与试图防范在此处传递的某些形式的文字值被认为是模棱两可有关；然而，目标案例无缘无故地不应被解释为像任何其他地方一样开放的 SQL 表达式，并且文字字符串或元组将转换为绑定参数，就像在其他地方一样。

    参考：[#7287](https://www.sqlalchemy.org/trac/ticket/7287)

### 模式

+   **[模式] [错误]**

    在`Table`中修复了问题，当将`Table.implicit_returning`参数与`Table.extend_existing`一起传递以增强现有的`Table`时，参数未能正确地被接受。

    参考：[#7295](https://www.sqlalchemy.org/trac/ticket/7295)

### postgresql

+   **[postgresql] [用例] [asyncpg]**

    添加了可重写的方法`PGDialect_asyncpg.setup_asyncpg_json_codec`和`PGDialect_asyncpg.setup_asyncpg_jsonb_codec`编解码器，这些编解码器在使用 asyncpg 时处理注册 JSON/JSONB 编解码器所需的任务。变化在于，方法被拆分为单独的、可重写的方法，以支持需要更改或禁用这些特定编解码器设置方式的第三方方言。 

    参考：[#7284](https://www.sqlalchemy.org/trac/ticket/7284)

+   **[postgresql] [错误] [asyncpg]**

    将 asyncpg 方言更改为将`Float`类型绑定到“float” PostgreSQL 类型，而不是“numeric”，以便接受值`float(inf)`。添加了对“inf”值的持久性的测试套件支持。

    参考：[#7283](https://www.sqlalchemy.org/trac/ticket/7283)

+   **[postgresql] [pg8000]**

    在使用 pg8000 方言的情况下改进了数组处理时与 PostgreSQL 一起使用的情况。

    参考：[#7167](https://www.sqlalchemy.org/trac/ticket/7167)

### mysql

+   **[mysql] [错误] [mariadb]**

    将保留字列表重新组织为两个单独的列表，一个用于 MySQL，另一个用于 MariaDB，以便更准确地管理这些不同的单词集合；调整了 MySQL/MariaDB 方言，以根据明确配置或服务器版本检测到的“MySQL”或“MariaDB”后端在这些列表之间进行切换。添加了当前通过 MySQL 8 和当前 MariaDB 版本添加的所有保留字，包括最近添加的关键字，如“lead”。感谢 Kevin Kirsche 提供的拉取请求。

    参考：[#7167](https://www.sqlalchemy.org/trac/ticket/7167)

+   **[mysql] [bug]**

    修复了在 MySQL 中`Insert.on_duplicate_key_update()`中使用表达式时渲染错误列名的问题。感谢 Cristian Sabaila 提供的拉取请求。

    参考：[#7281](https://www.sqlalchemy.org/trac/ticket/7281)

### mssql

+   **[mssql] [bug]**

    调整了编译器生成的“后编译”符号，包括用于“扩展 IN”以及“模式转换映射”的符号，以不直接基于带下划线的普通括号字符串，因为这与 SQL Server 的引号格式冲突，SQL Server 在引号时也使用括号，当编译器替换“后编译”和“模式转换”符号时会产生错误匹配。该问题在使用`Inspector.get_schema_names()`方法与`Connection.execution_options.schema_translate_map`功能结合使用时，以及在与内部名称“POSTCOMPILE”重叠的符号在像“扩展中”这样的特性中使用的情况下都会创建易于复现的示例。

    参考：[#7300](https://www.sqlalchemy.org/trac/ticket/7300)

### orm

+   **[orm] [bug]**

    在与别名类的关系中引入的“关系到别名类”功能中修复了一个错误，该错误是直接在第二个加载器选项中使用`aliased()`构造来针对目标上的属性创建加载器策略选项时出现的，在这种情况下，如`selectinload(A.aliased_bs).joinedload(aliased_b.cs)`，必须明确使用`PropComparator.of_type()`来对路径的前一个元素进行限定。另外，直接定位非别名类也会被接受（不合适），但会静默失败，例如`selectinload(A.aliased_bs).joinedload(B.cs)`；现在会引发一个关于类型不匹配的错误。

    参考：[#7224](https://www.sqlalchemy.org/trac/ticket/7224)

+   **[orm] [bug]**

    所有`Result`对象现在都会一致地在强制关闭后引发`ResourceClosedError`，包括在调用“单行或值”方法如`Result.first()`和`Result.scalar()`后发生的“强制关闭”。这已经是最常见的基于`CursorResult`的 Core 语句执行返回的结果对象的行为了，因此这个行为并不新。但是，该更改已经扩展到正确地适应在使用 2.0 风格 ORM 查询时返回的 ORM“过滤”结果对象，以前的行为是以“软关闭”样式返回空结果，或者根本不会“软关闭”，并会继续从底层游标中产生。

    作为这一改变的一部分，还在基本的`Result`类中添加了`Result.close()`，并且为 ORM 使用的过滤结果实现了该方法，以便在使用`yield_per`执行选项时可以调用底层的`CursorResult.close()`方法关闭服务器端游标，在剩余的 ORM 结果被获取之前。这已经对于 Core 结果集可用了，但此更改也使其对 2.0 风格的 ORM 结果可用。

    参考：[#7274](https://www.sqlalchemy.org/trac/ticket/7274)

+   **[orm] [bug] [regression]**

    修复了 1.4 中`Query.union()`、`Query.from_self()`或类似操作生成的`Query`上`Query.filter_by()`无法正确运行的回归问题。

    参考：[#7239](https://www.sqlalchemy.org/trac/ticket/7239)

+   **[orm] [bug]**

    修复了延迟多态加载从联接表继承子类的属性时，如果最初使用 `load_only()` 选项将该属性排除在外，则无法正确填充属性的问题，在 `load_only` 从关系加载器选项下降的情况下。修复使得其他有效选项（例如 `defer(..., raiseload=True)` 等）仍然如预期般工作。

    引用：[#7304](https://www.sqlalchemy.org/trac/ticket/7304)

+   **[ORM] [错误] [回归]**

    修复了 1.4 版本的回归问题，当 `Query.join()` 连接到一个使用 `PropComparator.of_type()` 来指定目标实体的别名版本的实体时，`Query.filter_by()` 不会正常工作。该问题也适用于使用 `select()` 构造的未来样式的 ORM 查询。

    引用：[#7244](https://www.sqlalchemy.org/trac/ticket/7244)

### 引擎

+   **[引擎] [错误]**

    修复了未来 `Connection` 对象中的问题，即 `Connection.execute()` 方法未能接受非字典映射对象，例如 SQLAlchemy 自身的 `RowMapping` 或其他 `abc.collections.Mapping` 对象作为参数字典。

    引用：[#7291](https://www.sqlalchemy.org/trac/ticket/7291)

+   **[引擎] [错误] [回归]**

    修复了一个回归问题，即当结果完全耗尽时，`CursorResult.fetchmany()` 方法未能自动关闭服务器端游标（即在使用 `stream_results` 或 `yield_per` 时，无论是 Core 还是 ORM 导向的结果），会出现问题。

    引用：[#7274](https://www.sqlalchemy.org/trac/ticket/7274)

+   **[引擎] [错误]**

    在将来的`Engine`中修复了一个问题，在调用 `Engine.begin()` 并进入上下文管理器时，如果实际的 BEGIN 操作由于某些原因（例如事件处理程序引发异常）而失败，则不会关闭连接；这种用例未能在将来版本的引擎中进行测试。请注意，“将来”上下文管理器在实际进入上下文管理器之前不会运行“BEGIN”操作。这与旧版本不同，旧版本会立即运行“BEGIN”操作。

    参考：[#7272](https://www.sqlalchemy.org/trac/ticket/7272)

### sql

+   **[sql] [usecase]**

    将 `TupleType` 添加到顶级 `sqlalchemy` 导入命名空间。

+   **[sql] [bug] [regression]**

    修复了一个回归，其中对 ORM 查询返回的行对象（现在是常规的 `Row` 对象）不会被`ColumnOperators.in_()`操作符解释为要分解为单个绑定参数的元组值，并且而是将它们作为单个值传递给驱动程序，导致失败。对“扩展 IN” 系统的更改现在已经适应了表达式已经是 `TupleType` 类型的情况，并且如果是，则相应地处理值。在使用“元组-在”与未类型化语句（例如没有类型信息的文本语句）的不常见情况下，检测到实现 `collections.abc.Sequence` 的值的元组值，但不是 `str` 或 `bytes` 时，像以往一样测试 `Sequence`。

    参考：[#7292](https://www.sqlalchemy.org/trac/ticket/7292)

+   **[sql] [bug]**

    修复了使用 Ordering or Grouping by a Label 描述的按标签排序或分组的功能时，如果用于 `CTE` 构造，在 CTE 被嵌入到一个设置为标量子查询的封闭 `Select` 语句内时，将无法正确运行的问题。

    参考：[#7269](https://www.sqlalchemy.org/trac/ticket/7269)

+   **[sql] [bug] [regression]**

    修复了一个回归问题，即 `text()` 构造不再被接受为“whens”列表中的目标情况在 `case()` 构造内。这个回归问题似乎与试图防范在此处传递的某些形式的文字值被视为模棱两可有关；然而，目标情况不应该被解释为开放式的 SQL 表达式，就像其他地方一样，文字字符串或元组将被转换为绑定参数，就像其他情况一样。

    参考：[#7287](https://www.sqlalchemy.org/trac/ticket/7287)

### 模式

+   **[schema] [bug]**

    修复了在 `Table` 中的问题，当与 `Table.extend_existing` 一起传递时，`Table.implicit_returning` 参数将不会被正确处理以增强现有的 `Table`。

    参考：[#7295](https://www.sqlalchemy.org/trac/ticket/7295)

### postgresql

+   **[postgresql] [usecase] [asyncpg]**

    添加了可重写方法 `PGDialect_asyncpg.setup_asyncpg_json_codec` 和 `PGDialect_asyncpg.setup_asyncpg_jsonb_codec` 编解码器，用于在使用 asyncpg 时注册 JSON/JSONB 编解码器所需的任务。更改是将方法拆分为单独的、可重写的方法，以支持需要修改或禁用这些特定编解码器设置的第三方方言。

    参考：[#7284](https://www.sqlalchemy.org/trac/ticket/7284)

+   **[postgresql] [bug] [asyncpg]**

    将 asyncpg 方言绑定到“float” PostgreSQL 类型的 `Float` 类型，而不是“numeric”，以便容纳值 `float(inf)`。添加了对“inf”值持久性的测试套件支持。

    参考：[#7283](https://www.sqlalchemy.org/trac/ticket/7283)

+   **[postgresql] [pg8000]**

    改进了在使用 pg8000 方言的 PostgreSQL 时的数组处理。

    参考：[#7167](https://www.sqlalchemy.org/trac/ticket/7167)

### mysql

+   **[mysql] [bug] [mariadb]**

    重新组织了保留字列表，分为两个独立的列表，一个用于 MySQL，一个用于 MariaDB，以便更准确地管理这些不同的单词集；调整了 MySQL/MariaDB 方言，以根据明确配置或服务器版本检测到的“MySQL”或“MariaDB”后端在这些列表之间切换。添加了通过 MySQL 8 和当前 MariaDB 版本的所有当前保留字，包括最近添加的关键字如“lead”。感谢 Kevin Kirsche 的拉取请求。

    参考：[#7167](https://www.sqlalchemy.org/trac/ticket/7167)

+   **[mysql] [bug]**

    修复了 MySQL 中`Insert.on_duplicate_key_update()`的问题，当在 VALUES 表达式中使用表达式时，会渲染错误的列名。感谢 Cristian Sabaila 的拉取请求。

    参考：[#7281](https://www.sqlalchemy.org/trac/ticket/7281)

### mssql

+   **[mssql] [bug]**

    调整了编译器生成的“后编译”符号，包括用于“扩展 IN”以及“模式转换映射”的符号，不再直接基于带有下划线的普通括号字符串，因为这与 SQL Server 的引用格式直接冲突，后者也使用括号，这会在编译器替换“后编译”和“模式转换”符号时产生错误匹配。该问题在使用`Inspector.get_schema_names()`方法与`Connection.execution_options.schema_translate_map`功能结合使用时创建了易于重现的示例，以及在使用与内部名称“POSTCOMPILE”重叠的符号与“扩展 IN”等功能一起使用时的不太可能的情况。

    参考：[#7300](https://www.sqlalchemy.org/trac/ticket/7300)

## 1.4.26

发布日期：2021 年 10 月 19 日

### orm

+   **[orm] [bug]**

    改进了在配置具有联合表继承的映射时生成的异常消息，其中两个表没有设置外键关系，或者它们设置了多个外键关系。消息现在是 ORM 特定的，并包含了可能需要使用`Mapper.inherit_condition`参数，特别是对于模糊的外键情况。

+   **[orm] [bug]**

    修复了`with_loader_criteria()`功能的问题，其中 ON 条件不会被添加到一个形如`select(A).join(B)`的查询的 JOIN 中，指定一个目标同时使用隐式 ON 子句。

    参考：[#7189](https://www.sqlalchemy.org/trac/ticket/7189)

+   **[orm] [bug]**

    修复了一个 bug，即 ORM “插件”，用于功能如`with_loader_criteria()` 正常工作，将不会应用于从 ORM 列表达式查询的 `select()`，如果它使用了 `ColumnElement.label()` 修饰符。

    参考：[#7205](https://www.sqlalchemy.org/trac/ticket/7205)

+   **[orm] [bug]**

    将在 `scoped_session` 和 `async_scoped_session()` 中添加在 [#6991](https://www.sqlalchemy.org/trac/ticket/6991) 中添加的缺失方法。

    参考：[#7103](https://www.sqlalchemy.org/trac/ticket/7103)

+   **[orm] [bug]**

    在 `Query.join()` 和 `Select.join()` 的功能中添加了额外的警告消息层，其中现在将继续发生“自动别名”的几个地方被指出为要避免的模式，主要是特定于联接表继承的领域，其中共享共同基表的类被联接在一起而不使用显式别名。一个情况发出了一个不推荐的传统警告，另一个情况已完全被弃用。

    在 ORM join() 中自动别名的功能，对于重叠映射表并不一致地适用于所有 API，比如`contains_eager()`，而不是继续尝试使这些用例在所有地方都能正常工作，用更明确的用户模式替换更清晰，更不容易出错，并进一步简化 SQLAlchemy 的内部结构。

    警告包括指向错误.rst 页面的链接，其中展示了每个模式以及建议的修复模式。

    另请参阅

    由于原始 clauseelement 而自动生成别名

    由于表重叠而自动生成别名

    参考：[#6972](https://www.sqlalchemy.org/trac/ticket/6972), [#6974](https://www.sqlalchemy.org/trac/ticket/6974)

+   **[orm] [bug]**

    修复了一个 bug，即在关闭`Session`后迭代`Result`会部分附加对象到该会话中，处于基本无效状态。现在，如果从已关闭或在生成`Result`后调用`Session.expunge_all()`方法的`Session`中迭代**未缓冲**的结果，将引发异常，并附带指向新文档的链接。`prebuffer_rows` 执行选项，如由客户端端结果集的 asyncio 扩展自动使用，可用于生成一个预缓冲 ORM 对象的`Result`，在这种情况下，迭代结果将产生一系列分离的对象。

    另请参见

    对象无法转换为“持久”状态，因为此标识映射已不再有效。

    参考：[#7128](https://www.sqlalchemy.org/trac/ticket/7128)

+   **[orm] [bug]**

    与[#7153](https://www.sqlalchemy.org/trac/ticket/7153)相关，修复了一个问题，即对于选择“常量”值表达式的“适应” SELECT 语句的结果列查找将失败，最典型的是 NULL 表达式，例如在联接的急加载与限制/偏移量一起发生的情况。这在总体上是一个回归问题，因为问题[#6259](https://www.sqlalchemy.org/trac/ticket/6259)删除了对常量（如 NULL、“true”和“false”）的所有“适应”，当重写 SQL 语句中的表达式时，但这破坏了使用相同适应逻辑将常量解析为标记表达式以用于结果集定位的情况。

    参考：[#7154](https://www.sqlalchemy.org/trac/ticket/7154)

+   **[orm] [bug] [regression]**

    修复了一个回归问题，即在某些组合中使用使得对象无法被 pickled 的加载器选项的情况下，例如将`joinedload()`加载策略与子元素的`raiseload('*')`结合使用时，ORM 加载的对象无法被 pickled。

    参考：[#7134](https://www.sqlalchemy.org/trac/ticket/7134)

+   **[orm] [bug] [regression]**

    修复了一个回归，其中将`hybrid_property`属性或映射的`composite()`属性作为键传递给`Update.values()`方法以进行 ORM 启用的`Update`语句，以及在通过遗留的`Query.update()`方法使用它时，会在 UPDATE 语句的编译阶段内对传入的 ORM/hybrid/composite 值进行处理，这意味着在发生缓存的情况下，同一语句的后续调用将不再接收到正确的值。这不仅包括使用`hybrid_property.update_expression()`方法的混合体，还包括任何使用纯混合属性的情况。对于复合体，该问题导致生成一个不可重复的缓存键，这将破坏缓存并且可能用重复语句填满语句缓存。

    `Update`构造现在在首次生成构造之前处理传递给`Update.values()`和`Update.ordered_values()`的键/值对，并在生成缓存键之前进行处理，以便每次都处理键/值对，并且生成的缓存键是针对最终将在语句中使用的各个列/值对。

    参考：[#7209](https://www.sqlalchemy.org/trac/ticket/7209)

+   **[orm]**

    将`Query`对象传递给`Session.execute()`不是该对象的预期用法，并且现在会引发弃用警告。

    参考：[#6284](https://www.sqlalchemy.org/trac/ticket/6284)

### 例子

+   **[例子] [错误]**

    修复了 examples/versioned_rows 中示例使用 SQLAlchemy 1.4 API 的正确性问题；这些示例在进行 API 更改（例如从 `Session.is_modified()` 中移除“passive”以及添加 `SessionEvents.do_orm_execute()` 事件钩子）时被忽略。

    参考：[#7169](https://www.sqlalchemy.org/trac/ticket/7169)

### engine

+   **[engine] [bug]**

    修复了一个问题，即当传递了七个参数的完整位置参数列表时，`URL` 构造函数的弃用警告（指示应使用 `URL.create()` 方法）不会发出；此外，如果以这种方式调用构造函数，则现在将对 URL 参数进行验证，而以前则会跳过此验证。

    参考：[#7130](https://www.sqlalchemy.org/trac/ticket/7130)

+   **[engine] [bug] [postgresql]**

    `Inspector.reflect_table()` 方法现在支持反映没有用户定义列的表格。这使得 `MetaData.reflect()` 能够正确地完成对包含这种表格的数据库的反射。目前，已知常见的数据库后端中只有 PostgreSQL 支持这样的构造。

    参考：[#3247](https://www.sqlalchemy.org/trac/ticket/3247)

+   **[engine] [bug]**

    为所有 SQLAlchemy 异常对象实现了适当的 `__reduce__()` 方法，以确保它们在被 pickle 化时都支持干净的往返，因为异常对象经常被序列化用于各种调试工具的目的。

    参考：[#7077](https://www.sqlalchemy.org/trac/ticket/7077)

### sql

+   **[sql] [bug]**

    修复了使用 `FunctionElement.within_group()` 构造的 SQL 查询无法被 pickle 化的问题，通常在使用 `sqlalchemy.ext.serializer` 扩展时出现，但也适用于一般的通用 pickle 化。

    参考：[#6520](https://www.sqlalchemy.org/trac/ticket/6520)

+   **[sql] [bug]**

    修复了新引入的 `HasCTE.cte.nesting` 参数中的问题，该问题与 [#4123](https://www.sqlalchemy.org/trac/ticket/4123) 一起使用递归 `CTE` 和 UNION 时不会正确编译。另外，对一些调整进行了一些调整，以使 `CTE` 构造函数创建正确的缓存键。拉取请求由 Eric Masseran 提供。

    参考：[#4123](https://www.sqlalchemy.org/trac/ticket/4123)

+   **[sql] [bug]**

    考虑到传递给 `table()` 构造函数的 `table.schema` 参数，使其在访问 `TableClause.fullname` 属性时被考虑进去。

    参考：[#7061](https://www.sqlalchemy.org/trac/ticket/7061)

+   **[sql] [bug]**

    修复了 `ColumnOperators.any_()` / `ColumnOperators.all_()` 函数 / 方法中的一致性问题，这些函数具有的特殊行为是“翻转”表达式，使得“ANY” / “ALL” 表达式始终位于右侧，如果比较的对象是 None 值，则不会起作用，即，“column.any_() == None” 应该产生与 “null() == column.any_()” 相同的 SQL 表达式。此外，增加了更多文档来澄清这一点，还提到 any_() / all_() 通常优于 ARRAY 版本的 “any()” / “all()”。

    参考：[#7140](https://www.sqlalchemy.org/trac/ticket/7140)

+   **[sql] [bug] [regression]**

    修复了“扩展 IN”在使用 `TypeEngine.bind_expression()` 方法的数据类型上无法正确运行的问题，其中该方法需要应用于 IN 表达式的每个元素而不是整个 IN 表达式本身。

    参考：[#7177](https://www.sqlalchemy.org/trac/ticket/7177)

+   **[sql] [bug]**

    调整了 1.4 中新的“列消歧义”逻辑，其中重复的相同表达式会得到“额外匿名”标签，以便在重复元素每次都是相同的 Python 表达式对象时更积极地对这些标签进行去重，例如在使用像 `null()` 这样的“单例”值时发生的情况。这是基于这样一个观察：至少一些数据库（例如 MySQL，但不包括 SQLite）将在子查询中重复使用相同的标签时引发错误。

    参考：[#7153](https://www.sqlalchemy.org/trac/ticket/7153)

### mypy

+   **[mypy] [bug]**

    修复了在 mypy 插件中的问题，以改进一些检测包含自定义 Python 枚举类的`Enum()` SQL 类型的问题。拉取请求由 Hiroshi Ogawa 提供。

    参考：[#6435](https://www.sqlalchemy.org/trac/ticket/6435)

### postgresql

+   **[postgresql] [bug]**

    增加了“断开连接”条件，以处理“SSL SYSCALL 错误：地址错误”错误消息，由 psycopg2 报告。拉取请求由 Zeke Brechtel 提供。

    参考：[#5387](https://www.sqlalchemy.org/trac/ticket/5387)

+   **[postgresql] [bug] [regression]**

    修复了针对数组元素系列的 IN 表达式的问题，如 PostgreSQL 中所做的，由于 SQLAlchemy Core 中“expanding IN”功能的多个问题，导致功能无法正常运行。psycopg2 方言现在使用 `TypeEngine.bind_expression()` 方法与 `ARRAY` 结合使用正确的类型转换。asyncpg 方言不受此问题影响，因为它在驱动程序级别而不是在编译器级别应用绑定级别的转换。

    参考：[#7177](https://www.sqlalchemy.org/trac/ticket/7177)

### mysql

+   **[mysql] [bug] [mariadb]**

    修复了适应 MariaDB 10.6 系列的问题，包括 mariadb-connector Python 驱动程序（仅支持 SQLAlchemy 1.4）和 mysqlclient DBAPI（适用于 1.3 和 1.4）中的向后不兼容更改。当编码被声明为“utf8”时，这些客户端库现在会报告“utf8mb3”编码符号，导致 MySQL 方言中出现查找和编码错误，而这种方言不希望出现此符号。更新了 MySQL 基础库以适应此 utf8mb3 符号的报告，以及测试套件。感谢 Georg Richter 的支持。

    此更改也**回溯**到：1.3.25

    参考：[#7115](https://www.sqlalchemy.org/trac/ticket/7115), [#7136](https://www.sqlalchemy.org/trac/ticket/7136)

+   **[mysql] [bug]**

    修复了 MySQL 中`match()`构造中的问题，其中传递类似`bindparam()`或其他 SQL 表达式作为“against”参数的子句表达式会失败。感谢 Anton Kovalevich 的拉取请求。

    参考：[#7144](https://www.sqlalchemy.org/trac/ticket/7144)

+   **[mysql] [bug]**

    修复了安装问题，如果未安装“greenlet”，则无法导入`sqlalchemy.dialects.mysql`模块。

    参考：[#7204](https://www.sqlalchemy.org/trac/ticket/7204)

### mssql

+   **[mssql] [usecase]**

    添加了对 SQL Server 外键选项的反射支持，包括“ON UPDATE”和“ON DELETE”值为“CASCADE”和“SET NULL”。

+   **[mssql] [bug]**

    修复了`Inspector.get_foreign_keys()`中存在的问题，如果外键建立在唯一索引而不是唯一约束上，则会被省略。

    参考：[#7160](https://www.sqlalchemy.org/trac/ticket/7160)

+   **[mssql] [bug]**

    修复了`Inspector.has_table()`存在的问题，当查询 tempdb 时，如果一个本地临时表与另一个会话中返回的同名表首先返回，则会返回 False。这是[#6910](https://www.sqlalchemy.org/trac/ticket/6910)的延续，该问题仅考虑了临时表存在于另一个会话中而不是当前会话中。

    参考：[#7168](https://www.sqlalchemy.org/trac/ticket/7168)

+   **[mssql] [bug] [regression]**

    修复了 SQL Server 中`DATETIMEOFFSET`数据类型中的错误，其中 ODBC 实现不会生成正确的 DDL，对于使用`dialect.type_descriptor()`方法转换类型的情况，该方法的使用在一些文档示例中有所说明，尽管对于大多数数据类型并不是必需的。回归由[#6366](https://www.sqlalchemy.org/trac/ticket/6366)引入。作为这一变更的一部分，SQL Server 日期类型的完整列表已经修改为返回生成与超类型相同 DDL 名称的“方言实现”。

    参考：[#7129](https://www.sqlalchemy.org/trac/ticket/7129)

### orm

+   **[orm] [bug]**

    当配置具有联接表继承的映射时，其中两个表没有设置外键关系或者它们设置了多个外键关系时，改进了生成的异常消息。该消息现在是 ORM 特定的，并包含了可能特别针对模糊外键情况需要 `Mapper.inherit_condition` 参数的上下文。

+   **[orm] [bug]**

    修复了 `with_loader_criteria()` 功能中的问题，其中对于形式为 `select(A).join(B)` 的查询，ON 条件不会添加到 JOIN 中，同时使用隐式 ON 子句指定目标。

    参考：[#7189](https://www.sqlalchemy.org/trac/ticket/7189)

+   **[orm] [bug]**

    修复了 ORM “插件”存在的错误，这是使 `with_loader_criteria()` 等功能正常工作所必需的，如果它使用了 `ColumnElement.label()` 修饰符，则不会将其应用于从 ORM 列表达式查询的 `select()`。

    参考：[#7205](https://www.sqlalchemy.org/trac/ticket/7205)

+   **[orm] [bug]**

    将在 [#6991](https://www.sqlalchemy.org/trac/ticket/6991) 中添加的缺失方法添加到 `scoped_session` 和 `async_scoped_session()` 中。

    参考：[#7103](https://www.sqlalchemy.org/trac/ticket/7103)

+   **[orm] [bug]**

    在 `Query.join()` 和 ORM 版本的 `Select.join()` 功能中增加了一层额外的警告消息，当“自动别名”继续发生的地方有几处时，将现在标记为要避免的模式，主要是特定于联接表继承的区域，其中共享相同基表的类被一起连接而不使用显式别名。其中一种情况发出了一个不推荐的遗留警告，另一种情况已被完全弃用。

    ORM join() 中的自动别名对于重叠映射表并不与所有 API（如`contains_eager()`）一致工作，而不是继续尝试使这些用例在所有地方工作，用更明确的用户模式替换更清晰，更不容易出错，并进一步简化了 SQLAlchemy 的内部结构。

    警告包括链接到错误.rst 页面的链接，其中演示了每个模式以及修复建议。

    另请参见

    自动生成原始 clauseelement 的别名

    由于重叠表而自动生成别名

    参考：[#6972](https://www.sqlalchemy.org/trac/ticket/6972), [#6974](https://www.sqlalchemy.org/trac/ticket/6974)

+   **[orm] [bug]**

    修复了一个 bug，即在关闭`Session`后迭代`Result`，会部分附加对象到该会话中，处于基本无效状态。现在，如果从已关闭或在生成`Result`后调用`Session.expunge_all()`方法的`Session`中迭代**未缓冲**的结果，将引发异常，并附带指向新文档的链接。`prebuffer_rows`执行选项，如由客户端端结果集的 asyncio 扩展自动使用，可用于生成一个预缓冲 ORM 对象的`Result`，在这种情况下，迭代结果将产生一系列分离的对象。

    另请参见

    对象无法转换为“持久”状态，因为此标识映射不再有效。

    参考：[#7128](https://www.sqlalchemy.org/trac/ticket/7128)

+   **[orm] [bug]**

    修复了与[#7153](https://www.sqlalchemy.org/trac/ticket/7153)相关的问题，即结果列查找在“适应”SELECT 语句中失败的问题，这些语句通常选择“常量”值表达式，最典型的是 NULL 表达式，例如在联接的急加载与 limit/offset 结合使用时。这在总体上是由问题[#6259](https://www.sqlalchemy.org/trac/ticket/6259)引起的回归，该问题删除了对常量（如 NULL、“true”和“false”）的所有“适应”，当在 SQL 语句中重写表达式时，但这破坏了使用相同适应逻辑将常量解析为标记表达式以用于结果集定位的情况。

    参考：[#7154](https://www.sqlalchemy.org/trac/ticket/7154)

+   **[orm] [bug] [regression]**

    修复了 ORM 加载的对象无法在某些组合中使用 loader 选项的情况下进行 pickle 的回归问题，例如将`joinedload()`加载策略与子元素的`raiseload('*')`组合在一起时。

    参考：[#7134](https://www.sqlalchemy.org/trac/ticket/7134)

+   **[orm] [bug] [regression]**

    修复了使用`hybrid_property`属性或作为传递给 ORM 启用的`Update`语句的`Update.values()`方法的键的映射`composite()`属性时的回归，以及在通过传统的`Query.update()`方法使用它时，将在 UPDATE 语句的编译阶段内处理传入的 ORM/hybrid/composite 值的情况，这意味着在发生缓存的情况下，对相同语句的后续调用将不再接收到正确的值。这不仅包括使用`hybrid_property.update_expression()`方法的混合体，还包括任何对普通混合属性的使用。对于复合体，该问题导致生成一个不可重复的缓存键，这将破坏缓存并可能使语句缓存中充满重复的语句。

    当构造生成时，`Update` 构造现在会提前处理传递给 `Update.values()` 和 `Update.ordered_values()` 的键/值对，而不是在生成缓存密钥之前，这样每次都会处理键/值对，并且生成的缓存密钥是针对最终在语句中使用的单个列/值对生成的。

    参考：[#7209](https://www.sqlalchemy.org/trac/ticket/7209)

+   **[orm]**

    将 `Query` 对象传递给 `Session.execute()` 不是该对象的预期用法，现在会引发弃用警告。

    参考：[#6284](https://www.sqlalchemy.org/trac/ticket/6284)

### 示例

+   **[examples] [bug]**

    修复了示例中使用 SQLAlchemy 1.4 API 的错误示例；当像从 `Session.is_modified()` 中移除“被动”这样的 API 更改以及添加 `SessionEvents.do_orm_execute()` 事件钩子时，这些示例被忽略了。

    参考：[#7169](https://www.sqlalchemy.org/trac/ticket/7169)

### 引擎

+   **[engine] [bug]**

    修复了一个问题，即当传递了七个参数的完整位置参数列表时，用于指示应使用 `URL.create()` 方法的 `URL` 构造函数的弃用警告不会发出；另外，如果以这种方式调用构造函数，现在将对 URL 参数进行验证，而以前是跳过验证的。

    参考：[#7130](https://www.sqlalchemy.org/trac/ticket/7130)

+   **[engine] [bug] [postgresql]**

    `Inspector.reflect_table()` 方法现在支持反映没有用户定义列的表。这允许 `MetaData.reflect()` 在包含此类表的数据库上正确完成反射。目前，已知常见数据库后端中只有 PostgreSQL 支持这样的构造。

    参考：[#3247](https://www.sqlalchemy.org/trac/ticket/3247)

+   **[engine] [bug]**

    为所有 SQLAlchemy 异常对象实现了适当的`__reduce__()`方法，以确保它们在被 pickle 时都能支持干净的往返，因为异常对象经常被序列化用于各种调试工具的目的。

    参考：[#7077](https://www.sqlalchemy.org/trac/ticket/7077)

### sql

+   **[sql] [bug]**

    修复了使用`FunctionElement.within_group()`构造的 SQL 查询无法被 pickle 的问题，通常在使用`sqlalchemy.ext.serializer`扩展时会出现，但也适用于一般的通用 pickle。

    参考：[#6520](https://www.sqlalchemy.org/trac/ticket/6520)

+   **[sql] [bug]**

    修复了在新的`HasCTE.cte.nesting`参数中引入的问题，该问题是在[#4123](https://www.sqlalchemy.org/trac/ticket/4123)中引入的，其中使用`HasCTE.cte.recursive`的递归`CTE`在典型情况下与 UNION 结合使用时无法正确编译。此外，对一些调整使得`CTE`构造函数创建正确的缓存键。拉取请求由 Eric Masseran 提供。

    参考：[#4123](https://www.sqlalchemy.org/trac/ticket/4123)

+   **[sql] [bug]**

    考虑传递给`table()`构造函数的`table.schema`参数，这样在访问`TableClause.fullname`属性时会考虑到它。

    参考：[#7061](https://www.sqlalchemy.org/trac/ticket/7061)

+   **[sql] [bug]**

    修复了`ColumnOperators.any_()` / `ColumnOperators.all_()`函数/方法中的一个不一致性，这些函数具有的特殊行为是“翻转”表达式，使得“ANY” / “ALL”表达式始终位于右侧，如果比较的对象是 None 值，则这种行为将不起作用，也就是说，“column.any_() == None”应该产生与“null() == column.any_()”相同的 SQL 表达式。还添加了更多文档来澄清这一点，以及提到 any_() / all_()通常优先于 ARRAY 版本的“any()” / “all()”。

    参考：[#7140](https://www.sqlalchemy.org/trac/ticket/7140)

+   **[sql] [bug] [regression]**

    修复了“扩展 IN”无法正确处理使用`TypeEngine.bind_expression()`方法的数据类型的问题，其中该方法需要应用于 IN 表达式的每个元素而不是整个 IN 表达式本身。

    参考：[#7177](https://www.sqlalchemy.org/trac/ticket/7177)

+   **[sql] [bug]**

    调整了 1.4 版本中新引入的“列消歧”逻辑，其中重复的相同表达式会获得“额外匿名”标签，以便在重复元素是每次相同的 Python 表达式对象时更积极地去重这些标签，例如在使用像`null()`这样的“单例”值时发生的情况。这是基于这样的观察，至少一些数据库（例如 MySQL，但不是 SQLite）会在子查询中重复相同标签时引发错误。

    参考：[#7153](https://www.sqlalchemy.org/trac/ticket/7153)

### mypy

+   **[mypy] [bug]**

    修复了 mypy 插件中的问题，改进了一些检测包含自定义 Python 枚举类的`Enum()` SQL 类型的问题。感谢 Hiroshi Ogawa 的拉取请求。

    参考：[#6435](https://www.sqlalchemy.org/trac/ticket/6435)

### postgresql

+   **[postgresql] [bug]**

    添加了一个“断开连接”条件，用于“SSL SYSCALL error: Bad address”错误消息，如 psycopg2 报告。感谢 Zeke Brechtel 的拉取请求。

    参考：[#5387](https://www.sqlalchemy.org/trac/ticket/5387)

+   **[postgresql] [bug] [regression]**

    修复了针对一系列数组元素的 IN 表达式的问题，就像在 PostgreSQL 中可以做的那样，由于 SQLAlchemy Core 中“扩展 IN”功能的多个问题，在 1.4 版本中标准化，因此无法正确运行。现在，psycopg2 方言使用`TypeEngine.bind_expression()`方法与`ARRAY`结合使用，以便对元素应用正确的转换。asyncpg 方言不受此问题影响，因为它在驱动程序级别而不是在编译器级别应用绑定级别的转换。

    参考：[#7177](https://www.sqlalchemy.org/trac/ticket/7177)

### mysql

+   **[mysql] [bug] [mariadb]**

    修复以适应 MariaDB 10.6 系列的变化，包括 mariadb-connector Python 驱动程序（仅支持 SQLAlchemy 1.4）和自动使用的 mysqlclient DBAPI（适用于 1.3 和 1.4）中的不兼容变化。当编码状态为“utf8”时，这些客户端库现在报告“utf8mb3”编码符号，导致 MySQL 方言中出现查找和编码错误。更新了 MySQL 基础库以适应报告此 utf8mb3 符号以及测试套件的更改。感谢 Georg Richter 的支持。

    此更改也已**反向移植**到：1.3.25

    参考：[#7115](https://www.sqlalchemy.org/trac/ticket/7115), [#7136](https://www.sqlalchemy.org/trac/ticket/7136)

+   **[mysql] [bug]**

    修复了 MySQL 中`match()`构造中的问题，其中传递诸如`bindparam()`或其他 SQL 表达式作为“against”参数的子句表达式将失败。感谢 Anton Kovalevich 的 Pull 请求。

    参考：[#7144](https://www.sqlalchemy.org/trac/ticket/7144)

+   **[mysql] [bug]**

    修复了安装问题，即如果未安装“greenlet”，则无法导入`sqlalchemy.dialects.mysql`模块。

    参考：[#7204](https://www.sqlalchemy.org/trac/ticket/7204)

### mssql

+   **[mssql] [usecase]**

    为 SQL Server 外键选项添加了反射支持，包括“ON UPDATE”和“ON DELETE”值为“CASCADE”和“SET NULL”。

+   **[mssql] [bug]**

    修复了`Inspector.get_foreign_keys()`的问题，如果外键建立在唯一索引而不是唯一约束上，则会省略外键。

    参考：[#7160](https://www.sqlalchemy.org/trac/ticket/7160)

+   **[mssql] [bug]**

    修复了`Inspector.has_table()`的问题，在查询 tempdb 时，如果一个具有相同名称的本地临时表首先从不同会话返回，它会返回 False。这是对[#6910](https://www.sqlalchemy.org/trac/ticket/6910)的延续，该问题仅考虑了在备用会话中存在临时表而不是当前会话中存在临时表的情况。

    参考：[#7168](https://www.sqlalchemy.org/trac/ticket/7168)

+   **[mssql] [bug] [regression]**

    修复了 SQL Server `DATETIMEOFFSET` 数据类型中的错误，其中 ODBC 实现未生成正确的 DDL，对于使用 `dialect.type_descriptor()` 方法转换类型的情况，该方法的使用在一些文档示例中有所说明，尽管对于大多数数据类型并非必需。回归由 [#6366](https://www.sqlalchemy.org/trac/ticket/6366) 引入。作为这一更改的一部分，SQL Server 日期类型的完整列表已经修改，以返回生成与超类型相同的 DDL 名称的“方言实现”。

    参考：[#7129](https://www.sqlalchemy.org/trac/ticket/7129)

## 1.4.25

发布日期：2021 年 9 月 22 日

### 平台

+   **[platform] [bug] [regression]**

    由于 [#7024](https://www.sqlalchemy.org/trac/ticket/7024) 导致的回归已修复，`greenlet` 依赖使用的“平台机器”名称重新组织时拼写错误了“aarch64”，并且还遗漏了大写的“AMD64”，这是 Windows 机器所需的。感谢 James Dow 提交的拉取请求。

    参考：[#7024](https://www.sqlalchemy.org/trac/ticket/7024)

### 平台

+   **[platform] [bug] [regression]**

    由于 [#7024](https://www.sqlalchemy.org/trac/ticket/7024) 导致的回归已修复，`greenlet` 依赖使用的“平台机器”名称重新组织时拼写错误了“aarch64”，并且还遗漏了大写的“AMD64”，这是 Windows 机器所需的。感谢 James Dow 提交的拉取请求。

    参考：[#7024](https://www.sqlalchemy.org/trac/ticket/7024)

## 1.4.24

发布日期：2021 年 9 月 22 日

### 平台

+   **[platform] [bug]**

    进一步调整了 setup.cfg 中“greenlet” 包的指定符，使用了一长串“or”表达式，以便将 `platform_machine` 的比较与特定标识符仅匹配完整字符串。

    参考：[#7024](https://www.sqlalchemy.org/trac/ticket/7024)

### orm

+   **[orm] [usecase]**

    通过新的 `Session.merge.options` 参数，为 `Session.merge()` 和 `AsyncSession.merge()` 添加了加载器选项，这将应用给定的加载器选项到 merge 内部使用的 `get()`，允许在合并过程中加载新对象时应用关系的急切加载等。感谢 Daniel Stone 提交的拉取请求。

    参考：[#6955](https://www.sqlalchemy.org/trac/ticket/6955)

+   **[orm] [bug] [regression]**

    修复了 ORM 问题，其中传递给 `query()` 或启用 ORM 的 `select()` 的列表达式将根据对象的标识进行去重，例如像 `select(A.id, null(), null())` 这样的短语将只产生一个“NULL”表达式，而在 1.3 版本中并非如此。然而，这个改变也允许 ORM 表达式按照给定的方式呈现，例如 `select(A.data, A.data)` 将产生一个包含两列的结果行。

    参考：[#6979](https://www.sqlalchemy.org/trac/ticket/6979)

+   **[orm] [bug] [regression]**

    修复了最近修复的 `Query.with_entities()` 方法中的问题，其中确定仅对传统 ORM `Query` 对象进行自动去重的标志会在 `with_entities()` 调用设置 `Query` 为返回仅列的行时不适当地设置为 `True` 的情况下。

    参考：[#6924](https://www.sqlalchemy.org/trac/ticket/6924)

### engine

+   **[engine] [usecase] [asyncio]**

    改进了适配驱动程序（如 asyncio 驱动程序）使用的接口，以访问驱动程序返回的实际连接对象。

    `_ConnectionFairy` 对象有两个新属性：

    +   `_ConnectionFairy.dbapi_connection` 总是表示一个与 DBAPI 兼容的对象。对于 pep-249 驱动程序，这是一直以来的 DBAPI 连接，以前通过 `.connection` 属性访问。对于 SQLAlchemy 转换为 pep-249 接口的 asyncio 驱动程序，返回的对象通常是一个名为 `AdaptedConnection` 的 SQLAlchemy 适配对象。

    +   `_ConnectionFairy.driver_connection` 总是表示第三方 pep-249 DBAPI 或异步驱动程序维护的实际连接对象。对于标准的 pep-249 DBAPI，这将始终是与 `dbapi_connection` 相同的对象。对于 asyncio 驱动程序，它将是底层的仅限 asyncio 的连接对象。

    `.connection` 属性仍然可用，现在是 `.dbapi_connection` 的一个传统别名。

    另请参阅

    在使用 Engine 时如何获取原始的 DBAPI 连接？

    参考：[#6832](https://www.sqlalchemy.org/trac/ticket/6832)

+   **[engine] [usecase] [orm]**

    添加了新方法`Session.scalars()`、`Connection.scalars()`、`AsyncSession.scalars()`和`AsyncSession.stream_scalars()`，它们提供了一个快捷方式，用于接收基于行的`Result`对象并通过`Result.scalars()`方法将其转换为`ScalarResult`对象的用例，以返回值列表而不是行列表。这些新方法类似于长期存在的`Session.scalar()`和`Connection.scalar()`方法，用于仅从第一行返回单个值。拉取请求由 Miguel Grinberg 提供。

    参考：[#6990](https://www.sqlalchemy.org/trac/ticket/6990)

+   **[engine] [bug] [regression]**

    修复了`ConnectionEvents.before_execute()`方法能够修改传递的 SQL 语句对象并返回新对象以被调用的能力被意外移除的问题。此行为已经恢复。

    参考：[#6913](https://www.sqlalchemy.org/trac/ticket/6913)

+   **[engine] [bug]**

    确保在 `URL.create.password` 参数上调用`str()`，允许使用实现了`__str__()`方法作为密码属性的对象。还澄清了这样的一个对象不适合动态更改每个数据库连接的密码；应该使用生成动态认证令牌中的方法。

    参考：[#6958](https://www.sqlalchemy.org/trac/ticket/6958)

+   **[engine] [bug]**

    修复了`URL`中“drivername”的验证在期望字符串的地方未能适当响应`None`值的问题。

    参考：[#6983](https://www.sqlalchemy.org/trac/ticket/6983)

+   **[engine] [bug] [postgresql]**

    修复了一个问题，即当引擎的`create_engine.implicit_returning`设置为 False 时，如果与`Sequence`一起使用 PostgreSQL 的“快速插入多个”功能，以及如果与`Sequence`一起使用任何类型的“executemany”和“return_defaults()”组合，则会导致引擎无法正常工作。请注意，PostgreSQL 的“快速插入多个”在将 SQL 语句传递给驱动程序时默认使用“RETURNING”；总体而言，`create_engine.implicit_returning`标志是过时的，在现代 SQLAlchemy 中没有实际用途，并将在单独的更改中被弃用。

    参考：[#6963](https://www.sqlalchemy.org/trac/ticket/6963)

### sql

+   **[sql] [usecase]**

    在`CTE`构造函数和`HasCTE.cte()`方法中添加了新参数`HasCTE.cte.nesting`，标记 CTE 应保持在封闭 CTE 内部嵌套，而不是移动到最外层 SELECT 的顶层。虽然在绝大多数情况下，SQL 功能没有区别，但用户已经确定了一些真正需要 CTE 结构嵌套的边缘情况。非常感谢 Eric Masseran 在这个复杂功能上的大量工作。

    参考：[#4123](https://www.sqlalchemy.org/trac/ticket/4123)

+   **[sql] [bug]**

    在`FunctionElement`中实现了缺失的方法，虽然���被使用，但会导致 pylint 报告它们为未实现的抽象方法。

    参考：[#7052](https://www.sqlalchemy.org/trac/ticket/7052)

+   **[sql] [bug]**

    修复了两个问题，即当`select()`和`join()`的组合被调整以形成元素的副本时，不会完全复制与子查询相关的所有列对象的状态。这导致的一个关键问题是，使用`ClauseElement.params()`方法（可能应该移入传统类别，因为它效率低下且容易出错）会留下旧的`BindParameter`对象的副本，导致在执行时正确设置参数时出现问题。

    参考：[#7055](https://www.sqlalchemy.org/trac/ticket/7055)

+   **[sql] [bug]**

    修复了与新的`HasCTE.add_cte()`功能相关的问题，该功能会同时对两个“INSERT..FROM SELECT”语句进行配对，从而导致丢失对两个独立 SELECT 语句的跟踪，从而导致错误的 SQL。

    参考：[#7036](https://www.sqlalchemy.org/trac/ticket/7036)

+   **[sql] [bug]**

    修复了在“多值插入”中将 ORM 列表达式用作传递给`Insert.values()`的字典列表中的键时不会正确处理为正确的列表达式的问题。

    参考：[#7060](https://www.sqlalchemy.org/trac/ticket/7060)

### mypy

+   **[mypy] [bug]**

    修复了在解释 `query_expression()` 构造时 mypy 插件会崩溃的问题。

    参考：[#6950](https://www.sqlalchemy.org/trac/ticket/6950)

+   **[mypy] [bug]**

    修复了 mypy 插件中的问题，在这个问题中，如果映射类依赖于来自超类的 `__tablename__` 例程，则 mixin 上的列不会被正确解释。

    参考：[#6937](https://www.sqlalchemy.org/trac/ticket/6937)

### asyncio

+   **[asyncio] [feature] [mysql]**

    为 MySQL 和 MariaDB 添加了对 `asyncmy` asyncio 数据库驱动程序的初始支持。然而，这个驱动程序非常新，似乎是当前唯一的替代方案，因为目前似乎没有人维护 `aiomysql` 驱动程序，并且它不适用于当前的 Python 版本。非常感谢 long2ice 提交了这个方言的拉取请求。

    请参阅

    asyncmy

    参考：[#6993](https://www.sqlalchemy.org/trac/ticket/6993)

+   **[asyncio] [usecase]**

    `AsyncSession` 现在支持覆盖其作为代理实例使用的 `Session`。可以通过 `AsyncSession.sync_session_class` 参数传递自定义的 `Session` 类，或者通过子类化 `AsyncSession` 并指定自定义的 `AsyncSession.sync_session_class` 来传递自定义的 `Session` 类。

    参考：[#6746](https://www.sqlalchemy.org/trac/ticket/6746)

+   **[asyncio] [bug]**

    在修复了`AsyncSession.execute()`和`AsyncSession.stream()`中的一个 bug 后，要求 `execution_options` 在定义时必须是 `immutabledict` 的实例。现在它正确地接受任何映射。

    参考：[#6943](https://www.sqlalchemy.org/trac/ticket/6943)

+   **[asyncio] [bug]**

    在 `AsyncSession.connection()` 方法中添加了缺少的 `**kw` 参数。

+   **[asyncio] [bug]**

    弃用了在 asyncio 驱动程序中使用 `scoped_session`。在使用 Asyncio 时，应该使用 `async_scoped_session`。

    参考：[#6746](https://www.sqlalchemy.org/trac/ticket/6746)

### postgresql

+   **[postgresql] [bug]**

    为了避免用户配置了不同的搜索路径而导致阴影问题，对 `version()` 调用进行了限定。

    参考：[#6912](https://www.sqlalchemy.org/trac/ticket/6912)

+   **[postgresql] [bug]**

    `ENUM` 数据类型是 PostgreSQL 原生的，因此不应该与 `native_enum=False` 标志一起使用。如果将此标志传递给 `ENUM` 数据类型，此标志现在将被忽略，并发出警告；之前，此标志将导致类型对象无法正常工作。

    参考：[#6106](https://www.sqlalchemy.org/trac/ticket/6106)

### sqlite

+   **[sqlite] [bug]**

    修复了当 pysqlite 驱动程序的 SQLite 无效隔离级别时的错误消息未指示“AUTOCOMMIT”是有效隔离级别之一的错误。

### mssql

+   **[mssql] [bug] [reflection]**

    修复了 `sqlalchemy.engine.reflection.has_table()` 返回 `True` 的问题，这些表实际上属于不同的 SQL Server 会话（连接）。现在进行了额外的检查，以确保检测到的临时表实际上由当前会话拥有。

    参考：[#6910](https://www.sqlalchemy.org/trac/ticket/6910)

### oracle

+   **[oracle] [performance] [bug]**

    在反射查询中对 Oracle 系统视图（如 ALL_TABLES、ALL_TAB_CONSTRAINTS 等）中用于表名、所有者和其他 DDL 名称参数添加了 CAST(VARCHAR2(128))，以更好地支持针对这些列进行索引，因为它们以前会由于 Python 使用 Unicode 字符串而隐式处理为 NVARCHAR2；这些列在所有 Oracle 版本中都被记录为 VARCHAR2，长度根据服务器版本不同而变化，从 30 到 128 个字符不等。此外，已针对 Oracle 数据库启用了针对 Unicode 命名的 DDL 结构的测试支持。

    参考：[#4486](https://www.sqlalchemy.org/trac/ticket/4486)

### 平台

+   **[platform] [bug]**

    进一步调整了 setup.cfg 中“greenlet”包的规范，以使用一长串“or”表达式，这样 `platform_machine` 与特定标识符的比较只匹配完整的字符串。

    参考：[#7024](https://www.sqlalchemy.org/trac/ticket/7024)

### orm

+   **[orm] [usecase]**

    通过新的 `Session.merge.options` 参数向 `Session.merge()` 和 `AsyncSession.merge()` 添加了加载器选项，这将使 merge 过程内部使用的 `get()` 应用给定的加载器选项，从而在合并过程中加载新对象时应用关系的急切加载等。Daniel Stone 提供的拉取请求。

    参考：[#6955](https://www.sqlalchemy.org/trac/ticket/6955)

+   **[orm] [bug] [regression]**

    修复了一个 ORM 问题，其中传递给 `query()` 或启用 ORM 的 `select()` 的列表达式将根据对象的身份进行去重，例如像 `select(A.id, null(), null())` 这样的短语将只产生一个“NULL”表达式，而以前不是这样在 1.3 版中。然而，该变更还允许 ORM 表达式按照给定的方式呈现，例如 `select(A.data, A.data)` 将生成一个具有两列的结果行。

    参考：[#6979](https://www.sqlalchemy.org/trac/ticket/6979)

+   **[orm] [bug] [regression]**

    修复了最近修复的 `Query.with_entities()` 方法中的问题，其中确定仅对传统 ORM `Query` 对象进行自动唯一化的标志在 `with_entities()` 调用将 `Query` 设置为返回仅列的行时会不适当地设置为 `True`，这些行不是唯一的。

    参考：[#6924](https://www.sqlalchemy.org/trac/ticket/6924)

### engine

+   **[engine] [usecase] [asyncio]**

    改进了适配驱动程序（如 asyncio 驱动程序）用于访问驱动程序返回的实际连接对象的接口。

    `_ConnectionFairy` 对象有两个新属性：

    +   `_ConnectionFairy.dbapi_connection` 总是代表一个兼容 DBAPI 的对象。对于 pep-249 驱动程序，这是一直以来的 DBAPI 连接，以前是通过 `.connection` 属性访问的。对于 SQLAlchemy 适配为 pep-249 接口的 asyncio 驱动程序，返回的对象通常是一个称为`AdaptedConnection`的 SQLAlchemy 适配对象。

    +   `_ConnectionFairy.driver_connection` 总是表示由使用的第三方 pep-249 DBAPI 或 async 驱动程序维护的实际连接对象。对于标准的 pep-249 DBAPI，这将始终是与 `dbapi_connection` 相同的对象。对于 asyncio 驱动程序，它将是底层的 asyncio-only 连接对象。

    `.connection` 属性仍然可用，现在是`.dbapi_connection` 的传统别名。

    另请参阅

    在使用 Engine 时如何获取原始的 DBAPI 连接？

    参考：[#6832](https://www.sqlalchemy.org/trac/ticket/6832)

+   **[engine] [usecase] [orm]**

    添加了新方法`Session.scalars()`，`Connection.scalars()`，`AsyncSession.scalars()`和`AsyncSession.stream_scalars()`，提供了一个快捷方式来接收基于行的`Result`对象，并通过`Result.scalars()`方法将其转换为`ScalarResult`对象，返回一个值列表而不是行列表。这些新方法类似于长期存在的`Session.scalar()`和`Connection.scalar()`方法，用于仅从第一行返回单个值。感谢 Miguel Grinberg 的 Pull request。

    参考：[#6990](https://www.sqlalchemy.org/trac/ticket/6990)

+   **[engine] [bug] [regression]**

    修复了`ConnectionEvents.before_execute()`方法修改传递的 SQL 语句对象并返回要调用的新对象的能力被意外移除的问题。已恢复此行为。

    参考：[#6913](https://www.sqlalchemy.org/trac/ticket/6913)

+   **[engine] [bug]**

    确保在`URL.create.password`参数上调用了`str()`，允许使用实现了`__str__()`方法的对象作为密码属性。还澄清了这样的对象不适合为每个数据库连接动态更改密码；应使用生成动态认证令牌中的方法。

    参考：[#6958](https://www.sqlalchemy.org/trac/ticket/6958)

+   **[engine] [bug]**

    在`URL`中修复了一个问题，在预期为字符串的地方，“drivername”的验证未能正确响应`None`值。

    参考：[#6983](https://www.sqlalchemy.org/trac/ticket/6983)

+   **[engine] [bug] [postgresql]**

    修复了一个问题，即将`create_engine.implicit_returning`设置为 False 的引擎在与`Sequence`结合使用 PostgreSQL 的“快速插入多个”功能时无法工作，以及在与`Sequence`结合使用任何类型的“executemany”并使用“return_defaults()”时也无法工作。请注意，PostgreSQL 的“快速插入多个”功能在 SQL 语句传递给驱动程序时使用“RETURNING”来定义；总的来说，`create_engine.implicit_returning`标志是遗留的，在现代 SQLAlchemy 中没有真正用途，并将在单独的更改中被弃用。

    参考：[#6963](https://www.sqlalchemy.org/trac/ticket/6963)

### sql

+   **[sql] [usecase]**

    向`CTE`构造函数和`HasCTE.cte()`方法添加了新参数`HasCTE.cte.nesting`，该参数标记了 CTE 应保持嵌套在封闭的 CTE 中，而不是移动到最外层 SELECT 的顶层。虽然在绝大多数情况下，SQL 功能没有区别，但用户已经确定了各种真正嵌套 CTE 结构有用的边缘情况。非常感谢 Eric Masseran 在这个复杂功能上的大量工作。

    参考：[#4123](https://www.sqlalchemy.org/trac/ticket/4123)

+   **[sql] [bug]**

    在`FunctionElement`中实现了缺失的方法，虽然未使用，但会导致 pylint 报告它们为未实现的抽象方法。

    参考：[#7052](https://www.sqlalchemy.org/trac/ticket/7052)

+   **[sql] [bug]**

    修复了当使用`select()`和`join()`的组合来形成元素的副本时，不会完全复制与子查询相关的所有列对象状态的两个问题。这导致的一个关键问题是，使用`ClauseElement.params()`方法（可能应该移入传统类别，因为它效率低下且容易出错）会留下旧的`BindParameter`对象的副本，导致在执行时正确设置参数时出现问题。

    参考：[#7055](https://www.sqlalchemy.org/trac/ticket/7055)

+   **[sql] [错误]**

    修复了与新的`HasCTE.add_cte()`功能相关的问题，同时配对两个“INSERT..FROM SELECT”语句会同时丢失两个独立的 SELECT 语句的跟踪，导致错误的 SQL。

    参考：[#7036](https://www.sqlalchemy.org/trac/ticket/7036)

+   **[sql] [错误]**

    修复了在“多值插入”中将 ORM 列表达式用作传递给`Insert.values()`的字典列表中的键时，不会正确处理为正确的列表达式的问题。

    参考：[#7060](https://www.sqlalchemy.org/trac/ticket/7060)

### mypy

+   **[mypy] [错误]**

    修复了当解释`query_expression()`构造时，mypy 插件会崩溃的问题。

    参考：[#6950](https://www.sqlalchemy.org/trac/ticket/6950)

+   **[mypy] [错误]**

    修复了 mypy 插件中的问题，当映射类依赖于来自超类的`__tablename__`例程时，混合中的列不会被正确解释的问题。

    参考：[#6937](https://www.sqlalchemy.org/trac/ticket/6937)

### asyncio

+   **[asyncio] [功能] [mysql]**

    添加了对`asyncmy` asyncio 数据库驱动程序的初步支持，用于 MySQL 和 MariaDB。然而，这个驱动程序非常新，似乎是目前唯一的`aiomysql`驱动程序的替代品，后者目前似乎没有维护，并且不能与当前的 Python 版本一起使用。非常感谢 long2ice 为这个方言提供的拉取请求。

    另请参阅

    asyncmy

    参考：[#6993](https://www.sqlalchemy.org/trac/ticket/6993)

+   **[asyncio] [用例]**

    `AsyncSession`现在支持覆盖使用哪个`Session`作为代理实例。可以通过`AsyncSession.sync_session_class`参数传递自定义的`Session`类，或者通过子类化`AsyncSession`并指定自定义的`AsyncSession.sync_session_class`来传递。

    参考：[#6746](https://www.sqlalchemy.org/trac/ticket/6746)

+   **[asyncio] [bug]**

    修复了`AsyncSession.execute()`和`AsyncSession.stream()`中的一个 bug，需要在定义时将`execution_options`作为`immutabledict`的实例。现在可以正确接受任何映射。

    参考：[#6943](https://www.sqlalchemy.org/trac/ticket/6943)

+   **[asyncio] [bug]**

    向`AsyncSession.connection()`方法添加了缺失的`**kw`参数。

+   **[asyncio] [bug]**

    弃用在 asyncio 驱动程序中使用`scoped_session`。在使用 Asyncio 时，应改用`async_scoped_session`。

    参考：[#6746](https://www.sqlalchemy.org/trac/ticket/6746)

### postgresql

+   **[postgresql] [bug]**

    修正了在用户配置了不同搜索路径时避免遮蔽问题的`version()`调用。

    参考：[#6912](https://www.sqlalchemy.org/trac/ticket/6912)

+   **[postgresql] [bug]**

    `ENUM` 数据类型是 PostgreSQL 本地的，因此不应与`native_enum=False`标志一起使用。如果传递给`ENUM`数据类型，则此标志现在将被��略，并发出警告；以前，该标志会导致类型对象无法正确运行。

    参考：[#6106](https://www.sqlalchemy.org/trac/ticket/6106)

### sqlite

+   **[sqlite] [bug]**

    修复了在 pysqlite 驱动程序上对 SQLite 无效隔离级别的错误消息未指示“AUTOCOMMIT”是有效隔离级别之一的 bug。

### mssql

+   **[mssql] [bug] [reflection]**

    修复了一个问题，即`sqlalchemy.engine.reflection.has_table()`对于实际上属于不同 SQL Server 会话（连接）的本地临时表返回`True`的问题。现在会额外进行检查，以确保检测到的临时表实际上由当前会话拥有。

    参考：[#6910](https://www.sqlalchemy.org/trac/ticket/6910)

### oracle

+   **[oracle] [性能] [错误]**

    在反射查询中对 Oracle 系统视图（如 ALL_TABLES、ALL_TAB_CONSTRAINTS 等）中使用的“表名”、“所有者”和其他 DDL 名称参数添加了 CAST(VARCHAR2(128))，以更好地支持对这些列进行索引，因为由于 Python 对字符串使用 Unicode，它们以前会被隐式处理为 NVARCHAR2；这些列在所有 Oracle 版本中都被记录为 VARCHAR2，长度根据服务器版本的不同而变化，从 30 到 128 个字符不等。此外，已启用针对 Oracle 数据库的 Unicode 命名 DDL 结构的测试支持。

    参考：[#4486](https://www.sqlalchemy.org/trac/ticket/4486)

## 1.4.23

发布日期：2021 年 8 月 18 日

### 通用

+   **[通用] [错误]**

    修改了设置要求，使得`greenlet`仅作为默认要求出现在那些已知可以安装`greenlet`且在 pypi 上已有预构建二进制文件的平台上；当前列表包括`x86_64 aarch64 ppc64le amd64 win32`。对于其他平台，默认情况下不会安装 greenlet，这应该可以在不支持`greenlet`的平台上安装和运行 SQLAlchemy 1.4 的测试套件，但不包括任何 asyncio 功能。要在上述列表之外的机器架构上包含`greenlet`依赖项进行安装，可以通过运行`pip install sqlalchemy[asyncio]`并尝试安装`greenlet`。

    此外，测试套件已修复，以便在未安装 greenlet 时可以完全完成测试，并为与 asyncio 相关的测试适当跳过。

    参考：[#6136](https://www.sqlalchemy.org/trac/ticket/6136)

### orm

+   **[orm] [用例]**

    添加了新属性`Select.columns_clause_froms`，它将检索由`Select`语句的列子句隐含的 FROM 列表。这与旧的`Select.froms`集合不同，它不执行任何 ORM 编译步骤，这些步骤必要地去注释 FROM 元素并执行诸如计算 joinedloads 等操作，这使得它不适合于`Select.select_from()`方法。另外，还添加了一个新参数`Select.with_only_columns.maintain_column_froms`，该参数在替换列集合之前将此集合转移到`Select.select_from()`。

    此外，`Select.froms`被重命名为`Select.get_final_froms()`，以强调这个集合不是一个简单的访问器，而是根据对象的完整状态计算出来的，当在 ORM 上下文中使用时，这可能是一个昂贵的调用。

    另外修复了涉及`with_only_columns()`函数的一个回归问题，以支持对被替换为`Select.with_only_columns()`或`Query.with_entities()`的列元素应用条件，该问题在 1.4.19 中发布的[#6503](https://www.sqlalchemy.org/trac/ticket/6503)中发生了破坏。

    参考：[#6808](https://www.sqlalchemy.org/trac/ticket/6808)

+   **[orm] [bug] [sql]**

    修复了一个问题，即“克隆”的绑定参数对象会导致编译器中的名称冲突，如果在单个语句中同时使用了此参数的多个克隆体。这可能特别发生在像 ORM 单表继承查询中，其中在一个查询中多次指定了相同的“鉴别器”值。

    参考：[#6824](https://www.sqlalchemy.org/trac/ticket/6824)

+   **[orm] [bug]**

    修复了加载策略中使用`Load.options()`方法时的问题，特别是当嵌套多次调用时，会生成一个过长且更重要的非确定性缓存键，导致缓存键非常大，而且也不允许有效的缓存使用，无论是在总内存使用还是在缓存本身中使用的条目数方面。

    引用：[#6869](https://www.sqlalchemy.org/trac/ticket/6869)

+   **【orm】【bug】**

    修订了`ORMExecuteState.user_defined_options`访问器从上下文接收`UserDefinedOption`和相关选项对象的方式，特别强调了“selectinload”加载策略，以前此处未能正常工作；其他策略没有此问题。当前正在执行的查询相关联的对象，而不是缓存查询的对象，现在无条件地传播。这本质上将它们与“加载器策略”选项分开，这些选项明确与查询的编译状态关联，并且需要与缓存查询相关联使用。

    此修复的效果是，用户定义的选项，例如像 dogpile.caching 示例中使用的选项以及用于其他配方（如为水平共享扩展定义“分片 ID”）的选项，将正确传播到急切和懒惰的加载器，而不管最终是否调用了缓存的查询。

    引用：[#6887](https://www.sqlalchemy.org/trac/ticket/6887)

+   **【orm】【bug】**

    修复了工作单元在内部使用了 2.0 弃用的 SQL 表达式形式的问题，在启用 SQLALCHEMY_WARN_20 时会发出弃用警告。

    引用：[#6812](https://www.sqlalchemy.org/trac/ticket/6812)

+   **【orm】【bug】**

    修复了`selectinload()`中在嵌套超过一级的选项中使用新的`PropComparator.and_()`功能时无法更新绑定参数值的问题，这是 SQL 语句缓存的副作用。

    引用：[#6881](https://www.sqlalchemy.org/trac/ticket/6881)

+   **【orm】【bug】**

    调整了 ORM 加载器内部，不再使用在 1.4 中添加的“lambda 缓存”系统，同时修复了一个仍在使用先前“烘焙查询”系统的位置。lambda 缓存系统仍然是减少构建具有相对固定使用模式的查询开销的有效方式。在加载器策略的情况下，使用的查询负责通过大量的任意选项和条件，这些选项和条件由最终用户代码生成和有时消耗，使得 lambda 缓存概念不比不使用更有效，代价是更复杂。特别是通过内部删除此功能，可以使[#6881](https://www.sqlalchemy.org/trac/ticket/6881)和[#6887](https://www.sqlalchemy.org/trac/ticket/6887)中指出的问题变得更加简单。

    参考：[#6079](https://www.sqlalchemy.org/trac/ticket/6079), [#6889](https://www.sqlalchemy.org/trac/ticket/6889)

+   **[orm] [bug]**

    修复了`Bundle`构造未能创建正确的缓存键的问题，导致查询缓存的低效使用。这对“selectinload”策略产生了一定影响，并且被识别为[#6889](https://www.sqlalchemy.org/trac/ticket/6889)的一部分。

    参考：[#6889](https://www.sqlalchemy.org/trac/ticket/6889)

### sql

+   **[sql] [bug]**

    修复了`CTE`中的问题，在版本 1.4.21 / [#6752](https://www.sqlalchemy.org/trac/ticket/6752)中添加的新`HasCTE.add_cte()`方法未能正确处理“复合选择”结构，如`union()`，`union_all()`，`except()`等。感谢 Eric Masseran 提供的拉取请求。

    参考：[#6752](https://www.sqlalchemy.org/trac/ticket/6752)

+   **[sql] [bug]**

    修复了`CacheKey.to_offline_string()`方法中的问题，该方法由 dogpile.caching 示例使用，尝试从懒加载器生成的特殊“lambda”查询创建正确的缓存键时，未能包含参数值，导致缓存键不正确。

    参考：[#6858](https://www.sqlalchemy.org/trac/ticket/6858)

+   **[sql] [bug]**

    调整了“from linter”警告功能，以适应多于一级深度的连接链，其中 ON 子句不明确匹配目标，例如“ON TRUE”这样的表达式。这种用法旨在通过“a 到 b”的 JOIN 来取消笛卡尔积警告，但对于连接链具有多个元素的情况并不起作用。

    参考：[#6886](https://www.sqlalchemy.org/trac/ticket/6886)

+   **[sql] [错误]**

    修复了 lambda 缓存系统中的问题，其中查询的一个元素产生没有缓存键的情况，比如自定义选项或子句元素，仍然会不适当地填充“lambda 缓存”中的表达式。

### 模式

+   **[模式] [枚举]**

    统一 `Enum` 在本地和非本地实现中的行为，关于具有别名元素的枚举的接受值。当 `Enum.omit_aliases` 为 `False` 时，所有值，包括别名，都被接受为有效值。当 `Enum.omit_aliases` 为 `True` 时，只有非别名值被接受为有效值。

    参考：[#6146](https://www.sqlalchemy.org/trac/ticket/6146)

### mypy

+   **[mypy] [用例]**

    添加了对 SQLAlchemy 类在用户代码中使用“泛型类”语法进行定义的支持，如 `sqlalchemy2-stubs` 所定义的，例如 `Column[String]`，无需在 `TYPE_CHECKING` 块中对这些构造进行限定，通过实现 Python 特殊方法 `__class_getitem__()`，使得这种语法在运行时可以无错误通过。

    参考：[#6759](https://www.sqlalchemy.org/trac/ticket/6759)，[#6804](https://www.sqlalchemy.org/trac/ticket/6804)

### postgresql

+   **[postgresql] [错误]**

    为 PostgreSQL 的 “overlaps”、“contained_by”、“contains” 操作符添加了 “is_comparison” 标志，以便它们在相关的 ORM 上下文中以及与 “from linter” 功能一起使用时能够正常工作。

    参考：[#6886](https://www.sqlalchemy.org/trac/ticket/6886)

### mssql

+   **[mssql] [错误] [sql]**

    修复了 `literal_binds` 编译器标志中的问题，当与一类称为“literal_execute”的参数一起使用时，会导致无法正常工作，该参数涵盖了像 LIMIT 和 OFFSET 值这样的东西，对于驱动程序不允许绑定参数的方言，比如 SQL Server 的 “TOP” 子句。该问题似乎只影响 MSSQL 方言。

    参考：[#6863](https://www.sqlalchemy.org/trac/ticket/6863)

### 杂项

+   **[错误] [扩展]**

    修复了水平分片扩展无法正确适应传递给 `Session.execute()` 的纯文本 SQL 语句的问题。

    参考：[#6816](https://www.sqlalchemy.org/trac/ticket/6816)

### 一般

+   **[一般] [错误]**

    设置要求已修改，使 `greenlet` 仅在已知支持 `greenlet` 的平台且 pypi 上已有预构建的二进制文件的平台上成为默认要求；当前列表为 `x86_64 aarch64 ppc64le amd64 win32`。对于其他平台，默认情况下不会安装 greenlet，这应该使得 SQLAlchemy 1.4 可以在不支持 `greenlet` 的平台上进行安装和测试套件运行，但排除了任何 asyncio 功能。为了在上述列表之外的机器架构上包含 `greenlet` 依赖项进行安装，可以通过运行 `pip install sqlalchemy[asyncio]` 来包含 `[asyncio]` 附加项，然后它将尝试安装 `greenlet`。

    另外，测试套件已修复，以便在未安装 greenlet 时测试可以完全完成，并为 asyncio 相关的测试适当地跳过。

    参考：[#6136](https://www.sqlalchemy.org/trac/ticket/6136)

### orm

+   **[orm] [usecase]**

    添加了新属性 `Select.columns_clause_froms`，将检索由 `Select` 语句的列子句暗示的 FROM 列表。这与旧的 `Select.froms` 集合不同，它不执行任何 ORM 编译步骤，这些步骤必然会去除 FROM 元素的注释并执行诸如计算 joinedloads 等操作，这使得它不适合 `Select.select_from()` 方法的候选项。此外，还添加了一个新参数 `Select.with_only_columns.maintain_column_froms`，该参数在替换列集合之前将此集合转移到 `Select.select_from()`。

     `Select.froms` 被重命名为 `Select.get_final_froms()`，强调该集合不是简单的访问器，而是根据对象的完整状态进行计算的，当在 ORM 上下文中使用时可能是一个昂贵的调用。

    另外修复了涉及`with_only_columns()`函数的回归问题，以支持对使用`Select.with_only_columns()`或`Query.with_entities()`替换的列元素应用条件，这在 1.4.19 版本中作为[#6503](https://www.sqlalchemy.org/trac/ticket/6503)的一部分发布时出现了问题。

    参考：[#6808](https://www.sqlalchemy.org/trac/ticket/6808)

+   **[orm] [bug] [sql]**

    修复了绑定参数对象“克隆”会在单个语句中同时使用多个克隆参数时在编译器中引起名称冲突的问题。这可能会在特定情况下发生，例如 ORM 单表继承查询在一个查询中多次指示相同的“鉴别器”值。

    参考：[#6824](https://www.sqlalchemy.org/trac/ticket/6824)

+   **[orm] [bug]**

    修复了加载策略中使用`Load.options()`方法时的问题，特别是在嵌套多次调用时，会生成一个过长且更重要的非确定性缓存键，导致缓存键非常大，同时也不允许高效的缓存使用，无论是在总内存使用量还是在缓存本身中使用的条目数量方面。

    参考：[#6869](https://www.sqlalchemy.org/trac/ticket/6869)

+   **[orm] [bug]**

    修改了`ORMExecuteState.user_defined_options`访问器从上下文中接收`UserDefinedOption`和相关选项对象的方式，特别强调了“selectinload”加载策略中此前无法正常工作的问题；其他策略没有这个问题。与当前执行的查询相关联的对象，而不是缓存查询的对象，现在无条件地传播。这基本上将它们与“加载策略”选项分开，这些选项明确与查询的编译状态相关联，并且需要与缓存查询相关联使用。

    此修复的效果是，用户定义的选项，例如 dogpile.caching 示例中使用的选项以及用于其他配方（例如为水平共享扩展定义“分片 ID”）的选项，将正确传播到急切加载器和延迟加载器，无论最终是否调用了缓存查询。

    参考：[#6887](https://www.sqlalchemy.org/trac/ticket/6887)

+   **[orm] [bug]**

    修复了工作单元内部使用 2.0 弃用的 SQL 表达式形式的问题，当启用 SQLALCHEMY_WARN_20 时会发出弃用警告。

    参考：[#6812](https://www.sqlalchemy.org/trac/ticket/6812)

+   **[orm] [bug]**

    修复了`selectinload()`中的问题，使用嵌套超过一层深的选项内部使用新的`PropComparator.and_()`功能将无法更新嵌套条件中的绑定参数值，这是 SQL 语句缓存的副作用。

    参考：[#6881](https://www.sqlalchemy.org/trac/ticket/6881)

+   **[orm] [bug]**

    调整了 ORM 加载器内部，不再使用在 1.4 版本中添加的“lambda 缓存”系统，同时修复了一个仍在使用先前“烘烤查询”系统的位置。Lambda 缓存系统仍然是减少构建具有相对固定使用模式的查询开销的有效方式。在加载器策略的情况下，使用的查询负责移动通过大量任意选项和条件，这些选项和条件由最终用户代码生成并有时消耗，使得 lambda 缓存概念不比不使用更有效，但代价是更复杂。特别是通过内部删除此功能，可以使[#6881](https://www.sqlalchemy.org/trac/ticket/6881)和[#6887](https://www.sqlalchemy.org/trac/ticket/6887)中指出的问题变得不那么复杂。

    参考：[#6079](https://www.sqlalchemy.org/trac/ticket/6079)，[#6889](https://www.sqlalchemy.org/trac/ticket/6889)

+   **[orm] [bug]**

    修复了`Bundle`构造未能创建正确缓存键的问题，导致查询缓存的低效使用。这对“selectinload”策略产生了一定影响，并且被识别为[#6889](https://www.sqlalchemy.org/trac/ticket/6889)的一部分。

    参考：[#6889](https://www.sqlalchemy.org/trac/ticket/6889)

### sql

+   **[sql] [bug]**

    修复了`CTE`中的问题，在 1.4.21 版本/ [#6752](https://www.sqlalchemy.org/trac/ticket/6752)中添加的新`HasCTE.add_cte()`方法未能正确处理“复合选择”结构，如`union()`，`union_all()`，`except()`等。感谢 Eric Masseran 的拉取请求。

    参考：[#6752](https://www.sqlalchemy.org/trac/ticket/6752)

+   **[sql] [bug]**

    修复了`CacheKey.to_offline_string()`方法中的问题，该方法由 dogpile.caching 示例使用，尝试从懒惰加载器生成的特殊“lambda”查询创建正确的缓存键时，将无法包含参数值，导致缓存键不正确。

    参考文献：[#6858](https://www.sqlalchemy.org/trac/ticket/6858)

+   **[sql] [bug]**

    调整了“from linter”警告功能，以适应一个级别深度超过一个的连接链，其中 ON 子句没有明确匹配目标，比如“ON TRUE”这样的表达式。这种用法旨在通过“a 到 b”的连接取消笛卡尔积警告，这在连接链包含多个元素的情况下并不起作用。

    参考文献：[#6886](https://www.sqlalchemy.org/trac/ticket/6886)

+   **[sql] [bug]**

    修复了 lambda 缓存系统中的问题，其中一个查询的元素（例如自定义选项或子句元素）产生了没有缓存键的表达式，仍然不合适地填充了“lambda 缓存”中的表达式。

### 模式

+   **[schema] [enum]**

    统一行为`Enum`在本机和非本机实现中关于具有别名元素的枚举的接受值。当`Enum.omit_aliases`为`False`时，所有值，包括别名，都被接受为有效值。当`Enum.omit_aliases`为`True`时，只有非别名值被接受为有效值。

    参考文献：[#6146](https://www.sqlalchemy.org/trac/ticket/6146)

### mypy

+   **[mypy] [usecase]**

    添加了对 SQLAlchemy 类使用“通用类”语法的支持，如`sqlalchemy2-stubs`所定义的，例如`Column[String]`，无需在`TYPE_CHECKING`块内限定这些构造，通过实现 Python 特殊方法`__class_getitem__()`，允许此语法在运行时无错误地通过。

    参考文献：[#6759](https://www.sqlalchemy.org/trac/ticket/6759)，[#6804](https://www.sqlalchemy.org/trac/ticket/6804)

### postgresql

+   **[postgresql] [bug]**

    在 PostgreSQL 的“overlaps”、“contained_by”、“contains”运算符中添加了“is_comparison”标志，以便它们在相关的 ORM 上下文中以及与“from linter”功能结合使用时起作用。

    参考文献：[#6886](https://www.sqlalchemy.org/trac/ticket/6886)

### mssql

+   **[mssql] [bug] [sql]**

    修复了`literal_binds`编译器标志的问题，该标志用于在外部呈现绑定参数时内联绑定参数失败，当与“literal_execute”这类参数的某些类一起使用时，即使限制了像 SQL Server 的“TOP”子句这样的驱动程序不允许绑定参数的参数，也无法正常工作。该问题在本地似乎仅影响 MSSQL 方言。

    参考文献：[#6863](https://www.sqlalchemy.org/trac/ticket/6863)

### 杂项

+   **[bug] [ext]**

    修复了水平分片扩展无法正确处理传递给`Session.execute()`的纯文本 SQL 语句的问题。

    参考：[#6816](https://www.sqlalchemy.org/trac/ticket/6816)

## 1.4.22

发布日期：2021 年 7 月 21 日

### orm

+   **[orm] [bug]**

    修复了新的`Table.table_valued()`方法中的问题，导致生成的`TableValuedColumn`构造在 ORM 中无法正确响应别名适配，例如用于急加载、多态加载等。

    参考：[#6775](https://www.sqlalchemy.org/trac/ticket/6775)

+   **[orm] [bug]**

    修复了在使用带有不可哈希类型的列表达式的 ORM 结果（例如使用非元组的`JSON`或`ARRAY`）时，使用`Result.unique()`方法会静默回退到使用`id()`函数而不是引发错误的问题。现在，在 2.0 风格的 ORM 查询中使用`Result.unique()`方法时会引发错误。此外，对于未知类型的结果值，例如在使用未知返回类型的 SQL 函数时经常发生的情况，假定可哈希性为 True；如果值确实不可哈希，则`hash()`本身会引发错误。

    对于传统的 ORM 查询，由于传统的`Query`对象在所有情况下都进行唯一化处理，旧规则仍然有效，即对于未知类型的结果值使用`id()`，因为这种传统的唯一化主要用于唯一化 ORM 实体而不是列值。

    参考：[#6769](https://www.sqlalchemy.org/trac/ticket/6769)

+   **[orm] [bug]**

    修复了在诸如测试套件拆��期间清除映射器可能导致“字典大小更改”警告的问题，这是由于弱引用字典的迭代。已应用`list()`以防止并发 GC 影响此操作。

    参考：[#6771](https://www.sqlalchemy.org/trac/ticket/6771)

+   **[orm] [bug] [regression]**

    修复了一个关键的缓存问题，当 ORM 的持久性功能使用 INSERT..RETURNING 时，混合“批量保存”和标准“flush”形式的 INSERT 会导致缓存一个不正确的查询。

    参考：[#6793](https://www.sqlalchemy.org/trac/ticket/6793)

### engine

+   **[engine] [bug]**

    添加了一些防范措施，以应对事件系统中可能发生的`KeyError`，以适应解释器在调用`Engine.dispose()`时同时关闭的情况，这可能会导致堆栈跟踪警告。

    参考：[#6740](https://www.sqlalchemy.org/trac/ticket/6740)

### sql

+   **[sql] [bug]**

    修复了使用`case.whens`参数以字典位置方式而不是关键字参数传递会发出 2.0 弃用警告的问题，指的是位置传递列表的弃用。仍支持以字典格式传递的“whens”，位置传递，这是一个意外标记为弃用的格式。

    参考：[#6786](https://www.sqlalchemy.org/trac/ticket/6786)

+   **[sql] [bug]**

    修复了在使用 Python 的`None`值时，类型特定的绑定参数处理程序不会被调用的问题；特别是在使用`JSON`数据类型以及相关的 PostgreSQL 特定类型如`JSONB`时，会无法将 Python 的`None`值编码为 JSON null，然而该问题泛化为任何绑定参数处理程序与此特定的`Insert`方法结合使用时会出现问题。

    参考：[#6770](https://www.sqlalchemy.org/trac/ticket/6770)

### orm

+   **[orm] [bug]**

    修复了新的`Table.table_valued()`方法中生成的`TableValuedColumn`构造不会正确响应别名适配的问题，这在整个 ORM 中被用于急加载、多态加载等。

    参考：[#6775](https://www.sqlalchemy.org/trac/ticket/6775)

+   **[orm] [bug]**

    修复了在 ORM 结果中使用具有不可哈希类型的列表达式，例如使用非元组的`JSON`或`ARRAY`时，使用`Result.unique()`方法会静默回退到使用`id()`函数而不是引发错误的问题。现在，在使用 2.0 样式 ORM 查询时，当使用`Result.unique()`方法时会引发错误。此外，假定未知类型的结果值是可哈希的，例如在使用未知返回类型的 SQL 函数时经常发生；如果值确实不可哈希，则`hash()`本身会引发错误。

    对于传统的 ORM 查询，由于传统的`Query`对象在所有情况下都是唯一的，旧规则仍然有效，即对于未知类型的结果值使用`id()`，因为这种传统的唯一性主要用于 ORM 实体而不是列值的唯一性。

    参考：[#6769](https://www.sqlalchemy.org/trac/ticket/6769)

+   **[orm] [bug]**

    修复了在清除映射器时（例如测试套件拆卸）可能导致“字典大小更改”警告的问题，这是由于弱引用字典的迭代导致垃圾回收期间的并发 GC 影响此操作。已应用`list()`以防止并发 GC 影响此操作。

    参考：[#6771](https://www.sqlalchemy.org/trac/ticket/6771)

+   **[orm] [bug] [regression]**

    修复了 ORM 的持久性功能在使用 INSERT..RETURNING 时会缓存不正确查询的严重缓存问题，当混合“批量保存”和标准“flush”形式的 INSERT 时会出现此问题。

    参考：[#6793](https://www.sqlalchemy.org/trac/ticket/6793)

### engine

+   **[engine] [bug]**

    添加了一些防范措施，以防止事件系统中的`KeyError`，以适应解释器在调用`Engine.dispose()`时同时关闭的情况，这将导致堆栈跟踪警告。

    参考：[#6740](https://www.sqlalchemy.org/trac/ticket/6740)

### sql

+   **[sql] [bug]**

    修复了使用`case.whens`参数将字典位置传递而不是作为关键字参数会发出 2.0 弃用警告的问题，该警告指的是弃用位置传递列表。仍支持以位置方式传递的“whens”字典格式，并且意外地标记为弃用。

    参考：[#6786](https://www.sqlalchemy.org/trac/ticket/6786)

+   **[sql] [bug]**

    修复了在使用 Python 的`None`值与`Insert.values()`方法时，类型特定的绑定参数处理程序不会被调用的问题；特别是在使用`JSON`数据类型以及相关的 PostgreSQL 特定类型（如`JSONB`）时，将无法将 Python 的`None`值编码为 JSON null，但是该问题泛化为与此特定方法的`Insert`结合使用的任何绑定参数处理程序。

    参考：[#6770](https://www.sqlalchemy.org/trac/ticket/6770)

## 1.4.21

发布日期：2021 年 7 月 14 日

### orm

+   **[orm] [usecase]**

    修改了用于跟踪标量对象关系历史的方法，这些关系不是多对一，即本来是一对多关系的一对一关系。当替换一对一值时，将被替换的“旧”值不再立即加载，而是在刷新过程中处理。这消除了在分配给一对一属性时经常发生的历史上令人困扰的延迟加载，特别是在使用“lazy='raise'”以及异步 IO 使用情况下。

    此更改确实会导致`AttributeEvents.set()`事件中的行为变化，尽管目前已经记录在案，即如果未加载“旧”参数且未设置`relationship.active_history`标志，则此类一对一属性应用的事件将不再接收“旧”参数。如`AttributeEvents.set()`中所记录的，如果事件处理程序在事件触发时需要接收“旧”值，则必须在事件监听器或关系中建立 active_history 标志。这已经是其他类型属性（如多对一和列值引用）的行为。

    此更改还将推迟在较少见的情况下更新“旧”值上的反向引用，即“旧”值在会话中本地存在，但在相关关系中未加载，直到下一次刷新发生。如果这造成问题，可以再次在关系上设置正常的`relationship.active_history`标志为`True`。

    参考：[#6708](https://www.sqlalchemy.org/trac/ticket/6708)

+   **[orm] [bug] [regression]**

    由于[#6503](https://www.sqlalchemy.org/trac/ticket/6503)引起的 1.4.19 版本中的回归问题已得到修复，涉及`Query.with_entities()`的相关问题，新结构在使用`Query.union()`等集合操作时会不适当地传递给封闭的`Query`，导致 JOIN 指令也被应用到外部查询中。

    参考：[#6698](https://www.sqlalchemy.org/trac/ticket/6698)

+   **[orm] [bug] [regression]**

    修复了在 1.4.3 版本中出现的回归，原因是[#6060](https://www.sqlalchemy.org/trac/ticket/6060)，其中限制 ORM 对派生可选择性的适应的规则干扰了其他基于 ORM 适应的情况，在这种情况下，当对使用了`column_property()`的映射应用`with_polymorphic()`的适应时，后者又使用了一个包含映射表的`aliased()`对象的标量选择时。

    参考：[#6762](https://www.sqlalchemy.org/trac/ticket/6762)

+   **[orm] [regression]**

    修复了 ORM 回归，其中为混合属性和潜在的其他类似类型的 ORM 启用的表达式生成的临时标签名称通常会通过子查询向外传播，即使从子查询中选择，该名称也会在结果集的最终键中保留。在这种情况下现在跟踪了额外的状态，当一个混合被选中并从一个 Core select / 子查询中选择时，该状态不会丢失。

    参考：[#6718](https://www.sqlalchemy.org/trac/ticket/6718)

### sql

+   **[sql] [usecase]**

    向每个`select()`，`insert()`，`update()` 和 `delete()` 构造添加了新方法`HasCTE.add_cte()`。该方法将给定的`CTE`作为主语句的“独立”CTE 添加到语句中，这意味着它无条件地在主语句上方的 WITH 子句中呈现，即使在主语句中没有引用它。这在 PostgreSQL 数据库中是一个常见用例，其中 CTE 用于独立于主语句运行的 DML 语句。

    参考：[#6752](https://www.sqlalchemy.org/trac/ticket/6752)

+   **[sql] [bug]**

    修复了 CTE 构造中的问题，其中引用了一个具有重复列名称的 SELECT 的递归 CTE，通常在 1.4 中使用标签逻辑对列名称进行去重，但是在 WITH 子句中无法正确引用去重后的标签名称。

    参考：[#6710](https://www.sqlalchemy.org/trac/ticket/6710)

+   **[sql] [bug] [regression]**

    修复了一个回归问题，即当给定一个嵌入在 SQL 函数中的浮点采样值时，`tablesample()` 构造无法执行。

    参考：[#6735](https://www.sqlalchemy.org/trac/ticket/6735)

### postgresql

+   **[postgresql] [错误]**

    修复了`Insert.on_conflict_do_nothing()`和`Insert.on_conflict_do_update()`中的问题，其中作为`constraint`参数传递的唯一约束的名称如果基于生成 PostgreSQL 最大标识符长度为 63 个字符的命名约定，则不会被正确截断长度，与在 CREATE TABLE 语句中发生的方式相同。

    参考：[#6755](https://www.sqlalchemy.org/trac/ticket/6755)

+   **[postgresql] [错误]**

    修复了一个问题，即当 `ENUM` 数据类型嵌入在 `ARRAY` 数据类型中时，在使用`schema_translate_map`功能时，当创建/删除时，将无法正确发出。另外，修复了一个相关问题，即当`schema_translate_map`功能与 `ENUM` 数据类型结合使用时，与 `CAST` 也是如何在 PostgreSQL 方言上的 `ARRAY(ENUM)` 组合中工作的。

    参考：[#6739](https://www.sqlalchemy.org/trac/ticket/6739)

+   **[postgresql] [错误]**

    修复了一个问题，在`Insert.on_conflict_do_nothing()`和`Insert.on_conflict_do_update()`中，如果唯一约束的名称包含需要引用的字符，则不会正确引用。

    参考：[#6696](https://www.sqlalchemy.org/trac/ticket/6696)

### mssql

+   **[mssql] [错误] [回归]**

    修复了一个回归问题，即 SQL Server 方言的特殊点模式名称处理如果在`schema_translate_map`功能中使用点模式名称，则不会正确工作。

    参考：[#6697](https://www.sqlalchemy.org/trac/ticket/6697)

### orm

+   **[orm] [用例]**

    修改了用于标量对象关系的历史跟踪的方法，这些关系不是多对一，即本应是一对多的一对一关系。当替换一对一值时，将被替换的“旧”值不再立即加载，而是在刷新过程中处理。这消除了在分配给一对一属性时通常发生的历史上令人困扰的延迟加载，特别是在使用“lazy='raise'”以及异步 IO 用例时尤为麻烦。

    此更改确实会导致`AttributeEvents.set()`事件中的行为变化，尽管目前已经记录，即如果一个一对一属性未加载且未设置`relationship.active_history`标志，则该事件将不再接收“旧”参数。正如在`AttributeEvents.set()`中记录的那样，如果事件处理程序在事件触发时需要接收“旧”值，则必须在事件监听器或关系中建立 active_history 标志。这已经是其他类型属性（如多对一和列值引用）的行为。

    此更改还将推迟在较少见的情况下更新“旧”值上的反向引用，即使“旧”值在会话中本地存在，但在相关关系中未加载，直到下一次刷新发生。如果这造成问题，可以在关系上再次设置正常的`relationship.active_history`标志为`True`。

    参考：[#6708](https://www.sqlalchemy.org/trac/ticket/6708)

+   **[orm] [bug] [regression]**

    修复了 1.4.19 中由于[#6503](https://www.sqlalchemy.org/trac/ticket/6503)和相关问题导致的回归，涉及`Query.with_entities()`的新结构在使用诸如`Query.union()`等集合操作时不当地传递给外部`Query`，导致内部的 JOIN 指令也被应用到外部查询中。

    参考：[#6698](https://www.sqlalchemy.org/trac/ticket/6698)

+   **[orm] [bug] [regression]**

    修复了 1.4.3 版本中出现的由于[#6060](https://www.sqlalchemy.org/trac/ticket/6060)导致的回归问题，其中限制 ORM 对派生可选择性适应的规则干扰了其他基于 ORM 适应的情况，例如当对使用`column_property()`的映射应用`with_polymorphic()`适应时，该映射又使用了包含映射表的`aliased()`对象的标量选择时。

    参考：[#6762](https://www.sqlalchemy.org/trac/ticket/6762)

+   **[orm] [regression]**

    修复了 ORM 回归问题，其中为混合属性和潜在的其他类似类型的 ORM 启用表达式生成的临时标签名称通常会通过子查询传播，即使在从子查询中选择时，该名称也会保留在结果集的最终键中。在这种情况下现在会跟踪额外的状态，当从核心选择/子查询中选择混合时不会丢失。

    参考：[#6718](https://www.sqlalchemy.org/trac/ticket/6718)

### sql

+   **[sql] [usecase]**

    为每个`select()`，`insert()`，`update()`和`delete()`构造添加了新方法`HasCTE.add_cte()`。此方法将给定的`CTE`作为语句的“独立”CTE 添加，这意味着它无条件地在 WITH 子句中呈现在语句之上，即使它在主语句中没有被引用。这是在 PostgreSQL 数据库上的一个常见用例，其中 CTE 用于独立于主语句运行的 DML 语句。

    参考：[#6752](https://www.sqlalchemy.org/trac/ticket/6752)

+   **[sql] [bug]**

    修复了 CTE 结构中的问题，其中递归 CTE 引用了具有重复列名的 SELECT，通常使用 1.4 中的标签逻辑进行去重，将无法在 WITH 子句中正确引用去重后的标签名称。

    参考：[#6710](https://www.sqlalchemy.org/trac/ticket/6710)

+   **[sql] [bug] [regression]**

    修复了一个回归问题，即当构造时给定一个不嵌入在 SQL 函数中的浮点采样值时，`tablesample()` 构造将无法执行。

    参考文献：[#6735](https://www.sqlalchemy.org/trac/ticket/6735)

### postgresql

+   **[postgresql] [bug]**

    修复了`Insert.on_conflict_do_nothing()`和`Insert.on_conflict_do_update()`中的问题，在这个问题中，作为`constraint`参数传递的唯一约束的名称如果基于生成的名称约定，生成的名称超过了 PostgreSQL 最大标识符长度 63 个字符，那么将无法正确截断，就像在 CREATE TABLE 语句中发生的一样。

    参考文献：[#6755](https://www.sqlalchemy.org/trac/ticket/6755)

+   **[postgresql] [bug]**

    修复了一个问题，在这个问题中，如果 `schema_translate_map` 功能也在使用中，则嵌入在 `ARRAY` 数据类型中的 PostgreSQL `ENUM` 数据类型在 create/drop 中将无法正确发出。此外，修复了一个相关问题，即相同的 `schema_translate_map` 功能将无法与 `CAST` 结合使用，这也是 `ARRAY(ENUM)` 组合在 PostgreSQL 方言上的工作方式的一部分。

    参考文献：[#6739](https://www.sqlalchemy.org/trac/ticket/6739)

+   **[postgresql] [bug]**

    修复了`Insert.on_conflict_do_nothing()`和`Insert.on_conflict_do_update()`中的问题，其中作为`constraint`参数传递的唯一约束的名称，如果包含需要引用的字符，则不会被正确地引用。

    参考文献：[#6696](https://www.sqlalchemy.org/trac/ticket/6696)

### mssql

+   **[mssql] [bug] [regression]**

    修复了一个回归问题，即如果在 `schema_translate_map` 功能中使用了点分模式名称，则 SQL Server 方言的特殊点分模式名称处理将无法正确工作。

    参考文献：[#6697](https://www.sqlalchemy.org/trac/ticket/6697)

## 1.4.20

发布日期：2021 年 6 月 28 日

### orm

+   **[orm] [bug] [regression]**

    修复了 ORM 中的回归问题，涉及`with_polymorphic()`构造的内部重构步骤，当用户界面对象在查询处理过程中被垃圾回收时。重构未确保处理“多态”情况下的子实体，导致`AttributeError`。

    参考：[#6680](https://www.sqlalchemy.org/trac/ticket/6680)

+   **[orm] [bug] [regression]**

    调整了`Query.union()`等类似集合操作，以正确兼容最新添加的功能，适用于 SQLAlchemy 1.4.19 版本。现在，UNION 或其他集合操作中呈现的 SELECT 语句将包括直接映射为延迟加载的列；这不仅修复了涉及多层嵌套联合的回归问题，导致列不匹配，还允许在这样一个`Query`的顶层使用`undefer()`选项，而无需将该选项应用于 UNION 中的每个元素。

    参考：[#6678](https://www.sqlalchemy.org/trac/ticket/6678)

+   **[orm] [bug]**

    调整了映射器中对可调用对象的检查，该对象用作`@validates`验证器函数或`@reconstructor`重构函数，更宽松地检查“callable”，以适应基本属性（如`__func__`和`__call__`）为基础的对象，而不是测试`MethodType` / `FunctionType`，从而使像 cython 函数这样的东西能够正常工作。感谢 Miłosz Stypiński 提供的拉取请求。

    参考：[#6538](https://www.sqlalchemy.org/trac/ticket/6538)

### engine

+   **[engine] [bug]**

    修复了 C 扩展中`Row`类的问题，可能导致内存泄漏，即`Row`对象引用了一个 ORM 对象，然后该 ORM 对象被改变以再次引用`Row`本身，从而创建一个循环。为了解决这个问题，Python C API 已添加到本地`Row`实现中，用于跟踪 GC 循环。

    参考：[#5348](https://www.sqlalchemy.org/trac/ticket/5348)

+   **[engine] [bug]**

    修复了旧问题，即针对标记“*”进行的`select()`，然后产生一个列，未能正确将`cursor.description`列名组织到结果对象的键中。

    引用：[#6665](https://www.sqlalchemy.org/trac/ticket/6665)

### sql

+   **[sql] [用例]**

    向`PickleType`构造函数添加了一个 impl 参数，允许任何任意类型用于替代默认实现的`LargeBinary`。拉取请求由 jason3gb 提供。

    引用：[#6646](https://www.sqlalchemy.org/trac/ticket/6646)

+   **[sql] [错误] [orm]**

    为`Sequence`和更一般的`DefaultGenerator`基类修复了类层次结构，因为它们作为语句是“可执行的”，所以它们的层次结构中需要包含`Executable`，而不仅仅是之前随意应用于`Sequence`的`StatementRole`。此修复允许`Sequence`在所有的`.execute()`方法中工作，包括与`Session.execute()`一起使用，这在同时建立了`SessionEvents.do_orm_execute()`处理程序的情况下是不起作用的。

    引用：[#6668](https://www.sqlalchemy.org/trac/ticket/6668)

### 模式

+   **[模式] [错误]**

    修复了将`None`作为`Table.prefixes`值传递时不会存储空列表，而是常量`None`的问题，这可能会让第三方方言感到意外。该问题是由于 Alembic 的最新版本在此值上传递`None`而暴露出来的。拉取请求由 Kai Mueller 提供。

    引用：[#6685](https://www.sqlalchemy.org/trac/ticket/6685)

### mysql

+   **[mysql] [用例]**

    在 MySQL 方言的表反射功能中进行了小的调整，以适应诸如 TiDB 之类的备选 MySQL 导向数据库，这些数据库在“CREATE TABLE”内部的约束指令的末尾包括自己的“注释”指令，其中格式不包括注释之后的额外空格字符，在这种情况下，TiDB 的“集群索引”功能。拉取请求由 Daniël van Eeden 提供。

    引用：[#6659](https://www.sqlalchemy.org/trac/ticket/6659)

### 杂项

+   **[错误] [扩展] [回归]**

    修复了`sqlalchemy.ext.automap`扩展中的回归问题，使得显式映射到也是`relationship.secondary`元素的表的用例（该表也是由 automap 生成的`relationship()`的一部分）会触发在 1.4 版本中引入的“重叠”警告，并在 relationship X will copy column Q to column P, which conflicts with relationship(s): ‘Y’中讨论。虽然从 automap 生成这种情况仍然受到“重叠”警告中提到的相同注意事项的约束，但由于 automap 主要用于更为临时的用例，当生成具有这种特定模式的多对多关系时，触发警告的条件会被禁用。

    参考文献：[#6679](https://www.sqlalchemy.org/trac/ticket/6679)

### orm

+   **[orm] [bug] [regression]**

    修复了 ORM 中的回归问题，涉及`with_polymorphic()`构造的内部重构步骤，当用户可见对象在查询处理过程中被垃圾回收时。重构未确保处理“多态”情况下的子实体，导致`AttributeError`。

    参考文献：[#6680](https://www.sqlalchemy.org/trac/ticket/6680)

+   **[orm] [bug] [regression]**

    调整了`Query.union()`和类似的集合操作，以正确兼容刚刚在[#6661](https://www.sqlalchemy.org/trac/ticket/6661)中添加的新功能，使用 SQLAlchemy 1.4.19，这样，作为 UNION 或其他集合操作的元素呈现的 SELECT 语句将包括作为延迟映射的直接映射列；这既修复了涉及多层嵌套的联合会产生列不匹配的回归问题，也允许在这样的`Query`的顶层使用`undefer()`选项，而无需将该选项应用于 UNION 中的每个元素。

    参考文献：[#6678](https://www.sqlalchemy.org/trac/ticket/6678)

+   **[orm] [bug]**

    调整了映射器中对可调用对象的检查，该对象用作`@validates`验证器函数或`@reconstructor`重构函数，以更宽松地检查“callable”，以适应基本属性如`__func__`和`__call__`的对象，而不是测试`MethodType` / `FunctionType`，从而使像 cython 函数这样的东西能够正常工作。感谢 Miłosz Stypiński 提供的拉取请求。

    参考：[#6538](https://www.sqlalchemy.org/trac/ticket/6538)

### engine

+   **[engine] [bug]**

    修复了`Row`类的 C 扩展中的问题，可能导致在罕见情况下，一个引用了 ORM 对象的`Row`对象被突变为引用回`Row`本身，从而创建一个循环引用导致内存泄漏的问题。为了适应这种情况，已将用于跟踪 GC 循环的 Python C API 添加到本机`Row`实现中。

    参考：[#5348](https://www.sqlalchemy.org/trac/ticket/5348)

+   **[engine] [bug]**

    修复了针对标记“*”进行的`select()`查询的旧问题，该查询返回确切的一列，但未能正确将`cursor.description`列名组织到结果对象的键中。

    参考：[#6665](https://www.sqlalchemy.org/trac/ticket/6665)

### sql

+   **[sql] [usecase]**

    在`PickleType`构造函数中添加了一个 impl 参数，允许使用任意类型来替代`LargeBinary`的默认实现。感谢 jason3gb 提供的拉取请求。

    参考：[#6646](https://www.sqlalchemy.org/trac/ticket/6646)

+   **[sql] [bug] [orm]**

    修复了`Sequence`和更一般的`DefaultGenerator`基类的类层次结构，因为它们作为语句是“可执行的”，所以它们需要在其层次结构中包含`Executable`，而不仅仅是像以前随意应用于`Sequence`的`StatementRole`。修复允许`Sequence`在所有`.execute()`方法中工作，包括与`Session.execute()`一起使用，这在建立`SessionEvents.do_orm_execute()`处理程序的情况下不起作用。

    参考：[#6668](https://www.sqlalchemy.org/trac/ticket/6668)

### 模式

+   **[schema] [bug]**

    修复了将`None`传递给`Table.prefixes`值时不会存储空列表，而是常量`None`的问题，这可能会让第三方方言感到意外。这个问题是由 Alembic 的最新版本中传递`None`值引起的。感谢 Kai Mueller 的拉取请求。

    参考：[#6685](https://www.sqlalchemy.org/trac/ticket/6685)

### mysql

+   **[mysql] [usecase]**

    对 MySQL 方言的表反射功能进行了小的调整，以适应诸如 TiDB 之类的备选 MySQL 导向数据库，这些数据库在“CREATE TABLE”中的约束指令末尾包含自己的“comment”指令，其中格式在注释后没有额外的空格字符，在这种情况下是 TiDB 的“clustered index”功能。感谢 Daniël van Eeden 的拉取请求。

    参考：[#6659](https://www.sqlalchemy.org/trac/ticket/6659)

### 杂项

+   **[bug] [ext] [regression]**

    修复了`sqlalchemy.ext.automap`扩展中的回归问题，即创建一个显式映射类到一个表，同时该表也是`relationship.secondary`元素的用例，会触发 1.4 版本引入的“重叠”警告，并在 relationship X will copy column Q to column P, which conflicts with relationship(s): ‘Y’中讨论。尽管从 automap 生成此用例仍然受到“重叠”警告中提到的相同注意事项的限制，但由于 automap 主要用于更为临时的用例，当生成具有这种特定模式的多对多关系时，触发警告的条件将被禁用。

    参考：[#6679](https://www.sqlalchemy.org/trac/ticket/6679)

## 1.4.19

发布日期：2021 年 6 月 22 日

### orm

+   **[orm] [bug] [regression]**

    在与[#6052](https://www.sqlalchemy.org/trac/ticket/6052)相同领域进一步修复了回归问题，即加载器选项以及像`Query.join()`这样的方法调用，如果语句的左侧被替换为使用`Query.with_entities()`方法，或者在使用`Select.with_only_columns()`方法时使用 2.0 风格查询时，将会失败。已向对象添加了一组新的状态，用于跟踪选项/连接所依赖的“左侧”实体，当主实体发生变化时进行了备忘。

    参考：[#6253](https://www.sqlalchemy.org/trac/ticket/6253), [#6503](https://www.sqlalchemy.org/trac/ticket/6503)

+   **[orm] [bug]**

    优化了 ORM 子查询在延迟列和列属性方面的呈现行为，以使其更兼容 1.3 版本，同时还提供了 1.4 版本的新功能。由于 1.4 版本中的子查询不使用加载器选项，包括 `undefer()`，因此针对具有延迟属性的 ORM 实体的子查询现在将呈现那些直接引用映射表列的延迟属性，因为如果外部 SELECT 使用了这些列，则这些列是需要的；但是，如果延迟属性引用了我们通常使用的组合 SQL 表达式，例如 `column_property()`，则不会成为子查询的一部分，因为如果在子查询中需要，这些可以明确选择。如果实体是从此子查询中选择的，则列表达式仍然可以在派生子查询列的“外部”呈现。这基本上产生了与在 1.3 版本中工作时相同的行为。但是在这种情况下，修复还必须确保 ORM 启用的 `select()` 的 `.selected_columns` 集合也遵循这些规则，这特别允许递归 CTE 在这种情况下正确呈现，以前由于此问题未能正确呈现。

    参考：[#6661](https://www.sqlalchemy.org/trac/ticket/6661)

### sql

+   **[sql] [bug]**

    修复了与 ORM 使用情况相关的 CTE 构造中的问题，在这些情况下，针对“匿名”标签（例如在 ORM `column_property()` 映射中看到的标签）的递归 CTE 会在 `WITH RECURSIVE xyz(...)` 部分中呈现为其原始内部标签，而不是一个干净的匿名化名称。

    参考：[#6663](https://www.sqlalchemy.org/trac/ticket/6663)

### mypy

+   **[mypy] [bug]**

    修复了 mypy 插件中的问题，在缓存的 mypy 通行证上，一个自定义声明基类的类信息将不会被正确处理，导致引发 AssertionError。

    参考：[#6476](https://www.sqlalchemy.org/trac/ticket/6476)

### asyncio

+   **[asyncio] [usecase]**

    实现了 `async_scoped_session` 以解决 `scoped_session` 和 `AsyncSession` 之间的一些与 asyncio 相关的不兼容性，其中一些方法（特别是 `async_scoped_session.remove()` 方法）应该使用 `await` 关键字。

    另请参阅

    使用 asyncio scoped session

    参考：[#6583](https://www.sqlalchemy.org/trac/ticket/6583)

+   **[asyncio] [bug] [postgresql]**

    修复了 asyncio 实现中的 bug，在那里绿色线程适配系统未能传播 `BaseException` 子类，尤其是包括 `asyncio.CancelledError`，到引擎用于使连接无效并清理连接时使用的异常处理逻辑，从而阻止在任务被取消时正确处理连接的 bug。

    参考：[#6652](https://www.sqlalchemy.org/trac/ticket/6652)

### postgresql

+   **[postgresql] [bug] [oracle]**

    修复了在 PostgreSQL 和 Oracle 上使用 `INTERVAL` 数据类型与 `timedelta()` 对象进行比较操作时会产生 `AttributeError` 的问题。感谢 MajorDallas 的拉取请求。

    参考：[#6649](https://www.sqlalchemy.org/trac/ticket/6649)

+   **[postgresql] [bug]**

    修复了池“预先 ping”功能会隐式启动事务的问题，这会干扰自定义事务标志，例如在与 psycopg2 驱动程序一起使用时，与 PostgreSQL 的“只读”模式。

    参考：[#6621](https://www.sqlalchemy.org/trac/ticket/6621)

### mysql

+   **[mysql] [用例]**

    添加了新的构造 `match`，提供了 MySQL 的 MATCH 运算符的全范围支持，包括多列支持和修饰符。感谢 Anton Kovalevich 的拉取请求。

    另请参阅

    `match`

    参考：[#6132](https://www.sqlalchemy.org/trac/ticket/6132)

### mssql

+   **[mssql] [更改]**

    对 pymssql 方言使用的服务器版本正则表达式进行了改进，以防止在无效版本字符串的情况下发生正则表达式溢出。

    参考：[#6253](https://www.sqlalchemy.org/trac/ticket/6253)，[#6503](https://www.sqlalchemy.org/trac/ticket/6503)

+   **[mssql] [bug]**

    修复了“schema_translate_map”功能在与具有 IDENTITY 列的表进行插入时无法正确运行的 bug，其中 IDENTITY 列的值在插入的值中指定，从而触发 SQLAlchemy 设置 IDENTITY INSERT 为“on”的功能；正是在这个指令中，schema translate map 无法被遵守。

    参考：[#6658](https://www.sqlalchemy.org/trac/ticket/6658)

### orm

+   **[orm] [bug] [回归]**

    进一步修复了与[#6052](https://www.sqlalchemy.org/trac/ticket/6052)相同区域的进一步退化，其中加载器选项以及像`Query.join()`这样的方法的调用会失败，如果语句的左侧被替换为使用`Query.with_entities()`方法，或者在使用`Select.with_only_columns()`方法时使用 2.0 风格的查询。已向对象添加了一组新的状态，用于跟踪选项/连接所依赖的“左侧”实体，当主实体发生变化时进行备忘。

    参考：[#6253](https://www.sqlalchemy.org/trac/ticket/6253), [#6503](https://www.sqlalchemy.org/trac/ticket/6503)

+   **[orm] [bug]**

    优化了 ORM 子查询在处理延迟列和列属性时的行为，使其更加兼容 1.3 版本，同时也提供了 1.4 版本的新功能。在 1.4 版本中，由于子查询不使用加载器选项，包括`undefer()`，对于具有延迟属性的 ORM 实体的子查询现在将呈现那些直接引用映射表列的延迟属性，因为如果外部 SELECT 使用这些列，这些列是必需的；然而，如果延迟属性引用了我们通常使用`column_property()`定义的组合 SQL 表达式，那么这些属性将不会成为子查询的一部分，因为如果需要，在子查询中可以显式选择这些属性。如果从这个子查询中选择实体，则列表达式仍然可以在派生子查询列的“外部”呈现。这实际上产生了与在 1.3 版本中工作时基本相同的行为。然而，在这种情况下，修复还必须确保 ORM 启用的`select()`的`.selected_columns`集合也遵循这些规则，这特别允许递归 CTE 在这种情况下正确呈现，之前由于这个问题而无法正确呈现。

    参考：[#6661](https://www.sqlalchemy.org/trac/ticket/6661)

### sql

+   **[sql] [bug]**

    修复了 CTE 构造中的问题，主要与 ORM 用例相关，其中针对“匿名”标签的递归 CTE（例如在 ORM `column_property()`映射中看到的标签）将在`WITH RECURSIVE xyz(...)`部分中呈现为其原始内部标签，而不是一个干净的匿名化名称。

    参考：[#6663](https://www.sqlalchemy.org/trac/ticket/6663)

### mypy

+   **[mypy] [bug]**

    修复了在 mypy 插件中的问题，自定义声明基类的类信息在缓存的 mypy 通行证上未能正确处理，导致引发 AssertionError。

    参考：[#6476](https://www.sqlalchemy.org/trac/ticket/6476)

### asyncio

+   **[asyncio] [usecase]**

    实现了 `async_scoped_session` 来解决 `scoped_session` 和 `AsyncSession` 之间一些与 asyncio 相关的不兼容性，其中一些方法（特别是 `async_scoped_session.remove()` 方法）应该使用 `await` 关键字。

    另请参阅

    使用 asyncio 作用域会话

    参考：[#6583](https://www.sqlalchemy.org/trac/ticket/6583)

+   **[asyncio] [bug] [postgresql]**

    修复了 asyncio 实现中的错误，其中绿色适配系统未能传播 `BaseException` 子类，尤其是包括 `asyncio.CancelledError` 在内，到引擎用于使连接无效并清理连接时使用的异常处理逻辑，从而阻止在取消任务时正确处理连接的情况。

    参考：[#6652](https://www.sqlalchemy.org/trac/ticket/6652)

### postgresql

+   **[postgresql] [bug] [oracle]**

    修复了在 PostgreSQL 和 Oracle 上使用 `INTERVAL` 数据类型与 `timedelta()` 对象进行比较操作时会产生 `AttributeError` 的问题。感谢 MajorDallas 提交的拉取请求。

    参考：[#6649](https://www.sqlalchemy.org/trac/ticket/6649)

+   **[postgresql] [bug]**

    修复了池“预先 ping”功能会隐式启动事务的问题，这将干扰自定义事务标志，例如当与 psycopg2 驱动程序一起使用时，会干扰 PostgreSQL 的“只读”模式。

    参考：[#6621](https://www.sqlalchemy.org/trac/ticket/6621)

### mysql

+   **[mysql] [usecase]**

    添加了新的构造 `match`，提供了 MySQL 的 MATCH 运算符的全部范围，包括多列支持和修饰符。感谢 Anton Kovalevich 提交的拉取请求。

    另请参阅

    `match`

    参考：[#6132](https://www.sqlalchemy.org/trac/ticket/6132)

### mssql

+   **[mssql] [change]**

    对 pymssql 方言使用的服务器版本正则表达式进行了改进，以防止在无效版本字符串的情况下发生正则表达式溢出。

    参考：[#6253](https://www.sqlalchemy.org/trac/ticket/6253), [#6503](https://www.sqlalchemy.org/trac/ticket/6503)

+   **[mssql] [bug]**

    修复了“schema_translate_map”功能在与具有 IDENTITY 列的表进行 INSERT 操作时无法正确运行的错误，其中 IDENTITY 列的值在 INSERT 的值中指定，从而触发 SQLAlchemy 将 IDENTITY INSERT 设置为“on”的功能；正是在这个指令中，模式转换映射将无法被遵守。

    参考：[#6658](https://www.sqlalchemy.org/trac/ticket/6658)

## 1.4.18

发布日期：2021 年 6 月 10 日

### orm

+   **[orm] [性能] [bug] [回归]**

    修复了 ORM 在解析给定映射列到结果行时的回归问题，其中在诸如连接式急加载之类的情况下，由于一些自 1.3 版本以来已删除的逻辑，可能会发生稍微昂贵的“回退”以设置此解析。该问题还可能导致在使用 1.4 风格查询与连接式急加载时发出涉及列解析的弃用警告。

    参考：[#6596](https://www.sqlalchemy.org/trac/ticket/6596)

+   **[orm] [bug]**

    澄清了`relationship.bake_queries`标志的当前目的，在 1.4 版本中，其目的是启用或禁用“lambda 缓存”在“lazyload”和“selectinload”加载策略中的语句；这与用于大多数语句的更基础的 SQL 查询缓存是分开的。此外，懒加载器不再为一对多的 SQL 查询使用自己的缓存，这是一个实现怪癖，对于任何其他加载程序场景都不存在。最后，处理各种类/关系组合时，懒加载器和 selectinloader 策略可能发出的“lru 缓存”警告已被移除；根据对一些最终用户案例的分析，这个警告并不意味着有任何重大问题。虽然为这样的关系设置`bake_queries=False`将阻止使用此缓存，但在这种情况下，与使用需要经常刷新的缓存相比，使用不缓存与使用缓存仍然胜出。

    参考：[#6072](https://www.sqlalchemy.org/trac/ticket/6072), [#6487](https://www.sqlalchemy.org/trac/ticket/6487)

+   **[orm] [bug] [回归]**

    调整了从基本`Session`类生成类（如`scoped_session`和`AsyncSession`）的方式，使得像 Flask-SQLAlchemy 使用的自定义`Session`子类在调用超类方法时不需要实现位置参数，并且可以继续使用与之前版本相同的参数样式。

    参考：[#6285](https://www.sqlalchemy.org/trac/ticket/6285)

+   **[orm] [错误] [回归]**

    修复了一个问题，即针对涉及联接表继承的复杂左侧进行 joinedload 查询生成可能无法生成正确查询的问题，由于存在子句适应问题。

    参考：[#6595](https://www.sqlalchemy.org/trac/ticket/6595)

+   **[orm] [错误]**

    修复了在实验性“从 INSERT/UPDATE 中选择 ORM 对象”用例中的问题，如果语句针对单表继承子类，则会引发错误。

    参考：[#6591](https://www.sqlalchemy.org/trac/ticket/6591)

+   **[orm] [错误]**

    当多个关系重叠至于外键属性时，为`relationship()`发出的警告现在包括用于每个警告的特定“重叠”参数，以便在不更改映射的情况下消除警告。

    参考：[#6400](https://www.sqlalchemy.org/trac/ticket/6400)

### asyncio

+   **[asyncio] [用例]**

    实现了一种新的注册表架构，允许定位对象的`Async`版本，如`AsyncSession`、`AsyncConnection`等，给定代理的“同步”对象，即`Session`、`Connection`。 以前，如果使用查找函数，每次都会重新创建一个`Async`对象，这不太理想，因为“async”对象的标识和状态不会在调用之间保留。

    从那里开始，新增了新的辅助函数`async_object_session()`，`async_session()`以及一个新的`InstanceState`属性`InstanceState.async_session`，用于检索与 ORM 映射对象关联的原始`AsyncSession`，与`AsyncSession`关联的`Session`，以及与`InstanceState`关联的`AsyncSession`。

    此补丁还实现了新方法`AsyncSession.in_nested_transaction()`，`AsyncSession.get_transaction()`，`AsyncSession.get_nested_transaction()`。

    参考：[#6319](https://www.sqlalchemy.org/trac/ticket/6319)

+   **[asyncio] [bug]**

    修复了在使用`NullPool`或`StaticPool`与异步引擎时出现的问题。这主要影响了 aiosqlite 方言。

    参考：[#6575](https://www.sqlalchemy.org/trac/ticket/6575)

+   **[asyncio] [bug]**

    添加了`asyncio.exceptions.TimeoutError`，`asyncio.exceptions.CancelledError`作为所谓的“退出异常”，一类包括`GreenletExit`和`KeyboardInterrupt`等事件的异常，这些事件被认为需要考虑将 DBAPI 连接视为不可用状态，应该对其进行回收。

    参考：[#6592](https://www.sqlalchemy.org/trac/ticket/6592)

### postgresql

+   **[postgresql] [bug] [regression]**

    修复了使用 PostgreSQL “INSERT..ON CONFLICT” 结构在“executemany”上下文中与“SET”子句中的绑定参数一起使用时，如果使用 psycopg2 驱动程序会失败的回归问题，由于在 1.4 中默认使用了 psycopg2 快速执行助手，这实际上是一个回归。添加了额外的检查来排除这种类型的语句不适用于特定扩展。

    参考：[#6581](https://www.sqlalchemy.org/trac/ticket/6581)

### sqlite

+   **[sqlite] [bug]**

    添加关于在 URL 中传递给 pysqlcipher 的与加密相关的 pragma 的注释。

    此更改也已经 **回溯** 到：1.3.25

    参考：[#6589](https://www.sqlalchemy.org/trac/ticket/6589)

+   **[sqlite] [bug] [regression]**

    在版本 1.4.3 中发布的 pysqlcipher 修复了问题[#5848](https://www.sqlalchemy.org/trac/ticket/5848)遗憾的是，新的 `on_connect_url` 钩子在正常使用 `create_engine()` 时错误地没有接收到一个 `URL` 对象，而是接收到一个未处理的字符串；测试套件未能完全设置调用此钩子的实际条件。这个问题已经修复。

    参考：[#6586](https://www.sqlalchemy.org/trac/ticket/6586)

### orm

+   **[orm] [performance] [bug] [regression]**

    修复了关于 ORM 如何解析给定映射列到结果行的回归问题，在某些情况下，比如连接的急切加载，由于一些在 1.3 版本中移除的逻辑，会稍微昂贵一些的“回退”可能会发生，以设置这个解析。该问题还可能导致在使用带有连接的急切加载的 1.4 风格查询时发出涉及列解析的弃用警告。

    参考：[#6596](https://www.sqlalchemy.org/trac/ticket/6596)

+   **[orm] [bug]**

    澄清了`relationship.bake_queries`标志的当前目的，在 1.4 版本中是启用或禁用“延迟加载”和“selectinload”加载策略中语句的“lambda 缓存”；这与用于大多数语句的更基础的 SQL 查询缓存是分开的。此外，懒加载器不再为多对一 SQL 查询使用自己的缓存，这是一个不存在于任何其他加载器场景的实现怪癖。最后，处理各种类/关系组合时懒加载器和 selectinloader 策略可能发出的“lru 缓存”警告已被移除；根据对一些最终用户案例的分析，此警告并不暗示任何重大问题。虽然为这样的关系设置`bake_queries=False`将阻止使用此缓存，但在这种情况下，与需要经常刷新的缓存相比，使用不缓存与使用缓存仍然可能在使用缓存方面胜出。

    参考：[#6072](https://www.sqlalchemy.org/trac/ticket/6072), [#6487](https://www.sqlalchemy.org/trac/ticket/6487)

+   **[orm] [bug] [regression]**

    调整了生成诸如`scoped_session`和`AsyncSession`等类的方式，使其从基础`Session`类生成，这样像 Flask-SQLAlchemy 使用的自定义`Session`子类在调用超类方法时不需要实现位置参数，并且可以继续使用与之前版本相同的参数样式。

    参考：[#6285](https://www.sqlalchemy.org/trac/ticket/6285)

+   **[orm] [bug] [regression]**

    修复了针对涉及联合表继承的复杂左侧的 joinedload 的查询生成问题，可能由于子句适应问题而无法生成正确的查询。

    参考：[#6595](https://www.sqlalchemy.org/trac/ticket/6595)

+   **[orm] [bug]**

    修复了在实验性“从 INSERT/UPDATE 中选择 ORM 对象”用例中的问题，如果语句针对单表继承子类，则会引发错误。

    参考：[#6591](https://www.sqlalchemy.org/trac/ticket/6591)

+   **[orm] [bug]**

    当`relationship()`存在多个关系会重叠至外键属性时，现在会包含用于每个警告的特定“重叠”参数，以便在不更改映射的情况下消除警告。

    参考：[#6400](https://www.sqlalchemy.org/trac/ticket/6400)

### asyncio

+   **[asyncio] [usecase]**

    实现了一种新的注册表架构，允许定位对象的`Async`版本，如`AsyncSession`，`AsyncConnection`等，给定代理的“sync”对象，即`Session`，`Connection`。 以前，到某种程度上使用这样的查找函数时，每次都会重新创建一个`Async`对象，这不太理想，因为“async”对象的标识和状态不会在调用之间保留。

    从那里开始，新增了一些新的辅助函数`async_object_session()`，`async_session()`以及一个新的`InstanceState`属性`InstanceState.async_session`，它们用于检索与 ORM 映射对象关联的原始`AsyncSession`，与`AsyncSession`关联的`Session`，以及与`InstanceState`关联的`AsyncSession`。

    此补丁还实现了新方法`AsyncSession.in_nested_transaction()`，`AsyncSession.get_transaction()`，`AsyncSession.get_nested_transaction()`。

    参考：[#6319](https://www.sqlalchemy.org/trac/ticket/6319)

+   **[asyncio] [bug]**

    修复了在使用`NullPool`或`StaticPool`与异步引擎时出现的问题。 这主要影响了 aiosqlite 方言。

    参考：[#6575](https://www.sqlalchemy.org/trac/ticket/6575)

+   **[asyncio] [bug]**

    添加了 `asyncio.exceptions.TimeoutError`，`asyncio.exceptions.CancelledError` 作为所谓的“退出异常”，一类包括 `GreenletExit` 和 `KeyboardInterrupt` 在内的异常，这被认为是需要考虑 DBAPI 连接处于不可用状态并应该被回收的事件。

    参考：[#6592](https://www.sqlalchemy.org/trac/ticket/6592)

### postgresql

+   **[postgresql] [bug] [regression]**

    修复了在使用 PostgreSQL 的“INSERT..ON CONFLICT”结构时出现的回归问题，如果它在“executemany”上下文中与“SET”子句中的绑定参数一起使用，并且由于隐式使用 psycopg2 快速执行助手，这些助手不适用于此样式的 INSERT 语句；由于这些助手是 1.4 中的默认设置，这实际上是一个回归。 已添加额外的检查以排除这种语句从该特定扩展中。

    参考：[#6581](https://www.sqlalchemy.org/trac/ticket/6581)

### sqlite

+   **[sqlite] [bug]**

    添加有关在 URL 中传递给 pysqlcipher 的加密相关 pragma 的说明。

    此更改也已**回溯**到：1.3.25

    参考：[#6589](https://www.sqlalchemy.org/trac/ticket/6589)

+   **[sqlite] [bug] [regression]**

    在版本 1.4.3 中发布的 pysqlcipher 修复 [#5848](https://www.sqlalchemy.org/trac/ticket/5848) 不幸地无效，因为新的 `on_connect_url` 钩子在正常使用 `create_engine()` 时错误地未接收到 `URL` 对象，而是接收到一个未处理的字符串；测试套件未能完全设置调用此钩子的实际条件。 这个问题已经修复。

    参考：[#6586](https://www.sqlalchemy.org/trac/ticket/6586)

## 1.4.17

发布日期：2021 年 5 月 29 日

### orm

+   **[orm] [bug] [regression]**

    修复了刚发布的性能修复引起的回归问题，该修复在 #6550 中提到，其中对关系进行 query.join() 可能会导致 AttributeError，如果查询仅针对非 ORM 结构进行，这是一种相当不寻常的调用模式。

    参考：[#6558](https://www.sqlalchemy.org/trac/ticket/6558)

### orm

+   **[orm] [bug] [regression]**

    修复了刚发布的性能修复引起的回归问题，该修复在 #6550 中提到，其中对关系进行 query.join() 可能会导致 AttributeError，如果查询仅针对非 ORM 结构进行，这是一种相当不寻常的调用模式。

    参考：[#6558](https://www.sqlalchemy.org/trac/ticket/6558)

## 1.4.16

发布日期：2021 年 5 月 28 日

### 通用

+   **[general] [bug]**

    解决了各种在 Python 版本 3.10.0b1 中出现的弃用警告。

    参考：[#6540](https://www.sqlalchemy.org/trac/ticket/6540), [#6543](https://www.sqlalchemy.org/trac/ticket/6543)

### orm

+   **[orm] [bug]**

    修复了当使用 `relationship.cascade_backrefs` 参数设置为 `False` 时的问题，根据 在 SQLAlchemy 2.0 中将被移除的 cascade_backrefs 行为，在 SQLAlchemy 2.0 中将成为标准行为，如果将项目添加到一个具有唯一性的集合（如 `set` 或 `dict`）中，如果对象已经通过反向引用与该集合关联，则不会触发级联事件。此修复引入了一个基本的集合机制变更，通过引入一个新的事件状态，即使集合没有净变化，也可以触发集合变异的事件；现在使用一个新的事件钩子 `AttributeEvents.append_wo_mutation()` 来执行该操作。

    引用：[#6471](https://www.sqlalchemy.org/trac/ticket/6471)

+   **[orm] [bug] [regression]**

    修复了涉及带有条件或 CASE 表达式的标记 ORM 复合元素的子句适应的回归问题，这可能导致在 ORM 连接 / joinedload 操作中使用的别名表达式无法正确适应，例如在连接的 ON 子句中引用错误的表。

    此更改还改进了在将 ORM 属性作为目标传递给 `Select.join()` 时调用的性能提升。

    引用：[#6550](https://www.sqlalchemy.org/trac/ticket/6550)

+   **[orm] [bug] [regression]**

    修复了全组合继承、全局 with_polymorphic、自引用关系和连接加载的组合无法生成具有延迟加载范围和对象刷新操作的查询，同时还尝试渲染连接加载器的回归问题。

    引用：[#6495](https://www.sqlalchemy.org/trac/ticket/6495)

+   **[orm] [bug]**

    加强了`Session.execute()`的绑定解析规则，以便当针对 ORM 对象构建非 ORM 语句（如 `insert()` 构造）时，尽可能地使用 ORM 实体来解析绑定，例如对于在公共超类上设置了绑定映射但没有具体映射器或表命名在映射中的 `Session`。

    引用：[#6484](https://www.sqlalchemy.org/trac/ticket/6484)

### 引擎

+   **[engine] [bug]**

    修复了在 URL 的数据库部分中存在 `@` 符号且 URL 还具有用户名:密码部分时无法正确解释的问题。

    参考：[#6482](https://www.sqlalchemy.org/trac/ticket/6482)

+   **[引擎] [错误]**

    修复了`URL`的一个长期问题，即如果 URL 不包含带有反斜杠的数据库部分，则问号后面的查询参数将无法正确解析。

    参考：[#6329](https://www.sqlalchemy.org/trac/ticket/6329)

### sql

+   **[sql] [错误] [回归]**

    修复了动态加载器策略和`relationship()`的回归，其中`relationship.order_by`参数被存储为可变列表，当与动态查询对象一起使用其他“order_by”方法时，可能会发生变异，导致 ORDER BY 条件继续重复增长。

    参考：[#6549](https://www.sqlalchemy.org/trac/ticket/6549)

### mssql

+   **[mssql] [用例]**

    实现了对`CTE`构造的支持，可以直接用作`delete()`构造的目标，即“WITH … AS cte DELETE FROM cte”。这似乎是 SQL Server 的一个有用功能。

    参考：[#6464](https://www.sqlalchemy.org/trac/ticket/6464)

### 杂项

+   **[错误] [扩展]**

    修复了在使用`automap_base()`时发出的弃用警告，如果没有传递现有的`Base`。

    参考：[#6529](https://www.sqlalchemy.org/trac/ticket/6529)

+   **[错误] [pep484]**

    从代码中删除了 pep484 类型。当前的工作重点是在存根包周围，而在 SQLAlchemy 源代码中有两个地方的类型会使事情变得更糟，因为 SQLAlchemy 源代码中的类型通常比存根中的版本旧。

    参考：[#6461](https://www.sqlalchemy.org/trac/ticket/6461)

+   **[错误] [扩展] [回归]**

    修复了`sqlalchemy.ext.instrumentation`扩展中的回归，完全阻止了仪器处置的工作。此修复包括 1.4 回归修复以及 1.3 中存在的相关问题的修复。作为此更改的一部分，`sqlalchemy.ext.instrumentation.InstrumentationManager`类现在有一个新方法`unregister()`，取代了以前的方法`dispose()`，自 1.4 版本以来未被调用。

    参考：[#6390](https://www.sqlalchemy.org/trac/ticket/6390)

### 通用

+   **[通用] [错误]**

    解决了在 Python 版本 3.10.0b1 中出现的各种弃用警告。

    参考：[#6540](https://www.sqlalchemy.org/trac/ticket/6540)，[#6543](https://www.sqlalchemy.org/trac/ticket/6543)

### orm

+   **[orm] [bug]**

    修复了当使用`relationship.cascade_backrefs`参数设置为`False`时的问题，根据在 2.0 中将删除的 cascade_backrefs 行为，这将成为 SQLAlchemy 2.0 中的标准行为，如果将项目添加到一个唯一化的集合，如`set`或`dict`，如果对象已经通过反向引用与该集合关联，则不会触发级联事件。此修复通过引入一个新的事件状态来表示集合机制的根本变化，即使集合上没有净变化，也可以触发集合变异的事件；现在使用一个新的事件钩子`AttributeEvents.append_wo_mutation()`来执行该操作。

    参考：[#6471](https://www.sqlalchemy.org/trac/ticket/6471)

+   **[orm] [bug] [regression]**

    修复了涉及带有条件或 CASE 表达式的标记 ORM 复合元素的子句适应，这可能导致 ORM 连接/连接加载操作中使用的别名表达式无法正确适应，例如在连接的 ON 子句中引用错误的表。

    此更改还改进了在将 ORM 属性作为目标传递给`Select.join()`时调用过程中的性能提升。

    参考：[#6550](https://www.sqlalchemy.org/trac/ticket/6550)

+   **[orm] [bug] [regression]**

    修复了完全组合的连接继承、全局 with_polymorphic、自引用关系和连接加载无法生成具有延迟加载范围和尝试渲染连接加载程序的查询的回归问题。

    参考：[#6495](https://www.sqlalchemy.org/trac/ticket/6495)

+   **[orm] [bug]**

    加强了对`Session.execute()`的绑定解析规则，以便当针对 ORM 对象构建非 ORM 语句（例如`insert()`构造）时，尽可能地使用 ORM 实体来解析绑定，例如对于在公共超类上设置了绑定映射的`Session`，而没有在映射中命名特定的映射器或表。

    参考：[#6484](https://www.sqlalchemy.org/trac/ticket/6484)

### 引擎

+   **[engine] [bug]**

    修复了 URL 数据库部分中的 `@` 符号无法正确解释的问题，如果 URL 还包含用户名:密码部分，则会出现问题。

    参考：[#6482](https://www.sqlalchemy.org/trac/ticket/6482)

+   **[引擎] [错误]**

    修复了 `URL` 的一个长期问题，在此问题中，如果 URL 不包含带反斜杠的数据库部分，则不会正确解析跟在问号后面的查询参数。

    参考：[#6329](https://www.sqlalchemy.org/trac/ticket/6329)

### SQL

+   **[sql] [错误] [回归]**

    修复了动态加载器策略和 `relationship()` 整体中的回归问题，其中 `relationship.order_by` 参数被存储为可变列表，当与针对动态查询对象使用的其他“order_by”方法结合时，可能会发生突变，导致 ORDER BY 条件继续重复增长。

    参考：[#6549](https://www.sqlalchemy.org/trac/ticket/6549)

### MSSQL

+   **[mssql] [用例]**

    实现了对 `CTE` 结构的支持，可以直接用作 `delete()` 结构的目标，即 “WITH … AS cte DELETE FROM cte”。这似乎是 SQL Server 的一个有用功能。

    参考：[#6464](https://www.sqlalchemy.org/trac/ticket/6464)

### 杂项

+   **[错误] [扩展]**

    修复了在使用 `automap_base()` 时发出的弃用警告，而不传递现有的 `Base`。 

    参考：[#6529](https://www.sqlalchemy.org/trac/ticket/6529)

+   **[错误] [pep484]**

    从代码中移除 pep484 类型。当前的工作重点在于存根包，而在 SQLAlchemy 源代码中添加类型会使情况变得更糟，因为与存根中的版本相比，SQLAlchemy 源代码中的类型通常过时。

    参考：[#6461](https://www.sqlalchemy.org/trac/ticket/6461)

+   **[错误] [扩展] [回归]**

    修复了 `sqlalchemy.ext.instrumentation` 扩展中的回归问题，该问题完全阻止了仪器处置工作。此修复包括对 1.4 回归的修复以及对 1.3 中存在的相关问题的修复。作为此更改的一部分，`sqlalchemy.ext.instrumentation.InstrumentationManager` 类现在具有一个新方法 `unregister()`，它替换了先前的方法 `dispose()`，自 1.4 版本以来未调用。

    参考：[#6390](https://www.sqlalchemy.org/trac/ticket/6390)

## 1.4.15

发布日期：2021 年 5 月 11 日

### 通用

+   **[通用] [特性]**

    SQLAlchemy 中的警告系统应用了一种新方法，动态地准确预测每个警告的适当堆栈级别。这允许更直接地评估 SQLAlchemy 生成的警告和弃用警告的来源，因为警告将指示在最终用户代码中的源行，而不是从 SQLAlchemy 自身源代码的任意级别。

    参考：[#6241](https://www.sqlalchemy.org/trac/ticket/6241)

### orm

+   **[orm] [错误] [回归]**

    修复了“急切加载器在未过期时运行”功能引起的额外回归问题[#1763](https://www.sqlalchemy.org/trac/ticket/1763)，其中该功能将在`contains_eager()`急切加载选项中运行，如果`contains_eager()`链接到另一个急切加载器选项，则会产生一个不正确的查询，因为原始的查询绑定连接条件不再存在。

    参考：[#6449](https://www.sqlalchemy.org/trac/ticket/6449)

+   **[orm] [错误]**

    修复了子查询加载器策略中的问题，该问题阻止了缓存的正确工作。这将在日志中显示为所有子查询加载 SQL 的“生成”消息，而不是“缓存”，通过用新键饱和缓存来降低整体性能；它还会产生“LRU 大小警告”。

    参考：[#6459](https://www.sqlalchemy.org/trac/ticket/6459)

### sql

+   **[sql] [错误]**

    调整了 1.4.12 中作为[#6397](https://www.sqlalchemy.org/trac/ticket/6397)的一部分添加的逻辑，以便内部对`BindParameter`对象的变异发生在子句构造阶段，而不是在编译阶段。在后一种情况下，变异仍会对传入的构造产生副作用，并且可能会干扰其他内部变异例程。

    参考：[#6460](https://www.sqlalchemy.org/trac/ticket/6460)

### mysql

+   **[mysql] [错误] [文档]**

    增加了对 mysql 连接 URI 中`ssl_check_hostname=`参数的支持，并更新了有关安全连接的 mysql 方言文档。原始拉取请求由 Jerry Zhao 提供。

    参考：[#5397](https://www.sqlalchemy.org/trac/ticket/5397)

### 通用

+   **[通用] [特性]**

    SQLAlchemy 中的警告系统应用了一种新方法，动态地准确预测每个警告的适当堆栈级别。这允许更直接地评估 SQLAlchemy 生成的警告和弃用警告的来源，因为警告将指示在最终用户代码中的源行，而不是从 SQLAlchemy 自身源代码的任意级别。

    参考：[#6241](https://www.sqlalchemy.org/trac/ticket/6241)

### orm

+   **[orm] [错误] [回归]**

    修复了“急切加载器在未过期时运行”功能引起的额外回归问题，该功能会在`contains_eager()`急切加载选项中运行，如果`contains_eager()`链接到另一个急切加载器选项，那么原始查询绑定的连接条件将不再存在，从而产生不正确的查询。

    参考：[#6449](https://www.sqlalchemy.org/trac/ticket/6449)

+   **[orm] [bug]**

    修复了子查询加载策略中的问题，该问题导致缓存无法正常工作。这将在日志中显示为所有子查询加载 SQL 的“generated”消息，而不是“cached”，通过用新键饱和缓存会降低整体性能；还会产生“LRU size alert”警告。

    参考：[#6459](https://www.sqlalchemy.org/trac/ticket/6459)

### sql

+   **[sql] [bug]**

    调整了 1.4.12 中作为[#6397](https://www.sqlalchemy.org/trac/ticket/6397)的一部分添加的逻辑，使得内部对`BindParameter`对象的变异发生在构造子句阶段，而不是在编译阶段。在后一种情况下，变异仍会对传入的构造产生副作用，并且可能干扰其他内部变异例程。

    参考：[#6460](https://www.sqlalchemy.org/trac/ticket/6460)

### mysql

+   **[mysql] [bug] [documentation]**

    增加了对 mysql 连接 URI 中`ssl_check_hostname=`参数的支持，并更新了 mysql 方言文档有关安全连接的内容。原始拉取请求由 Jerry Zhao 提供。

    参考：[#5397](https://www.sqlalchemy.org/trac/ticket/5397)

## 1.4.14

发布日期：2021 年 5 月 6 日

### orm

+   **[orm] [bug] [regression]**

    修复了`lazy='dynamic'`加载器与分离对象结合时的回归问题。先前的行为是，动态加载器在调用`.all()`等方法时，对于分离对象返回空列表而不报错，这已经恢复；但现在会发出警告，因为这不是正确的结果。其他动态加载器场景会正确地引发`DetachedInstanceError`。

    参考：[#6426](https://www.sqlalchemy.org/trac/ticket/6426)

### engine

+   **[engine] [usecase] [orm]**

    对在现有`.begin()`上下文管理器内调用`.commit()`或`.rollback()`的用例应用了一致的行为，还可能在提交或回滚后的块内部发出 SQL。这一变更延续了首次添加在[#6155](https://www.sqlalchemy.org/trac/ticket/6155)中的变更，其中提出了在`.begin()`上下文管理器块内调用“rollback”的用例：

    +   在所有范围内，包括传统和未来的`Engine`、ORM `Session`、asyncio `AsyncEngine`，现在允许调用`.commit()`或`.rollback()`而不会出现错误或警告。之前，`Session`是不允许这样做的。

    +   然后关闭上下文管理器的剩余范围；当块结束时，会发出检查以查看事务是否已经结束，如果是，则块返回而不执行任何操作。

    +   如果在调用`.commit()`或`.rollback()`之后的块内发出任何类型的后续 SQL，则现在会引发**错误**。在这种状态下，应该关闭块，否则可执行对象的状态将变得不确定。

    参考：[#6288](https://www.sqlalchemy.org/trac/ticket/6288)

+   **[engine] [bug] [regression]**

    为调用返回空行的语句的`CursorResult.keys()`方法建立了一个弃用路径，以支持“records”包以及任何其他未迁移的应用程序使用的传统模式。之前，这将无条件引发`ResourceClosedException`，就像尝试获取行时一样。虽然这是正确的行为，但`LegacyCursorResult`对象现在在这种情况下将返回一个空列表给`.keys()`，就像在 1.3 中一样，并发出一个 2.0 弃用警告。当使用 2.0 风格的“future”引擎时，使用的`_cursor.CursorResult`将继续像现在一样引发异常。

    参考：[#6427](https://www.sqlalchemy.org/trac/ticket/6427)

### sql

+   **[sql] [bug] [regression]**

    修复了在 1.4.12 中刚刚进行的“empty in”更改引起的回归，其中表达式需要对“not in”用例进行括号化，否则条件将干扰其他过滤条件。

    参考：[#6428](https://www.sqlalchemy.org/trac/ticket/6428)

+   **[sql] [bug] [regression]**

    当在 SQL 编译中使用`TypeDecorator`类时，除非将`.cache_ok`标志设置为`True`或`False`，否则现在会发出警告。可以设置一个新的类级属性`TypeDecorator.cache_ok`，如果设置为`True`，则表示传递给对象的所有参数都可以安全用作缓存键，`False`表示不能。

    参考：[#6436](https://www.sqlalchemy.org/trac/ticket/6436)

### orm

+   **[orm] [bug] [regression]**

    修复了`lazy='dynamic'`加载器与分离对象结合时的回归问题。先前的行为是，动态加载器在调用`.all()`等方法时，对于分离对象返回空列表而不报错，这已经恢复；然而现在会发出警告，因为这不是正确的结果。其他动态加载器场景会正确地引发`DetachedInstanceError`。

    参考：[#6426](https://www.sqlalchemy.org/trac/ticket/6426)

### engine

+   **[engine] [usecase] [orm]**

    对在现有`.begin()`上下文管理器内调用`.commit()`或`.rollback()`的用例应用了一致的行为，还可能在提交或回滚后的块内发出 SQL。这个变化延续了首次添加在[#6155](https://www.sqlalchemy.org/trac/ticket/6155)中的变化，其中提出了在`.begin()`上下文管理器块内调用“rollback”的用例：

    +   现在在所有范围内，包括传统和未来`Engine`、ORM `Session`、asyncio `AsyncEngine` 的作用域内，都允许调用`.commit()`或`.rollback()`而不会出现错误或警告。先前，`Session`不允许这样做。

    +   然后关闭上下文管理器的剩余范围；当块结束时，会发出检查以查看事务是否已经结束，如果是，则块返回而不执行任何操作。

    +   如果在调用`.commit()`或`.rollback()`后，在块内发出任何类型的后续 SQL，将会引发**错误**。在这种状态下，应该关闭块，因为可执行对象的状态在这种状态下将是未定义的。

    参考：[#6288](https://www.sqlalchemy.org/trac/ticket/6288)

+   **[engine] [bug] [regression]**

    为调用返回零行的语句的`CursorResult.keys()`方法建立了一个弃用路径，以支持“records”包以及任何其他未迁移的应用程序使用的传统模式。先前，这会无条件引发`ResourceClosedException`，就像尝试获取行时一样。虽然这是正确的行为，但`LegacyCursorResult`对象现在在这种情况下将返回一个空列表给`.keys()`，就像在 1.3 中一样，并发出一个 2.0 弃用警告。当使用 2.0 风格“future”引擎时，使用的`_cursor.CursorResult`将继续像现在一样引发。

    参考：[#6427](https://www.sqlalchemy.org/trac/ticket/6427)

### sql

+   **[sql] [bug] [regression]**

    修复了在 1.4.12 版本中刚刚引入的“空 in”更改导致的回归问题，其中表达式需要在“not in”用例中加括号，否则条件将干扰其他过滤条件。

    参考���[#6428](https://www.sqlalchemy.org/trac/ticket/6428)

+   **[sql] [bug] [regression]**

    当在 SQL 编译中使用`TypeDecorator`类时，除非设置了`.cache_ok`标志为`True`或`False`，否则现在会发出警告。可以设置一个新的类级属性`TypeDecorator.cache_ok`，如果设置为`True`，则表示传递给对象的所有参数都可以安全地用作缓存键，`False`表示不能。

    参考：[#6436](https://www.sqlalchemy.org/trac/ticket/6436)

## 1.4.13

发布日期：2021 年 5 月 3 日

### orm

+   **[orm] [bug] [regression]**

    修复了`selectinload`加载器策略中的回归问题，当处理跨越多个列连接的关系时，例如使用复合外键时，会导致其错误地缓存内部状态。无效的缓存然后会导致其他不相关的加载器操作失败。

    参考：[#6410](https://www.sqlalchemy.org/trac/ticket/6410)

+   **[orm] [bug] [regression]**

    修复了当`Query.filter_by()`的主实体是来自问题主实体的 SQL 函数或其他表达式而不是该实体的简单实体或列时将无法工作的回归问题。此外，改进了`Select.filter_by()`的行为，使其在非 ORM 上下文中甚至可以与列表达式一起使用。

    参考：[#6414](https://www.sqlalchemy.org/trac/ticket/6414)

+   **[orm] [bug] [regression]**

    修复了使用`selectinload()`和`subqueryload()`加载两级深度路径时会导致属性错误的回归问题。

    参考：[#6419](https://www.sqlalchemy.org/trac/ticket/6419)

+   **[orm] [bug] [regression]**

    修复了在“动态”关系中与`noload()`加载器策略一起使用会导致属性错误的回归问题，因为 noload 策略会尝试应用于动态加载器。

    参考：[#6420](https://www.sqlalchemy.org/trac/ticket/6420)

### engine

+   **[engine] [bug] [regression]**

    恢复了一个意外从`Connection`中删除的传统事务行为，因为在先前版本中从未作为已知用例进行测试，以前调用`Connection.begin_nested()`方法时，当没有事务存在时，根本不会创建 SAVEPOINT，而是启动外部事务，返回一个`RootTransaction`对象而不是`NestedTransaction`对象。然后，当提交时，此`RootTransaction`将在数据库连接上发出真正的 COMMIT。以前，在所有会自动开始但不提交事务的情况下都存在 2.0 样式行为，这是一种行为变更。

    当使用 2.0 样式连接对象时，行为与之前的 1.4 版本保持不变；调用`Connection.begin_nested()`将“自动开始”外部事务（如果尚未存在），然后按照指示发出 SAVEPOINT，返回`NestedTransaction`对象。通过调用`Connection.commit()`来提交外部事务，就像“随时提交”样式的用法一样。

    在非“future”模式下，虽然恢复了旧行为，但也会发出 2.0 版本弃用警告，因为这是一种传统行为。

    参考：[#6408](https://www.sqlalchemy.org/trac/ticket/6408)

### asyncio

+   **[asyncio] [bug] [regression]**

    修复了由[#6337](https://www.sqlalchemy.org/trac/ticket/6337)引入的回归，当在启动任何 asyncio 循环之前实例化异步引擎时，可能会创建一个附加到错误循环的`asyncio.Lock`，在某些情况下尝试使用引擎时会导致 asyncio 错误消息。

    参考：[#6409](https://www.sqlalchemy.org/trac/ticket/6409)

### postgresql

+   **[postgresql] [usecase]**

    为 PostgreSQL 的 pg8000 方言添加了对服务器端游标的支持。这允许使用`Connection.execution_options.stream_results`选项。

    参考：[#6198](https://www.sqlalchemy.org/trac/ticket/6198)

### orm

+   **[orm] [bug] [regression]**

    修复了`selectinload`加载器策略中的回归问题，该问题会导致其在处理跨越多个列的关系时错误地缓存其内部状态，例如在使用复合外键时。无效的缓存然后会导致其他无关的加载器操作失败。

    参考：[#6410](https://www.sqlalchemy.org/trac/ticket/6410)

+   **[orm] [bug] [regression]**

    修复了如果主导实体是来自所讨论的主要实体的 SQL 函数或其他表达式，而不是该实体的简单实体或列，则`Query.filter_by()`将无法工作的回归问题。此外，改进了`Select.filter_by()`的行为，使其总体上即使在非 ORM 上下文中也能与列表达式一起使用。

    参考：[#6414](https://www.sqlalchemy.org/trac/ticket/6414)

+   **[orm] [bug] [regression]**

    修复了使用`selectinload()`和`subqueryload()`加载两级路径时会导致属性错误的回归问题。

    参考：[#6419](https://www.sqlalchemy.org/trac/ticket/6419)

+   **[orm] [bug] [regression]**

    修复了使用`noload()`加载器策略与“动态”关系结合时会导致属性错误的回归问题，因为 noload 策略会尝试应用于动态加载器。

    参考：[#6420](https://www.sqlalchemy.org/trac/ticket/6420)

### engine

+   **[engine] [bug] [regression]**

    恢复了一个传统的事务行为，该行为在以前的版本中从未作为已知用例进行测试，因此在`Connection`中被无意中移除，当调用`Connection.begin_nested()`方法时，如果没有事务存在，则根本不会创建 SAVEPOINT，而是启动一个外部事务，返回一个`RootTransaction`对象而不是`NestedTransaction`对象。当提交时，此`RootTransaction`将在数据库连接上发出真正的 COMMIT。以前，在所有会自动开始但不提交事务的情况下都存在 2.0 风格的行为，这是一种行为变更。

    当使用 2.0 风格连接对象时，行为与以前的 1.4 版本相同；调用`Connection.begin_nested()`将“自动开始”外部事务（如果尚未存在），然后按照指示发出 SAVEPOINT，返回`NestedTransaction`对象。通过调用`Connection.commit()`来提交外部事务，就像是“随时提交”式的用法。

    在非“future”模式下，虽然恢复了旧行为，但也会发出 2.0 弃用警告，因为这是一种传统行为。

    参考：[#6408](https://www.sqlalchemy.org/trac/ticket/6408)

### asyncio

+   **[asyncio] [bug] [regression]**

    通过[#6337](https://www.sqlalchemy.org/trac/ticket/6337)引入的回归错误已修复，该错误会在实例化异步引擎之前未启动任何 asyncio 循环时创建一个可能附加到错误循环的`asyncio.Lock`，在某些情况下尝试使用引擎时会导致 asyncio 错误消息。

    参考：[#6409](https://www.sqlalchemy.org/trac/ticket/6409)

### postgresql

+   **[postgresql] [usecase]**

    为 PostgreSQL 的 pg8000 方言添加了对服务器端游标的支持。这允许使用`Connection.execution_options.stream_results`选项。

    参考：[#6198](https://www.sqlalchemy.org/trac/ticket/6198)

## 1.4.12

发布日期：2021 年 4 月 29 日

### orm

+   **[orm] [bug]**

    修复了使用`Session.bulk_save_objects()`时的问题，当与持久对象一起使用时，会无法跟踪主键映射的主键，其中主键列名与属性名不同。

    此更改还**反向移植**到：1.3.25

    参考：[#6392](https://www.sqlalchemy.org/trac/ticket/6392)

+   **[orm] [bug] [caching] [regression]**

    修复了关键回归问题，即在 SQL 缓存系统中使用的绑定参数跟踪可能无法跟踪所有参数的情况，即当相同的 SQL 表达式包含参数时，在使用 ORM 相关查询（例如使用类继承等功能）时，然后将该表达式嵌入到包含该表达式多次使用的封闭表达式中，例如`UNION`。 ORM 会在编译时单独复制各个 SELECT 语句作为类继承的一部分，然后嵌入到封闭语句中时将无法适应所有参数。 跟踪此条件的逻辑已调整为适用于参数的多个副本。

    参考：[#6391](https://www.sqlalchemy.org/trac/ticket/6391)

+   **[orm] [bug]**

    修复了两个主要影响`hybrid_property`的问题，这些问题将在常见的错误配置方案下起作用，在 1.3 中被悄悄忽略，在 1.4 中失败，其中“expression”实现将返回非`ClauseElement`，例如布尔值。 对于这两个问题，1.3 的行为是悄悄忽略错误的配置，并最终尝试将该值解释为 SQL 表达式，这将导致查询不正确。

    +   修复了属性系统与`hybrid_property`交互的问题，其中如果属性的`__clause_element__()`方法返回非`ClauseElement`对象，内部的`AttributeError`将导致属性返回`hybrid_property`本身的`expression`函数，因为属性错误针对的名称是`.expression`，这将调用`__getattr__()`方法作为回退。 现在明确提出。 在 1.3 中直接返回了非`ClauseElement`。

    +   修复了 SQL 参数强制系统中的问题，其中将错误类型的对象传递给期望列表达式的方法会失败，如果对象根本不是 SQLAlchemy 对象，例如 Python 函数，在对象不仅被强制转换为绑定值的情况下。再次，1.3 没有全面的参数强制系统，因此这种情况也会悄无声息地通过。

    参考：[#6350](https://www.sqlalchemy.org/trac/ticket/6350)

+   **[orm] [bug]**

    在 ORM 上下文中使用 `Select` 作为子查询会导致对该对象进行就地修改以禁用该对象上的急加载，然后如果在顶层执行上下文中重新使用相同的 `Select`，则该 `Select` 将不会急加载。

    参考：[#6378](https://www.sqlalchemy.org/trac/ticket/6378)

+   **[orm] [bug] [regression]**

    修复了新的 autobegin 行为在现有持久对象具有属性更改的情况下未能“自动开始”的问题，这将影响 `Session.rollback()` 的行为，因为没有创建要回滚的快照。已更新“属性修改”机制以确保“自动开始”，当持久属性以与调用 `Session.add()` 相同的方式更改时发生。这是一个回归，因为在 1.3 中，rollback() 方法始终有一个要回滚的事务，并且每次都会过期。

    参考：[#6359](https://www.sqlalchemy.org/trac/ticket/6359), [#6360](https://www.sqlalchemy.org/trac/ticket/6360)

+   **[orm] [bug] [regression]**

    修复了 ORM 中的回归问题，其中使用混合属性指示来自不同实体的表达式会混淆 ORM 中的列标记逻辑，并尝试从其他类中推导混合的名称，导致属性错误。现在跟踪混合属性的所有者类以及名称。

    参考：[#6386](https://www.sqlalchemy.org/trac/ticket/6386)

+   **[orm] [bug] [regression]**

    修复了混合属性中的回归问题，其中针对 SQL 函数的混合会在某些情况下在尝试为子查询的 `.c` 集合生成条目时生成 `AttributeError`；在其他情况下，这将影响其在 `Query.count()` 等情况下的使用。

    参考：[#6401](https://www.sqlalchemy.org/trac/ticket/6401)

+   **[orm] [bug] [dataclasses]**

    调整了对数据类扫描的声明性扫描，以便当在一个 mixin 上使用新形式将`declared_attr()`放在`dataclasses.field()`构造内而不是实际上是类的描述符属性时，正确地适应已经映射了该`declared_attr()`的现有映射类的子类作为要映射的目标类的情况，因此不应该重新应用到这个类。

    参考：[#6346](https://www.sqlalchemy.org/trac/ticket/6346)

+   **[orm] [bug]**

    修复了（在 1.4 版本中已弃用的）`ForeignKeyConstraint.copy()`方法在使用`schema`参数调用时导致错误的问题。

    参考：[#6353](https://www.sqlalchemy.org/trac/ticket/6353)

### engine

+   **[engine] [bug]**

    修复了使用显式`Sequence`会导致包含多个值短语的`Insert`构造的“内联”行为不一致的问题；第一个序列会是内联的，但随后的序列会是“预执行”，导致序列顺序不一致。现在序列表达式完全内联。

    参考：[#6361](https://www.sqlalchemy.org/trac/ticket/6361)

### sql

+   **[sql] [bug]**

    修订了“EMPTY IN”表达式，不再依赖于使用子查询，因为这会导致一些兼容性和性能问题。对于选定的数据库，新方法利用使用返回 NULL 的 IN 表达式，结合通常的“1 != 1”或“1 = 1”表达式，后面加上 AND 或 OR。该表达式现在是除 SQLite 外所有后端的默认值，对于旧版本 SQLite 仍然存在一些关于元组“IN”的兼容性问题。

    第三方方言仍然可以通过实现一个新的编译器方法`def visit_empty_set_op_expr(self, type_, expand_op)`来覆盖“空集合”表达式的渲染方式，该方法优先于现有的`def visit_empty_set_expr(self, element_types)`方法，后者仍然保留。

    参考：[#6258](https://www.sqlalchemy.org/trac/ticket/6258)，[#6397](https://www.sqlalchemy.org/trac/ticket/6397)

+   **[sql] [bug] [regression]**

    修复了在`Select`构造的列子句中使用`text()`构造，最好使用`literal_column()`构造，但仍会阻止`union()`等构造正常工作的回归。其他用例，如构建子查询，在先前版本中与以前一样工作，其中`text()`构造被静默地从导出列集合中省略。还修复了 ORM 中类似的用法。

    参考：[#6343](https://www.sqlalchemy.org/trac/ticket/6343)

+   **[sql] [错误] [回归]**

    修复了涉及旧方法（如`Select.append_column()`）的回归，其中内部断言会失败。

    参考：[#6261](https://www.sqlalchemy.org/trac/ticket/6261)

+   **[sql] [错误] [回归]**

    修复了由[#5395](https://www.sqlalchemy.org/trac/ticket/5395)引起的回归，即在使用具有`__iter__()`方法的映射类进行 2.0 风格查询时，回调检查序列在`select()`中现在会导致失败。进一步调整检查以适应这种情况以及其他一些有趣的`__iter__()`场景。

    参考：[#6300](https://www.sqlalchemy.org/trac/ticket/6300)

### 模式

+   **[模式] [错误] [mariadb] [mysql] [oracle] [postgresql]**

    确保 MySQL 和 MariaDB 方言在渲染`AUTO_INCREMENT`关键字时忽略`Identity`构造。

    更新了 Oracle 和 PostgreSQL 编译器，如果数据库版本不支持`Identity`（Oracle < 12 和 PostgreSQL < 10），则不会渲染`Identity`。以前无论数据库版本如何都会渲染。

    参考：[#6338](https://www.sqlalchemy.org/trac/ticket/6338)

### postgresql

+   **[postgresql] [错误]**

    修复了一个很久以前的问题，即`Enum`数据类型在使用`Enum.metadata`将`MetaData`对象传递给`Enum`时，不会继承`MetaData.schema`参数的问题。

    参考：[#6373](https://www.sqlalchemy.org/trac/ticket/6373)

### sqlite

+   **[sqlite] [用例]**

    默认使用`SingletonThreadPool`来处理使用 URI 文件名创建的内存 SQLite 数据库。先前使用的默认池是`NullPool`，阻止了在多个引擎之间共享同一数据库。

    参考：[#6379](https://www.sqlalchemy.org/trac/ticket/6379)

### mssql

+   **[mssql] [bug] [schema]**

    为`sqlalchemy.dialects.mysql.BIT`列添加`TypeEngine.as_generic()`支持，将其映射为`Boolean`。

    参考：[#6345](https://www.sqlalchemy.org/trac/ticket/6345)

+   **[mssql] [bug] [regression]**

    修复了由[#6306](https://www.sqlalchemy.org/trac/ticket/6306)引起的回归问题，该问题为`DateTime(timezone=True)`添加了支持，其中 pyodbc 驱动程序以前的行为是在将时区感知日期插入到时区无关的 DATETIME 列时，隐式删除 tzinfo，导致在将时区感知日期对象插入到时区本地数据库列时导致 SQL Server 错误。

    参考：[#6366](https://www.sqlalchemy.org/trac/ticket/6366)

### orm

+   **[orm] [bug]**

    修复了在与持久对象一起使用时`Session.bulk_save_objects()`中的问题，该问题将无法跟踪主键映射的主键，其中主键列名与属性名不同。

    此更改也已**回溯**至：1.3.25

    参考：[#6392](https://www.sqlalchemy.org/trac/ticket/6392)

+   **[orm] [bug] [caching] [regression]**

    修复了一个关键的回归问题，即绑定参数跟踪在 SQL 缓存系统中的使用可能无法跟踪所有参数的情况，即当包含参数的相同 SQL 表达式在使用 ORM 相关查询时，使用了诸如类继承之类的功能，然后嵌入在一个包含表达式中，该表达式会多次使用相同的表达式，比如 UNION。ORM 将在编译时逐个复制各个 SELECT 语句作为类继承的一部分，然后嵌入在包含语句中时，将无法容纳所有参数。跟踪此条件的逻辑已经调整为适用于多个参数的副本。

    参考：[#6391](https://www.sqlalchemy.org/trac/ticket/6391)

+   **[orm] [bug]**

    修复了两个不同的问题，主要影响`hybrid_property`，这些问题在常见的错误配置方案下会发生，1.3 中会悄悄忽略，而在 1.4 中会失败，其中“expression”实现将返回一个非`ClauseElement`，例如布尔值。对于这两个问题，1.3 的行为是悄悄忽略错误的配置，并最终尝试将值解释为 SQL 表达式，这将导致查询不正确。

    +   修复了属性系统与`hybrid_property`交互的问题，即如果属性的`__clause_element__()`方法返回一个非`ClauseElement`对象，则内部`AttributeError`将导致属性返回`hybrid_property`本身的`expression`函数，因为属性错误是针对名称`.expression`，这将调用`__getattr__()`方法作为后备。现在会显式引发异常。在 1.3 中，非`ClauseElement`会直接返回。

    +   修复了 SQL 参数强制系统中的问题，即将错误类型的对象传递给期望列表达式的方法会失败，如果对象根本不是 SQLAlchemy 对象，例如 Python 函数，在对象不仅被强制转换为绑定值的情况下。再次强调 1.3 没有全面的参数强制系统，因此这种情况也会悄悄通过。

    参考：[#6350](https://www.sqlalchemy.org/trac/ticket/6350)

+   **[orm] [bug]**

    修复了在 ORM 上下文中将`Select`用作子查询时会直接修改`Select`以禁用该对象的急加载，然后如果在顶层执行上下文中重新使用相同的`Select`，则不会急加载的问题。

    参考：[#6378](https://www.sqlalchemy.org/trac/ticket/6378)

+   **[orm] [bug] [regression]**

    修复了新的 autobegin 行为在现有持久对象具有属性更改的情况下失败“自动开始”的问题，这将影响`Session.rollback()`的行为，因为没有创建要回滚的快照。已更新“属性修改”机制以确保“自动开始”，即在持久属性以与调用`Session.add()`时相同的方式更改时发生，这是一个回归，因为在 1.3 中，rollback()方法总是有一个要回滚的事务并且每次都会过期。

    参考：[#6359](https://www.sqlalchemy.org/trac/ticket/6359), [#6360](https://www.sqlalchemy.org/trac/ticket/6360)

+   **[orm] [bug] [regression]**

    修复了 ORM 中的回归，其中使用混合属性指示来自不同实体的表达式会混淆 ORM 中的列标记逻辑，并尝试从另一个类中推导混合的名称，导致属性错误。现在跟踪混合属性的所有者类以及名称。

    参考：[#6386](https://www.sqlalchemy.org/trac/ticket/6386)

+   **[orm] [bug] [regression]**

    修复了混合属性中的回归，其中针对 SQL 函数的混合会在某些情况下在尝试为子查询的`.c`集合生成条目时生成`AttributeError`；在其他情况下，这将影响其在`Query.count()`等情况下的使用。

    参考：[#6401](https://www.sqlalchemy.org/trac/ticket/6401)

+   **[orm] [bug] [dataclasses]**

    调整了对于数据类的声明式扫描，以便当在一个混合类上使用`declared_attr()`的新形式，即将其放在`dataclasses.field()`构造内而不是实际上是类上的描述符属性时，正确地适应目标类被映射为已经映射了该`declared_attr()`的现有映射类的子类的情况，并且因此不应该重新应用到这个类。

    参考：[#6346](https://www.sqlalchemy.org/trac/ticket/6346)

+   **[orm] [bug]**

    修复了（在 1.4 中已弃用的）`ForeignKeyConstraint.copy()`方法的问题，当使用`schema`参数调用时会导致错误。

    参考：[#6353](https://www.sqlalchemy.org/trac/ticket/6353)

### 引擎

+   **[engine] [bug]**

    修复了明确使用`Sequence`会导致包含多个值短语的`Insert`构造的“内联”行为不一致的问题；第一个序列会是内联的，但后续的序列会是“预执行”，导致序列顺序不一致。现在序列表达式完全是内联的。

    参考：[#6361](https://www.sqlalchemy.org/trac/ticket/6361)

### sql

+   **[sql] [bug]**

    修正了“EMPTY IN”表达式不再依赖于使用子查询，因为这导致了一些兼容性和性能问题。对于选定的数据库，新的方法利用了使用返回 NULL 的 IN 表达式，结合通常的“1 != 1”或“1 = 1”表达式，再附加 AND 或 OR。该表达式现在是除了 SQLite 之外的所有后端的默认值，对于旧版本的 SQLite 仍然存在一些关于元组“IN”的兼容性问题。

    第三方方言仍然可以通过实现新的编译器方法`def visit_empty_set_op_expr(self, type_, expand_op)`来覆盖“空集”表达式的呈现方式，该方法优先于现有的`def visit_empty_set_expr(self, element_types)`，后者仍然保留。

    参考：[#6258](https://www.sqlalchemy.org/trac/ticket/6258), [#6397](https://www.sqlalchemy.org/trac/ticket/6397)

+   **[sql] [bug] [regression]**

    修复了在`Select`构造的列子句中使用`text()`构造，最好使用`literal_column()`构造，但仍然会阻止像`union()`这样的构造正常工作的回归问题。其他用例，如构建子查询，在先前版本中与以前一样工作，其中`text()`构造被静默地从导出列的集合中省略。还修复了 ORM 中类似的用法。

    参考：[#6343](https://www.sqlalchemy.org/trac/ticket/6343)

+   **[sql] [bug] [regression]**

    修复了涉及旧方法如`Select.append_column()`的回归问题，其中内部断言会失败。

    参考：[#6261](https://www.sqlalchemy.org/trac/ticket/6261)

+   **[sql] [bug] [regression]**

    修复了由[#5395](https://www.sqlalchemy.org/trac/ticket/5395)引起的回归，即在使用具有`__iter__()`方法的映射类进行 2.0 风格查询时，调整回对`select()`中序列的检查现在会导致失败。进一步调整了检查以适应这种情况以及其他一些有趣的`__iter__()`场景。

    引用：[#6300](https://www.sqlalchemy.org/trac/ticket/6300)

### schema

+   **[schema] [bug] [mariadb] [mysql] [oracle] [postgresql]**

    确保 MySQL 和 MariaDB 方言在渲染`AUTO_INCREMENT`关键字时忽略`Identity`构造。

    Oracle 和 PostgreSQL 编译器已更新，如果数据库版本不支持`Identity`（Oracle < 12 和 PostgreSQL < 10），则不会渲染它。先前无论数据库版本如何都会渲染。

    引用：[#6338](https://www.sqlalchemy.org/trac/ticket/6338)

### postgresql

+   **[postgresql] [bug]**

    修复了一个很久以前的问题，即`Enum`数据类型在使用`Enum.metadata`将对象传递给`Enum`时不会继承`MetaData`对象的`MetaData.schema`参数。

    引用：[#6373](https://www.sqlalchemy.org/trac/ticket/6373)

### sqlite

+   **[sqlite] [usecase]**

    默认使用`SingletonThreadPool`来处理使用 URI 文件名创建的内存中 SQLite 数据库。先前使用的默认池是`NullPool`，这会阻止多个引擎之间共享同一数据库。

    引用：[#6379](https://www.sqlalchemy.org/trac/ticket/6379)

### mssql

+   **[mssql] [bug] [schema]**

    为`sqlalchemy.dialects.mysql.BIT`列添加`TypeEngine.as_generic()`支持，将其映射为`Boolean`。

    引用：[#6345](https://www.sqlalchemy.org/trac/ticket/6345)

+   **[mssql] [bug] [regression]**

    修复了由[#6306](https://www.sqlalchemy.org/trac/ticket/6306)引起的回归，该回归添加了对`DateTime(timezone=True)`的支持，其中 pyodbc 驱动程序以前的行为是在将时区感知日期插入到时区无关的 DATETIME 列时隐式删除 tzinfo，导致在将时区感知日期对象插入到时区本地数据库列时导致 SQL Server 错误。

    引用：[#6366](https://www.sqlalchemy.org/trac/ticket/6366)

## 1.4.11

发布日期：2021 年 4 月 21 日

### orm 声明式

+   **[orm] [declarative] [bug] [regression]**

    修复了一个回归问题，最近对支持 Python 数据类的更改无意中导致 ORM 映射类无法成功覆盖`__new__()`方法。

    参考：[#6331](https://www.sqlalchemy.org/trac/ticket/6331)

### engine

+   **[engine] [bug] [regression]**

    修复了由[#5497](https://www.sqlalchemy.org/trac/ticket/5497)中的更改引起的关键回归，其中连接池的“init”阶段不再在互斥隔离中发生，允许其他线程继续进行未初始化的方言，这可能会影响 SQL 语句的编译。

    参考：[#6337](https://www.sqlalchemy.org/trac/ticket/6337)

### orm 声明式

+   **[orm] [declarative] [bug] [regression]**

    修复了一个回归问题，最近对支持 Python 数据类的更改无意中导致 ORM 映射类无法成功覆盖`__new__()`方法。

    参考：[#6331](https://www.sqlalchemy.org/trac/ticket/6331)

### engine

+   **[engine] [bug] [regression]**

    修复了由[#5497](https://www.sqlalchemy.org/trac/ticket/5497)中的更改引起的关键回归，其中连接池的“init”阶段不再在互斥隔离中发生，允许其他线程继续进行未初始化的方言，这可能会影响 SQL 语句的编译。

    参考：[#6337](https://www.sqlalchemy.org/trac/ticket/6337)

## 1.4.10

发布日期：2021 年 4 月 20 日

### orm

+   **[orm] [usecase]**

    修改了一些在[#6232](https://www.sqlalchemy.org/trac/ticket/6232)中修复的行为，其中`immediateload`加载策略不再进入递归循环；修改是，从 A->bs->B 进行急加载（joinedload、selectinload 或 subqueryload），然后为简单的多对一 B->a->A 声明`immediateload`，当 A/A.bs 的集合加载时，将填充 B->A，因此在分离时这个属性被反向填充。这允许对象在分离时正常工作。

+   **[orm] [bug]**

    修复了新的`with_loader_criteria()`功能中的错误，其中在自定义 lambda 内访问使用`declared_attr()`的混合类属性时，当首次初始化 lambda 可调用时，会发出有关使用未映射的声明属性的警告。现在通过为此 lambda 初始化步骤使用特殊的工具来阻止此警告。

    参考：[#6320](https://www.sqlalchemy.org/trac/ticket/6320)

+   **[orm] [bug] [regression]**

    修复了由“刷新时的预加载器”功能引起的额外回归，该功能在 [#1763](https://www.sqlalchemy.org/trac/ticket/1763) 中添加，刷新操作历史上会设置 `populate_existing`，而给定新功能现在在自动刷新为 false 时会覆盖急加载对象上的挂起更改。对于这种情况，已关闭 populate_existing 标志，并使用更具体的方法确保正确刷新属性。

    参考：[#6326](https://www.sqlalchemy.org/trac/ticket/6326)

+   **[orm] [bug] [result]**

    修复了使用 2.0 风格执行时的问题，该问题在调用 `Result.unique()` 后阻止了使用 `Result.scalar_one()` 或 `Result.scalar_one_or_none()`，对于 ORM 在任何情况下返回单个元素行的情况。

    参考：[#6299](https://www.sqlalchemy.org/trac/ticket/6299)

### sql

+   **[sql] [bug]**

    修复了 SQL 编译器中的问题，其中为 `Values` 构造设置的绑定参数如果在 `CTE` 中，将无法正确地进行位置跟踪，影响支持 VALUES + ctes 并使用位置参数的数据库驱动程序，特别是 SQL Server 以及 asyncpg。此修复还修复了对编译器标志（如 `literal_binds`）的支持。

    参考：[#6327](https://www.sqlalchemy.org/trac/ticket/6327)

+   **[sql] [bug]**

    修复并巩固了关于自定义函数和其他任意表达式构造的问题，这些问题在 SQLAlchemy 的列标记机制中会寻求使用 `str(obj)` 来获取一个字符串表示以用作子查询的 `.c` 集合中的匿名列名。这是一个非常古老的行为，性能不佳且会导致许多问题，因此已经修改为不再执行任何编译，而是在 `FunctionElement` 上建立特定方法来处理这种情况，因为 SQL 函数是唯一会出现这种情况的用例。这种行为的一个效果是，一个没有可推导名称的未标记列表达式将在子查询的 `.c` 集合中被赋予一个以 `"_no_label"` 前缀开头的任意标签；这些先前要么被表示为该表达式的通用字符串化，要么被表示为内部符号。

    参考：[#6256](https://www.sqlalchemy.org/trac/ticket/6256)

### schema

+   **[schema] [bug]**

    修复了`next_value()`未从相应的`Sequence`派生其类型的问题，而是硬编码为`Integer`。现在使用特定的数值类型。

    参考：[#6287](https://www.sqlalchemy.org/trac/ticket/6287)

### mypy

+   **[mypy] [bug]**

    修复了 mypy 插件无法正确解释显式`Mapped`注释与引用字符串名称的`relationship()`结合使用时的问题；正确的注释将被降级为不太具体的注释，导致类型错误。

    参考：[#6255](https://www.sqlalchemy.org/trac/ticket/6255)

### mssql

+   **[mssql] [usecase]**

    当`DateTime.timezone`参数设置为`True`时，现在在使用 DDL 时将使用`DATETIMEOFFSET`列类型而不是`DATETIME`，之前该标志被静默忽略。

    参考：[#6306](https://www.sqlalchemy.org/trac/ticket/6306)

### misc

+   **[bug] [declarative] [regression]**

    修复了调用不存在的注册方法的`instrument_declarative()`。

    参考：[#6291](https://www.sqlalchemy.org/trac/ticket/6291)

### orm

+   **[orm] [usecase]**

    修改了在[#6232](https://www.sqlalchemy.org/trac/ticket/6232)中修复的一些行为，其中`immediateload`加载策略不再进入递归循环；修改是，从 A->bs->B 进行急加载（joinedload、selectinload 或 subqueryload），然后为简单的 manytoone B->a->A 声明`immediateload`，当 A/A.bs 的集合加载时，将填充 B->A，以便在加载 A/A.bs 集合时回填此属性。这允许对象在分离时正常工作。

+   **[orm] [bug]**

    修复了新`with_loader_criteria()`功能中的错误，当在自定义 lambda 中访问使用`declared_attr()`的属性时，会发出关于使用未映射的 declared attr 的警告，当 lambda 可调用对象首次初始化时。现在通过特殊的仪器化步骤阻止了这个警告。

    参考：[#6320](https://www.sqlalchemy.org/trac/ticket/6320)

+   **[orm] [bug] [regression]**

    修复了在[#1763](https://www.sqlalchemy.org/trac/ticket/1763)中添加的“刷新时的预加载器”功能引起的额外回归，其中刷新操作历史上会设置`populate_existing`，鉴于新功能现在在 autoflush 为 false 时会覆盖急加载对象上的挂起更改。对于这种情况，已关闭 populate_existing 标志，并使用更具体的方法来确保正确刷新属性。

    参考：[#6326](https://www.sqlalchemy.org/trac/ticket/6326)

+   **[orm] [bug] [result]**

    修复了在使用 2.0 风格执行时的问题，该问题阻止在调用`Result.unique()`后使用`Result.scalar_one()`或`Result.scalar_one_or_none()`，对于 ORM 在任何情况下返回单个元素行的情况。

    参考：[#6299](https://www.sqlalchemy.org/trac/ticket/6299)

### sql

+   **[sql] [bug]**

    修复了 SQL 编译器中的问题，即为`Values`构造设置的绑定参数如果在`CTE`内部，则不会正确地进行位置跟踪，这会影响支持 VALUES + ctes 并使用位置参数的数据库驱动程序，特别是 SQL Server 以及 asyncpg。修复还修复了对`literal_binds`等编译器标志的支持。

    参考：[#6327](https://www.sqlalchemy.org/trac/ticket/6327)

+   **[sql] [bug]**

    修复并巩固了关于自定义函数和其他任意表达式结构的问题，这些问题在 SQLAlchemy 的列标记机制中会使用`str(obj)`来获取字符串表示形式，以用作子查询的`.c`集合中的匿名列名。这是一个非常古老的行为，性能低下且会导致许多问题，因此已经通过在`FunctionElement`上建立特定方法来处理这种情况，不再执行任何编译，因为 SQL 函数是唯一需要使用的情况。这种行为的一个效果是，一个没有可推导名称的未标记列表达式将在子查询的`.c`集合中被赋予一个以前缀`"_no_label"`开头的任意标签；这些以前要么被表示为该表达式的通用字符串化，要么被表示为内部符号。

    参考：[#6256](https://www.sqlalchemy.org/trac/ticket/6256)

### 模式

+   **[schema] [bug]**

    修复了`next_value()`未从相应的`Sequence`派生其类型，而是硬编码为`Integer`的问题。现在使用特定的数值类型。

    参考：[#6287](https://www.sqlalchemy.org/trac/ticket/6287)

### mypy

+   **[mypy] [bug]**

    修复了一个问题，即 mypy 插件无法正确解释显式`Mapped`注释与通过字符串名称引用类的`relationship()`结合使用时，正确的注释会被降级为一个不太具体的注释，导致类型错误。

    参考：[#6255](https://www.sqlalchemy.org/trac/ticket/6255)

### mssql

+   **[mssql] [usecase]**

    当`DateTime.timezone`参数设置为`True`时，现在在使用时会使用 SQL Server 的`DATETIMEOFFSET`列类型来发出 DDL，而不是在标志被悄悄忽略时使用`DATETIME`。

    参考：[#6306](https://www.sqlalchemy.org/trac/ticket/6306)

### 杂项

+   **[bug] [declarative] [regression]**

    修复了调用不存在的注册方法的`instrument_declarative()`。

    参考：[#6291](https://www.sqlalchemy.org/trac/ticket/6291)

## 1.4.9

发布日期：2021 年 4 月 17 日

### orm

+   **[orm] [usecase]**

    在与混合属性一起支持`synoynm()`的同时，完全设置了 associationproxy，包括可以建立与这些构造链接的同义词，这是以前半显式禁止的行为，但由于它并非在每种情况下都失败，因此已添加了对 assoc proxy 和 hybrids 的显式支持。

    参考：[#6267](https://www.sqlalchemy.org/trac/ticket/6267)

+   **[orm] [performance] [bug] [regression] [sql]**

    修复了一个关键性能问题，即`select()`构造的遍历会遍历由列子句中的每个引用的 FROM 子句的重复乘积；对于具有大量列的一系列嵌套子查询，这可能导致较大的延迟和显著的内存增长。这种遍历被广泛用于各种 SQL 和 ORM 函数，包括当 ORM `Session` 配置为具有“table-per-bind”时；虽然这不是常见用例，但似乎是 Flask-SQLAlchemy 硬编码使用的方式，因此该问题影响了 Flask-SQLAlchemy 用户。已修复遍历以在 FROM 子句上进行唯一化，这实际上是在 1.4 版本之前的架构中隐式发生的。

    参考：[#6304](https://www.sqlalchemy.org/trac/ticket/6304)

+   **[orm] [bug] [regression]**

    修复了一个问题，即映射到`synonym()`的属性无法在列加载器选项中使用，例如`load_only()`。

    参考：[#6272](https://www.sqlalchemy.org/trac/ticket/6272)

### sql

+   **[sql] [bug] [regression]**

    修复了一个问题，即在元组上的空 in 语句在使用`literal_binds=True`选项编译时会导致错误。

    参考：[#6290](https://www.sqlalchemy.org/trac/ticket/6290)

### postgresql

+   **[postgresql] [bug] [regression] [sql]**

    修复了默认和 PostgreSQL 编译器中的参数错误，该错误会干扰 UPDATE..FROM 或 DELETE..FROM..USING 语句，然后将其作为 CTE 从中选择。 

    参考：[#6303](https://www.sqlalchemy.org/trac/ticket/6303)

### orm

+   **[orm] [usecase]**

    已完全支持`synoynm()`与混合属性、关联代理的结合，包括可以建立与这些构造物链接的同义词，这些同义词可以完全工作。这是以前半显式禁止的行为，但由于它并非在每种情况下都失败，因此已添加了对关联代理和混合属性的显式支持。

    参考：[#6267](https://www.sqlalchemy.org/trac/ticket/6267)

+   **[orm] [performance] [bug] [regression] [sql]**

    修复了一个关键性能问题，即`select()`构造的遍历会遍历由列子句中的每个引用的 FROM 子句的重复乘积；对于具有大量列的一系列嵌套子查询，这可能导致较大的延迟和显着的内存增长。这种遍历被广泛用于各种 SQL 和 ORM 函数，包括当 ORM `Session`配置为具有“table-per-bind”时，虽然这不是常见用例，但似乎是 Flask-SQLAlchemy 硬编码使用的方式，因此该问题影响了 Flask-SQLAlchemy 用户。遍历已修复为在 FROM 子句上进行唯一化，这实际上是在 1.4 版本之前的架构中隐式发生的。

    参考：[#6304](https://www.sqlalchemy.org/trac/ticket/6304)

+   **[orm] [bug] [regression]**

    修复了一个回归问题，其中映射到`synonym()`的属性无法在列加载器选项中使用，例如`load_only()`。

    参考：[#6272](https://www.sqlalchemy.org/trac/ticket/6272)

### sql

+   **[sql] [bug] [regression]**

    修复了一个回归问题，其中对元组的空 in 语句在使用选项`literal_binds=True`编译时会导致错误。

    参考：[#6290](https://www.sqlalchemy.org/trac/ticket/6290)

### postgresql

+   **[postgresql] [bug] [regression] [sql]**

    修复了默认和 PostgreSQL 编译器中的参数错误，该错误会干扰 UPDATE..FROM 或 DELETE..FROM..USING 语句，然后作为 CTE 从中选择。

    参考：[#6303](https://www.sqlalchemy.org/trac/ticket/6303)

## 1.4.8

发布日期：2021 年 4 月 15 日

### orm

+   **[orm] [bug] [regression]**

    修复了涉及`with_expression()`加载器选项的缓存泄漏问题，其中给定的 SQL 表达式不会被正确地视为缓存键的一部分。

    此外，修复了涉及对应`query_expression()`功能的回归。虽然这个 bug 在 1.3 版本中也存在，但直到 1.4 版本才暴露出来。当不需要时，“null()`的“默认表达式”值会被渲染，而且在 ORM 重写语句时也没有正确适应，比如在使用连接式急加载时。修复确保“单例”表达式如`NULL`和`true`不会被“适应”为引用 ORM 语句中的列，并且还确保如果没有使用`with_expression()`，那么没有默认表达式的`query_expression()`不会在语句中渲染。

    参考：[#6259](https://www.sqlalchemy.org/trac/ticket/6259)

+   **[orm] [bug]**

    修复了由[#1763](https://www.sqlalchemy.org/trac/ticket/1763)引入的`Session.refresh()`新功能中的问题，其中也会刷新急加载的关系，`lazy="raise"`和`lazy="raise_on_sql"`加载策略会干扰`immediateload()`加载策略，从而破坏了使用`selectinload()`、`subqueryload()`加载的关系的功能。

    参考：[#6252](https://www.sqlalchemy.org/trac/ticket/6252)

### engine

+   **[engine] [bug]**

    当传递非 Connection 对象时，`Dialect.has_table()` 方法现在会引发一个信息性异常，因为这种不正确的行为似乎很常见。此方法不打算在方言之外的外部使用。请使用`Inspector.has_table()` 方法，或者为了与旧版本的 SQLAlchemy 兼容性，使用`Engine.has_table()`方法。

### sql

+   **[sql] [feature]**

    `CursorResult.inserted_primary_key` 返回的元组现在是一个带有命名元组接口的`Row`对象。

    参考：[#3314](https://www.sqlalchemy.org/trac/ticket/3314)

+   **[sql] [bug] [regression]**

    修复了一个回归问题，即如果从内部克隆操作或 pickle 操作中复制`BindParameter`对象，并且参数名称包含空格或其他特殊字符，则该对象在 IN 表达式（即在 1.4 中使用“后编译”功能）中将无法正确呈现。

    参考：[#6249](https://www.sqlalchemy.org/trac/ticket/6249)

+   **[sql] [bug] [regression] [sqlite]**

    修复了一个回归问题，即在一些后端不支持“INSERT… VALUES (DEFAULT)”但支持“INSERT..DEFAULT VALUES”的情况下，引入 INSERT 语法“INSERT… VALUES (DEFAULT)”不受支持，包括 SQLite。现在每个方言都分别支持或不支持这两种语法，例如 MySQL 支持“VALUES (DEFAULT)”但不支持“DEFAULT VALUES”。还启用了对 Oracle 的支持。

    参考：[#6254](https://www.sqlalchemy.org/trac/ticket/6254)

### mypy

+   **[mypy] [change]**

    更新了 Mypy 插件，只使用语义分析器的公共插件接口。

+   **[mypy] [bug]**

    修订了对版本 1.4.7 中`OrderingList`的修复，该修复测试针对不正确的 API。

    参考：[#6205](https://www.sqlalchemy.org/trac/ticket/6205)

### asyncio

+   **[asyncio] [bug]**

    修复了一个拼写错误，阻止将`AsyncSession`的`bind`属性设置为正确值。

    参考：[#6220](https://www.sqlalchemy.org/trac/ticket/6220)

### mssql

+   **[mssql] [bug] [regression]**

    修复了与[#6173](https://www.sqlalchemy.org/trac/ticket/6173)、[#6184](https://www.sqlalchemy.org/trac/ticket/6184)相同区域的另一个回归问题，在 SQL Server 中使用 OFFSET 的值为 0 与 LIMIT 结合使用时，会创建一个使用“TOP”语句的语句，这是 1.3 中的行为，但由于缓存的原因，无法正确响应其他 OFFSET 值。如果“0”不是第一个，那么就没问题。为了修复，现在只有在完全省略 OFFSET 值时才会发出“TOP”语法，也就是，不使用`Select.offset()`。请注意，此更改现在要求如果使用“with_ties”或“percent”修饰符，则语句不能指定零的 OFFSET，现在必须完全省略。

    参考：[#6265](https://www.sqlalchemy.org/trac/ticket/6265)

### orm

+   **[orm] [bug] [regression]**

    修复了涉及`with_expression()`加载器选项的缓存泄漏问题，其中给定的 SQL 表达式不会被正确地视为缓存键的一部分。

    另外，修复了涉及相应`query_expression()`特性的回归。虽然该错误在 1.3 版本中也存在，但直到 1.4 版本才暴露出来。当不需要时，“null()`的“默认表达式”值会被渲染，此外，当 ORM 重写语句时也不会被正确地调整，例如在使用连接急切加载时。修复确保“单例”表达式如`NULL`和`true`不会被“调整”以引用 ORM 语句中的列，并确保如果未使用`with_expression()`，则不会在语句中呈现没有默认表达式的`query_expression()`。

    引用：[#6259](https://www.sqlalchemy.org/trac/ticket/6259)

+   **[orm] [错误]**

    修复了由[#1763](https://www.sqlalchemy.org/trac/ticket/1763)引入的`Session.refresh()`新特性中的问题，其中急切加载的关系也会被刷新，`lazy="raise"`和`lazy="raise_on_sql"`加载策略会干扰`immediateload()`加载策略，从而破坏了使用`selectinload()`、`subqueryload()`加载的关系的特性。

    引用：[#6252](https://www.sqlalchemy.org/trac/ticket/6252)

### 引擎

+   **[engine] [错误]**

    `Dialect.has_table()`方法现在会在传递非连接对象时引发一个信息性异常，因为这种不正确的行为似乎很常见。此方法不打算在方言之外的外部使用。请使用`Inspector.has_table()`方法，或者为了与旧版本的 SQLAlchemy 跨兼容性，使用`Engine.has_table()`��法。

### sql

+   **[sql] [特性]**

    `CursorResult.inserted_primary_key`返回的元组现在是一个带有命名元组接口的`Row`对象，位于现有元组接口之上。

    引用：[#3314](https://www.sqlalchemy.org/trac/ticket/3314)

+   **[sql] [错误] [回归]**

    修复了一个问题，即如果`BindParameter`对象从内部克隆操作或 pickle 操作中复制，并且参数名称包含空格或其他特殊字符，则在 IN 表达式（即使用 1.4 中的“后编译”功能）中，该对象不会正确地呈现。

    参考：[#6249](https://www.sqlalchemy.org/trac/ticket/6249)

+   **[sql] [bug] [regression] [sqlite]**

    修复了一个问题，即引入 INSERT 语法“INSERT… VALUES (DEFAULT)”不受支持的回归，其中一些后端不支持“INSERT..DEFAULT VALUES”，包括 SQLite。现在，每种方言都支持或不支持这两种语法中的每一种，例如 MySQL 支持“VALUES (DEFAULT)”但不支持“DEFAULT VALUES”。还启用了对 Oracle 的支持。

    参考：[#6254](https://www.sqlalchemy.org/trac/ticket/6254)

### mypy

+   **[mypy] [change]**

    更新了 Mypy 插件，仅使用语义分析器的公共插件接口。

+   **[mypy] [bug]**

    修订了版本 1.4.7 中针对`OrderingList`的修复，该修复针对的是错误的 API。

    参考：[#6205](https://www.sqlalchemy.org/trac/ticket/6205)

### asyncio

+   **[asyncio] [bug]**

    修复了一个拼写错误，导致无法将`AsyncSession`的`bind`属性设置为正确的值。

    参考：[#6220](https://www.sqlalchemy.org/trac/ticket/6220)

### mssql

+   **[mssql] [bug] [regression]**

    修复了与[#6173](https://www.sqlalchemy.org/trac/ticket/6173)、[#6184](https://www.sqlalchemy.org/trac/ticket/6184)相同区域的另一个回归，其中在与 SQL Server 一起使用 OFFSET 为 0 的值与 LIMIT 时，会创建使用“TOP”的语句，这与 1.3 中的行为相同，但由于缓存而无法相应地响应其他 OFFSET 值。如果“0”不是第一个，那么就没问题。对于修复，只有在完全省略 OFFSET 值时才发出“TOP”语法，也就是，不使用`Select.offset()`。请注意，此更改现在要求如果使用“with_ties”或“percent”修饰符，则语句不能指定零的 OFFSET，现在必须完全省略它。

    参考：[#6265](https://www.sqlalchemy.org/trac/ticket/6265)

## 1.4.7

发布日期：2021 年 4 月 9 日

### orm

+   **[orm] [bug] [regression]**

    修复了一个问题，即当`subqueryload()`加载策略无法正确适应子选项时出现回归，例如在列上使用`defer()`选项，如果子查询加载的“路径”超过一级深度。

    参考：[#6221](https://www.sqlalchemy.org/trac/ticket/6221)

+   **[orm] [bug] [regression]**

    修复了一个回归问题，`merge_frozen_result()`函数依赖于 dogpile.caching 示例中未包含在测试中并由于内部参数不正确而开始失败的问题。

    参考：[#6211](https://www.sqlalchemy.org/trac/ticket/6211)

+   **[orm] [bug] [regression]**

    修复了一个关键的回归问题，即当在没有现有事务的情况下发生刷新时，`Session`可能无法“自动开始”新事务，将`Session`隐式放置在传统的自动提交模式中，提交事务。现在，`Session`具有一个检查，将防止发生此条件，同时修复刷新问题。

    此外，将作为[#5226](https://www.sqlalchemy.org/trac/ticket/5226)的一部分进行的更改的一部分缩减，可以在未过期操作期间运行自动刷新，以便在使用传统`Session.autocommit`模式的`Session`中实际上不执行此操作，因为这会在刷新操作中引发提交。

    参考：[#6233](https://www.sqlalchemy.org/trac/ticket/6233)

+   **[orm] [bug] [regression]**

    修复了一个回归问题，其中 ORM 编译方案假定混合属性的函数名称与结果元组中每个元素的正确名称相同，导致在尝试确定每个元素的正确名称时引发`AttributeError`，在 1.3 中存在类似问题，但仅影响元组行的名称。此处的修复添加了一个检查，即在分配此名称之前，混合函数的函数名称实际上存在于类或其超类的`__dict__`中；否则，将认为混合属性是“未命名的”，ORM 结果元组将使用底层表达式的命名方案。

    参考：[#6215](https://www.sqlalchemy.org/trac/ticket/6215)

+   **[orm] [bug] [regression]**

    修复了由作为[#1763](https://www.sqlalchemy.org/trac/ticket/1763)的一部分添加的新功能引起的关键回归问题，即在未过期操作中调用了急切加载器。新功能使用“immediateload”急切加载器策略作为集合加载策略的替代品，与其他“post-load”策略不同，该策略未考虑到相互依赖关系之间的递归调用，导致递归溢出错误。

    参考：[#6232](https://www.sqlalchemy.org/trac/ticket/6232)

### 引擎

+   **[engine] [bug] [regression]**

    修复了当对`Row`对象使用字典访问时的行为，即通过`dict(row)`转换为字典或使用字符串或其他对象访问成员，例如`row["some_key"]`，它的工作方式与字典相同，而不像元组那样引发`TypeError`，无论是否存在 C 扩展。最初，这应该对“非未来”情况下使用`LegacyRow`发出 2.0 弃用警告，并且对于“未来”`Row`类应该引发`TypeError`。但是，`Row`的 C 版本未能引发此`TypeError`，并且为了使事情变得更加复杂，`Session.execute()`方法现在在所有情况下返回`Row`以保持与 ORM 结果情况的一致性，因此未安装 C 扩展的用户将在这一情况下看到现有的 1.4 之前样式代码的不同行为。

    因此，为了缓和整体升级方案，因为大多数用户尚未接触到`Row`在 1.4.6 之前更严格的行为，`LegacyRow`和`Row`都提供了字符串键访问以及对`dict(row)`的支持，在所有情况下，当启用`SQLALCHEMY_WARN_20`时都会发出 2.0 弃用警告。`Row`对象仍然使用类似元组的行为来进行`__contains__`，这可能是与`LegacyRow`相比唯一显著的行为变化，除了删除字典样式方法`values()`和`items()`之外。

    参考：[#6218](https://www.sqlalchemy.org/trac/ticket/6218)

### sql

+   **[sql] [bug] [regression]**

    增强了用于`ColumnOperators.in_()`操作的“扩展”功能，以从右侧元素列表中推断表达式的类型，如果左侧没有设置任何显式类型。这允许表达式支持字符串化等功能。在 1.3 中，“扩展”不会自动用于`ColumnOperators.in_()`表达式，因此从这个意义上说，这个更改修复了行为回归。

    参考：[#6222](https://www.sqlalchemy.org/trac/ticket/6222)

+   **[sql] [bug]**

    修复了“字符串化”编译器，以支持“multirow” INSERT 语句的基本字符串化，即在 VALUES 关键字之后有多个元组。

### 模式

+   **[模式] [错误] [回归]**

    修复了使用 `Connection.execution_options.schema_translate_map` 字典中包含大括号等特殊字符的令牌的回归问题，这些字符可能无法正确替换。现在显式禁止使用方括号字符 `[]`，因为这些字符用作当前实现中的分隔符。

    参考：[#6216](https://www.sqlalchemy.org/trac/ticket/6216)

### mypy

+   **[mypy] [错误]**

    修复了 Mypy 插件中的问题，即插件未正确推断不直接从 `TypeEngine` 下降的子类的列的类型，特别是 `TypeDecorator` 和 `UserDefinedType` 的类型。

### 测试

+   **[测试] [更改]**

    在 `DefaultDialect` 中添加了一个名为 `supports_schemas` 的新标志；第三方方言可以将此标志设置为 `False`，以在运行第三方方言的测试套件时禁用 SQLAlchemy 的模式级测试。

### orm

+   **[orm] [错误] [回归]**

    修复了 `subqueryload()` 加载器策略的回归问题，如果子查询加载的“路径”超过一级，则无法正确适应子选项，例如在列上的 `defer()` 选项。

    参考：[#6221](https://www.sqlalchemy.org/trac/ticket/6221)

+   **[orm] [错误] [回归]**

    修复了 `merge_frozen_result()` 函数的回归问题，在 dogpile.caching 示例中依赖的函数未包含在测试中，并由于内部参数错误而开始失败。

    参考：[#6211](https://www.sqlalchemy.org/trac/ticket/6211)

+   **[orm] [错误] [回归]**

    修复了关键的回归问题，即在没有现有事务的情况下发生刷新时，`Session` 可能无法“自动开始”新事务，将 `Session` 隐式放置到旧式自动提交模式中，提交事务。 `Session` 现在具有检查，将防止此条件发生，同时修复刷新问题。

    另外，缩减了作为[#5226](https://www.sqlalchemy.org/trac/ticket/5226)的一部分所做的更改，该更改可以在取消过期操作期间运行自动刷新，以避免在使用传统`Session`的情况下实际执行此操作`Session.autocommit`模式，因为这会在刷新操作中引发提交。

    参考：[#6233](https://www.sqlalchemy.org/trac/ticket/6233)

+   **[orm] [bug] [regression]**

    修复了 ORM 编译方案假定混合属性的函数名称与结果元组中每个元素的正确名称相同的回归问题，当它尝试确定结果元组中每个元素的正确名称时，将引发`AttributeError`。1.3 中存在类似问题，但仅影响元组行的名称。此处的修复添加了一个检查，即混合属性的函数名称实际上存在于类或其超类的`__dict__`中，然后再分配此名称；否则，该混合属性被视为“无名称”，ORM 结果元组将使用底层表达式的命名方案。

    参考：[#6215](https://www.sqlalchemy.org/trac/ticket/6215)

+   **[orm] [bug] [regression]**

    修复了由于添加作为[#1763](https://www.sqlalchemy.org/trac/ticket/1763)的新功能而引起的关键回归问题，急切加载器在取消过期操作时被调用。新功能利用了“immediateload”急切加载器策略作为集合加载策略的替代，与其他“post-load”策略不同，它没有考虑到相互依赖关系之间的递归调用，导致递归溢出错误。

    参考：[#6232](https://www.sqlalchemy.org/trac/ticket/6232)

### engine

+   **[engine] [bug] [regression]**

    修复了当对`Row`对象使用字典访问时的行为，即通过`dict(row)`转换为字典或使用字符串或其他对象访问成员，例如`row["some_key"]`，它的行为与字典一样，而不是像元组一样引发`TypeError`，无论是否存在 C 扩展。最初，这应该对“非未来”情况下使用`LegacyRow`发出 2.0 弃用警告，并且对于“未来”`Row`类应该引发`TypeError`。然而，`Row`的 C 版本未能引发此`TypeError`，并且为了使事情变得更加复杂，`Session.execute()`方法现在在所有情况下返回`Row`以保持与 ORM 结果情况的一致性，因此未安装 C 扩展的用户将在这一情况下看到现有的 1.4 之前风格代码的不同行为。

    因此，为了缓解整体升级方案，因为大多数用户尚未接触到`Row`在 1.4.6 之前更严格的行为，`LegacyRow`和`Row`都提供了字符串键访问以及对`dict(row)`的支持，在所有情况下启用`SQLALCHEMY_WARN_20`时都会发出 2.0 弃用警告。`Row`对象仍然使用类似元组的行为来进行`__contains__`，这可能是与`LegacyRow`相比唯一显着的行为变化，除了删除字典样式方法`values()`和`items()`之外。

    参考：[#6218](https://www.sqlalchemy.org/trac/ticket/6218)

### sql

+   **[sql] [bug] [regression]**

    增强了用于`ColumnOperators.in_()`操作的“扩展”功能，以从右侧元素列表推断表达式的类型，如果左侧没有设置任何显式类型。这允许表达式支持字符串化等功能。在 1.3 中，“扩展”不会自动用于`ColumnOperators.in_()`表达式，因此从这个意义上说，此更改修复了行为回归。

    参考：[#6222](https://www.sqlalchemy.org/trac/ticket/6222)

+   **[sql] [bug]**

    修复了“stringify”编译器，以支持“multirow” INSERT 语句的基本字符串化，即在 VALUES 关键字后跟随多个元组的语句。 

### 模式

+   **[模式] [错误] [回归]**

    修复了一个回归，其中在`Connection.execution_options.schema_translate_map`字典中使用包含大括号等特殊字符的令牌会导致替换不正确。当前实现中现在明确禁止使用方括号字符`[]`，因为它们被用作分隔符字符。

    参考：[#6216](https://www.sqlalchemy.org/trac/ticket/6216)

### mypy

+   **[mypy] [错误]**

    修复了 Mypy 插件中的问题，插件未能正确推断不直接从`TypeEngine`继承的子类的列的正确类型，特别是`TypeDecorator`和`UserDefinedType`的列。

### 测试

+   **[测试] [更改]**

    在`DefaultDialect`中添加了一个名为`supports_schemas`的新标志；第三方方言可以将此标志设置为`False`，以在运行第三方方言的测试套件时禁用 SQLAlchemy 的模式级测试。

## 1.4.6

发布日期：2021 年 4 月 6 日

### ORM

+   **[ORM] [错误] [回归]**

    修复了一个回归，其中在单个`Query.join()`调用中使用了一系列实体进行连接，而没有任何 ON 子句，将无法正确运行。

    参考：[#6203](https://www.sqlalchemy.org/trac/ticket/6203)

+   **[ORM] [错误] [回归]**

    修复了关键的回归，其中 ORM 中的`Query.yield_per()`方法设置内部`Result`以一次产生一块，但是使用了新的`Result.unique()`方法，该方法在整个结果中唯一化。这将导致丢失行，因为 ORM 使用`id(obj)`作为唯一化函数，这导致新对象的重复标识符被垃圾回收。 1.3 在这里的行为是在每个块中“唯一化”，当结果以块形式产生时，实际上不会产生“唯一化”的结果。由于当存在连接的急加载时已明确禁止使用`Query.yield_per()`方法，这是“唯一化”功能的主要原因，因此当使用`Query.yield_per()`时，“唯一化”功能现在完全关闭。

    这个回归仅适用于传统的`Query`对象；在使用 2.0 风格执行时，不会自动应用“唯一化”。为了防止由于显式使用`Result.unique()`而引起的问题，如果从“唯一化”ORM 级别的`Result`中获取了行，并且还使用了任何 yield per API，则现在会引发错误，因为`yield_per`的目的是允许任意大量的行，而不会在内存中唯一化，而不会增加条目数以适应完整的结果大小。

    参考：[#6206](https://www.sqlalchemy.org/trac/ticket/6206)

### sql

+   **[sql] [bug] [mssql] [oracle] [regression]**

    修复了与 1.4.5 发布的[#6173](https://www.sqlalchemy.org/trac/ticket/6173)相同领域中进一步的回归，其中“postcompile”参数再次最常用于 Oracle 和 SQL Server 中的 LIMIT/OFFSET 渲染，如果相同参数在语句中的多个位置呈现，则无法正确处理。

    参考：[#6202](https://www.sqlalchemy.org/trac/ticket/6202)

+   **[sql] [bug]**

    使用 `Connection.execute()` 执行 `Subquery` 现在已被弃用，并将发出弃用警告；这种用例是一个疏忽，应该在 1.4 版本中删除。该操作现在将直接执行底层的 `Select` 对象以保持向后兼容性。同样，`CTE` 类也不适合执行。在 1.3 版本中，尝试执行 CTE 将导致执行一个无效的“空”SQL 语句；由于这种用例无法正常工作，现在会引发 `ObjectNotExecutableError`。以前，1.4 版本尝试将 CTE 作为语句执行，但它只能偶尔工作。

    参考：[#6204](https://www.sqlalchemy.org/trac/ticket/6204)

### schema

+   **[schema] [bug]**

    如果 `Table` 对象在实例化时没有传递至少 `Table.name` 和 `Table.metadata` 参数的位置参数，则现在会引发一个信息性错误消息。以前，如果这些参数作为关键字参数传递，对象会悄悄地初始化失败。

    此更改也已**回溯**至：1.3.25

    参考：[#6135](https://www.sqlalchemy.org/trac/ticket/6135)

### mypy

+   **[mypy] [bug]**

    应用了一系列重构和修复来适应 Mypy 的“增量”模式跨多个文件，之前没有考虑到这一点。在这种模式下，Mypy 插件必须适应其他文件中表达的 Python 数据类型，这些数据类型比直接运行时提供的信息少。

    此外，添加了一个新的装饰器 `declarative_mixin()`，这对于 Mypy 插件能够明确识别一个在特定 Python 文件中未使用的 Declarative mixin 类是必要的。

    另请参阅

    使用 @declared_attr 和 Declarative Mixins

    参考：[#6147](https://www.sqlalchemy.org/trac/ticket/6147)

+   **[mypy] [bug]**

    修复了一个问题，即如果关系的“collection_class”是可调用的而不是类，则 Mypy 插件将无法解释它。还改进了集合导向关系的类型匹配和错误报告。

    参考：[#6205](https://www.sqlalchemy.org/trac/ticket/6205)

### asyncio

+   **[asyncio] [usecase] [postgresql]**

    在 asyncpg DBAPI 适配器引发的 SQLAlchemy 异常类的 `.orig` 属性中添加了访问器 `.sqlstate` 和同义词 `.pgcode`，即在 asyncpg 库本身引发的异常对象之上包装的中间异常对象，但低于 SQLAlchemy 方言的级别。

    参考：[#6199](https://www.sqlalchemy.org/trac/ticket/6199)

### orm

+   **[orm] [bug] [regression]**

    修复了一个回归，即在单个 `Query.join()` 调用中使用了一系列实体进行联接而没有任何 ON 子句的过时形式，将无法正确运行。

    参考：[#6203](https://www.sqlalchemy.org/trac/ticket/6203)

+   **[orm] [bug] [regression]**

    修复了一个关键的回归，即 ORM 中的 `Query.yield_per()` 方法会设置内部的 `Result` 以一次产生一块数据，但是使用了新的 `Result.unique()` 方法，该方法在整个结果集上进行唯一化。这会导致丢失行，因为 ORM 使用 `id(obj)` 作为唯一化函数，这会导致新对象的重复标识符被垃圾回收。1.3 版本在这里的行为是在每个块上进行“唯一化”，当结果以块形式产生时，实际上并不会产生“唯一化”的结果。由于当存在联接的急加载时已明确禁止使用 `Query.yield_per()` 方法，这是“唯一化”功能的主要原因，因此当使用 `Query.yield_per()` 时，“唯一化”功能现在完全关闭。

    此回归仅适用于传统的`Query`对象；在使用 2.0 风格执行时，不会自动应用“唯一性”。为了防止由于显式使用`Result.unique()`而引起的问题，如果从“唯一化”ORM 级别的`Result`中获取行，并且同时使用了任何 yield per API，则现在会引发错误，因为`yield_per`的目的是允许处理任意大量的行，而在内存中无法唯一化而不增加条目数以适应完整的结果大小。

    参考：[#6206](https://www.sqlalchemy.org/trac/ticket/6206)

### SQL

+   **[sql] [bug] [mssql] [oracle] [regression]**

    在 1.4.5 版本中发布的[#6173](https://www.sqlalchemy.org/trac/ticket/6173)中修复了与同一区域的进一步回归，其中“postcompile”参数，通常用于 Oracle 和 SQL Server 中的 LIMIT/OFFSET 渲染，如果相同参数在语句中的多个位置呈现，则无法正确处理。

    参考：[#6202](https://www.sqlalchemy.org/trac/ticket/6202)

+   **[sql] [bug]**

    使用`Connection.execute()`执行`Subquery`已被弃用，并将发出弃用警告；这种用法是一个疏忽，应该在 1.4 版本中删除。该操作现在将直接执行底层的`Select`对象以保持向后兼容性。同样，`CTE`类也不适合执行。在 1.3 版本中，尝试执行 CTE 将导致执行一个无效的“空白”SQL 语句；由于这种用法不起作用，现在会引发`ObjectNotExecutableError`。此前，1.4 版本尝试将 CTE 作为语句执行，但其工作不稳定。

    参考：[#6204](https://www.sqlalchemy.org/trac/ticket/6204)

### 模式

+   **[schema] [bug]**

    如果`Table`对象在实例化时没有至少传递`Table.name`和`Table.metadata`参数的位置参数，则现在会引发一个信息性错误消息。以前，如果这些作为关键字参数传递，对象将无声地无法正确初始化。

    此更改也**回溯**到：1.3.25

    参考：[#6135](https://www.sqlalchemy.org/trac/ticket/6135)

### mypy

+   **[mypy] [bug]**

    应用了一系列重构和修复以适应 Mypy 的“增量”模式跨多个文件，以前没有考虑到这一点。在这种模式下，Mypy 插件必须适应在其他文件中表达的 Python 数据类型，这些数据类型带有比直接运行时更少的信息。

    此外，添加了一个新的装饰器`declarative_mixin()`，这对于 Mypy 插件能够明确识别否则在特定 Python 文件中未使用的声明性 mixin 类是必要的。

    另请参见

    使用@declared_attr 和声明性 Mixin

    参考：[#6147](https://www.sqlalchemy.org/trac/ticket/6147)

+   **[mypy] [bug]**

    修复了 Mypy 插件无法解释关系的“collection_class”如果它是可调用的而不是类的问题。还改进了集合导向关系的类型匹配和错误报告。

    参考：[#6205](https://www.sqlalchemy.org/trac/ticket/6205)

### asyncio

+   **[asyncio] [usecase] [postgresql]**

    向 SQLAlchemy 异常类的`.orig`属性添加了访问器`.sqlstate`和同义词`.pgcode`，这些异常类由 asyncpg DBAPI 适配器引发，即在 asyncpg 库本身引发的异常之上包装的中间异常对象，但低于 SQLAlchemy 方言的级别。

    参考：[#6199](https://www.sqlalchemy.org/trac/ticket/6199)

## 1.4.5

发布日期：2021 年 4 月 2 日

### orm

+   **[orm] [bug] [regression]**

    修复了`joinedload()`加载策略无法成功加载到针对`CTE`构造的映射器的回归问题。

    参考：[#6172](https://www.sqlalchemy.org/trac/ticket/6172)

+   **[orm] [bug] [regression]**

    缩减了在[#5171](https://www.sqlalchemy.org/trac/ticket/5171)中添加的警告消息，不再警告在继承场景中存在重叠列的情况，其中特定关系局限于子类，因此不代表重叠。

    参考：[#6171](https://www.sqlalchemy.org/trac/ticket/6171)

### sql

+   **[sql] [bug] [postgresql]**

    修复了新的`FunctionElement.render_derived()`功能中的错误，其中在别名 SQL 中显式呈现的列名不会对大小写敏感的名称和其他非字母数字名称应用正确的引用。

    参考：[#6183](https://www.sqlalchemy.org/trac/ticket/6183)

+   **[sql] [bug] [regression]**

    修复了使用`Operators.in_()`方法与非绑定表列的`Select`对象或者更一般地使用没有数据类型的`ScalarSelect`在二进制表达式中会产生`AttributeError`的回归问题。

    参考：[#6181](https://www.sqlalchemy.org/trac/ticket/6181)

+   **[sql] [bug]**

    在`Dialect`类中添加了一个名为`Dialect.supports_statement_cache`的新标志。现在，为了使 SQLAlchemy 的查询缓存对该方言生效，该标志现在需要直接存在于方言类中。这一决定基于发现的问题，例如[#6173](https://www.sqlalchemy.org/trac/ticket/6173)揭示了那些硬编码来自编译语句的文字值的方言，通常是用于 LIMIT / OFFSET 的数字参数，直到这些方言被修改为仅使用语句中存在的参数，才能与缓存兼容。对于未应用此标志的第三方方言，SQL 日志将显示消息“方言不支持缓存”，表明该方言应在验证在编译阶段没有每个语句的文字值被呈现后，应该应用此标志。

    另请参见

    第三方方言的缓存

    参考：[#6184](https://www.sqlalchemy.org/trac/ticket/6184)

### 模式

+   **[schema] [bug]**

    在`Enum`类型中引入了一个新参数`Enum.omit_aliases`，允许在使用 pep435 Enum 时过滤别名。之前的 SQLAlchemy 版本在所有情况下都保留别名，创建具有额外状态的数据库枚举类型，这意味着它们在数据库中被视为不同的值。为了向后兼容，该标志在 1.4 系列中默认为`False`，但将在将来的版本中切换为`True`。如果未指定此标志并且传递的枚举包含别名，则会引发弃用警告。

    参考：[#6146](https://www.sqlalchemy.org/trac/ticket/6146)

### mypy

+   **[mypy] [bug]**

    修复了 mypy 插件中的问题，其中对`as_declarative()`的新添加支持需要更充分地向 mypy 解释器的状态中添加`DeclarativeMeta`类，以免导致名称未找到错误；此外，改进了插件设置全局名称的方式，包括`Mapped`名称。

    引用：[#sqlalchemy/sqlalchemy2-stubs/#14](https://www.sqlalchemy.org/trac/ticket/sqlalchemy/sqlalchemy2-stubs/#14)

### asyncio

+   **[asyncio] [错误]**

    修复了在运行 Python 3.6 并安装了`contextvars`的后端库时无法加载 asyncio 扩展的问题。

    引用：[#6166](https://www.sqlalchemy.org/trac/ticket/6166)

### postgresql

+   **[postgresql] [错误] [回归]**

    修复了由[#6023](https://www.sqlalchemy.org/trac/ticket/6023)引起的回归，其中在使用 psycopg2 时，对`ARRAY`中的元素应用 PostgreSQL 转换操作符将无法在数据类型嵌入在`Variant`适配器实例中的情况下使用正确的类型。

    此外，修复了使用`Variant(ARRAY(some_schema_type))`时正确发出`CREATE TYPE`的支持。

    此更改还**回溯**到：1.3.25

    引用：[#6182](https://www.sqlalchemy.org/trac/ticket/6182)

+   **[postgresql] [错误]**

    修复了在 1.4.4 中发布的[#6099](https://www.sqlalchemy.org/trac/ticket/6099)的修复中的拼写错误，完全阻止了这个更改的正确工作，即错误消息与实际由 pg8000 发出的消息不匹配。

    引用：[#6099](https://www.sqlalchemy.org/trac/ticket/6099)

+   **[postgresql] [错误]**

    修复了对 PostgreSQL `PGInspector`的错误，当针对“future”样式引擎而不是直接连接时，会导致`.get_enums()`、`.get_view_names()`、`.get_foreign_table_names()`和`.get_table_oid()`失败。

    引用：[#6170](https://www.sqlalchemy.org/trac/ticket/6170)

### mysql

+   **[mysql] [错误] [回归]**

    修复了 MySQL 方言中的回归，其中用于检测表是否存在的反射查询会在非常旧的 MySQL 5.0 和 5.1 版本上失败。

    引用：[#6163](https://www.sqlalchemy.org/trac/ticket/6163)

### mssql

+   **[mssql] [错误]**

    修复了在 MSSQL 2012+中的回归，当在子查询中使用`offset=0`时，无法呈现 order by 子句。

    引用：[#6163](https://www.sqlalchemy.org/trac/ticket/6163)

### oracle

+   **[oracle] [错误] [回归]**

    修复了 Oracle 编译器由于缓存问题无法维护 SELECT 中 LIMIT/OFFSET 的正确参数值的严重回归问题。

    参考：[#6173](https://www.sqlalchemy.org/trac/ticket/6173)

### orm

+   **[orm] [bug] [regression]**

    修复了`joinedload()`加载策略不能成功地加载到针对`CTE`构造的映射器的回归问题。

    参考：[#6172](https://www.sqlalchemy.org/trac/ticket/6172)

+   **[orm] [bug] [regression]**

    缩小了在[#5171](https://www.sqlalchemy.org/trac/ticket/5171)中添加的警告消息，不会对继承场景中的重叠列发出警告，其中特定关系是局部于子类的，因此不代表重叠。 

    参考：[#6171](https://www.sqlalchemy.org/trac/ticket/6171)

### sql

+   **[sql] [bug] [postgresql]**

    修复了新的`FunctionElement.render_derived()`功能中列名在别名 SQL 中显式呈现时不适用于区分大小写名称和其他非字母数字名称的适当引用的错误。

    参考：[#6183](https://www.sqlalchemy.org/trac/ticket/6183)

+   **[sql] [bug] [regression]**

    修复了使用`Operators.in_()`方法与非表绑定列相对应的`Select`对象会产生`AttributeError`的回归问题，或者更一般地使用没有数据类型的`ScalarSelect`在二进制表达式中会产生无效状态的问题。

    参考：[#6181](https://www.sqlalchemy.org/trac/ticket/6181)

+   **[sql] [bug]**

    在`Dialect`类中添加了一个名为`Dialect.supports_statement_cache`的新标志。现在，为了使 SQLAlchemy 的查询缓存对该方言生效，该标志现在必须直接存在于方言类中。这一决定基于发现的问题，例如 [#6173](https://www.sqlalchemy.org/trac/ticket/6173) 揭示了那些从编译语句中硬编码文字值的方言，通常是用于 LIMIT / OFFSET 的数字参数，直到这些方言被修改为仅使用语句中存在的参数，否则将无法与缓存兼容。对于未应用此标志的第三方方言，SQL 日志将显示消息“方言不支持缓存”，表明方言应在验证没有在编译阶段中呈现每个语句的文字值后，应用此标志。

    另请参阅

    第三方方言的缓存

    参考：[#6184](https://www.sqlalchemy.org/trac/ticket/6184)

### schema

+   **[schema] [bug]**

    在`Enum`类型中引入一个新参数`Enum.omit_aliases`，允许在使用 pep435 Enum 时过滤别名。在之前的版本中，SQLAlchemy 在所有情况下都保留别名，导致创建具有额外状态的数据库枚举类型，这意味着它们在数据库中被视为不同的值。为了向后兼容，该标志在 1.4 系列中默认为`False`，但将在将来的版本中切换为`True`。如果未指定此标志并且传递的枚举包含别名，则会引发弃用警告。

    参考：[#6146](https://www.sqlalchemy.org/trac/ticket/6146)

### mypy

+   **[mypy] [bug]**

    修复了 mypy 插件中的问题，其中对 `as_declarative()` 新添加的支持需要更完全地将`DeclarativeMeta`类添加到 mypy 解释器的状态中，以免导致名称未找到错误；此外，还改进了插件设置全局名称的方式，包括`Mapped`名称。

    参考：[#sqlalchemy/sqlalchemy2-stubs/#14](https://www.sqlalchemy.org/trac/ticket/sqlalchemy/sqlalchemy2-stubs/#14)

### asyncio

+   **[asyncio] [bug]**

    修复了在运行安装了`contextvars`回溯库的 Python 3.6 时无法加载 asyncio 扩展的问题。

    参考：[#6166](https://www.sqlalchemy.org/trac/ticket/6166)

### postgresql

+   **[postgresql] [bug] [regression]**

    修复了由[#6023](https://www.sqlalchemy.org/trac/ticket/6023)引起的回归，当使用 psycopg2 时，对`ARRAY`中的元素应用 PostgreSQL 转换运算符时，如果数据类型也嵌入在`Variant`适配器的实例中，将无法使用正确的类型。

    另外，修复了在使用`Variant(ARRAY(some_schema_type))`时正确发出 CREATE TYPE 的支持。

    此更改也被**回溯**到：1.3.25

    参考：[#6182](https://www.sqlalchemy.org/trac/ticket/6182)

+   **[postgresql] [bug]**

    修复了 1.4.4 中发布的[#6099](https://www.sqlalchemy.org/trac/ticket/6099)修复中的拼写错误，完全阻止了此更改的正确工作，即错误消息与 pg8000 实际发出的内容不匹配。

    参考：[#6099](https://www.sqlalchemy.org/trac/ticket/6099)

+   **[postgresql] [bug]**

    修复了当针对`Engine`生成的 PostgreSQL `PGInspector`在“future”风格引擎上使用时，对`.get_enums()`、`.get_view_names()`、`.get_foreign_table_names()`和`.get_table_oid()`的失败问题，而不是直接针对连接使用时。

    参考：[#6170](https://www.sqlalchemy.org/trac/ticket/6170)

### mysql

+   **[mysql] [bug] [regression]**

    修复了 MySQL 方言中的回归，用于检测表是否存在的反射查询在非常旧的 MySQL 5.0 和 5.1 版本上失败的问题。

    参考：[#6163](https://www.sqlalchemy.org/trac/ticket/6163)

### mssql

+   **[mssql] [bug]**

    修复了 MSSQL 2012+中的回归，当在子查询中使用`offset=0`时，无法呈现 order by 子句。

    参考：[#6163](https://www.sqlalchemy.org/trac/ticket/6163)

### oracle

+   **[oracle] [bug] [regression]**

    修复了 Oracle 编译器在选择中不维护正确的 LIMIT/OFFSET 参数值的关键回归，这是由于缓存问题导致的。

    参考：[#6173](https://www.sqlalchemy.org/trac/ticket/6173)

## 1.4.4

发布日期：2021 年 3 月 30 日

### orm

+   **[orm] [bug]**

    修复了新的`PropComparator.and_()`功能中的关键问题，其中发出次要 SELECT 语句的加载器策略（如`selectinload()`和`lazyload()`）将无法适应用户定义的条件中的绑定参数，这些条件是针对当前执行的语句而不是缓存的语句，导致使用过时的绑定值。

    这还为尝试序列化使用`lazyload()`与`PropComparator.and_()`的对象添加了警告；加载器标准无法可靠地序列化和反序列化，应该为此情况使用急加载。

    参考：[#6139](https://www.sqlalchemy.org/trac/ticket/6139)

+   **[orm] [错误] [回归]**

    修复了`ScopedSession`接口中缺少的方法`Session.get()`。

    参考：[#6144](https://www.sqlalchemy.org/trac/ticket/6144)

### 引擎

+   **[引擎] [用例]**

    修改了`Transaction`使用的上下文管理器，以便在上下文管理器本身结束时不会发出“已分离”警告，如果事务已在块内手动回滚。这适用于常规事务、保存点事务和传统的“标记”事务。如果显式调用`.rollback()`方法超过一次，则仍会发出警告。

    参考：[#6155](https://www.sqlalchemy.org/trac/ticket/6155)

+   **[引擎] [错误]**

    修复了 CursorResult 中异常处理方法的错误参数。

    参考：[#6138](https://www.sqlalchemy.org/trac/ticket/6138)

### postgresql

+   **[postgresql] [错误] [反射]**

    修复了 PostgreSQL 反射中的问题，其中表示“NOT NULL”的列将取代相应域的可空性。

    此更改也**回溯**到：1.3.24

    参考：[#6161](https://www.sqlalchemy.org/trac/ticket/6161)

+   **[postgresql] [错误]**

    修改了 pg8000 方言的`is_disconnect()`处理程序，现在适应了 pg8000 1.19.0 发出的新的`InterfaceError`。感谢 Hamdi Burak Usul 的拉取请求。

    参考：[#6099](https://www.sqlalchemy.org/trac/ticket/6099)

### 杂项

+   **[杂项] [错误]**

    调整了`importlib_metadata`库的使用，以加载 setuptools 入口点，以适应一些弃用更改。

### orm

+   **[orm] [错误]**

    修复了新的`PropComparator.and_()`功能中的关键问题，其中发出次要 SELECT 语句的加载策略（如`selectinload()`和`lazyload()`）将无法适应用户定义的标准中的绑定参数，而不是当前执行的语句，而是缓存的语句，导致使用过时的绑定值。

    还为使用`lazyload()`与`PropComparator.and_()`的对象添加了警告；加载器条件无法可靠地序列化和反序列化，应该为此情况使用急加载。

    参考：[#6139](https://www.sqlalchemy.org/trac/ticket/6139)

+   **[orm] [错误] [回归]**

    修复了`ScopedSession`接口中缺少的方法`Session.get()`。

    参考：[#6144](https://www.sqlalchemy.org/trac/ticket/6144)

### 引擎

+   **[引擎] [用例]**

    修改了`Transaction`使用的上下文管理器，以便在上下文管理器本身结束时不会发出“已分离”警告，如果事务已在块内手动回滚。这适用于常规事务、保存点事务和传统的“标记”事务。如果显式调用`.rollback()`方法超过一次，仍会发出警告。

    参考：[#6155](https://www.sqlalchemy.org/trac/ticket/6155)

+   **[引擎] [错误]**

    修复了在`CursorResult`中异常处理方法的错误参数。

    参考：[#6138](https://www.sqlalchemy.org/trac/ticket/6138)

### postgresql

+   **[postgresql] [错误] [反射]**

    修复了在 PostgreSQL 反射中的问题，其中表达“NOT NULL”列将取代相应域的可空性。

    此更改也被**回溯**到：1.3.24

    参考：[#6161](https://www.sqlalchemy.org/trac/ticket/6161)

+   **[postgresql] [错误]**

    修改了`pg8000`方言的`is_disconnect()`处理程序，现在适应了`pg8000 1.19.0`发出的新的`InterfaceError`。感谢 Hamdi Burak Usul 的拉取请求。

    参考：[#6099](https://www.sqlalchemy.org/trac/ticket/6099)

### 杂项

+   **[杂项] [错误]**

    调整了`importlib_metadata`库的使用，以加载 setuptools 入口点，以适应一些弃用更改。

## 1.4.3

发布日期：2021 年 3 月 25 日

### orm

+   **[orm] [错误]**

    修复了一个 bug，即 Python 2.7.5（CentOS 7 上的默认版本）无法导入 SQLAlchemy，因为在这个 Python 版本上，`exec "statement"`和`exec("statement")`的行为不同。使用了兼容性`exec_()`函数。

    参考：[#6069](https://www.sqlalchemy.org/trac/ticket/6069)

+   **[orm] [错误]**

    修复了在使用相关子查询与`column_property()`一起的 ORM 查询时，当在属性中使用`Select.correlate_except()`来控制关联时，相关子查询中包含与意图不关联的相关子查询中的相同可选择项时，将无法正确关联到封闭子查询或 CTE 的错误。

    参考：[#6060](https://www.sqlalchemy.org/trac/ticket/6060)

+   **[orm] [错误]**

    修复了新“带条件关系”的功能组合可能与使用新“lambda SQL”功能的功能一起失败的错误，包括选择加载策略，如 selectinload 和 lazyload，在更复杂的场景中，如多态加载。

    参考：[#6131](https://www.sqlalchemy.org/trac/ticket/6131)

+   **[orm] [错误]**

    修复了支持，使得`ClauseElement.params()`方法可以正确地与包含跨越 ORM 关系结构的连接的`Select`对象一起工作，这是 1.4 版本中的新功能。

    参考：[#6124](https://www.sqlalchemy.org/trac/ticket/6124)

+   **[orm] [错误]**

    修复了关系加载器机制内部生成“在 2.0 中删除”的警告的问题。

    参考：[#6115](https://www.sqlalchemy.org/trac/ticket/6115)

### orm 声明式

+   **[orm] [声明式] [错误] [回归]**

    修复了回归，其中在每个类级别的`.metadata`属性将不被尊重，破坏了抽象声明类和混合类的每类层次结构`MetaData`的用例。

    另请参阅

    元数据

    参考：[#6128](https://www.sqlalchemy.org/trac/ticket/6128)

### 引擎

+   **[engine] [错误] [回归]**

    将`ResultProxy`名称恢复到`sqlalchemy.engine`命名空间。此名称指的是`LegacyCursorResult`对象。

    参考：[#6119](https://www.sqlalchemy.org/trac/ticket/6119)

### 模式

+   **[模式] [错误]**

    调整了在删除多个表时发出 DROP 语句的逻辑，以便在所有表之后删除所有`Sequence`对象，即使给定的`Sequence`仅与`Table`对象相关，而不是直接与整体`MetaData`对象相关。该用例支持同一`Sequence`同时关联多个`Table`。

    此更改也已**回溯**至：1.3.24

    参考：[#6071](https://www.sqlalchemy.org/trac/ticket/6071)

### mypy

+   **[mypy] [bug]**

    添加了对 Mypy 扩展的支持，以正确解释使用`as_declarative()`函数生成的声明基类以及`registry.as_declarative_base()`方法生成的声明基类的情况。

+   **[mypy] [bug]**

    修复了 Mypy 插件中的错误，其中对于`Boolean`列类型的 Python 类型检测会产生异常；另外，实现了对`Enum`的支持，包括检测基于字符串的枚举与使用 Python `enum.Enum`的区别。

    参考：[#6109](https://www.sqlalchemy.org/trac/ticket/6109)

### postgresql

+   **[postgresql] [bug] [types]**

    调整了 psycopg2 方言以发出一个明确的 PostgreSQL 风格的转换，用于包含 ARRAY 元素的绑定参数。这允许在数组中正确使用完整的数据类型范围。 asyncpg 方言已在最终语句中生成了这些内部转换。 这还包括对数组切片更新以及 PostgreSQL 特定的`ARRAY.contains()`方法的支持。

    此更改也已**回溯**至：1.3.24

    参考：[#6023](https://www.sqlalchemy.org/trac/ticket/6023)

+   **[postgresql] [bug] [reflection]**

    修复了在 PostgreSQL 中具有混合大小写名称的表中反射标识列的问题。

    参考：[#6129](https://www.sqlalchemy.org/trac/ticket/6129)

### sqlite

+   **[sqlite] [feature] [asyncio]**

    为 aiosqlite 数据库驱动程序添加了与 SQLAlchemy asyncio 扩展一起使用的支持。

    另请参见

    Aiosqlite

    参考：[#5920](https://www.sqlalchemy.org/trac/ticket/5920)

+   **[sqlite] [bug] [regression]**

    修复了在 1.4 中出现回归的 `pysqlcipher` 方言以正确连接的问题，并添加了测试 + CI 支持以保持驱动程序处于工作状态。该方言现在默认导入 Python 3 的 `sqlcipher3` 模块，然后回退到已记录为不再维护的 `pysqlcipher3`。

    另请参见

    Pysqlcipher

    参考：[#5848](https://www.sqlalchemy.org/trac/ticket/5848)

### orm

+   **【orm】【错误】**

    修复了一个 bug，在此 bug 中，python 2.7.5（CentOS 7 上的默认版本）无法导入 sqlalchemy，因为在这个 Python 版本上 `exec "statement"` 和 `exec("statement")` 不会表现出相同的行为。兼容性 `exec_()` 函数被使用。

    参考：[#6069](https://www.sqlalchemy.org/trac/ticket/6069)

+   **【orm】【错误】**

    修复了一个 bug，在此 bug 中，使用关联子查询与`column_property()`一起的 ORM 查询在控制相关性的属性中使用 `Select.correlate_except()` 时，会导致与封闭子查询或 CTE 的相关性无法正确关联，在子查询中包含了与意图不相关的相关子查询相同的 selectables 的情况下。

    参考：[#6060](https://www.sqlalchemy.org/trac/ticket/6060)

+   **【orm】【错误】**

    修复了新的“带条件关系”的组合可能与使用新的“lambda SQL”特性的特性一起失败的 bug，包括 loader 策略，如 selectinload 和 lazyload，用于更复杂的场景，如多态加载。

    参考：[#6131](https://www.sqlalchemy.org/trac/ticket/6131)

+   **【orm】【错误】**

    修复了支持，使得`ClauseElement.params()`方法可以正确地与包含 ORM 关系结构跨越联接的`Select`对象一起工作，这是 1.4 中的一个新功能。

    参考：[#6124](https://www.sqlalchemy.org/trac/ticket/6124)

+   **【orm】【错误】**

    修复了一个问题，其中关系加载程序机制在内部生成了一个“在 2.0 中删除”的警告。

    参考：[#6115](https://www.sqlalchemy.org/trac/ticket/6115)

### orm 声明

+   **【orm】【声明】【错误】【回归】**

    修复了在每个类级别上的`.metadata`属性不会被遵循的回归，从而破坏了针对抽象声明类和混合类的每类层次结构的使用案例`MetaData`。

    另请参见

    metadata

    参考：[#6128](https://www.sqlalchemy.org/trac/ticket/6128)

### 引擎

+   **[引擎] [错误] [回归]**

    将`ResultProxy`名称恢复到`sqlalchemy.engine`命名空间。此名称指代`LegacyCursorResult`对象。

    参考：[#6119](https://www.sqlalchemy.org/trac/ticket/6119)

### 模式

+   **[模式] [错误]**

    调整了在删除多个表时发出`Sequence`对象的 DROP 语句的逻辑，以便在删除所有表之后删除所有`Sequence`对象，即使给定的`Sequence`仅与一个`Table`对象相关联，而不直接与整体`MetaData`对象相关联。该用例支持同一`Sequence`同时与多个`Table`相关联。

    此更改也**回溯**到：1.3.24

    参考：[#6071](https://www.sqlalchemy.org/trac/ticket/6071)

### mypy

+   **[mypy] [错误]**

    增加了对 Mypy 扩展的支持，以正确解释使用`as_declarative()`函数生成的声明基类以及`registry.as_declarative_base()`方法。

+   **[mypy] [错误]**

    修复了 Mypy 插件中的错误，其中对于`Boolean`列类型的 Python 类型检测会产生异常；此外还实现了对`Enum`的支持，包括检测基于字符串的枚举与使用 Python `enum.Enum`的区别。

    参考：[#6109](https://www.sqlalchemy.org/trac/ticket/6109)

### postgresql

+   **[postgresql] [错误] [类型]**

    调整了 psycopg2 方言以发出包含 ARRAY 元素的绑定参数的显式 PostgreSQL 风格转换。这允许各种数据类型在数组中正确运行。asyncpg 方言已在最终语句中生成了这些内部转换。这还包括对数组切片更新的支持以及 PostgreSQL 特定的`ARRAY.contains()`方法。

    此更改也**回溯**到：1.3.24

    参考：[#6023](https://www.sqlalchemy.org/trac/ticket/6023)

+   **[postgresql] [错误] [反射]**

    修复了在具有混合大小写名称的表中反射标识列在 PostgreSQL 中的问题。

    参考：[#6129](https://www.sqlalchemy.org/trac/ticket/6129)

### sqlite

+   **[sqlite] [特性] [asyncio]**

    添加了对 aiosqlite 数据库驱动程序的支持，用于与 SQLAlchemy 异步扩展一起使用。

    另见

    Aiosqlite

    参考：[#5920](https://www.sqlalchemy.org/trac/ticket/5920)

+   **[sqlite] [bug] [regression]**

    修复了在 1.4 中发生的 `pysqlcipher` 方言连接正确的问题，并添加了测试 + CI 支持以保持驱动程序处于工作状态。方言现在默认在 Python 3 中导入 `sqlcipher3` 模块，然后再退回到已记录为不再维护的 `pysqlcipher3`。

    另见

    Pysqlcipher

    参考：[#5848](https://www.sqlalchemy.org/trac/ticket/5848)

## 1.4.2

发布日期：2021 年 3 月 19 日

### orm

+   **[orm] [usecase] [dataclasses]**

    添加了对 `declared_attr` 对象在数据类字段上工作的支持。

    另见

    使用声明性混合与现有的数据类

    参考：[#6100](https://www.sqlalchemy.org/trac/ticket/6100)

+   **[orm] [bug] [dataclasses]**

    修复了新 ORM 数据类功能中的问题，其中包含列或其他映射构造的抽象基类或混入上的数据类字段不会被映射，如果它们还包含数据类字段对象中的“default”键，则不会映射。

    参考：[#6093](https://www.sqlalchemy.org/trac/ticket/6093)

+   **[orm] [bug] [regression]**

    修复了一个问题，即 `Query.selectable` 访问器，它是 `Query.__clause_element__()` 的同义词，被移除了，现在已恢复。

    参考：[#6088](https://www.sqlalchemy.org/trac/ticket/6088)

+   **[orm] [bug] [regression]**

    修复了一个回归问题，即如果查询本身使用了实体的 joinedload 并且也被 joinedload 的预加载过程包装在子查询中，则使用未命名的 SQL 表达式（例如 SQL 函数）会引发列定位错误。

    参考：[#6086](https://www.sqlalchemy.org/trac/ticket/6086)

+   **[orm] [bug] [regression]**

    修复了当 `Query.filter_by()` 方法无法定位正确的源实体时的回归问题，如果 `Query.join()` 方法已使用并且没有任何 ON 子句的实体作为目标，则会失败。

    参考：[#6092](https://www.sqlalchemy.org/trac/ticket/6092)

+   **[orm] [bug] [regression]**

    修复了`Function`的 SQL 编译在对象被“注释”时无法正确工作的回归问题，这是 ORM 主要使用的内部记忆化过程。特别是在 1.4 中，它可能会影响 ORM 的延迟加载，因为在 1.4 中更多地使用了这个特性。

    参考：[#6095](https://www.sqlalchemy.org/trac/ticket/6095)

+   **[orm] [bug]**

    修复了`ConcreteBase`的回归问题，当映射列名与鉴别器列名重叠时，会导致断言错误，完全无法映射。这里的用例在 1.3 版本中无法正确运行，因为多态联合会生成一个查询，完全忽略鉴别器列，同时发出重复列警告。由于 1.4 的架构目前在`select()`级别无法轻松复制 1.3 的这种基本上错误的行为，现在该用例会引发一个信息性错误消息，指示用户使用`.ConcreteBase._concrete_discriminator_name`属性来解决冲突。为了帮助进行配置，`.ConcreteBase._concrete_discriminator_name`只能放在基类上，子类将自动使用；以前不是这样。

    参考：[#6090](https://www.sqlalchemy.org/trac/ticket/6090)

### engine

+   **[engine] [bug] [regression]**

    恢复了`sqlalchemy.engine.reflection`的顶级导入。这确保了基本的`Inspector`类被正确注册，以便第三方方言可以正常使用`inspect()`。

### sql

+   **[sql] [bug] [regression]**

    修复了使用包含点分隔包名的`func`会由于 Python 名称列表需要是元组而无法被 SQL 缓存系统缓存的问题。

    参考：[#6101](https://www.sqlalchemy.org/trac/ticket/6101)

+   **[sql] [bug] [regression]**

    修复了`case()`构造中的回归问题，在“字典”形式的参数规范如果按位置传递而不是作为“whens”关键字参数传递时，无法正确工作。

    参考：[#6097](https://www.sqlalchemy.org/trac/ticket/6097)

### mypy

+   **[mypy] [bug]**

    修复了 MyPy 扩展中的问题，当以模块前缀形式给出类型时，例如`sa.Integer()`，会导致崩溃。

    参考：[#sqlalchemy/sqlalchemy2-stubs/2](https://www.sqlalchemy.org/trac/ticket/sqlalchemy/sqlalchemy2-stubs/2)

### postgresql

+   **[postgresql] [usecase]**

    重命名在某些兼容 postgresql 数据库中使用保留字的反射查询使用的列名。

    参考：[#6982](https://www.sqlalchemy.org/trac/ticket/6982)

### orm

+   **[orm] [用例] [数据类]**

    支持`declared_attr`对象在数据类字段上的工作。

    另请参阅

    在现有数据类中使用声明性混合类

    参考：[#6100](https://www.sqlalchemy.org/trac/ticket/6100)

+   **[orm] [错误] [数据类]**

    修复了新 ORM 数据类功能中的问题，即在包含列或其他映射构造的抽象基类或混合类上的数据类字段如果还包含数据类.field()对象中的“default”键，则不会被映射。

    参考：[#6093](https://www.sqlalchemy.org/trac/ticket/6093)

+   **[orm] [错误] [回归]**

    修复了`Query.selectable`访问器的回归问题，它是`Query.__clause_element__()`的同义词，现已恢复。

    参考：[#6088](https://www.sqlalchemy.org/trac/ticket/6088)

+   **[orm] [错误] [回归]**

    修复了使用未命名 SQL 表达式（如 SQL 函数）会引发列定位错误的回归问题，如果查询本身正在使用 joinedload 加载实体，并且也被 joinedload 贪婪加载过程包装在子查询中。

    参考：[#6086](https://www.sqlalchemy.org/trac/ticket/6086)

+   **[orm] [错误] [回归]**

    修复了当`Query.join()`方法针对没有任何 ON 子句的实体时，`Query.filter_by()`方法无法定位正确源实体的回归问题。

    参考：[#6092](https://www.sqlalchemy.org/trac/ticket/6092)

+   **[orm] [错误] [回归]**

    修复了当`Function`的 SQL 编译在对象已被“注释”时无法正确工作的回归问题，这是 ORM 主要使用的内部记忆化过程。特别是它可能影响在 1.4 中更多地使用此功能的 ORM 延迟加载。

    参考：[#6095](https://www.sqlalchemy.org/trac/ticket/6095)

+   **[orm] [错误]**

    修复了 `ConcreteBase` 在映射列名与鉴别器列名重叠时完全无法映射的回归问题，导致断言错误。在这种情况下，1.3 中的用例无法正确运行，因为多态联合会生成一个查询，完全忽略鉴别器列，同时发出重复列警告。由于 1.4 的架构目前无法在 `select()` 级别轻松复制 1.3 的这种基本上错误行为，因此现在该用例会引发一个信息性错误消息，指示用户使用 `.ConcreteBase._concrete_discriminator_name` 属性解决冲突。为了帮助配置，`.ConcreteBase._concrete_discriminator_name` 只能放在基类上，子类会自动使用；之前不是这样的。

    参考：[#6090](https://www.sqlalchemy.org/trac/ticket/6090)

### engine

+   **[engine] [bug] [regression]**

    恢复了 `sqlalchemy.engine.reflection` 的顶层导入。这确保了基础 `Inspector` 类被正确注册，以便于第三方方言可以正常使用 `inspect()`，而不需要额外导入此包。

### sql

+   **[sql] [bug] [regression]**

    修复了使用包含点分隔的包名的 `func` 无法被 SQL 缓存系统缓存的问题，因为需要一个 Python 名称列表，而不是一个元组。

    参考：[#6101](https://www.sqlalchemy.org/trac/ticket/6101)

+   **[sql] [bug] [regression]**

    修复了 `case()` 构造中的回归问题，当以“字典”形式的参数规范传递时，如果按位置传递而不是作为“whens”关键字参数传递，则无法正确工作。

    参考：[#6097](https://www.sqlalchemy.org/trac/ticket/6097)

### mypy

+   **[mypy] [bug]**

    修复了 MyPy 扩展中的问题，当以模块前缀形式给出 `sa.Integer()` 这样的类型时，会导致崩溃，无法检测到 `Column` 的类型。

    参考：[#sqlalchemy/sqlalchemy2-stubs/2](https://www.sqlalchemy.org/trac/ticket/sqlalchemy/sqlalchemy2-stubs/2)

### postgresql

+   **[postgresql] [usecase]**

    重命名了反射查询中使用的列名，该列名在某些兼容 postgresql 数据库中使用了保留字。

    参考：[#6982](https://www.sqlalchemy.org/trac/ticket/6982)

## 1.4.1

发布日期：2021 年 3 月 17 日

### orm

+   **[orm] [bug] [regression]**

    修复了一个回归问题，即使用 ORM 实体生成核心表达式构造（例如`select()`）时，会急切地配置映射器，以保持与`Query`对象的兼容性，后者为了支持许多与反向引用相关的遗留情况而必要。然而，核心`select()`构造也用于映射器配置等情况，因此在没有 ORM 加载类型的函数（如`Load`）的情况下，这种急切的配置更像是一种不便，因此在缺少 ORM 加载类型的情况下，已禁用了`select()`和其他核心构造的急切配置。

    此更改保持了`Query`的行为，以保持向后兼容性。然而，当与 ORM 实体一起使用`select()`时，如果“反向引用”直到映射器配置时间才明确放置在一个类上，则除非在其他地方调用了`configure_mappers()`或较新的`configure()`，否则该“反向引用”将不可用。更倾向于使用`relationship.back_populates`进行更明确的关系配置，该配置不需要急切配置要求。

    参考：[#6066](https://www.sqlalchemy.org/trac/ticket/6066)

+   **[orm] [bug] [regression]**

    修复了关系惰性加载器中的一个关键回归问题，即用于获取相关的多对一对象的 SQL 条件可能会与加载器内的其他记忆化结构不一致，如果映射器有配置更改，例如在映射器被晚配置或按需配置时可能会出现的情况，会生成与 None 的比较并返回没有对象。非常感谢 Alan Hamlett 在深夜追踪此问题时提供的帮助。

    参考：[#6055](https://www.sqlalchemy.org/trac/ticket/6055)

+   **[orm] [bug] [regression]**

    修复了一个回归问题，即`Query.exists()`方法在实体列表是任意 SQL 列表达式时会失败创建表达式的问题。

    参考：[#6076](https://www.sqlalchemy.org/trac/ticket/6076)

+   **[orm] [bug] [regression]**

    修复了一个回归问题，即在调用`Query.count()`与`joinedload()`等加载选项一起时，会忽略加载选项。这是一个一直非常特定于`Query.count()`方法的行为；如果给定的`Query`具有不适用于其返回内容的选项，则通常会引发错误。

    参考：[#6052](https://www.sqlalchemy.org/trac/ticket/6052)

+   **[orm] [bug] [regression]**

    修复了`Session.identity_key()`中的回归问题，包括该方法及相关方法未被任何单元测试覆盖，以及该方法包含一个拼写错误，导致其无法正常运行。

    参考：[#6067](https://www.sqlalchemy.org/trac/ticket/6067)

### orm declarative

+   **[orm] [declarative] [bug] [regression]**

    修复了一个 bug，即包含名为“registry”的属性的用户映射类在使用`DeclarativeMeta`时会与新的基于注册表的映射系统发生冲突。虽然该属性仍然可以在声明基类上显式设置以供元类使用，但一旦找到，它就会被放置在私有类变量下，以避免与将来使用相同名称用于其他目的的子类发生冲突。

    参考：[#6054](https://www.sqlalchemy.org/trac/ticket/6054)

### engine

+   **[engine] [bug] [regression]**

    Python 的`namedtuple()`具有这样的行为，即如果命名元组包含名称`count`和`index`，则它们将作为元组值提供；如果它们不存在，则将保持其作为`collections.abc.Sequence`方法的行为。因此，已修复`Row`和`LegacyRow`类，使其以相同方式工作，保持对具有列名称`index`或`count`的数据库行的预期行为。

    参考：[#6074](https://www.sqlalchemy.org/trac/ticket/6074)

### mssql

+   **[mssql] [bug] [regression]**

    修复了一个回归，即启用了一个新的可用于 pyodbc 的 setinputsizes() API，显然与 pyodbc 的 fast_executemany()模式不兼容，除非提供更准确的类型信息，但目前尚未完全实现或测试。已修改 pyodbc 方言和连接器，以便除非通过参数`use_setinputsizes`传递给方言（例如通过`create_engine()`），否则根本不使用 setinputsizes()，在这种情况下，其行为可以使用`DialectEvents.do_setinputsizes()`钩子进行自定义。

    另请参阅

    Setinputsizes 支持

    参考：[#6058](https://www.sqlalchemy.org/trac/ticket/6058)

### 杂项

+   **[bug] [regression]**

    将`items`和`values`重新添加到`ColumnCollection`类中。在添加对票号#4753 中 from 子句和 selectable 中重复列的支持时引入了回归。

    参考：[#6068](https://www.sqlalchemy.org/trac/ticket/6068)

### orm

+   **[orm] [bug] [regression]**

    修复了一个回归，即生成 Core 表达式构造（例如`select()`）时，使用 ORM 实体会急切地配置映射器，以保持与`Query`对象的兼容性，后者必须这样做以支持许多与 backref 相关的传统情况。但是，Core `select()`构造也用于映射器配置等，从这个角度来看，这种急切的配置更多地是一个不便，因此在没有 ORM 加载类型函数（例如`Load`）的情况下，已禁用了对`select()`和其他 Core 构造的急切配置。

    此更改维护了`Query`的行为，以保持向后兼容性。但是，当与 ORM 实体一起使用 `select()` 时，如果直到映射器配置时间才将 “backref” 明确放置在其中一个类上，则该 “backref” 将不可用，除非在其他地方调用了 `configure_mappers()` 或更新的 `configure()`。更倾向于使用 `relationship.back_populates` 进行更明确的关系配置，该配置不需要急切的配置要求。

    引用：[#6066](https://www.sqlalchemy.org/trac/ticket/6066)

+   **[orm] [bug] [regression]**

    修复了一个关键的回归，在关系惰性加载器中，用于获取相关多对一对象的 SQL 条件可能会与加载器内的其他记忆化结构过期，如果映射器有配置更改，例如在映射器后配置或按需配置时可能发生，产生与 None 的比较并返回没有对象的情况。非常感谢 Alan Hamlett 深夜跟踪此问题。

    引用：[#6055](https://www.sqlalchemy.org/trac/ticket/6055)

+   **[orm] [bug] [regression]**

    修复了一个回归，其中`Query.exists()`方法在`Query`的实体列表是任意的 SQL 列表表达式时会失败。

    引用：[#6076](https://www.sqlalchemy.org/trac/ticket/6076)

+   **[orm] [bug] [regression]**

    修复了一个回归，其中在与 `joinedload()` 这样的加载器选项一起调用`Query.count()`时，会忽略加载器选项。这是一个一直非常特定于 `Query.count()` 方法的行为；通常会引发错误，如果给定的 `Query` 有不适用于其返回内容的选项。

    引用：[#6052](https://www.sqlalchemy.org/trac/ticket/6052)

+   **[orm] [bug] [regression]**

    修复了`Session.identity_key()`中的回归，包括该方法及相关方法未被任何单元测试覆盖，以及该方法包含一个拼写错误，导致其无法正常运行。

    参考：[#6067](https://www.sqlalchemy.org/trac/ticket/6067)

### orm 声明

+   **[orm] [declarative] [bug] [regression]**

    修复了包含名为“registry”的属性的用户映射类在使用`DeclarativeMeta`时会与新的基于注册表的映射系统发生冲突的错误。虽然该属性仍然可以在声明基类上显式设置以供元类使用，但一旦找到，它就会被放置在一个私有类变量下，以避免与将来使用相同名称用于其他目的的子类发生冲突。

    参考：[#6054](https://www.sqlalchemy.org/trac/ticket/6054)

### 引擎

+   **[engine] [bug] [regression]**

    Python 的`namedtuple()`具有这样的行为，即如果命名元组包含这些名称，则名称`count`和`index`将作为元组值提供；如果它们不存在，则将保持它们作为`collections.abc.Sequence`的方法的行为。因此，已修复`Row`和`LegacyRow`类，使它们以相同的方式工作，保持对具有列名`index`或`count`的数据库行的预期行为。

    参考：[#6074](https://www.sqlalchemy.org/trac/ticket/6074)

### mssql

+   **[mssql] [bug] [regression]**

    修复了一个回归问题，即启用了一个新的`setinputsizes()` API，该 API 对于 pyodbc 是可用的，但在没有更准确的类型信息的情况下，它显然与 pyodbc 的`fast_executemany()`模式不兼容，这种类型信息目前尚未完全实现或测试。已修改 pyodbc 方言和连接器，以便除非通过参数`use_setinputsizes`传递给方言（例如通过`create_engine()`），否则根本不使用`setinputsizes()`，在这种情况下，可以使用`DialectEvents.do_setinputsizes()`钩子来自定义其行为。

    另请参阅

    Setinputsizes Support

    参考：[#6058](https://www.sqlalchemy.org/trac/ticket/6058)

### 杂项

+   **[bug] [regression]**

    将`ColumnCollection`类中的`items`和`values`重新添加。这个回归是在为票号#4753 中的 from 子句和 selectable 添加对重复列的支持时引入的。

    参考：[#6068](https://www.sqlalchemy.org/trac/ticket/6068)

## 1.4.0

发布日期：2021 年 3 月 15 日

### orm

+   **[orm] [bug]**

    移除了非常古老的警告，指出`passive_deletes`不适用于多对一关系。虽然在许多情况下，在多对一关系上放置此参数可能不是预期的操作，但在某些情况下，可能希望禁止从此类关系进行级联删除。

    此更改也**回溯**到：1.3.24

    参考：[#5983](https://www.sqlalchemy.org/trac/ticket/5983)

+   **[orm] [bug]**

    修复了连接两个表的过程可能失败的问题，如果其中一个表具有无关的、无法解决的外键约束，这将在连接过程中引发`NoReferenceError`，尽管可以绕过此问题以允许连接完成。在处理中测试异常重要性的逻辑会对构造做出失败的假设。

    此更改也**回溯**到：1.3.24

    参考：[#5952](https://www.sqlalchemy.org/trac/ticket/5952)

+   **[orm] [bug]**

    修复了当父对象已加载并且后续查询覆盖时，`MutableComposite`构造可能处于无效状态的问题，因为复合属性的刷新处理程序将对象替换为可变扩展未处理的新对象。

    此更改也**回溯**到：1.3.24

    参考：[#6001](https://www.sqlalchemy.org/trac/ticket/6001)

+   **[orm] [bug]**

    修复了`relationship.query_class`参数对“dynamic”关系停止功能的回归问题。`AppenderQuery`仍依赖于传统的`Query`类；鼓励用户从使用“dynamic”关系转而使用`with_parent()`。

    参考：[#5981](https://www.sqlalchemy.org/trac/ticket/5981)

+   **[orm] [bug] [regression]**

    修复了`Query.join()`在查询本身以及连接目标都针对`Table`对象而不是映射类时不会产生效果的回归问题。这是一个更为系统性的问题，即传统的 ORM 查询编译器在语句中没有 ORM 实体时不会被正确使用。

    参考：[#6003](https://www.sqlalchemy.org/trac/ticket/6003)

+   **[orm] [bug] [asyncio]**

    `AsyncSession.delete()`的 API 现在是可等待的；这个方法会沿着必须以类似方式加载的关系级联，就像`AsyncSession.merge()`方法一样。

    参考：[#5998](https://www.sqlalchemy.org/trac/ticket/5998)

+   **[ORM] [错误]**

    当刷新进行时，工作单元过程现在完全关闭所有“lazy='raise'”行为。虽然有些情况下，工作单元有时会加载一些最终不需要的内容，但在这里“lazy='raise'”策略并不起作用，因为用户通常无法对刷新过程进行控制或查看。

    参考：[#5984](https://www.sqlalchemy.org/trac/ticket/5984)

### 引擎

+   **[引擎] [错误]**

    修复了“schema_translate_map”功能未考虑到直接执行`DefaultGenerator`对象（如序列）的用例失败的错误，其中包括它们“预执行”以生成主键值的情况，当禁用 implicit_returning 时。

    这个更改也**回溯**到：1.3.24

    参考：[#5929](https://www.sqlalchemy.org/trac/ticket/5929)

+   **[引擎] [错误]**

    改进了引擎日志记录，以记录在 DBAPI 驱动程序处于 AUTOCOMMIT 模式时记录的 ROLLBACK 和 COMMIT。这些 ROLLBACK/COMMIT 是库级别的，当 AUTOCOMMIT 生效时没有任何影响，但仍然值得记录，因为这些指示了 SQLAlchemy 看到的“事务”界定。

    参考：[#6002](https://www.sqlalchemy.org/trac/ticket/6002)

+   **[引擎] [错误] [回归]**

    修复了连接池的“重置代理”在`Connection`关闭时实际上并未被利用的回归，也导致了一种有些浪费的双回滚场景。引擎的新架构已更新，以便在`Connection`在返回连接池之前显式关闭事务时跳过连接池的“返回时重置”逻辑。

    参考：[#6004](https://www.sqlalchemy.org/trac/ticket/6004)

### SQL

+   **[SQL] [更改]**

    更改了`CTE`构造的编译，以便在`CTE`直接字符串化时返回表示内部 SELECT 语句的字符串，而不是在封闭 SELECT 的上下文之外；这与`FromClause.alias()`和`Select.subquery()`的行为相同。以前，由于 CTE 通常放置在生成 SELECT 之后的 SELECT 之上，因此会返回空字符串，这在调试时通常会产生误导。

+   **[sql] [bug]**

    修复了使用“format”或“pyformat”绑定参数样式的方言发生的“百分比转义”功能未对使用百分号的自定义操作符`Operators.op()`和`custom_op`构造启用的错误，必要时百分号现在将根据 paramstyle 自动加倍。

    参考：[#6016](https://www.sqlalchemy.org/trac/ticket/6016)

+   **[sql] [bug] [regression]**

    修复了未知数据类型的“不支持编译错误”在引发时未能正确引发的回归问题。

    参考：[#5979](https://www.sqlalchemy.org/trac/ticket/5979)

+   **[sql] [bug] [regression]**

    修复了在直接 SELECT 中使用独立`distinct()`形式时，无法通过列标识在结果集中定位的回归问题，这是 ORM 定位列的方式。虽然独立的`distinct()`不是面向直接 SELECT 的（对于常规的`SELECT DISTINCT..`使用`select.distinct()`），但以前在这种方式上有一定程度的可用性（但在子查询中不起作用，例如）。对于像“DISTINCT <col>”这样的一元表达式的列定位已经改进，以便此情况再次起作用，并且已经进行了额外的改进，以便至少在子查询中使用此形式生成有效的 SQL，这在以前不是这种情况。

    此更改还增强了在 ORM 启用的 SELECT 语句中基于 SQL 表达式对象定位`row._mapping`中元素的能力，包括语句是由`connection.execute()`还是`session.execute()`调用的。

    参考：[#6008](https://www.sqlalchemy.org/trac/ticket/6008)

### 模式

+   **[schema] [bug] [sqlite]**

    修复了由`Boolean`或`Enum`生成的 CHECK 约束在第一次编译后无法正确呈现命名约定的问题，这是由于约束名称中的状态意外更改导致的。此问题首次在 0.9 中引入，用于修复问题＃3067，修复了当时采取的方法，该方法似乎比所需的更复杂。

    这个更改也被**反向移植**到：1.3.24

    参考：[#6007](https://www.sqlalchemy.org/trac/ticket/6007)

+   **[schema] [bug]**

    修复/实现了支持使用列名/键等作为约定的主键约束命名规范。特别是，这包括`Table`自动关联的`PrimaryKeyConstraint`对象将在新的主键`Column`对象添加到表格，然后添加到约束时更新其名称。现在已经适应了与此约束构造过程相关的内部故障模式，包括没有列存在，没有名称存在或空名称存在。

    这个更改也被**反向移植**到：1.3.24

    参考：[#5919](https://www.sqlalchemy.org/trac/ticket/5919)

+   **[schema] [bug]**

    弃用了所有基于模式级别的`.copy()`方法，并重命名为`_copy()`。这些不是标准的 Python“copy()”方法，因为它们通常依赖于在特定上下文中实例化，这些上下文作为可选关键字参数传递给方法。`Table.tometadata()`方法是为`Table`对象提供复制的公共 API。 

    参考：[#5953](https://www.sqlalchemy.org/trac/ticket/5953)

### mypy

+   **[mypy] [feature]**

    已添加了基本和实验性的 Mypy 支持，形式为一个新插件，该插件本身依赖于 SQLAlchemy 的新类型存根。该插件允许声明性映射以标准形式与 Mypy 兼容，并为映射类和实例提供类型支持。

    另请参阅

    Mypy / Pep-484 对 ORM 映射的支持

    参考：[#4609](https://www.sqlalchemy.org/trac/ticket/4609)

### postgresql

+   **[postgresql] [usecase] [asyncio] [mysql]**

    在 SQLAlchemy 模拟的 DBAPI 游标中为 asyncpg 和 aiomysql 方言添加了一个`asyncio.Lock()`，局限于`cursor.execute()`和`cursor.executemany()`方法的连接范围内。其理念是防止在连接同时用于多个可等待时出现失败和损坏的情况。

    虽然这种用例也可能出现在多线程代码和非 asyncio 方言中，但我们预计这种用法在 asyncio 下会更常见，因为 asyncio API 鼓励这种用法。然而，最好为每个并发的可等待使用一个独立的连接，否则将无法实现并发。

    对于 asyncpg 方言，这样做是为了防止在`prepare()`和`fetch()`之间的空间允许连接上的并发执行导致接口错误异常，以及在启动新事务时防止竞争条件。其他 PostgreSQL DBAPI 在连接级别上是线程安全的，因此这意图提供类似的行为，超出了服务器端游标的范围。

    对于 aiomysql 方言，互斥锁将提供安全性，使语句执行和结果集获取这两个不同步骤在连接级别上不会被同一连接上的并发执行破坏。

    参考：[#5967](https://www.sqlalchemy.org/trac/ticket/5967)

+   **[postgresql] [bug]**

    修复了在某些条件下使用`aggregate_order_by`会返回 ARRAY(NullType)的问题，干扰了结果对象正确返回数据的能力。

    此更改也被**回溯**到：1.3.24

    参考：[#5989](https://www.sqlalchemy.org/trac/ticket/5989)

### mssql

+   **[mssql] [bug]**

    修复了由过滤索引反射引入的 MSSQL 2005 的反射错误。

    参考：[#5919](https://www.sqlalchemy.org/trac/ticket/5919)

### 杂项

+   **[usecase] [ext]**

    添加新参数`AutomapBase.prepare.reflection_options`，允许传递`MetaData.reflect()`选项，如`only`或特定于方言的反射选项，如`oracle_resolve_synonyms`。

    参考：[#5942](https://www.sqlalchemy.org/trac/ticket/5942)

+   **[bug] [ext]**

    `sqlalchemy.ext.mutable` 扩展现在使用与对象关联的 `InstanceState` 跟踪“父”集合，而不是对象本身。后一种方法要求对象是可散列的，以便它可以在 `WeakKeyDictionary` 中，这与 ORM 的行为契约相违背，即 ORM 映射对象不需要提供任何特定类型的 `__hash__()` 方法，并且支持不可散列的对象。

    参考：[#6020](https://www.sqlalchemy.org/trac/ticket/6020)

### orm

+   **[orm] [bug]**

    移除了非常古老的警告，指出 `passive_deletes` 不适用于多对一关系。虽然在许多情况下，将此参数放在多对一关系上可能不是预期的操作，但在某些情况下，可能希望禁止从此关系进行级联删除。

    此更改也**回溯**到：1.3.24

    参考：[#5983](https://www.sqlalchemy.org/trac/ticket/5983)

+   **[orm] [bug]**

    修复了连接两个表的过程可能失败的问题，如果其中一个表具有无关的、无法解决的外键约束，这将在连接过程中引发 `NoReferenceError`，尽管可以绕过此异常以允许连接完成。在处理中测试异常重要性的逻辑会对构造做出可能失败的假设。

    此更改也**回溯**到：1.3.24

    参考：[#5952](https://www.sqlalchemy.org/trac/ticket/5952)

+   **[orm] [bug]**

    修复了当父对象已加载并且后续查询覆盖时，`MutableComposite` 构造可能处于无效状态的问题，这是由于复合属性的刷新处理程序用新对象替换了对象，而这些对象未被可变扩展处理。

    此更改也**回溯**到：1.3.24

    参考：[#6001](https://www.sqlalchemy.org/trac/ticket/6001)

+   **[orm] [bug]**

    修复了对于“动态”关系，`relationship.query_class` 参数停止起作用的回归。`AppenderQuery` 仍依赖于传统的 `Query` 类；建议用户从使用“动态”关系迁移到使用 `with_parent()`。

    参考：[#5981](https://www.sqlalchemy.org/trac/ticket/5981)

+   **[orm] [bug] [regression]**

    修复了`Query.join()`在查询本身以及连接目标都是`Table`对象而不是映射类时产生无效的回归问题。这是一个更为系统性的问题的一部分，即如果生成的语句中没有 ORM 实体存在，传统的 ORM 查询编译器将不会被正确使用。

    参考：[#6003](https://www.sqlalchemy.org/trac/ticket/6003)

+   **[orm] [bug] [asyncio]**

    `AsyncSession.delete()`方法现在是可等待的；此方法沿关系级联，必须以与`AsyncSession.merge()`方法类似的方式加载关系。

    参考：[#5998](https://www.sqlalchemy.org/trac/ticket/5998)

+   **[orm] [bug]**

    当刷新进行时，工作单元过程现在完全关闭所有“lazy='raise'”行为。虽然有时候工作单元有时加载不需要的内容，但在这里 lazy=”raise”策略并不有用，因为用户通常对刷新过程没有太多控制或可见性。

    参考：[#5984](https://www.sqlalchemy.org/trac/ticket/5984)

### 引擎

+   **[engine] [bug]**

    修复了“schema_translate_map”功能未考虑直接执行`DefaultGenerator`对象（如序列）的情况失败的错误，其中包括在禁用 implicit_returning 时“预执行”它们以生成主键值的情况。

    此更改也被**回溯**到：1.3.24

    参考：[#5929](https://www.sqlalchemy.org/trac/ticket/5929)

+   **[engine] [bug]**

    改进了引擎日志记录，以记录在 DBAPI 驱动程序处于 AUTOCOMMIT 模式时记录的 ROLLBACK 和 COMMIT。这些 ROLLBACK/COMMIT 是库级别的，当 AUTOCOMMIT 生效时没有任何影响，但仍然值得记录，因为这些指示了 SQLAlchemy 看到的“事务”分界点。

    参考：[#6002](https://www.sqlalchemy.org/trac/ticket/6002)

+   **[engine] [bug] [regression]**

    修复了一个回归问题，即当`Connection`关闭时，“连接池复位代理”实际上并未真正被使用，并且还导致了一个有些浪费的双回滚场景。引擎的新架构已更新，以便在`Connection`在返回连接池之前显式关闭事务时，将跳过连接池“返回时复位”逻辑。

    参考：[#6004](https://www.sqlalchemy.org/trac/ticket/6004)

### sql

+   **[sql] [change]**

    更改了`CTE`构造的编译，以便在直接字符串化`CTE`时返回表示内部 SELECT 语句的字符串；这与`FromClause.alias()`和`Select.subquery()`的行为相同。以前，当调试时，通常会在生成 SELECT 之后将 CTE 放置在 SELECT 之上，这通常会导致误导性的空字符串返回。

+   **[sql] [bug]**

    修复了一个 bug，即在使用“format”或“pyformat”绑定参数样式的方言中，未为`Operators.op()`和`custom_op`构造启用“百分号转义”功能，用于使用百分号的自定义运算符。根据需要，百分号现在将自动根据参数样式加倍。

    参考：[#6016](https://www.sqlalchemy.org/trac/ticket/6016)

+   **[sql] [bug] [regression]**

    修复了一个回归问题，即对于未知数据类型的“不支持的编译错误”将无法正确引发。

    参考：[#5979](https://www.sqlalchemy.org/trac/ticket/5979)

+   **[sql] [bug] [regression]**

    修复了使用独立的 `distinct()` 直接作为 SELECT 的形式时，无法通过列标识在结果集中定位的回归问题，这是 ORM 定位列的方式。虽然独立的 `distinct()` 不是面向直接 SELECT 的（使用 `select.distinct()` 来进行常规的 `SELECT DISTINCT..`)，但以前在这种方式上有一定程度的可用性（但在子查询中无法工作，例如）。对于像“DISTINCT <col>”这样的一元表达式的列定位已经改进，以使此案例再次工作，并且还进行了额外的改进，以使此形式在子查询中的使用至少生成有效的 SQL，而以前不是这样的情况。

    此更改还增强了在 ORM 启用的 SELECT 语句中基于 SQL 表达式对象定位`row._mapping`中元素的能力，包括语句是由`connection.execute()`还是`session.execute()`调用的。

    引用：[#6008](https://www.sqlalchemy.org/trac/ticket/6008)

### 模式

+   **[模式] [错误] [sqlite]**

    修复了使用 `Boolean` 或 `Enum` 生成的 CHECK 约束在第一次编译后无法正确呈现命名约定的问题，这是由于约束名称中的状态不经意地改变所致。此问题最初是在修复问题＃3067 时于 0.9 中引入的，修复了当时采取的方法，该方法似乎比需要的更复杂。

    此更改也已**回溯**到：1.3.24

    引用：[#6007](https://www.sqlalchemy.org/trac/ticket/6007)

+   **[模式] [错误]**

    修复/实现了使用列名/键等作为约定一部分的主键约束命名约定的支持。特别是，这包括与 `Table` 自动关联的 `PrimaryKeyConstraint` 对象将在向表添加新的主键 `Column` 对象，然后向约束对象添加新的主键时更新其名称。现在，与此约束构造过程相关的内部故障模式，包括没有列存在，没有名称存在或存在空白名称，现在得到了解决。

    此更改也已**回溯**到：1.3.24

    引用：[#5919](https://www.sqlalchemy.org/trac/ticket/5919)

+   **[模式] [错误]**

    废弃了所有基于模式级别的`.copy()`方法，并重命名为`_copy()`。这些方法不是标准的 Python“copy()”方法，因为它们通常依赖于在特定上下文中实例化，这些上下文作为可选关键字参数传递给方法。`Table.tometadata()`方法是提供`Table`对象复制的公共 API。

    参考：[#5953](https://www.sqlalchemy.org/trac/ticket/5953)

### mypy

+   **[mypy] [feature]**

    已添加了基本和实验性的 Mypy 支持，以一种新的插件形式存在，该插件本身依赖于 SQLAlchemy 的新类型存根。该插件允许声明性映射以它们的标准形式与 Mypy 兼容，并为映射类和实例提供类型支持。

    另请参阅

    Mypy / Pep-484 对 ORM 映射的支持

    参考：[#4609](https://www.sqlalchemy.org/trac/ticket/4609)

### postgresql

+   **[postgresql] [usecase] [asyncio] [mysql]**

    在 SQLAlchemy 的模拟 DBAPI 游标中添加了一个`asyncio.Lock()`，局限于连接范围内，用于 asyncpg 和 aiomysql 方言的`cursor.execute()`和`cursor.executemany()`方法。其目的是防止在连接同时用于多个可等待对象时出现故障和损坏。

    虽然这种用例也可能出现在多线程代码和非 asyncio 方言中，但我们预计这种用法在 asyncio 下会更常见，因为 asyncio API 鼓励这种用法。然而，最好为每个并发可等待对象使用一个独立的连接，否则将无法实现并发。

    对于 asyncpg 方言，这样做是为了防止在调用`prepare()`和`fetch()`之间的空间允许连接上的并发执行导致接口错误异常，以及在启动新事务时防止竞争条件。其他 PostgreSQL DBAPI 在连接级别上是线程安全的，因此这意在提供类似的行为，超出了服务器端游标的范围。

    对于 aiomysql 方言，互斥锁将提供安全性，确保语句执行和结果集获取这两个在连接级别上的不同步骤不会被同一连接上的并发执行破坏。

    参考：[#5967](https://www.sqlalchemy.org/trac/ticket/5967)

+   **[postgresql] [bug]**

    修复了在某些条件下使用`aggregate_order_by`会返回 ARRAY(NullType)的问题，干扰了结果对象正确返回数据的能力。

    此更改也**回溯**至：1.3.24

    参考：[#5989](https://www.sqlalchemy.org/trac/ticket/5989)

### mssql

+   **[mssql] [bug]**

    修复了由于过滤索引的反射引入的 MSSQL 2005 的反射错误。

    参考：[#5919](https://www.sqlalchemy.org/trac/ticket/5919)

### 杂项

+   **[用例] [扩展]**

    添加了新参数`AutomapBase.prepare.reflection_options`，允许传递`MetaData.reflect()` 的选项，如 `only` 或特定于方言的反射选项，如 `oracle_resolve_synonyms`。

    参考：[#5942](https://www.sqlalchemy.org/trac/ticket/5942)

+   **[错误] [扩展]**

    `sqlalchemy.ext.mutable` 扩展现在使用与对象关联的 `InstanceState` 来跟踪 “父” 集合，而不是对象本身。后一种方法要求对象是可哈希的，以便它可以在 `WeakKeyDictionary` 中，这与 ORM 的整体行为合同相违背，即 ORM 映射对象不需要提供任何特定类型的 `__hash__()` 方法，并且支持不可哈希的对象。

    参考：[#6020](https://www.sqlalchemy.org/trac/ticket/6020)

## 1.4.0b3

发布日期：2021 年 2 月 15 日

### orm

+   **[orm] [功能]**

    在 2.0 风格中使用的 ORM 现在可以从 UPDATE..RETURNING 或 INSERT..RETURNING 语句返回的行中返回 ORM 对象，通过在 ORM 上下文中向`Select.from_statement()` 提供构造。

    另请参阅

    使用 INSERT、UPDATE 和 ON CONFLICT（即 upsert）返回 ORM 对象

+   **[orm] [错误]**

    修复了新的 1.4/2.0 风格 ORM 查询中存在的问题，其中语句级别的标签样式不会在结果行使用的键中被保留；此更改已应用于所有 Core/ORM 列 / 会话 vs. 连接等的组合，以便在所有情况下从语句到结果行的链接都相同。作为此更改的一部分，行中列表达式的标记已改进为保留 ORM 属性的原始名称，即使在子查询中使用。

    参考：[#5933](https://www.sqlalchemy.org/trac/ticket/5933)

### 引擎

+   **[引擎] [错误] [postgresql]**

    继续改进[#5653](https://www.sqlalchemy.org/trac/ticket/5653)所做的改进，进一步支持绑定参数名称，包括针对列名称生成的那些名称，用于包括冒号、括号和问号的名称，以及改进的测试支持，因此，即使绑定参数名称是根据列名称自动生成的，也不应该在 psycopg2 的“pyformat”风格中包含括号也没有问题。

    作为这一变化的一部分，asyncpg DBAPI 适配器（仅适用于 SQLAlchemy 的 asyncpg 方言）使用的格式已从“qmark”参数样式更改为“format”，因为对于使用“format”样式的名称（即双百分号）有一个标准且内部支持的 SQL 字符串转义样式，而不是使用“qmark”样式的名称（其中 pep-249 或 Python 未定义转义系统）。

    另请参阅

    psycopg2 方言不再限制绑定参数名称

    参考：[#5941](https://www.sqlalchemy.org/trac/ticket/5941)

### sql

+   **[sql] [用例] [postgresql] [sqlite]**

    增强`OnConflictDoUpdate`的`set_`关键字，以接受`ColumnCollection`，例如来自`Selectable`的`.c.`集合或`.excluded`上下文对象。

    参考：[#5939](https://www.sqlalchemy.org/trac/ticket/5939)

+   **[sql] [错误]**

    修复了“笛卡尔积”断言未正确适应依赖于使用 LATERAL 从子查询连接到封闭上下文中的另一个子查询的表之间连接的错误。

    参考：[#5924](https://www.sqlalchemy.org/trac/ticket/5924)

+   **[sql] [错误]**

    修复了 1.4 版本中`Function.in_()`方法未经测试并在所有情况下无法正常运行的回归。

    参考：[#5934](https://www.sqlalchemy.org/trac/ticket/5934)

+   **[sql] [错误]**

    修复了在`select()`函数中使用任意可迭代对象（而不仅限于普通列表）时不起作用的回归，此处的向前/向后兼容性逻辑现在检查更广泛的传入“可迭代”类型，包括可直接传递可选择的`.c`集合的情况。由 Oliver Rice 提供的拉取请求。

    参考：[#5935](https://www.sqlalchemy.org/trac/ticket/5935)

### orm

+   **[orm] [功能]**

    在 2.0 风格中使用的 ORM 现在可以通过在 ORM 上下文中向`Select.from_statement()`提供构造来从 UPDATE..RETURNING 或 INSERT..RETURNING 语句返回的行中返回 ORM 对象。

    另请参阅

    使用 INSERT、UPDATE 和 ON CONFLICT（即 upsert）返回 ORM 对象

+   **[orm] [错误]**

    修复了新的 1.4/2.0 风格 ORM 查询中的问题，其中语句级别的标签样式在结果行使用的键中不会被保留；这已应用于所有 Core/ORM 列/会话与连接等的组合，以便在所有情况下从语句到结果行的链接是相同的。作为这一变化的一部分，行中列表达式的标记已经得到改进，以保留 ORM 属性的原始名称，即使在子查询中使用也是如此。

    参考：[#5933](https://www.sqlalchemy.org/trac/ticket/5933)

### engine

+   **[engine] [bug] [postgresql]**

    继续改进作为[#5653](https://www.sqlalchemy.org/trac/ticket/5653)的一部分所做的改进，进一步支持绑定参数名称，包括那些根据列名生成的名称，对于包含冒号、括号和问号的名称，以及改进的测试支持，以便绑定参数名称即使是从列名自动生成的也不应该有问题，包括在 psycopg2 的“pyformat”样式中包含括号。

    作为这一变化的一部分，asyncpg DBAPI 适配器使用的格式（这是局限于 SQLAlchemy 的 asyncpg 方言的）已从使用“qmark” paramstyle 更改为“format”，因为对于使用“format”样式的名称，存在一种标准且内部支持的 SQL 字符串转义样式（即将百分号双倍），而不是使用“qmark”样式的名称（其中未定义由 pep-249 或 Python 定义的转义系统）。

    另请参阅

    psycopg2 方言不再对绑定参数名称有限制

    参考：[#5941](https://www.sqlalchemy.org/trac/ticket/5941)

### sql

+   **[sql] [usecase] [postgresql] [sqlite]**

    增强`OnConflictDoUpdate`的`set_`关键字，以接受`ColumnCollection`，例如从`Selectable`的`.c.`集合或`.excluded`上下文对象中的集合。

    参考：[#5939](https://www.sqlalchemy.org/trac/ticket/5939)

+   **[sql] [bug]**

    修复了“笛卡尔积”断言未正确适应依赖于使用 LATERAL 从子查询连接到封闭上下文中的另一个子查询的表之间连接的 bug。

    参考：[#5924](https://www.sqlalchemy.org/trac/ticket/5924)

+   **[sql] [bug]**

    修复了 1.4 版本中`Function.in_()`方法未经测试并在所有情况下无法正常运行的回归问题。

    参考：[#5934](https://www.sqlalchemy.org/trac/ticket/5934)

+   **[sql] [bug]**

    修复了使用 `select()` 函数的任意可迭代对象不起作用的回归问题，除了普通列表之外。这里的前向/后向兼容性逻辑现在检查更广泛的“可迭代”类型，包括可直接传递可选择项的 `.c` 集合。由 Oliver Rice 提供的拉取请求。

    参考：[#5935](https://www.sqlalchemy.org/trac/ticket/5935)

## 1.4.0b2

发布日期：2021 年 2 月 3 日

### general

+   **[general] [bug]**

    修复了一个 SQLite 源文件，在“INSERT..ON CONFLICT”功能中引入了非 ASCII 字符的文档字符串，没有源编码，这会导致在 Python 2 下失败。

### platform

+   **[platform] [performance]**

    调整了与导入时内部类生成相关的一些元素，这些元素导致与 1.3 版本相比导入库所花费的时间显著增加。现在所花费的时间比 1.3 版本慢了约 20-30%，而不是 200%。

    参考：[#5681](https://www.sqlalchemy.org/trac/ticket/5681)

### orm

+   **[orm] [usecase]**

    添加了`ORMExecuteState.bind_mapper`和`ORMExecuteState.all_mappers`访问器到`ORMExecuteState`事件对象，以便处理程序可以响应于 ORM 语句执行中涉及的目标映射器和/或映射类。

+   **[orm] [usecase] [asyncio]**

    添加了`AsyncSession.scalar()`，`AsyncSession.get()`以及对`sessionmaker.begin()`的支持，使其与`AsyncSession`一起作为异步上下文管理器工作。还添加了`AsyncSession.in_transaction()`访问器。

    参考：[#5796](https://www.sqlalchemy.org/trac/ticket/5796)，[#5797](https://www.sqlalchemy.org/trac/ticket/5797)，[#5802](https://www.sqlalchemy.org/trac/ticket/5802)

+   **[orm] [changed]**

    在`configure_mappers()`函数中发生的映射器“配置”现在按注册表基础组织。这允许例如配置某个声明性基类内的映射器，但不配置内存中还存在的另一个基类的映射器。目标是通过仅针对所需的映射器集运行“配置”过程来提供一种减少应用程序启动时间的方法。这还添加了`registry.configure()`方法，该方法仅对特定注册表中的本地映射器运行配置。

    参考：[#5897](https://www.sqlalchemy.org/trac/ticket/5897)

+   **[orm] [bug]**

    对于将映射类或字符串映射类名传递给`relationship.secondary`的情况，添加了全面的检查和信息丰富的错误消息。这是一个极其常见的错误，需要清晰的消息。

    此外，添加了对类注册表解析的新规则，即对于`relationship.secondary`参数，如果映射类及其表具有相同的字符串名称，将优先选择`Table`。在所有其他情况下，如果类和表具有相同的名称，则继续优先选择类。

    此更改也**反向移植**到：1.3.21

    参考：[#5774](https://www.sqlalchemy.org/trac/ticket/5774)

+   **[orm] [bug]**

    修复了与 ORM 事件（例如`InstanceEvents.load()`）的`restore_load_context`选项相关的错误，使得标志不会随着首次建立事件处理程序之后映射的子类而传递。

    此更改也**反向移植**到：1.3.21

    参考：[#5737](https://www.sqlalchemy.org/trac/ticket/5737)

+   **[orm] [bug] [regression]**

    修复了类似于`Connection`中的新“autobegin”逻辑的新`Session`中的问题，其中如果在`SessionEvents.after_transaction_create()`事件挂钩内执行了 SQL，则新逻辑可能陷入可重入（递归）状态。

    参考：[#5845](https://www.sqlalchemy.org/trac/ticket/5845)

+   **[orm] [bug] [unitofwork]**

    改进了工作单元拓扑排序系统，使拓扑排序现在是基于输入集的排序而确定性的，这些输入集本身现在在映射器的级别进行了排序，因此受影响的映射器的相同输入应该每次都产生相同的输出，对于彼此不具有任何依赖关系的映射器 / 表。这进一步降低了死锁的机会，如在刷新中可以观察到的对多个不相关表之间的 UPDATE，从而生成行锁。

    参考：[#5735](https://www.sqlalchemy.org/trac/ticket/5735)

+   **[orm] [bug]**

    修复了一个退化，即 `Bundle.single_entity` 标志将对 `Bundle` 生效，即使没有设置。此外，此标志是传统的，因为它仅对 `Query` 对象有意义，而不是 2.0 风格的执行。在新样式执行时使用时会发出弃用警告。

    参考：[#5702](https://www.sqlalchemy.org/trac/ticket/5702)

+   **[orm] [bug]**

    修复了一个退化，即针对普通可选择的创建 `aliased` 构造并包含名称会引发断言错误。

    参考：[#5750](https://www.sqlalchemy.org/trac/ticket/5750)

+   **[orm] [bug]**

    与核心中的 lambda 标准系统的修复相关，在 ORM 内部实现了各种修复，包括 `with_loader_criteria()` 功能以及经常与之结合使用的 `SessionEvents.do_orm_execute()` 事件处理程序 [ticket:5760]：

    +   修复了一个问题，即如果给定的实体或基类包含其向下类层次结构中的非映射混合时，`with_loader_criteria()` 函数将失败 [ticket:5766]。

    +   现在对 ORM “刷新”操作的情况下无条件禁用了 `with_loader_criteria()` 功能，包括加载延迟或过期的列属性以及像 `Session.refresh()` 这样的显式操作。这些加载必然是基于主键标识，其中额外的 WHERE 条件是不合适的。[ticket:5762]

    +   新增了新属性`ORMExecuteState.is_column_load`来指示`SessionEvents.do_orm_execute()`处理程序指示特定操作是针对主键定向的列属性加载，其中不应添加其他条件。`with_loader_criteria()` 函数现在无论如何都会忽略这些。[ticket:5761]

    +   修复了一个问题，即`ORMExecuteState.is_relationship_load` 属性对于许多延迟加载以及所有 selectinloads 都不会正确设置。该标志对于测试是否应向语句添加选项或是否已通过关系加载传播了它们至关重要。[ticket:5764]

    参考：[#5760](https://www.sqlalchemy.org/trac/ticket/5760), [#5761](https://www.sqlalchemy.org/trac/ticket/5761), [#5762](https://www.sqlalchemy.org/trac/ticket/5762), [#5764](https://www.sqlalchemy.org/trac/ticket/5764), [#5766](https://www.sqlalchemy.org/trac/ticket/5766)

+   **[orm] [bug]**

    修复了 1.4 回归，其中在与内部适应的 SQL 元素一起使用`Query.having()`（在继承场景中常见）的查询会因为错误的函数调用而失败。贡献者：esoh。

    参考：[#5781](https://www.sqlalchemy.org/trac/ticket/5781)

+   **[orm] [bug]**

    修复了一个问题，即根据多年来一直存在的文档使用 `sqlalchemy.ext.compiles` 扩展创建自定义可执行 SQL 构造的 API 如果仅使用了 `Executable, ClauseElement` 作为基类，如果想要使用 `Session.execute()` 将不再起作用，如果想要使用，需要额外的类。已解决，这样就不再需要这些额外的类。

+   **[orm] [bug] [regression]**

    修复了 ORM 工作单元回归，其中一个错误的“assert primary_key”语句干扰了实际上不考虑表中的列以使用真正的主键约束生成序列的主键生成序列，而是使用`Mapper.primary_key`来将某些列确定为“主要”。

    参考：[#5867](https://www.sqlalchemy.org/trac/ticket/5867)

### orm 声明式

+   **[orm] [declarative] [feature]**

    在 Declarative 中添加了一种替代的解析方案，可以从 dataclasses.Field 对象的“metadata”字典中提取 SQLAlchemy 列或映射属性。这允许完整的声明性映射与 dataclass 字段结合使用。

    另请参阅

    使用声明式字段映射预先存在的 dataclasses

    参考：[#5745](https://www.sqlalchemy.org/trac/ticket/5745)

### 引擎

+   **[engine] [feature]**

    特定于方言的构造，如`Insert.on_conflict_do_update()`现在可以在不需要指定显式方言对象的情况下原地字符串化。这些构造在调用`str()`、`print()`等时，现在内部会调用适当的方言而不是“默认”方言，后者不知道如何将其字符串化。该方法也适用于通用的模式级别创建/删除，如`AddConstraint`，它将调整其字符串化方言以适应其中的元素指示的方言，如`ExcludeConstraint`对象。

+   **[engine] [feature]**

    添加了新的执行选项`Connection.execution_options.logging_token`。此选项将在`Connection`执行语句时生成的日志消息中添加一个额外的每条消息令牌。此令牌不是日志记录器名称本身的一部分（该部分可以使用现有的`create_engine.logging_name`参数进行影响），因此适用于临时连接使用，而不会产生创建许多新记录器的副作用。该选项可以在`Connection`或`Engine`级别设置。

    另请参阅

    设置每个连接/子引擎令牌

    参考：[#5911](https://www.sqlalchemy.org/trac/ticket/5911)

+   **[engine] [bug] [sqlite]**

    修复了 2.0 版本中 `Engine` 的“future”版本中的 bug，在 `EngineEvents.begin()` 事件钩子期间发出 SQL 会导致由于自动开始而产生递归条件，影响了 SQLite 文档中记录的允许保存点和可序列化隔离支持的方法。

    参考：[#5845](https://www.sqlalchemy.org/trac/ticket/5845)

+   **[engine] [bug] [oracle] [postgresql]**

    调整了 cx_Oracle、asyncpg 和 pg8000 方言依赖的“setinputsizes”逻辑，以支持包含覆盖`TypeDecorator`的`TypeDecorator.get_dbapi_type()`方法。

+   **[engine] [bug]**

    将“future”关键字添加到`engine_from_config()`函数已知的单词列表中，以便在使用 `sqlalchemy.future = true` 或 `sqlalchemy.future = false` 这样的键时，“true” 和 “false” 值可以配置为“boolean”值。

### sql

+   **[sql] [feature]**

    实现了对“表值函数”的支持，包括 PostgreSQL 支持的其他语法，这是最常被请求的功能之一。表值函数是返回值或行列表的 SQL 函数，在 PostgreSQL 中非常普遍，特别是在 JSON 函数领域，其中“表值”通常被称为“记录”数据类型。Oracle 和 SQL Server 也支持表值函数。

    添加的功能包括：

    +   `FunctionElement.table_valued()` 修改器，从 SQL 函数创建类似表的可选择对象

    +   `TableValuedAlias` 结构，将 SQL 函数呈现为命名表

    +   支持 PostgreSQL 的特殊“派生列”语法，包括列名和有时数据类型，例如 `json_to_recordset` 函数，使用`TableValuedAlias.render_derived()`方法。

    +   支持 PostgreSQL 的“WITH ORDINALITY” 构造，使用`FunctionElement.table_valued.with_ordinality`参数

    +   支持将 SQL 函数作为列值标量的选择，这是由 PostgreSQL 和 Oracle 支持的语法，通过`FunctionElement.column_valued()`方法

    +   一种在不使用 FROM 子句的情况下从表值表达式中选择单个列的方法，通过`FunctionElement.scalar_table_valued()`方法。

    另请参阅

    表值函数 - 在 SQLAlchemy 统一教程中

    参考：[#3566](https://www.sqlalchemy.org/trac/ticket/3566)

+   **[sql] [usecase]**

    多次调用“returning”，例如`Insert.returning()`，现在可以链接以向 RETURNING 子句添加新列。

    参考：[#5695](https://www.sqlalchemy.org/trac/ticket/5695)

+   **[sql] [usecase]**

    添加了`Select.outerjoin_from()`方法，以补充`Select.join_from()`。

+   **[sql] [usecase]**

    调整了`Compiler`的“literal_binds”功能，以在绑定参数具有`None`作为值时呈现 NULL，无论是显式传递还是省略。之前的错误消息“没有可呈现值的绑定参数”已被移除，缺失或`None`值现在将在所有情况下呈现为 NULL。以前，由于内部重构，DML 语句开始呈现 NULL，但并未明确包含在测试覆盖范围内，现在已经包含在测试覆盖范围内。

    虽然不会引发错误，但当上下文在列比较的情况下，操作符不是“IS”/“IS NOT”时，会发出警告，指出从 SQL 的角度来看，这通常不是有用的。

    参考：[#5888](https://www.sqlalchemy.org/trac/ticket/5888)

+   **[sql] [bug]**

    修复了新的`Select.join()`方法中的问题，其中从当前 JOIN 链到正确的状态，导致类似“FROM a JOIN b <onclause>, b JOIN c <onclause>”而不是“FROM a JOIN b <onclause> JOIN c <onclause>”的表达式。

    参考：[#5858](https://www.sqlalchemy.org/trac/ticket/5858)

+   **[sql] [bug]**

    当将纯字符串传递给`Session.execute()`时，在“SQLALCHEMY_WARN_20”模式下会发出弃用警告。

    参考：[#5754](https://www.sqlalchemy.org/trac/ticket/5754)

+   **[sql] [bug] [orm]**

    根据用户反馈，对“lambda SQL”功能进行了各种修复，重点放在其在 `with_loader_criteria()` 功能中的使用上，这是它最常用的地方 [ticket:5760]:

    +   修复了 lambda 闭包变量中引用的布尔值 True/False 导致失败的问题。[ticket:5763]

    +   修复了 lambda 中产生绑定值的 Python 函数的无效检测；这种情况可能不可支持，因此会引发一个信息性错误，函数应在 lambda 之外调用。还添加了新的文档以进一步详细说明这种行为。[ticket:5770]

    +   现在 lambda 系统默认拒绝在闭包变量中使用非 SQL 元素，错误提示建议明确忽略不是 SQL 参数的闭包变量，或者根据哈希值指定一组特定的值作为缓存键的一部分。这在关键时刻防止了 lambda 系统假设 lambda 闭包中的任意对象适合缓存，同时默认拒绝忽略它们，防止它们的状态可能不是恒定的并对生成的 SQL 结构产生影响的情况。错误消息很详尽，还添加了新的文档以进一步详细说明这种行为。[ticket:5765]

    +   修复了针对 SQL 元素列表（如 `literal()` 对象）的 `in_()` 表达式无法正确适应的边缘情况支持。[ticket:5768]

    参考：[#5760](https://www.sqlalchemy.org/trac/ticket/5760), [#5763](https://www.sqlalchemy.org/trac/ticket/5763), [#5765](https://www.sqlalchemy.org/trac/ticket/5765), [#5768](https://www.sqlalchemy.org/trac/ticket/5768), [#5770](https://www.sqlalchemy.org/trac/ticket/5770)

+   **[sql] [bug] [mysql] [postgresql] [sqlite]**

    现在对一组选定的 DML 方法（目前全部属于`Insert` 构造）调用第二次时会引发一个信息丰富的错误消息，该消息会隐式取消先前的设置。受影响的方法包括：`on_conflict_do_update`, `on_conflict_do_nothing` (SQLite), `on_conflict_do_update`, `on_conflict_do_nothing` (PostgreSQL), `on_duplicate_key_update` (MySQL)

    参考文献：[#5169](https://www.sqlalchemy.org/trac/ticket/5169)

+   **[sql] [bug]**

    在新的`Values` 构造中修复了一个问题，其中传递对象的元组将会回退到每个值类型检测，而不是直接传递给 `Values` 的 `Column` 对象，该对象告诉 SQLAlchemy 预期的类型是什么。这会导致对于诸如枚举和 numpy 字符串之类的对象出现问题，这些对象实际上并不是必需的，因为给出了预期的类型。

    参考文献：[#5785](https://www.sqlalchemy.org/trac/ticket/5785)

+   **[sql] [bug]**

    修复了当内部访问对象时会错误地发出 `RemovedIn20Warning` 的问题，特别是当串化 SQL 构造时。

    参考文献：[#5717](https://www.sqlalchemy.org/trac/ticket/5717)

+   **[sql] [bug]**

    在 `Sequence` 和 `Identity` 对象中正确渲染 `cycle=False` 和 `order=False` 为 `NO CYCLE` 和 `NO ORDER`。

    参考文献：[#5722](https://www.sqlalchemy.org/trac/ticket/5722)

+   **[sql]**

    用显式的获取器和设置器`GenerativeSelect.get_label_style()`和`GenerativeSelect.set_label_style()`替换`Query.with_labels()`和`GenerativeSelect.apply_labels()`，以适应三种支持的标签样式：`LABEL_STYLE_DISAMBIGUATE_ONLY`、`LABEL_STYLE_TABLENAME_PLUS_COL`和`LABEL_STYLE_NONE`。

    此外，对于 Core 和“未来风格”ORM 查询，`LABEL_STYLE_DISAMBIGUATE_ONLY`现在是默认的标签样式。这种样式与现有的“无标签”样式不同之处在于，在列名冲突的情况下应用标签；使用`LABEL_STYLE_NONE`，重复的列名在任何情况下都无法通过名称访问。

    对于标签具有重要意义的情况，即子查询的`.c`集合能够明确引用所有列，`LABEL_STYLE_DISAMBIGUATE_ONLY`的行为现在对涉及此行为的所有 SQLAlchemy 功能（包括 Core 和 ORM）已经足够。自 SQLAlchemy 1.0 以来，结果集行通常与列构造在位置上对齐。

    对于使用`Query`的传统 ORM 查询，继续使用`LABEL_STYLE_TABLENAME_PLUS_COL`应用的表加列名标签样式，以便现有的测试套件和日志记录设施默认情况下不会看到行为变化。

    参考：[#4757](https://www.sqlalchemy.org/trac/ticket/4757)

### 模式

+   **[模式] [特性]**

    添加了`TypeEngine.as_generic()`，用于将方言特定类型（例如`sqlalchemy.dialects.mysql.INTEGER`）映射到“最佳匹配”通用 SQLAlchemy 类型，本例中为`Integer`。感谢 Andrew Hannigan 的拉取请求。

    另请参阅

    使用与数据库无关的类型进行反射 - 示例用法

    参考：[#5659](https://www.sqlalchemy.org/trac/ticket/5659)

+   **[模式] [用例]**

    `DDLEvents.column_reflect()`事件现在可以应用于`MetaData`对象，它将对该集合本地的`Table`对象产生影响。

    另请参阅

    `DDLEvents.column_reflect()`

    从反射表自动化列命名方案 - 在 ORM 映射文档中

    拦截列定义 - 在 Automap 文档中

    参考：[#5712](https://www.sqlalchemy.org/trac/ticket/5712)

+   **[schema] [usecase]**

    向`CreateTable`、`DropTable`、`CreateIndex`和`DropIndex`构造添加了参数`CreateTable.if_not_exists`、`CreateIndex.if_not_exists`、`DropTable.if_exists`和`DropIndex.if_exists`，导致在 CREATE/DROP 中添加“IF NOT EXISTS” / “IF EXISTS” DDL。这些短语并不被所有数据库接受，如果数据库不支持它，操作将失败，因为在单个 DDL 语句的范围内没有类似兼容的回退。拉取请求由 Ramon Williams 提供。

    参考：[#2843](https://www.sqlalchemy.org/trac/ticket/2843)

+   **[schema] [changed]**

    修改了`Identity`构造的行为，使其应用于`Column`时，将自动暗示`Column.nullable`的值默认为`False`，类似于将`Column.primary_key`参数设置为`True`时的方式。这与所有支持的数据库的默认行为相匹配，其中`IDENTITY`意味着`NOT NULL`。 PostgreSQL 后端是唯一支持向`IDENTITY`列添加`NULL`的后端，这里通过同时将`Column.nullable`参数传递为`True`来支持。

    参考：[#5775](https://www.sqlalchemy.org/trac/ticket/5775)

### asyncio

+   **[asyncio] [usecase]**

    `AsyncEngine`、`AsyncConnection`和`AsyncTransaction`对象可以使用 Python `==`或`!=`进行比较，这将根据它们代理的“同步”对象比较两个给定对象。这在特定情况下特别适用于`AsyncTransaction`，其中多个`AsyncTransaction`实例可以代理到相同的同步`Transaction`，并且实际上是等效的。`AsyncConnection.get_transaction()`方法当前每次都会返回一个新的代理`AsyncTransaction`，因为`AsyncTransaction`与其来源的`AsyncConnection`没有其他状态关联。

+   **[asyncio] [bug]**

    调整了提供对 Python asyncio 的支持的 greenlet 集成，以适应处理 Python `contextvars`（引入于 Python 3.7）的情况，适用于大于 0.4.17 的`greenlet`版本。Greenlet 版本 0.4.17 以一种不兼容的方式自动处理 contextvars；我们已与 greenlet 作者协调，在 0.4.17 之后的版本中添加了此项首选 API，现在受 SQLAlchemy 的 greenlet 集成支持。对于早于 0.4.17 的 greenlet 版本，不需要行为更改，版本 0.4.17 本身被阻止在依赖项中。

    引用：[#5615](https://www.sqlalchemy.org/trac/ticket/5615)

+   **[asyncio] [bug]**

    实现了对 `AsyncSession` 的“连接绑定”功能，即能够传递 `AsyncConnection` 来创建 `AsyncSession`。之前，此用例未实现，并且当传递连接时会使用关联的引擎。这修复了“将会话加入到外部事务”用例无法正确工作的问题。另外，添加了方法 `AsyncConnection.in_transaction()`、`AsyncConnection.in_nested_transaction()`、`AsyncConnection.get_transaction()`、`AsyncConnection.get_nested_transaction()` 以及 `AsyncConnection.info` 属性。

    参考文献：[#5811](https://www.sqlalchemy.org/trac/ticket/5811)

+   **[asyncio] [bug]**

    修复了 asyncio 连接池中的错误，此前会引发 `asyncio.TimeoutError` 而不是 `TimeoutError`。还修复了使用异步引擎时将 `create_engine.pool_timeout` 参数设置为零时的情况，此前会忽略超时并阻塞而不是像常规 `QueuePool` 那样立即超时。

    参考文献：[#5827](https://www.sqlalchemy.org/trac/ticket/5827)

+   **[asyncio] [bug] [pool]**

    当使用 asyncio 引擎时，连接池现在会在其跟踪对象被垃圾回收时分离并丢弃未显式关闭/返回到池中的连接，发出警告表明连接未正确关闭。由于此操作发生在 Python gc 终结器期间，因此不安全运行任何 IO 操作，包括事务回滚或连接关闭，因为这通常会在事件循环之外。

    默认情况下在 async dpapis 上使用的 `AsyncAdaptedQueue` 应该在首次使用时实例化队列，以避免将其绑定到可能错误的事件循环。

    参考：[#5823](https://www.sqlalchemy.org/trac/ticket/5823)

+   **[asyncio]**

    现在 SQLAlchemy 异步模式会检测并在使用不兼容 asyncio 的 DBAPI 时引发信息性错误。使用标准的 `DBAPI` 与异步 SQLAlchemy 将导致其像任何同步调用一样阻塞，中断执行中的 asyncio 循环。

### postgresql

+   **[postgresql] [usecase]**

    向 `ExcludeConstraint` 对象添加了新参数 `ExcludeConstraint.ops`，以支持此约束的操作符类规范。感谢 Alon Menczer 的拉取请求。

    此更改也已**回溯**至：1.3.21

    参考：[#5604](https://www.sqlalchemy.org/trac/ticket/5604)

+   **[postgresql] [usecase]**

    为 asyncpg 方言的 DBAPI 适配层添加了一个读/写的 `.autocommit` 属性。这样，当需要直接与 DBAPI 连接使用“autocommit” 的 DBAPI 特定方案时，可以使用同一个`.autocommit` 属性，该属性适用于 psycopg2 和 pg8000。

+   **[postgresql] [changed]**

    修复了 psycopg2 方言在 Python 3 下静默传递 `use_native_unicode=False` 标志而实际上在 Python 3 下没有任何效果的问题，因为 psycopg2 DBAPI 在 Python 3 下无条件使用 Unicode。在 Python 3 下使用此功能现在会引发 `ArgumentError`。添加了对 Python 2 的测试支持。

+   **[postgresql] [performance]**

    通过在每个连接基础上缓存 asyncpg 方言的 asyncpg PreparedStatement 对象，增强了 asyncpg 方言的性能。对于在一组池化连接上使用相同语句的测试用例，这似乎提供了 10-20% 的速度改进。缓存大小可调整，也可以禁用。

    另请参阅

    预编译语句缓存

+   **[postgresql] [bug] [mysql]**

    修复了在 1.3.2 中引入的针对 PostgreSQL 方言的回归，也在 1.3.18 中复制到了 MySQL 方言的功能，其中对非`Table`构造的使用，例如`text()`作为`Select.with_for_update.of`参数将无法在 PostgreSQL 或 MySQL 编译器中正确处理。

    此更改也**回溯**到：1.3.21

    参考：[#5729](https://www.sqlalchemy.org/trac/ticket/5729)

+   **[postgresql] [bug]**

    修复了一个小的回归，即在初始化时“show standard_conforming_strings”的查询将被发出，即使检测到服务器版本信息小于版本 8.2，以前只会发生在服务器版本 8.2 或更高版本。该查询在报告比此值旧的 PG 服务器版本的 Amazon Redshift 上失败。

    参考：[#5698](https://www.sqlalchemy.org/trac/ticket/5698)

+   **[postgresql] [bug]**

    已为`Column`对象以及 ORM 仪器化属性建立了支持，作为传递给`Insert.on_conflict_do_update()`和`Insert.on_conflict_do_update()`方法中的`set_`字典的键，这些键与目标`Table`的`.c`集合中的`Column`对象相匹配。以前，只期望字符串列名；列表达式将被假定为一个超出表达式，将与警告一起完全呈现。

    参考：[#5722](https://www.sqlalchemy.org/trac/ticket/5722)

+   **[postgresql] [bug] [asyncio]**

    修复了 asyncpg 方言中的错误，即在“commit”期间失败或更少可能在“rollback”期间失败应取消整个事务；不再可能发出回滚。以前，连接将继续等待无法成功的回滚，因为 asyncpg 将拒绝它。

    参考：[#5824](https://www.sqlalchemy.org/trac/ticket/5824)

### mysql

+   **[mysql] [feature]**

    在使用 asyncio SQLAlchemy 扩展时，添加了对 aiomysql 驱动程序的支持。

    另请参阅

    aiomysql

    参考：[#5747](https://www.sqlalchemy.org/trac/ticket/5747)

+   **[mysql] [bug] [reflection]**

    修复了在仅包含值中的小数点的 MariaDB 上反射服务器默认值时无法正确反映的问题，导致反映表缺乏任何服务器默认值。

    此更改也**回溯**至：1.3.21

    参考：[#5744](https://www.sqlalchemy.org/trac/ticket/5744)

### sqlite

+   **[sqlite] [usecase]**

    为 SQLite 实现了 INSERT… ON CONFLICT 子句。感谢 Ramon Williams 的拉取请求。

    另请参阅

    插入…冲突时执行（Upsert）

    参考：[#4010](https://www.sqlalchemy.org/trac/ticket/4010)

+   **[sqlite] [bug]**

    在使用 sqlite 时，使用 python `re.search()`代替`re.match()`作为`Column.regexp_match()`方法的操作。这样可以匹配其他数据库上正则表达式的行为，以及众所周知的 SQLite 插件的行为。

    参考：[#5699](https://www.sqlalchemy.org/trac/ticket/5699)

### mssql

+   **[mssql] [bug] [datatypes] [mysql]**

    当使用`Comparator.as_float()`方法从 JSON 字符串中提取浮点数和/或十进制值时，十进制精度和行为已得到改进；当 JSON 字符串中的数值具有许多有效数字时，以前，MySQL 后端会截断具有许多有效数字的值，而 SQL Server 后端会由于 DECIMAL 转换具有不足有效数字而引发异常。现在，两个后端都使用与 FLOAT 兼容的方法，不会为浮点值硬编码有效数字。对于精确数值，添加了一个新方法`Comparator.as_numeric()`，它接受精度和标度参数，并将值作为 Python `Decimal`对象返回，假设 DBAPI 支持它（除了 pysqlite）。

    参考：[#5788](https://www.sqlalchemy.org/trac/ticket/5788)

### oracle

+   **[oracle] [bug]**

    修复了由于[#5755](https://www.sqlalchemy.org/trac/ticket/5755)导致的回归，该回归为 Oracle 实现了隔离级别支持。据报道，许多 Oracle 帐户实际上没有权限查询`v$transaction`视图，因此当在数据库连接时失败时，此功能已被更改为优雅地回退，其中方言将假定“READ COMMITTED”是 SQLAlchemy 1.3.21 之前的默认隔离级别。然而，现在必须明确使用`Connection.get_isolation_level()`方法引发异常，因为具有此限制的 Oracle 数据库明确禁止用户读取当前隔禅级别。

    此更改也**回溯**至：1.3.22

    参考：[#5784](https://www.sqlalchemy.org/trac/ticket/5784)

+   **[oracle] [bug]**

    现在，Oracle 的二阶段事务在基本水平上不再被弃用。在得到 cx_Oracle 开发人员的支持后，我们可以提供一些限制的基本 xid + begin/prepare 支持，这将在 cx_Oracle 的即将发布的版本中更完整地工作。目前不支持两阶段“恢复”。

    参考：[#5884](https://www.sqlalchemy.org/trac/ticket/5884)

+   **[oracle] [bug]**

    Oracle 方言现在使用 `select sys_context( 'userenv', 'current_schema' ) from dual` 来获取默认模式名称，而不是 `SELECT USER FROM DUAL`，以适应 Oracle 下会话本地模式名称的更改。

    参考：[#5716](https://www.sqlalchemy.org/trac/ticket/5716)

### 测试

+   **[tests] [usecase] [pool]**

    改进文档并为亚秒级连接池超时添加测试。感谢 Jordan Pittier 提交的拉取请求。

    参考：[#5582](https://www.sqlalchemy.org/trac/ticket/5582)

### 杂项

+   **[usecase] [pool]**

    引擎连接例程的内部机制已经改变，现在可以保证使用 `insert=True` 建立的 `PoolEvents.connect()` 处理程序的用户定义事件处理程序将允许在任何特定于方言的初始化启动之前**确实**运行事件处理程序，尤其是当它执行诸如检测默认模式名称之类的操作时。以前，在大多数情况下会发生这种情况，但不是无条件的。在模式文档中添加了一个新示例，说明如何在连接时建立“默认模式名称”。

    参考：[#5497](https://www.sqlalchemy.org/trac/ticket/5497), [#5708](https://www.sqlalchemy.org/trac/ticket/5708)

+   **[bug] [reflection]**

    修复了在反射相关表时，当一个相关表被反射时，内部调用了现在已弃用的 `autoload` 参数的 bug。

    参考：[#5684](https://www.sqlalchemy.org/trac/ticket/5684)

+   **[bug] [pool]**

    修复了使用关键字指定的连接池事件，尤其是 `insert=True`，在设置事件时会丢失的回归问题。这将阻止需要在方言级事件之前触发的启动事件正常工作。

    参考：[#5708](https://www.sqlalchemy.org/trac/ticket/5708)

+   **[bug] [pool] [pypy]**

    修复了在 pypy 下，如果已检出的连接超出范围而未关闭，则连接池不会将连接返回到池中或在垃圾回收时被终结的问题。这是一个长期存在的问题，因为 pypy 的 GC 行为不会调用弱引用的终结器，如果它们相对于另一个也正在被垃圾回收的对象。现在维护了与相关记录的强引用，以便弱引用有一个强引用的“基础”来触发。

    参考：[#5842](https://www.sqlalchemy.org/trac/ticket/5842)

### 通用

+   **[general] [bug]**

    修复了一个 SQLite 源文件，在“INSERT..ON CONFLICT”功能中引入了没有源编码的 docstring 中的非 ASCII 字符，这会导致在 Python 2 下失败。

### platform

+   **[platform] [performance]**

    在导入时调整了与内部类生成相关的一些元素，这导致导入库所花费的时间明显增加，与 1.3 版本相比，现在的时间比 1.3 版本慢了约 20-30%，而不是 200%。

    参考：[#5681](https://www.sqlalchemy.org/trac/ticket/5681)

### orm

+   **[orm] [usecase]**

    向`ORMExecuteState.bind_mapper`和`ORMExecuteState.all_mappers`访问器添加到`ORMExecuteState`事件对象，以便处理程序可以响应 ORM 语句执行中涉及的目标映射器和/或映射类。

+   **[orm] [usecase] [asyncio]**

    添加`AsyncSession.scalar()`，`AsyncSession.get()`以及对`sessionmaker.begin()`的支持，使其作为异步上下文管理器与`AsyncSession`一起工作。还添加了`AsyncSession.in_transaction()`访问器。

    参考：[#5796](https://www.sqlalchemy.org/trac/ticket/5796)，[#5797](https://www.sqlalchemy.org/trac/ticket/5797)，[#5802](https://www.sqlalchemy.org/trac/ticket/5802)

+   **[orm] [changed]**

    现在，在`configure_mappers()`函数中发生的映射器“配置”现在按照每个注册表的基础组织。例如，这允许配置某个声明基类中的映射器，但不配置内存中还存在的另一个基类的映射器。目标是通过仅对所需的映射器集运行“配置”过程来提供减少应用程序启动时间的手段。这还添加了`registry.configure()`方法，该方法仅对特定注册表中的本地映射器运行配置。

    参考：[#5897](https://www.sqlalchemy.org/trac/ticket/5897)

+   **[orm] [bug]**

    当`relationship.secondary`参数传递了映射类或字符串映射类名称时，添加了全面的检查和信息丰富的错误消息。这是一个极其常见的错误，需要清晰的提示。

    另外，添加了一个新规则到类注册解析中，关于`relationship.secondary`参数，如果一个映射类及其表具有相同的字符串名称，解析此参数时将优先选择`Table`。在所有其他情况下，如果类和表共享相同的名称，仍然优先选择类。

    这个更改也被**回溯**到：1.3.21

    参考：[#5774](https://www.sqlalchemy.org/trac/ticket/5774)

+   **[orm] [bug]**

    修复了涉及 ORM 事件的`restore_load_context`选项的错误，例如`InstanceEvents.load()`，使得标志不会传递给在首次建立事件处理程序之后映射的子类。

    这个更改也被**回溯**到：1.3.21

    参考：[#5737](https://www.sqlalchemy.org/trac/ticket/5737)

+   **[orm] [bug] [regression]**

    修复了类似于`Connection`的新`Session`中的问题，其中新的“autobegin”逻辑如果在`SessionEvents.after_transaction_create()`事件钩子中执行 SQL，可能会陷入可重入（递归）状态。

    参考：[#5845](https://www.sqlalchemy.org/trac/ticket/5845)

+   **[orm] [bug] [unitofwork]**

    改进了工作单元拓扑排序系统，使得拓扑排序现在基于输入集的排序是确定性的，输入集本身在映射器级别排序，因此受影响的映射器的相同输入应该每次产生相同的输出，而且在不互相依赖的映射器/表之间。这进一步降低了死锁的几率，如在更新多个不相关表之间的刷新时可能观察到的，从而生成行锁。

    参考：[#5735](https://www.sqlalchemy.org/trac/ticket/5735)

+   **[orm] [bug]**

    修复了回归问题，其中`Bundle.single_entity` 标志会对`Bundle` 产生影响，即使未设置。此外，此标志是遗留的，因为它只对`Query` 对象有意义，而不适用于 2.0 风格的执行。在新风格执行时使用时会发出弃用警告。

    参考：[#5702](https://www.sqlalchemy.org/trac/ticket/5702)

+   **[orm] [bug]**

    修复了回归问题，其中针对普通可选择项创建一个`aliased` 构造并包含名称会引发断言错误。

    参考：[#5750](https://www.sqlalchemy.org/trac/ticket/5750)

+   **[orm] [bug]**

    与 Core 中 lambda 标准系统的修复相关，在 ORM 中实现了各种修复`with_loader_criteria()` 功能以及经常与之一起使用的`SessionEvents.do_orm_execute()` 事件处理程序 [ticket:5760]:

    +   修复了`with_loader_criteria()` 函数在给定实体或基类包含非映射混入类的情况下失败的问题 [ticket:5766]

    +   现在，`with_loader_criteria()` 功能在 ORM “刷新”操作的情况下无条件禁用，包括加载延迟或过期列属性以及显式操作如`Session.refresh()`。这些加载必须基于主键标识，额外的 WHERE 条件是不合适的。 [ticket:5762]

    +   添加了新属性`ORMExecuteState.is_column_load`，用于指示`SessionEvents.do_orm_execute()` 处理程序是主键定向列属性加载，不应添加额外条件。`with_loader_criteria()` 函数现在在任何情况下都会忽略这些。 [ticket:5761]

    +   修复了 `ORMExecuteState.is_relationship_load` 属性未正确设置为许多延迟加载以及所有 selectinloads 的问题。 该标志是必不可少的，以便测试是否应该向语句添加选项，或者它们是否已通过关系加载传播。[ticket:5764]

    参考：[#5760](https://www.sqlalchemy.org/trac/ticket/5760), [#5761](https://www.sqlalchemy.org/trac/ticket/5761), [#5762](https://www.sqlalchemy.org/trac/ticket/5762), [#5764](https://www.sqlalchemy.org/trac/ticket/5764), [#5766](https://www.sqlalchemy.org/trac/ticket/5766)

+   **[orm] [bug]**

    修复了 1.4 版本中的回归，其中在与内部调整的 SQL 元素一起使用 `Query.having()` 时，查询（在继承方案中常见）会由于调用的错误函数而失败。感谢 esoh 提交的拉取请求。

    参考：[#5781](https://www.sqlalchemy.org/trac/ticket/5781)

+   **[orm] [bug]**

    修复了一个问题，即根据多年来一直存在的文档创建自定义可执行 SQL 构造的 API，如果只使用 `Executable, ClauseElement` 作为基类，则不再能使用 `Session.execute()`。 现在已解决了这个问题，因此不再需要那些额外的类。

+   **[orm] [bug] [regression]**

    修复了 ORM 单元的回归，其中错误的“assert primary_key”语句干扰了实际上并不考虑表中的列来使用真实主键约束的主键生成序列，而是使用 `Mapper.primary_key` 来将某些列确定为“主键”。

    参考：[#5867](https://www.sqlalchemy.org/trac/ticket/5867)

### ORM 声明式

+   **[orm] [declarative] [feature]**

    在 Declarative 中添加了一种替代解析方案，该方案将从 dataclasses.Field 对象的“metadata”字典中提取 SQLAlchemy 列或映射属性。 这允许将完整的声明性映射与数据类字段结合使用。

    另请参见

    使用声明式字段映射预先存在的数据类

    参考：[#5745](https://www.sqlalchemy.org/trac/ticket/5745)

### 引擎

+   **[engine] [feature]**

    现在，诸如`Insert.on_conflict_do_update()`之类的特定方言构造可以在不需要指定显式方言对象的情况下原地字符串化。当调用这些构造进行`str()`、`print()`等操作时，现在内部会调用适当的方言而不是“默认”方言，后者不知道如何字符串化这些内容。该方法还适用于通用的模式级别创建/删除，例如`AddConstraint`，它将调整其字符串化方言以适应其中的元素指示的方言，例如`ExcludeConstraint`对象。

+   **[engine] [feature]**

    添加了新的执行选项`Connection.execution_options.logging_token`。此选项将在`Connection`执行语句时，为生成的日志消息添加一个额外的每条消息令牌。该令牌不是日志记录器名称本身的一部分（该部分可以使用现有的`create_engine.logging_name`参数进行影响），因此适用于临时连接使用，而不会产生创建许多新日志记录器的副作用。该选项可以在`Connection`或`Engine`的级别设置。

    另请参阅

    设置每个连接/子引擎令牌

    参考：[#5911](https://www.sqlalchemy.org/trac/ticket/5911)

+   **[engine] [bug] [sqlite]**

    修复了 2.0“future”版本中`Engine`在`EngineEvents.begin()`事件钩子期间发出 SQL 会导致由于自动开始而引起的递归条件的错误，影响了 SQLite 文档中记录的允许保存点和可串行化隔离支持的配方等内容。

    参考：[#5845](https://www.sqlalchemy.org/trac/ticket/5845)

+   **[engine] [bug] [oracle] [postgresql]**

    调整了 cx_Oracle、asyncpg 和 pg8000 方言依赖的“setinputsizes”逻辑，以支持包含覆盖`TypeDecorator.get_dbapi_type()` 方法的`TypeDecorator`。

+   **[引擎] [错误]**

    将“future”关键字添加到`engine_from_config()` 函数已知的单词列表中，以便在使用 `sqlalchemy.future = true` 或 `sqlalchemy.future = false` 这样的键时，可以将值“true”和“false” 配置为“布尔”值。

### sql

+   **[sql] [特性]**

    实现了对“表值函数”的支持，以及 PostgreSQL 支持的其他语法，这是最常被请求的功能之一。表值函数是返回值列表或行的 SQL 函数，在 PostgreSQL 中广泛存在于 JSON 函数领域，其中“表值”通常被称为“记录”数据类型。Oracle 和 SQL Server 也支持表值函数。

    添加的功能包括:

    +   `FunctionElement.table_valued()` 修改器，从 SQL 函数创建类似表的可选择对象

    +   一个`TableValuedAlias` 构造，将 SQL 函数呈现为一个命名表

    +   支持 PostgreSQL 的特殊“派生列”语法，包括列名和有时数据类型，例如 `json_to_recordset` 函数，使用`TableValuedAlias.render_derived()` 方法。

    +   使用`FunctionElement.table_valued.with_ordinality` 参数支持 PostgreSQL 的“WITH ORDINALITY” 构造

    +   支持从 SQL 函数选择作为列值标量，这是 PostgreSQL 和 Oracle 支持的语法，通过`FunctionElement.column_valued()` 方法

    +   通过`FunctionElement.scalar_table_valued()` 方法，从表值表达式中选择单个列而不使用 FROM 子句的方法。

    另请参阅

    表值函数 - 在 SQLAlchemy 统一教程 中

    参考：[#3566](https://www.sqlalchemy.org/trac/ticket/3566)

+   **[sql] [usecase]**

    现在可以多次调用“returning”，例如 `Insert.returning()`，以添加新列到 RETURNING 子句。

    参考：[#5695](https://www.sqlalchemy.org/trac/ticket/5695)

+   **[sql] [usecase]**

    添加了 `Select.outerjoin_from()` 方法，以补充 `Select.join_from()`。

+   **[sql] [usecase]**

    调整了 `Compiler` 的“literal_binds”功能，对于将 `None` 作为值的绑定参数，现在会渲染为 NULL，无论是显式传递还是省略。之前的错误消息“绑定参数没有可渲染的值”已被移除，现在在所有情况下，缺失或 `None` 值都会渲染为 NULL。之前，由于内部重构，DML 语句开始渲染为 NULL，但并未明确包含在测试覆盖范围内，现在已经包含。

    当上下文位于列比较之内，且操作符不是“IS”/“IS NOT”时，会发出警告，表示从 SQL 视角来看这并不通常有用，尽管没有引发错误。

    参考：[#5888](https://www.sqlalchemy.org/trac/ticket/5888)

+   **[sql] [bug]**

    修复了新 `Select.join()` 方法中从当前 JOIN 进行链式调用时未查看正确状态的问题，导致表达式类似于“FROM a JOIN b <onclause>, b JOIN c <onclause>”而不是“FROM a JOIN b <onclause> JOIN c <onclause>”。

    参考：[#5858](https://www.sqlalchemy.org/trac/ticket/5858)

+   **[sql] [bug]**

    当以“SQLALCHEMY_WARN_20”模式传递纯字符串给 `Session.execute()` 时，会发出弃用警告。

    参考：[#5754](https://www.sqlalchemy.org/trac/ticket/5754)

+   **[sql] [bug] [orm]**

    根据用户反馈，对“lambda SQL”功能进行了广泛的修复，重点放在其在 `with_loader_criteria()` 功能中的使用上，这是最常用的地方 [ticket:5760]：

    +   修复了在 lambda 的闭包变量中引用布尔值 True/False 会导致失败的问题。[ticket:5763]

    +   修复了对于嵌入在 lambda 中产生绑定值的 Python 函数的检测不起作用的问题；这种情况可能不支持，因此会显示一个信息丰富的错误，函数应该在 lambda 之外调用。新文档已添加以进一步详细说明此行为。[ticket:5770]

    +   默认情况下，lambda 系统现在拒绝在 lambda 的闭包变量中使用非 SQL 元素，错误提示建议两种选择：要么明确忽略不是 SQL 参数的闭包变量，要么根据哈希值指定一组特定的值作为缓存键的一部分。这在关键时刻防止了 lambda 系统假设 lambda 的闭包中的任意对象适合缓存，同时拒绝默认忽略它们，防止它们的状态可能不是恒定的并对生成的 SQL 结构产生影响的情况。错误消息是全面的，新文档已添加以进一步详细说明此行为。[ticket:5765]

    +   修复了针对 SQL 元素列表（如`literal()` 对象）的`in_()`表达式无法正确适应的边缘情况。[ticket:5768]

    参考：[#5760](https://www.sqlalchemy.org/trac/ticket/5760), [#5763](https://www.sqlalchemy.org/trac/ticket/5763), [#5765](https://www.sqlalchemy.org/trac/ticket/5765), [#5768](https://www.sqlalchemy.org/trac/ticket/5768), [#5770](https://www.sqlalchemy.org/trac/ticket/5770)

+   **[sql] [bug] [mysql] [postgresql] [sqlite]**

    对于一组选定的 DML 方法（目前所有属于`Insert` 构造的方法），如果它们被第二次调用，将会显示一个信息丰富的错误消息，这将隐式地取消之前的设置。被修改的方法包括：`on_conflict_do_update`, `on_conflict_do_nothing`（SQLite），`on_conflict_do_update`, `on_conflict_do_nothing`（PostgreSQL），`on_duplicate_key_update`（MySQL）

    参考：[#5169](https://www.sqlalchemy.org/trac/ticket/5169)

+   **[sql] [bug]**

    修复了新的`Values`构造中的问题，其中传递对象元组会回退到每个值类型检测，而不是利用直接传递给`Values`的`Column`对象，告诉 SQLAlchemy 预期的类型是什么。这会导致诸如枚举和 numpy 字符串等对象出现问题，这些对象实际上并不是必需的，因为已经给出了预期的类型。

    参考：[#5785](https://www.sqlalchemy.org/trac/ticket/5785)

+   **[sql] [bug]**

    修复了在内部访问对象时，特别是在字符串化 SQL 构造时，会错误地发出`RemovedIn20Warning`的问题。

    参考：[#5717](https://www.sqlalchemy.org/trac/ticket/5717)

+   **[sql] [bug]**

    在`Sequence`和`Identity`对象中，正确地将`cycle=False`和`order=False`渲染为`NO CYCLE`和`NO ORDER`。

    参考：[#5722](https://www.sqlalchemy.org/trac/ticket/5722)

+   **[sql]**

    用显式的获取器和设置器`GenerativeSelect.get_label_style()`和`GenerativeSelect.set_label_style()`替换`Query.with_labels()`和`GenerativeSelect.apply_labels()`，以适应三种支持的标签样式：`LABEL_STYLE_DISAMBIGUATE_ONLY`、`LABEL_STYLE_TABLENAME_PLUS_COL`和`LABEL_STYLE_NONE`。

    此外，对于 Core 和“未来风格”ORM 查询，`LABEL_STYLE_DISAMBIGUATE_ONLY`现在是默认的标签样式。这种样式与现有的“无标签”样式不同，因为在列名冲突的情况下应用标签；使用`LABEL_STYLE_NONE`，重复的列名在任何情况下都无法通过名称访问。

    对于标签很重要的情况，即子查询的`.c`集合能够明确地引用所有列，`LABEL_STYLE_DISAMBIGUATE_ONLY`的行为现在对涉及此行为的所有 SQLAlchemy 功能足够了。自 SQLAlchemy 1.0 以来，结果集行通常与列构造在位置上对齐。

    对于使用`Query`进行传统 ORM 查询的情况，继续使用`LABEL_STYLE_TABLENAME_PLUS_COL`应用的表加列命名风格，以便现有的测试套件和日志设施默认情况下不会看到行为上的变化。

    参考：[#4757](https://www.sqlalchemy.org/trac/ticket/4757)

### 模式

+   **[模式] [特性]**

    添加了 `TypeEngine.as_generic()` 以映射方言特定类型，例如 `sqlalchemy.dialects.mysql.INTEGER`，与“最佳匹配”通用的 SQLAlchemy 类型，本例中为 `Integer`。拉取请求由 Andrew Hannigan 提供。

    请参阅

    使用数据库无关类型进行反射 - 示例用法

    参考资料：[#5659](https://www.sqlalchemy.org/trac/ticket/5659)

+   **[模式] [用例]**

    现在可以将 `DDLEvents.column_reflect()` 事件应用于 `MetaData` 对象，在那里它将对该集合中的本地 `Table` 对象生效。

    请参阅

    `DDLEvents.column_reflect()`

    从反射表自动化列命名方案 - 在 ORM 映射文档中

    拦截列定义 - 在 Automap 文档中

    参考资料：[#5712](https://www.sqlalchemy.org/trac/ticket/5712)

+   **[模式] [用例]**

    向 `CreateTable.if_not_exists`、`CreateIndex.if_not_exists`、`DropTable.if_exists` 和 `DropIndex.if_exists` 参数添加了“IF NOT EXISTS” / “IF EXISTS” DDL，这些参数被添加到 `CreateTable`、`DropTable`、`CreateIndex` 和 `DropIndex` 构造中。这些短语不被所有数据库接受，如果数据库不支持这些短语，操作将失败，因为在单个 DDL 语句的范围内没有类似的兼容回退。拉取请求由 Ramon Williams 提供。

    参考资料：[#2843](https://www.sqlalchemy.org/trac/ticket/2843)

+   **[schema] [changed]**

    修改了`Identity`构造的行为，使其在应用于`Column`时，将自动暗示`Column.nullable`的值应默认为`False`，类似于将`Column.primary_key`参数设置为`True`时的方式。这与所有支持的数据库的默认行为相匹配，其中`IDENTITY`暗示`NOT NULL`。 PostgreSQL 后端是唯一支持向`IDENTITY`列添加`NULL`的后端，在此通过同时为`Column.nullable`参数传递`True`值来支持这一点。

    引用：[#5775](https://www.sqlalchemy.org/trac/ticket/5775)

### asyncio

+   **[asyncio] [usecase]**

    `AsyncEngine`、`AsyncConnection`和`AsyncTransaction`对象可以使用 Python 的`==`或`!=`进行比较，这将根据它们代理向的“sync”对象比较给定的两个对象。这对于`AsyncTransaction`特别有用，因为在某些情况下，特别是对于`AsyncTransaction`，多个实例可能代理向同一个同步`Transaction`，并且实际上是等价的。`AsyncConnection.get_transaction()`方法当前每次都会返回一个新的代理`AsyncTransaction`，因为`AsyncTransaction`否则不会与其原始`AsyncConnection`有状态地关联。

+   **[asyncio] [bug]**

    调整了绿色线程集成，为了支持 Python asyncio 在 SQLAlchemy 中的使用，以适应处理 Python `contextvars`（引入于 Python 3.7）对于`greenlet`版本大于 0.4.17 的情况。Greenlet 版本 0.4.17 以一种不兼容的方式自动处理 contextvars；我们已经与 greenlet 作者协调，在 0.4.17 之后的版本中添加了一个首选的 API，现在 SQLAlchemy 的 greenlet 集成支持这个 API。对于早于 0.4.17 的 greenlet 版本，不需要行为更改，版本 0.4.17 本身被阻止作为依赖项。

    参考：[#5615](https://www.sqlalchemy.org/trac/ticket/5615)

+   **[asyncio] [bug]**

    实现了“连接绑定”功能，用于`AsyncSession`，即可以传递一个`AsyncConnection`来创建一个`AsyncSession`。之前，这种用例尚未实现，当传递连接时会使用关联的引擎。这修复了“将会话加入外部事务”用例对于`AsyncSession`无法正常工作的问题。此外，添加了方法`AsyncConnection.in_transaction()`、`AsyncConnection.in_nested_transaction()`、`AsyncConnection.get_transaction()`、`AsyncConnection.get_nested_transaction()` 以及`AsyncConnection.info` 属性。

    参考：[#5811](https://www.sqlalchemy.org/trac/ticket/5811)

+   **[asyncio] [bug]**

    修复了 asyncio 连接池中的 bug，此前会引发`asyncio.TimeoutError`而不是`TimeoutError`。还修复了在使用异步引擎时将`create_engine.pool_timeout`参数设置为零时会忽略超时并阻塞而不是立即超时的问题，这与常规`QueuePool`的行为不同。

    参考：[#5827](https://www.sqlalchemy.org/trac/ticket/5827)

+   **[asyncio] [错误] [池]**

    在使用 asyncio 引擎时，连接池现在会在跟踪对象被垃圾回收时分离和丢弃未显式关闭/返回到池中的池化连接，并发出警告，指出连接未正确关闭。由于此操作发生在 Python gc 终结器期间，因此不安全运行任何 IO 操作，包括事务回滚或连接关闭，因为这通常会在事件循环之外。

    在 async dpapis 上默认使用的`AsyncAdaptedQueue`应该在首次使用时实例化队列，以避免将其绑定到可能错误的事件循环。

    参考：[#5823](https://www.sqlalchemy.org/trac/ticket/5823)

+   **[asyncio]**

    SQLAlchemy 异步模式现在会检测并在使用不兼容 asyncio 的 DBAPI 时引发一个信息性错误。在异步 SQLAlchemy 中使用标准的`DBAPI`会导致其像任何同步调用一样阻塞，中断执行中的 asyncio 循环。

### postgresql

+   **[postgresql] [用例]**

    为`ExcludeConstraint`对象添加了新参数`ExcludeConstraint.ops`，以支持此约束的操作符类规范。感谢 Alon Menczer 的拉取请求。

    此更改也**回溯**到：1.3.21

    参考：[#5604](https://www.sqlalchemy.org/trac/ticket/5604)

+   **[postgresql] [用例]**

    为 asyncpg 方言的 DBAPI 适配层添加了一个读/写`.autocommit`属性。这样，当需要直接在 DBAPI 连接中使用“autocommit”时，可以使用相同的`.autocommit`属性，该属性适用于 psycopg2 和 pg8000。

+   **[postgresql] [更改]**

    修复了 psycopg2 方言在 Python 3 下悄悄传递`use_native_unicode=False`标志，实际上在 Python 3 下，psycopg2 DBAPI 无条件使用 Unicode。在 Python 3 下使用此功能现在会引发`ArgumentError`。增加了对 Python 2 的测试支持。

+   **[postgresql] [performance]**

    通过在每个连接上缓存 asyncpg PreparedStatement 对象，增强了 asyncpg 方言的性能。对于在一组池化连接上使用相同语句的测试用例，这似乎带来了 10-20%的速度提升。缓存大小可调整，也可以禁用。

    另请参阅

    预编译语句缓存

+   **[postgresql] [bug] [mysql]**

    修复了 1.3.2 中引入的 PostgreSQL 方言的回归，也在 1.3.18 中复制到了 MySQL 方言的功能，其中对于非`Table`构造的使用，例如`text()`作为`Select.with_for_update.of`参数的情况未能在 PostgreSQL 或 MySQL 编译器中正确处理。

    此更改也被**回溯**到：1.3.21

    参考：[#5729](https://www.sqlalchemy.org/trac/ticket/5729)

+   **[postgresql] [bug]**

    修复了一个小的回归，即在初始化时，“show standard_conforming_strings”查询即使检测到服务器版本信息小于 8.2，也会被发出，以前只会发生在服务器版本为 8.2 或更高版本时。该查询在 Amazon Redshift 上失败，因为它报告的 PG 服务器版本旧于此值。

    参考：[#5698](https://www.sqlalchemy.org/trac/ticket/5698)

+   **[postgresql] [bug]**

    已为`Column`对象以及 ORM 仪器化属性建立了支持，作为传递给`Insert.on_conflict_do_update()`和`Insert.on_conflict_do_update()`方法的`set_`字典中的键，这些键与目标`Table`的`.c`集合中的`Column`对象相匹配。以前，只期望字符串列名；列表达式将被假定为一个在表外的表达式，将完全呈现并附带警告。

    参考：[#5722](https://www.sqlalchemy.org/trac/ticket/5722)

+   **[postgresql] [错误] [asyncio]**

    修复了 asyncpg 方言中的错误，在“提交”期间发生故障或更少可能发生“回滚”时，应取消整个事务；不再可能发出回滚。以前，连接将继续等待无法成功的回滚，因为 asyncpg 将拒绝它。

    参考：[#5824](https://www.sqlalchemy.org/trac/ticket/5824)

### mysql

+   **[mysql] [功能]**

    在使用 asyncio SQLAlchemy 扩展时，为 aiomysql 驱动程序添加了支持。

    另请参阅

    aiomysql

    参考：[#5747](https://www.sqlalchemy.org/trac/ticket/5747)

+   **[mysql] [错误] [反射]**

    修复了在仅包含值中的十进制点的 MariaDB 上反射服务器默认值时无法正确反射的问题，导致反射表缺乏任何服务器默认值。

    此更改也已**回溯**至：1.3.21

    参考：[#5744](https://www.sqlalchemy.org/trac/ticket/5744)

### sqlite

+   **[sqlite] [用例]**

    为 SQLite 实现了 INSERT… ON CONFLICT 子句。拉取请求由 Ramon Williams 提供。

    另请参阅

    INSERT…ON CONFLICT (Upsert)

    参考：[#4010](https://www.sqlalchemy.org/trac/ticket/4010)

+   **[sqlite] [错误]**

    在 sqlite 中使用 python `re.search()` 而不是 `re.match()` 作为`Column.regexp_match()`方法使用的操作。这与其他数据库上的正则表达式的行为以及众所周知的 SQLite 插件的行为相匹配。

    参考：[#5699](https://www.sqlalchemy.org/trac/ticket/5699)

### mssql

+   **[mssql] [错误] [数据类型] [mysql]**

    当使用`Comparator.as_float()`方法从 JSON 字符串中提取浮点和/或十进制值时，当 JSON 字符串中的数值具有许多有效数字时，十进制精度和行为已得到改进；以前，MySQL 后端会截断具有许多有效数字的值，而 SQL Server 后端会由于 DECIMAL 转换具有不足有效数字而引发异常。现在，两个后端都使用了一种与 FLOAT 兼容的方法，不会为浮点值硬编码有效数字。对于精确数值，添加了一个新方法`Comparator.as_numeric()`，它接受精度和标度参数，并将值作为 Python `Decimal`对象返回，假设 DBAPI 支持它（除了 pysqlite）。

    参考：[#5788](https://www.sqlalchemy.org/trac/ticket/5788)

### oracle

+   **[oracle] [bug]**

    修复了由于[#5755](https://www.sqlalchemy.org/trac/ticket/5755)引起的回归，该回归实现了 Oracle 的隔离级别支持。据报道，许多 Oracle 账户实际上没有权限查询`v$transaction`视图，因此当在数据库连接时失败时，此功能已被修改为优雅地回退，其中方言将假定“READ COMMITTED”是默认隔离级别，就像在 SQLAlchemy 1.3.21 之前的情况一样。但是，现在必须明确使用`Connection.get_isolation_level()`方法会引发异常，因为具有此限制的 Oracle 数据库明确禁止用户读取当前隔离级别。

    此更改也已**回溯**至：1.3.22

    参考：[#5784](https://www.sqlalchemy.org/trac/ticket/5784)

+   **[oracle] [bug]**

    在基本水平上，Oracle 的两阶段事务现在不再被弃用。在获得 cx_Oracle 开发人员的支持后，我们可以提供基本的 xid + begin/prepare 支持，但存在一些限制，这将在即将发布的 cx_Oracle 版本中更全面地工作。目前不支持两阶段“恢复”。

    参考：[#5884](https://www.sqlalchemy.org/trac/ticket/5884)

+   **[oracle] [bug]**

    Oracle 方言现在使用`select sys_context( 'userenv', 'current_schema' ) from dual`来获取默认模式名称，而不是`SELECT USER FROM DUAL`，以适应 Oracle 下会话本地模式名称的更改。

    参考：[#5716](https://www.sqlalchemy.org/trac/ticket/5716)

### 测试

+   **[tests] [usecase] [pool]**

    改进文档并为亚秒级池超时添加测试。感谢 Jordan Pittier 提供的拉取请求。

    参考：[#5582](https://www.sqlalchemy.org/trac/ticket/5582)

### 杂项

+   **[usecase] [pool]**

    引擎连接程序的内部机制已更改，因此现在保证了使用`insert=True`建立的`PoolEvents.connect()`处理程序的用户定义事件处理程序将允许在任何特定于方言的初始化开始之前一定会运行的事件处理程序，特别是当它执行诸如检测默认模式名称之类的操作时。以前，在大多数情况下会发生这种情况，但不是绝对的。在模式文档中添加了一个新示例，说明如何在连接时建立“默认模式名称”。

    参考文献：[#5497](https://www.sqlalchemy.org/trac/ticket/5497)，[#5708](https://www.sqlalchemy.org/trac/ticket/5708)

+   **[bug] [reflection]**

    修复了在相关表被反射时，现在已弃用的`autoload`参数在反射例程内部被调用的错误。

    参考文献：[#5684](https://www.sqlalchemy.org/trac/ticket/5684)

+   **[bug] [pool]**

    修复了使用关键字指定的连接池事件（特别是`insert=True`）在设置事件时丢失的回归。这将阻止需要在方言级事件之前触发的启动事件的正确工作。

    参考文献：[#5708](https://www.sqlalchemy.org/trac/ticket/5708)

+   **[bug] [pool] [pypy]**

    修复了连接池在 pypy 下如果已经检出的连接超出范围而未关闭时，连接池将不会将连接返回到池中或以其他方式完成的问题。这是一个长期存在的问题，因为 pypy 的 GC 行为与其他不会调用弱引用终结器的对象相对而言不同。现在，维护与相关记录的强引用，以便弱引用具有强引用的“基础”来触发。

    参考文献：[#5842](https://www.sqlalchemy.org/trac/ticket/5842)

## 1.4.0b1

发布日期：2020 年 11 月 2 日

### 一般

+   **[general] [change]**

    “python setup.py test”不再是一个测试运行程序，因为这已被 Pypa 弃用。请使用“tox”而不带参数进行基本测试运行。

    参考文献：[#4789](https://www.sqlalchemy.org/trac/ticket/4789)

+   **[general] [bug]**

    重新设计了用于交叉导入具有彼此之间的互相依赖关系的模块的内部约定，以便不再修改函数和方法的检查参数。这允许像 pylint、Pycharm、其他代码检查器以及将来可能添加的 pep-484 实现等工具正常工作，因为它们不再看到函数调用中缺少的参数。新方法也更简单，性能更好。

    请参阅

    修复了内部导入约定，以便代码检查器可以正常工作

    参考文献：[#4656](https://www.sqlalchemy.org/trac/ticket/4656)，[#4689](https://www.sqlalchemy.org/trac/ticket/4689)

### 平台

+   **[platform] [change]**

    使用`importlib_metadata`库来扫描 setuptools 入口点，而不是 pkg_resources。由于`importlib_metadata`是 Python 3.8 及以上版本中包含的一个小型库，兼容性库会作为 Python 3.8 以下版本的依赖项安装。

    参考：[#5400](https://www.sqlalchemy.org/trac/ticket/5400)

+   **[平台] [更改]**

    安装已现代化，大部分包元数据使用 setup.cfg。

    参考：[#5404](https://www.sqlalchemy.org/trac/ticket/5404)

+   **[平台] [已移除]**

    放弃了已达到 EOL 的 python 3.4 和 3.5 的支持。SQLAlchemy 1.4 系列需要 python 2.7 或 3.6+。

    另请参阅

    Python 3.6 是最低要求的 Python 3 版本；仍支持 Python 2.7

    参考：[#5634](https://www.sqlalchemy.org/trac/ticket/5634)

+   **[平台] [已移除]**

    移除了与支持 Jython 和 zxJDBC 相关的所有方言代码。SQLAlchemy 多年来一直不支持 Jython，并且当前的 zxJDBC 代码预计根本不起作用；目前它只占用空间，并通过出现在文档中增加了混乱。目前，Jython 在其发布中已实现了对 Python 2.7 的支持，但尚未实现对 Python 3 的支持。如果再次支持 Jython，应该针对 Python 3 版本的 Jython 进行支持，并且各种后端的 zxJDBC 存根应该作为第三方方言实现。

    参考：[#5094](https://www.sqlalchemy.org/trac/ticket/5094)

### orm

+   **[orm] [功能]**

    ORM 现在可以直接使用 `select()` 构造生成以前仅在使用 `Query` 时可用的查询。ORM “插件”现在可以在 Core `Select` 中建立自己的新系统，允许以前在 `Query` 中的大部分查询构建逻辑现在在编译级扩展中进行。对于 `Update` 和 `Delete` 构造也进行了类似的更改。当使用 `Session.execute()` 调用这些构造时，现在会在方法内执行与 ORM 相关的工作。对于 `Select`，返回的 `Result` 对象现在包含 ORM 级别的实体和结果。

    另请参阅

    ORM 查询现在与 select、update、delete 内部统一；2.0 风格执行可用

    参考：[#5159](https://www.sqlalchemy.org/trac/ticket/5159)

+   **[orm] [功能]**

    增加了向查询中的关系属性生成的 ON 子句添加任意条件的功能，适用于诸如 `Query.join()` 这样的方法，以及像 `joinedload()` 这样的加载器选项。此外，该选项的“全局”版本允许将限制条件应用于查询中的特定实体。

    另请参阅

    向加载器选项添加条件

    添加全局 WHERE / ON 条件

    `with_loader_criteria()`

    参考：[#4472](https://www.sqlalchemy.org/trac/ticket/4472)

+   **[orm] [功能]**

    ORM 声明式系统现已统一到 ORM 本身，其中包括了新的导入空间，如 `sqlalchemy.orm` 和新类型的映射。现在支持基于装饰器的映射，而无需使用基类，支持具有访问声明类注册表以获取关系的经典样式 mapper() 调用，并且完全集成了声明式与第三方类属性系统，如 `dataclasses` 和 `attrs`。

    另请参阅

    声明式现已与 ORM 集成，并带有新功能

    Python 数据类、attrs 支持 w/ 声明式、命令式映射

    参考文献：[#5508](https://www.sqlalchemy.org/trac/ticket/5508)

+   **[orm] [功能]**

    当在映射器上或通过查询选项配置了急加载器（如连接加载、选择 IN 加载等）时，将在对象过期刷新时调用它们；在 selectinload 和 subqueryload 的情况下，由于额外的加载仅针对单个对象，因此在这些情况下使用了“immediateload”方案，这类似于延迟加载发出的单个父查询。

    另请参阅

    急加载器在未过期操作期间发出

    参考文献：[#1763](https://www.sqlalchemy.org/trac/ticket/1763)

+   **[orm] [功能]**

    添加了对使用 Python `dataclasses` 装饰器定义的 Python 类的直接映射的支持。感谢 Václav Klusák 提交的拉取请求。新功能集成到声明式级别的新支持中，支持诸如 `dataclasses` 和 `attrs` 等系统。

    另请参阅

    Python 数据类、attrs 支持 w/ 声明式、命令式映射

    声明式现已与 ORM 集成，并带有新功能

    参考文献：[#5027](https://www.sqlalchemy.org/trac/ticket/5027)

+   **[orm] [功能]**

    通过 `defer.raiseload` 参数在 `defer()` 和 `deferred()` 上添加了“raiseload”特性，用于 ORM 映射列。这为列表达式映射属性提供了类似于关系映射属性的 `raiseload()` 选项的行为。该变更还包括对延迟列在过期方面的一些行为变更；详细信息请参阅迁移说明。

    另请参阅

    用于列的 Raiseload

    参考文献：[#4826](https://www.sqlalchemy.org/trac/ticket/4826)

+   **[orm] [用例]**

    在 ORM 批量更新和删除的“synchronize_session=”evaluate””操作中，现在支持 IN 和 NOT IN 运算符的评估器。还支持元组 IN。

    参考文献：[#1653](https://www.sqlalchemy.org/trac/ticket/1653)

+   **[orm] [用例]**

    增强逻辑，跟踪关系是否会在写入相同列时相互冲突，包括两个关系之间应该有“backref”的简单情况。这意味着如果两个关系不是只读的，没有通过 back_populates 链接，也没有其他继承兄弟/覆盖安排，而且将填充相同的外键列，那么在映射器配置时会发出警告，警告可能会出现冲突。为了适应那些非常罕见的情况，其中这种重叠的持久性安排可能是不可避免的，添加了一个新参数`relationship.overlaps`。

    参考：[#5171](https://www.sqlalchemy.org/trac/ticket/5171)

+   **[orm] [用例]**

    ORM 批量更新和删除操作，历来通过`Query.update()`和`Query.delete()`方法以及通过`Update`和`Delete`构造进行 2.0 风格执行，现在将自动适应单表继承鉴别器所需的额外 WHERE 条件，以限制语句仅限于引用请求的特定子类型的行。新的`with_loader_criteria()`构造也支持批量更新/删除操作。

    参考：[#3903](https://www.sqlalchemy.org/trac/ticket/3903), [#5018](https://www.sqlalchemy.org/trac/ticket/5018)

+   **[orm] [用例]**

    更新`relationship.sync_backref`标志，在关系中将其隐式设置为`False`，以防止在`viewonly=True`关系中发生同步事件。

    另请参阅

    只读关系不同步 backrefs

    参考：[#5237](https://www.sqlalchemy.org/trac/ticket/5237)

+   **[orm] [更改]**

    调整了刷新具有已存在于标识映射中的标识的挂起对象的条件，以发出警告，而不是抛出`FlushError`。理由是刷新将继续并引发`IntegrityError`，就像如果现有对象尚未存在于标识映射中一样。这有助于使用`IntegrityError`作为捕获表中是否已存在行的手段的方案。

    另请参阅

    “新实例与现有标识冲突”错误现在是一个警告

    参考：[#4662](https://www.sqlalchemy.org/trac/ticket/4662)

+   **[orm] [change] [sql]**

    现在，一些核心和 ORM 查询对象在编译阶段执行更多的 Python 计算任务，而不是在构造时执行。这是为了支持即将推出的缓存模型，该模型将根据从语句构造派生的缓存键缓存编译后的语句结构，该语句构造本身预计每次在 Python 代码中使用时都会新建。这意味着这些对象的内部状态可能与以前不同，以及各种参数验证的一些但不是所有错误引发场景将在编译/执行阶段而不是在语句构造时发生。请查看下面链接的迁移说明以获取完整详细信息。

    另请参阅

    许多核心和 ORM 语句对象现在在编译阶段执行大部分构造和验证工作

+   **[orm] [change]**

    对于新的 2.0 风格的 ORM 查询，客户端端的行自动唯一性已关闭。这提高了清晰度和性能。然而，当使用连接的急切加载集合时，客户端端的行唯一性通常是必要的，因为每个集合元素都会有主实体的重复，因为使用了连接。现在必须手动启用此唯一性，并且可以使用新的`Result.unique()`修饰符来实现。为避免静默失败，ORM 在 2.0 风格的 ORM 查询结果使用连接加载集合时明确要求调用该方法。在任何情况下，较新的`selectinload()`策略可能更可取用于集合的急切加载。

    另请参阅

    ORM 行默认情况下不是唯一的

    参考：[#4395](https://www.sqlalchemy.org/trac/ticket/4395)

+   **[orm] [变更]**

    ORM 现在会在隐式地将`select()`构造转换为子查询时发出警告。这种情况发生在`Query.select_entity_from()`和`Query.select_from()`方法以及`with_polymorphic()`函数等地方。当`SelectBase`（由`select()`生成的对象）或`Query`对象直接传递给这些函数和其他函数时，ORM 通常会通过自动调用`SelectBase.alias()`方法来将它们强制转换为子查询（现在已被`SelectBase.subquery()`方法取代）。有关更多详细信息，请参阅下面链接的迁移说明。

    另请参阅

    SELECT 语句不再隐式视为 FROM 子句

    参考：[#4617](https://www.sqlalchemy.org/trac/ticket/4617)

+   **[orm] [变更]**

    `Query`返回的“KeyedTuple”类现已被 Core 的`Row`类替换，其行为与 KeyedTuple 相同。在 SQLAlchemy 2.0 中，Core 和 ORM 将使用相同的`Row`对象返回结果行。在此期间，Core 使用一个向后兼容的类`LegacyRow`，保持了以前由“RowProxy”使用的映射/元组混合行为。

    另请参阅

    Query 返回的“KeyedTuple”对象被 Row 替换

    参考：[#4710](https://www.sqlalchemy.org/trac/ticket/4710)

+   **[orm] [性能]**

    批量更新和删除方法`Query.update()`和`Query.delete()`，以及它们的 2.0 风格的对应方法，现在在使用“获取”策略时将使用 RETURNING 来获取受影响的主键标识列表，而不是在使用支持 RETURNING 的后端时发出单独的 SELECT。此外，在普通情况下，“获取”策略不会使已更新的属性过期，而是会直接应用更新的值，就像“评估”策略一样，以避免刷新对象。如果“评估”策略也会回退到使 Python 中无法评估的 SQL 表达式已更新的属性过期。

    另请参阅

    ORM 批量更新和删除在可用时使用 RETURNING 进行“获取”策略

+   **[orm] [performance] [postgresql]**

    实现了对 psycopg2 `execute_values()` 扩展的支持，通过对 Core 的增强在[#5401](https://www.sqlalchemy.org/trac/ticket/5401)中进行了 ORM 刷新过程，因此该扩展被用作批量提交 INSERT 语句的策略，同时 RETURNING 现在可以在多个参数集之间使用，以批量检索主键值。这使得 ORM 代表 PostgreSQL 发出的几乎所有 INSERT 语句都可以批量提交，而且还可以通过`execute_values()`扩展进行提交，对于这个特定的后端，这种扩展的性能比普通的 executemany()快五倍。

    另请参阅

    ORM 使用 psycopg2 批量插入现在在大多数情况下使用 RETURNING 批量语句

    参考：[#5263](https://www.sqlalchemy.org/trac/ticket/5263)

+   **[orm] [bug]**

    对于针对映射的继承子类的查询，还使用`Query.select_entity_from()`或类似技术以提供现有子查询以从中选择的情况，如果给定的子查询返回与给定子类不对应的实体，即它们是同一层次结构中的兄弟或超类，则现在将引发错误。以前，这些将无错误返回。此外，如果继承映射是单一继承映射，则给定的子查询必须针对多态鉴别器列应用适当的过滤以避免此错误；以前，`Query`会将此条件添加到外部查询中，但这会干扰一些返回其他类型实体的查询。

    另请参阅

    使用自定义查询查询继承映射时更严格的行为

    参考：[#5122](https://www.sqlalchemy.org/trac/ticket/5122)

+   **[orm] [bug]**

    内部属性符号 NO_VALUE 和 NEVER_SET 已统一，因为这两个符号之间没有实质性的区别，除了在一些代码路径中它们以微妙且未记录的方式进行区分之外，这些已经被修复。

    参考：[#4696](https://www.sqlalchemy.org/trac/ticket/4696)

+   **[orm] [bug]**

    修复了一个 bug，该 bug 在对映射器指定了一个版本列，并且该版本列针对的是底层表的情况下，在访问时会产生额外的加载，即使该值已经通过 flush 进行了本地持久化。实际的修复结果是由 [#4617](https://www.sqlalchemy.org/trac/ticket/4617) 中的更改导致的，由于 `select()` 对象不再具有 `.c` 属性，因此不会再让映射器误以为存在未知的列值而混淆。

    参考：[#4194](https://www.sqlalchemy.org/trac/ticket/4194)

+   **[orm] [bug]**

    如果实例是一个未映射的对象，则现在会为 `InstrumentedAttribute` 抛出 `UnmappedInstanceError`。在此之前，会抛出 `AttributeError`。拉取请求由 Ramon Williams 提供。

    参考：[#3858](https://www.sqlalchemy.org/trac/ticket/3858)

+   **[orm] [bug]**

    `Session`对象在构造后或上一个事务关闭后不再立即初始化`SessionTransaction`对象；相反，“autobegin”逻辑现在在下次需要时按需初始化新的`SessionTransaction`。理由包括从已关闭的`Session`中删除引用循环，以及消除由创建经常立即丢弃的`SessionTransaction`对象所产生的开销。此更改影响`SessionEvents.after_transaction_create()`钩子的行为，即当`Session`首次需要存在`SessionTransaction`时，事件将被触发，而不是在创建`Session`或关闭先前的`SessionTransaction`时。与`Engine`和数据库本身的交互保持不变。

    另请参阅

    Session features new “autobegin” behavior

    参考：[#5074](https://www.sqlalchemy.org/trac/ticket/5074)

+   **[orm] [bug]**

    在 ORM 查询上下文中添加了新的实体定位功能，用于处理当`Session`使用绑定字典针对映射类，而不是单个绑定时的情况，并且`Query`针对的是最终生成自诸如`Query.subquery()`等方法的 Core 语句。首先使用深度搜索实现，当前方法利用统一的`select()`构造来跟踪构造中的第一个映射器。

    参考：[#4829](https://www.sqlalchemy.org/trac/ticket/4829)

+   **[orm] [bug] [inheritance]**

    如果在 `with_polymorphic()` 中同时设置了 `selectable` 和 `flat` 参数为 True，则现在会引发 `ArgumentError`。selectable 名称已经被别名化，并且应用 flat=True 会使用匿名名称覆盖 selectable 名称，这以前会导致代码中断。拉取请求由 Ramon Williams 提供。

    参考：[#4212](https://www.sqlalchemy.org/trac/ticket/4212)

+   **[orm] [bug]**

    修复了多态加载内部的问题，当在某些未过期的情况下回退到一种更昂贵的、即将过时的结果列查找形式，并与“with_polymorphic”一起使用时，会出现这种情况。

    参考：[#4718](https://www.sqlalchemy.org/trac/ticket/4718)

+   **[orm] [bug]**

    如果在 `relationship()` 上设置了任何与持久性相关的“级联”设置，同时也设置了 viewonly=True，则会引发错误。当 viewonly 也设置时，“级联”设置现在默认为非持久性相关设置。这是从[#4993](https://www.sqlalchemy.org/trac/ticket/4993)继续的，该设置在 1.3 中被更改为发出警告。

    另见

    持久性相关的级联操作在 viewonly=True 时不允许

    参考：[#4994](https://www.sqlalchemy.org/trac/ticket/4994)

+   **[orm] [bug]**

    改进了声明式继承扫描，以避免当同一个基类多次出现在基础继承列表中时出错。

    参考：[#4699](https://www.sqlalchemy.org/trac/ticket/4699)

+   **[orm] [bug]**

    修复了 ORM 版本化功能中的错误，在映射选择器配置为对映射的可选择器进行版本化计数时，如果之前的值已过期，则会失败；这是由于映射属性未配置为 active_history=True。

    参考：[#4195](https://www.sqlalchemy.org/trac/ticket/4195)

+   **[orm] [bug]**

    如果 ORM 加载了一个具有主键但鉴别器列为 NULL 的多态实例的行，则现在会引发异常，因为鉴别器列不应为 null。

    参考：[#4836](https://www.sqlalchemy.org/trac/ticket/4836)

+   **[orm] [bug]**

    在新创建的对象上访问面向集合的属性不再改变`__dict__`，但仍然返回一个空集合，就像一直以来一样。这使得集合属性可以与标量属性一致地工作，后者返回`None`，但也不会改变`__dict__`。为了适应集合被改变的情况，一旦初始创建，每次都会返回相同的空集合，并且当它被改变（例如添加了一个项目）时，它将被移动到`__dict__`中。这消除了 ORM 中只读属性访问中的最后一个改变副作用。

    另请参阅

    在瞬态对象上访问未初始化的集合属性不再改变 __dict__

    参考：[#4519](https://www.sqlalchemy.org/trac/ticket/4519)

+   **[orm] [bug]**

    刷新过期对象现在将触发自动刷新，如果过期属性列表包括一个或多个属性，这些属性是通过`Session.expire()`或`Session.refresh()`方法明确过期或刷新的。这是为了在许多情况下不希望发生自动刷新的情况下找到一种折中，与属性被明确过期或刷新并且这些属性依赖于会话中需要刷新的其他挂起状态的情况相对应。这两种方法现在还增加了一个新标志`Session.expire.autoflush`和`Session.refresh.autoflush`，默认为 True；当设置为 False 时，这将禁用对这些属性取消过期时发生的自动刷新。

    参考：[#5226](https://www.sqlalchemy.org/trac/ticket/5226)

+   **[orm] [bug]**

    `relationship.cascade_backrefs`标志的行为将在 2.0 中被反转，并无条件地设置为`False`，以使反向引用不会从前向赋值到后向赋值中级联保存更新操作。当参数保持默认值`True`时，如果发生这样的级联操作，将发出 2.0 弃用警告。可以通过在特定的`relationship()`上将标志设置为`False`来始终建立新行为，或者更一般地可以通过将`Session.future`标志设置为 True 来全面设置。

    另请参阅

    2.0 中将删除的 cascade_backrefs 行为已弃用

    参考：[#5150](https://www.sqlalchemy.org/trac/ticket/5150)

+   **[orm] [已弃用]**

    在 SQLAlchemy 2.0 中，`Query`以及动态关系加载器使用的“切片索引”功能将不再接受负索引。这些操作效率低下，并且加载整个集合，这既令人惊讶又不可取。在 1.4 中，除非设置了`Session.future`标志，否则将发出警告，否则将引发 IndexError。

    参考：[#5606](https://www.sqlalchemy.org/trac/ticket/5606)

+   **[orm] [已弃用]**

    调用不传递`QueryContext`的`Query.instances()`方法已被弃用。最初的用例是，当只给出要选择的实体以及一个 DBAPI 游标对象时，`Query`可以产生 ORM 对象。但是，为了使其正常工作，必须从源自映射列表达式的 SQLAlchemy `ResultProxy`传递基本元数据，这最初来自`QueryContext`。要从任意 SELECT 语句中检索 ORM 结果，应使用`Query.from_statement()`方法。

    参考：[#4719](https://www.sqlalchemy.org/trac/ticket/4719)

+   **[orm] [已弃用]**

    在 ORM 操作中使用字符串表示关系名称（例如`Query.join()`）以及在加载器选项中使用字符串表示所有 ORM 属性名称（例如`selectinload()`）已被弃用，并将在 SQLAlchemy 2.0 中移除。应传递类绑定属性。这为给定方法提供了更好的特异性，允许使用`of_type()`等修饰符，并减少了内部复杂性。

    此外，`Query.join()`方法中的`aliased`和`from_joinpoint`参数也已被弃用。现在，`aliased()`构造提供了很大的灵活性和功能，应直接使用。

    另请参阅

    ORM 查询 - 在关系上加入/加载使用属性，而不是字符串

    ORM 查询 - join(…, aliased=True)，已移除 from_joinpoint

    参考：[#4705](https://www.sqlalchemy.org/trac/ticket/4705)、[#5202](https://www.sqlalchemy.org/trac/ticket/5202)

+   **[orm] [不推荐使用]**

    在 `Query.distinct()` 中已经弃用的逻辑自动将 ORDER BY 子句中的列添加到列子句中；这将在 2.0 版本中移除。

    另见

    使用 DISTINCT 选择额外列，但仅选择实体

    参考：[#5134](https://www.sqlalchemy.org/trac/ticket/5134)

+   **[orm] [不推荐使用]**

    将关键字参数传递给诸如 `Session.execute()` 的方法，并传递给 `Session.get_bind()` 方法已经弃用；应该传递新的 `Session.execute.bind_arguments` 字典。

    参考：[#5573](https://www.sqlalchemy.org/trac/ticket/5573)

+   **[orm] [不推荐使用]**

    `eagerload()` 和 `relation()` 是旧别名，现在已弃用。分别使用 `joinedload()` 和 `relationship()`。

    参考：[#5192](https://www.sqlalchemy.org/trac/ticket/5192)

+   **[orm] [已移除]**

    所有长期弃用的“扩展”类都已移除，包括 MapperExtension、SessionExtension、PoolListener、ConnectionProxy、AttributeExtension。这些类自版本 0.7 起已弃用，长期以来被事件监听器系统取代。

    参考：[#4638](https://www.sqlalchemy.org/trac/ticket/4638)

+   **[orm] [已移除]**

    移除了不推荐使用的加载器选项 `joinedload_all`、`subqueryload_all`、`lazyload_all`、`selectinload_all`。应该使用正常版本的方法链代替它们。

    参考：[#4642](https://www.sqlalchemy.org/trac/ticket/4642)

+   **[orm] [已移除]**

    移除了弃用的函数 `comparable_property`。请参阅 `hybrid` 扩展。这还移除了声明式扩展中的函数 `comparable_using`。

    移除了弃用的函数 `compile_mappers`。请使用 `configure_mappers()`

    移除已弃用的方法`collection.linker`。请参考`AttributeEvents.init_collection()`和`AttributeEvents.dispose_collection()`事件处理程序。

    移除已弃用的方法`Session.prune`和参数`Session.weak_identity_map`。请参考 Session Referencing Behavior 中的示例，以事件驱动的方式维护强引用关系。此更改还移除了类`StrongInstanceDict`。

    移除已弃用的参数`mapper.order_by`。使用`Query.order_by()`确定结果集的排序方式。

    移除已弃用的���数`Session._enable_transaction_accounting`。

    移除已弃用的参数`Session.is_modified.passive`。

    参考：[#4643](https://www.sqlalchemy.org/trac/ticket/4643)

### engine

+   **[engine] [feature]**

    实现了全新的`Result`对象，取代了以前的`ResultProxy`对象。作为 Core 中的子类，`CursorResult`具有与以前的`ResultProxy`兼容的调用接口，并且还添加了大量新功能，可应用于 Core 结果集以及现在集成到同一模型中的 ORM 结果集。`Result`包括列选择和重新排列、改进的 fetchmany 模式、唯一性等功能，以及各种实现，可用于从内存结构创建数据库结果。

    另请参阅

    新 Result 对象

    参考：[#4395](https://www.sqlalchemy.org/trac/ticket/4395), [#4959](https://www.sqlalchemy.org/trac/ticket/4959), [#5087](https://www.sqlalchemy.org/trac/ticket/5087)

+   **[engine] [feature] [orm]**

    SQLAlchemy 现在在 Core 和 ORM 中都包含对 Python asyncio 的支持，使用包含的 asyncio 扩展。该扩展利用了[greenlet](https://greenlet.readthedocs.io/en/latest/)库，以调整 SQLAlchemy 的同步导向内部，从而使得与 asyncio 数据库适配器交互的 asyncio 接口现在可行。目前仅支持的单个驱动程序是用于 PostgreSQL 的 asyncpg 驱动程序。

    另请参阅

    Core 和 ORM 的异步 IO 支持

    参考：[#3414](https://www.sqlalchemy.org/trac/ticket/3414)

+   **[引擎] [特性] [alchemy2]**

    实现了`create_engine.future`参数，它可以与 SQLAlchemy 2\.的向前兼容性一起使用。该引擎始终具有自动开始的事务行为。

    另请参阅

    SQLAlchemy 2.0 - 主要迁移指南

    参考：[#4644](https://www.sqlalchemy.org/trac/ticket/4644)

+   **[引擎] [特性] [pyodbc]**

    重新设计了“setinputsizes()”一组方言钩子，以便对任意任意的 DBAPI 进行正确的扩展，通过允许方言具有单独的钩子，这些钩子可以以适合该 DBAPI 的方式调用 cursor.setinputsizes()。特别是这是为了支持 pyodbc 的使用方式，这种方式与 cx_Oracle 的基本不同。增加了对 pyodbc 的支持。

    参考：[#5649](https://www.sqlalchemy.org/trac/ticket/5649)

+   **[引擎] [特性]**

    添加了新的反射方法`Inspector.get_sequence_names()`，它返回所有定义的序列，...`Inspector.has_sequence()`

+   **[引擎] [特性]**

    `Table.autoload_with`参数现在直接接受一个`Inspector`对象，以及任何`Engine`或`Connection`，就像以前一样。

    参考：[#4755](https://www.sqlalchemy.org/trac/ticket/4755)

+   **[引擎] [变更]**

    `RowProxy` 类不再是“代理”对象，而是在构造时直接填充了经过后处理的 DBAPI 行元组的内容。现在命名为 `Row`，Python 级别的值处理器的机制已经简化，特别是对 C 代码格式的影响，以便将 DBAPI 行处理为结果元组。`ResultProxy` 返回的对象现在是 `LegacyRow` 子类，保持映射/元组混合行为，但是基本的 `Row` 类现在更像一个命名元组。

    另请参阅

    RowProxy 不再是“代理”；现在称为 Row 并且行为类似增强的命名元组

    参考：[#4710](https://www.sqlalchemy.org/trac/ticket/4710)

+   **[引擎] [性能]**

    池“预连接”功能已经优化，不会对刚刚在同一检出操作中打开的 DBAPI 连接进行调用。预连接仅适用于已经检入池中并且再次被检出的 DBAPI 连接。

    参考：[#4524](https://www.sqlalchemy.org/trac/ticket/4524)

+   **[引擎] [性能] [更改] [py3k]**

    在 Python 3 下运行时禁用了在方言启动时运行的“unicode 返回”检查，多年来一直用于测试当前 DBAPI 是否为 VARCHAR 和 NVARCHAR 数据类型返回 Python Unicode 或 Py2K 字符串的行为。在 Python 2 下仍然默认进行检查，但是当 SQLAlchemy 2.0 移除对 Python 2 的支持时，测试行为的机制也将被移除。

    当需要时，这种逻辑非常有效，但是现在 Python 3 已经成为标准，所有的 DBAPI 都应该为字符数据类型返回 Python 3 字符串。在第三方 DBAPI 不支持这一点的极少情况下，`String` 中的转换逻辑仍然可用，并且第三方方言可以通过在其初始方言标志中设置方言级别标志 `returns_unicode_strings` 为 `String.RETURNS_CONDITIONAL` 或 `String.RETURNS_BYTES` 中的一个来启用 Unicode 转换，即使在 Python 3 下也会启用。

    参考：[#5315](https://www.sqlalchemy.org/trac/ticket/5315)

+   **[引擎] [错误]**

    重新设计了 `Connection.execution_options.schema_translate_map` 功能，以便在语句的执行阶段而不是编译阶段对 SQL 语句进行处理，以支持语句的高效缓存。以前，为了一个运行对应数百个模式而对语句进行渲染的当前模式被视为缓存键的一部分，这意味着对于运行针对数百个模式的情况，将会有数百个缓存键，使得缓存性能大大降低。新的行为是在执行过程中类似于 1.4 中添加的“后编译”渲染 [#4645](https://www.sqlalchemy.org/trac/ticket/4645), [#4808](https://www.sqlalchemy.org/trac/ticket/4808)。

    参考：[#5004](https://www.sqlalchemy.org/trac/ticket/5004)

+   **[引擎] [错误]**

    `Connection` 对象现在不会在外层事务明确回滚之前清除已回滚的事务。这实质上与 ORM `Session` 长期以来的行为相同，需要在所有封闭事务上显式调用 `.rollback()`，以便事务在逻辑上清除，即使 DBAPI 级别的事务已经被回滚。新的行为有助于处理诸如“ORM 回滚测试套件”模式之类的情况，在这种模式中，测试套件在 ORM 范围内回滚事务，但试图在外部控制事务范围的测试工具不希望隐式开始新事务。

    参见

    基于子事务，现在可以使连接级事务处于非活动状态

    参考：[#4712](https://www.sqlalchemy.org/trac/ticket/4712)

+   **[引擎] [错误]**

    调整了方言初始化过程，使得在第一个连接上不会第二次调用 `Dialect.on_connect()`。如果该连接是该方言的第一个连接，则首先调用钩子，然后调用 `Dialect.initialize()`，然后不再调用更多事件。这消除了对“on_connect”函数的两次调用，这可能会产生非常困难的调试情况。

    参考：[#5497](https://www.sqlalchemy.org/trac/ticket/5497)

+   **[引擎] [已弃用]**

    `URL` 对象现在是一个不可变命名元组。要修改 URL 对象，请使用 `URL.set()` 方法生成一个新的 URL 对象。

    另见

    URL 对象现在是不可变的 - 迁移说明

    参考：[#5526](https://www.sqlalchemy.org/trac/ticket/5526)

+   **[engine] [弃用]**

    `MetaData.bind` 参数以及“绑定元数据”的整体概念在 SQLAlchemy 1.4 中已被弃用，并将在 SQLAlchemy 2.0 中移除。当使用 SQLAlchemy 2.0 弃用模式 时，该参数以及相关函数现在会发出 `RemovedIn20Warning`。

    另见

    “隐式”和“无连接”执行，“绑定元数据”已移除

    参考：[#4634](https://www.sqlalchemy.org/trac/ticket/4634)

+   **[engine] [弃用]**

    `server_side_cursors` 引擎级参数已被弃用，并将在将来的版本中移除。对于无缓冲游标，应在每次执行时使用 `Connection.execution_options.stream_results` 执行选项。

+   **[engine] [弃用]**

    `Connection.connect()` 方法已被弃用，以及“连接分支”的概念也是如此，它将 `Connection` 复制到一个新的连接中，该连接具有一个无操作的“.close()”方法。这种模式围绕着“无连接执行”概念，这个概念在 2.0 版本中也被移除。

    参考：[#5131](https://www.sqlalchemy.org/trac/ticket/5131)

+   **[engine] [弃用]**

    在 `create_engine()` 上的 `case_sensitive` 标志已被弃用；该标志是过渡结果行对象以允许默认情况下进行区分大小写列匹配的一部分，同时为以前的匹配方法提供向后兼容性的一部分。应该假定对行的所有字符串访问都是区分大小写的，就像任何其他 Python 映射一样。

    参考：[#4878](https://www.sqlalchemy.org/trac/ticket/4878)

+   **[engine] [弃用]**

    “隐式自动提交”是指在连接上发出 DML 或 DDL 语句时发生的 COMMIT，已被弃用，并且不会成为 SQLAlchemy 2.0 的一部分。当自动提交生效时会发出 2.0 风格的警告，以便调用代码可以调整为使用显式事务。

    作为此更改的一部分，当针对`Engine`使用诸如`MetaData.create_all()`之类的 DDL 方法时，如果尚未启动 BEGIN 块，则会运行该操作。

    另请参阅

    SQLAlchemy 2.0 弃用模式

    参考：[#4846](https://www.sqlalchemy.org/trac/ticket/4846)

+   **[engine] [已弃用]**

    已弃用了`Column`作为结果集行查找中的键的行为，当该`Column`不是被选择的 SQL 可选部分时；也就是说，它只是按名称匹配。现在针对这种情况会发出弃用警告。已改进了各种 ORM 使用情况，例如涉及`text()`构造的情况，以避免在大多数情况下使用此回退逻辑。

    参考：[#4877](https://www.sqlalchemy.org/trac/ticket/4877)

+   **[engine] [已弃用]**

    已弃用剩余的引擎级内省和实用方法，包括`Engine.run_callable()`、`Engine.transaction()`、`Engine.table_names()`、`Engine.has_table()`。这些实用方法已被现代上下文管理器模式取代，表内省任务适用于`Inspector`对象。

    参考：[#4755](https://www.sqlalchemy.org/trac/ticket/4755)

+   **[engine] [已移除]**

    移除了`Dialect`和`Inspector`类中已弃用的方法`get_primary_keys`。请参考`Dialect.get_pk_constraint()`和`Inspector.get_primary_keys()`方法。

    移除了已弃用的事件`dbapi_error`和方法`ConnectionEvents.dbapi_error`。请参考`ConnectionEvents.handle_error()`事件。此更改还移除了属性`ExecutionContext.is_disconnect`和`ExecutionContext.exception`。

    参考：[#4643](https://www.sqlalchemy.org/trac/ticket/4643)

+   **[engine] [已移除]**

    内部方言方法`Dialect.reflecttable`已移除。对第三方方言的审查未发现任何使用此方法的情况，因为它已经被记录为不应该被外部方言使用的方法。此外，私有方法`Engine._run_visitor`也已移除。

    参考：[#4755](https://www.sqlalchemy.org/trac/ticket/4755)

+   **[engine] [已移除]**

    已删除了长期弃用的 `Inspector.get_table_names.order_by` 参数。

    参考：[#4755](https://www.sqlalchemy.org/trac/ticket/4755)

+   **[engine] [renamed]**

    `Inspector.reflecttable()` 被重命名为 `Inspector.reflect_table()`。

    参考：[#5244](https://www.sqlalchemy.org/trac/ticket/5244)

### sql

+   **[sql] [feature]**

    将“from linting”作为 SQL 编译器的内置功能添加。这使得编译器能够维护特定 SELECT 语句中所有 FROM 子句的图形，这些 FROM 子句通过 WHERE 或 JOIN 子句中的条件相连。如果任何两个 FROM 子句之间没有路径，将发出警告，表明查询可能会产生笛卡尔积。由于 Core 表达语言以及 ORM 建立在“隐式 FROMs”模型上，在该模型中，如果查询的任何部分引用了它，特定的 FROM 子句将自动添加，因此可能会无意中发生这种情况，并希望新特性能够解决这个问题。

    请参见

    内置的 FROM linting 将警告任何 SELECT 语句中的潜在笛卡尔积

    参考：[#4737](https://www.sqlalchemy.org/trac/ticket/4737)

+   **[sql] [feature] [mssql] [oracle]**

    增加了新的“编译后参数”特性。该特性允许 `bindparam()` 构造在传递给 DBAPI 驱动程序之前，但在编译步骤之后，将其值呈现到 SQL 字符串中，使用编译器的“字面呈现”特性。该特性的即时原因是支持不适用或不好地由数据库驱动程序处理的 LIMIT/OFFSET 方案，同时仍允许 SQLAlchemy SQL 构造以编译形式进行缓存。新特性的即时目标是 SQL Server（和 Sybase）使用的“TOP N”子句，它不支持绑定参数，以及 Oracle 方言使用的“ROWNUM”和可选的“FIRST_ROWS()”方案，前者已知在没有绑定参数时表现更好，后者不支持绑定参数。该特性建立在首次开发用于支持 IN 表达式的“扩展”参数的机制之上。作为该特性的一部分，Oracle 的 `use_binds_for_limits` 特性被无条件地打开，此标志现在已被弃用。

    请参见

    Oracle、SQL Server 中用于 LIMIT/OFFSET 的新“编译后”绑定参数

    参考：[#4808](https://www.sqlalchemy.org/trac/ticket/4808)

+   **[sql] [feature]**

    增加对支持的后端正则表达式的支持。定义了两种操作：

    +   `ColumnOperators.regexp_match()` 实现了类似正则表达式匹配的功能。

    +   `ColumnOperators.regexp_replace()` 实现了正则表达式字符串替换功能。

    支持的后端包括 SQLite、PostgreSQL、MySQL/MariaDB 和 Oracle。

    另请参阅

    支持 SQL 正则表达式操作符

    参考：[#1390](https://www.sqlalchemy.org/trac/ticket/1390)

+   **[sql] [feature]**

    `select()` 构造及相关构造现在允许在列子句中重复列标签和列本身，完全反映了列表达式的传递方式。这允许执行结果返回的元组与首次 SELECT 中选择的内容相匹配，这是 ORM `Query` 的工作原理，因此在这两个构造之间建立了更好的交叉兼容性。此外，它允许对列位置敏感的结构（例如 UNION，即 `_selectable.CompoundSelect`）在特定情况下更直观地构建，在这些情况下，特定列可能会出现在多个位置。为了支持此更改，已经修订了 `ColumnCollection` 以支持重复列，并允许整数索引访问。

    另请参阅

    SELECT 对象和派生的 FROM 子句允许重复的列和列标签

    参考：[#4753](https://www.sqlalchemy.org/trac/ticket/4753)

+   **[sql] [feature]**

    加强了 `select()` 构造的消除歧义标签功能，使得当选择语句用于子查询时，来自不同表的重复列名现在会自动使用唯一的标签名称进行标记，而不需要使用完整的“apply_labels()”功能来合并表名加列名。消除歧义的标签可以作为子查询的 .c 集合中的普通字符串键使用，最重要的是，该功能允许针对实体和任意子查询的组合使用 ORM `aliased()` 构造正确地工作，尽管源表中存在同名列，也可以正确地定位到正确的列，而不需要使用“应用标签”警告。

    另请参阅

    从查询本身作为子查询中选择，例如“from_self()” - 说明了作为迁移策略的一部分的新消歧功能，以摆脱`Query.from_self()`方法。

    参考：[#5221](https://www.sqlalchemy.org/trac/ticket/5221)

+   **[sql] [特性]**

    “扩展 IN”功能在查询执行时生成基于语句执行相关参数的 IN 表达式，现在用于针对字面值列表进行的所有 IN 表达式。这使得 IN 表达式可以完全可缓存，而不受传递的值列表的影响，并且还包括对空列表的支持。对于包含非字面 SQL 表达式的 IN 表达式的任何情况，保留了每个位置预渲染的旧行为。该更改还完成了对使用元组扩展 IN 的支持，以前类型特定的绑定处理器未生效。

    另请参阅

    所有 IN 表达式都会在执行时为列表中的每个值渲染参数（例如，扩展参数）

    参考：[#4645](https://www.sqlalchemy.org/trac/ticket/4645)

+   **[sql] [特性]**

    除了作为[#4369](https://www.sqlalchemy.org/trac/ticket/4369)的一部分引入的新透明语句缓存功能之外，还添加了一个新功能，旨在减少创建语句时的 Python 开销，允许在指示传递给 select()、Query()、update()等语句对象的参数时使用 Lambda，并允许在 Lambda 中构建完整的语句，类似于“烘焙查询”系统的方式。使用 Lambda 的理念源自“烘焙查询”方法，该方法使用 Lambda 将任意数量的 Python 代码封装为可调用对象，只需在首次构建语句为字符串时调用即可。然而，新功能更为复杂，因为会自动提取作为参数传递的 Python 字面值，因此不再需要在此类查询中使用 bindparam()对象。使用该功能是可选的，并且可以根据需要使用到任何程度，同时仍然允许语句完全可缓存。

    另请参阅

    使用 Lambda 为语句生成带来显著速度提升

    参考：[#5380](https://www.sqlalchemy.org/trac/ticket/5380)

+   **[sql] [用例]**

    `Index.create()`和`Index.drop()`方法现在具有一个参数`Index.create.checkfirst`，与`Table`和`Sequence`的参数相同，当启用时，操作将在执行创建或删除操作之前检测索引是否存在（或不存在）。

    参考：[#527](https://www.sqlalchemy.org/trac/ticket/527)

+   **[sql] [用例]**

    `true()`和`false()`运算符现在可以作为不支持“本地布尔”表达式的后端（例如 Oracle 或 SQL Server）上的`join()`的“onclause”应用，表达式将渲染为“1=1”表示 true 和“1=0”表示 false。这是多年前在[#2804](https://www.sqlalchemy.org/trac/ticket/2804)中引入的 and/or 表达式的行为。

+   **[sql] [用例]**

    修改`ColumnCollection`的`__str`方法，避免将其与 Python 字符串列表混淆。

    参考：[#5191](https://www.sqlalchemy.org/trac/ticket/5191)

+   **[sql] [用例]**

    在支持的后端（目前为 PostgreSQL、Oracle 和 MSSQL）中为选择添加对`FETCH {FIRST | NEXT} [ count ] {ROW | ROWS} {ONLY | WITH TIES}`的支持。

    参考：[#5576](https://www.sqlalchemy.org/trac/ticket/5576)

+   **[sql] [用例]**

    添加了额外的逻辑，使得某些通常包装单个数据库列的 SQL 表达式将该列的名称用作其在 SELECT 语句中的“匿名标签”名称，这可能使得在结果元组中进行基于键的查找更直观。这个主要的例子是 CAST 表达式，例如`CAST(table.colname AS INTEGER)`，它将其默认名称导出为“colname”，而不是通常的“anon_1”标签，即`CAST(table.colname AS INTEGER) AS colname`。如果内部表达式没有名称，则使用先前的“匿名标签”逻辑。当使用`Select.apply_labels()`生成的 SELECT 语句时，例如 ORM 发出的语句，标签逻辑将产生与单独命名列相同的`<tablename>_<inner column name>`。这个逻辑现在适用于`cast()`和`type_coerce()`结构，以及一些单元素布尔表达式。

    另见

    改进了使用 CAST 或类似方法的简单列表达式的列标签

    参考：[#4449](https://www.sqlalchemy.org/trac/ticket/4449)

+   **[sql] [change]**

    “子句强制转换”系统，即 SQLAlchemy Core 接收参数并将其解析为`ClauseElement`结构以构建 SQL 表达式对象的系统，已从一系列临时函数重写为完全一致的基于类的系统。这个变化是内部的，除了在将错误类型的参数传递给表达式对象时提供更具体的错误消息外，不应该对最终用户产生任何影响，但是这个变化是一系列涉及`select()`对象的角色和行为的变化的一部分。

    参考：[#4617](https://www.sqlalchemy.org/trac/ticket/4617)

+   **[sql] [change]**

    添加了核心`Values`对象，使得在支持的数据库（主要是 PostgreSQL 和 SQL Server）的 SQL 语句的 FROM 子句中可以使用 VALUES 构造。

    参考：[#4868](https://www.sqlalchemy.org/trac/ticket/4868)

+   **[sql] [change]**

    `select()`构造正朝着一个新的调用形式发展，即`select(col1, col2, col3, ..)`，所有其他关键字参数被移除，因为这些都适用于生成方法。仍然接受传递给`select()`的列或表的参数的单个列表，但是如果表达式以简单的位置样式传递，则不再需要。当使用这种形式时，不允许使用其他关键字参数。

    另见

    select()，case()现在接受位置表达式

    参考：[#5284](https://www.sqlalchemy.org/trac/ticket/5284)

+   **[sql] [change]**

    作为 SQLAlchemy 2.0 迁移项目的一部分，对`SelectBase`类层次结构的角色进行了概念性更改，它是所有“SELECT”语句构造的根源，不再直接作为 FROM 子句，也就是说，它们不再是`FromClause`的子类。对于最终用户，这个变化主要意味着任何将`select()`构造放置在另一个`select()`的 FROM 子句中都需要首先将其包装在子查询中，这在历史上通常是通过`SelectBase.alias()`方法实现的，现在也可以通过`SelectBase.subquery()`方法实现。这通常是一个要求，因为一些数据库在任何情况下都不接受未命名的 SELECT 子查询作为其 FROM 子句的情况。

    另请参阅

    SELECT 语句不再隐式视为 FROM 子句

    参考：[#4617](https://www.sqlalchemy.org/trac/ticket/4617)

+   **[sql] [change]**

    添加了一个新的 Core 类`Subquery`，用于在`SelectBase`对象上创建命名子查询时取代`Alias`。`Subquery`与`Alias`的作用方式相同，并且是通过`SelectBase.subquery()`方法生成的；为了方便使用和向后兼容性，`SelectBase.alias()`方法与这个新方法是同义的。

    另请参阅

    SELECT 语句不再隐式视为 FROM 子句

    参考：[#4617](https://www.sqlalchemy.org/trac/ticket/4617)

+   **[sql] [performance]**

    对 Core 和 ORM 内部的全面重组和重构现在允许 DQL（例如 SELECTs）和 DML（例如 INSERT、UPDATE、DELETE）领域内的所有 Core 和 ORM 语句允许它们的 SQL 编译以及结果获取元数据的构造在大多数情况下完全缓存。这实际上提供了一个透明和通用化的版本，类似于过去版本中 ORM 提供的“Baked Query”扩展。新功能可以根据最终为给定方言生成的字符串计算任何给定 SQL 构造的缓存键，允许每次生成等效 select()、Query()、insert()、update()或 delete()对象的函数在第一次生成后缓存该语句。

    该功能是透明启用的，但包括一些新的编程范式，可以用来使缓存更加高效。

    另请参阅

    透明 SQL 编译缓存添加到 Core、ORM 中的所有 DQL、DML 语句

    SQL 编译缓存

    参考：[#4639](https://www.sqlalchemy.org/trac/ticket/4639)

+   **[sql] [bug]**

    修复了从 ORM 绑定列构造约束时的问题，主要是`ForeignKey`对象，但也包括`UniqueConstraint`、`CheckConstraint`等，ORM 级别的`InstrumentedAttribute`被完全丢弃，所有列的 ORM 级别注释都被删除；这样做是为了使约束仍然完全可 pickleable，而不引入 ORM 级别的实体。这些注释不需要出现在模式/元数据级别。

    参考：[#5001](https://www.sqlalchemy.org/trac/ticket/5001)

+   **[sql] [bug]**

    基于`GenericFunction`注册的函数名称现在在所有情况下以不区分大小写的方式检索，从 1.3 中删除了临时允许存在具有不同大小写的多个`GenericFunction`对象的弃用逻辑。一个替换另一个相同名称的`GenericFunction`，无论是否区分大小写，在替换对象之前发出警告。

    参考：[#4569](https://www.sqlalchemy.org/trac/ticket/4569), [#4649](https://www.sqlalchemy.org/trac/ticket/4649)

+   **[sql] [bug]**

    创建一个没有参数或空`*args`的`and_()`或`or_()`结构现在会发出弃用警告，因为生成的 SQL 是一个空操作（即呈现为空字符串）。这种行为被认为是不直观的，因此对于空或可能为空的`and_()`或`or_()`结构，应包含适当的默认布尔值，例如`and_(True, *args)`或`or_(False, *args)`。就像在许多主要版本的 SQLAlchemy 中一样，如果`*args`部分不为空，这些特定的布尔值将不会呈现。

    参考：[#5054](https://www.sqlalchemy.org/trac/ticket/5054)

+   **[sql] [bug]**

    改进了`tuple_()`结构，使其在列子句上下文中的行为更加可预测。大多数后端不支持 SQL 元组作为“SELECT”列子句元素；在支持的后端（毫不奇怪的是 PostgreSQL）上，Python DBAPI 没有“嵌套类型”概念，因此在为这样的对象获取行时仍然存在挑战。在`select()`或`Query`中使用`tuple_()`现在会在看到`tuple_()`对象呈现自身以获取行时引发`CompileError`（即，如果元组在子查询的列子句中，不会引发错误）。对于 ORM 使用，`Bundle`对象是一个明确的指令，表示应将一系列列作为子元组返回每行，并且建议由错误消息。此外，在所有上下文中，元组现在将用括号呈现。以前，在列上下文中，括号化不会呈现，导致行为未定义。

    参考：[#5127](https://www.sqlalchemy.org/trac/ticket/5127)

+   **[sql] [bug] [postgresql]**

    改进了对包含百分号的列名的支持，修复了涉及匿名标签的问题，这些问题还嵌入了一个带有百分号的列名，以及重新建立了对在 psycopg2 方言中嵌入百分号的绑定参数名称的支持，使用了类似于 cx_Oracle 方言使用的晚期转义过程。

    参考：[#5653](https://www.sqlalchemy.org/trac/ticket/5653)

+   **[sql] [bug]**

    作为`FunctionElement`的子类创建的自定义函数现在将基于函数的“名称”生成一个“匿名标签”，就像任何其他`Function`对象一样，例如`"SELECT myfunc() AS myfunc_1"`。虽然 SELECT 语句不再需要标签以使结果代理对象正常工作，但 ORM 仍然通过使用对象作为映射键来定位行中的列，当列表达式具有不同的名称时，这种方法更可靠。无论如何，现在通过`func`生成的函数和作为自定义`FunctionElement`对象生成的函数之间的行为都是一致的。

    参考：[#4887](https://www.sqlalchemy.org/trac/ticket/4887)

+   **[sql] [bug]**

    重新设计了`ClauseElement.compare()`方法，采用了一种新的基于访问者的方法，并额外添加了测试覆盖，确保所有`ClauseElement`子类可以在结构上准确地相互比较。结构比较功能目前在 ORM 中仅被轻微使用，但它也可能成为新缓存功能的基础。

    参考：[#4336](https://www.sqlalchemy.org/trac/ticket/4336)

+   **[sql] [bug]**

    弃用在除了 PostgreSQL 之外的方言中使用`DISTINCT ON`。弃用 MySQL 方言中旧的字符串 distinct 用法

    参考：[#4002](https://www.sqlalchemy.org/trac/ticket/4002)

+   **[sql] [bug]**

    `_selectable.CompoundSelect` 的 ORDER BY 子句，例如 UNION、EXCEPT 等，在应用 `CompoundSelect.order_by()` 时不会呈现与给定列相关联的表名称，以表绑定列的形式。大多数数据库要求 ORDER BY 子句中的名称仅表示为标签名称，这些名称与第一个 SELECT 语句中的名称匹配。此更改与 [#4617](https://www.sqlalchemy.org/trac/ticket/4617) 相关，以前的解决方法是引用 `_selectable.CompoundSelect` 的 `.c` 属性，以便访问没有表名称的列。由于子查询现在已命名，此更改允许继续使用解决方法，并允许表绑定列以及 `CompoundSelect.selected_columns` 集合在 `CompoundSelect.order_by()` 方法中可用。

    参考：[#4617](https://www.sqlalchemy.org/trac/ticket/4617)

+   **[sql] [错误]**

    `Join` 构造不再将“onclause”视为要从包含 `Select` 对象的 FROM 列表中省略的其他 FROM 对象的来源。这适用于包含对 JOIN 外部另一个 FROM 对象的引用的 ON 子句；虽然从 SQL 视角来看通常不正确，但省略它也是不正确的，行为变更使得 `Select` / `Join` 的行为更加直观。

    参考：[#4621](https://www.sqlalchemy.org/trac/ticket/4621)

+   **[sql] [已弃用]**

    `Join.alias()` 方法已弃用，并将在 SQLAlchemy 2.0 中移除。应该使用显式的 select + 子查询，或者对内部表进行别名处理。

    参考：[#5010](https://www.sqlalchemy.org/trac/ticket/5010)

+   **[sql] [已弃用]**

    当定义具有相同名称的列时，`Table` 类现在会引发弃用警告。为了替换列，新增了一个参数 `Table.append_column.replace_existing` 到 `Table.append_column()` 方法中。

    当使用字符串调用 `ColumnCollection.contains_column()` 时，现在会引发错误，建议调用者改用 `in`。

+   **[sql] [已移除]**

    在 1.3 版本中废弃的 “threadlocal” 执行策略已在 1.4 版本中移除，以及“引擎策略”概念和 `Engine.contextual_connect` 方法。暂时仍然接受“strategy=‘mock’”关键字参数，并发出废弃警告；对于此用例，请使用 `create_mock_engine()`。

    另请参阅

    “threadlocal” 引擎策略已废弃 - 从 1.3 迁移说明中讨论了废弃的原因。

    参考：[#4632](https://www.sqlalchemy.org/trac/ticket/4632)

+   **[sql] [已移除]**

    移除了 `sqlalchemy.sql.visitors.iterate_depthfirst` 和 `sqlalchemy.sql.visitors.traverse_depthfirst` 函数。这些函数在 SQLAlchemy 的任何部分都没有被使用。`iterate()` 和 `traverse()` 函数通常用于这些功能。还从剩余的函数中移除了未使用的选项，包括“column_collections”、“schema_visitor”。

+   **[sql] [已移除]**

    从 `Compiler` 对象中移除了绑定引擎的概念，并移除了 `Compiler` 的 `.execute()` 和 `.scalar()` 方法。这些方法实际上是十多年前遗忘的方法，没有实际用途，而且 `Compiler` 对象本身维护对 `Engine` 的引用是不合适的。

+   **[sql] [已移除]**

    移除了废弃的方法 `Compiled.compile`、`ClauseElement.__and__` 和 `ClauseElement.__or__` 以及属性 `Over.func`。

    移除了废弃的 `FromClause.count` 方法。请使用 `func` 命名空间中提供的 `count` 函数。

    参考：[#4643](https://www.sqlalchemy.org/trac/ticket/4643)

+   **[sql] [已移除]**

    移除了废弃的参数 `text.bindparams` 和 `text.typemap`。请参阅 `TextClause.bindparams()` 和 `TextClause.columns()` 方法。

    移除了废弃的参数 `Table.useexisting`。请使用 `Table.extend_existing`。

    参考：[#4643](https://www.sqlalchemy.org/trac/ticket/4643)

+   **[sql] [已重命名]**

    `Table` 参数 `mustexist` 已更名为 `Table.must_exist`，并且在使用时会发出警告。

+   **[sql] [renamed]**

    `SelectBase.as_scalar()` 和 `Query.as_scalar()` 方法已分别更名为 `SelectBase.scalar_subquery()` 和 `Query.scalar_subquery()`。旧名称在 1.4 系列中仍然存在，并带有弃用警告。此外，在列上下文中评估时，将 `SelectBase`、`Alias` 和其他 SELECT 导向对象隐式转换为标量子查询的行为也已弃用，并发出警告，建议显式调用 `SelectBase.scalar_subquery()` 方法。这个警告在以后的主要版本中将变为错误，但是当需要调用 `SelectBase.scalar_subquery()` 时，消息始终会很清晰。此更改的后半部分是为了清晰起见，并减少查询强制系统的隐式决策。之前称为 `Alias.as_scalar` 的 `Subquery.as_scalar()` 方法也已弃用；应直接从 `select()` 对象或 `Query` 对象调用 `.scalar_subquery()`。

    这个更改是将 `select()` 对象转换为不再直接属于“from clause”类层次结构的一部分的更大更改的一部分，这也包括对子句强制系统的彻底改革。

    参考：[#4617](https://www.sqlalchemy.org/trac/ticket/4617)

+   **[sql] [renamed]**

    为了实现更一致的命名，几个运算符已更名。

    运算符的更改包括：

    +   `isfalse` 现在是 `is_false`

    +   `isnot_distinct_from` 现在是 `is_not_distinct_from`

    +   `istrue` 现在是 `is_true`

    +   `notbetween` 现在是 `not_between`

    +   `notcontains` 现在是 `not_contains`

    +   `notendswith` 现在是 `not_endswith`

    +   `notilike` 现在是 `not_ilike`

    +   `notlike` 现在是 `not_like`

    +   `notmatch` 现在是 `not_match`

    +   `notstartswith` 现在是 `not_startswith`

    +   `nullsfirst` 现在是 `nulls_first`

    +   `nullslast` 现在是 `nulls_last`

    +   `isnot` 现在是 `is_not`

    +   `notin_` 现在是 `not_in`

    由于这些是核心操作符，此更改的内部迁移策略是支持传统术语一段时间 - 如果不是永久地 - 但更新所有文档、教程和内部使用到新术语。新术语用于定义函数，传统术语已被弃用为新术语的别名。

    参考：[#5429](https://www.sqlalchemy.org/trac/ticket/5429), [#5435](https://www.sqlalchemy.org/trac/ticket/5435)

+   **[sql] [postgresql]**

    允许在 PostgreSQL 中创建`Sequence`时指定数据类型，使用参数`Sequence.data_type`。

    参考：[#5498](https://www.sqlalchemy.org/trac/ticket/5498)

+   **[sql] [reflection]**

    外键“ON UPDATE”的“NO ACTION”关键字现在被视为所有支持的后端（SQlite、MySQL、PostgreSQL）上外键的默认级联，并且在检测到时不包含在反射字典中；在任何情况下，这已经是 PostgreSQL 和 MySQL 所有以前的 SQLAlchemy 版本的行为。当检测到时，“RESTRICT”关键字被积极存储；PostgreSQL 报告此关键字，MySQL 从版本 8.0 开始也是如此。在较早的 MySQL 版本中，数据库不会报告此关键字。

    参考：[#4741](https://www.sqlalchemy.org/trac/ticket/4741)

+   **[sql] [reflection]**

    增加了对“identity”列的反射支持，这些列现在作为`Inspector.get_columns()`返回的结构的一部分返回。在反射完整的`Table`对象时，identity 列将使用`Identity`构造表示。目前支持的后端是 PostgreSQL >= 10、Oracle >= 12 和 MSSQL（具有不同的语法和功能子集）。

    参考：[#5324](https://www.sqlalchemy.org/trac/ticket/5324), [#5527](https://www.sqlalchemy.org/trac/ticket/5527)

### schema

+   **[schema] [change]**

    `Enum.create_constraint`和`Boolean.create_constraint`参数现在默认为 False，表示当创建这两种数据类型的所谓“非本地”版本时，默认不会生成 CHECK 约束。这些 CHECK 约束会带来应该选择的模式管理维护复杂性，而不是默认打开。

    另请参阅

    枚举和布尔数据类型不再默认为“创建约束”

    参考：[#5367](https://www.sqlalchemy.org/trac/ticket/5367)

+   **[schema] [错误]**

    清理了数据类型的内部`str()`，使得所有类型都可以生成一个没有任何方言的字符串表示，包括第三方方言类型也可以在没有该方言的情况下工作。字符串表示默认为该类型的大写名称，没有其他内容。

    参考：[#4262](https://www.sqlalchemy.org/trac/ticket/4262)

+   **[schema] [已移除]**

    移除了弃用的类`Binary`。请使用`LargeBinary`。

    参考：[#4643](https://www.sqlalchemy.org/trac/ticket/4643)

+   **[schema] [重命名]**

    将`Table.tometadata()`方法重命名为`Table.to_metadata()`。之前的名称仍然存在，但会有弃用警告。

    参考：[#5413](https://www.sqlalchemy.org/trac/ticket/5413)

+   **[schema] [sql]**

    添加了`Identity`构造，可用于配置生成的具有 GENERATED { ALWAYS | BY DEFAULT } AS IDENTITY 的标识列。目前支持的后端是 PostgreSQL >= 10、Oracle >= 12 和 MSSQL（具有不同语法和功能子集）。

    参考：[#5324](https://www.sqlalchemy.org/trac/ticket/5324)、[#5360](https://www.sqlalchemy.org/trac/ticket/5360)、[#5362](https://www.sqlalchemy.org/trac/ticket/5362)

### 扩展

+   **[extensions] [用例]**

    使用`sqlalchemy.ext.compiled`扩展创建的自定义编译器构造将在将自定义构造解释为 SELECT 语句中列子句的元素时自动向编译器添加上下文信息，以便自定义元素可以作为结果行映射中的键进行定位，这是 ORM 用于将列元素匹配到结果元组中的定位方式。

    参考：[#4887](https://www.sqlalchemy.org/trac/ticket/4887)

+   **[extensions] [更改]**

    添加了新参数 `AutomapBase.prepare.autoload_with`，该参数取代了 `AutomapBase.prepare.reflect` 和 `AutomapBase.prepare.engine`。

    参考文献：[#5142](https://www.sqlalchemy.org/trac/ticket/5142)

### postgresql

+   **[postgresql] [用例]**

    为所有 psycopg2、asyncpg 和 pg8000 方言添加了对 PostgreSQL “readonly” 和 “deferrable” 标志的支持。这利用了一种新的通用化的“隔离级别”API，以支持通过执行选项设置的其他类型的会话属性，当连接返回到连接池时，这些属性会被可靠地重置。

    另请参阅

    设置为 READ ONLY / DEFERRABLE

    参考文献：[#5549](https://www.sqlalchemy.org/trac/ticket/5549)

+   **[postgresql] [用例]**

    当 `stream_results=True` 时由诸如 PostgreSQL 等方言使用的 `BufferedRowResultProxy` 的最大缓冲区大小现在可以设置为大于 1000 的数字，并且缓冲区将增长到该大小。之前，即使将值设置得更大，缓冲区也不会超过 1000。缓冲区的增长现在还基于一个简单的乘法因子，目前设置为 5。感谢 Soumaya Mauthoor 提供的拉取请求。

    参考文献：[#4914](https://www.sqlalchemy.org/trac/ticket/4914)

+   **[postgresql] [更改]**

    在使用 psycopg2 方言连接到 PostgreSQL 时，psycopg2 的最低版本设置为 2.7。psycopg2 方言依赖于过去几年中发布的许多 psycopg2 特性，因此为了简化方言，现在要求最低版本为 2017 年 3 月发布的版本 2.7。

+   **[postgresql] [性能]**

    psycopg2 方言现在默认使用非常高效的 `execute_values()` psycopg2 扩展来编译 INSERT 语句，并且在使用此扩展时还实现了 RETURNING 支持。这允许包括自增 SERIAL 或 IDENTITY 值的 INSERT 语句运行速度非常快，同时仍然能够返回新生成的主键值。ORM 将在另一个单独的更改中集成此新功能。

    另请参阅

    psycopg2 方言特性，默认情况下 INSERT 语句使用“execute_values”并带有 RETURNING - 有关 `executemany_mode` 参数的所有更改的完整列表。

    参考文献：[#5401](https://www.sqlalchemy.org/trac/ticket/5401)

+   **[postgresql] [错误]**

    pg8000 方言已经针对最新版本的 pg8000 驱动程序进行了修订和现代化，适用于 PostgreSQL。感谢 Tony Locke 的拉取请求。请注意，这必然将 pg8000 固定在 1.16.6 或更高版本，不再支持 Python 2。需要 pg8000 的 Python 2 用户应确保其要求被固定为 `SQLAlchemy<1.4`。

+   **[postgresql] [已弃用]**

    pygresql 和 py-postgresql 方言已弃用。

    参考：[#5189](https://www.sqlalchemy.org/trac/ticket/5189)

+   **[postgresql] [已移除]**

    移除了形式为 `postgres://` 的弃用引擎 URL 的支持；多年来一直发出警告，项目应该使用 `postgresql://`。

    参考：[#4643](https://www.sqlalchemy.org/trac/ticket/4643)

### mysql

+   **[mysql] [特性]**

    向 mysql 方言添加了对 MariaDB Connector/Python 的支持。原始拉取请求感谢 Georg Richter。

    参考：[#5459](https://www.sqlalchemy.org/trac/ticket/5459)

+   **[mysql] [用例]**

    添加了一个新的方言标记 “mariadb”，可用于 `create_engine()` URL 中代替 “mysql”。这将提供一个 MariaDB 方言子类，强制使用中的 MySQLDialect 的 “is_mariadb” 标志为 True。如果接收到的服务器版本字符串不指示正在使用 MariaDB，则方言将引发错误。这对于特定于 MariaDB 的测试场景以及支持仅对 MariaDB 概念进行硬编码的应用程序非常有用。随着 MariaDB 和 MySQL 的功能集和使用模式继续分歧，这种模式可能会变得更加突出。

    参考：[#5496](https://www.sqlalchemy.org/trac/ticket/5496)

+   **[mysql] [用例]**

    添加了对 MariaDB 10.3 及更高版本使用 `Sequence` 构造的支持，因为该数据库现在支持该功能。该构造与 `Table` 对象集成方式与其他数据库（如 PostgreSQL 和 Oracle）相同；如果在整数主键 “autoincrement” 列上存在，它将用于生成默认值。为了向后兼容，以支持具有 `Sequence` 的 `Table`，以支持仅支持序列的数据库，如 Oracle，同时仍然不触发 MariaDB 的序列，应该设置 optional=True 标志，表示仅当目标数据库没有其他选项时才使用该序列生成主键。

    另请参阅

    添加了对 MariaDB 10.3 的 Sequence 支持

    参考：[#4976](https://www.sqlalchemy.org/trac/ticket/4976)

+   **[mysql] [错误]**

    MySQL 和 MariaDB dialect 现在从 `information_schema.tables` 系统视图中查询以确定特定表是否存在。之前，使用“DESCRIBE”命令与异常捕获来检测不存在的表，这将导致在连接上发出 ROLLBACK 的不良效果。似乎存在遗留的编码问题，阻止了“SHOW TABLES”的使用，但由于 MySQL 支持现在已经是 5.0.2 或更高版本，因此在所有情况下都可以使用 `information_schema` 表。

+   **[mysql] [错误]**

    使用 `with_for_update()` 中的“skip_locked”关键字将在所有 MySQL 后端上呈现“SKIP LOCKED”，这意味着它将在 MySQL 小于 8 版本和当前 MariaDB 后端上失败。这是因为这些后端不支持“SKIP LOCKED”或任何等效功能，因此不应该悄悄地忽略此错误。这是在 1.3 系列中从警告升级的。

    参考：[#5568](https://www.sqlalchemy.org/trac/ticket/5568)

+   **[mysql] [错误]**

    MySQL dialect 的 `server_version_info` 元组现在全部是数值。不再包含像“MariaDB”这样的字符串标记，以便在所有情况下进行数值比较。应该咨询 dialect 上的 `.is_mariadb` 标志以确定是否检测到 mariadb。另外，还删除了旨在支持极旧的 MySQL 版本 3.x 和 4.x 的结构；现在支持的最低 MySQL 版本是 5.0.2。

    参考：[#4189](https://www.sqlalchemy.org/trac/ticket/4189)

+   **[mysql] [已弃用]**

    OurSQL dialect 已弃用。

    参考：[#5189](https://www.sqlalchemy.org/trac/ticket/5189)

+   **[mysql] [已移除]**

    删除了自版本 1.0 起已弃用的 dialect `mysql+gaerdbms`。直接使用 MySQLdb dialect。

    从 `ENUM` 和 `SET` 中移除了已弃用的参数 `quoting`。当需要时，SQLAlchemy 会自动引用传递给 enum 或 set 的值。

    参考：[#4643](https://www.sqlalchemy.org/trac/ticket/4643)

### sqlite

+   **[sqlite] [变更]**

    放弃了对右嵌套联接重写的支持，以支持发布于 2013 年之前的旧 SQLite 版本 3.7.16。预计现在支持的所有现代 Python 版本都应该包含更新的 SQLite 版本。

    另见

    从 SQLite dialect 中删除“join rewriting”逻辑；更新导入

    参考：[#4895](https://www.sqlalchemy.org/trac/ticket/4895)

### mssql

+   **[mssql] [功能] [sql]**

    使用 `JSON` 实现，在 SQL Server 方言上添加了对 `JSON` 数据类型的支持，该实现针对 `NVARCHAR(max)` 数据类型实现了 SQL Server 的 JSON 功能，符合 SQL Server 文档。实现感谢 Gord Thompson。

    参考：[#4384](https://www.sqlalchemy.org/trac/ticket/4384)

+   **[mssql] [feature]**

    为 Microsoft SQL Server 添加了“CREATE SEQUENCE”和完整的 `Sequence` 支持。这移除了使用 `Sequence` 对象操纵 IDENTITY 特性的已弃用功能，现在应该使用文档中记录的 `mssql_identity_start` 和 `mssql_identity_increment` 执行此操作。该变更包括一个新参数 `Sequence.data_type`，以适应 SQL Server 的数据类型选择，该后端的选择包括 INTEGER、BIGINT 和 DECIMAL(n, 0)。对于 SQL Server 版本的 `Sequence`，默认起始值已设置为 1；该默认值现在在所有后端的 CREATE SEQUENCE DDL 中被发出。

    另请参阅

    向 SQL Server 添加了与 IDENTITY 不同的 Sequence 支持

    参考：[#4235](https://www.sqlalchemy.org/trac/ticket/4235), [#4633](https://www.sqlalchemy.org/trac/ticket/4633)

+   **[mssql] [usecase] [postgresql] [reflection] [schema]**

    改进了对覆盖索引（包含 INCLUDE 列）的支持。增加了从核心渲染带有 INCLUDE 子句的 CREATE INDEX 语句的 postgresql 的能力。索引反射还为 mssql 和 postgresql（11+）分别报告了 INCLUDE 列。

    参考：[#4458](https://www.sqlalchemy.org/trac/ticket/4458)

+   **[mssql] [usecase] [postgresql]**

    增加了对部分索引/过滤索引的检查/反射支持，即使用 `mssql_where` 或 `postgresql_where` 参数的索引，可以通过 `Index` 进行反映。该条目既是由 `Inspector.get_indexes()` 返回的字典的一部分，也是通过反射的 `Index` 构造的一部分。感谢 Ramon Williams 提交的拉取请求。

    参考：[#4966](https://www.sqlalchemy.org/trac/ticket/4966)

+   **[mssql] [usecase] [reflection]**

    增加了对使用 SQL Server 方言的临时表的反射支持。现在可以从 MSSQL “tempdb” 系统目录中自省以“#”号前缀的表名。

    参考：[#5506](https://www.sqlalchemy.org/trac/ticket/5506)

+   **[mssql] [更改]**

    SQL Server OFFSET 和 FETCH 关键字现在用于限制/偏移，而不是使用窗口函数，适用于 SQL Server 版本 11 及更高版本。对于仅包含 LIMIT 的查询仍然使用 TOP。感谢 Elkin 的拉取请求。

    参考：[#5084](https://www.sqlalchemy.org/trac/ticket/5084)

+   **[mssql] [错误] [模式]**

    修复了`sqlalchemy.engine.reflection.has_table()`对临时表始终返回`False`的问题。

    参考：[#5597](https://www.sqlalchemy.org/trac/ticket/5597)

+   **[mssql] [错误]**

    修复了`DATETIMEOFFSET`数据类型的基类，基于`DateTime`类层次结构，因为这是一个持有日期时间的数据类型。

    参考：[#4980](https://www.sqlalchemy.org/trac/ticket/4980)

+   **[mssql] [已弃用]**

    adodbapi 和 mxODBC 方言已弃用。

    参考：[#5189](https://www.sqlalchemy.org/trac/ticket/5189)

+   **[mssql]**

    mssql 方言将假定至少使用 MSSQL 2005。如果检测到旧版本，不会引发硬异常，但对于旧版本可能会导致操作失败。

+   **[mssql] [反射]**

    作为反映`Identity`对象支持的一部分，方法`Inspector.get_columns()`不再作为`dialect_options`的一部分返回`mssql_identity_start`和`mssql_identity_increment`。请改用`identity`键中的信息。

    参考：[#5527](https://www.sqlalchemy.org/trac/ticket/5527)

+   **[mssql] [引擎]**

    弃用了`legacy_schema_aliasing`参数给`sqlalchemy.create_engine()`。这是一个自版本 1.1 起默认为 False 的长期过时参数。

    参考：[#4809](https://www.sqlalchemy.org/trac/ticket/4809)

### oracle

+   **[oracle] [用例]**

    Oracle 方言的默认`max_identifier_length`现在为 128 个字符，除非在首次连接时兼容性版本小于 12.2，此时使用传统的 30 个字符长度。这是继 1.3 系列提交的问题的延续，该系列在首次连接时添加了最大标识符长度检测，并警告 Oracle 服务器的更改。

    另请参见

    最大标识符长度 - 在 Oracle ���言文档中

    参考：[#4857](https://www.sqlalchemy.org/trac/ticket/4857)

+   **[oracle] [变更]**

    Oracle 现在在使用 LIMIT / OFFSET 方案时，使用命名子查询而不是未命名子查询，当它透明地重写 SELECT 语句以使用包含 ROWNUM 的子查询时。这一变化是更大变化的一部分，其中未命名子查询不再直接受 Core 支持，以及现代化 Oracle 方言内部使用 select()构造的变化。

+   **[oracle] [错误]**

    在 Oracle 数据库上正确渲染`Sequence`和`Identity`列选项`nominvalue`和`nomaxvalue`为`NOMAXVALUE`和``NOMINVALUE`。

+   **[oracle] [错误]**

    Oracle 方言的`INTERVAL`类现在正确地是`Interval`的抽象版本的子类，以及正确的“模拟”基类，这允许在本地模式和非本地模式下正确行为；以前它只基于`TypeEngine`。

    参考：[#4971](https://www.sqlalchemy.org/trac/ticket/4971)

### 杂项

+   **[弃用] [firebird]**

    Firebird 方言已被弃用，因为现在有一个第三方方言支持这个数据库。

    参考：[#5189](https://www.sqlalchemy.org/trac/ticket/5189)

+   **[杂项] [弃用]**

    Sybase 方言已被弃用。

    参考：[#5189](https://www.sqlalchemy.org/trac/ticket/5189)

### 一般

+   **[一般] [变更]**

    ”python setup.py test”不再是一个测试运行程序，因为 Pypa 已经弃用了。请使用“tox”而不带参数进行基本测试运行。

    参考：[#4789](https://www.sqlalchemy.org/trac/ticket/4789)

+   **[一般] [错误]**

    重构了用于交叉导入具有彼此之间相互依赖关系的模块的内部约定，使得不再修改函数和方法的检查参数。这使得像 pylint、Pycharm、其他代码检查器以及未来可能添加的 pep-484 实现等工具能够正确运行，因为它们不再看到函数调用中缺少的参数。新方法也更简单、更高效。

    另请参阅

    修复了内部导入约定，使代码检查器可以正常工作

    参考：[#4656](https://www.sqlalchemy.org/trac/ticket/4656)，[#4689](https://www.sqlalchemy.org/trac/ticket/4689)

### 平台

+   **[平台] [变更]**

    `importlib_metadata`库用于扫描 setuptools 入口点，而不是 pkg_resources。由于 importlib_metadata 是 Python 3.8 及以上版本自带的一个小型库��因此兼容性库将作为 Python 3.8 之前版本的依赖项安装。

    参考：[#5400](https://www.sqlalchemy.org/trac/ticket/5400)

+   **[平台] [更改]**

    安装已现代化，大多数包元数据现在使用 setup.cfg。

    参考：[#5404](https://www.sqlalchemy.org/trac/ticket/5404)

+   **[平台] [已移除]**

    不再支持已达到 EOL 的 python 3.4 和 3.5。SQLAlchemy 1.4 系列需要 python 2.7 或 3.6+。

    另请参阅

    Python 3.6 是最低要求的 Python 3 版本；仍支持 Python 2.7

    参考：[#5634](https://www.sqlalchemy.org/trac/ticket/5634)

+   **[平台] [已移除]**

    移除了与支持 Jython 和 zxJDBC 相关的所有方言代码。SQLAlchemy 多年来一直不支持 Jython，并且当前的 zxJDBC 代码预计根本不起作用；目前它只是占用空间，并通过出现在文档中增加混乱。目前，Jython 在其发布中已实现了对 Python 2.7 的支持，但尚未实现对 Python 3 的支持。如果再次支持 Jython，它应该针对 Python 3 版本的 Jython，并且各种后端的各种 zxJDBC 存根应该作为第三方方言实现。

    参考：[#5094](https://www.sqlalchemy.org/trac/ticket/5094)

### orm

+   **[orm] [功能]**

    现在 ORM 可以生成以前仅在使用`Query`时可用的查询，直接使用`select()`构造。ORM“插件”现在可以在 Core `Select`中建立自己，允许以前在`Query`内部的大部分查询构建逻辑现在在编译级扩展中进行。类似的更改也已针对`Update`和`Delete`构造进行。当使用`Session.execute()`调用这些构造时，现在方法内部会执行 ORM 相关的工作。对于`Select`，现在返回的`Result`对象包含 ORM 级别的实体和结果。

    另请参阅

    ORM 查询现在统一了 select、update、delete；2.0 风格的执行方式可用

    参考：[#5159](https://www.sqlalchemy.org/trac/ticket/5159)

+   **[orm] [功能]**

    增加了向查询中的关系属性生成的 ON 子句添加任意条件的功能，适用于诸如 `Query.join()` 这样的方法以及像 `joinedload()` 这样的加载器选项。此外，该选项的“全局”版本允许全局地将条件限制应用于查询中的特定实体。

    另请参阅

    向加载器选项添加条件

    添加全局 WHERE / ON 条件

    `with_loader_criteria()`

    参考：[#4472](https://www.sqlalchemy.org/trac/ticket/4472)

+   **[orm] [功能]**

    ORM Declarative 系统现在统一到 ORM 本身中，具有新的导入空间在 `sqlalchemy.orm` 下以及新种类的映射。支持基于装饰器的映射，而无需使用基类，支持具有对关系的声明类注册表的经典样式-mapper() 调用，并完全集成 Declarative 与 `dataclasses` 和 `attrs` 等第三方类属性系统的支持。

    另请参阅

    Declarative 现在与 ORM 集成，具有新功能

    Python Dataclasses, attrs Supported w/ Declarative, Imperative Mappings

    参考：[#5508](https://www.sqlalchemy.org/trac/ticket/5508)

+   **[orm] [功能]**

    当在映射器上或通过查询选项配置时，急切加载器（如连接加载、SELECT IN 加载等）现在将在过期对象刷新时被调用；在 selectinload 和 subqueryload 的情况下，由于额外加载仅针对单个对象，因此在这些情况下使用“immediateload”方案，该方案类似于延迟加载发出的单个父查询。

    另请参阅

    急切加载器在取消过期操作期间发出

    参考：[#1763](https://www.sqlalchemy.org/trac/ticket/1763)

+   **[orm] [功能]**

    增加了对使用 Python `dataclasses` 装饰器定义的 Python 类的直接映射支持。感谢 Václav Klusák 提交的拉取请求。新功能集成到了 Declarative 级别的新支持中，支持诸如 `dataclasses` 和 `attrs` 等系统。

    另请参阅

    Python Dataclasses, attrs Supported w/ Declarative, Imperative Mappings

    Declarative 现在与 ORM 集成，具有新功能

    参考：[#5027](https://www.sqlalchemy.org/trac/ticket/5027)

+   **[orm] [功能]**

    通过`defer.raiseload`参数在`defer()`和`deferred()`上增加了“raiseload”功能，为 ORM 映射列提供了类似于关系映射属性的`raiseload()`选项的行为。此更改还包括有关延迟列在过期方面的一些行为更改；详细信息请参阅迁移说明。

    另请参阅

    列的 Raiseload

    参考：[#4826](https://www.sqlalchemy.org/trac/ticket/4826)

+   **[orm] [用例]**

    在 ORM 批量更新和删除中进行的评估器，对于`synchronize_session="evaluate"`现在支持 IN 和 NOT IN 操作符。同时也支持元组 IN。

    参考：[#1653](https://www.sqlalchemy.org/trac/ticket/1653)

+   **[orm] [用例]**

    增强了跟踪关系是否会相互冲突的逻辑，当它们写入相同列时，包括两个关系之间应该有“backref”的简单情况。这意味着如果两个关系不是只读的，没有通过 back_populates 连接，并且不以其他方式处于继承兄弟/覆盖安排中，并且将填充相同的外键列，则在映射器配置时会发出警告，警告可能会出现冲突。新增参数`relationship.overlaps`适用于那些非常罕见的情况，其中这种重叠的持久性安排可能是不可避免的。

    参考：[#5171](https://www.sqlalchemy.org/trac/ticket/5171)

+   **[orm] [用例]**

    ORM 批量更新和删除操作，历来可通过`Query.update()`和`Query.delete()`方法以及通过`Update`和`Delete`构造进行 2.0 风格执行，现在将自动适应单表继承鉴别器所需的额外 WHERE 条件，以限制语句仅限于引用特定子类型的行。新的`with_loader_criteria()`构造也支持批量更新/删除操作。

    引用：[#3903](https://www.sqlalchemy.org/trac/ticket/3903), [#5018](https://www.sqlalchemy.org/trac/ticket/5018)

+   **[orm] [用例]**

    更新`relationship.sync_backref`中的标志，在关系中将其隐式设为`False`，以防止在`viewonly=True`关系中发生同步事件。

    另见

    只读关系不同步反向引用

    引用：[#5237](https://www.sqlalchemy.org/trac/ticket/5237)

+   **[orm] [更改]**

    调整了待冲洗的对象具有已存在于标识映射中的标识的情况，以发出警告，而不是抛出`FlushError`。理由是使冲洗继续进行，并引发一个`IntegrityError`，就像如果已存在的对象尚未存在于标识映射中一样。这有助于使用`IntegrityError`来捕获表中是否已存在行的方案。

    另见

    “新实例与现有标识冲突”错误现在是警告

    引用：[#4662](https://www.sqlalchemy.org/trac/ticket/4662)

+   **[orm] [更改] [sql]**

    现在，一些核心和 ORM 查询对象在编译步骤中执行更多的 Python 计算任务，而不是在构造时。这是为了支持即将到来的缓存模型，该模型将根据语句构造派生的缓存键来缓存编译后的语句结构，该语句构造本身预计每次在 Python 代码中使用时都会新建。这意味着这些对象的内部状态可能不再与以前相同，并且某些但不是所有的错误引发场景，例如各种参数验证，都将发生在编译/执行阶段，而不是语句构造时。有关完整详细信息，请参阅下面链接的迁移说明。

    另见

    许多核心和 ORM 语句对象现在在编译阶段执行大部分构造和验证

+   **[orm] [更改]**

    客户端自动对行进行唯一化的功能在新的 2.0 风格 ORM 查询中已关闭。这既提高了清晰度，又提高了性能。然而，在使用连接式贪婪加载集合时，客户端自动对行进行唯一化通常是必要的，因为每个集合中的元素都会有主实体的重复，因为使用了连接。这种唯一化现在必须手动启用，并且可以使用新的`Result.unique()`修饰符来实现。为了避免静默失败，当 2.0 风格的 ORM 查询的结果使用连接式贪婪加载集合时，ORM 明确要求调用该方法。在任何情况下，较新的`selectinload()`策略可能更可取用于集合的贪婪加载。

    参见也

    ORM 行默认情况下不唯一化

    参考：[#4395](https://www.sqlalchemy.org/trac/ticket/4395)

+   **[orm] [更改]**

    当 ORM 被要求隐式地将`select()`结构强制转换为子查询时，ORM 现在会发出警告。这种情况发生在`Query.select_entity_from()`和`Query.select_from()`方法以及`with_polymorphic()`函数等地方。当直接将`SelectBase`(由`select()`产生)或`Query`对象传递给这些函数和其他函数时，ORM 通常会自动调用`SelectBase.alias()`方法将它们强制转换为子查询（现在已由`SelectBase.subquery()`方法取代）。有关更多详细信息，请参见下面链接的迁移说明。

    参见也

    SELECT 语句不再隐式地被认为是 FROM 子句

    参考：[#4617](https://www.sqlalchemy.org/trac/ticket/4617)

+   **[orm] [更改]**

    `Query`返回的“KeyedTuple”类现在被 Core 的`Row`类替换，其行为与 KeyedTuple 相同。在 SQLAlchemy 2.0 中，Core 和 ORM 都将使用相同的`Row`对象返回结果行。在此期间，Core 使用一个保持“RowProxy”以前映射/元组混合行为的向后兼容类`LegacyRow`。

    参见

    Query 返回的“KeyedTuple”对象被 Row 替换

    参考：[#4710](https://www.sqlalchemy.org/trac/ticket/4710)

+   **[orm] [performance]**

    `Query.update()`和`Query.delete()`以及它们的 2.0 风格的对应方法现在在使用“fetch”策略时使用 RETURNING，以获取受影响的主键标识列表，而不是在使用支持 RETURNING 的后端时发出单独的 SELECT。此外，当更新的属性不会过期时，“fetch”策略通常不会使已更新的属性过期，而是直接应用更新的值，以避免刷新对象。如果“evaluate”策略无法在 Python 中评估 SQL 表达式，则会将其回退到将已更新的属性过期的方式。

    参见

    ORM 批量更新和删除在可用时使用 RETURNING 进行“fetch”策略

+   **[orm] [performance] [postgresql]**

    通过对 Core 进行增强，实现了对 psycopg2 `execute_values()`扩展的支持，使其在 ORM 刷新过程中使用，从而可以将批量 INSERT 语句一起使用，并且 RETURNING 现在可以在多个参数集之间使用，以批量检索主键值。这使得 ORM 代表 PostgreSQL 发出的几乎所有 INSERT 语句都可以批量提交，而且通过`execute_values()`扩展，对于这个特定后端，其性能比普通的 executemany()快五倍。

    参见

    ORM 使用 psycopg2 批量插入现在在大多数情况下使用 RETURNING 批量语句

    参考：[#5263](https://www.sqlalchemy.org/trac/ticket/5263)

+   **[orm] [bug]**

    对于针对映射的继承子类的查询，如果还使用了`Query.select_entity_from()`或类似的技术来提供一个现有的子查询以供 SELECT，那么如果给定的子查询返回的实体与给定的子类不对应，即它们是同一层次结构中的兄弟或超类，则现在会引发错误。以前，这些实体会无错误返回。此外，如果继承映射是单一继承映射，则给定的子查询必须针对多态鉴别器列应用适当的过滤以避免此错误；以前，`Query`会将此条件添加到外部查询中，但这会干扰一些返回其他类型实体的查询。

    另请参阅

    在使用自定义查询查询继承映射时，查询行为更加严格 migration_14.html#change-5122

    参考：[#5122](https://www.sqlalchemy.org/trac/ticket/5122)

+   **[orm] [bug]**

    内部属性符号`NO_VALUE`和`NEVER_SET`已统一，因为这两个符号之间没有实质性的区别，除了一些代码路径在微妙且未记录的方式下对它们进行区分之外，这些问题已经修复。

    参考：[#4696](https://www.sqlalchemy.org/trac/ticket/4696)

+   **[orm] [bug]**

    修复了一个 bug，当在一个映射器上指定了一个版本列，而该版本列针对的是一个`select()`构造，其中`version_id_col`本身针对的是底层表时，即使该值在 flush 时被本地持久化，访问时也会产生额外的加载。实际的修复是由于[#4617](https://www.sqlalchemy.org/trac/ticket/4617)中的更改，由于`select()`对象不再具有`.c`属性，因此不会让映射器误以为存在未知的列值。

    参考：[#4194](https://www.sqlalchemy.org/trac/ticket/4194)

+   **[orm] [bug]**

    如果实例是一个未映射的对象，则现在对于`InstrumentedAttribute`会引发`UnmappedInstanceError`。在此之前会引发`AttributeError`。感谢 Ramon Williams 的拉取请求。

    参考：[#3858](https://www.sqlalchemy.org/trac/ticket/3858)

+   **[orm] [bug]**

    `Session` 对象在构造后或上一个事务关闭后不再立即初始化 `SessionTransaction` 对象；相反，“自动开始”逻辑现在在下次需要时按需初始化新的 `SessionTransaction`。理由包括从已关闭的 `Session` 中删除引用循环，以及消除由创建经常立即丢弃的 `SessionTransaction` 对象所产生的开销。此更改影响了 `SessionEvents.after_transaction_create()` 钩子的行为，即当 `Session` 首次需要存在 `SessionTransaction` 时，事件将被触发，而不是在创建 `Session` 或上一个 `SessionTransaction` 关闭时。与 `Engine` 和数据库本身的交互保持不变。

    另请参阅

    Session 功能新增“自动开始”行为

    参考：[#5074](https://www.sqlalchemy.org/trac/ticket/5074)

+   **[orm] [bug]**

    为 ORM 查询上下文添加了新的实体定位功能，以处理 `Session` 使用绑定字典针对映射类而不是单个绑定的情况，以及 `Query` 针对最终生成自诸如 `Query.subquery()` 方法的 Core 语句的情况。首先使用深度搜索实现，当前方法利用统一的 `select()` 构造来跟踪构造中的第一个映射器。

    参考：[#4829](https://www.sqlalchemy.org/trac/ticket/4829)

+   **[orm] [bug] [inheritance]**

    如果在 `with_polymorphic()` 中同时设置了 `selectable` 和 `flat` 参数为 True，则会引发 `ArgumentError`。selectable 名称已被别名化，并且应用 flat=True 会用匿名名称覆盖 selectable 名称，以前会导致代码中断。Pull request 由 Ramon Williams 提供。

    参考：[#4212](https://www.sqlalchemy.org/trac/ticket/4212)

+   **[orm] [bug]**

    修复了多态加载内部的问题，在某些不过期的情况下会回退到更昂贵、即将被弃用的结果列查找形式，并且与“with_polymorphic”一起使用。

    参考：[#4718](https://www.sqlalchemy.org/trac/ticket/4718)

+   **[orm] [bug]**

    如果在同时设置了`relationship()`的视图（viewonly=True）和与持久性相关的“级联”设置，则会引发错误。当 viewonly 也被设置时，“级联”设置现在默认为非持久性相关的设置。这是从 [#4993](https://www.sqlalchemy.org/trac/ticket/4993) 继续的，其中该设置在 1.3 版本中更改为发出警告。

    请参阅

    持久性相关的级联操作不允许设置 viewonly=True

    参考：[#4994](https://www.sqlalchemy.org/trac/ticket/4994)

+   **[orm] [bug]**

    改进了声明式继承扫描，以避免在基本继承列表中多次出现相同的基类时出错。

    参考：[#4699](https://www.sqlalchemy.org/trac/ticket/4699)

+   **[orm] [bug]**

    修复了 ORM 版本化功能中的一个错误，在映射可选择的配置的计数器上分配明确的 version_id，在前一个值过期时会失败；这是因为映射属性未配置 active_history=True。

    参考：[#4195](https://www.sqlalchemy.org/trac/ticket/4195)

+   **[orm] [bug]**

    现在，如果 ORM 为具有主键但鉴别器列为空的多态实例加载行，则会引发异常，因为鉴别器列不应为空。

    参考：[#4836](https://www.sqlalchemy.org/trac/ticket/4836)

+   **[orm] [bug]**

    在新创建的对象上访问集合属性不再改变`__dict__`，但仍然返回一个空集合，就像以前一直是的那样。这使得集合属性的工作方式与返回`None`的标量属性相一致，但也不会改变`__dict__`。为了适应集合的变化，一旦初始创建，每次都会返回相同的空集合，并且当它被改变（例如追加、添加等）时，它将被移动到`__dict__`中。这消除了 ORM 中只读属性访问中的最后一个变异副作用。

    另请参阅

    在瞬态对象上访问未初始化的集合属性不再改变 __dict__

    参考：[#4519](https://www.sqlalchemy.org/trac/ticket/4519)

+   **[orm] [bug]**

    刷新过期对象现在将触发自动刷新，如果过期属性列表包括一个或多个使用`Session.expire()`或`Session.refresh()`方法明确过期或刷新的属性。这是为了在许多情况下不希望发生自动刷新的情况下找到一种折中方案，与属性被明确过期或刷新并且这些属性依赖于会话中需要刷新的其他挂起状态的情况相对应。这两种方法现在还增加了一个新标志`Session.expire.autoflush`和`Session.refresh.autoflush`，默认为 True；当设置为 False 时，这将禁用对这些属性取消过期时发生的自动刷新。

    参考：[#5226](https://www.sqlalchemy.org/trac/ticket/5226)

+   **[orm] [bug]**

    `relationship.cascade_backrefs`标志的行为将在 2.0 中被反转，并无条件地设置为`False`，以使反向引用不会从前向赋值到后向赋值的操作中级联保存更新。当参数保持默认值`True`时，如果发生这样的级联操作，将发出 2.0 弃用警告。可以通过在特定的`relationship()`上将标志设置为`False`来始终建立新行为，或者更一般地，可以通过将`Session.future`标志设置为 True 来全面设置。

    另请参阅

    2.0 中将移除的 cascade_backrefs 行为已弃用

    参考：[#5150](https://www.sqlalchemy.org/trac/ticket/5150)

+   **[orm] [已弃用]**

    `Query`以及动态关系加载器使用的“切片索引”功能在 SQLAlchemy 2.0 中将不再接受负索引。这些操作效率不高，会将整个集合加载进内存，这既令人意外又不可取。在 1.4 中，除非设置了`Session.future`标志，否则将会发出警告，否则将会引发 IndexError。

    参考：[#5606](https://www.sqlalchemy.org/trac/ticket/5606)

+   **[orm] [已弃用]**

    调用`Query.instances()`方法时未传递`QueryContext`已被弃用。最初的用例是当给出要选择的实体以及 DBAPI 游标对象时，`Query`可以生成 ORM 对象。但是，为了使其正常工作，需要从 SQLAlchemy 的`ResultProxy`传递一些基本元数据，该元数据是从映射的列表达式派生的，最初来自于`QueryContext`。要从任意 SELECT 语句中检索 ORM 结果，应使用`Query.from_statement()`方法。

    参考：[#4719](https://www.sqlalchemy.org/trac/ticket/4719)

+   **[orm] [已弃用]**

    在 ORM 操作中使用字符串来表示关系名称，例如`Query.join()`，以及在加载器选项中使用字符串来表示所有 ORM 属性名称，例如`selectinload()` 已经被弃用，并将在 SQLAlchemy 2.0 中移除。应该传递类绑定的属性。这样可以为给定方法提供更好的特异性，允许使用诸如`of_type()`之类的修改器，并减少内部复杂性。

    此外，`Query.join()`中的`aliased`和`from_joinpoint`参数也已被弃用。现在，`aliased()`构造提供了极大的灵活性和功能，应直接使用。

    另请参阅

    ORM 查询 - 在关系上连接/加载使用属性，而不是字符串

    ORM 查询 - join(…, aliased=True)，from_joinpoint 已移除

    参考：[#4705](https://www.sqlalchemy.org/trac/ticket/4705)，[#5202](https://www.sqlalchemy.org/trac/ticket/5202)

+   **[orm] [弃用]**

    在`Query.distinct()`中弃用的逻辑，自动将 ORDER BY 子句中的列添加到列子句中；这将在 2.0 版本中移除。

    另请参阅

    使用 DISTINCT 与其他列，但仅选择实体

    参考：[#5134](https://www.sqlalchemy.org/trac/ticket/5134)

+   **[orm] [弃用]**

    将关键字参数传递给诸如`Session.execute()`的方法，以传递到`Session.get_bind()`方法已被弃用；应传递新的`Session.execute.bind_arguments`字典。

    参考：[#5573](https://www.sqlalchemy.org/trac/ticket/5573)

+   **[orm] [弃用]**

    `eagerload()`和`relation()`是旧的别名，现已被弃用。分别使用`joinedload()`和`relationship()`。

    参考：[#5192](https://www.sqlalchemy.org/trac/ticket/5192)

+   **[orm] [已移除]**

    所有长期弃用的“扩展”类都已被移除，包括 MapperExtension、SessionExtension、PoolListener、ConnectionProxy、AttributeExtension。这些类自版本 0.7 起就已被弃用，长期被事件监听器系统取代。

    参考：[#4638](https://www.sqlalchemy.org/trac/ticket/4638)

+   **[orm] [已移除]**

    移除弃用的加载器选项`joinedload_all`、`subqueryload_all`、`lazyload_all`、`selectinload_all`。应使用方法链接的正常版本替代。

    参考：[#4642](https://www.sqlalchemy.org/trac/ticket/4642)

+   **[orm] [已移除]**

    移除弃用的函数`comparable_property`。请参考`hybrid`扩展。这还移除了声明性扩展中的函数`comparable_using`。

    移除弃用的函数`compile_mappers`。请使用`configure_mappers()`

    移除不推荐使用的方法 `collection.linker`。请参考 `AttributeEvents.init_collection()` 和 `AttributeEvents.dispose_collection()` 事件处理程序。

    移除不推荐使用的方法 `Session.prune` 和参数 `Session.weak_identity_map`。请参考 Session 引用行为 中的配方，以使用基于事件的方法来维护强引用。此更改还移除了类 `StrongInstanceDict`。

    移除不推荐使用的参数 `mapper.order_by`。使用 `Query.order_by()` 来确定结果集的排序方式。

    移除不推荐使用的参数 `Session._enable_transaction_accounting`。

    移除不推荐使用的参数 `Session.is_modified.passive`。

    引用：[#4643](https://www.sqlalchemy.org/trac/ticket/4643)

### engine

+   **[engine] [feature]**

    实现了全新的 `Result` 对象，取代了以前的 `ResultProxy` 对象。作为 Core 中的一个子类，`CursorResult` 具有与以前的 `ResultProxy` 兼容的调用接口，并且还添加了大量的新功能，可以应用于 Core 和 ORM 结果集，这些结果集现在集成到同一模型中。`Result` 包括列选择和重新排列、改进的 fetchmany 模式、唯一化等功能，以及可以用于从内存结构创建数据库结果的各种实现。

    另见

    新 Result 对象

    引用：[#4395](https://www.sqlalchemy.org/trac/ticket/4395)，[#4959](https://www.sqlalchemy.org/trac/ticket/4959)，[#5087](https://www.sqlalchemy.org/trac/ticket/5087)

+   **[engine] [feature] [orm]**

    SQLAlchemy 现在在 Core 和 ORM 中都包含对 Python asyncio 的支持，使用的是包含的 asyncio 扩展。该扩展利用 [greenlet](https://greenlet.readthedocs.io/en/latest/) 库，以使 SQLAlchemy 的同步导向内部能够适应 asyncio 接口，最终与 asyncio 数据库适配器进行交互成为可能。目前支持的单个驱动程序是 asyncpg 针对 PostgreSQL 的驱动程序。

    另见

    Core 和 ORM 的异步 IO 支持

    引用：[#3414](https://www.sqlalchemy.org/trac/ticket/3414)

+   **[engine] [feature] [alchemy2]**

    实现了`create_engine.future`参数，该参数使得与 SQLAlchemy 2 的向前兼容成为可能。用于与 SQLAlchemy 2 的向前兼容。该引擎始终具有自动启动的事务行为。

    另请参阅

    SQLAlchemy 2.0 - 主要迁移指南

    参考：[#4644](https://www.sqlalchemy.org/trac/ticket/4644)

+   **[engine] [feature] [pyodbc]**

    重新设计了“setinputsizes()”一组方言钩子，以便对任意 DBAPI 进行正确的扩展，通过允许方言具有单独的钩子，这些钩子可以以适合该 DBAPI 的样式调用 cursor.setinputsizes()。特别是这是为了支持 pyodbc 的使用方式，这种方式与 cx_Oracle 的基本不同。增加了对 pyodbc 的支持。

    参考：[#5649](https://www.sqlalchemy.org/trac/ticket/5649)

+   **[engine] [feature]**

    添加了新的反射方法`Inspector.get_sequence_names()`，该方法返回所有定义的序列，以及`Inspector.has_sequence()`用于检查特定序列是否存在。已经为支持`Sequence`的后端添加了对此方法的支持：PostgreSQL、Oracle 和 MariaDB >= 10.3。

    参考：[#2056](https://www.sqlalchemy.org/trac/ticket/2056)

+   **[engine] [feature]**

    `Table.autoload_with`参数现在可以直接接受`Inspector`对象，以及之前的`Engine`或`Connection`。 

    参考：[#4755](https://www.sqlalchemy.org/trac/ticket/4755)

+   **[engine] [change]**

    `RowProxy` 类不再是一个“代理”对象，而是在构造时直接填充了经过处理的 DBAPI 行元组的内容。现在被命名为 `Row`，Python 层面的值处理器的机制已经简化，特别是对 C 代码格式的影响，因此 DBAPI 行在前期就被处理成结果元组。`ResultProxy` 返回的对象现在是 `LegacyRow` 子类，保持了映射/元组混合行为，但基础的 `Row` 类现在更像一个命名元组。

    另请参阅

    RowProxy 不再是一个“代理”对象；现在被称为 Row 并且表现得像一个增强的命名元组

    参考：[#4710](https://www.sqlalchemy.org/trac/ticket/4710)

+   **[engine] [performance]**

    池“预连接”功能已经优化，不会在同一次检出操作中刚打开的 DBAPI 连接上调用。预连接仅适用于已经检入池中并再次被检出的 DBAPI 连接。

    参考：[#4524](https://www.sqlalchemy.org/trac/ticket/4524)

+   **[engine] [performance] [change] [py3k]**

    在 Python 3 下运行时禁用了方言启动时运行的“unicode 返回”检查，多年来一直用于测试当前 DBAPI 是否为 VARCHAR 和 NVARCHAR 数据类型返回 Python Unicode 或 Py2K 字符串的行为。在 Python 2 下仍然默认进行检查，但是当 SQLAlchemy 2.0 移除 Python 2 支持时，测试行为的机制也将被移除。

    当需要时，这种逻辑非常有效，然而现在 Python 3 已经成为标准，所有 DBAPI 都应该返回 Python 3 字符串作为字符数据类型。在第三方 DBAPI 不支持这一点的极少情况下，`String` 中的转换逻辑仍然可用，并且第三方方言可以通过设置方言级别标志 `returns_unicode_strings` 为 `String.RETURNS_CONDITIONAL` 或 `String.RETURNS_BYTES` 中的一个来在其初始方言标志中指定这一点，这两者都将在 Python 3 下启用 Unicode 转换。

    参考：[#5315](https://www.sqlalchemy.org/trac/ticket/5315)

+   **[engine] [bug]**

    修订了 `Connection.execution_options.schema_translate_map` 功能，使得在执行阶段而不是编译阶段处理 SQL 语句以接收特定模式名称。这是为了支持语句被高效地缓存。以前，为特定运行渲染到语句中的当前模式将被视为缓存键本身的一部分，这意味着对数百个模式的运行将有数百个缓存键，使缓存性能大大降低。新行为是渲染类似于 1.4 中添加的“后编译”渲染的方式，作为 [#4645](https://www.sqlalchemy.org/trac/ticket/4645)、[#4808](https://www.sqlalchemy.org/trac/ticket/4808) 的一部分。

    参考：[#5004](https://www.sqlalchemy.org/trac/ticket/5004)

+   **[engine] [bug]**

    `Connection` 对象现在不会在外层事务明确回滚之前清除已回滚的事务。这本质上与 ORM `Session` 长期以来的行为相同，需要在所有封闭事务上显式调用 `.rollback()` 才能使事务在逻辑上清除，即使 DBAPI 级别的事务已经回滚。新的行为有助于处理诸如“ORM 回滚测试套件”模式之类的情况，其中测试套件在 ORM 范围内回滚事务，但试图在外部控制事务范围的测试工具不希望隐式启动新事务。

    另请参阅

    基于子事务，现在可以根据连接级事务是否活动来进行更改

    参考：[#4712](https://www.sqlalchemy.org/trac/ticket/4712)

+   **[engine] [bug]**

    调整了方言初始化过程，使得在第一个连接上不会第二次调用 `Dialect.on_connect()`。首先调用钩子，然后如果该连接是该方言的第一个连接，则调用 `Dialect.initialize()`，然后不再调用任何事件。这消除了对“on_connect”函数的两次调用，这可能会产生非常困难的调试情况。

    参考：[#5497](https://www.sqlalchemy.org/trac/ticket/5497)

+   **[engine] [deprecated]**

    `URL` 对象现在是一个不可变的命名元组。要修改 URL 对象，请使用`URL.set()` 方法生成一个新的 URL 对象。

    另请参阅

    URL 对象现在是不可变的 - 迁移说明

    参考：[#5526](https://www.sqlalchemy.org/trac/ticket/5526)

+   **[engine] [已弃用]**

    `MetaData.bind` 参数以及“绑定元数据”的整体概念在 SQLAlchemy 1.4 中已被弃用，并将在 SQLAlchemy 2.0 中移除。当使用 SQLAlchemy 2.0 弃用模式 时，该参数以及相关函数现在会发出`RemovedIn20Warning`。

    另请参阅

    “隐式”和“无连接”执行，移除“绑定元数据”

    参考：[#4634](https://www.sqlalchemy.org/trac/ticket/4634)

+   **[engine] [已弃用]**

    `server_side_cursors` 引擎全局参数已被弃用，并将在将来的版本中移除。对于无缓冲游标，应该在每次执行时使用`Connection.execution_options.stream_results` 执行选项。

+   **[engine] [已弃用]**

    `Connection.connect()` 方法已被弃用，以及“连接分支”概念，它将`Connection` 复制到一个具有无操作“.close()”方法的新连接中。这种模式围绕“无连接执行”概念，该概念也将在 2.0 版本中移除。

    参考：[#5131](https://www.sqlalchemy.org/trac/ticket/5131)

+   **[engine] [已弃用]**

    `create_engine()` 上的`case_sensitive`标志已被弃用；该标志是为了将结果行对象过渡为默认情况下允许区分大小写列匹配，同时为以前的匹配方法提供向后兼容性的一部分。应该假定行的所有字符串访问都是区分大小写的，就像任何其他 Python 映射一样。

    参考：[#4878](https://www.sqlalchemy.org/trac/ticket/4878)

+   **[engine] [已弃用]**

    “隐式自动提交”，即在连接上发出 DML 或 DDL 语句时发生的 COMMIT，已被弃用，并且不会成为 SQLAlchemy 2.0 的一部分。当自动提交生效时，会发出 2.0 风格的警告，以便调用代码可以调整为使用显式事务。

    作为此更改的一部分，当针对`Engine`使用`MetaData.create_all()`等 DDL 方法时，如果尚未启动 BEGIN 块，则将在其中运行操作。

    另请参阅

    SQLAlchemy 2.0 弃用模式

    参考：[#4846](https://www.sqlalchemy.org/trac/ticket/4846)

+   **[engine] [已弃用]**

    弃用了一种行为，即当`Column`作为结果集行查找的键时，当该`Column`不是被选择的 SQL 可选部分时，即仅按名称匹配时。现在针对此情况发出弃用警告。已改进各种 ORM 用例，例如涉及`text()`构造的用例，以避免在大��数情况下使用此回退逻辑。

    参考：[#4877](https://www.sqlalchemy.org/trac/ticket/4877)

+   **[engine] [已弃用]**

    弃用了剩余的引擎级内省和实用方法，包括`Engine.run_callable()`、`Engine.transaction()`、`Engine.table_names()`、`Engine.has_table()`。这些实用方法已被现代上下文管理器模式取代，而表内省任务则适用于`Inspector`对象。

    参考：[#4755](https://www.sqlalchemy.org/trac/ticket/4755)

+   **[engine] [已移除]**

    移除`Dialect`和`Inspector`类中已弃用的方法`get_primary_keys`。请参考`Dialect.get_pk_constraint()`和`Inspector.get_primary_keys()`方法。

    移除已弃用的事件`dbapi_error`和方法`ConnectionEvents.dbapi_error`。请参考`ConnectionEvents.handle_error()`事件。此更改还移除了属性`ExecutionContext.is_disconnect`和`ExecutionContext.exception`。

    参考：[#4643](https://www.sqlalchemy.org/trac/ticket/4643)

+   **[engine] [已移除]**

    内部方言方法`Dialect.reflecttable`已被移除。对第三方方言的审查未发现任何使用此方法的情况，因为它已被记录为不应被外部方言使用的方法之一。此外，还移除了私有方法`Engine._run_visitor`。

    参考：[#4755](https://www.sqlalchemy.org/trac/ticket/4755)

+   **[engine] [已移除]**

    长期弃用的 `Inspector.get_table_names.order_by` 参数已被移除。

    参考：[#4755](https://www.sqlalchemy.org/trac/ticket/4755)

+   **[engine] [renamed]**

    `Inspector.reflecttable()` 已重命名为 `Inspector.reflect_table()`。

    参考：[#5244](https://www.sqlalchemy.org/trac/ticket/5244)

### sql

+   **[sql] [feature]**

    添加了“from linting”作为 SQL 编译器的内置功能。这允许编译器维护特定 SELECT 语句中所有 FROM 子句的图形，这些 FROM 子句由 WHERE 或 JOIN 子句中的条件链接在一起。如果任何两个 FROM 子句之间没有路径，则发出警告，说明查询可能会产生笛卡尔积。由于 Core 表达式语言以及 ORM 建立在“隐式 FROMs”模型上，其中如果查询的任何部分引用了它，则会自动添加特定的 FROM 子句，因此可能会无意中发生这种情况，希望这个新特性能解决这个问题。

    另请参见

    内置的 FROM linting 将在 SELECT 语句中警告任何潜在的笛卡尔积

    参考：[#4737](https://www.sqlalchemy.org/trac/ticket/4737)

+   **[sql] [feature] [mssql] [oracle]**

    添加了新的“编译后参数”功能。此功能允许 `bindparam()` 构造在传递给 DBAPI 驱动程序之前将其值呈现为 SQL 字符串，但在编译步骤之后，使用编译器的“文字渲染”功能。此功能的直接理由是支持不支持或性能较差的由数据库驱动程序处理的绑定参数的 LIMIT/OFFSET 方案，同时仍允许 SQLAlchemy SQL 构造以其编译后的形式进行缓存。新功能的直接目标是 SQL Server（和 Sybase）使用的“TOP N”子句，该子句不支持绑定参数，以及 Oracle 方言使用的“ROWNUM”和可选的“FIRST_ROWS()”方案，前者已知在没有绑定参数的情况下表现更好，后者不支持绑定参数。该功能建立在首次开发以支持 IN 表达式的“展开”参数的机制之上。作为此功能的一部分，Oracle `use_binds_for_limits` 功能被无条件打开，并且此标志现已弃用。

    另请参见

    新的“编译后”绑定参数用于 Oracle、SQL Server 中的 LIMIT/OFFSET

    参考：[#4808](https://www.sqlalchemy.org/trac/ticket/4808)

+   **[sql] [feature]**

    为支持正则表达式在支持的后端上添加支持。已定义了两个操作：

    +   `ColumnOperators.regexp_match()` 实现了类似正则表达式匹配的功能。

    +   `ColumnOperators.regexp_replace()` 实现了正则表达式字符串替换功能。

    支持的后端包括 SQLite、PostgreSQL、MySQL / MariaDB 和 Oracle。

    另请参阅

    支持 SQL 正则表达式操作符

    参考：[#1390](https://www.sqlalchemy.org/trac/ticket/1390)

+   **[sql] [feature]**

    `select()` 构造和相关构造现在允许在列子句中复制列标签和列本身，完全反映了列表达式的传递方式。这样可以使执行结果返回的元组与最初选择的内容匹配，这也是 ORM `Query` 的工作方式，因此这为这两种构造之间的更好交叉兼容性奠定了基础。此外，它还允许对列位置敏感的结构（例如 UNIONs，即 `_selectable.CompoundSelect`）在某些情况下更直观地构建，其中特定列可能出现在多个位置。为了支持这一变化，`ColumnCollection` 已经进行了修订，以支持重复列，并允许整数索引访问。

    另请参阅

    SELECT 对象和派生的 FROM 子句允许重复列和列标签

    参考：[#4753](https://www.sqlalchemy.org/trac/ticket/4753)

+   **[sql] [feature]**

    增强了 `select()` 构造的消除歧义标签功能，使得当一个 select 语句在子查询中使用时，来自不同表的重复列名现在会自动使用唯一的标签名称标记，无需使用完整的“apply_labels()”功能，该功能结合了表名和列名。消除歧义的标签以纯字符串键的形式在子查询的 .c 集合中可用，最重要的是，该功能允许针对实体和任意子查询的组合正确工作的 ORM `aliased()` 构造，针对源表中同名列正确定位列，而无需使用“应用标签”警告。

    另请参阅

    从查询本身作为子查询选择，例如“from_self()” - 说明了新的消歧功能，作为迁移策略的一部分，远离`Query.from_self()`方法。

    参考：[#5221](https://www.sqlalchemy.org/trac/ticket/5221)

+   **[sql] [特性]**

    “扩展 IN”功能在查询执行时生成基于语句执行相关参数的 IN 表达式，现在用于针对字面值列表的所有 IN 表达式。这使得 IN 表达式可以完全独立于传递的值列表进行缓存，并且还包括对空列表的支持。对于 IN 表达式包含非字面 SQL 表达式的任何情况，将保持 IN 中每个位置的预渲染旧行为。此更改还完成了对元组扩展 IN 的支持，以前类型特定的绑定处理器未生效。

    另请参阅

    所有 IN 表达式都会即时为列表中的每个值渲染参数（例如扩展参数）

    参考：[#4645](https://www.sqlalchemy.org/trac/ticket/4645)

+   **[sql] [特性]**

    作为[#4369](https://www.sqlalchemy.org/trac/ticket/4369)的一部分引入的新透明语句缓存功能，增加了一个新功能，旨在减少创建语句的 Python 开销，允许在指示传递给语句对象（如 select()、Query()、update() 等）的参数时使用 lambdas，并允许在 lambdas 中构建完整语句，类似于“烘焙查询”系统。使用 lambdas 的理念源自使用 lambdas 封装任意量的 Python 代码为可调用对象，只需在首次构建语句为字符串时调用。然而，新功能更为复杂，因为会自动提取作为参数传递的 Python 字面值，因此不再需要在此类查询中使用 bindparam() 对象。使用该功能是可选的，可以根据需要使用小或大程度，同时仍然允许语句完全可缓存。

    另请参阅

    使用 Lambdas 为语句生成提速

    参考：[#5380](https://www.sqlalchemy.org/trac/ticket/5380)

+   **[sql] [用例]**

    `Index.create()`和`Index.drop()`方法现在具有一个参数`Index.create.checkfirst`，与`Table`和`Sequence`的参数相同，当启用时，操作将在执行创建或删除操作之前检测索引是否存在（或不存在）。

    参考：[#527](https://www.sqlalchemy.org/trac/ticket/527)

+   **[sql] [用例]**

    `true()`和`false()`运算符现在可以作为不支持“本地布尔”表达式的后端（例如 Oracle 或 SQL Server）上的`join()`的“onclause”应用，表达式将渲染为“1=1”表示 true 和“1=0”表示 false。这是多年前在[#2804](https://www.sqlalchemy.org/trac/ticket/2804)中引入的 and/or 表达式的行为。

+   **[sql] [用例]**

    更改`ColumnCollection`的`__str`方法，避免将其与 Python 字符串列表混淆。

    参考：[#5191](https://www.sqlalchemy.org/trac/ticket/5191)

+   **[sql] [用例]**

    在支持的后端（目前为 PostgreSQL、Oracle 和 MSSQL）的选择语句中添加对`FETCH {FIRST | NEXT} [ count ] {ROW | ROWS} {ONLY | WITH TIES}`的支持。

    参考：[#5576](https://www.sqlalchemy.org/trac/ticket/5576)

+   **[sql] [用例]**

    添加了额外的逻辑，使得某些通常包装单个数据库列的 SQL 表达式在 SELECT 语句中使用该列的名称作为其“匿名标签”名称，从而使结果元组中基于键的查找更直观。主要示例是 CAST 表达式，例如`CAST(table.colname AS INTEGER)`，它将导出其默认名称为“colname”，而不是通常的“anon_1”标签，即`CAST(table.colname AS INTEGER) AS colname`。如果内部表达式没有名称，则使用先前的“匿名标签”逻辑。当使用`Select.apply_labels()`的 SELECT 语句（例如 ORM 发出的那些）时，标记逻辑将产生与仅命名列相同的`<tablename>_<inner column name>`。该逻辑现在适用于`cast()`和`type_coerce()`构造，以及一些单元素布尔表达式。

    另请参见

    使用 CAST 或类似方法改进简单列表达式的列标签

    参考：[#4449](https://www.sqlalchemy.org/trac/ticket/4449)

+   **[sql] [变更]**

    “子句强制转换”系统是 SQLAlchemy Core 的一个系统，用于接收参数并将其解析为`ClauseElement`结构，以构建 SQL 表达式对象，已从一系列特定函数重写为一个完全一致的基于类的系统。这个变更是内部的，对终端用户除了在将错误类型的参数传递给表达式对象时会得到更具体的错误消息外，不会产生任何影响，但这个变更是涉及`select()`对象的角色和行为的一系列变更的一部分。

    参考：[#4617](https://www.sqlalchemy.org/trac/ticket/4617)

+   **[sql] [变更]**

    添加了一个核心的`Values`对象，使得可以在支持的数据库（主要是 PostgreSQL 和 SQL Server）的 SQL 语句的 FROM 子句中使用 VALUES 构造。

    参考：[#4868](https://www.sqlalchemy.org/trac/ticket/4868)

+   **[sql] [变更]**

    `select()` 构造正在向一个新的调用形式迈进，即`select(col1, col2, col3, ..)`，所有其他关键字参数都被移除，因为这些都适用于生成方法。传递给`select()`的列或表参数的单个列表仍然被接受，但是如果表达式以简单的位置样式传递，则不再需要。当使用这种形式时，其他关键字参数是不允许的。

    另请参见

    select(), case() 现在接受位置表达式

    引用：[#5284](https://www.sqlalchemy.org/trac/ticket/5284)

+   **[sql] [change]**

    作为 SQLAlchemy 2.0 迁移项目的一部分，对 `SelectBase` 类层次结构的角色进行了概念上的改变，它是所有“SELECT”语句构造的根，不再直接作为 FROM 子句，也就是说，它们不再直接是 `FromClause` 的子类。对于最终用户来说，这个改变主要意味着在另一个 `select()` 的 FROM 子句中放置任何 `select()` 构造都需要首先将其包装在一个子查询中，历史上是通过使用 `SelectBase.alias()` 方法，现在也可以通过使用 `SelectBase.subquery()` 方法。这通常是必需的，因为一些数据库无论如何都不接受未命名的 SELECT 子查询作为其 FROM 子句中的任何情况。

    另请参见

    SELECT 语句不再隐式地被视为 FROM 子句

    引用：[#4617](https://www.sqlalchemy.org/trac/ticket/4617)

+   **[sql] [change]**

    添加了一个新的核心类 `Subquery`，它取代了创建针对 `SelectBase` 对象的命名子查询时使用的 `Alias`。`Subquery` 的作用方式与 `Alias` 相同，并且由 `SelectBase.subquery()` 方法生成；为了使用方便和向后兼容，`SelectBase.alias()` 方法与这个新方法是同义的。

    另请参见

    SELECT 语句不再隐式地被视为 FROM 子句

    引用：[#4617](https://www.sqlalchemy.org/trac/ticket/4617)

+   **[sql] [性能]**

    对核心和 ORM 内部的全面重组和重构现在允许在 DQL（例如 SELECT）和 DML（例如 INSERT、UPDATE、DELETE）领域内的所有核心和 ORM 语句允许它们的 SQL 编译以及构造结果提取元数据在大多数情况下完全缓存。这实际上提供了一个透明且通用的版本，类似于过去版本中 ORM 提供的“Baked Query”扩展。新功能可以根据最终为给定方言产生的字符串来计算任何给定 SQL 构造的缓存键，从而允许每次生成等效 select()、Query()、insert()、update()或 delete()对象的函数在第一次生成后将该语句缓存。

    该功能会自动启用，但包含一些新的编程范例，可以使用这些范例使缓存更加高效。

    另请参阅

    透明 SQL 编译缓存添加到核心，ORM 中的所有 DQL、DML 语句

    SQL 编译缓存

    参考：[#4639](https://www.sqlalchemy.org/trac/ticket/4639)

+   **[sql] [错误]**

    修复了从 ORM 绑定列（主要是`ForeignKey`对象，但也包括`UniqueConstraint`、`CheckConstraint`等）构造约束时的问题，ORM 级别的`InstrumentedAttribute`被完全丢弃，所有来自列的 ORM 级别的注释都被删除；这样做是为了使约束仍然完全可挑选，而不需要引入 ORM 级别的实体。这些注释不需要在模式/元数据级别存在。

    参考：[#5001](https://www.sqlalchemy.org/trac/ticket/5001)

+   **[sql] [错误]**

    基于`GenericFunction`注册的函数名称现在在所有情况下以不区分大小写的方式检索，移除了 1.3 中的废弃逻辑，该逻辑暂时允许存在具有不同大小写的多个`GenericFunction`对象。一个替换另一个具有相同名称的`GenericFunction`对象无论是否区分大小写，都会在替换对象之前发出警告。

    参考：[#4569](https://www.sqlalchemy.org/trac/ticket/4569), [#4649](https://www.sqlalchemy.org/trac/ticket/4649)

+   **[sql] [bug]**

    创建一个没有参数或空`*args`的`and_()`或`or_()`构造现在将发出弃用警告，因为生成的 SQL 是一个空操作（即呈现为空字符串）。这种行为被认为是不直观的，因此对于空或可能为空的`and_()`或`or_()`构造，应包含适当的默认布尔值，例如`and_(True, *args)`或`or_(False, *args)`。就像在许多主要版本的 SQLAlchemy 中一样，如果`*args`部分不为空，这些特定的布尔值将不会呈现。

    参考：[#5054](https://www.sqlalchemy.org/trac/ticket/5054)

+   **[sql] [bug]**

    改进了`tuple_()`构造，使其在列子句上下文中的行为更加可预测。大多数后端不支持将 SQL 元组作为“SELECT”列子句元素；在支持的后端（毫不奇怪的是，如 PostgreSQL）上，Python DBAPI 没有“嵌套类型”概念，因此在获取此类对象的行时仍然存在挑战。在`select()`或`Query`中使用`tuple_()`现在会在看到`tuple_()`对象准备获取行时引发`CompileError`（即，如果元组在子查询的列子句中，不会引发错误）。对于 ORM 使用，`Bundle`对象是一个明确的指令，表示应将一系列列作为子元组返回，并由错误消息建议。此外，在所有上下文中，元组现在将用括号呈现。以前，在列上下文中，括号化不会呈现，导致行为未定义。

    参考：[#5127](https://www.sqlalchemy.org/trac/ticket/5127)

+   **[sql] [bug] [postgresql]**

    改进了对包含百分号的列名的支持，修复了涉及匿名标签的问题，这些问题还嵌入了一个包含百分号的列名，以及重新建立了对在 psycopg2 方言上嵌入百分号的绑定参数名的支持，使用了类似于 cx_Oracle 方言使用的延迟转义过程。

    参考：[#5653](https://www.sqlalchemy.org/trac/ticket/5653)

+   **[sql] [bug]**

    作为`FunctionElement`的子类创建的自定义函数现在将基于函数的“名称”生成“匿名标签”，就像任何其他`Function`对象一样，例如`"SELECT myfunc() AS myfunc_1"`。虽然 SELECT 语句不再需要标签才能使结果代理对象正常工作，但 ORM 仍然通过使用对象作为映射键来定位行中的列，当列表达式具有不同的名称时，这种方法更可靠。无论如何，现在通过`func`生成的函数和作为自定义`FunctionElement`对象生成的函数之间的行为都是一致的。

    参考：[#4887](https://www.sqlalchemy.org/trac/ticket/4887)

+   **[sql] [bug]**

    重新设计了基于新的基于访问者的方法的`ClauseElement.compare()`方法，并另外添加了测试覆盖，确保所有`ClauseElement`子类可以在结构上准确地相互比较。结构比较功能目前在 ORM 中仅被轻微使用，但它也可能成为新缓存功能的基础。

    参考：[#4336](https://www.sqlalchemy.org/trac/ticket/4336)

+   **[sql] [bug]**

    弃用除 PostgreSQL 以外的方言中的`DISTINCT ON`的用法。弃用 MySQL 方言中旧的字符串 distinct 用法

    参考：[#4002](https://www.sqlalchemy.org/trac/ticket/4002)

+   **[sql] [bug]**

    `_selectable.CompoundSelect`的`ORDER BY`子句，例如 UNION，EXCEPT 等，在应用`CompoundSelect.order_by()`时不会呈现与给定列相关联的表名，这是针对`Table` -绑定列的。大多数数据库要求`ORDER BY`子句中的名称仅表示为标签名称，这些名称与第一个 SELECT 语句中的名称匹配。此更改与[#4617](https://www.sqlalchemy.org/trac/ticket/4617)相关，以前的解决方法是引用`_selectable.CompoundSelect`的`.c`属性，以便访问没有表名的列。由于子查询现在有名称，此更改允许继续使用解决方法，并且允许表绑定列以及`CompoundSelect.selected_columns`集合在`CompoundSelect.order_by()`方法中可用。

    参考：[#4617](https://www.sqlalchemy.org/trac/ticket/4617)

+   **[sql] [bug]**

    `Join`构造不再将“onclause”视为要从`Select`对象的 FROM 列表中省略的其他 FROM 对象的来源。这适用于包含对 JOIN 外部另一个 FROM 对象的引用的 ON 子句；虽然从 SQL 的角度来看，这通常是不正确的，但省略它也是不正确的，行为上的更改使得`Select` / `Join`的行为更加直观。

    参考：[#4621](https://www.sqlalchemy.org/trac/ticket/4621)

+   **[sql] [deprecated]**

    `Join.alias()`方法已弃用，并将在 SQLAlchemy 2.0 中移除。应改为使用显式的 select +子查询，或内部表的别名。

    参考：[#5010](https://www.sqlalchemy.org/trac/ticket/5010)

+   **[sql] [deprecated]**

    当定义具有相同名称的列时，`Table`类现在会引发弃用警告。为了替换列，`Table.append_column()`方法添加了一个新参数`Table.append_column.replace_existing`。

    当使用字符串调用`ColumnCollection.contains_column()`时，现在会引发错误，建议调用者改用`in`。

+   **[sql] [已移除]**

    在 1.3 版中废弃的 “threadlocal” 执行策略已在 1.4 版中移除，以及 “engine 策略” 的概念和 `Engine.contextual_connect` 方法。目前仍然接受 “strategy=’mock’” 关键字参数，并附有废弃警告；对于这种用例，请改用 `create_mock_engine()`。

    请参阅

    “threadlocal” 引擎策略已废弃 - 1.3 版迁移说明中讨论了废弃的原因。

    参考资料：[#4632](https://www.sqlalchemy.org/trac/ticket/4632)

+   **[sql] [已移除]**

    移除了 `sqlalchemy.sql.visitors.iterate_depthfirst` 和 `sqlalchemy.sql.visitors.traverse_depthfirst` 函数。这些函数未被 SQLAlchemy 的任何部分使用。`iterate()` 和 `traverse()` 函数通常用于这些功能。同时还移除了剩余函数中未使用的选项，包括“column_collections”、“schema_visitor”。

+   **[sql] [已移除]**

    从 `Compiler` 对象中移除了绑定引擎的概念，并从 `Compiler` 中移除了 `.execute()` 和 `.scalar()` 方法。这些实际上是十多年前被遗忘的方法，并且没有实际用途，`Compiler` 对象本身不应该维护对 `Engine` 的引用。

+   **[sql] [已移除]**

    移除了废弃的方法 `Compiled.compile`、`ClauseElement.__and__` 和 `ClauseElement.__or__` 以及属性 `Over.func`。

    移除了废弃的 `FromClause.count` 方法。请使用 `func` 命名空间中可用的 `count` 函数。

    参考资料：[#4643](https://www.sqlalchemy.org/trac/ticket/4643)

+   **[sql] [已移除]**

    移除了废弃的参数 `text.bindparams` 和 `text.typemap`。请参考 `TextClause.bindparams()` 和 `TextClause.columns()` 方法。

    移除了废弃的参数 `Table.useexisting`。请使用 `Table.extend_existing`。

    参考资料：[#4643](https://www.sqlalchemy.org/trac/ticket/4643)

+   **[sql] [已重命名]**

    `Table` 参数 `mustexist` 已更名为 `Table.must_exist`，现在在使用时会发出警告。

+   **[sql] [重命名]**

    `SelectBase.as_scalar()` 和 `Query.as_scalar()` 方法分别更名为 `SelectBase.scalar_subquery()` 和 `Query.scalar_subquery()`。旧名称在 1.4 系列中仍然存在，并带有弃用警告。此外，在列上下文中评估时，对 `SelectBase`、`Alias` 和其他 SELECT 导向对象进行标量子查询的隐式强制转换也已弃用，并发出警告，建议显式调用 `SelectBase.scalar_subquery()` 方法。这个警告在以后的主要版本中将变为错误，但当需要调用 `SelectBase.scalar_subquery()` 时，消息始终是清晰的。这个变化的后半部分是为了清晰起见，并减少查询强制转换系统的隐式决策。`Subquery.as_scalar()` 方法，以前是 `Alias.as_scalar`，也已弃用；应直接从 `select()` 对象或 `Query` 对象调用 `.scalar_subquery()`。

    这个变化是将 `select()` 对象转换为不再直接属于“from clause”类层次结构的一部分的更大变化的一部分，这也包括对子句强制转换系统的彻底改革。

    参考：[#4617](https://www.sqlalchemy.org/trac/ticket/4617)

+   **[sql] [重命名]**

    为了实现更一致的命名，几个运算符已被重命名。

    运算符的更改包括：

    +   `isfalse` 现在是 `is_false`

    +   `isnot_distinct_from` 现在是 `is_not_distinct_from`

    +   `istrue` 现在是 `is_true`

    +   `notbetween` 现在是 `not_between`

    +   `notcontains` 现在是 `not_contains`

    +   `notendswith` 现在是 `not_endswith`

    +   `notilike` 现在是 `not_ilike`

    +   `notlike` 现在是 `not_like`

    +   `notmatch` 现在是 `not_match`

    +   `notstartswith` 现在是 `not_startswith`

    +   `nullsfirst` 现在是 `nulls_first`

    +   `nullslast` 现在是 `nulls_last`

    +   `isnot` 现在是 `is_not`

    +   `notin_` 现在是 `not_in`

    由于这些是核心操作符，此更改的内部迁移策略是支持传统术语一段时间（如果不是无限期）但更新所有文档、教程和内部用法到新术语。新术语用于定义函数，而传统术语已被弃用为新术语的别名。

    参考：[#5429](https://www.sqlalchemy.org/trac/ticket/5429), [#5435](https://www.sqlalchemy.org/trac/ticket/5435)

+   **[sql] [postgresql]**

    在 PostgreSQL 中创建 `Sequence` 时允许指定数据类型，使用参数 `Sequence.data_type`。

    参考：[#5498](https://www.sqlalchemy.org/trac/ticket/5498)

+   **[sql] [反射]**

    外键“ON UPDATE”的“NO ACTION”关键字现在被视为所有支持后端（SQlite、MySQL、PostgreSQL）上外键的默认级联，当检测到时不包含在反射字典中；这已经是 PostgreSQL 和 MySQL 在任何情况下所有以前的 SQLAlchemy 版本的行为。当检测到时，“RESTRICT”关键字被积极存储；PostgreSQL 确实报告此关键字，而 MySQL 从版本 8.0 开始也是如此。在较早的 MySQL 版本中，数据库不会报告此关键字。

    参考：[#4741](https://www.sqlalchemy.org/trac/ticket/4741)

+   **[sql] [反射]**

    增加了对“identity”列的反射支持，这些列现在作为 `Inspector.get_columns()` 返回的结构的一部分。在反射完整的 `Table` 对象时，identity 列将使用 `Identity` 结构表示。目前支持的后端是 PostgreSQL >= 10、Oracle >= 12 和 MSSQL（具有不同语法和功能子集）。

    参考：[#5324](https://www.sqlalchemy.org/trac/ticket/5324), [#5527](https://www.sqlalchemy.org/trac/ticket/5527)

### 模式

+   **[模式] [更改]**

    `Enum.create_constraint`和`Boolean.create_constraint`参数现在默认为 False，表示当创建这两种数据类型的所谓“非本地”版本时，默认不会生成 CHECK 约束。这些 CHECK 约束会带来模式管理维护复杂性，应该选择性地使用，而不是默认开启。

    另请参阅

    枚举和布尔数据类型不再默认为“创建约束”

    参考：[#5367](https://www.sqlalchemy.org/trac/ticket/5367)

+   **[模式] [错误]**

    清理了数据类型的内部`str()`，使得所有类型都能产生一个没有任何方言的字符串表示，包括第三方方言类型也能正常工作。字符串表示默认为该类型的大写名称，没有其他内容。

    参考：[#4262](https://www.sqlalchemy.org/trac/ticket/4262)

+   **[模式] [已移除]**

    移除了废弃的类`Binary`。请使用`LargeBinary`。

    参考：[#4643](https://www.sqlalchemy.org/trac/ticket/4643)

+   **[模式] [重命名]**

    将`Table.tometadata()`方法重命名为`Table.to_metadata()`。之前的名称仍然存在，但会有弃用警告。

    参考：[#5413](https://www.sqlalchemy.org/trac/ticket/5413)

+   **[模式] [SQL]**

    添加了`Identity`构造，可用于配置生成的标识列为 GENERATED { ALWAYS | BY DEFAULT } AS IDENTITY。目前支持的后端是 PostgreSQL >= 10、Oracle >= 12 和 MSSQL（具有不同的语法和功能子集）。

    参考：[#5324](https://www.sqlalchemy.org/trac/ticket/5324), [#5360](https://www.sqlalchemy.org/trac/ticket/5360), [#5362](https://www.sqlalchemy.org/trac/ticket/5362)

### 扩展

+   **[扩展] [用例]**

    使用`sqlalchemy.ext.compiled`扩展创建的自定义编译器构造，在自定义构造被解释为 SELECT 语句中列子句的元素时，将自动向编译器添加上下文信息，使得自定义元素可以作为结果行映射中的键进行定位，这是 ORM 用于将列元素匹配到结果元组中的定位方式。

    参考：[#4887](https://www.sqlalchemy.org/trac/ticket/4887)

+   **[扩展] [更改]**

    添加了新参数 `AutomapBase.prepare.autoload_with`，它取代了 `AutomapBase.prepare.reflect` 和 `AutomapBase.prepare.engine`。

    参考：[#5142](https://www.sqlalchemy.org/trac/ticket/5142)

### postgresql

+   **[postgresql] [用例]**

    为所有 psycopg2、asyncpg 和 pg8000 方言添加了对 PostgreSQL “readonly” 和 “deferrable” 标志的支持。这利用了一种新的广义化的“隔离级别”API 版本，以支持通过执行选项设置的其他类型的会话属性，在连接返回到连接池时可靠地重置。

    另请参阅

    设置 READ ONLY / DEFERRABLE

    参考：[#5549](https://www.sqlalchemy.org/trac/ticket/5549)

+   **[postgresql] [用例]**

    `BufferedRowResultProxy` 的最大缓冲区大小，当 `stream_results=True` 时由 PostgreSQL 等方言使用，现在可以设置为大于 1000 的数字，并且缓冲区将增长到该大小。之前，即使值设置得更大，缓冲区也不会超过 1000。缓冲区的增长现在也基于一个简单的乘法因子，目前设置为 5。感谢 Soumaya Mauthoor 提供的拉取请求。

    参考：[#4914](https://www.sqlalchemy.org/trac/ticket/4914)

+   **[postgresql] [更改]**

    当使用 psycopg2 方言进行 PostgreSQL 时，psycopg2 的最低版本设置为 2.7。psycopg2 方言依赖于过去几年中发布的许多 psycopg2 特性，因此为了简化方言，现在要求的最低版本是 2017 年 3 月发布的版本 2.7。

+   **[postgresql] [性能]**

    psycopg2 方言现在默认使用非常高效的 `execute_values()` psycopg2 扩展来编译 INSERT 语句，并且当使用该扩展时，还实现了 RETURNING 支持。这使得即使包含自增的 SERIAL 或 IDENTITY 值的 INSERT 语句也能运行得非常快，同时仍然能够返回新生成的主键值。ORM 将在另一个单独的更改中集成这个新特性。

    另请参阅

    psycopg2 方言特性默认使用“execute_values”和 RETURNING 用于 INSERT 语句 - 关于 `executemany_mode` 参数的所有更改的完整列表。

    参考：[#5401](https://www.sqlalchemy.org/trac/ticket/5401)

+   **[postgresql] [错误]**

    pg8000 方言已经针对最新版本的 pg8000 驱动程序进行了修订和现代化，用于 PostgreSQL。拉取请求由 Tony Locke 提供。请注意，这必然将 pg8000 固定在 1.16.6 或更高版本，不再支持 Python 2。需要 pg8000 的 Python 2 用户应确保他们的要求固定在`SQLAlchemy<1.4`。

+   **[postgresql] [已弃用]**

    pygresql 和 py-postgresql 方言已弃用。

    参考：[#5189](https://www.sqlalchemy.org/trac/ticket/5189)

+   **[postgresql] [已移除]**

    移除了对形式为`postgres://`的已弃用引擎 URL 的支持；这已经发出警告多年，项目应该使用`postgresql://`。

    参考：[#4643](https://www.sqlalchemy.org/trac/ticket/4643)

### mysql

+   **[mysql] [功能]**

    增加了对 MariaDB Connector/Python 在 mysql 方言中的支持。原始拉取请求由 Georg Richter 提供。

    参考：[#5459](https://www.sqlalchemy.org/trac/ticket/5459)

+   **[mysql] [用例]**

    增加了一个新的方言标记“mariadb”，可以在`create_engine()` URL 中代替“mysql”使用。这将提供一个 MariaDB 方言的 MySQLDialect 子类，强制“is_mariadb”标志为 True。如果接收到不指示使用 MariaDB 的服务器版本字符串，该方言将引发错误。这对于 MariaDB 特定的测试场景以及支持硬编码到 MariaDB 专用概念的应用程序非常有用。随着 MariaDB 和 MySQL 的功能集和使用模式继续发散，这种模式可能变得更加突出。

    参考：[#5496](https://www.sqlalchemy.org/trac/ticket/5496)

+   **[mysql] [用例]**

    增加了对 MariaDB 10.3 及更高版本使用`Sequence`构造的支持，因为这个数据库现在支持这个构造。该构造与`Table`对象集成方式与其他数据库（如 PostgreSQL 和 Oracle）相同；如果存在于整数主键“自增”列上，它将用于生成默认值。为了向后兼容，支持在`Table`上有一个`Sequence`以支持仅支持序列的数据库（如 Oracle），同时仍然不让序列在 MariaDB 上触发，应该设置 optional=True 标志，这表示序列应仅用于生成主键，如果目标数据库没有其他选项。

    另请参阅

    为 MariaDB 10.3 添加了序列支持

    参考：[#4976](https://www.sqlalchemy.org/trac/ticket/4976)

+   **[mysql] [错误]**

    MySQL 和 MariaDB 方言现在查询 information_schema.tables 系统视图，以确定特定表是否存在。之前，使用“DESCRIBE”命令，并带有异常捕获来检测不存在的表，这将导致在连接上发出 ROLLBACK。似乎存在遗留的编码问题，导致无法使用“SHOW TABLES”，但由于[#4189](https://www.sqlalchemy.org/trac/ticket/4189)，MySQL 支持现在已经达到 5.0.2 或更高版本，在所有情况下现在都可以使用 information_schema 表。

+   **[mysql] [错误]**

    使用`with_for_update()`与“skip_locked”关键字将在所有 MySQL 后端上生成“SKIP LOCKED”，这意味着在 MySQL 8 之前的版本以及当前的 MariaDB 后端上将失败。这是因为这些后端不支持“SKIP LOCKED”或任何等价物，因此不应该默默地忽略此错误。在 1.3 系列中，这已经升级为警告。

    引用：[#5568](https://www.sqlalchemy.org/trac/ticket/5568)

+   **[mysql] [错误]**

    MySQL 方言的 server_version_info 元组现在全部是数字。不再出现像“MariaDB”这样的字符串标记，以便在所有情况下都能进行数字比较。应该咨询 dialect 上的.is_mariadb 标志，以确定是否检测到了 mariadb。另外，删除了用于支持极老的 MySQL 版本 3.x 和 4.x 的结构；现在支持的最低 MySQL 版本是版本 5.0.2。

    引用：[#4189](https://www.sqlalchemy.org/trac/ticket/4189)

+   **[mysql] [已弃用]**

    OurSQL 方言已弃用。

    引用：[#5189](https://www.sqlalchemy.org/trac/ticket/5189)

+   **[mysql] [已移除]**

    删除了自版本 1.0 起已弃用的 dialect `mysql+gaerdbms`。直接使用 MySQLdb 方言。

    在`mysql`方言中从`ENUM`和`SET`中移除了已弃用的参数`quoting`。当需要时，SQLAlchemy 会自动为枚举或集合中传递的值添加引号。

    引用：[#4643](https://www.sqlalchemy.org/trac/ticket/4643)

### sqlite

+   **[sqlite] [更改]**

    放弃对右嵌套连接重写的支持，以支持 2013 年发布的旧 SQLite 版本 3.7.16 之前的版本。预计现在支持的所有现代 Python 版本都应包含更新的 SQLite 版本。

    另请参阅

    从 SQLite 方言中删除了“join rewriting”逻辑；更新了导入

    引用：[#4895](https://www.sqlalchemy.org/trac/ticket/4895)

### mssql

+   **[mssql] [特性] [sql]**

    在 SQL Server 方言上添加了对`JSON`数据类型的支持，使用了`JSON`实现，该实现根据 SQL Server 文档针对`NVARCHAR(max)`数据类型实现了 SQL Server 的 JSON 功能。实现由 Gord Thompson 提供。

    参考：[#4384](https://www.sqlalchemy.org/trac/ticket/4384)

+   **[mssql] [功能]**

    为 Microsoft SQL Server 添加了“CREATE SEQUENCE”和完整的`Sequence`支持。这消除了使用`Sequence`对象来操作 IDENTITY 特性的已弃用功能，现在应该使用`mssql_identity_start`和`mssql_identity_increment`来执行，如自增行为 / IDENTITY 列文档中所述。此更改包括一个新参数`Sequence.data_type`，以适应 SQL Server 的数据类型选择，对于该后端，包括 INTEGER、BIGINT 和 DECIMAL(n, 0)。SQL Server 版本的`Sequence`的默认起始值已设置为 1；这个默认值现在在所有后端的 CREATE SEQUENCE DDL 中被输出。

    另请参阅

    为 SQL Server 添加了与 IDENTITY 不同的 Sequence 支持

    参考：[#4235](https://www.sqlalchemy.org/trac/ticket/4235), [#4633](https://www.sqlalchemy.org/trac/ticket/4633)

+   **[mssql] [用例] [postgresql] [反射] [模式]**

    改进了对覆盖索引（包含 INCLUDE 列）的支持。增加了 postgresql 从 Core 渲染带有 INCLUDE 子句的 CREATE INDEX 语句的能力。索引反射还分别报告了 mssql 和 postgresql（11+）的 INCLUDE 列。

    参考：[#4458](https://www.sqlalchemy.org/trac/ticket/4458)

+   **[mssql] [用例] [postgresql]**

    添加了对部分索引 / 过滤索引的检查 / 反射支持，即使用`mssql_where`或`postgresql_where`参数的索引，使用`Index`。该条目既是由`Inspector.get_indexes()`返回的字典的一部分，也是被反射的`Index`构造的一部分。拉取请求由 Ramon Williams 提供。

    参考：[#4966](https://www.sqlalchemy.org/trac/ticket/4966)

+   **[mssql] [用例] [反射]**

    增加了对 SQL Server 方言临时表的反射支持。现在可以从 MSSQL“tempdb”系统目录中自省以井号“#”为前缀的表名。

    参考：[#5506](https://www.sqlalchemy.org/trac/ticket/5506)

+   **[mssql] [更改]**

    SQL Server OFFSET 和 FETCH 关键字现在用于限制/偏移，而不是使用窗口函数，适用于 SQL Server 版本 11 及更高版本。仅在查询中仅包含 LIMIT 时仍然使用 TOP。拉取请求由 Elkin 提供。

    参考：[#5084](https://www.sqlalchemy.org/trac/ticket/5084)

+   **[mssql] [错误] [模式]**

    修复了`sqlalchemy.engine.reflection.has_table()`始终对临时表返回`False`的问题。

    参考：[#5597](https://www.sqlalchemy.org/trac/ticket/5597)

+   **[mssql] [错误]**

    将`DATETIMEOFFSET`数据类型的基类修复为基于`DateTime`类层次结构，因为这是一个持有日期时间的数据类型。

    参考：[#4980](https://www.sqlalchemy.org/trac/ticket/4980)

+   **[mssql] [已弃用]**

    adodbapi 和 mxODBC 方言已被弃用。

    参考：[#5189](https://www.sqlalchemy.org/trac/ticket/5189)

+   **[mssql]**

    mssql 方言将假定至少使用 MSSQL 2005。如果检测到旧版本，不会引发硬异常，但对于旧版本可能会导致操作失败。

+   **[mssql] [反射]**

    作为支持反射`Identity`对象的一部分，方法`Inspector.get_columns()`不再将`mssql_identity_start`和`mssql_identity_increment`作为`dialect_options`的一部分返回。请使用`identity`键中的信息。

    参考：[#5527](https://www.sqlalchemy.org/trac/ticket/5527)

+   **[mssql] [引擎]**

    弃用了`legacy_schema_aliasing`参数`sqlalchemy.create_engine()`。这是一个长期过时的参数，自 1.1 版本以来默认为 False。

    参考：[#4809](https://www.sqlalchemy.org/trac/ticket/4809)

### oracle

+   **[oracle] [用例]**

    Oracle 方言的 max_identifier_length 现在默认为 128 个字符，除非兼容版本小于 12.2 在首次连接时，此时使用 30 个字符的传统长度。这是 1.3 系列中提交的问题的延续，该问题在首次连接时添加了最大标识符长度检测，并警告 Oracle 服务器的更改。

    另请参阅

    最大标识符长度 - 在 Oracle 方言文档中

    参考：[#4857](https://www.sqlalchemy.org/trac/ticket/4857)

+   **[oracle] [change]**

    在 Oracle 中使用的 LIMIT / OFFSET 方案现在使用命名子查询而不是未命名子查询，当它透明地重写 SELECT 语句以使用包含 ROWNUM 的子查询时。这一变化是一个更大变化的一部分，其中未命名子查询不再直接受 Core 支持，以及在 Oracle 方言内部使用 select()构造的现代化。

+   **[oracle] [bug]**

    在 Oracle 数据库上正确渲染`Sequence`和`Identity`列选项`nominvalue`和`nomaxvalue`为`NOMAXVALUE`和`NOMINVALUE`。

+   **[oracle] [bug]**

    Oracle 方言的`INTERVAL`类现在正确地是`Interval`的抽象版本的子类，以及正确的“模拟”基类，这允许在本地模式和非本地模式下正确行为；以前它只基于`TypeEngine`。

    参考：[#4971](https://www.sqlalchemy.org/trac/ticket/4971)

### 杂项

+   **[deprecated] [firebird]**

    Firebird 方言已被弃用，因为现在有第三方方言支持这个数据库。

    参考：[#5189](https://www.sqlalchemy.org/trac/ticket/5189)

+   **[misc] [deprecated]**

    Sybase 方言已被弃用。

    参考：[#5189](https://www.sqlalchemy.org/trac/ticket/5189)
