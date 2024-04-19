# 0.4 变更日志

> 原文：[`docs.sqlalchemy.org/en/20/changelog/changelog_04.html`](https://docs.sqlalchemy.org/en/20/changelog/changelog_04.html)

## 0.4.8

发布日期：2008 年 10 月 12 日星期日

### orm

+   **[orm]**

    修复了关于传递“A=B”与“B=A”时导致错误的 inherit_condition 的错误

    参考：[#1039](https://www.sqlalchemy.org/trac/ticket/1039)

+   **[orm]**

    在 SessionExtension.before_flush()中对新的、脏的和已删除的集合进行的更改将对该刷新生效。

+   **[orm]**

    向 InstrumentedAttribute 添加了 label()方法，以确保与 0.5 的向前兼容性。

### sql

+   **[sql]**

    column.in_(someselect)现在可以作为一个 columns-clause 表达式使用，而不会使子查询泄漏到 FROM 子句中

    参考：[#1074](https://www.sqlalchemy.org/trac/ticket/1074)

### mysql

+   **[mysql]**

    添加了 MSMediumInteger 类型。

    参考：[#1146](https://www.sqlalchemy.org/trac/ticket/1146)

### sqlite

+   **[sqlite]**

    提供了一个自定义的 strftime()函数，用于处理 1900 年之前的日期。

    参考：[#968](https://www.sqlalchemy.org/trac/ticket/968)

+   **[sqlite]**

    在 sqlite 方言中禁用了 String（和 Unicode，UnicodeText 等）的 convert_unicode 逻辑，以适应 pysqlite 2.5.0 的新要求，即仅接受 Python unicode 对象；[`itsystementwicklung.de/pipermail/list-pysqlite/2008-March/000018.html`](https://itsystementwicklung.de/pipermail/list-pysqlite/2008-March/000018.html)

### oracle

+   **[oracle]**

    has_sequence()现在考虑模式名称

    参考：[#1155](https://www.sqlalchemy.org/trac/ticket/1155)

+   **[oracle]**

    将 BFILE 添加到反射类型列表中

    参考：[#1121](https://www.sqlalchemy.org/trac/ticket/1121)

## 0.4.7p1

发布日期：2008 年 7 月 31 日星期四

### orm

+   **[orm]**

    向 scoped_session 方法添加了“add()”和“add_all()”。 0.4.7 的解决方法：

    ```py
    from sqlalchemy.orm.scoping import ScopedSession, instrument

    setattr(ScopedSession, "add", instrument("add"))
    setattr(ScopedSession, "add_all", instrument("add_all"))
    ```

+   **[orm]**

    修复了在 relation()中调用 set()和生成器表达式时与非 2.3 兼容的用法。

## 0.4.7

发布日期：2008 年 7 月 26 日星期六

### orm

+   **[orm]**

    当与 many-to-many 一起使用 contains()运算符时，将为 secondary（关联）表设置别名，以便多个 contains()调用不会互相冲突

    参考：[#1058](https://www.sqlalchemy.org/trac/ticket/1058)

+   **[orm]**

    修复了阻止 merge()与 comparable_property()一起运行的错误

+   **[orm]**

    现在，关于 relation()上的 enable_typechecks=False 设置仅允许具有继承映射器的子类型。 完全不相关的类型，或者没有针对目标映射器设置映射器继承的子类型仍然不被允许。

+   **[orm]**

    向 Sessions 添加了 is_active 标志，以检测事务是否正在进行中。 该标志始终为 True，对于“transactional”（在 0.5 中为非“autocommit”）会话。

    参考：[#976](https://www.sqlalchemy.org/trac/ticket/976)

### sql

+   **[sql]**

    修复了调用 select([literal('foo')])或 select([bindparam('foo')])时的错误。

### schema

+   **[schema]**

    如果表名或模式名包含的字符超过该方言配置的字符限制，则 create_all()、drop_all()、create()、drop()都会引发错误。一些数据库可以在使用过程中处理过长的表名，SQLA 也可以处理这个问题。但是由于我们正在 DB 的目录表中查找名称，因此各种反射/在创建期间进行 checkfirst 的场景失败。

    参考：[#571](https://www.sqlalchemy.org/trac/ticket/571)

+   **[schema]**

    当在 Column 上说“index=True”时生成的索引名称被截断为适合该方言的长度。此外，具有太长名称的索引不能使用 Index.drop()显式删除，类似于。

    参考：[#571](https://www.sqlalchemy.org/trac/ticket/571), [#820](https://www.sqlalchemy.org/trac/ticket/820)

### mysql

+   **[mysql]**

    将“CALL”添加到返回结果行的 SQL 关键字列表中。

### oracle

+   **[oracle]**

    Oracle 的 get_default_schema_name()在返回之前“规范化”名称，这意味着当检测到标识符为大小写不敏感时，它返回一个小写名称。

+   **[oracle]**

    在搜索现有表时，创建/删除表时考虑模式名称，以便其他所有者命名空间中具有相同名称的表不发生冲突。

    参考：[#709](https://www.sqlalchemy.org/trac/ticket/709)

+   **[oracle]**

    游标现在默认情况下将“arraysize”设置为 50，其值可以使用 Oracle 方言的 create_engine()的“arraysize”参数进行配置。这是为了考虑 cx_oracle 的默认设置为“1”，这会导致向 Oracle 发送许多往返。这实际上与 BLOB/CLOB 绑定的游标结合使用效果很好，其中有任意数量可用，但仅在该行请求的生命周期内（因此仍然需要 BufferedColumnRow，但不那么需要）。

    参考：[#1062](https://www.sqlalchemy.org/trac/ticket/1062)

+   **[oracle]**

    sqlite

    +   添加了 SLFloat 类型，与 SQLite REAL 类型亲和。以前只提供了 SLNumeric，它满足 NUMERIC 亲和，但与 REAL 不同。

### 杂项

+   **[postgres]**

    修复了 server_side_cursors 以正确检测 text()子句。

+   **[postgres]**

    添加了 PGCidr 类型。

    参考：[#1092](https://www.sqlalchemy.org/trac/ticket/1092)

## 0.4.6

发布日期：2008 年 5 月 10 日

### orm

+   **[orm]**

    修复了最近的 relation()重构，修复了在本地和远程表之间多次连接的奇异 viewonly 关系，这些关系之间有一个共同的列用于连接。

+   **[orm]**

    还重新建立了跨多个表连接的 viewonly relation()配置。

+   **[orm]**

    添加了一个实验性 relation()标志，以帮助跨函数等进行 primaryjoins，_local_remote_pairs=[tuples]。这补充了一个复杂的 primaryjoin 条件，允许您提供构成关系本地和远程侧的各个列对。还改进了惰性加载 SQL 生成，以处理将 bind 参数放置在函数和其他表达式内的情况。（部分进展）

    参考：[#610](https://www.sqlalchemy.org/trac/ticket/610)

+   **[orm]**

    修复了单表继承，使您可以从已加入表继承的映射器中单表继承而无需问题。

    参考：[#1036](https://www.sqlalchemy.org/trac/ticket/1036)

+   **[orm]**

    修复了如果发生了子句适应，Query.order_by()可能会出现的“连接元组”错误。

    参考：[#1027](https://www.sqlalchemy.org/trac/ticket/1027)

+   **[orm]**

    删除了关于映射的 selectables 需要“别名”的古老断言 - 如果没有，映射器现在会自己创建别名。尽管在这种情况下，您需要使用类，而不是映射的可选择项，作为列属性的来源 - 因此仍然会发出警告。

+   **[orm]**

    修复了涉及继承的“exists”函数的问题（any()、has()、~contains()）；完整的目标连接将被呈现到关系中，以链接到子类。

+   **[orm]**

    恢复了对主查询行使用 append_result()扩展方法的用法，当扩展存在且只返回单个实体结果时。

+   **[orm]**

    还重新建立了跨多个表连接的 viewonly relation()配置。

+   **[orm]**

    删除了关于映射的 selectables 需要“别名”的古老断言 - 如果没有，映射器现在会自己创建别名。尽管在这种情况下，您需要使用类，而不是映射的可选择项，作为列属性的来源 - 因此仍然会发出警告。

+   **[orm]**

    优化了 mapper._save_obj()，在 flush 期间不必要地调用 __ne__()来比较标量值。

    参考：[#1015](https://www.sqlalchemy.org/trac/ticket/1015)

+   **[orm]**

    添加了一个特性，使得贪婪加载中设置为 column_property()的子查询具有显式标签名称（顺便说一句，这不是必需的）时，当实例是贪婪连接的一部分时，标签会被匿名化，以防止与父对象上的同名子查询或列冲突。

    参考：[#1019](https://www.sqlalchemy.org/trac/ticket/1019)

+   **[orm]**

    基于集合的集合 |=、-=、^= 和 &= 对其操作数更加严格，只操作集合、frozensets 或集合类型的子类。以前，它们接受任何鸭式类型的集合。

+   **[orm]**

    添加了一个示例 dynamic_dict/dynamic_dict.py，演示了在 dynamic_loader 上放置字典行为的简单方法。

### sql

+   **[sql]**

    通过.collate(<collation>)表达式运算符和 collate(<expr>, <collation>) sql 函数添加了 COLLATE 支持。

+   **[sql]**

    修复了当应用于非表连接的选择语句时 union()的错误。

+   **[sql]**

    当作为 FROM 子句使用时，text()表达式的行为得到了改进，例如 select().select_from(text(“sometext”))

    参考：[#1014](https://www.sqlalchemy.org/trac/ticket/1014)

+   **[SQL]**

    Column.copy()尊重“autoincrement”的值，修复了与 Migrate 一起使用的问题

    参考：[#1021](https://www.sqlalchemy.org/trac/ticket/1021)

### mssql

+   **[mssql]**

    添加了“odbc_autotranslate”参数到引擎/dburi 参数。任何给定的字符串都将传递到 ODBC 连接字符串中：

    > ”AutoTranslate=%s” % odbc_autotranslate

    参考：[#1005](https://www.sqlalchemy.org/trac/ticket/1005)

+   **[mssql]**

    添加了“odbc_options”参数到引擎/dburi 参数。给定的字符串简单地附加到 SQLAlchemy 生成的 ODBC 连接字符串中。

    这应该消除将来需要添加大量 ODBC 选项的需求。

### 杂项

+   **[声明式] [扩展]**

    连接表继承映射器使用了一个稍微放松的函数来创建“继承条件”到父表，以便对尚未声明的 Table 对象的其他外键不会触发错误。

+   **[声明式] [扩展]**

    修复了在 ForeignKey 中使用已声明属性时的重入映射器编译挂起问题，即 ForeignKey(MyOtherClass.someattribute)

+   **[引擎]**

    现在可以将池监听器提供为可调用的字典或（可能是部分的）PoolListener 的鸭子类型，由您选择。

+   **[引擎]**

    添加了“rollback_returned”选项到 Pool，它将禁用在连接返回时发出的 rollback()。此标志仅在不支持事务的数据库（即 MySQL/MyISAM）中使用是安全的。

+   **[扩展]**

    基于集合的关联代理 |=，-=，^=和&=对其操作数更为严格，仅对集合，frozenset 或其他关联代理进行操作。以前，它们将接受任何鸭子类型的集合。

+   **[firebird]**

    处理“SUBSTRING(:string FROM :start FOR :length)”内置函数。

## 0.4.5

发布日期：2008 年 4 月 4 日星期五

### ORM

+   **[ORM]**

    对 session.merge()的行为进行了一点改变 - 现有对象是根据主键属性而不一定是 _instance_key 进行检查的。因此，广泛请求的功能是：

    > x = MyObject(id=1) x = sess.merge(x)

    如果存在，现在可以从数据库中加载具有 id＃1 的 MyObject。merge()仍然会将给定对象的状态复制到持久对象上，因此像上面的示例通常会将“x”的所有属性的“None”复制到持久副本上。这些可以使用 session.expire(x)来恢复。

+   **[ORM]**

    还修复了 merge()中的行为，即目标上存在但未在合并集合中的集合元素未从目标中删除。

+   **[ORM]**

    添加了对“未编译映射器”的更积极检查，特别有助于声明式层。

    参考：[#995](https://www.sqlalchemy.org/trac/ticket/995)

+   **[ORM]**

    “primaryjoin”/“secondaryjoin”背后的方法论已进行了重构。行为应该稍微更智能一些，主要表现在错误消息方面，已经简化为更易读的形式。在少量情况下，它可以更好地解析正确的外键。

+   **[orm]**

    添加 comparable_property()，将查询比较器行为添加到常规的、未受管理的 Python 属性中

+   **[orm] ['machines'] [Company.employees.of_type(Engineer)]**

    查询.with_polymorphic() 的功能已添加到 mapper() 中作为配置选项。

    它通过几种形式设置：

    with_polymorphic=' * ' with_polymorphic=[mappers] with_polymorphic=(' * ', selectable) with_polymorphic=([mappers], selectable)

    这控制了继承映射器的默认多态加载策略。当没有给定可选择项时，将为所有请求的联接表继承映射器创建外连接。请注意，联接的自动创建与具体表继承不兼容。

    现有的 mapper() 上的 select_table 标志现在已弃用，并且与 with_polymorphic(' * ', select_table) 同义。请注意，select_table 的底层“内涵”已被完全删除并替换为更新、更灵活的方法。

    新方法还自动允许对子类进行及早加载，如果它们存在的话，例如：

    ```py
    sess.query(Company).options(eagerload_all())
    ```

    以加载 Company 对象、其员工和作为工程师的员工的‘machines’集合为例。很快还应该引入一个“with_polymorphic”查询选项，该选项将允许对关系上的 with_polymorphic() 进行每个查询的控制。

+   **[orm]**

    向查询中添加了两个“实验性”功能，“实验性”是因为它们的具体名称/行为还没有最终确定：_values() 和 _from_self()。我们希望得到关于这些功能的反馈。

    +   _values(*columns) 接收列表达式列表，并返回一个只返回这些列的新查询。在求值时，返回值是一个元组列表，就像使用 add_column() 或 add_entity() 时一样，唯一的区别是“实体零”，即映射类，不包含在结果中。这意味着现在终于可以在 Query 上使用 group_by() 和 having()，这些功能到目前为止一直没有用。

        该方法的未来更改可能包括删除其加入、过滤和允许其他与“结果集”无关的选项的能力，因此我们寻求的反馈是人们希望如何使用 _values()……即在最后，还是人们更喜欢在调用后继续生成。

    +   _from_self()编译 Query 的 SELECT 语句（减去任何急加载器），并返回一个从该 SELECT 选择的新 Query。所以基本上你可以从一个 Query 中查询而不需要手动提取 SELECT 语句。这赋予了 query[3:5]._from_self().filter(some criterion)之类的操作意义。这里没有太多争议，除了你可以快速创建效率较低的高度嵌套查询，我们希望对命名选择进行反馈。

+   **[orm]**

    query.order_by()和 query.group_by()将接受多个参数使用*args（就像 select()已经做的那样）。

+   **[orm]**

    向 Query 添加了一些便利描述符：query.statement 返回完整的 SELECT 结构，query.whereclause 仅返回 SELECT 结构的 WHERE 部分。

+   **[orm]**

    修复/覆盖了使用 False/0 值作为多态鉴别器时的情况。

+   **[orm]**

    修复了阻止 synonym()属性与继承一起使用的错误

+   **[orm]**

    修复了 SQL 函数截断尾随下划线的问题

    参考：[#996](https://www.sqlalchemy.org/trac/ticket/996)

+   **[orm]**

    当挂起实例上的属性过期时，当触发“refresh”操作并且找不到结果时不会引发错误。

+   **[orm]**

    Session.execute 现在可以从元数据中找到绑定

+   **[orm]**

    调整了“自引用”的定义，即任何具有共同父级的两个映射器（这会影响是否需要在与 Query 连接时需要 aliased=True）。

+   **[orm]**

    对 query.join()中的“from_joinpoint”参数进行了一些修复，以便如果前一个连接被别名化而这个连接没有，连接仍然成功进行。

+   **[orm]**

    各种“级联删除”修复：

    +   修复了动态关系的“级联删除”操作，该操作仅在 0.4.2 中实现了外键空值行为，而不是实际级联删除

    +   在多对一上没有 delete-orphan 级联的删除级联将不会删除在调用 session.delete()之前从父级断开连接的孤立节点（一对多已经有了这个）。

    +   在删除级联与 delete-orphan 级联时，无论它是否仍附加到也已删除的父级，都将删除孤立节点。

    +   在使用继承时，可以正确检测到存在于超类上的关系上的 delete-orphan 级联。

    参考：[#895](https://www.sqlalchemy.org/trac/ticket/895)

+   **[orm]**

    修复了在使用 select_from()时，Query 中正确别名映射器配置的 order_by 计算顺序。

+   **[orm]**

    重构了在用另一个集合替换一个集合时触发的差异逻辑为 collections.bulk_replace，对于构建多级集合的任何人都很有用。

+   **[orm]**

    将级联遍历算法从递归转换为迭代，以支持深层对象图。

### sql

+   **[sql]**

    在模式限定的表中，现在将模式名称放在所有列表达式以及生成列标签时的表名之前。这在所有情况下都可以防止跨模式名称冲突

    参考：[#999](https://www.sqlalchemy.org/trac/ticket/999)

+   **[sql]**

    现在可以允许选择所有 FROM 子句相关联且自身没有 FROM 的查询。这些通常在标量上下文中使用，即 SELECT x, (SELECT x WHERE y) FROM table。需要显式调用 correlate()。

+   **[sql]**

    ‘name’不再是 Column()的必需构造参数。现在可以推迟直到列添加到表中。

+   **[sql]**

    like()，ilike()，contains()，startswith()，endswith()接受一个可选的关键字参数“escape=<somestring>”，使用语法“x LIKE y ESCAPE ‘<somestring>’”设置为转义字符。

    参考：[#791](https://www.sqlalchemy.org/trac/ticket/791)，[#993](https://www.sqlalchemy.org/trac/ticket/993)

+   **[sql]**

    random()现在是一个通用的 sql 函数，并将编译为数据库的随机实现（如果有）。

+   **[sql]**

    update().values()和 insert().values()接受关键字参数。

+   **[sql]**

    修复了 select()在生成 FROM 子句方面的问题，在罕见情况下，可能会产生两个子句，而本意是取消另一个。一些具有大量急加载的 ORM 查询可能会看到这种症状。

+   **[sql]**

    case()函数现在还将字典作为其 whens 参数。它还默认将“THEN”表达式解释为值，这意味着 case([(x==y, “foo”)])将“foo”解释为绑定值，而不是 SQL 表达式。在这种情况下，对于标准本身，如果“value”关键字存在，这些可能仅为文字字符串，否则 SA 将强制使用 text()或 literal()。

### mysql

+   **[mysql]**

    方言用于缓存服务器设置的连接.info 键已更改并现在已命名空间化。

### mssql

+   **[mssql]**

    反映的表现在将自动加载其他被自动加载表中的外键引用的表。

    参考：[#979](https://www.sqlalchemy.org/trac/ticket/979)

+   **[mssql]**

    添加了 executemany 检查以跳过标识获取。

    参考：[#916](https://www.sqlalchemy.org/trac/ticket/916)

+   **[mssql]**

    添加了小日期类型的存根。

    参考：[#884](https://www.sqlalchemy.org/trac/ticket/884)

+   **[mssql]**

    为 pyodbc 方言添加了一个新的‘driver’关键字参数。如果给定，将替换为 ODBC 连接字符串，默认为‘SQL Server’。

+   **[mssql]**

    为 pyodbc 方言添加了一个新的‘max_identifier_length’关键字参数。

+   **[mssql]**

    对 pyodbc + Unix 的改进。如果以前无法使该组合工作，请再试一次。

### oracle

+   **[oracle]**

    Table 上的“owner”关键字现在已被弃用，并且与“schema”关键字完全同义。现在可以使用替代的“owner”属性反映表，明确在 Table 对象上声明或不使用“schema”。

+   **[oracle]**

    在表反射期间，所有“魔术”搜索同义词、DBLINK 等的功能默认情况下都已禁用，除非在 Table 对象上指定了 “oracle_resolve_synonyms=True”。解析同义词必然会导致一些混乱的猜测，我们宁愿默认情况下将其关闭。当设置了标志时，表和相关表将在所有情况下针对同义词进行解析，这意味着如果特定表存在同义词，则在反射相关表时将使用它。这比以前的行为更粘性，这就是为什么默认情况下关闭的原因。

### 杂项

+   **[declarative] [extension]**

    “synonym” 函数现在可以直接与 “declarative” 一起使用。通过使用“descriptor” 关键字参数传入装饰的属性，例如：`somekey = synonym('_somekey', descriptor=property(g, s))`。

+   **[declarative] [extension]**

    “deferred” 函数可以与 “declarative” 一起使用。最简单的用法是一起声明 `deferred` 和 `Column`，例如：`data = deferred(Column(Text))`

+   **[declarative] [extension]**

    Declarative 还增加了 `@synonym_for(…)` 和 `@comparable_using(…)`，作为 `synonym` 和 `comparable_property` 的前端。

+   **[declarative] [extension]**

    在使用 declarative 时，对映射器编译进行了改进；已经编译的映射器在使用时仍会触发其他未编译的映射器的编译。

    参考：[#995](https://www.sqlalchemy.org/trac/ticket/995)

+   **[declarative] [extension]**

    Declarative 将为缺少名称的列完成设置，允许更干燥的语法。

    > `class Foo(Base):`
    > 
    > `__tablename__ = 'foos' id = Column(Integer, primary_key=True)`

+   **[declarative] [extension]**

    在 declarative 中，可以通过将 “inherits=None” 发送到 `__mapper_args__` 来禁用继承。

+   **[declarative] [extension]**

    `declarative_base()` 接受可选的关键字参数“mapper”，这是任何可调用/类/方法，用于生成映射器，例如 `declarative_base(mapper=scopedsession.mapper)`。这个属性也可以在单独的声明类中使用“__mapper_cls__”属性进行设置。

+   **[postgres]**

    修复了 PG 服务器端游标的问题，将固定单元测试作为默认测试套件的一部分。为游标 ID 添加了更好的唯一性。

    参考：[#1001](https://www.sqlalchemy.org/trac/ticket/1001)

## 0.4.4

发布日期：2008 年 3 月 12 日（星期三）

### orm

+   **[orm]**

    `any()`, `has()`, `contains()`, `~contains()`, 属性级别的 `==` 和 `!=` 现在可以正确地与自引用关系一起使用 - 在 EXISTS 内部的子句在“remote”一侧被别名化，以区分它与父表。这适用于单表自引用以及基于继承的自引用。

+   **[orm]**

    修复了在与 NULL 比较一对一关系时，关系() 级别的 == 和 != 运算符的行为。

    参考：[#985](https://www.sqlalchemy.org/trac/ticket/985)

+   **[orm]**

    修复了一个错误，即当由 select_table 映射的多态映射实例未加载 session.expire() 属性时。 

+   **[orm]**

    添加了 query.with_polymorphic() - 指定了从基类继承的类列表，这些类将被添加到查询的 FROM 子句中。允许在过滤器()条件中使用子类，并且急切加载这些子类的属性。

+   **[orm]**

    你的呼声已经被听到：使用 delete-orphan 从属性或集合中删除挂起的项目会将该项目从会话中删除；不会引发 FlushError。请注意，如果您显式地 session.save()了挂起的项目，则属性/集合的移除仍会将其排除。

+   **[orm]**

    当在会话中调用 session.refresh()和 session.expire()时，对于不在会话中持久存在的实例会引发错误。

+   **[orm]**

    当使用 join()生成多个 Query 对象时，可能会出现生成性 bug。

+   **[orm]**

    修复了在 0.4.3 中引入的 bug，当使用默认的“select” polymorphic_fetch 时，加载已持久化的实例映射到连接表继承时，会触发一个无用的从其连接表加载的“次要”加载。这是由于属性在第一次加载时被标记为过期，并且在上一个“次要”加载中未被标记为未过期。属性现在基于任何加载或提交操作成功后在 __dict__ 中的存在而被标记为未过期。

+   **[orm]**

    废弃的 Query 方法 apply_sum()、apply_max()、apply_min()、apply_avg()。更好的方法学正在到来……。

+   **[orm]**

    relation()可以接受一个可调用对象作为其第一个参数，该可调用对象返回要相关联的类。这是为了帮助声明式包在类尚未就位时定义关系。

+   **[orm]**

    添加了一个新的“更高级别”的操作符称为“of_type()”：在 join()中以及与 any()和 has()一起使用时，用于限定将在过滤条件中使用的子类，例如：

    > query.filter(Company.employees.of_type(Engineer).
    > 
    > any(Engineer.name==’foo’))
    > 
    > 或
    > 
    > query.join(Company.employees.of_type(Engineer)).
    > 
    > filter(Engineer.name==’foo’)

+   **[orm]**

    针对 flush()中潜在的丢失引用错误的预防性代码。

+   **[orm]**

    在 filter()、filter_by()等中使用的表达式，当它们使用从关系生成的子对象的标识生成的子句时（例如，filter(Parent.child==<somechild>))，在执行时评估<somechild>的实际主键值，以便 Query 的自动刷新步骤可以完成，从而在<somechild>是挂起的情况下填充<somechild>的 PK 值。

+   **[orm]**

    将关系级别的 order by 设置为“secondary”表中的列现在可以与急切加载一起使用，以前的“order by”不会对次要表的别名进行别名处理。

+   **[orm]**

    现在，基于现有描述符的同义词现在是这些描述符的完全代理。

### sql

+   **[sql]**

    可以再次对文本 FROM 子句创建选择的别名。

    参考：[#975](https://www.sqlalchemy.org/trac/ticket/975)

+   **[sql]**

    bindparam() 的值可以是可调用的，这样在语句执行时会被评估以获取值。

+   **[SQL]**

    添加了对结果集获取的异常包装/重新连接支持。重新连接适用于那些在结果集期间引发可捕获数据错误的数据库（即在 MySQL 上不起作用）。

    参考：[#978](https://www.sqlalchemy.org/trac/ticket/978)

+   **[SQL]**

    实现了“threadlocal”引擎的两阶段 API，通过 engine.begin_twophase()，engine.prepare()

    参考：[#936](https://www.sqlalchemy.org/trac/ticket/936)

+   **[SQL]**

    修复了阻止 UNION 可克隆的错误。

    参考：[#986](https://www.sqlalchemy.org/trac/ticket/986)

+   **[SQL]**

    在 insert()、update()、delete() 和 DDL() 中添加了“bind”关键字参数。现在这些语句上的 .bind 属性也可以赋值，就像在 select() 上一样。

+   **[SQL]**

    现在可以在 INSERT 和 INTO 之间编译带有额外“前缀”单词的插入语句，用于供应商扩展，如 MySQL 的 INSERT IGNORE INTO table。

### 扩展

+   **[扩展]**

    添加了一个新的超小型“声明式”扩展，允许在类声明下方内联进行 Table 和 mapper() 配置。此扩展与 ActiveMapper 和 Elixir 不同之处在于它根本不重新定义任何 SQLAlchemy 语义；字面 Column、Table 和 relation() 构造用于定义类行为和表定义。

### 杂项

+   **[方言]**

    无效的 SQLite 连接 URL 现在会引发错误。

+   **[方言]**

    postgres TIMESTAMP 现在可以正确渲染

    参考：[#981](https://www.sqlalchemy.org/trac/ticket/981)

+   **[方言]**

    postgres PGArray 默认是“可变”类型；在与 ORM 一起使用时，使用可变风格的相等性/写时复制技术来测试更改。

## 0.4.3

发布日期：Thu Feb 14 2008

### 通用

+   **[通用]**

    修复了许多隐藏的以及一些不那么隐藏的兼容性问题，感谢对 Python 2.3 的新支持，使得完整测试套件可以在 2.3 上运行。

+   **[通用]**

    现在警告被作为类型异常.SAWarning 发出。

### ORM

+   **[ORM]**

    每个 Session.begin() 现在必须伴随相应的 commit() 或 rollback()，除非会话使用 Session.close() 关闭。这也包括使用 transactional=True 创建的会话隐式的 begin()。这里引入的最大变化是，当使用 transactional=True 创建的会话在 flush() 期间引发异常时，您必须调用 Session.rollback() 或 Session.close() 以便该会话在异常后继续。

+   **[ORM]**

    修复了在合并瞬时实体与带有反向引用集合的实体时出现的 merge() 集合加倍错误。

    参考：[#961](https://www.sqlalchemy.org/trac/ticket/961)

+   **[ORM]**

    merge(dont_load=True) 不接受瞬时实体，这与 merge(dont_load=True) 不接受任何“脏”对象的事实相一致。

+   **[ORM]**

    添加了由 scoped_session 生成的独立“query”类属性。这提供了 MyClass.query，而不使用 Session.mapper。通过以下方式使用：

    > MyClass.query = Session.query_property()

+   **[orm]**

    当尝试在没有会话的情况下访问过期实例属性时，会引发适当的错误消息

+   **[orm]**

    dynamic_loader() / lazy=”dynamic”现在接受并使用 order_by 参数，方式与 relation()中的工作方式相同。

+   **[orm]**

    添加了 expire_all()方法到 Session。为所有持久实例调用 expire()。这在与...一起使用时很方便。

+   **[orm]**

    部分或完全过期的实例在影响这些对象的常规查询操作期间将其过期属性填充，从而防止每个实例需要不必要的第二个 SQL 语句。

+   **[orm]**

    动态关系在引用时会创建对父对象的强引用，以便查询仍然有一个父对象可以调用，即使父对象只在单个表达式的范围内创建（并且在其他情况下被取消引用）。

    参考：[#938](https://www.sqlalchemy.org/trac/ticket/938)

+   **[orm]**

    添加了一个 mapper()标志“eager_defaults”。当设置为 True 时，在 INSERT 或 UPDATE 操作期间生成的默认值会立即后获取，而不是推迟到以后。这模仿了旧的 0.3 行为。

+   **[orm]**

    query.join()现在可以接受类映射的属性作为参数。这些可以用于替代或与字符串任意组合。特别是这允许在多态关系的子类上构建连接，即：

    > query(Company).join([‘employees’, Engineer.name])

+   **[orm] [(‘employees’] [Engineer.name] [people.join(engineer))]**

    query.join()还可以接受属性名称/某些可选择的元组作为参数。这允许构建从多态关系的子类连接的连接，即：

    > query(Company).join(
    > 
    > )

+   **[orm]**

    与多态映射器一起使用 join()的行为的一般改进，即从/到多态映射器的连接，并正确应用别名。

+   **[orm]**

    当映射器确定映射连接的自然“主键”时，修复/改进了行为，它将更有效地减少通过外键关系等效的列。这会影响需要发送给 query.get()的参数数量，等等。

    参考：[#933](https://www.sqlalchemy.org/trac/ticket/933)

+   **[orm]**

    惰性加载器现在可以处理连接条件，其中“绑定”列（即将父 id 作为绑定参数发送的列）在连接条件中出现多次。具体来说，这允许包含父相关子查询的 relation()的常见任务，例如“仅选择最近的子项”。

    参考：[#946](https://www.sqlalchemy.org/trac/ticket/946)

+   **[orm]**

    修复了多态继承中的一个 bug，当基本的多态 _on 列不对应于继承映射器的本地可选择的任何列时，引发了不正确的异常，超过一层。

+   **[orm]**

    修复了多态继承中的一个 bug，这使得很难在多态映射器上设置一个工作的 “order_by”。

+   **[orm]**

    修复了 Query 中的一个相当昂贵的调用，该调用减慢了多态查询的速度。

+   **[orm]**

    如果需要，”被动默认值“ 和其他 ”内联“ 默认值现在可以在 flush() 调用时加载；特别是，这允许构建关系()，其中外键列引用服务器端生成的非主键列。

    参考：[#954](https://www.sqlalchemy.org/trac/ticket/954)

+   **[orm]**

    其他 Session 事务修复/更改：

    +   修复了会话事务管理中的一个 bug：当向嵌套事务添加连接时，父事务没有在连接上启动。

    +   现在 `session.transaction` 总是指向最内层的活动事务，即使在会话事务对象上直接调用 commit/rollback。

    +   两阶段事务现在可以被准备。

    +   当在一个连接上准备两阶段事务失败时，所有的连接都会回滚。

    +   当使用嵌套事务时，`session.close()` 没有关闭所有事务。

    +   rollback() 以前错误地将当前事务直接设置为可以回滚的事务的父事务。现在它将回滚到能够处理它的下一个事务，但将当前事务设置为其父事务，并且使之间的事务无效。无效的事务只能回滚或关闭，任何其他调用都会导致错误。

    +   对于 `commit()`，autoflush 没有对简单的子事务进行刷新。

    +   `unitofwork flush` 在会话不在事务中且提交事务失败时，没有关闭失败的事务。

+   **[orm]**

    杂项票据：

    参考：[#940](https://www.sqlalchemy.org/trac/ticket/940), [#964](https://www.sqlalchemy.org/trac/ticket/964)

### sql

+   **[sql]**

    添加了 “schema.DDL”，一个可执行的自由格式 DDL 语句。DDL 可以独立执行，也可以附加到 Table 或 MetaData 实例上，并在创建和/或删除这些对象时自动执行。

+   **[sql]**

    可以使用 ‘useexisting=True’ 标志覆盖表列和约束，该标志现在考虑与其一起传递的参数。

+   **[sql]**

    添加了基于可调用的 DDL 事件接口，添加了在创建和删除 Tables 和 MetaData 之前和之后的钩子。

+   **[sql]**

    添加了 `where(<criterion>)` 方法到 `delete()` 和 `update()` 构造函数中，该方法返回一个新对象，其中的条件与现有条件通过 AND 连接，就像 `select().where()` 一样。

+   **[sql]**

    向列操作添加了 “ilike()” 操作符。在 postgres 上编译为 ILIKE，在其他所有地方为 lower(x) LIKE lower(y)。

    参考：[#727](https://www.sqlalchemy.org/trac/ticket/727)

+   **[sql]**

    添加了“now()”作为通用函数；在 SQLite、Oracle 和 MSSQL 上编译为“CURRENT_TIMESTAMP”；在其他所有情况下为“now()”。

    参考：[#943](https://www.sqlalchemy.org/trac/ticket/943)

+   **[sql]**

    startswith()、endswith()和 contains()运算符现在在 SQL 中将通配符运算符与给定操作数连接起来，即在所有情况下为“’%’ || <bindparam>”，正确接受 text(‘something’)操作数

    参考：[#962](https://www.sqlalchemy.org/trac/ticket/962)

+   **[sql]**

    cast()正确接受 text(‘something’)和其他非文字操作数

    参考：[#962](https://www.sqlalchemy.org/trac/ticket/962)

+   **[sql]**

    修复了结果代理中的错误，匿名生成的列标签将无法使用其直接字符串名称访问

+   **[sql]**

    可以定义可延迟的约束。

+   **[sql]**

    在 select()和 text()中添加了“autocommit=True”关键字参数，以及 select()上的生成 autocommit()方法；对于通过某些用户定义的方式修改数据库的语句，而不是通常的 INSERT/UPDATE/DELETE 等。如果没有进行中的事务，则此标志将在执行期间启用“autocommit”行为。

    参考：[#915](https://www.sqlalchemy.org/trac/ticket/915)

+   **[sql]**

    可选择对象上的‘.c.’属性现在为其列子句中的每个列表达式添加一个条目。以前，“未命名”列如函数和 CASE 语句没有被放在那里。现在它们将被放在那里，如果没有‘name’可用，则使用它们的完整字符串表示。

+   **[sql]**

    CompositeSelect，即任何 union()、union_all()、intersect()等现在断言每个可选择对象包含相同数量的列。这符合相应的 SQL 要求。

+   **[sql]**

    为否则未标记的函数和表达式生成的匿名‘label’现在在编译时向外传播，例如 select([select([func.foo()])])的表达式。

+   **[sql]**

    在上述思想的基础上，CompositeSelects 现在根据第一个可选择对象中存在的名称构建其“.c.”集合；corresponding_column()现在对所有嵌入式可选择对象完全有效。

+   **[sql]**

    Oracle 和其他数据库现在正确编码用于序列等默认值的 SQL，即使没有使用 unicode 标识符，因为标识符准备器可能返回缓存的 unicode 标识符。

+   **[sql]**

    表和子句与表达式左侧的 datetime 对象现在可以比较（d < table.c.col）。（右侧的 datetimes 一直有效，左侧的异常是 datetime 实现的怪癖。）

### 杂项

+   **[方言]**

    SQLite 中对模式的更好支持（通过 ATTACH DATABASE … AS name 链接）。在过去的一些情况下，SQLite 生成的 SQL 中省略了模式名称。���在不再这样。

+   **[方言]**

    SQLite 上的 table_names 现在也会选择临时表。

+   **[方言]**

    在反射操作期间自动检测未指定的 MySQL ANSI_QUOTES 模式，支持在中间更改模式。如果不使用反射，则仍然需要手动设置模式。

+   **[方言]**

    修复了 SQLite 上 TIME 列的反射。

+   **[方言]**

    最终将 PGMacAddr 类型添加到 postgres 中

    参考：[#580](https://www.sqlalchemy.org/trac/ticket/580)

+   **[方言]**

    在 Firebird 下反映与 PK 字段关联的序列（通常具有 BEFORE INSERT 触发器）

+   **[方言]**

    在生成 LIMIT/OFFSET 子查询时，Oracle 组装结果集列映射的正确列，即使长名称截断也允许列正确映射到结果集

    参考：[#941](https://www.sqlalchemy.org/trac/ticket/941)

+   **[方言]**

    MSSQL 现在在 _is_select 正则表达式中包含 EXEC，这应该允许使用返回行的存储过程。

+   **[方言]**

    MSSQL 现在包含对 ANSI SQL row_number() 函数的 LIMIT/OFFSET 的实验性实现，因此需要 MSSQL-2005 或更高版本。要启用此功能，请将 “has_window_funcs” 添加到 connect 的关键字参数中，或将 “?has_window_funcs=1” 添加到您的 dburi 查询参数中。

+   **[扩展]**

    更改 ext.activemapper 以使用非事务性会话进行对象存储。

+   **[扩展]**

    修复了关联代理列表上的 “[‘a’] + obj.proxied” 二元操作的输出顺序。

## 0.4.2p3

发布日期：2008 年 1 月 9 日

### 通用

+   **[通用]**

    子版本编号方案更改为适应 setuptools 版本号规则；easy_install -u 现在应该获取此版本而不是 0.4.2。

### ORM

+   **[ORM]**

    在使用“可变标量”（如 PickleTypes）时，修复了关于 session.dirty 的错误

+   **[ORM]**

    当在主或次要关联条件中使用非本地映射列刷新时，添加了更详细的错误消息

+   **[ORM]**

    现在在 InstanceState.__cleanup() 中抑制 *所有* 错误。

+   **[ORM]**

    修复了属性历史记录错误，即对于已经有挂起更改的基于集合的属性，将新集合分配给该属性会生成不正确的历史记录

    参考：[#922](https://www.sqlalchemy.org/trac/ticket/922)

+   **[ORM]**

    修复了 delete-orphan 级联错误，即将相同对象两次设置到标量属性可能会将其记录为孤儿

    参考：[#925](https://www.sqlalchemy.org/trac/ticket/925)

+   **[ORM]**

    修复了对基于列表的关系的 += 赋值上的级联。

+   **[ORM]**

    现在可以对尚不存在的属性创建同义词，稍后通过 add_property() 添加。这通常包括反向引用。（即您可以创建反向引用的同义词，而不必担心操作顺序）

    参考：[#919](https://www.sqlalchemy.org/trac/ticket/919)

+   **[ORM]**

    修复了可能发生的多态“union”映射器的错误，该错误会回退到继承表的“延迟”加载

+   **[ORM]**

    映射器/映射类（即‘c’）上的“columns”集合针对映射表，而不是在多态“union”加载情况下的 select_table（这不应该是可察觉的）。

+   **[orm]**

    修复了一个相当关键的 bug，即相同实例可能会在 unitofwork.new 集合中列出多次；最常见的情况是在使用继承映射器和 ScopedSession.mapper 的组合时，每个实例的多个 __init__ 调用可能会使用不同的 _state 对象保存()对象

+   **[orm]**

    向 Query 添加了非常基本的迭代器行为。调用 query.yield_per(<number of rows>)并在迭代上下文中评估 Query；每个 N 行的集合将被打包并 yield。请极度谨慎使用此方法，因为它不会尝试在结果批次边界上协调急切加载的集合，也不会在同一批次中出现相同实例时表现良好。这意味着如果在多个批次中引用了急切加载的集合，它将被清除，并且在所有情况下，如果在多个批次中出现相同实例，属性将被覆盖。

+   **[orm]**

    为集合和关联代理集合提供了固定的原地设置变异操作符。

    参考：[#920](https://www.sqlalchemy.org/trac/ticket/920)

### sql

+   **[sql]**

    现在 Text 类型已正确导出，并且在 DDL 创建时不会引发警告；没有长度的 String 类型仅在 CREATE TABLE 时引发警告

    参考：[#912](https://www.sqlalchemy.org/trac/ticket/912)

+   **[sql]**

    添加了新的 UnicodeText 类型，用于指定编码的、无长度的 Text 类型

+   **[sql]**

    修复了 union()中的 bug，使得不从 FromClause 对象派生的 select()语句可以进行 union 操作

+   **[sql]**

    将 TEXT 的名称更改为 Text，因为它是一个“通用”类型；TEXT 名称在 0.5 版本之前已被弃用。当没有长度时，String 转为 Text 的“升级”行为也在 0.5 版本之前被弃用；在用于 CREATE TABLE 语句时将发出警告（在 SQL 表达式目的上没有长度的 String 仍然可以使用）

    参考：[#912](https://www.sqlalchemy.org/trac/ticket/912)

+   **[sql]**

    修复了 generative select.order_by(None) / group_by(None)未能重置 order by/group by 条件的问题

    参考：[#924](https://www.sqlalchemy.org/trac/ticket/924)

### 杂项

+   **[dialects]**

    修复了 mysql 空字符串列默认值的反射。

+   **[ext]**

    为关联代理列表添加了‘+’、‘*’、‘+=’和‘*=’支持。

+   **[dialects]**

    mssql - 缩小了对 MSDate/MSDateTime 子类中“date”/“datetime”的测试范围，以防止传入的“datetime”对象被错误解释为“date”对象，反之亦然。

    参考：[#923](https://www.sqlalchemy.org/trac/ticket/923)

+   **[dialects]**

    修复了 PGArray 类型的 subtype 结果处理器的缺失调用。

    参考：[#913](https://www.sqlalchemy.org/trac/ticket/913)

## 0.4.2

发布日期：2008 年 1 月 2 日（星期三）

### orm

+   **[orm]**

    针对基于集合的反向引用的主要行为更改：它们不再触发延迟加载！“reverse”添加和删除被排队，并在实际读取和加载集合时与集合合并；但在此之前不会触发加载。对于注意到这种行为的用户，在某些情况下，这应该比在某些情况下使用动态关系更方便；对于那些没有注意到的用户，您可能会注意到您的应用程序在某些情况下使用的查询比以前少得多。

    参考：[#871](https://www.sqlalchemy.org/trac/ticket/871)

+   **[orm]**

    添加了可变主键支持。主键列可以自由更改，并且在刷新时实例的标识将发生变化。此外，支持沿关系更新外键引用（主键或非主键），可以与数据库的 ON UPDATE CASCADE（对于像 Postgres 这样的数据库是必需的）一起使用，或者直接由 ORM 以 UPDATE 语句的形式发出，通过设置标志“passive_cascades=False”。

+   **[orm]**

    继承映射器现在直接继承其父映射器的 MapperExtensions，因此特定 MapperExtension 的所有方法也会为子类调用。与往常一样，任何 MapperExtension 都可以返回 EXT_CONTINUE 以继续扩展处理，或者返回 EXT_STOP 以停止处理。映射器解析的顺序为：<在类的映射器上声明的扩展> <在类的父映射器上声明的扩展> <全局声明的扩展>。

    请注意，如果您单独实例化相同的扩展类，然后分别将其应用于同一继承链中两个映射器，扩展将应用于继承类两次，并且每个方法将被调用两次。

    要显式地将映射扩展应用于每个继承类，但每个方法每次操作只调用一次，请为两个映射器使用相同的扩展实例。

    参考：[#490](https://www.sqlalchemy.org/trac/ticket/490)

+   **[orm]**

    现在对 MapperExtension.before_update()和 after_update()进行对称调用；以前，一个没有修改列属性（但有关系()修改）的实例可能会调用 before_update()但不调用 after_update()

    参考：[#907](https://www.sqlalchemy.org/trac/ticket/907)

+   **[orm]**

    查询语句中缺失的列现在在加载时会自动延迟加载。

+   **[orm]**

    扩展“object”并且在实例构造时没有提供 __init__()方法的映射类现在会在实例构造时引发 TypeError，如果存在非空*args 或**kwargs（并且不被任何扩展（如 scoped_session mapper）消耗），与普通 Python 类的行为一致

    参考：[#908](https://www.sqlalchemy.org/trac/ticket/908)

+   **[orm]**

    修复了当 filter_by()将关系与 None 进行比较时的 Query bug

    参考：[#899](https://www.sqlalchemy.org/trac/ticket/899)

+   **[orm]**

    改进了对映射实体的 pickling 支持。现在，每个实例的惰性/延迟/过期可调用对象都是可序列化的，因此它们可以与 _state 一起序列化和反序列化。

+   **[orm]**

    新的 synonym()行为：如果映射类上不存在属性，则属性将放置在映射类上。如果类上已经存在属性，则 synonym 将使用适当的比较运算符装饰属性，以便它可以像任何其他映射属性一样在列表达式中使用（即可在 filter()中使用等）。“proxy=True”标志已弃用，不再具有任何意义。此外，“map_column=True”标志将自动生成与同义词名称相对应的 ColumnProperty，即：'somename':synonym('_somename', map_column=True)将把名为'somename'的列映射到属性'_somename'。请参阅映射器文档中的示例。

    参考：[#801](https://www.sqlalchemy.org/trac/ticket/801)

+   **[orm]**

    Query.select_from()现在用给定的参数替换所有现有的 FROM 条件；以前的行为通常不太有用，因为它需要 filter()调用来创建连接条件，并且在 filter()中引入的新表已经添加到了 FROM 子句中。新行为不仅允许从主表进行连接，还允许选择语句。过滤条件、排序条件、急加载子句将与给定语句进行“别名”对比。

+   **[orm]**

    本月对属性工具化的重构改变了自 0.3 中途以来我们一直拥有的“加载时复制”行为，大多数情况下改为了“修改时复制”。这样做可以大幅减少加载操作的延迟，并且总体工作量更少，因为只有实际被修改的属性才会复制其“已提交状态”。只有“可变标量”属性（即 pickle 对象或其他可变项）才保留了旧行为，这也是改变加载时复制的原因。

+   **[orm] [attrname]**

    对属性的轻微行为变化是，删除属性不会再次触发该属性的惰性加载器；“del”使属性的有效值变为“None”。要重新触发属性的“加载器”，请使用 session.expire(instance,)。

+   **[orm]**

    当将多对一属性与`None`进行比较时，query.filter(SomeClass.somechild == None)会正确生成“id IS NULL”，包括 NULL 位于右侧的情况。

+   **[orm]**

    query.order_by()会考虑到别名连接，即 query.join('orders', aliased=True).order_by(Order.id)

+   **[orm]**

    eagerload()、lazyload()、eagerload_all()接受可选的第二个类或映射器参数，它将选择要应用选项的映射器。这可以从使用 add_entity()添加的其他映射器中进行选择。

+   **[orm]**

    eagerloading 将与通过 add_entity()添加的映射器一起工作。

+   **[orm]**

    在“dynamic”关系中添加了“级联删除”行为，就像常规关系一样。如果未设置 passive_deletes 标志（也刚刚添加），则删除父项将触发对子项的完全加载，以便可以相应地删除或更新它们。

+   **[orm]**

    还使用 dynamic，实现了正确的 count()行为以及其他辅助方法。

+   **[orm]**

    修复了多态关系上的级联问题，使得从对象到多态集合的级联继续沿着集合中每个元素特定的属性集进行级联。

+   **[orm]**

    query.get()和 query.load()不考虑现有的过滤器或其他条件；这些方法*总是*在数据库中查找给定的 id 或从标识映射中返回当前实例，而不考虑已配置的任何现有过滤器、连接、group_by 或其他条件。

    参考：[#893](https://www.sqlalchemy.org/trac/ticket/893)

+   **[orm]**

    在继承映射器中添加了对 version_id_col 的支持。version_id_col 通常在继承关系中的基本映射器上设置，在那里它对所有继承映射器生效。

    参考：[#883](https://www.sqlalchemy.org/trac/ticket/883)

+   **[orm]**

    放宽了 column_property()表达式具有标签的规则；现在接受任何 ColumnElement，因为编译器现在会自动为没有标签的 ColumnElement 添加标签。可选择的，如 select()语句，仍然需要通过 as_scalar()或 label()转换为 ColumnElement。

+   **[orm]**

    修复了无法删除 instance.attr 的 backref 错误，如果 attr 为 None 的话

+   **[orm]**

    已删除或私有化了几个 ORM 属性：mapper.get_attr_by_column()，mapper.set_attr_by_column()，mapper.pks_by_table，mapper.cascade_callable()，MapperProperty.cascade_callable()，mapper.canload()，mapper.save_obj()，mapper.delete_obj()，mapper._mapper_registry，attributes.AttributeManager

+   **[orm]**

    现在，将不兼容的集合类型分配给关系属性将引发 TypeError，而不是 SQLAlchemy 的 ArgumentError。

+   **[orm]**

    如果传入字典中的键与集合的 keyfunc 为该值使用的键不匹配，则对 MappedCollection 进行批量赋值将引发错误。

    参考：[#886](https://www.sqlalchemy.org/trac/ticket/886)

+   **[orm] [newval1] [newval2]**

    自定义集合现在可以指定一个@converter 方法，将用于“批量”赋值的对象转换为一系列值，如下所示：

    ```py
    obj.col =
    # or
    obj.dictcol = {'foo': newval1, 'bar': newval2}
    ```

    MappedCollection 使用此钩子来确保从集合的角度来看传入的键/值对是合理的。

+   **[orm]**

    在双向关系的两侧同时使用 lazy=”dynamic”时修复了无限循环问题

    参考：[#872](https://www.sqlalchemy.org/trac/ticket/872)

+   **[orm]**

    在 Query + eagerloads 中应用 LIMIT/OFFSET 别名的更多修复，这种情况下映射到 select 语句时

    参考：[#904](https://www.sqlalchemy.org/trac/ticket/904)

+   **[orm]**

    修复了自引用的急切加载，以便如果同一映射实例出现在同一结果集中的两个或更多不同列集中，其急切加载的集合将被填充，无论所有行是否包含该集合的“急切”列集。当打开 join_depth 时，这也会显示为 KeyError。

+   **[orm]**

    修复了当在继承映射器中使用 LIMIT 与仅在父映射器中存在急切加载器时，Query 不会将子查询应用于 SQL 的错误。

+   **[orm]**

    澄清了当您尝试使用相同标识键更新()一个已经存在于会话中的实例时出现的错误消息。

+   **[orm]**

    对 merge(instance, dont_load=True)进行了一些澄清和修复。修复了懒加载器在返回的实例上被禁用的错误。此外，我们目前不支持合并具有未提交更改的实例，在使用 dont_load=True 的情况下….现在会引发错误。这是因为合并给定实例的“已提交状态”以正确对应新复制的实例以及其他修改状态的复杂性。由于 dont_load=True 的用例是缓存，给定实例不应该有任何未提交的更改。我们现在也不使用任何事件将实例复制到新会话中，因此新会话上的“脏”列表保持不受影响。

+   **[orm]**

    修复了在使用 session.begin_nested()与多于一级的封闭 session.begin()语句结合使用时可能出现的错误。

+   **[orm]**

    修复了具有自定义 entity_name 的实例的 session.refresh()。

    参考：[#914](https://www.sqlalchemy.org/trac/ticket/914)

### sql

+   **[sql]**

    通用函数！我们引入了一个已知 SQL 函数的数据库，例如 current_timestamp，coalesce，并创建了表示它们的显式函数对象。这些对象具有受限制的参数列表，具有类型意识，并且可以以特定于方言的方式编译。因此，说 func.char_length(“foo”, “bar”)会引发错误（参数太多），func.coalesce(datetime.date(2007, 10, 5), datetime.date(2005, 10, 15))知道其返回类型是一个日期。到目前为止，我们只表示了一些函数，但将继续向系统添加。

    参考：[#615](https://www.sqlalchemy.org/trac/ticket/615)

+   **[sql]**

    自动重新连接支持改进；连接现在可以在其基础连接无效后自动重新连接，而不需要从引擎重新连接()。这允许绑定到单个连接的 ORM 会话不需要重新连接。连接上的打开事务在基础连接无效后必须回滚，否则会引发错误。还修复了在 cursor()，rollback()或 commit()中未调用 disconnect detect 的错误。

+   **[sql]**

    为 String 和 create_engine() 添加了新标志，assert_unicode=(True|False|’warn’|None)。 在 create_engine() 和 String 上默认为 False 或 None，在 Unicode 类型上为 ‘warn’。 当为 True 时，当传递非 Unicode 字节字符串作为绑定参数时，所有 Unicode 转换操作都会引发异常。 ‘warn’ 会产生警告。 强烈建议所有支持 Unicode 的应用程序正确使用 Python Unicode 对象（即 u’hello’ 而不是 ‘hello’），以便数据往返准确。

+   **[sql]**

    生成“unique”绑定参数的机制已简化为使用与其他所有内容相同的“唯一标识符”机制。 这不会影响用户代码，除非可能已经针对生成的名称进行了硬编码。 生成的绑定参数现在具有“<paramname>_<num>”的形式，而以前只有同名的第二个绑定会具有这种形式。

+   **[sql]**

    select().as_scalar() 如果 select 在其列子句中没有确切一个表达式，则会引发异常。

+   **[sql]**

    bindparam() 对象本身可以用作 execute() 的键，即 statement.execute({bind1:’foo’, bind2:’bar’）

+   **[sql]**

    添加了新的方法到 TypeDecorator，process_bind_param() 和 process_result_value()，它们自动利用底层类型的处理。非常适合与 Unicode 或 Pickletype 一起使用。TypeDecorator 现在应该是增强任何现有类型行为的主要方式，包括其他 TypeDecorator 子类，如 PickleType。

+   **[sql]**

    selectables（以及其他对象）在其导出列集合中基于名称冲突的两列时将发出警告。

+   **[sql]**

    具有模式的表仍然可以在 sqlite、firebird 中使用，模式名称只是被删除

    参考：[#890](https://www.sqlalchemy.org/trac/ticket/890)

+   **[sql]**

    将各种“literal”生成函数更改为使用匿名绑定参数。 这里没有太多变化，除了它们的标签现在看起来像“:param_1”，“:param_2”而不是“:literal”

+   **[sql]**

    现在支持形式为“tablename.columname”的列标签，即带有点的形式。

+   **[sql]**

    select() 的 from_obj 关键字参数可以是标量或列表。

### 杂项

+   **[dialects]**

    sqlite SLDate 类型不会错误地呈现日期时间或时间对象的“微秒”部分。

+   **[dialects]**

    oracle

    +   为 Oracle 添加了断开连接检测支持

    +   对二进制/原始类型进行了一些清理，以便在需要时检测 cx_oracle.LOB

    参考：[#902](https://www.sqlalchemy.org/trac/ticket/902)

+   **[dialects]**

    MSSQL

    +   PyODBC 不再具有全局的 “set nocount on”。

    +   修复 autoload 上的非标识整数 PKs

    +   更好地支持 convert_unicode

    +   对于 pyodbc/adodbapi，日期转换更加宽松

    +   模式限定的表 / autoload

    参考：[#824](https://www.sqlalchemy.org/trac/ticket/824), [#839](https://www.sqlalchemy.org/trac/ticket/839), [#842](https://www.sqlalchemy.org/trac/ticket/842), [#901](https://www.sqlalchemy.org/trac/ticket/901)

+   **[backend] [firebird]**

    正确反映域（部分修复）和 PassiveDefaults

    参考：[#410](https://www.sqlalchemy.org/trac/ticket/410)

+   **[3562] [后端] [firebird]**

    恢复使用默认的池类（在 0.4.0 中设置为 SingletonThreadPool 用于测试目的）

+   **[后端] [firebird]**

    将 func.length()映射到‘char_length’（在旧版本的 Firebird 上可以轻松使用 UDF‘strlen’进行覆盖）

## 0.4.1

发布日期：2007 年 11 月 18 日星期日

### orm

+   **[orm]**

    eager loading 与应用 LIMIT/OFFSET 不再将主表连接到其自身的有限子查询；现在，eager loads 直接连接到提供主表列给结果集的子查询。这消除了所有带有 LIMIT/OFFSET 的 eager loads 的 JOIN。

    参考：[#843](https://www.sqlalchemy.org/trac/ticket/843)

+   **[orm]**

    session.refresh()和 session.expire()现在支持额外的参数“attribute_names”，一个包含要刷新或过期的单个属性键名的列表，允许对已加载实例的属性进行部分重新加载。

    参考：[#802](https://www.sqlalchemy.org/trac/ticket/802)

+   **[orm]**

    向 instrumented 属性添加了 op()运算符；例如，User.name.op(‘ilike’)(‘%somename%’)

    参考：[#767](https://www.sqlalchemy.org/trac/ticket/767)

+   **[orm]**

    映射类现在可以定义具有任意语义的 __eq__、__hash__ 和 __nonzero__ 方法。orm 现在仅基于标识处理所有映射实例。（例如，‘is’ vs ‘==’）

    参考：[#676](https://www.sqlalchemy.org/trac/ticket/676)

+   **[orm]**

    Mapper 上的“properties”访问器已被移除；现在会抛出一个信息性异常，解释 mapper.get_property()和 mapper.iterate_properties 的用法

+   **[orm]**

    在 Query 中添加了 having()方法，与 filter()类似，将 HAVING 应用于生成的语句。

+   **[orm]**

    query 现在完全基于路径的选项，即诸如 eagerload_all(‘x.y.z.y.x’)的选项将仅应用于这些路径，即不包括‘x.y.x’；eagerload(‘children.children’)仅适用于正好两级深度等。

    参考：[#777](https://www.sqlalchemy.org/trac/ticket/777)

+   **[orm]**

    当设置为 mutable=False 时，PickleType 将使用==进行比较，而不是 is 运算符。要使用 is 或任何其他比较器，请使用 PickleType(comparator=my_custom_comparator)发送自定义比较函数。

+   **[orm]**

    如果您同时使用 distinct()和包含 UnaryExpressions（或其他）的 order_by()，查询不会抛出错误

    参考：[#848](https://www.sqlalchemy.org/trac/ticket/848)

+   **[orm]**

    使用 distinct()时，来自连接表的 order_by()表达式将正确添加到列子句中

    参考：[#786](https://www.sqlalchemy.org/trac/ticket/786)

+   **[orm]**

    修复了 Query.add_column()不接受类绑定属性作为参数的错误；Query 还会在 add_column()（在 instances()时间）发送无效参数时引发错误

    参考：[#858](https://www.sqlalchemy.org/trac/ticket/858)

+   **[orm]**

    在 InstanceState.__cleanup() 中增加了更多的垃圾收集解引用检查，以减少应用程序关闭时的 "gc ignored" 错误。

+   **[orm]**

    会话 API 已经稳定：

+   **[orm]**

    对已经持久化的对象执行 session.save() 是一个错误

    参考：[#840](https://www.sqlalchemy.org/trac/ticket/840)

+   **[orm]**

    对于 *非* 持久对象执行 session.delete() 是一个错误。

+   **[orm]**

    当更新或删除已在会话中具有不同标识的实例时，session.update() 和 session.delete() 将抛出错误。

+   **[orm]**

    在确定 "对象 X 已在另一个会话中" 时，会话检查得更加仔细；例如，如果你 pickle 一系列对象然后 unpickle（即在 Pylons HTTP 会话或类似情况下），它们可以进入新的会话而不会产生任何冲突。

+   **[orm]**

    merge() 包含了一个关键字参数 "dont_load=True"。设置此标志将导致合并操作不会从数据库中加载任何数据以响应传入的分离对象，并且将接受传入的分离对象，就好像它已经存在于该会话中。用于将外部缓存系统中的分离对象合并到会话中。

+   **[orm]**

    当属性为延迟列属性时，不再触发加载操作，当属性被赋值时，新分配的值将无条件地出现在 flushes 的 UPDATE 语句中。

+   **[orm]**

    修复了重新分配集合子集（obj.relation = obj.relation[1:]）时的截断错误。

    参考：[#834](https://www.sqlalchemy.org/trac/ticket/834)

+   **[orm]**

    简化了反向引用配置代码，现在会为现有属性抛出错误的反向引用。

    参考：[#832](https://www.sqlalchemy.org/trac/ticket/832)

+   **[orm]**

    改进了 add_property() 等的行为，修复了涉及 synonym/deferred 的问题。

    参考：[#831](https://www.sqlalchemy.org/trac/ticket/831)

+   **[orm]**

    修复了 clear_mappers() 行为，以更好地清理自身。

+   **[orm]**

    修复了 "行切换" 行为，即当一个 INSERT/DELETE 结合到一个单独的 UPDATE 时；父对象上的多对多关系得到了正确更新。

    参考：[#841](https://www.sqlalchemy.org/trac/ticket/841)

+   **[orm]**

    为关联代理修复了 __hash__ ——这些集合是不可哈希的，就像它们的可变 Python 对应物一样。

+   **[orm]**

    为了减少作用域会话的 "save_or_update"、"__contains__" 和 "__iter__" 方法的代理。

+   **[orm]**

    修复了非常难以重现的问题，即查询的 FROM 子句可能会被某些生成调用污染。

    参考：[#852](https://www.sqlalchemy.org/trac/ticket/852)

### sql

+   **[sql]**

    "bindparam()" 上的 "shortname" 关键字参数已被弃用。

+   **[sql]**

    添加了包含运算符（生成一个 "LIKE %<other>%" 子句）。

+   **[sql]**

    匿名列表达式会自动标记。例如，select([x* 5])会产生“SELECT x * 5 AS anon_1”。这允许标签名称出现在 cursor.description 中，然后可以与结果列处理规则相匹配。（我们不能可靠地使用位置跟踪进行结果列匹配，因为 text()表达式可能代表多个列）。

+   **[sql]**

    运算符重载现在由 TypeEngine 对象控制 - 到目前为止内置的运算符重载是 String 类型重载‘+’成为字符串连接运算符。用户定义的类型也可以通过覆盖 adapt_operator(self, op)方法定义自己的运算符重载。

+   **[sql]**

    在二元表达式的右侧使用未命名的绑定参数将被分配为操作左侧的类型，以更好地启用适当的绑定参数处理。

    参考：[#819](https://www.sqlalchemy.org/trac/ticket/819)

+   **[sql]**

    从大多数语句编译中删除了正则表达式步骤。同时也修复了

    参考：[#833](https://www.sqlalchemy.org/trac/ticket/833)

+   **[sql]**

    修复了空（零列）sqlite 插入，允许在自动增量单列表上插入。

+   **[sql]**

    修复了 text()子句的表达式翻译；这修复了各种 ORM 场景中使用文字文本作为 SQL 表达式的情况。

+   **[sql]**

    删除了 ClauseParameters 对象；compiled.params 现在返回一个常规字典，以及 result.last_inserted_params() / last_updated_params()。

+   **[sql]**

    修复了关于具有基于 SQL 表达式的默认生成器的主键列的 INSERT 语句；SQL 表达式会像正常情况下内联执行，但不会为那些通过 cursor.lastrowid 提供“postfetch”条件的列触发。

+   **[sql]**

    func.对象可以被 pickle/unpickle

    参考：[#844](https://www.sqlalchemy.org/trac/ticket/844)

+   **[sql]**

    重写并简化了用于在可选择表达式之间“定位”列的系统。在 SQL 端，这由“corresponding_column()”方法表示。该方法在 ORM 中被广泛使用，用于将表达式的元素“适应”到类似的，别名的表达式，以及将最初绑定到表或可选择表达式的结果集列定位到别名的“对应”表达式。新的重写功能具有完全一致和准确的行为。

+   **[sql]**

    添加了一个字段（“info”）用于在模式项上存储任意数据。

    参考：[#573](https://www.sqlalchemy.org/trac/ticket/573)

+   **[sql]**

    连接上的“properties”集合已重命名为“info”，以匹配模式的可写集合。直到 0.5 版本，仍可通过“properties”名称访问。

+   **[sql]**

    修复了使用 strategy=’threadlocal’时 Transaction 上的 close()方法。

+   **[sql]**

    修复了编译绑定参数不会错误地填充 None。

    参考：[#853](https://www.sqlalchemy.org/trac/ticket/853)

+   **[sql]**

    <Engine|Connection>._execute_clauseelement 变成了一个公共方法 Connectable.execute_clauseelement

### 杂项

+   **[dialects]**

    添加了对 MaxDB 的实验性支持（仅适用于版本 >= 7.6.03.007）。

+   **[dialects]**

    现在 oracle 会将“DATE”反映为 OracleDateTime 列，而不是 OracleDate

+   **[dialects]**

    在 Oracle table_names() 函数中增加了对模式名称的支持，修复了 metadata.reflect(schema=’someschema’)

    参考：[#847](https://www.sqlalchemy.org/trac/ticket/847)

+   **[dialects]**

    MSSQL 匿名标签用于选择函数，使其具有确定性

+   **[dialects]**

    sqlite 将“DECIMAL”反映为数字列。

+   **[dialects]**

    使访问 dao 检测更加可靠

    参考：[#828](https://www.sqlalchemy.org/trac/ticket/828)

+   **[dialects]**

    将 Dialect 属性 ‘preexecute_sequences’ 重命名为 ‘preexecute_pk_sequences’。对于使用旧名称的 out-of-tree 方言，现在有一个属性代理。

+   **[dialects]**

    为未知类型反射添加了测试覆盖。修复了 sqlite/mysql 对于未知类型反射的处理。

+   **[dialects]**

    为 mysql 方言添加了 REAL（用于利用 REAL_AS_FLOAT sql 模式的人）。

+   **[dialects]**

    mysql Float、MSFloat 和 MSDouble 现在构造时不带参数会产生无参数 DDL，例如’FLOAT’。

+   **[misc]**

    删除了未使用的 util.hash()。

## 0.4.0

发布日期：2007 年 10 月 17 日 星期三

+   **[no_tags]**

    （请查看 0.4.0beta1，了解针对 0.3 的重大更改的开始，以及 [`www.sqlalchemy.org/trac/wiki/WhatsNewIn04`](https://www.sqlalchemy.org/trac/wiki/WhatsNewIn04)）

+   **[no_tags]**

    增加了对 Sybase 的初始支持（目前仅限于 mxODBC）

    参考：[#785](https://www.sqlalchemy.org/trac/ticket/785)

+   **[no_tags]**

    为 PostgreSQL 添加了部分索引支持。在索引上使用 postgres_where 关键字。

+   **[no_tags]**

    基于字符串的查询参数解析/配置文件解析现在可以理解更广泛范围的布尔值字符串

    参考：[#817](https://www.sqlalchemy.org/trac/ticket/817)

+   **[no_tags]**

    如果另一侧集合不包含项目，则 backref 删除对象操作不会失败，支持 noload 集合

    参考：[#813](https://www.sqlalchemy.org/trac/ticket/813)

+   **[no_tags]**

    从“dynamic”集合中删除了 __len__，因为这将需要发出 SQL “count()” 操作，从而迫使所有列表评估发出冗余的 SQL

    参考：[#818](https://www.sqlalchemy.org/trac/ticket/818)

+   **[no_tags]**

    对 locate_dirty() 进行了内联优化，可以大大加快对 flush() 的重复调用，如在 autoflush=True 的情况下发生的情况

    参考：[#816](https://www.sqlalchemy.org/trac/ticket/816)

+   **[no_tags]**

    IdentifierPreprarer 的 _requires_quotes 测试现在基于正则表达式。任何提供自定义合法字符集或非法初始字符集的 out-of-tree 方言都需要转移到正则表达式或覆盖 _requires_quotes。

+   **[no_tags]**

    Firebird 由于 ticket #370（正确方式）而将 supports_sane_rowcount 和 supports_sane_multi_rowcount 设置为 False。

+   **[no_tags]**

    Firebird 反射的改进和修复：

    +   FBDialect 现在模仿 OracleDialect，关于 TABLE 和 COLUMN 名称的大小写敏感性（请参见本文件中的“case_sensitive remotion”主题）。

    +   FBDialect.table_names()不会带来系统表（票号：796）。

    +   FB 现在正确反映了 Column 的 nullable 属性。

+   **[无标签]**

    修复了 SQL 编译器对顶级列标签的意识，用于结果集处理；包含相同列名的嵌套选择不会影响结果或与结果列元数据冲突。

+   **[无标签]**

    query.get()和相关函数（如一对多的延迟加载）使用编译时别名绑定参数名称，以防止与已存在于映射可选择项中的绑定参数名称发生名称冲突。

+   **[无标签]**

    修复了三级和多级选择以及延迟继承加载（即没有选择表的 abc 继承）。

    参考：[#795](https://www.sqlalchemy.org/trac/ticket/795)

+   **[无标签]**

    在 shard.py 中传递给 id_chooser 的 Ident 始终是一个列表。

+   **[无标签]**

    无参数 ResultProxy._row_processor()现在是类属性 _process_row。

+   **[无标签]**

    增加了对从 PostgreSQL 8.2+插入和更新返回值的支持。

    参考：[#797](https://www.sqlalchemy.org/trac/ticket/797)

+   **[无标签]**

    PG 反射，在看到默认模式名称明确用作表中的“模式”参数时，将假定这是用户期望的约定��并将在外键相关的反射表中明确设置“模式”参数，从而使它们仅与也使用显式“模式”参数的 Table 构造函数匹配（即使其为默认模式）。换句话说，SA 假定用户在此使用中是一致的。

+   **[无标签]**

    修复了 BOOL/BOOLEAN 的 sqlite 反射。

    参考：[#808](https://www.sqlalchemy.org/trac/ticket/808)

+   **[无标签]**

    增加了对在 mysql 上使用 LIMIT 进行 UPDATE 的支持。

+   **[无标签]**

    m2o 上的空外键不会触发延迟加载。

    参考：[#803](https://www.sqlalchemy.org/trac/ticket/803)

+   **[无标签]**

    Oracle 不会隐式地将非类型化结果集转换为 unicode（即，当没有使用 TypeEngine/String/Unicode 类型时；以前它会检测 DBAPI 类型并进行转换）。应该修复

    参考：[#800](https://www.sqlalchemy.org/trac/ticket/800)

+   **[无标签]**

    修复了长表/列名称的匿名标签生成问题。

    参考：[#806](https://www.sqlalchemy.org/trac/ticket/806)

+   **[无标签]**

    Firebird 方言现在使用 SingletonThreadPool 作为 poolclass。

+   **[无标签]**

    Firebird 现在使用 dialect.preparer 来格式化序列名称。

+   **[无标签]**

    修复了与 postgres 和多个两阶段事务的中断。两阶段提交和回滚不会像通常的 dbapi 提交/回滚那样自动结束一个新事务。

    参考：[#810](https://www.sqlalchemy.org/trac/ticket/810)

+   **[无标签]**

    向 _ScopedExt 映射器扩展添加了一个选项，以便在对象初始化时不自动将新对象保存到会话中。

+   **[无标签]**

    修复了 Oracle 非 ANSI 连接语法

+   **[无标签]**

    PickleType 和 Interval 类型（在不支持它的数据库上）现在略微更快。

+   **[无标签]**

    在 Firebird 中添加了 Float 和 Time 类型（FBFloat 和 FBTime）。修复了 TEXT 和 Binary 类型的 BLOB SUB_TYPE。

+   **[无标签]**

    更改了 in_ 运算符的 API。in_() 现在接受一个作为值序列或可选择的单个参数。传递值作为可变参数的旧 API 仍然有效，但已被弃用。

## 0.4.0beta6

发布日期：Thu Sep 27 2007

+   **[无标签]**

    会话标识映射现在默认为*弱引用*，使用 weak_identity_map=False 来使用常规字典。我们使用的弱字典被定制为检测“脏”实例，并保持对这些实例的临时强引用，直到更改被刷新。

+   **[无标签]**

    Mapper 编译已重新组织，大部分编译发生在 mapper 构造时。这使我们可以减少对 mapper.compile() 的调用，并且允许基于类的属性强制进行编译（即 User.addresses == 7 将编译所有映射器；这是）。唯一的注意事项是，继承映射器现在在构造时寻找其继承的映射器；因此，继承关系中的映射器需要按照继承顺序进行构造（这应该是正常情况）。

    参考：[#758](https://www.sqlalchemy.org/trac/ticket/758)

+   **[无标签]**

    在 Postgres 中添加了“FETCH”关键字，用于指示结果行保持语句（即除了“SELECT”之外）。

+   **[无标签]**

    添加了 SQLite 保留关键字的完整列表，以便正确转义它们。

+   **[无标签]**

    加强了 Query 生成“eager load” 别名与 Query.instances() 之间的关系，Query.instances() 实际上获取了急切加载的行。如果别名不是由 EagerLoader 专门为该语句生成的，则在获取行时 EagerLoader 将不起作用。这可以防止意外地抓取列作为急切加载的一部分，当它们不是为此目的而设计时，这可能会发生在文本 SQL 以及一些继承情况下。这一点尤为重要，因为“匿名别名”现在使用简单的整数计数来生成标签。

+   **[无标签]**

    从 clauseelement.compile() 中移除了“parameters”参数，替换为“column_keys”。传递给 execute() 的参数仅与插入/更新语句的编译过程中存在的列名交互，而不涉及这些列的值。产生更一致的 execute/executemany 行为，内部稍微简化了一些事情。

+   **[无标签]**

    在 PickleType 中添加了 'comparator' 关键字参数。默认情况下，“mutable” PickleType 使用它们的 dumps() 表示进行对象的“深度比较”。但这对于字典不起作用。提供了足够 __eq__() 实现的 Pickled 对象可以设置为 “PickleType(comparator=operator.eq)”

    参考资料：[#560](https://www.sqlalchemy.org/trac/ticket/560)

+   **[无标签]**

    添加了 session.is_modified(obj) 方法；执行与刷新操作中发生的“历史”比较操作相同；设置 include_collections=False 将得到与刷新确定是否为实例的行发出 UPDATE 相同的结果。

+   **[无标签]**

    向 Sequence 添加了“schema”参数；当序列位于备用模式中时，与 Postgres /Oracle 一起使用。实现部分内容，应该修复。

    参考资料：[#584](https://www.sqlalchemy.org/trac/ticket/584), [#761](https://www.sqlalchemy.org/trac/ticket/761)

+   **[无标签]**

    修复了 mysql 枚举类型的空字符串反射问题。

+   **[无标签]**

    将 MySQL 方言更改为使用旧的 LIMIT <offset>, <limit> 语法，而不是 LIMIT <l> OFFSET <o>，适用于使用 3.23 版本的用户。

    参考资料：[#794](https://www.sqlalchemy.org/trac/ticket/794)

+   **[无标签]**

    向 relation() 添加了 ‘passive_deletes=”all”’ 标志；在删除父对象时禁用所有外键属性的置空。

+   **[无标签]**

    执行内联的列默认值和 onupdates 将为子查询和其他需要括号的表达式添加括号

+   **[无标签]**

    现在关于 String/Unicode 类型的行为是，只有在没有长度的情况下才会自动转换为 TEXT/CLOB 类型，仅适用于没有参数的确切类型的 String 或 Unicode。如果您使用没有长度的 VARCHAR 或 NCHAR（String/Unicode 的子类），它们将被方言解释为 VARCHAR/NCHAR；这里不会发生“神奇”的转换。这是更少令人惊讶的行为，特别是这有助于 Oracle 将基于字符串的绑定参数保持为 VARCHAR，而不是 CLOB。

    参考资料：[#793](https://www.sqlalchemy.org/trac/ticket/793)

+   **[无标签]**

    对 ShardedSession 进行了修复，以使其与延迟列一起工作。

    参考资料：[#771](https://www.sqlalchemy.org/trac/ticket/771)

+   **[无标签]**

    用户定义的 shard_chooser() 函数必须接受“clause=None”参数；这是传递给 session.execute(statement) 的 ClauseElement，并且可以用于确定正确的分片 id（因为 execute() 不接受实例）。

+   **[无标签]**

    调整了 NOT 运算符的优先级，以匹配‘==’和其他运算符，因此~(x <operator> y)会产生 NOT (x <op> y)，这与旧版 MySQL 更兼容。这不适用于“~(x==y)”，因为在 0.3 版本中像这样的表达式 ~(x==y) 会被编译成 “x != y”，但仍适用于像 BETWEEN 这样的运算符。

    参考资料：[#764](https://www.sqlalchemy.org/trac/ticket/764)

+   **[无标签]**

    其他票据：,,.

    参考资料：[#728](https://www.sqlalchemy.org/trac/ticket/728), [#757](https://www.sqlalchemy.org/trac/ticket/757), [#768](https://www.sqlalchemy.org/trac/ticket/768), [#779](https://www.sqlalchemy.org/trac/ticket/779)

## 0.4.0beta5

无发布日期

+   **[无标签]**

    连接池修复；beta4 版本的更好性能仍然存在，但修复了“连接溢出”和其他存在的 bug（如）。

    参考：[#754](https://www.sqlalchemy.org/trac/ticket/754)

+   **[无标签]**

    修复了从自定义继承条件中确定适当同步子句的错误。

    参考：[#769](https://www.sqlalchemy.org/trac/ticket/769)

+   **[无标签]**

    对于 QueuePool 大小/溢出，扩展了'engine_from_config'的转换。

    参考：[#763](https://www.sqlalchemy.org/trac/ticket/763)

+   **[无标签]**

    mysql 视图现在可以反射了。

    参考：[#748](https://www.sqlalchemy.org/trac/ticket/748)

+   **[无标签]**

    AssociationProxy 现在可以使用自定义的 getter 和 setter。

+   **[无标签]**

    修复了 orm 查询中 BETWEEN 的失效。

+   **[无标签]**

    修复了 OrderedProperties 的 pickling 问题。

    参考：[#762](https://www.sqlalchemy.org/trac/ticket/762)

+   **[无标签]**

    SQL 表达式的默认值和序列现在在 INSERT 或 UPDATE 期间对所有非主键列执行“内联”操作，并在 executemany()样式的调用期间对所有列执行。任何 insert/update 语句上的 inline=True 标志也会强制执行相同的行为，即单个 execute()。result.postfetch_cols()是之前的单个 insert 或 update 语句中包含 SQL 端默认表达式的列的集合。

+   **[无标签]**

    修复了 PG executemany()的行为。

    参考：[#759](https://www.sqlalchemy.org/trac/ticket/759)

+   **[无标签]**

    postgres 针对没有默认值的主键列的表反射自动增量=False。

+   **[无标签]**

    postgres 现在不再用单独的 execute()调用包装 executemany()，而是更偏向性能。在 PG 上，使用 executemany()的“rowcount”/“concurrency”检查与删除项目（使用 executemany）被禁用，因为 psycopg2 不会报告 executemany()的正确 rowcount。

+   **[修复] [票务]**

    参考：[#742](https://www.sqlalchemy.org/trac/ticket/742)

+   **[修复] [票务]**

    参考：[#748](https://www.sqlalchemy.org/trac/ticket/748)

+   **[修复] [票务]**

    参考：[#760](https://www.sqlalchemy.org/trac/ticket/760)

+   **[修复] [票务]**

    参考：[#762](https://www.sqlalchemy.org/trac/ticket/762)

+   **[修复] [票务]**

    参考：[#763](https://www.sqlalchemy.org/trac/ticket/763)

## 0.4.0beta4

发布日期：2007 年 8 月 22 日（星期三）

+   **[无标签]**

    当您从 SQLAlchemy 导入*时，整理了您的命名空间中的内容：

+   **[无标签]**

    'table'和'column'不再被导入。它们仍可通过直接引用（如'sql.table'和'sql.column'）或从 sql 包进行全局导入使用。在刚开始使用 SQLAlchemy 时，意外使用 sql.expressions.table 而不是 schema.Table 变得太容易了，同样的情况也出现在 column 中。

+   **[无标签]**

    类似 ClauseElement、FromClause、NullTypeEngine 等的内部类也不再被导入到您的命名空间中。

+   **[无标签]**

    'Smallinteger'兼容名称（小 i！）不再被导入，但暂时仍保留在 schema.py 中。SmallInteger（大 I！）仍然被导入。

+   **[无标签]**

    连接池在内部使用“threadlocal”策略，以返回已绑定到线程的相同连接，用于“上下文”连接；这些连接在执行“无连接”操作时使用，比如 insert().execute()。这类似于“threadlocal”引擎策略的“部分”版本，但没有其中的线程本地事务部分。我们希望它减少连接池开销以及数据库使用。但是，如果证明对稳定性产生负面影响，我们将立即撤销。

+   **[无标签]**

    修复了绑定参数处理，使得“False”值（如空字符串）仍然会被处理/编码。

+   **[无标签]**

    修复了 select()的“生成”行为，使得调用 column()、select_from()、correlate()和 with_prefix()不会修改原始 select 对象

    参考：[#752](https://www.sqlalchemy.org/trac/ticket/752)

+   **[无标签]**

    添加了一个“legacy”适配器到 types，使得用户定义的 TypeEngine 和 TypeDecorator 类，定义了 convert_bind_param()和/或 convert_result_value()的仍然可以正常工作。还支持调用这些方法的 super()版本。

+   **[无标签]**

    添加了 session.prune()，修剪会话中不再在其他地方引用的实例。 （用于强引用标识映射的实用程序）。

+   **[无标签]**

    向 Transaction 添加了 close()方法。如果是最外层事务，则使用回滚结束事务，否则仅结束而不影响外部事务。

+   **[无标签]**

    事务性和非事务性 Session 与绑定连接更好地集成；close()将确保连接的事务状态与绑定到 Session 之前的状态相同。

+   **[无标签]**

    修改了 SQL 操作函数为模块级操作符，允许 SQL 表达式可被 pickle 化。

    参考：[#735](https://www.sqlalchemy.org/trac/ticket/735)

+   **[无标签]**

    对 mapper 类的 __init__ 进行小调整，以允许 Py2.6 对象的 __init__()行为。

+   **[无标签]**

    修复了 select()的‘prefix’参数

+   **[无标签]**

    Connection.begin()不再接受 nested=True，这个逻辑现在都在 begin_nested()中。

+   **[无标签]**

    修复了涉及级联的新“动态”关系加载器的问题

+   **[修复] [票务]**

    参考：[#735](https://www.sqlalchemy.org/trac/ticket/735)

+   **[修复] [票务]**

    参考：[#752](https://www.sqlalchemy.org/trac/ticket/752)

## 0.4.0beta3

发布日期：Thu Aug 16 2007

+   **[无标签]**

    SQL 类型优化：

+   **[无标签]**

    新的性能测试显示，与 0.3 版本相比，组合的大规模插入/大规模选择测试的函数调用减少了 68%。

+   **[无标签]**

    结果集迭代的一般性能提升约为 10-20%。

+   **[无标签]**

    在 types.AbstractType 中，convert_bind_param()和 convert_result_value()已迁移到返回可调用的 bind_processor()和 result_processor()方法。如果没有返回可调用对象，则不会调用任何预处理/后处理函数。

+   **[无标签]**

    在 base/sql/defaults 中添加了钩子以优化绑定参数/结果处理器的调用，以便最小化方法调用开销。

+   **[无标签]**

    支持添加了对 executemany()场景的支持，以便不需要的“最后一行 id”逻辑不会触发，参数不会被过度遍历。

+   **[无标签]**

    添加了‘inherit_foreign_keys’参数到 mapper()。

+   **[无标签]**

    添加了对 sqlite 中字符串日期传递的支持。

+   **[修复] [票务]**

    参考：[#738](https://www.sqlalchemy.org/trac/ticket/738)

+   **[修复] [票务]**

    参考：[#739](https://www.sqlalchemy.org/trac/ticket/739)

+   **[修复] [票务]**

    参考：[#743](https://www.sqlalchemy.org/trac/ticket/743)

+   **[修复] [票务]**

    参考：[#744](https://www.sqlalchemy.org/trac/ticket/744)

## 0.4.0beta2

发布日期：2007 年 8 月 14 日星期二

### oracle

+   **[oracle] [改进]**

    在 mysql 中 LOAD DATA INFILE 后自动提交。

+   **[oracle] [改进]**

    添加了一个基本的 SessionExtension 类，允许在 flush()、commit()和 rollback()边界处发生用户定义的功能。

+   **[oracle] [改进]**

    添加了 engine_from_config()函数，以帮助从.ini 样式配置创建 engine()。

+   **[oracle] [改进]**

    base_mapper()变成了一个普通属性。

+   **[oracle] [改进]**

    session.execute()和 scalar()可以通过给定的 ClauseElement 搜索要绑定的表。

+   **[oracle] [改进]**

    会话自动从具有绑定的映射器中推断表，还使用 base_mapper，以便继承层次结构自动绑定。

+   **[oracle] [改进]**

    将 ClauseVisitor 遍历移回到内联的非递归方式。

### 杂项

+   **[修复] [票务]**

    参考：[#730](https://www.sqlalchemy.org/trac/ticket/730)

+   **[修复] [票务]**

    参考：[#732](https://www.sqlalchemy.org/trac/ticket/732)

+   **[修复] [票务]**

    参考：[#733](https://www.sqlalchemy.org/trac/ticket/733)

+   **[修复] [票务]**

    参考：[#734](https://www.sqlalchemy.org/trac/ticket/734)

## 0.4.0beta1

发布日期：2007 年 8 月 12 日星期日

### orm

+   **[orm]**

    速度！除了对 ResultProxy 的最近加速，对于大量加载，函数调用总数显著减少。

+   **[orm]**

    test/perf/masseagerload.py 报告 0.4 版本在所有 SA 版本（0.1、0.2 和 0.3）中具有最少的函数调用次数。

+   **[orm]**

    新的 collection_class api 和实现。现在通过装饰而不是代理来检测集合。现在可以有管理自己成员资格的集合，并且您的类实例将直接暴露在关系属性上。对于大多数用户来说，这些更改是透明的。

    参考：[#213](https://www.sqlalchemy.org/trac/ticket/213)

+   **[orm]**

    InstrumentedList（如之前）已被移除，关系属性不再具有‘clear()’、‘.data’或任何其他除集合类型提供的方法。当然，您可以将它们添加到自定义类中。

+   **[orm]**

    类似 __setitem__ 的赋值现在会为现有值触发删除事件，如果有的话。

+   **[orm]**

    作为集合类使用的类似字典的对象不再需要更改 __iter__ 语义- 默认使用 itervalues()。这是一个不兼容的变更。

+   **[orm]**

    在大多数情况下，不再需要为映射集合子类化 dict。orm.collections 提供了按指定列或自定义函数键入对象的预制实现。

+   **[orm]**

    现在集合赋值需要兼容的类型- 将 None 赋给一个集合以清空它，或将列表赋给一个字典集合将会引发参数错误。

+   **[orm]**

    AttributeExtension 移至接口，并且 .delete 现在是 .remove 事件方法签名也已经���换。

+   **[orm]**

    Query 进行了重大改进：

+   **[orm]**

    所有 selectXXX 方法已被弃用。生成方法现在是执行操作的标准方式，即 filter()、filter_by()、all()、one() 等。弃用的方法在其新替代品的文档字符串中有说明。

+   **[orm]**

    现在可以将类级属性用作查询元素… 不再需要 ‘.c.’！“Class.c.propname” 现在被 “Class.propname” 取代。支持所有子句操作符，以及更高级别的操作符，如 Class.prop==<some instance> 用于标量属性，Class.prop.contains(<some instance>) 和 Class.prop.any(<some expression>) 用于基于集合的属性（所有这些也是可否定的）。当然，基于表的列表达式以及通过 ‘c’ 挂载在映射类上的列仍然完全可用，并且可以与新属性自由混合。

    参考：[#643](https://www.sqlalchemy.org/trac/ticket/643)

+   **[orm]**

    移除了古老的 query.select_by_attributename() 功能。

+   **[orm]**

    急加载使用的别名逻辑已经泛化，因此还为 Query 添加了完全自动的别名支持。不再需要为多次连接到相同表创建显式别名；*即使是自引用关系也是如此*。

    +   join() 和 outerjoin() 接受参数 “aliased=True”。这将导致它们的连接建立在别名表上；随后调用 filter() 和 filter_by() 将会将所有表达式（是的，使用原始映射表的真实表达式）转换为别名的表达式，直到该 join() 结束（即重置 joinpoint() 或调用另一个 join()）。

    +   join() 和 outerjoin() 接受参数 “id=<somestring>”。当与 “aliased=True” 一起使用时，可以通过 add_entity(cls, id=<somestring>) 引用 id，以便在从别名获取的情况下选择连接的实例。

    +   join() 和 outerjoin() 现在可以用于自引用关系！使用 “aliased=True”，可以连接到任意深度的级别，例如 query.join([‘children’, ‘children’], aliased=True)；过滤条件将针对最右侧连接的表

+   **[orm]**

    添加了 query.populate_existing()，标记查询以重新加载查询中触及的所有实例的所有属性和集合，包括急加载的实体。

    参考：[#660](https://www.sqlalchemy.org/trac/ticket/660)

+   **[orm]**

    添加了 eagerload_all()，允许 eagerload_all(‘x.y.z’)指定给定路径中所有属性的急加载。

+   **[orm]**

    会话的重大改进：

+   **[orm]**

    新功能“配置”一个名为“sessionmaker()”的会话。一次向该函数发送各种关键字参数，返回一个新类，该类针对该模式创建一个会话。

+   **[orm]**

    从“public”API 中移除了 SessionTransaction。现在可以在 Session 本身上调用 begin()/commit()/rollback()。

+   **[orm]**

    会话还支持 SAVEPOINT 事务；调用 begin_nested()。

+   **[orm]**

    当垂直或水平分区（即，使用多个引擎）时，会话支持两阶段提交行为。使用 twophase=True。

+   **[orm]**

    会话标志“transactional=True”会产生一个会话，当首次使用时总是将自身置于事务中。在 commit()、rollback()或 close()时，事务结束；但在下一次使用时重新开始。

+   **[orm]**

    会话支持“autoflush=True”。这会在每次查询之前发出一个 flush()。与 transactional 一起使用，您可以只保存()/更新()然后查询，新对象将在那里。在最后使用 commit()（或 flush()如果非事务性）来刷新剩余的更改。

+   **[orm]**

    新的 scoped_session()函数取代了 SessionContext 和 assignmapper。构建在“sessionmaker()”概念之上，以产生一个类，其 Session()构造返回线程本地会话。或者，将所有 Session 方法作为类方法调用，即 Session.save(foo); Session.commit()。就像旧的“objectstore”时代一样。

+   **[orm]**

    添加了新的“binds”参数到 Session，以支持使用 sessionmaker()函数配置多个绑定。

+   **[orm]**

    添加了一个基本的 SessionExtension 类，允许在 flush()、commit()和 rollback()边界发生时进行用户定义的功能。

+   **[orm]**

    基于查询的 relation()可通过 dynamic_loader()使用。这是一个*writable*集合（支持 append()和 remove()），当用于读取时也是一个活动的 Query 对象。适用于处理仅希望部分加载的非常大的集合。

+   **[orm]**

    flush()-嵌入式的内联 INSERT/UPDATE 表达式。将任何 SQL 表达式，如“sometable.c.column + 1”，分配给实例的属性。在 flush()时，映射器检测到表达式并直接嵌入到 INSERT 或 UPDATE 语句中；属性在实例上被延迟，因此在下次访问时加载新值。

+   **[orm]**

    引入了一个基本的分片（水平扩展）系统。该系统使用修改后的 Session，可以根据用户定义的“分片策略”分发读取和写入操作到多个数据库。实例及其依赖项可以根据属性值、轮询方法或任何其他用户定义的系统分布和查询到多个数据库。

    参考：[#618](https://www.sqlalchemy.org/trac/ticket/618)

+   **[orm]**

    急切加载已经增强，允许更多的连接在更多的地方。现在它可以在自引用和循环结构的任意深度处运行。在加载循环结构时，在 relation()上指定“join_depth”，指示您希望表自连接多少次；每个级别都会得到一个不同的表别名。别名名称现在是在编译时使用简单的计数方案生成的，更容易阅读，当然完全确定性。

    参考：[#659](https://www.sqlalchemy.org/trac/ticket/659)

+   **[orm]**

    添加了复合列属性。这允许您创建一个由多个列表示的类型，当使用 ORM 时。新类型的对象在查询表达式、比较、query.get()子句等方面都是完全功能的，并且表现得就像是常规的单列标量…除了它们不是！在映射器的“属性”字典中使用函数 composite(cls, *columns)，并且 cls 的实例将被创建/映射到一个单属性，由对应于*columns 的值组成。

    参考：[#211](https://www.sqlalchemy.org/trac/ticket/211)

+   **[orm]**

    对具有相关子查询的自定义 column_property()属性的支持得到了改进，现在与急切加载更好地配合。

+   **[orm]**

    主键“折叠”行为；映射器将分析其给定可选择的所有列，以获取主键“等效性”，即通过外键关系或显式继承条件等方式等效的列。主要用于联合表继承场景，其中继承表中的不同命名 PK 列应“折叠”为单值（或更少值）主键。修复了诸如此类的问题。

    参考：[#611](https://www.sqlalchemy.org/trac/ticket/611)

+   **[orm]**

    联合表继承现在将所有继承类的主键列生成到联接的根表中。这意味着根表中的每一行对应一个实例。如果出于某种罕见原因不希望这样，单独映射器上的显式 primary_key 设置将覆盖它。

+   **[orm]**

    当“多态”标志与联合表或单表继承一起使用时，所有标识键都针对继承层次结构的根类生成；这允许查询.get()以与非多态 get 相同的缓存语义多态工作。请注意，这目前不适用于具体继承。

+   **[orm]**

    次要继承加载：可以构建多态映射器而不需要 select_table 参数。继承映射器的表在初始加载中没有被表示，将立即发出第二个 SQL 查询，每个实例一次（对于大型列表来说效率不高），以加载剩余的列。

+   **[orm]**

    次要继承加载也可以将其第二个查询移动到列级“延迟”加载中，通过“polymorphic_fetch”参数，可以设置为“select”或“deferred”。

+   **[orm]**

    现在可以仅将可选择的列的子集映射到映射器属性中，使用 include_columns/exclude_columns。

    参考：[#696](https://www.sqlalchemy.org/trac/ticket/696)

+   **[orm]**

    添加了 undefer_group() MapperOption，设置一组由“group”连接的“延迟”列以作为“未延迟”加载。

+   **[orm]**

    重写了“确定性别名”逻辑，使其成为 SQL 层的一部分，生成更简单的别名和标签名称，更符合 Hibernate 的风格

### sql

+   **[sql]**

    速度！子句编译以及 SQL 构造的机制已经被简化和简化到一个显著程度，使语句构造/编译的开销减少了 20-30%，为 0.3。

+   **[sql]**

    所有“type”关键字参数，如 bindparam()、column()、Column()和 func.<something>()，都重命名为“type_”。这些对象仍然将它们的“type”属性命名为“type”。

+   **[sql]**

    从模式项中移除了 case_sensitive=(True|False)设置，因为检查这种状态会增加很多方法调用开销，而且从来没有一个合理的理由将其设置为 False。所有小写的表名和列名都将被视为不区分大小写（是的，我们也适应了 Oracle 的大写风格）。

### extensions

+   **[extensions]**

    proxyengine 暂时移除，等待一个真正有效的替代品。

+   **[extensions]**

    SelectResults 已被 Query 取代。SelectResults / SelectResultsExt 仍然存在，但只是返回一个稍微修改的 Query 对象以保持向后兼容性。SelectResults 的 join_to()方法不再存在，需要使用 join()。

### mysql

+   **[mysql]**

    通过反射加载的表名和列名现在都是 Unicode。

+   **[mysql]**

    现在支持所有标准列类型，包括 SET。

+   **[mysql]**

    表反射现在可以在一次往返中执行。

+   **[mysql]**

    现在支持 ANSI 和 ANSI_QUOTES SQL 模式。

+   **[mysql]**

    现在反映了索引。

### oracle

+   **[oracle]**

    添加了对 OUT 参数的非常基本的支持；使用 sql.outparam(name, type)设置一个 OUT 参数，就像 bindparam()一样；执行后，值可以通过 result.out_parameters 字典获得。

    参考：[#507](https://www.sqlalchemy.org/trac/ticket/507)

### 杂项

+   **[transactions]**

    为事务添加了上下文管理器（with 语句）支持。

+   **[transactions]**

    添加了对两阶段提交的支持，目前与 mysql 和 postgres 一起使用。

+   **[transactions]**

    添加了一个使用保存点的子事务实现。

+   **[事务]**

    添加了保存点的支持。

+   **[元数据]**

    可以从数据库中一次性反射出表，而不需要预先声明它们。MetaData(engine, reflect=True) 将加载数据库中存在的所有表，或者使用 metadata.reflect() 进行更精细的控制。

+   **[元数据]**

    DynamicMetaData 已重命名为 ThreadLocalMetaData

+   **[元数据]**

    ThreadLocalMetaData 构造函数现在不接受参数。

+   **[元数据]**

    BoundMetaData 已被移除，常规 MetaData 是等效的。

+   **[元数据]**

    数字和浮点类型现在有一个“asdecimal”标志；对于 Numeric，默认值为 True，对于 Float，默认值为 False。当为 True 时，值以 decimal.Decimal 对象返回；当为 False 时，值以 float() 返回。True/False 的默认值已经是 PG 和 MySQL 的 DBAPI 模块的行为。

    参考：[#646](https://www.sqlalchemy.org/trac/ticket/646)

+   **[元数据]**

    新的 SQL 运算符实现将所有硬编码的运算符从表达式结构中移除，并将它们移入编译中；允许更大灵活性的运算符编译；例如，在字符串上下文中，“+” 编译为“||”，在 MySQL 上则编译为“concat(a,b)”；而在数值上下文中，它编译为“+”。修复。

    参考：[#475](https://www.sqlalchemy.org/trac/ticket/475)

+   **[元数据]**

    “匿名”别名和标签名称现在以完全确定性的方式在 SQL 编译时生成……不再是随机的十六进制 ID

+   **[元数据]**

    对 SQL 元素（ClauseElement）进行了重大的架构改造。所有元素共享一个通用的“可变性”框架，允许对元素进行一致的原地修改以及生成行为。改进了 ORM 的稳定性，ORM 对 SQL 表达式的变异使用频繁。

+   **[元数据]**

    select() 和 union() 现在具有“生成”行为。order_by() 和 group_by() 等方法返回一个 *新* 实例——原始实例保持不变。非生成方法也保留不变。

+   **[元数据]**

    select/union 的内部大大简化——所有关于“是否为子查询”和“关联性”的决策都推迟到 SQL 生成阶段。select() 元素现在永远不会被其包含的容器或任何方言的编译过程修改。

    参考：[#52](https://www.sqlalchemy.org/trac/ticket/52)，[#569](https://www.sqlalchemy.org/trac/ticket/569)

+   **[元数据]**

    select(scalar=True) 参数已弃用；使用 select(..).as_scalar()。生成的对象遵循完整的“列”接口，并在表达式中表现更好。

+   **[元数据]**

    添加了 select().with_prefix('foo')，允许在 SELECT 的列子句之前放置任何一组关键字。

    参考：[#504](https://www.sqlalchemy.org/trac/ticket/504)

+   **[元数据]**

    添加了对行[<index>]的数组切片支持。

    参考：[#686](https://www.sqlalchemy.org/trac/ticket/686)

+   **[元数据]**

    结果集将更好地尝试将游标描述中存在的 DBAPI 类型与方言定义的 TypeEngine 对象进行匹配，然后用于结果处理。请注意，这仅对文本 SQL 生效；构造的 SQL 语句始终具有显式类型映射。

+   **[metadata]**

    CRUD 操作的结果集立即关闭其底层游标，并且如果为操作定义了自动关闭连接，则也将自动关闭连接；这允许更有效地使用连接进行连续的 CRUD 操作，减少“悬挂连接”的机会。

+   **[metadata]**

    列默认值和 onupdate Python 函数（即传递给 ColumnDefault 的函数）可以接受零个或一个参数；一个参数是 ExecutionContext，您可以从中调用“context.parameters[someparam]”来访问附加到语句的其他绑定参数值。所使用的连接也可用，以便您可以预执行语句。

    参考：[#559](https://www.sqlalchemy.org/trac/ticket/559)

+   **[metadata]**

    为序列（即你可以在 Sequence 的每个方法上传递一个“connectable”来支持“显式”创建/删除/执行）添加了“显式”创建/删除/执行的支持。

+   **[metadata]**

    在操纵模式标识符时更好地引用标识符。

+   **[metadata]**

    标准化了表反射的行为，当无法定位类型时，将替换为 NullType，并发出警告。

+   **[metadata]**

    ColumnCollection（即表上的‘c’属性）遵循“__contains__”的字典语义。

    参考：[#606](https://www.sqlalchemy.org/trac/ticket/606)

+   **[engines]**

    速度！结果处理和绑定参数处理的机制已进行了彻底的改进、简化和优化，以尽可能少地发出方法调用。大量 INSERT 和大量行集迭代的基准测试显示，0.4 的速度是 0.3 的两倍多，使用的函数调用减少了 68%。

+   **[engines]**

    您现在可以钩入池的生命周期，并在每次新的 DBAPI 连接、池检出和检入时运行 SQL 语句或其他逻辑。

+   **[engines]**

    连接增加了 .properties 集合，其中的内容限定为底层 DBAPI 连接的生命周期

+   **[engines]**

    从 Pool 中删除了 auto_close_cursors 和 disallow_open_cursors 参数；由于游标通常由 ResultProxy 和 Connection 关闭，因此减少了开销。

+   **[postgres]**

    添加了 PGArray 数据类型，用于使用 postgres 数组数据类型。

## 0.4.8

发布日期：Sun Oct 12 2008

### orm

+   **[orm]**

    修复了与传递的 inherit_condition 中的“A=B”与“B=A”导致错误的错误。

    参考：[#1039](https://www.sqlalchemy.org/trac/ticket/1039)

+   **[orm]**

    在 SessionExtension.before_flush() 中对新的、脏的和已删除的集合所做的更改将在该次 flush 中生效。

+   **[orm]**

    在 InstrumentedAttribute 中添加了 label() 方法，以确立与 0.5 的前向兼容性。

### sql

+   **[sql]**

    column.in_(someselect) 现在可以作为列子句表达式使用，而不会使子查询蔓延到 FROM 子句中

    参考：[#1074](https://www.sqlalchemy.org/trac/ticket/1074)

### mysql

+   **[mysql]**

    添加了 MSMediumInteger 类型。

    参考：[#1146](https://www.sqlalchemy.org/trac/ticket/1146)

### sqlite

+   **[sqlite]**

    提供了一个自定义的 strftime() 函数，用于处理 1900 年之前的日期。

    参考：[#968](https://www.sqlalchemy.org/trac/ticket/968)

+   **[sqlite]**

    在 sqlite 方言中禁用了 String（以及 Unicode、UnicodeText 等）的 convert_unicode 逻辑，以适应 pysqlite 2.5.0 的新要求，即只接受 Python unicode 对象；[`itsystementwicklung.de/pipermail/list-pysqlite/2008-March/000018.html`](https://itsystementwicklung.de/pipermail/list-pysqlite/2008-March/000018.html)

### oracle

+   **[oracle]**

    has_sequence() 现在考虑了模式名称

    参考：[#1155](https://www.sqlalchemy.org/trac/ticket/1155)

+   **[oracle]**

    将 BFILE 添加到反射类型列表中

    参考：[#1121](https://www.sqlalchemy.org/trac/ticket/1121)

### orm

+   **[orm]**

    修复了关于 inherit_condition 传递“A=B”与“B=A”导致错误的 bug

    参考：[#1039](https://www.sqlalchemy.org/trac/ticket/1039)

+   **[orm]**

    在 SessionExtension.before_flush() 中对新的、脏的和已删除的集合所做的更改将对该刷新生效。

+   **[orm]**

    在 InstrumentedAttribute 中添加了 label() 方法，以确立与 0.5 的向前兼容性。

### sql

+   **[sql]**

    column.in_(someselect) 现在可以作为一个列子句表达式使用，而不会导致子查询泄漏到 FROM 子句中

    参考：[#1074](https://www.sqlalchemy.org/trac/ticket/1074)

### mysql

+   **[mysql]**

    添加了 MSMediumInteger 类型。

    参考：[#1146](https://www.sqlalchemy.org/trac/ticket/1146)

### sqlite

+   **[sqlite]**

    提供了一个自定义的 strftime() 函数，用于处理 1900 年之前的日期。

    参考：[#968](https://www.sqlalchemy.org/trac/ticket/968)

+   **[sqlite]**

    在 sqlite 方言中禁用了 String（以及 Unicode、UnicodeText 等）的 convert_unicode 逻辑，以适应 pysqlite 2.5.0 的新要求，即只接受 Python unicode 对象；[`itsystementwicklung.de/pipermail/list-pysqlite/2008-March/000018.html`](https://itsystementwicklung.de/pipermail/list-pysqlite/2008-March/000018.html)

### oracle

+   **[oracle]**

    has_sequence() 现在考虑了模式名称

    参考：[#1155](https://www.sqlalchemy.org/trac/ticket/1155)

+   **[oracle]**

    将 BFILE 添加到反射类型列表中

    参考：[#1121](https://www.sqlalchemy.org/trac/ticket/1121)

## 0.4.7p1

发布日期：Thu Jul 31 2008

### orm

+   **[orm]**

    在 scoped_session 方法中添加了“add()”和“add_all()”。0.4.7 的解决方法：

    ```py
    from sqlalchemy.orm.scoping import ScopedSession, instrument

    setattr(ScopedSession, "add", instrument("add"))
    setattr(ScopedSession, "add_all", instrument("add_all"))
    ```

+   **[orm]**

    修复了在 relation() 内部使用非 2.3 兼容的 set() 和生成器表达式的问题。

### orm

+   **[orm]**

    在 scoped_session 方法中添加了“add()”和“add_all()”。0.4.7 的解决方法：

    ```py
    from sqlalchemy.orm.scoping import ScopedSession, instrument

    setattr(ScopedSession, "add", instrument("add"))
    setattr(ScopedSession, "add_all", instrument("add_all"))
    ```

+   **[orm]**

    修复了在 relation() 内部使用非 2.3 兼容的 set() 和生成器表达式的问题。

## 0.4.7

发布日期：Sat Jul 26 2008

### orm

+   **[orm]**

    当与多对多一起使用时，contains() 操作符会对次要（关联）表进行别名处理，以便多个 contains() 调用不会互相冲突

    参考：[#1058](https://www.sqlalchemy.org/trac/ticket/1058)

+   **[orm]**

    修复了阻止 merge() 与 comparable_property() 结合使用的错误

+   **[orm]**

    现在在 relation() 上设置 enable_typechecks=False 只允许具有继承映射器的子类型。完全不相关的类型，或者未设置为针对目标映射器进行映射器���承的子类型仍然不被允许。

+   **[orm]**

    向 Sessions 添加了 is_active 标志以检测事务是否正在进行中。该标志在“transactional”（在 0.5 版本中为非“自动提交”）会话中始终为 True。

    参考：[#976](https://www.sqlalchemy.org/trac/ticket/976)

### sql

+   **[sql]**

    修复了调用 select([literal(‘foo’)]) 或 select([bindparam(‘foo’)]) 时的错误。

### 模式

+   **[模式]**

    create_all()、drop_all()、create()、drop() 如果表名或模式名包含的字符数超过该方言配置的字符限制，则会引发错误。一些数据库可以在使用过程中处理过长的表名，SQLA 也可以处理这种情况。但是由于我们正在查找 DB 的目录表中的名称，因此各种反射/在创建期间进行 checkfirst 的场景会失败。

    参考：[#571](https://www.sqlalchemy.org/trac/ticket/571)

+   **[模式]**

    当在 Column 上说“index=True”时生成的索引名称会被截断为适合该方言的长度。此外，具有过长名称的索引不能使用 Index.drop() 明确删除，类似于。

    参考：[#571](https://www.sqlalchemy.org/trac/ticket/571)，[#820](https://www.sqlalchemy.org/trac/ticket/820)

### mysql

+   **[mysql]**

    将“CALL”添加到返回结果行的 SQL 关键字列表中。

### oracle

+   **[oracle]**

    Oracle get_default_schema_name() 在返回之前会“规范化”名称，这意味着当检测到标识符不区分大小写时，它会返回一个小写名称。

+   **[oracle]**

    创建/删除表时会考虑模式名称，以便在搜索现有表时不会与其他所有者命名空间中具有相同名称的表发生冲突

    参考：[#709](https://www.sqlalchemy.org/trac/ticket/709)

+   **[oracle]**

    游标现在默认情况下的“arraysize”设置为 50，其值可以使用 Oracle 方言的 create_engine() 中的“arraysize”参数进行配置。这是为了考虑 cx_oracle 的默认设置为“1”，这会导致向 Oracle 发送许多往返。这实际上与 BLOB/CLOB 绑定的游标很好地配合使用，其中有任意数量可用，但仅在该行请求的生命周期内（因此仍然需要 BufferedColumnRow，但不那么多）。

    参考：[#1062](https://www.sqlalchemy.org/trac/ticket/1062)

+   **[oracle]**

    sqlite

    +   添加了 SLFloat 类型，与 SQLite REAL 类型亲和。以前只提供了 SLNumeric，它满足 NUMERIC 亲和性，但与 REAL 不同。

### 杂项

+   **[postgres]**

    修复了 server_side_cursors 以正确检测 text() 子句。

+   **[postgres]**

    添加了 PGCidr 类型。

    参考：[#1092](https://www.sqlalchemy.org/trac/ticket/1092)

### orm

+   **[orm]**

    当与多对多一起使用时，contains() 运算符将为次要（关联）表设置别名，以便多个 contains() 调用不会互相冲突。

    参考：[#1058](https://www.sqlalchemy.org/trac/ticket/1058)

+   **[orm]**

    修复了阻止 merge() 与 comparable_property() 结合使用的 bug。

+   **[orm]**

    relation() 上的 enable_typechecks=False 设置现在只允许具有继承映射器的子类型。完全不相关的类型，或者未设置与目标映射器的映射器继承的子类型仍然不被允许。

+   **[orm]**

    在 Sessions 中添加了 is_active 标志，用于检测事务是否正在进行中。当使用“transactional”（在 0.5 版本中是非“autocommit”）会话时，此标志始终为 True。

    参考：[#976](https://www.sqlalchemy.org/trac/ticket/976)

### sql

+   **[sql]**

    修复了调用 select([literal(‘foo’)]) 或 select([bindparam(‘foo’)]) 时的 bug。

### schema

+   **[schema]**

    create_all()、drop_all()、create()、drop() 如果表名或模式名包含的字符数超过该方言配置的字符限制，则会引发错误。一些数据库可以在使用过程中处理过长的表名，SQLA 也可以处理这种情况。但是由于我们正在查找 DB 的目录表中的名称，因此各种反射/在创建期间进行 checkfirst 的场景会失败。

    参考：[#571](https://www.sqlalchemy.org/trac/ticket/571)

+   **[schema]**

    当在 Column 上说“index=True”时生成的索引名称会被截断为适合该方言的长度。此外，具有过长名称的索引无法使用 Index.drop() 明确删除，类��于。

    参考：[#571](https://www.sqlalchemy.org/trac/ticket/571), [#820](https://www.sqlalchemy.org/trac/ticket/820)

### mysql

+   **[mysql]**

    将 'CALL' 添加到返回结果行的 SQL 关键字列表中。

### oracle

+   **[oracle]**

    Oracle 的 get_default_schema_name() 在返回之前会“规范化”名称，这意味着当检测到标识符为大小写不敏感时，它会返回小写名称。

+   **[oracle]**

    在搜索现有表时，创建/删除表时会考虑模式名称，以避免与其他所有者命名空间中具有相同名称的表发生冲突。

    参考：[#709](https://www.sqlalchemy.org/trac/ticket/709)

+   **[oracle]**

    游标现在默认情况下具有“arraysize”设置为 50，其值可以使用 Oracle 方言的“arraysize”参数进行配置。这是为了考虑 cx_oracle 的默认设置为“1”，这会导致向 Oracle 发送许多往返。这实际上与 BLOB/CLOB 绑定的游标很好地配合使用，其中有任意数量可用，但仅在该行请求的生命周期内（因此仍然需要 BufferedColumnRow，但不那么需要）。

    参考：[#1062](https://www.sqlalchemy.org/trac/ticket/1062)

+   **[oracle]**

    sqlite

    +   添加了`SLFloat`类型，该类型与 SQLite 的`REAL`类型亲和。以前，只提供了`SLNumeric`，它满足`NUMERIC`亲和性，但这与`REAL`不同。

### 杂项

+   **[postgres]**

    修复了`server_side_cursors`以正确检测`text()`子句。

+   **[postgres]**

    添加了`PGCidr`类型。

    参考：[#1092](https://www.sqlalchemy.org/trac/ticket/1092)

## 0.4.6

发布日期：2008 年 5 月 10 日（周六）

### orm

+   **[orm]**

    修复了最近的`relation()`重构，修复了在本地表和远程表之间多次连接的异国`viewonly`关系，这些连接在连接之间共享公共列。

+   **[orm]**

    还重新建立了跨多个表连接的`viewonly relation()`配置。

+   **[orm]**

    增加了实验性的`relation()`标志，以帮助跨函数等处的`primaryjoins`，_local_remote_pairs=[tuples]。这补充了一个复杂的`primaryjoin`条件，允许您提供构成关系的本地和远程方的单个列对。还改进了延迟加载 SQL 生成，以处理将绑定参数放置在函数和其他表达式中的情况。（部分进展）

    参考：[#610](https://www.sqlalchemy.org/trac/ticket/610)

+   **[orm]**

    修复了单表继承，使您可以从加入表继承的映射器中单表继承而无需问题。

    参考：[#1036](https://www.sqlalchemy.org/trac/ticket/1036)

+   **[orm]**

    修复了“连接元组”错误，该错误可能发生在`Query.order_by()`中，如果进行了子句调整。

    参考：[#1027](https://www.sqlalchemy.org/trac/ticket/1027)

+   **[orm]**

    移除了古老的断言，即映射的可选择性需要“别名” - 如果没有，则映射器现在会创建自己的别名。尽管在这种情况下，您需要使用类，而不是映射的可选择性，作为列属性的来源 - 因此仍会发出警告。

+   **[orm]**

    修复了涉及继承的“exists”函数的问题（`any()`，`has()`，`~contains()`）；完整的目标连接将被呈现到用于与子类关联的`EXISTS`子句中。

+   **[orm]**

    恢复了仅在扩展存在且仅返回单个实体结果时为主查询行添加`append_result()`扩展方法的用法。

+   **[orm]**

    还重新建立了跨多个表连接的`viewonly relation()`配置。

+   **[orm]**

    移除了古老的断言，即映射的可选择性需要“别名” - 如果没有，则映射器现在会创建自己的别名。尽管在这种情况下，您需要使用类，而不是映射的可选择性，作为列属性的来源 - 因此仍会发出警告。

+   **[orm]**

    优化了`mapper._save_obj()`，在`flush`期间不必要地调用了标量值的`__ne__()`。

    参考：[#1015](https://www.sqlalchemy.org/trac/ticket/1015)

+   **[orm]**

    对急加载功能进行了更新，其中使用具有显式标签名称的 column_property() 设置的子查询（顺便说一句，这并不是必要的）在实例作为急加载连接的一部分时，将对标签进行匿名化处理，以防止与父对象上具有相同名称的子查询或列发生冲突。

    参考：[#1019](https://www.sqlalchemy.org/trac/ticket/1019)

+   **[orm]**

    集合运算符 |=、-=、^= 和 &= 对其操作数更加严格，只能操作集合、frozensets 或集合类型的子类。以前，它们接受任何鸭子类型的集合。

+   **[orm]**

    添加了一个示例 dynamic_dict/dynamic_dict.py，演示了在 dynamic_loader 之上放置字典行为的简单方法。

### sql

+   **[sql]**

    通过 .collate(<collation>) 表达式操作符和 collate(<expr>, <collation>) sql 函数添加了 COLLATE 支持。

+   **[sql]**

    修复了应用于非表连接选择语句的 union() 的错误。

+   **[sql]**

    改进了在作为 FROM 子句使用时的 text() 表达式的行为，例如 select().select_from(text(“sometext”))

    参考：[#1014](https://www.sqlalchemy.org/trac/ticket/1014)

+   **[sql]**

    Column.copy() 尊重 “autoincrement” 的值，修复了与 Migrate 的使用。

    参考：[#1021](https://www.sqlalchemy.org/trac/ticket/1021)

### mssql

+   **[mssql]**

    在 engine / dburi 参数中添加了 “odbc_autotranslate” 参数。任何给定的字符串将传递到 ODBC 连接字符串中：

    > ”AutoTranslate=%s” % odbc_autotranslate

    参考：[#1005](https://www.sqlalchemy.org/trac/ticket/1005)

+   **[mssql]**

    在 engine / dburi 参数中添加了 “odbc_options” 参数。给定的字符串将简单地附加到 SQLAlchemy 生成的 odbc 连接字符串中。

    这应该可以避免将来需要添加大量 ODBC 选项的需要。

### 杂项

+   **[declarative] [extension]**

    联合表继承映射器使用稍微放松的函数来创建父表的 “继承条件”，以便对尚未声明的 Table 对象的其他外键不会触发错误。

+   **[declarative] [extension]**

    修复了当在 ForeignKey 中使用已声明属性时出现的重入映射器编译挂起的 bug，即 ForeignKey(MyOtherClass.someattribute)。

+   **[engines]**

    现在可以将池监听器提供为可调用对象的字典或 PoolListener 的（可能是部分的）鸭子类型，任你选择。

+   **[engines]**

    在 Pool 中添加了 “rollback_returned” 选项，该选项将禁用返回连接时发出的 rollback()。此标志仅在不支持事务的数据库（例如 MySQL/MyISAM）中使用时才安全。

+   **[ext]**

    集合运算符 |=、-=、^= 和 &= 对其操作数更加严格，只能操作集合、frozensets 或其他关联代理。以前，它们接受任何鸭子类型的集合。

+   **[firebird]**

    处理 “SUBSTRING(:string FROM :start FOR :length)” 内建函数。

### orm

+   **[orm]**

    修复了最近 relation() 重构中的问题，修复了在本地表和远程表之间多次进行连接的特殊视图只读关系，并且这些连接之间有一个共享列的情况。

+   **[orm]**

    同时重新建立了跨多个表连接的视图只读关系配置。

+   **[orm]**

    添加了用于帮助跨函数等进行主连接的实验性 relation() 标志，_local_remote_pairs=[tuples]。这与复杂的 primaryjoin 条件相辅相成，允许您提供构成关系本地和远程侧的个别列对。还改进了惰性加载 SQL 生成，以处理将绑定参数放置在函数和其他表达式内的情况。（部分进展）

    参考：[#610](https://www.sqlalchemy.org/trac/ticket/610)

+   **[orm]**

    修复了单表继承，以便您可以从具有连接表继承的映射器进行单表继承而不会出现问题。

    参考：[#1036](https://www.sqlalchemy.org/trac/ticket/1036)

+   **[orm]**

    修复了 Query.order_by() 中可能发生的“连接元组”错误，如果进行了子句适应。

    参考：[#1027](https://www.sqlalchemy.org/trac/ticket/1027)

+   **[orm]**

    移除了关于映射可选择项需要“别名”的古老断言 - 如果没有别名，则映射现在会自己创建别名。不过，在这种情况下，您需要使用类，而不是映射的可选择项，作为列属性的来源 - 因此仍会发出警告。

+   **[orm]**

    修复了涉及继承的“exists”函数（any()、has()、~contains()）的问题；对于连接到子类的关系，完整的目标连接将被呈现到 EXISTS 子句中。

+   **[orm]**

    恢复了对主查询行使用 append_result() 扩展方法的使用，当存在该扩展并且只返回单个实体结果时。

+   **[orm]**

    同时重新建立了跨多个表连接的视图只读关系配置。

+   **[orm]**

    移除了关于映射可选择项需要“别名”的古老断言 - 如果没有别名，则映射现在会自己创建别名。不过，在这种情况下，您需要使用类，而不是映射的可选择项，作为列属性的来源 - 因此仍会发出警告。

+   **[orm]**

    优化了 mapper._save_obj()，在 flush 期间不必要地对标量值调用 __ne__()。

    参考：[#1015](https://www.sqlalchemy.org/trac/ticket/1015)

+   **[orm]**

    增加了一个特性，即贪婪加载中的子查询被设置为 column_property() 并具有显式标签名称（顺便说一句，这并不是必需的）时，当实例成为贪婪连接的一部分时，标签将被匿名化，以防止与父对象上的同名子查询或列发生冲突。

    参考：[#1019](https://www.sqlalchemy.org/trac/ticket/1019)

+   **[orm]**

    使用集合操作符 |=、-=、^= 和 &= 时对其操作数的要求更为严格，只能操作集合、冻结集合或其子类的集合类型。之前，它们会接受任何类似集合的对象。

+   **[orm]**

    添加了一个示例 dynamic_dict/dynamic_dict.py，演示了在 dynamic_loader 之上放置字典行为的简单方法。

### sql

+   **[sql]**

    通过.collate(<collation>)表达式操作符和 collate(<expr>, <collation>) sql 函数添加了 COLLATE 支持。

+   **[sql]**

    修复了将 union()应用于非 Table 连接的 select 语句时的错误

+   **[sql]**

    当作为 FROM 子句使用时，text()表达式的行为得到改进，例如 select().select_from(text(“sometext”))

    参考：[#1014](https://www.sqlalchemy.org/trac/ticket/1014)

+   **[sql]**

    Column.copy()尊重“autoincrement”的值，修复了与 Migrate 的使用

    参考：[#1021](https://www.sqlalchemy.org/trac/ticket/1021)

### mssql

+   **[mssql]**

    向 engine / dburi 参数添加了“odbc_autotranslate”参数。任何给定的字符串都将传递到 ODBC 连接字符串中：

    > ”AutoTranslate=%s” % odbc_autotranslate

    参考：[#1005](https://www.sqlalchemy.org/trac/ticket/1005)

+   **[mssql]**

    向 engine / dburi 参数添加了“odbc_options”参数。给定的字符串简单地附加到 SQLAlchemy 生成的 odbc 连接字符串上。

    这应该消除将来添加大量 ODBC 选项的需要。

### 杂项

+   **[声明式] [扩展]**

    连接表继承映射器使用稍微放松的函数来创建到父表的“继承条件”，以便对尚未声明的 Table 对象的其他外键不触发错误。

+   **[声明式] [扩展]**

    修复了在 ForeignKey 中使用已声明属性时的重入映射编译挂起问题，即 ForeignKey(MyOtherClass.someattribute)

+   **[引擎]**

    现在可以将池监听器提供为可调用的字典或（可能是部分）PoolListener 的鸭子类型，任君选择。

+   **[引擎]**

    向 Pool 添加了“rollback_returned”选项，该选项将禁用在连接返回时发出的 rollback()。此标志仅适用于不支持事务的数据库（���MySQL/MyISAM）。

+   **[扩展]**

    基于集合的关联代理 |=, -=, ^= 和 &= 对其操作数更为严格，仅操作于集合、frozenset 或其他关联代理。以前，它们会接受任何鸭子类型的集合。

+   **[firebird]**

    处理“SUBSTRING(:string FROM :start FOR :length)”内置函数。

## 0.4.5

发布日期：2008 年 4 月 4 日星期五

### ORM

+   **[ORM]**

    对 session.merge()行为的微小更改 - 现有对象根据主键属性而不一定是 _instance_key 进行检查。因此，广泛请求的功能是：

    > x = MyObject(id=1) x = sess.merge(x)

    如果存在，现在可以加载数据库中具有 id＃1 的 MyObject。merge()仍将给定对象的状态复制到持久对象上，因此上述示例通常会将“x”的所有属性的“None”复制到持久副本上。这些可以使用 session.expire(x)来恢复。

+   **[ORM]**

    还修复了 merge() 中的行为，合并后集合中存在但合并集合中不存在的集合元素未从目标中删除的问题。

+   **[orm]**

    添加了对“未编译的映射器”的更积极的检查，特别有助于声明性层

    参考：[#995](https://www.sqlalchemy.org/trac/ticket/995)

+   **[orm]**

    “primaryjoin”/“secondaryjoin”背后的方法论已经重构。行为应该稍微更智能一些，主要体现在错误消息方面，已经简化为更易读的形式。在少量情况下，它可以比以前更好地解析正确的外键。

+   **[orm]**

    添加了 comparable_property()，将查询比较器行为添加到常规的、非托管的 Python 属性

+   **[orm] [‘machines’] [Company.employees.of_type(Engineer)]**

    将 query.with_polymorphic() 的功能添加到 mapper() 作为配置选项。

    它可以通过几种形式进行设置：

    with_polymorphic=’*’ with_polymorphic=[mappers] with_polymorphic=(‘*’，selectable) with_polymorphic=([mappers]，selectable)

    这控制了继承映射器的默认多态加载策略。当没有给出可选择项时，将为所有请求的连接表继承映射器创建外连接。请注意，连接的自动创建与具体表继承不兼容。

    现有的 mapper() 上的 select_table 标志现在已弃用，并且与 with_polymorphic(' *'，select_table) 等同。请注意，select_table 的底层“要点”已完全删除并替换为更新的、更灵活的方法。

    新方法还自动允许对子类进行急切加载，如果它们存在，例如：

    ```py
    sess.query(Company).options(eagerload_all())
    ```

    加载公司对象、其员工以及偶然是工程师的员工集合的“machines”集合。很快将引入“with_polymorphic”查询选项，该选项允许每个查询对关系上的 with_polymorphic()进行控制。

+   **[orm]**

    在查询中添加了两个“实验性”功能，“实验性”是因为它们的具体名称/行为尚未确定：_values() 和 _from_self()。我们希望获得对这些功能的反馈。

    +   _values(*columns) 接收列表达式列表，并返回一个仅返回这些列的新查询。在评估时，返回值是一个元组列表，就像使用 add_column() 或 add_entity() 时一样，唯一的区别是“entity zero”，即映射类，不包含在结果中。这意味着现在终于可以在查询上使用 group_by() 和 having()，直到现在一直没有用。

        未来对此方法的更改可能包括删除其连接、过滤和允许与“结果集”无关的其他选项的能力，因此我们寻求的反馈是人们希望如何使用 _values()…即在最后完成后，还是人们更喜欢在调用后继续生成。

    +   _from_self() 编译 Query 的 SELECT 语句（减去任何急加载器），并返回一个从该 SELECT 选择的新 Query。所以基本上你可以从一个 Query 中查询而不需要手动提取 SELECT 语句。这样做就给了 query[3:5]._from_self().filter(some criterion)这样的操作有意义。这里没有什么争议性的，除了你可以快速创建效率较低的高度嵌套的查询，并且我们希望得到有关命名选择的反馈。

+   **[orm]**

    query.order_by() 和 query.group_by() 现在将接受多个参数使用 *args（就像 select() 已经做的那样）。

+   **[orm]**

    在 Query 中添加了一些便利描述符：query.statement 返回完整的 SELECT 构造，query.whereclause 只返回 SELECT 构造的 WHERE 部分。

+   **[orm]**

    修复/涵盖了使用 False/0 值作为多态判别器时的情况。

+   **[orm]**

    修复了一个 bug，该 bug 阻止了同义词（synonym()）属性与继承一起使用。

+   **[orm]**

    修复了对尾随下划线的 SQL 函数截断

    参考：[#996](https://www.sqlalchemy.org/trac/ticket/996)

+   **[orm]**

    当在挂起的实例上过期属性时，当触发“刷新”操作并且未找到结果时将不会引发错误。

+   **[orm]**

    Session.execute 现在可以从元数据中找到绑定

+   **[orm]**

    调整了“自引用”的定义，使其成为具有共同父项的任何两个映射器（这会影响与 Query 连接时是否需要 aliased=True）。

+   **[orm]**

    对 query.join() 的 “from_joinpoint” 参数进行了一些修正，以便如果前一个连接被别名化，而这个连接没有，则连接仍然成功进行。

+   **[orm]**

    各种“级联删除”修复：

    +   修复了动态关系的“级联删除”操作，在 0.4.2 中该操作仅针对外键置空行为进行了实现，并未实际进行级联删除

    +   在 many-to-one 上没有 delete-orphan 级联的 delete 级联将不会删除与父项断开连接的孤立项，直到在父项上调用 session.delete()（one-to-many 已经有了这个）。

    +   delete 级联与 delete-orphan 将删除孤立项，无论它是否仍附加到已删除的父项。

    +   在使用继承时，delete-orphan 级联现在可以正确地检测到存在于超类上的关系。

    参考：[#895](https://www.sqlalchemy.org/trac/ticket/895)

+   **[orm]**

    修复了 Query 中 order_by 计算的错误，在使用 select_from()时正确别名 mapper-config 中的 order_by。

+   **[orm]**

    将在替换一个集合为另一个集合时启动的差异逻辑重构为 collections.bulk_replace，对于构建多级集合的任何人都很有用。

+   **[orm]**

    级联遍历算法从递归转换为迭代，以支持深层对象图。

### sql

+   **[sql]**

    现在，对架构限定的表将在所有列表达式以及生成列标签时将架构名称放在表名称之前。这样可以防止在所有情况下跨架构名称冲突

    参考：[#999](https://www.sqlalchemy.org/trac/ticket/999)

+   **[sql]**

    现在可以允许选择将所有 FROM 子句相关联且自身没有 FROM 的选择。这些通常在标量上下文中使用，即 SELECT x, (SELECT x WHERE y) FROM table。需要显式 correlate() 调用。

+   **[sql]**

    'name' 不再是 Column() 的必需构造函数参数。现在可以将其（和 .key）推迟到列添加到表时。

+   **[sql]**

    like()、ilike()、contains()、startswith()、endswith() 接受一个可选的关键字参数“escape=<somestring>”，使用“x LIKE y ESCAPE '<somestring>'”语法设置为转义字符。

    参考：[#791](https://www.sqlalchemy.org/trac/ticket/791), [#993](https://www.sqlalchemy.org/trac/ticket/993)

+   **[sql]**

    random() 现在是一个通用的 SQL 函数，并且将编译为数据库的随机实现（如果有）。

+   **[sql]**

    update().values() 和 insert().values() 接受关键字参数。

+   **[sql]**

    修复了 select() 中关于其生成 FROM 子句的问题，在罕见情况下，两个子句可能会被生成，而实际上只想取消另一个。一些具有大量急加载的 ORM 查询可能会出现这种症状。

+   **[sql]**

    case() 函数现在还接受字典作为其 whens 参数。它还默认将“THEN”表达式解释为值，这意味着 case([(x==y, “foo”)]) 将“foo”解释为绑定值，而不是 SQL 表达式。在这种情况下，对于标准本身，这些只能是文本字符串，除非存在“value”关键字，否则 SA 将强制使用 text() 或 literal()。

### mysql

+   **[mysql]**

    方言用于缓存服务器设置的连接信息键已更改，并且现在有了命名空间。

### mssql

+   **[mssql]**

    反射表现在将自动加载被自动加载表中的外键引用的其他表。

    参考：[#979](https://www.sqlalchemy.org/trac/ticket/979)

+   **[mssql]**

    添加了 executemany 检查以跳过标识获取。

    参考：[#916](https://www.sqlalchemy.org/trac/ticket/916)

+   **[mssql]**

    添加了小日期类型的存根。

    参考：[#884](https://www.sqlalchemy.org/trac/ticket/884)

+   **[mssql]**

    为 pyodbc 方言添加了新的 'driver' 关键字参数。如果给出，将替代 ODBC 连接字符串，默认为 'SQL Server'。

+   **[mssql]**

    为 pyodbc 方言添加了新的 'max_identifier_length' 关键字参数。

+   **[mssql]**

    对 pyodbc + Unix 的改进。如果以前无法使该组合工作，请再次尝试。

### oracle

+   **[oracle]**

    Table 上的 "owner" 关键字现已弃用，并且与 "schema" 关键字完全同义。现在可以在 Table 对象上明确声明或不使用 "schema" 来反射具有替代 "owner" 属性的表。

+   **[oracle]**

    所有在表反射期间搜索同义词、DBLINK 等“魔术”功能默认情况下都已禁用，除非在 Table 对象上指定“oracle_resolve_synonyms=True”。解析同义词必然会导致一些混乱的猜测，我们宁愿默认情况下将其排除在外。当设置了标志时，表和相关表将在所有情况下针对同义词进行解析，这意味着如果特定表存在同义词，则在反映相关表时将使用它。这比以前的行为更粘性，这就是为什么默认情况下它是关闭的。

### 杂项

+   **[声明式] [扩展]**

    “synonym”函数现在可以直接与“declarative”一起使用。使用“descriptor”关键字参数传递装饰的属性，例如：somekey = synonym(‘_somekey’, descriptor=property(g, s))

+   **[声明式] [扩展]**

    “deferred”函数可与“declarative”一起使用。最简单的用法是一起声明 deferred 和 Column，例如：data = deferred(Column(Text))

+   **[声明式] [扩展]**

    Declarative 还获得了@synonym_for(…)和@comparable_using(…)，这是 synonym 和 comparable_property 的前端。

+   **[声明式] [扩展]**

    在使用 declarative 时改进 mapper 编译；已经编译的 mapper 在使用时仍会触发其他未编译的 mapper 的编译

    参考：[#995](https://www.sqlalchemy.org/trac/ticket/995)

+   **[声明式] [扩展]**

    Declarative 将为缺少名称的 Columns 完成设置，允许更 DRY 的语法。

    > class Foo(Base):
    > 
    > __tablename__ = ‘foos’ id = Column(Integer, primary_key=True)

+   **[声明式] [扩展]**

    在 declarative 中，通过将“inherits=None”发送到 __mapper_args__ 可以禁用继承。

+   **[声明式] [扩展]**

    declarative_base()接受可选的 kwarg“mapper”，这是任何可调用/类/方法，可以生成一个 mapper，比如 declarative_base(mapper=scopedsession.mapper)。这个属性也可以在单独的 declarative 类上使用“__mapper_cls__”属性进行设置。

+   **[postgres]**

    将 PG 服务器端游标恢复正常，作为默认测试套件的一部分添加了固定的单元测试。为游标 ID 添加了更好的唯一性

    参考：[#1001](https://www.sqlalchemy.org/trac/ticket/1001)

### orm

+   **[orm]**

    对 session.merge()的行为进行了一点改变 - 现有对象是基于主键属性而不一定是 _instance_key 进行检查的。因此，广泛请求的功能，即：

    > x = MyObject(id=1) x = sess.merge(x)

    实际上，如果存在，将从数据库中加载具有 id＃1 的 MyObject，现在可用。merge()仍将给定对象的状态复制到持久对象上，因此像上面的示例通常会将“x”的所有属性的“None”复制到持久副本上。这些可以使用 session.expire(x)来恢复。

+   **[orm]**

    修复了 merge()中的行为，即目标上存在但合并集合中不存在的元素未从目标中删除。

+   **[orm]**

    添加了对“未编译的映射器”更积极的检查，特别有助于声明层

    参考：[#995](https://www.sqlalchemy.org/trac/ticket/995)

+   **[orm]**

    “primaryjoin”/“secondaryjoin”背后的方法已经重构。行为应该稍微更智能，主要是在错误消息方面，已经简化为更易读。在少数情况下，它可以比以前更好地解析正确的外键。

+   **[orm]**

    添加 comparable_property()，将查询比较器行为添加到常规的、未管理的 Python 属性

+   **[orm] ['machines'] [Company.employees.of_type(Engineer)]**

    query.with_polymorphic()的功能已添加到 mapper()作为配置选项。

    它通过几种形式设置：

    with_polymorphic='*' with_polymorphic=[mappers] with_polymorphic=(' * ', selectable) with_polymorphic=([mappers], selectable)

    这控制了继承映射器的默认多态加载策略。当没有给出可选择项时，将为所有请求的连接表继承映射器创建外连接。请注意，连接的自动创建与具体表继承不兼容。

    现有的 mapper()上的 select_table 标志现在已被弃用，并与 with_polymorphic(' * ', select_table)同义。请注意，select_table 的底层“内部”已被完全删除，并替换为更新更灵活的方法。

    新方法还自动允许对子类进行急加载，如果存在的话，例如：

    ```py
    sess.query(Company).options(eagerload_all())
    ```

    加载 Company 对象、它们的员工以及恰好是工程师的员工的“machines”集合。很快还应该引入一个“with_polymorphic”Query 选项，它将允许对关系上的 with_polymorphic()进行每个 Query 的控制。

+   **[orm]**

    在 Query 中添加了两个“实验性”功能，“实验性”是因为它们的具体名称/行为尚未最终确定：_values()和 _from_self()。我们希望得到这些的反馈。

    +   _values(*columns)给出一个列表达式列表，并返回一个仅返回这些列的新 Query。在评估时，返回值是一个元组列表，就像使用 add_column()或 add_entity()时一样，唯一的区别是结果中不包括“实体零”，即映射类。这意味着现在终于可以在 Query 上使用 group_by()和 having()，这些功能一直无用至今。

        对该方法的未来更改可能包括删除其加入、过滤和允许其他与“结果集”无关的选项的能力，因此我们寻求的反馈是人们如何想要使用 _values()...即在最后，还是人们更喜欢在调用后继续生成。

    +   _from_self()编译 Query 的 SELECT 语句（减去任何急加载器），并返回一个新的 Query，从该 SELECT 中选择。因此，您可以从一个 Query 中查询而无需手动提取 SELECT 语句。这使得像 query[3:5]._from_self().filter(some criterion)这样的操作具有意义。这里没有太多争议，除了您可以快速创建效率较低的高度嵌套查询，我们希望得到有关命名选择的反馈。

+   **[orm]**

    query.order_by()和 query.group_by()将接受使用*args 的多个参数（就像 select()已经做的那样）。

+   **[orm]**

    在 Query 中添加了一些便利描述符：query.statement 返回完整的 SELECT 结构，query.whereclause 仅返回 SELECT 结构中的 WHERE 部分。

+   **[orm]**

    修复了在使用 False/0 值作为多态鉴别器时的情况。

+   **[orm]**

    修复了阻止使用继承的 synonym()���性的错误。

+   **[orm]**

    修复了 SQL 函数截断尾随下划线。

    参考：[#996](https://www.sqlalchemy.org/trac/ticket/996)

+   **[orm]**

    当挂起实例上的属性过期时，在触发“refresh”操作并且没有找到结果时不会引发错误。

+   **[orm]**

    Session.execute 现在可以从元数据中找到绑定

+   **[orm]**

    调整了“自引用”的定义，即任何具有共同父级的两个映射器（这会影响是否在与 Query 连接时需要 aliased=True）。 

+   **[orm]**

    对 query.join()中的“from_joinpoint”参数进行了一些修复，以便如果前一个连接被别名化而当前连接没有，连接仍然成功进行。

+   **[orm]**

    各种“级联删除”修复：

    +   修复了动态关系的“级联删除”操作，该操作在 0.4.2 版本中仅实现了外键空值行为，而不是实际的级联删除。

    +   在多对一关系上没有设置级联删除或者孤儿级联时，将不会删除那些在调用父对象的 session.delete()之前与父对象断开连接的孤儿对象（一对多已经具备此功能）。

    +   设置了级联删除和孤儿删除时，无论孤儿对象是否仍然与已删除的父对象相关联，都将删除孤儿对象。

    +   在使用继承时，能够正确检测到存在于超类上的关系的级联删除。

    参考：[#895](https://www.sqlalchemy.org/trac/ticket/895)

+   **[orm]**

    修复了 Query 中 order_by 计算的问题，当使用 select_from()时，正确别名映射器配置的 order_by。

+   **[orm]**

    重构了在用另一个集合替换一个集合时触发的差异逻辑为 collections.bulk_replace，对于构建多级集合的任何人都很有用。

+   **[orm]**

    将级联遍历算法从递归转换为迭代，以支持深层对象图。

### sql

+   **[sql]**

    在架构限定的表中，现在将在所有列表达式以及生成列标签时将架构名放在表名之前。这样可以在所有情况下防止跨架构名称冲突。

    参考：[#999](https://www.sqlalchemy.org/trac/ticket/999)

+   **[sql]**

    现在可以允许选择将所有 FROM 子句相关联并且自身没有 FROM 的情况。这些通常在标量上下文中使用，即 SELECT x, (SELECT x WHERE y) FROM table。需要显式的 correlate() 调用。

+   **[sql]**

    ‘name’ 不再是 Column() 的必需构造函数参数。现在可以延迟直到将列添加到 Table 时（以及 .key）。 

+   **[sql]**

    like()、ilike()、contains()、startswith()、endswith() 接受一个可选的关键字参数“escape=<somestring>”，该参数被设置为使用语法“x LIKE y ESCAPE ‘<somestring>’”的转义字符。

    参考：[#791](https://www.sqlalchemy.org/trac/ticket/791), [#993](https://www.sqlalchemy.org/trac/ticket/993)

+   **[sql]**

    random() 现在是一个通用的 SQL 函数，并且将编译为数据库的随机实现（如果有）。

+   **[sql]**

    update().values() 和 insert().values() 接受关键字参数。

+   **[sql]**

    修复了 select() 中关于生成 FROM 子句的问题，在罕见情况下可能会产生两个子句，而本意是取消对方。一些具有大量急加载的 ORM 查询可能会看到此症状。

+   **[sql]**

    case() 函数现在还接受字典作为其 whens 参数。它还默认将“THEN”表达式解释为值，这意味着 case([(x==y, “foo”)]) 将“foo”解释为绑定值，而不是 SQL 表达式。在这种情况下，使用 text(expr) 来获取字面 SQL 表达式。对于标准本身，这些只能是字面字符串，如果“value”关键字不存在，那么 SA 将强制明确使用 text() 或 literal()。

### mysql

+   **[mysql]**

    方言用于缓存服务器设置的连接.info 键已更改，并且现在有命名空间。

### mssql

+   **[mssql]**

    反映的表现在将自动加载由自动加载表中的外键引用的其他表。

    参考：[#979](https://www.sqlalchemy.org/trac/ticket/979)

+   **[mssql]**

    添加了 executemany 检查以跳过标识获取。

    参考：[#916](https://www.sqlalchemy.org/trac/ticket/916)

+   **[mssql]**

    为小日期类型添加了存根。

    参考：[#884](https://www.sqlalchemy.org/trac/ticket/884)

+   **[mssql]**

    为 pyodbc 方言添加了一个新的‘driver’关键字参数。如果给定，将替换为 ODBC 连接字符串，默认为‘SQL Server’。

+   **[mssql]**

    为 pyodbc 方言添加了一个新的‘max_identifier_length’关键字参数。

+   **[mssql]**

    对 pyodbc + Unix 进行了改进。如果以前无法使该组合工作，请再试一次。

### oracle

+   **[oracle]**

    Table 上的“owner”关键字现已弃用，与“schema”关键字完全同义。现在可以在 Table 对象上明确声明或不使用“schema”来反映具有备用“owner”属性的表。

+   **[oracle]**

    除非在 Table 对象上指定了“oracle_resolve_synonyms=True”，否则默认情况下，表反射期间所有搜索同义词、DBLINK 等的“魔术”都将被禁用。解析同义词必然会导致一些混乱的猜测，我们宁愿默认不使用。当设置了标志时，表和相关表将在所有情况下解析同义词，这意味着如果特定表存在同义词，则在反映相关表时将使用它。这比以前的行为更具黏性，因此默认情况下关闭了它。

### 杂项

+   **[声明式] [扩展]**

    现在“synonym”函数直接可用于“声明式”。使用“descriptor”关键字参数传递装饰的属性，例如：`somekey = synonym(‘_somekey’, descriptor=property(g, s))`

+   **[声明式] [扩展]**

    “deferred”函数可与“声明式”一起使用。最简单的用法是一起声明延迟和列，例如：`data = deferred(Column(Text))`

+   **[声明式] [扩展]**

    声明式还增加了`@synonym_for(…)`和`@comparable_using(…)`，这是同义词和可比较属性的前端。

+   **[声明式] [扩展]**

    在使用声明式时，对映射器编译进行了改进；已编译的映射器在使用时仍会触发其他未编译的映射器的编译

    参考：[#995](https://www.sqlalchemy.org/trac/ticket/995)

+   **[声明式] [扩展]**

    声明式将为缺少名称的列完成设置，允许更加 DRY 的语法。

    > class Foo(Base):
    > 
    > `__tablename__ = ‘foos’ id = Column(Integer, primary_key=True)`

+   **[声明式] [扩展]**

    在声明式中，当将“inherits=None”发送到`__mapper_args__`时，可以禁用继承。

+   **[声明式] [扩展]**

    `declarative_base()`接受可选参数“mapper”，它是任何可调用的函数/类/方法，用于生成一个映射器，例如`declarative_base(mapper=scopedsession.mapper)`。这个属性也可以在单独的声明类中使用“__mapper_cls__”属性设置。

+   **[PostgreSQL]**

    使 PG 服务器端游标恢复正常，将固定单元测试添加到默认测试套件中。为游标 ID 添加了更好的唯一性

    参考：[#1001](https://www.sqlalchemy.org/trac/ticket/1001)

## 0.4.4

发布日期：Wed Mar 12 2008

### ORM

+   **[ORM]**

    `any()`, `has()`, `contains()`, `~contains()`, 属性级别的 `==` 和 `!=` 现在与自引用关系正常工作 - EXISTS 中的子句在“远程”一侧进行别名处理，以区别于父表。这适用于单表自引用以及基于继承的自引用。

+   **[ORM]**

    修复了在关系()级别使用 == 和 != 操作符与 NULL 比较时的行为，用于一对一关系

    参考：[#985](https://www.sqlalchemy.org/trac/ticket/985)

+   **[ORM]**

    修复了一个 bug，即在由 select_table 映射的多态映射实例上，session.expire() 属性未加载。

+   **[ORM]**

    添加了 query.with_polymorphic() - 指定了从基类继承的类列表，这些类将被添加到查询的 FROM 子句中。允许子类在 filter() 条件中使用，以及急切加载这些子类的属性。

+   **[orm]**

    你的呼声已经被听到：使用 delete-orphan 从属性或集合中移除一个待处理项会将该项从会话中清除；不会引发 FlushError。请注意，如果你显式地 session.save() 了待处理项，那么属性/集合的移除仍会将其排除在外。

+   **[orm]**

    当在会话中调用 session.refresh() 和 session.expire() 时，对于不在会话中的实例会引发错误。

+   **[orm]**

    修复了当使用 join() 生成多个 Query 对象时可能出现的生成性 bug。

+   **[orm]**

    修复了在 0.4.3 中引入的 bug，当使用默认的“select” polymorphic_fetch 时，加载已经持久化的实例映射到连接表继承时，会触发一个无用的从其连接表中加载的“secondary”加载。这是由于属性在第一次加载时被标记为过期，并且在之前的“secondary”加载中没有被标记为未过期。根据任何加载或提交操作成功后在 __dict__ 中的存在情况，现在会取消属性的过期标记。

+   **[orm]**

    弃用了 Query 方法 apply_sum()、apply_max()、apply_min()、apply_avg()。更好的方法学正在到来……

+   **[orm]**

    relation() 可以接受一个可调用对象作为其第一个参数，该可调用对象返回要关联的类。这是为了帮助声明性包在类尚未就位时定义关系。

+   **[orm]**

    添加了一个新的“更高级”的操作符称为“of_type()”：在 join() 中以及与 any() 和 has() 一起使用，限定将在过滤条件中使用的子类，例如：

    > query.filter(Company.employees.of_type(Engineer).
    > 
    > any(Engineer.name==’foo’))
    > 
    > 或
    > 
    > query.join(Company.employees.of_type(Engineer)).
    > 
    > filter(Engineer.name==’foo’)

+   **[orm]**

    针对 flush() 中潜在的丢失引用 bug 的预防性代码。

+   **[orm]**

    在 filter()、filter_by() 和其他表达式中使用时，如果它们使用从关系生成的子对象的标识（例如，filter(Parent.child==<somechild>))，则在执行时评估 <somechild> 的实际主键值，以便 Query 的 autoflush 步骤可以完成，从而在 <somechild> 是待处理状态时填充 <somechild> 的 PK 值。

+   **[orm]**

    将关系()-级别的 order by 设置为“secondary”表中的列现在可以与急切加载一起使用，以前的“order by”不会针对 secondary 表的别名进行别名处理。

+   **[orm]**

    现在，存在于现有描述符之上的同义词现在是这些描述符的完整代理。

### sql

+   **[sql]**

    可以再次对文本 FROM 子句创建选择的别名。

    参考：[#975](https://www.sqlalchemy.org/trac/ticket/975)

+   **[sql]**

    bindparam() 的值可以是一个可调用对象，这样在语句执行时它会被评估以获取值。

+   **[sql]**

    添加了结果集获取的异常包装/重连支持。重新连接适用于在结果集期间引发可捕获数据错误的数据库（例如 MySQL 上不起作用）

    参考：[#978](https://www.sqlalchemy.org/trac/ticket/978)

+   **[sql]**

    实现了“线程本地”引擎的两阶段 API，通过 engine.begin_twophase()，engine.prepare()

    参考：[#936](https://www.sqlalchemy.org/trac/ticket/936)

+   **[sql]**

    修复了一个阻止 UNION 可克隆的错误。

    参考：[#986](https://www.sqlalchemy.org/trac/ticket/986)

+   **[sql]**

    在 insert()、update()、delete() 和 DDL() 中添加了“bind”关键字参数。这些语句上现在也可以分配 .bind 属性。

+   **[sql]**

    现在可以在 INSERT 和 INTO 之间编译带有额外“前缀”单词的插入语句，用于供应商扩展，如 MySQL 的 INSERT IGNORE INTO table。

### 扩展

+   **[extensions]**

    添加了一个新的超小型“声明式”扩展，它允许在类声明下方内联地进行 Table 和 mapper() 配置。该扩展与 ActiveMapper 和 Elixir 不同之处在于它完全不重新定义任何 SQLAlchemy 语义；字面 Column、Table 和 relation() 构造用于定义类行为和表定义。

### 杂项

+   **[dialects]**

    无效的 SQLite 连接 URL 现在会引发错误。

+   **[dialects]**

    postgres TIMESTAMP 现在可以正确呈现

    参考：[#981](https://www.sqlalchemy.org/trac/ticket/981)

+   **[dialects]**

    postgres 的 PGArray 默认是一个“可变”类型；当与 ORM 一起使用时，使用可变风格的相等性/写时复制技术来检测更改。

### orm

+   **[orm]**

    any()、has()、contains()、~contains()、属性级别的 == 和 != 现在可以与自引用关系正常工作 - 存在子句在“远程”侧上别名以区分它与父表。这适用于单表自引用以及基于继承的自引用。

+   **[orm]**

    在与 NULL 比较一对一关系的 relation() 级别时，== 和 != 操作符的行为已修复。

    参考：[#985](https://www.sqlalchemy.org/trac/ticket/985)

+   **[orm]**

    修复了一个错误，该错误使 session.expire() 属性在由 select_table 映射的多态映射实例上加载时无法正常工作。

+   **[orm]**

    添加了 query.with_polymorphic() - 指定了从基类继承的类列表，这些类将被添加到查询的 FROM 子句中。允许在 filter() 条件中使用子类，并且可以急切地加载这些子类的属性。

+   **[orm]**

    你们的呼声已经被听到：使用 delete-orphan 从属性或集合中删除待处理项会从会话中删除该项；不会引发 FlushError。请注意，如果您显式 session.save() 待处理项，属性/集合的移除仍然会将其排除。

+   **[orm]**

    当在会话中调用 session.refresh()和 session.expire()时，如果调用的实例不在会话中持久化，则会引发错误。

+   **[orm]**

    修复了当使用相同的 Query 生成多个使用 join()的 Query 对象时可能出现的生成性 bug。

+   **[orm]**

    修复了在 0.4.3 版本中引入的一个 bug，即在使用默认的“select” polymorphic_fetch 时，加载一个已经持久化的实例，该实例映射到了联合表继承，会触发一个无用的来自其联合表的“secondary”加载。这是因为属性在第一次加载时被标记为过期，并且在之前的“secondary”加载中没有被取消标记。现在，属性在任何加载或提交操作成功后基于 __dict__ 中的存在而被取消过期。

+   **[orm]**

    废弃了 Query 方法 apply_sum()、apply_max()、apply_min()、apply_avg()。更好的方法正在到来……

+   **[orm]**

    relation()可以接受��个可调用对象作为其第一个参数，该可调用对象返回要关联的类。这是为了帮助声明性包在没有类的情况下定义关系。

+   **[orm]**

    添加了一个名为“of_type()”的新的“更高级”运算符：在 join()中使用，以及与 any()和 has()一起使用，用于限定将在过滤条件中使用的子类，例如：

    > query.filter(Company.employees.of_type(Engineer).
    > 
    > any(Engineer.name==’foo’))
    > 
    > 或
    > 
    > query.join(Company.employees.of_type(Engineer)).
    > 
    > filter(Engineer.name==’foo’)

+   **[orm]**

    针对 flush()中潜在的丢失引用 bug 进行了预防性代码编写。

+   **[orm]**

    在 filter()、filter_by()等中使用的表达式，当它们使用从关系生成的子对象的标识（例如，filter(Parent.child==<somechild>)）生成的子句时，在执行时评估<somechild>的实际主键值，以便 Query 的自动刷新步骤可以完成，从而在<somechild>处于挂起状态时填充<somechild>的 PK 值。

+   **[orm]**

    将关系()-级别的 order by 设置为“secondary”表中的列现在可以与急加载一起使用，以前，“order by”没有针对 secondary 表的别名进行别名处理。

+   **[orm]**

    现在，对现有描述符进行的同义词代理是对这些描述符的完全代理。

### sql

+   **[sql]**

    可以再次对文本 FROM 子句的选择创建别名。

    参考：[#975](https://www.sqlalchemy.org/trac/ticket/975)

+   **[sql]**

    bindparam()的值可以是可调用的，这样在语句执行时会评估它以获取值。

+   **[sql]**

    为结果集获取添加了异常包装/重新连接支持。重新连接适用于在结果集期间引发可捕获数据错误的数据库（即在 MySQL 上不起作用）。

    参考：[#978](https://www.sqlalchemy.org/trac/ticket/978)

+   **[sql]**

    实现了“threadlocal”引擎的两阶段 API，通过 engine.begin_twophase()、engine.prepare()

    参考：[#936](https://www.sqlalchemy.org/trac/ticket/936)

+   **[sql]**

    修复了阻止 UNIONS 可克隆的 bug。

    参考：[#986](https://www.sqlalchemy.org/trac/ticket/986)

+   **[sql]**

    在 insert()、update()、delete()和 DDL()中添加了“bind”关键字参数。现在这些语句上的.bind 属性也是可分配的，就像在 select()上一样。

+   **[sql]**

    现在可以在 INSERT 和 INTO 之间编译带有额外“前缀”单词的插入语句，用于供应商扩展，如 MySQL 的 INSERT IGNORE INTO table。

### 扩展

+   **[扩展]**

    新增了一个超小型的“声明式”扩展，允许在类声明下方内联进行 Table 和 mapper()配置。该扩展与 ActiveMapper 和 Elixir 不同，它根本不重新定义任何 SQLAlchemy 语义；直接使用字面 Column、Table 和 relation()构造来定义类行为和表定义。

### 杂项

+   **[方言]**

    无效的 SQLite 连接 URL 现在会引发错误。

+   **[方言]**

    postgres TIMESTAMP 现在正确呈现

    参考：[#981](https://www.sqlalchemy.org/trac/ticket/981)

+   **[方言]**

    postgres PGArray 默认是“可变”类���；在与 ORM 一起使用时，使用可变风格的相等性/写时复制技术来测试更改。

## 0.4.3

发布日期：2008 年 2 月 14 日 星期四

### 通用

+   **[通用]**

    修复了一系列隐藏的以及一些不那么隐藏的 Python 2.3 兼容性问题，感谢对在 2.3 上运行完整测试套件的新支持。

+   **[通用]**

    现在警告被作为类型异常 SAWarning 发出。

### orm

+   **[orm]**

    每个 Session.begin()现在必须伴随相应的 commit()或 rollback()，除非会话已通过 Session.close()关闭。这也包括使用 transactional=True 创建的会话隐含的 begin()。这里引入的最大变化是，当使用 transactional=True 创建的会话在 flush()期间引发异常时，必须调用 Session.rollback()或 Session.close()才能使该会话在异常后继续。

+   **[orm]**

    修复了在合并具有 backref 集合的瞬时实体时出现的 merge()集合翻倍错误。

    参考：[#961](https://www.sqlalchemy.org/trac/ticket/961)

+   **[orm]**

    merge(dont_load=True)不接受瞬时实体，这与 merge(dont_load=True)不接受任何“脏”对象的事实一致。

+   **[orm]**

    添加了由 scoped_session 生成的独立“query”类属性。这提供了 MyClass.query，而不使用 Session.mapper。通过以下方式使用：

    > MyClass.query = Session.query_property()

+   **[orm]**

    当尝试在没有会话的情况下访问过期实例属性时，会引发适当的错误消息。

+   **[orm]**

    dynamic_loader() / lazy=”dynamic”现在以与 relation()相同的方式接受和使用 order_by 参数。

+   **[orm]**

    添加了 expire_all()方法到 Session。调用 expire()以使所有持久实例过期。这在与…结合使用时非常方便。

+   **[orm]**

    部分或完全过期的实例在常规 Query 操作期间将其过期属性填充，影响那些对象，防止每个实例需要不必要的第二个 SQL 语句。

+   **[orm]**

    动态关系在被引用时会创建对父对象的强引用，以便查询仍然有一个父对象可以调用，即使父对象仅在单个表达式范围内创建（并且在其他情况下取消引用）。

    参考：[#938](https://www.sqlalchemy.org/trac/ticket/938)

+   **[orm]**

    添加了一个 mapper()标志“eager_defaults”。当设置为 True 时，在 INSERT 或 UPDATE 操作期间生成的默认值会立即后获取，而不是延迟到以后。这模仿了旧的 0.3 行为。

+   **[orm]**

    query.join()现在可以接受类映射属性作为参数。这些可以用于替代字符串或任何组合。特别是这允许在多态关系上构建到子类的连接，即：

    > query(Company).join([‘employees’, Engineer.name])

+   **[orm] [(‘employees’] [Engineer.name] [people.join(engineer))]**

    query.join()还可以接受属性名/某些可选择的元组作为参数。这允许构建从多态关系的子类连接，即：

    > query(Company).join(
    > 
    > )

+   **[orm]**

    对 join()的行为进行了一般性改进，与多态映射器一起使用时，即从/到多态映射器进行连接并正确应用别名。

+   **[orm]**

    当映射器确定映射连接的自然“主键”时，它将更有效地减少通过外键关系等效的列。这影响了需要发送给 query.get()的参数数量，等等。

    参考：[#933](https://www.sqlalchemy.org/trac/ticket/933)

+   **[orm]**

    惰性加载器现在可以处理一个连接条件，其中“绑定”列（即将父 id 作为绑定参数发送的列）在连接条件中出现多次。具体来说，这允许了包含父相关子查询的 relation()的常见任务，比如“仅选择最近的子项”。

    参考：[#946](https://www.sqlalchemy.org/trac/ticket/946)

+   **[orm]**

    修复了多态继承中的错误，当基本多态 _on 列与继承映射器的本地可选择中的任何列不对应时，会引发不正确的异常，超过一级深度

+   **[orm]**

    修复了多态继承中的错误，使得在多态映射器上设置一个有效的“order_by”变得困难。

+   **[orm]**

    修复了 Query 中一个较昂贵的调用，减缓了多态查询的速度。

+   **[orm]**

    如果需要，”被动默认值”和其他“内联”默认值现在可以在 flush()调用期间加载；特别是，这允许构建 relations()，其中外键列引用服务器端生成的非主键列。

    参考：[#954](https://www.sqlalchemy.org/trac/ticket/954)

+   **[orm]**

    附加的会话事务修复/更改：

    +   修复了会话事务管理的错误：在向嵌套事务添加连接时，没有在连接上启动父事务。

    +   session.transaction 现在始终指代最内层的活动事务，即使在会话事务对象上直接调用 commit/rollback 也是如此。

    +   现在可以准备两阶段事务。

    +   当一个连接上的两阶段事务准备失败时，所有连接都会回滚。

    +   当使用嵌套事务时，session.close()没有关闭所有事务。

    +   rollback()之前错误地将当前事务直接设置为可回滚到的事务的父事务。现在它回滚到可以处理它的下一个事务，但将当前事务设置为其父事务，并使之间的事务无效。无效的事务只能回滚或关闭，任何其他调用都会导致错误。

    +   对于 commit()，autoflush 没有对简单的子事务进行刷新。

    +   unitofwork flush 在会话不在事务中且提交事务失败时没有关闭失败的事务。

+   **[orm]**

    杂项票据：

    引用：[#940](https://www.sqlalchemy.org/trac/ticket/940)，[#964](https://www.sqlalchemy.org/trac/ticket/964)

### sql

+   **[sql]**

    添加了“schema.DDL”，一条可执行的自由形式 DDL 语句。DDL 可以单独执行，也可以附加到 Table 或 MetaData 实例上，在创建和/或删除这些对象时自动执行。

+   **[sql]**

    可以使用‘useexisting=True’标志覆盖表的列和约束，该表已经存在（例如已经反映的表），现在还考虑了与其一起传递的参数。

+   **[sql]**

    添加了基于可调用的 DDL 事件接口，在 Tables 和 MetaData 创建和删除之前和之后添加钩子。

+   **[sql]**

    在 delete()和 update()构造中添加了 generative where(<criterion>)方法，该方法返回一个通过 AND 连接到现有条件的新对象，就像 select().where()一样。

+   **[sql]**

    向列操作添加了“ilike()”操作符。在 postgres 上编译为 ILIKE，在其他所有数据库上为 lower(x) LIKE lower(y)。  

    引用：[#727](https://www.sqlalchemy.org/trac/ticket/727)

+   **[sql]**

    添加了“now()”作为通用函数；在 SQLite、Oracle 和 MSSQL 上编译为“CURRENT_TIMESTAMP”；在其他所有数据库上是“now()”。

    引用：[#943](https://www.sqlalchemy.org/trac/ticket/943)

+   **[sql]**

    startswith()、endswith()和 contains()操作符现在在 SQL 中将通配符操作符与给定操作数连接起来，即在所有情况下都是“’%’ || <bindparam>”，适当地接受 text(‘something’)操作数

    引用：[#962](https://www.sqlalchemy.org/trac/ticket/962)

+   **[sql]**

    cast()适当地接受 text(‘something’)和其他非文字操作数

    引用：[#962](https://www.sqlalchemy.org/trac/ticket/962)

+   **[sql]**

    修复了结果代理中匿名生成的列标签无法使用其直接字符串名称访问的错误

+   **[sql]**

    现在可以定义可延迟的约束。

+   **[sql]**

    在 select()和 text()中添加了“autocommit=True”关键字参数，以及 select()上的生成 autocommit()方法；对于通过某些用户定义的方式修改数据库的语句，而不是通常的 INSERT/UPDATE/DELETE 等。如果没有进行事务，则此标志将在执行期间启用“autocommit”行为。

    参考：[#915](https://www.sqlalchemy.org/trac/ticket/915)

+   **[sql]**

    可选择上的“.c.”属性现在为其列子句中的每个列表达式添加一个条目。以前，“未命名”列（如函数和 CASE 语句）没有放在那里。现在，如果没有“名称”可用，它们将使用其完整字符串表示���

+   **[sql]**

    CompositeSelect，即任何 union()，union_all()，intersect()等现在断言每个可选择包含相同数量的列。这符合相应的 SQL 要求。

+   **[sql]**

    为否则未标记的函数和表达式生成的匿名“标签”现在在编译时向外传播，例如 select([select([func.foo()])])的表达式。

+   **[sql]**

    基于上述思想，CompositeSelects 现在根据第一个可选择中存在的名称构建其“.c.”集合；corresponding_column()现在对所有嵌入式可选择完全有效。

+   **[sql]**

    Oracle 和其他数据库现在正确编码用于默认值（如序列等）的 SQL，即使没有使用 unicode 标识符，因为标识符准备器可能返回缓存的 unicode 标识符。

+   **[sql]**

    现在左侧表达式中的列和子句与 datetime 对象的比较现在可以工作（d < table.c.col）。 （RHS 上的 datetimes 一直有效，LHS 的异常是 datetime 实现的怪癖。）

### 杂项

+   **[dialects]**

    在 SQLite 中对模式（通过 ATTACH DATABASE … AS name 链接）的支持更好。在过去的一些情况下，SQLite 生成的 SQL 中省略了模式名称。现在不再这样。

+   **[dialects]**

    SQLite 上的 table_names 现在也会选择临时表。

+   **[dialects]**

    在反射操作期间自动检测未指定的 MySQL ANSI_QUOTES 模式，支持在中途更改模式。如果没有使用反射，则仍然需要手动设置模式。

+   **[dialects]**

    修复了 SQLite 上 TIME 列的反射。

+   **[dialects]**

    最终在 postgres 中添加了 PGMacAddr 类型

    参考：[#580](https://www.sqlalchemy.org/trac/ticket/580)

+   **[dialects]**

    在 Firebird 下反映与主键字段关联的序列（通常具有 BEFORE INSERT 触发器）。

+   **[dialects]**

    当生成 LIMIT/OFFSET 子查询时，Oracle 会组装结果集列映射中的正确列，即使长名称截断启动也允许列正确映射到结果集

    参考：[#941](https://www.sqlalchemy.org/trac/ticket/941)

+   **[dialects]**

    MSSQL 现在在 _is_select 正则表达式中包含 EXEC，这应该允许使用返回行的存储过程。

+   **[方言]**

    MSSQL 现在包含了使用 ANSI SQL row_number() 函数的 LIMIT/OFFSET 的实验性实现，因此需要 MSSQL-2005 或更高版本。要启用此功能，请在 connect 的关键字参数中添加 “has_window_funcs”，或在 dburi 查询参数中添加 “?has_window_funcs=1”。

+   **[扩展]**

    更改了 ext.activemapper 以使用非事务性会话进行对象存储。

+   **[扩展]**

    修复了关联代理列表上的 “[‘a’] + obj.proxied” 二进制操作的输出顺序。

### 通用

+   **[通用]**

    修复了一系列隐藏的以及一些不那么隐藏的 Python 2.3 兼容性问题，感谢对在 2.3 上运行完整测试套件的新支持。

+   **[通用]**

    现在警告被作为异常类型 SAWarning 发出。

### ORM

+   **[ORM]**

    每个 Session.begin() 现在必须伴随着相应的 commit() 或 rollback()，除非会话被 Session.close() 关闭。这也包括了对于使用 transactional=True 创建的会话隐含的 begin()。这里引入的最大变化是，当使用 transactional=True 创建的会话在 flush() 过程中引发异常时，你必须调用 Session.rollback() 或 Session.close() 以便让该会话在异常后继续运行。

+   **[ORM]**

    修复了在将瞬时实体与带有 backref 的集合合并时出现的 merge() 集合翻倍 bug。

    参考：[#961](https://www.sqlalchemy.org/trac/ticket/961)

+   **[ORM]**

    merge(dont_load=True) 不接受瞬时实体，这是为了与 merge(dont_load=True) 不接受任何“脏”对象的事实保持一致。

+   **[ORM]**

    添加了由 scoped_session 生成的独立的 “query” 类属性。这提供了 MyClass.query 而不使用 Session.mapper。使用方式：

    > MyClass.query = Session.query_property()

+   **[ORM]**

    当尝试在没有会话的情况下访问过期实例属性时，会引发适当的错误消息。

+   **[ORM]**

    dynamic_loader() / lazy=”dynamic” 现在接受并使用 order_by 参数，方式与 relation() 中的工作方式相同。

+   **[ORM]**

    添加了 Session 的 expire_all() 方法。对所有持久实例调用 expire()。这在与…

+   **[ORM]**

    部分或完全过期的实例在常规查询操作中将填充其过期属性，这样可以防止每个实例需要多余的第二个 SQL 语句。

+   **[ORM]**

    动态关系在被引用时会创建对父对象的强引用，以便查询仍然有一个父对象可以调用，即使父对象仅在单个表达式范围内创建（并且在其他情况下被取消引用）。

    参考：[#938](https://www.sqlalchemy.org/trac/ticket/938)

+   **[ORM]**

    添加了一个 mapper()标志“eager_defaults”。当设置为 True 时，在 INSERT 或 UPDATE 操作期间生成的默认值将立即后获取，而不是延迟到以后。这模仿了旧的 0.3 行为。

+   **[orm]**

    query.join()现在可以接受类映射属性作为参数。这些可以用于替代或与字符串任意组合。特别是这允许在多态关系上构建到子类的连接，即：

    > query(Company).join([‘employees’, Engineer.name])

+   **[orm] [(‘employees’] [Engineer.name] [people.join(engineer))]**

    query.join()还可以接受属性名称/某些可选择的元组作为参数。这允许构建从多态关系的子类连接，即：

    > query(Company).join(
    > 
    > )

+   **[orm]**

    对与多态映射器一起使用 join()的行为进行了一般改进，即从/到多态映射器进行连接并正确应用别名。

+   **[orm]**

    当映射器确定映射连接的自然“主键”时，它将更有效地减少通过外键关系等效的列。这影响了需要发送给 query.get()的参数数量，等等。

    参考：[#933](https://www.sqlalchemy.org/trac/ticket/933)

+   **[orm]**

    惰性加载器现在可以处理连接条件中“绑定”列（即作为绑定参数发送父 id 的列）出现多次的情况。具体来说，这允许了包含父相关子查询的 relation()的常见任务，比如“仅选择最近的子项”。

    参考：[#946](https://www.sqlalchemy.org/trac/ticket/946)

+   **[orm]**

    修复了多态继承中的错误，当基多态 _on 列与继承映射器的本地可选择中的任何列不对应时，会引发不正确的异常，超过一级深度

+   **[orm]**

    修复了多态继承中的错误，使得在多态映射器上设置有效的“order_by”变得困难。

+   **[orm]**

    修复了 Query 中一个相当昂贵的调用，导致多态查询变慢。

+   **[orm]**

    如果需要，在 flush()调用期间现在可以加载“被动默认值”和其他“内联”默认值；特别是，这允许构建 relations()，其中外键列引用服务器生成的非主键列。

    参考：[#954](https://www.sqlalchemy.org/trac/ticket/954)

+   **[orm]**

    会话事务的其他修复/更改：

    +   修复了会话事务管理中的错误：在将连接添加到嵌套事务时，父事务未在连接上启动。

    +   session.transaction 现在始终指向最内部的活动事务，即使在会话事务对象上直接调用 commit/rollback。

    +   现在可以准备两阶段事务。

    +   当在一个连接上准备两阶段事务失败时，所有连接都将回滚。

    +   当使用嵌套事务时，session.close()不会关闭所有事务。

    +   rollback()先前错误地将当前事务直接设置为可以回滚到的事务的父事务。现在它将回滚到可以处理的下一个事务，但将当前事务设置为其父事务，并使中间的事务无效。无效的事务只能回滚或关闭，任何其他调用都会导致错误。

    +   对于简单子事务，commit()的 autoflush 不会刷新。

    +   当会话不在事务中且提交事务失败时，unitofwork flush 不会关闭失败的事务。

+   **[orm]**

    其他票据：

    参考：[#940](https://www.sqlalchemy.org/trac/ticket/940), [#964](https://www.sqlalchemy.org/trac/ticket/964)

### sql

+   **[sql]**

    添加了“schema.DDL”，一个可执行的自由形式 DDL 语句。DDL 可以独立执行，也可以附加到表或 MetaData 实例上，并在创建和/或删除这些对象时自动执行。

+   **[sql]**

    可以使用“useexisting=True”标志在现有表上覆盖表列和约束（例如已经反射的表），该标志现在考虑传递的参数。

+   **[sql]**

    添加了基于可调用函数的 DDL 事件接口，可以在表和 MetaData 创建和删除之前和之后添加钩子。

+   **[sql]**

    在 delete()和 update()构造中添加了生成 where(<criterion>)方法，该方法返回一个通过 AND 连接到现有条件的新对象，就像 select().where()一样。

+   **[sql]**

    在列操作中添加了“ilike()”运算符。在 postgres 上编译为 ILIKE，在其他所有数据库上编译为 lower(x) LIKE lower(y)。

    参考：[#727](https://www.sqlalchemy.org/trac/ticket/727)

+   **[sql]**

    添加了“now()”作为通用函数；在 SQLite、Oracle 和 MSSQL 上编译为“CURRENT_TIMESTAMP”；在其他所有数据库上为“now()”。

    参考：[#943](https://www.sqlalchemy.org/trac/ticket/943)

+   **[sql]**

    startswith()、endswith()和 contains()运算符现在在 SQL 中将通配符运算符与给定操作数连接，即在所有情况下，“'%' || <bindparam>”，正确接受 text('something')操作数

    参考：[#962](https://www.sqlalchemy.org/trac/ticket/962)

+   **[sql]**

    cast()正确接受 text('something')和其他非文字操作数

    参考：[#962](https://www.sqlalchemy.org/trac/ticket/962)

+   **[sql]**

    修复了结果代理中的错误，匿名生成的列标签将无法使用其直接字符串名称访问。

+   **[sql]**

    现在可以定义可延迟的约束。

+   **[sql]**

    在 select()和 text()中添加了“autocommit=True”关键字参数，以及 select()上的生成 autocommit()方法；对于通过某些用户定义的方式修改数据库的语句，而不是通常的 INSERT/UPDATE/DELETE 等。如果没有进行事务，则此标志将在执行期间启用“autocommit”行为。

    参考：[#915](https://www.sqlalchemy.org/trac/ticket/915)

+   **[sql]**

    现在，可选择项的“.c.” 属性将为其列子句中的每个列表达式添加一个条目。 以前，“未命名”列（如函数和 CASE 语句）未放置在那里。 现在，如果没有“名称”可用，它们将使用其完整的字符串表示形式。。

+   **[SQL]**

    一个 CompositeSelect，即任何 union()、union_all()、intersect() 等现在都断言每个可选择项包含相同数量的列。 这符合相应的 SQL 要求。

+   **[SQL]**

    为否则未标记的函数和表达式生成的匿名“标签”现在在编译时传播出去，例如 select([select([func.foo()])]) 中的表达式。

+   **[SQL]**

    在上述思想的基础上，CompositeSelects 现在根据第一个可选择项中存在的名称构建其“.c.” 集合；corresponding_column() 现在对所有嵌套可选择项都完全有效。

+   **[SQL]**

    Oracle 和其他数据库在生成诸如序列等默认值的 SQL 时会正确编码，即使没有使用 unicode idents 也是如此，因为标识符准备程序可能返回缓存的 unicode 标识符。

+   **[SQL]**

    对表达式左侧的 datetime 对象进行列和子句比较现在可以工作了（d < table.c.col）。 （RHS 上的 datetime 一直都有效，LHS 上的异常是 datetime 实现的一个怪癖。）

### 杂项

+   **[方言]**

    更好地支持 SQLite 中的模式（通过 ATTACH DATABASE … AS name 链接）。 在过去的某些情况下，SQLite 生成的 SQL 中省略了模式名称。 这不再是这样。

+   **[方言]**

    SQLite 上的 table_names 现在也会拾取临时表。

+   **[方言]**

    在反射操作期间自动检测未指定的 MySQL ANSI_QUOTES 模式，支持在中途更改模式。 如果不使用反射，则仍然需要手动设置模式。

+   **[方言]**

    修复了 SQLite 上 TIME 列的反射。

+   **[方言]**

    最终在 postgres 中添加了 PGMacAddr 类型

    参考：[#580](https://www.sqlalchemy.org/trac/ticket/580)

+   **[方言]**

    在 Firebird 下反射与 PK 字段关联的序列（通常使用 BEFORE INSERT 触发器）。

+   **[方言]**

    在生成 LIMIT/OFFSET 子查询时，Oracle 会将正确的列组装到结果集列映射中，即使长名称截断发生，也允许列正确映射到结果集

    参考：[#941](https://www.sqlalchemy.org/trac/ticket/941)

+   **[方言]**

    MSSQL 现在在 _is_select 正则表达式中包含 EXEC，这应该允许使用返回行的存储过程。

+   **[方言]**

    MSSQL 现在包括对 ANSI SQL row_number() 函数的实验性 LIMIT/OFFSET 实现，因此需要 MSSQL-2005 或更高版本。要启用此功能，请在连接的关键字参数中添加“has_window_funcs”，或在 dburi 查询参数中添加“?has_window_funcs=1”。

+   **[ext]**

    更改 ext.activemapper 以使用非事务性会话来处理对象存储。

+   **[ext]**

    修复了关联代理列表上“[‘a’] + obj.proxied” 二元操作的输出顺序。

## 0.4.2p3

发布日期：2008 年 1 月 9 日

### 通用

+   **[通用]**

    子版本编号方案已更改以符合 setuptools 版本号规则；现在应该通过 easy_install -u 获取此版本，而不是 0.4.2 以上。

### orm

+   **[对象关系映射]**

    修复了使用“可变标量”（如 PickleTypes）时 session.dirty 的 bug

+   **[对象关系映射]**

    在关系()上刷新时，当其主要或次要连接条件中存在非本地映射列时，添加了更具描述性的错误消息

+   **[对象关系映射]**

    现在在 InstanceState.__cleanup()中抑制*所有*错误。

+   **[对象关系映射]**

    修复了属性历史记录中的一个 bug，即将新集合分配给已经具有待处理更改的基于集合的属性会生成不正确的历史记录

    参考：[#922](https://www.sqlalchemy.org/trac/ticket/922)

+   **[对象关系映射]**

    修复了删除孤立级联 bug，即将同一对象两次设置为标量属性可能会记录为孤立对象

    参考：[#925](https://www.sqlalchemy.org/trac/ticket/925)

+   **[对象关系映射]**

    修复了对基于列表的关系进行+=赋值时的级联。

+   **[对象关系映射]**

    现在可以针对尚不存在的属性创建同义词，稍后通过 add_property()添加。这通常包括反向引用。（即，您可以为反向引用创建同义词，而不必担心操作顺序）

    参考：[#919](https://www.sqlalchemy.org/trac/ticket/919)

+   **[对象关系映射]**

    修复了多态“union”映射器出现 bug 的情况，该映射器回退到继承表的“延迟”加载

+   **[对象关系映射]**

    映射器/映射类（即‘c’）上的“columns”集合针对映射表，而不是在多态“union”加载的情况下的 select_table（这不应该是可察觉的）。

+   **[对象关系映射]**

    修复了一个相当关键的 bug，即相同实例可能会在 unitofwork.new 集合中列出多次；最常在使用组合继承映射器和 ScopedSession.mapper 时重现，因为每个实例的多个 __init__ 调用可能会使用不同的 _state 对象保存()具有不同 _state 对象的对象

+   **[对象关系映射]**

    在 Query 中添加了非常基本的迭代器行为。调用 query.yield_per(<行数>)并在迭代上下文中评估 Query；每个 N 行的集合将被打包并产生。请谨慎使用此方法，因为它不会尝试在结果批次边界上协调急切加载的集合，也不会在同一批次中出现相同实例时表现良好。这意味着如果在多个批次中引用急切加载的集合，它将被清除，并且在所有情况下，如果在多个批次中出现相同实例，属性将被覆盖。

+   **[对象关系映射]**

    为集合和关联代理集合添加了原地设置变异操作符。

    参考：[#920](https://www.sqlalchemy.org/trac/ticket/920)

### sql

+   **[结构化查询语言]**

    现在 Text 类型已正确导出，不会在 DDL 创建时引发警告；没有长度的 String 类型仅在 CREATE TABLE 期间引发警告

    参考：[#912](https://www.sqlalchemy.org/trac/ticket/912)

+   **[结构化查询语言]**

    添加了新的 UnicodeText 类型，用于指定编码的、无长度的 Text 类型。

+   **[sql]**

    修复了 `union()` 中的错误，以便可以将不派生自 `FromClause` 对象的 `select()` 语句进行联合。

+   **[sql]**

    将 TEXT 名称更改为 Text，因为它是一种“通用”类型；TEXT 名称已弃用，直到 0.5 版本。当用于 CREATE TABLE 语句时，没有长度的 String 转为 Text 的“升级”行为也已弃用，将在使用时发出警告（对于 SQL 表达式目的而言，没有长度的 String 仍然可以正常使用）。

    参考：[#912](https://www.sqlalchemy.org/trac/ticket/912)

+   **[sql]**

    生成式 `select.order_by(None)` / `group_by(None)` 未能重置排序/分组准则，已修复。

    参考：[#924](https://www.sqlalchemy.org/trac/ticket/924)

### 杂项

+   **[方言]**

    修复了反射 mysql 空字符串列默认值的问题。

+   **[扩展]**

    对于关联代理列表的 `+`、`*`、`+=` 和 `*=` 支持。

+   **[方言]**

    修复了 `mssql` 中 `MSDate`/`MSDateTime` 子类中“日期”/“日期时间”测试的范围，以便传入的“日期时间”对象不会被误解为“日期”对象，反之亦然。

    参考：[#923](https://www.sqlalchemy.org/trac/ticket/923)

+   **[方言]**

    修复了 PGArray 类型的子类型结果处理器缺失的问题。

    参考：[#913](https://www.sqlalchemy.org/trac/ticket/913)

### 一般

+   **[一般]**

    子版本编号方案更改以适应 setuptools 版本号规则；现在 easy_install -u 应该能获取此版本超过 0.4.2。

### orm

+   **[orm]**

    当使用“可变标量”（例如 PickleTypes）时，修复了 session.dirty 的错误。

+   **[orm]**

    当在具有主要或次要连接条件中具有非本地映射列的关系()上刷新时，添加了更具描述性的错误消息。

+   **[orm]**

    现在在 `InstanceState.__cleanup()` 中抑制 *所有* 错误。

+   **[orm]**

    修复了属性历史记录中的错误，即将新集合分配给已经具有挂起更改的基于集合的属性将生成不正确的历史记录。

    参考：[#922](https://www.sqlalchemy.org/trac/ticket/922)

+   **[orm]**

    修复了删除孤立级联错误，即将相同对象两次设置为标量属性可能会将其记录为孤立对象。

    参考：[#925](https://www.sqlalchemy.org/trac/ticket/925)

+   **[orm]**

    修复了对列表型关系执行 `+=` 赋值时的级联问题。

+   **[orm]**

    现在可以针对尚不存在的属性创建同义词，后续通过 `add_property()` 添加。这通常包括反向引用。（即您可以为反向引用创建同义词，而不必担心操作顺序）

    参考：[#919](https://www.sqlalchemy.org/trac/ticket/919)

+   **[orm]**

    修复了多态“union”映射器可能出现的错误，该映射器退回到继承表的“延迟”加载。

+   **[orm]**

    在映射器/映射类（即 'c'）上的“columns”集合针对映射表，而不是针对多态“union”加载中的 select_table（这不应该被注意到）。

+   **[orm]**

    修复了一个相当关键的错误，即相同的实例可能会在 unitofwork.new 集合中列出多次；最常见的情况是在使用继承映射器和 ScopedSession.mapper 的组合时，因为每个实例的多个 __init__ 调用可能会使用不同的 _state 对象保存()对象。

+   **[ORM]**

    向 Query 添加了非常基本的迭代器行为。调用 query.yield_per(<行数>)并在迭代上下文中评估 Query；每个 N 行的集合将被打包并产生。请谨慎使用此方法，因为它不会尝试在结果批次边界上协调急切加载的集合，也不会在同一批次中同一实例出现多次时表现良好。这意味着如果在多个批次中引用了急切加载的集合，它将被清除，并且在所有情况下，如果实例出现在多个批次中，属性将被覆盖。

+   **[ORM]**

    修复了集合和关联代理集合的原地设置变异运算符。

    参考：[#920](https://www.sqlalchemy.org/trac/ticket/920)

### sql

+   **[sql]**

    现在正确导出文本类型，不会在 DDL 创建时引发警告；没有长度的字符串类型只会在 CREATE TABLE 时引发警告。

    参考：[#912](https://www.sqlalchemy.org/trac/ticket/912)

+   **[sql]**

    添加了新的 UnicodeText 类型，用于指定编码的、无长度的文本类型。

+   **[sql]**

    修复了 union()中的错误，以便可以将不派生自 FromClause 对象的 select()语句联合。

+   **[sql]**

    将 TEXT 的名称更改为 Text，因为它是一个“通用”类型；TEXT 名称在 0.5 版本之前已被弃用。当没有长度时，String 类型升级为 Text 的行为也在 0.5 版本之前已被弃用；在用于 CREATE TABLE 语句时会发出警告（String 没有长度用于 SQL 表达式目的仍然有效）。

    参考：[#912](https://www.sqlalchemy.org/trac/ticket/912)

+   **[sql]**

    生成式 select.order_by(None) / group_by(None)未能重置 order by/group by 条件，已修复。

    参考：[#924](https://www.sqlalchemy.org/trac/ticket/924)

### 杂项

+   **[方言]**

    修复了反射 mysql 空字符串列默认值的问题。

+   **[扩展]**

    对关联代理列表支持‘+’、‘*’、‘+=’和‘*=’。

+   **[方言]**

    mssql - 缩小了对 MSDate/MSDateTime 子类中“date”/“datetime”的测试范围，以防止传入的“datetime”对象被错误解释为“date”对象或反之。

    参考：[#923](https://www.sqlalchemy.org/trac/ticket/923)

+   **[方言]**

    修复了 PGArray 类型的子类型结果处理器的缺失调用。

    参考：[#913](https://www.sqlalchemy.org/trac/ticket/913)

## 0.4.2

发布日期：Wed Jan 02 2008

### ORM

+   **[ORM]**

    针对基于集合的反向引用的重大行为更改：它们不再触发延迟加载！“reverse”添加和删除被排队并在实际从中读取和加载集合时与集合合并；但不会事先触发加载。对于注意到这种行为的用户，在某些情况下，这应该比在某些情况下使用动态关系更方便；对于那些没有注意到的用户，您可能会注意到您的应用程序在某些情况下使用的查询比以前少得多。

    参考：[#871](https://www.sqlalchemy.org/trac/ticket/871)

+   **[orm]**

    可变主键支持已添加。主键列可以自由更改，并且在刷新时实例的标识将发生变化。此外，支持沿关系更新外键引用（主键或非主键），可以与数据库的 ON UPDATE CASCADE（对于像 Postgres 这样的数据库是必需的）一起使用，或者通过设置标志“passive_cascades=False”直接由 ORM 以 UPDATE 语句的形式发出。

+   **[orm]**

    继承映射器现在直接继承其父映射器的 MapperExtensions，以便为子类调用特定 MapperExtension 的所有方法。与往常一样，任何 MapperExtension 都可以返回 EXT_CONTINUE 以继续扩展处理或 EXT_STOP 以停止处理。映射器解析的顺序是：<在类的映射器上声明的扩展> <在类的父映射器上声明的扩展> <全局声明的扩展>。

    请注意，如果您单独实例化相同的扩展类，然后分别将其应用于同一继承链中的两个映射器，该扩展将应用于继承类两次，并且每个方法将被调用两次。

    要将映射扩展明确应用于每个继承类，但每个方法每次操作只调用一次，请为两个映射器使用相同的扩展实例。

    参考：[#490](https://www.sqlalchemy.org/trac/ticket/490)

+   **[orm]**

    现在对 MapperExtension.before_update()和 after_update()进行对称调用；以前，一个没有修改列属性（但有关系()修改）的实例可能会调用 before_update()但不会调用 after_update()

    参考：[#907](https://www.sqlalchemy.org/trac/ticket/907)

+   **[orm]**

    查询语句中缺少的列现在在加载时会自动延迟加载。

+   **[orm]**

    扩展“object”并且在实例构造时没有提供 __init__()方法的映射类现在将在实例构造时引发 TypeError，如果非空*args 或**kwargs 在实例构造时存在（并且未被任何扩展（如 scoped_session mapper）消耗），与普通 Python 类的行为一致

    参考：[#908](https://www.sqlalchemy.org/trac/ticket/908)

+   **[orm]**

    修复了当 filter_by()将关系与 None 进行比较时的查询错误

    参考：[#899](https://www.sqlalchemy.org/trac/ticket/899)

+   **[orm]**

    改进了对映射实体的 pickling 支持。现在每个实例的 lazy/deferred/expired 可调用对象都是可序列化的，因此它们与 _state 一起序列化和反序列化。

+   **[orm]**

    新的 synonym() 行为：如果类上不存在属性，则会在映射类上放置一个属性。如果类上已经存在属性，则 synonym 将使用适当的比较运算符修饰该属性，以便可以像任何其他映射属性一样在列表达式中使用（即可用于 filter() 等）。“proxy=True”标志已被弃用并且不再起作用。“map_column=True”标志将自动生成与同义词名称对应的 ColumnProperty，例如：‘somename’:synonym(‘_somename’, map_column=True) 将列名为 ‘somename’ 的列映射到属性 ‘_somename’。请参阅映射器文档中的示例。

    参考：[#801](https://www.sqlalchemy.org/trac/ticket/801)

+   **[orm]**

    Query.select_from() 现在将给定参数替换所有现有的 FROM 条件；以前的行为构造一个 FROM 子句列表通常是没有用的，因为需要 filter() 调用来创建连接条件，并且在 filter() 中引入的新表已经将自己添加到 FROM 子句中。新行为不仅允许从主表进行连接，还允许选择语句。过滤条件、排序条件、急加载条件将与给定语句“别名”对齐。

+   **[orm]**

    本月的属性检测重构改变了自 0.3 中期以来我们一直拥有的“加载时复制”行为，大多数情况下将其改为“修改时复制”。这减少了加载操作的大量延迟，并且总体上做更少的工作，因为只有实际上被修改的属性才会复制其“已提交状态”。只有“可变标量”属性（即 pickled 对象或其他可变项）保留了旧的行为。

+   **[orm] [attrname]**

    对属性进行 del 操作不会再次触发该属性的 lazyloader；“del”操作使属性的有效值为“None”。要重新触发属性的“loader”，请使用 session.expire(instance,)。

+   **[orm]**

    当将 many-to-one 属性与 None 进行比较时，query.filter(SomeClass.somechild == None) 将正确生成“id IS NULL”，包括 NULL 位于右侧的情况。

+   **[orm]**

    query.order_by() 考虑了别名连接，即 query.join('orders', aliased=True).order_by(Order.id)

+   **[orm]**

    eagerload()、lazyload()、eagerload_all() 接受可选的第二个类或映射器参数，该参数将选择要应用选项的映射器。这可以选择其他使用 add_entity() 添加的映射器之一。

+   **[orm]**

    eagerloading 会与通过 add_entity() 添加的映射器一起工作。

+   **[orm]**

    为“动态”关系添加了“级联删除”行为，就像常规关系一样。如果未设置 `passive_deletes` 标志（也刚刚添加），则删除父项将触发对子项的完整加载，以便可以相应地删除或更新它们。

+   **[orm]**

    也使用动态，实现了正确的 `count()` 行为以及其他辅助方法。

+   **[orm]**

    修复了多态关系上的级联，使得从对象到多态集合的级联继续沿着集合中每个元素特定的属性集进行级联。

+   **[orm]**

    `query.get()` 和 `query.load()` 不考虑现有的过滤器或其他条件；这些方法*始终*在数据库中查找给定的 id 或从标识映射中返回当前实例，而忽略已配置的任何现有过滤器、连接、group_by 或其他条件。

    参考：[#893](https://www.sqlalchemy.org/trac/ticket/893)

+   **[orm]**

    在继承映射器中添加了对 `version_id_col` 的支持。 `version_id_col` 通常在继承关系的基本映射器上设置，在这种关系中，它对所有继承的映射器都生效。

    参考：[#883](https://www.sqlalchemy.org/trac/ticket/883)

+   **[orm]**

    放宽了对 `column_property()` 表达式标签的规则；现在任何 `ColumnElement` 都被接受了，因为编译器现在会自动给没有标签的 `ColumnElements` 加上标签。一个可选择的，像 `select()` 语句一样，仍然需要通过 `as_scalar()` 或 `label()` 转换为 `ColumnElement`。

+   **[orm]**

    修复了反向引用错误，如果 `attr` 为 `None`，则无法删除 `instance.attr`。

+   **[orm]**

    删除或私有化了几个 ORM 属性：`mapper.get_attr_by_column()`、`mapper.set_attr_by_column()`、`mapper.pks_by_table`、`mapper.cascade_callable()`、`MapperProperty.cascade_callable()`、`mapper.canload()`、`mapper.save_obj()`、`mapper.delete_obj()`、`mapper._mapper_registry`、`attributes.AttributeManager`

+   **[orm]**

    将不兼容的集合类型分配给关系属性现在会引发 TypeError 而不是 sqlalchemy 的 ArgumentError。

+   **[orm]**

    如果传入字典中的键与集合的 `keyfunc` 为该值使用的键不匹配，则对 MappedCollection 进行的批量赋值将引发错误。

    参考：[#886](https://www.sqlalchemy.org/trac/ticket/886)

+   **[orm] [newval1] [newval2]**

    现在自定义集合可以指定一个 `@converter` 方法，将“批量”赋值中使用的对象转换为一系列值，例如：

    ```py
    obj.col =
    # or
    obj.dictcol = {'foo': newval1, 'bar': newval2}
    ```

    MappedCollection 使用此钩子确保从集合的角度来看传入的键/值对是合理的。

+   **[orm]**

    在双向关系的两端同时使用 `lazy="dynamic"` 时修复了无限循环问题。

    参考：[#872](https://www.sqlalchemy.org/trac/ticket/872)

+   **[orm]**

    在 Query + eagerloads 中应用 LIMIT/OFFSET 别名时修复了更多问题，例如当与 select 语句进行映射时。

    参考：[#904](https://www.sqlalchemy.org/trac/ticket/904)

+   **[orm]**

    修复了自引用的急加载问题，即使在同一结果集中的两个或更多不同列集中出现相同映射的实例，其急加载集合也将被填充，无论所有行是否包含该集合的“急加载”列集。当打开 join_depth 时，这也会显示为 KeyError。

+   **[orm]**

    修复了在使用 LIMIT 与仅在父映射器中存在急加载器的继承映射器时，Query 不会将子查询应用于 SQL 的 bug。

+   **[orm]**

    澄清了当尝试使用相同标识键更新()已经存在于会话中的实例时发生的错误消息。

+   **[orm]**

    对 merge(instance, dont_load=True)进行了一些澄清和修复。修复了在返回的实例上禁用延迟加载器的 bug。此外，我们目前不支持在实例上有未提交更改时使用 dont_load=True 的合并实例的情况...这将引发错误。这是因为合并给定实例的“已提交状态”以正确对应新复制的实例以及其他修改状态的复杂性。由于 dont_load=True 的用例是缓存，给定实例不应该有任何未提交更改。我们现在也不使用任何事件复制实例，以便新会话上的“脏”列表保持不受影响。

+   **[orm]**

    修复了在使用 session.begin_nested()与多于一级的封闭 session.begin()语句时可能出现的 bug

+   **[orm]**

    修复了使用具有自定义实体名称的实例进行 session.refresh()时的问题

    参考：[#914](https://www.sqlalchemy.org/trac/ticket/914)

### sql

+   **[sql]**

    通用函数！我们引入了一个已知 SQL 函数的数据库，例如 current_timestamp、coalesce，并创建了表示它们的显式函数对象。这些对象具有受限制的参数列表，具有类型意识，并且可以以特定于方言的方式编译。因此，说 func.char_length(“foo”, “bar”)会引发错误（参数太多），func.coalesce(datetime.date(2007, 10, 5), datetime.date(2005, 10, 15))知道其返回类型是 Date。到目前为止，我们只表示了一些函数，但将继续添加到系统中

    参考：[#615](https://www.sqlalchemy.org/trac/ticket/615)

+   **[sql]**

    改进了自动重新连接支持；现在，Connection 在其基础连接失效后可以自动重新连接，而不需要再次从 engine 连接。这使得绑定到单个 Connection 的 ORM 会话不需要重新连接。在基础连接失效后，Connection 上的未完成事务必须回滚，否则会引发错误。还修复了在 cursor()、rollback()或 commit()中未调用 disconnect detect 的 bug。

+   **[sql]**

    为 String 和 create_engine()添加了新标志，assert_unicode=(True|False|’warn’|None)。在 create_engine()和 String 上默认为 False 或 None，Unicode 类型上为‘warn’。当为 True 时，所有 Unicode 转换操作在传递非 Unicode 字节字符串作为绑定参数时会引发异常。‘warn’会产生警告。强烈建议所有支持 Unicode 的应用程序正确使用 Python Unicode 对象（即 u’hello’而不是‘hello’），以便数据往返准确。

+   **[sql]**

    “unique”绑定参数的生成已简化为使用与其他所有内容相同的“唯一标识符”机制。这不会影响用户代码，除非可能已经针对生成的名称硬编码的任何代码。生成的绑定参数现在具有“<paramname>_<num>”的形式，而以前只有相同名称的第二个绑定会具有这种形式。

+   **[sql]**

    select().as_scalar()如果 select 在其 columns 子句中没有确切一个表达式，则会引发异常。

+   **[sql]**

    bindparam()对象本身可以用作 execute()的键，即 statement.execute({bind1:’foo’, bind2:’bar’})

+   **[sql]**

    添加了 TypeDecorator 的新方法，process_bind_param()和 process_result_value()，它们自动利用底层类型的处理。非常适合与 Unicode 或 PickleType 一起使用。TypeDecorator 现在应该是增强任何现有类型行为的主要方式，包括其他 TypeDecorator 子类，如 PickleType。

+   **[sql]**

    selectables（和其他对象）在其导出的列集合中基于名称冲突的两列时将发出警告。

+   **[sql]**

    具有模式的表仍然可以在 sqlite、firebird 中使用，模式名称只是被删除

    参考：[#890](https://www.sqlalchemy.org/trac/ticket/890)

+   **[sql]**

    更改各种“literal”生成函数以使用匿名绑定参数。这里没有太多变化，只是它们的标签现在看起来像“:param_1”，“:param_2”，而不是“:literal”

+   **[sql]**

    现在支持形式为“tablename.columname”的列标签，即带有点的形式。

+   **[sql]**

    select()的 from_obj 关键字参数可以是标量或列表。

### 杂项

+   **[dialects]**

    sqlite SLDate 类型不会错误地呈现日期时间或时间对象的“微秒”部分。

+   **[dialects]**

    oracle

    +   添加了对 Oracle 的断开连接检测支持

    +   对二进制/原始类型进行了一些清理，以便在特定情况下检测到 cx_oracle.LOB

    参考：[#902](https://www.sqlalchemy.org/trac/ticket/902)

+   **[dialects]**

    MSSQL

    +   PyODBC 不再具有全局“set nocount on”。

    +   修复自动加载时非标识整数主键

    +   更好地支持 convert_unicode

    +   对于 pyodbc/adodbapi 的日期转换不那么严格

    +   模式限定的表/自动加载

    参考：[#824](https://www.sqlalchemy.org/trac/ticket/824), [#839](https://www.sqlalchemy.org/trac/ticket/839), [#842](https://www.sqlalchemy.org/trac/ticket/842), [#901](https://www.sqlalchemy.org/trac/ticket/901)

+   **[backend] [firebird]**

    正确反映域（部分修复）和 PassiveDefaults

    参考：[#410](https://www.sqlalchemy.org/trac/ticket/410)

+   **[3562] [backend] [firebird]**

    回退到使用默认的 poolclass（在 0.4.0 中设置为 SingletonThreadPool 用于测试目的）

+   **[backend] [firebird]**

    将 func.length() 映射到 ‘char_length’（在旧版本的 Firebird 上可以轻松通过 UDF ‘strlen’ 进行覆盖）

### orm

+   **[orm]**

    针对基于集合的反向引用的重大行为更改：它们不再触发延迟加载！“reverse” 添加和删除被排队并在实际读取和加载集合时与集合合并；但不会事先触发加载。对于注意到这种行为的用户，在某些情况下，这应该比在某些情况下使用动态关系更方便；对于那些没有注意到的用户，您可能会注意到您的应用程序在某些情况下使用的查询比以前少得多。

    参考：[#871](https://www.sqlalchemy.org/trac/ticket/871)

+   **[orm]**

    添加了可变主键支持。主键列可以自由更改，并且在刷新时实例的标识将会改变。此外，支持沿关系更新外键引用（主键或非主键），可以与数据库的 ON UPDATE CASCADE（对于像 Postgres 这样的数据库是必需的）一起使用，或者直接由 ORM 以 UPDATE 语句的形式发出，通过设置标志“passive_cascades=False”。

+   **[orm]**

    继承的映射器现在直接继承其父映射器的 MapperExtensions，因此特定 MapperExtension 的所有方法也将为子类调用。与往常一样，任何 MapperExtension 都可以返回 EXT_CONTINUE 继续扩展处理或 EXT_STOP 停止处理。映射器解析的顺序是：<在类的映射器上声明的扩展> <在类的父映射器上声明的扩展> <全局声明的扩展>。

    请注意，如果您单独实例化相同的扩展类，然后分别将其应用于同一继承链中的两个映射器，那么该扩展将应用两次于继承类，并且每个方法将被调用两次。

    要将映射器扩展显式应用于每个继承类，但每个方法每次操作只调用一次，请为两个映射器使用相同的扩展实例。

    参考：[#490](https://www.sqlalchemy.org/trac/ticket/490)

+   **[orm]**

    MapperExtension.before_update() 和 after_update() 现在被对称调用；以前，一个实例如果没有修改的列属性（但有关系() 修改）可能会在 before_update() 被调用但不会在 after_update() 被调用。

    参考：[#907](https://www.sqlalchemy.org/trac/ticket/907)

+   **[orm]**

    查询语句中缺失的列现在在加载时会自动延迟加载。

+   **[orm]**

    扩展“object”并且在实例构造时没有提供 __init__() 方法的映射类现在会在实例构造时引发 TypeError，如果非空的 *args 或 **kwargs 在实例构造���存在（并且没有被任何扩展（如 scoped_session 映射器）消耗），与普通 Python 类的行为一致。

    参考：[#908](https://www.sqlalchemy.org/trac/ticket/908)

+   **[orm]**

    修复了 Query 在 filter_by() 将关系与 None 进行比较时的 bug

    参考：[#899](https://www.sqlalchemy.org/trac/ticket/899)

+   **[orm]**

    改进了对映射实体的 pickling 支持。每个实例的延迟/延迟/过期可调用现在是可序列化的，因此它们与 _state 一起序列化和反序列化。

+   **[orm]**

    新的 synonym() 行为：如果映射类上不存在属性，则属性将放置在映射类上。如果类上已经存在属性，则 synonym 将使用适当的比较运算符装饰属性，以便它可以像任何其他映射属性一样在列表达式中使用（即可用于 filter() 等）。“proxy=True”标志已被弃用，不再起作用。此外，“map_column=True”标志将自动生成与同义词名称对应的 ColumnProperty，例如：‘somename’:synonym(‘_somename’, map_column=True) 将将名为‘somename’的列映射到属性‘_somename’。请参阅映射器文档中的示例。

    参考：[#801](https://www.sqlalchemy.org/trac/ticket/801)

+   **[orm]**

    Query.select_from() 现在用给定的参数替换所有现有的 FROM 条件；以前构造 FROM 子句列表的行为通常不实用，因为需要 filter() 调用来创建连接条件，并且在 filter() 中引入的新表已经添加到 FROM 子句中。新行为不仅允许从主表进行连接，还允许选择语句。过滤条件、排序和急加载子句将“针对”给定语句进行别名处理。

+   **[orm]**

    本月对属性检测的重构改变了自 0.3 中期以来我们一直拥有的“加载时复制”行为，大多数情况下改为“修改时复制”。这减少了加载操作中的相当大的延迟，并且总体上做的工作更少，因为只有实际修改的属性才会复制其“已提交状态”。只有“可变标量”属性（即 pickled 对象或其他可变项），即第一次更改加载时的复制行为的原因，保留了旧行为。

+   **[orm] [attrname]**

    对属性进行轻微的行为更改，删除属性不会再次触发该属性的延迟加载器；“del”使属性的有效值为“None”。要重新触发属性的“加载器”，请使用 session.expire(instance,)。

+   **[orm]**

    query.filter(SomeClass.somechild == None)，当将一个多对一属性与 None 进行比较时，会正确生成“id IS NULL”，包括 NULL 在右侧的情况。

+   **[orm]**

    query.order_by() 考虑到别名连接，即 query.join('orders', aliased=True).order_by(Order.id)

+   **[orm]**

    eagerload()、lazyload()、eagerload_all() 接受可选的第二个类或映射器参数，这将选择要应用选项的映射器。这可以选择使用 add_entity() 添加的其他映射器之一。

+   **[orm]**

    eagerloading 将与通过 add_entity() 添加的映射器一起工作。

+   **[orm]**

    就像普通关系一样，对“动态”关系增加了“级联删除”行为。如果未设置 passive_deletes 标志（刚刚添加），则删除父项目将触发对子项目的完整加载，以便可以相应地删除或更新它们。

+   **[orm]**

    同样对动态实现了正确的 count() 行为以及其他辅助方法。

+   **[orm]**

    修复了关于多态关系的级联错误，使得从对象到多态集合的级联继续沿着集合中每个元素特定的属性集合进行级联。

+   **[orm]**

    query.get() 和 query.load() 不考虑现有的过滤器或其他条件；这些方法*总是*在数据库中查找给定的 id 或从标识映射中返回当前实例，而不考虑任何已配置的现有过滤器、连接、group_by 或其他条件。

    引用：[#893](https://www.sqlalchemy.org/trac/ticket/893)

+   **[orm]**

    在继承映射器的情况下增加了对 version_id_col 的支持。version_id_col 通常在继承关系中设置在基映射器上，它对所有继承映射器生效。

    引用：[#883](https://www.sqlalchemy.org/trac/ticket/883)

+   **[orm]**

    放宽了 column_property() 表达式的标签规则；现在接受任何 ColumnElement，因为编译器现在自动为非标记的 ColumnElement 添加标签。可选择的，比如 select() 语句，仍然需要通过 as_scalar() 或 label() 转换为 ColumnElement。

+   **[orm]**

    修复了 backref bug，如果 attr 为 None，则无法删除 instance.attr。

+   **[orm]**

    几个 ORM 属性已被删除或设置为私有：mapper.get_attr_by_column()、mapper.set_attr_by_column()、mapper.pks_by_table、mapper.cascade_callable()、MapperProperty.cascade_callable()、mapper.canload()、mapper.save_obj()、mapper.delete_obj()、mapper._mapper_registry、attributes.AttributeManager

+   **[orm]**

    将不兼容的集合类型分配给关系属性现在会引发 TypeError 而不是 SQLAlchemy 的 ArgumentError。

+   **[orm]**

    如果传入字典中的键与集合的 keyfunc 为该值使用的键不匹配，则 MappedCollection 的批量赋值现在会引发错误。

    引用：[#886](https://www.sqlalchemy.org/trac/ticket/886)

+   **[orm] [newval1] [newval2]**

    自定义集合现在可以指定一个 @converter 方法来将用于“批量”赋值的对象转换为值流，如下所示：

    ```py
    obj.col =
    # or
    obj.dictcol = {'foo': newval1, 'bar': newval2}
    ```

    MappedCollection 使用此钩子来确保传入的键值对从集合的角度看是合理的。

+   **[orm]**

    修复了在双向关系的两侧都使用 lazy=”dynamic”时出现无限循环问题

    参考：[#872](https://www.sqlalchemy.org/trac/ticket/872)

+   **[orm]**

    在 Query + 急加载中应用 LIMIT/OFFSET 别名的更多修复，特别是在与 select 语句映射时

    参考：[#904](https://www.sqlalchemy.org/trac/ticket/904)

+   **[orm]**

    修复了自引用急加载的问题，即使同一映射实例出现在同一结果集中的两个或更多不同列集中，其急加载的集合也将被填充，无论所有行是否包含该集合的“急加载”列集。当打开 join_depth 时，这也会显示为 KeyError。

+   **[orm]**

    修复了在使用继承映射器时，当在父映射器中仅存在急加载器时，Query 在使用 LIMIT 与子查询时不会将子查询应用于 SQL 的错误。

+   **[orm]**

    澄清了当尝试使用相同标识键更新具有与会话中已存在实例相同标识键的实例时发生的错误消息。

+   **[orm]**

    对 merge(instance, dont_load=True)进行了一些澄清和修复。修复了在返回的实例上禁用延迟加载器的错误。此外，我们目前不支持在实例上有未提交更改时使用 dont_load=True 进行合并….这将引发错误。这是由于将给定实例的“已提交状态”正确合并到新复制的实例以正确对应其他修改状态的复杂性。由于 dont_load=True 的用例是缓存，因此给定实例不应该有任何未提交的更改。我们现在也在不使用任何事件的情况下复制实例，以使新会话上的“脏”列表保持不受影响。

+   **[orm]**

    修复了在使用 session.begin_nested()与多层嵌套 session.begin()语句时可能出现的错误

+   **[orm]**

    修复了具有自定义 entity_name 的实例使用 session.refresh()时的无限循环问题

    参考：[#914](https://www.sqlalchemy.org/trac/ticket/914)

### sql

+   **[sql]**

    通用函数！我们引入了一个已知 SQL 函数的数据库，例如 current_timestamp，coalesce，并创建了表示它们的显式函数对象。这些对象具有受限制的参数列表，具有类型意识，并且可以以特定于方言的方式编译。因此，说 func.char_length(“foo”, “bar”)会引发错误（参数太多），func.coalesce(datetime.date(2007, 10, 5), datetime.date(2005, 10, 15))知道其返回类型是 Date。到目前为止，我们只表示了一些函数，但将继续添加到系统中

    参考：[#615](https://www.sqlalchemy.org/trac/ticket/615)

+   **[sql]**

    自动重新连接支持得到改进；现在连接在其底层连接失效后可以自动重新连接，而不需要再次从引擎连接()。这使得绑定到单个连接的 ORM 会话不需要重新连接。在底层连接失效后，连接上的打开事务必须回滚，否则会引发错误。还修复了在 cursor()、rollback()或 commit()中未调用断开检测的错误。

+   **[sql]**

    为 String 和 create_engine()添加了新标志，assert_unicode=(True|False|’warn’|None)。在 create_engine()和 String 上默认为 False 或 None，Unicode 类型上为‘warn’。当为 True 时，当传递非 Unicode 字节字符串作为绑定参数时，所有 Unicode 转换操作都会引发异常。‘warn’会产生警告。强烈建议所有支持 Unicode 的应用程序正确使用 Python Unicode 对象（即 u’hello’而不是‘hello’），以便数据往返准确。

+   **[sql]**

    “唯一”绑定参数的生成已简化为使用与其他所有内容相同的“唯一标识符”机制。这不会影响用户代码，除非可能已经针对生成的名称进行了硬编码。生成的绑定参数现在的形式为“<paramname>_<num>”，而以前只有同名的第二个绑定才会有这种形式。

+   **[sql]**

    select().as_scalar()如果在其列子句中没有确切一个表达式，将会引发异常。

+   **[sql]**

    bindparam()对象本身可以用作 execute()的键，即 statement.execute({bind1:’foo’, bind2:’bar’})

+   **[sql]**

    添加了 TypeDecorator 的新方法 process_bind_param()和 process_result_value()，它们自动利用底层类型的处理。非常适合与 Unicode 或 PickleType 一起使用。TypeDecorator 现在应该是增强任何现有类型行为的主要方式，包括其他 TypeDecorator 子类，如 PickleType。

+   **[sql]**

    selectables（以及其他对象）在其导出列集合中基于名称冲突的两个列时会发出警告。

+   **[sql]**

    在 sqlite、firebird 中仍然可以使用具有模式的表，模式名称只会被删除

    参考：[#890](https://www.sqlalchemy.org/trac/ticket/890)

+   **[sql]**

    将各种“literal”生成函数更改为使用匿名绑定参数。这里没有太多变化，除了它们的标签现在看起来像“:param_1”，“:param_2”而不是“:literal”

+   **[sql]**

    现在支持形式为“tablename.columname”的列标签，即带有点的形式。

+   **[sql]**

    select()的 from_obj 关键字参数可以是标量或列表。

### misc

+   **[dialects]**

    sqlite SLDate 类型不会错误地呈现日期时间或时间对象的“微秒”部分。

+   **[dialects]**

    oracle

    +   为 Oracle 添加了断开检测支持

    +   对二进制/原始类型进行了一些清理，以便在需要时检测 cx_oracle.LOB

    参考：[#902](https://www.sqlalchemy.org/trac/ticket/902)

+   **[dialects]**

    MSSQL

    +   PyODBC 不再具有全局“set nocount on”。

    +   修复 autoload 上的非标识整数 PKs

    +   更好地支持 convert_unicode

    +   对于 pyodbc/adodbapi 的日期转换不再那么严格

    +   模式限定的表/自动加载

    参考：[#824](https://www.sqlalchemy.org/trac/ticket/824), [#839](https://www.sqlalchemy.org/trac/ticket/839), [#842](https://www.sqlalchemy.org/trac/ticket/842), [#901](https://www.sqlalchemy.org/trac/ticket/901)

+   **[backend] [firebird]**

    现在正确反映域（部分修复）和 PassiveDefaults

    参考：[#410](https://www.sqlalchemy.org/trac/ticket/410)

+   **[3562] [backend] [firebird]**

    恢复为使用默认的 poolclass（在 0.4.0 中设置为 SingletonThreadPool 仅用于测试目的）

+   **[backend] [firebird]**

    将 func.length()映射到‘char_length’（在旧版本的 Firebird 上可以轻松覆盖为 UDF‘strlen’）

## 0.4.1

发布日期：Sun Nov 18 2007

### orm

+   **[orm]**

    eager loading with LIMIT/OFFSET applied no longer adds the primary table joined to a limited subquery of itself; the eager loads now join directly to the subquery which also provides the primary table’s columns to the result set. This eliminates a JOIN from all eager loads with LIMIT/OFFSET.

    参考：[#843](https://www.sqlalchemy.org/trac/ticket/843)

+   **[orm]**

    session.refresh()和 session.expire()现在支持额外的参数“attribute_names”，一个包含要刷新或过期的单个属性键名列表，允许对已加载实例进行部分重新加载。

    参考：[#802](https://www.sqlalchemy.org/trac/ticket/802)

+   **[orm]**

    添加 op()操作符到 instrumented attributes；例如 User.name.op(‘ilike’)(‘%somename%’)

    参考：[#767](https://www.sqlalchemy.org/trac/ticket/767)

+   **[orm]**

    映射类现在可以定义具有任意语义的 __eq__、__hash__ 和 __nonzero__ 方法。orm 现在仅基于标识处理所有映射实例。（例如‘is’ vs ‘==’）

    参考：[#676](https://www.sqlalchemy.org/trac/ticket/676)

+   **[orm]**

    Mapper 上的“properties”访问器已被移除；现在会抛出一个信息性异常，解释 mapper.get_property()和 mapper.iterate_properties 的用法

+   **[orm]**

    添加 having()方法到 Query，将 HAVING 应用于生成的语句，方式与 filter()追加到 WHERE 子句相同。

+   **[orm]**

    现在，query.options()的行为完全基于路径，即诸如 eagerload_all(‘x.y.z.y.x’)这样的选项将仅应用于这些路径，即不包括‘x.y.x’；eagerload(‘children.children’)仅适用于确切的两级深度等。

    参考：[#777](https://www.sqlalchemy.org/trac/ticket/777)

+   **[orm]**

    当设置为 mutable=False 时，PickleType 将使用==进行比较，而不是 is 操作符。要使用 is 或任何其他比较器，请使用 PickleType(comparator=my_custom_comparator)发送自定义比较函数。

+   **[orm]**

    如果同时使用 distinct()和包含 UnaryExpressions（或其他）的 order_by()，查询不会引发错误

    参考：[#848](https://www.sqlalchemy.org/trac/ticket/848)

+   **[orm]**

    当使用 distinct()时，来自连接表的 order_by()表达式会被正确添加到列子句中

    参考：[#786](https://www.sqlalchemy.org/trac/ticket/786)

+   **[orm]**

    修复了 Query.add_column()不接受类绑定属性作为参数的错误；Query 在 add_column()（在 instances()时间）发送无效参数时也会引发错误

    参考：[#858](https://www.sqlalchemy.org/trac/ticket/858)

+   **[orm]**

    在 InstanceState.__cleanup()中增加了更多垃圾回收解除引用的检查，以减少应用程序关闭时的“gc ignored”错误

+   **[orm]**

    会话 API 已经稳定：

+   **[orm]**

    对已经持久化的对象进行 session.save()会报错

    参考：[#840](https://www.sqlalchemy.org/trac/ticket/840)

+   **[orm]**

    对于*非*持久化的对象进行 session.delete()会报错。

+   **[orm]**

    当更新或删除已在会话中具有不同标识的实例时，session.update()和 session.delete()会引发错误。

+   **[orm]**

    在确定“对象 X 已经在另一个会话中”时，会话会更加仔细检查；例如，如果您 pickle 一系列对象并 unpickle（即在 Pylons HTTP 会话或类似情况下），它们可以进入新会话而不会发生冲突

+   **[orm]**

    merge()包括一个关键字参数“dont_load=True”。设置此标志将导致合并操作不从数据库中加载任何数据以响应传入的分离对象，并将接受传入的分离对象，就好像它已经存在于该会话中。使用此功能将外部缓存系统中的分离对象合并到会话中。

+   **[orm]**

    当将 Deferred 列属性分配给时，不再触发加载操作。在这些情况下，新分配的值将无条件地出现在 flushes 的 UPDATE 语句中。

+   **[orm]**

    重新分配集合的子集时修复了截断错误（obj.relation = obj.relation[1:]）

    参考：[#834](https://www.sqlalchemy.org/trac/ticket/834)

+   **[orm]**

    精简了 backref 配置代码，覆盖现有属性的 backrefs 现在会引发错误

    参考：[#832](https://www.sqlalchemy.org/trac/ticket/832)

+   **[orm]**

    改进了 add_property()等的行为，修复了涉及 synonym/deferred 的问题。

    参考：[#831](https://www.sqlalchemy.org/trac/ticket/831)

+   **[orm]**

    修复了 clear_mappers()行为，以更好地清理自身之后。

+   **[orm]**

    修复了“行切换”行为，即当 INSERT/DELETE 合并为单个 UPDATE 时；父对象上的多对多关系会正确更新。

    参考：[#841](https://www.sqlalchemy.org/trac/ticket/841)

+   **[orm]**

    修复了关联代理的哈希值，这些集合是不可哈希的，就像它们的可变 Python 对应物一样。

+   **[orm]**

    为作用域会话添加了 save_or_update、__contains__ 和 __iter__ 方法的代理。

+   **[orm]**

    修复了一个非常难以重现的问题，即 Query 的 FROM 子句可能会被某些生成调用污染

    参考：[#852](https://www.sqlalchemy.org/trac/ticket/852)

### sql

+   **[sql]**

    bindparam()上的“shortname”关键字参数已被弃用。

+   **[sql]**

    添加了包含运算符（生成一个“LIKE %<other>%”子句）。

+   **[sql]**

    匿名列表达式会自动标记。例如，select([x* 5])会产生“SELECT x * 5 AS anon_1”。这允许标签名出现在 cursor.description 中，然后可以与结果列处理规则适当匹配（我们无法可靠地使用位置跟踪进行结果列匹配，因为 text()表达式可能代表多个列）。

+   **[sql]**

    运算符重载现在由 TypeEngine 对象控制 - 到目前为止内置的运算符重载是 String 类型重载‘+’成为字符串连接运算符。用户定义的类型也可以通过覆盖 adapt_operator(self, op)方法来定义自己的运算符重载。

+   **[sql]**

    二元表达式右侧的无类型绑定参数将被分配为操作左侧的类型，以更好地���用适当的绑定参数处理

    参考：[#819](https://www.sqlalchemy.org/trac/ticket/819)

+   **[sql]**

    从大多数语句编译中删除了正则表达式步骤。同时修复了

    参考：[#833](https://www.sqlalchemy.org/trac/ticket/833)

+   **[sql]**

    修复了空（零列）sqlite 插入，允许在自增单列表上插入。

+   **[sql]**

    修复了 text()子句的表达式翻译；这修复了各种 ORM 场景，其中使用文字文本作为 SQL 表达式

+   **[sql]**

    移除了 ClauseParameters 对象；现在 compiled.params 返回一个普通的字典，以及 result.last_inserted_params() / last_updated_params()。

+   **[sql]**

    修复了关于具有基于 SQL 表达式的默认生成器的主键列的 INSERT 语句；SQL 表达式像往常一样内联执行，但不会触发列的“postfetch”条件，对于那些通过 cursor.lastrowid 提供它的 DB

+   **[sql]**

    func.对象可以被 pickle/unpickle

    参考：[#844](https://www.sqlalchemy.org/trac/ticket/844)

+   **[sql]**

    重写并简化了用于在可选择表达式之间“定位”列的系统。在 SQL 方面，这由“corresponding_column()”方法表示。这种方法在 ORM 中被大量使用，用于将表达式的元素“适应”到类似的，别名的表达式，以及将最初绑定到表或可选择到别名的结果集列的“对应”表达式。新的重写功能具有完全一致和准确的行为。

+   **[sql]**

    添加了一个字段（“info”）用于在模式项上存储任意数据

    参考：[#573](https://www.sqlalchemy.org/trac/ticket/573)

+   **[sql]**

    Connections 上的 “properties” 集合已重命名为 “info” 以匹配 schema 的可写集合。直到 0.5 版本，仍然可以通过 “properties” 名称访问。

+   **[sql]**

    修复了在使用 strategy=’threadlocal’ 时 Transaction 上的 close() 方法

+   **[sql]**

    修复编译绑定参数不会错误地填充 None 的问题

    参考：[#853](https://www.sqlalchemy.org/trac/ticket/853)

+   **[sql]**

    <Engine|Connection>._execute_clauseelement 变为公共方法 Connectable.execute_clauseelement

### 杂项

+   **[方言]**

    添加了对 MaxDB（版本 >= 7.6.03.007）的实验性支持。

+   **[方言]**

    现在 Oracle 将“DATE”反映为 OracleDateTime 列，而不是 OracleDate

+   **[方言]**

    在 oracle table_names() 函数中添加了对模式名称的意识，修复了 metadata.reflect(schema=’someschema’) 的问题

    参考：[#847](https://www.sqlalchemy.org/trac/ticket/847)

+   **[方言]**

    MSSQL 匿名标签用于选择函数变得确定性

+   **[方言]**

    sqlite 将“DECIMAL”反映为数字列。

+   **[方言]**

    使访问 dao 检测更可靠

    参考：[#828](https://www.sqlalchemy.org/trac/ticket/828)

+   **[方言]**

    将 Dialect 属性 ‘preexecute_sequences’ 重命名为 ‘preexecute_pk_sequences’。对于使用旧名称的 out-of-tree 方言，现在有一个属性代理。

+   **[方言]**

    为未知类型反射添加了测试覆盖。修复了 sqlite/mysql 对未知类型的类型反射处理。

+   **[方言]**

    为 mysql 方言添加了 REAL（用于利用 REAL_AS_FLOAT sql 模式的人）。

+   **[方言]**

    mysql Float、MSFloat 和 MSDouble 现在在没有参数的情况下生成无参数 DDL，例如 ‘FLOAT’。

+   **[杂项]**

    删除了未使用的 util.hash()。

### orm

+   **[orm]**

    应用了 LIMIT/OFFSET 的急加载不再将主表连接到自身的限制子查询；急加载现在直接连接到提供主表列给结果集的子查询。这消除了所有带有 LIMIT/OFFSET 的急加载的 JOIN。

    参考：[#843](https://www.sqlalchemy.org/trac/ticket/843)

+   **[orm]**

    session.refresh() 和 session.expire() 现在支持额外的参数“attribute_names”，一个要刷新或过期的单个属性键名列表，允许对已加载实例的属性进行部分重新加载。

    参考：[#802](https://www.sqlalchemy.org/trac/ticket/802)

+   **[orm]**

    为被检测属性添加了 op() 操作符；即 User.name.op(‘ilike’)(‘%somename%’)

    参考：[#767](https://www.sqlalchemy.org/trac/ticket/767)

+   **[orm]**

    映射类现在可以定义具有任意语义的 __eq__、__hash__ 和 __nonzero__ 方法。orm 现在仅基于标识处理所有映射实例。（例如 ‘is’ vs ‘==’）

    参考：[#676](https://www.sqlalchemy.org/trac/ticket/676)

+   **[orm]**

    Mapper 上的 “properties” 访问器已移除；现在会抛出一个信息性异常，解释 mapper.get_property() 和 mapper.iterate_properties 的用法

+   **[orm]**

    向 Query 添加了 having() 方法，将 HAVING 应用于生成的语句，方式与 filter() 将条件追加到 WHERE 子句中的方式相同。

+   **[orm]**

    现在 query.options() 的行为完全基于路径，即诸如 eagerload_all(‘x.y.z.y.x’) 这样的选项将仅应用于这些路径，并且不会应用于 ‘x.y.x’；eagerload(‘children.children’) 仅应用于恰好两层深度等。

    参考：[#777](https://www.sqlalchemy.org/trac/ticket/777)

+   **[orm]**

    如果设置 PickleType 的 mutable=False，则 PickleType 将使用 == 进行比较，而不是 is 操作符。要使用 is 或其他比较器，请使用 PickleType(comparator=my_custom_comparator) 发送自定义比较函数。

+   **[orm]**

    如果在 query 中同时使用 distinct() 和包含 UnaryExpressions（或其他）的 order_by()，则不会抛出错误。

    参考：[#848](https://www.sqlalchemy.org/trac/ticket/848)

+   **[orm]**

    在使用 distinct() 时，来自连接表的 order_by() 表达式将正确添加到列子句中。

    参考：[#786](https://www.sqlalchemy.org/trac/ticket/786)

+   **[orm]**

    修复了 Query.add_column() 不接受类绑定属性作为参数的错误；Query 还会在传递无效参数给 add_column()（在 instances() 时间）时报错。

    参考：[#858](https://www.sqlalchemy.org/trac/ticket/858)

+   **[orm]**

    在 InstanceState.__cleanup() 中增加了更多对垃圾收集引用的检查，以减少应用程序关闭时出现的“gc ignored”错误。

+   **[orm]**

    会话 API 已经稳定：

+   **[orm]**

    当尝试对已经持久化的对象进行 session.save() 时会报错。

    参考：[#840](https://www.sqlalchemy.org/trac/ticket/840)

+   **[orm]**

    当尝试对 *未* 持久化的对象进行 session.delete() 时会报错。

+   **[orm]**

    当尝试对已经在会话中以不同标识符存在的实例进行更新或删除时，session.update() 和 session.delete() 会报错。

+   **[orm]**

    在确定“对象 X 已经在另一个会话中”时，会话现在会更仔细地进行检查；例如，如果您将一系列对象进行 pickle，并在稍后反 pickle（即在 Pylons HTTP 会话或类似情况下），它们可以在没有任何冲突的情况下进入新会话。

+   **[orm]**

    merge() 包含一个名为 “dont_load=True” 的关键字参数。设置此标志将导致合并操作不会在响应传入的分离对象时从数据库中加载任何数据，并且将接受传入的分离对象，就好像它已经存在于该会话中。可以使用此功能将外部缓存系统中的分离对象合并到会话中。

+   **[orm]**

    当分配属性时，延迟列属性不再触发加载操作。在这些情况下，新分配的值将无条件地出现在刷新的 UPDATE 语句中。

+   **[orm]**

    修复了重新分配集合子集时出现截断错误的问题（obj.relation = obj.relation[1:]）。

    参考：[#834](https://www.sqlalchemy.org/trac/ticket/834)

+   **[orm]**

    精简了 backref 配置代码，覆盖现有属性的 backrefs 现在会引发错误

    参考：[#832](https://www.sqlalchemy.org/trac/ticket/832)

+   **[orm]**

    改进了 add_property()等的行为，修复了涉及 synonym/deferred 的问题。

    参考：[#831](https://www.sqlalchemy.org/trac/ticket/831)

+   **[orm]**

    修复了 clear_mappers()行为，以更好地清理自身之后。

+   **[orm]**

    修复了“行切换”行为，即当 INSERT/DELETE 合并为单个 UPDATE 时；父对象上的多对多关系正确更新。

    参考：[#841](https://www.sqlalchemy.org/trac/ticket/841)

+   **[orm]**

    修复了关联代理的 __hash__ - 这些集合是不可哈希的，就像它们的可变 Python 对应物一样。

+   **[orm]**

    为作用域会话添加了 save_or_update、__contains__ 和 __iter__ 方法的代理。

+   **[orm]**

    修复了一个非常难以复现的问题，即 Query 的 FROM 子句可能会被某些生成调用污染

    参考：[#852](https://www.sqlalchemy.org/trac/ticket/852)

### sql

+   **[sql]**

    bindparam()上的“shortname”关键字参数已被弃用。

+   **[sql]**

    添加了 contains 运算符（生成一个“LIKE %<other>%”子句）。

+   **[sql]**

    匿名列表达式会自动标记。例如，select([x* 5])会产生“SELECT x * 5 AS anon_1”。这允许标签名存在于 cursor.description 中，然后可以与结果列处理规则适当匹配。（我们无法可靠地使用位置跟踪进行结果列匹配，因为 text()表达式可能代表多个列）。

+   **[sql]**

    运算符重载现在由 TypeEngine 对象控制 - 到目前为止内置的运算符重载是 String 类型重载‘+’成为字符串连接运算符。用户定义的类型也可以通过覆盖 adapt_operator(self, op)方法来定义自己的运算符重载。

+   **[sql]**

    二元表达式右侧的无类型绑定参数将被分配为操作左侧的类型，以更好地启用适当的绑定参数处理

    参考：[#819](https://www.sqlalchemy.org/trac/ticket/819)

+   **[sql]**

    从大多数语句编译中移除了正则表达式步骤。还修复了

    参考：[#833](https://www.sqlalchemy.org/trac/ticket/833)

+   **[sql]**

    修复了空（零列）sqlite 插入，允许在自动递增单列表上插入。

+   **[sql]**

    修复了 text()子句的表达式转换；这修复了各种 ORM 场景中使用文字文本作为 SQL 表达式的情况

+   **[sql]**

    移除了 ClauseParameters 对象；compiled.params 现在返回一个常规字典，以及 result.last_inserted_params() / last_updated_params()。

+   **[sql]**

    修复了关于具有基于 SQL 表达式的默认生成器的主键列的 INSERT 语句；SQL 表达式像往常一样内联执行，但不会为该列触发“postfetch”条件，对于那些通过 cursor.lastrowid 提供它的 DB

+   **[sql]**

    func. 对象可以被 pickled/unpickled

    参考：[#844](https://www.sqlalchemy.org/trac/ticket/844)

+   **[SQL]**

    重写并简化了用于在可选择表达式之间“定位”列的系统。在 SQL 方面，这由 “corresponding_column()” 方法表示。此方法由 ORM 大量使用，以将表达式的元素“适应”为类似的、别名的表达式，以及将原始绑定到表或可选择到别名的结果集列的表达式“定位”为相应的表达式。新的重写具有完全一致和准确的行为。

+   **[SQL]**

    添加了一个字段（“info”）用于在模式项上存储任意数据

    参考：[#573](https://www.sqlalchemy.org/trac/ticket/573)

+   **[SQL]**

    连接上的“properties”集合已重命名为“info”，以匹配模式的可写集合。在 0.5 版本之前，仍可通过“properties”名称访问。

+   **[SQL]**

    修复了使用 strategy='threadlocal' 时 Transaction 上的 close() 方法

+   **[SQL]**

    修复了编译绑定参数不会错误地填充 None 的问题

    参考：[#853](https://www.sqlalchemy.org/trac/ticket/853)

+   **[SQL]**

    <Engine|Connection>._execute_clauseelement 变为公共方法 Connectable.execute_clauseelement

### 杂项

+   **[方言]**

    添加了对 MaxDB（版本 >= 7.6.03.007）的实验性支持。

+   **[方言]**

    oracle 现在将“DATE”反映为 OracleDateTime 列，而不是 OracleDate

+   **[方言]**

    在 oracle table_names() 函数中增加了对模式名称的意识，修复了 metadata.reflect(schema='someschema')

    参考：[#847](https://www.sqlalchemy.org/trac/ticket/847)

+   **[方言]**

    MSSQL 对函数选择的匿名标签使确定性

+   **[方言]**

    sqlite 将“DECIMAL”反映为数字列。

+   **[方言]**

    使访问 dao 检测更可靠

    参考：[#828](https://www.sqlalchemy.org/trac/ticket/828)

+   **[方言]**

    将 Dialect 属性 ‘preexecute_sequences’ 重命名为 ‘preexecute_pk_sequences’。针对使用旧名称的 out-of-tree 方言，已放置了属性代理。

+   **[方言]**

    为未知类型反射添加了测试覆盖。修复了 sqlite/mysql 处理未知类型反射的问题。

+   **[方言]**

    为 mysql 方言添加了 REAL（供利用 REAL_AS_FLOAT sql 模式的人使用）。

+   **[方言]**

    mysql Float、MSFloat 和 MSDouble 现在构造时不带参数将产生无参数的 DDL，例如 'FLOAT'。

+   **[杂项]**

    删除了未使用的 util.hash()。

## 0.4.0

发布日期：2007 年 10 月 17 日

+   **[无标签]**

    （请参阅 0.4.0beta1，了解对 0.3 的主要更改的开始，以及 [`www.sqlalchemy.org/trac/wiki/WhatsNewIn04`](https://www.sqlalchemy.org/trac/wiki/WhatsNewIn04)）

+   **[无标签]**

    添加了初始的 Sybase 支持（目前仅支持 mxODBC）

    参考：[#785](https://www.sqlalchemy.org/trac/ticket/785)

+   **[无标签]**

    为 PostgreSQL 添加了部分索引支持。在索引上使用 postgres_where 关键字。

+   **[无标签]**

    基于字符串的查询参数解析/配置文件解析器理解更广泛范围的字符串值作为布尔值

    参考：[#817](https://www.sqlalchemy.org/trac/ticket/817)

+   **[无标签]**

    如果另一侧集合不包含项目，则 backref 删除对象操作不会失败，支持 noload 集合

    参考：[#813](https://www.sqlalchemy.org/trac/ticket/813)

+   **[无标签]**

    从“动态”集合中删除了 __len__，因为这将需要发出 SQL“count()”操作，从而迫使所有列表评估发出冗余的 SQL。

    参考：[#818](https://www.sqlalchemy.org/trac/ticket/818)

+   **[无标签]**

    添加了内联优化以定位 locate_dirty()，这可以大大加快对 flush()的重复调用，如 autoflush=True 时发生的情况。

    参考：[#816](https://www.sqlalchemy.org/trac/ticket/816)

+   **[无标签]**

    IdentifierPreprarer 的 _requires_quotes 测试现在基于正则表达式。任何提供自定义合法字符集或非法初始字符集的树外方言都需要转移到正则表达式或覆盖 _requires_quotes。

+   **[无标签]**

    Firebird 的 supports_sane_rowcount 和 supports_sane_multi_rowcount 设置为 False，因为票号#370（正确方式）。

+   **[无标签]**

    在 Firebird 反射上的改进和修复：

    +   FBDialect 现在模仿 OracleDialect，关于 TABLE 和 COLUMN 名称的大小写敏感性（请参见本文件中的“case_sensitive remotion”主题）。

    +   FBDialect.table_names()不会带来系统表（票号：796）。

    +   FB 现在正确反映了 Column 的 nullable 属性。

+   **[无标签]**

    修复了 SQL 编译器对结果集处理中使用的顶级列标签的意识；包含相同列名的嵌套选择不会影响结果或与结果列元数据冲突。

+   **[无标签]**

    query.get()和相关函数（如一对多的延迟加载）使用编译时别名绑定参数名称，以防止与已存在于映射可选择项中的绑定参数发生名称冲突。

+   **[无标签]**

    修复了三级和多级选择以及延迟继承加载（即没有 select_table 的 abc 继承）。

    参考：[#795](https://www.sqlalchemy.org/trac/ticket/795)

+   **[无标签]**

    传递给 shard.py 中的 id_chooser 的标识符始终是一个列表。

+   **[无标签]**

    无参数 ResultProxy._row_processor()现在是类属性 _process_row。

+   **[无标签]**

    增加了对从 PostgreSQL 8.2+插入和更新返回值的支持。

    参考：[#797](https://www.sqlalchemy.org/trac/ticket/797)

+   **[无标签]**

    PG 反射，在看到默认模式名称被明确用作表中的“模式”参数时，将假定这是用户期望的约定，并将在外键相关的反射表中明确设置“模式”参数，从而使它们仅与也使用显式“模式”参数的 Table 构造函数匹配（即使其为默认模式）。换句话说，SA 假定用户在此使用中是一致的。

+   **[无标签]**

    修复了 BOOL/BOOLEAN 的 sqlite 反射

    参考：[#808](https://www.sqlalchemy.org/trac/ticket/808)

+   **[无标签]**

    在 mysql 上添加了对带有 LIMIT 的 UPDATE 的支持。

+   **[无标签]**

    m2o 上的空外键不会触发延迟加载

    引用：[#803](https://www.sqlalchemy.org/trac/ticket/803)

+   **[无标签]**

    Oracle 不会隐式转换为 unicode 以用于非类型化结果集（即当没有使用 TypeEngine/String/Unicode 类型时；以前它会检测 DBAPI 类型并进行转换）。应该修复

    引用：[#800](https://www.sqlalchemy.org/trac/ticket/800)

+   **[无标签]**

    修复了对长表/列名称的匿名标签生成

    引用：[#806](https://www.sqlalchemy.org/trac/ticket/806)

+   **[无标签]**

    Firebird 方言现在使用 SingletonThreadPool 作为池类。

+   **[无标签]**

    Firebird 现在使用 dialect.preparer 格式化序列名称

+   **[无标签]**

    修复了 postgres 和多个两阶段事务之间的故障。两阶段提交和回滚不会像通常的 dbapi 提交/回滚一样自动结束一个新事务。

    引用：[#810](https://www.sqlalchemy.org/trac/ticket/810)

+   **[无标签]**

    向 _ScopedExt 映射器扩展添加了一个选项，以便在对象初始化时不自动将新对象保存到会话中。

+   **[无标签]**

    修复了 Oracle 非 ANSI 连接语法

+   **[无标签]**

    PickleType 和 Interval 类型（在不原生支持它的数据库上）现在稍微更快。

+   **[无标签]**

    在 Firebird 中添加了 Float 和 Time 类型（FBFloat 和 FBTime）。修复了 TEXT 和 Binary 类型的 BLOB SUB_TYPE。

+   **[无标签]**

    更改了 in_ 运算符的 API。现在 in_() 接受一个参数，该参数是一个值序列或可选择的。传递值作为可变参数的旧 API 仍然有效，但已被弃用。

## 0.4.0beta6

发布日期：2007 年 9 月 27 日星期四

+   **[无标签]**

    会话标识映射现在默认为*弱引用*，使用 weak_identity_map=False 来使用常规字典。我们正在使用的弱字典被定制为检测“脏”实例，并保持对这些实例的临时强引用，直到更改被刷新。

+   **[无标签]**

    Mapper 编译已重新组织，大部分编译发生在映射器构造时。这使我们可以减少对 mapper.compile() 的调用，并且还允许基于类的属性强制进行编译（即 User.addresses == 7 将编译所有映射器；这是）。唯一的注意事项是，现在继承映射器在构造时会查找其继承的映射器；因此，继承关系中的映射器需要按照继承顺序进行构造（这应该是正常情况）。

    引用：[#758](https://www.sqlalchemy.org/trac/ticket/758)

+   **[无标签]**

    在 Postgres 中检测到的关键字中添加了“FETCH”，以指示一个包含结果行的语句（即除了“SELECT”之外）。

+   **[无标签]**

    添加了 SQLite 保留关键字的完整列表，以便正确转义它们。

+   **[无标签]**

    加强了 Query 生成“eager load”别名与 Query.instances()之间的关系，Query.instances()实际上获取了被急加载的行。如果别名不是由 EagerLoader 专门为该语句生成的，则在获取行时 EagerLoader 将不起作用。这可以防止列被意外地抓取为急加载的一部分，当它们不是为此目的而设计时，这可能会发生在文本 SQL 以及一些继承情况下。这一点尤为重要，因为“匿名别名”现在使用简单的整数计数来生成标签。

+   **[无标签]**

    从 clauseelement.compile()中删除了“parameters”参数，替换为“column_keys”。传递给 execute()的参数仅与插入/更新语句的编译过程中存在的列名交互，而不涉及这些列的值。这样可以产生更一致的 execute/executemany 行为，内部简化了一些事情。

+   **[无标签]**

    在 PickleType 中添加了‘comparator’关键字参数。默认情况下，“mutable” PickleType 使用它们的 dumps()表示进行对象的“深度比较”。但这对于字典不起作用。提供适当 __eq__()实现的 Pickled 对象可以设置为“PickleType(comparator=operator.eq)”

    参考：[#560](https://www.sqlalchemy.org/trac/ticket/560)

+   **[无标签]**

    添加了 session.is_modified(obj)方法；执行与 flush 操作中发生的“历史”比较操作相同；设置 include_collections=False 会得到与 flush 确定是否为实例的行发出 UPDATE 相同的结果。

+   **[无标签]**

    在 Sequence 中添加了“schema”参数；在 Postgres/Oracle 中，当序列位于替代模式中时，请使用此参数。实现部分，应该修复。

    参考：[#584](https://www.sqlalchemy.org/trac/ticket/584), [#761](https://www.sqlalchemy.org/trac/ticket/761)

+   **[无标签]**

    修复了 mysql 枚举的空字符串反射。

+   **[无标签]**

    将 MySQL 方言更改为使用旧的 LIMIT <offset>, <limit>语法，而不是对于使用 3.23 的人使用 LIMIT <l> OFFSET <o>。

    参考：[#794](https://www.sqlalchemy.org/trac/ticket/794)

+   **[无标签]**

    向 relation()添加了‘passive_deletes=”all”’标志，禁用了在删除父对象时刷新期间所有外键属性的置空。

+   **[无标签]**

    列默认值和 onupdates，在执行内联时，将为子查询和其他需要括号的表达式添加括号。

+   **[无标签]**

    关于 String/Unicode 类型的行为，当没有长度时它们自动转换为 TEXT/CLOB 现在仅适用于没有参数的 String 或 Unicode 精确类型。如果您使用没有长度的 VARCHAR 或 NCHAR（String/Unicode 的子类），它们将被方言解释为 VARCHAR/NCHAR；在那里不会发生“魔术”转换。这是更少令人惊讶的行为，特别是这有助于 Oracle 将基于字符串的绑定参数保持为 VARCHAR 而不是 CLOB。

    参考：[#793](https://www.sqlalchemy.org/trac/ticket/793)

+   **[无标签]**

    修复了 ShardedSession 与延迟列一起工作的问题。

    参考：[#771](https://www.sqlalchemy.org/trac/ticket/771)

+   **[无标签]**

    用户定义的 shard_chooser() 函数必须接受“clause=None”参数；这是传递给 session.execute(statement) 的 ClauseElement，并可用于确定正确的分片 id（因为 execute() 不接受实例）。

+   **[无标签]**

    调整了 NOT 运算符的优先级，以匹配 '==' 和其他运算符，因此 ~(x <operator> y) 会产生 NOT (x <op> y)，这与旧版 MySQL 更兼容。这不适用于“~(x==y)”如同在 0.3 版本中一样，因为 ~(x==y) 编译为“x != y”，但仍适用于像 BETWEEN 这样的运算符。

    参考：[#764](https://www.sqlalchemy.org/trac/ticket/764)

+   **[无标签]**

    其他票据:,,.

    参考：[#728](https://www.sqlalchemy.org/trac/ticket/728), [#757](https://www.sqlalchemy.org/trac/ticket/757), [#768](https://www.sqlalchemy.org/trac/ticket/768), [#779](https://www.sqlalchemy.org/trac/ticket/779)

## 0.4.0beta5

无发布日期

+   **[无标签]**

    连接池修复；beta4 的更好性能仍然存在，但修复了“连接溢出”和其他存在的 bug（例如）。

    参考：[#754](https://www.sqlalchemy.org/trac/ticket/754)

+   **[无标签]**

    修复了从自定义继承条件确定正确同步子句的错误。

    参考：[#769](https://www.sqlalchemy.org/trac/ticket/769)

+   **[无标签]**

    扩展了对 QueuePool 大小/溢出的 'engine_from_config' 强制转换。

    参考：[#763](https://www.sqlalchemy.org/trac/ticket/763)

+   **[无标签]**

    mysql 视图可以再次反射。

    参考：[#748](https://www.sqlalchemy.org/trac/ticket/748)

+   **[无标签]**

    AssociationProxy 现在可以使用自定义的 getter 和 setter。

+   **[无标签]**

    修复了 orm 查询中 BETWEEN 的故障。

+   **[无标签]**

    修复了 OrderedProperties 的 pickling

    参考：[#762](https://www.sqlalchemy.org/trac/ticket/762)

+   **[无标签]**

    SQL 表达式默认值和序列现在在 INSERT 或 UPDATE 期间对所有非主键列“内联”执行，并且在执行类似 executemany() 的调用期间对所有列执行。在任何 insert/update 语句上的 inline=True 标志也会强制相同的行为与单个 execute()。result.postfetch_cols() 是一个集合，其中包含上一个单个 insert 或 update 语句包含 SQL 端默认表达式的列。

+   **[无标签]**

    修复了 PG executemany() 的行为。

    参考：[#759](https://www.sqlalchemy.org/trac/ticket/759)

+   **[无标签]**

    postgres 反映具有无默认值的主键列的表，autoincrement=False。

+   **[无标签]**

    postgres 不再使用单独的 execute()调用包装 executemany()，而是更倾向于性能。在使用 PG 时，对已删除项目进行“rowcount”/“concurrency”检查（使用 executemany）被禁用，因为 psycopg2 不会为 executemany()报告正确的 rowcount。

+   **[已修复] [票务]**

    参考：[#742](https://www.sqlalchemy.org/trac/ticket/742)

+   **[已修复] [票务]**

    参考：[#748](https://www.sqlalchemy.org/trac/ticket/748)

+   **[已修复] [票务]**

    参考：[#760](https://www.sqlalchemy.org/trac/ticket/760)

+   **[已修复] [票务]**

    参考：[#762](https://www.sqlalchemy.org/trac/ticket/762)

+   **[已修复] [票务]**

    参考：[#763](https://www.sqlalchemy.org/trac/ticket/763)

## 0.4.0beta4

发布日期：2007 年 8 月 22 日（星期三）

+   **[无标签]**

    当您‘from sqlalchemy import *’时，整理了最终进入您命名空间的内容：

+   **[无标签]**

    ‘table’和‘column’不再被导入。它们仍然可以通过直接引用（如‘sql.table’和‘sql.column’）或从 sql 包进行全局导入来使用。在刚开始使用 SQLAlchemy 时，很容易意外使用 sql.expressions.table 而不是 schema.Table，同样也是 column。

+   **[无标签]**

    类似于 ClauseElement、FromClause、NullTypeEngine 等的内部类也不再导入到您的命名空间中

+   **[无标签]**

    ‘Smallinteger’兼容性名称（小写 i！）不再被导入，但目前仍保留在 schema.py 中。SmallInteger（大写 I！）仍然被导入。

+   **[无标签]**

    连接池在内部使用“threadlocal”策略，以返回已绑定到线程的相同连接，用于“上下文”连接；这些连接在执行“无连接”操作时使用，如 insert().execute()。这类似于“threadlocal”引擎策略的“部分”版本，但没有其中的线程本地事务部分。我们希望它能减少连接池的开销以及数据库使用。但是，如果证明对稳定性产生负面影响，我们将立即撤销。

+   **[无标签]**

    修复了绑定参数处理，使“False”值（如空字符串）仍然被处理/编码。

+   **[无标签]**

    修复了 select()的“生成”行为，使调用 column()、select_from()、correlate()和 with_prefix()不会修改原始 select 对象

    参考：[#752](https://www.sqlalchemy.org/trac/ticket/752)

+   **[无标签]**

    添加了一个“legacy”适配器到 types，这样用户定义的 TypeEngine 和 TypeDecorator 类，定义了 convert_bind_param()和/或 convert_result_value()的将继续正常运行。还支持调用这些方法的 super()版本。

+   **[无标签]**

    添加了 session.prune()，清除会话中不再在其他地方引用的实例缓存。（用于强引用标识映射的实用程序）。

+   **[无标签]**

    添加了 Transaction 的 close() 方法。如果是最外层事务，则使用 rollback 结束事务，否则仅结束而不影响外部事务。

+   **[无标签]**

    事务性和非事务性 Session 与绑定连接更好地集成；close() 将确保连接的事务状态与绑定到 Session 前存在的状态相同。

+   **[无标签]**

    修改了 SQL 操作函数为模块级别的操作符，允许 SQL 表达式可被 pickle。

    参考：[#735](https://www.sqlalchemy.org/trac/ticket/735)

+   **[无标签]**

    对 mapper 类的 __init__ 进行了小的调整，以允许 Py2.6 object.__init__() 行为。

+   **[无标签]**

    修复了 select() 的 ‘prefix’ 参数

+   **[无标签]**

    Connection.begin() 不再接受 nested=True，这个逻辑现在都在 begin_nested() 中。

+   **[无标签]**

    修复了新的“动态”关系加载器涉及级联的问题

+   **[修复] [问题]**

    参考：[#735](https://www.sqlalchemy.org/trac/ticket/735)

+   **[修复] [问题]**

    参考：[#752](https://www.sqlalchemy.org/trac/ticket/752)

## 0.4.0beta3

发布日期：2007 年 8 月 16 日 星期四

+   **[无标签]**

    SQL 类型优化：

+   **[无标签]**

    新的性能测试显示，与 0.3 版本相比，组合的大规模插入/选择测试的函数调用减少了 68%。

+   **[无标签]**

    结果集迭代的一般性能提升约为 10-20%。

+   **[无标签]**

    在 types.AbstractType 中，convert_bind_param() 和 convert_result_value() 已迁移到返回可调用的 bind_processor() 和 result_processor() 方法。如果没有返回可调用函数，则不会调用预处理/后处理函数。

+   **[无标签]**

    在 base/sql/defaults 中添加了钩子以优化调用 bind 参数/结果处理器的性能，以减少方法调用开销。

+   **[无标签]**

    添加了对 executemany() 场景的支持，以避免不必要的��最后一行 id”逻辑，参数不会被过度遍历。

+   **[无标签]**

    向 mapper() 添加了 ‘inherit_foreign_keys’ 参数。

+   **[无标签]**

    在 sqlite 中添加了对字符串日期透传的支持。

+   **[修复] [问题]**

    参考：[#738](https://www.sqlalchemy.org/trac/ticket/738)

+   **[修复] [问题]**

    参考：[#739](https://www.sqlalchemy.org/trac/ticket/739)

+   **[修复] [问题]**

    参考：[#743](https://www.sqlalchemy.org/trac/ticket/743)

+   **[修复] [问题]**

    参考：[#744](https://www.sqlalchemy.org/trac/ticket/744)

## 0.4.0beta2

发布日期：2007 年 8 月 14 日 星期二

### oracle

+   **[oracle] [改进]**

    在 mysql 中的 LOAD DATA INFILE 后自动提交。

+   **[oracle] [改进]**

    添加了 rudimental SessionExtension 类，允许在 flush()、commit() 和 rollback() 边界处进行用户定义功能。

+   **[oracle] [改进]**

    添加了 engine_from_config() 函数，以帮助从 .ini 风格的配置文件创建 create_engine()。

+   **[oracle] [改进]**

    base_mapper() 变为普通属性。

+   **[oracle] [改进]**

    session.execute() 和 scalar() 现在可以通过给定的 ClauseElement 搜索要绑定的表。

+   **[oracle] [improvements.]**

    Session 自动从具有绑定的映射器中推断表，还使用 base_mapper，以便继承层次结构自动绑定。

+   **[oracle] [improvements.]**

    将 ClauseVisitor 遍历移回到内联的非递归方式。

### 杂项

+   **[fixed] [tickets]**

    参考：[#730](https://www.sqlalchemy.org/trac/ticket/730)

+   **[fixed] [tickets]**

    参考：[#732](https://www.sqlalchemy.org/trac/ticket/732)

+   **[fixed] [tickets]**

    参考：[#733](https://www.sqlalchemy.org/trac/ticket/733)

+   **[fixed] [tickets]**

    参考：[#734](https://www.sqlalchemy.org/trac/ticket/734)

### oracle

+   **[oracle] [improvements.]**

    mysql 的 LOAD DATA INFILE 后自动提交。

+   **[oracle] [improvements.]**

    添加了一个基本的 SessionExtension 类，允许在 flush()、commit() 和 rollback() 边界处发生用户定义的功能。

+   **[oracle] [improvements.]**

    添加了 engine_from_config() 函数，以帮助从 .ini 样式配置创建 create_engine()。

+   **[oracle] [improvements.]**

    base_mapper() 变为普通属性。

+   **[oracle] [improvements.]**

    session.execute() 和 scalar() 现在可以通过给定的 ClauseElement 搜索要绑定的表。

+   **[oracle] [improvements.]**

    Session 自动从具有绑定的映射器中推断表，还使用 base_mapper，以便继承层次结构自动绑定。

+   **[oracle] [improvements.]**

    将 ClauseVisitor 遍历移回到内联的非递归方式。

### 杂项

+   **[fixed] [tickets]**

    参考：[#730](https://www.sqlalchemy.org/trac/ticket/730)

+   **[fixed] [tickets]**

    参考：[#732](https://www.sqlalchemy.org/trac/ticket/732)

+   **[fixed] [tickets]**

    参考：[#733](https://www.sqlalchemy.org/trac/ticket/733)

+   **[fixed] [tickets]**

    参考：[#734](https://www.sqlalchemy.org/trac/ticket/734)

## 0.4.0beta1

发布日期：2007 年 8 月 12 日 星期日

### orm

+   **[orm]**

    速度！除了对 ResultProxy 的最近加速，大负载的函数调用总数显著减少。

+   **[orm]**

    test/perf/masseagerload.py 报告称 0.4 版本在所有 SA 版本（0.1、0.2 和 0.3）中具有最少的函数调用次数。

+   **[orm]**

    新的 collection_class api 和实现。现在通过装饰而不是代理来对集合进行检测。现在可以有管理自身成员的集合，并且你的类实例将直接暴露在关系属性上。这些变化对大多数用户来说是透明的。

    参考：[#213](https://www.sqlalchemy.org/trac/ticket/213)

+   **[orm]**

    InstrumentedList（如之前所述）已被移除，关系属性不再具有‘clear()’、‘.data’或任何其他由集合类型提供之外的附加方法。当然，你可以将它们添加到自定义类中。

+   **[orm]**

    __setitem__-like 分配现在会为现有值触发删除事件（如果有）。

+   **[orm]**

    作为集合类使用的类似字典的对象不再需要更改 __iter__ 语义- 默认使用 itervalues()。这是一个不兼容的更改。

+   **[orm]**

    对于映射集合，不再需要为其子类化 dict。orm.collections 提供了按指定列或自定义函数键入对象的预制实现。

+   **[orm]**

    现在集合赋值需要兼容的类型- 将 None 分配给清除集合或将列表分配给字典集合现在会引发参数错误。

+   **[orm]**

    AttributeExtension 移动到接口，并且.delete 现在是.remove 事件方法签名也已经交换。

+   **[orm]**

    Query 的重大改革：

+   **[orm]**

    所有 selectXXX 方法已被弃用。生成方法现在是标准操作方式，即 filter()，filter_by()，all()，one()等。弃用的方法在其新替代品中有文档字符串。

+   **[orm]**

    类级属性现在可以用作查询元素…不再需要‘.c.’！“Class.c.propname”现在被“Class.propname”取代。支持所有子句操作符，以及高级操作符，如标量属性的 Class.prop==<some instance>，基于集合的属性的 Class.prop.contains(<some instance>)和 Class.prop.any(<some expression>)（所有这些也是可否定的）。当然，基于表的列表达式以及通过‘c’挂载在映射类上的列仍然完全可用，并且可以自由与新属性混合使用。

    参考：[#643](https://www.sqlalchemy.org/trac/ticket/643)

+   **[orm]**

    移除了古老的 query.select_by_attributename()功能。

+   **[orm]**

    急加载使用的别名逻辑已经泛化，因此它还为 Query 添加了完全自动的别名支持。不再需要为多次加入相同表创建显式别名；*即使是自引用关系*。

    +   join()和 outerjoin()接受参数“aliased=True”。这将导致它们的加入建立在别名表上；对 filter()和 filter_by()的后续调用将把所有表达式（是的，使用原始映射表的真实表达式）翻译为别名的表，直到 join()被重置或另一个 join()被调用。

    +   join()和 outerjoin()接受参数“id=<somestring>”。当与“aliased=True”一起使用时，可以通过 add_entity(cls, id=<somestring>)引用 id，以便选择加入的实例，即使它们来自别名。

    +   join()和 outerjoin()现在适用于自引用关系！使用“aliased=True”，可以加入任意深度的级别，即 query.join(['children', 'children'], aliased=True); 过滤条件将针对最右侧的加入表

+   **[orm]**

    添加了 query.populate_existing()，标记查询以重新加载查询中触及的所有实例的所有属性和集合，包括急加载的实体。

    参考：[#660](https://www.sqlalchemy.org/trac/ticket/660)

+   **[orm]**

    添加了 eagerload_all()，允许 eagerload_all(‘x.y.z’)指定给定路径中所有属性的急切加载。

+   **[orm]**

    Session 进行了重大改革：

+   **[orm]**

    新函数“configures���一个会话，称为“sessionmaker()”。一次向该函数发送各种关键字参数，返回一个根据该原型创建会话的新类。

+   **[orm]**

    从“public”API 中移除了 SessionTransaction。现在可以在 Session 本身上调用 begin()/commit()/rollback()。

+   **[orm]**

    Session 还支持 SAVEPOINT 事务；调用 begin_nested()。

+   **[orm]**

    当垂直或水平分区（即使用多个引擎）时，Session 支持两阶段提交行为。使用 twophase=True。

+   **[orm]**

    Session 标志“transactional=True”会产生一个会话，当首次使用时总是将自己放入事务中。在 commit()、rollback()或 close()时，事务结束；但在下一次使用时重新开始。

+   **[orm]**

    Session 支持“autoflush=True”。这会在每次查询之前发出一个 flush()。与 transactional 一起使用，您可以只需 save()/update()然后查询，新对象就会出现。在最后使用 commit()（或者如果不是事务性的话使用 flush()）来刷新剩余的更改。

+   **[orm]**

    新的 scoped_session()函数取代了 SessionContext 和 assignmapper。构建在“sessionmaker()”概念之上，以生成一个类，其 Session()构造返回线程本地会话。或者，将所有 Session 方法作为类方法调用，例如 Session.save(foo); Session.commit()。就像旧的“objectstore”时代一样。

+   **[orm]**

    Session 新增了“binds”参数，以支持使用 sessionmaker()函数配置多个绑定。

+   **[orm]**

    添加了一个基本的 SessionExtension 类，允许在 flush()、commit()和 rollback()边界处进行用户定义的功能。

+   **[orm]**

    基于查询的 relation()现在可以使用 dynamic_loader()。这是一个*writable*集合（支持 append()和 remove()），在读取时也是一个活动的 Query 对象。适用于处理非常大的集合，只需部分加载。

+   **[orm]**

    flush()-内嵌的 INSERT/UPDATE 表达式。将任何 SQL 表达式，如“sometable.c.column + 1”，分配给实例的属性。在 flush()时，映射器检测到表达式并直接嵌入到 INSERT 或 UPDATE 语句中；属性在实例上被延迟，所以在下次访问时加载新值。

+   **[orm]**

    引入了一个基本的分片（水平扩展）系统。该系统使用修改后的 Session，可以根据用户定义的“分片策略”在多个数据库之间分发读写操作。实例及其依赖项可以根据属性值、轮询方法或任何其他用户定义的系统在多个数据库之间分发和查询。

    参考：[#618](https://www.sqlalchemy.org/trac/ticket/618)

+   **[orm]**

    急加载已经增强，允许在更多地方进行更多的连接。现在它可以在自引用和循环结构的任意深度处运行。在加载循环结构时，在 relation()上指定“join_depth”表示您希望表自连接多少次；每个级别都会得到一个不同的表别名。别名名称现在是在编译时使用简单的计数方案生成的，更容易阅读，当然完全确定性。

    参考：[#659](https://www.sqlalchemy.org/trac/ticket/659)

+   **[orm]**

    添加了复合列属性。这允许您在使用 ORM 时创建由多个列表示的类型。新类型的对象在查询表达式、比较、query.get()子句等方面都是完全功能的，并且表现得就像是常规的单列标量... 除了它们不是！在映射器的“properties”字典中使用函数 composite(cls, *columns)，并且 cls 的实例将被创建/映射到一个由*columns 对应的值组成的单个属性。

    参考：[#211](https://www.sqlalchemy.org/trac/ticket/211)

+   **[orm]**

    改进了对具有相关子查询的自定义 column_property()属性的支持，现在在使用急加载时效果更好。

+   **[orm]**

    主键“折叠”行为；映射器将分析其给定可选择的所有列，以获取主键“等效性”，即通过外键关系或显式 inherit_condition 等价的列。主要用于连接表继承方案，其中继承表中的不同命名的 PK 列应“折叠”为单值（或更少值）主键。修复了诸如此类的问题。

    参考：[#611](https://www.sqlalchemy.org/trac/ticket/611)

+   **[orm]**

    连接表继承现在将生成所有继承类的主键列针对连接的根表。这意味着根表中的每一行对应一个实例。如果出于某种罕见原因这不是理想的情况，单独映射器上的显式 primary_key 设置将覆盖它。

+   **[orm]**

    当使用“polymorphic”标志与连接表或单表继承时，所有标识键都针对继承层次结构的根类生成；这允许 query.get()以与非多态 get 相同的缓存语义多态地工作。请注意，这目前不适用于具体继承。

+   **[orm]**

    次要继承加载：可以构建没有 select_table 参数的多态映射器。继承映射器的表在初始加载中未表示将立即发出第二个 SQL 查询，每个实例一次（即对于大型列表来说效率不高），以加载其余列。

+   **[orm]**

    次要继承加载也可以将其第二个查询移动到列级“延迟”加载，通过“polymorphic_fetch”参数，可以设置为“select”或“deferred”。

+   **[orm]**

    现在可以仅将可用选择列的子集映射到映射器属性中，使用 include_columns/exclude_columns。

    参考：[#696](https://www.sqlalchemy.org/trac/ticket/696)

+   **[orm]**

    添加了 undefer_group() MapperOption，设置一组由“group”连接的“延迟”列以作为“未延迟”加载。

+   **[orm]**

    重写了“确定性别名”逻辑，使其成为 SQL 层的一部分，生成更简单的别名和标签名称，更符合 Hibernate 的风格。

### sql

+   **[sql]**

    速度！子句编译以及 SQL 构造的机制已经被简化和优化到相当程度，使语句构造/编译的开销提高了 20-30%。

+   **[sql]**

    所有“type”关键字参数，如 bindparam()、column()、Column()和 func.<something>()中的参数，重命名为“type_”。这些对象仍然将它们的“type”属性命名为“type”。

+   **[sql]**

    从模式项中移除了 case_sensitive=(True|False)设置，因为检查此状态会增加很多方法调用开销，而且从来没有合理的理由将其设置为 False。所有小写的表和列名称将被视为不区分大小写（是的，我们也适应了 Oracle 的大写风格）。

### 扩展

+   **[扩展]**

    proxyengine 暂时移除，等待一个真正有效的替代品。

+   **[扩展]**

    SelectResults 已被 Query 取代。SelectResults / SelectResultsExt 仍然存在，但只返回一个稍微修改的 Query 对象以保持向后兼容性。SelectResults 的 join_to()方法不再存在，需要使用 join()。

### mysql

+   **[mysql]**

    通过反射加载的表和列名称现在是 Unicode。

+   **[mysql]**

    现在支持所有标准列类型，包括 SET。

+   **[mysql]**

    表反射现在可以在一次往返中完成。

+   **[mysql]**

    现在支持 ANSI 和 ANSI_QUOTES SQL 模式。

+   **[mysql]**

    索引现在也被反射。

### oracle

+   **[oracle]**

    添加了对 OUT 参数的非常基本的支持；使用 sql.outparam(name, type)设置一个 OUT 参数，就像 bindparam()一样；执行后，值可以通过 result.out_parameters 字典获得。

    参考：[#507](https://www.sqlalchemy.org/trac/ticket/507)

### 杂项

+   **[事务]**

    为事务添加了上下文管理器（with 语句）支持。

+   **[transactions]**

    添加了对两阶段提交的支持，目前与 mysql 和 postgres 一起使用。

+   **[事务]**

    添加了一个使用保存点的子事务实现。

+   **[事务]**

    添加了对保存点的支持。

+   **[元数据]**

    可以在不事先声明的情况下从数据库中批量反射表。MetaData(engine, reflect=True)将加载数据库中存在的所有表，或者使用 metadata.reflect()进行更精细的控制。

+   **[元数据]**

    DynamicMetaData 已重命名为 ThreadLocalMetaData

+   **[元数据]**

    ThreadLocalMetaData 构造函数现在不接受参数。

+   **[元数据]**

    BoundMetaData 已被移除- 普通 MetaData 等效。

+   **[元数据]**

    Numeric 和 Float 类型现在具有一个“asdecimal”标志；对于 Numeric，默认为 True，对于 Float，默认为 False。当为 True 时，值以 decimal.Decimal 对象返回；当为 False 时，值以 float() 返回。True/False 的默认值已经是 PG 和 MySQL 的 DBAPI 模块的行为。

    参考：[#646](https://www.sqlalchemy.org/trac/ticket/646)

+   **[元数据]**

    新的 SQL 操作符实现将所有硬编码的操作符从表达式结构中移除，并将它们移入编译中；这样可以更灵活地编译操作符；例如，在字符串上下文中使用“+”编译为“||”，在 MySQL 上则编译为“concat(a,b)”；而在数值上下文中则编译为“+”。修复。

    参考：[#475](https://www.sqlalchemy.org/trac/ticket/475)

+   **[元数据]**

    ”匿名“别名和标签名称现在在 SQL 编译时以完全确定性的方式生成... 不再是随机的十六进制 ID

+   **[元数据]**

    对 SQL 元素（ClauseElement）进行了重大的架构改造。所有元素共享一个通用的“可变性”框架，允许对元素进行一致的就地修改以及生成式行为。提高了 ORM 的稳定性，该 ORM 对 SQL 表达式进行了大量的变异。

+   **[元数据]**

    select() 和 union() 现在具有“生成式”行为。像 order_by() 和 group_by() 这样的方法返回一个 *新的* 实例 - 原始实例保持不变。非生成式方法也保持不变。

+   **[元数据]**

    select/union 的内部大幅简化- 关于“是否子查询”和“相关性”的所有决策都推迟到 SQL 生成阶段。select() 元素现在 *永远* 不会被其封闭容器或任何方言的编译过程改变。

    参考：[#52](https://www.sqlalchemy.org/trac/ticket/52)，[#569](https://www.sqlalchemy.org/trac/ticket/569)

+   **[元数据]**

    select(scalar=True) 参数已被弃用；使用 select(..).as_scalar()。结果对象遵循完整的“列”接口，并在表达式中更好地发挥作用。

+   **[元数据]**

    添加了 select().with_prefix('foo')，允许在 SELECT 的列子句之前放置任何一组关键字

    参考：[#504](https://www.sqlalchemy.org/trac/ticket/504)

+   **[元数据]**

    向 row[<index>] 添加了数组切片支持

    参考：[#686](https://www.sqlalchemy.org/trac/ticket/686)

+   **[元数据]**

    结果集现在更好地尝试将游标描述中存在的 DBAPI 类型与方言定义的 TypeEngine 对象进行匹配，然后用于结果处理。请注意，这仅对文本 SQL 有效；构造的 SQL 语句始终具有显式类型映射。

+   **[元数据]**

    CRUD 操作的结果集立即关闭其底层游标，并且如果为操作定义了，还将自动关闭连接；这允许更有效地使用连接来进行连续的 CRUD 操作，减少“悬挂连接”的机会。

+   **[元数据]**

    列默认值和 onupdate Python 函数（即传递给 ColumnDefault 的函数）可能需要零个或一个参数；一个参数是 ExecutionContext，您可以从中调用 “context.parameters[someparam]” 来访问附加到语句的其他绑定参数值。用于执行的连接也可用，以便您可以预先执行语句。

    参考：[#559](https://www.sqlalchemy.org/trac/ticket/559)

+   **[元数据]**

    为序列添加了“显式”创建/删除/执行支持（即您可以将“connectable”传递给 Sequence 上的每个方法）。

+   **[元数据]**

    在操作模式时更好地引用标识符。

+   **[元数据]**

    标准化了表反射的行为，其中类型无法定位时将替换为 NullType，并引发警告。

+   **[元数据]**

    ColumnCollection（即表上的 'c' 属性）遵循“__contains__”的字典语义

    参考：[#606](https://www.sqlalchemy.org/trac/ticket/606)

+   **[引擎]**

    速度！结果处理和绑定参数处理的机制已进行了全面改进、简化和优化，以尽可能少地发出方法调用。批量插入和大量行集迭代的性能测试都显示 0.4 比 0.3 快两倍以上，并且函数调用减少了 68%。

+   **[引擎]**

    现在您可以钩入池的生命周期，并在每个新的 DBAPI 连接、池检出和池检入时运行 SQL 语句或其他逻辑。

+   **[引擎]**

    连接获得了 `.properties` 集合，其中的内容在基础 DBAPI 连接的生命周期范围内。

+   **[引擎]**

    从 Pool 中删除了 `auto_close_cursors` 和 `disallow_open_cursors` 参数；由于游标通常由 ResultProxy 和 Connection 关闭，因此减少了开销。

+   **[postgres]**

    添加了 PGArray 数据类型以使用 PostgreSQL 数组数据类型。

### ORM

+   **[ORM]**

    速度！除了对 ResultProxy 的最近加速之外，对大型加载的函数调用总数显著减少。

+   **[ORM]**

    `test/perf/masseagerload.py` 报告称，0.4 在所有 SA 版本（0.1、0.2 和 0.3）中的函数调用次数最少。

+   **[ORM]**

    新的 `collection_class` API 和实现。现在通过装饰来实现对集合的检测，而不是代理。现在您可以拥有管理自己成员的集合，并且您的类实例将直接暴露在关系属性上。这些更改对大多数用户是透明的。

    参考：[#213](https://www.sqlalchemy.org/trac/ticket/213)

+   **[ORM]**

    InstrumentedList（如之前的情况）已移除，并且关系属性不再具有‘clear()’、‘.data’或任何其他添加的方法，除了集合类型提供的方法。当然，您可以将它们添加到自定义类中。

+   **[ORM]**

    类似 __setitem__ 的赋值现在会为现有值触发删除事件（如果有）。

+   **[orm]**

    作为集合类使用的类似字典的对象不再需要更改 __iter__ 语义- 默认情况下使用 itervalues()。这是一个不兼容的变化。

+   **[orm]**

    在大多数情况下，不再需要为映射的集合子类化 dict。orm.collections 提供了按指定列或自定义函数键入对象的预制实现。

+   **[orm]**

    现在集合赋值需要兼容的类型-将 None 分配给清除集合或将列表分配给字典集合现在将引发参数错误。

+   **[orm]**

    AttributeExtension 移动到接口，并且.delete 现在是.remove 事件方法签名也已经交换。

+   **[orm]**

    Query 进行了重大改进：

+   **[orm]**

    所有 selectXXX 方法都已弃用。生成方法现在是执行操作的标准方式，即 filter()，filter_by()，all()，one()等。弃用的方法在其新替代品的文档字符串中有说明。

+   **[orm]**

    现在可以将类级属性用作查询元素...不再需要‘.c.’！“Class.c.propname”现在被“Class.propname”取代。支持所有子句操作符，以及更高级别的操作符，如标量属性的 Class.prop==<some instance>，基于集合的属性的 Class.prop.contains(<some instance>)和 Class.prop.any(<some expression>)（所有这些也是可否定的）。当然，基于表的列表达式以及通过‘c’挂载在映射类上的列仍然完全可用，并且可以自由地与新属性混合使用。

    参考：[#643](https://www.sqlalchemy.org/trac/ticket/643)

+   **[orm]**

    移除了古老的 query.select_by_attributename()功能。

+   **[orm]**

    急加载使用的别名逻辑已经泛化，因此它还为 Query 添加了完全自动的别名支持。不再需要为多次加入相同表创建显式别名；*即使是自引用关系也是如此*。

    +   join()和 outerjoin()接受参数“aliased=True”。这将导致它们的连接建立在别名表上；随后对 filter()和 filter_by()的调用将把所有表达式（是的，使用原始映射表的实际表达式）翻译为别名的表达式，直到该 join()结束（即直到 reset_joinpoint()或另一个 join()被调用）。

    +   join()和 outerjoin()接受参数“id=<somestring>”。当与“aliased=True”一起使用时，可以通过 add_entity(cls, id=<somestring>)引用 id，以便即使它们来自别名，也可以选择加入的实例。

    +   join()和 outerjoin()现在可以与自引用关系一起使用！使用“aliased=True”，您可以加入任意深度的级别，即 query.join([‘children’, ‘children’], aliased=True)；过滤条件将针对最右边的连接表

+   **[orm]**

    添加了 query.populate_existing()，标记查询以重新加载查询中触及的所有实例的所有属性和集合，包括急切加载的实体。

    参考：[#660](https://www.sqlalchemy.org/trac/ticket/660)

+   **[orm]**

    添加了 eagerload_all()，允许 eagerload_all('x.y.z') 来指定在给定路径中所有属性的急切加载。

+   **[orm]**

    会话的重大改革：

+   **[orm]**

    新函数“配置”一个会话称为“sessionmaker()”。向该函数发送各种关键字参数一次，返回一个新类，该类对该原型创建一个会话。

+   **[orm]**

    会话事务从“public” API 中移除。现在你可以在会话本身上调用 begin()/commit()/rollback()。

+   **[orm]**

    会话还支持 SAVEPOINT 事务；调用 begin_nested()。

+   **[orm]**

    当纵向或横向分区（即使用多个引擎）时，会话支持两阶段提交行为。使用 twophase=True。

+   **[orm]**

    会话标志“transactional=True”生成一个会话，当首次使用时总是将自身置于事务中。在 commit()、rollback() 或 close() 时，事务结束；但在下次使用时重新开始。

+   **[orm]**

    会话支持“autoflush=True”。这会在每次查询之前发出 flush()。与 transactional 结合使用，你可以仅保存()/更新() 然后查询，新对象就会出现。在最后使用 commit()（或者如果不是在事务中使用 flush()）来刷新剩余的更改。

+   **[orm]**

    新的 scoped_session() 函数替换了 SessionContext 和 assignmapper。构建到“sessionmaker()”概念上，以产生一个类，其 Session() 构造返回线程本地会话。或者，将所有 Session 方法作为类方法调用，例如 Session.save(foo); Session.commit()。就像旧的“objectstore”时代一样。

+   **[orm]**

    添加了会话的新“binds”参数，以支持通过 sessionmaker() 函数配置多个绑定。

+   **[orm]**

    添加了一个基本的 SessionExtension 类，允许在 flush()、commit() 和 rollback() 边界处发生用户定义的功能。

+   **[orm]**

    使用 dynamic_loader() 可用于基于查询的 relation()。这是一个 *可写* 集合（支持 append() 和 remove()），当访问时也是一个活动的查询对象。对于处理只需要部分加载的非常大的集合非常理想。

+   **[orm]**

    flush() -内嵌的内联 INSERT/UPDATE 表达式。将任何 SQL 表达式，如“sometable.c.column + 1”，分配给实例的属性。在 flush() 时，映射器检测到表达式并将其直接嵌入到 INSERT 或 UPDATE 语句中；属性在实例上被延迟，因此在下次访问时加载新值。

+   **[orm]**

    引入了一个基本的分片（水平扩展）系统。该系统使用修改后的 Session，可以根据用户定义的“分片策略”将读写操作分布在多个数据库之间。实例及其依赖项可以根据属性值、轮询方法或任何其他用户定义的系统在多个数据库之间分布和查询。

    参考：[#618](https://www.sqlalchemy.org/trac/ticket/618)

+   **[orm]**

    急切加载已经增强，允许在更多地方进行更多的连接。它现在可以在自引用和循环结构的任意深度上运行。在加载循环结构时，在 relation()上指定“join_depth”，指示您希望表自连接多少次；每个级别都会获得一个不同的表别名。别名名称现在是在编译时使用简单的计数方案生成的，更容易阅读，当然完全确定性。

    参考：[#659](https://www.sqlalchemy.org/trac/ticket/659)

+   **[orm]**

    添加了复合列属性。这允许您在使用 ORM 时创建由多个列表示的类型。新类型的对象在查询表达式、比较、query.get()子句等方面都是完全功能的，并且表现得就像是常规的单列标量... 除了它们不是！在映射器的“属性”字典中使用函数 composite(cls, *columns)，并且 cls 的实例将被创建/映射到一个属性，由与*columns 对应的值组成。

    参考：[#211](https://www.sqlalchemy.org/trac/ticket/211)

+   **[orm]**

    改进了对具有相关子查询的自定义 column_property()属性的支持，现在与急切加载更好地配合。

+   **[orm]**

    主键“合并”行为；映射器将分析其给定可选择的所有列以获取主键“等效性”，即通过外键关系或显式继承条件等方式等效的列。主要用于连接表继承场景，其中继承表中的不同命名主键列应“合并”为单值（或更少值）主键。修复了一些问题。

    参考：[#611](https://www.sqlalchemy.org/trac/ticket/611)

+   **[orm]**

    加入表继承现在只会针对连接的根表生成所有继承类的主键列。这意味着根表中的每一行都对应一个单独的实例。如果出于某种罕见原因这不是理想的情况，那么在各个映射器上设置显式的主键设置将覆盖它。

+   **[orm]**

    当“多态”标志与连接表或单表继承一起使用时，所有标识键都针对继承层次结构的根类生成；这允许 query.get()以与非多态 get 相同的缓存语义多态工作。请注意，目前这不适用于具体继承。

+   **[orm]**

    次要继承加载：可以构建多态映射器，*无需*选择表参数。继承映射器，其表在初始加载中未被表示，将立即发出第二个 SQL 查询，每个实例一次（即对于大型列表来说效率不高），以加载剩余列。

+   **[orm]**

    次要继承加载也可以将其第二个查询移动到列级“延迟”加载中，通过“polymorphic_fetch”参数，可以设置为‘select’或‘deferred’。

+   **[orm]**

    现在可以仅将可用的可选择列的子集映射到映射器属性，使用 include_columns/exclude_columns。

    参考：[#696](https://www.sqlalchemy.org/trac/ticket/696)

+   **[orm]**

    添加了 undefer_group() MapperOption，设置一组由“group”连接的“延迟”列以作为“未延迟”加载。

+   **[orm]**

    重写了“确定性别名”逻辑，使其成为 SQL 层的一部分，生成更简单的别名和标签名称，更符合 Hibernate 的风格。

### sql

+   **[sql]**

    速度！子句编译以及 SQL 构造的机制已经被简化和简化到相当程度，使语句构造/编译的开销减少了 20-30%，为 0.3。

+   **[sql]**

    所有��type”关键字参数，如 bindparam()、column()、Column()和 func.<something>()，重命名为“type_”。这些对象仍然将它们的“type”属性命名为“type”。

+   **[sql]**

    从模式项中删除了 case_sensitive=(True|False)设置，因为检查此状态会增加很多方法调用开销，而且从来没有合理的理由将其设置为 False。所有小写的表和列名将被视为不区分大小写（是的，我们也适应了 Oracle 的大写风格）。

### extensions

+   **[extensions]**

    proxyengine 暂时移除，等待一个真正有效的替代品。

+   **[extensions]**

    SelectResults 已被 Query 取代。 SelectResults / SelectResultsExt 仍然存在，但只是返回一个稍微修改的 Query 对象以保持向后兼容性。 SelectResults 的 join_to()方法不再存在，需要使用 join()。

### mysql

+   **[mysql]**

    通过反射加载的表和列名现在是 Unicode。

+   **[mysql]**

    现在支持所有标准列类型，包括 SET。

+   **[mysql]**

    现在可以在一次往返中执行表反射。

+   **[mysql]**

    现在支持 ANSI 和 ANSI_QUOTES SQL 模式。

+   **[mysql]**

    索引现在已经反映出来。

### oracle

+   **[oracle]**

    添加了对 OUT 参数的非常基本支持；使用 sql.outparam(name, type)设置 OUT 参数，就像 bindparam()一样；执行后，值可以通过 result.out_parameters 字典获得。

    参考：[#507](https://www.sqlalchemy.org/trac/ticket/507)

### 杂项

+   **[transactions]**

    为事务添加了上下文管理器（with 语句）支持。

+   **[transactions]**

    支持了两阶段提交，目前与 mysql 和 postgres 兼容。

+   **[transactions]**

    添加了使用保存点的子事务实现。

+   **[事务]**

    添加了对保存点的支持。

+   **[元数据]**

    可以从数据库中反射表，而无需事先声明它们。MetaData(engine, reflect=True)将加载数据库中存在的所有表，或使用 metadata.reflect()进行更精细的控制。

+   **[元数据]**

    DynamicMetaData 已更名为 ThreadLocalMetaData。

+   **[元数据]**

    ThreadLocalMetaData 构造函数现在不接受任何参数。

+   **[元数据]**

    BoundMetaData 已被移除- 普通的 MetaData 是等效的。

+   **[元数据]**

    数值和浮点类型现在有一个“asdecimal”标志；Numeric 默认为 True，Float 默认为 False。当为 True 时，值以 decimal.Decimal 对象返回；当为 False 时，值以 float()返回。对于 PG 和 MySQL 的 DBAPI 模块，True/False 的默认值已经是行为。

    参考：[#646](https://www.sqlalchemy.org/trac/ticket/646)

+   **[元数据]**

    新的 SQL 运算符实现将所有硬编码的运算符从表达式结构中移除，并将它们移到编译中；允许更大的运算符编译灵活性；例如，在字符串上下文中使用“+”时，编译为“||”，或在 MySQL 上使用“concat(a,b)”；而在数值上下文中编译为“+”。修复。

    参考：[#475](https://www.sqlalchemy.org/trac/ticket/475)

+   **[元数据]**

    “匿名”别名和标签名称现在在 SQL 编译时以完全确定性的方式生成…不再是随机的十六进制 ID。

+   **[元数据]**

    对 SQL 元素（ClauseElement）进行了重大的架构改造。所有元素共享一个通用的“可变性”框架，允许对元素进行一致的原地修改以及生成行为。改进了 ORM 的稳定性，ORM 大量使用对 SQL 表达式的修改。

+   **[元数据]**

    select()和 union()现在具有“生成”行为。像 order_by()和 group_by()这样的方法返回一个*新*实例-原始实例保持不变。非生成方法也保留。

+   **[元数据]**

    select/union 的内部大幅简化- 所有关于“是否为子查询”和“相关性”的决策都推迟到 SQL 生成阶段。select()元素现在*永远*不会被其封闭容器或任何方言的编译过程改变。

    参考：[#52](https://www.sqlalchemy.org/trac/ticket/52), [#569](https://www.sqlalchemy.org/trac/ticket/569)

+   **[元数据]**

    select(scalar=True)参数已弃用；使用 select(..).as_scalar()。生成的对象遵循完整的“列”接口，并在表达式中更好地发挥作用。

+   **[元数据]**

    添加了 select().with_prefix(‘foo’)方法，允许在 SELECT 语句的列子句之前放置任何关键字。

    参考：[#504](https://www.sqlalchemy.org/trac/ticket/504)

+   **[元数据]**

    添加了对行[<index>]的数组切片支持。

    参考：[#686](https://www.sqlalchemy.org/trac/ticket/686)

+   **[元数据]**

    结果集更好地尝试将 cursor.description 中存在的 DBAPI 类型与方言定义的 TypeEngine 对象进行匹配，然后用于结果处理。请注意，这仅对文本 SQL 生效；构造的 SQL 语句始终具有显式的类型映射。

+   **[元数据]**

    CRUD 操作的结果集立即关闭其底层游标，并且如果为操作定义了自动关闭连接，则也会自动关闭连接；这样可以更有效地使用连接来进行连续的 CRUD 操作，减少“悬空连接”的可能性。

+   **[元数据]**

    列默认值和 onupdate Python 函数（即传递给 ColumnDefault 的函数）可以接受零个或一个参数；一个参数是 ExecutionContext，您可以从中调用“context.parameters[someparam]”来访问附加到语句的其他绑定参数值。执行使用的连接也可用，以便您可以预执行语句。

    参考：[#559](https://www.sqlalchemy.org/trac/ticket/559)

+   **[元数据]**

    为序列添加了“显式”创建/删除/执行支持（即您可以将“connectable”传递给 Sequence 上的这些方法）。

+   **[元数据]**

    在操作模式模式时更好地引用标识符。

+   **[元数据]**

    标准化了无法定位类型的表反射的行为；NullType 将替换为警告被引发。

+   **[元数据]**

    ColumnCollection（即表上的‘c’属性）遵循字典语义中的“__contains__”

    参考：[#606](https://www.sqlalchemy.org/trac/ticket/606)

+   **[引擎]**

    速度！结果处理和绑定参数处理的机制已经进行了彻底的改进、简化和优化，以尽可能少地发出方法调用。批量 INSERT 和批量行集迭代的基准测试都表明 0.4 版本比 0.3 版本快两倍以上，使用的函数调用次数减少了 68%。

+   **[引擎]**

    现在可以钩入池生命周期，并在每次新的 DBAPI 连接、池签出和签入时运行 SQL 语句或其他逻辑。

+   **[引擎]**

    连接获取一个.properties 集合，其内容范围限定在底层 DBAPI 连接的生命周期内。

+   **[引擎]**

    从池中移除了 auto_close_cursors 和 disallow_open_cursors 参数；由于游标通常由 ResultProxy 和 Connection 关闭，因此减少了开销。

+   **[Postgres]**

    添加了 PGArray 数据类型，用于使用 Postgres 数组数据类型。
