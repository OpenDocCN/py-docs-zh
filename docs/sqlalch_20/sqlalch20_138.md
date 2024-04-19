# 0.3 变更日志

> 原文：[`docs.sqlalchemy.org/en/20/changelog/changelog_03.html`](https://docs.sqlalchemy.org/en/20/changelog/changelog_03.html)

## 0.3.11

发布日期：2007 年 10 月 14 日星期日

### orm

+   **[orm]**

    添加了一个检查，用 join()从 A->B 连接时，沿着两个不同的 m2m 表。这在 0.3 中会引发错误，但在 0.4 中使用别名时是可能的。

    参考：[#687](https://www.sqlalchemy.org/trac/ticket/687)

+   **[orm]**

    修复了 Session.merge()中的小异常抛出 bug

+   **[orm]**

    修复了映射器的 bug，当链接到一个表没有主键列时，不会检测到连接的表没有主键。

+   **[orm]**

    修复了从自定义继承条件确定正确同步子句的错误

    参考：[#769](https://www.sqlalchemy.org/trac/ticket/769)

+   **[orm]**

    如果另一侧集合不包含项目，则 backref 删除对象操作不会失败，支持 noload 集合

    参考：[#813](https://www.sqlalchemy.org/trac/ticket/813)

### engine

+   **[engine]**

    修复了在使用带有 threadlocal 设置的池时可能发生的另一个偶发竞争条件

### sql

+   **[sql]**

    调整了像 func.count(t.c.col.distinct())这样的子句的 DISTINCT 优先级

+   **[sql]**

    修复了在:bind$params 中检测内部‘$’字符的 bug

    参考：[#719](https://www.sqlalchemy.org/trac/ticket/719)

+   **[sql]**

    不要假设连接条件仅由列对象组成

    参考：[#768](https://www.sqlalchemy.org/trac/ticket/768)

+   **[sql]**

    调整了 NOT 的运算符优先级，以匹配‘==’和其他运算符，因此~(x==y)产生 NOT (x=y)，这与 MySQL < 5.0 兼容（不喜欢“NOT x=y”）

    参考：[#764](https://www.sqlalchemy.org/trac/ticket/764)

### mysql

+   **[mysql]**

    修复了生成模式时 YEAR 列的规范

### sqlite

+   **[sqlite]**

    传递字符串化日期

### mssql

+   **[mssql]**

    增加了对 TIME 列的支持（使用 DATETIME 模拟）

    参考：[#679](https://www.sqlalchemy.org/trac/ticket/679)

+   **[mssql]**

    增加对 BIGINT、MONEY、SMALLMONEY、UNIQUEIDENTIFIER 和 SQL_VARIANT 的支持

    参考：[#721](https://www.sqlalchemy.org/trac/ticket/721)

+   **[mssql]**

    从反射表中删除索引时现在会对索引名称进行引用

    参考：[#684](https://www.sqlalchemy.org/trac/ticket/684)

+   **[mssql]**

    现在可以为 PyODBC 指定 DSN，使用类似 mssql:///?dsn=bob 的 URI

### oracle

+   **[oracle]**

    从“binary”类型中移除了 LONG_STRING、LONG_BINARY，因此类型对象不会尝试将其值读取为 LOB。

    参考：[#622](https://www.sqlalchemy.org/trac/ticket/622)，[#751](https://www.sqlalchemy.org/trac/ticket/751)

### misc

+   **[postgres]**

    从替代模式反射表时，放在主键上的“default”（通常是序列名称）的“schema”名称无条件引用，因此需要引用的模式名称是可以的。对于不需要引用的模式名称来说，这有点多余但不会有害。

+   **[firebird]**

    由于票号#370（正确方式），supports_sane_rowcount()设置为 False。

+   **[firebird]**

    修复了 Column 的 nullable 属性的反射。

## 0.3.10

发布日期：2007 年 7 月 20 日

### 通用

+   **[general]**

    在 0.3.9 中添加的新互斥体导致在竞争条件下 pool_timeout 功能失败；如果许多线程同时将池推入溢出，线程将立即引发 TimeoutError，没有延迟。此问题已得到解决。

### orm

+   **[orm]**

    清理连接绑定的会话、SessionTransaction

### sql

+   **[sql]**

    使连接绑定的元数据与隐式执行一起工作

+   **[sql]**

    外键规范可以在其标识符中包含任何字符

    参考：[#667](https://www.sqlalchemy.org/trac/ticket/667)

+   **[sql]**

    对二进制子句之间的可交换性进行了改进，改善了 ORM 的延迟加载优化

    参考：[#664](https://www.sqlalchemy.org/trac/ticket/664)

### 杂项

+   **[postgres]**

    修复了最大标识符长度（63）

    参考：[#571](https://www.sqlalchemy.org/trac/ticket/571)

## 0.3.9

发布日期：2007 年 7 月 15 日

### 通用

+   **[general]**

    为 NoSuchColumnError 添加了更好的错误消息

    参考：[#607](https://www.sqlalchemy.org/trac/ticket/607)

+   **[general]**

    最终弄清楚了如何将 setuptools 版本引入，可作为 sqlalchemy.__version__ 使用

    参考：[#428](https://www.sqlalchemy.org/trac/ticket/428)

+   **[general]**

    各种“engine”参数，如“engine”、“connectable”、“engine_or_url”、“bind_to”等都存在，但已被弃用。它们都被单个术语“bind”替代。您还可以使用 metadata.bind = <engine or connection>来设置 MetaData 的“bind”。

### orm

+   **[orm]**

    与 0.4 的向前兼容性：向 Query 添加了 one()、first()和 all()。几乎所有 0.4 的 Query 功能在 0.3.9 中都存在，以实现向前兼容。

+   **[orm]**

    reset_joinpoint()这次真的有效了，保证！允许您从根重新加入：query.join(['a', 'b']).filter(<crit>).reset_joinpoint().join(['a', 'c']).filter(<some other crit>).all()在 0.4 中，所有 join()调用都从“root”开始

+   **[orm]**

    在 mapper()构造步骤中添加了同步，以避免在不同线程中编译预先存在的映射器时发生线程冲突

    参考：[#613](https://www.sqlalchemy.org/trac/ticket/613)

+   **[orm]**

    当 Mapper 将两个同名主键列混合为单个属性时，会发出警告。这在映射到连接（或继承）时经常发生。

+   **[orm]**

    synonym()属性完全受到所有 Query joining/ with_parent 操作的支持

    参考：[#598](https://www.sqlalchemy.org/trac/ticket/598)

+   **[orm]**

    修复了使用 many-to-many uselist=False 关系删除项目时的非常愚蠢的错误

+   **[orm]**

    还记得关于多态联合的那些东西吗？用于连接表继承？有趣的是...对于连接表继承，你实际上不需要它，你可以通过 outerjoin()将所有表串在一起。但如果涉及具体表，则 UNION 仍然适用（因为没有东西可以将它们连接起来）。

+   **[orm]**

    对急切加载进行了一些修正，以更好地与使用直接“outerjoin”子句的多态映射的急切加载配合使用

### sql

+   **[sql]**

    到默认模式不是默认模式的表的`ForeignKey`需要明确指定模式；即`ForeignKey('alt_schema.users.id')`。

+   **[sql]**

    `MetaData`现在可以使用引擎或 URL 作为第一个参数构造，就像`BoundMetaData`一样

+   **[sql]**

    `BoundMetaData`现已弃用，`MetaData`是直接的替代品。

+   **[sql]**

    `DynamicMetaData`已更名为`ThreadLocalMetaData`。`DynamicMetaData`名称已被弃用，并且是`ThreadLocalMetaData`的别名，或者如果`threadlocal=False`，则是常规`MetaData`。

+   **[sql]**

    表示复合主键的非键集，以允许由相同名称的列组成的复合键；出现在 Join 中。有助于继承方案制定正确的 PK。

+   **[sql]**

    改进了从联接中获取“正确”的和最小的主键列集的能力，等同于外键和其他等价列。这主要是为了帮助继承方案制定最佳选择的主键列。

    参考：[#185](https://www.sqlalchemy.org/trac/ticket/185)

+   **[sql]**

    添加了‘bind’参数到`Sequence.create()/drop()`，`ColumnDefault.execute()`

+   **[sql]**

    可以使用与列名不同的“key”属性在反射表中重写列，包括主键列。

    参考：[#650](https://www.sqlalchemy.org/trac/ticket/650)

+   **[sql]**

    修复了“模糊列”结果检测，在结果中存在重复列名时

    参考：[#657](https://www.sqlalchemy.org/trac/ticket/657)

+   **[sql]**

    对“column targeting”进行了一些增强，即将列与另一个可选择的“对应”列匹配的能力。这主要影响 ORM 映射到复杂连接的能力。

+   **[sql]**

    `MetaData`和所有`SchemaItems`可安全使用`pickle`。可以将缓慢的表反射转储到`pickle`文件中以供以后重用。只需在`unpickling`后重新连接到元数据的引擎。

    参考：[#619](https://www.sqlalchemy.org/trac/ticket/619)

+   **[sql]**

    向`QueuePool`的“overflow”计算添加了互斥体，以防止绕过`max_overflow`的竞争条件

+   **[sql]**

    修复了复合选择的分组以给出正确的结果。在某些情况下会在 sqlite 上中断，但是那些情况无论如何都会产生不正确的结果，sqlite 不支持分组的复合选择。

    参考：[#623](https://www.sqlalchemy.org/trac/ticket/623)

+   **[sql]**

    修正了运算符的优先级，以正确应用括号。

    参考：[#620](https://www.sqlalchemy.org/trac/ticket/620)

+   **[sql]**

    调用`<column>.in_()`（即不带参数）将返回“CASE WHEN (<column> IS NULL) THEN NULL ELSE 0 END = 1)” ，以便在所有情况下返回 NULL 或 False，而不是抛出错误。

    参考：[#545](https://www.sqlalchemy.org/trac/ticket/545)

+   **[sql]**

    修复了 select() 的“where”/“from”条件，除了常规字符串外还接受 Unicode 字符串 - 两者都转换为 text()

+   **[sql]**

    添加了独立的 distinct() 函数，除了 column.distinct()

    参考：[#558](https://www.sqlalchemy.org/trac/ticket/558)

+   **[sql]**

    result.last_inserted_ids() 应返回与表的主键约束大小相同的列表。通过“被动”创建且不通过 cursor.lastrowid 可用的值将为 None。

+   **[sql]**

    修复了长标识符检测，使用 > 而不是 >= 作为最大标识符长度

    参考：[#589](https://www.sqlalchemy.org/trac/ticket/589)

+   **[sql]**

    修复了 bug，当 selectable 是表和涉及相同表的另一���连接的连接时，selectable.corresponding_column(selectable.c.col) 不会返回 selectable.c.col。搞乱了 ORM 决策

    参考：[#593](https://www.sqlalchemy.org/trac/ticket/593)

+   **[sql]**

    在 types.py 中添加了 Interval 类型

    参考：[#595](https://www.sqlalchemy.org/trac/ticket/595)

### mysql

+   **[mysql]**

    修复了一些错误的捕获，这些错误暗示连接已断开

    参考：[#625](https://www.sqlalchemy.org/trac/ticket/625)

+   **[mysql]**

    修复了模运算符的转义

    参考：[#624](https://www.sqlalchemy.org/trac/ticket/624)

+   **[mysql]**

    添加了 'fields' 作为保留字

    参考：[#590](https://www.sqlalchemy.org/trac/ticket/590)

+   **[mysql]**

    各种反射增强/修复

### sqlite

+   **[sqlite]**

    重新排列了方言初始化，以便及时警告 pysqlite1 过于陈旧。

+   **[sqlite]**

    sqlite 更好地处理混合使用各种 Date/Time/DateTime 列的日期时间/日期/时间对象

+   **[sqlite]**

    字符串 PK 列插入不会被 OID 覆盖

    参考：[#603](https://www.sqlalchemy.org/trac/ticket/603)

### mssql

+   **[mssql]**

    修复了 pyodbc 的端口选项处理

    参考：[#634](https://www.sqlalchemy.org/trac/ticket/634)

+   **[mssql]**

    现在能够反射标识列的起始和增量值

+   **[mssql]**

    初步支持使用 scope_identity() 与 pyodbc

### oracle

+   **[oracle]**

    日期时间修复：使亚秒级 TIMESTAMP 正常工作，添加支持仅具有年/月/日的 types.Date 的 OracleDate

    参考：[#604](https://www.sqlalchemy.org/trac/ticket/604)

+   **[oracle]**

    添加了方言标志“auto_convert_lobs”，默认为 True；将强制将结果集中检测到的任何 LOB 对象转换为 OracleBinary，以便自动读取 LOB，如果没有 typemap 存在（即，如果发出了文本执行()）。

+   **[oracle]**

    mod 运算符‘%’ 会产生 MOD

    参考：[#624](https://www.sqlalchemy.org/trac/ticket/624)

+   **[oracle]**

    在使用 Python 2.3 时，将 cx_oracle datetime 对象转换为 Python datetime.datetime

    参考：[#542](https://www.sqlalchemy.org/trac/ticket/542)

+   **[oracle]**

    修复了 Oracle TEXT 类型中的 Unicode 转换

### 杂项

+   **[ext]**

    现在对 dict association proxies 进行迭代时类似于 dict，而不是类似于 InstrumentedList（例如，通过键而不是值）

+   **[ext]**

    关联代理不再紧密绑定到源集合，并且使用一个 thunk 构造

    参考：[#597](https://www.sqlalchemy.org/trac/ticket/597)

+   **[ext]**

    添加了 selectone_by()到 assignmapper

+   **[postgres]**

    修复了百分比运算符的转义

    参考：[#624](https://www.sqlalchemy.org/trac/ticket/624)

+   **[postgres]**

    增加了对域反射的支持

    参考：[#570](https://www.sqlalchemy.org/trac/ticket/570)

+   **[postgres]**

    在反射期间缺少的类型解析为 Null 类型而不是引发错误

+   **[postgres]**

    上述“schema”中的修复修复了从一个 alt-schema 表反射到一个 public schema 表的外键的问题

## 0.3.8

发布日期：2007 年 6 月 2 日 星期六

### orm

+   **[orm]**

    添加了 reset_joinpoint()方法到 Query，将“连接点”移回起始映射。0.4 版本将更改 join()的行为以在所有情况下重置“连接点”，因此这是一个临时方法。为了向前兼容，请确保跨多个关系指定连接使用单个 join()，即 join([‘a’, ‘b’, ‘c’])。

+   **[orm]**

    修复了 query.instances()中的 bug，该 bug 无法处理多个额外的映射或一个额外的列。

+   **[orm]**

    “delete-orphan”不再暗示“delete”。持续努力分离这两个操作的行为。

+   **[orm]**

    多对多关系正确设置了关联表上删除操作的绑定参数类型

+   **[orm]**

    多对多关系检查通过删除操作从关联表中删除的行数是否与预期结果匹配

+   **[orm]**

    session.get()和 session.load()将**kwargs 传播到查询

+   **[orm]**

    修复了允许原始多态联合嵌入到相关子查询中的多态查询

    参考：[#577](https://www.sqlalchemy.org/trac/ticket/577)

+   **[orm]**

    修复了 select_by(<propname>=<object instance>)风格的连接与多对多关系结合时的 bug，该 bug 是在 r2556 中引入的

+   **[orm]**

    mapper()的“primary_key”参数传播到“polymorphic”映射。此列表中的主键列被规范化为映射的本地表的主键。

+   **[orm]**

    恢复了在 sa.orm.strategies 记录器下记录“延迟加载子句”的日志，该日志在 0.3.7 中被删除

+   **[orm]**

    改进了对映射到 select()语句的属性进行急加载的支持；即，急加载器更擅长定位正确的可选择项以附加其 LEFT OUTER JOIN。

### sql

+   **[sql]**

    _Label 类覆盖 compare_self 以返回其最终对象。也就是说，如果你说 someexpr.label(‘foo’) == 5，它会产生正确的“someexpr == 5”。

+   **[sql]**

    _Label 传播“_hide_froms()”，使得标量选择在 FROM 子句方面表现更正常 #574

+   **[sql]**

    修复了在使用 oid_column 作为排序依据时生成长名称的 bug（在映射查询中大量使用 oids）

+   **[sql]**

    ResultProxy 的速度显著提高，预先缓存 TypeEngine 方言实现，并节省每列的函数调用

+   **[sql]**

    通过新的 _Grouping 构造将括号应用于子句。使用运算符优先级更智能地将括号应用于子句，提供更清晰的子句嵌套（不会改变放置在其他子句中的子句，即没有“parens”标志）

+   **[sql]**

    添加了‘modifier’关键字，类似于 func.<foo>，但不添加括号。例如 select([modifier.DISTINCT(…)]) 等。

+   **[sql]**

    移除了“在 UNION 的一部分的 SELECT 中不能有 GROUP BY”的限制

    参考：[#578](https://www.sqlalchemy.org/trac/ticket/578)

### mysql

+   **[mysql]**

    现在几乎支持所有 MySQL 列类型的声明和反射。添加了 NCHAR、NVARCHAR、VARBINARY、TINYBLOB、LONGBLOB、YEAR

+   **[mysql]**

    sqltypes.Binary 透传现在始终构建一个 BLOB，避免与非常旧的数据库版本的问题

+   **[mysql]**

    支持列级 CHARACTER SET 和 COLLATE 声明，以及 ASCII、UNICODE、NATIONAL 和 BINARY 简写。

### 杂项

+   **[engines]**

    添加了 detach()到 Connection，允许底层的 DBAPI 连接从其池中分离，在取消引用/关闭()时关闭，而不是被池重用。

+   **[engines]**

    添加了 invalidate()到 Connection，立即使 Connection 及其底层的 DBAPI 连接无效。

+   **[firebird]**

    将最大标识符长度设置为 31

+   **[firebird]**

    由于票号#370，supports_sane_rowcount()设置为 False。versioned_id_col 功能在 FB 中不起作用。

+   **[firebird]**

    一些执行修复

+   **[firebird]**

    新的 association proxy 实现，实现对基于列表、字典和集合的关系集合的完整代理

+   **[firebird]**

    添加了 orderinglist，一个自定义列表类，将对象属性与列表中对象的位置同步。

+   **[firebird]**

    对 SelectResultsExt 进行了小修复，以避免在 select()期间绕过自身。

+   **[firebird]**

    添加了 filter()、filter_by()到 assignmapper

## 0.3.7

发布日期：2007 年 4 月 29 日

### orm

+   **[orm]**

    修复了一个关键问题，即在使用 options(eagerload())后，映射器将始终对所有后续 LIMIT/OFFSET/DISTINCT 查询应用查询“包装”行为，即使在这些后续查询中没有应用急加载。

+   **[orm]**

    添加了 query.with_parent(someinstance)方法。使用父实例的延迟连接条件搜索目标实例。可选字符串“property”用于隔离所需关系。还添加了静态 Query.query_from_parent(instance, property)版本。

    参考：[#541](https://www.sqlalchemy.org/trac/ticket/541)

+   **[orm]**

    改进了 query.XXX_by(someprop=someinstance)查询，使用与 with_parent 类似的方法，即使用“lazy”子句，防止将远程实例的表添加到 SQL 中，从而使更复杂的条件成为可能

    参考：[#554](https://www.sqlalchemy.org/trac/ticket/554)

+   **[orm]**

    在查询中添加了聚合函数的生成版本，如 sum()、avg()等。通过 query.apply_max()、query.apply_sum()等使用。#552

+   **[orm]**

    修复了在 join()等操作中与 distinct()或 distinct=True 组合使用时的问题。

+   **[orm]**

    对应于标签/bindparam 名称生成，急切加载器使用 md5 哈希为它们创建的别名生成确定性名称。

+   **[orm]**

    当将“set”/“sets.Set”类或子类传递给自定义集合类时，改进/修复了其行为（在惰性加载期间仍在寻找 append()方法）

+   **[orm]**

    恢复了旧的“column_property()”ORM 函数（曾称为“column()”），以强制将任何列表达式添加为映射器的属性，特别是那些在映射的可选择部分中不存在的列表达式。这允许将任何类型的“标量表达式”添加为关系（尽管它们在急切加载方面存在问题）。

+   **[orm]**

    修复了针对多对多关系的目标为多态映射器的问题

    参考：[#533](https://www.sqlalchemy.org/trac/ticket/533)

+   **[orm]**

    在 session.merge()的使用以及与 entity_name 的结合方面取得了进展

    参考：[#543](https://www.sqlalchemy.org/trac/ticket/543)

+   **[orm]**

    在继承映射器之间的关系上进行了通常的调整，本例中建立了与子类映射器的关系，其中连接条件来自超类的表

### sql

+   **[sql]**

    结果集列的 keys()不会小写化，会以与它们在 cursor.description 中的表达方式完全相同的方式返回。请注意，这会导致在 oracle 中 colnames 全部大写。

+   **[sql]**

    为支持 Unicode 表名、列名和 SQL 语句添加了初步支持，适用于可以支持它们的数据库。到目前为止，sqlite 和 postgres 可以正常工作。MySQL *大部分* 可以工作，除了 has_table()函数无法工作。反射也可以工作。

+   **[sql]**

    Unicode 类型现在是 String 的直接子类，其中包含了所有的“convert_unicode”逻辑。这有助于更好地处理像 MS-SQL 这样的数据库中出现的各种 unicode 情况，并允许对 Unicode 数据类型进行子类化。

    参考：[#522](https://www.sqlalchemy.org/trac/ticket/522)

+   **[sql]**

    ClauseElements 现在可以在 in_()子句中使用，例如绑定参数等。#476

+   **[sql]**

    为 CompareMixin 元素实现了反向运算符，允许像“5 + somecolumn”等表达式。#474

+   **[sql]**

    在 update()和 delete()的“where”条件中，现在将嵌套的 select()语句与正在更新或删除的表进行关联。这与嵌套的 select()语句关联方式相同，并且可以通过嵌套 select()的 correlate=False 标志来禁用。

+   **[sql]**

    列标签现在在编译阶段生成，这意味着它们的长度取决于方言。因此，在 oracle 上被截断为 30 个字符的标签将在 postgres 上扩展为 63 个字符。此外，真实的标签名称始终附加为父可选择部分的访问器，因此无需关注“被截断”的标签名称。

    参考：[#512](https://www.sqlalchemy.org/trac/ticket/512)

+   **[sql]**

    列标签和绑定参数“截断”现在也生成确定性名称，基于它们在编译的完整语句中的顺序。这意味着相同的语句将在应用程序重新启动时产生相同的字符串，并且允许更好地工作的 DB 查询计划缓存。

+   **[sql]**

    当使用子查询时生成的“mini”列标签，用于解决 SQLite 行为不佳的问题，不理解“foo.id”等同于“id”，现在只在从中选择这些命名列的情况下生成

    参考：[#513](https://www.sqlalchemy.org/trac/ticket/513)

+   **[sql]**

    ColumnElement 上的 label() 方法将正确地将基本元素的 TypeEngine 传播到标签，包括从 scalar=True select() 语句创建的标签。

+   **[sql]**

    MS-SQL 更好地检测查询是否为子查询，并知道不为这些生成 ORDER BY 语句

    参考：[#513](https://www.sqlalchemy.org/trac/ticket/513)

+   **[sql]**

    修复了在大多数 dbapis 中 fetchmany() 的 “size” 参数是位置参数的问题

    参考：[#505](https://www.sqlalchemy.org/trac/ticket/505)

+   **[sql]**

    将 None 作为参数传递给 func.<something> 将产生一个 NULL 参数

+   **[sql]**

    Unicode URL 中的查询字符串将键编码为 ascii 以进行 **kwargs 兼容

+   **[sql]**

    对原始 execute() 更改的微小调整，也支持元组作为位置参数，而不仅仅是列表

    参考：[#523](https://www.sqlalchemy.org/trac/ticket/523)

+   **[sql]**

    修复了 case() 构造的问题，将第一个 WHEN 条件的类型作为 case 语句的返回类型传播

### 扩展

+   **[extensions]**

    修复了 AssociationProxy 的问题，使得多个 AssociationProxy 对象可以与单个关联集合关联。

+   **[extensions]**

    根据它们的键（即 __name__）为 assign_mapper 方法命名

### mysql

+   **[mysql]**

    支持将 SSL 参数作为内联 URL 查询字符串给出，以“ssl_”为前缀，感谢 terjeros@gmail.com。

+   **[mysql] [<schemaname>]**

    mysql 使用 “DESCRIBE.<tablename>” 来确定表是否存在，如果表不存在则捕获异常，以支持 Unicode 表名以及模式名。已在 MySQL5 上测试，但应该也适用于 4.1 系列。(#557)

### sqlite

+   **[sqlite]**

    移除了 SQLite 将唯一索引作为主键的一部分反映的愚蠢行为（？！）

### mssql

+   **[mssql]**

    pyodbc 现在是 MSSQL 的首选 DB-API，如果没有明确请求模块，则将在模块探测时首先加载。

+   **[mssql]**

    现在使用 @@SCOPE_IDENTITY 而不是 @@IDENTITY。此行为可以通过 engine_connect “use_scope_identity” 关键字参数覆盖，也可以在 dburi 中指定。

### oracle

+   **[oracle]**

    允许对具有 LIMIT/OFFSET 特性的相同 SELECT 对象进行连续编译的小修复。oracle 方言需要修改对象以具有 ROW_NUMBER OVER，并且在连续编译时未执行完整系列步骤。

### 杂项

+   **[引擎]**

    用于发出警告的 warnings 模块（而不是日志记录）

+   **[引擎]**

    清理所有引擎中的 DBAPI 导入策略

    参考：[#480](https://www.sqlalchemy.org/trac/ticket/480)

+   **[引擎]**

    引擎内部的重构减少了复杂性，代码路径数量；将更多状态放在 ExecutionContext 中，以允许更多方言控制游标处理，结果集。 ResultProxy 完全重构，还有两个用于不同目的的“缓冲”结果集版本。

+   **[引擎]**

    postgres 中完全支持服务器端游标

    参考：[#514](https://www.sqlalchemy.org/trac/ticket/514)

+   **[引擎]**

    改进的框架用于自动失效连接，这些连接已丢失其底层数据库，通过特定于方言的检测异常来对应该数据库的断开相关错误消息。此外，当检测到“连接不再打开”条件时，整个连接池将被丢弃并替换为新实例。＃516

+   **[引擎]**

    sqlalchemy.databases 中的方言成为 setuptools 入口点。加载内置数据库方言的方式与以往相同，但如果找不到任何方言，则会回退尝试使用 pkg_resources 加载外部模块

    参考：[#521](https://www.sqlalchemy.org/trac/ticket/521)

+   **[引擎]**

    引擎包含一个“url”属性，引用 create_engine() 使用的 url.URL 对象。

+   **[informix]**

    添加 informix 支持！感谢 James Zhang，他付出了大量努力。

## 0.3.6

发布日期：2007 年 3 月 23 日 星期五

### orm

+   **[orm]**

    SelectResults 扩展的全部功能集已合并到 Query 的一组新方法中。这些方法都提供“生成”行为，即复制 Query 并返回一个添加了额外标准的新 Query。新方法包括：

    > +   filter() - 将选择标准应用于查询
    > +   
    > +   filter_by() - 将“by”样式标准应用于查询
    > +   
    > +   avg() - 返回给定列上的 avg() 函数
    > +   
    > +   join() - 加入到属性（或跨属性列表）
    > +   
    > +   outerjoin() - 类似于 join() 但使用 LEFT OUTER JOIN
    > +   
    > +   limit()/offset() - 应用基于范围的 LIMIT/OFFSET 访问，即应用 limit/offset：session.query(Foo)[3:5]
    > +   
    > +   distinct() - 应用 DISTINCT
    > +   
    > +   list() - 评估标准并返回结果

    Query 的 API 没有进行不兼容的更改，也没有废弃任何方法。现有方法如 select()、select_by()、get()、get_by() 都会立即执行查询并像以前一样返回结果。join_to()/join_via() 仍然存在，尽管生成的 join()/outerjoin() 方法更容易使用。

+   **[orm]**

    与 instances()一起使用的多个映射器的返回值现在返回所请求的映射器列表的笛卡尔积，表示为元组的列表。这对应于文档中的行为。为了正确匹配实例，当使用此功能时，“唯一化”被禁用。

+   **[orm]**

    Query 具有 add_entity()和 add_column()生成方法。这些方法将在编译时将给定的映射器/类或 ColumnElement 添加到查询中，并将它们应用于 instances()方法。用户负责构建合理的连接条件（否则可能会得到完整的笛卡尔积）。结果集是元组的列表，不唯一。

+   **[orm]**

    字符串和列也可以发送到 instances()的*args 中，其中这些确切的结果列将成为结果元组的一部分。

+   **[orm]**

    完整的 select()构造可以传递给 query.select()（无论如何都有效），还可以使用 query.selectfirst()、query.selectone()，它们将被直接使用（即，不会编译查询）。与将结果发送到 instances()类似。

+   **[orm]**

    急切加载不会对由急切加载器之外的内容放置在选择语句中的“order by”子句进行“别名化”，以修复在中所示的重复列的可能性。然而，这意味着您必须更加小心地处理放置在 Query.select()的“order by”中的列，您必须在您的条件中明确命名它们（即，您不能依赖于急切加载器为您添加它们）

    参考：[#495](https://www.sqlalchemy.org/trac/ticket/495)

+   **[orm]**

    在 Session 中添加了一个方便的多用途“identity_key()”方法，允许为主键值、实例和行生成标识键，感谢 Daniel Miller

+   **[orm]**

    即使在操作发生在操作的“backref”侧时，也将正确处理多对多表

    参考：[#249](https://www.sqlalchemy.org/trac/ticket/249)

+   **[orm]**

    添加了“refresh-expire”级联。允许刷新()和 expire()调用沿关系传播。

    参考：[#492](https://www.sqlalchemy.org/trac/ticket/492)

+   **[orm]**

    对多态关系进行了更多修复，涉及到对多对一关系到多态映射器的正确延迟子句生成。还修复了“方向”检测，更具体地针对属于多态联合的列与不属于多态联合的列。

    参考：[#493](https://www.sqlalchemy.org/trac/ticket/493)

+   **[orm]**

    当使用“viewonly=True”从其他表中拉取到连接条件中不是关系的父/子映射的其他表时，对关系计算进行了一些修复

+   **[orm]**

    在包含对循环链之外的其他实例的引用的循环引用关系上进行了刷新修复，当循环中的某些对象实际上不是刷新的一部分时

+   **[orm]**

    对“将 A 对象与 B 集合一起刷新，但是您将 C 放入集合中”的“检查条件”进行了强制性检查 - **即使 C 是 B 的子类**，除非 B 的映射器以多态方式加载。否则，集合稍后将加载应为“C”的“B”（因为它不是多态的），这会破坏双向关系（即 C 有其 A，但 A 的反向引用将惰性加载为类型为“B”的不同实例）。此检查将会影响一些未出现问题的人，因此错误消息还将记录一个标志“enable_typechecks=False”，以禁用此检查。但请注意，特别是在没有此检查的情况下，双向关系变得脆弱。

    参考：[#500](https://www.sqlalchemy.org/trac/ticket/500)

### sql

+   **[sql]**

    bindparam() 名称现在是可重复的！在单个语句中指定两个不同的 bindparam()，并且名称相同，键将是共享的。适当的位置/命名参数在编译时转换。对于具有冲突名称的绑定参数的“别名”旧行为，请指定“unique=True” - 此选项仍然用于所有自动生成的（基于值的）绑定参数的内部使用。

+   **[sql]**

    对绑定参数作为列子句的支持稍微改进，可以通过 bindparam() 或 literal() 实现，例如 select([literal('foo')])。

+   **[sql]**

    MetaData 可以通过构造函数的 “url” 或 “engine” kwargs，或使用 connect() 方法绑定到引擎。BoundMetaData 与 MetaData 相同，只是 engine_or_url 参数是必需的。DynamicMetaData 与之相同，并默认提供线程本地连接。

+   **[sql]**

    exists() 现在可以作为一个独立的可选择对象使用，而不仅仅是在 WHERE 子句中使用，即 exists([columns], criterion).select()。

+   **[sql]**

    相关子查询可以在 ORDER BY、GROUP BY 中使用。

+   **[sql]**

    修复了使用显式连接执行函数的功能，即 conn.execute(func.dosomething())。

+   **[sql]**

    select() 上的 use_labels 标志不会为文字文本列元素自动创建标签，因为我们无法对文本进行任何假设。要为文字列创建标签，可以说“somecol AS somelabel”，或使用 literal_column(“somecol”).label(“somelabel”)。

+   **[sql]**

    当将文字列在其可选择的列集合中“代理”时，对于字面列，不会发生引用（is_literal 标志被传播）。通过 literal_column(“somestring”) 指定文字列。

+   **[sql]**

    在 Join.select() 中添加了 “fold_equivalents” 布尔参数，它根据连接条件从结果列子句中删除已知为等效的“重复”列。当构造与 Postgres 抱怨如果存在重复列名的连接子查询时非常有用。

+   **[sql]**

    修复了 ForeignKeyConstraint 上的 use_alter 标志。

    参考：[#503](https://www.sqlalchemy.org/trac/ticket/503)

+   **[sql]**

    在 topological.py 中使用了 2.4-only “reversed”的修复用法。

    参考：[#506](https://www.sqlalchemy.org/trac/ticket/506)

+   **[sql]**

    对于黑客，重构了 ClauseElement 和 SchemaItem 的“visitor”系统，以便项的遍历由 ClauseVisitor 本身控制，使用方法 visitor.traverse(item)。accept_visitor() 方法仍然可以直接调用，但不会遍历子项。ClauseElement/SchemaItem 现在有一个可配置的 get_children() 方法，用于返回每个父对象的子元素集合。这允许项的完整��历清晰和明确（以及可记录），并且有一种限制遍历的简单方法（只需传递标志，这些标志由适当的 get_children() 方法捕获）。

    参考：[#501](https://www.sqlalchemy.org/trac/ticket/501)

+   **[sql]**

    当 else_ 参数设置为零时，case 语句现在可以正常工作。

### 扩展

+   **[extensions]**

    SelectResults 上的 options() 方法现在像其他 SelectResults 方法一样实现了“生成”。但无论如何，你现在只会使用 Query。

    参考：[#472](https://www.sqlalchemy.org/trac/ticket/472)

+   **[extensions]**

    query() 方法由 assignmapper 添加。这有助于导航到 Query 上的所有新的生成方法。

### mysql

+   **[mysql]**

    为 MSString 添加了一个通用的 **kwargs，以帮助反射模糊类型（如 MS 4.0 中的“varchar() binary”）

+   **[mysql]**

    添加了明确的 MSTimeStamp 类型，当使用 types.TIMESTAMP 时生效。

### oracle

+   **[oracle]**

    使任何大小输入的二进制工作！cx_oracle 工作正常，是我的错，因为 BINARY 被传递而不是 BLOB 用于 setinputsizes（还有单元测试甚至没有设置输入大小）。

+   **[oracle]**

    也修复了在单独的更改集上对 CLOB 的读写。

+   **[oracle]**

    auto_setinputsizes 对于 Oracle 默认为 True，修复了它错误地传播错误类型的情况。

### 杂项

+   **[ms-sql]**

    移除了 DATE 列类型上的秒输入（可能

    应该完全删除时间）

+   **[ms-sql]**

    浮点字段中的空值不再引发错误

+   **[ms-sql]**

    LIMIT 与 OFFSET 现在会引发错误（MS-SQL 不支持 OFFSET）

+   **[ms-sql]**

    添加了一个使用 MSSQL 类型 VARCHAR(max) 而不是 TEXT 用于大型无大小字符串字段的功能。使用新的“text_as_varchar”来启用它。

    参考：[#509](https://www.sqlalchemy.org/trac/ticket/509)

+   **[ms-sql]**

    没有 LIMIT 的 ORDER BY 子句现在在子查询中被剥离，因为 MS-SQL 禁止这种用法

+   **[ms-sql]**

    清理模块导入代码；可指定的 DB-API 模块；模块偏好的更明确排序。

    参考：[#480](https://www.sqlalchemy.org/trac/ticket/480)

## 0.3.5

发布日期：Thu Feb 22 2007

### orm

+   **[orm] [bugs]**

    另一个重构关系计算。允许更准确的 ORM 行为与来自/到/之间的映射器的关系，特别是多态映射器，还有它们与 Query、SelectResults 的使用。票据包括,,。

    参考：[#439](https://www.sqlalchemy.org/trac/ticket/439), [#441](https://www.sqlalchemy.org/trac/ticket/441), [#448](https://www.sqlalchemy.org/trac/ticket/448)

+   **[orm] [错误]**

    删除了在类上指定自定义集合的已弃用方法；现在必须使用“collection_class”选项。旧方法开始在人们使用 assign_mapper()时产生冲突，现在修补了一个“options”方法，与一个名为“options”的关系一起使用。 （关系优先于 monkeypatched assign_mapper 方法）。

+   **[orm] [错误]**

    extension()查询选项传播到 Mapper._instance()方法，以便调用所有与加载相关的方法

    参考：[#454](https://www.sqlalchemy.org/trac/ticket/454)

+   **[orm] [错误]**

    继承映射器的急切关系如果关系没有返回任何行也不会失败。

+   **[orm] [错误]**

    修复了对多个后代类上的急切关系加载错误

    参考：[#486](https://www.sqlalchemy.org/trac/ticket/486)

+   **[orm] [错误]**

    修复了非常大的拓扑排序问题，感谢 ants.aasma at gmail

    参考：[#423](https://www.sqlalchemy.org/trac/ticket/423)

+   **[orm] [错误]**

    急切加载在检测“自引用”关系时稍微更严格，特别是在多态映射器之间。这导致“急切降级”到延迟加载。

+   **[orm] [错误]**

    改进了对嵌入到查询.select()的“where”条件中的复杂查询的支持

    参考：[#449](https://www.sqlalchemy.org/trac/ticket/449)

+   **[orm] [错误]**

    映射器选项如 eagerload()、lazyload()、deferred()，将适用于“synonym()”关系

    参考：[#485](https://www.sqlalchemy.org/trac/ticket/485)

+   **[orm] [错误]**

    修复了级联操作错误地包括已删除的集合项的错误

    参考：[#445](https://www.sqlalchemy.org/trac/ticket/445)

+   **[orm] [错误]**

    修复了当一个一对多子项在单个工作单元中移动到新父项时的关系删除错误

    参考：[#478](https://www.sqlalchemy.org/trac/ticket/478)

+   **[orm] [错误]**

    修复了关系删除错误，其中父/子关系的子项只有一个列作为主键/外键时，如果手动删除或使用“delete”级联而没有使用“delete-orphan”，则会引发“清空主键”错误

+   **[orm] [错误]**

    修复了延迟加载，以便在只设置 PK 列属性时不会错误地发生加载操作

+   **[orm] [增强]**

    实现了 mapper 的 foreign_keys 参数。与 primaryjoin/secondaryjoin 参数一起使用，以指定/覆盖在 Table 实例上定义的外键。

    参考：[#385](https://www.sqlalchemy.org/trac/ticket/385)

+   **[orm] [增强]**

    contains_eager('foo')自动意味着 eagerload('foo')

+   **[orm] [增强]**

    添加了 contains_eager()的“alias”参数。使用它来指定查询中用于急切加载的子项的别名的字符串名称或 Alias 实例。比“装饰器”更容易使用

+   **[orm] [增强]**

    添加了“contains_alias()”选项，用于将结果集映射到映射表的别名

+   **[orm] [增强]**

    添加了对`py2.5`“with”语句的支持到`SessionTransaction`

    参考：[#468](https://www.sqlalchemy.org/trac/ticket/468)

### sql

+   **[sql]**

    “case_sensitive”的值现在默认为`True`，不管标识符的大小写如何，除非明确设置为`False`。这是因为对象可能被标记为其他包含混合大小写的内容，传播“case_sensitive=False”会破坏这一点。在使用标签和“fake”列对象时修复引用的其他问题

+   **[sql]**

    添加了一个“supports_execution()”方法到`ClauseElement`，以便各种类型的子句可以表达是否适合执行...例如，您可以执行“select”，但不能执行“Table”或“Join”。

+   **[sql]**

    修复了对引擎、连接上直接文本`execute()`的参数传递。可以处理`*args`或列表实例作为位置参数，`**kwargs`或字典实例作为命名参数，或列表的列表或字典以调用`executemany()`

+   **[sql]**

    对`BoundMetaData`进行了小修复，以接受`unicode`或字符串`URLs`

+   **[sql]**

    修复了由`andrija at gmail`提供的命名`PrimaryKeyConstraint`生成

    参考：[#466](https://www.sqlalchemy.org/trac/ticket/466)

+   **[sql]**

    修复了在列上生成`CHECK`约束的问题

    参考：[#464](https://www.sqlalchemy.org/trac/ticket/464)

+   **[sql]**

    修复了`tometadata()`操作以传播列和表级别的约束

### `extensions`

+   **[extensions]**

    添加了`distinct()`方法到`SelectResults`。通常只在使用`count()`时会有所不同。

+   **[extensions]**

    添加了`options()`方法到`SelectResults`，相当于`query.options()`

    参考：[#472](https://www.sqlalchemy.org/trac/ticket/472)

+   **[extensions]**

    添加了可选的`__table_opts__`字典到`ActiveMapper`，将`kw`选项发送到`Table`对象

    参考：[#462](https://www.sqlalchemy.org/trac/ticket/462)

+   **[extensions]**

    添加了`selectfirst()`，`selectfirst_by()`到`assign_mapper`

    参考：[#467](https://www.sqlalchemy.org/trac/ticket/467)

### `mysql`

+   **[mysql]**

    修复了在旧的数据库上反射可能返回“show variables like”语句的`array()`类型的问题

### `mssql`

+   **[mssql]**

    对`pyodbc`的初步支持（耶！）

    参考：[#419](https://www.sqlalchemy.org/trac/ticket/419)

+   **[mssql]**

    添加了对`NVARCHAR`类型的更好支持

    参考：[#298](https://www.sqlalchemy.org/trac/ticket/298)

+   **[mssql]**

    修复了`pymssql`上的提交逻辑

+   **[mssql]**

    修复了带有模式的`query.get()`

    参考：[#456](https://www.sqlalchemy.org/trac/ticket/456)

+   **[mssql]**

    修复了非整数关系

    参考：[#473](https://www.sqlalchemy.org/trac/ticket/473)

+   **[mssql]**

    现在可以在运行时选择`DB-API`模块

    参考：[#419](https://www.sqlalchemy.org/trac/ticket/419)

+   **[mssql] [415] [481] [tickets:422]**

    现在通过了更多的单元测试

+   **[mssql]**

    与`ANSI`函数更好的单元测试兼容性

    参考：[#479](https://www.sqlalchemy.org/trac/ticket/479)

+   **[mssql]**

    改进了对具有自动插入的隐式序列`PK`列的支持

    参考：[#415](https://www.sqlalchemy.org/trac/ticket/415)

+   **[mssql]**

    修复了 adodbapi 中空密码的问题

    参考：[#371](https://www.sqlalchemy.org/trac/ticket/371)

+   **[mssql]**

    修复了使用 pyodbc 时单元测试无法正常工作的问题

    参考：[#481](https://www.sqlalchemy.org/trac/ticket/481)

+   **[mssql]**

    修复了 db-url 查询中的 auto_identity_insert

+   **[mssql]**

    在 db-url 查询参数中添加了 query_timeout。目前仅适用于 pymssql

+   **[mssql]**

    已测试通过 pymssql 0.8.0（现在是 LGPL）

### oracle

+   **[oracle]**

    当将“rowid”作为 ORDER BY 列返回或在 ROW_NUMBER OVER 中使用时，oracle 方言会检查其应用的可选择项，并在不适用时切换到表 PK，即对于 UNION。仍需检查 DISTINCT、GROUP BY（rowid 无效的其他地方）等。允许多态映射正常运行。

    参考：[#436](https://www.sqlalchemy.org/trac/ticket/436)

+   **[oracle]**

    非主键列上的序列将在插入时正确触发

+   **[oracle]**

    添加了 PrefetchingResultProxy 支持，以在已知存在时预取 LOB 列，修复了

    参考：[#435](https://www.sqlalchemy.org/trac/ticket/435)

+   **[oracle]**

    实现了基于同义词的表反射，包括跨数据库链接

    参考：[#379](https://www.sqlalchemy.org/trac/ticket/379)

+   **[oracle]**

    当由于某些权限错误而无法反映相关表时，会发出日志警告

    参考：[#363](https://www.sqlalchemy.org/trac/ticket/363)

### 杂项

+   **[postgres]**

    更好地反映了备用模式表的序列

    参考：[#442](https://www.sqlalchemy.org/trac/ticket/442)

+   **[postgres]**

    非主键列上的序列将在插入时正确触发

+   **[postgres]**

    添加了 PGInterval 类型，PGInet 类型

    参考：[#444](https://www.sqlalchemy.org/trac/ticket/444), [#460](https://www.sqlalchemy.org/trac/ticket/460)

## 0.3.4

发布日期：2007 年 1 月 23 日

### 通用

+   **[general]**

    全局“insure”->”ensure”更改。在美式英语中，“insure”实际上与“ensure”在很大程度上是可以互换的（字典上是这么说的），所以我并不是完全文盲，但“ensure”无疑是非歧义的更佳选择。

### orm

+   **[orm]**

    在这个潘多拉魔盒中打开了第一个漏洞：说 query.select_by(somerelationname=someinstance) 将创建“somerelationname”映射器表示的主键列与“someinstance”实际主键的连接。

+   **[orm]**

    重新设计了关系与“多态”映射器的交互方式，即具有 select_table 和多态标志的映射器。更好地确定适当的连接条件，与用户定义的连接条件的交互，以及对自引用多态映射器的支持。

+   **[orm]**

    关于多态映射关系，当编译关系时进行了一些更深入的错误检查，以检测在关系的两侧都在主连接条件中具有外键引用的模糊“primaryjoin”的情况。还加强了用于定位“关系方向”的条件，将关系的“外键”与“主连接”关联起来

+   **[orm]**

    对“concrete”继承映射概念进行了一点改进，尽管该概念尚未完全明确（添加了支持在多态基础上创建具体映射器的测试用例）。

+   **[orm]**

    修复了在 synonym()上“proxy=True”行为的问题

+   **[orm]**

    修复了删除孤儿在许多对多关系中基本上无法工作的错误，反向引用通常隐藏了症状

    参考：[#427](https://www.sqlalchemy.org/trac/ticket/427)

+   **[orm]**

    在映射器编译步骤中添加了互斥锁。我一直不愿意在 SA 中添加任何线程内容，但这是真正需要的一个地方，因为映射器通常是“全局”的，虽然它们的状态在正常操作期间不会改变，但初始编译步骤会显著修改内部状态，并且这一步通常不会发生在模块级别的初始化时间（除非调用 compile()），而是在第一次请求时发生

+   **[orm]**

    实际上实现了“session.merge()”的基本概念。需要更多测试。

+   **[orm]**

    添加了“compile_mappers()”函数作为编译所有映射器的快捷方式

+   **[orm]**

    修复了 MapperExtension 创建实例的问题，使实体名称正确关联到新实例

+   **[orm]**

    ORM 对象实例化的速度增强，行的急切加载

+   **[orm]**

    发送到“cascade”字符串的无效选项将引发异常

    参考：[#406](https://www.sqlalchemy.org/trac/ticket/406)

+   **[orm]**

    修复了映射器刷新/过期中的错误，急切加载器未正确重新填充项目列表

    参考：[#407](https://www.sqlalchemy.org/trac/ticket/407)

+   **[orm]**

    修复了 post_update 以确保即使在非插入/删除场景下也会更新行

    参考：[#413](https://www.sqlalchemy.org/trac/ticket/413)

+   **[orm]**

    如果您尝试修改实体的主键值然后刷新它，将添加错误消息

    参考：[#412](https://www.sqlalchemy.org/trac/ticket/412)

### sql

+   **[sql]**

    添加了对 ResultProxy 的“fetchmany()”支持

+   **[sql]**

    添加了对列“key”属性在 row[<key>]/row.<key>中可用的支持

+   **[sql]**

    将“BooleanExpression”更改为从“BinaryExpression”子类化，以便布尔表达式也可以遵循列子句行为（即 label()等）。

+   **[sql]**

    从 func.<xxx>调用中修剪尾随下划线，例如 func.if_()

+   **[sql]**

    修复了使用 append_column()单独调用构建选择语句的列列表时子查询相关性的问题；这修复了一个 ORM 错误，即嵌套的选择语句未与 Query 对象生成的主选择相关联。

+   **[sql]**

    另一个修复子查询相关性的方法，以便仅具有一个 FROM 元素的子查询*不*相关联该单个元素，因为查询中至少需要一个 FROM 元素。

+   **[sql]**

    默认的“timezone”设置现在为 False。这对应于 Python 的 datetime 行为以及 Postgres 的 timestamp/time 类型（目前是唯一有时区敏感的方言）

    参考：[#414](https://www.sqlalchemy.org/trac/ticket/414)

+   **[sql]**

    “op()”函数现在被视为“操作”，而不是“比较”。区别在于，操作会产生一个 BinaryExpression，从中可以进行进一步的操作，而比较会产生更严格的 BooleanExpression

+   **[sql]**

    尝试将反射的主键列重新定义为非主键会引发错误

+   **[sql]**

    类型系统略有修改，以支持可以被方言覆盖的 TypeDecorators（好吧，这不是很清楚，它允许下面的 mssql 调整成为可能）

### 扩展

+   **[扩展]**

    为 assign_mapper 添加了“validate=False”参数，如果为 True，则确保只有映射的属性被命名

    参考：[#426](https://www.sqlalchemy.org/trac/ticket/426)

+   **[扩展]**

    assign_mapper 添加了“options”、“instances”函数（例如 MyClass.instances()）

### mysql

+   **[mysql]**

    mysql 在 SHOW CREATE TABLE 期间使用的外键引号类型不一致，反射已更新以适应所有三种样式

    参考：[#420](https://www.sqlalchemy.org/trac/ticket/420)

+   **[mysql]**

    mysql 表创建选项现在可以在通用传递上工作，例如 Table(…, mysql_engine=’InnoDB’, mysql_collate=”latin1_german2_ci”, mysql_auto_increment=”5”, mysql_<somearg>…), 有助于

    参考：[#418](https://www.sqlalchemy.org/trac/ticket/418)

### mssql

+   **[mssql]**

    添加了一个 NVarchar 类型（生成 NVARCHAR），还有一个提供 Unicode 翻译的 MSUnicode，不管方言的 convert_unicode 设置如何。

### oracle

+   **[oracle]**

    *轻微*支持二进制，但仍需弄清如何插入相当大的值（超过 4K）。需要将 auto_setinputsizes=True 发送到 create_engine()，行必须逐个完全获取，等等。

### 杂项

+   **[postgres]**

    修复了对表的初始 checkfirst 以考虑当前模式

    参考：[#424](https://www.sqlalchemy.org/trac/ticket/424)

+   **[postgres]**

    postgres 有一个可选的“server_side_cursors=True”标志，将利用服务器端游标。这些适用于仅获取部分结果并且必须处理非常大的无界结果集。虽然我们希望这是默认行为，但不同的环境似乎有不同的结果，原因尚未确定，因此我们目前将该功能默认关闭。使用了最近在 psycopg 邮件列表上发现的一个显然未记录的 psycopg2 行为。

+   **[postgres]**

    为具有 PGBigInteger/autoincrement 的 postgres 表添加了“BIGSERIAL”支持

+   **[postgres]**

    修复了 Postgres 反射以更好地处理存在模式名称时的情况；感谢 jason (at) ncsmags.com

    参考：[#402](https://www.sqlalchemy.org/trac/ticket/402)

+   **[火鸟]**

    约束创建顺序将主键放在所有其他约束之前；对于 Firebird 是必需的，对于其他数据库也是一个好主意

    参考：[#408](https://www.sqlalchemy.org/trac/ticket/408)

+   **[火鸟]**

    Firebird 修复以自动加载多字段外键

    参考：[#409](https://www.sqlalchemy.org/trac/ticket/409)

+   **[火鸟]**

    Firebird NUMERIC 类型正确处理没有精度的类型

    参考：[#409](https://www.sqlalchemy.org/trac/ticket/409)

## 0.3.3

发布日期：2006 年 12 月 15 日 星期五

+   **[无标签]**

    修复了基于字符串的 FROM 子句，即 select(…, from_obj=[“sometext”])

+   **[无标签]**

    修复了 passive_deletes 标志，lazy=None（noload）标志

+   **[无标签]**

    添加了处理大型集合的示例/文档

+   **[无标签]**

    在 sqlalchemy 命名空间中添加了 object_session()方法

+   **[无标签]**

    修复了 QueuePool 错误，使其更好地重新连接到无法访问的数据库（感谢 Sébastien Lelong），还修复了 dispose()方法

+   **[无标签]**

    使 MySQL rowcount 正确工作的补丁！

    参考：[#396](https://www.sqlalchemy.org/trac/ticket/396)

+   **[无标签]**

    修复了 MySQL 捕获 2006/2014 错误以正确重新引发 OperationalError 异常

## 0.3.2

发布日期：2006 年 12 月 10 日 星期日

+   **[无标签]**

    修复了主要连接池错误。修复了 MySQL 不同步错误，还将防止事务在所有数据库中意外回滚

    参考：[#387](https://www.sqlalchemy.org/trac/ticket/387)

+   **[无标签]**

    与 0.3.1 相比的主要速度增强，将速度提升至 0.2.8 水平

+   **[无标签]**

    使数十个生成日志消息的调试日志调用成为有条件的

+   **[无标签]**

    修复了级联规则中的错误，从而可能在保存/更新级联时不必要地级联整个对象图

+   **[无标签]**

    属性模块中的各种加速

+   **[无标签]**

    会话中的身份映射默认情况下*不再使用弱引用*。要使其使用弱引用，请使用 create_session(weak_identity_map=True)修复

    参考：[#388](https://www.sqlalchemy.org/trac/ticket/388)

+   **[无标签]**

    MySQL 检测到错误 2006（服务器已断开连接）和 2014（命令不同步）并使发生错误的连接无效。

+   **[无标签]**

    MySQL bool 类型修复：

    参考：[#307](https://www.sqlalchemy.org/trac/ticket/307)

+   **[无标签]**

    Postgres 反射修复：

    参考：[#349](https://www.sqlalchemy.org/trac/ticket/349)，[#382](https://www.sqlalchemy.org/trac/ticket/382)

+   **[无标签]**

    为 EXCEPT、INTERSECT、EXCEPT ALL、INTERSECT ALL 添加关键字

    参考：[#247](https://www.sqlalchemy.org/trac/ticket/247)

+   **[无标签]**

    assignmapper 扩展中的 assign_mapper 返回创建的映射器

    参考：[#2110](https://www.sqlalchemy.org/trac/ticket/2110)

+   **[无标签]**

    为 Select 类添加了 label()函数，当使用 scalar=True 创建标量子查询时，例如“select x, y, (select max(foo) from table) AS foomax from table”

+   **[no_tags]**

    为 ForeignKey 添加了 onupdate 和 ondelete 关键字参数；如果存在，传播到底层 ForeignKeyConstraint。（但是不要在另一个方向传播）

+   **[no_tags]**

    修复了 session.update()以保留传入对象的“脏”状态

+   **[no_tags]**

    通过将 selectable 发送到 IN，通过 in_()函数不再创建多个 select 的“union”；现在只允许一个 selectable 传递给 in_()函数（如果需要 union，请自行创建 union）

+   **[no_tags]**

    改进了通过 cascade=”none”等禁用保存-更新级联的支持

+   **[no_tags]**

    为 relation()添加了“remote_side”参数，仅与自引用映射器一起使用，以强制父/子关系的方向。替换了用于“切换”方向的“foreignkey”参数的使用。“foreignkey”参数对于所有用途都已弃用，并最终将被一个专用于映射器上的 ForeignKey 规范的参数所取代。

## 0.3.1

发布日期：Mon Nov 13 2006

### orm

+   **[orm]**

    “delete”级联将加载所有子对象，如果它们尚未加载。可以通过在 relation()上设置 passive_deletes=True 来关闭此功能（即旧行为）。

+   **[orm]**

    重新设计的急切查询生成调整，以避免在循环急切加载关系（如反向引用）上失败

+   **[orm]**

    修复了 eagerload()（也不是 lazyload()）选项未正确指示 Query 在生成 LIMIT 查询时是否使用“嵌套”的错误。

+   **[orm]**

    修复了刷新时循环依赖排序中的错误；如果对象 A 包含一个循环的多对一关系到对象 B，并且对象 B 刚刚附加到对象 A，*但*对象 B 本身没有更改，则不会发生 B 的主键属性与 A 的外键属性的同步。

    参考：[#360](https://www.sqlalchemy.org/trac/ticket/360)

+   **[orm]**

    为 query.count 实现了 from_obj 参数，改进了 selectresults 上的 count 函数

    参考：[#325](https://www.sqlalchemy.org/trac/ticket/325)

+   **[orm]**

    在 ORM 关系的“级联”步骤中添加了一个断言，检查附加到父对象的对象的类是否合适（即如果 A.items 存储 B 对象，则如果向 A.items 附加 C，则引发错误）

+   **[orm]**

    新扩展 sqlalchemy.ext.associationproxy，提供透明的“关联对象”映射。新示例 examples/association/proxied_association.py 进行了说明。

+   **[orm]**

    改进了单表继承，以加载目标类下面的完整层次结构

+   **[orm]**

    修复了拓扑排序中一个微妙条件的问题，其中一个节点可能出现两次

    参考：[#362](https://www.sqlalchemy.org/trac/ticket/362)

+   **[orm]**

    对拓扑排序进行了额外的重构，重构，用于

    参考：[#365](https://www.sqlalchemy.org/trac/ticket/365)

+   **[orm]**

    对于某种类型，“delete-orphan” 可以在多个���类上设置；如果实例未附加到*任何*这些父类中，则该实例是“孤立的”

### 杂项

+   **[engine/pool]**

    一些新的 Pool 实用程序类，更新了文档

+   **[engine/pool]**

    “use_threadlocal” 在 Pool 上默认为 False（与 create_engine 相同）

+   **[engine/pool]**

    修复了对 Compiled 对象的直接执行

+   **[engine/pool]**

    重新设计 create_engine() 以严格处理传入的 **kwargs。所有关键字参数必须由方言、连接池和引擎构造函数中的一个消耗，否则将抛出 TypeError，其中描述了与所选方言/池/引擎配置相关的无效 kwargs 的完整集合。

+   **[databases/types]**

    MySQL 在“describe”时捕获异常，并报告为 NoSuchTableError

+   **[databases/types]**

    进一步修复了 sqlite 布尔值，默认情况下不起作用

+   **[databases/types]**

    修复了在使用模式时对 postgres 序列引用的问题

## 0.3.0

发布日期：2006 年 10 月 22 日（星期日）

### 一般

+   **[general]**

    现在通过标准的 Python “logging” 模块实现日志记录。“echo”关键字参数仍然可用，但为它们各自的类/实例设置/取消日志级别。所有日志记录都可以通过 Python API 直接控制，通过为“sqlalchemy”命名空间中的记录器设置 INFO 和 DEBUG 级别。类级别的日志记录在“sqlalchemy.<module>.<classname>”下，实例级别的日志记录在“sqlalchemy.<module>.<classname>.0x..<00-FF>”下。测试套件包括“--log-info”和“--log-debug”参数，它们独立于--verbose/--quiet 工作。在 orm 中添加了日志记录，以便跟踪映射器配置、行迭代。

+   **[general]**

    文档生成系统已经进行了大幅简化设计，并与 Markdown 更加集成

### orm

+   **[orm]**

    修改了属性跟踪，更智能地检测更改，特别是对可变类型。TypeEngine 对象现在在定义如何比较两个标量实例方面发挥更大作用，包括通过 PickleType 实现的 MutableType 混合。unit-of-work 现在将“脏”列表跟踪为所有属性管理器检测到更改的所有持久对象的表达式。修复的基本问题是检测 PickleType 对象上的更改，但也将类型处理和“修改”对象检查泛化为更完整和可扩展的形式。

+   **[orm]**

    对“属性加载器”和“选项”架构进行了广泛的重构。ColumnProperty 和 PropertyLoader 通过可切换的“策略”定义它们的加载行为，MapperOptions 不再使用映射器/属性复制来运行；它们通过查询/实例时间通过 QueryContext 和 SelectionContext 对象传播。所有用于处理继承以及 options() 的内部映射器和属性的复制都已被移除；映射器和属性的结构比以前简单得多，并且在新的“interfaces”模块中清晰地列出。

+   **[orm]**

    关于映射器/属性大修，内部重构到映射器 instances() 方法，使用 SelectionContext 对象来跟踪操作过程中的状态。轻微的 API 破坏：由于更改，MapperExtension 上的 append_result() 和 populate_instances() 方法现在具有略有不同的方法签名；希望这些方法目前还没有广泛使用。

+   **[orm]**

    instances() 方法现在移至 Query，向后兼容版本仍然在 Mapper 上保留。

+   **[orm]**

    添加了 contains_eager() MapperOption，与 instances() 结合使用，用于指定应该从结果集中急切加载的属性，默认情况下使用它们的普通列名，或者在给定自定义行翻译函数的情况下进行翻译。

+   **[orm]**

    更多的工作单元提交方案重新排列，以更好地允许循环刷新内的依赖关系正常工作...更新了任务遍历/日志记录实现

+   **[orm]**

    多态映射器（即使用继承）现在按照所有继承类的表的顺序生成 INSERT 语句

    参考：[#321](https://www.sqlalchemy.org/trac/ticket/321)

+   **[orm]**

    添加了一个自动的“行切换”功能到映射中，它将检测具有相同标识键的待处理实例/已删除实例对，并将 INSERT/DELETE 转换为单个 UPDATE

+   **[orm]**

    “关联”映射简化以利用自动“行切换”功能

+   **[orm]**

    现在通过 relation() 的 “collection_class” 关键字参数来实现“自定义列表类”，旧方式仍然有效但已弃用

    参考：[#212](https://www.sqlalchemy.org/trac/ticket/212)

+   **[orm]**

    向 relation() 添加了 “viewonly” 标志，允许构建对 flush() 过程没有影响的关系。

+   **[orm]**

    向基本 Query select/get 函数添加了 “lockmode” 参数，包括 “with_lockmode” 函数以获取具有默认锁定模式的 Query 副本。将“read”/“update”参数转换为 select 方面的 for_update 参数。

    参考：[#292](https://www.sqlalchemy.org/trac/ticket/292)

+   **[orm]**

    在 Query/Mapper 中实现了“版本检查”逻辑，在 version_id_col 生效并使用 query.with_lockmode() 来获取已加载实例时使用

+   **[orm]**

    post_update 行为改进；在更新太多行时表现更好，仅更新必需的列

    参考：[#208](https://www.sqlalchemy.org/trac/ticket/208)

+   **[orm]**

    调整急切加载，使其“急切链”与正常映射器设置分开，从而防止与惰性加载器操作冲突，修复

    参考：[#308](https://www.sqlalchemy.org/trac/ticket/308)

+   **[orm]**

    修复延迟组加载

+   **[orm]**

    session.flush() 不会关闭它打开的连接

    参考：[#346](https://www.sqlalchemy.org/trac/ticket/346)

+   **[orm]**

    向映射器添加了“batch=True”标志；如果为 False，save_obj 将完全逐个保存一个对象，包括对 before_XXXX 和 after_XXXX 的调用

+   **[orm]**

    向 mapper 添加了“column_prefix=None”参数；自动将给定字符串（通常为‘_’）前置到从 mapper 的 Table 设置的列属性中。

+   **[orm]**

    在 query.select()的 from_obj 参数中指定连接将替换查询的主表，如果表在给定的 from_obj 中的某处。这使得在查询中产生自定义连接和外连接而不会使主表被添加两次成为可能。

    参考：[#315](https://www.sqlalchemy.org/trac/ticket/315)

+   **[orm]**

    eagerloading 被调整为更加周到地将其 LEFT OUTER JOIN 附加到给定查询中，查找可能已经设置的自定义“FROM”子句。

+   **[orm]**

    向 SelectResults 添加了 join_to 和 outerjoin_to 转换方法，根据属性名称构建连接/外连接条件。还添加了 select_from 以明确设置 from_obj 参数。

+   **[orm]**

    从 mapper 中移除了“is_primary”标志。

### sql

+   **[sql] [construction]**

    将“for_update”参数更改为接受 False/True/“nowait”和“read”，后两者仅由 Oracle 和 MySQL 解释。

    参考：[#292](https://www.sqlalchemy.org/trac/ticket/292)

+   **[sql] [construction]**

    向 sql 方言添加了 extract()函数（SELECT extract(field FROM expr)）

+   **[sql] [construction]**

    BooleanExpression 包括新的“negate”参数，以指定适当的否定运算符（如果有的话）。

+   **[sql] [construction]**

    对“IN”或“IS”子句进行否定将导致“NOT IN”，“IS NOT”（而不是 NOT (x IN y)）。

+   **[sql] [construction]**

    现在函数对象知道如何在 FROM 子句中操作。它们的行为应该是相同的，只是现在你还可以做一些像 select([‘*’], from_obj=[func.my_function()])这样的事情，以从结果中获取多个列，甚至使用 sql.column()构造来命名返回的列。

    参考：[#172](https://www.sqlalchemy.org/trac/ticket/172)

### schema

+   **[schema]**

    对 schema 包进行了相当多的清理，删除了模糊的方法，不再需要的方法。稍微更受限制的使用，更加强调明确性

+   **[schema]**

    Table 和其他可选择对象的“primary_key”属性变成了一个类似集合的 ColumnCollection 对象；它是有序的，但没有数值索引。从相同基础表派生的两个主键之间的比较子句（即两个 Alias 对象之间）可以通过 table1.primary_key==table2.primary_key 生成。

+   **[schema]**

    ForeignKey(Constraint)支持“use_alter=True”，通过 ALTER 创建/删除外键。这允许设置循环外键关系。

+   **[schema]**

    从 Table 和 Column 中删除了 append_item()方法；最好是内联构造 Table/Column/相关对象，但如果需要，可以使用 append_column()、append_foreign_key()、append_constraint()等方法。

+   **[schema]**

    table.create() 不再返回 Table 对象，而是没有返回值。通常情况下，表是通过 metadata 创建的，这是更好的方式，因为它会处理表的依赖关系。

+   **[模式]**

    添加了 UniqueConstraint（在 Table 级别）、CheckConstraint（在 Table 或 Column 级别）。

+   **[模式]**

    Column 上的 index=False/unique=True 现在创建一个 UniqueConstraint，index=True/unique=False 创建一个普通的 Index，index=True/unique=True 在 Column 上创建一个唯一的 Index。column 上的 ‘index’ 和 ‘unique’ 关键字参数现在仅为布尔值；对于索引或唯一约束的显式名称和分组，请显式使用 UniqueConstraint/Index 构造。

+   **[模式]**

    将 autoincrement=True 添加到 Column；如果明确设置为 False，则会禁用 postgres/mysql/mssql 的 SERIAL/AUTO_INCREMENT/identity 序列的模式生成

+   **[模式]**

    TypeEngine 对象现在具有处理其特定类型的值的方法。目前被 ORM 使用，参见下文。

+   **[模式]**

    修复了反射时当主键列被显式覆盖时出现的条件，其中 PrimaryKeyConstraint 会重复获取反映和编程列。

+   **[模式]**

    在 Column 和 ColumnElement 上的“foreign_key” 属性已被弃用，而是采用了“foreign_keys” 基于列表/集合的属性，该属性考虑了一个列上的多个外键。“foreign_key” 将返回“foreign_keys” 列表/集合中的第一个元素，如果列表为空则返回 None。

### sqlite

+   **[sqlite]**

    sqlite 布尔数据类型默认将 False/True 转换为 0/1

+   **[sqlite]**

    修复了 Date/Time（SLDate/SLTime）类型的问题；现在与 postgres 一样好用

    参考：[#335](https://www.sqlalchemy.org/trac/ticket/335)

### oracle

+   **[oracle]**

    Oracle 对 cx_Oracle.TIMESTAMP 有试验性支持，这需要对现在通过 oracle 方言的游标进行 setinputsizes() 调用，现在通过 ‘auto_setinputsizes’ 标志启用。

### 杂项

+   **[ms-sql]**

    修复了 bug 261（对于区分大小写的数据库，表反射失效的问题）

+   **[ms-sql]**

    现在可以为 pymssql 指定端口

+   **[ms-sql]**

    引入了新的 “auto_identity_insert” 选项，用于在为 IDENTITY 列指定值时自动切换到 “SET IDENTITY_INSERT” 模式

+   **[ms-sql]**

    现在支持多列外键

+   **[ms-sql]**

    修复了反射日期/时间列的问题

+   **[ms-sql]**

    添加了 NCHAR 和 NVARCHAR 类型支持

+   **[firebird]**

    别名不使用 “AS”

+   **[firebird]**

    在反映不存在的表时，正确引发 NoSuchTableError

+   **[连接/池/执行]**

    连接池跟踪打开的游标，并在连接返回到仍然打开游标的池时自动关闭它们。可以受到导致其引发错误或不执行任何操作的选项的影响。修复了与 MySQL 等的问题

+   **[连接/池/执行]**

    修复了连接在提交/回滚后不会丢失其事务的错误。

+   **[连接/池/执行]**

    ComposedSQLEngine、ResultProxy 现在添加了 scalar() 方法。

+   **[connections/pooling/execution]**

    ResultProxy 在其自身关闭时将关闭底层游标。当所有行都已提取（或已调用 scalar()）的 ResultProxy 对象将自动关闭游标。

+   **[connections/pooling/execution]**

    ResultProxy.fetchall() 在内部使用 DBAPI fetchall() 实现更好的效率，也添加到 mapper 迭代中（感谢 Michael Twomey）。

## 0.3.11

发布日期：2007 年 10 月 14 日（星期日）

### orm

+   **[orm]**

    添加了一个检查，用 join() 从 A->B 连接时，同时使用两个不同的 m2m 表。这在 0.3 中会引发错误，但在使用别名时在 0.4 中是可能的。

    参考：[#687](https://www.sqlalchemy.org/trac/ticket/687)

+   **[orm]**

    修复了 Session.merge() 中的小异常抛出 bug。

+   **[orm]**

    修复了映射器的一个 bug，该 bug 与一个表没有 PK 列的联接有关，映射器未检测到联接的表没有 PK。

+   **[orm]**

    修复了从自定义继承条件确定适当的同步子句的 bug。

    参考：[#769](https://www.sqlalchemy.org/trac/ticket/769)

+   **[orm]**

    如果另一侧集合不包含该项，则 backref 移除对象操作不会失败，支持 noload 集合。

    参考：[#813](https://www.sqlalchemy.org/trac/ticket/813)

### engine

+   **[engine]**

    修复了另一个偶发性竞态条件，在使用带 threadlocal 设置的池时可能会发生。

### sql

+   **[sql]**

    调整了像 func.count(t.c.col.distinct()) 这样的 DISTINCT 优先级。

+   **[sql]**

    修复了在 :bind$params 中检测内部“$”字符的 bug。

    参考：[#719](https://www.sqlalchemy.org/trac/ticket/719)

+   **[sql]**

    不要假设连接条件仅包含列对象。

    参考：[#768](https://www.sqlalchemy.org/trac/ticket/768)

+   **[sql]**

    调整了 NOT 的运算符优先级，使其与 ‘==’ 和其他运算符匹配，因此 ~(x==y) 产生 NOT (x=y)，这与 MySQL < 5.0 兼容（不喜欢 “NOT x=y”）。

    参考：[#764](https://www.sqlalchemy.org/trac/ticket/764)

### mysql

+   **[mysql]**

    生成模式时修正了 YEAR 列的规范。

### sqlite

+   **[sqlite]**

    日期字符串的传递

### mssql

+   **[mssql]**

    添加了对 TIME 列（使用 DATETIME 模拟）的支持。

    参考：[#679](https://www.sqlalchemy.org/trac/ticket/679)

+   **[mssql]**

    添加了对 BIGINT、MONEY、SMALLMONEY、UNIQUEIDENTIFIER 和 SQL_VARIANT 的支持。

    参考：[#721](https://www.sqlalchemy.org/trac/ticket/721)

+   **[mssql]**

    删除反射表时现在会引用索引名称。

    参考：[#684](https://www.sqlalchemy.org/trac/ticket/684)

+   **[mssql]**

    现在可以为 PyODBC 指定 DSN，使用类似 mssql:///?dsn=bob 的 URI。

### oracle

+   **[oracle]**

    从“binary”类型中移除 LONG_STRING、LONG_BINARY，因此类型对象不会尝试将其值读取为 LOB。

    参考：[#622](https://www.sqlalchemy.org/trac/ticket/622)，[#751](https://www.sqlalchemy.org/trac/ticket/751)

### 杂项

+   **[postgres]**

    当从备用模式中反映表时，“默认”放置在主键上的内容，即通常是序列名称，无条件地引用了 “schema” 名称，因此需要引用的模式名称是正确的。对于不需要引用的模式名称来说，这有点多余但不会有害。

+   **[firebird]**

    由于票证 #370（正确的方式），supports_sane_rowcount() 设置为 False。

+   **[firebird]**

    修复了 Column 的 nullable 属性的反射。

### orm

+   **[orm]**

    添加了一个检查，用于使用 join() 从 A->B 进行连接，沿两个不同的 m2m 表。这在 0.3 中引发错误，但在 0.4 中使用别名时是可能的。

    参考：[#687](https://www.sqlalchemy.org/trac/ticket/687)

+   **[orm]**

    在 Session.merge() 中修复了一个小的异常抛出错误。

+   **[orm]**

    修复了一个错误，其中 mapper 与一个表连接在一起，该表没有主键列，但不会检测到连接表没有主键。

+   **[orm]**

    修复了从自定义继承条件中确定正确同步子句的错误。

    参考：[#769](https://www.sqlalchemy.org/trac/ticket/769)

+   **[orm]**

    如果另一侧的集合不包含该项，则 backref 移除对象操作不会失败，支持 noload 集合。

    参考：[#813](https://www.sqlalchemy.org/trac/ticket/813)

### engine

+   **[engine]**

    修复了在使用带有 threadlocal 设置的池时可能发生的另一个偶发竞争条件。

### sql

+   **[sql]**

    调整了对像 func.count(t.c.col.distinct()) 这样的子句的 DISTINCT 优先级。

+   **[sql]**

    修复了对 :bind$params 中内部 ‘$’ 字符的检测。

    参考：[#719](https://www.sqlalchemy.org/trac/ticket/719)

+   **[sql]**

    不再假设连接标准仅由列对象组成。

    参考：[#768](https://www.sqlalchemy.org/trac/ticket/768)

+   **[sql]**

    调整了 NOT 运算符的优先级，使其与 ‘==’ 和其他运算符匹配，以便 ~(x==y) 产生 NOT (x=y)，这与 MySQL < 5.0 兼容（不喜欢 “NOT x=y”）。

    参考：[#764](https://www.sqlalchemy.org/trac/ticket/764)

### mysql

+   **[mysql]**

    在生成模式时修复了 YEAR 列的规范。

### sqlite

+   **[sqlite]**

    对字符串化的日期进行了穿透处理。

### mssql

+   **[mssql]**

    添加了对 TIME 列（使用 DATETIME 模拟）的支持。

    参考：[#679](https://www.sqlalchemy.org/trac/ticket/679)

+   **[mssql]**

    添加了对 BIGINT、MONEY、SMALLMONEY、UNIQUEIDENTIFIER 和 SQL_VARIANT 的支持。

    参考：[#721](https://www.sqlalchemy.org/trac/ticket/721)

+   **[mssql]**

    从反映的表中删除索引名称时现在会引用它们。

    参考：[#684](https://www.sqlalchemy.org/trac/ticket/684)

+   **[mssql]**

    现在可以为 PyODBC 指定 DSN，使用类似 mssql:///?dsn=bob 的 URI。

### oracle

+   **[oracle]**

    从 “binary” 类型中删除了 LONG_STRING、LONG_BINARY，因此类型对象不会尝试将其值读取为 LOB。

    参考：[#622](https://www.sqlalchemy.org/trac/ticket/622), [#751](https://www.sqlalchemy.org/trac/ticket/751)

### 杂项

+   **[postgres]**

    从替代模式反射表时，放在主键上的“默认”（通常是序列名称）无条件地引用了“模式”名称，因此需要引用的模式名称是可以的。对于不需要引用的模式名称来说，这有点多余但不会有害。

+   **[firebird]**

    supports_sane_rowcount()设置为 False，因为 ticket＃370（正确方式）。

+   **[firebird]**

    修复了 Column 的 nullable 属性的反射。

## 0.3.10

发布日期：2007 年 7 月 20 日星期五

### 通用

+   **[general]**

    在 0.3.9 中添加的新互斥锁导致在竞争条件下 pool_timeout 功能失败；如果许多线程同时将池推入溢出，线程将立即引发 TimeoutError，没有延迟。此问题已得到解决。

### orm

+   **[orm]**

    清理连接绑定会话，SessionTransaction

### sql

+   **[sql]**

    使连接绑定的元数据与隐式执行一起工作

+   **[sql]**

    外键规范可以在其标识符中包含任何字符

    参考：[#667](https://www.sqlalchemy.org/trac/ticket/667)

+   **[sql]**

    向二进制子句比较添加了可交换性意识，改进了 ORM 延迟加载优化

    参考：[#664](https://www.sqlalchemy.org/trac/ticket/664)

### 杂项

+   **[postgres]**

    修复了最大标识符长度（63）

    参考：[#571](https://www.sqlalchemy.org/trac/ticket/571)

### 通用

+   **[general]**

    在 0.3.9 中添加的新互斥锁导致在竞争条件下 pool_timeout 功能失败；如果许多线程同时将池推入溢出，线程将立即引发 TimeoutError，没有延迟。此问题已得到解决。

### orm

+   **[orm]**

    清理连接绑定会话，SessionTransaction

### sql

+   **[sql]**

    使连接绑定的元数据与隐式执行一起工作

+   **[sql]**

    外键规范可以在其标识符中包含任何字符

    参考：[#667](https://www.sqlalchemy.org/trac/ticket/667)

+   **[sql]**

    向二进制子句比较添加了可交换性意识，改进了 ORM 延迟加载优化

    参考：[#664](https://www.sqlalchemy.org/trac/ticket/664)

### 杂项

+   **[postgres]**

    修复了最大标识符长度（63）

    参考：[#571](https://www.sqlalchemy.org/trac/ticket/571)

## 0.3.9

发布日期：2007 年 7 月 15 日星期日

### 通用

+   **[general]**

    NoSuchColumnError 的更好错误消息

    参考：[#607](https://www.sqlalchemy.org/trac/ticket/607)

+   **[general]**

    最终弄清楚了如何将 setuptools 版本引入，可作为 sqlalchemy.__version__ 使用

    参考：[#428](https://www.sqlalchemy.org/trac/ticket/428)

+   **[general]**

    各种“engine”参数，如“engine”，“connectable”，“engine_or_url”，“bind_to”等都存在，但已弃用。它们都被单个术语“bind”取代。您还可以使用 metadata.bind = <engine or connection>设置 MetaData 的“bind”。

### orm

+   **[orm]**

    与 0.4 向前兼容：向 Query 添加了 one()，first()和 all()。几乎所有 0.4 中的 Query 功能在 0.3.9 中都存在，以实现向前兼容。

+   **[orm]**

    reset_joinpoint()这次真的真的有效，保证！允许您从根重新加入：query.join([‘a’, ‘b’]).filter(<crit>).reset_joinpoint().join([‘a’, ‘c’]).filter(<some other crit>).all()在 0.4 中，所有的 join()调用都从“root”开始。

+   **[orm]**

    在 mapper()构造步骤中添加了同步，以避免在不同线程中编译预先存在的 mapper 时出现线程冲突

    参考：[#613](https://www.sqlalchemy.org/trac/ticket/613)

+   **[orm]**

    当两个相同名称的主键列被混合成一个属性时，Mapper 会发出警告。这在映射到联接（或继承）时经常发生。

+   **[orm]**

    synonym()属性现已完全受到所有 Query joining/ with_parent 操作的支持。

    参考：[#598](https://www.sqlalchemy.org/trac/ticket/598)

+   **[orm]**

    修复了使用 uselist=False 关系删除项目时的非常愚蠢的错误。

+   **[orm]**

    还记得关于多态联合的所有东西吗？用于连接表继承？有趣的事情是...对于连接表继承，你基本上不需要它，你可以通过 outerjoin()将所有表串在一起。然而，如果涉及具体表，则 UNION 仍然适用（因为没有东西可以加入它们）。

+   **[orm]**

    对贪婪加载进行了小修复，以更好地与使用直接“outerjoin”子句的多态映射器的贪婪加载配合使用。

### sql

+   **[sql]**

    对不是默认模式的表的外键需要明确指定模式；即 ForeignKey(‘alt_schema.users.id’)

+   **[sql]**

    MetaData 现在可以用引擎或 url 作为第一个参数来构造，就像 BoundMetaData 一样。

+   **[sql]**

    BoundMetaData 现已弃用，并且 MetaData 是一个直接的替代品。

+   **[sql]**

    DynamicMetaData 已更名为 ThreadLocalMetaData。DynamicMetaData 名称已弃用，并且是 ThreadLocalMetaData 或者当 threadlocal=False 时是常规 MetaData 的别名。

+   **[sql]**

    复合主键被表示为非键集，以允许由具有相同名称的列组成的复合键；在 Join 中发生。帮助继承方案制定正确的 PK。

+   **[sql]**

    提高了从联接中获取“正确”的和最小化的主键列集的能力，将外键和其他等值列等效。这也主要是为了帮助继承方案制定最佳选择的主键列。

    参考：[#185](https://www.sqlalchemy.org/trac/ticket/185)

+   **[sql]**

    在 Sequence.create()/drop()，ColumnDefault.execute()中添加了‘bind’参数。

+   **[sql]**

    列可以使用与列名不同的“key”属性在反射表中被重写，包括主键列

    参考：[#650](https://www.sqlalchemy.org/trac/ticket/650)

+   **[sql]**

    修复了在结果中存在重复列名时“模棱两可的列”结果检测的问题。

    参考：[#657](https://www.sqlalchemy.org/trac/ticket/657)

+   **[sql]**

    对“列定位”进行了一些增强，即将一列与另一个可选择的“对应”列进行匹配的能力。这主要影响 ORM 映射到复杂连接的能力

+   **[sql]**

    MetaData 和所有 SchemaItems 都可以安全地与 pickle 一起使用。可以将缓慢的表反射转储到一个 pickled 文件中以供以后重用。只需在解 pickle 后重新连接到元数据。

    参考：[#619](https://www.sqlalchemy.org/trac/ticket/619)

+   **[sql]**

    在 QueuePool 的“overflow”计算中添加了互斥锁，以防止绕过 max_overflow 的竞争条件

+   **[sql]**

    修复了复合选择的分组，以便给出正确的结果。在某些情况下会在 sqlite 上出现问题，但这些情况本来就会产生不正确的结果，sqlite 不支持分组的复合选择

    参考：[#623](https://www.sqlalchemy.org/trac/ticket/623)

+   **[sql]**

    修复了运算符的优先级，以便正确��用括号

    参考：[#620](https://www.sqlalchemy.org/trac/ticket/620)

+   **[sql]**

    调用<column>.in_()（即不带参数）将返回“CASE WHEN (<column> IS NULL) THEN NULL ELSE 0 END = 1”，以便在所有情况下返回 NULL 或 False，而不是抛出错误

    参考：[#545](https://www.sqlalchemy.org/trac/ticket/545)

+   **[sql]**

    修复了 select()的“where”/“from”条件，以接受 unicode 字符串以及常规字符串 - 两者都转换为 text()

+   **[sql]**

    添加了独立的 distinct()函数，以补充 column.distinct()

    参考：[#558](https://www.sqlalchemy.org/trac/ticket/558)

+   **[sql]**

    result.last_inserted_ids()应返回一个与表的主键约束大小相同的列表。通过“被动”创建而不通过 cursor.lastrowid 可用的值将为 None。

+   **[sql]**

    修复了长标识符检测，使用>而不是>=来确定最大标识符长度

    参考：[#589](https://www.sqlalchemy.org/trac/ticket/589)

+   **[sql]**

    修复了一个错误，即 selectable.corresponding_column(selectable.c.col)如果 selectable 是一个表和另一个涉及相同表的连接的连接，则不会返回 selectable.c.col。搞乱了 ORM 的决策

    参考：[#593](https://www.sqlalchemy.org/trac/ticket/593)

+   **[sql]**

    在 types.py 中添加了 Interval 类型

    参考：[#595](https://www.sqlalchemy.org/trac/ticket/595)

### mysql

+   **[mysql]**

    修复了捕获一些暗示连接中断的错误

    参考：[#625](https://www.sqlalchemy.org/trac/ticket/625)

+   **[mysql]**

    修复了百分号运算符的转义

    参考：[#624](https://www.sqlalchemy.org/trac/ticket/624)

+   **[mysql]**

    添加了对“fields”保留字的支持

    参考：[#590](https://www.sqlalchemy.org/trac/ticket/590)

+   **[mysql]**

    各种反射增强/修复

### sqlite

+   **[sqlite]**

    重新排列了方言初始化，以便有时间警告 pysqlite1 太旧。

+   **[sqlite]**

    sqlite 更好地处理混合使用各种 Date/Time/DateTime 列的日期时间对象

+   **[sqlite]**

    字符串 PK 列插入不会被 OID 覆盖

    参考：[#603](https://www.sqlalchemy.org/trac/ticket/603)

### mssql

+   **[mssql]**

    修复了 pyodbc 的端口选项处理

    参考：[#634](https://www.sqlalchemy.org/trac/ticket/634)

+   **[mssql]**

    现在能够反射标识列的起始和增量值

+   **[mssql]**

    与 pyodbc 一起使用 scope_identity()的初步支持

### oracle

+   **[oracle]**

    日期时间修复：使亚秒级 TIMESTAMP 正常工作，添加支持仅具有年/月/日的 types.Date 的 OracleDate

    参考：[#604](https://www.sqlalchemy.org/trac/ticket/604)

+   **[oracle]**

    添加方言标志“auto_convert_lobs”，默认为 True；将导致在结果集中检测到的任何 LOB 对象被强制转换为 OracleBinary，以便 LOB 自动读取，如果没有 typemap 存在（即，如果发出了文本执行()）。

+   **[oracle]**

    mod 运算符‘%’产生 MOD

    参考：[#624](https://www.sqlalchemy.org/trac/ticket/624)

+   **[oracle]**

    在使用 Python 2.3 时，将 cx_oracle 日期时间对象转换为 Python datetime.datetime

    参考：[#542](https://www.sqlalchemy.org/trac/ticket/542)

+   **[oracle]**

    修复了 Oracle TEXT 类型中的 Unicode 转换

### 杂项

+   **[ext]**

    遍历字典关联代理现在类似于字典，而不是类似于 InstrumentedList（例如，通过键而不是值）

+   **[ext]**

    关联代理不再紧密绑定到源集合，并且使用 thunk 构造

    参考：[#597](https://www.sqlalchemy.org/trac/ticket/597)

+   **[ext]**

    添加了 selectone_by()以分配映射器

+   **[postgres]**

    修复了百分号运算符的转义

    参考：[#624](https://www.sqlalchemy.org/trac/ticket/624)

+   **[postgres]**

    添加了对域反射的支持

    参考：[#570](https://www.sqlalchemy.org/trac/ticket/570)

+   **[postgres]**

    反射期间缺失的类型解析为 Null 类型，而不是引发错误

+   **[postgres]**

    上面“schema”中的修复解决了从 alt-schema 表反射到 public schema 表的外键反射

### general

+   **[general]**

    NoSuchColumnError 的更好错误消息

    参考：[#607](https://www.sqlalchemy.org/trac/ticket/607)

+   **[general]**

    最终找到了如何获取 setuptools 版本，可作为 sqlalchemy.__version__ 使用

    参考：[#428](https://www.sqlalchemy.org/trac/ticket/428)

+   **[general]**

    各种“engine”参数，如“engine”，“connectable”，“engine_or_url”，“bind_to”等都存在，但已弃用。它们都被单个术语“bind”取代。您还可以使用 metadata.bind = <engine or connection>设置 MetaData 的“bind”

### orm

+   **[orm]**

    与 0.4 的向前兼容：向 Query 添加了 one()，first()和 all()。几乎所有 0.4 中的 Query 功能在 0.3.9 中都存在，以实现向前兼容。

+   **[orm]**

    reset_joinpoint()这次真的有效，承诺！允许您从根重新加入：query.join(['a', 'b']).filter(<crit>).reset_joinpoint().join(['a', 'c']).filter(<some other crit>).all()在 0.4 中，所有 join()调用都从“root”开始

+   **[orm]**

    在 mapper() 构造步骤中添加了同步，以避免在不同线程中编译预先存在的映射器时发生线程冲突

    参考：[#613](https://www.sqlalchemy.org/trac/ticket/613)

+   **[orm]**

    当 Mapper 将两个同名主键列混合成一个属性时，会发出警告。这在映射到连接（或继承）时经常发生。

+   **[orm]**

    synonym() 属性完全支持所有 Query 的 joining/with_parent 操作

    参考：[#598](https://www.sqlalchemy.org/trac/ticket/598)

+   **[orm]**

    修复了在删除具有 many-to-many uselist=False 关系的项目时的非常愚蠢的错误

+   **[orm]**

    还记得关于 polymorphic_union 的那些东西吗？用于 joined table inheritance？有趣的是… 对于 joined table inheritance，你实际上不需要它，你可以通过 outerjoin() 将所有表串在一起。但如果涉及具体表，则 UNION 仍然适用（因为没有东西可以将它们连接起来）。

+   **[orm]**

    对急加载进行了小修复，以更好地与使用直接“outerjoin”子句的多态映射器的急加载配合使用

### sql

+   **[sql]**

    对于不是默认模式的表的 ForeignKey 需要显式指定模式；即 ForeignKey(‘alt_schema.users.id’)

+   **[sql]**

    MetaData 现在可以用引擎或 url 作为第一个参数构建，就像 BoundMetaData 一样

+   **[sql]**

    BoundMetaData 现在已被弃用，MetaData 是直接替代品。

+   **[sql]**

    DynamicMetaData 已更名为 ThreadLocalMetaData。DynamicMetaData 名称已被弃用，并且是 ThreadLocalMetaData 或者如果 threadlocal=False 则是常规 MetaData 的别名

+   **[sql]**

    复合主键表示为非键集，以允许由具有相同名称的列组成的复合键；出现在 Join 中。有助于继承场景制定正确的 PK。

+   **[sql]**

    改进了从连接中获取“正确”和最小一组主键列的能力，将外键和其他等效列等同起来。这主要是为了帮助继承场景制定最佳选择的主键列。

    参考：[#185](https://www.sqlalchemy.org/trac/ticket/185)

+   **[sql]**

    在 Sequence.create()/drop()、ColumnDefault.execute() 中添加了 ‘bind’ 参数

+   **[sql]**

    可以使用与列名不同的“key”属性在反射表中覆盖列，包括主键列

    参考：[#650](https://www.sqlalchemy.org/trac/ticket/650)

+   **[sql]**

    修复了“模糊列”结果检测，当结果中存在重复列名时

    参考：[#657](https://www.sqlalchemy.org/trac/ticket/657)

+   **[sql]**

    对“列定位”的一些增强，即将一列与另一个可选择的“对应”列匹配的能力。这主要影响 ORM 映射到复杂连接的能力

+   **[sql]**

    MetaData 和所有 SchemaItems 都可以与 pickle 一起使用。可以将缓慢的表反射转储到 pickle 文件中以供以后重用。只需在解除 pickle 后重新连接引擎到元数据。

    参考：[#619](https://www.sqlalchemy.org/trac/ticket/619)

+   **[sql]**

    向 QueuePool 的“overflow”计算添加了互斥锁，以防止绕过 max_overflow 的竞争条件。

+   **[sql]**

    修复了复合选择的分组以提供正确的结果。在某些情况下，这将在 sqlite 上中断，但是这些情况无论如何都会产生不正确的结果，sqlite 不支持分组的复合选择。

    参考：[#623](https://www.sqlalchemy.org/trac/ticket/623)

+   **[sql]**

    修复了操作符的优先级，以便正确应用括号

    参考：[#620](https://www.sqlalchemy.org/trac/ticket/620)

+   **[sql]**

    调用<column>.in_()（即不带参数）将返回“CASE WHEN（<column> IS NULL）THEN NULL ELSE 0 END = 1”，以便在所有情况下返回 NULL 或 False，而不是抛出错误。

    参考：[#545](https://www.sqlalchemy.org/trac/ticket/545)

+   **[sql]**

    修复了 select()的“where”/“from”条件，以接受 unicode 字符串以及常规字符串 - 两者都转换为 text()。

+   **[sql]**

    除了 column.distinct()之外，还添加了独立的 distinct()函数。

    参考：[#558](https://www.sqlalchemy.org/trac/ticket/558)

+   **[sql]**

    result.last_inserted_ids()应返回与表的主键约束大小相同的列表。通过“被动”创建并且不可通过 cursor.lastrowid 获得的值将为 None。

+   **[sql]**

    长标识符检测修复为使用>而不是>=用于最大标识符长度

    参考：[#589](https://www.sqlalchemy.org/trac/ticket/589)

+   **[sql]**

    修复了 selectable.corresponding_column(selectable.c.col)不返回 selectable.c.col 的错误，如果 selectable 是一个表和涉及相同表的另一个连接的连接。混乱的 ORM 决策制定

    参考：[#593](https://www.sqlalchemy.org/trac/ticket/593)

+   **[sql]**

    在 types.py 中添加了 Interval 类型

    参考：[#595](https://www.sqlalchemy.org/trac/ticket/595)

### mysql

+   **[mysql]**

    修复了捕获一些暗示连接已断开的错误

    参考：[#625](https://www.sqlalchemy.org/trac/ticket/625)

+   **[mysql]**

    修复了模运算符的转义

    参考：[#624](https://www.sqlalchemy.org/trac/ticket/624)

+   **[mysql]**

    将‘fields’添加到保留字中

    参考：[#590](https://www.sqlalchemy.org/trac/ticket/590)

+   **[mysql]**

    各种反射增强/修复

### sqlite

+   **[sqlite]**

    重新排列方言初始化，以便及时警告 pysqlite1 过旧。

+   **[sqlite]**

    sqlite 更好地处理混合匹配各种日期/时间/日期时间列的日期时间对象

+   **[sqlite]**

    字符串 PK 列插入不会被 OID 覆盖

    参考：[#603](https://www.sqlalchemy.org/trac/ticket/603)

### mssql

+   **[mssql]**

    修复了 pyodbc 的端口选项处理

    参考：[#634](https://www.sqlalchemy.org/trac/ticket/634)

+   **[mssql]**

    现在能够反射标识列的起始和增量值

+   **[mssql]**

    使用 pyodbc 预备支持使用 scope_identity()方法

### oracle

+   **[oracle]**

    日期时间修复：使亚秒级别的时间戳工作，添加了支持仅具有年/月/日的 types.Date 的 OracleDate

    参考：[#604](https://www.sqlalchemy.org/trac/ticket/604)

+   **[oracle]**

    添加了 dialect 标志“auto_convert_lobs”，默认为 True；将导致在结果集中检测到任何 LOB 对象都被强制转换为 OracleBinary，以便 LOB 自动读取，如果没有 typemap 存在（即，如果发出了文本执行()）。

+   **[oracle]**

    mod 运算符‘%’产生 MOD

    参考：[#624](https://www.sqlalchemy.org/trac/ticket/624)

+   **[oracle]**

    当使用 Python 2.3 时，将 cx_oracle datetime 对象转换为 Python datetime.datetime

    参考：[#542](https://www.sqlalchemy.org/trac/ticket/542)

+   **[oracle]**

    修复了 Oracle TEXT 类型中的 Unicode 转换

### 杂项

+   **[ext]**

    现在对字典关联代理进行迭代类似于字典，而不是 InstrumentedList（例如，通过键而不是值）

+   **[ext]**

    关联代理不再紧密绑定到源集合，并且使用 thunk 构造

    参考：[#597](https://www.sqlalchemy.org/trac/ticket/597)

+   **[ext]**

    添加了 selectone_by()方法到 assignmapper

+   **[postgres]**

    修复了百分号运算符的转义

    参考：[#624](https://www.sqlalchemy.org/trac/ticket/624)

+   **[postgres]**

    添加了对域的反射支持

    参考：[#570](https://www.sqlalchemy.org/trac/ticket/570)

+   **[postgres]**

    在反射期间缺少的类型将解析为 Null 类型，而不是引发错误

+   **[postgres]**

    上面“schema”中的修复修复了从 alt-schema 表反射到 public schema 表的外键反射

## 0.3.8

发布日期：2007 年 6 月 2 日星期六

### orm

+   **[orm]**

    在 Query 中添加了 reset_joinpoint()方法，将“连接点”移回起始映射器。0.4 版本将更改 join()方法的行为，以在所有情况下重置“连接点”，因此这是一个临时方法。为了向前兼容，请确保跨多个关系指定的连接使用单个 join()方法，即 join(['a', 'b', 'c'])。

+   **[orm]**

    修复了 query.instances()方法中的一个 bug，该 bug 无法处理多个额外的映射器或一个额外的列。

+   **[orm]**

    “delete-orphan”不再意味着“delete”。持续努力分离这两个操作的行为。

+   **[orm]**

    多对多关系正确设置了关联表上删除操作的绑定参数类型

+   **[orm]**

    多对多关系检查删除操作从关联表中删除的行数是否符合预期结果

+   **[orm]**

    session.get()和 session.load()方法将**kwargs 传播到查询中

+   **[orm]**

    修复了允许原始多态联合嵌入到相关子查询中的多态查询

    参考：[#577](https://www.sqlalchemy.org/trac/ticket/577)

+   **[orm]**

    ��复了在与多对多关系一起使用 select_by(<propname>=<object instance>) -style joins 的 bug，引入于 r2556

+   **[orm]**

    mapper()的“primary_key”参数传播到“polymorphic”映射器。此列表中的主键列被规范化为映射器的本地表的主键。

+   **[orm]**

    恢复了在 sa.orm.strategies 记录器下记录“延迟加载子句”的日志，在 0.3.7 中被移除

+   **[orm]**

    改进了对映射到 select()语句的属性的预加载支持；即 eagerloader 更善于定位正确的可选择项以附加其 LEFT OUTER JOIN。

### sql

+   **[sql]**

    _Label 类覆盖 compare_self 以返回其最终对象。意味着，如果你说 someexpr.label(‘foo’) == 5，它会产生正确的“someexpr == 5”。

+   **[sql]**

    _Label 传播“_hide_froms()”，使标量选择在 FROM 子句方面表现更正常 #574

+   **[sql]**

    修复了在使用 oid_column 作为排序依据时生成过长名称的问题（oids 在映射器查询中被大量使用）

+   **[sql]**

    ResultProxy 的显著速度提升，预先缓存 TypeEngine 方言实现，并节省每列的函数调用

+   **[sql]**

    通过新的 _Grouping 构造将括号应用于子句。使用运算符优先级更智能地将括号应用于子句，提供更清晰的子句嵌套（不会改变放置在其他子句中的子句，即没有‘parens’标志）

+   **[sql]**

    添加了‘modifier’关键字，类似于 func.<foo>，但不添加括号。例如 select([modifier.DISTINCT(…)]) 等。

+   **[sql]**

    移除了“在 UNION 的一部分的 select 中没有 group by”的限制

    参考：[#578](https://www.sqlalchemy.org/trac/ticket/578)

### mysql

+   **[mysql]**

    现在几乎支持所有 MySQL 列类型的声明和反射。添加了 NCHAR、NVARCHAR、VARBINARY、TINYBLOB、LONGBLOB、YEAR

+   **[mysql]**

    sqltypes.Binary 透传现在始终构建一个 BLOB，避免与非常旧的数据库版本的问题

+   **[mysql]**

    支持列级别的 CHARACTER SET 和 COLLATE 声明，以及 ASCII、UNICODE、NATIONAL 和 BINARY 简写。

### 杂项

+   **[engines]**

    添加了 detach()到 Connection，允许底层 DBAPI 连接从其池中分离，在取消引用/关闭()时关闭而不是被池重用。

+   **[engines]**

    添加了 invalidate()到 Connection，立即使 Connection 及其底层 DBAPI 连接无效。

+   **[firebird]**

    将最大标识符长度设置为 31

+   **[firebird]**

    由于票号#370，supports_sane_rowcount()设置为 False。versioned_id_col 功能在 FB 中不起作用。

+   **[firebird]**

    一些执行修复

+   **[firebird]**

    新的关联代理实现，实现了完整的代理到基于列表、字典和集合的关系集合

+   **[firebird]**

    添加了 orderinglist，一个自定义列表类，将对象属性与该对象在列表中的位置同步

+   **[firebird]**

    对 SelectResultsExt 进行了小修复，以避免在 select()期间绕过自身。

+   **[firebird]**

    为 assignmapper 添加了 filter()、filter_by()

### orm

+   **[orm]**

    为 Query 添加了 reset_joinpoint()方法，将“join point”移回起始映射器。0.4 版本将更改 join()的行为，以在所有情况下重置“join point”。为了向前兼容，请确保跨多个关系指定的连接使用单个 join()，即 join([‘a’, ‘b’, ‘c’）。

+   **[orm]**

    修复了 query.instances()中的 bug，该 bug 无法处理多个额外的映射器或一个额外的列。

+   **[orm]**

    “delete-orphan”不再意味着“delete”。持续努力分离这两个操作的行为。

+   **[orm]**

    多对多关系正确设置了关联表上删除操作的绑定参数类型

+   **[orm]**

    多对多关系检查删除操作从关联表中删除的行数是否符合预期结果

+   **[orm]**

    session.get() 和 session.load() 将 **kwargs 传播到查询

+   **[orm]**

    修复了允许原始多态查询嵌入到相关子查询中的多态查询的问题

    参考：[#577](https://www.sqlalchemy.org/trac/ticket/577)

+   **[orm]**

    修复了 select_by(<propname>=<object instance>)风格的连接与多对多关系一起使用时的 bug，该 bug 在 r2556 中引入

+   **[orm]**

    mapper()的“primary_key”参数传播到“polymorphic”映射器。此列表中的主键列将被规范化为映射器的本地表的主键列。

+   **[orm]**

    恢复了在 sa.orm.strategies 记录器下记录“lazy loading clause”的功能，该功能在 0.3.7 中被移除

+   **[orm]**

    改进了对映射到 select()语句的属性的预加载支持；即 eagerloader 更善于定位正确的可选择项以附加其 LEFT OUTER JOIN。

### sql

+   **[sql]**

    _Label 类重写 compare_self 以返回其最终对象。意思是，如果你说 someexpr.label(‘foo’) == 5，它会产生正确的“someexpr == 5”。

+   **[sql]**

    _Label 传播“_hide_froms()”，使标量选择在 FROM 子句方面表现更正常 #574

+   **[sql]**

    修复了在使用 oid_column 作为排序依据时生成过长名称的问题（oids 在映射查询中被大量使用）

+   **[sql]**

    ResultProxy 的显著速度提升，预先缓存 TypeEngine 方言实现并减少每列的函数调用

+   **[sql]**

    通过新的 _Grouping 结构将括号应用于子句。使用运算符优先级更智能地将括号应用于子句，提供更清晰的子句嵌套（不会改变放置在其他子句中的子句，即没有‘parens’标志）

+   **[sql]**

    添加了‘modifier’关键字，类似于 func.<foo>，但不会添加括号。例如 select([modifier.DISTINCT(…)]) 等。

+   **[sql]**

    移除了“在 UNION 的一部分的 select 中不允许 group by”的限制

    参考：[#578](https://www.sqlalchemy.org/trac/ticket/578)

### mysql

+   **[mysql]**

    现在支持几乎所有 MySQL 列类型的声明和反射。添加了 NCHAR、NVARCHAR、VARBINARY、TINYBLOB、LONGBLOB、YEAR

+   **[mysql]**

    sqltypes.Binary 透传现在始终构建一个 BLOB，避免与非常旧的数据库版本出现问题

+   **[mysql]**

    支持列级别的 CHARACTER SET 和 COLLATE 声明，以及 ASCII、UNICODE、NATIONAL 和 BINARY 简写。

### 其他

+   **[engines]**

    在 Connection 中添加了 detach()，允许底层的 DBAPI 连接从其池中分离，在取消引用/关闭时关闭，而不是被池重用。

+   **[engines]**

    在 Connection 中添加了 invalidate()，立即使 Connection 及其底层的 DBAPI 连接无效。

+   **[firebird]**

    将最大标识符长度设置为 31

+   **[firebird]**

    由于问题#370，supports_sane_rowcount()设置为 False。versioned_id_col 功能在 FB 中不起作用。

+   **[firebird]**

    一些执行修复

+   **[firebird]**

    新的关联代理实现，实现对基于列表、字典和集合的关系集合的完全代理

+   **[firebird]**

    添加 orderinglist，一个自定义列表类，将对象属性与列表中对象的位置同步

+   **[firebird]**

    对 SelectResultsExt 进行了小修复，以避免在 select()期间绕过自身。

+   **[firebird]**

    将 filter()、filter_by()添加到 assignmapper

## 0.3.7

发布日期：2007 年 4 月 29 日星期日

### orm

+   **[orm]**

    修复了一个严重问题，即在使用 options(eagerload())后，映射器将始终对所有后续的 LIMIT/OFFSET/DISTINCT 查询应用“包装”行为，即使在这些后续查询中没有应用急加载。

+   **[orm]**

    添加了 query.with_parent(someinstance)方法。使用父实例的延迟连接条件搜索目标实例。接受可选字符串“property”以隔离所需的关系。还添加了静态 Query.query_from_parent(instance, property)版本。

    参考：[#541](https://www.sqlalchemy.org/trac/ticket/541)

+   **[orm]**

    改进了 query.XXX_by(someprop=someinstance)查询，使用类似于 with_parent 的方法，即使用“lazy”子句，防止将远程实例的表添加到 SQL 中，从而使更复杂的条件成为可能

    参考：[#554](https://www.sqlalchemy.org/trac/ticket/554)

+   **[orm]**

    向查询添加了聚合函数的生成版本，例如 sum()、avg()等。通过 query.apply_max()、apply_sum()等使用。#552

+   **[orm]**

    修复使用 distinct()或 distinct=True 与 join()和类似操作结合时的问题

+   **[orm]**

    与标签/bindparam 名称生成相对应，急加载器使用 md5 哈希为它们创建的别名生成确定性名称。

+   **[orm]**

    在给定“set”/“sets.Set”类或子类时，改进/修复了自定义集合类（在惰性加载期间仍在寻找 append()方法）

+   **[orm]**

    恢复了旧的“column_property()” ORM 函数（以前称为“column()”），以强制将任何列表达式添加为映射器的属性，特别是那些不在映射选择中的列。这允许将任何类型的“标量表达式”添加为关系（尽管它们在急加载方面存在问题）。

+   **[orm]**

    修复了针对多对多关系定位多态映射器的问题。

    参考：[#533](https://www.sqlalchemy.org/trac/ticket/533)

+   **[orm]**

    在 session.merge() 和结合其使用与 entity_name 方面取得了进展

    参考：[#543](https://www.sqlalchemy.org/trac/ticket/543)

+   **[orm]**

    在继承映射器之间的关系中进行了通常的调整，在这种情况下，建立了到子类映射器的 relation()，其中连接条件来自超类的表

### sql

+   **[sql]**

    结果集列的 keys() 不会小写化，会与 cursor.description 中的表达方式完全一致。请注意，这会导致 Oracle 中的列名全大写。

+   **[sql]**

    为支持 Unicode 表名、列名和 SQL 语句添加了初步支持，适用于可以支持它们的数据库。到目前为止，与 sqlite 和 postgres 一起工作。MySQL *大部分* 可以工作，除了 has_table() 函数无法工作。反射也可以工作。

+   **[sql]**

    Unicode 类型现在是 String 的直接子类，其中包含所有“convert_unicode”逻辑。这有助于更好地处理在数据库中出现的各种 Unicode 情况，如 MS-SQL，并允许对 Unicode 数据类型进行子类化。

    参考：[#522](https://www.sqlalchemy.org/trac/ticket/522)

+   **[sql]**

    现在可以在 in_() 子句中使用 ClauseElements，如绑定参数等。＃476

+   **[sql]**

    为 CompareMixin 元素实现了反向操作符，允许表达式如“5 + somecolumn”等。＃474

+   **[sql]**

    update() 和 delete() 的“where”条件现在将嵌入式 select() 语句与正在更新或删除的表进行关联。这与嵌套 select() 语句关联的方式相同，并且可以通过嵌入式 select() 上的 correlate=False 标志禁用。

+   **[sql]**

    现在在编译阶段生成列标签，这意味着它们的长度取决于方言。因此，在 Oracle 上，被截断为 30 个字符的标签将在 postgres 上扩展到 63 个字符。此外，真实的标签名称始终作为父可选择的访问器附加，因此无需关注“截断”的标签名称。

    参考：[#512](https://www.sqlalchemy.org/trac/ticket/512)

+   **[sql]**

    现在列标签和绑定参数“截断”也生成确定性名称，基于它们在编译的完整语句中的顺序。这意味着相同的语句将在应用程序重新启动时产生相同的字符串，并且允许 DB 查询计划缓存更好地工作。

+   **[sql]**

    当使用子查询时生成的“mini”列标签，用于解决 SQLite 行为不良的问题，不理解“foo.id”等同于“id”，现在仅在从中选择这些命名列的情况下生成

    参考：[#513](https://www.sqlalchemy.org/trac/ticket/513)

+   **[sql]**

    ColumnElement 上的 label()方法将正确地将基本元素的 TypeEngine 传播到标签，包括从 scalar=True select()语句创建的 label()。

+   **[sql]**

    MS-SQL 更好地检测查询是否为子查询，并知道不为其生成 ORDER BY 短语

    参考：[#513](https://www.sqlalchemy.org/trac/ticket/513)

+   **[sql]**

    修复了 fetchmany()“size”参数在大多数 dbapis 中是位置参数的问题

    参考：[#505](https://www.sqlalchemy.org/trac/ticket/505)

+   **[sql]**

    将 None 作为参数发送给 func.<something>将产生一个 NULL 参数

+   **[sql]**

    Unicode URL 中的查询字符串的键被编码为 ascii 以进行**kwargs 兼容

+   **[sql]**

    对原始 execute()更改进行微调，以支持位置参数的元组，而不仅仅是列表

    参考：[#523](https://www.sqlalchemy.org/trac/ticket/523)

+   **[sql]**

    修复了 case()构造，以将第一个 WHEN 条件的类型传播为 case 语句的返回类型

### extensions

+   **[extensions]**

    修复了 AssociationProxy，使得多个 AssociationProxy 对象可以与单个关联集合关联。

+   **[extensions]**

    根据它们的键（即 __name__）为 assign_mapper 命名方法 #551

### mysql

+   **[mysql]**

    支持作为 URL 查询字符串内联给出的 SSL 参数，以“ssl_”为前缀，感谢 terjeros@gmail.com。

+   **[mysql] [<schemaname>]**

    mysql 使用“DESCRIBE.<tablename>”来确定表是否存在，如果表不存在则捕获异常，这支持 unicode 表名以及模式名。已在 MySQL5 上测试，但应该也适用于 4.1 系列。（#557）

### sqlite

+   **[sqlite]**

    移除了 sqlite 将唯一索引反映为主键的愚蠢行为（？！）

### mssql

+   **[mssql]**

    pyodbc 现在是 MSSQL 的首选 DB-API，如果没有明确请求模块，将在模块探测时首先加载。

+   **[mssql]**

    现在使用@@SCOPE_IDENTITY 而不是@@IDENTITY。此行为可以通过 engine_connect“use_scope_identity”关键字参数覆盖，也可以在 dburi 中指定。

### oracle

+   **[oracle]**

    修复允许对具有 LIMIT/OFFSET 的相同 SELECT 对象进行连续编译的小问题。oracle 方言需要修改对象以具有 ROW_NUMBER OVER，并且在连续编译时未执行完整的步骤。

### 杂项

+   **[engines]**

    用于发出警告的警告模块（而不��日志记录）

+   **[engines]**

    清理所有引擎上的 DBAPI 导入策略

    参考：[#480](https://www.sqlalchemy.org/trac/ticket/480)

+   **[engines]**

    重新设计了引擎内部，减少了复杂性，代码路径数量；将更多状态放在 ExecutionContext 内部，以允许更多方言控制游标处理，结果集。 ResultProxy 完全重构，还有两个用于不同目的的“缓冲”结果集版本。

+   **[engines]**

    postgres 中的服务器端游标支持完全可用。

    参考：[#514](https://www.sqlalchemy.org/trac/ticket/514)

+   **[engines]**

    改进了自动失效连接的框架，当检测到与该数据库的断开相关的异常时，通过特定于方言的异常检测，整个连接池被丢弃并替换为新实例。#516

+   **[engines]**

    sqlalchemy.databases 中的方言成为 setuptools 入口点。加载内置数据库方言的方式与以往相同，但如果找不到任何方言，将尝试使用 pkg_resources 加载外部模块。

    参考：[#521](https://www.sqlalchemy.org/trac/ticket/521)

+   **[engines]**

    Engine 包含一个“url”属性，引用 create_engine()使用的 url.URL 对象。

+   **[informix]**

    添加了 informix 支持！感谢 James Zhang 付出了大量努力。

### orm

+   **[orm]**

    修复了关键问题，即在使用 options(eagerload())后，映射器将始终对所有后续 LIMIT/OFFSET/DISTINCT 查询应用查询“包装”行为，即使在这些后续查询中没有应用急加载。

+   **[orm]**

    添加了 query.with_parent(someinstance)方法。使用父实例的延迟连接条件搜索目标实例。接受可选字符串“property”以隔离所需的关系。还添加了静态 Query.query_from_parent(instance, property)版本。

    参考：[#541](https://www.sqlalchemy.org/trac/ticket/541)

+   **[orm]**

    改进了 query.XXX_by(someprop=someinstance)查询，使用与 with_parent 类似的方法，即使用“lazy”子句防止将远程实例的表添加到 SQL 中，从而使更复杂的条件成为可能

    参考：[#554](https://www.sqlalchemy.org/trac/ticket/554)

+   **[orm]**

    在查询中添加了聚合函数的生成版本，例如 sum()，avg()等。通过 query.apply_max()，apply_sum()等使用。#552

+   **[orm]**

    修复了在 join()和类似情况下使用 distinct()或 distinct=True 时的问题

+   **[orm]**

    对于标签/bindparam 名称生成，急加载器使用 md5 哈希为它们创建的别名生成确定性名称。

+   **[orm]**

    改进/修复了自定义集合类在给定“set”/“sets.Set”类或子类时（在惰性加载期间仍在寻找 append()方法）

+   **[orm]**

    恢复了旧的“column_property()”ORM 函数（以前称为“column()”），以强制将任何列表达式添加为映射器的属性，特别是那些在映射的可选择性中不存在的表达式。这允许将任何类型的“标量表达式”添加为关系（尽管它们存在与急加载的问题）。

+   **[orm]**

    修复了多对多关系定位到多态映射器的问题

    引用：[#533](https://www.sqlalchemy.org/trac/ticket/533)

+   **[orm]**

    在 session.merge() 上取得进展以及将其使用与 entity_name 结合起来

    引用：[#543](https://www.sqlalchemy.org/trac/ticket/543)

+   **[orm]**

    在这种情况下，通常需要调整继承映射器之间的关系，从父类表建立到子类映射器的关系

### sql

+   **[sql]**

    结果集列的 keys() 不会小写化，会与游标描述中的表达完全相同返回。注意这会导致在 Oracle 中 colnames 全部大写。

+   **[sql]**

    针对可以支持它们的数据库，添加了对 Unicode 表名、列名和 SQL 语句的初步支持。到目前为止，只支持 SQLite 和 postgres。MySQL *基本上*可以工作，除了 has_table() 函数无法工作。反射也能工作。

+   **[sql]**

    Unicode 类型现在是 String 的直接子类，现在包含所有“convert_unicode”逻辑。这有助于更好地处理各种数据库中出现的 Unicode 情况，例如 MS-SQL，并允许 Unicode 数据类型的子类化。

    引用：[#522](https://www.sqlalchemy.org/trac/ticket/522)

+   **[sql]**

    ClauseElements 现在可以在 in_() 子句中使用，例如绑定参数等。＃476

+   **[sql]**

    为 CompareMixin 元素实现了反向操作符，允许表达式如“5 + somecolumn”等。＃474

+   **[sql]**

    update() 和 delete() 的“where”条件现在会将嵌入的 select() 语句与正在更新或删除的表进行关联。这与嵌套的 select() 语句关联方式相同，并且可以通过嵌入式 select() 上的 correlate=False 标志进行禁用。

+   **[sql]**

    现在列标签在编译阶段生成，这意味着它们的长度是方言相关的。因此，在 Oracle 上被截断为 30 个字符的标签在 postgres 上将扩展到 63 个字符。此外，真实的标签名称始终作为父可选择性的访问器附加，因此不需要注意“截断”的标签名称。

    引用：[#512](https://www.sqlalchemy.org/trac/ticket/512)

+   **[sql]**

    列标签和绑定参数“截断”现在也会基于它们在编译的完整语句中的顺序生成确定性名称。这意味着相同的语句在应用程序重新启动时会产生相同的字符串，并且允许 DB 查询计划缓存工作得更好。

+   **[sql]**

    当使用子查询时生成的“mini”列标签，用于解决 SQLite 的问题，它不理解“foo.id”等同于“id”，现在仅在从中选择这些命名列的情况下生成

    参考：[#513](https://www.sqlalchemy.org/trac/ticket/513)

+   **[sql]**

    ColumnElement 上的 label()方法将正确地将基本元素的 TypeEngine 传播到标签，包括从 scalar=True select()语句创建的标签。

+   **[sql]**

    MS-SQL 更好地检测查询是否为子查询，并知道不为其生成 ORDER BY 短语

    参考：[#513](https://www.sqlalchemy.org/trac/ticket/513)

+   **[sql]**

    修复了 fetchmany()中“size”参数在大多数 dbapis 中是位置参数的问题

    参考：[#505](https://www.sqlalchemy.org/trac/ticket/505)

+   **[sql]**

    将 None 作为参数发送给 func.<something>将产生一个 NULL 参数

+   **[sql]**

    在 unicode URL 中的查询字符串将键编码为 ascii 以进行**kwargs 兼容。

+   **[sql]**

    对原始 execute()更改进行了轻微调整，以支持元组作为位置参数，而不仅仅是列表

    参考：[#523](https://www.sqlalchemy.org/trac/ticket/523)

+   **[sql]**

    修复了 case()构造以将第一个 WHEN 条件的类型传播为 case 语句的返回类型

### 扩展

+   **[extensions]**

    对 AssociationProxy 进行了重大修复，使得多个 AssociationProxy 对象可以与单个关联集合关联。

+   **[extensions]**

    根据它们的键（即 __name__）为 assign_mapper 命名方法 #551

### mysql

+   **[mysql]**

    支持作为 URL 查询字符串内联给出的 SSL 参数，以“ssl_”为前缀，感谢 terjeros@gmail.com。

+   **[mysql] [<schemaname>]**

    mysql 使用“DESCRIBE.<tablename>”来确定表是否存在，如果表不存在则捕获异常，以支持 unicode 表名以及模式名。已在 MySQL5 上测试，但也应该适用于 4.1 系列。(#557)

### sqlite

+   **[sqlite]**

    删除了 silly 行为，其中 sqlite 将唯一索引作为主键的一部分反映出来（？！）

### mssql

+   **[mssql]**

    pyodbc 现在是 MSSQL 的首选 DB-API，如果没有明确请求模块，则将在模块探测时首先加载。

+   **[mssql]**

    现在使用@@SCOPE_IDENTITY 而不是@@IDENTITY。此行为可以通过 engine_connect“use_scope_identity”关键字参数覆盖，也可以在 dburi 中指定。

### oracle

+   **[oracle]**

    对允许对具有 LIMIT/OFFSET 特性的相同 SELECT 对象进行连续编译的小修复。oracle 方言需要修改对象以具有 ROW_NUMBER OVER，并且在连续编译时未执行完整系列步骤。

### 杂项

+   **[engines]**

    用于发出警告的警告模块（而不是日志记录）

+   **[engines]**

    清理了所有引擎中的 DBAPI 导入策略

    参考：[#480](https://www.sqlalchemy.org/trac/ticket/480)

+   **[engines]**

    引擎内部的重构，减少了复杂性，代码路径数量；将更多状态放在 ExecutionContext 内，以允许更多方言控制游标处理，结果集。ResultProxy 完全重构，并且还有两个用于不同目的的“缓冲”结果集的版本。

+   **[引擎]**

    服务器端游标支持在 postgres 中完全可用。

    参考：[#514](https://www.sqlalchemy.org/trac/ticket/514)

+   **[引擎]**

    改进了自动失效连接的框架，通过方言特定的异常检测，对应于该数据库的断开相关错误消息。此外，当检测到“连接不再打开”的条件时，整个连接池将被丢弃，并用新实例替换。＃516

+   **[引擎]**

    sqlalchemy.databases 中的方言成为了 setuptools 的入口点。加载内置数据库方言的方式与以往相同，但如果找不到任何方言，将会退回到尝试使用 pkg_resources 加载外部模块。

    参考：[#521](https://www.sqlalchemy.org/trac/ticket/521)

+   **[引擎]**

    Engine 包含一个“url”属性，引用了 create_engine() 使用的 url.URL 对象。

+   **[informix]**

    informix 支持添加！感谢 James Zhang，他付出了大量努力。

## 0.3.6

发布日期：Fri Mar 23 2007

### ORM

+   **[ORM]**

    SelectResults 扩展的全部特性已经合并到一个新的一组方法中，这些方法都提供“生成式”行为，其中查询被复制并返回带有附加条件的新查询。新方法包括：

    > +   filter() - 对查询应用选择条件
    > +   
    > +   filter_by() - 对查询应用“by”风格的条件
    > +   
    > +   avg() - 返回给定列上的 avg() 函数
    > +   
    > +   join() - 加入到一个属性（或跨越属性列表）
    > +   
    > +   outerjoin() - 类似于 join()，但使用 LEFT OUTER JOIN
    > +   
    > +   limit()/offset() - 应用基于范围的 LIMIT/OFFSET 访问，该访问应用 limit/offset：session.query(Foo)[3:5]
    > +   
    > +   distinct() - 应用 DISTINCT
    > +   
    > +   list() - 评估条件并返回结果

    对于 Query 的 API 没有进行不兼容的更改，也没有弃用任何方法。现有方法如 select()、select_by()、get()、get_by() 等一次执行查询并返回结果，与以往一样。join_to()/join_via() 仍然存在，尽管生成式 join()/outerjoin() 方法更容易使用。

+   **[ORM]**

    使用 instances() 与多个映射器的返回值现在返回所请求的映射器列表的笛卡尔积，表示为元组列表。这对应于文档化的行为。为了使实例正确匹配，当使用此功能时，“唯一化”被禁用。

+   **[ORM]**

    Query 具有 add_entity() 和 add_column() 生成方法。这些方法将给定的映射器/类或 ColumnElement 在编译时添加到查询中，并将它们应用于 instances() 方法。用户负责构建合理的连接条件（否则可能会得到完全的笛卡尔积）。结果集是元组的列表，不唯一。

+   **[orm]**

    也可以将字符串和列发送到 instances() 的 *args 中，这些精确的结果列将成为结果元组的一部分。

+   **[orm]**

    可以将完整的 select() 构造传递给 query.select()（这已经可以工作了），但也可以使用 query.selectfirst()、query.selectone()，它们将按原样使用（即不会编译查询）。类似地，结果会被发送到 instances()。

+   **[orm]**

    对于由除了急加载器本身之外的其他内容放置在选择语句中的“order by”子句，急加载器不会“别名化”它们，以修复可能导致重复列的问题，如示例中所示。然而，这意味着你必须更加注意在 Query.select() 的“order by”中放置的列，你必须显式地在你的条件中命名它们（即，你不能依赖于急加载器为你添加它们）

    参考：[#495](https://www.sqlalchemy.org/trac/ticket/495)

+   **[orm]**

    在 Session 中添加了一个方便的多用途“identity_key()”方法，允许为主键值、实例和行生成标识键，由 Daniel Miller 提供

+   **[orm]**

    即使在操作发生在“反向引用”方的情况下，也将正确处理多对多表

    参考：[#249](https://www.sqlalchemy.org/trac/ticket/249)

+   **[orm]**

    添加了“refresh-expire”级联。允许 refresh() 和 expire() 调用沿关系传播。

    参考：[#492](https://www.sqlalchemy.org/trac/ticket/492)

+   **[orm]**

    对多态关系进行了更多修复，包括在多对一关系到多态映射器上生成正确的延迟子句。还修复了“direction”的检测，更具体地针对属于多态联合的列和不属于多态联合的列。

    参考：[#493](https://www.sqlalchemy.org/trac/ticket/493)

+   **[orm]**

    使用“viewonly=True”时对关系计算进行了一些修复，以将其他表引入到连接条件中，这些表不是关系的父/子映射的父级。

+   **[orm]**

    修复了循环引用关系中的刷新问题，其中包含对循环链之外的其他实例的引用，当循环中的一些对象实际上不是刷新的一部分时。

+   **[orm]**

    对“使用集合 B 刷新对象 A，但将 C 放入集合中”错误条件进行了严格检查 - **即使 C 是 B 的子类**，除非 B 的映射器以多态方式加载。否则，集合稍后将加载一个应该是“C”的“B”（因为它不是多态的），这会在双向关系中出现问题（即 C 有它的 A，但 A 的反向引用将懒加载为类型“B”的不同实例）。这个检查将会影响一些这样做而没有问题的人，因此错误消息还将记录一个标志“enable_typechecks=False”以禁用此检查。但请注意，特别是在没有此检查的情况下，双向关系会变得脆弱。

    参考：[#500](https://www.sqlalchemy.org/trac/ticket/500)

### sql

+   **[sql]**

    bindparam()名称现在是可重复的！在单个语句中指定两个不同的 bindparam()具有相同的名称，键将被共享。适当的位置/命名参数在编译时转换。对于“别名”具有冲突名称的绑定参数的旧行为，指定“unique=True” - 此选项仍然在所有自动生成的（基于值的）绑定参数内部使用。

+   **[sql]**

    对绑定参数作为列子句的绑定参数或文字列的支持稍微改进，例如通过 bindparam()或 literal()，即 select([literal('foo')])

+   **[sql]**

    MetaData 可以通过构造函数的“url”或“engine”参数，或使用 connect()方法绑定到引擎。BoundMetaData 与 MetaData 相同，只是需要 engine_or_url 参数。DynamicMetaData 相同，并默认提供线程本地连接。

+   **[sql]**

    exists()现在可以作为独立的可选择项使用，不仅仅在 WHERE 子句中，例如 exists([columns], criterion).select()

+   **[sql]**

    相关子查询可以在 ORDER BY、GROUP BY 中工作

+   **[sql]**

    修复了使用显式连接执行函数的问题，即 conn.execute(func.dosomething())

+   **[sql]**

    在 select()上使用 use_labels 标志不会自动为文字列元素创建标签，因为我们无法对文本做出任何假设。要为文字列创建标签，可以说“somecol AS somelabel”，或使用 literal_column(“somecol”).label(“somelabel”)

+   **[sql]**

    当将文字列列“代理”到其可选择的列集合中时，不会对文字列进行引用（is_literal 标志被传播）。文字列通过 literal_column(“somestring”)指定。

+   **[sql]**

    在 Join.select()中添加了“fold_equivalents”布尔参数，该参数根据连接条件删除结果列子句中已知等效的“重复”列。在构建 Postgres 抱怨存在重复列名的连接子查询时，这非常有用。

+   **[sql]**

    修复了 ForeignKeyConstraint 上的 use_alter 标志

    参考：[#503](https://www.sqlalchemy.org/trac/ticket/503)

+   **[sql]**

    修复了在 topological.py 中使用仅限于 2.4 的“reversed”的问题

    参考：[#506](https://www.sqlalchemy.org/trac/ticket/506)

+   **[sql]**

    对于黑客，重构了 ClauseElement 和 SchemaItem 的“visitor”系统，以便项目的遍历由 ClauseVisitor 本身控制，使用方法 visitor.traverse(item)。accept_visitor()方法仍然可以直接调用，但不会遍历子项目。ClauseElement/SchemaItem 现在有一个可配置的 get_children()方法，用于返回每个父对象的子元素集合。这允许项目的完整遍历清晰明了（以及可记录），并且有一种限制遍历的简单方法（只需传递标志，这些标志由适当的 get_children()方法捕获）。

    参考：[#501](https://www.sqlalchemy.org/trac/ticket/501)

+   **[sql]**

    当设置为零时，case 语句的“else_”参数现在正常工作。

### 扩展

+   **[extensions]**

    SelectResults 上的 options()方法现在像其他 SelectResults 方法一样实现了“生成”。但是你现在无论如何都会只使用 Query。

    参考：[#472](https://www.sqlalchemy.org/trac/ticket/472)

+   **[extensions]**

    通过 assignmapper 添加了 query()方法。这有助于导航到 Query 上的所有新的生成方法。

### mysql

+   **[mysql]**

    为 MSString 添加了一个通用的**kwargs，以帮助反射模糊类型（例如 MS 4.0 中的“varchar() binary”）

+   **[mysql]**

    添加了明确的 MSTimeStamp 类型，当使用 types.TIMESTAMP 时生效。

### oracle

+   **[oracle]**

    为任意大小的输入使二进制工作！cx_oracle 工作正常，这是我的错，因为 BINARY 被传递而不是 BLOB 用于 setinputsizes（而且单元测试甚至没有设置输入大小）。

+   **[oracle]**

    还修复了 CLOB 读/写在一个单独的变更集上。

+   **[oracle]**

    auto_setinputsizes 默认为 True 用于 Oracle，修复了它错误地传播错误类型的情况。

### 杂项

+   **[ms-sql]**

    在 DATE 列类型上删除了秒输入（可能应该完全删除时间）

    应该完全删除时间）

+   **[ms-sql]**

    浮点字段中的空值不再引发错误

+   **[ms-sql]**

    带有 OFFSET 的 LIMIT 现在会引发错误（MS-SQL 不支持 OFFSET）

+   **[ms-sql]**

    添加了一种使用 MSSQL 类型 VARCHAR(max)而不是 TEXT 用于大型无大小字���串字段的方法。使用新的“text_as_varchar”来打开它。

    参考：[#509](https://www.sqlalchemy.org/trac/ticket/509)

+   **[ms-sql]**

    没有 LIMIT 的 ORDER BY 子句现在在子查询中被剥离，因为 MS-SQL 禁止此用法

+   **[ms-sql]**

    清理模块导入代码；可指定的 DB-API 模块；模块偏好的更明确排序。

    参考：[#480](https://www.sqlalchemy.org/trac/ticket/480)

### orm

+   **[orm]**

    SelectResults 扩展的完整功能集已合并到可用于 Query 的一组新方法中。这些方法都提供“生成”行为，其中查询被复制并返回一个添加了额外条件的新查询。新方法包括：

    > +   filter() - 将选择条件应用于查询
    > +   
    > +   filter_by() - 将“by”样式的条件应用于查询
    > +   
    > +   avg() - 返回给定列上的 avg()函数
    > +   
    > +   join() - 加入到属性（或跨��性列表）
    > +   
    > +   outerjoin() - 类似于 join()，但使用 LEFT OUTER JOIN
    > +   
    > +   limit()/offset() - 应用基于范围的 LIMIT/OFFSET 访问，该访问应用 limit/offset：session.query(Foo)[3:5]
    > +   
    > +   distinct() - 应用 DISTINCT
    > +   
    > +   list() - 评估条件并返回结果

    Query 的 API 没有进行不兼容的更改，也没有废弃任何方法。现有方法如 select()、select_by()、get()、get_by()都会立即执行查询并返回结果，就像以前一样。join_to()/join_via()仍然存在，尽管生成的 join()/outerjoin()方法更容易使用。

+   **[orm]**

    对于与 instances()一起使用的多个 mapper 的返回值现在返回请求的 mapper 列表的笛卡尔积，表示为元组的列表。这与文档中的行为相对应。为了正确匹配实例，当使用此功能时，“唯一化”被禁用。

+   **[orm]**

    Query 具有 add_entity()和 add_column()生成方法。这些方法将在编译时将给定的 mapper/class 或 ColumnElement 添加到查询中，并将它们应用于 instances()方法。用户负责构建合理的连接条件（否则可能会得到完整的笛卡尔积）。结果集是元组的列表，不唯一。

+   **[orm]**

    字符串和列也可以发送到 instances()的*args 中，其中这些确切的结果列将成为结果元组的一部分。

+   **[orm]**

    可以将完整的 select()构造传递给 query.select()（这在任何情况下都有效），但也可以使用 query.selectfirst()、query.selectone()，它们将按原样使用（即不会编译查询）。与将结果发送到 instances()类似。

+   **[orm]**

    急加载不会“别名化”由急加载器本身放置在 select 语句中的“order by”子句，以修复可能出现的重复列的问题。但是，这意味着您必须更加小心地处理放置在 Query.select()的“order by”中的列，您必须明确地在您的条件中命名它们（即您不能依赖于急加载器为您添加它们）

    参考：[#495](https://www.sqlalchemy.org/trac/ticket/495)

+   **[orm]**

    为 Session 添加了一个方便的多用途“identity_key()”方法，允许为主键值、实例和行生成身份键，感谢 Daniel Miller

+   **[orm]**

    多对多表将被正确处理，即使在操作发生在操作的“backref”一侧时也是如此

    参考：[#249](https://www.sqlalchemy.org/trac/ticket/249)

+   **[orm]**

    添加了“refresh-expire”级联。允许 refresh()和 expire()调用沿关系传播。

    参考：[#492](https://www.sqlalchemy.org/trac/ticket/492)

+   **[orm]**

    对多态关系进行了更多修复，涉及到对多对一关系到多态 mapper 的正确惰性子句生成。还修复了“方向”的检测，更具体地针对属于多态联合的列与不属于多态联合的列的定位。

    参考：[#493](https://www.sqlalchemy.org/trac/ticket/493)

+   **[orm]**

    在使用“viewonly=True”时，修复了关系计算的一些问题，以将其他表引入到连接条件中，这些表不是关系的父/子映射的一部分

+   **[orm]**

    修复了包含对循环引用关系的刷新的问题，当循环链中的某些对象实际上不是刷新的一部分时

+   **[orm]**

    对“将对象 A 与 B 的集合刷新，但将 C 放入集合中”错误条件进行了积极检查 - **即使 C 是 B 的子类**，除非 B 的映射器以多态方式加载。否则，集合稍后将加载一个“B”，而应该是一个“C”（因为它不是多态的），这会破坏双向关系（即 C 有其 A，但 A 的反向引用将懒加载为类型“B”的不同实例）。这个检查将会影响一些没有问题的人，因此错误消息还将记录一个标志“enable_typechecks=False”来禁用此检查。但请注意，特别是在没有此检查的情况下，双向关系变得脆弱。

    参考：[#500](https://www.sqlalchemy.org/trac/ticket/500)

### sql

+   **[sql]**

    bindparam()现在可以重复使用！在单个语句中指定两个不同的 bindparam()具有相同的名称，键将被共享。适当的位置/命名参数在编译时转换。对于具有冲突名称的绑定参数进行“别名”处理的旧行为，指定“unique=True” - 此选项仍然在所有自动生成的（基于值的）绑定参数的内部使用。

+   **[sql]**

    对绑定参数作为列子句提供了更好的支持，可以通过 bindparam()或 literal()来实现，例如 select([literal(‘foo’)])

+   **[sql]**

    MetaData 可以通过构造函数的“url”或“engine”参数绑定到引擎，也可以使用 connect()方法。BoundMetaData 与 MetaData 相同，只是 engine_or_url 参数是必需的。DynamicMetaData 相同，并默认提供线程本地连接。

+   **[sql]**

    exists()现在可以作为一个独立的可选择项使用，不仅仅在 WHERE 子句中，例如 exists([columns], criterion).select()

+   **[sql]**

    相关子查询可以在 ORDER BY、GROUP BY 中工作

+   **[sql]**

    修复了使用显式连接执行函数的问题，例如 conn.execute(func.dosomething())

+   **[sql]**

    在 select()上使用 use_labels 标志不会为文字列元素自动创建标签，因为我们无法对文本做出任何假设。要为文字列创建标签，可以说“somecol AS somelabel”，或使用 literal_column(“somecol”).label(“somelabel”)

+   **[sql]**

    当将文字列代理到其可选择的列集合中时，文字列不会被引用（is_literal 标志被传播）。文字列通过 literal_column(“somestring”)指定。

+   **[sql]**

    在 Join.select() 中添加了一个“fold_equivalents”布尔参数，它根据连接条件从结果列子句中移除‘重复’列，这些列基于连接条件被认为是等效的。在构造 Postgres 抱怨有重复列名的子查询时，这非常有用。

+   **[sql]**

    修复了 ForeignKeyConstraint 上的 use_alter 标志

    参考：[#503](https://www.sqlalchemy.org/trac/ticket/503)

+   **[sql]**

    在 topological.py 中修复了仅限于 2.4 的“reversed”的使用。

    参考：[#506](https://www.sqlalchemy.org/trac/ticket/506)

+   **[sql]**

    重构了 ClauseElement 和 SchemaItem 的“访问者”系统，使得项的遍历由 ClauseVisitor 本身控制，使用方法 visitor.traverse(item)。accept_visitor() 方法仍然可以直接调用，但不会遍历子项。ClauseElement/SchemaItem 现在有一个可配置的 get_children() 方法，用于返回每个父对象的子元素集合。这允许项的完整遍历清晰明了（以及可记录），并且有一种简单的限制遍历的方法（只需传递标志，这些标志由适当的 get_children() 方法拾取）。

    参考：[#501](https://www.sqlalchemy.org/trac/ticket/501)

+   **[sql]**

    当 else_ 参数设置为零时，case 语句现在正常工作。

### extensions

+   **[extensions]**

    SelectResults 上的 options() 方法现在像其他 SelectResults 方法一样实现了“生成”。但是现在你只会使用 Query。

    参考：[#472](https://www.sqlalchemy.org/trac/ticket/472)

+   **[extensions]**

    query() 方法由 assignmapper 添加。这有助于浏览 Query 上的所有新生成方法。

### mysql

+   **[mysql]**

    在 MSString 上添加了一个捕获所有 **kwargs，以帮助反射模糊类型（例如 MS 4.0 中的“varchar() binary”）

+   **[mysql]**

    添加了显式的 MSTimeStamp 类型，当使用 types.TIMESTAMP 时生效。

### oracle

+   **[oracle]**

    让二进制对任何大小的输入都有效！cx_oracle 正常工作，这是我的错，因为传递的是 BINARY 而不是 BLOB 以设置输入大小（而且单元测试甚至没有设置输入大小）。

+   **[oracle]**

    还在另一个变更集中修复了 CLOB 读写问题。

+   **[oracle]**

    对于 Oracle，auto_setinputsizes 默认为 True，修复了不正确传播坏类型的情况。

### 杂项

+   **[ms-sql]**

    在 DATE 列类型上删除了秒输入（可能是为了黑客）。

    应该完全移除时间）

+   **[ms-sql]**

    浮点字段中的空值不再引发错误

+   **[ms-sql]**

    使用 OFFSET 的 LIMIT 现在会引发错误（MS-SQL 不支持 OFFSET）

+   **[ms-sql]**

    添加了一种使用 MSSQL 类型 VARCHAR(max) 而不是 TEXT 用于大型未定大小的字符串字段的方法。使用新的“text_as_varchar”来打开它。

    参考：[#509](https://www.sqlalchemy.org/trac/ticket/509)

+   **[ms-sql]**

    没有 LIMIT 的 ORDER BY 子句现在在子查询中被剥离，因为 MS-SQL 禁止此用法

+   **[ms-sql]**

    清理模块导入代码；可指定的 DB-API 模块；模块偏好的更明确排序。

    参考：[#480](https://www.sqlalchemy.org/trac/ticket/480)

## 0.3.5

发布日期：Thu Feb 22 2007

### orm

+   **[orm] [bugs]**

    另一个关系计算的重构。允许更准确的 ORM 行为与映射器之间的关系，特别是多态映射器，以及它们与 Query、SelectResults 的使用。票据包括,,。

    参考：[#439](https://www.sqlalchemy.org/trac/ticket/439), [#441](https://www.sqlalchemy.org/trac/ticket/441), [#448](https://www.sqlalchemy.org/trac/ticket/448)

+   **[orm] [bugs]**

    删除了在类上指定自定义集合的已弃用方法；现在必须使用“collection_class”选项。旧方法开始在人们使用 assign_mapper() 时产生冲突，现在修补���一个“options”方法，与一个名为“options”的关系一起使用。 （关系优先于猴子补丁的 assign_mapper 方法）。

+   **[orm] [bugs]**

    extension() 查询选项传播到 Mapper._instance() 方法，以便调用所有与加载相关的方法

    参考：[#454](https://www.sqlalchemy.org/trac/ticket/454)

+   **[orm] [bugs]**

    对继承映射器的急切关系如果关系没有返回任何行不会失败。

+   **[orm] [bugs]**

    修复了多个后代类上急切关系加载的错误

    参考：[#486](https://www.sqlalchemy.org/trac/ticket/486)

+   **[orm] [bugs]**

    修复了非常大的拓扑排序问题，感谢 ants.aasma at gmail

    参考：[#423](https://www.sqlalchemy.org/trac/ticket/423)

+   **[orm] [bugs]**

    急切加载在检测“自引用”关系方面稍微更严格，特别是在多态映射器之间。这导致“急切降级”为延迟加载。

+   **[orm] [bugs]**

    改进了对嵌入到查询.select() 的“where”条件的复杂查询的支持

    参考：[#449](https://www.sqlalchemy.org/trac/ticket/449)

+   **[orm] [bugs]**

    映射器选项如 eagerload()、lazyload()、deferred()，将适用于“synonym()”关系

    参考：[#485](https://www.sqlalchemy.org/trac/ticket/485)

+   **[orm] [bugs]**

    修复了级联操作错误地将已删除的集合项包含在级联中的错误

    参考：[#445](https://www.sqlalchemy.org/trac/ticket/445)

+   **[orm] [bugs]**

    修复了一个关系删除错误，当一个一对多的子项在单个工作单元中移动到新的父项时

    参考：[#478](https://www.sqlalchemy.org/trac/ticket/478)

+   **[orm] [bugs]**

    修复了关系删除错误，其中具有单个列作为子项的 PK/FK 的父项/子项在手动删除或使用“delete-orphan”而不是“delete”级联时会引发“清空主键”错误

+   **[orm] [bugs]**

    修复了延迟加载，以便在只设置 PK 列属性时不会错误地发生加载操作

+   **[orm] [enhancements]**

    实现了 mapper 的 foreign_keys 参数。与 primaryjoin/secondaryjoin 参数一起使用，指定/覆盖在 Table 实例上定义的外键。

    参考：[#385](https://www.sqlalchemy.org/trac/ticket/385)

+   **[orm] [增强]**

    contains_eager('foo')自动意味着 eagerload('foo')

+   **[orm] [增强]**

    在 contains_eager()中添��了“alias”参数。使用它来指定在查询中用于急加载子项的别名的字符串名称或 Alias 实例。比“decorator”更容易使用

+   **[orm] [增强]**

    为映射到映射表的别名的结果集映射添加了“contains_alias()”选项

+   **[orm] [增强]**

    添加了对 py2.5“with”语句与 SessionTransaction 的支持

    参考：[#468](https://www.sqlalchemy.org/trac/ticket/468)

### sql

+   **[sql]**

    “case_sensitive”的值现在默认为 True，不管标识符的大小写如何，除非明确设置为 False。这是因为对象可能被标记为其他内容，其中包含混合大小写，传播“case_sensitive=False”会破坏这一点。在使用标签和“fake”列对象时修复引用的其他修复

+   **[sql]**

    添加了“supports_execution()”方法到 ClauseElement，以便各种类型的子句可以表达是否适合执行...例如，您可以执行“select”，但不能执行“Table”或“Join”。

+   **[sql]**

    修复了对引擎、连接上直接文本执行 execute()的参数传递。可以处理*args 或列表实例用于位置参数，**kwargs 或字典实例用于命名参数，或列表的列表或字典用于调用 executemany()

+   **[sql]**

    对 BoundMetaData 进行了小修复，接受 unicode 或字符串 URL

+   **[sql]**

    修复了由 andrija at gmail 提供的命名 PrimaryKeyConstraint 生成

    参考：[#466](https://www.sqlalchemy.org/trac/ticket/466)

+   **[sql]**

    修复了在列上生成 CHECK 约束

    参考：[#464](https://www.sqlalchemy.org/trac/ticket/464)

+   **[sql]**

    修复了在 tometadata()操作中传播列和表级别的约束

### 扩展

+   **[扩展]**

    添加了 distinct()方法到 SelectResults。通常只在使用 count()时会有所不同。

+   **[扩展]**

    在 SelectResults 中添加了 options()方法，相当于 query.options()

    参考：[#472](https://www.sqlalchemy.org/trac/ticket/472)

+   **[扩展]**

    添加了可选的 __table_opts__ 字典到 ActiveMapper，将 kw 选项发送到 Table 对象

    参考：[#462](https://www.sqlalchemy.org/trac/ticket/462)

+   **[扩展]**

    添加了 selectfirst()、selectfirst_by()到 assign_mapper

    参考：[#467](https://www.sqlalchemy.org/trac/ticket/467)

### mysql

+   **[mysql]**

    修复了在旧数据库上反射可能返回“show variables like”语句的 array()类型的问题

### mssql

+   **[mssql]**

    对 pyodbc 的初步支持（耶！）

    参考：[#419](https://www.sqlalchemy.org/trac/ticket/419)

+   **[mssql]**

    添加了对 NVARCHAR 类型的更好支持

    参考：[#298](https://www.sqlalchemy.org/trac/ticket/298)

+   **[mssql]**

    修复了 pymssql 上的提交逻辑问题

+   **[mssql]**

    修复了使用 schema 的 query.get() 问题

    参考：[#456](https://www.sqlalchemy.org/trac/ticket/456)

+   **[mssql]**

    修复了非整数关系的问题

    参考：[#473](https://www.sqlalchemy.org/trac/ticket/473)

+   **[mssql]**

    现在可以在运行时选择 DB-API 模块

    参考：[#419](https://www.sqlalchemy.org/trac/ticket/419)

+   **[mssql] [415] [481] [tickets:422]**

    现在通过了更多的单元测试

+   **[mssql]**

    与 ANSI 函数的自动单元测试兼容性更好

    参考：[#479](https://www.sqlalchemy.org/trac/ticket/479)

+   **[mssql]**

    对具有自动插入的隐式序列 PK 列的支持改进

    参考：[#415](https://www.sqlalchemy.org/trac/ticket/415)

+   **[mssql]**

    修复了 adodbapi 中空密码的问题

    参考：[#371](https://www.sqlalchemy.org/trac/ticket/371)

+   **[mssql]**

    修复了与 pyodbc 一起使用单元测试的问题

    参考：[#481](https://www.sqlalchemy.org/trac/ticket/481)

+   **[mssql]**

    修复了 db-url 查询中的 auto_identity_insert 问题

+   **[mssql]**

    在 db-url 查询参数中添加了 query_timeout。 目前仅对 pymssql 有效

+   **[mssql]**

    使用了 pymssql 0.8.0 进行了测试（现在是 LGPL）

### oracle

+   **[oracle]**

    在返回“rowid”作为 ORDER BY 列或与 ROW_NUMBER OVER 一起使用时，oracle 方言会检查其应用于的可选择项，并在不适用时切换到表 PK，即对于 UNION。 检查 DISTINCT、GROUP BY（rowid 无效的其他地方）仍然是一个 TODO。 允许多态映射功能。

    参考：[#436](https://www.sqlalchemy.org/trac/ticket/436)

+   **[oracle]**

    非主键列上的序列现在在插入时会正确触发

+   **[oracle]**

    添加了对 PrefetchingResultProxy 的支持，以在已知存在时预提取 LOB 列，修复

    参考：[#435](https://www.sqlalchemy.org/trac/ticket/435)

+   **[oracle]**

    实现了基于同义词的表反射，包括跨数据库链接的反射

    参考：[#379](https://www.sqlalchemy.org/trac/ticket/379)

+   **[oracle]**

    当由于某些权限错误而无法反映相关表时，会发出日志警告

    参考：[#363](https://www.sqlalchemy.org/trac/ticket/363)

### 杂项

+   **[postgres]**

    更好地反映了交替模式表的序列

    参考：[#442](https://www.sqlalchemy.org/trac/ticket/442)

+   **[postgres]**

    非主键列上的序列现在在插入时会正确触发

+   **[postgres]**

    添加了 PGInterval 类型，PGInet 类型

    参考：[#444](https://www.sqlalchemy.org/trac/ticket/444), [#460](https://www.sqlalchemy.org/trac/ticket/460)

### orm

+   **[orm] [bugs]**

    对关系计算进行了进一步的重构。 允许更准确的 ORM 行为，包括来自/到/之间的映射器，特别是多态映射器，以及它们与 Query、SelectResults 的使用。 票据包括,,。

    参考：[#439](https://www.sqlalchemy.org/trac/ticket/439), [#441](https://www.sqlalchemy.org/trac/ticket/441), [#448](https://www.sqlalchemy.org/trac/ticket/448)

+   **[orm] [bugs]**

    移除了在类上指定自定义集合的已弃用方法；现在必须使用“collection_class”选项。旧方法开始在人们使用 assign_mapper() 时产生冲突，现在它会修补一个“options”方法，与一个名为“options”的关系一起使用（关系优先于 monkeypatched assign_mapper 方法）。

+   **[orm] [bugs]**

    extension() 查询选项传播到 Mapper._instance() 方法，以便调用所有与加载相关的方法

    参考：[#454](https://www.sqlalchemy.org/trac/ticket/454)

+   **[orm] [bugs]**

    如果关系到一个继承的映射器，即使关系没有返回任何行也不会失败。

+   **[orm] [bugs]**

    修复了多个后代类上急切关系加载的 bug

    参考：[#486](https://www.sqlalchemy.org/trac/ticket/486)

+   **[orm] [bugs]**

    修复了非常大的拓扑排序的 bug，感谢 ants.aasma@gmail

    参考：[#423](https://www.sqlalchemy.org/trac/ticket/423)

+   **[orm] [bugs]**

    eager loading 对于检测“自引用”关系稍微更加严格，特别是在多态映射器之间。这会导致“急切降级”到懒加载。

+   **[orm] [bugs]**

    改进了嵌入到查询.select() 的“where”条件中的复杂查询的支持

    参考：[#449](https://www.sqlalchemy.org/trac/ticket/449)

+   **[orm] [bugs]**

    mapper 选项如 eagerload()、lazyload()、deferred()，将适用于“synonym()”关系

    参考：[#485](https://www.sqlalchemy.org/trac/ticket/485)

+   **[orm] [bugs]**

    修复了级联操作错误地将已删除的集合项包含在级联中的 bug

    参考：[#445](https://www.sqlalchemy.org/trac/ticket/445)

+   **[orm] [bugs]**

    在单个工作单元中将一对多子项移动到新父项时，修复了关系删除错误

    参考：[#478](https://www.sqlalchemy.org/trac/ticket/478)

+   **[orm] [bugs]**

    修复了关系删除错误，其中父/子项的主键/外键只有一个列时，如果手动删除或使用“delete”级联而没有使用“delete-orphan”，则会引发“清空主键”错误

+   **[orm] [bugs]**

    修复了延迟加载，以便在只设置了 PK 列属性时不会错误地发生加载操作

+   **[orm] [enhancements]**

    实现了 mapper 的 foreign_keys 参数。与 primaryjoin/secondaryjoin 参数一起使用，用于指定/覆盖在 Table 实例上定义的外键。

    参考：[#385](https://www.sqlalchemy.org/trac/ticket/385)

+   **[orm] [enhancements]**

    contains_eager('foo') 自动意味着 eagerload('foo')

+   **[orm] [enhancements]**

    为 contains_eager() 添加了“alias”参数。使用它来指定查询中用于急切加载子项的别名的字符串名称或 Alias 实例。比“decorator”更容易使用

+   **[orm] [enhancements]**

    为映射到映射表的别名的结果集映射添加了“contains_alias()”选项

+   **[orm] [enhancements]**

    添加了对 py2.5“with”语句与 SessionTransaction 的支持

    参考：[#468](https://www.sqlalchemy.org/trac/ticket/468)

### sql

+   **[sql]**

    “case_sensitive”的值现在默认为 True，不管标识符的大小写如何，除非明确设置为 False。这是因为对象可能被标记为其他内容，其中包含混合大小写，传播“case_sensitive=False”会破坏这一点。在使用标签和“fake”列对象时修复引用的其他修复

+   **[sql]**

    在 ClauseElement 中添加了一个“supports_execution()”方法，以便各种类型的子句可以表达是否适合执行...例如，您可以执行“select”，但不能执行“Table”或“Join”。

+   **[sql]**

    修复了对引擎、连接上直接文本执行 execute()的参数传递，可以处理*args 或用于位置参数的列表实例，**kwargs 或用于命名参数的字典实例，或用于调用 executemany()的列表列表或字典

+   **[sql]**

    对 BoundMetaData 进行了小修复，以接受 unicode 或字符串 URL

+   **[sql]**

    修复了由 andrija at gmail 提供的命名 PrimaryKeyConstraint 生成

    参考：[#466](https://www.sqlalchemy.org/trac/ticket/466)

+   **[sql]**

    修复了在列上生成 CHECK 约束

    参考：[#464](https://www.sqlalchemy.org/trac/ticket/464)

+   **[sql]**

    修复了 tometadata()操作以在列和表级别传播 Constraints

### extensions

+   **[extensions]**

    在 SelectResults 中添加了 distinct()方法。通常只在使用 count()时才会有所不同。

+   **[extensions]**

    在 SelectResults 中添加了 options()方法，相当于 query.options()

    参考：[#472](https://www.sqlalchemy.org/trac/ticket/472)

+   **[extensions]**

    在 ActiveMapper 中添加了可选的 __table_opts__ 字典，将 kw 选项发送到 Table 对象

    参考：[#462](https://www.sqlalchemy.org/trac/ticket/462)

+   **[extensions]**

    在 assign_mapper 中添加了 selectfirst()、selectfirst_by()

    参考：[#467](https://www.sqlalchemy.org/trac/ticket/467)

### mysql

+   **[mysql]**

    修复了在旧 DB 上可能返回“show variables like”语句的 array()类型的反射

### mssql

+   **[mssql]**

    对 pyodbc 进行了初步支持（耶！）

    参考：[#419](https://www.sqlalchemy.org/trac/ticket/419)

+   **[mssql]**

    添加了对 NVARCHAR 类型的更好支持

    参考：[#298](https://www.sqlalchemy.org/trac/ticket/298)

+   **[mssql]**

    修复了 pymssql 上的提交逻辑

+   **[mssql]**

    修复了带有模式的 query.get()

    参考：[#456](https://www.sqlalchemy.org/trac/ticket/456)

+   **[mssql]**

    修复了非整数关系

    参考：[#473](https://www.sqlalchemy.org/trac/ticket/473)

+   **[mssql]**

    现在可以在运行时选择 DB-API 模块

    参考：[#419](https://www.sqlalchemy.org/trac/ticket/419)

+   **[mssql] [415] [481] [tickets:422]**

    现在通过了更多的单元测试

+   **[mssql]**

    与 ANSI 函数更好的 unittest 兼容性

    参考：[#479](https://www.sqlalchemy.org/trac/ticket/479)

+   **[mssql]**

    改进了对具有自动插入的隐式序列 PK 列的支持

    参考: [#415](https://www.sqlalchemy.org/trac/ticket/415)

+   **[mssql]**

    修复了 adodbapi 中的空密码

    参考: [#371](https://www.sqlalchemy.org/trac/ticket/371)

+   **[mssql]**

    修复以使单元测试与 pyodbc 正常工作

    参考: [#481](https://www.sqlalchemy.org/trac/ticket/481)

+   **[mssql]**

    修复了 db-url 查询上的 auto_identity_insert

+   **[mssql]**

    在 db-url 查询参数中添加了 query_timeout。目前仅对 pymssql 有效

+   **[mssql]**

    使用 pymssql 0.8.0 进行测试（现在是 LGPL）

### oracle

+   **[oracle]**

    当返回 "rowid" 作为 ORDER BY 列或与 ROW_NUMBER OVER 一起使用时，oracle 方言会检查其被应用的可选择对象，并在不适用时切换到表 PK，即对于 UNION。检查 DISTINCT、GROUP BY（其他位置无效的 rowid）仍然是一个 TODO。允许多态映射功能。

    参考: [#436](https://www.sqlalchemy.org/trac/ticket/436)

+   **[oracle]**

    在非主键列上的序列将会正确地在插入时触发

+   **[oracle]**

    添加了 PrefetchingResultProxy 支持以在已知存在时预取 LOB 列，修复

    参考: [#435](https://www.sqlalchemy.org/trac/ticket/435)

+   **[oracle]**

    根据同义词反射表的实现，包括跨数据库链接的情况

    参考: [#379](https://www.sqlalchemy.org/trac/ticket/379)

+   **[oracle]**

    在由于某些权限错误而无法反射相关表时，记录警告

    参考: [#363](https://www.sqlalchemy.org/trac/ticket/363)

### 杂项

+   **[postgres]**

    更好地反射了备用模式表的序列

    参考: [#442](https://www.sqlalchemy.org/trac/ticket/442)

+   **[postgres]**

    在非主键列上的序列将会正确地在插入时触发

+   **[postgres]**

    添加了 PGInterval 类型，PGInet 类型

    参考: [#444](https://www.sqlalchemy.org/trac/ticket/444), [#460](https://www.sqlalchemy.org/trac/ticket/460)

## 0.3.4

发布日期: 2007 年 1 月 23 日 星期二

### 一般

+   **[一般]**

    全局的 "保险"->"确保" 更改。在美国英语中，“保险”实际上与“确保”在很大程度上是可以互换的（字典上是这样说的），所以我不完全是文盲，但是“确保”是非歧义的，绝对是次优的。

### orm

+   **[orm]**

    在打开"一罐蠕虫"的第一个孔的时候: 说 query.select_by(somerelationname=someinstance) 将会创建由 "somerelationname" 的映射器表示的主键列与 "someinstance" 中实际主键的连接。

+   **[orm]**

    重新设计了关系如何与 "多态" 映射器交互，即具有 select_table 和多态标志的映射器。更好地确定适当的连接条件，与用户定义的连接条件的交互，以及支持自引用多态映射器。

+   **[orm]**

    关于多态映射关系，当编译关系时进行了一些更深入的错误检查，以检测在关系的两侧都具有外键引用的情况下的模糊“primaryjoin”。还加强了用于定位“关系方向”的条件，将关系的“外键”与“primaryjoin”关联起来

+   **[orm]**

    对“具体”继承映射概念稍作改进，尽管该概念尚未完全明确（添加测试用例以支持在多态基础上的具体映射）。

+   **[orm]**

    修复了在 synonym()上的“proxy=True”行为

+   **[orm]**

    修复了删除孤儿基本上无法与多对多关系一起工作的错误，反向引用通常隐藏了症状

    参考：[#427](https://www.sqlalchemy.org/trac/ticket/427)

+   **[orm]**

    在映射器编译步骤中添加了互斥锁。我一直不愿意在 SA 中添加任何线程内容，但这是真正需要的一个地方，因为映射器通常是“全局”的，虽然它们的状态在正常操作期间不会改变，但初始编译步骤会显著修改内部状态，并且这一步通常不会发生在模块级别的初始化时间（除非您调用 compile()），而是在第一次请求时

+   **[orm]**

    实际实现了“session.merge()”的基本概念。需要更多测试。

+   **[orm]**

    添加了“compile_mappers()”函数作为编译所有映射器的快捷方式

+   **[orm]**

    修复了 MapperExtension create_instance，以便 entity_name 正确关联到新实例

+   **[orm]**

    ORM 对象实例化速度提升，行的急加载

+   **[orm]**

    发送到“cascade”字符串的无效选项将引发异常

    参考：[#406](https://www.sqlalchemy.org/trac/ticket/406)

+   **[orm]**

    修复了映射器刷新/过期中的错误，急加载器未正确重新填充项目列表

    参考：[#407](https://www.sqlalchemy.org/trac/ticket/407)

+   **[orm]**

    修复了 post_update 以确保即使在非插入/删除场景下也更新行

    参考：[#413](https://www.sqlalchemy.org/trac/ticket/413)

+   **[orm]**

    如果您尝试修改实体的主键值然后刷新它，将添加错误消息

    参考：[#412](https://www.sqlalchemy.org/trac/ticket/412)

### sql

+   **[sql]**

    添加了“fetchmany()”支持到 ResultProxy

+   **[sql]**

    添加了对列“key”属性在 row[<key>]/row.<key>中可用的支持

+   **[sql]**

    将“BooleanExpression”更改为从“BinaryExpression”继承，以便布尔表达式也可以遵循列子句行为（例如 label()等）。

+   **[sql]**

    从 func.<xxx>调用中修剪尾随下划线，例如 func.if_()

+   **[sql]**

    修复了当通过单独调用 append_column()构建选择语句的列列表时，子查询的相关性；这修复了 ORM 错误，即嵌套的选择语句未与 Query 对象生成的主选择相关联。

+   **[sql]**

    另一个修复子查询相关性的方法，以便仅当子查询只有一个 FROM 元素时*不*相关联该单个元素，因为查询中至少需要一个 FROM 元素。

+   **[sql]**

    默认���“timezone”设置现在为 False。这对应于 Python 的 datetime 行为以及 Postgres 的 timestamp/time 类型（目前是唯一有时区敏感的方言）

    参考：[#414](https://www.sqlalchemy.org/trac/ticket/414)

+   **[sql]**

    “op()”函数现在被视为“操作”，而不是“比较”。区别在于，操作产生一个 BinaryExpression，可以进行进一步的操作，而比较产生更严格的 BooleanExpression

+   **[sql]**

    尝试将反射的主键列重新定义为非主键会引发错误

+   **[sql]**

    类型系统略有修改，以支持可以被方言覆盖的 TypeDecorators（好吧，这不太清楚，它允许下面的 mssql 调整成为可能）

### extensions

+   **[extensions]**

    在 assign_mapper 中添加了“validate=False”参数，如果为 True，则确保只有映射的属性被命名

    参考：[#426](https://www.sqlalchemy.org/trac/ticket/426)

+   **[extensions]**

    assign_mapper 添加了“options”、“instances”函数（即 MyClass.instances()）

### mysql

+   **[mysql]**

    mysql 在 SHOW CREATE TABLE 期间使用的外键引号类型不一致，反射已更新以适应所有三种样式

    参考：[#420](https://www.sqlalchemy.org/trac/ticket/420)

+   **[mysql]**

    mysql 表创建选项现在可以通用传递，即 Table(…, mysql_engine=’InnoDB’, mysql_collate=”latin1_german2_ci”, mysql_auto_increment=”5”, mysql_<somearg>…), 有助于

    参考：[#418](https://www.sqlalchemy.org/trac/ticket/418)

### mssql

+   **[mssql]**

    添加了一个 NVarchar 类型（产生 NVARCHAR），还有一个提供 Unicode 翻译的 MSUnicode，无论方言 convert_unicode 设置如何。

### oracle

+   **[oracle]**

    *轻微*支持二进制，但仍需弄清如何插入相当大的值（超过 4K）。需要将 auto_setinputsizes=True 发送到 create_engine()，行必须逐个完全获取，等等。

### 杂项

+   **[postgres]**

    修复对表的初始 checkfirst 以考虑当前模式

    参考：[#424](https://www.sqlalchemy.org/trac/ticket/424)

+   **[postgres]**

    postgres 有一个可选的“server_side_cursors=True”标志，将利用服务器端游标。这些适用于仅获取部分结果并且必须处理非常大的无界结果集。虽然我们希望这成为默认行为，但不同的环境似乎有不同的结果，原因尚未被分离，因此我们目前将该功能默认关闭。使用了最近在 psycopg 邮件列表上发现的一个显然未记录的 psycopg2 行为。

+   **[postgres]**

    为 postgres 表添加了“BIGSERIAL”支持，具有 PGBigInteger/autoincrement

+   **[postgres]**

    修复了 postgres 反射以更好处理存在模式名称时的情况；感谢 jason (at) ncsmags.com

    参考：[#402](https://www.sqlalchemy.org/trac/ticket/402)

+   **[firebird]**

    约束创建顺序将主键放在所有其他约束之前；对于 Firebird 是必需的，对于其他数据库也不是坏主意

    参考：[#408](https://www.sqlalchemy.org/trac/ticket/408)

+   **[firebird]**

    修复了 Firebird 自动加载多字段外键的问题

    参考：[#409](https://www.sqlalchemy.org/trac/ticket/409)

+   **[firebird]**

    Firebird NUMERIC 类型正确处理没有精度的类型

    参考：[#409](https://www.sqlalchemy.org/trac/ticket/409)

### 一般

+   **[general]**

    全局“insure”->“ensure”更改。在美国英语中，“insure”实际上与“ensure”在很大程度上是可以互换的（字典上是这么说的），所以我并不是完全文盲，但“ensure”明显更好，因为它是不含糊的。

### orm

+   **[orm]**

    在罐头中开了第一个洞：说 query.select_by(somerelationname=someinstance) 将创建“somerelationname”映射器表示的主键列与“someinstance”实际主键之间的连接。

+   **[orm]**

    重新设计了关系与“多态”映射器的交互方式，即具有 select_table 和多态标志的映射器。更好地确定适当的连接条件，与用户定义的连接条件的交互，以及支持自引用多态映射器。

+   **[orm]**

    在编译关系时，与多态映射关系相关，在检测关系时进行了更深入的错误检查，以检测在关系的两侧都在主连接条件中具有外键引用的情况下的模棱两可的“primaryjoin”。还加强了用于定位“关系方向”的条件，将关系的“外键”与“primaryjoin”关联起来。

+   **[orm]**

    对“具体”继承映射概念进行了一点改进，尽管该概念尚未完全阐明（添加了支持在多态基础上使用具体映射器的测试用例）。

+   **[orm]**

    修复了在 synonym() 上的“proxy=True”行为

+   **[orm]**

    修复了删除孤立基本上无法与多对多关系一起工作的错误，反向引用通常隐藏了症状

    参考：[#427](https://www.sqlalchemy.org/trac/ticket/427)

+   **[orm]**

    在映射器编译步骤中添加了互斥锁。我一直不愿意在 SA 中添加任何线程相关的东西，但这是真正需要的地方，因为映射器通常是“全局”的，虽然它们的状态在正常操作期间不会改变，但初始编译步骤会显著修改内部状态，并且这一步通常不会在模块级别初始化时发生（除非调��� compile()），而是在第一次请求时发生

+   **[orm]**

    实际实现了“session.merge()”的基本思想。需要更多测试。

+   **[orm]**

    添加了“compile_mappers()”函数作为编译所有映射器的快捷方式

+   **[orm]**

    修复了 MapperExtension create_instance，以便实体名称正确地与新实例关联

+   **[orm]**

    增加了 ORM 对象实例化、行的急切加载的速度优化

+   **[orm]**

    发送给 'cascade' 字符串的无效选项将引发异常

    参考：[#406](https://www.sqlalchemy.org/trac/ticket/406)

+   **[orm]**

    修复了在 mapper 刷新/过期中的错误，因此急切加载器没有正确地重新填充项目列表

    参考：[#407](https://www.sqlalchemy.org/trac/ticket/407)

+   **[orm]**

    修复了 post_update 以确保即使对于非插入/删除场景，行也会被更新

    参考：[#413](https://www.sqlalchemy.org/trac/ticket/413)

+   **[orm]**

    如果您实际尝试修改实体上的主键值，然后刷新它，将添加错误消息

    参考：[#412](https://www.sqlalchemy.org/trac/ticket/412)

### sql

+   **[sql]**

    向 ResultProxy 添加了“fetchmany()”支持

+   **[sql]**

    添加了对列“key”属性可在 row[<key>]/row.<key> 中使用的支持

+   **[sql]**

    将“BooleanExpression”更改为从“BinaryExpression”子类化，以便布尔表达式也可以遵循列子句行为（即 label() 等）。

+   **[sql]**

    func.<xxx> 调用中的尾部下划线会被修剪掉，例如 func.if_()

+   **[sql]**

    修复了当选择语句的列列表是使用对 append_column() 的单独调用构造时子查询相关性的问题；这修复了一个 ORM 错误，即嵌套的 select 语句没有与 Query 对象生成的主要 select 相关联。

+   **[sql]**

    修复子查询相关性的另一个方法，以便只有一个 FROM 元素的子查询不会与该单个元素相关联，因为查询中至少需要一个 FROM 元素。

+   **[sql]**

    默认的“timezone”设置现在为 False。这与 Python 的 datetime 行为以及 Postgres 的 timestamp/time 类型相对应（目前唯一对时区敏感的方言）

    参考：[#414](https://www.sqlalchemy.org/trac/ticket/414)

+   **[sql]**

    “op()” 函数现在被视为一个“操作”，而不是一个“比较”。区别在于，操作会产生一个二进制表达式，而后续操作可以在其中进行，而比较会产生更严格的布尔表达式。

+   **[sql]**

    试图将反射的主键列重新定义为非主键会引发错误

+   **[sql]**

    类型系统略有修改以支持可以被方言覆盖的 TypeDecorators（好吧，这不是很清楚，它允许下面的 mssql 调整成为可能）

### 扩展

+   **[extensions]**

    对 assign_mapper 添加了“validate=False”参数，如果为 True，则会确保仅命名了映射的属性

    参考：[#426](https://www.sqlalchemy.org/trac/ticket/426)

+   **[extensions]**

    assign_mapper 添加了“options”、“instances”函数（即 MyClass.instances()）

### mysql

+   **[mysql]**

    mysql 在 SHOW CREATE TABLE 期间使用的引号类型不一致，反射已更新以适应所有三种风格。

    参考：[#420](https://www.sqlalchemy.org/trac/ticket/420)

+   **[mysql]**

    mysql 表创建选项现在可以通用传递，即 Table(…, mysql_engine=’InnoDB’, mysql_collate=”latin1_german2_ci”, mysql_auto_increment=”5”, mysql_<somearg>…), 有助于

    参考：[#418](https://www.sqlalchemy.org/trac/ticket/418)

### mssql

+   **[mssql]**

    添加了一个 NVarchar 类型（生成 NVARCHAR），还有一个 MSUnicode，为 NVarchar 提供 Unicode-翻译，无论方言 convert_unicode 设置如何。

### oracle

+   **[oracle]**

    *轻微* 支持二进制，但仍需找出如何插入相当大的值（超过 4K）。需要将 auto_setinputsizes=True 发送到 create_engine()，行必须逐个完全获取，等等。

### 杂项

+   **[postgres]**

    修复了对表的初始 checkfirst 考虑当前模式的问题

    参考：[#424](https://www.sqlalchemy.org/trac/ticket/424)

+   **[postgres]**

    postgres 有一个可选的 “server_side_cursors=True” 标志，将利用服务器端游标。这些适用于仅获取部分结果并且必要用于处理非常大的无界结果集。虽然我们希望这成为默认行为，但不同的环境似乎有不同的结果，原因尚未被分离，因此我们目前将该功能默认关闭。使用了最近在 psycopg 邮件列表上发现的一个显然未记录的 psycopg2 行为。

+   **[postgres]**

    添加了对带有 PGBigInteger/autoincrement 的 postgres 表的 “BIGSERIAL” 支持

+   **[postgres]**

    修复了对包含模式名称的 postgres 反射的处理方式；感谢 jason (at) ncsmags.com

    参考：[#402](https://www.sqlalchemy.org/trac/ticket/402)

+   **[firebird]**

    约束创建的顺序将主键放在所有其他约束之前；对于 firebird 是必需的，对于其他数据库也是个好主意

    参考：[#408](https://www.sqlalchemy.org/trac/ticket/408)

+   **[firebird]**

    Firebird 修复了自动加载多字段外键

    参考：[#409](https://www.sqlalchemy.org/trac/ticket/409)

+   **[firebird]**

    Firebird NUMERIC 类型正确处理没有精度的类型

    参考：[#409](https://www.sqlalchemy.org/trac/ticket/409)

## 0.3.3

发布日期：2006 年 12 月 15 日 星期五

+   **[no_tags]**

    修复了基于字符串的 FROM 子句，即 select(…, from_obj=[“sometext”])

+   **[no_tags]**

    修复了 passive_deletes 标志，lazy=None（noload）标志

+   **[no_tags]**

    添加了处理大型集合的示例/文档

+   **[no_tags]**

    在 sqlalchemy 命名空间中添加了 object_session() 方法

+   **[no_tags]**

    修复了 QueuePool bug，使其更好地重新连接到无法访问的数据库（感谢 SÃ©bastien Lelong），还修复了 dispose() 方法

+   **[no_tags]**

    修复了使 MySQL rowcount 正常工作的补丁！

    参考：[#396](https://www.sqlalchemy.org/trac/ticket/396)

+   **[no_tags]**

    修复了 MySQL 捕获 2006/2014 错误以正确重新引发 OperationalError 异常

## 0.3.2

发布日期：2006 年 12 月 10 日 星期日

+   **[no_tags]**

    修复了重大的连接池错误。修复了 MySQL 不同步错误，还将防止所有数据库中意外回滚事务

    参考资料：[#387](https://www.sqlalchemy.org/trac/ticket/387)

+   **[no_tags]**

    与 0.3.1 相比的主要速度增强，将速度提高到 0.2.8 的水平

+   **[no_tags]**

    使得数十个耗时生成日志消息的调试日志调用条件化

+   **[no_tags]**

    修复了级联规则中的错误，导致整个对象图可能在保存/更新级联时不必要地级联

+   **[no_tags]**

    属性模块中的各种加速

+   **[no_tags]**

    会话中的身份映射默认情况下不再是弱引用。要使其成为弱引用，请使用 create_session(weak_identity_map=True) 修复。

    参考资料：[#388](https://www.sqlalchemy.org/trac/ticket/388)

+   **[no_tags]**

    MySQL 检测到错误 2006（服务器已断开连接）和 2014（命令不同步），并在发生错误的连接上使其无效。

+   **[no_tags]**

    MySQL 布尔类型修复：

    参考资料：[#307](https://www.sqlalchemy.org/trac/ticket/307)

+   **[no_tags]**

    postgres 反射修复：

    参考资料：[#349](https://www.sqlalchemy.org/trac/ticket/349), [#382](https://www.sqlalchemy.org/trac/ticket/382)

+   **[no_tags]**

    添加了 EXCEPT、INTERSECT、EXCEPT ALL、INTERSECT ALL 的关键字

    参考资料：[#247](https://www.sqlalchemy.org/trac/ticket/247)

+   **[no_tags]**

    assignmapper 扩展中的 assign_mapper 返回创建的映射器

    参考资料：[#2110](https://www.sqlalchemy.org/trac/ticket/2110)

+   **[no_tags]**

    向 Select 类添加了 label() 函数，当使用 scalar=True 创建标量子查询时，例如“select x, y, (select max(foo) from table) AS foomax from table”。

+   **[no_tags]**

    向 ForeignKey 添加了 onupdate 和 ondelete 关键字参数；如果存在，则传播到底层的 ForeignKeyConstraint。（但是不要在另一个方向上传播）

+   **[no_tags]**

    修复了 session.update() 以保留传入对象的“脏”状态

+   **[no_tags]**

    不再通过 in_() 函数向 IN 发送一个可选的“union”多选；现在只允许一个可选项到 in_() 函数（如果需要 union，请自行创建 union）

+   **[no_tags]**

    改进了通过 cascade=”none” 等禁用保存-更新级联的支持。

+   **[no_tags]**

    向 relation() 添加了“remote_side”参数，仅用于自引用的映射，以强制父/子关系的方向。替换了“foreignkey”参数用于“切换”方向的用法。“foreignkey”参数已被弃用，将最终由专用于映射器上的 ForeignKey 规范的参数替换。

## 0.3.1

发布日期：2006 年 11 月 13 日（星期一）

### orm

+   **[orm]**

    “删除”级联将加载所有子对象，如果它们尚未加载。这可以通过在 relation() 上设置 passive_deletes=True 来关闭（即旧行为）。

+   **[orm]**

    调整了重写的急切查询生成，以不在循环急切加载的关系（例如反向引用）上失败

+   **[orm]**

    修复了一个 bug，即 eagerload()（或 lazyload()）选项未正确指示查询是否在生成 LIMIT 查询时使用“嵌套”。

+   **[ORM]**

    修复了刷新时循环依赖排序中的 bug；如果对象 A 包含一个循环的多对一关系到对象 B，并且对象 B 刚刚附加到对象 A，*但*对象 B 本身没有改变，那么 B 的主键属性与 A 的外键属性的多对一同步不会发生。

    参考：[#360](https://www.sqlalchemy.org/trac/ticket/360)

+   **[ORM]**

    为 query.count 实现了 from_obj 参数，改进了 selectresults 上的 count 函数

    参考：[#325](https://www.sqlalchemy.org/trac/ticket/325)

+   **[ORM]**

    在 ORM 关系的“级联”步骤中添加了一���断言，以检查附加到父对象的对象类是否合适（即如果 A.items 存储 B 对象，则如果向 A.items 附加 C，则引发错误）

+   **[ORM]**

    新扩展 sqlalchemy.ext.associationproxy，提供透明的“关联对象”映射。新示例 examples/association/proxied_association.py 进行了说明。

+   **[ORM]**

    改进了单表继承以加载目标类下面的完整层次结构

+   **[ORM]**

    修复了拓扑排序中一个细微条件的问题，其中一个节点可能会出现两次，用于

    参考：[#362](https://www.sqlalchemy.org/trac/ticket/362)

+   **[ORM]**

    对拓扑排序进行了额外的重构和重新设计，用于

    参考：[#365](https://www.sqlalchemy.org/trac/ticket/365)

+   **[ORM]**

    对于某种类型，“delete-orphan”可以在多个父类上设置；只有当实例未附加到*任何*这些父类中时，它才是“孤立的”

### 杂项

+   **[引擎/池]**

    一些新的 Pool 实用类，更新了文档

+   **[引擎/池]**

    “use_threadlocal”在 Pool 上默认为 False（与 create_engine 相同）

+   **[引擎/池]**

    修复了对 Compiled 对象的直接执行

+   **[引擎/池]**

    重新设计 create_engine()以严格处理传入的**kwargs。所有关键字参数必须被方言、连接池和引擎构造函数中的一个消耗，否则将抛出一个 TypeError，其中描述了与所选方言/池/引擎配置相关的全部无效 kwargs。

+   **[数据库/类型]**

    MySQL 在“describe”时捕获异常并报告为 NoSuchTableError

+   **[数据库/类型]**

    进一步修复了 sqlite 布尔值，默认情况下不起作用

+   **[数据库/类型]**

    修复了在使用模式时对 postgres 序列引用的问题

### ORM

+   **[ORM]**

    “delete”级联将加载所有子对象，如果它们尚未加载。可以通过在 relation()上设置 passive_deletes=True 来关闭此功能（即旧行为）。

+   **[ORM]**

    调整了重新设计的急切查询生成，以避免在循环急切加载的关系（如反向引用）上失败

+   **[ORM]**

    修复了一个 bug，即 eagerload()（或 lazyload()）选项未正确指示查询是否在生成 LIMIT 查询时使用“嵌套”。

+   **[ORM]**

    修复了刷新时循环依赖排序中的错误；如果对象 A 包含一个循环的多对一关系到对象 B，而对象 B 刚刚附加到对象 A，*但*对象 B 本身没有改变，那么 B 的主键属性与 A 的外键属性的多对一同步不会发生。

    参考：[#360](https://www.sqlalchemy.org/trac/ticket/360)

+   **[orm]**

    为 query.count 实现了 from_obj 参数，改进了 selectresults 上的 count 函数

    参考：[#325](https://www.sqlalchemy.org/trac/ticket/325)

+   **[orm]**

    在 ORM 关系的“级联”步骤中添加了一个断言，以检查附加到父对象的对象的类是否合适（即如果 A.items 存储 B 对象，则如果向 A.items 附加 C，则引发错误）

+   **[orm]**

    新��展 sqlalchemy.ext.associationproxy，提供透明的“关联对象”映射。新示例 examples/association/proxied_association.py 进行了说明。

+   **[orm]**

    对单表继承进行了改进，以加载目标类下的完整层次结构

+   **[orm]**

    修复了拓扑排序中一个细微条件的问题，其中一个节点可能出现两次，用于

    参考：[#362](https://www.sqlalchemy.org/trac/ticket/362)

+   **[orm]**

    对拓扑排序进行了额外的重构和重构，用于

    参考：[#365](https://www.sqlalchemy.org/trac/ticket/365)

+   **[orm]**

    对于某种类型，“delete-orphan”可以设置在多个父类上；如果实例未附加到*任何*这些父类中，则该实例是“孤立的”

### 杂项

+   **[engine/pool]**

    一些新的 Pool 实用类，更新了文档

+   **[engine/pool]**

    “use_threadlocal”在 Pool 上默认为 False（与 create_engine 相同）

+   **[engine/pool]**

    修复了对编译对象的直接执行

+   **[engine/pool]**

    重新设计 create_engine()以严格处理传入的**kwargs。所有关键字参数必须被方言、连接池和引擎构造函数中的一个消耗，否则将抛出 TypeError，其中描述了与所选方言/池/引擎配置相关的全部无效 kwargs。

+   **[databases/types]**

    MySQL 在“describe”时捕获异常并报告为 NoSuchTableError

+   **[databases/types]**

    进一步修复 sqlite 布尔值，默认情况下未起作用

+   **[databases/types]**

    修复了在使用模式时对 postgres 序列引用的问题

## 0.3.0

发布日期：2006 年 10 月 22 日星期日

### 一般

+   **[general]**

    现在通过标准的 Python“logging”模块实现日志记录。“echo”关键字参数仍然可用，但设置/取消日志级别为其各自的类/实例。所有日志记录可以通过直接通过 Python API 设置“sqlalchemy”命名空间中的 INFO 和 DEBUG 级别来控制。类级别的日志记录在“sqlalchemy.<module>.<classname>”下，实例级别的日志记录在“sqlalchemy.<module>.<classname>.0x..<00-FF>”下。测试套件包括“--log-info”和“--log-debug”参数，独立于--verbose/--quiet 工作。在 ORM 中添加了日志记录，以允许跟踪映射器配置、行迭代。

+   **[general]**

    文档生成系统已经进行了全面改进，设计更简单，与 Markdown 更加集成

### orm

+   **[orm]**

    修改了属性跟踪以更智能地检测更改，特别是对于可变类型。TypeEngine 对象现在在定义如何比较两个标量实例方面发挥更大作用，包括通过 PickleType 实现的 MutableType mixin 的添加。工作单元现在将“脏”列表跟踪为所有属性管理器检测到更改的持久对象的表达式。解决的基本问题是检测 PickleType 对象上的更改，但也将类型处理和“修改”对象检查泛化为更完整和可扩展。

+   **[orm]**

    对“属性加载器”和“选项”架构进行了广泛重构。ColumnProperty 和 PropertyLoader 通过可切换的“策略”定义其加载行为，MapperOptions 不再使用映射器/属性复制来运行；它们通过 QueryContext 和 SelectionContext 对象在查询/实例时间传播。所有用于处理继承以及 options()的内部映射器和属性复制已被移除；映射器和属性的结构比以前简单得多，并且在新的“interfaces”模块中清晰地列出。

+   **[orm]**

    与映射器/属性改组相关，对映射器实例()方法进行了内部重构，使用 SelectionContext 对象跟踪操作期间的状态。轻微的 API 破坏：由于更改的结果，MapperExtension 上的 append_result()和 populate_instances()方法现在具有稍微不同的方法签名；希望这些方法目前尚未广泛使用。

+   **[orm]**

    instances()方法现在移至 Query，向后兼容版本仍保留在 Mapper 上。

+   **[orm]**

    添加了 contains_eager() MapperOption，与 instances()一起使用，用于指定应该从结果集中急切加载的属性，默认情况下使用它们的普通列名，或者根据自定义行转换函数进行转换。

+   **[orm]**

    对工作单元提交方案进行了更多重新排列，以更好地允许循环刷新中的依赖关系正常工作…更新了任务遍历/日志记录实现

+   **[orm]**

    多态映射器（即使用继承）现在按照所有继承类的表顺序生成 INSERT 语句

    参考：[#321](https://www.sqlalchemy.org/trac/ticket/321)

+   **[orm]**

    添加了自动“行切换”功能到映射，它将检测具有相同标识键的待处理实例/已删除实例对，并将 INSERT/DELETE 转换为单个 UPDATE

+   **[orm]**

    “关联”映射简化以利用自动“行切换”功能

+   **[orm]**

    “自定义列表类”现在通过 relation()的“collection_class”关键字参数实现。旧方法仍然有效，但已被弃用

    参考：[#212](https://www.sqlalchemy.org/trac/ticket/212)

+   **[orm]**

    在 relation()中添加了“viewonly”标志，允许构建对 flush()过程没有影响的关系。

+   **[orm]**

    在基本 Query 选择/获取函数中添加了“lockmode”参数，包括“with_lockmode”函数，以获取具有默认锁定模式的 Query 副本。将“read”/“update”参数转换为 select 端的 for_update 参数。

    参考：[#292](https://www.sqlalchemy.org/trac/ticket/292)

+   **[orm]**

    在 Query/Mapper 中实现了“版本检查”逻辑，当 version_id_col 生效且使用 query.with_lockmode()获取已加载实例时使用

+   **[orm]**

    改进了 post_update 行为；在更新行时更好地避免更新过多行，仅更新所需的列

    参考：[#208](https://www.sqlalchemy.org/trac/ticket/208)

+   **[orm]**

    调整了急切加载，使其“急切链”与正常的 mapper 设置保持分离，从而防止与延迟加载器操作发生冲突，修复了问题

    参考：[#308](https://www.sqlalchemy.org/trac/ticket/308)

+   **[orm]**

    修复了延迟组加载问题

+   **[orm]**

    session.flush()不会关闭它打开的连接

    参考：[#346](https://www.sqlalchemy.org/trac/ticket/346)

+   **[orm]**

    在 mapper 中添加了“batch=True”标志；如果为 False，save_obj 将完全逐个保存一个对象，包括调用 before_XXXX 和 after_XXXX

+   **[orm]**

    在 mapper 中添加了“column_prefix=None”参数；将给定字符串（通常为‘_’）预先添加到从 mapper 的 Table 设置的基于列的属性中。

+   **[orm]**

    在 query.select()的 from_obj 参数中指定连接将替换查询的主表，如果表在给定的 from_obj 中的某处。这样可以在查询中生成自定义连接和外连接，而不会使主表添加两次。

    参考：[#315](https://www.sqlalchemy.org/trac/ticket/315)

+   **[orm]**

    调整了急切加载，更加周到地将其 LEFT OUTER JOIN 附加到给定查询中，查找可能已经设置的自定义“FROM”子句。

+   **[orm]**

    在 SelectResults 中添加了 join_to 和 outerjoin_to 转换方法，根据属性名称构建连接/外连接条件。还添加了 select_from 以明确设置 from_obj 参数。

+   **[orm]**

    从 mapper 中删除了“is_primary”标志。

### sql

+   **[sql] [construction]**

    将“for_update”参数更改为接受 False/True/“nowait”和“read”，后两者仅由 Oracle 和 MySQL 解释。

    参考：[#292](https://www.sqlalchemy.org/trac/ticket/292)

+   **[sql] [construction]**

    在 sql 方言中添加了 extract()函数（SELECT extract(field FROM expr)）

+   **[sql] [construction]**

    BooleanExpression 包括新的“negate”参数，用于指定适当的否定运算符（如果有的话）。

+   **[sql] [construction]**

    对“IN”或“IS”子句进行否定将导致“NOT IN”、“IS NOT”（而不是 NOT (x IN y)）。

+   **[sql] [construction]**

    Function 对象现在知道在 FROM 子句中该做什么。它们的行为应该是相同的，除了现在你还可以做像 select([‘*’], from_obj=[func.my_function()])这样的事情来从结果中获取多个列，甚至使用 sql.column()构造来命名返回列。

    参考：[#172](https://www.sqlalchemy.org/trac/ticket/172)

### 模式

+   **[模式]**

    对模式包进行了相当多的清理，删除了模糊方法、不再需要的方法。使用略微更受限制，更加强调明确性。

+   **[模式]**

    Table 和其他可选择对象的“primary_key”属性变成了一个类似集合的 ColumnCollection 对象；它是有序的，但没有数值索引。从相同基础表（例如两个 Alias 对象）派生的两个主键之间的比较子句可以通过 table1.primary_key==table2.primary_key 生成。

+   **[模式]**

    ForeignKey(Constraint)支持“use_alter=True”，通过 ALTER 创建/删除外键。这允许建立循环外键关系。

+   **[模式]**

    从 Table 和 Column 中删除了 append_item()方法；最好内联构造 Table/Column/相关对象，但如果需要，可以使用 append_column()、append_foreign_key()、append_constraint()等。

+   **[模式]**

    table.create()不再返回 Table 对象，而是没有返回值。通常情况下，通过 metadata 创建表，这是更可取的，因为它将处理表依赖关系。

+   **[模式]**

    添加了 UniqueConstraint（放在 Table 级别）、CheckConstraint（放在 Table 或 Column 级别）。

+   **[模式]**

    Column 上的 index=False/unique=True 现在创建 UniqueConstraint，index=True/unique=False 创建普通 Index，Column 上的 index=True/unique=True 创建唯一 Index。column 的‘index’和‘unique’关键字参数现在只能是布尔值；对于索引或唯一约束的显式名称和分组，使用 UniqueConstraint/Index 构造显式。

+   **[模式]**

    将 autoincrement=True 添加到 Column；如果明确设置为 False，则将禁用 postgres/mysql/mssql 的 SERIAL/AUTO_INCREMENT/identity seq 的模式生成。

+   **[模式]**

    TypeEngine 对象现在有处理复制和比较其特定类型值的方法。目前被 ORM 使用，见下文。

+   **[模式]**

    修复了在显式覆盖主键列时反射时出现的条件，其中 PrimaryKeyConstraint 会使反射和程序化列重复。

+   **[模式]**

    Column 和 ColumnElement 上的“foreign_key”属性一般不建议使用，而是使用基于“foreign_keys”列表/集合的属性，该属性考虑了一个列上的多个外键。“foreign_key”将返回“foreign_keys”列表/集合中的第一个元素，如果列表为空，则返回 None。

### sqlite

+   **[sqlite]**

    sqlite 布尔数据类型默认将 False/True 转换为 0/1。

+   **[sqlite]**

    修复了 Date/Time（SLDate/SLTime）类型；现在与 postgres 一样好用。

    参考：[#335](https://www.sqlalchemy.org/trac/ticket/335)

### oracle

+   **[oracle]**

    Oracle 对 cx_Oracle.TIMESTAMP 有实验性支持，需要在光标上调用 setinputsizes()，现在可以通过 oracle 方言的‘auto_setinputsizes’标志启用。

### 杂项

+   **[ms-sql]**

    修复了 bug 261（对于区分大小写的 MS-SQL 数据库，表反射出现问题）

+   **[ms-sql]**

    现在可以为 pymssql 指定端口

+   **[ms-sql]**

    引入了新的“auto_identity_insert”选项，用于在为 IDENTITY 列指定值时自动切换到“SET IDENTITY_INSERT”模式

+   **[ms-sql]**

    现在支持多列外键

+   **[ms-sql]**

    修复了反射日期/时间列的问题

+   **[ms-sql]**

    添加了 NCHAR 和 NVARCHAR 类型支持

+   **[firebird]**

    别名不使用“AS”

+   **[firebird]**

    在反射不存在的表时正确地引发 NoSuchTableError

+   **[连接/池/执行]**

    连接池跟踪打开的光标，并在连接返回到池中时自动关闭它们。可以受到导致引发错误或不执行任何操作的选项的影响。修复了 MySQL 等其他问题

+   **[连接/池/执行]**

    修复了 Connection 在提交/回滚后不会失去其 Transaction 的 bug

+   **[连接/池/执行]**

    向 ComposedSQLEngine、ResultProxy 添加了 scalar()方法

+   **[连接/池/执行]**

    当 ResultProxy 本身关闭时，ResultProxy 将关闭底层光标。这将自动关闭已获取所有行（或已调用 scalar()）的 ResultProxy 对象的光标。

+   **[连接/池/执行]**

    ResultProxy.fetchall()在内部使用 DBAPI fetchall()以提高效率，也添加到映射器迭代中（感谢 Michael Twomey）

### 通用

+   **[通用]**

    现在通过标准的 Python“logging”模块实现日志记录。“echo”关键字参数仍然有效，但为它们各自的类/实例设置/取消设置日志级别。所有日志记录可以通过直接在 Python API 中为“sqlalchemy”命名空间的记录器设置 INFO 和 DEBUG 级别来直接控制。类级别的日志记录在“sqlalchemy.<module>.<classname>”下，实例级别的日志记录在“sqlalchemy.<module>.<classname>.0x..<00-FF>”下。测试套件包括“--log-info”和“--log-debug”参数，它们独立于--verbose/--quiet 工作。在 orm 中添加了日志记录，以允许跟踪映射器配置，行迭代。

+   **[通用]**

    文档生成系统已经进行了全面改进，设计更简单，与 Markdown 更加集成

### orm

+   **[orm]**

    修改属性跟踪以更智能地检测更改，特别是对可变类型。TypeEngine 对象现在在定义如何比较两个标量实例方面发挥更大作用，包括通过 PickleType 实现的 MutableType 混合。工作单元现在跟踪“脏”列表，表示所有属性管理器检测到更改的持久对象的表达式。解决的基本问题是检测 PickleType 对象上的更改，但也将类型处理和“修改”对象检查泛化为更完整和可扩展。

+   **[orm]**

    对“属性加载器”和“选项”架构进行了广泛重构。ColumnProperty 和 PropertyLoader 通过可切换的“策略”定义其加载行为，MapperOptions 不再使用映射器/属性复制来运行；它们通过 QueryContext 和 SelectionContext 对象在查询/实例时间传播。已删除用于处理继承以及 options()的所有内部映射器和属性复制；映射器和属性的结构比以前简单得多，并且在新的“interfaces”模块中清晰地列出。

+   **[orm]**

    与映射器/属性重构相关，内部重构到映射器实例()方法，使用 SelectionContext 对象在操作期间跟踪状态。轻微的 API 破坏：由于更改，MapperExtension 上的 append_result()和 populate_instances()方法现在具有稍微不同的方法签名；希望这些方法目前还没有被广泛使用。

+   **[orm]**

    instances()方法现在移动到 Query 中，向后兼容版本仍保留在 Mapper 上。

+   **[orm]**

    添加了 contains_eager() MapperOption，与 instances()一起使用，指定应该从结果集中急切加载的属性，默认情况下使用它们的普通列名，或者根据自定义行转换函数进行转换。

+   **[orm]**

    对工作单元提交方案进行了更多重新排列，以更好地允许循环刷新内的依赖关系正常工作...更新了任务遍历/日志记录实现

+   **[orm]**

    多态映射器（即使用继承）现在按照所有继承类的表的顺序生成 INSERT 语句

    参考：[#321](https://www.sqlalchemy.org/trac/ticket/321)

+   **[orm]**

    添加了自动的“行切换”功能到映射，它将检测到具有相同标识键的待处理实例/已删除实例对，并将 INSERT/DELETE 转换为单个 UPDATE

+   **[orm]**

    “关联”映射简化以利用自动“行切换”功能

+   **[orm]**

    “自定义列表类”现在通过 relation()的“collection_class”关键字参数实现。旧方法仍然有效，但已被弃用。

    参考：[#212](https://www.sqlalchemy.org/trac/ticket/212)

+   **[orm]**

    在 relation()中添加了“viewonly”标志，允许构建对 flush()过程没有影响的关系。

+   **[orm]**

    向基本 Query select/get 函数添加了“lockmode”参数，包括“with_lockmode”函数以获取具有默认锁定模式的 Query 副本。将“read”/“update”参数转换为 select 侧的 for_update 参数。

    参考：[#292](https://www.sqlalchemy.org/trac/ticket/292)

+   **[orm]**

    在 Query/Mapper 中实现了“版本检查”逻辑，当 version_id_col 生效且使用 query.with_lockmode() 获取已加载实例时使用

+   **[orm]**

    改进了 post_update 行为；在不更新过多行的情况下，仅更新所需列

    参考：[#208](https://www.sqlalchemy.org/trac/ticket/208)

+   **[orm]**

    调整了 eager loading，使其“eager chain”与正常 mapper 设置保持分离，从而防止与延迟加载器操作发生冲突，修复

    参考：[#308](https://www.sqlalchemy.org/trac/ticket/308)

+   **[orm]**

    修复了延迟组加载的问题

+   **[orm]**

    session.flush() 不会关闭它打开的连接

    参考：[#346](https://www.sqlalchemy.org/trac/ticket/346)

+   **[orm]**

    向 mapper 添加了“batch=True”标志；如果为 False，save_obj 将完全一次保存一个对象，包括调用 before_XXXX 和 after_XXXX

+   **[orm]**

    向 mapper 添加了“column_prefix=None”参数；将给定字符串（通常为‘_’）预先添加到从 mapper 的 Table 设置的基于列的属性中

+   **[orm]**

    在 query.select() 的 from_obj 参数中指定连接将替换查询的主表，如果表在给定的 from_obj 中的某处。这样可以在查询中生成自定义连接和外连接，而不会使主表添加两次。

    参考：[#315](https://www.sqlalchemy.org/trac/ticket/315)

+   **[orm]**

    eagerloading 被调整以更加周到地将其 LEFT OUTER JOIN 附加到给定查询中，查找可能已经设置的自定义“FROM”子句。

+   **[orm]**

    在 SelectResults 中添加了 join_to 和 outerjoin_to 转换方法，用于根据属性名称构建连接/外连接条件。还添加了 select_from 方法，用于显式设置 from_obj 参数。

+   **[orm]**

    从 mapper 中移除了“is_primary”标志。

### sql

+   **[sql] [construction]**

    将“for_update”参数更改为接受 False/True/“nowait” 和 “read”，后两者仅由 Oracle 和 MySQL 解释

    参考：[#292](https://www.sqlalchemy.org/trac/ticket/292)

+   **[sql] [construction]**

    添加了对 SQL 方言的 extract() 函数（SELECT extract(field FROM expr)）的支持

+   **[sql] [construction]**

    BooleanExpression 包括新的“negate”参数，以指定适当的否定运算符（如果有的话）。

+   **[sql] [construction]**

    对“IN”或“IS”子句进行否定操作将导致“NOT IN”、“IS NOT”（而不是 NOT (x IN y)）。

+   **[sql] [construction]**

    Function 对象现在在 FROM 子句中知道该做什么。它们的行为应该是相同的，除了现在你还可以像这样做 select([‘*’], from_obj=[func.my_function()]) 从结果中获取多列，甚至使用 sql.column() 构造来命名返回列

    参考：[#172](https://www.sqlalchemy.org/trac/ticket/172)

### 模式

+   **[模式]**

    对模式包进行了相当多的清理，删除了模棱两可的方法，不再需要的方法。使用略微受限，更加强调明确性

+   **[模式]**

    Table 和其他可选项的 “primary_key” 属性成为了一个类似集合的 ColumnCollection 对象；它是有序的，但没有数值索引。从同一基础表派生的两个 pks 之间的比较子句（例如两个 Alias 对象）可以通过 table1.primary_key==table2.primary_key 生成

+   **[模式]**

    ForeignKey(Constraint) 支持 “use_alter=True”，以通过 ALTER 创建/删除外键。这允许设置循环外键关系。

+   **[模式]**

    从 Table 和 Column 中移除了 append_item() 方法；最好是内联构造 Table/Column/相关对象，但如果需要，可以使用 append_column()、append_foreign_key()、append_constraint() 等。

+   **[模式]**

    table.create() 现在不再返回 Table 对象，而是没有返回值。通常情况下，通过元数据创建表是可取的，因为它会处理表依赖关系。

+   **[模式]**

    添加了 UniqueConstraint（放置在 Table 级别）、CheckConstraint（放置在 Table 或 Column 级别）。

+   **[模式]**

    Column 上的 index=False/unique=True 现在会创建一个 UniqueConstraint，index=True/unique=False 创建一个普通 Index，Column 上的 index=True/unique=True 会创建一个唯一 Index。Column 上的 ‘index’ 和 ‘unique’ 关键字参数现在只能是布尔值；对于索引或唯一约束的显式名称和分组，使用 UniqueConstraint/Index 构造显式。

+   **[模式]**

    向 Column 添加了 autoincrement=True；如果显式设置为 False，则会禁用 postgres/mysql/mssql 的 SERIAL/AUTO_INCREMENT/identity seq 的模式生成

+   **[模式]**

    TypeEngine 对象现在具有处理其特定类型值的复制和比较方法。目前由 ORM 使用，请参见下文。

+   **[模式]**

    在反射期间出现了一个条件，在该条件中，当主键列被显式覆盖时，PrimaryKeyConstraint 会将反射和编程列都重复

+   **[模式]**

    Column 和 ColumnElement 上的 “foreign_key” 属性已被弃用，而是采用了基于 “foreign_keys” 列表/集合的属性，该属性考虑了一列上的多个外键。“foreign_key” 将返回 “foreign_keys” 列表/集合中的第一个元素，如果列表为空，则返回 None。

### sqlite

+   **[sqlite]**

    sqlite 布尔数据类型默认将 False/True 转换为 0/1

+   **[sqlite]**

    对日期/时间（SLDate/SLTime）类型进行修复；现在与 postgres 一样工作良好

    参考：[#335](https://www.sqlalchemy.org/trac/ticket/335)

### oracle

+   **[oracle]**

    Oracle 对 cx_Oracle.TIMESTAMP 有试验性支持，这需要在光标上调用 setinputsizes() 方法，现在通过 oracle 方言的 'auto_setinputsizes' 标志启用。

### 其他

+   **[ms-sql]**

    修复了 bug 261（对于 MS-SQL 区分大小写的数据库，表反射出现问题）

+   **[ms-sql]**

    现在可以为 pymssql 指定端口

+   **[ms-sql]**

    引入了新的“auto_identity_insert”选项，用于在为 IDENTITY 列指定值时自动切换到“SET IDENTITY_INSERT”模式

+   **[ms-sql]**

    现在支持多列外键

+   **[ms-sql]**

    修复了反射日期/日期时间列的问题

+   **[ms-sql]**

    添加了对 NCHAR 和 NVARCHAR 类型的支持

+   **[firebird]**

    别名不使用“AS”

+   **[firebird]**

    当反映不存在的表时，正确引发 NoSuchTableError

+   **[connections/pooling/execution]**

    连接池跟踪打开的游标，并在连接返回到连接池时自动关闭它们，如果仍然打开了游标。可以受到使其引发错误或不做任何操作的选项的影响。修复了 MySQL 等数据库的问题

+   **[connections/pooling/execution]**

    修复了 Connection 在提交/回滚后不会失去其 Transaction 的 bug

+   **[connections/pooling/execution]**

    在 ComposedSQLEngine、ResultProxy 中添加了 scalar() 方法

+   **[connections/pooling/execution]**

    当 ResultProxy 被关闭时，将关闭底层的游标。当 ResultProxy 对象的所有行都被提取（或者调用了 scalar() 方法）时，这将自动关闭游标。

+   **[connections/pooling/execution]**

    ResultProxy.fetchall() 内部使用 DBAPI 的 fetchall() 方法以提高效率，并添加到映射迭代中（由 Michael Twomey 提供）
