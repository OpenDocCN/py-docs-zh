# 0.2 更新日志

> 原文：[`docs.sqlalchemy.org/en/20/changelog/changelog_02.html`](https://docs.sqlalchemy.org/en/20/changelog/changelog_02.html)

## 0.2.8

发布日期：2006 年 9 月 5 日星期二

+   **[无标签]**

    清理连接方法和文档。在查询字符串中指定自定义 DBAPI 参数，“create_engine”中的‘connect_args’参数，或通过‘creator’函数指定自定义创建函数到‘create_engine’。

+   **[无标签]**

    在 Pool 中添加了“recycle”参数，在 create_engine 中是“pool_recycle”，默认为 3600 秒；超过此时间的连接将被关闭并替换为新连接，以处理自动关闭陈旧连接的数据库

    参考：[#274](https://www.sqlalchemy.org/trac/ticket/274)

+   **[无标签]**

    使用池化连接时更改了“invalidate”语义；将指示底层连接记录在下一次调用时重新连接。“invalidate”也将在底层调用 connection.cursor()时抛出任何错误时自动调用。这将希望连接池能够重新连接到已停止并重新启动的数据库，而无需重新启动连接应用程序

    参考：[#121](https://www.sqlalchemy.org/trac/ticket/121)

+   **[无标签]**

    哎呀！教程的 doctest 很长一段时间都是错误的。

+   **[无标签]**

    在映射器上的 add_property()方法会执行“编译所有映射器”步骤，以防所给属性引用未编译的映射器（就像在教程中的情况一样！）

+   **[无标签]**

    在创建之前检查 pg 序列是否已经存在

    参考：[#277](https://www.sqlalchemy.org/trac/ticket/277)

+   **[无标签]**

    如果通过 MapperExtension.get_session（例如使用 sessioncontext 插件等）建立了上下文会话，则懒加载操作将默认使用该会话，如果父对象尚未与会话关联。

+   **[无标签]**

    对于没有数据库标识的对象，惰性加载不会触发（为什么？请参见[`www.sqlalchemy.org/trac/wiki/WhyDontForeignKeysLoadData`](https://www.sqlalchemy.org/trac/wiki/WhyDontForeignKeysLoadData)）

+   **[无标签]**

    工作单元对于“delete-orphan”级联中的“孤立”对象进行了更好的检查，适用于某些情况下父对象无法进行级联的情况。

+   **[无标签]**

    映射器可以根据与属性包的交互来判断它们的对象是否是“孤立”的。此检查基于每个关系维护的状态标志，当对象相互附加和分离时进行维护。

+   **[无标签]**

    现在，使用“delete-orphan”声明自引用关系是无效的（因为上述检查会使它们无法保存）。

+   **[无标签]**

    改进了在工作单元试图将对象作为关系的一部分进行 flush()时检查对象是否属于会话的方法。

+   **[无标签]**

    语句执行支持在表达式中多次使用相同的 BindParam 对象；简化了位置参数的处理。Bill Noon 在基本思想方面做得很好。

    参考：[#280](https://www.sqlalchemy.org/trac/ticket/280)

+   **[无标签]**

    postgres 反射移至使用 pg_schema 表，可以通过传递 use_information_schema=True 参数给 create_engine 来覆盖。

    参考：[#60](https://www.sqlalchemy.org/trac/ticket/60)，[#71](https://www.sqlalchemy.org/trac/ticket/71)

+   **[无标签]**

    在 MetaData、Table、Column 中添加了 case_sensitive 参数，根据父模式项是否具有标志的非 None 设置或否，以及标识符名称是否全为小写来自动确定。当设置为 True 时，对具有混合或大写标识符的标识符应用引用。对已知为保留字或包含其他非标准字符的标识符也自动应用引用。各种数据库方言可以覆盖所有这些行为，但目前它们都使用默认行为。已在 postgres、mysql、sqlite、oracle 上进行了测试。需要在 firebird、ms-sql 上进行更多测试。正在进行的工作的一部分

    参考：[#155](https://www.sqlalchemy.org/trac/ticket/155)

+   **[无标签]**

    单元测试已更新，可以在没有安装任何 pysqlite 的情况下运行；池测试使用模拟的 DBAPI

+   **[无标签]**

    URL 支持密码中的转义字符

    参考：[#281](https://www.sqlalchemy.org/trac/ticket/281)

+   **[无标签]**

    添加了对 UNION 查询的 limit/offset 支持（尽管 oracle 中还没有）

+   **[无标签]**

    添加了“timezone=True”标志到 DateTime 和 Time 类型。到目前为止，postgres 将其转换为“TIME[STAMP] (WITH|WITHOUT) TIME ZONE”，以便更可控地控制时区的存在（如果可用，psycopg2 会返回带有 tzinfo 的 datetimes，这可能会与不带 tzinfo 的 datetimes 造成混淆）。

+   **[无标签]**

    修复了使用 query.count()与 distinct、SelectResults count()的 kwargs 的问题

    参考：[#287](https://www.sqlalchemy.org/trac/ticket/287)

+   **[无标签]**

    当 autoload 失败时，从 MetaData 中注销 Table；

    参考：[#289](https://www.sqlalchemy.org/trac/ticket/289)

+   **[无标签]**

    导入 py2.5 的 sqlite3

    参考：[#293](https://www.sqlalchemy.org/trac/ticket/293)

+   **[无标签]**

    修复了 startswith()/endswith()的 unicode 问题

    参考：[#296](https://www.sqlalchemy.org/trac/ticket/296)

## 0.2.7

发布日期：Sat Aug 12 2006

+   **[无标签]**

    引用设施设置，以便在所有查询/创建/删除中为单个表、模式和列标识符启用特定于数据库的引用。通过在 Table 或 Column 中启用“quote=True”以及在 Table 中启用“quote_schema=True”来启用。感谢 Aaron Spike 的出色工作。

+   **[无标签]**

    assignmapper 设置 is_primary=True，导致设置冗余映射器时未引发错误，已修复

+   **[无标签]**

    在 Mapper 中添加了 allow_null_pks 选项，允许某些主键列为空的行（例如映射到外连接等情况）

+   **[无标签]**

    修改 unitofwork 以不在“new”列表或 UOWTask“objects”列表中维护排序；相反，新对象在向会话注册为新对象时被标记为排序标识符，并且 INSERT 语句然后在映射器 save_obj 中排序。INSERT 排序基本上被推迟到刷新周期的最后。这样，在 UOWTask 中发生的各种排序和组织（特别是循环任务排序）就不必担心维护顺序（它们本来也不会）

+   **[无标签]**

    修复了外键反射以自动加载引用表（如果尚未加载）

+   **[无标签]**

    +   将 URL 查询字符串参数传递给 connect()函数

    参考：[#256](https://www.sqlalchemy.org/trac/ticket/256)

+   **[无标签]**

    +   oracle 布尔类型

    参考：[#257](https://www.sqlalchemy.org/trac/ticket/257)

+   **[无标签]**

    在关系中自定义主/次连接条件 *将* 默认传播到 backrefs。指定 backref()将覆盖此行为。

+   **[无标签]**

    在 sql.Join 中更好地检查模糊的连接条件；在 PropertyLoader 中传播到更好的错误消息（即 relation()/backref()）以处理无法合理确定连接条件的情况。

+   **[无标签]**

    sqlite 在表反射时正确创建 ForeignKeyConstraint 对象。

+   **[无标签]**

    由于为所做的更改而起源的池调整。如果连接实际成功，溢出计数器应该只在连接成功时递减。添加了一个测试脚本来尝试测试这一点。

    参考：[#224](https://www.sqlalchemy.org/trac/ticket/224)

+   **[无标签]**

    修复了 mysql 反射默认值为 PassiveDefault

+   **[无标签]**

    在 MS-SQL 中添加了反射的‘tinyint’、‘mediumint’类型。

    参考：[#263](https://www.sqlalchemy.org/trac/ticket/263), [#264](https://www.sqlalchemy.org/trac/ticket/264)

+   **[无标签]**

    SingletonThreadPool 有一个大小，并进行清理，以便只有给定数量的线程本地连接保持活动（对于大量释放线程的 sqlite 应用程序是必要的）

+   **[无标签]**

    修复了懒加载器中的小 pickle 错误

    参考：[#265](https://www.sqlalchemy.org/trac/ticket/265), [#267](https://www.sqlalchemy.org/trac/ticket/267)

+   **[无标签]**

    修复了 mysql 反射可能出现的错误，某些版本返回数组而不是字符串以进行 SHOW CREATE TABLE 调用

+   **[无标签]**

    修复了映射到连接时的懒加载

    参考：[#1770](https://www.sqlalchemy.org/trac/ticket/1770)

+   **[无标签]**

    所有 create()/drop()调用都有一个“connectable”的关键字参数。“engine”已被弃用。

+   **[无标签]**

    修复了 ms-sql connect()与 adodbapi 一起工作的问题

+   **[无标签]**

    向 Select()添加了“nowait”标志

+   **[无标签]**

    继承检查使用 issubclass()而不是直接的 __mro__ 检查，以确保类 A 继承自 B，使映射继承更灵活地对应于类继承

    参考：[#271](https://www.sqlalchemy.org/trac/ticket/271)

+   **[无标签]**

    当在具有 ORDER BY 子句的 SelectResults 上调用聚合函数（如 max、min 等）时，SelectResults 将使用子选择

    参考：[#252](https://www.sqlalchemy.org/trac/ticket/252)

+   **[无标签]**

    对类型进行了修复，以便更容易使用特定于数据库的类型；修复了与此方法一起使用 mysql 文本类型的问题

    参考：[#269](https://www.sqlalchemy.org/trac/ticket/269)

+   **[无标签]**

    对 sqlite 日期类型组织进行了一些修复

+   **[无标签]**

    将 MSTinyInteger 添加到 MS-SQL

    参考：[#263](https://www.sqlalchemy.org/trac/ticket/263)

## 0.2.6

发布日期：2006 年 7 月 20 日星期四

+   **[无标签]**

    对模式进行了重大改进，以允许真正的复合主键和外键约束，通过新的 ForeignKeyConstraint 和 PrimaryKeyConstraint 对象。现有的主键/外键创建方法没有改变，但在幕后使用这些新对象。表的创建和反射现在更加面向表而不是面向列。

    参考：[#76](https://www.sqlalchemy.org/trac/ticket/76)

+   **[无标签]**

    对 MapperExtension 调用方案进行了全面改进，之前的工作效果不是很好

+   **[无标签]**

    对 ActiveMapper 进行了调整，支持自引用关系

+   **[无标签]**

    对 objectstore（在 activemapper/threadlocal 中）进行了轻微重新排列，以便 SessionContext 通过'.context'引用而不是直接子类化。

+   **[无标签]**

    如果在导入 activemapper 时激活了 mod，则 activemapper 将使用 threadlocal 的 objectstore

+   **[无标签]**

    对 URL 正则表达式进行了小修复，以允许文件名中包含'@'

+   **[无标签]**

    对 Session 的 expunge/update 等进行了修复...需要更多清理。

+   **[无标签]**

    select_table 映射器仍然不总是编译

+   **[无标签]**

    修复了 Boolean 数据类型

+   **[无标签]**

    将 count()/count_by()添加到 assignmapper 代理的方法列表中；这也将它们添加到 activemapper 中

+   **[无标签]**

    连接异常包装在 DBAPIError 中

+   **[无标签]**

    如果在映射内部类中提供了 __autoload__ = True 属性，则 ActiveMapper 现在支持从数据库自动加载列定义。目前不支持反射任何关系。

+   **[无标签]**

    在某些情况下，延迟列加载可能会在 flush()中破坏连接状态，这个问题已经修复

+   **[无标签]**

    修复了 expunge()在级联时无法工作的问题

+   **[无标签]**

    修复了级联操作中潜在的无限循环

+   **[无标签]**

    添加了“synonym()”函数，应用于属性，使一个属性名与另一个属性名相同，以便覆盖属性并允许在 select_by()中访问原始属性名。

+   **[无标签]**

    修复了在子句构造中的类型问题，特别有助于多态联合的类型问题（CAST/ColumnClause 将其类型传播到代理列）

+   **[无标签]**

    映射器编译工作正在进行中，总有一天会成功……将 MapperProperty 对象的初始化移动到所有映射器创建后以更好地处理循环编译。现在所有属性都调用 do_init()方法，如果需要，它们更加了解自己的“继承”状态。

+   **[无标签]**

    禁止在自引用关系或与继承映射器（也是自引用关系）的关系上显式急切加载

+   **[无标签]**

    减少了查询._get 中绑定参数的大小，以满足挑剔的 Oracle

    参考：[#244](https://www.sqlalchemy.org/trac/ticket/244)

+   **[无标签]**

    在 table.create()/table.drop()中添加了‘checkfirst’参数，以及 table.exists()

    参考：[#234](https://www.sqlalchemy.org/trac/ticket/234)

+   **[无标签]**

    对继承进行了一些其他持续修复

    参考：[#245](https://www.sqlalchemy.org/trac/ticket/245)

+   **[无标签]**

    属性/backref/orphan/history-tracking 调整如常…

## 0.2.5

发布日期：2006 年 7 月 8 日

+   **[无标签]**

    修复了在 select_by()中的无限循环错误，如果遍历命中两个相互引用的映射器

+   **[无标签]**

    将所有单元测试升级为将‘./lib/’插入 sys.path，解决了新的 setuptools PYTHONPATH 破坏行为

+   **[无标签]**

    进一步修复属性/依赖关系等……

+   **[无标签]**

    改进了当 DynamicMetaData 未连接时的错误处理

+   **[无标签]**

    MS-SQL 支持基本正常（已使用 pymssql 进行测试）

+   **[无标签]**

    组内 UPDATE 和 DELETE 语句的排序现在按主键值的顺序进行，以获得更具确定性的排序

+   **[无标签]**

    在插入/删除/更新映射器扩展现在按对象而不是按对象-表调用

+   **[无标签]**

    进一步修复/重构映射器编译

## 0.2.4

发布日期：2006 年 6 月 27 日

+   **[无标签]**

    当映射器在映射类上设置 init.__name__ 时，支持 Python 2.3 时使用 try/except

+   **[无标签]**

    修复了线程本地引擎仍会自动提交尽管事务正在进行的错误

+   **[无标签]**

    惰性加载和延迟加载操作需要父对象在会话中才能执行操作；而以前操作只会返回空列表或 None，现在会引发异常。

+   **[无标签]**

    如果给定对象曾经附加到的会话被垃圾回收，Session.update()会稍微宽松一些；否则仍然需要显式从先前的会话中删除实例。

+   **[无标签]**

    修复了映射器编译时的错误检查

+   **[无标签]**

    修复了与排序/限制/偏移量结合使用时的急切加载问题

+   **[无标签]**

    非常引人注目：在‘CREATE TABLE’和‘(<the rest of it>’之间添加了一个空格，因为*这就是 MySQL 指示非保留字表名的方式……*

    参考：[#206](https://www.sqlalchemy.org/trac/ticket/206)

+   **[无标签]**

    对继承的更多修复，涉及到许多对多关系的正确保存

+   **[无标签]**

    修复了在指定显式模块给 mysql 方言时的 bug

+   **[无标签]**

    当 QueuePool 超时时，会引发 TimeoutError 而不是错误地建立另一个连接

+   **[无标签]**

    在池中使用 Queue.Queue 已被替换为一个经过本地修改的版本（在 py2.3/2.4 中可用！），它使用 threading.RLock 作为互斥锁。这是为了修复一个报告的情况，其中 ConnectionFairy 的 __del__() 方法在 Queue 的 get() 方法中被调用，然后通过 put() 方法将其连接返回给 Queue，导致在不使用 threading.RLock 的情况下发生重入挂起。

+   **[无标签]**

    如果主键列有外键约束，Postgres 将不会在主键列上放置 SERIAL 关键字

+   **[无标签]**

    ConnectionFairy 上的 cursor() 方法允许传播特定于数据库的扩展参数

    参考：[#221](https://www.sqlalchemy.org/trac/ticket/221)

+   **[无标签]**

    惰性加载绑定参数正确传播列类型

    参考：[#225](https://www.sqlalchemy.org/trac/ticket/225)

+   **[无标签]**

    新的 MySQL 类型：MSEnum、MSTinyText、MSMediumText、MSLongText 等。在数值类型中更多支持 MS 特定的长度/精度参数，补丁由 Mike Bernson 提供

+   **[无标签]**

    对连接池 invalidate() 进行了一些修复

    参考：[#224](https://www.sqlalchemy.org/trac/ticket/224)

## 0.2.3

发布日期：2006 年 6 月 17 日（星期六）

+   **[无标签]**

    重大改动，将映射器编译推迟。这样可以按任意顺序构建映射器，并且它们之间的关系在首次使用映射器时编译。

+   **[无标签]**

    修复了在使用反向引用时级联行为中的一个相当大的速度瓶颈

+   **[无标签]**

    属性检测模块已完全重写；现在更简单、更清晰，稍微更快。属性的“历史”不再在每次更改时进行微观管理，而是作为实例首次加载时创建的“CommittedState”对象的一部分。HistoryArraySet 已删除，列表属性的行为现在更加开放（即它们不再是集合）。

+   **[无标签]**

    在内部使用 py2.4 的“set”构造，当“set”不可用/需要排序时，会回退到 sets.Set。

+   **[无标签]**

    修复了事务控制，使得重复的 rollback() 调用不会失败（在较大的 try/except 事务块中 flush() 引发异常时会失败得相当严重）

+   **[无标签]**

    ”foreignkey” 参数在 relation() 中也可以是一个列表。修复了自动外键检测

    参考：[#151](https://www.sqlalchemy.org/trac/ticket/151)

+   **[无标签]**

    修复了一个 bug，导致具有模式名称的表在 MetaData 对象中未正确索引

+   **[无标签]**

    修复了重新定义“key”属性的 Column 在 ResultProxy 中未进行类型转换的 bug

    参考：[#207](https://www.sqlalchemy.org/trac/ticket/207)

+   **[无标签]**

    修复了 URL 的‘port’属性，如果存在则为整数

+   **[无标签]**

    修复了旧 bug，即如果作为“secondary”映射的多对多表具有额外列，则删除操作将无法正常工作

+   **[无标签]**

    修复了针对 UNION 查询的映射的 bug

+   **[无标签]**

    修复了当没有 DB 驱动程序时抛出的不正确异常类

+   **[无标签]**

    当反射一个不存在的表时，添加了 NonExistentTable 异常抛出

    参考：[#138](https://www.sqlalchemy.org/trac/ticket/138)

+   **[无标签]**

    对 ActiveMapper 进行了小修复，涉及一对一 backrefs，其他重构

+   **[无标签]**

    在映射类中重写的构造函数从原始类获取 __name__ 和 __doc__

+   **[无标签]**

    ���复了 selectresult.py 中关于 mapper 扩展的小 bug

    参考：[#200](https://www.sqlalchemy.org/trac/ticket/200)

+   **[无标签]**

    对 cascade_mappers 进行了小调整，目前不是非常强力支持的功能

+   **[无标签]**

    对 between()、column.between()进行了一些修正，以更好地传播类型信息

    参考：[#202](https://www.sqlalchemy.org/trac/ticket/202)

+   **[无标签]**

    如果对象无法构建，则不会添加到会话中

    参考：[#203](https://www.sqlalchemy.org/trac/ticket/203)

+   **[无标签]**

    CAST 函数已被制作为其自己的子句对象，具有自己的编译函数在 ansicompiler 中；允许 MySQL 默默地忽略大多数 CAST 调用，因为 MySQL 似乎只支持具有 Date 类型的标准 CAST 语法。为字符串、整数等提供与 MySQL 兼容的 CAST 支持，一个 TODO

## 0.2.2

发布日期：Mon Jun 05 2006

+   **[无标签]**

    对多态继承行为进行了重大改进，使其能够与邻接列表表结构一起使用

    参考：[#190](https://www.sqlalchemy.org/trac/ticket/190)

+   **[无标签]**

    对继承关系的主要修复和重构，更多单元测试

+   **[无标签]**

    修复了在 create_engine()上的“echo_pool”标志

+   **[无标签]**

    修复文档，删除了与 threadlocal 策略一起使用 close()不安全的错误信息（完全安全！）

+   **[无标签]**

    create_engine()可以接受字符串或 unicode 格式的 URL

    参考：[#188](https://www.sqlalchemy.org/trac/ticket/188)

+   **[无标签]**

    firebird 支持部分完成；感谢 James Ralston 和 Brad Clements 的努力。

+   **[无标签]**

    Oracle 的 url 翻译出现问题，已修复，如果‘database’字段存在，则将主机/端口/sid 传递给 cx_oracle makedsn()，否则直接使用‘host’字段中的 TNS 名称

+   **[无标签]**

    修复了对于 query.get()/query.load()使用 unicode 条件的 bug

+   **[无标签]**

    count()函数在 selectables 上现在使用表的主键或第一列而不是“1”作为条件，还使用标签“rowcount”而不是“count”。

+   **[无标签]**

    对映射到多个表的基本功能进行了清理，更正确地记录

+   **[无标签]**

    恢复了 global_connect()函数，将其附加到名为“default_metadata”的 DynamicMetaData 实例。如果在 Table 中留出 MetaData 参数，则将使用默认元数据。

+   **[无标签]**

    修复了会话级联行为、entity_name 传播的问题

+   **[无标签]**

    将 unittest 重新组织到子目录中

+   **[无标签]**

    对线程本地连接嵌套模式进行了更多修复

## 0.2.1

发布日期：2006 年 5 月 29 日 星期一

+   **[无标签]**

    create_engine() 的“pool”参数正确传播

+   **[无标签]**

    修复了 URL，如果未解析则引发异常，不会将空字段传递给 DB 连接字符串（例如 user:host@/db 这样的字符串在 postgres 上会出错）

+   **[无标签]**

    在 Mapper 插入并尝试获取新的主键值时进行了一些小修复

+   **[无标签]**

    重写了 TLEngine 的一半，与 ‘strategy=”threadlocal”’ 一起使用的 ComposedSQLEngine。现在它正确地实现了 engine.begin()/ engine.commit()，与 connection.begin()/trans.commit() 完全嵌套。添加了大约六个 unittest。

+   **[无标签]**

    在 pool.Pool 中发现了一个重大的“duh”问题，忘记将 WeakValueDictionary 放回去。应该检查这一点的 unittest 也悄悄地遗漏了。修复了 unittest 以确保 ConnectionFairy 正确地退出作用域。

+   **[无标签]**

    添加了一个占位符 dispose() 方法到 SingletonThreadPool，目前还没有任何功能

+   **[无标签]**

    当引发异常时自动调用 rollback()，但仅当没有正在进行的事务时（即更像是自动提交）。

+   **[无标签]**

    修复了 sqlite 中如果没有 sqlite 模块存在时引发的异常

+   **[无标签]**

    为关联对象文档添加了额外的示例细节

+   **[无标签]**

    Connection 添加了检查是否已关闭的功能

## 0.2.0

发布日期：2006 年 5 月 27 日 星期六

+   **[无标签]**

    对 Engine 系统进行了全面改进，以前的 SQLEngine 现在是一个 ComposedSQLEngine，包括各种组件，包括 Dialect、ConnectionProvider 等。这影响了所有的 db 模块以及 Session 和 Mapper。

+   **[无标签]**

    create_engine 现在只接受 RFC-1738 格式的字符串：`driver://user:password@host:port/database`

    **update** 这种格式通常但不完全符合 RFC-1738，包括“scheme”部分接受下划线而不是破折号或句点。

+   **[无标签]**

    对连接作用域方法进行了全面重写，Connection 对象现在可以直接执行子句元素，增加了显式的“close”以及在整个 Engine/ORM 中处理关闭的支持，不再依赖于内部的 __del__ 来将连接返回到池中。

    参考：[#152](https://www.sqlalchemy.org/trac/ticket/152)

+   **[无标签]**

    对 Session 接口和作用域进行了全面改进。使用 hibernate 风格的方法，包括 query(class)、save()、save_or_update() 等。默认情况下不安装 threadlocal 作用域。提供了一个绑定接口到特定 Engines 和/或 Connections 的方法，使底层 Schema 对象不需要绑定到一个 Engine。添加了一个基本的 SessionTransaction 对象，可以简单地跨多个 Engines 聚合事务。

+   **[无标签]**

    对映射器的依赖性和“级联”行为进行了彻底改革；将依赖性逻辑从 properties.py 中拆分到一个单独的模块“dependency.py”中。 “级联”行为现在是明确可控的，“delete”，“delete-orphan”等的正确实现。依赖性系统现在可以在刷新时确定子对象是否有父对象，以便在删除方面做出更好的决策。

+   **[无标签]**

    对 Schema 进行彻底改革，以构建在 MetaData 对象之上，而不是 Engine。整个 SQL/Schema 系统可以不使用任何 Engines，仅通过显式的 Connection 对象执行。通过 BoundMetaData 存在“bound”方法用于模式对象。ProxyEngine 现在通常不再需要，并被 DynamicMetaData 取代。

+   **[无标签]**

    实现了真正的多态行为，修复

    参考文献：[#167](https://www.sqlalchemy.org/trac/ticket/167)

+   **[无标签]**

    “oid” 系统已完全移至编译时行为；如果在不可用的情况下在 order_by 中使用它们，order_by 不会被编译，修复

    参考文献：[#147](https://www.sqlalchemy.org/trac/ticket/147)

+   **[无标签]**

    包装程序进行了彻底改革；“mapping”现在是“orm”，“objectstore”现在是“session”，旧的“objectstore”命名空间通过“threadlocal”模块加载（如果使用）。

+   **[无标签]**

    现在通过“import <modname>”调用 mods。扩展优先于 mods，因为 mods 全局性地进行了修改。

+   **[无标签]**

    修复 add_property 以将属性传播到继承映射器

    参考文献：[#154](https://www.sqlalchemy.org/trac/ticket/154)

+   **[无标签]**

    backrefs 自动针对其原始属性的主映射器创建自己，可以指定主/次要 join 参数来覆盖。有助于它们与多态映射器的使用

+   **[无标签]**

    实现了“表存在”功能

    参考文献：[#31](https://www.sqlalchemy.org/trac/ticket/31)

+   **[无标签]**

    ”create_all/drop_all” 添加到 MetaData 对象

    参考文献：[#98](https://www.sqlalchemy.org/trac/ticket/98)

+   **[无标签]**

    对拓扑排序算法进行了改进和修复，以及更多单元测试

+   **[无标签]**

    在文档中添加了教程页面，还可以使用自定义的 doctest 运行器来运行，以确保其正常工作。文档通常进行了改革，以处理新的代码模式

+   **[无标签]**

    进行了许多其他修复和重构。

+   **[无标签]**

    迁移指南可在 Wiki 上找到 [`www.sqlalchemy.org/trac/wiki/02Migration`](https://www.sqlalchemy.org/trac/wiki/02Migration)

## 0.2.8

发布日期：Tue Sep 05 2006

+   **[无标签]**

    连接方法和文档进行清理。在查询字符串中指定自定义 DBAPI 参数，在 'create_engine' 中使用 'connect_args' 参数，或通过 'create_engine' 的 'creator' 函数指定自定义创建函数。

+   **[无标签]**

    在 Pool 中新增了“recycle”参数，在 create_engine 中为“pool_recycle”，默认为 3600 秒；此时，超过这个年龄的连接将被关闭并替换为新连接，以处理自动关闭陈旧连接的数据库。

    参考：[#274](https://www.sqlalchemy.org/trac/ticket/274)

+   **[无标签]**

    改变了使用池化连接的“使无效”语义；将指示底层连接记录在下次调用时重新连接。“使无效”还将在底层调用 connection.cursor() 时抛出任何错误时自动调用。这将希望连接池能够重新连接到已停止并重新启动的数据库，而无需重新启动连接的应用程序。

    参考：[#121](https://www.sqlalchemy.org/trac/ticket/121)

+   **[无标签]**

    哎呀！教程中的 doctest 已经很长时间没有修复。

+   **[无标签]**

    在映射器上的 add_property() 方法会执行“编译所有映射器”步骤，以防给定的属性引用了未编译的映射器（就像教程中的情况！）

+   **[无标签]**

    在创建之前检查 pg 序列是否已存在

    参考：[#277](https://www.sqlalchemy.org/trac/ticket/277)

+   **[无标签]**

    如果通过 MapperExtension.get_session（例如使用 sessioncontext 插件等）建立了上下文会话，则惰性加载操作将默认使用该会话，如果父对象尚未与会话持久化。

+   **[无标签]**

    惰性加载不会为没有数据库标识的对象触发（为什么？请参阅 [`www.sqlalchemy.org/trac/wiki/WhyDontForeignKeysLoadData`](https://www.sqlalchemy.org/trac/wiki/WhyDontForeignKeysLoadData)）

+   **[无标签]**

    unit-of-work 对“删除孤立”级联中的“孤儿”对象进行更好的检查，对于某些情况下父级不可用于级联的情况。

+   **[无标签]**

    映射器可以根据与属性包的交互来判断它们的对象是否为“孤儿”。当对象相互附加和分离时，此检查基于为每个关系维护的状态标志。

+   **[无标签]**

    现在使用“delete-orphan”声明自引用关系是无效的（因为上述检查将使它们无法保存）

+   **[无标签]**

    当工作单元试图将它们作为关系的一部分 flush() 时，对于对象是否为会话的一部分进行了更好的检查。

+   **[无标签]**

    语句执行支持在表达式中多次使用相同的 BindParam 对象；简化了位置参数的处理。Bill Noon 在理解基本思想时做得很好。

    参考：[#280](https://www.sqlalchemy.org/trac/ticket/280)

+   **[无标签]**

    postgres 反射移至使用 pg_schema 表，可以通过传递 use_information_schema=True 参数给 create_engine 进行覆盖。

    参考：[#60](https://www.sqlalchemy.org/trac/ticket/60), [#71](https://www.sqlalchemy.org/trac/ticket/71)

+   **[无标签]**

    在 MetaData、Table、Column 中添加了 case_sensitive 参数，根据父模式项是否具有标志的非 None 设置或标识符名称是否全为小写来自动确定。当设置为 True 时，对具有混合或大写标识符的标识符应用引号。对已知为保留字或包含其他非标准字符的标识符也自动应用引号。各种数据库方言可以覆盖所有这些行为，但目前它们都使用默认行为。已在 postgres、mysql、sqlite、oracle 上进行测试。需要在 firebird、ms-sql 上进行更多测试。这是正在进行的工作的一部分

    参考：[#155](https://www.sqlalchemy.org/trac/ticket/155)

+   **[无标签]**

    更新了单元测试，可以在没有安装任何 pysqlite 的情况下运行；pool 测试使用模拟的 DBAPI

+   **[无标签]**

    urls 支持密码中的转义字符

    参考：[#281](https://www.sqlalchemy.org/trac/ticket/281)

+   **[无标签]**

    向 UNION 查询添加了 limit/offset（尽管 oracle 尚未支持）

+   **[无标签]**

    在 DateTime 和 Time 类型中添加了“timezone=True”标志。到目前为止，postgres 将其转换为“TIME[STAMP] (WITH|WITHOUT) TIME ZONE”，以便更可控地控制时区的存在（如果可用，psycopg2 会返回带有 tzinfo 的 datetimes，这可能会与不带 tzinfo 的 datetimes 造成混淆）。

+   **[无标签]**

    修复了在 distinct、**kwargs 与 SelectResults count()中使用 query.count()的问题

    参考：[#287](https://www.sqlalchemy.org/trac/ticket/287)

+   **[无标签]**

    当 autoload 失败时，从 MetaData 中注销 Table；

    参考：[#289](https://www.sqlalchemy.org/trac/ticket/289)

+   **[无标签]**

    导入 py2.5 的 sqlite3

    参考：[#293](https://www.sqlalchemy.org/trac/ticket/293)

+   **[无标签]**

    修复 startswith()/endswith()的 unicode 问题

    参考：[#296](https://www.sqlalchemy.org/trac/ticket/296)

## 0.2.7

发布日期：2006 年 8 月 12 日星期六

+   **[无标签]**

    设置引号设施，以便在所有查询/创建/删除中使用时可以为单个表、模式和列标识符启用特定于数据库的引号。通过在 Table 或 Column 中启用“quote=True”，以及在 Table 中启用“quote_schema=True”来启用。感谢 Aaron Spike 的出色努力。

+   **[无标签]**

    assignmapper 设置 is_primary=True，导致设置冗余映射器时未引发任何错误，已修复

+   **[无标签]**

    向 Mapper 添加了 allow_null_pks 选项，允许某些主键列为空的行（即在映射到外连接等情况下）

+   **[无标签]**

    修改了 unitofwork，不再在“new”列表或 UOWTask “objects” 列表中维护排序；相反，新对象在与会话注册为新对象时被标记为排序标识符，然后 INSERT 语句在映射器 save_obj 中进行排序。INSERT 排序基本上一直推迟到刷新周期的最后。这样，UOWTask 中发生的各种排序和组织（特别是循环任务排序）就不必担心维护顺序（它们本来也不会）

+   **[无标签]**

    修复了外键的反射，如果引用的表尚未加载，则自动加载

+   **[无标签]**

    +   将 URL 查询字符串参数传递给 connect() 函数

    参考：[#256](https://www.sqlalchemy.org/trac/ticket/256)

+   **[无标签]**

    +   Oracle 布尔类型

    参考：[#257](https://www.sqlalchemy.org/trac/ticket/257)

+   **[无标签]**

    在关系中自定义主/次连接条件 *将* 默认传播到反向引用。指定 backref() 将覆盖此行为。

+   **[无标签]**

    在 sql.Join 中更好地检查模糊的连接条件；在 PropertyLoader 中传播到更好的错误消息（即 relation()/backref()）以处理无法合理确定连接条件的情况。

+   **[无标签]**

    SQLite 在表反射时正确创建 ForeignKeyConstraint 对象。

+   **[无标签]**

    由于为所做的更改而进行的池调整。只有在连接实际成功时，溢出计数器才会递减。添加了一个测试脚本来尝试测试这一点。

    参考：[#224](https://www.sqlalchemy.org/trac/ticket/224)

+   **[无标签]**

    修复了 MySQL 反射默认值为 PassiveDefault 的问题

+   **[无标签]**

    在 MS-SQL 中添加了反射的 ‘tinyint’、‘mediumint’ 类型。

    参考：[#263](https://www.sqlalchemy.org/trac/ticket/263)，[#264](https://www.sqlalchemy.org/trac/ticket/264)

+   **[无标签]**

    SingletonThreadPool 具有大小并进行清理，因此只有给定数量的线程本地连接保留下来（对于大量丢弃线程的 SQLite 应用程序是必需的）

+   **[无标签]**

    修复了延迟加载器中的小 pickle bug(s)

    参考：[#265](https://www.sqlalchemy.org/trac/ticket/265)，[#267](https://www.sqlalchemy.org/trac/ticket/267)

+   **[无标签]**

    修复了 MySQL 反射中可能出现的错误，某些版本在执行 SHOW CREATE TABLE 时返回数组而不是字符串

+   **[无标签]**

    修复了映射到连接时的延迟加载问题

    参考：[#1770](https://www.sqlalchemy.org/trac/ticket/1770)

+   **[无标签]**

    所有 create()/drop() 调用都有一个名为“connectable”的关键字参数。“engine” 已被弃用。

+   **[无标签]**

    修复了 ms-sql connect() 与 adodbapi 的兼容��问题

+   **[无标签]**

    在 Select() 中添加了“nowait”标志

+   **[无标签]**

    继承检查使用 issubclass() 而不是直接的 __mro__ 检查，以确保类 A 继承自 B，使映射继承更灵活地对应于类继承

    参考：[#271](https://www.sqlalchemy.org/trac/ticket/271)

+   **[无标签]**

    当对具有 ORDER BY 子句的 SelectResults 调用聚合函数（如 max、min 等）时，SelectResults 将使用子选择

    参考：[#252](https://www.sqlalchemy.org/trac/ticket/252)

+   **[无标签]**

    修复了类型问题，以便更轻松地使用特定于数据库的类型；修复了 mysql 文本类型以使其与此方法配合使用

    参考：[#269](https://www.sqlalchemy.org/trac/ticket/269)

+   **[无标签]**

    一些修复 sqlite 日期类型组织的问题

+   **[无标签]**

    添加了 MSTinyInteger 到 MS-SQL

    参考：[#263](https://www.sqlalchemy.org/trac/ticket/263)

## 0.2.6

发布日期：Thu Jul 20 2006

+   **[无标签]**

    对模式进行了大规模改进，以允许真正的复合主键和外键约束，通过新的 ForeignKeyConstraint 和 PrimaryKeyConstraint 对象。现有的主键/外键创建方法没有改变，但在幕后使用这些新对象。表的创建和反射现在更加面向表而不是面向列。

    参考：[#76](https://www.sqlalchemy.org/trac/ticket/76)

+   **[无标签]**

    对 MapperExtension 调用方案进行了彻底改进，之前的工作效果不是很好

+   **[无标签]**

    对 ActiveMapper 进行了调整，支持自引用关系

+   **[无标签]**

    对 objectstore（在 activemapper/threadlocal 中）进行了轻微重新排列，以便 SessionContext 被引用为 ‘.context’ 而不是直接子类化。

+   **[无标签]**

    如果在导入 activemapper 时激活了 mod，activemapper 将使用 threadlocal 的 objectstore

+   **[无标签]**

    对 URL 正则表达式进行了小修复，以允许带有 ‘@’ 的文件名

+   **[无标签]**

    修复了 Session expunge/update 等的问题...需要更多清理。

+   **[无标签]**

    select_table mappers *仍然* 不总是编译

+   **[无标签]**

    修复了 Boolean 数据类型

+   **[无标签]**

    将 count()/count_by() 添加到 assignmapper 代理的方法列表中；这也将它们添加到 activemapper 中

+   **[无标签]**

    连接异常包装在 DBAPIError 中

+   **[无标签]**

    ActiveMapper 现在支持从数据库自动加载列定义，如果在映射内部类中提供了 __autoload__ = True 属性。目前这不支持反射任何关系。

+   **[无标签]**

    延迟列加载可能会在某些情况下破坏 flush() 中的连接状态，已修复

+   **[无标签]**

    expunge() 在级联时无法工作，已修复。

+   **[无标签]**

    修复了级联操作中的潜在无限循环

+   **[无标签]**

    添加了“synonym()”函数，应用于属性，使一个属性名与另一个属性名相同，以便覆盖属性并允许原始属性名在 select_by() 中可访问。

+   **[无标签]**

    修复了在子句构造中的类型问题，特别有助于 polymorphic_union 的类型问题（CAST/ColumnClause 将其类型传播到代理列）

+   **[无标签]**

    映射器编译工作正在进行中，总有一天会成功……将 MapperProperty 对象的初始化移动到所有映射器创建后，以更好地处理循环编译。现在所有属性都调用 do_init()方法，如果有的话，它们更加了解自己的“继承”状态。

+   **[无标签]**

    禁止在自引用关系或与继承映射器（也是自引用关系）的关系上急切加载。

+   **[无标签]**

    减少了查询._get 中的绑定参数大小，以取悦挑剔的 oracle。

    参考：[#244](https://www.sqlalchemy.org/trac/ticket/244)

+   **[无标签]**

    在 table.create()/table.drop()和 table.exists()中添加了‘checkfirst’参数。

    参考：[#234](https://www.sqlalchemy.org/trac/ticket/234)

+   **[无标签]**

    一些其他正在进行的继承修复。

    参考：[#245](https://www.sqlalchemy.org/trac/ticket/245)

+   **[无标签]**

    属性/backref/orphan/history-tracking 调整如常…

## 0.2.5

发布日期：2006 年 7 月 8 日 星期六

+   **[无标签]**

    修复了 select_by()中的无限循环错误，如果遍历到相互引用的两个映射器。

+   **[无标签]**

    将所有 unittests 升级为将‘./lib/’插入 sys.path，解决了新的 setuptools PYTHONPATH 破坏行为。

+   **[无标签]**

    进一步修复属性/依赖关系等等…

+   **[无标签]**

    对于未连接 DynamicMetaData 时的错误处理进行了改进。

+   **[无标签]**

    MS-SQL 支持基本正常（使用 pymssql 进行测试）。

+   **[无标签]**

    在组内的 UPDATE 和 DELETE 语句的排序现在按照主键值的顺序进行，以获得更确定的排序。

+   **[无标签]**

    现在每个对象的 after_insert/delete/update 映射器扩展都是按对象而不是按对象-按表调用的。

+   **[无标签]**

    进一步修复/重构映射器编译。

## 0.2.4

发布日期：2006 年 6 月 27 日 星期二

+   **[无标签]**

    当映射器在映射类上设置 init.__name__ 时，支持 python 2.3 的 try/except。

+   **[无标签]**

    修复了线程本地引擎仍会自动提交的错误，尽管事务正在进行中。

+   **[无标签]**

    惰性加载和延迟加载操作需要父对象在会话中才能执行操作；而以前操作只会返回一个空列表或 None，现在会引发异常。

+   **[无标签]**

    如果给定对象曾经附加到的会话被垃圾回收，Session.update()会稍微宽松一些；否则仍然需要显式从先前的会话中删除实例。

+   **[无标签]**

    修复了映射器编译，检查更多错误条件。

+   **[无标签]**

    修复了与排序/限制/偏移量结合使用时的急切加载小问题。

+   **[无标签]**

    非常引人注目：在‘CREATE TABLE’和‘(<其余部分>’之间添加了一个空格，因为*这就是 MySQL 指示非保留字表名的方式……*

    参考：[#206](https://www.sqlalchemy.org/trac/ticket/206)

+   **[无标签]**

    进一步修复继承，与许多对多关系相��的正确保存。

+   **[无标签]**

    修复了指定 mysql 方言的显式模块时的错误。

+   **[无标签]**

    当 QueuePool 超时时，会引发 TimeoutError 而不是错误地建立另一个连接

+   **[无标签]**

    在池中使用 Queue.Queue 的用法已被替换为一个本地修改版本（在 py2.3/2.4 中有效！），它使用 threading.RLock 作为互斥锁。这是为了修复一个报告的情况，其中 ConnectionFairy 的 __del__()方法在 Queue 的 get()方法中被调用，然后通过 put()方法将其连接返回给 Queue，除非使用 threading.RLock，否则会导致可重入挂起。

+   **[无标签]**

    如果主键列有外键约束，postgres 不会在主键列上放置 SERIAL 关键字

+   **[无标签]**

    ConnectionFairy 上的 cursor()方法允许传播特定于数据库的扩展参数

    参考：[#221](https://www.sqlalchemy.org/trac/ticket/221)

+   **[无标签]**

    惰性加载绑定参数正确传播列类型

    参考：[#225](https://www.sqlalchemy.org/trac/ticket/225)

+   **[无标签]**

    新的 MySQL 类型：MSEnum、MSTinyText、MSMediumText、MSLongText 等。对于数字类型中的 MS 特定长度/精度参数提供更多支持，补丁由 Mike Bernson 提供

+   **[无标签]**

    对连接池 invalidate()进行了一些修复

    参考：[#224](https://www.sqlalchemy.org/trac/ticket/224)

## 0.2.3

发布日期：2006 年 6 月 17 日（星期六）

+   **[无标签]**

    对映射器编译进行了彻底的改造。这允许映射器以任何顺序构建，并且它们之间的关系在首次使用映射器时编译。

+   **[无标签]**

    修复了级联行为中的一个相当严重的速度瓶颈，特别是在使用反向引用时

+   **[无标签]**

    属性仪器模块已完全重写；现在更简单、更清晰，稍微更快。一个属性的“历史”不再在每次更改时进行微观管理，而是在实例首次加载时创建一个“CommittedState”对象的一部分。HistoryArraySet 已经消失，列表属性的行为现在更加开放（即它们不再是集合）。

+   **[无标签]**

    内部使用 py2.4 的“set”构造，当“set”不可用/需要排序时，会回退到 sets.Set。

+   **[无标签]**

    修复了事务控制的问题，使得重复的 rollback()调用不会失败（在更大的 try/except 事务块中 flush()引发异常时会失败得相当严重）

+   **[无标签]**

    ”relation()“的”foreignkey“参数也可以是一个列表。修复了自动外键检测

    参考：[#151](https://www.sqlalchemy.org/trac/ticket/151)

+   **[无标签]**

    修复了模式名称的表在 MetaData 对象中未正确索引的 bug

+   **[无标签]**

    修复了重新定义“key”属性的 Column 未进行类型转换的 bug

    参考：[#207](https://www.sqlalchemy.org/trac/ticket/207)

+   **[无标签]**

    修复了 URL 的‘port’属性，如果存在的话，应该是一个整数

+   **[无标签]**

    修复了一个旧 bug，即如果一个映射为“secondary”的多对多表有额外列，则删除操作不起作用

+   **[无标签]**

    修复了针对 UNION 查询的映射错误

+   **[无标签]**

    修复了当没有 DB 驱动程序时抛出的错误异常类

+   **[无标签]**

    当反射一个不存在的表时，添加了 NonExistentTable 异常抛出

    参考：[#138](https://www.sqlalchemy.org/trac/ticket/138)

+   **[无标签]**

    对 ActiveMapper 进行了小修复，涉及一对一反向引用，其他重构

+   **[无标签]**

    映射类中重写的构造函数从原始类获取 __name__ 和 __doc__

+   **[无标签]**

    修复了 selectresult.py 中关于映射扩展的小错误

    参考：[#200](https://www.sqlalchemy.org/trac/ticket/200)

+   **[无标签]**

    对 cascade_mappers 进行了小调整，目前支持程度不是很强

+   **[无标签]**

    对 between()、column.between()进行了一些修复，更好地传播类型信息

    参考：[#202](https://www.sqlalchemy.org/trac/ticket/202)

+   **[无标签]**

    如果对象无法构造，则不会添加到会话中

    参考：[#203](https://www.sqlalchemy.org/trac/ticket/203)

+   **[无标签]**

    CAST 函数已经成为自己的子句对象，并在 ansicompiler 中具有自己的编译函数；允许 MySQL 静默忽略大多数 CAST 调用，因为 MySQL 似乎只支持具有 Date 类型的标准 CAST 语法。为字符串、整数等提供与 MySQL 兼容的 CAST 支持，一个 TODO

## 0.2.2

发布日期：2006 年 6 月 5 日星期一

+   **[无标签]**

    对多态继承行为进行了重大改进，使其能够与邻接列表表结构一起使用

    参考：[#190](https://www.sqlalchemy.org/trac/ticket/190)

+   **[无标签]**

    对继承关系进行了重大修复和重构，增加了更多单元测试

+   **[无标签]**

    修复了 create_engine()中的“echo_pool”标志

+   **[无标签]**

    修复了文档中的错误信息，删除了与 threadlocal 策略一起使用 close()不安全的不正确信息（完全安全！）

+   **[无标签]**

    create_engine()可以接受字符串或 Unicode 格式的 URL

    参考：[#188](https://www.sqlalchemy.org/trac/ticket/188)

+   **[无标签]**

    firebird 支持部分完成；感谢 James Ralston 和 Brad Clements 的努力。

+   **[无标签]**

    Oracle 的 url 转换出现问题，已修复，如果‘database’字段存在，则将 host/port/sid 传递给 cx_oracle 的 makedsn()，否则使用‘host’字段中的直接 TNS 名称

+   **[无标签]**

    修复了使用 unicode 条件进行 query.get()/query.load()的问题

+   **[无标签]**

    selectables 上的 count()函数现在使用表主键或第一列而不是“1”作为条件，还使用标签“rowcount”而不是“count”。

+   **[无标签]**

    对“映射到多个表”功能进行了基本清理，更正确地记录

+   **[无标签]**

    恢复了 global_connect()函数，连接到名为“default_metadata”的 DynamicMetaData 实例。如果在 Table 中不使用 MetaData 参数，则将使用默认元数据。

+   **[无标签]**

    修复了会话级联行为、entity_name 传播的问题

+   **[无标签]**

    将 unittest 重新组织到子目录中

+   **[无标签]**

    对 threadlocal 连接嵌套模式进行了更多修复

## 0.2.1

发布日期：Mon May 29 2006

+   **[无标签]**

    `pool`参数传递给 create_engine()方法

+   **[无标签]**

    修复了 URL，如果未解析则引发异常，不会将空字段传递给数据库连接字符串（例如像`user:host@/db`这样的字符串在 postgres 上会出错）。

+   **[无标签]**

    在 Mapper 插入并尝试获取新主键值时进行了小修复

+   **[无标签]**

    重写了 TLEngine 的一半，用于“strategy=”threadlocal”的 ComposedSQLEngine。现在它正确地实现了 engine.begin()/engine.commit()，与 connection.begin()/trans.commit()完全嵌套。添加了约六个单元测试。

+   **[无标签]**

    在 pool.Pool 中有一个主要的“duh”，忘记将 WeakValueDictionary 放回。应该检查此项的单元测试也被悄悄地遗漏了。修复了单元测试以确保 ConnectionFairy 正确地退出作用域。

+   **[无标签]**

    给 SingletonThreadPool 添加了占位符 dispose()方法，但目前什么也没做

+   **[无标签]**

    当引发异常时自动调用 rollback()，但仅在没有正在进行的事务时（即更像是自动提交）。

+   **[无标签]**

    修复了在 sqlite 中没有 sqlite 模块时引发异常的问题

+   **[无标签]**

    为关联对象文档添加了额外的示例细节

+   **[无标签]**

    Connection 增加了对已关闭的检查

## 0.2.0

发布日期：Sat May 27 2006

+   **[无标签]**

    对引擎系统进行了全面改造，以前是 SQLEngine 现在是由各种组件组成的 ComposedSQLEngine，包括方言、连接提供程序等。这影响了所有的 db 模块以及 Session 和 Mapper。

+   **[无标签]**

    create_engine 现在只接受 RFC-1738 样式的字符串：`driver://user:password@host:port/database`

    **更新** 此格式通常但并非完全符合 RFC-1738，包括在“scheme”部分接受下划线，而不是短横线或点。

+   **[无标签]**

    对连接作用域方法进行了全面重写，Connection 对象现在可以直接执行子句元素，明确添加了“close”以及引擎/ORM 中的支持，以正确处理关闭，不再依赖于内部的 __del__ 来将连接返回到池中。

    参考：[#152](https://www.sqlalchemy.org/trac/ticket/152)

+   **[无标签]**

    对 Session 接口和作用域进行了全面改造。使用 hibernate 风格的方法，包括 query(class)、save()、save_or_update()等。默认情况下不安装 threadlocal 作用域。提供了一个绑定接口到特定 Engines 和/或 Connections 的界面，使得底层 Schema 对象不需要绑定到 Engine。添加了一个简单地聚合多个引擎之间事务的基本 SessionTransaction 对象。

+   **[无标签]**

    对映射器的依赖性和“级联”行为进行了全面改进；依赖逻辑从 properties.py 中分离出来到一个单独的模块“dependency.py”中。“级联”行为现在可以明确控制，正确实现“delete”，“delete-orphan”等。依赖系统现在可以在刷新时确定子对象是否有父对象，以便更好地决定如何更新该子对象在数据库中关于删除的情况。

+   **[无标签]**

    对 Schema 进行了全面改革，建立在 MetaData 对象的基础上而不是 Engine。整个 SQL/Schema 系统可以在没有任何 Engine 的情况下使用，仅由显式 Connection 对象执行。通过 BoundMetaData 存在“bound”方法论用于模式对象。ProxyEngine 通常不再需要，由 DynamicMetaData 替代。

+   **[无标签]**

    实现了真正的多态行为，修复

    参考：[#167](https://www.sqlalchemy.org/trac/ticket/167)

+   **[无标签]**

    “oid”系统已完全移入编译时行为；如果它们在不可用的情况下用于 order_by，order_by 不会被编译，修复

    参考：[#147](https://www.sqlalchemy.org/trac/ticket/147)

+   **[无标签]**

    包装方式进行了全面改革；“mapping”现在是“orm”，“objectstore”现在是“session”，旧的“objectstore”命名空间通过“threadlocal”模块加载（如果使用）。

+   **[无标签]**

    现在通过“import <modname>”调用 mods。扩展优于 mods，因为 mods 是全局 monkeypatching

+   **[无标签]**

    修复 add_property 以便将属性传播到继承映射器

    参考：[#154](https://www.sqlalchemy.org/trac/ticket/154)

+   **[无标签]**

    backrefs 自动生成在其原始属性的主映射器上，可以指定主/次要连接参数以覆盖。有助于它们与多态映射器的使用

+   **[无标签]**

    实现了“table exists”函数

    参考：[#31](https://www.sqlalchemy.org/trac/ticket/31)

+   **[无标签]**

    ”create_all/drop_all”添加到 MetaData 对象

    参考：[#98](https://www.sqlalchemy.org/trac/ticket/98)

+   **[无标签]**

    对拓扑排序算法进行了改进和修复，以及更多单元测试

+   **[无标签]**

    文档中添加了教程页面，也可以使用自定义 doctest 运行器运行以确保其正常工作。文档通常已进行全面改进以处理新的代码模式

+   **[无标签]**

    许多其他修复，重构。

+   **[无标签]**

    迁移指南可在 Wiki 上找到：[`www.sqlalchemy.org/trac/wiki/02Migration`](https://www.sqlalchemy.org/trac/wiki/02Migration)
