# 0.6 变更日志

> 原文：[`docs.sqlalchemy.org/en/20/changelog/changelog_06.html`](https://docs.sqlalchemy.org/en/20/changelog/changelog_06.html)

## 0.6.9

发布日期：2012 年 5 月 5 日星期六

### 一般

+   **[general]**

    调整了“importlater”机制，该机制在内部用于解决导入循环，使得在导入 sqlalchemy 或 sqlalchemy.orm 之后完成对 __import__ 的使用，从而避免在应用程序启动新线程后继续使用 __import__，修复了问题。

    参考：[#2279](https://www.sqlalchemy.org/trac/ticket/2279)

### orm

+   **[orm] [bug]**

    修复了在查询.get()中对用户映射对象在布尔上下文中的不当评估。

    参考：[#2310](https://www.sqlalchemy.org/trac/ticket/2310)

+   **[orm] [bug]**

    修复了当意外传递元组给 session.query() 时引发的错误格式化。

    参考：[#2297](https://www.sqlalchemy.org/trac/ticket/2297)

+   **[orm]**

    修复了一个错误，即由 query.join() 使用的源子句在针对将多个实体组合在一起的列表达式时会不一致。

    参考：[#2197](https://www.sqlalchemy.org/trac/ticket/2197)

+   **[orm]**

    修复了仅在 Python 3 中显现的错误，即在 flush 期间对持久性 + 待处理对象进行排序会产生非法比较，如果持久性对象的主键不是单个整数。

    参考：[#2228](https://www.sqlalchemy.org/trac/ticket/2228)

+   **[orm]**

    修复了一个错误，即在从具有 join 条件的子表到自身的 joined-inh 结构上使用 query.join() + aliased=True 时，将不适当地将主实体转换为连接实体。

    参考：[#2234](https://www.sqlalchemy.org/trac/ticket/2234)

+   **[orm]**

    修复了一个错误，即 mapper.order_by 属性在子查询急加载中的“内部”查询中将被忽略。

    参考：[#2287](https://www.sqlalchemy.org/trac/ticket/2287)

+   **[orm]**

    修复了一个错误，即如果映射类重新定义了 __hash__() 或 __eq__() 为非标准内容，这是一个受支持的用例，因为 SQLA 不应该查询这些内容，如果该类是“复合”（即非单个实体）结果集的一部分，则会查询这些方法。

    参考：[#2215](https://www.sqlalchemy.org/trac/ticket/2215)

+   **[orm]**

    修复了一个微妙的错误，导致如果出现：column_property()针对子查询 + joinedload + LIMIT + 按列属性排序，则 SQL 会崩溃。

    参考：[#2188](https://www.sqlalchemy.org/trac/ticket/2188)

+   **[orm]**

    由 with_parent 生成的连接条件以及针对父级使用“dynamic”关系时将生成唯一的 bindparams，而不是错误地重复相同的 bindparam。

    参考：[#2207](https://www.sqlalchemy.org/trac/ticket/2207)

+   **[orm]**

    修复了 Query 中的“无语句条件”断言，如果在调用 from_statement() 后调用生成方法，则会尝试引发异常。

    参考：[#2199](https://www.sqlalchemy.org/trac/ticket/2199)

+   **[orm]**

    Cls.column.collate(“some collation”) 现在可用。

    参考：[#1776](https://www.sqlalchemy.org/trac/ticket/1776)

### 示例

+   **[示例]**

    调整 dictlike-polymorphic.py 示例以应用 CAST，使其在 PG 和其他数据库上运行。

    参考：[#2266](https://www.sqlalchemy.org/trac/ticket/2266)

### 引擎

+   **[引擎]**

    回溯了在 0.7.4 中引入的修复，确保在尝试在保存点和两阶段事务上调用 rollback()/prepare()/release() 之前连接处于有效状态。

    参考：[#2317](https://www.sqlalchemy.org/trac/ticket/2317)

### SQL

+   **[SQL]**

    修复了两个关于在可选择的列中的列对应的微妙错误，一个是重复使用相同标记的子查询，另一个是当标记被“分组”并丢失时。影响。

    参考：[#2188](https://www.sqlalchemy.org/trac/ticket/2188)

+   **[SQL]**

    修复了当与某些方言一起使用 String 类型时，“warn on unicode” 标志会被设置的 bug。这个 bug 不在 0.7 版本中。

+   **[SQL]**

    修复了 Select 的 with_only_columns() 方法如果传递了可选择的话会失败的 bug。然而，这里的 FROM 行为仍然不正确，所以无论如何你需要 0.7 版本才能使用这种用例。

    参考：[#2270](https://www.sqlalchemy.org/trac/ticket/2270)

### 模式

+   **[模式]**

    当 ForeignKeyConstraint 引用父级中未找到的列名时，添加了一个信息性错误消息。

### PostgreSQL

+   **[postgresql]**

    修复了与 PG 9 中相同修改的索引行为影响重���名列上的主键反射的 bug。

    参考：[#2141](https://www.sqlalchemy.org/trac/ticket/2141), [#2291](https://www.sqlalchemy.org/trac/ticket/2291)

### MySQL

+   **[mysql]**

    修复了 OurSQL 方言在 XA 命令中使用 ansi-neutral 引号符“’”而不是‘”’的问题。

    参考：[#2186](https://www.sqlalchemy.org/trac/ticket/2186)

+   **[mysql]**

    CREATE TABLE 将在 CHARSET 之后放置 COLLATE 选项，这似乎是 MySQL 关于它是否实际上会起作用的任意规则的一部分。

    参考：[#2225](https://www.sqlalchemy.org/trac/ticket/2225)

### MSSQL

+   **[mssql] [bug]**

    在检索索引名称列表和这些索引中的列名时解码传入的值。

    参考：[#2269](https://www.sqlalchemy.org/trac/ticket/2269)

### Oracle

+   **[oracle]**

    将 ORA-00028 添加到断开代码中，使用 cx_oracle _Error.code 来获取代码。

    参考：[#2200](https://www.sqlalchemy.org/trac/ticket/2200)

+   **[oracle]**

    修复了未生成正确 DDL 的 Oracle.RAW 类型。

    参考：[#2220](https://www.sqlalchemy.org/trac/ticket/2220)

+   **[oracle]**

    将 CURRENT 添加到保留字列表中。

    参考：[#2212](https://www.sqlalchemy.org/trac/ticket/2212)

## 0.6.8

发布日期：2011 年 6 月 5 日 星期日

### ORM

+   **[ORM]**

    对基于列的实体调用 query.get() 是无效的，现在会引发弃用警告。

    参考：[#2144](https://www.sqlalchemy.org/trac/ticket/2144)

+   **[ORM]**

    非主映射器将继承主映射器的 _identity_class。这样，针对通常处于继承映射中的类建立的非主映射器将产生与主映射器兼容的标识映射结果。

    参考：[#2151](https://www.sqlalchemy.org/trac/ticket/2151)

+   **[orm]**

    回溯了 0.7 的标识映射实现，该实现不在删除周围使用互斥体。由于一些用户尽管在 0.6.7 中进行了调整仍然遇到死锁问题；0.7 不使用互斥体的方法似乎不会产生“字典更改大小”问题，这是互斥体的最初理由。

    参考：[#2148](https://www.sqlalchemy.org/trac/ticket/2148)

+   **[orm]**

    修复了“无法为目标列‘q’执行同步规则；映射‘X’未映射此列”发出的错误消息，以引用正确的映射器。

    参考：[#2163](https://www.sqlalchemy.org/trac/ticket/2163)

+   **[orm]**

    修复了确定“自引用”关系时出现的错误，对于没有与自身相关的 joined-inh 子类或与没有在连接条件中的子子类中的列相关的 joined-inh 子类，没有解决方法。

    参考：[#2149](https://www.sqlalchemy.org/trac/ticket/2149)

+   **[orm]**

    在确定父类和子类之间的继承条件时，mapper()将忽略与不相关表的未配置外键。这等同于已应用于声明性的行为。请注意，0.7 有一个更全面的解决方案，改变了 join()本身如何确定 FK 错误。

    参考：[#2153](https://www.sqlalchemy.org/trac/ticket/2153)

+   **[orm]**

    修复了映射到匿名别名的映射器如果使用日志记录将失败的错误，因为别名中的未转义%符号。

    参考：[#2171](https://www.sqlalchemy.org/trac/ticket/2171)

+   **[orm]**

    修改了在刷新时未检测到“identity”键时出现的消息文本，包括常见原因，即列未正确设置以检测自动增量。

    参考：[#2170](https://www.sqlalchemy.org/trac/ticket/2170)

+   **[orm]**

    修复了事务级别的“已删除”集合不会清除已删除状态的错误，如果它们后来变为瞬态，则会引发错误。

    参考：[#2182](https://www.sqlalchemy.org/trac/ticket/2182)

### 引擎

+   **[engine]**

    调整了 RowProxy 结果行的 __contains__()方法，使其在内部不生成异常抛出；无论列构造是否可以强制转换为字符串，NoSuchColumnError()也将生成其消息。

    参考：[#2178](https://www.sqlalchemy.org/trac/ticket/2178)

### sql

+   **[sql]**

    修复了如果将 FetchedValue 传递给列 server_onupdate，则其父“列”不会被分配的错误，为所有列默认分配模式添加了��试覆盖。

    参考：[#2147](https://www.sqlalchemy.org/trac/ticket/2147)

+   **[sql]**

    修复了将 select() 的标签嵌套在另一个标签中会产生不正确导出列的 bug。其中之一是这会破坏针对另一个 column_property() 的 ORM column_property() 映射。

    参考：[#2167](https://www.sqlalchemy.org/trac/ticket/2167)

### postgresql

+   **[postgresql]**

    修复了影响 PG 9 的 bug，即反射索引会失败，如果反射的列名已更改。

    参考：[#2141](https://www.sqlalchemy.org/trac/ticket/2141)

+   **[postgresql]**

    修复了关于数字数组、MATCH 运算符的一些单元测试修复。修复了潜在的浮点不准确性问题，并且目前某�� MATCH 运算符的测试仅在 EN 本地环境中执行。

    参考：[#2175](https://www.sqlalchemy.org/trac/ticket/2175)

### mssql

+   **[mssql]**

    修复了 MSSQL 方言中的 bug，即应用于模式限定表的别名会泄漏到封闭的 select 语句中。

    参考：[#2169](https://www.sqlalchemy.org/trac/ticket/2169)

+   **[mssql]**

    修复了 DATETIME2 类型在结果集或绑定参数中使用时在“适应”步骤中失败的 bug。此问题不在 0.7 版本中。

    参考：[#2159](https://www.sqlalchemy.org/trac/ticket/2159)

## 0.6.7

发布日期：2011 年 4 月 13 日（星期三）

### orm

+   **[orm]**

    加强了关于标识映射迭代与删除的互斥锁，试图减少极其罕见的重新进入 gc 操作导致死锁的机会。可能会在 0.7 版本中移除互斥锁。

    参考：[#2087](https://www.sqlalchemy.org/trac/ticket/2087)

+   **[orm]**

    向 Query.subquery() 添加了一个 name 参数，以允许为别名对象分配固定名称。

    参考：[#2030](https://www.sqlalchemy.org/trac/ticket/2030)

+   **[orm]**

    当连接表继承的映射器在本地映射表上没有主键（但在超类表上有主键）时，会发出警告。

    参考：[#2019](https://www.sqlalchemy.org/trac/ticket/2019)

+   **[orm]**

    修复了多态层次结构中的“中间”类如果没有指定“polymorphic_identity”也没有“polymorphic_on”列时会出现奇怪错误的 bug，导致在查询该目标时加载错误的类。在使用单表继承时也会发出正确的 WHERE 条件。

    参考：[#2038](https://www.sqlalchemy.org/trac/ticket/2038)

+   **[orm]**

    修复了具有 SQL 或服务器端默认值的列，如果使用 include_properties 或 exclude_properties 从映射中排除，将导致 UnmappedColumnError 的 bug。

    参考：[#1995](https://www.sqlalchemy.org/trac/ticket/1995)

+   **[orm]**

    在罕见情况下，如果在父对象被取消引用后发生了追加或类似事件的情况，会发出警告，这会阻止将父对象标记为会话中的“脏”状态。这将在 0.7 版本中成为异常。

    参考：[#2046](https://www.sqlalchemy.org/trac/ticket/2046)

+   **[orm]**

    修复了 query.options() 中的一个 bug，其中应用于使用字符串键的延迟加载的路径可能会与错误的实体上的同名属性重叠。注意，0.7 版本已更新了此修复版本。

    参考文献：[#2098](https://www.sqlalchemy.org/trac/ticket/2098)

+   **[orm]**

    重写了尝试刷新非多态子类时引发的异常的异常信息。

    参考文献：[#2063](https://www.sqlalchemy.org/trac/ticket/2063)

+   **[orm]**

    关于反向引用的状态处理进行了一些修复，通常在 autoflush=False 时，当反向引用的集合没有真正处理没有净变化的添加/删除时。感谢 Richard Murri 提供了测试用例 + 补丁。

    参考文献：[#2123](https://www.sqlalchemy.org/trac/ticket/2123)

+   **[orm]**

    如果使用 from_self()，则“having”子句将从内部复制到外部查询中。

    参考文献：[#2130](https://www.sqlalchemy.org/trac/ticket/2130)

### examples

+   **[examples]**

    Beaker 缓存示例允许在 query_callable() 函数中使用 “query_cls” 参数。

    参考文献：[#2090](https://www.sqlalchemy.org/trac/ticket/2090)

### engine

+   **[engine]**

    修复了 QueuePool、SingletonThreadPool 中的错误，在溢出或定期 cleanup() 时丢弃的连接未显式关闭，导致垃圾回收任务未执行。这通常只影响像 Jython 和 PyPy 这样的非引用计数后端。感谢 Jaimy Azle 发现了这个问题。

    参考文献：[#2102](https://www.sqlalchemy.org/trac/ticket/2102)

### sql

+   **[sql]**

    Column.copy()，如在 table.tometadata() 中使用，将复制 'doc' 属性。

    参考文献：[#2028](https://www.sqlalchemy.org/trac/ticket/2028)

+   **[sql]**

    在 resultproxy.c 扩展中添加了一些 defs，以便扩展能够在 Python 2.4 上编译和运行。

    参考文献：[#2023](https://www.sqlalchemy.org/trac/ticket/2023)

+   **[sql]**

    编译器扩展现在支持重写 expression._BindParamClause 的默认编译，包括 insert()/update() 语句中 VALUES/SET 子句中的自动生成绑定也将使用新的编译规则。

    参考文献：[#2042](https://www.sqlalchemy.org/trac/ticket/2042)

+   **[sql]**

    为 ResultProxy 添加了访问器 “returns_rows”、“is_insert”

    参考文献：[#2089](https://www.sqlalchemy.org/trac/ticket/2089)

+   **[sql]**

    select() 中的 limit/offset 关键字以及传递给 select.limit()/offset() 的值将被强制转换为整数。

    参考文献：[#2116](https://www.sqlalchemy.org/trac/ticket/2116)

### postgresql

+   **[postgresql]**

    当显式序列执行派生 SERIAL 列的自动生成序列的名称时，目前只有在 implicit_returning=False 时才会发生，现在会适应如果表名 + 列名大于 63 个字符，则使用与 PostgreSQL 相同的逻辑。

    参考文献：[#1083](https://www.sqlalchemy.org/trac/ticket/1083)

+   **[postgresql]**

    将一个额外的 libpq 消息添加到“断开连接”异常列表中，“无法从服务器接收数据”

    参考：[#2044](https://www.sqlalchemy.org/trac/ticket/2044)

+   **[postgresql]**

    添加了 postgresql 方言的 RESERVED_WORDS。

    参考：[#2092](https://www.sqlalchemy.org/trac/ticket/2092)

+   **[postgresql]**

    修复了 BIT 类型，允许“length”参数和“varying”参数。反射也修复了。

    参考：[#2073](https://www.sqlalchemy.org/trac/ticket/2073)

### MySQL

+   **[mysql]**

    在 create_engine()中，oursql 方言接受与 MySQLdb 相同的“ssl”参数。

    参考：[#2047](https://www.sqlalchemy.org/trac/ticket/2047)

### SQLite

+   **[sqlite]**

    修复了没有列名创建的外键反射失败的错误。

    参考：[#2115](https://www.sqlalchemy.org/trac/ticket/2115)

### MSSQL

+   **[mssql]**

    重写了用于获取视图定义的查询，通常在使用 Inspector 接口时，使用 sys.sql_modules 而不是信息模式，从而允许完全返回超过 4000 个字符的视图定义。

    参考：[#2071](https://www.sqlalchemy.org/trac/ticket/2071)

### Oracle

+   **[oracle]**

    现在正确地将对 cx_oracle 的绑定参数键进行转换，以便与会引起列本身需要引号或者是为列生成的绑定参数，例如具有特殊字符、下划线、非 ASCII 字符的名称。

    参考：[#2100](https://www.sqlalchemy.org/trac/ticket/2100)

+   **[oracle]**

    Oracle 方言添加了 use_binds_for_limits=False create_engine()标志，将 LIMIT/OFFSET 值内联呈现，而不是作为绑定，据说修改了 Oracle 使用的执行计划。

    参考：[#2116](https://www.sqlalchemy.org/trac/ticket/2116)

### 其他

+   **[informix]**

    添加了 informix 方言的 RESERVED_WORDS。

    参考：[#2092](https://www.sqlalchemy.org/trac/ticket/2092)

+   **[firebird]**

    如果将“implicit_returning”标志设置为 False，则在 create_engine()上将其视为有效。

    参考：[#2083](https://www.sqlalchemy.org/trac/ticket/2083)

+   **[ext]**

    horizontal_shard ShardedSession 类接受公共 Session 参数“query_cls”作为构造函数参数，以启用对 ShardedQuery 的进一步子类化。

    参考：[#2090](https://www.sqlalchemy.org/trac/ticket/2090)

+   **[declarative]**

    添加了对在声明类的列属性上使用名称‘metadata’的情况的明确检查。

    参考：[#2050](https://www.sqlalchemy.org/trac/ticket/2050)

+   **[declarative]**

    修复错误消息引用旧的@classproperty 名称以引用@declared_attr

    参考：[#2061](https://www.sqlalchemy.org/trac/ticket/2061)

+   **[declarative]**

    __mapper_args__ 中的参数如果不是“可哈希的”，则不会被错误地视为总是可哈希的，可能是列参数。

    参考：[#2091](https://www.sqlalchemy.org/trac/ticket/2091)

+   **[documentation]**

    记录了 SQLite DATE/TIME/DATETIME 类型。

    参考：[#2029](https://www.sqlalchemy.org/trac/ticket/2029)

## 0.6.6

发布日期：2011 年 1 月 8 日（星期六）

### orm

+   **[orm]**

    修复了一个 bug，即在干净的对象上发生非“mutable”属性修改事件，除了之前的可变属性更改之外，对象将无法强引用自身在标识映射中。这将导致对象被垃圾回收，丢失任何之前未保存在“mutable changes”字典中的更改。

+   **[orm]**

    修复了“passive_deletes='all'”未在 flush 期间向懒加载器传递正确符号的 bug，从而导致不必要的加载。

    参考：[#2013](https://www.sqlalchemy.org/trac/ticket/2013)

+   **[orm]**

    修复了阻止复合映射属性在映射选择语句中使用的 bug。请注意，复合的工作方式在 0.7 中将发生重大变化。

    参考：[#1997](https://www.sqlalchemy.org/trac/ticket/1997)

+   **[orm]**

    active_history 标志也添加到 composite()。该标志在 0.6 中没有效果，而是一个用于向前兼容性的占位符标志，因为它在 0.7 中适用于复合物。

    参考：[#1976](https://www.sqlalchemy.org/trac/ticket/1976)

+   **[orm]**

    修复了 uow bug，即传递给 Session.delete()的过期对象在删除对象时不会考虑未加载的引用或集合，尽管 passive_deletes 保持默认值 False。

    参考：[#2002](https://www.sqlalchemy.org/trac/ticket/2002)

+   **[orm]**

    当在继承映射器上指定 version_id_col 时，如果继承的映射器已经有一个，并且这些列表达式不相同时，会发出警告。

    参考：[#1987](https://www.sqlalchemy.org/trac/ticket/1987)

+   **[orm]**

    如果在 joinedload()连接链中的先前连接是外连接，则“innerjoin”标志不会沿着连接链生效，从而允许正确返回没有引用子行的主行。

    参考：[#1954](https://www.sqlalchemy.org/trac/ticket/1954)

+   **[orm]**

    修复了关于“subqueryload”策略的错误，即如果实体是 aliased()构造，则策略将失败。

    参考：[#1964](https://www.sqlalchemy.org/trac/ticket/1964)

+   **[orm]**

    修复了关于“subqueryload”策略的 bug，即如果使用形式为 A->joined-subclass->C 的多级加载，则连接将失败。

    参考：[#2014](https://www.sqlalchemy.org/trac/ticket/2014)

+   **[orm]**

    修复了通过-1 对 Query 对象进行索引的错误。它错误地转换为导致 IndexError 的空切片-1:0。

    参考：[#1968](https://www.sqlalchemy.org/trac/ticket/1968)

+   **[orm]**

    映射器参数“primary_key”可以作为单个列传递，也可以作为列表或元组传递。以标量值为例的文档示例已更改为列表。

    参考：[#1971](https://www.sqlalchemy.org/trac/ticket/1971)

+   **[orm]**

    向 relationship()和 column_property()添加了 active_history 标志，强制属性事件始终加载“旧”值，以便 attributes.get_history()可以访问它。

    参考：[#1961](https://www.sqlalchemy.org/trac/ticket/1961)

+   **[orm]**

    如果复合键中的参数数量过大或过小，Query.get() 将会引发异常。

    参考：[#1977](https://www.sqlalchemy.org/trac/ticket/1977)

+   **[orm]**

    从 0.7 中回溯了“优化获取”修复，改善了联合继承“加载过期行”行为的生成。

    参考：[#1992](https://www.sqlalchemy.org/trac/ticket/1992)

+   **[orm]**

    在“primaryjoin”错误中添加了更多详细信息，对于一个异常条件，join 条件对于 viewonly 工作但对于非 viewonly 不工作，且未使用 foreign_keys - 在建议中添加“foreign_keys”。还将“foreign_keys”添加到一般的“direction”错误的建议中。

### 示例

+   **[examples]**

    版本示例现在支持检测关联关系()中的更改。

### 引擎

+   **[engine]**

    当显式使用 Unicode 类型时，只有当 convert_unicode=True 用于引擎或 String 类型时，才会引发针对非 Unicode 绑定数据的“unicode warning”，而不是当 convert_unicode=True 用于引擎或 String 类型时。

+   **[engine]**

    修复了 Decimal 结果处理器 C 版本的内存泄漏问题。

    参考：[#1978](https://www.sqlalchemy.org/trac/ticket/1978)

+   **[engine]**

    为 RowProxy 的 C 版本实现了序列检查功能，以及对 RowProxy 实现了 2.7 风格的“collections.Sequence”注册。

    参考：[#1871](https://www.sqlalchemy.org/trac/ticket/1871)

+   **[engine]**

    Threadlocal 引擎方法 rollback()、commit()、prepare() 在没有事务进行时不会引发异常；这是在 0.6 中引入的一个回归。

    参考：[#1998](https://www.sqlalchemy.org/trac/ticket/1998)

+   **[engine]**

    Threadlocal 引擎在 begin()、begin_nested()后返回自身；然后引擎实现了上下文管理器方法，以允许“with”语句。

    参考：[#2004](https://www.sqlalchemy.org/trac/ticket/2004)

### sql

+   **[sql]**

    修复了单个非关联操作符链的操作符优先级规则。即“x - (y - z)”将编译为“x - (y - z)”而不是“x - y - z”。也适用于标签，即“x - (y - z).label('foo')”

    参考：[#1984](https://www.sqlalchemy.org/trac/ticket/1984)

+   **[sql]**

    在 Column.copy() 期间复制了 Column 的‘info’属性，即在声明性 mixin 中使用列时发生的情况。

    参考：[#1967](https://www.sqlalchemy.org/trac/ticket/1967)

+   **[sql]**

    为布尔值添加了一个绑定处理器，将其强制转换为 int，用于像 pymssql 这样的 DBAPI，其对值简单地调用 str()。

+   **[sql]**

    CheckConstraint 将在 copy()/tometadata() 中复制其‘initially’、‘deferrable’和‘_create_rule’属性

    参考：[#2000](https://www.sqlalchemy.org/trac/ticket/2000)

### postgresql

+   **[postgresql]**

    IN 子句内的单元素元组表达式正确地加上了括号，同样来自于

    参考：[#1984](https://www.sqlalchemy.org/trac/ticket/1984)

+   **[postgresql]**

    确保 psycopg2 和 pg8000 的“numeric”基本类型能够识别每个数字、浮点数、整数代码、标量 + 数组。

    参考：[#1955](https://www.sqlalchemy.org/trac/ticket/1955)

+   **[postgresql]**

    为 UUID 类型添加了 as_uuid=True 标志，将接收和返回值作为 Python UUID() 对象而不是字符串。目前，UUID 类型仅已知与 psycopg2 兼容。

    参考：[#1956](https://www.sqlalchemy.org/trac/ticket/1956)

+   **[postgresql]**

    修复了一个错误，即在池销毁+重新创建后，非 ENUM 支持的 PG 版本会出现 KeyError。

    参考：[#1989](https://www.sqlalchemy.org/trac/ticket/1989)

### mysql

+   **[mysql]**

    修复了 Jython + zxjdbc 的错误处理，使 has_table() 属性再次有效。这是从 0.6.3 版本开始的回归（我们没有 Jython 的构建机器，抱歉）

    ��考：[#1960](https://www.sqlalchemy.org/trac/ticket/1960)

### sqlite

+   **[sqlite]**

    在 CREATE TABLE 中的 REFERENCES 子句中，如果包含了指向具有相同模式名称的另一个表的远程模式，现在将按照 SQLite 的要求渲染远程名称而不包含模式子句。

    参考：[#1851](https://www.sqlalchemy.org/trac/ticket/1851)

+   **[sqlite]**

    在相同主题上，如果在 CREATE TABLE 中的 REFERENCES 子句中包含了指向父表模式不同的远程模式的表，则根本不会渲染，因为似乎不支持跨模式引用。

### mssql

+   **[mssql]**

    对索引反射的重写遗憾地没有经过正确测试，并返回了不正确的结果。这个回归现在已经修复。

    参考：[#1770](https://www.sqlalchemy.org/trac/ticket/1770)

### oracle

+   **[oracle]**

    cx_oracle 的“十进制检测”逻辑，用于具有模糊数值特征的结果集列，现在使用由区域设置/ NLS_LANG 设置确定的小数点字符，使用首次连接时检测此字符。在使用非句点小数点 NLS_LANG 设置时，还需要 cx_oracle 5.0.3 或更高版本。

    参考：[#1953](https://www.sqlalchemy.org/trac/ticket/1953)

### 杂项

+   **[firebird]**

    Firebird 数值类型现在明确检查 Decimal，让 float() 直接通过，从而允许特殊值如 float('inf')。

    参考：[#2012](https://www.sqlalchemy.org/trac/ticket/2012)

+   **[declarative]**

    如果 __table_args__ 不是元组或字典格式，并且不是 None，则会引发错误。

    参考：[#1972](https://www.sqlalchemy.org/trac/ticket/1972)

+   **[sqlsoup]**

    为 SqlSoup 添加了“map_to()”方法，这是一个“主”方法，接受每个可选择和映射的显式参数，包括每个映射的基类。

    参考：[#1975](https://www.sqlalchemy.org/trac/ticket/1975)

+   **[sqlsoup]**

    与 map()、with_labels()、join() 方法一起使用的映射可选择不再将给定参数放入内部“缓存”字典中。特别是因为 join() 和 select() 对象是在方法本身中创建的，这几乎是一种纯粹的内存泄漏行为。

## 0.6.5

发布日期：2010 年 10 月 24 日星期日

### orm

+   **[orm]**

    添加了一个新的“lazyload”选项“immediateload”。在对象被填充时自动发出通常的“lazy”加载操作。这里的用例是在加载对象以放置在离线缓存中，或在会话不可用后使用时，希望进行直接的‘select’加载，而不是‘joined’或‘subquery’。

    参考：[#1914](https://www.sqlalchemy.org/trac/ticket/1914)

+   **[orm]**

    新的 Query 方法：query.label(name)，query.as_scalar()，将查询的语句作为标量子查询返回/不返回标签；query.with_entities(*ent)，用新实体替换查询的 SELECT 列表。大致相当于接受映射实体以及列表达式的 query.values()的生成形式。

    参考：[#1920](https://www.sqlalchemy.org/trac/ticket/1920)

+   **[orm]**

    修复了递归 bug，当将一个对象从一个引用移动到另一个引用时可能发生，涉及到反向引用，其中发起父类是以前父类的子类（具有自己的 mapper）。

+   **[orm]**

    修复了 0.6.4 中的一个回归，如果您在 mapper()上传递一个空列表给“include_properties”。

    参考：[#1918](https://www.sqlalchemy.org/trac/ticket/1918)

+   **[orm]**

    修复了 Query 中的标签错误，如果任何列表达式未标记，则 NamedTuple 会错误应用标签。

+   **[orm]**

    修复了一个问题，即 query.join()会不适当地将右侧适应为左侧连接的右侧

    参考：[#1925](https://www.sqlalchemy.org/trac/ticket/1925)

+   **[orm]**

    Query.select_from()已经加强，以确保后续调用 query.join()将使用 select_from()实体，假设它是一个映射实体而不是一个普通可选择的实体，作为默认的“左”侧，而不是 Query 对象的实体列表中的第一个实体。

+   **[orm]**

    当在子事务回滚后（这是在 autocommit=False 模式下刷新失败时发生的情况）继续使用 Session 时引发的异常现在已经重新表述（这是“由于子事务回滚而处于非活动状态”消息）。特别是，如果回滚是由于刷新期间的异常引起的，则消息会说明这种情况，并重申在刷新期间发生的原始异常的字符串形式。如果会话由于显式使用子事务而关闭（这种情况并不常见），消息只会说明这种情况。

+   **[orm]**

    当 Mapper 在初始化失败后重复请求其初始化时引发的异常不再假定“hasattr”情况，因为还有其他情况会发出此消息，并且消息也不会多次叠加 - 每次尝试使用时都会得到相同的消息。误称“编译”正在被“初始化”替换。

+   **[orm]**

    修复了 query.update()中的 bug，其中‘evaluate’或‘fetch’到期会失败，如果列表达式键是具有不同键名的类属性作为实际列名。

    参考：[#1935](https://www.sqlalchemy.org/trac/ticket/1935)

+   **[orm]**

    在 flush 期间添加了一个断言，确保“新持久”对象上没有生成包含 NULL 的标识键。当用户定义的代码无意中触发未完全加载的对象的 flush 时，可能会发生这种情况。

+   **[orm]**

    现在，关系属性的惰性加载在发出 SQL 时使用当前状态而不是“已提交”状态的外键和主键属性，如果没有进行 flush。以前，只会使用数据库已提交的状态。特别是，这会导致许多对一的 get()-on-lazyload 操作失败，因为在这些加载时不会触发自动 flush，属性被确定时“已提交”状态可能不可用。

    参考：[#1910](https://www.sqlalchemy.org/trac/ticket/1910)

+   **[orm]**

    在 relationship()上的一个新标志，load_on_pending，允许延迟加载器在未进行 flush 的情况下对待挂起的对象进行触发，以及手动“附加”到会话的瞬态对象。请注意，此标志在加载对象时阻止属性事件发生，因此直到 flush 之后才可用反向引用。该标志仅用于非常特定的用例。

+   **[orm]**

    另一个新标志 relationship()，cascade_backrefs，当事件在双向关系的“反向”侧启动时禁用“save-update”级联。这是一种更清晰的行为，使得可以在瞬态对象上设置多对一而不会被吸入子对象的会话，同时仍允许前向集合级联。我们*可能*会在 0.7 中将其默认设置为 False。

+   **[orm]**

    在关系上仅在多对一的一侧放置 passive_updates=False 时，对“passive_updates=False”行为进行了轻微改进；文档已澄清 passive_updates=False 应该真正放在一对多的一侧。

+   **[orm]**

    在多对一上放置 passive_deletes=True 会发出警告，因为您可能打算将其放在一对多的一侧。

+   **[orm]**

    修复了一个 bug，该 bug 会阻止“subqueryload”与子类的关系在单表继承中正常工作-“where type in (x, y, z)”只会被放置在内部，而不是重复放置。

+   **[orm]**

    当在单表继承中使用 from_self()时，“where type in (x, y, z)”仅放在查询的外部，而不是重复放置。可能需要对此进行一些调整。

+   **[orm]**

    当调用 configure()时，scoped_session 会在当前线程中检查是否已经存在 Session，如果存在则会发出警告。

    参考：[#1924](https://www.sqlalchemy.org/trac/ticket/1924)

+   **[orm]**

    重新设计了 mapper.cascade_iterator()的内部，以在某些情况下减少约 9%的方法调用。

    参考：[#1932](https://www.sqlalchemy.org/trac/ticket/1932)

### engine

+   **[engine]**

    修复了 0.6.4 中的一个回归，其中允许一致地引发游标错误的更改破坏了 result.lastrowid 访问器。为 result.lastrowid 添加了测试覆盖范围。请注意，lastrowid 仅由 Pysqlite 和一些 MySQL 驱动程序支持，因此在一般情况下并不是特别有用。

+   **[engine]**

    当连接首次被使用时，引擎发出的日志消息现在是“BEGIN (implicit)”，以强调 DBAPI 没有显式的 begin()。

+   **[engine]**

    添加了“views=True”选项到 metadata.reflect()，将向正在反映的视图列表中添加可用视图。

    参考：[#1936](https://www.sqlalchemy.org/trac/ticket/1936)

+   **[engine]**

    engine_from_config()现在接受“debug”用于“echo”，“echo_pool”，“force”用于“convert_unicode”，布尔值用于“use_native_unicode”。

    参考：[#1899](https://www.sqlalchemy.org/trac/ticket/1899)

### sql

+   **[sql]**

    修复了 TypeDecorator 中的错误，其中方言特定类型被引入以生成给定类型的 DDL，这不总是返回正确的结果。

+   **[sql]**

    TypeDecorator 现在可以将完全构造的类型指定为其“impl”，而不仅仅是类型类。

+   **[sql]**

    TypeDecorator 现在会将自己作为二元表达式的结果类型，其中类型强制转换规则通常会返回其实现类型 - 以前，将返回 impl 类型的副本，该类型将 TypeDecorator 嵌入到其中作为“方言”实现，这可能是实现所需效果的无意之举。

+   **[sql]**

    TypeDecorator.load_dialect_impl() 默认返回“self.impl”，即不返回“self.impl”的方言实现类型。这样做是为了支持正确的编译。行为可以像以前一样由用户重写，效果相同。

+   **[sql]**

    添加了 type_coerce(expr, type_)表达式元素。在评估表达式和处理结果行时，将给定表达式视为给定类型，但不影响 SQL 的生成，除了一个匿名标签。

+   **[sql]**

    Table.tometadata()现在还会复制与 Table 关联的 Index 对象。

+   **[sql]**

    如果给定的表已经存在于目标 MetaData 中，则 Table.tometadata()会发出警告 - 将返回现有的 Table 对象。

+   **[sql]**

    如果尚未为列分配名称（即在声明时），则在将其导出到封闭的 select()构造的列集合或在分配名称之前编译包含该列的任何构造的上下文中使用列，将引发一个信息性错误消息。

+   **[sql]**

    as_scalar()，label()可以在包含尚未命名列的可选项上调用。

    参考：[#1862](https://www.sqlalchemy.org/trac/ticket/1862)

+   **[sql]**

    修复了操作两个表达式都是“NullType”类型但不是单例 NULLTYPE 实例时可能发生的递归溢出。

    参考：[#1907](https://www.sqlalchemy.org/trac/ticket/1907)

### postgresql

+   **[postgresql]**

    为 ARRAY 类型添加了“as_tuple”标志，返回结果为元组而不是列表，以允许哈希。

+   **[postgresql]**

    修复了一个 bug，阻止了从自定义类型（如“enum”）构建的“domain”被反射。

    参考：[#1933](https://www.sqlalchemy.org/trac/ticket/1933)

### mysql

+   **[mysql]**

    修复了涉及使用 ON UPDATE 子句的 CURRENT_TIMESTAMP 默认值的反射 bug，感谢 Taavi Burns。

    参考：[#1940](https://www.sqlalchemy.org/trac/ticket/1940)

### mssql

+   **[mssql]**

    修复了一个未能正确处理未知类型反射的 bug。

    参考：[#1946](https://www.sqlalchemy.org/trac/ticket/1946)

+   **[mssql]**

    修复了使用“schema”别名表时无法正确编译的 bug。

    参考：[#1943](https://www.sqlalchemy.org/trac/ticket/1943)

+   **[mssql]**

    重写了索引的反射以使用 sys.目录，以便反射任何配置的列名称（空格，嵌入逗号等）。请注意，反射索引需要 SQL Server 2005 或更高版本。

    参考：[#1770](https://www.sqlalchemy.org/trac/ticket/1770)

+   **[mssql]**

    mssql+pymssql 方言现在尊重 URL 的“port”部分，而不是丢弃它。

    参考：[#1952](https://www.sqlalchemy.org/trac/ticket/1952)

### oracle

+   **[oracle]**

    无论检测到的 Oracle 版本如何，create_engine()的 implicit_returning 参数现在都会被尊重。以前，如果服务器版本信息<10，则该标志将被强制为 False。

    参考：[#1878](https://www.sqlalchemy.org/trac/ticket/1878)

### tests

+   **[tests]**

    NoseSQLAlchemyPlugin 已移至新包“sqlalchemy_nose”，该包与“sqlalchemy”一起安装。这样���“nosetests”脚本仍然可以正常工作，但也允许在导入 SQLAlchemy 模块之前打开覆盖率，从而使覆盖率能够正常工作。

### misc

+   **[declarative]**

    @classproperty（即将/现在@declared_attr）对于不是 mixin 的基类以及 mixins 上的 __mapper_args__，__table_args__，__tablename__ 生效。

    参考：[#1922](https://www.sqlalchemy.org/trac/ticket/1922)

+   **[declarative]**

    @classproperty 的官方名称/位置用于与 declarative 一起使用是 sqlalchemy.ext.declarative.declared_attr。虽然是同一件事，但由于它更多地是一个特定于 declarative 的“标记”，而不仅仅是一个属性技术，所以将其移动到那里。

    参考：[#1915](https://www.sqlalchemy.org/trac/ticket/1915)

+   **[declarative]**

    修复了一个 bug，即在一个 mixin 上的列无法正确传播到单表或联合表继承方案，其中属性名称与列的名称不同。

    参考：[#1930](https://www.sqlalchemy.org/trac/ticket/1930)，[#1931](https://www.sqlalchemy.org/trac/ticket/1931)

+   **[declarative]**

    现在，mixin 可以指定一个覆盖与超类关联的同名列的列。感谢 Oystein Haaland。

+   **[informix]**

    *重大*清理/现代化 Informix 方言为 0.6，感谢 Florian Apolloner。

    参考：[#1906](https://www.sqlalchemy.org/trac/ticket/1906)

+   **[misc]**

    CircularDependencyError 现在有 .cycles 和 .edges 成员，它们是一个或多个循环中涉及的元素集合，以及作为 2 元组的边的集合。

    参考：[#1890](https://www.sqlalchemy.org/trac/ticket/1890)

## 0.6.4

发布日期：Tue Sep 07 2010

### orm

+   **[orm]**

    ConcurrentModificationError 的名称已更改为 StaleDataError，并且描述性错误消息已经修订以准确反映问题所在。在可预见的未来，这两个名称都将保持可用，以供可能在“except:”子句中指定 ConcurrentModificationError 的方案使用。

+   **[orm]**

    在标识映射中添加了一个互斥锁，该互斥锁对迭代方法中的删除操作进行了互斥，这些方法现在在返回可迭代对象之前进行了预缓冲。这是因为异步 gc 可以随时通过 gc 线程删除项目。

    参考：[#1891](https://www.sqlalchemy.org/trac/ticket/1891)

+   **[orm]**

    Session 类现在存在于 sqlalchemy.orm.* 中。我们正在摆脱使用 create_session()，该函数具有非标准默认值，用于需要一步构造会话的情况。然而，大多数用户应该坚持使用 sessionmaker() 进行一般用途。

+   **[orm]**

    query.with_parent() 现在接受瞬态对象，并将使用它们的 pk/fk 属性的非持久化值来制定条件。文档也澄清了 with_parent() 的目的。

+   **[orm]**

    include_properties 和 exclude_properties 参数现在接受 Column 对象作为成员，而不仅仅是字符串。这样，可以消除 join() 中的同名 Column 对象等歧义。

+   **[orm]**

    如果针对包含多个具有相同名称的列的 join 或其他单个可选择的映射器创建了一个映射器，并且这些列没有明确命名为相同或不同的属性（或排除），则现在会发出警告。在 0.7 中，此警告将是一个异常。请注意，当组合发生时，不会发出此警告作为继承的结果，因此属性仍然允许自然覆盖。在 0.7 中，这将进一步改进。

    参考：[#1896](https://www.sqlalchemy.org/trac/ticket/1896)

+   **[orm]**

    mapper() 的 primary_key 参数现在可以指定一系列列，这些列仅是映射可选择的计算“主键”列的子集，而不会引发错误。这有助于在可选择的有效主键比实际标记为“主键”的可选择的列数更简单的情况下，例如在两个表的主键列上进行连接时。

    参考：[#1896](https://www.sqlalchemy.org/trac/ticket/1896)

+   **[orm]**

    已删除的对象现在会得到一个名为 'deleted' 的标志，这将阻止该对象被重新添加到会话中，因为以前该对象将悄悄地存在于标识映射中，直到其属性被访问。make_transient() 函数现在重置此标志以及“key”标志。

+   **[orm]**

    make_transient() 可以安全地在已经瞬态的实例上调用。

+   **[orm]**

    在 mapper() 中如果 polymorphic_on 列在映射的可选择对象或 with_polymorphic 可选择对象中以直接或派生形式不存在，则会发出警告，而不是悄悄地忽略它。请注意，预计在 0.7 版本中会将此行为更改为异常。

+   **[orm]**

    当 relationship() 配置具有模糊参数时，对发出的一系列错误消息进行另一次遍历。不再提及“foreign_keys”设置，因为它几乎从不需要，并且更推荐用户设置正确的 ForeignKey 元数据，这也是现在的推荐做法。如果使用了'foreign_keys'并且不正确，该消息将建议该属性可能是不必要的。对属性的文档进行了加强。这是因为所有在 ML 上困惑的 relationship() 用户似乎都试图使用 foreign_keys，而消息只会进一步使他们困惑，因为 Table 元数据更加清晰。

+   **[orm]**

    如果“secondary”表没有 ForeignKey 元数据并且没有设置 foreign_keys，即使用户传递了错误的信息，也假定主/辅助连接表达式应仅考虑“secondary”中的所有列作为外键。在任何情况下，“secondary”中的外键都不可能在其他地方。现在会发出警告而不是错误，并且映射成功。

    参考资料：[#1877](https://www.sqlalchemy.org/trac/ticket/1877)

+   **[orm]**

    在 flush 期间，如果从一个集合中移动一个 o2m 对象到另一个集合，或者通过更改 m2o 引用的对象，其中外键也是主键的成员，现在将更加谨慎地检查，如果“多”一侧的外键值的更改是由于“一”侧主键的更改引起的，或者如果“一”侧只是一个不同的对象。在一种情况下，可进行级联的 DB 已经级联了该值，我们需要查看“新”PK 值以进行 UPDATE，在另一种情况下，我们需要继续查看“旧”的 PK。我们现在查看“旧”的 PK，假设 passive_updates=True，除非我们知道触发更改的是 PK 切换。

    参考资料：[#1856](https://www.sqlalchemy.org/trac/ticket/1856)

+   **[orm]**

    可以手动更改 version_id_col 的值，这将导致行的 UPDATE。版本化的 UPDATE 和 DELETE 现在使用 version_id_col 的“committed”值而不是待定的更改值作为 WHERE 子句，并且如果属性上存在手动更改，则版本生成器也会被绕过。

    参考资料：[#1857](https://www.sqlalchemy.org/trac/ticket/1857)

+   **[orm]**

    修复了在与具体继承的映射器一起使用 merge() 时的使用情况。这种映射器经常具有所谓的“具体”属性，即“禁用”从父级传播的子类属性 - 这些属性需要允许 merge() 操作通过而不产生效果。

+   **[orm]**

    对于 column_mapped_collection 的非列基础参数的指定，包括 string、text() 等，将会引发一个错误消息，明确要求列元素，不再误导关于 text() 或 literal() 的错误信息。

    参考：[#1863](https://www.sqlalchemy.org/trac/ticket/1863)

+   **[orm]**

    类似地，对于 relationship()、foreign_keys、remote_side、order_by - 所有基于列的表达式都受到强制约束 - 明确禁止使用字符串列表，因为这是一个非常常见的错误

+   **[orm]**

    动态属性不支持集合填充 - 当调用 set_committed_value() 时添加了一个断言，以及当对动态属性应用 joinedload() 或 subqueryload() 选项时，而不是失败 / 静默失败。

    参考：[#1864](https://www.sqlalchemy.org/trac/ticket/1864)

+   **[orm]**

    修复了一个 bug，该 bug 导致从一个具有不同标签名重复的同一列的查询派生的查询，在一些 UNION 情况下通常会失败，无法完全传播内部列到外部查询。

    参考：[#1852](https://www.sqlalchemy.org/trac/ticket/1852)

+   **[orm]**

    当提供一个未映射实例时，object_session() 现在会引发正确的 UnmappedInstanceError。

    参考：[#1881](https://www.sqlalchemy.org/trac/ticket/1881)

+   **[orm]**

    对计算的 Mapper 属性进一步进行了记忆化处理，在高度多态映射配置中显著减少了（约 90%）运行时 mapper.py 的调用次数。

+   **[orm]**

    被版本示例使用的 mapper _get_col_to_prop 私有方法已被弃用；现在使用 mapper.get_property_by_column()，该方法将保持对此的公共访问。

+   **[orm]**

    现在版本示例在以前为 NULL 的列上进行版本控制时工作正常。

### 示例

+   **[examples]**

    beaker_caching 示例已重新组织，使 Session、cache 管理器、declarative_base 成为环境的一部分，并且自定义缓存代码是可移植的，现在在“caching_query.py”中。这使得示例更容易“插入”到现有项目中。

+   **[examples]**

    当复制列时，history_meta 版本控制示例设置“unique=False”，以便版本控制表处理具有重复值的多行。

    参考：[#1887](https://www.sqlalchemy.org/trac/ticket/1887)

### engine

+   **[engine]**

    对已经耗尽、已关闭或不是返回结果的结果执行 fetchone() 或类似操作现在会在所有情况下引发 ResourceClosedError，这是 InvalidRequestError 的子类，无论后端如何。以前，一些 DBAPI 会引发 ProgrammingError（例如 pysqlite），其他会返回 None，导致下游断裂（例如 MySQL-python）。

+   **[engine]**

    修复了在 `Connection` 中的一个错误，即如果在第一个连接池连接的“初始化”阶段发生“断开连接”事件，那么当 `Connection` 尝试使 DBAPI 连接无效时将引发 `AttributeError`。

    参考：[#1894](https://www.sqlalchemy.org/trac/ticket/1894)

+   **[engine]**

    对于所有“此连接/事务/结果已关闭”的错误类型，`Connection`、`ResultProxy` 以及 `Session` 现在使用 `ResourceClosedError`。

+   **[engine]**

    `Connection.invalidate()` 可以被多次调用，而后续的调用将不起作用。

### sql

+   **[sql]**

    在 `alias()` 构造上调用 `execute()` 方法在 0.7 版本中将被废弃，因为它本身不是一个“可执行”构造。它目前“代理”其内部元素并且有条件地“可执行”，但这不是我们喜欢的模糊情况。

+   **[sql]**

    `ClauseElement` 的 `execute()` 和 `scalar()` 方法现在已经适当地移动到 `Executable` 子类中。`ClauseElement.execute()` / `scalar()` 仍然存在，并且在 0.7 版本中待废弃，但请注意，如果你不是 `Executable`（除非你是 `alias()`，请参阅前面的注释），这些方法将始终引发错误。

+   **[sql]**

    为 Numeric->Integer 添加了基本的数学表达式强制转换，以便无论表达式的方向如何，结果类型都是 Numeric。

+   **[sql]**

    当使用 Column 的 “index=True” 标志时，更改了用于生成截断的“自动”索引名称的方案。截断仅针对自动生成的名称进行，而不是用户定义的名称（会引发错误），截断方案本身现在基于标识符名称的 md5 哈希片段，以便具有类似名称的多个列上的多个索引仍然具有唯一的名称。

    参考：[#1855](https://www.sqlalchemy.org/trac/ticket/1855)

+   **[sql]**

    生成的索引名称还基于一个“最大索引名称长度”属性，该属性与“最大标识符长度”分开 - 这是为了满足 MySQL，因为 MySQL 对索引名称的最大长度为 64，与它们的总体最大长度 255 分开。

    参考：[#1412](https://www.sqlalchemy.org/trac/ticket/1412)

+   **[sql]**

    如果将 `text()` 构造放置在面向列的情况下，它将至少返回其类型的 `NULLTYPE`，而不是 `None`，使得它可以比以前更自由地用于临时列表达式。但是，`literal_column()` 仍然是更好的选择。

+   **[sql]**

    当 `ForeignKey` 无法解析目标时，在错误消息中添加了对父表/列、目标表/列的完整描述。

+   **[sql]**

    修复了一个错误，即在反射表中替换复合外键列将导致尝试第二次从表中删除反射约束，从而引发 `KeyError`。

    参考：[#1865](https://www.sqlalchemy.org/trac/ticket/1865)

+   **[sql]**

    _Label 构造，即每当你说 somecol.label() 时产生的构造，现在在其“proxy_set”中计数自身与其包含列的代理集的并集，而不是直接返回包含列的代理集。这允许依赖于 _Labels 本身身份的列对应操作返回正确的结果。

+   **[sql]**

    修复 ORM bug。

    参考：[#1852](https://www.sqlalchemy.org/trac/ticket/1852)

### postgresql

+   **[postgresql]**

    修复了 psycopg2 方言，使用其 set_isolation_level() 方法而不是依赖基本的 “SET SESSION ISOLATION” 命令，因为否则 psycopg2 在每个新事务中重置隔离级别。

### mssql

+   **[mssql]**

    修复了与 pymssql 后端一起使用 “default schema” 查询的问题。

### oracle

+   **[oracle]**

    在 Oracle 方言中添加了 ROWID 类型，用于那些可能需要显式 CAST 的情况。

    参考：[#1879](https://www.sqlalchemy.org/trac/ticket/1879)

+   **[oracle]**

    Oracle 索引反映已经调整，以便反映包含一些或全部主键列的索引，但不包含与主键相同的列集的索引。在反映中跳过包含与主键相同列的索引，因为在这种情况下假定该索引是自动生成的主键索引。以前，任何包含 PK 列的索引都会被跳过。感谢 Kent Bower 提供的补丁。

    参考：[#1867](https://www.sqlalchemy.org/trac/ticket/1867)

+   **[oracle]**

    Oracle 现在反映主键约束的名称 - 还要感谢 Kent Bower。

    参考：[#1868](https://www.sqlalchemy.org/trac/ticket/1868)

### 杂项

+   **[declarative]**

    如果 @classproperty 与常规的类绑定映射器属性属性一起使用，它将在初始化期间被调用以获取实际属性值。目前，在声明类的列或关系属性上使用 @classproperty 没有任何优势 - 评估与未使用 @classproperty 时同时进行。但至少在这里我们允许它按预期运行。

+   **[declarative]**

    修复了“无法添加额外列”消息显示错误名称的 bug。

+   **[firebird]**

    修复了一个 bug，即如果 “default” 关键字为小写，则列默认值将无法反映。

+   **[informix]**

    应用了来自的补丁，以再次使基本 Informix 功能正常运行。我们依赖最终用户测试来确保 Informix 在某种程度上正常工作。

    参考：[#1904](https://www.sqlalchemy.org/trac/ticket/1904)

+   **[documentation]**

    文档已重新组织，使“API 参考”部分消失 - 所有公共 API 的文档字符串都移动到主要文档部分的上下文中。主文档分为 “SQLAlchemy Core” 和 “SQLAlchemy ORM” 部分，映射器/关系文档已拆分出来。许多部分已被重写和/或重新组织。

## 0.6.3

发布日期：2010 年 7 月 15 日

### orm

+   **[orm]**

    移除了在单元操作中触发的多对多加载错误，该错误在过期/未加载的集合上不必要地触发。现在，只有在 passive_updates 为 False 且父主键已更改，或者 passive_deletes 为 False 且父项已删除时，才会进行此加载。

    参考：[#1845](https://www.sqlalchemy.org/trac/ticket/1845)

+   **[orm]**

    列实体（即 query(Foo.id)）在从自身派生的查询更全面地复制其状态时，以及从自身 + 可选择的查询（即 from_self()、union() 等）派生的查询，以便 join() 等操作具有正确的状态。

    参考：[#1853](https://www.sqlalchemy.org/trac/ticket/1853)

+   **[orm]**

    修复了 Query.join() 在查询非 ORM 列然后在已经存在 FROM 子句的情况下没有加入 on 子句时会失败的错误，现在会像在没有子句的情况下一样引发一个经过检查的异常。

    参考：[#1853](https://www.sqlalchemy.org/trac/ticket/1853)

+   **[orm]**

    改进了对“未映射类”的检查，包括超类已映射但子类未映射的情况。任何尝试访问 cls._sa_class_manager.mapper 现在都会引发 UnmappedClassError()。

    参考：[#1142](https://www.sqlalchemy.org/trac/ticket/1142)

+   **[orm]**

    向 Query 添加了“column_descriptions”访问器，返回一个包含有关查询将返回的实体的命名/类型信息的字典列表。对于在 ORM 查询之上构建 GUI 非常有帮助。

### mysql

+   **[mysql]**

    _extract_error_code() 方法现在可以正确地与每个 MySQL 方言（MySQL-python、OurSQL、MySQL-Connector-Python、PyODBC）一起工作。以前，重新连接逻辑会在 OperationalError 条件下失败，但由于 MySQLdb 和 OurSQL 有自己的重新连接功能，因此在这些驱动程序中没有任何症状，除非有人观看日志。

    参考：[#1848](https://www.sqlalchemy.org/trac/ticket/1848)

### oracle

+   **[oracle]**

    对 cx_oracle Decimal 处理进行了更多微调。没有小数点的“模糊”数值在连接处理程序级别被强制转换为 int。这里的优势是，int 以 int 返回，而无需涉及 SQLA 类型对象，也无需先转换为 Decimal。

    不幸的是，一些奇特的子查询情况甚至可以在单个结果行之间看到不同类型，因此当 Numeric 处理程序被指示返回 Decimal 时，无法充分利用“本机十进制”模式，必须对每个值运行 isinstance() 来检查其是否已经是 Decimal。重新打开

    参考：[#1840](https://www.sqlalchemy.org/trac/ticket/1840)

## 0.6.2

发布日期：Tue Jul 06 2010

### orm

+   **[orm]**

    Query.join() 将检查是否调用了形式为 query.join(target, clause_expression) 的调用，即缺少元组，并提出信息性错误消息，说明这是错误的调用形式。

+   **[orm]**

    修复了关于自引用双向多对多关系刷新的错误，其中在一个刷新中使两个对象相互引用的情况下，将无法为双方插入行。从 0.5 版本开始的回归。

    参考：[#1824](https://www.sqlalchemy.org/trac/ticket/1824)

+   **[orm]**

    relationship()的 post_update 功能在架构上进行了重新设计，以更紧密地与新的 0.6 工作单元集成。更改的动机是为了使多个影响同一行不同外键列的“post update”调用在单个 UPDATE 语句中执行，而不是每列每行一个 UPDATE 语句。多行更新也尽可能地批量处理到 executemany()中，同时保持一致的行顺序。

+   **[orm]**

    Query.statement、Query.subquery()等现在将查询.params()指定的绑定参数的值传输到生成的 SQL 表达式中。以前，这些值不会被传输，绑定参数会变成 None。

+   **[orm]**

    子查询预加载现在可以与包含 params()的 Query 对象一起使用，以及 get()查询。

+   **[orm]**

    现在可以在被父对象通过多对一引用的实例上调用 make_transient()，而不会导致父对象的外键值暂时设置为 None - 这是“检测主键切换”刷新处理程序的功能。它现在忽略不再处于“持久”状态的对象，父对象的外键标识符保持不受影响。

+   **[orm]**

    query.order_by()现在接受 False，取消 Query 上的任何现有 order_by()状态，允许调用后续不支持 ORDER BY 的生成方法。这与已经存在的传递 None 的功能不同，后者会抑制任何现有的 order_by()设置，包括在映射器上配置的设置。False 将使 order_by()好像从未调用过，而 None 是一个活动设置。

+   **[orm]**

    如果将移至“瞬态”状态的实例具有不完整或缺失的主键属性集，并且包含过期属性，则在访问过期属性时会引发 InvalidRequestError，而不是出现递归溢出。

+   **[orm]**

    make_transient()函数现在在生成的文档中。

+   **[orm]**

    make_transient()从被设置为瞬态的状态中删除所有“loader”可调用项，删除任何“过期”状态 - 所有未加载的属性在访问时重置为未定义、None/空。

### sql

+   **[sql]**

    当 convert_unicode=True 的 Unicode 和 String 类型发出警告时，不再嵌入传递的实际值。这样做是为了避免 Python 警告注册表继续增长，根据警告过滤器设置，警告只会发出一次，大字符串值不会污染输出。

    参考：[#1822](https://www.sqlalchemy.org/trac/ticket/1822)

+   **[sql]**

    修复了一个 bug，该 bug 会阻止“annotated”表达式元素的重写子句编译正常工作，这些表达式元素通常由 ORM 生成。

+   **[sql]**

    LIKE 运算符或类似运算符的“ESCAPE”参数通过 render_literal_value()传递，该方法可能实现反斜杠的转义。

    参考：[#1400](https://www.sqlalchemy.org/trac/ticket/1400)

+   **[sql]**

    修复了 Enum 类型的 bug，当与 TypeDecorators 或其他适配场景一起使用时会清除 native_enum 标志。

+   **[sql]**

    当调用 Inspector 时，会触发 bind.connect() 以确保已调用 initialize。内部名称“.conn”更改为“.bind”，因为那才是它的名称。

+   **[sql]**

    修改了“列注释”的内部结构，使得自定义 Column 子类可以安全地重写 _constructor 以返回 Column，用于创建不涉及代理等的“配置”列类。

+   **[sql]**

    Column.copy() 包括“unique”属性在内，修复了关于声明性混合的问题

    参考：[#1829](https://www.sqlalchemy.org/trac/ticket/1829)

### postgresql

+   **[postgresql]**

    render_literal_value() 被重写以转义反斜杠，目前适用于 LIKE 等表达式的 ESCAPE 子句。最终，这将必须检测“standard_conforming_strings”的值以获得完整行为。

    参考：[#1400](https://www.sqlalchemy.org/trac/ticket/1400)

+   **[postgresql]**

    如果在 PG 版本低于 8.3 上使用 types.Enum，则不会生成“CREATE TYPE” / “DROP TYPE” - supports_native_enum 标志将被完全遵守。

    参考：[#1836](https://www.sqlalchemy.org/trac/ticket/1836)

### mysql

+   **[mysql]**

    MySQL 方言在检测到 MySQL 版本小于 4.0.2 时不会发出 CAST()。这允许在连接时进行 unicode 检查。

    参考：[#1826](https://www.sqlalchemy.org/trac/ticket/1826)

+   **[mysql]**

    MySQL 方言现在检测到 NO_BACKSLASH_ESCAPES sql 模式，除了 ANSI_QUOTES。

+   **[mysql]**

    render_literal_value() 被重写以转义反斜杠，目前适用于 LIKE 等表达式的 ESCAPE 子句。此行为源自检测到 NO_BACKSLASH_ESCAPES 的值。

    参考：[#1400](https://www.sqlalchemy.org/trac/ticket/1400)

### mssql

+   **[mssql]**

    如果 server_version_info 超出通常范围（8, ），（9, ），（10, ），则会发出警告，建议检查 FreeTDS 版本配置是否使用 7.0 或 8.0，而不是 4.2。

    参考：[#1825](https://www.sqlalchemy.org/trac/ticket/1825)

### oracle

+   **[oracle]**

    修复了 ora-8 兼容性标志，使其不会缓存在第一次数据库连接之前的旧值。

    参考：[#1819](https://www.sqlalchemy.org/trac/ticket/1819)

+   **[oracle]**

    当 Oracle 的“本地十进制”元数据在子查询中嵌入列时以及在使用子查询进行 ROWNUM 查询时开始返回关于数值的模糊类型信息，就像我们为 limit/offset 所做的那样。我们已将这些模糊条件添加到 cx_oracle 的“转换为 Decimal()”处理程序中，以便在更多情况下将数值作为 Decimal 而不是浮点数接收。然后，如果需要，这些数值将被转换为整数或浮点数，否则将保留为无损 Decimal。

    参考：[#1840](https://www.sqlalchemy.org/trac/ticket/1840)

### misc

+   **[firebird]**

    修复了 do_execute() 中的错误签名，在 0.6.1 中引入的错误。

    参考：[#1823](https://www.sqlalchemy.org/trac/ticket/1823)

+   **[firebird]**

    Firebird 方言添加了接受“charset”标志的 CHAR、VARCHAR 类型，以支持 Firebird 的“CHARACTER SET”子句。

    参考：[#1813](https://www.sqlalchemy.org/trac/ticket/1813)

+   **[declarative]**

    添加了对 @classproperty 的支持，以从声明性 mixin 提供任何类型的模式/映射构造，包括具有外键、关系、column_property、deferred 的列。如果在 mixin 上指定了任何 MapperProperty 子类，而不使用 @classproperty，则会引发错误。

    参考：[#1751](https://www.sqlalchemy.org/trac/ticket/1751), [#1796](https://www.sqlalchemy.org/trac/ticket/1796), [#1805](https://www.sqlalchemy.org/trac/ticket/1805)

+   **[declarative]**

    一个 mixin 类现在可以定义一个与子类中定义的 __table__ 上存在的列相匹配的列。然而，它不能定义一个在 __table__ 中不存在的列，这里的错误消息现在已经可以正常工作。

    参考：[#1821](https://www.sqlalchemy.org/trac/ticket/1821)

+   **[compiler] [extension]**

    当覆盖内置子句构造的编译时，“default”编译器会自动复制过去，因此如果用户定义的编译器特定于某些后端，并且调用了不同后端的编译，就不会引发 KeyError。

    参考：[#1838](https://www.sqlalchemy.org/trac/ticket/1838)

+   **[documentation]**

    为 Inspector 添加了文档。

    参考：[#1820](https://www.sqlalchemy.org/trac/ticket/1820)

+   **[documentation]**

    修复了 @memoized_property 和 @memoized_instancemethod 装饰器，以便 Sphinx 文档能够捕捉到这些属性和方法，例如 ResultProxy.inserted_primary_key。

    参考：[#1830](https://www.sqlalchemy.org/trac/ticket/1830)

## 0.6.1

发布日期：Mon May 31 2010

### orm

+   **[orm]**

    修复了在 0.6.0 中引入的关于可变属性的不正确历史记录账务的回归。

    参考：[#1782](https://www.sqlalchemy.org/trac/ticket/1782)

+   **[orm]**

    修复了在 0.6.0 中引入的工作单元重构中破坏了带有 post_update=True 的双向 relationship() 更新的回归。

    参考：[#1807](https://www.sqlalchemy.org/trac/ticket/1807)

+   **[orm]**

    如果返回的实例是“pending”，session.merge() 将不会使返回的实例上的属性过期。

    参考：[#1789](https://www.sqlalchemy.org/trac/ticket/1789)

+   **[orm]**

    修复了 CollectionAdapter 的 __setstate__ 方法，在未反序列化父 InstanceState 的情况下不会失败。

    参考：[#1802](https://www.sqlalchemy.org/trac/ticket/1802)

+   **[orm]**

    在实例没有完整主键的情况下，添加了内部警告，如果实例已过期并且被要求刷新。

    参考：[#1797](https://www.sqlalchemy.org/trac/ticket/1797)

+   **[orm]**

    对映射器在使用 UPDATE、INSERT 和 DELETE 表达式时增加了更积极的缓存。假设语句没有附加每个对象的 SQL 表达式，表达式对象在第一次创建后会被映射器缓存，并且它们的编译形式会持久地存储在与相关引擎的持续时间相关的缓存字典中。对于极少数情况下，如果映射器接收到大量不同的列模式作为 UPDATE，缓存是一个 LRUCache。

### sql

+   **[sql]**

    expr.in_()现在接受一个 text()构造作为参数。自动添加分组括号，即使用方式类似于 col.in_(text(“select id from table”)).

    参考：[#1793](https://www.sqlalchemy.org/trac/ticket/1793)

+   **[sql]**

    _Binary 类型的列（即 LargeBinary、BLOB 等）将右侧的“basestring”强制转换为 _Binary，以便进行必要的 DBAPI 处理。

+   **[sql]**

    增加了 table.add_is_dependent_on(othertable)，允许在 create_all()、drop_all()、sorted_tables 中手动放置两个 Table 对象之间的依赖规则。

    参考：[#1801](https://www.sqlalchemy.org/trac/ticket/1801)

+   **[sql]**

    修复了一个 bug，该 bug 阻止了包含零的复合主键的隐式 RETURNING 功能正常运行。

    参考：[#1778](https://www.sqlalchemy.org/trac/ticket/1778)

+   **[sql]**

    修复了为命名的 UNIQUE 约束生成 ADD CONSTRAINT 时出现的额外空格字符。

+   **[sql]**

    修复了 ForeignKeyConstraint 构造函数中“table”参数的 bug

    参考：[#1571](https://www.sqlalchemy.org/trac/ticket/1571)

+   **[sql]**

    修复了连接池游标包装器中的 bug，即如果游标在 close()时抛出异常，则消息的记录将失败。

    参考：[#1786](https://www.sqlalchemy.org/trac/ticket/1786)

+   **[sql]**

    ColumnClause 和 Column 的 _make_proxy()方法现在使用 self.__class__ 来确定要返回的对象类，而不是硬编码为 ColumnClause/Column，这样更容易生成在别名/子查询情况下工作的特定子类。

+   **[sql]**

    func.XXX()不会意外地解析为非 Function 类（例如修复了 func.text()）。

    参考：[#1798](https://www.sqlalchemy.org/trac/ticket/1798)

### mysql

+   **[mysql]**

    func.sysdate()在 MySQL 上发出“SYSDATE()”，即带有结束括号。

    参考：[#1794](https://www.sqlalchemy.org/trac/ticket/1794)

### sqlite

+   **[sqlite]**

    修复了当“PRIMARY KEY”约束由于 SQLite AUTOINCREMENT 关键字被渲染时移动到列级别时约束的连接错误。

    参考：[#1812](https://www.sqlalchemy.org/trac/ticket/1812)

### oracle

+   **[oracle]**

    增加了对低于版本 5 的 cx_oracle 版本的检查，如果是这种情况，将不使用不兼容的“输出类型处理程序”。这将影响十进制精度和一些 Unicode 处理问题。

    参考：[#1775](https://www.sqlalchemy.org/trac/ticket/1775)

+   **[oracle]**

    修复了 use_ansi=False 模式，在几乎所有情况下都会产生错误的 WHERE 子句。

    参考：[#1790](https://www.sqlalchemy.org/trac/ticket/1790)

+   **[oracle]**

    重新支持使用 cx_oracle 的 Oracle 8，包括自动将 use_ansi 设置为 False，对于 Unicode，不会为 NVARCHAR2 和 NCLOB 渲染，“native unicode” 检查不会失败，cx_oracle 的“native unicode” 模式被禁用，VARCHAR() 以字节计数而不是字符计数发出。

    参考：[#1808](https://www.sqlalchemy.org/trac/ticket/1808)

+   **[oracle]**

    在正常的 Python 2.x 模式下，oracle_xe 5 不接受 Python Unicode 对象作为连接字符串 - 因此我们直接强制转换为 str()。由于我们不知道可以使用的编码，这里连接字符串中不支持非 ASCII 字符。

    参考：[#1670](https://www.sqlalchemy.org/trac/ticket/1670)

+   **[oracle]**

    当使用 limit/offset 时，在语法上正确的位置发出 FOR UPDATE，即 ROWNUM 子查询。但是，Oracle 实际上无法处理带有 ORDER BY 或子查询的 FOR UPDATE，因此仍然不太可用，但至少 SQLA 能够将 SQL 传递给 Oracle 解析器。

    参考：[#1815](https://www.sqlalchemy.org/trac/ticket/1815)

### 杂项

+   **[引擎]**

    修复了在 Python 2.4 上构建 C 扩展的问题。

    参考：[#1781](https://www.sqlalchemy.org/trac/ticket/1781)

+   **[引擎]**

    在 dispose() 发生后，池类将重用相同的“pool_logging_name”设置。

+   **[引擎]**

    引擎获得了一个“execution_options”参数和 update_execution_options() 方法，将应用于此引擎生成的所有连接。

+   **[firebird]**

    在 has_table() 和 has_sequence() 中使用的查询中添加了一个标签，以便与不提供结果列标签的旧版本 Firebird 一起使用。

    参考：[#1521](https://www.sqlalchemy.org/trac/ticket/1521)

+   **[firebird]**

    在通过查询字符串传递“type_conv”属性时，添加了整数强制转换，以便由 Kinterbasdb 正确解释。

    参考：[#1779](https://www.sqlalchemy.org/trac/ticket/1779)

+   **[firebird]**

    将“连接关闭”添加到异常字符串列表中，表示连接已断开。

    参考：[#1646](https://www.sqlalchemy.org/trac/ticket/1646)

+   **[sqlsoup]**

    SqlSoup 构造函数接受一个 base 参数，指定用于映射类的基类，默认为 object。

    参考：[#1783](https://www.sqlalchemy.org/trac/ticket/1783)

## 0.6.0

发布日期：Sun Apr 18 2010

### orm

+   **[orm]**

    工作单元内部已经重写。具有大量相互依赖对象的工作单元现在可以在没有递归溢出的情况下刷新，因为不再依赖递归调用。对于特定会话状态，内部结构的数量现在保持恒定，而不管映射上存在多少关系。事件流现在对应于由映射器和基于实际工作的关系生成的线性步骤列表，通过单个拓扑排序进行正确排序。刷新操作使用的步骤更少，占用更少的内存。

    参考：[#1081](https://www.sqlalchemy.org/trac/ticket/1081), [#1742](https://www.sqlalchemy.org/trac/ticket/1742)

+   **[orm]**

    随着 UOW 重写，这也解决了 0.6beta3 中关于具有长依赖循环的工作单元的拓扑循环检测问题。我们现在使用 Guido 编写的算法（感谢 Guido！）。

+   **[orm]**

    一对多关系现在在 flush 中维护一个正的父子关联列表，防止之前标记为已删除的父项在级联删除或在旧关联中未将子项从中删除的情况下设置 NULL 外键。 

    参考：[#1764](https://www.sqlalchemy.org/trac/ticket/1764)

+   **[orm]**

    集合的延迟加载将关闭反向多对一端的默认急加载，因为该加载在定义上是不必要的。

    参考：[#1495](https://www.sqlalchemy.org/trac/ticket/1495)

+   **[orm]**

    现在 Session.refresh() 首先对给定实例执行等效的 expire()，以便“刷新-过期”级联被传播。以前，refresh() 不受“刷新-过期”级联的任何影响。这是与 0.6beta2 的行为不同之处，其中传递给 refresh() 的“lockmode”标志会导致版本检查发生。由于实例首先被过期，refresh() 总是将对象升级到最新版本。

+   **[orm]**

    当“刷新-过期”级联到达待处理对象时，如果级联还包括“删除孤儿”，则会将对象删除；否则，将简单分离它。

    参考：[#1754](https://www.sqlalchemy.org/trac/ticket/1754)

+   **[orm]**

    不再在 topological.py 内部使用 id(obj)，因为排序函数现在仅需要可哈希对象。

    参考：[#1756](https://www.sqlalchemy.org/trac/ticket/1756)

+   **[orm]**

    ORM 现在默认将所有生成的描述符的文档字符串设置为 None。可以使用 'doc' 进行覆盖（或者如果使用 Sphinx，则属性文档字符串也有效）。

+   **[orm]**

    在所有映射器属性可调用以及 Column() 中添加了 kw 参数 'doc'。将字符串 'doc' 组装为描述符上的 '__doc__' 属性。

+   **[orm]**

    在支持 cursor.rowcount 用于 execute() 但不支持 executemany() 的后端上，现在在发出删除时可以使用 version_id_col（已经适用于保存，因为这些不使用 executemany()）。对于根本不支持 cursor.rowcount 的后端，与保存一样会发出警告。

    参考：[#1761](https://www.sqlalchemy.org/trac/ticket/1761)

+   **[orm]**

    ORM 现在会在刷新相同类别对象列表时短期缓存 insert() 和 update() 构造的“编译”形式，从而避免在单个 flush() 调用中每个 INSERT/UPDATE 都进行冗余编译。

+   **[orm]**

    ColumnProperty、CompositeProperty、RelationshipProperty 上的内部 getattr()、setattr()、getcommitted() 方法已经被下划线标记为私有（即私有），签名已更改。

### 示例

+   **[examples]**

    更新了 attribute_shard.py 示例，使用了更健壮的方法来搜索查询中将列与文字值进行比较的二进制表达式。

### sql

+   **[sql]**

    从 0.5 版本中恢复了一些绑定标签逻辑，确保具有与“<tablename>_<columnname>”形式重叠列名的表在 UPDATE 过程中使用 column._label 作为绑定名称时不会产生错误。增加了 0.5 版本中缺少的测试覆盖率。

    参考：[#1755](https://www.sqlalchemy.org/trac/ticket/1755)

+   **[sql]**

    somejoin.select(fold_equivalents=True) 不再被弃用，并最终将被合并到更全面的功能版本中。

    参考：[#1729](https://www.sqlalchemy.org/trac/ticket/1729)

+   **[sql]**

    Numeric 类型在期望从返回浮点数的 DBAPI 转换为 Decimal 时会引发*巨大*警告。这包括 SQLite、Sybase、MS-SQL。

    参考：[#1759](https://www.sqlalchemy.org/trac/ticket/1759)

+   **[sql]**

    修复了表达式类型错误的问题，导致具有两个 NULL 类型的表达式陷入无限循环。

+   **[sql]**

    修复了 execution_options() 功能中的错误，其中来自父���接的现有事务和其他状态信息不会传播到子连接。

+   **[sql]**

    添加了新的‘compiled_cache’执行选项。一个字典，当连接将一个子句表达式编译成特定于方言和参数的 Compiled 对象时，Compiled 对象将被缓存。用户有责任管理这个字典的大小，它将具有与方言、子句元素、INSERT 或 UPDATE 语句的 VALUES 或 SET 子句中的列名以及 INSERT 或 UPDATE 语句的“批处理”模式相对应的键。

+   **[sql]**

    在 reflection.Inspector 中添加了 get_pk_constraint() 方法，类似于 get_primary_keys()，但返回一个包含约束名称的字典，适用于支持的后端（目前仅支持 PG）。

    参考：[#1769](https://www.sqlalchemy.org/trac/ticket/1769)

+   **[sql]**

    Table.create() 和 Table.drop() 不再应用于元数据级别的创建/删除事件。

    参考：[#1771](https://www.sqlalchemy.org/trac/ticket/1771)

### postgresql

+   **[postgresql]**

    PostgreSQL 现在正确反映与 SERIAL 列关联的序列名称，之后序列名称已更改。感谢 Kumar McMillan 提供的补丁。

    参考：[#1071](https://www.sqlalchemy.org/trac/ticket/1071)

+   **[postgresql]**

    当接收到未知的数字时，修复了 psycopg2._PGNumeric 类型中缺失的导入。

+   **[postgresql]**

    psycopg2/pg8000 方言现在能够识别 REAL[]、FLOAT[]、DOUBLE_PRECISION[]、NUMERIC[] 返回类型，而不会引发异常。

+   **[postgresql]**

    如果存在主键约束，PostgreSQL 反映主键约束的名称。

    参考：[#1769](https://www.sqlalchemy.org/trac/ticket/1769)

### oracle

+   **[oracle]**

    现在使用 cx_oracle 输出转换器，以便 DBAPI 原生返回我们喜欢的值类型：

+   **[oracle]**

    具有正精度 + 小数位数的 NUMBER 值转换为 cx_oracle.STRING，然后转换为 Decimal。这允许在使用 cx_oracle 时 Numeric 类型具有完美的精度。

    参考：[#1759](https://www.sqlalchemy.org/trac/ticket/1759)

+   **[oracle]**

    STRING/FIXED_CHAR 现在原生转换为 Unicode。SQLAlchemy 的 String 类型不需要应用任何类型的转换。

### misc

+   **[engines]**

    C 扩展现在也适用于使用自定义序列作为行（而不仅仅是元组）的 DBAPI。

    参考：[#1757](https://www.sqlalchemy.org/trac/ticket/1757)

+   **[ext]**

    编译器扩展现在允许在扩展到子类的基类上使用 @compiles 装饰器，在子类上使用 @compiles 装饰器不会被基类上的 @compiles 装饰器破坏。

+   **[ext]**

    当在基于字符串的 relationship() 参数中引用非映射类属性时，Declarative 将引发一个信息性错误消息。

+   **[ext]**

    进一步重新调整了 declarative 中的“mixin”逻辑，还允许在 mixin 上作为 @classproperty 动态分配 polymorphic_identity 等参数。

+   **[firebird]**

    可以通过在 create_engine() 上设置 'enable_rowcount=False' 来在每个引擎上禁用 result.rowcount 的功能。通常，在任何 UPDATE 或 DELETE 语句之后无条件地调用 cursor.rowcount，因为然后游标被关闭，而 Firebird 需要一个打开的游标才能获取 rowcount。然而，这个调用略微昂贵，因此可以禁用。要在每次执行时重新启用，可以使用 'enable_rowcount=True' 执行选项。

## 0.6beta3

发布日期：Sun Mar 28 2010

### orm

+   **[orm]**

    主要功能：向 relationship() 添加了新的“subquery”加载功能。这是一种急加载选项，为查询中表示的每个集合生成第二个 SELECT，跨所有父级一次加载。查询重新发出原始的最终用户查询，包装在一个子查询中，应用连接到目标集合，一次完全加载所有这些集合的结果，类似于“joined”急加载，但使用所有内连接，不会重复重新获取完整的父行（即使跳过列，大多数 DBAPI 似乎也会这样做）。子查询加载在映射器配置级别使用“lazy='subquery'”和在查询选项级别使用“subqueryload(props..)”��“subqueryload_all(props…)”可用。

    参考：[#1675](https://www.sqlalchemy.org/trac/ticket/1675)

+   **[orm]**

    为了适应现在有两种可用的急加载类型的事实，eagerload() 和 eagerload_all() 的新名称分别为 joinedload() 和 joinedload_all()。旧名称将在可预见的未来保留为同义词。

+   **[orm]**

    relationship() 函数上的“lazy”标志现在接受字符串参数，用于所有加载类型：“select”、“joined”、“subquery”、“noload” 和 “dynamic”，其中默认值现在为“select”。True/False/None 的旧值仍保留其通常含义，并将在可预见的未来保留为同义词。

+   **[orm]**

    添加了 with_hint() 方法到 Query() 构造中。这直接调用 select().with_hint()，并且还接受实体以及表和别名。请参见下面 SQL 部分中的 with_hint()。

    参考：[#921](https://www.sqlalchemy.org/trac/ticket/921)

+   **[orm]**

    修复了 Query 中的一个 bug，即调用 q.join(prop).from_self(…). join(prop) 时，当在内部使用相同的条件进行连接时，第二个连接未能在子查询之外呈现。

+   **[orm]**

    修复了 Query 中的一个 bug，即在使用 aliased() 构造时，如果在 q.from_self() 或 q.select_from() 生成的子查询中引用了基础表（但实际别名未被引用），则会失败。

+   **[orm]**

    修复了一个 bug，该 bug 影响了所有 eagerload() 和类似选项，即“remote” 急加载，即从延迟加载（例如 query(A).options(eagerload(A.b, B.c))）进行急加载不会加载任何内容，但使用 eagerload(“b.c”) 将正常工作。

+   **[orm]**

    Query 增加了一个 add_columns(*columns) 方法，这是 add_column(col) 的多版本。add_column(col) 将在未来被弃用。

+   **[orm]**

    Query.join() 将检测最终结果是否为“FROM A JOIN A”，如果是，将引发错误。

+   **[orm]**

    Query.join(Cls.propname, from_joinpoint=True) 将更仔细地检查“Cls”是否与当前连接点兼容，并在这方面与 Query.join(“propname”, from_joinpoint=True) 采取相同的方式。

### sql

+   **[sql]**

    添加了 with_hint() 方法到 select() 构造中。指定表/别名、提示文本和可选的方言名称，“hints” 将在语句中的适当位置呈现。适用于 Oracle、Sybase、MySQL。

    参考：[#921](https://www.sqlalchemy.org/trac/ticket/921)

+   **[sql]**

    修复了在 0.6beta2 中引入的 bug，该 bug 会导致列标签在已分配标签的列表达式内部呈现。

    参考：[#1747](https://www.sqlalchemy.org/trac/ticket/1747)

### postgresql

+   **[postgresql]**

    psycopg2 方言将通过 “sqlalchemy.dialects.postgresql” 记录器名称记录 NOTICE 消息。

    参考：[#877](https://www.sqlalchemy.org/trac/ticket/877)

+   **[postgresql]**

    TIME 和 TIMESTAMP 类型现在直接从 postgresql 方言中可用，这两个类型都添加了 PG 特定参数 'precision'。 对于 TIME 和 TIMEZONE 类型，'precision' 和 'timezone' 也都正确反映。

    参考：[#997](https://www.sqlalchemy.org/trac/ticket/997)

### mysql

+   **[mysql]**

    不再猜测 TINYINT(1) 应该是 BOOLEAN 当进行反射时 - TINYINT(1) 被返回。在表定义中使用 Boolean/ BOOLEAN 来获取布尔转换行为。

    参考：[#1752](https://www.sqlalchemy.org/trac/ticket/1752)

### oracle

+   **[oracle]**

    Oracle 方言将使用字符计数发出 VARCHAR 类型定义，即 VARCHAR2(50 CHAR)，因此列的大小是以字符而不是字节为单位。 字符类型的列反射也将使用 ALL_TAB_COLUMNS.CHAR_LENGTH 而不是 ALL_TAB_COLUMNS.DATA_LENGTH。 当服务器版本为 9 或更高版本时，这两种行为都会生效 - 对于版本 8，则使用旧行为。

    参考：[#1744](https://www.sqlalchemy.org/trac/ticket/1744)

### 杂项

+   **[声明式]**

    如果 mixin 实现了一个不可预测的 __getattribute__()，即 Zope 接口，使用 mixin 将不会出错。

    参考：[#1746](https://www.sqlalchemy.org/trac/ticket/1746)

+   **[声明式]**

    在 mixins 上使用 @classdecorator 和类似方法来定义 __tablename__、__table_args__ 等，如果该方法引用了最终子类的属性，现在可以正常工作。

    参考：[#1749](https://www.sqlalchemy.org/trac/ticket/1749)

+   **[声明式]**

    在声明式 mixins 上不允许有带外键的关系和列，抱歉。

    参考：[#1751](https://www.sqlalchemy.org/trac/ticket/1751)

+   **[扩展]**

    sqlalchemy.orm.shard 模块现在成为扩展，即 sqlalchemy.ext.horizontal_shard。 旧的导入将带有弃用警告。

## 0.6beta2

发布日期：Sat Mar 20 2010

### ORM

+   **[ORM]**

    关系() 函数的官方名称现在是 relationship()，以消除对关系代数术语的混淆。 relation() 但是在可预见的将来仍将以相同的功能提供。

    参考：[#1740](https://www.sqlalchemy.org/trac/ticket/1740)

+   **[ORM]**

    在 Mapper 中添加了 “version_id_generator” 参数，这是一个可调用对象，给定 “version_id_col” 的当前值，返回下一个版本号。 可用于替代版本控制方案，如 uuid、时间戳。

    参考：[#1692](https://www.sqlalchemy.org/trac/ticket/1692)

+   **[ORM]**

    向 Session.refresh()添加了“lockmode”kw 参数，将字符串值传递给 Query 与 with_lockmode()中的相同值，还将对启用了 version_id_col 的映射进行版本检查。

+   **[orm]**

    修复了在连接继承情景中调用 query(A).join(A.bs).add_entity(B)会双重添加 B 作为目标并产生无效查询的 bug。

    参考：[#1188](https://www.sqlalchemy.org/trac/ticket/1188)

+   **[orm]**

    修复了 session.rollback()中的一个 bug，涉及在将“已删除”对象重新整合到会话之前未删除以前“待定”对象的问题，通常出现在自然主键中。如果它们之间存在主键冲突，删除的附加将在内部失败。现在首先清除了以前“待定”的对象。

    参考：[#1674](https://www.sqlalchemy.org/trac/ticket/1674)

+   **[orm]**

    删除了很多没人真正关心的日志记录，剩下的日志记录将响应日志级别的实时更改。不会增加显著的开销。

    参考：[#1719](https://www.sqlalchemy.org/trac/ticket/1719)

+   **[orm]**

    修复了 session.merge()中的一个 bug，该 bug 导致类似字典的集合无法合并。

+   **[orm]**

    session.merge()与明确不包括“merge”在其级联选项中的关系一起工作-目标将被完全忽略。

+   **[orm]**

    如果目标具有该属性的值，session.merge()将不会使现有目标上的现有标量属性过期，即使传入的合并对象没有该属性的值也是如此。这可以防止对现有项进行不必要的加载。但是，如果目标没有该属性，仍会将属性标记为过期，这样就可以满足某些延迟列的约定。

    参考：[#1681](https://www.sqlalchemy.org/trac/ticket/1681)

+   **[orm]**

    “allow_null_pks”标志现在称为“allow_partial_pks”，默认为 True，再次起到 0.5 中的作用。除此之外，它也在 merge()内实现，如果标志为 False，则不会为具有部分 NULL 主键的传入实例发出 SELECT。

    参考：[#1680](https://www.sqlalchemy.org/trac/ticket/1680)

+   **[orm]**

    修复了 0.6 重新制定的“多对一”优化中的一个 bug，使得对远程表上的非主键列（即针对唯一列的外键）进行更改时，将“旧”值从数据库中拉入，因为如果它在会话中，我们将需要它来进行正确的历史/反向引用账务，而在非主键列上无法从本地标识图中拉取。

    参考：[#1737](https://www.sqlalchemy.org/trac/ticket/1737)

+   **[orm]**

    修复了在单表继承关系上调用 has()或类似复杂表达式时可能发生的内部错误。

    参考：[#1731](https://www.sqlalchemy.org/trac/ticket/1731)

+   **[orm]**

    query.one()不再对查询应用 LIMIT，以确保完全计算结果中存在的所有对象标识，即使在连接可能隐藏两个或更多行的多个标识的情况下也是如此。作为奖励，由于不再修改查询，现在也可以使用从 from_statement()开始的查询调用 one()。

    参考：[#1688](https://www.sqlalchemy.org/trac/ticket/1688)

+   **[orm]**

    现在如果查询一个在标识映射中具有不同类别的标识符的对象，则 query.get()会返回 None，即在使用多态加载时。

    参考：[#1727](https://www.sqlalchemy.org/trac/ticket/1727)

+   **[orm]**

    在 query.join()中进行了重大修复，当“on”子句是 aliased()构造的属性时，但已经存在一个指向兼容目标的现有连接时，query 会正确地连接到正确的 aliased()构造，而不是粘附到现有连接的右侧。

    参考：[#1706](https://www.sqlalchemy.org/trac/ticket/1706)

+   **[orm]**

    对于不需要在所谓的“行切换”操作期间不必要地更新主键列的修复进行了轻微改进，即在添加+删除具有相同 PK 的两个对象时。

    参考：[#1362](https://www.sqlalchemy.org/trac/ticket/1362)

+   **[orm]**

    现在在属性加载或刷新操作由于对象从任何会话中分离而失败时，使用 sqlalchemy.orm.exc.DetachedInstanceError。UnboundExecutionError 特定于绑定到会话和语句的引擎。

+   **[orm]**

    在表达式上下文中调用的查询将在所有情况下呈现消除歧义的标签。请注意，这不适用于现有的.statement 和.subquery()访问器/方法，它仍然遵循默认为 False 的.with_labels()设置。

+   **[orm]**

    Query.union()在返回的语句中保留了消除歧义的标签，从而避免了由于列名冲突而导致的各种 SQL 组合错误。

    参考：[#1676](https://www.sqlalchemy.org/trac/ticket/1676)

+   **[orm]**

    修复了属性历史中的错误，无意中在映射实例上调用了 __eq__。

+   **[orm]**

    对对象加载的一些内部优化使大结果的速度提高了一点，估计约为 10-15%。对“state”内部进行了彻底的清理，减少了复杂性，数据成员，方法调用，空字典的创建。

+   **[orm]**

    对 query.delete()进行了文档澄清

    参考：[#1689](https://www.sqlalchemy.org/trac/ticket/1689)

+   **[orm]**

    修复了在 many-to-one relation()中的级联错误，当属性设置为 None 时，在 r6711 中引入（在 add()期间将删除的项目级联到会话中）。

+   **[orm]**

    现在在调用 query.order_by()或 query.distinct()之前调用 query.select_from()、query.with_polymorphic()或 query.from_statement()会引发异常，而不是悄悄地丢弃这些条件。

    参考：[#1736](https://www.sqlalchemy.org/trac/ticket/1736)

+   **[orm]**

    query.scalar() 现在如果返回多行将会引发异常。所有其他行为保持不变。

    参考：[#1735](https://www.sqlalchemy.org/trac/ticket/1735)

+   **[orm]**

    修复了一个 bug，导致“行切换”逻辑（即 INSERT 和 DELETE 被替换为 UPDATE）在使用 version_id_col 时失败。

    参考：[#1692](https://www.sqlalchemy.org/trac/ticket/1692)

### 示例

+   **[examples]**

    稍微修改了 beaker 缓存示例，为延迟加载缓存添加了一个单独的 RelationCache 选项。这个对象通过将多个潜在属性分组到一个共同的结构中，更有效地进行查找。FromCache 和 RelationCache 单独使用更简单。

### sql

+   **[sql]**

    join() 现在默认会模拟自然连接（NATURAL JOIN）。也就是说，如果左侧是一个连接，它将尝试将右侧连接到左侧最右侧的一侧，即使在左侧的其余部分有进一步的连接目标时也不会引发任何关于模糊连接条件的异常。

    参考：[#1714](https://www.sqlalchemy.org/trac/ticket/1714)

+   **[sql]**

    最常见的结果处理器转换函数已移至新的“processors”模块。鼓励方言作者在符合其需求时使用这些函数，而不是实现自定义函数。

+   **[sql]**

    SchemaType 和其子类 Boolean、Enum 现在是可序列化的，包括它们的 ddl 监听器和其他事件可调用对象。

    参考：[#1694](https://www.sqlalchemy.org/trac/ticket/1694), [#1698](https://www.sqlalchemy.org/trac/ticket/1698)

+   **[sql]**

    现在一些平台将会将某些文字值解释为非绑定参数，直接呈现到 SQL 语句中。这是为了支持一些平台（包括 MS-SQL 和 Sybase）强制执行的严格 SQL-92 规则。在这种模式下，绑定参数不允许出现在 SELECT 的列子句中，也不允许出现一些模糊的表达式如“?=?”。当启用此模式时，基础编译器将会将绑定参数呈现为内联文字，但仅限于字符串和数字值。其他类型如日期将会引发错误，除非方言子类为其定义了文字呈现函数。绑定参数必须已经包含嵌入的文字值，否则将引发错误（即不适用于直接的 bindparam(‘x’)）。方言还可以扩展绑定不被接受的领域，比如在函数的参数列表中（当使用本地 SQL 绑定时在 MS-SQL 上不起作用）。

+   **[sql]**

    向 String、Unicode 等添加了“unicode_errors”参数。行为类似于标准库的 string.decode() 函数的‘errors’关键字参数。此标志要求 convert_unicode 设置为“force” - 否则，不能保证 SQLAlchemy 处理 Unicode 转换的任务。请注意，对于已经原生返回 Unicode 对象的后端（大多数 DBAPI 都是如此），此标志会给行提取操作带来显著的性能开销。此标志应仅作为从具有不同或损坏编码的列中读取字符串的绝对最后手段使用，这仅适用于首先接受无效编码的数据库（即 MySQL，*不*是 PG、Sqlite 等）。

+   **[sql]**

    添加了数学取反运算符支持，-x。

+   **[sql]**

    FunctionElement 子类现在可以直接执行，方式与任何 func.foo() 构造一样，在传递给 execute() 时会自动应用 SELECT。

+   **[sql]**

    func.foo() 构造函数的“type”和“bind”关键字参数现在局限于“func.”构造中，并不是 FunctionElement 基类的一部分，允许“type”在自定义构造函数或类级变量中处理。

+   **[sql]**

    将 keys() 方法恢复到 ResultProxy。

+   **[sql]**

    类型/表达式系统现在更完整地确定表达式的返回类型以及将 Python 运算符适应为 SQL 运算符，基于给定表达式的完整左/右/运算符。特别是为 PostgreSQL EXTRACT 创建的日期/时间/间隔系统现在已经泛化为类型系统。以前经常发生的表达式“column + literal”强制“literal”类型与“column”相同的行为现在通常不会发生 - “literal” 的类型首先从字面量的 Python 类型派生，假设标准的本机 Python 类型 + 日期类型，然后回退到表达式另一侧的已知类型。如果“回退”类型兼容（即从 String 到 CHAR），则字面量一侧将使用该类型。TypeDecorator 类型默认覆盖此行为，无条件地强制“literal”一侧，可以通过实现 coerce_compared_value() 方法进行更改。还有一部分。

    参考：[#1647](https://www.sqlalchemy.org/trac/ticket/1647), [#1683](https://www.sqlalchemy.org/trac/ticket/1683)

+   **[sql]**

    将 sqlalchemy.sql.expressions.Executable 设为公共 API 的一部分，用于可以发送到 execute() 的任何表达式构造。FunctionElement 现在继承 Executable，以便获得 execution_options()，这些选项也传播到 execute() 中生成的 select()。Executable 又继承自 _Generative，标记任何支持 @_generative 装饰器的 ClauseElement - 这些也可能在某个时候成为编译器扩展的“公共”部分。

+   **[sql]**

    对于 - 直接与更新/插入的 SET 或 VALUES 子句生成的与列命名绑定直接冲突的最终用户定义的绑定参数名称进行了解决方案更改，会生成编译错误。这减少了调用次数，并消除了一些仍然可能发生不良名称冲突的情况。

    参考：[#1579](https://www.sqlalchemy.org/trac/ticket/1579)

+   **[sql]**

    如果 Column() 没有外键，则需要一个类型（这不是新功能）。如果 Column() 没有类型和外键，则现在会引发错误。

    参考：[#1705](https://www.sqlalchemy.org/trac/ticket/1705)

+   **[sql]**

    在将返回的浮点值强制转换为字符串时，Numeric() 类型的“scale”参数将被尊重 - 这允许在 SQLite、MySQL 上功能的准确性。

    参考：[#1717](https://www.sqlalchemy.org/trac/ticket/1717)

+   **[sql]**

    Column 的 copy() 方法现在会复制未初始化的“在表附加”事件。有助于新的声明式“mixin”功能。

### mysql

+   **[mysql]**

    修复了反射错误，即当 COLLATE 存在时，将不会反映出可空标志和服务器默认值。

    参考：[#1655](https://www.sqlalchemy.org/trac/ticket/1655)

+   **[mysql]**

    修复了对带有整数标志（如 UNSIGNED）定义的 TINYINT(1)“boolean”列的反射。

+   **[mysql]**

    进一步修复了 mysql-connector 方言的问题。

    参考：[#1668](https://www.sqlalchemy.org/trac/ticket/1668)

+   **[mysql]**

    在 InnoDB 上的 Composite PK 表中，“autoincrement” 列不是第一列将在 CREATE TABLE 中发出显式的 “KEY” 短语，从而避免错误。

    参考：[#1496](https://www.sqlalchemy.org/trac/ticket/1496)

+   **[mysql]**

    为广泛的 MySQL 关键字添加了反射/创建表格支持。

    参考：[#1634](https://www.sqlalchemy.org/trac/ticket/1634)

+   **[mysql]**

    修复了在 Windows 主机上反射表时可能出现的导入错误。

    参考：[#1580](https://www.sqlalchemy.org/trac/ticket/1580)

### sqlite

+   **[sqlite]**

    在 create_engine() 中添加了“native_datetime=True”标志。这将导致 DATE 和 TIMESTAMP 类型跳过所有绑定参数和结果行处理，假设已在连接上启用了 PARSE_DECLTYPES。请注意，这与“func.current_date()”不完全兼容，它将作为字符串返回。

    参考：[#1685](https://www.sqlalchemy.org/trac/ticket/1685)

### mssql

+   **[mssql]**

    重新建立了对 pymssql 方言的支持。

+   **[mssql]**

    对于隐式返回、反射等进行了各种修复 - 0.6 版本中的 MS-SQL 方言还不完全（但接近完善）

+   **[mssql]**

    添加了对 mxODBC 的基本支持。

    参考：[#1710](https://www.sqlalchemy.org/trac/ticket/1710)

+   **[mssql]**

    删除了 text_as_varchar 选项。

### oracle

+   **[oracle]**

    “out” 参数需要一个由 cx_oracle 支持的类型。如果找不到 cx_oracle 类型，则会引发错误。

+   **[oracle]**

    Oracle 的‘DATE’现在不执行任何结果处理，因为 Oracle 中的 DATE 类型存储完整的日期+时间对象，这就是你会得到的。请注意，通用的 types.Date 类型仍将在传入值上调用 value.date()。在反射表时，反射的类型将是‘DATE’。

+   **[Oracle]**

    增加了对 Oracle 的 WITH_UNICODE 模式的初步支持。至少这为 Python 3 中的 cx_Oracle 建立了初始支持。当在 Python 2.xx 中使用 WITH_UNICODE 模式时，会发出一个大而可怕的警告，要求用户认真考虑这种困难的操作模式的使用。

    参考：[#1670](https://www.sqlalchemy.org/trac/ticket/1670)

+   **[Oracle]**

    except_()方法现在在 Oracle 上呈现为 MINUS，这在该平台上更或多是等效的。

    参考：[#1712](https://www.sqlalchemy.org/trac/ticket/1712)

+   **[Oracle]**

    添加了对渲染和反射 TIMESTAMP WITH TIME ZONE，即 TIMESTAMP(timezone=True)的支持。

    参考：[#651](https://www.sqlalchemy.org/trac/ticket/651)

+   **[Oracle]**

    Oracle INTERVAL 类型现在可以反射。

### 杂项

+   **[py3k]**

    改进了关于 Python 3 的安装/测试设置，现在 Distribute 在 Py3k 上运行。现在包含 distribute_setup.py。请参阅 README.py3k 以获取 Python 3 的安装/测试说明。

+   **[引擎]**

    添加了一个可选的 C 扩展，通过重新实现 RowProxy 和最常见的结果处理器来加速 sql 层。实际的加速将严重依赖于您的 DBAPI 和表中使用的数据类型的混合，并且可以从 30%的改进到 200%以上。对于大查询，它还为 ORM 速度提供了适度的（~15-20%）间接改进。请注意，默认情况下*不*构建/安装它。请参阅 README 以获取安装说明。

+   **[引擎]**

    在“自动提交”场景中，在调用 DBAPI 连接上的 commit()之前，执行顺序会从游标中提取所有的 rowcount/last inserted ID 信息。这有助于 mxodbc 处理 rowcount，并且总体上可能是一个好主意。

+   **[引擎]**

    稍微放宽了日志记录，以便更频繁地调用 isEnabledFor()，这样对引擎/池的日志级别的更改将在下次连接时反映出来。这增加了一点方法调用开销。这是微不足道的，将使所有在调用 create_engine()之后配置日志记录的情况变得更加容易。

    参考：[#1719](https://www.sqlalchemy.org/trac/ticket/1719)

+   **[引擎]**

    assert_unicode 标志已被弃用。在要求对非 Unicode Python 字符串进行编码时，SQLAlchemy 将在所有情况下引发警告，以及当显式传递字节字符串给 Unicode 或 UnicodeType 类型时。String 类型对于已经接受 Python Unicode 对象的 DBAPI 不会执行任何操作。

+   **[引擎]**

    绑定参数以元组形式发送，而不是列表。一些后端驱动程序不接受绑定参数作为列表。

+   **[引擎]**

    threadlocal 引擎在 close() 时没有正确关闭连接 - 已修复。

+   **[引擎]**

    如果事务对象不是“活动”的话，就不会回滚或提交，允许更准确地嵌套 begin/rollback/commit。

+   **[引擎]**

    Python unicode 对象作为绑定结果会产生 Unicode 类型，而不是字符串，从而消除了在不支持 unicode 绑定的驱动程序上的某类 unicode 错误。

+   **[引擎]**

    在 create_engine()、Pool() 构造函数以及 create_engine() 中添加了“logging_name”参数，该参数会传递到 Pool 中的“pool_logging_name”参数。在日志消息的“name”字段中发出给定的字符串名称，而不是默认的十六进制标识符字符串。

    参考：[#1555](https://www.sqlalchemy.org/trac/ticket/1555)

+   **[引擎]**

    Dialect 的 visit_pool() 方法被移除，并替换为 on_connect()。该方法返回一个可调用对象，在每次创建原始 DBAPI 连接后接收该连接。如果非 None，则该可调用对象会被连接策略组装成一个 first_connect/connect 池监听器。为方言提供了更简单的接口。

+   **[引擎]**

    StaticPool 现在在不打开新连接的情况下初始化、释放和重新创建 - 只有在首次请求时才会打开连接。dispose() 现在也适用于 AssertionPool。

    参考：[#1728](https://www.sqlalchemy.org/trac/ticket/1728)

+   **[元数据] [票号: 1673]**

    添加了在使用“tometadata”时去除模式信息的功能，方法是通过传递“schema=None”作为参数。如果未指定模式，则保留表的模式。

+   **[声明性]**

    DeclarativeMeta 专门使用 cls.__dict__（而不是 dict_）作为类信息的来源；_as_declarative 专门使用传递给它的 dict_ 作为类信息的来源（当使用 DeclarativeMeta 时，这是 cls.__dict__）。理论上，这应该使得自定义元类更容易修改传递给 _as_declarative 的状态。

+   **[声明性]**

    现在 declarative 直接接受 mixin 类，作为在所有子类上提供常见功能和基于列的元素的手段，以及传播一组固定的 __table_args__ 或 __mapper_args__ 到子类的手段。对于从继承的 mixin 到本地的 __table_args__/__mapper_args__ 的自定义组合，现在可以使用描述符。新的详细信息都在声明性文档中。感谢 Chris Withers 在这方面对我的痛苦的包容。

    参考：[#1707](https://www.sqlalchemy.org/trac/ticket/1707)

+   **[声明性]**

    当传播到子类时，__mapper_args__ 字典会被复制，并直接从类 __dict__ 中取出，以避免从父类传播。映射器继承已经传播了你从父映射器中想要的东西。

    参考：[#1393](https://www.sqlalchemy.org/trac/ticket/1393)

+   **[声明性]**

    当单表子类指定已经存在于基类上的列时，会引发异常。

    参考：[#1732](https://www.sqlalchemy.org/trac/ticket/1732)

+   **[sybase]**

    实现了一个初步可用的 Sybase 方言，包括对 Python-Sybase 和 Pyodbc 的子实现。处理表的创建/删除和基本的往返功能。尚未包括反射或全面支持 unicode/特殊表达式等。

+   **[documentation]**

    在文档中进行了大量清理工作，将类、函数和方法名称链接到 API 文档中。

    参考：[#1700](https://www.sqlalchemy.org/trac/ticket/1700)

## 0.6beta1

发布日期：2010 年 2 月 3 日 星期三

### orm

+   **[orm]**

    对 query.update()和 query.delete()的更改：

    +   query.update()上的‘expire’选项已更名为‘fetch’，与 query.delete()的匹配。‘expire’已被弃用并发出警告。

    +   query.update()和 query.delete()默认都使用‘evaluate’作为同步策略。

    +   update()和 delete()的‘synchronize’策略在失败时会引发错误。没有隐式回退到“fetch”。评估的失败基于条件的结构，因此成功/失败是基于代码结构确定性的。

+   **[orm]**

    多对一关系的增强：

    +   多对一关系现在在较少的情况下触发延迟加载，包括在大多数情况下，当替换新值时不会获取“旧”值。

    +   多对一关系到一个联接表子类现在使用 get()进行简单加载（称为“use_get”条件），即 Related->Sub(Base)，无需重新定义基表的主连接条件。

    +   使用声明性列指定外键，即 ForeignKey(MyRelatedClass.id)，不会破坏“use_get”条件的发生。

    +   relation()、eagerload()和 eagerload_all()现在具有一个名为“innerjoin”的选项。指定 True 或 False 来控制是否将急加载构造为 INNER 或 OUTER 连接。默认始终为 False。映射器选项将覆盖 relation()上指定的任何设置。通常应该为多对一、非空外键关系设置以允许改进的连接性能。

    +   eagerloading 的行为，当 LIMIT/OFFSET 存在时，主查询现在在大多数情况下被包装在子查询中，现在对于所有急加载都是多对一连接的情况做了一个例外。在这些情况下，急加载直接针对父表进行，同时具有 limit/offset，而不会增加子查询的额外开销，因为多对一连接不会向结果添加行。

    参考：[#1186](https://www.sqlalchemy.org/trac/ticket/1186), [#1492](https://www.sqlalchemy.org/trac/ticket/1492), [#1544](https://www.sqlalchemy.org/trac/ticket/1544)

+   **[orm]**

    Session.merge()的增强/更改：

+   **[orm]**

    Session.merge()上的“dont_load=True”标志已被弃用，现在是“load=False”。

+   **[orm]**

    Session.merge()经过性能优化，在“load=False”模式下的调用次数减少了一半，与 0.5 相比，在“load=True”模式下对于集合的 SQL 查询显著减少。

+   **[orm]**

    如果给定的实例与已经存在的实例相同，merge()将不会发出不必要的属性合并。

+   **[orm]**

    merge()现在还会合并与给定状态相关联的“options”，即通过 query.options()传递的选项，这些选项随实例一起传递，例如选择各种属性进行 eager-或 lazy-加载的选项。这对于构建高度集成的缓存方案至关重要。这与 0.5 相比是一个细微的行为变化。

+   **[orm]**

    修复了有关“loader path”序列化的错误，该错误存在于实例状态中，并且在使用 merge()与序列化状态和应保留的关联选项的组合时也是必要的。

+   **[orm]**

    全新的 merge()在一个全面的新示例中展示了如何将 Beaker 与 SQLAlchemy 集成。请参阅下面的“示例”注意事项中的说明。

+   **[orm]**

    在联接表继承对象上现在可以更改主键值，并且在刷新时将考虑 ON UPDATE CASCADE。当使用 SQLite 或 MySQL/MyISAM 时，在 mapper()上设置新的“passive_updates”标志为 False。

    参考：[#1362](https://www.sqlalchemy.org/trac/ticket/1362)

+   **[orm]**

    flush()现在会检测到通过其他主键的 ON UPDATE CASCADE 操作更新主键列时，然后可以找到一个子后续 UPDATE 的新 PK 值的行。当关系()用于建立关系以及 passive_updates=True 时，会发生这种情况。

    参考：[#1671](https://www.sqlalchemy.org/trac/ticket/1671)

+   **[orm]**

    “save-update”级联现在将挂起的*removed*值从标量或集合属性级联到新会话中，以在 add()操作期间删除或修改这些断开连接的项目的行。

+   **[orm]**

    使用“dynamic”加载器和“secondary”表现在产生一个查询，其中“secondary”表不被别名。这允许在关系()的“order_by”属性中使用次要 Table 对象，并且还允许在动态关系的过滤条件中使用它。

    参考：[#1531](https://www.sqlalchemy.org/trac/ticket/1531)

+   **[orm]**

    当 eager 或 lazy 加载找到一行中超过一个有效值时，relation()与 uselist=False 将发出警告。这可能是由于 primaryjoin/secondaryjoin 条件不适合 eager LEFT OUTER JOIN 或其他条件所致。

    参考：[#1643](https://www.sqlalchemy.org/trac/ticket/1643)

+   **[orm]**

    当 synonym()与 map_column=True 一起使用时，会显式检查一个属性(延迟加载或其他)是否在分开发送到具有相同键名的 mapper 的属性字典中存在。而不是默默地替换现有属性(和可能存在的属性选项)，会引发错误。

    参考：[#1633](https://www.sqlalchemy.org/trac/ticket/1633)

+   **[orm]**

    “动态”加载器在构建时设置其查询条件，以便从非克隆访问器（如“statement”）返回实际查询。

+   **[orm]**

    当迭代一个 Query() 时返回的“命名元组”对象现在是可 pickle 的。

+   **[orm]**

    现在，将映射到 select()构造的需要确保您对其进行明确的别名(alias())。这样可以消除诸如此类的混淆问题，例如

    参考：[#1542](https://www.sqlalchemy.org/trac/ticket/1542)

+   **[orm]**

    query.join() 已重新设计以提供更一致的行为和更灵活的方式（包括）

    参考：[#1537](https://www.sqlalchemy.org/trac/ticket/1537)

+   **[orm]**

    query.select_from() 接受多个子句以在 FROM 子句中生成多个以逗号分隔的条目。在从多个 join() 子句中选择时很有用。

+   **[orm]**

    query.select_from() 也接受映射类、aliased() 构造和 mappers 作为参数。特别是在从多个连接表类中查询时，这有助于确保完整的连接得到呈现。

+   **[orm]**

    可以使用 query.get() 与映射到外连接的映射，其中一个或多个主键值为 None。

    参考：[#1135](https://www.sqlalchemy.org/trac/ticket/1135)

+   **[orm]**

    query.from_self()、query.union() 等执行“SELECT * from (SELECT…)”类型嵌套的操作现在更好地将子查询中的列表达式转换为外部查询的列子句。这可能与 0.5 不兼容，因为这可能会破坏没有应用标签的文字表达式的查询（即 literal(‘foo’) 等）。

    参考：[#1568](https://www.sqlalchemy.org/trac/ticket/1568)

+   **[orm]**

    relation primaryjoin 和 secondaryjoin 现在检查它们是否是列表达式，而不仅仅是子句元素。这禁止了像 FROM 表达式直接放在那里的事情。

    参考：[#1622](https://www.sqlalchemy.org/trac/ticket/1622)

+   **[orm]**

    在查询.filter()、filter_by() 等中比较对象/集合引用属性时，expression.null() 完全像 None 一样被理解。

    参考：[#1415](https://www.sqlalchemy.org/trac/ticket/1415)

+   **[orm]**

    添加了“make_transient()”辅助函数，该函数将持久化/分离实例转换为瞬时实例（即删除实例键并从任何会话中删除）。

    参考：[#1052](https://www.sqlalchemy.org/trac/ticket/1052)

+   **[orm]**

    mapper() 上的 allow_null_pks 标志已弃用，并且该功能默认处于“on”状态。这意味着对于任何主键列的值非空的行将被视为标识。仅当映射到外连接时才会出现此场景的需求。

    参考：[#1339](https://www.sqlalchemy.org/trac/ticket/1339)

+   **[orm]**

    “backref”的机制已完全合并到更细粒度的 “back_populates” 系统中，并且完全在 RelationProperty 的 _generate_backref() 方法中进行。这使得 RelationProperty 的初始化过程更简单，并且允许更容易地传播设置（例如从 RelationProperty 的子类）到反向引用中。内部的 BackRef() 已经消失，backref() 返回一个普通的元组，RelationProperty 可以理解。

+   **[orm]**

    mapper() 上的 version_id_col 功能在与不适当支持 “rowcount” 的方言一起使用时会引发警告。

    参考：[#1569](https://www.sqlalchemy.org/trac/ticket/1569)

+   **[orm]**

    Query 上添加了 “execution_options()”，以便可以将选项传递给生成的语句。目前只有 Select 语句具有这些选项，使用的唯一选项是 “stream_results”，而且唯一知道 “stream_results” 的方言是 psycopg2。

+   **[orm]**

    Query.yield_per() 将自动设置 “stream_results” 语句选项。

+   **[orm]**

    弃用或移除：

    +   mapper() 上的 ‘allow_null_pks’ 标志被弃用。它现在不起作用，而且在所有情况下都是 “on”。

    +   sessionmaker() 和其他地方的 ‘transactional’ 标志已被移除。使用 ‘autocommit=True’ 表示 ‘transactional=False’。

    +   mapper() 上的 ‘polymorphic_fetch’ 参数已移除。可以使用 ‘with_polymorphic’ 选项来控制加载。

    +   mapper() 上的 ‘select_table’ 参数已移除。对于此功能，请使用 ‘with_polymorphic=(“*”, <some selectable>)’。

    +   synonym() 上的 ‘proxy’ 参数已被移除。在 0.5 版本中，此标志在整个期间都没有起作用，因为 “proxy 生成” 行为现在是自动的。

    +   将单个元素列表传递给 eagerload()、eagerload_all()、contains_eager()、lazyload()、defer() 和 undefer()，而不是多个位置参数 *args 已被弃用。

    +   将单个元素列表传递给 query.order_by()、query.group_by()、query.join() 或 query.outerjoin()，而不是多个位置参数 *args 已被弃用。

    +   query.iterate_instances() 被移除。使用 query.instances()。

    +   Query.query_from_parent() 被移除了。使用 sqlalchemy.orm.with_parent() 函数生成“parent”子句，或者使用 query.with_parent()。

    +   query._from_self() 已移除，改用 query.from_self()。

    +   composite() 的 “comparator” 参数已被移除。使用 “comparator_factory”。

    +   RelationProperty._get_join() 已被移除。

    +   Session 上的 ‘echo_uow’ 标志已被移除。在 “sqlalchemy.orm.unitofwork” 名称上记录日志。

    +   session.clear() 已被移除。使用 session.expunge_all()。

    +   session.save()、session.update()、session.save_or_update() 被移除。使用 session.add() 和 session.add_all()。

    +   session.flush() 上的 “objects” 标志仍然被弃用。

    +   session.merge() 上的 “dont_load=True” 标志已被弃用，改用 “load=False”。

    +   ScopedSession.mapper 仍然被弃用。请参阅 [`www.sqlalchemy.org/trac/wiki/UsageRecipes/SessionAwareMapper`](https://www.sqlalchemy.org/trac/wiki/UsageRecipes/SessionAwareMapper) 上的用法示例。

    +   将一个 InstanceState（内部 SQLAlchemy 状态对象）传递给 attributes.init_collection()或 attributes.get_history()已被弃用。这些函数是公共 API，并且通常希望接受正常映射的对象实例。

    +   declarative_base()的‘engine’参数已删除。请使用‘bind’关键字参数。

### sql

+   **[sql]**

    选中（select()）和文本（text()）上的“autocommit”标志以及 select().autocommit()都已被弃用 - 现在在这些结构的任意一个上调用.execution_options(autocommit=True)，也可以直接在 Connection 和 orm.Query 上使用。

+   **[sql]**

    column 上的 autoincrement 标志现在指示应该链接到 cursor.lastrowid 的列，如果使用了该方法。有关详细信息，请参阅 API 文档。

+   **[sql]**

    现在，executemany()要求所有绑定的参数集中的所有键都必须与第一个绑定的参数集中存在的键相同。插入/更新语句的结构和行为在很大程度上由第一个参数集确定，包括哪些默认值将触发，而对其余的参数不进行最小量的猜测，以确保不影响性能。因此，否则默认值会对缺少的参数“失败”，因此现在进行了保护。

    参考：[#1566](https://www.sqlalchemy.org/trac/ticket/1566)

+   **[sql]**

    returning()支持已经内置到 insert()、update()、delete()。对于 PostgreSQL、Firebird、MSSQL 和 Oracle，具有不同功能级别的实现存在。可以显式调用 returning()，并提供列表达式，然后这些表达式将在结果集中返回，通常通过 fetchone()或 first()。

    如果正在使用的数据库版本支持（执行版本号检查），则 insert()构造还将隐式使用 RETURNING 来获取新生成的主键值。如果没有指定最终用户的 returning()，则会发生这种情况。

+   **[sql]**

    union()、intersect()、except()和其他“复合”类型的语句在括号使用上具有更一致的行为。现在，嵌入到另一个元素中的每个复合元素都将与括号分组 - 以前，列表中的第一个复合元素不会被分组，因为 SQLite 不喜欢以括号开始的语句。但是，特别是 PostgreSQL 具有关于 INTERSECT 的优先规则，并且将括号应用于所有子元素更一致。因此，现在，SQLite 的解决方法也是以前 PG 的解决方法 - 在嵌套复合元素时，通常需要在其上调用“.alias().select()”以将其包装在子查询中。

    参考：[#1665](https://www.sqlalchemy.org/trac/ticket/1665)

+   **[sql]**

    insert()和 update()构造现在可以使用与列键匹配的名称嵌入 bindparam()对象。这些绑定参数将绕过生成的 SQL 的 VALUES 或 SET 子句中出现的这些键的通常路线。

    参考：[#1579](https://www.sqlalchemy.org/trac/ticket/1579)

+   **[sql]**

    Binary 类型现在将数据作为 Python 字符串返回（在 Python 3 中为“bytes”类型），而不是内置的“buffer”类型。这允许二进制数据的对称往返。

    参考：[#1524](https://www.sqlalchemy.org/trac/ticket/1524)

+   **[SQL]**

    添加了 tuple_() 构造，允许将一组表达式与另一组表达式进行比较，通常与复合主键或类似的 IN 进行比较。也接受具有多列的 IN。已移除了“标量选择只能有一列”错误消息 - 将依赖数据库报告列不匹配的问题。

+   **[SQL]**

    用户定义的“default”和“onupdate”可调用项现在应调用“context.current_parameters”来获取当前正在处理的绑定参数字典。无论是单次执行还是 executemany-style 语句执行，此字典都以相同的方式可用。

+   **[SQL]**

    多部分模式名称，例如带有点的名称，如“dbo.master”，现在在 select() 标签中以下划线形式呈现，例如“dbo_master_table_column”。这是一个“友好”的标签，在结果集中的行为更好。

    参考：[#1428](https://www.sqlalchemy.org/trac/ticket/1428)

+   **[SQL]**

    删除了不必要的“计数器”行为，即使选择() 标签名称与表中的列名匹配，例如为“id”生成“tablename_id”，而不是在尝试避免命名冲突时为“tablename_id_1”生成“tablename_id”，当表实际上具有一个名为“tablename_id”的列时 - 这是因为标签逻辑总是应用于所有列，因此永远不会发生命名冲突。

+   **[SQL]**

    调用 expr.in_([])，即使用空列表，会在发出通常的“expr != expr”子句之前发出警告。 “expr != expr”可能非常昂贵，建议用户如果列表为空，则不要发出 in_()，而是简单地不查询，或根据更复杂的情况修改条件。

    参考：[#1628](https://www.sqlalchemy.org/trac/ticket/1628)

+   **[SQL]**

    在 select()/text() 中添加了“execution_options()”，它设置了连接的默认选项。请参阅“engines”中的说明。

+   **[SQL]**

    已弃用或移除：

    +   在 select() 上的“scalar”标志已被移除，请使用 select.as_scalar()。

    +   在 bindparam() 上的“shortname”属性已被移除。

    +   在 insert()、update()、delete() 上的 postgres_returning、firebird_returning 标志已被弃用，请使用新的 returning() 方法。

    +   在 join 上的 fold_equivalents 标志已被弃用（将保留直到实现）。

    参考：[#1131](https://www.sqlalchemy.org/trac/ticket/1131)

### 模式

+   **[模式]**

    MetaData 的 __contains__() 方法现在接受字符串或表对象作为参数。如果给定了一个表，参数首先被转换为 table.key，即“[模式名.]<表名>”。

    参考：[#1541](https://www.sqlalchemy.org/trac/ticket/1541)

+   **[模式]**

    已移除了已弃用的 MetaData.connect() 和 ThreadLocalMetaData.connect() - 将“bind”属性发送到绑定元数据。

+   **[模式]**

    删除了弃用的 metadata.table_iterator() 方法（使用 sorted_tables）

+   **[schema]**

    弃用的 PassiveDefault - 使用 DefaultClause。

+   **[schema]**

    “metadata”参数已从 DefaultGenerator 和其子类中删除，但仍然在 Sequence 上本地存在，Sequence 是 DDL 中的独立构造。

+   **[schema]**

    从索引和约束对象中删除了公共的可变性：

    > +   ForeignKeyConstraint.append_element()
    > +   
    > +   Index.append_column()
    > +   
    > +   UniqueConstraint.append_column()
    > +   
    > +   PrimaryKeyConstraint.add()
    > +   
    > +   PrimaryKeyConstraint.remove()

    这些应该被声明式构建（即一次构建）。

+   **[schema]**

    Sequence 上的“start”和“increment”属性现在默认生成 Oracle 和 PostgreSQL 上的“START WITH”和“INCREMENT BY”。 Firebird 目前不支持这些关键字。

    引用：[#1545](https://www.sqlalchemy.org/trac/ticket/1545)

+   **[schema]**

    UniqueConstraint、Index、PrimaryKeyConstraint 接受列名或列对象列表作为参数。

+   **[schema]**

    其他已移除的东西：

    +   Table.key（不知道这是干什么的）

    +   Table.primary_key 不可分配 - 使用 table.append_constraint(PrimaryKeyConstraint(…))。

    +   Column.bind（通过 column.table.bind 获得）

    +   Column.metadata（通过 column.table.metadata 获得）

    +   Column.sequence（使用 column.default）

    +   ForeignKey（constraint=some_parent）（现在是私有 _constraint）

+   **[schema]**

    ForeignKey 上的 use_alter 标志现在是手工构建使用 DDL() 事件系统的操作的快捷选项。此重构的副作用是，具有 use_alter=True 的 ForeignKeyConstraint 对象将不会在 SQLite 上发出，SQLite 不支持外键的 ALTER。

+   **[schema]**

    ForeignKey 和 ForeignKeyConstraint 对象现在正确地复制了它们所有的公共关键字参数。

    引用：[#1605](https://www.sqlalchemy.org/trac/ticket/1605)

### postgresql

+   **[postgresql]**

    新方言：pg8000、zxjdbc 和 py3k 上的 pypostgresql。

+   **[postgresql]**

    “postgres”方言现在被命名为“postgresql”！连接字符串如下：

    > > postgresql://scott:tiger@localhost/test postgresql+pg8000://scott:tiger@localhost/test
    > > 
    > “postgres”名称以以下方式保留向后兼容性：
    > 
    > > +   存在一个“postgres.py”虚拟方言，允许旧的 URL 正常工作，即 postgres://scott:tiger@localhost/test
    > > +   
    > > +   可以从旧的“databases”模块中导入“postgres”名称，即“from sqlalchemy.databases import postgres”，以及“dialects”，“from sqlalchemy.dialects.postgres import base as pg”，将发送一个弃用警告。
    > > +   
    > > +   现在特殊的表达式参数被命名为“postgresql_returning”和“postgresql_where”，但是旧的“postgres_returning”和“postgres_where”名称仍然与弃用警告一起工作。

+   **[postgresql]**

    “postgresql_where”现在接受 SQL 表达式，这些表达式也可以包含文字，需要时将进行引用。

+   **[postgresql]**

    psycopg2 方言现在在所有新连接上使用 psycopg2 的“unicode extension”，这允许所有 String/Text/等类型跳过将字节字符串后处理为 unicode 的步骤（这是一个昂贵的步骤，因为其量大）。其他本地返回 unicode 的方言（如 pg8000，zxjdbc）也跳过了 unicode 后处理。

+   **[postgresql]**

    添加了新的 ENUM 类型，它作为一个模式级别的构造存在，并扩展了通用的 Enum 类型。自动将自己与表及其父元数据关联起来，以在需要时发出适当的 CREATE TYPE/DROP TYPE 命令，支持 unicode 标签，支持反射。

    参考：[#1511](https://www.sqlalchemy.org/trac/ticket/1511)

+   **[postgresql]**

    INTERVAL 支持一个可选的“precision”参数，对应 PG 接受的参数。

+   **[postgresql]**

    使用新的 dialect.initialize() 功能来设置版本相关的行为。

+   **[postgresql]**

    对表/列名称中的%符号提供了更好的支持；然而 psycopg2 无法处理绑定参数名为 %(foobar)s 的情况，SQLA 不想添加额外开销来处理那个不存在的用例。

    参考：[#1279](https://www.sqlalchemy.org/trac/ticket/1279)

+   **[postgresql]**

    插入 NULL 到主键 + 外键列将引发“非空约束”错误，而不是尝试执行不存在的“col_id_seq”序列。

    参考：[#1516](https://www.sqlalchemy.org/trac/ticket/1516)

+   **[postgresql]**

    自增 SELECT 语句，即从修改行的过程中选择的语句，现在可以与服务器端游标模式一起工作（对于这样的语句不使用命名游标）。

+   **[postgresql]**

    postgresql 方言现在可以正确检测 pg 的“devel”版本字符串，即“8.5devel”。

    参考：[#1636](https://www.sqlalchemy.org/trac/ticket/1636)

+   **[postgresql]**

    psycopg2 现在支持语句选项“stream_results”。此选项将覆盖连接设置“server_side_cursors”。如果为 true，则将使用服务器端游标执行语句。如果为 false，则不会使用，即使连接上的“server_side_cursors”为 true。

    参考：[#1619](https://www.sqlalchemy.org/trac/ticket/1619)

### mysql

+   **[mysql]**

    新的方言：oursql，一个新的本地方言，MySQL Connector/Python，MySQLdb 的一个本地 Python 移植，当然还有 Jython 上的 zxjdbc。

+   **[mysql]**

    VARCHAR/NVARCHAR 在没有长度的情况下不会渲染，在传递到 MySQL 之前会引发错误。在 CAST 中没有影响，因为在 MySQL CAST 中不允许 VARCHAR，方言在这些情况下会渲染 CHAR/NCHAR。

+   **[mysql]**

    所有的 _detect_XXX() 函数现在都在 dialect.initialize() 下运行一次。

+   **[mysql]**

    对表/列名称中的%符号提供了更好的支持；当使用 executemany() 时，MySQLdb 无法处理 SQL 中的%符号，而 SQLA 不想添加额外开销来处理那个不存在的用例。

    参考：[#1279](https://www.sqlalchemy.org/trac/ticket/1279)

+   **[mysql]**

    BINARY 和 MSBinary 类型现在在所有情况下生成“BINARY”。省略“length”参数将生成没有长度的“BINARY”。使用 BLOB 生成一个无长度的二进制列。

+   **[mysql]**

    “quoting=’quoted’”参数对 MSEnum/ENUM 已经不推荐使用。最好依赖自动引用。

+   **[mysql]**

    ENUM 现在是新的通用 Enum 类型的子类，并且如果给定的标签名称是 Unicode 对象，则隐式处理 Unicode 值。

+   **[mysql]**

    TIMESTAMP 类型的列现在默认为 NULL，如果未传递“nullable=False”给 Column()，并且没有默认值。这现在与所有其他类型一致，并且在 TIMESTAMP 的情况下明确渲染为“NULL”，因为 MySQL 对 TIMESTAMP 列的默认可空性进行了“切换”。

    参考：[#1539](https://www.sqlalchemy.org/trac/ticket/1539)

### sqlite

+   **[sqlite]**

    DATE、TIME 和 DATETIME 类型现在可以接受可选的 storage_format 和 regexp 参数。storage_format 可用于使用自定义字符串格式存储这些类型。regexp 允许使用自定义正则表达式匹配数据库中的字符串值。

+   **[sqlite]**

    Time 和 DateTime 类型现在默认使用更严格的正则表达式来匹配数据库中的字符串。如果使用存储在旧格式中的数据，请使用 regexp 参数。

+   **[sqlite]**

    SQLite Time 和 DateTime 类型上的 __legacy_microseconds__ 不再受支持。您应该使用 storage_format 参数。

+   **[sqlite]**

    Date、Time 和 DateTime 类型现在在接受绑定参数时更严格：Date 类型只接受日期对象（和日期时间对象，因为它们继承自日期），Time 只接受时间对象，DateTime���接受日期和日期时间对象。

+   **[sqlite]**

    Table()支持一个关键字参数“sqlite_autoincrement”，在生成 DDL 时将 SQLite 关键字“AUTOINCREMENT”应用于单个整数主键列。将阻止生成单独的 PRIMARY KEY 约束。

    参考：[#1016](https://www.sqlalchemy.org/trac/ticket/1016)

### mssql

+   **[mssql]**

    MSSQL + Pyodbc + FreeTDS 现在大部分情况下可以正常工作，可能会有关于二进制数据以及 Unicode 模式标识符的异常情况。

+   **[mssql]**

    “has_window_funcs”标志已被移除。LIMIT/OFFSET 使用将始终使用 ROW NUMBER，如果在较旧版本的 SQL Server 上，则操作将失败。行为完全相同，只是错误由 SQL 服务器而不是方言引发，并且不需要设置标志来启用它。

+   **[mssql]**

    “auto_identity_insert”标志已被移除。当 INSERT 语句覆盖已知具有序列的列时，此功能始终生效。与“has_window_funcs”一样，如果底层驱动程序不支持此功能，则无论如何都无法执行此操作，因此没有必要设置标志。

+   **[mssql]**

    使用新的 dialect.initialize()功能来设置版本相关的行为。

+   **[mssql]**

    删除了不再使用的序列引用。在 mssql 中，隐式标识与其他方言上的隐式序列相同。通过使用“default=Sequence()”启用显式序列。有关更多信息，请参阅 MSSQL 方言文档。

### oracle

+   **[oracle]**

    单元测试与 cx_oracle 完全通过！

+   **[oracle]**

    支持 cx_Oracle 的“本地 unicode”模式，不需要设置 NLS_LANG��请使用最新的 cx_oracle 5.0.2 或更高版本。

+   **[oracle]**

    添加了 NCLOB 类型到基本类型。

+   **[oracle]**

    use_ansi=False 不会泄漏到选择子查询的 FROM/WHERE 子句中，该子查询还使用 JOIN/OUTERJOIN。

+   **[oracle]**

    向方言添加了本机 INTERVAL 类型。目前仅支持 DAY TO SECOND 间隔类型，因为 cx_oracle 不支持 YEAR TO MONTH。

    参考：[#1467](https://www.sqlalchemy.org/trac/ticket/1467)

+   **[oracle]**

    使用 CHAR 类型会导致 cx_oracle 的 FIXED_CHAR dbapi 类型绑定到语句。

+   **[oracle]**

    Oracle 方言现在具有 NUMBER，旨在像 Oracle 的 NUMBER 类型一样运行。它是通过表反射返回的主要数值类型，并尝试根据精度/比例参数返回 Decimal()/float/int。

    参考：[#885](https://www.sqlalchemy.org/trac/ticket/885)

+   **[oracle]**

    func.char_length 是用于 LENGTH 的通用函数

+   **[oracle]**

    包含 onupdate=<value>的 ForeignKey()将发出警告，不会发出不受 Oracle 支持的 ON UPDATE CASCADE

+   **[oracle]**

    RowProxy()的 keys()方法现在返回结果列名*规范化*为 SQLAlchemy 不区分大小写的名称。这意味着对于不区分大小写的名称，它们将是小写，而 DBAPI 通常会将它们返回为大写名称。这使得行键()与进一步的 SQLAlchemy 操作兼容。

+   **[oracle]**

    使用新的 dialect.initialize()功能设置版本相关行为。

+   **[oracle]**

    在 Oracle 中使用 types.BigInteger 将生成 NUMBER(19)

    参考：[#1125](https://www.sqlalchemy.org/trac/ticket/1125)

+   **[oracle]**

    ”区分大小写”功能将在反射期间检测到全小写的区分大小写列名，并向生成的 Column 添加“quote=True”，以便保持适当的引用。

### 杂项

+   **[major] [release]**

    有关功能描述的完整集合，请参阅[`docs.sqlalchemy.org/en/latest/changelog/migration_06.html`](https://docs.sqlalchemy.org/en/latest/changelog/migration_06.html)。此文档正在进行中。

+   **[major] [release]**

    所有最新 0.5 版本及以下的错误修复和功能增强也包含在 0.6 中。

+   **[major] [release]**

    现在针对的平台包括 Python 2.4/2.5/2.6，Python 3.1，Jython2.5。

+   **[engines]**

    可以使用 create_engine(… isolation_level=”…”)指定事务隔离级别；适用于 postgresql 和 sqlite。

    参考：[#443](https://www.sqlalchemy.org/trac/ticket/443)

+   **[engines]**

    Connection 具有 execution_options()，生成方法接受影响语句在 DBAPI 方面执行方式的关键字。当前支持“stream_results”，导致 psycopg2 使用服务器端游标执行该语句，以及“autocommit”，这是 select() 和 text() 中“autocommit”选项的新位置。select() 和 text() 也有 .execution_options()，以及 ORM Query()。

+   **[引擎]**

    修复了 entrypoint 驱动的方言导入，不再依赖愚蠢的 tb_info 技巧来确定导入错误状态。

    参考：[#1630](https://www.sqlalchemy.org/trac/ticket/1630)

+   **[引擎]**

    为 ResultProxy 添加了 first() 方法，返回第一行并立即关闭结果集。

+   **[引擎]**

    RowProxy 对象现在可以被 pickle 化，即 result.fetchone()、result.fetchall() 等返回的对象。

+   **[引擎]**

    RowProxy 不再有 close() 方法，因为行不再保持对父级的引用。而是在父级 ResultProxy 上调用 close()，或者使用 autoclose。

+   **[引擎]**

    ResultProxy 内部已进行了大幅改进，大大减少了在提取列时的方法调用次数。在提取大型结果集时，可以提供大幅度的速度提升（高达 100%以上）。当提取没有应用类型级处理的列，并且将结果作为元组（而不是字典）时，提升效果更大。非常感谢 Elixir 的 Gaëtan de Menten 带来了这一巨大的改进！

    参考：[#1586](https://www.sqlalchemy.org/trac/ticket/1586)

+   **[引擎]**

    依赖于“最后插入 id”后获取生成序列值的数据库（例如 MySQL、MS-SQL）现在在表中“自增”列不是第一个主键列时，可以正确工作。

+   **[引擎]**

    last_inserted_ids() 方法已重命名为描述符“inserted_primary_key”。

+   **[引擎]**

    在 create_engine() 上设置 echo=False 现在将日志级别设置为 WARN 而不是 NOTSET。这样，即使总体上启用了“sqlalchemy.engine”的日志记录，也可以为特定引擎禁用日志记录。请注意，“echo”的默认设置是 None。

    参考：[#1554](https://www.sqlalchemy.org/trac/ticket/1554)

+   **[引擎]**

    ConnectionProxy 现在为所有事务生命周期事件提供包装方法，包括 begin()、rollback()、commit()、begin_nested()、begin_prepared()、prepare()、release_savepoint() 等。

+   **[引擎]**

    连接池日志现在使用 INFO 和 DEBUG 日志级别进行记录。INFO 用于主要事件，如无效的连接，DEBUG 用于所有获取/返回日志记录。echo_pool 可以是 False、None、True 或“debug”，与 echo 的工作方式相同。

+   **[引擎]**

    所有 pyodbc 方言现在支持额外的 pyodbc 特定关键字参数 'ansi'、'unicode_results'、'autocommit'。

    参考：[#1621](https://www.sqlalchemy.org/trac/ticket/1621)

+   **[引擎]**

    “threadlocal” 引擎已经重写和简化，现在支持 SAVEPOINT 操作。

+   **[引擎]**

    已弃用或移除

    +   result.last_inserted_ids()已弃用。使用 result.inserted_primary_key

    +   dialect.get_default_schema_name(connection)现在通过 dialect.default_schema_name 公开。

    +   从 engine.transaction()和 engine.run_callable()中删除了“connection”参数 - Connection 本身现在具有这些方法。所有四种方法都接受*args 和**kwargs，这些参数将传递给给定的可调用对象，以及操作连接。

+   **[反射/检查]**

    表反射已扩展并泛化为一个名为“sqlalchemy.engine.reflection.Inspector”的新 API。 Inspector 对象提供有关各种模式信息的细粒度信息，包括表名、列名、视图定义、序列、索引等，还有扩展的空间。

+   **[反射/检查]**

    视图现在可以反映为普通的 Table 对象。使用相同的 Table 构造函数，但要注意“有效”的主键和外键约束不是反射结果的一部分；如果需要，必须显式指定这些约束。

+   **[反射/检查]**

    现有的 autoload=True 系统现在在底层使用 Inspector，因此每个方言只需返回关于表和其他对象的“原始”数据 - Inspector 是将信息编译成 Table 对象的唯一位置，以便一致性最大化。

+   **[ddl]**

    DDL 系统已大大扩展。 DDL()类现在扩展了更通用的 DDLElement()，后者构成了许多新构造的基础：

    > > +   CreateTable()
    > > +   
    > > +   DropTable()
    > > +   
    > > +   AddConstraint()
    > > +   
    > > +   DropConstraint()
    > > +   
    > > +   CreateIndex()
    > > +   
    > > +   DropIndex()
    > > +   
    > > +   CreateSequence()
    > > +   
    > > +   DropSequence()
    > > +   
    > 这些支持“on”和“execute-at()”，就像普通的 DDL()一样。用户定义的 DDLElement 子类可以被创建并链接到一个编译器，使用 sqlalchemy.ext.compiler 扩展。

+   **[ddl]**

    传递给 DDL()和 DDLElement()的“on”可调用的签名如下所示：

    > ddl
    > 
    > DDLElement 对象本身
    > 
    > 事件
    > 
    > 字符串事件名称。
    > 
    > 目标
    > 
    > 以前是“schema_item”，触发事件的 Table 或 MetaData 对象。
    > 
    > 连接
    > 
    > 用于操作的 Connection 对象。
    > 
    > **kw
    > 
    > 关键字参数。在 MetaData 之前/之后创建/删除的情况下，要发出 CREATE/DROP DDL 的 Table 对象列表作为 kw 参数“tables”传递。这对于依赖于特定表存在的元数据级 DDL 是必要的。

    DDL 的���schema_item”属性已更名为

    ”目标”。

+   **[方言] [重构]**

    方言模块现在分为数据库方言和 DBAPI 实现。现在更倾向于使用 dialect+driver://…指定连接 URL，即“mysql+mysqldb://scott:tiger@localhost/test”。有关示例，请参阅 0.6 文档。

+   **[方言] [重构]**

    外部方言的 setuptools 入口现在称为“sqlalchemy.dialects”。

+   **[方言] [重构]**

    “owner” 关键字参数已从 Table 中移除。使用 “schema” 来表示要预置到表名前面的任何命名空间。

+   **[方言] [重构]**

    server_version_info 变成了一个静态属性。

+   **[方言] [重构]**

    方言在初始连接时接收 initialize() 事件以确定连接属性。

+   **[方言] [重构]**

    方言接收 visit_pool 事件有机会建立池监听器。

+   **[方言] [重构]**

    缓存的 TypeEngine 类现在按方言类而不是按方言缓存。

+   **[方言] [重构]**

    新的 UserDefinedType 应该作为新类型的基类，保留了 get_col_spec() 的 0.5 版本行为。

+   **[方言] [重构]**

    所有类型类的 result_processor() 方法现在接受第二个参数“coltype”，这是来自 cursor.description 的 DBAPI 类型参数。这个参数可以帮助一些类型决定对结果值进行最有效的处理。

+   **[方言] [重构]**

    废弃的 Dialect.get_params() 已移除。

+   **[方言] [重构]**

    Dialect.get_rowcount() 已重命名为描述符 “rowcount”，并直接调用 cursor.rowcount。需要为某些调用硬编码 rowcount 的方言应该重写该方法以提供不同的行为。

+   **[方言] [重构]**

    DefaultRunner 和其子类已被移除。这个对象的工作已经简化并移至 ExecutionContext 中。支持序列的方言应该在其执行上下文实现中添加一个 fire_sequence() 方法。

    参考：[#1566](https://www.sqlalchemy.org/trac/ticket/1566)

+   **[方言] [重构]**

    编译器生成的函数和操作现在使用（几乎）常规的分发函数形式 “visit_<opname>” 和 “visit_<funcname>_fn” 来提供定制处理。这取代了在编译器子类中复制 “functions” 和 “operators” 字典的需要，改为使用直接的访问方法，并且还允许编译器子类完全控制渲染，因为完整的 _Function 或 _BinaryExpression 对象被传递进来。

+   **[firebird]**

    RowProxy() 的 keys() 方法现在返回 *规范化* 为 SQLAlchemy 不区分大小写名称的结果列名。这意味着对于不区分大小写的名称，它们将是小写的，而 DBAPI 通常会将它们返回为大写名称。这使得行键() 可与进一步的 SQLAlchemy 操作兼容。

+   **[firebird]**

    使用新的 dialect.initialize() 特性来设置版本相关的行为。

+   **[firebird]**

    “case sensitivity” 特性将在反射期间检测到全小写的区分大小写的列名，并在生成的 Column 中添加 “quote=True”，以便保持正确的引用。

+   **[类型]**

    方言内的类型构造已经彻底改写。方言现在只以大写名称定义公开可用的类型，使用下划线标识符（即私有）定义内部实现类型。类型在 SQL 和 DDL 中的表示方式已经移到编译器系统。这样做的效果是，在大多数方言中类型对象大大减少了。有关方言作者的此体系结构的详细文档位于 lib/sqlalchemy/dialects/type_migration_guidelines.txt 中。

+   **[类型]**

    类型不再对默认参数进行任何猜测。特别是，Numeric、Float、NUMERIC、FLOAT、DECIMAL 不会生成任何长度或标度，除非指定。

+   **[类型]**

    types.Binary 被重命名为 types.LargeBinary，它只生成 BLOB、BYTEA 或类似的“长二进制”类型。新增了基础的 BINARY 和 VARBINARY 类型，以便以一种不可知的方式访问这些 MySQL/MS-SQL 特定的类型。

    参考：[#1664](https://www.sqlalchemy.org/trac/ticket/1664)

+   **[类型]**

    字符串/文本/Unicode 类型现在如果方言检测到 DBAPI 原生返回 Python Unicode 对象，则会跳过对每个结果列值的 unicode() 检查。在首次连接时使用“SELECT CAST 'some text' AS VARCHAR(10)”或等价物，然后检查返回的对象是否为 Python Unicode。这允许原生 Unicode DBAPI（包括 pysqlite/sqlite3、psycopg2 和 pg8000）获得巨大的性能提升。

+   **[类型]**

    大多数类型结果处理器都已经检查过可能的速度改进。具体而言，已经优化了以下通用类型，产生了不同程度的速度提升：Unicode、PickleType、Interval、TypeDecorator、Binary。此外，以下特定于 dbapi 的实现已经得到改进：Sqlite 上的 Time、Date 和 DateTime，PostgreSQL 上的 ARRAY，MySQL 上的 Time，MySQL 上的 Numeric（as_decimal=False），oursql 和 pypostgresql，cx_oracle 上的 DateTime 和基于 LOB 的类型。

+   **[类型]**

    现在类型的反射会返回 types.py 中的确切大写类型，或者如果类型不是标准 SQL 类型，则返回方言本身的大写类型。这意味着反射现在返回有关反射类型的更准确的信息。

+   **[类型]**

    添加了一个新的 Enum 通用类型。Enum 是一个模式感知对象，支持需要特定 DDL 才能使用枚举或等效的数据库；在 PG 的情况下，它处理 CREATE TYPE 的详细信息，在其他没有原生枚举支持的数据库上，它将生成 VARCHAR + 内联 CHECK 约束以强制执行枚举。

    参考：[#1109](https://www.sqlalchemy.org/trac/ticket/1109)，[#1511](https://www.sqlalchemy.org/trac/ticket/1511)

+   **[类型]**

    Interval 类型包括一个“native”标志，控制是否选择原生的 INTERVAL 类型（postgresql + oracle），如果可用，则选择。还添加了“day_precision”和“second_precision”参数，适当地传递到这些原生类型。相关联。

    参考：[#1467](https://www.sqlalchemy.org/trac/ticket/1467)

+   **[类型]**

    当在不具有本地布尔支持的后端使用布尔类型时，将生成一个 CHECK 约束“col IN (0, 1)”以及基于 int/smallint 的列类型。如果需要，可以通过 create_constraint=False 关闭此功能。请注意，MySQL 没有本地布尔或 CHECK 约束支持，因此该功能在该平台上不可用。

    参考：[#1589](https://www.sqlalchemy.org/trac/ticket/1589)

+   **[类型]**

    当 mutable=True 时，PickleType 现在使用 == 进行值比较，除非为该类型指定了带有比较函数的“comparator”参数。如果未重写 __eq__() 或未提供比较函数，则将基于标识比较被 pickled 的对象（这会使 mutable=True 失去意义）。

+   **[类型]**

    Numeric 和 Float 的默认“精度”和“标度”参数已被移除，现在默认为 None。NUMERIC 和 FLOAT 将默认不带任何数字参数呈现，除非提供这些值。

+   **[类型]**

    AbstractType.get_search_list() 已被移除 - 不再需要使用它的游戏。

+   **[类型]**

    添加了一个通用的 BigInteger 类型，编译为 BIGINT 或 NUMBER(19)。

    参考：[#1125](https://www.sqlalchemy.org/trac/ticket/1125)

+   **[类型]**

    sqlsoup 已进行了全面改进，明确支持 0.5 风格的会话，使用 autocommit=False、autoflush=True。SQLSoup 的默认行为现在需要常规的 commit() 和 rollback() 使用，这些方法已添加到其接口中。可以将显式的 Session 或 scoped_session 传递给构造函数，允许覆盖这些参数。

+   **[类型]**

    sqlsoup db.<sometable>.update() 和 delete() 现在分别调用 query(cls).update() 和 delete()。

+   **[类型]**

    sqlsoup 现在具有 execute() 和 connection()，这两个方法调用了具有相同名称的 Session 方法，确保绑定是基于 SqlSoup 对象的绑定。

+   **[类型]**

    sqlsoup 对象不再具有 ‘query’ 属性 - 对于 sqlsoup 的使用范例而言，这并不需要，而且实际上会妨碍一个名为 ‘query’ 的列。

+   **[类型]**

    传递给 association_proxy 的 proxy_factory 可调用对象的签名现在是 (lazy_collection, creator, value_attr, association_proxy)，添加了第四个参数，即父 AssociationProxy 参数。允许内置集合的��列化和子类化。

    参考：[#1259](https://www.sqlalchemy.org/trac/ticket/1259)

+   **[类型]**

    association_proxy 现在具有基本的比较方法 .any()、.has()、.contains()、==、!=，感谢 Scott Torborg。

    参考：[#1372](https://www.sqlalchemy.org/trac/ticket/1372)

## 0.6.9

发布日期：2012 年 5 月 5 日 星期六

### 一般

+   **[一般]**

    调整了“importlater”机制，该机制在内部用于解决导入循环，使得在导入 sqlalchemy 或 sqlalchemy.orm 完成时使用 __import__，从而避免在应用程序启动新线程后使用 __import__，修复了问题。

    参考：[#2279](https://www.sqlalchemy.org/trac/ticket/2279)

### orm

+   **[orm] [bug]**

    修复了在 query.get()中对用户映射对象在布尔上下文中的不当评估。

    参考：[#2310](https://www.sqlalchemy.org/trac/ticket/2310)

+   **[orm] [bug]**

    修复了当意外传递元组给 session.query()时引发的错误格式化问题。

    参考：[#2297](https://www.sqlalchemy.org/trac/ticket/2297)

+   **[orm]**

    修复了一个错误，即 query.join()使用的源子句在针对将多个实体组合在一起的列表达式时会不一致。

    参考：[#2197](https://www.sqlalchemy.org/trac/ticket/2197)

+   **[orm]**

    修复了仅在 Python 3 中显现的错误，即在 flush 期间对持久性+挂起对象进行排序会产生非法比较，如果持久性对象的主键不是单个整数。

    参考：[#2228](https://www.sqlalchemy.org/trac/ticket/2228)

+   **[orm]**

    修复了一个错误，即在 relationship()上使用 join 条件将查询.join() + aliased=True 从一个连接到自身的连接结构时，会不适当地将主实体转换为连接实体。

    参考：[#2234](https://www.sqlalchemy.org/trac/ticket/2234)

+   **[orm]**

    修复了一个错误，即 mapper.order_by 属性在子查询急加载中的“内部”查询中将被忽略。

    参考：[#2287](https://www.sqlalchemy.org/trac/ticket/2287)

+   **[orm]**

    修复了一个错误，即如果映射类重新定义了 __hash__()或 __eq__()为非标准内容，这是一个受支持的用例，因为 SQLA 不应该查询这些内容，如果类是“composite”（即非单实体）结果集的一部分，则会查询这些方法。

    参考：[#2215](https://www.sqlalchemy.org/trac/ticket/2215)

+   **[orm]**

    修复了一个微妙的错误，即如果发生：对子查询的 column_property() + joinedload + LIMIT +按列属性排序，则会导致 SQL 崩溃。

    参考：[#2188](https://www.sqlalchemy.org/trac/ticket/2188)

+   **[orm]**

    由 with_parent 生成的连接条件以及针对父级使用“dynamic”关系时，将生成唯一的绑定参数，而不是错误地重复相同的绑定参数。

    参考：[#2207](https://www.sqlalchemy.org/trac/ticket/2207)

+   **[orm]**

    修复了在查询中“无语句条件”断言的问题，该问题会在调用 from_statement()后调用生成方法时尝试引发异常。

    参考：[#2199](https://www.sqlalchemy.org/trac/ticket/2199)

+   **[orm]**

    Cls.column.collate(“some collation”) 现在可以正常工作。

    参考：[#1776](https://www.sqlalchemy.org/trac/ticket/1776)

### 示例

+   **[examples]**

    调整了 dictlike-polymorphic.py 示例，以便在 PG 和其他数据库上运行 CAST。

    参考：[#2266](https://www.sqlalchemy.org/trac/ticket/2266)

### 引擎

+   **[engine]**

    回溯了在 0.7.4 中引入的修复，确保在尝试在保存点和两阶段事务上调用 rollback()/prepare()/release()之前，连接处于有效状态。

    参考：[#2317](https://www.sqlalchemy.org/trac/ticket/2317)

### sql

+   **[sql]**

    修复了在可选择项中涉及列对应的两个微妙错误，一个是重复的具有相同标记的子查询，另一个是当标记已被“分组”并丢失自身时。影响。

    参考：[#2188](https://www.sqlalchemy.org/trac/ticket/2188)

+   **[sql]**

    修复了在某些方言中与 String 类型一起使用时“warn on unicode”标志会被设置的 bug。此 bug 不在 0.7 中。

+   **[sql]**

    修复了 Select 的 with_only_columns()方法如果传递了可选择项，则会失败的错误。但是，在这里 FROM 的行为仍然不正确，因此无论如何，您需要 0.7 才能使此用例可用。

    参考：[#2270](https://www.sqlalchemy.org/trac/ticket/2270)

### 模式

+   **[schema]**

    当 ForeignKeyConstraint 引用父级中未找到的列名时，添加了一个信息性错误消息。

### postgresql

+   **[postgresql]**

    修复了与 PG 9 中相同的修改索引行为影响重命名列上的主键反射的 bug。

    参考：[#2141](https://www.sqlalchemy.org/trac/ticket/2141), [#2291](https://www.sqlalchemy.org/trac/ticket/2291)

### mysql

+   **[mysql]**

    修复了 OurSQL 方言在 XA 命令中使用 ansi-neutral 引号符“’”而不是‘”’的 bug。

    参考：[#2186](https://www.sqlalchemy.org/trac/ticket/2186)

+   **[mysql]**

    CREATE TABLE 将在 CHARSET 之后放置 COLLATE 选项，这似乎是 MySQL 关于它是否实际工作的任意规则的一部分。

    参考：[#2225](https://www.sqlalchemy.org/trac/ticket/2225)

### mssql

+   **[mssql] [bug]**

    在检索索引名称列表和这些索引中的列名称时解码传入值。

    参考：[#2269](https://www.sqlalchemy.org/trac/ticket/2269)

### oracle

+   **[oracle]**

    将 ORA-00028 添加到断开代码中，使用 cx_oracle _Error.code 来获取代码。

    参考：[#2200](https://www.sqlalchemy.org/trac/ticket/2200)

+   **[oracle]**

    修复了未生成正确 DDL 的 oracle.RAW 类型。

    参考：[#2220](https://www.sqlalchemy.org/trac/ticket/2220)

+   **[oracle]**

    将 CURRENT 添加到保留字列表中。

    参考：[#2212](https://www.sqlalchemy.org/trac/ticket/2212)

### 一般

+   **[general]**

    调整了“importlater”机制，该机制在内部用于解决导入循环，使得在导入 sqlalchemy 或 sqlalchemy.orm 之后完成 __import__ 的使用，从而避免在应用程序启动新线程后使用 __import__，���复了。

    参考：[#2279](https://www.sqlalchemy.org/trac/ticket/2279)

### ORM

+   **[orm] [bug]**

    修复了在 query.get()中对用户映射对象进行布尔上下文中不适当评估的 bug。

    引用：[#2310](https://www.sqlalchemy.org/trac/ticket/2310)

+   **[orm] [bug]**

    修复了当不小心将元组传递给 session.query()时引发的错误格式化。

    引用：[#2297](https://www.sqlalchemy.org/trac/ticket/2297)

+   **[orm]**

    修复了 query.join()使用的源子句如果针对将多个实体组合在一起的列表达式，则会不一致的 bug。

    引用：[#2197](https://www.sqlalchemy.org/trac/ticket/2197)

+   **[orm]**

    修复了仅在 Python 3 中显现的 bug，即在 flush 期间对持久性+挂起对象进行排序会产生非法比较，如果持久性对象的主键不是单个整数。

    引用：[#2228](https://www.sqlalchemy.org/trac/ticket/2228)

+   **[orm]**

    修复了一个 bug，即当从一个关系()上的 joined-inh 结构到自身的 query.join() + aliased=True，并且关系()上的连接条件在子表上时，会不适当地将主实体转换为连接实体。

    引用：[#2234](https://www.sqlalchemy.org/trac/ticket/2234)

+   **[orm]**

    修复了 mapper.order_by 属性在子查询急加载中的“内部”查询中将被忽略的 bug。

    引用：[#2287](https://www.sqlalchemy.org/trac/ticket/2287)

+   **[orm]**

    修复了一个 bug，即如果一个映射类重新定义了 __hash__()或 __eq__()为非标准内容，这是一个支持的用例，因为 SQLA 不应该查询这些方法，但如果该类是“复合”（即非单实体）结果集的一部分，则这些方法将被查询。

    引用：[#2215](https://www.sqlalchemy.org/trac/ticket/2215)

+   **[orm]**

    修复了一个微妙的 bug，导致如果：column_property()针对子查询+joinedload+LIMIT+按列属性()排序，则 SQL 会崩溃。

    引用：[#2188](https://www.sqlalchemy.org/trac/ticket/2188)

+   **[orm]**

    由 with_parent 生成的连接条件以及使用“动态”关系针对父级时将生成唯一的 bindparams，而不是错误地重复相同的 bindparam。

    引用：[#2207](https://www.sqlalchemy.org/trac/ticket/2207)

+   **[orm]**

    修复了 Query 中的“无语句条件”断言，如果在调用 from_statement()之后调用生成方法，则会尝试引发异常。

    引用：[#2199](https://www.sqlalchemy.org/trac/ticket/2199)

+   **[orm]**

    Cls.column.collate(“some collation”)现��可以工作。

    引用：[#1776](https://www.sqlalchemy.org/trac/ticket/1776)

### 示例

+   **[examples]**

    调整了 dictlike-polymorphic.py 示例，以应用 CAST，使其在 PG 和其他数据库上运行。

    引用：[#2266](https://www.sqlalchemy.org/trac/ticket/2266)

### 引擎

+   **[engine]**

    回溯了在 0.7.4 中引入的修复，确保在尝试在保存点和两阶段事务上调用 rollback()/prepare()/release()之前，连接处于有效状态。

    引用：[#2317](https://www.sqlalchemy.org/trac/ticket/2317)

### sql

+   **[sql]**

    修复了两个关于可选列的微妙 bug，一个是同一标记的子查询重复出现，另一个是当标签被“分组”且失去自身时。影响到。

    参考：[#2188](https://www.sqlalchemy.org/trac/ticket/2188)

+   **[sql]**

    修复了一个 bug，当与某些方言一起使用时，“warn on unicode” 标志会设置为 String 类型。这个 bug 不在 0.7 中。

+   **[sql]**

    修复了一个 bug，当可选列被传递时，Select 的 with_only_columns() 方法会失败。但是，这里的 FROM 行为仍然不正确，因此无论如何您都需要 0.7 才能使用这种用法。

    参考：[#2270](https://www.sqlalchemy.org/trac/ticket/2270)

### schema

+   **[schema]**

    当 ForeignKeyConstraint 引用在父级中找不到的列名时，添加了一条信息性错误消息。

### postgresql

+   **[postgresql]**

    修复了一个与 PG 9 中相同的修改索引行为相关的 bug，该 bug 影响了对重命名列上的主键反射。

    参考：[#2141](https://www.sqlalchemy.org/trac/ticket/2141), [#2291](https://www.sqlalchemy.org/trac/ticket/2291)

### mysql

+   **[mysql]**

    修复了 OurSQL 方言，在 XA 命令中使用了 ANSI 中性的引号符“’”而不是‘”’。

    参考：[#2186](https://www.sqlalchemy.org/trac/ticket/2186)

+   **[mysql]**

    CREATE TABLE 将在 CHARSET 之后放置 COLLATE 选项，这似乎是 MySQL 关于其是否实际工作的任意规则的一部分。

    参考：[#2225](https://www.sqlalchemy.org/trac/ticket/2225)

### mssql

+   **[mssql] [bug]**

    解码传入值时，检索索引名称列表及这些索引内列的名称。

    参考：[#2269](https://www.sqlalchemy.org/trac/ticket/2269)

### oracle

+   **[oracle]**

    将 ORA-00028 添加到断开连接代码中，使用 cx_oracle _Error.code 获取代码。

    参考：[#2200](https://www.sqlalchemy.org/trac/ticket/2200)

+   **[oracle]**

    修复了未生成正确 DDL 的 oracle.RAW 类型。

    参考：[#2220](https://www.sqlalchemy.org/trac/ticket/2220)

+   **[oracle]**

    将 CURRENT 添加到保留字列表中。

    参考：[#2212](https://www.sqlalchemy.org/trac/ticket/2212)

## 0.6.8

发布日期：2011 年 6 月 5 日

### orm

+   **[orm]**

    对基于列的实体调用 query.get() 是无效的，现在会引发弃用警告。

    参考：[#2144](https://www.sqlalchemy.org/trac/ticket/2144)

+   **[orm]**

    非主映射器将继承主映射器的 _identity_class。这样，针对通常处于继承映射中的类建立的非主映射器将产生与主映射器兼容的标识映射结果。

    参考：[#2151](https://www.sqlalchemy.org/trac/ticket/2151)

+   **[orm]**

    回溯到了 0.7 的标识映射实现，它在删除时不使用互斥锁。因为一些用户仍然在 0.6.7 中进行了调整，尽管这个版本没有使用互斥锁的 0.7 方法似乎不会产生“字典更改大小”问题，这是原始互斥锁的理由。

    参考：[#2148](https://www.sqlalchemy.org/trac/ticket/2148)

+   **[orm]**

    修复了“无法为目标列‘q’执行 syncrule；mapper‘X’未映射此列”发出的错误消息，以引用正确的 mapper。

    参考：[#2163](https://www.sqlalchemy.org/trac/ticket/2163)

+   **[orm]**

    修复了“自引用”关系的确定失败的 bug，对于没有解决方案的 joined-inh 子类与自身相关联，或者与没有子类中的列在连接条件中的子子类相关联。

    参考：[#2149](https://www.sqlalchemy.org/trac/ticket/2149)

+   **[orm]**

    在确定父类和子类之间的继承条件时，mapper()将忽略与无关表的未配置外键。这等同于已应用于 declarative 的行为。请注意，0.7 版本对此有更全面的解决方案，改变了 join()本身如何确定 FK 错误。

    参考：[#2153](https://www.sqlalchemy.org/trac/ticket/2153)

+   **[orm]**

    修复了如果 mapper 映射到匿名别名，则在使用日志记录时会失败的 bug，因为别名中有未转义的%符号。

    参考：[#2171](https://www.sqlalchemy.org/trac/ticket/2171)

+   **[orm]**

    修改了当在 flush 时未检测到“identity”键时出现的消息文本，以包括常见原因，即 Column 未设置正确检测自增。

    参考：[#2170](https://www.sqlalchemy.org/trac/ticket/2170)

+   **[orm]**

    修复了事务级别的“已删除”集合不会在后来变为瞬态时清除已删除状态的错误，导致后续出现错误。

    参考：[#2182](https://www.sqlalchemy.org/trac/ticket/2182)

### 引擎

+   **[engine]**

    调整了 RowProxy 结果行的 __contains__()方法，以便在内部不生成异常抛出；无论列构造是否可以强制转换为字符串，NoSuchColumnError()也将生成其消息。

    参考：[#2178](https://www.sqlalchemy.org/trac/ticket/2178)

### sql

+   **[sql]**

    修复了如果将 FetchedValue 传递给 column server_onupdate，则其父“column”不会被分配的 bug，为所有列默认分配模式添加了测试覆盖。

    参考：[#2147](https://www.sqlalchemy.org/trac/ticket/2147)

+   **[sql]**

    修复了嵌套一个带有另一个标签的 select()的标签会产生不正确导出列的错误的 bug。其中之一是这会破坏对另一个 column_property()的 ORM column_property()映射。

    参考：[#2167](https://www.sqlalchemy.org/trac/ticket/2167)

### postgresql

+   **[postgresql]**

    修复了影响 PG 9 的 bug，即反射索引会失败，如果反射的列名已更改。

    参考：[#2141](https://www.sqlalchemy.org/trac/ticket/2141)

+   **[postgresql]**

    修复了关于数字数组、MATCH 运算符的一些单元测试问题。修复了潜在的浮点不精确性问题，并且目前只有在 EN 定向区域设置内才执行 MATCH 运算符的某些测试。.

    参考：[#2175](https://www.sqlalchemy.org/trac/ticket/2175)

### mssql

+   **[mssql]**

    修复了 MSSQL 方言中的一个 bug，即应用于模式限定表的别名会泄漏到封闭的 select 语句中。

    参考：[#2169](https://www.sqlalchemy.org/trac/ticket/2169)

+   **[mssql]**

    修复了一个 bug，即在结果集或绑定参数中使用 DATETIME2 类型时，在“适应”步骤中会失败。这个问题不在 0.7 中。

    参考：[#2159](https://www.sqlalchemy.org/trac/ticket/2159)

### orm

+   **[orm]**

    对基于列的实体调用 query.get()是无效的，现在会引发弃用警告。

    参考：[#2144](https://www.sqlalchemy.org/trac/ticket/2144)

+   **[orm]**

    非主映射器将继承主映射器的 _identity_class。这样，针对通常处于继承映射中的类建立的非主映射器将产生与主映射器的身份映射兼容的结果。

    参考：[#2151](https://www.sqlalchemy.org/trac/ticket/2151)

+   **[orm]**

    回溯了 0.7 的身份映射实现，不使用互斥锁进行删除。这是因为一些用户尽管在 0.6.7 中进行了调整，仍然出现死锁；不使用互斥锁的 0.7 方法似乎不会产生“字典更改大小”问题，这是互斥锁的最初原因。

    参考：[#2148](https://www.sqlalchemy.org/trac/ticket/2148)

+   **[orm]**

    修复了“无法为目标列‘q’执行 syncrule；映射器‘X’未映射此列”的错误消息，以引用正确的映射器。。

    参考：[#2163](https://www.sqlalchemy.org/trac/ticket/2163)

+   **[orm]**

    修复了一个 bug，即在没有解决方案的情况下，确定“自引用”关系会失败，或者与自身相关的 joined-inh 子类，或者与没有在子子类中的列在连接条件中的子类相关的 joined-inh 子类。

    参考：[#2149](https://www.sqlalchemy.org/trac/ticket/2149)

+   **[orm]**

    在确定父类和子类之间的继承条件时，mapper()将忽略与不相关表的非配置外键。这等同于已应用于 declarative 的行为。请注意，0.7 有一个更全面的解决方案，改变了 join()本身如何确定 FK 错误。

    参考：[#2153](https://www.sqlalchemy.org/trac/ticket/2153)

+   **[orm]**

    修复了一个 bug，即映射到匿名别名的映射器在使用日志记录时会失败，因为别名中有未转义的%符号。

    参考：[#2171](https://www.sqlalchemy.org/trac/ticket/2171)

+   **[orm]**

    修改了当在刷新时未检测到“identity”键时发生的消息文本，以包括列未设置正确检测自增的常见原因;。

    参考：[#2170](https://www.sqlalchemy.org/trac/ticket/2170)

+   **[orm]**

    修复了事务级别的“已删除”集合不会清除已删除状态，如果它们后来变为瞬态会引发错误的 bug。

    参考：[#2182](https://www.sqlalchemy.org/trac/ticket/2182)

### engine

+   **[engine]**

    调整了 RowProxy 结果行的 __contains__() 方法，使其在内部不生成异常抛出；无论列构造是否可以强制转换为字符串，NoSuchColumnError() 也会生成其消息。

    参考：[#2178](https://www.sqlalchemy.org/trac/ticket/2178)

### sql

+   **[sql]**

    修复了如果将 FetchedValue 传递给列 server_onupdate，它将不会分配其父“列”的 bug，为所有列默认分配模式添加了测试覆盖。

    参考：[#2147](https://www.sqlalchemy.org/trac/ticket/2147)

+   **[sql]**

    修复了在一个 select() 的标签中嵌套另一个标签会产生不正确导出列的 bug。其中一个问题是会破坏针对另一个 column_property() 的 ORM column_property() 映射。

    参考：[#2167](https://www.sqlalchemy.org/trac/ticket/2167)

### postgresql

+   **[postgresql]**

    修复了影响 PG 9 的 bug，即反射索引会失败，如果针对一个列名已更改的列。。

    参考：[#2141](https://www.sqlalchemy.org/trac/ticket/2141)

+   **[postgresql]**

    一些关于数字数组、MATCH 运算符的单元测试修复。修复了潜在的浮点不精确问题，并且目前只有在 EN 定向区域设置内执行 MATCH 运算符的某些测试。

    参考：[#2175](https://www.sqlalchemy.org/trac/ticket/2175)

### mssql

+   **[mssql]**

    修复了 MSSQL 方言中的 bug，即应用于模式限定表的别名会泄漏到封闭的 select 语句中。

    参考：[#2169](https://www.sqlalchemy.org/trac/ticket/2169)

+   **[mssql]**

    修复了 DATETIME2 类型在结果集或绑定参数中使用时在“适应”步骤中失败的 bug。此问题不在 0.7 版本中出现。

    参考：[#2159](https://www.sqlalchemy.org/trac/ticket/2159)

## 0.6.7

发布日期：2011 年 4 月 13 日

### orm

+   **[orm]**

    加强了在身份映射迭代周围的迭代与移除互斥锁，试图减少极其罕见的重新进入 gc 操作导致死锁的机会。可能会在 0.7 版本中移除互斥锁。

    参考：[#2087](https://www.sqlalchemy.org/trac/ticket/2087)

+   **[orm]**

    向 Query.subquery() 添加了一个 name 参数，允许为别名对象分配一个固定名称。

    参考：[#2030](https://www.sqlalchemy.org/trac/ticket/2030)

+   **[orm]**

    当一个联接表继承映射器在本地映射表上没有主键（但在超类表上有主键）时，会发出警告。

    参考：[#2019](https://www.sqlalchemy.org/trac/ticket/2019)

+   **[orm]**

    修复了一个 bug，即多态层次结构中的“中间”类如果没有指定‘polymorphic_identity’，则不会有‘polymorphic_on’列，导致在刷新时出现奇怪的错误，从该目标查询时加载错误的类。还在使用单表继承时发出正确的 WHERE 条件。

    参考：[#2038](https://www.sqlalchemy.org/trac/ticket/2038)

+   **[orm]**

    修复了一个 bug，即具有 SQL 或服务器端默认值的列，如果在使用 include_properties 或 exclude_properties 时被排除在映射之外，将导致 UnmappedColumnError。

    参考：[#1995](https://www.sqlalchemy.org/trac/ticket/1995)

+   **[orm]**

    在集合上发生附加或类似事件之后，如果父对象已被取消引用，将发出警告，这将阻止父对象在会话中被标记为“脏”。在 0.7 中将是一个异常。

    参考：[#2046](https://www.sqlalchemy.org/trac/ticket/2046)

+   **[orm]**

    修复了 query.options()中的一个 bug，即应用于使用字符串键的 lazyload 的路径可能与错误实体上的同名属性重叠。请注意，0.7 版本已更新了此修复版本。

    参考：[#2098](https://www.sqlalchemy.org/trac/ticket/2098)

+   **[orm]**

    重新定义了当尝试刷新一个不与超类型多态化的子类时引发的异常。

    参考：[#2063](https://www.sqlalchemy.org/trac/ticket/2063)

+   **[orm]**

    关于反向引用(backrefs)的状态处理进行了一些修复，通常在 autoflush=False 时，当反向引用的集合没有净变化时，不会正确处理添加/删除操作。感谢 Richard Murri 提供的测试用例和补丁。

    参考：[#2123](https://www.sqlalchemy.org/trac/ticket/2123)

+   **[orm]**

    如果使用 from_self()，则“having”子句将从内部复制到外部查询中。

    参考：[#2130](https://www.sqlalchemy.org/trac/ticket/2130)

### 示例

+   **[示例]**

    Beaker 缓存示例允许在 query_callable()函数中使用“query_cls”参数。

    参考：[#2090](https://www.sqlalchemy.org/trac/ticket/2090)

### engine

+   **[engine]**

    修复了 QueuePool、SingletonThreadPool 中的 bug，即通过溢出或定期 cleanup()丢弃的连接没有被显式关闭，而是留给任务进行垃圾回收。这通常只影响像 Jython 和 PyPy 这样的非引用计数后端。感谢 Jaimy Azle 发现这个问题。

    参考：[#2102](https://www.sqlalchemy.org/trac/ticket/2102)

### sql

+   **[sql]**

    Column.copy()，在 table.tometadata()中使用，会复制‘doc’属性。

    参考：[#2028](https://www.sqlalchemy.org/trac/ticket/2028)

+   **[sql]**

    在 resultproxy.c 扩展中添加了一些 defs，以便该扩展在 Python 2.4 上编译和运行。

    参考：[#2023](https://www.sqlalchemy.org/trac/ticket/2023)

+   **[sql]**

    编译器扩展现在支持覆盖 expression._BindParamClause 的默认编译，包括 insert()/update() 语句的 VALUES/SET 子句内自动生成的绑定也将使用新的编译规则。

    参考：[#2042](https://www.sqlalchemy.org/trac/ticket/2042)

+   **[sql]**

    添加了 ResultProxy 的访问器“returns_rows”、“is_insert”

    参考：[#2089](https://www.sqlalchemy.org/trac/ticket/2089)

+   **[sql]**

    select() 中的 limit/offset 关键字以及传递给 select.limit()/offset() 的值将被强制转换为整数。

    参考：[#2116](https://www.sqlalchemy.org/trac/ticket/2116)

### postgresql

+   **[postgresql]**

    当显式序列执行推导出 SERIAL 列的自动生成序列的名称时，目前仅在 implicit_returning=False 时发生，现在如果表 + 列名大于 63 个字符，则使用与 PostgreSQL 相同的逻辑。

    参考：[#1083](https://www.sqlalchemy.org/trac/ticket/1083)

+   **[postgresql]**

    添加了一个额外的 libpq 消息到“断开连接”异常列表中，“无法从服务器接收数据”

    参考：[#2044](https://www.sqlalchemy.org/trac/ticket/2044)

+   **[postgresql]**

    为 postgresql 方言添加了 RESERVED_WORDS。

    参考：[#2092](https://www.sqlalchemy.org/trac/ticket/2092)

+   **[postgresql]**

    修复了 BIT 类型允许“length”参数、“varying”参数的问题。反射也已修复。

    参考：[#2073](https://www.sqlalchemy.org/trac/ticket/2073)

### mysql

+   **[mysql]**

    oursql 方言接受与 MySQLdb 相同的“ssl”参数。

    参考：[#2047](https://www.sqlalchemy.org/trac/ticket/2047)

### sqlite

+   **[sqlite]**

    修复了当外键反射创建为 “REFERENCES <tablename>” 而没有列名时失败的 bug。

    参考：[#2115](https://www.sqlalchemy.org/trac/ticket/2115)

### mssql

+   **[mssql]**

    重写了用于获取视图定义的查询，通常在使用 Inspector 接口时使用，改为使用 sys.sql_modules 而不是信息模式，从而允许完整返回超过 4000 个字符的视图定义。

    参考：[#2071](https://www.sqlalchemy.org/trac/ticket/2071)

### oracle

+   **[oracle]**

    当使用需要引号的列名用于列本身或为名称生成的绑定参数，例如具有特殊字符、下划线、非 ASCII 字符的名称时，现在在与 cx_oracle 通信时会正确翻译绑定参数键。

    参考：[#2100](https://www.sqlalchemy.org/trac/ticket/2100)

+   **[oracle]**

    Oracle 方言添加了 use_binds_for_limits=False create_engine() 标志，将限制/偏移值内联渲染而不是作为绑定，据报告修改了 Oracle 使用的执行计划。

    参考：[#2116](https://www.sqlalchemy.org/trac/ticket/2116)

### 杂项

+   **[informix]**

    添加了 RESERVED_WORDS informix 方言。

    参考：[#2092](https://www.sqlalchemy.org/trac/ticket/2092)

+   **[firebird]**

    在 create_engine() 上设置为 False 时，“implicit_returning” 标志会被尊重。

    参考：[#2083](https://www.sqlalchemy.org/trac/ticket/2083)

+   **[ext]**

    horizontal_shard ShardedSession 类接受常见的 Session 参数“query_cls”作为构造函数参数，以便进一步对 ShardedQuery 进行子类化。

    参考：[#2090](https://www.sqlalchemy.org/trac/ticket/2090)

+   **[declarative]**

    添加了一个明确的检查，以确保在声明类的列属性上使用名称‘metadata’的情况。

    参考：[#2050](https://www.sqlalchemy.org/trac/ticket/2050)

+   **[declarative]**

    修复错误消息引用旧的 @classproperty 名称以引用 @declared_attr

    参考：[#2061](https://www.sqlalchemy.org/trac/ticket/2061)

+   **[declarative]**

    在 __mapper_args__ 中不可“哈希化”的参数不会被误认为始终可哈希化，可能会被误认为列参数。

    参考：[#2091](https://www.sqlalchemy.org/trac/ticket/2091)

+   **[documentation]**

    记录了 SQLite DATE/TIME/DATETIME 类型。

    参考：[#2029](https://www.sqlalchemy.org/trac/ticket/2029)

### orm

+   **[orm]**

    加强了关于身份映射迭代与删除互斥锁的控制，试图减少（极其罕见的）重新进入的 gc 操作导致死锁的机会。可能会在 0.7 版本中移除互斥锁。

    参考：[#2087](https://www.sqlalchemy.org/trac/ticket/2087)

+   **[orm]**

    向 Query.subquery() 添加了一个 name 参数，以允许为别名对象分配一个固定的名称。

    参考：[#2030](https://www.sqlalchemy.org/trac/ticket/2030)

+   **[orm]**

    当连接表继承映射器在本地映射表上没有主键（但在超类表上有主键）时，会发出警告。

    参考：[#2019](https://www.sqlalchemy.org/trac/ticket/2019)

+   **[orm]**

    修复了多态层次结构中的“中间”类没有‘polymorphic_on’列的 bug，如果它没有指定‘polymorphic_identity’，则在刷新时会导致奇怪的错误，从该目标查询时加载错误的类。在使用单表继承时，还会发出正确的 WHERE 条件。

    参考：[#2038](https://www.sqlalchemy.org/trac/ticket/2038)

+   **[orm]**

    修复了一个 bug，即具有 SQL 或服务器端默认值的列，如果使用 include_properties 或 exclude_properties 从映射中排除，将导致 UnmappedColumnError。

    参考：[#1995](https://www.sqlalchemy.org/trac/ticket/1995)

+   **[orm]**

    在罕见情况下，如果在父对象被取消引用后发生附加或类似事件的情况，将发出警告，这会阻止父对象在会话中被标记为“脏”。这将在 0.7 版本中成为异常。

    参考：[#2046](https://www.sqlalchemy.org/trac/ticket/2046)

+   **[orm]**

    修复了 query.options() 中的 bug，即应用于使用字符串键的延迟加载的路径可能会与错误实体上的同名属性重叠。请注意，0.7 版本中有此修复的更新版本。

    参考：[#2098](https://www.sqlalchemy.org/trac/ticket/2098)

+   **[orm]**

    重新修订了当尝试刷新不与超类型多态的子类时引发的异常。

    参考：[#2063](https://www.sqlalchemy.org/trac/ticket/2063)

+   **[orm]**

    关于反向引用(backrefs)的状态处理进行了一些修复，通常在 autoflush=False 时，当反向引用集合未正确处理没有净变化的 add/remove 时。感谢 Richard Murri 提供测试用例和补丁。

    参考：[#2123](https://www.sqlalchemy.org/trac/ticket/2123)

+   **[orm]**

    如果使用 from_self()，则“having”子句将从内部复制到外部查询中。

    参考：[#2130](https://www.sqlalchemy.org/trac/ticket/2130)

### examples

+   **[examples]**

    Beaker 缓存示例允许“query_cls”参数传递给 query_callable()函数。

    参考：[#2090](https://www.sqlalchemy.org/trac/ticket/2090)

### engine

+   **[engine]**

    修复了 QueuePool、SingletonThreadPool 中的 bug，其中通过溢出或定期 cleanup()丢弃的连接没有明确关闭，导致垃圾回收任务。这通常只影响像 Jython 和 PyPy 这样不支持引用计数的后端。感谢 Jaimy Azle 发现了这个问题。

    参考：[#2102](https://www.sqlalchemy.org/trac/ticket/2102)

### sql

+   **[sql]**

    Column.copy()，如在 table.tometadata()中使用，将复制‘doc’属性。

    参考：[#2028](https://www.sqlalchemy.org/trac/ticket/2028)

+   **[sql]**

    为 resultproxy.c 扩展添加了一些 defs，以便该扩展在 Python 2.4 上编译和运行。

    参考：[#2023](https://www.sqlalchemy.org/trac/ticket/2023)

+   **[sql]**

    编译器扩展现在支持覆盖 expression._BindParamClause 的默认编译，包括 insert()/update()语句的 VALUES/SET 子句中的自动生成绑定也将使用新的编译规则。

    参考：[#2042](https://www.sqlalchemy.org/trac/ticket/2042)

+   **[sql]**

    为 ResultProxy 添加了访问器“returns_rows”、“is_insert”

    参考：[#2089](https://www.sqlalchemy.org/trac/ticket/2089)

+   **[sql]**

    select()中的 limit/offset 关键字以及传递给 select.limit()/offset()的值将被强制转换为整数。

    参考：[#2116](https://www.sqlalchemy.org/trac/ticket/2116)

### postgresql

+   **[postgresql]**

    当显式序列执行派生 SERIAL 列的自动生成序列的名称时，当前仅在 implicit_returning=False 时发生，现在适应了如果表+列名大于 63 个字符，则使用 PostgreSQL 使用的相同逻辑。

    参考：[#1083](https://www.sqlalchemy.org/trac/ticket/1083)

+   **[postgresql]**

    将“disconnect”异常列表中的另一个 libpq 消息“could not receive data from server”添加到列表中。

    参考：[#2044](https://www.sqlalchemy.org/trac/ticket/2044)

+   **[postgresql]**

    为 postgresql 方言添加了 RESERVED_WORDS。

    参考：[#2092](https://www.sqlalchemy.org/trac/ticket/2092)

+   **[postgresql]**

    修复了 BIT 类型以允许“length”参数，“varying”参数。反射也已修复。

    参考：[#2073](https://www.sqlalchemy.org/trac/ticket/2073)

### mysql

+   **[mysql]**

    oursql 方言在 create_engine()中接受与 MySQLdb 相同的“ssl”参数。

    参考：[#2047](https://www.sqlalchemy.org/trac/ticket/2047)

### sqlite

+   **[sqlite]**

    修复了外键反射创建为“REFERENCES <tablename>”而没有列名的 bug。

    参考：[#2115](https://www.sqlalchemy.org/trac/ticket/2115)

### mssql

+   **[mssql]**

    重写了用于获取视图定义的查询，通常在使用 Inspector 接口时使用 sys.sql_modules 而不是信息模式，从而允许完全返回长于 4000 个字符的视图定义。

    参考：[#2071](https://www.sqlalchemy.org/trac/ticket/2071)

### oracle

+   **[oracle]**

    使用需要为列本身或为名称生成的绑定参数需要引号的列名，例如具有特殊字符、下划线、非 ASCII 字符的名称，现在在与 cx_oracle 通信时正确地转换绑定参数键。

    参考：[#2100](https://www.sqlalchemy.org/trac/ticket/2100)

+   **[oracle]**

    Oracle 方言添加了 use_binds_for_limits=False create_engine()标志，将在行内呈现 LIMIT/OFFSET 值，而不是作为绑定，据报道修改了 Oracle 使用的执行计划。

    参考：[#2116](https://www.sqlalchemy.org/trac/ticket/2116)

### misc

+   **[informix]**

    添加了 RESERVED_WORDS informix 方言。

    参考：[#2092](https://www.sqlalchemy.org/trac/ticket/2092)

+   **[firebird]**

    如果将“implicit_returning”标志设置为 False，则 create_engine()上的“implicit_returning”标志将被尊重。

    参考：[#2083](https://www.sqlalchemy.org/trac/ticket/2083)

+   **[ext]**

    horizontal_shard ShardedSession 类接受常见的 Session 参数“query_cls”作为构造函数参数，以启用 ShardedQuery 的进一步子类化。

    参考：[#2090](https://www.sqlalchemy.org/trac/ticket/2090)

+   **[declarative]**

    明确检查了在声明类上的列属性中使用名称“metadata”的情况。

    参考：[#2050](https://www.sqlalchemy.org/trac/ticket/2050)

+   **[declarative]**

    修复错误消息引用旧的@classproperty 名称以引用@declared_attr

    参考：[#2061](https://www.sqlalchemy.org/trac/ticket/2061)

+   **[declarative]**

    在 __mapper_args__ 中的参数如果不是“可哈希”的，则不会被误认为始终是可哈希的，可能是列参数。

    参考：[#2091](https://www.sqlalchemy.org/trac/ticket/2091)

+   **[documentation]**

    记录了 SQLite DATE/TIME/DATETIME 类型。

    参考：[#2029](https://www.sqlalchemy.org/trac/ticket/2029)

## 0.6.6

发布日期：2011 年 1 月 8 日星期六

### orm

+   **[orm]**

    修复了非“mutable”属性修改事件发生在干净对象上，除了之前保存在“mutable changes”字典中的更改之外，将无法强引用自身在标识映射中。这将导致对象被垃圾回收，丢失任何之前未保存在“mutable changes”字典中的更改。

+   **[orm]**

    修复了“passive_deletes='all'”在 flush 期间未将正确符号传递给懒加载程序的错误，从而导致不必要的加载。

    参考：[#2013](https://www.sqlalchemy.org/trac/ticket/2013)

+   **[orm]**

    修复了阻止复合映射属性在映射的选择语句中使用的错误。请注意，复合的工作方式在 0.7 中将发生重大变化。

    参考：[#1997](https://www.sqlalchemy.org/trac/ticket/1997)

+   **[orm]**

    还向 composite()添加了 active_history 标志。该标志在 0.6 中没有效果，而是一个用于向前兼容的占位符标志，因为它在 0.7 中适用于 composites。

    参考：[#1976](https://www.sqlalchemy.org/trac/ticket/1976)

+   **[orm]**

    修复了 uow 错误，即当传递到 Session.delete()的过期对象在删除对象时不考虑未加载的引用或集合时，尽管 passive_deletes 保持默认值 False。

    参考：[#2002](https://www.sqlalchemy.org/trac/ticket/2002)

+   **[orm]**

    当继承的映射器上已经指定 version_id_col 时，在继承的映射器上指定 version_id_col 时会发出警告，如果这些列表达式不相同。

    参考：[#1987](https://www.sqlalchemy.org/trac/ticket/1987)

+   **[orm]**

    如果在 joinedload()连接链中的先前连接是外连接，则“innerjoin”标志不会沿着连接链生效，因此允许正确返回没有引用子行的主行结果。

    参考：[#1954](https://www.sqlalchemy.org/trac/ticket/1954)

+   **[orm]**

    修复了关于“subqueryload”策略的错误，即如果实体是一个 aliased()构造，则策略将失败。

    参考：[#1964](https://www.sqlalchemy.org/trac/ticket/1964)

+   **[orm]**

    修复了关于“subqueryload”策略的错误，即如果使用形式为从 A->joined-subclass->C 的多级加载，则连接将失败。

    参考：[#2014](https://www.sqlalchemy.org/trac/ticket/2014)

+   **[orm]**

    修复了通过-1 对 Query 对象进行索引的错误。它错误地转换为空切片-1:0，导致 IndexError。

    参考：[#1968](https://www.sqlalchemy.org/trac/ticket/1968)

+   **[orm]**

    mapper 参数“primary_key”可以传递为单个列，也可以传递为列表或元组。文档中将其示例更改为列表。

    参考：[#1971](https://www.sqlalchemy.org/trac/ticket/1971)

+   **[orm]**

    向 relationship()和 column_property()添加了 active_history 标志，强制属性事件始终加载“旧”值，以便它在 attributes.get_history()中可用。

    参考：[#1961](https://www.sqlalchemy.org/trac/ticket/1961)

+   **[ORM]**

    如果复合键中的参数数量过大或过小，Query.get() 将引发异常。

    参考：[#1977](https://www.sqlalchemy.org/trac/ticket/1977)

+   **[ORM]**

    从 0.7 中回退了“优化的获取”修复，改进了联合继承“加载过期行”行为的生成。

    参考：[#1992](https://www.sqlalchemy.org/trac/ticket/1992)

+   **[ORM]**

    在视图只读时 join 条件“有效”但在非视图只读时不起作用的异常条件下，为“primaryjoin”错误添加了更多描述，且未使用 foreign_keys - 在建议中添加了“foreign_keys”。还为通用“direction”错误的建议添加了“foreign_keys”。

### 示例

+   **[示例]**

    版本示例现在支持检测关联关系中的更改。

### 引擎

+   **[引擎]**

    当显式使用 Unicode 类型时，只有在引擎或字符串类型上使用 convert_unicode=True 时才会引发“unicode 警告”；而不是在绑定数据为非 Unicode 时。

+   **[引擎]**

    修复了 Decimal 结果处理器的 C 版本中的内存泄漏。

    参考：[#1978](https://www.sqlalchemy.org/trac/ticket/1978)

+   **[引擎]**

    为 RowProxy 的 C 版本实现了序列检查功能，以及对 RowProxy 的 2.7 风格“collections.Sequence”注册。

    参考：[#1871](https://www.sqlalchemy.org/trac/ticket/1871)

+   **[引擎]**

    Threadlocal 引擎方法 rollback()、commit()、prepare() 在没有事务进行时不会引发异常；这是 0.6 中引入的一个回归。

    参考：[#1998](https://www.sqlalchemy.org/trac/ticket/1998)

+   **[引擎]**

    Threadlocal 引擎在 begin()、begin_nested() 后返回自身；然后引擎实现上下文管理器方法，以允许“with”语句。

    参考：[#2004](https://www.sqlalchemy.org/trac/ticket/2004)

### SQL

+   **[SQL]**

    修复了单个非关联运算符链的运算符优先级规则。即“x - (y - z)”将编译为“x - (y - z)”而不是“x - y - z”。也适用于标签，即“x - (y - z).label(‘foo’)”

    参考：[#1984](https://www.sqlalchemy.org/trac/ticket/1984)

+   **[SQL]**

    Column 的 ‘info’ 属性在 Column.copy() 期间被复制，即在声明性 mixin 中使用列时发生的情况。

    参考：[#1967](https://www.sqlalchemy.org/trac/ticket/1967)

+   **[SQL]**

    为布尔值添加了一个绑定处理器，将其强制转换为 int，用于像 pymssql 这样的 DBAPI，在值上天真地调用 str()。

+   **[SQL]**

    CheckConstraint 将在 copy()/tometadata() 中复制其‘initially’、‘deferrable’和‘_create_rule’属性。

    参考：[#2000](https://www.sqlalchemy.org/trac/ticket/2000)

### PostgreSQL

+   **[PostgreSQL]**

    IN 子句中的单个元组表达式正确地加括号，也来自

    参考：[#1984](https://www.sqlalchemy.org/trac/ticket/1984)

+   **[PostgreSQL]**

    确保 psycopg2 和 pg8000 的“numeric”基本类型识别每个数字、浮点数、整数代码、标量 + 数组。

    引用：[#1955](https://www.sqlalchemy.org/trac/ticket/1955)

+   **[postgresql]**

    对 UUID 类型添加了 as_uuid=True 标志，将接收和返回值作为 Python UUID()对象而不是字符串。目前，UUID 类型只能与 psycopg2 一起使用。

    引用：[#1956](https://www.sqlalchemy.org/trac/ticket/1956)

+   **[postgresql]**

    修复了在池处理+重新创建后，非 ENUM 支持的 PG 版本会出现 KeyError 的错误。

    引用：[#1989](https://www.sqlalchemy.org/trac/ticket/1989)

### mysql

+   **[mysql]**

    修复了 Jython + zxjdbc 的错误处理，使 has_table()属性再次起作用。这是从 0.6.3 开始的回归（我们没有 Jython 构建机，抱歉）

    引用：[#1960](https://www.sqlalchemy.org/trac/ticket/1960)

### sqlite

+   **[sqlite]**

    在 CREATE TABLE 中包含对具有相同模式名称的另一个表的远程模式的 REFERENCES 子句现在呈现远程名称而不包含模式子句，这是 SQLite 所要求的。

    引用：[#1851](https://www.sqlalchemy.org/trac/ticket/1851)

+   **[sqlite]**

    在相同的主题上，在 CREATE TABLE 中包含对*不同*模式的远程模式的 REFERENCES 子句不会呈现，因为似乎不支持跨模式引用。

### mssql

+   **[mssql]**

    对索引反射的重写不幸地没有经过正确测试，并返回了不正确的结果。这个回归现在已经修复。

    引用：[#1770](https://www.sqlalchemy.org/trac/ticket/1770)

### oracle

+   **[oracle]**

    cx_oracle 的“十进制检测”逻辑，对于具有模糊数值特征的结果集列，现在使用由 locale/NLS_LANG 设置确定的小数点字符，使用首次连接时检测此字符。在使用非句点小数点 NLS_LANG 设置时，还需要 cx_oracle 5.0.3 或更高版本。

    引用：[#1953](https://www.sqlalchemy.org/trac/ticket/1953)

### 杂项

+   **[firebird]**

    Firebird 数值类型现在明确检查 Decimal，让 float()直接通过，从而允许特殊值如 float('inf')。

    引用：[#2012](https://www.sqlalchemy.org/trac/ticket/2012)

+   **[declarative]**

    如果 __table_args__ 不是元组或字典格式，并且不是 None，则会引发错误。

    引用：[#1972](https://www.sqlalchemy.org/trac/ticket/1972)

+   **[sqlsoup]**

    向 SqlSoup 添加了“map_to()”方法，这是一个“主”方法，接受每个可选择对象和映射的显式参数，包括每个映射的基类。

    引用：[#1975](https://www.sqlalchemy.org/trac/ticket/1975)

+   **[sqlsoup]**

    与 map()、with_labels()、join()方法一起使用的映射可选择对象不再将给定参数放入内部“缓存”字典中。特别是因为 join()和 select()对象是在方法本身中创建的，这几乎是一种纯粹的内存泄漏行为。

### orm

+   **[orm]**

    修复了一个 bug，即在干净的对象上发生的非“mutable”属性修改事件，除了之前保存在“mutable changes”字典中的更改之外，不会强引用自身在标识映射中。这将导致对象被垃圾回收，丢失任何之前未保存在“mutable changes”字典中的更改的跟踪。

+   **[orm]**

    修复了一个 bug，即“passive_deletes='all'”在 flush 期间未将正确的符号传递给 lazy loaders，从而导致不必要的加载。

    参考：[#2013](https://www.sqlalchemy.org/trac/ticket/2013)

+   **[orm]**

    修复了一个 bug，阻止了复合映射属性在映射的选择语句中使用。请注意，复合的工作方式在 0.7 中将发生重大变化。

    参考：[#1997](https://www.sqlalchemy.org/trac/ticket/1997)

+   **[orm]**

    active_history 标志也添加到 composite()。该标志在 0.6 中没有效果，而是一个用于向前兼容性的占位符标志，因为它在 0.7 中适用于 composites。

    参考：[#1976](https://www.sqlalchemy.org/trac/ticket/1976)

+   **[orm]**

    修复了一个 uow bug，即当传递到 Session.delete() 的过期对象在删除对象时不考虑未加载的引用或集合，尽管 passive_deletes 仍保持默认值 False。

    参考：[#2002](https://www.sqlalchemy.org/trac/ticket/2002)

+   **[orm]**

    当继承的映射器上指定 version_id_col 时，如果继承的映射器已经有一个，并且这些列表达式不相同时，将发出警告。

    参考：[#1987](https://www.sqlalchemy.org/trac/ticket/1987)

+   **[orm]**

    如果链中的先前连接是外连接，则“innerjoin”标志不会沿着 joinedload() 连接链生效，从而允许正确返回没有引用子行的主行结果。

    参考：[#1954](https://www.sqlalchemy.org/trac/ticket/1954)

+   **[orm]**

    修复了关于“subqueryload”策略的 bug，即如果实体是 aliased() 构造，则策略将失败。

    参考：[#1964](https://www.sqlalchemy.org/trac/ticket/1964)

+   **[orm]**

    修复了关于“subqueryload”策略的 bug，即如果使用形式从 A->joined-subclass->C 的多级加载，则连接将失败。

    参考：[#2014](https://www.sqlalchemy.org/trac/ticket/2014)

+   **[orm]**

    修复了对 Query 对象按 -1 进行索引的错误。它错误地转换为空切片 -1:0，导致 IndexError。

    参考：[#1968](https://www.sqlalchemy.org/trac/ticket/1968)

+   **[orm]**

    mapper 参数“primary_key”可以传递为单个列，也可以传递为列表或元组。文档中以标量值示例的示例已更改为列表。

    参考：[#1971](https://www.sqlalchemy.org/trac/ticket/1971)

+   **[orm]**

    在 relationship() 和 column_property() 中添加了 active_history 标志，强制属性事件始终加载“旧”值，以便它在 attributes.get_history() 中可用。

    参考：[#1961](https://www.sqlalchemy.org/trac/ticket/1961)

+   **[orm]**

    如果复合键中的参数数量过大或过小，Query.get()将引发异常。

    参考：[#1977](https://www.sqlalchemy.org/trac/ticket/1977)

+   **[orm]**

    从 0.7 中回退了“优化的获取”修复，改进了联合继承“加载过期行”行为的生成。

    参考：[#1992](https://www.sqlalchemy.org/trac/ticket/1992)

+   **[orm]**

    在“primaryjoin”错误中增加了更多描述，对于一个不寻常的情况，即连���条件对于 viewonly 有效但对于非 viewonly 无效，并且未使用 foreign_keys - 在建议中添加了“foreign_keys”。还在通用的“direction”错误建议中添加了“foreign_keys”。

### 示例

+   **[examples]**

    版本示例现在支持检测关联关系中的更改。

### engine

+   **[engine]**

    “unicode 警告”只有在显式使用 Unicode 类型时才会引发，而不是在引擎或 String 类型上使用 convert_unicode=True 时。

+   **[engine]**

    修复了 Decimal 结果处理器的 C 版本中的内存泄漏。

    参考：[#1978](https://www.sqlalchemy.org/trac/ticket/1978)

+   **[engine]**

    为 RowProxy 的 C 版本实现了序列检查功能，以及对 RowProxy 的 2.7 风格“collections.Sequence”注册。

    参考：[#1871](https://www.sqlalchemy.org/trac/ticket/1871)

+   **[engine]**

    Threadlocal 引擎方法 rollback()、commit()、prepare()如果没有事务正在进行，则不会引发异常；这是 0.6 中引入的一个回归。

    参考：[#1998](https://www.sqlalchemy.org/trac/ticket/1998)

+   **[engine]**

    Threadlocal 引擎在 begin()、begin_nested()时返回自身；然后引擎实现了上下文管理器方法，以允许“with”语句。

    参考：[#2004](https://www.sqlalchemy.org/trac/ticket/2004)

### sql

+   **[sql]**

    修复了单个非关联运算符链的运算符优先级规则。即“x - (y - z)”将编译为“x - (y - z)”而不是“x - y - z”。也适用于标签，即“x - (y - z).label(‘foo’)”

    参考：[#1984](https://www.sqlalchemy.org/trac/ticket/1984)

+   **[sql]**

    Column 的‘info’属性在 Column.copy()期间被复制，即在声明性 mixin 中使用列时发生的情况。

    参考：[#1967](https://www.sqlalchemy.org/trac/ticket/1967)

+   **[sql]**

    为布尔值添加了一个绑定处理器，将其强制转换为 int，用于像 pymssql 这样的 DBAPI，它们会简单地对值调用 str()。

+   **[sql]**

    CheckConstraint 将在 copy()/tometadata()中复制其‘initially’、‘deferrable’和‘_create_rule’属性

    参考：[#2000](https://www.sqlalchemy.org/trac/ticket/2000)

### postgresql

+   **[postgresql]**

    单元素元组表达式在 IN 子句中正确地加括号，也来自

    参考：[#1984](https://www.sqlalchemy.org/trac/ticket/1984)

+   **[postgresql]**

    确保每个数字、浮点数、整数代码、标量+数组都被 psycopg2 和 pg8000 的“numeric”基本类型识别。

    引用：[#1955](https://www.sqlalchemy.org/trac/ticket/1955)

+   **[postgresql]**

    添加了 as_uuid=True 标志到 UUID 类型，将接收和返回 Python UUID()对象而不是字符串。目前，UUID 类型只能与 psycopg2 一起使用。

    引用：[#1956](https://www.sqlalchemy.org/trac/ticket/1956)

+   **[postgresql]**

    修复了在池销毁+重新创建后，对不支持 ENUM 的 PG 版本会出现 KeyError 的错误。

    引用：[#1989](https://www.sqlalchemy.org/trac/ticket/1989)

### mysql

+   **[mysql]**

    修复了 Jython + zxjdbc 的错误处理，使 has_table()属性再次有效。从 0.6.3 开始的退化（我们没有 Jython buildbot，抱歉）

    引用：[#1960](https://www.sqlalchemy.org/trac/ticket/1960)

### sqlite

+   **[sqlite]**

    在包含远程模式的 CREATE TABLE 中的 REFERENCES 子句指向具有相同模式名称的另一表时，现在呈现远程名称而不带模式子句，这是 SQLite 所要求的。

    引用：[#1851](https://www.sqlalchemy.org/trac/ticket/1851)

+   **[sqlite]**

    在同一主题下，包含远程模式的 CREATE TABLE 中的 REFERENCES 子句指向与父表不同的模式，由于不支持跨模式引用，因此根本不会呈现。

### mssql

+   **[mssql]**

    不幸地，索引反射的重写没有经过正确测试，返回了不正确的结果。这个退化现在已经修复。

    引用：[#1770](https://www.sqlalchemy.org/trac/ticket/1770)

### oracle

+   **[oracle]**

    cx_oracle 的“十进制检测”逻辑，用于具有模糊数值特征的结果集列，现在使用由 locale/NLS_LANG 设置确定的小数点字符，使用首次连接时检测此字符。在使用非句点小数点 NLS_LANG 设置时，还需要 cx_oracle 5.0.3 或更高版本。

    引用：[#1953](https://www.sqlalchemy.org/trac/ticket/1953)

### 杂项

+   **[firebird]**

    Firebird 数字类型现在明确检查 Decimal，让 float()直接通过，从而允许特殊值如 float(‘inf’)。

    引用：[#2012](https://www.sqlalchemy.org/trac/ticket/2012)

+   **[declarative]**

    如果 __table_args__ 不是元组或字典格式，并且不是 None，则会引发错误。

    引用：[#1972](https://www.sqlalchemy.org/trac/ticket/1972)

+   **[sqlsoup]**

    向 SqlSoup 添加了“map_to()”方法，这是一个“主”方法，接受每个可选择和映射的显式参数，包括每个映射的基类。

    引用：[#1975](https://www.sqlalchemy.org/trac/ticket/1975)

+   **[sqlsoup]**

    与 map()、with_labels()、join()方法一起使用的映射可选择现在不再将给定参数放入内部“缓存”字典中。特别是由于 join()和 select()对象是在方法本身中创建的，这几乎是一种纯粹的内存泄漏行为。

## 0.6.5

发布日期：2010 年 10 月 24 日星期日

### orm

+   **[orm]**

    添加了一个新的“lazyload”选项“immediateload”。当对象被填充时，自动执行常规的“lazy”加载操作。这里的用例是当加载对象以放置在离线缓存中，或在会话不可用后使用对象时，以及希望使用直接的‘select’加载而不是‘joined’或‘subquery’时。

    参考：[#1914](https://www.sqlalchemy.org/trac/ticket/1914)

+   **[orm]**

    新的 Query 方法：query.label(name)、query.as_scalar()，返回查询的语句作为标量子查询，带/不带标签；query.with_entities(*ent)，用新的实体替换查询的 SELECT 列表。与 query.values() 的生成形式大致等效，接受映射实体以及列表达式。

    参考：[#1920](https://www.sqlalchemy.org/trac/ticket/1920)

+   **[orm]**

    修复了递归错误，当将对象从一个引用移动到另一个引用时，涉及到反向引用，其中启动父类是前一个父类的子类（带有自己的映射器）时，可能会出现递归错误。

+   **[orm]**

    修复了在 0.6.4 中出现的一个回归，如果你在 mapper() 上传递一个空列表给“include_properties”，就会出现这个问题。

    参考：[#1918](https://www.sqlalchemy.org/trac/ticket/1918)

+   **[orm]**

    修复了 Query 中的标记错误，即如果任何列表达式未标记，则 NamedTuple 会错误地应用标签。

+   **[orm]**

    修补了一个情况，其中 query.join() 会不适当地将右侧适应为左侧连接的右侧

    参考：[#1925](https://www.sqlalchemy.org/trac/ticket/1925)

+   **[orm]**

    Query.select_from() 已经增强，以确保后续对 query.join() 的调用将使用 select_from() 实体，假设它是一个映射实体而不是一个普通的可选择对象，作为默认的“左”侧，而不是查询对象的实体列表中的第一个实体。

+   **[orm]**

    当会话在子事务回滚后（这是在 autocommit=False 模式下发生刷新失败时发生的情况）被用于后续操作时，会话引发的异常现在已经被重新表述了（这是“由于子事务中的回滚而无效”消息）。特别是，如果回滚是由于刷新期间的异常引起的，消息将说明这是情况，并重申在刷新期间发生的原始异常的字符串形式。如果会话由于显式使用子事务而关闭（这种情况并不常见），消息只说明这是情况。

+   **[orm]**

    当初始化已经失败后，如果对 Mapper 进行重复请求，则不再假设“hasattr”情况，因为这个消息被发出的其他场景，并且消息也不会多次叠加 - 每次尝试使用时都会得到相同的消息。误用的“compiles”正在被“initialize”替换。

+   **[orm]**

    修复了在 query.update() 中的错误，在这里‘evaluate’或‘fetch’过期将失败，如果列表达式键是具有不同键名的类属性作为实际列名。

    参考：[#1935](https://www.sqlalchemy.org/trac/ticket/1935)

+   **[orm]**

    在 flush 过程中添加了一个断言，确保“新持久化”对象上没有生成持有 NULL 的标识键。当用户定义的代码无意中触发了对未完全加载的对象的 flush 时，就会出现这种情况。

+   **[orm]**

    当关系属性的延迟加载使用当前状态而不是“已提交”状态的外键和主键属性发出 SQL 时，如果没有进行 flush，则现在会使用当前状态。以前，只会使用数据库已提交的状态。特别是，这会导致许多对一的 get()-on-lazyload 操作失败，因为在这些加载时不会触发自动 flush，当确定属性时，“已提交”状态可能不可用。

    参考：[#1910](https://www.sqlalchemy.org/trac/ticket/1910)

+   **[orm]**

    relationship() 上的一个新标志 load_on_pending，允许延迟加载器在未进行 flush 的情况下对待待定对象进行触发，以及手动“附加”到会话的瞬态对象。请注意，此标志在加载对象时阻止属性事件发生，因此直到 flush 之后才可用反向引用。该标志仅用于非常特定的用例。

+   **[orm]**

    另一个 relationship() 上的新标志 cascade_backrefs，在“反向”关系的事件被初始化时禁用“save-update”级联。这是一种更干净的行为，使得可以在瞬态对象上设置许多对一，而不会被吸入子对象的会话，同时仍允许前向集合级联。我们*可能*会在 0.7 中将其默认设置为 False。

+   **[orm]**

    在关系的“passive_updates=False”行为略有改进，当仅放在关系的多对一侧时；文档已经澄清 passive_updates=False 应该真正放在一对多侧。

+   **[orm]**

    在多对一上放置 passive_deletes=True 会发出警告，因为您可能打算将其放在一对多侧。

+   **[orm]**

    修复了一个 bug，该 bug 会导致“subqueryload”与子类的关系在单表继承中无法正常工作 - “where type in (x, y, z)” 只会放在内部，而不是重复出现。

+   **[orm]**

    在使用 from_self() 与单表继承时，“where type in (x, y, z)” 只会放在查询的外部，而不是重复出现。可能会对此进行一些调整。

+   **[orm]**

    当 configure() 被调用时，scoped_session 会发出警告，如果已经存在一个 Session（仅检查当前线程）

    参考：[#1924](https://www.sqlalchemy.org/trac/ticket/1924)

+   **[orm]**

    重新设计了 mapper.cascade_iterator() 的内部结构，在某些情况下减少了约 9% 的方法调用。

    参考：[#1932](https://www.sqlalchemy.org/trac/ticket/1932)

### 引擎

+   **[engine]**

    修复了 0.6.4 中的一个回归，即允许一致引发游标错误的更改破坏了 result.lastrowid 访问器。为 result.lastrowid 添加了测试覆盖。请注意，lastrowid 仅受 Pysqlite 和一些 MySQL 驱动程序支持，因此在一般情况下并不是特别有用。

+   **[engine]**

    当首次使用连接时，引擎发出的日志消息现在是“BEGIN (implicit)”，以强调 DBAPI 没有显式的 begin()。

+   **[engine]**

    在 metadata.reflect()中添加了“views=True”选项，将添加可用视图列表到被反射的视图中。

    参考：[#1936](https://www.sqlalchemy.org/trac/ticket/1936)

+   **[engine]**

    engine_from_config()现在接受‘debug’作为‘echo’，‘echo_pool’的‘force’，‘convert_unicode’的布尔值作为‘use_native_unicode’。

    参考：[#1899](https://www.sqlalchemy.org/trac/ticket/1899)

### sql

+   **[sql]**

    修复了 TypeDecorator 中的错误，其中特定于方言的类型被拉入以生成给定类型的 DDL，这并不总是返回正确的结果。

+   **[sql]**

    TypeDecorator 现在可以将一个完全构造的类型指定为其“impl”，而不仅仅是一个类型类。

+   **[sql]**

    TypeDecorator 现在将自身放置为二进制表达式的结果类型，其中类型强制转换规则通常会返回其 impl 类型 - 以前，将返回 impl 类型的副本，该副本将 TypeDecorator 嵌入其中作为“dialect” impl，这可能是实现所需效果的无意的方式。

+   **[sql]**

    TypeDecorator.load_dialect_impl()默认返回“self.impl”，即不返回“self.impl”的方言实现类型。这是为了正确支持编译。行为可以像以前一样被用户覆盖，产生相同的效果。

+   **[sql]**

    添加了 type_coerce(expr, type_)表达式元素。在评估表达式和处理结果行时，将给定表达式视为给定类型，但不影响 SQL 的生成，除了一个匿名标签。

+   **[sql]**

    Table.tometadata()现在也复制与 Table 关联的 Index 对象。

+   **[sql]**

    Table.tometadata()如果给定的 Table 已经存在于目标 MetaData 中，则会发出警告 - 返回现有的 Table 对象。

+   **[sql]**

    如果一个尚未分配名称的列（即在声明中）在导出到封闭 select()构造的列集合的上下文中使用，或者在分配其名称之前编译涉及该列的任何构造时，将引发一个信息性错误消息。

+   **[sql]**

    as_scalar()，label()可以在包含尚未命名的列的可选择项上调用。

    参考：[#1862](https://www.sqlalchemy.org/trac/ticket/1862)

+   **[sql]**

    修复了在操作两个类型均为“NullType”但不是单例 NULLTYPE 实例时可能发生的递归溢出。

    参考：[#1907](https://www.sqlalchemy.org/trac/ticket/1907)

### postgresql

+   **[postgresql]**

    为 ARRAY 类型添加了“as_tuple”标志，返回结果作为元组而不是列表以允许哈希。

+   **[postgresql]**

    修复了阻止从自定义类型（如“enum”）构建的“domain”被反射的错误。

    参考：[#1933](https://www.sqlalchemy.org/trac/ticket/1933)

### mysql

+   **[mysql]**

    修复了涉及使用 ON UPDATE 子句的 CURRENT_TIMESTAMP 默认值的反射错误，感谢 Taavi Burns。

    参考：[#1940](https://www.sqlalchemy.org/trac/ticket/1940)

### mssql

+   **[mssql]**

    修复了未正确处理未知类型反射的错误。

    参考：[#1946](https://www.sqlalchemy.org/trac/ticket/1946)

+   **[mssql]**

    修复了使用“schema”别名表时无法正确编译的错误。

    参考：[#1943](https://www.sqlalchemy.org/trac/ticket/1943)

+   **[mssql]**

    重写了索引的反射，使用 sys.目录，以便可以反射任何配置的列名称（空格、嵌入逗号等）。请注意，反射索引需要 SQL Server 2005 或更高版本。

    参考：[#1770](https://www.sqlalchemy.org/trac/ticket/1770)

+   **[mssql]**

    mssql+pymssql 方言现在尊重 URL 中的“port”���分，而不是丢弃它。

    参考：[#1952](https://www.sqlalchemy.org/trac/ticket/1952)

### oracle

+   **[oracle]**

    现在无论检测到的 Oracle 版本如何，create_engine()中的 implicit_returning 参数都会被尊重。以前，如果服务器版本信息<10，则该标志将被强制为 False。

    参考：[#1878](https://www.sqlalchemy.org/trac/ticket/1878)

### 测试

+   **[测试]**

    NoseSQLAlchemyPlugin 已移至新包“sqlalchemy_nose”，与“sqlalchemy”一起安装。这样，“nosetests”脚本仍然可以正常工作，但也允许在导入 SQLAlchemy 模块之前打开覆盖率，从而使覆盖率能够正确工作。

### 杂项

+   **[声明式]**

    @classproperty（即将/现在 @declared_attr）对于不是混合类的基类上的 __mapper_args__、__table_args__、__tablename__ 以及混合类都生效。

    参考：[#1922](https://www.sqlalchemy.org/trac/ticket/1922)

+   **[声明式]**

    @classproperty 在声明式中的官方名称/位置是 sqlalchemy.ext.declarative.declared_attr。虽然是同一件事情，但由于它更像是一个特定于声明式的“标记”，而不仅仅是一个属性技术，所以将其移动到那里。

    参考：[#1915](https://www.sqlalchemy.org/trac/ticket/1915)

+   **[声明式]**

    修复了混合类上的列无法正确传播到单表或联接表继承方案的错误，其中属性名称与列的名称不同。

    参考：[#1930](https://www.sqlalchemy.org/trac/ticket/1930)，[#1931](https://www.sqlalchemy.org/trac/ticket/1931)

+   **[声明式]**

    现在混合类可以指定一个列，该列覆盖了与超类关联的同名列。感谢 Oystein Haaland。

+   **[informix]**

    对 Informix 方言进行了*重大*清理/现代化，感谢 Florian Apolloner。

    引用：[#1906](https://www.sqlalchemy.org/trac/ticket/1906)

+   **【杂项】**

    CircularDependencyError 现在具有 .cycles 和 .edges 成员，它们是一个或多个循环中涉及的元素集合，以及 2 元组的边集合。

    引用：[#1890](https://www.sqlalchemy.org/trac/ticket/1890)

### orm

+   **【orm】**

    添加了一个新的“lazyload”选项“immediateload”。在对象填充时自动发出通常的“lazy”加载操作。这里的用例是在加载要放置在离线缓存中的对象或在会话不可用后使用，并且希望进行直接的“select”加载，而不是“joined”或“subquery”。

    引用：[#1914](https://www.sqlalchemy.org/trac/ticket/1914)

+   **【orm】**

    新的 Query 方法：query.label(name)，query.as_scalar()，将查询的语句返回为带有/不带有标签的标量子查询；query.with_entities(*ent)，用新实体替换查询的 SELECT 列表。大致相当于 query.values() 的生成形式，它接受映射实体以及列表达式。

    引用：[#1920](https://www.sqlalchemy.org/trac/ticket/1920)

+   **【orm】**

    修复了递归错误，该错误可能在将对象从一个引用移动到另一个引用时发生，并涉及反向引用，其中发起父级是前一个父级的子类（具有自己的映射器）。

+   **【orm】**

    修复了在 0.6.4 中发生的回归，如果您将空列表传递给“include_properties”在 mapper() 上

    引用：[#1918](https://www.sqlalchemy.org/trac/ticket/1918)

+   **【orm】**

    修复了查询中的标签错误，在其中，如果任何列表达式未标记，则命名元组会错误地应用标签。

+   **【orm】**

    修复了 query.join() 在不适当地将右侧适应为左侧连接的右侧的情况下的情况

    引用：[#1925](https://www.sqlalchemy.org/trac/ticket/1925)

+   **【orm】**

    Query.select_from() 已经得到加强，以确保后续调用 query.join() 将使用 select_from() 实体，假设它是一个映射实体而不是一个普通可选择项，并且默认“左”侧，而不是查询对象的实体列表中的第一个实体。

+   **【orm】**

    当 Session 在子事务回滚后（这是在 autocommit=False 模式下刷新失败时发生的情况）后续使用时引发的异常已经被重新措辞（这是“由于子事务回滚而不活跃”消息）。特别地，如果回滚是由于刷新期间发生异常引起的，则消息说明了这一点，并且重申了在刷新期间发生的原始异常的字符串形式。如果会话由于明确使用子事务而关闭（并不常见），则消息只是说明了这一点。

+   **【orm】**

    当 Mapper 在初始化失败后再次对其进行重复请求时，引发的异常不再假定“hasattr”情况，因为还有其他情况会导致该消息被发出，并且该消息也不会多次复合 - 每次尝试使用时都会得到相同的消息。误称“compiles”正在被“initialize”交换。

+   **[orm]**

    修复了在 query.update() 中的一个 bug，即当列表达式键是一个具有不同键名的类属性时，“evaluate”或“fetch”过期会失败。

    参考：[#1935](https://www.sqlalchemy.org/trac/ticket/1935)

+   **[orm]**

    在刷新过程中添加了一个断言，确保“新持久化”对象上没有生成包含 NULL 的标识键。当用户定义的代码无意中触发了对尚未完全加载的对象的刷新时，就会发生这种情况。

+   **[orm]**

    当发出 SQL 时，关系属性的惰性加载现在使用当前状态，而不是“已提交”状态，如果刷新未在进行中。以前，只会使用数据库提交的状态。特别是，这将导致许多对一的 get()-on-lazyload 操作失败，因为这些加载时不会触发自动刷新，并且可能无法使用“已提交”状态。

    参考：[#1910](https://www.sqlalchemy.org/trac/ticket/1910)

+   **[orm]**

    在 relationship() 上的一个新标志，load_on_pending，允许懒加载器在不进行刷新的情况下对待挂起的对象进行处理，以及手动“附加”到会话的临时对象。请注意，当加载对象时，此标志会阻止属性事件发生，因此在刷新后才能使用反向引用。该标志仅用于非常特定的用例。

+   **[orm]**

    在 relationship() 上的另一个新标志，cascade_backrefs，在双向关系的“反向”一侧启动事件时禁用了“save-update”级联。这是一种更清晰的行为，以便可以在临时对象上设置多对一，而不会被吸入到子对象的会话中，同时仍允许向前集合进行级联。我们*可能*会在 0.7 中将其默认设置为 False。

+   **[orm]**

    当 passive_updates=False 仅放置在关系的多对一一侧时，对“passive_updates=False”的行为进行了轻微改进；文档已经澄清，passive_updates=False 实际上应该放在一对多的一侧。

+   **[orm]**

    在多对一上放置 passive_deletes=True 会发出警告，因为您可能打算将其放在一对多的一侧。

+   **[orm]**

    修复了一个 bug，该 bug 会阻止“subqueryload”与单表继承一起使用时正常工作，即从子类到关系的关系 - “where type in (x, y, z)” 只会放置在内部，而不是重复出现。

+   **[orm]**

    在使用单表继承时，from_self()中的“where type in (x, y, z)”仅放在查询的外部，而不是重复出现。可能需要对此进行一些调整。

+   **[orm]**

    如果已经存在一个 Session（仅检查当前线程），则 scoped_session 在调用 configure()时会发出警告。

    参考：[#1924](https://www.sqlalchemy.org/trac/ticket/1924)

+   **[orm]**

    重新设计了 mapper.cascade_iterator()的内部结构，在某些情况下减少了约 9%的方法调用。

    参考：[#1932](https://www.sqlalchemy.org/trac/ticket/1932)

### 引擎

+   **[engine]**

    修复了 0.6.4 中的一个回归问题，即允许一致引发游标错误的更改破坏了 result.lastrowid 访问器。为 result.lastrowid 添加了测试覆盖。请注意，lastrowid 仅由 Pysqlite 和一些 MySQL 驱动程序支持，在一般情况下并不是特别有用。

+   **[engine]**

    当连接首次被使用时，引擎发出的日志消息现在是“BEGIN (implicit)”，以强调 DBAPI 没有显式的 begin()。

+   **[engine]**

    在 metadata.reflect()中添加了“views=True”选项，将可用视图列表添加到要反射的视图中。

    参考：[#1936](https://www.sqlalchemy.org/trac/ticket/1936)

+   **[engine]**

    engine_from_config()现在接受‘debug’作为‘echo’的‘echo_pool’，‘force’作为‘convert_unicode’，布尔值作为‘use_native_unicode’。

    参考：[#1899](https://www.sqlalchemy.org/trac/ticket/1899)

### sql

+   **[sql]**

    修复了 TypeDecorator 中的一个错误，即方言特定类型被引入以生成给定类型的 DDL，这并不总是返回正确的结果。

+   **[sql]**

    TypeDecorator 现在可以指定一个完全构造的类型作为其“impl”，而不仅仅是一个类型类。

+   **[sql]**

    TypeDecorator 现在会将自身作为二进制表达式的结果类型，其中类型强制转换规则通常会返回其实现类型 - 以前，会返回一个 impl 类型的副本，其中 TypeDecorator 被嵌入其中作为“方言”实现，这可能是一种无意中实现所需效果的方式。

+   **[sql]**

    TypeDecorator.load_dialect_impl()默认返回“self.impl”，即不返回“self.impl”的方言实现类型。这样可以支持正确的编译。行为可以像以前一样被用户覆盖，产生相同的效果。

+   **[sql]**

    添加了 type_coerce(expr, type_)表达式元素。在评估表达式和处理结果行时，将给定表达式视为给定类型，但不影响 SQL 的生成，除了一个匿名标签。

+   **[sql]**

    Table.tometadata()现在也会复制与 Table 相关联的 Index 对象。

+   **[sql]**

    Table.tometadata()如果给定的 Table 已经存在于目标 MetaData 中，则会发出警告 - 返回现有的 Table 对象。

+   **[sql]**

    如果一个尚未分配名称的 Column，在声明时使用，在导出到封闭的 select() 构造的 columns 集合中使用，或者在分配名称之前编译涉及该列的任何构造时，会引发一个信息性错误消息。

+   **[sql]**

    `as_scalar()`、`label()` 可以在包含尚未命名的 Column 的可选对象上调用。

    参考：[#1862](https://www.sqlalchemy.org/trac/ticket/1862)

+   **[sql]**

    修复了操作两个类型均为 “NullType” 的表达式时可能发生的递归溢出，但不是单例 NULLTYPE 实例。

    参考：[#1907](https://www.sqlalchemy.org/trac/ticket/1907)

### postgresql

+   **[postgresql]**

    在 ARRAY 类型中添加了 “as_tuple” 标志，返回结果为元组而不是列表，以允许哈希。

+   **[postgresql]**

    修复了一个 bug，阻止了从自定义类型（如 “enum”）构建的 “domain” 反射。

    参考：[#1933](https://www.sqlalchemy.org/trac/ticket/1933)

### mysql

+   **[mysql]**

    修复了与 ON UPDATE 子句一起使用的 CURRENT_TIMESTAMP 默认值的反射 bug，感谢 Taavi Burns。

    参考：[#1940](https://www.sqlalchemy.org/trac/ticket/1940)

### mssql

+   **[mssql]**

    修复了一个反射 bug，未能正确处理未知类型的反射。

    参考：[#1946](https://www.sqlalchemy.org/trac/ticket/1946)

+   **[mssql]**

    修复了带有 “schema” 别名的表的别名化可能无法正确编译的 bug。

    参考：[#1943](https://www.sqlalchemy.org/trac/ticket/1943)

+   **[mssql]**

    重写了索引的反射，以使用 sys. 目录，以便可以反射任何配置的列名（空格，嵌入逗号等）。请注意，反射索引需要 SQL Server 2005 或更高版本。

    参考：[#1770](https://www.sqlalchemy.org/trac/ticket/1770)

+   **[mssql]**

    mssql+pymssql 方言现在会尊重 URL 中的 “port” 部分，而不是将其丢弃。

    参考：[#1952](https://www.sqlalchemy.org/trac/ticket/1952)

### oracle

+   **[oracle]**

    现在，不管 Oracle 检测到的版本如何，`create_engine()` 的 `implicit_returning` 参数都会被正确处理。以前，如果服务器版本信息小于 10，则该标志会被强制设为 False。

    参考：[#1878](https://www.sqlalchemy.org/trac/ticket/1878)

### 测试

+   **[tests]**

    NoseSQLAlchemyPlugin 已移至新包 “sqlalchemy_nose”，该包随 “sqlalchemy” 一起安装。这样，“nosetests” 脚本仍然可以正常工作，但也允许使用 --with-coverage 选项在导入 SQLAlchemy 模块之前打开覆盖，从而使覆盖工作正常。

### misc

+   **[declarative]**

    @classproperty（即将/现在 @declared_attr）对于不是 mixin 的基类上的 __mapper_args__、__table_args__、__tablename__ 以及 mixins 都生效。

    参考：[#1922](https://www.sqlalchemy.org/trac/ticket/1922)

+   **[declarative]**

    @classproperty 的官方名称/位置，用于与 declarative 一起使用的是 sqlalchemy.ext.declarative.declared_attr。虽然是同一件事情，但由于它更像是一个特定于 declarative 的“标记”，而不仅仅是一个属性技术，所以将其移动到那里。

    参考：[#1915](https://www.sqlalchemy.org/trac/ticket/1915)

+   **[declarative]**

    修复了一个 bug，即 mixin 上的列无法正确传播到单表或联合表继承方案中，其中属性名称与列的名称不同。

    参考：[#1930](https://www.sqlalchemy.org/trac/ticket/1930), [#1931](https://www.sqlalchemy.org/trac/ticket/1931)

+   **[declarative]**

    现在 mixin 可以指定一个列，该列覆盖了与超类关联的同名列。感谢 Oystein Haaland。

+   **[informix]**

    *重大*清理/现代化 Informix 方言为 0.6，由 Florian Apolloner 提供。

    参考：[#1906](https://www.sqlalchemy.org/trac/ticket/1906)

+   **[misc]**

    CircularDependencyError 现在具有.cycles 和.edges 成员，它们是一个或多个循环中涉及的元素集合，以及作为 2 元组的边集合。

    参考：[#1890](https://www.sqlalchemy.org/trac/ticket/1890)

## 0.6.4

发布日期：Tue Sep 07 2010

### orm

+   **[orm]**

    ConcurrentModificationError 的名称已更改为 StaleDataError，并且描述性错误消息已经修订，以准确反映问题所在。在可预见的未来，这两个名称都将保持可用，以供可能在“except:”子句中指定 ConcurrentModificationError 的方案使用。

+   **[orm]**

    向 identity map 添加了一个互斥锁，该互斥锁对迭代方法中的删除操作进行了互斥，现在在返回可迭代对象之前会预先缓冲。这是因为异步 gc 可以随时通过 gc 线程删除项目。

    参考：[#1891](https://www.sqlalchemy.org/trac/ticket/1891)

+   **[orm]**

    Session 类现在存在于 sqlalchemy.orm.*中。我们正在摆脱 create_session()的使用，该方法具有非标准的默认值，用于那些需要一步构造 Session 的情况。然而，大多数用户应该继续使用 sessionmaker()进行一般用途。

+   **[orm]**

    query.with_parent()现在接受瞬态对象，并将使用它们的主键/外键属性的非持久化值来制定条件。文档也对 with_parent()的目的进行了澄清。

+   **[orm]**

    mapper()的 include_properties 和 exclude_properties 参数现在除了字符串外，还接受列对象作为成员。这样，可以消除 join()中的同名列对象等情况。

+   **[orm]**

    如果对包含多个具有相同名称的列的 join 或其他单个可选择项创建了映射器，并且这些列没有明确命名为相同或不同的属性（或排除），则现在会发出警告。在 0.7 版本中，此警告将变为异常。请注意，当组合发生在继承的结果时，不会发出此警告，因此属性仍然可以自然地被覆盖。在 0.7 版本中，这将进一步改进。

    参考：[#1896](https://www.sqlalchemy.org/trac/ticket/1896)

+   **[orm]**

    mapper()的 primary_key 参数现在可以指定一系列列，这些列仅是映射可选择项的计算“主键”列的子集，而不会引发错误。这有助于处理情况，其中可选择项的有效主键比实际标记为“主键”的可选择项中的列数更简单，例如在两个表的主键列上进行连接。

    参考：[#1896](https://www.sqlalchemy.org/trac/ticket/1896)

+   **[orm]**

    现在已删除的对象会得到一个名为‘deleted’的标志，这会阻止将对象重新添加到会话中，因为以前对象会在其属性被访问之前悄悄地存在于标识映射中。make_transient()函数现在会重置此标志以及“key”标志。

+   **[orm]**

    make_transient()可以安全地在已经是瞬态实例上调用。

+   **[orm]**

    如果在 mapper()中没有在映射的可选择项中或在 with_polymorphic 可选择项中以直接或派生形式存在 polymorphic_on 列，则会发出警告，而不是默默地忽略它。预计在 0.7 版本中将变为异常。

+   **[orm]**

    当 relationship()配置具有模糊参数时，再次查看发出的一系列错误消息。现在不再提到“foreign_keys”设置，因为几乎不需要，最好用户设置正确的 ForeignKey 元数据，这是现在的建议。如果使用了‘foreign_keys’并且不正确，消息会建议该属性可能是不必要的。增加了属性的文档。这是因为 ML 上所有困惑的 relationship()用户似乎都试图使用 foreign_keys，因为消息只会让他们更加困惑，因为 Table 元数据更加清晰。

+   **[orm]**

    如果“secondary”表没有 ForeignKey 元数据并且没有设置 foreign_keys，即使用户传递了错误的信息，也假定 primary/secondaryjoin 表达式应仅考虑“secondary”中的所有列为外键。在任何情况下，“secondary”中的外键都不可能在其他地方。现在发出警告而不是错误，并且映射成功。

    参考：[#1877](https://www.sqlalchemy.org/trac/ticket/1877)

+   **[orm]**

    将一个 o2m 对象从一个集合移动到另一个集合，或者通过 m2o 更改引用对象，其中外键也是主键的成员，现在在 flush 期间将更加仔细地检查，如果“多”一侧的外键值的变化是由于“一”一侧主键的变化引起的，或者如果“一”只是一个不同的对象。在一个情况下，具有级联功能的数据库可能已经级联了该值，我们需要查看“新”的 PK 值来执行 UPDATE，在另一种情况下，我们需要继续查看“旧”的 PK 值。我们现在查看“旧”的 PK 值，假设 passive_updates=True，除非我们知道触发更改的是 PK 切换。

    参考：[#1856](https://www.sqlalchemy.org/trac/ticket/1856)

+   **[orm]**

    version_id_col 的值可以手动更改，这将导致行的 UPDATE。版本化的 UPDATE 和 DELETE 现在使用 version_id_col 的“已提交”值作为 WHERE 子句，而不是挂起的更改值。如果属性上存在手动更改，则版本生成器也会被绕过。

    参考：[#1857](https://www.sqlalchemy.org/trac/ticket/1857)

+   **[orm]**

    修复了与具体继承映射器一起使用 merge()时的问题。这样的映射器经常具有所谓的“具体”属性，即“禁用”从父类传播的子类属性 - 这些属性需要允许 merge()操作无效。

+   **[orm]**

    为 column_mapped_collection 指定非基于列的参数，包括字符串、text()等，将引发一个错误消息，明确要求一个列元素，不再提供关于 text()或 literal()的错误信息。

    参考：[#1863](https://www.sqlalchemy.org/trac/ticket/1863)

+   **[orm]**

    同样地，对于 relationship()、foreign_keys、remote_side、order_by 等 - 所有基于列的表达式都是强制执行的 - 字符串列表明确禁止，因为这是一个非常常见的错误。

+   **[orm]**

    动态属性不支持集合填充 - 当调用 set_committed_value()时添加了一个断言，以及当将 joinedload()或 subqueryload()选项应用于动态属性时，而不是失败/静默失败。

    参考：[#1864](https://www.sqlalchemy.org/trac/ticket/1864)

+   **[orm]**

    修复了一个 bug，即从一个具有相同列但具有不同标签名称的 Query 生成的 Query，在某些 UNION 情况下通常会失败，无法完全将内部列传播到外部查询。

    参考：[#1852](https://www.sqlalchemy.org/trac/ticket/1852)

+   **[orm]**

    当提供一个未映射的实例时，object_session()会引发正确的 UnmappedInstanceError。

    参考：[#1881](https://www.sqlalchemy.org/trac/ticket/1881)

+   **[orm]**

    对计算的 Mapper 属性应用了进一步的记忆化，显著减少了在高度多态映射配置中的运行时 mapper.py 调用次数（约 90%）。

+   **[orm]**

    由版本控制示例使用的 mapper _get_col_to_prop 私有方法已经过时；现在请使用 mapper.get_property_by_column()，这将保持为此公共方法。

+   **[ORM]**

    如果在以前是 NULL 的列上进行版本控制，那么版本示例现在可以正确地工作了。

### 示例

+   **[示例]**

    beaker_caching 示例已经重新组织，使得 Session、缓存管理器、declarative_base 成为环境的一部分，并且自定义的缓存代码是可移植的，现在在“caching_query.py”中。这样可以让示例更容易“插入”到现有项目中。

+   **[示例]**

    history_meta 版本控制示例在复制列时设置了“unique=False”，这样版本控制表就可以处理具有重复值的多行。

    参考：[#1887](https://www.sqlalchemy.org/trac/ticket/1887)

### 引擎

+   **[引擎]**

    对已经耗尽、已经关闭或者不是返回结果的结果集调用 fetchone() 或类似方法现在会在所有情况下都引发 ResourceClosedError 错误，这是 InvalidRequestError 的子类，无论后端如何。以前，一些 DBAPI 会引发 ProgrammingError（例如 pysqlite），其他一些则会返回 None，导致下游出现故障（例如 MySQL-python）。

+   **[引擎]**

    修复了 Connection 中的一个 bug，即如果在第一个连接池连接的“初始化”阶段发生了“断开”事件，那么当 Connection 尝试使 DBAPI 连接无效时会引发 AttributeError。

    参考：[#1894](https://www.sqlalchemy.org/trac/ticket/1894)

+   **[引擎]**

    Connection、ResultProxy 以及 Session 现在对于所有“此连接/事务/结果已关闭”类型的错误使用 ResourceClosedError。

+   **[引擎]**

    Connection.invalidate() 可以被多次调用，后续调用不会产生任何效果。

### SQL

+   **[SQL]**

    对 alias() 构造调用 execute() 方法将在 0.7 版本中待废弃，因为它本身不是一个“可执行”的构造。它当前“代理”其内部元素，并且条件上是“可执行”的，但这不是我们当前喜欢的模糊性。

+   **[SQL]**

    ClauseElement 的 execute() 和 scalar() 方法现在已经适当地移动到了 Executable 子类。ClauseElement.execute()/ scalar() 仍然存在，并在 0.7 版本中待废弃，但请注意，如果你不是 Executable（除非你是 alias()，请参阅前面的注释），这些方法总是会引发错误。

+   **[SQL]**

    为 Numeric->Integer 添加了基本的数学表达式强制转换，以便无论表达式的方向如何，结果类型都是 Numeric。

+   **[SQL]**

    更改了在 Column 上使用“index=True”标志生成截断的“auto”索引名称的方案。截断只对自动生成的名称起作用，不适用于用户定义的名称（会引发错误），截断方案本身现在基于标识符名称的 md5 哈希的片段，这样具有相似名称的多个列的索引仍然具有唯一的名称。

    参考：[#1855](https://www.sqlalchemy.org/trac/ticket/1855)

+   **[sql]**

    生成的索引名称也是基于“最大索引名称长度”属性的，这个属性与“最大标识符长度”是分开的 - 这是为了迎合 MySQL，因为 MySQL 对索引名称的最大长度为 64，与其总体最大长度 255 是分开的。

    参考：[#1412](https://www.sqlalchemy.org/trac/ticket/1412)

+   **[sql]**

    如果将 text() 构造放置在面向列的情况下，它至少会返回 NULLTYPE 作为其类型，而不是 None，这使得它可以比以前更自由地用于临时列表达式。然而，literal_column() 仍然是更好的选择。

+   **[sql]**

    在 ForeignKey 无法解析目标时，错误消息中添加了父表/列、目标表/列的完整描述。

+   **[sql]**

    修复了一个 bug，即在反射表中替换复合外键列会导致尝试第二次从表中删除反射的约束，从而引发 KeyError。

    参考：[#1865](https://www.sqlalchemy.org/trac/ticket/1865)

+   **[sql]**

    _Label 构造，即每当你说 somecol.label() 时产生的构造，现在在其“proxy_set”中计算自身与其包含列的代理集的并集，而不是直接返回包含列的代理集。这允许依赖于 _Labels 本身身份的列对应操作返回正确的结果。

+   **[sql]**

    修复 ORM bug。

    参考：[#1852](https://www.sqlalchemy.org/trac/ticket/1852)

### postgresql

+   **[postgresql]**

    修复了 psycopg2 方言，使用其 set_isolation_level() 方法，而不是依赖于基本的 “SET SESSION ISOLATION” 命令，因为否则 psycopg2 在每个新事务中重置隔离级别。

### mssql

+   **[mssql]**

    修复了与 pymssql 后端一起使用“默认模式”查询的问题。

### oracle

+   **[oracle]**

    在 Oracle 方言中添加了 ROWID 类型，用于那些可能需要显式 CAST 的情况。

    参考：[#1879](https://www.sqlalchemy.org/trac/ticket/1879)

+   **[oracle]**

    Oracle 反射索引已经调整，以便反射包含一些或全部主键列的索引，但不包含与主键相同的列集的索引。在反射中跳过包含与主键相同列的索引，因为在这种情况下，该索引被假定为自动生成的主键索引。以前，任何包含 PK 列的索引都会被跳过。感谢 Kent Bower 提供的补丁。

    参考：[#1867](https://www.sqlalchemy.org/trac/ticket/1867)

+   **[oracle]**

    Oracle 现在反映主键约束的名称 - 还要感谢 Kent Bower。

    参考：[#1868](https://www.sqlalchemy.org/trac/ticket/1868)

### 杂项

+   **[declarative]**

    如果@classproperty 与常规类绑定的 mapper 属性属性一起使用，它将在初始化期间被调用以获取实际属性值。目前，在不是 mixin 的声明类的列或关系属性上使用@classproperty 没有任何优势 - 评估与未使用@classproperty 时同时进行。但至少在这里，我们允许其按预期运行。

+   **[声明式]**

    修复了“无法添加额外列”的错误，显示的名称不正确的问题。

+   **[Firebird]**

    修复了一个 bug，即如果“default”关键字为小写，则列默认值将无法反映。

+   **[Informix]**

    应用了来自的补丁，以重新启用基本的 Informix 功能。我们依赖最终用户的测试来确保 Informix 在某种程度上正常工作。

    参考：[#1904](https://www.sqlalchemy.org/trac/ticket/1904)

+   **[文档]**

    文档已重新组织，删除了“API 参考”部分 - 所有公共 API 的 docstrings 都移动到了讨论它的主要文档部分的上下文中。主要文档分为“SQLAlchemy 核心”和“SQLAlchemy ORM”部分，mapper/relationship 文档已拆分出来。许多部分已被重写和/或重新组织。

### ORM

+   **[ORM]**

    ConcurrentModificationError 的名称已更改为 StaleDataError，并且描述性错误消息已经修订以准确反映问题所在。在可预见的未来，这两个名称将保持可用，以供在“except:”子句中指定 ConcurrentModificationError 的方案使用。

+   **[ORM]**

    向 identity map 添加了互斥锁，该锁用于互斥删除操作，这些操作针对迭代方法，在返回可迭代对象之前现在会预先缓冲。这是因为异步 gc 可以随时通过 gc 线程删除项目。

    参考：[#1891](https://www.sqlalchemy.org/trac/ticket/1891)

+   **[ORM]**

    Session 类现在存在于 sqlalchemy.orm.*中。我们正在摆脱使用 create_session()，对于需要一步构造 Session 的情况，它具有非标准的默认值。大多数用户应该继续使用 sessionmaker()进行一般用途，然而。

+   **[ORM]**

    query.with_parent()现在接受瞬态对象，并将使用其 pk/fk 属性的非持久化值来制定条件。文档还澄清了 with_parent()的目的。

+   **[ORM]**

    mapper()的 include_properties 和 exclude_properties 参数现在除了字符串外还接受列对象作为成员。这样，可以消除在 join()中存在的同名列对象的歧义。

+   **[ORM]**

    如果对一个连接或其他单个可选择的映射器创建了一个包含多个具有相同名称的列的警告，而这些列没有明确命名为相同或不同的属性（或排除），则会发出警告。在 0.7 版本中，此警告将变为异常。请注意，当组合发生在继承的结果时，不会发出此警告，因此属性仍然允许自然覆盖。在 0.7 版本中，这将进一步改进。

    参考：[#1896](https://www.sqlalchemy.org/trac/ticket/1896)

+   **[orm]**

    mapper()的 primary_key 参数现在可以指定一系列列，这些列仅是映射可选择的“主键”列的子集，而不会引发错误。这对于可选择的有效主键比实际标记为“primary_key”的列数更简单的情况很有帮助，例如在两个表的主键列上进行连接。

    参考：[#1896](https://www.sqlalchemy.org/trac/ticket/1896)

+   **[orm]**

    已删除的对象现在会得到一个名为‘deleted’的标志，这会阻止该对象重新添加到会话中，因为以前对象会悄无声息地存在于标识映射中，直到访问其属性。make_transient()函数现在会重置此标志以及“key”标志。

+   **[orm]**

    make_transient()现在可以安全地在已经是瞬态实例上调用。

+   **[orm]**

    如果在映射器()中发出警告，如果 polymorphic_on 列在映射的可选择中或在 with_polymorphic 可选择中不存在，而不是悄悄地忽略它。在 0.7 版本中，预计这将变为异常。

+   **[orm]**

    当 relationship()配置具有模糊参数时，再次通过一系列错误消息。不再提及“foreign_keys”设置，因为几乎从不需要，并且更希望用户设置正确的 ForeignKey 元数据，这现在是推荐的做法。如果使用了‘foreign_keys’并且不正确，消息会建议该属性可能是不必要的。属性的文档得到加强。这是因为所有在 ML 上困惑的 relationship()用户似乎都试图使用 foreign_keys，因为消息只会进一步使他们困惑，而 Table 元数据则更清晰。

+   **[orm]**

    如果“secondary”表没有 ForeignKey 元数据并且没有设置 foreign_keys，即使用户传递了错误的信息，也假定主/次级联表达式应仅考虑“secondary”中的所有列为外键。在任何情况下，“secondary”中的外键都不可能在其他地方。现在发出警告而不是错误，并且映射成功。

    参考：[#1877](https://www.sqlalchemy.org/trac/ticket/1877)

+   **[orm]**

    将一个 o2m 对象从一个集合移动到另一个集合，或者通过 m2o 更改引用对象，其中外键也是主键的成员，现在在 flush 期间将更加仔细地进行检查，如果“many”一侧的外键值的更改是由于“one”一侧主键的更改导致的，或者如果“one”只是一个不同的对象。在一个情况下，可级联的 DB 已经级联了该值，我们需要查看“new”PK 值来执行 UPDATE，在另一个情况下，我们需要继续查看“old”。我们现在查看“old”，假设 passive_updates=True，除非我们知道它是触发更改的 PK 切换。

    参考：[#1856](https://www.sqlalchemy.org/trac/ticket/1856)

+   **[orm]**

    version_id_col 的值可以手动更改，这将导致行的 UPDATE。现在，版本化的 UPDATE 和 DELETE 使用 WHERE 子句中的 version_id_col 的“committed”值，而不是挂起的更改值。如果属性上存在手动更改，则版本生成器也将被绕过。

    参考：[#1857](https://www.sqlalchemy.org/trac/ticket/1857)

+   **[orm]**

    修复了在与具体继承的映射器一起使用 merge() 时的使用情况。这样的映射器经常具有所谓的“具体”属性，这些属性是“禁用”从父类传播的子类属性 - 这些属性需要允许 merge() 操作通过而不产生效果。

+   **[orm]**

    指定非基于列的参数作为 column_mapped_collection，包括字符串、text() 等，将会触发错误消息，明确要求使用列元素，不再误导使用 text() 或 literal() 关于不正确的信息。

    参考：[#1863](https://www.sqlalchemy.org/trac/ticket/1863)

+   **[orm]**

    类似地，对于 relationship()、foreign_keys、remote_side、order_by - 所有基于列的表达式都被强制执行 - 字符串列表明确不允许，因为这是一个非常常见的错误。

+   **[orm]**

    动态属性不支持集合填充 - 当调用 set_committed_value() 时，以及将 joinedload() 或 subqueryload() 选项应用于动态属性时，添加了一个断言，而不是失败/静默失败。

    参考：[#1864](https://www.sqlalchemy.org/trac/ticket/1864)

+   **[orm]**

    修复了一个错误，即从具有相同列但具有不同标签名称的列重复的 Query 生成的 Query 在某些 UNION 情况下会无法完全传播内部列到外部查询。

    参考：[#1852](https://www.sqlalchemy.org/trac/ticket/1852)

+   **[orm]**

    当出现未映射实例时，object_session() 会引发正确的 UnmappedInstanceError。

    参考：[#1881](https://www.sqlalchemy.org/trac/ticket/1881)

+   **[orm]**

    对计算的 Mapper 属性应用了更进一步的记忆化，在重度多态映射配置中减少了显著（约 90%）的 runtime mapper.py 调用次数。

+   **[orm]**

    版本示例中使用的 mapper _get_col_to_prop 私有方法已被弃用；现在请使用 mapper.get_property_by_column()，这将保持为此公共方法。

+   **[ORM]**

    版本示例现在在以前为 NULL 的列上进行版本控制时可以正常工作。

### 示例

+   **[示例]**

    beaker_caching 示例已重新组织，使得 Session、缓存管理器、declarative_base 成为环境的一部分，自定义缓存代码现在在“caching_query.py”中，这使得示例更容易“插入”到现有项目中。

+   **[示例]**

    history_meta 版本控制方案在复制列时设置“unique=False”，以便版本控制表处理具有重复值的多行。

    参考：[#1887](https://www.sqlalchemy.org/trac/ticket/1887)

### 引擎

+   **[引擎]**

    在已经耗尽、已关闭或不是返回结果的结果上调用 fetchone()或类似方法现在会在所有情况下引发 ResourceClosedError，这是 InvalidRequestError 的子类，不受后端影响。以前，一些 DBAPI 会引发 ProgrammingError（例如 pysqlite），其他会返回 None 导致下游故障（例如 MySQL-python）。

+   **[引擎]**

    修复了 Connection 中的错误，如果在第一个连接池连接的“初始化”阶段发生“断开”事件，那么当 Connection 尝试使 DBAPI 连接无效时会引发 AttributeError。

    参考：[#1894](https://www.sqlalchemy.org/trac/ticket/1894)

+   **[引擎]**

    Connection、ResultProxy 以及 Session 现在对所有“此连接/事务/结果已关闭”类型的错误使用 ResourceClosedError。

+   **[引擎]**

    Connection.invalidate()可以被多次调用，后续调用不会产生任何效果。

### SQL

+   **[SQL]**

    在 alias()构造上调用 execute()在 0.7 版本中将被��用，因为它本身不是一个“可执行”构造。它当前“代理”其内部元素，并且有条件地“可执行”，但这不是我们现在喜欢的模糊性。

+   **[SQL]**

    ClauseElement 的 execute()和 scalar()方法现在适当地移动到了 Executable 子类中。ClauseElement.execute()/scalar()仍然存在，并且在 0.7 版本中将被弃用，但请注意，如果您不是 Executable（除非您是 alias()，请参见前面的说明），这些方法无论如何都会引发错误。

+   **[SQL]**

    为 Numeric->Integer 添加了基本的数学表达式强制转换，以便无论表达式的方向如何，结果类型始终为 Numeric。

+   **[SQL]**

    更改了在 Column 上使用“index=True”标志生成截断的“auto”索引名称的方案。截断仅在自动生成的名称上进行，而不是在用户定义的名称上（否则会引发错误），截断方案本身现在基于标识符名称的 md5 哈希的片段，以便具有类似名称的列上的多个索引仍具有唯一名称。

    参考：[#1855](https://www.sqlalchemy.org/trac/ticket/1855)

+   **[sql]**

    生成的索引名称也基于“最大索引名称长度”属性，该属性与“最大标识符长度”分开 - 这是为了取悦 MySQL，因为 MySQL 对索引名称的最大长度为 64，而总体最大长度为 255。

    参考：[#1412](https://www.sqlalchemy.org/trac/ticket/1412)

+   **[sql]**

    如果将 text()构造放置在面向列的情况下，它至少会返回 NULLTYPE 作为其类型，而不是 None，这使得它可以比以前更自由地用于临时列表达式。但是，literal_column()仍然是更好的选择。

+   **[sql]**

    在 ForeignKey 无法解析目标时引发错误消息时，添加了父表/列，目标表/列的完整描述。

+   **[sql]**

    修复了一个错误，即在反射表中替换复合外键列会导致尝试第二次从表中删除反射的约束，从而引发 KeyError。

    参考：[#1865](https://www.sqlalchemy.org/trac/ticket/1865)

+   **[sql]**

    _Label 构造，即每当您说 somecol.label()时产生的构造，现在将其自身计入其“proxy_set”中，与其包含的列的 proxy set 相结合，而不是直接返回包含的列的 proxy set。这允许依赖 _Label 本身身份的列对应操作返回正确的结果。

+   **[sql]**

    修复了 ORM 错误。

    参考：[#1852](https://www.sqlalchemy.org/trac/ticket/1852)

### postgresql

+   **[postgresql]**

    修复了 psycopg2 方言，使用其 set_isolation_level()方法，而不是依赖基本的“SET SESSION ISOLATION”命令，否则 psycopg2 会在每个新事务中重置隔离级别。

### mssql

+   **[mssql]**

    修复了与 pymssql 后端一起使用“默认模式”查询的问题。

### oracle

+   **[oracle]**

    在 Oracle 方言中添加了 ROWID 类型，用于那些可能需要显式 CAST 的情况。

    参考：[#1879](https://www.sqlalchemy.org/trac/ticket/1879)

+   **[oracle]**

    Oracle 反射索引已经调整，以便反射包含一些或全部主键列，但不包含与主键相同的列集的索引。在反射中跳过包含与主键相同列的索引，因为在这种情况下，该索引被假定为自动生成的主键索引。以前，任何包含 PK 列的索引都会被跳过。感谢 Kent Bower 的补丁。

    参考：[#1867](https://www.sqlalchemy.org/trac/ticket/1867)

+   **[oracle]**

    现在 Oracle 反映了主键约束的名称 - 这也要感谢 Kent Bower。

    参考：[#1868](https://www.sqlalchemy.org/trac/ticket/1868)

### 杂项

+   **[declarative]**

    如果 @classproperty 与常规的类绑定映射器属性属性一起使用，则在初始化期间将调用它以获取实际的属性值。目前，在不是 mixin 的声明类的列或关系属性上使用 @classproperty 没有任何优势 - 评估的时间与未使用 @classproperty 时相同。但是在这里，我们至少允许它按预期工作。

+   **[declarative]**

    修复了“无法添加额外列”的错误消息显示错误名称的 bug。

+   **[firebird]**

    修复了一个 bug，即如果“default”关键字为小写，则列默认值将失败反映。

+   **[informix]**

    应用了来自的补丁，以重新启用基本的 Informix 功能。我们依赖最终用户的测试来确保 Informix 在某种程度上正常工作。

    参考：[#1904](https://www.sqlalchemy.org/trac/ticket/1904)

+   **[documentation]**

    文档已重新组织，以至于“API 参考”部分已经消失 - 所有从那里的文档字符串中公开的 API 都移动到了主文档部分的上下文中，该部分讨论了它。主文档分为“SQLAlchemy Core”和“SQLAlchemy ORM”两个部分，映射器/关系文档已被拆分。许多部分已被重写和/或重新组织。

## 0.6.3

发布日期：Thu Jul 15 2010

### orm

+   **[orm]**

    在单元测试中移除了不必要的多对多加载，该加载在已过期/未加载的集合上不必要地触发。现在，仅当 passive_updates 为 False 且父级主键已更改，或者 passive_deletes 为 False 且已删除父级时，才执行此加载。

    参考：[#1845](https://www.sqlalchemy.org/trac/ticket/1845)

+   **[orm]**

    列实体（即 query(Foo.id)）在查询从自身派生的查询时更全面地复制其状态+可选择的（即 from_self()、union() 等），以便 join() 等从正确的状态开始工作。

    参考：[#1853](https://www.sqlalchemy.org/trac/ticket/1853)

+   **[orm]**

    修复了 Query.join() 在查询非 ORM 列然后在已经存在 FROM 子句的情况下没有使用 on 子句进行连接时会失败的 bug，现在会像没有子句时一样引发一个经过检查的异常。

    参考：[#1853](https://www.sqlalchemy.org/trac/ticket/1853)

+   **[orm]**

    改进了对“未映射类”的检查，包括超类已映射但子类未映射的情况。任何尝试访问 cls._sa_class_manager.mapper 现在都会引发 UnmappedClassError()。

    参考：[#1142](https://www.sqlalchemy.org/trac/ticket/1142)

+   **[orm]**

    添加了“column_descriptions”访问器到 Query，返回一个包含查询将返回的实体的命名/类型信息的字典列表。对于在 ORM 查询之上构建 GUI 很有帮助。

### mysql

+   **[mysql]**

    _extract_error_code() 方法现在可以正确地与每个 MySQL 方言（MySQL-python、OurSQL、MySQL-Connector-Python、PyODBC）一起工作。以前，重新连接逻辑会在 OperationalError 条件下失败，但由于 MySQLdb 和 OurSQL 有自己的重新连接功能，因此除非观察日志，否则这些驱动程序在这里没有任何症状。

    参考：[#1848](https://www.sqlalchemy.org/trac/ticket/1848)

### oracle

+   **[oracle]**

    对 cx_oracle Decimal 处理进行了更多微调。没有小数点的“模糊”数值在连接处理程序级别被强制转换为 int。这里的优势在于 ints 以 int 形式返回，而无需涉及 SQLA 类型对象，也无需先转换为 Decimal。

    不幸的是，一些奇特的子查询情况甚至可能在单个结果行之间看到不同的类型，因此当 Numeric 处理程序被指示返回 Decimal 时，无法充分利用“本机十进制”模式，必须对每个值运行 isinstance() 来检查其��否已经是 Decimal。重新打开

    参考：[#1840](https://www.sqlalchemy.org/trac/ticket/1840)

### orm

+   **[orm]**

    在 unitofwork 中删除了错误的多对多加载，这会在过期/未加载的集合上不必要地触发。现在，只有在 passive_updates 为 False 且父主键已更改，或者 passive_deletes 为 False 且父项已删除时，才会进行此加载。

    参考：[#1845](https://www.sqlalchemy.org/trac/ticket/1845)

+   **[orm]**

    Column-entities（即 query(Foo.id)）在从自身派生的查询（即 from_self()、union() 等）中更全面地复制其状态，以便 join() 等操作可以正确地进行。

    参考：[#1853](https://www.sqlalchemy.org/trac/ticket/1853)

+   **[orm]**

    修复了 Query.join() 在查询非 ORM 列然后在已经存在 FROM 子句的情况下没有 on 子句进行连接时会失败的 bug，现在会像没有子句时一样引发一个经过检查的异常。

    参考：[#1853](https://www.sqlalchemy.org/trac/ticket/1853)

+   **[orm]**

    改进了对“未映射类”的检查，包括超类已映射但子类未映射的情况。任何尝试访问 cls._sa_class_manager.mapper 现在都会引发 UnmappedClassError()。

    参考：[#1142](https://www.sqlalchemy.org/trac/ticket/1142)

+   **[orm]**

    向 Query 添加了 “column_descriptions” 访问器，返回一个包含查询将返回的实体的命名/类型信息的字典列表。对于在 ORM 查询之上构建 GUI 非常有帮助。

### mysql

+   **[mysql]**

    _extract_error_code() 方法现在可以正确地与每个 MySQL 方言（MySQL-python、OurSQL、MySQL-Connector-Python、PyODBC）一起工作。以前，重新连接逻辑会在 OperationalError 条件下失败，但由于 MySQLdb 和 OurSQL 有自己的重新连接功能，因此除非观察日志，否则这些驱动程序在这里没有任何症状。

    参考：[#1848](https://www.sqlalchemy.org/trac/ticket/1848)

### oracle

+   **[oracle]**

    对 cx_oracle Decimal 处理进行了更多调整。没有小数点的“模糊”数值在连接处理程序级别被强制转换为 int。这里的优势在于 ints 会作为 ints 返回，而不涉及 SQLA 类型对象，也不会不必要地先转换为 Decimal。

    不幸的是，一些奇特的子查询情况甚至可能在单个结果行之间看到不同的类型，因此当 Numeric 处理程序被指示返回 Decimal 时，无法充分利用“本地十进制”模式，必须对每个值运行 isinstance() 来检查它是否已经是 Decimal。重新打开

    参考：[#1840](https://www.sqlalchemy.org/trac/ticket/1840)

## 0.6.2

发布日期：Tue Jul 06 2010

### orm

+   **[orm]**

    Query.join() 将检查是否调用了 query.join(target, clause_expression) 形式的调用，即缺少元组，并提出信息性错误消息，说明这是错误的调用形式。

+   **[orm]**

    修复了关于自引用双向多对多关系刷新的 bug，在一个刷新中使两个对象相互引用会导致两侧都无法插入行。从 0.5 开始的回归。

    参考：[#1824](https://www.sqlalchemy.org/trac/ticket/1824)

+   **[orm]**

    relationship() 的 post_update 功能在架构上进行了重新设计，以更紧密地与新的 0.6 工作单元集成。更改的动机是，多个影响同一行的不同外键列的“post update”调用将在单个 UPDATE 语句中执行，而不是每列每行一个 UPDATE 语句。多行更新也尽可能批量执行，同时保持一致的行顺序。

+   **[orm]**

    Query.statement, Query.subquery(), 等现在会将绑定参数的值，即由 query.params() 指定的值，传递到生成的 SQL 表达式中。以前这些值不会被传递，绑定参数会变成 None。

+   **[orm]**

    现在子查询预加载可以与包括 params() 的 Query 对象一起工作，以及 get() 查询。

+   **[orm]**

    现在可以在被父对象通过多对一引用的实例上调用 make_transient()，而不会暂时将父对象的外键值设置为 None - 这是“检测主键切换”刷新处理程序的功能。它现在会忽略不再处于“持久”状态的对象，父对象的外键标识符也不受影响。

+   **[orm]**

    query.order_by() 现在接受 False，这会取消 Query 上任何现有的 order_by() 状态，允许调用后续不支持 ORDER BY 的生成方法。这与已经存在的传递 None 的功能不同，后者会抑制任何现有的 order_by() 设置，包括在映射器上配置的设置。False 会使 order_by() 就好像从未调用过一样，而 None 是一个活跃的设置。

+   **[orm]**

    当一个被移至“瞬态”状态的实例具有不完整或缺失的主键属性集，并且包含过期属性时，如果访问了一个过期属性，将会引发 InvalidRequestError，而不是递归溢出。

+   **[orm]**

    make_transient() 函数现在在生成的文档中。

+   **[orm]**

    make_transient() 会将所有“loader”可调用函数从被设置为瞬态的状态中移除，将任何“过期”状态重置为未定义的、None/空值。

### sql

+   **[sql]**

    当 convert_unicode=True 时，Unicode 和 String 类型发出的警告不再嵌入传递的实际值。这样做是为了避免 Python 警告注册表继续增长，警告根据警告过滤器设置只发出一次，并且大字符串值不会污染输出。

    参考：[#1822](https://www.sqlalchemy.org/trac/ticket/1822)

+   **[sql]**

    修复了一个 bug，该 bug 会阻止对“带注��”的表达式元素进行重写子句编译，这些表达式元素通常由 ORM 生成。

+   **[sql]**

    LIKE 操作符或类似操作符的“ESCAPE”参数会通过 render_literal_value() 进行处理，该函数可能会实现反斜杠的转义。

    参考：[#1400](https://www.sqlalchemy.org/trac/ticket/1400)

+   **[sql]**

    修复了 Enum 类型的一个 bug，当与 TypeDecorators 或其他适配场景一起使用时，会破坏 native_enum 标志。

+   **[sql]**

    当 Inspector 被调用时，会调用 bind.connect() 来确保 initialize 已被调用。内部名称“.conn”被更改为“.bind”，因为那才是它的实际名称。

+   **[sql]**

    修改了“列注释”的内部机制，使得自定义 Column 子类可以安全地重写 _constructor 返回 Column，用于创建不涉及代理等的“配置”列类。

+   **[sql]**

    Column.copy() 包括“unique”属性在内，修复了关于声明性 mixin 的问题

    参考：[#1829](https://www.sqlalchemy.org/trac/ticket/1829)

### postgresql

+   **[postgresql]**

    render_literal_value() 被重写以转义反斜杠，目前适用于 LIKE 和类似表达式的 ESCAPE 子句。最终，这将需要检测“standard_conforming_strings”的值以实现完整的行为。

    参考：[#1400](https://www.sqlalchemy.org/trac/ticket/1400)

+   **[postgresql]**

    如果在 PG 版本低于 8.3 上使用 types.Enum，则不会生成“CREATE TYPE” / “DROP TYPE” - supports_native_enum 标志将被完全遵守。

    参考：[#1836](https://www.sqlalchemy.org/trac/ticket/1836)

### mysql

+   **[mysql]**

    MySQL 方言在检测到 MySQL 版本小于 4.0.2 时不会发出 CAST()。这允许在连接时进行 unicode 检查。

    参考：[#1826](https://www.sqlalchemy.org/trac/ticket/1826)

+   **[mysql]**

    MySQL 方言现在检测到 NO_BACKSLASH_ESCAPES sql 模式，除了 ANSI_QUOTES。

+   **[mysql]**

    覆盖了 render_literal_value()，它会转义反斜杠，目前适用于 LIKE 和类似表达式的 ESCAPE 子句。��行为源自检测 NO_BACKSLASH_ESCAPES 的值。

    参考：[#1400](https://www.sqlalchemy.org/trac/ticket/1400)

### mssql

+   **[mssql]**

    如果 server_version_info 超出通常范围（8, ）、（9, ）、（10, ），则会发出警告，建议检查 FreeTDS 版本配置是否使用 7.0 或 8.0，而不是 4.2。

    参考：[#1825](https://www.sqlalchemy.org/trac/ticket/1825)

### oracle

+   **[oracle]**

    修复了 ora-8 兼容性标志，使其不会缓存在第一次数据库连接之前的过时值。

    参考：[#1819](https://www.sqlalchemy.org/trac/ticket/1819)

+   **[oracle]**

    当列嵌入子查询时，Oracle 的“本地十进制”元数据开始返回关于数值的模糊类型信息，以及当在子查询中查询 ROWNUM 时，我们用于 limit/offset 时。我们已将这些模糊条件添加到 cx_oracle 的“转换为 Decimal()”处理程序中，以便在更多情况下将数值作为 Decimal 而不是浮点数接收。然后，如果需要，将其转换为 Integer 或 Float，否则保留为无损 Decimal。

    参考：[#1840](https://www.sqlalchemy.org/trac/ticket/1840)

### 杂项

+   **[firebird]**

    修复了 do_execute() 中的错误签名，该错误在 0.6.1 中引入。

    参考：[#1823](https://www.sqlalchemy.org/trac/ticket/1823)

+   **[firebird]**

    Firebird 方言添加了接受“charset”标志的 CHAR、VARCHAR 类型，以支持 Firebird 的“CHARACTER SET”子句。

    参考：[#1813](https://www.sqlalchemy.org/trac/ticket/1813)

+   **[声明式]**

    添加了对 @classproperty 的支持，以从声明性混合中提供任何类型的模式/映射构造，包括具有外键、关系、column_property、deferred 的列。如果在混合中指定任何 MapperProperty 子类而不使用 @classproperty，则会引发错误。

    参考：[#1751](https://www.sqlalchemy.org/trac/ticket/1751), [#1796](https://www.sqlalchemy.org/trac/ticket/1796), [#1805](https://www.sqlalchemy.org/trac/ticket/1805)

+   **[声明式]**

    现在，一个混合类可以定义一个与子类上定义的 __table__ 上存在的列匹配的列。但是，它不能定义一个在 __table__ 中不存在的列，现在的错误消息可以正常工作。

    参考：[#1821](https://www.sqlalchemy.org/trac/ticket/1821)

+   **[编译器] [扩展]**

    当覆盖内置子句构造的编译时，“default”编译器会自动复制过去，因此如果调用不同后端的编译，不会引发 KeyError。

    参考：[#1838](https://www.sqlalchemy.org/trac/ticket/1838)

+   **[文档]**

    为 Inspector 添加了文档。

    参考：[#1820](https://www.sqlalchemy.org/trac/ticket/1820)

+   **[文档]**

    修复了@memoized_property 和@memoized_instancemethod 装饰器，以便 Sphinx 文档捕获这些属性和方法，例如 ResultProxy.inserted_primary_key。

    参考：[#1830](https://www.sqlalchemy.org/trac/ticket/1830)

### orm

+   **[orm]**

    Query.join()将检查是否调用了 query.join(target, clause_expression)形式的调用，即缺少元组，并引发一个信息性错误消息，说明这是错误的调用形式。

+   **[orm]**

    修复了关于自引用双向多对多关系刷新的错误，其中在一个刷新中使两个对象相互引用将无法为双方插入行。从 0.5 版本开始的回归。

    参考：[#1824](https://www.sqlalchemy.org/trac/ticket/1824)

+   **[orm]**

    relationship()的 post_update 功能在架构上进行了重新设计，以更紧密地与新的 0.6 工作单元集成。更改的动机是，多个影响同一行的不同外键列的“post update”调用将在单个 UPDATE 语句中执行，而不是每列每行一个 UPDATE 语句。多行更新也尽可能批量处理为 executemany()，同时保持一致的行顺序。

+   **[orm]**

    Query.statement，Query.subquery()等现在将绑定参数的值，即由 query.params()指定的值，传输到生成的 SQL 表达式中。以前，这些值不会被传输，绑定参数将变为 None。

+   **[orm]**

    子查询预加载现在可以与包括 params()的 Query 对象一起工作，以及 get()查询。

+   **[orm]**

    现在可以在通过多对一引用的父对象引用的实例上调用 make_transient()，而不会将父对象的外键值临时设置为 None - 这是“检测主键切换”刷新处理程序的功能。现在它会忽略不再处于“持久”状态的对象，并且父对象的外键标识符不受影响。

+   **[orm]**

    query.order_by()现在接受 False，取消 Query 上的任何现有 order_by()状态，允许调用不支持 ORDER BY 的后续生成方法。这与已存在的传递 None 的功能不同，后者会抑制任何现有的 order_by()设置，包括在映射器上配置的设置。False 将使 order_by()好像从未被调用过，而 None 是一个活动设置。

+   **[orm]**

    将移至“瞬态”状态的实例，具有不完整或缺失的主键属性集，并包含已过期属性，如果访问了已过期属性，将引发 InvalidRequestError，而不是出现递归溢出。

+   **[orm]**

    make_transient()函数现在在生成的文档中。

+   **[orm]**

    make_transient()从被转换为瞬态状态的状态中删除所有“加载器”可调用项，删除任何“过期”状态 - 所有未加载的属性在访问时重置为未定义，None/空。

### sql

+   **[sql]**

    当 convert_unicode=True 时，Unicode 和 String 类型发出的警告不再嵌入传递的实际值。这样做是为了防止 Python 警告注册表继续增长，根据警告过滤器设置，警告只发出一次，并且大字符串值不会污染输出。

    引用：[#1822](https://www.sqlalchemy.org/trac/ticket/1822)

+   **[sql]**

    修复了一个错误，该错误会阻止“annotated”表达式元素的覆盖子句编译正常工作，这些元素通常由 ORM 生成。

+   **[sql]**

    LIKE 运算符或类似的“ESCAPE”的参数通过 render_literal_value()传递，该函数可能会实现反斜杠的转义。

    引用：[#1400](https://www.sqlalchemy.org/trac/ticket/1400)

+   **[sql]**

    Enum 类型中的错误修复了当与 TypeDecorators 或其他适配场景一起使用时会取消 native_enum 标志的情况。

+   **[sql]**

    当调用 Inspector 时，确保 initialize 已被调用。内部名称“.conn”更改为“.bind”，因为这就是它的名称。

+   **[sql]**

    修改了“列注释”的内部，使得自定义的 Column 子类可以安全地覆盖 _constructor 返回 Column，用于创建“配置型”列类，这些类不涉及代理等。

+   **[sql]**

    Column.copy()在其他属性中带有“unique”属性，修复有关声明性混合的问题

    引用：[#1829](https://www.sqlalchemy.org/trac/ticket/1829)

### postgresql

+   **[postgresql]**

    render_literal_value()被覆盖，其转义反斜杠，当前适用于 LIKE 及类似表达式的 ESCAPE 子句。最终这将需要检测“standard_conforming_strings”的值以实现完整的行为。

    引用：[#1400](https://www.sqlalchemy.org/trac/ticket/1400)

+   **[postgresql]**

    如果在 8.3 之前的 PG 版本上使用 types.Enum，则不会生成“CREATE TYPE”/“DROP TYPE” - supports_native_enum 标志完全受到尊重。

    引用：[#1836](https://www.sqlalchemy.org/trac/ticket/1836)

### mysql

+   **[mysql]**

    MySQL 方言不会为检测到的 MySQL 版本< 4.0.2 发出 CAST()。这允许在连接时进行 unicode 检查。

    引用：[#1826](https://www.sqlalchemy.org/trac/ticket/1826)

+   **[mysql]**

    MySQL 方言现在检测到 NO_BACKSLASH_ESCAPES sql 模式，除了 ANSI_QUOTES 之外。

+   **[mysql]**

    render_literal_value()被覆盖，其转义反斜杠，当前适用于 LIKE 及类似表达式的 ESCAPE 子句。此行为源自检测 NO_BACKSLASH_ESCAPES 的值。

    引用：[#1400](https://www.sqlalchemy.org/trac/ticket/1400)

### mssql

+   **[mssql]**

    如果 server_version_info 在常规范围之外（8，），（9，），（10，），则会发出警告，建议检查 FreeTDS 版本配置是否使用 7.0 或 8.0，而不是 4.2。

    引用：[#1825](https://www.sqlalchemy.org/trac/ticket/1825)

### oracle

+   **[oracle]**

    修复了 ora-8 兼容性标志，使其不会缓存第一个数据库连接实际发生之前的过时值。

    参考：[#1819](https://www.sqlalchemy.org/trac/ticket/1819)

+   **[oracle]**

    当列嵌入子查询时，Oracle 的“本机小数”元数据开始返回关于数值的模糊类型信息，以及当子查询中咨询 ROWNUM 时，我们对于 limit/offset 所做的情况。我们已将这些模糊条件添加到 cx_oracle 的“转换为 Decimal()”处理程序中，以便我们在更多情况下将数值作为 Decimal 而不是作为浮点数接收。然后，如果请求，将其转换为 Integer 或 Float，否则保留为无损 Decimal。

    参考：[#1840](https://www.sqlalchemy.org/trac/ticket/1840)

### 杂项

+   **[firebird]**

    修复了 do_execute() 中的错误签名，这是在 0.6.1 中引入的错误。

    参考：[#1823](https://www.sqlalchemy.org/trac/ticket/1823)

+   **[firebird]**

    Firebird 方言添加了 CHAR、VARCHAR 类型，它们接受一个“charset”标志，以支持 Firebird 的“CHARACTER SET”子句。

    参考：[#1813](https://www.sqlalchemy.org/trac/ticket/1813)

+   **[声明式]**

    添加了对 @classproperty 的支持，以从声明式混合中提供任何类型的模式/映射构造，包括具有外键、关系、column_property、deferred 的列。如果在混合中指定任何 MapperProperty 子类而不使用 @classproperty，则会引发错误。

    参考：[#1751](https://www.sqlalchemy.org/trac/ticket/1751), [#1796](https://www.sqlalchemy.org/trac/ticket/1796), [#1805](https://www.sqlalchemy.org/trac/ticket/1805)

+   **[声明式]**

    现在一个混合类可以定义一个与子类上定义的 __table__ 上存在的列相匹配的列。但是，它不能定义一个在 __table__ 中不存在的列，这里的错误消息现在可以工作。

    参考：[#1821](https://www.sqlalchemy.org/trac/ticket/1821)

+   **[编译器] [扩展]**

    当覆盖内置子句结构的编译时，默认编译器会自动复制过来，因此如果用户定义的编译器特定于某些后端，并且为不同后端调用了编译，就不会引发 KeyError。

    参考：[#1838](https://www.sqlalchemy.org/trac/ticket/1838)

+   **[文档]**

    添加了 Inspector 的文档。

    参考：[#1820](https://www.sqlalchemy.org/trac/ticket/1820)

+   **[文档]**

    修复了 @memoized_property 和 @memoized_instancemethod 装饰器，使 Sphinx 文档能够捕获这些属性和方法，例如 ResultProxy.inserted_primary_key。

    参考：[#1830](https://www.sqlalchemy.org/trac/ticket/1830)

## 0.6.1

发布日期：2010 年 5 月 31 日（星期一）

### orm

+   **[orm]**

    修复了 0.6.0 中引入的关于可变属性不正确历史记录的回归。

    参考：[#1782](https://www.sqlalchemy.org/trac/ticket/1782)

+   **[orm]**

    修复了在 0.6.0 工作单元重构中引入的回归问题，导致带有 post_update=True 的双向 relationship() 更新失败。

    参考：[#1807](https://www.sqlalchemy.org/trac/ticket/1807)

+   **[orm]**

    session.merge() 不会使返回的实例上的属性过期，如果该实例处于 “pending” 状态。

    参考：[#1789](https://www.sqlalchemy.org/trac/ticket/1789)

+   **[orm]**

    修复了 CollectionAdapter 的 __setstate__ 方法，在父 InstanceState 尚未反序列化时不会失败。

    参考：[#1802](https://www.sqlalchemy.org/trac/ticket/1802)

+   **[orm]**

    在实例没有完整主键的情况下，如果发生过期并且被要求刷新，则会添加内部警告。

    参考：[#1797](https://www.sqlalchemy.org/trac/ticket/1797)

+   **[orm]**

    对映射器在使用 UPDATE、INSERT 和 DELETE 表达式时进行了更积极的缓存。假设语句没有附加到每个对象的 SQL 表达式，那么在第一次创建后，映射器会将表达式对象缓存，并将其编译形式持久存储在与相关 Engine 相关联的缓存字典中。对于极少数情况下，如果映射器接收到极高数量的不同列模式作为 UPDATE，缓存将是一个 LRUCache。

### sql

+   **[sql]**

    expr.in_() 现在接受 text() 结构作为参数。自动添加了分组括号，即使用方式类似于 col.in_(text(“select id from table”))。

    参考：[#1793](https://www.sqlalchemy.org/trac/ticket/1793)

+   **[sql]**

    _Binary 类型的列（即 LargeBinary、BLOB 等）也会将右侧的 “basestring” 强制转换为 _Binary，以便进行必要的 DBAPI 处理。

+   **[sql]**

    添加了 table.add_is_dependent_on(othertable)，允许手动在两个 Table 对象之间放置依赖规则，供 create_all()、drop_all()、sorted_tables 使用。

    参考：[#1801](https://www.sqlalchemy.org/trac/ticket/1801)

+   **[sql]**

    修复了阻止具有包含零的复合主键的隐式 RETURNING 正常运行的错误。

    参考：[#1778](https://www.sqlalchemy.org/trac/ticket/1778)

+   **[sql]**

    修复了生成具有命名 UNIQUE 约束的 ADD CONSTRAINT 时错误的空格字符。

+   **[sql]**

    修复了 ForeignKeyConstraint 构造函数中 “table” 参数的问题。

    参考：[#1571](https://www.sqlalchemy.org/trac/ticket/1571)

+   **[sql]**

    修复了连接池游标包装器中的错误，即如果游标在 close() 时抛出异常，则消息的记录将失败。

    参考：[#1786](https://www.sqlalchemy.org/trac/ticket/1786)

+   **[sql]**

    ColumnClause 和 Column 的 _make_proxy() 方法现在使用 self.__class__ 来确定要返回的对象类，而不是硬编码到 ColumnClause/Column，这使得更容易生成在别名/子查询情况下工作的特定子类。

+   **[sql]**

    func.XXX() 不会意外解析为非函数类（例如修复了 func.text()）。

    参考：[#1798](https://www.sqlalchemy.org/trac/ticket/1798)

### mysql

+   **[mysql]**

    func.sysdate() 在 MySQL 上发出“SYSDATE()”，即带有结尾括号。

    参考：[#1794](https://www.sqlalchemy.org/trac/ticket/1794)

### sqlite

+   **[sqlite]**

    修复了在 SQLite AUTOINCREMENT 关键字被渲染时，将“PRIMARY KEY”约束移动到列级别时约束的连接。

    参考：[#1812](https://www.sqlalchemy.org/trac/ticket/1812)

### oracle

+   **[oracle]**

    添加了对低于 5 版本的 cx_oracle 的检查，在这种情况下，不兼容的“输出类型处理程序”将不会被使用。这将影响小数精度和一些 Unicode 处理问题。

    参考：[#1775](https://www.sqlalchemy.org/trac/ticket/1775)

+   **[oracle]**

    修复了 use_ansi=False 模式，该模式在几乎所有情况下都会产生损坏的 WHERE 子句。

    参考：[#1790](https://www.sqlalchemy.org/trac/ticket/1790)

+   **[oracle]**

    重新建立了对 Oracle 8 的 cx_oracle 支持，包括将 use_ansi 自动设置为 False，不为 Unicode 渲染 NVARCHAR2 和 NCLOB，不会导致“本地 Unicode”检查失败，禁用 cx_oracle 的“本地 Unicode”模式，以字节计数形式发出 VARCHAR()。

    参考：[#1808](https://www.sqlalchemy.org/trac/ticket/1808)

+   **[oracle]**

    在正常的 Python 2.x 模式下，oracle_xe 5 不接受 Python unicode 对象作为其连接字符串 - 因此我们直接将其强制转换为 str()。由于我们不知道可以使用的编码，因此这里的连接字符串不支持非 ASCII 字符。

    参考：[#1670](https://www.sqlalchemy.org/trac/ticket/1670)

+   **[oracle]**

    当使用 limit/offset 时，在语法上正确的位置发出 FOR UPDATE，即 ROWNUM 子查询。但是，Oracle 实际上无法处理带有 ORDER BY 或子查询的 FOR UPDATE，因此它仍然无法使用，但至少 SQLA 可以将 SQL 传递给 Oracle 解析器。

    参考：[#1815](https://www.sqlalchemy.org/trac/ticket/1815)

### misc

+   **[engines]**

    修复了在 Python 2.4 上构建 C 扩展的问题。

    参考：[#1781](https://www.sqlalchemy.org/trac/ticket/1781)

+   **[engines]**

    在 dispose()发生后，池类将重用相同的“pool_logging_name”设置。

+   **[engines]**

    Engine 增加了一个“execution_options”参数和 update_execution_options()方法，将应用于此引擎生成的所有连接。

+   **[firebird]**

    在 has_table() 和 has_sequence() 中使用的查询中添加了一个标签，以适应不提供结果列标签的旧版本 Firebird。

    参考：[#1521](https://www.sqlalchemy.org/trac/ticket/1521)

+   **[firebird]**

    当通过查询字符串传递时，将整数强制转换为“type_conv”属性，以便 Kinterbasdb 正确解释它。

    参考：[#1779](https://www.sqlalchemy.org/trac/ticket/1779)

+   **[firebird]**

    将“连接关闭”添加到指示连接中断的异常字符串列表中。

    参考：[#1646](https://www.sqlalchemy.org/trac/ticket/1646)

+   **[sqlsoup]**

    SqlSoup 构造函数接受一个 base 参数，该参数指定用于映射类的基类，默认为 object。

    参考：[#1783](https://www.sqlalchemy.org/trac/ticket/1783)

### orm

+   **[orm]**

    修复了在 0.6.0 版本中引入的涉及可变属性不正确历史记录的回归问题。

    参考：[#1782](https://www.sqlalchemy.org/trac/ticket/1782)

+   **[orm]**

    修复了在 0.6.0 版本中引入的回归问题，该问题破坏了具有 post_update=True 的双向 relationship()的更新。

    参考：[#1807](https://www.sqlalchemy.org/trac/ticket/1807)

+   **[orm]**

    ��果返回的实例是“pending”，session.merge()将不会使实例上的属性过期。

    参考：[#1789](https://www.sqlalchemy.org/trac/ticket/1789)

+   **[orm]**

    修复了 CollectionAdapter 的 __setstate__ 方法，在反序列化时不会因为父 InstanceState 尚未反序列化而失败。

    参考：[#1802](https://www.sqlalchemy.org/trac/ticket/1802)

+   **[orm]**

    在实例没有完整主键的情况下，如果实例被过期并要求刷新，则会添加内部警告。

    参考：[#1797](https://www.sqlalchemy.org/trac/ticket/1797)

+   **[orm]**

    对 mapper 在使用 UPDATE、INSERT 和 DELETE 表达式时进行了更积极的缓存。假设语句没有附加到每个对象的 SQL 表达式，那么在第一次创建后，mapper 会将表达式对象缓存，并且它们的编译形式将持久地存储在与相关 Engine 相关的缓存字典中。缓存是一个 LRUCache，用于极少数情况下 mapper 接收到极高数量的不同列模式作为 UPDATE。

### sql

+   **[sql]**

    expr.in_()现在接受一个 text()构造作为参数。自动添加分组括号，即使用方式类似于 col.in_(text(“select id from table”)).

    参考：[#1793](https://www.sqlalchemy.org/trac/ticket/1793)

+   **[sql]**

    _Binary 类型的列（即 LargeBinary、BLOB 等）将右侧的“basestring”强制转换为 _Binary，以便进行必要的 DBAPI 处理。

+   **[sql]**

    添加了 table.add_is_dependent_on(othertable)，允许在 create_all()、drop_all()、sorted_tables 中手动设置两个 Table 对象之间的依赖规则。

    参考：[#1801](https://www.sqlalchemy.org/trac/ticket/1801)

+   **[sql]**

    修复了阻止具有包含零的复合主键的隐式 RETURNING 正常运行的错误。

    参考：[#1778](https://www.sqlalchemy.org/trac/ticket/1778)

+   **[sql]**

    修复了为命名 UNIQUE 约束生成 ADD CONSTRAINT 时生成的额外空格字符。

+   **[sql]**

    修复了 ForeignKeyConstraint 构造函数上的“table”参数。

    参考：[#1571](https://www.sqlalchemy.org/trac/ticket/1571)

+   **[sql]**

    修复了连接池游标包装器中的错误，即如果游标在 close()时抛出异常，则消息的记录将失败。

    参考：[#1786](https://www.sqlalchemy.org/trac/ticket/1786)

+   **[sql]**

    ColumnClause 和 Column 的 _make_proxy() 方法现在使用 self.__class__ 来确定要返回的对象类，而不是硬编码为 ColumnClause/Column，这使得在别名/子查询情况下更容易生成特定的子类。

+   **[sql]**

    func.XXX() 不会错误地解析为非 Function 类（例如修复了 func.text()）。

    参考：[#1798](https://www.sqlalchemy.org/trac/ticket/1798)

### mysql

+   **[mysql]**

    func.sysdate() 在 MySQL 上发出“SYSDATE()”，即带有结束括号。

    参考：[#1794](https://www.sqlalchemy.org/trac/ticket/1794)

### sqlite

+   **[sqlite]**

    修复了当“PRIMARY KEY”约束由于 SQLite AUTOINCREMENT 关键字被渲染而移动到列级别时的约束连接。

    参考：[#1812](https://www.sqlalchemy.org/trac/ticket/1812)

### oracle

+   **[oracle]**

    添加了一个检查，用于低于版本 5 的 cx_oracle 版本，如果是这种情况，则不会使用不兼容的“output type handler”。这将影响十进制精度和一些 Unicode 处理问题。

    参考：[#1775](https://www.sqlalchemy.org/trac/ticket/1775)

+   **[oracle]**

    修复了 use_ansi=False 模式，在几乎所有情况下都会产生错误的 WHERE 子句。

    参考：[#1790](https://www.sqlalchemy.org/trac/ticket/1790)

+   **[oracle]**

    重新建立了对 Oracle 8 的 cx_oracle 支持，包括自动将 use_ansi 设置为 False，对 Unicode 不渲染 NVARCHAR2 和 NCLOB，不会因为“native unicode”检查失败，禁用 cx_oracle 的“native unicode”模式，用字节计数而不是字符计数发出 VARCHAR()。

    参考：[#1808](https://www.sqlalchemy.org/trac/ticket/1808)

+   **[oracle]**

    在正常的 Python 2.x 模式下，oracle_xe 5 不接受 Python unicode 对象作为其连接字符串 - 因此我们直接强制转换为 str()。这里不支持连接字符串中的非 ASCII 字符，因为我们不知道可以使用什么编码。

    参考：[#1670](https://www.sqlalchemy.org/trac/ticket/1670)

+   **[oracle]**

    当使用 limit/offset 时，FOR UPDATE 被放置在语法正确的位置，即 ROWNUM 子查询。然而，Oracle 实际上无法处理带有 ORDER BY 或子查询的 FOR UPDATE，因此仍然不太可用，但至少 SQLA 能够将 SQL 传递给 Oracle 解析器。

    参考：[#1815](https://www.sqlalchemy.org/trac/ticket/1815)

### misc

+   **[engines]**

    修复了在 Python 2.4 上构建 C 扩展的问题。

    参考：[#1781](https://www.sqlalchemy.org/trac/ticket/1781)

+   **[engines]**

    在 dispose() 发生后，Pool 类将重用相同的“pool_logging_name”设置。

+   **[engines]**

    Engine 增加了“execution_options”参数和 update_execution_options()方法，将应用于此 engine 生成的所有连接。

+   **[firebird]**

    在 has_table() 和 has_sequence() 中使用的查询添加了一个标签，以便与不提供结果列标签的旧版本 Firebird 一起使用。

    参考：[#1521](https://www.sqlalchemy.org/trac/ticket/1521)

+   **[firebird]**

    在通过查询字符串传递“type_conv”属性时，添加了整数强制转换，以便 Kinterbasdb 正确解释它。

    参考：[#1779](https://www.sqlalchemy.org/trac/ticket/1779)

+   **[firebird]**

    将“连接关闭”添加到指示连接中断的异常字符串列表中。

    参考：[#1646](https://www.sqlalchemy.org/trac/ticket/1646)

+   **[sqlsoup]**

    SqlSoup 构造函数接受一个 base 参数，该参数指定用于映射类的基类，默认为 object。

    参考：[#1783](https://www.sqlalchemy.org/trac/ticket/1783)

## 0.6.0

发布日期：2010 年 4 月 18 日星期日

### orm

+   **[orm]**

    工作单元内部已经重写。具有大量相互依赖对象的工作单元现在可以在没有递归溢出的情况下刷新，因为不再依赖递归调用。对于特定会话状态，内部结构的数量现在保持恒定，无论映射上存在多少关系。事件流现在对应于由映射器和基于实际工作的关系生成的线性步骤列表，通过单个拓扑排序进行正确排序。刷新操作使用更少的步骤和更少的内存进行组装。

    参考：[#1081](https://www.sqlalchemy.org/trac/ticket/1081)，[#1742](https://www.sqlalchemy.org/trac/ticket/1742)

+   **[orm]**

    除了 UOW 重写之外，这还解决了 0.6beta3 中关于具有长依赖循环的工作单元的拓扑循环检测的问题。我们现在使用 Guido 编写的算法（感谢 Guido！）。

+   **[orm]**

    一对多关系现在在刷新时维护一个正的父子关联列表，防止之前标记为已删除的父项对这些子对象进行级联删除或设置为 NULL 外键，尽管最终用户没有从旧关联中移除子项。

    参考：[#1764](https://www.sqlalchemy.org/trac/ticket/1764)

+   **[orm]**

    集合的延迟加载将关闭反向的多对一端的默认急加载，因为根据定义，该加载是不必要的。

    参考：[#1495](https://www.sqlalchemy.org/trac/ticket/1495)

+   **[orm]**

    Session.refresh()现在首先对给定实例执行等效的 expire()，以便“refresh-expire”级联被传播。以前，refresh()不受“refresh-expire”级联的影响。这是与 0.6beta2 的行为不同之处，其中传递给 refresh()的“lockmode”标志会导致版本检查发生。由于实例首先被过期，refresh()总是将对象升级到最新版本。

+   **[orm]**

    “refresh-expire”级联在到达挂起对象时，如果级联还包括“delete-orphan”，则会将对象清除；否则，将简单地分离它。

    参考：[#1754](https://www.sqlalchemy.org/trac/ticket/1754)

+   **[orm]**

    在 topological.py 中不再内部使用 id(obj)，因为排序函数现在仅需要可哈希对象。

    参考：[#1756](https://www.sqlalchemy.org/trac/ticket/1756)

+   **[orm]**

    ORM 现在默认将所有生成的描述符的文档字符串设置为 None。可以使用‘doc’进行覆盖（或者如果使用 Sphinx，属性文档字符串也可以）。

+   **[orm]**

    对所有映射器属性可调用的 Column()添加了 kw 参数‘doc’。将字符串‘doc’组装为描述符上的‘__doc__’属性。

+   **[orm]**

    在支持 cursor.rowcount 执行()但不支持 executemany()的后端上，现在在发出删除时可以使用 version_id_col（已经适用于保存，因为这些不使用 executemany()）。对于根本不支持 cursor.rowcount 的后端，会发出与保存相同的警告。

    参考：[#1761](https://www.sqlalchemy.org/trac/ticket/1761)

+   **[orm]**

    当刷新所有同一类对象列表时，ORM 现在会短期缓存 insert()和 update()构造的“编译”形式，从而避免在单个 flush()调用中每个 INSERT/UPDATE 都进行冗余编译。

+   **[orm]**

    ColumnProperty、CompositeProperty、RelationshipProperty 上的内部 getattr()、setattr()、getcommitted()方法已被下划线标记（即为私有），签名已更改。

### 示例

+   **[examples]**

    更新了 attribute_shard.py 示例，使用更健壮的方法搜索 Query，比较列与文字值的二进制表达式。

### sql

+   **[sql]**

    从 0.5 版本中恢复了一些绑定标签逻辑，确保具有与另一列重叠的列名的表，如“<tablename>_<columnname>”，在 UPDATE 期间使用 column._label 作为绑定名称时不会产生错误。添加了 0.5 版本中不存在的测试覆盖率。

    参考：[#1755](https://www.sqlalchemy.org/trac/ticket/1755)

+   **[sql]**

    somejoin.select(fold_equivalents=True)不再被弃用，并最终将被整合到更全面的功能版本中。

    参考：[#1729](https://www.sqlalchemy.org/trac/ticket/1729)

+   **[sql]**

    当 Numeric 类型预期将浮点数转换为来自返回浮点数的 DBAPI 的 Decimal 时，会引发*巨大*警告。这包括 SQLite、Sybase、MS-SQL。

    参考：[#1759](https://www.sqlalchemy.org/trac/ticket/1759)

+   **[sql]**

    修复了表达式类型化中的错误，导致具有两个 NULL 类型的表达式出现无限循环。

+   **[sql]**

    修复了 execution_options()功能中的错误，即父连接中现有的事务和其他状态信息不会传播到子连接。

+   **[sql]**

    添加了新的‘compiled_cache’执行选项。一个字典，在 Connection 将一个子句表达式编译成特定于方言和参数的 Compiled 对象时，Compiled 对象将被缓存。用户有责任管理这个字典的大小，它将具有对应于方言、子句元素、INSERT 或 UPDATE 语句的 VALUES 或 SET 子句中的列名称，以及 INSERT 或 UPDATE 语句的“批量”模式的键。

+   **[sql]**

    添加了 get_pk_constraint()到 reflection.Inspector，类似于 get_primary_keys()，但返回一个包括约束名称的字典，用于支持的后端（目前为 PG）。

    参考：[#1769](https://www.sqlalchemy.org/trac/ticket/1769)

+   **[sql]**

    Table.create()和 Table.drop()不再应用于元数据级别的创建/删除事件。

    参考：[#1771](https://www.sqlalchemy.org/trac/ticket/1771)

### postgresql

+   **[postgresql]**

    现在，当序列的名称更改后，PostgreSQL 会正确地反映与 SERIAL 列关联的序列名称。感谢 Kumar McMillan 提供的补丁。

    参考：[#1071](https://www.sqlalchemy.org/trac/ticket/1071)

+   **[postgresql]**

    修复了在接收到未知数字时，psycopg2._PGNumeric 类型中缺少的导入。

+   **[postgresql]**

    psycopg2/pg8000 方言现在知道 REAL[]、FLOAT[]、DOUBLE_PRECISION[]、NUMERIC[]返回类型，而不会引发异常。

+   **[postgresql]**

    如果存在，则 PostgreSQL 反映主键约束的名称。

    参考：[#1769](https://www.sqlalchemy.org/trac/ticket/1769)

### oracle

+   **[oracle]**

    现在使用 cx_oracle 输出转换器，以便 DBAPI 原生返回我们喜欢的值类型：

+   **[oracle]**

    具有正精度+标度的 NUMBER 值将转换为 cx_oracle.STRING，然后转换为 Decimal。这样在使用 cx_oracle 时，Numeric 类型可以完美精确。

    参考：[#1759](https://www.sqlalchemy.org/trac/ticket/1759)

+   **[oracle]**

    STRING/FIXED_CHAR 现在本地转换为 unicode。因此，SQLAlchemy 的 String 类型不需要应用任何类型的转换。

### 杂项

+   **[engines]**

    现在 C 扩展还适用于使用自定义序列作为行（而不仅仅是元组）的 DBAPI。

    参考：[#1757](https://www.sqlalchemy.org/trac/ticket/1757)

+   **[ext]**

    编译器扩展现在允许在扩展到子类的基类上使用@compiles 装饰器，在子类上使用@compiles 装饰器不会被基类上的@compiles 装饰器破坏。

+   **[ext]**

    如果在基于字符串的 relationship()参数中引用了非映射类属性，Declarative 将引发信息性错误消息。

+   **[ext]**

    进一步重新设计了 Declarative 中的“mixin”逻辑，以允许 __mapper_args__ 作为 mixin 上的@classproperty，例如动态分配 polymorphic_identity。

+   **[firebird]**

    在每个引擎上可以通过在 create_engine() 上设置 'enable_rowcount=False' 来禁用 result.rowcount 的功能。通常，在任何 UPDATE 或 DELETE 语句之后无条件地调用 cursor.rowcount，因为然后游标被关闭，而 Firebird 需要一个打开的游标才能获得行数。然而，这个调用稍微昂贵，所以可以禁用它。要在每个执行基础上重新启用，可以使用 'enable_rowcount=True' 执行选项。

### orm

+   **[orm]**

    工作单元内部已被重写。具有大量相互依赖对象的工作单元现在可以在没有递归溢出的情况下刷新，因为不再依赖于递归调用。对于特定的会话状态，内部结构的数量现在保持不变，无论映射上存在多少关系。事件的流现在对应于由映射器和基于实际工作的关系生成的线性步骤列表，通过单个拓扑排序进行正确排序。刷新操作使用更少的步骤和更少的内存进行组装。

    参考：[#1081](https://www.sqlalchemy.org/trac/ticket/1081), [#1742](https://www.sqlalchemy.org/trac/ticket/1742)

+   **[orm]**

    伴随 UOW 重写，这还解决了在 0.6beta3 中引入的有关拓扑循环检测的问题，针对具有长依赖循环的工作单元。我们现在使用由 Guido 编写的算法（感谢 Guido！）。

+   **[orm]**

    一对多关系现在在刷新期间维护一个正父子关联的列表，防止以前标记为已删除的父项从级联删除或将空外键集设置在这些子对象上，尽管最终用户没有从旧关联中删除子项。

    参考：[#1764](https://www.sqlalchemy.org/trac/ticket/1764)

+   **[orm]**

    集合的延迟加载将关闭反向的多对一端上的默认急加载，因为该加载从定义上来说是不必要的。

    参考：[#1495](https://www.sqlalchemy.org/trac/ticket/1495)

+   **[orm]**

    现在，Session.refresh() 首先对给定的实例执行等效的 expire()，以便“refresh-expire”级联被传播。之前，refresh() 在任何方式上都不受“refresh-expire”级联的影响。这与 0.6beta2 的行为不同，在那里传递给 refresh() 的“lockmode”标志会导致版本检查发生。由于实例首先过期，因此 refresh() 总是将对象升级到最新版本。

+   **[orm]**

    当“refresh-expire”级联到达挂起的对象时，如果级联还包括“delete-orphan”，则将清除该对象；否则，将简单地分离它。

    参考：[#1754](https://www.sqlalchemy.org/trac/ticket/1754)

+   **[orm]**

    id(obj) 不再在 topological.py 内部使用，因为排序函数现在仅需要可哈希对象。

    参考：[#1756](https://www.sqlalchemy.org/trac/ticket/1756)

+   **[orm]**

    ORM 现在默认将所有生成的描述符的文档字符串设置为 None。可以使用‘doc’进行覆盖（或者如果使用 Sphinx，则属性文档字符串也有效）。

+   **[ORM]**

    对所有 mapper 属性可调用对象以及 Column()添加了 kw 参数‘doc’。将字符串‘doc’组装为描述符的‘__doc__’属性。

+   **[ORM]**

    在支持 cursor.rowcount 用于 execute()但不支持 executemany()的后端上，现在在发出删除时使用 version_id_col 可以正常工作（对于保存已经可以正常工作，因为它们不使用 executemany()）。对于根本不支持 cursor.rowcount 的后端，与保存一样会发出警告。

    参考：[#1761](https://www.sqlalchemy.org/trac/ticket/1761)

+   **[ORM]**

    当刷新相同类别对象列表时，ORM 现在会短期缓存 insert()和 update()构造的“编译”形式，从而避免在单个 flush()调用中每个 INSERT/UPDATE 都进行冗余编译。

+   **[ORM]**

    ColumnProperty、CompositeProperty、RelationshipProperty 上的内部 getattr()、setattr()、getcommitted()方法已被下划线标记（即为私有），签名已更改。

### 示例

+   **[示例]**

    更新了 attribute_shard.py 示例，以使用更健壮的方法搜索 Query，该方法将列与文字值进行比较。

### sql

+   **[sql]**

    从 0.5 版本中恢复了一些绑定标签逻辑，确保具有与另一列重叠的列名的表在 UPDATE 期间使用 column._label 作为绑定名称时不会产生错误。已添加 0.5 中不存在的测试覆盖率。 

    参考：[#1755](https://www.sqlalchemy.org/trac/ticket/1755)

+   **[sql]**

    somejoin.select(fold_equivalents=True)不再被弃用，并最终将被合并到更全面的功能版本中。

    参考：[#1729](https://www.sqlalchemy.org/trac/ticket/1729)

+   **[sql]**

    当 Numeric 类型预期将浮点数转换为 Decimal 时，如果来自返回浮点数的 DBAPI（包括 SQLite、Sybase、MS-SQL）会引发*巨大*警告。

    参考：[#1759](https://www.sqlalchemy.org/trac/ticket/1759)

+   **[sql]**

    修复了表达式类型化中的错误，该错误导致具有两个 NULL 类型的表达式陷入无限循环。

+   **[sql]**

    修复了 execution_options()功能中的错误，该错误导致父连接中的现有事务和其他状态信息无法传播到子连接。

+   **[sql]**

    添加了新的‘compiled_cache’执行选项。一个字典，当 Connection 将一个子句表达式编译成特定于方言和参数的 Compiled 对象时，Compiled 对象将被缓存。用户有责任管理这个字典的大小，它将具有与方言、子句元素、INSERT 或 UPDATE 的 VALUES 或 SET 子句中的列名以及 INSERT 或 UPDATE 语句的“batch”模式相对应的键。

+   **[sql]**

    在 reflection.Inspector 中添加了 get_pk_constraint()，类似于 get_primary_keys()，但返回一个包含约束名称的字典，适用于支持的后端（目前仅支持 PG）。

    参考：[#1769](https://www.sqlalchemy.org/trac/ticket/1769)

+   **[sql]**

    Table.create() 和 Table.drop() 不再应用于元数据级别的创建/删除事件。

    参考：[#1771](https://www.sqlalchemy.org/trac/ticket/1771)

### postgresql

+   **[postgresql]**

    PostgreSQL 现在正确反映了与 SERIAL 列关联的序列名称，序列名称已更改后。感谢 Kumar McMillan 提交的补丁。

    参考：[#1071](https://www.sqlalchemy.org/trac/ticket/1071)

+   **[postgresql]**

    修复了在接收到未知数字时 psycopg2._PGNumeric 类型中缺少的导入。

+   **[postgresql]**

    psycopg2/pg8000 方言现在能够识别 REAL[]、FLOAT[]、DOUBLE_PRECISION[]、NUMERIC[] 返回类型，而不会引发异常。

+   **[postgresql]**

    如果存在主键约束，PostgreSQL 将反映主键约束的名称。

    参考：[#1769](https://www.sqlalchemy.org/trac/ticket/1769)

### oracle

+   **[oracle]**

    现在使用 cx_oracle 输出转换器，以便 DBAPI 本地返回我们更喜欢的值类型：

+   **[oracle]**

    具有正精度 + 标度的 NUMBER 值转换为 cx_oracle.STRING，然后转换为 Decimal。这允许在使用 cx_oracle 时 Numeric 类型具有完美的精度。

    参考：[#1759](https://www.sqlalchemy.org/trac/ticket/1759)

+   **[oracle]**

    STRING/FIXED_CHAR 现在本地转换为 unicode。因此，SQLAlchemy 的 String 类型不需要应用任何类型的转换。

### 杂项

+   **[引擎]**

    C 扩展现在还可以与使用自定义序列作为行的 DBAPI 一起使用（不仅仅是元组）。

    参考：[#1757](https://www.sqlalchemy.org/trac/ticket/1757)

+   **[ext]**

    编译器扩展现在允许在扩展到子类的基类上使用 @compiles 装饰器，在子类上使用 @compiles 装饰器不会被基类上的 @compiles 装饰器破坏。

+   **[扩展]**

    如果在基于字符串的 relationship() 参数中引用了非映射类属性，声明式将引发一个信息性错误消息。

+   **[ext]**

    进一步重新调整了声明式中的“mixin”逻辑，还允许在 mixin 上作为 @classproperty 动态分配 polymorphic_identity，例如 __mapper_args__。

+   **[firebird]**

    可以通过在 create_engine() 上设置 'enable_rowcount=False' 来在每个引擎上禁用 result.rowcount 的功能。通常，在任何 UPDATE 或 DELETE 语句之后无条件地调用 cursor.rowcount，因为然后游标将被关闭，而 Firebird 需要一个打开的游标才能获取行数。但是，这个调用稍微昂贵，因此可以禁用。要在每次执行时重新启用，可以使用 'enable_rowcount=True' 执行选项。

## 0.6beta3

发布日期：2010 年 3 月 28 日 星期日

### orm

+   **[orm]**

    主要特性：为 relationship()添加了新的“subquery”加载功能。这是一种急切加载选项，为查询中表示的每个集合生成第二个 SELECT，跨所有父级一次性加载。该查询重新发出原始的最终用户查询，包装在一个子查询中，将连接应用到目标集合，并一次性完全加载所有这些集合的结果，类似于“joined”急切加载，但使用所有内连接，不会重复重新获取完整的父行（即使跳过列，大多数 DBAPI 似乎也会这样做）。子查询加载在映射器配置级别使用“lazy='subquery'”和在查询选项级别使用“subqueryload(props..)”，“subqueryload_all(props…)”可用。

    参考：[#1675](https://www.sqlalchemy.org/trac/ticket/1675)

+   **[orm]**

    为了适应现在有两种可用的急��加载的事实，eagerload()和 eagerload_all()的新名称是 joinedload()和 joinedload_all()。旧名称将作为可预见的将来的同义词保留。

+   **[orm]**

    relationship()函数上的“lazy”标志现在接受字符串参数，用于所有种类的加载：“select”、“joined”、“subquery”、“noload”和“dynamic”，其中默认值现在为“select”。True/False/None 的旧值仍保留其通常含义，并将作为可预见的将来的同义词。

+   **[orm]**

    向 Query()构造添加了 with_hint()方法。这直接调用 select().with_hint()，并且还接受实体以及表和别名。请参见下面 SQL 部分中的 with_hint()。

    参考：[#921](https://www.sqlalchemy.org/trac/ticket/921)

+   **[orm]**

    修复了 Query 中的一个 bug，即调用 q.join(prop).from_self(…). join(prop)会在内部连接的相同条件上连接第二个连接时，未能在子查询之外呈现第二个连接。

+   **[orm]**

    修复了 Query 中的一个 bug，即在 q.from_self()或 q.select_from()生成的子查询中引用 aliased()构造会失败，如果引用了底层表（但实际别名未被引用）。

+   **[orm]**

    修复了影响所有 eagerload()和类似选项的 bug，即“remote”急切加载，即从延迟加载中进行急切加载，例如 query(A).options(eagerload(A.b, B.c))不会急切加载任何内容，但使用 eagerload(“b.c”)将正常工作。

+   **[orm]**

    Query 获得了一个 add_columns(*columns)方法，这是 add_column(col)的多版本。add_column(col)将来将被弃用。

+   **[orm]**

    Query.join()将检测最终结果是否为“FROM A JOIN A”，如果是，则会引发错误。

+   **[orm]**

    Query.join(Cls.propname, from_joinpoint=True)将更仔细地检查“Cls”是否与当前连接点兼容，并在这方面与 Query.join(“propname”, from_joinpoint=True)表现相同。

### sql

+   **[sql]**

    向 select()构造添加了 with_hint()方法。指定表/别名、提示文本和可选的方言名称，“hints”将在语句中适当的位置呈现。适用于 Oracle、Sybase、MySQL。

    参考：[#921](https://www.sqlalchemy.org/trac/ticket/921)

+   **[sql]**

    修复了在 0.6beta2 中引入的 bug，其中列标签会在已分配标签的列表达式内部呈现。

    参考：[#1747](https://www.sqlalchemy.org/trac/ticket/1747)

### postgresql

+   **[postgresql]**

    psycopg2 方言将通过“sqlalchemy.dialects.postgresql”记录 NOTICE 消息。

    参考：[#877](https://www.sqlalchemy.org/trac/ticket/877)

+   **[postgresql]**

    时间和时间戳类型现在可以直接从 postgresql 方言中使用，这为两者都添加了 PG 特定参数‘precision’。‘precision’和‘timezone’对于时间和时区类型都正确反映。

    参考：[#997](https://www.sqlalchemy.org/trac/ticket/997)

### mysql

+   **[mysql]**

    不再猜测 TINYINT(1)应该是 BOOLEAN 当反射时 - 返回 TINYINT(1)。在表定义中使用 Boolean/BOOLEAN 以获得布尔转换行为。

    参考：[#1752](https://www.sqlalchemy.org/trac/ticket/1752)

### oracle

+   **[oracle]**

    Oracle 方言将使用字符计数发出 VARCHAR 类型定义，即 VARCHAR2(50 CHAR)，因此列的大小是以字符而不是字节为单位。字符类型的列反射还将使用 ALL_TAB_COLUMNS.CHAR_LENGTH 而不是 ALL_TAB_COLUMNS.DATA_LENGTH。这些行为在服务器版本为 9 或更高版本时生效 - 对于版本 8，将使用旧行为。

    参考：[#1744](https://www.sqlalchemy.org/trac/ticket/1744)

### 杂项

+   **[declarative]**

    如果 mixin 实现了不可预测的 __getattribute__()，即 Zope 接口，使用 mixin 不会出错。

    参考：[#1746](https://www.sqlalchemy.org/trac/ticket/1746)

+   **[declarative]**

    现在在 mixin 上使用@classdecorator 等来定义 __tablename__、__table_args__ 等，如果方法引用了最终子类上的属性，现在可以正常工作。

    参考：[#1749](https://www.sqlalchemy.org/trac/ticket/1749)

+   **[declarative]**

    在声明性 mixin 上不允许有带有外键的关系和列，抱歉。

    参考：[#1751](https://www.sqlalchemy.org/trac/ticket/1751)

+   **[ext]**

    sqlalchemy.orm.shard 模块现在成为扩展，sqlalchemy.ext.horizontal_shard。旧的导入会有一个弃用警告。

### orm

+   **[orm]**

    主要功能：为 relationship()添加了新的“subquery”加载功能。这是一种急切加载选项，为查询中表示的每个集合生成第二个 SELECT，跨所有父级一次性加载。查询重新发出原始的最终用户查询，包装在一个子查询中，应用连接到目标集合，一次性完全加载所有这些集合的结果，类似于“joined”急切加载，但使用所有内连接，不会重复重新获取完整的父行（即使跳过列，大多数 DBAPI 似乎也会这样做）。子查询加载在映射器配置级别使用“lazy=‘subquery’”和在查询选项级别使用“subqueryload(props..)”，“subqueryload_all(props…)”可用。

    参考：[#1675](https://www.sqlalchemy.org/trac/ticket/1675)

+   **[orm]**

    为了适应现在有两种可用的 eager loading，eagerload() 和 eagerload_all() 的新名称分别为 joinedload() 和 joinedload_all()。旧名称将在可预见的未来保留为同义词。

+   **[orm]**

    relationship() 函数上的 “lazy” 标志现在接受字符串参数用于所有加载类型：“select”、“joined”、“subquery”、“noload” 和 “dynamic”，其中默认值现在为 “select”。True/False/None 的旧值仍保留其通常含义，并将在可预见的未来保留为同义词。

+   **[orm]**

    向 Query() 构造添加了 with_hint() 方法。这直接调用 select().with_hint()，并且还接受实体以及表和别名。请参见下面 SQL 部分中的 with_hint()。

    参考：[#921](https://www.sqlalchemy.org/trac/ticket/921)

+   **[orm]**

    修复了 Query 中的 bug，调用 q.join(prop).from_self(…).join(prop) 将无法在与内部相同的条件上连接第二个 join 到子查询之外时失败。

+   **[orm]**

    修复了 Query 中的 bug，如果在 q.from_self() 或 q.select_from() 生成的子查询中引用了基础表（但实际别名未被引用），则使用 aliased() 构造将失败。

+   **[orm]**

    修复了影响所有 eagerload() 和类似选项的 bug，即“远程” eager loads，即从延迟加载（如 query(A).options(eagerload(A.b, B.c)）中的 eagerloads 不会加载任何内容，但使用 eagerload(“b.c”) 将正常工作。

+   **[orm]**

    Query 增加了 add_columns(*columns) 方法，这是 add_column(col) 的多版本。add_column(col) 将被未来弃用。

+   **[orm]**

    Query.join() 将检测结果是否为 “FROM A JOIN A”，如果是，则会引发错误。

+   **[orm]**

    Query.join(Cls.propname, from_joinpoint=True) 将更仔细地检查 “Cls” 是否与当前连接点兼容，并在这方面与 Query.join(“propname”, from_joinpoint=True) 表现相同。

### sql

+   **[sql]**

    添加了 with_hint() 方法到 select() 构造中。指定表/别名、提示文本和可选的方言名称，“提示”将在语句中适当的位置呈现。适用于 Oracle、Sybase、MySQL。

    参考：[#921](https://www.sqlalchemy.org/trac/ticket/921)

+   **[sql]**

    修复了在 0.6beta2 中引入的 bug，其中列标签会呈现在已分配标签的列表达式内部。

    参考：[#1747](https://www.sqlalchemy.org/trac/ticket/1747)

### postgresql

+   **[postgresql]**

    psycopg2 方言将通过 “sqlalchemy.dialects.postgresql” 记录 NOTICE 消息。

    参考：[#877](https://www.sqlalchemy.org/trac/ticket/877)

+   **[postgresql]**

    TIME 和 TIMESTAMP 类型现在可以直接从 postgresql 方言中使用，这为两者添加了 PG 特定的参数 ‘precision’。‘precision’ 和 ‘timezone’ 对于 TIME 和 TIMEZONE 类型都正确反映。

    参考：[#997](https://www.sqlalchemy.org/trac/ticket/997)

### mysql

+   **[mysql]**

    不再猜测 TINYINT(1) 应该在反射时转换为 BOOLEAN - 返回的是 TINYINT(1)。在表定义中使用 Boolean/ BOOLEAN 可以获得布尔转换行为。

    参考：[#1752](https://www.sqlalchemy.org/trac/ticket/1752)

### oracle

+   **[oracle]**

    Oracle 方言将使用字符计数发出 VARCHAR 类型定义，即 VARCHAR2(50 CHAR)，因此列的大小是以字符而不是字节为单位。字符类型的列反射也将使用 ALL_TAB_COLUMNS.CHAR_LENGTH 而不是 ALL_TAB_COLUMNS.DATA_LENGTH。这些行为在服务器版本为 9 或更高时生效 - 对于版本 8，将使用旧的行为。

    参考：[#1744](https://www.sqlalchemy.org/trac/ticket/1744)

### 杂项

+   **[declarative]**

    如果 mixin 实现了不可预测的 __getattribute__()，即 Zope 接口，使用 mixin 不会出错。

    参考：[#1746](https://www.sqlalchemy.org/trac/ticket/1746)

+   **[declarative]**

    在 mixins 上使用 @classdecorator 和类似方法来定义 __tablename__、__table_args__ 等现在可以正常工作，如果方法引用了最终子类的属性。

    参考：[#1749](https://www.sqlalchemy.org/trac/ticket/1749)

+   **[declarative]**

    不允许在声明性 mixins 上使用带有外键的关系和列，抱歉。

    参考：[#1751](https://www.sqlalchemy.org/trac/ticket/1751)

+   **[ext]**

    sqlalchemy.orm.shard 模块现在成为扩展，sqlalchemy.ext.horizontal_shard。旧的导入会有弃用警告。

## 0.6beta2

发布日期：2010 年 3 月 20 日

### orm

+   **[orm]**

    relation() 函数的官方名称现在是 relationship()，以消除对关系代数术语的混淆。然而，relation() 在可预见的未来仍将以相同的能力可用。

    参考：[#1740](https://www.sqlalchemy.org/trac/ticket/1740)

+   **[orm]**

    向 Mapper 添加了 “version_id_generator” 参数，这是一个可调用对象，给定 “version_id_col” 的当前值，返回下一个版本号。可用于替代版本控制方案，如 uuid、timestamps。

    参考：[#1692](https://www.sqlalchemy.org/trac/ticket/1692)

+   **[orm]**

    在 Session.refresh() 中添加了 “lockmode” kw 参数，将字符串值传递给 Query，与 with_lockmode() 中的行为相同，还将为启用 version_id_col 的映射执行版本检查。

+   **[orm]**

    修复了在联合继承场景中调用 query(A).join(A.bs).add_entity(B) 会将 B 重复添加为目标并生成无效查询的 bug。

    参考：[#1188](https://www.sqlalchemy.org/trac/ticket/1188)

+   **[orm]**

    修复了 session.rollback() 中的 bug，涉及在重新整合“删除”对象之前未从会话中移除以前的“挂起”对象，通常发生在自然主键上。如果它们之间存在主键冲突，删除的附加将在内部失败。现在首先清除以前的“挂起”对象。

    参考：[#1674](https://www.sqlalchemy.org/trac/ticket/1674)

+   **[orm]**

    删除了很多没有人真正关心的日志记录，保留的日志记录将响应日志级别的实时更改。不会增加显著的开销。

    参考：[#1719](https://www.sqlalchemy.org/trac/ticket/1719)

+   **[orm]**

    修复了 session.merge()中的 bug，该 bug 阻止了类似字典的集合的合并。

+   **[orm]**

    session.merge()可以与明确不包括“merge”在其级联选项中的关系一起工作-目标完全被忽略。

+   **[orm]**

    如果目标具有该属性的值，则 session.merge()不会使现有目标上的现有标量属性过期，即使传入的合并没有该属性的值。这可以防止对现有项目进行不必要的加载。但是，如果目标没有该属性，它仍将标记为过期，这符合延迟列的某些约定。

    参考：[#1681](https://www.sqlalchemy.org/trac/ticket/1681)

+   **[orm]**

    “allow_null_pks”标志现在称为“allow_partial_pks”，默认为 True，像 0.5 中一样起作用。除此之外，它还在 merge()中实现，如果标志为 False，则不会为具有部分 NULL 主键的传入实例发出 SELECT。

    参考：[#1680](https://www.sqlalchemy.org/trac/ticket/1680)

+   **[orm]**

    修复了 0.6 重做的“多对一”优化中的 bug，即针对远程表上的非主键列（即针对唯一列的外键）的多对一在更改时会从数据库中拉取“旧”值，因为如果它在会话中，我们将需要它进行正确的历史/反向引用记账，而在非主键列上无法从本地标识映射中拉取。

    参考：[#1737](https://www.sqlalchemy.org/trac/ticket/1737)

+   **[orm]**

    修复了在单表继承关系上调用 has()或类似复杂表达式时可能发生的内部错误。

    参考：[#1731](https://www.sqlalchemy.org/trac/ticket/1731)

+   **[orm]**

    query.one()不再对查询应用 LIMIT，以确保它完全计算结果中存在的所有对象标识，即使在连接可能隐藏两个或更多行的多个标识的情况下也是如此。作为奖励，由于它不再修改查询，现在也可以使用从 from_statement()发出的查询调用 one()。

    参考：[#1688](https://www.sqlalchemy.org/trac/ticket/1688)

+   **[orm]**

    调用 query.get()现在会返回 None，如果查询的标识符在身份映射中以不同于请求的类的方式存在，即在使用多态加载时。

    参考：[#1727](https://www.sqlalchemy.org/trac/ticket/1727)

+   **[orm]**

    在 query.join()中进行了重大修复，当“on”子句是 aliased()构造的属性时，但已经存在对兼容目标的现有连接时，查询会正确地连接到正确的 aliased()构造，而不是粘附到现有连接的右侧。

    参考：[#1706](https://www.sqlalchemy.org/trac/ticket/1706)

+   **[orm]**

    对于不会在所谓的“行切换”操作中不必要地更新主键列的修复进行了轻微改进，即对具有相同主键的两个对象进行添加 + 删除。

    参考：[#1362](https://www.sqlalchemy.org/trac/ticket/1362)

+   **[orm]**

    现在在属性加载或刷新操作失败时使用 sqlalchemy.orm.exc.DetachedInstanceError，因为对象从任何会话中分离。UnboundExecutionError 专用于绑定到会话和语句的引擎。

+   **[orm]**

    在表达式上下文中调用的 Query 将在所有情况下呈现消除歧义的标签。请注意，这不适用于现有的 .statement 和 .subquery() 访问器/方法，它仍然遵循默认值为 False 的 .with_labels() 设置。

+   **[orm]**

    Query.union() 在返回的语句中保留了消除歧义的标签，从而避免了由列名冲突导致的各种 SQL 组合错误。

    参考：[#1676](https://www.sqlalchemy.org/trac/ticket/1676)

+   **[orm]**

    修复了属性历史中的错误，意外地在映射实例上调用了 __eq__。

+   **[orm]**

    对对象加载的一些内部优化使大型结果加速了一小部分，估计约为 10-15%。对“state”内部进行了彻底的清理，减少了复杂性、数据成员、方法调用以及空字典的创建。

+   **[orm]**

    对 query.delete() 进行了文档澄清。

    参考：[#1689](https://www.sqlalchemy.org/trac/ticket/1689)

+   **[orm]**

    修复了许多对一关系（many-to-one relation()）中级联 bug，在属性设置为 None 时引入，出现在 r6711 中（在 add() 过程中将已删除的项级联到会话中）。

+   **[orm]**

    在调用 query.select_from()、query.with_polymorphic() 或 query.from_statement() 之前调用 query.order_by() 或 query.distinct() 现在会引发异常，而不是静默地删除这些条件。

    参考：[#1736](https://www.sqlalchemy.org/trac/ticket/1736)

+   **[orm]**

    如果返回多于一行，则 query.scalar() 现在会引发异常。所有其他行为保持不变。

    参考：[#1735](https://www.sqlalchemy.org/trac/ticket/1735)

+   **[orm]**

    修复了使用 version_id_col 时“行切换”逻辑（即用 UPDATE 替换的 INSERT 和 DELETE）失败的 bug。

    参考：[#1692](https://www.sqlalchemy.org/trac/ticket/1692)

### 示例

+   **[示例]**

    稍微修改了 beaker 缓存示例，以便为懒加载缓存设置单独的 RelationCache 选项。通过将多个潜在属性分组到一个共同的结构中，此对象可以更有效地进行查找。FromCache 和 RelationCache 分别更简单。

### sql

+   **[sql]**

    join() 现在默认情况下会模拟 NATURAL JOIN。这意味着，如果左侧是一个连接，它将尝试首先将右侧连接到左侧的最右侧，并且如果成功，则不会引发有关模糊连接条件的任何异常，即使在左侧的其余连接目标中还有进一步的连接目标。

    参考：[#1714](https://www.sqlalchemy.org/trac/ticket/1714)

+   **[sql]**

    最常见的结果处理器转换函数已移至新的“processors”模块。鼓励方言作者在需要时使用这些函数，而不是实现自定义函数。

+   **[sql]**

    SchemaType 和子类 Boolean、Enum 现在是可序列化的，包括它们的 ddl 监听器和其他事件可调用函数。

    参考：[#1694](https://www.sqlalchemy.org/trac/ticket/1694), [#1698](https://www.sqlalchemy.org/trac/ticket/1698)

+   **[sql]**

    一些平台现在会将某些文字值解释为非绑定参数，直接呈现在 SQL 语句中。这是为了支持某些平台（包括 MS-SQL 和 Sybase）强制执行的严格 SQL-92 规则。在这种模式下，绑定参数不允许出现在 SELECT 的列子句中，也不允许出现某些模糊表达式如“?=?”。当启用此模式时，基本编译器将把绑定参数呈现为内联文字，但仅限于字符串和数值。其他类型如日期将会引发错误，除非方言子类为其定义了文字呈现函数。绑定参数必须已经包含嵌入的文字值，否则会引发错误（即无法使用直接的 bindparam(‘x’)）。方言还可以扩展绑定不被接受的领域，比如在函数的参数列表中（当使用本地 SQL 绑定时，在 MS-SQL 上无法工作）。

+   **[sql]**

    为 String、Unicode 等添加了“unicode_errors”参数。行为类似于标准库的 string.decode()函数的‘errors’关键字参数。此标志要求 convert_unicode 设置为“force” - 否则，SQLAlchemy 不能保证处理 unicode 转换的任务。请注意，对于已经本地返回 unicode 对象的后端（大多数 DBAPI 都是这样），此标志会给行提取操作带来显著的性能开销。此标志应仅作为从具有不同或损坏编码的列中读取字符串的绝对最后手段使用，这仅适用于首先接受无效编码的数据库（即 MySQL，*不*是 PG，Sqlite 等）。

+   **[sql]**

    增加了数学否定运算符支持，-x。

+   **[sql]**

    FunctionElement 子类现在可以直接执行，就像任何 func.foo()构造一样，在传递给 execute()时会自动应用 SELECT。

+   **[sql]**

    func.foo()构造的“type”和“bind”关键字参数现在局限于“func.”构造，并不是 FunctionElement 基类的一部分，允许在自定义构造函数或类级变量中处理“type”。

+   **[sql]**

    将 keys()方法恢复到 ResultProxy。

+   **[sql]**

    现在，类型/表达式系统会更完整地确定表达式的返回类型以及将 Python 运算符适应为 SQL 运算符，这是根据给定表达式的完整 left/right/operator 基础上进行的。特别是针对 PostgreSQL EXTRACT 创建的日期/时间/间隔系统现已被泛化到类型系统中。以前的行为通常是表达式“column + literal”强制“literal”的类型与“column”的类型相同，现在通常不会发生 - “literal”的类型首先是从字面的 Python 类型派生的，假设是标准的本机 Python 类型 + 日期类型，然后再回退到表达式另一侧的已知类型。如果“回退”类型是兼容的（即 String 的 CHAR），则字面量侧将使用该类型。TypeDecorator 类型默认覆盖此行为以无条件地强制转换“literal”侧，这可以通过实现 coerce_compared_value() 方法来更改。同样是部分。

    参考：[#1647](https://www.sqlalchemy.org/trac/ticket/1647), [#1683](https://www.sqlalchemy.org/trac/ticket/1683)

+   **[sql]**

    使 sqlalchemy.sql.expressions.Executable 成为公共 API 的一部分，用于可以发送到 execute() 的任何表达式构造。FunctionElement 现在继承了 Executable，因此它获得了 execution_options()，这些选项也会传播到 execute() 内部生成的 select()。Executable 又是 _Generative 的子类，标记了任何支持 @_generative 装饰器的 ClauseElement - 这些在某些时候也可能变为编译器扩展的“公共”。

+   **[sql]**

    解决了 - 用户定义的绑定参数名称直接与 update/insert 的 SET 或 VALUES 子句生成的列名绑定发生冲突的解决方案生成编译错误。这降低了调用次数，并消除了一些仍可能发生不良名称冲突的情况。

    参考：[#1579](https://www.sqlalchemy.org/trac/ticket/1579)

+   **[sql]**

    如果 Column() 没有外键，则需要一个类型（这不是新的）。如果 Column() 没有类型和没有外键，则会引发错误。

    参考：[#1705](https://www.sqlalchemy.org/trac/ticket/1705)

+   **[sql]**

    在将返回的浮点值强制转换为 Decimal 的字符串时，Numeric() 类型的“scale”参数会受到尊重 - 这允许 SQLite、MySQL 上的准确性功能。

    参考：[#1717](https://www.sqlalchemy.org/trac/ticket/1717)

+   **[sql]**

    Column 的 copy() 方法现在会复制未初始化的“on table attach”事件。有助于新的声明性“mixin”功能。

### mysql

+   **[mysql]**

    修复了反射 bug，即当 COLLATE 存在时，nullable 标志和服务器默认值不会反映出来。

    参考：[#1655](https://www.sqlalchemy.org/trac/ticket/1655)

+   **[mysql]**

    修复了 TINYINT(1) “boolean” 列的反射问题，该列使用像 UNSIGNED 这样的整数标志定义。

+   **[mysql]**

    进一步修复了 mysql-connector 方言。

    参考：[#1668](https://www.sqlalchemy.org/trac/ticket/1668)

+   **[mysql]**

    在 InnoDB 上的复合 PK 表中，“autoincrement”列不是第一个将在 CREATE TABLE 中发出显式的“KEY”短语，从而避免错误。

    参考：[#1496](https://www.sqlalchemy.org/trac/ticket/1496)

+   **[mysql]**

    为广泛的 MySQL 关键字增加了反射/创建表支持。

    参考：[#1634](https://www.sqlalchemy.org/trac/ticket/1634)

+   **[mysql]**

    修复了在 Windows 主机上反射表时可能发生的导入错误

    参考：[#1580](https://www.sqlalchemy.org/trac/ticket/1580)

### sqlite

+   **[sqlite]**

    在 create_engine()中添加了“native_datetime=True”标志。这将导致 DATE 和 TIMESTAMP 类型跳过所有绑定参数和结果行处理，假设连接上已启用 PARSE_DECLTYPES。请注意，这与“func.current_date()”不完全兼容，它将作为字符串返回。

    参考：[#1685](https://www.sqlalchemy.org/trac/ticket/1685)

### mssql

+   **[mssql]**

    重新建立了对 pymssql 方言的支持。

+   **[mssql]**

    隐式返回、反射等的各种修复 - MS-SQL 方言在 0.6 中还不完全（但接近了）

+   **[mssql]**

    增加了对 mxODBC 的基本支持。

    参考：[#1710](https://www.sqlalchemy.org/trac/ticket/1710)

+   **[mssql]**

    移除了 text_as_varchar 选项。

### oracle

+   **[oracle]**

    ”out”参数需要一个 cx_oracle 支持的类型。如果找不到 cx_oracle 类型，将引发错误。

+   **[oracle]**

    Oracle 的‘DATE’现在不执行任何结果处理，因为 Oracle 中的 DATE 类型存储完整的日期+时间对象，这就是你将得到的。请注意，通用类型.Date 类型仍将在传入值上调用 value.date()，但是在反射表时，反射的类型将是‘DATE’。

+   **[oracle]**

    增加了对 Oracle 的 WITH_UNICODE 模式的初步支持。至少在 Python 3 中建立了与 cx_Oracle 的初始支持。在 Python 2.xx 中使用 WITH_UNICODE 模式时，会发出一个大而可怕的警告，要求用户认真考虑这种困难的操作模式。

    参考：[#1670](https://www.sqlalchemy.org/trac/ticket/1670)

+   **[oracle]**

    except_()方法现在在 Oracle 上呈现为 MINUS，这在该平台上更或多是等效的。

    参考：[#1712](https://www.sqlalchemy.org/trac/ticket/1712)

+   **[oracle]**

    增加了对渲染和反射 TIMESTAMP WITH TIME ZONE 的支持，即 TIMESTAMP(timezone=True)。

    参考：[#651](https://www.sqlalchemy.org/trac/ticket/651)

+   **[oracle]**

    Oracle INTERVAL 类型现在可以反射。

### 杂项

+   **[py3k]**

    改进了关于 Python 3 的安装/测试设置，现在 Distribute 在 Py3k 上运行。现在包含 distribute_setup.py。请查看 README.py3k 以获取 Python 3 的安装/测试说明。

+   **[engines]**

    添加了一个可选的 C 扩展，通过重新实现 RowProxy 和最常见的结果处理器来加速 sql 层。实际的加速将严重依赖于您的 DBAPI 和表中使用的数据类型的混合，并且可以从 30%的改进到 200%以上不等。对于大查询，还为 ORM 速度提供了适度的（~15-20%）间接改进。请注意，默认情况下*不*构建/安装它。请参阅 README 以获取安装说明。

+   **[引擎]**

    在“自动提交”场景中，在调用 DBAPI 连接上的 commit()之前，执行顺序会从游标中提取所有的 rowcount/last inserted ID 信息。这有助于 mxodbc 处理 rowcount，并且总体上可能是一个好主意。

+   **[引擎]**

    打开了日志记录，使得更频繁地调用 isEnabledFor()，以便在下次连接时反映引擎/池的日志级别更改。这会增加一点方法调用开销。这是可以忽略的，将使得在调用 create_engine()后配置日志记录的所有情况变得更加容易。

    参考：[#1719](https://www.sqlalchemy.org/trac/ticket/1719)

+   **[引擎]**

    assert_unicode 标志已被弃用。当要求对非 unicode Python 字符串进行编码时，SQLAlchemy 将在所有情况下引发警告，以及当显式传递一个 bytestring 给 Unicode 或 UnicodeType 类型时。String 类型对于已经接受 Python unicode 对象的 DBAPI 不会执行任何操作。

+   **[引擎]**

    绑定参数将作为元组而不是列表发送。一些后端驱动程序将不接受绑定参数作为列表。

+   **[引擎]**

    线程本地引擎在 close()时未正确关闭连接 - 已修复。

+   **[引擎]**

    如果事务对象不是“活动”的话，将不会回滚或提交，允许更准确地嵌套 begin/rollback/commit。

+   **[引擎]**

    Python unicode 对象作为绑定结果会产生 Unicode 类型，而不是字符串，从而消除了一定类别的不支持 unicode 绑定的驱动程序上的 unicode 错误。

+   **[引擎]**

    在 create_engine()、Pool()构造函数中添加了“logging_name”参数，以及在 create_engine()中添加了“pool_logging_name”参数，该参数过滤到 Pool 的名称。在日志消息的“name”字段中发出给定的字符串名称，而不是默认的十六进制标识符字符串。

    参考：[#1555](https://www.sqlalchemy.org/trac/ticket/1555)

+   **[引擎]**

    Dialect 的 visit_pool()方法已被移除，并替换为 on_connect()。该方法返回一个可调用对象，在每次创建原始 DBAPI 连接后接收该连接。如果非 None，则由连接策略将其组装成一个 first_connect/connect 池监听器。为方言提供了更简单的接口。

+   **[引擎]**

    StaticPool 现在在不打开新连接的情况下初始化、处理和重新创建 - 仅在首次请求时才打开连接。dispose()现在也适用于 AssertionPool。

    参考：[#1728](https://www.sqlalchemy.org/trac/ticket/1728)

+   **[元数据] [票号：1673]**

    通过传递“schema=None”作为参数，现在可以在使用“tometadata”时剥离模式信息。如果未指定模式，则保留表的模式。

+   **[declarative]**

    DeclarativeMeta 专门使用 cls.__dict__（而不是 dict_）作为类信息的来源；_as_declarative 专门使用传递给它的 dict_ 作为类信息的来源（当使用 DeclarativeMeta 时，这是 cls.__dict__）。理论上，这应该使得自定义元类更容易修改传递给 _as_declarative 的状态。

+   **[declarative]**

    declarative 现在直接接受 mixin 类，作为在所有子类上提供共同功能和基于列的元素的手段，以及传播一组固定的 __table_args__ 或 __mapper_args__ 到子类的手段。对于从继承的 mixin 到本地的自定义组合 __table_args__/__mapper_args，现在可以使用描述符。有关所有新细节，请参阅 Declarative 文档。感谢 Chris Withers 在这方面的支持。

    参考：[#1707](https://www.sqlalchemy.org/trac/ticket/1707)

+   **[declarative]**

    在传播到子类时，__mapper_args__ 字典被复制，并直接从类 __dict__ 中取出，以避免从父类传播任何内容。映射器继承已经传播了您从父映射器中希望的内容。

    参考：[#1393](https://www.sqlalchemy.org/trac/ticket/1393)

+   **[declarative]**

    当单表子类指定已经存在于基类上的列时，会引发异常。

    参考：[#1732](https://www.sqlalchemy.org/trac/ticket/1732)

+   **[sybase]**

    实现了针对 Sybase 的初步工作方言，包括 Python-Sybase 和 Pyodbc 的子实现。处理表的创建/删除和基本的往返功能。目前还不包括反射或全面支持 unicode/特殊表达式等。

+   **[documentation]**

    在文档中进行了重要的清理工作，将类、函数和方法名称链接到 API 文档中。

    参考：[#1700](https://www.sqlalchemy.org/trac/ticket/1700)

### orm

+   **[orm]**

    relation()函数的官方名称现在是 relationship()，以消除关于关系代数术语的混淆。然而，relation()将在可预见的未来保持相同的功能。

    参考：[#1740](https://www.sqlalchemy.org/trac/ticket/1740)

+   **[orm]**

    向 Mapper 添加了“version_id_generator”参数，这是一个可调用对象，给定“version_id_col”的当前值，返回下一个版本号。可用于替代版本控制方案，如 uuid、时间戳等。

    参考：[#1692](https://www.sqlalchemy.org/trac/ticket/1692)

+   **[orm]**

    向 Session.refresh()添加了“lockmode”关键字参数，将字符串值传递给 Query，与 with_lockmode()中的操作相同，还将为启用 version_id_col 的映射执行版本检查。

+   **[orm]**

    修复了在联合继承场景中调用 query(A).join(A.bs).add_entity(B)会将 B 重复添加为目标并生成无效查询的错误。

    参考：[#1188](https://www.sqlalchemy.org/trac/ticket/1188)

+   **[orm]**

    修复了 session.rollback()中的错误，涉及在重新整合“已删除”对象之前未从会话中移除以前“挂起”对象的问题，通常出现在自然主键上。如果它们之间存在主键冲突，那么删除的附加将在内部失败。以前“挂起”的对象现在首先被清除。

    参考：[#1674](https://www.sqlalchemy.org/trac/ticket/1674)

+   **[orm]**

    移除了很多没有人真正关心的日志记录，保留的日志记录将响应日志级别的实时更改。不会增加显著的开销。

    参考：[#1719](https://www.sqlalchemy.org/trac/ticket/1719)

+   **[orm]**

    修复了 session.merge()中阻止类似字典的集合合并的错误。

+   **[orm]**

    session.merge()与明确不包括“merge”在其级联选项中的关系一起工作-目标完全被忽略。

+   **[orm]**

    如果目标具有该属性的值，则 session.merge()不会使现有目标上的现有标量属性过期，即使传入的合并没有该属性的值。这可以避免对现有项目进行不必要的加载。但是，如果目标没有该属性，它仍将标记为过期，这可以满足延迟列的某些约定。

    参考：[#1681](https://www.sqlalchemy.org/trac/ticket/1681)

+   **[orm]**

    “allow_null_pks”标志现在称为“allow_partial_pks”，默认为 True，像 0.5 中一样起作用。但是，如果标志为 False，则在 merge()中实现了这一点，对于具有部分 NULL 主键的传入实例不会发出 SELECT。

    参考：[#1680](https://www.sqlalchemy.org/trac/ticket/1680)

+   **[orm]**

    修复了 0.6 重做的“多对一”优化中的错误，使得针对远程表上的非主键列（即针对唯一列的外键）的多对一在更改时会从数据库中拉取“旧”值，因为如果它在会话中，我们将需要它进行正确的历史/反向引用计算，并且在非主键列上无法从本地标识映射中拉取。

    参考：[#1737](https://www.sqlalchemy.org/trac/ticket/1737)

+   **[orm]**

    修复了在单表继承关系上调用 has()或类似复杂表达式时可能出现的内部错误。

    参考：[#1731](https://www.sqlalchemy.org/trac/ticket/1731)

+   **[orm]**

    query.one()不再对查询应用 LIMIT，以确保它完全计算结果中存在的所有对象标识，即使在连接可能隐藏两个或更多行的多个标识的情况下也是如此。作为奖励，现在也可以使用从 from_statement()发出的查询调用 one()，因为它不再修改查询。

    参考：[#1688](https://www.sqlalchemy.org/trac/ticket/1688)

+   **[orm]**

    现在，如果查询一个在标识映射中具有不同类的标识符，即在使用多态加载时，query.get()会返回 None。

    参考：[#1727](https://www.sqlalchemy.org/trac/ticket/1727)

+   **[orm]**

    在 query.join()中进行了一个重大修复，当“on”子句是 aliased()构造的属性时，但已经存在一个指向兼容目标的现有连接时，query 会正确地连接到正确的 aliased()构造，而不是粘附到现有连接的右侧。

    参考：[#1706](https://www.sqlalchemy.org/trac/ticket/1706)

+   **[orm]**

    对于不发出不必要的更新主键列的修复进行了轻微改进，即在所谓的“行切换”操作期间，即两个具有相同 PK 的对象的添加+删除。

    参考：[#1362](https://www.sqlalchemy.org/trac/ticket/1362)

+   **[orm]**

    现在，当由于对象与任何会话分离而导致属性加载或刷新操作失败时，会使用 sqlalchemy.orm.exc.DetachedInstanceError。UnboundExecutionError 特定于绑定到会话和语句的引擎。

+   **[orm]**

    在表达式上调用的 Query 将在所有情况下呈现消除歧义的标签。请注意，这不适用于现有的.statement 和.subquery()访问器/方法，它仍然遵循默认为 False 的.with_labels()设置。

+   **[orm]**

    Query.union()在返回的语句中保留了消除歧义的标签，从而避免了由于列名冲突而导致的各种 SQL 组合错误。

    参考：[#1676](https://www.sqlalchemy.org/trac/ticket/1676)

+   **[orm]**

    修复了属性历史中的错误，无意中调用了映射实例上的 __eq__。

+   **[orm]**

    对对象加载进行了一些内部优化，为大结果提供了一点速度提升，估计在 10-15%左右。对“state”内部进行了彻底的清理，减少了复杂性，数据成员，方法调用，空字典的创建。

+   **[orm]**

    对 query.delete()进行了文档澄清

    参考：[#1689](https://www.sqlalchemy.org/trac/ticket/1689)

+   **[orm]**

    修复了 many-to-one relation()中的级联 bug，当属性设置为 None 时，在 r6711 中引入（在 add()期间将删除的项目级联到会话中）。

+   **[orm]**

    在调用 query.select_from()、query.with_polymorphic()或 query.from_statement()之前调用 query.order_by()或 query.distinct()现在会引��异常，而不是悄悄地丢弃这些条件。

    参考：[#1736](https://www.sqlalchemy.org/trac/ticket/1736)

+   **[orm]**

    现在，如果查询返回多行，query.scalar()会引发异常。所有其他行为保持不变。

    参考：[#1735](https://www.sqlalchemy.org/trac/ticket/1735)

+   **[orm]**

    修复了一个 bug，当 version_id_col 在使用时，“行切换”逻辑失败，即 INSERT 和 DELETE 被 UPDATE 替换。

    参考：[#1692](https://www.sqlalchemy.org/trac/ticket/1692)

### 示例

+   **[examples]**

    稍微修改了 beaker 缓存示例，为 lazyload 缓存添加了一个单独的 RelationCache 选项。这个对象通过将几个潜在属性分组到一个公共结构中，更有效地进行查找。FromCache 和 RelationCache 两者单独来说更简单。

### sql

+   **[sql]**

    join()现在默认模拟 NATURAL JOIN。这意味着，如果左侧是一个连接，它将尝试首先将右侧连接到左侧的最右侧，并且如果成功，即使在左侧的其余部分有更多的连接目标，也不会引发任何关于模糊连接条件的异常。

    参考：[#1714](https://www.sqlalchemy.org/trac/ticket/1714)

+   **[sql]**

    最常见的结果处理器转换函数已移至新的“processors”模块。鼓励方言作者在需要时使用这些函数，而不是实现自定义函数。

+   **[sql]**

    SchemaType 和其子类 Boolean、Enum 现在是可序列化的，包括它们的 ddl 监听器和其他事件可调用对象。

    参考：[#1694](https://www.sqlalchemy.org/trac/ticket/1694), [#1698](https://www.sqlalchemy.org/trac/ticket/1698)

+   **[sql]**

    现在，某些平台将解释某些文字值为非绑定参数，直接呈现到 SQL 语句中。这是为了支持一些平台（包括 MS-SQL 和 Sybase）强制执行的严格 SQL-92 规则。在这种模式下，不允许在 SELECT 的列子句中使用绑定参数，也不允许使用诸如“?=?”之类的模糊表达式。当启用此模式时，基本编译器将把绑定呈现为内联文字，但仅限于字符串和数值。其他类型如日期将引发错误，除非方言子类为其定义了文字呈现函数。绑定参数必须已经嵌入文字值，否则将引发错误（即不适用于直接 bindparam(‘x’)）。方言还可以扩展绑定不被接受的领域，例如在函数的参数列表中（当使用本地 SQL 绑定时，在 MS-SQL 上不起作用）。

+   **[sql]**

    在 String、Unicode 等中添加了“unicode_errors”参数。其行为类似于标准库的 string.decode()函数的‘errors’关键字参数。此标志要求将 convert_unicode 设置为“force” - 否则，SQLAlchemy 不能保证处理 Unicode 转换的任务。请注意，对于已经原生返回 Unicode 对象的后端（大多数 DBAPI 都是如此），此标志会给行提取操作增加显著的性能开销。此标志应仅在从具有不同或损坏编码的列中读取字符串时使用，这仅适用于首先接受无效编码的数据库（即 MySQL。*不是*PG、Sqlite 等）。

+   **[sql]**

    添加了数学否定运算符支持，-x。

+   **[sql]**

    FunctionElement 子类现在可以直接执行，与任何 func.foo() 结构一样，当传递给 execute() 时会自动应用 SELECT。

+   **[sql]**

    func.foo() 结构的“type”和“bind”关键字参数现在仅局限于“func.” 结构，并不属于 FunctionElement 基类，允许在自定义构造函数或类级别变量中处理“type”。

+   **[sql]**

    恢复了 ResultProxy 的 keys() 方法。

+   **[sql]**

    现在，类型/表达式系统在确定表达式的返回类型以及将 Python 运算符转换为 SQL 运算符方面做得更完整了，基于给定表达式的完整左/右/运算符。特别是为 PostgreSQL 中的 EXTRACT 创建的日期/时间/间隔系统现在已经泛化为类型系统。以前经常发生的“列 + 文字”的表达式强制“文字”的类型与“列”的类型相同的行为现在通常不会发生 - “文字”的类型首先从文字的 Python 类型推导出来，假设标准的原生 Python 类型 + 日期类型，然后再回退到表达式另一侧已知类型的类型。如果“回退”类型兼容（即来自 String 的 CHAR），则文字一侧将使用该类型。TypeDecorator 类型默认覆盖此行为，无条件地强制“文字”一侧，这可以通过实现 coerce_compared_value() 方法来更改。也是其中的一部分。

    参考资料：[#1647](https://www.sqlalchemy.org/trac/ticket/1647)，[#1683](https://www.sqlalchemy.org/trac/ticket/1683)

+   **[sql]**

    使 sqlalchemy.sql.expressions.Executable 成为公共 API 的一部分，用于可以发送到 execute() 的任何表达式构造。FunctionElement 现在继承 Executable，以便获得 execution_options()，这些选项也传播到 execute() 中生成的 select()。Executable 又继承了 _Generative，标记了任何支持 @_generative 装饰器的 ClauseElement - 这些也可能在某些时候成为编译器扩展的“公共”部分。

+   **[sql]**

    解决方案的变更为 - 一个端用户定义的绑定参数名称，它直接与来自 update/insert 的 SET 或 VALUES 子句生成的列命名绑定发生冲突时，将生成编译错误。这减少了调用次数，并消除了一些仍可能发生不良名称冲突的情况。

    参考资料：[#1579](https://www.sqlalchemy.org/trac/ticket/1579)

+   **[sql]**

    如果 Column() 没有外键，则需要类型（这不是新内容）。如果 Column() 没有类型和外键，则现在会引发错误。

    参考资料：[#1705](https://www.sqlalchemy.org/trac/ticket/1705)

+   **[sql]**

    Numeric() 类型的“scale”参数在将返回的浮点值转换为 Decimal 的字符串时被尊重 - 这允许 SQLite、MySQL 上的准确性工作。

    参考资料：[#1717](https://www.sqlalchemy.org/trac/ticket/1717)

+   **[sql]**

    Column 的 copy()方法现在会复制未初始化的“on table attach”事件。有助于新的声明性“mixin”功能。

### mysql

+   **[mysql]**

    修复了反射错误，即当 COLLATE 存在时，nullable 标志和服务器默认值不会被反映。

    参考：[#1655](https://www.sqlalchemy.org/trac/ticket/1655)

+   **[mysql]**

    修复了反射 TINYINT(1)“boolean”列时定义为 UNSIGNED 的整数标志的问题。

+   **[mysql]**

    进一步修复了 mysql-connector 方言的问题。

    参考：[#1668](https://www.sqlalchemy.org/trac/ticket/1668)

+   **[mysql]**

    在 InnoDB 上的复合主键表中，“autoincrement”列不是第一个将在 CREATE TABLE 中发出显式的“KEY”短语，从而避免错误。

    参考：[#1496](https://www.sqlalchemy.org/trac/ticket/1496)

+   **[mysql]**

    为许多 MySQL 关键字添加了反射/创建表支持。

    参考：[#1634](https://www.sqlalchemy.org/trac/ticket/1634)

+   **[mysql]**

    修复了在 Windows 主机上反射表时可能发生的导入错误。

    参考：[#1580](https://www.sqlalchemy.org/trac/ticket/1580)

### sqlite

+   **[sqlite]**

    添加了“native_datetime=True”标志到 create_engine()。这将导致 DATE 和 TIMESTAMP 类型跳过所有绑定参数和结果行处理，假设连接上已启用 PARSE_DECLTYPES。请注意，这与“func.current_date()”不完全兼容，它将返回一个字符串。

    参考：[#1685](https://www.sqlalchemy.org/trac/ticket/1685)

### mssql

+   **[mssql]**

    重新支持了 pymssql 方言。

+   **[mssql]**

    隐式返回、反射等方面的各种修复 - MS-SQL 方言在 0.6 版本中还不完全（但接近完成）

+   **[mssql]**

    添加了对 mxODBC 的基本支持。

    参考：[#1710](https://www.sqlalchemy.org/trac/ticket/1710)

+   **[mssql]**

    移除了 text_as_varchar 选项。

### oracle

+   **[oracle]**

    ”out”参数需要 cx_oracle 支持的类型。如果找不到 cx_oracle 类型，将引发错误。

+   **[oracle]**

    Oracle 的‘DATE’现在不执行任何结果处理，因为 Oracle 中的 DATE 类型存储完整的日期+时间对象，这就是你将得到的。请注意，通用类型 Date 类型*仍然*会在传入值上调用 value.date()。在反射表时，反射的类型将是‘DATE’。

+   **[oracle]**

    添加了对 Oracle 的 WITH_UNICODE 模式的初步支持。至少这为 Python 3 中的 cx_Oracle 建立了初始支持。在 Python 2.xx 中使用 WITH_UNICODE 模式时，会发出一个大而可怕的警告，要求用户认真考虑这种困难的操作模式的使用。

    参考：[#1670](https://www.sqlalchemy.org/trac/ticket/1670)

+   **[oracle]**

    在 Oracle��，except_()方法现在呈现为 MINUS，这在该平台上更或多或少是等效的。

    参考：[#1712](https://www.sqlalchemy.org/trac/ticket/1712)

+   **[oracle]**

    添加了对渲染和反射 TIMESTAMP WITH TIME ZONE 的支持，即 TIMESTAMP(timezone=True)。

    参考：[#651](https://www.sqlalchemy.org/trac/ticket/651)

+   **[oracle]**

    Oracle INTERVAL 类型现在可以反射。

### 杂项

+   **[py3k]**

    改进了关于 Python 3 的安装/测试设置，现在 Distribute 在 Py3k 上运行。distribute_setup.py 现在已包含在内。请参阅 README.py3k 以获取 Python 3 安装/测试说明。

+   **[engines]**

    添加了一个可选的 C 扩展，通过重新实现 RowProxy 和最常见的结果处理器来加速 sql 层。实际的加速将严重依赖于您的 DBAPI 和表中使用的数据类型的混合，并且可以从 30%的改进到 200%以上的变化。对于大查询，它还为 ORM 速度提供了适度的（~15-20%）间接改进。请注意，默认情况下*不*构建/安装它。请参阅 README 以获取安装说明。

+   **[engines]**

    在“自动提交”场景中，在调用 DBAPI 连接上的 commit()之前，执行顺序会从游标中提取所有 rowcount/最后插入的 ID 信息。这有助于 mxodbc 处理 rowcount，并且总体上可能是一个好主意。

+   **[engines]**

    打开了日志记录，使得更频繁地调用 isEnabledFor()，以便在下次连接时反映引擎/池的日志级别更改。这会增加一点方法调用开销。这是微不足道的，将使得在调用 create_engine()之后配置日志记录变得更加容易。

    参考：[#1719](https://www.sqlalchemy.org/trac/ticket/1719)

+   **[engines]**

    assert_unicode 标志已被弃用。在所有要求对非 Unicode Python 字符串进行编码的情况下，SQLAlchemy 都会发出警告，以及当 Unicode 或 UnicodeType 类型明确传递了字节字符串时。String 类型对于已经接受 Python unicode 对象的 DBAPI 不会执行任何操作。

+   **[engines]**

    绑定参数被发送为元组而不是列表。一些后端驱动程序将不接受绑定参数作为列表。

+   **[engines]**

    threadlocal 引擎在 close()时没有正确关闭连接 - 已修复。

+   **[engines]**

    如果事务对象不是“活动”的话，就不会回滚或提交，允许更准确地嵌套 begin/rollback/commit。

+   **[engines]**

    Python unicode 对象作为绑定结果会产生 Unicode 类型，而不是字符串，从而消除了一定类别的不支持 Unicode 绑定的驱动程序上的 Unicode 错误。

+   **[engines]**

    在 create_engine()、Pool()构造函数以及 create_engine()中添加了“logging_name”参数，该参数会过滤到 Pool 的名称。在日志消息的“name”字段中发出给定的字符串名称，而不是默认的十六进制标识符字符串。

    参考：[#1555](https://www.sqlalchemy.org/trac/ticket/1555)

+   **[engines]**

    Dialect 的 visit_pool() 方法已被移除，并替换为 on_connect()。此方法返回一个可调用对象，在每次创建原始 DBAPI 连接后接收该连接。如果非 None，则该可调用对象将被连接策略组装成一个 first_connect/connect 池监听器。为方言提供了更简单的接口。

+   **[engines]**

    StaticPool 现在在不打开新连接的情况下初始化、释放和重新创建 - 只有在首次请求时才会打开连接。dispose() 现在也适用于 AssertionPool。

    参考：[#1728](https://www.sqlalchemy.org/trac/ticket/1728)

+   **[metadata] [ticket: 1673]**

    添加了在使用“tometadata”时剥离模式信息的能力，通过传递“schema=None”作为参数。如果未指定模式，则保留表的模式。

+   **[declarative]**

    DeclarativeMeta 专门使用 cls.__dict__（而不是 dict_）作为类信息的来源；_as_declarative 专门使用传递给它的 dict_ 作为类信息的来源（当使用 DeclarativeMeta 时是 cls.__dict__）。理论上，这应该使得自定义元类更容易修改传递给 _as_declarative 的状态。

+   **[declarative]**

    declarative 现在直接接受 mixin 类，作为在所有子类上提供常见功能和基于列的元素的手段，以及传播一组固定的 __table_args__ 或 __mapper_args__ 到子类的手段。对于从继承的 mixin 到本地的自定义组合的 __table_args__/__mapper_args__，现在可以使用描述符。有关所有新细节，请参阅 Declarative 文档。感谢 Chris Withers 在这��面的支持。

    参考：[#1707](https://www.sqlalchemy.org/trac/ticket/1707)

+   **[declarative]**

    在传播到子类时，__mapper_args__ 字典被复制，并直接从类 __dict__ 中取出，以避免从父类传播。映射器继承已经传播了您从父映射器中想要的内容。

    参考：[#1393](https://www.sqlalchemy.org/trac/ticket/1393)

+   **[declarative]**

    当单表子类指定已经存在于基类上的列时，会引发异常。

    参考：[#1732](https://www.sqlalchemy.org/trac/ticket/1732)

+   **[sybase]**

    实现了一个初步可用的 Sybase 方言，包括 Python-Sybase 和 Pyodbc 的子实现。处理表的创建/删除和基本的往返功能。尚未包括反射或全面支持 unicode/特殊表达式等。

+   **[documentation]**

    在文档中进行了重大清理工作，将类、函数和方法名称链接到 API 文档中。

    参考：[#1700](https://www.sqlalchemy.org/trac/ticket/1700)

## 0.6beta1

发布日期：2010 年 2 月 3 日 星期三

### orm

+   **[orm]**

    query.update() 和 query.delete() 的更改：

    +   query.update() 上的 ‘expire’ 选项已重命名为 ‘fetch’，与 query.delete() 的匹配。‘expire’ 已被弃用并发出警告。

    +   query.update()和 query.delete()在同步策略上默认为“evaluate”。

    +   对于 update()和 delete()的“同步”策略在失败时会引发错误。没有隐式回退到“fetch”。评估的失败基于条件的结构，因此成功/失败是基于代码结构的确定性。

+   **[orm]**

    多对一关系的增强：

    +   多对一关系现在在更少的情况下触发延迟加载，包括在大多数情况下，当替换新值时不会获取“旧”值。

    +   多对一关系到一个连接表子类现在使用 get()进行简单加载（称为“use_get”条件），即 Related->Sub(Base)，无需重新定义基表的主连接条件。

    +   使用声明性列指定外键，即 ForeignKey(MyRelatedClass.id)不会破坏“use_get”条件的发生

    +   relation()，eagerload()和 eagerload_all()现在具有一个名为“innerjoin”的选项。指定 True 或 False 以控制急切连接是构建为内连接还是外连接。默认始终为 False。映射器选项将覆盖 relation()上指定的任何设置。通常应为多对一，非空外键关系设置，以允许改进的连接性能。

    +   急切加载的行为，即当 LIMIT/OFFSET 存在时，主查询被包装在子查询中，现在对所有急切加载都是多对一连接的情况做了一个例外。在这些情况下，急切连接直接针对父表进行，同时具有限制/偏移量，而不会增加子查询的额外开销，因为多对一连接不会向结果添加行。

    参考：[#1186](https://www.sqlalchemy.org/trac/ticket/1186), [#1492](https://www.sqlalchemy.org/trac/ticket/1492), [#1544](https://www.sqlalchemy.org/trac/ticket/1544)

+   **[orm]**

    Session.merge()的增强/更改：

+   **[orm]**

    “dont_load=True”标志在 Session.merge()上已被弃用，现在是“load=False”。

+   **[orm]**

    Session.merge()经过性能优化，在“load=False”模式下调用次数减少一半，与 0.5 相比，在“load=True”模式下在集合的情况下显著减少 SQL 查询。

+   **[orm]**

    如果给定实例与已经存在的实例相同，则 merge()不会发出属性的不必要合并。

+   **[orm]**

    merge()现在还会合并与给定状态相关联的“options”，即通过 query.options()传递的那些随实例一起传递的选项，例如急切加载或懒加载各种属性的选项。这对于构建高度集成的缓存方案至关重要。这与 0.5 版本相比是一个微妙的行为变化。

+   **[orm]**

    修复了关于实例状态上存在的“loader path”序列化的错误，这在将 merge()与序列化状态和应保留的相关选项结合使用时也是必要的。

+   **[orm]**

    全新的 merge()在一个新的全面示例中展示了如何将 Beaker 与 SQLAlchemy 集成。请参见下面的“examples”注释中的说明。

+   **[orm]**

    现在可以在联接表继承对象上更改主键值，并且在刷新时将考虑 ON UPDATE CASCADE。在使用 SQLite 或 MySQL/MyISAM 时，在 mapper()上设置新的“passive_updates”标志为 False。

    参考：[#1362](https://www.sqlalchemy.org/trac/ticket/1362)

+   **[orm]**

    flush()现在可以检测到主键列是否被另一个主键的 ON UPDATE CASCADE 操作更新，并且可以在刷新时定位用于对新 PK 值进行后续 UPDATE 的行。当存在 relation()以建立关系并且 passive_updates=True 时会发生这种情况。

    参考：[#1671](https://www.sqlalchemy.org/trac/ticket/1671)

+   **[orm]**

    “save-update”级联现在将挂起的*已移除*值级联到新会话中的 add()操作中。这样，flush()操作也将删除或修改这些断开连接项目的行。

+   **[orm]**

    使用“dynamic”加载器与“secondary”表现在会产生一个查询，其中“secondary”表不被别名化。这允许在 relation()的“order_by”属性中使用 secondary Table 对象，并且还允许在动态关系的筛选条件中使用它。

    参考：[#1531](https://www.sqlalchemy.org/trac/ticket/1531)

+   **[orm]**

    当 relation()的 uselist=False 在急切或延迟加载中找到多个有效值时，将发出警告。这可能是由于不适合急切 LEFT OUTER JOIN 或其他条件的 primaryjoin/secondaryjoin 条件造成的。

    参考：[#1643](https://www.sqlalchemy.org/trac/ticket/1643)

+   **[orm]**

    当 synonym()与 map_column=True 一起使用时，会显式检查是否在与相同键名一起发送到 mapper 的属性字典中存在单独的 ColumnProperty（延迟或其他）。而不是默默地替换现有属性（以及可能在该属性上的选项），会引发错误。

    参考：[#1633](https://www.sqlalchemy.org/trac/ticket/1633)

+   **[orm]**

    “dynamic”加载器在构造时设置其查询条件，以便从非克隆访问器（如“statement”）返回实际查询。

+   **[orm]**

    迭代 Query()时返回的“named tuple”对象现在是可 pickle 的。

+   **[orm]**

    映射到 select()构造现在要求您将其明确地制作为别名()。这是为了消除对诸如

    参考：[#1542](https://www.sqlalchemy.org/trac/ticket/1542)

+   **[orm]**

    query.join()已经重新设计，以提供更一致的行为和更多的灵活性（包括）

    参考：[#1537](https://www.sqlalchemy.org/trac/ticket/1537)

+   **[orm]**

    query.select_from() 接受多个子句以在 FROM 子句中产生多个逗号分隔的条目。在从多个 join()子句选择时非常有用。

+   **[orm]**

    query.select_from() 也接受映射类、别名构造和映射器作为参数。特别是在从多个连接表类查询时，确保完整连接被渲染时非常有用。

+   **[orm]**

    query.get() 可以与映射到外连接的情况一起使用，其中一个或多个主键值为 None。

    参考：[#1135](https://www.sqlalchemy.org/trac/ticket/1135)

+   **[orm]**

    query.from_self(), query.union(), 其他执行“SELECT * from (SELECT…)”类型嵌套的操作将更好地将子查询中的列表达式转换为外部查询的列子句。这可能与 0.5 版本不兼容，因为这可能会破坏没有应用标签的文字表达式的查询（即 literal(‘foo’), 等等）。

    参考：[#1568](https://www.sqlalchemy.org/trac/ticket/1568)

+   **[orm]**

    主连接和次连接现在检查它们是否是列表达式，而不仅仅是子句元素。这禁止了直接在那里放置 FROM 表达式等情况。

    参考：[#1622](https://www.sqlalchemy.org/trac/ticket/1622)

+   **[orm]**

    expression.null() 在使用 query.filter()、filter_by()等方法时与 None 的比较方式完全相同。

    参考：[#1415](https://www.sqlalchemy.org/trac/ticket/1415)

+   **[orm]**

    添加了“make_transient()”辅助函数，将持久化/分离实例转换为瞬态实例（即删除实例键并从任何会话中移除）。

    参考：[#1052](https://www.sqlalchemy.org/trac/ticket/1052)

+   **[orm]**

    mapper()上的 allow_null_pks 标志已被弃用，并且该功能默认为“on”。这意味着对于任何主键列具有非空值的行将被视为标识。通常只有在映射到外连接时才会出现这种情况。

    参考：[#1339](https://www.sqlalchemy.org/trac/ticket/1339)

+   **[orm]**

    “backref”的机制已完全合并到更精细的“back_populates”系统中，并完全在 RelationProperty 的 _generate_backref()方法中进行。这使得 RelationProperty 的初始化过程更简单，允许更容易地将设置（例如从 RelationProperty 的子类）传播到反向引用中。内部的 BackRef()已经消失，backref()返回一个普通元组，RelationProperty 能够理解。

+   **[orm]**

    mapper()上的 version_id_col 功能在使用不充分支持“rowcount”的方言时将引发警告。

    参考：[#1569](https://www.sqlalchemy.org/trac/ticket/1569)

+   **[orm]**

    在 Query 中添加了“execution_options()”，以便将选项传递给生成的语句。目前只有 Select 语句有这些选项，而且唯一使用的选项是“stream_results”，唯一知道“stream_results”的方言是 psycopg2。

+   **[orm]**

    Query.yield_per()将自动设置“stream_results”语句选项。

+   **[orm]**

    已弃用或移除：

    +   `mapper()`上的‘allow_null_pks’标志已被弃用。现在它不起作用，设置在所有情况下都是“on”。

    +   sessionmaker()和其他地方的“transactional”标志已移除。使用‘autocommit=True’表示‘transactional=False’。

    +   `mapper()`中的‘polymorphic_fetch’参数已移除。加载可以通过‘with_polymorphic’选项进行控制。

    +   `mapper()`中的‘select_table’参数已移除。使用‘with_polymorphic=(“*”, <some selectable>)’来实现此功能。

    +   `synonym()`中的‘proxy’参数已移除。此标志在 0.5 版本中没有任何作用，因为“proxy generation”行为现在是自动的。

    +   对`eagerload()`、`eagerload_all()`、`contains_eager()`、`lazyload()`、`defer()`和`undefer()`传递单个元素列表而不是多个位置参数已被弃用。

    +   对`query.order_by()`、`query.group_by()`、`query.join()`或`query.outerjoin()`传递单个元素列表而不是多个位置参数已被弃用。

    +   ��除了 query.iterate_instances()。使用 query.instances()。

    +   移除了 Query.query_from_parent()。使用 sqlalchemy.orm.with_parent()函数生成“parent”子句，或者使用 query.with_parent()。

    +   移除了 query._from_self()，请改用 query.from_self()。

    +   `composite()`中的“comparator”参数已移除。使用“comparator_factory”。

    +   移除了 RelationProperty._get_join()。

    +   Session 上的‘echo_uow’标志已移除。在“sqlalchemy.orm.unitofwork”名称上记录日志。

    +   移除了 session.clear()。使用 session.expunge_all()。

    +   移除了 session.save()、session.update()、session.save_or_update()。使用 session.add()和 session.add_all()。

    +   session.flush()上的“objects”标志仍然被弃用。

    +   session.merge()上的“dont_load=True”标志已被弃用，改用“load=False”。

    +   ScopedSession.mapper 仍然被弃用。请参阅[`www.sqlalchemy.org/trac/wiki/UsageRecipes/SessionAwareMapper`](https://www.sqlalchemy.org/trac/wiki/UsageRecipes/SessionAwareMapper)中的用法示例。

    +   将 InstanceState（内部 SQLAlchemy 状态对象）传递给 attributes.init_collection()或 attributes.get_history()已被弃用。这些函数是公共 API，通常期望一个常规映射对象实例。

    +   declarative_base()中的‘engine’参数已移除。使用‘bind’关键字参数。

### sql

+   **[sql]**

    select()和 text()上的“autocommit”标志以及 select().autocommit()已被弃用 - 现在在这些构造上调用.execution_options(autocommit=True)，也可以直接在 Connection 和 orm.Query 上使用。

+   **[sql]**

    现在，列上的 autoincrement 标志表示应链接到 cursor.lastrowid 的列，如果使用该方法。有关详细信息，请参阅 API 文档。

+   **[sql]**

    现在，executemany()要求所有绑定参数集中都要包含第一个绑定参数集中存在的所有键。插入/更新语句的结构和行为在很大程度上由第一个参数集确定，包括哪些默认值将触发，并且对其余所有参数不进行最少的猜测，以确保不影响性能。因此，否则默认值会对缺少的参数“失败”，因此现在已经防范。

    参考：[#1566](https://www.sqlalchemy.org/trac/ticket/1566)

+   **[sql]**

    returning()支持原生于 insert()、update()、delete()。对于 PostgreSQL、Firebird、MSSQL 和 Oracle，存在不同功能级别的实现。returning()可以显式调用列表达式，然后通过 fetchone()或 first()通常返回结果集。

    insert()构造还将隐式使用 RETURNING 来获取新生成的主键值，如果正在使用的数据库版本支持它（执行版本号检查）。如果没有指定最终用户 returning()，则会发生这种情况。

+   **[sql]**

    union()、intersect()、except()和其他“复合”类型的语句在括号方面具有更一致的行为。现在，嵌入在另一个中的每个复合元素将使用括号分组 - 以前，列表中的第一个复合元素不会被分组，因为 SQLite 不喜欢以括号开头的语句。然而，特别是 PostgreSQL 对于 INTERSECT 有优先规则，对于所有子元素应用括号更一致。因此，现在，SQLite 的解决方法也是以前 PG 的解决方法 - 在嵌套复合元素时，通常需要对第一个元素调用“.alias().select()”以将其包装在子查询中。

    参考：[#1665](https://www.sqlalchemy.org/trac/ticket/1665)

+   **[sql]**

    insert()和 update()构造现在可以使用与列键匹配的名称嵌入 bindparam()对象。这些绑定参数将绕过通常的路径，使这些键出现在生成的 SQL 的 VALUES 或 SET 子句中。

    参考：[#1579](https://www.sqlalchemy.org/trac/ticket/1579)

+   **[sql]**

    Binary 类型现在将数据作为 Python 字符串（或 Python 3 中的“bytes”类型）返回，而不是内置的“buffer”类型。这允许二进制数据进行对称往返。

    参考：[#1524](https://www.sqlalchemy.org/trac/ticket/1524)

+   **[sql]**

    添加了一个 tuple_()构造，允许将一组表达式与另一组进行比较，通常使用 IN 针对复合主键或类似的情况。还接受具有多个列的 IN。删除了“标量选择只能有一列”错误消息 - 将依赖数据库报告列不匹配的问题。

+   **[sql]**

    用户定义的“default”和“onupdate”可调用函数现在应该调用“context.current_parameters”来获取当前正在处理的绑定参数字典。无论是单次执行还是 executemany-style 语句执行，这个字典都以相同的方式可用。

+   **[SQL]**

    多部分模式名称，例如带有点号的“dbo.master”，现在在 select()标签中用下划线代替点号，即“dbo_master_table_column”。这是一个在结果集中表现更好的“友好”标签。

    参考：[#1428](https://www.sqlalchemy.org/trac/ticket/1428)

+   **[SQL]**

    移除了不必要的“counter”行为，即对于与表中列名匹配的 select()标签名生成“tablename_id”而不是“tablename_id_1”，以避免命名冲突，当表中实际上有一个名为“tablename_id”的列时 - 这是因为标签逻辑总是应用于所有列，因此永远不会发生命名冲突。

+   **[SQL]**

    调用 expr.in_([])，即使用空列表，会在发出通常的“expr != expr”子句之前发出警告。 “expr != expr”可能非常昂贵，最好是如果列表为空，则用户不发出 in_()，而是简单地不查询，或根据更复杂的情况修改条件。

    参考：[#1628](https://www.sqlalchemy.org/trac/ticket/1628)

+   **[SQL]**

    为 select()/text()添加了“execution_options()”，用于设置 Connection 的默认选项。请参见“engines”中的说明。

+   **[SQL]**

    已弃用或移除：

    +   select()上的“scalar”标志已移除，使用 select.as_scalar()。

    +   在 bindparam()上的“shortname”属性已移除。

    +   insert()，update()，delete()上的 postgres_returning，firebird_returning 标志已弃用，使用新的 returning()方法。

    +   join 上的 fold_equivalents 标志已弃用（将保留直到实现）

    参考：[#1131](https://www.sqlalchemy.org/trac/ticket/1131)

### 模式

+   **[模式]**

    MetaData 的 __contains__()方法现在接受字符串或 Table 对象作为参数。如果给定一个 Table，则首先将参数转换为 table.key，即“[schemaname.]<tablename>”

    参考：[#1541](https://www.sqlalchemy.org/trac/ticket/1541)

+   **[模式]**

    移除了不推荐使用的 MetaData.connect()和 ThreadLocalMetaData.connect() - 将“bind”属性发送到绑定元数据。

+   **[模式]**

    移除了不推荐使用的 metadata.table_iterator()方法（使用 sorted_tables）

+   **[模式]**

    不推荐使用 PassiveDefault - 使用 DefaultClause。

+   **[模式]**

    DefaultGenerator 及其子类中移除了“metadata”参数，但在 Sequence 上仍然存在，Sequence 是 DDL 中的一个独立构造。

+   **[模式]**

    从 Index 和 Constraint 对象中移除了公共可变性：

    > +   ForeignKeyConstraint.append_element()
    > +   
    > +   Index.append_column()
    > +   
    > +   UniqueConstraint.append_column()
    > +   
    > +   PrimaryKeyConstraint.add()
    > +   
    > +   PrimaryKeyConstraint.remove()

    这些应该以声明性的方式构建（即在一个构造中）。

+   **[模式]**

    Sequence 上的 “start” 和 “increment” 属性现在默认生成 “START WITH” 和 “INCREMENT BY”，在 Oracle 和 PostgreSQL 上。Firebird 目前不支持这些关键字。

    参���：[#1545](https://www.sqlalchemy.org/trac/ticket/1545)

+   **[schema]**

    UniqueConstraint、Index、PrimaryKeyConstraint 都接受列名或列对象的列表作为参数。

+   **[schema]**

    其他已移除的内容：

    +   Table.key（不知道这是用来做什么的）

    +   Table.primary_key 不可分配 - 使用 table.append_constraint(PrimaryKeyConstraint(…))

    +   Column.bind（通过 column.table.bind 获取）

    +   Column.metadata（通过 column.table.metadata 获取）

    +   Column.sequence（使用 column.default）

    +   ForeignKey(constraint=some_parent)（现在是私有 _constraint）

+   **[schema]**

    ForeignKey 上的 use_alter 标志现在是一个快捷选项，用于可以使用 DDL() 事件系统手动构建的操作。这次重构的副作用是，具有 use_alter=True 的 ForeignKeyConstraint 对象将 *不会* 在 SQLite 上发出，因为 SQLite 不支持外键的 ALTER。

+   **[schema]**

    ForeignKey 和 ForeignKeyConstraint 对象现在正确地复制所有它们的公共关键字参数。

    参考：[#1605](https://www.sqlalchemy.org/trac/ticket/1605)

### postgresql

+   **[postgresql]**

    新的方言：pg8000、zxjdbc 和 pypostgresql 在 py3k 上。

+   **[postgresql]**

    “postgres” 方言现在更名为 “postgresql”！连接字符串如下：

    > > postgresql://scott:tiger@localhost/test postgresql+pg8000://scott:tiger@localhost/test
    > > 
    > “postgres” 名称仍然保留以确保向后兼容性，具体如下：
    > 
    > > +   存在一个“postgres.py”虚拟方言，允许旧的 URL 正常工作，即 postgres://scott:tiger@localhost/test
    > > +   
    > > +   “postgres” 名称可以从旧的 “databases” 模块中导入，即 “from sqlalchemy.databases import postgres”，以及 “dialects”，“from sqlalchemy.dialects.postgres import base as pg”，将发出弃用警告。
    > > +   
    > > +   特殊表达式参数现在命名为 “postgresql_returning” 和 “postgresql_where”，但旧的 “postgres_returning” 和 “postgres_where” 名称仍然可以使用，会发出弃用警告。

+   **[postgresql]**

    ”postgresql_where” 现在接受 SQL 表达式，这些表达式也可以包含文字，需要时会加上引号。

+   **[postgresql]**

    psycopg2 方言现在在所有新连接上使用 psycopg2 的 “unicode extension”，这允许所有 String/Text 等类型跳过将字节字符串后处理为 Unicode 的步骤（由于其数量大，这是一个昂贵的步骤）。其他本地返回 Unicode 的方言（pg8000、zxjdbc）也跳过 Unicode 后处理。

+   **[postgresql]**

    添加了新的 ENUM 类型，作为一个模式级别的构造，并扩展了通用的 Enum 类型。自动与表及其父元数据关联，根据需要发出适当的 CREATE TYPE/DROP TYPE 命令，支持 Unicode 标签，支持反射。

    参考：[#1511](https://www.sqlalchemy.org/trac/ticket/1511)

+   **[postgresql]**

    INTERVAL 支持一个可选的“precision”参数，对应 PG 接受的参数。

+   **[postgresql]**

    使用新的 dialect.initialize() 功能来设置版本相关的行为。

+   **[postgresql]**

    对于表/列名称中的 % 符号提供了更好的支持；然而 psycopg2 无法处理绑定参数名为 %(foobar)s 的情况，SQLA 不想增加开销来处理这种不存在的用例。

    参考：[#1279](https://www.sqlalchemy.org/trac/ticket/1279)

+   **[postgresql]**

    向主键 + 外键列插入 NULL 将允许“非空约束”错误引发，而不是尝试执行不存在的“col_id_seq”序列。

    参考：[#1516](https://www.sqlalchemy.org/trac/ticket/1516)

+   **[postgresql]**

    自增 SELECT 语句，即从修改行的过程中选择的语句，现在可以与服务器端游标模式一起工作（命名游标不用于这种语句。）

+   **[postgresql]**

    postgresql 方言现在可以正确识别 pg 的“devel”版本字符串，例如“8.5devel”

    参考：[#1636](https://www.sqlalchemy.org/trac/ticket/1636)

+   **[postgresql]**

    psycopg2 现在尊重语句选项“stream_results”。此选项会覆盖连接设置“server_side_cursors”。如果为 true，则语句将使用服务器端游标。如果为 false，则不会使用，即使连接上的“server_side_cursors”为 true。

    参考：[#1619](https://www.sqlalchemy.org/trac/ticket/1619)

### mysql

+   **[mysql]**

    新的方言：oursql，一个新的本地方言，MySQL Connector/Python，MySQLdb 的本地 Python 移植，当然还有 Jython 上的 zxjdbc。

+   **[mysql]**

    VARCHAR/NVARCHAR 如果没有长度将无法渲染，在传递给 MySQL 之前会引发错误。不影响 CAST，因为在 MySQL CAST 中不允许 VARCHAR，在这种情况下方言会渲染 CHAR/NCHAR。

+   **[mysql]**

    所有 _detect_XXX() 函数现在在 dialect.initialize() 下运行一次

+   **[mysql]**

    对于表/列名称中的 % 符号提供了更好的支持；当使用 executemany() 时，MySQLdb 无法处理 SQL 中的 % 符号，SQLA 不想增加开销来处理这种不存在的用例。

    参考：[#1279](https://www.sqlalchemy.org/trac/ticket/1279)

+   **[mysql]**

    BINARY 和 MSBinary 类型现在在所有情况下生成“BINARY”。省略“length”参数将生成没有长度的“BINARY”。使用 BLOB 来生成一个无长度的二进制列。

+   **[mysql]**

    MSEnum/ENUM 的“quoting='quoted'”参数已被弃用。最好依赖自动引用。

+   **[mysql]**

    ENUM 现在是新的通用 Enum 类型的子类，并且如果给定的标签名是 unicode 对象，还会隐式处理 unicode 值。

+   **[mysql]**

    如果未传递“nullable=False”给 Column()，并且没有默认值，则 TIMESTAMP 类型的列现在默认为 NULL。这与所有其他类型保持一致，并且在 TIMESTAMP 的情况下明确渲染为“NULL”，因为 MySQL 对 TIMESTAMP 列的默认可空性进行了“切换”。

    参考：[#1539](https://www.sqlalchemy.org/trac/ticket/1539)

### sqlite

+   **[sqlite]**

    DATE、TIME 和 DATETIME 类型现在可以使用可选的 storage_format 和 regexp 参数。storage_format 可以用于使用自定义字符串格式存储这些类型。regexp 允许使用自定义正则表达式来匹配数据库中的字符串值。

+   **[sqlite]**

    时间和日期时间类型现在默认使用更严格的正则表达式来匹配数据库中的字符串。如果使用存储在传统格式中的数据，请使用 regexp 参数。

+   **[sqlite]**

    SQLite 时间和日期时间类型上的 __legacy_microseconds__ 不再受支持。您应该使用 storage_format 参数代替。

+   **[sqlite]**

    日期、时间和日期时间类型现在在接受绑定参数方面更加严格：日期类型只接受日期对象（以及日期时间对象，因为它们继承自日期），时间类型只接受时间对象，日期时间类型只接受日期和日期时间对象。

+   **[sqlite]**

    Table() 支持一个关键字参数“sqlite_autoincrement”，在生成 DDL 时将 SQLite 关键字“AUTOINCREMENT”应用于单个整数主键列。将阻止生成单独的 PRIMARY KEY 约束。

    参考：[#1016](https://www.sqlalchemy.org/trac/ticket/1016)

### mssql

+   **[mssql]**

    MSSQL + Pyodbc + FreeTDS 现在在很大程度上可以正常工作，可能会有关于二进制数据以及 Unicode 模式标识符的异常情况。

+   **[mssql]**

    “has_window_funcs”标志已被移除。LIMIT/OFFSET 的使用将像以往一样使用 ROW NUMBER，如果在较旧版本的 SQL Server 上，则操作将失败。行为完全相同，只是错误由 SQL Server 而不是方言引发，不需要设置标志来启用它。

+   **[mssql]**

    “auto_identity_insert”标志已被移除。当 INSERT 语句覆盖已知具有序列的列时，此功能始终生效。与“has_window_funcs”一样，如果底层驱动程序不支持此功能，则无论如何都无法执行此操作，因此没有设置标志的必要。

+   **[mssql]**

    使用新的 dialect.initialize() 功能来设置版本相关的行为。

+   **[mssql]**

    移除了不再使用的序列引用。在 MSSQL 中，隐式标识与其他方言上的隐式序列相同。通过使用“default=Sequence()”启用显式序列。有关更多信息，请参阅 MSSQL 方言文档。

### oracle

+   **[oracle]**

    使用 cx_oracle 进行的单元测试全部通过！

+   **[oracle]**

    支持 cx_Oracle 的“本地 Unicode”模式，不需要设置 NLS_LANG。请使用最新的 5.0.2 或更高版本的 cx_oracle。

+   **[oracle]**

    基本类型中添加了 NCLOB 类型。

+   **[oracle]**

    use_ansi=False 不会泄漏到从子查询中选择的语句的 FROM/WHERE 子句中，该子查询还使用了 JOIN/OUTERJOIN。

+   **[oracle]**

    向方言添加了本机 INTERVAL 类型。目前仅支持 DAY TO SECOND 的间隔类型，因为 cx_oracle 不支持 YEAR TO MONTH。

    参考：[#1467](https://www.sqlalchemy.org/trac/ticket/1467)

+   **[oracle]**

    使用 CHAR 类型会导致 cx_oracle 的 FIXED_CHAR dbapi 类型绑定到语句。

+   **[oracle]**

    Oracle 方言现在具有 NUMBER，旨在像 Oracle 的 NUMBER 类型一样运行。它是表反射返回的主要数值类型，并尝试根据精度/比例参数返回 Decimal()/float/int。

    参考：[#885](https://www.sqlalchemy.org/trac/ticket/885)

+   **[oracle]**

    func.char_length 是用于 LENGTH 的通用函数

+   **[oracle]**

    ForeignKey() 包括 onupdate=<value> 将发出警告，不会发出不受 Oracle 支持的 ON UPDATE CASCADE

+   **[oracle]**

    RowProxy() 的 keys() 方法现在返回已*标准化*为 SQLAlchemy 大小写不敏感名称的结果列名。这意味着对于大小写不敏感名称，它们将是小写的，而 DBAPI 通常会将它们返回为大写名称。这使得行键() 可与进一步的 SQLAlchemy 操作兼容。

+   **[oracle]**

    使用新的 dialect.initialize() 功能来设置版本相关的行为。

+   **[oracle]**

    在 Oracle 中使用 types.BigInteger 将生成 NUMBER(19)

    参考：[#1125](https://www.sqlalchemy.org/trac/ticket/1125)

+   **[oracle]**

    “大小写敏感”功能将在反射期间检测到全小写大小写敏感列名，并向生成的 Column 添加“quote=True”，以便保持正确的引用。

### 杂项

+   **[主要] [发布]**

    有关完整功能描述集，请参阅 [`docs.sqlalchemy.org/en/latest/changelog/migration_06.html`](https://docs.sqlalchemy.org/en/latest/changelog/migration_06.html) 。此文档正在进行中。

+   **[主要] [发布]**

    所有最新版本 0.5 及以下的所有错误修复和功能增强也包含在 0.6 中。

+   **[主要] [发布]**

    现在针对的平台包括 Python 2.4/2.5/2.6，Python 3.1，Jython2.5。

+   **[引擎]**

    可以使用 create_engine(… isolation_level=”…”) 指定事务隔离级别；在 postgresql 和 sqlite 上可用。

    参考：[#443](https://www.sqlalchemy.org/trac/ticket/443)

+   **[引擎]**

    Connection 具有 execution_options()，接受影响语句在 DBAPI 方面执行方式的关键字的生成方法。目前支持“stream_results”，导致 psycopg2 使用服务器端游标执行该语句，以及“autocommit”，这是 select() 和 text() 中“autocommit”选项的新位置。select() 和 text() 也有 .execution_options()，以及 ORM Query()。

+   **[引擎]**

    修复了为基于入口点驱动的方言导入不依赖于愚蠢的 tb_info 技巧来确定导入错误状态。

    参考：[#1630](https://www.sqlalchemy.org/trac/ticket/1630)

+   **[引擎]**

    在 ResultProxy 中添加了 first()方法，返回第一行并立即关闭结果集。

+   **[引擎]**

    RowProxy 对象现在是可 pickle 化的，即 result.fetchone()、result.fetchall()等返回的对象。

+   **[引擎]**

    RowProxy 不再有 close()方法，因为行不再保持对父行的引用。请在父 ResultProxy 上调用 close()，或使用 autoclose。

+   **[引擎]**

    ResultProxy 内部已进行了大规模的改进，大大减少了在获取列时的方法调用次数。在获取大型结果集时，可以提供大幅度的速度提升（高达 100%以上）。当获取没有应用类型级处理的列并且将结果用作元组（而不是字典）时，改进更大。非常感谢 Elixir 的 Gaëtan de Menten 为这一巨大改进所做的努力！

    参考：[#1586](https://www.sqlalchemy.org/trac/ticket/1586)

+   **[引擎]**

    依赖于“last inserted id”进行生成序列值的数据库（例如 MySQL、MS-SQL）现在在复合主键中正常工作，其中“autoincrement”列不是表中的第一个主键列。

+   **[引擎]**

    last_inserted_ids()方法已重命名为描述符“inserted_primary_key”。

+   **[引擎]**

    将 echo=False 设置在 create_engine()上现在会将日志级别设置为 WARN，而不是 NOTSET。这样，即使总体上启用了“sqlalchemy.engine”的日志记录，也可以为特定引擎禁用日志记录。请注意，“echo”的默认设置为 None。

    参考：[#1554](https://www.sqlalchemy.org/trac/ticket/1554)

+   **[引擎]**

    ConnectionProxy 现在对所有事务生命周期事件都有包装方法，包括 begin()、rollback()、commit()、begin_nested()、begin_prepared()、prepare()、release_savepoint()等。

+   **[引擎]**

    连接池日志现在使用 INFO 和 DEBUG 日志级别进行记录。INFO 用于主要事件，如无效的连接，DEBUG 用于所有获取/返回日志记录。echo_pool 可以是 False、None、True 或“debug”，方式与 echo 相同。

+   **[引擎]**

    所有 pyodbc 方言现在支持额外的 pyodbc 特定的 kw 参数‘ansi’、‘unicode_results’、‘autocommit’。

    参考：[#1621](https://www.sqlalchemy.org/trac/ticket/1621)

+   **[引擎]**

    “threadlocal”引擎已重写和简化，现在支持 SAVEPOINT 操作。

+   **[引擎]**

    已弃用或移除

    +   result.last_inserted_ids()已弃用。使用 result.inserted_primary_key

    +   dialect.get_default_schema_name(connection)现在通过 dialect.default_schema_name 公开。

    +   从 engine.transaction()和 engine.run_callable()中移除了“connection”参数——Connection 本身现在具有这些方法。所有四个方法都接受*args 和**kwargs，这些参数被传递给给定的可调用对象，以及操作连接。

+   **[反射/检查]**

    表反射已扩展并泛化为一个名为“sqlalchemy.engine.reflection.Inspector”的新 API。Inspector 对象提供关于各种模式信息的细粒度信息，包括表名、列名、视图定义、序列、索引等，还有扩展的空间。

+   **[反射/检查]**

    视图现在可以反映为普通的 Table 对象。使用相同的 Table 构造函数，但要注意“有效”的主键和外键约束不是反射结果的一部分；如果需要，必须明确指定这些。

+   **[反射/检查]**

    现有的 autoload=True 系统现在在底层使用 Inspector，以便每个方言只需返回关于表和其他对象的“原始”数据 - Inspector 是将信息编译成 Table 对象的单一位置，以确保一致性最大化。

+   **[ddl]**

    DDL 系统已大大扩展。DDL()类现在扩展了更通用的 DDLElement()，它构成了许多新构造的基础：

    > > +   CreateTable()
    > > +   
    > > +   DropTable()
    > > +   
    > > +   AddConstraint()
    > > +   
    > > +   DropConstraint()
    > > +   
    > > +   CreateIndex()
    > > +   
    > > +   DropIndex()
    > > +   
    > > +   CreateSequence()
    > > +   
    > > +   DropSequence()
    > > +   
    > 这些支持“on”和“execute-at()”，就像普通 DDL()一样。用户定义的 DDLElement 子类可以被创建并链接到一个编译器，使用 sqlalchemy.ext.compiler 扩展。

+   **[ddl]**

    传递给 DDL()和 DDLElement()���“on”可调用的签名如下所示：

    > ddl
    > 
    > DDLElement 对象本身
    > 
    > 事件
    > 
    > 字符串事件名称。
    > 
    > 目标
    > 
    > 以前是“schema_item”，触发事件的 Table 或 MetaData 对象。
    > 
    > 连接
    > 
    > 用于操作的 Connection 对象。
    > 
    > **kw
    > 
    > 关键字参数。在 MetaData 之前/之后创建/删除的情况下，要发出 CREATE/DROP DDL 的 Table 对象列表作为 kw 参数“tables”传递。这对于依赖于特定表存在的元数据级 DDL 是必要的。

    DDL 的“schema_item”属性已重命名为

    ”目标”。

+   **[方言] [重构]**

    方言模块现在分为数据库方言和 DBAPI 实现。现在更倾向于使用方言+驱动程序的连接 URL 来指定，即“mysql+mysqldb://scott:tiger@localhost/test”。请参阅 0.6 文档以获取示例。

+   **[方言] [重构]**

    外部方言的 setuptools 入口现在称为“sqlalchemy.dialects”。

+   **[方言] [重构]**

    从 Table 中删除了“owner”关键字参数。使用“schema”表示要预先添加到表名的任何命名空间。

+   **[方言] [重构]**

    server_version_info 成为静态属性。

+   **[方言] [重构]**

    方言在初始连接时接收到 initialize()事件，以确定连接属性。

+   **[方言] [重构]**

    方言在接收到 visit_pool 事件时有机会建立池监听器。

+   **[方言] [重构]**

    缓存的 TypeEngine 类现在按照每个方言类进行缓存，而不是按照每个方言进行缓存。

+   **[方言] [重构]**

    新的 UserDefinedType 应作为新类型的基类使用，以保留 get_col_spec() 的 0.5 行为。

+   **[方言] [重构]**

    所有类型类的 result_processor() 方法现在接受第二个参数“coltype”，这是来自 cursor.description 的 DBAPI 类型参数。此参数可以帮助某些类型决定对结果值进行最有效的处理。

+   **[方言] [重构]**

    废弃的 Dialect.get_params() 已移除。

+   **[方言] [重构]**

    Dialect.get_rowcount() 已重命名为描述符“rowcount”，并直接调用 cursor.rowcount。需要为某些调用硬编码行数的方言应重写该方法以提供不同的行为。

+   **[方言] [重构]**

    DefaultRunner 及其子类已移除。该对象的工作已简化并移至 ExecutionContext 中。支持序列的方言应在其执行上下文实现中添加 fire_sequence() 方法。

    参考：[#1566](https://www.sqlalchemy.org/trac/ticket/1566)

+   **[方言] [重构]**

    编译器生成的函数和运算符现在使用（几乎）常规的分发函数形式“visit_<opname>”和“visit_<funcname>_fn”来提供定制处理。这取代了在编译器子类中复制“functions”和“operators”字典的需要，改为使用直接的访问方法，并且还允许编译器子类完全控制渲染，因为完整的 _Function 或 _BinaryExpression 对象被传递进来。

+   **[firebird]**

    RowProxy() 的 keys() 方法现在返回结果列名*标准化*为 SQLAlchemy 不区分大小写的名称。这意味着对于不区分大小写的名称，它们将是小写的，而 DBAPI 通常会将它们返回为大写名称。这使得行键() 可与进一步的 SQLAlchemy 操作兼容。

+   **[firebird]**

    使用新的 dialect.initialize() 功能设置版本相关行为。

+   **[firebird]**

    “大小写敏感”功能将在反射期间检测到全小写的大小写敏感列名，并向生成的 Column 添加 “quote=True”，以保持适当的引用。

+   **[类型]**

    方言内部类型的构建已完全重构。方言现在仅以大写名称定义公开可用的类型，并使用下划线标识符（即私有）定义内部实现类型。表达类型在 SQL 和 DDL 中的系统已移至编译器系统。这样做的效果是大多数方言中的类型对象大大减少。有关方言作者的详细文档在 lib/sqlalchemy/dialects/type_migration_guidelines.txt 中。

+   **[类型]**

    类型不再对默认参数进行任何猜测。特别是，Numeric、Float、NUMERIC、FLOAT、DECIMAL 不会生成任何长度或精度，除非指定。

+   **[类型]**

    types.Binary 被重命名为 types.LargeBinary，它只产生 BLOB、BYTEA 或类似的“长二进制”类型。新增了基本的 BINARY 和 VARBINARY 类型，以一种与 MySQL/MS-SQL 特定类型无关的方式访问这些类型。

    参考：[#1664](https://www.sqlalchemy.org/trac/ticket/1664)

+   **[类型]**

    如果 dialect 检测到 DBAPI 原生返回 Python unicode 对象，则 String/Text/Unicode 类型现在会跳过每个结果列值上的 unicode()检查。此检查在首次连接时使用“SELECT CAST 'some text' AS VARCHAR(10)”或等效方式发出，然后检查返回的对象是否为 Python unicode。这允许对原生 unicode DBAPI（包括 pysqlite/sqlite3、psycopg2 和 pg8000）进行大幅性能提升。

+   **[类型]**

    大多数类型结果处理器已经经过检查，以寻找可能的速度改进。具体来说，以下通用类型已经经过优化，导致不同程度的速度改进：Unicode、PickleType、Interval、TypeDecorator、Binary。此外，以下特定于 dbapi 的实现已经得到改进：Sqlite 上的 Time、Date 和 DateTime，PostgreSQL 上的 ARRAY，MySQL 上的 Time，MySQL 上的 Numeric（as_decimal=False），oursql 和 pypostgresql，cx_oracle 上的 DateTime 以及 cx_oracle 上的 LOB 类型。

+   **[类型]**

    现在类型的反射将返回 types.py 中的确切大写类型，或者如果类型不是标准 SQL 类型，则返回 dialect 本身中的大写类型。这意味着反射现在返回有关反射类型的更准确信息。

+   **[类型]**

    添加了一个新的 Enum 通用类型。Enum 是一个 schema-aware 对象，用于支持需要特定 DDL 才能使用 enum 或等效的数据库；在 PG 的情况下，它处理 CREATE TYPE 的详细信息，在其他没有本地枚举支持的数据库上，将生成 VARCHAR +内联 CHECK 约束以强制执行枚举。

    参考：[#1109](https://www.sqlalchemy.org/trac/ticket/1109)，[#1511](https://www.sqlalchemy.org/trac/ticket/1511)

+   **[类型]**

    Interval 类型包括一个“native”标志，控制是否选择本机 INTERVAL 类型（postgresql + oracle）（如果可用）或不选择。还添加了“day_precision”和“second_precision”参数，适当地传播到这些本机类型。相关链接。

    参考：[#1467](https://www.sqlalchemy.org/trac/ticket/1467)

+   **[类型]**

    当在没有本地布尔支持的后端上使用 Boolean 类型时，将生成一个 CHECK 约束“col IN (0, 1)”以及基于 int/smallint 的列类型。如果需要，可以通过 create_constraint=False 关闭此功能。请注意，MySQL 没有本地布尔*或*CHECK 约束支持，因此该功能在该平台上不可用。

    参考：[#1589](https://www.sqlalchemy.org/trac/ticket/1589)

+   **[类型]**

    当 mutable=True 时，PickleType 现在使用 == 比较值，除非为类型指定了带有比较函数的 “comparator” 参数。如果未重写 __eq__() 或未提供比较函数，则将根据标识比较被 pickled 的对象（这会破坏 mutable=True 的目的）。

+   **[types]**

    Numeric 和 Float 的默认 “precision” 和 “scale” 参数已被移除，现在默认为 None。除非提供这些值，否则 NUMERIC 和 FLOAT 将默认不带数字参数呈现。

+   **[types]**

    AbstractType.get_search_list() 被移除 - 不再需要使用它的游戏。

+   **[types]**

    添加了一个通用的 BigInteger 类型，编译为 BIGINT 或 NUMBER(19)。

    参考：[#1125](https://www.sqlalchemy.org/trac/ticket/1125)

+   **[types]**

    sqlsoup 已进行了全面改进，明确支持 0.5 风格的会话，使用 autocommit=False，autoflush=True。SQLSoup 的默认行为现在需要通常使用 commit() 和 rollback()，这些已添加到其接口中。可以将显式的 Session 或 scoped_session 传递给构造函数，允许覆盖这些参数。

+   **[types]**

    sqlsoup db.<sometable>.update() 和 delete() 现在分别调用 query(cls).update() 和 delete()。

+   **[types]**

    sqlsoup 现在具有 execute() 和 connection()，它们调用那些名称的 Session 方法，确保绑定是基于 SqlSoup 对象的绑定。

+   **[types]**

    sqlsoup 对象不再具有 ‘query’ 属性 - 对于 sqlsoup 的使用范式而言不需要它，并且它会妨碍一个实际上命名为 ‘query’ 的列。

+   **[types]**

    传递给 association_proxy 的 proxy_factory 可调用对象的签名现在是 (lazy_collection, creator, value_attr, association_proxy)，添加了第四个参数，即父级 AssociationProxy 参数。允许内置集合的序列化和子类化。

    参考：[#1259](https://www.sqlalchemy.org/trac/ticket/1259)

+   **[types]**

    association_proxy 现在具有基本的比较方法 .any()、.has()、.contains()、==、!=，感谢 Scott Torborg。

    参考：[#1372](https://www.sqlalchemy.org/trac/ticket/1372)

### orm

+   **[orm]**

    对 query.update() 和 query.delete() 的更改：

    +   query.update() 上的 ‘expire’ 选项已重命名为 ‘fetch’，与 query.delete() 的命名相匹配。‘expire’ 已被弃用并发出警告。

    +   query.update() 和 query.delete() 默认都使用“evaluate”作为同步策略。

    +   update() 和 delete() 的“同步”策略在失败时会引发错误。没有隐式回退到“fetch”。评估的成功/失败基于条件的结构，因此成功/失败是基于代码结构的确定性的。

+   **[orm]**

    对于多对一关系的增强：

    +   多对一关系现在在更少的情况下触发延迟加载，包括在大多数情况下在替换新值时不会获取“旧”值。

    +   与联接表子类的多对一关系现在使用 get()进行简单加载（称为“use_get”条件），即 Related->Sub(Base)，无需重新定义基表的 primaryjoin 条件。

    +   使用声明性列指定外键，即 ForeignKey(MyRelatedClass.id)，不会阻止“use_get”条件的发生。

    +   relation()、eagerload()和 eagerload_all()现在具有名为“innerjoin”的选项。指定 True 或 False 以控制是否将急加载构建为 INNER 或 OUTER 连接。默认始终为 False。mapper 选项将覆盖 relation()上指定的任何设置。通常应为多对一、非空外键关系设置以允许改进的连接性能。

    +   对于急加载的行为，当存在 LIMIT/OFFSET 时，主查询现在会在所有急加载都是多对一连接时进行包装成子查询，这种情况下，急加载将直接针对父表进行连接，同时带有限制/偏移量，而不会增加子查询的额外开销，因为多对一连接不会向结果添加行。

    参考：[#1186](https://www.sqlalchemy.org/trac/ticket/1186), [#1492](https://www.sqlalchemy.org/trac/ticket/1492), [#1544](https://www.sqlalchemy.org/trac/ticket/1544)

+   **[orm]**

    Session.merge()的增强/更改：

+   **[orm]**

    在 Session.merge()上的“dont_load=True”标志已被弃用，现在是“load=False”。

+   **[orm]**

    Session.merge()经过性能优化，在“load=False”模式下的调用次数仅为 0.5 的一半，并且在“load=True”模式下对于集合的 SQL 查询显著减少。

+   **[orm]**

    如果给定的实例与已经存在的实例相同，则 merge()不会发出不必要的属性合并。

+   **[orm]**

    merge()现在还会合并与给定状态相关联的“options”，即通过 query.options()传递的那些随实例一起传递的选项，例如急加载或延迟加载各种属性的选项。这对于构建高度集成的缓存方案至关重要。这是与 0.5 版本相比的一个微妙的行为变化。

+   **[orm]**

    修复了关于实例状态中存在的“loader path”序列化的 bug，这在将 merge()与应保留的序列化状态和相关选项结合使用时也是必要的。

+   **[orm]**

    全新的 merge()在一个全面的示例中展示了如何将 Beaker 与 SQLAlchemy 集成。请参见下面的“示例”注释中的说明。

+   **[orm]**

    现在可以在联接表继承对象上更改主键值，并且在刷新时将考虑 ON UPDATE CASCADE。在使用 SQLite 或 MySQL/MyISAM 时，在 mapper()上设置新的“passive_updates”标志为 False。

    参考：[#1362](https://www.sqlalchemy.org/trac/ticket/1362)

+   **[orm]**

    flush() 现在可以检测到主键列是否被另一个主键的 ON UPDATE CASCADE 操作更新，并且可以在新的主键值上进行后续 UPDATE 时定位行。当存在 relation() 来建立关系以及 passive_updates=True 时会发生这种情况。

    参考：[#1671](https://www.sqlalchemy.org/trac/ticket/1671)

+   **[orm]**

    “save-update” 级联现在会将标量或集合属性中待处理的*移除*值级联到新会话中的 add() 操作中。这样，flush() 操作也会删除或修改这些断开连接项目的行。

+   **[orm]**

    使用“dynamic”加载器与“secondary”表现在会生成一个查询，其中“secondary”表不会被别名化。这允许在 relation() 的“order_by”属性中使用“secondary”表对象，并且还允许在动态关系的筛选条件中使用它。

    参考：[#1531](https://www.sqlalchemy.org/trac/ticket/1531)

+   **[orm]**

    当 relation() 的 uselist=False 时，当 eager 或 lazy load 定位到多个有效值时会发出警告。这可能是由于 primaryjoin/secondaryjoin 条件不适用于 eager LEFT OUTER JOIN 或其他条件。

    参考：[#1643](https://www.sqlalchemy.org/trac/ticket/1643)

+   **[orm]**

    当使用 synonym() 与 map_column=True 时，会显式检查是否在发送到 mapper 的属性字典中存在与同一键名的 ColumnProperty（延迟或其他方式）。不会静默替换现有属性（以及可能存在的属性选项），而是会引发错误。

    参考：[#1633](https://www.sqlalchemy.org/trac/ticket/1633)

+   **[orm]**

    “dynamic”加载器在构建时设置其查询条件，以便从非克隆访问器（如“statement”）返回实际查询。

+   **[orm]**

    迭代 Query() 时返回的“named tuple”对象现在是可 pickle 的。

+   **[orm]**

    映射到 select() 构造现在要求您将其明确地制作为 alias()。这是为了消除诸如此类问题的混淆

    参考：[#1542](https://www.sqlalchemy.org/trac/ticket/1542)

+   **[orm]**

    重新设计了 query.join()，以提供更一致的行为和更灵活的功能（包括）。

    参考：[#1537](https://www.sqlalchemy.org/trac/ticket/1537)

+   **[orm]**

    query.select_from() 接受多个子句以在 FROM 子句中生成多个逗号分隔的条目。在从多个 join() 子句中选择时很有用。

+   **[orm]**

    query.select_from() 还接受映射类、aliased() 构造和 mappers 作为参数。特别是在从多个连接表类查询时，确保完整连接被渲染。

+   **[orm]**

    query.get() 可以与映射到一个外连接的情况一起使用，其中一个或多个主键值为 None。

    参考：[#1135](https://www.sqlalchemy.org/trac/ticket/1135)

+   **[orm]**

    query.from_self()、query.union()等执行“SELECT * from (SELECT…)”类型嵌套的操作将更好地将子查询中的列表达式转换为外部查询的列子句。这可能与 0.5 版本不兼容，因为这可能会破坏没有应用标签的文字表达式的查询（即 literal(‘foo’)等）。

    参考：[#1568](https://www.sqlalchemy.org/trac/ticket/1568)

+   **[orm]**

    relation primaryjoin 和 secondaryjoin 现在检查它们是否是列表达式，而不仅仅是子句元素。这禁止了直接在那里放置 FROM 表达式等情况。

    参考：[#1622](https://www.sqlalchemy.org/trac/ticket/1622)

+   **[orm]**

    expression.null()在使用 query.filter()、filter_by()等时与 None 的比较方式完全相同。

    参考：[#1415](https://www.sqlalchemy.org/trac/ticket/1415)

+   **[orm]**

    添加了“make_transient()”辅助函数，将持久化/分离实例转换为瞬态实例（即删除实例键并从任何会话中移除）。

    参考：[#1052](https://www.sqlalchemy.org/trac/ticket/1052)

+   **[orm]**

    在 mapper()上的 allow_null_pks 标志已被弃用，并且该功能默认为“on”。这意味着对于任何主键列具有非空值的行将被视为标识。通常只有在映射到外连接时才会出现这种情况。

    参考：[#1339](https://www.sqlalchemy.org/trac/ticket/1339)

+   **[orm]**

    “backref”的机制已完全合并到更精细的“back_populates”系统中，并完全在 RelationProperty 的 _generate_backref()方法中进行。这使得 RelationProperty 的初始化过程更简单，允许更容易地将设置（例如从 RelationProperty 的子类）传播到反向引用中。内部的 BackRef()已经消失，backref()返回一个 RelationProperty 理解的普通元组。 

+   **[orm]**

    在 mapper()上的 version_id_col 功能在使用不充分支持“rowcount”的方言时会发出警告。

    参考：[#1569](https://www.sqlalchemy.org/trac/ticket/1569)

+   **[orm]**

    在 Query 中添加了“execution_options()”，以便可以将选项传递给生成的语句。目前只有 Select 语句有这些选项，而且唯一使用的选项是“stream_results”，唯一知道“stream_results”的方言是 psycopg2。

+   **[orm]**

    Query.yield_per()将自动设置“stream_results”语句选项。

+   **[orm]**

    已弃用或移除：

    +   在 mapper()上的’allow_null_pks’标志已被弃用。现在它什么也不做，而且在所有情况下设置为“on”。

    +   在 sessionmaker()和其他地方的’transactional’标志已被移除。使用‘autocommit=True’来表示‘transactional=False’。

    +   在 mapper()上的’polymorphic_fetch’参数已被移除。加载可以使用‘with_polymorphic’选项来控制。

    +   mapper() 上的 ‘select_table’ 参数已被移除。对于此功能，请使用 ‘with_polymorphic=(“*”, <some selectable>)’。

    +   synonym() 上的 ‘proxy’ 参数已被移除。这个标志在 0.5 版本中没有起作用，因为“代理生成”行为现在是自动的。

    +   将一组元素传递给 eagerload()、eagerload_all()、contains_eager()、lazyload()、defer() 和 undefer() 而不是多个位置参数已被弃用。

    +   将一组元素传递给 query.order_by()、query.group_by()、query.join() 或 query.outerjoin() 而不是多个位置参数已被弃用。

    +   query.iterate_instances() 被移除。使用 query.instances()。

    +   Query.query_from_parent() 被移除。使用 sqlalchemy.orm.with_parent() 函数生成一个“parent”子句，或者使用 query.with_parent()。

    +   query._from_self() 被移除，改用 query.from_self()。

    +   composite() 上的 “comparator” 参数已被移除。使用 “comparator_factory”。

    +   RelationProperty._get_join() 被移除。

    +   Session 上的 ‘echo_uow’ 标志已被移除。在 “sqlalchemy.orm.unitofwork” 名称上使用日志记录。

    +   session.clear() 被移除。使用 session.expunge_all()。

    +   session.save()、session.update()、session.save_or_update() 被移除。使用 session.add() 和 session.add_all()。

    +   session.flush() 上的 “objects” 标志仍然被弃用。

    +   session.merge() 上的 “dont_load=True” 标志已被弃用，改用 “load=False”。

    +   ScopedSession.mapper 仍然被弃用。请参阅 [`www.sqlalchemy.org/trac/wiki/UsageRecipes/SessionAwareMapper`](https://www.sqlalchemy.org/trac/wiki/UsageRecipes/SessionAwareMapper) 上的用法示例。

    +   将 InstanceState（内部 SQLAlchemy 状态对象）传递给 attributes.init_collection() 或 attributes.get_history() 已被弃用。这些函数是公共 API，通常期望一个常规映射对象实例。

    +   declarative_base() 上的 ‘engine’ 参数已被移除。使用 ‘bind’ 关键字参数。

### sql

+   **[sql]**

    select() 和 text() 上的 “autocommit” 标志以及 select().autocommit() 都已被弃用 - 现在在这些构造之一上调用 .execution_options(autocommit=True)，也可以直接在 Connection 和 orm.Query 上使用。

+   **[sql]**

    column 上的 autoincrement 标志现在指示应链接到 cursor.lastrowid 的列，如果使用该方法。有关详细信息，请参阅 API 文档。

+   **[sql]**

    现在，executemany() 要求所有绑定的参数集中都必须包含第一个绑定的参数集中存在的所有键。插入/更新语句的结构和行为在很大程度上由第一个参数集确定，包括哪些默认值将触发，所有其他参数集都会进行最少的猜测，以确保不影响性能。因此，默认值否则会对缺少的参数“失败”，因此现在已经受到保护。

    参考：[#1566](https://www.sqlalchemy.org/trac/ticket/1566)

+   **[sql]**

    insert()、update()、delete()现在原生支持 returning()。对于 PostgreSQL、Firebird、MSSQL 和 Oracle，存在不同级别功能的实现。returning()可以显式调用列表达式，然后通过 fetchone()或 first()通常返回结果集。

    如果正在使用的数据库版本支持它（执行版本号检查），insert()构造也将隐式使用 RETURNING 来获取新生成的主键值。如果没有指定最终用户 returning()，则会发生这种情况。

+   **[sql]**

    union()、intersect()、except()和其他“复合”类型的语句在括号方面有更一致的行为。现在，嵌套在另一个中的每个复合元素将使用括号分组 - 以前，列表中的第一个复合元素不会被分组，因为 SQLite 不喜欢以括号开头的语句。然而，特别是 PostgreSQL 对 INTERSECT 有优先规则，并且对所有子元素应用括号更一致。因此，现在，SQLite 的解决方法也是以前 PG 的解决方法 - 在嵌套复合元素时，通常需要对第一个元素调用“.alias().select()”来将其包装在子查询中。

    参考：[#1665](https://www.sqlalchemy.org/trac/ticket/1665)

+   **[sql]**

    insert()和 update()构造现在可以使用与列键匹配的名称嵌入 bindparam()对象。这些绑定参数将绕过通常导致这些键出现在生成的 SQL 的 VALUES 或 SET 子句中的路径。

    参考：[#1579](https://www.sqlalchemy.org/trac/ticket/1579)

+   **[sql]**

    Binary 类型现在将数据作为 Python 字符串（或 Python 3 中的“bytes”类型）返回，而不是内置的“buffer”类型。这允许二进制数据的对称往返。

    参考：[#1524](https://www.sqlalchemy.org/trac/ticket/1524)

+   **[sql]**

    添加了一个 tuple_()构造，允许将一组表达式与另一组进行比较，通常使用 IN 针对复合主键或类似的情况。还接受具有多个列的 IN。删除了“标量选择只能有一列”错误消息 - 将依赖数据库报告列不匹配的问题。

+   **[sql]**

    现在，接受上下文的用户定义的“default”和“onupdate”可调用程序应该调用“context.current_parameters”来获取当前正在处理的绑定参数字典。无论是单次执行还是 executemany-style 语句执行，这个字典都以相同的方式可用。

+   **[sql]**

    多部分模式名称，例如带有点号的“dbo.master”，现在在 select()标签中以下划线代替点号，即“dbo_master_table_column”。这是一个在结果集中表现更好的“友好”标签。

    参考：[#1428](https://www.sqlalchemy.org/trac/ticket/1428)

+   **[sql]**

    移除了不必要的“counter”行为，当 select() 标签名称与表中的列名匹配时，例如为“id”生成“tablename_id”，而不是尝试避免命名冲突生成“tablename_id_1”，当表中实际上有一个名为“tablename_id”的列时，因为标签逻辑总是应用于所有列，所以永远不会发生命名冲突。

+   **[sql]**

    调用 expr.in_([])，即使用空列表，会在发出通常的“expr != expr”子句之前发出警告。 “expr != expr” 可能非常昂贵，最好用户在列表为空时不发出 in_()，而是简单地不查询，或根据更复杂的情况修改条件。

    参考：[#1628](https://www.sqlalchemy.org/trac/ticket/1628)

+   **[sql]**

    在 select()/text() 中添加了“execution_options()”，用于为 Connection 设置默认选项。请参阅“engines”中的说明。

+   **[sql]**

    弃用或移除的内容：

    +   在 select() 上的“scalar”标志被移除，使用 select.as_scalar()。

    +   bindparam() 上的“shortname”属性已被移除。

    +   在 insert()、update()、delete() 上弃用了 postgres_returning、firebird_returning 标志，使用新的 returning() 方法。

    +   join 上的 fold_equivalents 标志被弃用（将保留直到实现）。

    参考：[#1131](https://www.sqlalchemy.org/trac/ticket/1131)

### 模式

+   **[schema]**

    MetaData 的 __contains__() 方法现在接受字符串或 Table 对象作为参数。如果给定一个 Table，参数首先转换为 table.key，即“[schemaname.]<tablename>”。

    参考：[#1541](https://www.sqlalchemy.org/trac/ticket/1541)

+   **[schema]**

    弃用的 MetaData.connect() 和 ThreadLocalMetaData.connect() 已被移除 - 将“bind”属性发送到绑定元数据。

+   **[schema]**

    移除了弃用的 metadata.table_iterator() 方法（使用 sorted_tables）

+   **[schema]**

    弃用的 PassiveDefault - 使用 DefaultClause。

+   **[schema]**

    DefaultGenerator 和子类中移除了“metadata”参数，但在 Sequence 上仍然本地存在，Sequence 是 DDL 中的一个独立构造。

+   **[schema]**

    从 Index 和 Constraint 对象中移除了公共的可变性：

    > +   ForeignKeyConstraint.append_element()
    > +   
    > +   Index.append_column()
    > +   
    > +   UniqueConstraint.append_column()
    > +   
    > +   PrimaryKeyConstraint.add()
    > +   
    > +   PrimaryKeyConstraint.remove()

    这些应该以声明方式构建（即在一个构造中）。

+   **[schema]**

    Sequence 上的“start”和“increment”属性现在默认生成“START WITH”和“INCREMENT BY”，在 Oracle 和 PostgreSQL 上。 Firebird 目前不支持这些关键字。

    参考：[#1545](https://www.sqlalchemy.org/trac/ticket/1545)

+   **[schema]**

    UniqueConstraint、Index、PrimaryKeyConstraint 都接受列名或列对象的列表作为参数。

+   **[schema]**

    其他被移除的内容：

    +   Table.key（不知道这是用来做什么的）

    +   Table.primary_key 不可分配 - 使用 table.append_constraint(PrimaryKeyConstraint(…))

    +   Column.bind（通过 column.table.bind 获取）

    +   Column.metadata（通过 column.table.metadata 获取）

    +   Column.sequence（使用 column.default）

    +   ForeignKey（约束=some_parent）（现在是私有 _constraint）

+   **[schema]**

    ForeignKey 上的 use_alter 标志现在是使用 DDL() 事件系统手工构造的操作的快捷选项。此重构的一个副作用是，具有 use_alter=True 的 ForeignKeyConstraint 对象将不会在 SQLite 上生成，因为 SQLite 不支持外键的 ALTER。

+   **[schema]**

    ForeignKey 和 ForeignKeyConstraint 对象现在正确地复制了它们所有的公共关键字参数。

    参考：[#1605](https://www.sqlalchemy.org/trac/ticket/1605)

### postgresql

+   **[postgresql]**

    新的方言：pg8000、zxjdbc 和 pypostgresql 在 py3k 上。

+   **[postgresql]**

    “postgres”方言现在命名为“postgresql”！连接字符串如下所示：

    > > postgresql://scott:tiger@localhost/test postgresql+pg8000://scott:tiger@localhost/test
    > > 
    > “postgres”名称在以下方面保持向后兼容性：
    > 
    > > +   有一个“postgres.py”虚拟方言，允许旧的 URL 工作，即 postgres://scott:tiger@localhost/test
    > > +   
    > > +   “postgres”名称可以从旧的“databases”模块导入，即“from sqlalchemy.databases import postgres”以及“dialects”、“from sqlalchemy.dialects.postgres import base as pg”，将发送警告。
    > > +   
    > > +   特殊表达式参数现在命名为“postgresql_returning”和“postgresql_where”，但较旧的“postgres_returning”和“postgres_where”名称仍然可以与警告进行使用。

+   **[postgresql]**

    “postgresql_where”现在接受 SQL 表达式，该表达式还可以包括字面值，需要时将进行引用。

+   **[postgresql]**

    psycopg2 方言现在在所有新连接上使用 psycopg2 的“Unicode 扩展”，这允许所有 String/Text/等类型跳过将字节字符串后处理为 Unicode 的需要（由于其数量而成为昂贵的步骤）。其他本机返回 Unicode 的方言（pg8000、zxjdbc）也跳过了 Unicode 后处理。

+   **[postgresql]**

    添加了新的枚举类型，作为模式级构造存在，并扩展了通用的枚举类型。自动将自身与表及其父级元数据关联起来，以根据需要发出适当的 CREATE TYPE/DROP TYPE 命令，支持 Unicode 标签，支持反射。

    参考：[#1511](https://www.sqlalchemy.org/trac/ticket/1511)

+   **[postgresql]**

    INTERVAL 支持一个可选的“精度”参数，对应于 PG 接受的参数。

+   **[postgresql]**

    使用新的 dialect.initialize() 特性来设置版本相关的行为。

+   **[postgresql]**

    对表/列名称中的%符号提供了更好的支持；然而，psycopg2 无法处理绑定参数名称为%(foobar)s 的情况，而 SQLA 不希望仅为处理那种不存在的用例而增加开销。

    参考：[#1279](https://www.sqlalchemy.org/trac/ticket/1279)

+   **[postgresql]**

    将 NULL 插入主键 + 外键列将允许引发“非空约束”错误，而不是尝试执行不存在的“col_id_seq”序列。

    参考：[#1516](https://www.sqlalchemy.org/trac/ticket/1516)

+   **[postgresql]**

    自增 SELECT 语句，即从修改行的过程中选择的语句，现在可以使用服务器端游标模式（对于这些语句不使用命名游标）。

+   **[postgresql]**

    postgresql dialect 现在可以正确检测 pg“devel”版本字符串，即“8.5devel”。

    参考：[#1636](https://www.sqlalchemy.org/trac/ticket/1636)

+   **[postgresql]**

    psycopg2 现在尊重语句选项“stream_results”。此选项将覆盖连接设置“server_side_cursors”。如果为 true，则语句将使用服务器端游标。如果为 false，则不会使用，即使连接上的“server_side_cursors”为 true。

    参考：[#1619](https://www.sqlalchemy.org/trac/ticket/1619)

### mysql

+   **[mysql]**

    新的 dialects：oursql，一个新的本地 dialect，MySQL Connector/Python，MySQLdb 的本地 Python 端口，当然还有 Jython 上的 zxjdbc。

+   **[mysql]**

    VARCHAR/NVARCHAR 如果没有长度将无法渲染，在传递给 MySQL 之前会引发错误。在 MySQL CAST 中不会影响 CAST，因为 MySQL CAST 中不允许 VARCHAR，dialect 在这种情况下渲染 CHAR/NCHAR。

+   **[mysql]**

    所有 _detect_XXX()函数现在在 dialect.initialize()下运行一次。

+   **[mysql]**

    对表/列名称中的%符号提供了更好的支持；当使用 executemany()时，MySQLdb 无法处理 SQL 中的%符号，而 SQLA 不希望为了处理那一个不存在的用例而增加开销。

    参考：[#1279](https://www.sqlalchemy.org/trac/ticket/1279)

+   **[mysql]**

    BINARY 和 MSBinary 类型现在在所有情况下生成“BINARY”。省略“length”参数将生成没有长度的“BINARY”。使用 BLOB 生成一个无长度的二进制列。

+   **[mysql]**

    MSEnum/ENUM 的“quoting='quoted'”参数已弃用。最好依赖于自动引用。

+   **[mysql]**

    ENUM 现在是新的通用 Enum 类型的子类，并且如果给定的标签名是 unicode 对象，则隐式处理 unicode 值。

+   **[mysql]**

    如果“nullable=False”未传递给 Column()，且没有默认值，则类型为 TIMESTAMP 的列现在默认为 NULL。这与所有其他类型一致，对于 TIMESTAMP，由于 MySQL 对 TIMESTAMP 列的默认可空性进行了“切换”，因此明确呈现“NULL”。

    参考：[#1539](https://www.sqlalchemy.org/trac/ticket/1539)

### sqlite

+   **[sqlite]**

    DATE、TIME 和 DATETIME 类型现在可以接受可选的 storage_format 和 regexp 参数。storage_format 可用于使用自定义字符串格式存储这些类型。regexp 允许使用自定义正则表达式匹配数据库中的字符串值。

+   **[sqlite]**

    Time 和 DateTime 类型现在默认使用更严格的正则表达式来匹配数据库中的字符串。如果使用存储在旧格式中的数据，请使用 regexp 参数。

+   **[sqlite]**

    SQLite Time 和 DateTime 类型上的 __legacy_microseconds__ 不再受支持。您应该使用 storage_format 参数。

+   **[sqlite]**

    Date、Time 和 DateTime 类型现在在接受绑定参数时更加严格：Date 类型仅接受日期对象（以及日期时间对象，因为它们继承自日期），Time 仅接受时间对象，DateTime 仅接受日期和日期时间对象。

+   **[sqlite]**

    Table()支持一个关键字参数“sqlite_autoincrement”，在生成 DDL 时将 SQLite 关键字“AUTOINCREMENT”应用于单个整数主键列。这将阻止生成单独的 PRIMARY KEY 约束。

    参考：[#1016](https://www.sqlalchemy.org/trac/ticket/1016)

### mssql

+   **[mssql]**

    MSSQL + Pyodbc + FreeTDS 现在大部分情况下可以正常工作，可能会有关于二进制数据以及 Unicode 模式标识符的异常情况。

+   **[mssql]**

    “has_window_funcs”标志已移除。LIMIT/OFFSET 使用将像以前一样使用 ROW NUMBER，如果在较旧版本的 SQL Server 上，则操作失败。行为完全相同，只是错误由 SQL Server 引发而不是方言，并且不需要设置标志来启用它。

+   **[mssql]**

    “auto_identity_insert”标志已移除。当 INSERT 语句覆盖已知具有序列的列时，此功能始终生效。与“has_window_funcs”一样，如果底层驱动程序不支持此功能，则无论如何都无法执行此操作，因此没有必��设置标志。

+   **[mssql]**

    使用新的 dialect.initialize()功能来设置版本相关的行为。

+   **[mssql]**

    删除了不再使用的 sequence 的引用。在 mssql 中，隐式标识与其他方言上的隐式序列相同。通过使用“default=Sequence()”启用显式序列。有关更多信息，请参阅 MSSQL 方言文档。

### oracle

+   **[oracle]**

    单元测试与 cx_oracle 完全通过！

+   **[oracle]**

    支持 cx_Oracle 的“本地 unicode”模式，不需要设置 NLS_LANG。使用最新的 cx_oracle 5.0.2 或更高版本。

+   **[oracle]**

    在基本类型中添加了 NCLOB 类型。

+   **[oracle]**

    use_ansi=False 不会泄漏到从子查询中选择的语句的 FROM/WHERE 子句中，该子查询还使用 JOIN/OUTERJOIN。

+   **[oracle]**

    在方言中添加了本地 INTERVAL 类型。目前仅支持 DAY TO SECOND 间隔类型，因为 cx_oracle 不支持 YEAR TO MONTH。

    参考：[#1467](https://www.sqlalchemy.org/trac/ticket/1467)

+   **[oracle]**

    使用 CHAR 类型会导致 cx_oracle 的 FIXED_CHAR dbapi 类型绑定到语句。

+   **[oracle]**

    Oracle 方言现在具有 NUMBER，旨在像 Oracle 的 NUMBER 类型一样运行。它是表反射返回的主要数值类型，并尝试根据精度/比例参数返回 Decimal()/float/int。

    参考：[#885](https://www.sqlalchemy.org/trac/ticket/885)

+   **[oracle]**

    func.char_length 是 LENGTH 的通用函数

+   **[Oracle]**

    ForeignKey() 包括 onupdate=<value> 将发出警告，不会发出不受 Oracle 支持的 ON UPDATE CASCADE。

+   **[Oracle]**

    RowProxy() 的 keys() 方法现在返回结果列名*规范化*为 SQLAlchemy 不区分大小写的名称。这意味着对于不区分大小写的名称，它们将是小写的，而 DBAPI 通常会将它们返回为大写名称。这使得行键() 可与进一步的 SQLAlchemy 操作兼容。

+   **[Oracle]**

    使用新的 dialect.initialize() 功能来设置版本相关的行为。

+   **[Oracle]**

    在 Oracle 中使用 types.BigInteger 将生成 NUMBER(19)

    参考：[#1125](https://www.sqlalchemy.org/trac/ticket/1125)

+   **[Oracle]**

    “区分大小写”功能将在反射期间检测到全小写的区分大小写列名，并向生成的 Column 添加“quote=True”，以便保持适当的引用。

### 杂项

+   **[主要] [发布]**

    要查看完整的功能描述集，请参阅 [`docs.sqlalchemy.org/en/latest/changelog/migration_06.html`](https://docs.sqlalchemy.org/en/latest/changelog/migration_06.html)。此文档正在进行中。

+   **[主要] [发布]**

    所有最新版本 0.5 版本及以下的错误修复和功能增强也包含在 0.6 版本中。

+   **[主要] [发布]**

    现在针对的平台包括 Python 2.4/2.5/2.6，Python 3.1，Jython2.5。

+   **[引擎]**

    可以使用 create_engine(… isolation_level=”…”) 指定事务隔离级别；在 postgresql 和 sqlite 上可用。

    参考：[#443](https://www.sqlalchemy.org/trac/ticket/443)

+   **[引擎]**

    Connection 现在具有 execution_options()，接受影响语句在 DBAPI 方面执行方式的关键字的生成方法。目前支持“stream_results”，导致 psycopg2 使用服务器端游标执行该语句，以及“autocommit”，这是 select() 和 text() 中“autocommit”选项的新位置。select() 和 text() 也有 .execution_options() 以及 ORM Query()。

+   **[引擎]**

    修复了基于 entrypoint 的方言导入，不再依赖于愚蠢的 tb_info 技巧来确定导入错误状态。

    参考：[#1630](https://www.sqlalchemy.org/trac/ticket/1630)

+   **[引擎]**

    向 ResultProxy 添加了 first() 方法，返回第一行并立即关闭结果集。

+   **[引擎]**

    RowProxy 对象现在是可 pickle 的，即 result.fetchone()，result.fetchall() 等返回的对象。

+   **[引擎]**

    RowProxy 不再具有 close() 方法，因为行不再保留对父级的引用。而是在父级 ResultProxy 上调用 close()，或者使用 autoclose。

+   **[引擎]**

    ResultProxy 内部已进行了大幅改进，以大大减少在获取列时的方法调用次数。在获取大型结果集时，可以提供大幅度的速度提升（高达 100%以上）。当获取没有应用类型级处理的列并且使用结果作为元组（而不是字典）时，改进效果更大。非常感谢 Elixir 的 Gaëtan de Menten 为这一巨大的改进！

    参考：[#1586](https://www.sqlalchemy.org/trac/ticket/1586)

+   **[引擎]**

    依赖于“最后插入的 id”进行后获取生成的序列值的数据库（例如 MySQL，MS-SQL）现在在表中“自增”列不是第一个主键列时可以正常工作。

+   **[引擎]**

    last_inserted_ids()方法已重命名为描述符“inserted_primary_key”。

+   **[引擎]**

    在 create_engine()上设置 echo=False 现在将日志级别设置为 WARN 而不是 NOTSET。这样，即使整体启用了“sqlalchemy.engine”的日志记录，也可以禁用特定引擎的日志记录。请注意，“echo”的默认设置为 None。

    参考：[#1554](https://www.sqlalchemy.org/trac/ticket/1554)

+   **[引擎]**

    ConnectionProxy 现在具有所有事务生命周期事件的包装方法，包括 begin()，rollback()，commit()，begin_nested()，begin_prepared()，prepare()，release_savepoint()等。

+   **[引擎]**

    连接池日志现在使用 INFO 和 DEBUG 日志级别进行记录。INFO 用于主要事件，如无效的连接，DEBUG 用于所有获取/返回日志记录。echo_pool 可以为 False，None，True 或“debug”，与 echo 的工作方式相同。

+   **[引擎]**

    所有 pyodbc 方言现在支持额外的 pyodbc 特定的 kw 参数‘ansi’，‘unicode_results’，‘autocommit’。

    参考：[#1621](https://www.sqlalchemy.org/trac/ticket/1621)

+   **[引擎]**

    “threadlocal”引擎已重写和简化，现在支持 SAVEPOINT 操作。

+   **[引擎]**

    已弃用或移除

    +   result.last_inserted_ids()已弃用。请使用 result.inserted_primary_key

    +   dialect.get_default_schema_name(connection)现在通过 dialect.default_schema_name 公开。

    +   从 engine.transaction()和 engine.run_callable()中删除了“connection”参数 - Connection 本身现在具有这些方法。所有四个方法都接受*args 和**kwargs，这些参数将传递给给定的可调用对象，以及操作连接。

+   **[反射/检查]**

    表反射已扩展和泛化为一个名为“sqlalchemy.engine.reflection.Inspector”的新 API。Inspector 对象提供关于各种模式信息的细粒度信息，包括表名，列名，视图定义，序列，索引等，还有扩展的空间。

+   **[反射/检查]**

    视图现在可以反射为普通的 Table 对象。使用相同的 Table 构造函数，但要注意“有效”的主键和外键约束不是反射结果的一部分；如果需要，必须显式指定这些。

+   **[reflection/inspection]**

    现有的 autoload=True 系统现在在底层使用 Inspector，以便每个方言只需返回关于表和其他对象的“原始”数据 - Inspector 是将信息编译成 Table 对象的唯一位置，以确保一致性最大化。

+   **[ddl]**

    DDL 系统已大大扩展。DDL()类现在扩展了更通用的 DDLElement()，这形成了许多新构造的基础：

    > > +   CreateTable()
    > > +   
    > > +   DropTable()
    > > +   
    > > +   AddConstraint()
    > > +   
    > > +   DropConstraint()
    > > +   
    > > +   CreateIndex()
    > > +   
    > > +   DropIndex()
    > > +   
    > > +   CreateSequence()
    > > +   
    > > +   DropSequence()
    > > +   
    > 这些支持“on”和“execute-at()”，就像普通的 DDL()一样。用户定义的 DDLElement 子类可以被创建并链接到一个编译器，使用 sqlalchemy.ext.compiler 扩展。

+   **[ddl]**

    传递给 DDL()和 DDLElement()的“on”可调用的签名如下所示：

    > ddl
    > 
    > DDLElement 对象本身
    > 
    > 事件
    > 
    > 字符串事件名称。
    > 
    > 目标
    > 
    > 以前是“schema_item”，触发事件的 Table 或 MetaData 对象。
    > 
    > 连接
    > 
    > 用于操作的 Connection 对象。
    > 
    > **kw
    > 
    > 关键字参数。在 MetaData 之前/之后创建/删除的情况下，要发出 CREATE/DROP DDL 的 Table 对象列表作为 kw 参数“tables”传递。这对于依赖于特定表存在的元数据级 DDL 是必要的。

    DDL 的“schema_item”属性已重命名为

    “target”。

+   **[dialect] [refactor]**

    方言模块现在分为数据库方言和 DBAPI 实现。现在更倾向于使用 dialect+driver://…来指定连接 URL，即“mysql+mysqldb://scott:tiger@localhost/test”。请参阅 0.6 文档以获取示例。

+   **[dialect] [refactor]**

    外部方言的 setuptools 入口现在称为“sqlalchemy.dialects”。

+   **[dialect] [refactor]**

    从 Table 中删除了“owner”关键字参数。使用“schema”表示要预先添加到表名的任何命名空间。

+   **[dialect] [refactor]**

    server_version_info 变成了一个静态属性。

+   **[dialect] [refactor]**

    方言在初始连接时接收 initialize()事件以确定连接属性。

+   **[dialect] [refactor]**

    方言在接收 visit_pool 事件时有机会建立池监听器。

+   **[dialect] [refactor]**

    缓存的 TypeEngine 类现在按照每个方言类而不是每个方言进行缓存���

+   **[dialect] [refactor]**

    新的 UserDefinedType 应该作为新类型的基类使用，它保留了 get_col_spec()的 0.5 行为。

+   **[dialect] [refactor]**

    所有类型类的 result_processor()方法现在接受第二个参数“coltype”，这是来自 cursor.description 的 DBAPI 类型参数。这个参数可以帮助一些类型决定对结果值进行最有效的处理。

+   **[dialect] [重构]**

    已弃用的 Dialect.get_params()已被移除。

+   **[dialect] [重构]**

    Dialect.get_rowcount()已重命名为描述符“rowcount”，并直接调用 cursor.rowcount。需要为某些调用硬编码行数的方言应该重写该方法以提供不同的行为。

+   **[dialect] [重构]**

    DefaultRunner 和其子类已被移除。该对象的工作已被简化并移至 ExecutionContext 中。支持序列的方言应该在其执行上下文实现中添加一个 fire_sequence()方法。

    参考：[#1566](https://www.sqlalchemy.org/trac/ticket/1566)

+   **[dialect] [重构]**

    编译器生成的函数和运算符现在使用（几乎）常规的分发函数形式“visit_<opname>”和“visit_<funcname>_fn”来提供定制处理。这取代了在编译器子类中复制“functions”和“operators”字典的需要，改为使用直接的访问方法，并且还允许编译器子类完全控制渲染，因为完整的 _Function 或 _BinaryExpression 对象被传递进来。

+   **[firebird]**

    RowProxy()的 keys()方法现在返回结果列名*规范化*为 SQLAlchemy 大小写不敏感名称。这意味着对于大小写不敏感的名称，它们将以小写形式返回，而 DBAPI 通常会将它们返回为大写名称。这使得行键()与进一步的 SQLAlchemy 操作兼容。

+   **[firebird]**

    使用新的 dialect.initialize()功能来设置版本相关的行为。

+   **[firebird]**

    “大小写敏感”功能将在反射期间检测到全小写的大小写敏感列名，并向生成的列添加“quote=True”，以便保持适当的引用。

+   **[types]**

    方言中类型的构建已完全改写。方言现在仅以大写名称定义公开可用的类型，并使用下划线标识符（即私有）定义内部实现类型。用于在 SQL 和 DDL 中表达类型的系统已移至编译器系统。这样做的效果是在大多数方言中减少了许多类型对象。有关方言作者的详细文档在 lib/sqlalchemy/dialects/type_migration_guidelines.txt 中。

+   **[types]**

    类型不再对默认参数进行任何猜测。特别是，Numeric、Float、NUMERIC、FLOAT、DECIMAL 不会生成任何长度或精度，除非指定。

+   **[types]**

    types.Binary 已重命名为 types.LargeBinary，它仅生成 BLOB、BYTEA 或类似的“长二进制”类型。新增了基本的 BINARY 和 VARBINARY 类型，以以一种与 MySQL/MS-SQL 特定类型无关的方式访问这些类型。

    参考：[#1664](https://www.sqlalchemy.org/trac/ticket/1664)

+   **[types]**

    字符串/文本/Unicode 类型现在在每个结果列值上跳过 unicode() 检查，如果方言检测到 DBAPI 本地返回 Python unicode 对象。这个检查是在第一次连接时使用“SELECT CAST ‘some text’ AS VARCHAR(10)”或等效方式发出的，然后检查返回的对象是否是 Python unicode。这允许本地支持 unicode 的 DBAPI（包括 pysqlite/sqlite3、psycopg2 和 pg8000）获得巨大的性能提升。

+   **[类型]**

    大多数类型结果处理器已经检查可能的速度改进。具体来说，以下通用类型已经经过优化，导致不同程度的速度提升：Unicode、PickleType、Interval、TypeDecorator、Binary。此外，以下特定于 dbapi 的实现已经得到改进：Sqlite 上的 Time、Date 和 DateTime，PostgreSQL 上的 ARRAY，MySQL 上的 Time，MySQL 上的 Numeric（as_decimal=False），oursql 和 pypostgresql，cx_oracle 上的 DateTime，以及 cx_oracle 上的 LOB 类型。

+   **[类型]**

    现在类型的反射将返回 types.py 中的确切大写类型，或者如果类型不是标准 SQL 类型，则返回方言本身中的大写类型。这意味着反射现在返回有关反射类型的更准确信息。

+   **[类型]**

    添加了一个新的 Enum 通用类型。Enum 是一个 schema-aware 对象，用于支持需要特定 DDL 才能使用 enum 或等效的数据库；在 PG 的情况下，它处理 CREATE TYPE 的详细信息，在其他没有本地 enum 支持的数据库上，将生成 VARCHAR + 内联 CHECK 约束以强制执行 enum。

    参考：[#1109](https://www.sqlalchemy.org/trac/ticket/1109)，[#1511](https://www.sqlalchemy.org/trac/ticket/1511)

+   **[类型]**

    Interval 类型包括一个“native”标志，控制是否选择本地 INTERVAL 类型（postgresql + oracle）（如果可用）或不选择。还添加了“day_precision”和“second_precision”参数，适当地传播到这些本地类型。相关于。

    参考：[#1467](https://www.sqlalchemy.org/trac/ticket/1467)

+   **[类型]**

    当在没有本地布尔支持的后端使用布尔类型时，将生成一个 CHECK 约束“col IN (0, 1)”以及基于 int/smallint 的列类型。如果需要，可以通过 create_constraint=False 关闭此功能。请注意，MySQL 没有本地布尔或 CHECK 约束支持，因此该功能在该平台上不可用。

    参考：[#1589](https://www.sqlalchemy.org/trac/ticket/1589)

+   **[类型]**

    PickleType 现在在 mutable=True 时使用 == 进行值比较，除非为类型指定了带有比较函数的“comparator”参数。如果未重写 __eq__() 或未提供比较函数，则将基于标识比较被 pickled 的对象（这会破坏 mutable=True 的目的）。

+   **[类型]**

    Numeric 和 Float 的默认“precision”和“scale”参数已移除，现在默认为 None。除非提供这些值，否则 NUMERIC 和 FLOAT 将默认不带任何数字参数。

+   **[类型]**

    AbstractType.get_search_list() 已移除 - 不再需要使用它的游戏。

+   **[类型]**

    添加了一个通用的 BigInteger 类型，编译为 BIGINT 或 NUMBER(19)。

    参考：[#1125](https://www.sqlalchemy.org/trac/ticket/1125)

+   **[类型]**

    sqlsoup 已进行了全面改进，明确支持 0.5 风格的会话，使用 autocommit=False、autoflush=True。SQLSoup 的默认行为现在需要通常的 commit() 和 rollback() 的使用，这些方法已添加到其接口中。可以传递一个显式的 Session 或 scoped_session 给构造函数，允许覆盖这些参数。

+   **[类型]**

    sqlsoup db.<sometable>.update() 和 delete() 现在分别调用 query(cls).update() 和 delete()。

+   **[类型]**

    sqlsoup 现在具有 execute() 和 connection() 方法，调用这些名称的 Session 方法，确保绑定是基于 SqlSoup 对象的绑定。

+   **[类型]**

    sqlsoup 对象不再具有‘query’属性 - 对于 sqlsoup 的使用范式而言，这并不需要，而且会妨碍一个实际命名为‘query’的列。

+   **[类型]**

    传递给 association_proxy 的 proxy_factory 可调用对象的签名现在为 (lazy_collection, creator, value_attr, association_proxy)，添加了第四个参数，即父级 AssociationProxy 参数。允许内置集合的序列化和子类化。

    参考：[#1259](https://www.sqlalchemy.org/trac/ticket/1259)

+   **[类型]**

    association_proxy 现在具有基本的比较方法 .any()、.has()、.contains()、==、!=，感谢 Scott Torborg。

    参考：[#1372](https://www.sqlalchemy.org/trac/ticket/1372)
