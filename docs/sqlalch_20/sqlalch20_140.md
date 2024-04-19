# 0.1 更新日志

> 原文：[`docs.sqlalchemy.org/en/20/changelog/changelog_01.html`](https://docs.sqlalchemy.org/en/20/changelog/changelog_01.html)

## 0.1.7

发布日期：2006 年 5 月 5 日 星期五

+   **[无标签]**

    对拓扑排序算法进行了一些修复

+   **[无标签]**

    添加了对 Postgres 的 DISTINCT ON 支持（只需提供 distinct=[col1,col2..]）

+   **[无标签]**

    向 sql 表达式添加了 __mod__（%运算符）

+   **[无标签]**

    从继承映射器继承的“order_by”映射器属性

+   **[无标签]**

    修复了映射器在更新/删除时使用的列类型

+   **[无标签]**

    当 convert_unicode=True 时，反射失败，已经修复

+   **[无标签]**

    类型 类型 类型！仍然无法工作….必须再次使用 TypeDecorator :(

+   **[无标签]**

    mysql 二进制类型将数组输出转换为缓冲区，修复了 PickleType

+   **[无标签]**

    最终解决了 attributes.py 的内存泄漏问题

+   **[无标签]**

    基于支持每个数据库的数据库的 unittests 进行了限定

+   **[无标签]**

    修复了列默认值会破坏插入对象的 VALUES 子句的错误

+   **[无标签]**

    修复了表定义中带有模式名称会强制引擎连接的错误

+   **[无标签]**

    修复了在 INSERT/UPDATE 中子查询中括号无法正确工作的问题

+   **[无标签]**

    HistoryArraySet 获得了 extend()方法

+   **[无标签]**

    修复了除=之外其他比较运算符的 lazyload 支持

+   **[无标签]**

    修复了 lazyload 问题，其中连接条件中的两个比较指向相同的列

+   **[无标签]**

    向映射器添加了“construct_new”标志，将使用 __new__ 来创建实例，而不是 __init__（在 0.2 中是标准的）

+   **[无标签]**

    将 selectresults.py 添加到 SVN 中，上次漏掉了

+   **[无标签]**

    对允许从表到自身的关联表进行多对多关系的微调

+   **[无标签]**

    对多态示例中使用的“translate_row”函数进行了小修复

+   **[无标签]**

    create_engine 使用 cgi.parse_qsl 来读取查询字符串（在 0.2 中已经取消）

+   **[无标签]**

    对 CAST 操作符进行微调

+   **[无标签]**

    修复了函数名称 LOCAL_TIME/LOCAL_TIMESTAMP -> LOCALTIME/LOCALTIMESTAMP

+   **[无标签]**

    修复了编译中 ORDER BY/HAVING 的顺序

## 0.1.6

发布日期：2006 年 4 月 12 日 星期三

+   **[无标签]**

    由 Rick Morrison，Runar Petursson 提供的对 MS-SQL 的支持

+   **[无标签]**

    来自 J. Ellis 的最新 SQLSoup

+   **[无标签]**

    ActiveMapper 初步支持继承（Jeff Watkins）

+   **[无标签]**

    添加了一个“mods”系统，允许修改/增强核心功能的可插拔模块，使用函数“install_mods(*modnames)”。

+   **[无标签]**

    添加了第一个“mod”，SelectResults，它修改了 mapper selects 以返回将范围转换为 LIMIT/OFFSET 查询的生成器（Jonas Borgstr?）

+   **[无标签]**

    将 Mapper 的查询功能分解为一个独立的 Query 对象，该对象以 Session 为中心。这提高了 mapper.using(session)的性能，并使其他功能成为可能。

+   **[无标签]**

    对 objectstore/Session 进行了重构，现在保存对象的官方方式是通过 flush()方法。Session 的 begin/commit 功能被分解为 LegacySession，仍然被建立为默认行为，直到 0.2 系列。

+   **[无标签]**

    类型系统在查询编译时绑定到引擎，而不是在模式构建时。这简化了类型系统以及代理引擎。

+   **[无标签]**

    向 mapper 添加了‘version_id’关键字参数。此关键字应引用一个具有整数类型的 Column 对象，最好是非空的，该对象将用于在映射表上跟踪版本号。此数字在每次保存操作时递增，并在 UPDATE/DELETE 条件中指定，以便将其计入返回的行数，如果接收到的值不是预期计数，则会导致 ConcurrencyError。

+   **[无标签]**

    向 mapper 添加了‘entity_name’关键字参数。现在，通过类对象以及可选的 entity_name 参数（默认为 None）将 mapper 与类关联起来。可以为类创建任意数量的主 mapper，由实体名称限定。这些类的实例将通过其 entity_name 限定的 mapper 发出所有加载和保存操作，并为否则等效的对象在标识映射中维护单独的标识。

+   **[无标签]**

    对属性系统进行了全面改进。代码已经澄清，并且修复以支持对象属性的适当多态行为。

+   **[无标签]**

    向 Select 对象添加了“for_update”标志

+   **[无标签]**

    一些用于反向引用的修复

+   **[无标签]**

    修复了 postgres1 DateTime 类型

+   **[无标签]**

    文档页面大部分已切换为 Markdown 语法

## 0.1.5

发布日期：2006 年 3 月 27 日星期一

+   **[无标签]**

    向 SQLEngine 添加了 SQLSession 概念。此对象跟踪从连接池检索连接以及正在进行的事务。向 SQLEngine 添加了 push_session()和 pop_session()方法，它们将新的 SQLSession 推送/弹出到引擎上，允许在前一个连接“嵌套”内进行第二个连接的操作，从而允许嵌套事务。关于 SQLSession，以后肯定会有其他技巧。

+   **[无标签]**

    在 objectstore.Session 中添加了 nest_on 参数。这是一个 SQLEngine 或引擎列表，每当此 Session 成为活动会话（通过 objectstore.push_session()或等效方式）时，将调用 push_session()/pop_session()。这允许工作单元 Session 利用嵌套事务功能，而无需在引擎上显式调用 push_session/pop_session。

+   **[无标签]**

    将 objectstore/unitofwork 分解为单独的“会话作用域”和“uow 提交重要工作”

+   **[无标签]**

    添加了 populate_instance() 方法到 MapperExtension。允许扩展修改对象属性的填充。此方法可以调用另一个映射器上的 populate_instance() 方法，以代理从一个映射器到另一个映射器的属性填充；还内置了一些行翻译逻辑以帮助完成此操作。

+   **[no_tags]**

    修复了 Oracle8 兼容性的 “use_ansi” 标志，将 JOIN 转换为与 = 和 (+) 操作符的比较，通过了基本的单元测试。

+   **[no_tags]**

    优化了对 Oracle LIMIT/OFFSET 的支持。

+   **[no_tags]**

    Oracle 反射使用 ALL_** 视图而不是 USER_**，以从中获取更大的反射列表。

+   **[no_tags]**

    修复了 Oracle 外键反射。

    参考：[#105](https://www.sqlalchemy.org/trac/ticket/105)

+   **[no_tags]**

    objectstore.commit(obj1, obj2, ...) 添加了一个额外的步骤，以查找属性上的私有关系并删除子对象，即使它不是全局提交。

+   **[no_tags]**

    对使用继承的映射器进行了大量修复，加强了映射器上关系的概念，该关系指向映射器的“本地”表而不是它继承的表。可以让更复杂的组合模式与延迟/急加载一起使用。

+   **[no_tags]**

    添加了支持，使映射器能够根据相同表从其他映射器继承，只需指定与父/子映射器相同的表。

+   **[no_tags]**

    对属性系统进行了一些次要的速度改进，关于实例化和填充新对象。

+   **[no_tags]**

    修复了 MySQL 二进制单元测试。

+   **[no_tags]**

    INSERTs 可以接收子句元素作为 VALUES 参数，而不仅仅是文字值。

+   **[no_tags]**

    支持调用多标记函数，即 schema.mypkg.func()。

+   **[no_tags]**

    将 J. Ellis 的 SQLSoup 模块添加到扩展包中。

+   **[no_tags]**

    添加了“多态”示例，说明了从一个映射器加载多种对象类型的方法，第二种方法使用了新的 populate_instance() 方法。对映射器、UNION 构造进行了小的改进，以帮助示例。

+   **[no_tags]**

    改进/修复了 session.refresh()/session.expire()（之前可能称为“invalidate”）。

+   **[no_tags]**

    添加了 session.expunge()，完全从当前会话中删除一个对象。

+   **[no_tags]**

    添加了 *args, **kwargs 传递到 engine.transaction(func)，从而使事务化装饰器函数更容易创建。

+   **[no_tags]**

    为 ResultProxy 添加了迭代器接口：“for row in result:…”

+   **[no_tags]**

    添加了断言到 tx = session.begin(); tx.rollback(); tx.begin()，即回滚后不能使用它。

+   **[no_tags]**

    在 SQLite 中的绑定参数修复中添加了日期转换，使日期能够与 pysqlite1 兼容。

+   **[no_tags]**

    对子查询进行了改进，以更智能地构建其 FROM 子句。

    参考：[#116](https://www.sqlalchemy.org/trac/ticket/116)

+   **[no_tags]**

    将 PickleType 添加到类型中。

+   **[no_tags]**

    修复了两个与绑定参数相关的列标签错误：现在在所有相关情况下，绑定参数键名都是从列“标签”生成的，以利用过长名称规则，并检查是否与列名相同的“tablename_colname”发生了奇怪的冲突。

+   **[无标签]**

    对工作单元文档和其他文档部分进行了重大改进。

+   **[无标签]**

    修复了属性错误，如果对象已提交，则其惰性加载的列表如果尚未加载则会被清除。

+   **[无标签]**

    向引擎、连接池添加了 unique_connection()方法，以返回不属于线程本地上下文或任何当前事务的连接。

+   **[无标签]**

    在池化连接中添加了 invalidate()函数。将从池中移除连接。但仍需要处理引擎自动重新连接到陈旧数据库的问题。

+   **[无标签]**

    向列元素添加了 distinct()函数，这样您就可以执行 func.count(mycol.distinct())。

+   **[无标签]**

    向 Mapper 添加了“always_refresh”标志，创建一个始终刷新从数据库获取/选择的对象属性的映射器，覆盖任何已做的更改。

## 0.1.4

发布日期：2006 年 3 月 13 日星期一

+   **[无标签]**

    create_engine()现在使用通用化参数；主机/主机名、数据库/数据库名、密码/密码等对于所有引擎连接。使引擎 URI 更加“通用”。

+   **[无标签]**

    添加了对嵌入到列子句中的 SELECT 语句的支持，使用标志“scalar=True”。

+   **[无标签]**

    对 EagerLoading 进行了另一次全面改进，当与继承的映射器一起使用时；改进了急加载正确确定其别名查询的功能，还有针对与具有继承映射器的关系设置的情况，将创建针对特定于映射器本身的表的连接（即不会针对任何继承/进一步下降继承链的表），这可以通过使用自定义主/次要连接来覆盖。

+   **[无标签]**

    在 mapper.py 中添加了 J.Ellis 的补丁，使得 selectone()在查询返回多个对象行时抛出异常，selectfirst()则不会抛出异常。还添加了 selectfirst_by（与 get_by 同义）和 selectone_by。

+   **[无标签]**

    向 Column 添加了 onupdate 参数，将在更新语句执行时执行 SQL/python。还将“for_update=True”添加到所有 DefaultGenerator 子类。

+   **[无标签]**

    添加了对 Oracle 表反射的支持，由 Andrija Zaric 贡献；仍需解决一些关于复合主键/字典选择的错误。

+   **[无标签]**

    提交了一个初始的 Firebird 模块，等待测试。

+   **[无标签]**

    向 sql.ClauseParameters 字典对象添加了作为 compiled.get_params()结果的对象，对绑定参数进行了延迟类型处理，以便更容易访问原始值。

+   **[无标签]**

    为索引、列默认值、连接池、引擎构建添加了更多文档。

+   **[无标签]**

    对类型系统的构建进行了全面改进。使用了更简单的继承模式，以便任何通用类型都可以轻松子类化，无需 TypeDecorator。

+   **[无标签]**

    添加了“convert_unicode=False”参数到 SQLEngine，将导致所有 String 类型执行 unicode 编码/解码（使 Strings 表现得像 Unicodes）

+   **[无标签]**

    添加了‘encoding=”utf8”’参数到 engine。给定的编码将用于 Unicode 类型内所有编码/解码调用以及当 convert_unicode=True 时的 Strings。

+   **[无标签]**

    改进了对 UNION 映射的支持，添加了 polymorph.py 示例以说明多类映射对 UNION 的映射

+   **[无标签]**

    修复了 SQLite LIMIT/OFFSET 语法

+   **[无标签]**

    修复了 Oracle LIMIT 语法

+   **[无标签]**

    添加了 backref()函数，允许反向引用具有将传递给 backref 的关键字参数。

+   **[无标签]**

    Sequences 和 ColumnDefault 对象可以独立执行()/标量()

+   **[无标签]**

    SQL 函数（即 func.foo()）可以独立执行()/标量()

+   **[无标签]**

    修复了 SQL 函数，使得符合 ANSI 标准的函数，即 current_timestamp 等，不指定括号。所有其他函数都需要。

+   **[无标签]**

    添加了 settattr_clean 和 append_clean 到 SmartProperty，可以设置属性而不触发“dirty”事件或任何历史记录。用法如：myclass.prop1.setattr_clean(myobject, ‘hi’)

+   **[无标签]**

    改进了映射器在使用列默认值时的支持；映射器将从语句的执行绑定参数（预转换）中提取预执行的默认值，以将其填充到保存对象的属性中；如果任何 PassiveDefaults 已触发，则将从数据库中后获取行以填充对象。

+   **[无标签]**

    添加了‘get_session().invalidate(*obj)’方法到 objectstore，实例将在下一次属性访问时刷新自己。

+   **[无标签]**

    改进了 SQL 函数调用，包括一个“engine”关键字参数，以便它们可以独立执行()或标量()，还向 SQLEngine 添加了 func 访问器

+   **[无标签]**

    修复了 MySQL4 自定义表引擎，即 TYPE 而不是 ENGINE

+   **[无标签]**

    稍微增强了日志记录，包括时间戳和一种可配置的格式化系统，而不是完整的日志记录系统

+   **[无标签]**

    对来自 TG 团队的 ActiveMapper 类进行了改进，包括多对多关系

+   **[无标签]**

    添加了 Double 和 TinyInt 支持到 mysql

## 0.1.3

发布日期：Thu Mar 02 2006

+   **[无标签]**

    完成了“post_update”功能，将在插入和删除之前添加第二个更新语句，以便在没有创建任何依赖关系的情况下协调关系；在持久化两行相互依赖时使用

+   **[无标签]**

    完成了 mapper.using(session)函数，针对每个对象的 Session 功能进行本地化；对象可以声明并作为本地用户定义的 Session 进行操作

+   **[无标签]**

    修复了 Oracle 带有多个表的“row_number over”子句

+   **[无标签]**

    如果 mapper 的表是一个连接，比如在继承关系中，mapper.get() 将不会选择多键对象，这个问题已经修复。

+   **[无标签]**

    对 sql/schema 包进行了全面改造，使得 sql 包可以独立运行，生成 select、insert 等，而不依赖于任何引擎。基于新的 TableClause/ColumnClause 词法对象。Schema 的 Table/Column 对象是它们的 “物理” 子类。简化了 schema/sql 关系、扩展（如 proxyengine）并大幅提高了整体性能。移除了困扰 0.1.1 版本的整个 getattr() 行为。

+   **[无标签]**

    重新设计了 mapper 在两个对象之间 “同步” 数据的方式，更好地处理附加到具有额外继承关系的 mapper 的属性，同时 mapper 用于在继承和被继承的 mapper 之间同步的方法也用于在继承和被继承的 mapper 之间同步。

+   **[无标签]**

    使 objectstore 更积极地 “检查超出 identitymap” ，当修改对象属性或删除对象时执行检查

+   **[无标签]**

    Index 对象完全实现，可以独立构建，也可以通过 Columns 上的 “index” 和 “unique” 参数构建。

+   **[无标签]**

    向 SQLEngine 添加了 “convert_unicode” 标志，将所有 String/CHAR 类型视为 Unicode 类型，在绑定参数和结果集方面进行原始字节/utf-8 转换。

+   **[无标签]**

    postgres 维护一个 ANSI 函数列表，这些函数必须没有括号，以便无参数的函数调用能够一致地工作

+   **[无标签]**

    可以创建没有指定引擎的表。这将默认将它们的引擎设置为一个模块范围的 “默认引擎”，这是一个 ProxyEngine。可以通过函数 “global_connect” 连接这个引擎。

+   **[无标签]**

    向 objectstore / Session 添加了“refresh(*obj)”方法，可以无条件地重新加载数据库中任意一组对象的属性

## 0.1.2

发布日期：2006 年 2 月 24 日 星期五

+   **[无标签]**

    修复了模式中的递归调用，该调用以某种方式运行了 994 次，然后正常返回。没有破坏任何东西，但减慢了一切。感谢 jpellerin 发现这个问题。

## 0.1.1

发布日期：2006 年 2 月 23 日 星期四

+   **[无标签]**

    对 Function 类进行了小修复，以便具有 func.foo() 的表达式使用 Function 对象的类型（即左侧）作为布尔表达式的类型，而不是另一侧，后者更像是一个移动目标（变更集 1020）。

+   **[无标签]**

    创建具有反向引用 backrefs 的自引用映射稍微更容易了（但仍然不是那么容易 - 变更集 1019）

+   **[无标签]**

    修复了一对一映射的问题（变更集 1015）

+   **[无标签]**

    修复了 psycopg1 中与 None 相关的日期/时间问题（变更集 1005）

+   **[无标签]**

    与 postgres 相关的两个问题，因为 oids 已经被弃用，所以它不想给你 “lastrowid”：

    > +   postgres 数据库端默认值在主键列上是明确执行的，尽管这并不是被动默认值的概念。这是因为列上的序列被反映为被动默认值，但需要在主键列上明确执行，这样我们才知道刚刚插入的是什么。
    > +   
    > +   如果您添加了一行，并在其上有一堆数据库端的默认值，并且被动默认值的事情是以旧方式工作的，即它们只在数据库端执行，那么发生的“没有 OID 就无法获取行”的异常也不会发生，除非有人（通常是 ORM）明确要求它。

+   **[无标签]**

    修复了 engine.execute_compiled 的一个小问题，它会生成一个只被丢弃的第二个 ResultProxy。

+   **[无标签]**

    开始在对象属性中实现更新逻辑。现在你可以说 myclass.attr.property，这将给你对应于该属性的 PropertyLoader，即 myclass.mapper.props[‘attr’]

+   **[无标签]**

    急切加载已在内部进行了改进，始终使用别名。现在可以创建更复杂的急切加载链，而不需要任何显式的“使用别名”类型的指令。EagerLoader 代码现在也简单得多。

+   **[无标签]**

    关系中添加了一个新的有些试验性的标志“use_update”，表示这个关系应该由第二个 UPDATE 语句处理，要么在主要的 INSERT 之后，要么在主要的 DELETE 之前。处理循环行依赖关系。

+   **[无标签]**

    添加了 exceptions 模块，所有引发的异常（除了一些 KeyError/AttributeError 异常）都是这些类的子类。

+   **[无标签]**

    修复了 MySQL 中日期类型的问题，返回的 timedelta 转换为 datetime.time

+   **[无标签]**

    两阶段 objectstore.commit 操作（即开始/提交）现在返回一个事务性对象（SessionTrans），以更清楚地指示事务边界。

+   **[无标签]**

    Index 对象添加了对 schema 的创建/删除支持

+   **[无标签]**

    修复了 postgres 的问题，在主键列上如果它是一个被动默认值，它将显式预先执行，符合持续存在的“我们无法从 postgres 获取插入的行”的问题

+   **[无标签]**

    更改了获取 postgres 表定义的 information_schema 查询，现在使用显式的 JOIN 关键字，因为有一个用户在 8.1 上表现更快的性能

+   **[无标签]**

    修复了 engine.process_defaults，使其在具有不同列名/列键的表上正确工作（更改集 982）

+   **[无标签]**

    一列只能附加到一个表上 - 这现在被断言了

+   **[无标签]**

    postgres 时间类型源自 Time 类型

+   **[无标签]**

    修复了所有测试，以便运行类型测试（现在命名为 testtypes）

+   **[无标签]**

    修复了 Join 对象，以便正确导出其外键（cs 973）

+   **[无标签]**

    创建与使用继承的映射器的关系已修复（cs 973）

## 0.1.7

发布日期：2006 年 5 月 5 日星期五

+   **[无标签]**

    对拓扑排序算法进行了一些修复

+   **[无标签]**

    添加了对 Postgres 的 DISTINCT ON 支持（只需提供 distinct=[col1,col2..]）

+   **[无标签]**

    添加了 __mod__（% 运算符）到 SQL 表达式

+   **[无标签]**

    从继承的映射器继承的“order_by”映射器属性

+   **[无标签]**

    修复了映射器 UPDATE/DELETE 时使用的列类型

+   **[无标签]**

    使用 convert_unicode=True，反射失败，已修复

+   **[无标签]**

    类型类型类型！仍然无法工作….必须再次使用 TypeDecorator :(

+   **[无标签]**

    mysql 二进制类型将数组输出转换为缓冲区，修复了 PickleType

+   **[无标签]**

    最终解决了 attributes.py 的内存泄漏问题

+   **[无标签]**

    单元测试根据支持每个数据库的数据库进行了限定

+   **[无标签]**

    修复了列默认值会破坏插入对象的 VALUES 子句的错误

+   **[无标签]**

    修复了表定义中带有模式名称会强制引擎连接的错误

+   **[无标签]**

    修复了子查询中括号与 INSERT/UPDATE 正确工作的问题

+   **[无标签]**

    HistoryArraySet 获取了 extend() 方法

+   **[无标签]**

    修复了 lazyload 对其他比较运算符的支持，而不仅仅是 =

+   **[无标签]**

    修复了 lazyload 中两个比较在连接条件中指向相同列的问题

+   **[无标签]**

    添加了“construct_new”标志到映射器，将使用 __new__ 创建实例而不是 __init__（在 0.2 中是标准）

+   **[无标签]**

    添加了 selectresults.py 到 SVN，上次遗漏了

+   **[无标签]**

    调整允许通过关联表从一个表到自身的多对多关系

+   **[无标签]**

    用于多态示例的“translate_row”函数的小修复

+   **[无标签]**

    create_engine 使用 cgi.parse_qsl 读取查询字符串（在 0.2 中已废弃）

+   **[无标签]**

    对 CAST 运算符进行了调整

+   **[无标签]**

    修复了函数名称 LOCAL_TIME/LOCAL_TIMESTAMP -> LOCALTIME/LOCALTIMESTAMP

+   **[无标签]**

    修复了编译中 ORDER BY/HAVING 的顺序

## 0.1.6

发布日期：2006 年 4 月 12 日星期三

+   **[无标签]**

    添加了对 MS-SQL 的支持，感谢 Rick Morrison、Runar Petursson

+   **[无标签]**

    来自 J. Ellis 的最新 SQLSoup

+   **[无标签]**

    ActiveMapper 对继承有初步支持（Jeff Watkins）

+   **[无标签]**

    添加了一个“mods”系统，允许修改/增强核心功能的可插拔模块，使用函数“install_mods(*modnames)”。

+   **[无标签]**

    添加了第一个“mod”，SelectResults，修改了 mapper selects 以返回将范围转换为 LIMIT/OFFSET 查询的生成器（Jonas Borgstr?）

+   **[无标签]**

    将 Mapper 的查询功能分离为一个独立的 Query 对象，该对象是 Session-centric 的。这提高了 mapper.using(session) 的性能，并使其他功能成为可能。

+   **[无标签]**

    objectstore/Session 重构，现在保存对象的官方方式是通过 flush() 方法。Session 的 begin/commit 功能被分解为 LegacySession，仍然作为默认行为，直到 0.2 系列。

+   **[无标签]**

    类型系统在查询编译时绑定到引擎，而不是模���构建时。这简化了类型系统以及 ProxyEngine。

+   **[无标签]**

    向 mapper 添加了 ‘version_id’ 关键字参数。此关键字应引用一个具有整数类型的 Column 对象，最好是非空的，该对象将用于在映射表上跟踪版本号。此数字在每次保存操作时递增，并在 UPDATE/DELETE 条件中指定，以便将其纳入返回的行数计数中，如果接收到的值不是预期的计数，则会导致 ConcurrencyError。

+   **[无标签]**

    向 mapper 添加了 ‘entity_name’ 关键字参数。现在，通过类对象以及可选的 entity_name 参数（默认为 None）将 mapper 与类关联起来。可以为一个类创建任意数量的主要 mapper，由实体名称限定。这些类的实例将通过其实体名称限定的 mapper 发出所有的加载和保存操作，并在标识映射中维护一个对于另一个等效对象的独立标识。

+   **[无标签]**

    对属性系统进行了全面改进。代码已经澄清，并且还修复了对对象属性上正确多态行为的支持。

+   **[无标签]**

    向 Select 对象添加了 “for_update” 标志

+   **[无标签]**

    一些用于反向引用的修复

+   **[无标签]**

    修复了 postgres1 DateTime 类型

+   **[无标签]**

    文档页面大部分已切换到 Markdown 语法

## 0.1.5

发布日期：2006 年 3 月 27 日 星���一

+   **[无标签]**

    向 SQLEngine 添加了 SQLSession 概念。此对象跟踪从连接池中检索连接以及正在进行的事务。向 SQLEngine 添加了 push_session() 和 pop_session() 方法，这些方法将新的 SQLSession 推送/弹出到引擎上，允许在前一个连接“嵌套”内操作第二个连接，从而允许嵌套事务。关于 SQLSession 的其他技巧肯定会在以后出现。

+   **[无标签]**

    向 objectstore.Session 添加了 nest_on 参数。这是一个单个 SQLEngine 或引擎列表，每当此 Session 成为活动会话时（通过 objectstore.push_session() 或等效方式），都将调用 push_session()/pop_session()。这允许一个工作单元 Session 利用嵌套事务功能，而无需在引擎上显式调用 push_session/pop_session。

+   **[无标签]**

    将 objectstore/unitofwork 分解为单独的“Session 作用域”和“uow 提交重要工作”

+   **[无标签]**

    向 MapperExtension 添加了 populate_instance() 方法。允许扩展修改对象属性的填充。此方法可以调用另一个 mapper 上的 populate_instance() 方法，以将属性填充从一个 mapper 代理到另一个 mapper；还内置了一些行翻译逻辑来帮助实现这一点。

+   **[无标签]**

    修复了 Oracle8 兼容性的“use_ansi”标志，将 JOIN 转换为使用 = 和 (+) 运算符进行比较，通过了基本的单元测试

+   **[无标签]**

    调整了 Oracle LIMIT/OFFSET 支持

+   **[无标签]**

    Oracle 反射使用 ALL_** 视图而不是 USER_** 来获取更多要反射的内容列表

+   **[无标签]**

    修复了 Oracle 外键反射问题

    参考：[#105](https://www.sqlalchemy.org/trac/ticket/105)

+   **[无标签]**

    objectstore.commit(obj1, obj2,…)增加了一个额外步骤，寻找属性上的私有关系并删除子对象，即使它不是全局提交

+   **[无标签]**

    对使用继承的映射器进行了大量修复，加强了映射器上关系指向“本地”表的概念，而不是它继承的表。允许更复杂的组合模式与延迟/即时加载一起使用。

+   **[无标签]**

    添加了支持映射器从其他映射器继承的功能，只需指定与父/子映射器相同的表。

+   **[无标签]**

    关于实例化和填充新对象的属性系统进行了一些轻微的速度改进。

+   **[无标签]**

    修复了 MySQL 二进制单元测试

+   **[无标签]**

    INSERTs 可以接收子句元素作为 VALUES 参数，而不仅仅是文字值

+   **[无标签]**

    支持调用多令牌函数，即 schema.mypkg.func()

+   **[无标签]**

    将 J. Ellis 的 SQLSoup 模块添加到扩展包中

+   **[无标签]**

    添加了展示从一个映射器加载多个对象类型的“多态”示例，第二个示例使用了新的 populate_instance()方法。对映射器进行了小的改进，使用 UNION 构造来帮助示例。

+   **[无标签]**

    对 session.refresh()/session.expire()进行了改进/修复（之前可能称为“invalidate”..）

+   **[无标签]**

    添加了 session.expunge()，完全从当前会话中删除对象

+   **[无标签]**

    在 engine.transaction(func)中添加了*args, **kwargs 传递，允许更容易地创建事务化装饰器函数

+   **[无标签]**

    对 ResultProxy 添加了迭代器接口：“for row in result:…”

+   **[无标签]**

    在 tx = session.begin(); tx.rollback(); tx.begin()中添加了断言，即在回滚后不能再使用它()

+   **[无标签]**

    在 SQLite 上添加了绑定参数修复日期转换���使日期能够与 pysqlite1 一起使用

+   **[无标签]**

    对子查询进行了改进，更智能地构造它们的 FROM 子句

    参考：[#116](https://www.sqlalchemy.org/trac/ticket/116)

+   **[无标签]**

    在 types 中添加了 PickleType。

+   **[无标签]**

    修复了与绑定参数相关的列标签两个错误：绑定参数键名现在在所有相关情况下都从列“标签”生成，以利用过长名称规则，并检查是否与列名为“tablename_colname”的列发生奇怪的冲突

+   **[无标签]**

    对工作单元文档和其他文档部分进行了重大改进。

+   **[无标签]**

    修复了属性 bug，即如果对象已提交，则其延迟加载的列表如果尚未加载则会被清除

+   **[无标签]**

    在 engine 中添加了 unique_connection()方法，连接池返回一个不属于线程本地上下文或任何当前事务的连接

+   **[无标签]**

    在池化连接中添加了 invalidate()函数。将从池中移除连接。但仍需工作以使引擎自动重新连接到陈旧的数据库。

+   **[无标签]**

    在列元素中添加了 distinct()函数，以便可以执行 func.count(mycol.distinct())。

+   **[无标签]**

    在 Mapper 中添加了“always_refresh”标志，创建一个始终刷新从数据库获取/选择的对象属性的映射器，覆盖任何已做出的更改。

## 版本 0.1.4

发布日期：2006 年 3 月 13 日

+   **[无标签]**

    create_engine()现在使用通用化参数；主机/主机名、数据库/dbname/数据库、密码/passwd 等对所有引擎连接都有效。使引擎 URI 更加“通用”。

+   **[无标签]**

    添加了对嵌入到列子句中的 SELECT 语句的支持，使用标志“scalar=True”。

+   **[无标签]**

    在与继承的映射器一起使用时，对 EagerLoading 进行了另一次全面改进；改进了急加载正确确定其别名查询的能力，还有针对与具有继承映射器的关系设置的表的连接，将创建针对映射器本身特定表的连接（即不会针对任何继承/进一步下降继承链的表），这可以通过使用自定义主/次要连接来覆盖。

+   **[无标签]**

    添加了 J.Ellis 对 mapper.py 的补丁，使 selectone()在查询返回多个对象行时抛出异常，selectfirst()不抛出异常。还添加了 selectfirst_by（与 get_by 同义）和 selectone_by。

+   **[无标签]**

    在 Column 中添加了 onupdate 参数，将在更新语句时执行 SQL/python。还为所有 DefaultGenerator 子类添加了“for_update=True”。

+   **[无标签]**

    添加了对 Oracle 表反射的支持，由 Andrija Zaric 贡献；仍然需要解决一些关于复合主键/字典选择的错误。

+   **[无标签]**

    提交了一个初始的 Firebird 模块，等待测试。

+   **[无标签]**

    在 compiled.get_params()的结果中添加了 sql.ClauseParameters 字典对象，对绑定参数进行了延迟类型处理，以便更容易访问原始值。

+   **[无标签]**

    为索引、列默认值、连接池、引擎构建添加了更多文档。

+   **[无标签]**

    对类型系统的构建进行了全面改进。使用了更简单的继承模式，使得任何通用类型都可以轻松地被子类化，无需 TypeDecorator。

+   **[无标签]**

    在 SQLEngine 中添加了“convert_unicode=False”参数，将导致所有 String 类型执行 unicode 编码/解码（使 Strings 的行为类似于 Unicodes）。

+   **[无标签]**

    在引擎中添加了‘encoding=”utf8”’参数。给定的编码将用于 Unicode 类型内所有编码/解码调用以及当 convert_unicode=True 时的 Strings。

+   **[无标签]**

    改进了对 UNION 的映射支持，添加了 polymorph.py 示例以说明多类映射对 UNION 的映射。

+   **[无标签]**

    修复了 SQLite LIMIT/OFFSET 语法问题

+   **[无标签]**

    修复了 Oracle LIMIT 语法问题

+   **[无标签]**

    添加了 backref()函数，允许反向引用具有将传递给 backref 的关键字参数。

+   **[无标签]**

    Sequences 和 ColumnDefault 对象可以独立执行/标量执行

+   **[无标签]**

    SQL 函数（即 func.foo()）可以独立执行/标量执行

+   **[无标签]**

    修复了 SQL 函数，使得符合 ANSI 标准的函数，即 current_timestamp 等，不指定括号。所有其他函数都会指定括号。

+   **[无标签]**

    向 SmartProperty 添加了 settattr_clean 和 append_clean，可以设置属性而不触发“脏”事件或任何历史记录。用法如：myclass.prop1.setattr_clean(myobject, ‘hi’)

+   **[无标签]**

    当 mappers 使用列默认值时提供了改进支持；mappers 将从语句的执行绑定参数（预转换）中提取预执行的默认值，以填充到保存对象的属性中；如果任何 PassiveDefaults 已经触发，将从数据库中后获取行以填充对象。

+   **[无标签]**

    在 objectstore 中添加了‘get_session().invalidate(*obj)’方法，实例将在下一次属性访问时自动刷新。

+   **[无标签]**

    对 SQL 函数的支持进行了改进，包括一个“engine”关键字参数，以便它们可以独立执行或标量执行，还向 SQLEngine 添加了 func 访问器

+   **[无标签]**

    修复了 MySQL4 自定义表引擎，即 TYPE 而不是 ENGINE

+   **[无标签]**

    稍微增强了日志记录，包括时间戳和一种可配置的格式系统，而不是完整的日志记录系统

+   **[无标签]**

    对 TG 团队的 ActiveMapper 类进行了改进，包括多对多关系

+   **[无标签]**

    向 mysql 添加了 Double 和 TinyInt 支持

## 0.1.3

发布日期：2006 年 3 月 2 日

+   **[无标签]**

    完成了“post_update”功能，将在插入和删除之前添加第二个更新语句，以便在没有创建任何依赖关系的情况下协调关系；在持久化两行相互依赖的情况下使用

+   **[无标签]**

    完成了 mapper.using(session)函数，针对每个对象的 Session 功能进行本地化；对象可以声明并操作为任何用户定义的 Session 的本地对象

+   **[无标签]**

    修复了 Oracle 中带有多个表的“row_number over”子句

+   **[无标签]**

    如果 mapper 的表是一个连接，例如在继承关系中，mapper.get()不会选择多键对象，这个问题已经修复。

+   **[无标签]**

    对 sql/schema 包进行了全面改进，使得 sql 包可以独立运行，生成 selects、inserts 等，而不依赖于任何引擎。基于新的 TableClause/ColumnClause 词法对象。Schema 的 Table/Column 对象是它们的“物理”子类。简化了 schema/sql 关系，扩展（如 proxyengine），并通过大幅提高整体性能。删除了困扰 0.1.1 版本的整个 getattr()行为。

+   **[无标签]**

    重构了映射器如何在两个对象之间“同步”数据的方式，将其放入一个单独的模块中，与附加到具有与相关表之一的附加继承关系的映射器的属性一起使用效果更好，现在映射器用于在继承和被继承的映射器之间同步的相同方法也用于同步父/子对象。

+   **[无标签]**

    使 objectstore 的“检查是否超出 identitymap”更加积极，当修改对象属性或删除对象时将执行检查

+   **[无标签]**

    索引对象已完全实现，可以独立构建，也可以通过列上的“index”和“unique”参数构建。

+   **[无标签]**

    向 SQLEngine 添加了“convert_unicode”标志，将所有 String/CHAR 类型视为 Unicode 类型，在绑定参数和结果集方面进行原始字节/utf-8 转换。

+   **[无标签]**

    postgres 维护一个 ANSI 函数列表，这些函数不能带括号，因此没有参数的函数调用可以一致工作

+   **[无标签]**

    可以创建没有指定引擎的表。这将默认将它们的引擎设置为模块范围的“默认引擎”，这是一个 ProxyEngine。可以通过函数“global_connect”连接此引擎。

+   **[无标签]**

    向 objectstore / Session 添加了“refresh(*obj)”方法，从数据库无条件重新加载任意一组对象的属性

## 0.1.2

发布日期：2006 年 2 月 24 日 星期五

+   **[无标签]**

    修复了模式中的递归调用，某种方式运行了 994 次然后正常返回。没有破坏任何东西，但减慢了一切。感谢 jpellerin 发现这个问题。

## 0.1.1

发布日期：2006 年 2 月 23 日 星期四

+   **[无标签]**

    对 Function 类进行了小修复，以便具有 func.foo() 的表达式使用 Function 对象的类型（即左侧）作为布尔表达式的类型，而不是另一侧，这更像是一个移动目标（变更集 1020）。

+   **[无标签]**

    使用 backrefs 创建自引用映射器稍微更容易了（但仍然不是那么容易 - 变更集 1019）

+   **[无标签]**

    修复了一对一映射的问题（变更集 1015）

+   **[无标签]**

    修复了 psycopg1 中 None 的日期/时间问题（变更集 1005）

+   **[无标签]**

    与 postgres 相关的两个问题，因为 oids 已被弃用，所以它不想给你“lastrowid”：

    > +   在主键列上的 postgres 数据库端默认值 *确实* 在之前显式执行，尽管这不是 PassiveDefault 的想法。这是因为列上的序列被反映为 PassiveDefaults，但需要在主键列上显式执行，以便我们知道刚刚插入了什么。
    > +   
    > +   如果您添加了一行，并且该行具有许多数据库端默认值，并且 PassiveDefault 的工作方式是旧的，即它们只在数据库端执行，那么也不会发生“没有 OID 无法获取行”的异常，除非有人（通常是 ORM）明确要求它。

+   **[无标签]**

    修复了 engine.execute_compiled 中的一个小问题，它创建了一个第二个 ResultProxy，然后被丢弃。

+   **[无标签]**

    开始在对象属性中实现更新逻辑。现在可以说 myclass.attr.property，这将给您对应于该属性的 PropertyLoader，即 myclass.mapper.props['attr']

+   **[无标签]**

    急切加载已在内部进行了彻底改进，始终使用别名。现在可以创建更复杂的急切加载链，而无需任何明确的“使用别名”类型的指令。EagerLoader 代码现在也简单得多。

+   **[无标签]**

    为关系添加了一个新的有点实验性的标志“use_update”，表示这个关系应该由第二个 UPDATE 语句处理，要么在主要 INSERT 之后，要么在主要 DELETE 之前。处理循环行依赖关系。

+   **[无标签]**

    添加 exceptions 模块，所有引发的异常（除了一些 KeyError/AttributeError 异常）都继承自这些类。

+   **[无标签]**

    修复与 MySQL 的日期类型，返回的 timedelta 转换为 datetime.time

+   **[无标签]**

    两阶段 objectstore.commit 操作（即 begin/commit）现在返回一个事务性对象（SessionTrans），以更清晰地指示事务边界。

+   **[无标签]**

    具有创建/删除支持的 Index 对象已添加到 schema

+   **[无标签]**

    修复 postgres，在主键列上明确预执行 PassiveDefault，根据持续存在的“我们无法从 postgres 获取插入的行”问题

+   **[无标签]**

    更改 information_schema 查询，获取 postgres 表定义，现在使用显式的 JOIN 关键字，因为一个用户在 8.1 上有更快的性能

+   **[无标签]**

    修复 engine.process_defaults，使其在具有不同列名/列键的表上正确工作（changeset 982）

+   **[无标签]**

    一个列只能附加到一个表上 - 现在已经断言了这一点

+   **[无标签]**

    postgres 时间类型继承自 Time 类型

+   **[无标签]**

    修复 alltests，使其运行 types 测试（现在命名为 testtypes）

+   **[无标签]**

    修复 Join 对象，使其正确导出其外键（cs 973）

+   **[无标签]**

    创建与使用继承固定的映射器关系（cs 973）
