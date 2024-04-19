# 0.5 变更日志

> 原文：[`docs.sqlalchemy.org/en/20/changelog/changelog_05.html`](https://docs.sqlalchemy.org/en/20/changelog/changelog_05.html)

## 0.5.9

无发布日期

### sql

+   **[sql]**

    在表达式包中修复了错误的 self_group()调用。

    参考：[#1661](https://www.sqlalchemy.org/trac/ticket/1661)

## 0.5.8

发布日期：2010 年 1 月 16 日

### sql

+   **[sql]**

    Column 上的 copy() 方法现在支持未初始化、未命名的 Column 对象。这允许在多个子类上放置常见列的声明性助手的简单创建。

+   **[sql]**

    类似 Sequence() 的默认生成器在复制操作中正确转换。

+   **[sql]**

    Sequence() 和其他 DefaultGenerator 对象现在可以作为 Column 的“default”和“onupdate”关键字参数的值被接受，除了被接受为位置参数。

+   **[sql]**

    修复了一个影响包含独立列表达式的克隆可选择项的列算术错误。这个 bug 通常只在使用 0.6 版本中仅在 ORM 行为中出现时才会注意到，但在 SQL 表达式级别上更加正确。

    参考：[#1568](https://www.sqlalchemy.org/trac/ticket/1568), [#1617](https://www.sqlalchemy.org/trac/ticket/1617)

### postgresql

+   **[postgresql]**

    extract() 函数在 0.5.7 中略有改进，但需要更多工作来生成正确的类型转换（在 PG 的 EXTRACT 中，类型转换在很多情况下都是必要的）。现在使用基于 PG 文档的日期/时间/间隔算术的规则字典生成类型转换。它再次接受 text() 构造，这在 0.5.7 中被破坏。

    参考：[#1647](https://www.sqlalchemy.org/trac/ticket/1647)

### misc

+   **[firebird]**

    将更多错误识别为断开连接。

    参考：[#1646](https://www.sqlalchemy.org/trac/ticket/1646)

## 0.5.7

发布日期：2009 年 12 月 26 日

### orm

+   **[orm]**

    contains_eager() 现在与自动生成的子查询一起工作，当您说“query(Parent).join(Parent.somejoinedsubclass)”时会产生，即当 Parent 加入到一个联接表继承子类时。以前，contains_eager() 会错误地将子类表单独添加到查询中，产生笛卡尔积。票据描述中有一个示例。

    参考：[#1543](https://www.sqlalchemy.org/trac/ticket/1543)

+   **[orm]**

    query.options() 现在仅对于相关行为的选项才传播到已加载对象，以进一步潜在地加载更多子加载，保持各种不可序列化的选项（如由 contains_eager() 生成的选项）不会进入各个实例状态。

    参考：[#1553](https://www.sqlalchemy.org/trac/ticket/1553)

+   **[orm]**

    Session.execute() 现在根据传入的表达式（insert()/update()/delete() 构造）定位特定于表和映射器的绑定。

    参考：[#1054](https://www.sqlalchemy.org/trac/ticket/1054)

+   **[orm]**

    Session.merge()现在正确地将 many-to-one 或 uselist=False 属性覆盖为 None，如果给定要合并的对象中该属性也为 None。

+   **[orm]**

    修复了在合并包含空主键标识符的瞬态对象时会发生不必要的 select 的问题。

    参考：[#1618](https://www.sqlalchemy.org/trac/ticket/1618)

+   **[orm]**

    传递给 relation()、column_property()等的“extension”属性的可变集合不会被修改或在多个 instrumentation 调用之间共享，防止重复的扩展（例如 backref populators）被插入到列表中。

    参考：[#1585](https://www.sqlalchemy.org/trac/ticket/1585)

+   **[orm]**

    修复了对 CompositeProperty 上 get_committed_value()的调用。

    参考：[#1504](https://www.sqlalchemy.org/trac/ticket/1504)

+   **[orm]**

    修复了一个 bug，即当在列列表中出现非映射列实体时，如果调用了没有明确“左”侧的 join()，Query 会崩溃。

    参考：[#1602](https://www.sqlalchemy.org/trac/ticket/1602)

+   **[orm]**

    修复了当在联接表子类上配置复合列时，由于 0.5.6 版本中为修复而引入的问题，复合列加载不正确的 bug。感谢 Scott Torborg。

    参考：[#1480](https://www.sqlalchemy.org/trac/ticket/1480), [#1616](https://www.sqlalchemy.org/trac/ticket/1616)

+   **[orm]**

    许多对一关系的“use get”行为，即懒加载将回退到可能缓存的 query.get()值，现在在两个比较类型不完全相同类，但共享相同“亲和性” - 即 Integer 和 SmallInteger 的连接条件中起作用。还允许反射和非反射类型的组合与 0.5 风格类型反射一起工作，例如 PGText/Text（请注意 0.6 将类型反射为它们的通用版本）。

    参考：[#1556](https://www.sqlalchemy.org/trac/ticket/1556)

+   **[orm]**

    修复了在 query.update()中将 Cls.attribute 作为值字典中的键传递，并使用 synchronize_session='expire'（0.6 中为'fetch'）时的 bug。

    参考：[#1436](https://www.sqlalchemy.org/trac/ticket/1436)

### sql

+   **[sql]**

    修复了两阶段事务中 commit()方法未设置完整状态的 bug，这允许随后的 close()调用成功。

    参考：[#1603](https://www.sqlalchemy.org/trac/ticket/1603)

+   **[sql]**

    修复了“numeric” paramstyle，显然这是 Informixdb 使用的默认 paramstyle。

+   **[sql]**

    在 select 的列子句中重复表达式根据每个子句元素的标识而不是实际字符串进行去重。这允许位置元素正确呈现，即使它们都呈现相同，例如“qmark”样式绑定参数。

    参考：[#1574](https://www.sqlalchemy.org/trac/ticket/1574)

+   **[sql]**

    与连接池连接相关联的光标（即 _CursorFairy）现在正确地将 __iter__()代理到底层光标。

    参考：[#1632](https://www.sqlalchemy.org/trac/ticket/1632)

+   **[sql]**

    类型现在支持“亲和性比较”操作，即 Integer/SmallInteger 是“兼容的”，或者 Text/String，PickleType/Binary 等。的一部分。

    参考：[#1556](https://www.sqlalchemy.org/trac/ticket/1556)

+   **[sql]**

    修复了阻止对别名()的别名()进行克隆或适应的错误（在 ORM 操作中经常发生）。

    参考：[#1641](https://www.sqlalchemy.org/trac/ticket/1641)

### postgresql

+   **[postgresql]**

    添加了对反射 DOUBLE PRECISION 类型的支持，通过一个新的 postgres.PGDoublePrecision 对象。这在 0.6 中是 postgresql.DOUBLE_PRECISION。

    参考：[#1085](https://www.sqlalchemy.org/trac/ticket/1085)

+   **[postgresql]**

    添加了对反射 INTERVAL 类型的 INTERVAL YEAR TO MONTH 和 INTERVAL DAY TO SECOND 语法的支持。

    参考：[#460](https://www.sqlalchemy.org/trac/ticket/460)

+   **[postgresql]**

    修正了“has_sequence”查询以考虑当前模式，或显式序列指定的模式。

    参考：[#1576](https://www.sqlalchemy.org/trac/ticket/1576)

+   **[postgresql]**

    修复了 extract()的行为，以应用运算符优先规则到“::”运算符，当应用“timestamp”转换时确保正确的括号。

    参考：[#1611](https://www.sqlalchemy.org/trac/ticket/1611)

### sqlite

+   **[sqlite]**

    sqlite 方言正确生成了一个在备用模式中的表的 CREATE INDEX。

    参考：[#1439](https://www.sqlalchemy.org/trac/ticket/1439)

### mssql

+   **[mssql]**

    在构造 pyodbc 连接参数时，将 TrustedConnection 的名称更改为 Trusted_Connection

    参考：[#1561](https://www.sqlalchemy.org/trac/ticket/1561)

### oracle

+   **[oracle]**

    “table_names”方言函数，被 MetaData .reflect()使用，省略了“索引溢出表”，这是 Oracle 在使用“溢出索引表”时生成的系统表。这些表无法通过 SQL 访问，也无法反射。

    参考：[#1637](https://www.sqlalchemy.org/trac/ticket/1637)

### 杂项

+   **[ext]**

    在构造类后（即通过类级属性赋值），可以向连接表声明的超类添加列，并且列将传播到子类。这与 0.5.6 中修复的情况相反。

    参考：[#1523](https://www.sqlalchemy.org/trac/ticket/1523), [#1570](https://www.sqlalchemy.org/trac/ticket/1570)

+   **[ext]**

    修复了分片示例中的轻微不准确之处。在 ORM 中比较列的等价性最好使用 col1.shares_lineage(col2)来实现。

    参考：[#1491](https://www.sqlalchemy.org/trac/ticket/1491)

+   **[ext]**

    从 ShardedQuery 中删除未使用的 load()方法。

    参考：[#1606](https://www.sqlalchemy.org/trac/ticket/1606)

## 0.5.6

发布日期：2009 年 9 月 12 日 星期六

### orm

+   **[orm]**

    修复了继承鉴别器作为复合主键的一部分在更新时失败的错误。的延续。

    参考：[#1300](https://www.sqlalchemy.org/trac/ticket/1300)

+   **[orm]**

    修复了一个 bug，该 bug 不允许双向多对多引用的一侧声明自己为“viewonly”。

    参考：[#1507](https://www.sqlalchemy.org/trac/ticket/1507)

+   **[orm]**

    添加了一个断言，防止@validates 函数或其他 AttributeExtension 加载未加载的集合，从而可能损坏内部状态。

    参考：[#1526](https://www.sqlalchemy.org/trac/ticket/1526)

+   **[orm]**

    修复了一个 bug，该 bug 阻止了两个实体在单个 flush()中相互替换主键值的某些操作顺序。

    参考：[#1519](https://www.sqlalchemy.org/trac/ticket/1519)

+   **[orm]**

    修复了一个隐晦的问题，即具有基类上自引用的急切加载的连接表子类将使用来自父类的“子类”表的数据填充相关对象的“子类”表。

    参考：[#1485](https://www.sqlalchemy.org/trac/ticket/1485)

+   **[orm]**

    relations()现在具有更大的“覆盖”能力，意味着明确指定覆盖父类关系的子类的关系()将在刷新期间被尊重。目前支持从具体继承设置中的多对多关系。除此用例外，效果可能有所不同。

    参考：[#1477](https://www.sqlalchemy.org/trac/ticket/1477)

+   **[orm]**

    从 relation()中挤出了更多不必要的“延迟加载”。当集合发生变化时，另一侧的多对一反向引用不会加载“旧”值，除非设置了“single_parent=True”。直接分配多对一仍会加载“旧”值，以便更新该值上的反向引用集合，该值可能已经存在于会话中，从而保持 0.5 行为契约。

    参考：[#1483](https://www.sqlalchemy.org/trac/ticket/1483)

+   **[orm]**

    修复了一个 bug，即基于 column_property()或类似属性的加载/刷新连接表继承属性将无法评估。

    参考：[#1480](https://www.sqlalchemy.org/trac/ticket/1480)

+   **[orm]**

    改进了对 MapperProperty 对象的支持，覆盖了非具体继承设置中继承映射器的属性扩展不会随机冲突。

    参考：[#1488](https://www.sqlalchemy.org/trac/ticket/1488)

+   **[orm]**

    标准 SQL 中不支持 UPDATE 和 DELETE 支持 ORDER BY、LIMIT、OFFSET 等。如果调用了 limit()、offset()、order_by()、group_by()或 distinct()，Query.update()和 Query.delete()现在会引发异常。

    参考：[#1487](https://www.sqlalchemy.org/trac/ticket/1487)

+   **[orm]**

    将 AttributeExtension 添加到 sqlalchemy.orm.__all__

+   **[orm]**

    当使用非 SQL /实体表达式调用 query()时，改进了错误消息。

    参考：[#1476](https://www.sqlalchemy.org/trac/ticket/1476)

+   **[orm]**

    现在在基类和子类中使用 False 或 0 作为多态鉴别器也能正常工作。

    参考：[#1440](https://www.sqlalchemy.org/trac/ticket/1440)

+   **[orm]**

    在查询中添加了`enable_assertions(False)`，用于禁用对预期状态的通常断言 - 用于由查询子类工程化自定义状态。参见 [`www.sqlalchemy.org/trac/wiki/UsageRecipes/PreFilteredQuery`](https://www.sqlalchemy.org/trac/wiki/UsageRecipes/PreFilteredQuery) 以获得示例。

    参考文献：[#1424](https://www.sqlalchemy.org/trac/ticket/1424)

+   **[orm]**

    修复了一个递归问题，该问题在映射对象的`__len__()`或`__nonzero__()`方法导致状态更改时发生。

    参考文献：[#1501](https://www.sqlalchemy.org/trac/ticket/1501)

+   **[orm]**

    修复了`Weak/StrongIdentityMap.add()`中的错误异常。

    参考文献：[#1506](https://www.sqlalchemy.org/trac/ticket/1506)

+   **[orm]**

    修复了对于“无法找到 FROM 子句”在`query.join()`中的错误消息，如果查询针对纯 SQL 结构，则无法正确发出。

    参考文献：[#1522](https://www.sqlalchemy.org/trac/ticket/1522)

+   **[orm]**

    修复了一个有点假设性的问题，该问题会导致使用旧的`polymorphic_union`函数的映射器计算出错误的主键 - 但这是旧问题。

    参考文献：[#1486](https://www.sqlalchemy.org/trac/ticket/1486)

### sql

+   **[sql]**

    修复了`column.copy()`以复制默认值和 onupdates。

    参考文献：[#1373](https://www.sqlalchemy.org/trac/ticket/1373)

+   **[sql]**

    修复了在 0.5.4 中引入的`extract()`中的一个错误，其中字符串“field”参数被视为 ClauseElement，导致更复杂的 SQL 转换中出现各种错误。

+   **[sql]**

    一元表达式（例如`DISTINCT`）将其类型处理传播到结果集，允许进行 unicode 等转换。

    参考文献：[#1420](https://www.sqlalchemy.org/trac/ticket/1420)

+   **[sql]**

    修复了在`Table`和`Column`中传递空字典作为“info”参数会引发异常的错误。

    参考文献：[#1482](https://www.sqlalchemy.org/trac/ticket/1482)

### oracle

+   **[oracle]**

    向 Oracle 别名名称未被截断的 0.6 修复进行回溯。

    参考文献：[#1309](https://www.sqlalchemy.org/trac/ticket/1309)

### misc

+   **[ext]**

    由`associationproxy`生成的集合代理现在可以进行 pickle 化。 但是，用户定义的`proxy_factory`仍然不能进行 pickle 化，除非它定义了`__getstate__`和`__setstate__`。

    参考文献：[#1446](https://www.sqlalchemy.org/trac/ticket/1446)

+   **[ext]**

    如果将`__table_args__`作为没有字典参数的元组传递，则`Declarative`将引发一条信息性异常。 改进了文档。

    参考文献：[#1468](https://www.sqlalchemy.org/trac/ticket/1468)

+   **[ext]**

    在`MetaData`中声明的`Table`对象现在可以在发送到`primaryjoin/secondaryjoin/secondary`的字符串表达式中使用 - 名称从声明基础的`MetaData`中提取。

    参考文献：[#1527](https://www.sqlalchemy.org/trac/ticket/1527)

+   **[ext]**

    在构造类之后（即通过类级别属性赋值），可以向联接表子类添加列。该列始终会添加到底层的 Table 中，但现在映射器将重建其“join”以包括新列，而不是引发关于“没有这样的列，请使用 column_property()”的错误。

    参考资料：[#1523](https://www.sqlalchemy.org/trac/ticket/1523)

+   **[test]**

    将示例添加到测试套件中，以便定期进行测试，并清理掉一些不必要的弃用警告。

## 0.5.5

发布日期：2009 年 7 月 13 日星期一

### 通用

+   **[general]**

    单元测试已从 unittest 迁移到 nose。有关如何运行测试的信息，请参阅 README.unittests。

    参考资料：[#970](https://www.sqlalchemy.org/trac/ticket/970)

### orm

+   **[orm]**

    relation() 的 “foreign_keys” 参数现在会自动传播到相同方式的 backref 中，就像 primaryjoin 和 secondaryjoin 一样。对于极其罕见的情况，backref 的 relation() 有意配置了不同的 “foreign_keys”，现在双方都需要显式配置（如果它们实际上需要此设置，请参见下一个注释…）。

+   **[orm]**

    …唯一已知的（而且真的很少见）使用案例，其中在前向/后向方面使用了不同的 foreign_keys 设置，即部分指向自己列的复合外键，已得到增强，以便不再使用 fk->itself 方面来确定关系方向。

+   **[orm]**

    Session.mapper 现在已被 *弃用*。

    如果您希望一个独立的对象成为会话的一部分，请调用 session.add()。否则，现在在 [`www.sqlalchemy.org/trac/wiki/UsageRecipes/SessionAwareMapper`](https://www.sqlalchemy.org/trac/wiki/UsageRecipes/SessionAwareMapper) 文档中记录了 Session.mapper 的 DIY 版本。该方法在 0.6 版本中仍将被弃用。

+   **[orm]**

    修复了 Query 可以从联接表子类实体的单个列进行 join() 的 bug，即 query(SubClass.foo, SubClass.bar).join(<anything>)。在大多数情况下，会引发错误“找不到要从中联接的 FROM 子句”。在另外一些情况下，结果将以基类而不是子类的方式返回，因此依赖于这个错误结果的应用程序需要进行调整。

    参考资料：[#1431](https://www.sqlalchemy.org/trac/ticket/1431)

+   **[orm]**

    修复了一个涉及 contains_eager() 的 bug，在一个特定的罕见情况下，它会应用于次要（即惰性）加载，产生笛卡尔积。改进了对次要加载的 query.options() 的定位。

    参考资料：[#1461](https://www.sqlalchemy.org/trac/ticket/1461)

+   **[orm]**

    修复了 0.5.4 版本中引入的一个 bug，即当默认保持列被刷新时，组合类型失败。

+   **[orm]**

    修复了另一个 0.5.4 版本的 bug，即当整个对象被序列化时，可变属性（即 PickleType）不会被正确地反序列化。

    参考资料：[#1426](https://www.sqlalchemy.org/trac/ticket/1426)

+   **[orm]**

    修复了如果使用了任何同义词，session.is_modified() 将引发异常的错误。

+   **[orm]**

    修复了以前 pickled 对象重新放回会话时可能不会完全垃圾回收的潜在内存泄漏。

+   **[orm]**

    修复了基于列表的属性（如 pickletype 和 PGArray）未能正确合并的错误。

+   **[orm]**

    修复了不起作用的 attributes.set_committed_value 函数。

+   **[orm]**

    精简了 InstanceState 的 pickle 格式，这应该进一步减少 pickled 实例的内存占用。该格式应与 0.5.4 及之前的版本向后兼容。

+   **[orm]**

    sqlalchemy.orm.join 和 sqlalchemy.orm.outerjoin 现在已添加到 sqlalchemy.orm.* 的 __all__ 中。

    参考：[#1463](https://www.sqlalchemy.org/trac/ticket/1463)

+   **[orm]**

    修复了当传递给 get() 的复合主键值过短时，Query 异常引发失败的错误。

    参考：[#1458](https://www.sqlalchemy.org/trac/ticket/1458)

### sql

+   **[sql]**

    移除了 execute() 的一个晦涩特性（包括 connection、engine、Session），其中 bindparam() 构造可以作为 params 字典的键发送。这种用法未记录在案，并且是一个问题的核心，即由 text() 构造隐式创建的 bindparam() 对象可能具有与放置在 params 字典中的字符串相同的哈希值，并且在计算最终绑定参数时可能导致不适当的匹配。对于可能使用此功能的任何应用程序，此条件的内部检查将显著增加关键任务参数呈现的延迟，因此移除了此行为。这是一个向后不兼容的更改，但该功能从未被记录。

### 杂项

+   **[engine/pool]**

    为 StaticPool 实现了 recreate()。

## 0.5.4p2

发布日期：2009 年 5 月 26 日

### sql

+   **[sql]**

    修复了基于参数或不是 executemany() 样式的 SQL 异常打印。

### postgresql

+   **[postgresql]**

    废弃了硬编码的 TIMESTAMP 函数，当作为 func.TIMESTAMP(value) 使用时，会呈现“TIMESTAMP value”。这在某些平台上会中断，因为 PostgreSQL 不允许在此上下文中使用绑定��数。硬编码的大写也不合适，我们需要支持许多其他 PG 强制转换。因此，改用文本构造，即 select([“timestamp ‘12/05/09’”])。

## 0.5.4p1

发布日期：2009 年 5 月 18 日

### orm

+   **[orm]**

    修复了在 0.5.4 版本中引入的一个属性错误，当使用 merge() 与不完整对象时会出现。

## 0.5.4

发布日期：2009 年 5 月 17 日

### orm

+   **[orm]**

    在会话/flush() 与大型映射图、大量对象一起使用时，有关性能的显著增强：

    +   从 flush() 过程中移除了所有* O(N) 扫描行为，即扫描整个会话的操作，包括一个极其昂贵的操作，错误地假设主键值正在更改，而实际上并非如此。

        +   仍然存在一个特例，可能会调用完整扫描，如果现有的主键属性被修改为新值。

    +   会话的“弱引用”行为现在是*完全*的 - 对映射对象或相关项目/集合在其 __dict__ 中不会有任何强引用。对象中的反向引用和其他循环不再影响会话丢失对未修改对象的所有引用的能力。具有待处理更改的对象仍然会被强制保留，直到 flush。

        该实现还通过将垃圾回收项的“复活”过程仅与映射“可变”属性（即 PickleType、复合属性）相关联来提高性能。这消除了 gc 过程的开销，并简化了内部行为。

        如果“可变”属性更改是对象上唯一的更改，然后该对象被取消引用，那么在发出 UPDATE 时，映射器将无法访问其他属性状态。这可能对某些 MapperExtensions 表现出不同的行为。

        此更改还影响了内部属性 API，但不影响 AttributeExtension 接口或任何公开文档化的属性函数。

    +   工作单元在 flush()期间不再为整个映射器图生成“依赖”处理器图，而是仅为表示具有待处理更改的对象的那些映射器创建这样的处理器。在大型互连映射器图的情况下，这节省了大量方法调用。

    +   缓存了以前在每次 flush 时多次发生的“表排序”操作，还从 flush()中删除了大量方法调用次数。

    +   映射器._save_obj()中的其他冗余行为已经简化。

    参考：[#1398](https://www.sqlalchemy.org/trac/ticket/1398)

+   **[orm]**

    修改了 DynamicAttributeImpl 上的 query_cls，以接受 AppenderQuery 的完整 mixin 版本，从而允许对 AppenderMixin 进行子类化。

+   **[orm]**

    “多态鉴别器”列可能是主键的一部分，并且将填充正确的鉴别器值。

    参考：[#1300](https://www.sqlalchemy.org/trac/ticket/1300)

+   **[orm]**

    修复了评估器无法评估 IS NULL 子句的问题。

+   **[orm]**

    修复了“动态”关系上的“设置集合”功能，以正确启动事件。以前，只能将集合分配给待处理的父实例，否则修改事件将无法正确触发。现在，设置集合与 merge()兼容，修复了这个问题。

    参考：[#1352](https://www.sqlalchemy.org/trac/ticket/1352)

+   **[orm]**

    允许对使用仪器描述符构建的 PropertyOption 对象进行 pickling；以前，在对使用描述符选项（例如 query.options(eagerload(MyClass.foo)）加载的对象进行 pickling 时会出现 pickling 错误。

+   **[orm]**

    如果“延迟加载”SQL 子句与 get() 中使用的子句匹配，但包含一些硬编码的参数，则懒加载器将不使用 get()。以前，懒加载策略会在 get() 上失败。理想情况下，应该使用带有硬编码参数的 get()，但这需要进一步开发。

    参考：[#1357](https://www.sqlalchemy.org/trac/ticket/1357)

+   **[orm]**

    在加载期间，与 query.options() 关联的 MapperOptions 和其他状态不再捆绑在每个延迟/延迟加载属性的可调用对象中。这些选项现在仅与实例的状态对象关联一次，当它被填充时。这在大多数情况下消除了每个实例/属性加载器对象的需要，提高了单个实例的加载速度和内存开销。

    参考：[#1391](https://www.sqlalchemy.org/trac/ticket/1391)

+   **[orm]**

    修复了另一个位置，autoflush 干扰 session.merge() 的问题。现在，在 merge() 期间完全禁用 autoflush。

    参考：[#1360](https://www.sqlalchemy.org/trac/ticket/1360)

+   **[orm]**

    修复了一个 bug，该 bug 阻止了“可变主键”依赖逻辑在一对一关系上正常运行。

    参考：[#1406](https://www.sqlalchemy.org/trac/ticket/1406)

+   **[orm]**

    修复了 relation() 中的 bug，在 0.5.3 中引入，其中从基类到连接表子类的自引用关系将无法正确配置。

+   **[orm]**

    修复了当使用继承映射器时可能出现的模糊映射器编译问题，这会导致未初始化的属性。

+   **[orm]**

    修正了 session weak_identity_map 的文档 - 默认值为 True，表示正在使用弱引用映射。

+   **[orm]**

    修复了工作单位问题，即在被删除的对象所拥有的集合内的项目上的外键属性将不会在 relation() 自引用时设置为 None。

    参考：[#1376](https://www.sqlalchemy.org/trac/ticket/1376)

+   **[orm]**

    修复了 Query.update() 和 Query.delete() 在使用 eagerloaded 关系时的失败。

    参考：[#1378](https://www.sqlalchemy.org/trac/ticket/1378)

+   **[orm]**

    现在，在 foreign_keys 或 remote_side 集合中指定二进制 primaryjoin 条件的两个列是一个错误。而以前这只是荒谬的，但是以非确定性的方式会成功。

### sql

+   **[sql]**

    从 SQLA 0.6 版本开始，将“编译器”扩展进行了反向移植。这是一个标准化接口，允许创建自定义的 ClauseElement 子类和编译器。特别是当您想要构建具有特定于数据库的编译的构造时，它是 text() 的一个方便的替代品。有关详细信息，请参阅扩展文档。

+   **[sql]**

    当绑定参数的列表大于 10 时，异常消息将被截断，防止大型 executemany() 语句填满屏幕和日志文件，导致巨大的多页异常。

    参考：[#1413](https://www.sqlalchemy.org/trac/ticket/1413)

+   **[sql]**

    `sqlalchemy.extract()`现在是方言敏感的，可以在支持的数据库中按照惯例提取时间戳的组件，包括 SQLite。

+   **[sql]**

    修复了从 __clause_element__()样式构造（即声明列）构造的 ForeignKey 上的 __repr__()和其他 _get_colspec()方法的问题。

    参考：[#1353](https://www.sqlalchemy.org/trac/ticket/1353)

### schema

+   **[schema] [1341] [ticket: 594]**

    添加了一个 quote_schema()方法到 IdentifierPreparer 类，以便方言可以覆盖如何处理模式。这使得 MSSQL 方言可以将模式视为多部分标识符，例如'database.owner'。

### extensions

+   **[extensions]**

    修复了向声明类添加延迟或其他列属性的问题。

    参考：[#1379](https://www.sqlalchemy.org/trac/ticket/1379)

### mysql

+   **[mysql]**

    反射 FOREIGN KEY 构造将考虑到点分模式.表名组合，如果外键引用远程模式中的表。

    参考：[#1405](https://www.sqlalchemy.org/trac/ticket/1405)

### sqlite

+   **[sqlite]**

    修正了 SLBoolean 类型，使其正确地将只有 1 视为 True。

    参考：[#1402](https://www.sqlalchemy.org/trac/ticket/1402)

+   **[sqlite]**

    修正了 float 类型，使其在反射时正确地映射到 SLFloat 类型。

    参考：[#1273](https://www.sqlalchemy.org/trac/ticket/1273)

### mssql

+   **[mssql]**

    修改了保存点逻辑的工作方式，以防止它干扰非保存点导向的例程。保存点支持仍然非常实验性。

+   **[mssql]**

    添加了 MSSQL 的保留字，涵盖了 2008 年及以前的所有版本。

    参考：[#1310](https://www.sqlalchemy.org/trac/ticket/1310)

+   **[mssql]**

    修正了与基于二进制排序的数据库不兼容的信息模式的问题。清理了信息模式，因为现在只有 MSSQL 在使用。

    参考：[#1343](https://www.sqlalchemy.org/trac/ticket/1343)

## 0.5.3

发布日期：2009 年 3 月 24 日星期二

### orm

+   **[orm]**

    session.flush()中的“objects”参数已被弃用。表示父对象和子对象之间链接的状态不支持一个链接的一侧处于“flushed”状态而另一侧不处于该状态，因此支持此操作会导致误导性的结果。

    参考：[#1315](https://www.sqlalchemy.org/trac/ticket/1315)

+   **[orm]**

    查询现在实现了 __clause_element__()，它生成了可选择的内容，这意味着查询实例可以在许多 SQL 表达式中被接受，包括 col.in_(query)，union(query1, query2)，select([foo]).select_from(query)等。

+   **[orm]**

    Query.join()现在可以构造多个 FROM 子句，如果需要的话。例如，query(A, B).join(A.x).join(B.y)可能会生成 SELECT A.*, B.* FROM A JOIN X, B JOIN Y。贪婪加载也可以将其连接到这些多个 FROM 子句上。

    参考：[#1337](https://www.sqlalchemy.org/trac/ticket/1337)

+   **[orm]**

    修复了 dynamic_loader()中的错误，即在构建后未传播追加/删除事件到 UOW 以在 flush()中进行处理。

    参考：[#1347](https://www.sqlalchemy.org/trac/ticket/1347)

+   **[orm]**

    修复了在未映射已具有类级名称的属性之前未检查 column_prefix 的错误。

+   **[orm]**

    对特定集合属性执行 session.expire() 将清除任何待处理的反向引用添加，以便下一次访问正确返回数据库中存在的内容。对于 flush([objects]) 功能的一定程度的解决方案，尽管我们正在考虑完全删除该功能。

    参考：[#1315](https://www.sqlalchemy.org/trac/ticket/1315)

+   **[orm]**

    Session.scalar() 现在将原始 SQL 字符串转换为 text()，与 Session.execute() 相同，并接受相同的替代**kw args。

+   **[orm]**

    改进了 relation() 的“确定方向”逻辑，以便确定类似 mapper(A.join(B)) -> relation-> mapper(B) 的棘手情况的方向。

+   **[orm]**

    使用 session.flush([somelist]) 刷新部分对象集时，操作后仍保持待处理状态的对象不会被意外添加为持久对象。

    参考：[#1306](https://www.sqlalchemy.org/trac/ticket/1306)

+   **[orm]**

    添加了“post_configure_attribute”方法到 InstrumentationManager，以便“listen_for_events.py”示例再次正常工作。

    参考：[#1314](https://www.sqlalchemy.org/trac/ticket/1314)

+   **[orm]**

    现在检测到具有相同方向的前向和补充后向引用，即 ONETOMANY 或 MANYTOONE，将引发错误消息。以避免后续出现疯狂的 CircularDependencyErrors。

+   **[orm]**

    修复了 Query 中关于同时选择具有共同基类的多个连接表继承实体的错误：

    +   以前对“A JOIN B”上的“B”应用的适应会错误地部分应用于“A”。

    +   在关系比较（即 A.related==someb）时，当应该进行适应时未进行适应。

    +   其他过滤，如 query(A).join(A.bs).filter(B.foo==’bar’)，错误地将“B.foo”适应为“A”。

+   **[orm]**

    修复了通过 any()、has() 等与左侧别名对象和右侧 of_type() 结合使用的 EXISTS 子句的适应。

    参考：[#1325](https://www.sqlalchemy.org/trac/ticket/1325)

+   **[orm]**

    在 sqlalchemy.orm.attributes 中添加了一个属性助手方法 `set_committed_value`。给定一个对象、属性名称和值，将在对象上设置值作为其“已提交”状态的一部分，��被理解为已从数据库加载的状态。有助于创建自定义集合加载器等功能。

+   **[orm]**

    当传递非映射器/类的受仪器化描述符时，查询不会因弱引用错误而失败，而是引发“无效列表达式”。

+   **[orm]**

    Query.group_by() 正确考虑应用于 FROM 子句的别名，例如使用 select_from()、使用 with_polymorphic() 或使用 from_self()。

### sql

+   **[sql]**

    select()的 alias()在明确的标量上下文中使用时将转换为“标量子查询”，即在比较操作中使用时。当在 ORM 中使用 query.subquery()时也适用。

+   **[sql]**

    在使用 use_labels 时，在 select()中使用 Function 对象等时，修复了缺少的 _label 属性。

    参考：[#1302](https://www.sqlalchemy.org/trac/ticket/1302)

+   **[sql]**

    匿名别名现在会截断到方言允许的最大长度。在像 Oracle 这样具有非常小字符限制的数据库上更为重要。

    参考：[#1309](https://www.sqlalchemy.org/trac/ticket/1309)

+   **[sql]**

    __selectable__()接口已完全被 __clause_element__()替换。

+   **[sql]**

    TypeEngine 用于缓存特定于方言的类型的每个方言缓存现在是 WeakKeyDictionary。这是为了防止方言对象被引用，以便为创建任意数量的引擎或方言的应用程序永远引用。这会有一点性能损失，将在 0.6 中解决。

    参考：[#1299](https://www.sqlalchemy.org/trac/ticket/1299)

### extensions

+   **[extensions]**

    修复了 serializer 中的递归 pickling 问题，由 EXISTS 或其他嵌入的 FROM 构造触发。

+   **[extensions]**

    Declarative 通过 __bases__ 搜索定位“inherits”类，以跳过仅限于子类的 mixin。

+   **[extensions]**

    Declarative 即使明确给出“inherits”mapper 参数，也会找出联接表继承的主要联接条件。

+   **[extensions]**

    Declarative 将正确解释 backref()上的“foreign_keys”参数，如果它是一个字符串。

+   **[extensions]**

    当与 __table__ 一起使用时，Declarative 将接受一个绑定到表的列作为属性，如果该列已经存在于 __table__ 中。该列将被重新映射到给定的键，就像添加到 mapper()属性字典时一样。

### postgresql

+   **[postgresql]**

    当遇到具有多个表达式的索引时，索引反射不会失败。

+   **[postgresql]**

    在 sqlalchemy.databases.postgres 中添加了 PGUuid 和 PGBit 类型。

    参考：[#1327](https://www.sqlalchemy.org/trac/ticket/1327)

+   **[postgresql]**

    在域中指定未知 PG 类型的反射不会在这些类型被指定时崩溃。

    参考：[#1327](https://www.sqlalchemy.org/trac/ticket/1327)

### sqlite

+   **[sqlite]**

    修复了 SQLite 反射方法，以便在检测到不存在的 cursor.description 时（触发自动关闭游标），不会在最近版本的 pysqlite 上调用 fetchone()时失败，因为没有行存在时会引发错误。

### mssql

+   **[mssql]**

    对 pymssql 1.0.1 的初步支持

+   **[mssql]**

    修正了在 mssql 上未遵守 max_identifier_length 的问题。

## 0.5.2

发布日期：2009 年 1 月 24 日星期六

### orm

+   **[orm]**

    进一步完善了 0.5.1 对在多对多关系上放置 delete-orphan 级联的警告。首先，坏消息：警告将适用于多对多关系和多对一关系。这是必要的，因为在这两种情况下，SQLA 在确定“孤立”的状态时不会扫描全部潜在的父项集合 - 对于持久化对象，它仅检测到一个在 Python 中的取消关联事件来将对象确定为“孤立”。接下来，好消息：为了通过外键或关联表支持一对一关系，或者通过关联表支持一对多关系，可以设置一个新标志 single_parent=True，表示链接到关系的对象只应该有一个父对象。如果在 Python 中发生多个父关联事件，关系将引发错误。

+   **[orm]**

    调整了从 0.5.1 开始的属性检测更改，以便在超类已完全被检测后创建映射器的子类中完全建立属性检测。

    参考：[#1292](https://www.sqlalchemy.org/trac/ticket/1292)

+   **[orm]**

    修复了 delete-orphan 级联中的错误，其中来自两个不同父类到同一目标类的两个一对一关系会过早地删除实例。

+   **[orm]**

    修复了一个急加载错误，即自引用急加载会阻止其他急加载，无论是否是自引用，都无法正确连接到父 JOIN。感谢 Alex K 创建了一个很好的测试用例。

+   **[orm]**

    session.expire() 和相关方法将不会使未加载的延迟属性过期。这样可以防止在刷新实例时不必要地加载它们。

+   **[orm]**

    query.join()/outerjoin() 现在将正确地将 alias() 构造与现有的左侧连接起来，即使调用了 query.from_self() 或 query.select_from(someselectable)。

    参考：[#1293](https://www.sqlalchemy.org/trac/ticket/1293)

### sql

+   **[sql]**

    进一步修复了“百分号和列/表中的空格。

    名称”功能。

    参考：[#1284](https://www.sqlalchemy.org/trac/ticket/1284)

### mssql

+   **[mssql]**

    恢复了 convert_unicode 处理。结果被直接传递，而不进行转换。

    参考：[#1291](https://www.sqlalchemy.org/trac/ticket/1291)

+   **[mssql]**

    这次真的修复了 decimal 处理问题。

    参考：[#1282](https://www.sqlalchemy.org/trac/ticket/1282)

+   **[mssql] [Ticket:1289]**

    修改表反射代码，仅在构造表时使用 kwargs。

## 0.5.1

发布日期：Sat Jan 17 2009

### orm

+   **[orm]**

    移除了一个内部连接缓存，当对临时 selectables 重复发出 query.join() 时可能会泄漏内存。

+   **[orm]**

    “clear()”、“save()”、“update()”、“save_or_update()” Session 方法已被弃用，替换为“expunge_all()”和“add()”。“expunge_all()”也已添加到 ScopedSession。

+   **[orm]**

    现代化了“没有映射表”异常，并为声明式添加了一个更明确的 __table__/__tablename__ 异常。

+   **[orm]**

    现在具体继承映射器会对从超类继承的属性进行实例化，但对于具体映射器本身未定义的属性，会使用一个发出描述性错误的 InstrumentedAttribute。

    参考：[#1237](https://www.sqlalchemy.org/trac/ticket/1237)

+   **[orm]**

    添加了一个新的 relation()关键字 back_populates。这允许使用显式关系配置反向引用。在具体映射器层次结构和另一个类之间创建双向关系时，这是必需的。

    参考：[#1237](https://www.sqlalchemy.org/trac/ticket/1237), [#781](https://www.sqlalchemy.org/trac/ticket/781)

+   **[orm]**

    为在具体映射器上指定的 relation()对象添加了测试覆盖。

    参考：[#1237](https://www.sqlalchemy.org/trac/ticket/1237)

+   **[orm]**

    Query.from_self()以及 query.subquery()都会禁用生成的子查询内部的急切连接的渲染。“禁用所有急切连接”功能可以通过新的 query.enable_eagerloads()生成器公开使用。

    参考：[#1276](https://www.sqlalchemy.org/trac/ticket/1276)

+   **[orm]**

    在 Query 中添加了一系列基本的集合操作，接受 Query 对象作为参数，包括 union()、union_all()、intersect()、except_()、intersect_all()、except_all()。查看 Query.union()的 API 文档以获取示例。

+   **[orm]**

    修复了阻止 Query.join()和 eagerloads 附加到从联合或别名联合选择的查询的错误。

+   **[orm]**

    为在具体映射器上指定的双向关系添加了一个简短的文档示例。

    参考：[#1237](https://www.sqlalchemy.org/trac/ticket/1237)

+   **[orm]**

    现在映射器在构造时会使用最终的 InstrumentedAttribute 对象对类属性进行实例化，该对象保持持久。已删除 _CompiledOnAttr/__getattribute__()方法。其净效果是，基于列的映射类属性现在可以在类级别完全使用，而不需要调用映射器编译操作，大大简化了声明中的典型使用模式。

    参考：[#1269](https://www.sqlalchemy.org/trac/ticket/1269)

+   **[orm]**

    ColumnProperty（以及前端辅助程序如`deferred`）不再忽略未知的关键字参数。

+   **[orm]**

    修复了与 unitofwork 的“行切换”机制相关的错误，即将 INSERT/DELETE 转换为 UPDATE 时，与联接表继承和一个不包含子表定义值的对象相结合，将渲染出一个没有 SET 子句的 UPDATE。

+   **[orm]**

    在多对多关系上使用 delete-orphan 已被弃用。这会产生误导性或错误的结果，因为 SQLA 不会检索 m2m 的完整“父级”列表。要在 m2m 表上获得 delete-orphan 行为，请使用显式关联类，以便将单个关联行视为父级。

    参考：[#1281](https://www.sqlalchemy.org/trac/ticket/1281)

+   **[orm]**

    delete-orphan 级联总是需要 delete 级联。指定 delete-orphan 而不带 delete 现在会引发弃用警告。

    参考：[#1281](https://www.sqlalchemy.org/trac/ticket/1281)

### sql

+   **[sql]**

    改进了处理列名中百分号的方法。增加了更多测试。MySQL 和 PostgreSQL 方言仍然无法正确发出带有百分号的标识符的 CREATE TABLE 语句。

    参考：[#1256](https://www.sqlalchemy.org/trac/ticket/1256)

### 模式

+   **[模式]**

    Index 现在接受基于列的 InstrumentedAttributes（即基于列的映射类属性）作为列参数。

    参考：[#1214](https://www.sqlalchemy.org/trac/ticket/1214)

+   **[模式]**

    在没有名称的列（如在声明中）请求其字符串输出时，不会引发 NoneType 错误（例如在堆栈跟踪中）。

+   **[模式]**

    修复了在反射表上覆盖具有 ForeignKey 的列时的 bug，其中派生列（即 select 的“虚拟”列等）会无意中调用仅用于原始列的模式级清理逻辑。

    参考：[#1278](https://www.sqlalchemy.org/trac/ticket/1278)

### mysql

+   **[mysql]**

    添加了 MySQL 4.1 中缺失的关键字，以便正确转义。

### mssql

+   **[mssql]**

    更正了对大十进制值的处理，增加了更健壮的测试。移除了对浮点数的字符串操作。

    参考：[#1280](https://www.sqlalchemy.org/trac/ticket/1280)

+   **[mssql]**

    修改了 mssql 中 do_begin 处理，使用 Cursor 而不是 Connection，以使其与 DBAPI 兼容。

+   **[mssql]**

    通过更改 savepoint_release 的处理方式，纠正了 adodbapi 上的 SAVEPOINT 支持，因为在 mssql 上不支持。

### 杂项

+   **[声明]**

    现在可以在没有自己表的子类上指定 Column 对象（即使用单表继承）。这些列将附加到基本表，但仅由子类映射。

+   **[声明]**

    对于连接和单一继承子类，子类只会映射那些已在超类上映射的列以及子类上显式指定的列。默认情况下，将排除表上存在但未映射的其他列，可以通过将空的 exclude_properties 集合传递给 __mapper_args__ 来禁用此功能。这样，定义自己列的单一继承类将是唯一映射这些列的类。实际效果比通常使用显式 mapper() 调用设置 exclude_properties 参数更有组织。

+   **[声明]**

    向已使用 __table__ 指定现有表的声明类添加新的 Column 对象是错误的。

## 0.5.0

发布日期：2009 年 1 月 6 日 星期二

### 通用

+   **[通用]**

    文档已转换为 Sphinx。特别是，生成的 API 文档已构建为完整的“API 参考”部分，其中组织了编辑文档和生成的文档字符串。各个部分和 API 文档之间的交叉链接得到了极大改善，提供了一个基于 JavaScript 的搜索功能，并提供了所有类、函数和成员的完整索引。

+   **[general]**

    setup.py 现在仅在可选情况下导入 setuptools。如果不存在，将使用 distutils。新的“pip”安装程序建议使用 easy_install，因为它以更简化的方式安装。

+   **[general]**

    在示例文件夹中添加了一个极其基本的 PostGIS 集成示例。

### orm

+   **[orm]**

    Query.with_polymorphic()现在接受第三个参数“discriminator”，这将替换该查询的 mapper.polymorphic_on 的值。现在，即使 mapper 具有 polymorphic_identity，也不再需要设置 mapper 的 polymorphic_on。如果未设置，mapper 将默认以非多态方式加载。这两个特性共同允许非多态具体继承设置在每个查询基础上使用多态加载，因为在所有情况下使用多态具体设置会导致许多问题。

+   **[orm]**

    dynamic_loader 接受一个 query_class=来自定义用于动态集合和从中构建的查询的 Query 类。

+   **[orm]**

    query.order_by()接受 None，这将从查询中删除任何待处理的 order_by 状态，同时取消任何映射/关系配置的排序。这主要用于覆盖在 dynamic_loader()上指定的排序。

    参考：[#1079](https://www.sqlalchemy.org/trac/ticket/1079)

+   **[orm]**

    现在保留在 compile_mappers()期间引发的异常以提供“粘性行为”-如果对预编译映射属性的 hasattr()调用触发失败的编译并抑制异常，则会阻止后续编译，并且异常将在下一次 compile()调用时被重复。在使用 declarative 时经常出现此问题。

+   **[orm]**

    当在 prop.of_type(..).any()/has()的上下文中使用时，property.of_type()现在可以在单表继承目标上识别，以及 query.join(prop.of_type(…))。

+   **[orm]**

    当 join 的目标与基于属性的属性不匹配时，query.join()会引发错误-虽然不太可能有人这样做，但 SQLAlchemy 的作者却犯了这种特定的松散行为。

+   **[orm]**

    修复了在使用 weak_instance_map=False 时，修改事件不会被拦截进行 flush()的错误。

    参考：[#1272](https://www.sqlalchemy.org/trac/ticket/1272)

+   **[orm]**

    修复了一些可能影响针对包含同一表的多个版本的可选择性的查询的深层“列对应”问题，以及包含不同列位置的相同表列的联合和类似情况在不同级别。

    参考：[#1268](https://www.sqlalchemy.org/trac/ticket/1268)

+   **[orm]**

    与 column_property()、relation() 等一起使用的自定义比较器类可以在 Comparator 上定义新的比较方法，这些方法将通过 InstrumentedAttribute 上的 __getattr__() 变得可用。在 synonym() 或 comparable_property() 的情况下，首先在用户定义的描述符上解析属性，然后在用户定义的比较器上解析属性。

+   **[orm]**

    添加了 ScopedSession.is_active 访问器。

    参考：[#976](https://www.sqlalchemy.org/trac/ticket/976)

+   **[orm]**

    可以将映射属性和列对象作为键传递给 query.update({})。

    参考：[#1262](https://www.sqlalchemy.org/trac/ticket/1262)

+   **[orm]**

    传递给表达式级别 insert() 或 update() 的 values() 的映射属性将使用映射列的键，而不是映射属性的键。

+   **[orm]**

    修正了 Query.delete() 和 Query.update() 与绑定参数不正常工作的问题。

    参考：[#1242](https://www.sqlalchemy.org/trac/ticket/1242)

+   **[orm]**

    Query.select_from()、from_statement() 确保给定的参数是 FromClause，或 Text/Select/Union。

+   **[orm]**

    Query() 可以将“复合”属性作为列表达式传递，并进行扩展。与某种程度相关。

    参考：[#1253](https://www.sqlalchemy.org/trac/ticket/1253)

+   **[orm]**

    当传递各种列表达式（如字符串、clauselists、text() 构造）给 Query() 时，Query() 更加健壮（这可能意味着它只是更好地提供错误提示）。

+   **[orm]**

    first() 在 Query.from_statement() 中按预期工作。

+   **[orm]**

    修复了在 0.5rc4 中引入的关于对使用 add_property() 或等效方法在编译后向 mapper 添加属性的属性不起作用的急切加载的 bug。

+   **[orm]**

    修复了 many-to-many relation() 中 viewonly=True 的 bug，该 bug 未正确引用 secondary->remote 之间的链接。

+   **[orm]**

    在向“次要”表中的“二对多”关系发出 INSERT 时，列表型集合中的重复项将被保留。假设 m2m 表上有唯一或主键约束，这将引发预期的约束违规，而不是悄悄地删除重复条目。请注意，对于一对多关系，旧行为仍然保留，因为在这种情况下，集合条目不会导致 INSERT 语句，并且 SQLA 不会手动监视集合。

    参考：[#1232](https://www.sqlalchemy.org/trac/ticket/1232)

+   **[orm]**

    Query.add_column() 可以像 session.query() 一样接受 FromClause 对象。

+   **[orm]**

    将多对一关系与 NULL 的比较正确转换为基于 not_() 的 IS NOT NULL。

+   **[orm]**

    添加了额外的检查以确保显式的 primaryjoin/secondaryjoin 是 ClauseElement 实例，以防止后续出现更令人困惑的错误。

    参考：[#1087](https://www.sqlalchemy.org/trac/ticket/1087)

+   **[orm]**

    改进了 mapper() 对非类类的检查。

    参考：[#1236](https://www.sqlalchemy.org/trac/ticket/1236)

+   **[orm]**

    comparator_factory 参数现在已记录并受到所有 MapperProperty 类型的支持，包括 column_property()、relation()、backref() 和 synonym()。

    参考：[#5051](https://www.sqlalchemy.org/trac/ticket/5051)

+   **[orm]**

    将 PropertyLoader 的名称更改为 RelationProperty，以与所有其他名称保持一致。PropertyLoader 仍然存在作为同义词。

+   **[orm]**

    修复了在 shard API 中导致总线错误的“双重 iter()”调用，删除了 0.4 版本中遗留的错误 result.close()。

    参考：[#1099](https://www.sqlalchemy.org/trac/ticket/1099), [#1228](https://www.sqlalchemy.org/trac/ticket/1228)

+   **[orm]**

    使 Session.merge 级联不触发自动刷新。修复了合并实例过早插入且缺少值的问题。

+   **[orm]**

    为了帮助防止在多态联合继承场景中呈��出带外列而进行了两次修复（导致在 FROM 子句中呈现额外表，导致笛卡尔积）：

    > +   对于 a->b->c 继承情况的“列适配”进行了改进，以更好地定位通过多级间接关系相关的列，而不是呈现未适配的列。
    > +   
    > +   “多态鉴别器”列仅针对实际查询的映射器呈现。该列不会从子类或超类映射器中“拉取”，因为不需要。

+   **[orm]**

    修复了 ShardedSession.execute() 上的 shard_id 参数。

    参考：[#1072](https://www.sqlalchemy.org/trac/ticket/1072)

### sql

+   **[sql]**

    RowProxy 对象可以用于替代发送给 connection.execute() 和相关函数的字典参数。

    参考：[#935](https://www.sqlalchemy.org/trac/ticket/935)

+   **[sql]**

    列名中可以再次包含百分号。

    参考：[#1256](https://www.sqlalchemy.org/trac/ticket/1256)

+   **[sql]**

    sqlalchemy.sql.expression.Function 现在是一个公共类。可以通过子类化以提供用户定义的 SQL 函数，采用命令式风格，包括预先建立的行为。postgis.py 示例展示了其中一种用法。

+   **[sql]**

    PickleType 现在默认偏爱 == 比较，如果传入对象（如字典）实现了 __eq__()。如果对象没有实现 __eq__() 并且 mutable=True，则会引发弃用警告。

+   **[sql]**

    修复了 sqlalchemy.sql 中导出 __names__ 的奇怪问题。

    参考：[#1215](https://www.sqlalchemy.org/trac/ticket/1215)

+   **[sql]**

    多次使用相同的 ForeignKey 对象会引发错误，而不是在后期默默失败。

    参考：[#1238](https://www.sqlalchemy.org/trac/ticket/1238)

+   **[sql]**

    在 Insert/Update/Delete 构造中为 params() 方法添加了 NotImplementedError。这些项目目前不支持此功能，与 values() 相比也会有点误导。

+   **[sql]**

    反射的外键将正确定位其引用列，即使该列具有与反射名称不同的“key”属性。通过 ForeignKey/ForeignKeyConstraint 上的一个名为“link_to_name”的新标志来实现这一点，如果为 True，则表示给定名称是所引用列的名称，而不是其分配的键。

    参考：[#650](https://www.sqlalchemy.org/trac/ticket/650)

+   **[sql]**

    select()可以接受 ClauseList 作为列，方式与 Table 或其他可选择的方式相同，内部表达式将用作列元素。

    参考：[#1253](https://www.sqlalchemy.org/trac/ticket/1253)

+   **[sql]**

    session.is_modified()上的“passive”标志正确传播到属性管理器。

+   **[sql]**

    union()和 union_all()不会破坏应用于 select()内部的 order_by()。如果您将带有 order_by()的 select()与 union()结合使用（可能是为了支持 LIMIT/OFFSET），还应在其上调用 self_group()以应用括号。

### mysql

+   **[mysql]**

    在 text()构造中的“%”符号会自动转义为“%%”。由于这一更改的向后不兼容性，如果在字符串中检测到‘%%’，则会发出警告。

+   **[mysql]**

    修复了在反射期间 FK 列不存在时引发异常的错误。

    参考：[#1241](https://www.sqlalchemy.org/trac/ticket/1241)

+   **[mysql]**

    修复了涉及反射远程模式表的 bug，该表具有对该模式中另一个表的外键引用。

### sqlite

+   **[sqlite]**

    表反射现在存储列的实际 DefaultClause 值。

    参考：[#1266](https://www.sqlalchemy.org/trac/ticket/1266)

+   **[sqlite]**

    bug 修复，行为变更

### mssql

+   **[mssql]**

    添加了新的 MSGenericBinary 类型。这对应于 Binary 类型，因此它可以实现将长度指定类型视为固定宽度 Binary 类型和非长度类型视为无限长度 Binary 类型的专门行为。

+   **[mssql]**

    添加了新类型：MSVarBinary 和 MSImage。

    参考：[#1249](https://www.sqlalchemy.org/trac/ticket/1249)

+   **[mssql]**

    添加了 MSReal、MSNText、MSSmallDateTime、MSTime、MSDateTimeOffset 和 MSDateTime2 类型

+   **[mssql]**

    重构了日期/时间类型。`smalldatetime`数据类型不再截断为仅日期，现在将映射到 MSSmallDateTime 类型。

    参考：[#1254](https://www.sqlalchemy.org/trac/ticket/1254)

+   **[mssql]**

    修正了接受 int 的 Numerics 的问题。

+   **[mssql]**

    将`char_length`映射到`LEN()`函数。

+   **[mssql]**

    如果`INSERT`包含子选择，则`INSERT`将从`INSERT INTO VALUES`构造转换为`INSERT INTO SELECT`构造。

+   **[mssql]**

    如果列是`primary_key`的一部分，则将是`NOT NULL`，因为 MSSQL 不允许在`primary_key`列中使用`NULL`。

+   **[mssql]**

    `MSBinary`现在返回`BINARY`而不是`IMAGE`。这是一个向后不兼容的更改，因为`BINARY`是固定长度数据类型，而`IMAGE`是可变长度数据类型。

    参考：[#1249](https://www.sqlalchemy.org/trac/ticket/1249)

+   **[mssql]**

    `get_default_schema_name` 现在从数据库中反射，基于用户的默认模式。这仅适用于 MSSQL 2005 及更高版本。

    参考：[#1258](https://www.sqlalchemy.org/trac/ticket/1258)

+   **[mssql]**

    通过使用新的 collation 参数添加了排序规则支持。此功能支持以下类型：char、nchar、varchar、nvarchar、text、ntext。

    参考：[#1248](https://www.sqlalchemy.org/trac/ticket/1248)

+   **[mssql]**

    对连接字符串参数进行了更改，优先选择 DSN 作为 pyodbc 的默认规范。有关详细使用说明，请参阅 mssql.py 的文档字符串。

+   **[mssql]**

    增加了对保存点的实验性支持。目前与会话不完全兼容。

+   **[mssql]**

    支持三个级别的列可空性：NULL、NOT NULL 和数据库配置的默认值。默认列配置（nullable=True）现在将在 DDL 中生成 NULL。之前没有发出任何规范，数据库默认值会生效（通常为 NULL，但不总是）。要显式请求数据库默认值，请使用 nullable=None 配置列，DDL 中将不发出任何规范。这是不兼容的行为。

    参考：[#1243](https://www.sqlalchemy.org/trac/ticket/1243)

### oracle

+   **[oracle]**

    调整了 create_xid() 的格式以修复两阶段提交。我们现在收到了有关使用此更改正常运行 Oracle 两阶段提交的现场报告。

+   **[oracle]**

    添加了 OracleNVarchar 类型，生成 NVARCHAR2，并且还继承了 Unicode，因此默认情况下 convert_unicode=True。NVARCHAR2 自动反映到此类型中，因此这些列在没有显式 convert_unicode=True 标志的反射表上传递 Unicode。

    参考：[#1233](https://www.sqlalchemy.org/trac/ticket/1233)

+   **[oracle]**

    修复了某些类型的 out 参数无法接收的错误；感谢 huddlej 在 wwu.edu 的帮助！

    参考：[#1265](https://www.sqlalchemy.org/trac/ticket/1265)

### 杂项

+   **[dialect]**

    在方言上添加了一个新的 description_encoding 属性，用于在处理元数据时编码列名。这通常默认为 utf-8。

+   **[engine/pool]**

    Connection.invalidate() 检查关闭状态以避免属性错误。

    参考：[#1246](https://www.sqlalchemy.org/trac/ticket/1246)

+   **[engine/pool]**

    NullPool 支持失败时重新连接的行为。

    参考：[#1094](https://www.sqlalchemy.org/trac/ticket/1094)

+   **[engine/pool]**

    在使用 pool.manage(dbapi) 时，添加了一个互斥锁以防止启动时发生“dogpile”行为的小问题。这可以防止出现严重负载启动时可能发生的一种次要情况。

    参考：[#799](https://www.sqlalchemy.org/trac/ticket/799)

+   **[engine/pool]**

    _execute_clauseelement() 现在再次成为私有方法。现在可用 ConnectionProxy，因此不再需要子类化 Connection。

+   **[documentation]**

    票务。

    参考：[#1149](https://www.sqlalchemy.org/trac/ticket/1149), [#1200](https://www.sqlalchemy.org/trac/ticket/1200)

+   **[documentation]**

    添加了关于 create_session() 默认值的注释。

+   **[documentation]**

    添加了关于 metadata.reflect() 的部分。

+   **[documentation]**

    更新了 TypeDecorator 部分。

+   **[documentation]**

    由于最近对此功能的混淆，重写了文档中的“threadlocal”策略部分。

+   **[documentation]**

    从继承中删除了过时的‘polymorphic_fetch’和‘select_table’文档，重写了“joined table inheritance”的后半部分。

+   **[documentation]**

    记录了 comparator_factory kwarg，增加了新的文档部分“自定义比较器”。

+   **[postgres]**

    在 text() 结构中的“%”符号会自动转义为“%%”。由于此更改的向后不兼容性，如果在字符串中检测到‘%%’，则会发出警告。

    参考：[#1267](https://www.sqlalchemy.org/trac/ticket/1267)

+   **[postgres]**

    在与 server_side_cursors 结合使用时，调用 alias.execute() 不会引发 AttributeError。

+   **[postgres]**

    已添加索引反射支持到 PostgreSQL，使用了我们长期忽视的一个很棒的补丁，由 Ken Kuhlman 提交。

    参考：[#714](https://www.sqlalchemy.org/trac/ticket/714)

+   **[associationproxy]**

    关联代理属性现在在类级别上可用，例如 MyClass.aproxy。以前这会评估为 None。

+   **[declarative]**

    由 backref() 作为字符串接受的参数列表包括 ‘primaryjoin’、‘secondaryjoin’、‘secondary’、‘foreign_keys’、‘remote_side’、‘order_by’。

## 0.5.0rc4

发布日期：Fri Nov 14 2008

### 通用

+   **[general]**

    全局“propigate”->“propagate”变更。

### orm

+   **[orm]**

    Query.count() 已增强以在更广泛的情况下执行“正确的操作”。现在它可以计算多实体查询，以及基于列的查询。请注意，这意味着如果你说 query(A, B).count() 没有任何连接条件，它将计算 A*B 的笛卡尔积。任何针对基于列的实体的查询都会自动发出“SELECT count(1) FROM (SELECT…)”以返回实际的行数，这意味着这样的查询 query(func.count(A.name)).count() 将返回一个值，因为该查询将返回一行。

+   **[orm]**

    大量的性能调优。对各种 ORM 操作的粗略估计将其速度提高了 10%，比 0.5.0rc3 快 25-30%。

+   **[orm]**

    修复错误和行为变更

+   **[orm]**

    调整了增强的 InstanceState 上的垃圾收集，以更好地防止由于丢失状态而引起的错误。

+   **[orm]**

    当针对多个实体执行时，Query.get() 返回更具信息性的错误消息。

    参考：[#1220](https://www.sqlalchemy.org/trac/ticket/1220)

+   **[orm]**

    在 Cls.relation.in_ 上恢复了 NotImplementedError。

    参考：[#1140](https://www.sqlalchemy.org/trac/ticket/1140), [#1221](https://www.sqlalchemy.org/trac/ticket/1221)

+   **[orm]**

    修复了涉及 relation() 上 order_by 参数的 PendingDeprecationWarning。

    参考：[#1226](https://www.sqlalchemy.org/trac/ticket/1226)

### sql

+   **[sql]**

    移除了 Connection 对象的 ‘properties’ 属性，应该使用 Connection.info。

+   **[sql]**

    恢复了在 ResultProxy 自动关闭游标之前获取 “active rowcount” 的情况。这在 0.5rc3 中被移除。

+   **[sql]**

    重新排列了 TypeDecorator 中的 load_dialect_impl() 方法，使其即使用户定义的 TypeDecorator 使用另一个 TypeDecorator 作为其实现也会生效。

### mssql

+   **[mssql]**

    大量清理和修复以纠正限制和偏移的问题。

+   **[mssql]**

    修正了作为二进制表达式一部分的子查询需要被转换为使用 IN 和 NOT IN 语法的情况。

+   **[mssql]**

    修复了 E Notation 问题，该问题导致无法插入小于 1E-6 的十进制值。

    参考：[#1216](https://www.sqlalchemy.org/trac/ticket/1216)

+   **[mssql]**

    修正了处理模式时反射的问题，特别是当这些模式是默认模式时。

    参考：[#1217](https://www.sqlalchemy.org/trac/ticket/1217)

+   **[mssql]**

    修正了将零长度项转换为 varchar 时的问题。现在它会正确调整 CAST。

### 杂项

+   **[access]**

    添加了对 Currency 类型的支持。

+   **[access]**

    函数没有返回它们的结果。

    参考：[#1017](https://www.sqlalchemy.org/trac/ticket/1017)

+   **[access]**

    修正了连接的问题。Access 仅支持 LEFT OUTER 或 INNER，而不仅仅是 JOIN 本身。

    参考：[#1017](https://www.sqlalchemy.org/trac/ticket/1017)

+   **[ext]**

    现在可以在使用 declarative 时在 __mapper_args__ 中使用自定义的 “inherit_condition”。

+   **[ext]**

    修复了基于字符串的“remote_side”、“order_by”等在 backref() 中使用时未正确传播的问题。

## 0.5.0rc3

发布日期：2008 年 11 月 07 日 星期五

### orm

+   **[orm]**

    添加了两个新的 SessionExtension 钩子：after_bulk_delete() 和 after_bulk_update()。after_bulk_delete() 在查询上进行批量 delete() 操作后调用。after_bulk_update() 在查询上进行批量 update() 操作��调用。

+   **[orm]**

    简单的一对多关系的“不等于”比较不会掉入到一个 EXISTS 子句中，而是会比较外键列。

+   **[orm]**

    移除了比较集合和可迭代对象的不太有效的用例。使用 contains() 来测试集合成员资格。

+   **[orm]**

    改进了 aliased() 对象的行为，使其更准确地适应生成的表达式，这对于自引用比较特别有帮助。

    参考：[#1171](https://www.sqlalchemy.org/trac/ticket/1171)

+   **[orm]**

    修复了涉及从类绑定属性构造的 primaryjoin/secondaryjoin 条件的 bug（在使用 declarative 时经常发生），后者稍后会被 Query 不适当地别名化，特别是在各种基于 EXISTS 的比较器中。

+   **[orm]**

    修复了使用多个 query.join() 与绑定到别名的描述符时会丢失左别名的 bug。

+   **[orm]**

    改进了弱引用标识映射的内存管理，不再需要互斥，对于具有挂起更改的 InstanceState，在需要时延迟恢复被垃圾回收的实例。

+   **[orm]**

    InstanceState 对象现在在处理销毁时会删除对自身的循环引用，以使其保持在循环垃圾收集之外。

+   **[orm]**

    relation() 在确定联接条件时不会在“请指定 primaryjoin”消息中隐藏不相关的 ForeignKey 错误。

+   **[orm]**

    修复了在与同一类的多个别名结合使用时涉及 order_by() 的查询中的错误（将添加测试）。

    参考：[#1218](https://www.sqlalchemy.org/trac/ticket/1218)

+   **[orm]**

    使用 Query.join() 时，如果 ON 子句明确指定了一个子句，该子句将被别名化为连接的左侧，从而使诸如 query(Source). from_self().join((Dest, Source.id==Dest.source_id)) 这样的场景能够正常工作。

+   **[orm]**

    如果 polymorphic_union() 函数中的每个列的 “key” 与列的名称不同，则将尊重每个列的 “key”。

+   **[orm]**

    修复了在具有“delete”级联的多对一关系上的“被动删除”支持。

    参考：[#1183](https://www.sqlalchemy.org/trac/ticket/1183)

+   **[orm]**

    修复了复合类型中的错误，该错误阻止了主键复合类型的变异。

    参考：[#1213](https://www.sqlalchemy.org/trac/ticket/1213)

+   **[orm]**

    对内部属性访问增加了更多细节，这样级联和刷新操作将不会初始化未加载的属性和集合，保留它们以便稍后进行延迟加载。对于待处理的实例，backref 事件仍然会初始化属性和集合。

    参考：[#1202](https://www.sqlalchemy.org/trac/ticket/1202)

### sql

+   **[sql]**

    SQL 编译器优化和复杂性降低。编译典型 select() 结构的调用次数比 0.5.0rc2 减少了 20%。

+   **[sql]**

    方言现在可以生成可调长度的标签名称。传递参数 “label_length=<value>” 到 create_engine() 来调整动态生成的列标签中最多包含多少个字符，例如“somecolumn AS somelabel”。任何小于 6 的值将导致标签的最小尺寸，包括下划线和数字计数器。编译器使用 dialect.max_identifier_length 的值作为默认值。

    参考：[#1211](https://www.sqlalchemy.org/trac/ticket/1211)

+   **[sql]**

    现在简化了对于 ResultProxy “无结果自动关闭” 的检查，仅基于光标描述的存在。所有基于正则表达式的有关返回行的猜测都已经被移除。

    参考：[#1212](https://www.sqlalchemy.org/trac/ticket/1212)

+   **[sql]**

    直接执行 union() 构造将正确设置结果行处理。

    参考：[#1194](https://www.sqlalchemy.org/trac/ticket/1194)

+   **[sql]**

    “OID”或“ROWID”列的内部概念已被移除。基本上没有任何方言使用它，现在使用 INSERT..RETURNING，使用 psycopg2 的 cursor.lastrowid 的可能性基本上已经消失。

+   **[sql]**

    在所有 FromClause 对象上删除了“default_order_by()”方法。

+   **[sql]**

    修复了 table.tometadata()方法，以便传递的 schema 参数传播到 ForeignKey 构造中。

+   **[sql]**

    稍微改变了对比空集合的 IN 运算符的行为。现在会导致与自身的不等比较。更具可移植性，但会破坏不是纯函数的存储过程。

### mysql

+   **[mysql]**

    修复了外键反射的 bug，在这种情况下，一个 Table 的显式 schema=与连接附加到的 schema（数据库）相同。

+   **[mysql]**

    不再期望在表反射中包含 include_columns 为小写。

### oracle

+   **[oracle]**

    为 Oracle 方言编写了一个文档字符串。显然，Ohloh 的“few source code comments”标签开始串联了 :).

+   **[oracle]**

    在使用 LIMIT/OFFSET 时，删除了 FIRST_ROWS()优化标志，可以通过 optimize_limits=True create_engine()标志重新启用。

    参考：[#536](https://www.sqlalchemy.org/trac/ticket/536)

+   **[oracle]**

    修复了错误和行为变化

+   **[oracle]**

    在 create_engine()上将 auto_convert_lobs 设置为 False 也会指示 OracleBinary 类型将 cx_oracle LOB 对象原样返回。

### 杂项

+   **[ext]**

    添加了一个新的扩展 sqlalchemy.ext.serializer。提供了与 Pickle/Unpickle 相对应的 Serializer/Deserializer“类”，以及 dumps()和 loads()。这个序列化器实现了一个“外部对象”pickler，它保留了关键的上下文敏感对象，包括 engines、sessions、metadata、Tables/Columns 和 mappers，使其保持在 pickle 流之外，并且可以在以后使用任何 engine/metadata/session 提供程序来恢复 pickle。这不是用于 pickle 常规对象实例，这些对象实例可以在没有任何特殊逻辑的情况下进行 pickle，而是用于 pickle 表达式对象和完整的 Query 对象，以便在 unpickle 时可以恢复所有 mapper/engine/session 依赖关系。

+   **[ext]**

    修复了一个 bug，阻止了声明绑定的“column”对象在 column_mapped_collection()中的使用。

    参考：[#1174](https://www.sqlalchemy.org/trac/ticket/1174)

+   **[misc]**

    util.flatten_iterator()函数不会将具有 __iter__()方法的字符串解释为迭代器，例如在 pypy 中。

    参考：[#1077](https://www.sqlalchemy.org/trac/ticket/1077)

## 0.5.0rc2

发布日期：2008 年 10 月 12 日星期日

### orm

+   **[orm]**

    修复了涉及包含字面值或其他非列表达式的 read/write relation()在其 primaryjoin 条件等于外键列的 bug。

+   **[orm]**

    在 mapper()中的“non-batch”模式，这个功能允许在每个实例更新/插入时调用 mapper 扩展方法，现在遵循给定对象的插入顺序。

+   **[orm]**

    修复了 mapper 中与 RLock 相关的 bug，该 bug 可能在重新进入 mapper compile() 调用时发生死锁，这种情况发生在 ForeignKey 对象中使用声明性构造时。

+   **[orm]**

    ScopedSession.query_property 现在接受一个 query_cls 工厂，覆盖了会话配置的 query_cls。

+   **[orm]**

    修复了共享状态 bug，干扰了 ScopedSession.mapper 对象能够在对象子类上应用默认 __init__ 实现的能力。

+   **[orm]**

    修复了 Query 上的切片（即 query[x:y]）在长度为零的切片、两端为 None 的切片上正常工作的问题。

    参考：[#1177](https://www.sqlalchemy.org/trac/ticket/1177)

+   **[orm]**

    添加了一个示例，说明了 Celko 的“嵌套集”作为 SQLA 映射的示例。

+   **[orm]**

    包含别名参数的 contains_eager() 即使别名嵌入在 SELECT 中，也可以在通过 query.select_from() 发送到 Query 时工作。

+   **[orm]**

    contains_eager() 的用法现在与同时包含常规 eager load 和 limit/offset 的 Query 兼容，因为这些列被添加到 Query 生成的子查询中。

    参考：[#1180](https://www.sqlalchemy.org/trac/ticket/1180)

+   **[orm]**

    session.execute() 将执行传递给它的 Sequence 对象（从 0.4 版本开始的回归）。

+   **[orm]**

    从 object_mapper() 和 class_mapper() 中删除了“raiseerror”关键字参数。如果给定的类/实例未映射，这些函数在所有情况下都会引发异常。

+   **[orm]**

    修复了在 autocommit=False 会话上调用 session.transaction.commit() 时未启动新事务的 bug。

+   **[orm]**

    对 Session.identity_map 的弱引用行为进行了一些调整，以减少异步 GC 副作用。

+   **[orm]**

    调整了 Session 在后刷新时对新“干净”对象的记账，以更好地防止在异步 gc 时对对象进行操作。

    参考：[#1182](https://www.sqlalchemy.org/trac/ticket/1182)

### sql

+   **[sql]**

    column.in_(someselect) 现在可以作为一个列子句表达式使用，而不会使子查询泄漏到 FROM 子句中。

    参考：[#1074](https://www.sqlalchemy.org/trac/ticket/1074)

### mysql

+   **[mysql]**

    临时表现在可以反射。

### sqlite

+   **[sqlite]**

    对 SQLite 日期/时间绑定/结果处理进行了全面改进，使用正则表达式和格式字符串，而不是 strptime/strftime，以通用支持 1900 年前的日期、带微秒的日期。

    参考：[#968](https://www.sqlalchemy.org/trac/ticket/968)

+   **[sqlite]**

    在 sqlite 方言中禁用了 String（以及 Unicode、UnicodeText 等）的 convert_unicode 逻辑，以适应 pysqlite 2.5.0 的新要求，即只接受 Python unicode 对象；[`itsystementwicklung.de/pipermail/list-pysqlite/2008-March/000018.html`](https://itsystementwicklung.de/pipermail/list-pysqlite/2008-March/000018.html)

### oracle

+   **[oracle]**

    Oracle 将检测包含在 SELECT 前面的字符串型语句中的注释作为 SELECT 语句。

    参考：[#1187](https://www.sqlalchemy.org/trac/ticket/1187)

## 0.5.0rc1

发布日期：2008 年 9 月 11 日 星期四

### orm

+   **[orm]**

    Query 现在具有 delete() 和 update(values) 方法。这允许使用 Query 对象执行批量删除/更新。

+   **[orm]**

    查询(Query(*cols)返回的 RowTuple 对象现在优先使用映射属性名称而不是列键，列键而不是列名作为键名，即使这些不是底层 Column 对象的名称。直接的 Column 对象，如 Query(table.c.col)，将返回 Column 的“key”属性。

+   **[orm]**

    Query 添加了 scalar() 和 value() 方法，每个方法返回一个标量值。scalar() 不带参数，大致相当于 first()[0]，value() 接受一个列表达式，大致相当于 values(expr).next()[0]。

+   **[orm]**

    改进了在查询() 实体列表中放置 SQL 表达式时确定 FROM 子句的方法。特别是标量子查询不应该将其内部 FROM 对象泄露到外部查询中。

+   **[orm]**

    从一个映射类到一个映射子类的关系(relation())的连接，在映射子类配置为单表继承时，将包括一个 IN 子句，该子句限制了连接类的子类型为请求的类型，在连接的 ON 子句中生效。这对于急加载连接和查询.join()都生效。请注意，在某些情况下，IN 子句也会出现在查询的 WHERE 子句中，因为这个区分有多个触发点。

+   **[orm]**

    AttributeExtension 已经得到改进，使得事件在突变实际发生之前触发。此外，append() 和 set() 方法现在必须返回给定的值，该值用作突变操作中要使用的值。这允许创建验证 AttributeListeners，在实际操作发生之前引发异常，并且可以将给定的值更改为其他值。

+   **[orm]**

    column_property()、composite_property() 和 relation() 现在接受一个或多个 AttributeExtensions，使用“extension”关键字参数。

+   **[orm]**

    query.order_by().get() 从 GET 所发出的查询中悄悄删除了“ORDER BY”，但不会引发异常。

+   **[orm]**

    添加了一个验证器 AttributeExtension，以及一个 @validates 装饰器，其用法与 @reconstructor 类似，标记一个方法为验证一个或多个映射属性。

+   **[orm]**

    class.someprop.in_() 在关系“in_”实现之前引发 NotImplementedError

    参考：[#1140](https://www.sqlalchemy.org/trac/ticket/1140)

+   **[orm]**

    修复了尚未加载集合的多对多集合的主键更新问题

    参考：[#1127](https://www.sqlalchemy.org/trac/ticket/1127)

+   **[orm]**

    修复了使用组的 deferred() 列与一个无关的 synonym() 一起产生 AttributeError 的 bug，在延迟加载时。

+   **[orm]**

    SessionExtension 上的 before_flush()挂钩在最终计算新/脏/已删除列表之前发生，允许在 flush 继续之前在 before_flush()中进一步更改 Session 的状态。

    参考：[#1128](https://www.sqlalchemy.org/trac/ticket/1128)

+   **[orm]**

    Session 和其他对象的“extension”参数现在可以选择性地是一个列表，支持发送到多个 SessionExtension 实例的事件。Session 将 SessionExtensions 放在 Session.extensions 中。

+   **[orm]**

    对 flush()的重入调用会引发错误。这也作为针对 Session.flush()并发调用的基本但不是绝对可靠的检查。

+   **[orm]**

    改进了 query.join()在连接到已连接表继承子类时的行为，使用显式连接条件（即不是在关系上）。

+   **[orm]**

    @orm.attributes.reconstitute 和 MapperExtension.reconstitute 已重命名为@orm.reconstructor 和 MapperExtension.reconstruct_instance

+   **[orm]**

    修复了从基类继承的子类的@reconstructor 挂钩。

    参考：[#1129](https://www.sqlalchemy.org/trac/ticket/1129)

+   **[orm]**

    composite()属性类型现在支持 composite 类上的 __set_composite_values__()方法，如果类使用属性名称而不是列的键名表示状态，则需要该方法；默认生成的值现在在 flush 时正确填充。此外，将属性设置为 None 的复合体现在比较时是正确的。

    参考：[#1132](https://www.sqlalchemy.org/trac/ticket/1132)

+   **[orm]**

    attributes.get_history()返回的三元组现在可以是列表和元组的混合。（以前成员总是列表。）

+   **[orm]**

    修复了在实体上更改主键属性的错误，其中属性的先前值已过期会在 flush()时产生错误。

    参考：[#1151](https://www.sqlalchemy.org/trac/ticket/1151)

+   **[orm]**

    修复了自定义仪器化 bug，即对于 ORM 未加载的新构造实例，未调用 get_instance_dict()。

+   **[orm]**

    如果给定对象尚未存在，则 Session.delete()将给定对象添加到会话中。这是从 0.4 版本中的一个回归错误。

    参考：[#1150](https://www.sqlalchemy.org/trac/ticket/1150)

+   **[orm]**

    Session 上的 echo_uow 标志已弃用，工作单元日志现在仅适用于应用程序级别，而不是每个会话级别。

+   **[orm]**

    从 InstrumentedAttribute 中删除了不接受 escape kwaarg 的冲突 contains()运算符。

    参考：[#1153](https://www.sqlalchemy.org/trac/ticket/1153)

### sql

+   **[sql]**

    暂时回滚了“ORDER BY”增强功能。此功能暂停等待进一步开发。

    参考：[#1068](https://www.sqlalchemy.org/trac/ticket/1068)

+   **[sql]**

    exists()构造不会将其包含的元素列表作为 FROM 子句“导出”，从而使它们可以更有效地在 SELECT 的列子句中使用。

+   **[sql]**

    and_() 和 or_() 现在生成一个 ColumnElement，允许布尔表达式作为结果列，例如 select([and_(1, 0)]).

    参考：[#798](https://www.sqlalchemy.org/trac/ticket/798)

+   **[sql]**

    Bind 参数现在是 ColumnElement 的子类，这使得它们可以被 orm.query 选择（它们已经具有大多数 ColumnElement 语义）。

+   **[sql]**

    为 exists() 构造添加了 select_from() 方法，使其与常规 select() 更加兼容。

+   **[sql]**

    添加了 func.min()、func.max()、func.sum() 作为“通用函数”，基本上允许它们的返回类型自动确定。有助于 SQLite 上的日期、十进制类型等。

    参考：[#1160](https://www.sqlalchemy.org/trac/ticket/1160)

+   **[sql]**

    添加了 decimal.Decimal 作为“自动检测”类型；当使用 Decimal ��，绑定参数和通用函数将将它们的类型设置为 Numeric。

### schema

+   **[schema]**

    添加了“sorted_tables”访问器到 MetaData，它返回按依赖顺序排序的 Table 对象列表。这使得 MetaData.table_iterator() 方法被弃用。util.sort_tables() 中的“reverse=False”关键字参数也已被移除；使用 Python 的 ‘reversed’ 函数来反转结果。

    参考：[#1033](https://www.sqlalchemy.org/trac/ticket/1033)

+   **[schema]**

    所有 Numeric 类型的 ‘length’ 参数已重命名为 ‘scale’。‘length’ 已被弃用，仍会发出警告。

+   **[schema]**

    放弃了对用户定义类型（convert_result_value, convert_bind_param）的 0.3 兼容性。

### mysql

+   **[mysql]**

    MSInteger、MSBigInteger、MSTinyInteger、MSSmallInteger 和 MSYear 的 ‘length’ 参数已重命名为 ‘display_width’。

+   **[mysql]**

    添加了 MSMediumInteger 类型。

    参考：[#1146](https://www.sqlalchemy.org/trac/ticket/1146)

+   **[mysql]**

    函数 func.utc_timestamp() 编译为 UTC_TIMESTAMP，不带括号，当与 executemany() 结合使用时，括号似乎会妨碍使用。

### oracle

+   **[oracle]**

    limit/offset 不再使用 ROW NUMBER OVER 来限制行数，而是与特殊的 Oracle 优化注释一起使用子查询。允许 LIMIT/OFFSET 与 DISTINCT 一起使用。

    参考：[#536](https://www.sqlalchemy.org/trac/ticket/536)

+   **[oracle]**

    has_sequence() 现在考虑当前的 “schema” 参数

    参考：[#1155](https://www.sqlalchemy.org/trac/ticket/1155)

+   **[oracle]**

    将 BFILE 添加到反射类型名称中

    参考：[#1121](https://www.sqlalchemy.org/trac/ticket/1121)

### 杂项

+   **[declarative]**

    修复了如果复合主键引用另一个尚未定义的表，映射器无法初始化的 bug。

    参考：[#1161](https://www.sqlalchemy.org/trac/ticket/1161)

+   **[declarative]**

    修复了在使用 backref 时使用基于字符串的 primaryjoin 条件会引发异常的问题。

## 0.5.0beta3

发布日期：2008 年 8 月 4 日 星期一

### orm

+   **[orm]**

    SQLAlchemy 映射器的“entity_name” 功能已被移除。有关原因，请参见 [`tinyurl.com/6nm2ne`](https://tinyurl.com/6nm2ne)

+   **[orm]**

    Session、sessionmaker() 和 scoped_session() 上的“autoexpire” 标志已重命名为“expire_on_commit”。它不会影响 rollback() 的过期行为。

+   **[orm]**

    修复了在映射器延迟加载继承属性时可能发生的无限循环 bug。

+   **[orm]**

    为 Session 添加了一个名为“_enable_transaction_accounting”的遗留支持标志，当为 False 时，禁用所有事务级对象计数，包括回滚时的过期、提交时的过期、新建/删除列表维护和开始时的自动刷新。

+   **[orm]**

    relation() 的“cascade” 参数接受 None 作为值，这等效于没有级联。

+   **[orm]**

    对动态关系进行了关键修复，允许在 flush() 后正确清除“修改”历史记录。

+   **[orm]**

    类上的用户定义的@property 在映射器初始化期间被检测并保留在原位。这意味着如果@property 挡住了同名的表列，那么该列将根本不会被映射（并且列没有被重新映射到不同的名称），也不会应用来自继承类的受控属性。使用 include_properties/exclude_properties 集合排除的名称也适用相同的规则。

+   **[orm]**

    添加了一个名为 after_attach() 的新 SessionExtension 钩子。这在通过 add()、add_all()、delete() 和 merge() 附加对象时调用。

+   **[orm]**

    从另一个映射器继承的映射器，在继承其继承映射器的列时，将使用在该继承映射器中指定的重新分配的属性名称。以前，如果“Base” 将“base_id” 重新分配为名称“id”，那么“SubBase(Base)” 仍将获得一个名为“base_id”的属性。这可以通过在每个子映射器中显式声明列来解决，但这相当难以实现，而且在使用声明时也是不可能的。

    参考：[#1111](https://www.sqlalchemy.org/trac/ticket/1111)

+   **[orm]**

    修复了 Session 中潜在的一系列竞争条件，异步 GC 可能会在会话中删除未修改、不再引用的项目，因为它们存在于要处理的项目列表中，通常在 session.expunge_all() 和相关方法中。

+   **[orm]**

    对 _CompileOnAttr 机制进行了一些改进，这应该会减少“在编译期间未替换属性 x”警告的概率。（这通常适用于 SQLA 黑客，如 Elixir 开发人员）。

+   **[orm]**

    修复了一个 bug，即对于挂起的孤立实例引发的“未保存的挂起实例” FlushError 在生成负责错误的关系列表时不会考虑超类映射器。

### sql

+   **[sql]**

    没有参数的 func.count() 渲染为 COUNT(*)，相当于 func.count(text(‘*’))。

+   **[sql]**

    ORDER BY 表达式中的简单标签名称呈现为它们自身，而不是其对应表达式的重新陈述。目前，此功能仅对 SQLite、MySQL 和 PostgreSQL 启用。可以在显示支持此行为的每个方言时启用它。

    参考：[#1068](https://www.sqlalchemy.org/trac/ticket/1068)

### mysql

+   **[mysql]**

    用于在 CREATE TABLE 中使用的 MSEnum 值的引用现在是可选的，并将根据需要进行引用。 (对于现有表格，引用始终是可选的。)

    参考：[#1110](https://www.sqlalchemy.org/trac/ticket/1110)

### 杂项

+   **[扩展]**

    作为参数发送给 relation() 的 remote_side 和 foreign_keys 参数的类绑定属性现在被接受，允许它们与声明式一起使用。此外，修复了在与急加载一起指定为类绑定属性时指定 order_by 的错误。

+   **[扩展]**

    调整了声明式初始化的 Columns，使得非重命名列以与非声明式映射器相同的方式初始化。这允许继承映射器设置其同名的“id”列，以便父“id”列优先于子列，减少请求此值时的数据库往返。

## 0.5.0beta2

发布日期：2008 年 7 月 14 日星期一

### orm

+   **[对象关系映射]**

    除了过期属性，如果延迟属性的数据存在于结果集中，则也会加载。

    参考：[#870](https://www.sqlalchemy.org/trac/ticket/870)

+   **[对象关系映射]**

    如果属性列表不包含任何基于列的属性，则 session.refresh() 将引发一个信息性错误消息。

+   **[对象关系映射]**

    如果未指定列或映射器，query() 将引发一个信息性错误消息。

+   **[对象关系映射]**

    惰性加载器现在在继续之前会触发自动刷新。这使得在自动刷新的情况下，集合或标量关系的 expire() 函数能够正常工作。

+   **[对象关系映射]**

    column_property() 属性代表不在映射表中的 SQL 表达式或列（例如来自视图的列），在 INSERT 或 UPDATE 后会自动过期，假设它们没有在本地修改，以便在访问时刷新为最新数据。

    参考：[#887](https://www.sqlalchemy.org/trac/ticket/887)

+   **[对象关系映射]**

    在使用 query.join(cls, aliased=True) 时，修复了两个连接表继承映射器之间的显式自引用连接。

    参考：[#1082](https://www.sqlalchemy.org/trac/ticket/1082)

+   **[对象关系映射]**

    在与仅包含列子句和 SQL 表达式 ON 子句的 join 一起使用时，修复了 query.join()。

+   **[对象关系映射]**

    mapper() 中的“allow_column_override”标志已被移除。这个标志几乎总是被误解。其特定功能可以通过 include_properties/exclude_properties 映射器参数实现。

+   **[对象关系映射]**

    修复了 Query 上的 __str__() 方法。

    参考：[#1066](https://www.sqlalchemy.org/trac/ticket/1066)

+   **[对象关系映射]**

    即使定义了特定于表/映射器的绑定，Session.bind 仍然作为默认值使用。

### sql

+   **[sql]**

    添加了新的 match() 运算符，执行全文搜索。支持 PostgreSQL、SQLite、MySQL、MS-SQL 和 Oracle 后端。

### 模式

+   **[模式]**

    添加了 prefixes 选项到 Table，接受一个字符串列表，在 CREATE TABLE 语句中的 CREATE 后插入。

    参考：[#1075](https://www.sqlalchemy.org/trac/ticket/1075)

+   **[模式]**

    Unicode、UnicodeText 类型现在默认设置为“assert_unicode”和“convert_unicode”，但接受这些值的覆盖 **kwargs。

### extensions

+   **[扩展]**

    声明性支持一个 __table_args__ 类变量，它可以是一个字典，或者是一个元组形式的 (arg1, arg2, …, {kwarg1:value, …})，其中包含要传递给 Table 构造函数的位置参数 + 关键字参数。

    参考：[#1096](https://www.sqlalchemy.org/trac/ticket/1096)

### sqlite

+   **[sqlite]**

    修改了 SQLite 对“微秒”的表示，使其与 str(somedatetime) 的输出匹配，即微秒以字符串格式的小数秒表示。这使得 SQLA 的 SQLite 日期类型与直接使用 Pysqlite 保存的日期时间兼容（Pysqlite 只调用 str()）。请注意，这与 SQLA 0.4 生成的 SQLite 数据库文件中现有的微秒值不兼容。

    要全局获取旧行为：

    > from sqlalchemy.databases.sqlite import DateTimeMixin DateTimeMixin.__legacy_microseconds__ = True

    要在单个 DateTime 类型上获取行为：

    > t = sqlite.SLDateTime() t.__legacy_microseconds__ = True

    然后在 Column 上使用“t”作为类型。

    参考：[#1090](https://www.sqlalchemy.org/trac/ticket/1090)

+   **[sqlite]**

    SQLite 的 Date、DateTime 和 Time 类型现在只接受 Python datetime 对象，而不是字符串。如果您想要自己用 SQLite 格式化日期为字符串，请使用 String 类型。如果您希望它们无论如何都返回 datetime 对象，尽管它们接受字符串作为输入，请围绕 String 创建一个 TypeDecorator - SQLA 不鼓励这种模式。

## 0.5.0beta1

发布日期：2008 年 6 月 12 日 星期四

### 一般

+   **[一般]**

    全局“propigate”->”propagate” 更改。

### orm

+   **[ORM]**

    如果每个 Column 的“key”与列的名称不同，则 polymorphic_union() 函数将尊重每个 Column 的“key”。

+   **[ORM]**

    修复了一个仅在 0.4 版本中存在的 bug，该 bug 阻止了复合列与继承映射器正常工作。

    参考：[#1199](https://www.sqlalchemy.org/trac/ticket/1199)

+   **[ORM]**

    修复了映射器中与 RLock 相关的 bug，该 bug 可能在重新进入映射器 compile() 调用时发生死锁，这在 ForeignKey 对象中使用声明性构造时会发生。从 0.5 移植而来。

+   **[ORM]**

    修复了复合类型中的 bug，该 bug 阻止了主键复合类型的变异。

    参考：[#1213](https://www.sqlalchemy.org/trac/ticket/1213)

+   **[ORM]**

    添加了 ScopedSession.is_active 访问器。

    参考：[#976](https://www.sqlalchemy.org/trac/ticket/976)

+   **[ORM]**

    类绑定访问器可以作为 relation() order_by 的参数使用。

    参考：[#939](https://www.sqlalchemy.org/trac/ticket/939)

+   **[ORM]**

    修复了 ShardedSession.execute() 上的 shard_id 参数。

    参考：[#1072](https://www.sqlalchemy.org/trac/ticket/1072)

### sql

+   **[sql]**

    Connection.invalidate() 检查关闭状态以避免属性错误。

    参考：[#1246](https://www.sqlalchemy.org/trac/ticket/1246)

+   **[sql]**

    NullPool 支持失��时重新连接的行为。

    参考：[#1094](https://www.sqlalchemy.org/trac/ticket/1094)

+   **[sql]**

    TypeEngine 用于缓存特定于方言的类型的每个方言缓存现在是 WeakKeyDictionary。这是为了防止方言对象被引用，以便应用程序创建任意数量的引擎或方言。这会带来一些性能损失，将在 0.6 版本中解决。

    参考：[#1299](https://www.sqlalchemy.org/trac/ticket/1299)

+   **[sql]**

    修复了 SQLite 反射方法，以便检测到不存在的 cursor.description，触发自动关闭游标，以便在最近的 pysqlite 版本上不会在 fetchone() 调用时失败，因为没有结果。

### mysql

+   **[mysql]**

    修复了在反射期间未出现 FK 列时引发异常的错误。

    参考：[#1241](https://www.sqlalchemy.org/trac/ticket/1241)

### oracle

+   **[oracle]**

    修复了某些类型的 out 参数无法接收的错误；非常感谢 huddlej at wwu.edu！

    参考：[#1265](https://www.sqlalchemy.org/trac/ticket/1265)

### 杂项

+   **[no_tags]**

    现在 mapper 添加的 “__init__” 触发器/装饰器尝试精确地模仿原始 __init__ 的参数签名。对于 ‘_sa_session’ 的传递不再是隐式的-您必须在构造函数中允许此关键字参数。

+   **[no_tags]**

    ClassState 更名为 ClassManager。

+   **[no_tags]**

    类可以通过提供 __sa_instrumentation_manager__ 属性来提供自己的 InstrumentationManager。

+   **[no_tags]**

    自定义仪器可能使用任何机制将 ClassManager 与类关联，并将 InstanceState 与实例关联。这些对象上的属性仍然是 SQLAlchemy 原生仪器使用的默认关联机制。

+   **[no_tags]**

    将 entity_name、_sa_session_id 和 _instance_key 从实例对象移动到实例状态。这些值仍然以旧方式可用，现在已弃用，使用附加到类的描述符。在访问时将发出弃用警告。

+   **[no_tags]**

    _prepare_instrumentation 的 _prepare_instrumentation 别名已被移除。

+   **[no_tags]**

    sqlalchemy.exceptions 已更名为 sqlalchemy.exc。该模块可以使用任一名称导入。

+   **[no_tags]**

    与 ORM 相关的异常现在在 sqlalchemy.orm.exc 中定义。在导入 sqlalchemy.orm 期间，ConcurrentModificationError、FlushError 和 UnmappedColumnError 的兼容性别名将安装在 sqlalchemy.exc 中。

+   **[no_tags]**

    sqlalchemy.logging 已更名为 sqlalchemy.log。

+   **[no_tags]**

    sqlalchemy.log.SADeprecationWarning 的过渡别名已被移除。

+   **[no_tags]**

    移除了 exc.AssertionError，并使用 Python 内置的 AssertionError 替代使用。

+   **[no_tags]**

    对于单个类的多个 entity_name= 主映射器附加的 MapperExtensions 的行为已更改。对于一个类定义的第一个 mapper() 是唯一符合 MapperExtension 'instrument_class'、'init_instance' 和 'init_failed' 事件的 mapper。这是不向后兼容的；以前定义的最后一个 mapper 的扩展将接收这些事件。

+   **[firebird]**

    增加了对插入（仅限 2.0+）、更新和删除（仅限 2.1+）返回值的支持。

+   **[postgres]**

    添加了 Postgres 的索引反射支持，使用了 Ken Kuhlman 提交的一个很棒的我们长期忽视的补丁。

    参考：[#714](https://www.sqlalchemy.org/trac/ticket/714)

## 0.5.9

无发布日期

### sql

+   **[sql]**

    在 expression 包中修复了错误的 self_group() 调用。

    参考：[#1661](https://www.sqlalchemy.org/trac/ticket/1661)

### sql

+   **[sql]**

    在 expression 包中修复了错误的 self_group() 调用。

    参考：[#1661](https://www.sqlalchemy.org/trac/ticket/1661)

## 0.5.8

发布日期：2010 年 1 月 16 日（周六）

### sql

+   **[sql]**

    现在 Column 上的 copy() 方法支持未初始化的、未命名的 Column 对象。这使得可以轻松创建声明性助手，将常见列放置在多个子类上。

+   **[sql]**

    类似 Sequence() 的默认生成器在复制操作中正确转换。

+   **[sql]**

    Sequence() 和其他 DefaultGenerator 对象现在可以作为 Column 的 “default” 和 “onupdate” 关键字参数的值接受，除了被位置接受。 

+   **[sql]**

    修复了一个列算术错误，该错误影响包含独立列表达式的克隆可选项的列对应关系。这个错误通常只有在使用新的 ORM 行为（仅在 0.6 中可用）时才会注意到，但在 SQL 表达式级别也更加正确。

    参考：[#1568](https://www.sqlalchemy.org/trac/ticket/1568)，[#1617](https://www.sqlalchemy.org/trac/ticket/1617)

### postgresql

+   **[postgresql]**

    extract() 函数在 0.5.7 中稍微改进了，但仍需要更多工作来生成正确的类型转换（大部分时间在 PG 的 EXTRACT 中似乎是必要的）。现在，类型转换是使用基于 PG 的日期/时间/间隔算术文档的规则字典生成的。它再次接受 text() 构造，这在 0.5.7 中已经损坏了。

    参考：[#1647](https://www.sqlalchemy.org/trac/ticket/1647)

### misc

+   **[firebird]**

    将更多错误识别为断开连接。

    参考：[#1646](https://www.sqlalchemy.org/trac/ticket/1646)

### sql

+   **[sql]**

    现在 Column 上的 copy() 方法支持未初始化的、未命名的 Column 对象。这使得可以轻松创建声明性助手，将常见列放置在多个子类上。

+   **[sql]**

    像 Sequence() 这样的默认生成器在复制操作中能够正确转换。

+   **[sql]**

    Sequence() 和其他 DefaultGenerator 对象可以作为 Column 的 “default” 和 “onupdate” 关键字参数的值被接受，除了可以按位置接受之外。

+   **[sql]**

    修复了一个影响包含独立列表达式的克隆可选择项的列算术错误。这个 bug 通常只在使用 0.6 中可用的新 ORM 行为时才会注意到，但在 SQL 表达式级别也更正确。

    参考：[#1568](https://www.sqlalchemy.org/trac/ticket/1568), [#1617](https://www.sqlalchemy.org/trac/ticket/1617)

### postgresql

+   **[postgresql]**

    extract() 函数在 0.5.7 中稍微改进了一下，需要更多的工作来生成正确的类型转换（在 PG 的 EXTRACT 中，类型转换在很多情况下是必要的）。现在，根据 PG 的日期/时间/间隔算术文档生成一个规则字典来生成类型转换。它还再次接受 text() 构造，这在 0.5.7 中是有问题的。

    参考：[#1647](https://www.sqlalchemy.org/trac/ticket/1647)

### 杂项

+   **[firebird]**

    更多错误被识别为断开连接。

    参考：[#1646](https://www.sqlalchemy.org/trac/ticket/1646)

## 版本号：0.5.7

发布日期：2009 年 12 月 26 日 星期六

### orm

+   **[orm]**

    contains_eager() 现在可以与自动生成的子查询一起使用，当你说“query(Parent).join(Parent.somejoinedsubclass)”时，即当 Parent 加入到一个联接表继承子类时。以前，contains_eager() 会错误地将子类表单独添加到查询中，产生笛卡尔积。票务描述中有一个示例。

    参考：[#1543](https://www.sqlalchemy.org/trac/ticket/1543)

+   **[orm]**

    query.options() 现在只对加载的对象传播，以便进一步子加载只对相关的选项进行传播，将像 contains_eager() 生成的各种不可序列化选项排除在各个实例状态之外。

    参考：[#1553](https://www.sqlalchemy.org/trac/ticket/1553)

+   **[orm]**

    Session.execute() 现在根据传入的表达式（insert()/update()/delete() 构造）定位特定于表和映射器的绑定。

    参考：[#1054](https://www.sqlalchemy.org/trac/ticket/1054)

+   **[orm]**

    Session.merge() 现在可以正确地将一个多对一或 uselist=False 属性覆盖为 None，如果在要合并的对象中该属性也为 None。

+   **[orm]**

    修复了一个不必要的选择，在合并包含空主键标识符的瞬态对象时会发生。

    参考：[#1618](https://www.sqlalchemy.org/trac/ticket/1618)

+   **[orm]**

    传递给 relation()、column_property() 等的 “extension” 属性的可变集合不会被改变或在多个仪器调用之间共享，防止重复的扩展，比如 backref populators，被插入到列表中。

    参考：[#1585](https://www.sqlalchemy.org/trac/ticket/1585)

+   **[orm]**

    修复了 CompositeProperty 上 get_committed_value() 的调用。

    参考：[#1504](https://www.sqlalchemy.org/trac/ticket/1504)

+   **[orm]**

    修复了如果在列列表中出现非映射列实体，则 Query 在调用没有明确“左”侧的 join() 时会崩溃的 bug。

    参考：[#1602](https://www.sqlalchemy.org/trac/ticket/1602)

+   **[orm]**

    修复了在联接表子类上配置复合列时，复合列无法正确加载的 bug，在版本 0.5.6 中引入，作为修复的结果。感谢 Scott Torborg。

    参考：[#1480](https://www.sqlalchemy.org/trac/ticket/1480), [#1616](https://www.sqlalchemy.org/trac/ticket/1616)

+   **[orm]**

    对于多对一关系的“使用 get”行为，即懒加载将回退到可能缓存的 query.get() 值，现在可以跨越连接条件工作，其中两个比较的类型不是完全相同的类，但共享相同的“亲和性” - 即 Integer 和 SmallInteger。还允许反射和非反射类型的组合使用 0.5 风格的类型反射，例如 PGText/Text（注意 0.6 反射类型为其泛型版本）。

    参考：[#1556](https://www.sqlalchemy.org/trac/ticket/1556)

+   **[orm]**

    修复了在 query.update() 中将 Cls.attribute 作为值字典中的键并使用 synchronize_session='expire'（在 0.6 中为 'fetch'）时的 bug。

    参考：[#1436](https://www.sqlalchemy.org/trac/ticket/1436)

### sql

+   **[sql]**

    修复了两阶段事务中的一个 bug，即 commit() 方法未设置完整状态，从而允许后续的 close() 调用成功。

    参考：[#1603](https://www.sqlalchemy.org/trac/ticket/1603)

+   **[sql]**

    修复了“numeric”参数风格，默认参数风格使用 Informixdb。

+   **[sql]**

    在 select 的列子句中重复表达式根据每个子句元素的标识进行去重，而不是实际字符串。这允许位置元素正确呈现，即使它们都呈现相同，比如“问号”样式的绑定参数。

    参考：[#1574](https://www.sqlalchemy.org/trac/ticket/1574)

+   **[sql]**

    与连接池连接相关联的游标（即 _CursorFairy）现在正确地代理 __iter__() 到底层游标。

    参考：[#1632](https://www.sqlalchemy.org/trac/ticket/1632)

+   **[sql]**

    类型现在支持“亲和性比较”操作，即 Integer/SmallInteger 是“兼容的”，或者 Text/String，PickleType/Binary 等。的一部分。

    参考：[#1556](https://www.sqlalchemy.org/trac/ticket/1556)

+   **[sql]**

    修复了阻止别名的别名() 被克隆或适应的 bug（在 ORM 操作中经常发生）。

    参考：[#1641](https://www.sqlalchemy.org/trac/ticket/1641)

### postgresql

+   **[postgresql]**

    添加了对 DOUBLE PRECISION 类型的反射支持，通过新的 postgres.PGDoublePrecision 对象。这是 postgresql.DOUBLE_PRECISION 在 0.6 中的表示。

    参考：[#1085](https://www.sqlalchemy.org/trac/ticket/1085)

+   **[postgresql]**

    增加了对反映 INTERVAL 类型的 INTERVAL YEAR TO MONTH 和 INTERVAL DAY TO SECOND 语法的支持。

    参考：[#460](https://www.sqlalchemy.org/trac/ticket/460)

+   **[postgresql]**

    修正了“has_sequence”查询，以考虑当前模式或显式序列状态的模式。

    参考：[#1576](https://www.sqlalchemy.org/trac/ticket/1576)

+   **[postgresql]**

    修复了 extract()的行为，以应用运算符优先规则到应用“timestamp”转换时的“::”运算符 - 确保正确的括号。

    参考：[#1611](https://www.sqlalchemy.org/trac/ticket/1611)

### sqlite

+   **[sqlite]**

    sqlite 方言正确生成用于在备用模式中的表的 CREATE INDEX。

    参考：[#1439](https://www.sqlalchemy.org/trac/ticket/1439)

### mssql

+   **[mssql]**

    在构造 pyodbc 连接参数时，将 TrustedConnection 的名称更改为 Trusted_Connection

    参考：[#1561](https://www.sqlalchemy.org/trac/ticket/1561)

### oracle

+   **[oracle]**

    “table_names”方言函数，被 MetaData .reflect()使用，省略了“索引溢出表”，这是 Oracle 在使用“只有索引表”并且溢出时生成的系统表。这些表无法通过 SQL 访问，也无法反射。

    参考：[#1637](https://www.sqlalchemy.org/trac/ticket/1637)

### misc

+   **[ext]**

    在构建类之后（即通过类级属性赋值），可以向连接表声明的超类添加列，并且该列将传播到子类。这是与 0.5.6 中修复的相反情况。

    参考：[#1523](https://www.sqlalchemy.org/trac/ticket/1523), [#1570](https://www.sqlalchemy.org/trac/ticket/1570)

+   **[ext]**

    修复了分片示例中的轻微不准确性。在 ORM 中比较列的等价性最好使用 col1.shares_lineage(col2)。

    参考：[#1491](https://www.sqlalchemy.org/trac/ticket/1491)

+   **[ext]**

    从 ShardedQuery 中删除了未使用的 load()方法。

    参考：[#1606](https://www.sqlalchemy.org/trac/ticket/1606)

### orm

+   **[orm]**

    contains_eager()现在可以与自动生成的子查询一起使用，当您说“query(Parent).join(Parent.somejoinedsubclass)”时，即当 Parent 连接到一个连接表继承子类时。以前，contains_eager()会错误地将子类表单独添加到查询中，产生笛卡尔积。票务描述中有一个示例。

    参考：[#1543](https://www.sqlalchemy.org/trac/ticket/1543)

+   **[orm]**

    query.options()现在仅对可能进一步子加载的对象传播，仅对这种行为相关的选项进行传播，将由 contains_eager()生成的各种不可序列化选项排除在各个实例状态之外。

    参考：[#1553](https://www.sqlalchemy.org/trac/ticket/1553)

+   **[orm]**

    现在，Session.execute()根据传入的表达式（即 insert()/update()/delete()构造）定位特定于表和映射器的绑定。

    参考：[#1054](https://www.sqlalchemy.org/trac/ticket/1054)

+   **[orm]**

    Session.merge() 现在可以正确地将 many-to-one 或 uselist=False 属性重写为 None，如果给定要合并的对象中的属性也为 None。

+   **[orm]**

    修复了在合并包含空主键标识符的瞬态对象时会发生不必要的选择的 bug。

    参考：[#1618](https://www.sqlalchemy.org/trac/ticket/1618)

+   **[orm]**

    传递给 relation()、column_property() 等的“extension”属性的可变集合不会在多个插装调用之间被突变或共享，防止重复扩展，例如插入到列表中的 backref populator。

    参考：[#1585](https://www.sqlalchemy.org/trac/ticket/1585)

+   **[orm]**

    修复了对 CompositeProperty 上 get_committed_value() 的调用。

    参考：[#1504](https://www.sqlalchemy.org/trac/ticket/1504)

+   **[orm]**

    修复了如果在列列表中出现非映射列实体时调用没有明确“左”侧的 join()，则 Query 会崩溃的 bug。

    参考：[#1602](https://www.sqlalchemy.org/trac/ticket/1602)

+   **[orm]**

    修复了复合列在配置为连接表子类时无法正确加载的 bug，该 bug 由 0.5.6 版本中的修复引入，感谢 Scott Torborg。

    参考：[#1480](https://www.sqlalchemy.org/trac/ticket/1480), [#1616](https://www.sqlalchemy.org/trac/ticket/1616)

+   **[orm]**

    许多对一关系的“使用 get”行为，即延迟加载将回退到可能被缓存的 query.get() 值，现在可以跨越连接条件工作，其中两个比较的类型不完全相同，但共享相同的“亲和性” - 即 Integer 和 SmallInteger。还允许反射和非反射类型的组合与 0.5 风格的类型反射一起工作，例如 PGText/Text（请注意 0.6 反映类型为它们的通用版本）。

    参考：[#1556](https://www.sqlalchemy.org/trac/ticket/1556)

+   **[orm]**

    在传递 Cls.attribute 作为值字典中的键并使用 synchronize_session=’expire’（0.6 版本中为‘fetch’）时，修复了 query.update() 中的 bug。

    参考：[#1436](https://www.sqlalchemy.org/trac/ticket/1436)

### sql

+   **[sql]**

    修复了两阶段事务中 commit() 方法未设置完整状态的 bug，从而允许后续的 close() 调用成功。

    参考：[#1603](https://www.sqlalchemy.org/trac/ticket/1603)

+   **[sql]**

    修复了“numeric” paramstyle，默认情况下是 Informixdb 使用的默认 paramstyle。

+   **[sql]**

    在 select 的 columns 子句中重复表达式基于每个子句元素的标识进行去重，而不是实际字符串。这允许位置元素在所有元素都以相同方式呈现时正确呈现，例如“问号”样式的绑定参数。

    参考：[#1574](https://www.sqlalchemy.org/trac/ticket/1574)

+   **[sql]**

    与连接池连接关联的游标（即 _CursorFairy）现在正确地代理 __iter__() 到底层游标。

    引用：[#1632](https://www.sqlalchemy.org/trac/ticket/1632)

+   **[sql]**

    types 现在支持“关联比较”操作，即 Integer/SmallInteger 是“兼容的”，或者 Text/String、PickleType/Binary 等。部分。

    引用：[#1556](https://www.sqlalchemy.org/trac/ticket/1556)

+   **[sql]**

    修复了阻止 alias() 的 alias() 被克隆或适应的错误（在 ORM 操作中经常发生）。

    引用：[#1641](https://www.sqlalchemy.org/trac/ticket/1641)

### postgresql

+   **[postgresql]**

    增加了对反映 DOUBLE PRECISION 类型的支持，通过一个新的 postgres.PGDoublePrecision 对象。这在 0.6 中是 postgresql.DOUBLE_PRECISION。

    引用：[#1085](https://www.sqlalchemy.org/trac/ticket/1085)

+   **[postgresql]**

    增加了对反映 INTERVAL 类型的 INTERVAL YEAR TO MONTH 和 INTERVAL DAY TO SECOND 语法的支持。

    引用：[#460](https://www.sqlalchemy.org/trac/ticket/460)

+   **[postgresql]**

    修正了“has_sequence”查询以考虑当前模式或显式序列指定的模式。

    引用：[#1576](https://www.sqlalchemy.org/trac/ticket/1576)

+   **[postgresql]**

    修复了 extract() 的行为，以应用操作符优先级规则到“::”操作符时应用“timestamp”转换 - 确保正确的括号化。

    引用：[#1611](https://www.sqlalchemy.org/trac/ticket/1611)

### sqlite

+   **[sqlite]**

    sqlite 方言正确地为位于替代模式中的表生成 CREATE INDEX。

    引用：[#1439](https://www.sqlalchemy.org/trac/ticket/1439)

### mssql

+   **[mssql]**

    在构造 pyodbc 连接参数时，将 TrustedConnection 的名称更改为 Trusted_Connection

    引用：[#1561](https://www.sqlalchemy.org/trac/ticket/1561)

### oracle

+   **[oracle]**

    “table_names” 方言函数，被 MetaData.reflect() 使用，省略了“索引溢出表”，这是 Oracle 在使用“仅索引表”且溢出时生成的系统表。这些表无法通过 SQL 访问，也无法反映。

    引用：[#1637](https://www.sqlalchemy.org/trac/ticket/1637)

### misc

+   **[ext]**

    在类构造完成后（即通过类级属性分配），可以向连接表声明超类添加列，并且列将向下传播到子类。这与 0.5.6 中的情况相反。

    引用：[#1523](https://www.sqlalchemy.org/trac/ticket/1523)，[#1570](https://www.sqlalchemy.org/trac/ticket/1570)

+   **[ext]**

    修复了分片示例中的轻微不准确。在 ORM 中比较列的等效性最好使用 col1.shares_lineage(col2)。

    引用：[#1491](https://www.sqlalchemy.org/trac/ticket/1491)

+   **[ext]**

    从 ShardedQuery 中删除了未使用的 load() 方法。

    引用：[#1606](https://www.sqlalchemy.org/trac/ticket/1606)

## 0.5.6

发布日期：Sat Sep 12 2009

### orm

+   **[orm]**

    修复了继承主键的一部分在更新时会失败的错误。持续。

    引用：[#1300](https://www.sqlalchemy.org/trac/ticket/1300)

+   **[orm]**

    修复了一个 bug，不允许多对多双向引用的一侧声明自身为“viewonly”

    参考：[#1507](https://www.sqlalchemy.org/trac/ticket/1507)

+   **[orm]**

    添加了一个断言，防止@validates 函数或其他 AttributeExtension 加载未加载的集合，从而可能损坏内部状态。

    参考：[#1526](https://www.sqlalchemy.org/trac/ticket/1526)

+   **[orm]**

    修复了一个 bug，阻止了两个实体在单个 flush()中相互替换主键值的某些操作顺序。

    参考：[#1519](https://www.sqlalchemy.org/trac/ticket/1519)

+   **[orm]**

    修复了一个隐晦的问题，即具有基类上自引用的延迟加载的连接表子类将使用来自父类的“子类”表的数据填充相关对象的“子类”表。

    参考：[#1485](https://www.sqlalchemy.org/trac/ticket/1485)

+   **[orm]**

    relations()现在具有更大的“覆盖”能力，意味着显式指定覆盖父类 relation()的子类将在 flush 期间被尊重。目前支持具体继承设置中的多对多关系。除此之外，效果可能有所不同。

    参考：[#1477](https://www.sqlalchemy.org/trac/ticket/1477)

+   **[orm]**

    从 relation()中挤出了更多不必要的“懒加载”。当集合发生变化时，另一侧的多对一反向引用不会触发加载“旧”值，除非设置了“single_parent=True”。直接赋值多对一仍会加载“旧”值以更新该值上的反向引用集合，该值可能已经存在于会话中，从而保持 0.5 行为契约。

    参考：[#1483](https://www.sqlalchemy.org/trac/ticket/1483)

+   **[orm]**

    修复了一个 bug，即基于 column_property()或类似属性的连接表继承属性的加载/刷新将无法评估。

    参考：[#1480](https://www.sqlalchemy.org/trac/ticket/1480)

+   **[orm]**

    改进了对 MapperProperty 对象覆盖非具体继承设置中继承映射器的支持 - 属性扩展不会随机冲突。

    参考：[#1488](https://www.sqlalchemy.org/trac/ticket/1488)

+   **[orm]**

    标准 SQL 中的 UPDATE 和 DELETE 不支持 ORDER BY、LIMIT、OFFSET 等。如果调用了 limit()、offset()、order_by()、group_by()或 distinct()，Query.update()和 Query.delete()现在会引发异常。

    参考：[#1487](https://www.sqlalchemy.org/trac/ticket/1487)

+   **[orm]**

    将 AttributeExtension 添加到 sqlalchemy.orm.__all__

+   **[orm]**

    当使用非 SQL /entity 表达式调用 query()时，改进了错误消息。

    参考：[#1476](https://www.sqlalchemy.org/trac/ticket/1476)

+   **[orm]**

    在基类以及子类中使用 False 或 0 作为多态鉴别器现在也可以正常工作。

    参考：[#1440](https://www.sqlalchemy.org/trac/ticket/1440)

+   **[orm]**

    向 Query 中添加了 enable_assertions(False)，用于禁用对预期状态的常规断言 - 由 Query 子类使用以设计自定义状态。参见[`www.sqlalchemy.org/trac/wiki/UsageRecipes/PreFilteredQuery`](https://www.sqlalchemy.org/trac/wiki/UsageRecipes/PreFilteredQuery)以获取示例。

    参考资料：[#1424](https://www.sqlalchemy.org/trac/ticket/1424)

+   **[ORM]**

    修复了递归问题，该问题会在映射对象的 __len__()或 __nonzero__()方法导致状态更改时发生。

    参考资料：[#1501](https://www.sqlalchemy.org/trac/ticket/1501)

+   **[ORM]**

    修复了 Weak/StrongIdentityMap.add()中的错误异常引发。

    参考资料：[#1506](https://www.sqlalchemy.org/trac/ticket/1506)

+   **[ORM]**

    修复了 query.join()中“找不到 FROM 子句”的错误消息，如果查询针对纯 SQL 构造，则无法正确发出。

    参考资料：[#1522](https://www.sqlalchemy.org/trac/ticket/1522)

+   **[ORM]**

    修复了一个有点假设性的问题，该问题会导致使用旧的 polymorphic_union 函数的映射器计算出错的主键 - 但这是旧的东西。

    参考资料：[#1486](https://www.sqlalchemy.org/trac/ticket/1486)

### SQL

+   **[SQL]**

    修复了 column.copy()的问题，以便复制默认值和 onupdates。

    参考资料：[#1373](https://www.sqlalchemy.org/trac/ticket/1373)

+   **[SQL]**

    修复了在 0.5.4 中引入的 extract()中的一个错误，其中字符串“field”参数被视为 ClauseElement，导致更复杂的 SQL 转换中出现各种错误。

+   **[SQL]**

    诸如 DISTINCT 之类的一元表达式将其类型处理传播到结果集，从而允许进行类似 unicode 等的转换。

    参考资料：[#1420](https://www.sqlalchemy.org/trac/ticket/1420)

+   **[SQL]**

    修复了 Table 和 Column 中传递空字典作为“info”参数会引发异常的错误。

    参考资料：[#1482](https://www.sqlalchemy.org/trac/ticket/1482)

### Oracle

+   **[Oracle]**

    将 0.6 版本的 Oracle 别名名称不被截断的修复引入到 0.6 中。

    参考资料：[#1309](https://www.sqlalchemy.org/trac/ticket/1309)

### 杂项

+   **[扩展]**

    由 associationproxy 生成的集合代理现在可以进行 pickle 化。然而，用户定义的 proxy_factory 除非定义了 __getstate__ 和 __setstate__，否则仍然不能 pickle 化。

    参考资料：[#1446](https://www.sqlalchemy.org/trac/ticket/1446)

+   **[扩展]**

    如果将 __table_args__ 作为没有字典参数的元组传递，Declarative 将引发一个提供信息的异常。改进了文档。

    参考资料：[#1468](https://www.sqlalchemy.org/trac/ticket/1468)

+   **[扩展]**

    在 MetaData 中声明的 Table 对象现在可以在发送到 primaryjoin/secondaryjoin/secondary 的字符串表达式中使用 - 名称从声明基类的 MetaData 中提取。

    参考资料：[#1527](https://www.sqlalchemy.org/trac/ticket/1527)

+   **[扩展]**

    在构造类之后（即通过类级属性分配），可以向连接表子类添加列。该列始终添加到底层表中，但现在映射器将重建其“join”以包括新列，而不是引发关于“没有这样的列，请改用 column_property()”的错误。

    参考：[#1523](https://www.sqlalchemy.org/trac/ticket/1523)

+   **[测试]**

    在测试套件中添加了示例，以便定期执行，并清理了一些弃用警告。

### ORM

+   **[ORM]**

    修复了一个问题，即继承的复合主键的鉴别器部分在更新时会失败。继续。

    参考：[#1300](https://www.sqlalchemy.org/trac/ticket/1300)

+   **[ORM]**

    修复了一个问题，该问题不允许双向多对多引用的一侧声明自己为“viewonly”。

    参考：[#1507](https://www.sqlalchemy.org/trac/ticket/1507)

+   **[ORM]**

    添加了一个断言，防止@validates 函数或其他 AttributeExtension 加载未加载的集合，从而可能损坏内部状态。

    参考：[#1526](https://www.sqlalchemy.org/trac/ticket/1526)

+   **[ORM]**

    修复了一个问题，该问题阻止两个实体在单个 flush()中相互替换主键值的某些操作顺序。

    参考：[#1519](https://www.sqlalchemy.org/trac/ticket/1519)

+   **[ORM]**

    修复了一个晦涩的问题，即具有基类上自引用贪婪加载的连接表子类将使用来自父类“子类”表的数据填充相关对象的“子类”表。

    参考：[#1485](https://www.sqlalchemy.org/trac/ticket/1485)

+   **[ORM]**

    relations()现在具有更大的“覆盖”能力，这意味着明确指定覆盖父类的关系()的子类将在 flush 期间受到尊重。目前，这是为了支持具体继承设置的多对多关系。除此用例外，效果可能有所不同。

    参考：[#1477](https://www.sqlalchemy.org/trac/ticket/1477)

+   **[ORM]**

    从 relation()中挤出了更多不必要的“懒加载”。当集合发生变化时，另一侧的多对一反向引用不会触发加载“旧”值，除非设置了“single_parent=True”。直接分配一个多对一仍然会加载“旧”值，以便更新该值上的反向引用集合，该值可能已经存在于会话中，从而保持 0.5 行为契约。

    参考：[#1483](https://www.sqlalchemy.org/trac/ticket/1483)

+   **[ORM]**

    修复了一个问题，即基于 column_property()或类似属性的连接表继承属性的加载/刷新将无法评估。

    参考：[#1480](https://www.sqlalchemy.org/trac/ticket/1480)

+   **[ORM]**

    改进了对 MapperProperty 对象覆盖非具体继承设置的继承映射器的支持 - 属性扩展不会随机冲突。

    参考：[#1488](https://www.sqlalchemy.org/trac/ticket/1488)

+   **[orm]**

    标准 SQL 不支持 UPDATE 和 DELETE 中的 ORDER BY、LIMIT、OFFSET 等。如果调用了 limit()、offset()、order_by()、group_by()或 distinct()中的任何一个，Query.update()和 Query.delete()现在会引发异常。

    参考：[#1487](https://www.sqlalchemy.org/trac/ticket/1487)

+   **[orm]**

    将 AttributeExtension 添加到 sqlalchemy.orm.__all__

+   **[orm]**

    当使用非 SQL /实体表达式调用 query()时，改进了错误消息。

    参考：[#1476](https://www.sqlalchemy.org/trac/ticket/1476)

+   **[orm]**

    在基类和子类上使用 False 或 0 作为多态鉴别器现在也可以正常工作。

    参考：[#1440](https://www.sqlalchemy.org/trac/ticket/1440)

+   **[orm]**

    在 Query 中添加了 enable_assertions(False)，用于禁用通常的对预期状态的断言 - 由 Query 子类用于设计自定义状态。参见[`www.sqlalchemy.org/trac/wiki/UsageRecipes/PreFilteredQuery`](https://www.sqlalchemy.org/trac/wiki/UsageRecipes/PreFilteredQuery)以获取示例。

    参考：[#1424](https://www.sqlalchemy.org/trac/ticket/1424)

+   **[orm]**

    修复了递归问题，如果映射对象的 __len__()或 __nonzero__()方法导致状态更改，则会发生。

    参考：[#1501](https://www.sqlalchemy.org/trac/ticket/1501)

+   **[orm]**

    修复了 Weak/StrongIdentityMap.add()中不正确的异常抛出。

    参考：[#1506](https://www.sqlalchemy.org/trac/ticket/1506)

+   **[orm]**

    修复了在 query.join()中“找不到 FROM 子句”错误消息，如果查询针对纯 SQL 构造，则无法正确发出。

    参考：[#1522](https://www.sqlalchemy.org/trac/ticket/1522)

+   **[orm]**

    修复了一个有点假设的问题，即使用旧的 polymorphic_union 函数计算映射器的错误主键 - 但这是旧的东西。

    参考：[#1486](https://www.sqlalchemy.org/trac/ticket/1486)

### sql

+   **[sql]**

    修复了 column.copy()以复制默认值和 onupdates。

    参考：[#1373](https://www.sqlalchemy.org/trac/ticket/1373)

+   **[sql]**

    修复了在 0.5.4 中引入的 extract()中的错误，其中字符串“field”参数被视为 ClauseElement，导致更复杂的 SQL 转换中出现各种错误。

+   **[sql]**

    一元表达式（如 DISTINCT）将其类型处理传播到结果集，允许进行类似 unicode 的转换。

    参考：[#1420](https://www.sqlalchemy.org/trac/ticket/1420)

+   **[sql]**

    修复了 Table 和 Column 中传递空字典给“info”参数会引发异常的 bug。

    参考：[#1482](https://www.sqlalchemy.org/trac/ticket/1482)

### oracle

+   **[oracle]**

    为 Oracle 别名名称未被截断的 0.6 修复进行了回溯。

    参考：[#1309](https://www.sqlalchemy.org/trac/ticket/1309)

### 杂项

+   **[ext]**

    associationproxy 生成的集合代理现在是可 pickle 的。然而，用户定义的 proxy_factory 仍然不可 pickle，除非它定义了 __getstate__ 和 __setstate__。

    参考：[#1446](https://www.sqlalchemy.org/trac/ticket/1446)

+   **[ext]**

    如果将 __table_args__ 作为没有字典参数的元组传递给 Declarative，将引发一个信息性异常。改进了文档。

    参考：[#1468](https://www.sqlalchemy.org/trac/ticket/1468)

+   **[ext]**

    在 MetaData 中声明的 Table 对象现在可以在发送到 primaryjoin/secondaryjoin/secondary 的字符串表达式中使用 - 名称从 declarative base 的 MetaData 中提取。

    参考：[#1527](https://www.sqlalchemy.org/trac/ticket/1527)

+   **[ext]**

    在构造类后（即通过类级属性赋值）可以向 joined-table 子类添加列。该列始终添加到底层表中，但现在映射器将重建其“join”以包括新列，而不是引发关于“没有这样的列，请改用 column_property()”的错误。

    参考：[#1523](https://www.sqlalchemy.org/trac/ticket/1523)

+   **[test]**

    将示例添加到测试套件中，以便定期执行，并清理了一些弃用警告。

## 0.5.5

发布日期：2009 年 7 月 13 日星期一

### 一般

+   **[general]**

    单元测试已从 unittest 迁移到 nose。有关如何运行测试的信息，请参阅 README.unittests。

    参考：[#970](https://www.sqlalchemy.org/trac/ticket/970)

### orm

+   **[orm]**

    relation()的“foreign_keys”参数现在将自动传播到同一 backref 中，就像 primaryjoin 和 secondaryjoin 一样。对于极为罕见的情况，即 relation()的 backref 有意不同的“foreign_keys”配置，现在两侧都需要显式配置（如果它们确实需要此设置，请参阅下一个注释…）。

+   **[orm]**

    …唯一已知的（而且真的非常罕见）使用情况是，在前向/后向方面使用了不同的 foreign_keys 设置，部分指向自身列的复合外键已经得到增强，以便关系的 fk->itself 方面不会用于确定关系方向。

+   **[orm]**

    Session.mapper 现在已被*弃用*。

    如果您希望一个独立的对象成为会话的一部分，请调用 session.add()。否则，现在在[`www.sqlalchemy.org/trac/wiki/UsageRecipes/SessionAwareMapper`](https://www.sqlalchemy.org/trac/wiki/UsageRecipes/SessionAwareMapper)上记录了 Session.mapper 的 DIY 版本。该方法将在 0.6 版本中继续被弃用。

+   **[orm]**

    修复了 Query 能够从 joined-table 子类实体的单独列（即 query(SubClass.foo, SubClass.bar).join(<anything>)）进行 join()的问题。在大多数情况下，会引发错误“找不到要加入的 FROM 子句”。在少数情况下，结果将以基类而不是子类的形式返回 - 因此依赖于这种错误结果的应用程序需要进行调整。

    参考：[#1431](https://www.sqlalchemy.org/trac/ticket/1431)

+   **[orm]**

    修复了涉及 contains_eager()的错误，该错误会在特定情况下将其应用于次要（即懒加载）加载，从而产生笛卡尔积。改进了对次要加载上的 query.options()的定位。

    参考：[#1461](https://www.sqlalchemy.org/trac/ticket/1461)

+   **[orm]**

    修复了 0.5.4 版本引入的 bug，即当默认保持列被刷新时，复合类型会失败。

+   **[orm]**

    修复了另一个 0.5.4 版本的 bug，即当整个对象被序列化时，可变属性（即 PickleType）无法正确反序列化。

    参考：[#1426](https://www.sqlalchemy.org/trac/ticket/1426)

+   **[orm]**

    修复了 session.is_modified()在使用任何同义词时会引发异常的 bug。

+   **[orm]**

    修复了潜在的内存泄漏问题，即以前被 pickled 的对象放回会话后，除非显式关闭 Session，否则不会完全被垃圾回收。

+   **[orm]**

    修复了基于列表的属性（如 pickletype 和 PGArray）未能正确合并的 bug。

+   **[orm]**

    修复了 attributes.set_committed_value 函数无法正常工作的问题。

+   **[orm]**

    修剪了 InstanceState 的 pickle 格式，这应进一步减少 pickled 实例的内存占用。该格式应向后兼容 0.5.4 及以前的版本。

+   **[orm]**

    sqlalchemy.orm.join 和 sqlalchemy.orm.outerjoin 现在添加到 sqlalchemy.orm.*的 __all__ 中。

    参考：[#1463](https://www.sqlalchemy.org/trac/ticket/1463)

+   **[orm]**

    修复了当传递给 get()的复合主键值过短时，Query 异常引发失败的 bug。

    参考：[#1458](https://www.sqlalchemy.org/trac/ticket/1458)

### sql

+   **[sql]**

    移除了 execute()的一个晦涩特性（包括 connection、engine、Session），即可以将 bindparam()构造作为 params 字典的键发送。这种用法未记录在案，并且是一个问题的核心，即 text()构造隐式创建的 bindparam()对象可能具有与放置在 params 字典中的字符串相同的哈希值，并且在计算最终绑定参数时可能导致不适当的匹配。对于可能使用此功能的任何应用程序来说，这是一个向后不兼容的更改，但是该功能从未被记录在案。

### misc

+   **[engine/pool]**

    为 StaticPool 实现了 recreate()。

### general

+   **[general]**

    单元测试已从 unittest 迁移到 nose。有关如何运行测试的信息，请参阅 README.unittests。

    参考：[#970](https://www.sqlalchemy.org/trac/ticket/970)

### orm

+   **[orm]**

    relation()的“foreign_keys”参数现在会自动传播到 backref，就像 primaryjoin 和 secondaryjoin 一样。对于极为罕见的情况，即 relation()的 backref 有意配置了不同的“foreign_keys”，现在两侧都需要显式配置（如果它们确实需要这个设置，请参见下一个注释…）。

+   **[orm]**

    …唯一已知的（而且真的非常罕见）使用不同 foreign_keys 设置在前向/后向方向上的情况，部分指向自己列的复合外键，已经增强，使得关系的 fk->itself 方面不会用于确定关系方向。

+   **[orm]**

    Session.mapper 现在已经*弃用*。

    如果您希望一个独立的对象成为会话的一部分，请调用 session.add()。否则，现在在[`www.sqlalchemy.org/trac/wiki/UsageRecipes/SessionAwareMapper`](https://www.sqlalchemy.org/trac/wiki/UsageRecipes/SessionAwareMapper)上记录了 Session.mapper 的 DIY 版本。该方法将在整个 0.6 版本中保持弃用。

+   **[orm]**

    修复了 Query 能够从连接表子类实体的单个列进行 join()的 bug，即 query(SubClass.foo, SubClass.bar).join(<anything>)。在大多数情况下，会引发错误“找不到要加入的 FROM 子句”。在少数情况下，结果将以基类而不是子类的形式返回，因此依赖于这个错误结果的应用程序需要进行调整。

    参考：[#1431](https://www.sqlalchemy.org/trac/ticket/1431)

+   **[orm]**

    修复了涉及 contains_eager()的 bug，在一个特定罕见情况下会应用于次要（即懒加载）加载，产生笛卡尔积。改进了对次要加载上的 query.options()的定位。

    参考：[#1461](https://www.sqlalchemy.org/trac/ticket/1461)

+   **[orm]**

    修复了 0.5.4 中引入的 bug，即当刷新默认保持列时，复合类型会失败。

+   **[orm]**

    修复了 0.5.4 中的另一个 bug，即可变属性（即 PickleType）在整个对象被序列化时无法正确反序列化。

    参考：[#1426](https://www.sqlalchemy.org/trac/ticket/1426)

+   **[orm]**

    修复了 session.is_modified()在使用任何同义词时会引发异常的 bug。

+   **[orm]**

    修复了潜在的内存泄漏 bug，即以前 pickled 的对象重新放入会话中时，除非显式关闭 Session，否则不会完全被垃圾回收。

+   **[orm]**

    修复了基于列表的属性（如 pickletype 和 PGArray）未能正确合并()的 bug。

+   **[orm]**

    修复了不起作用的 attributes.set_committed_value 函数。

+   **[orm]**

    修剪了 InstanceState 的 pickle 格式，这应该进一步减少 pickled 实例的内存占用。该格式应该向后兼容 0.5.4 及之前的版本。

+   **[orm]**

    sqlalchemy.orm.join 和 sqlalchemy.orm.outerjoin 现在已添加到 sqlalchemy.orm.*的 __all__ 中。

    参考：[#1463](https://www.sqlalchemy.org/trac/ticket/1463)

+   **[orm]**

    修复了当向 get()传递太短的复合主键值时，Query 异常引发失败的错误。

    参考：[#1458](https://www.sqlalchemy.org/trac/ticket/1458)

### sql

+   **[sql]**

    移除了 execute()的一个晦涩特性（包括 connection、engine、Session），其中 bindparam()构造可以作为 params 字典的键发送。这种用法未记录在案，并且是一个问题的核心，即 text()构造隐式创建的 bindparam()对象可能具有与放置在 params 字典中的字符串相同的哈希值，并且在计算最终绑定参数时可能导致不恰当的匹配。对于这种情况的内部检查将显著增加关键任务参数渲染的延迟，因此移除了该行为。对于可能使用此功能的任何应用程序，这是一个不兼容的更改，但是该功能从未被记录。

### 杂项

+   **[engine/pool]**

    为 StaticPool 实现了 recreate()。

## 0.5.4p2

发布日期：2009 年 5 月 26 日星期二

### sql

+   **[sql]**

    修复了不基于参数或不是 executemany()风格的 SQL 异常打印。

### postgresql

+   **[postgresql]**

    弃用了硬编码的 TIMESTAMP 函数，当作为 func.TIMESTAMP(value)使用时，会呈现“TIMESTAMP value”。这在某些平台上会出现问题，因为 PostgreSQL 不允许在此上下文中使用绑定参数。硬编码的大写也是不合适的，我们需要支持很多其他 PG 转换。因此，使用文本构造，即 select([“timestamp ‘12/05/09’”])。

### sql

+   **[sql]**

    修复了不基于参数或不是 executemany()风格的 SQL 异常打印。

### postgresql

+   **[postgresql]**

    弃用了硬编码的 TIMESTAMP 函数，当作为 func.TIMESTAMP(value)使用时，会呈现“TIMESTAMP value”。这在某些平台上会出现问题，因为 PostgreSQL 不允许在此上下文中使用绑定参数。硬编码的大写也是不合适的，我们需要支持很多其他 PG 转换。因此，使用文本构造，即 select([“timestamp ‘12/05/09’”])。

## 0.5.4p1

发布日期：2009 年 5 月 18 日星期一

### orm

+   **[orm]**

    修复了 0.5.4 中引入的属性错误，当使用不完整对象进行 merge()时会发生。

### orm

+   **[orm]**

    修复了 0.5.4 中引入的属性错误，当使用不完整对象进行 merge()时会发生。

## 0.5.4

发布日期：2009 年 5 月 17 日星期日

### orm

+   **[orm]**

    在与大型映射器图、大量对象一起使用时，关于 Sessions/flush()的显著性能增强：

    +   从 flush()过程中移除了所有* O(N)扫描行为，即扫描整个会话的操作，包括一个极其昂贵的操作，错误地假设主键值正在更改，而实际情况并非如此。

        +   仍然存在一个边缘情况，可能会调用完整扫描，如果现有的主键属性被修改为新值。

    +   会话的“弱引用”行为现在是*完全*的 - 对映射对象或相关项/集合在其 __dict__ 中不再进行任何强引用。对象中的反向引用和其他循环不再影响会话丢失对未修改对象的所有引用的能力。具有待处理更改的对象仍然会在 flush 之前被强制保留。

        该实现还通过将垃圾回收项的“复活”过程仅适用于映射“可变”属性（即 PickleType、复合属性）来提高性能。这消除了 gc 过程的开销，并简化了内部行为。

        如果“可变”属性更改是对象上唯一的更改，然后该对象被取消引用，那么在发出 UPDATE 时，映射器将无法访问其他属性状态。这可能会对某些 MapperExtensions 产生不同的影响。

        此更改还影响了内部属性 API，但不影响 AttributeExtension 接口或任何公开文档化的属性函数。

    +   工作单元在 flush()期间不再为整个映射器图生成“依赖”处理器图，而是仅为表示具有待处理更改的对象的那些映射器创建这样的处理器。在大型相互连接的映射器图的上下文中，这节省了大量的方法调用。

    +   缓存了以前在每次 flush 时多次发生的“表排序”操作，还从 flush()中删除了大量的方法调用次数。

    +   在 mapper._save_obj()中简化了其他冗余行为。

    参考：[#1398](https://www.sqlalchemy.org/trac/ticket/1398)

+   **[orm]**

    修改了 DynamicAttributeImpl 上的 query_cls，以接受 AppenderQuery 的完整 mixin 版本，这允许对 AppenderMixin 进行子类化。

+   **[orm]**

    “多态鉴别器”列可能是主键的一部分，并且将填充正确的鉴别器值。

    参考：[#1300](https://www.sqlalchemy.org/trac/ticket/1300)

+   **[orm]**

    修复了评估器无法评估 IS NULL 子句的问题。

+   **[orm]**

    修复了“dynamic”关系上的“set collection”函数以正确启动事件。以前，只能将集合分配给待处理的父实例，否则修改事件将无法正确触发。现在，set collection 与 merge()兼容，修复了问题。

    参考：[#1352](https://www.sqlalchemy.org/trac/ticket/1352)

+   **[orm]**

    允许对使用 instrumented descriptors 构造的 PropertyOption 对象进行 pickle；以前，在对加载了基于描述符的选项的对象进行 pickle 时会出现 pickle 错误，例如 query.options(eagerload(MyClass.foo))。

+   **[orm]**

    如果“lazy load” SQL 子句与 get()使用的子句匹配，但包含一些硬编码参数，则惰性加载器将不使用 get()。以前，惰性策略会因为 get()失败。理想情况下，应该使用带有硬编码参数的 get()，但这需要进一步开发。

    参考：[#1357](https://www.sqlalchemy.org/trac/ticket/1357)

+   **[orm]**

    现在，与 query.options()关联的 MapperOptions 和其他状态不再在每次加载期间与每个惰性/延迟加载属性相关联的可调用对象中捆绑。这些选项现在与实例的状态对象关联一次，当它被填充时。这在大多数情况下消除了每个实例/属性加载器对象的需要，提高了单个实例的加载速度和内存开销。

    参考：[#1391](https://www.sqlalchemy.org/trac/ticket/1391)

+   **[orm]**

    修复了另一个位置，其中自动刷新干扰了 session.merge()。现在在 merge()期间完全禁用了 autoflush。

    参考：[#1360](https://www.sqlalchemy.org/trac/ticket/1360)

+   **[orm]**

    修复了阻止“可变主键”依赖逻辑在一对一 relation()上正常运行的错误。

    参考：[#1406](https://www.sqlalchemy.org/trac/ticket/1406)

+   **[orm]**

    修复了 relation()中的错误，该错误在 0.5.3 中引入，其中从基类到连接表子类的自引用关系不会正确配置。

+   **[orm]**

    修复了继承映射器使用时的模糊编译问题，这会导致未初始化的属性。

+   **[orm]**

    修复了 session weak_identity_map 的文档错误-默认值为 True，表示正在使用弱引用映射。

+   **[orm]**

    修复了一个工作单元问题，即在要删除的对象所拥有的集合中的项目上的外键属性如果 relation()是自引用的，则不会设置为 None。

    参考：[#1376](https://www.sqlalchemy.org/trac/ticket/1376)

+   **[orm]**

    修复了 Query.update()和 Query.delete()在急加载关系中的失败。

    参考：[#1378](https://www.sqlalchemy.org/trac/ticket/1378)

+   **[orm]**

    现在，在 foreign_keys 或 remote_side 集合中指定二进制主连接条件的两列是一个错误。而以前只是荒谬的，但以一种非确定性的方式成功。

### sql

+   **[sql]**

    从 SQLA 0.6 中回溯了“compiler”扩展。这是一个标准化接口，允许创建自定义 ClauseElement 子类和编译器。特别是作为 text()的替代方案很方便，当您想要构建具有数据库特定编译的构造时。有关详细信息，请参阅扩展文档。

+   **[sql]**

    当绑定参数列表大于 10 时，异常消息会被截断，防止大量的多页异常填满屏幕和大型 executemany()语句的日志文件。

    参考：[#1413](https://www.sqlalchemy.org/trac/ticket/1413)

+   **[sql]**

    `sqlalchemy.extract()` 现在是方言敏感的，并且可以在支持的数据库中惯用地提取时间戳组件，包括 SQLite。

+   **[sql]**

    修复了从 __clause_element__() 样式构造的 ForeignKey 上的 __repr__() 和其他 _get_colspec() 方法（即声明性列）。

    参考：[#1353](https://www.sqlalchemy.org/trac/ticket/1353)

### schema

+   **[schema] [1341] [ticket: 594]**

    向 IdentifierPreparer 类添加了 quote_schema() 方法，以便方言可以重写模式的处理方式。这使得 MSSQL 方言可以将模式视为多部分标识符，例如 'database.owner'。

### extensions

+   **[extensions]**

    修复了将延迟或其他列属性添加到声明性类的问题。

    参考：[#1379](https://www.sqlalchemy.org/trac/ticket/1379)

### mysql

+   **[mysql]**

    反射 FOREIGN KEY 构造将考虑点分模式.表名组合，如果外键引用远程模式中的表。

    参考：[#1405](https://www.sqlalchemy.org/trac/ticket/1405)

### sqlite

+   **[sqlite]**

    修正了 SLBoolean 类型，使其正确地将只有 1 视为 True。

    参考：[#1402](https://www.sqlalchemy.org/trac/ticket/1402)

+   **[sqlite]**

    修正了 float 类型，使其在反射时正确地映射到 SLFloat 类型。

    参考：[#1273](https://www.sqlalchemy.org/trac/ticket/1273)

### mssql

+   **[mssql]**

    修改了保存点逻辑的工作方式，以防止其干扰非保存点导向的例程。保存点支持仍然非常实验性。

+   **[mssql]**

    添加了适用于 MSSQL 的保留字，涵盖了 2008 年及以前的所有版本。

    参考：[#1310](https://www.sqlalchemy.org/trac/ticket/1310)

+   **[mssql]**

    修正了信息模式在基于二进制排序的数据库上不起作用的问题。清理了信息模式，因为它现在只由 mssql 使用。

    参考：[#1343](https://www.sqlalchemy.org/trac/ticket/1343)

### orm

+   **[orm]**

    在与大型映射器图、大量对象一起使用时，针对 Sessions/flush() 的显著性能增强：

    +   从 flush() 过程中删除了所有* O(N) 扫描行为，即扫描整个会话的操作，包括一个极其昂贵的操作，错误地假定主键值正在更改，而事实并非如此。

        +   仍然存在一个边缘情况，可能会触发完全扫描，如果现有的主键属性被修改为新值。

    +   会话的“弱引用”行为现在是*完整的* - 在其 __dict__ 中不再对映射对象或相关项目/集合进行任何强引用。对象中的反向引用和其他循环不再影响会话完全失去对未修改对象的所有引用的能力。具有待处理更改的对象仍然会强烈保留，直到 flush。

        该实现还通过将垃圾回收项的“复活”过程仅与映射“可变”属性（即 PickleType、复合属性）相关的映射器相关联，从而提高了性能。这消除了 gc 过程的开销，并简化了内部行为。

        如果“可变”属性更改是对象上唯一的更改，然后该对象被取消引用，那么在发出 UPDATE 时，映射器将无法访问其他属性状态。这可能会对某些 MapperExtensions 产生不同的影响。

        这一变化也影响了内部属性 API，但不影响 AttributeExtension 接口，也不影响任何公开文档中记录的属性函数。

    +   工作单元在 flush() 期间不再为整个映射器图生成“依赖”处理器图，而是仅为表示具有待处理更改的对象的那些映射器创建这样的处理器。这在大型互连映射器图的情况下节省了大量方法调用。

    +   优化了以前每次刷新都会发生多次“表排序”操作的性能浪费，同时还从 flush() 中删除了大量方法调用次数。

    +   其他冗余行为已经在 mapper._save_obj() 中简化。

    参考：[#1398](https://www.sqlalchemy.org/trac/ticket/1398)

+   **[orm]**

    修改了 DynamicAttributeImpl 上的 query_cls，以接受 AppenderQuery 的完整混合版本，这允许对 AppenderMixin 进行子类化。

+   **[orm]**

    “多态鉴别器”列可以是主键的一部分，并且将填充正确的鉴别器值。

    参考：[#1300](https://www.sqlalchemy.org/trac/ticket/1300)

+   **[orm]**

    修复了评估器无法评估 IS NULL 子句的问题。

+   **[orm]**

    修复了在“动态”关系上的“设置集合”函数无法正确触发事件的问题。以前，只能将集合分配给待处理的父实例，否则修改事件将无法正确触发。现在，设置集合与 merge() 兼容，修复了这个问题。

    参考：[#1352](https://www.sqlalchemy.org/trac/ticket/1352)

+   **[orm]**

    允许对使用 instrumented descriptors 构建的 PropertyOption 对象进行 pickling；以前，在 pickle 一个使用基于描述符的选项（例如 query.options(eagerload(MyClass.foo)）加载的对象时，会出现 pickle 错误。

+   **[orm]**

    如果“延迟加载”SQL 子句与 get() 使用的子句匹配，但包含一些硬编码参数，则惰性加载器将不使用 get()。以前，惰性策略会在 get() 中失败。理想情况下，应该使用带有硬编码参数的 get()，但这需要进一步开发。

    参考：[#1357](https://www.sqlalchemy.org/trac/ticket/1357)

+   **[orm]**

    MapperOptions 和与 query.options()关联的其他状态不再在每次加载期间与每个延迟/延迟加载属性相关的可调用对象捆绑在一起。现在，选项仅在填充实例时与实例的状态对象关联一次。这在大多数情况下消除了每个实例/属性加载器对象的需要，提高了单个实例的加载速度和内存开销。

    参考：[#1391](https://www.sqlalchemy.org/trac/ticket/1391)

+   **[orm]**

    修复了另一个位置，其中 autoflush 干扰了 session.merge()。现在，在 merge()期间完全禁用 autoflush。

    参考：[#1360](https://www.sqlalchemy.org/trac/ticket/1360)

+   **[orm]**

    修复了阻止“可变主键”依赖逻辑在一对一关系()上正常运行的错误。

    参考：[#1406](https://www.sqlalchemy.org/trac/ticket/1406)

+   **[orm]**

    修复了在 0.5.3 版本中引入的 relation()中的错误，即从基类到连接表子类的自引用关系将无法正确配置。

+   **[orm]**

    修复了当使用继承映射器时会导致未初始化属性的模糊映射器编译问题。

+   **[orm]**

    修复了 session weak_identity_map 的文档 - 默认值为 True，表示正在使用弱引用映射。

+   **[orm]**

    修复了一个工作单元问题，即在要删除的对象拥有的集合中的项目上的外键属性如果关系()是自引用的，则不会设置为 None。

    参考：[#1376](https://www.sqlalchemy.org/trac/ticket/1376)

+   **[orm]**

    修复了 Query.update()和 Query.delete()在预加载关系时的失败。

    参考：[#1378](https://www.sqlalchemy.org/trac/ticket/1378)

+   **[orm]**

    现在，在 foreign_keys 或 remote_side 集合中指定二进制 primaryjoin 条件的两个列是一个错误。而以前只是荒谬的，但会以一种非确定性的方式成功。

### sql

+   **[sql]**

    从 SQLA 0.6 版本中回溯了“编译器”扩展。这是一个标准化接口，允许创建自定义的 ClauseElement 子类和编译器。特别是当您想要构建具有特定于数据库的编译的构造时，它是 text()的一个替代方案。有关详细信息，请参阅扩展文档。

+   **[sql]**

    当绑定参数列表大于 10 时，异常消息将被截断，防止大型 executemany()语句填满屏幕和日志文件。

    参考：[#1413](https://www.sqlalchemy.org/trac/ticket/1413)

+   **[sql]**

    `sqlalchemy.extract()`现在是方言敏感的，并且可以在支持的数据库中习惯地提取时间戳的组件，包括 SQLite。

+   **[sql]**

    修复了由 __clause_element__()样式构造（即声明列）构造的 ForeignKey 上的 __repr__()和其他 _get_colspec()方法。

    参考：[#1353](https://www.sqlalchemy.org/trac/ticket/1353)

### schema

+   **[模式] [1341] [票号：594]**

    添加了一个 quote_schema() 方法到 IdentifierPreparer 类，以便方言可以覆盖如何处理模式。这使得 MSSQL 方言可以将模式视为多部分标识符，例如 'database.owner'。

### **[扩展]**

+   **[扩展]**

    修复了将延迟或其他列属性添加到声明类的问题。

    参考：[#1379](https://www.sqlalchemy.org/trac/ticket/1379)

### mysql

+   **[mysql]**

    反射 FOREIGN KEY 构造将考虑到点分模式.表名组合，如果外键引用远程模式中的表。

    参考：[#1405](https://www.sqlalchemy.org/trac/ticket/1405)

### **[sqlite]**

+   **[sqlite]**

    修正了 SLBoolean 类型，使其正确地将只有 1 视为 True。

    参考：[#1402](https://www.sqlalchemy.org/trac/ticket/1402)

+   **[sqlite]**

    修正了 float 类型，使其在反射时正确映射为 SLFloat 类型。

    参考：[#1273](https://www.sqlalchemy.org/trac/ticket/1273)

### mssql

+   **[mssql]**

    修改了保存点逻辑的工作方式，以防止它干扰非保存点导向的例程。保存点支持仍然是非常实验性的。

+   **[mssql]**

    添加了 MSSQL 的保留字，涵盖了 2008 版本和所有之前的版本。

    参考：[#1310](https://www.sqlalchemy.org/trac/ticket/1310)

+   **[mssql]**

    修正了与基于二进制排序的数据库不兼容的信息模式的问题。清理了信息模式，因为现在只有 mssql 在使用。

    参考：[#1343](https://www.sqlalchemy.org/trac/ticket/1343)

## 0.5.3

发布日期：2009 年 3 月 24 日 星期二

### ORM

+   **[ORM]**

    对于 session.flush() 的 “objects” 参数已被弃用。表示父对象和子对象之间链接的状态不支持在链接的一侧处于“flushed”状态而在另一侧不是，因此支持此操作会导致误导性的结果。

    参考：[#1315](https://www.sqlalchemy.org/trac/ticket/1315)

+   **[ORM]**

    Query 现在实现了 __clause_element__()，它产生其可选择的，这意味着 Query 实例可以在许多 SQL 表达式中被接受，包括 col.in_(query)，union(query1, query2)，select([foo]).select_from(query) 等。

+   **[ORM]**

    Query.join() 现在可以构造多个 FROM 子句，如果需要的话。例如，query(A, B).join(A.x).join(B.y) 可能会生成 SELECT A.*, B.* FROM A JOIN X, B JOIN Y。Eager loading 也可以将其连接附加到这些多个 FROM 子句上。

    参考：[#1337](https://www.sqlalchemy.org/trac/ticket/1337)

+   **[ORM]**

    修复了 dynamic_loader() 中的 bug，其中在构建后未传播追加/移除事件到 UOW 以进行 flush()。

    参考：[#1347](https://www.sqlalchemy.org/trac/ticket/1347)

+   **[ORM]**

    修复了在未检查 column_prefix 的情况下不映射已经存在类级别名称的属性的 bug。

+   **[ORM]**

    对特定集合属性的 session.expire()也会清除任何挂起的反向引用添加，以便下一次访问正确地只返回数据库中存在的内容。虽然我们正在考虑完全删除 flush([objects])功能，但也提供了一定程度的解决方案。

    参考：[#1315](https://www.sqlalchemy.org/trac/ticket/1315)

+   **[orm]**

    Session.scalar()现在将原始 SQL 字符串转换为 text()，与 Session.execute()相同，并接受相同的替代**kw args。

+   **[orm]**

    改进了 relation()的“确定方向”逻辑，以便可以确定复杂情况的方向，比如 mapper(A.join(B)) -> relation-> mapper(B)。

+   **[orm]**

    使用 session.flush([somelist])刷新对象的部分集合时，操作后保留的挂起对象不会不经意地被添加为持久对象。

    参考：[#1306](https://www.sqlalchemy.org/trac/ticket/1306)

+   **[orm]**

    添加了“post_configure_attribute”方法到 InstrumentationManager，以便“listen_for_events.py”示例再次起作用。

    参考：[#1314](https://www.sqlalchemy.org/trac/ticket/1314)

+   **[orm]**

    发现了前向和补充的反向引用，两者都是相同方向的，即 ONETOMANY 或 MANYTOONE，并引发了错误消息。以后可以避免疯狂的 CircularDependencyErrors。

+   **[orm]**

    修复了 Query 在同时选择具有相同基类的多个联接表继承实体时的错误：

    +   以前对“A JOIN B”上的“B”的适应会错误地部分应用到“A”上。

    +   在关系比较（即 A.related==someb）时，当应该进行适应时没有适应。

    +   其他过滤，比如 query(A).join(A.bs).filter(B.foo=='bar')，错误地将“B.foo”适应为“A”。

+   **[orm]**

    修复了在与左侧的别名对象和 of_type()右侧一起使用 EXISTS 子句的适应。

    参考：[#1325](https://www.sqlalchemy.org/trac/ticket/1325)

+   **[orm]**

    在 sqlalchemy.orm.attributes 中添加了一个属性助手方法`set_committed_value`。给定对象、属性名称和值，将值设置在对象上作为其“已提交”状态的一部分，即被理解为从数据库加载的状态。有助于创建自制集合加载器等。

+   **[orm]**

    当传递非映射器/类的被仪器化描述符时，查询不会因为弱引用错误而失败，而是引发“无效的列表达式”。

+   **[orm]**

    Query.group_by()正确考虑了应用于 FROM 子句的别名，比如 select_from()，使用 with_polymorphic()，或使用 from_self()。

### sql

+   **[sql]**

    select()的 alias()在明确的标量上下文中使用时将转换为“标量子查询”，即它在比较操作中使用。当使用 query.subquery()时，这也适用于 ORM。

+   **[sql]**

    在 select() 中使用 use_labels（例如在 ORM column_property() 中使用时）时，Function 对象上的 _label 属性等属性不会丢失。

    参考：[#1302](https://www.sqlalchemy.org/trac/ticket/1302)

+   **[sql]**

    匿名别名现在会截断到方言允许的最大长度。在像 Oracle 这样具有非常小字符限制的数据库上更为重要。

    参考：[#1309](https://www.sqlalchemy.org/trac/ticket/1309)

+   **[sql]**

    __selectable__() 接口已完全被 __clause_element__() 取代。

+   **[sql]**

    TypeEngine 用于缓存特定于方言的类型的每个方言缓存现在是 WeakKeyDictionary。这是为了防止方言对象被引用，对于创建任意数量的引擎或方言的应用程序，这会永远存在。这会带来一些性能损失，将在 0.6 中解决。

    参考：[#1299](https://www.sqlalchemy.org/trac/ticket/1299)

### extensions

+   **[extensions]**

    修复了序列化器中的递归 pickling 问题，由 EXISTS 或其他嵌入的 FROM 构造触发。

+   **[extensions]**

    Declarative 使用 __bases__ 中的搜索来定位 “inherits” 类，以跳过局部于子类的混入。

+   **[extensions]**

    即使显式给出 “inherits” 映射器参数，Declarative 也会找出联接表继承的主要联接条件。

+   **[extensions]**

    如果 backref() 的 “foreign_keys” 参数是字符串，Declarative 将正确解释它。

+   **[extensions]**

    当与 __table__ 结合使用时，Declarative 将接受一个绑定到表的列作为属性，如果该列已经存在于 __table__ 中。该列将被重新映射到给定的键，方式与添加到 mapper() 属性字典时相同。

### postgresql

+   **[postgresql]**

    当遇到具有多个表达式的索引时，索引反射不会失败。

+   **[postgresql]**

    向 sqlalchemy.databases.postgres 添加了 PGUuid 和 PGBit 类型。

    参考：[#1327](https://www.sqlalchemy.org/trac/ticket/1327)

+   **[postgresql]**

    当这些类型在域内指定时，未知 PG 类型的反射不会崩溃。

    参考：[#1327](https://www.sqlalchemy.org/trac/ticket/1327)

### sqlite

+   **[sqlite]**

    修复了 SQLite 反射方法，以便在最近版本的 pysqlite 上检测到不存在的 cursor.description，触发自动关闭游标，以便在调用 fetchone() 时没有行时不会失败。

### mssql

+   **[mssql]**

    对 pymssql 1.0.1 的初步支持

+   **[mssql]**

    修正了在 mssql 上未遵守 max_identifier_length 的问题。

### orm

+   **[orm]**

    session.flush() 的 “objects” 参数已被弃用。表示父对象和子对象之间链接的状态不支持链接的一侧处于 “flushed�� 状态而另一侧不是，因此支持此操作会导致误导性的结果。

    参考：[#1315](https://www.sqlalchemy.org/trac/ticket/1315)

+   **[orm]**

    Query 现在实现了 __clause_element__()，它生成其可选择的内容，这意味着 Query 实例可以在许多 SQL 表达式中被接受，包括 col.in_(query)，union(query1, query2)，select([foo]).select_from(query)等。

+   **[orm]**

    Query.join()现在可以构造多个 FROM 子句，如果需要的话。例如，query(A, B).join(A.x).join(B.y)可能会生成 SELECT A.*, B.* FROM A JOIN X, B JOIN Y。Eager loading 也可以将其连接附加到这些多个 FROM 子句上。

    参考：[#1337](https://www.sqlalchemy.org/trac/ticket/1337)

+   **[orm]**

    修复了 dynamic_loader()中在构造后追加/删除事件未传播到 UOW 以在 flush()中捕获的 bug。

    参考：[#1347](https://www.sqlalchemy.org/trac/ticket/1347)

+   **[orm]**

    修复了在未检查 column_prefix 之前未映射已具有类级名称的属性的 bug。

+   **[orm]**

    对特定集合属性进行 session.expire()将清除任何待定的反向引用添加，以便下一次访问正确地返回仅在数据库中存在的内容。对于 flush([objects])功能的某种程度的解决方案，尽管我们正在考虑完全删除它。

    参考：[#1315](https://www.sqlalchemy.org/trac/ticket/1315)

+   **[orm]**

    Session.scalar()现在将原始 SQL 字符串转换为 text()，与 Session.execute()相同，并接受相同的替代**kw 参数。

+   **[orm]**

    改进了 relation()的“确定方向”逻辑，以便可以确定类似 mapper(A.join(B)) -> relation-> mapper(B)这样的棘手情况的方向。

+   **[orm]**

    使用 session.flush([somelist])刷新部分对象集时，操作后仍保持待定状态的对象不会被意外添加为持久对象。

    参考：[#1306](https://www.sqlalchemy.org/trac/ticket/1306)

+   **[orm]**

    添加了“post_configure_attribute”方法到 InstrumentationManager，以便“listen_for_events.py”示例再次起作用。

    参考：[#1314](https://www.sqlalchemy.org/trac/ticket/1314)

+   **[orm]**

    现在检测到具有相同方向的前向和补充后向引用，即 ONETOMANY 或 MANYTOONE，并引发错误消息。以后可以避免疯狂的 CircularDependencyErrors。

+   **[orm]**

    修复了 Query 中关于同时选择具有共同基类的多个连接表继承实体的 bug：

    +   以前对“A JOIN B”中的“B”的适应会错误地部分应用于“A”。

    +   对关系的比较（即 A.related==someb）在应该适应时未被适应。

    +   其他过滤，如 query(A).join(A.bs).filter(B.foo==’bar’)，错误地将“B.foo”适应为“A”。

+   **[orm]**

    修复了通过 any()，has()等与左侧别名对象和右侧 of_type()结合使用的 EXISTS 子句的适应。

    参考：[#1325](https://www.sqlalchemy.org/trac/ticket/1325)

+   **[orm]**

    在 sqlalchemy.orm.attributes 中添加了一个属性助手方法`set_committed_value`。给定一个对象、属性名称和值，将在对象上设置该值作为其“已提交”状态的一部分，即从数据库加载的状态。有助于创建自制集合加载器等。

+   **[orm]**

    当传递非映射器/类的受仪器化描述符时，查询不会因弱引用错误而失败，而会引发“无效列表达式”。

+   **[orm]**

    Query.group_by()正确考虑应用于 FROM 子句的别名，例如使用 select_from()、使用 with_polymorphic()或使用 from_self()。

### sql

+   **[sql]**

    当在明确的标量上下文中使用时，select()的 alias()将转换为“标量子查询”，即在比较操作中使用。当在 ORM 中使用 query.subquery()时也适用。

+   **[sql]**

    在使用 use_labels（例如在 ORM column_property()中使用时）的 select()中使用 Function 对象时，修复了缺少的 _label 属性等。

    参考：[#1302](https://www.sqlalchemy.org/trac/ticket/1302)

+   **[sql]**

    匿名别名现在会截断到方言允许的最大长度。在像 Oracle 这样具有非常小字符限制的数据库上更为重要。

    参考：[#1309](https://www.sqlalchemy.org/trac/ticket/1309)

+   **[sql]**

    __selectable__()接口已完全被 __clause_element__()取代。

+   **[sql]**

    TypeEngine 用于缓存特定于方言的类型的每方言缓存现在是 WeakKeyDictionary。这是为了防止方言对象被引用，以便应用程序创建任意数量的引擎或方言。这会有一点性能损失，将在 0.6 中解决。

    参考：[#1299](https://www.sqlalchemy.org/trac/ticket/1299)

### extensions

+   **[extensions]**

    修复了序列化器中的递归 pickling 问题，由 EXISTS 或其他嵌入的 FROM 构造触发。

+   **[extensions]**

    Declarative 通过 __bases__ 搜索定位“inherits”��，以跳过子类本地的混入。

+   **[extensions]**

    即使明确给出“inherits”映射器参数，声明性图形也会找出连接表继承的主连接条件。

+   **[extensions]**

    如果“foreign_keys”参数是字符串，Declarative 将正确解释 backref()上的“foreign_keys”参数。

+   **[extensions]**

    当与 __table__ 一起使用时，如果列已经存在于 __table__ 中，Declarative 将接受绑定到表的列作为属性。该列将被重新映射到给定键，方式与添加到 mapper()属性字典时相同。

### postgresql

+   **[postgresql]**

    当遇到具有多个表达式的索引时，索引反射不会失败。

+   **[postgresql]**

    在 sqlalchemy.databases.postgres 中添加了 PGUuid 和 PGBit 类型。

    参考：[#1327](https://www.sqlalchemy.org/trac/ticket/1327)

+   **[postgresql]**

    当在域中指定未知 PG 类型时，未知 PG 类型的反射不会崩溃。

    参考：[#1327](https://www.sqlalchemy.org/trac/ticket/1327)

### sqlite

+   **[sqlite]**

    修复了 SQLite 反射方法，以便检测到不存在的 cursor.description，触发自动关闭游标，这样在最近版本的 pysqlite 上调用 fetchone() 时不会因没有结果而失败，后者在没有行时会引发错误。

### mssql

+   **[mssql]**

    对 pymssql 1.0.1 的初步支持

+   **[mssql]**

    修复了在 mssql 上未正确处理 max_identifier_length 的问题。

## 0.5.2

发布日期：2009 年 1 月 24 日 星期六

### orm

+   **[orm]**

    进一步完善了 0.5.1 版本关于在多对多关系上放置 delete-orphan 级联的警告。首先，坏消息是：警告将适用于多对多关系和多对一关系。这是必要的，因为在这两种情况下，SQLA 在确定“孤儿”状态时不会扫描所有潜在的父对象集合 - 对于持久对象，它只会检测到一个在 Python 中的取消关联事件来确定对象是否为“孤儿”。接下来，好消息是：为了支持通过外键或关联表实现一对一，或者通过关联表支持一对多，可以设置一个新标志 single_parent=True，表示链接到关系的对象只能有一个父对象。如果在 Python 中发生多个父关联事件，关系将引发错误。

+   **[orm]**

    调整了从 0.5.1 版本开始的属性检测变化，完全为在超类已完全被检测后创建的子类建立属性检测。

    参考：[#1292](https://www.sqlalchemy.org/trac/ticket/1292)

+   **[orm]**

    修复了 delete-orphan 级联中的错误，即从两个不同的父类到相同目标类的两个一对一关系会过早地清除实例。

+   **[orm]**

    修复了一个急加载错误，即自引用急加载会阻止其他急加载（自引用或非自引用）正确地加入到父 JOIN 中。感谢 Alex K 创建了一个很好的测试用例。

+   **[orm]**

    session.expire() 和相关方法不会使未加载的延迟属性过期。这样可以防止在刷新实例时不必要地加载它们。

+   **[orm]**

    query.join()/outerjoin() 现在将正确地将一个 aliased() 构造加入到现有的左侧，即使已调用 query.from_self() 或 query.select_from(someselectable)。

    参考：[#1293](https://www.sqlalchemy.org/trac/ticket/1293)

### sql

+   **[sql]**

    进一步修复了“列/表中的百分号和空格

    names” 功能。

    参考：[#1284](https://www.sqlalchemy.org/trac/ticket/1284)

### mssql

+   **[mssql]**

    恢复了 convert_unicode 处理。结果被传递而没有转换。

    参考：[#1291](https://www.sqlalchemy.org/trac/ticket/1291)

+   **[mssql]**

    这次真的修复了十进制处理..

    参考：[#1282](https://www.sqlalchemy.org/trac/ticket/1282)

+   **[mssql] [工单:1289]**

    修改了表反射代码，只使用 kwargs 构建表。

### orm

+   **[orm]**

    进一步改进了 0.5.1 版本关于在多对多关系上放置的 delete-orphan 级联的警告。首先，坏消息是：该警告将适用于多对多关系和多对一关系。这是必要的，因为在这两种情况下，SQLA 在确定“孤儿”状态时并不会扫描全部潜在的父对象集合 - 对于持久对象，它只会检测到一个在 Python 中的取消关联事件以确立对象为“孤儿”。接下来是好消息：为了支持通过外键或关联表进行一对一，或者通过关联表进行一对多，可以设置一个新的标志 `single_parent=True`，表示与关系相关联的对象只能有一个父对象。如果 Python 中发生多个父关联事件，则该关系将引发错误。

+   **[orm]**

    调整了从 0.5.1 版本开始的属性检测变更，以完全为在超类已经完全被检测后创建的子类建立属性检测。

    参考：[#1292](https://www.sqlalchemy.org/trac/ticket/1292)

+   **[orm]**

    修复了删除孤立级联中的错误，其中两个从两个不同的父类到相同目标类的一对一关系会过早地删除实例。

+   **[orm]**

    修复了一个贪婪加载 bug，即自我引用的贪婪加载将阻止其他贪婪加载（自我引用或非自我引用）正确地加入到父 JOIN 中。感谢 Alex K 创建了一个很好的测试用例。

+   **[orm]**

    `session.expire()` 和相关方法将不会过期未加载的延迟属性。这样在刷新实例时就不会被不必要地加载。

+   **[orm]**

    `query.join()/outerjoin()` 现在将正确地将一个 `aliased()` 构造加入到现有的左侧，即使已调用了 `query.from_self()` 或 `query.select_from(someselectable)`。

    参考：[#1293](https://www.sqlalchemy.org/trac/ticket/1293)

### sql

+   **[sql]**

    进一步修复了在列/表中的百分号和空格。

    names” 功能。

    参考：[#1284](https://www.sqlalchemy.org/trac/ticket/1284)

### mssql

+   **[mssql]**

    恢复了 `convert_unicode` 处理。结果将不会转换而被传递。

    参考：[#1291](https://www.sqlalchemy.org/trac/ticket/1291)

+   **[mssql]**

    这次真正修复了十进制处理问题。

    参考：[#1282](https://www.sqlalchemy.org/trac/ticket/1282)

+   **[mssql] [Ticket:1289]**

    修改了表反射代码，只使用 kwargs 构建表。

## 0.5.1

发布日期：Sat Jan 17 2009

### orm

+   **[orm]**

    移除了一个内部连接缓存，当重复发出 query.join() 到 ad-hoc 可选择时，可能会泄漏内存。

+   **[orm]**

    “clear()”、“save()”、“update()”、“save_or_update()” Session 方法已被弃用，取而代之的是 “expunge_all()” 和 “add()”。 “expunge_all()” 也已添加到 `ScopedSession` 中。

+   **[orm]**

    现代化了“无映射表”异常，并在声明中添加了更明确的 `__table__/__tablename__` 异常。

+   **[orm]**

    具体继承的映射器现在会为从超类继承但未为具体映射器本身定义的属性实例化一个发出描述性错误的 InstrumentedAttribute。

    参考：[#1237](https://www.sqlalchemy.org/trac/ticket/1237)

+   **[orm]**

    添加了一个新的 relation() 关键字 back_populates。这允许使用显式关系配置反向引用。在具体映射器的层次结构和另一个类之间创建双向关系时，这是必需的。

    参考：[#1237](https://www.sqlalchemy.org/trac/ticket/1237), [#781](https://www.sqlalchemy.org/trac/ticket/781)

+   **[orm]**

    为在具体映射器上指定的 relation() 对象添加了测试覆盖。

    参考：[#1237](https://www.sqlalchemy.org/trac/ticket/1237)

+   **[orm]**

    Query.from_self() 以及 query.subquery() 都会禁用生成的子查询内部的 eager joins 渲染���通过新的 query.enable_eagerloads() 生成器，公开提供“禁用所有 eager joins”功能。

    参考：[#1276](https://www.sqlalchemy.org/trac/ticket/1276)

+   **[orm]**

    在 Query 中添加了一系列基本的集合操作，接受 Query 对象作为参数，包括 union()、union_all()、intersect()、except_()、intersect_all()、except_all()。请参阅 Query.union() 的 API 文档以获取示例。

+   **[orm]**

    修复了阻止 Query.join() 和 eagerloads 附加到从联合或别名联合选择的查询的 bug。

+   **[orm]**

    为在具体映射器上指定的双向关系添加了一个简短的文档示例。

    参考：[#1237](https://www.sqlalchemy.org/trac/ticket/1237)

+   **[orm]**

    现在，在构造时，Mappers 会使用最终的 InstrumentedAttribute 对象对类属性进行实例化，该对象保持持久性。_CompileOnAttr/__getattribute__() 方法论已被移除。其净效果是，基于 Column 的映射类属性现在可以在类级别完全使用，而不需要调用映射编译操作，大大简化了声明中的典型使用模式。

    参考：[#1269](https://www.sqlalchemy.org/trac/ticket/1269)

+   **[orm]**

    ColumnProperty（以及前端辅助程序，如 `deferred`）不再忽略未知的关键字参数。

+   **[orm]**

    修复了与 unitofwork 的“行切换”机制相关的 bug，即将 INSERT/DELETE 转换为 UPDATE，当与联接表继承和一个对象结合时，该对象对于子表没有定义值，将渲染一个没有 SET 子句的 UPDATE。

+   **[orm]**

    在多对多关系上使用 delete-orphan 已被弃用。这会产生误导性或错误的结果，因为 SQLA 不会检索 m2m 的完整“父级”列表。要在 m2m 表上获得 delete-orphan 行为，请使用显式关联类，以便将单个关联行视为父级。

    参考：[#1281](https://www.sqlalchemy.org/trac/ticket/1281)

+   **[orm]**

    delete-orphan 级联始终需要 delete 级联。指定 delete-orphan 而不删除现在会引发弃用警告。

    参考：[#1281](https://www.sqlalchemy.org/trac/ticket/1281)

### sql

+   **[SQL]**

    改进了处理列名中百分号的方法。增加了更多测试。MySQL 和 PostgreSQL 方言仍然无法正确发出带有百分号的标识符的 CREATE TABLE 语句。

    参考：[#1256](https://www.sqlalchemy.org/trac/ticket/1256)

### 模式

+   **[模式]**

    Index 现在接受基于列的 InstrumentedAttributes（即基于列的映射类属性）作为列参数。

    参考：[#1214](https://www.sqlalchemy.org/trac/ticket/1214)

+   **[模式]**

    列没有名称（如在声明式中），当请求其字符串输出时不会引发 NoneType 错误（例如在堆栈跟踪中）。

+   **[模式]**

    修复了在反射表上覆盖具有 ForeignKey 的列时的 bug，其中派生列（即 select 的“虚拟”列等）会无意中调用仅用于原始列的模式级清理逻辑。

    参考：[#1278](https://www.sqlalchemy.org/trac/ticket/1278)

### mysql

+   **[mysql]**

    添加了 MySQL 4.1 中缺失的关键字，以便正确转义它们。

### mssql

+   **[mssql]**

    更正了对大十进制值的处理，增加了更健壮的测试。移除了对浮点数的字符串操作。

    参考：[#1280](https://www.sqlalchemy.org/trac/ticket/1280)

+   **[mssql]**

    修改了 mssql 中的 do_begin 处理，使用 Cursor 而不是 Connection，以使其与 DBAPI 兼容。

+   **[mssql]**

    通过更改对 savepoint_release 的处理方式，修正了 adodbapi 上的 SAVEPOINT 支持，因为 mssql 不支持。

### 杂项

+   **[声明式]**

    现在可以在没有自己表的子类上指定 Column 对象（即使用单一表继承）。这些列将附加到基本表，但仅由子类映射。

+   **[声明式]**

    对于连接和单一继承子类，子类只会映射那些已在超类上映射的列以及子类上显式指定的列。默认情况下，表中存在的其他列将被排除在映射之外，可以通过将空的 exclude_properties 集合传递给 __mapper_args__ 来禁用此功能。这样，定义自己列的单一继承类将是唯一映射这些列的类。实际效果比通常使用显式 mapper() 调用获得的映射更有组织性，除非您显式设置 exclude_properties 参数。

+   **[声明式]**

    向声明式类添加新的 Column 对象，而该类使用 __table__ 指定了现有表，这是一个错误。

### ORM

+   **[ORM]**

    移除了一个内部连接缓存，当对 ad-hoc 可选择对象重复调用 query.join() 时可能会泄漏内存。

+   **[ORM]**

    “clear()”、“save()”、“update()”、“save_or_update()” Session 方法已被弃用，被“expunge_all()”和“add()”取代。“expunge_all()”也已添加到 ScopedSession。

+   **[orm]**

    现代化了“no mapped table”异常，并在 declarative 中添加了一个更明确的 __table__/__tablename__ 异常。

+   **[orm]**

    现在，具体继承的映射器会对从超类继承的但对于具体映射器本身未定义的属性进行仪器化，使用一个发出描述性错误的 InstrumentedAttribute 当被访问时。

    参考：[#1237](https://www.sqlalchemy.org/trac/ticket/1237)

+   **[orm]**

    添加了一个新的 relation()关键字 back_populates。这允许使用显式关系配置反向引用。在创建具体映射器层次结构和另一个类之间的双向关系时，这是必需的。

    参考：[#1237](https://www.sqlalchemy.org/trac/ticket/1237)，[#781](https://www.sqlalchemy.org/trac/ticket/781)

+   **[orm]**

    为在具体映射器上指定的 relation()对象添加了测试覆盖。

    参考：[#1237](https://www.sqlalchemy.org/trac/ticket/1237)

+   **[orm]**

    Query.from_self()以及 query.subquery()都会禁用生成的子查询内部的急切连接的渲染。通过新的 query.enable_eagerloads()生成器，公开提供“禁用所有急切连接”的功能。

    参考：[#1276](https://www.sqlalchemy.org/trac/ticket/1276)

+   **[orm]**

    在 Query 中添加了一系列基本的集合操作，接受 Query 对象作为参数，包括 union()、union_all()、intersect()、except_()、intersect_all()、except_all()。查看 Query.union()的 API 文档以获取示例。

+   **[orm]**

    修复了一个 bug，该 bug 阻止了 Query.join()和 eagerloads 附加到从联合或别名联合中选择的查询。

+   **[orm]**

    为在具体映射器上指定的双向关系添加了一个简短的文档示例。

    参考：[#1237](https://www.sqlalchemy.org/trac/ticket/1237)

+   **[orm]**

    现在，映射器在构造时会使用最终的 InstrumentedAttribute 对象对类属性进行仪器化，该对象保持持久。_CompileOnAttr/__getattribute__()方法论已被移除。其净效果是，基于列的映射类属性现在可以在类级别完全使用，而不需要调用映射器编译操作，大大简化了 declarative 中的典型使用模式。

    参考：[#1269](https://www.sqlalchemy.org/trac/ticket/1269)

+   **[orm]**

    ColumnProperty（以及前端助手如`deferred`）不再忽略未知的关键字参数。

+   **[orm]**

    修复了与 unitofwork 的“行切换”机制的 bug，即将 INSERT/DELETE 转换为 UPDATE 时，与联接表继承和一个不包含子表定义值的对象结合使用时，将渲染出一个没有 SET 子句的 UPDATE。

+   **[orm]**

    在多对多关系上使用 delete-orphan 已被弃用。这会产生误导或错误的结果，因为 SQLA 不会检索 m2m 的“父级”完整列表。要在 m2m 表上获得 delete-orphan 行为，请使用显式关联类，以便将单个关联行视为父级。

    参考：[#1281](https://www.sqlalchemy.org/trac/ticket/1281)

+   **[ORM]**

    delete-orphan 级联总是需要 delete 级联。指定 delete-orphan 而不指定 delete 现在会引发弃用警告。

    参考：[#1281](https://www.sqlalchemy.org/trac/ticket/1281)

### SQL

+   **[SQL]**

    改进了处理列名中百分号的方法。添加了更多测试。MySQL 和 PostgreSQL 方言仍无法为其中带有百分号的标识符发出正确的 CREATE TABLE 语句。

    参考：[#1256](https://www.sqlalchemy.org/trac/ticket/1256)

### 模式

+   **[模式]**

    Index 现在接受基于列的 InstrumentedAttributes（即基于列的映射类属性）作为列参数。

    参考：[#1214](https://www.sqlalchemy.org/trac/ticket/1214)

+   **[模式]**

    在没有名称的列（如在声明式中）请求其字符串输出时不会引发 NoneType 错误（例如在堆栈跟踪中）。

+   **[模式]**

    修复了在反射表上覆盖具有外键的列时的 bug，其中派生列（即 select 的“虚拟”列等）会无意中调用原始列专用的模式级清理逻辑。

    参考：[#1278](https://www.sqlalchemy.org/trac/ticket/1278)

### mysql

+   **[mysql]**

    添加了来自 MySQL 4.1 的缺失关键字，以便正确转义。

### mssql

+   **[mssql]**

    更正了对大十进制值的处理，增加了更健壮的测试。移除了对浮点数的字符串操作。

    参考：[#1280](https://www.sqlalchemy.org/trac/ticket/1280)

+   **[mssql]**

    修改了在 mssql 中处理 do_begin 的方法，使用 Cursor 而不是 Connection，以使其与 DBAPI 兼容。

+   **[mssql]**

    通过更改 savepoint_release 的处理方式来更正 adodbapi 上的 SAVEPOINT 支持，因为 mssql 不支持该功能。

### 杂项

+   **[声明式]**

    现在可以在没有自己表的子类上指定 Column 对象（即使用单表继承）。这些列将附加到基本表，但只由子类映射。

+   **[声明式]**

    对于连接和单一继承子类，子类只会映射那些已在超类上映射的列以及子类上显式指定的列。表中存在的其他列默认将被排除在映射之外，可以通过将空的 exclude_properties 集合传递给 __mapper_args__ 来禁用此功能。这样，定义自己列的单一继承类将是唯一映射这些列的类。实际效果比通常使用显式 mapper() 调用获得的映射更有组织性，除非你显式设置了 exclude_properties 参数。

+   **[声明式]**

    向指定了现有表使用 __table__ 的声明类添加新的 Column 对象是错误的。

## 0.5.0

发布日期：2009 年 1 月 6 日星期二

### 一般

+   **[general]**

    文档已转换为 Sphinx。特别是，生成的 API 文档已构建为完整的“API 参考”部分，其中组织了编辑文档和生成的文档字符串。各个部分和 API 文档之间的交叉链接得到了极大改善，提供了一个基于 JavaScript 的搜索功能，并提供了所有类、函数和成员的完整索引。

+   **[general]**

    setup.py 现在仅在可选情况下导入 setuptools。如果不存在，将使用 distutils。新的“pip”安装程序建议使用 easy_install，因为它以更简化的方式安装。 

+   **[general]**

    在示例文件夹中添加了一个极其基本的 PostGIS 集成示例。

### orm

+   **[orm]**

    Query.with_polymorphic()现在接受第三个参数“鉴别器”，该参数将替换该查询的 mapper.polymorphic_on 的值。即使映射器具有多态标识，也不再需要设置多态标识。当未设置时，默认情况下，映射器将以非多态方式加载。这两个特性共同允许非多态具体继承设置在每个查询基础上使用多态加载，因为在所有情况下使用多态的具体设置容易出现许多问题。

+   **[orm]**

    dynamic_loader 接受 query_class=来自定义用于动态集合和从中构建的查询的 Query 类。

+   **[orm]**

    query.order_by()接受 None，这将从查询中删除任何待定的 order_by 状态，同时取消任何映射器/关系配置的排序。这主要用于覆盖在 dynamic_loader()上指定的排序。

    参考：[#1079](https://www.sqlalchemy.org/trac/ticket/1079)

+   **[orm]**

    在 compile_mappers()期间引发的异常现在被保留以提供“粘性行为” - 如果对预编译映射属性的 hasattr()调用触发失败的编译并抑制异常，则后续编译将被阻止，并且异常将在下一次 compile()调用时被重复。在使用 declarative 时，这个问题经常发生。

+   **[orm]**

    property.of_type()现在在单表继承目标上被识别，当在 prop.of_type(..).any()/has()的上下文中使用时，以及 query.join(prop.of_type(…))。

+   **[orm]**

    当连接的目标与基于属性的属性不匹配时，query.join()会引发错误 - 虽然不太可能有人这样做，但 SQLAlchemy 的作者曾犯过这种特定的松散行为。

+   **[orm]**

    修复了在使用 weak_instance_map=False 时，修改事件不会被拦截以进行 flush()的 bug。

    参考：[#1272](https://www.sqlalchemy.org/trac/ticket/1272)

+   **[orm]**

    修复了一些可能影响针对包含同一表的多个版本的可选择项进行的查询的深层“列对应”问题，以及包含不同列位置的相同表列的联合等问题，这些问题在不同级别上。

    参考：[#1268](https://www.sqlalchemy.org/trac/ticket/1268)

+   **[orm]**

    与 column_property()，relation()等一起使用的自定义比较器类可以在比较器上定义新的比较方法，这些方法将通过 InstrumentedAttribute 上的 __getattr__()变得可用。在 synonym()或 comparable_property()的情况下，首先在用户定义的描述符上解析属性，然后在用户定义的比较器上解析属性。

+   **[orm]**

    添加了 ScopedSession.is_active 访问器。

    参考：[#976](https://www.sqlalchemy.org/trac/ticket/976)

+   **[orm]**

    可以将映射属性和列对象作为键传递给 query.update({})。

    参考：[#1262](https://www.sqlalchemy.org/trac/ticket/1262)

+   **[orm]**

    传递给表达式级别 insert()或 update()的 values()的映射属性将使用映射列的键，而不是映射属性的键��

+   **[orm]**

    修正了 Query.delete()和 Query.update()与绑定参数不正常工作的问题。

    参考：[#1242](https://www.sqlalchemy.org/trac/ticket/1242)

+   **[orm]**

    Query.select_from()，from_statement()确保给定的参数是 FromClause，或 Text/Select/Union，分别。

+   **[orm]**

    Query()可以将“composite”属性作为列表达式传递，并将其展开。与某种程度相关。

    参考：[#1253](https://www.sqlalchemy.org/trac/ticket/1253)

+   **[orm]**

    当传递各种列表达式（如字符串，clauselists，text()构造）时，Query()更加健壮（这可能意味着它只是更好地引发错误）。

+   **[orm]**

    first()与 Query.from_statement()一起按预期工作。

+   **[orm]**

    修复了 0.5rc4 中引入的有关延迟加载对使用 add_property()或等效方法在编译后向映射器添加属性的属性不起作用的错误。

+   **[orm]**

    修复了当 viewonly=True 时，许多对多关系（many-to-many relation()）无法正确引用 secondary->remote 之间链接的错误。

+   **[orm]**

    在向“secondary”表中的“many-to-many”关系发出 INSERT 时，列表型集合中的重复项将被保留。假设 m2m 表上有唯一或主键约束，这将引发预期的约束违规，而不是悄悄删除重复条目。请注意，对于一对多关系，旧行为仍然保留，因为在这种情况下，集合条目不会导致 INSERT 语句，并且 SQLA 不会手动监视集合。

    参考：[#1232](https://www.sqlalchemy.org/trac/ticket/1232)

+   **[orm]**

    Query.add_column()可以像 session.query()一样接受 FromClause 对象。

+   **[orm]**

    将对多对一关系的比较与 NULL 正确转换为基于 not_()的 IS NOT NULL。

+   **[orm]**

    添加额外检查以确保显式的 primaryjoin/secondaryjoin 是 ClauseElement 实例，以防止后续出现更令人困惑的错误。

    参考：[#1087](https://www.sqlalchemy.org/trac/ticket/1087)

+   **[orm]**

    改进了对非类类的 mapper()检查。

    参考：[#1236](https://www.sqlalchemy.org/trac/ticket/1236)

+   **[orm]**

    comparator_factory 参数现在已记录并受到所有 MapperProperty 类型的支持，包括 column_property()、relation()、backref()和 synonym()。

    参考：[#5051](https://www.sqlalchemy.org/trac/ticket/5051)

+   **[orm]**

    将 PropertyLoader 的名称更改为 RelationProperty，以与所有其他名称保持一致。PropertyLoader 仍然存在作为同义词。

+   **[orm]**

    修复了在 shard API 中导致总线错误的“双 iter()”调用，删除了 0.4 版本中遗留的错误 result.close()。

    参考：[#1099](https://www.sqlalchemy.org/trac/ticket/1099), [#1228](https://www.sqlalchemy.org/trac/ticket/1228)

+   **[orm]**

    使 Session.merge 级联不触发自动刷新。修复了合并实例过早插入丢失值的问题。

+   **[orm]**

    为了防止在多态联合继承场景中渲染出现额外列（导致在 FROM 子句中渲染额外表，导致笛卡尔积）的两个修复：

    > +   对于 a->b->c 继承情况的“列适应”进行了改进，以更好地定位通过多级间接关系相关的列，而不是呈现未适应的列。
    > +   
    > +   “多态鉴别器”列仅针对实际查询的 mapper 进行呈现。该列不会从子类或超类 mapper 中“拉入”，因为这是不需要的。

+   **[orm]**

    修复了 ShardedSession.execute()中的 shard_id 参数。

    参考：[#1072](https://www.sqlalchemy.org/trac/ticket/1072)

### sql

+   **[sql]**

    RowProxy 对象可以用于替代发送给 connection.execute()和其它函数的字典参数。

    参考：[#935](https://www.sqlalchemy.org/trac/ticket/935)

+   **[sql]**

    列名中再次可以包含百分号。

    参考：[#1256](https://www.sqlalchemy.org/trac/ticket/1256)

+   **[sql]**

    sqlalchemy.sql.expression.Function 现在是一个公共类。可以对其进行子类化，以提供用户定义的 SQL 函数，采用命令式风格，包括预先建立的行为。postgis.py 示例说明了其中一种用法。

+   **[sql]**

    PickleType 现在默认偏爱==比较，如果传入对象（如 dict）实现了 __eq__()。如果对象没有实现 __eq__()并且 mutable=True，则会引发弃用警告。

+   **[sql]**

    修复了 sqlalchemy.sql 中导出 __names__ 的奇怪问题。

    参考：[#1215](https://www.sqlalchemy.org/trac/ticket/1215)

+   **[sql]**

    多次使用相同的 ForeignKey 对象会引发错误，而不是在稍后默默失败。

    参考：[#1238](https://www.sqlalchemy.org/trac/ticket/1238)

+   **[sql]**

    为 Insert/Update/Delete 构造的 params() 方法添加了 NotImplementedError。这些项目目前不支持此功能，这也会与 values() 相比有点误导。

+   **[sql]**

    反射外键将正确定位其引用的列，即使该列具有与反射名称不同的“key”属性。这是通过 ForeignKey/ForeignKeyConstraint 上的一个名为“link_to_name”的新标志实现的，如果为 True，则表示给定名称是被引用列的名称，而不是其分配的键。

    参考：[#650](https://www.sqlalchemy.org/trac/ticket/650)

+   **[sql]**

    select() 可以接受 ClauseList 作为列，方式与 Table 或其他可选择的方式相同，内部表达式将用作列元素。

    参考：[#1253](https://www.sqlalchemy.org/trac/ticket/1253)

+   **[sql]**

    session.is_modified() 上的“passive”标志正确传播到属性管理器。

+   **[sql]**

    union() 和 union_all() 不会破坏应用于 select() 内部的 order_by()。如果您将带有 order_by() 的 select() 进行 union()（可能是为了支持 LIMIT/OFFSET），您还应该对其调用 self_group() 以应用括号。

### mysql

+   **[mysql]**

    在 text() 构造中的“%”符号会自动转义为“%%”。由于这种变化具有向后不兼容的性质，如果在字符串中检测到‘%%’，则会发出警告。

+   **[mysql]**

    修复了在反射期间未出现 FK 列时引发异常的错误。

    参考：[#1241](https://www.sqlalchemy.org/trac/ticket/1241)

+   **[mysql]**

    修复了反射远程模式表时出现的问题，该表具有对该模式中另一表的外键引用。

### sqlite

+   **[sqlite]**

    现在表反射会存储列的实际 DefaultClause ���。

    参考：[#1266](https://www.sqlalchemy.org/trac/ticket/1266)

+   **[sqlite]**

    修复了错误，行为更改

### mssql

+   **[mssql]**

    添加了新的 MSGenericBinary 类型。这将映射到 Binary 类型，以便实现将长度指定类型视为固定宽度的 Binary 类型，而非长度类型视为无限长度的 Binary 类型的专门行为。

+   **[mssql]**

    添加了新类型：MSVarBinary 和 MSImage。

    参考：[#1249](https://www.sqlalchemy.org/trac/ticket/1249)

+   **[mssql]**

    添加了 MSReal、MSNText、MSSmallDateTime、MSTime、MSDateTimeOffset 和 MSDateTime2 类型

+   **[mssql]**

    重构了日期/时间类型。`smalldatetime` 数据类型不再截断为仅日期，现在将映射到 MSSmallDateTime 类型。

    参考：[#1254](https://www.sqlalchemy.org/trac/ticket/1254)

+   **[mssql]**

    修正了接受 int 的 Numerics 的问题。

+   **[mssql]**

    将 `char_length` 映射到 `LEN()` 函数。

+   **[mssql]**

    如果 `INSERT` 包含子选择，则将 `INSERT` 从 `INSERT INTO VALUES` 构造转换为 `INSERT INTO SELECT` 构造。

+   **[mssql]**

    如果列是 `primary_key` 的一部分，则它将是 `NOT NULL`，因为 MSSQL 不允许在主键列中使用 `NULL`。

+   **[mssql]**

    `MSBinary`现在返回`BINARY`而不是`IMAGE`。这是一个不兼容的变化，因为`BINARY`是一个固定长度数据类型，而`IMAGE`是一个可变长度数据类型。

    参考：[#1249](https://www.sqlalchemy.org/trac/ticket/1249)

+   **[mssql]**

    `get_default_schema_name`现在根据用户的默认模式从数据库中反映��这仅适用于 MSSQL 2005 及更高版本。

    参考：[#1258](https://www.sqlalchemy.org/trac/ticket/1258)

+   **[mssql]**

    通过使用新的 collation 参数添加了排序规则支持。此功能支持以下类型：char，nchar，varchar，nvarchar，text，ntext。

    参考：[#1248](https://www.sqlalchemy.org/trac/ticket/1248)

+   **[mssql]**

    对连接字符串参数的更改支持 DSN 作为 pyodbc 的默认规范。有关详细使用说明，请参阅 mssql.py 文档字符串。

+   **[mssql]**

    添加了对保存点的实验性支持。目前与会话不完全兼容。

+   **[mssql]**

    支持三种列空值性质的级别：NULL，NOT NULL 和数据库配置的默认值。默认的列配置（nullable=True）现在将在 DDL 中生成 NULL。以前没有发出规范，数据库默认值会生效（通常为 NULL，但并非总是）。要显式请求数据库默认值，请使用 nullable=None 配置列，DDL 中将不会发出规范。这是不兼容的行为。

    参考：[#1243](https://www.sqlalchemy.org/trac/ticket/1243)

### oracle

+   **[oracle]**

    调整了 create_xid()的格式以修复两阶段提交。我们现在有关于 Oracle 两阶段提交正常工作的现场报告。

+   **[oracle]**

    添加了 OracleNVarchar 类型，生成 NVARCHAR2，并且默认情况下也继承 Unicode，因此 convert_unicode=True。NVARCHAR2 会自动反映到这种类型，因此这些列在没有显式 convert_unicode=True 标志的反射表上传递 unicode。

    参考：[#1233](https://www.sqlalchemy.org/trac/ticket/1233)

+   **[oracle]**

    修复了一个 bug，该 bug 阻止了某些类型的 out 参数被接收；非常感谢 wwu.edu 上的 huddlej！

    参考：[#1265](https://www.sqlalchemy.org/trac/ticket/1265)

### 杂项

+   **[dialect]**

    在 dialect 上添加了一个新的 description_encoding 属性，用于在处理元数据时对列名进行编码。通常默认为 utf-8。

+   **[engine/pool]**

    Connection.invalidate()检查关闭状态以避免属性错误。

    参考：[#1246](https://www.sqlalchemy.org/trac/ticket/1246)

+   **[engine/pool]**

    NullPool 支持失败时重新连接的行为。

    参考：[#1094](https://www.sqlalchemy.org/trac/ticket/1094)

+   **[engine/pool]**

    在使用 pool.manage(dbapi)时为初始池创建添加了互斥锁。这可以防止在启动时出现一种轻微的“堆积”行为。

    参考：[#799](https://www.sqlalchemy.org/trac/ticket/799)

+   **[engine/pool]**

    _execute_clauseelement()重新成为私有方法。现在有了 ConnectionProxy，不再需要对 Connection 进行子类化。

+   **[documentation]**

    Tickets.

    参考：[#1149](https://www.sqlalchemy.org/trac/ticket/1149), [#1200](https://www.sqlalchemy.org/trac/ticket/1200)

+   **[documentation]**

    添加了关于 create_session()默认值的说明。

+   **[documentation]**

    添加了关于 metadata.reflect()的部分。

+   **[documentation]**

    更新了 TypeDecorator 部分。

+   **[documentation]**

    由于最近对这一功能的混淆，重写了文档中关于“threadlocal”策略的部分。

+   **[documentation]**

    从继承中删除了非常过时的‘polymorphic_fetch’和‘select_table’文档，重新制作了“联接表继承”的后半部分。

+   **[documentation]**

    记录了 comparator_factory 关键字参数，添加了新的文档部分“自定义比较器”。

+   **[postgres]**

    在 text()构造中的“%”符号会自动转义为“%%”。由于这种变化的不兼容性，如果在字符串中检测到‘%%’，则会发出警告。

    参考：[#1267](https://www.sqlalchemy.org/trac/ticket/1267)

+   **[postgres]**

    在使用 server_side_cursors 时，调用 alias.execute()不会引发 AttributeError。

+   **[postgres]**

    添加了对 PostgreSQL 的索引反射支持，使用了我们长期忽视的一个很棒的补丁，由 Ken Kuhlman 提交。

    参考：[#714](https://www.sqlalchemy.org/trac/ticket/714)

+   **[associationproxy]**

    关联代理属性现在在类级别上可用，例如 MyClass.aproxy。以前这会评估为 None。

+   **[declarative]**

    backref()接受的作为字符串的参数完整列表包括‘primaryjoin’, ‘secondaryjoin’, ‘secondary’, ‘foreign_keys’, ‘remote_side’, ‘order_by’。

### general

+   **[general]**

    文档已转换为 Sphinx。特别是生成的 API 文档已构建为完整的“API 参考”部分，将编辑文档与生成的文档字符串组合在一起。各个部分和 API 文档之间的交叉链接得到了极大改善，提供了一个基于 JavaScript 的搜索功能，并提供了所有类、函数和成员的完整索引。

+   **[general]**

    setup.py 现在只是可选地导入 setuptools。如果不存在，将使用 distutils。新的“pip”安装程序建议使用 easy_install，因为它以更简化的方式安装。

+   **[general]**

    在示例文件夹中添加了一个极其基本的 PostGIS 集成示例。

### orm

+   **[orm]**

    Query.with_polymorphic()现在接受第三个参数“discriminator”，该参数将替换该查询的 mapper.polymorphic_on 的值。即使 mapper 具有 polymorphic_identity，也不再需要设置 mappers 的 polymorphic_on。当未设置时，默认情况下，mapper 将以非多态方式加载。这两个功能共同允许非多态具体继承设置在每个查询基础上使用多态加载，因为在所有情况下使用多态的具体设置容易出现许多问题。

+   **[orm]**

    dynamic_loader 接受 query_class=来自定义用于动态集合和从中构建的查询的 Query 类。

+   **[orm]**

    query.order_by()接受 None，这将从查询中删除任何待定的 order_by 状态，并取消任何映射器/关系配置的排序。这主要用于覆盖在 dynamic_loader()上指定的排序。

    参考：[#1079](https://www.sqlalchemy.org/trac/ticket/1079)

+   **[orm]**

    在 compile_mappers()期间引发的异常现在被保留以提供“粘性行为” - 如果对预编译映射属性的 hasattr()调用触发失败的编译并抑制异常，则后续编译将被阻止，并且异常将在下一次 compile()调用时被重复。在使用 declarative 时，这个问题经常发生。

+   **[orm]**

    property.of_type()现在在单表继承目标上被识别，当在 prop.of_type(..).any()/has()的上下文中使用时，以及在 query.join(prop.of_type(…))中使用时。

+   **[orm]**

    query.join()在连接的目标与基于属性的属性不匹配时会引发错误 - 虽然不太可能有人这样做，但 SQLAlchemy 的作者却犯了这种特定的错误行为。

+   **[orm]**

    修复了在使用 weak_instance_map=False 时，修改事件不会被拦截以进行 flush()的 bug。

    参考：[#1272](https://www.sqlalchemy.org/trac/ticket/1272)

+   **[orm]**

    修复了一些深层次的“列对应”问题，这可能会影响针对包含同一表的多个版本的 selectable 进行的查询，以及包含不同列位置的同一表列的联合等情况。

    参考：[#1268](https://www.sqlalchemy.org/trac/ticket/1268)

+   **[orm]**

    与 column_property()，relation()等一起使用的自定义比较器类可以在 Comparator 上定义新的比较方法，这些方法将通过 InstrumentedAttribute 上的 __getattr__()可用。在 synonym()或 comparable_property()的情况下，首先在用户定义的描述符上解析属性，然后在用户定义的比较器上解析属性。

+   **[orm]**

    添加了 ScopedSession.is_active 访问器。

    参考：[#976](https://www.sqlalchemy.org/trac/ticket/976)

+   **[orm]**

    可以将映射属性和列对象作为键传递给 query.update({})。

    参考：[#1262](https://www.sqlalchemy.org/trac/ticket/1262)

+   **[orm]**

    传递给表达式级 insert()或 update()的 values()的映射属性将使用映射列的键，而不是映射属性的键。

+   **[orm]**

    修正了 Query.delete()和 Query.update()与绑定参数不正常工作的问题。

    参考：[#1242](https://www.sqlalchemy.org/trac/ticket/1242)

+   **[orm]**

    Query.select_from()、from_statement()确保给定的参数是 FromClause，或 Text/Select/Union，分别。

+   **[orm]**

    Query()可以将“复合”属性作为列表达式传递，并将其展开。与某种程度相关。

    参考：[#1253](https://www.sqlalchemy.org/trac/ticket/1253)

+   **[orm]**

    当传递各种列表达式（如字符串、clauselists、text()构造）时，Query()更加健壮（这可能意味着它只是更好地提供错误信息）。

+   **[orm]**

    first()在 Query.from_statement()中按预期工作。

+   **[orm]**

    修复了 0.5rc4 中引入的关于 eager loading 对于后期使用 add_property()或等效方法添加到 mapper 的属性不起作用的错误。

+   **[orm]**

    修复了当 viewonly=True 的 many-to-many relation()不正确引用 secondary->remote 之间的链接时的错误。

+   **[orm]**

    在向“secondary”表中插入数据时，列表型集合中的重复项将被保留在许多对多关系中。假设 m2m 表上有唯一或主键约束，这将引发预期的约束违规，而不是悄悄地删除重复条目。请注意，对于一对多关系，旧的行为仍然保留，因为在这种情况下，集合条目不会导致 INSERT 语句，SQLA 不会手动监视集合。

    参考：[#1232](https://www.sqlalchemy.org/trac/ticket/1232)

+   **[orm]**

    Query.add_column()可以接受 FromClause 对象，方式与 session.query()相同。

+   **[orm]**

    将多对一关系与 NULL 的比较正确转换为基于 not_()的 IS NOT NULL。

+   **[orm]**

    为了确保显式的 primaryjoin/secondaryjoin 是 ClauseElement 实例，以防后续出现更加混乱的错误，添加了额外的检查。

    ���考：[#1087](https://www.sqlalchemy.org/trac/ticket/1087)

+   **[orm]**

    改进了对非类类的 mapper()检查。

    参考：[#1236](https://www.sqlalchemy.org/trac/ticket/1236)

+   **[orm]**

    comparator_factory 参数现在已被所有 MapperProperty 类型文档化和支持，包括 column_property()、relation()、backref()和 synonym()。

    参考：[#5051](https://www.sqlalchemy.org/trac/ticket/5051)

+   **[orm]**

    将 PropertyLoader 的名称更改为 RelationProperty，以与所有其他名称保持一致。PropertyLoader 仍然存在作为同义词。

+   **[orm]**

    修复了在 shard API 中导致总线错误的“双重 iter()”调用，删除了 0.4 版本中遗留的错误的 result.close()。

    参考：[#1099](https://www.sqlalchemy.org/trac/ticket/1099), [#1228](https://www.sqlalchemy.org/trac/ticket/1228)

+   **[orm]**

    使 Session.merge 的级联不触发自动刷新。修复了合并的实例过早插入丢失值的问题。

+   **[orm]**

    两个修复以防止在多态联合继承方案中渲染超出范围的列（然后导致在 FROM 子句中渲染额外的表以导致笛卡尔积）：

    > +   对于 a->b->c 继承情况的“列适应”进行了改进，以更好地定位通过多级间接关系相关的列，而不是渲染非适应列。
    > +   
    > +   “多态鉴别器”列仅对实际查询的映射器进行渲染。该列不会从子类或超类映射器中“拉入”，因为它不需要。

+   **[orm]**

    修复了 ShardedSession.execute() 上的 shard_id 参数。

    参考：[#1072](https://www.sqlalchemy.org/trac/ticket/1072)

### sql

+   **[sql]**

    RowProxy 对象可以用于替代发送给 `connection.execute()` 等函数的字典参数。

    参考：[#935](https://www.sqlalchemy.org/trac/ticket/935)

+   **[sql]**

    列名中可以再次包含百分号。

    参考：[#1256](https://www.sqlalchemy.org/trac/ticket/1256)

+   **[sql]**

    sqlalchemy.sql.expression.Function 现在是一个公共类。它可以被子类化以在命令式风格中提供用户定义的 SQL 函数，包括具有预先建立的行为。 postgis.py 示例说明了其一种用法。

+   **[sql]**

    PickleType 现在默认偏向于使用 == 比较，如果传入对象（例如字典）实现了 __eq__()。如果对象没有实现 __eq__() 并且 mutable=True，则会引发弃用警告。

+   **[sql]**

    修复了 sqlalchemy.sql 中导入的怪异行为，不再导出 __names__。

    参考：[#1215](https://www.sqlalchemy.org/trac/ticket/1215)

+   **[sql]**

    多次使用相同的 ForeignKey 对象会引发错误，而不是后来默默失败。

    参考：[#1238](https://www.sqlalchemy.org/trac/ticket/1238)

+   **[sql]**

    在 Insert/Update/Delete 构造上为 params() 方法添加了 NotImplementedError。这些项目目前不支持此功能，这与 values() 相比也会有一些误导。

+   **[sql]**

    反射的外键将正确定位其引用的列，即使该列被赋予了与反射名称不同的“key”属性。这通过 ForeignKey/ForeignKeyConstraint 上的一个名为“link_to_name”的新标志实现，如果为 True，则表示给定名称是被引用列的名称，而不是其分配的键。

    参考：[#650](https://www.sqlalchemy.org/trac/ticket/650)

+   **[sql]**

    select() 可以像表或其他可选择对象一样接受 ClauseList 作为列，内部表达式将被用作列元素。

    参考：[#1253](https://www.sqlalchemy.org/trac/ticket/1253)

+   **[sql]**

    session.is_modified() 上的“被动”标志正确地传播到属性管理器。

+   **[sql]**

    union()和 union_all()不会影响已应用到 select()内部的 order_by()。 如果你 union()一个带有 order_by()的 select()（可能是为了支持 LIMIT/OFFSET），你应该在它上面调用 self_group()来应用括号。

### mysql

+   **[mysql]**

    在 text()构造中的“%”符号会自动转义为“%%”。 由于此更改的向后不兼容性，如果在字符串中检测到‘%%’，则会发出警告。

+   **[mysql]**

    修复了在反射时未存在 FK 列时引发异常的错误。

    参考：[#1241](https://www.sqlalchemy.org/trac/ticket/1241)

+   **[mysql]**

    修复了涉及反映具有对该模式中另一个表的外键引用的远程模式表的错误。

### sqlite

+   **[sqlite]**

    现在，表反射将为列存储实际的 DefaultClause 值。

    参考：[#1266](https://www.sqlalchemy.org/trac/ticket/1266)

+   **[sqlite]**

    bug 修复，行为变更

### mssql

+   **[mssql]**

    添加了一个新的 MSGenericBinary 类型。 这将其映射到 Binary 类型，以便它可以实现将长度指定类型视为固定宽度 Binary 类型和将非长度类型视为未绑定变量长度 Binary 类型的专门行为。

+   **[mssql]**

    添加了新类型：MSVarBinary 和 MSImage。

    参考：[#1249](https://www.sqlalchemy.org/trac/ticket/1249)

+   **[mssql]**

    添加了 MSReal、MSNText、MSSmallDateTime、MSTime、MSDateTimeOffset 和 MSDateTime2 类型

+   **[mssql]**

    重构了日期/时间类型。 `smalldatetime`数据类型不再截断为仅日期，现在将映射到 MSSmallDateTime 类型。

    参考：[#1254](https://www.sqlalchemy.org/trac/ticket/1254)

+   **[mssql]**

    修正了一个关于 Numerics 接受 int 的问题。

+   **[mssql]**

    将`char_length`映射到`LEN()`函数。

+   **[mssql]**

    如果`INSERT`包含子查询，则`INSERT`会从`INSERT INTO VALUES`构造转换为`INSERT INTO SELECT`构造。

+   **[mssql]**

    如果列是`primary_key`的一部分，则它将是`NOT NULL`，因为 MSSQL 不允许在`primary_key`列中使用`NULL`。

+   **[mssql]**

    `MSBinary`现在返回`BINARY`而不是`IMAGE`。 这是一个向后不兼容的更改，因为`BINARY`是一个固定长度的数据类型，而`IMAGE`是一个可变长度的数据类型。

    参考：[#1249](https://www.sqlalchemy.org/trac/ticket/1249)

+   **[mssql]**

    `get_default_schema_name`现在从数据库中反映出来，基于用户的默认模式。 这仅适用于 MSSQL 2005 及更高版本。

    参考：[#1258](https://www.sqlalchemy.org/trac/ticket/1258)

+   **[mssql]**

    通过使用新的 collation 参数，添加了对 collation 的支持。 这在以下类型上受支持：char、nchar、varchar、nvarchar、text、ntext。

    参考：[#1248](https://www.sqlalchemy.org/trac/ticket/1248)

+   **[mssql]**

    连接字符串参数的更改更倾向于将 DSN 作为 pyodbc 的默认规范。 有关详细的使用说明，请参阅 mssql.py docstring。

+   **[mssql]**

    添加了对保存点的实验性支持。目前与会话不完全兼容。

+   **[mssql]**

    支持三种列的空值性质：NULL、NOT NULL 和数据库配置的默认值。默认列配置（nullable=True）现在将在 DDL 中生成 NULL。以前没有发出任何规范，并且将采取数据库默认值（通常是 NULL，但并非总是）。为了显式请求数据库默认值，请使用 nullable=None 配置列，DDL 中将不会发出任何规范。这是向后不兼容的行为。

    参考：[#1243](https://www.sqlalchemy.org/trac/ticket/1243)

### oracle

+   **[oracle]**

    调整了 create_xid() 的格式以修复两阶段提交。我们现在收到了 Oracle 两阶段提交正常工作的实地报告。

+   **[oracle]**

    添加了 OracleNVarchar 类型，生成 NVARCHAR2，并且默认情况下也是 Unicode 子类，因此 convert_unicode=True。NVARCHAR2 自动反映到此类型，因此这些列在没有显式 convert_unicode=True 标志的情况下在反映表上传递 Unicode。

    参考：[#1233](https://www.sqlalchemy.org/trac/ticket/1233)

+   **[oracle]**

    修复了阻止接收某些类型的 out 参数的错误；非常感谢 wwu.edu 上的 huddlej！

    参考：[#1265](https://www.sqlalchemy.org/trac/ticket/1265)

### misc

+   **[dialect]**

    在方言上添加了一个新的 description_encoding 属性，用于在处理元数据时对列名进行编码。这通常默认为 utf-8。

+   **[engine/pool]**

    Connection.invalidate() 检查关闭状态以避免属性错误。

    参考：[#1246](https://www.sqlalchemy.org/trac/ticket/1246)

+   **[engine/pool]**

    NullPool 支持失败时重新连接的行为。

    参考：[#1094](https://www.sqlalchemy.org/trac/ticket/1094)

+   **[engine/pool]**

    在使用 pool.manage(dbapi) 时，为初始池创建添加了互斥锁。这可以防止在启动时出现一种轻微的“堆积”行为。

    参考：[#799](https://www.sqlalchemy.org/trac/ticket/799)

+   **[engine/pool]**

    _execute_clauseelement() 方法重新变为私有方法。现在可以使用 ConnectionProxy，无需子类化 Connection。

+   **[documentation]**

    Tickets。

    参考：[#1149](https://www.sqlalchemy.org/trac/ticket/1149)，[#1200](https://www.sqlalchemy.org/trac/ticket/1200)

+   **[documentation]**

    添加了关于 create_session() 默认值的说明。

+   **[documentation]**

    添加了关于 metadata.reflect() 的部分。

+   **[documentation]**

    更新了 TypeDecorator 部分。

+   **[documentation]**

    由于最近对此功能的混淆，重写了文档中“threadlocal”策略部分。

+   **[documentation]**

    从继承中删除了过时的“polymorphic_fetch”和“select_table”文档，重新设计了“联接表继承”的后半部分。

+   **[documentation]**

    添加了 comparator_factory 参数的文档说明，新增了名为“自定义比较器”的新文档部分。

+   **[postgres]**

    在 text() 构造中的 “%” 符号自动转义为 “%%”。由于此更改的向后不兼容性，如果在字符串中检测到 ‘%%’，则会发出警告。

    参考: [#1267](https://www.sqlalchemy.org/trac/ticket/1267)

+   **[Postgres]**

    与 server_side_cursors 结合使用时，调用 alias.execute() 不会引发 AttributeError。

+   **[Postgres]**

    添加了对 PostgreSQL 的索引反射支持，使用了长期被忽视的出色补丁，由 Ken Kuhlman 提交。

    参考: [#714](https://www.sqlalchemy.org/trac/ticket/714)

+   **[关联代理]**

    关联代理属性现在在类级别上可用，例如 MyClass.aproxy。之前这会评估为 None。

+   **[声明式]**

    字符串形式的 backref() 接受的参数包括 ‘primaryjoin’, ‘secondaryjoin’, ‘secondary’, ‘foreign_keys’, ‘remote_side’, ‘order_by’。

## 0.5.0rc4

发布日期: 2008 年 11 月 14 日

### 一般

+   **[一般]**

    全局 “propigate”->”propagate” 更改。

### ORM

+   **[ORM]**

    Query.count() 已经增强，以在更广泛的情况下执行“正确的操作”。它现在可以计算多实体查询，以及基于列的查询。请注意，这意味着如果你说 query(A, B).count() 而没有任何连接条件，它将计算 A*B 的笛卡尔积。对于针对基于列的实体的任何查询，将自动发出 “SELECT count(1) FROM (SELECT…)”，以便返回真实的行数，这意味着像 query(func.count(A.name)).count() 这样的查询将返回一个值，因为该查询将返回一行。

+   **[ORM]**

    大量性能调优。对各种 ORM 操作的粗略估计将其速度提高了 10%，比 0.5.0rc3 快 25-30%。

+   **[ORM]**

    错误修复和行为变更

+   **[ORM]**

    调整了对 InstanceState 的增强垃圾回收，以更好地防止由于丢失状态而导致的错误。

+   **[ORM]**

    当针对多个实体执行时，Query.get() 返回一个更具信息性的错误消息。

    参考: [#1220](https://www.sqlalchemy.org/trac/ticket/1220)

+   **[ORM]**

    恢复了 Cls.relation.in_() 上的 NotImplementedError。

    参考: [#1140](https://www.sqlalchemy.org/trac/ticket/1140), [#1221](https://www.sqlalchemy.org/trac/ticket/1221)

+   **[ORM]**

    修复了涉及 relation() 上 order_by 参数的 PendingDeprecationWarning。

    参考: [#1226](https://www.sqlalchemy.org/trac/ticket/1226)

### SQL

+   **[SQL]**

    删除了 Connection 对象的 ‘properties’ 属性，应该使用 Connection.info。

+   **[SQL]**

    在 ResultProxy 自动关闭游标之前恢复了 “活动行数” 获取。这在 0.5rc3 中被移除。

+   **[SQL]**

    重新排列了 TypeDecorator 中的 load_dialect_impl() 方法，以便即使用户定义的 TypeDecorator 使用另一个 TypeDecorator 作为其 impl，也会生效。

### MSSQL

+   **[MSSQL]**

    大量清理和修复，以纠正限制和偏移的问题。

+   **[MSSQL]**

    修正了作为二进制表达式一部分的子查询需要被转换为使用 IN 和 NOT IN 语法的情况。

+   **[MSSQL]**

    修复了 E Notation 问题，导致无法插入小于 1E-6 的十进制值。

    参考：[#1216](https://www.sqlalchemy.org/trac/ticket/1216)

+   **[mssql]**

    修正了处理模式时反射的问题，特别是当这些模式是默认模式时。

    参考：[#1217](https://www.sqlalchemy.org/trac/ticket/1217)

+   **[mssql]**

    修正了将零长度项转换为 varchar 时出现的问题。现在它会正确调整 CAST。

### 杂项

+   **[access]**

    增加了对 Currency 类型的支持。

+   **[access]**

    函数未返回它们的结果。

    参考：[#1017](https://www.sqlalchemy.org/trac/ticket/1017)

+   **[access]**

    修正了关联的问题。Access 仅支持 LEFT OUTER 或 INNER，而不仅仅是 JOIN 本身。

    参考：[#1017](https://www.sqlalchemy.org/trac/ticket/1017)

+   **[ext]**

    在使用 declarative 时，现在可以在 __mapper_args__ 中使用自定义的“inherit_condition”。

+   **[ext]**

    修复了在 backref()中使用基于字符串的“remote_side”、“order_by”等时未正确传播的问题。

### 一般

+   **[general]**

    全局“propigate”->”propagate”更改。

### orm

+   **[orm]**

    Query.count()已经增强，以在更广泛的情况下执行“正确的操作”。它现在可以计算多实体查询，以及基于列的查询。请注意，这意味着如果你说 query(A, B).count()没有任何连接条件，它将计算 A*B 的笛卡尔积。对于针对基于列的实体的查询，将自动发出“SELECT count(1) FROM (SELECT…)”以返回真实的行数，这意味着像 query(func.count(A.name)).count()这样的查询将返回一个值，因为该查询将返回一行。

+   **[orm]**

    进行了大量性能调优。对各种 ORM 操作的粗略估计表明，它比 0.5.0rc3 快 10%，比 0.4.8 快 25-30%。

+   **[orm]**

    修复了错误和行为变化

+   **[orm]**

    调整了对 InstanceState 的增强垃圾回收，以更好地防止由于丢失状态而导致的错误。

+   **[orm]**

    Query.get()在针对多个实体执行时返回更具信息性的错误消息。

    参考：[#1220](https://www.sqlalchemy.org/trac/ticket/1220)

+   **[orm]**

    恢复了 Cls.relation.in_ 上的 NotImplementedError

    参考：[#1140](https://www.sqlalchemy.org/trac/ticket/1140), [#1221](https://www.sqlalchemy.org/trac/ticket/1221)

+   **[orm]**

    修复了涉及 relation()上 order_by 参数的 PendingDeprecationWarning。

    参考：[#1226](https://www.sqlalchemy.org/trac/ticket/1226)

### sql

+   **[sql]**

    移除了 Connection 对象的‘properties’属性，应该使用 Connection.info。

+   **[sql]**

    恢复了在 ResultProxy 自动关闭游标之前获取“活动行数”的获取。这在 0.5rc3 中被移除。

+   **[sql]**

    重新排列了 TypeDecorator 中的 load_dialect_impl()方法，以便即使用户定义的 TypeDecorator 使用另一个 TypeDecorator 作为其 impl，它也会生效。

### mssql

+   **[mssql]**

    进行了大量清理和修复，以纠正限制和偏移的问题。

+   **[mssql]**

    修正了作为二元表达式一部分的子查询需要转换为使用 IN 和 NOT IN 语法的情况。

+   **[mssql]**

    修复了 E 表示法问题，导致无法插入小于 1E-6 的十进制值。

    参考：[#1216](https://www.sqlalchemy.org/trac/ticket/1216)

+   **[mssql]**

    修正了在处理模式时反射的问题，特别是当这些模式是默认模式时。

    参考：[#1217](https://www.sqlalchemy.org/trac/ticket/1217)

+   **[mssql]**

    修正了将零长度项目转换为 varchar 的问题。现在它会正确调整 CAST。

### 杂项

+   **[access]**

    增加了对 Currency 类型的支持。

+   **[access]**

    函数没有返回它们的结果。

    参考：[#1017](https://www.sqlalchemy.org/trac/ticket/1017)

+   **[access]**

    修正了连接问题。Access 只支持 LEFT OUTER 或 INNER，而不是仅仅 JOIN。

    参考：[#1017](https://www.sqlalchemy.org/trac/ticket/1017)

+   **[ext]**

    在使用 declarative 时，现在可以在 __mapper_args__ 中使用自定义的 “inherit_condition”。

+   **[ext]**

    修复了在 backref() 中使用基于字符串的 “remote_side”、“order_by” 等时未正确传播的问题。

## 0.5.0rc3

发布日期：2008 年 11 月 07 日

### orm

+   **[orm]**

    在 SessionExtension 中添加了两个新的钩子：after_bulk_delete() 和 after_bulk_update()。在查询上执行 bulk delete() 操作后会调用 after_bulk_delete()。在查询上执行 bulk update() 操作后会调用 after_bulk_update()。

+   **[orm]**

    简单的一对多关系的“不等于”比较将不会陷入 EXISTS 子句，而是会比较外键列。

+   **[orm]**

    移除了将集合与可迭代对象进行比较的不太有效的用例。使用 contains() 来测试集合成员资格。

+   **[orm]**

    改进了 aliased() 对象的行为，使其更准确地适应生成的表达式，这在自引用比较中特别有帮助。

    参考：[#1171](https://www.sqlalchemy.org/trac/ticket/1171)

+   **[orm]**

    修复了从类绑定属性构造的 primaryjoin/secondaryjoin 条件的 bug（通常在使用 declarative 时发生），稍后 Query 会不正确地别名，特别是在使用各种基于 EXISTS 的比较器时。

+   **[orm]**

    修复了使用多个 query.join() 与别名绑定描述符时会丢失左别名的 bug。

+   **[orm]**

    改进了弱引用标识映射的内存管理，不再需要互斥锁，对具有挂起更改的 InstanceState 在懒惰基础上复活被垃圾回收的实例。

+   **[orm]**

    InstanceState 对象现在在处理后会删除对自身的循环引用，以使其不受循环垃圾回收的影响。

+   **[orm]**

    在确定连接条件时，relation() 不会在“请指定 primaryjoin”消息中隐藏无关的 ForeignKey 错误。

+   **[orm]**

    修复了 Query 中涉及 order_by() 与同一类的多个别名一起使用时的 bug（将添加测试）。

    参考：[#1218](https://www.sqlalchemy.org/trac/ticket/1218)

+   **[orm]**

    在使用 Query.join()时，如果 ON 子句有明确的条件，该条件将被别名化为连接的左侧，允许像 query(Source). from_self().join((Dest, Source.id==Dest.source_id))这样的场景正常工作。

+   **[orm]**

    如果每个列的“key”与列的名称不同，polymorphic_union()函数将尊重每个列的“key”。

+   **[orm]**

    修复了“被动删除”在具有“delete”级联的多对一关系()上的支持。

    参考：[#1183](https://www.sqlalchemy.org/trac/ticket/1183)

+   **[orm]**

    修复了组合类型中的错误，该错误阻止了主键组合类型的变异。

    参考：[#1213](https://www.sqlalchemy.org/trac/ticket/1213)

+   **[orm]**

    增加了对内部属性访问的更细粒度，使得级联和刷新操作不会初始化未加载的属性和集合，使它们保持原样以便稍后进行延迟加载。对于待处理实例，反向引用事件仍会初始化属性和集合。

    参考：[#1202](https://www.sqlalchemy.org/trac/ticket/1202)

### sql

+   **[sql]**

    SQL 编译器优化和复杂性降低。编译典型 select()构造的调用次数比 0.5.0rc2 少 20%。 

+   **[sql]**

    方言现在可以生成可调整长度的标签名称。传入参数“label_length=<value>”到 create_engine()以调整动态生成列标签中最多有多少个字符，即“somecolumn AS somelabel”。任何小于 6 的值将导致最小尺寸的标签，由下划线和数字计数器组成。编译器使用 dialect.max_identifier_length 的值作为默认值。

    参考：[#1211](https://www.sqlalchemy.org/trac/ticket/1211)

+   **[sql]**

    简化了对 ResultProxy“无结果自动关闭”的检查，仅基于光标描述的存在。所有基于正则表达式的关于返回行的猜测都已被移除。

    参考：[#1212](https://www.sqlalchemy.org/trac/ticket/1212)

+   **[sql]**

    直接执行 union()构造将正确设置结果行处理。

    参考：[#1194](https://www.sqlalchemy.org/trac/ticket/1194)

+   **[sql]**

    已移除“OID”或“ROWID”列的内部概念。它基本上没有被任何方言使用，而且现在使用 INSERT..RETURNING，基本上不再可能与 psycopg2 的 cursor.lastrowid 一起使用。

+   **[sql]**

    在所有 FromClause 对象上移除了“default_order_by()”方法。

+   **[sql]**

    修复了 table.tometadata()方法，使传入的模式参数传播到 ForeignKey 构造中。

+   **[sql]**

    稍微改变了用于与空集合比较的 IN 运算符的行为。现在会导致与自身的不等比较。更具可移植性，但会破坏不是纯函数的存储过程。

### mysql

+   **[mysql]**

    在表的显式 schema= 与连接附加到的 schema（数据库）相同时，修复了外键反射的边缘情况。

+   **[MySQL]**

    不再期望表反射中的 include_columns 为小写。

### Oracle

+   **[Oracle]**

    为 Oracle dialect 编写了一个文档字符串。显然，Ohloh 的“few source code comments”标签开始串联 :).

+   **[Oracle]**

    在使用 LIMIT/OFFSET 时移除了 FIRST_ROWS() 优化标志，可以通过 optimize_limits=True create_engine() 标志重新启用。

    参考：[#536](https://www.sqlalchemy.org/trac/ticket/536)

+   **[Oracle]**

    错误修复和行为更改

+   **[Oracle]**

    在 create_engine() 上将 auto_convert_lobs 设置为 False 也将指示 OracleBinary 类型返回 cx_oracle LOB 对象不变。

### 杂项

+   **[扩展]**

    添加了一个新的扩展 sqlalchemy.ext.serializer。提供了与 Pickle/Unpickle 镜像的 Serializer/Deserializer “类”，以及 dumps() 和 loads()。此序列化程序实现了一个“外部对象” pickler，将关键上下文敏感对象，包括 engines、sessions、metadata、Tables/Columns 和 mappers，保持在 pickle 流之外，并且可以稍后使用任何 engine/metadata/session 提供程序恢复 pickle。这不是用于 pickling 普通对象实例，这些对象实例可以在没有任何特殊逻辑的情况下进行 pickling，而是用于 pickling 表达式对象和完整的 Query 对象，以便在 unpickle 时可以恢复所有 mapper/engine/session 依赖项。

+   **[扩展]**

    修复了阻止 declarative-bound “column” 对象在 column_mapped_collection() 中使用的 bug。

    参考：[#1174](https://www.sqlalchemy.org/trac/ticket/1174)

+   **[杂项]**

    util.flatten_iterator() 函数不会将具有 __iter__() 方法的字符串解释为迭代器，例如在 pypy 中。

    参考：[#1077](https://www.sqlalchemy.org/trac/ticket/1077)

### ORM

+   **[ORM]**

    向 SessionExtension 添加了两个新的 hooks：after_bulk_delete() 和 after_bulk_update()。在查询上执行 bulk delete() 操作后调用 after_bulk_delete()。在查询上执行 bulk update() 操作后调用 after_bulk_update()。

+   **[ORM]**

    简单的多对一关系的“不等于”比较将不会陷入 EXISTS 子句，并将比较外键列。

+   **[ORM]**

    移除了比较集合与可迭代对象的不真正有效的用例。使用 contains() 来测试集合成员资格。

+   **[ORM]**

    改进了 aliased() 对象的行为，使其更准确地适应生成的表达式，特别有助于自引用比较。

    参考：[#1171](https://www.sqlalchemy.org/trac/ticket/1171)

+   **[ORM]**

    修复了从类绑定属性构造的 primaryjoin/secondaryjoin 条件的 bug（在使用 declarative 时经常发生），稍后 Query 会不适当地别名化这些条件，��别是在各种基于 EXISTS 的比较器中。

+   **[ORM]**

    修复了使用多个 query.join() 与别名绑定描述符时会丢失左别名的 bug。

+   **[orm]**

    改进了弱引用标识映射的内存管理，不再需要互斥锁，对具有挂起更改的 InstanceState 在懒惰基础上复活被垃圾回收的实例。

+   **[orm]**

    InstanceState 对象现在在处理后会删除对自身的循环引用，以使其保持在循环垃圾回收之外。

+   **[orm]**

    在确定连接条件时，relation() 不会在“请指定 primaryjoin”消息中隐藏不相关的 ForeignKey 错误。

+   **[orm]**

    修复了 Query 中涉及 order_by() 与同一类的多个别名结合时的 bug（将添加测试）

    参考：[#1218](https://www.sqlalchemy.org/trac/ticket/1218)

+   **[orm]**

    使用 Query.join() 时，对 ON 子句进行显式定义，该子句将别名化为连接的左侧，允许像 query(Source). from_self().join((Dest, Source.id==Dest.source_id)) 这样的场景正常工作。

+   **[orm]**

    如果 polymorphic_union() 函数中的每个 Column 的“key”与列名不同，则会予以尊重。

+   **[orm]**

    修复了在“delete”级联关系上支持“被动删除”对多对一 relation() 的支持。

    参考：[#1183](https://www.sqlalchemy.org/trac/ticket/1183)

+   **[orm]**

    修复了组合类型中的 bug，该 bug 阻止了主键组合类型的变异。

    参考：[#1213](https://www.sqlalchemy.org/trac/ticket/1213)

+   **[orm]**

    对内部属性访问增加了更多细粒度，使得级联和刷新操作不会初始化未加载的属性和集合，保持它们完整以供稍后进行延迟加载。对于待处理实例，反向引用事件仍会初始化属性和集合。

    参考：[#1202](https://www.sqlalchemy.org/trac/ticket/1202)

### sql

+   **[sql]**

    SQL 编译器优化和复杂性降低。编译典型 select() 构造的调用次数比 0.5.0rc2 少 20%。

+   **[sql]**

    方言现在可以生成可调整长度的标签名称。传入参数“label_length=<value>”到 create_engine() 中，以调整动态生成列标签中最多存在多少个字符，即“somecolumn AS somelabel”。任何小于 6 的值将导致最小尺寸的标签，由下划线和数字计数器组成。编译器使用 dialect.max_identifier_length 的值作为默认值。

    参考：[#1211](https://www.sqlalchemy.org/trac/ticket/1211)

+   **[sql]**

    简化了对 ResultProxy “无结果自动关闭” 的检查，仅基于存在 cursor.description。所有基于正则表达式的关于语句返回行的猜测都已被移除。

    参考：[#1212](https://www.sqlalchemy.org/trac/ticket/1212)

+   **[sql]**

    直接执行 union() 构造将正确设置结果行处理。

    参考：[#1194](https://www.sqlalchemy.org/trac/ticket/1194)

+   **[sql]**

    已删除“OID”或“ROWID”列的内部概念。基本上没有任何方言使用它，而且现在使用 INSERT..RETURNING，使用 psycopg2 的 cursor.lastrowid 的可能性基本上已经消失。

+   **[sql]**

    在所有 FromClause 对象上删除了“default_order_by()”方法。

+   **[sql]**

    修复了 table.tometadata()方法，使传入的模式参数传播到 ForeignKey 构造中。

+   **[sql]**

    稍微改变了用于与空集合比较的 IN 运算符的行为。现在会导致与自身的不等式比较。更具可移植性，但会破坏不是纯函数的存储过程。

### mysql

+   **[mysql]**

    修复了在表的显式 schema=与连接附加到的 schema（数据库）相同时的外键反射问题。

+   **[mysql]**

    不再期望在表反射中包含的列为小写。

### oracle

+   **[oracle]**

    为 Oracle 方言编写了一个文档字符串。显然，Ohloh 的“few source code comments”标签开始串联了 :).

+   **[oracle]**

    在使用 LIMIT/OFFSET 时删除了 FIRST_ROWS()优化标志，可以通过 optimize_limits=True create_engine()标志重新启用。

    参考：[#536](https://www.sqlalchemy.org/trac/ticket/536)

+   **[oracle]**

    修复错误和行为更改

+   **[oracle]**

    在 create_engine()上将 auto_convert_lobs 设置为 False 也将指示 OracleBinary 类型将 cx_oracle LOB 对象原样返回。

### misc

+   **[ext]**

    添加了一个新的扩展 sqlalchemy.ext.serializer。提供了 Serializer/Deserializer“类”，这些类镜像了 Pickle/Unpickle，以及 dumps()和 loads()。这个序列化器实现了一个“外部对象”pickler，它将关键上下文敏感对象，包括 engines、sessions、metadata、Tables/Columns 和 mappers，保持在 pickle 流之外，并且可以在以后使用任何 engine/metadata/session 提供程序恢复 pickle。这不是用于 pickle 常规对象��例，这些对象实例可以在没有任何特殊逻辑的情况下进行 pickle，而是用于 pickle 表达式对象和完整的 Query 对象，以便在 unpickle 时可以恢复所有 mapper/engine/session 依赖关系。

+   **[ext]**

    修复了阻止声明绑定的“column”对象在 column_mapped_collection()中使用的错误。

    参考：[#1174](https://www.sqlalchemy.org/trac/ticket/1174)

+   **[misc]**

    util.flatten_iterator()函数不会将具有 __iter__()方法的字符串解释为迭代器，例如在 pypy 中。

    参考：[#1077](https://www.sqlalchemy.org/trac/ticket/1077)

## 0.5.0rc2

发布日期：2008 年 10 月 12 日星期日

### orm

+   **[orm]**

    修复了涉及包含文字或其他非列表达式的 read/write relation()在其 primaryjoin 条件中等于外键列的错误。

+   **[orm]**

    在 mapper()中的“non-batch”模式，这个功能允许在每个实例更新/插入时调用 mapper 扩展方法，现在遵守给定对象的插入顺序。

+   **[orm]**

    修复了 mapper 中与 RLock 相关的 bug，该 bug 可能在重新进入的 mapper compile()调用时发生死锁，这种情况发生在在 ForeignKey 对象中使用声明性构造时。

+   **[orm]**

    ScopedSession.query_property 现在接受一个 query_cls 工厂，覆盖了会话配置的 query_cls。

+   **[orm]**

    修复了共享状态 bug，干扰了 ScopedSession.mapper 在对象子类上应用默认 __init__ 实现的能力。

+   **[orm]**

    修复了 Query 上的切片（即 query[x:y]）对于零长度切片、两端为 None 的切片的正确工作。

    参考：[#1177](https://www.sqlalchemy.org/trac/ticket/1177)

+   **[orm]**

    添加了一个示例，说明了 Celko 的“嵌套集合”作为 SQLA 映射的方法。

+   **[orm]**

    包含带有别名参数的 contains_eager()即使别名嵌入在 SELECT 中，也可以在通过 query.select_from()发送到查询时正常工作。

+   **[orm]**

    contains_eager()的用法现在与同时包含常规急加载和限制/偏移的查询兼容，因为列被添加到由查询生成的子查询中。

    参考：[#1180](https://www.sqlalchemy.org/trac/ticket/1180)

+   **[orm]**

    session.execute()将执行传递给它的 Sequence 对象（从 0.4 版本中的回归）。

+   **[orm]**

    从 object_mapper()和 class_mapper()中删除了“raiseerror”关键字参数。如果给定的类/实例未映射，这些函数在所有情况下都会引发异常。

+   **[orm]**

    修复了在 autocommit=False 会话上调用 session.transaction.commit()不启动新事务的问题。

+   **[orm]**

    对 Session.identity_map 的弱引用行为进行了一些调整，以减少异步 GC 的副作用。

+   **[orm]**

    调整了 Session 在后刷新时对新“干净”对象的计数，以更好地防止在异步 gc 时对对象进行操作。

    参考：[#1182](https://www.sqlalchemy.org/trac/ticket/1182)

### sql

+   **[sql]**

    column.in_(someselect)现在可以作为一个列子句表达式使用，而不会导致子查询泄漏到 FROM 子句中

    参考：[#1074](https://www.sqlalchemy.org/trac/ticket/1074)

### mysql

+   **[mysql]**

    临时表现在可以反射。

### sqlite

+   **[sqlite]**

    对 SQLite 日期/时间绑定/结果处理进行了全面改进，使用正则表达式和格式字符串，而不是 strptime/strftime，以通用支持 1900 年前的日期、带微秒的日期。

    参考：[#968](https://www.sqlalchemy.org/trac/ticket/968)

+   **[sqlite]**

    在 sqlite 方言中禁用了 String（以及 Unicode，UnicodeText 等）的 convert_unicode 逻辑，以适应 pysqlite 2.5.0 的新要求，即只接受 Python unicode 对象；[`itsystementwicklung.de/pipermail/list-pysqlite/2008-March/000018.html`](https://itsystementwicklung.de/pipermail/list-pysqlite/2008-March/000018.html)

### oracle

+   **[oracle]**

    Oracle 将检测包含在 SELECT 之前的字符串语句中的注释作为 SELECT 语句。

    参考：[#1187](https://www.sqlalchemy.org/trac/ticket/1187)

### orm

+   **[orm]**

    修复了涉及包含文字或其他非列表达式的 read/write relation() 在其 primaryjoin 条件中等于外键列的 bug。

+   **[orm]**

    ”非批量”模式在 mapper() 中，这个特性允许在每个实例更新/插入时调用映射扩展方法，现在遵循给定对象的插入顺序。

+   **[orm]**

    修复了 mapper 中与 RLock 相关的 bug，该 bug 可能在重新进入 mapper compile() 调用时发生死锁，这种情况发生在 ForeignKey 对象中使用声明性构造时。

+   **[orm]**

    ScopedSession.query_property 现在接受一个 query_cls 工厂，覆盖了会话配置的 query_cls。

+   **[orm]**

    修复了共享状态 bug，干扰了 ScopedSession.mapper 对对象子类应用默认 __init__ 实现的能力。

+   **[orm]**

    修复了 Query 上的切片（即 query[x:y]）对于零长度切片、任一端为 None 的切片的正确工作。

    参考：[#1177](https://www.sqlalchemy.org/trac/ticket/1177)

+   **[orm]**

    添加了一个示例，说明了 Celko 的“嵌套集”作为 SQLA 映射。

+   **[orm]**

    包含了一个别名参数的 contains_eager() 在 SELECT 中嵌入别名时也能正常工作，就像通过 query.select_from() 发送给 Query 一样。

+   **[orm]**

    contains_eager() 的用法现在与同时包含常规 eager load 和 limit/offset 的 Query 兼容，因此列被添加到 Query 生成的子查询中。

    参考：[#1180](https://www.sqlalchemy.org/trac/ticket/1180)

+   **[orm]**

    session.execute() 现在可以执行传递给它的 Sequence 对象（从 0.4 版本中的回归）。

+   **[orm]**

    从 object_mapper() 和 class_mapper() 中移除了“raiseerror”关键字参数。如果给定的类/实例未映射，这些函数在所有情况下都会引发异常。

+   **[orm]**

    修复了在 autocommit=False 会话上调用 session.transaction.commit() 未启动新事务的 bug。

+   **[orm]**

    对 Session.identity_map 的弱引用行为进行了一些调整，以减少异步 GC 的副作用。

+   **[orm]**

    调整了 Session 在后刷新时对新“干净”对象的计数，以更好地防止在对象异步 gc 时对其进行操作。

    参考：[#1182](https://www.sqlalchemy.org/trac/ticket/1182)

### sql

+   **[sql]**

    column.in_(someselect) 现在可以作为一个列子句表达式使用，而不会导致子查询泄漏到 FROM 子句中。

    参考：[#1074](https://www.sqlalchemy.org/trac/ticket/1074)

### mysql

+   **[mysql]**

    临时表现在可以反射。

### sqlite

+   **[sqlite]**

    对 SQLite 日期/时间绑定/结果处理进行了全面改进，使用正则表达式和格式字符串，而不是 strptime/strftime，以通用支持 1900 年前的日期、带微秒的日期。

    参考：[#968](https://www.sqlalchemy.org/trac/ticket/968)

+   **[sqlite]**

    在 sqlite 方言中禁用了 String（和 Unicode，UnicodeText 等）的 convert_unicode 逻辑，以适应 pysqlite 2.5.0 的新要求，即只接受 Python unicode 对象；[`itsystementwicklung.de/pipermail/list-pysqlite/2008-March/000018.html`](https://itsystementwicklung.de/pipermail/list-pysqlite/2008-March/000018.html)

### oracle

+   **[oracle]**

    Oracle 将检测包含在 SELECT 之前的前面的注释的基于字符串的语句作为 SELECT 语句。

    参考：[#1187](https://www.sqlalchemy.org/trac/ticket/1187)

## 0.5.0rc1

发布日期：Thu Sep 11 2008

### orm

+   **[orm]**

    Query 现在具有 delete()和 update(values)方法。这允许使用 Query 对象执行批量删除/更新。

+   **[orm]**

    Query(*cols)返回的 RowTuple 对象现在具有 keynames，它们更喜欢映射属性名称而不是列键，列键而不是列名称，即 Query(Class.foo, Class.bar)将具有名称“foo”和“bar”，即使它们不是基础 Column 对象的名称。直接的 Column 对象，如 Query(table.c.col)，将���回 Column 的“key”属性。

+   **[orm]**

    向 Query 添加了 scalar()和 value()方法，每个方法返回单个标量值。scalar()不带参数，大致相当于 first()[0]，value()接受单个列表达式，大致相当于 values(expr).next()[0]。

+   **[orm]**

    改进了在将 SQL 表达式放置在查询()实体列表中时确定 FROM 子句的方法。特别是标量子查询不应该将其内部 FROM 对象泄漏到封闭查询中。

+   **[orm]**

    通过从映射类到映射子类的关系()进行连接，其中映射子类配置为单表继承，将在连接的 ON 子句中包含一个 IN 子句，该子句将限制所请求的连接类的子类型。这对急加载连接以及 query.join()都有效。请注意，在某些情况下，由于此区分具有多个触发点，IN 子句也将出现在查询的 WHERE 子句中。

+   **[orm]**

    AttributeExtension 已经得到改进，使得事件在实际发生变化之前被触发。此外，append()和 set()方法现在必须返回给定的值，该值用作要在变异操作中使用的值。这允许创建验证 AttributeListeners，它们在实际操作发生之前引发，并且可以将给定的值更改为其他值。

+   **[orm]**

    column_property()，composite_property()和 relation()现在接受一个单个或 AttributeExtensions 列表，使用“extension”关键字参数。

+   **[orm]**

    query.order_by().get()会在 GET 发出的查询中静默删除“ORDER BY”，但不会引发异常。

+   **[orm]**

    添加了一个验证器 AttributeExtension，以及一个@validates 装饰器，类似于@reconstructor，标记一个方法验证一个或多个映射属性。

+   **[orm]**

    class.someprop.in_()在关系的“in_”实现之前引发 NotImplementedError

    参考：[#1140](https://www.sqlalchemy.org/trac/ticket/1140)

+   **[orm]**

    修复了许多对多集合的主键更新，其中集合尚未加载的 bug

    参考：[#1127](https://www.sqlalchemy.org/trac/ticket/1127)

+   **[orm]**

    修复了具有组合在一起的 deferred()列与与其无关的 synonym()一起产生 AttributeError 的 bug。

+   **[orm]**

    在 SessionExtension 的 before_flush()钩子在最终计算新/脏/已删除列表之前发生，允许在 flush 继续之前在 before_flush()中进一步更改 Session 的状态。

    参考：[#1128](https://www.sqlalchemy.org/trac/ticket/1128)

+   **[orm]**

    Session 和其他对象的“extension”参数现在可以选择性地是一个列表，支持发送到多个 SessionExtension 实例的事件。 Session 将 SessionExtensions 放在 Session.extensions 中。

+   **[orm]**

    对 flush()的可重入调用会引发错误。 这也作为针对 Session.flush()的并发调用的基本但不是绝对可靠的检查。

+   **[orm]**

    改进了在使用显式连接条件（即不是在关系上）连接到继承子类时，query.join()的行为。

+   **[orm]**

    @orm.attributes.reconstitute 和 MapperExtension.reconstitute 已重命名为@orm.reconstructor 和 MapperExtension.reconstruct_instance

+   **[orm]**

    修复了从基类继承的子类的@reconstructor 钩子。

    参考：[#1129](https://www.sqlalchemy.org/trac/ticket/1129)

+   **[orm]**

    composite()属性类型现在支持在复合类上的 __set_composite_values__()方法，如果该类使用属性名称而不是列的键名来表示状态，则需要该方法；默认生成的值现在在 flush 时正确填充。此外，将属性设置为 None 的复合类现在可以正确比较。

    参考：[#1132](https://www.sqlalchemy.org/trac/ticket/1132)

+   **[orm]**

    由 attributes.get_history()返回的三元组现在可以是列表和元组的混合。 （以前成员总是列表。）

+   **[orm]**

    修复了在更改实体上的主键属性时，如果属性的先前值已过期，则在 flush()时会产生错误的错误。

    参考：[#1151](https://www.sqlalchemy.org/trac/ticket/1151)

+   **[orm]**

    修复了自定义仪器化 bug，其中对于 ORM 尚未加载的新构造实例未调用 get_instance_dict()。

+   **[orm]**

    如果给定对象尚未存在，则 Session.delete()将该对象添加到会话中。这是从 0.4 版本开始的一个回归错误。

    参考：[#1150](https://www.sqlalchemy.org/trac/ticket/1150)

+   **[orm]**

    Session 上的 echo_uow 标志已弃用，现在单元操作日志记录仅适用于应用程序级别，而不是每个会话级别。

+   **[orm]**

    从 InstrumentedAttribute 中删除了冲突的 contains()运算符，该运算符不接受 escape kwaarg。

    参考：[#1153](https://www.sqlalchemy.org/trac/ticket/1153)

### sql

+   **[sql]**

    暂时回滚了“ORDER BY”增强功能。此功能暂停等待进一步开发。

    参考：[#1068](https://www.sqlalchemy.org/trac/ticket/1068)

+   **[sql]**

    exists()构造不会将其包含的元素列表作为 FROM 子句“导出”，这使得它们可以更有效地在 SELECT 的列子句中使用。

+   **[sql]**

    and_()和 or_()现在生成 ColumnElement，允许布尔表达式作为结果列，例如 select([and_(1, 0)]）。

    参考：[#798](https://www.sqlalchemy.org/trac/ticket/798)

+   **[sql]**

    绑定参数现在是 ColumnElement 的子类，这使得它们可以被 orm.query 选择（它们已经具有大多数 ColumnElement 语义）。

+   **[sql]**

    为 exists()构造添加了 select_from()方法，使其与常规 select()更加兼容。

+   **[sql]**

    添加了 func.min()，func.max()，func.sum()作为“通用函数”，基本上允许它们的返回类型自动确定。有助于 SQLite 上的日期，十进制类型等。

    参考：[#1160](https://www.sqlalchemy.org/trac/ticket/1160)

+   **[sql]**

    添加了 decimal.Decimal 作为“自动检测”类型；当使用 Decimal 时，绑定参数和通用函数将将它们的类型设置为 Numeric。

### 模式

+   **[schema]**

    添加了“sorted_tables”访问器到 MetaData，它返回按依赖顺序排序的 Table 对象列表。这使得 MetaData.table_iterator()方法已被弃用。util.sort_tables()中的“reverse=False”关键字参数也已被移除；使用 Python 的‘reversed’函数来反转结果。

    参考：[#1033](https://www.sqlalchemy.org/trac/ticket/1033)

+   **[schema]**

    所有数字类型的“长度”参数已更名为“精度”。 “长度”已被弃用，但仍会发出警告。

+   **[schema]**

    放弃了对用户定义类型（convert_result_value，convert_bind_param）的 0.3 兼容性。

### mysql

+   **[mysql]**

    MSInteger，MSBigInteger，MSTinyInteger，MSSmallInteger 和 MSYear 的“长度”参数已更名为“显示宽度”。

+   **[mysql]**

    添加了 MSMediumInteger 类型。

    参考：[#1146](https://www.sqlalchemy.org/trac/ticket/1146)

+   **[mysql]**

    函数 func.utc_timestamp()编译为 UTC_TIMESTAMP，不带括号，这似乎在与 executemany()一起使用时会有问题。

### oracle

+   **[oracle]**

    limit/offset 不再使用 ROW NUMBER OVER 来限制行数，而是与特殊的 Oracle 优化注释一起使用子查询。允许 LIMIT/OFFSET 与 DISTINCT 一起使用。

    参考：[#536](https://www.sqlalchemy.org/trac/ticket/536)

+   **[oracle]**

    has_sequence()现在考虑当前的“模式”参数

    参考：[#1155](https://www.sqlalchemy.org/trac/ticket/1155)

+   **[oracle]**

    添加了 BFILE 到反射类型名称

    参考：[#1121](https://www.sqlalchemy.org/trac/ticket/1121)

### 杂项

+   **[declarative]**

    修复了如果复合主键引用另一个尚未定义的表，则映射器无法初始化的 bug。

    参考：[#1161](https://www.sqlalchemy.org/trac/ticket/1161)

+   **[declarative]**

    修复了在使用基于字符串的 primaryjoin 条件与 backref 结合使用时会抛出异常的 bug。

### orm

+   **[orm]**

    Query 现在具有 delete() 和 update(values) 方法。这允许使用 Query 对象执行批量删除/更新。

+   **[orm]**

    Query(*cols) 返回的 RowTuple 对象现在具有 keynames，优先使用映射属性名称而不是列键，列键优先于列名称，即 Query(Class.foo, Class.bar) 将具有名称“foo”和“bar”，即使这些不是底层 Column 对象的名称���直接的 Column 对象，如 Query(table.c.col)，将返回 Column 的“key”属性。

+   **[orm]**

    Query 现在具有 scalar() 和 value() 方法，每个方法返回一个单个标量值。scalar() 不带参数，大致相当于 first()[0]，value() 接受一个列表达式，大致相当于 values(expr).next()[0]。

+   **[orm]**

    改进了在将 SQL 表达式放置在实体的 query() 列表中时确定 FROM 子句的方法。特别是标量子查询不应该将其内部的 FROM 对象泄漏到封闭查询中。

+   **[orm]**

    从映射类到映射子类的 relation() 连接，其中映射子类配置为单表继承，将包括一个 IN 子句，限制连接类的子类型为请求的子类型，在连接的 ON 子句中生效。这适用于急加载连接以及 query.join()。请注意，在某些情况下，由于此区分具有多个触发点，IN 子句也会出现在查询的 WHERE 子句中。

+   **[orm]**

    AttributeExtension 已经得到改进，使事件在实际发生变化之前触发。此外，append() 和 set() 方法现在必须返回给定的值，该值用作要在变异操作中使用的值。这允许创建验证 AttributeListeners，在实际操作发生之前引发，并且可以将给定的值更改为其他值。

+   **[orm]**

    column_property()、composite_property() 和 relation() 现在接受一个或多个 AttributeExtensions，使用“extension”关键字参数。

+   **[orm]**

    query.order_by().get() 会在 GET 发出的查询中静默删除“ORDER BY”，但不会引发异常。

+   **[orm]**

    添加了一个验证器属性扩展（Validator AttributeExtension），以及一个 @validates 装饰器，类似于 @reconstructor，标记一个方法验证一个或多个映射属性。

+   **[orm]**

    class.someprop.in_() 在 relation 的“in_”实现之前会引发 NotImplementedError

    参考：[#1140](https://www.sqlalchemy.org/trac/ticket/1140)

+   **[orm]**

    修复了在尚未加载集合的多对多集合的主键更新问题

    参考：[#1127](https://www.sqlalchemy.org/trac/ticket/1127)

+   **[orm]**

    修复了具有组并与否无关的 synonym()的 deferred()列在延迟加载期间会产生 AttributeError 的 bug。

+   **[orm]**

    SessionExtension 上的 before_flush()钩子在最终计算新/脏/已删除列表之前发生，允许在 flush 继续之前在 before_flush()中进一步更改 Session 的状态。

    参考：[#1128](https://www.sqlalchemy.org/trac/ticket/1128)

+   **[orm]**

    “extension”参数现在可以选择性地是一个列表，支持发送到多个 SessionExtension 实例的事件。Session 将 SessionExtensions 放在 Session.extensions 中。

+   **[orm]**

    对 flush()的重入调用会引发错误。这也作为针对 Session.flush()的并发调用的基本但不是绝对可靠的检查。

+   **[orm]**

    改进了 query.join()在连接到连接表继承子类时的行为，使用显式的连接条件（即不是在关系上）。

+   **[orm]**

    @orm.attributes.reconstitute 和 MapperExtension.reconstitute 已重命名为@orm.reconstructor 和 MapperExtension.reconstruct_instance

+   **[orm]**

    修复了从基类继承的子类的@reconstructor 钩子。

    参考：[#1129](https://www.sqlalchemy.org/trac/ticket/1129)

+   **[orm]**

    composite()属性类型现在支持 composite 类上的 __set_composite_values__()方法，如果该类使用除列键名之外的属性名称表示状态，则该方法是必需的；默认生成的值现在在 flush 时正确填充。此外，将属性设置为 None 的复合体现在比较时是正确的。

    参考：[#1132](https://www.sqlalchemy.org/trac/ticket/1132)

+   **[orm]**

    attributes.get_history()返回的三元组现在可以是列表和元组的混合。（以前成员总是列表。）

+   **[orm]**

    修复了在属性的先前值已过期的实体上更改主键属性会在 flush()时产生错误的 bug。

    参考：[#1151](https://www.sqlalchemy.org/trac/ticket/1151)

+   **[orm]**

    修复了自定义仪器 bug，其中 get_instance_dict()未对未由 ORM 加载的新构造实例调用。

+   **[orm]**

    如果给定对象尚未存在，则 Session.delete()将给定对象添加到会话中。这是从 0.4 版本开始的一个回归 bug。

    参考：[#1150](https://www.sqlalchemy.org/trac/ticket/1150)

+   **[orm]**

    Session 上的 echo_uow 标志已弃用，工作单元日志现在仅适用于应用程序级别，而不是每个会话级别。

+   **[orm]**

    从 InstrumentedAttribute 中删除了冲突的 contains()运算符，该运算符不接受 escape kwaarg。

    参考：[#1153](https://www.sqlalchemy.org/trac/ticket/1153)

### sql

+   **[sql]**

    暂时回滚了“ORDER BY”增强功能。该功能暂停等待进一步开发。

    参考：[#1068](https://www.sqlalchemy.org/trac/ticket/1068)

+   **[sql]**

    exists()构造不会将其包含的元素列表作为 FROM 子句“导出”，这样可以更有效地在 SELECT 的 columns 子句中使用它们。

+   **[sql]**

    and_()和 or_()现在生成 ColumnElement，允许布尔表达式作为结果列，即 select([and_(1, 0)]).

    参考：[#798](https://www.sqlalchemy.org/trac/ticket/798)

+   **[sql]**

    绑定参数现在是 ColumnElement 的子类，这使它们可以被 orm.query 选择（它们已经具有大多数 ColumnElement 语义）。

+   **[sql]**

    为 exists()构造添加了 select_from()方法，使其与常规 select()更加兼容。

+   **[sql]**

    添加了 func.min()、func.max()、func.sum()作为“通用函数”，基本上允许它们的返回类型自动确定。有助于 SQLite 上的日期、十进制类型等。

    参考：[#1160](https://www.sqlalchemy.org/trac/ticket/1160)

+   **[sql]**

    添加了 decimal.Decimal 作为“自动检测”类型；当使用 Decimal 时，绑定参数和通用函数将将它们的类型设置为 Numeric。

### schema

+   **[schema]**

    添加了“sorted_tables”访问器到 MetaData，它返回按依赖顺序排序的 Table 对���列表。这使 MetaData.table_iterator()方法过时。util.sort_tables()中的“reverse=False”关键字参数也已被移除；使用 Python 的‘reversed’函数来反转结果。

    参考：[#1033](https://www.sqlalchemy.org/trac/ticket/1033)

+   **[schema]**

    所有 Numeric 类型的‘length’参数已重命名为‘scale’。‘length’已被弃用，但仍会发出警告。

+   **[schema]**

    放弃了用户定义类型（convert_result_value, convert_bind_param）的 0.3 兼容性。

### mysql

+   **[mysql]**

    MSInteger、MSBigInteger、MSTinyInteger、MSSmallInteger 和 MSYear 的‘length’参数已重命名为‘display_width’。

+   **[mysql]**

    添加了 MSMediumInteger 类型。

    参考：[#1146](https://www.sqlalchemy.org/trac/ticket/1146)

+   **[mysql]**

    函数 func.utc_timestamp()编译为 UTC_TIMESTAMP，不带括号，当与 executemany()一起使用时，括号似乎会妨碍使用。

### oracle

+   **[oracle]**

    limit/offset 不再使用 ROW NUMBER OVER 来限制行数，而是与特殊的 Oracle 优化注释一起使用子查询。允许 LIMIT/OFFSET 与 DISTINCT 一起使用。

    参考：[#536](https://www.sqlalchemy.org/trac/ticket/536)

+   **[oracle]**

    has_sequence()现在考虑了当前的“schema”参数

    参考：[#1155](https://www.sqlalchemy.org/trac/ticket/1155)

+   **[oracle]**

    将 BFILE 添加到反射类型名称中

    参考：[#1121](https://www.sqlalchemy.org/trac/ticket/1121)

### 杂项

+   **[declarative]**

    修复了一个 bug，即如果复合主键引用另一个尚未定义的表，则映射器无法初始化。

    参考：[#1161](https://www.sqlalchemy.org/trac/ticket/1161)

+   **[declarative]**

    修复了在使用 backref 时使用基于字符串的 primaryjoin 条件时可能发生的异常抛出。

## 0.5.0beta3

发布日期：2008 年 8 月 4 日星期一

### orm

+   **[orm]**

    SQLAlchemy 映射器的“entity_name”功能已被移除。有关原因，请参见[`tinyurl.com/6nm2ne`](https://tinyurl.com/6nm2ne)

+   **[orm]**

    Session、sessionmaker()和 scoped_session()上的“autoexpire”标志已重命名为“expire_on_commit”。它不会影响 rollback()的过期行为。

+   **[orm]**

    修复了在映射器延迟加载继承属性时可能发生的无限循环错误。

+   **[orm]**

    为 Session 添加了一个名为“_enable_transaction_accounting”的遗留支持标志，当为 False 时，禁用所有事务级对象计数，包括回滚时的过期，提交时的过期，新建/删除列表维护以及在开始时的自动刷新。

+   **[orm]**

    relation()的“cascade”参数接受 None 作为值，这等效于没有级联。

+   **[orm]**

    修复动态关系的关键问题，允许在 flush()之后正确清除“修改后”的历史记录。

+   **[orm]**

    类上的用户定义的@property 在映射器初始化期间被检测并保留在原位。这意味着如果@property 挡住了同名的表绑定列（且该列未被重新映射为不同名称），或者从继承类中继承的 instrumented 属性被应用，那么同名的表绑定列将根本不会被映射。使用 include_properties/exclude_properties 集合排除的名称也适用相同的规则。

+   **[orm]**

    添加了一个名为 after_attach()的新 SessionExtension 钩子。这在通过 add()、add_all()、delete()和 merge()附加对象时调用。

+   **[orm]**

    继承自另一个映射器的映射器，在继承其继承映射器的列时，将使用在该继承映射器中指定的任何重新分配的属性名称。以前，如果“Base”将“base_id”重新分配为名称“id”，那么“SubBase(Base)”仍将获得名为“base_id”的属性。可以通过在每个子映射器中明确声明列来解决此问题，但这在使用声明性时也是相当不可行的。

    参考：[#1111](https://www.sqlalchemy.org/trac/ticket/1111)

+   **[orm]**

    修复了 Session 中潜在的一系列竞争条件，异步 GC 可能会在 session.expunge_all()和相关方法期间将未修改且不再引用的项目从会话中移除，因为它们存在于要处理的项目列表中。

+   **[orm]**

    对于应该减少“在编译期间未替换属性 x”警告概率的 _CompileOnAttr 机制进行了一些改进。（这通常适用于 SQLA 黑客，如 Elixir 开发人员）。

+   **[orm]**

    修复了一个 bug，即对于挂起的孤立实例，如果在生成负责错误的关系列表时不考虑超类映射器，则会引发“未保存的挂起实例”FlushError。

### sql

+   **[sql]**

    func.count()如果没有参数，则呈现为 COUNT(*)，相当于 func.count(text(‘*’))。

+   **[sql]**

    ORDER BY 表达式中的简单标签名称呈现为它们自身，而不是其对应表达式的重新陈述。此功能目前仅对 SQLite、MySQL 和 PostgreSQL 启用。可以在其他方言上启用此行为，只要每个方言都支持此行为。

    参考：[#1068](https://www.sqlalchemy.org/trac/ticket/1068)

### mysql

+   **[mysql]**

    在创建表时，对于 MSEnum 值的引用现在是可选的，并且将根据需要进行引用。（对于现有表的使用，引用始终是可选的。）

    参考：[#1110](https://www.sqlalchemy.org/trac/ticket/1110)

### 杂项

+   **[ext]**

    作为参数发送给 relation()的 remote_side 和 foreign_keys 参数的类绑定属性现在被接受，允许它们与 declarative 一起使用。此外，修复了在与急加载一起指定为类绑定属性的 order_by 时出现的错误。

+   **[ext]**

    调整了列的声明式初始化，使得非重命名列以与非声明式映射器相同的方式初始化。这允许继承映射器设置其同名的“id”列，以便父“id”列优先于子列，减少请求此值时的数据库往返。

### orm

+   **[orm]**

    SQLAlchemy 映射器的“entity_name”功能已被移除。有关原因，请参见[`tinyurl.com/6nm2ne`](https://tinyurl.com/6nm2ne)

+   **[orm]**

    Session、sessionmaker()和 scoped_session()上的“autoexpire”标志已重命名为“expire_on_commit”。它不影响 rollback()的过期行为。

+   **[orm]**

    修复了在映射器的延迟加载中可能发生的无限循环错误。

+   **[orm]**

    为 Session 添加了一个名为“_enable_transaction_accounting”的遗留支持标志，当为 False 时，禁用所有事务级对象计数，包括回滚时的 expire，提交时的 expire，新建/删除列表维护以及 begin 时的自动刷新。

+   **[orm]**

    relation()的“cascade”参数接受 None 作为值，这等效于不进行级联。

+   **[orm]**

    对于动态关系的关键修复允许在 flush()后正确清除“modified”历史记录。

+   **[orm]**

    类中的用户定义的@property 属性在映射器初始化期间被检测并保留在原位。这意味着如果@property 属性挡在路上，那么具有相同名称的表绑定列将根本不被映射（且列未被重新映射为不同名称），也不会应用来自继承类的 instrumented 属性。使用 include_properties/exclude_properties 集合排除的名称也适用相同的规则。

+   **[orm]**

    添加了一个名为 after_attach()的新的 SessionExtension 钩子。这在通过 add()、add_all()、delete()和 merge()附加对象时调用。

+   **[orm]**

    当一个继承自另一个映射器的映射器继承其继承映射器中重新分配的属性名称时，将使用在继承映射器中指定的重新分配的属性名称。以前，如果“Base”将“base_id”重新分配为名称“id”，那么“SubBase(Base)”仍将获得一个名为“base_id”的属性。可以通过在每个子映射器中明确声明列来解决此问题，但这在使用声明性时也是相当不可行的。

    参考：[#1111](https://www.sqlalchemy.org/trac/ticket/1111)

+   **[orm]**

    修复了 Session 中潜在的一系列竞争条件，在异步 GC 可能会从会话中删除未修改���、不再引用的项目，因为它们存在于要处理的项目列表中，通常在 session.expunge_all()和相关方法期间。

+   **[orm]**

    对于应该减少“在编译期间未替换属性 x”警告概率的 _CompileOnAttr 机制进行了一些改进（这通常适用于 SQLA 黑客，如 Elixir 开发人员）。

+   **[orm]**

    修复了一个 bug，即对于挂起的孤立实例引发的“未保存的挂起实例”FlushError 在生成负责错误的关系列表时不会考虑超类映射器。

### sql

+   **[sql]**

    没有参数的 func.count()呈现为 COUNT(*)，相当于 func.count(text(‘*’))。

+   **[sql]**

    ORDER BY 表达式中的简单标签名称呈现为它们自身，而不是其对应表达式的重新陈述。目前，此功能仅对 SQLite、MySQL 和 PostgreSQL 启用。可以在其他方言上启用此行为，因为每个方言都显示支持此行为。

    参考：[#1068](https://www.sqlalchemy.org/trac/ticket/1068)

### mysql

+   **[mysql]**

    用于在 CREATE TABLE 中使用的 MSEnum 值的引用现在是可选的，并将根据需要进行引用。 （对于与现有表一起使用，引用始终是可选的。）

    参考：[#1110](https://www.sqlalchemy.org/trac/ticket/1110)

### 杂项

+   **[ext]**

    作为 relation()的 remote_side 和 foreign_keys 参数的参数发送给类绑定属性现在被接受，允许它们与声明性一起使用。此外，修复了在与急加载一起指定 order_by 作为类绑定属性时涉及的错误。

+   **[ext]**

    调整了列的声明式初始化，使得非重命名列以与非声明式映射器相同的方式初始化。这允许继承映射器特别设置其同名的“id”列，以便父“id”列优先于子列，减少在请求此值时的数据库往返次数。

## 0.5.0beta2

发布日期：Mon Jul 14 2008

### orm

+   **[orm]**

    除了过期属性外，延迟属性也会在结果集中存在其数据时加载。

    参考：[#870](https://www.sqlalchemy.org/trac/ticket/870)

+   **[orm]**

    如果属性列表不包含任何基于列的属性，则 session.refresh()会引发一个信息性错误消息。

+   **[orm]**

    如果未指定列或映射器，则 query() 会引发一个信息丰富的错误消息。

+   **[orm]**

    惰性加载器现在在继续之前会触发自动刷新。这允许在自动刷新的上下文中正确地执行集合或标量关系的 expire()。

+   **[orm]**

    column_property() 属性代表 SQL 表达式或映射表中不存在的列（例如来自视图的列），在 INSERT 或 UPDATE 后会自动过期，假设它们没有被本地修改，以便在访问时使用最新的数据进行刷新。

    参考文献：[#887](https://www.sqlalchemy.org/trac/ticket/887)

+   **[orm]**

    修复了在使用 query.join(cls, aliased=True) 时两个连接表继承映射器之间的显式自引用连接。

    参考文献：[#1082](https://www.sqlalchemy.org/trac/ticket/1082)

+   **[orm]**

    修复了在与仅包含列子句和 SQL 表达式 ON 子句一起使用时的 query.join()。

+   **[orm]**

    mapper() 中的 “allow_column_override” 标志已被移除。这个标志几乎总是被误解。其特定功能可以通过 include_properties/exclude_properties mapper 参数来实现。

+   **[orm]**

    修复了 Query 上的 __str__() 方法。

    参考文献：[#1066](https://www.sqlalchemy.org/trac/ticket/1066)

+   **[orm]**

    即使定义了表/映射器特定绑定，Session.bind 也会被默认使用。

### sql

+   **[sql]**

    添加了新的 match() 操作符，执行全文搜索。在 PostgreSQL、SQLite、MySQL、MS-SQL 和 Oracle 后端支持。

### schema

+   **[schema]**

    向 Table 添加了 prefixes 选项，它接受一个字符串列表，用于在 CREATE TABLE 语句中的 CREATE 之后插入。

    参考文献：[#1075](https://www.sqlalchemy.org/trac/ticket/1075)

+   **[schema]**

    Unicode、UnicodeText 类型现在默认设置 “assert_unicode” 和 “convert_unicode”，但可以接受这些值的覆盖 **kwargs。

### 扩展

+   **[extensions]**

    Declarative 支持一个名为 __table_args__ 的类变量，它可以是一个字典，或者一个形如 (arg1, arg2, …, {kwarg1:value, …}) 的元组，其中包含传递给 Table 构造函数的位置参数和关键字参数。

    参考文献：[#1096](https://www.sqlalchemy.org/trac/ticket/1096)

### sqlite

+   **[sqlite]**

    修改了 SQLite 对 “微秒” 的表示，使其与 str(somedatetime) 的输出匹配，即微秒以字符串格式的小数秒表示。这使得 SQLA 的 SQLite 日期类型与直接使用 Pysqlite 保存的日期时间兼容（它只是调用 str()）。请注意，这与 SQLA 0.4 生成的 SQLite 数据库文件中现有的微秒值不兼容。

    要全局获取旧行为：

    > from sqlalchemy.databases.sqlite import DateTimeMixin DateTimeMixin.__legacy_microseconds__ = True

    要在个别 DateTime 类型上获取行为：

    > t = sqlite.SLDateTime() t.__legacy_microseconds__ = True

    然后在 Column 上使用 “t” 作为类型。

    参考文献：[#1090](https://www.sqlalchemy.org/trac/ticket/1090)

+   **[sqlite]**

    SQLite 的 Date、DateTime 和 Time 类型现在只接受 Python datetime 对象，而不是字符串。如果您想要自己用 SQLite 格式化日期为字符串，请使用 String 类型。如果您希望它们无论如何返回 datetime 对象，尽管它们接受字符串作为输入，请使用 TypeDecorator 包装 String - SQLA 不鼓励这种模式。

### orm

+   **[orm]**

    除了过期属性，延迟属性也会在结果集中加载。

    参考：[#870](https://www.sqlalchemy.org/trac/ticket/870)

+   **[orm]**

    如果属性列表不包含任何基于列的属性，session.refresh()会引发一个信息性错误消息。

+   **[orm]**

    如果未指定列或映射器，query()会引发一个信息性错误消息。

+   **[orm]**

    惰性加载器现在在继续之前会触发自动刷新。这允许在自动刷新的上下文中使集合或标量关系的 expire()函数正常工作。

+   **[orm]**

    column_property()属性代表 SQL 表达式或映射表中不存在的列（例如来自视图的列），在 INSERT 或 UPDATE 后会自动过期，假设它们没有在本地修改，以便在访问时使用最新数据进行刷新。

    参考：[#887](https://www.sqlalchemy.org/trac/ticket/887)

+   **[orm]**

    修复了在使用 query.join(cls, aliased=True)时两个连接表继承映射器之间的显式自引用连接。

    参考：[#1082](https://www.sqlalchemy.org/trac/ticket/1082)

+   **[orm]**

    修复了 query.join()在与仅列子句和 SQL 表达式 ON 子句一起使用时的问题。

+   **[orm]**

    mapper()中的“allow_column_override”标志已被移除。这个标志几乎总是被误解。其特定功能可以通过 mapper 参数中的 include_properties/exclude_properties 实现。

+   **[orm]**

    修复了 Query 上的 __str__()方法。

    参考：[#1066](https://www.sqlalchemy.org/trac/ticket/1066)

+   **[orm]**

    即使定义了特定于表/映射器的绑定，Session.bind 仍然作为默认值使用。

### sql

+   **[sql]**

    添加了新的 match()操作符，执行全文搜索。在 PostgreSQL、SQLite、MySQL、MS-SQL 和 Oracle 后端支持。

### schema

+   **[schema]**

    Table 添加了一个 prefixes 选项，接受一个字符串列表，在 CREATE TABLE 语句中的 CREATE 后插入。

    参考：[#1075](https://www.sqlalchemy.org/trac/ticket/1075)

+   **[schema]**

    Unicode、UnicodeText 类型现在默认设置为“assert_unicode”和“convert_unicode”，但接受这些值的覆盖**kwargs。

### extensions

+   **[extensions]**

    Declarative 支持一个 __table_args__ 类变量，它可以是一个字典，或者是一个元组形式的(arg1, arg2, …, {kwarg1:value, …})，其中包含传递给 Table 构造函数的位置参数和关键字参数。

    参考：[#1096](https://www.sqlalchemy.org/trac/ticket/1096)

### sqlite

+   **[sqlite]**

    修改了 SQLite 对“微秒”的表示，使其与 str(somedatetime) 的输出匹配，即微秒以字符串格式表示为分数秒。这使得 SQLA 的 SQLite 日期类型与直接使用 Pysqlite（仅调用 str()）保存的日期时间兼容。请注意，这与 SQLA 0.4 生成的 SQLite 数据库文件中现有微秒值不兼容。

    要全局获得旧的行为：

    > from sqlalchemy.databases.sqlite import DateTimeMixin DateTimeMixin.__legacy_microseconds__ = True

    要在个别 DateTime 类型上获得此行为：

    > t = sqlite.SLDateTime() t.__legacy_microseconds__ = True

    然后在 Column 上使用“t”作为类型。

    参考：[#1090](https://www.sqlalchemy.org/trac/ticket/1090)

+   **[sqlite]**

    SQLite 的 Date、DateTime 和 Time 类型现在只接受 Python datetime 对象，不接受字符串。如果你想要用 SQLite 格式化日期为字符串，请使用 String 类型。如果你想要它们返回 datetime 对象，尽管它们接受字符串作为输入，请使用 TypeDecorator 包装 String——SQLA 不鼓励这种模式。

## 0.5.0beta1

发布日期：Thu Jun 12 2008

### general

+   **[general]**

    全局“propigate”->“propagate” 更改。

### orm

+   **[orm]**

    polymorphic_union() 函数尊重每个列的“key”，如果它们与列的名称不同。

+   **[orm]**

    修复了一个仅针对 0.4 版本的 bug，该 bug 阻止复合列与继承映射器正常工作

    参考：[#1199](https://www.sqlalchemy.org/trac/ticket/1199)

+   **[orm]**

    修复了 mapper 中的 RLock 相关 bug，该 bug 在重新进入 mapper compile() 调用时可能导致死锁，当使用 ForeignKey 对象内部的声明性构造时会发生这种情况。从 0.5 移植。

+   **[orm]**

    修复了复合类型中的 bug，该 bug 阻止了主键复合类型的变异。

    参考：[#1213](https://www.sqlalchemy.org/trac/ticket/1213)

+   **[orm]**

    添加了 ScopedSession.is_active 访问器。

    参考：[#976](https://www.sqlalchemy.org/trac/ticket/976)

+   **[orm]**

    类绑定访问器可以用作 relation() 的 order_by 参数。

    参考：[#939](https://www.sqlalchemy.org/trac/ticket/939)

+   **[orm]**

    修复了 ShardedSession.execute() 上的 shard_id 参数。

    参考：[#1072](https://www.sqlalchemy.org/trac/ticket/1072)

### sql

+   **[sql]**

    Connection.invalidate() 检查关闭状态以避免属性错误。

    参考：[#1246](https://www.sqlalchemy.org/trac/ticket/1246)

+   **[sql]**

    NullPool 支持失败时重新连接的行为。

    参考：[#1094](https://www.sqlalchemy.org/trac/ticket/1094)

+   **[sql]**

    TypeEngine 使用的每种方言缓存现在是一个 WeakKeyDictionary。这样可以防止方言对象被永久引用，对于创建任意数量的引擎或方言的应用程序来说，这是必要的。这会有一点性能损失，将在 0.6 中解决。

    参考：[#1299](https://www.sqlalchemy.org/trac/ticket/1299)

+   **[sql]**

    修复了 SQLite 反��方法，以便检测到不存在的 cursor.description，触发自动关闭游标，以便在最近版本的 pysqlite 上不会因为调用 fetchone() 时没有行而失败，后者在没有结果时会引发错误。

### mysql

+   **[mysql]**

    修复了在反射期间未出现 FK 列时引发异常的 bug。

    参考：[#1241](https://www.sqlalchemy.org/trac/ticket/1241)

### oracle

+   **[oracle]**

    修复了阻止接收某些类型的 out 参数的 bug；非常感谢 huddlej at wwu.edu！

    参考：[#1265](https://www.sqlalchemy.org/trac/ticket/1265)

### 杂项

+   **[无标签]**

    现在 mapper 添加的 "__init__" 触发器/装饰器尝试精确复制原始 __init__ 的参数签名。对于 ‘_sa_session’ 的传递不再是隐式的-您必须在构造函数中允许此关键字参数。

+   **[无标签]**

    ClassState 已更名为 ClassManager。

+   **[无标签]**

    类可以通过提供 __sa_instrumentation_manager__ 属性来提供自己的 InstrumentationManager。

+   **[无标签]**

    自定义仪器可以使用任何机制将 ClassManager 与类和 InstanceState 与实例关联起来。这些对象上的属性仍然是 SQLAlchemy 本机仪器使用的默认关联机制。

+   **[无标签]**

    从实例对象中移动了 entity_name、_sa_session_id 和 _instance_key 到实例状态。这些值仍然以旧方式可用，现在已弃用，使用附加到类的描述符。在访问时将发出弃用警告。

+   **[无标签]**

    _prepare_instrumentation 的别名 prepare_instrumentation 已被移除。

+   **[无标签]**

    sqlalchemy.exceptions 已更名为 sqlalchemy.exc。该模块可以使用任一名称导入。

+   **[无标签]**

    与 ORM 相关的异常现在在 sqlalchemy.orm.exc 中定义。在导入 sqlalchemy.orm 时，ConcurrentModificationError、FlushError 和 UnmappedColumnError 的兼容性别名将安装在 sqlalchemy.exc 中。

+   **[无标签]**

    sqlalchemy.logging 已更名为 sqlalchemy.log。

+   **[无标签]**

    过渡性别名 sqlalchemy.log.SADeprecationWarning 用于 sqlalchemy.exc 中警告的定义已被移除。

+   **[无标签]**

    exc.AssertionError 已被移除，使用 Python 的内置 AssertionError 替换。

+   **[无标签]**

    已更改附加到单个类的多个 entity_name= 主映射器的 MapperExtensions 的行为。为类定义的第一个 mapper() 是唯一符合 MapperExtension 'instrument_class'、'init_instance' 和 'init_failed' 事件的映射器。这是不兼容的；以前定义的最后一个映射器的扩展将接收这些事件。

+   **[firebird]**

    添加了从插入（仅限 2.0+）、更新和删除（仅限 2.1+）中返回值的支持。

+   **[postgres]**

    添加了对 Postgres 的索引反射支持，使用了长期被忽视的优秀补丁，由 Ken Kuhlman 提交。

    参考：[#714](https://www.sqlalchemy.org/trac/ticket/714)

### 一般的

+   **[一般的]**

    全局“propigate”->”propagate”更改。

### orm

+   **[orm]**

    polymorphic_union()函数尊重每个 Column 的“key”，如果它们与列的名称不同。

+   **[orm]**

    修复了仅在 0.4 中阻止复合列与继承映射器正常工作的 bug

    参考：[#1199](https://www.sqlalchemy.org/trac/ticket/1199)

+   **[orm]**

    修复了 mapper 中与 RLock 相关的 bug，该 bug 可能在重新进入的 mapper compile()调用时发生死锁，这在 ForeignKey 对象内部使用声明性构造时会发生。从 0.5 移植。

+   **[orm]**

    修复了组合类型中的错误，该错误阻止了主键组合类型的变异。

    参考：[#1213](https://www.sqlalchemy.org/trac/ticket/1213)

+   **[orm]**

    添加了 ScopedSession.is_active 访问器。

    参考：[#976](https://www.sqlalchemy.org/trac/ticket/976)

+   **[orm]**

    类绑定访问器可以用作 relation() order_by 的参数。

    参考：[#939](https://www.sqlalchemy.org/trac/ticket/939)

+   **[orm]**

    修复了 ShardedSession.execute()中的 shard_id 参数。

    参考：[#1072](https://www.sqlalchemy.org/trac/ticket/1072)

### sql

+   **[sql]**

    Connection.invalidate()检查关闭状态以避免属性错误。

    参考：[#1246](https://www.sqlalchemy.org/trac/ticket/1246)

+   **[sql]**

    NullPool 支持失败时重新连接的行为。

    参考：[#1094](https://www.sqlalchemy.org/trac/ticket/1094)

+   **[sql]**

    TypeEngine 用于缓存特定于方言的类型的每个方言缓存现在是 WeakKeyDictionary。这是为了防止方言对象被引用，以便为创建任意数量的引擎或方言的应用程序永远存在。这会有一点性能损失，将在 0.6 中解决。

    参考：[#1299](https://www.sqlalchemy.org/trac/ticket/1299)

+   **[sql]**

    修复了 SQLite 反射方法，以便检测到不存在的 cursor.description，这会触发自动关闭游标，以便在最近版本的 pysqlite 上调用 fetchone()时不会失败，因为在没有行的情���下会引发错误。

### mysql

+   **[mysql]**

    修复了在反射期间未出现 FK 列时引发异常的 bug。

    参考：[#1241](https://www.sqlalchemy.org/trac/ticket/1241)

### oracle

+   **[oracle]**

    修复了一个 bug，该 bug 阻止了某些类型的输出参数被接收；非常感谢 wwu.edu 的 huddlej！

    参考：[#1265](https://www.sqlalchemy.org/trac/ticket/1265)

### 杂项

+   **[无标签]**

    mapper 添加的“__init__”触发器/装饰器现在尝试精确地模仿原始 __init__ 的参数签名。对于‘_sa_session’的传递不再是隐式的-您必须在构造函数中允许此关键字参数。

+   **[无标签]**

    ClassState 重命名为 ClassManager。

+   **[无标签]**

    类可以通过提供 __sa_instrumentation_manager__ 属性来提供自己的 InstrumentationManager。

+   **[无标签]**

    自定义仪器可以使用任何机制将 ClassManager 与类关联，将 InstanceState 与实例关联。这些对象上的属性仍然是 SQLAlchemy 原生仪器使用的默认关联机制。

+   **[no_tags]**

    将 entity_name、_sa_session_id 和 _instance_key 从实例对象移动到实例状态。这些值仍然可以以旧方式访问，但现在已被弃用，使用附加到类的描述符。访问时将发出弃用警告。

+   **[no_tags]**

    _prepare_instrumentation 的别名 prepare_instrumentation 已被移除。

+   **[no_tags]**

    sqlalchemy.exceptions 已重命名为 sqlalchemy.exc。该模块可以使用任一名称导入。

+   **[no_tags]**

    ORM 相关的异常现在在 sqlalchemy.orm.exc 中定义。在导入 sqlalchemy.orm 时，会在 sqlalchemy.exc 中安装 ConcurrentModificationError、FlushError 和 UnmappedColumnError 的兼容别名。

+   **[no_tags]**

    sqlalchemy.logging 已重命名为 sqlalchemy.log。

+   **[no_tags]**

    过渡性的 sqlalchemy.log.SADeprecationWarning 别名已被移除。

+   **[no_tags]**

    exc.AssertionError 已被移除，使用被 Python 内置的 AssertionError 替代。

+   **[no_tags]**

    对于一个类的多个 entity_name= 主映射器附加的 MapperExtensions 的行为已更改。为一个类定义的第一个 mapper() 是唯一符合 MapperExtension 'instrument_class'、'init_instance' 和 'init_failed' 事件的映射器。这是不兼容的；以前定义的最后一个映射器的扩展将接收这些事件。

+   **[firebird]**

    增加了对插入（仅限 2.0+）、更新和删除（仅限 2.1+）返回值的支持。

+   **[postgres]**

    添加了对 Postgres 的索引反射支持，使用了 Ken Kuhlman 提交的长期被忽视的优秀补丁。

    参考：[#714](https://www.sqlalchemy.org/trac/ticket/714)
