# 0.7 更新日志

> 原文：[`docs.sqlalchemy.org/en/20/changelog/changelog_07.html`](https://docs.sqlalchemy.org/en/20/changelog/changelog_07.html)

## 0.7.11

没有发布日期

### orm

+   **[orm] [bug]**

    修复了一个错误，列表操作会无法正确表示 `[0:0]` 的切片，尤其是在使用关联代理时可能发生。由于 Python 集合的一些怪癖，该问题在 Python 3 中更有可能发生而不是在 Python 2 中。

    参考：[#2807](https://www.sqlalchemy.org/trac/ticket/2807)

+   **[orm] [bug]**

    修复了一个错误，当查询形式为：`query(SubClass).options(subqueryload(Baseclass.attrname))`，其中 `SubClass` 是 `BaseClass` 的联接继承时，会在属性加载时未能在子查询中应用 `JOIN`，从而产生笛卡尔积。生成的结果仍然往往是正确的，因为额外的行只是被忽略，所以这个问题可能存在于其它方面正常工作的应用中作为性能下降。

    参考：[#2699](https://www.sqlalchemy.org/trac/ticket/2699)

+   **[orm] [bug]**

    修复了工作单元中的错误，即如果两个表之间没有设置 ForeignKey 约束，则联接继承子类可能会在父表之前插入“子”表的行。

    参考：[#2689](https://www.sqlalchemy.org/trac/ticket/2689)

+   **[orm] [bug]**

    改进了当检测到“backref 循环”时发出的错误消息，即当属性事件触发两个其他属性之间的双向赋值时。这种情况不仅会在分配错误类型的对象时发生，还会在属性被错误配置为 backref 到现有 backref 对时发生。

    参考：[#2674](https://www.sqlalchemy.org/trac/ticket/2674)

+   **[orm] [bug]**

    当将 MapperProperty 分配给替换现有属性的映射器时，如果涉及的属性不是基于普通列的属性，则会发出警告。很少（或者从来没有？）有人打算替换关系属性，通常是指映射器错误配置。如果在继承关系中 backref 配置在现有的 backref 上（在 0.8 中是错误的），也会发出警告。

    参考：[#2674](https://www.sqlalchemy.org/trac/ticket/2674)

### engine

+   **[engine] [bug]**

    `make_url()` 函数使用的正则表达式现在可以解析 ipv6 地址，例如用方括号括起来。

    参考：[#2851](https://www.sqlalchemy.org/trac/ticket/2851)

### sql

+   **[sql] [bug]**

    修复了自 0.7.9 起存在的回归，即如果在多个 FROM 子句中引用了 CTE 的名称，则可能未正确引用该名称。

    参考：[#2801](https://www.sqlalchemy.org/trac/ticket/2801)

+   **[sql] [bug] [cte]**

    修复了公共表达式系统中的错误，如果 CTE 仅用作 `alias()` 构造，则不会使用 WITH 关键字进行渲染。

    参考：[#2783](https://www.sqlalchemy.org/trac/ticket/2783)

+   **[sql] [bug]**

    修复了`CheckConstraint` DDL 中的 bug，其中来自`Column` 对象的“quote”标志不会传播。

    参考：[#2784](https://www.sqlalchemy.org/trac/ticket/2784)

### postgresql

+   **[postgresql] [feature]**

    增加了对 PostgreSQL 传统 SUBSTRING 函数语法的支持，当使用常规的`func.substring()`时，呈现为“SUBSTRING(x FROM y FOR z)”。感谢 Gunnlaugur Þór Briem。

    参考：[#2676](https://www.sqlalchemy.org/trac/ticket/2676)

### mysql

+   **[mysql] [bug]**

    更新 MySQL 5.5、5.6 版本的保留字，感谢 Hanno Schlichting。

    参考：[#2791](https://www.sqlalchemy.org/trac/ticket/2791)

### 测试

+   **[tests] [bug]**

    修复了在某些 Linux 平台上无法在 test_execute 中导入“logging”的问题。

    参考：[#2669](https://www.sqlalchemy.org/trac/ticket/2669)，[pull request 41](https://github.com/sqlalchemy/sqlalchemy/pull/41)

## 0.7.10

发布日期：2013 年 2 月 7 日 星期四

### orm

+   **[orm] [bug]**

    修复了潜在的内存泄漏问题，如果创建了任意数量的`sessionmaker`对象。当解除引用时，sessionmaker 创建的匿名子类由于事件包中保留的类级引用而无法被垃圾回收。这个问题也适用于任何与事件调度程序一起使用临时子类的自定义系统。

    参考：[#2650](https://www.sqlalchemy.org/trac/ticket/2650)

+   **[orm] [bug]**

    `Query.merge_result()` 现在可以从外连接加载行，其中一个实体可能为`None`而不会抛出错误。

    参考：[#2640](https://www.sqlalchemy.org/trac/ticket/2640)

+   **[orm] [bug]**

    `MutableComposite` 类型不允许使用`MutableBase.coerce()` 方法，尽管代码似乎表明了这个意图，所以现在可以使用，并添加了一个简短的示例。作为副作用，此事件处理程序的机制已更改，以便新的`MutableComposite` 类型不再添加每种类型的全局事件处理程序。也适用于 0.8.0b2 版本。

    参考：[#2624](https://www.sqlalchemy.org/trac/ticket/2624)

+   **[orm] [bug]**

    修复了 Session 计算错误的 bug，替换已删除对象在身份映射中的另一个具有相同主键的对象会在 rollback()时引发“冲突状态”错误，如果被替换的主键是通过非工作单位建立的 INSERT 语句或通过另一个实例的主键切换来建立的。

    参考：[#2583](https://www.sqlalchemy.org/trac/ticket/2583)

### engine

+   **[engine] [bug]**

    修复了`MetaData.reflect()`在正确使用给定的`Connection`时不会打开第二个连接的错误，如果给定的话。

    参考：[#2604](https://www.sqlalchemy.org/trac/ticket/2604)

### sql

+   **[sql] [bug]**

    将`__repr__`调整为`TypeDecorator`的调整反向移植到 0.7，允许`PickleType`生成干净的`repr()`以帮助 Alembic。

    参考：[#2584](https://www.sqlalchemy.org/trac/ticket/2584), [#2594](https://www.sqlalchemy.org/trac/ticket/2594)

+   **[sql] [bug]**

    修复了`Table.tometadata()`的错误，如果`Column`同时具有外键和替代的“.key”名称，则会失败。

    参考：[#2643](https://www.sqlalchemy.org/trac/ticket/2643)

+   **[sql] [bug]**

    修复了在没有传递“for_update=True”标志的情况下使用 server_onupdate=<FetchedValue|DefaultClause>时会将默认对象应用于 server_default 的错误，从而清除了那里的任何内容。不应该使用显式的 for_update=True 参数来进行此使用（特别是因为文档显示了一个不使用该参数的示例），因此现在如果该标志未设置为对应于该参数的内容，则会内部使用给定默认对象的副本。

    参考：[#2631](https://www.sqlalchemy.org/trac/ticket/2631)

+   **[sql] [gae] [mysql]**

    在`gaerdbms`方言中添加了条件导入，尝试导入 rdbms_apiproxy 和 rdbms_googleapi 以在开发和生产平台上工作。现在还支持`instance`属性。感谢 Sean Lynch 的贡献。还将增强功能反向移植，允许用户名/密码，并修复了从 0.8 开始的错误代码解释。

    参考：[#2649](https://www.sqlalchemy.org/trac/ticket/2649)

### mysql

+   **[mysql] [feature]**

    在 OurSQL 方言中添加了“raise_on_warnings”标志。

    参考：[#2523](https://www.sqlalchemy.org/trac/ticket/2523)

+   **[mysql] [feature]**

    在 MySQLdb 方言中添加了“read_timeout”标志。

    参考：[#2554](https://www.sqlalchemy.org/trac/ticket/2554)

### sqlite

+   **[sqlite] [bug]**

    针对这个与 SQLite 相关的问题作了更多调整，该问题已在 0.7.9 中发布，以拦截反映外键时的旧 SQLite 引用字符。除了拦截双引号之外，现在还拦截其他引用字符，如括号、反引号和单引号。

    参考：[#2568](https://www.sqlalchemy.org/trac/ticket/2568)

### mssql

+   **[mssql] [bug]**

    修复了一个错误，即在与“schema”结合使用 Column 的“key”时，对于拥有表的“schema”会由于 MSSQL 方言的“schema 渲染”逻辑未考虑 .key 而无法定位结果行的错误。

+   **[mssql] [bug]**

    在 mssql 信息模式中，增加了 Py3K 条件，围绕不必要的 .decode() 调用，修复了 Py3k 中反射的问题。

    参考：[#2638](https://www.sqlalchemy.org/trac/ticket/2638)

### oracle

+   **[oracle] [bug]**

    Oracle LONG 类型，虽然是一个无界文本类型，但在返回结果行时似乎不使用 cx_Oracle.LOB 类型，因此方言已修复，以排除 LONG 从中获得 cx_Oracle.LOB 过滤的情况。

    参考：[#2620](https://www.sqlalchemy.org/trac/ticket/2620)

+   **[oracle] [bug]**

    修复了在与 cx_Oracle 结合使用 `.prepare()` 时的用法，以便当返回值为 `False` 时，不会调用 `connection.commit()`，从而避免“无事务”错误。已经证明了 SQLAlchemy 和 cx_oracle 与两阶段事务一起以基本方式工作，但受到与驱动程序观察到的警告的限制；请查看文档了解详情。

    参考：[#2611](https://www.sqlalchemy.org/trac/ticket/2611)

+   **[oracle] [bug]**

    更改了从 setinputsizes() 步骤中排除的 cx_oracle 类型列表，现在只包括 STRING 和 UNICODE；CLOB 和 NCLOB 已移除。这是为了解决 cx_oracle 对 executemany() 调用的行为问题。在 0.8 中，相同的更改也适用，但也可以通过 exclude_setinputsizes 参数进行配置。

    参考：[#2561](https://www.sqlalchemy.org/trac/ticket/2561)

## 0.7.9

发布日期：Mon Oct 01 2012

### orm

+   **[orm] [bug]**

    修复了一个主要局限于新 AbstractConcreteBase 助手的错误，其中来自超类的“type”属性不会被子类覆盖，从而产生“保留给基类”的错误消息，而是在那里放置一个无操作属性。这与使用 ConcreteBase 以及所有经典的具体映射行为不一致，在这些行为中，多态基类的“type”列会在子类上明确禁用，除非显式覆盖。

+   **[orm] [bug]**

    当 lazy=’dynamic’ 与 uselist=False 结合时，会发出警告。这在 0.8 中会引发异常。

+   **[orm] [bug]**

    修复了一个错误，即相关对象赋值中的用户错误可能导致递归溢出，如果赋值触发与错误类上的双向属性同名的 backref 到相同的目标。现在会引发一个信息性错误。

+   **[orm] [bug]**

    修复了当 ORM 绑定“version”列时传递错误类型信息的 bug，当使用“version”功能时。测试感谢 Daniel Miller。

    参考：[#2539](https://www.sqlalchemy.org/trac/ticket/2539)

+   **[orm] [bug]**

    在 Session.commit()中发生的“flush”中添加了额外的逻辑，以便在随后的 flush 之前刷新由 after_flush()或 after_flush_postexec()钩子添加的额外状态，然后“commit”完成。后续调用 flush()将继续，直到 after_flush 钩子停止添加新状态。在事件 after_flush()钩子每次添加新内容时，还设置了一个“overflow”计数器为 100。

    参考：[#2566](https://www.sqlalchemy.org/trac/ticket/2566)

### engine

+   **[engine] [feature]**

    事件系统的内存使用情况显著改善；直到为该事件建立了实例级别的监听器，才会为特定类型的事件创建实例级别集合。

    参考：[#2516](https://www.sqlalchemy.org/trac/ticket/2516)

+   **[engine] [bug]**

    修复了当 QueuePool 有线程等待连接时发生的 disconnect detect + dispose bug，会使这些线程等待旧池的超时时间（如果禁用超时，则会无限期等待）。现在修复了这个问题，通过特殊异常情况通知这些等待者，并让它们转移到新池。

    参考：[#2522](https://www.sqlalchemy.org/trac/ticket/2522)

+   **[engine] [bug]**

    在 mysql/__init__.py 中添加了 gaerdbms 导入，缺少此导入会导致无法加载新的 GAE 方言。

    参考：[#2529](https://www.sqlalchemy.org/trac/ticket/2529)

+   **[engine] [bug]**

    修复了 cextension bug，其中如果给定的索引是 Column 对象而不是字符串，则“模糊列错误”将无法正常工作。请注意，这里仍然存在一些列定位问题，在 0.8 中已修复。

    参考：[#2553](https://www.sqlalchemy.org/trac/ticket/2553)

+   **[engine] [bug]**

    修复了 Enum 的 repr()，包括“name”和“native_enum”标志。有助于 Alembic 自动生成。

### sql

+   **[sql] [bug]**

    修复了 DropIndex 构造以支持与远程模式中的表关联的索引。

    参考：[#2571](https://www.sqlalchemy.org/trac/ticket/2571)

+   **[sql] [bug]**

    修复了 over()构造中的 bug，其中将空列表传递给 partition_by 或 order_by，而不是 None，将无法正确生成。感谢 Gunnlaugur Þór Briem。

    参考：[#2574](https://www.sqlalchemy.org/trac/ticket/2574)

+   **[sql] [bug]**

    修复了 CTE bug，其中 CTE 本身存在位置绑定参数会破坏绑定参数的整体排序。这主要影响支持位置绑定+ CTE 支持的 SQL Server 平台。

    参考：[#2521](https://www.sqlalchemy.org/trac/ticket/2521)

+   **[sql] [bug]**

    修复了 CTE 中更多的不直观问题，这些问题阻止了在不使用别名的情况下引用 CTE 自身的联合。现在，CTE 根据名称唯一呈现，仅呈现给定名称的最外层 CTE - 所有其他引用只是作为名称呈现。这甚至包括引用不同版本的相同 CTE 对象的其他 CTE/SELECT，比如引用该 SELECT 的 SELECT 或 UNION ALL。在这种情况下，我们在对象标识和词法标识之间有些放松通常的联系。两个不相关的 CTE 之间的真实名称冲突现在会引发错误。

+   **[sql] [bug]**

    在通用表达式的 WITH RECURSIVE 子句中，对列名应用引用规则，根据原始列的引用规则进行引用。

    参考：[#2512](https://www.sqlalchemy.org/trac/ticket/2512)

+   **[sql] [bug]**

    修复了在 0.7.6 中引入的回归问题，即在某些“clone+replace”场景中，SELECT 语句的 FROM 列表可能不正确。

    参考：[#2518](https://www.sqlalchemy.org/trac/ticket/2518)

+   **[sql] [bug]**

    修复了在嵌���子查询中使用 UNION 或类似操作会干扰结果列定位的错误，即在结果列与嵌套 UNION 中的名称相同时。 

    参考：[#2552](https://www.sqlalchemy.org/trac/ticket/2552)

+   **[sql] [bug]**

    自 0.6 版本以来修复了一个关于结果行定位的回归问题。现在可以在 select()语句中使用基于字符串的列，比如 select(['id', 'name']).select_from('mytable')，并且可以通过具有这些名称的 Column 对象来定位该语句；这是 query(MyClass).from_statement(some_statement)工作的机制。在某个时候，使用 select(['id'])这种特定情况停止工作了，这等同于 select([literal_column('id')])，所以这已经重新启用并当然进行了测试。

    参考：[#2558](https://www.sqlalchemy.org/trac/ticket/2558)

+   **[sql] [bug]**

    将 is_()和 isnot()操作符添加到 ColumnOperators 基类中，以便这些长期可用的操作符作为其他操作符一样存在。

    参考：[#2544](https://www.sqlalchemy.org/trac/ticket/2544)

### postgresql

+   **[postgresql] [bug]**

    反射主键约束中的列现在按照约束本身定义的顺序返回，而不是表的顺序。感谢 Gunnlaugur Þór Briem。

    参考：[#2531](https://www.sqlalchemy.org/trac/ticket/2531)

+   **[postgresql] [bug]**

    将“terminating connection”添加到我们用于检测与 PG 断开连接的消息列表中，当服务器重新启动时，某些版本中似乎存在这种情况。

    参考：[#2570](https://www.sqlalchemy.org/trac/ticket/2570)

### mysql

+   **[mysql] [bug]**

    更新了 mysqlconnector 接口，使用了更新的“client flag”和“charset”API，感谢 David McNelis。

### sqlite

+   **[sqlite] [feature]**

    增加了对 SQLite 中实现的 localtimestamp() SQL 函数的支持，感谢 Richard Mitchell。

+   **[sqlite] [bug]**

    调整了一个非常古老的 bug 修复，该修复试图解决一个 SQLite 问题，而该问题本身已在 sqlite 3.6.14 时“修复”，即在使用“foreign_key_list”pragma 时，表名周围的引号问题。修复已调整为不干扰实际上在列或表的名称中*的引号，尽可能大程度地；如果目标表实际上在其名称周围有引号（即“”“mytable”””），则 sqlite 仍不会正确返回 foreign_key_list()的结果。

    参考文献：[#2568](https://www.sqlalchemy.org/trac/ticket/2568)

+   **[sqlite] [bug]**

    调整了列默认反射代码，将非字符串值转换为字符串，以适应旧版本 SQLite 不将默认信息作为字符串提供的情况。

    参考文献：[#2265](https://www.sqlalchemy.org/trac/ticket/2265)

### mssql

+   **[mssql] [bug]**

    修复了编译器错误，其中在 ORDER BY 中使用相关子查询将无法正确呈现，如果语句还使用了 LIMIT/OFFSET，这是由于在 ROW_NUMBER() OVER 子句中的错误呈现导致的。修复由 sayap 提供。

    参考文献：[#2538](https://www.sqlalchemy.org/trac/ticket/2538)

+   **[mssql] [bug]**

    修复了编译器错误，其中给定的 select()如果具有“offset”属性，则会修改它，导致构造在第二次编译时无法正确编译。

    参考文献：[#2545](https://www.sqlalchemy.org/trac/ticket/2545)

+   **[mssql] [bug]**

    修复了主键约束的反射 bug，如果相同的约束/表存在于多个模式中，则会使列加倍。

## 0.7.8

发布日期：2012 年 6 月 16 日

### orm

+   **[orm] [feature]**

    flush()的‘objects’参数不再被弃用，因为已经确定了一些有效的用例。

+   **[orm] [bug]**

    修复了 subqueryload()从多态映射到目标的问题，将为多态结果中遇到的每个不同的类产生一个新的查询调用。

    参考文献：[#2480](https://www.sqlalchemy.org/trac/ticket/2480)

+   **[orm] [bug]**

    修复了在声明中的 bug，其中加入表的列的优先级（通常是用于 id 的复合列）如果列包含与其属性名称不同的名称，则将无法正确。这将导致针对实体属性的主键条件不正确。与此相关，因为这应该是那个的一部分，这是。

    参考文献：[#1892](https://www.sqlalchemy.org/trac/ticket/1892)，[#2491](https://www.sqlalchemy.org/trac/ticket/2491)

+   **[orm] [bug]**

    修复了 identity_key()函数不接受标量参数作为标识的问题。

    参考文献：[#2508](https://www.sqlalchemy.org/trac/ticket/2508)

+   **[orm] [bug]**

    修复了 populate_existing 选项不会传播到子查询急切加载程序的 bug。

    参考文献：[#2497](https://www.sqlalchemy.org/trac/ticket/2497)

### 引擎

+   **[engine] [bug]**

    修复了结果代理的 C 版本中的内存泄漏 bug，即不提供纯 Python 元组作为结果行的 DBAPI 将无法正确减少引用计数。受影响最严重的 DBAPI 是 pyodbc。

    参考：[#2489](https://www.sqlalchemy.org/trac/ticket/2489)

+   **[engine] [bug]**

    修复了影响 Py3K 的 bug，即传递给 engine/connection execute()的字符串位置参数将无法被正确解释，因为 Py3K 字符串上存在 __iter__。

    参考：[#2503](https://www.sqlalchemy.org/trac/ticket/2503)

### sql

+   **[sql] [bug]**

    将 BIGINT 添加到 types.__all__，将 BIGINT、BINARY、VARBINARY 添加到 sqlalchemy 模块命名空间，以及确保此破坏不会再次发���的测试。

    参考：[#2499](https://www.sqlalchemy.org/trac/ticket/2499)

+   **[sql] [bug]**

    修复了当 SELECT 语句包含 UNION 或其他复合表达式时，公共表达式渲染无法正确运行的 bug，感谢 btbuilder。

    参考：[#2490](https://www.sqlalchemy.org/trac/ticket/2490)

+   **[sql] [bug]**

    修复了在克隆的 select()构造上 append_column()无法正确运行的 bug，感谢 Gunnlaugur Þór Briem。

    参考：[#2482](https://www.sqlalchemy.org/trac/ticket/2482)

### postgresql

+   **[postgresql] [bug]**

    移除了在反射枚举时不必要的表子句，感谢 Gunnlaugur Þór Briem。

    参考：[#2510](https://www.sqlalchemy.org/trac/ticket/2510)

### mysql

+   **[mysql] [feature]**

    添加了一个新的 Google App Engine 方言。感谢 Richie Foreman。

    参考：[#2484](https://www.sqlalchemy.org/trac/ticket/2484)

### oracle

+   **[oracle] [bug]**

    向 oracle.*添加了 ROWID。

    参考：[#2483](https://www.sqlalchemy.org/trac/ticket/2483)

## 0.7.7

发布日期：2012 年 5 月 5 日星期六

### orm

+   **[orm] [feature]**

    向 Query 添加了 prefix_with()方法，调用 select().prefix_with()以允许在语句中放置 MySQL SELECT 指令。感谢 Diana Clarke。

    参考：[#2443](https://www.sqlalchemy.org/trac/ticket/2443)

+   **[orm] [feature]**

    添加了新标志@validates include_removes。当为 True 时，集合删除和属性删除事件也将发送到验证函数，当使用此标志时，该函数将接受额外参数“is_remove”。

+   **[orm] [bug]**

    修复了工作单元中的问题，即将非 None 的自引用多对一关系设置为 None 时，如果前一个值尚未加载，则无法持久化更改。

    参考：[#2477](https://www.sqlalchemy.org/trac/ticket/2477)

+   **[orm] [bug]**

    修复了 0.7.6 中引入的错误，即针对被映射为连接或其他间接可选择的列映射集合在使用时会失败的 bug。

    参考：[#2409](https://www.sqlalchemy.org/trac/ticket/2409)

+   **[orm] [bug]**

    修复了一个 bug，即在 merge()操作中不正确地包含未在类中映射的 polymorphic_on 列，导致错误。

    参考：[#2449](https://www.sqlalchemy.org/trac/ticket/2449)

+   **[orm] [bug]**

    修复了表达式注释机制中的错误，可能导致使用 column_property() 时，SELECT 语句的别名和连接渲染不正确。

    引用：[#2453](https://www.sqlalchemy.org/trac/ticket/2453)

+   **[orm] [bug]**

    修复了 OrderingList 无法被 pickle 的 bug。感谢 Jeff Dairiki

    引用：[#2454](https://www.sqlalchemy.org/trac/ticket/2454)

+   **[orm] [bug]**

    修复了关系比较中的 bug，调用未实现的方法如 SomeClass.somerelationship.like() 会产生递归溢出，而不是 NotImplementedError。

### sql

+   **[sql] [feature]**

    添加了新的连接事件 dbapi_error()。对于所有 DBAPI 级别的错误，传递原始的 DBAPI 异常，在 SQLAlchemy 修改游标状态之前调用。

+   **[sql] [bug]**

    当 Index 创建时没有列时，移除警告；虽然这可能不是用户想要的，但作为一个 Index 可能只是某个名称的索引的占位符是一个有效的用例。

+   **[sql] [bug]**

    如果在调用“with engine.begin()”时 conn.begin() 失败，则新获取的 Connection 在正常传播异常之前会被显式关闭。

+   **[sql] [bug]**

    将 BINARY、VARBINARY 添加到 types.__all__ 中。

    引用：[#2474](https://www.sqlalchemy.org/trac/ticket/2474)

### postgresql

+   **[postgresql] [feature]**

    为 PostgreSQL 添加了新的 for_update/with_lockmode() 选项：for_update=”read”/ with_lockmode(“read”)，for_update=”read_nowait”/ with_lockmode(“read_nowait”)。这些分别发出“FOR SHARE”和“FOR SHARE NOWAIT”。感谢 Diana Clarke

    引用：[#2445](https://www.sqlalchemy.org/trac/ticket/2445)

+   **[postgresql] [bug]**

    在反射域时删除了不必要的表子句。

    引用：[#2473](https://www.sqlalchemy.org/trac/ticket/2473)

### mysql

+   **[mysql] [bug]**

    修复了在 InnoDB 中自动增量复合列的“KEY”子句中的列名会双引号保留字的错误。感谢 Jeff Dairiki。

    引用：[#2460](https://www.sqlalchemy.org/trac/ticket/2460)

+   **[mysql] [bug]**

    修复了对“information_schema”模式的 get_view_names() 无法检索标记为“SYSTEM VIEW”的视图的 bug。感谢 Matthew Turland.

+   **[mysql] [bug]**

    修复了如果在不支持 cast() 的 SQL 表达式上使用 cast() 并且因此方言未呈现 CAST，则评估顺序可能会更改的 bug；现在对这些表达式应用分组。

    引用：[#2467](https://www.sqlalchemy.org/trac/ticket/2467)

### sqlite

+   **[sqlite] [feature]**

    添加了 SQLite 执行选项“sqlite_raw_colnames=True”，将绕过 SQLite cursor.description 返回的列名中的“.” 的尝试删除。

    引用：[#2475](https://www.sqlalchemy.org/trac/ticket/2475)

+   **[sqlite] [bug]**

    当 Table 的主键列被替换时，例如通过 extend_existing，由 insert()构造使用的“自动增量”列将被重置。以前它会继续引用先前的主键列。

    参考：[#2525](https://www.sqlalchemy.org/trac/ticket/2525)

### mssql

+   **[mssql] [feature]**

    向 PyODBC 方言添加了临时 create_engine 标志 supports_unicode_binds，以强制方言是否将 Python unicode 文字传递给 PyODBC。

+   **[mssql] [bug]**

    修复了在使用 pyodbc 方言时使用 use_scope_identity create_engine()标志的 bug。以前，如果设置为 False，此标志将被忽略。当设置为 False 时，每次 INSERT 后都会得到“SELECT @@identity”，以获取最后插入的 ID，对于那些“implicit_returning”设置为 False 的表。

+   **[mssql] [bug]**

    使用 SQL Server 的 UPDATE..FROM 语法要求在 FROM 子句中存在更新的表，当表的别名也存在于 FROM 子句中时。现在，当 FROM 存在时，更新的表总��存在于 FROM 中。感谢 sayap。

    参考：[#2468](https://www.sqlalchemy.org/trac/ticket/2468)

## 0.7.6

发布日期：2012 年 3 月 14 日

### orm

+   **[orm] [feature]**

    向 Session 添加了“no_autoflush”上下文管理器，与 with 一起使用：将暂时禁用自动刷新。

+   **[orm] [feature]**

    在 Query 中添加了 cte()方法，调用 Core 中的公共表达式支持（见下文）。

    参考：[#1859](https://www.sqlalchemy.org/trac/ticket/1859)

+   **[orm] [feature]**

    添加了在使用 query(sometable).filter_by(colname=value)时查询绑定到表的列名的能力。

    参考：[#2400](https://www.sqlalchemy.org/trac/ticket/2400)

+   **[orm] [bug]**

    修复了事件注册 bug，主要表现为在事件与创建事件之后与 Session 类关联的 sessionmaker()实例未注册的情况。

    参考：[#2424](https://www.sqlalchemy.org/trac/ticket/2424)

+   **[orm] [bug]**

    修复了一个 bug，即在具有“literal”的 primaryjoin 条件中，当需要多次呈现相同绑定参数名称的某些类型的深度嵌套表达式上编译时会引发错误。

    参考：[#2425](https://www.sqlalchemy.org/trac/ticket/2425)

+   **[orm] [bug]**

    在对映对象进行多行删除时，移除了对受影响行数的检查。如果两行之间存在 ON DELETE CASCADE，我们无法从 DBAPI 中获得准确的行数；在大多数 DBAPI 中，这种特定计数在任何情况下都不受支持，MySQLdb 是一个明显的例外。

    参考：[#2403](https://www.sqlalchemy.org/trac/ticket/2403)

+   **[orm] [bug]**

    修复了使用 attribute_mapped_collection 或 column_mapped_collection 的对象无法被 pickle 的 bug。

    参考：[#2409](https://www.sqlalchemy.org/trac/ticket/2409)

+   **[orm] [bug]**

    修复了 MappedCollection 在仅在使用了 @collection.internally_instrumented 的自定义子类中使用时无法获得适当的集合仪器的错误。

    参考：[#2406](https://www.sqlalchemy.org/trac/ticket/2406)

+   **[orm] [bug]**

    修复了 SQL 适配机制在涉及联合继承、joinedload()、limit() 和列子句中的衍生函数的非常嵌套的情况下会失败的 bug。

    参考：[#2419](https://www.sqlalchemy.org/trac/ticket/2419)

+   **[orm] [bug]**

    修复了 CascadeOptions 的 repr() 以包括 refresh-expire。还重新设计了 CascadeOptions 以成为 <frozenset>。

    参考：[#2417](https://www.sqlalchemy.org/trac/ticket/2417)

+   **[orm] [bug]**

    改进了“声明式反射”示例，以支持单表继承，多次调用 prepare()，在替代模式下存在的表，仅将一部分类作为反射。

+   **[orm] [bug]**

    缩小了 flush() 中应用的测试范围，以检查表内部分 NULL PK 的 UPDATE，只有在真正需要进行 UPDATE 时才会发生。

    参考：[#2390](https://www.sqlalchemy.org/trac/ticket/2390)

+   **[orm] [bug]**

    修复了一个 bug，即如果方法名与列名冲突，则当 mapper 尝试检查方法对象上的 __get__() 方法时会引发 TypeError。

    参考：[#2352](https://www.sqlalchemy.org/trac/ticket/2352)

### 示例

+   **[examples] [bug]**

    修改了 Beaker 示例中的 _params_from_query() 函数，以从完全编译的语句中获取 bindparams，作为获取包括子查询在内的所有内容（列子句中等等）的快速方法。

### engine

+   **[engine] [feature]**

    添加了 “no_parameters=True” 执行选项用于连接。如果没有参数存在，则会将语句传递给 cursor.execute(statement)，从而调用 DBAPI 在不存在参数集合时的行为；对于 psycopg2 和 mysql-python，这意味着不会解释字符串中的 % 符号。只有在此选项下才会发生这种情况，并且不仅仅是如果参数列表为空，否则这将产生通常会转义百分号的 SQL 表达式的不一致行为（并且在编译时，无法提前知道在某些情况下是否会存在参数）。

    参考：[#2407](https://www.sqlalchemy.org/trac/ticket/2407)

+   **[engine] [feature]**

    添加了 pool_reset_on_return 参数到 create_engine，允许控制“连接返回”行为。还向 pool.reset_on_return 添加了新参数 'rollback'、'commit'、None，以允许更多控制连接返回活动。

    参考：[#2378](https://www.sqlalchemy.org/trac/ticket/2378)

+   **[engine] [feature]**

    向 Engine、Connection 添加了一些不错的上下文管理器：

    ```py
    with engine.begin() as conn:
        # <work with conn in a transaction>
        ...
    ```

    并且：

    ```py
    with engine.connect() as conn:
        # <work with conn>
        ...
    ```

    当完成时，都会关闭连接，并在 engine.begin() 时提交或回滚事务中的错误。

+   **[engine] [bug]**

    向 MockConnection（即与 strategy=”mock” 一起使用的）添加了 execution_options() 调用，该调用作为参数的传递。

### sql

+   **[sql] [feature]**

    添加了对 SQL 标准通用表达式（CTE）的支持，允许将 SELECT 对象作为 CTE 源（尚不支持 DML）。这通过在任何 select()构造上调用 cte()方法来调用。

    参考：[#1859](https://www.sqlalchemy.org/trac/ticket/1859)

+   **[sql] [bug]**

    修复了核心中的内存泄漏问题，当使用特定类型的结果获取时会发生，特别是在调用 orm query.count()时。

    参考：[#2427](https://www.sqlalchemy.org/trac/ticket/2427)

+   **[sql] [bug]**

    修复了在行上基于属性的列访问会引发 AttributeError（对于非 C 版本）或 NoSuchColumnError（对于 C 版本）的问题。现在在两种情况下都引发 AttributeError。 

    参考：[#2398](https://www.sqlalchemy.org/trac/ticket/2398)

+   **[sql] [bug]**

    添加了对使用列的.key 作为结果集行中的字符串标识符的支持。.key 目前被列为列的“备用”名称，并且被具有该键值作为其常规名称的列所取代。对于 SQLAlchemy 的下一个主要版本，我们可能会颠倒这种优先顺序，使.key 优先，但目前尚未决定。

    参考：[#2392](https://www.sqlalchemy.org/trac/ticket/2392)

+   **[sql] [bug]**

    当在 insert()或 update()构造的 values()子句中声明不存在的列时，会发出警告。将在 0.8 版本中转为异常。

    参考：[#2413](https://www.sqlalchemy.org/trac/ticket/2413)

+   **[sql] [bug]**

    在 SELECT 语句中应用标签的方式发生了重大变化，允许“截断”标签，即在 Python 中生成的标签名称超过最大标识符长度时（请注意，这可以通过 create_engine()上的 label_length 进行配置），在子查询中正确引用，并且在结果集行中使用它们的原始 Python 名称。

    参考：[#2396](https://www.sqlalchemy.org/trac/ticket/2396)

+   **[sql] [bug]**

    修复了新“autoload_replace”标志中的错误，该错误会导致无法保留反射表的主键约束。

    参考：[#2402](https://www.sqlalchemy.org/trac/ticket/2402)

+   **[sql] [bug]**

    当传递的参数无法解释为列或表达式时，Index 将引发异常。当创建 Index 时没有任何列时将发出警告。

    参考：[#2380](https://www.sqlalchemy.org/trac/ticket/2380)

### mysql

+   **[mysql] [feature]**

    通过 Index 和 PrimaryKeyConstraint 的新 mysql_using 参数，添加了对 MySQL 索引和主键约束类型（即 USING）的支持，感谢 Diana Clarke。

    参考：[#2386](https://www.sqlalchemy.org/trac/ticket/2386)

+   **[mysql] [feature]**

    为所有 MySQL 方言添加了对“isolation_level”参数的支持。感谢 mu_mind 在这里的补丁。

    参考：[#2394](https://www.sqlalchemy.org/trac/ticket/2394)

### sqlite

+   **[sqlite] [bug]**

    修复了 C 扩展中的错误，其中字符串格式不会应用于作为整数返回的 Numeric 值；这主要影响 SQLite，因为它不维护数字比例设置。

    参考：[#2432](https://www.sqlalchemy.org/trac/ticket/2432)

### mssql

+   **[mssql] [功能]**

    添加了对 MSSQL INSERT、UPDATE 和 DELETE 表提示的支持，使用 UpdateBase 上的新 with_hint()方法。

    参考：[#2430](https://www.sqlalchemy.org/trac/ticket/2430)

### oracle

+   **[oracle] [功能]**

    添加了一个新的 create_engine()标志 coerce_to_decimal=False，禁用了精度数字处理，可以通过将所有数字值转换为 Decimal 来增加大量开销。

    参考：[#2399](https://www.sqlalchemy.org/trac/ticket/2399)

+   **[oracle] [错误]**

    为 LONG 添加了缺失的编译支持

    参考：[#2401](https://www.sqlalchemy.org/trac/ticket/2401)

+   **[oracle] [错误]**

    将“LEVEL”添加到 Oracle 的保留字列表中。

    参考：[#2435](https://www.sqlalchemy.org/trac/ticket/2435)

## 0.7.5

发布日期：2012 年 1 月 28 日

### orm

+   **[orm] [功能]**

    添加了“class_registry”参数到 declarative_base()。允许两个或更多声明基类共享相同的类名注册表。

+   **[orm] [功能]**

    query.filter()接受多个条件，这些条件将通过 AND 连接，即 query.filter(x==y, z>q, …)

+   **[orm] [功能]**

    添加了新的关系加载器选项功能，允许“默认”加载器策略。将‘*’传递给 joinedload()、lazyload()、subqueryload()或 noload()中的任何一个，这将成为用于所有关系的加载器策略，除了在查询中明确声明的关系。感谢新兴贡献者 Kent Bower 提供了详尽而写得很好的测试套件！

    参考：[#2351](https://www.sqlalchemy.org/trac/ticket/2351)

+   **[orm] [功能]**

    添加了新的声明性反射示例，说明了如何最好地将表反射与声明性混合使用，并使用了一些新功能。

    参考：[#2356](https://www.sqlalchemy.org/trac/ticket/2356)

+   **[orm] [错误]**

    修复了在失败的 flush 后建立的修改会作为随后由手动调用 rollback()自动开始的事务的一部分提交的问题。在 rollback()中检查会话的状态，如果存在新状态，则发出警告并第二次调用 restore_snapshot()，丢弃这些更改。

    参考：[#2389](https://www.sqlalchemy.org/trac/ticket/2389)

+   **[orm] [错误]**

    修复了从 0.7.4 中的回归，其中使用已经被超类作为“polymorphic_on”的列无法解析基础列的问题。

    参考：[#2345](https://www.sqlalchemy.org/trac/ticket/2345)

+   **[orm] [错误]**

    如果两个未连接的关系不当地使用 xyzload_all()，则引发异常。

    参考：[#2370](https://www.sqlalchemy.org/trac/ticket/2370)

+   **[orm] [错误]**

    修复了一个 bug，其中 event.listen(SomeClass)强制进行了一个完全不必要的 mapper 编译，使得在模块导入时设置事件非常困难（没人注意到这个问题吗？）

    参考：[#2367](https://www.sqlalchemy.org/trac/ticket/2367)

+   **[orm] [bug]**

    修复了一个 bug，其中 hybrid_property 在 any()、has()中作为 kw 参数时无法正常工作。

+   **[orm] [bug]**

    确保所有 ORM 异常都可以 pickle 化，以实现多进程兼容性。

    参考：[#2371](https://www.sqlalchemy.org/trac/ticket/2371)

+   **[orm] [bug]**

    当在一个混合属性上使用 setattr/delattr 时，如果混合属性没有定义 fset 或 fdel，实现了标准的“无法设置属性”/“无法删除属性”AttributeError。

    参考：[#2353](https://www.sqlalchemy.org/trac/ticket/2353)

+   **[orm] [bug]**

    修复了一个 bug，即未 pickle 化的对象在 mutable 对象扩展建立的 unpickle()事件中没有设置足够的状态以在 __eq__()或类似方法中正确工作，如果对象需要在 __eq__()或类似方法中访问 ORM 属性。

    参考：[#2362](https://www.sqlalchemy.org/trac/ticket/2362)

+   **[orm] [bug]**

    修复了一个 bug，当使用 relationship()的 load_on_pending 标志时，“merge”级联可能会错误地解释未加载的属性。感谢 Kent Bower 提供的测试。

    参考：[#2374](https://www.sqlalchemy.org/trac/ticket/2374)

+   **[orm]**

    从 0.6 版本中修复了一个回归问题，即如果在一个待处理对象上需要发出非“get()”懒惰子句的情况下使用了“load_on_pending”relationship()标志，它将无法加载。

### 示例

+   **[examples] [feature]**

    简化了版本示例，使用了声明性 mixin 以及事件监听器，而不是元类+SessionExtension。

    参考：[#2313](https://www.sqlalchemy.org/trac/ticket/2313)

+   **[examples] [bug]**

    修复了 large_collection.py 在删除表之前关闭会话的问题。

    参考：[#2346](https://www.sqlalchemy.org/trac/ticket/2346)

### engine

+   **[engine] [bug]**

    为 StatementError、DBAPIError、列错误添加了 __reduce__，以便异常在使用多进程时可 pickle 化。但是，目前并非所有的 DBAPI 都支持这一点，比如 psycopg2。

    参考：[#2371](https://www.sqlalchemy.org/trac/ticket/2371)

+   **[engine] [bug]**

    当将非字符串或无效字符串传递给 SQLite 使用的任何日期/时间处理器之一时，包括 C 和 Python 版本时，改进了错误消息。

    参考：[#2382](https://www.sqlalchemy.org/trac/ticket/2382)

+   **[engine] [bug]**

    修复了一个 bug，其中一个与表绑定的名为“<a>_<b>”的 Column 对象与标记为“<tablename>_<colname>”的列匹配时，可能会在结果集行中不当地匹配。

    参考：[#2377](https://www.sqlalchemy.org/trac/ticket/2377)

+   **[engine] [bug]**

    修复了“mock”策略中的 bug，其中未调用正确的 DDL 访问方法，导致“CREATE/DROP SEQUENCE”语句重复。

    参考：[#2384](https://www.sqlalchemy.org/trac/ticket/2384)

### sql

+   **[sql] [feature]**

    新的反射功能“autoload_replace”；在表上设置为 False 时，可以加载表而不替换现有列。允许构建更灵活的表构造/反射链，包括它有助于将声明性与表反射结合起来。请参阅维基上的新示例。

    引用：[#2356](https://www.sqlalchemy.org/trac/ticket/2356)

+   **[sql] [feature]**

    在 sqlalchemy.sql 命名空间中添加了“false()”和“true()”表达式构造，尽管目前还不是 __all__ 的一部分。

+   **[sql] [feature]**

    特定于方言的编译器现在对所有类型/语句编译问题引发 CompileError，而不是 InvalidRequestError 或 ArgumentError。对于有问题的列，CREATE TABLE 的 DDL 将重新引发 CompileError 以包含表/列信息。

    引用：[#2361](https://www.sqlalchemy.org/trac/ticket/2361)

+   **[sql] [bug]**

    改进了 add_column()的 API，如果将相同的列添加到其自身的表中，则不会引发错误，并且约束不会被重复。还有助于一些反射/声明性模式。

    引用：[#2356](https://www.sqlalchemy.org/trac/ticket/2356)

+   **[sql] [bug]**

    修复了一个问题，即当`bindparam()`中`required=True`时，如果语句没有给出任何参数，则不会引发“required”异常。

    引用：[#2381](https://www.sqlalchemy.org/trac/ticket/2381)

### mysql

+   **[mysql] [bug]**

    修复了一个正则表达式，过滤掉非反射“PARTITION”指令的警告，感谢 George Reilly

    引用：[#2376](https://www.sqlalchemy.org/trac/ticket/2376)

### sqlite

+   **[sqlite] [bug]**

    在 SQLite 中，FK 约束的“name”反映为“None”，而不是“0”或其他整数值。SQLite 似乎在任何情况下都不支持约束命名。

    引用：[#2364](https://www.sqlalchemy.org/trac/ticket/2364)

+   **[sqlite] [bug]**

    sql.false()和 sql.true()在 sqlite 中分别编译为 0 和 1

    引用：[#2368](https://www.sqlalchemy.org/trac/ticket/2368)

+   **[sqlite] [bug]**

    在获取表名和视图名时，删除了 SQLite 方言中的一个错误的“raise”，其中逻辑设置为退回到没有“sqlite_temp_master”表的旧版本的 SQLite。

### mssql

+   **[mssql] [bug]**

    调整了在 mssql.TIME 类型中使用的正则表达式，以确保仅接收六位数字作为值的“microseconds”部分，这是 Python 的 datetime.time()所期望的。请注意，至少在 pyodbc 中似乎尚不支持发送微秒。

    引用：[#2340](https://www.sqlalchemy.org/trac/ticket/2340)

+   **[mssql] [bug]**

    根据有关 pymssql 做得更好的报告，取消了对 pymssql 的“30 char”限制。pymssql 尚未经过充分测试，由于 DBAPI 正在变化，目前还不清楚该驱动程序的状态以及 SQLAlchemy 的实现应该如何适应。

    引用：[#2347](https://www.sqlalchemy.org/trac/ticket/2347)

### oracle

+   **[oracle] [bug]**

    将 ORA-03135 添加到永无止境的 Oracle “连接丢失”错误列表中

    参考：[#2388](https://www.sqlalchemy.org/trac/ticket/2388)

### 杂项

+   **[bug] [core]**

    将用于缓存 INSERT/UPDATE/DELETE 语句的映射器的 LRUCache 更改为使用递增计数器而不是时间戳来跟踪条目，以提高可靠性，而不是使用 time.time()，后者可能会导致某些平台上的测试失败。

    参考：[#2379](https://www.sqlalchemy.org/trac/ticket/2379)

+   **[bug] [core]**

    在调用池连接代理的弱引用回调函数之前，为“finalize”函数添加了一个布尔检查，以避免在应用程序退出时发出警告，当应用程序退出并且 gc 在调用弱引用回调之前从模块中删除函数时，不会发出此函数为 None 的警告。

    参考：[#2383](https://www.sqlalchemy.org/trac/ticket/2383)

+   **[bug] [py3k]**

    修复了对 util.py3k 标志的不当使用，并将其重命名为 util.py3k_warning，因为此标志仅用于检测 -3 标志系列的导入限制。

    参考：[#2348](https://www.sqlalchemy.org/trac/ticket/2348)

## 0.7.4

发布日期：2011 年 12 月 09 日星期五

### orm

+   **[orm] [feature]**

    polymorphic_on 现在接受许多新类型的值：

    > +   否则未映射的独立表达式
    > +   
    > +   column_property() 对象
    > +   
    > +   任何 column_property() 的字符串名称或映射列的属性名称

    文档包含了一个使用 case() 构造的示例，这很可能是一个常见的构造用法，并且是

    polymorphic_on 中的独立表达式传播到单表继承子类，以便它们在 WHERE / JOIN 子句中用于限制行到该子类，这是通常的行为。

    参考：[#2238](https://www.sqlalchemy.org/trac/ticket/2238), [#2345](https://www.sqlalchemy.org/trac/ticket/2345)

+   **[orm] [feature]**

    IdentitySet 支持 - 运算符，与 difference() 相同，处理 Session.dirty 等时非常方便。

    参考：[#2301](https://www.sqlalchemy.org/trac/ticket/2301)

+   **[orm] [feature]**

    为 Column 的 autoincrement 添加了一个名为“ignore_fk”的新值，可用于强制在仍然属于 ForeignKeyConstraint 的列上自动增量。关系文档中的新示例说明了其用法。

+   **[orm] [bug]**

    修复了在从陈旧的一对多中删除值时“弹出”值时的 backref 行为 - 该操作被跳过，因为一对多已经被更新。

    参考：[#2315](https://www.sqlalchemy.org/trac/ticket/2315)

+   **[orm] [bug]**

    在多年不这样做之后，对“X 是否为 Y 的父级”功能增加了更多细分，这在确定“Y”上的 FK 是否也需要“被置空”以及是否应该使用 delete-orphan 级联删除时使用。测试现在考虑了父级的 Python 标识以及其标识键，以查看 Y 的最后已知父级是否确实为 X。如果无法做出决定，则会引发 StaleDataError。引发此错误的条件非常罕见，要求以前的父级已被垃圾回收，并且以前可能会不适当地更新/删除已经移至新父级的记录，尽管以前可能会在面对模棱两可时产生“静默成功”的情况，但现在会引发错误。过期“Y”会重置“父级”跟踪器，这意味着 X.remove(Y)可能会删除 Y，即使 X 已过期，但这与以前的行为相同；建议在这种情况下也过期 X。

    参考：[#2264](https://www.sqlalchemy.org/trac/ticket/2264)

+   **[orm] [bug]**

    修复了在 query.get()中对用户映射对象在布尔上下文中的不适当评估。也在 0.6.9 中。

    参考：[#2310](https://www.sqlalchemy.org/trac/ticket/2310)

+   **[orm] [bug]**

    添加了缺失的逗号到 PASSIVE_RETURN_NEVER_SET 符号

    参考：[#2304](https://www.sqlalchemy.org/trac/ticket/2304)

+   **[orm] [bug]**

    Cls.column.collate(“some collation”)现在可以工作。也在 0.6.9 中

    参考：[#1776](https://www.sqlalchemy.org/trac/ticket/1776)

+   **[orm] [bug]**

    现在，在插入或更新操作后，复合属性的值会过期，而不是在原地重新生成。这确保了在刷新时过期的列值将首先被加载，然后再使用该值重新生成复合值。

    参考：[#2309](https://www.sqlalchemy.org/trac/ticket/2309)

+   **[orm] [bug]**

    修复了当访问加载复合值时，即使所有列值已经存在，也会发出“refresh”事件的问题，这是合适的。这修复了依赖于“load”事件确保 _parents 字典是最新的“mutable”扩展。感谢 Scott Torborg 在这里提供的测试用例。

    参考：[#2308](https://www.sqlalchemy.org/trac/ticket/2308)，[#2309](https://www.sqlalchemy.org/trac/ticket/2309)

+   **[orm] [bug]**

    修复了一个 bug，即使用具体继承与新的 ConcreteBase 或 AbstractConcreteBase 结合使用的子类的子类无法将超过一级的子类应用于每个基类的“多态加载器”。

    参考：[#2312](https://www.sqlalchemy.org/trac/ticket/2312)

+   **[orm] [bug]**

    修复了一个 bug，即使用新的 AbstractConcreteBase 的子类的子类在生成“base”映射器时无法获取正确的“base_mapper”属性，从而导致后续失败。

    参考：[#2312](https://www.sqlalchemy.org/trac/ticket/2312)

+   **[orm] [bug]**

    修复了对 ORM 级列创建的 column_property() 在生成某些类型的 joined-inh 连接时可能被视为一个不同的实体的 bug。

    参考：[#2316](https://www.sqlalchemy.org/trac/ticket/2316)

+   **[orm] [bug]**

    修复了当意外传递元组给 session.query() 时引发的错误格式化。同时适用于 0.6.9 版本。

    参考：[#2297](https://www.sqlalchemy.org/trac/ticket/2297)

+   **[orm] [bug]**

    跟踪对单表继承子类的 query.join() 调用，并用它们来消除通常附加到单表继承的额外 WHERE.. IN 条件，因为 join 应该适应它。这允许 OUTER JOIN 到单表子类产生正确的结果，并且在处理单表继承连接时将产生较少的 WHERE 条件。

    参考：[#2328](https://www.sqlalchemy.org/trac/ticket/2328)

+   **[orm] [bug]**

    __table_args__ 现在可以传递为空元组或空字典。感谢 Fayaz Yusuf Khan 提交的补丁。

    参考：[#2339](https://www.sqlalchemy.org/trac/ticket/2339)

+   **[orm] [bug]**

    当设置 delete-orphan 而没有删除时，更新警告消息不再引用 0.6 版本，因为我们从未升级为异常处理。理想情况下，这可能更好地作为一个异常，但无论如何都不是关键的。

    参考：[#2325](https://www.sqlalchemy.org/trac/ticket/2325)

+   **[orm] [bug]**

    在引用没有值的复合属性时修复了 get_history() 中的错误；增加了对 composites 的 get_history() 的覆盖，否则这只是一个用户级功能。

### 示例

+   **[examples] [bug]**

    修复了 history_meta.py 示例中的 bug，在其中“unique”标志未从生成列放在基类上的单表继承子类中移除。

### engine

+   **[engine] [bug]**

    修复了 transaction.rollback() 在事务是两阶段或保存点事务时在无效连接上抛出错误的 bug。对于普通事务，如果连接无效，rollback() 是一个空操作，所以虽然不清楚它是否应该是一个空操作，但至少现在接口是一致的。

    参考：[#2317](https://www.sqlalchemy.org/trac/ticket/2317)

### sql

+   **[sql] [feature]**

    update() 构造现在可以在 WHERE 子句中容纳多个表，这将生成一个“UPDATE..FROM”构造，由 PostgreSQL 和 MSSQL 识别。在 MySQL 上编译时，将生成“UPDATE t1, t2, ..”。此外，如果“values”参数或生成方法中使用 Column 对象作为键，则 MySQL 还可以在 SET 子句中渲染对多个表的更新。

    参考：[#1944](https://www.sqlalchemy.org/trac/ticket/1944), [#2166](https://www.sqlalchemy.org/trac/ticket/2166)

+   **[sql] [feature]**

    添加了名为“python_type”的类型访问器，返回特定 TypeEngine 实例的基本 Python 类型对象，如果已知，否则引发 NotImplementedError。

    参考：[#77](https://www.sqlalchemy.org/trac/ticket/77)

+   **[SQL] [错误]**

    关于，对于关于 select()中“from”列表的更改进行了一些调整。_froms 集合不再被记忆，因为这简化了各种用例，并消除了在表在表达式中使用之后附加列时需要“警告”的需要 - select()构造现在将始终生成正确的表达式。这里可能没有真实世界的性能损失；select()对象几乎总是临时制作的，希望优化 select()的重用的系统将使用“compiled_cache”功能。调用 select.bind 时会减少一个命中，但绝大多数用户不应该使用“bound metadata”：）。

    参考：[#2261](https://www.sqlalchemy.org/trac/ticket/2261), [#2316](https://www.sqlalchemy.org/trac/ticket/2316)

+   **[SQL] [错误]**

    进一步调整了来自的修复，以便生成方法在克隆后更好地工作（尽管这几乎是一个不常见的用例）。特别是这允许 with_only_columns()更一致地行为。添加了额外的文档以澄清 with_only_columns()的预期行为，这是由于结果而改变的。

    参考：[#2261](https://www.sqlalchemy.org/trac/ticket/2261), [#2319](https://www.sqlalchemy.org/trac/ticket/2319)

### 模式

+   **[模式] [特性]**

    添加了对���程“模式”的新支持：

+   **[模式] [特性]**

    Table 上的“extend_existing”标志现在允许反射过程对已经定义的 Table 对象生效；当 autoload=True 和 extend_existing=True 都设置时，将从 Table 中反射出完整的列集，然后将*覆盖*已经存在的列，而不是不发生任何活动。然而，直接在 autoload 运行中存在的列将像往常一样使用。

    参考：[#1410](https://www.sqlalchemy.org/trac/ticket/1410)

+   **[模式] [错误]**

    修复了 TypeDecorator 在使用“切换”类型的 TypeDecorator 时会返回过时值的错误，例如 CHAR/UUID 类型。

+   **[模式] [错误]**

    修复了 Inspector.get_table_names 中“order_by='foreign_key'”选项未正确实现排序的错误，替换为现有的排序算法

+   **[模式] [错误]**

    如果存在列级 CHECK 约束的“name”，现在在 CREATE TABLE 语句中使用“CONSTRAINT <name> CHECK <expression>”进行呈现。

    参考：[#2305](https://www.sqlalchemy.org/trac/ticket/2305)

+   **[模式]**

    MetaData()接受“schema”和“quote_schema”参数，这些参数将应用于同名参数的 Table 或 Sequence，这些参数保持默认值为`None`。

+   **[模式]**

    Sequence 接受“quote_schema”参数

+   **[模式]**

    对于 Table，如果 schema 参数明确为“None”，tometadata()将使用传入的 MetaData 的“schema”来创建新的 Table

+   **[模式]**

    添加了 CreateSchema 和 DropSchema DDL 构造 - 这些仅接受模式的字符串名称和一个“quote”标志。

+   **[schema]**

    当在 MetaData 中使用默认“schema”时，ForeignKey 在定位远程表时也会假定“default”模式。这允许将 MetaData 上的“schema”参数应用于任何一组否则没有“schema”的 Table 对象。

+   **[schema]**

    在方言上实现了一个“has_schema”方法，但目前只在 PostgreSQL 上有效。感谢 Manlio Perillo。

    参考：[#1679](https://www.sqlalchemy.org/trac/ticket/1679)

### postgresql

+   **[postgresql] [feature]**

    添加了 create_type 构造函数参数到 pg.ENUM。当为 False 时，在表创建/删除事件中不会执行 CREATE/DROP 或检查类型；只有直接调用 create()/drop()方法时才会执行此操作。有助于 Alembic 的“离线”脚本。

+   **[postgresql] [bug]**

    PostgreSQL 方言会记忆一个特定名称的 ENUM 在创建/删除序列期间已被处理。这使得在不需要任何“checkfirst”调用的情况下，创建/删除序列可以正常工作，并且在打开“checkfirst”时，只需要检查 ENUM 一次。

    参考：[#2311](https://www.sqlalchemy.org/trac/ticket/2311)

### mysql

+   **[mysql] [bug]**

    Unicode 调整允许最新的 pymysql（0.4 之后）在 Python 2 上通过 100%。

### mssql

+   **[mssql] [feature]**

    解除了对 SQL Server 的 SAVEPOINT 限制。所有测试都通过了，但不清楚是否存在更深层次的问题。

    参考：[#822](https://www.sqlalchemy.org/trac/ticket/822)

+   **[mssql] [bug]**

    修复了在 MSSQL 上未正确实现的 with_hint()功能 - 通常用于“WITH (NOLOCK)”提示（你不应该使用！应该使用快照隔离 :) ）

    参考：[#2336](https://www.sqlalchemy.org/trac/ticket/2336)

+   **[mssql] [bug]**

    使用新的 pyodbc 版本检测选项来修复 _need_decimal_fix。

    参考：[#2318](https://www.sqlalchemy.org/trac/ticket/2318)

+   **[mssql] [bug]**

    不要在 SQL Server 2000 上将“表名”转换为 NVARCHAR。然而，目前仍不清楚如何使 PyODBC 与 FreeTDS 0.91 完全配合。

    参考：[#2343](https://www.sqlalchemy.org/trac/ticket/2343)

+   **[mssql] [bug]**

    解码传入值以检索索引名称列表和这些索引中列的名称。

    参考：[#2269](https://www.sqlalchemy.org/trac/ticket/2269)

### 杂项

+   **[feature] [ext]**

    在混合文档中添加了一个“transformer”的示例 - 一个返回查询转换可调用对象的混合，与自定义比较器结合使用。使用 Query 上的新方法 with_transformation()。这里的用例相当实验性，但只需在 Query 中添加一行代码。

+   **[bug] [pyodbc]**

    基于 pyodbc 的方言现在可以准确解析 pyodbc 字符串，包括“py3-3.0.1-beta4”等珍贵内容。

    参考：[#2318](https://www.sqlalchemy.org/trac/ticket/2318)

+   **[bug] [ext]**

    当没有“默认”编译处理程序时，@compiles 装饰器会引发一个信息性错误消息，而不是 KeyError。

## 0.7.3

发布日期：2011 年 10 月 16 日（星期日）

### 通用

+   **[通用]**

    调整了“importlater”机制，该机制在内部用于解析导入循环，使得在导入 sqlalchemy 或 sqlalchemy.orm 之后完成了对 __import__ 的使用，从而避免在应用程序启动新线程后使用 __import__。修复。也适用于 0.6.9 版本。

    参考：[#2279](https://www.sqlalchemy.org/trac/ticket/2279)

### ORM

+   **[ORM]**

    改进了 query.join()，使得“左”侧可以更灵活地是非 ORM 可选择的，例如一个子查询。现在放置在 select_from()中的可选择项将被用作左侧，并优先于隐式使用映射的实体。如果基于外键缺失仍然失败，错误消息将包含此细节。感谢 IRC 上的 brianrhude 提供的测试用例。

    参考：[#2298](https://www.sqlalchemy.org/trac/ticket/2298)

+   **[ORM]**

    添加了 after_soft_rollback() Session 事件。无论是否发生实际的 DBAPI 级别回滚，此事件都会在调用 rollback()时无条件触发。该事件专门设计用于在回滚后，当 Session.is_active 为 True 时，允许对 Session 进行操作。

    参考：[#2241](https://www.sqlalchemy.org/trac/ticket/2241)

+   **[ORM]**

    向 orm.aliased()构造添加了“adapt_on_names”布尔标志。允许一个 aliased()构造将 ORM 实体链接到一个包含聚合或其他派生形式特定属性的可选择项，前提是名称与实体映射列的名称相同。

+   **[ORM]**

    向 column_property()添加了新标志 expire_on_flush=False，将那些本来被视为“只读”的属性标记为，在刷新发生后保留它们的值，即使父对象本身参与了更新。

+   **[ORM]**

    增强了 ORM 中的仪器支持 Py3K 的新参数样式“必需的 kw 参数”，即 fn(a, b, *, c, d)，fn(a, b, *args, c, d)。映射对象的 __init__ 方法的参数签名将被保留，包括必需的 kw 规则。

    参考：[#2237](https://www.sqlalchemy.org/trac/ticket/2237)

+   **[ORM]**

    修复了单位工作中的错误，即在高度相互链接的模式中的类之间检测“循环”时不会产生确定性结果；因此有时会漏掉一些应该被认为是循环的节点，并在后续出现问题。请注意，此错误也存在于 0.6 版本中；目前未进行回溯。

    参考文献：[#2282](https://www.sqlalchemy.org/trac/ticket/2282)

+   **[ORM]**

    从 0.6 版本中修复了各种与 synonym()相关的回归问题：

    > +   现在允许对同义词进行同义词操作。
    > +   
    > +   对 relationship()创建的同义词可以传递给 query.join()、传递给 query.options()的选项，以及通过名称传递给 query.with_parent()。

+   **[ORM]**

    修复了一个 bug，其中 mapper.order_by 属性在子查询急加载中的“内部”查询中会被忽略。同时也在 0.6.9 版本中修复。

    参考：[#2287](https://www.sqlalchemy.org/trac/ticket/2287)

+   **[orm]**

    Identity map .discard()在内部使用 dict.pop(,None)而不是“del”来避免在非确定性 gc 拆除期间出现 KeyError/警告。

    参考：[#2267](https://www.sqlalchemy.org/trac/ticket/2267)

+   **[orm]**

    修复了新的 composite 重写中的回归，由于缺少导入而导致 deferred=True 选项失败。

    参考：[#2253](https://www.sqlalchemy.org/trac/ticket/2253)

+   **[orm]**

    重新引入了“comparator_factory”参数到 composite()，在 0.7 发布时被移除。

    参考：[#2248](https://www.sqlalchemy.org/trac/ticket/2248)

+   **[orm]**

    修复了在 query.join()中出现的 bug，在复杂的多重重叠路径场景中会发生，同一张表可能会被连接两次。特别感谢 Dave Vitek 在这里的出色修复。

    参考：[#2247](https://www.sqlalchemy.org/trac/ticket/2247)

+   **[orm]**

    Query 将把 OFFSET 为零的切片转换为 None，以避免不必要的 OFFSET 子句被调用。

+   **[orm]**

    修复了一个边缘情况，其中当新映射上的关系在第一个映射上建立一个 backref 时，mapper 会无法完全更新内部状态。

+   **[orm]**

    修复了一个 bug，即如果重新定义了 __eq__()，一个关系多对一的 lazyload 会触发 __eq__()并失败。不适用于 0.6.9 版本。

    参考：[#2260](https://www.sqlalchemy.org/trac/ticket/2260)

+   **[orm]**

    调用 class_mapper()并传入一个不是“类型”的对象（即可能被映射的类）现在会引发一个信息性的 ArgumentError，而不是 UnmappedClassError。

    参考：[#2196](https://www.sqlalchemy.org/trac/ticket/2196)

+   **[orm]**

    新的事件钩子，MapperEvents.after_configured()。在 configure()步骤完成并且映射实际受到影响后调用。理论上，该事件在应用程序中只调用一次，除非在已经使用现有映射之后构建新映射。

+   **[orm]**

    当一个打开的 Session 被垃圾回收时，其中保留的对象再次被认为是分离的，当它们被 add()-ed 到一个新的 Session 时。这是通过额外检查前一个“session_key”是否实际存在于 Sessions 池中来实现的。

    参考：[#2281](https://www.sqlalchemy.org/trac/ticket/2281)

+   **[orm]**

    新的声明性特性：

    > +   __declare_last__()方法，为类方法建立一个事件监听器，当映射完成最终的“configure”步骤时将被调用。
    > +   
    > +   __abstract__ 标志。当该标志存在于类上时，该类将不会被映射。
    > +   
    > +   新的辅助类 ConcreteBase，AbstractConcreteBase。允许使用声明性进行具体映射，当“configure”映射步骤被调用时自动设置“polymorphic_union”。
    > +   
    > +   映射器本身有半私有方法，允许在配置完成后将“with_polymorphic”可选分配给映射器。

    参考：[#2239](https://www.sqlalchemy.org/trac/ticket/2239)

+   **[orm]**

    当子类的基类使用 @declared_attr 来定义常规列时，声明性将发出警告——此属性不会传播到子类。

    参考：[#2283](https://www.sqlalchemy.org/trac/ticket/2283)

+   **[orm]**

    用于将映射实例与其所属会话链接的整数“id”现在由序列生成函数生成，而不是 id(Session)，以消除回收的 id() 值可能导致不正确结果的可能性，无需检查对象实际是否在会话中。

    参考：[#2280](https://www.sqlalchemy.org/trac/ticket/2280)

+   **[orm]**

    行为改进：空的连接词，如 and_() 和 or_()，在包含连接词的上下文中将被展开，即 and_(x, or_()) 将产生 'X' 而不是 'X AND ()'。

    参考：[#2257](https://www.sqlalchemy.org/trac/ticket/2257)

+   **[orm]**

    修复了有关选择器的“from”列表计算的错误。现在，“from”计算被延迟，所以如果构造使用尚未附加到表的列对象，但稍后与表关联，则会使用该表生成 SQL 作为 FROM。这个更改对“froms”列表以及“correlates”集合的计算机制产生了相当深远的影响，因为一些“clause adaption”方案（这些方案在 ORM 中被大量使用）依赖于“froms”集合通常在适应完成之前会被缓存的事实。重写允许在任何时候清除并重新生成“froms”集合。

    参考：[#2261](https://www.sqlalchemy.org/trac/ticket/2261)

+   **[orm]**

    修复了 Select 的 with_only_columns() 方法如果传递了可选择对象，则会失败的错误。还包括在 0.6.9 版本中。

    参考：[#2270](https://www.sqlalchemy.org/trac/ticket/2270)

### examples

+   **[examples]**

    调整了 dictlike-polymorphic.py 示例以应用 CAST，使其在 PG 和其他数据库上运行。还包括在 0.6.9 版本中。

    参考：[#2266](https://www.sqlalchemy.org/trac/ticket/2266)

### engine

+   **[engine]**

    所有池类中的 recreate() 方法都使用 self.__class__ 来获取要生成的池类型，以支持子类化的情况。请注意，通常不需要对池进行子类化。

    参考：[#2254](https://www.sqlalchemy.org/trac/ticket/2254)

+   **[engine]**

    对于多参数语句记录的改进，长列表的绑定参数集将使用信息性指示器进行压缩。异常消息使用相同的改进格式。

    参考：[#2243](https://www.sqlalchemy.org/trac/ticket/2243)

+   **[engine]**

    在 pool.manage(dbapi).connect() 中添加了可选的“sa_pool_key”参数，因此不需要对参数进行序列化。

+   **[engine]**

    create_engine() 支持的入口点解析现在支持在内置或入口点解析的方言之上解析单独的 DBAPI 驱动程序，使用标准的‘+’符号 - 在解析为入口点之前会将其转换为‘.’。

    参考：[#2286](https://www.sqlalchemy.org/trac/ticket/2286)

+   **[引擎]**

    在连接中为“返回 unicode 检测”步骤添加了异常捕获和警告，允许在 NVARCHAR 上崩溃的数据库继续初始化，假设没有实现 NVARCHAR 类型。

    参考：[#2299](https://www.sqlalchemy.org/trac/ticket/2299)

### 模式

+   **[模式]**

    修改了 Column.copy() 使用 _constructor()，默认为 self.__class__，以创建新对象。这样可以更容易地支持 Column 的子类化。

    参考：[#2284](https://www.sqlalchemy.org/trac/ticket/2284)

+   **[模式]**

    为 SchemaItem 类添加了稍微更好的 __repr__()。请注意，这里的 repr 无法完全支持“repr 是构造函数”的想法，因为模式项可以非常深度嵌套/循环，某些内容的初始化较晚，等等。

    参考：[#2223](https://www.sqlalchemy.org/trac/ticket/2223)

### postgresql

+   **[postgresql]**

    在 Index() 中添加了“postgresql_using”参数，生成 USING 子句以指定 PG 的索引实现。感谢 Ryan P. Kelly 提供的补丁。

    参考：[#2290](https://www.sqlalchemy.org/trac/ticket/2290)

+   **[postgresql]**

    在使用 postgresql+psycopg2 方言时，为 create_engine() 添加了 client_encoding 参数；在连接时使用值调用 psycopg2 的 set_client_encoding() 方法。

    参考：[#1839](https://www.sqlalchemy.org/trac/ticket/1839)

+   **[postgresql]**

    修复了一个与 PG 9 中修改的相同索引行为相关的错误，影响了重命名列上的主键反射。也适用于 0.6.9 版本。

    参考：[#2141](https://www.sqlalchemy.org/trac/ticket/2141), [#2291](https://www.sqlalchemy.org/trac/ticket/2291)

+   **[postgresql]**

    Table、Sequence 的反射函数不再区分大小写。名称只能在大小写上有区别，并且将被正确区分。 

    参考：[#2256](https://www.sqlalchemy.org/trac/ticket/2256)

+   **[postgresql]**

    使用原子计数器作为服务器端游标名称的“随机数”来源；在极少数情况下已报告冲突。

+   **[postgresql]**

    缩小了在反射具有当前搜索路径中模式的外键引用表时所做的假设；仅当显式模式与引用表的模式匹配时，才会将显式模式应用于引用表。以前假定“当前”模式与完整搜索路径是同义词。

    参考：[#2249](https://www.sqlalchemy.org/trac/ticket/2249)

### mysql

+   **[mysql]**

    CREATE TABLE 将在 CHARSET 之后放置 COLLATE 选项，这似乎是 MySQL 关于其是否实际工作的任意规则的一部分。也适用于 0.6.9 版本。

    参考：[#2225](https://www.sqlalchemy.org/trac/ticket/2225)

+   **[mysql]**

    向 Index 构造添加了 mysql_length 参数，指定索引的“长度”。

    参考：[#2293](https://www.sqlalchemy.org/trac/ticket/2293)

### sqlite

+   **[sqlite]**

    确保无论是否使用 C 扩展，从数据库解析的非法日期/时间/日期时间字符串都会引发相同的 ValueError。

### mssql

+   **[mssql]**

    尝试支持 FreeTDS 0.91 与 Pyodbc 的结合。这包括当检测到 FreeTDS 0.91 时，将字符串绑定发送为 Python unicode 对象，并在检测到表时使用 CAST(? AS NVARCHAR)。然而，我会继续将 Pyodbc + FreeTDS 0.91 的行为描述为相当糟糕，仍然有许多查询（例如在反射中使用的查询）在 Linux 上导致核心转储，在 OSX 上根本无法使用，MemoryErrors 随处可见，而且对 Unicode 的支持完全有问题。

    参考：[#2273](https://www.sqlalchemy.org/trac/ticket/2273)

+   **[mssql]**

    当将标量选择与值进行比较时，= /！=的行为将不再产生 IN / NOT IN，从 0.8 版本开始；这种行为有点过于武断（如果要发出 IN，请使用`in_()`），现在会发出弃用警告。要立即获得 0.8 版本的行为并消除警告，可以在[`www.sqlalchemy.org/docs/07/dialects/mssql.html#scalar-select-comparisons`](https://www.sqlalchemy.org/docs/07/dialects/mssql.html#scalar-select-comparisons)给出一个编译器配方来覆盖 visit_binary()的行为。

    参考：[#2277](https://www.sqlalchemy.org/trac/ticket/2277)

+   **[mssql]**

    “0”被接受为 limit()的参数，将生成“TOP 0”。

    参考：[#2222](https://www.sqlalchemy.org/trac/ticket/2222)

### oracle

+   **[oracle]**

    修复了 zxjdbc 方言的 ReturningResultProxy。从 0.6 版本开始的回归。

    参考：[#2272](https://www.sqlalchemy.org/trac/ticket/2272)

+   **[oracle]**

    现在 String 类型在 Oracle 上生成 VARCHAR2，这是推荐的默认 VARCHAR。在 Oracle 方言中还添加了显式的 VARCHAR2 和 NVARCHAR2。使用 NVARCHAR 仍然生成“NVARCHAR2” - 在 Oracle 上没有“NVARCHAR” - 这仍然是“大写类型始终给出完全相同结果”政策的轻微破坏。VARCHAR 仍然生成“VARCHAR”，遵循该政策。如果 Oracle 曾经定义“VARCHAR”为他们声称的不同内容（在我看来永远不会发生），该类型将可用。

    参考：[#2252](https://www.sqlalchemy.org/trac/ticket/2252)

### 杂项

+   **[types]**

    在“精度”和“asdecimal”之外的基本 Float 类型的额外关键字参数将被忽略；这里添加了一个弃用警告和额外的文档，相关于

    参考：[#2258](https://www.sqlalchemy.org/trac/ticket/2258)

+   **[ext]**

    SQLSoup 将不包含在 SQLAlchemy 的 0.8 版本中；虽然很有用，但我们希望将 SQLAlchemy 本身专注于一种 ORM 使用范式。SQLSoup 很快将被第三方项目取代。

    参考：[#2262](https://www.sqlalchemy.org/trac/ticket/2262)

+   **[ext]**

    为 AssociationProxy 添加了 local_attr、remote_attr 和 attr 访问器，提供了在类级别快速访问代理属性的能力。

    参考：[#2236](https://www.sqlalchemy.org/trac/ticket/2236)

+   **[ext]**

    更改了关联代理字典上的 update() 方法，采用了鸭子类型方法，即检查“键”，以区分 update({}) 和 update((a, b))。以前，传递具有元组作为键的字典将被误解为序列。

    参考：[#2275](https://www.sqlalchemy.org/trac/ticket/2275)

## 0.7.2

发布日期：2011 年 7 月 31 日（星期日）

### orm

+   **[orm]**

    功能增强：现在，对已存在的相关对象和集合进行连接和子查询加载将会在已定义的急加载范围内遍历，以查找未填充的属性，从而使通过映射或查询选项指定的急加载无条件地进行到底，填充尚未填充的内容。此前，如果已存在相关对象或集合，则此遍历将停止，导致不一致的行为（尽管对于已加载的图表可以节省加载/循环）。对于子查询加载，这意味着子查询加载发出的额外的 SELECT 语句将无条件地调用，无论现有图表的多少（因此引起了争议）。当查询是由属性发起的延迟加载的结果时，之前的“停止”行为仍然有效，否则当重复遇到相同的相关对象时，“N+1”风格的集合迭代会变得不必要昂贵。还有一个尚未公开的生成式 Query 方法 _with_invoke_all_eagers()，它选择了旧/新的行为。

    参考：[#2213](https://www.sqlalchemy.org/trac/ticket/2213)

+   **[orm]**

    在 ORM 中对“替换遍历”进行了重新设计，因为它改变了可选择的内容以针对事物的别名（即子句调整），包括针对联接表结构的多重嵌套 any()/has() 构造的修复。

    参考：[#2195](https://www.sqlalchemy.org/trac/ticket/2195)

+   **[orm]**

    修复了在具有 join 条件的关系的 join() + aliased=True 从联接继承结构到自身的 query.join() 时，不适当地将主要实体转换为联接实体的错误。同样适用于 0.6.9 版本。

    参考：[#2234](https://www.sqlalchemy.org/trac/ticket/2234)

+   **[orm]**

    修复了从 0.6 版本中的回归，其中对包含集合中的 None 的对象执行 Session.add() 将引发内部异常。将此回滚到 0.6 版本的行为，即接受 None，但显然不会持久化任何内容。理想情况下，存在 None 的集合或在 append() 上的集合至少应该发出警告，这将在 0.8 版本中考虑。

    参考：[#2205](https://www.sqlalchemy.org/trac/ticket/2205)

+   **[orm]**

    在无法定位行的对象上加载了延迟属性引发 ObjectDeletedError，而不是稍后失败；改进了 ObjectDeletedError 中的消息，以包括除简单的“删除”外的其他条件。

    参考：[#2191](https://www.sqlalchemy.org/trac/ticket/2191)

+   **[orm]**

    修复了从 0.6 版本中出现的回归，当某些基于 relationship() 的属性上的 get history 操作失败时，当懒加载发出时；在某些条件下，这可能会在 flush() 中触发。感谢提交了这个出色测试的用户。

    参考：[#2224](https://www.sqlalchemy.org/trac/ticket/2224)

+   **[orm]**

    修复了在 Python 3 中显现的错误，即在 flush 期间对持久化 + 待定对象进行排序将产生非法比较，如果持久化对象的主键不是单个整数。也在 0.6.9 版中。

    参考：[#2228](https://www.sqlalchemy.org/trac/ticket/2228)

+   **[orm]**

    修复了一个错误，即如果 query.join() 针对将多个实体组合在一起的列表达式，那么所使用的源子句将不一致。也在 0.6.9 版中。

    参考：[#2197](https://www.sqlalchemy.org/trac/ticket/2197)

+   **[orm]**

    修复了一个错误，即如果映射类重新定义了 __hash__() 或 __eq__() 为非标准内容，这是一个受支持的用例，因为 SQLA 不应该查阅这些内容，那么如果该类是“复合”（即非单实体）结果集的一部分，则方法将被查阅。也在 0.6.9 版中。

    参考：[#2215](https://www.sqlalchemy.org/trac/ticket/2215)

+   **[orm]**

    向 Mapper 添加了公共属性“.validators”，这是所有已用@validates 装饰器装饰过的属性的不可变字典视图。由 Stefano Fontanelli 提供。

    参考：[#2240](https://www.sqlalchemy.org/trac/ticket/2240)

+   **[orm]**

    修复了一个微妙的错误，如果：针对子查询的 column_property() + joinedload + LIMIT + 按列属性排序，会导致 SQL 出现问题。也在 0.6.9 版中。

    参考：[#2188](https://www.sqlalchemy.org/trac/ticket/2188)

+   **[orm]**

    与 with_parent 生成的连接条件以及针对父级使用“动态”关系时将生成唯一的 bindparams，而不是错误地重复相同的 bindparam。也在 0.6.9 版中。

    参考：[#2207](https://www.sqlalchemy.org/trac/ticket/2207)

+   **[orm]**

    向 mapper.polymorphic_on 添加了与接收到的用户参数到 relationship.order_by、foreign_keys、remote_side 等时使用的“仅列”检查相同的检查。

+   **[orm]**

    修复了一个错误，即对比列表达式和 Query() 时不会调用底层 SELECT 语句的 as_scalar() 来生成标量子查询，方式与调用 Query().subquery() 时发生的方式相同。

    参考：[#2190](https://www.sqlalchemy.org/trac/ticket/2190)

+   **[orm]**

    修复了声明性错误，其中从相同名称的超类继承的类会由于在 _decl_class_registry 中进行不必要的查找而失败。

    参考：[#2194](https://www.sqlalchemy.org/trac/ticket/2194)

+   **[orm]**

    修复了 Query 中“无语句条件”断言的 bug，如果在调用 from_statement()之后调用生成方法，则会尝试引发异常。也在 0.6.9 版本中。

    参考：[#2199](https://www.sqlalchemy.org/trac/ticket/2199)

### 示例

+   **[examples]**

    修复了示例/versioning 测试运行程序，不再依赖 SQLAlchemy 测试库，必须从 examples/versioning 中运行 nosetests 以避免 setup.cfg 破坏它。

+   **[examples]**

    调整示例/versioning 以在多级继承情况下选择正确的外键。

+   **[examples]**

    修复了属性 shard 示例，在 0.7 风格中正确检查绑定参数可调用的问题。

### engine

+   **[engine]**

    Connection.begin()提供的上下文管理器如果提交失败将发出 rollback()，不仅在发生异常时。

+   **[engine]**

    在 Python 2.6 及以上版本中使用 urllib.parse_qsl()，不会有关于 cgi.parse_qsl()的弃用警告。

    参考：[#1682](https://www.sqlalchemy.org/trac/ticket/1682)

+   **[engine]**

    添加了混合类 sqlalchemy.ext.DontWrapMixin。当用户定义的此类型异常在语句执行上下文中发生时，永远不会被包装在 StatementException 中。

+   **[engine]**

    StatementException 包装将在消息中显示原始异常类。

+   **[engine]**

    连接失败引发 dbapi.Error 的情况将将错误转发给 dialect.is_disconnect()，如果方言知道这是一个可能的“可重试”条件，则设置“connection_invalidated”标志。目前只实现了 Oracle ORA-01033。

    参考：[#2201](https://www.sqlalchemy.org/trac/ticket/2201)

### sql

+   **[sql]**

    修复了在 selectable 中涉及列对应的两个微妙 bug，一个是重复使用相同标记的子查询，另一个是当标记被“分组”并丢失时。影响。

    参考：[#2188](https://www.sqlalchemy.org/trac/ticket/2188)

### schema

+   **[schema]**

    新功能：所有类型上的 with_variant()方法。生成 Variant()的实例，一个特殊的 TypeDecorator，根据使用的方言选择不同类型的用法。

    参考：[#2187](https://www.sqlalchemy.org/trac/ticket/2187)

+   **[schema]**

    当 ForeignKeyConstraint 引用父级中不存在的列名时，添加了一个信息丰富的错误消息。也在 0.6.9 版本中。

+   **[schema]**

    修复了旧的 append_ddl_listener()函数适配的 bug，将意外的**kw 传递给 Table 事件。Table 不接受 kw 参数，0.6 中的 MetaData 事件将获得“tables=somecollection”，此行为得以保留。

    参考：[#2206](https://www.sqlalchemy.org/trac/ticket/2206)

+   **[schema]**

    修复了 Table 上“autoincrement”检测失败的 bug，如果类型没有“affinity”值，则会发生这种情况，特别是在使用 TypeEngine 作为“impl”的站点上使用 UUID 示例时会发生这种情况。

+   **[schema]**

    为 TypeEngine 对象添加了改进的 repr()，只显示构造函数参数中的位置参数或与默认值不同的 kwargs。

    参考：[#2209](https://www.sqlalchemy.org/trac/ticket/2209)

### postgresql

+   **[postgresql]**

    为 Index 添加了新的“postgresql_ops”参数，允许为索引列指定 PostgreSQL 操作符类。感谢 Filip Zyzniewski。

    参考：[#2198](https://www.sqlalchemy.org/trac/ticket/2198)

### mysql

+   **[mysql]**

    修复了 OurSQL 方言在 XA 命令中使用 ansi-neutral 引号符“’”而不是‘”’的问题。也在 0.6.9 版本中。

    参考：[#2186](https://www.sqlalchemy.org/trac/ticket/2186)

### sqlite

+   **[sqlite]**

    SQLite 方言不再剥离反射默认值的引号，允许往返 CREATE TABLE 正常工作。这与其他方言一致，它们也保持默认值的确切形式。

    参考：[#2189](https://www.sqlalchemy.org/trac/ticket/2189)

### mssql

+   **[mssql]**

    调整了 pyodbc 方言，如果检测到“Easysoft” unix 驱动程序，则绑定值将作为字节而不是 Unicode 传递。这与 FreeTDS 发生的行为相同。在某些情况下，Easysoft 似乎会在传递 Python Unicode 时崩溃。

### oracle

+   **[oracle]**

    将 ORA-00028 添加到断开代码中，使用 cx_oracle _Error.code 来获取代码。也在 0.6.9 版本中。

    参考：[#2200](https://www.sqlalchemy.org/trac/ticket/2200)

+   **[oracle]**

    添加了 ORA-01033 到断开代码中，可以在连接事件中捕获。

    参考：[#2201](https://www.sqlalchemy.org/trac/ticket/2201)

+   **[oracle]**

    修复了 oracle.RAW 类型未生成正确 DDL 的问题。也在 0.6.9 版本中。

    参考：[#2220](https://www.sqlalchemy.org/trac/ticket/2220)

+   **[oracle]**

    将 CURRENT 添加到保留字列表中。也在 0.6.9 版本中。

    参考：[#2212](https://www.sqlalchemy.org/trac/ticket/2212)

+   **[oracle]**

    修复了可变扩展中的一个错误，即如果在一个映射中两次使用相同类型，则第一个之后的属性不会被检测。

+   **[oracle]**

    修复了可变扩展中的一个错误，即如果设置为 None 或非对应类型，则会引发错误。现在接受 None，将 None 分配给所有属性，非法值会引发 ValueError。

## 0.7.1

发布日期：2011 年 6 月 5 日 星期日

### 通用

+   **[general]**

    为 Python bug 7511 添加了一个解决方法，在 Windows 64 位+ VC express 上，C 扩展构建失败不会引发适当的异常。

    参考：[#2184](https://www.sqlalchemy.org/trac/ticket/2184)

### orm

+   **[orm]**

    现在允许在自引用关系上使用“delete-orphan”级联 - 这是因为 SQLA 0.7 不再在 ORM 级别强制执行“父项没有子项”的检查；此检查留给外键的可空性。相关链接

    参考：[#1912](https://www.sqlalchemy.org/trac/ticket/1912)

+   **[orm]**

    修复了新的“mutable”扩展以正确传播事件到子类；也不要为子类创建多个事件监听器。

    参考：[#2180](https://www.sqlalchemy.org/trac/ticket/2180)

+   **[orm]**

    修改了在刷新时未检测到“identity”键时出现的消息文本，以包括列未正确设置以正确检测自增的常见原因；也在 0.6.8 版本中。

    参考：[#2170](https://www.sqlalchemy.org/trac/ticket/2170)

+   **[orm]**

    修复了事务级别“deleted”集合不会清除已删除状态的 bug，如果它们后来变为瞬态，则会引发错误。也在 0.6.8 版本中。

    参考：[#2182](https://www.sqlalchemy.org/trac/ticket/2182)

### engine

+   **[engine]**

    废弃 Connection/Engine 上从未被广泛知晓且多余的 schema/SQL 导向方法：reflecttable()、create()、drop()、text()、engine.func

+   **[engine]**

    调整了 RowProxy 结果行的 __contains__() 方法，使其在内部不会生成异常抛出；无论列构造是否可以强制转换为字符串，NoSuchColumnError() 也将生成其消息。也在 0.6.8 版本中。

    参考：[#2178](https://www.sqlalchemy.org/trac/ticket/2178)

### sql

+   **[sql]**

    修复了 metadata.reflect(bind) 会关闭传递为绑定参数的 Connection 的 bug。从 0.6 版本中的回归。

+   **[sql]**

    简化了 Select 确定其‘.c’集合中内容的过程。行为完全相同，只是现在传递给 select([]) 的原始 ClauseList()（这本来就不是一个文档化的情况）将被扩展为其各个列元素，而不是被忽略。

### postgresql

+   **[postgresql]**

    修复了关于数字数组、MATCH 运算符的一些单元测试问题。修复了潜在的浮点精度问题，并且目前仅在 EN 本地环境中执行 MATCH 运算符的某些测试。也在 0.6.8 版本中。

    参考：[#2175](https://www.sqlalchemy.org/trac/ticket/2175)

### mysql

+   **[mysql]**

    单元测试在安装在 Windows 上的 MySQL 上通过率达到 100%。

+   **[mysql]**

    移除了在反射具有混合大小写名称的 MySQL 表时在 Windows 上“调整大小写”步骤。经过一些对 Windows MySQL 服务器的实验后，确定这一步骤实际上并没有帮助太多；MySQL 在非 Windows 平台上也不会返回正确大小写的 FK 名称，移除这一步骤至少使反射更像在其他操作系统上的行为。考虑过在此处发出警告，但很难确定在什么条件下可以引发这样的警告，因此暂时搁置 - 而是添加了一些文档。

    参考：[#2181](https://www.sqlalchemy.org/trac/ticket/2181)

+   **[mysql]**

    如果使用 MySQLdb 并且 DBAPI 不提供 constants.CLIENT 模块，则 supports_sane_rowcount 将设置为 False。

### sqlite

+   **[sqlite]**

    在调用“PRAGMA read_uncommitted”以确定连接时的当前隔离模式并默认为 SERIALIZABLE 时，接受来自 cursor.fetchone() 的 None；这是为了支持 SQLite 3.3.0 之前的版本，这些版本没有这个功能。

    参考：[#2173](https://www.sqlalchemy.org/trac/ticket/2173)

## 0.7.0

发布日期：2011 年 5 月 20 日 星期五

### orm

+   **[orm]**

    修复了在 0.7b4 中引入的回归问题，即 query.options(someoption(“nonexistent name”)) 将无法引发错误。还为尝试基于基于列的元素构建选项的情况添加了额外的错误捕获，进一步修正了一些定制的错误消息

    参考：[#2069](https://www.sqlalchemy.org/trac/ticket/2069)

+   **[orm]**

    query.count() 发出“count(*)”而不是“count(1)”。

    参考：[#2162](https://www.sqlalchemy.org/trac/ticket/2162)

+   **[orm]**

    对查询子句适应进行微调，当使用 from_self()、union() 或其他“从自身选择”操作时，添加到 filter()、order_by() 等中的普通 SQL 表达式元素将以与 ORM 表达式元素相同的方式进行适应，因为这些元素通常不容易访问。

    参考：[#2155](https://www.sqlalchemy.org/trac/ticket/2155)

+   **[orm]**

    修复了“自引用”关系的确定会失败的 bug，对于没有解决方案的与自身相关的 joined-inh 子类，或者与没有在连接条件中的子子类的子类相关的 joined-inh 子类。同样在 0.6.8 版本中。

    参考：[#2149](https://www.sqlalchemy.org/trac/ticket/2149)

+   **[orm]**

    mapper() 在确定父子类之间的继承条件时，会忽略与无关表的非配置外键，但对于未解析的列和关于继承表的表名，会像往常一样引发异常。这是对先前应用于声明性的行为的增强泛化。0.6.8 版本有一个更保守的版本，不会从根本上改变确定连接条件的方式。

    参考：[#2153](https://www.sqlalchemy.org/trac/ticket/2153)

+   **[orm]**

    在给定实体不是单个完整类实体或映射器（即列）时调用 query.get() 是一个错误。这是在 0.6.8 版本中的一个弃用警告。

    参考：[#2144](https://www.sqlalchemy.org/trac/ticket/2144)

+   **[orm]**

    修复了在某些情况下可能发生的与标识映射相关的潜在 KeyError，部分

    参考：[#2148](https://www.sqlalchemy.org/trac/ticket/2148)

+   **[orm]**

    添加了 Query.with_session() 方法，将 Query 切换到使用不同的会话。

+   **[orm]**

    水平分片查询应根据每个连接使用执行选项

    参考：[#2131](https://www.sqlalchemy.org/trac/ticket/2131)

+   **[orm]**

    非主映射器将继承主映射器的 _identity_class。这样，针对通常处于继承映射中的类建立的非主映射器将产生与主映射器兼容的标识映射结果（也在 0.6.8 版本中）。

    参考：[#2151](https://www.sqlalchemy.org/trac/ticket/2151)

+   **[orm]**

    修复了“无法为目标列‘q’执行同步规则；映射‘X’未映射此列”发出的错误消息，以引用正确的映射。同样在 0.6.8 版本中。

    参考：[#2163](https://www.sqlalchemy.org/trac/ticket/2163)

+   **[orm]**

    polymorphic_union() 现在有一个“cast_nulls”选项，当渲染带标签的 NULL 列时禁用 CAST 的使用。

    参考：[#1502](https://www.sqlalchemy.org/trac/ticket/1502)

+   **[orm]**

    polymorphic_union() 根据它们在多态联合列表中出现的第一个表/可选择的顺序呈现列。（除非传递了 OrderedDict，否则它本身是无序映射）。

+   **[orm]**

    修复了一个 bug，即如果 mapper 映射到匿名别名，如果使用了日志记录，由于别名中未转义的 % 符号，将会失败。也在 0.6.8 版本中修复。

    参考：[#2171](https://www.sqlalchemy.org/trac/ticket/2171)

### 示例

+   **[examples]**

    删除了古老的“多态关联”示例，并替换为使用声明性 mixin、“generic_associations”的更新示例集。每个示例呈现了一种替代的表布局。

### sql

+   **[sql]**

    修复了一个 bug，即在 select() 中嵌套另一个带有标签的 select() 会产生不正确的导出列。其中之一是会破坏针对另一个 column_property() 的 ORM column_property() 映射。也在 0.6.8 版本中修复。

    参考：[#2167](https://www.sqlalchemy.org/trac/ticket/2167)

+   **[sql]**

    更改了确定连接条件的处理方式，使得外键错误仅在两个给定表之间考虑。也就是说，t1.join(t2) 将报告涉及‘t1’或‘t2’的 FK 错误，但涉及‘t3’的任何内容将被跳过。这影响 join()，以及 ORM 关系和继承条件逻辑。

+   **[sql]**

    在执行过程中对错误处理进行了一些改进，以确保在发生非常不寻常的 DBAPI 错误时确实关闭自动关闭的连接。

+   **[sql]**

    metadata.reflect() 和 reflection.Inspector() 在关闭内部获取的连接时有一些依赖于 GC，已修复。

+   **[sql]**

    添加了对 Column .name 赋空字符串时的显式检查。

    参考：[#2140](https://www.sqlalchemy.org/trac/ticket/2140)

+   **[sql]**

    修复了一个 bug，即如果将 FetchedValue 传递给 column server_onupdate，它将不会被分配给其父“column”，为所有列默认分配模式添加了测试覆盖。也在 0.6.8 版本中修复。

    参考：[#2147](https://www.sqlalchemy.org/trac/ticket/2147)

### postgresql

+   **[postgresql]**

    修复了 psycopg2 方言中 psycopg2_version 解析的问题。

+   **[postgresql]**

    修复了影响 PG 9 的 bug，当反射一个列名已更改的列时，索引反射会失败。也在 0.6.8 版本中修复。

    参考：[#2141](https://www.sqlalchemy.org/trac/ticket/2141)

### mssql

+   **[mssql]**

    修复了 MSSQL 方言中的 bug，即对架构限定表进行别名处理会泄漏到封闭的 select 语句中。也在 0.6.8 版本中修复。

    参考：[#2169](https://www.sqlalchemy.org/trac/ticket/2169)

### 杂项

+   **[no_tags]**

    本节记录了从 0.7b4 到 0.7.0 的更改。有关 SQLAlchemy 0.7 的新功能概述，请参阅[`docs.sqlalchemy.org/en/latest/changelog/migration_07.html`](https://docs.sqlalchemy.org/en/latest/changelog/migration_07.html)

+   **[documentation]**

    从 ext.mutable 文档中删除了对“collections.MutableMapping” abc 的使用，因为它被错误使用，并且在任何情况下都使示例更难理解。

    参考：[#2152](https://www.sqlalchemy.org/trac/ticket/2152)

+   **[ext]**

    修复了 sqlalchemy.ext.mutable 扩展中的一些错误，其中未适当处理 None，替换事件也未适当处理。

    参考：[#2143](https://www.sqlalchemy.org/trac/ticket/2143)

## 0.7.0b4

发布日期：2011 年 4 月 17 日 星期日

### general

+   **[general]**

    对 CHANGES 文件的格式进行了更改。这些格式更改已应用于 0.7 版本的发布。

+   **[general]**

    “-declarative”更改现在将直接列在“-orm”部分下面，因为它们密切相关。

+   **[general]**

    0.5 系列的更改已移至文件 CHANGES_PRE_06，取代了 CHANGES_PRE_05。

+   **[general]**

    0.6.7 版本及其后续版本的更改日志现在仅在 0.6 分支的 CHANGES 文件中列出。在 0.7 版本的 CHANGES 文件（即本文件）中，所有 0.6 的更改都内联列在它们被应用的 0.7 部分中（因为所有 0.6 的更改也在 0.7 中）。这里适用于 0.6 版本的更改被记录，如果存在实现/行为上的任何差异也会被注明。

### orm

+   **[orm]**

    当调用 query.update()、query.delete()时，对“evaluate”和“fetch”评估进行了一些修复。在所有情况下，记录的检索都是在自动刷新之后进行的，并在发出更新/删除之前进行，以防止未刷新的数据存在以及在评估过程中出现过期对象失败。

    参考：[#2122](https://www.sqlalchemy.org/trac/ticket/2122)

+   **[orm]**

    重新修订了尝试刷新非多态子类的异常抛出时的异常。

    参考：[#2063](https://www.sqlalchemy.org/trac/ticket/2063)

+   **[orm]**

    当查询选项无法找到目标实体时，对于路径必须从根实体之一开始的情况进行了更多措辞调整。解释了路径必须从根实体之一开始。

+   **[orm]**

    关于反向引用(backrefs)的状态处理进行了一些修复，通常在`autoflush=False`时，当反向引用的集合没有净变化时，无法正确处理添加/删除操作。感谢 Richard Murri 提供的测试用例和补丁（也适用于 0.6.7 版本）。

    参考：[#2123](https://www.sqlalchemy.org/trac/ticket/2123)

+   **[orm]**

    在 UOW 内部添加了检查，以检测在主键值中包含 NULL 的异常条件被要求进行 UPDATE 或 DELETE 的异常条件。

    参考：[#2127](https://www.sqlalchemy.org/trac/ticket/2127)

+   **[orm]**

    对属性历史进行了一些细微调整。更多的变化可能在 0.8 版本中进行，但目前历史已经被修改，使得标量历史不会为不存在的值填充 None 带来“副作用”。这样稍微更好地区分了 None 设置和实际变化的能力，也会受到影响。

    参考：[#2127](https://www.sqlalchemy.org/trac/ticket/2127)

+   **[orm]**

    如果使用了 from_self()，则“having”子句将从内部复制到外部查询；特别是这会破坏 0.7 风格的 count()查询。（也适用于 0.6.7 版本）

    参考：[#2130](https://www.sqlalchemy.org/trac/ticket/2130)

+   **[orm]**

    Query.execution_options()方法现在将这些选项传递给 Connection 而不是 SELECT 语句，以便可以使用所有可用选项，包括隔离级别��编译缓存。

    参考：[#2131](https://www.sqlalchemy.org/trac/ticket/2131)

### engine

+   **[engine]**

    C 扩展现在在 CPython 2.x 上默认启用，如果编译失败则回退到纯 Python。

    参考：[#2129](https://www.sqlalchemy.org/trac/ticket/2129)

### sql

+   **[sql]**

    “compiled_cache”执行选项现在在传递给 SELECT 语句而不是 Connection 时会引发错误。以前它完全被忽略。我们可能会考虑在某个时候使此选项在每个语句级别上工作。

    参考：[#2131](https://www.sqlalchemy.org/trac/ticket/2131)

+   **[sql]**

    恢复了基本 TypeEngine 类上的“catchall”构造函数，并附带弃用警告。这样，像 Integer(11)这样的代码仍然可以成功执行。

+   **[sql]**

    修复了反序列化后的 MetaData()未能跟踪新内容的回归问题，即 Sequence 对象的集合、模式名称列表。

    参考：[#2104](https://www.sqlalchemy.org/trac/ticket/2104)

+   **[sql]**

    选择`select()`中的 limit/offset 关键字以及传递给`select.limit()/offset()`的值将被强制转换为整数。（也适用于 0.6.7 版本）

    参考：[#2116](https://www.sqlalchemy.org/trac/ticket/2116)

+   **[sql]**

    修复了从 over()子句中收集的“from”子句将成为 itertools.chain()而不是列表的错误，导致与其他子句组合时出现“can only concatenate list” TypeError。

+   **[sql]**

    修复了在 over()子句中错误使用“,”放置在“partition”和“order by”子句之间的问题。

    参考：[#2134](https://www.sqlalchemy.org/trac/ticket/2134)

+   **[sql]**

    PrimaryKeyConstraint 的 before/after attach 事件现在起作用，为所有约束类型添加了 before/after 事件的测试。

    参考：[#2105](https://www.sqlalchemy.org/trac/ticket/2105)

+   **[sql]**

    在表达式库中添加了显式的 true()/false()构造 - 强制转换规则将“False”/“True”拦截到这些构造中。在 0.6 版本中，这些构造通常直接转换为字符串，而在 0.7 中不再接受。

    参考：[#2117](https://www.sqlalchemy.org/trac/ticket/2117)

### schema

+   **[schema]**

    Table 上的 ‘useexisting’ 标志已被新的 ‘keep_existing’ 和 ‘extend_existing’ 一对标志取代。‘extend_existing’ 等同于 ‘useexisting’ - 返回现有的 Table，并添加额外的构造元素。使用 ‘keep_existing’ 时，返回现有的 Table，但不添加额外的构造元素 - 这些元素仅在新创建 Table 时应用。

    参考：[#2109](https://www.sqlalchemy.org/trac/ticket/2109)

### postgresql

+   **[postgresql]**

    现在支持 Python 3 的 Psycopg2。

+   **[postgresql]**

    修复了在使用 pg8000 时支持精度数值的问题。

    参考：[#2132](https://www.sqlalchemy.org/trac/ticket/2132)

### sqlite

+   **[sqlite]**

    修复了反射外键时创建为 “REFERENCES <tablename>” 而没有列名的 bug。（也在 0.6.7 中）

    参考：[#2115](https://www.sqlalchemy.org/trac/ticket/2115)

### oracle

+   **[oracle]**

    现在在与 cx_oracle 通信时，使用需要为列本身或名称生成的绑定参数引号的列名，例如具有特殊字符、下划线、非 ASCII 字符的列名，正确地转换绑定参数键。（也在 0.6.7 中）

    参考：[#2100](https://www.sqlalchemy.org/trac/ticket/2100)

+   **[oracle]**

    Oracle 方言添加了 use_binds_for_limits=False create_engine() 标志，将 LIMIT/OFFSET 值内联呈现，而不是作为绑定，据报告修改了 Oracle 使用的执行计划。（也在 0.6.7 中）

    参考：[#2116](https://www.sqlalchemy.org/trac/ticket/2116)

### misc

+   **[types]**

    REAL 已添加到核心类型。由 PostgreSQL、SQL Server、MySQL、SQLite 支持。请注意，SQL Server 和 MySQL 版本，添加了额外的参数，仍然可以从这些方言中使用。

    参考：[#2081](https://www.sqlalchemy.org/trac/ticket/2081)

+   **[types]**

    添加了 @event.listens_for() 装饰器，给定目标 + 事件名称，将装饰的函数应用为监听器。

    参考：[#2106](https://www.sqlalchemy.org/trac/ticket/2106)

+   **[pool]**

    AssertionPool 现在存储了当前已检出连接获取的回溯信息；在第二次并发检出时，此回溯信息将在引发的断言中报告；由 Gunnlaugur Briem 提供

    参考：[#2103](https://www.sqlalchemy.org/trac/ticket/2103)

+   **[pool]**

    “pool.manage” 功能不再使用 pickle 来为每个 pool 哈希参数。

+   **[documentation]**

    记录了 SQLite DATE/TIME/DATETIME 类型。（也在 0.6.7 中）

    参考：[#2029](https://www.sqlalchemy.org/trac/ticket/2029)

+   **[documentation]**

    修正了可变扩展文档，以显示正确的类型关联方法。

    参考：[#2118](https://www.sqlalchemy.org/trac/ticket/2118)

## 0.7.0b3

发布日期：2011 年 3 月 20 日 星期日

### general

+   **[general]**

    在 PyPy 下运行时修复了许多单元测试问题（由 Alex Gaynor 提供）。

### orm

+   **[orm]**

    更改了查询.count() 的基础��法。现在，在所有情况下，查询.count() 精确地为：

    > query.
    > 
    > from_self(func.count(literal_column(‘1’))). scalar()

    也就是说，“select count(1) from (<完整查询>)”。这在所有情况下都会产生一个子查询，但大大简化了之前 count() 尝试做的所有猜测，之前在许多情况下仍然会失败，特别是当涉及联接表继承和其他联接时。如果为否则非常简单的计数生成的子查询真的是一个问题，请使用 query(func.count()) 进行优化。

    参考：[#2093](https://www.sqlalchemy.org/trac/ticket/2093)

+   **[orm]**

    关于迭代过程中罕见的弱引用回调的身份映射发生了一些变化。已删除互斥体，因为显然可能会导致重入（即在一个线程中）死锁，也许是在迭代过程中 gc 在获取更多内存时收集对象时发生。希望“迭代过程中字典发生变化”会非常罕见，因为迭代方法在内部通过单个 values() 调用获取完整的对象列表。请注意，0.6.7 在这里有一个更为保守的修复，仍然保留了互斥体。

    参考：[#2087](https://www.sqlalchemy.org/trac/ticket/2087)

+   **[orm]**

    对工作单元进行微调，使其按照 relationship() 的依赖关系排序刷新，即使给定的对象在内存中没有任何属性间的引用，这是 0.5 及更早版本的行为，因此只设置外键/主键的 Parent/Child 刷新将成功。同时，仍然保持 0.6 及以上版本在刷新时不生成大量与当前刷新状态实际不符的无用内部依赖结构。

    参考：[#2082](https://www.sqlalchemy.org/trac/ticket/2082)

+   **[orm]**

    在针对仅包含列的实体进行查询时与（通常不正确地）使用加载器选项一起使用时，改进了发出的错误消息，其中父实体不完全存在。

    参考：[#2069](https://www.sqlalchemy.org/trac/ticket/2069)

+   **[orm]**

    修复了在查询选项（query.options()）中的一个 bug，即应用于使用字符串键的 lazyload 的路径可能会与错误实体上的同名属性重叠。请注意，0.6.7 对此进行了更为保守的修复。

    参考：[#2098](https://www.sqlalchemy.org/trac/ticket/2098)

### 例子

+   **[例子]**

    更新了关联、关联代理示例以使用声明式，并添加了一个新的示例 dict_of_sets_with_default.py，这是一个关联代理的“突破极限”示例。

+   **[例子]**

    Beaker 缓存示例允许在 query_callable() 函数中传递一个“query_cls”参数。（也适用于 0.6.7）

    参考：[#2090](https://www.sqlalchemy.org/trac/ticket/2090)

### 引擎

+   **[引擎]**

    修复了 AssertionPool 的回归 bug。

    参考：[#2097](https://www.sqlalchemy.org/trac/ticket/2097)

+   **[引擎]**

    当指定无效的方言时，将引发 ArgumentError 异常。

    参考：[#2060](https://www.sqlalchemy.org/trac/ticket/2060)

### sql

+   **[sql]**

    在 Column 被子类化且 _make_proxy() 由于构造函数的 TypeError 失败而无法进行复制时，添加了一个完整的描述性错误消息。在这种情况下应该实现 _constructor 方法。

+   **[sql]**

    为 Table 对象添加了新的事件“column_reflect”。在反射内生成对象之前接收有关 Column 的信息字典，并允许修改字典以控制生成的 Column 的大多数方面，包括键、名称、类型、信息字典。

    参考：[#2095](https://www.sqlalchemy.org/trac/ticket/2095)

+   **[sql]**

    为了帮助“column_reflect”事件与特定的 Table 对象一起使用而不是所有 Table 实例，可以在 Table 的构造过程中使用一个新的参数“listeners”来添加监听器，一个元组列表 (<eventname>, <fn>)，它们在反射过程开始之前被应用于 Table。

+   **[sql]**

    添加了新的通用函数“next_value()”，接受一个 Sequence 对象作为参数，并在目标平台上渲染适当的“next value”生成字符串，如果支持的话。也在 Sequence 本身上提供了“.next_value()”方法。

    参考：[#2085](https://www.sqlalchemy.org/trac/ticket/2085)

+   **[sql]**

    func.next_value() 或其他 SQL 表达式可以直接嵌入到 insert() 构造中，如果隐式或显式地与主键列一起使用“returning”，则新生成的值将出现在 result.inserted_primary_key 中。

    参考：[#2084](https://www.sqlalchemy.org/trac/ticket/2084)

+   **[sql]**

    添加了对 ResultProxy 的访问器“returns_rows”、“is_insert” (也在 0.6.7 中)

    参考：[#2089](https://www.sqlalchemy.org/trac/ticket/2089)

### postgresql

+   **[postgresql]**

    为 postgresql 方言添加了 RESERVED_WORDS。(也在 0.6.7 中)

    参考：[#2092](https://www.sqlalchemy.org/trac/ticket/2092)

+   **[postgresql]**

    修复了 BIT 类型以允许“length”参数、“varying”参数。反射也已修复。(也在 0.6.7 中)

    参考：[#2073](https://www.sqlalchemy.org/trac/ticket/2073)

### mssql

+   **[mssql]**

    重写了用于获取视图定义的查询，通常在使用 Inspector 接口时使用 sys.sql_modules 而不是信息模式，从而允许超过 4000 个字符的视图定义被完全返回。(也在 0.6.7 中)

    参考：[#2071](https://www.sqlalchemy.org/trac/ticket/2071)

### 杂项

+   **[declarative]**

    `__mapper_args__` 中的非“可哈希”参数不会被误认为总是可哈希的，可能是列参数。(也在 0.6.7 中)

    参考：[#2091](https://www.sqlalchemy.org/trac/ticket/2091)

+   **[firebird]**

    如果将“implicit_returning”标志设置为 False，则 create_engine() 上的“implicit_returning”标志将被尊重。(也在 0.6.7 中)

    参考：[#2083](https://www.sqlalchemy.org/trac/ticket/2083)

+   **[informix]**

    添加了 RESERVED_WORDS informix 方言。(也在 0.6.7 中)

    参考：[#2092](https://www.sqlalchemy.org/trac/ticket/2092)

+   **[ext]**

    horizontal_shard ShardedSession 类接受公共 Session 参数“query_cls”作为构造函数参数，以便进一步对 ShardedQuery 进行子类化（也适用于 0.6.7）。

    参考：[#2090](https://www.sqlalchemy.org/trac/ticket/2090)

## 0.7.0b2

发布日期：2011 年 2 月 19 日

### orm

+   **[orm]**

    修复了一个 bug，即 Session.merge() 会调用 load() 事件但参数少了一个。

    参考：[#2053](https://www.sqlalchemy.org/trac/ticket/2053)

+   **[orm]**

    添加了逻辑，防止从 MapperExtension 或 SessionExtension 生成的事件为所有未覆盖的方法生成无用事件。

    参考：[#2052](https://www.sqlalchemy.org/trac/ticket/2052)

### 示例

+   **[examples]**

    在生成缓存键时，Beaker 示例现在考虑了嵌套的 FROM 子句中的 ‘limit’ 和 ‘offset’、绑定参数（比如在使用 union() 或 from_self() 时）。

### sql

+   **[sql]**

    将 EngineEvents 事件类重命名为 ConnectionEvents。由于这些类从未直接被终端用户代码访问，因此这严格来说是针对终端用户的文档更改。还简化了内部如何将事件与引擎和连接关联的方式。

    参考：[#2059](https://www.sqlalchemy.org/trac/ticket/2059)

+   **[sql]**

    当通过其 ‘metadata’ 参数传递一个 MetaData() 对象时，Sequence() 构造将包含在 CREATE/DROP 语句中，包括“checkfirst”逻辑。

    参考：[#2055](https://www.sqlalchemy.org/trac/ticket/2055)

+   **[sql]**

    Column.references() 方法现在在具有引用给定列的外键的情况下返回 True，而不仅仅是其父表。

    参考：[#2064](https://www.sqlalchemy.org/trac/ticket/2064)

### postgresql

+   **[postgresql]**

    从 0.6 版本开始修复了一个回归问题，其中 SMALLINT 和 BIGINT 类型都将在整数 PK 列上生成 SERIAL，而不是 SMALLINT 和 BIGSERIAL。

    参考：[#2065](https://www.sqlalchemy.org/trac/ticket/2065)

### 杂项

+   **[declarative]**

    修复了一个回归问题，即将 Column 对象与 composite() 内联将无法初始化。现在可以将 Column 对象与 composite() 内联或外部，并通过名称或对象引用引入。

    参考：[#2058](https://www.sqlalchemy.org/trac/ticket/2058)

+   **[declarative]**

    修复了错误消息，该消息引用了旧的 @classproperty 名称，现在引用 @declared_attr（也适用于 0.6.7）。

    参考：[#2061](https://www.sqlalchemy.org/trac/ticket/2061)

+   **[declarative]**

    __table_args__ 元组末尾的字典现在是可选的。

    参考：[#1468](https://www.sqlalchemy.org/trac/ticket/1468)

+   **[ext]**

    当代理一个多对一标量属性到一个一对多集合时，关联代理现在对 any()、has() 和 contains() 具有正确的行为（即‘典型’关联代理用例的反向）。

    参考：[#2054](https://www.sqlalchemy.org/trac/ticket/2054)

## 0.7.0b1

发布日期：2011 年 2 月 12 日星期六

### general

+   **[general]**

    新的事件系统，取代所有扩展、监听器等。

    参考：[#1902](https://www.sqlalchemy.org/trac/ticket/1902)

+   **[general]**

    日志增强

    参考：[#1926](https://www.sqlalchemy.org/trac/ticket/1926)

+   **[general]**

    设置不再安装 Nose 插件

    参考：[#1949](https://www.sqlalchemy.org/trac/ticket/1949)

+   **[general]**

    sys.modules 中的“sqlalchemy.exceptions”别名已被移除。基本 SQLA 异常可通过“from sqlalchemy import exc”获得。“exceptions”别名用于“exc”的“sqlalchemy”现在仍然存在，只是没有被打补丁到 sys.modules 中。

### orm

+   **[orm]**

    更简洁的查询.join(target, onclause)形式

    参考：[#1923](https://www.sqlalchemy.org/trac/ticket/1923)

+   **[orm]**

    混合属性，实现/取代同义词()

    参考：[#1903](https://www.sqlalchemy.org/trac/ticket/1903)

+   **[orm]**

    复合体的重写

    参考：[#2008](https://www.sqlalchemy.org/trac/ticket/2008)

+   **[orm]**

    变异事件扩展，取代“mutable=True”

    另请参阅

    变异事件扩展，取代“mutable=True”

+   **[orm]**

    PickleType 和 ARRAY 的可变性默认关闭

    参考：[#1980](https://www.sqlalchemy.org/trac/ticket/1980)

+   **[orm]**

    简化的多态 _on 赋值

    参考：[#1895](https://www.sqlalchemy.org/trac/ticket/1895)

+   **[orm]**

    允许刷新没有父级的孤立对象

    参考：[#1912](https://www.sqlalchemy.org/trac/ticket/1912)

+   **[orm]**

    调整刷新记账步骤，以在 autocommit=True 的情况下在提交之前发生。这允许 autocommit=True 与 expire_on_commit=True 正常工��，并且还允许后刷新会话钩子在与 autocommit=False 时相同的事务上下文中运行。

    参考：[#2041](https://www.sqlalchemy.org/trac/ticket/2041)

+   **[orm]**

    在刷新时生成警告，集合成员，标量引用不是刷新的一部分

    参考：[#1973](https://www.sqlalchemy.org/trac/ticket/1973)

+   **[orm]**

    非表派生结构可以映射

    参考：[#1876](https://www.sqlalchemy.org/trac/ticket/1876)

+   **[orm]**

    查询改进中的元组标签名称

    参考：[#1942](https://www.sqlalchemy.org/trac/ticket/1942)

+   **[orm]**

    映射列属性首先引用最具体的列

    参考：[#1892](https://www.sqlalchemy.org/trac/ticket/1892)

+   **[orm]**

    映射到具有两个或更多同名列的连接需要明确声明

    参考：[#1896](https://www.sqlalchemy.org/trac/ticket/1896)

+   **[orm]**

    Mapper 要求映射选择中存在多态 _on 列

    参考：[#1875](https://www.sqlalchemy.org/trac/ticket/1875)

+   **[orm]**

    compile_mappers()重命名为 configure_mappers()，简化配置内部

    参考：[#1966](https://www.sqlalchemy.org/trac/ticket/1966)

+   **[orm]**

    如果传递了一个 SQL FromClause 元素（即不是映射类），aliased()函数将返回 element.alias()，而不是在 AliasedClass 上引发错误。

    参考：[#2018](https://www.sqlalchemy.org/trac/ticket/2018)

+   **[orm]**

    Session.merge() 将检查传入状态的版本 id 与数据库的版本 id 是否匹配，假设映射使用版本 id 并且传入状态已分配版本 id，并在它们不匹配时引发 StaleDataError。

    参考：[#2027](https://www.sqlalchemy.org/trac/ticket/2027)

+   **[orm]**

    Session.connection()、Session.execute()接受‘bind’，允许执行/连接操作显式参与引擎的打开事务。

    参考：[#1996](https://www.sqlalchemy.org/trac/ticket/1996)

+   **[orm]**

    Query.join()、Query.outerjoin()、eagerload()、eagerload_all()等不再允许将属性列表作为参数（即 option([x, y, z])形式，自 0.5 版本起已弃用）。

+   **[orm]**

    ScopedSession.mapper 已移除（自 0.5 版本起已弃用）。

+   **[orm]**

    水平分片查询将“shard_id”放在上下文属性中，可以通过“load()”事件访问。

    参考：[#2031](https://www.sqlalchemy.org/trac/ticket/2031)

+   **[orm]**

    在多个实体上进行单个 contains_eager()调用将指示沿该路径的所有集合应该加载，而不是要求为每个端点分别进行不同的 contains_eager()调用（这从未被正确记录）。

    参考：[#2032](https://www.sqlalchemy.org/trac/ticket/2032)

+   **[orm]**

    在 orm.aliased()中使用的“name”字段现在在生成的 SQL 语句中呈现。

+   **[orm]**

    Session weak_instance_dict=False 已弃用。

    参考：[#1473](https://www.sqlalchemy.org/trac/ticket/1473)

+   **[orm]**

    在罕见情况下，如果在父对象被取消引用后发生附加或类似事件的情况，将引发异常，这将阻止父对象在会话中被标记为“脏”。在 0.6.6 版本中是一个警告。

    参考：[#2046](https://www.sqlalchemy.org/trac/ticket/2046)

+   **[orm]**

    Query.distinct() 现在接受列表达式作为*args，由 PostgreSQL 方言解释为 DISTINCT ON (<expr>)。

    参考：[#1069](https://www.sqlalchemy.org/trac/ticket/1069)

+   **[orm]**

    在 flush()期间对“多对一”关系加载进行了额外调整。在 0.6.6 版本中的更改（[ticket:2002]）要求在 flush 期间可能发生更多“不必要”的 m2o 加载。已添加额外的加载模式，以便在这种特定情况下发出的 SQL 被修剪回来，同时仍然检索 flush 所需的信息，以免遗漏任何内容。

    参考：[#2049](https://www.sqlalchemy.org/trac/ticket/2049)

+   **[orm]**

    作为传递给 attributes.get_history()的“passive”值应该是 attributes 包中定义的常量之一。发送 True 或 False 已弃用。

+   **[orm]**

    添加了一个 name 参数到 Query.subquery()，允许为别名对象分配一个固定的名称。（也适用于 0.6.7 版本）

    参考：[#2030](https://www.sqlalchemy.org/trac/ticket/2030)

+   **[orm]**

    当连接表继承映射器在本地映射表上没有主键（但在超类表上有主键）时，会发出警告。（也适用于 0.6.7 版本）

    参考：[#2019](https://www.sqlalchemy.org/trac/ticket/2019)

+   **[orm]**

    修复了多态层次结构中“中间”类如果没有指定“polymorphic_identity”列，就不会有“polymorphic_on”列的错误，导致刷新时出现奇怪的错误，从该目标查询时加载错误的类。在使用单表继承时也会发出正确的 WHERE 条件。（也适用于 0.6.7 版本）

    参考：[#2038](https://www.sqlalchemy.org/trac/ticket/2038)

+   **[orm]**

    修复了一个 bug，即具有 SQL 或服务器端默认值的列被排除在包含属性或排除属性的映射中，会导致 UnmappedColumnError。（也适用于 0.6.7 版本）

    参考：[#1995](https://www.sqlalchemy.org/trac/ticket/1995)

+   **[orm]**

    在罕见情况下，如果在父对象被取消引用后发生附加或类似事件的情况下，会发出警告，这会阻止父对象在会话中被标记为“脏”。这将在 0.7 版本中成为异常。（也适用于 0.6.7 版本）

    参考：[#2046](https://www.sqlalchemy.org/trac/ticket/2046)

### sql

+   **[sql]**

    添加了 over() 函数，FunctionElement 类的方法，生成 _Over() 构造，进而生成“窗口函数”，即“<window function> OVER (PARTITION BY <partition by>, ORDER BY <order by>)”。

    参考：[#1844](https://www.sqlalchemy.org/trac/ticket/1844)

+   **[sql]**

    LIMIT/OFFSET 子句现在使用绑定参数

    参考：[#805](https://www.sqlalchemy.org/trac/ticket/805)

+   **[sql]**

    select.distinct() 现在接受列表达式作为 *args，由 PostgreSQL 方言解释为 DISTINCT ON (<expr>)。请注意，通过将列表传递给 select() 的 distinct 关键字参数已经可以实现此功能。

    参考：[#1069](https://www.sqlalchemy.org/trac/ticket/1069)

+   **[sql]**

    select.prefix_with() 接受多个表达式（即 *expr），select() 的 'prefix' 关键字参数接受列表或元组。

+   **[sql]**

    将字符串传递给 select() 的 distinct 关键字参数以发出特殊的 MySQL 关键字（DISTINCTROW 等）已被弃用 - 请使用 prefix_with()。

+   **[sql]**

    TypeDecorator 与主键列一起使用

    参考：[#2005](https://www.sqlalchemy.org/trac/ticket/2005), [#2006](https://www.sqlalchemy.org/trac/ticket/2006)

+   **[sql]**

    DDL() 构造现在会转义百分号

    参考：[#1897](https://www.sqlalchemy.org/trac/ticket/1897)

+   **[sql]**

    Table.c / MetaData.tables 稍微调整，不允许直接变异

    参考：[#1893](https://www.sqlalchemy.org/trac/ticket/1893), [#1917](https://www.sqlalchemy.org/trac/ticket/1917)

+   **[sql]**

    传递给 bindparam() 的可调用对象不会被评估

    参考：[#1950](https://www.sqlalchemy.org/trac/ticket/1950)

+   **[sql]**

    types.type_map 现在是私有的，types._type_map

    参考：[#1870](https://www.sqlalchemy.org/trac/ticket/1870)

+   **[sql]**

    非公共的 Pool 方法已经加下划线标记

    参考：[#1982](https://www.sqlalchemy.org/trac/ticket/1982)

+   **[sql]**

    添加了 NULLS FIRST 和 NULLS LAST 支持。它作为 asc()和 desc()操作符的扩展实现，称为 nullsfirst()和 nullslast()。

    参考：[#723](https://www.sqlalchemy.org/trac/ticket/723)

+   **[sql]**

    Index()构造可以与 Table 定义内联创建，使用字符串作为列名，作为在 Table 之外创建索引的替代方法。

+   **[sql]**

    Connection 上的 execution_options()接受“isolation_level”参数，仅为该连接设置事务隔离级别，直到返回到连接池，对于支持它的后端（SQLite，PostgreSQL）

    参考：[#2001](https://www.sqlalchemy.org/trac/ticket/2001)

+   **[sql]**

    Integer 的 TypeDecorator 可以与主键列一起使用，并且各种方言的“autoincrement”特性以及“sqlite_autoincrement”标志将尊重底层数据库类型为基于 Integer 的情况。

    参考：[#2005](https://www.sqlalchemy.org/trac/ticket/2005)

+   **[sql]**

    当 Integer 主键列上存在 server_default 时确保了一致性。SQLA 不会预取这些值，它们也不会在 cursor.lastrowid（DBAPI）中返回。确保所有后端在这种情况下一致地在 result.inserted_primary_key 中返回 None。关于这种情况的反射，具有 server_default 的 int 主键列的反射会将“autoincrement”标志设置为 False，除了在检测到序列默认值的 PG SERIAL 列的情况下。

    参考：[#2020](https://www.sqlalchemy.org/trac/ticket/2020), [#2021](https://www.sqlalchemy.org/trac/ticket/2021)

+   **[sql]**

    在确定 result.inserted_primary_key 的内容时，结果行处理器会应用于预执行的 SQL 默认值，以及 cursor.lastrowid。

    参考：[#2006](https://www.sqlalchemy.org/trac/ticket/2006)

+   **[sql]**

    现在，在 select 语句的“columns clause”中存在的绑定参数会像其他“匿名”子句一样自动标记，这样在获取行时它们的“类型”就会有意义，就像结果行处理器一样。

+   **[sql]**

    TypeDecorator 存在于“sqlalchemy”导入空间中。

+   **[sql]**

    在 execute()调用范围内发生的非 DBAPI 错误现在被包装在 sqlalchemy.exc.StatementError 中，并包含 SQL 语句的文本和 params 的 repr()。这样可以更容易地识别在 DBAPI 介入之前失败的语句执行。

    参考：[#2015](https://www.sqlalchemy.org/trac/ticket/2015)

+   **[sql]**

    将直接将“.bind”与 ClauseElement 关联的概念明确地移动到 Executable，即描述表示引擎可执行构造的混合体。这个改变是对内部组织的改进，不太可能影响任何真实世界的使用。

    参考：[#2048](https://www.sqlalchemy.org/trac/ticket/2048)

+   **[sql]**

    Column.copy()，如在 table.tometadata() 中使用，将复制 'doc' 属性。（也适用于 0.6.7 版本）

    参考：[#2028](https://www.sqlalchemy.org/trac/ticket/2028)

+   **[sql]**

    在 resultproxy.c 扩展中添加了一些 defs，以便在 Python 2.4 上编译和运行扩展。（也适用于 0.6.7 版本）

    参考：[#2023](https://www.sqlalchemy.org/trac/ticket/2023)

+   **[sql]**

    编译器扩展现在支持覆盖默认的表达式编译。_BindParamClause，包括在 insert() / update() 语句的 VALUES/SET 子句中自动生成的绑定也将使用新的编译规则。（也适用于 0.6.7 版本）

    参考：[#2042](https://www.sqlalchemy.org/trac/ticket/2042)

+   **[sql]**

    SQLite 方言现在对于基于文件的数据库使用 NullPool。

    参考：[#1921](https://www.sqlalchemy.org/trac/ticket/1921)

+   **[sql]**

    现在，作为 sqlite 数据库位置的路径通过 os.path.abspath() 进行标准化，以便进程内的目录更改不会影响相对文件路径的最终位置。

    参考：[#2036](https://www.sqlalchemy.org/trac/ticket/2036)

### postgresql

+   **[postgresql]**

    当显式序列执行推断 SERIAL 列的自动生成序列的名称时，这仅在 implicit_returning=False 时发生，现在使用与 PostgreSQL 相同的逻辑，适应如果表 + 列名大于 63 个字符。（也适用于 0.6.7 版本）

    参考：[#1083](https://www.sqlalchemy.org/trac/ticket/1083)

+   **[postgresql]**

    添加了一个额外的 libpq 消息到“断开”异常列表中，“could not receive data from server”（也适用于 0.6.7 版本）

    参考：[#2044](https://www.sqlalchemy.org/trac/ticket/2044)

### mysql

+   **[mysql]**

    新的 DBAPI 支持 pymysql，这是 MySQL-python 的纯 Python 移植。

    参考：[#1991](https://www.sqlalchemy.org/trac/ticket/1991)

+   **[mysql]**

    oursql 方言在 create_engine() 中接受与 MySQLdb 相同的 “ssl” 参数。（也适用于 0.6.7 版本）

    参考：[#2047](https://www.sqlalchemy.org/trac/ticket/2047)

### mssql

+   **[mssql]**

    当没有指定长度时，String/Unicode 类型及其对应的 VARCHAR/NVARCHAR 类型将以 “max” 作为长度，因此默认长度，通常根据 SQL Server 文档为 '1'，现在为 'unbounded'。对于 VARBINARY 类型也是如此。

    当没有指定长度时，此行为使得这些类型更加兼容 PostgreSQL 的 VARCHAR 类型，后者在没有指定长度时同样是无界的。

    参考：[#1833](https://www.sqlalchemy.org/trac/ticket/1833)

### 杂项

+   **[no_tags]**

    每个变化的详细描述如下：[`docs.sqlalchemy.org/en/latest/changelog/migration_07.html`](https://docs.sqlalchemy.org/en/latest/changelog/migration_07.html)

+   **[declarative]**

    添加了一个显式检查的情况，即在声明类的列属性上使用名称 ‘metadata’。（也适用于 0.6.7 版本）

    参考：[#2050](https://www.sqlalchemy.org/trac/ticket/2050)

+   **[firebird]**

    进行了一些调整，以支持 Interbase。FB/Interbase 版本标识被解析为类似(8, 1, 1, ‘interbase’)或(2, 1, 588, ‘firebird’)的结构，以便区分它们。

    参考：[#1885](https://www.sqlalchemy.org/trac/ticket/1885)

## 0.7.11

无发布日期

### orm

+   **[orm] [bug]**

    修复了列表插入操作`insert(0, item)`时，列表仪器化无法正确表示`[0:0]`的 bug，特别是在使用关联代理时可能发生。由于 Python 集合中的一些怪癖，这个问题在 Python 3 中比在 Python 2 中更有可能发生。

    参考：[#2807](https://www.sqlalchemy.org/trac/ticket/2807)

+   **[orm] [bug]**

    修复了一个 bug，当查询形式为：`query(SubClass).options(subqueryload(Baseclass.attrname))`，其中`SubClass`是`BaseClass`的联接继承时，会导致在属性加载时子查询内部的`JOIN`未应用到，产生笛卡尔积。填充的结果仍然倾向于是正确的，因为额外的行只是被忽略，所以这个问题可能会在其他方面正常工作的应用程序中表现为性能下降。

    参考：[#2699](https://www.sqlalchemy.org/trac/ticket/2699)

+   **[orm] [bug]**

    修复了在工作单元中的一个 bug，即当两个表之间没有设置 ForeignKey 约束时，联接继承子类可能会在父表之前插入“子”表的行。

    参考：[#2689](https://www.sqlalchemy.org/trac/ticket/2689)

+   **[orm] [bug]**

    改进了在检测到“反向引用循环”时发出的错误消息，即当属性事件触发两个其他属性之间的双向赋值时。这种情况不仅会在分配错误类型的对象时发生，还会在属性被错误配置为反向引用到现有反向引用对时发生。

    参考：[#2674](https://www.sqlalchemy.org/trac/ticket/2674)

+   **[orm] [bug]**

    当将 MapperProperty 分配给替换现有属性的映射器时，如果涉及的属性不是简单的基于列的属性，则会发出警告。替换关系属性很少（或从未？）是预期的，通常指的是映射器配置错误。如果在继承关系中的现有属性上配置了 backref 以覆盖现有属性，也会发出警告（这在 0.8 中是一个错误）。

    参考：[#2674](https://www.sqlalchemy.org/trac/ticket/2674)

### engine

+   **[engine] [bug]**

    `make_url()`函数使用的正则表达式现在可以解析 ipv6 地址，例如被方括号括起来的地址。

    参考：[#2851](https://www.sqlalchemy.org/trac/ticket/2851)

### sql

+   **[sql] [bug]**

    修复了自 0.7.9 以来的回归，即如果在多个 FROM 子句中引用了 CTE 的名称，则可能无法正确引用 CTE 的名称。

    参考：[#2801](https://www.sqlalchemy.org/trac/ticket/2801)

+   **[sql] [bug] [cte]**

    修复了公共表达式系统中的 bug，如果 CTE 仅用作`alias()`构造，则不会使用 WITH 关键字进行呈现。

    参考：[#2783](https://www.sqlalchemy.org/trac/ticket/2783)

+   **[sql] [bug]**

    修复了`CheckConstraint` DDL 中的 bug，其中`Column`对象的“quote”标志不会传播的问题。

    参考：[#2784](https://www.sqlalchemy.org/trac/ticket/2784)

### postgresql

+   **[postgresql] [feature]**

    添加了对 PostgreSQL 传统的 SUBSTRING 函数语法的支持，当使用常规`func.substring()`时，呈现为“SUBSTRING(x FROM y FOR z)”形式。感谢 Gunnlaugur Þór Briem。

    参考：[#2676](https://www.sqlalchemy.org/trac/ticket/2676)

### mysql

+   **[mysql] [bug]**

    更新了 MySQL 版本 5.5、5.6 的保留字，感谢 Hanno Schlichting。

    参考：[#2791](https://www.sqlalchemy.org/trac/ticket/2791)

### 测试

+   **[tests] [bug]**

    修复了在一些 Linux 平台上无法正常工作的 test_execute 中“logging”的导入问题。

    参考：[#2669](https://www.sqlalchemy.org/trac/ticket/2669)，[pull request 41](https://github.com/sqlalchemy/sqlalchemy/pull/41)

### orm

+   **[orm] [bug]**

    修复了列表插入操作`insert(0, item)`与关联代理一起使用时，列表插入操作`[0:0]`的错误表示问题，特别是在使用 Python 3 时更容易出现此问题，而不是 Python 2。

    参考：[#2807](https://www.sqlalchemy.org/trac/ticket/2807)

+   **[orm] [bug]**

    修复了查询形式为：`query(SubClass).options(subqueryload(Baseclass.attrname))`的 bug，其中`SubClass`是`BaseClass`的联合继承，将无法在属性加载中应用子查询中的`JOIN`，导致产生笛卡尔积。填充的结果仍然往往是正确的，因为额外的行只是被忽略，所以这个问题可能存在于其他方面正常工作的应用程序中作为性能下降。

    参考：[#2699](https://www.sqlalchemy.org/trac/ticket/2699)

+   **[orm] [bug]**

    修复了在联合继承子类在两个表之间没有设置外键约束的情况下，可能会在“sub”表之前插入父表的行的工作单元中的错误。

    参考：[#2689](https://www.sqlalchemy.org/trac/ticket/2689)

+   **[orm] [bug]**

    当检测到“backref 循环”时，即当属性事件触发两个其他属性之间的双向赋值时，改进了发出的错误消息。这种情况不仅发生在分配错误类型的对象时，还发生在属性配置错误地反向引用到现有的反向引用对时。

    参考：[#2674](https://www.sqlalchemy.org/trac/ticket/2674)

+   **[orm] [bug]**

    当将 MapperProperty 分配给替换现有属性的映射器时，如果相关属性不是基于简单列的属性，则会发出警告。很少（甚至从未？）预期替换关系属性，并且通常指的是映射器配置错误。如果在继承关系中的现有属性之上配置了 backref，则此警告还将警告（这在 0.8 中是错误的）。

    参考：[#2674](https://www.sqlalchemy.org/trac/ticket/2674)

### engine

+   **[engine] [bug]**

    `make_url()`函数使用的正则表达式现在解析 ipv6 地址，例如，用方括号括起来。

    参考：[#2851](https://www.sqlalchemy.org/trac/ticket/2851)

### sql

+   **[sql] [bug]**

    修复了自 0.7.9 以来的退化，即如果在多个 FROM 子句中引用了 CTE 的名称，则可能无法正确引用该名称。

    参考：[#2801](https://www.sqlalchemy.org/trac/ticket/2801)

+   **[sql] [bug] [cte]**

    修复了公共表达式系统中的错误，在该系统中，如果 CTE 仅用作`alias()`构造，则不会使用 WITH 关键字呈现。

    参考：[#2783](https://www.sqlalchemy.org/trac/ticket/2783)

+   **[sql] [bug]**

    修复了`CheckConstraint` DDL 中的错误，其中来自`Column`对象的“quote”标志不会被传播。

    参考：[#2784](https://www.sqlalchemy.org/trac/ticket/2784)

### postgresql

+   **[postgresql] [feature]**

    添加了对 PostgreSQL 传统 SUBSTRING 函数语法的支持，当使用常规的`func.substring()`时，渲染为“SUBSTRING(x FROM y FOR z)” 。感谢 Gunnlaugur Þór Briem。

    参考：[#2676](https://www.sqlalchemy.org/trac/ticket/2676)

### mysql

+   **[mysql] [bug]**

    MySQL 版本 5.5、5.6 的保留字更新，感谢 Hanno Schlichting。

    参考：[#2791](https://www.sqlalchemy.org/trac/ticket/2791)

### tests

+   **[tests] [bug]**

    修复了在 test_execute 中导入“logging”时在某些 Linux 平台上无法工作的问题。

    参考：[#2669](https://www.sqlalchemy.org/trac/ticket/2669)，[pull request 41](https://github.com/sqlalchemy/sqlalchemy/pull/41)

## 0.7.10

发布日期：2013 年 2 月 7 日星期四

### orm

+   **[orm] [bug]**

    修复了潜在的内存泄漏问题，该问题可能在创建任意数量的`sessionmaker`对象时发生。当 sessionmaker 创建的匿名子类在解除引用时，由于事件包中仍然存在类级别的引用，该子类不会被垃圾收集。此问题也适用于与事件调度程序结合使用临时子类的任何自定义系统。

    参考：[#2650](https://www.sqlalchemy.org/trac/ticket/2650)

+   **[orm] [bug]**

    `Query.merge_result()` 现在可以在外连接中加载可能为 `None` 的实体的行，而不会引发错误。

    参考：[#2640](https://www.sqlalchemy.org/trac/ticket/2640)

+   **[orm] [bug]**

    `MutableComposite` 类型以前不允许使用 `MutableBase.coerce()` 方法，尽管代码似乎表明了这一意图，所以现在这个方法可以使用了，并添加了一个简短的示例。作为副作用，此事件处理程序的机制已更改，以便新的 `MutableComposite` 类型不再添加每个类型的全局事件处理程序。同时也是在 0.8.0b2 中。

    参考：[#2624](https://www.sqlalchemy.org/trac/ticket/2624)

+   **[orm] [bug]**

    修复了 Session 计算错误的 bug，当用另一个具有相同主键的对象替换身份映射中的已删除对象时，如果替换的主键是通过非工作单元建立的 INSERT 语句或通过另一个实例的主键切换建立的，则在回滚（）时会引发“冲突状态”错误。

    参考：[#2583](https://www.sqlalchemy.org/trac/ticket/2583)

### engine

+   **[engine] [bug]**

    修复了 `MetaData.reflect()` 以正确使用给定的 `Connection`，如果给定，而不是从该连接的 `Engine` 中打开第二个连接。

    参考：[#2604](https://www.sqlalchemy.org/trac/ticket/2604)

### sql

+   **[sql] [bug]**

    回溯调整了 `TypeDecorator` 的 `__repr__` 到 0.7 版本，允许 `PickleType` 产生一个干净的 `repr()`，以帮助 Alembic。

    参考：[#2584](https://www.sqlalchemy.org/trac/ticket/2584), [#2594](https://www.sqlalchemy.org/trac/ticket/2594)

+   **[sql] [bug]**

    修复了当 `Table.tometadata()` 既有外键又有列的备用 “.key” 名称时会失败的 bug。

    参考：[#2643](https://www.sqlalchemy.org/trac/ticket/2643)

+   **[sql] [bug]**

    修复了在不传递“for_update=True”标志的情况下使用 server_onupdate=<FetchedValue|DefaultClause> 会将默认对象应用于 server_default，覆盖原有内容的 bug。在这种用法中不应该需要显式的 for_update=True 参数（特别是因为文档显示的示例没有使用它），因此现在在内部使用给定默认对象的副本来安排，如果标志未设置为对应于该参数的值。

    参考：[#2631](https://www.sqlalchemy.org/trac/ticket/2631)

+   **[sql] [gae] [mysql]**

    向 `gaerdbms` 方言添加了条件导入，尝试导入 rdbms_apiproxy vs. rdbms_googleapi 以在开发和生产平台上同时工作。现在也支持 `instance` 属性。感谢 Sean Lynch。还将增强功能回溯到允许用户名/密码以及修复从 0.8 中解释错误代码的问题。

    参考：[#2649](https://www.sqlalchemy.org/trac/ticket/2649)

### mysql

+   **[mysql] [feature]**

    向 OurSQL 方言添加了“raise_on_warnings”标志。

    参考：[#2523](https://www.sqlalchemy.org/trac/ticket/2523)

+   **[mysql] [feature]**

    向 MySQLdb 方言添加了“read_timeout”标志。

    参考：[#2554](https://www.sqlalchemy.org/trac/ticket/2554)

### sqlite

+   **[sqlite] [bug]**

    对于在 0.7.9 中发布的与 SQLite 相关的问题进行了更多调整，以拦截旧版 SQLite 引用外键时的引号字符。除了拦截双引号外，现在还拦截其他引号字符，如括号、反引号和单引号。

    参考：[#2568](https://www.sqlalchemy.org/trac/ticket/2568)

### mssql

+   **[mssql] [bug]**

    修复了在与 MSSQL 方言的“模式渲染”逻辑未考虑 .key 的情况下，在 Column 中使用“key”与拥有表的“模式”一起会导致无法定位结果行的 bug。

+   **[mssql] [bug]**

    在 mssql 信息模式中添加了 Py3K 条件，修复了不必要的 .decode() 调用，修复了 Py3k 中的反射问题。

    参考：[#2638](https://www.sqlalchemy.org/trac/ticket/2638)

### oracle

+   **[oracle] [bug]**

    Oracle LONG 类型，虽然是一个无界文本类型，但在返回结果行时似乎没有使用 cx_Oracle.LOB 类型，因此方言已被修复，排除了对 LONG 的 cx_Oracle.LOB 过滤应用。

    参考：[#2620](https://www.sqlalchemy.org/trac/ticket/2620)

+   **[oracle] [bug]**

    修复了在与 cx_Oracle 结合使用 `.prepare()` 时的用法，以便返回值为 `False` 时不会调用 `connection.commit()`，从而避免“无事务”错误。已经证明 SQLAlchemy 和 cx_oracle 可以以一种基本的方式工作，但受到驱动程序观察到的警告的影响；请查看文档以获取详细信息。

    参考：[#2611](https://www.sqlalchemy.org/trac/ticket/2611)

+   **[oracle] [bug]**

    更改了从 setinputsizes()步骤中排除的 cx_oracle 类型列表，仅包括 STRING 和 UNICODE；CLOB 和 NCLOB 已被移除。这是为了解决 cx_oracle 在 executemany()调用中存在问题的行为。在 0.8 中，相同的更改也适用，但也可以通过 exclude_setinputsizes 参数进行配置。

    参考：[#2561](https://www.sqlalchemy.org/trac/ticket/2561)

### orm

+   **[orm] [bug]**

    修复了潜在的内存泄漏问题，如果创建了任意数量的`sessionmaker`对象，可能会发生。当通过 sessionmaker 创建的匿名子类被解引用时，由于事件包中仍然存在类级别的引用，该子类将无法被垃圾回收。这个问题也适用于任何与事件调度程序一起使用临时子类的自定义系统。

    参考：[#2650](https://www.sqlalchemy.org/trac/ticket/2650)

+   **[orm] [bug]**

    `Query.merge_result()`现在可以从外连接中加载行，其中一个实体可能为`None`而不会抛出错误。

    参考：[#2640](https://www.sqlalchemy.org/trac/ticket/2640)

+   **[orm] [bug]**

    `MutableComposite`类型不允许使用`MutableBase.coerce()`方法，尽管代码似乎表明了这一意图，所以现在可以使用，并添加了一个简短的示例。作为副作用，此事件处理程序的机制已更改，以便新的`MutableComposite`类型不再添加每种类型的全局事件处理程序。也适用于 0.8.0b2。

    参考：[#2624](https://www.sqlalchemy.org/trac/ticket/2624)

+   **[orm] [bug]**

    修复了 Session 计数错误的 bug，即在身份映射中用另一个具有相同主键的对象替换已删除的对象，如果替换的主键是通过非工作单元建立的 INSERT 语句或通过另一个实例的主键切换建立的，则在 rollback()时会引发“冲突状态”错误。

    参考：[#2583](https://www.sqlalchemy.org/trac/ticket/2583)

### engine

+   **[engine] [bug]**

    修复了`MetaData.reflect()`以正确使用给定的`Connection`，如果给定的话，而不是从该连接的`Engine`中打开第二个连接。

    参考：[#2604](https://www.sqlalchemy.org/trac/ticket/2604)

### sql

+   **[sql] [bug]**

    将对`TypeDecorator`的`__repr__`的调整回溯到 0.7 版本，允许`PickleType`生成一个干净的`repr()`以帮助 Alembic。

    参考：[#2584](https://www.sqlalchemy.org/trac/ticket/2584), [#2594](https://www.sqlalchemy.org/trac/ticket/2594)

+   **[sql] [bug]**

    修复了一个 bug，即如果一个 Column 既有外键又有列的替代“.key”名称，那么`Table.tometadata()`将失败。

    参考：[#2643](https://www.sqlalchemy.org/trac/ticket/2643)

+   **[sql] [bug]**

    修复了一个 bug，即在没有传递“for_update=True”标志的情况下使用 server_onupdate=<FetchedValue|DefaultClause>会将默认对象应用于 server_default，覆盖原有内容。这种用法不应该需要显式的 for_update=True 参数（尤其是文档中显示的示例没有使用它），因此现在在内部使用给定默认对象的副本，如果标志未设置为对应该参数的值。

    参考：[#2631](https://www.sqlalchemy.org/trac/ticket/2631)

+   **[sql] [gae] [mysql]**

    在`gaerdbms`方言中添加了一个条件导入，尝试导入 rdbms_apiproxy 和 rdbms_googleapi 以在开发和生产平台上工作。现在也支持`instance`属性。感谢 Sean Lynch。还将允许用户名/密码以及修复从 0.8 版本开始的错误代码解释的增强功能回溯。

    参考：[#2649](https://www.sqlalchemy.org/trac/ticket/2649)

### mysql

+   **[mysql] [feature]**

    在 OurSQL 方言中添加了“raise_on_warnings”标志。

    参考：[#2523](https://www.sqlalchemy.org/trac/ticket/2523)

+   **[mysql] [feature]**

    在 MySQLdb 方言中添加了“read_timeout”标志。

    参考：[#2554](https://www.sqlalchemy.org/trac/ticket/2554)

### sqlite

+   **[sqlite] [bug]**

    对与 0.7.9 版本中发布的这个与 SQLite 相关的问题进行了进一步调整，以拦截反射外键时的传统 SQLite 引号字符。除了拦截双引号外，现在还拦截其他引号字符，如方括号、反引号和单引号。

    参考：[#2568](https://www.sqlalchemy.org/trac/ticket/2568)

### mssql

+   **[mssql] [bug]**

    修复了一个 bug，即在 Column 中与拥有 Table 的“schema”一起使用“key”会由于 MSSQL 方言的“schema 渲染”逻辑未考虑.key 而无法定位结果行。

+   **[mssql] [bug]**

    在 mssql 信息模式中添加了一个 Py3K 条件，解决了 Py3k 中反射的问题。

    参考：[#2638](https://www.sqlalchemy.org/trac/ticket/2638)

### oracle

+   **[oracle] [bug]**

    Oracle 的 LONG 类型，虽然是一个无界文本类型，但在返回结果行时似乎不使用 cx_Oracle.LOB 类型，因此方言已被修复以排除 LONG 应用 cx_Oracle.LOB 过滤。 

    参考：[#2620](https://www.sqlalchemy.org/trac/ticket/2620)

+   **[oracle] [bug]**

    修复了在与 cx_Oracle 一起使用`.prepare()`时的 bug，以便返回值为`False`将导致不调用`connection.commit()`，从而避免“无事务”错误。已经证明在 SQLAlchemy 和 cx_oracle 中以一种基本方式工作两阶段事务，但受到与驱动程序观察到的警告的限制；查看文档以获取详细信息。

    参考：[#2611](https://www.sqlalchemy.org/trac/ticket/2611)

+   **[oracle] [bug]**

    更改了从`setinputsizes()`步骤中排除的 cx_oracle 类型列表，现在只包括 STRING 和 UNICODE；CLOB 和 NCLOB 已被移除。这是为了解决 cx_oracle 在 executemany()调用中存在问题的情况。在 0.8 版本中，同样的更改也被应用，但也可以通过 exclude_setinputsizes 参数进行配置。

    参考：[#2561](https://www.sqlalchemy.org/trac/ticket/2561)

## 0.7.9

发布日期：2012 年 10 月 01 日

### orm

+   **[orm] [bug]**

    修复了主要局限于新的 AbstractConcreteBase 辅助程序的 bug，其中从超类继承的“type”属性不会在子类上被覆盖以生成“reserved for base”错误消息，而是在那里放置一个无效的属性。这与使用 ConcreteBase 以及所有经典具体映射的行为不一致，其中多态基类的“type”列在子类上会被显式禁用，除非显式覆盖。

+   **[orm] [bug]**

    当 lazy='dynamic'与 uselist=False 组合时会发出警告。这在 0.8 中会引发异常。

+   **[orm] [bug]**

    修复了用户在相关对象赋值中的错误可能导致递归溢出的 bug，如果赋值触发了与同一目标上的双向属性同名的 backref，则会引发信息性错误。

+   **[orm] [bug]**

    修复了当 ORM 绑定“version”列时传递错误类型信息的 bug，当使用“version”功能时。测试由 Daniel Miller 提供。

    参考：[#2539](https://www.sqlalchemy.org/trac/ticket/2539)

+   **[orm] [bug]**

    在 Session.commit()中发生的“flush”中添加了额外的逻辑，以便在随后的 flush 中也刷新由 after_flush()或 after_flush_postexec()钩子添加的额外状态，然后“commit”完成。后续对 flush()的调用将继续，直到 after_flush 钩子停止添加新状态。在事件发生时，还有一个“overflow”计数器为 100，以防一个破损的 after_flush()钩子每次都添加新内容。

    参考：[#2566](https://www.sqlalchemy.org/trac/ticket/2566)

### engine

+   **[engine] [feature]**

    事件系统的内存使用显著改善；在为特定类型的事件建立实例级监听器之前，不再为该事件创建实例级集合。

    参考：[#2516](https://www.sqlalchemy.org/trac/ticket/2516)

+   **[engine] [bug]**

    修复了一个 bug，即当 QueuePool 中有线程等待连接时，发生断开检测 + 丢弃会使这些线程在旧池的超时期间保持等待状态（如果禁用了超时，则会无限期等待）。修复现在使用一个特殊的异常情况通知这些等待线程，并使它们转移到新池中。

    参考：[#2522](https://www.sqlalchemy.org/trac/ticket/2522)

+   **[engine] [bug]**

    在 mysql/__init__.py 中添加了 gaerdbms 导入，缺少此导入会导致无法加载新的 GAE 方言。

    参考：[#2529](https://www.sqlalchemy.org/trac/ticket/2529)

+   **[engine] [bug]**

    修复了 cextension bug，即如果给定的索引是一个 Column 对象而不是字符串，则“模糊列错误”将无法正常工作。请注意，这里仍然存在一些列定位问题，在 0.8 中已经修复。

    参考：[#2553](https://www.sqlalchemy.org/trac/ticket/2553)

+   **[engine] [bug]**

    修复了 Enum 的 repr()，以包括 “name” 和 “native_enum” 标志。有助于 Alembic 自动生成。

### sql

+   **[sql] [bug]**

    修复了 DropIndex 构造，以支持与远程模式中的表相关联的索引。

    参考：[#2571](https://www.sqlalchemy.org/trac/ticket/2571)

+   **[sql] [bug]**

    修复了 over() 构造中的 bug，即当将空列表传递给 partition_by 或 order_by 中的任意一个时，而不是 None，会导致生成失败。由 Gunnlaugur Þór Briem 提供。

    参考：[#2574](https://www.sqlalchemy.org/trac/ticket/2574)

+   **[sql] [bug]**

    修复了 CTE bug，即 CTE 中存在的位置绑定参数会破坏绑定参数的整体顺序。这主要影响支持位置绑定 + CTE 的 SQL Server 平台。

    参考：[#2521](https://www.sqlalchemy.org/trac/ticket/2521)

+   **[sql] [bug]**

    在 CTE 中修复了更多的不直观问题，这些问题阻止了在没有别名的情况下引用 CTE 中的自身联合。现在，CTE 根据名称唯一地呈现，仅呈现给定名称的最外层 CTE - 所有其他引用都只是名称。这甚至包括引用不同版本的同一 CTE 对象的其他 CTE/SELECT，比如对该 SELECT 或该 SELECT 的 UNION ALL 的其他 CTE/SELECT 的引用。在这种情况下，我们在对象标识和词法标识之间有些放松了通常的链接。两个不相关的 CTE 之间的真正名称冲突现在会引发错误。

+   **[sql] [bug]**

    在 common table expression 的 WITH RECURSIVE 子句中，对列名应用 quoting 规则，根据原始 Column 的 quoting 规则。

    参考：[#2512](https://www.sqlalchemy.org/trac/ticket/2512)

+   **[sql] [bug]**

    修复了 0.7.6 中引入的回归，即在某些“clone+replace”场景中，SELECT 语句的 FROM 列表可能不正确。

    参考：[#2518](https://www.sqlalchemy.org/trac/ticket/2518)

+   **[sql] [bug]**

    修复了一个 bug，即在嵌套子查询中使用 UNION 或类似操作会干扰结果列的定位，如果结果列与嵌套 UNION 中的某个名称相同，则会出现问题。

    参考：[#2552](https://www.sqlalchemy.org/trac/ticket/2552)

+   **[sql] [bug]**

    修复了自 0.6 以来的一个回归，涉及结果行定位。应该可以在其中使用基于字符串的列的 select()语句，即 select(['id', 'name']).select_from('mytable')，并且可以通过具有这些名称的 Column 对象定位此语句；这是 query(MyClass).from_statement(some_statement)工作的机制。在某个时刻，使用 select(['id'])的特定情况停止工作，这等同于 select([literal_column('id')])，因此已重新实施并当然进行了测试。

    参考：[#2558](https://www.sqlalchemy.org/trac/ticket/2558)

+   **[sql] [bug]**

    在 ColumnOperators 基类中添加了缺失的 is_()和 isnot()操作符，使得这些长期可用的操作符以方法的形式存在，就像其他操作符一样。

    参考：[#2544](https://www.sqlalchemy.org/trac/ticket/2544)

### postgresql

+   **[postgresql] [bug]**

    反射主键约束中的列现在按照约束本身定义它们的顺序返回，而不是表的顺序。感谢 Gunnlaugur Þór Briem。

    参考：[#2531](https://www.sqlalchemy.org/trac/ticket/2531)

+   **[postgresql] [bug]**

    将“terminating connection”��加到我们用于检测与 PG 断开连接的消息列表中，这在某些版本中似乎在服务器重新启动时出现。

    参考：[#2570](https://www.sqlalchemy.org/trac/ticket/2570)

### mysql

+   **[mysql] [bug]**

    更新了 mysqlconnector 接口，使用了更新的“client flag”和“charset”API，感谢 David McNelis。

### sqlite

+   **[sqlite] [feature]**

    添加了对 SQLite 中 localtimestamp() SQL 函数的支持，感谢 Richard Mitchell。

+   **[sqlite] [bug]**

    调整了一个非常古老的 bug 修复，尝试解决一个 SQLite 问题，该问题在 sqlite 3.6.14 中已经“修复”，涉及在使用“foreign_key_list” pragma 时表名周围的引号。修复已调整为不干扰实际在列或表名中的引号，尽可能地减少干扰；如果目标表的名称实际上在其名称中有引号（即“”“mytable”””），sqlite 仍然不会正确返回 foreign_key_list()的结果。

    参考：[#2568](https://www.sqlalchemy.org/trac/ticket/2568)

+   **[sqlite] [bug]**

    调整了列默认反射代码，将非字符串值转换为字符串，以适应旧的 SQLite 版本，这些版本不将默认信息作为字符串传递。

    参考：[#2265](https://www.sqlalchemy.org/trac/ticket/2265)

### mssql

+   **[mssql] [bug]**

    修复了编译器错误，即在 ORDER BY 中使用相关子查询，如果语句还使用了 LIMIT/OFFSET，由于 ROW_NUMBER() OVER 子句中的错误渲染，将无法正确呈现。修复由 sayap 提供。

    参考：[#2538](https://www.sqlalchemy.org/trac/ticket/2538)

+   **[mssql] [bug]**

    修复了编译器错误，即如果给定的 select()具有“offset”属性，则在第二次编译时，构造将无法正确编译。

    参考：[#2545](https://www.sqlalchemy.org/trac/ticket/2545)

+   **[mssql] [bug]**

    修复了一个错误，即如果同一约束/表存在于多个模式中，则主键约束的反射会使列重复。

### orm

+   **[orm] [bug]**

    修复了主要局限于新的 AbstractConcreteBase 助手的错误，其中从超类继承的“type”属性不会在子类上被覆盖，以产生“保留给基类”的错误消息，而是在那里放置一个无效属性。这与使用 ConcreteBase 以及所有经典具体映射的行为不一致，其中多态基类的“type”列在子类上会被显式禁用，除非显式覆盖。

+   **[orm] [bug]**

    当 lazy='dynamic'与 uselist=False 结合时，会发出警告。这在 0.8 中是一个异常。

+   **[orm] [bug]**

    修复了一个错误，即在相关对象赋值中的用户错误可能会导致递归溢出，如果赋值触发了与不正确类上的同名双向属性相同名称的 backref，则会引发一个信息性错误。

+   **[orm] [bug]**

    修复了一个错误，即当 ORM 绑定“version”列时，如果使用“version”功能，则会传递不正确的类型信息。测试由 Daniel Miller 提供。

    参考：[#2539](https://www.sqlalchemy.org/trac/ticket/2539)

+   **[orm] [bug]**

    在 Session.commit()中添加了额外的逻辑，使得在后续的 flush 中也会刷新由 after_flush()或 after_flush_postexec()钩子添加的额外状态，直到“commit”完成。后续对 flush()的调用将继续，直到 after_flush 钩子停止添加新状态。在“overflow”计数器达到 100 时，如果由于破损的 after_flush()钩子每次都添加新内容，则会触发。

    参考：[#2566](https://www.sqlalchemy.org/trac/ticket/2566)

### engine

+   **[engine] [feature]**

    事件系统的内存使用显著改善；直到为该事件建立了实例级别的监听器，才会为特定类型的事件创建实例级别的集合。

    参考：[#2516](https://www.sqlalchemy.org/trac/ticket/2516)

+   **[engine] [bug]**

    修复了一个 bug，即当 QueuePool 中有线程等待连接时，断开检测 + 释放会使这些线程等待旧池的超时时间（或者如果禁用超时，则无限期等待）。现在修复通知这些等待者有一个特殊的异常情况，并让它们转移到新池。

    参考：[#2522](https://www.sqlalchemy.org/trac/ticket/2522)

+   **[engine] [bug]**

    在 mysql/__init__.py 中添加了 gaerdbms 导入，缺少此导入会导致无法加载新的 GAE 方言。

    参考：[#2529](https://www.sqlalchemy.org/trac/ticket/2529)

+   **[engine] [bug]**

    修复了 cextension 的 bug，当给定的索引是一个 Column 对象而不是一个字符串时，“模糊列错误”无法正常工作。请注意，这里仍然存在一些列定位问题，在 0.8 版本中已修复。

    参考：[#2553](https://www.sqlalchemy.org/trac/ticket/2553)

+   **[engine] [bug]**

    修复了 Enum 的 repr() 方法，包括“name”和“native_enum”标志。有助于 Alembic 的自动生成。

### sql

+   **[sql] [bug]**

    修复了 DropIndex 构造以支持与远程模式中的表关联的索引。

    参考：[#2571](https://www.sqlalchemy.org/trac/ticket/2571)

+   **[sql] [bug]**

    修复了 over() 构造中的 bug，即将空列表传递给 partition_by 或 order_by，而不是 None，将无法正确生成。感谢 Gunnlaugur Þór Briem。

    参考：[#2574](https://www.sqlalchemy.org/trac/ticket/2574)

+   **[sql] [bug]**

    修复了 CTE 的 bug，即 CTE 本身中存在的位置绑定参数会破坏绑定参数的整体排序。这主要影响支持位置绑定 + CTE 支持的 SQL Server 平台。

    参考：[#2521](https://www.sqlalchemy.org/trac/ticket/2521)

+   **[sql] [bug]**

    修复了 CTE 中的更多不直观之处，这些不直观之处阻止了在不使用别名的情况下引用 CTE 自身的联合。CTE 现在根据名称唯一呈现，仅呈现给定名称的最外层 CTE - 所有其他引用只是作为名称呈现。这甚至包括引用同一 CTE 对象的其他 CTE/SELECT，例如引用该 SELECT 的 SELECT 或 UNION ALL。在这种情况下，我们在对象标识和词法标识之间有些放松通常的链接。两个不相关的 CTE 之间的真实名称冲突现在会引发错误。

+   **[sql] [bug]**

    在通用表达式 WITH RECURSIVE 子句中，对列名应用引用规则，根据原始列的引用规则。

    参考：[#2512](https://www.sqlalchemy.org/trac/ticket/2512)

+   **[sql] [bug]**

    修复了在 0.7.6 版本中引入的回归，导致在某些“克隆+替换”场景中 SELECT 语句的 FROM 列表可能不正确。

    参考：[#2518](https://www.sqlalchemy.org/trac/ticket/2518)

+   **[sql] [bug]**

    修复了一个 bug，即在嵌套子查询中使用 UNION 或类似操作会干扰结果��定位，如果结果列与嵌套 UNION 中的名称相同，则会出现问题。

    参考：[#2552](https://www.sqlalchemy.org/trac/ticket/2552)

+   **[sql] [bug]**

    修复了自 0.6 以来的一个回归问题，涉及结果行定位。应该可以使用一个带有字符串列的 select() 语句，即 select(['id', 'name']).select_from('mytable')，并且使这个语句可以被具有这些名称的 Column 对象定位；这是 query(MyClass).from_statement(some_statement) 的机制。在某个时候，使用 select(['id']) 的特定情况停止工作，这等同于 select([literal_column('id')])，因此已经重新安装并当然测试。

    参考：[#2558](https://www.sqlalchemy.org/trac/ticket/2558)

+   **[sql] [bug]**

    添加了缺失的操作符 is_()，isnot() 到 ColumnOperators 基类，以便这些长期可用的操作符作为其他操作符一样作为方法存在。

    参考：[#2544](https://www.sqlalchemy.org/trac/ticket/2544)

### postgresql

+   **[postgresql] [bug]**

    反射主键约束中的列现在按照约束本身定义的顺序返回，而不是表如何排序它们的顺序。感谢 Gunnlaugur Þór Briem。

    参考：[#2531](https://www.sqlalchemy.org/trac/ticket/2531)

+   **[postgresql] [bug]**

    将“terminating connection”添加到我们用于检测与 PG 断开连接的消息列表中，当服务器重新启动时，某些版本中似乎存在这种情况。

    参考：[#2570](https://www.sqlalchemy.org/trac/ticket/2570)

### mysql

+   **[mysql] [bug]**

    更新了 mysqlconnector 接口，以使用更新的“client flag”和“charset” API，感谢 David McNelis。

### sqlite

+   **[sqlite] [feature]**

    增加了对 SQLite 中实现的 localtimestamp() SQL 函数的支持，感谢 Richard Mitchell。

+   **[sqlite] [bug]**

    调整了一个非常古老的错误修复，尝试解决一个 SQLite 问题，该问题本身在 sqlite 3.6.14 中已经“修复”，涉及在使用“foreign_key_list” pragma 时围绕表名的引号。修复已经调整为不干扰实际上在列或表名中的引号，尽可能地；如果目标表实际上在其名称周围有引号，作为其名称的一部分（即“”“mytable”””），sqlite 仍然不会返回 foreign_key_list() 的正确结果。

    参考：[#2568](https://www.sqlalchemy.org/trac/ticket/2568)

+   **[sqlite] [bug]**

    调整了列默认反射代码，将非字符串值转换为字符串，以适应旧的 SQLite 版本，这些版本不将默认信息作为字符串传递。

    参考：[#2265](https://www.sqlalchemy.org/trac/ticket/2265)

### mssql

+   **[mssql] [bug]**

    修复了编译器错误，其中在 ORDER BY 中使用相关子查询，如果语句还使用 LIMIT/OFFSET，由于在 ROW_NUMBER() OVER 子句中的错误渲染而无法正确呈现。修复来自 sayap。

    参考：[#2538](https://www.sqlalchemy.org/trac/ticket/2538)

+   **[mssql] [bug]**

    修复了编译器错误，即如果给定的 select()具有“offset”属性，则会修改该构造，导致第二次编译时构造不正确。

    参考：[#2545](https://www.sqlalchemy.org/trac/ticket/2545)

+   **[mssql] [bug]**

    修复了主键约束的反射错误，如果相同的约束/表存在于多个模式中，则会使列重复。

## 0.7.8

发布日期：2012 年 6 月 16 日星期六

### orm

+   **[orm] [feature]**

    flush()的“objects”参数不再被弃用，因为已经确定了一些有效的用例。

+   **[orm] [bug]**

    修复了一个错误，即从多态映射到目标的 subqueryload()将为多态结果中遇到的每个不同类别调用一个新的查询。

    参��：[#2480](https://www.sqlalchemy.org/trac/ticket/2480)

+   **[orm] [bug]**

    修复了声明中的错误，其中连接表中的列的优先级（通常用于 id）如果列包含与其属性名称不同的名称，则会失败。这将导致针对实体属性进行的 primaryjoin 条件不正确。相关于这应该是那个的一部分，这是。

    参考：[#1892](https://www.sqlalchemy.org/trac/ticket/1892)，[#2491](https://www.sqlalchemy.org/trac/ticket/2491)

+   **[orm] [bug]**

    修复了 identity_key()函数不接受标量参数作为标识的问题。

    参考：[#2508](https://www.sqlalchemy.org/trac/ticket/2508)

+   **[orm] [bug]**

    修复了 populate_existing 选项无法传播到子查询急加载器的错误。

    参考：[#2497](https://www.sqlalchemy.org/trac/ticket/2497)

### engine

+   **[engine] [bug]**

    修复了 C 版本结果代理中的内存泄漏问题，其中对于结果行不提供纯 Python 元组的 DBAPI 将无法正确减少引用计数。受影响最严重的 DBAPI 是 pyodbc。

    参考：[#2489](https://www.sqlalchemy.org/trac/ticket/2489)

+   **[engine] [bug]**

    修复了影响 Py3K 的错误，即传递给 engine/connection execute()的字符串位置参数将无法正确解释，因为 Py3K 字符串上存在 __iter__。

    参考：[#2503](https://www.sqlalchemy.org/trac/ticket/2503)

### sql

+   **[sql] [bug]**

    将 BIGINT 添加到 types.__all__，将 BIGINT、BINARY、VARBINARY 添加到 sqlalchemy 模块命名空间，以及确保不再发生此类破坏的测试。

    参考：[#2499](https://www.sqlalchemy.org/trac/ticket/2499)

+   **[sql] [bug]**

    修复了当 SELECT 语句包含 UNION 或其他复合表达式时，公共表达式的正确渲染，感谢 btbuilder。

    参考：[#2490](https://www.sqlalchemy.org/trac/ticket/2490)

+   **[sql] [bug]**

    修复了在克隆的 select() 构造上 append_column() 无法正确运行的错误。由 Gunnlaugur Þór Briem 提供。

    参考：[#2482](https://www.sqlalchemy.org/trac/ticket/2482)

### postgresql

+   **[postgresql] [bug]**

    移除了在反射枚举时不必要的表子句。由 Gunnlaugur Þór Briem 提供。

    参考：[#2510](https://www.sqlalchemy.org/trac/ticket/2510)

### mysql

+   **[mysql] [feature]**

    添加了一个用于 Google App Engine 的新方言。由 Richie Foreman 提供。

    参考：[#2484](https://www.sqlalchemy.org/trac/ticket/2484)

### oracle

+   **[oracle] [bug]**

    向 oracle.* 添加了 ROWID。

    参考：[#2483](https://www.sqlalchemy.org/trac/ticket/2483)

### orm

+   **[orm] [feature]**

    flush() 的 'objects' 参数不再被弃用，因为已经确定了一些有效的用例。

+   **[orm] [bug]**

    修复了从多态映射到目标的 subqueryload() 会为多态结果中遇到的每个不同类别重新调用查询的错误。

    参考：[#2480](https://www.sqlalchemy.org/trac/ticket/2480)

+   **[orm] [bug]**

    修复了在声明中，对于连接表的复合列（通常用于 id）中列的优先级不正确的错误，如果列的名称与其属性名称不同，将导致像针对实体属性制定的 primaryjoin 条件不正确。这与此相关，应该是其中的一部分，这是。

    参考：[#1892](https://www.sqlalchemy.org/trac/ticket/1892), [#2491](https://www.sqlalchemy.org/trac/ticket/2491)

+   **[orm] [bug]**

    修复了 identity_key() 函数不接受标量参数作为标识的问题。

    参考：[#2508](https://www.sqlalchemy.org/trac/ticket/2508)

+   **[orm] [bug]**

    修复了 populate_existing 选项无法传播到子查询急加载器的错误。

    参考：[#2497](https://www.sqlalchemy.org/trac/ticket/2497)

### engine

+   **[engine] [bug]**

    修复了 C 版本的结果代理中的内存泄漏问题，即不提供纯 Python 元组用于结果行的 DBAPI 无法正确减少引用计数的问题。受影响最严重的 DBAPI 是 pyodbc。

    参考：[#2489](https://www.sqlalchemy.org/trac/ticket/2489)

+   **[engine] [bug]**

    修复了影响 Py3K 的 bug，即传递给 engine/connection execute() 的字符串位置参数无法正确解释的问题，因为 Py3K 字符串上存在 __iter__。

    参考：[#2503](https://www.sqlalchemy.org/trac/ticket/2503)

### sql

+   **[sql] [bug]**

    向 types.__all__ 添加了 BIGINT，向 sqlalchemy 模块命名空间添加了 BIGINT、BINARY、VARBINARY，并添加了测试以确保不再发生此类破坏。

    参考：[#2499](https://www.sqlalchemy.org/trac/ticket/2499)

+   **[sql] [bug]**

    修复了当 SELECT 语句包含 UNION 或其他复合表达式时，公共表达式渲染无法正确运行的问题。由 btbuilder 提供。

    参考：[#2490](https://www.sqlalchemy.org/trac/ticket/2490)

+   **[sql] [bug]**

    修复了在克隆的 select() 构造上不会正确运行 append_column() 的错误，由 Gunnlaugur Þór Briem 提供。

    参考：[#2482](https://www.sqlalchemy.org/trac/ticket/2482)

### postgresql

+   **[postgresql] [bug]**

    移除了在反射枚举时不必要的表子句，由 Gunnlaugur Þór Briem 提供。

    参考：[#2510](https://www.sqlalchemy.org/trac/ticket/2510)

### mysql

+   **[mysql] [feature]**

    添加了 Google App Engine 的新方言。由 Richie Foreman 提供。

    参考：[#2484](https://www.sqlalchemy.org/trac/ticket/2484)

### oracle

+   **[oracle] [bug]**

    将 ROWID 添加到 oracle.*。

    参考：[#2483](https://www.sqlalchemy.org/trac/ticket/2483)

## 0.7.7

发布日期：Sat May 05 2012

### orm

+   **[orm] [feature]**

    向 Query 添加了 prefix_with() 方法，调用 select().prefix_with() 以允许在语句中放置 MySQL SELECT 指令。由 Diana Clarke 提供。

    参考：[#2443](https://www.sqlalchemy.org/trac/ticket/2443)

+   **[orm] [feature]**

    添加了新的标志 @validates include_removes。当为 True 时，还将集合移除和属性删除事件发送到验证函数，当使用此标志时，该函数接受额外的参数“is_remove”。

+   **[orm] [bug]**

    在工作单元中修复了一个问题，即将非空的自引用多对一关系设置为 None 时，如果原值尚未加载，则更改将无法持久化。

    参考：[#2477](https://www.sqlalchemy.org/trac/ticket/2477)

+   **[orm] [bug]**

    修复了 0.7.6 中引入的错误，即在针对已映射为联接或其他间接可选择项的列使用 column_mapped_collection 时，将无法正常运行。

    参考：[#2409](https://www.sqlalchemy.org/trac/ticket/2409)

+   **[orm] [bug]**

    修复了 polymorphic_on 列在类中未被映射时错误地包含在 merge() 操作中引发错误的错误。

    参考：[#2449](https://www.sqlalchemy.org/trac/ticket/2449)

+   **[orm] [bug]**

    修复了表达式注释机制中的错误，这可能导致 SELECT 语句的错误渲染，特别是在使用 column_property() 时使用别名和连接时。

    参考：[#2453](https://www.sqlalchemy.org/trac/ticket/2453)

+   **[orm] [bug]**

    修复了会阻止 OrderingList 可以 pickle 的错误。由 Jeff Dairiki 提供

    参考：[#2454](https://www.sqlalchemy.org/trac/ticket/2454)

+   **[orm] [bug]**

    修复了关系比较中的错误，即调用未实现的方法如 SomeClass.somerelationship.like() 将产生递归溢出，而不是 NotImplementedError。

### sql

+   **[sql] [feature]**

    添加了新的连接事件 dbapi_error()。在所有 DBAPI 级错误时调用，传递原始的 DBAPI 异常，然后 SQLAlchemy 修改游标的状态。

+   **[sql] [bug]**

    移除了在创建索引时没有列时的警告；虽然这可能不是用户期望的行为，但作为索引只是某个名称的索引是有效的用例。

+   **[sql] [bug]**

    如果在调用 “with engine.begin()” 时 conn.begin() 失败，则会在正常传播异常之前显式关闭新获取的 Connection。 

+   **[sql] [bug]**

    将 BINARY 和 VARBINARY 添加到 types.__all__。

    参考：[#2474](https://www.sqlalchemy.org/trac/ticket/2474)

### postgresql

+   **[postgresql] [feature]**

    为 PostgreSQL 添加了新的 for_update/with_lockmode() 选项：for_update=”read”/ with_lockmode(“read”)，for_update=”read_nowait”/ with_lockmode(“read_nowait”)。这些分别发出 “FOR SHARE” 和 “FOR SHARE NOWAIT”。感谢 Diana Clarke。

    参考：[#2445](https://www.sqlalchemy.org/trac/ticket/2445)

+   **[postgresql] [bug]**

    在反射域时删除了不必要的表子句。

    参考：[#2473](https://www.sqlalchemy.org/trac/ticket/2473)

### mysql

+   **[mysql] [bug]**

    修复了一个 bug，即在 InnoDB 中使用自增复合列的“KEY”子句中的列名会双引号一个是保留字的名称。感谢 Jeff Dairiki。

    参考：[#2460](https://www.sqlalchemy.org/trac/ticket/2460)

+   **[mysql] [bug]**

    修复了一个 bug，即当“information_schema”模式的 get_view_names() 无法检索标记为“SYSTEM VIEW”的视图时会失败。感谢 Matthew Turland。

+   **[mysql] [bug]**

    修复了一个 bug，即如果 cast() 在不支持 cast() 的 SQL 表达式上使用，因此方言不会渲染 CAST，则如果转换的表达式需要分组，则评估顺序可能会更改；现在将对这些表达式应用分组。

    参考：[#2467](https://www.sqlalchemy.org/trac/ticket/2467)

### sqlite

+   **[sqlite] [feature]**

    添加了 SQLite 执行选项 “sqlite_raw_colnames=True”，将绕过 SQLite cursor.description 返回的列名中的 “.” 的尝试删除。

    参考：[#2475](https://www.sqlalchemy.org/trac/ticket/2475)

+   **[sqlite] [bug]**

    当表的主键列被替换时，比如通过 extend_existing，由 insert() 构造函数使用的“自动增量”列会被重置。之前它会继续引用以前的主键列。

    参考：[#2525](https://www.sqlalchemy.org/trac/ticket/2525)

### mssql

+   **[mssql] [feature]**

    向 PyODBC 方言添加了临时 create_engine 标志 supports_unicode_binds，以强制该方言是否将 Python unicode 文字传递给 PyODBC。

+   **[mssql] [bug]**

    修复了使用 pyodbc 方言时 use_scope_identity create_engine() 标志的问题。之前，如果设置为 False，此标志将被忽略。当设置为 False 时，对于那些“implicit_returning”设置为 False 的表，将在每次插入后获得“SELECT @@identity”以获取最后插入的 ID。

+   **[mssql] [bug]**

    使用 SQL Server 的 UPDATE..FROM 语法要求在 FROM 子句中存在被更新的表，当该表的别名也存在于 FROM 子句中时。如果 FROM 子句首次出现，则更新后的表现在始终存在于 FROM 子句中。感谢 sayap。

    参考：[#2468](https://www.sqlalchemy.org/trac/ticket/2468)

### orm

+   **[orm] [feature]**

    为 Query 添加了 prefix_with()方法，调用 select().prefix_with()以允许在语句中放置 MySQL SELECT 指令。感谢 Diana Clarke

    参考：[#2443](https://www.sqlalchemy.org/trac/ticket/2443)

+   **[orm] [feature]**

    为@validates 添加了新标志 include_removes。当为 True 时，集合删除和属性删除事件也将发送到验证函数，当使用此标志时，验证函数将接受额外参数“is_remove”。

+   **[orm] [bug]**

    修复了工作单元中的问题，即将非 None 的自引用多对一关系设置为 None 时，如果前一个值尚未加载，则无法持久化更改。

    参考：[#2477](https://www.sqlalchemy.org/trac/ticket/2477)

+   **[orm] [bug]**

    修复了 0.7.6 版本中引入的 bug，即对于被映射为连接或其他间接可选择的列使用 column_mapped_collection 会导致功能失效。

    参考：[#2409](https://www.sqlalchemy.org/trac/ticket/2409)

+   **[orm] [bug]**

    修复了多态 _on 列未在类中映射的���况下错误地包含在 merge()操作中的 bug，导致错误。

    参考：[#2449](https://www.sqlalchemy.org/trac/ticket/2449)

+   **[orm] [bug]**

    修复了表达式注释机制中的 bug，可能导致带有别名和连接的 SELECT 语句的不正确渲染，特别是在使用 column_property()时。

    参考：[#2453](https://www.sqlalchemy.org/trac/ticket/2453)

+   **[orm] [bug]**

    修复了阻止 OrderingList 可被 pickle 的 bug。感谢 Jeff Dairiki

    参考：[#2454](https://www.sqlalchemy.org/trac/ticket/2454)

+   **[orm] [bug]**

    修复了关系比较中的 bug，即调用未实现的方法（如 SomeClass.somerelationship.like()）会导致递归溢出，而不是 NotImplementedError。

### sql

+   **[sql] [feature]**

    添加了新的 connection event dbapi_error()。对于所有 DBAPI 级别的错误，会在 SQLAlchemy 修改游标状态之前调用原始的 DBAPI 异常。

+   **[sql] [bug]**

    当创建 Index 时没有列时移除警告；虽然这可能不是用户想要的，但作为一个只有特定名称的索引的占位符是有效的用例。

+   **[sql] [bug]**

    在调用“with engine.begin()”时，如果 conn.begin()失败，则新获取的连接在正常传播异常之前会被显式关闭。

+   **[sql] [bug]**

    将 BINARY、VARBINARY 添加到 types.__all__ 中。

    参考：[#2474](https://www.sqlalchemy.org/trac/ticket/2474)

### postgresql

+   **[postgresql] [feature]**

    为 PostgreSQL 添加了新的 for_update/with_lockmode()选项：for_update=”read”/with_lockmode(“read”)，for_update=”read_nowait”/with_lockmode(“read_nowait”)。这些分别发出“FOR SHARE”和“FOR SHARE NOWAIT”。感谢 Diana Clarke

    参考：[#2445](https://www.sqlalchemy.org/trac/ticket/2445)

+   **[postgresql] [bug]**

    在反射域时移除了不必要的表子句。

    参考：[#2473](https://www.sqlalchemy.org/trac/ticket/2473)

### mysql

+   **[mysql] [bug]**

    修复了一个 bug，即对于 InnoDB 中具有自增复合列的“KEY”子句内的列名，会将名称重复引用为保留字。由 Jeff Dairiki 提供。

    参考：[#2460](https://www.sqlalchemy.org/trac/ticket/2460)

+   **[mysql] [bug]**

    修复了一个 bug，即对于标记为“SYSTEM VIEW”的视图，get_view_names() 对于“information_schema”模式将无法检索到；由 Matthew Turland 提供。

+   **[mysql] [bug]**

    修复了一个 bug，即如果对不支持的 SQL 表达式使用了 cast()，因此方言不会渲染 CAST，则如果要求对被转换的表达式进行分组，则评估顺序可能会更改；现在将对这些表达式应用分组。

    参考：[#2467](https://www.sqlalchemy.org/trac/ticket/2467)

### sqlite

+   **[sqlite] [feature]**

    添加了 SQLite 执行选项“sqlite_raw_colnames=True”，将绕过 SQLite cursor.description 返回的列名中的“.” 移除尝试。 

    参考：[#2475](https://www.sqlalchemy.org/trac/ticket/2475)

+   **[sqlite] [bug]**

    当替换 Table 的主键列时，例如通过 extend_existing，由 insert() 构造使用的“自动增量”列将被重置。以前，它将继续引用先前的主键列。

    参考：[#2525](https://www.sqlalchemy.org/trac/ticket/2525)

### mssql

+   **[mssql] [feature]**

    添加了临时 create_engine 标志 supports_unicode_binds 到 PyODBC 方言，用于强制该方言是否将 Python unicode 文字传递给 PyODBC。

+   **[mssql] [bug]**

    修复了在使用 pyodbc 方言时，使用 use_scope_identity create_engine() 标志的问题。以前，如果将此标志设置为 False，则会被忽略。当设置为 False 时，每次 INSERT 后都会获得“SELECT @@identity”，以获取最后插入的 ID，对于那些将“implicit_returning”设置为 False 的表。

+   **[mssql] [bug]**

    使用 SQL Server 的 UPDATE..FROM 语法要求在 FROM 子句中同时存在更新的表和该表的别名。现在，当 FROM 存在时，更新的表始终存在于 FROM 中。由 sayap 提供。

    参考：[#2468](https://www.sqlalchemy.org/trac/ticket/2468)

## 0.7.6

发布日期：2012 年 3 月 14 日（星期三）

### orm

+   **[orm] [feature]**

    向 Session 添加了“no_autoflush”上下文管理器，与 with 一起使用：将暂时禁用自动刷新。

+   **[orm] [feature]**

    向 Query 添加了 cte() 方法，调用来自 Core 的公共表达式支持（见下文）。

    参考：[#1859](https://www.sqlalchemy.org/trac/ticket/1859)

+   **[orm] [feature]**

    添加了通过 query(sometable).filter_by(colname=value) 查询绑定到 Table 的列名的能力。

    参考：[#2400](https://www.sqlalchemy.org/trac/ticket/2400)

+   **[orm] [bug]**

    修复了一个事件注册 bug，主要表现为在与 Session 类关联的事件在创建了 sessionmaker() 实例后未注册。

    参考：[#2424](https://www.sqlalchemy.org/trac/ticket/2424)

+   **[orm] [bug]**

    修复了一个错误，即在主键连接条件中带有“literal”会在某些深度嵌套表达式中的编译时多次渲染相同的绑定参数名称时引发错误。

    参考：[#2425](https://www.sqlalchemy.org/trac/ticket/2425)

+   **[orm] [bug]**

    移除了对执行多行删除时受影响行数的检查。如果两行之间存在 ON DELETE CASCADE，我们无法从 DBAPI 中获取准确的行数；在大多数情况下，这个特定计数也不受大多数 DBAPI 的支持，MySQLdb 是一个例外情况。

    参考：[#2403](https://www.sqlalchemy.org/trac/ticket/2403)

+   **[orm] [bug]**

    修复了一个错误，即使用 attribute_mapped_collection 或 column_mapped_collection 的对象无法被 pickled。

    参考：[#2409](https://www.sqlalchemy.org/trac/ticket/2409)

+   **[orm] [bug]**

    修复了一个错误，即如果 MappedCollection 仅在使用了 @collection.internally_instrumented 的自定义子类中使用，则无法获得适当的集合工具。

    参考：[#2406](https://www.sqlalchemy.org/trac/ticket/2406)

+   **[orm] [bug]**

    修复了一个错误，即 SQL 适配机制在涉及联合继承、joinedload()、limit() 和列子句中的衍生函数的非常嵌套场景中会失败。

    参考：[#2419](https://www.sqlalchemy.org/trac/ticket/2419)

+   **[orm] [bug]**

    修复了 CascadeOptions 的 repr()，以包括 refresh-expire。同时重新设计了 CascadeOptions 为 <frozenset>。

    参考：[#2417](https://www.sqlalchemy.org/trac/ticket/2417)

+   **[orm] [bug]**

    改进了“声明式反射”示例，以支持单表继承、多次调用 prepare()、存在于备选模式中的表以及仅建立一部分类作为反射。

+   **[orm] [bug]**

    缩小了在 flush() 中应用的测试范围，仅在确实有更新时才检查对同一表中部分空主键的 UPDATE 是否存在。

    参考：[#2390](https://www.sqlalchemy.org/trac/ticket/2390)

+   **[orm] [bug]**

    修复了一个错误，即如果方法名与列名冲突，当映射器尝试检查方法对象上的 __get__() 方法时，会引发 TypeError。

    参考：[#2352](https://www.sqlalchemy.org/trac/ticket/2352)

### examples

+   **[examples] [bug]**

    在 Beaker 示例中修改了 _params_from_query() 函数，从完全编译的语句中提取绑定参数，作为获取包括列子句中的子查询等所有内容的快速手段。

### engine

+   **[engine] [feature]**

    为连接添加了“no_parameters=True”执行选项。如果没有参数，则会将语句传递给 cursor.execute(statement)，从而调用 DBAPI 在没有参数集合时的行为；对于 psycopg2 和 mysql-python，这意味着不解释字符串中的%符号。只有在使用此选项时才会发生这种情况，而不仅仅是参数列表为空，否则这将产生 SQL 表达式的不一致行为，这些表达式通常会转义百分号（并且在编译时，有时无法提前知道参数是否存在）。

    参考：[#2407](https://www.sqlalchemy.org/trac/ticket/2407)

+   **[engine] [feature]**

    添加了 pool_reset_on_return 参数到 create_engine，允许控制“连接返回”行为。还添加了新参数‘rollback’，‘commit’，None 到 pool.reset_on_return，以允许更多控制连接返回活动。

    参考：[#2378](https://www.sqlalchemy.org/trac/ticket/2378)

+   **[engine] [feature]**

    为 Engine、Connection 添加了一些不错的上下文管理器：

    ```py
    with engine.begin() as conn:
        # <work with conn in a transaction>
        ...
    ```

    并且：

    ```py
    with engine.connect() as conn:
        # <work with conn>
        ...
    ```

    在 engine.begin()上完成连接时，无论是提交还是回滚事务都会关闭连接。

+   **[engine] [bug]**

    在 MockConnection（即与 strategy=”mock”一起使用的连接）中添加了 execution_options()调用，作为参数的传递。

### sql

+   **[sql] [feature]**

    添加了对 SQL 标准通用表达式（CTE）的支持，允许将 SELECT 对象作为 CTE 源（尚不支持 DML）。这通过任何 select()构造上的 cte()方法调用。

    参考：[#1859](https://www.sqlalchemy.org/trac/ticket/1859)

+   **[sql] [bug]**

    修复了在使用特定类型的结果获取时，当使用 C 扩展时会发生内存泄漏的核心问题，特别是在调用 orm query.count()时。

    参考：[#2427](https://www.sqlalchemy.org/trac/ticket/2427)

+   **[sql] [bug]**

    修复了在行上基于属性访问时，非 C 版本会引发 AttributeError，C 版本会引发 NoSuchColumnError 的问题。现在在两种情况下都会引发 AttributeError。

    参考：[#2398](https://www.sqlalchemy.org/trac/ticket/2398)

+   **[sql] [bug]**

    添加了使用 Column 的.key 作为结果集行的字符串标识符的支持。.key 目前被列为列的“备用”名称，并且被具有该键值作为其常规名称的列所取代。对于 SQLAlchemy 的下一个主要版本，我们可能会反转这种优先顺序，使.key 优先，但目前尚未决定。

    参考：[#2392](https://www.sqlalchemy.org/trac/ticket/2392)

+   **[sql] [bug]**

    在 insert()或 update()构造的 values()子句中声明不存在的列时会发出警告。将在 0.8 版本中转为异常。

    参考：[#2413](https://www.sqlalchemy.org/trac/ticket/2413)

+   **[sql] [bug]**

    对于在 SELECT 语句中应用标签的方式进行了重大更改，允许“截断”标签，即在 Python 中生成的标签名称超过最大标识符长度时（请注意，可以通过 create_engine() 中的 label_length 进行配置），在子查询中正确引用，以及在结果集行中使用它们的原始 Python 名称。

    参考：[#2396](https://www.sqlalchemy.org/trac/ticket/2396)

+   **[sql] [bug]**

    修复了新的“autoload_replace”标志中的 bug，该 bug 会导致无法保留反射表的主键约束。

    参考：[#2402](https://www.sqlalchemy.org/trac/ticket/2402)

+   **[sql] [bug]**

    当传递的参数无法解释为列或表达式时，Index 将引发异常。当创建 Index 时没有传递任何列时将发出警告。

    参考：[#2380](https://www.sqlalchemy.org/trac/ticket/2380)

### mysql

+   **[mysql] [feature]**

    通过 Index 和 PrimaryKeyConstraint 的新 mysql_using 参数，支持 MySQL 索引和主键约束类型（即 USING），感谢 Diana Clarke。

    参考：[#2386](https://www.sqlalchemy.org/trac/ticket/2386)

+   **[mysql] [feature]**

    为所有 MySQL 方言添加了对“isolation_level”参数的支持。感谢 mu_mind 提供的补丁。

    参考：[#2394](https://www.sqlalchemy.org/trac/ticket/2394)

### sqlite

+   **[sqlite] [bug]**

    修复了 C 扩展中的一个 bug，即当作为整数返回的 Numeric 值不会应用字符串格式；这主要影响了 SQLite，因为它不维护数字比例设置。

    参考：[#2432](https://www.sqlalchemy.org/trac/ticket/2432)

### mssql

+   **[mssql] [feature]**

    添加了对 MSSQL INSERT、UPDATE 和 DELETE 表提示的支持，使用 UpdateBase 上的新 with_hint() 方法。

    参考：[#2430](https://www.sqlalchemy.org/trac/ticket/2430)

### oracle

+   **[oracle] [feature]**

    添加了一个新的 create_engine() 标志 coerce_to_decimal=False，禁用精确数值处理，可以通过将所有数值转换为 Decimal 来增加大量开销。

    参考：[#2399](https://www.sqlalchemy.org/trac/ticket/2399)

+   **[oracle] [bug]**

    添加了对 LONG 的缺失编译支持

    参考：[#2401](https://www.sqlalchemy.org/trac/ticket/2401)

+   **[oracle] [bug]**

    将“LEVEL”添加到 Oracle 的保留字列表中。

    参考：[#2435](https://www.sqlalchemy.org/trac/ticket/2435)

### orm

+   **[orm] [feature]**

    向 Session 添加了“no_autoflush”上下文管理器，与 with 一起使用：将临时禁用自动刷新。

+   **[orm] [feature]**

    向 Query 添加了 cte() 方法，调用 Core 中的公共表达式支持（见下文）。

    参考：[#1859](https://www.sqlalchemy.org/trac/ticket/1859)

+   **[orm] [feature]**

    在使用 query(sometable).filter_by(colname=value) 时，添加了查询表绑定列名的功能。

    参考：[#2400](https://www.sqlalchemy.org/trac/ticket/2400)

+   **[orm] [bug]**

    修复了事件注册 bug，主要表现为在与事件关联到 Session 类之后创建的 sessionmaker() 实例中未注册事件。

    参考：[#2424](https://www.sqlalchemy.org/trac/ticket/2424)

+   **[orm] [bug]**

    修复了一个 bug，当主键连接条件中存在“字面值”时，在某些需要多次渲染相同绑定参数名称的深度嵌套表达式编译时会引发错误。

    参考：[#2425](https://www.sqlalchemy.org/trac/ticket/2425)

+   **[orm] [bug]**

    移除了在对映射对象执行多重删除时检查受影响行数的检查。如果两行之间存在 ON DELETE CASCADE，则无法从 DBAPI 中获取准确的行数；在大多数 DBAPI 中，此特定计数不受支持，MySQLdb 是一个显著的例外。

    参考：[#2403](https://www.sqlalchemy.org/trac/ticket/2403)

+   **[orm] [bug]**

    修复了一个 bug，使用 attribute_mapped_collection 或 column_mapped_collection 的对象无法被 pickle。

    参考：[#2409](https://www.sqlalchemy.org/trac/ticket/2409)

+   **[orm] [bug]**

    修复了一个 bug，当仅在使用了 @collection.internally_instrumented 的自定义子类中使用 attribute_mapped_collection 或 column_mapped_collection 时，MappedCollection 将无法获得适当的集合仪器。

    参考：[#2406](https://www.sqlalchemy.org/trac/ticket/2406)

+   **[orm] [bug]**

    修复了一个 bug，当涉及到 joined-inheritance、joinedload()、limit() 和列子句中的派生函数的非常嵌套的情况时，SQL 适配机制会失败。

    参考：[#2419](https://www.sqlalchemy.org/trac/ticket/2419)

+   **[orm] [bug]**

    修复了 CascadeOptions 的 repr()，以包含 refresh-expire。还重新设计了 CascadeOptions 为 <frozenset>。

    参考：[#2417](https://www.sqlalchemy.org/trac/ticket/2417)

+   **[orm] [bug]**

    改进了“声明式反射”示例，以支持单表继承、多次调用 prepare()、存在于备选模式中的表，以及仅将部分类反映为反映的子集。

+   **[orm] [bug]**

    缩小了在 flush() 中应用的测试范围，以检查在一个表内部对部分为空的主键进行 UPDATE，只有在真正有 UPDATE 发生时才会发生。

    参考：[#2390](https://www.sqlalchemy.org/trac/ticket/2390)

+   **[orm] [bug]**

    修复了一个 bug，当方法名与列名冲突时，映射器尝试检查方法对象上的 __get__() 方法时会引发 TypeError。

    参考：[#2352](https://www.sqlalchemy.org/trac/ticket/2352)

### 示例

+   **[examples] [bug]**

    修改了 Beaker 示例中的 _params_from_query() 函数，从完全编译的语句中提取 bindparams，作为快速获取包括子查询在内的列子句中的一切的手段。

### 引擎

+   **[engine] [feature]**

    对连接添加了“no_parameters=True”执行选项。如果没有参数，将会将语句传递给`cursor.execute(statement)`，从而调用 DBAPI 在没有参数集合时的行为；对于 psycopg2 和 mysql-python，这意味着不解释字符串中的%符号。这仅在使用此选项时发生，并且不仅仅是如果参数列表为空，否则这将产生通常转义百分号的 SQL 表达式的不一致行为（并且在编译时，有时候无法提前知道参数是否存在）。

    参考：[#2407](https://www.sqlalchemy.org/trac/ticket/2407)

+   **[engine] [feature]**

    添加了 pool_reset_on_return 参数到 create_engine，允许控制“连接返回”行为。还添加了新参数‘rollback’，‘commit’，None 到 pool.reset_on_return，以允许更多对连接返回活动的控制。

    参考：[#2378](https://www.sqlalchemy.org/trac/ticket/2378)

+   **[engine] [feature]**

    为 Engine、Connection 添加了一些体面的上下文管理器：

    ```py
    with engine.begin() as conn:
        # <work with conn in a transaction>
        ...
    ```

    和：

    ```py
    with engine.connect() as conn:
        # <work with conn>
        ...
    ```

    在完成时关闭连接，使用 engine.begin()提交或回滚事务时产生错误。

+   **[engine] [bug]**

    对 MockConnection（即 strategy=”mock”时使用的）添加了 execution_options()调用，其作为参数的传递。

### sql

+   **[sql] [feature]**

    添加了对 SQL 标准通用表达式（CTE）的支持，允许 SELECT 对象作为 CTE 源（尚未支持 DML）。这通过任何 select()构造的 cte()方法调用。

    参考：[#1859](https://www.sqlalchemy.org/trac/ticket/1859)

+   **[sql] [bug]**

    修复了当 C 扩展与特定类型的结果提取一起使用时会发生的核心内存泄漏，特别是在调用 orm query.count()时。

    参考：[#2427](https://www.sqlalchemy.org/trac/ticket/2427)

+   **[sql] [bug]**

    修复了在行上进行基于属性的列访问会引发 AttributeError（对于非 C 版本）或 NoSuchColumnError（对于 C 版本）的问题。现在在两种情况下都引发 AttributeError。

    参考：[#2398](https://www.sqlalchemy.org/trac/ticket/2398)

+   **[sql] [bug]**

    添加了使用列的.key 作为结果集行中字符串标识符的支持。.key 目前被列为列的“替代”名称，并且被具有该键值作为其常规名称的列所取代。对于 SQLAlchemy 的下一个主要版本，我们可能会改变这种优先级，以便.key 优先，但目前尚未决定。

    参考：[#2392](https://www.sqlalchemy.org/trac/ticket/2392)

+   **[sql] [bug]**

    在 insert()或 update()构造的 values()子句中声明了不存在的列时会发出警告。将在 0.8 中移动到异常。

    参考：[#2413](https://www.sqlalchemy.org/trac/ticket/2413)

+   **[sql] [bug]**

    对于 SELECT 语句中应用到列的标签的应用方式发生了重大变化，允许“截断”标签，即在 Python 中生成的标签名称超过最大标识符长度时（请注意，这可以通过 create_engine() 中的 label_length 进行配置），在子查询中正确引用，并且在结果集行中以其原始的 Python 名称存在。

    参考：[#2396](https://www.sqlalchemy.org/trac/ticket/2396)

+   **[sql] [错误]**

    修复了新的“autoload_replace”标志中的错误，该标志会失败地保留反映表的主键约束。

    参考：[#2402](https://www.sqlalchemy.org/trac/ticket/2402)

+   **[sql] [错误]**

    当传递的参数无法解释为列或表达式时，Index 将引发错误。当创建 Index 时没有传递任何列时会发出警告。

    参考：[#2380](https://www.sqlalchemy.org/trac/ticket/2380)

### mysql

+   **[mysql] [功能]**

    通过新的 mysql_using 参数向 Index 和 PrimaryKeyConstraint 添加了对 MySQL 索引和主键约束类型（即 USING）的支持，感谢 Diana Clarke。

    参考：[#2386](https://www.sqlalchemy.org/trac/ticket/2386)

+   **[mysql] [功能]**

    向所有 MySQL 方言添加了“isolation_level”参数的支持。感谢 mu_mind 提供的补丁。

    参考：[#2394](https://www.sqlalchemy.org/trac/ticket/2394)

### sqlite

+   **[sqlite] [错误]**

    修复了 C 扩展中的错误，即当作为整数返回的 Numeric 值不会应用字符串格式时出错；这主要影响了不维护数字精度设置的 SQLite。

    参考：[#2432](https://www.sqlalchemy.org/trac/ticket/2432)

### mssql

+   **[mssql] [功能]**

    添加了对 MSSQL INSERT、UPDATE 和 DELETE 表提示的支持，使用 UpdateBase 上的新 with_hint() 方法。

    参考：[#2430](https://www.sqlalchemy.org/trac/ticket/2430)

### oracle

+   **[oracle] [功能]**

    添加了一个新的 create_engine() 标志 coerce_to_decimal=False，禁用精度数值处理，因为将所有数值转换为 Decimal 可以增加大量开销。

    参考：[#2399](https://www.sqlalchemy.org/trac/ticket/2399)

+   **[oracle] [错误]**

    添加了对 LONG 的编译支持的缺失。

    参考：[#2401](https://www.sqlalchemy.org/trac/ticket/2401)

+   **[oracle] [错误]**

    将“LEVEL”添加到 Oracle 的保留字列表中。

    参考：[#2435](https://www.sqlalchemy.org/trac/ticket/2435)

## 版本：0.7.5

发布日期：2012 年 1 月 28 日（周六）

### orm

+   **[orm] [功能]**

    向 declarative_base() 添加了“class_registry”参数。允许两个或更多声明基类共享相同的类名注册表。

+   **[orm] [功能]**

    query.filter() 接受多个条件，它们将通过 AND 连接，即 query.filter(x==y, z>q, …)

+   **[orm] [功能]**

    添加了新的功能，使关系加载器选项可以允许“默认”加载策略。将‘*’传递给 joinedload()、lazyload()、subqueryload() 或 noload() 中的任何一个，这将成为用于所有关系的加载策略，除非在查询中明确指定了其他策略。感谢新兴贡献者 Kent Bower 提供了详尽而写得很好的测试套件！

    参考：[#2351](https://www.sqlalchemy.org/trac/ticket/2351)

+   **[orm] [feature]**

    添加了新的声明式反射示例，演示了如何最好地将表反射与声明式混合使用，并使用了一些新功能。

    参考：[#2356](https://www.sqlalchemy.org/trac/ticket/2356)

+   **[orm] [bug]**

    修复了在失败的 flush 后建立修改的会话状态会作为随后由手动调用 rollback() 后自动开始的事务的一部分提交的问题。在 rollback() 中检查会话状态，如果存在新状态，则发出警告并第二次调用 restore_snapshot()，丢弃这些更改。

    参考：[#2389](https://www.sqlalchemy.org/trac/ticket/2389)

+   **[orm] [bug]**

    修复了从 0.7.4 版本开始的回归问题，即使用超类中已经被检测的列作为“polymorphic_on”时无法解析基础列的问题。

    参考：[#2345](https://www.sqlalchemy.org/trac/ticket/2345)

+   **[orm] [bug]**

    如果 xyzload_all() 与两个未连接的关系不当使用，则引发异常。

    参考：[#2370](https://www.sqlalchemy.org/trac/ticket/2370)

+   **[orm] [bug]**

    修复了事件监听器 event.listen(SomeClass) 强制进行完全不必要的映射器编译的 bug，使得在模块导入时设置事件非常困难（难道没有人注意到这个问题吗？）

    参考：[#2367](https://www.sqlalchemy.org/trac/ticket/2367)

+   **[orm] [bug]**

    修复了 hybrid_property 作为 any()、has() 中的关键字参数时无法正常工作的 bug。

+   **[orm] [bug]**

    确保所有 ORM 异常都可以被 pickle，以实现多进程兼容性。

    参考：[#2371](https://www.sqlalchemy.org/trac/ticket/2371)

+   **[orm] [bug]**

    实现了标准的“无法设置属性”/“无法删除属性” AttributeError，当在混合属性上使用 setattr/delattr 时，如果没有定义 fset 或 fdel。

    参考：[#2353](https://www.sqlalchemy.org/trac/ticket/2353)

+   **[orm] [bug]**

    修复了反序列化对象时，如果对象在 __eq__() 或类似方法中需要 ORM 属性访问，未设置足够状态以正确工作的 bug。

    参考：[#2362](https://www.sqlalchemy.org/trac/ticket/2362)

+   **[orm] [bug]**

    修复了“合并”级联可能会错误解释未加载属性的 bug，如果在 relationship() 中使用了 load_on_pending 标志。感谢 Kent Bower 提供的测试。

    参考：[#2374](https://www.sqlalchemy.org/trac/ticket/2374)

+   **[orm]**

    修复了从 0.6 版本开始的回归问题，即如果在挂起对象上需要发出非“get()”懒惰子句的“load_on_pending”relationship()标志，则加载将失败。

### 示例

+   **[examples] [feature]**

    简化了版本示例，使用了一个声明性 mixin 和一个事件监听器，而不是元类+SessionExtension。

    引用：[#2313](https://www.sqlalchemy.org/trac/ticket/2313)

+   **[examples] [bug]**

    修复了在删除表之前关闭会话的大型集合.py 的问题。

    引用：[#2346](https://www.sqlalchemy.org/trac/ticket/2346)

### 引擎

+   **[engine] [bug]**

    为 StatementError、DBAPIError、列错误添加了 __reduce__，以便异常可被 pickle 化，例如在使用多进程时。但是，目前并非所有的 DBAPI 都支持这一点，比如 psycopg2。

    引用：[#2371](https://www.sqlalchemy.org/trac/ticket/2371)

+   **[engine] [bug]**

    当向 SQLite 使用的任何日期/时间处理器传递非字符串或无效字符串时，改进了错误消息，包括 C 和 Python 版本。

    引用：[#2382](https://www.sqlalchemy.org/trac/ticket/2382)

+   **[engine] [bug]**

    修复了一个 bug，即一个名为“<a>_<b>”的表绑定的 Column 对象与一个标记为“<tablename>_<colname>”的列匹配时，在定位结果集行时可能会不当匹配。

    引用：[#2377](https://www.sqlalchemy.org/trac/ticket/2377)

+   **[engine] [bug]**

    修复了“mock”策略中未调用正确 DDL 访问方法的 bug，导致“CREATE/DROP SEQUENCE”语句重复。

    引用：[#2384](https://www.sqlalchemy.org/trac/ticket/2384)

### sql

+   **[sql] [feature]**

    新的反射功能“autoload_replace”；当在 Table 上设置为 False 时，可以在不替换现有列的情况下自动加载 Table。允许构建更灵活的 Table 构建/反射链，包括它有助于将声明性与表反射结合起来。请参阅维基上的新示例。

    引用：[#2356](https://www.sqlalchemy.org/trac/ticket/2356)

+   **[sql] [feature]**

    在 sqlalchemy.sql 命名空间中添加了“false()”和“true()”表达式构造，尽管目前还不是 __all__ 的一部分。

+   **[sql] [feature]**

    特定方言的编译器现在对所有类型/语句编译问题引发 CompileError，而不是 InvalidRequestError 或 ArgumentError。对于 CREATE TABLE 的 DDL，将重新引发 CompileError 以包含有问题的列的表/列信息。

    引用：[#2361](https://www.sqlalchemy.org/trac/ticket/2361)

+   **[sql] [bug]**

    改进了 add_column()的 API，如果将相同的列添加到其自己的表中，则不会引发错误，并且约束不会加倍。还有助于一些反射/声明性模式。

    引用：[#2356](https://www.sqlalchemy.org/trac/ticket/2356)

+   **[sql] [bug]**

    修复了如果给 bindparam()传递 required=True，但语句没有任何参数，则不会引发“required”异常的问题。

    参考：[#2381](https://www.sqlalchemy.org/trac/ticket/2381)

### mysql

+   **[mysql] [bug]**

    修复了过滤掉非反射“PARTITION”指令警告的正则表达式，感谢 George Reilly

    参考：[#2376](https://www.sqlalchemy.org/trac/ticket/2376)

### sqlite

+   **[sqlite] [bug]**

    在 SQLite 中，FK 约束的“name”反映为“None”，而不是“0”或其他整数值。在任何情况下，SQLite 似乎都不支持约束命名。

    参考：[#2364](https://www.sqlalchemy.org/trac/ticket/2364)

+   **[sqlite] [bug]**

    sql.false()和 sql.true()在 sqlite 中分别编译为 0 和 1

    参考：[#2368](https://www.sqlalchemy.org/trac/ticket/2368)

+   **[sqlite] [bug]**

    在 SQLite 方言中，当获取表名和视图名时，删除了一个错误的“raise”，在那里逻辑是回退到一个没有“sqlite_temp_master”表的旧版本的 SQLite。

### mssql

+   **[mssql] [bug]**

    调整了在 mssql.TIME 类型中使用的正则表达式，以确保仅接收值的“微秒”部分为六位数，这是 Python 的 datetime.time()所期望的。请注意，目前似乎无法使用 pyodbc 发送微秒。

    参考：[#2340](https://www.sqlalchemy.org/trac/ticket/2340)

+   **[mssql] [bug]**

    根据报告，基于 pymssql 的“30 char”限制已被取消，因为它现在做得更好。pymssql 尚未经过充分测试，由于 DBAPI 仍在变化中，目前还不清楚此驱动程��的状态以及 SQLAlchemy 的实现应如何调整。

    参考：[#2347](https://www.sqlalchemy.org/trac/ticket/2347)

### oracle

+   **[oracle] [bug]**

    将 ORA-03135 添加到永无止境的 oracle“连接丢失”错误列表中

    参考：[#2388](https://www.sqlalchemy.org/trac/ticket/2388)

### misc

+   **[bug] [core]**

    更改了由映射器使用的 LRUCache，用于缓存 INSERT/UPDATE/DELETE 语句，以使用递增计数器而不是时间戳来跟踪条目，以提高可靠性，而不是使用 time.time()，这可能会导致某些平台上的测试失败。

    参考：[#2379](https://www.sqlalchemy.org/trac/ticket/2379)

+   **[bug] [core]**

    在调用池连接代理的弱引用回调之前，添加了对“finalize”函数的布尔检查，以避免在应用程序退出时和 gc 在调用弱引用回调之前从模块中删除函数时发出警告，此函数为 None。

    参考：[#2383](https://www.sqlalchemy.org/trac/ticket/2383)

+   **[bug] [py3k]**

    修复了对 util.py3k 标志的不当使用，并将其重命名为 util.py3k_warning，因为此标志仅用于检测导入限制系列“-3”标志。

    参考：[#2348](https://www.sqlalchemy.org/trac/ticket/2348)

### orm

+   **[orm] [feature]**

    在 declarative_base()中添加了“class_registry”参数。允许两个或更多声明基类共享相同的类名注册表。

+   **[orm] [feature]**

    query.filter()接受多个标准，这些标准将通过 AND 连接，即 query.filter(x==y, z>q, …)

+   **[orm] [feature]**

    添加了关系加载器选项的新功能，允许“默认”加载策略。将‘*’传递给 joinedload()、lazyload()、subqueryload()或 noload()中的任何一个，这将成为用于所有关系的加载策略，除了在查询中明确指定的那些关系之外。感谢新兴的贡献者 Kent Bower 为详尽而逻辑清晰的测试套件！

    引用：[#2351](https://www.sqlalchemy.org/trac/ticket/2351)

+   **[orm] [feature]**

    添加了新的声明式反射示例，说明了如何最好地将表反射与声明式混合使用，并使用了一些新功能。

    引用：[#2356](https://www.sqlalchemy.org/trac/ticket/2356)

+   **[orm] [bug]**

    修复了在失败的 flush 之后建立的修改会作为随后由手动调用 rollback()自动开始的事务的一部分提交的问题。在 rollback()内部检查会话的状态，如果存在新状态，则发出警告并第二次调用 restore_snapshot()，丢弃这些更改。

    引用：[#2389](https://www.sqlalchemy.org/trac/ticket/2389)

+   **[orm] [bug]**

    修复了从 0.7.4 开始的回归，即使用已经被超类作为“polymorphic_on”的列失败地解析了底层列的问题。

    引用：[#2345](https://www.sqlalchemy.org/trac/ticket/2345)

+   **[orm] [bug]**

    如果 xyzload_all()与两个未连接的关系不合适地使用，则引发异常。

    引用：[#2370](https://www.sqlalchemy.org/trac/ticket/2370)

+   **[orm] [bug]**

    修复了事件监听（event.listen(SomeClass)）强制进行了完全不必要的映射器编译的错误，导致在模块导入时设置事件非常困难（竟然没有人注意到这个问题？？）

    引用：[#2367](https://www.sqlalchemy.org/trac/ticket/2367)

+   **[orm] [bug]**

    修复了 hybrid_property 在 any()、has()中无法作为关键字参数工作的错误。

+   **[orm] [bug]**

    确保所有 ORM 异常都可以被 pickle，以实现多处理兼容性。

    引用：[#2371](https://www.sqlalchemy.org/trac/ticket/2371)

+   **[orm] [bug]**

    实现了在混合类型没有定义 fset 或 fdel 时，使用 setattr/delattr 设置或删除混合类型属性时引发标准“无法设置属性”/“无法删除属性”AttributeError。

    引用：[#2353](https://www.sqlalchemy.org/trac/ticket/2353)

+   **[orm] [bug]**

    修复了无法正确设置未反 pickle 对象的状态的错误，在 __eq__()或类似方法中需要 ORM 属性访问时，这些对象需要在可变对象扩展中建立的 unpickle()事件内正确工作。

    引用：[#2362](https://www.sqlalchemy.org/trac/ticket/2362)

+   **[orm] [bug]**

    修复了“merge”级联可能错误解释未加载属性的错误，如果使用 relationship()的 load_on_pending 标志。感谢 Kent Bower 提供的测试。

    引用：[#2374](https://www.sqlalchemy.org/trac/ticket/2374)

+   **[orm]**

    修复了从 0.6 版本中的回归，即如果在挂起对象上需要发出非“get()”延迟子句的“load_on_pending”relationship()标志被使用，它将无法加载。

### 例子

+   **[例子] [feature]**

    简化了版本示例，使用了一个声明性的 mixin 和一个事件监听器，而不是一个元类+SessionExtension。

    参考：[#2313](https://www.sqlalchemy.org/trac/ticket/2313)

+   **[例子] [bug]**

    修复了 large_collection.py 在删除表之前关闭会话的 bug。

    参考：[#2346](https://www.sqlalchemy.org/trac/ticket/2346)

### engine

+   **[engine] [bug]**

    为 StatementError、DBAPIError、列错误添加了 __reduce__，以便异常可被 pickle 化，例如在使用多进程时。但是，目前并非所有 DBAPI 都支持这一点，比如 psycopg2。

    参考：[#2371](https://www.sqlalchemy.org/trac/ticket/2371)

+   **[engine] [bug]**

    当传递给 SQLite 的任何日期/时间处理器的非字符串或无效字符串时，改进了错误消息，包括 C 和 Python 版本。

    参考：[#2382](https://www.sqlalchemy.org/trac/ticket/2382)

+   **[engine] [bug]**

    修复了一个 bug，即一个绑定到表的 Column 对象命名为“<a>_<b>”，与标记为“<tablename>_<colname>”的列匹配时，可能在目标结果集行中不恰当地匹配。

    参考：[#2377](https://www.sqlalchemy.org/trac/ticket/2377)

+   **[engine] [bug]**

    修复了“模拟”策略中的错误，其中正确的 DDL 访问方法未被调用，导致“CREATE/DROP SEQUENCE”语句被重复。

    参考：[#2384](https://www.sqlalchemy.org/trac/ticket/2384)

### sql

+   **[sql] [feature]**

    新的反射功能“autoload_replace”；当在 Table 上设置为 False 时，可以自动加载 Table 而不替换现有列。允许构建更灵活的 Table 构建/反射���，包括它有助于将声明性与表反射结合起来。请参阅维基上的新示例。

    参考：[#2356](https://www.sqlalchemy.org/trac/ticket/2356)

+   **[sql] [feature]**

    在 sqlalchemy.sql 命名空间中添加了“false()”和“true()”表达式构造，尽管目前还不是 __all__ 的一部分。

+   **[sql] [feature]**

    特定方言的编译器现在对所有类型/语句编译问题引发 CompileError，而不是 InvalidRequestError 或 ArgumentError。对于 CREATE TABLE 的 DDL，将重新引发 CompileError 以包含有问题的列的表/列信息。

    参考：[#2361](https://www.sqlalchemy.org/trac/ticket/2361)

+   **[sql] [bug]**

    改进了 add_column()的 API，如果将相同的列添加到其自身的表中，不会引发错误，约束也不会重复。还有助于一些反射/声明性模式。

    参考：[#2356](https://www.sqlalchemy.org/trac/ticket/2356)

+   **[sql] [bug]**

    修复了“required”异常不会对 bindparam()使用 required=True 时引发，如果语句根本没有给出参数。

    参考：[#2381](https://www.sqlalchemy.org/trac/ticket/2381)

### mysql

+   **[mysql] [bug]**

    修复了过滤��非反射“PARTITION”指令警告的正则表达式，感谢 George Reilly

    参考：[#2376](https://www.sqlalchemy.org/trac/ticket/2376)

### sqlite

+   **[sqlite] [bug]**

    在 SQLite 中，外键约束的“name”反映为“None”，而不是“0”或其他整数值。在任何情况下，SQLite 似乎都不支持约束命名。

    参考：[#2364](https://www.sqlalchemy.org/trac/ticket/2364)

+   **[sqlite] [bug]**

    在 sqlite 中，sql.false()和 sql.true()分别编译为 0 和 1。

    参考：[#2368](https://www.sqlalchemy.org/trac/ticket/2368)

+   **[sqlite] [bug]**

    在 SQLite 方言中，当获取表名和视图名时，删除了一个错误的“raise”，在那里逻辑已经设置好，以便回退到一个没有“sqlite_temp_master”表的旧版本的 SQLite。

### mssql

+   **[mssql] [bug]**

    调整了 mssql.TIME 类型中使用的正则表达式，以确保仅接收“microseconds”部分的六位数字，这是 Python 的 datetime.time()所期望的。请注意，目前似乎无法使用 pyodbc 发送微秒。

    参考：[#2340](https://www.sqlalchemy.org/trac/ticket/2340)

+   **[mssql] [bug]**

    pymssql 上的“30 char”限制已被取消，根据报告称现在的情况更好。由于 pymssql 没有经过充分测试，而且 DBAPI 仍在变化中，目前还不清楚该驱动程序的状态以及 SQLAlchemy 的实现应该如何调整。

    参考：[#2347](https://www.sqlalchemy.org/trac/ticket/2347)

### oracle

+   **[oracle] [bug]**

    将 ORA-03135 添加到永无止境的 Oracle“连接丢失”错误列表中

    参考：[#2388](https://www.sqlalchemy.org/trac/ticket/2388)

### 杂项

+   **[bug] [core]**

    将用于缓存 INSERT/UPDATE/DELETE 语句的映射器的 LRUCache 更改为使用递增计数器而不是时间戳来跟踪条目，以提高可靠性，而不是使用 time.time()，后者可能会导致某些平台上的测试失败。

    参考：[#2379](https://www.sqlalchemy.org/trac/ticket/2379)

+   **[bug] [core]**

    在调用池连接代理的弱引用回调之前，添加了对“finalize”函数的布尔检查，以避免在应用程序退出时发出警告，即当应用程序退出时，gc 已将该函数从模块中删除，而弱引用回调尚未被调用。

    参考：[#2383](https://www.sqlalchemy.org/trac/ticket/2383)

+   **[bug] [py3k]**

    修复了 util.py3k 标志的不当使用，并将其重命名为 util.py3k_warning，因为该标志仅用于检测导入限制系列“-3”标志。

    参考：[#2348](https://www.sqlalchemy.org/trac/ticket/2348)

## 0.7.4

发布日期：2011 年 12 月 09 日

### orm

+   **[orm] [feature]**

    polymorphic_on 现在接受许多新类型的值：

    > +   未映射的独立表达式
    > +   
    > +   column_property()对象
    > +   
    > +   任何 column_property() 的字符串名称或映射列的属性名称

    文档包含了一个使用 case() 构造的示例，这很可能是一个常见的构造。以及部分

    多态 _on 中的独立表达式会传播到单表继承的子类中，以便它们被用于 WHERE / JOIN 子句来限制行为该子类的行为，这是通常的行为。

    参考：[#2238](https://www.sqlalchemy.org/trac/ticket/2238)，[#2345](https://www.sqlalchemy.org/trac/ticket/2345)

+   **[orm] [feature]**

    IdentitySet 支持 `-` 运算符，与 `difference()` 相同，处理 Session.dirty 等情况时非常方便。

    参考：[#2301](https://www.sqlalchemy.org/trac/ticket/2301)

+   **[orm] [feature]**

    添加了 Column autoincrement 的新值称为 “ignore_fk”，可用于强制对仍然是 ForeignKeyConstraint 一部分的列进行自增。关系文档中的新示例说明了其用法。

+   **[orm] [bug]**

    修复了在从陈旧的一对多中删除时，从多对一中“弹出”值时的反向引用行为 - 该操作被跳过，因为多对一已经被更新。

    参考：[#2315](https://www.sqlalchemy.org/trac/ticket/2315)

+   **[orm] [bug]**

    几年后重新进行此操作，对“X 是否为 Y 的父级”功能添加了更多细粒度，这在确定是否需要将“Y”的 FK “null”掉以及是否应该使用 delete-orphan 级联删除时使用。测试现在考虑到了父级的 Python 标识以及其标识键，以查看 Y 的最后已知父级是否确实为 X。如果无法做出决定，则会引发 StaleDataError。引发此错误的条件非常罕见，要求以前的父级已被垃圾回收，并且以前可能会不恰当地更新/删除已经移至新父级的记录，尽管以前可能存在一些“静默成功”的情况，但现在在面对不确定性时可能会引发。到期的 “Y” 会重置 “父级” 跟踪器，这意味着 X.remove(Y) 可能会删除 Y，即使 X 已经过期，但这与以前的行为相同；在这种情况下也建议过期 X。

    参考：[#2264](https://www.sqlalchemy.org/trac/ticket/2264)

+   **[orm] [bug]**

    修复了在查询.get() 中对用户映射对象进行布尔上下文中的不适当评估。同时也在 0.6.9 版本中。

    参考：[#2310](https://www.sqlalchemy.org/trac/ticket/2310)

+   **[orm] [bug]**

    对 PASSIVE_RETURN_NEVER_SET 符号添加了缺失的逗号。

    参考：[#2304](https://www.sqlalchemy.org/trac/ticket/2304)

+   **[orm] [bug]**

    Cls.column.collate(“some collation”) 现在可以工作。同时也在 0.6.9 版本中。

    参考：[#1776](https://www.sqlalchemy.org/trac/ticket/1776)

+   **[orm] [bug]**

    组合属性的值现在在插入或更新操作后会过期，而不是在原地重新生成。这确保了在刷新时过期的列值将首先被加载，然后再使用该值重新生成组合属性。

    参考：[#2309](https://www.sqlalchemy.org/trac/ticket/2309)

+   **[orm] [bug]**

    修复了当在访问时加载组合值时，即使所有列值已经存在，也会触发“refresh”事件的问题，这是适当的。这修复了依赖于“load”事件确保 _parents 字典是最新的“mutable”扩展，修复了的问题。感谢 Scott Torborg 在这里提供的测试用例。

    参考：[#2308](https://www.sqlalchemy.org/trac/ticket/2308), [#2309](https://www.sqlalchemy.org/trac/ticket/2309)

+   **[orm] [bug]**

    修复了一个 bug，即当一个子类使用具体继承与新的 ConcreteBase 或 AbstractConcreteBase 时，无法将深于一级的子类应用于每个基类的“多态加载器”。

    参考：[#2312](https://www.sqlalchemy.org/trac/ticket/2312)

+   **[orm] [bug]**

    修复了一个 bug，即当一个子类使用新的 AbstractConcreteBase 时，其子类无法在生成“base”映射器时获取正确的“base_mapper”属性，从而导致后续失败。

    参考：[#2312](https://www.sqlalchemy.org/trac/ticket/2312)

+   **[orm] [bug]**

    修复了当针对 ORM 级别的列创建 column_property()时，在生成某些类型的 joined-inh 连接时可能会将其视为不同的实体的 bug。

    参考：[#2316](https://www.sqlalchemy.org/trac/ticket/2316)

+   **[orm] [bug]**

    修复了当不小心将元组传递给 session.query()时引发的错误格式化。也在 0.6.9 中修复。

    参考：[#2297](https://www.sqlalchemy.org/trac/ticket/2297)

+   **[orm] [bug]**

    对单表继承子类进行 query.join()调用现在被跟踪，并用于消除通常附加的额外的 WHERE.. IN 条件，因为连接应该适应它。这允许对单表子类进行 OUTER JOIN 以产生正确的结果，并且在处理单表继承连接时将产生更少的 WHERE 条件。

    参考：[#2328](https://www.sqlalchemy.org/trac/ticket/2328)

+   **[orm] [bug]**

    __table_args__ 现在可以作为空元组传递，也可以作为空字典传递。感谢 Fayaz Yusuf Khan 提供的补丁。

    参考：[#2339](https://www.sqlalchemy.org/trac/ticket/2339)

+   **[orm] [bug]**

    更新了设置 delete-orphan 而没有 delete 时的警告消息，不再提及 0.6，因为我们从未考虑将其升级为异常。理想情况下，这可能更好地作为一个异常，但无论如何都不是关键问题。

    参考：[#2325](https://www.sqlalchemy.org/trac/ticket/2325)

+   **[orm] [bug]**

    修复了在 get_history() 中引用没有值的复合属性时的 bug；增加了对 get_history() 的覆盖范围，关于复合属性，否则只是一个用户自定义函数。

### 示例

+   **[示例] [bug]**

    修复了 history_meta.py 示例中的 bug，其中“unique”标志未从生成列放置到基类上的单表继承子类中移除。

### 引擎

+   **[引擎] [bug]**

    修复了当事务.rollback() 在事务是两阶段或保存点事务时，如果连接无效，则会在无效的连接上抛出错误的 bug。对于普通事务，如果连接无效，则 rollback() 是一个空操作，因此虽然不是 100% 清楚它是否应该是一个空操作，但至少现在接口是一致的。

    参考：[#2317](https://www.sqlalchemy.org/trac/ticket/2317)

### sql

+   **[sql] [特性]**

    现在，update() 构造可以容纳 WHERE 子句中的多个表，这将生成一个被 PostgreSQL 和 MSSQL 识别的“UPDATE..FROM” 构造。在 MySQL 上编译时，将生成“UPDATE t1, t2, ..”。如果在“values”参数或生成方法中使用 Column 对象作为键，则 MySQL 还可以针对 SET 子句中的多个表进行渲染。

    参考：[#1944](https://www.sqlalchemy.org/trac/ticket/1944), [#2166](https://www.sqlalchemy.org/trac/ticket/2166)

+   **[sql] [特性]**

    添加了一个名为“python_type”的类型访问器，如果已知，返回特定 TypeEngine 实例的基本 Python 类型对象，否则引发 NotImplementedError。

    参考：[#77](https://www.sqlalchemy.org/trac/ticket/77)

+   **[sql] [bug]**

    相关的，对于关于 select() 中的“from”列表的更改进行了一些调整。_froms 集合不再被记忆化，因为这简化了各种用例，并消除了在表已经在表达式中使用后附加列时需要“警告”的需要 - select() 构造现在将始终生成正确的表达式。这里可能没有真正的性能损失；select() 对象几乎总是临时制作的，并且希望优化 select() 的系统将使用“compiled_cache”功能。调用 select.bind 时会减少一个命中，但绝大多数用户不应该使用“bound metadata” :).

    参考：[#2261](https://www.sqlalchemy.org/trac/ticket/2261), [#2316](https://www.sqlalchemy.org/trac/ticket/2316)

+   **[sql] [bug]**

    进一步调整了对修复的修正，以便生成方法在克隆后更好地工作（尽管这几乎不是一个使用案例）。特别是这允许 with_only_columns() 表现更一致。添加了额外的文档到 with_only_columns()，以澄清预期行为，这是由于结果而改变的。

    参考：[#2261](https://www.sqlalchemy.org/trac/ticket/2261), [#2319](https://www.sqlalchemy.org/trac/ticket/2319)

### 模式

+   **[模式] [特性]**

    添加了对远程“模式”的新支持：

+   **[模式] [特性]**

    Table 上的“extend_existing”标志现在允许反射过程对已经定义的 Table 对象生效；当 autoload=True 和 extend_existing=True 都设置时，将从 Table 中反射出完整的列集，然后将覆盖已经存在的列，而不是不进行任何操作。然而，直接在 autoload 运行中存在的列将像往常一样使用。

    参考：[#1410](https://www.sqlalchemy.org/trac/ticket/1410)

+   **[模式] [bug]**

    修复了 TypeDecorator 在使用“切换”类型的 TypeDecorator（如 CHAR/UUID 类型）时返回过时值的 bug。

+   **[模式] [bug]**

    修复了 Inspector.get_table_names 中“order_by='foreign_key'”选项未正确实现排序的 bug，替换为现有的排序算法

+   **[模式] [bug]**

    如果存在列级别的 CHECK 约束的“name”，则现在在 CREATE TABLE 语句中使用“CONSTRAINT <name> CHECK <expression>”来呈现。

    参考：[#2305](https://www.sqlalchemy.org/trac/ticket/2305)

+   **[模式]**

    MetaData() 接受“schema”和“quote_schema”参数，这些参数将应用于同名参数的 Table 或 Sequence，如果这些参数保持默认值`None`。

+   **[模式]**

    Sequence 接受“quote_schema”参数

+   **[模式]**

    对于 Table，如果 schema 参数明确为“None”，则 tometadata() 方法将使用传入 MetaData 的“schema” 创建新 Table

+   **[模式]**

    添加了 CreateSchema 和 DropSchema DDL 构造 - 这些仅接受模式的字符串名称和一个“quote”标志。

+   **[模式]**

    当使用默认的“schema”与 MetaData 时，ForeignKey 在定位远程表时也会假定“default” schema。这允许将 MetaData 上的“schema”参数应用于任何一组 Table 对象，否则这些对象没有“schema”。

+   **[模式]**

    在 dialect 上实现了一个“has_schema”方法，但目前只在 PostgreSQL 上有效。感谢 Manlio Perillo。

    参考：[#1679](https://www.sqlalchemy.org/trac/ticket/1679)

### postgresql

+   **[postgresql] [feature]**

    添加了 create_type 构造参数到 pg.ENUM。当为 False 时，不会执行 CREATE/DROP 或检查类型作为表创建/删除事件的一部分；只有直接调用 create()/drop() 方法时才会执行此操作。有助于 Alembic 的“离线”脚本。

+   **[postgresql] [bug]**

    PostgreSQL dialect 缓存了特定名称的 ENUM 在创建/删除序列期间已处理。这使得在不需要任何“checkfirst”调用的情况下可以正常工作，并且在打开“checkfirst”时只需要检查 ENUM 一次。

    参考：[#2311](https://www.sqlalchemy.org/trac/ticket/2311)

### mysql

+   **[mysql] [bug]**

    Unicode 调整使最新的 pymysql（0.4 之后）在 Python 2 上通过了 100% 的测试。

### mssql

+   **[mssql] [feature]**

    解除了 SQL Server 对 SAVEPOINT 的限制。所有测试都通过了，但不清楚是否存在更深层次的问题。

    参考：[#822](https://www.sqlalchemy.org/trac/ticket/822)

+   **[mssql] [错误]**

    修复了在 MSSQL 上未正确实现的 with_hint() 功能 - 通常用于“WITH (NOLOCK)”提示（你不应该使用它！改用快照隔离 :)

    参考：[#2336](https://www.sqlalchemy.org/trac/ticket/2336)

+   **[mssql] [错误]**

    为 _need_decimal_fix 选项使用新的 pyodbc 版本检测。

    参考：[#2318](https://www.sqlalchemy.org/trac/ticket/2318)

+   **[mssql] [错误]**

    不要在 SQL Server 2000 上将“表名”转换为 NVARCHAR。仍然不太清楚在这里完全使 PyODBC 与 FreeTDS 0.91 兼容需要哪些咒语。

    参考：[#2343](https://www.sqlalchemy.org/trac/ticket/2343)

+   **[mssql] [错误]**

    解码在检索索引名称列表和这些索引中的列名称时的传入值。

    参考：[#2269](https://www.sqlalchemy.org/trac/ticket/2269)

### 杂项

+   **[功能] [扩展]**

    在混合文档中添加了一个“转换器”的示例 - 一个混合，它返回一个查询转换的可调用对象与自定义比较器结合使用。在 Query 上使用的新方法称为 with_transformation()。这里的用例相当实验性，但只需要在 Query 中添加一行代码。

+   **[错误] [pyodbc]**

    基于 pyodbc 的方言现在可以准确解析 pyodbc 字符串，包括“py3-3.0.1-beta4”等宝藏。

    参考：[#2318](https://www.sqlalchemy.org/trac/ticket/2318)

+   **[错误] [扩展]**

    当没有“默认”编译处理程序时，@compiles 装饰器引发一条信息丰富的错误消息，而不是 KeyError。

### orm

+   **[orm] [功能]**

    polymorphic_on 现在接受许多新种类的值：

    > +   否则不被映射的独立表达式
    > +   
    > +   column_property() 对象
    > +   
    > +   任何 column_property() 的字符串名称或映射 Column 的属性名称

    文档包括一个使用 case() 构造的示例，这可能是一个常见的构造用法。和部分

    polymorphic_on 中的独立表达式传播到单表继承子类，因此它们被用于 WHERE /JOIN 子句以将行限制为该子类，这是通常的行为。

    参考：[#2238](https://www.sqlalchemy.org/trac/ticket/2238)，[#2345](https://www.sqlalchemy.org/trac/ticket/2345)

+   **[orm] [功能]**

    IdentitySet 支持 - 运算符与 difference() 相同，处理 Session.dirty 等时非常方便。

    参考：[#2301](https://www.sqlalchemy.org/trac/ticket/2301)

+   **[orm] [功能]**

    添加了称为“ignore_fk”的 Column 自增值的新值，可用于强制在仍然属于 ForeignKeyConstraint 的列上自增。关系文档中的新示例说明了其用法。

+   **[orm] [错误]**

    修复了当从过时的一对多中删除时从多对一中“弹出”值的反向引用行为 - 该操作被跳过，因为多对一已经被更新。

    参考：[#2315](https://www.sqlalchemy.org/trac/ticket/2315)

+   **[orm] [错误]**

    几年后，对“X 是否是 Y 的父级”功能增加了更多细粒度的控制，这在确定“Y”的外键是否也需要“被置空”以及是否应该使用删除孤立级联时使用。测试现在考虑了父级的 Python 标识以及其标识键，以查看 Y 的上一个已知父级是否绝对是 X。如果无法做出决定，则会引发 StaleDataError。引发此错误的条件相当罕见，要求以前的父级已被垃圾收集，并且以前可能会不适当地更新/删除已经移至新父级的记录，尽管以前可能存在一些“静默成功”的情况，但现在在面对模糊性时将会引发。使“Y”过期会重置“父”跟踪器，这意味着 X.remove(Y)然后可能会删除 Y，即使 X 已过期，但这与以前的行为相同；建议在这种情况下也使 X 过期。

    引用：[#2264](https://www.sqlalchemy.org/trac/ticket/2264)

+   **[orm] [bug]**

    修复了在 query.get()中不适当评估用户映射对象的布尔上下文的错误。还在 0.6.9 中。

    引用：[#2310](https://www.sqlalchemy.org/trac/ticket/2310)

+   **[orm] [bug]**

    添加了缺少的逗号到 PASSIVE_RETURN_NEVER_SET 符号

    引用：[#2304](https://www.sqlalchemy.org/trac/ticket/2304)

+   **[orm] [bug]**

    Cls.column.collate(“some collation”)现在可以工作。也在 0.6.9 中。

    引用：[#1776](https://www.sqlalchemy.org/trac/ticket/1776)

+   **[orm] [bug]**

    现在，复合属性的值在插入或更新操作后会过期，而不是在原地重新生成。这确保了在刷新时过期的列值会首先被加载，然后才会使用该值重新生成复合值。

    引用：[#2309](https://www.sqlalchemy.org/trac/ticket/2309)

+   **[orm] [bug]**

    修复了当访问时加载复合值时也会发出“刷新”事件的问题，即使所有列值已经存在，这是合适的。这修复了依赖于“load”事件来确保 _parents 字典是最新的“mutable”扩展，修复。感谢 Scott Torborg 提供的测试用例。

    引用：[#2308](https://www.sqlalchemy.org/trac/ticket/2308)，[#2309](https://www.sqlalchemy.org/trac/ticket/2309)

+   **[orm] [bug]**

    修复了一个错误，即使用新的 ConcreteBase 或 AbstractConcreteBase 与具体继承一起使用的子类的子类将无法将更深层次的子类应用于每个基类的“多态加载器”。

    引用：[#2312](https://www.sqlalchemy.org/trac/ticket/2312)

+   **[orm] [bug]**

    修复了使用新的 AbstractConcreteBase 的子类的子类在生成“基”映射器时无法获得正确的“base_mapper”属性的错误，从而导致后来的失败。

    引用：[#2312](https://www.sqlalchemy.org/trac/ticket/2312)

+   **[orm] [bug]**

    修复了一个错误，即当针对 ORM 级别的列创建 column_property() 时，在生成某些类型的联合继承联接时，可能会将其视为单独的实体。

    参考：[#2316](https://www.sqlalchemy.org/trac/ticket/2316)

+   **[orm] [错误]**

    修复了当元组被错误地传递给 session.query() 时引发的错误格式化问题。也适用于 0.6.9 版本。

    参考：[#2297](https://www.sqlalchemy.org/trac/ticket/2297)

+   **[orm] [错误]**

    对单表继承子类的 query.join() 现在被跟踪，并且用于消除通常在单表继承时附加的额外 WHERE.. IN 准则，因为联接应该适应它。这允许对单表子类进行 OUTER JOIN 以产生正确的结果，并且在处理单表继承联接时通常会生成更少的 WHERE 准则。

    参考：[#2328](https://www.sqlalchemy.org/trac/ticket/2328)

+   **[orm] [错误]**

    __table_args__ 现在可以作为空元组和空字典传递。感谢 Fayaz Yusuf Khan 提供的修补程序。

    参考：[#2339](https://www.sqlalchemy.org/trac/ticket/2339)

+   **[orm] [错误]**

    更新了设置 delete-orphan 但没有删除时的警告消息，不再提到 0.6 版本，因为我们从未考虑过将其升级为异常。理想情况下，这可能更好地作为一个异常，但无论如何都不是关键。

    参考：[#2325](https://www.sqlalchemy.org/trac/ticket/2325)

+   **[orm] [错误]**

    在 get_history() 中修复了引用没有值的复合属性时的错误；为 get_history() 添加了关于否则只是一个用户自定义函数的复合的覆盖。

### 示例

+   **[示例] [错误]**

    修复了 history_meta.py 示例中的错误，其中“unique”标志未从生成要放在基础上的列的单表继承子类中删除。

### 引擎

+   **[引擎] [错误]**

    修复了当事务.rollback()在事务是两阶段或保存点事务时，如果连接无效，则会抛出错误的错误。对于普通事务，如果连接无效，rollback() 是一个空操作，所以虽然不太清楚它是否应该是一个空操作，但至少现在接口是一致的。

    参考：[#2317](https://www.sqlalchemy.org/trac/ticket/2317)

### sql

+   **[sql] [功能]**

    update() 构造现在可以在 WHERE 子句中容纳多个表格，这将生成一个被 PostgreSQL 和 MSSQL 识别的“UPDATE..FROM”构造。在 MySQL 上编译时，将生成“UPDATE t1, t2, ..”。如果在“values”参数或生成方法中使用 Column 对象作为键，则 MySQL 还可以针对多个表格在 SET 子句中渲染。

    参考：[#1944](https://www.sqlalchemy.org/trac/ticket/1944), [#2166](https://www.sqlalchemy.org/trac/ticket/2166)

+   **[sql] [功能]**

    添加了名为“python_type”的类型访问器，如果已知，则返回特定 TypeEngine 实例的基本 Python 类型对象，否则引发 NotImplementedError。

    参考：[#77](https://www.sqlalchemy.org/trac/ticket/77)

+   **[sql] [错误]**

    关于，对 select()的“from”列表的更改进行了一些调整。_froms 集合不再被记忆，因为这简化了各种用例，并消除了在表达式中使用列后，如果在表达式中使用列后再将其附加到表中，则无需“警告”的需要 - select()构造现在将始终生成正确的表达式。这里可能没有真实的性能损失；select()对象几乎总是临时制作的，并且希望优化 select()的重用的系统将使用“compiled_cache”功能。调用 select.bind 时会减少一个命中，但绝大多数用户不应该使用“bound metadata”：）。

    参考：[#2261](https://www.sqlalchemy.org/trac/ticket/2261), [#2316](https://www.sqlalchemy.org/trac/ticket/2316)

+   **[sql] [错误]**

    进一步调整了修复，以便生成方法在克隆后更好地工作（尽管这几乎是一个非使用情况）。特别是这允许 with_only_columns()更一致地行为。添加了额外的文档到 with_only_columns()以澄清预期行为，这是由于结果而改变的。

    参考：[#2261](https://www.sqlalchemy.org/trac/ticket/2261), [#2319](https://www.sqlalchemy.org/trac/ticket/2319)

### 模式

+   **[模式] [功能]**

    增加了对远程“模式”的新支持：

+   **[模式] [功能]**

    Table 上的“extend_existing”标志现在允许反射过程对已经定义的 Table 对象生效；当 autoload=True 和 extend_existing=True 都设置时，将从 Table 中反射出完整的列集，然后*覆盖*已经存在的列，而不是不发生任何活动。然而，直接在 autoload 运行中存在的列将像往常一样使用。

    参考：[#1410](https://www.sqlalchemy.org/trac/ticket/1410)

+   **[模式] [错误]**

    修复了 TypeDecorator 在使用“切换”类型的 TypeDecorator 时会返回过时值的错误，比如 CHAR/UUID 类型。

+   **[模式] [错误]**

    修复了 Inspector.get_table_names 中“order_by='foreign_key'”选项未正确实现排序的错误，替换为现有的排序算法

+   **[模式] [错误]**

    如果存在列级别的 CHECK 约束的“名称”，则现在在 CREATE TABLE 语句中使用“CONSTRAINT <name> CHECK <expression>”来呈现。

    参考：[#2305](https://www.sqlalchemy.org/trac/ticket/2305)

+   **[模式]**

    MetaData()接受“模式”和“quote_schema”参数，这些参数将应用于 Table 或 Sequence 的同名参数，这些参数保持默认值为`None`。

+   **[模式]**

    Sequence 接受“quote_schema”参数

+   **[模式]**

    对于 Table，如果 schema 参数明确为“None”，tometadata()将使用传入的 MetaData 的“模式”为新 Table。

+   **[模式]**

    添加了 CreateSchema 和 DropSchema DDL 构造 - 这些仅接受模式的字符串名称和“quote”标志。

+   **[schema]**

    当使用默认的“schema”与 MetaData 时，ForeignKey 在定位远程表时也会假定“default”模式。这允许将 MetaData 上的“schema”参数应用于任何一组否则没有“schema”的 Table 对象。

+   **[schema]**

    在方言上实现了一个“has_schema”方法，但目前只在 PostgreSQL 上有效。感谢 Manlio Perillo。

    参考：[#1679](https://www.sqlalchemy.org/trac/ticket/1679)

### postgresql

+   **[postgresql] [feature]**

    向 pg.ENUM 添加了 create_type 构造函数参数。当为 False 时，在表创建/删除事件中不会执行 CREATE/DROP 或检查类型；只有直接调用 create()/drop()方法才会执行此操作。有助于 Alembic 的“离线”脚本。

+   **[postgresql] [bug]**

    PostgreSQL 方言在创建/删除序列期间缓存了特定名称的 ENUM。这允许创建/删除序列在没有任何“checkfirst”调用的情况下工作，并且也意味着在打开“checkfirst”时只需要检查 ENUM 一次。

    参考：[#2311](https://www.sqlalchemy.org/trac/ticket/2311)

### mysql

+   **[mysql] [bug]**

    Unicode 调整允许最新的 pymysql（0.4 之后）在 Python 2 上通过 100%。

### mssql

+   **[mssql] [feature]**

    解除了对 SQL Server 的 SAVEPOINT 的限制。所有测试都通过了，但不清楚是否存在更深层次的问题。

    参考：[#822](https://www.sqlalchemy.org/trac/ticket/822)

+   **[mssql] [bug]**

    修复了在 MSSQL 上未��确实现的 with_hint()功能 - 通常用于“WITH (NOLOCK)”提示（你不应该使用的！改用快照隔离 :)）

    参考：[#2336](https://www.sqlalchemy.org/trac/ticket/2336)

+   **[mssql] [bug]**

    使用新的 pyodbc 版本检测来确定 _need_decimal_fix 选项。

    参考：[#2318](https://www.sqlalchemy.org/trac/ticket/2318)

+   **[mssql] [bug]**

    不要在 SQL Server 2000 上将“表名”转换为 NVARCHAR。然而，对于如何使 PyODBC 与 FreeTDS 0.91 完全配合，仍然大多数是未知的。

    参考：[#2343](https://www.sqlalchemy.org/trac/ticket/2343)

+   **[mssql] [bug]**

    在检索索引名称列表和这些索引中列的名称时，解码传入的值。

    参考：[#2269](https://www.sqlalchemy.org/trac/ticket/2269)

### 杂项

+   **[feature] [ext]**

    在混合文档中添加了一个“transformer”的示例 - 一个返回查询转换可调用对象的混合，结合自定义比较器。使用 Query 上的新方法 with_transformation()。这里的用例相当实验性，但只需要向 Query 添加一行代码。

+   **[bug] [pyodbc]**

    基于 pyodbc 的方言现在准确解析 pyodbc 字符串，包括诸如“py3-3.0.1-beta4”之类的珍品。

    参考：[#2318](https://www.sqlalchemy.org/trac/ticket/2318)

+   **[bug] [ext]**

    当没有“default”编译处理程序时，@compiles 装饰器会引发信息性错误消息，而不是 KeyError。

## 0.7.3

发布日期：2011 年 10 月 16 日 星期日

### 通用

+   **[general]**

    调整了“importlater”机制，该机制在内部用于解决导入循环，使得在导入 sqlalchemy 或 sqlalchemy.orm 之后完成 __import__ 的使用，从而避免在应用程序启动新线程后使用 __import__，修复了问题。也适用于 0.6.9 版本。

    参考：[#2279](https://www.sqlalchemy.org/trac/ticket/2279)

### orm

+   **[orm]**

    改进了 query.join()，使“左”侧可以更灵活地是非 ORM 可选择项，例如子查询。放置在 select_from()中的可选择项现在将被用作左侧，优先于隐式使用映射实体。如果基于外键不足而加入仍然失败，则错误消息将包含此详细信息。感谢 IRC 上的 brianrhude 提供测试用例。

    参考：[#2298](https://www.sqlalchemy.org/trac/ticket/2298)

+   **[orm]**

    添加了 after_soft_rollback() Session 事件。无论是否发生实际的 DBAPI 级别回滚，此事件都会在调用 rollback()时无条件触发。此事件专门设计用于在 Session.is_active 为 True 时允许在回滚后继续与 Session 进行操作。

    参考：[#2241](https://www.sqlalchemy.org/trac/ticket/2241)

+   **[orm]**

    在 orm.aliased()构造中添加了“adapt_on_names”布尔标志。允许 aliased()构造将 ORM 实体链接到包含聚合或其他派生形式特定属性的可选择项，前提是名称与实体映射列的名称相同。

+   **[orm]**

    向 column_property()添加了新标志 expire_on_flush=False，标记那些在刷新后否则被视为“只读”的属性，即从 SQL 表达式派生，以保留它们的值，即使父对象本身参与了更新。

+   **[orm]**

    增强了 ORM 中的仪器支持 Py3K 的新参数样式“required kw arguments”，即 fn(a, b, *, c, d)，fn(a, b, *args, c, d)。映射对象的 __init__ 方法的参数签名将被保留，包括必需的 kw 规则。

    参考：[#2237](https://www.sqlalchemy.org/trac/ticket/2237)

+   **[orm]**

    修复了工作单元中的错误，其中在高度相互链接的模式中的类之间检测“循环”不会产生确定性结果；因此，有时会错过应该被视为循环的一些节点，并在后续引起更多问题。请注意，此错误也存在于 0.6 版本中；目前尚未回溯。

    参考：[#2282](https://www.sqlalchemy.org/trac/ticket/2282)

+   **[orm]**

    修复了从 0.6 版本开始的各种与 synonym()相关的回归问题：

    > +   现在可以对同义词进行同义词处理。
    > +   
    > +   对 relationship()创建的同义词可以传递给 query.join()，传递给 query.options()的选项，按名称传递给 query.with_parent()。

+   **[orm]**

    修复了在子查询的预加载中，“mapper.order_by” 属性将被“inner”查询中忽略的错误。同时也修复了 0.6.9 版本中的此问题。

    参考：[#2287](https://www.sqlalchemy.org/trac/ticket/2287)

+   **[orm]**

    Identity map 中的 `.discard()` 方法在内部使用 `dict.pop(,None)` 而不是 “del”，以避免在非确定性的垃圾回收拆除期间出现 KeyError/警告。

    参考：[#2267](https://www.sqlalchemy.org/trac/ticket/2267)

+   **[orm]**

    在新的复合重写中修复了由于缺少导入而导致 `deferred=True` 选项失败的错误。

    参考：[#2253](https://www.sqlalchemy.org/trac/ticket/2253)

+   **[orm]**

    重新添加了 `composite()` 函数中的“comparator_factory”参数，该参数在 0.7 版本发布时被移除。

    参考：[#2248](https://www.sqlalchemy.org/trac/ticket/2248)

+   **[orm]**

    修复了在复杂的多重重叠路径方案中会发生的查询.join()中的错误，其中相同的表可能会被加入两次。感谢 Dave Vitek 在这里提供了出色的修复。

    参考：[#2247](https://www.sqlalchemy.org/trac/ticket/2247)

+   **[orm]**

    当切片的 `OFFSET` 为零时，查询将将其转换为 `None`，以避免调用不必要的 `OFFSET` 子句。

+   **[orm]**

    修复了一个极端情况，即当一个新的 mapper 上的关系在第一个 mapper 上建立反向引用时，mapper 会在更新内部状态时失败的情况。

+   **[orm]**

    修复了一个错误，即如果重新定义了 `__eq__()`，那么关系多对一的延迟加载会命中 `__eq__()` 并失败。不适用于 0.6.9。

    参考：[#2260](https://www.sqlalchemy.org/trac/ticket/2260)

+   **[orm]**

    调用 `class_mapper()` 并传入一个不是“type”的对象（即一个可能被映射的类）现在会引发一个信息性的 `ArgumentError`，而不是 `UnmappedClassError`。

    参考：[#2196](https://www.sqlalchemy.org/trac/ticket/2196)

+   **[orm]**

    新的事件钩子 `MapperEvents.after_configured()`。在 configure() 步骤完成并且 mapper 实际受到影响后调用。理论上，除非在使用现有映射之后构建新映射，否则每个应用程序都会调用此事件一次。

+   **[orm]**

    当一个打开的 Session 被垃圾回收时，其中保留的对象再次被添加到新的 Session 中时被视为分离。这是通过额外检查之前的“session_key”是否实际存在于 Sessions 池中来完成的。

    参考：[#2281](https://www.sqlalchemy.org/trac/ticket/2281)

+   **[orm]**

    新的声明性特性：

    > +   `__declare_last__()` 方法，为类方法建立一个事件监听器，在完成最终的“configure”步骤时将调用该方法。
    > +   
    > +   `__abstract__` 标志。当类上存在此标志时，该类将不被映射。
    > +   
    > +   新的辅助类 `ConcreteBase`，`AbstractConcreteBase`。允许使用声明性创建具体映射，当“configure” mapper 步骤被调用时，将自动设置“polymorphic_union”。
    > +   
    > +   映射器本身具有半私有方法，允许在配置后将“with_polymorphic”可选择项分配给映射器。

    参考：[#2239](https://www.sqlalchemy.org/trac/ticket/2239)

+   **[ORM]**

    当子类的基类使用@declared_attr 用于常规列时，Declarative 将发出警告-此属性不会传播到子类。

    参考：[#2283](https://www.sqlalchemy.org/trac/ticket/2283)

+   **[ORM]**

    用于将映射实例与其所属会话关联的整数“id”现在由序列生成函数生成，而不是 id(Session)，以消除回收的 id()值可能导致不正确结果的可能性，无需检查对象是否实际在会话中。

    参考：[#2280](https://www.sqlalchemy.org/trac/ticket/2280)

+   **[ORM]**

    行为改进：空的连接词，如 and_()和 or_()将在封闭连接词的上下文中被展平，即 and_(x, or_())将产生‘X’而不是‘X AND ()’。

    参考：[#2257](https://www.sqlalchemy.org/trac/ticket/2257)

+   **[ORM]**

    修复了有关计算 select()元素的“from”列表的错误。现在“from”计算被延迟，因此如果构造使用尚未附加到表的 Column 对象，但稍后与表关联，它将使用表作为 FROM 生成 SQL。这个更改对 FROM 列表以及“correlates”集合的计算机制产生了相当深远的影响，因为一些“子句适应”方案（这些在 ORM 中被广泛使用）依赖于“froms”集合通常在适应完成之前被缓存的事实。重做允许“froms”集合可以随时清除并重新生成。

    参考：[#2261](https://www.sqlalchemy.org/trac/ticket/2261)

+   **[ORM]**

    修复了 Select 的 with_only_columns()方法如果传递了可选择项将失败的错误。也在 0.6.9 版本中。

    参考：[#2270](https://www.sqlalchemy.org/trac/ticket/2270)

### 示例

+   **[示例]**

    调整了 dictlike-polymorphic.py 示例，应用 CAST 使其在 PG 和其他数据库上运行。也在 0.6.9 版本中。

    参考：[#2266](https://www.sqlalchemy.org/trac/ticket/2266)

### 引擎

+   **[引擎]**

    所有池类中的 recreate()方法使用 self.__class__ 来获取要生成的池类型，在子类化的情况下。请注意，通常不需要对池进行子类化。

    参考：[#2254](https://www.sqlalchemy.org/trac/ticket/2254)

+   **[引擎]**

    对多参数语句日志记录的改进，长列表的绑定参数集将被压缩，并提供有关正在进行的压缩的信息指示器。异常消息使用相同的改进格式。

    参考：[#2243](https://www.sqlalchemy.org/trac/ticket/2243)

+   **[引擎]**

    添加了可选的“sa_pool_key”参数到 pool.manage(dbapi).connect()，以便不需要序列化参数。

+   **[引擎]**

    create_engine() 支持的入口点解析现在支持在内置或入口点解析的方言上解析单独的 DBAPI 驱动程序，使用标准的“+”表示法 - 在解析为入口点之前，它会被转换为“.”。

    参考：[#2286](https://www.sqlalchemy.org/trac/ticket/2286)

+   **[engine]**

    添加了对连接中“返回 unicode 检测”步骤的异常捕获和警告，允许在 NVARCHAR 上崩溃的数据库继续初始化，假设没有实现 NVARCHAR 类型。

    参考：[#2299](https://www.sqlalchemy.org/trac/ticket/2299)

### 模式

+   **[schema]**

    修改了 Column.copy() 方法以使用 _constructor()，它默认为 self.__class__，以创建新对象。这样可以更容易地支持 Column 的子类化。

    参考：[#2284](https://www.sqlalchemy.org/trac/ticket/2284)

+   **[schema]**

    为 SchemaItem 类添加了稍微更好的 __repr__()。注意，这里的 repr 不能完全支持“repr 就是构造函数”的想法，因为模式项可能会非常深层次地嵌套/循环，有些东西可能会延迟初始化等。

    参考：[#2223](https://www.sqlalchemy.org/trac/ticket/2223)

### postgresql

+   **[postgresql]**

    为 Index() 添加了“postgresql_using”参数，用于产生 USING 子句以指定 PG 的索引实现方式。感谢 Ryan P. Kelly 提供补丁。

    参考：[#2290](https://www.sqlalchemy.org/trac/ticket/2290)

+   **[postgresql]**

    在使用 postgresql+psycopg2 方言时，create_engine() 添加了 client_encoding 参数；连接时调用 psycopg2 的 set_client_encoding() 方法并传递值。

    参考：[#1839](https://www.sqlalchemy.org/trac/ticket/1839)

+   **[postgresql]**

    修复了一个 bug，该 bug 与 PG 9 中修改索引行为影响重命名列上的主键反射有关。也出现在 0.6.9 版本中。

    参考：[#2141](https://www.sqlalchemy.org/trac/ticket/2141), [#2291](https://www.sqlalchemy.org/trac/ticket/2291)

+   **[postgresql]**

    对于 Table、Sequence 的反射函数不再区分大小写。名称只能在大小写上有所不同，并且将被正确区分。

    参考：[#2256](https://www.sqlalchemy.org/trac/ticket/2256)

+   **[postgresql]**

    使用原子计数器作为服务器端游标名称的“随机数”源；在极少数情况下报告了冲突。

+   **[postgresql]**

    缩小了在反射当前搜索路径中具有模式的外键引用表时所做的假设范围；只有在引用表的模式确实与引用表的模式匹配时，才会应用显式模式到引用表。以前假设“当前”模式与完整的搜索路径是等同的。

    参考：[#2249](https://www.sqlalchemy.org/trac/ticket/2249)

### mysql

+   **[mysql]**

    CREATE TABLE 将在 CHARSET 之后放置 COLLATE 选项，这似乎是 MySQL 关于它是否实际工作的任意规则的一部分。也出现在 0.6.9 版本中。

    参考：[#2225](https://www.sqlalchemy.org/trac/ticket/2225)

+   **[mysql]**

    向 Index 构造添加了 mysql_length 参数，指定索引的“长度”。

    参考：[#2293](https://www.sqlalchemy.org/trac/ticket/2293)

### sqlite

+   **[sqlite]**

    确保无论是否使用 C 扩展，从数据库解析的非法日期/时间/日期时间字符串都会引发相同的 ValueError。

### mssql

+   **[mssql]**

    尝试支持 FreeTDS 0.91 与 Pyodbc。这包括在检测到 FreeTDS 0.91 时将字符串绑定发送为 Python unicode 对象，并在检测到表时使用 CAST(? AS NVARCHAR)。然而，我会继续将 Pyodbc + FreeTDS 0.91 的行为描述为相当糟糕，仍然有许多查询（例如在反射中使用的查询）在 Linux 上导致核心转储，在 OSX 上根本无法使用，MemoryErrors 随处可见，对 Unicode 支持有严重问题。

    参考：[#2273](https://www.sqlalchemy.org/trac/ticket/2273)

+   **[mssql]**

    当比较标量选择与值时，= /！=的行为将不再在 0.8 版本中产生 IN / NOT IN；这种行为有点过于武断（如果要发出 IN，请使用`in_()`），现在会发出弃用警告。要立即获得 0.8 版本的行为并消除警告，可以在[`www.sqlalchemy.org/docs/07/dialects/mssql.html#scalar-select-comparisons`](https://www.sqlalchemy.org/docs/07/dialects/mssql.html#scalar-select-comparisons)给出的编译器配方中覆盖 visit_binary()的行为。

    参考：[#2277](https://www.sqlalchemy.org/trac/ticket/2277)

+   **[mssql]**

    “0”被接受为 limit()的参数，将产生“TOP 0”。

    参考：[#2222](https://www.sqlalchemy.org/trac/ticket/2222)

### oracle

+   **[oracle]**

    修复了 zxjdbc 方言的 ReturningResultProxy。从 0.6 开始的回归。

    参考：[#2272](https://www.sqlalchemy.org/trac/ticket/2272)

+   **[oracle]**

    String 类型现在在 Oracle 上生成 VARCHAR2，这是推荐的默认 VARCHAR。在 Oracle 方言中还添加了显式的 VARCHAR2 和 NVARCHAR2。使用 NVARCHAR 仍然生成“NVARCHAR2” - 在 Oracle 上没有“NVARCHAR” - 这仍然是“大写类型始终给出确切内容”的政策的轻微破坏。VARCHAR 仍然生成“VARCHAR”，遵循该政策。如果 Oracle 曾经定义“VARCHAR”为他们声称的不同内容（在我看���永远不会发生），该类型将可用。

    参考：[#2252](https://www.sqlalchemy.org/trac/ticket/2252)

### 杂项

+   **[types]**

    超出基本 Float 类型的“精度”和“asdecimal”之外的额外关键字参数将被忽略；这里添加了一个弃用警告和额外的文档，相关于

    参考：[#2258](https://www.sqlalchemy.org/trac/ticket/2258)

+   **[ext]**

    SQLSoup 将不包含在 SQLAlchemy 的 0.8 版本中；虽然有用，但我们希望保持 SQLAlchemy 本身专注于一个 ORM 使用范例。SQLSoup 很快将被第三方项目取代。

    参考：[#2262](https://www.sqlalchemy.org/trac/ticket/2262)

+   **[ext]**

    向 AssociationProxy 添加了 local_attr、remote_attr、attr 访问器，提供对类级别的代理属性的快速访问。

    参考：[#2236](https://www.sqlalchemy.org/trac/ticket/2236)

+   **[ext]**

    更改了关联代理字典上的 update() 方法，以使用鸭子类型方法，即检查“keys”，以区分 update({}) 和 update((a, b))。先前，传递具有元组作为键的字典会被错误解释为序列。

    参考：[#2275](https://www.sqlalchemy.org/trac/ticket/2275)

### 一般

+   **[general]**

    调整了“importlater”机制，该机制在内部用于解决导入循环，使得在导入 sqlalchemy 或 sqlalchemy.orm 后完成对 __import__ 的使用，从而避免在应用程序启动新线程后使用任何 __import__。也适用于 0.6.9 版本。

    参考：[#2279](https://www.sqlalchemy.org/trac/ticket/2279)

### orm

+   **[orm]**

    改进了 query.join()，使得“左”侧可以更灵活地是非 ORM 可选择的，比如子查询。放置在 select_from() 中的可选择项现在将用作左侧，优先于对映射实体的隐式使用。如果基于外键缺失而连接仍然失败，则错误消息将包含此详细信息。感谢 IRC 上的 brianrhude 提供的测试用例。

    参考：[#2298](https://www.sqlalchemy.org/trac/ticket/2298)

+   **[orm]**

    添加了 after_soft_rollback() 会话事件。无论是否发生了实际的 DBAPI 级别的回滚，此事件都会无条件触发 rollback() 被调用。此事件专门设计用于在 rollback() 后 Session.is_active 为 True 时允许会话操作继续进行。

    参考：[#2241](https://www.sqlalchemy.org/trac/ticket/2241)

+   **[orm]**

    向 orm.aliased() 构造添加了“adapt_on_names”布尔标志。允许 aliased() 构造将 ORM 实体链接到包含聚合或特定属性的其他派生形式的可选择项，前提是名称与映射列的实体相同。

+   **[orm]**

    向 column_property() 添加了新标志 expire_on_flush=False，将那些否则被认为是“只读”的属性标记为，在刷新后保留它们的值，包括父对象本身参与更新的情况。

+   **[orm]**

    增强了 ORM 中的仪器设备以支持 Py3K 的新参数样式“required kw arguments”，即 fn(a, b, *, c, d)，fn(a, b, *args, c, d)。映射对象的 __init__ 方法的参数签名将被保留，包括必需的 kw 规则。

    参考：[#2237](https://www.sqlalchemy.org/trac/ticket/2237)

+   **[orm]**

    修复了工作单元中的错误，其中在高度相互链接的模式中检测类之间的“循环”不会产生确定性结果；因此，有时会错过应该被视为循环的一些节点，并在后续引发更多问题。请注意，此错误也存在于 0.6 版本中；目前没有回溯。

    参考：[#2282](https://www.sqlalchemy.org/trac/ticket/2282)

+   **[orm]**

    修复了从 0.6 版本中引入的各种 synonym()-相关回归问题：

    > +   现在对一个同义词进行同义词处理是有效的。
    > +   
    > +   与 relationship()相关的同义词可以传递给 query.join()、发送到 query.options()的选项，并按名称传递给 query.with_parent()。

+   **[orm]**

    修复了一个 bug，其中 mapper.order_by 属性在子查询贪婪加载中的“内部”查询中将被忽略。同时也在 0.6.9 版本中修复。

    参考：[#2287](https://www.sqlalchemy.org/trac/ticket/2287)

+   **[orm]**

    Identity map .discard()在内部使用 dict.pop(,None)而不是“del”，以避免在非确定性 gc 拆除期间出现 KeyError/警告。

    参考：[#2267](https://www.sqlalchemy.org/trac/ticket/2267)

+   **[orm]**

    修复了新的 composite 重写中由于缺少导入而导致 deferred=True 选项失败的回归问题。

    参考：[#2253](https://www.sqlalchemy.org/trac/ticket/2253)

+   **[orm]**

    重新引入了 composite()的“comparator_factory”参数，在 0.7 发布时删除。

    参考：[#2248](https://www.sqlalchemy.org/trac/ticket/2248)

+   **[orm]**

    修复了在复杂的多重重叠路径场景中会发生的 query.join()中的 bug，在此场景中，同一表可能会被连接两次。非常感谢 Dave Vitek 在这里的出色修复。

    参考：[#2247](https://www.sqlalchemy.org/trac/ticket/2247)

+   **[orm]**

    当查询（Query）将 OFFSET 设置为零时，将其转换为 None，以避免不必要的 OFFSET 子句被调用。

+   **[orm]**

    修复了一个边缘情况，在新映射器上的关系（relationship）建立了第一个映射器上的反向引用时，mapper 将无法完全更新内部状态。

+   **[orm]**

    修复了一个 bug，其中如果重新定义了 __eq__()，则关系（relationship）多对一懒加载会触发 __eq__()并失败。不适用于 0.6.9 版本。

    参考：[#2260](https://www.sqlalchemy.org/trac/ticket/2260)

+   **[orm]**

    调用 class_mapper()并传入一个不是“类型”的对象（即可能被映射的类）现在会引发一个具有信息性的 ArgumentError，而不是 UnmappedClassError。

    参考：[#2196](https://www.sqlalchemy.org/trac/ticket/2196)

+   **[orm]**

    新的事件钩子，MapperEvents.after_configured()。在配置（configure()）步骤完成并实际受到影响的映射器（mappers）后调用。理论上，此事件每个应用程序调用一次，除非在已使用现有映射器之后构造新映射。

+   **[orm]**

    当一个打开的 Session 被垃圾回收时，其中仍然存在的对象在被添加到新的 Session 时被认为是分离的。这是通过额外检查之前的“session_key”是否实际上存在于 Sessions 池中完成的。

    参考：[#2281](https://www.sqlalchemy.org/trac/ticket/2281)

+   **[orm]**

    新的声明性功能：

    > +   __declare_last__()方法，为类方法建立一个事件监听器，该监听器将在映射器（mappers）完成最终的“configure”步骤时调用。
    > +   
    > +   __abstract__ 标志。当类上存在此标志时，该类将不会被映射。
    > +   
    > +   新的辅助类 ConcreteBase、AbstractConcreteBase。允许使用声明性进行具体映射，当“configure”映射器步骤被调用时，自动设置“polymorphic_union”。
    > +   
    > +   映射器本身具有半私有方法，允许在配置后将“with_polymorphic”可选择的分配给映射器。

    参考：[#2239](https://www.sqlalchemy.org/trac/ticket/2239)

+   **[orm]**

    当子类的基类使用 @declared_attr 用于常规列时，声明性将发出警告 - 此属性不会传播到子类。

    参考：[#2283](https://www.sqlalchemy.org/trac/ticket/2283)

+   **[orm]**

    用于将映射实例与其所属会话关联的整数“id”现在由序列生成函数生成，而不是 id(Session)，以消除回收的 id() 值可能导致不正确结果的可能性，无需检查对象实际上是���在会话中。

    参考：[#2280](https://www.sqlalchemy.org/trac/ticket/2280)

+   **[orm]**

    行为改进：空连接词，如 and_() 和 or_()，将在包含连接词的上下文中被展开，即 and_(x, or_()) 将产生‘X’而不是‘X AND ()’。

    参考：[#2257](https://www.sqlalchemy.org/trac/ticket/2257)

+   **[orm]**

    修复了有关计算“from”列表的 bug，用于 select() 元素。现在，“from”计算被延迟，因此如果构造使用尚未附加到表格但稍后与表格关联的 Column 对象，则会生成使用表格作为 FROM 的 SQL。这个更改对 FROM 列表以及“correlates”集合的计算机制产生了相当深远的影响，因为一些“子句适应”方案（这些方案在 ORM 中被大量使用）依赖于“froms”集合通常在适应完成之前被缓存的事实。重新设计使得“froms”集合可以随时清除并重新生成。

    参考：[#2261](https://www.sqlalchemy.org/trac/ticket/2261)

+   **[orm]**

    修复了一个 bug，即如果传递了可选择项，则 Select 的 with_only_columns() 方法将失败。也适用于 0.6.9 版本。

    参考：[#2270](https://www.sqlalchemy.org/trac/ticket/2270)

### 示例

+   **[examples]**

    调整 dictlike-polymorphic.py 示例以应用 CAST，使其在 PG 和其他数据库上运行。也适用于 0.6.9 版本。

    参考：[#2266](https://www.sqlalchemy.org/trac/ticket/2266)

### engine

+   **[engine]**

    所有池类中的 recreate() 方法使用 self.__class__ 来获取要生成的池类型，在子类化的情况下。请注意，通常不需要对池进行子类化。

    参考：[#2254](https://www.sqlalchemy.org/trac/ticket/2254)

+   **[engine]**

    改进多参数语句记录，长列表的绑定参数集将被压缩，并附带有指示压缩正在进行的信息。异常消息使用相同的改进格式。

    参考：[#2243](https://www.sqlalchemy.org/trac/ticket/2243)

+   **[引擎]**

    添加了可选的“sa_pool_key”参数到 pool.manage(dbapi).connect()，这样就不需要对参数进行序列化。

+   **[引擎]**

    create_engine()现在支持解析单独的 DBAPI 驱动程序的入口点解析，使用标准的‘+’符号 - 在解析为入口点之前会将其转换为‘.’。

    参考：[#2286](https://www.sqlalchemy.org/trac/ticket/2286)

+   **[引擎]**

    在连接中添加了一个异常捕获+警告，用于“返回 unicode 检测”步骤，允许在 NVARCHAR 上崩溃的数据库继续初始化，假设没有实现 NVARCHAR 类型。

    参考：[#2299](https://www.sqlalchemy.org/trac/ticket/2299)

### 模式

+   **[模式]**

    修改了 Column.copy()以使用 _constructor()，默认为 self.__class__，以创建新对象。这样可以更容易地支持 Column 的子类化。

    参考：[#2284](https://www.sqlalchemy.org/trac/ticket/2284)

+   **[模式]**

    对 SchemaItem 类添加了稍微更好的 __repr__()。请注意，这里的 repr 不能完全支持“repr 即构造函数”的想法，因为模式项可以非常深层嵌套/循环，某些内容的初始化较晚等。

    参考：[#2223](https://www.sqlalchemy.org/trac/ticket/2223)

### postgresql

+   **[postgresql]**

    向 Index()添加了“postgresql_using”参数，生成 USING 子句以指定 PG 的索引实现方式。感谢 Ryan P. Kelly 的补丁。

    参考：[#2290](https://www.sqlalchemy.org/trac/ticket/2290)

+   **[postgresql]**

    当使用 postgresql+psycopg2 方言时，向 create_engine()添加 client_encoding 参数；在连接时调用 psycopg2 的 set_client_encoding()方法并传递值。

    参考：[#1839](https://www.sqlalchemy.org/trac/ticket/1839)

+   **[postgresql]**

    修复了一个与 PG 9 中相同修改的索引行为影响重命名列上的主键反射的错误。也在 0.6.9 中修复。

    参考：[#2141](https://www.sqlalchemy.org/trac/ticket/2141), [#2291](https://www.sqlalchemy.org/trac/ticket/2291)

+   **[postgresql]**

    Table、Sequence 的反射函数不再区分大小写。名称只能在大小写上有所不同，并且将被正确区分。

    参考：[#2256](https://www.sqlalchemy.org/trac/ticket/2256)

+   **[postgresql]**

    使用原子计数器作为服务器端游标名称的“随机数”来源；在极少数情况下已报告冲突。

+   **[postgresql]**

    缩小了在反射具有当前搜索路径中模式的外键引用表时所做的假设；只有当显式模式与引用表的模式实际匹配时，才会将显式模式应用于引用表。先前假定“当前”模式与完整搜索路径是同义词。

    参考：[#2249](https://www.sqlalchemy.org/trac/ticket/2249)

### mysql

+   **[mysql]**

    CREATE TABLE 将在 CHARSET 后放置 COLLATE 选项，这似乎是 MySQL 关于其是否实际工作的任意规则的一部分。也适用于 0.6.9 版本。

    参考：[#2225](https://www.sqlalchemy.org/trac/ticket/2225)

+   **[mysql]**

    向 Index 构造添加了 mysql_length 参数，指定索引的“length”。

    参考：[#2293](https://www.sqlalchemy.org/trac/ticket/2293)

### sqlite

+   **[sqlite]**

    确保无论是否使用 C 扩展，从数据库解析的非法日期/时间/日期时间字符串都会引发相同的 ValueError。

### mssql

+   **[mssql]**

    更改以尝试支持 FreeTDS 0.91 与 Pyodbc。当检测到 FreeTDS 0.91 时，字符串绑定将作为 Python unicode 对象发送，并且在检测到表时使用 CAST(? AS NVARCHAR)。然而，我会继续将 Pyodbc + FreeTDS 0.91 的行为描述为相当糟糕，仍然有许多查询（例如在反射中使用的查询）在 Linux 上导致核心转储，在 OSX 上根本无法使用，MemoryErrors 丰富，而且对 Unicode 的支持完全破碎。

    参考：[#2273](https://www.sqlalchemy.org/trac/ticket/2273)

+   **[mssql]**

    当将标量选择与值进行比较时，= /！= 的行为将不再在 0.8 版本中产生 IN / NOT IN；这种行为有点过于武断（如果要发出 IN，请使用 `in_()`），现在会发出弃用警告。要立即获得 0.8 版本的行为并消除警告，可以在 [`www.sqlalchemy.org/docs/07/dialects/mssql.html#scalar-select-comparisons`](https://www.sqlalchemy.org/docs/07/dialects/mssql.html#scalar-select-comparisons) 给出的编译器配方中覆盖 visit_binary() 的行为。

    参考：[#2277](https://www.sqlalchemy.org/trac/ticket/2277)

+   **[mssql]**

    “0” 被接受为 limit() 的参数，这将产生“TOP 0”。

    参考：[#2222](https://www.sqlalchemy.org/trac/ticket/2222)

### oracle

+   **[oracle]**

    修复了 zxjdbc 方言的 ReturningResultProxy。从 0.6 版本开始的退化。

    参考：[#2272](https://www.sqlalchemy.org/trac/ticket/2272)

+   **[oracle]**

    String 类型现在在 Oracle 上生成 VARCHAR2，这是推荐的默认 VARCHAR。在 Oracle 方言中还添加了显式的 VARCHAR2 和 NVARCHAR2。使用 NVARCHAR 仍然生成“NVARCHAR2” - 在 Oracle 上没有“NVARCHAR” - 这仍然是“大写类型总是给出确切内容”的政策的轻微破坏。VARCHAR 仍然生成“VARCHAR”，遵循该政策。如果 Oracle 曾经定义“VARCHAR”为他们声称的不同内容（在我看来永远不会发生），该类型将可用。

    参考：[#2252](https://www.sqlalchemy.org/trac/ticket/2252)

### 杂项

+   **[types]**

    在基本 Float 类型之外的额外关键字参数“precision”和“asdecimal”将被忽略；在此处添加了弃用警告和额外文档，相关于

    参考：[#2258](https://www.sqlalchemy.org/trac/ticket/2258)

+   **[ext]**

    SQLSoup 将不包含在 SQLAlchemy 的 0.8 版本中；虽然有用，但我们希望将 SQLAlchemy 本身集中在一个 ORM 使用范例上。SQLSoup 希望很快会被第三方项目取代。

    参考：[#2262](https://www.sqlalchemy.org/trac/ticket/2262)

+   **[扩展]**

    为 AssociationProxy 添加了 local_attr、remote_attr、attr 访问器，提供了在类级别快速访问代理属性的功能。

    参考：[#2236](https://www.sqlalchemy.org/trac/ticket/2236)

+   **[扩展]**

    更改了关联代理字典上的 update()方法，采用了鸭子类型的方法，即检查“键”，以区分 update({})和 update((a, b))。以前，传递具有元组作为键的字典会被误解为序列。

    参考：[#2275](https://www.sqlalchemy.org/trac/ticket/2275)

## 0.7.2

发布日期：2011 年 7 月 31 日 星期日

### ORM

+   **[ORM]**

    功能增强：现在，连接和子查询加载将遍历已经存在的相关对象和集合，以查找未填充的属性，贯穿定义的急加载范围，以便通过映射或查询选项指定的急加载无条件地发生在整个深度上，填充尚未填充的内容。以前，如果已经存在相关对象或集合，此遍历将停止，导致不一致的行为（尽管会节省已加载图的加载/循环）。对于子查询加载，这意味着子查询加载发出的额外 SELECT 语句将无条件调用，无论现有图形的多少（因此有争议）。当查询是属性启动的延迟加载的结果时，以前的“停止”行为仍然有效，否则当重复遇到相同的相关对象时，“N+1”风格的集合迭代可能变得不必要昂贵。还有一个尚未公开的生成 Query 方法 _with_invoke_all_eagers()，选择旧/新行为

    参考：[#2213](https://www.sqlalchemy.org/trac/ticket/2213)

+   **[ORM]**

    在 ORM 中重新设计了“替换遍历”，因为它会将可选择的内容更改为针对事物的别名（即子句适配），包括修复了针对连接表结构的多层嵌套 any()/has()构造的问题。

    参考：[#2195](https://www.sqlalchemy.org/trac/ticket/2195)

+   **[ORM]**

    修复了一个错误，即在 relationship()上从一个连接到自身的 joined-inh 结构上使用 query.join() + aliased=True，并且在子表上具有连接条件时，会不适当地将主实体转换为连接实体。也在 0.6.9 中。

    参考：[#2234](https://www.sqlalchemy.org/trac/ticket/2234)

+   **[ORM]**

    修复了从 0.6 中的回归，其中 Session.add()针对包含集合中的 None 的对象会引发内部异常。将此恢复为 0.6 的行为，即接受 None，但显然不会持久化任何内容。理想情况下，存在 None 的集合或在 append()上的集合至少应发出警告，这将在 0.8 中考虑。 

    参考：[#2205](https://www.sqlalchemy.org/trac/ticket/2205)

+   **[orm]**

    在无法定位行的对象上加载延迟属性会引发 ObjectDeletedError，而不是稍后失败；改进了 ObjectDeletedError 中的消息，包括除了简单的“删除”之外的其他条件。

    参考：[#2191](https://www.sqlalchemy.org/trac/ticket/2191)

+   **[orm]**

    修复了从 0.6 中的回归，其中对一些基于 relationship()的属性执行 get history 操作时，当 lazyload 会发出时会失败；在某些条件下，这可能在 flush()中触发。感谢提交了这个伟大测试的用户。

    参考：[#2224](https://www.sqlalchemy.org/trac/ticket/2224)

+   **[orm]**

    修复了仅在 Python 3 中明显的 bug，即在 flush 期间对持久性+挂起对象进行排序会产生非法比较，如果持久性对象的主键不是单个整数。也在 0.6.9 中。

    参考：[#2228](https://www.sqlalchemy.org/trac/ticket/2228)

+   **[orm]**

    修复了一个 bug，即 query.join()使用的源子句在针对将多个实体组合在一起的列表达式时会不一致。也在 0.6.9 中。

    参考：[#2197](https://www.sqlalchemy.org/trac/ticket/2197)

+   **[orm]**

    修复了一个 bug，即如果映射类重新定义了 __hash__()或 __eq__()为非标准内容，这是一个受支持的用例，因为 SQLA 不应该查询这些内容，那么如果该类是“复合”（即非单实体）结果集的一部分，这些方法将被查询。也在 0.6.9 中。

    参考：[#2215](https://www.sqlalchemy.org/trac/ticket/2215)

+   **[orm]**

    向 Mapper 添加了公共属性“.validators”，这是所有已使用@validates 装饰器装饰的属性的不可变字典视图。感谢 Stefano Fontanelli

    参考：[#2240](https://www.sqlalchemy.org/trac/ticket/2240)

+   **[orm]**

    修复了一个微妙的 bug，导致 SQL 在以下情况下崩溃：column_property()针对子查询 + joinedload + LIMIT +按列属性排序。也在 0.6.9 中。

    参考：[#2188](https://www.sqlalchemy.org/trac/ticket/2188)

+   **[orm]**

    由 with_parent 生成的连接条件以及针对父级使用“dynamic”关系时将生成唯一的 bindparams，而不是错误地重复相同的 bindparam。也在 0.6.9 中。

    参考：[#2207](https://www.sqlalchemy.org/trac/ticket/2207)

+   **[orm]**

    将“仅列”检查添加到 mapper.polymorphic_on 中，与接收 relationship.order_by、foreign_keys、remote_side 等用户参数时使用的检查相同。

+   **[orm]**

    修复了比较列表达式和 Query()的 bug，该比较不会调用 as_scalar()来在基础 SELECT 语句上生成标量子查询，方式类似于如果在 Query().subquery()上调用它时发生的情况。

    参考：[#2190](https://www.sqlalchemy.org/trac/ticket/2190)

+   **[orm]**

    修复了声明性 bug，当一个类继承自同名的超类时，由于在 _decl_class_registry 中不必要地查找名称，会导致失败。

    参考：[#2194](https://www.sqlalchemy.org/trac/ticket/2194)

+   **[orm]**

    修复了 Query 中的“无语句条件”断言，该断言在调用 from_statement()之后调用生成方法时会尝试引发异常。也适用于 0.6.9。

    参考：[#2199](https://www.sqlalchemy.org/trac/ticket/2199)

### 示例

+   **[示例]**

    修复了示例/版本控制测试运行程序，不再依赖 SQLAlchemy 测试库，必须从示例/版本控制中运行 nosetests 以解决 setup.cfg 导致的问题。

+   **[示例]**

    调整示例/版本控制以在多层继承情况下选择正确的外键。

+   **[示例]**

    修复了属性分片示例，以正确检查 0.7 风格中的绑定参数可调用性。

### engine

+   **[引擎]**

    由 Connection.begin()提供的上下文管理器在提交失败时会执行 rollback()，不仅在发生异常时。

+   **[引擎]**

    在 Python 2.6 及以上版本中使用 urllib.parse_qsl()，不再发出有关 cgi.parse_qsl()的弃用警告。

    参考：[#1682](https://www.sqlalchemy.org/trac/ticket/1682)

+   **[引擎]**

    添加了 mixin 类 sqlalchemy.ext.DontWrapMixin。在语句执行的上下文中发生时，此类型的用户定义异常永远不会在 StatementException 中包装。

+   **[引擎]**

    StatementException 包装将在消息中显示原始异常类。

+   **[引擎]**

    连接失败会引发 dbapi.Error 的错误将转发到 dialect.is_disconnect()，如果方言知道这是一种可能的“可重试”条件，则设置“connection_invalidated”标志。目前只有 Oracle ORA-01033 实现。

    参考：[#2201](https://www.sqlalchemy.org/trac/ticket/2201)

### sql

+   **[sql]**

    修复了两个关于可选择的列对应的微妙错误，一个是相同的标记子查询重复，另一个是当标签已被“分组”并丢失时。影响。

    参考：[#2188](https://www.sqlalchemy.org/trac/ticket/2188)

### 模式

+   **[模式]**

    新特性：对所有类型增加了`with_variant()`方法。产生一个 Variant()实例，这是一个特殊的 TypeDecorator，根据当前使用的方言选择不同类型的用法。

    参考：[#2187](https://www.sqlalchemy.org/trac/ticket/2187)

+   **[模式]**

    当 ForeignKeyConstraint 引用父级中不存在的列名时，添加了一个信息性错误消息。也适用于 0.6.9。

+   **[模式]**

    修复了旧的 append_ddl_listener()函数适应错误，通过到表事件的意外**kw。表不得到 kws，0.6 中的 MetaData 事件将得到“tables=somecollection”，这种行为得到保留。

    引用：[#2206](https://www.sqlalchemy.org/trac/ticket/2206)

+   **[schema]**

    修复了在表上检测“autoincrement”时的错误，如果类型没有“亲和性”值，特别是当在网站上使用 TypeEngine 作为“impl”的 UUID 示例时会发生这种情况。

+   **[schema]**

    为 TypeEngine 对象添加了改进的 repr()，它只会显示位置参数或偏离默认值的 kwargs。

    引用：[#2209](https://www.sqlalchemy.org/trac/ticket/2209)

### postgresql

+   **[postgresql]**

    添加了新的“postgresql_ops”参数到 Index，允许为索引列指定 PostgreSQL 操作符类。由 Filip Zyzniewski 提供。

    引用：[#2198](https://www.sqlalchemy.org/trac/ticket/2198)

### mysql

+   **[mysql]**

    修复了 OurSQL 方言，在 XA 命令中使用 ansi-neutral 引号符“’”而不是‘”’的错误。也在 0.6.9 中。

    引用：[#2186](https://www.sqlalchemy.org/trac/ticket/2186)

### sqlite

+   **[sqlite]**

    SQLite 方言不再从反映的默认值中去除引号，从而允许往返 CREATE TABLE 工作。这与其他方言一致，它们也保留默认值的精确形式。

    引用：[#2189](https://www.sqlalchemy.org/trac/ticket/2189)

### mssql

+   **[mssql]**

    调整了 pyodbc 方言，如果检测到“Easysoft”unix 驱动程序，则绑定值将作为字节而不是 unicode 传递。这与 FreeTDS 发生的行为相同。在某些情况下，Easysoft 似乎会导致 segfault，如果传递 Python unicode。

### oracle

+   **[oracle]**

    添加了 ORA-00028 到断开连接代码中，使用 cx_oracle _Error.code 来获取代码，也在 0.6.9 中。

    引用：[#2200](https://www.sqlalchemy.org/trac/ticket/2200)

+   **[oracle]**

    添加了 ORA-01033 到断开连接代码中，它可以在连接事件中捕获。

    引用：[#2201](https://www.sqlalchemy.org/trac/ticket/2201)

+   **[oracle]**

    修复了 oracle.RAW 类型，它未生成正确的 DDL。也在 0.6.9 中。

    引用：[#2220](https://www.sqlalchemy.org/trac/ticket/2220)

+   **[oracle]**

    添加了 CURRENT 到保留字列表中。也在 0.6.9 中。

    引用：[#2212](https://www.sqlalchemy.org/trac/ticket/2212)

+   **[oracle]**

    修复了可变扩展中的错误，其中如果一个映射中两次使用相同的类型，则第一个之后的属性不会被检测。

+   **[oracle]**

    修复了可变扩展中的错误，其中如果设置为 None 或非对应类型，则会引发错误。现在接受 None，将 None 分配给所有属性，非法值引发 ValueError。

### orm

+   **[orm]**

    功能增强：现在，对已经存在的相关对象和集合进行 joined 和子查询加载，以搜索未填充的属性，这是在定义了急加载的范围内进行的，因此通过映射或查询选项指定的急加载将无条件地对整个深度进行加载，填充尚未填充的内容。之前，如果已经存在相关对象或集合，则此遍历将停止，导致不一致的行为（尽管对于已加载的图表会节省加载/循环）。对于子查询加载，这意味着子查询加载发出的额外 SELECT 语句将无条件调用，无论现有图形的多少（因此引发了争议）。当查询是由属性启动的懒加载的结果时，先前的“停止”行为仍然有效，否则当重复遇到相同的相关对象时，“N+1”样式的集合迭代可能会变得不必要地昂贵。还有一个尚未公开的生成性 Query 方法 _with_invoke_all_eagers()，它选择旧/新行为。

    参考：[#2213](https://www.sqlalchemy.org/trac/ticket/2213)

+   **[orm]**

    对 ORM 中的“替换遍历”进行了重新设计，因为它会修改为针对事物的别名的可选择性（即子句适配），包括对针对加入表结构的多重嵌套 any()/has() 构造的修复。

    参考：[#2195](https://www.sqlalchemy.org/trac/ticket/2195)

+   **[orm]**

    修复了一个错误，即在 relationship() 中从一个关联到自身的 joined-inh 结构的子表上执行 query.join() + aliased=True 时，会不适当地将主实体转换为已加入的实体。也适用于 0.6.9 版本。

    参考：[#2234](https://www.sqlalchemy.org/trac/ticket/2234)

+   **[orm]**

    修复了 0.6 版本中的一个退化，即针对一个包含空集合的对象进行 Session.add() 操作会引发内部异常的问题。将此恢复为 0.6 版本的行为，即接受空集合，但显然不会持久化任何内容。理想情况下，包含 None 的集合或在 append() 时应至少发出警告，这将在 0.8 版本中考虑。

    参考：[#2205](https://www.sqlalchemy.org/trac/ticket/2205)

+   **[orm]**

    在找不到行的对象上加载延迟属性时，会引发 ObjectDeletedError 而不是稍后失败；改进了 ObjectDeletedError 中的消息，以包括除了简单的“删除”之外的其他条件。

    参考：[#2191](https://www.sqlalchemy.org/trac/ticket/2191)

+   **[orm]**

    修复了 0.6 版本中的一个退化，在某些基于 relationship() 的属性上进行 get history 操作时，当懒加载会发出时会失败；这可能会在某些条件下在 flush() 中触发。感谢提交了此问题的出色测试的用户。

    参考：[#2224](https://www.sqlalchemy.org/trac/ticket/2224)

+   **[orm]**

    修复了仅在 Python 3 中明显的 bug，在刷新期间对持久性+挂起对象进行排序会产生非法比较，如果持久对象的主键不是单个整数。同时也在 0.6.9 中修复。

    参考：[#2228](https://www.sqlalchemy.org/trac/ticket/2228)

+   **[orm]**

    修复了一个 bug，即 query.join()使用的源子句在针对将多个实体组合在一起的列表达式时会不一致。同时也在 0.6.9 中修复。

    参考：[#2197](https://www.sqlalchemy.org/trac/ticket/2197)

+   **[orm]**

    修复了一个 bug，即如果映射类重新定义了 __hash__()或 __eq__()为非标准内容，这是一个受支持的用例，因为 SQLA 不应该查询这些方法，如果类是复合（即非单实体）结果集的一部分，这些方法将被查询。同时也在 0.6.9 中修复。

    参考：[#2215](https://www.sqlalchemy.org/trac/ticket/2215)

+   **[orm]**

    向 Mapper 添加了公共属性“.validators”，这是所有已使用@validates 装饰器装饰的属性的不可变字典视图。由 Stefano Fontanelli 提供。

    参考：[#2240](https://www.sqlalchemy.org/trac/ticket/2240)

+   **[orm]**

    修复了一个微妙的 bug，导致如果发生了：对子查询的列属性(column_property()) + joinedload + LIMIT + 按列属性排序，SQL 会出现问题。同时也在 0.6.9 中修复。

    参考：[#2188](https://www.sqlalchemy.org/trac/ticket/2188)

+   **[orm]**

    由 with_parent 生成的连接条件以及对父项使用“dynamic”关系时会生成唯一的 bindparams，而不是错误地重复相同的 bindparam。同时也在 0.6.9 中修复。

    参考：[#2207](https://www.sqlalchemy.org/trac/ticket/2207)

+   **[orm]**

    向 mapper.polymorphic_on 添加了与 relationship.order_by、foreign_keys、remote_side 等接收用户参数时使用的相同“仅列”检查。

+   **[orm]**

    修复了一个 bug，使得对列表达式与 Query()进行比较时不会调用底层 SELECT 语句上的 as_scalar()方法来生成标量子查询，这种情况发生在如果你在 Query().subquery()上调用它时发生。

    参考：[#2190](https://www.sqlalchemy.org/trac/ticket/2190)

+   **[orm]**

    修复了一个声明性 bug，在此 bug 中，从具有相同名称的超类继承的类将由于不必要地在 _decl_class_registry 中查找名称而失败。

    参考：[#2194](https://www.sqlalchemy.org/trac/ticket/2194)

+   **[orm]**

    修复了在调用 from_statement()之后调用生成方法会尝试引发“无语句条件”断言的 Query 中的 bug。同时也在 0.6.9 中修复。

    参考：[#2199](https://www.sqlalchemy.org/trac/ticket/2199)

### 示例

+   **[示例]**

    修复了示例/版本化测试运行程序，不再依赖于 SQLAlchemy 测试库，nosetests 必须在示例/版本化目录中运行以避免 setup.cfg 破坏它。

+   **[示例]**

    对示例/版本化进行微调以在多级继承情况下选择正确的外键。

+   **[示例]**

    修复了属性 shard 示例，以正确检查 0.7 风格中的绑定参数可调用。

### engine

+   **[engine]**

    Connection.begin()提供的上下文管理器如果提交失败，将执行 rollback()，不仅在发生异常时执行。

+   **[engine]**

    在 Python 2.6 及以上版本中使用 urllib.parse_qsl()，不会有关于 cgi.parse_qsl()的弃用警告。

    参考：[#1682](https://www.sqlalchemy.org/trac/ticket/1682)

+   **[engine]**

    添加了 mixin 类 sqlalchemy.ext.DontWrapMixin。当用户定义的此类型的异常在语句执行上下文中发生时，它们永远不会被包装在 StatementException 中。

+   **[engine]**

    StatementException 包装将在消息中显示原始异常类。

+   **[engine]**

    连接失败时引发 dbapi.Error 的错误将转发到 dialect.is_disconnect()，如果方言知道这是一个可能的“可重试”条件，则设置“connection_invalidated”标志。目前仅实现了 Oracle ORA-01033。

    参考：[#2201](https://www.sqlalchemy.org/trac/ticket/2201)

### sql

+   **[sql]**

    修复了两个涉及可选择的列对应的微妙 bug，一个是重复使用相同标记的子查询，另一个是标记已被“分组”且丢失自身。影响。

    参考：[#2188](https://www.sqlalchemy.org/trac/ticket/2188)

### schema

+   **[schema]**

    新功能：在所有类型上添加了 with_variant()方法。生成 Variant()的实例，这是一个特殊的 TypeDecorator，根据使用的方言选择不同类型的用法。

    参考：[#2187](https://www.sqlalchemy.org/trac/ticket/2187)

+   **[schema]**

    当 ForeignKeyConstraint 引用父级中未找到的列名时，添加了一个信息性错误消息。也适用于 0.6.9 版本。

+   **[schema]**

    修复了旧的 append_ddl_listener()函数适配的 bug，该函数将意外地将**kw 传递给 Table 事件。Table 不接收 kw 参数，0.6 版本中的 MetaData 事件将接收“tables=somecollection”，此行为得以保留。

    参考：[#2206](https://www.sqlalchemy.org/trac/ticket/2206)

+   **[schema]**

    修复了 Table 上“autoincrement”检测的 bug，如果类型没有“affinity”值，则会失败，特别是在使用 TypeEngine 作为“impl”的站点上使用 UUID 示例时会发生这种情况。

+   **[schema]**

    为 TypeEngine 对象添加了改进的 repr()，只显示构造函数参数，这些参数是位置参数或与默认值不同的 kwargs。

    参考：[#2209](https://www.sqlalchemy.org/trac/ticket/2209)

### postgresql

+   **[postgresql]**

    为 Index 添加了新的“postgresql_ops”参数，允许为索引列指定 PostgreSQL 操作符类。感谢 Filip Zyzniewski。

    参考：[#2198](https://www.sqlalchemy.org/trac/ticket/2198)

### mysql

+   **[mysql]**

    修复了 OurSQL 方言，使用 ansi-neutral 引号符“’”代替‘”’来执行 XA 命令。也适用于 0.6.9 版本。

    参考：[#2186](https://www.sqlalchemy.org/trac/ticket/2186)

### sqlite

+   **[sqlite]**

    SQLite 方言不���剥离反射默认值的引号，允许往返 CREATE TABLE 正常工作。这与其他方言一致，它们也保持默认值的确切形式。

    参考：[#2189](https://www.sqlalchemy.org/trac/ticket/2189)

### mssql

+   **[mssql]**

    调整了 pyodbc 方言，如果检测到“Easysoft” unix 驱动程序，则绑定值将作为字节而不是 Unicode 传递。这与 FreeTDS 发生的行为相同。在某些情况下，Easysoft 似乎会在传递 Python Unicode 时导致段错误。

### oracle

+   **[oracle]**

    将 ORA-00028 添加到断开连接代码中，使用 cx_oracle _Error.code 来获取代码。也适用于 0.6.9 版本。

    参考：[#2200](https://www.sqlalchemy.org/trac/ticket/2200)

+   **[oracle]**

    将 ORA-01033 添加到断开连接代码中，可以在连接事件中捕获。

    参考：[#2201](https://www.sqlalchemy.org/trac/ticket/2201)

+   **[oracle]**

    修复了未生成正确 DDL 的 oracle.RAW 类型。也适用于 0.6.9 版本。

    参考：[#2220](https://www.sqlalchemy.org/trac/ticket/2220)

+   **[oracle]**

    将 CURRENT 添加到保留字列表中。也适用于 0.6.9 版本。

    参考：[#2212](https://www.sqlalchemy.org/trac/ticket/2212)

+   **[oracle]**

    修复了可变扩展中的错误，如果在一个映射中两次使用相同类型，则第一个之后的属性将不会被激活。

+   **[oracle]**

    修复了可变扩展中的错误，如果设置为 None 或非对应类型，则会引发错误。现在接受 None，将 None 分配给所有属性，非法值引发 ValueError。

## 0.7.1

发布日期：2011 年 6 月 5 日星期日

### 通用

+   **[general]**

    为 Python bug 7511 添加了一个解决方法，在 Windows 64 位 + VC express 上，C 扩展构建失败不会引发适当的异常。

    参考：[#2184](https://www.sqlalchemy.org/trac/ticket/2184)

### orm

+   **[orm]**

    “delete-orphan”级联现在允许在自引用关系上使用 - 自 SQLA 0.7 以来不再在 ORM 级别强制执行“父级没有子级”的检查；此检查留给外键的可空性。相关内容

    参考：[#1912](https://www.sqlalchemy.org/trac/ticket/1912)

+   **[orm]**

    修复了新的“mutable”扩展，以正确传播事件到子类；也不为子类创建多个事件侦听器。

    参考：[#2180](https://www.sqlalchemy.org/trac/ticket/2180)

+   **[orm]**

    修改了在刷新时未检测到“identity”键时出现的消息文本，包括常见原因，即列未正确设置以检测自增。也适用于 0.6.8 版本。

    参考：[#2170](https://www.sqlalchemy.org/trac/ticket/2170)

+   **[orm]**

    修复了事务级别的“deleted”集合不会清除已删除状态的错误，如果它们后来变为瞬态，则会引发错误。也适用于 0.6.8 版本。

    参考：[#2182](https://www.sqlalchemy.org/trac/ticket/2182)

### engine

+   **[engine]**

    废弃 Connection/Engine 上从未被广泛知晓且多余的基于模式/SQL 的方法：reflecttable()、create()、drop()、text()、engine.func

+   **[engine]**

    调整了 RowProxy 结果行的 __contains__()方法，使其在内部不会生成异常；无论列构造是否可以强制转换为字符串，NoSuchColumnError()也会生成其消息。也在 0.6.8 中。

    参考：[#2178](https://www.sqlalchemy.org/trac/ticket/2178)

### sql

+   **[sql]**

    修复了 metadata.reflect(bind)会关闭作为绑定参数传递的 Connection 的 bug。从 0.6 开始的回归。

+   **[sql]**

    简化了 Select 确定其‘.c’集合中内容的过程。行为完全相同，只是传递给 select([])的原始 ClauseList()（这本来就不是一个文档化的情况）现在将被扩展为其各个列元素，而不是被忽略。

### postgresql

+   **[postgresql]**

    关于数字数组、MATCH 运算符的一些单元测试修复。修复了潜在的浮点不准确问题，并且目前某些 MATCH 运算符的测试仅在 EN 定向的区域设置下执行。也在 0.6.8 中。

    参考：[#2175](https://www.sqlalchemy.org/trac/ticket/2175)

### mysql

+   **[mysql]**

    单元测试在安装在 Windows 上的 MySQL 上通过率达到 100%。

+   **[mysql]**

    移除了在反射具有混合大小写名称的 MySQL 上的表时会失败的“调整大小写”步骤。经过一些对 Windows MySQL 服务器的实验后，确定这一步骤并没有真正帮助解决问题；MySQL 在非 Windows 平台上也不会返回带有正确大小写的 FK 名称，移除这一步骤至少使反射的行为更像在其他操作系统上的行为。这里考虑过发出警告，但很难确定在什么条件下可以发出这样的警告，所以暂时搁置了这个问题 - 而是添加了一些文档。

    参考：[#2181](https://www.sqlalchemy.org/trac/ticket/2181)

+   **[mysql]**

    如果使用 MySQLdb 且 DBAPI 不提供 constants.CLIENT 模块，则 supports_sane_rowcount 将设置为 False。

### sqlite

+   **[sqlite]**

    当调用“PRAGMA read_uncommitted”以确定连接时的当前隔离模式并默认为 SERIALIZABLE 时，接受来自 cursor.fetchone()的 None；这是为了支持 SQLite 版本 3.3.0 之前不具备此功能的情况。

    参考：[#2173](https://www.sqlalchemy.org/trac/ticket/2173)

### 一般

+   **[general]**

    为 Python bug 7511 添加了一个解决方法，其中 C 扩展构建失败不会在 Windows 64 位+ VC express 上引发适当的异常。

    参考：[#2184](https://www.sqlalchemy.org/trac/ticket/2184)

### orm

+   **[orm]**

    ”delete-orphan”级联现在允许在自引用关系上使用 - 这是因为 SQLA 0.7 不再在 ORM 级别强制执行“父项没有子项”的检查；此检查留给外键的可空性。相关的

    参考：[#1912](https://www.sqlalchemy.org/trac/ticket/1912)

+   **[orm]**

    修复了新的“mutable”扩展正确传播事件到子类的问题；也不会为子类创建多个事件监听器。

    参考：[#2180](https://www.sqlalchemy.org/trac/ticket/2180)

+   **[orm]**

    修改了在刷新时未检测到“identity”键时出现的消息文本，以包括列未正确设置以正确检测自增的常见原因；也适用于 0.6.8 版本。

    参考：[#2170](https://www.sqlalchemy.org/trac/ticket/2170)

+   **[orm]**

    修复了事务级别的“已删除”集合不会清除已删除状态的 bug，如果后来变为瞬态，则会引发错误。也适用于 0.6.8 版本。

    参考：[#2182](https://www.sqlalchemy.org/trac/ticket/2182)

### 引擎

+   **[engine]**

    在 Connection/Engine 上弃用了从未被广泛知晓且多余的基于模式/SQL 的方法：reflecttable()、create()、drop()、text()、engine.func

+   **[engine]**

    调整了 RowProxy 结果行的 __contains__()方法，使其在内部不生成异常；无论列构造是否可以强制转换为字符串，NoSuchColumnError()也将生成其消息。也适用于 0.6.8 版本。

    参考：[#2178](https://www.sqlalchemy.org/trac/ticket/2178)

### sql

+   **[sql]**

    修复了 metadata.reflect(bind)会关闭传递为绑定参数的 Connection 的 bug。从 0.6 版本开始的回归。

+   **[sql]**

    简化了 Select 确定其‘.c’集合中内容的过程。行为完全相同，只是传递给 select([])的原始 ClauseList()（这本来就不是一个文档化的情况）现在将被扩展为其各个列元素，而不是被忽略。

### postgresql

+   **[postgresql]**

    修复了关于数字数组、MATCH 运算符的一些单元测试问题。修复了潜在的浮点不精确性问题，并且目前某些 MATCH 运算符的测试仅在 EN 定向的区域设置下执行。也适用于 0.6.8 版本。

    参考：[#2175](https://www.sqlalchemy.org/trac/ticket/2175)

### mysql

+   **[mysql]**

    单元测试在安装在 Windows 上的 MySQL 上 100%通过。

+   **[mysql]**

    移除了在 Windows 上使用混合大小写名称的 MySQL 反射表时会失败的“调整大小写”步骤。经过一些对 Windows MySQL 服务器的实验后，确定这一步骤并没有真正帮助解决问题；MySQL 在非 Windows 平台上也不会返回正确大小写的 FK 名称，移除这一步骤至少使反射表的行为更像在其他操作系统上的行为。考虑过在此处发出警告，但很难确定在什么条件下可以发出这样的警告，因此暂时搁置了这个想法 - 只是添加了一些文档。

    参考：[#2181](https://www.sqlalchemy.org/trac/ticket/2181)

+   **[mysql]**

    如果使用 MySQLdb 并且 DBAPI 不提供 constants.CLIENT 模块，则 supports_sane_rowcount 将设置为 False。

### sqlite

+   **[sqlite]**

    在调用“PRAGMA read_uncommitted”以确定连接时的当前隔离模式并默认为 SERIALIZABLE 时，接受来自 cursor.fetchone()的 None；这是为了支持 SQLite 版本 3.3.0 之前没有此功能的情况。

    参考：[#2173](https://www.sqlalchemy.org/trac/ticket/2173)

## 0.7.0

发布日期：2011 年 5 月 20 日星期五

### orm

+   **[orm]**

    修复了在 0.7b4 中引入的回归，即 query.options(someoption(“nonexistent name”))将无法引发错误。还为尝试基于基于列的元素构建选项的情况添加了额外的错误捕获，进一步修正了一些错误消息

    参考：[#2069](https://www.sqlalchemy.org/trac/ticket/2069)

+   **[orm]**

    query.count()发出“count(*)”而不是“count(1)”。

    参考：[#2162](https://www.sqlalchemy.org/trac/ticket/2162)

+   **[orm]**

    当使用 from_self()，union()或其他“从自身选择”的操作时，对 Query 子句适应进行微调，以便对添加到 filter()，order_by()等中的普通 SQL 表达式元素进行适应，这些元素存在于嵌套的“从自身”查询中，*将*以与 ORM 表达式元素相同的方式进行适应，因为否则这些元素不容易访问。

    参考：[#2155](https://www.sqlalchemy.org/trac/ticket/2155)

+   **[orm]**

    修复了一个 bug，即“自引用”关系的确定会失败，没有解决方案，涉及到自身的 joined-inh 子类，或者与没有在 join 条件中的子子类的子类相关联。也适用于 0.6.8 版本。

    参考：[#2149](https://www.sqlalchemy.org/trac/ticket/2149)

+   **[orm]**

    mapper()将在确定父类和子类之间的继承条件时忽略未配置的与不相关表的外键，但对于关于继承表的未解析列和表名，将像往常一样引发异常。这是对先前应用于声明性的行为的增强泛化。0.6.8 具有更保守的版本，不会从根本上改变确定连接条件的方式。

    参考：[#2153](https://www.sqlalchemy.org/trac/ticket/2153)

+   **[orm]**

    在给定实体不是单个完整类实体或映射器（即列）时调用 query.get()是一个错误。这是 0.6.8 中的一个弃用警告。

    参考：[#2144](https://www.sqlalchemy.org/trac/ticket/2144)

+   **[orm]**

    修复了在某些情况下可能发生的潜在 KeyError，部分

    参考：[#2148](https://www.sqlalchemy.org/trac/ticket/2148)

+   **[orm]**

    添加了 Query.with_session()方法，将 Query 切换到使用不同的会话。

+   **[orm]**

    水平分片查询应该根据每个连接使用执行选项

    参考：[#2131](https://www.sqlalchemy.org/trac/ticket/2131)

+   **[orm]**

    非主 mapper 将继承主 mapper 的 _identity_class。这样，对于通常处于继承映射中的类建立的非主 mapper，将产生与主 mapper 兼容的 identity-map 结果（也在 0.6.8 中）

    参考：[#2151](https://www.sqlalchemy.org/trac/ticket/2151)

+   **[orm]**

    修复了“无法为目标列‘q’执行 syncrule；mapper ‘X’ 不映射此列”的错误消息，以引用正确的 mapper。也在 0.6.8 中。

    参考：[#2163](https://www.sqlalchemy.org/trac/ticket/2163)

+   **[orm]**

    polymorphic_union() 得到一个 “cast_nulls” 选项，当它渲染标记为 NULL 的列时禁用 CAST 的使用。

    参考：[#1502](https://www.sqlalchemy.org/trac/ticket/1502)

+   **[orm]**

    polymorphic_union() 将列按照它们在原始表中的顺序渲染，根据它们在出现的多态联合列表中的第一个表/可选择的表来排序。（除非你传递了一个 OrderedDict，否则这本身就是一个无序映射）。

+   **[orm]**

    修复了一个错误，即如果将 mapper 映射到一个匿名别名，则在使用日志记录时会失败，因为别名中的 % 符号未经转义。也在 0.6.8 中。

    参考：[#2171](https://www.sqlalchemy.org/trac/ticket/2171)

### 示例

+   **[示例]**

    删除了古老的“多态关联”示例，并用使用声明性 mixin、“通用关联”更新的示例集进行替换。每个示例都提供了一种替代的表布局。

### sql

+   **[sql]**

    修复了一个错误，即将 select() 的一个标签嵌套到另一个标签中会产生错误的导出列。除其他事项外，这会破坏对另一个 column_property() 的 ORM column_property() 映射。也在 0.6.8 中

    参考：[#2167](https://www.sqlalchemy.org/trac/ticket/2167)

+   **[sql]**

    更改了确定连接条件的处理方式，使得外键错误仅在两个给定的表之间考虑。也就是说，t1.join(t2) 将报告涉及‘t1’或‘t2’的 FK 错误，但任何涉及‘t3’的内容都将被跳过。这会影响 join()，以及 ORM 关系和继承条件逻辑。

+   **[sql]**

    在执行过程中对错误处理进行了一些改进，以确保在出现非常不寻常的 DBAPI 错误时自动关闭的连接确实关闭。

+   **[sql]**

    metadata.reflect() 和 reflection.Inspector() 对内部获取的连接有一些依赖关系，这些依赖关系已经被修复。

+   **[sql]**

    添加了一个显式检查，当列 .name 被赋予空字符串时

    参考：[#2140](https://www.sqlalchemy.org/trac/ticket/2140)

+   **[sql]**

    修复了一个错误，即如果将 FetchedValue 传递给列 server_onupdate，则其父“列”将不会被赋值，为所有列默认赋值模式添加了测试覆盖。也在 0.6.8 中。

    参考：[#2147](https://www.sqlalchemy.org/trac/ticket/2147)

### postgresql

+   **[postgresql]**

    修复了在 psycopg2 方言中解析 psycopg2_version 的问题。

+   **[postgresql]**

    修复了影响 PG 9 的 bug，其中反射索引会失败，如果针对更改名称的列。也在 0.6.8 中。

    参考：[#2141](https://www.sqlalchemy.org/trac/ticket/2141)

### mssql

+   **[mssql]**

    修复了 MSSQL 方言中的 bug，其中应用于模式限定表的别名会泄漏到封闭的 select 语句中。也在 0.6.8 中。

    参考：[#2169](https://www.sqlalchemy.org/trac/ticket/2169)

### 杂项

+   **[no_tags]**

    本节记录了从 0.7b4 到 0.7.0 的更改。有关 SQLAlchemy 0.7 中的新功能概述，请参见[`docs.sqlalchemy.org/en/latest/changelog/migration_07.html`](https://docs.sqlalchemy.org/en/latest/changelog/migration_07.html)

+   **[documentation]**

    从 ext.mutable 文档中删除了对“collections.MutableMapping” abc 的使用，因为它被错误地使用，并且在任何情况下都使示例更难理解。

    参考：[#2152](https://www.sqlalchemy.org/trac/ticket/2152)

+   **[ext]**

    修复了 sqlalchemy.ext.mutable 扩展中的 bug，其中 None 没有得到适当处理，替换事件也没有得到适当处理。

    参考：[#2143](https://www.sqlalchemy.org/trac/ticket/2143)

### orm

+   **[orm]**

    修复了在 0.7b4 中引入的回归（！），在那里查询选项（someoption(“不存在的名称”))将无法引发错误。还为尝试基于基于列的元素构建选项的情况添加了额外的错误捕获，进一步修复了一些定制的错误消息

    参考：[#2069](https://www.sqlalchemy.org/trac/ticket/2069)

+   **[orm]**

    query.count() 发出“count(*)”而不是“count(1)”。

    参考：[#2162](https://www.sqlalchemy.org/trac/ticket/2162)

+   **[orm]**

    在 from_self()，union()或其他“从自己选择”的操作时，对 Query 子句适应进行微调，以便在嵌套的“from myself”查询中添加到 filter()，order_by()等中的纯 SQL 表达式元素将以与 ORM 表达式元素相同的方式进行适应，因为否则这些元素不容易访问。

    参考：[#2155](https://www.sqlalchemy.org/trac/ticket/2155)

+   **[orm]**

    修复了一个 bug，当判断“自引用”关系失败时，没有解决方法，导致与自身相关的 joined-inh 子类或与没有在子子类中的列在连接条件中的子类相关的 joined-inh 子类。也在 0.6.8 中。

    参考：[#2149](https://www.sqlalchemy.org/trac/ticket/2149)

+   **[orm]**

    mapper()将在确定父类和子类之间的继承条件时忽略与无关表的非配置外键，但对于未解析的列和表名，关于继承表的继承条件将像往常一样引发异常。这是对以前已应用于声明的行为的增强泛化。0.6.8 有一个更保守的版本，不会根本改变如何确定连接条件。

    参考：[#2153](https://www.sqlalchemy.org/trac/ticket/2153)

+   **[orm]**

    在给定实体不是单一的、完整的类实体或映射器（即一个列）时调用 query.get()是一个错误。这在 0.6.8 中是一个废弃的警告。

    参考：[#2144](https://www.sqlalchemy.org/trac/ticket/2144)

+   **[orm]**

    修复了在某些情况下可能会发生的潜在的 KeyError，该错误可能会在标识映射中发生，这是的一部分

    参考：[#2148](https://www.sqlalchemy.org/trac/ticket/2148)

+   **[orm]**

    添加了 Query.with_session()方法，将 Query 切换到使用不同的会话。

+   **[orm]**

    水平分片查询应该使用每个连接的执行选项，如

    参考：[#2131](https://www.sqlalchemy.org/trac/ticket/2131)

+   **[orm]**

    非主要映射器将继承主映射器的 _identity_class。这样，对于通常处于继承映射中的类进行非主要建立的情况，将产生与主映射器的标识映射兼容的结果（也在 0.6.8 中）。

    参考：[#2151](https://www.sqlalchemy.org/trac/ticket/2151)

+   **[orm]**

    修复了“无法为目标列‘q’执行 syncrule；映射‘X’未映射此列”的错误消息，以引用正确的映射器。也在 0.6.8 中。

    参考：[#2163](https://www.sqlalchemy.org/trac/ticket/2163)

+   **[orm]**

    polymorphic_union()获得了一个“cast_nulls”选项，当它呈现标记的 NULL 列时，禁用了 CAST 的使用。

    参考：[#1502](https://www.sqlalchemy.org/trac/ticket/1502)

+   **[orm]**

    polymorphic_union()以它们在原始表中的顺序呈现列，根据它们在多态联合列表中首次出现的第一个表/可选择表。 （除非您传递了一个 OrderedDict，否则它本身是一个无序映射）。

+   **[orm]**

    修复了一个 bug，即如果使用了日志记录，则映射到匿名别名的映射器将失败，因为别名名称中存在未转义的%符号。同样也在 0.6.8 中。

    参考：[#2171](https://www.sqlalchemy.org/trac/ticket/2171)

### 例子

+   **[examples]**

    删除了古老的“多态关联”示例，并用使用声明性混合项、“通用关联”替换了一组更新后的示例。每个示例呈现了一种替代的表布局。

### sql

+   **[sql]**

    修复了一个 bug，即将 select()的标签嵌套在另一个标签中时，将产生不正确的导出列。除其他外，这会破坏针对另一个列属性()映射的 ORM 列属性()映射。也在 0.6.8 中

    参考：[#2167](https://www.sqlalchemy.org/trac/ticket/2167)

+   **[sql]**

    更改了决定连接条件的处理方式，使外键错误仅在两个给定的表之间考虑。也就是说，t1.join(t2)将报告涉及‘t1’或‘t2’的 FK 错误，但任何涉及‘t3’的都将被跳过。这影响 join()，以及 ORM 关系和继承条件逻辑。

+   **[sql]**

    在 execute 过程内部的错误处理中进行了一些改进，以确保在发生非常不寻常的 DBAPI 错误时真正关闭自动关闭的连接。

+   **[sql]**

    metadata.reflect() 和 reflection.Inspector() 在调用 GC 关闭内部获取的连接时存在一些依赖关系，已修复此问题。

+   **[sql]**

    添加了对 Column .name 被分配为空字符串时的显式检查

    参考：[#2140](https://www.sqlalchemy.org/trac/ticket/2140)

+   **[sql]**

    修复了一个错误，即如果将 FetchedValue 传递给列 server_onupdate，则其父“列”将不会被分配，为所有列默认分配模式增加了测试覆盖。同时也在 0.6.8 版中进行了修复。

    参考：[#2147](https://www.sqlalchemy.org/trac/ticket/2147)

### postgresql

+   **[postgresql]**

    修复了 psycopg2 方言中 psycopg2_version 解析的错误。

+   **[postgresql]**

    修复了影响 PG 9 的错误，即索引反射将失败，如果针对更改了名称的列进行。同时也在 0.6.8 版中进行了修复。

    参考：[#2141](https://www.sqlalchemy.org/trac/ticket/2141)

### mssql

+   **[mssql]**

    修复了 MSSQL 方言中的错误，其中应用于带模式限定的表的别名会泄漏到封闭的 select 语句中。同时也在 0.6.8 版中进行了修复。

    参考：[#2169](https://www.sqlalchemy.org/trac/ticket/2169)

### 杂项

+   **[无标签]**

    此部分记录了从 0.7b4 到 0.7.0 的更改。有关 SQLAlchemy 0.7 中的新功能概述，请参阅 [`docs.sqlalchemy.org/en/latest/changelog/migration_07.html`](https://docs.sqlalchemy.org/en/latest/changelog/migration_07.html)

+   **[文档]**

    从 ext.mutable 文档中删除了对“collections.MutableMapping” abc 的使用，因为它使用不正确，而且在任何情况下都使示例更难理解。

    参考：[#2152](https://www.sqlalchemy.org/trac/ticket/2152)

+   **[ext]**

    在 sqlalchemy.ext.mutable 扩展中修复了一些错误，其中 None 没有得到适当处理，替换事件也没有得到适当处理。

    参考：[#2143](https://www.sqlalchemy.org/trac/ticket/2143)

## 0.7.0b4

发布日期：2011 年 4 月 17 日

### 通用

+   **[通用]**

    更改了 CHANGES，即本文件的格式。格式更改已应用于 0.7 版本。

+   **[通用]**

    现在，“-declarative” 更改将直接列在“-orm”部分下面，因为这些更改是密切相关的。

+   **[通用]**

    0.5 系列的更改已经移至文件 CHANGES_PRE_06，该文件替代了 CHANGES_PRE_05。

+   **[通用]**

    0.6.7 及其后续版本在 0.6 系列中现在只在 0.6 分支中的 CHANGES 文件中列出。在 0.7 CHANGES 文件中（即本文件），所有 0.6 的更改都以行内方式列在它们被应用的 0.7 部分中（因为所有 0.6 更改也都在 0.7 中）。这里适用于 0.6 版本的更改被记录，以及如果存在任何实现/行为上的差异则予以注明。

### orm

+   **[orm]**

    当调用 query.update()、query.delete() 时，对“evaluate”和“fetch”评估进行了一些修复。在所有情况下，记录的检索都在自动提交之后进行，并且在发出更新/删除之前进行，以防止存在未提交的数据以及在评估期间失败的过期对象。

    参考：[#2122](https://www.sqlalchemy.org/trac/ticket/2122)

+   **[orm]**

    重新修订了当尝试刷新一个不是对超类型进行多态的子类时引发的异常。

    参考：[#2063](https://www.sqlalchemy.org/trac/ticket/2063)

+   **[orm]**

    当查询选项无法找到目标实体时，仍需进行更多的措辞调整。解释路径必须来自一个根实体。

+   **[orm]**

    关于反向引用的状态处理进行了一些修复，通常在 autoflush=False 时，反向引用的集合不会正确处理没有净变化的添加/删除。感谢 Richard Murri 提供的测试用例和补丁。（也在 0.6.7 中）。

    参考：[#2123](https://www.sqlalchemy.org/trac/ticket/2123)

+   **[orm]**

    在 UOW 内添加了检查，以检测在主键值中包含 NULL 时被要求进行 UPDATE 或 DELETE 的异常情况。

    参考：[#2127](https://www.sqlalchemy.org/trac/ticket/2127)

+   **[orm]**

    对属性历史进行了一些改进。更多的变化可能会在 0.8 版本中进行，但目前历史已经被修改，使得标量历史不会在非存在值时填充为 None。这样稍微更好地区分了 None 设置和实际没有变化的能力，也会产生影响。

    参考：[#2127](https://www.sqlalchemy.org/trac/ticket/2127)

+   **[orm]**

    如果使用 from_self()，则“having��子句将从内部复制到外部查询；特别是这将破坏 0.7 风格的 count()查询。（也在 0.6.7 中）

    参考：[#2130](https://www.sqlalchemy.org/trac/ticket/2130)

+   **[orm]**

    Query.execution_options()方法现在将这些选项传递给 Connection 而不是 SELECT 语句，以便可以使用所有可用选项，包括隔离级别和编译缓存。

    参考：[#2131](https://www.sqlalchemy.org/trac/ticket/2131)

### engine

+   **[engine]**

    C 扩展现在在 CPython 2.x 上默认启用，如果编译失败，则回退到纯 Python。

    参考：[#2129](https://www.sqlalchemy.org/trac/ticket/2129)

### sql

+   **[sql]**

    “compiled_cache”执行选项现在在传递给 SELECT 语句而不是 Connection 时会引发错误。之前它完全被忽略。我们可能会考虑在某个时候使这个选项在每个语句级别上工作。

    参考：[#2131](https://www.sqlalchemy.org/trac/ticket/2131)

+   **[sql]**

    恢复了基本 TypeEngine 类上的“catchall”构造函数，并附带弃用警告。这样，像 Integer(11)这样的代码仍然可以成功。

+   **[sql]**

    修复了回归，即从反序列化中返回的 MetaData()没有跟踪现在跟踪的新内容，即 Sequence 对象的集合，模式名称列表。

    参考：[#2104](https://www.sqlalchemy.org/trac/ticket/2104)

+   **[sql]**

    select()中的 limit/offset 关键字以及传递给 select.limit()/offset()的值将被强制转换为整数。（也在 0.6.7 中）

    参考资料：[#2116](https://www.sqlalchemy.org/trac/ticket/2116)

+   **[sql]**

    修复了一个 bug，即从 over() 子句中收集的“from”子句将是一个 itertools.chain() 而不是一个列表，当与其他子句组合时会导致“can only concatenate list” TypeError。

+   **[sql]**

    修复了在 over() 子句中不正确使用“,”的问题，该问题被放置在“partition”和“order by”子句之间。

    参考资料：[#2134](https://www.sqlalchemy.org/trac/ticket/2134)

+   **[sql]**

    PrimaryKeyConstraint 的 before/after attach 事件现在可以正常使用，为所有约束类型添加了 before/after 事件的测试。

    参考资料：[#2105](https://www.sqlalchemy.org/trac/ticket/2105)

+   **[sql]**

    添加了显式的 true()/false() 构造到表达式库 - 强制规则将“False”/“True”拦截到这些构造中。在 0.6 版本中，这些构造通常直接转换为字符串，但在 0.7 版本中不再接受。

    参考资料：[#2117](https://www.sqlalchemy.org/trac/ticket/2117)

### schema

+   **[schema]**

    Table 上的 ‘useexisting’ 标志已被一对新标志 ‘keep_existing’ 和 ‘extend_existing’ 所取代。‘extend_existing’ 等效于 ‘useexisting’ - 返回现有的 Table，并添加附加的构造元素。使用 ‘keep_existing’，返回现有的 Table，但不添加附加的构造元素 - 这些元素仅在新创建 Table 时应用。

    参考资料：[#2109](https://www.sqlalchemy.org/trac/ticket/2109)

### postgresql

+   **[postgresql]**

    支持 Python 3 的 Psycopg2 现已支持。

+   **[postgresql]**

    修复了使用 pg8000 时对精度数字的支持。

    参考资料：[#2132](https://www.sqlalchemy.org/trac/ticket/2132)

### sqlite

+   **[sqlite]**

    修复了如果作为“REFERENCES <tablename>”创建的外键的反射没有列名，则会失败的 bug。 （也在 0.6.7 版本中）

    参考资料：[#2115](https://www.sqlalchemy.org/trac/ticket/2115)

### oracle

+   **[oracle]**

    当与 cx_oracle 通信时，对于需要为列本身或名称生成的绑定参数，例如具有特殊字符、下划线、非 ASCII 字符的名称，现在正确地将绑定参数键翻译为列名。 （也在 0.6.7 版本中）

    参考资料：[#2100](https://www.sqlalchemy.org/trac/ticket/2100)

+   **[oracle]**

    Oracle 方言添加了 use_binds_for_limits=False create_engine() 标志，将在行内呈现 LIMIT/OFFSET 值，而不是作为绑定，据报道修改了 Oracle 使用的执行计划。 （也在 0.6.7 版本中）

    参考资料：[#2116](https://www.sqlalchemy.org/trac/ticket/2116)

### 杂项

+   **[types]**

    REAL 已添加到核心类型中。由 PostgreSQL、SQL Server、MySQL、SQLite 支持。请注意，SQL Server 和 MySQL 版本添加了额外的参数，仍然可以从这些方言中获取。

    参考资料：[#2081](https://www.sqlalchemy.org/trac/ticket/2081)

+   **[types]**

    添加了 @event.listens_for() 装饰器，给定目标 + 事件名称，将装饰函数应用为监听器。

    参考资料：[#2106](https://www.sqlalchemy.org/trac/ticket/2106)

+   **[pool]**

    AssertionPool 现在存储了指示当前检出连接获取位置的回溯信息；在第二次并发检出时，此回溯信息将在引发的断言中报告；感谢 Gunnlaugur Briem

    参考：[#2103](https://www.sqlalchemy.org/trac/ticket/2103)

+   **[池]**

    “pool.manage”功能不再使用 pickle 来为每个池的参数进行哈希。

+   **[文档]**

    记录了 SQLite 的 DATE/TIME/DATETIME 类型。（也在 0.6.7 中）

    参考：[#2029](https://www.sqlalchemy.org/trac/ticket/2029)

+   **[文档]**

    修复可变扩展文档，显示正确的类型关联方法。

    参考：[#2118](https://www.sqlalchemy.org/trac/ticket/2118)

### 通用

+   **[通用]**

    对 CHANGES 文件的格式进行更改。这些格式更改已应用于 0.7 版本发布。

+   **[通用]**

    “-declarative”更改现在将直接列在“-orm”部分下面，因为它们密切相关。

+   **[通用]**

    0.5 系列的更改已移至 CHANGES_PRE_06 文件，取代了 CHANGES_PRE_05 文件。

+   **[通用]**

    0.6.7 及其后续版本的更改现在仅在 0.6 分支内的 CHANGES 文件中列出。在 0.7 CHANGES 文件（即本文件）中，所有 0.6 更改都内联列在它们也应用的 0.7 部分中（因为所有 0.6 更改也在 0.7 中）。这里适用于 0.6 版本的更改被记录，如果存在实现/行为上的任何差异也会被注明。

### orm

+   **[orm]**

    当调用 query.update()，query.delete()时，对“evaluate”和“fetch”评估进行一些修复。在所有情况下，记录的检索都是在自动刷新之后进行的，并且在发出 update/delete 之前进行，以防止未刷新的数据以及在评估过程中失败的过期对象。

    参考：[#2122](https://www.sqlalchemy.org/trac/ticket/2122)

+   **[orm]**

    重新措辞了当尝试刷新不是对超类型进行多态处理的子类时引发的异常。

    参考：[#2063](https://www.sqlalchemy.org/trac/ticket/2063)

+   **[orm]**

    当查询选项无法找到目标实体时，进一步调整措辞。解释路径必须从根实体之一开始。

+   **[orm]**

    关于反向引用(backrefs)的状态处理进行了一些修复，通常在 autoflush=False 时，当反向引用集合未正确处理没有净变化的 add/remove 时。感谢 Richard Murri 提供的测试用例+补丁。（也在 0.6.7 中）。

    参考：[#2123](https://www.sqlalchemy.org/trac/ticket/2123)

+   **[orm]**

    在 UOW 内添加检查，以检测在主键值中包含 NULL 的情况下被要求进行 UPDATE 或 DELETE 的异常条件。

    参考：[#2127](https://www.sqlalchemy.org/trac/ticket/2127)

+   **[orm]**

    对属性历史进行了一些细化。更多的更改可能在 0.8 中进行，但目前历史已被修改，使得标量历史不再具有为不存在的值填充 None 的“副作用”。这样可以稍微更好地区分 None 集和没有实际更改的能力，也会受到影响。

    参考：[#2127](https://www.sqlalchemy.org/trac/ticket/2127)

+   **[ORM]**

    如果使用 from_self()，则“having”子句将从内部复制到外部查询；特别是这将破坏 0.7 风格的 count()查询。（也适用于 0.6.7）

    参考：[#2130](https://www.sqlalchemy.org/trac/ticket/2130)

+   **[ORM]**

    Query.execution_options()方法现在将这些选项传递给 Connection 而不是 SELECT 语句，因此可以使用所有可用选项，包括隔离级别和编译缓存。

    参考：[#2131](https://www.sqlalchemy.org/trac/ticket/2131)

### 引擎

+   **[引擎]**

    C 扩展现在在 CPython 2.x 上默认启用，如果编译失败，则回退到纯 Python。

    参考：[#2129](https://www.sqlalchemy.org/trac/ticket/2129)

### SQL

+   **[SQL]**

    “compiled_cache”执行选项现在在传递给 SELECT 语句而不是 Connection 时会引发错误。以前它完全被忽略。我们可能会考虑在某个时候使此选项在每个语句级别上工作。

    参考：[#2131](https://www.sqlalchemy.org/trac/ticket/2131)

+   **[SQL]**

    恢复了基本 TypeEngine 类上的“catchall”构造函数，并附带弃用警告。这样，像 Integer(11)这样的代码仍然可以成功。

+   **[SQL]**

    修复了从反序列化中返回的 MetaData()未能跟踪现在跟踪的新内容的回归，即 Sequence 对象的集合，模式名称列表。

    参考：[#2104](https://www.sqlalchemy.org/trac/ticket/2104)

+   **[SQL]**

    传递给 select()的 limit/offset 关键字以及传递给 select.limit()/offset()的值将被强制转换为整数。（也适��于 0.6.7）

    参考：[#2116](https://www.sqlalchemy.org/trac/ticket/2116)

+   **[SQL]**

    修复了一个 bug，即从 over()子句中收集的“from”子句将是一个 itertools.chain()而不是一个列表，导致与其他子句组合时出现“can only concatenate list” TypeError。

+   **[SQL]**

    修复了在 over()子句中不正确使用“,”被放置在“partition”和“order by”子句之间的问题。

    参考：[#2134](https://www.sqlalchemy.org/trac/ticket/2134)

+   **[SQL]**

    在 PrimaryKeyConstraint 的前/后附加事件现在起作用，对所有约束类型的前/后事件进行了测试。

    参考：[#2105](https://www.sqlalchemy.org/trac/ticket/2105)

+   **[SQL]**

    在表达式库中添加了显式的 true()/false()构造 - 强制规则将“False”/“True”拦截到这些构造中。在 0.6 中，这些构造通常直接转换为字符串，而在 0.7 中不再接受。

    参考：[#2117](https://www.sqlalchemy.org/trac/ticket/2117)

### 模式

+   **[模式]**

    Table 上的‘useexisting’标志已被新的一对标志‘keep_existing’和‘extend_existing’取代。‘extend_existing’等同于‘useexisting’ - 返回现有的 Table，并添加额外的构造元素。使用‘keep_existing’时，返回现有的 Table，但不添加额外的构造元素 - 这些元素仅在新创建 Table 时应用。

    参考：[#2109](https://www.sqlalchemy.org/trac/ticket/2109)

### postgresql

+   **[postgresql]**

    现在支持 Python 3 的 Psycopg2。

+   **[postgresql]**

    修复了在使用 pg8000 时支持精度数字的问题。

    参考：[#2132](https://www.sqlalchemy.org/trac/ticket/2132)

### sqlite

+   **[sqlite]**

    修复了一个 bug，即反射创建的外键为“REFERENCES <tablename>”而没有列名时会失败。（也适用于 0.6.7 版本）

    参考：[#2115](https://www.sqlalchemy.org/trac/ticket/2115)

### oracle

+   **[oracle]**

    使用需要为列本身或名称生成的绑定参数引号的列名，例如具有特殊字符、下划线、非 ASCII 字符的名称，现在在与 cx_oracle 通信���正确地转换绑定参数键。（也适用于 0.6.7 版本）

    参考：[#2100](https://www.sqlalchemy.org/trac/ticket/2100)

+   **[oracle]**

    Oracle 方言添加了 use_binds_for_limits=False create_engine()标志，将在行内呈现 LIMIT/OFFSET 值，而不是作为绑定，据报道会修改 Oracle 使用的执行计划。（也适用于 0.6.7 版本）

    参考：[#2116](https://www.sqlalchemy.org/trac/ticket/2116)

### 杂项

+   **[类型]**

    REAL 已添加到核心类型中。由 PostgreSQL、SQL Server、MySQL、SQLite 支持。请注意，SQL Server 和 MySQL 版本，添加了额外的参数，仍然可以从这些方言中使用。

    参考：[#2081](https://www.sqlalchemy.org/trac/ticket/2081)

+   **[类型]**

    添加了@event.listens_for()装饰器，给定目标+事件名称，将装饰的函数应用为监听器。

    参考：[#2106](https://www.sqlalchemy.org/trac/ticket/2106)

+   **[池]**

    AssertionPool 现在存储了当前检出连接获取的轨迹，这个轨迹在第二次并发检出时引发的断言中报告；感谢 Gunnlaugur Briem

    参考：[#2103](https://www.sqlalchemy.org/trac/ticket/2103)

+   **[池]**

    “pool.manage”功能不再使用 pickle 来为每个池的参数进行哈希处理。

+   **[文档]**

    记录了 SQLite DATE/TIME/DATETIME 类型。（也适用于 0.6.7 版本）

    参考：[#2029](https://www.sqlalchemy.org/trac/ticket/2029)

+   **[文档]**

    修复了可变扩展文档以显示正确的类型关联方法。

    参考：[#2118](https://www.sqlalchemy.org/trac/ticket/2118)

## 0.7.0b3

发布日期：2011 年 3 月 20 日

### 通用

+   **[通用]**

    在 PyPy 下运行时，对单元测试进行了大量修复（感谢 Alex Gaynor）。

### orm

+   **[orm]**

    改变了查询.count()的基本方法。现在 query.count()在所有情况下都是确切的：

    > 查询。
    > 
    > from_self(func.count(literal_column(‘1’))). scalar()

    也就是说，“select count(1) from (<full query>)”。这在所有情况下都会产生一个子查询，但大大简化了以前 count() 尝试做的所有猜测，尤其是当涉及到联合表继承和其他连接时，仍然会在许多情况下失败。如果对于本来非常简单的计数而产生的子查询确实是个问题，请使用 query(func.count()) 作为优化。

    参考：[#2093](https://www.sqlalchemy.org/trac/ticket/2093)

+   **[orm]**

    对于迭代过程中罕见的弱引用回调，对身份映射进行了一些更改。已经删除了互斥锁，因为它显然可能导致重新进入（即在一个线程中）死锁，可能是因为 gc 在迭代点收集对象以获得更多内存。希望“迭代过程中字典发生了变化”将会非常罕见，因为迭代方法在内部通过单个 values() 调用获取完整的对象列表。请注意，0.6.7 中的修复更加保守，仍然保持了互斥锁的位置。

    参考：[#2087](https://www.sqlalchemy.org/trac/ticket/2087)

+   **[orm]**

    对工作单元的微调导致按照关系依赖关系对刷新进行排序，即使给定的对象在内存中没有任何属性间的引用，这是 0.5 及更早版本的行为，因此仅设置外键/主键的 Parent/Child 的刷新将成功。这样一来，仍然保持了 0.6 及以上版本不生成大量与当前刷新实际状态不符的无用内部依赖结构。

    参考：[#2082](https://www.sqlalchemy.org/trac/ticket/2082)

+   **[orm]**

    在查询只针对列实体时与（通常不正确地）使用加载器选项一起使用时，错误消息的改进，其中父实体未完全存在。

    参考：[#2069](https://www.sqlalchemy.org/trac/ticket/2069)

+   **[orm]**

    修复了 query.options() 中的错误，该错误使得对使用字符串键的 lazyload 应用路径的属性可能与错误的实体上的同名属性重叠。请注意，0.6.7 对此进行了更加保守的修复。

    参考：[#2098](https://www.sqlalchemy.org/trac/ticket/2098)

### 示例

+   **[示例]**

    更新了关联、关联代理示例以使用声明式，添加了一个新的示例 dict_of_sets_with_default.py，这是一个关联代理的“挑战极限”的示例。

+   **[示例]**

    Beaker 缓存示例允许在 query_callable() 函数中传递一个“query_cls”参数。（也在 0.6.7 中）

    参考：[#2090](https://www.sqlalchemy.org/trac/ticket/2090)

### 引擎

+   **[引擎]**

    修复了 AssertionPool 的回归 bug。

    参考：[#2097](https://www.sqlalchemy.org/trac/ticket/2097)

+   **[引擎]**

    当指定无效的方言时，将引发 ArgumentError 异常。

    参考：[#2060](https://www.sqlalchemy.org/trac/ticket/2060)

### sql

+   **[sql]**

    为 Column 被子类化且 _make_proxy()由于构造函数的 TypeError 而无法复制时添加了完全描述性的错误消息。在这种情况下应该实现 _method _constructor。

+   **[sql]**

    为 Table 对象添加了新的事件“column_reflect”。在反射生成对象之前接收关于 Column 的信息字典，并允许修改字典以控制生成的 Column 的大多数方面，包括键、名称、类型、信息字典。

    参考���[#2095](https://www.sqlalchemy.org/trac/ticket/2095)

+   **[sql]**

    为了帮助使用特定 Table 对象而不是所有 Table 实例的“column_reflect”事件，可以在 Table 对象的构造中使用一个新参数“listeners”添加监听器，一个形式为(<eventname>, <fn>)的元组列表，这些监听器在反射过程开始之前应用于 Table。

+   **[sql]**

    添加了新的通用函数“next_value()”，接受一个 Sequence 对象作为其参数，并在目标平台上呈现适当的“next value”生成字符串，如果支持的话。还在 Sequence 本身上提供“.next_value()”方法。

    参考：[#2085](https://www.sqlalchemy.org/trac/ticket/2085)

+   **[sql]**

    func.next_value()或其他 SQL 表达式可以直接嵌入到 insert()构造中，如果与主键列一起使用隐式或显式的“returning”，则新生成的值将出现在 result.inserted_primary_key 中。

    参考：[#2084](https://www.sqlalchemy.org/trac/ticket/2084)

+   **[sql]**

    为 ResultProxy 添加了访问器“returns_rows”、“is_insert”（也在 0.6.7 中）

    参考：[#2089](https://www.sqlalchemy.org/trac/ticket/2089)

### postgresql

+   **[postgresql]**

    为 postgresql 方言添加了 RESERVED_WORDS。（也在 0.6.7 中）

    参考：[#2092](https://www.sqlalchemy.org/trac/ticket/2092)

+   **[postgresql]**

    修复了 BIT 类型以允许“length”参数、“varying”参数。反射也已修复。（也在 0.6.7 中）

    参考：[#2073](https://www.sqlalchemy.org/trac/ticket/2073)

### mssql

+   **[mssql]**

    重写了用于获取视图定义的查询，通常在使用 Inspector 接口时，使用 sys.sql_modules 而不是信息模式，从而允许完全返回长于 4000 个字符的视图定义。（也在 0.6.7 中）

    参考：[#2071](https://www.sqlalchemy.org/trac/ticket/2071)

### misc

+   **[declarative]**

    在 __mapper_args__ 中不是“可哈希”的参数不会被误认为总是可哈希的，可能是列参数。（也在 0.6.7 中）

    参考：[#2091](https://www.sqlalchemy.org/trac/ticket/2091)

+   **[firebird]**

    如果将“implicit_returning”标志设置为 False，则 create_engine()上的“implicit_returning”标志将被尊重。（也在 0.6.7 中）

    参考：[#2083](https://www.sqlalchemy.org/trac/ticket/2083)

+   **[informix]**

    添加了 RESERVED_WORDS informix 方言。（也在 0.6.7 中）

    参考：[#2092](https://www.sqlalchemy.org/trac/ticket/2092)

+   **[ext]**

    horizontal_shard ShardedSession 类接受常见的 Session 参数“query_cls”作为构造函数参数，以启用对 ShardedQuery 的进一步子类化。（也适用于 0.6.7）

    参考：[#2090](https://www.sqlalchemy.org/trac/ticket/2090)

### 一般

+   **[一般]**

    在 PyPy 下运行时修复了许多单元测试（由 Alex Gaynor 提供）。

### ORM

+   **[ORM]**

    更改了 query.count()的基本方法。现在，在所有情况下，query.count()都是确切的：

    > 查询。
    > 
    > from_self(func.count(literal_column(‘1’))). scalar()

    也就是说，“select count(1) from (<full query>)”。这在所有情况下都会产生一个子查询，但大大简化了以���count()尝试做的所有猜测，以前在许多情况下仍然会失败，特别是当涉及联接表继承和其他联接时。如果为否则非常简单的计数生成的子查询真的是一个问题，请使用 query(func.count())作为优化。

    参考：[#2093](https://www.sqlalchemy.org/trac/ticket/2093)

+   **[ORM]**

    关于迭代期间罕见的弱引用回调进行了一些对身份映射的更改。互斥锁已被移除，因为它显然会导致一个（即在一个线程中）可重入的死锁，也许是在迭代时 gc 在获取更多内存时收集对象的时候。希望“在迭代期间更改字典”会非常罕见，因为迭代方法在内部通过单个 values()调用获取完整的对象列表。请注意，0.6.7 在这里有一个更为保守的修复，仍然保留了互斥锁。

    参考：[#2087](https://www.sqlalchemy.org/trac/ticket/2087)

+   **[ORM]**

    对工作单元进行微调，使其沿着 relationship()依赖关系排序刷新，即使给定的对象在内存中没有任何属性间引用，这是 0.5 及更早版本的行为，因此只设置外键/主键的 Parent/Child 的刷新将成功。同时，仍然保持 0.6 及以上版本不会在刷新中生成大量无用的内部依赖结构，这些结构与当前刷新中实际状态不符。

    参考：[#2082](https://www.sqlalchemy.org/trac/ticket/2082)

+   **[ORM]**

    在与（通常是错误地）使用加载器选项一起查询仅列实体时，改进了发出的错误消息，其中父实体不完全存在。

    参考：[#2069](https://www.sqlalchemy.org/trac/ticket/2069)

+   **[ORM]**

    修复了 query.options()中的一个 bug，其中应用于使用字符串键的延迟加载的路径可能会与错误实体上的同名属性重叠。请注意，0.6.7 对此有一个更为保守的修复。

    参考：[#2098](https://www.sqlalchemy.org/trac/ticket/2098)

### 示例

+   **[示例]**

    更新了关联、关联代理示例以使用声明性，并添加了一个新示例 dict_of_sets_with_default.py，这是一个关联代理的“突破极限”示例。

+   **[示例]**

    Beaker 缓存示例允许在 query_callable()函数中使用“query_cls”参数。（也适用于 0.6.7）

    参考：[#2090](https://www.sqlalchemy.org/trac/ticket/2090)

### engine

+   **[engine]**

    修复了 AssertionPool 回归错误。

    参考：[#2097](https://www.sqlalchemy.org/trac/ticket/2097)

+   **[engine]**

    当指定无效的方言时，将引发 ArgumentError 异常。

    参考：[#2060](https://www.sqlalchemy.org/trac/ticket/2060)

### sql

+   **[sql]**

    为 Column 子类化并由于构造函数的 TypeError 而使 _make_proxy()无法复制时，添加了一个完全描述性的错误消息。在这种情况下应该实现 _method _constructor。

+   **[sql]**

    添加了 Table 对象的新事件“column_reflect”。在生成对象之前，接收有关 Column 的信息字典，并允许修改字典以控制生成的 Column 的大多数方面，包括键、名称、类型、信息字典。

    参考：[#2095](https://www.sqlalchemy.org/trac/ticket/2095)

+   **[sql]**

    为了帮助“column_reflect”事件与特定 Table 对象一起使用，而不是所有 Table 实例，可以在 Table 对象的构造中内联添加监听器，使用一个新的参数“listeners”，一个形式为（<eventname>，<fn>）的元组列表，这些应用于 Table 在反射过程开始之前。

+   **[sql]**

    添加了新的通用函数“next_value()”，接受一个 Sequence 对象作为其参数，并在目标平台上呈现适当的“下一个值”生成字符串，如果支持的话。还在 Sequence 本身上提供了“.next_value()”方法。

    参考：[#2085](https://www.sqlalchemy.org/trac/ticket/2085)

+   **[sql]**

    func.next_value()或其他 SQL 表达式可以直接嵌入到 insert()构造中，如果与主键列一起使用隐式或显式的“returning”，则新生成的值将出现在 result.inserted_primary_key 中。

    参考：[#2084](https://www.sqlalchemy.org/trac/ticket/2084)

+   **[sql]**

    为 ResultProxy 添加了访问器“returns_rows”、“is_insert”（也适用于 0.6.7 版本）

    参考：[#2089](https://www.sqlalchemy.org/trac/ticket/2089)

### postgresql

+   **[postgresql]**

    为 postgresql 方言添加了 RESERVED_WORDS。（也适用于 0.6.7 版本）

    参考：[#2092](https://www.sqlalchemy.org/trac/ticket/2092)

+   **[postgresql]**

    修复了 BIT 类型以允许“length”参数，“varying”参数。反射也已修复。（也适用于 0.6.7 版本）

    参考：[#2073](https://www.sqlalchemy.org/trac/ticket/2073)

### mssql

+   **[mssql]**

    重写了用于获取视图定义的查询，通常在使用 Inspector 接口时，使用 sys.sql_modules 而不是信息模式，从而允许完全返回超过 4000 个字符的视图定义。（也适用于 0.6.7 版本）

    参考：[#2071](https://www.sqlalchemy.org/trac/ticket/2071)

### misc

+   **[declarative]**

    在 __mapper_args__ 中不可“哈希化”的参数不会被误认为始终可哈希化，可能是列参数。（也适用于 0.6.7 版本）

    参考：[#2091](https://www.sqlalchemy.org/trac/ticket/2091)

+   **[firebird]**

    如果将“implicit_returning”标志设置为 False，则 create_engine()上的标志将被遵守。（也适���于 0.6.7 版本）

    参考：[#2083](https://www.sqlalchemy.org/trac/ticket/2083)

+   **[informix]**

    添加了 RESERVED_WORDS informix 方言。（也适用于 0.6.7 版本）

    参考：[#2092](https://www.sqlalchemy.org/trac/ticket/2092)

+   **[ext]**

    horizontal_shard ShardedSession 类接受常见的 Session 参数“query_cls”作为构造函数参数，以便进一步对 ShardedQuery 进行子类化。（也适用于 0.6.7 版本）

    参考：[#2090](https://www.sqlalchemy.org/trac/ticket/2090)

## 0.7.0b2

发布日期：2011 年 2 月 19 日

### orm

+   **[orm]**

    修复了 Session.merge()调用 load()事件时参数少一个的错误。

    参考：[#2053](https://www.sqlalchemy.org/trac/ticket/2053)

+   **[orm]**

    添加了逻辑，防止从 MapperExtension 或 SessionExtension 生成的事件为所有未重写的方法生成无效事件。

    参考：[#2052](https://www.sqlalchemy.org/trac/ticket/2052)

### examples

+   **[examples]**

    Beaker 示例现在考虑了在生成缓存键时嵌入 FROM 子句内的‘limit’和‘offset’、绑定参数（比如当您使用 union()或 from_self()时）。

### sql

+   **[sql]**

    将 EngineEvents 事件类重命名为 ConnectionEvents。由于这些类从不直接被最终用户代码访问，因此这严格来说是一个面向最终用户的文档更改。还简化了内部如何将事件与引擎和连接关联起来的方式。

    参考：[#2059](https://www.sqlalchemy.org/trac/ticket/2059)

+   **[sql]**

    当通过其“metadata”参数传递一个 MetaData()对象给 Sequence()构造函数时，在 metadata.create_all()和 metadata.drop_all()中将包含在 CREATE/DROP 语句中，“checkfirst”逻辑也会被包括进去。

    参考：[#2055](https://www.sqlalchemy.org/trac/ticket/2055)

+   **[sql]**

    Column.references()方法现在在确切引用给定列的外键时返回 True，而不仅仅是其父表。

    参考：[#2064](https://www.sqlalchemy.org/trac/ticket/2064)

### postgresql

+   **[postgresql]**

    修复了从 0.6 版本开始的一个回归问题，即 SMALLINT 和 BIGINT 类型都会在整数主键列上生成 SERIAL，而不是 SMALLINT 和 BIGSERIAL。

    参考：[#2065](https://www.sqlalchemy.org/trac/ticket/2065)

### misc

+   **[declarative]**

    修复了一个回归问题，即将 Column 对象与 composite()一起放置在内联位置时会导致初始化失败。现在 Column 对象可以与 composite()内联或外部，并通过名称或对象引用引入。

    参考：[#2058](https://www.sqlalchemy.org/trac/ticket/2058)

+   **[declarative]**

    修复了错误消息引用旧的@classproperty 名称以引用@declared_attr 的问题（也适用于 0.6.7 版本）

    参考：[#2061](https://www.sqlalchemy.org/trac/ticket/2061)

+   **[declarative]**

    __table_args__ 元组末尾的字典现在是可选的。

    参考：[#1468](https://www.sqlalchemy.org/trac/ticket/1468)

+   **[ext]**

    当代理一个多对一标量属性到一个一对多集合时，关联代理现在对 any()、has() 和 contains() 有了正确的行为（即“典型”关联代理用例的反向）

    参考：[#2054](https://www.sqlalchemy.org/trac/ticket/2054)

### orm

+   **[orm]**

    修复了 Session.merge() 调用 load() 事件时参数少一个的错误。

    参考：[#2053](https://www.sqlalchemy.org/trac/ticket/2053)

+   **[orm]**

    添加了逻辑，防止从 MapperExtension 或 SessionExtension 生成事件，为所有未重写的方法生成无用事件。

    参考：[#2052](https://www.sqlalchemy.org/trac/ticket/2052)

### 示例

+   **[示例]**

    Beaker 示例现在考虑了在生成缓存键时嵌入 FROM 子句内的 'limit' 和 'offset'、绑定参数（比如当使用 union() 或 from_self() 时）。

### sql

+   **[sql]**

    将 EngineEvents 事件类重命名为 ConnectionEvents。由于这些类从不直接被最终用户代码访问，因此这严格来说是最终用户的文档更改。还简化了内部如何将事件链接到引擎和连接的方式。

    参考：[#2059](https://www.sqlalchemy.org/trac/ticket/2059)

+   **[sql]**

    当通过其 'metadata' 参数传递一个 MetaData() 对象给 Sequence() 构造时，将在 metadata.create_all() 和 metadata.drop_all() 中的 CREATE/DROP 语句中包含它，包括“checkfirst”逻辑。

    参考：[#2055](https://www.sqlalchemy.org/trac/ticket/2055)

+   **[sql]**

    Column.references() 方法现在如果有一个外键引用给定列的外���，而不仅仅是其父表，则返回 True。

    参考：[#2064](https://www.sqlalchemy.org/trac/ticket/2064)

### postgresql

+   **[postgresql]**

    从 0.6 中的回归中修复了 SMALLINT 和 BIGINT 类型都会在整数 PK 列上生成 SERIAL，而不是 SMALLINT 和 BIGSERIAL

    参考：[#2065](https://www.sqlalchemy.org/trac/ticket/2065)

### 杂项

+   **[声明式]**

    修复了复合() 与内联放置的 Column 对象会初始化失败的回归。现在，Column 对象可以与复合() 内联或外部，并通过名称或对象引用引入。

    参考：[#2058](https://www.sqlalchemy.org/trac/ticket/2058)

+   **[声明式]**

    修复错误消息引用旧的 @classproperty 名称以引用 @declared_attr（也在 0.6.7 中）

    参考：[#2061](https://www.sqlalchemy.org/trac/ticket/2061)

+   **[声明式]**

    现在，在 __table_args__ 元组末尾的字典是可选的。

    参考：[#1468](https://www.sqlalchemy.org/trac/ticket/1468)

+   **[ext]**

    当代理一个多对一标量属性到一个一对多集合时，关联代理现在对 any()、has() 和 contains() 有了正确的行为（即“典型”关联代理用例的反向）

    参考：[#2054](https://www.sqlalchemy.org/trac/ticket/2054)

## 0.7.0b1

发布日期：2011 年 2 月 12 日星期六

### 通用

+   **[通用]**

    新事件系统，取代所有扩展、监听器等。

    参考：[#1902](https://www.sqlalchemy.org/trac/ticket/1902)

+   **[general]**

    日志增强

    参考：[#1926](https://www.sqlalchemy.org/trac/ticket/1926)

+   **[general]**

    安装不再安装 Nose 插件

    参考：[#1949](https://www.sqlalchemy.org/trac/ticket/1949)

+   **[general]**

    在 sys.modules 中已删除“sqlalchemy.exceptions”别名。基本 SQLA 异常可通过“from sqlalchemy import exc”获得。对于“exc”，“exceptions”别名目前仍在“sqlalchemy”中保留，只是没有被修补到 sys.modules 中。

### orm

+   **[orm]**

    更简洁的 query.join(target, onclause) 形式

    参考：[#1923](https://www.sqlalchemy.org/trac/ticket/1923)

+   **[orm]**

    混合属性，实现/取代 synonym()

    参考：[#1903](https://www.sqlalchemy.org/trac/ticket/1903)

+   **[orm]**

    重写复合物

    参考：[#2008](https://www.sqlalchemy.org/trac/ticket/2008)

+   **[orm]**

    变异事件扩展，取代“mutable=True”

    参见

    变异事件扩展，取代“mutable=True”

+   **[orm]**

    PickleType 和 ARRAY 的可变性默认关闭

    参考：[#1980](https://www.sqlalchemy.org/trac/ticket/1980)

+   **[orm]**

    简化的 polymorphic_on 赋值

    参考：[#1895](https://www.sqlalchemy.org/trac/ticket/1895)

+   **[orm]**

    允许刷新没有父��的孤立体

    参考：[#1912](https://www.sqlalchemy.org/trac/ticket/1912)

+   **[orm]**

    调整刷新记账步骤，以在 autocommit=True 的情况下在提交之前发生。这允许 autocommit=True 与 expire_on_commit=True 正常工作，并且还允许后刷新会话钩子在与 autocommit=False 时相同的事务上下文中运行。

    参考：[#2041](https://www.sqlalchemy.org/trac/ticket/2041)

+   **[orm]**

    在刷新时生成警告，集合成员，标量引用不是刷新的一部分

    参考：[#1973](https://www.sqlalchemy.org/trac/ticket/1973)

+   **[orm]**

    非 Table 派生结构可以映射

    参考：[#1876](https://www.sqlalchemy.org/trac/ticket/1876)

+   **[orm]**

    查询中的元组标签名称改进

    参考：[#1942](https://www.sqlalchemy.org/trac/ticket/1942)

+   **[orm]**

    映射列属性首先引用最具体的列

    参考：[#1892](https://www.sqlalchemy.org/trac/ticket/1892)

+   **[orm]**

    映射到具有两个或更多同名列的连接需要明确声明

    参考：[#1896](https://www.sqlalchemy.org/trac/ticket/1896)

+   **[orm]**

    映射器要求 polymorphic_on 列存在于映射的可选择中

    参考：[#1875](https://www.sqlalchemy.org/trac/ticket/1875)

+   **[orm]**

    compile_mappers() 重命名为 configure_mappers()，简化配置内部

    参考：[#1966](https://www.sqlalchemy.org/trac/ticket/1966)

+   **[orm]**

    如果传递了 SQL FromClause 元素（即非映射类）给 aliased() 函数，它将返回 element.alias() 而不是在 AliasedClass 上引发错误。

    参考：[#2018](https://www.sqlalchemy.org/trac/ticket/2018)

+   **[orm]**

    Session.merge()将检查传入状态的版本 id 与数据库的版本 id 是否匹配，假设映射使用版本 id 并且传入状态已分配版本 id，并在它们不匹配时引发 StaleDataError。

    参考：[#2027](https://www.sqlalchemy.org/trac/ticket/2027)

+   **[orm]**

    Session.connection()，Session.execute()接受‘bind’，以允许执行/连接操作显式参与引擎的开放事务。

    参考：[#1996](https://www.sqlalchemy.org/trac/ticket/1996)

+   **[orm]**

    Query.join()，Query.outerjoin()，eagerload()，eagerload_all()等不再允许将属性列表作为参数（即 option([x, y, z])形式，自 0.5 版本起已被弃用）。

+   **[orm]**

    ScopedSession.mapper 已被移除（自 0.5 版本起已被弃用）。

+   **[orm]**

    水平分片查询将“shard_id”放置在 context.attributes 中，可以通过“load()”事件访问。

    参考：[#2031](https://www.sqlalchemy.org/trac/ticket/2031)

+   **[orm]**

    跨多个实体进行单个 contains_eager()调用将指示沿该路径的所有集合应该加载，而不需要为每个端点分别进行不同的 contains_eager()调用（这从未被正确记录）。

    参考：[#2032](https://www.sqlalchemy.org/trac/ticket/2032)

+   **[orm]**

    在 orm.aliased()中使用的“name”字段现在会在生成的 SQL 语句中呈现。

+   **[orm]**

    Session weak_instance_dict=False 已被弃用。

    参考：[#1473](https://www.sqlalchemy.org/trac/ticket/1473)

+   **[orm]**

    在罕见情况下，如果在父对象被取消引用后发生附加或类似事件的情况，将引发异常，这将阻止父对象在会话中被标记为“脏”。在 0.6.6 中是一个警告。

    参考：[#2046](https://www.sqlalchemy.org/trac/ticket/2046)

+   **[orm]**

    Query.distinct()现在接受列表达式作为*args，由 PostgreSQL 方言解释为 DISTINCT ON (<expr>)。

    参考：[#1069](https://www.sqlalchemy.org/trac/ticket/1069)

+   **[orm]**

    在 flush()期间对“多对一”关系加载进行了额外调整。0.6.6 版本的更改（[ticket:2002]）要求在 flush 期间可能会发生更多“不必要”的 m2o 加载。已添加额外的加载模式，以便在这种特定用例中发出的 SQL 被修剪回来，同时仍然检索 flush 所需的信息，以免遗漏任何内容。

    参考：[#2049](https://www.sqlalchemy.org/trac/ticket/2049)

+   **[orm]**

    作为传递给 attributes.get_history()的“passive”值应该是 attributes 包中定义的常量之一。发送 True 或 False 已被弃用。

+   **[orm]**

    向 Query.subquery()添加了一个 name 参数，以允许为别名对象分配固定名称。（也适用于 0.6.7）

    参考：[#2030](https://www.sqlalchemy.org/trac/ticket/2030)

+   **[orm]**

    当一个联接表继承映射器在本地映射表上没有主键（但在超类表上有主键）时，会发出警告。（也适用于 0.6.7）

    参考：[#2019](https://www.sqlalchemy.org/trac/ticket/2019)

+   **[orm]**

    修复了一个 bug，当多态层次结构中的“中间”类没有指定 'polymorphic_identity' 时，将没有 'polymorphic_on' 列，导致刷新时出现奇怪的错误，从目标查询时加载错误的类。 当使用单表继承时，也会发出正确的 WHERE 条件。 （也适用于 0.6.7）

    参考：[#2038](https://www.sqlalchemy.org/trac/ticket/2038)

+   **[orm]**

    修复了一个 bug，当具有 SQL 或服务器端默认值的列被使用 include_properties 或 exclude_properties 排除时，会导致 UnmappedColumnError。 （也适用于 0.6.7）

    参考：[#1995](https://www.sqlalchemy.org/trac/ticket/1995)

+   **[orm]**

    在罕见情况下，当在父对象被取消引用后发生附加或类似事件时，会发出警告，这会阻止父对象在会话中被标记为“脏”。 这在 0.7 中将是一个异常。（也适用于 0.6.7）

    参考：[#2046](https://www.sqlalchemy.org/trac/ticket/2046)

### sql

+   **[sql]**

    添加了 over() 函数，FunctionElement 类的方法，生成 _Over() 结构，进而生成“窗口函数”，即“<窗口函数> OVER (PARTITION BY <按分区>, ORDER BY <按顺序>)”。

    参考：[#1844](https://www.sqlalchemy.org/trac/ticket/1844)

+   **[sql]**

    LIMIT/OFFSET 子句现在使用绑定参数

    参考：[#805](https://www.sqlalchemy.org/trac/ticket/805)

+   **[sql]**

    select.distinct() 现在接受列表达式作为 *args，由 PostgreSQL 方言解释为 DISTINCT ON (<表达式>)。请注意，通过将列表传递给 select() 的 distinct 关键字参数已经可用。

    参考：[#1069](https://www.sqlalchemy.org/trac/ticket/1069)

+   **[sql]**

    select.prefix_with() 接受多个表达式（即 *表达式），select() 的 'prefix' 关键字参数接受列表或元组。

+   **[sql]**

    将字符串传递给 select() 的 distinct 关键字参数以发出特殊的 MySQL 关键字（DISTINCTROW 等）已被弃用 - 为此使用 prefix_with()。

+   **[sql]**

    TypeDecorator 适用于主键列

    参考：[#2005](https://www.sqlalchemy.org/trac/ticket/2005), [#2006](https://www.sqlalchemy.org/trac/ticket/2006)

+   **[sql]**

    DDL() 构造现在转义百分号

    参考：[#1897](https://www.sqlalchemy.org/trac/ticket/1897)

+   **[sql]**

    Table.c / MetaData.tables 稍微调整，不允许直接修改

    参考：[#1893](https://www.sqlalchemy.org/trac/ticket/1893), [#1917](https://www.sqlalchemy.org/trac/ticket/1917)

+   **[sql]**

    传递给 bindparam() 的可调用对象不会被评估

    参考：[#1950](https://www.sqlalchemy.org/trac/ticket/1950)

+   **[sql]**

    types.type_map 现在是私有的，types._type_map

    参考：[#1870](https://www.sqlalchemy.org/trac/ticket/1870)

+   **[sql]**

    非公共 Pool 方法使用下划线标记

    参考：[#1982](https://www.sqlalchemy.org/trac/ticket/1982)

+   **[sql]**

    添加了 NULLS FIRST 和 NULLS LAST 支持。它作为 asc() 和 desc() 操作符的扩展实现，称为 nullsfirst() 和 nullslast()。

    参考：[#723](https://www.sqlalchemy.org/trac/ticket/723)

+   **[sql]**

    Index() 构造可以与 Table 定义内联创建，使用字符串作为列名，作为在 Table 外创建索引的替代方法。

+   **[sql]**

    Connection 上的 execution_options() 接受“isolation_level”参数，仅为该连接设置事务隔离级别，直到返回到连接池，对于支持它的后端（SQLite，PostgreSQL）

    参考：[#2001](https://www.sqlalchemy.org/trac/ticket/2001)

+   **[sql]**

    Integer 的 TypeDecorator 可以与主键列一起使用，并且各种方言的“autoincrement”特性以及“sqlite_autoincrement”标志将遵守底层数据库类型为 Integer 的设定。

    参考：[#2005](https://www.sqlalchemy.org/trac/ticket/2005)

+   **[sql]**

    当 Integer 主键列存在 server_default 时确立了一致性。SQLA 不会预取这些值，也不会在 cursor.lastrowid（DBAPI）中返回。确保所有后端在这种情况下一致地返回 None 给 result.inserted_primary_key。关于这种情况的反射，具有 server_default 的 int 主键列的反射会将“autoincrement”标志设置为 False，除非是 PG SERIAL 列，我们检测到一个序列默认值。

    参考：[#2020](https://www.sqlalchemy.org/trac/ticket/2020)，[#2021](https://www.sqlalchemy.org/trac/ticket/2021)

+   **[sql]**

    结果行处理器应用于预执行的 SQL 默认值，以及 cursor.lastrowid，在确定 result.inserted_primary_key 的内容时。

    参考：[#2006](https://www.sqlalchemy.org/trac/ticket/2006)

+   **[sql]**

    在 select 的“columns clause”中存在的绑定参数现在像其��“匿名”子句一样自动标记，这样在获取行时它们的“类型”就有意义，就像结果行处理器一样。

+   **[sql]**

    TypeDecorator 存在于“sqlalchemy”导入空间中。

+   **[sql]**

    在 execute() 调用范围内发生的非 DBAPI 错误现在被包装在 sqlalchemy.exc.StatementError 中，并包含 SQL 语句的文本和 params 的 repr()。这使得更容易识别在 DBAPI 参与之前失败的语句执行。

    参考：[#2015](https://www.sqlalchemy.org/trac/ticket/2015)

+   **[sql]**

    将“.bind”直接与 ClauseElement 关联的概念明确地移动到 Executable，即描述表示引擎可执行构造的 ClauseElements 的 mixin。这个改变是对内部组织的改进，不太可能影响任何真实世界的使用。

    参考：[#2048](https://www.sqlalchemy.org/trac/ticket/2048)

+   **[sql]**

    Column.copy() 在 table.tometadata() 中使用，会复制 'doc' 属性。（也适用于 0.6.7）

    参考：[#2028](https://www.sqlalchemy.org/trac/ticket/2028)

+   **[sql]**

    向 resultproxy.c 扩展添加了一些 defs，以便该扩展在 Python 2.4 上编译和运行。（也适用于 0.6.7）

    参考：[#2023](https://www.sqlalchemy.org/trac/ticket/2023)

+   **[sql]**

    编译器扩展现在支持覆盖 expression._BindParamClause 的默认编译，包括 insert()/update() 语句的 VALUES/SET 子句中的自动生成绑定也将使用新的编译规则。（也适用于 0.6.7）

    参考：[#2042](https://www.sqlalchemy.org/trac/ticket/2042)

+   **[sql]**

    SQLite 方言现在对基于文件的数据库使用 NullPool

    参考：[#1921](https://www.sqlalchemy.org/trac/ticket/1921)

+   **[sql]**

    通过 os.path.abspath() 规范给定为 sqlite 数据库位置的路径，以便进程内的目录更改不会影响相对文件路径的最终位置。

    参考：[#2036](https://www.sqlalchemy.org/trac/ticket/2036)

### postgresql

+   **[postgresql]**

    当显式序列执行导致自动生成序列的 SERIAL 列的名称时，目前仅在 implicit_returning=False 时发生，现在采用与 PostgreSQL 相同的逻辑，如果表 + 列名称大于 63 个字符，则予以适应。（也适用于 0.6.7）

    参考：[#1083](https://www.sqlalchemy.org/trac/ticket/1083)

+   **[postgresql]**

    添加了一条额外的 libpq 消息到“断开”异常列表中，“无法从服务器接收数据”（也适用于 0.6.7）

    参考：[#2044](https://www.sqlalchemy.org/trac/ticket/2044)

### mysql

+   **[mysql]**

    新增对 pymysql 的 DBAPI 支持，pymysql 是 MySQL-python 的纯 Python 移植版。

    参考：[#1991](https://www.sqlalchemy.org/trac/ticket/1991)

+   **[mysql]**

    oursql 方言在 create_engine() 中接受与 MySQLdb 相同的“ssl”参数。（也适用于 0.6.7）

    参考：[#2047](https://www.sqlalchemy.org/trac/ticket/2047)

### mssql

+   **[mssql]**

    当未指定长度时，字符串/Unicode 类型及其对应的 VARCHAR/NVARCHAR 类型在不特定长度时发出 “max” 作为长度，因此默认长度，通常为 SQL 服务器文档中的 ‘1’，改为 ‘无界’。对于 VARBINARY 类型也是如此。

    此行为使这些类型与 PostgreSQL 的 VARCHAR 类型更加兼容，当未指定长度时同样是无界的。

    参考：[#1833](https://www.sqlalchemy.org/trac/ticket/1833)

### misc

+   **[no_tags]**

    下面对每个变更的详细描述在这里描述：[`docs.sqlalchemy.org/en/latest/changelog/migration_07.html`](https://docs.sqlalchemy.org/en/latest/changelog/migration_07.html)

+   **[declarative]**

    添加了对在声明类的列属性上使用名称 'metadata' 的情况的显式检查。（也适用于 0.6.7）

    参考：[#2050](https://www.sqlalchemy.org/trac/ticket/2050)

+   **[firebird]**

    一些调整以支持 Interbase。FB/Interbase 版本标识被解析为结构，如(8, 1, 1, ‘interbase’)或(2, 1, 588, ‘firebird’)，以便区分它们。

    参考：[#1885](https://www.sqlalchemy.org/trac/ticket/1885)

### 通用

+   **[通用]**

    新事件系统，取代所有扩展、监听器等

    参考：[#1902](https://www.sqlalchemy.org/trac/ticket/1902)

+   **[通用]**

    日志增强

    参考：[#1926](https://www.sqlalchemy.org/trac/ticket/1926)

+   **[通用]**

    设置不再安装 Nose 插件

    参考：[#1949](https://www.sqlalchemy.org/trac/ticket/1949)

+   **[通用]**

    “sqlalchemy.exceptions”在 sys.modules 中的别名已被移除。基本 SQLA 异常可通过“from sqlalchemy import exc”获得。“exceptions”对于“exc”的别名目前仍在“sqlalchemy”中，只是没有被打补丁到 sys.modules 中。

### orm

+   **[orm]**

    更简洁的查询.join(target, onclause)形式

    参考：[#1923](https://www.sqlalchemy.org/trac/ticket/1923)

+   **[orm]**

    混合属性，实现/取代 synonym()

    参考：[#1903](https://www.sqlalchemy.org/trac/ticket/1903)

+   **[orm]**

    重写复合体

    参考：[#2008](https://www.sqlalchemy.org/trac/ticket/2008)

+   **[orm]**

    变异事件扩展，取代“mutable=True”

    另请参阅

    变异事件扩展，取代“mutable=True”

+   **[orm]**

    PickleType 和 ARRAY 的可变性默认关闭

    参考：[#1980](https://www.sqlalchemy.org/trac/ticket/1980)

+   **[orm]**

    简化的多态性分配

    参考：[#1895](https://www.sqlalchemy.org/trac/ticket/1895)

+   **[orm]**

    允许刷新没有父级的孤立对象

    参考：[#1912](https://www.sqlalchemy.org/trac/ticket/1912)

+   **[orm]**

    调整了在 autocommit=True 情况下在提交之前发生的刷新记账步骤。这使得 autocommit=True 能够与 expire_on_commit=True 正常工作，并且还允许后刷新会话钩子在与 autocommit=False 时相同的事务上下文中运行。

    参考：[#2041](https://www.sqlalchemy.org/trac/ticket/2041)

+   **[orm]**

    在集合成员、标量引用不属于刷新时生成的警告

    参考：[#1973](https://www.sqlalchemy.org/trac/ticket/1973)

+   **[orm]**

    非 Table 派生构造可以映射

    参考：[#1876](https://www.sqlalchemy.org/trac/ticket/1876)

+   **[orm]**

    查询改进中的元组标签名称

    参考：[#1942](https://www.sqlalchemy.org/trac/ticket/1942)

+   **[orm]**

    映射列属性首先引用最具体的列

    参考：[#1892](https://www.sqlalchemy.org/trac/ticket/1892)

+   **[orm]**

    映射到具有两个或更多同名列的连接需要明确声明

    参考：[#1896](https://www.sqlalchemy.org/trac/ticket/1896)

+   **[orm]**

    映射器要求在映射的可选择性中存在多态性列

    参考：[#1875](https://www.sqlalchemy.org/trac/ticket/1875)

+   **[orm]**

    compile_mappers()重命名为 configure_mappers()，简化了配置内部

    参考：[#1966](https://www.sqlalchemy.org/trac/ticket/1966)

+   **[orm]**

    如果传递了 SQL FromClause 元素（即非映射类），aliased()函数将返回 element.alias()，而不会在 AliasedClass 上引发错误。

    参考：[#2018](https://www.sqlalchemy.org/trac/ticket/2018)

+   **[orm]**

    Session.merge()将检查传入状态的版本 id 与数据库的版本 id 是否匹配，假设映射使用版本 id 并且传入状态已分配了 version_id，并且如果它们不匹配，则引发 StaleDataError。

    参考：[#2027](https://www.sqlalchemy.org/trac/ticket/2027)

+   **[orm]**

    Session.connection()，Session.execute()接受‘bind’，以允许执行/连接操作明确参与引擎的开放事务。

    参考：[#1996](https://www.sqlalchemy.org/trac/ticket/1996)

+   **[orm]**

    Query.join()，Query.outerjoin()，eagerload()，eagerload_all()，其他不再接受属性列表作为参数（即 option([x, y, z])形式，自 0.5 版本起已弃用）

+   **[orm]**

    ScopedSession.mapper 已移除（自 0.5 版起已弃用）。

+   **[orm]**

    水平分片查询将“shard_id”放置在 context.attributes 中，可以通过“load()”事件访问。

    参考：[#2031](https://www.sqlalchemy.org/trac/ticket/2031)

+   **[orm]**

    跨多个实体进行单个 contains_eager()调用将指示沿该路径加载所有集合，而不需要针对每个端点进行不同的 contains_eager()调用（这从未被正确记录）。

    参考：[#2032](https://www.sqlalchemy.org/trac/ticket/2032)

+   **[orm]**

    orm.aliased()中使用的“name”字段现在在生成的 SQL 语句中呈现。

+   **[orm]**

    Session weak_instance_dict=False 已弃用。

    参考：[#1473](https://www.sqlalchemy.org/trac/ticket/1473)

+   **[orm]**

    在罕见情况下，如果在父对象被取消引用后发生附加或类似事件，则会引发异常，这会阻止将父对象在会话中标记为“脏”。在 0.6.6 版中是一个警告。

    参考：[#2046](https://www.sqlalchemy.org/trac/ticket/2046)

+   **[orm]**

    Query.distinct()现在接受列表达式作为*args 参数，由 PostgreSQL 方言解释为 DISTINCT ON (<expr>)。

    参考：[#1069](https://www.sqlalchemy.org/trac/ticket/1069)

+   **[orm]**

    在 flush()期间进一步调整“多对一”关系加载。0.6.6 版本中的更改([ticket:2002])要求在 flush 期间可能发生更多“不必要的”m2o 加载。已添加额外的加载模式，以便在这种特定用例中发出的 SQL 被修剪回来，同时仍然检索 flush 需要的信息，以免漏掉任何内容。

    参考：[#2049](https://www.sqlalchemy.org/trac/ticket/2049)

+   **[orm]**

    作为传递给 attributes.get_history() 的“被动”值应该是属性包中定义的常量之一。发送 True 或 False 已弃用。

+   **[orm]**

    向 Query.subquery() 添加了一个 name 参数，以允许将固定名称分配给别名对象。（同时也适用于 0.6.7 版）

    参考：[#2030](https://www.sqlalchemy.org/trac/ticket/2030)

+   **[orm]**

    当一个连接的表继承映射器在本地映射表上没有主键（但在超类表上有主键）时会发出警告。（同时也适用于 0.6.7 版）

    参考：[#2019](https://www.sqlalchemy.org/trac/ticket/2019)

+   **[orm]**

    修复了多态层次结构中的“中间”类如果没有指定“多态标识”则不会有“多态标识”列的错误，导致刷新时出现奇怪的错误，从该目标查询时加载错误的类。还在单表继承时发出正确的 WHERE 条件。（同时也适用于 0.6.7 版）

    参考：[#2038](https://www.sqlalchemy.org/trac/ticket/2038)

+   **[orm]**

    修复了将具有 SQL 或服务器端默认值的列从映射中排除（使用 include_properties 或 exclude_properties）会导致 UnmappedColumnError 的错误。（同时也适用于 0.6.7 版）

    参考：[#1995](https://www.sqlalchemy.org/trac/ticket/1995)

+   **[orm]**

    在罕见情况下，在父对象被取消引用后发生集合的追加或类似事件时会发出警告，这会阻止将父对象标记为会话中的“脏”对象。这将在 0.7 版中成为异常。（同时也适用于 0.6.7 版）

    参考：[#2046](https://www.sqlalchemy.org/trac/ticket/2046)

### sql

+   **[sql]**

    添加了 over() 函数，作为 FunctionElement 类的方法，生成 _Over() 结构，进而生成“窗口函数”，即“<window function> OVER (PARTITION BY <partition by>, ORDER BY <order by>)”。

    参考：[#1844](https://www.sqlalchemy.org/trac/ticket/1844)

+   **[sql]**

    LIMIT/OFFSET 子句现在使用绑定参数

    参考：[#805](https://www.sqlalchemy.org/trac/ticket/805)

+   **[sql]**

    select.distinct() 现在接受列表达式作为 *args，由 PostgreSQL 方言解释为 DISTINCT ON (<expr>)。请注意，通过将列表传递给 select() 的 distinct 关键字参数已经可以实现这一点。

    参考：[#1069](https://www.sqlalchemy.org/trac/ticket/1069)

+   **[sql]**

    select.prefix_with() 接受多个表达式（即 *expr），select() 的 ‘prefix’ 关键字参数接受列表或元组。

+   **[sql]**

    将字符串传递给 select() 的 distinct 关键字参数以发出特殊的 MySQL 关键字（DISTINCTROW 等）已弃用 - 为此使用 prefix_with()。

+   **[sql]**

    TypeDecorator 与主键列一起使用

    参考：[#2005](https://www.sqlalchemy.org/trac/ticket/2005), [#2006](https://www.sqlalchemy.org/trac/ticket/2006)

+   **[sql]**

    DDL() 构造现在转义百分号

    参考：[#1897](https://www.sqlalchemy.org/trac/ticket/1897)

+   **[sql]**

    Table.c / MetaData.tables 稍作调整，不允许直接变异

    参考：[#1893](https://www.sqlalchemy.org/trac/ticket/1893), [#1917](https://www.sqlalchemy.org/trac/ticket/1917)

+   **[sql]**

    传递给 bindparam()的可调用对象不会被评估

    参考：[#1950](https://www.sqlalchemy.org/trac/ticket/1950)

+   **[sql]**

    types.type_map 现在是私有的，types._type_map

    参考：[#1870](https://www.sqlalchemy.org/trac/ticket/1870)

+   **[sql]**

    下划线表示非公共 Pool 方法

    参考：[#1982](https://www.sqlalchemy.org/trac/ticket/1982)

+   **[sql]**

    添加了 NULLS FIRST 和 NULLS LAST 支持。它被实现为 asc()和 desc()运算符的扩展，称为 nullsfirst()和 nullslast()。

    参考：[#723](https://www.sqlalchemy.org/trac/ticket/723)

+   **[sql]**

    Index()构造可以与 Table 定义内联创建，使用字符串作为列名，作为在 Table 之外创建索引的替代方法。

+   **[sql]**

    Connection 上的 execution_options()接受“isolation_level”参数，仅为该连接设置事务隔离级别，直到返回到连接池，对于支持它的后端（SQLite，PostgreSQL）

    参考：[#2001](https://www.sqlalchemy.org/trac/ticket/2001)

+   **[sql]**

    可以使用 Integer 的 TypeDecorator 与主键列一起使用，并且各种方言的“autoincrement”特性以及“sqlite_autoincrement”标志将尊重底层数据库类型为基于 Integer 的情况。

    参考：[#2005](https://www.sqlalchemy.org/trac/ticket/2005)

+   **[sql]**

    当 Integer PK 列上存在 server_default 时确立了一致性。SQLA 不会预取这些，它们也不会在 cursor.lastrowid（DBAPI）中返回。确保所有后端在这种情况下一致地在 result.inserted_primary_key 中返回 None。关于此情况的反射，具有 server_default 的 int PK 列的反射将“autoincrement”标志设置为 False，除了在检测到序列默认值的 PG SERIAL 列的情况下。

    参考：[#2020](https://www.sqlalchemy.org/trac/ticket/2020), [#2021](https://www.sqlalchemy.org/trac/ticket/2021)

+   **[sql]**

    在确定 result.inserted_primary_key 的内容时，结果行处理器应用于预执行的 SQL 默认值，以及 cursor.lastrowid。

    参考：[#2006](https://www.sqlalchemy.org/trac/ticket/2006)

+   **[sql]**

    在 select 的“columns clause”中存在的绑定参数现在像其他“匿名”子句一样自动标记，这样在获取行时它们的“type”就有意义，就像结果行处理器一样。

+   **[sql]**

    TypeDecorator 存在于“sqlalchemy”导入空间中。

+   **[sql]**

    在执行（execute()）调用范围内发生的非 DBAPI 错误现在被包装在 sqlalchemy.exc.StatementError 中，并包含 SQL 语句的文本和 params 的 repr()。这样可以更容易地识别在 DBAPI 介入之前失败的语句执行。

    参考：[#2015](https://www.sqlalchemy.org/trac/ticket/2015)

+   **[sql]**

    将“.bind”直接与 ClauseElement 关联的概念明确地移动到 Executable，即描述表示引擎可执行构造的 ClauseElement 的 mixin。这一变化是对内部组织的改进，不太可能影响任何实际使用。

    参考：[#2048](https://www.sqlalchemy.org/trac/ticket/2048)

+   **[sql]**

    Column.copy()，在 table.tometadata()中使用，复制了‘doc’属性。（也适用于 0.6.7 版本）

    参考：[#2028](https://www.sqlalchemy.org/trac/ticket/2028)

+   **[sql]**

    向 resultproxy.c 扩展添加了一些 defs，以便该扩展在 Python 2.4 上编译和运行。（也适用于 0.6.7 版本）

    参考：[#2023](https://www.sqlalchemy.org/trac/ticket/2023)

+   **[sql]**

    编译器扩展现在支持覆盖默认的表达式编译。_BindParamClause，包括在 insert()/update()语句的 VALUES/SET 子句中自动生成的绑定也将使用新的编译规则。（也适用于 0.6.7 版本）

    参考：[#2042](https://www.sqlalchemy.org/trac/ticket/2042)

+   **[sql]**

    SQLite 方言现在对基于文件的数据库使用 NullPool

    参考：[#1921](https://www.sqlalchemy.org/trac/ticket/1921)

+   **[sql]**

    现在通过 os.path.abspath()对作为 sqlite 数据库位置的路径进行规范化，以便进程内的目录更改不会影响相对文件路径的最终位置。

    参考：[#2036](https://www.sqlalchemy.org/trac/ticket/2036)

### postgresql

+   **[postgresql]**

    当显式序列执行推导出 SERIAL 列的自动生成序列的名称时，当前仅在 implicit_returning=False 时发生，现在使用与 PostgreSQL 相同的逻辑，如果表名+列名大于 63 个字符。 （也适用于 0.6.7 版本）

    参考：[#1083](https://www.sqlalchemy.org/trac/ticket/1083)

+   **[postgresql]**

    将“从服务器接收数据失败”添加到“断开连接”异常列表中的其他 libpq 消息（也适用于 0.6.7 版本）

    参考：[#2044](https://www.sqlalchemy.org/trac/ticket/2044)

### mysql

+   **[mysql]**

    新的 DBAPI 支持 pymysql，这是 MySQL-python 的纯 Python 移植。

    参考：[#1991](https://www.sqlalchemy.org/trac/ticket/1991)

+   **[mysql]**

    oursql 方言在 create_engine()中接受与 MySQLdb 相同的“ssl”参数。（也适用于 0.6.7 版本）

    参考：[#2047](https://www.sqlalchemy.org/trac/ticket/2047)

### mssql

+   **[mssql]**

    String/Unicode 类型及其对应的 VARCHAR/NVARCHAR 类型在未指定长度时会将“max”作为长度发出，因此默认长度，通常根据 SQL 服务器文档为‘1’，现在改为‘无限制’。对于 VARBINARY 类型也是如此。

    这种行为使这些类型更加与 PostgreSQL 的 VARCHAR 类型兼容，当未指定长度时也是无限制的。

    参考：[#1833](https://www.sqlalchemy.org/trac/ticket/1833)

### 杂项

+   **[no_tags]**

    以下是每个更改的详细描述：[`docs.sqlalchemy.org/en/latest/changelog/migration_07.html`](https://docs.sqlalchemy.org/en/latest/changelog/migration_07.html)

+   **[声明性]**

    添加了一个显式检查，以防止在声明性类上使用名称‘metadata’作为列属性的情况。（也在 0.6.7 中）

    参考：[#2050](https://www.sqlalchemy.org/trac/ticket/2050)

+   **[火鸟]**

    进行了一些调整，以支持 Interbase。 FB/Interbase 版本标识被解析成一个结构，如（8, 1, 1, ‘interbase’）或（2, 1, 588, ‘firebird’），以便它们可以被区分。

    参考：[#1885](https://www.sqlalchemy.org/trac/ticket/1885)
