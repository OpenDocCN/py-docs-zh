# 2.0 更新日志

> 原文：[`docs.sqlalchemy.org/en/20/changelog/changelog_20.html`](https://docs.sqlalchemy.org/en/20/changelog/changelog_20.html)

## 2.0.30

无发布日期

### orm

+   **[orm] [bug]**

    添加了新的属性 `ORMExecuteState.is_from_statement`，用于检测形式为 `select().from_statement()` 的语句，并且还增强了`FromStatement`以设置 `ORMExecuteState.is_select`、`ORMExecuteState.is_insert`、`ORMExecuteState.is_update` 和 `ORMExecuteState.is_delete` 根据发送到 `Select.from_statement()` 方法本身的元素。

    References: [#11220](https://www.sqlalchemy.org/trac/ticket/11220)

### engine

+   **[engine] [bug]**

    在 `Connection.execution_options.logging_token` 选项中修复了问题，当在已经记录了消息的连接上更改`logging_token`的值时，不会更新以反映新的日志令牌。具体来说，这会阻止使用 `Session.connection()` 来更改连接上的选项，因为 BEGIN 记录消息已经被发出。

    References: [#11210](https://www.sqlalchemy.org/trac/ticket/11210)

### 打字

+   **[typing] [bug] [regression]**

    修复了由版本 2.0.29 中 PR [#11055](https://www.sqlalchemy.org/trac/ticket/11055) 引起的打字退化，该版本试图将`ParamSpec`添加到 asyncio 的`run_sync()`方法中，使用 `AsyncConnection.run_sync()` 与 `MetaData.reflect()` 将会由于错误导致 mypy 失败。详细信息请参阅 [`github.com/python/mypy/issues/17093`](https://github.com/python/mypy/issues/17093)。由 Francisco R. Del Roio 提供的拉取请求。

    References: [#11200](https://www.sqlalchemy.org/trac/ticket/11200)

### 杂项

+   **[bug] [test]**

    确保在测试中使用`subprocess.run`时正确初始化`PYTHONPATH`变量。

    References: [#11268](https://www.sqlalchemy.org/trac/ticket/11268)

## 2.0.29

发布日期：2024 年 3 月 23 日

### orm

+   **[orm] [usecase]**

    增加了对[**PEP 695**](https://peps.python.org/pep-0695/) `TypeAliasType`构造以及 python 3.12 原生的`type`关键字的支持，以便在使用这些构造链接到[**PEP 593**](https://peps.python.org/pep-0593/) `Annotated`容器时，允许解析`Annotated`在这些构造用于`Mapped`类型容器时继续进行。

    参考：[#11130](https://www.sqlalchemy.org/trac/ticket/11130)

+   **[orm] [bug]**

    修复了声明性问题，其中使用`Relationship`而不是`Mapped`来定义关系会意外地为该属性引入“动态”关系加载策略。

    参考：[#10611](https://www.sqlalchemy.org/trac/ticket/10611)

+   **[orm] [bug]**

    修复了在 ORM 注释声明中使用`mapped_column()`与`mapped_column.index`或`mapped_column.unique`设置为 False 时，会被具有该参数设置为`True`的传入`Annotated`元素覆盖的问题，即使直接的`mapped_column()`元素更具体且应优先考虑。增强了协调布尔值的逻辑，以适应本地值为`False`仍然优先于来自注释元素的`True`值的情况。

    参考：[#11091](https://www.sqlalchemy.org/trac/ticket/11091)

+   **[orm] [bug] [regression]**

    修复了从版本 2.0.28 引起的回归，该回归是由于修复[#11085](https://www.sqlalchemy.org/trac/ticket/11085)而引起的，其中调整后缓存绑定参数值的新方法会干扰`subqueryload()`加载器选项的实现，该加载器选项在内部使用一些更具传统模式的模式，当使用此加载器选项与此加载器选项一起使用附加加载器条件功能时。

    参考：[#11173](https://www.sqlalchemy.org/trac/ticket/11173)

### 引擎

+   **[engine] [bug]**

    修复了 “插入多个值”行为对 INSERT 语句的行为 功能中的问题，其中使用带有“内联执行”默认生成器的主键列，例如具有显式 `Sequence` 并具有显式模式名称的生成器，同时使用 `Connection.execution_options.schema_translate_map` 功能将无法正确呈现序列或参数，导致错误。

    参考: [#11157](https://www.sqlalchemy.org/trac/ticket/11157)

+   **[engine] [bug]**

    对版本 2.0.10 中对 [#9618](https://www.sqlalchemy.org/trac/ticket/9618) 所做的调整进行了更改，该调整增加了从批量 INSERT 中协调 RETURNING 行到传递给它的参数的行为。该行为包括已经 DB 转换的绑定参数值与返回的行值之间的比较，并不总是对于 SQL 列类型（如 UUID）是“对称”的，具体取决于不同的 DBAPI 如何接收这些值以及它们如何返回它们，因此需要在这些列类型上添加额外的“标志值解析器”方法。不幸的是，这破坏了第三方列类型，如 SQLModel 中未实现此特殊方法的 UUID/GUID 类型，引发错误“无法将结果集中的标志值与参数集匹配”。与其试图进一步解释和文档化“insertmanyvalues”功能的此实现细节，包括新方法的公共版本，不如将方法调整为不再需要此额外的转换步骤，并且执行比较的逻辑现在在预转换的绑定参数值与后处理结果值之间进行，后者应始终具有匹配的数据类型。在不寻常的情况下，如果自定义 SQL 列类型同时也用作批量 INSERT 的“标志”列不接收和返回相同类型的值，则将引发“无法匹配”错误，但缓解方法很简单，即应传递与返回值相同的 Python 数据类型。

    参考: [#11160](https://www.sqlalchemy.org/trac/ticket/11160)

### sql

+   **[sql] [bug] [regression]**

    从 1.4 系列的回归中修复了重构`TypeEngine.with_variant()`方法的问题，该问题在“with_variant()”克隆原始 TypeEngine 而不是更改类型中介绍。该问题未考虑到`.copy()`方法，该方法会丢失设置的变体映射。对于“schema”类型的非常特定情况而言，这是一个问题，该类型包括`Enum`和`ARRAY`等类型，当它们在 ORM Declarative 映射与混入一起使用时，类型的复制就会起作用。现在还复制了变体映射。

    参考：[#11176](https://www.sqlalchemy.org/trac/ticket/11176)

### typing

+   **[typing] [错误]**

    修复了允许 asyncio `run_sync()`方法正确类型化参数的类型问题，该方法根据传递的可调用函数使用了[**PEP 612**](https://peps.python.org/pep-0612/) `ParamSpec`变量。感谢 Francisco R. Del Roio 提供的拉取请求。

    参考：[#11055](https://www.sqlalchemy.org/trac/ticket/11055)

### postgresql

+   **[postgresql] [用例]**

    PostgreSQL 方言现在在反射具有域作为类型的列时返回`DOMAIN`实例。之前，返回的是域数据类型。作为此更改的一部分，改进了域反射以返回文本类型的校对。感谢 Thomas Stephenson 提供的拉取请求。

    参考：[#10693](https://www.sqlalchemy.org/trac/ticket/10693)

### 测试

+   **[测试] [错误]**

    将对与 asyncio 相关的测试运行方式进行了改进，并将其后移至 SQLAlchemy 2.0，现在使用较新的 Python 3.11 `asyncio.Runner`或后移的等效版本，而不是依赖于以前基于`asyncio.get_running_loop()`的实现。这样做有望防止在 CPU 负载硬件上进行大型测试套件运行时出现问题，其中事件循环似乎会损坏，从而导致级联失败。

    参考：[#11187](https://www.sqlalchemy.org/trac/ticket/11187)

## 2.0.28

发布日期：2024 年 3 月 4 日

### orm

+   **[orm] [性能] [错误] [回归]**

    调整了在[#10570](https://www.sqlalchemy.org/trac/ticket/10570)中进行的修复，发布在 2.0.23 中，其中添加了新的逻辑来协调可能在`with_expression()`构造中用于缓存键生成的绑定参数值的更改。新的逻辑改变了将新的绑定参数值与语句关联的方法，避免了需要深复制语句的需要，这可能会对非常深/复杂的 SQL 结构造成重大性能损失。新方法不再需要这个深复制步骤。

    参考：[#11085](https://www.sqlalchemy.org/trac/ticket/11085)

+   **[orm] [错误] [回归]**

    修复了由[#9779](https://www.sqlalchemy.org/trac/ticket/9779)引起的回归，其中在关系`and_()`表达式中使用“secondary”表将无法被别名化以匹配“secondary”表在`Select.join()`表达式中通常的渲染方式，导致查询无效。

    参考：[#11010](https://www.sqlalchemy.org/trac/ticket/11010)

### 引擎

+   **[引擎] [用例]**

    添加了新的核心执行选项`Connection.execution_options.preserve_rowcount`。设置后，将在语句执行时无条件地将 DBAPI 游标的`cursor.rowcount`属性存储，以便无论 DBAPI 为任何类型的语句提供的值都可以使用`CursorResult.rowcount`属性从`CursorResult`中获取。这允许访问行计数，例如 INSERT 和 SELECT 语句，至少在使用的 DBAPI 支持的程度上。INSERT 语句的“插入多个值”行为也支持此选项，并将在设置时确保为批量插入行时正确设置`CursorResult.rowcount`。

    参考：[#10974](https://www.sqlalchemy.org/trac/ticket/10974)

### asyncio

+   **[asyncio] [错误]**

    如果将 `QueuePool` 或其他非异步池类传递给 `create_async_engine()`，则会引发错误。此引擎仅接受与 asyncio 兼容的池类，包括 `AsyncAdaptedQueuePool`。其他池类（例如 `NullPool`）与同步和异步引擎都兼容，因为它们不执行任何锁定。

    参见

    API 文档 - 可用的连接池实现

    参考资料：[#8771](https://www.sqlalchemy.org/trac/ticket/8771)

### 测试

+   **[tests] [change]**

    tox.ini 文件中的 pytest 支持已更新，以支持 pytest 8.1。

## 2.0.27

发布日期：2024 年 2 月 13 日

### postgresql

+   **[postgresql] [bug] [regression]**

    由于刚发布的修复了[#10863](https://www.sqlalchemy.org/trac/ticket/10863)的修复导致的回归已经修复，其中将一个无效的异常类添加到了“except”块中，除非实际发生这样的捕获，否则不会被执行。已经添加了一种模拟式测试，以确保在单元测试中执行此捕获。

    参考资料：[#11005](https://www.sqlalchemy.org/trac/ticket/11005)

## 2.0.26

发布日期：2024 年 2 月 11 日

### orm

+   **[orm] [bug]**

    已用一个较短的消息替换了“加载器深度过深”的警告，该消息被添加到 SQL 日志中的缓存徽章中，用于 ORM 由于加载器选项的过深链而禁用缓存的语句。此警告突出显示的条件难以解决，并且通常只是 ORM 在应用 SQL 缓存时的限制。未来的功能可能包括调整禁用缓存的阈值的能力，但目前此警告将不再是一个麻烦。

    参考资料：[#10896](https://www.sqlalchemy.org/trac/ticket/10896)

+   **[orm] [bug]**

    修复了在类主体内部声明类型（如枚举）时无法在`Mapped`容器类型中使用该类型的问题。现在，用于评估的本地变量范围包括类主体本身。此外，`Mapped`内的表达式还可以引用类名本身，如果作为字符串使用或者使用了未来的注释模式。

    参考资料：[#10899](https://www.sqlalchemy.org/trac/ticket/10899)

+   **[orm] [bug]**

    修复了使用 `Session.delete()` 与 `Mapper.version_id_col` 功能时，如果由于对象上使用了 `relationship.post_update` 而导致目标对象上发出了额外的 UPDATE，则会失败使用正确的版本标识符的问题。这个问题类似于[#10800](https://www.sqlalchemy.org/trac/ticket/10800)，只是对于仅有更新的情况，版本 2.0.25 中刚刚修复了。

    参考：[#10967](https://www.sqlalchemy.org/trac/ticket/10967)

+   **[orm] [错误]**

    修复了 `with_expression()` 实现中的断言，如果使用了不可缓存的 SQL 表达式，则会引发断言错误；这是从 1.4 版本以来的一个 2.0 回归。

    参考：[#10990](https://www.sqlalchemy.org/trac/ticket/10990)

### 示例

+   **[示例] [错误]**

    修复了 history_meta 示例中的回归，其中使用 `MetaData.to_metadata()` 复制历史表也会复制索引（这是一件好事），但不管用于这些索引的命名方案如何，都会导致命名冲突。现在这些索引都会添加一个“_history”后缀，方式与表名的方式相同。

    参考：[#10920](https://www.sqlalchemy.org/trac/ticket/10920)

+   **[示例] [错误]**

    通过将 `Identity` 构造添加到所有表中，并允许在此后端上进行主键生成，修复了 examples/performance 中性能示例脚本在 Oracle 数据库上基本可用的问题。一些“原始 DBAPI” 情况仍与 Oracle 不兼容。

### sql

+   **[sql] [错误]**

    修复了 `case()` 中的问题，即确定表达式类型的逻辑可能导致如果“whens”中的最后一个元素没有类型，或在其他情况下类型可能解析为 `None`，则会导致 `NullType`。逻辑已更新为扫描所有给定表达式，以便使用第一个非空类型，并始终确保存在类型。感谢 David Evans 提交的拉取请求。

    参考：[#10843](https://www.sqlalchemy.org/trac/ticket/10843)

### typing

+   **[类型] [错误]**

    修复了 `PoolEvents.checkin()` 事件的类型签名，指示给定的 `DBAPIConnection` 参数在连接无效时可能为 `None` 的情况。

### postgresql

+   **[postgresql] [usecase] [reflection]**

    增加对以“NO INHERIT”标记的 PostgreSQL CHECK 约束的反射支持，将 `no_inherit=True` 设置为反射数据的键。感谢 Ellis Valentiner 的拉取请求。

    引用：[#10777](https://www.sqlalchemy.org/trac/ticket/10777)

+   **[postgresql] [usecase]**

    支持 `USING <method>` 选项用于 PostgreSQL `CREATE TABLE`，以指定用于存储新表内容的访问方法。感谢 Edgar Ramírez-Mondragón 的拉取请求。

    另请参阅

    PostgreSQL 表选项

    引用：[#10904](https://www.sqlalchemy.org/trac/ticket/10904)

+   **[postgresql] [usecase]**

    正确地将 PostgreSQL RANGE 和 MULTIRANGE 类型标记为 `Range[T]` 和 `Sequence[Range[T]]`。引入了实用程序序列 `MultiRange` ，以允许更好地支持 MULTIRANGE 类型的互操作性。

    引用：[#9736](https://www.sqlalchemy.org/trac/ticket/9736)

+   **[postgresql] [usecase]**

    当从 `Range` 或 `MultiRange` 实例推断数据库类型时，区分 INT4 和 INT8 范围以及多范围类型，如果值适合 INT4，则优先选择 INT4。

+   **[postgresql] [bug] [regression]**

    修复了在 2.0.24 版本中由 [#10717](https://www.sqlalchemy.org/trac/ticket/10717) 引起的 asyncpg 方言中的回归，该版本中现在尝试在终止之前优雅地关闭 asyncpg 连接的更改将不会对其他可能的与连接相关的异常（除了超时错误之外）回退到 `terminate()` ，没有考虑到优雅的 `.close()` 尝试因其他原因失败，如连接错误。

    引用：[#10863](https://www.sqlalchemy.org/trac/ticket/10863)

+   **[postgresql] [bug]**

    修复了在使用 PostgreSQL 方言时，`Uuid` 数据类型与 `Uuid.as_uuid` 参数设置为 False 时的问题。ORM 优化的 INSERT 语句（例如，“insertmanyvalues”功能）将不会正确地对齐主键 UUID 值以进行批量 INSERT 语句，导致错误。类似的问题也针对 pymssql 驱动程序进行了修复。

### mysql

+   **[mysql] [bug]**

    修复了一个问题，即当一个 MySQL 列同时指定了 VIRTUAL 或 STORED 指令时，NULL/NOT NULL 无法正确反映出来的问题。拉取请求由 Georg Wicke-Arndt 提供。

    参考：[#10850](https://www.sqlalchemy.org/trac/ticket/10850)

+   **[mysql] [bug]**

    修复了 asyncio 方言 asyncmy 和 aiomysql 中的问题，其中它们的 `.close()` 方法显然不是优雅关闭的。用非标准的 `.ensure_closed()` 方法替换，该方法是可等待的，并将 `.close()` 移动到所谓的“终止”情况。

    参考：[#10893](https://www.sqlalchemy.org/trac/ticket/10893)

### mssql

+   **[mssql] [bug]**

    修复了在使用 pymssql 方言时，`Uuid` 数据类型与 `Uuid.as_uuid` 参数设置为 False 时的问题。ORM 优化的 INSERT 语句（例如，“insertmanyvalues”功能）将不会正确地对齐主键 UUID 值以进行批量 INSERT 语句，导致错误。类似的问题也针对 PostgreSQL 驱动程序进行了修复。

### Oracle

+   **[oracle] [performance] [bug]**

    更改了 Oracle 方言的默认数组大小，以便使用驱动程序设置的值，即写入时的 cx_oracle 和 oracledb 的值为 100。以前默认设置为 50 的值可能会导致在较慢的网络上使用 cx_oracle/oracledb 单独提取许多行时出现显着的性能回归。

    参考：[#10877](https://www.sqlalchemy.org/trac/ticket/10877)

## 2.0.25

发布日期：2024 年 1 月 2 日

### orm

+   **[orm] [usecase]**

    添加了对 Python 3.12 pep-695 类型别名结构的初步支持，用于解析 ORM Annotated Declarative 映射的自定义类型映射时。

    参考：[#10807](https://www.sqlalchemy.org/trac/ticket/10807)

+   **[orm] [bug]**

    修复了一种情况，即在同时使用 `relationship.post_update` 功能和使用 mapper version_id_col 时，后者 UPDATE 语句可能会未能正确使用正确的版本标识符，假定在该刷新中已经发出了一个已经增加了版本计数器的 UPDATE。

    参考文献：[#10800](https://www.sqlalchemy.org/trac/ticket/10800)

+   **[orm] [错误]**

    修复了 ORM 注解式声明中的问题，如果左侧类型被指定为类而不是字符串，并且没有使用 future 风格的注释，当左侧没有指定任何集合为 uselist=True 时，会误解关系的左侧。 

    参考文献：[#10815](https://www.sqlalchemy.org/trac/ticket/10815)

### sql

+   **[sql] [错误]**

    改进了在布尔比较的否定上下文中编译 `any_()` / `all_()` 的方式，现在将呈现 `NOT (expr)` 而不是将等式操作符反转为不等于，允许对这些非典型运算符进行更精细的否定控制。

    参考文献：[#10817](https://www.sqlalchemy.org/trac/ticket/10817)

### 类型提示

+   **[类型提示] [错误]**

    修复了在版本 2.0.24 中添加到 `sqlalchemy.sql.functions` 模块的类型提示引起的回归，作为 [#6810](https://www.sqlalchemy.org/trac/ticket/6810) 的一部分：

    +   进一步增强了 pep-484 类型提示，以允许从 `sqlalchemy.sql.expression.func` 派生的元素更有效地与 ORM 映射的属性一起使用 ([#10801](https://www.sqlalchemy.org/trac/ticket/10801))

    +   修复了传递给函数的参数类型，以便像字符串和整数这样的字面表达式再次被正确解释 ([#10818](https://www.sqlalchemy.org/trac/ticket/10818))

    参考文献：[#10801](https://www.sqlalchemy.org/trac/ticket/10801), [#10818](https://www.sqlalchemy.org/trac/ticket/10818)

### asyncio

+   **[asyncio] [错误]**

    修复了 asyncio 版本连接池中的关键问题，调用 `AsyncEngine.dispose()` 会产生一个新的连接池，该连接池没有完全重新建立对 asyncio 兼容互斥锁的使用，导致在使用类似于 `asyncio.gather()` 的并发特性时，在 asyncio 上下文中产生死锁，因为它使用了普通的 `threading.Lock()`。

    这个改变也**回溯**到了：1.4.51

    参考文献：[#10813](https://www.sqlalchemy.org/trac/ticket/10813)

### oracle

+   **[oracle] [asyncio]**

    在 asyncio 模式下添加了对 python-oracledb 的支持，使用了新发布的支持 asyncio 的 `oracledb` DBAPI 版本。 对于 2.0 系列，这是一个预览版本，当前实现尚未包括对 `AsyncConnection.stream()` 的支持。 SQLAlchemy 计划改进支持以适用于 2.1 版本。

    参考文献：[#10679](https://www.sqlalchemy.org/trac/ticket/10679)

## 2.0.24

发布日期：2023 年 12 月 28 日

### orm

+   **[orm] [bug]**

    对在版本 0.9.8 中发布的[#3208](https://www.sqlalchemy.org/trac/ticket/3208)首次实施的修复进行了改进，其中声明性内部使用的类注册表可能会受到竞争条件的影响，这种情况下在同时清理个别映射类并构造新映射类时可能会发生，如一些测试套件配置或动态类创建环境中可能发生的情况。除了已经添加的弱引用检查外，还首先复制正在迭代的项目列表，以避免“在迭代时更改列表”的错误。感谢 Yilei Yang 提供的拉取请求。

    此更改也被**回溯**到：1.4.51

    参考：[#10782](https://www.sqlalchemy.org/trac/ticket/10782)

+   **[orm] [bug]**

    修复了在未对非初始化的`mapped_column()`构造上使用`foreign()`注释会产生没有类型的表达式的问题，这样在实际列初始化时不会更新，导致关系无法适当地确定`use_get`的问题。

    参考：[#10597](https://www.sqlalchemy.org/trac/ticket/10597)

+   **[orm] [bug]**

    改进了工作单元进程将主键列的值设置为 NULL 的错误消息，因为具有对该列的依赖规则的相关对象被删除，包括不仅目标对象和列名，还包括来源列。感谢 Jan Vollmer 提供的拉取请求。

    参考：[#10668](https://www.sqlalchemy.org/trac/ticket/10668)

+   **[orm] [bug]**

    修改了`MappedAsDataclass`、`DeclarativeBase`和`DeclarativeBaseNoMeta`使用的`__init_subclass__()`方法，使其接受任意的`**kw`并将其传播到`super()`调用，从而允许更灵活地安排使用`__init_subclass__()`关键字参数的自定义超类和混入。感谢 Michael Oliver 提供的拉取请求。

    参考：[#10732](https://www.sqlalchemy.org/trac/ticket/10732)

+   **[orm] [bug]**

    确保了在 ORM 启用的 INSERT、UPDATE 和 DELETE 语句的 `returning()` 部分中使用 `Bundle` 对象的用例已经得到测试并且完全可用。这在以前从未被明确实现或测试过，并且在 1.4 系列中没有正常工作；在 2.0 系列中，ORM UPDATE/DELETE 缺少了一个实现方法，导致 `Bundle` 对象无法正常工作。

    参考：[#10776](https://www.sqlalchemy.org/trac/ticket/10776)

+   **[orm] [bug]**

    修复了 2.0 版本中 `MutableList` 中的一个回归问题，该问题导致检测序列的例程无法正确地过滤掉字符串或字节实例，从而无法将字符串值分配给特定索引（而非序列值则可以正常工作）。

    参考：[#10784](https://www.sqlalchemy.org/trac/ticket/10784)

### engine

+   **[engine] [bug]**

    修复了将 `URL` 对象的用户名和密码部分进行 URL 编码时的问题，在使用 `URL.render_as_string()` 方法将其转换为字符串时，采用了 Python 标准库 `urllib.parse.quote`，同时允许加号和空格保持不变，以便与 SQLAlchemy 的非标准 URL 解析兼容，而不是多年前的遗留自行编写的例程。感谢 Xavier NUNN 提交的拉取请求。

    参考：[#10662](https://www.sqlalchemy.org/trac/ticket/10662)

### sql

+   **[sql] [bug]**

    修复了 SQL 元素的字符串化问题，在未传递特定方言的情况下，遇到诸如 PostgreSQL 的“on conflict do update”构造之类的方言特定元素，然后无法提供具有适当状态以呈现构造的字符串化方言，导致内部错误。

    参考：[#10753](https://www.sqlalchemy.org/trac/ticket/10753)

+   **[sql] [bug]**

    修复了针对 DML 构造（如 `insert()` 构造）的 `CTE` 进行字符串化或编译时失败的问题，由于错误地检测到了语句整体是一个 INSERT，导致内部错误。

### schema

+   **[schema] [bug]**

    修复了创建 `Table` 等对象时出现意外模式项的错误报告问题，该问题会错误地处理作为元组传递的参数，导致格式错误。错误消息已经使用 f-strings 进行了现代化处理。

    参考：[#10654](https://www.sqlalchemy.org/trac/ticket/10654)

### typing

+   **[typing] [bug]**

    为 `sqlalchemy.sql.functions` 模块完成了 pep-484 类型化。对于针对 `func` 元素进行的 `select()` 构造现在应该填充返回类型。

    参考：[#6810](https://www.sqlalchemy.org/trac/ticket/6810)

### asyncio

+   **[asyncio] [change]**

    `async_fallback` 方言参数现已弃用，并将在 SQLAlchemy 2.1 中删除。这个标志在一段时间内没有被用于 SQLAlchemy 的测试套件。通过使用 `greenlet_spawn()` 在 greenlet 中运行代码，asyncio 方言仍然可以以同步方式运行。

### postgresql

+   **[postgresql] [bug]**

    调整了 asyncpg 方言，以便当使用 `terminate()` 方法丢弃无效的连接时，方言首先会尝试使用带有超时的 `.close()` 优雅地关闭连接，如果操作仅在异步事件循环上下文中进行。这允许 asyncpg 驱动程序处理最终化 `TimeoutError`，包括能够在程序退出后继续运行的情况下关闭长时间运行的查询服务器端。

    参考：[#10717](https://www.sqlalchemy.org/trac/ticket/10717)

### mysql

+   **[mysql] [bug]**

    修复了使用旧于 1.0 版本的 PyMySQL 的 pool pre-ping 时，在票证 [#10492](https://www.sqlalchemy.org/trac/ticket/10492) 中的修复引入的回归。

    此更改也被**回溯**至：1.4.51

    参考：[#10650](https://www.sqlalchemy.org/trac/ticket/10650)

### 测试

+   **[tests] [bug]**

    对测试套件进行了改进，进一步加强了在未安装 Python `greenlet` 时运行的能力。现在有一个 tox 目标，其中包含标记“nogreenlet”，该目标将在未安装 greenlet 的情况下运行套件（请注意，它仍然作为 tox 配置的一部分临时安装 greenlet）。

    参考：[#10747](https://www.sqlalchemy.org/trac/ticket/10747)

## 2.0.23

发布日期：2023 年 11 月 2 日

### orm

+   **[orm] [usecase]**

    为新式批量 ORM 插入实现了 `Session.bulk_insert_mappings.render_nulls` 参数，允许 `render_nulls=True` 作为执行选项。这允许使用参数字典中的 `None` 值进行批量 ORM 插入，并使用给定的字典键的单个行批处理，而不是将其拆分为每个 INSERT 中省略 NULL 列的批次。

    另请参阅

    在 ORM 批量 INSERT 语句中发送 NULL 值](../orm/queryguide/dml.html#orm-queryguide-insert-null-params)

    参考：[#10575](https://www.sqlalchemy.org/trac/ticket/10575)

+   **[orm] [bug]**

    修复了`__allow_unmapped__`指令无法允许具有注释（如`Any`或具有特定类型但没有`Mapped[]`作为其类型的）的遗留`Column` / `deferred()`映射，而无需与定位属性名称相关的错误。

    参考：[#10516](https://www.sqlalchemy.org/trac/ticket/10516)

+   **[orm] [bug]**

    修复了缓存错误，当与加载器选项`selectinload()`、`lazyload()`一起使用`with_expression()`构造时，在后续缓存运行中无法正确替换绑定参数值的问题。

    参考：[#10570](https://www.sqlalchemy.org/trac/ticket/10570)

+   **[orm] [bug]**

    修复了 ORM 注释声明中的错误，其中使用`ClassVar`，但仍然以某种方式引用 ORM 映射类名会导致无法解释为未映射的`ClassVar`。

    参考：[#10472](https://www.sqlalchemy.org/trac/ticket/10472)

### sql

+   **[sql] [usecase]**

    为 PostgreSQL 和 Oracle 方言的`Interval`数据类型实现了“字面值处理”，允许对间隔值进行字面渲染。感谢 Indivar Mishra 的拉取请求。

    参考：[#9737](https://www.sqlalchemy.org/trac/ticket/9737)

+   **[sql] [bug]**

    修复了使用`literal_execute=True`时，与其他字面渲染参数的某些组合中多次使用相同绑定参数会导致值渲染错误的问题，这是由于迭代问题引起的。

    此更改也被**回溯**到：1.4.50

    参考：[#10142](https://www.sqlalchemy.org/trac/ticket/10142)

+   **[sql] [bug]**

    为所有包含字面处理的数据类型的“字面处理器”添加了编译器级别的 None/NULL 处理，即在 SQL 语句中内联渲染值而不是作为绑定参数的所有这些类型，对于那些不具有显式“null 值”处理的类型。以前，此行为是未定义且不一致的。

    参考：[#10535](https://www.sqlalchemy.org/trac/ticket/10535)

+   **[sql]**

    删除了未使用的占位符方法`TypeEngine.compare_against_backend()`，这个方法是由非常旧版本的 Alembic 使用的。有关详细信息，请参见[`github.com/sqlalchemy/alembic/issues/1293`](https://github.com/sqlalchemy/alembic/issues/1293)。

### asyncio

+   **[asyncio] [bug]**

    修复了方法`AsyncSession.close_all()`的 bug，该方法之前未能正确工作。还添加了函数`close_all_sessions()`，它是`close_all_sessions()`的等效函数。拉取请求由 Bryan 不可思议提供。

    参考：[#10421](https://www.sqlalchemy.org/trac/ticket/10421)

### postgresql

+   **[postgresql] [bug]**

    修复了 2.0 版本中由[#7744](https://www.sqlalchemy.org/trac/ticket/7744)引起的回归问题，该问题涉及到与其他操作符（如字符串连接）组合使用的 PostgreSQL JSON 运算符的表达式链失去了正确的括号化，这是由于 PostgreSQL 方言特有的实现细节导致的。

    参考：[#10479](https://www.sqlalchemy.org/trac/ticket/10479)

+   **[postgresql] [bug]**

    当使用 asyncpg 后端并使用`BIT`数据类型时，修复了“insertmanyvalues”的 SQL 处理。在 asyncpg 上，`BIT`显然需要使用一个 asyncpg 特定的`BitString`类型，该类型目前在使用此 DBAPI 时被公开，使其与其他所有在此处使用普通位字符串的 PostgreSQL DBAPI 不兼容。在版本 2.1 中的未来修复将会使所有 PG 后端规范化此数据类型。拉取请求由 Sören Oldag 提供。

    参考：[#10532](https://www.sqlalchemy.org/trac/ticket/10532)

### mysql

+   **[mysql] [bug]**

    修复了 MySQL 的“预先 ping”例程中的新不兼容性问题，其中传递给`connection.ping()`的`False`参数，用于禁用不需要的“自动重新连接”功能，在 MySQL 驱动程序和后端中被弃用，并且对于某些版本的 MySQL 原生客户端驱动程序产生警告。它已被 mysqlclient 移除，而对于 PyMySQL 和基于 PyMySQL 的驱动程序，该参数将在某个时间点被弃用并移除，因此使用 API 内省来未来保证这些不同移除阶段的兼容性。

    此更改也被**回溯**到：1.4.50

    参考：[#10492](https://www.sqlalchemy.org/trac/ticket/10492)

### mariadb

+   **[mariadb] [bug]**

    调整了 MySQL / MariaDB 方言，当使用 MariaDB 时，将生成的列默认为 NULL，如果未使用明确的`True`或`False`值指定`Column.nullable`，因为 MariaDB 不支持生成列的“NOT NULL”短语。拉取请求由 Indivar 提供。

    参考：[#10056](https://www.sqlalchemy.org/trac/ticket/10056)

+   **[mariadb] [bug] [regression]**

    为 MySQL/MariaDB 驱动程序之间似乎存在的一个固有问题建立了一个解决方法，即使用 SQLAlchemy 的“空 IN”条件删除 DML 的 RETURNING 结果返回没有行时，不提供 cursor.description，然后产生返回没有行的结果，导致 ORM 中的回归，在 2.0 系列中使用 RETURNING 用于“同步会话”功能的批量删除语句。为了解决这个问题，当检测到“给出 RETURNING 时没有描述”的特定情况时，将生成一个带有正确游标描述的“空结果”，并用于替代不起作用的游标。

    参考：[#10505](https://www.sqlalchemy.org/trac/ticket/10505)

### mssql

+   **[mssql] [用例]**

    增加了对为 SQL Server 实现的`aioodbc`驱动程序的支持，该驱动程序建立在 pyodbc 和通用 aio* 方言架构之上。

    另请参阅

    aioodbc - 在 SQL Server 方言文档中。

    参考：[#6521](https://www.sqlalchemy.org/trac/ticket/6521)

+   **[mssql] [bug] [反射]**

    修复了身份列反射失败的问题，对于具有大于 18 位数的大整数起始值的 bigint 列。

    此更改也**回溯**到：1.4.50

    参考：[#10504](https://www.sqlalchemy.org/trac/ticket/10504)

### oracle

+   **[oracle] [bug]**

    修复了`Interval`数据类型中的问题，在 Oracle 实现未用于 DDL 生成，导致`day_precision`和`second_precision`参数被忽略，尽管此方言支持。感谢 Indivar 的拉取请求。

    参考：[#10509](https://www.sqlalchemy.org/trac/ticket/10509)

+   **[oracle] [bug]**

    修复了 cx_Oracle 方言声称支持比实际上在 SQLAlchemy 的 2.0 系列中支持的更低的 cx_Oracle 版本（7.x）的问题。该方言导入仅在 cx_Oracle 8 或更高版本中才存在的符号，因此运行时方言检查以及 setup.cfg 要求已更新以反映此兼容性。

    参考：[#10470](https://www.sqlalchemy.org/trac/ticket/10470)

## 2.0.22

发布日期：2023 年 10 月 12 日

### orm

+   **[orm] [用例]**

    添加了`Session.get_one()`方法，其行为类似于`Session.get()`，但如果未找到具有提供的主键的实例，则引发异常。感谢 Carlos Sousa 的拉取请求。

    参考：[#10202](https://www.sqlalchemy.org/trac/ticket/10202)

+   **[orm] [用例]**

    添加了一个选项来永久关闭会话。将新参数`Session.close_resets_only`设置为`False`将阻止`Session`在调用`Session.close()`后执行任何其他操作。

    添加了新方法`Session.reset()`，将会将`Session`重置为其初始状态。这是`Session.close()`的别名，除非`Session.close_resets_only`设置为`False`。

    参考：[#7787](https://www.sqlalchemy.org/trac/ticket/7787)

+   **[orm] [bug]**

    修复了一系列`mapped_column()`参数，在使用 pep-593 `Annotated`对象内的`mapped_column()`对象时未被传递，包括`mapped_column.sort_order`，`mapped_column.deferred`，`mapped_column.autoincrement`，`mapped_column.system`，`mapped_column.info`等。

    此外，在`Annotated`中接收的`mapped_column()`中仍不支持有数据类参数，例如`mapped_column.kw_only`，`mapped_column.default_factory`等。当以这种方式在`Annotated`中使用这些参数时，现在会发出警告（并且它们继续被忽略）。

    参考：[#10046](https://www.sqlalchemy.org/trac/ticket/10046)，[#10369](https://www.sqlalchemy.org/trac/ticket/10369)

+   **[orm] [bug]**

    修复了在 ORM 中使用新式`select()`查询调用`Result.unique()`时出现的问题，其中一个或多个列产生的值是“未知可哈希性”，通常在使用像`func.json_build_object()`这样的 JSON 函数时没有提供类型时会在返回的值实际上不可哈希时内部失败。在这种情况下，修复了对接收到的对象进行哈希性测试，如果不可哈希，则提出了信息性错误消息。请注意，对于“已知不可哈希性”的值，例如直接使用`JSON`或`ARRAY`类型时，已经提出了信息性错误消息。

    此处的“哈希性测试”修复也适用于传统的`Query`，但在传统情况下，几乎所有查询都使用`Result.unique()`，因此此处不会发出新的警告；在这种情况下，将继续保持回退到在此情况下使用`id()`的传统行为，改进是现在将未知类型（结果证明是可哈希的）进行唯一化，而以前不会。

    参考：[#10459](https://www.sqlalchemy.org/trac/ticket/10459)

+   **[orm] [bug]**

    修复了最近修订的“insertmanyvalues”功能中的回归问题（可能是问题[#9618](https://www.sqlalchemy.org/trac/ticket/9618)），在这种情况下，ORM 会误将非 RETURNING 结果解释为具有 RETURNING 结果，如果应用了`implicit_returning=False`参数到映射的`Table`，则表示“insertmanyvalues”不能在未提供主键值的情况下使用。

    参考：[#10453](https://www.sqlalchemy.org/trac/ticket/10453)

+   **[orm] [bug]**

    修复了 ORM `with_loader_criteria()`不适用于将 ON 子句给定为普通 SQL 比较而不是关系目标或类似的`Select.join()`的 bug。

    参考：[#10365](https://www.sqlalchemy.org/trac/ticket/10365)

+   **[orm] [bug]**

    修复了`Mapped`等符号在作为子模块元素引用时无法正确解析的问题，假设是基于字符串或“未来注释”样式注释。

    参考：[#10412](https://www.sqlalchemy.org/trac/ticket/10412)

+   **[orm] [bug]**

    修复了使用`__allow_unmapped__`声明选项时的类型问题，其中使用集合类型（如`list[SomeClass]`）声明的类型与使用 typing 构造`List[SomeClass]`声明的类型无法被正确识别的问题。感谢 Pascal Corpet 提供的拉取请求。

    参考：[#10385](https://www.sqlalchemy.org/trac/ticket/10385)

### engine

+   **[engine] [bug]**

    修复了某些方言中的问题，在这些方言中，对于一个根本不返回任何行的 INSERT 语句，方言可能会错误地返回一个空结果集，这是由于仍然存在来自预取或后取主键的遗留物。受影响的方言包括 asyncpg，所有 mssql 方言。

+   **[engine] [bug]**

    修复了在某些垃圾回收/异常场景下，连接池的清理例程会由于意外的状态集而引发错误的问题，这种情况可以在特定条件下重现。

    参考：[#10414](https://www.sqlalchemy.org/trac/ticket/10414)

### sql

+   **[sql] [bug]**

    修复了在 UPDATE 语句的 SET 子句中引用 FROM 条目时，如果该条目在语句中没有其他地方出现，则不会将其包含在 UPDATE 语句的 FROM 子句中的问题；目前对于通过`Update.add_cte()`添加的 CTE，以在语句顶部提供所需的 CTE，会发生这种情况。

    参考：[#10408](https://www.sqlalchemy.org/trac/ticket/10408)

+   **[sql] [bug]**

    修复了 2.0 版本中的回归问题，`DDL`构造不再`__repr__()`，因为已删除的`on`属性未被容纳。感谢 Iuri de Silvio 提供的拉取请求。

    参考：[#10443](https://www.sqlalchemy.org/trac/ticket/10443)

### typing

+   **[typing] [bug]**

    修复了传递给`Values`的参数列表过于严格地与`List`绑定而不是`Sequence`的问题。感谢 Iuri de Silvio 提供的拉取请求。

    参考：[#10451](https://www.sqlalchemy.org/trac/ticket/10451)

+   **[typing] [bug]**

    更新了代码库以支持 Mypy 1.6.0。

### asyncio

+   **[asyncio] [bug]**

    修复了 `AsyncSession.get.execution_options` 参数未传播到底层 `Session` 并被忽略的问题。

### mariadb

+   **[mariadb] [bug]**

    修改了 mariadb-connector 驱动程序，预加载了所有查询的 `cursor.rowcount` 值，以适应像 Pandas 这样硬编码调用 `Result.rowcount` 的工具。SQLAlchemy 通常仅为 UPDATE/DELETE 语句预加载 `cursor.rowcount`，否则传递给 DBAPI，在那里如果没有值可用，则可以返回 -1。但是，mariadb-connector 不支持在关闭游标本身后调用 `cursor.rowcount`，而是引发错误。已添加通用测试支持，以确保所有后端支持在结果关闭后允许 `Result.rowcount` 成功（即返回一个整数值，-1 表示“不可用”）。

    参考：[#10396](https://www.sqlalchemy.org/trac/ticket/10396)

+   **[mariadb] [bug]**

    为 mariadb-connector 方言添加了额外的修复，以支持 INSERT..RETURNING 语句中结果中的 UUID 数据值。

### mssql

+   **[mssql] [bug]**

    修复了一个 bug，即在 SQL Server 中阻止 ORDER BY 在子查询中发出的规则没有在使用 `select.fetch()` 方法限制行数与 WITH TIES 或 PERCENT 结合时被禁用，导致无法使用带有 TOP / ORDER BY 的有效子查询。

    参考：[#10458](https://www.sqlalchemy.org/trac/ticket/10458)

## 2.0.21

发布日期：2023 年 9 月 18 日

### orm

+   **[orm] [bug]**

    调整了 ORM 对“target”实体在 `Update` 和 `Delete` 中的解释，以不干扰传递给语句的目标“from”对象，例如在传递 ORM 映射的 `aliased` 构造时应在“UPDATE FROM”等短语中保留。像使用“SELECT”语句进行 ORM 会话同步的情况，如与 MySQL/MariaDB 一起使用 UPDATE/DELETE 这种形式仍然会有问题，因此最好在使用此类 DML 语句时禁用 synchonize_session。

    参考：[#10279](https://www.sqlalchemy.org/trac/ticket/10279)

+   **[orm] [bug]**

    为 `selectin_polymorphic()` 加入了新的功能，允许其他加载器选项被捆绑为同级，并且引用其子类中的一个，位于父加载器选项的子选项中。以前，只有当 `selectin_polymorphic()` 处于查询选项的顶层时才支持此模式。请参阅新文档部分以获取示例。

    作为这个改变的一部分，改进了`Load.selectin_polymorphic()` 方法/加载策略的行为，因此子类加载不会加载来自父表的大多数已加载列，当选项用于已经进行关系加载的类时。以前，只有在顶级类加载时才有效的加载子类列的逻辑。

    也请参阅

    当 `selectin_polymorphic` 本身是子选项时应用加载器选项

    参考：[#10348](https://www.sqlalchemy.org/trac/ticket/10348)

### 引擎

+   **[引擎] [错误]**

    修复了一系列反射问题，影响到 PostgreSQL、MySQL/MariaDB 和 SQLite 方言，在反射外键约束时，目标列包含一个或两个表名或列名中的括号时。

    参考：[#10275](https://www.sqlalchemy.org/trac/ticket/10275)

### sql

+   **[sql] [用例]**

    调整了 `Enum` 数据类型，接受 `None` 参数作为 `Enum.length` 参数，导致在结果 DDL 中生成没有长度的 VARCHAR 或其他文本类型。这允许在模式中存在后，为该类型添加任何长度的新元素。感谢 Eugene Toder 提交的拉取请求。

    参考：[#10269](https://www.sqlalchemy.org/trac/ticket/10269)

+   **[sql] [用例]**

    添加了新的通用 SQL 函数`aggregate_strings`，它接受一个 SQL 表达式和一个分隔符，将多行字符串连接成单个聚合值。该函数根据后端编译成函数，例如`group_concat()`、`string_agg()`或`LISTAGG()`。感谢 Joshua Morris 提交的拉取请求。

    参考：[#9873](https://www.sqlalchemy.org/trac/ticket/9873)

+   **[sql] [错误]**

    调整了字符串连接运算符的运算优先级，使其与字符串匹配运算符（如`ColumnElement.like()`，`ColumnElement.regexp_match()`，`ColumnElement.match()`等）以及纯粹的`==`相等，该运算符与字符串比较运算符具有相同的优先级，因此将在跟随字符串匹配运算符的字符串连接表达式中应用括号。这为后端（如 PostgreSQL）提供了便利，其中“regexp match”运算符显然比字符串连接运算符的优先级高。

    参考：[#9610](https://www.sqlalchemy.org/trac/ticket/9610)

+   **[sql] [bug]**

    限定了 DDL 编译器中`hashlib.md5()`的使用，该函数用于在 DDL 语句中为长索引和约束名称生成确定性的四个字符后缀，以包含 Python 3.9+中的`usedforsecurity=False`参数，以便于 Python 解释器构建用于受限环境（如 FIPS）时，不将此调用视为与安全问题相关联。

    参考：[#10342](https://www.sqlalchemy.org/trac/ticket/10342)

+   **[sql] [bug]**

    `Values`构造现在将自动为与现有 FROM 子句关联的`column`创建代理（即副本）。这使得像`values_obj.c.colname`这样的表达式将在`colname`作为已与先前的`Values`或其他表构造一起使用的`column`的情况下生成正确的 FROM 子句。最初认为这可能是一个错误条件的候选项，但很可能这种模式已经被广泛使用，所以现在添加以支持。

    参考：[#10280](https://www.sqlalchemy.org/trac/ticket/10280)

### 模式

+   **[schema] [bug]**

    修改了仅适用于 Oracle 后端的 `Identity.order` 参数的呈现方式，该参数是 `Sequence` 和 `Identity` 的一部分，并且不适用于其他后端，例如 PostgreSQL 的后端。未来版本将重命名 `Identity.order`、`Sequence.order` 和 `Identity.on_null` 参数为 Oracle 特定名称，并弃用旧名称，这些参数仅适用于 Oracle。

    此更改也被**回溯**到：1.4.50

    参考：[#10207](https://www.sqlalchemy.org/trac/ticket/10207)

### 输入

+   **[输入] [用例]**

    使 `Mapped` 的包含类型协变；这是为了允许更大的灵活性，以适应端用户类型化场景，例如使用协议表示传递给其他函数的特定映射类结构。作为这个改变的一部分，还使依赖和相关类型（如 `SQLORMOperations`、`WriteOnlyMapped` 和 `SQLColumnExpression`）的包含类型也是协变的。感谢 Roméo Després 提交的拉取请求。

    参考：[#10288](https://www.sqlalchemy.org/trac/ticket/10288)

+   **[输入] [错误]**

    修复了在 2.0.20 中引入的回归问题，通过 [#9600](https://www.sqlalchemy.org/trac/ticket/9600) 修复，尝试为 `MetaData.naming_convention` 添加更正式的类型。这个改变阻止了基本命名约定字典通过类型检查，并且已经进行了调整，以便再次接受字符串键的普通字典以及使用约束类型作为键或两者混合使用的字典。

    作为这个改变的一部分，还对命名约定字典的较少使用的形式进行了类型化，包括它目前允许 `Constraint` 类型对象作为键。

    参考：[#10264](https://www.sqlalchemy.org/trac/ticket/10264)，[#9284](https://www.sqlalchemy.org/trac/ticket/9284)

+   **[输入] [错误]**

    修复了应用于表达式构造的基础`Visitable`类的`__class_getitem__()`方法的类型注释，以接受`Any`作为键，而不是`str`，这有助于一些 IDE（例如 PyCharm）在尝试为包含通用选择器的 SQL 构造编写类型注释时。感谢 Jordan Macdonald 的拉取请求。

    参考：[#9878](https://www.sqlalchemy.org/trac/ticket/9878)

+   **[typing] [bug]**

    修复了核心“SQL 元素”类`SQLCoreOperations`的类型问题，以支持从类型的角度来看`__hash__()`方法，因为对象（如`Column`和 ORM `InstrumentedAttribute`）是可散列的，并且在`Update`和`Insert`构造的公共 API 中用作字典键。先前，类型检查器不知道根 SQL 元素是可散列的。

    参考：[#10353](https://www.sqlalchemy.org/trac/ticket/10353)

+   **[typing] [bug]**

    修复了`Existing.select_from()`的类型问题，该问题阻止了它与 ORM 类的使用。

    参考：[#10337](https://www.sqlalchemy.org/trac/ticket/10337)

+   **[typing] [bug]**

    更新 ORM 加载选项的类型注释，将其限制为仅接受“*”而不是任何字符串作为字符串参数。感谢 Janek Nouvertné的拉取请求。

    参考：[#10131](https://www.sqlalchemy.org/trac/ticket/10131)

### postgresql

+   **[postgresql] [bug]**

    修复了在 2.0 版本中出现的回归问题，该问题由于[#8491](https://www.sqlalchemy.org/trac/ticket/8491)而引起，其中当使用`create_engine.pool_pre_ping`参数时，用于 PostgreSQL 方言的修订的“ping”会干扰 asyncpg 与 PGBouncer“transaction”模式的使用，因为 asnycpg 发出的多个 PostgreSQL 命令可能会被分解到多个连接中导致错误，由于这个新修订的“ping”周围没有任何事务。现在在事务内调用 ping，与所有其他基于 pep-249 DBAPI 的其他后端隐式使用的方式相同；这确保了为此命令发送的一系列 PG 命令在同一个后端连接上调用，而不会在命令中途跳转到不同的连接。如果使用 asyncpg 方言处于“AUTOCOMMIT”模式，则不使用事务，这仍然与 pgbouncer 事务模式不兼容。

    参考：[#10226](https://www.sqlalchemy.org/trac/ticket/10226)

### 杂项

+   **[bug] [setup]**

    修复了一个很久以前的问题，在 pytest 运行之外无法导入 SQLAlchemy 模块的全部内容，包括`sqlalchemy.testing.fixtures`。这适用于诸如 `pkgutil` 等尝试导入所有包中所有安装的模块的检查工具。

    参考：[#10321](https://www.sqlalchemy.org/trac/ticket/10321)

## 2.0.20

发布日期：2023 年 8 月 15 日

### orm

+   **[orm] [usecase]**

    实现了对启用 ORM 的 DML 语句的“RETURNING '*'”用例。这将尽可能地呈现，并返回未经过滤的结果集，但不支持具有特定列渲染要求的多参数“ORM 批量 INSERT”语句。

    参考：[#10192](https://www.sqlalchemy.org/trac/ticket/10192)

+   **[orm] [bug]**

    修复了一个基本问题，阻止了某些形式的 ORM “注释” 对使用 `Select.join()` 进行关系目标的连接的子查询进行。在诸如 `PropComparator.and_()` 和其他 ORM 特定情况下使用这些注释。  

    此更改也被**回溯**到：1.4.50

    参考：[#10223](https://www.sqlalchemy.org/trac/ticket/10223)

+   **[orm] [bug]**

    修复了一个问题，即 ORM 在具有相同名称列的超类和子类的连接继承模型中生成 SELECT 时，某种方式未能将正确的列名列表发送到 `CTE` 构造函数，当生成 RECURSIVE 列表时。

    参考：[#10169](https://www.sqlalchemy.org/trac/ticket/10169)

+   **[orm] [bug]**

    修复了一个相当重要的问题，即传递给 `Session.execute()` 的执行选项以及本地于 ORM 执行的语句本身的执行选项不会传播到 eager loaders，如 `selectinload()`、`immediateload()` 和 `sqlalchemy.orm.subqueryload()`，使得不可能禁用单个语句的缓存或使用 `schema_translate_map` 用于单个语句，以及使用用户自定义执行选项。已做出更改，使得所有针对 `Session.execute()` 的面向用户的执行选项都将传播到其他加载器。

    作为此更改的一部分，可以通过将 `execution_options={"compiled_cache": None}` 发送到`Session.execute()` 来在每个语句范围内为“过度深入”的急加载器警告消除缓存被禁用，这将禁用该范围内所有语句的缓存。

    参考：[#10231](https://www.sqlalchemy.org/trac/ticket/10231)

+   **[orm] [bug]**

    修复了 ORM 在像 `Comparator.any()` 这样的表达式中使用的内部克隆，以生成相关 EXISTS 构造会干扰 SQL 编译器的“笛卡尔积警告”功能，导致 SQL 编译器在所有语句元素都正确连接时发出警告。

    参考：[#10124](https://www.sqlalchemy.org/trac/ticket/10124)

+   **[orm] [bug]**

    修复了 `lazy="immediateload"` 加载策略在某些情况下会将内部加载令牌放置到 ORM 映射属性中的问题，例如在递归自引用加载中不应发生加载的情况。作为此更改的一部分，`lazy="immediateload"` 策略现在以与其他急加载器相同的方式尊重 `relationship.join_depth` 参数进行自引用急加载，其中将其设置为未设置或设置为零将导致自引用的即时加载不会发生，将其设置为一个或更大的值将即时加载直到给定深度。

    参考：[#10139](https://www.sqlalchemy.org/trac/ticket/10139)

+   **[orm] [bug]**

    修复了诸如`attribute_keyed_dict()`之类的基于字典的集合在完全正确地 pickle/unpickle 时未能完全 pickle/unpickle 的问题，导致在 unpickling 后尝试变异此类集合时出现问题。

    参考：[#10175](https://www.sqlalchemy.org/trac/ticket/10175)

+   **[orm] [bug]**

    修复了从另一个急加载器使用 `aliased()` 对加入的继承子类进行列局部操作时，链式调用 `load_only()` 或其他通配符使用 `defer()` 会失败的问题。

    参考：[#10125](https://www.sqlalchemy.org/trac/ticket/10125)

+   **[orm] [bug]**

    修复了一个问题，即启用 ORM 的`select()` 构造不会渲染任何仅通过`Select.add_cte()` 方法添加的 CTE，这些 CTE 在语句中没有被引用。

    参考资料：[#10167](https://www.sqlalchemy.org/trac/ticket/10167)

### 例子

+   **[examples] [bug]**

    dogpile_caching 例子已更新为 2.0 风格的查询。在“缓存查询”逻辑中，添加了一个条件来区分 `Query` 和 `select()` 在执行无效操作时的情况。

### 引擎

+   **[engine] [bug]**

    修复了一个关键问题，即将 `create_engine.isolation_level` 设置为 `AUTOCOMMIT`（而不是使用 `Engine.execution_options()` 方法），如果临时选择了替代隔离级别，那么会在使用 `Connection.execution_options.isolation_level` 时，无法将“自动提交”恢复到池连接中。

    参考资料：[#10147](https://www.sqlalchemy.org/trac/ticket/10147)

### sql

+   **[sql] [bug]**

    修复了反序列化 `Column` 或其他 `ColumnElement` 时无法恢复正确“比较器”对象的问题，该对象用于生成特定于类型对象的 SQL 表达式。

    这个更改也被 **回溯** 到了：1.4.50

    参考资料：[#10213](https://www.sqlalchemy.org/trac/ticket/10213)

### 类型

+   **[typing] [usecase]**

    添加了新的类型仅实用函数 `Nullable()` 和 `NotNullable()` 以分别将列或 ORM 类型定义为可为空或不可为空。这些函数在运行时无操作，返回未更改的输入。

    参考资料：[#10173](https://www.sqlalchemy.org/trac/ticket/10173)

+   **[typing] [bug]**

    类型改进：

    +   对一些没有返回的 DML 使用 `Session.execute()` 时，返回 `CursorResult`

    +   修复了 `Query.with_for_update.of` 参数在 `Query.with_for_update()` 中的类型错误

    +   对一些 DML 方法使用的 `_DMLColumnArgument` 类型进行改进，以传递列表达式

    +   添加了对`literal()`的重载，因此推断返回类型为`BindParameter[NullType]`，其中`literal.type_`参数为 None

    +   为`ColumnElement.op()`添加重载，以便在未提供`ColumnElement.op.return_type`时推断类型为`Callable[[Any], BinaryExpression[Any]]`

    +   添加了对`ColumnElement.__add__()`的丢失重载

    拉取请求由 Mehdi Gmira 提供。

    参考：[#9185](https://www.sqlalchemy.org/trac/ticket/9185)

+   **[typing] [错误]**

    修复了`Session`和`AsyncSession`等方法中的问题，例如`Session.connection()`在`Session.connection.execution_options`参数硬编码为不面向用户的内部类型的情况。

    参考：[#10182](https://www.sqlalchemy.org/trac/ticket/10182)

### asyncio

+   **[asyncio] [用例]**

    添加了新方法`AsyncConnection.aclose()`作为`AsyncConnection.close()`的同义词和`AsyncSession.aclose()`作为`AsyncSession.close()`的同义词，以提供与 Python 标准库`@contextlib.aclosing`构造的兼容性。拉取请求由 Grigoriev Semyon 提供。

    参考：[#9698](https://www.sqlalchemy.org/trac/ticket/9698)

### mysql

+   **[mysql] [用例]**

    更新了 aiomysql 方言，因为该方言似乎再次得到维护。重新添加到使用版本 0.2.0 进行的 ci 测试。

    这个变更也被**回溯**到：1.4.50

## 2.0.19

发布日期：2023 年 7 月 15 日

### orm

+   **[orm] [错误]**

    修复了直接设置关系集合的问题，其中新集合中的对象已经存在时，不会触发该对象的级联事件，导致如果该对象不存在，则不会被添加到`Session`中。这与[#6471](https://www.sqlalchemy.org/trac/ticket/6471)类似，并且由于在 2.0 系列中删除了`cascade_backrefs`，这个问题更加明显。作为[#6471](https://www.sqlalchemy.org/trac/ticket/6471)的一部分添加的`AttributeEvents.append_wo_mutation()`事件现在也会对同一集合的现有成员发出信号，这些成员在该集合的批量设置中存在。

    参考：[#10089](https://www.sqlalchemy.org/trac/ticket/10089)

+   **[orm] [bug]**

    修复了通过 backref 与未加载集合关联的对象，但由于在 2.0 系列中删除了`cascade_backrefs`而未合并到`Session`中的问题，将不会发出警告，表明这些对象未包含在刷新中，即使它们是集合的待处理成员；在其他情况下，当要刷新的集合包含将被基本丢弃的非附加对象时，会发出警告。对于 backref 待处理集合成员的警告的添加建立了与可能根据不同的关系加载策略在不同时间基于不同时间刷新或不刷新的集合的更大一致性。

    参考：[#10090](https://www.sqlalchemy.org/trac/ticket/10090)

+   **[orm] [bug] [regression]**

    修复了由[#9805](https://www.sqlalchemy.org/trac/ticket/9805)引起的额外回归，其中对语句上“ORM”标志的更积极传播可能导致在 ORM `Query`构造中嵌入一个不包含 ORM 实体的 Core SQL 语句时出现内部属性错误， 在这种情况下，ORM 启用的 UPDATE 和 DELETE 语句。

    参考：[#10098](https://www.sqlalchemy.org/trac/ticket/10098)

### engine

+   **[engine] [bug]**

    将 `Row.t` 和 `Row.tuple()` 重命名为 `Row._t` 和 `Row._tuple()`；这是为了符合所有方法和预定义属性都应采用 Python 标准库 `namedtuple` 风格的策略，其中所有固定名称都有一个前导下划线，以避免与现有列名称冲突。先前的方法/属性现已被弃用，并将发出弃用警告。

    参考：[#10093](https://www.sqlalchemy.org/trac/ticket/10093)

+   **[engine] [bug]**

    向 `make_url()` 函数添加了对非字符串、非 `URL` 对象的检测，允许立即抛出 `ArgumentError`，而不是稍后引发故障。特殊逻辑确保允许通过模拟 `URL` 的形式。拉取请求由 Grigoriev Semyon 提供。

    参考：[#10079](https://www.sqlalchemy.org/trac/ticket/10079)

### postgresql

+   **[postgresql] [bug]**

    修复了 PostgreSQL URL 解析改进引起的回归问题 [#10004](https://www.sqlalchemy.org/trac/ticket/10004)，其中带有冒号的“host”查询字符串参数，以支持各种第三方代理服务器和/或方言，将无法正确解析，因为这些被解析为 `host:port` 组合。解析已更新，只有当主机名仅包含字母数字字符，并且只包含点或短划线时（例如，没有斜杠），才将冒号视为表示 `host:port` 值的标记，后跟一个零个或多个整数的整数标记。在所有其他情况下，将整个字符串视为主机。

    参考：[#10069](https://www.sqlalchemy.org/trac/ticket/10069)

+   **[postgresql] [bug]**

    修复了对 `CITEXT` 数据类型进行比较时将右侧强制转换为 `VARCHAR` 的问题，导致右侧未被解释为 `CITEXT` 数据类型，对于 asyncpg、psycopg3 和 pg80000 方言。这导致 `CITEXT` 类型在实际使用中基本上无法使用；现已修复此问题，并已更正测试套件以正确断言表达式是否被正确渲染。

    参考：[#10096](https://www.sqlalchemy.org/trac/ticket/10096)

## 2.0.18

发布日期：2023 年 7 月 5 日

### 引擎

+   **[engine] [bug]**

    调整了`create_engine.schema_translate_map`功能，以便**所有**语句中的模式名称都现在被标记化，无论指定的名称是否在给定的立即模式翻译映射中，并在执行时回退到原始名称的替换。这两个更改允许在每次运行时使用包含或不包含各种键的模式翻译映射来重复使用已编译的对象，从而允许在每次使用具有不同键集的模式翻译映射时继续运行时缓存 SQL 构造。另外，增加了检测在同一语句的多次调用中获得或失去`None`键的 schema_translate_map 字典，这会影响语句的编译，并且与缓存不兼容；针对这些情况引发异常。

    References: [#10025](https://www.sqlalchemy.org/trac/ticket/10025)

### sql

+   **[sql] [bug]**

    修复了使用“flags”时`ColumnOperators.regexp_match()`不会生成“稳定”缓存键的问题，也就是说，缓存键每次都会改变，导致缓存污染。对于`ColumnOperators.regexp_replace()`也存在同样的问题，包括标志和实际替换表达式。现在，标志被表示为固定的修饰符字符串，呈现为安全字符串，而不是绑定参数，并且替换表达式在“binary”元素的主要部分中建立，以便生成适当的缓存键。

    请注意，作为此更改的一部分，`ColumnOperators.regexp_match.flags`和`ColumnOperators.regexp_replace.flags` 已修改为仅呈现为文字字符串，而以前它们是呈现为完整的 SQL 表达式，通常是绑定参数。这些参数应始终作为普通的 Python 字符串传递，而不是作为 SQL 表达式构造；不希望在实践中使用 SQL 表达式构造该参数，因此这是一个不向后兼容的更改。

    该更改还修改了生成的表达式的内部结构，对于带有或不带有标志的 `ColumnOperators.regexp_replace()`，以及对于带有标志的 `ColumnOperators.regexp_match()`。可能已经实现了自己的 regexp 实现的第三方方言（在搜索中找不到此类方言，因此影响预计很低）需要调整结构的遍历以适应。

    此更改也 **被回溯** 至：1.4.49

    参考：[#10042](https://www.sqlalchemy.org/trac/ticket/10042)

+   **[sql] [错误]**

    修复了主要内部 `CacheKey` 构造中 `__ne__()` 运算符未正确实现的问题，导致比较 `CacheKey` 实例时结果荒谬。

    此更改也 **被回溯** 至：1.4.49

### 扩展

+   **[扩展] [用例]**

    为 `association_proxy()` `association_proxy.create_on_none_assignment` 添加了新选项；当一个关联代理引用标量关系被赋值为 `None` 且引用的对象不存在时，通过创建者创建一个新对象。这显然是 1.2 系列中的一个未定义行为，被悄悄移除了。

    参考：[#10013](https://www.sqlalchemy.org/trac/ticket/10013)

### 键入

+   **[键入] [用例]**

    在使用来自 `sqlalchemy.sql.operators` 的独立运算符函数（如 `sqlalchemy.sql.operators.eq`）时，改进了类型。

    参考：[#10054](https://www.sqlalchemy.org/trac/ticket/10054)

+   **[键入] [错误]**

    修复了在 `aliased()` 构造中的一些类型问题，以正确接受已用 `Table.alias()` 别名的 `Table` 对象，以及对 `FromClause` 对象作为“selectable”参数的一般支持，因为这是完全支持的。

    参考：[#10061](https://www.sqlalchemy.org/trac/ticket/10061)

### postgresql

+   **[postgresql] [usecase]**

    为 asyncpg 方言添加了多主机支持。 还对“多主机”用例的 PostgreSQL URL 例程进行了一般改进和错误检查。 感谢 Ilia Dmitriev 提供的拉取请求。

    另见

    多主机连接

    参考：[#10004](https://www.sqlalchemy.org/trac/ticket/10004)

+   **[postgresql] [bug]**

    为所有 PostgreSQL 方言添加了新参数 `native_inet_types=False`，表示 DBAPI 使用的转换器将禁用 PostgreSQL `INET` 和 `CIDR` 列的行转换为 Python `ipaddress` 数据类型，而返回字符串。 这允许编写代码以使用这些数据类型的字符串进行迁移，而无需进行代码更改，只需将此参数添加到 `create_engine()` 或 `create_async_engine()` 函数调用中。

    另见

    网络数据类型

    参考：[#9945](https://www.sqlalchemy.org/trac/ticket/9945)

### mariadb

+   **[mariadb] [usecase] [reflection]**

    允许从 MariaDB 反射 `UUID` 列。 这使得 Alembic 能够正确检测现有 MariaDB 数据库中此类列的类型。

    参考：[#10028](https://www.sqlalchemy.org/trac/ticket/10028)

### mssql

+   **[mssql] [usecase]**

    添加了对 MSSQL 方言中 COLUMNSTORE 索引的创建和反射的支持。 可以在指定 `mssql_columnstore=True` 的索引上指定。

    参考：[#7340](https://www.sqlalchemy.org/trac/ticket/7340)

+   **[mssql] [bug] [sql]**

    修复了将 `Cast` 执行到具有显式排序规则的字符串类型时，将在 CAST 函数内部渲染 COLLATE 子句的问题，从而导致语法错误。

    参考：[#9932](https://www.sqlalchemy.org/trac/ticket/9932)

## 2.0.17

发布日期：2023 年 6 月 23 日

### orm

+   **[orm] [bug] [regression]**

    修复了 2.0 系列中的回归，其中使用 `undefer_group()` 与 `selectinload()` 或 `subqueryload()` 的查询会引发 `AttributeError`。 感谢 Matthew Martin 提供的拉取请求。

    参考：[#9870](https://www.sqlalchemy.org/trac/ticket/9870)

+   **[orm] [bug]**

    修复了在 ORM Annotated Declarative 中的问题，该问题阻止了在不返回`Mapped`数据类型的混合使用`declared_attr`的情况，而是返回了额外的 ORM 数据类型，如`AssociationProxy`。声明式运行时错误地尝试将此注释解释为需要`Mapped`并引发错误。

    参考：[#9957](https://www.sqlalchemy.org/trac/ticket/9957)

+   **[orm] [bug] [typing]**

    修复了使用`declared_attr`函数从`AssociationProxy`返回类型的类型问题被禁止的问题。

    参考：[#9957](https://www.sqlalchemy.org/trac/ticket/9957)

+   **[orm] [bug] [regression]**

    修复了在 2.0.16 中由[#9879](https://www.sqlalchemy.org/trac/ticket/9879)引入的回归，其中将可调用对象传递给`mapped_column.default`参数时，同时设置`init=False`会将此值解释为 Dataclass 默认值，该值将直接分配给新实例的对象，绕过了作为底层`Column.default`值生成器的默认生成器的过程。现在检测到这种情况，以保持先前的行为，但对于这种模棱两可的用法会发出弃用警告；要为`Column`填充默认生成器，应使用`mapped_column.insert_default`参数，该参数与固定名称的`mapped_column.default`参数相区分，其名称根据 pep-681 固定。

    参考：[#9936](https://www.sqlalchemy.org/trac/ticket/9936)

+   **[orm] [bug]**

    对 ORM `Session` “状态更改”系统进行了额外的加固和文档，该系统检测到同时使用 `Session` 和 `AsyncSession` 对象的并发使用；在获取来自底层引擎的连接的过程中添加了额外的检查，这是关于内部连接管理的关键部分。

    参考：[#9973](https://www.sqlalchemy.org/trac/ticket/9973)

+   **[orm] [bug]**

    在 ORM 加载器策略逻辑中修复了问题，进一步允许在复杂的继承多态/别名/of_type()关系链上的长链中使用`contains_eager()`加载器选项，以便在查询中正确生效。

    参考：[#10006](https://www.sqlalchemy.org/trac/ticket/10006)

+   **[orm] [bug]**

    修复了对 `Enum` 数据类型在 `registry.type_annotation_map` 中的支持，该支持是作为 [#8859](https://www.sqlalchemy.org/trac/ticket/8859) 的一部分首次添加的，在此过程中，如果在映射中使用了带有固定配置的自定义 `Enum`，则会失败传递 `Enum.name` 参数，这将导致 PostgreSQL 枚举无法正常工作，如果枚举值被传递为单个值，则会产生其他问题。逻辑已更新，以便传递“名称”，但也使默认 `Enum` 不会设置硬编码名称为`"enum"`，该默认枚举是针对纯 Python 枚举 enum.Enum 类或其他“空”枚举的。

    参考：[#9963](https://www.sqlalchemy.org/trac/ticket/9963)

### ORM 声明式

+   **[orm] [declarative] [bug]**

    当将 ORM `relationship()` 和其他 `MapperProperty` 对象同时分配给两个不同的类属性时，会发出警告；只有其中一个属性会被映射。对于此条件，已经为 `Column` 和 `mapped_column` 对象设置了警告。

    参考：[#3532](https://www.sqlalchemy.org/trac/ticket/3532)

### 扩展

+   **[extensions] [bug]**

    修复了与 mypy 1.4 结合使用的 mypy 插件中的问题。

    这个更改也被 **回溯** 到：1.4.49

### typing

+   **[typing] [bug]**

    修复了类型问题，该问题导致无法完全在 ORM 查询中使用 `WriteOnlyMapped` 和 `DynamicMapped` 属性。

    引用：[#9985](https://www.sqlalchemy.org/trac/ticket/9985)

### postgresql

+   **[postgresql] [usecase]**

    pg8000 方言现在支持 RANGE 和 MULTIRANGE 数据类型，使用现有的 Range and Multirange Types 中描述的 RANGE API。 Range 和 multirange 类型在 pg8000 驱动程序的版本 1.29.8 中受支持。感谢 Tony Locke 提供的拉取请求。

    引用：[#9965](https://www.sqlalchemy.org/trac/ticket/9965)

## 2.0.16

发布日期：2023 年 6 月 10 日

### 平台

+   **[platform] [usecase]**

    兼容性改进，使完整的测试套件可以在 Python 3.12.0b1 上通过。

### orm

+   **[orm] [usecase]**

    改进了 `DeferredReflection.prepare()`，使其接受传递给 `MetaData.reflect()` 的任意 `**kw` 参数，允许使用诸如反射视图等用例以及传递给方言特定参数。另外，还现代化了 `DeferredReflection.prepare.bind` 参数，以便将 `Engine` 或 `Connection` 作为“bind”参数接受。

    引用：[#9828](https://www.sqlalchemy.org/trac/ticket/9828)

+   **[orm] [bug]**

    修复了 `DeclarativeBaseNoMeta` 声明基类无法与非映射混合类或抽象类一起使用的问题，而是引发 `AttributeError`。

    引用：[#9862](https://www.sqlalchemy.org/trac/ticket/9862)

+   **[orm] [bug] [regression]**

    修复了 2.0 系列中的一个回归，其中 `validates.include_backrefs` 的默认值在 `validates()` 函数中更改为 `False`。现在将该默认值恢复为 `True`。

    引用：[#9820](https://www.sqlalchemy.org/trac/ticket/9820)

+   **[orm] [bug]**

    在版本 2.0.11 中作为[#9583](https://www.sqlalchemy.org/trac/ticket/9583)的一部分新增的功能中修复了一个 bug，该功能允许在 ORM 按主键批量更新时与 WHERE 子句一起使用，发送不包含每行主键值的字典将通过批量处理，并为行包括“pk=NULL”，但不会引发异常。如果未提供批量更新的主键值，则现在会引发异常。

    引用：[#9917](https://www.sqlalchemy.org/trac/ticket/9917)

+   **[orm] [bug] [dataclasses]**

    修复了一个问题，其中生成指定了`default`值并设置`init=False`的 dataclasses 字段将无效。在这种情况下，dataclasses 行为是在类上设置默认值，这与 SQLAlchemy 使用的描述符不兼容。为了支持这种情况，在生成 dataclass 时将默认值转换为`default_factory`。

    引用：[#9879](https://www.sqlalchemy.org/trac/ticket/9879)

+   **[orm] [bug]**

    当向`Mapper`添加属性时已经配置了 ORM 映射属性，或者类上已经存在属性时，将发出弃用警告。先前，对于此情况存在一个非弃用警告，但并非始终一致发出。已改进此警告的逻辑，以便在检测到属性的终端用户替换时发出警告，同时对于内部 Declarative 和其他情况，其中用新的属性替换描述符是预期的情况，不会产生误报。

    引用：[#9841](https://www.sqlalchemy.org/trac/ticket/9841)

+   **[orm] [bug]**

    改进了`registry.map_imperatively()`方法的`map_imperatively.local_table`参数上的参数检查，确保只传递`Table`或其他`FromClause`，而不是已存在的映射类，因为该对象将被进一步解释为新的映射时会导致未定义的行为。

    引用：[#9869](https://www.sqlalchemy.org/trac/ticket/9869)

+   **[orm] [bug]**

    `InstanceState.unloaded_expirable`属性是`InstanceState.unloaded`的同义词，现在已弃用；此属性始终是特定于实现的，不应公开。

    参考：[#9913](https://www.sqlalchemy.org/trac/ticket/9913)

### asyncio

+   **[asyncio] [用例]**

    添加了新的`create_async_engine.async_creator`参数到`create_async_engine()`，其完成了与`create_engine.creator`参数相同的目的`create_engine()`。这是一个不带参数的可调用对象，提供一个新的 asyncio 连接，直接使用 asyncio 数据库驱动程序。`create_async_engine()`函数将在适当的结构中封装驱动程序级别的连接。贡献者 Jack Wotherspoon。

    参考：[#8215](https://www.sqlalchemy.org/trac/ticket/8215)

### postgresql

+   **[postgresql] [用例] [反射]**

    在 PostgreSQL 反射中使用`ARRAY_AGG`时，将`NAME`列强制转换为`TEXT`。这似乎改善了一些 PostgreSQL 派生产品可能不支持`NAME`类型的聚合的兼容性。

    参考：[#9838](https://www.sqlalchemy.org/trac/ticket/9838)

+   **[postgresql] [用例]**

    统一了自定义 PostgreSQL 运算符定义，因为它们在多种不同的数据类型之间共享。

    参考：[#9041](https://www.sqlalchemy.org/trac/ticket/9041)

+   **[postgresql] [用例]**

    添加了对 PostgreSQL 10 `NULLS NOT DISTINCT` 唯一索引和唯一约束功能的支持，使用方言选项 `postgresql_nulls_not_distinct`。更新反射逻辑以正确考虑此选项。贡献者 Pavel Siarchenia。

    参考：[#8240](https://www.sqlalchemy.org/trac/ticket/8240)

+   **[postgresql] [错误]**

    在 PostgreSQL 特定运算符上使用适当的优先级，如`@>`。以前优先级错误，导致针对`ANY`或`ALL`结构呈现时括号错误。

    参考：[#9836](https://www.sqlalchemy.org/trac/ticket/9836)

+   **[postgresql] [错误]**

    修复了一个问题，即`ColumnOperators.like.escape`和类似的参数不允许空字符串作为参数，该参数将作为“转义”字符传递；这是 PostgreSQL 支持的语法。贡献者 Martin Caslavsky。

    参考：[#9907](https://www.sqlalchemy.org/trac/ticket/9907)

## 2.0.15

发布日期：2023 年 5 月 19 日

### orm

+   **[orm] [错误]**

    随着越来越多的项目使用新式“2.0”ORM 查询，显然，“autoflush”的条件性，即基于给定语句是否引用 ORM 实体，正在变得更加重要。到目前为止，“ORM”标志对于语句是否返回与 ORM 实体或列对应的行已经存在了一定程度的松散关联；“ORM”标志的原始目的是启用应用于 Core 结果集的 ORM 实体获取规则以及应用于语句的 ORM 加载策略的后处理。对于不基于包含 ORM 实体的行构建的语句，认为“ORM”标志基本上是不必要的。

    仍然可能存在“autoflush”对于*所有*使用`Session.execute()`和相关方法的情况更加有效的情况，即使是对纯粹的 Core SQL 构造也是如此。然而，这可能会影响到未预期的旧有情况，并且可能更多地成为 2.1 版本的事情。但是，现在“ORM-标志”的规则已经被放宽，因此包含任何 ORM 实体或属性的语句，包括仅在 WHERE / ORDER BY / GROUP BY 子句中的语句，在标量子查询中等等，都将启用此标志。这将导致这样的语句发生“autoflush”，并且还可以通过`ORMExecuteState.is_orm_statement`事件级别属性可见。

    引用：[#9805](https://www.sqlalchemy.org/trac/ticket/9805)

### postgresql

+   **[postgresql] [bug] [regression]**

    修复了针对 PostgreSQL 方言的基础`Uuid`数据类型，以便在选择“native_uuid”时充分利用 PG 特定的 `UUID` 方言特定的数据类型，以便包含 PG 驱动程序行为。这个问题因为作为[#9618](https://www.sqlalchemy.org/trac/ticket/9618)的一部分进行的 insertmanyvalues 改进而显现出来，其中类似于[#9739](https://www.sqlalchemy.org/trac/ticket/9739)，asyncpg 驱动程序对是否存在数据类型转换非常敏感，当使用此通用类型时，必须调用 PostgreSQL 驱动程序特定的本机 `UUID` 数据类型，以便进行这些转换。

    引用：[#9808](https://www.sqlalchemy.org/trac/ticket/9808)

## 2.0.14

发布日期：2023 年 5 月 18 日

### orm

+   **[orm] [bug]**

    在一个特定领域修改了`JoinedLoader`的实现，以使用更简单的方法，此前它使用了一个在多个线程之间共享的缓存结构。这样做的理由是为了避免潜在的竞争条件，这被怀疑是导致一个特定崩溃的原因，该崩溃已经被多次报告。所讨论的缓存结构最终仍然通过编译的 SQL 缓存“缓存”，因此不会预期性能下降。

    参考：[#9777](https://www.sqlalchemy.org/trac/ticket/9777)

+   **[orm] [bug] [回归]**

    修复了在`CTE`构造中使用`update()`或`delete()`后，在`select()`中使用会引发`CompileError`的回归问题，这是由于执行 ORM 级别更新/删除语句的 ORM 相关规则造成的。

    参考：[#9767](https://www.sqlalchemy.org/trac/ticket/9767)

+   **[orm] [bug]**

    在新的 ORM 注释式声明中修复了问题，其中通过`pep-593`的`Annotated`将`ForeignKey`（或其他列级约束）放在`mapped_column()`中，然后通过模型复制到模型会将每个约束的副本应用到生成的`Table`中的`Column`中，导致不正确的 CREATE TABLE DDL 以及在 Alembic 下的迁移指令。

    参考：[#9766](https://www.sqlalchemy.org/trac/ticket/9766)

+   **[orm] [bug]**

    使用`joinedload()`加载器选项时，如果使用额外的关系标准，其中额外的标准本身包含引用联接实体的相关子查询，因此也需要对别名实体进行“调整”，则将排除这种适应，从而为`joinedload`生成错误的 ON 子句。

    参考：[#9779](https://www.sqlalchemy.org/trac/ticket/9779)

### sql

+   **[sql] [用例]**

    将 MSSQL 的`try_cast()`函数泛化到`sqlalchemy.`导入命名空间中，以便第三方方言也可以实现它。在 SQLAlchemy 中，`try_cast()`函数仍然是一个仅适用于 SQL Server 的构造，如果在不支持它的后端使用它，则会引发`CompileError`。

    `try_cast()` 实现了一个 CAST，其中无法转换的转换返回为 NULL，而不是引发错误。理论上，该构造可以由 Google BigQuery、DuckDB 和 Snowflake 等第三方方言实现，并可能是其他方言。

    感谢 Nick Crews 的拉取请求。

    引用：[#9752](https://www.sqlalchemy.org/trac/ticket/9752)

+   **[sql] [bug]**

    修复了 `values()` 构造中的问题，在标量子查询中使用该构造将导致内部编译错误。

    引用：[#9772](https://www.sqlalchemy.org/trac/ticket/9772)

### postgresql

+   **[postgresql] [bug]**

    修复了一个显然非常久远的问题，即当 `ENUM.create_type` 参数设置为其非默认值 `False` 时，当复制其所属的 `Column` 时，它将不会被传播，这在使用 ORM 声明性混合时很常见。

    引用：[#9773](https://www.sqlalchemy.org/trac/ticket/9773)

### 测试

+   **[tests] [bug] [pypy]**

    修复了依赖于 `sys.getsizeof()` 函数不在 pypy 上运行的测试问题，在 pypy 上，该函数似乎具有与 cpython 不同的行为。

    引用：[#9789](https://www.sqlalchemy.org/trac/ticket/9789)

## 2.0.13

发布日期：2023 年 5 月 10 日

### orm

+   **[orm] [bug]**

    修复了 ORM 声明式注释在所有情况下无法正确解析前向引用的问题；特别是在与 Pydantic 数据类结合使用 `from __future__ import annotations` 时。

    引用：[#9717](https://www.sqlalchemy.org/trac/ticket/9717)

+   **[orm] [bug]**

    修复了新的使用 RETURNING 与 upsert 语句功能中的问题，其中 `populate_existing` 执行选项未被传播到加载选项，导致现有属性无法被就地刷新。

    引用：[#9746](https://www.sqlalchemy.org/trac/ticket/9746)

+   **[orm] [bug]**

    修复了加载器策略路径问题，例如 `joinedload()` / `selectinload()` 将无法完全遍历多层次深度的加载问题，后跟具有 `with_polymorphic()` 或类似构造的中间成员。

    引用：[#9715](https://www.sqlalchemy.org/trac/ticket/9715)

+   **[orm] [bug]**

    修复了 `mapped_column()` 构造中的问题，当 ORM 映射属性引用相同的 `Column` 时，如果涉及 `mapped_column()` 构造，将不会发出“直接多次命名列 X”的正确警告，而是引发内部断言。

    参考：[#9630](https://www.sqlalchemy.org/trac/ticket/9630)

### sql

+   **[sql] [usecase]**

    实现了对包含多个未相关的表的 UPDATE 和 DELETE 语句的“笛卡尔积警告”。

    参考：[#9721](https://www.sqlalchemy.org/trac/ticket/9721)

+   **[sql] [bug]**

    修复了特定方言浮点/双精度类型的基类；Oracle `BINARY_DOUBLE` 现在是 `Double` 的子类，而用于 asyncpg 和 pg8000 的内部类型现在正确地是 `Float` 的子类。

+   **[sql] [bug]**

    修复了包含多个表且没有 VALUES 子句的 `update()` 构造会引发内部错误的问题。当前对于没有值的 `Update` 的行为是生成一个带有空“set”子句的 SQL UPDATE 语句，因此对于这个特定子情况已经保持一致。

### schema

+   **[schema] [performance]**

    改进了添加表列的方式，避免了不必要的分配，显著加快了创建许多表的速度，比如在反射整个模式时。

    参考：[#9597](https://www.sqlalchemy.org/trac/ticket/9597)

### typing

+   **[typing] [bug]**

    修复了 `Session.get()` 和 `Session.refresh()`（以及 `AsyncSession` 上的相应方法）的 `Session.get.with_for_update` 参数的类型，以接受布尔值 `True` 和运行时参数接受的所有其他形式。

    参考：[#9762](https://www.sqlalchemy.org/trac/ticket/9762)

+   **[typing] [sql]**

    添加了类型 `ColumnExpressionArgument` 作为一个公共类型，指示传递给 SQLAlchemy 构造的面向列的参数，例如 `Select.where()`、`and_()` 等。这可以用于为调用这些方法的最终用户函数添加类型。

    参考：[#9656](https://www.sqlalchemy.org/trac/ticket/9656)

### asyncio

+   **[asyncio] [usecase]**

    添加了一个新的辅助混合类 `AsyncAttrs`，旨在改进使用 asyncio 的延迟加载器和其他已过期或延迟的 ORM 属性，提供一个简单的属性访问器，为任何 ORM 属性提供一个 `await` 接口，无论它是否需要发出 SQL。

    另请参阅

    `AsyncAttrs`

    参考：[#9731](https://www.sqlalchemy.org/trac/ticket/9731)

+   **[asyncio] [bug]**

    修复了半私有的 `await_only()` 和 `await_fallback()` 并发函数中的问题，如果函数抛出 `GreenletError`，则给定的可等待对象将保持未等待状态，这可能会导致程序后续出现“未等待”警告。在这种情况下，在抛出异常之前，给定的可等待对象现在将被取消。

### postgresql

+   **[postgresql] [bug] [regression]**

    修复了 2.0.10 中“insertmanyvalues”更改导致的另一个回归问题，类似于回归问题[#9701](https://www.sqlalchemy.org/trac/ticket/9701)，在使用 asyncpg 驱动程序时，`LargeBinary` 数据类型也需要额外的转换以便与新的批量插入格式一起使用。

    参考：[#9739](https://www.sqlalchemy.org/trac/ticket/9739)

### oracle

+   **[oracle] [reflection]**

    在 Oracle 方言中为基于表达式的索引和索引表达式的排序方向添加了反射支持。

    参考：[#9597](https://www.sqlalchemy.org/trac/ticket/9597)

### 杂项

+   **[bug] [ext]**

    修复了`Mutable`中的问题，其中为 ORM 映射属性注册事件会在映射继承子类中重复调用，导致在继承层次结构中调用重复事件。

    参考：[#9676](https://www.sqlalchemy.org/trac/ticket/9676)

## 2.0.12

发布日期：2023 年 4 月 30 日

### orm

+   **[orm] [bug]**

    修复了严重的缓存问题，其中`aliased()`和`hybrid_property()`表达式组合的组合会导致缓存键不匹配，导致缓存键持有实际`aliased()`对象，同时又不匹配相等结构的缓存键，填充了缓存。

    此更改也已**回溯**到：1.4.48

    参考：[#9728](https://www.sqlalchemy.org/trac/ticket/9728)

### mysql

+   **[mysql] [bug] [mariadb]**

    修复了关于`Table`和`Column`对象的注释反射问题，其中注释包含控制字符，如换行符。对于这些字符以及扩展的 Unicode 字符在表和列注释中的测试支持也已添加到总体测试中。

    参考：[#9722](https://www.sqlalchemy.org/trac/ticket/9722)

## 2.0.11

发布日期：2023 年 4 月 26 日

### orm

+   **[orm] [用例]**

    ORM 批量 INSERT 和 UPDATE 现在添加了这些功能：

    +   使用“orm” dml_strategy 设置时，不再需要传递额外参数的要求。

    +   使用 ORM UPDATE 时，不再需要满足“bulk” dml_strategy 设置时不传递额外的 WHERE 条件的要求。注意，在这种情况下，预期行数的检查被关闭了。

    参考：[#9583](https://www.sqlalchemy.org/trac/ticket/9583), [#9595](https://www.sqlalchemy.org/trac/ticket/9595)

+   **[orm] [bug]**

    修复了 2.0 中的回归问题，其中在使用 ORM `Session` 执行 `Insert` 语句时，在 `Insert.values()` 中使用 `bindparam()` 将无法被正确解释，因为新的 ORM 启用的插入功能 没有实现这种用例。

    参考：[#9583](https://www.sqlalchemy.org/trac/ticket/9583), [#9595](https://www.sqlalchemy.org/trac/ticket/9595)

### engine

+   **[engine] [performance]**

    对`Row`进行了一系列性能增强：

    +   `__getattr__` 方法在行的“命名元组”接口的性能得到了提升；在这个变化中，`Row` 的实现已经简化，移除了在 1.4 版本及以前的 SQLAlchemy 中特有的构造和逻辑。作为这一变化的一部分，`Row` 的序列化格式已经略微修改，然而使用之前的 SQLAlchemy 2.0 版本进行 pickle 的行将在新格式中被识别。Pull request 由 J. Nick Koston 提供。

    +   通过使“bytes”处理程序在每个驱动程序基础上具有条件性，改进了“二进制”数据类型的行处理性能。因此，除了 psycopg2 之外的几乎所有现代形式的驱动程序都已删除了“bytes”结果处理程序，它们都支持直接返回 Python “bytes”。Pull request 由 J. Nick Koston 提供。

    +   在 `Row` 内部进行了进一步的重构，以提高性能。由 Federico Caselli 提供。

    引用：[#9678](https://www.sqlalchemy.org/trac/ticket/9678)、[#9680](https://www.sqlalchemy.org/trac/ticket/9680)

+   **[engine] [bug] [regression]**

    修复了阻止 `URL.normalized_query` 属性正常运行的回归问题。

    引用：[#9682](https://www.sqlalchemy.org/trac/ticket/9682)

### sql

+   **[sql] [usecase]**

    增加了对 `ColumnCollection` 的切片访问支持，例如 `table.c[0:5]`、`subquery.c[:-1]` 等。切片访问返回一个子 `ColumnCollection`，方式与传递键元组相同。这是对 [#8285](https://www.sqlalchemy.org/trac/ticket/8285) 添加的键元组访问的自然延续，其中切片访问用例被省略似乎是一个疏忽。

    引用：[#8285](https://www.sqlalchemy.org/trac/ticket/8285)

### typing

+   **[typing] [bug]**

    改进了对 `RowMapping` 的类型标注，以表明它还支持 `Column` 作为索引对象，而不仅仅是字符串名称。Pull request 由 Andy Freeland 提供。

    引用：[#9644](https://www.sqlalchemy.org/trac/ticket/9644)

### postgresql

+   **[postgresql] [bug] [regression]**

    修复了由[#9618](https://www.sqlalchemy.org/trac/ticket/9618)引起的严重回归，该回归修改了 2.0.10 的 insertmanyvalues 功能的架构，导致使用 psycopg2 或 psycopg 驱动程序使用 insertmanyvalues 功能插入时，浮点值失去所有小数位。

    参考：[#9701](https://www.sqlalchemy.org/trac/ticket/9701)

### mssql

+   **[mssql] [bug]**

    为 SQL Server 实现了`Double`类型，在 DDL 时将渲染`DOUBLE PRECISION`。这是使用新的 MSSQL 数据类型`DOUBLE_PRECISION`实现的，也可以直接使用。

### oracle

+   **[oracle] [bug]**

    修复了 Oracle 方言中的问题，例如在使用`Insert.returning()`子句返回 INSERT 的值时，`Decimal`返回类型（例如`Numeric`）会返回浮点值，而不是`Decimal`对象的问题。

## 2.0.10

发布日期：2023 年 4 月 21 日

### orm

+   **[orm] [bug]**

    修复了各种 ORM 特定的获取器，例如`ORMExecuteState.is_column_load`，`ORMExecuteState.is_relationship_load`，`ORMExecuteState.loader_strategy_path` 等，如果 SQL 语句本身是“复合选择”，如 UNION，则会抛出`AttributeError`。

    此更改还**回溯**到：1.4.48

    参考：[#9634](https://www.sqlalchemy.org/trac/ticket/9634)

+   **[orm] [bug]**

    修复了当应用于`__mapper_args__`特殊方法名时，`declared_attr.directive()`修改器未正确地应用于子类的问题，与直接使用`declared_attr`相反。这两种构造在运行时行为应该相同。

    参考：[#9625](https://www.sqlalchemy.org/trac/ticket/9625)

+   **[orm] [bug]**

    对 `with_loader_criteria()` 加载器选项进行了改进，允许在不是 ORM 语句本身的顶级语句的 `Executable.options()` 方法中指示它。示例包括嵌入在诸如 `union()` 的复合语句中的 `select()`，在 `Insert.from_select()` 构造中，以及在不是 ORM 相关的顶级 CTE 表达式中。

    参考：[#9635](https://www.sqlalchemy.org/trac/ticket/9635)

+   **[orm] [bug]**

    修复了 ORM 批量插入功能中的错误，如果请求返回单独列，则在 INSERT 语句中会渲染出额外的不必要列。

    参考：[#9685](https://www.sqlalchemy.org/trac/ticket/9685)

+   **[orm] [bug]**

    在 ORM 声明式数据类中修复了一个错误，`query_expression()` 和 `column_property()` 构造被记录为只读构造，不能在没有添加 `init=False` 的情况下与 `MappedAsDataclass` 类一起使用，而在 `query_expression()` 中没有可能添加 `init` 参数，因此这个问题不可避免。这些构造已从数据类的角度进行了修改，假定为“只读”，默认设置 `init=False`，并不再包含在 pep-681 构造函数中。`column_property()` 的数据类参数 `init`、`default`、`default_factory`、`kw_only` 现已弃用；这些字段不适用于在 Declarative 数据类配置中使用的 `column_property()`，因为该构造将是只读的。还为 `query_expression()` 添加了一个特定于读取的参数 `query_expression.compare`；`query_expression.repr` 已经存在。

    参考：[#9628](https://www.sqlalchemy.org/trac/ticket/9628)

+   **[orm] [错误]**

    向 `mapped_column()` 构造添加了缺失的 `mapped_column.active_history` 参数。

### 引擎

+   **[引擎] [用例]**

    添加了 `create_pool_from_url()` 和 `create_async_pool_from_url()` 来自字符串或 `URL` 传入的输入 url 创建 `Pool` 实例。

    参考：[#9613](https://www.sqlalchemy.org/trac/ticket/9613)

+   **[引擎] [错误]**

    修复了在 2.0 系列首次引入的 “Insert Many Values” Behavior for INSERT statements 性能优化功能中发现的一个主要缺陷。这是对 2.0.9 中的更改的延续，该更改禁用了 SQL Server 版本的功能，因为 ORM 依赖于似乎不保证发生的行排序。该修复为所有“insertmanyvalues”操作应用了新逻辑，当在 `Insert.returning()` 或 `UpdateBase.return_defaults()` 方法上设置了一个新参数 `Insert.returning.sort_by_parameter_order` 时生效，通过一种交替的 SQL 形式、客户端参数的直接对应以及在某些情况下降级到逐行运行，将对每个返回行批次应用与主键或其他唯一值的对应关系，这些值可以与输入数据相关联。

    预计性能影响将是最小的，因为几乎所有常见的主键场景都适合于在除了 SQLite 之外的所有后端实现参数排序批处理，而“逐行”模式与 1.x 系列中使用的非常沉重的方法相比，具有最少的 Python 开销。对于 SQLite，当使用“逐行”模式时，性能没有区别。

    预计通过具有高效的“逐行”INSERT 与 RETURNING 批处理功能，后续可以更容易地将“insertmanyvalues”功能推广到包括 RETURNING 支持但不一定易于保证与参数顺序对应的第三方后端。

    另请参阅

    将返回的行与参数集相关联

    参考：[#9603](https://www.sqlalchemy.org/trac/ticket/9603), [#9618](https://www.sqlalchemy.org/trac/ticket/9618)

### typing

+   **[typing] [bug]**

    为最近添加的运算符 `ColumnOperators.icontains()`、`ColumnOperators.istartswith()`、`ColumnOperators.iendswith()` 和位运算符 `ColumnOperators.bitwise_and()`、`ColumnOperators.bitwise_or()`、`ColumnOperators.bitwise_xor()`、`ColumnOperators.bitwise_not()`、`ColumnOperators.bitwise_lshift()` 和 `ColumnOperators.bitwise_rshift()` 添加了类型信息。感谢 Martijn Pieters 的拉取请求。

    参考：[#9650](https://www.sqlalchemy.org/trac/ticket/9650)

+   **[typing] [bug]**

    对代码库进行更新，以通过 Mypy 1.2.0 的类型检查。

+   **[typing] [bug]**

    修复了在加载器选项中（如 `selectinload()`）中，`PropComparator.and_()` 表达式未正确类型化的问题。

    参考：[#9669](https://www.sqlalchemy.org/trac/ticket/9669)

### postgresql

+   **[postgresql] [usecase]**

    在 asyncpg 方言中添加了 `prepared_statement_name_func` 连接参数选项。此选项允许传递一个可调用对象，用于自定义执行查询时驱动程序将创建的准备语句的名称。感谢 Pavel Sirotkin 的拉取请求。

    另请参阅

    使用 PGBouncer 的预准备语句名称

    参考：[#9608](https://www.sqlalchemy.org/trac/ticket/9608)

+   **[postgresql] [usecase]**

    添加了缺失的 `Range.intersection()` 方法。感谢 Yurii Karabas 的拉取请求。

    参考：[#9509](https://www.sqlalchemy.org/trac/ticket/9509)

+   **[postgresql] [bug]**

    在 `ENUM` 的签名中，恢复了 `ENUM.name` 参数作为可选参数，因为这是根据给定的 pep-435 `Enum` 类型自动选择的。

    参考：[#9611](https://www.sqlalchemy.org/trac/ticket/9611)

+   **[postgresql] [bug]**

    修复了对普通字符串进行 `ENUM` 比较时将右侧类型转换为 VARCHAR 的问题，这是由于在诸如 asyncpg 等方言中添加了更明确的转换而导致的 PostgreSQL 类型不匹配错误。

    参考：[#9621](https://www.sqlalchemy.org/trac/ticket/9621)

+   **[postgresql] [bug]**

    修复了在 PostgreSQL 中无法反射基于表达式的长表达式索引的问题。表达式错误地被截断为标识符长度（默认为 63 字节）。

    参考：[#9615](https://www.sqlalchemy.org/trac/ticket/9615)

### mssql

+   **[mssql] [bug]**

    恢复了 Microsoft SQL Server 的 insertmanyvalues 功能。该功能在版本 2.0.9 中被禁用，因为似乎依赖于 RETURNING 的排序，这是不被保证的。"insertmanyvalues" 功能的架构已经重做，以适应 INSERT 语句的特定组织和可以保证返回行与输入记录对应的结果行处理。

    参见

    将 RETURNING 行与参数集对应

    参考：[#9603](https://www.sqlalchemy.org/trac/ticket/9603), [#9618](https://www.sqlalchemy.org/trac/ticket/9618)

### oracle

+   **[oracle] [bug]**

    修复了在 Oracle 方言的 INSERT..RETURNING 子句中无法使用 `Uuid` 数据类型的问题。

## 2.0.9

发布日期：2023 年 4 月 5 日

### orm

+   **[orm] [bug]**

    修复了当使用“关联到别名类”的特性并且在加载器中指定了一个递归的 eager 加载器（如 `lazy="selectinload"`）与另一个相对端的另一个 eager 加载器结合使用时可能发生的无限循环问题。循环检查已修复以包括别名类关系。

    这个变更也被 **反向移植** 到了：1.4.48

    参考：[#9590](https://www.sqlalchemy.org/trac/ticket/9590)

### mariadb

+   **[mariadb] [bug]**

    在 MariaDb 中将 `row_number` 添加为保留字。

    参考：[#9588](https://www.sqlalchemy.org/trac/ticket/9588)

### mssql

+   **[mssql] [bug]**

    SQLAlchemy 的“insertmanyvalues”功能允许快速插入许多行，同时还支持 RETURNING，目前已暂时禁用了 SQL Server。由于工作单元当前依赖于此功能，以便将现有 ORM 对象匹配到返回的主键标识，因此此特定使用模式在某些情况下无法与 SQL Server 一起使用，因为“OUTPUT inserted” 返回的行的顺序可能并不总是与发送元组的顺序匹配，导致 ORM 在后续操作中对这些对象做出错误决策。

    此功能将在即将发布的版本中重新启用，并且将再次对多行 INSERT 语句产生影响，但是工作单元对此功能的使用将被禁用，可能对所有方言都禁用，除非 ORM 映射的表还包括一个“sentinel”列，以便可以将返回的行引用回传递的原始数据。

    参考：[#9603](https://www.sqlalchemy.org/trac/ticket/9603)

+   **[mssql] [bug]**

    当 `fast_executemany` 设置为 `True` 时，已更改用于 SQL Server 的批量 INSERT 策略“executemany”与 pyodbc，使用 `fast_executemany` / `cursor.executemany()` 用于不包含 RETURNING 的批量 INSERT，当此参数设置时，恢复了与 SQLAlchemy 1.4 中使用的相同行为。

    最新的终端用户性能细节显示，对于非常大的数据集，`fast_executemany` 仍然比较快，因为它使用可以在单次往返中接收所有行的 ODBC 命令，允许比“insertmanyvalues”发送的批次更大得多的数据大小，后者已为 SQL Server 实现。

    尽管此更改使得“insertmanyvalues”仍然被用于包含 RETURNING 的 INSERT，并且如果未设置 `fast_executemany`，由于[#9603](https://www.sqlalchemy.org/trac/ticket/9603)，在任何情况下，“insertmanyvalues” 策略已被完全禁用于 SQL Server。

    参考：[#9586](https://www.sqlalchemy.org/trac/ticket/9586)

## 2.0.8

发布日期：2023 年 3 月 31 日

### orm

+   **[orm] [usecase]**

    当使用 `MappedAsDataclass` 混合类或 `registry.mapped_as_dataclass()` 装饰器时，Python 数据类引发的诸如 `TypeError` 和 `ValueError` 等异常现在将在一个 `InvalidRequestError` 包装器中进行包装，其中包含有关错误消息的信息性上下文，参考 Python 数据类文档作为异常原因的官方来源。

    参见

    创建<类名>数据类时遇到的 Python 数据类错误

    参考：[#9563](https://www.sqlalchemy.org/trac/ticket/9563)

+   **[orm] [bug]**

    修复了 ORM 注释声明中的问题，其中使用递归类型（例如使用嵌套的字典类型）会导致 ORM 的注释解析逻辑中发生递归溢出，即使这种数据类型不是必要的来映射列。

    参考：[#9553](https://www.sqlalchemy.org/trac/ticket/9553)

+   **[orm] [bug]**

    修复了在 Declarative mixin 上使用`mapped_column()` 构造时，如果包含`mapped_column.deferred` 参数会引发内部错误的问题。

    参考：[#9550](https://www.sqlalchemy.org/trac/ticket/9550)

+   **[orm] [bug]**

    扩展了在声明式映射中存在普通`column()`对象时发出的警告，以包括任何未在适当属性类型内声明的任意 SQL 表达式，例如`column_property()`、`deferred()`等。这些属性在类字典中保持不变且未映射。由于这种表达式通常不是预期的内容，因此现在对所有这些否则被忽略的表达式发出警告，而不仅仅是`column()`的情况。

    参考：[#9537](https://www.sqlalchemy.org/trac/ticket/9537)

+   **[orm] [bug]**

    修复了在访问一个混合属性的表达式值时出现的回归问题，该属性位于一个未映射或尚未映射的类上（例如在`declared_attr()`方法中调用它），会引发内部错误，因为对父类映射器的内部获取将失败，并且对于此失败的指令被无意中在 2.0 中删除。

    参考：[#9519](https://www.sqlalchemy.org/trac/ticket/9519)

+   **[orm] [bug]**

    在声明式 Mixins 上声明的字段，然后与使用`MappedAsDataclass`的类组合在一起，其中这些 Mixin 字段本身不是数据类的一部分，现在会发出弃用警告，因为这些字段将在将来的版本中被忽略，因为 Python 数据类的行为是忽略这些字段。类型检查器在 pep-681 下不会看到这些字段。

    另请参阅

    当将<cls>转换为数据类时，属性(s)源自不是数据类的超类<cls>。 - 关于背景的理由

    使用 Mixins 和抽象超类

    参考：[#9350](https://www.sqlalchemy.org/trac/ticket/9350)

+   **[orm] [bug]**

    修复了一个问题，其中 `BindParameter.render_literal_execute()` 方法在调用带有 ORM 注释的参数时会失败。在实践中，当使用一些类似于 Oracle 的使用“FETCH FIRST”的方言以及使用 `Select` 构造的 `Select.limit()` 的组合时，在一些 ORM 上下文中会观察到这种情况，包括如果该语句嵌入在关系 primaryjoin 表达式中时，SQL 编译失败。

    参考：[#9526](https://www.sqlalchemy.org/trac/ticket/9526)

+   **[orm] [bug]**

    为了与为 [#5984](https://www.sqlalchemy.org/trac/ticket/5984) 和 [#8862](https://www.sqlalchemy.org/trac/ticket/8862) 所做的工作单元一致性保持一致，这两者都禁用了在 `Session` 进程中的“惰性='raise'”处理，这些处理并非由属性访问触发，`Session.delete()` 方法现在在遍历关系路径以处理“delete”和“delete-orphan”级联规则时，也将禁用“惰性='raise'”处理。以前，没有简单的方法可以通用地调用 `Session.delete()` 在设置了“惰性='raise'”的对象上，以便只加载必要的关系。由于“惰性='raise'”主要用于捕获在属性访问时发出的 SQL 加载，因此 `Session.delete()` 现在被制作成像其他 `Session` 方法一样，包括 `Session.merge()` 以及 `Session.flush()` 以及 autoflush。

    参考：[#9549](https://www.sqlalchemy.org/trac/ticket/9549)

+   **[orm] [bug]**

    修复了一个问题，其中仅注释的 `Mapped` 指令无法在声明性混合类中使用，而不会尝试让该属性对已经映射了该属性的超类的单个或联合继承子类产生影响，从而产生冲突的列错误和/或警告。

    参考：[#9564](https://www.sqlalchemy.org/trac/ticket/9564)

+   **[orm] [bug] [typing]**

    适当地对`Insert.from_select.names`进行类型定义，以接受字符串列表或列或映射属性。

    参考：[#9514](https://www.sqlalchemy.org/trac/ticket/9514)

### 示例

+   **[examples] [bug]**

    修复了“版本历史”示例中的问题，使用从`DeclarativeBase`派生的声明性基类会导致映射失败的问题。此外，修复了给定的测试套件，以便通过 Python unittest 运行示例的文档说明现在再次有效。

### 类型

+   **[typing] [bug]**

    修复了`deferred()`和`query_expression()`在与 2.0 样式映射正确配合使用时的类型。

    参考：[#9536](https://www.sqlalchemy.org/trac/ticket/9536)

### postgresql

+   **[postgresql] [bug]**

    修复了 PostgreSQL 方言中的关键回归问题，例如 asyncpg 依赖于 SQL 中的显式转换，以便将数据类型正确传递给驱动程序，其中`String`数据类型将与要比较的确切列长度一起转换，导致在比较较小长度的`VARCHAR`与较大长度的字符串时进行隐式截断，而不管使用的操作符是什么（例如 LIKE，MATCH 等）。 PostgreSQL 方言现在在呈现这些转换时省略`VARCHAR`的长度。

    参考：[#9511](https://www.sqlalchemy.org/trac/ticket/9511)

### mysql

+   **[mysql] [bug]**

    修复了字符串数据类型（如`CHAR`、`VARCHAR`、`TEXT`），以及二进制`BLOB`无法使用零长度明确生成的问题，这在 MySQL 中具有特殊含义。感谢 J. Nick Koston 提出的拉取请求。

    参考：[#9544](https://www.sqlalchemy.org/trac/ticket/9544)

### 杂项

+   **[bug] [util]**

    在 OrderedSet 类中实现了缺失的`copy`和`pop`方法。

    参考：[#9487](https://www.sqlalchemy.org/trac/ticket/9487)

## 2.0.7

发布日期：2023 年 3 月 18 日

### 类型

+   **[typing] [bug]**

    修复了`composite()`不允许任意可调用对象作为复合类来源的类型问题。

    参考：[#9502](https://www.sqlalchemy.org/trac/ticket/9502)

### postgresql

+   **[postgresql] [usecase]**

    添加了新的 PostgreSQL 类型 `CITEXT`。拉取请求由 Julian David Rath 提供。

    参考：[#9416](https://www.sqlalchemy.org/trac/ticket/9416)

+   **[postgresql] [用例]**

    修改了基本 PostgreSQL 方言，以更好地与 SQLAlchemy 2.0 的第三方方言 sqlalchemy-redshift 进行集成。拉取请求由 matthewgdv 提供。

    参考：[#9442](https://www.sqlalchemy.org/trac/ticket/9442)

## 2.0.6

发布日期：2023 年 3 月 13 日

### orm

+   **[orm] [错误]**

    修复了“活动历史记录”功能对复合属性未完全实现的错误，这使得无法接收包含“旧”值的事件成为可能。这似乎也是旧的 SQLAlchemy 版本的情况，其中“active_history”将传播到基础基于列的属性，但是即使设置了 active_history=True 的复合()，监听复合属性本身的事件处理程序也不会收到被替换的“旧”值。

    另外，修复了一个局限于 2.0 的回归，该回归禁止了对复合的 active_history 被分配到带有 `attr.impl.active_history=True` 的实现。

    参考：[#9460](https://www.sqlalchemy.org/trac/ticket/9460)

+   **[orm] [错误]**

    修复了在为版本 2.0 重构代码时，涉及 Python 行的 pickling 在 cython 和纯 Python 实现之间发生的回归，这发生在对 `Row` 进行了带有类型信息的代码重构时。对于纯 Python 版本的 `Row`，一个特定的常量被转换为基于字符串的 `Enum`，而 cython 版本继续使用整数常量，导致反序列化失败。

    参考：[#9418](https://www.sqlalchemy.org/trac/ticket/9418)

### sql

+   **[sql] [错误] [回归]**

    修复了在 1.4 系列中发布的用于 lambda SQL API 的一层并发安全检查的修复引起的回归，该修复针对的是[#8098](https://www.sqlalchemy.org/trac/ticket/8098)，在补丁中包含了未能应用到主分支的额外修复。这些额外的修复已经应用。

    参考：[#9461](https://www.sqlalchemy.org/trac/ticket/9461)

+   **[sql] [错误]**

    修复了在 `select()` 构造中，如果没有给定列而然后在 EXISTS 的上下文中使用，则无法呈现的回归，而是引发了内部异常。虽然一个空的“SELECT”通常不是有效的 SQL，但在 EXISTS 数据库中（例如 PostgreSQL）允许它，在任何情况下，该条件现在不再引发内部异常。

    参考：[#9440](https://www.sqlalchemy.org/trac/ticket/9440)

### 打字

+   **[打字] [错误]**

    修复了 `ColumnElement.cast()` 中的类型问题，它不允许独立于 `ColumnElement` 本身的类型引擎参数，而 `ColumnElement.cast()` 的目的就是这样。

    参考：[#9451](https://www.sqlalchemy.org/trac/ticket/9451)

+   **[typing] [bug]**

    修复问题以允许在 Mypy 1.1.1 下通过类型测试。

### oracle

+   **[oracle] [bug]**

    修复了 Oracle “名称标准化”在反射 `"PUBLIC"` 模式下无法正确工作的反射错误，例如在 Python 端不能将 PUBLIC 名称指定为小写用于 `Table.schema` 参数。使用大写的 `"PUBLIC"` 将起作用，但会导致包括引号的 SQL 查询以及在大写 `"PUBLIC"` 下索引表，这是不一致的。

    参考：[#9459](https://www.sqlalchemy.org/trac/ticket/9459)

## 2.0.5.post1

发布日期：2023 年 3 月 5 日

### orm

+   **[orm] [bug]**

    向内置映射集合类型添加构造函数参数，包括 `KeyFuncDict`、`attribute_keyed_dict()`、`column_keyed_dict()` 等，以便可以根据提供的数据即时构造这些字典类型；这进一步与诸如 Python 数据类 `.asdict()` 等工具兼容，后者依赖于直接调用这些类作为普通字典类。

    参考：[#9418](https://www.sqlalchemy.org/trac/ticket/9418)

+   **[orm] [bug] [regression]**

    由于 [#8372](https://www.sqlalchemy.org/trac/ticket/8372) 引起的多个退化已修复，涉及 `attribute_mapped_collection()`（现在称为 `attribute_keyed_dict()`）。

    首先，收集现在不再可用于具有不是普通映射属性的“键”属性；已修复了与描述符和/或关联代理属性相关的属性。

    其次，如果事件或其他操作需要访问“key”以便从未加载的映射属性填充字典，那么这也会不适当地引发错误，而不是像 1.4 版本中那样尝试加载属性。这个问题也已经修复。

    对于这两种情况，扩展了[#8372](https://www.sqlalchemy.org/trac/ticket/8372)的行为。[#8372](https://www.sqlalchemy.org/trac/ticket/8372)引入了一个错误，当作为映射字典键使用的派生键实际上未被赋值时会引发错误。在此更改中，仅在“.key”属性的有效值为`None`时才发出警告，无法明确确定这个`None`是否是有意的。`None`将不再作为映射集合字典键的支持（因为它通常指的是 NULL，表示“未知”）。设置`attribute_keyed_dict.ignore_unpopulated_attribute`现在也将导致忽略这样的`None`键。

    参考：[#9424](https://www.sqlalchemy.org/trac/ticket/9424)

+   **[orm] [bug]**

    确定了`sqlite`和`mssql+pyodbc`方言现在与 SQLAlchemy ORM 的“versioned rows”功能兼容，因为 SQLAlchemy 现在通过计算返回的行数来计算 RETURNING 语句的行数，而不是依赖于`cursor.rowcount`。特别是，ORM 版本的行用例（在配置版本计数器文档中有描述）现在应该完全支持与 SQL Server pyodbc 方言一起使用。

+   **[orm] [bug]**

    添加了对`Mapper.polymorphic_load`参数的支持，该参数应用于继承层次结构中超过一级的每个映射器，允许通过单个语句为层次结构中的所有类加载列，这些列指示使用 `"selectin"`，而不是忽略那些中间类上的元素，尽管它们也指示它们将参与 `"selectin"` 加载，但它们不是基本的 SELECT 语句的一部分。

    参考：[#9373](https://www.sqlalchemy.org/trac/ticket/9373)

+   **[orm] [bug]**

    继续修复 [#8853](https://www.sqlalchemy.org/trac/ticket/8853)，允许 `Mapped` 名称完全合格，无论是否存在 `from __annotations__ import future`。此问题首次在 2.0.0b3 中修复，确认此情况通过测试套件工作，但是测试套件显然没有测试名称 `Mapped` 完全不存在的行为；字符串解析已更新以确保 ORM 如何使用这些函数。

    参考：[#8853](https://www.sqlalchemy.org/trac/ticket/8853), [#9335](https://www.sqlalchemy.org/trac/ticket/9335)

### orm 声明式

+   **[bug] [orm declarative]**

    修复了新 `mapped_column.use_existing_column` 功能无法工作的问题，如果两个同名列被映射到与列本身的显式名称不同的属性名下。现在，当使用此参数时，属性名称可以不同。 

    参考：[#9332](https://www.sqlalchemy.org/trac/ticket/9332)

### 引擎

+   **[engine] [performance]**

    对 `Result` 的 Cython 实现进行了小优化，使用了一个特定 int 值的 cdef 来避免 Python 开销。拉取请求由 Matus Valo 提供。

    参考：[#9343](https://www.sqlalchemy.org/trac/ticket/9343)

+   **[engine] [bug]**

    修复了 `Row` 对象由于意外依赖不稳定的哈希值而无法在进程间可靠地反序列化的 bug。

    参考：[#9423](https://www.sqlalchemy.org/trac/ticket/9423)

### SQL

+   **[sql] [bug] [regression]**

    将 `nullslast()` 和 `nullsfirst()` 旧版本函数恢复到 `sqlalchemy` 导入命名空间中。之前，较新的 `nulls_last()` 和 `nulls_first()` 函数是可用的，但是旧版本函数不小心被移除了。

    参考：[#9390](https://www.sqlalchemy.org/trac/ticket/9390)

### 模式

+   **[schema]**

    验证当提供给`MetaData`的`MetaData.schema`参数时，它是一个字符串。

### typing

+   **[typing] [usecase]**

    导出了由`scoped_session.query_property()`返回的类型，使用了一个新的公共类型`QueryPropertyDescriptor`。

    参考：[#9338](https://www.sqlalchemy.org/trac/ticket/9338)

+   **[typing] [bug]**

    修复了`Connection.scalars()`方法未被标记为允许多参数列表的错误，现在支持使用`insertmanyvalues`操作。

+   **[typing] [bug]**

    对传递给`Insert.values()`和`Update.values()`的映射的类型进行了改进，使之更加开放，指示只读`Mapping`而不是可写`Dict`，后者在键类型过于有限时会出错。

    参考：[#9376](https://www.sqlalchemy.org/trac/ticket/9376)

+   **[typing] [bug]**

    在`Numeric`类型对象中添加了缺失的初始化重载，以便 pep-484 类型检查器可以正确解析完整的类型，从`Numeric.asdecimal`参数派生，确定将表示`Decimal`还是`float`对象。

    参考：[#9391](https://www.sqlalchemy.org/trac/ticket/9391)

+   **[typing] [bug]**

    修复了类型错误，`Select.from_statement()`将不接受`text()`或`TextualSelect`对象作为有效类型的 bug。此外，修复了`columns`方法的返回类型缺失的问题。

    参考：[#9398](https://www.sqlalchemy.org/trac/ticket/9398)

+   **[typing] [bug]**

    修复了`with_polymorphic()`未正确记录类类型的类型问题。

    参考：[#9340](https://www.sqlalchemy.org/trac/ticket/9340)

### postgresql

+   **[postgresql] [bug]**

    修复了在 PostgreSQL `ExcludeConstraint` 中的问题，其中文字值被编译为绑定参数而不是直接的内联值，这是 DDL 所必需的。

    参考：[#9349](https://www.sqlalchemy.org/trac/ticket/9349)

+   **[postgresql] [bug]**

    修复了 PostgreSQL `ExcludeConstraint` 构造中的问题，如果约束包含文本表达式元素，则在 `Table.to_metadata()` 等操作中以及在一些 Alembic 方案中无法复制。

    参考：[#9401](https://www.sqlalchemy.org/trac/ticket/9401)

### mysql

+   **[mysql] [bug] [postgresql]**

    为了解决在 2.0.0b1 中为 [#5648](https://www.sqlalchemy.org/trac/ticket/5648) 添加的池 ping 监听器通过 `DialectEvents.handle_error()` 事件接收异常事件的支持未考虑到诸如 MySQL 和 PostgreSQL 的特定方言的 ping 程序的问题。方言特性已重新设计，使得所有方言都参与事件处理。另外，添加了一个新的布尔元素 `ExceptionContext.is_pre_ping`，用于标识此操作是否在预 ping 操作中进行。

    对于此版本，实现自定义 `Dialect.do_ping()` 方法的第三方方言可以选择通过不再捕获异常或检查异常是否为“is_disconnect”，而是直接将所有异常传播出去来选择新的改进行为。现在由默认方言的一个包围方法来检查异常是否为“is_disconnect”，这确保了在测试异常是否为“断开连接”异常之前调用事件挂钩以处理所有异常情况。如果现有的 `do_ping()` 方法继续捕获异常并检查“is_disconnect”，则它将像以前一样工作，但是如果不将异常传播出去，`handle_error` 钩子将无法访问异常。

    参考：[#5648](https://www.sqlalchemy.org/trac/ticket/5648)

### sqlite

+   **[sqlite] [bug] [regression]**

    修复了在 SQLite 连接中的回归，其中在建立数据库函数时使用 `deterministic` 参数会导致旧版 SQLite 版本（3.8.3 之前的版本）失败。版本检查逻辑已经改进以适应此情况。

    参考：[#9379](https://www.sqlalchemy.org/trac/ticket/9379)

### mssql

+   **[mssql] [bug]**

    修复了新的 `Uuid` 数据类型中的问题，该问题导致它无法与 pymssql 驱动程序一起工作。由于 pymssql 似乎又开始维护，因此恢复了对 pymssql 的测试支持。

    参考：[#9414](https://www.sqlalchemy.org/trac/ticket/9414)

+   **[mssql] [bug]**

    调整了 pymssql 方言，以更好地利用 RETURNING 来获取 INSERT 语句的最后插入的主键值，与当前的 mssql+pyodbc 方言一样。

### 杂项

+   **[bug] [ext]**

    修复了在 automap 中的问题，调用 `AutomapBase.prepare()` 时，从特定映射类而不是直接从 `AutomapBase` 调用，当 automap 检测到新表时，不会使用正确的基类，而是使用给定的类，导致映射器尝试配置继承关系。虽然通常情况下应该从基类调用 `AutomapBase.prepare()`，但在从子类调用时不应该出现严重问题。

    参考：[#9367](https://www.sqlalchemy.org/trac/ticket/9367)

+   **[bug] [ext] [regression]**

    由于针对 [#8667](https://www.sqlalchemy.org/trac/ticket/8667) 添加的类型，导致了 `sqlalchemy.ext.mutable` 的回归错误，其中 `.pop()` 方法的语义发生了变化，使得该方法无法工作。感谢 Nils Philippsen 提交的拉取请求。

    参考：[#9380](https://www.sqlalchemy.org/trac/ticket/9380)

## 2.0.4

发布日期：2023 年 2 月 17 日

### orm

+   **[orm] [usecase]**

    为了适应 SQLAlchemy 2.0 中 ORM 声明式使用的列顺序的变化，新增了一个参数 `mapped_column.sort_order`，可用于控制 ORM 定义的表中列的顺序，适用于常见用例，如具有应首先出现在表中的主键列的混合类。变更说明在 ORM 声明式以不同方式应用列顺序；使用 sort_order 控制行为 中说明了默认的顺序变更行为（这是所有 SQLAlchemy 2.0 发行版的一部分），以及在使用混合类和多个类时使用 `mapped_column.sort_order` 控制列顺序的用法（2.0.4 中新增）。

    另请参阅

    ORM 声明式以不同方式应用列顺序；使用 sort_order 控制行为

    参考：[#9297](https://www.sqlalchemy.org/trac/ticket/9297)

+   **[orm] [usecase]**

    `Session.refresh()`方法现在将立即加载在`Session.refresh.attribute_names`集合中明确命名的与关系绑定的属性，即使它当前链接到“select”加载程序，通常是一个不会在刷新期间触发的“lazy”加载程序。 “懒加载器”策略现在将检测到操作明确是用户发起的`Session.refresh()`操作，该操作明确命名了此属性，然后将调用“immediateload”策略实际发出 SQL 以加载属性。这对于某些 asyncio 情况特别有帮助，其中必须强制加载未加载的惰性加载属性，而不使用实际不支持 asyncio 的惰性加载属性模式。

    参考：[#9298](https://www.sqlalchemy.org/trac/ticket/9298)

+   **[orm] [bug] [regression]**

    由于[#9217](https://www.sqlalchemy.org/trac/ticket/9217)引入的版本 2.0.2 中的回归错误已修复，其中使用 DML RETURNING 语句以及`Select.from_statement()`构造，如在[#9217](https://www.sqlalchemy.org/trac/ticket/9217)中“修复”的那样，与使用`column_property()`等表达式的 ORM 映射类一起使用，会导致 Core 内部错误，其中它会尝试按名称匹配表达式。修复了 Core 问题，并且还调整了[#9217](https://www.sqlalchemy.org/trac/ticket/9217)中的修复，以便不会对 DML RETURNING 用例产生影响，其中它增加了不必要的开销。

    参考：[#9273](https://www.sqlalchemy.org/trac/ticket/9273)

+   **[orm] [bug]**

    将内部`EvaluatorCompiler`模块标记为 ORM 私有，并将其重命名为`_EvaluatorCompiler`。对于可能依赖于此的用户，名称`EvaluatorCompiler`仍然存在，但不支持此用法，并将在将来的版本中删除。

### orm declarative

+   **[usecase] [orm declarative]**

    向`MappedAsDataclass`类和`registry.mapped_as_dataclass()`方法添加了新参数`dataclasses_callable`，允许使用替代的可调用对象来生成 Python `dataclasses.dataclass`。这里的用例是替换为 Pydantic 的 dataclass 函数。对版本 2.0.1 中为[#9179](https://www.sqlalchemy.org/trac/ticket/9179)添加的 mixin 支持进行了调整，以便将 mixin 的`__annotations__`集合重写，不包括`Mapped`容器，与映射类一样，以便 Pydantic dataclasses 构造函数不会暴露给未知类型。

    另请参阅

    与 Pydantic 等替代 Dataclass 提供者集成

    参考：[#9266](https://www.sqlalchemy.org/trac/ticket/9266)

### sql

+   **[sql] [bug]**

    修复了元组值的元素类型将被硬编码为从比较的元组中获取类型的问题，当比较使用`ColumnOperators.in_()`运算符时。这与通常确定二进制表达式类型的方式不一致，通常情况下会首先考虑右侧的实际元素类型，然后再应用左侧的类型。

    参考：[#9313](https://www.sqlalchemy.org/trac/ticket/9313)

+   **[sql]**

    添加了公共属性`Table.autoincrement_column`，该属性返回在列中标识为自增的列。

    参考：[#9277](https://www.sqlalchemy.org/trac/ticket/9277)

### typing

+   **[typing] [usecase]**

    改进了 Hybrid Attributes 扩展的类型支持，更新了所有文档以使用 ORM Annotated Declarative mappings，并添加了一个名为 `hybrid_property.inplace` 的新修改器。此修改器提供了一种改变 `hybrid_property` 的状态的方式，这在 SQLAlchemy 版本 1.2.0 之前的非常早期版本的混合属性中基本上是做的，版本 1.2.0 [#3912](https://www.sqlalchemy.org/trac/ticket/3912) 改变了这一点，删除了原地突变。现在，在**选择加入**的基础上恢复了这种原地突变，以允许单个混合具有多个设置的方法，无需命名所有方法相同，也无需仔细“链”不同命名的方法以维护组合。类型工具如 Mypy 和 Pyright 不允许在类上使用同名方法，因此通过此更改恢复了一种简洁的设置混合与类型支持的方法。

    另请参阅

    使用 inplace 创建符合 pep-484 标准的混合属性

    参考：[#9321](https://www.sqlalchemy.org/trac/ticket/9321)

### oracle

+   **[oracle] [bug]**

    调整了 `thick_mode` 参数的行为，以使 python-oracledb 方言正确接受 `False` 作为值。以前，只有 `None` 会表示应禁用 thick mode。

    参考：[#9295](https://www.sqlalchemy.org/trac/ticket/9295)

## 2.0.3

发布日期：2023 年 2 月 9 日

### sql

+   **[sql] [bug] [regression]**

    由于 [#7744](https://www.sqlalchemy.org/trac/ticket/7744)，在 2.0 系列中修复了 SQL 表达式制定的严重回归，改进了对包含许多相同操作符的 SQL 表达式的支持；表达式元素超过前两个元素后，括号分组将丢失。

    参考：[#9271](https://www.sqlalchemy.org/trac/ticket/9271)

### typing

+   **[typing] [bug]**

    删除了 `typing.Self` 的临时解决方案，现在使用了 [**PEP 673**](https://peps.python.org/pep-0673/) 来处理大多数返回 `Self` 的方法。由于这个变化，现在需要 `mypy>=1.0.0` 来对 SQLAlchemy 代码进行类型检查。感谢 Yurii Karabas 提供的拉取请求。

    参考：[#9254](https://www.sqlalchemy.org/trac/ticket/9254)

## 2.0.2

发布日期：2023 年 2 月 6 日

### orm

+   **[orm] [usecase]**

    添加了新的事件钩子`MapperEvents.after_mapper_constructed()`，提供了一个事件钩子，可以在`Mapper`对象完全构建完成后但在调用`registry.configure()`之前发生。这允许根据`Mapper`的初始配置创建额外映射和表结构的代码，也与声明性配置集成。以前，在使用声明性时，`Mapper`对象是在类创建过程中创建的，此时没有记录的方法来运行代码。这个改变立即使得自定义映射方案受益，比如带有历史表的版本控制示例，该示例根据映射类的创建生成额外的映射和表。

    参考：[#9220](https://www.sqlalchemy.org/trac/ticket/9220)

+   **[orm] [用例]**

    很少使用的`Mapper.iterate_properties`属性和`Mapper.get_property()`方法，主要用于内部，不再隐式调用`registry.configure()`过程。对这些方法的公开访问非常罕见，而拥有`registry.configure()`的唯一好处是允许这些集合中存在“backref”属性。为了支持新的`MapperEvents.after_mapper_constructed()`事件，现在可以迭代和访问内部的`MapperProperty`对象，而不会触发映射器本身的隐式配置。

    更公开的迭代所有映射属性的方式，`Mapper.attrs`集合等，仍会隐式调用`registry.configure()`步骤，从而使得反向引用属性可用。

    在所有情况下，`registry.configure()`始终可供直接调用。

    参考：[#9220](https://www.sqlalchemy.org/trac/ticket/9220)

+   **[ORM] [错误] [回归]**

    修复了由[#8705](https://www.sqlalchemy.org/trac/ticket/8705)引起的晦涩的 ORM 继承问题，其中一些从本地表和继承表一起指示列组的映射器的情况在`column_property()`下仍然会警告，即使隐式地组合了同名属性。

    参考：[#9232](https://www.sqlalchemy.org/trac/ticket/9232)

+   **[ORM] [错误] [回归]**

    修复了在使用具有常规 Python 端递增列的`Mapper.version_id_col`功能时，对于不支持“rowcount”和“RETURNING”的 SQLite 和其他数据库，将“RETURNING”用于这些列，即使实际上并非如此。

    参考：[#9228](https://www.sqlalchemy.org/trac/ticket/9228)

+   **[ORM] [错误] [回归]**

    修复了在 ORM 上下文中使用`Select.from_statement()`时的回归，其中仅基于名称匹配列到 SQL 标签的 ORM 语句被禁用，这将阻止具有列名标签的任意 SQL 表达式与要加载的实体匹配，以前在 1.4 和之前的系列中可以工作，因此已恢复了先前的行为。

    参考：[#9217](https://www.sqlalchemy.org/trac/ticket/9217)

### ORM 声明式

+   **[错误] [ORM 声明式]**

    修复了由于修复[#9171](https://www.sqlalchemy.org/trac/ticket/9171)引起的回归，该问题本身修复了涉及从`DeclarativeBase`继承的类的`__init__()`机制的回归。更改使得如果类上没有直接的`__init__()`方法，则`__init__()`将应用于用户定义的基类。现在已经调整为只有在用户定义的基类的层次结构中没有其他类具有`__init__()`方法时才应用`__init__()`。这再次允许基于`DeclarativeBase`的用户定义的基类包含包含自定义`__init__()`方法的混入。

    参考：[#9249](https://www.sqlalchemy.org/trac/ticket/9249)

+   **[错误] [ORM 声明式]**

    修复了与 2.0.1 中新增的对混合类支持相关的 ORM 声明式数据类映射中的问题，该问题通过[#9179](https://www.sqlalchemy.org/trac/ticket/9179)解决，其中在某些情况下使用混合类加上 ORM 继承会导致字段错误分类，导致字段级数据类参数（如`init=False`）丢失。

    参考：[#9226](https://www.sqlalchemy.org/trac/ticket/9226)

+   **[错误] [ORM 声明式]**

    修复了 ORM 声明式映射，允许在使用`mapped_column()`时在`__mapper_args__`中指定`Mapper.primary_key`参数。尽管这种用法直接在 2.0 文档中，但`Mapper`在这种情况下不接受`mapped_column()`构造。这个功能已经适用于`Mapper.version_id_col`和`Mapper.polymorphic_on`参数。

    作为这一变化的一部分，可以在非映射混合类上指定`__mapper_args__`属性，而无需使用`declared_attr()`，包括引用本地存在于混合类上的`Column`或`mapped_column()`对象的`"primary_key"`条目；声明式还将这些列转换为特定映射类的正确列。这在`Mapper.version_id_col`和`Mapper.polymorphic_on`参数中已经起���用。此外，`"primary_key"`中的元素可以指示为现有映射属性的字符串名称。

    参考：[#9240](https://www.sqlalchemy.org/trac/ticket/9240)

+   **[错误] [ORM 声明式]**

    如果映射尝试在同一类层次结构中混合使用`MappedAsDataclass`和`registry.mapped_as_dataclass()`，则会引发明确的错误，因为这会导致数据类函数在错误的时间应用于映射类，从而在映射过程中导致错误。

    参考：[#9211](https://www.sqlalchemy.org/trac/ticket/9211)

### 例子

+   **[examples] [bug]**

    重新设计了带有历史表的版本控制，以适用于版本 2.0，同时改进了此示例的整体工作，以使用更新的 API，包括新添加的钩子`MapperEvents.after_mapper_constructed()`。

    参考：[#9220](https://www.sqlalchemy.org/trac/ticket/9220)

### SQL

+   **[sql] [usecase]**

    添加了一套全新的 SQL 位运算符，用于在适当的数据值（如整数、位字符串等）上执行数据库端的位运算表达式。 拉取请求由 Yegor Statkevich 提供。

    另请参阅

    位运算符

    参考：[#8780](https://www.sqlalchemy.org/trac/ticket/8780)

### asyncio

+   **[asyncio] [bug]**

    修复了由于修复[#8419](https://www.sqlalchemy.org/trac/ticket/8419)而引起的回归，这导致了 asyncpg 连接被重置（即事务`rollback()`被调用），并且在连接未被显式返回到连接池并且被 Python 垃圾收集拦截时，正常返回到池中，如果垃圾收集操作在 asyncio 事件循环外被调用，则会失败，导致大量堆栈跟踪活动被转储到日志和标准输出中。

    恢复了正确的行为，即所有由于未被显式返回到连接池而被垃圾收集的 asyncio 连接都会从池中分离并且被丢弃，同时伴随着一条警告，而不是被返回到池中，因为它们无法可靠地重置。在 asyncpg 连接的情况下，将使用 asyncpg 特定的`terminate()`方法来更优雅地结束该连接，而不仅仅是将其丢弃。

    此更改包括了一个小的行为变更，希望对调试 asyncio 应用程序有用，其中在 asyncio 连接意外被垃圾收集时发出的警告已经通过将其移出`try/except`块并移到`finally:`块中而变得稍微更加激进，无论分离/终止操作是否成功，它都会无条件地发出。这也将使得将 Python 警告提升为异常的应用程序或测试套件会将此视为完整的异常抛出，而以前这个警告是不可能作为异常传播的。在此期间需要容忍此警告的应用程序和测试套件应该调整 Python 警告过滤器，以允许这些警告不会被提升为异常。

    传统同步连接的行为保持不变，即垃圾收集的连接继续正常返回到池中，而不会发出警告。在未来的主要发布版本中，这可能会发生变化，至少会像为 asyncio 驱动程序发出的类似警告一样发出警告，因为对于池化连接被垃圾收集拦截而未被正确返回到池中是一种使用错误。

    参考：[#9237](https://www.sqlalchemy.org/trac/ticket/9237)

### mysql

+   **[mysql] [bug] [回归]**

    修复了由问题[#9058](https://www.sqlalchemy.org/trac/ticket/9058)引起的回归，调整了 MySQL 方言的`has_table()`，再次使用“DESCRIBE”，当 MySQL 版本 8 在使用不存在的模式名称时引发的特定错误代码是意外的，并且无法解释为布尔结果。

    参考：[#9251](https://www.sqlalchemy.org/trac/ticket/9251)

+   **[mysql] [bug]**

    添加了对 MySQL 8 的新`AS <name> ON DUPLICATE KEY`语法的支持，当使用`Insert.on_duplicate_key_update()`时，这对于 MySQL 8 的较新版本是必需的，因为先前使用`VALUES()`的语法现在在这些版本中会发出弃用警告。服务器版本检测被用来确定是否应该使用传统的 MariaDB / MySQL < 8 `VALUES()`语法，还是新的 MySQL 8 所需的语法。感谢 Caspar Wylie 的拉取请求。

    参考：[#8626](https://www.sqlalchemy.org/trac/ticket/8626)

### sqlite

+   **[sqlite] [bug]**

    修复了 SQLite 方言的`has_table()`函数，以便对于包含不存在的模式的非 None 模式名称的查询，正确地报告 False；以前，会引发数据库错误。

    参考：[#9251](https://www.sqlalchemy.org/trac/ticket/9251)

## 2.0.1

发布日期：2023 年 2 月 1 日

### orm

+   **[orm] [bug] [回归]**

    修复了使用具有复合外键的连接表继承的 ORM 模型会在映射器内部遇到内部错误的回归。

    参考：[#9164](https://www.sqlalchemy.org/trac/ticket/9164)

+   **[orm] [bug]**

    当将基类的链接策略选项链接到子类的另一个属性时，错误报告得到改进，应该使用`of_type()`。以前，当使用`Load.options()`时，消息缺乏`of_type()`应该使用的信息详细信息，而在直接链接选项时不是这样。即使使用`Load.options()`，信息详细信息现在也会发出。

    参考：[#9182](https://www.sqlalchemy.org/trac/ticket/9182)

### orm 声明

+   **[bug] [orm 声明]**

    添加了对 [**PEP 484**](https://peps.python.org/pep-0484/) `NewType` 的支持，可以在 `registry.type_annotation_map` 中使用，以及在 `Mapped` 构造中使用。这些类型将与当前的自定义类型子类相同；它们必须显式出现在 `registry.type_annotation_map` 中进行映射。

    参考资料：[#9175](https://www.sqlalchemy.org/trac/ticket/9175)

+   **[bug] [orm declarative]**

    在使用 `MappedAsDataclass` 超类时，层次结构中的所有类（不管它们是否实际上已映射）都将通过 `@dataclasses.dataclass` 函数运行，因此在映射的子类被转换为数据类时，层次结构中声明的非 ORM 字段将被使用。该行为既适用于使用 `__abstract__ = True` 映射的中介类，也适用于用户定义的声明基类本身，假设 `MappedAsDataclass` 作为这些类的超类存在。

    这允许像 `InitVar` 声明这样的非映射属性在超类上使用，而无需在每个非映射类上显式运行 `@dataclasses.dataclass` 装饰器。新行为被认为是正确的，因为这是当使用超类来指示数据类行为时 [**PEP 681**](https://peps.python.org/pep-0681/) 实现所期望的。

    参考资料：[#9179](https://www.sqlalchemy.org/trac/ticket/9179)

+   **[bug] [orm declarative]**

    添加了对 [**PEP 586**](https://peps.python.org/pep-0586/) `Literal[]` 的支持，可以在 `registry.type_annotation_map` 中使用，以及在 `Mapped` 构造中使用。要使用这样的自定义类型，它们必须显式出现在 `registry.type_annotation_map` 中进行映射。感谢 Frederik Aalund 提交的拉取请求。

    作为此更改的一部分，将 `Enum` 在 `registry.type_annotation_map` 中的支持扩展为还包括 `Literal[]` 类型，该类型由字符串值组成，除了 `enum.Enum` 数据类型之外。如果在 `Mapped[]` 中使用了一个未在 `registry.type_annotation_map` 中链接到特定数据类型的 `Literal[]` 数据类型，那么默认将使用一个 `Enum`。

    另请参阅

    在类型映射中使用 Python Enum 或 pep-586 Literal 类型

    参考：[#9187](https://www.sqlalchemy.org/trac/ticket/9187)

+   **[bug] [orm declarative]**

    在 `registry.type_annotation_map` 中使用 `Enum` 时，修复了一个问题，即如果根据文档将 `Enum.native_enum` 参数重写为 False，则该参数将不会正确地复制到映射的列数据类型中。

    参考：[#9200](https://www.sqlalchemy.org/trac/ticket/9200)

+   **[bug] [orm declarative] [regression]**

    在 `DeclarativeBase` 类中修复了一个回归，其中注册表的默认构造函数不会应用于基类本身，这与以前的 `declarative_base()` 构造方式不同。这会阻止具有自己的 `__init__()` 方法的映射类调用 `super().__init__()` 以访问注册表的默认构造函数并自动填充属性，而是会命中 `object.__init__()`，这会导致在任何参数上引发 `TypeError`。

    参考：[#9171](https://www.sqlalchemy.org/trac/ticket/9171)

+   **[bug] [orm declarative]**

    改进了在使用 Annotated Declarative mapping 时解释 [**PEP 593**](https://peps.python.org/pep-0593/) `Annotated` 类型的规则集，内部类型将始终检查“Optional”，并将其添加到确定列是否为“nullable”的条件中；如果 `Annotated` 容器中的类型是可选的（或与 `None` 联合），则如果没有显式的 `mapped_column.nullable` 参数覆盖它，将考虑该列为可空。

    参考：[#9177](https://www.sqlalchemy.org/trac/ticket/9177)

### sql

+   **[sql] [bug]**

    修正了版本 2.0.0 中发布的对[#7664](https://www.sqlalchemy.org/trac/ticket/7664)的修复，还包括了`DropSchema`，这在修复中被无意中忽略，允许在没有方言的情况下进行字符串化。这两个构造的修复已经回溯到 1.4.47 版本的 1.4 系列。

    参考：[#7664](https://www.sqlalchemy.org/trac/ticket/7664)

+   **[sql] [bug] [regression]**

    修复了与新“insertmanyvalues”功能实现相关的回归，其中在 CTE 中引用另一个`insert()`时会出现内部`TypeError`的情况；为此情况进行了额外的修复，适用于像 asyncpg 这样的位置方言在使用“insertmanyvalues”时。

    参考：[#9173](https://www.sqlalchemy.org/trac/ticket/9173)

### typing

+   **[typing] [bug]**

    打开了对`Select.with_for_update.of`的类型，也接受表和映射类参数，似乎适用于 MySQL 方言。

    参考：[#9174](https://www.sqlalchemy.org/trac/ticket/9174)

+   **[typing] [bug]**

    修复了限制/偏移方法的类型，包括`Select.limit()`、`Select.offset()`、`Query.limit()`、`Query.offset()`，允许`None`，这是“取消”当前限制/偏移的文档化 API。

    参考：[#9183](https://www.sqlalchemy.org/trac/ticket/9183)

+   **[typing] [bug]**

    修复了`mapped_column()`对象被类型化为`Mapped`时不会被接受在模式约束中，如`ForeignKey`、`UniqueConstraint`或`Index`。

    参考：[#9170](https://www.sqlalchemy.org/trac/ticket/9170)

+   **[typing] [bug]**

    修复了对 `ColumnElement.cast()` 的类型检查，以接受 `Type[TypeEngine[T]]` 和 `TypeEngine[T]`；先前只接受 `TypeEngine[T]`。拉取请求由 Yurii Karabas 提供。

    参考资料：[#9156](https://www.sqlalchemy.org/trac/ticket/9156)

## 2.0.0

发布日期：2023 年 1 月 26 日

### ORM

+   **[ORM] [错误]**

    改进了在配置映射器或刷新过程中发出的警告的通知，这些警告通常作为不同操作的一部分调用，以在可能不明显相关的操作中添加附加上下文到警告的消息，指示其中一个这些操作作为警告源在操作中的消息内。

    参考资料：[#7305](https://www.sqlalchemy.org/trac/ticket/7305)

### ORM 扩展

+   **[功能] [ORM 扩展]**

    向水平分片 API `set_shard_id` 添加了新选项，该选项设置了要针对其进行查询的有效分片标识符，包括主查询以及所有次要加载程序，包括关系急加载程序以及关系和列延迟加载程序。

    参考资料：[#7226](https://www.sqlalchemy.org/trac/ticket/7226)

+   **[用例] [ORM 扩展]**

    为 `AutomapBase` 添加了新功能，用于跨多个模式自动加载类，这些类可能具有重叠的名称，方法是提供一个 `AutomapBase.prepare.modulename_for_table` 参数，允许自定义新生成的类的 `__module__` 属性，以及一个新集合 `AutomapBase.by_module`，它存储了基于 `__module__` 属性的类的点分隔的模块名称空间。

    此外，现在可以任意次调用 `AutomapBase.prepare()` 方法，无论是否启用了反射；在每次调用时，只会处理之前未映射的新添加的表。先前，需要显式调用 `MetaData.reflect()` 方法。

    另请参阅

    从多个模式生成映射 - 同时演示两种技术的使用。

    参考资料：[#5145](https://www.sqlalchemy.org/trac/ticket/5145)

### SQL

+   **[SQL] [错误]**

    修复了`CreateSchema` DDL 构造的字符串化问题，当没有方言时，会导致`AttributeError`。更新：请注意，此修复未考虑到`DropSchema`；版本 2.0.1 中的后续修复解决了这个问题。这两个元素的修复已经回溯到 1.4.47。

    参考：[#7664](https://www.sqlalchemy.org/trac/ticket/7664)

### 输入

+   **[输入] [错误]**

    为从`func`命名空间中可用的内置通用函数添加了类型，这些函数接受一组特定的参数并返回特定的类型，例如`count`，`current_timestamp`等。

    参考：[#9129](https://www.sqlalchemy.org/trac/ticket/9129)

+   **[输入] [错误]**

    修正了“lambda 语句”传递的类型，以便 mypy、pyright 等可以接受普通 lambda 而不会出现关于参数类型的任何错误。此外，为更多的 lambda 语句公共 API 实现了类型，并确保`StatementLambdaElement`是`Executable`层次结构的一部分，因此它被类型化为被`Connection.execute()`接受。

    参考：[#9120](https://www.sqlalchemy.org/trac/ticket/9120)

+   **[输入] [错误]**

    `ColumnOperators.in_()`和`ColumnOperators.not_in()`方法的类型现在包括`Iterable[Any]`，而不是`Sequence[Any]`，以提供更灵活的参数类型。

    参考：[#9122](https://www.sqlalchemy.org/trac/ticket/9122)

+   **[输入] [错误]**

    从类型的角度来看，`or_()` 和 `and_()` 需要第一个参数存在，但这些函数仍然接受零个参数，这将在运行时发出弃用警告。还添加了类型支持，以支持将固定字面量`False`用于`or_()` 和 `True`用于`and_()` 作为唯一的第一个参数，但文档现在指示在这些情况下发送`false()` 和 `true()` 构造作为更明确的方法。

    参考：[#9123](https://www.sqlalchemy.org/trac/ticket/9123)

+   **[打字] [错误]**

    修复了在对`Query`对象进行迭代时类型不正确的问题。

    参考：[#9125](https://www.sqlalchemy.org/trac/ticket/9125)

+   **[打字] [错误]**

    修复了在使用`Result`作为上下文管理器时对象类型未被保留的问题，始终指示所有情况下的`Result`而不是特定的`Result`子类型。感谢 Martin Baláž 的拉取请求。

    参考：[#9136](https://www.sqlalchemy.org/trac/ticket/9136)

+   **[打字] [错误]**

    修复了使用`relationship.remote_side` 和类似参数时的问题，传递作为`Mapped`类型的注释声明对象将不被类型检查器接受。

    参考：[#9150](https://www.sqlalchemy.org/trac/ticket/9150)

+   **[打字] [错误]**

    为诸如`isnot()`、`notin_()`等旧操作符添加了类型，这些操作符以前引用了更新的操作符，但它们本身没有被类型化。

    参考：[#9148](https://www.sqlalchemy.org/trac/ticket/9148)

### postgresql

+   **[postgresql] [错误]**

    添加了对 asyncpg 方言的支持，以在可用时返回`cursor.rowcount`值用于 SELECT 语句。虽然这不是`cursor.rowcount`的典型用法，但其他 PostgreSQL 方言通常提供此值。感谢 Michael Gorven 的拉取请求。

    此更改也**回溯**到：1.4.47

    参考：[#9048](https://www.sqlalchemy.org/trac/ticket/9048)

### mysql

+   **[mysql] [用例]**

    添加了对 MySQL 索引反射的支持，以正确反映先前被忽略的`mysql_length`字典。

    此更改也**回溯**到：1.4.47

    参考：[#9047](https://www.sqlalchemy.org/trac/ticket/9047)

### mssql

+   **[mssql] [bug] [regression]**

    MSSQL 方言的新添加的注释反射和渲染功能，添加于[#7844](https://www.sqlalchemy.org/trac/ticket/7844)，如果无法确定是否使用不受支持的后端（如 Azure Synapse），则现在将默认禁用；这个后端不支持表和列注释，也不支持用于生成它们以及反映它们的 SQL Server 例程。方言添加了一个新参数`supports_comments`，默认值为`None`，表示应自动检测注释支持。当设置为`True`或`False`时，注释支持将被无条件启用或禁用。

    另请参阅

    DDL 注释支持

    参考：[#9142](https://www.sqlalchemy.org/trac/ticket/9142)

## 2.0.0rc3

发布日期：2023 年 1 月 18 日

### orm

+   **[orm] [feature]**

    向`Mapper`添加了一个名为`Mapper.polymorphic_abstract`的新参数。该指令的目的是让 ORM 不考虑该类被直接实例化或加载，只考虑子类。实际效果是，`Mapper`将阻止直接实例化该类的实例，并期望该类没有配置独特的多态标识。

    在实践中，使用`Mapper.polymorphic_abstract`映射的类可以作为`relationship()`的目标，也可以在查询中使用；当然，子类必须在映射中包含多态标识。

    新参数会自动应用于继承`AbstractConcreteBase`类的类，因为这个类不打算被实例化。

    另请参阅

    使用多态抽象构建更深层次的层次结构

    参考：[#9060](https://www.sqlalchemy.org/trac/ticket/9060)

+   **[orm] [bug]**

    修复了在`registry.type_annotation_map`中使用 pep-593 `Annotated` 类型，其本身包含一个通用的普通容器或`collections.abc`类型（例如 `list`, `dict`, `collections.abc.Sequence` 等）作为目标类型时会在 ORM 尝试解释`Annotated`实例时产生内部错误的问题。

    参考：[#9099](https://www.sqlalchemy.org/trac/ticket/9099)

+   **[orm] [bug]**

    当`relationship()`映射到抽象容器类型（例如`Mapped[Sequence[B]]`）时未提供`relationship.container_class`参数时，添加了一个错误消息，此参数在类型为抽象时是必需的。以前，抽象容器会在稍后的步骤中尝试实例化并失败。

    参考：[#9100](https://www.sqlalchemy.org/trac/ticket/9100)

### sql

+   **[sql] [bug]**

    修复了使用与`Update.values()`方法中的列相同名称的`bindparam()`，以及在 2.0 中的 `Insert.values()` 方法中，如果在构造的语句中使用相同名称的参数，则在某些情况下会静默失败，替换为同名的新参数，并丢弃 SQL 表达式的其他元素，例如 SQL 函数等。特定情况将是针对 ORM 实体而不是普通`Table`实例构造的语句，但如果使用 `Session` 或 `Connection` 调用语句，则会发生。

    `Update` 部分的问题既存在于 2.0 中也存在于 1.4 中，并被回溯到 1.4。

    此更改还**被回溯**到：1.4.47

    参考：[#9075](https://www.sqlalchemy.org/trac/ticket/9075)

### typing

+   **[typing] [bug]**

    修复了在`sqlalchemy.ext.hybrid`扩展中注释的问题，以更有效地对用户定义的方法进行类型化。现在，typing 使用[**PEP 612**](https://peps.python.org/pep-0612/)功能，最近的 Mypy 版本也支持，以维护`hybrid_method`的参数签名。 混合方法的返回值在`Select.where()`等上下文中被接受为 SQL 表达式，同时仍支持 SQL 方法。

    参考：[#9096](https://www.sqlalchemy.org/trac/ticket/9096)

### mypy

+   **[mypy] [bug]**

    调整了 mypy 插件，以适应 SQLAlchemy 1.4 时可能进行的一些更改，这些更改是针对问题＃236 sqlalchemy2-stubs 而进行的。这些更改与 SQLAlchemy 2.0 保持同步。这些更改也向后兼容旧版本的 sqlalchemy2-stubs。

    此更改还**回溯到**：1.4.47

+   **[mypy] [bug]**

    修复了 mypy 插件中的崩溃，在 1.4 和 2.0 版本中均可能发生，如果装饰器用于与具有两个以上组件的表达式（例如`@Backend.mapper_registry.mapped`）中引用，则会发生。 现在，此场景被忽略； 使用插件时，装饰器表达式需要是两个组件（即`@reg.mapped`）。

    此更改还**回溯到**：1.4.47

    参考：[#9102](https://www.sqlalchemy.org/trac/ticket/9102)

### postgresql

+   **[postgresql] [bug]**

    修复了 psycopg3 版本 3.1.8 更改了 API 调用的回归，以期望先前未强制执行的特定对象类型，从而破坏了 psycopg3 方言的连接性。

    参考：[#9106](https://www.sqlalchemy.org/trac/ticket/9106)

### oracle

+   **[oracle] [usecase]**

    添加了对 Oracle SQL 类型`TIMESTAMP WITH LOCAL TIME ZONE`的支持，使用新添加的 Oracle 特定的`TIMESTAMP`数据类型。

    参考：[#9086](https://www.sqlalchemy.org/trac/ticket/9086)

## 2.0.0rc2

Released: January 9, 2023

### orm

+   **[orm] [bug]**

    修复了在 2.0 中添加的过于严格的 ORM 映射规则，该规则阻止了对`TableClause`对象的映射，例如在 wiki 上使用的视图配方中使用的那些。

    参考：[#9071](https://www.sqlalchemy.org/trac/ticket/9071)

### typing

+   **[typing] [bug]**

    数据类转换参数`field_descriptors`在 PEP 681 的已接受版本中更名为`field_specifiers`。

    参考：[#9067](https://www.sqlalchemy.org/trac/ticket/9067)

### postgresql

+   **[postgresql] [json]**

    Implemented missing `JSONB` operations:

    +   `@@` 使用`Comparator.path_match()`

    +   `@?` 使用`Comparator.path_exists()`

    +   `#-` 使用`Comparator.delete_path()`

    感谢 Guilherme Martins Crocetti 提供的拉取请求。

    参考文献：[#7147](https://www.sqlalchemy.org/trac/ticket/7147)

### mysql

+   **[mysql] [bug]**

    恢复了`Inspector.has_table()`的行为，以报告 MySQL / MariaDB 的临时表。这是所有其他包含的方言当前的行为，但是在 1.4 中由于不再使用 DESCRIBE 命令而删除了 MySQL 的该行为；在此版本或任何以前的版本上都没有记录支持通过`Inspector.has_table()`方法报告临时表的文件，因此以前的行为未定义。

    由于 SQLAlchemy 2.0 已经为临时表状态添加了正式支持通过`Inspector.has_table()`，因此 MySQL / MariaDB 方言已恢复为使用“DESCRIBE”语句，就像 SQLAlchemy 1.3 系列和以前一样，并且添加了测试支持以包含 MySQL / MariaDB 的此行为。由于 SQLAlchemy 2.0 中`Connection`如何处理事务的简化，1.4 试图改进的 ROLLBACK 引发的先前问题不适用。

    DESCRIBE 是必需的，因为 MariaDB 特别是没有任何一致可用的公共信息模式以报告临时表，除了依赖于抛出错误以报告无结果的 DESCRIBE/SHOW COLUMNS。

    参考文献：[#9058](https://www.sqlalchemy.org/trac/ticket/9058)

### oracle

+   **[oracle] [bug]**

    支持外键约束的用例，其中本地列标记为“不可见”。当反射创建检查目标列的`ForeignKeyConstraint`时，通常生成的错误被禁用，并且与已存在的具有类似问题的`Index`一样，跳过该约束并发出警告。

    参考：[#9059](https://www.sqlalchemy.org/trac/ticket/9059)

## 2.0.0rc1

发布日期：2022 年 12 月 28 日

### 通用

+   **[通用] [错误]**

    修复了基础兼容模块调用`platform.architecture()`以检测某些系统属性的回归错误，结果是对系统级别的`file`调用进行了过于广泛的系统调用，在某些情况下不可用，包括某些安全环境配置中。

    此更改也已被**回溯**到：1.4.46

    参考：[#8995](https://www.sqlalchemy.org/trac/ticket/8995)

### ORM

+   **[orm] [功能]**

    添加了一个新的默认值为`Mapper.eager_defaults`参数“auto”的值，这将在工作单元刷新期间自动获取表默认值，如果方言支持 INSERT 的 RETURNING，以及可用的 insertmanyvalues。对于服务器端 UPDATE 默认值的及时获取，这是非常罕见的，只有当`Mapper.eager_defaults`设置为`True`时才会发生，因为对于 UPDATE 语句没有批量 RETURNING 形式。

    参考：[#8889](https://www.sqlalchemy.org/trac/ticket/8889)

+   **[orm] [用例]**

    关于`Session`的可扩展性调整，以及`ShardedSession`扩展的更新：

    +   `Session.get()`现在接受`Session.get.bind_arguments`，特别是在使用水平分片扩展时可能会有用。

    +   `Session.get_bind()`接受任意关键字参数，这有助于开发使用覆盖此方法的 `Session` 类的代码，该方法具有附加参数。

    +   添加了一个新的 ORM 执行选项 `identity_token`，它可用于直接影响与新加载的 ORM 对象关联的“身份令牌”。这个令牌是分片方法（主要是`ShardedSession`，但也可以在其他情况下使用）在不同“分片”之间分离对象标识的方式。

        另见

        身份令牌

    +   `SessionEvents.do_orm_execute()`事件钩子现在可以用于影响所有与 ORM 相关的选项，包括`autoflush`、`populate_existing`和`yield_per`；这些选项在事件钩子被调用后重新消耗，然后才被执行。以前，像`autoflush`这样的选项在这一点上已经被评估过了。新的`identity_token`选项也在这种模式下受支持，并且现在被水平分片扩展使用。

    +   `ShardedSession`类用新的钩子`ShardedSession.identity_chooser`替换了`ShardedSession.id_chooser`钩子，不再依赖于传统的`Query`对象。在替代`ShardedSession.identity_chooser`时，会发出弃用警告，仍然接受`ShardedSession.id_chooser`。

    参考：[#7837](https://www.sqlalchemy.org/trac/ticket/7837)

+   **[orm] [usecase]**

    “将外部事务加入到会话中”的行为已经进行了修订和改进，允许显式控制`Session`如何适应已经建立事务和可能已经建立保存点的传入`Connection`。新参数`Session.join_transaction_mode`包括一系列选项值，可以以多种方式适应现有事务，最重要的是允许`Session`以完全事务方式操作，仅使用保存点，同时在任何情况下保持外部启动的事务未提交且活动，允许测试套件回滚测试中发生的所有更改。

    此外，对`Session.close()`方法进行了修订，以完全关闭可能仍存在的保存点，这也允许“外部事务”配方在`Session`未明确结束其自身 SAVEPOINT 事务时继续进行而不产生警告。

    另请参阅

    会话的新事务加入模式

    参考：[#9015](https://www.sqlalchemy.org/trac/ticket/9015)

+   **[orm] [usecase]**

    移除了在检测到非`Mapped[]`注释时必须使用`__allow_unmapped__`属性的要求；以前，如果检测到旨在支持遗留 ORM 类型映射的错误消息将被引发，此外还未提及与 Dataclasses 特别相关的正确模式。如果使用了`registry.mapped_as_dataclass()`或`MappedAsDataclass`，则不再引发此错误消息。

    另请参阅

    使用非映射数据类字段

    参考：[#8973](https://www.sqlalchemy.org/trac/ticket/8973)

+   **[orm] [bug]**

    修复了内部 SQL 遍历中的问题，例如对`Update`和`Delete`等 DML 语句，这将导致除其他潜在问题之外，使用 ORM 更新/删除功能的 lambda 语句的特定问题。

    此更改也被**回溯**到：1.4.46

    参考：[#9033](https://www.sqlalchemy.org/trac/ticket/9033)

+   **[orm] [bug]**

    修复了一个 bug，即`Session.merge()`无法保留使用`relationship.viewonly`参数指示的关系属性的当前加载内容，从而破坏了使用`Session.merge()`从缓存和其他类似技术中拉取完全加载的对象的策略。在相关更改中，修复了一个问题，即包含已配置为在映射上`lazy='raise'`的已加载关系的对象在传递给`Session.merge()`时会失败；假设`Session.merge.load`参数保持其默认值`True`，则合并过程中的“raise”检查现在被暂停了。

    总体而言，这是对 1.4 系列中引入的更改的行为调整，截至[#4994](https://www.sqlalchemy.org/trac/ticket/4994)，该更改将“merge”从“viewonly”关系的默认级联集中移除。由于“viewonly”关系在任何情况下都不会被持久化，因此在“merge”期间允许其内容传输不会影响目标对象的持久化行为。这使得`Session.merge()`能够正确地适用于其用例之一，即将在其他地方加载的对象添加到`Session`中，通常是为了从缓存中恢复。

    此更改也被**回溯**到：1.4.45

    参考：[#8862](https://www.sqlalchemy.org/trac/ticket/8862)

+   **[orm] [bug]**

    修复了`with_expression()`中的问题，在某些情况下，由于表达式由从外部 SELECT 中引用的列组成，因此不会正确地在某些上下文中呈现 SQL，在表达式具有与使用`query_expression()`的属性匹配的标签名称的情况下，即使`query_expression()`没有默认表达式。暂时，如果`query_expression()`确实具有默认表达式，则仍将使用该标签名称作为该默认表达式，并且将继续忽略具有相同名称的其他标签。总体而言，这种情况相当棘手，因此可能需要进一步调整。

    此更改还**回溯**到：1.4.45

    参考：[#8881](https://www.sqlalchemy.org/trac/ticket/8881)

+   **[orm] [bug]**

    如果在`relationship()`中使用的反向引用名称在目标类上命名了已经有方法或属性分配给该名称的属性，则会发出警告，因为反向引用声明将替换该属性。

    参考：[#4629](https://www.sqlalchemy.org/trac/ticket/4629)

+   **[orm] [bug]**

    一系列关于`Session.refresh()`的变更和改进。总体变更是，当要刷新与关系绑定的属性时，对象的主键属性现在无条件地包含在刷新操作中，即使未过期，即使未在刷新中指定。

    +   改进了`Session.refresh()`，以便如果启用了自动刷新（如`Session`的默认值），则自动刷新将在刷新过程的较早部分发生，以便应用待处理的主键更改而不会引发错误。以前，此自动刷新发生得太晚，并且 SELECT 语句不会使用正确的键来定位行，并且会引发`InvalidRequestError`。

    +   当存在上述条件时，即对象上存在未刷新的主键更改，但未启用自动刷新时，refresh()方法现在明确禁止操作继续进行，并引发一个信息性的`InvalidRequestError`，要求首先刷新待处理的主键更改。以前，这种用例简单地被破坏，无论如何都会引发`InvalidRequestError`。这个限制是为了安全起见，以便刷新主键属性，这对于能够刷新具有 relationship 绑定的次要急切加载器的对象是必要的。无论是否实际上需要刷新 PK 列，这个规则都适用于保持 API 行为一致，因为在任何情况下，刷新对象的某些属性而保留其他属性“待处理”是不寻常的。

    +   `Session.refresh()` 方法已经得到增强，以便刷新那些在映射时或通过最近使用的加载器选项与`relationship()`绑定并链接到急切加载器的属性，在所有情况下都会被刷新，即使传递了一个不包括父行上任何列的属性列表。这是在 1.4 中作为[#1763](https://www.sqlalchemy.org/trac/ticket/1763)的一部分首次实现的功能，允许急切加载的与 relationship 绑定的属性参与`Session.refresh()`操作。如果刷新操作没有指示要刷新父行上的任何列，则主键列仍将包括在刷新操作中，这允许加载继续到正常情况下指示的次要关系加载器。以前，对于这种情况会引发一个`InvalidRequestError`错误（[#8703](https://www.sqlalchemy.org/trac/ticket/8703))

    +   修复了一个问题，即在`Session.refresh()`与一组过期属性以及像`selectinload()`这样发出“secondary”查询的急加载器一起调用时，会发出不必要的额外 SELECT 的情况，如果主键属性也处于过期状态。由于主键属性现在自动包含在刷新中，因此当关系加载器开始为它们选择时，这些属性不会有额外的加载（[#8997](https://www.sqlalchemy.org/trac/ticket/8997)）。

    +   修复了由 2.0.0b1 中的[#8126](https://www.sqlalchemy.org/trac/ticket/8126)引起的回归，其中`Session.refresh()`方法将在传递过期的列名以及链接到“secondary” eagerloader（如`selectinload()`）的关系绑定属性的名称时失败，并引发`AttributeError`（[#8996](https://www.sqlalchemy.org/trac/ticket/8996)）。

    参考：[#8703](https://www.sqlalchemy.org/trac/ticket/8703), [#8996](https://www.sqlalchemy.org/trac/ticket/8996), [#8997](https://www.sqlalchemy.org/trac/ticket/8997)

+   **[orm] [bug]**

    改进了版本 1.4 中首次修复的问题，用于[#8456](https://www.sqlalchemy.org/trac/ticket/8456)，该问题减少了内部“多态适配器”的使用，用于在使用`Mapper.with_polymorphic`参数时渲染 ORM 查询。这些适配器非常复杂且容易出错，现在仅在使用用户提供的显式子查询用于`Mapper.with_polymorphic`的情况下使用，其中包括仅使用`polymorphic_union()`辅助程序的具体继承映射的用例，以及在不需要的现代用例中使用别名子查询的联合继承映射的传统用例。

    对于使用内置多态加载方案的联合继承映射的最常见情况，其中包括使用`Mapper.polymorphic_load`参数设置为`inline`的情况，现在不再使用多态适配器。这对查询构造的性能有积极影响，同时也极大简化了内部查询渲染过程。

    目标特定问题是允许`column_property()`引用标量子查询中的联合继承类，现在其工作方式尽可能直观。

    参考：[#8168](https://www.sqlalchemy.org/trac/ticket/8168)

### engine

+   **[engine] [bug]**

    修复了连接池中的长期竞争条件，该条件可能在 eventlet/gevent monkeypatching 方案与使用 eventlet/gevent `Timeout` 条件相结合时发生，其中由于超时而中断的连接池检出将无法清理失败的状态，导致底层连接记录以及有时是数据库连接本身“泄漏”，将池留在无效状态中，无法访问条目。这个问题首次在 SQLAlchemy 1.2 中被识别和修复，用于 [#4225](https://www.sqlalchemy.org/trac/ticket/4225)，然而在该修复中检测到的故障模式未能适应 `BaseException`，而不是 `Exception`，这导致无法捕获 eventlet/gevent `Timeout`。此外，在初始池连接中还确定了一个块，并通过 `BaseException` -> “清除失败的连接”块来加固，以适应在此位置的相同条件。非常感谢 Github 用户 @niklaus 在识别和描述这个复杂问题方面的顽强努力。

    此更改也被**回溯**到：1.4.46

    参考：[#8974](https://www.sqlalchemy.org/trac/ticket/8974)

+   **[engine] [bug]**

    修复了`Result.freeze()`方法在使用`text()`或`Connection.exec_driver_sql()`时无法工作的问题。

    此更改也被**回溯**到：1.4.45

    参考：[#8963](https://www.sqlalchemy.org/trac/ticket/8963)

### sql

+   **[sql] [usecase]**

    在“字面绑定参数”渲染操作失败的情况下，现在会抛出一个信息性的重新引发，指示值本身和正在使用的数据类型，以帮助调试在语句中渲染字面参数时的情况。

    此更改也被**回溯**到：1.4.45

    参考：[#8800](https://www.sqlalchemy.org/trac/ticket/8800)

+   **[sql] [bug]**

    修复了 Lambda SQL 功能中的问题，其中字面值的计算类型不会考虑“与类型比较”的类型强制转换规则，导致 SQL 表达式（例如与`JSON`元素的比较等）缺乏类型信息。

    此更改也被**回溯**到：1.4.46

    参考：[#9029](https://www.sqlalchemy.org/trac/ticket/9029)

+   **[sql] [bug]**

    修复了一系列关于渲染绑定参数的位置以及有时身份的问题，例如用于 SQLite、asyncpg、MySQL、Oracle 等的参数。一些编译形式不会正确维护参数的顺序，例如 PostgreSQL `regexp_replace()` 函数、首次在[#4123](https://www.sqlalchemy.org/trac/ticket/4123)中引入的 `CTE` 构造的“嵌套”特性，以及使用 Oracle 的 `FunctionElement.column_valued()` 方法形成的可选择表。

    此更改也**被回溯**到：1.4.45

    参考：[#8827](https://www.sqlalchemy.org/trac/ticket/8827)

+   **[sql] [bug]**

    添加了测试支持，以确保 SQLAlchemy 中所有`Compiler`实现中的所有编译器`visit_xyz()`方法都接受 `**kw` 参数，以便所有编译器在所有情况下都接受额外的关键字参数。

    参考：[#8988](https://www.sqlalchemy.org/trac/ticket/8988)

+   **[sql] [bug]**

    `SQLCompiler.construct_params()` 方法以及`SQLCompiler.params` 访问器现在将返回与使用 `render_postcompile` 参数编译的编译语句对应的确切参数。之前，该方法返回的参数结构本身既不对应原始参数也不对应扩展参数。

    不再允许将新参数字典传递给使用 `render_postcompile` 构造的`SQLCompiler` 的 `SQLCompiler.construct_params()`；相反，为了为另一组参数制作新的 SQL 字符串和参数集，添加了一个新方法 `SQLCompiler.construct_expanded_state()`，该方法将使用包含新的 SQL 语句和新的参数字典的 `ExpandedState` 容器产生给定参数集的新扩展形式，以及一个位置参数元组。

    参考：[#6114](https://www.sqlalchemy.org/trac/ticket/6114)

+   **[sql] [bug]**

    为了适应对绑定参数有不同字符转义需求的第三方方言，SQLAlchemy 中用于“转义”（即在其位置替换为另一个字符）绑定参数名称的系统已被扩展，使用 `SQLCompiler.bindname_escape_chars` 字典，可以在任何 `SQLCompiler` 子类的类声明级别进行覆盖。作为此更改的一部分，还将点号 `"."` 添加为默认的 “转义” 字符。

    引用：[#8994](https://www.sqlalchemy.org/trac/ticket/8994)

### typing

+   **[typing] [bug]**

    pep-484 typing 已完成对 `sqlalchemy.ext.horizontal_shard` 扩展以及 `sqlalchemy.orm.events` 模块的类型标注。感谢 Gleb Kisenkov 的努力。

    引用：[#6810](https://www.sqlalchemy.org/trac/ticket/6810), [#9025](https://www.sqlalchemy.org/trac/ticket/9025)

### asyncio

+   **[asyncio] [bug]**

    从 `AsyncResult` 中删除了不起作用的 `merge()` 方法。这个方法从未起作用，是错误地包含在 `AsyncResult` 中的。

    此更改还 **反向移植** 至：1.4.45

    引用：[#8952](https://www.sqlalchemy.org/trac/ticket/8952)

### postgresql

+   **[postgresql] [bug]**

    修复了一个 PostgreSQL 的 bug，其中 `Insert.on_conflict_do_update.constraint` 参数会接受一个 `Index` 对象，但是不会将此索引展开为其各个索引表达式，而是在 ON CONFLICT ON CONSTRAINT 子句中呈现其名称，这在 PostgreSQL 中不被接受；“约束名”形式仅接受唯一或排除约束名。该参数继续接受索引，但现在会将其展开为其组件表达式以进行呈现。

    此更改还 **反向移植** 至：1.4.46

    引用：[#9023](https://www.sqlalchemy.org/trac/ticket/9023)

+   **[postgresql] [bug]**

    对 PostgreSQL 方言在从表中反射列时考虑列类型的方式进行了调整，以适应可能从 PG 的 `format_type()` 函数返回 NULL 的替代后端。

    此更改还 **反向移植** 至：1.4.45

    引用：[#8748](https://www.sqlalchemy.org/trac/ticket/8748)

+   **[postgresql] [bug]**

    增加了对使用 `asyncpg` 和 `psycopg`（仅限于 SQLAlchemy 2.0）的 PG 全文函数的显式支持，关于第一个参数的 `REGCONFIG` 类型转换，之前会错误地转换为 VARCHAR，导致这些方言上的失败，这些方言依赖于显式类型转换。这包括对`to_tsvector`, `to_tsquery`, `plainto_tsquery`, `phraseto_tsquery`, `websearch_to_tsquery`, `ts_headline` 的支持，每个函数根据传递的参数数量来确定第一个字符串参数是否应解释为 PostgreSQL 的`REGCONFIG`值；如果是，则使用新添加的类型对象 `REGCONFIG` 进行类型转换，然后在 SQL 表达式中显式地转换。

    参考：[#8977](https://www.sqlalchemy.org/trac/ticket/8977)

+   **[postgresql] [错误]**

    修复了新修订的 PostgreSQL 范围类型（例如`INT4RANGE`自定义类型的实现，而是引发 `TypeError` 的回归问题。

    参考：[#9020](https://www.sqlalchemy.org/trac/ticket/9020)

+   **[postgresql] [错误]**

    当与不同类的实例进行比较时，`Range.__eq___()` 现在会返回 `NotImplemented`，而不是引发 `AttributeError` 异常。

    参考：[#8984](https://www.sqlalchemy.org/trac/ticket/8984)

### sqlite

+   **[sqlite] [用例]**

    为 SQLite 后端增加了反映可能存在于外键结构上的“DEFERRABLE”和“INITIALLY”关键字的支持。感谢 Michael Gorven 提交的拉取请求。

    这个变化也被**回溯**到：1.4.45

    参考：[#8903](https://www.sqlalchemy.org/trac/ticket/8903)

+   **[sqlite] [用例]**

    增加了对 SQLite 方言中索引中包含的表达式导向的 WHERE 条件的反射支持，类似于 PostgreSQL 方言的方式。感谢 Tobias Pfeiffer 提交的拉取请求。

    这个变化也被**回溯**到：1.4.45

    参考：[#8804](https://www.sqlalchemy.org/trac/ticket/8804)

### oracle

+   **[oracle] [错误]**

    修复了 Oracle 编译器中 `FunctionElement.column_valued()` 语法不正确的问题，导致 `COLUMN_VALUE` 名称没有正确限定源表。

    此更改还被**反向移植**至：1.4.45

    参考：[#8945](https://www.sqlalchemy.org/trac/ticket/8945)

### 测试

+   **[测试] [错误]**

    修复了 tox.ini 文件中的问题，其中在 tox 4.0 系列对“passenv”的格式进行更改导致 tox 无法正常工作，特别是在 tox 4.0.6 中引发错误。

    此更改还被**反向移植**至：1.4.46

+   **[测试] [错误]**

    添加了针对第三方方言的新排除规则 `unusual_column_name_characters`，可以将其关闭，以防第三方方言不支持具有不寻常字符的列名，例如点、斜杠或百分号，即使名称已正确引用。

    此更改还被**反向移植**至：1.4.46

    参考：[#9002](https://www.sqlalchemy.org/trac/ticket/9002)

## 2.0.0b4

发布日期：2022 年 12 月 5 日

### orm

+   **[orm] [功能]**

    添加了一个新参数 `mapped_column.use_existing_column` 以适应单表继承映射的用例，该映射使用一个以上的子类指示相同的列位于超类上。以前可以通过在超类的 `.__table__` 中使用 `declared_attr()` 与定位现有列的方法来实现此模式，但现在已更新为使用 `mapped_column()` 以及 pep-484 类型提示，以一种简单而简洁的方式。

    另请参阅

    使用 use_existing_column 解决列冲突

    参考：[#8822](https://www.sqlalchemy.org/trac/ticket/8822)

+   **[orm] [用例]**

    添加了对自定义用户定义类型的支持，这些类型扩展了 Python `enum.Enum` 基类，以便在使用注释式声明表功能时自动解析为 SQLAlchemy 的 `Enum` SQL 类型。该功能通过向 ORM 类型映射功能添加的新查找功能实现，并包括对默认生成的 `Enum` 的参数进行更改的支持，以及设置映射中特定的 `enum.Enum` 类型及其特定参数的支持。

    另请参阅

    在类型映射中使用 Python Enum 或 pep-586 字面类型

    参考：[#8859](https://www.sqlalchemy.org/trac/ticket/8859)

+   **[orm] [用例]**

    为相关的 ORM 属性构造（包括`mapped_column()`，`relationship()`等）添加了`mapped_column.compare`参数，以提供 Python 数据类`field()`的`compare`参数，当使用声明性数据类映射功能时。感谢 Simon Schiele 的拉取请求。

    参考：[#8905](https://www.sqlalchemy.org/trac/ticket/8905)

+   **[orm] [性能] [bug]**

    在 ORM 启用的 SQL 语句中进一步增强了性能，特别针对在构造 ORM 语句时的调用计数，使用`aliased()`与`union()`和类似“复合”结构的组合，以及对 ORM 频繁使用的`corresponding_column()`内部方法的直接性能改进，例如`aliased()`和类似构造。

    参考：[#8796](https://www.sqlalchemy.org/trac/ticket/8796)

+   **[orm] [bug]**

    修复了在列基础属性的`Mapped`注释中使用未知数据类型时静默失败而不是报告异常的问题；现在会引发一个信息性异常消息。

    参考：[#8888](https://www.sqlalchemy.org/trac/ticket/8888)

+   **[orm] [bug]**

    修复了一系列问题，涉及与字典类型一起使用`Mapped`的情况，例如`Mapped[Dict[str, str] | None]`，在声明性 ORM 映射中将不会被正确解释。已修复以正确“去可选化”此类型的支持，包括用于在`type_annotation_map`中查找的支持。

    参考：[#8777](https://www.sqlalchemy.org/trac/ticket/8777)

+   **[orm] [bug]**

    在声明性数据类映射功能中修复了一个 bug，该 bug 导致在映射中使用带有`__allow_unmapped__`指令的普通数据类字段时，不会为这些字段创建具有正确类级状态的数据类，不适当地在数据类自身已经用类级默认值替换`Field`对象后将原始`Field`对象复制到类中。

    参考：[#8880](https://www.sqlalchemy.org/trac/ticket/8880)

+   **[orm] [bug] [回归]**

    修复了一个回归问题，即刷新映射到子查询的映射类时（例如直接映射或某些形式的具体表继承），如果使用了 `Mapper.eager_defaults` 参数，将会失败。

    参考：[#8812](https://www.sqlalchemy.org/trac/ticket/8812)

+   **[ORM] [错误]**

    修复了 2.0.0b3 中的一个回归问题，由 [#8759](https://www.sqlalchemy.org/trac/ticket/8759) 导致，其中使用限定名称（如 `sqlalchemy.orm.Mapped`）指示 `Mapped` 名称将无法被 Declarative 认为指示 `Mapped` 结构。

    参考：[#8853](https://www.sqlalchemy.org/trac/ticket/8853)

### ORM 扩展

+   **[用例] [ORM 扩展]**

    增加了对 `association_proxy()` 扩展函数的支持，以在 Python `dataclasses` 配置中参与，使用了声明性数据类映射中描述的原生数据类功能。包括属性级参数，包括 `association_proxy.init` 和 `association_proxy.default_factory`。

    关联代理的文档也已更新，以在示例中使用 “带注解的声明性表格” 表单，包括用于 `AssocationProxy` 本身的类型注解。

    参考：[#8878](https://www.sqlalchemy.org/trac/ticket/8878)

### SQL

+   **[SQL] [用例]**

    添加了 `ScalarValues`，可用作列元素，允许在 `IN` 子句中使用 `Values`，或与 `ANY` 或 `ALL` 集合聚合一起使用。此新类是使用方法 `Values.scalar_values()` 生成的。当在 `IN` 或 `NOT IN` 操作中使用时，现在会将 `Values` 实例强制转换为 `ScalarValues`。

    参考：[#6289](https://www.sqlalchemy.org/trac/ticket/6289)

+   **[SQL] [错误]**

    修复了在缓存密钥生成中识别的关键内存问题，其中对于使用大量 ORM 别名和子查询的非常大且复杂的 ORM 语句，缓存密钥生成可能会产生比语句本身大几个数量级的大密钥。非常感谢 Rollo Konig Brock 在最终确定此问题方面的非常耐心和长期的帮助。

    此更改也**回溯**到：1.4.44

    参考：[#8790](https://www.sqlalchemy.org/trac/ticket/8790)

+   **[sql] [bug]**

    `numeric` pep-249 paramstyle 的方法已被重写，并且现在得到了完全支持，包括“扩展 IN”和“insertmanyvalues”等功能。参数名称也可以在源 SQL 构造中重复，这将在数值格式内正确表示为单个参数。引入了一个名为`numeric_dollar`的附加数值 paramstyle，它是由 asyncpg 方言使用的; 该 paramstyle 等效于`numeric`，只是数字指示器使用美元符号而不是冒号。asyncpg 方言现在直接使用`numeric_dollar` paramstyle，而不是首先编译为`format`样式。

    `numeric`和`numeric_dollar` paramstyles 假设目标后端能够以任何顺序接收数字参数，并将给定的参数值与语句匹配，基于将它们的位置（从 1 开始）与数字指示器进行匹配。这是`numeric` paramstyles 的正常行为，尽管观察到 SQLite DBAPI 实现了一个不使用的`numeric`样式，它不遵守参数排序。

    参考：[#8849](https://www.sqlalchemy.org/trac/ticket/8849)

+   **[sql] [bug]**

    调整了`RETURNING`的渲染，特别是在使用`Insert`时，现在会像`Select`构造一样渲染列，以生成标签，其中将包括消除歧义的标签，以及将命名列周围的 SQL 函数标记为列名本身。这在从`Select`构造或使用`UpdateBase.returning()`的 DML 语句中选择行时建立了更好的跨兼容性。1.4 系列还做了一项较窄的范围更改，仅调整了函数标签问题。

    参考：[#8770](https://www.sqlalchemy.org/trac/ticket/8770)

### 架构

+   **[schema] [bug]**

    对于将 `Column` 对象附加到 `Table` 对象，现在有更严格的规则，将一些先前的弃用警告转移到异常，并阻止一些先前可能导致表中出现重复列的情况，当设置 `Table.extend_existing` 为 `True` 时，无论是在编程式 `Table` 构建还是在反射操作期间。

    请参阅相同名称、键的表对象中列替换规则更严格以了解这些更改的概述。

    另请参阅

    相同名称、键的表对象中列替换规则更严格

    参考：[#8925](https://www.sqlalchemy.org/trac/ticket/8925)

### 类型

+   **[类型] [用例]**

    添加了一个新类型 `SQLColumnExpression`，可以在用户代码中表示任何 SQL 列导向表达式，包括基于 `ColumnElement` 和 ORM `QueryableAttribute` 的表达式。这个类型是一个真正的类，而不是别名，因此也可以用作其他对象的基础。另外还包括了一个额外的 ORM 特定子类 `SQLORMExpression`。

    参考：[#8847](https://www.sqlalchemy.org/trac/ticket/8847)

+   **[类型] [错误]**

    调整了内部对 Python `enum.IntFlag` 类的使用，该类在 Python 3.11 中改变了其行为契约。这并没有导致运行时失败，但导致了在 Python 3.11 下的类型运行失败。

    参考：[#8783](https://www.sqlalchemy.org/trac/ticket/8783)

+   **[类型] [错误]**

    `sqlalchemy.ext.mutable` 扩展和 `sqlalchemy.ext.automap` 扩展现在完全符合 pep-484 类型标准。非常感谢 Gleb Kisenkov 在这方面的努力。

    参考：[#6810](https://www.sqlalchemy.org/trac/ticket/6810), [#8667](https://www.sqlalchemy.org/trac/ticket/8667)

+   **[类型] [错误]**

    修正了对 `relationship.secondary` 参数的类型支持，该参数也可以接受返回 `FromClause` 的可调用对象（lambda）。

+   **[类型] [错误]**

    改进了 `sessionmaker` 和 `async_sessionmaker` 的类型，使得它们的返回值的默认类型将会是 `Session` 或 `AsyncSession`，而无需明确指定此类型。以前，Mypy 无法从其泛型基类自动推断出这些返回类型。

    这次更改的一部分是，对于`Session`、`AsyncSession`、`sessionmaker`和`async_sessionmaker`的参数除了初始的“bind”参数之外，已经被设置为关键字参数，其中包括一直以来都被记录为关键字参数的参数，比如`Session.autoflush`、`Session.class_`等。

    感谢 Sam Bull 提交的拉取请求。

    参考：[#8842](https://www.sqlalchemy.org/trac/ticket/8842)

+   **[typing] [bug]**

    修复了将返回列元素可迭代对象的可调用函数传递给 `relationship.order_by` 时在类型检查器中标记为错误的问题。

    参考：[#8776](https://www.sqlalchemy.org/trac/ticket/8776)

### postgresql

+   **[postgresql] [usecase]**

    作为 [#8690](https://www.sqlalchemy.org/trac/ticket/8690) 的补充，新增了诸如 `Range.adjacent_to()`、`Range.difference()`、`Range.union()` 等方法到 PG 特定的范围对象中，使其与底层 `AbstractRange.comparator_factory` 实现的标准操作保持一致。

    此外，类的`__bool__()`方法已校正，以与常见的 Python 容器行为以及其他流行的 PostgreSQL 驱动程序相一致：现在它告诉范围实例是否*不*为空，而不是相反。

    拉取请求由 Lele Gaifax 提供。

    参考：[#8765](https://www.sqlalchemy.org/trac/ticket/8765)

+   **[postgresql] [change] [asyncpg]**

    将 asyncpg 使用的 paramstyle 从`format`更改为`numeric_dollar`。这有两个主要好处，因为它不需要对语句进行额外处理，并且允许语句中存在重复的参数。

    参考：[#8926](https://www.sqlalchemy.org/trac/ticket/8926)

+   **[postgresql] [bug] [mssql]**

    仅针对 PostgreSQL 和 SQL Server 方言，调整了编译器，以便在渲染 RETURNING 子句中的列表达式时，建议使用 SELECT 语句中使用的“非匿名”标签作为 SQL 表达式元素的标签;主要示例是可能作为列类型的一部分发出的 SQL 函数，其中标签名称默认应与列名称匹配。这恢复了一个在 1.4.21 版本中由于[#6718](https://www.sqlalchemy.org/trac/ticket/6718)，[#6710](https://www.sqlalchemy.org/trac/ticket/6710)而改变的不好定义的行为。Oracle 方言具有不同的 RETURNING 实现，不受此问题的影响。版本 2.0 对其他后端广泛扩展的 RETURNING 支持进行了全面变更。

    此更改还**反向移植**到：1.4.44

    参考：[#8770](https://www.sqlalchemy.org/trac/ticket/8770)

+   **[postgresql] [bug]**

    对新的 PostgreSQL `Range`类型进行了额外的类型检测，以前允许直接通过 DBAPI 接收 psycopg2 原生范围对象的情况已停止工作，因为现在我们有了自己的值对象。 `Range`对象已得到增强，以便 SQLAlchemy 核心在其他模糊情况下检测到它（例如与日期的比较），并应用适当的绑定处理程序。拉取请求由 Lele Gaifax 提供。

    参考：[#8884](https://www.sqlalchemy.org/trac/ticket/8884)

### mssql

+   **[mssql] [bug]**

    由[#8177](https://www.sqlalchemy.org/trac/ticket/8177)的组合引起的回归，重新启用了 setinputsizes 用于 SQL 服务器，除非使用 fast_executemany + DBAPI executemany 用于语句，以及[#6047](https://www.sqlalchemy.org/trac/ticket/6047)，实现了“insertmanyvalues”，该值绕过了 DBAPI executemany，而是使用 INSERT 语句的自定义 DBAPI execute。如果打开 fast_executemany，setinputsizes 将不会用于使用“insertmanyvalues”的多个参数集 INSERT 语句，因为检查将错误地假设这是一个 DBAPI executemany 调用。然后，“回归”的问题就是，“insertmanyvalues”语句格式显然对于不使用相同类型的多行特别敏感，因此在这种情况下，尤其需要 setinputsizes。

    修复了 fast_executemany 检查，使其仅在使用 true DBAPI executemany 时禁用 setinputsizes。

    参考：[#8917](https://www.sqlalchemy.org/trac/ticket/8917)

### oracle

+   **[oracle] [bug]**

    对 Oracle 修复[#8708](https://www.sqlalchemy.org/trac/ticket/8708)的持续修复，在 1.4.43 中发布，其中以下划线开头的绑定参数名称（Oracle 不允许的）仍然未在所有情况下正确转义。

    此更改也**反向移植**到：1.4.45

    参考：[#8708](https://www.sqlalchemy.org/trac/ticket/8708)

### 测试

+   **[tests] [bug]**

    修复了测试套件中`--disable-asyncio`参数的问题，该参数实际上无法禁止运行 greenlet 测试，并且也无法阻止套件在整个运行过程中使用“wrapping” greenlet。此参数现在确保在设置时不会在整个运行期间发生 greenlet 或 asyncio 使用。

    此更改也**反向移植**到：1.4.44

    参考：[#8793](https://www.sqlalchemy.org/trac/ticket/8793)

## 2.0.0b3

发布日期：2022 年 11 月 4 日

### orm

+   **[orm] [bug]**

    修复了在连接式预加载中出现断言失败的问题，在使用特定外/内连接式预加载组合时会出现断言失败，在跨三个映射器进行预加载时，中间映射器是一个继承的子类映射器。

    此更改也**反向移植**到：1.4.43

    参考：[#8738](https://www.sqlalchemy.org/trac/ticket/8738)

+   **[orm] [bug]**

    修复了涉及`Select`构造的错误，其中`Select.select_from()`与`Select.join()`的组合，以及在使用`Select.join_from()`时，会导致`with_loader_criteria()`功能以及单表继承查询所需的 IN 条件在查询的列子句没有明确包含 JOIN 左侧实体时不会呈现。现在，正确的实体已传递给内部生成的`Join`对象，以便正确添加对左侧实体的条件。

    此更改也**回溯**到：1.4.43

    参考：[#8721](https://www.sqlalchemy.org/trac/ticket/8721)

+   **[orm] [bug]**

    当`with_loader_criteria()`选项作为特定“加载器路径”添加的加载器选项时，现在会引发一个信息性异常，例如在`Load.options()`中使用它时。这种用法不受支持，因为`with_loader_criteria()`只打算用作顶级加载器选项。以前会生成内部错误。

    此更改也**回溯**到：1.4.43

    参考：[#8711](https://www.sqlalchemy.org/trac/ticket/8711)

+   **[orm] [bug]**

    为`Session.get()`改进了“字典模式”，以便可以在命名字典中指示引用主键属性名称的同义词名称。

    此更改也**回溯**到：1.4.43

    参考：[#8753](https://www.sqlalchemy.org/trac/ticket/8753)

+   **[orm] [bug]**

    修复了继承映射器的“selectin_polymorphic”加载不会正确工作的问题，如果`Mapper.polymorphic_on`参数引用的 SQL 表达式不直接映射到类上。

    此更改也**回溯**到：1.4.43

    参考：[#8704](https://www.sqlalchemy.org/trac/ticket/8704)

+   **[orm] [bug]**

    修复了当使用`Query`对象作为迭代器时，如果在迭代过程中出现用户定义的异常情况，则底层的 DBAPI 游标不会被关闭的问题。当使用`Query.yield_per()`来创建服务器端游标时，这会导致通常与 MySQL 相关的服务器端游标不同步的问题，并且由于无法直接访问`Result`对象，最终用户的代码无法访问游标以关闭它。

    为了解决这个问题，在迭代器方法中应用了对`GeneratorExit`的捕获，这样当迭代器被中断时将关闭结果对象，并且按定义将被 Python 解释器关闭。

    作为针对 1.4 系列实现的这一变化的一部分，确保了在所有`Result`实现上都提供了`.close()`方法，包括`ScalarResult`、`MappingResult`。这一变化的 2.0 版本还包括了用于与`Result`类一起使用的新上下文管理器模式。

    这一变化也被**回溯**到了：1.4.43

    参考：[#8710](https://www.sqlalchemy.org/trac/ticket/8710)

### orm 声明式

+   **[orm] [declarative] [bug]**

    在 ORM 声明式注释中为`relationship()`指定的类名添加了支持，以及为`Mapped`符号本身的名称，使其与直接的类名不同，以支持诸如将`Mapped`导入为 `from sqlalchemy.orm import Mapped as M`，或者相关类名以类似方式导入为替代名称的情况。此外，作为`relationship()`的主参数给定的目标类名将始终优先于左手注释中给定的名称，以便仍然可以在注释中使用否则无法导入的名称，而且这些名称也不与类名匹配。

    参考：[#8759](https://www.sqlalchemy.org/trac/ticket/8759)

+   **[orm] [declarative] [bug]**

    改进对使用注释的遗留 1.4 映射的支持，这些注释不包括 `Mapped[]`，通过确保 `__allow_unmapped__` 属性可以用于允许这些遗留注释通过 Annotated Declarative 而不引发错误，并且不在 ORM 运行时上下文中被解释。此外，当检测到这种情况时改进了生成的错误消息，并为应该如何处理这种情况添加了更多文档。不幸的是，1.4 WARN_SQLALCHEMY_20 迁移警告不能在运行时使用当前架构检测到这个特定的配置问题。

    参考：[#8692](https://www.sqlalchemy.org/trac/ticket/8692)

+   **[orm] [declarative] [bug]**

    改变了 `Mapper` 的一个基本配置行为，其中在 `Mapper.properties` 字典中显式存在的 `Column` 对象，无论是直接还是包含在映射器属性对象内部，现在都将在映射的 `Table`（或其他可选择的）本身中以它们出现的顺序进行映射（假设它们实际上是该表的列列表的一部分），从而保持在映射的可选择上的列的顺序与在映射类中操纵的顺序相同，以及在 ORM SELECT 语句中为该映射器渲染的内容相同。以前（“以前”意味着自版本 0.0.1 以来），在 `Mapper.properties` 字典中的 `Column` 对象总是会首先映射，超过了在映射的 `Table` 中其他列的映射，导致在映射器分配属性给映射类时的顺序以及它们在语句中呈现的顺序之间存在差异。

    此更改最显著地发生在声明式将声明的列分配给`Mapper`的方式上，特别是在处理具有 DDL 名称明确不同于映射属性名称的`Column`（或`mapped_column()`）对象以及使用`deferred()`等构造时。新行为将使映射的`Table`中的列顺序与属性映射到类中的顺序相同，由`Mapper`本身分配，并在 ORM 语句（如 SELECT 语句）中呈现，独立于`Column`针对`Mapper`的配置方式。

    参考：[#8705](https://www.sqlalchemy.org/trac/ticket/8705)

+   **[orm] [declarative] [bug]**

    修复了新数据类映射功能中的问题，其中在某些情况下，在继承子类的构造函数中，声明在声明基类/抽象基类/混合类上的列会泄漏。

    参考：[#8718](https://www.sqlalchemy.org/trac/ticket/8718)

+   **[bug] [orm declarative]**

    修复了声明式类型解析器（即解析`ForwardRef`对象的解析器）中的问题，其中在一个特定的源文件中为列声明的类型在最终映射的类位于另一个源文件时会引发`NameError`。现在，这些类型是根据每个类所在的模块来解析的。

    参考：[#8742](https://www.sqlalchemy.org/trac/ticket/8742)

### 引擎

+   **[engine] [feature]**

    为了更好地支持用户定义的异常可能会中断迭代的`Result`和`AsyncResult`对象的使用情况，现在这两个对象以及诸如`ScalarResult`、`MappingResult`、`AsyncScalarResult`、`AsyncMappingResult`等变体都支持上下文管理器的使用，结果将在上下文管理器块的末尾关闭。

    另外，确保所有上述提到的`Result`对象都包括`Result.close()`方法以及`Result.closed`访问器，包括以前没有`.close()`方法的`ScalarResult`和`MappingResult`。

    另请参阅

    结果、AsyncResult 的上下文管理器支持

    引用：[#8710](https://www.sqlalchemy.org/trac/ticket/8710)

+   **[engine] [用例]**

    添加了新参数`PoolEvents.reset.reset_state`到`PoolEvents.reset()`事件中，同时放置了一个将继续接受使用先前一组参数的事件钩子的弃用逻辑。这指示了关于重置正在进行的方式的各种状态信息，并且被用于允许在给定完整上下文的情况下进行自定义重置方案。

    在这个改变中，也包括了一个在 1.4 中回溯的修复，该修复重新启用了`PoolEvents.reset()`事件，以便在所有情况下继续进行，包括当`Connection`已经“重置”连接时。

    这两个更改共同允许使用`PoolEvents.reset()`事件来实现自定义重置方案，而不是使用`PoolEvents.checkin()`事件（其功能与以往一样）。

    参考资料：[#8717](https://www.sqlalchemy.org/trac/ticket/8717)

+   **[engine] [bug] [regression]**

    修复了一个问题，即当`Connection`关闭并且正在将其 DBAPI 连接返回到连接池时，在某些情况下不会调用`PoolEvents.reset()`事件钩子。

    当`Connection`已经在将连接返回到池的过程中在其 DBAPI 连接上发出`.rollback()`时，场景是它随后会指示连接池放弃执行自己的“重置”以节省额外的方法调用。但是，这会阻止在此钩子中使用自定义池重置方案，因为此类钩子根据定义正在执行的不仅仅是调用`.rollback()`，而且需要在所有情况下调用。这是在版本 1.4 中出现的一种退化。

    对于版本 1.4，`PoolEvents.checkin()`仍然可作为用于自定义“重置”实现的备用事件钩子。版本 2.0 将提供一个改进版本的`PoolEvents.reset()`，它将被调用用于额外的场景，例如终止 asyncio 连接，并且还传递有关重置的上下文信息，以允许对不同的重置方案作出响应以不同的方式处理不同的重置场景。

    此更改也被**回溯**到：1.4.43

    参考资料：[#8717](https://www.sqlalchemy.org/trac/ticket/8717)

### sql

+   **[sql] [bug]**

    修复了一个问题，它阻止`literal_column()`构造在`Select`构造的上下文中正常工作，以及其他可能生成“匿名标签”的地方，如果文字表达式包含可能干扰格式字符串的字符，例如括号，由于“匿名标签”的实现细节。

    此更改也被**回溯**到：1.4.43

    参考资料：[#8724](https://www.sqlalchemy.org/trac/ticket/8724)

### typing

+   **[typing] [bug]**

    更正了引擎和异步引擎包中的各种类型问题。

### postgresql

+   **[postgresql] [feature]**

    添加了新方法`Range.contains()` 和 `Range.contained_by()` 到新的 `Range` 数据对象中，这些方法与 PostgreSQL 的 `@>` 和 `<@` 操作符的行为相同，以及 `comparator_factory.contains()` 和 `comparator_factory.contained_by()` SQL 操作符方法。感谢 Lele Gaifax 提供的拉取请求。

    参考：[#8706](https://www.sqlalchemy.org/trac/ticket/8706)

+   **[postgresql] [usecase]**

    优化了对 PostgreSQL 后端的新 RANGE / MULTIRANGE 支持和更改 中描述的范围对象的新方法，以适应驱动程序特定的范围和多范围对象，更好地适应传统代码以及将结果从原始 SQL 结果集传递回新范围或多范围表达式时。

    参考：[#8690](https://www.sqlalchemy.org/trac/ticket/8690)

### mssql

+   **[mssql] [bug]**

    修复了与使用 SQL Server 方言的临时表时使用`Inspector.has_table()`相关的问题，这会导致某些 Azure 变体上失败，因为不支持那些服务器版本上的一个不必要的信息模式查询。感谢 Mike Barry 提供的拉取请求。

    此更改还**回溯**到：1.4.43

    参考：[#8714](https://www.sqlalchemy.org/trac/ticket/8714)

+   **[mssql] [bug] [reflection]**

    修复了与`Inspector.has_table()`相关的问题，当对使用 SQL Server 方言的视图使用时，错误地返回 `False`，这是由于 1.4 系列中的一个回归导致的，该系列在 SQL Server 上删除了对此的支持。这个问题在使用不同反射架构的 2.0 系列中不存在。添加了测试支持，以确保 `has_table()` 符合视图的规范。

    此更改还**回溯**到：1.4.43

    参考：[#8700](https://www.sqlalchemy.org/trac/ticket/8700)

### oracle

+   **[oracle] [bug]**

    修复了一个问题，其中包含需要在 Oracle 中用引号引起的字符的绑定参数名称，包括那些从同名数据库列自动生成的名称，在使用 Oracle 方言的“扩展参数”时不会被转义，导致执行错误。 Oracle 方言使用的绑定参数的通常“引用”不与“扩展参数”架构一起使用，因此使用了大范围字符的转义，现在使用了一个针对 Oracle 的字符/转义列表。

    此更改也**回溯**到：1.4.43

    参考：[#8708](https://www.sqlalchemy.org/trac/ticket/8708)

+   **[oracle] [bug]**

    修复了一个问题，在新的 ORM 类型声明映射中，没有实现在关系配置中使用`Optional[MyClass]`或类似形式（例如`MyClass | None`）的类型注释的能力，导致错误。 文档还为这种用例添加了关于关系配置的文档。

    此更改也**回溯**到：1.4.43

    参考：[#8744](https://www.sqlalchemy.org/trac/ticket/8744)

## 2.0.0b2

发布日期：2022 年 10 月 20 日

### orm

+   **[orm] [bug]**

    移除了在使用 ORM 启用的更新/删除时发出的警告，该警告首次出现在[#4073](https://www.sqlalchemy.org/trac/ticket/4073)中；这个警告实际上掩盖了一个场景，否则可能会根据实际列而为 ORM 映射的属性填充错误的 Python 值，因此移除了这个不建议使用的情况。在 2.0 中，ORM 启用的更新/删除使用“auto”作为“synchronize_session”，这应该会自动为任何给定的 UPDATE 表达式执行正确的操作。

    参考：[#8656](https://www.sqlalchemy.org/trac/ticket/8656)

### orm 声明

+   **[orm] [declarative] [usecase]**

    添加了对映射类也是`Generic`子类的支持，可以在语句和调用`inspect()`时指定为`GenericAlias`对象（例如`MyClass[str]`）。

    参考：[#8665](https://www.sqlalchemy.org/trac/ticket/8665)

+   **[orm] [declarative] [bug]**

    改进了`DeclarativeBase`类，以便与其他混入类（如`MappedAsDataclass`）结合使用时，类的顺序可以是任意顺序。

    参考：[#8665](https://www.sqlalchemy.org/trac/ticket/8665)

+   **[orm] [declarative] [bug]**

    修复了一个问题，在新的 ORM 类型声明映射中，没有实现在关系配置中使用`Optional[MyClass]`或类似形式（例如`MyClass | None`）的类型注释的能力，导致错误。 文档还为这种用例添加了关于关系配置的文档。

    参考：[#8668](https://www.sqlalchemy.org/trac/ticket/8668)

+   **[orm] [declarative] [bug]**

    修复了新的 dataclass 映射功能中的问题，当处理覆盖`mapped_column()`声明的混入时，传递给 dataclasses API 的参数有时可能被错误排序，导致初始化问题。

    参考：[#8688](https://www.sqlalchemy.org/trac/ticket/8688)

### sql

+   **[sql] [bug] [regression]**

    修复了新的“insertmanyvalues”功能中的 bug，其中包含使用`bindparam()`的子查询的 INSERT 在“insertmanyvalues”格式中无法正确呈现的问题。这直接影响了 psycopg2，因为“insertmanyvalues”在此驱动程序中无条件使用。

    参考：[#8639](https://www.sqlalchemy.org/trac/ticket/8639)

### 类型

+   **[typing] [bug]**

    修复了 pylance 严格模式下报告“实例变量覆盖类变量”的类型问题，当使用方法定义`__tablename__`、`__mapper_args__`或`__table_args__`时。

    参考：[#8645](https://www.sqlalchemy.org/trac/ticket/8645)

+   **[typing] [bug]**

    修复了 pylance 严格模式下报告“部分未知”数据类型的`mapped_column()`构造的类型问题。

    参考：[#8644](https://www.sqlalchemy.org/trac/ticket/8644)

### mssql

+   **[mssql] [bug]**

    由于 SQL Server pyodbc 更改 [#8177](https://www.sqlalchemy.org/trac/ticket/8177) 引起的回归问题已修复，现在默认使用`setinputsizes()`；对于 VARCHAR，如果字符大小大于 4000（或 2000，取决于数据），则会失败，因为传入的数据类型是 NVARCHAR，其限制为 4000 个字符，尽管 VARCHAR 可以处理无限字符。 当数据类型的大小> 2000 个字符时，现在还将传递额外的 pyodbc 特定的类型信息给`setinputsizes()`。 这个更改也适用于受此问题影响的`JSON`类型，用于大型 JSON 序列化。

    参考：[#8661](https://www.sqlalchemy.org/trac/ticket/8661)

+   **[mssql] [bug]**

    `Sequence` 构造恢复到了 1.4 系列之前的 DDL 行为，即创建一个没有额外参数的 `Sequence` 将会发出一个简单的 `CREATE SEQUENCE` 指令，**没有**任何额外的“起始值”参数。对于大多数后端来说，无论如何，这都是之前的工作方式；**然而**，对于 MS SQL Server，此数据库上的默认值是 `-2**63`；为了防止这个通常不实用的默认值在 SQL Server 上生效，应该提供 `Sequence.start` 参数。由于对于多年以来一直在 `IDENTITY` 上标准化的 SQL Server 来说，对 `Sequence` 的使用是不寻常的，希望这个变化影响最小。

    另请参阅

    Sequence 构造不再具有任何显式默认的 “start” 值；影响 MS SQL Server

    参考：[#7211](https://www.sqlalchemy.org/trac/ticket/7211)

## 2.0.0b1

发布日期：2022 年 10 月 13 日

### 常规

+   **[常规] [更改]**

    迁移代码库以删除所有之前标记为在 2.0 中移除的预先 2.0 版本行为和架构，包括但不限于：

    +   删除所有 Python 2 代码，最低版本现在为 Python 3.7

    +   `Engine` 和 `Connection` 现在使用新的 2.0 工作风格，其中包括 “autobegin”，库级别的自动提交已删除，子事务和 “branched” 连接已删除。

    +   结果对象使用 2.0 风格的行为；`Row` 完全是一个具有命名元组而没有 “映射” 行为，使用 `RowMapping` 来进行 “映射” 行为

    +   所有 Unicode 编码/解码架构已从 SQLAlchemy 中删除。所有现代的 DBAPI 实现都通过 Python 3 透明地支持 Unicode，因此已经删除了 `convert_unicode` 功能以及相关的在 DBAPI `cursor.description` 等中查找字节字符串的机制。

    +   从 `MetaData`，`Table` 和从以前可以引用 “绑定引擎”的所有 DDL/DML/DQL 元素中的 `.bind` 属性和参数

    +   独立的`sqlalchemy.orm.mapper()`函数已移除；所有经典映射应通过`registry.map_imperatively()`方法的`registry`进行。

    +   `Query.join()`方法不再接受字符串作为关系名称；使用`Class.attrname`作为连接目标的长期记录方法现在是标准的。

    +   `Query.join()`不再接受“aliased”和“from_joinpoint”参数。

    +   `Query.join()`不再在一个方法调用中接受多个连接目标链。

    +   `Query.from_self()`，`Query.select_entity_from()`和`Query.with_polymorphic()`已移除。

    +   `relationship.cascade_backrefs`参数现在必须保持其新默认值`False`；`save-update`级联不再沿着反向引用级联。

    +   `Session.future`参数必须始终设置为`True`。 `Session`的 2.0 风格事务模式现在始终生效。

    +   加载器选项不再接受属性名称的字符串。使用`Class.attrname`作为加载器选项目标的长期记录方法现在是标准的。

    +   遗留形式的`select()`已移除，包括`select([cols])`，`some_table.select()`的“whereclause”和关键参数。

    +   `Select`上的遗留“就地变异器”方法，如`append_whereclause()`，`append_order_by()`等已移除。

    +   移除了非常古老的“dbapi_proxy”模块，早期 SQLAlchemy 版本中用于在原始 DBAPI 连接上提供透明连接池。

    参考：[#7257](https://www.sqlalchemy.org/trac/ticket/7257)

+   **[通用] [更改]**

    `Query.instances()`方法已弃用。该方法的行为约定，即可以通过任意结果集迭代对象，早已过时且不再测试。可以使用类似:meth`.Select.from_statement`或`aliased()`的构造来返回对象。

### 平台

+   **[平台] [特性]**

    SQLAlchemy 的 C 扩展已被全部使用 Cython 编写的新实现所取代。与以前的 C 扩展一样，针对许多常见平台的预构建轮文件可在 pypi 上获得，因此构建不是问题。对于自定义构建，`python setup.py build_ext` 与以前一样工作，只需额外的 Cython 安装即可。`pyproject.toml` 现在也是源代码的一部分，当使用 pip 时将建立正确的构建依赖关系。

    参见

    C 扩展现已移植到 Cython

    参考：[#7256](https://www.sqlalchemy.org/trac/ticket/7256)

+   **[platform] [change]**

    SQLAlchemy 的源代码构建和安装现在包括一个 `pyproject.toml` 文件，以完全支持 [**PEP 517**](https://peps.python.org/pep-0517/)。

    参见

    安装现在完全支持 pep-517

    参考：[#7311](https://www.sqlalchemy.org/trac/ticket/7311)

### orm

+   **[orm] [feature] [sql]**

    添加了对所有支持 RETURNING 的包含方言的新功能，称为“insertmanyvalues”。这是“fast executemany”功能的一般化，该功能首先在 psycopg2 驱动程序中的 1.4 版本中引入，详情请参阅 ORM Batch inserts with psycopg2 now batch statements with RETURNING in most cases，它允许 ORM 将 INSERT 语句批处理到一个更高效的 SQL 结构中，同时仍能够使用 RETURNING 检索新生成的主键和 SQL 默认值。

    此功能现在适用于支持 RETURNING 以及多个 VALUES 构造用于 INSERT 的许多方言，包括所有 PostgreSQL 驱动程序，SQLite，MariaDB，MS SQL Server。另外，Oracle 方言还使用本机 cx_Oracle 或 OracleDB 功能获得相同的能力。

    参考：[#6047](https://www.sqlalchemy.org/trac/ticket/6047)

+   **[orm] [feature]**

    添加了新参数 `AttributeEvents.include_key`，它将包含诸如 `__setitem__()`（例如 `obj[key] = value`）和 `__delitem__()`（例如 `del obj[key]`）等操作的字典或列表键，使用一个新的关键字参数“key”或“keys”，取决于事件，例如 `AttributeEvents.append.key`，`AttributeEvents.bulk_replace.keys`。这允许事件处理程序考虑传递给操作的键，对于与 `MappedCollection` 一起工作的字典操作尤其重要。

    参考：[#8375](https://www.sqlalchemy.org/trac/ticket/8375)

+   **[orm] [feature]**

    添加了新参数`Operators.op.python_impl`，可从`Operators.op()`以及直接使用`custom_op`构造函数时使用，允许提供一个在 Python 中进行评估的函数，以及自定义 SQL 运算符。当操作符对象在两侧使用普通 Python 对象作为操作数时，此评估函数将成为使用的实现，并且特别兼容与 ORM 启用的 INSERT、UPDATE 和 DELETE 语句一起使用的`synchronize_session='evaluate'`选项。

    参考：[#3162](https://www.sqlalchemy.org/trac/ticket/3162)

+   **[orm] [feature]**

    `Session`（以及间接`AsyncSession`）现在具有新的状态跟踪功能，将主动捕获在特定事务方法进行时发生的任何意外状态更改。这是为了允许在线程不安全的方式中使用`Session`的情况，其中事件钩子或类似的可能在操作中调用意外方法，以及在其他并发情况下（如 asyncio 或 gevent）在首次发生非法访问时引发信息性消息，而不是默默传递导致由于`Session`处于无效状态而导致的次要故障。

    另请参阅

    当检测到非法并发或重入访问时，会主动引发会话

    参考：[#7433](https://www.sqlalchemy.org/trac/ticket/7433)

+   **[orm] [feature]**

    当与 Python `dataclass`一起使用时，`composite()`映射构造现在支持值的自动解析；不再需要实现`__composite_values__()`方法，因为此方法是从数据类的检查中派生的。

    此外，由`composite`映射的类现在支持排序比较操作，例如`<`、`>=`等。

    请参阅复合列类型中的新文档以获取示例。

+   **[orm] [feature]**

    添加了对`selectinload()`和`immediateload()`加载器选项的非常实验性功能，名为`selectinload.recursion_depth` / `immediateload.recursion_depth`，它允许单个加载器选项自动递归到自引用关系中。设置为指示深度的整数，并且也可以设置为-1，以指示继续加载直到找不到更多层级为止。对`selectinload()`和`immediateload()`进行了主要的内部更改，以便使此功能在继续正确使用编译缓存的同时工作，并且不使用任意递归，因此支持任何深度级别（尽管会发出相同数量的查询）。这对于必须完全急切加载的自引用结构可能会有用，例如在使用 asyncio 时。

    当检测到相关对象加载中的过度递归深度时，还会发出警告，该警告也会在加载器选项以任意长度连接在一起时（即，不使用新的`recursion_depth`选项）发出。此操作继续使用大量内存，并且性能极差；当检测到此条件时，缓存会被禁用，以防止缓存被任意语句淹没。

    参考：[#8126](https://www.sqlalchemy.org/trac/ticket/8126)

+   **[orm] [feature]**

    添加了新参数`Session.autobegin`，当设置为`False`时，将阻止`Session`隐式地开始事务。必须首先显式调用`Session.begin()`方法，以便继续进行操作，否则在任何操作本应自动开始时都会引发错误。此选项可用于创建一个“安全”的`Session`，该会话不会隐式启动新事务。

    作为这一变化的一部分，还添加了一个新的状态变量 `origin`，可能对事件处理代码有用，以了解特定 `SessionTransaction` 的来源。

    参考：[#6928](https://www.sqlalchemy.org/trac/ticket/6928)

+   **[orm] [特性]**

    使用包含 `ForeignKey` 引用的 `Column` 对象的声明性混合物不再需要使用 `declared_attr()` 来实现此映射；当列应用于声明的映射时，`ForeignKey` 对象与 `Column` 本身一起复制。

+   **[orm] [用例]**

    在 `load_only()` 加载器选项中添加了 `load_only.raiseload` 参数，以便未加载的属性可能具有“raise”行为而不是延迟加载。以前没有真正直接使用 `load_only()` 选项来实现这一点。

+   **[orm] [更改]**

    为了更好地适应显式类型，一些通常在内部构造但有时也可见于消息传递和类型化的 ORM 构造的名称已更改为更简洁的名称，这些名称也与构造函数的名称（大小写不同）匹配，在所有情况下都保留了旧名称的别名以备将来使用：

    +   `RelationshipProperty` 成为主要名称 `Relationship` 的别名，始终由 `relationship()` 函数构建

    +   `SynonymProperty` 成为主要名称 `Synonym` 的别名，始终由 `synonym()` 函数构建

    +   `CompositeProperty`现在成为主要名称`Composite`的别名，始终由`composite()`函数构建。

+   **[orm] [change]**

    为了与突出的 ORM 概念`Mapped`保持一致，基于字典的集合`attribute_mapped_collection()`、`column_mapped_collection()`和`MappedCollection`的名称已更改为`attribute_keyed_dict()`、`column_keyed_dict()`和`KeyFuncDict`，使用“dict”短语以减少与术语“mapped”的混淆。旧名称将无限期保留，没有删除计划。

    参考：[#8608](https://www.sqlalchemy.org/trac/ticket/8608)

+   **[orm] [bug]**

    所有`Result`对象现在在硬关闭后使用时都会一致地引发`ResourceClosedError`，包括在调用类似`Result.first()`和`Result.scalar()`这样的“单行或值”方法后发生的“硬关闭”。这已经是基于`CursorResult`的最常见的 Core 语句执行返回的结果对象的行为，因此这种行为并不新鲜。然而，这一变化已经扩展到正确地适应使用 2.0 风格 ORM 查询时返回的 ORM“过滤”结果对象，以前这些对象会以“软关闭”方式返回空结果，或者根本不会真正“软关闭”并会继续从底层游标中产生结果。

    作为这一变更的一部分，还将`Result.close()`添加到基础`Result`类中，并为 ORM 使用的过滤结果实现了它，这样就可以在使用`yield_per`执行选项时调用底层`CursorResult`上的`CursorResult.close()`方法，在获取剩余的 ORM 结果之前关闭服务器端游标。这对于核心结果集已经可用，但此变更也使其适用于 2.0 风格的 ORM 结果。

    此更改也**回溯**到：1.4.27

    参考：[#7274](https://www.sqlalchemy.org/trac/ticket/7274)

+   **[orm] [bug]**

    修复了`registry.map_declaratively()`方法返回内部“映射器配置”对象而不是 API 文档中所述的`Mapper`对象的问题。

+   **[orm] [bug]**

    修复了至少在 1.3 版本中出现的性能回归，如果不是更早（在 1.0 之后的某个时候），那么从连接的继承子类加载延迟列（那些明确映射为`defer()`的列，而不是已过期的未延迟列），将不会使用“优化”查询，该查询仅查询包含未加载列的直接表，而是运行完整的 ORM 查询，该查询会为所有基本表发出 JOIN，当仅从子类加载列时，这是不必要的。

    参考：[#7463](https://www.sqlalchemy.org/trac/ticket/7463)

+   **[orm] [bug]**

    `Load`对象及相关加载策略模式的内部大部分已经重写，以利用现在仅支持属性绑定路径而不是字符串的事实。重写希望能更直接地解决加载策略系统中的新用例和微妙问题。

    参考：[#6986](https://www.sqlalchemy.org/trac/ticket/6986)

+   **[orm] [bug]**

    对“延迟加载”/“仅加载”一组策略选项进行了改进，其中如果一个对象从一个查询中的两个不同逻辑路径加载，那么至少有一个选项配置为填充的属性将在所有情况下被填充，即使该对象的其他加载路径没有设置此选项。以前，基于随机性来确定哪个“路径”首先处理对象。

    参考：[#8166](https://www.sqlalchemy.org/trac/ticket/8166)

+   **[orm] [bug]**

    修复了在针对联合继承子类创建语句时启用 ORM 的 UPDATE 时出现的问题，仅更新本地表列，其中“fetch”同步策略不会为使用 RETURNING 进行获取同步的数据库呈现正确的 RETURNING 子句。还调整了在 UPDATE FROM 和 DELETE FROM 语句中使用的 RETURNING 策略。

    参考：[#8344](https://www.sqlalchemy.org/trac/ticket/8344)

+   **[orm] [bug] [asyncio]**

    从`begin`和`begin_nested`中删除了未使用的`**kw`参数。这些 kw 没有被使用，似乎是错误地添加到 API 中的。

    参考：[#7703](https://www.sqlalchemy.org/trac/ticket/7703)

+   **[orm] [bug]**

    更改了`attribute_mapped_collection()`和`column_mapped_collection()`（现在称为`attribute_keyed_dict()`和`column_keyed_dict()`)在填充字典时使用的属性访问方法，断言对象上用作字典键的数据值实际上存在，并且不是因为属性从未被分配而使用“None”。这用于防止在通过反向引用进行分配时错误地为键分配 None，其中对象上的“键”属性尚未被分配。

    由于此处的失败模式是一种通常不会持续到数据库的瞬态条件，并且很容易通过类的构造函数根据分配参数的顺序产生，因此很有可能许多应用程序已经包含了这种行为，而这种行为被悄悄地忽略了。为了适应现在引发此错误的应用程序，还添加了一个新参数 `attribute_keyed_dict.ignore_unpopulated_attribute` 到 `attribute_keyed_dict()` 和 `column_keyed_dict()`，这个参数将导致错误的反向引用赋值被跳过。

    参考：[#8372](https://www.sqlalchemy.org/trac/ticket/8372)

+   **[orm] [bug]**

    添加了新参数 `AbstractConcreteBase.strict_attrs` 到 `AbstractConcreteBase` 声明混合类。这个参数的效果是，子类上的属性范围正确地限制在声明每个属性的子类中，而不是之前的行为，其中整个层次结构的所有属性都应用到基本的“抽象”类上。这样会产生更清晰、更正确的映射，子类不再具有仅对同级类有用的属性。此参数的默认值为 False，这保留了先前的行为不变；这是为了支持在查询中明确使用这些属性的现有代码。要迁移到更新的方法，根据需要将显式属性应用到抽象基类中。

    参考：[#8403](https://www.sqlalchemy.org/trac/ticket/8403)

+   **[orm] [bug]**

    `defer()` 在处理主键和“多态鉴别器”列时的行为已经修订，以使这些列不再是可延迟的，无论是明确指定还是使用诸如 `defer('*')` 这样的通配符。先前，通配符延迟不会加载主键/多态列，这导致在所有情况下都出现错误，因为 ORM 依赖于这些列来生成对象标识。对主键列的显式延迟行为不变，因为这些延迟已经被隐式忽略。

    参考：[#7495](https://www.sqlalchemy.org/trac/ticket/7495)

+   **[orm] [bug]**

    修复了`Mapper.eager_defaults` 参数行为中的错误，使得仅在表定义中存在客户端 SQL 默认值或 onupdate 表达式时，ORM 为行执行 INSERT 或 UPDATE 时触发 RETURNING 或 SELECT 的操作。之前，仅服务器端默认值作为表 DDL 的一部分或服务器端 onupdate 表达式会触发此次提取，尽管客户端 SQL 表达式在渲染提取时也会被包含在内。

    参考：[#7438](https://www.sqlalchemy.org/trac/ticket/7438)

### 引擎

+   **[引擎] [特性]**

    `DialectEvents.handle_error()` 事件现已从 `EngineEvents` 套件移至 `DialectEvents` 套件，并且现在参与连接池的“预 ping”事件，对于那些使用断开代码来检测数据库是否存活的方言。这使得最终用户代码能够更改“预 ping”的状态。请注意，这不包括包含本地“ping”方法的方言，如 psycopg2 或大多数 MySQL 方言。

    参考：[#5648](https://www.sqlalchemy.org/trac/ticket/5648)

+   **[引擎] [特性]**

    `ConnectionEvents.set_connection_execution_options()` 和 `ConnectionEvents.set_engine_execution_options()` 事件钩子现在允许对给定的选项字典进行就地修改，新内容将作为最终执行选项接收。以前，不支持对字典进行就地修改。

+   **[引擎] [用例]**

    将`create_engine.isolation_level` 参数泛化到基本方言，因此不再依赖于单独的方言。该参数为所有新数据库连接的“隔离级别”设置提供了设置，一旦连接池创建它们，该值就会保持设置而不是在每次 checkin 时重置。

    `create_engine.isolation_level` 参数在功能上与通过 `Engine.execution_options.isolation_level` 参数使用 `Engine.execution_options()` 设置相当。区别在于前者设置隔离级别只会在创建连接时执行一次，后者在每次连接检出时设置和重置给定级别。

    参考：[#6342](https://www.sqlalchemy.org/trac/ticket/6342)

+   **[engine] [change]**

    关于引擎和方言的一些小的 API 更改：

    +   `Dialect.set_isolation_level()`、`Dialect.get_isolation_level()`、:meth: 方言方法将始终传递原始的 DBAPI 连接

    +   `Connection` 和 `Engine` 类不再共享基础的 `Connectable` 超类，该超类已被移除。

    +   添加了一个新的接口类`PoolProxiedConnection` - 这是熟悉的`_ConnectionFairy` 类的公共接口，尽管它是一个私有类。

    参考：[#7122](https://www.sqlalchemy.org/trac/ticket/7122)

+   **[engine] [bug] [regression]**

    修复了当结果完全耗尽时，`CursorResult.fetchmany()` 方法未能自动关闭服务器端游标（即在使用 `stream_results` 或 `yield_per` 时，无论是核心还是 ORM 导向的结果）的回归问题。

    此更改也被**回溯**至：1.4.27

    参考：[#7274](https://www.sqlalchemy.org/trac/ticket/7274)

+   **[engine] [bug]**

    修复了未来 `Engine` 中的问题，当调用 `Engine.begin()` 并进入上下文管理器时，如果实际的 BEGIN 操作由于某种原因失败，例如事件处理程序引发异常，则连接不会关闭；未来版本的引擎未测试此用例。请注意，“未来”上下文管理器在实际输入上下文管理器之前不会运行 “BEGIN” 操作。这与立即运行 “BEGIN” 操作的传统版本不同。

    此更改也被**回溯**至：1.4.27

    参考：[#7272](https://www.sqlalchemy.org/trac/ticket/7272)

+   **[engine] [bug]**

    当 `pool_size=0` 时，`QueuePool` 现在会忽略 `max_overflow`，从而正确地使池在所有情况下都是无限的。

    参考：[#8523](https://www.sqlalchemy.org/trac/ticket/8523)

+   **[engine] [bug]**

    为了提高安全性，当调用 `str(url)` 时，`URL` 对象现在默认使用密码混淆。要将 URL 字符串化为明文密码，可以使用 `URL.render_as_string()`，将 `URL.render_as_string.hide_password` 参数传递为 `False`。感谢我们的贡献者提供了此拉取请求。

    另请参阅

    str(engine.url) 现在默认混淆密码

    参考：[#8567](https://www.sqlalchemy.org/trac/ticket/8567)

+   **[engine] [bug]**

    `Inspector.has_table()` 方法现在将一致地检查给定名称的视图和表格。先前，此行为取决于方言，其中 PostgreSQL、MySQL/MariaDB 和 SQLite 支持它，而 Oracle 和 SQL Server 不支持它。第三方方言也应确保它们的 `Inspector.has_table()` 方法搜索给定名称的视图和表格。

    参考：[#7161](https://www.sqlalchemy.org/trac/ticket/7161)

+   **[engine] [bug]**

    在`Result.columns()`方法中修复了一个问题，在某些情况下，特别是 ORM 结果对象的情况下，调用带有单个索引的`Result.columns()`可能导致`Result`生成标量对象而不是`Row`对象，就像调用了`Result.scalars()`方法一样。在 SQLAlchemy 1.4 中，此场景会发出警告，指出行为将在 SQLAlchemy 2.0 中更改。

    引用：[#7953](https://www.sqlalchemy.org/trac/ticket/7953)

+   **[engine] [错误]**

    将`DefaultGenerator`对象（例如`Sequence`）传递给`Connection.execute()`方法已弃用，因为此方法被类型化为返回一个`CursorResult`对象，而不是纯标量值。应改用`Connection.scalar()`方法，该方法已经重写了新的内部代码路径以适用于调用 SELECT 以获取默认生成对象而不经过`Connection.execute()`方法。

+   **[engine] [已移除]**

    从`create_engine()`中移除了之前弃用的`case_sensitive`参数，这只会影响 Core-only 结果集行中字符串列名称的查找；它不会影响 ORM 的行为。`case_sensitive`指向的有效行为保持其默认值`True`，意味着在`row._mapping`中查找的字符串名称将与大小写敏感地匹配，就像任何其他 Python 映射一样。

    请注意，`case_sensitive`参数与控制大小写敏感性、引用和“名称规范化”（即转换为将所有大写字母视为大小写不敏感的数据库）DDL 标识符名称的一般主题没有任何关系，这仍然是 SQLAlchemy 的一个常规核心功能。

+   **[engine] [已移除]**

    移除了已弃用的旧版包`sqlalchemy.databases`。请改用`sqlalchemy.dialects`。

    引用：[#7258](https://www.sqlalchemy.org/trac/ticket/7258)

+   **[engine] [已弃用]**

    `create_engine.implicit_returning` 参数已经在`create_engine()`函数中弃用；该参数仅在`Table`对象上保留。该参数最初旨在在 SQLAlchemy 首次开发时启用“implicit returning”功能，但默认情况下未启用。在现代用法下，没有理由禁用该参数，并且观察到它会引起混乱，因为它会降低性能，使 ORM 更难以检索最近插入的服务器默认值。该参数仅在`Table`上保留，以特别适应使 RETURNING 不可行的数据库级边缘情况，目前唯一的示例是 SQL Server 的限制，即不得在具有 INSERT 触发器的表上使用 INSERT RETURNING。

    参考：[#6962](https://www.sqlalchemy.org/trac/ticket/6962)

### sql

+   **[sql] [feature]**

    > 添加了长期要求的不区分大小写的字符串操作符`ColumnOperators.icontains()`，`ColumnOperators.istartswith()`，`ColumnOperators.iendswith()`。这些操作符生成不区分大小写的 LIKE 组合（在 PostgreSQL 上使用 ILIKE，在所有其他后端上使用 LOWER()函数），以补充现有的 LIKE 组合操作符`ColumnOperators.contains()`，`ColumnOperators.startswith()`等。对于实现这些新方法的 Matias Martinez Rebori 的细致和完整的工作表示巨大的感谢。

    参考：[#3482](https://www.sqlalchemy.org/trac/ticket/3482)

+   **[sql] [feature]**

    在所有`FromClause`对象的`FromClause.c`集合中添加了新的语法，允许传递键的元组给`__getitem__()`，以及支持`select()`构造处理直接处理结果类似元组的集合，允许`select(table.c['a', 'b', 'c'])`语法成为可能。返回的子集合本身也是一个`ColumnCollection`，现在也可以直接被`select()`和类似方法消耗。

    参见

    设置 COLUMNS 和 FROM 子句

    参考：[#8285](https://www.sqlalchemy.org/trac/ticket/8285)

+   **[sql] [功能]**

    添加了新的与后端无关的`Uuid`数据类型，从 PostgreSQL 方言泛化到核心类型，以及将`UUID`从 PostgreSQL 方言迁移到核心类型。SQL Server 的`UNIQUEIDENTIFIER`数据类型也变成了一个 UUID 处理数据类型。感谢 Trevor Gross 的帮助。

    参考：[#7212](https://www.sqlalchemy.org/trac/ticket/7212)

+   **[sql] [功能]**

    向`sqlalchemy.`模块命名空间添加了`Double`、`DOUBLE`、`DOUBLE_PRECISION`数据类型，用于显式使用 double/double precision 以及通用的“double”数据类型。使用`Double`来提供通用支持，根据不同后端需要解析为 DOUBLE/DOUBLE PRECISION/FLOAT。

    参考：[#5465](https://www.sqlalchemy.org/trac/ticket/5465)

+   **[sql] [用例]**

    更改了 `Insert` 构造的编译机制，使得“自动递增主键”列值将通过 `cursor.lastrowid` 或 RETURNING 获取，即使它存在于参数集中或在 `Insert.values()` 方法中作为普通绑定值，用于特定后端上已知在明确传递 NULL 时仍生成自动递增值的单行 INSERT 语句。这恢复了 1.3 系列中的行为，用于分离的参数集以及 `Insert.values()` 方法。在 1.4 中，参数集行为无意中更改为不再执行此操作，但`Insert.values()` 方法仍会获取自动递增值，直到 1.4.21，其中 [#6770](https://www.sqlalchemy.org/trac/ticket/6770) 再次无意中更改了行为，因为此用例从未得到覆盖。

    现在已定义行为为“工作”，以适应数据库（如 SQLite、MySQL 和 MariaDB 等）忽略显式 NULL 主键值并仍调用自动递增生成器的情况。

    引用: [#7998](https://www.sqlalchemy.org/trac/ticket/7998)

+   **[sql] [usecase]**

    当使用 PostgreSQL、MySQL、MariaDB、MSSQL、Oracle 方言提供的 SQL 编译器与 `literal_binds` 一起使用时，添加了修改后的 ISO-8601 渲染（即将 T 转换为空格的 ISO-8601），对于 Oracle，ISO 格式被包装在适当的 TO_DATE() 函数调用内。先前，此渲染未针对方言特定编译实现。

    另请参阅

    DATE、TIME、DATETIME 数据类型现在在所有后端上支持文本渲染

    引用: [#5052](https://www.sqlalchemy.org/trac/ticket/5052)

+   **[sql] [usecase]**

    添加了新的参数 `HasCTE.add_cte.nest_here` 到 `HasCTE.add_cte()`，该参数将在父语句级别“嵌套”给定的 `CTE`。该参数等同于使用 `HasCTE.cte.nesting` 参数，但在某些情况下可能更直观，因为它允许同时设置嵌套属性和 CTE 的显式级别。

    `HasCTE.add_cte()` 方法还接受多个 CTE 对象。

    参考：[#7759](https://www.sqlalchemy.org/trac/ticket/7759)

+   **[sql] [bug]**

    当使用 `Select.select_from()` 方法在 `select()` 构造上建立 FROM 子句时，这些子句现在将首先在渲染的 SELECT 的 FROM 子句中呈现，这有助于保持子句的顺序，就像它们传递给 `Select.select_from()` 方法本身时一样，而不受这些子句也在查询的其他部分提及的影响。如果 `Select` 的其他元素也生成 FROM 子句，例如列子句或 WHERE 子句，这些将在由 `Select.select_from()` 提供的子句之后呈现，假设它们未明确传递给 `Select.select_from()`。这一改进在某些情况下非常有用，其中特定数据库基于 FROM 子句的特定顺序生成理想的查询计划，并允许完全控制 FROM 子句的顺序。

    参考：[#7888](https://www.sqlalchemy.org/trac/ticket/7888)

+   **[sql] [bug]**

    `Enum.length` 参数，用于为非本地枚举类型的 `VARCHAR` 列设置长度，在为 `VARCHAR` 数据类型发出 DDL 时现在无条件使用，包括当为目标后端设置了 `Enum.native_enum` 参数为 `True` 时继续使用 `VARCHAR` 的情况。在这种情况下，先前该参数将被错误地忽略。先前为此情况发出的警告现已移除。

    参考：[#7791](https://www.sqlalchemy.org/trac/ticket/7791)

+   **[sql] [bug]**

    Python 整数的原地类型检测，如表达式`literal(25)`，现在也将应用基于值的适配，以适应 Python 大整数，其中确定的数据类型将是`BigInteger`而不是`Integer`。这适用于诸如 asyncpg 之类的方言，其既将隐式类型信息发送给驱动程序，又对数值刻度敏感。

    参考：[#7909](https://www.sqlalchemy.org/trac/ticket/7909)

+   **[sql] [bug]**

    为所有“创建”/“删除”结构（包括`CreateSequence`、`DropSequence`、`CreateIndex`、`DropIndex`等）添加了`if_exists`和`if_not_exists`参数，允许在 DDL 中呈现通用的“IF EXISTS”/“IF NOT EXISTS”短语。感谢 Jesse Bakker 提供的拉取请求。

    参考：[#7354](https://www.sqlalchemy.org/trac/ticket/7354)

+   **[sql] [bug]**

    改进了 SQL 二进制表达式的构造，以允许针对相同的关联操作符进行非常长的表达式，而无需特殊步骤以避免高内存使用和过度递归深度。现在，一个特定的二元操作`A op B`可以与另一个元素`op C`连接，结果结构将被“平铺”，以使表示以及 SQL 编译不需要递归。

    此更改的一个影响是使用 SQL 函数的字符串连接表达式现在变得“平坦”，例如，MySQL 现在将呈现`concat('x', 'y', 'z', ...)`而不是将两个元素函数嵌套在一起的`concat(concat('x', 'y'), 'z')`。重写字符串连接运算符的第三方方言将需要实现一个新方法`def visit_concat_op_expression_clauselist()`，以配合现有的`def visit_concat_op_binary()`方法。

    参考：[#7744](https://www.sqlalchemy.org/trac/ticket/7744)

+   **[sql] [bug]**

    使用“/”和“//”运算符实现了对“truediv”和“floordiv”的完全支持。`Integer`类型的两个表达式之间的“truediv”操作现在被视为`Numeric`类型，并且方言级别的编译将根据方言特定的基础将右操作数转换为数字类型，以确保实现 truediv。对于 floordiv，还添加了转换，对于那些默认情况下不执行 floordiv 的数据库（如 MySQL、Oracle），在这种情况下还会渲染`FLOOR()`函数，以及右操作数不是整数的情况（对于 PostgreSQL 等其他数据库也是需要的）。

    此更改解决了不同后端上除法运算符行为不一致的问题，并修复了 Oracle 上整数除法无法获取结果的问题，因为输出类型处理程序不合适的问题。

    另见

    Python 除法运算符对所有后端执行真除法；添加了地板除法

    参考：[#4926](https://www.sqlalchemy.org/trac/ticket/4926)

+   **[sql] [bug]**

    在编译器中增加了额外的查找步骤，用于跟踪所有的 FROM 子句，这些子句是表，可能在多个模式中共享具有相同名称的情况，其中一个模式是隐式的“默认”模式；在这种情况下，当在没有模式限定符的情况下引用该名称时，编译器级别将为表名称生成一个匿名别名，以消除两个（或更多）名称的歧义。还考虑了使用服务器检测到的“默认模式名称”值对通常未限定名称进行模式限定的方法，但是这种方法不适用于 Oracle，SQL Server 也不接受，而且不适用于 PostgreSQL 搜索路径中的多个条目。在此解决的名称冲突问题已被确认至少影响到 Oracle、PostgreSQL、SQL Server、MySQL 和 MariaDB。

    参考：[#7471](https://www.sqlalchemy.org/trac/ticket/7471)

+   **[sql] [bug]**

    从值的类型确定 SQL 类型的 Python 字符串值，主要是当使用 `literal()` 时，现在将应用 `String` 类型，而不是 `Unicode` 数据类型，对于使用 Python `str.isascii()` 测试为“ascii only”的 Python 字符串值。如果字符串不是 `isascii()`，则将绑定 `Unicode` 数据类型，这在以前所有字符串检测中都使用了。这种行为**仅适用于使用 ``literal()`` 或其他没有现有数据类型的上下文的数据类型的就地检测**，通常不适用于正常的 `Column` 比较操作，其中正在比较的 `Column` 的类型始终优先。

    在诸如 SQL Server 等后端中，使用`Unicode`数据类型可以确定文字字符串的格式化方式，其中文字值（即使用 `literal_binds`）将呈现为 `N'<value>'` 而不是 `'value'`。对于常规绑定值处理，`Unicode`数据类型还可能对传递值到 DBAPI 产生影响，再次以 SQL Server 为例，pyodbc 驱动程序支持使用 setinputsizes 模式，它将以不同方式处理 `String` 与 `Unicode`。

    参考：[#7551](https://www.sqlalchemy.org/trac/ticket/7551)

+   **[sql] [bug]**

    `array_agg`现在将数组维度设置为 1。改进了对`ARRAY`的处理，以接受`None`值作为多维数组的值。

    参考：[#7083](https://www.sqlalchemy.org/trac/ticket/7083)

### 架构

+   **[schema] [feature]**

    在 `ExecutableDDLElement` 类（从 `DDLElement` 重命名）实现的“条件 DDL”系统上进行了扩展，直接可用于 `SchemaItem` 构造，如 `Index`、`ForeignKeyConstraint` 等，使得用于生成这些元素的条件逻辑包含在默认的 DDL 发射过程中。这个系统也可以被 Alembic 的未来版本支持，以支持所有模式管理系统中的条件 DDL 元素。

    另请参阅

    新的条件 DDL 用于约束和索引

    参考：[#7631](https://www.sqlalchemy.org/trac/ticket/7631)

+   **[模式] [用例]**

    在 `DropConstraint` 构造中添加了参数 `DropConstraint.if_exists`，这将导致“IF EXISTS” DDL 被添加到 DROP 语句中。这个短语不被所有数据库接受，如果数据库不支持它，该操作将在一个单独的 DDL 语句的范围内失败，因为在这个范围内没有类似的兼容回退。感谢 Mike Fiedler 的拉取请求。

    参考：[#8141](https://www.sqlalchemy.org/trac/ticket/8141)

+   **[模式] [用例]**

    为所有包含不同的 CREATE 或 DROP 步骤的 `SchemaItem` 对象实现了 DDL 事件钩子 `DDLEvents.before_create()`、`DDLEvents.after_create()`、`DDLEvents.before_drop()`、`DDLEvents.after_drop()`，当该步骤被调用为一个独立的 SQL 语句时，包括 `ForeignKeyConstraint`、`Sequence`、`Index` 和 PostgreSQL 的 `ENUM`。

    参考：[#8394](https://www.sqlalchemy.org/trac/ticket/8394)

+   **[schema] [performance]**

    重构了模式反射 API，以允许参与的方言利用高性能的批量查询来一次反射多个表的模式，使用数量级较少的查询。新的性能特性首先针对 PostgreSQL 和 Oracle 后端，可以应用于使用 SELECT 查询反映表的系统目录表的任何方言。该变化还包括对`Inspector`对象的新 API 特性和行为改进，包括像`Inspector.has_table()`、`Inspector.get_table_names()`等方法的一致、缓存行为，以及新方法`Inspector.has_schema()`和`Inspector.has_index()`。

    另见

    数据库反射的主要架构、性能和 API 增强 - 完整背景

    参考：[#4379](https://www.sqlalchemy.org/trac/ticket/4379)

+   **[schema] [bug]**

    当使用`Table.include_columns`参数排除后发现仍然是这些约束的一部分的列时，关于索引或唯一约束的反射发出的警告已被移除。当使用`Table.include_columns`参数时，应该预期生成的`Table`构造将不包括依赖于被省略列的约束。这个变化是对[#8100](https://www.sqlalchemy.org/trac/ticket/8100)作出的回应，该问题修复了`Table.include_columns`与依赖于被省略列的外键约束的一起使用的情况，其中使用案例表明省略此类约束是可以预期的。

    参考：[#8102](https://www.sqlalchemy.org/trac/ticket/8102)

+   **[schema] [postgresql]**

    对`Constraint`对象添加了对评论的支持，包括 DDL 和反射；该字段已添加到基本的`Constraint`类和相应的构造函数中，但目前只有 PostgreSQL 是支持该功能的后端。请参阅参数，如`ForeignKeyConstraint.comment`，`UniqueConstraint.comment`或`CheckConstraint.comment`。

    参考：[#5677](https://www.sqlalchemy.org/trac/ticket/5677)

+   **[schema] [mariadb] [mysql]**

    添加对 MySQL 和 MariaDB 分区和示例页面的支持反映选项。这些选项存储在表方言选项字典中，因此以下关键字需要根据后端添加`mysql_`或`mariadb_`前缀。支持的选项包括：

    +   `stats_sample_pages`

    +   `partition_by`

    +   `partitions`

    +   `subpartition_by`

    当从数据库加载表时，这些选项也会反映出来，并将填充表`Table.dialect_options`。感谢 Ramon Will 的拉取请求。

    参考：[#4038](https://www.sqlalchemy.org/trac/ticket/4038)

### typing

+   **[typing] [improvement]**

    `TypeEngine.with_variant()`方法现在返回原始`TypeEngine`对象的副本，而不是将其包装在`Variant`类中，该类实际上已被移除（导入符号仍保留以向后兼容可能测试此符号的代码）。虽然以前的方法保持了 Python 中的行为，但保持原始类型允许更清晰的类型检查和调试。

    `TypeEngine.with_variant()`还可以在每次调用时接受多个方言名称，特别是对于相关的后端名称，如`"mysql", "mariadb"`。

    另请参阅

    “with_variant()”克隆原始 TypeEngine 而不是更改类型

    参考：[#6980](https://www.sqlalchemy.org/trac/ticket/6980)

### postgresql

+   **[postgresql] [feature]**

    添加了新的 PostgreSQL `DOMAIN` 数据类型，其遵循与 PostgreSQL `ENUM` 相同的 CREATE TYPE / DROP TYPE 行为。非常感谢 David Baumgold 的努力。

    另见

    `DOMAIN`

    参考：[#7316](https://www.sqlalchemy.org/trac/ticket/7316)

+   **[postgresql] [usecase] [asyncpg]**

    添加了可重写方法 `PGDialect_asyncpg.setup_asyncpg_json_codec` 和 `PGDialect_asyncpg.setup_asyncpg_jsonb_codec`，用于在使用 asyncpg 时注册这些数据类型所需的 JSON/JSONB 编解码器。变更之处在于将方法拆分为单独的可重写方法，以支持需要修改或禁用这些特定编解码器设置的第三方方言。

    此变更还被 **回溯** 到：1.4.27

    参考：[#7284](https://www.sqlalchemy.org/trac/ticket/7284)

+   **[postgresql] [usecase]**

    为 `ARRAY` 和 `ARRAY` 数据类型添加了文字类型渲染。通用的字符串化将使用方括号进行渲染，例如 `[1, 2, 3]`，而 PostgreSQL 特定的将使用 ARRAY 文字，例如 `ARRAY[1, 2, 3]`。还考虑了多维和引号。

    参考：[#8138](https://www.sqlalchemy.org/trac/ticket/8138)

+   **[postgresql] [usecase]**

    添加了对 PostgreSQL 多范围类型的支持，该类型引入于 PostgreSQL 14 中。现在已将对 PostgreSQL 范围和多范围的支持概括为 psycopg3、psycopg2 和 asyncpg 后端，并提供了进一步方言支持的空间，使用与以前使用的 psycopg2 对象兼容的后端无关 `Range` 数据对象。请参阅新文档以了解使用模式。

    此外，增强了范围类型处理，以便自动渲染类型转换，因此对于不提供任何上下文的语句的就地往返，不需要为数据库明确指定所需的类型（在 [#8540](https://www.sqlalchemy.org/trac/ticket/8540) 中讨论）。

    非常感谢 @zeeeeeb 提交并测试新数据类型和 psycopg 支持的拉取请求。

    另见

    PostgreSQL 后端的新 RANGE / MULTIRANGE 支持和变更

    范围和多范围类型

    参考：[#7156](https://www.sqlalchemy.org/trac/ticket/7156), [#8540](https://www.sqlalchemy.org/trac/ticket/8540)

+   **[postgresql] [usecase]**

    配置`create_engine.pool_pre_ping`时发出的“ping”查询，对于 psycopg、asyncpg 和 pg8000，但不适用于 psycopg2，已更改为一个空查询(`;`)，而不是`SELECT 1`；此外，对于 asyncpg 驱动程序，已修复了此查询不必要使用准备语句的问题。其理由是消除 PostgreSQL 在发出 ping 时产生查询计划的需要。当前不支持由`psycopg2`驱动程序执行此操作，它继续使用`SELECT 1`。

    参考：[#8491](https://www.sqlalchemy.org/trac/ticket/8491)

+   **[postgresql] [change]**

    SQLAlchemy 现在要求 PostgreSQL 版本为 9 或更高。在某些有限的用例中，旧版本可能仍然可以工作。

+   **[postgresql] [change] [mssql]**

    `UUID`的参数`UUID.as_uuid`，以前专门针对 PostgreSQL 方言，现在已经泛化为 Core（连同一个新的与后端无关的`Uuid`数据类型），现在默认为`True`，表示此数据类型默认接受 Python `UUID`对象。此外，SQL Server 的`UNIQUEIDENTIFIER`数据类型已转换为接收 UUID 的类型；对于使用字符串值的遗留代码，设置`UNIQUEIDENTIFIER.as_uuid`参数为`False`。

    参考：[#7225](https://www.sqlalchemy.org/trac/ticket/7225)

+   **[postgresql] [change]**

    PostgreSQL 特定的`ENUM`数据类型的`ENUM.name`参数现在是一个必需的关键字参数。在任何情况下，“name”都是必要的，否则在 SQL/DDL 渲染时会引发错误。

+   **[postgresql] [change]**

    支持新的 PostgreSQL 功能，包括 psycopg3 方言以及扩展的“快速插入多个”支持，用于将绑定参数的类型信息传递给 PostgreSQL 数据库的系统已经重新设计，现在使用 SQL 编译器发出的内联转换，并且现在适用于所有 PostgreSQL 方言。这与以前的方法相反，以前的方法依赖于正在使用的 DBAPI 来自行呈现这些转换，例如 pg8000 和适应的 asyncpg 驱动程序的情况下，将使用 pep-249 `setinputsizes()`方法，或者对于 psycopg2 驱动程序，在大多数情况下将依赖于驱动程序本身，对于 ARRAY 则会做一些特殊的例外。

    现在，所有 PostgreSQL 方言都使用 PostgreSQL 双冒号样式在编译器内呈现这些转换所需的转换，并且对于 PostgreSQL 方言，已删除了使用`setinputsizes()`，因为这在任何情况���通常不是这些 DBAPI 的一部分（pg8000 是唯一的例外，它在 SQLAlchemy 开发人员的请求下添加了该方法）。

    这种方法的优势包括每个语句的性能，因为在执行时不需要对编译后的语句进行第二次遍历，对所有 DBAPI 的更好支持，因为现在有一个一致的应用类型信息系统，以及改进的透明度，因为 SQL 日志输出以及编译语句的字符串输出将直接显示这些转换存在于语句中，而以前这些转换在日志输出中是不可见的，因为它们会在语句记录后发生。

+   **[postgresql] [错误]**

    `Operators.match()`运算符现在在 PostgreSQL 全文搜索中使用`plainto_tsquery()`，而不是`to_tsquery()`。这种更改的理由是为了提供更好的与其他数据库后端上的 match 的跨兼容性。通过与`Operators.bool_op()`（布尔运算符的改进版本`Operators.op()`）结合使用`func`，仍然可以通过使用所有 PostgreSQL 全文函数来获得完全支持。

    另请参阅

    PostgreSQL 上的 match()运算符使用 plainto_tsquery()而不是 to_tsquery()

    参考：[#7086](https://www.sqlalchemy.org/trac/ticket/7086)

+   **[postgresql] [已移除]**

    移除对多个已弃用驱动程序的支持：

    > +   用于 PostgreSQL 的 pypostgresql。这作为外部驱动程序可在[`github.com/PyGreSQL`](https://github.com/PyGreSQL)获得。
    > +   
    > +   用于 PostgreSQL 的 pygresql。

    请切换到受支持的驱动程序之一或同一驱动程序的外部版本。

    参考：[#7258](https://www.sqlalchemy.org/trac/ticket/7258)

+   **[postgresql] [方言]**

    添加了对 `psycopg` 方言的支持，支持同步和异步执行。此方言在 `create_engine()` 和 `create_async_engine()` 引擎创建函数下可用。

    另请参阅

    对 psycopg 3（又名“psycopg”）的方言支持

    psycopg

    参考：[#6842](https://www.sqlalchemy.org/trac/ticket/6842)

+   **[postgresql] [psycopg2]**

    更新了 psycopg2 方言，使用 DBAPI 接口执行两阶段事务。以前使用 SQL 命令处理这种类型的事务。

    参考：[#7238](https://www.sqlalchemy.org/trac/ticket/7238)

+   **[postgresql] [schema]**

    引入了类型 `JSONPATH`，可在转换表达式中使用。在使用诸如 `jsonb_path_exists` 或 `jsonb_path_match` 这样接受 `jsonpath` 作为输入的函数时，某些 PostgreSQL 方言需要这个类型。

    另请参阅

    JSON 类型 - PostgreSQL JSON 类型。

    参考：[#8216](https://www.sqlalchemy.org/trac/ticket/8216)

+   **[postgresql] [reflection]**

    PostgreSQL 方言现在支持基于表达式的索引的反射。在使用 `Inspector.get_indexes()` 时以及使用 `Table.autoload_with` 反射 `Table` 时都支持反射。感谢 immerrr 和 Aidan Kane 在这个问题上的帮助。

    参考：[#7442](https://www.sqlalchemy.org/trac/ticket/7442)

### mysql

+   **[mysql] [usecase] [mariadb]**

    `ROLLUP` 函数现在会在 MySql 和 MariaDB 上正确呈现 `WITH ROLLUP`，允许在这些后端使用 group by rollup。

    参考：[#8503](https://www.sqlalchemy.org/trac/ticket/8503)

+   **[mysql] [bug]**

    修复了 MySQL `Insert.on_duplicate_key_update()` 中的问题，当在 VALUES 表达式中使用表达式时，会呈现错误的列名。感谢 Cristian Sabaila 提供的拉取请求。

    此更改也 **被后移** 到：1.4.27

    参考：[#7281](https://www.sqlalchemy.org/trac/ticket/7281)

+   **[mysql] [removed]**

    移除了对 OurSQL 驱动程序的支持，该驱动程序不再维护。

    参考：[#7258](https://www.sqlalchemy.org/trac/ticket/7258)

### mariadb

+   **[mariadb] [usecase]**

    添加了一个新的执行选项`is_delete_using=True`，当使用 ORM 启用的 DELETE 语句与“fetch”同步策略一起使用时，该选项将被消耗；此选项表示预计 DELETE 语句将使用多个表，在 MariaDB 上是 DELETE..USING 语法。然后，该选项指示对于已知不支持“DELETE..USING..RETURNING”语法的数据库，不应使用在 MariaDB 中新实现的 SQLAlchemy 2.0 的 RETURNING（对于[#7011](https://www.sqlalchemy.org/trac/ticket/7011)）。尽管它们支持“DELETE..USING”，但它们不支持“DELETE..USING..RETURNING”语法，这是 MariaDB 的当前能力。

    这个选项的原因是，ORM 启用的 DELETE 当前不知道 DELETE 语句是否针对多个表，直到编译发生，无论如何，编译都会被缓存，但需要知道这一点，以便事先发出用于待删除行的 SELECT。与为了预先检查所有 DELETE 语句以获取这种相对不寻常的 SQL 模式而对所有 DELETE 语句应用全面性能惩罚相比，通过在编译步骤中引发一个新的异常消息来请求`is_delete_using=True`执行选项。此异常消息仅在以下情况下特定（且仅）引发：语句是启用了 ORM 的 DELETE，已请求“fetch”同步策略；后端是 MariaDB 或具有此特定限制的其他后端；已检测到初始编译中的语句，否则会发出“DELETE..USING..RETURNING”。通过应用执行选项，ORM 知道要首先运行一个 SELECT。ORM 启用的 UPDATE 也实现了类似的选项，但目前还没有需要它的后端。

    参考：[#8344](https://www.sqlalchemy.org/trac/ticket/8344)

+   **[mariadb] [用例]**

    为 MariaDB 方言添加了 INSERT..RETURNING 和 DELETE..RETURNING 支持。UPDATE..RETURNING 尚未得到 MariaDB 的支持。从 10.5.0 开始，MariaDB 支持 INSERT..RETURNING，从 10.0.5 开始，支持 DELETE..RETURNING。

    参考：[#7011](https://www.sqlalchemy.org/trac/ticket/7011)

### sqlite

+   **[sqlite] [用例]**

    为 SQLite 的反射方法添加了一个名为`sqlite_include_internal=True`的新参数；当省略时，以`sqlite_`为前缀的本地表（根据 SQLite 文档，这些表被称为“内部模式”表，例如生成以支持“AUTOINCREMENT”列的`sqlite_sequence`表），不会包含在返回本地对象列表的反射方法中。这样可以避免在使用 Alembic 自动生成时出现问题，以前会将这些由 SQLite 生成的表视为从模型中移除。

    另请参阅

    反射内部模式表

    参考：[#8234](https://www.sqlalchemy.org/trac/ticket/8234)

+   **[sqlite] [用例]**

    为 SQLite 方言添加了 RETURNING 支持。自 SQLite 版本 3.35 起，SQLite 支持 RETURNING。

    参考：[#6195](https://www.sqlalchemy.org/trac/ticket/6195)

+   **[sqlite] [用例]**

    SQLite 方言现在支持 UPDATE..FROM 语法，用于 UPDATE 语句可能在语句的 WHERE 条件中引用其他表而无需使用子查询。当使用`Update`构造时，当使用多个表或其他实体或可选择时，此语法会自动调用。

    参考：[#7185](https://www.sqlalchemy.org/trac/ticket/7185)

+   **[sqlite] [性能] [bug]**

    当使用基于文件的数据库时，SQLite 方言现在默认使用`QueuePool`。这是与将`check_same_thread`参数设置为`False`一起设置的。已经观察到，默认使用`NullPool`的先前方法，在释放数据库连接后不会保留连接，实际上会对性能产生可衡量的负面影响。如常，通过`create_engine.poolclass`参数可以自定义池类。

    参见

    SQLite 方言为基于文件的数据库使用 QueuePool

    参考：[#7490](https://www.sqlalchemy.org/trac/ticket/7490)

+   **[sqlite] [性能] [用例]**

    SQLite 的 datetime、date 和 time 数据类型现在使用 Python 标准库的`fromisoformat()`方法来解析传入的 datetime、date 和 time 字符串值。这比以前基于正则表达式的方法提高了性能，还自动适应包含六位“微秒”格式或三位“毫秒”格式的 datetime 和 time 格式。

    参考：[#7029](https://www.sqlalchemy.org/trac/ticket/7029)

+   **[sqlite] [bug]**

    移除了关于 `Numeric` 类型发出的关于 DBAPI 不原生支持 Decimal 值的警告。这个警告是针对 SQLite 的，因为 SQLite 没有任何真正的方法（除非使用额外的扩展或解决方法）来处理超过 15 个有效数字的精度数值，因为它只使用浮点数来表示数字。由于这是 SQLite 本身已知且有文档记录的限制，而不是 pysqlite 驱动程序的怪癖，因此 SQLAlchemy 不需要为此发出警告。这个更改不会修改精度数值的处理方式。值可以继续按照 `Numeric`、`Float` 和相关数据类型配置为 `Decimal()` 或 `float()`，只是在使用 SQLite 时无法保持超过 15 个有效数字的精度，除非使用字符串等替代表示方法。

    参考：[#7299](https://www.sqlalchemy.org/trac/ticket/7299)

### mssql

+   **[mssql] [用例]**

    实现了 SQL Server 方言的“clustered index” 标志 `mssql_clustered` 的反射。感谢 John Lennox 提供的拉取请求。

    参考：[#8288](https://www.sqlalchemy.org/trac/ticket/8288)

+   **[mssql] [用例]**

    在创建表时，为 MSSQL 添加了对表和列注释的支持。添加了反射表注释的支持。感谢 Daniel Hall 在此拉取请求中的帮助。

    参考：[#7844](https://www.sqlalchemy.org/trac/ticket/7844)

+   **[mssql] [错误]**

    `mssql+pyodbc` 方言的 `use_setinputsizes` 参数现在默认为 `True`；这样非 Unicode 字符串比较将由 pyodbc 绑定到 pyodbc.SQL_VARCHAR 而不是 pyodbc.SQL_WVARCHAR，从而使得对 VARCHAR 列的索引生效。为了让 `fast_executemany=True` 参数继续正常工作，`use_setinputsizes` 模式现在在 `fast_executemany` 为 True 且使用的具体方法是 `cursor.executemany()` 时会跳过 `cursor.setinputsizes()` 调用，因为该方法不支持 setinputsizes。此更改还为被标记为 `Unicode` 或 `UnicodeText` 的值添加了适当的 pyodbc DBAPI 类型，并将基础的 `JSON` 数据类型修改为将 JSON 字符串值视为 `Unicode` 而不是 `String`。

    参考：[#8177](https://www.sqlalchemy.org/trac/ticket/8177)

+   **[mssql] [已移除]**

    由于缺乏测试支持，已移除对 mxodbc 驱动程序的支持。ODBC 用户可以使用完全受支持的 pyodbc 方言。

    参考：[#7258](https://www.sqlalchemy.org/trac/ticket/7258)

### oracle

+   **[oracle] [feature]**

    添加对新的 Oracle 驱动程序 `oracledb` 的支持。

    另请参阅

    对 oracledb 的方言支持

    python-oracledb

    参考：[#8054](https://www.sqlalchemy.org/trac/ticket/8054)

+   **[oracle] [feature]**

    实现了 `FLOAT` 数据类型的 DDL 和反射支持，其中包括显式的“binary_precision”值。使用特定于 Oracle 的 `FLOAT` 数据类型，可以指定新参数 `FLOAT.binary_precision`，这将直接呈现 Oracle 的浮点类型精度。此值在反射期间解释。在反射回 `FLOAT` 数据类型时，返回的数据类型是 `DOUBLE_PRECISION`（对于精度为 126 的 `FLOAT`，这也是 Oracle 的默认精度）、`REAL`（对于精度为 63）、以及 `FLOAT`（对于自定义精度，按照 Oracle 文档）。 

    作为这一变更的一部分，当为 Oracle 生成 DDL 时，明确拒绝了通用的 `Float.precision` 值，因为此精度无法准确转换为“二进制精度”；相反，错误消息鼓励使用 `TypeEngine.with_variant()`，以便精确选择 Oracle 的特定精度形式。这是一种与以往行为不兼容的更改，因为以前的“精度”值对于 Oracle 被静默地忽略。

    另请参阅

    新的 Oracle FLOAT 类型，具有二进制精度；不直接接受十进制精度

    参考：[#5465](https://www.sqlalchemy.org/trac/ticket/5465)

+   **[oracle] [feature]**

    对于 cx_Oracle 方言，完全实现了“RETURNING”支持，涵盖了两种个别功能：

    +   实现了多行 RETURNING，意味着对于产生多于一个 RETURNING 行的 DML 语句，现在将收到多个 RETURNING 行。

    +   "executemany RETURNING" 也已实现 - 这允许当使用 `cursor.executemany()` 时，RETURNING 每个语句产生一行。这一特性的实现为 ORM 插入提供了显著的性能改进，就像 SQLAlchemy 1.4 变更 ORM Batch inserts with psycopg2 now batch statements with RETURNING in most cases 中为 psycopg2 添加的一样。

    参考：[#6245](https://www.sqlalchemy.org/trac/ticket/6245)

+   **[oracle] [用例]**

    Oracle 现在将在 Oracle 12c 及以上版本中默认使用 FETCH FIRST N ROWS / OFFSET 语法来支持 limit/offset。当直接使用 `Select.fetch()` 时，该语法已经可用，现在对 `Select.limit()` 和 `Select.offset()` 也适用。

    参考：[#8221](https://www.sqlalchemy.org/trac/ticket/8221)

+   **[oracle] [更改]**

    在 Oracle 上，物化视图现在被反映为视图。在之前的 SQLAlchemy 版本中，视图会在表名中返回，而不在视图名中返回。由于此更改的副作用，默认情况下它们不会被 `MetaData.reflect()` 反映，除非设置了 `views=True`。要获取物化视图列表，请使用新的检查方法 `Inspector.get_materialized_view_names()`。

+   **[oracle] [错误]**

    对 cx_Oracle 和 oracledb 方言中的 BLOB / CLOB / NCLOB 数据类型进行了调整，以根据 Oracle 开发人员的建议改善性能。

    参考：[#7494](https://www.sqlalchemy.org/trac/ticket/7494)

+   **[oracle] [错误]**

    关于 `create_engine.implicit_returning` 弃用的相关内容，现在 “implicit_returning” 特性在所有情况下都为 Oracle 方言启用；以前，当检测到 Oracle 8/8i 版本时，该特性会被关闭，然而在线文档显示这两个版本都支持与现代版本相同的 RETURNING 语法。

    参考：[#6962](https://www.sqlalchemy.org/trac/ticket/6962)

+   **[oracle]**

    cx_Oracle 7 现在是 cx_Oracle 的最低版本。

### 杂项

+   **[已移除] [sybase]**

    删除了在之前的 SQLAlchemy 版本中已弃用的 “sybase” 内部方言。第三方方言支持可用。

    另请参阅

    外部方言

    参考：[#7258](https://www.sqlalchemy.org/trac/ticket/7258)

+   **[已移除] [firebird]**

    移除了在以前的 SQLAlchemy 版本中已弃用的“firebird”内部方言。第三方方言支持可用。

    另请参阅

    外部方言

    参考：[#7258](https://www.sqlalchemy.org/trac/ticket/7258)

## 2.0.30

无发布日期

### orm

+   **[orm] [bug]**

    添加了新属性`ORMExecuteState.is_from_statement`，用于检测形式为`select().from_statement()`的语句，并增强了`FromStatement`以根据发送到`Select.from_statement()`方法本身的元素设置`ORMExecuteState.is_select`、`ORMExecuteState.is_insert`、`ORMExecuteState.is_update`和`ORMExecuteState.is_delete`。

    参考：[#11220](https://www.sqlalchemy.org/trac/ticket/11220)

### 引擎

+   **[engine] [bug]**

    修复了`Connection.execution_options.logging_token`选项中的问题，其中更改已经记录了消息的连接的`logging_token`值不会更新以反映新的记录令牌。特别是这阻止了使用`Session.connection()`在连接上更改选项，因为 BEGIN 记录消息已经被发出。

    参考：[#11210](https://www.sqlalchemy.org/trac/ticket/11210)

### typing

+   **[typing] [bug] [regression]**

    修复了在版本 2.0.29 中由 PR [#11055](https://www.sqlalchemy.org/trac/ticket/11055)引起的输入退化，该版本尝试将`ParamSpec`添加到 asyncio `run_sync()`方法中，其中使用`AsyncConnection.run_sync()`与`MetaData.reflect()`将由于 bug 在 mypy 上失败。有关详细信息，请参见[`github.com/python/mypy/issues/17093`](https://github.com/python/mypy/issues/17093)。Pull request 由 Francisco R. Del Roio 提供。

    参考：[#11200](https://www.sqlalchemy.org/trac/ticket/11200)

### 杂项

+   **[bug] [test]**

    在测试中使用`subprocess.run`时，请确保`PYTHONPATH`变量正确初始化。

    参考：[#11268](https://www.sqlalchemy.org/trac/ticket/11268)

### orm

+   **[orm] [bug]**

    添加了新属性`ORMExecuteState.is_from_statement`，用于检测形式为`select().from_statement()`的语句，并增强了`FromStatement`以根据发送到`Select.from_statement()`方法本身的元素设置`ORMExecuteState.is_select`、`ORMExecuteState.is_insert`、`ORMExecuteState.is_update`和`ORMExecuteState.is_delete`。

    参考：[#11220](https://www.sqlalchemy.org/trac/ticket/11220)

### engine

+   **[engine] [bug]**

    修复了`Connection.execution_options.logging_token`选项中的问题，其中在已经记录了消息的连接上更改`logging_token`的值不会更新以反映新的日志令牌。特别是这阻止了使用`Session.connection()`在连接上更改选项，因为 BEGIN 日志消息已经被发出。

    参考：[#11210](https://www.sqlalchemy.org/trac/ticket/11210)

### typing

+   **[typing] [bug] [regression]**

    修复了由版本 2.0.29 中 PR [#11055](https://www.sqlalchemy.org/trac/ticket/11055)引起的类型回归，该 PR 试图将`ParamSpec`添加到 asyncio `run_sync()`方法中，其中在 mypy 上使用`AsyncConnection.run_sync()`与`MetaData.reflect()`会由于错误而在 mypy 上失败。有关详细信息，请参见[`github.com/python/mypy/issues/17093`](https://github.com/python/mypy/issues/17093)。感谢 Francisco R. Del Roio 提供的拉取请求。

    参考：[#11200](https://www.sqlalchemy.org/trac/ticket/11200)

### misc

+   **[bug] [test]**

    在测试中使用`subprocess.run`时，请确保`PYTHONPATH`变量正确初始化。

    参考：[#11268](https://www.sqlalchemy.org/trac/ticket/11268)

## 2.0.29

发布日期：2024 年 3 月 23 日

### orm

+   **[orm] [usecase]**

    添加了对[**PEP 695**](https://peps.python.org/pep-0695/) `TypeAliasType` 构造的支持，以及与 python 3.12 本地 `type` 关键字配合使用 ORM 注释声明形式时，当使用这些构造将链接到 [**PEP 593**](https://peps.python.org/pep-0593/) `Annotated` 容器时，允许解析`Annotated`的过程。

    参考文献：[#11130](https://www.sqlalchemy.org/trac/ticket/11130)

+   **[orm] [bug]**

    修复了声明性问题，其中使用`Relationship`而不是 `Mapped` 来对关系进行类型化，会无意中为该属性引入“动态”关系加载器策略。

    参考文献：[#10611](https://www.sqlalchemy.org/trac/ticket/10611)

+   **[orm] [bug]**

    修复了在 ORM 注释声明中使用`mapped_column()`时出现的问题，其中使用`mapped_column.index`或`mapped_column.unique`设置为`False`的情况将被一个具有该参数设置为`True`的传入`Annotated`元素覆盖，即使直接`mapped_column()`元素更具体且应该优先。增强了协调布尔值的逻辑，以适应本地值为`False`仍优先于来自注释元素的传入`True`值的情况。

    参考文献：[#11091](https://www.sqlalchemy.org/trac/ticket/11091)

+   **[orm] [bug] [regression]**

    修复了从版本 2.0.28 引入的回归，该回归是由于修复了[#11085](https://www.sqlalchemy.org/trac/ticket/11085)中新方法调整后缓存的参数值，该方法会干扰到`subqueryload()`加载器选项的实现，当使用此加载器选项与此加载器选项一起使用时，会使用一些更多的旧模式。

    参考文献：[#11173](https://www.sqlalchemy.org/trac/ticket/11173)

### engine

+   **[engine] [bug]**

    修复了“INSERT 语句的“Insert Many Values”行为功能中的问题，其中使用具有“内联执行”默认生成器的主键列，例如显式`Sequence`和显式架构名称，同时使用`Connection.execution_options.schema_translate_map`功能将无法正确呈现序列或参数，导致错误。

    参考：[#11157](https://www.sqlalchemy.org/trac/ticket/11157)

+   **[engine] [错误]**

    在版本 2.0.10 中对[#9618](https://www.sqlalchemy.org/trac/ticket/9618)所做的调整进行了更改，该版本添加了批量 INSERT 的 RE​​TURNING 行协调到传递给它的参数的行为。此行为包括已转换为 DB 的绑定参数值与返回的行值的比较，并不总是对于 SQL 列类型（例如 UUID）“对称”，具体取决于不同 DBAPI 接收此类值的方式与它们返回的方式，因此需要在这些列类型上增加额外的“哨兵值解析器”方法。不幸的是，这破坏了第三方列类型，如 SQLModel 中未实现此特殊方法的 UUID/GUID 类型，引发了错误“无法将结果集中的哨兵值与参数集匹配”。与其尝试进一步解释和文档化此“insertmanyvalues”特性的实现细节，包括新方法的公共版本，不如修改方法以不再需要此额外的转换步骤，并且进行比较的逻辑现在作用于预转换的绑定参数值与后处理值相比，后者应始终是匹配的数据类型。在罕见情况下，如果自定义 SQL 列类型也恰好用于批量 INSERT 的“哨兵”列，并且未接收和返回相同的值类型，则将引发“无法匹配”错误，但是缓解方法很简单，即传递与返回的相同 Python 数据类型。

    参考：[#11160](https://www.sqlalchemy.org/trac/ticket/11160)

### sql

+   **[sql] [错误] [回归]**

    修复了 1.4 系列中的回归问题，在 `TypeEngine.with_variant()` 方法的重构中引入的问题，该方法在 “with_variant()” 克隆原始 TypeEngine 而不是更改类型 中未能考虑到 `.copy()` 方法，这将丢失设置的变体映射。对于“schema”类型的非常特定情况，这会成为一个问题，其中包括 `Enum` 和 `ARRAY` 等类型，当它们在 ORM Declarative 映射中与 mixin 一起使用时，类型的复制就会发挥作用。现在也复制了变体映射。

    参考：[#11176](https://www.sqlalchemy.org/trac/ticket/11176)

### typing

+   **[typing] [bug]**

    修复了允许 asyncio 的 `run_sync()` 方法正确对参数进行类型标记的问题，根据传递的可调用对象使用 [**PEP 612**](https://peps.python.org/pep-0612/) `ParamSpec` 变量。感谢 Francisco R. Del Roio 提供的拉取请求。

    参考：[#11055](https://www.sqlalchemy.org/trac/ticket/11055)

### postgresql

+   **[postgresql] [usecase]**

    PostgreSQL 方言现在在反射具有域作为类型的列时返回 `DOMAIN` 实例。之前，返回的是域数据类型。作为此更改的一部分，改进了域反射以同时返回文本类型的排序规则。感谢 Thomas Stephenson 提供的拉取请求。

    参考：[#10693](https://www.sqlalchemy.org/trac/ticket/10693)

### 测试

+   **[tests] [bug]**

    已将改进后的测试套件应用于 SQLAlchemy 2.0，改进了与 asyncio 相关的测试运行方式，现在使用更新的 Python 3.11 `asyncio.Runner` 或其等价物，而不是依赖于先前基于 `asyncio.get_running_loop()` 的实现。这应该能够在 CPU 负载硬件上运行大量套件时防止事件循环出现故障，导致级联失败。

    参考：[#11187](https://www.sqlalchemy.org/trac/ticket/11187)

### orm

+   **[orm] [usecase]**

    增加了对 [**PEP 695**](https://peps.python.org/pep-0695/) `TypeAliasType` 构造的支持，以及与 Python 3.12 本地 `type` 关键字一起使用 ORM Annotated Declarative 形式时的支持，当使用这些构造链接到 [**PEP 593**](https://peps.python.org/pep-0593/) `Annotated` 容器时，允许解析 `Annotated` 时继续进行。

    参考：[#11130](https://www.sqlalchemy.org/trac/ticket/11130)

+   **[orm] [bug]**

    修复了声明式中的问题，其中使用 `Relationship` 而不是 `Mapped` 来定义关系会无意中引入该属性的“动态”关系加载器策略。

    引用：[#10611](https://www.sqlalchemy.org/trac/ticket/10611)

+   **[orm] [bug]**

    修复了使用 ORM 注释的声明式时，使用带有 `mapped_column.index` 或 `mapped_column.unique` 设置为 False 的 `mapped_column()` 会被传入的 `Annotated` 元素覆盖，即使直接的 `mapped_column()` 元素更具体且应该优先。调解布尔值的逻辑已经得到增强，以适应本地值为 `False` 的情况，仍然优先于注释元素传入的 `True` 值。

    引用：[#11091](https://www.sqlalchemy.org/trac/ticket/11091)

+   **[orm] [bug] [regression]**

    修复了从版本 2.0.28 开始由于对[#11085](https://www.sqlalchemy.org/trac/ticket/11085)的修复而引起的回归，新的方法调整后缓存的参数值会干扰`subqueryload()`加载器选项的实现，当使用额外的加载器条件特性与此加载器选项一起使用时，内部使用了一些更传统的模式。

    引用：[#11173](https://www.sqlalchemy.org/trac/ticket/11173)

### engine

+   **[engine] [bug]**

    修复了在 “Insert Many Values” Behavior for INSERT statements 功能中的问题，其中使用主键列与“内联执行”默认生成器（如显式的 `Sequence` 并带有显式模式名称），同时使用 `Connection.execution_options.schema_translate_map` 功能将无法正确渲染序列或参数，导致错误。

    引用：[#11157](https://www.sqlalchemy.org/trac/ticket/11157)

+   **[engine] [bug]**

    对版本 2.0.10 中对 [#9618](https://www.sqlalchemy.org/trac/ticket/9618) 进行的调整进行了更改，该版本增加了从批量插入中协调 RETURNING 行到传递给它的参数的行为。此行为包括将已经转换为数据库绑定参数值与返回的行值进行比较，对于 SQL 列类型如 UUID，不同的 DBAPI 接收这些值的方式与它们返回的方式具体取决于细节，因此需要对这些列类型进行额外的“哨兵值解析器”方法。不幸的是，这破坏了第三方列类型，如 SQLModel 中没有实现此特殊方法的 UUID/GUID 类型，引发错误“无法将结果集中的哨兵值与参数集匹配”。与其尝试进一步解释和文档化“insertmanyvalues”功能的这一实现细节，包括新方法的公共版本，不如将方法改进为不再需要这个额外的转换步骤，现在进行比较的逻辑是对预先转换的绑定参数值与后处理的值进行比较，后者应始终是匹配的数据类型。在不寻常的情况下，如果一个自定义的 SQL 列类型也碰巧用作批量插入的“哨兵”列，并且不接收和返回相同的值类型，将引发“无法匹配”错误，但是减轻措施很简单，应传递与返回相同的 Python 数据类型。

    参考：[#11160](https://www.sqlalchemy.org/trac/ticket/11160)

### sql

+   **[sql] [错误] [回归]**

    修复了 1.4 系列中的回归，该系列中对 `TypeEngine.with_variant()` 方法的重构，引入了“with_variant()”克隆原始 TypeEngine 而不是更改类型，未能适应 `.copy()` 方法，这将丢失设置的变体映射。对于非常特定的“模式”类型，这成为问题，该类型包括在 ORM Declarative 映射中与混合使用时的类型，其中类型的复制变得重要。现在也复制了变体映射。

    参考：[#11176](https://www.sqlalchemy.org/trac/ticket/11176)

### 输入

+   **[输入] [错误]**

    修复了允许 asyncio `run_sync()` 方法正确对参数进行类型标记的输入问题，根据传递的可调用对象，使用了[**PEP 612**](https://peps.python.org/pep-0612/) `ParamSpec` 变量。感谢 Francisco R. Del Roio 提交的拉取请求。

    参考：[#11055](https://www.sqlalchemy.org/trac/ticket/11055)

### postgresql

+   **[postgresql] [用例]**

    当反射一个具有域类型的列时，PostgreSQL 方言现在返回 `DOMAIN` 实例。以前，会返回域数据类型。作为这一改变的一部分，域反射还改进了以返回文本类型的排序规则。感谢 Thomas Stephenson 提交的拉取请求。

    参考：[#10693](https://www.sqlalchemy.org/trac/ticket/10693)

### 测试

+   **[tests] [bug]**

    对于与 asyncio 相关的测试，对 SQLAlchemy 2.0 进行了改进，现在使用了更新的 Python 3.11 `asyncio.Runner` 或等价的后移版，而不是依赖于基于 `asyncio.get_running_loop()` 的先前实现。这样做希望能够防止在 CPU 负载硬件上运行大量测试时出现问题，其中事件循环似乎会变得损坏，导致级联故障。

    参考：[#11187](https://www.sqlalchemy.org/trac/ticket/11187)

## 2.0.28

发布日期：2024 年 3 月 4 日

### orm

+   **[orm] [performance] [bug] [regression]**

    调整了在 2.0.23 版中发布的 [#10570](https://www.sqlalchemy.org/trac/ticket/10570) 中进行的修复，其中添加了新逻辑来协调可能在 `with_expression()` 构造中使用的缓存键生成过程中可能变化的绑定参数值。新逻辑改变了将新绑定参数值与语句关联的方法，避免了需要深度复制语句的情况，这可能会对非常深/复杂的 SQL 构造造成显著的性能损耗。新方法不再需要这个深复制步骤。

    参考：[#11085](https://www.sqlalchemy.org/trac/ticket/11085)

+   **[orm] [bug] [regression]**

    修复了由 [#9779](https://www.sqlalchemy.org/trac/ticket/9779) 引起的回归，其中在关系的 `and_()` 表达式中使用“secondary”表会失败，无法将其别名为与 `Select.join()` 表达式中“secondary”表的正常渲染相匹配，导致查询无效。

    参考：[#11010](https://www.sqlalchemy.org/trac/ticket/11010)

### 引擎

+   **[engine] [usecase]**

    添加了新的核心执行选项`Connection.execution_options.preserve_rowcount`。设置后，DBAPI 游标的`cursor.rowcount`属性将在语句执行时无条件地被记忆化，因此无论 DBAPI 为任何类型的语句提供的值是什么，都可以使用`CursorResult.rowcount`属性从`CursorResult`中获取。这允许访问像 INSERT 和 SELECT 这样的语句的 rowcount，程度取决于所使用的 DBAPI 的支持。`INSERT`语句的“插入多个值”行为也支持此选项，并在设置时将确保为批量插入行时正确设置`CursorResult.rowcount`。

    参考：[#10974](https://www.sqlalchemy.org/trac/ticket/10974)

### asyncio

+   **[asyncio] [错误]**

    如果将`QueuePool`或其他非异步池类传递给`create_async_engine()`，则会引发错误。此引擎仅接受包括`AsyncAdaptedQueuePool`在内的符合 asyncio 的池类。其他池类，如`NullPool`，与同步和异步引擎兼容，因为它们不执行任何锁定。

    另请参阅

    API 文档 - 可用的池实现

    参考：[#8771](https://www.sqlalchemy.org/trac/ticket/8771)

### 测试

+   **[测试] [更改]**

    tox.ini 文件中的 pytest 支持已更新以支持 pytest 8.1。

### orm

+   **[orm] [性能] [错误] [回归]**

    调整了在[#10570](https://www.sqlalchemy.org/trac/ticket/10570)中进行的修复，发布于 2.0.23，其中添加了新逻辑，用于协调可能在`with_expression()`构造中使用的缓存键生成过程中可能更改的绑定参数值。新逻辑改变了将新绑定参数值与语句关联的方法，避免了需要深度复制语句的需求，这可能会对非常深/复杂的 SQL 结构造成显著的性能损耗。新方法不再需要这个深度复制步骤。

    参考：[#11085](https://www.sqlalchemy.org/trac/ticket/11085)

+   **[orm] [bug] [regression]**

    修复了由 [#9779](https://www.sqlalchemy.org/trac/ticket/9779) 引起的回归问题，其中在关系 `and_()` 表达式中使用 “secondary” 表会无法被别名化，以匹配 “secondary” 表在 `Select.join()` 表达式中通常的呈现方式，导致查询无效。

    参考：[#11010](https://www.sqlalchemy.org/trac/ticket/11010)

### engine

+   **[engine] [usecase]**

    添加了新的核心执行选项 `Connection.execution_options.preserve_rowcount`。当设置时，DBAPI 游标的 `cursor.rowcount` 属性将在语句执行时无条件地被存储，以便无论语句的任何种类，都可以使用 `CursorResult.rowcount` 属性从 `CursorResult` 获取 DBAPI 提供的任何值。这允许访问像 INSERT 和 SELECT 这样的语句的 rowcount，以 DBAPI 使用的程度支持。“INSERT 语句的插入多个值”行为 也支持此选项，并将确保在设置时对行进行批量插入时正确设置 `CursorResult.rowcount`。

    参考：[#10974](https://www.sqlalchemy.org/trac/ticket/10974)

### asyncio

+   **[asyncio] [bug]**

    如果将 `QueuePool` 或其他非 asyncio 连接池类传递给 `create_async_engine()`，则会引发错误。此引擎仅接受与 asyncio 兼容的连接池类，包括 `AsyncAdaptedQueuePool`。其他连接池类，如 `NullPool`，与同步和异步引擎均兼容，因为它们不执行任何锁定。

    另请参见

    API 文档 - 可用的连接池实现

    参考：[#8771](https://www.sqlalchemy.org/trac/ticket/8771)

### 测试

+   **[tests] [change]**

    tox.ini 文件中的 pytest 支持已更新以支持 pytest 8.1。

## 2.0.27

发布日期：2024 年 2 月 13 日

### postgresql

+   **[postgresql] [bug] [regression]**

    由于刚发布的修复导致的回归，修复了[#10863](https://www.sqlalchemy.org/trac/ticket/10863)中一个无效的异常类被添加到“except”块的问题，除非确实发生了这样的捕获，否则不会被执行。已添加了一个模拟式测试以确保这种捕获在单元测试中被执行。

    参考：[#11005](https://www.sqlalchemy.org/trac/ticket/11005)

### postgresql

+   **[postgresql] [bug] [regression]**

    由于刚发布的修复导致的回归，修复了[#10863](https://www.sqlalchemy.org/trac/ticket/10863)中一个无效的异常类被添加到“except”块的问题，除非确实发生了这样的捕获，否则不会被执行。已添加了一个模拟式测试以确保这种捕获在单元测试中被执行。

    参考：[#11005](https://www.sqlalchemy.org/trac/ticket/11005)

## 2.0.26

发布日期：2024 年 2 月 11 日

### orm

+   **[orm] [bug]**

    使用缓存徽章替换了“加载器深度过深”的警告，对于 ORM 由于加载器选项链过于深而禁用缓存的那些语句，向 SQL 日志中添加了一个较短的消息。此警告突出显示的条件难以解决，并且通常只是 ORM 在应用 SQL 缓存时的限制。未来的功能可能包括调整禁用缓存的阈值的能力，但目前这个警告将不再是一个麻烦。

    参考：[#10896](https://www.sqlalchemy.org/trac/ticket/10896)

+   **[orm] [bug]**

    修复了一个问题，即如果该类型在类体内部局部声明，则无法在`Mapped`容器类型中使用类型（例如枚举）。现在，用于评估的本地变量范围包括类体本身的范围。此外，如果以字符串形式或使用将来的注释模式，`Mapped`中的表达式也可以引用类名本身。

    参考：[#10899](https://www.sqlalchemy.org/trac/ticket/10899)

+   **[orm] [bug]**

    修复了一个问题，即在使用 `Session.delete()` 与 `Mapper.version_id_col` 功能时，如果由于对象上的 `relationship.post_update` 的使用导致针对目标对象的附加 UPDATE，则使用正确的版本标识符将失败。这个问题类似于版本 2.0.25 中刚刚修复的 [#10800](https://www.sqlalchemy.org/trac/ticket/10800) 的情况，仅对更新情况进行了修复。

    参考：[#10967](https://www.sqlalchemy.org/trac/ticket/10967)

+   **[orm] [bug]**

    修复了在实现`with_expression()`时，如果使用不可缓存的 SQL 表达式，则会引发断言错误的问题；这是自 1.4 以来的 2.0 回归。

    参考：[#10990](https://www.sqlalchemy.org/trac/ticket/10990)

### examples

+   **[examples] [bug]**

    修复了历史元示例中的回归，使用`MetaData.to_metadata()`复制历史表时也会复制索引（这是一件好事），但无论用于这些索引的命名方案如何，都会导致索引命名冲突。现在这些索引都添加了“_history”后缀，方式与表名相同。

    参考：[#10920](https://www.sqlalchemy.org/trac/ticket/10920)

+   **[examples] [bug]**

    通过在所有表中添加`Identity`构造，并允许在此后端上进行主键生成，修复了 examples/performance 中性能示例脚本在 Oracle 数据库中大部分情况下的运行问题。仍有一些“原始 DBAPI”情况与 Oracle 不兼容。

### sql

+   **[sql] [bug]**

    修复了`case()`中确定表达式类型的逻辑问题，可能导致如果“whens”中的最后一个元素没有类型，则结果为`NullType`，或者在其他情况下，类型可能解析为`None`。逻辑已更新为扫描所有给定表达式，以便使用第一个非空类型，并始终确保存在类型。拉取请求由 David Evans 提供。

    参考：[#10843](https://www.sqlalchemy.org/trac/ticket/10843)

### typing

+   **[typing] [bug]**

    修复了`PoolEvents.checkin()`事件的类型签名，指示给定的`DBAPIConnection`参数在连接被无效化的情况下可能为`None`。

### postgresql

+   **[postgresql] [usecase] [reflection]**

    添加了对带有“NO INHERIT”标记的 PostgreSQL CHECK 约束的反射支持，设置反射数据中的关键字 `no_inherit=True`。拉取请求由 Ellis Valentiner 提供。

    参考：[#10777](https://www.sqlalchemy.org/trac/ticket/10777)

+   **[postgresql] [usecase]**

    支持 PostgreSQL `CREATE TABLE` 的 `USING <method>` 选项，以指定用于存储新表内容的访问方法。拉取请求由 Edgar Ramírez-Mondragón 提供。

    另请参阅

    PostgreSQL 表选项

    参考：[#10904](https://www.sqlalchemy.org/trac/ticket/10904)

+   **[postgresql] [usecase]**

    正确地将 PostgreSQL 的 RANGE 和 MULTIRANGE 类型类型化为 `Range[T]` 和 `Sequence[Range[T]]`。引入了实用序列 `MultiRange`，以便更好地支持 MULTIRANGE 类型的互操作性。

    参考：[#9736](https://www.sqlalchemy.org/trac/ticket/9736)

+   **[postgresql] [usecase]**

    在从 `Range` 或 `MultiRange` 实例推断数据库类型时，区分 INT4 和 INT8 范围和多范围类型，如果值适合 INT4，则优先选择 INT4。

+   **[postgresql] [bug] [regression]**

    在发布 2.0.24 版本中由于 [#10717](https://www.sqlalchemy.org/trac/ticket/10717) 导致 asyncpg 方言中的回归问题，现在尝试在终止之前优雅地关闭 asyncpg 连接的更改不会对除超时错误之外的其他潜在连接相关异常回退到 `terminate()`，没有考虑到优雅的 `.close()` 尝试由于其他原因（如连接错误）失败的情况。

    参考：[#10863](https://www.sqlalchemy.org/trac/ticket/10863)

+   **[postgresql] [bug]**

    修复了在使用 PostgreSQL 方言时，使用 `Uuid` 数据类型且将 `Uuid.as_uuid` 参数设置为 False 时出现的问题，ORM 优化的 INSERT 语句（例如“insertmanyvalues”功能）将无法正确对齐批量 INSERT 语句的主键 UUID 值，导致错误。类似的问题也已经为 pymssql 驱动程序修复。

### mysql

+   **[mysql] [bug]**

    修复了在 MySQL 列中未正确反映 NULL/NOT NULL 的问题，该列还指定了 VIRTUAL 或 STORED 指令。感谢 Georg Wicke-Arndt 的拉取请求。

    参考：[#10850](https://www.sqlalchemy.org/trac/ticket/10850)

+   **[mysql] [bug]**

    修复了 asyncio 方言 asyncmy 和 aiomysql 中的问题，其中它们的 `.close()` 方法显然不是一个优雅的关闭。替换为非标准的 `.ensure_closed()` 方法，该方法是可等待的，并将 `.close()` 移动到所谓的“终止”情况。

    参考：[#10893](https://www.sqlalchemy.org/trac/ticket/10893)

### mssql

+   **[mssql] [bug]**

    修复了在使用 pymssql 方言时，当使用`Uuid`数据类型且`Uuid.as_uuid`参数设置为 False 时的问题。ORM 优化的 INSERT 语句（例如“insertmanyvalues”功能）将无法正确对齐用于批量 INSERT 语句的主键 UUID 值，导致错误。类似的问题也已针对 PostgreSQL 驱动程序进行了修复。

### oracle

+   **[oracle] [performance] [bug]**

    更改了 Oracle 方言的默认 arraysize，以使用驱动程序设置的值，即在撰写本文时，cx_oracle 和 oracledb 均为 100。以前默认值为 50。将值设置为 50 可能会导致与仅使用 cx_oracle/oracledb 在较慢的网络上获取许多行时相比，性能显着下降。

    参考：[#10877](https://www.sqlalchemy.org/trac/ticket/10877)

### orm

+   **[orm] [bug]**

    用更短的消息替换了“加载器深度过深”的警告，该消息添加到 SQL 日志中的缓存徽章中，对于那些由于 ORM 禁用缓存而导致的加载器选项链过深的语句。此警告突出显示的条件很难解决，通常只是 ORM 在应用 SQL 缓存时的一个限制。未来的功能可能包括调整禁用缓存的阈值的能力，但目前该警告将不再成为一个麻烦。

    参考：[#10896](https://www.sqlalchemy.org/trac/ticket/10896)

+   **[orm] [bug]**

    修复了在类体内部声明本地类型（例如枚举）时无法在`Mapped`容器类型中使用该类型的问题。现在，用于 eval 的本地变量范围包括类体本身。此外，`Mapped`中的表达式也可以引用类名本身，如果作为字符串或使用未来注释模式。

    参考：[#10899](https://www.sqlalchemy.org/trac/ticket/10899)

+   **[orm] [bug]**

    修复了使用`Session.delete()`与`Mapper.version_id_col`功能一起时，如果由于对象上的`relationship.post_update`的使用导致针对目标对象发出额外的 UPDATE，则会失败使用正确的版本标识符的问题。该问题类似于[#10800](https://www.sqlalchemy.org/trac/ticket/10800)，只是在仅有更新的情况下在版本 2.0.25 中修复了。

    参考：[#10967](https://www.sqlalchemy.org/trac/ticket/10967)

+   **[orm] [错误]**

    修复了在 `with_expression()` 实现中，如果使用的 SQL 表达式不可缓存，则会引发断言错误的问题；这是自 1.4 版以来的 2.0 版本的退化。

    参考：[#10990](https://www.sqlalchemy.org/trac/ticket/10990)

### 示例

+   **[示例] [错误]**

    修复了 history_meta 示例中的退化问题，其中使用 `MetaData.to_metadata()` 来复制历史表也会复制索引（这是好事），但无论使用的索引命名方案如何，都会导致索引命名冲突。现在为这些索引添加了“_history”后缀，方式与为表名添加后缀相同。

    参考：[#10920](https://www.sqlalchemy.org/trac/ticket/10920)

+   **[示例] [错误]**

    通过向所有表添加 `Identity` 结构并允许主键在此后端上生成，修复了 examples/performance 中性能示例脚本在 Oracle 数据库中的大部分兼容性问题。一些“原始 DBAPI” 情况仍不兼容 Oracle。

### sql

+   **[sql] [错误]**

    修正了 `case()` 中确定表达式类型的逻辑问题，如果“whens”中的最后一个元素没有类型或在其他情况下类型可能解析为 `None`，则可能导致 `NullType`。逻辑已更新以扫描所有给定表达式，以使用第一个非空类型，并始终确保存在类型。感谢 David Evans 提交的拉取请求。

    参考：[#10843](https://www.sqlalchemy.org/trac/ticket/10843)

### 类型

+   **[类型] [错误]**

    修正了 `PoolEvents.checkin()` 事件的类型签名，指示给定的 `DBAPIConnection` 参数在连接被失效的情况下可能为 `None`。

### postgresql

+   **[postgresql] [用例] [反射]**

    增加了对 PostgreSQL CHECK 约束的反射支持，标记为“NO INHERIT”，在反射数据中设置键 `no_inherit=True`。感谢 Ellis Valentiner 提交的拉取请求。

    参考：[#10777](https://www.sqlalchemy.org/trac/ticket/10777)

+   **[postgresql] [用例]**

    为了支持 PostgreSQL 的 `CREATE TABLE` 中的 `USING <method>` 选项，以指定用于存储新表内容的访问方法。感谢 Edgar Ramírez-Mondragón 提交的拉取请求。

    另请参阅

    PostgreSQL 表选项

    参考：[#10904](https://www.sqlalchemy.org/trac/ticket/10904)

+   **[postgresql] [usecase]**

    正确地将 PostgreSQL RANGE 和 MULTIRANGE 类型类型化为`Range[T]`和`Sequence[Range[T]]`。引入了实用程序序列`MultiRange`，以更好地支持 MULTIRANGE 类型的互操作性。

    参考：[#9736](https://www.sqlalchemy.org/trac/ticket/9736)

+   **[postgresql] [usecase]**

    当从`Range`或`MultiRange`实例推断数据库类型时，区分 INT4 和 INT8 范围和多范围类型，如果值适合 INT4，则优先使用 INT4。

+   **[postgresql] [bug] [regression]**

    在 2.0.24 版本中由于[#10717](https://www.sqlalchemy.org/trac/ticket/10717)导致的 asyncpg 方言中的回归问题修复，该变更现在在终止之前尝试优雅地关闭 asyncpg 连接，不会为除超时错误之外的其他潜在连接相关异常回退到`terminate()`，没有考虑到当优雅的`.close()`尝试由于其他原因（如连接错误）失败时的情况。

    参考：[#10863](https://www.sqlalchemy.org/trac/ticket/10863)

+   **[postgresql] [bug]**

    修复了在使用 PostgreSQL 方言时，当使用`Uuid`数据类型且`Uuid.as_uuid`参数设置为 False 时的问题。ORM 优化的 INSERT 语句（例如“insertmanyvalues”功能）不会正确对齐批量 INSERT 语句的主键 UUID 值，导致错误。pymssql 驱动程序也修复了类似的问题。

### mysql

+   **[mysql] [bug]**

    修复了一个问题，即当 MySQL 列还指定了 VIRTUAL 或 STORED 指令时，NULL/NOT NULL 未能正确反映。感谢 Georg Wicke-Arndt 的拉取请求。

    参考：[#10850](https://www.sqlalchemy.org/trac/ticket/10850)

+   **[mysql] [bug]**

    修复了 asyncio 方言 asyncmy 和 aiomysql 中的一个问题，即它们的`.close()`方法显然不是一个优雅的关闭。将其替换为非标准的`.ensure_closed()`方法，该方法可等待，并将`.close()`移到所谓的“终止”情况。

    参考：[#10893](https://www.sqlalchemy.org/trac/ticket/10893)

### mssql

+   **[mssql] [bug]**

    修复了使用`Uuid`数据类型以及设置`Uuid.as_uuid`参数为 False 时的问题，当使用 pymssql 方言时，ORM 优化的 INSERT 语句（例如“insertmanyvalues”功能）将不正确地对齐批量 INSERT 语句的主键 UUID 值，导致错误。类似的问题也已在 PostgreSQL 驱动程序中修复。

### oracle

+   **[oracle] [performance] [bug]**

    更改了 Oracle 方言的默认 arraysize，以使用驱动程序设置的值，在撰写本文时，cx_oracle 和 oracledb 的值均为 100。先前，默认值设置为 50。默认值为 50 可能导致与仅使用 cx_oracle/oracledb 在较慢的网络上获取数百行时相比出现显着的性能回退。

    参考：[#10877](https://www.sqlalchemy.org/trac/ticket/10877)

## 2.0.25

发布日期：2024 年 1 月 2 日

### orm

+   **[orm] [usecase]**

    添加了对 Python 3.12 pep-695 类型别名结构的初步支持，用于解析 ORM 注释声明映射的自定义类型映射。

    参考：[#10807](https://www.sqlalchemy.org/trac/ticket/10807)

+   **[orm] [bug]**

    修复了在同时使用`relationship.post_update`特性和使用映射器 version_id_col 时可能导致第二个 UPDATE 语句未能使用正确的版本标识符的问题，假设在该 flush 中已经发出了一个已经增加了版本计数器的 UPDATE。

    参考：[#10800](https://www.sqlalchemy.org/trac/ticket/10800)

+   **[orm] [bug]**

    修复了 ORM 注释声明会错误解释没有指定集合为 uselist=True 的关系左侧的问题，如果左侧类型被给定为类而不是字符串，并且没有使用 future-style 注释。

    参考：[#10815](https://www.sqlalchemy.org/trac/ticket/10815)

### sql

+   **[sql] [bug]**

    在布尔比较的否定情况下改进了`any_()` / `all_()` 的编译，现在将呈现`NOT (expr)`而不是将等式操作符反转为不等号，允许对这些非典型操作符进行更精细的否定控制。

    参考：[#10817](https://www.sqlalchemy.org/trac/ticket/10817)

### typing

+   **[typing] [bug]**

    修复了在版本 2.0.24 中向`sqlalchemy.sql.functions`模块添加了类型后引起的回归问题，作为[#6810](https://www.sqlalchemy.org/trac/ticket/6810)的一部分：

    +   进一步增强了 pep-484 类型提示，以便从 `sqlalchemy.sql.expression.func` 派生的元素更有效地与 ORM 映射属性一起使用（[#10801](https://www.sqlalchemy.org/trac/ticket/10801)）

    +   修复了传递给函数的参数类型，以便再次正确解释文本表达式，如字符串和整数（[#10818](https://www.sqlalchemy.org/trac/ticket/10818)）

    引用：[#10801](https://www.sqlalchemy.org/trac/ticket/10801)，[#10818](https://www.sqlalchemy.org/trac/ticket/10818)

### asyncio

+   **[asyncio] [错误]**

    修复了 asyncio 版本的连接池中的关键问题，调用 `AsyncEngine.dispose()` 会生成一个新的连接池，该连接池未完全重新建立对 asyncio 兼容互斥锁的使用，导致在使用像 `asyncio.gather()` 这样的并发特性时，在 asyncio 上下文中发生死锁时使用了普通的 `threading.Lock()`。

    此更改也 **回溯** 到：1.4.51

    引用：[#10813](https://www.sqlalchemy.org/trac/ticket/10813)

### oracle

+   **[oracle] [asyncio]**

    在 asyncio 模式下添加了对 python-oracledb 的支持，使用了新发布的支持 asyncio 的 `oracledb` DBAPI 版本。 对于 2.0 系列，这是一个预览版本，当前实现尚未包括对 `AsyncConnection.stream()` 的支持。 改进的支持计划在 SQLAlchemy 的 2.1 发布中实现。

    引用：[#10679](https://www.sqlalchemy.org/trac/ticket/10679)

### orm

+   **[orm] [使用情况]**

    在为 ORM 注释性声明映射解析自定义类型映射时，添加了对 Python 3.12 pep-695 类型别名结构的初步支持。

    引用：[#10807](https://www.sqlalchemy.org/trac/ticket/10807)

+   **[orm] [错误]**

    修复了同时使用 `relationship.post_update` 功能和使用 mapper version_id_col 时可能导致的问题，在这种情况下，后续更新功能发出的第二个 UPDATE 语句可能无法使用正确的版本标识符，假设在该刷新中已经发出了一个已经增加了版本计数器的 UPDATE。

    引用：[#10800](https://www.sqlalchemy.org/trac/ticket/10800)

+   **[orm] [错误]**

    修复了 ORM 注释性声明在没有指定任何集合的情况下误解释关系左侧的问题，如果左侧类型是作为类而不是字符串给出的，并且没有使用未来样式注释。

    引用：[#10815](https://www.sqlalchemy.org/trac/ticket/10815)

### sql

+   **[sql] [错误]**

    改进了在布尔比较的否定上下文中`any_()` / `all_()`的编译，现在将呈现`NOT (expr)`而不是将等式运算符反转为不等于，允许更精细地控制这些非典型运算符的否定。

    参考：[#10817](https://www.sqlalchemy.org/trac/ticket/10817)

### 输入

+   **[输入] [错误]**

    修复了在版本 2.0.24 中添加到`sqlalchemy.sql.functions`模块的类型提示引起的回归，作为[#6810](https://www.sqlalchemy.org/trac/ticket/6810)的一部分：

    +   进一步增强了 pep-484 类型提示，以允许从`sqlalchemy.sql.expression.func`派生的元素更有效地与 ORM 映射的属性一起使用（[#10801](https://www.sqlalchemy.org/trac/ticket/10801)）

    +   修复了传递给函数的参数类型，以便像字符串和整数这样的文字表达式再次被正确解释（[#10818](https://www.sqlalchemy.org/trac/ticket/10818)）

    参考：[#10801](https://www.sqlalchemy.org/trac/ticket/10801), [#10818](https://www.sqlalchemy.org/trac/ticket/10818)

### asyncio

+   **[asyncio] [错误]**

    修复了在连接池的 asyncio 版本中的关键问题，调用`AsyncEngine.dispose()`会产生一个未完全重新建立使用 asyncio 兼容互斥锁的新连接池，导致在使用并发功能时（如`asyncio.gather()`）在 asyncio 上下文中使用普通的`threading.Lock()`会导致死锁。

    此更改也**回溯**到：1.4.51

    参考：[#10813](https://www.sqlalchemy.org/trac/ticket/10813)

### oracle

+   **[oracle] [asyncio]**

    在 asyncio 模式下添加了对 python-oracledb 的支持，使用包含 asyncio 支持的新发布版本的`oracledb` DBAPI。对于 2.0 系列，这是一个预览版本，当前实现尚未包括对`AsyncConnection.stream()`的支持。改进的支持计划在 SQLAlchemy 的 2.1 版本中实现。

    参考：[#10679](https://www.sqlalchemy.org/trac/ticket/10679)

## 2.0.24

发布日期：2023 年 12 月 28 日

### orm

+   **[orm] [错误]**

    改进了首次在版本 0.9.8 中发布的用于[#3208](https://www.sqlalchemy.org/trac/ticket/3208)的修复，其中声明内部使用的类注册表可能会受到竞争条件的影响，即在个别映射类同时被垃圾回收时，同时正在构建新的映射类，这可能发生在某些测试套件配置或动态类创建环境中。除了已添加的弱引用检查外，还首先复制正在迭代的项目列表，以避免“在迭代时更改列表”错误。拉取请求由 Yilei Yang 提供。

    此更改也**回溯**到：1.4.51

    参考：[#10782](https://www.sqlalchemy.org/trac/ticket/10782)

+   **[orm] [bug]**

    修复了在未对非初始化的`mapped_column()`构造上使用`foreign()`注释会产生一个没有类型的表达式的问题，然后在实际列初始化时未更新，导致关系未适当确定`use_get`的问题。

    参考：[#10597](https://www.sqlalchemy.org/trac/ticket/10597)

+   **[orm] [bug]**

    改进了当工作单元过程将主键列的值设置为 NULL 时产生的错误消息，原因是具有对该列的依赖规则的相关对象被删除，包括不仅目标对象和列名，还包括源列，从中 NULL 值起源。拉取请求由 Jan Vollmer 提供。

    参考：[#10668](https://www.sqlalchemy.org/trac/ticket/10668)

+   **[orm] [bug]**

    修改了`MappedAsDataclass`、`DeclarativeBase`和`DeclarativeBaseNoMeta`使用的`__init_subclass__()`方法，以接受任意的`**kw`并将其传播到`super()`调用，允许更大的灵活性安排使用`__init_subclass__()`关键字参数的自定义超类和混入。拉取请求由 Michael Oliver 提供。

    参考：[#10732](https://www.sqlalchemy.org/trac/ticket/10732)

+   **[orm] [bug]**

    确保在 ORM 启用的 INSERT、UPDATE 和 DELETE 语句的`returning()`部分中使用的`Bundle`对象的用例经过测试并完全可用。这在以前从未明确实现或测试过，并且在 1.4 系列中无法正常工作；在 2.0 系列中，带有 WHERE 条件的 ORM UPDATE/DELETE 缺少实现方法，导致无法使用`Bundle`对象。

    参考：[#10776](https://www.sqlalchemy.org/trac/ticket/10776)

+   **[orm] [bug]**

    修复了 2.0 版本中`MutableList`的回归问题，其中检测序列的例程未能正确过滤字符串或字节实例，导致无法将字符串值分配给特定索引（而非序列值将正常工作）。

    参考：[#10784](https://www.sqlalchemy.org/trac/ticket/10784)

### engine

+   **[engine] [bug]**

    修复了在将`URL`对象转换为字符串时，使用`URL.render_as_string()`方法对用户名和密码组件进行 URL 编码的问题，通过使用 Python 标准库`urllib.parse.quote`，同时允许加号和空格保持不变，以支持 SQLAlchemy 的非标准 URL 解析，而不是多年前的传统自制例程。感谢 Xavier NUNN 的拉取请求。

    参考：[#10662](https://www.sqlalchemy.org/trac/ticket/10662)

### sql

+   **[sql] [bug]**

    修复了 SQL 元素的字符串化问题，在没有传递特定方言的情况下，遇到特定方言元素（如 PostgreSQL 的“on conflict do update”构造）时，未能提供适当状态以渲染构造，导致内部错误。

    参考：[#10753](https://www.sqlalchemy.org/trac/ticket/10753)

+   **[sql] [bug]**

    修复了针对 DML 构造（如`insert()`构造）的`CTE`的字符串化或编译失败的问题，由于错误地检���到语句整体为 INSERT，导致内部错误。

### schema

+   **[schema] [bug]**

    修复了在创建像`Table`这样的对象时，当参数本身作为元组传递时，错误报告对意外模式项的处理不正确，导致格式错误。错误消息已经更新为使用 f-strings。

    参考：[#10654](https://www.sqlalchemy.org/trac/ticket/10654)

### typing

+   **[typing] [bug]**

    为 `sqlalchemy.sql.functions` 模块完成了 pep-484 类型注解。针对 `func` 元素进行的 `select()` 构造现在应该具有填充的返回类型。

    参考：[#6810](https://www.sqlalchemy.org/trac/ticket/6810)

### asyncio

+   **[asyncio] [change]**

    `async_fallback` 方言参数现已弃用，并将在 SQLAlchemy 2.1 中移除。这个标志在 SQLAlchemy 的测试套件中已经有一段时间没有使用了。通过使用 `greenlet_spawn()` 在 greenlet 中运行代码，asyncio 方言仍然可以以同步方式运行。

### postgresql

+   **[postgresql] [bug]**

    调整了 asyncpg 方言，使得当使用 `terminate()` 方法丢弃一个无效的连接时，方言将首先尝试使用带有超时的 `.close()` 优雅地关闭连接，如果操作仅在异步事件循环上下文中进行。这允许 asyncpg 驱动程序处理最终化 `TimeoutError`，包括能够关闭长时间运行的查询服务器端，否则该查询可能会在程序退出后继续运行。

    参考：[#10717](https://www.sqlalchemy.org/trac/ticket/10717)

### mysql

+   **[mysql] [bug]**

    修复了使用旧于 1.0 版本的 PyMySQL 进行池预先检查时由修复票号 [#10492](https://www.sqlalchemy.org/trac/ticket/10492) 引入的回归。

    此更改也已**回溯**至：1.4.51

    参考：[#10650](https://www.sqlalchemy.org/trac/ticket/10650)

### tests

+   **[tests] [bug]**

    对测试套件进行了改进，进一步加固了在未安装 Python `greenlet` 时运行的能力。现在有一个 tox 目标，其中包含标记“nogreenlet”，将在未安装 greenlet 的情况下运行测试套件（请注意，它仍然会在 tox 配置中临时安装 greenlet）。

    参考：[#10747](https://www.sqlalchemy.org/trac/ticket/10747)

### orm

+   **[orm] [bug]**

    改进了首次在版本 0.9.8 中发布的针对 [#3208](https://www.sqlalchemy.org/trac/ticket/3208) 实施的修复，其中 declarative 内部使用的类注册表可能会在同时进行垃圾回收的个别映射类与新映射类构造时发生竞争条件，这可能会在某些测试套件配置或动态类创建环境中发生。除了已添加的 weakref 检查外，还首先复制正在迭代的项目列表，以避免“在迭代时更改列表”错误。感谢 Yilei Yang 提交的拉取请求。

    此更改也已**回溯**至：1.4.51

    参考：[#10782](https://www.sqlalchemy.org/trac/ticket/10782)

+   **[orm] [bug]**

    修复了在非初始化的 `mapped_column()` 构造上使用 `foreign()` 注释会产生没有类型的表达式的问题，然后在实际列的初始化时不会更新，导致关系无法适当地确定 `use_get` 等问题的问题。

    参考：[#10597](https://www.sqlalchemy.org/trac/ticket/10597)

+   **[orm] [bug]**

    改进了工作单元过程生成的错误消息，当由于相关对象对该列具有依赖规则并且被删除时，工作单元过程将主键列的值设置为 NULL 时，不仅包括目标对象和列名，还包括源列的列名，从而使 NULL 值起源于哪里。拉取请求由 Jan Vollmer 提供。

    参考：[#10668](https://www.sqlalchemy.org/trac/ticket/10668)

+   **[orm] [bug]**

    修改了 `MappedAsDataclass`、`DeclarativeBase` 和 `DeclarativeBaseNoMeta` 使用的 `__init_subclass__()` 方法，接受任意的 `**kw` 并将它们传播到 `super()` 调用，允许更灵活地安排使用 `__init_subclass__()` 关键字参数的自定义超类和混入。拉取请求由 Michael Oliver 提供。

    参考：[#10732](https://www.sqlalchemy.org/trac/ticket/10732)

+   **[orm] [bug]**

    确保在 ORM 启用的 INSERT、UPDATE 和 DELETE 语句的 `returning()` 部分中使用的 `Bundle` 对象的用例经过测试并且完全可用。这在之前从未明确实现或测试过，在 1.4 系列中没有正常工作；在 2.0 系列中，具有 WHERE 条件的 ORM UPDATE/DELETE 缺少实现方法，阻止了 `Bundle` 对象的正常工作。

    参考：[#10776](https://www.sqlalchemy.org/trac/ticket/10776)

+   **[orm] [bug]**

    修复了在 2.0 中的 `MutableList` 中的回归，其中检测序列的例程不会正确地过滤出字符串或字节实例，使得无法将字符串值分配给特定索引（而非序列值则正常工作）。

    参考：[#10784](https://www.sqlalchemy.org/trac/ticket/10784)

### 引擎

+   **[engine] [bug]**

    修复了在使用`URL.render_as_string()`方法将用户名和密码组件进行 URL 编码时的问题，通过使用 Python 标准库`urllib.parse.quote`，同时允许加号和空格保持不变，以支持 SQLAlchemy 的非标准 URL 解析，而不是多年前的传统自制程序。感谢 Xavier NUNN 的拉取请求。

    参考：[#10662](https://www.sqlalchemy.org/trac/ticket/10662)

### sql

+   **[sql] [bug]**

    修复了 SQL 元素的字符串化问题，其中未传递特定方言时，遇到特定方言元素（如 PostgreSQL 的“on conflict do update”构造），然后未提供适当状态以呈现构造的字符串化方言，导致内部错误。

    参考：[#10753](https://www.sqlalchemy.org/trac/ticket/10753)

+   **[sql] [bug]**

    修复了针对 DML 构造（如`insert()`）的`CTE`进行字符串化或编译时失败的问题，由于错误地检测到语句整体是一个 INSERT，导致内部错误。

### 模式

+   **[schema] [bug]**

    修复了在创建对象（如`Table`）时，对于意外模式项的错误报告处理不正确的问题，该参数本身被传递为元组，导致格式化错误。错误消息已经更新为使用 f-strings。

    参考：[#10654](https://www.sqlalchemy.org/trac/ticket/10654)

### 类型注释

+   **[typing] [bug]**

    完成了`sqlalchemy.sql.functions`模块的 pep-484 类型注释。针对`func`元素进行的`select()`构造现在应该具有填充的返回类型。

    参考：[#6810](https://www.sqlalchemy.org/trac/ticket/6810)

### asyncio

+   **[asyncio] [change]**

    `async_fallback`方言参数现已弃用，并将在 SQLAlchemy 2.1 中删除。这个标志已经有一段时间没有在 SQLAlchemy 的测试套件中使用了。通过使用`greenlet_spawn()`在 greenlet 中运行代码，asyncio 方言仍然可以以同步方式运行。

### postgresql

+   **[postgresql] [bug]**

    调整了 asyncpg 方言，使得当使用 `terminate()` 方法丢弃一个无效的连接时，方言将首先尝试使用带有超时的 `.close()` 优雅地关闭连接，仅在异步事件循环上下文中进行该操作时。这允许 asyncpg 驱动程序处理最终化 `TimeoutError`，包括能够在程序退出后继续运行的长时间运行的查询服务器端。

    参考：[#10717](https://www.sqlalchemy.org/trac/ticket/10717)

### mysql

+   **[mysql] [bug]**

    修复了使用旧于 1.0 版本的 PyMySQL 与池预先 PING 时由修复票证 [#10492](https://www.sqlalchemy.org/trac/ticket/10492) 引入的回归问题。

    此更改还**回溯**到：1.4.51

    参考：[#10650](https://www.sqlalchemy.org/trac/ticket/10650)

### 测试

+   **[tests] [bug]**

    对测试套件进行改进，以进一步加强在未安装 Python `greenlet` 时运行的能力。现在有一个 tox 目标包含标记“nogreenlet”，将以未安装 greenlet 的方式运行测试套件（注意，它仍然在 tox 配置中临时安装 greenlet）。

    参考：[#10747](https://www.sqlalchemy.org/trac/ticket/10747)

## 2.0.23

发布日期：2023 年 11 月 2 日

### orm

+   **[orm] [usecase]**

    对于新样式批量 ORM 插入，实现了 `Session.bulk_insert_mappings.render_nulls` 参数，允许 `render_nulls=True` 作为执行选项。这允许参数字典中含有混合的 `None` 值的批量 ORM 插入使用给定的字典键的单个行批次，而不是将每个 INSERT 中的 NULL 列分开成批次。

    另请参阅

    在 ORM 批量 INSERT 语句中发送 NULL 值

    参考：[#10575](https://www.sqlalchemy.org/trac/ticket/10575)

+   **[orm] [bug]**

    修复了 `__allow_unmapped__` 指令无法允许旧式 `Column` / `deferred()` 映射的问题，这些映射尽管没有 `Mapped[]` 作为它们的类型，但仍然具有诸如 `Any` 或特定类型的注释，并且不会出现有关定位属性名称的错误。

    参考：[#10516](https://www.sqlalchemy.org/trac/ticket/10516)

+   **[orm] [bug]**

    修复了使用 `with_expression()` 结构与加载器选项 `selectinload()`、`lazyload()` 结合使用时，在后续缓存运行中无法正确替换绑定参数值的缓存错误。

    参考：[#10570](https://www.sqlalchemy.org/trac/ticket/10570)

+   **[orm] [bug]**

    修复了 ORM 注释的声明式中的错误，其中使用 `ClassVar`，但仍然以某种方式引用了 ORM 映射类名，将无法解释为未映射的 `ClassVar`。

    参考：[#10472](https://www.sqlalchemy.org/trac/ticket/10472)

### sql

+   **[sql] [usecase]**

    为 PostgreSQL 和 Oracle 方言的 `Interval` 数据类型实现了“文字值处理”，允许文字渲染间隔值。Pull request 由 Indivar Mishra 提供。

    参考：[#9737](https://www.sqlalchemy.org/trac/ticket/9737)

+   **[sql] [bug]**

    修复了在某些与其他文字渲染参数的组合中，使用相同的绑定参数超过一次并且 `literal_execute=True` 会导致错误值渲染的问题，这是由于迭代问题造成的。

    此更改也已被**回溯**至：1.4.50

    参考：[#10142](https://www.sqlalchemy.org/trac/ticket/10142)

+   **[sql] [bug]**

    为所有包含文字处理的数据类型的“文字处理器”添加了编译器级的 None/NULL 处理，即在 SQL 语句中将值内联呈现而不是作为绑定参数，适用于所有那些不具有显式“null 值”处理的类型。之前，这种行为是未定义且不一致的。

    参考：[#10535](https://www.sqlalchemy.org/trac/ticket/10535)

+   **[sql]**

    移除了未使用的占位符方法 `TypeEngine.compare_against_backend()` 。该方法被 Alembic 的非常旧的版本使用。有关详细信息，请参见 [`github.com/sqlalchemy/alembic/issues/1293`](https://github.com/sqlalchemy/alembic/issues/1293) 。

### asyncio

+   **[asyncio] [bug]**

    修复了 `AsyncSession.close_all()` 方法的错误。还添加了函数 `close_all_sessions()` ，它是 `close_all_sessions()` 的等效项。Pull request 由 Bryan 不可思议 提供。

    参考：[#10421](https://www.sqlalchemy.org/trac/ticket/10421)

### postgresql

+   **[postgresql] [bug]**

    修复了 2.0 中由 [#7744](https://www.sqlalchemy.org/trac/ticket/7744) 引起的回归，其中涉及 PostgreSQL JSON 操作符与其他操作符（如字符串连接）组合的表达式链会由于特定于 PostgreSQL 方言的实现细节而失去正确的括号，。

    参考：[#10479](https://www.sqlalchemy.org/trac/ticket/10479)

+   **[postgresql] [bug]**

    当使用 asyncpg 后端和 `BIT` 数据类型时，修复了“insertmanyvalues”的 SQL 处理。 asyncpg 上的 `BIT` 显然需要使用 asyncpg 特定的 `BitString` 类型，该类型当前在使用此 DBAPI 时公开，使其与其他所有在此处使用普通位串的 PostgreSQL DBAPI 不兼容。 在版本 2.1 中的未来修复将会使这种数据类型在所有 PG 后端上正规化。 感谢 Sören Oldag 提交的拉取请求。

    参考：[#10532](https://www.sqlalchemy.org/trac/ticket/10532)

### mysql

+   **[mysql] [bug]**

    修复了 MySQL “预 Ping”例程中的新不兼容性，其中传递给`connection.ping()`的`False`参数，该参数旨在禁用不需要的“自动重新连接”功能，在 MySQL 驱动程序和后端中被弃用，并且对于某些版本的 MySQL 本机客户端驱动程序正在产生警告。 对于 mysqlclient，它已被删除，而对于 PyMySQL 和基于 PyMySQL 的驱动程序，该参数将在某个时候被弃用并删除，因此使用 API 内省来对抗这些不同阶段的移除。

    此更改也已**回溯**到：1.4.50

    参考：[#10492](https://www.sqlalchemy.org/trac/ticket/10492)

### mariadb

+   **[mariadb] [bug]**

    当使用 MariaDB 时，调整了 MySQL / MariaDB 方言，如果未使用显式的`True`或`False`值指定`Column.nullable`，则将生成的列默认设置为 NULL，因为 MariaDB 不支持具有生成列的“NOT NULL”短语。 感谢 Indivar 提交的拉取请求。

    参考：[#10056](https://www.sqlalchemy.org/trac/ticket/10056)

+   **[mariadb] [bug] [regression]**

    为似乎是 MySQL/MariaDB 驱动程序之间的固有问题建立了一个解决方法，该问题在使用 SQLAlchemy 的“空 IN”条件返回不包含行的 DELETE DML 的 RETURNING 结果时失败，该 DELETE DML 在 2.0 系列中用于“同步会话”功能的批量 DELETE 语句。 为了解决这个问题，当检测到“给定 RETURNING 时没有描述”的特定情况时，将生成一个带有正确游标描述的“空结果”，并将其用于替代不起作用的游标。

    参考：[#10505](https://www.sqlalchemy.org/trac/ticket/10505)

### mssql

+   **[mssql] [usecase]**

    已添加对基于 pyodbc 和通用 aio*方言架构构建的 SQL Server 的`aioodbc`驱动程序的支持。

    另请参阅

    aioodbc - 在 SQL Server 方言文档中。

    参考：[#6521](https://www.sqlalchemy.org/trac/ticket/6521)

+   **[mssql] [bug] [reflection]**

    修复了在具有大于 18 位数的大型标识起始值的 bigint 列的情况下，标识列反射将失败的问题。

    此更改也**反向移植**到：1.4.50

    参考：[#10504](https://www.sqlalchemy.org/trac/ticket/10504)

### oracle

+   **[oracle] [bug]**

    在`Interval`数据类型中修复了问题，其中 Oracle 实现未用于 DDL 生成，导致`day_precision`和`second_precision`参数被忽略，尽管该方言支持。感谢 Indivar 的 Pull 请求。

    参考：[#10509](https://www.sqlalchemy.org/trac/ticket/10509)

+   **[oracle] [bug]**

    修复了 cx_Oracle 方言声称支持比实际上在 SQLAlchemy 2.0 系列中实际支持的更低的 cx_Oracle 版本（7.x）的问题。方言导入了仅在 cx_Oracle 8 或更高版本中才有的符号，因此运行时方言检查以及 setup.cfg 要求已更新以反映此兼容性。

    参考：[#10470](https://www.sqlalchemy.org/trac/ticket/10470)

### orm

+   **[orm] [usecase]**

    实现了`Session.bulk_insert_mappings.render_nulls`参数，用于新样式的批量 ORM 插入，允许`render_nulls=True`作为执行选项。这允许在参数字典中使用`None`值的批量 ORM 插入使用给定的一组字典键的单个行批次，而不是将其拆分为省略每个 INSERT 中的 NULL 列的批次。

    另请参阅

    在 ORM 批量 INSERT 语句中发送 NULL 值

    参考：[#10575](https://www.sqlalchemy.org/trac/ticket/10575)

+   **[orm] [bug]**

    修复了`__allow_unmapped__`指令无法允许具有`Any`或没有`Mapped[]`作为其类型的特定类型的注释的遗留`Column` / `deferred()`映射的问题，而无需与定位属性名称相关的错误。

    参考：[#10516](https://www.sqlalchemy.org/trac/ticket/10516)

+   **[orm] [bug]**

    修复了使用 `with_expression()` 结构与加载器选项 `selectinload()`，`lazyload()` 结合使用时，与后续缓存运行中正确替换绑定参数值失败的缓存错误。

    引用：[#10570](https://www.sqlalchemy.org/trac/ticket/10570)

+   **[orm] [bug]**

    修复了 ORM 注释声明中的错误，其中使用了一个 `ClassVar`，尽管以某种方式引用了 ORM 映射的类名，但未能被解释为未映射的 `ClassVar`。

    引用：[#10472](https://www.sqlalchemy.org/trac/ticket/10472)

### sql

+   **[sql] [usecase]**

    实现了对于`Interval`数据类型的“字面值处理”，适用于 PostgreSQL 和 Oracle 方言，允许直接渲染间隔值。感谢 Indivar Mishra 提供的拉取请求。

    引用：[#9737](https://www.sqlalchemy.org/trac/ticket/9737)

+   **[sql] [bug]**

    修复了一个问题，即在某些与其他字面渲染参数组合使用`literal_execute=True`时，多次使用相同的绑定参数会由于迭代问题导致错误的值渲染。

    此更改还**回溯**到：1.4.50

    引用：[#10142](https://www.sqlalchemy.org/trac/ticket/10142)

+   **[sql] [bug]**

    为所有包含字面处理的数据类型的“字面处理器”添加了编译器级 None/NULL 处理，即在 SQL 语句中将值内联呈现而不是作为绑定参数，对于所有不具有显式“空值”处理的类型。以前，此行为是未定义的且不一致的。

    引用：[#10535](https://www.sqlalchemy.org/trac/ticket/10535)

+   **[sql]**

    移除了未使用的占位符方法 `TypeEngine.compare_against_backend()`，此方法仅用于非常旧版本的 Alembic。有关详细信息，请参见[`github.com/sqlalchemy/alembic/issues/1293`](https://github.com/sqlalchemy/alembic/issues/1293)。

### asyncio

+   **[asyncio] [bug]**

    修复了 `AsyncSession.close_all()` 方法无法正常工作的错误。还添加了函数 `close_all_sessions()` ，它等同于 `close_all_sessions()`。拉取请求由 Bryan 不可思议 提供。

    引用：[#10421](https://www.sqlalchemy.org/trac/ticket/10421)

### postgresql

+   **[postgresql] [bug]**

    修复了由 [#7744](https://www.sqlalchemy.org/trac/ticket/7744) 引起的 2.0 版本回归，其中涉及 PostgreSQL JSON 运算符的表达式链与其他运算符（如字符串连接）组合会丢失正确的括号化，这是由于特定于 PostgreSQL 方言的实现细节造成的。

    参考：[#10479](https://www.sqlalchemy.org/trac/ticket/10479)

+   **[postgresql] [bug]**

    修复了在使用 asyncpg 后端时使用 `BIT` 数据类型的 “insertmanyvalues” 的 SQL 处理问题。在 asyncpg 上，`BIT` 显然需要使用 asyncpg 特定的 `BitString` 类型，目前在使用此 DBAPI 时暴露，这使其与其他 PostgreSQL DBAPI 不兼容，所有这些 DBAPI 都在这里使用普通的位字符串。在版本 2.1 中，将通过将此数据类型在所有 PG 后端上归一化来解决此问题。拉取请求由 Sören Oldag 提供。

    参考：[#10532](https://www.sqlalchemy.org/trac/ticket/10532)

### mysql

+   **[mysql] [bug]**

    修复了 MySQL 的 “pre-ping” 例程中的一个新的不兼容性，其中传递给 `connection.ping()` 的 `False` 参数，用于禁用不想要的 “自动重新连接” 功能，正在被 MySQL 驱动程序和后端弃用，并且对某些版本的 MySQL 的本机客户端驱动程序产生警告。对于 mysqlclient，它已被移除，而对于 PyMySQL 和基于 PyMySQL 的驱动程序，该参数将在某个时候被弃用并移除，因此使用 API 内省来未来证明对这些移除的各个阶段进行了保护。

    这个更改也被**回溯**到了：1.4.50

    参考：[#10492](https://www.sqlalchemy.org/trac/ticket/10492)

### mariadb

+   **[mariadb] [bug]**

    调整了 MySQL / MariaDB 方言，当使用 MariaDB 时，默认将生成的列设置为 NULL，如果 `Column.nullable` 没有使用显式的 `True` 或 `False` 值进行指定，因为 MariaDB 不支持带有生成列的“NOT NULL”短语。拉取请求由 Indivar 提供。

    参考：[#10056](https://www.sqlalchemy.org/trac/ticket/10056)

+   **[mariadb] [bug] [regression]**

    建立了一个解决 MySQL/MariaDB 驱动程序中似乎存在的一个固有问题的方法，即对于使用 SQLAlchemy 的 “空 IN” 条件返回不返回任何行的 DELETE DML 的 RETURNING 结果，失败提供 cursor.description，然后返回没有行的结果，导致在 2.0 系列中为 ORM 使用 RETURNING 用于“同步会话”功能的批量 DELETE 语句时出现回归。为了解决这个问题，当检测到 “给出 RETURNING 时没有描述” 的特定情况时，会生成一个带有正确游标描述的“空结果”，并在非工作游标的位置使用它。

    参考：[#10505](https://www.sqlalchemy.org/trac/ticket/10505)

### mssql

+   **[mssql] [usecase]**

    为 SQL Server 实现的 `aioodbc` 驱动添加了支持，该驱动建立在 pyodbc 和通用 aio* 方言架构之上。

    另请参阅

    aioodbc - 在 SQL Server 方言文档中。

    参考：[#6521](https://www.sqlalchemy.org/trac/ticket/6521)

+   **[mssql] [bug] [reflection]**

    修复了对于具有大于 18 位数的大整数起始值的 bigint 列的身份列反射失败的问题。

    此更改也被 **回溯** 至：1.4.50

    参考：[#10504](https://www.sqlalchemy.org/trac/ticket/10504)

### oracle

+   **[oracle] [bug]**

    修复了 `Interval` 数据类型中的问题，在 Oracle 实现未用于 DDL 生成，导致 `day_precision` 和 `second_precision` 参数被忽略，尽管该方言支持这些参数。Indivar 提供的拉取请求。

    参考：[#10509](https://www.sqlalchemy.org/trac/ticket/10509)

+   **[oracle] [bug]**

    修复了 cx_Oracle 方言声称支持比实际在 SQLAlchemy 2.0 系列中支持的更低的 cx_Oracle 版本（7.x）的问题。方言导入了仅在 cx_Oracle 8 或更高版本中才存在的符号，因此运行时方言检查以及 setup.cfg 要求已更新以反映此兼容性。

    参考：[#10470](https://www.sqlalchemy.org/trac/ticket/10470)

## 2.0.22

发布日期：2023 年 10 月 12 日

### orm

+   **[orm] [usecase]**

    添加了 `Session.get_one()` 方法，其行为类似于 `Session.get()`，但是如果未找到具有提供的主键的实例，则引发异常而不是返回 `None`。由 Carlos Sousa 提供的拉取请求。

    参考：[#10202](https://www.sqlalchemy.org/trac/ticket/10202)

+   **[orm] [usecase]**

    添加了一个选项以永久关闭会话。将新参数 `Session.close_resets_only` 设置为 `False` 将阻止在调用 `Session.close()` 后执行任何其他操作。

    添加了新方法`Session.reset()`，它将一个`Session`重置为其初始状态。这是 `Session.close()` 的别名，除非 `Session.close_resets_only` 设置为`False`。

    参考：[#7787](https://www.sqlalchemy.org/trac/ticket/7787)

+   **[orm] [bug]**

    修复了一系列`mapped_column()`参数，在 pep-593 `Annotated` 对象中使用`mapped_column()`对象时未被传递，包括 `mapped_column.sort_order`，`mapped_column.deferred`，`mapped_column.autoincrement`，`mapped_column.system`，`mapped_column.info`等。

    另外，仍然不支持使用`mapped_column.kw_only`等数据类参数，这些参数在通过`Annotated`接收的`mapped_column()`中指定，因为这在 pep-681 数据类转换中不受支持。当以这种方式在`Annotated`中使用这些参数时，将发出警告（并且它们继续被忽略）。

    参考：[#10046](https://www.sqlalchemy.org/trac/ticket/10046), [#10369](https://www.sqlalchemy.org/trac/ticket/10369)

+   **[orm] [bug]**

    修复了使用 ORM 中的新式 `select()` 查询调用 `Result.unique()` 方法时的问题，在此查询中，一个或多个列产生的值是“未知可哈希性”，通常是在使用 `func.json_build_object()` 等 JSON 函数时没有提供类型时会导致内部失败。在这种情况下，修复了将对象作为接收到的对象测试其可哈希性的行为，并在不可哈希时引发一个信息性错误消息。请注意，对于“已知不可哈希性”的值，例如直接使用 `JSON` 或 `ARRAY` 类型时，已经会引发一个信息性错误消息。

    这里的“可哈希性测试”修复也适用于传统的 `Query`，然而在传统情况下，几乎所有的查询都使用 `Result.unique()`，因此这里不会发出新的警告；在这种情况下，保持了使用 `id()` 的传统行为，改进是一个未知类型如果被证明是可哈希的，那么现在将被独特化，而以前是不会的。

    参考：[#10459](https://www.sqlalchemy.org/trac/ticket/10459)

+   **[orm] [bug]**

    修复了最近修订的“insertmanyvalues”功能中的回归（可能是问题 [#9618](https://www.sqlalchemy.org/trac/ticket/9618)），在这种情况下，ORM 会不经意地将一个非 RETURNING 结果误解为具有 RETURNING 结果，这是因为将 `implicit_returning=False` 参数应用于映射的 `Table` 时指示“insertmanyvalues”不能在不提供主键值的情况下使用。

    参考：[#10453](https://www.sqlalchemy.org/trac/ticket/10453)

+   **[orm] [bug]**

    修复了 ORM `with_loader_criteria()` 不会应用于 `Select.join()` 的 bug，其中 ON 子句被给定为一个普通的 SQL 比较，而不是作为一个关系目标或类似的东西。

    参考：[#10365](https://www.sqlalchemy.org/trac/ticket/10365)

+   **[orm] [bug]**

    修复了当作为给定注释的子模块的元素引用时，无法正确解析 `Mapped` 符号（如 `WriteOnlyMapped` 和 `DynamicMapped`）的问题，假设注释是基于字符串或“未来注释”样式的。

    参考：[#10412](https://www.sqlalchemy.org/trac/ticket/10412)

+   **[orm] [bug]**

    修复了使用 `__allow_unmapped__` 声明选项时的问题，其中使用集合类型（例如 `list[SomeClass]`）声明的类型与使用 typing 构造 `List[SomeClass]` 的类型无法正确识别。由 Pascal Corpet 提供的拉取请求。

    参考：[#10385](https://www.sqlalchemy.org/trac/ticket/10385)

### engine

+   **[engine] [bug]**

    修复了一些方言中可能出现的问题，即方言可能会对根本不返回行的 INSERT 语句错误地返回空结果集，这是由于仍然存在来自行的主键的预获取或后获取的影响所致。受影响的方言包括 asyncpg 和所有的 mssql 方言。

+   **[engine] [bug]**

    修复了在某些垃圾回收/异常场景下，连接池的清理例程会由于意外的状态集而引发错误的问题，在特定条件下可以重现该问题。

    参考：[#10414](https://www.sqlalchemy.org/trac/ticket/10414)

### sql

+   **[sql] [bug]**

    修复了一个问题，即在 UPDATE 语句的 SET 子句中引用 FROM 条目时，如果该条目在语句中没有其他地方，则不会将其包括在 UPDATE 语句的 FROM 子句中；目前，对于使用 `Update.add_cte()` 添加的 CTE（通用表达式），以提供所需的 CTE 的情况，会出现这种情况。

    参考：[#10408](https://www.sqlalchemy.org/trac/ticket/10408)

+   **[sql] [bug]**

    修复了 2.0 版本的回归问题，即由于移除了未被考虑到的 `on` 属性而导致 `DDL` 结构不再执行 `__repr__()`。由 Iuri de Silvio 提供的拉取请求。

    参考：[#10443](https://www.sqlalchemy.org/trac/ticket/10443)

### typing

+   **[typing] [bug]**

    修复了传递给 `Values` 的参数列表过于严格地与 `List` 而不是 `Sequence` 绑定的类型问题。由 Iuri de Silvio 提供的拉取请求。

    参考：[#10451](https://www.sqlalchemy.org/trac/ticket/10451)

+   **[typing] [bug]**

    更新了代码库以支持 Mypy 1.6.0。

### asyncio

+   **[asyncio] [bug]**

    修复了未将`AsyncSession.get.execution_options`参数传播到底层`Session`并且被忽略的问题。

### mariadb

+   **[mariadb] [bug]**

    修改了 mariadb-connector 驱动程序，预加载所有查询的`cursor.rowcount`值，以适应像 Pandas 这样硬编码调用`Result.rowcount`的工具。SQLAlchemy 通常仅为 UPDATE/DELETE 语句预加载`cursor.rowcount`，否则会传递给 DBAPI，在那里如果没有值可用，则可以返回-1。然而，mariadb-connector 在关闭光标本身后不支持调用`cursor.rowcount`，而是引发错误。已添加通用测试支持，以确保所有后端支持在结果关闭后允许`Result.rowcount`成功（即返回一个整数值，-1 表示“不可用”）。

    参考：[#10396](https://www.sqlalchemy.org/trac/ticket/10396)

+   **[mariadb] [bug]**

    为 mariadb-connector 方言添加了额外的修复，以支持 INSERT..RETURNING 语句中结果中的 UUID 数据值。

### mssql

+   **[mssql] [bug]**

    修复了一个 bug，在这个 bug 中，阻止 ORDER BY 在 SQL Server 的子查询中发出的规则在使用`select.fetch()`方法限制行数与 WITH TIES 或 PERCENT 结合使用时未被禁用，从而阻止了可以使用带有 TOP / ORDER BY 的有效子查询。

    参考：[#10458](https://www.sqlalchemy.org/trac/ticket/10458)

### orm

+   **[orm] [usecase]**

    添加了类似于`Session.get()`但在未找到具有提供的主键的实例时引发异常而不是返回`None`的方法`Session.get_one()`。感谢 Carlos Sousa 的拉取请求。

    参考：[#10202](https://www.sqlalchemy.org/trac/ticket/10202)

+   **[orm] [usecase]**

    添加了一个选项来永久关闭会话。将新参数`Session.close_resets_only`设置为`False`将阻止`Session`在调用`Session.close()`后执行任何其他操作。

    添加了新方法`Session.reset()`，将`Session`重置为初始状态。这是`Session.close()`的别名，除非设置`Session.close_resets_only`为`False`。

    参考：[#7787](https://www.sqlalchemy.org/trac/ticket/7787)

+   **[orm] [bug]**

    修复了一系列`mapped_column()`参数，在使用`Annotated`对象内部的`mapped_column()`对象时未被传递，包括`mapped_column.sort_order`，`mapped_column.deferred`，`mapped_column.autoincrement`，`mapped_column.system`，`mapped_column.info` 等。

    此外，在`Annotated`中接收到

    参考：[#10046](https://www.sqlalchemy.org/trac/ticket/10046)，[#10369](https://www.sqlalchemy.org/trac/ticket/10369)

+   **[orm] [bug]**

    修复了在 ORM 中使用新风格的 `select()` 查询调用 `Result.unique()` 时的问题，在此情况下，如果一个或多个列产生的值是“未知的可哈希性”，通常是在使用像 `func.json_build_object()` 这样的 JSON 函数时没有提供类型时，会在返回的值实际上不可哈希时内部失败。此行为已修复，此时会对接收到的对象进行哈希性测试，如果不可哈希，则会引发一个信息性错误消息。请注意，对于“已知的不可哈希性”值，例如直接使用 `JSON` 或 `ARRAY` 类型时，已经会引发信息性错误消息。

    此处的“哈希性测试”修复也适用于传统的 `Query`，但在传统情况下，几乎所有查询都使用 `Result.unique()`，因此此处不会发出新的警告；在这种情况下，保留了使用 `id()` 的传统行为，改进是现在将被证明是可哈希的未知类型现在会被唯一化，而以前则不会。

    参考：[#10459](https://www.sqlalchemy.org/trac/ticket/10459)

+   **[orm] [bug]**

    修复了最近修订的“insertmanyvalues”功能中的回归（可能是问题 [#9618](https://www.sqlalchemy.org/trac/ticket/9618)），在这种情况下，如果将 `implicit_returning=False` 参数应用于映射的 `Table`，表示如果未提供主键值，则 ORM 会意外地尝试将非 RETURNING 结果解释为带有 RETURNING 结果，表明“insertmanyvalues”不能在不提供主键值的情况下使用。

    参考：[#10453](https://www.sqlalchemy.org/trac/ticket/10453)

+   **[orm] [bug]**

    修复了 ORM `with_loader_criteria()` 在 ON 子句被给定为普通 SQL 比较而不是作为关系目标或类似的情况下不应用于 `Select.join()` 的 bug。

    参考：[#10365](https://www.sqlalchemy.org/trac/ticket/10365)

+   **[orm] [bug]**

    修复了 `Mapped` 符号，例如 `WriteOnlyMapped` 和 `DynamicMapped` 在引用为给定注释的子模块的元素时无法正确解析的问题，假定使用基于字符串或“未来注释”样式注释。

    参考：[#10412](https://www.sqlalchemy.org/trac/ticket/10412)

+   **[ORM] [错误]**

    修复了 `__allow_unmapped__` 声明选项中的问题，其中使用集合类型（如 `list[SomeClass]`）声明的类型与使用 typing 构造 `List[SomeClass]` 相比将无法被正确识别。 感谢 Pascal Corpet 提供的拉取请求。

    参考：[#10385](https://www.sqlalchemy.org/trac/ticket/10385)

### 引擎

+   **[引擎] [错误]**

    修复了某些方言中的问题，其中方言可能会对根本不返回行的 INSERT 语句错误地返回空结果集，原因是仍然存在来自预先或后期获取行的主键的痕迹。受影响的方言包括 asyncpg，所有 mssql 方言。

+   **[引擎] [错误]**

    修复了在某些垃圾收集 / 异常情况下，连接池的清理例程会由于意外的状态集而引发错误的问题，该问题可以在特定条件下重现。

    参考：[#10414](https://www.sqlalchemy.org/trac/ticket/10414)

### SQL

+   **[SQL] [错误]**

    修复了在 UPDATE 语句的 SET 子句中引用 FROM 条目不会将其包括在 UPDATE 语句的 FROM 子句中的问题，如果该条目在语句中没有其他地方出现；这目前适用于通过 `Update.add_cte()` 添加的 CTE，以在语句顶部提供所需的 CTE。

    参考：[#10408](https://www.sqlalchemy.org/trac/ticket/10408)

+   **[SQL] [错误]**

    修复了 2.0 版中的回归，其中 `DDL` 构造由于已移除的 `on` 属性未被容纳而不再 `__repr__()`。 感谢 Iuri de Silvio 提供的拉取请求。

    参考：[#10443](https://www.sqlalchemy.org/trac/ticket/10443)

### 类型

+   **[类型] [错误]**

    修复了类型问题，其中传递给 `Values` 的参数列表过于严格地绑定到 `List` 而不是 `Sequence`。 感谢 Iuri de Silvio 提供的拉取请求。

    参考：[#10451](https://www.sqlalchemy.org/trac/ticket/10451)

+   **[类型] [错误]**

    更新了代码库以支持 Mypy 1.6.0。

### 异步 IO

+   **[异步 IO] [错误]**

    修复了未将 `AsyncSession.get.execution_options` 参数传播到底层 `Session` 并且被忽略的问题。

### mariadb

+   **[mariadb] [bug]**

    修改了 mariadb-connector 驱动程序，以预加载所有查询的 `cursor.rowcount` 值，以适应像 Pandas 这样硬编码调用 `Result.rowcount` 的工具。SQLAlchemy 通常仅为 UPDATE/DELETE 语句预加载 `cursor.rowcount`，否则传递给 DBAPI，在那里如果没有值可用则可以返回 -1。但是，mariadb-connector 不支持在关闭游标本身后调用 `cursor.rowcount`，而是引发错误。已添加通用测试支持，以确保所有后端支持在结果关闭后允许 `Result.rowcount` 成功（即返回一个整数值，-1 表示“不可用”）。

    参考：[#10396](https://www.sqlalchemy.org/trac/ticket/10396)

+   **[mariadb] [bug]**

    为 mariadb-connector 方言添加了额外的修复，以支持 INSERT..RETURNING 语句中结果中的 UUID 数据值。

### mssql

+   **[mssql] [bug]**

    修复了一个错误，即在 SQL Server 上阻止 ORDER BY 在子查询中发出的规则未在使用 `select.fetch()` 方法限制行数与 WITH TIES 或 PERCENT 结合时被禁用，导致无法使用带有 TOP / ORDER BY 的有效子查询。

    参考：[#10458](https://www.sqlalchemy.org/trac/ticket/10458)

## 2.0.21

发布日期：2023 年 9 月 18 日

### orm

+   **[orm] [bug]**

    调整了 ORM 对“target”实体的解释，用于 `Update` 和 `Delete` 中，以不干扰传递给语句的目标“from”对象，例如在传递 ORM 映射的 `aliased` 构造时应在“UPDATE FROM”等短语中保留。像使用“SELECT”语句进行 ORM 会话同步的情况，如与 MySQL/MariaDB 一起使用此类形式的 UPDATE/DELETE 仍然会有问题，因此最好在使用此类 DML 语句时禁用 synchonize_session。

    参考：[#10279](https://www.sqlalchemy.org/trac/ticket/10279)

+   **[orm] [bug]**

    为 `selectin_polymorphic()` 加载器选项添加了新的功能，允许其他加载器选项作为兄弟节点捆绑在其中，引用其子类之一，在父加载器选项的子选项中。以前，只有在查询的选项的顶层才支持此模式。参见新的文档部分示例。

    作为此更改的一部分，改进了`Load.selectin_polymorphic()`方法/加载策略的行为，因此在对已经关系加载的类使用该选项时，子类加载不会加载父表中已加载的大多数列。先前，仅对顶级类加载的逻辑才能仅加载子类列。

    另请参阅

    在 selectin_polymorphic 本身作为子选项时应用加载器选项

    参考：[#10348](https://www.sqlalchemy.org/trac/ticket/10348)

### 引擎

+   **[engine] [bug]**

    修复了一系列反射问题，影响到 PostgreSQL、MySQL/MariaDB 和 SQLite 方言，在反映外键约束时，目标列的表名或列名中包含括号的情况下。

    参考：[#10275](https://www.sqlalchemy.org/trac/ticket/10275)

### sql

+   **[sql] [用例]**

    调整了`Enum`数据类型，接受`Enum.length`参数的值为`None`，在生成的 DDL 中，结果为 VARCHAR 或其他文本类型而没有长度。这允许在模式中存在类型后向类型添加任意长度的新元素。感谢 Eugene Toder 的拉取请求。

    参考：[#10269](https://www.sqlalchemy.org/trac/ticket/10269)

+   **[sql] [用例]**

    添加了新的通用 SQL 函数`aggregate_strings`，接受一个 SQL 表达式和一个分隔符，将多行字符串连接为单个聚合值。该函数根据每个后端编译为诸如`group_concat()`、`string_agg()`或`LISTAGG()`等函数。感谢 Joshua Morris 的拉取请求。

    参考：[#9873](https://www.sqlalchemy.org/trac/ticket/9873)

+   **[sql] [bug]**

    调整了字符串连接运算符的操作符优先级，使其等于字符串匹配运算符的优先级，例如 `ColumnElement.like()`、`ColumnElement.regexp_match()`、`ColumnElement.match()` 等，以及与字符串比较运算符相同优先级的纯 `==`，这样括号将应用于跟在字符串匹配运算符后面的字符串连接表达式。这为后端，例如 PostgreSQL 提供了可能比字符串连接运算符优先级更高的 “regexp match” 运算符的情况。

    参考：[#9610](https://www.sqlalchemy.org/trac/ticket/9610)

+   **[sql] [bug]**

    在 DDL 编译器中对 `hashlib.md5()` 的使用进行了限定，该函数用于在 DDL 语句中为长索引和约束名称生成确定性的四字符后缀，以包括 Python 3.9+ 中的 `usedforsecurity=False` 参数，以便 Python 解释器构建为诸如 FIPS 之类的受限环境时不认为此调用与安全问题有关。

    参考：[#10342](https://www.sqlalchemy.org/trac/ticket/10342)

+   **[sql] [bug]**

    `Values` 构造现在将自动创建 `column` 的代理（即复制），如果该列已经与现有的 FROM 子句相关联。这允许像 `values_obj.c.colname` 这样的表达式即使在 `colname` 被传递为已经与以前的 `Values` 或其他表构造一起使用的 `column` 的情况下，也能产生正确的 FROM 子句。最初认为这可能是一个错误条件的候选项，但是很可能这种模式已经被广泛使用，所以现在添加了支持。

    参考：[#10280](https://www.sqlalchemy.org/trac/ticket/10280)

### schema

+   **[schema] [bug]**

    修改了仅适用于 Oracle 的`Identity.order`参数的渲染，该参数是`Sequence`和`Identity`的一部分，仅适用于 Oracle 后端，而不适用于其他后端，如 PostgreSQL。未来的版本将重命名`Identity.order`、`Sequence.order`和`Identity.on_null`参数为 Oracle 特定名称，弃用旧名称，这些参数仅适用于 Oracle。

    此更改也**回溯**到：1.4.50

    参考：[#10207](https://www.sqlalchemy.org/trac/ticket/10207)

### typing

+   **[typing] [usecase]**

    使`Mapped`的包含类型协变；这是为了允许更大的灵活性，以满足最终用户的类型化场景，例如使用协议来表示传递给其他函数的特定映射类结构。作为此更改的一部分，还使依赖和相关类型的包含类型协变，如`SQLORMOperations`、`WriteOnlyMapped`和`SQLColumnExpression`。拉取请求由 Roméo Després 提供。

    参考：[#10288](https://www.sqlalchemy.org/trac/ticket/10288)

+   **[typing] [bug]**

    修复了在 2.0.20 中引入的回归问题，通过 [#9600](https://www.sqlalchemy.org/trac/ticket/9600) 修复尝试为`MetaData.naming_convention`添加更正式的类型化。此更改阻止了基本命名约定字典通过类型化，并已调整为再次接受键为字符串的普通字典以及使用约束类型作为键或两者混合使用的字典。

    作为此更改的一部分，还对命名约定字典的较少使用形式进行了类型化，包括当前允许将`Constraint`类型对象用作键。

    参考：[#10264](https://www.sqlalchemy.org/trac/ticket/10264), [#9284](https://www.sqlalchemy.org/trac/ticket/9284)

+   **[typing] [bug]**

    修复了应用于表达式构造基类`Visitable`的`__class_getitem__()`的类型注释，使其接受`Any`作为键，而不是`str`，这有助于一些 IDE（如 PyCharm）在尝试为包含泛型选择器的 SQL 构造编写类型注释时。感谢 Jordan Macdonald 的拉取请求。

    参考资料：[#9878](https://www.sqlalchemy.org/trac/ticket/9878)

+   **[打字] [错误]**

    修复了核心“SQL 元素”类 `SQLCoreOperations` 以支持从类型角度来看的 `__hash__()` 方法，因为像 `Column` 和 ORM `InstrumentedAttribute` 这样的对象是可散列的，并且在 `Update` 和 `Insert` 构造的公共 API 中用作字典键。先前，类型检查器不知道根 SQL 元素是可散列的。

    参考资料：[#10353](https://www.sqlalchemy.org/trac/ticket/10353)

+   **[打字] [错误]**

    修复了使用 ORM 类时，`Existing.select_from()` 的类型注释问题。

    参考资料：[#10337](https://www.sqlalchemy.org/trac/ticket/10337)

+   **[打字] [错误]**

    更新 ORM 加载选项的类型注解，限制其只接受“*”而不是任何字符串作为字符串参数。感谢 Janek Nouvertné 的拉取请求。

    参考资料：[#10131](https://www.sqlalchemy.org/trac/ticket/10131)

### postgresql

+   **[postgresql] [错误]**

    修复了由于 [#8491](https://www.sqlalchemy.org/trac/ticket/8491) 在 2.0 中出现的回归问题，当使用 `create_engine.pool_pre_ping` 参数时，PostgreSQL 方言的修订“ping”会干扰 asyncpg 与 PGBouncer 的“transaction”模式的使用，因为 asnycpg 发出的多个 PostgreSQL 命令可能被分成多个连接导致错误，因为这种新修订的“ping”周围没有任何事务。现在，在事务内调用 ping，与所有其他基于 pep-249 DBAPI 的后端一样；这保证了由此命令发送的一系列 PG 命令在同一后端连接上被调用，而不是在命令执行中跳转到另一个连接。如果 asyncpg 方言以“AUTOCOMMIT”模式使用，则不使用事务，这与 pgbouncer 事务模式不兼容。

    参考资料：[#10226](https://www.sqlalchemy.org/trac/ticket/10226)

### 杂项

+   **[错误] [设置]**

    修复了一个很久以前的问题，即无法在 pytest 运行之外导入 SQLAlchemy 模块的全部范围，包括 `sqlalchemy.testing.fixtures`。这适用于诸如 `pkgutil` 等尝试在所有包中导入所有已安装模块的检查实用程序。

    参考：[#10321](https://www.sqlalchemy.org/trac/ticket/10321)

### orm

+   **[orm] [错误]**

    调整了 ORM 对于 `Update` 和 `Delete` 中使用的“target”实体的解释，以避免干扰语句中传递的目标“from”对象，比如传递 ORM 映射的 `aliased` 结构，在像“UPDATE FROM”这样的短语中应保持不变。像 ORM 会话同步使用“SELECT”语句的情况，比如 MySQL/MariaDB，仍然会出现这种形式的 UPDATE/DELETE 的问题，因此最好在使用此类 DML 语句时禁用 synchonize_session。

    参考：[#10279](https://www.sqlalchemy.org/trac/ticket/10279)

+   **[orm] [错误]**

    为 `selectin_polymorphic()` 加载器选项添加了新的功能，允许将其他加载器选项作为同级项捆绑在其中，引用其中一个子类，在父加载器选项的子选项中。以前，只有在查询的选项的顶级中才支持这种模式。请参阅新的文档部分以获取示例。

    作为这一变化的一部分，改进了 `Load.selectin_polymorphic()` 方法/加载器策略的行为，以便在已经对父表进行关系加载时，子类加载不会加载大部分已加载列。以前，仅加载子类列的逻辑仅适用于顶层类加载。

    参见

    在 selectin_polymorphic 本身是子选项时应用加载器选项

    参考：[#10348](https://www.sqlalchemy.org/trac/ticket/10348)

### engine

+   **[engine] [错误]**

    修复了一系列反射问题，影响到 PostgreSQL、MySQL/MariaDB 和 SQLite 方言，在反射外键约束时，目标列中包含括号的情况下，其中一个或两个表名或列名中都包含括号。

    参考：[#10275](https://www.sqlalchemy.org/trac/ticket/10275)

### sql

+   **[sql] [用例]**

    调整了`Enum`数据类型，使其接受`None`参数作为`Enum.length`参数，从而在生成的 DDL 中得到一个没有长度限制的 VARCHAR 或其他文本类型。这允许在模式中存在该类型后添加任意长度的新元素。感谢 Eugene Toder 的拉取请求。

    参考：[#10269](https://www.sqlalchemy.org/trac/ticket/10269)

+   **[sql] [usecase]**

    添加了新的通用 SQL 函数`aggregate_strings`，接受一个 SQL 表达式和一个分隔符，将多行字符串连接成单个聚合值。该函数根据每个后端编译成函数，如`group_concat()`、`string_agg()`或`LISTAGG()`。感谢 Joshua Morris 的拉取请求。

    参考：[#9873](https://www.sqlalchemy.org/trac/ticket/9873)

+   **[sql] [bug]**

    调整了字符串连接运算符的运算优先级，使其与字符串匹配运算符（如`ColumnElement.like()`、`ColumnElement.regexp_match()`、`ColumnElement.match()`等）以及普通的`==`运算符相等，这样括号将应用于跟在字符串匹配运算符后的字符串连接表达式。这为后端（如 PostgreSQL）提供了支持，其中“regexp match”运算符显然比字符串连接运算符的优先级高。

    参考：[#9610](https://www.sqlalchemy.org/trac/ticket/9610)

+   **[sql] [bug]**

    限定了 DDL 编译器中`hashlib.md5()`的使用，用于为 DDL 语句中的长索引和约束名称生成确定性的四字符后缀，以包括 Python 3.9+的`usedforsecurity=False`参数，以便 Python 解释器构建用于受限环境（如 FIPS）时不将此调用视为与安全问题相关。

    参考：[#10342](https://www.sqlalchemy.org/trac/ticket/10342)

+   **[sql] [bug]**

    `Values` 构造现在将自动创建一个代理（即复制），如果列已经与现有的 FROM 子句相关联。这样一来，即使`colname`已经作为一个`column`被传递给了先前的`Values`或其他表构造，表达式`values_obj.c.colname`也会产生正确的 FROM 子句。最初认为这可能会导致错误，但很可能这种模式已经被广泛使用，所以现在添加以支持。

    参考：[#10280](https://www.sqlalchemy.org/trac/ticket/10280)

### 模式

+   **[模式] [错误]**

    修改了仅适用于 Oracle 的`Identity.order`参数的呈现方式，该参数既属于`Sequence`又属于`Identity`，现在只在 Oracle 后端生效，而不是像 PostgreSQL 那样适用于其他后端。将来的版本将会将`Identity.order`、`Sequence.order`和`Identity.on_null`参数重命名为 Oracle 特定的名称，并弃用旧名称，这些参数仅适用于 Oracle。

    此更改也**回溯到**：1.4.50

    参考：[#10207](https://www.sqlalchemy.org/trac/ticket/10207)

### typing

+   **[类型] [用例]**

    对于`Mapped`中包含的类型进行了协变处理；这样做是为了在端用户的类型场景中提供更大的灵活性，比如使用协议来表示特定映射类结构，这些结构会传递给其他函数。作为这一变更的一部分，对于依赖和相关类型，如`SQLORMOperations`、`WriteOnlyMapped`和`SQLColumnExpression`，也对其中包含的类型进行了协变处理。感谢 Roméo Després 提交的拉取请求。

    参考：[#10288](https://www.sqlalchemy.org/trac/ticket/10288)

+   **[类型] [错误]**

    修复了在 2.0.20 中引入的回归问题，通过 [#9600](https://www.sqlalchemy.org/trac/ticket/9600) 修复，该修复尝试为 `MetaData.naming_convention` 添加更正式的类型注解。这一变更阻止了基本的命名约定字典通过类型检查，并已调整为再次接受纯字符串键的普通字典以及使用约束类型作为键或两者混合使用的字典。

    作为这一变更的一部分，还对命名约定字典的较少使用形式进行了类型化，其中目前允许 `Constraint` 类型对象作为键。

    参考：[#10264](https://www.sqlalchemy.org/trac/ticket/10264), [#9284](https://www.sqlalchemy.org/trac/ticket/9284)

+   **[typing] [bug]**

    修复了应用于表达式构造基础的 `Visitable` 类的 `__class_getitem__()` 的类型注解，以接受 `Any` 作为键，而不是 `str`，这有助于一些 IDE，如 PyCharm，在尝试为包含通用选择器的 SQL 构造编写类型注解时。感谢 Jordan Macdonald 的拉取请求。

    参考：[#9878](https://www.sqlalchemy.org/trac/ticket/9878)

+   **[typing] [bug]**

    修复了核心“SQL 元素”类 `SQLCoreOperations`，以支持从类型的角度看待 `__hash__()` 方法，因为像 `Column` 和 ORM `InstrumentedAttribute` 这样的对象是可哈希的，并且在公共 API 中用作字典键，用于 `Update` 和 `Insert` 构造。以前，类型检查器不知道根 SQL 元素是可哈希的。

    参考：[#10353](https://www.sqlalchemy.org/trac/ticket/10353)

+   **[typing] [bug]**

    修复了与 `Existing.select_from()` 的类型问题，这阻止了它与 ORM 类的使用。

    参考：[#10337](https://www.sqlalchemy.org/trac/ticket/10337)

+   **[typing] [bug]**

    更新了 ORM 加载选项的类型注解，将其限制为仅接受“*”而不是任何字符串作为字符串参数。感谢 Janek Nouvertné 的拉取请求。

    参考：[#10131](https://www.sqlalchemy.org/trac/ticket/10131)

### postgresql

+   **[postgresql] [bug]**

    修复了 2.0 版本中出现的回归问题，这是由于[#8491](https://www.sqlalchemy.org/trac/ticket/8491)导致的，其中当使用`create_engine.pool_pre_ping`参数时，用于 PostgreSQL 方言的修订“ping”会干扰 asyncpg 与 PGBouncer“事务”模式的使用，因为 asnycpg 发出的多个 PostgreSQL 命令可能会被分配到多个连接中，导致错误，由于对此新修订的“ping”周围没有任何事务。现在，在事务中调用 ping，就像所有其他基于 pep-249 DBAPI 的后端隐式使用的一样；这保证了由于此命令发送的 PG 命令系列会在同一后端连接上调用，而不是在命令中间跳到不同的连接。如果使用 asyncpg 方言处于“AUTOCOMMIT”模式下，则不使用事务，这仍然与 pgbouncer 事务模式不兼容。

    参考资料：[#10226](https://www.sqlalchemy.org/trac/ticket/10226)

### 杂项

+   **[错误] [设置]**

    修复了很旧的问题，即无法在 pytest 运行外导入 SQLAlchemy 模块的全部内容，包括`sqlalchemy.testing.fixtures`。这适用于诸如`pkgutil`等试图导入所有包中所有已安装模块的检查工具。

    参考资料：[#10321](https://www.sqlalchemy.org/trac/ticket/10321)

## 2.0.20

发布日期：2023 年 8 月 15 日

### orm

+   **[orm] [用例]**

    实现了 ORM 启用的 DML 语句的“RETURNING '*”用例。这将在尽可能多的情况下呈现，并返回未过滤的结果集，但不支持具有特定列呈现要求的多参数“ORM 批量 INSERT”语句。

    参考资料：[#10192](https://www.sqlalchemy.org/trac/ticket/10192)

+   **[orm] [错误]**

    修复了阻止某些形式的 ORM“注释”对使用`Select.join()`对关系目标进行连接的子查询进行的问题。在特殊情况下使用这些注释，例如在`PropComparator.and_()`和其他 ORM 特定场景中使用子查询时。

    此更改也被**回溯**到：1.4.50

    参考资料：[#10223](https://www.sqlalchemy.org/trac/ticket/10223)

+   **[orm] [错误]**

    修复了 ORM 从具有同名列的超类和子类的连接继承模型中生成 SELECT 时出现问题的问题，当生成 RECURSIVE 列列表时，不会发送正确的列名列表到`CTE`构造。。

    参考资料：[#10169](https://www.sqlalchemy.org/trac/ticket/10169)

+   **[orm] [错误]**

    修复了一个相当严重的问题，即传递给`Session.execute()`的执行选项以及本身 ORM 执行的语句的执行选项将不会传递给 eager loaders，例如 `selectinload()`、`immediateload()` 和 `sqlalchemy.orm.subqueryload()`，从而使得禁用单个语句的缓存或者为单个语句使用 `schema_translate_map` 等操作变得不可能，以及使用用户自定义执行选项。已经做出了更改，即现在所有对于 `Session.execute()` 的用户界面执行选项都将传递到附加的 loaders 中。

    作为这一更改的一部分，对于导致禁用缓存的“过度深层” eager loaders 的警告现在可以通过向 `Session.execute()` 发送 `execution_options={"compiled_cache": None}` 来在每个语句的基础上消除，这将禁用该范围内所有语句的缓存。

    参考文献：[#10231](https://www.sqlalchemy.org/trac/ticket/10231)

+   **【orm】【错误】**

    修复了 ORM 内部克隆的问题，该克隆用于像 `Comparator.any()` 这样的表达式，以生成相关 EXISTS 构造，会干扰 SQL 编译器的“笛卡尔积警告”特性，导致 SQL 编译器在语句的所有元素都正确连接时发出警告。

    参考文献：[#10124](https://www.sqlalchemy.org/trac/ticket/10124)

+   **【orm】【错误】**

    修复了`lazy="immediateload"`加载策略在某些情况下会将内部加载令牌放入 ORM 映射属性中的问题，例如在递归自引用加载时不应发生加载。作为这一更改的一部分，`lazy="immediateload"`策略现在与其他 eager loaders 一样尊重 `relationship.join_depth` 参数，对于自引用 eager 加载，如果将其设置为未设置或设置为零，将不会发生自引用 immediateload，如果将其设置为大于零的值，将会立即加载直到给定的深度。

    参考文献：[#10139](https://www.sqlalchemy.org/trac/ticket/10139)

+   **【orm】【错误】**

    修复了一个问题，即基于字典的集合（如`attribute_keyed_dict()`）未正确地完全序列化/反序列化，导致在反序列化后尝试突变此类集合时出现问题。

    参考文献：[#10175](https://www.sqlalchemy.org/trac/ticket/10175)

+   **[orm] [bug]**

    修复了一个问题，即从另一个急切加载器使用`aliased()`对于连接的继承子类链的超类的本地列失败时，链接`load_only()`或其他通配符使用`defer()`。

    参考文献：[#10125](https://www.sqlalchemy.org/trac/ticket/10125)

+   **[orm] [bug]**

    修复了一个问题，其中启用 ORM 的`select()`结构不会呈现仅通过`Select.add_cte()`方法添加的任何 CTEs，而这些 CTEs 在语句中没有被引用。

    参考文献：[#10167](https://www.sqlalchemy.org/trac/ticket/10167)

### 示例

+   **[examples] [bug]**

    dogpile_caching 示例已更新为 2.0 样式的查询。在“缓存查询”逻辑内部，添加了一个条件来区分`Query`和`select()`在执行无效操作时。

### 引擎

+   **[engine] [bug]**

    修复了一个严重问题，即将`create_engine.isolation_level`设置为`AUTOCOMMIT`（而不是使用`Engine.execution_options()`方法），如果临时选择了替代隔离级别，则会导致无法将“autocommit”恢复到池化连接中，使用`Connection.execution_options.isolation_level`。

    参考文献：[#10147](https://www.sqlalchemy.org/trac/ticket/10147)

### sql

+   **[sql] [bug]**

    修复了一个问题，即对`Column`或其他`ColumnElement`进行反序列化会失败恢复正确的“比较器”对象，该对象用于生成特定于类型对象的 SQL 表达式。

    此更改也已**回溯**至：1.4.50

    参考文献：[#10213](https://www.sqlalchemy.org/trac/ticket/10213)

### 类型

+   **[typing] [usecase]**

    添加了新的仅用于类型的实用函数`Nullable()`和`NotNullable()`，用于分别将列或 ORM 类的类型定义为可空或不可空。这些函数在运行时不起作用，返回不变的输入。

    参考：[#10173](https://www.sqlalchemy.org/trac/ticket/10173)

+   **[typing] [bug]**

    类型改进：

    +   当使用无返回的 DML 时，对于某些形式的`Session.execute()`，将返回`CursorResult`

    +   修正了`Query.with_for_update.of`参数的类型，该参数在`Query.with_for_update()`中被修复为正确的类型。

    +   改进了某些 DML 方法中使用的 `_DMLColumnArgument` 类型，用于传递列表达式

    +   给 `literal()` 添加了重载，以便在未提供`literal.type_`参数时推断出返回类型为 `BindParameter[NullType]`

    +   给 `ColumnElement.op()` 添加了重载，以便在未提供`ColumnElement.op.return_type`时推断出的类型为 `Callable[[Any], BinaryExpression[Any]]`

    +   给 `ColumnElement.__add__()` 添加了缺失的重载

    Pull request 由 Mehdi Gmira 提供。

    参考：[#9185](https://www.sqlalchemy.org/trac/ticket/9185)

+   **[typing] [bug]**

    修复了`Session`和`AsyncSession`方法中的问题，例如`Session.connection()`，其中`Session.connection.execution_options`参数被硬编码为不面向用户的内部类型。

    参考：[#10182](https://www.sqlalchemy.org/trac/ticket/10182)

### asyncio

+   **[asyncio] [usecase]**

    添加了新方法 `AsyncConnection.aclose()` 作为 `AsyncConnection.close()` 的同义词，以及 `AsyncSession.aclose()` 作为 `AsyncSession.close()` 的同义词，添加到 `AsyncConnection` 和 `AsyncSession` 对象中，以提供与 Python 标准库 `@contextlib.aclosing` 构造的兼容性。拉取请求由 Grigoriev Semyon 提供。

    参考：[#9698](https://www.sqlalchemy.org/trac/ticket/9698)

### mysql

+   **[mysql] [usecase]**

    由于 aiomysql 方言再次得到维护，已更新 aiomysql 方言。重新添加到 ci 测试中，使用版本 0.2.0。

    此更改也 **回溯** 到：1.4.50

### orm

+   **[orm] [usecase]**

    实现了 ORM 启用的 DML 语句的 “RETURNING ‘*’” 用例。这将尽可能地呈现，并返回未经过滤的结果集，但不支持具有特定列呈现要求的多参数 “ORM 批量插入” 语句。

    参考：[#10192](https://www.sqlalchemy.org/trac/ticket/10192)

+   **[orm] [bug]**

    修复了一些形式的 ORM “注释” 无法对使用 `Select.join()` 进行的子查询进行注释的根本问题，这些子查询在特殊情况下使用，比如在 `PropComparator.and_()` 和其他 ORM 特定场景中使用。

    此更改也 **回溯** 到：1.4.50

    参考：[#10223](https://www.sqlalchemy.org/trac/ticket/10223)

+   **[orm] [bug]**

    修复了 ORM 从具有同名列的超类和子类的联合继承模型生成 SELECT 时，当生成递归列列表时，某种方式未正确发送列名列表到 `CTE` 构造的问题，当递归列列表被生成时。

    参考：[#10169](https://www.sqlalchemy.org/trac/ticket/10169)

+   **[orm] [bug]**

    修复了将执行选项传递给`Session.execute()`以及 ORM 执行语句本身的执行选项不会传播到`selectinload()`、`immediateload()`和`sqlalchemy.orm.subqueryload()`等急切加载器的问题，使得无法禁用单个语句的缓存或对单个语句使用`schema_translate_map`，以及使用用户自定义执行选项。已经进行了更改，**所有**针对`Session.execute()`的用户可见执行选项都将传播到其他加载器。

    作为这一变化的一部分，可以通过向`Session.execute()`发送`execution_options={"compiled_cache": None}`来在每个语句的基础上消除“过度深入”急切加载器导致缓存被禁用的警告，这将禁用该范围内所有语句的缓存。

    参考：[#10231](https://www.sqlalchemy.org/trac/ticket/10231)

+   **[orm] [bug]**

    修复了内部克隆在 ORM 中用于生成类似`Comparator.any()`的表达式以产生相关的 EXISTS 结构时会干扰 SQL 编译器的“笛卡尔积警告”功能的问题，导致 SQL 编译器在所有语句元素正确连接时发出警告。

    参考：[#10124](https://www.sqlalchemy.org/trac/ticket/10124)

+   **[orm] [bug]**

    修复了`lazy="immediateload"`加载策略在某些情况下会将内部加载标记放入 ORM 映射属性中的问题，例如在递归自引用加载中不应发生加载的情况。作为这一变化的一部分，`lazy="immediateload"`策略现在以与其他急切加载器相同的方式尊重`relationship.join_depth`参数用于自引用急切加载，其中将其未设置或设置为零将导致自引用的 immediateload 不会发生，将其设置为一个或更大的值将会 immediateload 直到给定深度。

    参考：[#10139](https://www.sqlalchemy.org/trac/ticket/10139)

+   **[orm] [bug]**

    修复了诸如`attribute_keyed_dict()`之类基于字典的集合在反序列化时未能完全正确地 pickle/unpickle 的问题，导致在反序列化后尝试修改此类集合时出现问题。

    参考：[#10175](https://www.sqlalchemy.org/trac/ticket/10175)

+   **[orm] [bug]**

    修复了从另一个急切加载器使用`aliased()`针对连接继承子类的情况下，链式调用`load_only()`或其他通配符使用`defer()`会导致对于超类本地列无法生效的问题。

    参考：[#10125](https://www.sqlalchemy.org/trac/ticket/10125)

+   **[orm] [bug]**

    修复了 ORM 启用的`select()`构造不会呈现仅通过`Select.add_cte()`方法添加的任何 CTE，而这些 CTE 在语句中没有被引用。

    参考：[#10167](https://www.sqlalchemy.org/trac/ticket/10167)

### examples

+   **[examples] [bug]**

    dogpile_caching 示例已更新为 2.0 风格的查询。在“缓存查询”逻辑中，添���了一个条件来区分在执行无效操作时`Query`和`select()`之间的区别。

### engine

+   **[engine] [bug]**

    修复了将`create_engine.isolation_level`设置为`AUTOCOMMIT`（而不是使用`Engine.execution_options()`方法）会导致如果使用`Connection.execution_options.isolation_level`临时选择了替代隔离级别，则无法将“autocommit”恢复到池化连接的关键问题。

    参考：[#10147](https://www.sqlalchemy.org/trac/ticket/10147)

### sql

+   **[sql] [bug]**

    修复了反序列化`Column`或其他`ColumnElement`时无法恢复正确的“比较器”对象的问题，该对象用于生成特定于类型对象的 SQL 表达式。

    此更改也**回溯**到：1.4.50

    参考：[#10213](https://www.sqlalchemy.org/trac/ticket/10213)

### typing

+   **[typing] [usecase]**

    添加了新的仅用于类型的实用函数`Nullable()`和`NotNullable()`，用于分别将列或 ORM 类类型化为可空或不可空。这些函数在运行时不起作用，返回不变的输入。

    参考：[#10173](https://www.sqlalchemy.org/trac/ticket/10173)

+   **[typing] [bug]**

    类型改进：

    +   一些形式的`Session.execute()`中返回了`CursorResult`，其中使用了没有返回的 DML

    +   修正了`Query.with_for_update.of`参数的类型，在`Query.with_for_update()`内部。

    +   对一些 DML 方法使用的`_DMLColumnArgument`类型进行了改进，以传递列表达式。

    +   添加了对`literal()`的重载，以便推断返回类型为`BindParameter[NullType]`，其中`literal.type_`参数为 None

    +   添加了对`ColumnElement.op()`的重载，以便在未提供`ColumnElement.op.return_type`时推断类型为`Callable[[Any], BinaryExpression[Any]]`。

    +   添加了对`ColumnElement.__add__()`的缺失重载

    拉取请求由 Mehdi Gmira 提供。

    参考：[#9185](https://www.sqlalchemy.org/trac/ticket/9185)

+   **[typing] [bug]**

    修复了`Session`和`AsyncSession`方法中的问题，例如`Session.connection()`，其中`Session.connection.execution_options`参数被硬编码为不是面向用户的内部类型。

    参考：[#10182](https://www.sqlalchemy.org/trac/ticket/10182)

### asyncio

+   **[asyncio] [usecase]**

    新增了`AsyncConnection.aclose()`作为`AsyncConnection.close()`的同义词，以及`AsyncSession.aclose()`作为`AsyncSession.close()`的同义词，用于`AsyncConnection`和`AsyncSession`对象，以与 Python 标准库`@contextlib.aclosing`构造兼容。感谢 Grigoriev Semyon 提交的拉取请求。

    参考：[#9698](https://www.sqlalchemy.org/trac/ticket/9698)

### mysql

+   **[mysql] [usecase]**

    更新了 aiomysql 方言，因为该方言似乎再次得到维护。重新使用版本 0.2.0 进行 ci 测试。

    此更改也已**回溯**至：1.4.50

## 2.0.19

发布日期：2023 年 7 月 15 日

### orm

+   **[orm] [bug]**

    修复了直接设置关系集合的问题，其中新集合中的对象已经存在时，不会触发该对象的级联事件，导致如果该对象尚未存在，则不会添加到`Session`中。这与[#6471](https://www.sqlalchemy.org/trac/ticket/6471)类似，并且由于在 2.0 系列中删除了`cascade_backrefs`，这个问题更加明显。作为[#6471](https://www.sqlalchemy.org/trac/ticket/6471)的一部分，现在还为已存在于相同集合的批量设置中的现有成员触发`AttributeEvents.append_wo_mutation()`事件。

    参考：[#10089](https://www.sqlalchemy.org/trac/ticket/10089)

+   **[orm] [bug]**

    修复了通过 backref 与未合并到 `Session` 中的加载集合相关联的对象的问题，由于在 2.0 系列中删除了 `cascade_backrefs`，这些对象不会发出警告，即使它们是集合的待定成员；在其他类似情况下，当要刷新的集合包含将被实质性丢弃的未附加对象时，会发出警告。为 backref-待定集合成员添加警告增加了与可能存在或不存在的集合以及基于不同关系加载策略在不同时间可能刷新或不刷新的集合的一致性。

    引用：[#10090](https://www.sqlalchemy.org/trac/ticket/10090)

+   **[orm] [bug] [regression]**

    通过 [#9805](https://www.sqlalchemy.org/trac/ticket/9805) 引起的额外回归被修复，其中对语句上的“ORM”标志更积极的传播可能导致在包含没有 ORM 实体的 ORM `Query` 构造中引发内部属性错误，即使在这种情况下 ORM 启用的 UPDATE 和 DELETE 语句中也不包含 ORM 实体。

    引用：[#10098](https://www.sqlalchemy.org/trac/ticket/10098)

### engine

+   **[engine] [bug]**

    将 `Row.t` 和 `Row.tuple()` 重命名为 `Row._t` 和 `Row._tuple()`；这是为了遵循所有在`Row`上的方法和预定义属性应该以 Python 标准库 `namedtuple` 的风格命名的政策，所有固定名称都有一个前导下划线，以避免与现有列名称发生冲突。以前的方法/属性现在已被弃用，并将发出弃用警告。

    引用：[#10093](https://www.sqlalchemy.org/trac/ticket/10093)

+   **[engine] [bug]**

    对`make_url()`函数添加了对非字符串、非`URL`对象的检测，允许立即抛出`ArgumentError`，而不是在后来导致失败。特殊逻辑确保模拟形式的`URL`可以通过。感谢 Grigoriev Semyon 的拉取请求。

    引用：[#10079](https://www.sqlalchemy.org/trac/ticket/10079)

### postgresql

+   **[postgresql] [bug]**

    由于在[#10004](https://www.sqlalchemy.org/trac/ticket/10004)中对 PostgreSQL URL 解析进行了改进，导致的回归问题已修复，其中“host”查询字符串参数中包含冒号，以支持各种第三方代理服务器和/或方言，将无法正确解析，因为这些被视为`host:port`组合。解析已更新，只有当主机名仅包含字母数字字符，点或破折号（例如没有斜杠），后跟一个冒号，然后跟着一个零个或多个整数的令牌时，才将冒号视为指示`host:port`值。在所有其他情况下，整个字符串被视为主机。

    参考：[#10069](https://www.sqlalchemy.org/trac/ticket/10069)

+   **[postgresql] [bug]**

    修复了与`CITEXT`数据类型的比较会将右侧转换为`VARCHAR`的问题，导致右侧不被解释为`CITEXT`数据类型，适用于 asyncpg、psycopg3 和 pg80000 方言。这导致`CITEXT`类型在实际使用中基本上无法使用；现在已修复此问题，并且测试套件已经更正，以正确断言表达式是否被正确渲染。

    参考：[#10096](https://www.sqlalchemy.org/trac/ticket/10096)

### orm

+   **[orm] [bug]**

    修复了直接设置关系集合的问题，其中新集合中的对象已经存在时，不会触发该对象的级联事件，导致如果该对象不存在，则不会被添加到`Session`中。这与[#6471](https://www.sqlalchemy.org/trac/ticket/6471)类似，并且由于在 2.0 系列中删除了`cascade_backrefs`，这个问题更加明显。作为[#6471](https://www.sqlalchemy.org/trac/ticket/6471)的一部分添加的`AttributeEvents.append_wo_mutation()`事件现在也会对已存在于同一集合的批量设置中的现有成员发出。

    参考：[#10089](https://www.sqlalchemy.org/trac/ticket/10089)

+   **[orm] [bug]**

    修复了一个问题，即通过反向引用与未合并到`Session`的未加载集合相关联的对象，因为在 2.0 系列中删除了`cascade_backrefs`，所以不会发出警告，即这些对象未被包含在刷新中，即使它们是集合的待处理成员；在其他类似情况下，当正在刷新的集合包含将被基本丢弃的非附加对象时，将发出警告。对于反向引用挂起的集合成员添加警告，可以建立更大一致性，这些集合可能存在或不存在，并可能根据不同的关系加载策略在不同的时间进行刷新或不刷新。

    参考：[#10090](https://www.sqlalchemy.org/trac/ticket/10090)

+   **[orm] [bug] [regression]**

    修复了由[#9805](https://www.sqlalchemy.org/trac/ticket/9805)导致的额外回归，其中对语句上的“ORM”标志的更积极传播可能导致在嵌入了 ORM `Query` 构造的核心 SQL 语句中包含没有 ORM 实体的情况下导致内部属性错误，即使在这种情况下，ORM 启用的 UPDATE 和 DELETE 语句也不包含 ORM 实体。

    参考：[#10098](https://www.sqlalchemy.org/trac/ticket/10098)

### 引擎

+   **[engine] [bug]**

    将`Row.t`和`Row.tuple()`重命名为`Row._t`和`Row._tuple()`；这是为了符合所有方法和预定义属性在`Row`上应采用 Python 标准库`namedtuple`风格的政策，其中所有固定名称都带有前导下划线，以避免与现有列名称发生冲突。以前的方法/属性现已弃用，并将发出弃用警告。

    参考：[#10093](https://www.sqlalchemy.org/trac/ticket/10093)

+   **[engine] [bug]**

    添加了对非字符串、非`URL`对象的检测到`make_url()`函数，允许立即抛出`ArgumentError`，而不是后来导致失败。特殊逻辑确保允许通过 `URL` 的模拟形式。感谢 Grigoriev Semyon 的拉取请求。

    参考：[#10079](https://www.sqlalchemy.org/trac/ticket/10079)

### postgresql

+   **[postgresql] [bug]**

    修复了在[#10004](https://www.sqlalchemy.org/trac/ticket/10004)中改进 PostgreSQL URL 解析时引起的回归，其中在主机查询字符串参数中含有冒号的情况下，以支持各种第三方代理服务器和/或方言，将无法正确解析，因为这些被评估为`host:port`组合。 解析已更新为仅在主机名仅包含字母数字字符以及仅包含点或破折号（例如没有斜杠）的情况下，考虑冒号表示`host:port`值，后跟零个或多个整数的全整数标记的情况下，才表示主机。 在所有其他情况下，将完整字符串视为主机。

    引用：[#10069](https://www.sqlalchemy.org/trac/ticket/10069)

+   **[postgresql] [bug]**

    修复了与`CITEXT`数据类型的比较问题，导致右侧被转换为`VARCHAR`，导致右侧未被解释为`CITEXT`数据类型，适用于 asyncpg、psycopg3 和 pg80000 方言。 这导致`CITEXT`类型在实际使用中基本不可用；现已修复此问题，并已更正测试套件以正确断言表达式是否被正确呈现。

    引用：[#10096](https://www.sqlalchemy.org/trac/ticket/10096)

## 2.0.18

发布日期：2023 年 7 月 5 日

### 引擎

+   **[引擎] [bug]**

    调整了`create_engine.schema_translate_map`功能，使得语句中的**所有**模式名称现在都被标记化，而不管特定名称是否在给定的立即模式翻译映射中，并且在执行时当键不在实际模式翻译映射中时回退到替换原始名称。 这两个更改允许在每次运行时使用包含或不包含各种键的模式翻译映射来重复使用已编译的对象，从而使得当每次使用时都使用具有不同键集的模式翻译映射时，缓存的 SQL 结构可以继续在运行时正常工作。 另外，还添加了在相同语句的调用间获得或失去`None`键的 schema_translate_map 字典的检测，这会影响语句的编译，并且与缓存不兼容； 这些情况下会引发异常。

    引用：[#10025](https://www.sqlalchemy.org/trac/ticket/10025)

### sql

+   **[sql] [bug]**

    修复了当使用“标志”时 `ColumnOperators.regexp_match()` 不会产生“稳定”的缓存密钥的问题，即，缓存密钥每次都会更改，导致缓存污染。对于带有标志和实际替换表达式的 `ColumnOperators.regexp_replace()` 也存在相同的问题。现在，标志被表示为固定的修饰符字符串，呈现为 safestring，而不是绑定参数，替换表达式在“二进制”元素的主要部分中确定，因此它生成适当的缓存密钥。

    请注意，作为此更改的一部分，`ColumnOperators.regexp_match.flags` 和 `ColumnOperators.regexp_replace.flags` 已修改为仅渲染为文字字符串，而以前它们被渲染为完整的 SQL 表达式，通常是绑定参数。这些参数应始终作为普通的 Python 字符串传递，而不是作为 SQL 表达式构造；不希望在实践中使用 SQL 表达式构造此参数，因此这是一个不兼容的更改。

    此更改还修改了生成的表达式的内部结构，对于带有或不带有标志的 `ColumnOperators.regexp_replace()` 和带有标志的 `ColumnOperators.regexp_match()`。可能已经实现了自己的正则表达式实现的第三方方言（在搜索中找不到此类方言，因此预期影响很小）需要调整结构的遍历以适应。

    此更改还 **回溯到**：1.4.49

    参考：[#10042](https://www.sqlalchemy.org/trac/ticket/10042)

+   **[sql] [错误]**

    修复了在主要内部`CacheKey` 构造中的问题，其中`__ne__()` 运算符未正确实现，导致当比较 `CacheKey` 实例时产生荒谬的结果。

    此更改还 **回溯到**：1.4.49

### 扩展

+   **[扩展] [用例]**

    向 `association_proxy()` `association_proxy.create_on_none_assignment` 添加了新选项；当一个关联代理引用一个标量关系并被赋予值 `None`，并且引用的对象不存在时，通过创建者创建一个新对象。这显然是 1.2 系列中的一个未定义行为，已经被悄悄移除了。

    参考资料：[#10013](https://www.sqlalchemy.org/trac/ticket/10013)

### 类型提示

+   **[类型提示] [用例]**

    当使用`sqlalchemy.sql.operators`中的独立运算符函数（例如`sqlalchemy.sql.operators.eq`）时，改进了类型提示。

    参考资料：[#10054](https://www.sqlalchemy.org/trac/ticket/10054)

+   **[类型提示] [错误]**

    修正了 `aliased()` 结构内部的一些类型提示，以正确接受已使用 `Table.alias()` 别名的 `Table` 对象，以及对传递为“可选择”参数的 `FromClause` 对象的一般支持，因为这都得到了支持。

    参考资料：[#10061](https://www.sqlalchemy.org/trac/ticket/10061)

### postgresql

+   **[postgresql] [用例]**

    为 asyncpg 方言添加了多主机支持。还添加了对“多主机”用例的 PostgreSQL URL 例程的一般改进和错误检查。拉取请求由 Ilia Dmitriev 提供。

    请参阅

    多主机连接

    参考资料：[#10004](https://www.sqlalchemy.org/trac/ticket/10004)

+   **[postgresql] [错误]**

    向所有 PostgreSQL 方言添加了新参数 `native_inet_types=False`，该参数指示 DBAPI 使用的转换器将 PostgreSQL 的 `INET` 和 `CIDR` 列中的行转换为 Python `ipaddress` 数据类型时应禁用，返回字符串。这样，编写用于这些数据类型的字符串的代码可以在无需代码更改的情况下添加此参数到 `create_engine()` 或 `create_async_engine()` 函数调用中而迁移到 asyncpg、psycopg 或 pg8000。

    请参阅

    网络数据类型

    参考：[#9945](https://www.sqlalchemy.org/trac/ticket/9945)

### mariadb

+   **[mariadb] [用例] [反射]**

    允许从 MariaDB 反射 `UUID` 列。这使得 Alembic 能够正确地检测现有 MariaDB 数据库中这些列的类型。

    参考：[#10028](https://www.sqlalchemy.org/trac/ticket/10028)

### mssql

+   **[mssql] [用例]**

    在 MSSQL 方言中添加了对 COLUMNSTORE 索引的创建和反射支持。可以在指定了 `mssql_columnstore=True` 的索引上指定。

    参考：[#7340](https://www.sqlalchemy.org/trac/ticket/7340)

+   **[mssql] [错误] [sql]**

    修复了在对具有显式排序规则的字符串类型执行 `Cast` 时会在 CAST 函数内部呈现 COLLATE 子句的问题，从而导致语法错误。

    参考：[#9932](https://www.sqlalchemy.org/trac/ticket/9932)

### 引擎

+   **[引擎] [错误]**

    调整了 `create_engine.schema_translate_map` 功能，使得语句中的**所有**模式名称现在都被标记化，无论指定了具体名称是否在立即模式翻译映射中，都会在执行时回退到原始名称。这两个变化允许对具有包含或不包含不同键集的模式翻译映射的编译对象进行重复使用，每次运行时使用不同的模式翻译映射，从而使得缓存的 SQL 构造在运行时继续工作。此外，增加了在同一语句的不同调用中增加或减少 `None` 键的 schema_translate_map 字典的检测，这会影响语句的编译，并且不兼容缓存；对于这些情况，会引发异常。

    参考：[#10025](https://www.sqlalchemy.org/trac/ticket/10025)

### SQL

+   **[sql] [错误]**

    修复了使用“flags”时 `ColumnOperators.regexp_match()` 未生成“稳定”缓存键的问题，即每次缓存键都会发生变化，导致缓存污染。相同的问题也存在于带有 flags 和实际替换表达式的 `ColumnOperators.regexp_replace()` 中。现在，flags 被表示为固定的修改器字符串，呈现为安全字符串，而不是绑定参数，并且替换表达式在“binary”元素的主要部分内建立，以生成适当的缓存键。

    注意，作为这一变更的一部分，`ColumnOperators.regexp_match.flags` 和 `ColumnOperators.regexp_replace.flags` 已经修改为仅呈现为字面字符串，而以前它们呈现为完整的 SQL 表达式，通常是绑定参数。这些参数应始终作为普通的 Python 字符串传递，而不是作为 SQL 表达式构造；预计实践中不会使用 SQL 表达式构造来传递此参数，因此这是一个不兼容的变更。

    此变更还修改了生成的表达式的内部结构，用于带有或不带有 flags 的 `ColumnOperators.regexp_replace()`，以及带有 flags 的 `ColumnOperators.regexp_match()`。可能已经实现了自己的正则表达式的第三方方言（在搜索中找不到这样的方言，因此预期影响很小）需要调整结构的遍历以适应。

    此变更还 **回溯** 到：1.4.49

    参考文献：[#10042](https://www.sqlalchemy.org/trac/ticket/10042)

+   **[SQL] [错误]**

    修复了大部分内部 `CacheKey` 构造中 `__ne__()` 运算符未正确实现的问题，导致在将 `CacheKey` 实例相互比较时产生荒谬的结果。

    此变更还 **回溯** 到：1.4.49

### 扩展

+   **[扩展] [用例]**

    为`association_proxy()`添加了新选项`association_proxy.create_on_none_assignment`; 当将引用标量关系的关联代理分配为`None`值时，并且引用的对象不存在时，通过创建器创建一个新对象。在 1.2 系列中，这显然是一种未定义的行为，被默默删除了。

    参考：[#10013](https://www.sqlalchemy.org/trac/ticket/10013)

### 输入

+   **[输入] [用例]**

    当使用来自`sqlalchemy.sql.operators`的独立运算符函数（如`sqlalchemy.sql.operators.eq`）时，改进了类型提示。

    参考：[#10054](https://www.sqlalchemy.org/trac/ticket/10054)

+   **[输入] [错误]**

    修复了`aliased()`构造中的一些类型提示，以正确接受已使用`Table.alias()`别名的`Table`对象，以及一般性对于`FromClause`对象作为“selectable”参数的支持，因为这都是受支持的。

    参考：[#10061](https://www.sqlalchemy.org/trac/ticket/10061)

### postgresql

+   **[postgresql] [用例]**

    为 asyncpg 方言添加了多主机支持。还为“多主机”用例的 PostgreSQL URL 例程添加了一般性改进和错误检查。感谢 Ilia Dmitriev 的拉取请求。

    参见

    多主机连接

    参考：[#10004](https://www.sqlalchemy.org/trac/ticket/10004)

+   **[postgresql] [错误]**

    对所有 PostgreSQL 方言添加了新参数`native_inet_types=False`，指示 DBAPI 使用的转换器将 PostgreSQL `INET`和`CIDR`列的行转换为 Python `ipaddress` 数据类型时应禁用，而是返回字符串。这允许为这些数据类型编写使用字符串的代码迁移到 asyncpg、psycopg 或 pg8000 而无需其他代码更改，只需将此参数添加到`create_engine()`或`create_async_engine()`函数调用中。

    参见

    网络数据类型

    参考：[#9945](https://www.sqlalchemy.org/trac/ticket/9945)

### mariadb

+   **[mariadb] [usecase] [reflection]**

    允许从 MariaDB 反射`UUID`列。 这使得 Alembic 能够正确地检测到现有 MariaDB 数据库中此类列的类型。

    参考：[#10028](https://www.sqlalchemy.org/trac/ticket/10028)

### mssql

+   **[mssql] [usecase]**

    添加了对 MSSQL 方言中 COLUMNSTORE 索引的创建和反射支持。 可以在指定`mssql_columnstore=True`的索引上指定。

    参考：[#7340](https://www.sqlalchemy.org/trac/ticket/7340)

+   **[mssql] [bug] [sql]**

    修复了将具有显式排序规则的字符串类型执行`Cast`时会在 CAST 函数内部呈现 COLLATE 子句的问题，这导致语法错误。

    参考：[#9932](https://www.sqlalchemy.org/trac/ticket/9932)

## 2.0.17

发布日期：2023 年 6 月 23 日

### orm

+   **[orm] [bug] [regression]**

    修复了 2.0 系列中的回归问题，其中使用`undefer_group()`与`selectinload()`或`subqueryload()`的查询会引发`AttributeError`。 拉取请求由 Matthew Martin 提供。

    参考：[#9870](https://www.sqlalchemy.org/trac/ticket/9870)

+   **[orm] [bug]**

    修复了在 ORM 注释声明中的问题，该问题阻止了`declared_attr`在未返回`Mapped`数据类型的混合类型上使用，而是返回了诸如`AssociationProxy`之类的补充 ORM 数据类型。 声明式运行时会错误地尝试解释此注释为需要`Mapped`并引发错误。

    参考：[#9957](https://www.sqlalchemy.org/trac/ticket/9957)

+   **[orm] [bug] [typing]**

    修复了使用`declared_attr`函数的`AssociationProxy`返回类型被禁止的类型问题。

    参考：[#9957](https://www.sqlalchemy.org/trac/ticket/9957)

+   **[orm] [bug] [regression]**

    修复了 2.0.16 中由[#9879](https://www.sqlalchemy.org/trac/ticket/9879)引入的回归，其中将可调用对象传递给`mapped_column.default`参数时，同时设置`init=False`会将此值解释为 Dataclass 默认值，该值将直接分配给对象的新实例，绕过了作为底层`Column.default`值生成器的默认生成器所发生的情况。现在检测到此条件以保持先前的行为，但发出了对此模棱两可用法的弃用警告；为了为`Column`填充默认生成器，应使用`mapped_column.insert_default`参数，该参数使其与固定名称的`mapped_column.default`参数相区分，根据 pep-681。

    参考：[#9936](https://www.sqlalchemy.org/trac/ticket/9936)

+   **[orm] [bug]**

    对 ORM `Session` “状态更改”系统进行了额外的强化和文档编写，该系统检测到`Session`和`AsyncSession`对象的并发使用；在从底层引擎获取连接的过程中添加了额外的检查，在内部连接管理方面是一个关键部分。

    参考：[#9973](https://www.sqlalchemy.org/trac/ticket/9973)

+   **[orm] [bug]**

    修复了 ORM 加载器策略逻辑中的问题，进一步允许在复杂的继承多态/别名/`of_type()`关系链中横跨长链的`contains_eager()`加载器选项以正确生效于查询中。

    参考：[#10006](https://www.sqlalchemy.org/trac/ticket/10006)

+   **[orm] [bug]**

    修复了在 `registry.type_annotation_map` 中首次添加的 `Enum` 数据类型支持中的问题，其中使用映射中的自定义 `Enum` 并且在映射中使用固定配置会导致传输 `Enum.name` 参数失败，其中一个问题是如果枚举值作为单独的值传递，则会阻止 PostgreSQL 枚举正常工作。逻辑已更新，使“name”被传输，但也使默认的 `Enum` 不会设置一个硬编码的名称为 `"enum"`。

    参考：[#9963](https://www.sqlalchemy.org/trac/ticket/9963)

### ORM 声明性

+   **[ORM] [声明性] [错误]**

    当将 ORM `relationship()` 和其他 `MapperProperty` 对象同时分配给两个不同的类属性时，会发出警告；只会映射其中一个属性。对于此情况的警告已经适用于 `Column` 和 `mapped_column` 对象。

    参考：[#3532](https://www.sqlalchemy.org/trac/ticket/3532)

### 扩展

+   **[扩展] [错误]**

    修复了与 mypy 1.4 结合使用的 mypy 插件中的问题。

    此更改还被 **回溯** 至：1.4.49

### typing

+   **[类型] [错误]**

    修复了阻止 `WriteOnlyMapped` 和 `DynamicMapped` 属性在 ORM 查询中完全使用的类型问题。

    参考：[#9985](https://www.sqlalchemy.org/trac/ticket/9985)

### postgresql

+   **[postgresql] [用例]**

    pg8000 方言现在支持 RANGE 和 MULTIRANGE 数据类型，使用现有的 RANGE API 描述在 Range and Multirange Types。pg8000 驱动程序从版本 1.29.8 开始支持范围和多范围类型。感谢 Tony Locke 提交的拉取请求。

    参考：[#9965](https://www.sqlalchemy.org/trac/ticket/9965)

### ORM

+   **[ORM] [错误] [回归]**

    修复了 2.0 系列中的回归问题，即使用`undefer_group()`与`selectinload()`或`subqueryload()`的查询会引发`AttributeError`。感谢 Matthew Martin 提交的拉取请求。

    参考：[#9870](https://www.sqlalchemy.org/trac/ticket/9870)

+   **[orm] [bug]**

    修复了 ORM 注释声明中的问题，该问题导致无法在未返回`Mapped`数据类型的混合使用`declared_attr`，而是返回了诸如`AssociationProxy`等补充的 ORM 数据类型。声明式运行时错误地尝试将此注释解释为需要`Mapped`并引发错误。

    参考：[#9957](https://www.sqlalchemy.org/trac/ticket/9957)

+   **[orm] [bug] [typing]**

    修复了使用`declared_attr`函数中的`AssociationProxy`返回类型被禁止的类型问题。

    参考：[#9957](https://www.sqlalchemy.org/trac/ticket/9957)

+   **[orm] [bug] [regression]**

    修复了 2.0.16 中由 [#9879](https://www.sqlalchemy.org/trac/ticket/9879) 引入的回归问题，当将可调用对象传递给 `mapped_column.default` 参数时，同时设置 `init=False` 会将此值解释为 Dataclass 的默认值，该值将直接分配给对象的新实例，绕过了基础 `Column.default` 值生成器在 `Column` 上发生的默认生成器。现在检测到这种情况以保持先前的行为，但是对于这种模棱两可的使用会发出弃用警告；要为 `Column` 填充默认生成器，应使用 `mapped_column.insert_default` 参数，该参数与 `mapped_column.default` 参数相区分，其名称固定为 pep-681。

    参考：[#9936](https://www.sqlalchemy.org/trac/ticket/9936)

+   **[orm] [bug]**

    对于 ORM `Session` 的“状态更改”系统进行了额外的加固和文档编制，该系统检测到同时使用 `Session` 和 `AsyncSession` 对象；在获取来自底层引擎的连接过程中添加了额外的检查，这是关于内部连接管理的关键部分。

    参考：[#9973](https://www.sqlalchemy.org/trac/ticket/9973)

+   **[orm] [bug]**

    修复了 ORM 加载程序策略逻辑中的问题，进一步允许在复杂的继承多态/别名/ of_type() 关系链上跨长链的 `contains_eager()` 加载选项，以在查询中产生正确的效果。

    参考：[#10006](https://www.sqlalchemy.org/trac/ticket/10006)

+   **[orm] [bug]**

    修复了在 `registry.type_annotation_map` 中首次添加的 `Enum` 数据类型支持中出现的问题，该问题涉及使用映射中的固定配置的自定义 `Enum` 时会失败传输 `Enum.name` 参数，其中，如果将枚举值作为单独的值传递，则会导致阻止 PostgreSQL 枚举起作用的问题。逻辑已更新，使“name”被传递，但同时也确保了默认 `Enum` 不会设置硬编码的名称为`"enum"`。

    引用：[#9963](https://www.sqlalchemy.org/trac/ticket/9963)

### ORM 声明

+   **[orm] [声明] [错误]**

    当 ORM `relationship()` 和其他 `MapperProperty` 对象同时分配给两个不同的类属性时，会发出警告；只有其中一个属性会被映射。对于这种情况的警告已经存在于 `Column` 和 `mapped_column` 对象中。

    引用：[#3532](https://www.sqlalchemy.org/trac/ticket/3532)

### 扩展

+   **[扩展] [错误]**

    修复了与 mypy 1.4 一起使用时的 mypy 插件中的问题。

    此更改也被**回溯**到：1.4.49

### 打字

+   **[打字] [错误]**

    修复了一个打字错误问题，该问题导致`WriteOnlyMapped`和`DynamicMapped`属性在 ORM 查询中无法完全使用。

    引用：[#9985](https://www.sqlalchemy.org/trac/ticket/9985)

### postgresql

+   **[postgresql] [用例]**

    pg8000 方言现在支持 RANGE 和 MULTIRANGE 数据类型，使用现有的 RANGE 和 Multirange 类型中描述的 RANGE API。Range 和 multirange 类型在 pg8000 驱动程序的版本 1.29.8 中受支持。感谢 Tony Locke 提供的拉取请求。

    引用：[#9965](https://www.sqlalchemy.org/trac/ticket/9965)

## 2.0.16

发布日期：2023 年 6 月 10 日

### 平台

+   **[平台] [用例]**

    兼容性改进，允许完整测试套件在 Python 3.12.0b1 上通过。

### ORM

+   **[orm] [用例]**

    改进 `DeferredReflection.prepare()`，接受任意的 `**kw` 参数，这些参数将传递给 `MetaData.reflect()`，允许用例，例如反射视图以及传递特定于方言的参数。此外，现代化了 `DeferredReflection.prepare.bind` 参数，使得一个 `Engine` 或 `Connection` 都可作为“bind”参数。

    参考：[#9828](https://www.sqlalchemy.org/trac/ticket/9828)

+   **[orm] [bug]**

    修复了`DeclarativeBaseNoMeta`声明基类无法与非映射的混入或抽象类一起使用的问题，而是引发 `AttributeError`。

    参考：[#9862](https://www.sqlalchemy.org/trac/ticket/9862)

+   **[orm] [bug] [regression]**

    修复了 2.0 系列中的回归问题，其中 `validates.include_backrefs` 的默认值在 `validates()` 函数中更改为 `False`。现在将此默认值恢复为 `True`。

    参考：[#9820](https://www.sqlalchemy.org/trac/ticket/9820)

+   **[orm] [bug]**

    修复了一个新功能中的错误，该功能允许在 ORM 通过主键进行批量更新 时与 WHERE 子句一起使用，该功能是在版本 2.0.11 中作为 [#9583](https://www.sqlalchemy.org/trac/ticket/9583) 的一部分添加的，如果发送的字典不包括每行的主键值，则会通过批量处理过程并为行包含“pk=NULL”，这将导致静默失败。如果未提供批量更新的主键值，则现在会引发异常。

    参考：[#9917](https://www.sqlalchemy.org/trac/ticket/9917)

+   **[orm] [bug] [dataclasses]**

    修复了一个问题，即生成指定了 `default` 值并设置 `init=False` 的数据类字段不起作用。在这种情况下，数据类的行为是在类上设置默认值，这与 SQLAlchemy 使用的描述符不兼容。为了支持这种情况，生成数据类时将默认值转换为 `default_factory`。

    参考：[#9879](https://www.sqlalchemy.org/trac/ticket/9879)

+   **[orm] [bug]**

    每当在 `Mapper` 上添加属性时，如果已经配置了 ORM 映射属性，或者类上已经存在属性，则会发出弃用警告。以前，对于这种情况有一个不一致地发出的非弃用警告。对于这个警告的逻辑已经改进，以便检测到终端用户替换属性，同时不会误报内部声明式和其他情况，其中替换描述符为新描述符是预期的。

    参考：[#9841](https://www.sqlalchemy.org/trac/ticket/9841)

+   **[orm] [bug]**

    改进了 `registry.map_imperatively()` 方法的 `map_imperatively.local_table` 参数上的参数检查，确保仅传递 `Table` 或其他 `FromClause` ，而不是现有的映射类，这会导致未定义的行为，因为对象将进一步解释为新的映射。

    参考：[#9869](https://www.sqlalchemy.org/trac/ticket/9869)

+   **[orm] [bug]**

    `InstanceState.unloaded_expirable` 属性是 `InstanceState.unloaded` 的同义词，现已弃用；此属性始终是特定于实现的，并且不应该是公共的。

    参考：[#9913](https://www.sqlalchemy.org/trac/ticket/9913)

### asyncio

+   **[asyncio] [usecase]**

    在 `create_async_engine()` 中添加了新的 `create_async_engine.async_creator` 参数，其作用与 `create_engine()` 的 `create_engine.creator` 参数相同。这是一个无参数可调用对象，使用 asyncio 数据库驱动程序直接提供新的 asyncio 连接。`create_async_engine()` 函数将在适当的结构中包装驱动程序级连接。拉取请求由杰克·沃瑟斯普恩提供。

    参考：[#8215](https://www.sqlalchemy.org/trac/ticket/8215)

### postgresql

+   **[postgresql] [usecase] [reflection]**

    在 PostgreSQL 反射中，将 `NAME` 列转换为 `TEXT` 当使用 `ARRAY_AGG` 时。这似乎提高了与一些不支持 `NAME` 类型聚合的 PostgreSQL 派生产品的兼容性。

    参考：[#9838](https://www.sqlalchemy.org/trac/ticket/9838)

+   **[postgresql] [usecase]**

    统一了自定义的 PostgreSQL 运算符定义，因为它们在多个不同的数据类型之间共享。

    参考：[#9041](https://www.sqlalchemy.org/trac/ticket/9041)

+   **[postgresql] [usecase]**

    增加了对 PostgreSQL 10 `NULLS NOT DISTINCT` 特性的支持，使用方言选项 `postgresql_nulls_not_distinct`。更新了反射逻辑，以正确考虑此选项。感谢 Pavel Siarchenia 的拉取请求。

    参考：[#8240](https://www.sqlalchemy.org/trac/ticket/8240)

+   **[postgresql] [bug]**

    在 PostgreSQL 特定运算符上使用正确的优先级，比如 `@>`。之前的优先级错误，导致在与 `ANY` 或 `ALL` 结构渲染时括号错误。

    参考：[#9836](https://www.sqlalchemy.org/trac/ticket/9836)

+   **[postgresql] [bug]**

    修复了 `ColumnOperators.like.escape` 和类似参数不允许空字符串作为传递的“转义”字符的问题；这是 PostgreSQL 支持的语法。感谢 Martin Caslavsky 的拉取请求。

    参考：[#9907](https://www.sqlalchemy.org/trac/ticket/9907)

### platform

+   **[platform] [usecase]**

    兼容性改进，使完整的测试套件能够在 Python 3.12.0b1 上通过。

### orm

+   **[orm] [usecase]**

    改进了 `DeferredReflection.prepare()`，接受传递给 `MetaData.reflect()` 的任意 `**kw` 参数，允许用例如反射视图以及传递方言特定参数。此外，现代化了 `DeferredReflection.prepare.bind` 参数，以便接受 `Engine` 或 `Connection` 作为“bind”参数。

    参考：[#9828](https://www.sqlalchemy.org/trac/ticket/9828)

+   **[orm] [bug]**

    修复了 `DeclarativeBaseNoMeta` 声明基类无法与非映射混入类或抽象类一起使用的问题，而是引发 `AttributeError`。

    参考：[#9862](https://www.sqlalchemy.org/trac/ticket/9862)

+   **[orm] [bug] [regression]**

    修复了 2.0 系列中的回归问题，其中 `validates.include_backrefs` 的默认值在 `validates()` 函数中被更改为 `False`。现在将此默认值恢复为 `True`。

    参考：[#9820](https://www.sqlalchemy.org/trac/ticket/9820)

+   **[orm] [bug]**

    修复了新功能中的 bug，该功能允许在 ORM 按主键批量更新 中使用 WHERE 子句，该功能在版本 2.0.11 中作为 [#9583](https://www.sqlalchemy.org/trac/ticket/9583) 的一部分添加，其中发送的字典未包含每行的主键值时，将通过批量处理并为行包括“pk=NULL”，默默失败。如果未提供批量更新的主键值，则现在会引发异常。

    参考：[#9917](https://www.sqlalchemy.org/trac/ticket/9917)

+   **[orm] [bug] [dataclasses]**

    修复了生成指定 `default` 值并设置 `init=False` 的 dataclasses 字段不起作用的问题。在这种情况下，dataclasses 的行为是在类上设置默认值，这与 SQLAlchemy 使用的描述符不兼容。为了支持这种情况，在生成 dataclass 时将默认值转换为 `default_factory`。

    参考：[#9879](https://www.sqlalchemy.org/trac/ticket/9879)

+   **[orm] [bug]**

    当向 `Mapper` 添加属性时，如果已经配置了 ORM 映射属性，或者类上已经存在属性，则会发出弃用警告。以前，对于这种情况，存在一个不一致发出的非弃用警告。现在已经改进了此警告的逻辑，以便在检测到用户替换属性时发出警告，同时不会对内部 Declarative 和其他情况产生误报，其中预期使用新描述符替换旧描述符。

    参考：[#9841](https://www.sqlalchemy.org/trac/ticket/9841)

+   **[orm] [bug]**

    改进了 `registry.map_imperatively()` 方法的 `map_imperatively.local_table` 参数的参数检查，确保只传递了 `Table` 或其他 `FromClause`，而不是已存在的映射类，因为这将导致对象在进一步解释时出现未定义的行为。

    参考：[#9869](https://www.sqlalchemy.org/trac/ticket/9869)

+   **[orm] [错误]**

    `InstanceState.unloaded_expirable` 属性是 `InstanceState.unloaded` 的同义词，并已弃用；此属性始终是特定于实现的，不应公开。 

    参考：[#9913](https://www.sqlalchemy.org/trac/ticket/9913)

### asyncio

+   **[asyncio] [用例]**

    向 `create_async_engine()` 添加了新的 `create_async_engine.async_creator` 参数，它实现了与 `create_engine()` 的 `create_engine.creator` 参数相同的目的。这是一个无参数的可调用对象，提供一个新的 asyncio 连接，直接使用 asyncio 数据库驱动程序。`create_async_engine()` 函数将以适当的结构包装驱动程序级别的连接。感谢 Jack Wotherspoon 提交的拉取请求。

    参考：[#8215](https://www.sqlalchemy.org/trac/ticket/8215)

### postgresql

+   **[postgresql] [用例] [反射]**

    在使用 PostgreSQL 反射时，将 `NAME` 列转换为 `TEXT`。这似乎提高了与一些 PostgreSQL 派生产品的兼容性，这些产品可能不支持在 `NAME` 类型上进行聚合。

    参考：[#9838](https://www.sqlalchemy.org/trac/ticket/9838)

+   **[postgresql] [用例]**

    统一了自定义 PostgreSQL 操作符的定义，因为它们在多个不同的数据类型之间共享。

    参考：[#9041](https://www.sqlalchemy.org/trac/ticket/9041)

+   **[postgresql] [用例]**

    增加了对 PostgreSQL 10 `NULLS NOT DISTINCT` 特性的支持，该特性使用方言选项 `postgresql_nulls_not_distinct`。 更新了反射逻辑以正确考虑此选项。 Pavel Siarchenia 的 Pull request 提供支持。

    参考：[#8240](https://www.sqlalchemy.org/trac/ticket/8240)

+   **[postgresql] [bug]**

    在 PostgreSQL 特定运算符上使用正确的优先级，例如 `@>`。 之前优先级错误，导致在与 `ANY` 或 `ALL` 结构进行渲染时出现错误的括号。

    参考：[#9836](https://www.sqlalchemy.org/trac/ticket/9836)

+   **[postgresql] [bug]**

    修复了一个问题，`ColumnOperators.like.escape`和类似参数不允许空字符串作为参数，该参数将作为“转义”字符传递；这是 PostgreSQL 支持的语法。 感谢 Martin Caslavsky 的 Pull requset。

    参考：[#9907](https://www.sqlalchemy.org/trac/ticket/9907)

## 2.0.15

发布日期：2023 年 5 月 19 日

### orm

+   **[orm] [bug]**

    随着越来越多的项目开始使用新型“2.0” ORM 查询，显然“自动刷新”的条件性质，基于给定语句是否涉及 ORM 实体，正在变得更加关键。直到现在，“ORM”标志对于语句是否返回与 ORM 实体或列对应的行一直是松散的；“ORM”标志的原始目的是启用 ORM 实体获取规则，该规则将后处理应用于核心结果集以及将 ORM 加载器策略应用于语句。 对于不建立在包含 ORM 实体的行上的语句，认为“ORM”标志基本上是不必要的。

    仍然可能是“自动刷新”对于 *所有* 使用 `Session.execute()` 和相关方法更好，即使对于纯粹的 Core SQL 构造也是如此。 但是，这可能仍然会影响不期望的遗留情况，并且可能更多地成为 2.1 版本的事情。 然而，目前“ORM 标志”的规则已经放宽，因此任何语句，包括在 WHERE / ORDER BY / GROUP BY 子句中的 ORM 实体或属性，在标量子查询中等都将启用此标志。 这将导致对这些语句进行“自动刷新”，并且还可以通过 `ORMExecuteState.is_orm_statement` 事件级属性可见。

    参考：[#9805](https://www.sqlalchemy.org/trac/ticket/9805)

### postgresql

+   **[postgresql] [bug] [regression]**

    修复了 PostgreSQL 方言的基本`Uuid`数据类型，以充分利用 PG 特定的 `UUID` 方言特定数据类型，当选择“native_uuid”时，以便包含 PG 驱动程序的行为。由于作为[#9618](https://www.sqlalchemy.org/trac/ticket/9618)的一部分进行的 insertmanyvalues 改进，这个问题变得明显，类似于[#9739](https://www.sqlalchemy.org/trac/ticket/9739)，asyncpg 驱动程序对数据类型转换的存在与否非常敏感，当使用这种通用类型时，必须调用 PostgreSQL 驱动程序特定的本地 `UUID` 数据类型，以便进行这些转换。

    参考：[#9808](https://www.sqlalchemy.org/trac/ticket/9808)

### orm

+   **[orm] [bug]**

    随着越来越多的项目使用新式“2.0” ORM 查询，显而易见的是，“autoflush”的条件性，基于给定语句是否涉及 ORM 实体，正在变得更加关键。直到现在，对于语句是否返回与 ORM 实体或列对应的行，一直是围绕“ORM”标志的松散基础；“ORM”标志的最初目的是启用 ORM 实体获取规则，这些规则将后处理应用于 Core 结果集以及 ORM 加载器策略到语句。对于不建立在包含 ORM 实体的行上的语句，认为“ORM”标志基本上是不必要的。

    仍然可能是“autoflush”对于*所有*使用`Session.execute()`和相关方法更好，即使是纯粹的 Core SQL 构造。然而，这可能会影响到不期望的遗留情况，可能更多是 2.1 的事情。但是，目前，“ORM-flag”的规则已经放宽，以便包含 ORM 实体或属性的语句中的任何地方，包括仅在 WHERE / ORDER BY / GROUP BY 子句中，在标量子查询中等，将启用此标志。这将导致这些语句发生“autoflush”，并且还可以通过`ORMExecuteState.is_orm_statement`事件级属性可见。

    参考：[#9805](https://www.sqlalchemy.org/trac/ticket/9805)

### postgresql

+   **[postgresql] [bug] [regression]**

    修复了基本的`Uuid`数据类型，使其在选择“native_uuid”时充分利用了 PostgreSQL 方言的 PG 特定 `UUID` 方言特定数据类型，以便包含 PG 驱动程序的行为。这个问题由于作为[#9618](https://www.sqlalchemy.org/trac/ticket/9618)的一部分所做的 insertmanyvalues 改进而变得显而易见，其中与[#9739](https://www.sqlalchemy.org/trac/ticket/9739)类似，asyncpg 驱动程序对是否存在数据类型转换非常敏感，当使用此通用类型时，必须调用 PostgreSQL 驱动程序特定的本地 `UUID` 数据类型，以便进行这些转换。

    参考：[#9808](https://www.sqlalchemy.org/trac/ticket/9808)

## 2.0.14

发布日期：2023 年 5 月 18 日

### orm

+   **[orm] [bug]**

    修改了`JoinedLoader`的实现，在一个特定的区域使用了更简单的方法，之前它使用了一个缓存结构，该结构将在线程之间共享。这样做的原因是避免潜在的竞争条件，这被怀疑是导致特定崩溃的原因。所涉及的缓存结构最终仍然通过编译后的 SQL 缓存“缓存”，因此不预期出现性能下降。

    参考：[#9777](https://www.sqlalchemy.org/trac/ticket/9777)

+   **[orm] [bug] [regression]**

    修复了在 `CTE` 构造中使用 `update()` 或 `delete()`，然后在 `select()` 中使用会引发 `CompileError` 的回归，原因是 ORM 相关规则执行 ORM 级别的更新/删除语句。

    参考：[#9767](https://www.sqlalchemy.org/trac/ticket/9767)

+   **[orm] [bug]**

    在新的 ORM Annotated Declarative 中修复了使用 `ForeignKey`（或其他列级约束）位于 `mapped_column()` 中，然后通过 pep-593 `Annotated` 复制到模型中时，将每个约束的副本应用到目标 `Table` 中的 `Column`，从而导致不正确的 CREATE TABLE DDL 以及 Alembic 下的迁移指令。

    参考：[#9766](https://www.sqlalchemy.org/trac/ticket/9766)

+   **[orm] [bug]**

    修复了使用`joinedload()`加载器选项时，使用附加的关系条件的问题，其中附加的条件本身包含引用加入实体的相关子查询，因此还需要对别名实体进行“调整”，否则会排除这种调整，导致 joinedload 的 ON 子句错误。

    References: [#9779](https://www.sqlalchemy.org/trac/ticket/9779)

### sql

+   **[sql] [usecase]**

    将 MSSQL 的`try_cast()`函数泛化为`sqlalchemy.`导入命名空间，以便第三方方言也可以实现它。在 SQLAlchemy 中，`try_cast()`函数仍然是一个仅支持 SQL Server 的构造，如果在不支持它的后端使用它，则会引发`CompileError`。

    `try_cast()`实现了一个 CAST，其中无法转换的转换将返回为 NULL，而不是引发错误。从理论上讲，该结构可以由第三方方言实现，如 Google BigQuery、DuckDB 和 Snowflake，可能还有其他方言。

    Pull request courtesy Nick Crews.

    References: [#9752](https://www.sqlalchemy.org/trac/ticket/9752)

+   **[sql] [bug]**

    修复了在`values()`结构中使用时会出现内部编译错误的问题，如果该结构在标量子查询内使用，则会发生错误。

    References: [#9772](https://www.sqlalchemy.org/trac/ticket/9772)

### postgresql

+   **[postgresql] [bug]**

    修复了一个明显非常古老的问题，即当`ENUM.create_type`参数设置为其非默认值`False`时，它将不会在复制`Column`时传播，这在使用 ORM Declarative mixins 时很常见。

    References: [#9773](https://www.sqlalchemy.org/trac/ticket/9773)

### tests

+   **[tests] [bug] [pypy]**

    修复了依赖于`sys.getsizeof()`函数在 pypy 上不运行的测试问题，在 pypy 上，这个函数似乎与 cpython 上的行为不同。

    References: [#9789](https://www.sqlalchemy.org/trac/ticket/9789)

### orm

+   **[orm] [bug]**

    修改了 `JoinedLoader` 的实现，以在先前使用缓存结构的一个特定区域中使用更简单的方法，该缓存结构会在线程之间共享。其理由是避免潜在的竞争条件，这被怀疑是引起多次报告的特定崩溃的原因。所讨论的缓存结构最终仍通过编译的 SQL 缓存“缓存”，因此不预期性能下降。

    参考：[#9777](https://www.sqlalchemy.org/trac/ticket/9777)

+   **[orm] [bug] [regression]**

    修复了在使用 `CTE` 构造中使用 `update()` 或 `delete()` ，然后在 `select()` 中使用时，由于执行 ORM 级别的更新/删除语句的 ORM 相关规则而引发 `CompileError` 的回归。

    参考：[#9767](https://www.sqlalchemy.org/trac/ticket/9767)

+   **[orm] [bug]**

    修复了在新的 ORM Annotated Declarative 中使用 `ForeignKey`（或其他列级约束）在通过 pep-593 `Annotated` 复制到模型时，在 `mapped_column()` 内部应用每个约束的副本到由目标 `Table` 生成的 `Column` 上的问题，导致不正确的 CREATE TABLE DDL 以及 Alembic 下的迁移指令重复。

    参考：[#9766](https://www.sqlalchemy.org/trac/ticket/9766)

+   **[orm] [bug]**

    修复了在使用额外的关系标准与 `joinedload()` 加载器选项时的问题，其中额外的标准本身包含与联接实体相关的相关子查询，因此还需要对别名实体进行“适应”，但这些会被排除在此适应之外，从而为 joinedload 生成错误的 ON 子句。

    参考：[#9779](https://www.sqlalchemy.org/trac/ticket/9779)

### sql

+   **[sql] [usecase]**

    将 MSSQL 的`try_cast()`函数泛化到`sqlalchemy.`导入命名空间中，以便它也可以由第三方方言实现。在 SQLAlchemy 内部，`try_cast()`函数仍然是一个仅适用于 SQL Server 的构造，在不支持它的后端使用时将引发`CompileError`。

    `try_cast()`实现了一个 CAST，其中无法转换的转换将被返回为 NULL，而不是引发错误。理论上，该构造可以由 Google BigQuery、DuckDB 和 Snowflake 等第三方方言实现，并可能还有其他方言。

    感谢 Nick Crews 提供的拉取请求。

    参考：[#9752](https://www.sqlalchemy.org/trac/ticket/9752)

+   **[sql] [bug]**

    修复了在`values()`构造中出现内部编译错误的问题，如果该构造在标量子查询内部使用将会发生该错误。

    参考：[#9772](https://www.sqlalchemy.org/trac/ticket/9772)

### postgresql

+   **[postgresql] [bug]**

    修复了一个明显很旧的问题，即当`ENUM.create_type`参数被设置为非默认值`False`时，在复制其所属的`Column`时，该参数不会被传播，这在使用 ORM Declarative mixins 时很常见。

    参考：[#9773](https://www.sqlalchemy.org/trac/ticket/9773)

### tests

+   **[tests] [bug] [pypy]**

    修复了依赖于`sys.getsizeof()`函数在 pypy 上不运行的测试问题，pypy 上该函数的行为似乎与 cpython 上有所不同。

    参考：[#9789](https://www.sqlalchemy.org/trac/ticket/9789)

## 2.0.13

发布日期：2023 年 5 月 10 日

### orm

+   **[orm] [bug]**

    修复了 ORM Annotated Declarative 在所有情况下都无法正确解析正向引用的问题；特别是在与 Pydantic 数据类结合使用`from __future__ import annotations`时。

    参考：[#9717](https://www.sqlalchemy.org/trac/ticket/9717)

+   **[orm] [bug]**

    在新的使用 RETURNING 与 upsert 语句功能中修复了一个问题，即`populate_existing`执行选项未传播到加载选项，导致现有属性无法在原地刷新。

    参考：[#9746](https://www.sqlalchemy.org/trac/ticket/9746)

+   **[orm] [bug]**

    修复了加载器策略路径问题，其中像`joinedload()` / `selectinload()`这样的急加载器在遍历多级深度时会因为跟随具有`with_polymorphic()`或类似结构的中间成员而无法完全遍历。

    参考：[#9715](https://www.sqlalchemy.org/trac/ticket/9715)

+   **[ORM] [错误]**

    修复了`mapped_column()`构造中的问题，当 ORM 映射属性引用相同的`Column`时，如果涉及`mapped_column()`构造，则不会发出正确的“直接多次命名列 X”的警告，而是会引发内部断言。

    参考：[#9630](https://www.sqlalchemy.org/trac/ticket/9630)

### SQL

+   **[SQL] [用例]**

    对于包含多个未以某种方式相关联的表的 UPDATE 和 DELETE 语句实施了“笛卡尔积警告”。

    参考：[#9721](https://www.sqlalchemy.org/trac/ticket/9721)

+   **[SQL] [错误]**

    修复了特定于方言的浮点/双精度类型的基类；Oracle 的`BINARY_DOUBLE`现在是`Double`的子类，而用于 asyncpg 和 pg8000 的内部类型现在正确地是`Float`的子类。

+   **[SQL] [错误]**

    修复了包含多个表且没有 VALUES 子句的`update()`构造会导致内部错误的问题。当前对于没有值的`Update`的行为是生成一个带有空“set”子句的 SQL UPDATE 语句，因此对于这种特定子情况已经做出了一致的处理。

### 模式

+   **[模式] [性能]**

    改进了添加表列的方式，避免不必要的分配，显著加快了创建许多表的过程，比如在反射整个模式时。

    参考：[#9597](https://www.sqlalchemy.org/trac/ticket/9597)

### 类型

+   **[类型] [错误]**

    修复了`Session.get()` 和 `Session.refresh()`（以及`AsyncSession`上对应的方法）中的`Session.get.with_for_update`参数的类型问题，现在在运行时可以接受布尔值`True`以及参数在运行时可以接受的其他所有形式。

    参考：[#9762](https://www.sqlalchemy.org/trac/ticket/9762)

+   **[typing] [sql]**

    添加了类型`ColumnExpressionArgument`作为一个公共类型，指示传递给 SQLAlchemy 构造的基于列的参数，例如`Select.where()`、`and_()`等。这可以用于为调用这些方法的最终用户函数添加类型。

    参考：[#9656](https://www.sqlalchemy.org/trac/ticket/9656)

### asyncio

+   **[asyncio] [usecase]**

    添加了一个新的助手混合类`AsyncAttrs`，旨在改善在 asyncio 中使用惰性加载器和其他过期或延迟的 ORM 属性的情况，提供了一个简单的属性访问器，为任何 ORM 属性提供了一个`await`接口，无论它是否需要发出 SQL。

    另见

    `AsyncAttrs`

    参考：[#9731](https://www.sqlalchemy.org/trac/ticket/9731)

+   **[asyncio] [bug]**

    修复了半私有的`await_only()`和`await_fallback()`并发函数中的问题，在函数抛出`GreenletError`后，给定的可等待对象仍然未被等待，这可能导致如果程序继续运行，稍后会出现“未被等待”的警告。在这种情况下，在抛出异常之前，给定的可等待对象现在将被取消。

### postgresql

+   **[postgresql] [bug] [regression]**

    修复了另一个因为 2.0.10 版本中“insertmanyvalues”更改而引起的回归问题，作为[#9618](https://www.sqlalchemy.org/trac/ticket/9618)的一部分，以类似于回归问题[#9701](https://www.sqlalchemy.org/trac/ticket/9701)的方式，使用 asyncpg 驱动程序时，`LargeBinary`数据类型在使用新的批量插入格式时也需要额外的转换。

    参考：[#9739](https://www.sqlalchemy.org/trac/ticket/9739)

### oracle

+   **[oracle] [reflection]**

    在 Oracle 方言中，对基于表达式的索引和索引表达式的排序方向增加了反射支持。

    参考：[#9597](https://www.sqlalchemy.org/trac/ticket/9597)

### 杂项

+   **[bug] [ext]**

    修复了 `Mutable` 中的问题，其中 ORM 映射的属性的事件注册将对映射的继承子类重复调用，导致在继承层次结构中调用重复事件。

    参考：[#9676](https://www.sqlalchemy.org/trac/ticket/9676)

### orm

+   **[orm] [bug]**

    修复了 ORM 注释声明在某些情况下无法正确解析前向引用的问题；特别是，在使用 `from __future__ import annotations` 与 Pydantic 数据类结合使用时。

    参考：[#9717](https://www.sqlalchemy.org/trac/ticket/9717)

+   **[orm] [bug]**

    修复了新的 使用 RETURNING 语句进行 upsert 功能中的问题，其中 `populate_existing` 执行选项未传播到加载选项，导致现有属性无法在原位刷新。

    参考：[#9746](https://www.sqlalchemy.org/trac/ticket/9746)

+   **[orm] [bug]**

    修复了加载器策略路径问题，例如，当一个加载器，如 `joinedload()` / `selectinload()` 遇到一个 `with_polymorphic()` 或类似结构时，会导致许多级别的深度跟踪失败。

    参考：[#9715](https://www.sqlalchemy.org/trac/ticket/9715)

+   **[orm] [bug]**

    修复了 `mapped_column()` 构造中的问题，当 ORM 映射的属性引用相同的 `Column` 时，如果 `mapped_column()` 构造涉及，则不会发出“直接多次命名列 X”的正确警告，而是引发内部断言。

    参考：[#9630](https://www.sqlalchemy.org/trac/ticket/9630)

### sql

+   **[sql] [usecase]**

    对于包含多个未相关的表的 UPDATE 和 DELETE 语句，实施了“笛卡尔积警告”。

    参考：[#9721](https://www.sqlalchemy.org/trac/ticket/9721)

+   **[sql] [bug]**

    修复了方言特定的浮点/双精度类型的基类；Oracle `BINARY_DOUBLE` 现在是 `Double` 的子类，而且对于 asyncpg 和 pg8000 的内部类型 `Float` 现在正确地是 `Float` 的子类。

+   **[sql] [bug]**

    修复了包含多个表且没有 VALUES 子句的 `update()` 构造会导致内部错误的问题。不包含值的 `Update` 的当前行为是生成一个带有空“set”子句的 SQL UPDATE 语句，因此这一特定子情况的行为已经统一。

### schema

+   **[schema] [performance]**

    改进了添加表列的方式，避免了不必要的分配，显著加快了创建许多表，例如反射整个模式时的速度。

    参考：[#9597](https://www.sqlalchemy.org/trac/ticket/9597)

### typing

+   **[typing] [bug]**

    修复了 `Session.get.with_for_update` 参数的类型注解，该参数在运行时接受布尔值 `True` 和所有其他参数形式，在 `Session.get()` 和 `Session.refresh()` 方法（以及 `AsyncSession` 上对应的方法）中也适用。

    参考：[#9762](https://www.sqlalchemy.org/trac/ticket/9762)

+   **[typing] [sql]**

    添加了类型 `ColumnExpressionArgument` 作为一个公共类型，指示传递给 SQLAlchemy 构造的面向列的参数，例如 `Select.where()`、`and_()` 等。这可以用于为调用这些方法的最终用户函数添加类型注解。

    参考：[#9656](https://www.sqlalchemy.org/trac/ticket/9656)

### asyncio

+   **[asyncio] [usecase]**

    添加了一个新的辅助混合类`AsyncAttrs`，旨在改进与 asyncio 一起使用的懒加载器和其他已过期或延迟的 ORM 属性的使用，提供了一个简单的属性访问器，为任何 ORM 属性提供了一个`await`接口，无论它是否需要发出 SQL。

    另请参见

    `AsyncAttrs`

    参考：[#9731](https://www.sqlalchemy.org/trac/ticket/9731)

+   **[asyncio] [bug]**

    修复了半私有的`await_only()`和`await_fallback()`并发函数中的问题，其中给定的可等待对象如果函数抛出`GreenletError`，则会保持未等待状态，这可能会导致后续程序继续运行时出现“未等待”的警告。在这种情况下，在抛出异常之前现在会取消给定的可等待对象。

### postgresql

+   **[postgresql] [bug] [regression]**

    修复了另一个由于 2.0.10 中的“insertmanyvalues”更改而导致的回归，类似于回归[#9701](https://www.sqlalchemy.org/trac/ticket/9701)，其中`LargeBinary`数据类型在使用 asyncpg 驱动程序时也需要对新的批量 INSERT 格式进行额外的强制转换才能正常工作。

    参考：[#9739](https://www.sqlalchemy.org/trac/ticket/9739)

### oracle

+   **[oracle] [reflection]**

    在 Oracle 方言中增加了对基于表达式的索引和索引表达式的排序方向的反射支持。

    参考：[#9597](https://www.sqlalchemy.org/trac/ticket/9597)

### 杂项

+   **[bug] [ext]**

    修复了`Mutable`中的问题，其中为 ORM 映射属性注册的事件将被重复调用，导致在继承层次结构中调用重复事件。

    参考：[#9676](https://www.sqlalchemy.org/trac/ticket/9676)

## 2.0.12

发布日期：2023 年 4 月 30 日

### orm

+   **[orm] [bug]**

    修复了一个严重的缓存问题，即`aliased()`和`hybrid_property()`表达式组合的组合会导致缓存键不匹配，从而导致缓存键保存了实际的`aliased()`对象，同时又不匹配等效构造的缓存键，填满了缓存。

    此更改也已**回溯**到：1.4.48

    参考：[#9728](https://www.sqlalchemy.org/trac/ticket/9728)

### mysql

+   **[mysql] [bug] [mariadb]**

    修复了关于`Table`和`Column`对象反射注释的问题，其中注释包含控制字符，如换行符。还为这些字符以及扩展的 Unicode 字符在表和列注释中（后者不受 MySQL/MariaDB 支持）添加了额外的测试支持，以提高整体测试。

    参考：[#9722](https://www.sqlalchemy.org/trac/ticket/9722)

### orm

+   **[orm] [bug]**

    修复了关键的缓存问题，其中`aliased()`和`hybrid_property()`表达式组合的组合会导致缓存键不匹配，导致缓存键保留实际的`aliased()`对象，同时也不匹配等效构造的缓存键，填满缓存。

    这个更改也被**回溯**到：1.4.48

    参考：[#9728](https://www.sqlalchemy.org/trac/ticket/9728)

### mysql

+   **[mysql] [bug] [mariadb]**

    修复了关于`Table`和`Column`对象反射注释的问题，其中注释包含控制字符，如换行符。还为这些字符以及扩展的 Unicode 字符在表和列注释中（后者不受 MySQL/MariaDB 支持）添加了额外的测试支持，以提高整体测试。

    参考：[#9722](https://www.sqlalchemy.org/trac/ticket/9722)

## 2.0.11

发布日期：2023 年 4 月 26 日

### orm

+   **[orm] [usecase]**

    ORM 批量 INSERT 和 UPDATE 功能现在增加了以下功能：

    +   在使用 ORM INSERT 时，不再要求使用“orm” dml_strategy 设置时不传递额外的参数。

    +   在使用 ORM UPDATE 时，不再要求使用“bulk” dml_strategy 设置时不传递额外的 WHERE 条件。请注意，在这种情况下，预期行数的检查被关闭。

    参考：[#9583](https://www.sqlalchemy.org/trac/ticket/9583), [#9595](https://www.sqlalchemy.org/trac/ticket/9595)

+   **[orm] [bug]**

    修复了 2.0 版本中在使用 ORM `Session`执行`Insert`语句时，`Insert.values()`内部使用`bindparam()`无法正确解释的回归，这是由于新的启用 ORM 插入功能未实现此用例。

    参考：[#9583](https://www.sqlalchemy.org/trac/ticket/9583), [#9595](https://www.sqlalchemy.org/trac/ticket/9595)

### 引擎

+   **[引擎] [性能]**

    一系列针对`Row`的性能增强：

    +   改进了行的“命名元组”接口的`__getattr__`性能；在此更改中，`Row`的实现已经简化，删除了特定于 1.4 版本及之前版本的 SQLAlchemy 的构造和逻辑。作为此更改的一部分，`Row`的序列化格式略有修改，但是使用之前 SQLAlchemy 2.0 版本进行 pickle 的行将在新格式中被识别。感谢 J. Nick Koston 的拉取请求。

    +   通过使“bytes”处理程序在每个驱动程序基础上有条件地执行，改进了“二进制”数据类型的行处理性能。因此，除了 psycopg2 之外的几乎所有驱动程序都已删除了“bytes”结果处理程序，所有这些驱动程序在现代形式下都支持直接返回 Python“bytes”。感谢 J. Nick Koston 的拉取请求。

    +   由 Federico Caselli 对`Row`进行的额外重构以提高性能。

    参考：[#9678](https://www.sqlalchemy.org/trac/ticket/9678), [#9680](https://www.sqlalchemy.org/trac/ticket/9680)

+   **[引擎] [错误] [回归]**

    修复了阻止`URL.normalized_query`属性在`URL`中正常运行的回归。

    参考：[#9682](https://www.sqlalchemy.org/trac/ticket/9682)

### SQL

+   **[SQL] [用例]**

    为 `ColumnCollection` 添加了切片访问的支持，例如 `table.c[0:5]`、`subquery.c[:-1]` 等。切片访问返回一个子 `ColumnCollection`，与传递键元组的方式相同。这是对 [#8285](https://www.sqlalchemy.org/trac/ticket/8285) 添加的键元组访问的自然延续，其中切片访问用例被遗漏似乎是一个疏忽。

    参考：[#8285](https://www.sqlalchemy.org/trac/ticket/8285)

### typing

+   **[typing] [错误]**

    改进了 `RowMapping` 的类型提示，指示它也支持 `Column` 作为索引对象，而不仅仅是字符串名称。感谢 Andy Freeland 提交的拉取请求。

    参考：[#9644](https://www.sqlalchemy.org/trac/ticket/9644)

### postgresql

+   **[postgresql] [错误] [回归]**

    修复了由 [#9618](https://www.sqlalchemy.org/trac/ticket/9618) 引起的严重回归，该回归修改了 2.0.10 版本的 insertmanyvalues 功能的架构，导致在使用 insertmanyvalues 功能插入时，使用 psycopg2 或 psycopg 驱动程序时，浮点值失去了所有小数位。

    参考：[#9701](https://www.sqlalchemy.org/trac/ticket/9701)

### mssql

+   **[mssql] [错误]**

    在 SQL Server 中实现了 `Double` 类型，它将在 DDL 时渲染 `DOUBLE PRECISION`。这是使用一个新的 MSSQL 数据类型 `DOUBLE_PRECISION` 实现的，也可以直接使用。

### oracle

+   **[oracle] [错误]**

    在 Oracle 方言中修复了一个问题，即当这些列在 `Insert.returning()` 子句中用于返回 INSERT 的值时，`Decimal` 返回类型（如 `Numeric`）将返回浮点值，而不是 `Decimal` 对象。

### orm

+   **[orm] [用例]**

    ORM bulk INSERT and UPDATE 功能现在增加了以下功能：

    +   在使用“orm” dml_strategy 设置进行 ORM INSERT 时，解除了不传递额外参数的要求。

    +   在使用“bulk” dml_strategy 设置进行 ORM UPDATE 时，解除了不传递额外 WHERE 条件的要求。请注意，在这种情况下，关闭了对预期行数的检查。

    参考：[#9583](https://www.sqlalchemy.org/trac/ticket/9583)，[#9595](https://www.sqlalchemy.org/trac/ticket/9595)

+   **[orm] [bug]**

    修复了 2.0 版本中的回归，当在使用 ORM `Session` 执行 `Insert` 语句时，在 `Insert.values()` 中使用 `bindparam()` 会无法被正确解释的问题，原因是新的 ORM-enabled insert feature 没有实现这种用例。

    参考：[#9583](https://www.sqlalchemy.org/trac/ticket/9583)，[#9595](https://www.sqlalchemy.org/trac/ticket/9595)

### engine

+   **[engine] [performance]**

    一系列针对 `Row` 的性能增强：

    +   `__getattr__` 函数在行的“命名元组”接口的性能得到了提升；在这个改变中，`Row` 的实现已经被简化，移除了特定于 SQLAlchemy 1.4 及之前系列的构造和逻辑。作为这个改变的一部分，`Row` 的序列化格式已经稍微修改，然而之前的 SQLAlchemy 2.0 版本中使用 pickle 序列化的行将在新的格式中被识别。拉取请求由 J. Nick Koston 提供。

    +   通过使“bytes”处理器在每个驱动程序基础上有条件地生效，提高了对“binary”数据类型的行处理性能。因此，除了 psycopg2 外，几乎所有现代形式的驱动程序都已删除了“bytes”结果处理器，所有这些驱动程序都支持直接返回 Python “bytes”。拉取请求由 J. Nick Koston 提供。

    +   进一步重构 `Row` 以提高性能，由 Federico Caselli 提供。

    参考：[#9678](https://www.sqlalchemy.org/trac/ticket/9678)，[#9680](https://www.sqlalchemy.org/trac/ticket/9680)

+   **[engine] [bug] [regression]**

    修复了一个阻止 `URL` 的 `URL.normalized_query` 属性正常运行的回归。

    参考：[#9682](https://www.sqlalchemy.org/trac/ticket/9682)

### sql

+   **[sql] [usecase]**

    为 `ColumnCollection` 添加了切片访问支持，例如 `table.c[0:5]`，`subquery.c[:-1]` 等。 切片访问返回一个子 `ColumnCollection`，方式与传递键元组相同。 这是对 [#8285](https://www.sqlalchemy.org/trac/ticket/8285) 添加的键元组访问的自然延伸，在该案例中，切片访问用例被遗漏似乎是一个疏忽。

    参考：[#8285](https://www.sqlalchemy.org/trac/ticket/8285)

### typing

+   **[typing] [bug]**

    改进了 `RowMapping` 的类型，指示它也支持 `Column` 作为索引对象，而不仅仅是字符串名称。 感谢 Andy Freeland 提供的拉取请求。

    参考：[#9644](https://www.sqlalchemy.org/trac/ticket/9644)

### postgresql

+   **[postgresql] [bug] [regression]**

    修复了由于 [#9618](https://www.sqlalchemy.org/trac/ticket/9618) 引起的严重回归，该回归修改了 2.0.10 版本中的 insertmanyvalues 功能的架构，导致使用 psycopg2 或 psycopg 驱动程序的 insertmanyvalues 功能插入浮点值时丢失所有小数位。

    参考：[#9701](https://www.sqlalchemy.org/trac/ticket/9701)

### mssql

+   **[mssql] [bug]**

    为 SQL Server 实现了 `Double` 类型，在 DDL 时间将呈现 `DOUBLE PRECISION`。 这是使用新的 MSSQL 数据类型 `DOUBLE_PRECISION` 实现的，也可以直接使用。

### oracle

+   **[oracle] [bug]**

    修复了 Oracle 方言中的问题，当这些列在 `Insert.returning()` 子句中返回 INSERTed 值时，`Decimal` 返回类型（如 `Numeric`）将返回浮点值，而不是 `Decimal` 对象。

## 2.0.10

发布日期：2023 年 4 月 21 日

### orm

+   **[orm] [bug]**

    修复了各种 ORM 特定的获取器（如`ORMExecuteState.is_column_load`，`ORMExecuteState.is_relationship_load`，`ORMExecuteState.loader_strategy_path`等）在 SQL 语句本身是“复合选择”（如 UNION）时会抛出`AttributeError`的错误。

    此更改也**回溯**到：1.4.48

    参考：[#9634](https://www.sqlalchemy.org/trac/ticket/9634)

+   **[orm] [bug]**

    修复了当应用于`__mapper_args__`特殊方法名时，`declared_attr.directive()`修饰符未正确遵守子类的问题，而不是直接使用`declared_attr`。这两个构造应具有相同的运行时行为。

    参考：[#9625](https://www.sqlalchemy.org/trac/ticket/9625)

+   **[orm] [bug]**

    对`with_loader_criteria()`加载器选项进行了改进，允许在不是 ORM 语句本身的顶层语句的`Executable.options()`方法中指示它。示例包括嵌入在复合语句中的`select()`，在`Insert.from_select()`构造中，以及在顶层不与 ORM 相关的 CTE 表达式中。

    参考：[#9635](https://www.sqlalchemy.org/trac/ticket/9635)

+   **[orm] [bug]**

    修复了 ORM 批量插入功能中的错误，如果请求返回单独的列，则在 INSERT 语句中会渲染额外的不必要列。

    参考：[#9685](https://www.sqlalchemy.org/trac/ticket/9685)

+   **[orm] [bug]**

    修复了 ORM 声明式数据类中的错误，其中`query_expression()`和`column_property()`构造被文档化为在声明式映射上下文中的只读构造，不能在没有添加`init=False`的情况下与`MappedAsDataclass`类一起使用，因为在`query_expression()`的情况下未包含`init`参数，这是不可能的。 这些构造已从数据类的角度进行了修改，假定为“只读”，默认设置`init=False`，并且不再将它们包括在 pep-681 构造函数中。 `column_property()`的数据类参数`init`，`default`，`default_factory`，`kw_only`现已过时； 这些字段不适用于在构造为只读的声明性数据类配置中使用的`column_property()`。 还为`query_expression()`添加了仅读特定参数`query_expression.compare`； `query_expression.repr`已经存在。

    参考：[#9628](https://www.sqlalchemy.org/trac/ticket/9628)

+   **[orm] [bug]**

    向`mapped_column()`构造中添加了丢失的`mapped_column.active_history`参数。

### 引擎

+   **[engine] [usecase]**

    添加了`create_pool_from_url()`和`create_async_pool_from_url()`以从作为字符串或`URL`传递的输入 url 创建`Pool`实例。

    参考：[#9613](https://www.sqlalchemy.org/trac/ticket/9613)

+   **[engine] [bug]**

    修复了在 2.0 系列中首次引入的 INSERT 语句的“Insert Many Values”行为性能优化功能中发现的一个主要缺陷。这是对 2.0.9 中的更改的延续，该更改禁用了 SQL Server 版本的功能，因为 ORM 依赖于明显的行排序，而这种排序不能保证发生。修复将新逻辑应用于所有“insertmanyvalues”操作，当在`Insert.returning()`或`UpdateBase.return_defaults()`方法上设置了新参数`Insert.returning.sort_by_parameter_order`时生效，通过交替的 SQL 形式、客户端参数的直接对应以及在某些情况下降级到逐行运行，将对每个返回行批次应用排序，使用与每行中的主键或其他唯一值对应的值，这些值可以与输入数据相关联。

    预计性能影响将是最小的，因为几乎所有常见的主键场景都适合于为所有后端实现参数排序的批处理，而“逐行”模式与 1.x 系列中使用的非常沉重的方法相比，具有最少的 Python 开销。对于 SQLite，当使用“逐行”模式时，性能没有差异。

    预计通过高效的“逐行”插入与返回批处理功能，“insertmanyvalues”功能可以更容易地推广到第三方后端，这些后端包括返回支持但不一定易于保证与参数顺序对应的方式。

    另请参阅

    将返回的行与参数集相关联

    参考：[#9603](https://www.sqlalchemy.org/trac/ticket/9603), [#9618](https://www.sqlalchemy.org/trac/ticket/9618)

### typing

+   **[typing] [bug]**

    为最近添加的运算符`ColumnOperators.icontains()`、`ColumnOperators.istartswith()`、`ColumnOperators.iendswith()`以及位运算符`ColumnOperators.bitwise_and()`、`ColumnOperators.bitwise_or()`、`ColumnOperators.bitwise_xor()`、`ColumnOperators.bitwise_not()`、`ColumnOperators.bitwise_lshift()`、`ColumnOperators.bitwise_rshift()`添加了类型信息。感谢 Martijn Pieters 的拉取请求。

    参考：[#9650](https://www.sqlalchemy.org/trac/ticket/9650)

+   **[类型] [错误]**

    更新了代码库以通过 Mypy 1.2.0 进行类型检查。

+   **[类型] [错误]**

    修复了在`PropComparator.and_()`表达式在加载器选项中（如`selectinload()`）内部未正确类型化的问题。

    参考：[#9669](https://www.sqlalchemy.org/trac/ticket/9669)

### postgresql

+   **[postgresql] [用例]**

    在 asyncpg 方言中添加了`prepared_statement_name_func`连接参数选项。此选项允许传递一个可调用对象，用于自定义驱动程序在执行查询时将创建的准备好的语句的名称。感谢 Pavel Sirotkin 的拉取请求。

    另请参阅

    使用 PGBouncer 的准备好的语句名称

    参考：[#9608](https://www.sqlalchemy.org/trac/ticket/9608)

+   **[postgresql] [用例]**

    添加了缺失的`Range.intersection()`方法。感谢 Yurii Karabas 的拉取请求。

    参考：[#9509](https://www.sqlalchemy.org/trac/ticket/9509)

+   **[postgresql] [bug]**

    恢复了`ENUM`签名中的`ENUM.name`参数作为可选参数，因为这是从给定的 pep-435 `Enum`类型中自动选择的。

    参考：[#9611](https://www.sqlalchemy.org/trac/ticket/9611)

+   **[postgresql] [bug]**

    修复了针对`ENUM`与普通字符串的比较，会将右侧类型转换为 VARCHAR 的问题，这是由于像 asyncpg 这样的方言添加了更明确的转换，会产生 PostgreSQL 类型不匹配错误。

    参考：[#9621](https://www.sqlalchemy.org/trac/ticket/9621)

+   **[postgresql] [bug]**

    修复了在 PostgreSQL 中阻止基于表达式的长表达式的反射的问题。表达式错误地截断为标识符长度（默认为 63 字节）。

    参考：[#9615](https://www.sqlalchemy.org/trac/ticket/9615)

### mssql

+   **[mssql] [bug]**

    恢复了 Microsoft SQL Server 的 insertmanyvalues 功能。该功能在版本 2.0.9 中被禁用，因为显然依赖于未保证的 RETURNING 排序。 “insertmanyvalues”功能的架构已经重构，以适应 INSERT 语句的特定组织和可以保证返回行与输入记录对应的结果行处理。

    另见

    将 RETURNING 行与参数集相关联

    参考：[#9603](https://www.sqlalchemy.org/trac/ticket/9603), [#9618](https://www.sqlalchemy.org/trac/ticket/9618)

### oracle

+   **[oracle] [bug]**

    修复了使用`Uuid`数据类型在 Oracle 方言的 INSERT..RETURNING 子句中无法使用的问题。

### orm

+   **[orm] [bug]**

    修复了各种 ORM 特定的 getter（例如`ORMExecuteState.is_column_load`、`ORMExecuteState.is_relationship_load`、`ORMExecuteState.loader_strategy_path`等）如果 SQL 语句本身是“复合选择”（例如 UNION）时，会抛出`AttributeError`的错误。

    这个更改也被**回溯**到：1.4.48

    参考：[#9634](https://www.sqlalchemy.org/trac/ticket/9634)

+   **[orm] [bug]**

    修复了当应用于`__mapper_args__`特殊方法名时，`declared_attr.directive()`修改器未正确遵守子类的问题，而不是直接使用`declared_attr`。这两种构造应具有相同的运行时行为。

    参考：[#9625](https://www.sqlalchemy.org/trac/ticket/9625)

+   **[orm] [bug]**

    对`with_loader_criteria()`加载选项进行了改进，允许在不是 ORM 语句本身的顶级语句的`Executable.options()`方法中指定。示例包括嵌入在复合语句中的`select()`，在`Insert.from_select()`构造中以及在顶级与 ORM 无关的 CTE 表达式中。

    参考：[#9635](https://www.sqlalchemy.org/trac/ticket/9635)

+   **[orm] [bug]**

    修复了 ORM 批量插入功能中的错误，如果请求了单个列的 RETURNING，则插入语句中将呈现额外的不必要列。

    参考：[#9685](https://www.sqlalchemy.org/trac/ticket/9685)

+   **[orm] [bug]**

    修复了 ORM 声明性数据类中的 bug，其中`query_expression()`和`column_property()`构造，在声明性映射的上下文中被记录为只读构造，不能在不添加`init=False`的情况下与`MappedAsDataclass`类一起使用，因为在`query_expression()`的情况下不可能包含`init`参数。这些构造已经从数据类的角度进行了修改，被假定为“只读”，默认设置`init=False`，不再包括在 pep-681 构造函数中。对于`column_property()`的数据类参数`init`、`default`、`default_factory`、`kw_only`现已被弃用；这些字段不适用于在声明性数据类配置中使用的`column_property()`，其中该构造将是只读的。还为`query_expression()`添加了特定于读取的参数`query_expression.compare`；`query_expression.repr`已经存在。

    参考：[#9628](https://www.sqlalchemy.org/trac/ticket/9628)

+   **[orm] [bug]**

    在`mapped_column()`构造中添加了缺失的`mapped_column.active_history`参数。

### engine

+   **[engine] [usecase]**

    添加了`create_pool_from_url()`和`create_async_pool_from_url()`来从作为字符串或`URL`传递的输入 url 创建一个`Pool`实例。

    参考：[#9613](https://www.sqlalchemy.org/trac/ticket/9613)

+   **[engine] [bug]**

    修复了在 2.0 系列中首次引入的 “插入多个值”INSERT 语句性能优化功能 中识别出的主要缺陷。这是对 2.0.9 中的更改的延续，该更改禁用了 SQL Server 版本的该功能，因为 ORM 依赖于不保证发生的明显行排序。该修复将新逻辑应用于所有“insertmanyvalues”操作，当在 `Insert.returning()` 或 `UpdateBase.return_defaults()` 方法上设置了新参数`Insert.returning.sort_by_parameter_order` 时生效，通过交替的 SQL 表单、直接客户端参数对应以及在某些情况下降级到逐行运行，将对返回的每批行应用排序，使用与每行中的主键或其他唯一值的对应关系与输入数据相关联。

    预期性能影响将是最小的，因为几乎所有常见的主键场景都适用于为所有后端实现参数排序的批处理，除了 SQLite，而“逐行”模式与 1.x 系列中使用的非常重量级的方法相比，具有最小的 Python 开销。对于 SQLite，在使用“逐行”模式时，性能没有区别。

    预计通过高效的“逐行”插入与具有 RETURNING 支持但不一定易于保证与参数顺序对应的第三方后端的批处理能力，“insertmanyvalues”功能将更容易地推广到第三方后端。

    另请参阅

    将 RETURNING 行与参数集相关联

    参考：[#9603](https://www.sqlalchemy.org/trac/ticket/9603), [#9618](https://www.sqlalchemy.org/trac/ticket/9618)

### 输入

+   **[输入] [错误]**

    为最近添加的运算符`ColumnOperators.icontains()`、`ColumnOperators.istartswith()`、`ColumnOperators.iendswith()` 和位运算符`ColumnOperators.bitwise_and()`、`ColumnOperators.bitwise_or()`、`ColumnOperators.bitwise_xor()`、`ColumnOperators.bitwise_not()`、`ColumnOperators.bitwise_lshift()` 和 `ColumnOperators.bitwise_rshift()` 添加了类型信息。拉取请求由 Martijn Pieters 提供。

    参考：[#9650](https://www.sqlalchemy.org/trac/ticket/9650)

+   **[类型] [错误]**

    更新代码库以通过 Mypy 1.2.0 进行类型检查。

+   **[类型] [错误]**

    修复了`PropComparator.and_()` 表达式在诸如`selectinload()`等加载器选项内未正确类型化的问题。

    参考：[#9669](https://www.sqlalchemy.org/trac/ticket/9669)

### postgresql

+   **[postgresql] [用例]**

    在 asyncpg 方言中添加了 `prepared_statement_name_func` 连接参数选项。此选项允许传递一个可调用对象，用于自定义驱动程序在执行查询时将创建的准备语句的名称。拉取请求由 Pavel Sirotkin 提供。

    另请参阅

    使用 PGBouncer 的预处理语句名称

    参考：[#9608](https://www.sqlalchemy.org/trac/ticket/9608)

+   **[postgresql] [用例]**

    添加了缺失的 `Range.intersection()` 方法。拉取请求由 Yurii Karabas 提供。

    参考：[#9509](https://www.sqlalchemy.org/trac/ticket/9509)

+   **[postgresql] [bug]**

    恢复了 `ENUM` 的签名中的可选参数 `ENUM.name` ，因为这会自动从给定的 pep-435 `Enum` 类型中选择。

    参考：[#9611](https://www.sqlalchemy.org/trac/ticket/9611)

+   **[postgresql] [bug]**

    修复了对 plain string 的 `ENUM` 的比较会将右侧类型转换为 VARCHAR 的问题，由于像 asyncpg 这样的方言添加了更明确的转换，这会产生 PostgreSQL 类型不匹配错误。

    参考：[#9621](https://www.sqlalchemy.org/trac/ticket/9621)

+   **[postgresql] [bug]**

    修复了 PostgreSQL 中基于表达式的索引的反射问题，当表达式过长时会出现问题。表达式错误地被截断为标识符长度（默认为 63 字节）。

    参考：[#9615](https://www.sqlalchemy.org/trac/ticket/9615)

### mssql

+   **[mssql] [bug]**

    恢复了 Microsoft SQL Server 中 insertmanyvalues 功能。该功能在版本 2.0.9 中被禁用，因为似乎依赖于不能保证的 RETURNING 的排序。"insertmanyvalues" 功能的架构已经重新设计，以适应 INSERT 语句的特定组织和结果行处理，可以保证返回的行与输入记录的对应关系。

    请参阅

    将 RETURNING 行与参数集相关联

    参考：[#9603](https://www.sqlalchemy.org/trac/ticket/9603), [#9618](https://www.sqlalchemy.org/trac/ticket/9618)

### oracle

+   **[oracle] [bug]**

    修复了在 Oracle 方言中无法在 INSERT..RETURNING 子句中使用 `Uuid` 数据类型的问题。

## 2.0.9

发布日期：2023 年 4 月 5 日

### orm

+   **[orm] [bug]**

    修复了在使用“别名类关系”功能并在加载器中指定递归的 eager loader（如 `lazy="selectinload"`）时可能发生的无限循环。已经修复了检查循环是否包括别名类关系的问题。

    这个变更也被**回溯**到：1.4.48

    参考：[#9590](https://www.sqlalchemy.org/trac/ticket/9590)

### mariadb

+   **[mariadb] [bug]**

    在 MariaDb 中添加了 `row_number` 作为保留字。

    参考：[#9588](https://www.sqlalchemy.org/trac/ticket/9588)

### mssql

+   **[mssql] [bug]**

    SQLAlchemy 的“insertmanyvalues”功能允许快速插入多行数据，同时支持 RETURNING，在 SQL Server 上暂时禁用。由于工作单元目前依赖于此功能，使其匹配现有的 ORM 对象与返回的主键标识，这种特定的使用模式在某些情况下与 SQL Server 不兼容，因为“OUTPUT inserted”返回的行的顺序可能不总是与发送的元组的顺序相匹配，导致 ORM 在后续操作中对这些对象做出错误决定。

    此功能将在即将发布的版本中重新启用，并将再次对多行 INSERT 语句生效，但是工作单元对该功能的使用将被禁用，可能对所有方言都禁用，除非 ORM 映射的表也包括一个“sentinel”列，以便将返回的行引用回传入的原始数据。

    参考：[#9603](https://www.sqlalchemy.org/trac/ticket/9603)

+   **[mssql] [bug]**

    更改了在设置`fast_executemany`为`True`时用于 SQL Server“executemany”的批量 INSERT 策略，通过使用`fast_executemany` / `cursor.executemany()`来进行批量 INSERT，不包括 RETURNING，恢复了当此参数设置时使用的与 SQLAlchemy 1.4 相同的行为。

    最终用户提供的新性能细节显示，对于非常大的数据集，`fast_executemany`仍然比“insertmanyvalues”更快，因为它使用可以在单次往返中接收所有行的 ODBC 命令，允许比 SQL Server 实现的批量发送的数据大小大得多。

    尽管进行了此更改以使“insertmanyvalues”继续用于包含 RETURNING 的 INSERT，以及如果未设置`fast_executemany`，但由于[#9603](https://www.sqlalchemy.org/trac/ticket/9603)，“insertmanyvalues”策略已在任何情况下禁用了 SQL Server。

    参考：[#9586](https://www.sqlalchemy.org/trac/ticket/9586)

### orm

+   **[orm] [bug]**

    修复了使用“关联到别名类”功能时可能发生的无限循环问题，同时在加载器中指示了递归急加载器，例如`lazy="selectinload"`，与另一个相对立的急加载器组合使用时。检查循环的代码已修复以包括别名类关系。

    此更改还**被回溯**到：1.4.48

    参考：[#9590](https://www.sqlalchemy.org/trac/ticket/9590)

### mariadb

+   **[mariadb] [bug]**

    在 MariaDb 中添加了`row_number`作为保留字。

    参考：[#9588](https://www.sqlalchemy.org/trac/ticket/9588)

### mssql

+   **[mssql] [bug]**

    SQLAlchemy 的“insertmanyvalues”功能，允许快速插入多行数据同时支持 RETURNING，暂时在 SQL Server 上被禁用。由于工作单元目前依赖于此功能，以便将现有 ORM 对象与返回的主键标识匹配，这种特定的使用模式在某些情况下与 SQL Server 不兼容，因为“OUTPUT inserted”返回的行的顺序可能不总是与发送的元组的顺序匹配，导致 ORM 在后续操作中对这些对象做出错误决策。

    该功能将在即将发布的版本中重新启用，并且将再次对多行 INSERT 语句生效，但是除非 ORM 映射的表也包括一个“sentinel”列，以便返回的行可以被引用回传递的原始数据，否则该功能将被禁用，可能对所有方言都禁用。

    参考：[#9603](https://www.sqlalchemy.org/trac/ticket/9603)

+   **[mssql] [bug]**

    当`fast_executemany`设置为`True`时，使用 pyodbc 为 SQL Server 执行批量 INSERT 的“executemany”策略已更改为使用`fast_executemany` / `cursor.executemany()`，用于不包含 RETURNING 的批量 INSERT，恢复了在设置此参数时在 SQLAlchemy 1.4 中使用的相同行为。

    最终用户提供的新性能细节显示，对于非常大的数据集，`fast_executemany`仍然比“insertmanyvalues”更快，因为它使用可以在单次往返中接收所有行的 ODBC 命令，允许比 SQL Server 实现的批量发送的数据量大得多。

    尽管此更改是为了使“insertmanyvalues”继续用于包含 RETURNING 的 INSERT，以及如果未设置`fast_executemany`，但由于[#9603](https://www.sqlalchemy.org/trac/ticket/9603)，在任何情况下，“insertmanyvalues”策略已在 SQL Server 上被禁用。

    参考：[#9586](https://www.sqlalchemy.org/trac/ticket/9586)

## 2.0.8

发布日期：2023 年 3 月 31 日

### orm

+   **[orm] [用例]**

    当使用`MappedAsDataclass`混合类或`registry.mapped_as_dataclass()`装饰器时，Python dataclasses 引发的`TypeError`和`ValueError`等异常现在会被包装在一个带有有关错误消息的信息性上下文的`InvalidRequestError`包装器中，指向 Python dataclasses 文档作为异常原因的背景信息的权威来源。

    另请参阅

    为<classname>创建数据类时遇到的 Python dataclasses 错误

    参考：[#9563](https://www.sqlalchemy.org/trac/ticket/9563)

+   **[orm] [bug]**

    修复了在 ORM Annotated Declarative 中使用递归类型（例如使用嵌套的字典类型）会导致 ORM 的注释解析逻辑中发生递归溢出的问题，即使此数据类型对于映射列并非必需。

    参考：[#9553](https://www.sqlalchemy.org/trac/ticket/9553)

+   **[orm] [bug]**

    修复了在 Declarative 混合中使用 `mapped_column.deferred` 参数时，`mapped_column()` 结构会引发内部错误的问题。

    参考：[#9550](https://www.sqlalchemy.org/trac/ticket/9550)

+   **[orm] [bug]**

    扩展了当在 Declarative 映射中存在一个纯 `column()` 对象时发出的警告，以包括任何未在适当属性类型中声明的任意 SQL 表达式，例如 `column_property()`、`deferred()` 等。这些属性否则根本不被映射，并且在类字典中保持不变。由于这种表达式通常不是预期的情况，因此现在对所有这些否则被忽略的表达式发出警告，而不仅仅是 `column()` 情况。

    参考：[#9537](https://www.sqlalchemy.org/trac/ticket/9537)

+   **[orm] [bug]**

    修复了一个回归问题，在一个未映射或尚未映射的类上访问混合属性的表达式值（例如，在`declared_attr()`方法中调用它）会引发内部错误，因为对父类映射器的内部获取将失败，并且这种失败的指示在 2.0 中被意外删除。

    参考：[#9519](https://www.sqlalchemy.org/trac/ticket/9519)

+   **[orm] [bug]**

    在 Declarative Mixins 上声明的字段，然后与使用 `MappedAsDataclass` 的类组合，其中这些混合字段本身不是数据类的一部分，现在会发出弃用警告，因为在将来的版本中将会忽略这些字段，因为 Python 数据类的行为是忽略这些字段。类型检查器不会在 pep-681 下看到这些字段。

    另见

    将 <cls> 转换为数据类时，属性来源于不是数据类的超类 <cls>。 - 背景理由

    使用混合和抽象超类

    参考：[#9350](https://www.sqlalchemy.org/trac/ticket/9350)

+   **[orm] [bug]**

    修复了一个问题，即当调用具有 ORM 注释的参数的`BindParameter.render_literal_execute()`方法时会失败。在实践中，当使用一些使用“FETCH FIRST”（如 Oracle）的方言与使用`Select.limit()`的`Select`构造的某些组合时，会观察到 SQL 编译失败，包括在某些 ORM 上下文中，包括如果语句嵌入在关系主连接表达式中。

    参考：[#9526](https://www.sqlalchemy.org/trac/ticket/9526)

+   **[orm] [bug]**

    为了与为[#5984](https://www.sqlalchemy.org/trac/ticket/5984)和[#8862](https://www.sqlalchemy.org/trac/ticket/8862)所做的工作单元一致性更改保持一致，这两个更改禁用了在`Session`进程中“lazy='raise'”处理，这些进程不是由属性访问触发的，`Session.delete()`方法现在也会在遍历关系路径以处理“delete”和“delete-orphan”级联规则时禁用“lazy='raise'”处理。以前，没有简单的方法可以通用地调用具有设置了“lazy='raise'”的对象的`Session.delete()`，以便仅加载必要的关系。由于“lazy='raise'”主要用于捕获在属性访问时发出的 SQL 加载，`Session.delete()`现在被制作成像其他`Session`方法一样，包括`Session.merge()`以及`Session.flush()`以及自动刷新。

    参考：[#9549](https://www.sqlalchemy.org/trac/ticket/9549)

+   **[orm] [bug]**

    修复了一个问题，即只有注释的`Mapped`指令无法在声明性混合类中使用，而不会导致已经在超类上映射了该属性的映射类的单个或联接继承子类尝试生效，从而产生冲突的列错误和/或警告。

    参考文献：[#9564](https://www.sqlalchemy.org/trac/ticket/9564)

+   **[orm] [错误] [类型]**

    修正了 `Insert.from_select.names` 的类型，以接受字符串、列或映射属性的列表。

    参考文献：[#9514](https://www.sqlalchemy.org/trac/ticket/9514)

### 示例

+   **[示例] [错误]**

    修复了“版本化历史”示例中使用从 `DeclarativeBase` 派生的声明基类会失败映射的问题。此外，修复了给定的测试套件，以便通过 Python unittest 运行示例的文档说明现在再次有效。

### 类型

+   **[类型] [错误]**

    修复了 `deferred()` 和 `query_expression()` 的类型，使其与 2.0 风格的映射正确工作。

    参考文献：[#9536](https://www.sqlalchemy.org/trac/ticket/9536)

### postgresql

+   **[postgresql] [错误]**

    修复了在 PostgreSQL 方言中的关键回归，例如 asyncpg 依赖于 SQL 中的显式转换，以便正确传递数据类型给驱动程序的问题，其中 `String` 数据类型将与要比较的确切列长度一起转换，导致在比较较小长度的 `VARCHAR` 与较大长度的字符串时，无论使用的操作符如何（例如 LIKE、MATCH 等），都会导致隐式截断。现在，PostgreSQL 方言在呈现这些转换时省略了 `VARCHAR` 的长度。

    参考文献：[#9511](https://www.sqlalchemy.org/trac/ticket/9511)

### mysql

+   **[mysql] [错误]** 

    修复了字符串数据类型（如 `CHAR`、`VARCHAR`、`TEXT`）以及二进制 `BLOB` 不能以零长度显式生成的问题，这在 MySQL 中具有特殊含义。感谢 J. Nick Koston 提交的拉取请求。

    参考文献：[#9544](https://www.sqlalchemy.org/trac/ticket/9544)

### 杂项

+   **[错误] [工具]**

    在 OrderedSet 类中实现了缺失的 `copy` 和 `pop` 方法。

    参考文献：[#9487](https://www.sqlalchemy.org/trac/ticket/9487)

### orm

+   **[orm] [用例]**

    通过使用`MappedAsDataclass` mixin 类或`registry.mapped_as_dataclass()`装饰器时，由 Python 数据类引发的`TypeError`和`ValueError`等异常现在被包装在一个`InvalidRequestError`包装器中，并提供有关错误消息的信息上下文，指向 Python 数据类文档作为异常原因的权威来源。

    另请参阅

    创建<classname>的数据类时遇到的 Python 数据类错误

    参考：[#9563](https://www.sqlalchemy.org/trac/ticket/9563)

+   **[orm] [bug]**

    修复了在 ORM 注释式声明中的问题，当使用递归类型（例如使用嵌套的 Dict 类型）时，即使此数据类型不是必要的以映射列，也会导致 ORM 的注释解析逻辑出现递归溢出。

    参考：[#9553](https://www.sqlalchemy.org/trac/ticket/9553)

+   **[orm] [bug]**

    修复了在声明式混合中使用`mapped_column()`构造会引发内部错误的问题，并且包括`mapped_column.deferred`参数。

    参考：[#9550](https://www.sqlalchemy.org/trac/ticket/9550)

+   **[orm] [bug]**

    当在声明式映射中存在一个纯粹的`column()`对象时，扩展了发出警告的情况，以包括任何未在适当的属性类型内声明的任意 SQL 表达式，如`column_property()`，`deferred()`等。这些属性否则不被映射，并且在类字典中保持不变。由于这样的表达式通常不是所期望的，因此现在对所有这些被忽略的表达式发出警告，而不仅仅是`column()`的情况。

    参考：[#9537](https://www.sqlalchemy.org/trac/ticket/9537)

+   **[orm] [bug]**

    修复了一个回归问题，即在尝试访问未映射或尚未映射（例如在 `declared_attr()` 方法中调用它）的类上的混合属性的表达式值时，会引发内部错误，因为对父类映射器的内部获取将失败，并且无意中删除了对此失败忽略指令的引用。

    参考文献：[#9519](https://www.sqlalchemy.org/trac/ticket/9519)

+   **[orm] [bug]**

    在使用 `MappedAsDataclass` 的类中与声明在 Declarative Mixins 上然后与使用 `MappedAsDataclass` 的类结合在一起的字段（其中这些混合字段本身不是数据类的一部分）现在会发出停用警告，因为这些字段将在将来的版本中被忽略，因为 Python 数据类的行为是忽略这些字段。类型检查器将不会在 pep-681 下看到这些字段。

    另请参阅

    将 <cls> 转换为数据类时，属性来自于不是数据类的超类 <cls>。 - 背景和理由

    使用混合类和抽象超类

    参考文献：[#9350](https://www.sqlalchemy.org/trac/ticket/9350)

+   **[orm] [bug]**

    修复了一个问题，即当调用具有 ORM 注解的参数的 `BindParameter.render_literal_execute()` 方法时会失败。在实践中，这将被观察为在使用一些使用“FETCH FIRST”（例如 Oracle）的方言与使用 `Select` 构造的 `Select.limit()` 的某些组合时，当在某些 ORM 上下文中使用时，包括如果语句嵌入在关系主连接表达式中时，会触发 SQL 编译失败。

    参考文献：[#9526](https://www.sqlalchemy.org/trac/ticket/9526)

+   **[orm] [bug]**

    为了与为[#5984](https://www.sqlalchemy.org/trac/ticket/5984)和[#8862](https://www.sqlalchemy.org/trac/ticket/8862)所做的工作单元一致性更改保持一致，这两个更改禁用了在未通过属性访问触发的`Session`进程中的“lazy='raise'”处理，`Session.delete()`方法现在也会在遍历关系路径以处理“delete”和“delete-orphan”级联规则时禁用“lazy='raise'”处理。以前，没有简单的方法可以通用地调用具有设置“lazy='raise'”的对象的`Session.delete()`，以便仅加载必要的关系。由于“lazy='raise'”主要用于捕获在属性访问时发出的 SQL 加载，`Session.delete()`现在被制作成像其他`Session`方法一样，包括`Session.merge()`以及`Session.flush()`以及自动刷新。

    参考：[#9549](https://www.sqlalchemy.org/trac/ticket/9549)

+   **[orm] [bug]**

    修复了仅注释的`Mapped`指令无法在声明性混合类中使用的问题，而不会使该属性试图对已经在超类上映射了该属性的映射类的单个或联接继承子类产生冲突的列错误和/或警告。

    参考：[#9564](https://www.sqlalchemy.org/trac/ticket/9564)

+   **[orm] [bug] [typing]**

    正确地将`Insert.from_select.names`类型化为接受字符串列表或列或映射属性。

    参考：[#9514](https://www.sqlalchemy.org/trac/ticket/9514)

### 示例

+   **[示例] [bug]**

    修复了“版本历史”示例中的问题，使用从`DeclarativeBase`派生的声明基类会导致映射失败。此外，修复了给定的测试套件，使得使用 Python unittest 运行示例的文档说明现在再次有效。

### 输入

+   **[typing] [bug]**

    修复了`deferred()`和`query_expression()`在 2.0 风格映射中正确工作的类型问题。

    参考：[#9536](https://www.sqlalchemy.org/trac/ticket/9536)

### postgresql

+   **[postgresql] [bug]**

    修复了 PostgreSQL 方言中的关键回归，例如依赖 SQL 中显式转换以使数据类型正确传递给驱动程序的 asyncpg，其中`String`数据类型将与要比较的确切列长度一起转换，导致在比较`VARCHAR`长度较小的字符串与长度较大的字符串时隐式截断，无论使用的操作符是什么（例如 LIKE，MATCH 等）。现在，PostgreSQL 方言在渲染这些转换时省略了`VARCHAR`的长度。

    参考：[#9511](https://www.sqlalchemy.org/trac/ticket/9511)

### mysql

+   **[mysql] [bug]**

    修复了字符串数据类型如`CHAR`、`VARCHAR`、`TEXT`以及二进制`BLOB`无法以零长度显式生成的问题，这在 MySQL 中具有特殊含义。感谢 J. Nick Koston 的拉取请求。

    参考：[#9544](https://www.sqlalchemy.org/trac/ticket/9544)

### 杂项

+   **[bug] [util]**

    在 OrderedSet 类中实现了缺失的方法`copy`和`pop`。

    参考：[#9487](https://www.sqlalchemy.org/trac/ticket/9487)

## 2.0.7

发布日期：2023 年 3 月 18 日

### typing

+   **[typing] [bug]**

    修复了`composite()`不允许任意可调用项作为复合类来源的类型问题。

    参考：[#9502](https://www.sqlalchemy.org/trac/ticket/9502)

### postgresql

+   **[postgresql] [usecase]**

    添加了新的 PostgreSQL 类型`CITEXT`。感谢 Julian David Rath 的拉取请求。

    参考：[#9416](https://www.sqlalchemy.org/trac/ticket/9416)

+   **[postgresql] [usecase]**

    对基本 PostgreSQL 方言进行了修改，以便更好地与 SQLAlchemy 2.0 的第三方 dialect sqlalchemy-redshift 集成。感谢 matthewgdv 的拉取请求。

    参考：[#9442](https://www.sqlalchemy.org/trac/ticket/9442)

### typing

+   **[typing] [bug]**

    修复了`composite()`不允许任意可调用项作为复合类来源的类型问题。

    参考：[#9502](https://www.sqlalchemy.org/trac/ticket/9502)

### postgresql

+   **[postgresql] [usecase]**

    添加了新的 PostgreSQL 类型 `CITEXT`。感谢 Julian David Rath 提交的拉取请求。

    参考文献：[#9416](https://www.sqlalchemy.org/trac/ticket/9416)

+   **[postgresql] [usecase]**

    对基础的 PostgreSQL 方言进行修改，以便更好地与第三方方言 sqlalchemy-redshift 在 SQLAlchemy 2.0 中进行集成。感谢 matthewgdv 提交的拉取请求。

    参考文献：[#9442](https://www.sqlalchemy.org/trac/ticket/9442)

## 2.0.6

发布日期：2023 年 3 月 13 日

### orm

+   **[orm] [bug]**

    修复了一个错误，即“活动历史”功能对复合属性的实现不完整，导致无法接收包含“旧”值的事件。这似乎也是旧版 SQLAlchemy 的情况，其中“active_history”将传播到基础的基于列的属性，但是即使将 composite() 设置为 active_history=True，仍然不会向监听复合属性本身的事件处理程序提供将要替换的“旧”值。

    此外，修复了一个局限于 2.0 版本的回归，该回归禁止对复合属性的 active_history 分配给具有`attr.impl.active_history=True`的 impl。

    参考文献：[#9460](https://www.sqlalchemy.org/trac/ticket/9460)

+   **[orm] [bug]**

    修复了在 Python 行的 cython 和纯 Python 实现之间进行 pickling 时的回归，该回归发生在为版本 2.0 进行代码重构和添加类型时。特定常量被转换为纯 Python 版本的字符串型 `Enum`，而 cython 版本继续使用整数常量，导致反序列化失败。

    参考文献：[#9418](https://www.sqlalchemy.org/trac/ticket/9418)

### sql

+   **[sql] [bug] [regression]**

    修复了一个回归，即对于在 1.4 系列中发布的修复 [#8098](https://www.sqlalchemy.org/trac/ticket/8098)，该修复为 lambda SQL API 提供了一层并发安全检查，包含在补丁中的额外修复未能应用到主分支。已应用这些额外修复。

    参考文献：[#9461](https://www.sqlalchemy.org/trac/ticket/9461)

+   **[sql] [bug]**

    修复了一个回归，即如果 `select()` 构造未给定列并且在 EXISTS 的上下文中使用，则无法渲染，而是引发内部异常。虽然空“SELECT”通常不是有效的 SQL，但在 EXISTS 数据库（如 PostgreSQL）中允许它，在任何情况下，该条件现在不再引发内部异常。

    参考文献：[#9440](https://www.sqlalchemy.org/trac/ticket/9440)

### typing

+   **[typing] [bug]**

    修复了`ColumnElement.cast()`中的打字问题，该问题不允许独立于`ColumnElement`本身的`TypeEngine`参数，这就是`ColumnElement.cast()`的目的。

    参考：[#9451](https://www.sqlalchemy.org/trac/ticket/9451)

+   **[打字] [错误]**

    修复了允许打字测试在 Mypy 1.1.1 下通过的问题。

### oracle

+   **[oracle] [错误]**

    修复了反射错误，Oracle 的“名称规范化”对于`"PUBLIC"`模式中的符号的反射无法正确工作，比如同义词，这意味着在 Python 端无法将 PUBLIC 名称指示为小写以用于`Table.schema`参数。使用大写的`"PUBLIC"`会起作用，但会导致包括带引号的`"PUBLIC"`名称的尴尬 SQL 查询以及将表索引化为大写的`"PUBLIC"`，这是不一致的。

    参考：[#9459](https://www.sqlalchemy.org/trac/ticket/9459)

### orm

+   **[orm] [错误]**

    修复了一个 bug，即“活动历史”功能对于复合属性并未完全实现，使得无法接收包含“旧”值的事件。这似乎也是旧的 SQLAlchemy 版本的情况，其中“active_history”会传播到基于列的属性，但是一个监听复合属性本身的事件处理程序将不会收到被替换的“旧”值，即使复合()设置为 active_history=True。

    另外，修复了一个仅限于 2.0 的回归，不允许将 composite 上的 active_history 分配给`attr.impl.active_history=True`。

    参考：[#9460](https://www.sqlalchemy.org/trac/ticket/9460)

+   **[orm] [错误]**

    修复了在版本 2.0 中为了打字重构代码而发生的 Python 行在 cython 和纯 Python 实现之间的 pickling 问题，其中一个特定的常量被转换为纯 Python 版本的`Row`的基于字符串的`Enum`，而 cython 版本继续使用整数常量，导致反序列化失败。

    参考：[#9418](https://www.sqlalchemy.org/trac/ticket/9418)

### sql

+   **[sql] [错误] [回归]**

    修复了回归问题，该问题导致在 1.4 系列中发布的 [#8098](https://www.sqlalchemy.org/trac/ticket/8098) 的修复为 lambda SQL API 提供了一层并发安全检查，包含在未能应用到主分支的补丁中的其他修复。 这些额外的修复已经应用。

    参考：[#9461](https://www.sqlalchemy.org/trac/ticket/9461)

+   **[sql] [bug]**

    修复了当 `select()` 构造没有列并且在 EXISTS 上下文中使用时，将无法渲染的回归问题，而是引发内部异常。 虽然空的“SELECT”通常不是有效的 SQL，但在 EXISTS 数据库中，如 PostgreSQL 允许它，并且无论如何，该条件现在不再引发内部异常。

    参考：[#9440](https://www.sqlalchemy.org/trac/ticket/9440)

### typing

+   **[typing] [bug]**

    修复了 `ColumnElement.cast()` 不允许独立于 `ColumnElement` 类型本身的 `TypeEngine` 参数的类型问题，这是 `ColumnElement.cast()` 的目的。

    参考：[#9451](https://www.sqlalchemy.org/trac/ticket/9451)

+   **[typing] [bug]**

    修复了问题以允许在 Mypy 1.1.1 下通过类型测试。

### oracle

+   **[oracle] [bug]**

    修复了 Oracle 的“名称规范化”反射错误，该错误会导致对于位于`"PUBLIC"`模式中的符号的反射无法正确工作，例如同义词，意味着在 Python 侧无法将 PUBLIC 名称指示为小写形式以供`Table.schema`参数使用。 使用大写的`"PUBLIC"`会起作用，但随后会导致包括带引号的 `"PUBLIC"` 名称以及在大写`"PUBLIC"`下对表进行索引的尴尬 SQL 查询，这是不一致的。

    参考：[#9459](https://www.sqlalchemy.org/trac/ticket/9459)

## 2.0.5.post1

发布日期：2023 年 3 月 5 日

### orm

+   **[orm] [bug]**

    向内置映射集合类型添加构造函数参数，包括`KeyFuncDict`、`attribute_keyed_dict()`、`column_keyed_dict()`，以便这些字典类型可以立即根据提前给定的数据构建；这进一步与诸如 Python 数据类`.asdict()`之类的工具兼容，该工具依赖于直接调用这些类作为普通字典类。

    参考：[#9418](https://www.sqlalchemy.org/trac/ticket/9418)

+   **[orm] [bug] [regression]**

    由于[#8372](https://www.sqlalchemy.org/trac/ticket/8372)引起的多个回归已修复，涉及`attribute_mapped_collection()`（现在称为`attribute_keyed_dict()`）。

    首先，该集合不再可用于不是普通映射属性的“key”属性；与描述符和/或关联代理属性相关的属性已经修复。

    其次，如果事件或其他操作需要访问“key”以从未加载的映射属性填充字典，这也会不当地引发错误，而不是像 1.4 版本中的行为那样尝试加载属性。这也已修复。

    对于这两种情况，[#8372](https://www.sqlalchemy.org/trac/ticket/8372)的行为已扩展。[#8372](https://www.sqlalchemy.org/trac/ticket/8372)引入了一个错误，当将作为映射字典键使用的派生键实际上未分配时会引发错误。在此更改中，仅在“.key”属性的有效值为`None`时发出警告，其中无法明确确定此`None`是否是有意的。`None`将不再作为映射集合字典键支持（因为它通常指 NULL，表示“未知”）。设置`attribute_keyed_dict.ignore_unpopulated_attribute`现在将导致这些`None`键被忽略。

    参考：[#9424](https://www.sqlalchemy.org/trac/ticket/9424)

+   **[orm] [bug]**

    确定 `sqlite` 和 `mssql+pyodbc` 方言现在与 SQLAlchemy ORM 的“版本化行”功能兼容，因为 SQLAlchemy 现在通过计算返回的行数来计算 RETURNING 语句的行数，而不是依赖于 `cursor.rowcount`。特别是，ORM 版本化行用例（文档化的在 配置版本计数器）现在应该完全支持 SQL Server pyodbc 方言。

+   **[ORM] [错误]**

    在继承层次结构中的每个映射器上应用`Mapper.polymorphic_load`参数的支持已经添加，允许使用单个语句为层次结构中所有指示 `"selectin"` 的类加载列，而不是忽略那些中间类上的元素，尽管它们也指示它们将参与 `"selectin"` 加载，并且不是基本最底层的 SELECT 语句的一部分。

    参考：[#9373](https://www.sqlalchemy.org/trac/ticket/9373)

+   **[ORM] [错误]**

    继续修复了 [#8853](https://www.sqlalchemy.org/trac/ticket/8853)，允许 `Mapped` 名称在是否存在 `from __annotations__ import future` 的情况下都可以是完全限定的。此问题首次在 2.0.0b3 中修复，确认这种情况通过测试套件工作，但测试套件显然未测试名称为 `Mapped` 的行为是否根本不存在；字符串解析已更新以确保 ORM 如何使用这些函数的 `Mapped` 符号是可定位的。

    参考：[#8853](https://www.sqlalchemy.org/trac/ticket/8853), [#9335](https://www.sqlalchemy.org/trac/ticket/9335)

### ORM 声明式

+   **[错误] [ORM 声明式]**

    修复了一个问题，即如果两个同名列被映射到与列本身命名不同的属性名下，则新的`mapped_column.use_existing_column`功能将无法工作。当使用此参数时，现在可以对属性名称进行不同的命名。

    参考：[#9332](https://www.sqlalchemy.org/trac/ticket/9332)

### 引擎

+   **[引擎] [性能]**

    对 `Result` 的 Cython 实现进行了小优化，使用特定 int 值的 cdef 来避免 Python 开销。感谢 Matus Valo 的拉取请求。

    参考：[#9343](https://www.sqlalchemy.org/trac/ticket/9343)

+   **[引擎] [错误]**

    修复了`Row`对象由于意外依赖于不稳定的哈希值而无法可靠地跨进程反序列化的错误。

    参考：[#9423](https://www.sqlalchemy.org/trac/ticket/9423)

### sql

+   **[sql] [错误] [回归]**

    将`nullslast()`和`nullsfirst()`旧函数恢复到`sqlalchemy`导入命名空间中。先前，较新的`nulls_last()`和`nulls_first()`函数是可用的，但旧函数被不小心移除了。

    参考：[#9390](https://www.sqlalchemy.org/trac/ticket/9390)

### 模式

+   **[模式]**

    验证当提供了`MetaData.schema`参数时，`MetaData`是一个字符串。

### 输入

+   **[输入] [用例]**

    导出由`scoped_session.query_property()`返回的类型，使用一个新的公共类型`QueryPropertyDescriptor`。

    参考：[#9338](https://www.sqlalchemy.org/trac/ticket/9338)

+   **[输入] [错误]**

    修复了`Connection.scalars()`方法未被类型化为允许多参数列表的错误，现在支持使用`insertmanyvalues`操作。

+   **[输入] [错误]**

    改进了传递给`Insert.values()`和`Update.values()`的映射的类型，使其对集合类型更加开放，通过指示只读`Mapping`而不是可写`Dict`，后者在键类型过于受限时会出错。

    参考：[#9376](https://www.sqlalchemy.org/trac/ticket/9376)

+   **[输入] [错误]**

    为`Numeric`类型对象添加了缺失的 init 重载，以便 pep-484 类型检查器可以正确解析完整类型，从`Numeric.asdecimal`参数派生出`Decimal`或`float`对象将被表示。

    参考：[#9391](https://www.sqlalchemy.org/trac/ticket/9391)

+   **[键入] [错误]**

    修复了`Select.from_statement()`中的键入错误，该方法不接受`text()`或`TextualSelect`对象作为有效类型。此外，修复了`columns`方法的返回类型，该方法之前缺少了返回类型。

    参考：[#9398](https://www.sqlalchemy.org/trac/ticket/9398)

+   **[键入] [错误]**

    修复了`with_polymorphic()`中的键入问题，该问题导致类类型无法正确记录。

    参考：[#9340](https://www.sqlalchemy.org/trac/ticket/9340)

### postgresql

+   **[postgresql] [错误]**

    修复了 PostgreSQL `ExcludeConstraint` 中的问题，其中文字值被编译为绑定参数而不是直接的内联值，这是 DDL 所要求的。

    参考：[#9349](https://www.sqlalchemy.org/trac/ticket/9349)

+   **[postgresql] [错误]**

    修复了 PostgreSQL `ExcludeConstraint` 构造在一些操作中无法被复制的问题，例如`Table.to_metadata()`以及一些 Alembic 场景，如果约束包含文本表达式元素。

    参考：[#9401](https://www.sqlalchemy.org/trac/ticket/9401)

### mysql

+   **[mysql] [错误] [postgresql]**

    对于在 2.0.0b1 中添加的通过`DialectEvents.handle_error()`事件接收异常事件的池 ping 监听器的支持，未考虑到诸如 MySQL 和 PostgreSQL 的特定于方言的 ping 例程。方言功能已经重写，以便所有方言都参与事件处理。另外，添加了一个新的布尔元素`ExceptionContext.is_pre_ping`，用于标识此操作是否在预 ping 操作中发生。

    在这个版本中，实现自定义`Dialect.do_ping()`方法的第三方方言可以选择使用新的改进行为，方法不再捕获异常或检查异常是否为“is_disconnect”，而是将所有异常传播到外部。现在，默认方言的一个封闭方法会检查异常是否为“断开连接”异常，这样可以确保在将异常视为“断开连接”异常之前调用事件钩子处理所有异常情况。如果现有的 `do_ping()` 方法继续捕获异常并检查“is_disconnect”，则它将继续像以前一样工作，但是如果异常不被传播到外部，`handle_error` 钩子将无法访问异常。

    参考：[#5648](https://www.sqlalchemy.org/trac/ticket/5648)

### sqlite

+   **[sqlite] [bug] [regression]**

    修复了 SQLite 连接的回归问题，旧版本 SQLite（版本 3.8.3 之前）在建立数据库函数时使用 `deterministic` 参数会失败。改进了版本检查逻辑以适应这种情况。

    参考：[#9379](https://www.sqlalchemy.org/trac/ticket/9379)

### mssql

+   **[mssql] [bug]**

    修复了新 `Uuid` 数据类型与 pymssql 驱动程序不兼容的问题。由于 pymssql 看起来又开始维护了，因此恢复了对 pymssql 的测试支持。

    参考：[#9414](https://www.sqlalchemy.org/trac/ticket/9414)

+   **[mssql] [bug]**

    调整了 pymssql 方言以更好地利用 INSERT 语句的 RETURNING，在插入语句中检索最后插入的主键值，方式与当前 mssql+pyodbc 方言相同。

### misc

+   **[bug] [ext]**

    修复了 automap 中的问题，从特定映射类而不是直接从 `AutomapBase` 调用 `AutomapBase.prepare()` 时，当 automap 检测到新表时不会使用正确的基类，而是使用给定的类，导致映射器尝试配置继承。虽然通常应该从基类调用 `AutomapBase.prepare()`，但当从子类调用时不应该出现严重问题。

    参考：[#9367](https://www.sqlalchemy.org/trac/ticket/9367)

+   **[bug] [ext] [regression]**

    修复了由于增加到 `sqlalchemy.ext.mutable` 的类型而引起的回归问题，该类型用于 [#8667](https://www.sqlalchemy.org/trac/ticket/8667)，其中 `.pop()` 方法的语义发生了变化，导致该方法无法使用。拉取请求由 Nils Philippsen 提供。

    参考：[#9380](https://www.sqlalchemy.org/trac/ticket/9380)

### orm

+   **[orm] [bug]**

    对内置映射集合类型（包括`KeyFuncDict`、`attribute_keyed_dict()`、`column_keyed_dict()`）添加了构造函数参数，以便可以立即构造这些字典类型，从而进一步与诸如 Python 数据类`.asdict()`之类的工具兼容，后者依赖于直接调用这些类作为普通字典类。

    参考：[#9418](https://www.sqlalchemy.org/trac/ticket/9418)

+   **[orm] [bug] [regression]**

    由于[#8372](https://www.sqlalchemy.org/trac/ticket/8372)，修复了多个回归问题，涉及`attribute_mapped_collection()`（现在称为`attribute_keyed_dict()`）。

    首先，集合再也不能使用不是普通映射属性的“键”属性；已修复与描述符和/或关联代理属性相关联的属性。

    其次，如果事件或其他操作需要访问“键”以从未加载的映射属性填充字典，则这也会不适当地引发错误，而不是像 1.4 版本中的行为一样尝试加载属性。这也已经修复。

    对于这两种情况，扩展了[#8372](https://www.sqlalchemy.org/trac/ticket/8372)的行为。[#8372](https://www.sqlalchemy.org/trac/ticket/8372)引入了一个错误，当作为映射字典键使用的派生键实际上未被分配时，会引发错误。在这个变化中，仅在“.key”属性的有效值为`None`时发出警告，因此无法明确确定此`None`是有意的还是无意的。`None`将不再支持作为映射集合字典键（因为它通常指的是 NULL，意味着“未知”）。设置`attribute_keyed_dict.ignore_unpopulated_attribute`现在也将导致忽略此类`None`键。

    参考：[#9424](https://www.sqlalchemy.org/trac/ticket/9424)

+   **[orm] [bug]**

    发现`sqlite`和`mssql+pyodbc`方言现在与 SQLAlchemy ORM 的“版本行”功能兼容，因为 SQLAlchemy 现在通过计算返回的行来计算此特定情况下的 RETURNING 语句的行数，而不是依赖于`cursor.rowcount`。特别是 ORM 版本行用例（在配置版本计数器文档中记录）现在应该完全受到 SQL Server pyodbc 方言的支持。

+   **[orm] [bug]**

    增加对`Mapper.polymorphic_load`参数的支持，可以应用到继承层次结构中每个映射器超过一级深度，允许列为层次结构中的所有类加载指示为`"selectin"`的列，而不是忽略那些中间类上的元素，尽管它们也指示它们也将参与`"selectin"`加载，并且不是基本的 SELECT 语句的一部分。

    参考：[#9373](https://www.sqlalchemy.org/trac/ticket/9373)

+   **[orm] [bug]**

    继续修复[#8853](https://www.sqlalchemy.org/trac/ticket/8853)，允许`Mapped`名称完全限定，无论是否存在`from __annotations__ import future`。此问题首次在 2.0.0b3 中修复，确认了此情况通过测试套件工作，但测试套件显然未测试名为`Mapped`的名称是否根本不存在的情况；字符串解析已更新以确保 ORM 如何使用这些函数时，`Mapped`符号是可定位的。

    参考：[#8853](https://www.sqlalchemy.org/trac/ticket/8853)，[#9335](https://www.sqlalchemy.org/trac/ticket/9335)

### ORM 声明式

+   **[bug] [ORM 声明式]**

    修复了新的`mapped_column.use_existing_column`特性不会工作的问题，如果两个同名列被映射到与列本身给定的显式名称不同命名的属性名称下。当使用此参数时，属性名称现在可以不同命名。

    参考：[#9332](https://www.sqlalchemy.org/trac/ticket/9332)

### 引擎

+   **[engine] [performance]**

    对 Cython 实现的`Result`进行了一个小优化，使用 cdef 来避免 Python 开销。拉请求归功于 Matus Valo。

    参考：[#9343](https://www.sqlalchemy.org/trac/ticket/9343)

+   **[engine] [bug]**

    修复了一个 bug，`Row`对象由于意外依赖于不稳定的哈希值，跨进程无法可靠地反序列化。

    参考：[#9423](https://www.sqlalchemy.org/trac/ticket/9423)

### sql

+   **[sql] [bug] [regression]**

    将`nullslast()`和`nullsfirst()`这两个旧版函数恢复到`sqlalchemy`的导入命名空间中。之前，较新的`nulls_last()`和`nulls_first()`函数是可用的，但旧版函数被意外删除了。

    参考：[#9390](https://www.sqlalchemy.org/trac/ticket/9390)

### schema

+   **[schema]**

    验证当提供`MetaData`的`MetaData.schema`参数时，其值为字符串。

### typing

+   **[typing] [usecase]**

    导出了由`scoped_session.query_property()`返回的类型，使用了一个新的公共类型`QueryPropertyDescriptor`。

    参考：[#9338](https://www.sqlalchemy.org/trac/ticket/9338)

+   **[typing] [bug]**

    修复了一个 bug，`Connection.scalars()`方法未被类型化为允许多参数列表，现在支持使用 insertmanyvalues 操作。

+   **[typing] [bug]**

    改进了传递给`Insert.values()`和`Update.values()`的映射的类型，使其更加开放，指示只读`Mapping`而不是可写`Dict`，后者在键类型过于受限时会出错。

    参考：[#9376](https://www.sqlalchemy.org/trac/ticket/9376)

+   **[typing] [bug]**

    向`Numeric`类型对象添加了缺失的 init 重载，以便 pep-484 类型检查器可以正确解析完整类型，根据`Numeric.asdecimal`参数推断`Decimal`或`float`对象将如何表示。

    参考：[#9391](https://www.sqlalchemy.org/trac/ticket/9391)

+   **[输入] [错误]**

    修复了 `Select.from_statement()` 不接受 `text()` 或 `TextualSelect` 对象作为有效类型的类型错误。另外，修复了 `columns` 方法的返回类型，该方法缺失了。 

    参考：[#9398](https://www.sqlalchemy.org/trac/ticket/9398)

+   **[输入] [错误]**

    修复了 `with_polymorphic()` 无法正确记录类类型的输入问题。

    参考：[#9340](https://www.sqlalchemy.org/trac/ticket/9340)

### postgresql

+   **[postgresql] [错误]**

    修复了 PostgreSQL `ExcludeConstraint` 中的问题，其中文字值被编译为绑定参数而不是直接的内联值，这是 DDL 所要求的。

    参考：[#9349](https://www.sqlalchemy.org/trac/ticket/9349)

+   **[postgresql] [错误]**

    修复了 PostgreSQL `ExcludeConstraint` 构造中的问题，在某些情况下，如 `Table.to_metadata()` 中以及在一些 Alembic 场景中，如果约束包含文本表达式元素，则不可复制。 

    参考：[#9401](https://www.sqlalchemy.org/trac/ticket/9401)

### mysql

+   **[mysql] [错误] [postgresql]**

    对于 2.0.0b1 中通过 `DialectEvents.handle_error()` 事件添加的池预检监听器接收异常事件的支持，未考虑到诸如 MySQL 和 PostgreSQL 的方言特定的预检例程。方言特性已重新设计，以便所有方言在事件处理中参与其中。此外，添加了一个新的布尔元素 `ExceptionContext.is_pre_ping`，用于识别此操作是否发生在预检操作中。

    对于此版本，实现自定义 `Dialect.do_ping()` 方法的第三方方言可以选择通过不再捕获异常或检查异常的方式来选择新的改进行为，而只需将其方法外部传播所有异常。现在，默认方言的封闭方法执行异常检查“is_disconnect”，这确保在测试异常是否为“断开连接”异常之前调用事件钩子以处理所有异常场景。如果现有的 `do_ping()` 方法继续捕获异常并检查“is_disconnect”，则其行为将与以前一样工作，但是如果异常没有传播出去，则`handle_error`钩子将无法访问异常。

    参考：[#5648](https://www.sqlalchemy.org/trac/ticket/5648)

### sqlite

+   **[sqlite] [bug] [regression]**

    修复了针对 SQLite 连接的回归，其中在建立数据库函数时使用`deterministic`参数会在旧版本的 SQLite 中失败，即版本低于 3.8.3\. 版本检查逻辑已经改进以适应这种情况。

    参考：[#9379](https://www.sqlalchemy.org/trac/ticket/9379)

### mssql

+   **[mssql] [bug]**

    修复了新的 `Uuid` 数据类型与 pymssql 驱动程序无法正常工作的问题。由于 pymssql 看起来又得到了维护，因此恢复了对 pymssql 的测试支持。

    参考：[#9414](https://www.sqlalchemy.org/trac/ticket/9414)

+   **[mssql] [bug]**

    调整了 pymssql 方言，以更好地利用 RETURNING 语句来检索最后插入的主键值，与当前对 mssql+pyodbc 方言的情况相同。

### 杂项

+   **[bug] [ext]**

    修复了 automap 中的问题，其中从特定映射类而不是直接从 `AutomapBase` 调用 `AutomapBase.prepare()` 方法将不会在 automap 检测到新表时使用正确的基类，而是使用给定的类，导致映射器尝试配置继承关系。虽然通常情况下应该从基类调用 `AutomapBase.prepare()`，但是当从子类调用时不应该出现这种严重的问题。

    参考：[#9367](https://www.sqlalchemy.org/trac/ticket/9367)

+   **[bug] [ext] [regression]**

    修复了由于为[#8667](https://www.sqlalchemy.org/trac/ticket/8667)添加了类型而导致的回归，其中`.pop()`方法的语义发生了变化，使得该方法无法工作。感谢 Nils Philippsen 提交的拉取请求。

    参考：[#9380](https://www.sqlalchemy.org/trac/ticket/9380)

## 2.0.4

发布日期：2023 年 2 月 17 日

### orm

+   **[orm] [用例]**

    为了适应 SQLAlchemy 2.0 中 ORM 声明式使用的列排序更改，已添加了一个新参数 `mapped_column.sort_order` ，可用于控制 ORM 在表中定义的列的顺序，用于常见用例，例如具有应在表中首先显示的主键列的混合使用。变更说明中 ORM 声明式以不同方式应用列顺序；使用 sort_order 控制行为 说明了默认的排序行为更改（这是所有 SQLAlchemy 2.0 版本的一部分）以及在使用混合和多个类时使用 `mapped_column.sort_order` 控制列顺序。

    参见

    ORM 声明式以不同方式应用列顺序；使用 sort_order 控制行为

    参考：[#9297](https://www.sqlalchemy.org/trac/ticket/9297)

+   **[orm] [用例]**

    `Session.refresh()` 方法现在会立即加载一个在 `Session.refresh.attribute_names` 集合中明确命名的与关系绑定的属性，即使它当前连接到“select”加载器，通常是一个“懒加载器”，在刷新期间不会触发。现在，“懒加载器”策略将检测到该操作特别是一个由用户启动的 `Session.refresh()` 操作，该操作明确命名了此属性，然后将调用“immediateload”策略来实际发出 SQL 以加载属性。对于某些 asyncio 情况，必须强制加载未加载的惰性加载属性，而不使用实际不支持 asyncio 的惰性加载属性模式，这应该是有帮助的。

    参考：[#9298](https://www.sqlalchemy.org/trac/ticket/9298)

+   **[orm] [错误] [回归]**

    修复了在版本 2.0.2 中引入的回归问题，原因是 [#9217](https://www.sqlalchemy.org/trac/ticket/9217)，其中使用 DML RETURNING 语句以及`Select.from_statement()` 构造，就像在 [#9217](https://www.sqlalchemy.org/trac/ticket/9217) 中“修复”的一样，与使用表达式的 ORM 映射类一起，例如 `column_property()`，将导致 Core 内部错误，其中它将尝试按名称匹配表达式。修复修复了 Core 问题，并且还调整了在 DML RETURNING 用例中的修复 [#9217](https://www.sqlalchemy.org/trac/ticket/9217)，其中它添加了不必要的开销。

    参考：[#9273](https://www.sqlalchemy.org/trac/ticket/9273)

+   **[orm] [bug]**

    将内部的`EvaluatorCompiler`模块标记为 ORM 的私有，并将其重命名为`_EvaluatorCompiler`。对于可能依赖于此的用户，名称`EvaluatorCompiler`仍然存在，但不支持此用法，并将在未来的版本中移除。

### orm 声明式

+   **[用例] [orm 声明式]**

    在 `MappedAsDataclass` 类以及 `registry.mapped_as_dataclass()` 方法中添加了新参数 `dataclasses_callable`，该参数允许使用 Python `dataclasses.dataclass` 的替代可调用对象来生成数据类。此处的用例是替换 Pydantic 的 dataclass 函数。对于版本 2.0.1 中添加的对 [#9179](https://www.sqlalchemy.org/trac/ticket/9179) 的 mixin 支持进行了调整，以便重写 mixin 的 `__annotations__` 集合，以不包括 `Mapped` 容器，与映射类一样，这样 Pydantic 数据类的构造函数就不会暴露给未知类型。

    另请参阅

    与 Pydantic 等替代数据类提供程序集成

    参考：[#9266](https://www.sqlalchemy.org/trac/ticket/9266)

### sql

+   **[sql] [bug]**

    修复了元组值的元素类型将被硬编码为从比较元组中获取的类型的问题，当比较使用 `ColumnOperators.in_()` 运算符时。这与通常确定二元表达式类型的方式不一致，通常是首先考虑右侧的实际元素类型，然后应用左侧的类型。

    参考：[#9313](https://www.sqlalchemy.org/trac/ticket/9313)

+   **[sql]**

    添加了公共属性 `Table.autoincrement_column`，返回标识为自增的列。

    参考：[#9277](https://www.sqlalchemy.org/trac/ticket/9277)

### typing

+   **[typing] [usecase]**

    改进了对 Hybrid Attributes 扩展的类型支持，更新了所有文档以使用 ORM 注释性声明映射，并添加了一个名为 `hybrid_property.inplace` 的新修饰符。此修饰符提供了一种在原地更改 `hybrid_property` 状态的方法，这基本上是 SQLAlchemy 版本 1.2.0 之前非常早期的混合版本所做的，版本 1.2.0 [#3912](https://www.sqlalchemy.org/trac/ticket/3912) 改变了这一点，以删除原地变异。现在，这种原地变异已恢复为**选择加入**的基础，以允许单个混合具有多个设置的方法，而无需命名所有方法相同，也无需仔细地“链接”名称不同的方法以维护组合。类型工具如 Mypy 和 Pyright 不允许在类上具有相同名称的方法，因此通过此更改，恢复了一种具有类型支持的混合的简洁方法。

    另请参阅

    使用 inplace 创建符合 pep-484 标准的混合属性

    参考：[#9321](https://www.sqlalchemy.org/trac/ticket/9321)

### oracle

+   **[oracle] [bug]**

    调整了 python-oracledb 方言的 `thick_mode` 参数的行为，以正确接受 `False` 作为值。之前，只有 `None` 会指示应禁用 thick mode。

    参考：[#9295](https://www.sqlalchemy.org/trac/ticket/9295)

### orm

+   **[orm] [usecase]**

    为了适应 SQLAlchemy 2.0 中 ORM Declarative 使用的列排序变化，已添加了一个新参数 `mapped_column.sort_order`，可以用于控制 ORM 定义的表中列的顺序，用于常见用例，例如带有应首先出现在表中的主键列的混合的情况。ORM Declarative Applies Column Orders Differently; Control behavior using sort_order 中的更改说明描述了默认的顺序更改行为（这是所有 SQLAlchemy 2.0 发行版的一部分）以及在使用 mixins 和多个类时使用 `mapped_column.sort_order` 控制列排序的用法（2.0.4 中的新功能）。

    参见

    ORM Declarative Applies Column Orders Differently; Control behavior using sort_order

    参考：[#9297](https://www.sqlalchemy.org/trac/ticket/9297)

+   **[orm] [usecase]**

    `Session.refresh()` 方法现在将立即加载在`Session.refresh.attribute_names` 集合中显式命名的与关系绑定的属性，即使它当前连接到“select”加载器，这通常是一个“延迟”加载器，在刷新期间不会触发。 “延迟加载器”策略现在将检测到该操作明确是用户启动的 `Session.refresh()` 操作，该操作明确命名了此属性，并且将调用“immediateload”策略来实际发出 SQL 以加载属性。这对某些 asyncio 情况特别有帮助，其中必须强制加载未加载的惰性加载属性，而无需使用实际不支持的 asyncio 惰性加载属性模式。

    参考：[#9298](https://www.sqlalchemy.org/trac/ticket/9298)

+   **[orm] [bug] [regression]**

    修复了在版本 2.0.2 中引入的回归问题，原因是 [#9217](https://www.sqlalchemy.org/trac/ticket/9217)，其中使用 DML RETURNING 语句以及 `Select.from_statement()` 构造，在与使用了 `column_property()` 等表达式的 ORM 映射类一起使用时，会导致核心内部错误，该错误尝试按名称匹配表达式。该修复修复了核心问题，并且还调整了 [#9217](https://www.sqlalchemy.org/trac/ticket/9217) 中的修复，以便不会影响 DML RETURNING 的用例，因为它增加了不必要的开销。

    参考：[#9273](https://www.sqlalchemy.org/trac/ticket/9273)

+   **[orm] [bug]**

    将内部的 `EvaluatorCompiler` 模块标记为 ORM 的私有，并将其重命名为 `_EvaluatorCompiler`。对于可能依赖此功能的用户，名称 `EvaluatorCompiler` 仍然存在，但不支持此用法，并将在未来的版本中移除。

### orm 声明式

+   **[用例] [orm 声明式]**

    在`MappedAsDataclass` 类和 `registry.mapped_as_dataclass()` 方法中添加了新参数 `dataclasses_callable`，允许使用替代 Python `dataclasses.dataclass` 的可调用对象来生成数据类。这里的用例是使用 Pydantic 的 dataclass 函数。对版本 2.0.1 中添加的用于 [#9179](https://www.sqlalchemy.org/trac/ticket/9179) 的 mixin 支持进行了调整，以便将 mixin 的 `__annotations__` 集合重写为不包含 `Mapped` 容器，与映射类一样，这样 Pydantic 数据类的构造函数就不会暴露给未知类型。

    另请参阅

    与 Pydantic 等备选数据类提供程序集成

    参考：[#9266](https://www.sqlalchemy.org/trac/ticket/9266)

### sql

+   **[sql] [bug]**

    修复了元组值的元素类型会被硬编码为与比较元组相同类型的问题，当比较使用`ColumnOperators.in_()` 运算符时。这与通常确定二元表达式类型的方式不一致，通常情况下会首先考虑右侧的实际元素类型，然后再应用左侧的类型。

    参考：[#9313](https://www.sqlalchemy.org/trac/ticket/9313)

+   **[sql]**

    添加了公共属性 `Table.autoincrement_column`，该属性返回标识为自动增量的列。

    参考：[#9277](https://www.sqlalchemy.org/trac/ticket/9277)

### typing

+   **[typing] [usecase]**

    改进了对 Hybrid Attributes 扩展的类型支持，更新了所有文档以使用 ORM Annotated Declarative mappings，并添加了一个名为 `hybrid_property.inplace` 的新修饰符。此修饰符提供了一种**就地**修改 `hybrid_property` 状态的方式，这本质上是早期版本的 hybrids 在 SQLAlchemy 版本 1.2.0 之前所做的事情，它在此版本中取消了就地变异。现在，此就地变异已经恢复，以允许单个 hybrid 具有多个设置的方法，而无需将所有方法命名相同，并且无需仔细“链接”不同命名的方法以维护组成。例如 Mypy 和 Pyright 等类型工具不允许类上具有相同名称的方法，因此通过此更改，恢复了带有类型支持的简洁方法来设置 hybrids。

    另请参阅

    使用 inplace 创建符合 pep-484 的混合属性

    参考：[#9321](https://www.sqlalchemy.org/trac/ticket/9321)

### oracle

+   **[oracle] [bug]**

    调整了 python-oracledb 方言的`thick_mode`参数的行为，以正确接受`False`作为值。之前，只有`None`表示应禁用 thick mode。

    参考：[#9295](https://www.sqlalchemy.org/trac/ticket/9295)

## 2.0.3

发布日期：2023 年 2 月 9 日

### sql

+   **[sql] [bug] [regression]**

    由于 [#7744](https://www.sqlalchemy.org/trac/ticket/7744) 引入的严重回归问题，修复了 SQL 表达式在 2.0 系列中的组合，该问题改进了对重复使用同一运算符的 SQL 表达式的支持；表达式中包含的多个元素会丢失括号分组，除了前两个元素之外的表达式元素。

    参考：[#9271](https://www.sqlalchemy.org/trac/ticket/9271)

### typing

+   **[typing] [bug]**

    删除了 `typing.Self` 的解决方法，现在使用 [**PEP 673**](https://peps.python.org/pep-0673/) 大多数返回 `Self` 的方法。由于此更改，现在需要 `mypy>=1.0.0` 来类型检查 SQLAlchemy 代码。感谢 Yurii Karabas 提供的拉取请求。

    参考：[#9254](https://www.sqlalchemy.org/trac/ticket/9254)

### sql

+   **[sql] [bug] [regression]**

    修复了 2.0 系列中由于 [#7744](https://www.sqlalchemy.org/trac/ticket/7744) 导致的 SQL 表达式形成中的关键回归，该回归改进了对包含多个相同运算符的 SQL 表达式的支持；当表达式元素超过前两个元素时，括号分组将会丢失。

    参考：[#9271](https://www.sqlalchemy.org/trac/ticket/9271)

### 类型

+   **[typing] [错误]**

    删除了 `typing.Self` 的解决方法，现在大多数返回 `Self` 的方法使用 [**PEP 673**](https://peps.python.org/pep-0673/)。由于这一变更，现在需要 `mypy>=1.0.0` 来对 SQLAlchemy 代码进行类型检查。感谢 Yurii Karabas 提交的拉取请求。

    参考：[#9254](https://www.sqlalchemy.org/trac/ticket/9254)

## 2.0.2

发布日期：2023 年 2 月 6 日

### orm

+   **[orm] [用例]**

    添加了新的事件钩子 `MapperEvents.after_mapper_constructed()`，它提供了一个事件钩子，可以在 `Mapper` 对象完全构建完成后但在调用 `registry.configure()` 前发生。这允许根据 `Mapper` 的初始配置创建其他映射和表结构的代码，这也与声明性配置集成。以前，当使用声明性时，在类创建过程中创建了 `Mapper` 对象时，没有记录文档的方式在这一点运行代码。该变更立即受益于自定义映射方案，例如 带有历史表的版本控制 示例，该示例根据映射类的创建生成其他映射和表。

    参考：[#9220](https://www.sqlalchemy.org/trac/ticket/9220)

+   **[orm] [用例]**

    不常用的 `Mapper.iterate_properties` 属性和 `Mapper.get_property()` 方法，主要用于内部，不再隐式调用 `registry.configure()` 过程。对这些方法的公共访问极为罕见，而拥有 `registry.configure()` 的唯一好处将是允许这些集合中存在“backref”属性。为了支持新的 `MapperEvents.after_mapper_constructed()` 事件，现在可以遍历和访问内部的 `MapperProperty` 对象，而不触发映射器本身的隐式配置。

    更为公开的遍历所有映射器属性的路径，`Mapper.attrs` 集合等，仍将隐式调用 `registry.configure()` 步骤，从而使反向引用属性可用。

    在所有情况下，都可以直接调用 `registry.configure()`。

    参考：[#9220](https://www.sqlalchemy.org/trac/ticket/9220)

+   **[orm] [bug] [回归]**

    修复了由 [#8705](https://www.sqlalchemy.org/trac/ticket/8705) 引起的不常见的 ORM 继承问题，在某些情况下，继承自本地表和继承表的列组的映射器，会在 `column_property()` 下发出警告，指示同名属性被隐式合并。

    参考：[#9232](https://www.sqlalchemy.org/trac/ticket/9232)

+   **[orm] [bug] [回归]**

    修复了使用 `Mapper.version_id_col` 功能时的回归，当使用普通的 Python 递增列时，SQLite 和其他不支持“rowcount”与“RETURNING”的数据库将无法正常工作，因为对于这些列，将假定使用“RETURNING”，尽管实际情况并非如此。

    参考：[#9228](https://www.sqlalchemy.org/trac/ticket/9228)

+   **[orm] [bug] [回归]**

    修复了在 ORM 上下文中使用`Select.from_statement()`时的回归，其中仅基于名称匹配列到 SQL 标签的功能在不是完全文本的 ORM 语句上被禁用。这将阻止具有列名标签的任意 SQL 表达式与要加载的实体匹配，而以前在 1.4 和之前的系列中可以工作，因此已恢复了以前的行为。

    参考：[#9217](https://www.sqlalchemy.org/trac/ticket/9217)

### ORM 声明式

+   **[错误] [orm 声明式]**

    修复了由对[#9171](https://www.sqlalchemy.org/trac/ticket/9171)的修复引起的回归，该修复本身正在修复涉及从`DeclarativeBase`扩展的类上的`__init__()`机制的回归。更改使得如果类上没有直接`__init__()`方法，则`__init__()`将应用于用户定义的基类。已调整为仅在用户定义的基类的层次结构中没有其他类具有`__init__()`方法时才应用`__init__()`。这再次允许基于`DeclarativeBase`的用户定义基类包含自身包含自定义`__init__()`方法的混入。

    参考：[#9249](https://www.sqlalchemy.org/trac/ticket/9249)

+   **[错误] [orm 声明式]**

    修复了与 2.0.1 中新增的对混入支持相关的 ORM 声明式数据类映射问题，该支持是通过[#9179](https://www.sqlalchemy.org/trac/ticket/9179)添加的，其中在某些情况下使用混入加上 ORM 继承会导致字段被错误分类，从而导致像`init=False`这样的字段级数据类参数丢失。

    参考：[#9226](https://www.sqlalchemy.org/trac/ticket/9226)

+   **[错误] [orm 声明式]**

    修复了 ORM 声明式映射，允许在使用`mapped_column()`时在`__mapper_args__`中指定`Mapper.primary_key`参数。尽管这种用法直接在 2.0 文档中，但`Mapper`在这种情况下不接受`mapped_column()`构造。这个功能已经适用于`Mapper.version_id_col`和`Mapper.polymorphic_on`参数。

    作为这一变更的一部分，可以在非映射混合类上指定`__mapper_args__`属性，而无需使用`declared_attr()`，包括引用本地存在于混合类上的`Column`或`mapped_column()`对象的`"primary_key"`条目；Declarative 还将这些列转换为特定映射类的正确列。这对于`Mapper.version_id_col`和`Mapper.polymorphic_on`参数已经可以正常工作。此外，`"primary_key"`中的元素可以表示为现有映射属性的字符串名称。

    参考：[#9240](https://www.sqlalchemy.org/trac/ticket/9240)

+   **[bug] [orm declarative]**

    如果映射尝试在同一类层次结构中混合使用`MappedAsDataclass`和`registry.mapped_as_dataclass()`，则会显式引发错误，因为这会导致数据类函数在错误的时间应用于映射类，从而在映射过程中产生错误。

    参考：[#9211](https://www.sqlalchemy.org/trac/ticket/9211)

### examples

+   **[examples] [bug]**

    重新设计了带有历史表的版本控制，以适配版本 2.0，同时改进了此示例的整体工作方式，使用了更新的 API，包括新添加的钩子`MapperEvents.after_mapper_constructed()`。

    参考：[#9220](https://www.sqlalchemy.org/trac/ticket/9220)

### sql

+   **[sql] [usecase]**

    添加了一套全新的 SQL 位运算符，用于在适当的数据值（如整数、位字符串等）上执行数据库端的位运算表达式。感谢 Yegor Statkevich 的拉取请求。

    另请参阅

    位运算符

    参考：[#8780](https://www.sqlalchemy.org/trac/ticket/8780)

### asyncio

+   **[asyncio] [bug]**

    修复了由于修复了 [#8419](https://www.sqlalchemy.org/trac/ticket/8419) 而导致的回归，该回归导致 asyncpg 连接在连接未被显式返回到连接池的情况下（即未显式调用事务 `rollback()`）并且被 Python 垃圾收集拦截时，正常重置并返回到池中，如果垃圾收集操作在 asyncio 事件循环之外调用，则会导致大量堆栈跟踪活动被转储到日志和标准输出。

    恢复了正确的行为，即所有由于未被显式返回到连接池而被垃圾收集的 asyncio 连接都会被分离并丢弃，同时发出警告，而不是返回到池中，因为它们无法可靠地重置。对于 asyncpg 连接，将使用 asyncpg 特定的 `terminate()` 方法更优雅地结束连接，而不仅仅是将其丢弃。

    此更改包括了一个希望对调试 asyncio 应用程序有用的小行为更改，其中在 asyncio 连接意外垃圾收集的情况下发出的警告已经通过将其移到 `finally:` 块之外并放入 `try/except` 块中稍微更积极地发出，无论 detach/termination 操作成功与否都会无条件发出。它还将导致应用程序或测试套件将 Python 警告提升为异常，从而将此警告视为完整的异常引发，而以前不可能使此警告实际传播为异常。需要在过渡期容忍此警告的应用程序和测试套件应调整 Python 警告过滤器，以允许这些警告不引发异常。

    传统同步连接的行为保持不变，即垃圾收集的连接将继续正常返回到池中，而不会发出警告。这可能会在将来的主要版本发布中进行更改，至少会发出类似于 asyncio 驱动程序发出的警告，因为池化连接被垃圾收集拦截而没有正确返回到池中是使用错误。

    参考文献：[#9237](https://www.sqlalchemy.org/trac/ticket/9237)

### mysql

+   **[mysql] [bug] [regression]**

    修复了由问题 [#9058](https://www.sqlalchemy.org/trac/ticket/9058) 引起的回归，该问题调整了 MySQL 方言的 `has_table()`，再次使用“DESCRIBE”，在使用不存在的模式名称时，MySQL 版本 8 引发的特定错误代码是意外的，并且无法解释为布尔结果。

    参考文献：[#9251](https://www.sqlalchemy.org/trac/ticket/9251)

+   **[mysql] [bug]**

    在使用 `Insert.on_duplicate_key_update()` 时，为 MySQL 8 新的 `AS <name> ON DUPLICATE KEY` 语法添加了支持，这是新版本的 MySQL 8 所必需的，因为先前使用 `VALUES()` 的语法现在在这些版本中会发出弃用警告。服务器版本检测用于确定是使用传统的 MariaDB / MySQL < 8 `VALUES()` 语法，还是使用较新的 MySQL 8 必需的语法。感谢 Caspar Wylie 提供的拉取请求。

    参考资料：[#8626](https://www.sqlalchemy.org/trac/ticket/8626)

### sqlite

+   **[sqlite] [bug]**

    修复了 SQLite 方言的 `has_table()` 函数，以便在包含不存在的模式名称的查询中正确报告 False；之前，会引发数据库错误。

    参考资料：[#9251](https://www.sqlalchemy.org/trac/ticket/9251)

### orm

+   **[orm] [usecase]**

    添加了新的事件钩子 `MapperEvents.after_mapper_constructed()`，它提供了一个事件钩子，在完全构建了 `Mapper` 对象之后，但在调用 `registry.configure()` 之前发生。这允许基于初始配置创建其他映射和表结构的代码，这也与 Declarative 配置集成。之前，在使用 Declarative 时，`Mapper` 对象是在类创建过程中创建的，没有记录文档化的方法在此时运行代码。此更改立即有利于自定义映射方案，例如带有历史表的版本控制示例中生成额外映射和表的代码。

    参考资料：[#9220](https://www.sqlalchemy.org/trac/ticket/9220)

+   **[orm] [usecase]**

    很少使用的`Mapper.iterate_properties`属性和`Mapper.get_property()`方法，主要用于内部，现在不再隐式调用`registry.configure()`过程。对这些方法的公共访问极其罕见，而具有`registry.configure()`的唯一好处将是允许这些集合中存在“backref”属性。为了支持新的`MapperEvents.after_mapper_constructed()`事件，现在可以迭代和访问内部`MapperProperty`对象，而不会触发隐式的 mapper 配置。

    更公共的遍历所有 mapper 属性的路径，`Mapper.attrs` 集合等，仍然会隐式调用 `registry.configure()` 步骤，从而使 backref 属性可用。

    在所有情况下，`registry.configure()` 始终可供直接调用。

    参考文献：[#9220](https://www.sqlalchemy.org/trac/ticket/9220)

+   **[orm] [bug] [ression]**

    修复了由 [#8705](https://www.sqlalchemy.org/trac/ticket/8705) 引起的模糊的 ORM 继承问题，其中某些从本地表和继承表一起指示列组的继承映射器的情景，在 `column_property()` 下仍然会警告同名属性被隐式组合的问题。

    参考文献：[#9232](https://www.sqlalchemy.org/trac/ticket/9232)

+   **[orm] [bug] [regression]**

    修复了使用`Mapper.version_id_col`特性时的退化情况，当使用普通的 Python 端递增列时，该特性无法在 SQLite 和其他不支持“返回行数”的数据库中正常工作，因为会假设对于这些列使用“返回”，尽管实际上并非如此。

    参考文献：[#9228](https://www.sqlalchemy.org/trac/ticket/9228)

+   **[orm] [bug] [regression]**

    修复了在 ORM 上下文中使用 `Select.from_statement()` 时的回归问题，其中仅基于名称的列与 SQL 标签的匹配被禁用，以前完全不是文本的 ORM 语句。 这将阻止具有基于列名标签的任意 SQL 表达式与要加载的实体匹配，这在 1.4 和以前的系列中以前可以工作，因此已恢复了先前的行为。

    参考：[#9217](https://www.sqlalchemy.org/trac/ticket/9217)

### orm declarative

+   **[bug] [orm declarative]**

    由于对 [#9171](https://www.sqlalchemy.org/trac/ticket/9171) 修复引起的回归问题进行了修复，该问题本身修复了涉及从`DeclarativeBase` 派生的类的`__init__()` 机制的回归问题。 更改使得如果类上没有直接 `__init__()` 方法，则 `__init__()` 将应用于用户定义的基类。 现已调整为仅在用户定义的基类的层次结构中没有其他类具有 `__init__()` 方法时才应用 `__init__()`。 这再次允许基于`DeclarativeBase` 的用户定义的基类包括包含自定义 `__init__()` 方法的 mixins。

    参考：[#9249](https://www.sqlalchemy.org/trac/ticket/9249)

+   **[bug] [orm declarative]**

    修复了 ORM Declarative Dataclass 映射中的问题，与 2.0.1 中新增的对 mixins 的支持相关，通过 [#9179](https://www.sqlalchemy.org/trac/ticket/9179) 添加，其中使用 mixins 加上 ORM 继承的组合在某些情况下会错误地分类字段，导致字段级别的 dataclass 参数（如 `init=False`）丢失。

    参考：[#9226](https://www.sqlalchemy.org/trac/ticket/9226)

+   **[bug] [orm declarative]**

    修复了 ORM Declarative 映射，以允许在使用 `mapped_column()` 时在 `__mapper_args__` 中指定 `Mapper.primary_key` 参数。 尽管此用法直接在 2.0 文档中，但 `Mapper` 在此上下文中不接受 `mapped_column()` 构造。 此功能已经适用于 `Mapper.version_id_col` 和 `Mapper.polymorphic_on` 参数。

    作为此更改的一部分，可以在非映射的混合类上指定 `__mapper_args__` 属性，而无需使用 `declared_attr()`，其中包括引用了本地存在的 `Column` 或 `mapped_column()` 对象的 `"primary_key"` 条目；声明式还将这些列转换为特定映射类的正确列。这在`Mapper.version_id_col` 和 `Mapper.polymorphic_on` 参数中已经起作用。此外，`"primary_key"` 中的元素可以表示为现有映射属性的字符串名称。

    参考资料：[#9240](https://www.sqlalchemy.org/trac/ticket/9240)

+   **[错误] [orm 声明]**

    如果映射尝试在同一类层次结构中混合使用 `MappedAsDataclass` 和 `registry.mapped_as_dataclass()`，则会引发明确的错误，因为这会导致在错误的时间应用数据类函数到映射的类上，从而在映射过程中导致错误。

    参考资料：[#9211](https://www.sqlalchemy.org/trac/ticket/9211)

### 示例

+   **[示例] [错误]**

    重新设计了带历史表的版本化，使其适用于版本 2.0，同时改进了此示例的整体工作，以使用更新的 API，包括新添加的钩子 `MapperEvents.after_mapper_constructed()`。

    参考资料：[#9220](https://www.sqlalchemy.org/trac/ticket/9220)

### sql

+   **[sql] [用例]**

    添加了一套完整的新的 SQL 位运算符，用于对适当的数据值（如整数、位字符串等）执行数据库端的位运算表达式。拉取请求由 Yegor Statkevich 提供。

    另请参阅

    位运算符

    参考资料：[#8780](https://www.sqlalchemy.org/trac/ticket/8780)

### asyncio

+   **[asyncio] [错误]**

    修复了由修复[#8419](https://www.sqlalchemy.org/trac/ticket/8419)引起的回归，该修复导致 asyncpg 连接被重置（即调用事务`rollback()`）并正常返回到池中，在连接未显式返回到连接池而是被 Python 垃圾回收拦截的情况下发生这种情况，如果垃圾回收操作在 asyncio 事件循环之外被调用，将导致大量的堆栈跟踪活动被转储到日志和标准输出中。

    恢复了正确的行为，即所有由于未显式返回到连接池而被垃圾回收的 asyncio 连接都会从池中分离并被丢弃，并发出警告，而不是返回到池中，因为它们不能可靠地重置。对于 asyncpg 连接，在这个过程中，将使用 asyncpg 特定的`terminate()`方法更优雅地结束连接，而不是简单地丢弃它。

    这个改变包括一个小的行为改变，希望对调试 asyncio 应用程序有用，在 asyncio 连接意外被垃圾回收时发出的警告已经通过将其移出`try/except`块并放入`finally:`块稍微更加激进，无论分离/终止操作是否成功，它都会无条件地发出。它还将导致将 Python 警告提升为异常的应用程序或测试套件将其视为完整的异常引发，而以前这个警告不可能作为异常传播。需要在此期间容忍此警告的应用程序和测试套件应调整 Python 警告过滤器，以允许这些警告不引发异常。

    传统同步连接的行为保持不变，即垃圾回收的连接会正常返回到池中，而不会发出警告。这在未来的主要版本中可能会发生变化，至少会发出与 asyncio 驱动程序发出的类似警告，因为对于池化连接被垃圾回收而没有被正确返回到池中是一种使用错误。

    参考：[#9237](https://www.sqlalchemy.org/trac/ticket/9237)

### mysql

+   **[mysql] [bug] [regression]**

    修复了由问题[#9058](https://www.sqlalchemy.org/trac/ticket/9058)引起的回归，该问题调整了 MySQL 方言的`has_table()`以再次使用“DESCRIBE”，在使用不存在的模式名称时 MySQL 版本 8 引发的特定错误代码是意外的，并且无法解释为布尔结果。

    参考：[#9251](https://www.sqlalchemy.org/trac/ticket/9251)

+   **[mysql] [bug]**

    当使用`Insert.on_duplicate_key_update()`时，为 MySQL 8 的新`AS <name> ON DUPLICATE KEY`语法添加了支持，这在使用新版本的 MySQL 8 时是必需的，因为先前的语法使用`VALUES()`现在会针对这些版本发出弃用警告。服务器版本检测用于确定是否应使用传统的 MariaDB / MySQL < 8 `VALUES()`语法，还是应使用新的 MySQL 8 所需的语法。由 Caspar Wylie 提供的拉取请求。

    参考：[#8626](https://www.sqlalchemy.org/trac/ticket/8626)

### sqlite

+   **[sqlite] [bug]**

    修复了 SQLite 方言的`has_table()`函数，使其对于包含不存在的架构的非空架构的查询正确报告为 False；以前会引发数据库错误。

    参考：[#9251](https://www.sqlalchemy.org/trac/ticket/9251)

## 2.0.1

发布日期：2023 年 2 月 1 日

### orm

+   **[orm] [bug] [regression]**

    修复了使用具有复合外键的连接表继承的 ORM 模型遇到映射器内部错误的回归问题。

    参考：[#9164](https://www.sqlalchemy.org/trac/ticket/9164)

+   **[orm] [bug]**

    当从基类链接策略选项到另一个子类的属性时，当应该使用`of_type()`时，改进了错误报告。以前，当使用`Load.options()`时，消息会缺乏提示性的详细信息，即应该使用`of_type()`，而直接链接选项时则不是这种情况。即使使用`Load.options()`，提示性的详细信息现在也会发出。

    参考：[#9182](https://www.sqlalchemy.org/trac/ticket/9182)

### orm 声明式

+   **[bug] [orm 声明式]**

    添加了对[**PEP 484**](https://peps.python.org/pep-0484/) `NewType`的支持，以及在`registry.type_annotation_map`中以及在`Mapped`结构内部使用。这些类型现在将与当前的自定义类型子类以相同的方式运行；它们必须明确出现在`registry.type_annotation_map`中以进行映射。

    参考：[#9175](https://www.sqlalchemy.org/trac/ticket/9175)

+   **[bug] [orm 声明式]**

    在使用`MappedAsDataclass`超类时，现在将对属于此类的子类的所有类进行`@dataclasses.dataclass`函数处理，无论它们是否实际映射，以便在映射的子类转换为数据类时使用在层次结构中声明的非 ORM 字段。此行为适用于通过`__abstract__ = True`映射的中间类以及用户定义的声明基类本身，假设`MappedAsDataclass`作为这些类的超类存在。

    这允许使用超类上的非映射属性，例如`InitVar`声明，而无需显式在每个非映射类上运行`@dataclasses.dataclass`装饰器。新行为被认为是正确的，因为这是使用超类指示数据类行为时[**PEP 681**](https://peps.python.org/pep-0681/)实现预期的行为。

    参考：[#9179](https://www.sqlalchemy.org/trac/ticket/9179)

+   **[bug] [orm 声明式]**

    添加对[**PEP 586**](https://peps.python.org/pep-0586/) `Literal[]`的支持，以及在`Mapped`结构中的使用。要使用这些自定义类型，它们必须明确出现在`registry.type_annotation_map`中进行映射。感谢 Frederik Aalund 的拉取请求。

    作为这一变化的一部分，对`registry.type_annotation_map`中的`Enum`的支持已经扩展，包括支持由字符串值组成的`Literal[]`类型的使用，除了`enum.Enum`数据类型之外。如果在`Mapped[]`中使用的`Literal[]`数据类型未在`registry.type_annotation_map`中与特定数据类型关联，那么默认将使用`Enum`。

    另请参阅

    在类型映射中使用 Python Enum 或 pep-586 Literal 类型](../orm/declarative_tables.html#orm-declarative-mapped-column-enums)

    参考：[#9187](https://www.sqlalchemy.org/trac/ticket/9187)

+   **[bug] [orm 声明式]**

    修复了在`registry.type_annotation_map`中使用`Enum`时的问题，其中如果按照文档中所述覆盖为将此参数设置为 False，则`Enum.native_enum`参数将不会正确复制到映射列数据类型。

    参考：[#9200](https://www.sqlalchemy.org/trac/ticket/9200)

+   **[bug] [orm declarative] [regression]**

    修复了`DeclarativeBase`类中的回归问题，其中注册表的默认构造函数不会应用于基类本身，这与先前的`declarative_base()`构造方式不同。这将阻止具有自己`__init__()`方法的映射类调用`super().__init__()`以访问注册表的默认构造函数并自动填充属性，而是调用`object.__init__()`，这将导致任何参数引发`TypeError`。

    参考：[#9171](https://www.sqlalchemy.org/trac/ticket/9171)

+   **[bug] [orm declarative]**

    改进了用于解释[**PEP 593**](https://peps.python.org/pep-0593/) `Annotated`类型的规则集，当与 Annotated Declarative 映射一起使用时，内部类型将在所有情况下被检查是否为“Optional”，这将被添加到设置列是否为“nullable”的标准中；如果`Annotated`容器中的类型是可选的（或与`None`联合），则如果没有显式的`mapped_column.nullable`参数覆盖它，则列将被视为可为空。

    参考：[#9177](https://www.sqlalchemy.org/trac/ticket/9177)

### sql

+   **[sql] [bug]**

    更正了版本 2.0.0 中发布的[#7664](https://www.sqlalchemy.org/trac/ticket/7664)的修复，以便还包括无意中在此修复中遗漏的`DropSchema`，允许在没有方言的情况下进行字符串化。这两个构造的修复已经回溯到 1.4.47 版本的 1.4 系列。

    参考：[#7664](https://www.sqlalchemy.org/trac/ticket/7664)

+   **[sql] [bug] [regression]**

    修复了与新的“insertmanyvalues”功能实现相关的回归问题，在这种情况下，在 CTE 中引用 `insert()` 的情况下会出现内部 `TypeError`；针对此用例进行了额外的修复，例如在使用“insertmanyvalues”时使用异步 pg 时的位置方言。

    引用：[#9173](https://www.sqlalchemy.org/trac/ticket/9173)

### typing

+   **[typing] [bug]**

    对 `Select.with_for_update.of` 打开了类型标注，也接受表格和映射类参数，因为在 MySQL 方言中似乎可用。

    引用：[#9174](https://www.sqlalchemy.org/trac/ticket/9174)

+   **[typing] [bug]**

    修复了 limit/offset 方法的类型标注，包括 `Select.limit()`, `Select.offset()`, `Query.limit()`, `Query.offset()`，允许 `None`，这是“取消”当前 limit/offset 的文档化 API。

    引用：[#9183](https://www.sqlalchemy.org/trac/ticket/9183)

+   **[typing] [bug]**

    修复了 `mapped_column()` 对象作为 `Mapped` 类型无法被接受在模式约束中，如 `ForeignKey`, `UniqueConstraint` 或 `Index` 的类型标注问题。

    引用：[#9170](https://www.sqlalchemy.org/trac/ticket/9170)

+   **[typing] [bug]**

    修复了 `ColumnElement.cast()` 的类型标注，现在接受 `Type[TypeEngine[T]]` 和 `TypeEngine[T]` 两者；之前只接受 `TypeEngine[T]`。感谢 Yurii Karabas 的拉取请求。

    引用：[#9156](https://www.sqlalchemy.org/trac/ticket/9156)

### orm

+   **[orm] [bug] [regression]**

    修复了使用联接表继承和复合外键的 ORM 模型会在映射器内部遇到内部错误的回归问题。

    引用：[#9164](https://www.sqlalchemy.org/trac/ticket/9164)

+   **[orm] [bug]**

    当将基类的策略选项链接到另一个子类的属性，需要使用 `of_type()` 时，改进了链接策略选项时的错误报告。以前，当使用 `Load.options()` 时，消息缺乏关于应该使用 `of_type()` 的详细信息，而直接链接选项时不是这种情况。现在即使使用 `Load.options()`，也会发出详细的信息。

    参考：[#9182](https://www.sqlalchemy.org/trac/ticket/9182)

### ORM 声明式

+   **[错误] [ORM 声明式]**

    已添加对 [**PEP 484**](https://peps.python.org/pep-0484/) 中 `NewType` 的支持，可在 `registry.type_annotation_map` 和 `Mapped` 结构中使用。这些类型将与当前自定义类型的子类相同地行为；它们必须明确出现在 `registry.type_annotation_map` 中才能被映射。

    参考：[#9175](https://www.sqlalchemy.org/trac/ticket/9175)

+   **[错误] [ORM 声明式]**

    当使用 `MappedAsDataclass` 超类时，层次结构中所有的子类都将通过 `@dataclasses.dataclass` 函数运行，无论它们是否实际上被映射，以便在映射的子类转换为数据类时使用在层次结构中声明的非 ORM 字段。该行为既适用于使用 `__abstract__ = True` 映射的中介类，也适用于用户定义的声明基类本身，假设 `MappedAsDataclass` 作为这些类的超类。

    这允许在超类上使用非映射属性，例如 `InitVar` 声明，而无需在每个非映射类上显式运行 `@dataclasses.dataclass` 装饰器。新的行为被认为是正确的，因为这是使用超类指示数据类行为时 [**PEP 681**](https://peps.python.org/pep-0681/) 实现所期望的。

    参考：[#9179](https://www.sqlalchemy.org/trac/ticket/9179)

+   **[错误] [ORM 声明式]**

    已添加对[**PEP 586**](https://peps.python.org/pep-0586/) `Literal[]`的支持，可在`registry.type_annotation_map`以及`Mapped`结构中使用。要使用此类自定义类型，它们必须明确出现在`registry.type_annotation_map`中以进行映射。感谢 Frederik Aalund 提供的拉取请求。

    作为此更改的一部分，已扩展了`registry.type_annotation_map`中对`Enum`的支持，以包括对由字符串值组成的`Literal[]`类型的支持，以及对`enum.Enum`数据类型的支持。如果在`Mapped[]`中使用了未在`registry.type_annotation_map`中链接到特定数据类型的`Literal[]`数据类型，则将默认使用`Enum`。

    参见

    在类型映射中使用 Python 枚举或 pep-586 字面类型

    参考：[#9187](https://www.sqlalchemy.org/trac/ticket/9187)

+   **[错误] [orm 声明式]**

    修复了在`registry.type_annotation_map`中使用`Enum`时出现的问题，在其中，如果覆盖了`Enum.native_enum`参数，则不会正确地将该参数复制到映射列数据类型中，如文档中所述，如果将此参数设置为 False。 

    参考：[#9200](https://www.sqlalchemy.org/trac/ticket/9200)

+   **[错误] [orm 声明式] [回归]**

    修复了`DeclarativeBase`类中的回归，其中注册表的默认构造函数不会应用于基类本身，这与以前的`declarative_base()`构造方式不同。这将阻止具有自己的`__init__()`方法的映射类调用`super().__init__()`以访问注册表的默认构造函数并自动填充属性，而是会触发`object.__init__()`，这将在任何参数上引发`TypeError`。

    参考：[#9171](https://www.sqlalchemy.org/trac/ticket/9171)

+   **[错误] [orm 声明]**

    改进了用于解释[**PEP 593**](https://peps.python.org/pep-0593/) `Annotated`类型的规则集，当与 Annotated Declarative 映射一起使用时，内部类型将在所有情况下被检查是否为“Optional”，这将被添加到设置列是否“可为空”的标准中；如果`Annotated`容器内的类型是可选的（或与`None`联合），则如果没有显式的`mapped_column.nullable`参数覆盖它，列将被视为可为空。

    参考：[#9177](https://www.sqlalchemy.org/trac/ticket/9177)

### sql

+   **[sql] [错误]**

    修正了版本 2.0.0 中发布的对[#7664](https://www.sqlalchemy.org/trac/ticket/7664)的修复，还包括了`DropSchema`，这在修复中被意外遗漏，允许在没有方言的情况下进行字符串化。这两个构造的修复已经回溯到了 1.4.47 版本的 1.4 系列。

    参考：[#7664](https://www.sqlalchemy.org/trac/ticket/7664)

+   **[sql] [错误] [回归]**

    修复了与新“insertmanyvalues”功能实现相关的回归，其中在另一个通过 CTE 引用`insert()`的`insert()`内部会发生内部`TypeError`的情况；为这种情况进行了额外的修复，例如在使用“insertmanyvalues”时使用位置方言如 asyncpg 时。

    参考：[#9173](https://www.sqlalchemy.org/trac/ticket/9173)

### 类型化

+   **[输入] [错误]**

    打开了对`Select.with_for_update.of`的类型化，也接受表和映射类参数，就像 MySQL 方言中可用的那样。

    参考：[#9174](https://www.sqlalchemy.org/trac/ticket/9174)

+   **[输入] [错误]**

    修复了包括`Select.limit()`、`Select.offset()`、`Query.limit()`、`Query.offset()`在内的限制/偏移方法的类型化，允许`None`，这是“取消”当前限制/偏移的记录 API。

    参考：[#9183](https://www.sqlalchemy.org/trac/ticket/9183)

+   **[输入] [错误]**

    修复了 `mapped_column()` 对象在类型为 `Mapped` 时不会被接受到架构约束中的类型问题，例如 `ForeignKey`、`UniqueConstraint` 或 `Index`。

    参考：[#9170](https://www.sqlalchemy.org/trac/ticket/9170)

+   **[类型] [错误]**

    修复了 `ColumnElement.cast()` 的类型定义，现在接受 `Type[TypeEngine[T]]` 和 `TypeEngine[T]`，之前只接受 `TypeEngine[T]`。感谢 Yurii Karabas 提供的拉取请求。

    参考：[#9156](https://www.sqlalchemy.org/trac/ticket/9156)

## 2.0.0

发布日期：2023 年 1 月 26 日

### ORM

+   **[ORM] [错误]**

    改进了在配置映射器或刷新过程中发出的警告的通知方式，这些警告通常作为不同操作的一部分调用，以在可能不明显相关的操作中添加附加上下文到警告的消息中，指示其中一个操作作为警告来源，而这些操作可能不明显相关。

    参考：[#7305](https://www.sqlalchemy.org/trac/ticket/7305)

### ORM 扩展

+   **[特性] [ORM 扩展]**

    在水平分片 API `set_shard_id` 中添加了新选项，该选项设置要查询的有效分片标识符，用于主查询以及所有二级加载器，包括关系急切加载器以及关系和列惰性加载器。

    参考：[#7226](https://www.sqlalchemy.org/trac/ticket/7226)

+   **[用例] [ORM 扩展]**

    为 `AutomapBase` 添加了新功能，用于在可能具有重叠名称的多个模式中自动加载类，方法是提供一个 `AutomapBase.prepare.modulename_for_table` 参数，该参数允许自定义新生成的类的 `__module__` 属性，以及一个新的集合 `AutomapBase.by_module`，它存储基于 `__module__` 属性的类的点分隔的模块名称命名空间。

    另外，现在可以任意次调用`AutomapBase.prepare()` 方法，无论是否启用了反射；在每次调用时，只会处理未映射的新增表。以前，每次都需要显式调用`MetaData.reflect()` 方法。

    另请参阅

    从多个模式生成映射 - 同时演示了两种技术的使用。

    参考资料：[#5145](https://www.sqlalchemy.org/trac/ticket/5145)

### sql

+   **[sql] [错误]**

    修复了`CreateSchema` DDL 结构的字符串化，当没有方言时会导致`AttributeError`。更新：请注意，这个修复未能考虑到`DropSchema`；版本 2.0.1 中的后续修复解决了这个问题。对于这两个元素的修复已回溯到 1.4.47。

    参考资料：[#7664](https://www.sqlalchemy.org/trac/ticket/7664)

### 输入

+   **[输入] [错误]**

    为从`func` 命名空间可用的内置通用函数添加了类型，这些函数接受特定的参数集并返回特定的类型，例如 `count`、`current_timestamp` 等。

    参考资料：[#9129](https://www.sqlalchemy.org/trac/ticket/9129)

+   **[输入] [错误]**

    更正了“lambda 表达式”传递的类型，以便 mypy、pyright 等可以接受纯 lambda 而不会有任何关于参数类型的错误。另外，为 lambda 表达式的更多公共 API 实现了输入，并确保`StatementLambdaElement` 属于`Executable` 层次结构，因此被`Connection.execute()` 接受为已接受的类型。

    参考资料：[#9120](https://www.sqlalchemy.org/trac/ticket/9120)

+   **[输入] [错误]**

    `ColumnOperators.in_()` 和 `ColumnOperators.not_in()` 方法的类型已更改为包括`Iterable[Any]`，而不是`Sequence[Any]`，以获得更灵活的参数类型。

    参考资料：[#9122](https://www.sqlalchemy.org/trac/ticket/9122)

+   **[输入] [错误]**

    从类型的角度来看，`or_()`和`and_()`要求第一个参数必须存在，但是这些函数仍然接受零参数，这将在运行时发出弃用警告。还添加了类型支持，以仅将固定的文字`False`用于`or_()`和`True`用于`and_()`作为第一个参数，但是文档现在指示在这些情况下发送`false()`和`true()`构造作为更明确的方法。

    参考：[#9123](https://www.sqlalchemy.org/trac/ticket/9123)

+   **[typing] [错误]**

    修复了在迭代`Query`对象时出现的类型问题。

    参考：[#9125](https://www.sqlalchemy.org/trac/ticket/9125)

+   **[typing] [错误]**

    修复了使用`Result`作为上下文管理器时出现的类型问题，未保留对象类型，而是在所有情况下指示`Result`而不是特定的`Result`子类型。感谢 Martin Baláž提供的拉取请求。

    参考：[#9136](https://www.sqlalchemy.org/trac/ticket/9136)

+   **[typing] [错误]**

    修复了使用`relationship.remote_side`和类似参数时出现的问题，通过将声明为`Mapped`的注释对象传递给类型检查器时，类型检查器将不接受。

    参考：[#9150](https://www.sqlalchemy.org/trac/ticket/9150)

+   **[typing] [错误]**

    添加了对诸如`isnot()`、`notin_()`等旧操作符的类型支持，这些操作符以前引用了新操作符，但它们本身并未被类型化。

    参考：[#9148](https://www.sqlalchemy.org/trac/ticket/9148)

### postgresql

+   **[postgresql] [错误]**

    添加了对 asyncpg 方言的支持，以在可用时返回`cursor.rowcount`值。虽然这不是`cursor.rowcount`的典型用法，但其他 PostgreSQL 方言通常提供此值。感谢 Michael Gorven 提供的拉取请求。

    此更改也**被回溯到**：1.4.47

    参考：[#9048](https://www.sqlalchemy.org/trac/ticket/9048)

### mysql

+   **[mysql] [用例]**

    添加了对 MySQL 索引反射的支持，以正确反映先前被忽略的 `mysql_length` 字典。

    此更改还**回溯**至：1.4.47

    参考：[#9047](https://www.sqlalchemy.org/trac/ticket/9047)

### MSSQL

+   **[MSSQL] [错误] [回归]**

    新增的 MSSQL 方言中的注释反射和渲染功能，添加在 [#7844](https://www.sqlalchemy.org/trac/ticket/7844)，如果无法确定是否正在使用不受支持的后端（如 Azure Synapse），则现在将默认禁用；此后端不支持表和列注释，也不支持用于生成它们以及反映它们的 SQL Server 例程。在方言中添加了一个新参数 `supports_comments`，默认值为 `None`，表示应自动检测注释支持。当设置为 `True` 或 `False` 时，注释支持将被无条件地启用或禁用。

    另请参阅

    DDL 注释支持

    参考：[#9142](https://www.sqlalchemy.org/trac/ticket/9142)

### ORM

+   **[ORM] [错误]**

    改进了在配置映射器或刷新过程中发出的警告的通知，这些警告通常作为不同操作的一部分调用，以在可能不明显相关的操作中添加附加上下文到指示警告来源的消息。

    参考：[#7305](https://www.sqlalchemy.org/trac/ticket/7305)

### ORM 扩展

+   **[功能] [ORM 扩展]**

    在水平分片 API `set_shard_id` 中添加了一个新选项，该选项设置用于查询的有效分片标识符，用于主查询以及所有辅助加载器，包括关系及其懒加载器和列懒加载器。

    参考：[#7226](https://www.sqlalchemy.org/trac/ticket/7226)

+   **[用例] [ORM 扩展]**

    为 `AutomapBase` 添加了一个新功能，用于跨多个模式自动加载具有重叠名称的类，通过提供一个 `AutomapBase.prepare.modulename_for_table` 参数，该参数允许定制新生成的类的 `__module__` 属性，以及一个新集合 `AutomapBase.by_module`，它存储基于 `__module__` 属性的类的点分隔的模块名称命名空间。

    此外，现在可以任意次调用`AutomapBase.prepare()`方法，无论是否启用了反射；每次调用只会处理以前未映射的新添加的表。以前，需要显式调用`MetaData.reflect()`方法。

    另请参阅

    从多个模式生成映射 - 同时演示了两种技术的使用。

    参考：[#5145](https://www.sqlalchemy.org/trac/ticket/5145)

### sql

+   **[sql] [bug]**

    修复了`CreateSchema` DDL 构造的字符串化，当没有方言时会导致`AttributeError`错误。更新：请注意，此修复未能适应`DropSchema`；版本 2.0.1 中的后续修复解决了这种情况。这两个元素的修复已经回溯到 1.4.47。

    参考：[#7664](https://www.sqlalchemy.org/trac/ticket/7664)

### 输入

+   **[typing] [bug]**

    为从`func`命名空间中可用的内置通用函数添加了类型，这些函数接受特定的参数并返回特定的类型，例如`count`，`current_timestamp`等。

    参考：[#9129](https://www.sqlalchemy.org/trac/ticket/9129)

+   **[typing] [bug]**

    修正了“lambda 语句”传递的类型，以便 mypy、pyright 等可以接受普通 lambda 而不会出现关于参数类型的任何错误。此外，为 lambda 语句的公共 API 实现了更多的类型，并确保`StatementLambdaElement`是`Executable`层次结构的一部分，因此它被`Connection.execute()`接受。

    参考：[#9120](https://www.sqlalchemy.org/trac/ticket/9120)

+   **[typing] [bug]**

    `ColumnOperators.in_()`和`ColumnOperators.not_in()`方法的类型被定义为包含`Iterable[Any]`而不是`Sequence[Any]`，以便在参数类型上具有更大的灵活性。

    参考：[#9122](https://www.sqlalchemy.org/trac/ticket/9122)

+   **[typing] [bug]**

    从打字的角度看，`or_()`和`and_()`需要第一个参数存在，但这些函数仍然接受零个参数，这将在运行时发出弃用警告。还添加了打字以支持将固定的字面量`False`发送给`or_()`和`True`发送给`and_()`作为唯一的第一个参数，但文档现在指示在这些情况下发送`false()`和`true()`构造作为更明确的方法。

    参考：[#9123](https://www.sqlalchemy.org/trac/ticket/9123)

+   **[打字] [错误]**

    修复了在`Query`对象上迭代时类型不正确的打字问题。

    参考：[#9125](https://www.sqlalchemy.org/trac/ticket/9125)

+   **[打字] [错误]**

    修复了使用`Result`作为上下文管理器时对象类型未被保留的打字问题，始终在所有情况下指示`Result`而不是特定的`Result`子类型。感谢 Martin Baláž的拉取请求。

    参考：[#9136](https://www.sqlalchemy.org/trac/ticket/9136)

+   **[打字] [错误]**

    修复了使用`relationship.remote_side`和类似参数时的问题，传递一个注释的声明对象类型为`Mapped`，类型检查器不会接受。

    参考：[#9150](https://www.sqlalchemy.org/trac/ticket/9150)

+   **[打字] [错误]**

    为诸如`isnot()`、`notin_()`等旧操作符添加了打字，这些操作符以前引用了新操作符，但它们本身没有类型。

    参考：[#9148](https://www.sqlalchemy.org/trac/ticket/9148)

### postgresql

+   **[postgresql] [错误]**

    添加了对 asyncpg 方言的支持，以在可用时为 SELECT 语句返回`cursor.rowcount`值。虽然这不是`cursor.rowcount`的典型用法，但其他 PostgreSQL 方言通常提供此值。感谢 Michael Gorven 的拉取请求。

    此更改也**回溯**到：1.4.47

    参考：[#9048](https://www.sqlalchemy.org/trac/ticket/9048)

### mysql

+   **[mysql] [用例]**

    Added support to MySQL index reflection to correctly reflect the `mysql_length` dictionary, which previously was being ignored.  

    This change is also **backported** to: 1.4.47  

    References: [#9047](https://www.sqlalchemy.org/trac/ticket/9047)  

### mssql  

+   **[mssql] [bug] [regression]**  

    The newly added comment reflection and rendering capability of the MSSQL dialect, added in [#7844](https://www.sqlalchemy.org/trac/ticket/7844), will now be disabled by default if it cannot be determined that an unsupported backend such as Azure Synapse may be in use; this backend does not support table and column comments and does not support the SQL Server routines in use to generate them as well as to reflect them. A new parameter `supports_comments` is added to the dialect which defaults to `None`, indicating that comment support should be auto-detected. When set to `True` or `False`, the comment support is either enabled or disabled unconditionally.  

    另请参阅  

    DDL Comment Support  

    References: [#9142](https://www.sqlalchemy.org/trac/ticket/9142)  

## 2.0.0rc3  

Released: January 18, 2023  

### orm  

+   **[orm] [feature]**  

    Added a new parameter to `Mapper` called `Mapper.polymorphic_abstract`. The purpose of this directive is so that the ORM will not consider the class to be instantiated or loaded directly, only subclasses. The actual effect is that the `Mapper` will prevent direct instantiation of instances of the class and will expect that the class does not have a distinct polymorphic identity configured.  

    In practice, the class that is mapped with `Mapper.polymorphic_abstract` can be used as the target of a `relationship()` as well as be used in queries; subclasses must of course include polymorphic identities in their mappings.  

    The new parameter is automatically applied to classes that subclass the `AbstractConcreteBase` class, as this class is not intended to be instantiated.  

    另请参阅  

    Building Deeper Hierarchies with polymorphic_abstract  

    References: [#9060](https://www.sqlalchemy.org/trac/ticket/9060)  

+   **[orm] [bug]**  

    修复了使用 pep-593 `Annotated` 类型时的问题，在其中自身包含泛型普通容器或 `collections.abc` 类型（例如 `list`、`dict`、`collections.abc.Sequence` 等）的 `registry.type_annotation_map`，当 ORM 尝试解释 `Annotated` 实例时，会产生内部错误。

    参考：[#9099](https://www.sqlalchemy.org/trac/ticket/9099)

+   **[orm] [bug]**

    当将 `relationship()` 映射到抽象容器类型（例如 `Mapped[Sequence[B]]`）时，未提供必需的 `relationship.container_class` 参数时，添加了一个错误消息，此前，抽象容器会尝试在后续步骤中实例化并失败。

    参考：[#9100](https://www.sqlalchemy.org/trac/ticket/9100)

### sql

+   **[sql] [bug]**

    修复了一个错误/回归，即在 2.0 中，使用与 `Update.values()` 方法中的列相同的名称与 `bindparam()` 一起使用时，有时会无声地无法遵守 SQL 表达式，将表达式替换为同名的新参数，并丢弃 SQL 表达式的任何其他元素，例如 SQL 函数等。具体情况将是针对 ORM 实体而不是纯 `Table` 实例构造的语句，但如果使用 `Session` 或 `Connection` 调用语句，则会发生。

    问题的一部分存在于 2.0 和 1.4 中，并且已回溯到 1.4。

    此更改还**回溯**到：1.4.47

    参考：[#9075](https://www.sqlalchemy.org/trac/ticket/9075)

### typing

+   **[typing] [bug]**

    修复了 `sqlalchemy.ext.hybrid` 扩展中对用户定义方法进行更有效类型注释的问题。现在的类型注释使用了 [**PEP 612**](https://peps.python.org/pep-0612/) 功能，最近版本的 Mypy 已支持，以维护 `hybrid_method` 的参数签名。混合方法的返回值在诸如 `Select.where()` 这样的上下文中被接受为 SQL 表达式，同时仍支持 SQL 方法。

    参考：[#9096](https://www.sqlalchemy.org/trac/ticket/9096)

### mypy

+   **[mypy] [bug]**

    调整了 mypy 插件，以适应在使用 SQLAlchemy 1.4 时可能对 issue #236 sqlalchemy2-stubs 进行的一些潜在更改。这些更改在 SQLAlchemy 2.0 中保持同步。这些更改也向后兼容旧版本的 sqlalchemy2-stubs。

    此更改也被**回溯**到：1.4.47

+   **[mypy] [bug]**

    修复了 mypy 插件中的崩溃，该崩溃可能会在 1.4 和 2.0 版本中发生，如果在表达式中使用了一个装饰器，该装饰器具有超过两个组件的引用（例如 `@Backend.mapper_registry.mapped`）。现在会忽略这种情况；在使用插件时，装饰器表达式需要是两个组件（即 `@reg.mapped`）。

    此更改也被**回溯**到：1.4.47

    参考：[#9102](https://www.sqlalchemy.org/trac/ticket/9102)

### postgresql

+   **[postgresql] [bug]**

    修复了 psycopg3 在版本 3.1.8 中更改了一个 API 调用，要求先前未强制执行的特定对象类型，导致 psycopg3 方言的连接中断的回归。

    参考：[#9106](https://www.sqlalchemy.org/trac/ticket/9106)

### oracle

+   **[oracle] [usecase]**

    添加了对 Oracle SQL 类型 `TIMESTAMP WITH LOCAL TIME ZONE` 的支持，使用了新添加的 Oracle 特定的 `TIMESTAMP` 数据类型。

    参考：[#9086](https://www.sqlalchemy.org/trac/ticket/9086)

### orm

+   **[orm] [feature]**

    向 `Mapper` 添加了一个名为 `Mapper.polymorphic_abstract` 的新参数。此指令的目的是，ORM 不会考虑该类直接实例化或加载，只会考虑子类。实际效果是，`Mapper` 将阻止直接实例化类的实例，并期望该类没有配置独特的多态标识。

    实际上，与 `Mapper.polymorphic_abstract` 映射的类也可以作为 `relationship()` 的目标，以及在查询中使用；当然，子类必须在其映射中包含多态标识。

    新参数自动应用于子类化 `AbstractConcreteBase` 类的类，因为该类不打算被实例化。

    另请参阅

    使用 polymorphic_abstract 构建更深的层次结构

    参考：[#9060](https://www.sqlalchemy.org/trac/ticket/9060)

+   **[orm] [bug]**

    修复了在 `registry.type_annotation_map` 中使用 pep-593 `Annotated` 类型时的问题，其中包含一个通用的普通容器或 `collections.abc` 类型（例如 `list`、`dict`、`collections.abc.Sequence` 等）作为目标类型会导致 ORM 尝试解释 `Annotated` 实例时产生内部错误的问题。

    参考：[#9099](https://www.sqlalchemy.org/trac/ticket/9099)

+   **[orm] [bug]**

    当将 `relationship()` 映射到抽象容器类型（例如 `Mapped[Sequence[B]]`）时，添加了一个错误消息，但未提供必要的 `relationship.container_class` 参数，当类型为抽象时，这是必要的。以前，抽象容器将尝试在后续步骤中实例化并失败。

    参考：[#9100](https://www.sqlalchemy.org/trac/ticket/9100)

### sql

+   **[sql] [bug]**

    修复了一个 bug/回归，即在`Update.values()`的方法中使用与列相同名称的`bindparam()`以及在 2.0 版本中仅在`Insert.values()`的方法中使用相同名称的参数，在某些情况下会静默失败地遵循参数所呈现的 SQL 表达式，将表达式替换为相同名称的新参数并丢弃 SQL 表达式的任何其他元素，如 SQL 函数等。具体情况是针对 ORM 实体构建的语句而不是普通的`Table`实例，但如果使用`Session`或`Connection`调用语句，则会发生。

    `Update`部分的问题在 2.0 和 1.4 版本中均存在，并已回溯到 1.4。

    此更改也**回溯**到：1.4.47

    参考：[#9075](https://www.sqlalchemy.org/trac/ticket/9075)

### 输入

+   **[typing] [bug]**

    对`sqlalchemy.ext.hybrid`扩展中用户定义方法的类型注释进行修复，以更有效地类型化。现在的类型注释使用[**PEP 612**](https://peps.python.org/pep-0612/)的特性，最近的 Mypy 版本支持，以保持`hybrid_method`的参数签名。混合方法的返回值在诸如`Select.where()`等上下文中被接受为 SQL 表达式，同时仍支持 SQL 方法。

    参考：[#9096](https://www.sqlalchemy.org/trac/ticket/9096)

### mypy

+   **[mypy] [bug]**

    对 mypy 插件进行了调整，以适应使用 SQLAlchemy 1.4 时可能进行的对 issue #236 sqlalchemy2-stubs 的一些潜在更改。这些更改在 SQLAlchemy 2.0 内保持同步。这些更改也与旧版本的 sqlalchemy2-stubs 兼容。

    此更改也**回溯**到：1.4.47

+   **[mypy] [bug]**

    修复了在 mypy 插件中的崩溃，如果在表达式中引用了超过两个组件的装饰器（例如`@Backend.mapper_registry.mapped`），则在 1.4 和 2.0 版本上都可能发生。现在这种情况被忽略；在使用插件时，装饰器表达式需要是两个组件（即`@reg.mapped`）。

    此更改还被**回溯**到了：1.4.47

    参考：[#9102](https://www.sqlalchemy.org/trac/ticket/9102)

### postgresql

+   **[postgresql] [bug]**

    修复了回归，在版本 3.1.8 中 psycopg3 更改了一个 API 调用，以期望之前没有强制执行的特定对象类型，从而破坏了 psycopg3 方言的连通性。

    参考：[#9106](https://www.sqlalchemy.org/trac/ticket/9106)

### oracle

+   **[oracle] [usecase]**

    增加了对 Oracle SQL 类型`TIMESTAMP WITH LOCAL TIME ZONE`的支持，使用了新添加的 Oracle 特定`TIMESTAMP`数据类型。

    参考：[#9086](https://www.sqlalchemy.org/trac/ticket/9086)

## 2.0.0rc2

发布日期：2023 年 1 月 9 日

### orm

+   **[orm] [bug]**

    修复了在 2.0 中添加了一个过于严格的 ORM 映射规则的问题，该规则阻止了对`TableClause`对象的映射，例如在 wiki 上使用的视图配方中使用的那些。

    参考：[#9071](https://www.sqlalchemy.org/trac/ticket/9071)

### typing

+   **[typing] [bug]**

    在 PEP 681 的被接受版本中，Data Class Transforms 参数`field_descriptors`被重命名为`field_specifiers`。

    参考：[#9067](https://www.sqlalchemy.org/trac/ticket/9067)

### postgresql

+   **[postgresql] [json]**

    实现了缺失的`JSONB`操作：

    +   使用`Comparator.path_match()`来处理`@@`。

    +   使用`Comparator.path_exists()`来处理`@?`。

    +   使用`Comparator.delete_path()`来处理`#-`。

    由 Guilherme Martins Crocetti 提供的拉取请求。

    参考：[#7147](https://www.sqlalchemy.org/trac/ticket/7147)

### mysql

+   **[mysql] [bug]**

    恢复了`Inspector.has_table()`方法的行为，用于报告 MySQL / MariaDB 的临时表。这是所有其他包含的方言的当前行为，但在 1.4 中对 MySQL 进行了删除，因为不再使用 DESCRIBE 命令；在这个版本或任何之前的版本中，没有记录支持通过`Inspector.has_table()`方法报告临时表，因此之前的行为是未定义的。

    由于 SQLAlchemy 2.0 通过`Inspector.has_table()`正式支持临时表状态，MySQL / MariaDB 方言已经恢复为在 SQLAlchemy 1.3 系列和以前使用“DESCRIBE”语句的方式，并且添加了测试支持以包括 MySQL / MariaDB 的这种行为。由于`Connection`处理事务的简化，1.4 试图改进的以前发生的 ROLLBACK 问题在 SQLAlchemy 2.0 中不适用。

    DESCRIBE 是必要的，因为特别是 MariaDB 没有一致可用的公共信息模式来报告除 DESCRIBE/SHOW COLUMNS 之外的临时表，这依赖于抛出错误以报告无结果。

    参考：[#9058](https://www.sqlalchemy.org/trac/ticket/9058)

### oracle

+   **[oracle] [bug]**

    支持外键约束的用例，其中本地列标记为“不可见”。在反射时，创建`ForeignKeyConstraint`时通常生成的错误被禁用，并且与已经存在类似问题的`Index`一样，跳过约束并发出警告。

    参考：[#9059](https://www.sqlalchemy.org/trac/ticket/9059)

### orm

+   **[orm] [bug]**

    修复了在 2.0 版本中添加的一个过于严格的 ORM 映射规则，导致无法对`TableClause`对象进行映射，比如在维基百科上使用的视图配方中使用的对象。

    参考：[#9071](https://www.sqlalchemy.org/trac/ticket/9071)

### typing

+   **[typing] [bug]**

    在 PEP 681 的已接受版本中，Data Class Transforms 参数`field_descriptors`被重命名为`field_specifiers`。

    参考：[#9067](https://www.sqlalchemy.org/trac/ticket/9067)

### postgresql

+   **[postgresql] [json]**

    实现了缺失的`JSONB`操作：

    +   `@@` 使用`Comparator.path_match()`

    +   `@?` 使用`Comparator.path_exists()`

    +   `#-` 使用`Comparator.delete_path()`

    感谢 Guilherme Martins Crocetti 提供的拉取请求。

    References: [#7147](https://www.sqlalchemy.org/trac/ticket/7147)

### mysql

+   **[mysql] [bug]**

    恢复了`Inspector.has_table()`的行为，以报告 MySQL / MariaDB 的临时表。这是所有其他包含的方言的当前行为，但是在 1.4 中删除了 MySQL 的行为，因为不再使用 DESCRIBE 命令;在此版本或任何先前版本中，`Inspector.has_table()`方法没有记录支持临时表的报告，因此之前的行为未定义。

    由于 SQLAlchemy 2.0 已经增加了对临时表状态的正式支持，因此将 MySQL / MariaDB 方言恢复为使用“DESCRIBE”语句，就像 SQLAlchemy 1.3 系列和以前一样，并且添加了测试支持以包含 MySQL / MariaDB 用于此行为。由于`Connection`处理事务的简化，在 SQLAlchemy 2.0 中，1.4 试图改进的 ROLLBACK 的先前问题不适用。

    DESCRIBE 是必需的，因为特别是 MariaDB 没有任何一致可用的公共信息模式，以便报告除 DESCRIBE/SHOW COLUMNS 之外的临时表，这些信息模式依赖于抛出错误以报告没有结果。

    References: [#9058](https://www.sqlalchemy.org/trac/ticket/9058)

### oracle

+   **[oracle] [bug]**

    支持外键约束的用例，其中本地列标记为“不可见”。当反射时禁用通常在创建`ForeignKeyConstraint`时生成的检查目标列的错误，并且与发生类似问题的`Index`一样，以相同的警告跳过约束。

    参考：[#9059](https://www.sqlalchemy.org/trac/ticket/9059)

## 2.0.0rc1

发布日期：2022 年 12 月 28 日

### 一般

+   **[general] [bug]**

    修复了基本兼容模块调用`platform.architecture()`来检测某些系统属性的回归，结果是针对一些情况下不可用的系统级`file`调用进行了过度广泛的系统调用，包括一些安全环境配置。

    此更改也被**回溯**到：1.4.46

    参考：[#8995](https://www.sqlalchemy.org/trac/ticket/8995)

### orm

+   **[orm] [feature]**

    为`Mapper.eager_defaults`参数“auto”添加了一个新的默认值，该参数将在工作单元刷新时自动获取表默认值，如果方言支持正在运行的 INSERT 的 RETURNING，以及 insertmanyvalues 可用。对于服务器端 UPDATE 默认值的急切获取是非常罕见的，只有在`Mapper.eager_defaults`设置为`True`时才会发生，因为对于 UPDATE 语句，没有批量 RETURNING 形式。

    参考：[#8889](https://www.sqlalchemy.org/trac/ticket/8889)

+   **[orm] [usecase]**

    调整了`Session`的可扩展性，以及对`ShardedSession`扩展的更新：

    +   `Session.get()`现在接受`Session.get.bind_arguments`，尤其在使用水平分片扩展时可能会有用。

    +   `Session.get_bind()`接受任意关键字参数，这有助于开发使用覆盖此方法的`Session`类的代码，以及额外的参数。

    +   添加了一个新的 ORM 执行选项`identity_token`，可用于直接影响将与新加载的 ORM 对象关联的“身份令牌”。这个令牌是分片方法（主要是`ShardedSession`，但也可以在其他情况下使用）在不同“分片”之间分离对象标识的方式。

        请参阅

        身份令牌

    +   `SessionEvents.do_orm_execute()` 事件钩子现在可以用于影响所有 ORM 相关的选项，包括 `autoflush`、`populate_existing` 和 `yield_per`；在事件钩子被调用后，这些选项会被重新使用，然后才被执行。之前，像 `autoflush` 这样的选项在此时已经被评估过了。这种模式还支持新的 `identity_token` 选项，并且水平分片扩展现在正在使用它。

    +   `ShardedSession` 类替换了 `ShardedSession.id_chooser` 钩子，使用一个新的钩子 `ShardedSession.identity_chooser`，它不再依赖于传统的 `Query` 对象。虽然 `ShardedSession.id_chooser` 仍然可以在警告弃用的情况下替代 `ShardedSession.identity_chooser`。

    参考：[#7837](https://www.sqlalchemy.org/trac/ticket/7837)

+   **[orm] [用例]**

    “将外部事务加入到一个会话中”的行为已经进行了修订和改进，允许对 `Session` 如何适应已经建立了事务和可能已经建立了保存点的传入 `Connection` 进行显式控制。新参数 `Session.join_transaction_mode` 包含了一系列选项值，可以以几种方式适应现有的事务，最重要的是允许 `Session` 以完全事务化的方式使用独占的保存点，同时在所有情况下保持外部启动的事务未提交且处于活动状态，允许测试套件回滚在测试中发生的所有更改。

    另外，修改了`Session.close()` 方法，以完全关闭可能仍然存在的保存点，这还允许“外部事务”配方在`Session`没有显式结束其自己的 SAVEPOINT 事务时继续进行而不会出现警告。

    另见

    会话的新事务连接模式

    参考：[#9015](https://www.sqlalchemy.org/trac/ticket/9015)

+   **[orm] [usecase]**

    删除了在检测到非`Mapped[]`注释时必须在 Declarative Dataclass Mapped 类上使用`__allow_unmapped__` 属性的要求；以前，会引发一个意图支持传统 ORM 类型映射的错误消息，该错误消息还没有明确提到与 Dataclasses 特定相关的正确模式。如果使用了`registry.mapped_as_dataclass()` 或 `MappedAsDataclass`，则不再引发此错误消息。

    另见

    使用未映射的数据类字段

    参考：[#8973](https://www.sqlalchemy.org/trac/ticket/8973)

+   **[orm] [bug]**

    修复了用于 DML 语句（如`Update` 和 `Delete`）的内部 SQL 遍历中的问题，该问题会导致使用 ORM 更新/删除功能时出现一些潜在问题，其中包括使用 lambda 语句时出现的特定问题。

    此更改还已**回溯**到：1.4.46

    参考：[#9033](https://www.sqlalchemy.org/trac/ticket/9033)

+   **[orm] [bug]**

    修复了一个 bug，`Session.merge()`无法保留使用`relationship.viewonly`参数指示的关系属性的当前加载内容的情况，从而破坏了使用`Session.merge()`从缓存和其他类似技术中提取完全加载的对象的策略。在相关变更中，修复了一个问题，即一个包含已加载关系的对象，但仍然在映射上配置为 `lazy='raise'` 的对象，在传递给`Session.merge()`时会失败；在合并过程中暂停了“raise”检查，假定`Session.merge.load`参数保持其默认值为 `True`。

    总的来说，这是对 1.4 系列中引入的一个变更的行为调整，即 [#4994](https://www.sqlalchemy.org/trac/ticket/4994)，该变更将“merge”从默认应用于“viewonly”关系的级联集合中移除。由于“viewonly”关系在任何情况下都不会被持久化，允许它们的内容在“merge”期间传输不会影响目标对象的持久化行为。这使得`Session.merge()`能够正确地满足其中一个用例，即向`Session`添加在其他地方加载的对象，通常是为了从缓存中恢复。

    这个变更也被**反向移植**到：1.4.45

    参考：[#8862](https://www.sqlalchemy.org/trac/ticket/8862)

+   **[orm] [bug]**

    修复了`with_expression()`中的问题，在这个表达式中，由于列被引用自封闭的 SELECT，有些情况下 SQL 渲染不正确，即使表达式具有与使用`query_expression()`的属性匹配的标签名称，即使`query_expression()`没有默认表达式。目前，如果`query_expression()`确实具有默认表达式，该标签名称仍然用于该默认值，并且将继续忽略具有相同名称的其他标签。总的来说，这种情况相当棘手，因此可能需要进一步调整。

    这个改动也**回溯**到：1.4.45

    参考：[#8881](https://www.sqlalchemy.org/trac/ticket/8881)

+   **[orm] [bug]**

    如果在`relationship()`中使用的反向引用名称命名了目标类上已经分配给该名称的方法或属性，则会发出警告，因为反向引用声明将替换该属性。

    参考：[#4629](https://www.sqlalchemy.org/trac/ticket/4629)

+   **[orm] [bug]**

    一系列关于`Session.refresh()`的变更和改进。总的变化是，当要刷新关系绑定的属性时，对象的主键属性现在无条件地包含在刷新操作中，即使没有过期，甚至如果没有在刷新中指定。

    +   改进了`Session.refresh()`，以便如果启用了自动刷新（作为 `Session` 的默认设置），自动刷新会在刷新过程的较早阶段进行，以便应用待处理的主键更改而不会引发错误。以前，此自动刷新发生得太晚，SELECT 语句将不使用正确的键来定位行，并且会引发`InvalidRequestError`。

    +   当上述条件存在时，即对象上存在未刷新的主键更改，但未启用自动刷新时，`refresh()`方法现在明确禁止操作继续进行，并引发一个信息性的`InvalidRequestError`，要求先刷新待处理的主键更改。以前，这种用例简单地被破坏，无论如何都会引发`InvalidRequestError`。这个限制是为了安全地刷新主键属性，这在需要刷新具有关系绑定的次要预加载器的对象的情况下是必要的。无论是否实际上需要在刷新中使用 PK 列，此规则都适用于保持 API 行为一致，因为在任何情况下，刷新对象的某些属性而保留其他属性“待处理”是不寻常的。

    +   `Session.refresh()`方法已经得到增强，使得那些与`relationship()`绑定并链接到急加载器的属性，在所有情况下都将被刷新，即使传递了一个不包括父行上任何列的属性列表。这是在 1.4 中作为[#1763](https://www.sqlalchemy.org/trac/ticket/1763)的一部分首次实现的功能，允许急加载的关系绑定属性参与`Session.refresh()`操作。如果刷新操作没有指示要刷新父行上的任何列，则主键列仍将包括在刷新操作中，这允许加载继续到正常情况下指示的次要关系加载器。以前，对于这种情况会引发一个`InvalidRequestError`错误（[#8703](https://www.sqlalchemy.org/trac/ticket/8703))。

    +   修复了一个问题，即在调用`Session.refresh()`时，如果属性过期并且同时存在一个发射“次要”查询的急加载器（例如`selectinload()`），则会额外发出一个不必要的附加 SELECT。现在，由于主键属性已自动包含在刷新中，因此在关系加载器选择这些属性时不会有额外的加载。([#8997](https://www.sqlalchemy.org/trac/ticket/8997))

    +   修复了由 2.0.0b1 中引起的[#8126](https://www.sqlalchemy.org/trac/ticket/8126)引起的回归，其中`Session.refresh()`方法将在传递过期的列名以及与“次要”急加载器（如`selectinload()`）链接的关系绑定属性名称时导致`AttributeError`失败。 ([#8996](https://www.sqlalchemy.org/trac/ticket/8996))

    参考：[#8703](https://www.sqlalchemy.org/trac/ticket/8703), [#8996](https://www.sqlalchemy.org/trac/ticket/8996), [#8997](https://www.sqlalchemy.org/trac/ticket/8997)

+   **[orm] [bug]**

    改进了版本 1.4 中针对[#8456](https://www.sqlalchemy.org/trac/ticket/8456)的首次修复，该修复减少了在使用`Mapper.with_polymorphic`参数时用于呈现 ORM 查询的内部“多态适配器”的使用。这些非常复杂且容易出错的适配器现在仅在显式用户提供子查询用于`Mapper.with_polymorphic`的情况下使用，其中包括仅使用`polymorphic_union()`助手的具体继承映射的用例，以及使用别名子查询的传统用例，这在现代用例中不需要。

    对于使用内置多态加载方案的最常见的连接继承映射情况，包括那些使用`Mapper.polymorphic_load`参数设置为`inline`的情况，现在不再使用多态适配器。这对于查询的构造有着积极的性能影响，同时也极大地简化了内部查询渲染过程。

    具体针对的问题是允许`column_property()`引用标量子查询中的继承类，现在可以像可行的那样直观地工作。

    参考：[#8168](https://www.sqlalchemy.org/trac/ticket/8168)

### 引擎

+   **[engine] [错误]**

    修复了连接池中长期存在的竞争条件，在与 eventlet/gevent monkeypatching 方案一起使用 eventlet/gevent `Timeout`条件时可能会发生，在这种情况下，由于超时而中断的连接池检出将无法清理失败状态，导致底层连接记录和有时数据库连接本身“泄漏”，使池处于无效状态，无法访问条目。这个问题首次在 SQLAlchemy 1.2 中被识别和修复，用于[#4225](https://www.sqlalchemy.org/trac/ticket/4225)，然而在该修复中检测到的故障模式未能适应`BaseException`，而不是`Exception`，这阻止了 eventlet/gevent `Timeout`的捕获。此外，在初始池连接中还识别并加固了一个`BaseException` -> “清理失败连接”块，以适应此位置的相同条件。非常感谢 Github 用户@niklaus 为他们在识别和描述这个错综复杂问题中的顽强努力。

    此更改也被**回溯**到：1.4.46

    参考：[#8974](https://www.sqlalchemy.org/trac/ticket/8974)

+   **[engine] [错误]**

    修复了`Result.freeze()`方法无法用于使用`text()`或`Connection.exec_driver_sql()`的文本 SQL 的问题。

    此更改也被**回溯**到：1.4.45

    参考：[#8963](https://www.sqlalchemy.org/trac/ticket/8963)

### sql

+   **[sql] [用例]**

    当任何“文字绑定参数”渲染操作失败时，现在会抛出一个信息性的重新引发，指示值本身和正在使用的数据类型，以帮助调试在语句中渲染文字参数时发生的情况。

    此更改也被**回溯**到：1.4.45

    参考：[#8800](https://www.sqlalchemy.org/trac/ticket/8800)

+   **[sql] [错误]**

    修复了 lambda SQL 功能中的问题，其中文字值的计算类型不会考虑“与类型比较”的类型强制转换规则，导致 SQL 表达式缺乏类型信息，例如与`JSON`元素等的比较。

    此更改也被**回溯**到：1.4.46

    参考：[#9029](https://www.sqlalchemy.org/trac/ticket/9029)

+   **[sql] [错误]**

    修复了关于呈现的绑定参数的位置以及有时是身份的一系列问题，例如 SQLite、asyncpg、MySQL、Oracle 等使用的参数。一些编译的形式不会正确地保持参数的顺序，例如 PostgreSQL `regexp_replace()` 函数、首次在 [#4123](https://www.sqlalchemy.org/trac/ticket/4123) 中引入的 `CTE` 构造的“嵌套”功能，以及使用 Oracle 的 `FunctionElement.column_valued()` 方法形成的可选择表。

    此更改还被**反向移植**到：1.4.45

    参考资料：[#8827](https://www.sqlalchemy.org/trac/ticket/8827)

+   **[sql] [bug]**

    添加了测试支持，以确保 SQLAlchemy 中所有 `Compiler` 实现中的所有编译器 `visit_xyz()` 方法都接受一个 `**kw` 参数，以便所有编译器在所有情况下都接受额外的关键字参数。

    参考资料：[#8988](https://www.sqlalchemy.org/trac/ticket/8988)

+   **[sql] [bug]**

    `SQLCompiler.construct_params()` 方法以及 `SQLCompiler.params` 访问器现在将返回与使用 `render_postcompile` 参数编译的编译语句相对应的确切参数。以前，该方法返回的参数结构本身既不对应原始参数也不对应扩展参数。

    不再允许向使用 `render_postcompile` 构造的 `SQLCompiler` 传递新的参数字典；而是，为了为备选参数集制作新的 SQL 字符串和参数集，添加了一个新方法 `SQLCompiler.construct_expanded_state()`，该方法将使用 `ExpandedState` 容器为给定的参数集生成新的扩展形式，其中包括新的 SQL 语句和新的参数字典，以及位置参数元组。

    参考资料：[#6114](https://www.sqlalchemy.org/trac/ticket/6114)

+   **[sql] [bug]**

    为了适应具有不同字符转义需求的第三方方言，关于绑定参数的特殊字符的 SQLAlchemy“转义”（即用另一个字符替换）的系统已经被扩展为第三方方言可扩展，使用`SQLCompiler.bindname_escape_chars`字典，可以在任何`SQLCompiler`子类的类声明级别上进行覆盖。作为这一变化的一部分，还添加了点`"."`作为默认的“转义”字符。

    参考：[#8994](https://www.sqlalchemy.org/trac/ticket/8994)

### 输入

+   **[输入] [错误]**

    pep-484 输入已经完成了`sqlalchemy.ext.horizontal_shard`扩展以及`sqlalchemy.orm.events`模块的类型。感谢 Gleb Kisenkov 的努力。

    参考：[#6810](https://www.sqlalchemy.org/trac/ticket/6810)，[#9025](https://www.sqlalchemy.org/trac/ticket/9025)

### asyncio

+   **[asyncio] [错误]**

    从`AsyncResult`中删除了无效的`merge()`方法。这个方法从未起作用，并且错误地包含在`AsyncResult`中。

    此更改也**回溯**到：1.4.45

    参考：[#8952](https://www.sqlalchemy.org/trac/ticket/8952)

### postgresql

+   **[postgresql] [错误]**

    修复了一个错误，即 PostgreSQL `Insert.on_conflict_do_update.constraint`参数将接受一个`Index`对象，但不会将此索引展开为其各个索引表达式，而是在 ON CONFLICT ON CONSTRAINT 子句中呈现其名称，这不被 PostgreSQL 接受；“约束名”形式只接受唯一或排除约束名。该参数继续接受索引，但现在将其展开为其组成表达式以进行呈现。

    此更改也**回溯**到：1.4.46

    参考：[#9023](https://www.sqlalchemy.org/trac/ticket/9023)

+   **[postgresql] [错误]**

    调整了 PostgreSQL 方言在从表中反映列类型时的考虑方式，以适应可能从 PG `format_type()`函数返回 NULL 的替代后端。

    此更改也**回溯**到：1.4.45

    参考：[#8748](https://www.sqlalchemy.org/trac/ticket/8748)

+   **[postgresql] [错误]**

    增加了对 asyncpg 和 psycopg（仅适用于 SQLAlchemy 2.0）明确使用 PG 全文函数的支持，关于第一个参数的`REGCONFIG`类型转换，以前会错误地转换为 VARCHAR，导致这些方言上的失败依赖于明确的类型转换。这包括对`to_tsvector`、`to_tsquery`、`plainto_tsquery`、`phraseto_tsquery`、`websearch_to_tsquery`、`ts_headline`的支持，每个函数都会根据传递的参数数量确定第一个字符串参数是否应该被解释为 PostgreSQL 的`REGCONFIG`值；如果是，则使用新添加的类型对象`REGCONFIG`对参数进行显式类型化，然后在 SQL 表达式中显式转换。

    参考：[#8977](https://www.sqlalchemy.org/trac/ticket/8977)

+   **[postgresql] [bug]**

    修复了新修订的 PostgreSQL 范围类型（如`INT4RANGE`）无法设置为`TypeDecorator`自定义类型的 impl，而是引发了`TypeError`的回归问题。

    参考：[#9020](https://www.sqlalchemy.org/trac/ticket/9020)

+   **[postgresql] [bug]**

    当与不同类的实例进行比较时，`Range.__eq___()`现在会返回`NotImplemented`，而不是引发`AttributeError`异常。

    参考：[#8984](https://www.sqlalchemy.org/trac/ticket/8984)

### sqlite

+   **[sqlite] [usecase]**

    增加了对 SQLite 后端的支持，以反映可能存在于外键构造上的“DEFERRABLE”和“INITIALLY”关键字。感谢 Michael Gorven 的拉取请求。

    这个改变也**被回溯**到：1.4.45

    参考：[#8903](https://www.sqlalchemy.org/trac/ticket/8903)

+   **[sqlite] [usecase]**

    增加了对 SQLite 方言中包含在索引中的基于表达式的 WHERE 条件的反射支持，类似于 PostgreSQL 方言的方式。感谢 Tobias Pfeiffer 的拉取请求。

    这个改变也**被回溯**到：1.4.45

    参考：[#8804](https://www.sqlalchemy.org/trac/ticket/8804)

### oracle

+   **[oracle] [错误]**

    修复了 Oracle 编译器中 `FunctionElement.column_valued()` 语法错误的问题，导致 `COLUMN_VALUE` 名称没有正确限定源表。

    此更改也**被迁移**至：1.4.45

    参考：[#8945](https://www.sqlalchemy.org/trac/ticket/8945)

### 测试

+   **[测试] [错误]**

    修复了 tox.ini 文件中的问题，在 tox 4.0 系列对于 “passenv” 格式的更改导致 tox 无法正常工作，特别是在 tox 4.0.6 之后引发错误。

    此更改也**被迁移**至：1.4.46

+   **[测试] [错误]**

    为第三方方言添加了一个名为 `unusual_column_name_characters` 的新的排除规则，可以针对不支持具有不寻常字符的列名的第三方方言进行“关闭”，即使该名称已正确引用。

    此更改也**被迁移**至：1.4.46

    参考：[#9002](https://www.sqlalchemy.org/trac/ticket/9002)

### 通用

+   **[通用] [错误]**

    修复了基础兼容模块调用 `platform.architecture()` 以检测某些系统属性的回归，这会导致对于一些情况下不可用的系统级别 `file` 调用进行过度广泛的系统调用，包括在某些安全环境配置中。

    此更改也**被迁移**至：1.4.46

    参考：[#8995](https://www.sqlalchemy.org/trac/ticket/8995)

### orm

+   **[orm] [功能]**

    为 `Mapper.eager_defaults` 参数“auto”添加了一个新的默认值，“auto” 将在工作单元刷新期间自动获取表默认值，如果方言支持用于运行的 INSERT 的 RETURNING，并且可用 insertmanyvalues。对于服务端 UPDATE 默认值的急切获取，这是非常罕见的，仅在 `Mapper.eager_defaults` 设置为 `True` 时才会发生，因为没有 UPDATE 语句的批量 RETURNING 形式。

    参考：[#8889](https://www.sqlalchemy.org/trac/ticket/8889)

+   **[orm] [用例]**

    对 `Session` 在可扩展性方面的调整，以及对 `ShardedSession` 扩展的更新：

    +   `Session.get()` 现在接受 `Session.get.bind_arguments`，特别是在使用水平分片扩展时可能会很有用。

    +   `Session.get_bind()` 接受任意的关键字参数，这有助于开发使用重载了此方法并具有额外参数的 `Session` 类的代码。

    +   添加了一个新的 ORM 执行选项 `identity_token`，可直接影响将与新加载的 ORM 对象关联的“标识令牌”。该令牌是分片方法（主要是 `ShardedSession`，但也可用于其他情况）在不同“分片”中分隔对象标识的方式。

        另请参阅

        标识令牌

    +   `SessionEvents.do_orm_execute()` 事件钩子现在可用于影响所有与 ORM 相关的选项，包括 `autoflush`、`populate_existing` 和 `yield_per`；在事件钩子被调用后，这些选项会重新被消费，然后再执行。以前，像 `autoflush` 这样的选项在此时已经被评估过了。新的 `identity_token` 选项也在此模式中受支持，并且现在被水平分片扩展使用。

    +   `ShardedSession` 类用新的钩子 `ShardedSession.identity_chooser` 替换了 `ShardedSession.id_chooser` 钩子，不再依赖于传统的 `Query` 对象。在此情况下，`ShardedSession.id_chooser` 仍然被接受，并发出弃用警告。

    参考：[#7837](https://www.sqlalchemy.org/trac/ticket/7837)

+   **[orm] [用例]**

    “将外部事务加入到会话中”的行为已经修订和改进，允许对 `Session` 如何适应已经建立了事务和可能已经建立了保存点的传入 `Connection` 进行显式控制。新参数 `Session.join_transaction_mode` 包括一系列选项值，可以以几种方式适应现有事务，最重要的是允许 `Session` 以完全事务样式使用仅保存点，同时在所有情况下保持外部启动的事务未提交且活跃，允许测试套件在测试中回滚所有发生的更改。

    另外，修订了 `Session.close()` 方法以完全关闭可能仍然存在的保存点，这也允许“外部事务”方案在未显式结束其自己的 SAVEPOINT 事务的情况下进行而不会出现警告。

    另请参阅

    会话的新事务加入模式

    参考文献：[#9015](https://www.sqlalchemy.org/trac/ticket/9015)

+   **[orm] [用例]**

    当检测到非`Mapped[]`注释时，不再要求在 Declarative Dataclass Mapped 类上使用 `__allow_unmapped__` 属性；以前，如果使用 `registry.mapped_as_dataclass()` 或 `MappedAsDataclass`，就会触发一个旨在支持传统 ORM 类型映射的错误消息，而且此错误消息还未明确提到与 Dataclasses 特别相关的正确模式。如果使用 `registry.mapped_as_dataclass()` 或 `MappedAsDataclass`，则不再触发此错误消息。

    另请参阅

    使用非映射 Dataclass 字段

    参考文献：[#8973](https://www.sqlalchemy.org/trac/ticket/8973)

+   **[orm] [错误]** 

    修复了内部 SQL 遍历中关于 `Update` 和 `Delete` 等 DML 语句的问题，这些问题会导致一些潜在问题，其中之一是在使用 ORM 更新/删除功能时使用 lambda 表达式的特定问题。

    此更改还被**回溯**到：1.4.46

    参考：[#9033](https://www.sqlalchemy.org/trac/ticket/9033)

+   **[orm] [bug]**

    修复了一个 bug，`Session.merge()` 在指定了 `relationship.viewonly` 参数的关系属性的当前加载内容时会失败，从而破坏了使用 `Session.merge()` 从缓存和其他类似技术中提取完全加载的对象的策略。在相关变更中，修复了一个问题，即包含已加载关系的对象，而映射上仍配置为`lazy='raise'`时，当传递给 `Session.merge()` 时会失败；假定 `Session.merge.load` 参数保持默认值`True`的情况下，合并过程中的“raise”检查现在被挂起。

    总体而言，这是对 1.4 系列中引入的更改的行为调整，即 [#4994](https://www.sqlalchemy.org/trac/ticket/4994) 中的一项更改，将“merge”从默认应用于“viewonly”关系的级联集中删除。由于“viewonly”关系在任何情况下都不会持久化，因此允许它们的内容在“merge”期间转移不会影响目标对象的持久化行为。这使得 `Session.merge()` 正确地适应了其中一种用例，即将对象添加到其他地方加载的 `Session` 中，通常是为了从缓存中恢复。

    这个更改也被**回溯**到：1.4.45

    参考：[#8862](https://www.sqlalchemy.org/trac/ticket/8862)

+   **[orm] [bug]**

    修复了在 `with_expression()` 中的问题，其中由从封闭 SELECT 引用的列组成的表达式在某些情况下不会正确渲染 SQL，在表达式具有与使用 `query_expression()` 的属性匹配的标签名称的情况下，即使 `query_expression()` 没有默认表达式。目前，如果 `query_expression()` 确实有一个默认表达式，则该标签名称仍然用于该默认表达式，并且将继续忽略具有相同名称的附加标签。总体来说，这种情况相当棘手，因此可能需要进一步调整。

    此更改也**被后移**至：1.4.45

    引用：[#8881](https://www.sqlalchemy.org/trac/ticket/8881)

+   **[orm] [bug]**

    如果在 `relationship()` 中使用的反向引用名称指定了目标类上已经有方法或属性分配给该名称，则会发出警告，因为反向引用声明将替换该属性。

    引用：[#4629](https://www.sqlalchemy.org/trac/ticket/4629)

+   **[orm] [bug]**

    一系列关于 `Session.refresh()` 的变更和改进。总体变更是：当刷新与关系绑定的属性时，对象的主键属性现在无条件地包含在刷新操作中，即使未过期，即使未在刷新中指定。

    +   改进了 `Session.refresh()`，这样如果启用了自动刷新（默认情况下为 `Session`），则自动刷新会在刷新过程的较早阶段发生，以便应用挂起的主键更改而不引发错误。以前，这个自动刷新发生得太晚，而且 SELECT 语句将不使用正确的键来定位行，将会引发 `InvalidRequestError`。

    +   当存在上述条件时，即对象上存在未刷新的主键更改，但未启用自动刷新时，refresh() 方法现在明确禁止操作继续进行，并且会引发一个提示性的 `InvalidRequestError`，要求首先刷新挂起的主键更改。以前，这种用例仅仅是无法使用，并且无论如何都会引发 `InvalidRequestError`。此限制是为了安全地刷新主键属性，因为对于能够使用关系绑定的次要急切加载器刷新对象也是必要的。无论是否实际上需要在刷新中使用 PK 列，此规则都适用于保持 API 行为一致，因为在任何情况下都不太可能在刷新对象的同时保留其他属性的“挂起”状态。

    +   `Session.refresh()` 方法已经改进，以至于那些在映射时间或者通过最后使用的加载器选项与 `relationship()` 绑定并链接到急切加载器的属性，将在所有情况下刷新，即使传递的属性列表不包括父行上的任何列也是如此。这是在 [#1763](https://www.sqlalchemy.org/trac/ticket/1763) 中首次实现的功能的基础上构建的，在 1.4 中修复了非列属性的情况，允许急切加载的关系绑定属性参与 `Session.refresh()` 操作。如果刷新操作未指示刷新父行上的任何列，则仍将包括主键列在刷新操作中，这允许加载器按照通常的方式进行到所指示的次要关系加载器。以前，对于此条件，会引发一个 `InvalidRequestError` 错误（[#8703](https://www.sqlalchemy.org/trac/ticket/8703)）。

    +   修复了在 `Session.refresh()` 与过期属性的组合被调用时，在存在一个发出“secondary”查询的 eager loader（例如 `selectinload()`）的情况下，会发出不必要的额外 SELECT 语句的问题，如果主键属性也处于过期状态，则不会为这些属性额外加载。由于主键属性现在自动包含在刷新中，当关系加载程序选择它们时，这些属性不会产生额外的负载 ([#8997](https://www.sqlalchemy.org/trac/ticket/8997))

    +   修复了 2.0.0b1 版本中由 [#8126](https://www.sqlalchemy.org/trac/ticket/8126) 引起的回归问题，该问题是 `Session.refresh()` 方法将在传递了一个已过期的列名以及与“secondary” eagerloader 相关联的关系绑定属性的名称时失败，并且该 eagerloader 是例如 `selectinload()` 的“secondary” eagerloader ([#8996](https://www.sqlalchemy.org/trac/ticket/8996))

    参考：[#8703](https://www.sqlalchemy.org/trac/ticket/8703), [#8996](https://www.sqlalchemy.org/trac/ticket/8996), [#8997](https://www.sqlalchemy.org/trac/ticket/8997)

+   **[orm] [bug]**

    优化了 1.4 版本中首次修复的问题，该问题涉及减少了内部“多态适配器”的使用，该适配器用于在使用 `Mapper.with_polymorphic` 参数时呈现 ORM 查询。这些适配器非常复杂且容易出错，现在仅在使用显式用户提供的子查询的情况下使用这些适配器，其中包括仅适用于使用 `polymorphic_union()` 助手的具体继承映射的用例，以及使用联接继承映射的别名子查询的传统用例，后者在现代用例中不再需要。

    对于使用内置多态加载方案的联接继承映射的最常见情况，包括将 `Mapper.polymorphic_load` 参数设置为 `inline` 的情况，不再使用多态适配器。这既对查询的构造产生了积极的性能影响，也大大简化了内部查询呈现过程。

    具体的目标问题是允许`column_property()`引用标量子查询中的连接继承类，这个现在可以尽可能直观地工作了。

    参考：[#8168](https://www.sqlalchemy.org/trac/ticket/8168)

### engine

+   **[engine] [bug]**

    修复了连接池中长期存在的竞争条件，在与 eventlet/gevent 的 monkeypatching 方案一起使用时可能会发生，并且使用 eventlet/gevent 的`Timeout`条件，其中由于超时而中断的连接池检出将无法清理失败状态，导致底层连接记录和有时数据库连接本身“泄漏”，使连接池处于无效状态，其中的条目无法访问。这个问题首次在 SQLAlchemy 1.2 中被发现并修复了[#4225](https://www.sqlalchemy.org/trac/ticket/4225)，然而在那次修复中检测到的失败模式未能适应`BaseException`，而不是`Exception`，这阻止了 eventlet/gevent `Timeout`的捕获。此外，在初始连接池连接中还确定并加固了一个`BaseException` -> “清理失败连接”块，以适应在此位置中的相同条件。非常感谢 Github 用户@niklaus 在识别和描述这个复杂问题方面的顽强努力。

    这个改变也被**反向移植**到了：1.4.46

    参考：[#8974](https://www.sqlalchemy.org/trac/ticket/8974)

+   **[engine] [bug]**

    修复了`Result.freeze()`方法在使用`text()`或`Connection.exec_driver_sql()`时无法正常工作的问题。

    这个改变也被**反向移植**到了：1.4.45

    参考：[#8963](https://www.sqlalchemy.org/trac/ticket/8963)

### sql

+   **[sql] [用例]**

    在任何“文字绑定参数”渲染操作失败的情况下，现在会抛出一个信息性的重新引发，指示值本身和使用的数据类型，以帮助调试在语句中渲染文字参数时发生的情况。

    这个改变也被**反向移植**到了：1.4.45

    参考：[#8800](https://www.sqlalchemy.org/trac/ticket/8800)

+   **[sql] [bug]**

    修复了 lambda SQL 功能中的问题，其中文字值的计算类型不会考虑到“与类型进行比较的强制转换规则”，导致 SQL 表达式的缺乏类型信息，例如与`JSON`元素等进行比较。

    这个改变也被**反向移植**到了：1.4.46

    参考：[#9029](https://www.sqlalchemy.org/trac/ticket/9029)

+   **[sql] [bug]**

    修复了关于渲染绑定参数位置和有时身份的一系列问题，例如用于 SQLite、asyncpg、MySQL、Oracle 等的参数。一些编译形式无法正确维护参数的顺序，例如 PostgreSQL 的`regexp_replace()`函数，首次引入于[#4123](https://www.sqlalchemy.org/trac/ticket/4123)的`CTE`构造中的“嵌套”功能，以及使用 Oracle 的`FunctionElement.column_valued()`方法形成的可选择表。

    此更改也被**回溯**到：1.4.45

    参考：[#8827](https://www.sqlalchemy.org/trac/ticket/8827)

+   **[sql] [bug]**

    添加了测试支持，以确保 SQLAlchemy 中所有`Compiler`实现中的所有`visit_xyz()`方法都接受`**kw`参数，以便所有编译器在所有情况下都接受额外的关键字参数。

    参考：[#8988](https://www.sqlalchemy.org/trac/ticket/8988)

+   **[sql] [bug]**

    `SQLCompiler.construct_params()`方法以及`SQLCompiler.params`访问器现在将返回与使用`render_postcompile`参数编译的编译语句完全对应的参数。以前，该方法返回一个参数结构，它本身既不对应原始参数也不对应扩展参数。

    禁止向使用`render_postcompile`构造的`SQLCompiler`传递新的参数字典；相反，为了为另一组参数创建新的 SQL 字符串和参数集，添加了一个新方法`SQLCompiler.construct_expanded_state()`，它将为给定的参数集生成一个新的扩展形式，使用包含新 SQL 语句和新参数字典以及位置参数元组的`ExpandedState`容器。

    参考：[#6114](https://www.sqlalchemy.org/trac/ticket/6114)

+   **[sql] [bug]**

    为了适应具有不同字符转义需求的第三方方言，关于绑定参数的特殊字符的 SQLAlchemy “转义”（即，用另一个字符替换它）的系统已被扩展为第三方方言可扩展，使用 `SQLCompiler.bindname_escape_chars` 字典，该字典可以在任何 `SQLCompiler` 子类的类声明级别上进行覆盖。作为这个更改的一部分，还将点 `"."` 添加为默认的“转义”字符。

    参考：[#8994](https://www.sqlalchemy.org/trac/ticket/8994)

### typing

+   **[typing] [bug]**

    对于 `sqlalchemy.ext.horizontal_shard` 扩展以及 `sqlalchemy.orm.events` 模块完成了 pep-484 typing。感谢 Gleb Kisenkov 的努力。

    参考：[#6810](https://www.sqlalchemy.org/trac/ticket/6810)，[#9025](https://www.sqlalchemy.org/trac/ticket/9025)

### asyncio

+   **[asyncio] [bug]**

    从 `AsyncResult` 中删除了非功能性的 `merge()` 方法。该方法从未起作用，而且错误地包含在 `AsyncResult` 中。

    此更改还被**反向移植**到：1.4.45

    参考：[#8952](https://www.sqlalchemy.org/trac/ticket/8952)

### postgresql

+   **[postgresql] [bug]**

    修复了一个 bug，即 PostgreSQL `Insert.on_conflict_do_update.constraint` 参数将接受 `Index` 对象，但不会将此索引展开为其各个索引表达式，而是在 ON CONFLICT ON CONSTRAINT 子句中呈现其名称，而 PostgreSQL 不接受这种形式的“约束名称”；该参数仅接受唯一或排除约束名称。参数继续接受索引，但现在将其展开为其组成表达式以进行呈现。

    此更改还被**反向移植**到：1.4.46

    参考：[#9023](https://www.sqlalchemy.org/trac/ticket/9023)

+   **[postgresql] [bug]**

    对 PostgreSQL 方言在从表中反射列时如何考虑列类型进行了调整，以适应可能从 PG `format_type()` 函数返回 NULL 的替代后端。

    此更改还被**反向移植**到：1.4.45

    参考：[#8748](https://www.sqlalchemy.org/trac/ticket/8748)

+   **[postgresql] [bug]**

    增加了对 asyncpg 和 psycopg（仅适用于 SQLAlchemy 2.0）使用 PG 全文函数的显式支持，关于第一个参数的 `REGCONFIG` 类型转换，之前会错误地将其转换为 VARCHAR，在这些方言上依赖显式类型转换会导致失败。这包括对 `to_tsvector`、`to_tsquery`、`plainto_tsquery`、`phraseto_tsquery`、`websearch_to_tsquery`、`ts_headline` 的支持，其中每个函数都会根据传递的参数数量确定第一个字符串参数是否应该被解释为 PostgreSQL 的 `REGCONFIG` 值；如果是，则使用新添加的类型对象 `REGCONFIG` 将参数显式转换为 SQL 表达式。

    参考：[#8977](https://www.sqlalchemy.org/trac/ticket/8977)

+   **[postgresql] [错误]**

    修复了新修订的 PostgreSQL 范围类型（例如 `INT4RANGE`）不能被设置为 `TypeDecorator` 自定义类型的实现时引发 `TypeError` 的回归。

    参考：[#9020](https://www.sqlalchemy.org/trac/ticket/9020)

+   **[postgresql] [错误]**

    `Range.__eq___()` 现在会在与不同类的实例比较时返回 `NotImplemented`，而不是引发 `AttributeError` 异常。

    参考：[#8984](https://www.sqlalchemy.org/trac/ticket/8984)

### sqlite

+   **[sqlite] [用例]**

    增加了对 SQLite 后端反射“DEFERRABLE”和“INITIALLY”关键字的支持，这些关键字可能出现在外键构造中。贡献的拉取请求来自 Michael Gorven。

    这个变更也被 **反向移植** 到：1.4.45

    参考：[#8903](https://www.sqlalchemy.org/trac/ticket/8903)

+   **[sqlite] [用例]**

    增加了对 SQLite 方言中包含在索引中的表达式导向 WHERE 条件的反射支持，类似于 PostgreSQL 方言的方式。贡献的拉取请求来自 Tobias Pfeiffer。

    这个变更也被 **反向移植** 到：1.4.45

    参考：[#8804](https://www.sqlalchemy.org/trac/ticket/8804)

### oracle

+   **[oracle] [错误]**

    修复了 Oracle 编译器中`FunctionElement.column_valued()`的语法错误问题，导致名称`COLUMN_VALUE`未正确限定源表。

    此更改还**被迁移**到：1.4.45

    参考：[#8945](https://www.sqlalchemy.org/trac/ticket/8945)

### 测试

+   **[测试] [错误]**

    修复了 tox.ini 文件中的问题，tox 4.0 系列对“passenv”的格式进行了更改，导致 tox 无法正确运行，特别是在 tox 4.0.6 中引发错误。

    此更改还**被迁移**到：1.4.46

+   **[测试] [错误]**

    添加了针对第三方方言的新排除规则`unusual_column_name_characters`，可以将其“关闭”以不支持具有不寻常字符（如点、斜杠或百分号）的列名的第三方方言，即使名称已经正确引用。

    此更改还**被迁移**到：1.4.46

    参考：[#9002](https://www.sqlalchemy.org/trac/ticket/9002)

## 2.0.0b4

发布日期：2022 年 12 月 5 日

### orm

+   **[orm] [功能]**

    添加了一个新的参数`mapped_column.use_existing_column`，以适应单表继承映射的用例，其中超类上的多个子类指示使用相同的列。之前可以使用`declared_attr()`与在超类的`.__table__`中查找现有列来实现此模式，但现在也已更新为使用`mapped_column()`以及 pep-484 类型，以一种简单和简洁的方式。

    另请参阅

    使用`use_existing_column`解决列冲突

    参考：[#8822](https://www.sqlalchemy.org/trac/ticket/8822)

+   **[orm] [用例]**

    添加了对自定义用户定义类型的支持，这些类型扩展了 Python `enum.Enum`基类，当使用注释声明表特性时，可以自动解析为 SQLAlchemy `Enum` SQL 类型。该功能通过向 ORM 类型映射功能添加新的查找功能实现，并包括更改默认生成的`Enum`的参数以及在映射中设置具有特定参数的特定`enum.Enum`类型的支持。

    另请参阅

    在类型映射中使用 Python 枚举或 pep-586 字面类型

    参考：[#8859](https://www.sqlalchemy.org/trac/ticket/8859)

+   **[orm] [用例]**

    为相关的 ORM 属性构造（包括 `mapped_column()`、`relationship()` 等）添加了 `mapped_column.compare` 参数，以便在使用声明式数据类映射功能时，为 Python 数据类的 `field()` 提供 `compare` 参数。感谢 Simon Schiele 提供的拉取请求。

    参考：[#8905](https://www.sqlalchemy.org/trac/ticket/8905)

+   **[orm] [性能] [错误]**

    在启用 ORM 的 SQL 语句中进一步增强了性能，特别是针对 ORM 语句构造中的调用计数，使用 `aliased()` 与 `union()` 等“复合”构造的组合，以及直接改进了 ORM 通过 `aliased()` 等构造频繁使用的 `corresponding_column()` 内部方法的性能。

    参考：[#8796](https://www.sqlalchemy.org/trac/ticket/8796)

+   **[orm] [错误]**

    修复了在列属性的注解`Mapped`中使用未知数据类型会静默失败而不是报告异常的问题；现在会引发一个信息丰富的异常消息。

    参考：[#8888](https://www.sqlalchemy.org/trac/ticket/8888)

+   **[orm] [错误]**

    修复了与字典类型一起使用 `Mapped` 的一系列问题，例如 `Mapped[Dict[str, str] | None]` 在声明式 ORM 映射中无法正确解释的问题。修复了正确“去可选化”此类型的支持，包括在 `type_annotation_map` 中查找时修复。

    参考：[#8777](https://www.sqlalchemy.org/trac/ticket/8777)

+   **[orm] [错误]**

    修复了在声明式数据类映射功能中的错误，该功能在映射中使用带有 `__allow_unmapped__` 指令的普通数据类字段将不会为这些字段创建具有正确类级状态的数据类，不当地在数据类本身将 `Field` 对象替换为类级默认值后，将原始 `Field` 对象复制到类中。

    参考：[#8880](https://www.sqlalchemy.org/trac/ticket/8880)

+   **[orm] [错误] [回归]**

    修复了一个回归问题，即刷新映射到子查询的映射类（例如直接映射或某些形式的具体表继承），如果使用了`Mapper.eager_defaults`参数，则会失败。

    参考：[#8812](https://www.sqlalchemy.org/trac/ticket/8812)

+   **[orm] [bug]**

    修复了 2.0.0b3 中由[#8759](https://www.sqlalchemy.org/trac/ticket/8759)引起的回归问题，即使用限定名称（如`sqlalchemy.orm.Mapped`）指示`Mapped`名称时，声明性无法识别为指示`Mapped`构造。

    参考：[#8853](https://www.sqlalchemy.org/trac/ticket/8853)

### orm 扩展

+   **[用例] [orm 扩展]**

    支持`association_proxy()`扩展函数，以在使用原生数据类功能描述的 Python `dataclasses`配置中参与，详见声明性数据类映射。包括属性级参数，如`association_proxy.init`和`association_proxy.default_factory`。

    协会代理文档也已更新，示例中使用了“带注解的声明性表”形式，包括用于`AssocationProxy`本身的类型注解。

    参考：[#8878](https://www.sqlalchemy.org/trac/ticket/8878)

### sql

+   **[sql] [用例]**

    添加了`ScalarValues`，可用作列元素，允许在`IN`子句中使用`Values`，或与`ANY`或`ALL`集合聚合一起使用。当在`IN`或`NOT IN`操作中使用时，现在将`Values`实例强制转换为`ScalarValues`。

    参考：[#6289](https://www.sqlalchemy.org/trac/ticket/6289)

+   **[sql] [bug]**

    修复了在缓存键生成中发现的关键内存问题，对于使用大量 ORM 别名和子查询的非常大型和复杂的 ORM 语句，缓存键生成可能会产生比语句本身大得多的键。非常感谢 Rollo Konig Brock 对于最终识别此问题的非常耐心和长期的帮助。

    此更改也**被回溯**至：1.4.44

    引用：[#8790](https://www.sqlalchemy.org/trac/ticket/8790)

+   **[SQL] [错误]**

    对 `numeric` pep-249 参数风格的处理方式已经重写，并且现在得到了完全支持，包括“扩展 IN”和“insertmanyvalues”等特性。源 SQL 构造中的参数名称也可以重复，在数字格式中使用单个参数会被正确表示。引入了一个名为 `numeric_dollar` 的额外数字参数风格，它是 asyncpg 方言专门使用的；该参数风格等效于 `numeric`，但数字指示符使用美元符号而不是冒号。asyncpg 方言现在直接使用 `numeric_dollar` 参数风格，而不是首先编译为 `format` 风格。

    `numeric` 和 `numeric_dollar` 参数风格假定目标后端能够按任意顺序接收数字参数，并且将给定的参数值与语句进行匹配，基于匹配它们的位置（从 1 开始）与数字指示符。这是`numeric`参数风格的正常行为，尽管观察到 SQLite DBAPI 实现了一个未使用的`numeric`风格，它不遵守参数顺序。

    引用：[#8849](https://www.sqlalchemy.org/trac/ticket/8849)

+   **[SQL] [错误]**

    调整了 `RETURNING` 的渲染，特别是在使用 `Insert` 时，现在它使用与 `Select` 构造相同的逻辑来生成标签，这将包括消除歧义的标签，以及使用命名列周围的 SQL 函数将使用列名本身作为标签。这样做可以在从 `Select` 构造或使用 `UpdateBase.returning()` 的 DML 语句中选择行时建立更好的交叉兼容性。1.4 系列还进行了较窄范围的更改，仅调整了函数标签问题。

    引用：[#8770](https://www.sqlalchemy.org/trac/ticket/8770)

### 模式

+   **[模式] [错误]**

    现在，对于将`Column`对象附加到`Table`对象，制定了更严格的规则，包括将一些先前的弃用警告移至异常，并防止一些以前的情况导致重复的列出现在表中，当`Table.extend_existing`设置为`True`时，无论是在编程时构建`Table`还是在反射操作期间。

    有关这些更改的详细信息，请参见对具有相同名称、键的表对象列的替换规则更严格。

    另请参见

    对具有相同名称、键的表对象列的替换规则更严格

    参考：[#8925](https://www.sqlalchemy.org/trac/ticket/8925)

### 输入

+   **[输入] [用例]**

    添加了一个新类型`SQLColumnExpression`，用户可以在用户代码中指定该类型以表示任何 SQL 列导向表达式，包括基于`ColumnElement`和基于 ORM `QueryableAttribute`的表达式。此类型是一个真实的类，而不是别名，因此也可以用作其他对象的基础。还包括一个额外的 ORM 特定子类`SQLORMExpression`。

    参考：[#8847](https://www.sqlalchemy.org/trac/ticket/8847)

+   **[输入] [错误]**

    调整了 Python `enum.IntFlag`类的内部使用，该类在 Python 3.11 中更改了其行为合同。这并没有导致运行时失败，但在 Python 3.11 下导致了类型运行失败。

    参考：[#8783](https://www.sqlalchemy.org/trac/ticket/8783)

+   **[输入] [错误]**

    `sqlalchemy.ext.mutable`扩展和`sqlalchemy.ext.automap`扩展现在完全符合 pep-484 类型。非常感谢 Gleb Kisenkov 在此方面的努力。

    参考：[#6810](https://www.sqlalchemy.org/trac/ticket/6810), [#8667](https://www.sqlalchemy.org/trac/ticket/8667)

+   **[输入] [错误]**

    修正了对`relationship.secondary`参数的类型支持，该参数也可以接受返回`FromClause`的可调用函数（lambda）。

+   **[输入] [错误]**

    对`sessionmaker`和`async_sessionmaker`的输入进行了改进，使得它们的返回值的默认类型将是`Session`或`AsyncSession`，无需显式地进行类型声明。之前，Mypy 无法从其通用基类中自动推断这些返回类型。

    作为这一变更的一部分，`Session`，`AsyncSession`，`sessionmaker`和`async_sessionmaker`的参数除了初始的“bind”参数外，都已经变成了仅限关键字参数，其中包括一直被记录为关键字参数的参数，如`Session.autoflush`，`Session.class_`等。

    感谢 Sam Bull 的拉取请求。

    参考：[#8842](https://www.sqlalchemy.org/trac/ticket/8842)

+   **[typing] [bug]**

    修复了将返回列元素可迭代的可调用函数传递给`relationship.order_by`时，在类型检查器中被标记为错误的问题。

    参考：[#8776](https://www.sqlalchemy.org/trac/ticket/8776)

### postgresql

+   **[postgresql] [usecase]**

    补充[#8690](https://www.sqlalchemy.org/trac/ticket/8690)，新增了一些比较方法，如`Range.adjacent_to()`，`Range.difference()`，`Range.union()`等，添加到了 PG 特定的范围对象中，使其与底层的`AbstractRange.comparator_factory`实现的标准运算符相匹配。

    另外，该类的`__bool__()`方法已校正，以与常见的 Python 容器行为以及其他流行的 PostgreSQL 驱动程序一致：现在它告诉范围实例是否*不*为空，而不是相反。

    感谢 Lele Gaifax 的拉取请求。

    参考：[#8765](https://www.sqlalchemy.org/trac/ticket/8765)

+   **[postgresql] [change] [asyncpg]**

    将 asyncpg 使用的 paramstyle 从`format`更改为`numeric_dollar`。这有两个主要好处，因为它不需要对语句进行额外处理，并且允许语句中存在重复的参数。

    参考：[#8926](https://www.sqlalchemy.org/trac/ticket/8926)

+   **[postgresql] [bug] [mssql]**

    仅适用于 PostgreSQL 和 SQL Server 方言，调整了编译器，以便在渲染 RETURNING 子句中的列表达式时，对于生成标签的 SQL 表达式元素建议使用在 SELECT 语句中使用的“非匿名”标签；主要示例是可能作为列类型的一部分发出的 SQL 函数，其中标签名称应默认匹配列名称。这恢复了一个在版本 1.4.21 中由于[#6718](https://www.sqlalchemy.org/trac/ticket/6718)、[#6710](https://www.sqlalchemy.org/trac/ticket/6710)而发生变化的行为，该行为之前定义不清晰。Oracle 方言具有不同的 RETURNING 实现，不受此问题影响。版本 2.0 在广泛扩展的其他后端支持上进行了全面更改。

    此更改也已**回溯**至：1.4.44

    参考：[#8770](https://www.sqlalchemy.org/trac/ticket/8770)

+   **[postgresql] [bug]**

    为新的 PostgreSQL `Range`类型添加了额外的类型检测，之前允许 psycopg2 原生范围对象直接被 DBAPI 接收而不被 SQLAlchemy 拦截的情况停止工作，因为现在我们有了自己的值对象。`Range`对象已经得到增强，以便 SQLAlchemy Core 在否则模糊的情况下检测到它（例如与日期的比较）并应用适当的绑定处理程序。感谢 Lele Gaifax 的拉取请求。

    参考：[#8884](https://www.sqlalchemy.org/trac/ticket/8884)

### mssql

+   **[mssql] [bug]**

    由 [#8177](https://www.sqlalchemy.org/trac/ticket/8177) 的组合引起的回归修复，除非用于语句的 `fast_executemany` + DBAPI `executemany`，否则重新启用 SQL Server 的 `setinputsizes`，以及 [#6047](https://www.sqlalchemy.org/trac/ticket/6047)，实现“insertmanyvalues”，它绕过了 DBAPI 的 `executemany`，而是用于 INSERT 语句的自定义 DBAPI 执行。如果打开了 `fast_executemany`，则会错误地不使用 `setinputsizes` 用于多参数集 INSERT 语句，因为检查会错误地假定这是一个 DBAPI `executemany` 调用。然后“回归”是“insertmanyvalues”语句格式显然对于不使用相同类型的多行特别敏感，因此在这种情况下尤其需要 `setinputsizes`。

    此修复修复了 `fast_executemany` 检查，使其仅在要使用真正的 DBAPI `executemany` 时禁用 `setinputsizes`。

    参考资料：[#8917](https://www.sqlalchemy.org/trac/ticket/8917)

### oracle

+   **[oracle] [错误]**

    对于在 1.4.43 中发布的 Oracle 修复 [#8708](https://www.sqlalchemy.org/trac/ticket/8708)，继续修复了绑定参数名称以下划线开头的问题，Oracle 不允许这样做，在所有情况下仍然不能正确地转义。

    此更改也**被反向移植**至：1.4.45

    参考资料：[#8708](https://www.sqlalchemy.org/trac/ticket/8708)

### 测试

+   **[测试] [错误]**

    修复了测试套件中 `--disable-asyncio` 参数无法实际不运行 greenlet 测试并且也无法阻止测试套件在整个运行期间使用“包装” greenlet 的问题。设置此参数现在可以确保在整个运行期间不会使用 greenlet 或 asyncio。

    此更改也**被反向移植**至：1.4.44

    参考资料：[#8793](https://www.sqlalchemy.org/trac/ticket/8793)

### orm

+   **[orm] [功能]**

    添加了一个新参数`mapped_column.use_existing_column`，以适应使用单表继承映射的用例，该映射使用了一个以上的子类指示在超类上发生的相同列的模式。先前，可以通过在超类的 `.__table__` 中使用 `declared_attr()` 来定位现有列来实现此模式，但现在已更新为与 pep-484 类型提示一起使用 `mapped_column()` 以及简单而简洁地工作。

    另请参阅

    使用 `use_existing_column` 解决列冲突

    参考资料：[#8822](https://www.sqlalchemy.org/trac/ticket/8822)

+   **[orm] [用例]**

    增加了对自定义用户定义类型的支持，这些类型扩展了 Python `enum.Enum`基类，当使用注释声明表功能时，将自动解析为 SQLAlchemy `Enum` SQL 类型。 该功能是通过向 ORM 类型映射功能添加新的查找功能实现的，并包括支持更改默认生成的`Enum`的参数以及设置在映射中具有特定参数的特定`enum.Enum`类型。

    另请参阅

    在类型映射中使用 Python Enum 或 pep-586 Literal 类型

    参考：[#8859](https://www.sqlalchemy.org/trac/ticket/8859)

+   **[orm] [用例]**

    在相关的 ORM 属性构造中添加了`mapped_column.compare`参数，包括`mapped_column()`、`relationship()`等，以提供 Python dataclasses `field()`上的`compare`参数，当使用声明性数据类映射功能时。 拉取请求由 Simon Schiele 提供。

    参考：[#8905](https://www.sqlalchemy.org/trac/ticket/8905)

+   **[orm] [性能] [bug]**

    ORM 启用的 SQL 语句中的其他性能增强，特别针对 ORM 语句的构造中的调用次数，使用`aliased()`与`union()`和类似的“复合”构造的组合，以及直接改进了 ORM 大量使用的`corresponding_column()`内部方法的性能，例如`aliased()`等。

    参考：[#8796](https://www.sqlalchemy.org/trac/ticket/8796)

+   **[orm] [bug]**

    修复了在列属性的`Mapped`注释中使用未知数据类型会静默失败而不是报告异常的问题；现在会引发一个信息性异常消息。

    参考：[#8888](https://www.sqlalchemy.org/trac/ticket/8888)

+   **[orm] [bug]**

    修复了涉及与字典类型一起使用的 `Mapped` 在声明性 ORM 映射中无法正确解释的一系列问题，例如 `Mapped[Dict[str, str] | None]`，支持正确“去可选化”此类型，包括用于 `type_annotation_map` 中查找。

    参考：[#8777](https://www.sqlalchemy.org/trac/ticket/8777)

+   **[orm] [bug]**

    修复了在 声明性数据类映射 特性中的一个错误，其中在映射中使用带有 `__allow_unmapped__` 指令的普通数据类字段将不会为这些字段创建具有正确类级状态的数据类，在 dataclasses 自身替换了 `Field` 对象为类级默认值后不适当地将原始 `Field` 对象复制到类中。

    参考：[#8880](https://www.sqlalchemy.org/trac/ticket/8880)

+   **[orm] [bug] [regression]**

    修复了回归，其中刷新一个映射类，该映射类映射到一个子查询，例如直接映射或某些形式的具体表继承，如果使用了 `Mapper.eager_defaults` 参数，则会失败。

    参考：[#8812](https://www.sqlalchemy.org/trac/ticket/8812)

+   **[orm] [bug]**

    修复了 2.0.0b3 中由 [#8759](https://www.sqlalchemy.org/trac/ticket/8759) 引起的回归，其中指示使用限定名如 `sqlalchemy.orm.Mapped` 表示 `Mapped` 名称将无法被声明性识别为指示 `Mapped` 结构。

    参考：[#8853](https://www.sqlalchemy.org/trac/ticket/8853)

### orm 扩展

+   **[usecase] [orm 扩展]**

    增加了对 `association_proxy()` 扩展函数的支持，以便在使用原生数据类特性时，在 Python `dataclasses` 配置中参与，详情请参见 声明性数据类映射。包括属性级参数，如 `association_proxy.init` 和 `association_proxy.default_factory`。

    协会代理的文档也已更新，以在示例中使用“带注释的声明性表格”形式，包括用于 `AssocationProxy` 本身的类型注释。

    参考：[#8878](https://www.sqlalchemy.org/trac/ticket/8878)

### sql

+   **[sql] [usecase]**

    添加了`ScalarValues`，它可以用作列元素，允许在 `IN` 子句中使用 `Values` 或与 `ANY` 或 `ALL` 集合聚合一起使用。这个新类是使用方法 `Values.scalar_values()` 生成的。当在 `IN` 或 `NOT IN` 操作中使用时，`Values` 实例现在会被强制转换为 `ScalarValues`。

    参考：[#6289](https://www.sqlalchemy.org/trac/ticket/6289)

+   **[sql] [bug]**

    修复了缓存密钥生成中发现的关键内存问题，对于使用大量具有子查询的 ORM 别名的非常大且复杂的 ORM 语句，缓存密钥生成可能会生成比语句本身大数个数量级的密钥。非常感谢 Rollo Konig Brock 长期以来在最终确定此问题上提供的非常耐心和持久的帮助。

    此更改也已 **回溯到**：1.4.44

    参考：[#8790](https://www.sqlalchemy.org/trac/ticket/8790)

+   **[sql] [bug]**

    `numeric` pep-249 参数风格的方法已经重写，现在得到了完全支持，包括“扩展 IN”和“insertmanyvalues”等功能。参数名称也可以在源 SQL 结构中重复，这将在数字格式中使用单个参数正确表示。引入了一个额外的数字参数风格称为 `numeric_dollar`，这是 asyncpg 方言专门使用的；该参数风格等同于 `numeric`，只是数值指示符使用美元符号而不是冒号。asyncpg 方言现在直接使用 `numeric_dollar` 参数风格，而不是首先编译为 `format` 风格。

    `numeric` 和 `numeric_dollar` 参数风格假定目标后端能够以任意顺序接收数字参数，并将给定的参数值与语句匹配，基于匹配它们的位置（从 1 开始）到数字指示符。这是`numeric`参数风格的正常行为，尽管观察到 SQLite DBAPI 实现了一个未使用的`numeric`风格，不遵循参数顺序。

    参考：[#8849](https://www.sqlalchemy.org/trac/ticket/8849)

+   **[sql] [bug]**

    调整了`RETURNING`的渲染，特别是在使用`Insert`时，现在它会像`Select`构造一样使用相同的逻辑生成标签，这将包括消除歧义的标签，以及将围绕命名列的 SQL 函数标记为列名本身。这在从`Select`构造或使用`UpdateBase.returning()`的 DML 语句中选择行时建立了更好的交叉兼容性。1.4 系列还对函数标签问题进行了较小的调整。

    参考：[#8770](https://www.sqlalchemy.org/trac/ticket/8770)

### schema

+   **[schema] [bug]**

    现在对将`Column`对象附加到`Table`对象的规则更严格，将一些先前的弃用警告转换为异常，并阻止一些先前可能导致表中出现重复列的情况，当`Table.extend_existing`设置为`True`时，无论是在编程`Table`构造还是在反射操作期间。

    请参阅表对象中具有相同名称、键的列替换规则更严格以了解这些更改的概述。

    另请参见

    表对象中具有相同名称、键的列替换规则更严格

    参考：[#8925](https://www.sqlalchemy.org/trac/ticket/8925)

### typing

+   **[typing] [usecase]**

    添加了一个新类型`SQLColumnExpression`，用户可以在代码中指示表示任何 SQL 列导向表达式，包括基于`ColumnElement`和 ORM `QueryableAttribute`的表达式。这个类型是一个真正的类，而不是一个别名，因此也可以用作其他对象的基础。还包括了一个额外的 ORM 特定子类`SQLORMExpression`。

    参考：[#8847](https://www.sqlalchemy.org/trac/ticket/8847)

+   **[typing] [bug]**

    调整了 Python `enum.IntFlag` 类的内部使用，该类在 Python 3.11 中改变了其行为契约。这并没有导致运行时失败，但在 Python 3.11 下导致了类型运行失败。

    参考：[#8783](https://www.sqlalchemy.org/trac/ticket/8783)

+   **[typing] [bug]**

    `sqlalchemy.ext.mutable`扩展和`sqlalchemy.ext.automap`扩展现在完全符合 pep-484 类型。非常感谢 Gleb Kisenkov 在这方面的努力。

    参考：[#6810](https://www.sqlalchemy.org/trac/ticket/6810), [#8667](https://www.sqlalchemy.org/trac/ticket/8667)

+   **[typing] [bug]**

    修正了对`relationship.secondary`参数的类型支持，该参数也可以接受返回`FromClause`的可调用函数（lambda）。

+   **[typing] [bug]**

    改进了`sessionmaker`和`async_sessionmaker`的类型，使其返回值的默认类型将是`Session`或`AsyncSession`，无需显式地进行类型定义。以前，Mypy 无法从其通用基类自动推断这些返回类型。

    作为这一变更的一部分，`Session`、`AsyncSession`、`sessionmaker`和`async_sessionmaker`的参数除了初始的“bind”参数外，都已被设置为仅限关键字参数，其中包括一直被记录为关键字参数的参数，如`Session.autoflush`、`Session.class_`等。

    感谢 Sam Bull 提供的拉取请求。

    参考：[#8842](https://www.sqlalchemy.org/trac/ticket/8842)

+   **[typing] [bug]**

    修复了将返回列元素可迭代对象的可调用函数传递给`relationship.order_by`时，在类型检查器中被标记为错误的问题。

    参考：[#8776](https://www.sqlalchemy.org/trac/ticket/8776)

### postgresql

+   **[postgresql] [usecase]**

    补充[#8690](https://www.sqlalchemy.org/trac/ticket/8690)，新增了诸如`Range.adjacent_to()`、`Range.difference()`、`Range.union()`等新的比较方法，被添加到了 PG 特定的范围对象中，使其与底层`AbstractRange.comparator_factory`实现的标准操作符保持一致。

    此外，类的 `__bool__()` 方法已被更正，以使其与常见的 Python 容器行为以及其他流行的 PostgreSQL 驱动程序的行为保持一致：现在它告诉范围实例是否*不*为空，而不是相反。

    感谢 Lele Gaifax 提供的拉取请求。

    参考：[#8765](https://www.sqlalchemy.org/trac/ticket/8765)

+   **[postgresql] [change] [asyncpg]**

    更改了 asyncpg 使用的 paramstyle，从 `format` 更改为 `numeric_dollar`。这有两个主要好处，因为它不需要对语句进行额外处理，并允许语句中存在重复参数。

    参考：[#8926](https://www.sqlalchemy.org/trac/ticket/8926)

+   **[postgresql] [bug] [mssql]**

    仅针对 PostgreSQL 和 SQL Server 方言，调整了编译器，以便在 RETURNING 子句中渲染列表达式时，为生成标签的 SQL 表达式元素建议使用“non anon”标签，这是在 SELECT 语句中使用的标签；主要示例是可能作为列类型的一部分发出的 SQL 函数，在默认情况下标签名称应与列名称匹配。这恢复了一个在版本 1.4.21 中因[#6718](https://www.sqlalchemy.org/trac/ticket/6718)、[#6710](https://www.sqlalchemy.org/trac/ticket/6710)而更改的行为，该行为并不明确。Oracle 方言具有不同的 RETURNING 实现，不受此问题影响。版本 2.0 在其广泛扩展的对其他后端的 RETURNING 支持方面进行了全面更改。

    此更改也被**回溯**到：1.4.44

    参考：[#8770](https://www.sqlalchemy.org/trac/ticket/8770)

+   **[postgresql] [bug]**

    为新的 PostgreSQL `Range`类型增加了额外的类型检测，以前的情况允许 psycopg2 原生范围对象直接由 DBAPI 接收而不受 SQLAlchemy 拦截，因为我们现在有了我们自己的值对象。 `Range`对象已经得到增强，以便 SQLAlchemy Core 在其他模糊情况下检测到它（例如与日期进行比较），并应用适当的绑定处理程序。拉取请求由 Lele Gaifax 提供。

    参考：[#8884](https://www.sqlalchemy.org/trac/ticket/8884)

### mssql

+   **[mssql] [bug]**

    修复了由[#8177](https://www.sqlalchemy.org/trac/ticket/8177)的组合引起的回归，重新启用了用于 SQL Server 的 setinputsizes，除非使用了 fast_executemany + DBAPI executemany 用于语句，以及[#6047](https://www.sqlalchemy.org/trac/ticket/6047)，实现了“insertmanyvalues”，它绕过了 DBAPI executemany，而是为 INSERT 语句提供了自定义 DBAPI execute。如果 fast_executemany 被打开，则对于使用“insertmanyvalues”的多参数集 INSERT 语句，setinputsizes 将不会被正确使用，因为检查将错误地假设这是一个 DBAPI executemany 调用。然后，“回归”将是“insertmanyvalues”语句格式显然对于不使用相同类型的每一行的多行特别敏感，因此在这种情况下，特别需要 setinputsizes。

    修复了 fast_executemany 检查，以便仅在要使用真实 DBAPI executemany 时禁用 setinputsizes。

    参考：[#8917](https://www.sqlalchemy.org/trac/ticket/8917)

### oracle

+   **[oracle] [bug]**

    继续修复了 Oracle 修复[#8708](https://www.sqlalchemy.org/trac/ticket/8708)在 1.4.43 中发布的问题，其中以下划线开头的绑定参数名称，在所有情况下仍未正确转义。

    这个更改也**回溯到**: 1.4.45

    参考：[#8708](https://www.sqlalchemy.org/trac/ticket/8708)

### 测试

+   **[tests] [bug]**

    修复了测试套件中`--disable-asyncio`参数未能实际运行绿色测试的问题，也不会阻止套件使用“包装”绿色测试套件的问题。当设置时，此参数现在确保在整个运行期间不会发生绿色或 asyncio 的使用。

    这个更改也**回溯到**: 1.4.44

    参考：[#8793](https://www.sqlalchemy.org/trac/ticket/8793)

## 2.0.0b3

发布日期：2022 年 11 月 4 日

### orm

+   **[orm] [bug]**

    在联合加载中修复了一个问题，其中当在三个映射器之间进行快速加载时，如果在外部/内部联接的快速加载的特定组合中发生断言失败，那么断言将会失败，当在三个映射器之间进行快速加载时，中间映射器是一个继承的子类映射器。

    这个更改也**回溯到**: 1.4.43

    引用：[#8738](https://www.sqlalchemy.org/trac/ticket/8738)

+   **[orm] [bug]**

    修复了涉及 `Select` 构造的错误，其中 `Select.select_from()` 与 `Select.join()` 的组合，以及使用 `Select.join_from()` 时，当查询的列子句没有显式包含 JOIN 的左侧实体时，会导致 `with_loader_criteria()` 功能以及单表继承查询所需的 IN 条件不渲染。现在正确的实体已传递到内部生成的 `Join` 对象，以便正确添加左侧实体的条件。

    此更改也已**反向移植**到：1.4.43

    引用：[#8721](https://www.sqlalchemy.org/trac/ticket/8721)

+   **[orm] [bug]**

    当将`with_loader_criteria()`选项用作添加到特定“加载器路径”的加载器选项时，例如在`Load.options()`中使用时，现在会引发一个信息性异常。此用法不受支持，因为`with_loader_criteria()`仅应用于顶层加载器选项。以前会生成内部错误。

    此更改也已**反向移植**到：1.4.43

    引用：[#8711](https://www.sqlalchemy.org/trac/ticket/8711)

+   **[orm] [bug]**

    为`Session.get()`改进了“字典模式”，使得同义词名称可以指示为主键属性名称。

    此更改也已**反向移植**到：1.4.43

    引用：[#8753](https://www.sqlalchemy.org/trac/ticket/8753)

+   **[orm] [bug]**

    修复了继承映射器中“selectin_polymorphic”加载无法正确工作的问题，如果`Mapper.polymorphic_on`参数引用的是不直接映射到类的 SQL 表达式，则会出现此问题。

    此更改也已**反向移植**到：1.4.43

    引用：[#8704](https://www.sqlalchemy.org/trac/ticket/8704)

+   **[orm] [bug]**

    修复了在使用`Query`对象作为迭代器时，如果在迭代过程中出现用户定义的异常情况，底层的 DBAPI 游标将不会被关闭的问题，从而导致迭代器被 Python 解释器关闭。当使用`Query.yield_per()`创建服务器端游标时，这将导致通常与 MySQL 相关的服务器端游标不同步的问题，并且由于没有直接访问`Result`对象，最终用户代码无法访问游标以关闭它。

    为了解决这个问题，在迭代器方法中应用了对`GeneratorExit`的捕获，这将在迭代器被中断时关闭结果对象，并且根据定义将被 Python 解释器关闭。

    作为 1.4 系列实施的这一变更的一部分，确保了所有`Result`实现都可用`.close()`方法，包括`ScalarResult`、`MappingResult`。2.0 版本的这一变更还包括用于与`Result`类一起使用的新上下文管理器模式。

    这一变更也被**回溯**到：1.4.43

    参考：[#8710](https://www.sqlalchemy.org/trac/ticket/8710)

### orm 声明性

+   **[orm] [declarative] [bug]**

    在 ORM 声明性注释中添加了对于为`relationship()`指定的类名的支持，以及`Mapped`符号本身的名称，可以与直接类名不同，以支持诸如将`Mapped`导入为`from sqlalchemy.orm import Mapped as M`的情况，或者相关类名以类似方式用不同名称导入。此外，作为`relationship()`的主参数给定的目标类名将始终优先于左侧注释中给定的名称，以便仍然可以在注释中使用无法导入的名称，而且也不匹配类名的情况。

    参考：[#8759](https://www.sqlalchemy.org/trac/ticket/8759)

+   **[orm] [declarative] [bug]**

    改进了对使用不包括`Mapped[]`的注释的遗留 1.4 映射的支持，通过确保`__allow_unmapped__`属性可用于允许这种遗留注释通过未引发错误且不在 ORM 运行时上下文中被解释的注释性声明。此外，当检测到此条件时，改进了生成的错误消息，并为应如何处理此情况添加了更多文档。不幸的是，1.4 WARN_SQLALCHEMY_20 迁移警告无法使用当前架构在运行时检测到此特定的配置问题。

    参考：[#8692](https://www.sqlalchemy.org/trac/ticket/8692)

+   **[orm] [declarative] [bug]**

    改变了`Mapper`的基本配置行为，其中在`Mapper.properties`字典中明确存在的`Column`对象，无论是直接存在还是包含在映射属性对象中，现在都将根据它们在映射的`Table`（或其他可选择的）中出现的顺序进行映射（假设它们实际上是该表列的一部分），从而保持在映射的可选择项中的列的顺序与在映射类中被调制的顺序相同，以及在 ORM SELECT 语句中呈现的顺序。此前（“此前”指的是自版本 0.0.1 以来），`Mapper.properties`字典中的`Column`对象总是首先映射，超出了映射的其他列在映射的`Table`中的映射顺序，导致映射器分配属性给映射类的顺序与它们在语句中呈现的顺序不一致。

    最明显的变化发生在声明式将声明的列分配给`Mapper`的方式上，具体来说，当具有 DDL 名称明确不同于映射属性名称的`Column`（或`mapped_column()`）对象以及使用`deferred()`等结构时。新的行为将会看到在映射的`Table`中的列顺序与属性映射到类中的顺序相同，分配在`Mapper`本身内，并在 ORM 语句中（如 SELECT 语句）呈现，与`Column`配置为`Mapper`的方式无关。

    参考：[#8705](https://www.sqlalchemy.org/trac/ticket/8705)

+   **[orm] [declarative] [bug]**

    在新的数据类映射功能中修复了问题，其中在某些情况下，在声明基类/抽象基类/混合类上声明的列会泄漏到继承子类的构造函数中。

    参考：[#8718](https://www.sqlalchemy.org/trac/ticket/8718)

+   **[bug] [orm declarative]**

    在声明性类型解析器（即解析`ForwardRef`对象的类型）中修复了问题，其中在一个特定源文件中为列声明的类型，当最终映射的类位于另一个源文件中时会引发`NameError`。现在，这些类型根据每个类所在的模块来解析。

    参考：[#8742](https://www.sqlalchemy.org/trac/ticket/8742)

### engine

+   **[engine] [feature]**

    为了更好地支持迭代`Result`和`AsyncResult`对象的用例，其中用户定义的异常可能会中断迭代，现在这两个对象以及变体，如`ScalarResult`、`MappingResult`、`AsyncScalarResult`、`AsyncMappingResult`，现在都支持上下文管理器的使用，其中结果将在上下文管理器块的结束时关闭。

    此外，确保了上述所有`Result`对象都包括一个`Result.close()`方法以及`Result.closed`访问器，包括以前没有`.close()`方法的`ScalarResult`和`MappingResult`。

    参见

    为 Result、AsyncResult 提供上下文管理器支持

    参考：[#8710](https://www.sqlalchemy.org/trac/ticket/8710)

+   **[引擎] [用例]**

    添加了新参数`PoolEvents.reset.reset_state`参数到`PoolEvents.reset()`事件，其中包含已经放置了将继续接受使用先前参数集的事件钩子的弃用逻辑。这表示关于重置正在进行的各种状态信息，并且用于允许具有完整上下文的自定义重置方案的执行。

    在这个变化中，包括一个也已经回溯到 1.4 的修复，该修复重新启用了`PoolEvents.reset()`事件，以便在所有情况下都发生，包括当`Connection`已经“重置”连接时。

    这两个更改共同允许使用 `PoolEvents.reset()` 事件来实现自定义重置方案，而不是使用 `PoolEvents.checkin()` 事件（它仍然像以前一样运行）。

    引用：[#8717](https://www.sqlalchemy.org/trac/ticket/8717)

+   **[engine] [bug] [regression]**

    修复了在关闭`Connection`并正在将其 DBAPI 连接返回到连接池时，在某些情况下未调用 `PoolEvents.reset()` 事件钩子的问题。

    场景是当`Connection`在将连接返回到池的过程中已经发出了`.rollback()`时，然后它将指示连接池放弃执行自己的“重置”以节省额外的方法调用。但是，这样阻止了在此钩子中使用自定义池重置方案，因为此类钩子定义上做的不仅仅是调用`.rollback()`，并且需要在所有情况下调用。这是版本 1.4 中出现的退化。

    对于版本 1.4，`PoolEvents.checkin()` 仍然可行作为用于自定义“重置”实现的备用事件钩子。版本 2.0 将提供一个改进的 `PoolEvents.reset()` 版本，该版本将为其他场景调用，例如终止 asyncio 连接，并且还会传递有关重置的上下文信息，以允许对不同的重置方案做出不同的响应。

    此更改也**回溯**到：1.4.43

    引用：[#8717](https://www.sqlalchemy.org/trac/ticket/8717)

### sql

+   **[sql] [bug]**

    修复了在`Select` 构造上下文以及其他可能生成“匿名标签”的地方中，`literal_column()` 构造无法正常工作的问题，如果文字表达式包含可能干扰格式字符串的字符，例如括号，这是由于“匿名标签”结构的实现细节所致。

    此更改也**回溯**到：1.4.43

    引用：[#8724](https://www.sqlalchemy.org/trac/ticket/8724)

### typing

+   **[typing] [bug]**

    在引擎和异步引擎包中纠正了各种类型问题。

### postgresql

+   **[postgresql] [feature]**

    向新的`Range`数据对象添加了新的方法`Range.contains()`和`Range.contained_by()`，这些方法反映了 PostgreSQL 的`@>`和`<@`操作符的行为，以及`comparator_factory.contains()`和`comparator_factory.contained_by()` SQL 操作符方法。来自 Lele Gaifax 的拉取请求。

    参考：[#8706](https://www.sqlalchemy.org/trac/ticket/8706)

+   **[postgresql] [usecase]**

    优化了在 PostgreSQL 后端的新 RANGE / MULTIRANGE 支持和更改中描述的新范围对象的方法，以适应特定于驱动程序的范围和多范围对象，以更好地适应传统代码以及将结果从原始 SQL 结果集传回新范围或多范围表达式时。

    参考：[#8690](https://www.sqlalchemy.org/trac/ticket/8690)

### mssql

+   **[mssql] [bug]**

    修复了针对带有 SQL Server 方言的临时表使用`Inspector.has_table()`的问题，在某些 Azure 变体上会失败，因为不支持不必要的信息模式查询。来自 Mike Barry 的拉取请求。

    此更改也**回溯到**：1.4.43

    参考：[#8714](https://www.sqlalchemy.org/trac/ticket/8714)

+   **[mssql] [bug] [reflection]**

    修复了对`Inspector.has_table()`的问题，当针对带有 SQL Server 方言的视图时，由于 1.4 系列中删除了对 SQL Server 的支持而错误地返回`False`。该问题在使用不同反射架构的 2.0 系列中不存在。添加了测试支持，以确保`has_table()`按照视图的规范仍然正常工作。

    此更改也**回溯到**：1.4.43

    参考：[#8700](https://www.sqlalchemy.org/trac/ticket/8700)

### oracle

+   **[oracle] [bug]**

    修复了绑定参数名称的问题，包括那些从同名数据库列自动派生的参数名称，这些参数名称包含通常需要在 Oracle 中用引号引用的字符，在使用 Oracle 方言的“扩展参数”时不会被转义，导致执行错误。 Oracle 方言使用的绑定参数的通常“引用”不适用于“扩展参数”架构，因此现在使用一系列特定于 Oracle 的字符/转义列表来进行转义。

    此更改也**回溯**到：1.4.43

    参考：[#8708](https://www.sqlalchemy.org/trac/ticket/8708)

+   **[oracle] [bug]**

    修复了在首次连接时查询`nls_session_parameters`视图以获取默认小数点字符可能不可用的问题，具体取决于 Oracle 连接模式，并且因此会引发错误的问题。检测小数点字符的方法已简化为直接测试小数值，而不是读取系统视图，这适用于任何后端/驱动程序。

    此更改也**回溯**到：1.4.43

    参考：[#8744](https://www.sqlalchemy.org/trac/ticket/8744)

### orm

+   **[orm] [bug]**

    修复了在连接的急切加载中出现断言失败的问题，在跨三个映射器进行急切加载时，其中中间映射器是继承的子类映射器时，会发生特定外/内连接急切加载组合的断言失败。

    此更改也**回溯**到：1.4.43

    参考：[#8738](https://www.sqlalchemy.org/trac/ticket/8738)

+   **[orm] [bug]**

    修复了涉及`Select`构造的 bug，其中`Select.select_from()`与`Select.join()`的组合，以及使用`Select.join_from()`时，会导致`with_loader_criteria()`功能以及单表继承查询所需的 IN 条件在查询的列子句没有明确包含 JOIN 左侧实体时不会呈现的 bug。现在正确的实体已经传递给内部生成的`Join`对象，以便正确添加对左侧实体的条件。

    此更改也**回溯**到：1.4.43

    参考：[#8721](https://www.sqlalchemy.org/trac/ticket/8721)

+   **[orm] [bug]**

    当将`with_loader_criteria()`选项作为添加到特定“加载器路径”的加载器选项时，例如在`Load.options()`中使用时，现在会引发一个信息性异常。这种用法不受支持，因为`with_loader_criteria()`只打算作为顶级加载器选项使用。以前会生成内部错误。

    这个更改也被**回溯**到：1.4.43

    参考：[#8711](https://www.sqlalchemy.org/trac/ticket/8711)

+   **[orm] [bug]**

    为`Session.get()`改进了“字典模式”，以便可以在命名字典中指示引用主键属性名称的同义词名称。

    这个更改也被**回溯**到：1.4.43

    参考：[#8753](https://www.sqlalchemy.org/trac/ticket/8753)

+   **[orm] [bug]**

    修复了继承映射器的“selectin_polymorphic”加载在`Mapper.polymorphic_on`参数引用的 SQL 表达式不直接映射到类时无法正确工作的问题。

    这个更改也被**回溯**到：1.4.43

    参考：[#8704](https://www.sqlalchemy.org/trac/ticket/8704)

+   **[orm] [bug]**

    修复了在使用`Query`对象作为迭代器时，如果在迭代过程中引发了用户定义的异常情况，底层的 DBAPI 游标将不会被关闭的问题，从而导致迭代器被 Python 解释器关闭。当使用`Query.yield_per()`创建服务器端游标时，这将导致通常与 MySQL 相关的服务器端游标不同步的问题，并且由于无法直接访问`Result`对象，最终用户代码无法访问游标以关闭它。

    为了解决这个问题，在迭代器方法中应用了对`GeneratorExit`的捕获，这将在迭代器被中断时关闭结果对象，并且根据定义将被 Python 解释器关闭。

    作为实施给 1.4 系列的这一变更的一部分，确保了所有`Result`实现包括`ScalarResult`、`MappingResult`上都有`.close()`方法可用。此变更的 2.0 版本还包括用于与`Result`类一起使用的新上下文管理器模式。

    此更改也被**回溯**到：1.4.43

    参考：[#8710](https://www.sqlalchemy.org/trac/ticket/8710)

### orm declarative

+   **[orm] [declarative] [bug]**

    在 ORM 声明性注释中添加了对为`relationship()`指定的类名的支持，以及`Mapped`符号本身的名称，可以与其直接类名不同，以支持诸如`Mapped`被导入为`from sqlalchemy.orm import Mapped as M`的情况，或者相关类名以类似方式用替代名称导入的情况。此外，作为`relationship()`的主参数给定的目标类名将始终优先于左侧注释中给定的名称，以便仍然可以在注释中使用无法导入的名称，这些名称也不匹配类名。

    参考：[#8759](https://www.sqlalchemy.org/trac/ticket/8759)

+   **[orm] [declarative] [bug]**

    改进了对使用注释的传统 1.4 映射的支持，这些注释不包括`Mapped[]`，通过确保`__allow_unmapped__`属性可以用于允许这些传统注释通过 Annotated Declarative 而不会引发错误，并且不会在 ORM 运行时上下文中被解释。此外，改进了在检测到此条件时生成的错误消息，并为应该如何处理这种情况添加了更多文档。不幸的是，1.4 WARN_SQLALCHEMY_20 迁移警告不能检测到当前架构下的这种特定配置问题。

    参考：[#8692](https://www.sqlalchemy.org/trac/ticket/8692)

+   **[orm] [declarative] [bug]**

    改变了`Mapper`的一个基本配置行为，其中`Column`对象，无论是直接存在于`Mapper.properties`字典中，还是包含在映射属性对象中，现在将按照它们在映射的`Table`（或其他可选择的对象）中出现的顺序进行映射（假设它们实际上是该表的列列表的一部分），从而保持在映射的可选择对象中列的顺序与在映射类中被实现的顺序相同，以及在 ORM SELECT 语句中呈现的顺序。以前（“以前”指的是自版本 0.0.1 以来），`Mapper.properties`字典中的`Column`对象总是首先被映射，超过映射的`Table`中的其他列被映射的顺序，导致映射器分配属性给映射类的顺序与它们在语句中呈现的顺序不一致。

    最显著的变化发生在 Declarative 分配声明列给`Mapper`的方式上，特别是当`Column`（或`mapped_column()`）对象具有与映射属性名称明确不同的 DDL 名称时，以及当使用`deferred()`等构造时。新行为将使映射的`Table`中的列顺序与属性映射到类中的顺序相同，在`Mapper`本身内分配，并在 ORM 语句中呈现，独立于`Column`针对`Mapper`的配置方式。

    参考：[#8705](https://www.sqlalchemy.org/trac/ticket/8705)

+   **[orm] [declarative] [bug]**

    在新的 dataclass 映射功能中修复了一个问题，其中在某些情况下，在声明性基类/抽象基类/混合类上声明的列会泄漏到继承子类的构造函数中。

    参考：[#8718](https://www.sqlalchemy.org/trac/ticket/8718)

+   **[错误] [ORM 声明性]**

    修复了声明性类型解析器（即解析`ForwardRef`对象的解析器）中的问题，在一个特定源文件中声明的列类型，在最终映射的类位于另一个源文件时会引发`NameError`。现在，这些类型将根据每个类所在的模块进行解析。

    参考：[#8742](https://www.sqlalchemy.org/trac/ticket/8742)

### 引擎

+   **[引擎] [特性]**

    为了更好地支持迭代`Result`和`AsyncResult`对象的用例，用户定义的异常可能会中断迭代，现在这两个对象以及诸如`ScalarResult`、`MappingResult`、`AsyncScalarResult`、`AsyncMappingResult`等变体现在都支持上下文管理器用法，在上下文管理器块结束时将关闭结果。

    此外，确保了所有上述提到的`Result`对象都包括一个`Result.close()`方法以及`Result.closed`访问器，包括之前没有`.close()`方法的`ScalarResult`和`MappingResult`。

    另请参阅

    结果，AsyncResult 的上下文管理器支持

    参考：[#8710](https://www.sqlalchemy.org/trac/ticket/8710)

+   **[引擎] [用例]**

    添加了 `PoolEvents.reset.reset_state` 参数到 `PoolEvents.reset()` 事件，同时放置了一套逐渐废弃的逻辑，将继续接受使用先前参数集的事件钩子。这些参数指示了关于重置过程如何进行的各种状态信息，并且用于允许以给定的完整上下文进行自定义重置方案。

    在这个变化中，还包括了一个也被后移至 1.4 版本的修复，该修复重新启用了 `PoolEvents.reset()` 事件，以便在所有情况下都继续发生，包括当 `Connection` 已经“重置”连接时。

    这两个变化共同允许使用 `PoolEvents.reset()` 事件来实现自定义重置方案，而不是使用 `PoolEvents.checkin()` 事件（其功能保持不变）。

    参考：[#8717](https://www.sqlalchemy.org/trac/ticket/8717)

+   **[engine] [bug] [regression]**

    修复了当 `Connection` 关闭并正在将其 DBAPI 连接返回到连接池时，`PoolEvents.reset()` 事件钩子在所有情况下都不会被调用的问题。

    场景是当 `Connection` 在返回连接到连接池的过程中已经发出了 `.rollback()` 时，它将指示连接池放弃执行其自己的“重置”以节省额外的方法调用。然而，这阻止了在此钩子中使用自定义连接池重置方案，因为这样的钩子定义上做的不仅仅是调用 `.rollback()`，并且需要在所有情况下调用。这是在版本 1.4 中出现的一个回归。

    对于版本 1.4，`PoolEvents.checkin()`仍然可作为用于自定义“重置”实现的备用事件钩子。版本 2.0 将提供一个改进版本的`PoolEvents.reset()`，用于额外的场景，例如终止 asyncio 连接，并且还传递有关重置的上下文信息，以允许“自定义连接重置”方案，可以根据不同的重置场景以不同方式响应。

    此更改也**回溯**到：1.4.43

    参考：[#8717](https://www.sqlalchemy.org/trac/ticket/8717)

### sql

+   **[sql] [错误]**

    修复了一个问题，该问题阻止`literal_column()`构造在`Select`构造的上下文中正常工作，以及其他可能生成“匿名标签”的地方，如果文字表达式包含可能干扰格式字符串的字符，例如开括号，由于“匿名标签”结构的实现细节。

    此更改也**回溯**到：1.4.43

    参考：[#8724](https://www.sqlalchemy.org/trac/ticket/8724)

### 打字

+   **[打字] [错误]**

    纠正了引擎和异步引擎包中的各种打字问题。

### postgresql

+   **[postgresql] [特性]**

    添加了新的方法`Range.contains()`和`Range.contained_by()`到新的`Range`数据对象中，这些方法反映了 PostgreSQL `@>` 和 `<@` 运算符的行为，以及`comparator_factory.contains()`和`comparator_factory.contained_by()` SQL 运算符方法。感谢 Lele Gaifax 的拉取请求。

    参考：[#8706](https://www.sqlalchemy.org/trac/ticket/8706)

+   **[postgresql] [用例]**

    优化了对于 PostgreSQL 后端的新 RANGE / MULTIRANGE 支持和更改中描述的范围对象新方法，以适应特定于驱动程序的范围和多范围对象，更好地适应遗留代码以及在将原始 SQL 结果集返回到新的范围或多范围表达式时的情况。

    参考：[#8690](https://www.sqlalchemy.org/trac/ticket/8690)

### mssql

+   **[mssql] [bug]**

    修复了针对具有 SQL Server 方言的临时表使用 `Inspector.has_table()` 时，在某些 Azure 变体上失败的问题，原因是不支持这些服务器版本上的不必要的信息模式查询。感谢 Mike Barry 提供的拉取请求。

    此更改也被**回溯**到：1.4.43

    参考：[#8714](https://www.sqlalchemy.org/trac/ticket/8714)

+   **[mssql] [bug] [reflection]**

    修复了针对具有 SQL Server 方言的视图使用 `Inspector.has_table()` 时错误地返回`False`的问题，原因是 1.4 系列中的回归删除了对此的支持。在使用不同反射架构的 2.0 系列中不存在此问题。添加了测试支持，以确保 `has_table()` 在视图方面的工作符合规范。

    此更改也被**回溯**到：1.4.43

    参考：[#8700](https://www.sqlalchemy.org/trac/ticket/8700)

### oracle

+   **[oracle] [bug]**

    修复了绑定参数名称（包括自动从同名数据库列派生的参数名称）中包含通常需要在 Oracle 中用引号引起的字符的问题，在使用 Oracle 方言的“扩展参数”时，这些字符将不会被转义，导致执行错误。Oracle 方言用于绑定参数的通常“引用”不适用于“扩展参数”架构，因此改为使用一系列特定于 Oracle 的字符/转义进行转义。

    此更改也被**回溯**到：1.4.43

    参考：[#8708](https://www.sqlalchemy.org/trac/ticket/8708)

+   **[oracle] [bug]**

    修复了在第一次连接时查询`nls_session_parameters`视图以获取默认小数点字符可能不可用的问题，具体取决于 Oracle 连接模式，并因此引发错误的问题。检测小数点字符的方法已经简化为直接测试小数值，而不是读取系统视图，在任何后端 / 驱动程序上都有效。

    此更改也被**回溯**到：1.4.43

    参考：[#8744](https://www.sqlalchemy.org/trac/ticket/8744)

## 2.0.0b2

发布日期：2022 年 10 月 20 日

### orm

+   **[orm] [bug]**

    移除了在使用 ORM 启用的 update/delete 时发出的关于按名称评估列的警告，最初在[#4073](https://www.sqlalchemy.org/trac/ticket/4073)中添加；这个警告实际上掩盖了一个场景，否则可能会根据实际列的内容为 ORM 映射属性填充错误的 Python 值，因此移除了这个已弃用的情况。在 2.0 版本中，ORM 启用的 update/delete 使用“auto”作为“synchronize_session”，这应该自动为任何给定的 UPDATE 表达式执行正确的操作。

    参考：[#8656](https://www.sqlalchemy.org/trac/ticket/8656)

### orm declarative

+   **[orm] [declarative] [usecase]**

    添加了对作为`Generic`子类的映射类的支持，可以在语句和调用`inspect()`时指定为`GenericAlias`对象（例如`MyClass[str]`）。

    参考：[#8665](https://www.sqlalchemy.org/trac/ticket/8665)

+   **[orm] [declarative] [bug]**

    改进了`DeclarativeBase`类，使其与其他混入类（如`MappedAsDataclass`）结合时，类的顺序可以是任意的。

    参考：[#8665](https://www.sqlalchemy.org/trac/ticket/8665)

+   **[orm] [declarative] [bug]**

    修复了新的 ORM 类型化声明映射中的 bug，其中在一对多关系的类型注释中使用`Optional[MyClass]`或类似形式（如`MyClass | None`）未实现，导致错误。还向关系配置文档添加了此用例的文档。

    参考：[#8668](https://www.sqlalchemy.org/trac/ticket/8668)

+   **[orm] [declarative] [bug]**

    修复了新数据类映射功能中的问题，当处理覆盖`mapped_column()`声明的混入类时，传递给数据类 API 的参数有时可能被错误排序，导致初始化问题。

    参考：[#8688](https://www.sqlalchemy.org/trac/ticket/8688)

### sql

+   **[sql] [bug] [regression]**

    修复了新“insertmanyvalues”功能中的 bug，其中包含使用`bindparam()`的子查询的 INSERT 在“insertmanyvalues”格式中无法正确呈现的问题。这主要影响 psycopg2，因为“insertmanyvalues”在此驱动程序中无条件地使用。

    参考：[#8639](https://www.sqlalchemy.org/trac/ticket/8639)

### typing

+   **[typing] [bug]**

    修复了 pylance 严格模式下报告“实例变量覆盖类变量”的类型问题，当使用方法定义`__tablename__`、`__mapper_args__`或`__table_args__`时。

    参考：[#8645](https://www.sqlalchemy.org/trac/ticket/8645)

+   **[typing] [bug]**

    修复了 pylance 严格模式报告的数据类型“部分未知”的打字问题，对于`mapped_column()`构造。

    参考：[#8644](https://www.sqlalchemy.org/trac/ticket/8644)

### mssql

+   **[mssql] [bug]**

    修复了由 SQL Server pyodbc 改变引起的回归问题 [#8177](https://www.sqlalchemy.org/trac/ticket/8177)，我们现在默认使用 `setinputsizes()`；对于 VARCHAR，如果字符大小大于 4000（或 2000，取决于数据），则会失败，因为传入的数据类型是 NVARCHAR，它的字符限制为 4000 个字符，尽管 VARCHAR 可以处理无限字符。当数据类型的大小>2000 个字符时，现在会将额外的 pyodbc 特定的类型信息传递给`setinputsizes()`。此更改也适用于 `JSON` 类型，对于大型 JSON 序列化也受到此问题的影响。

    参考：[#8661](https://www.sqlalchemy.org/trac/ticket/8661)

+   **[mssql] [bug]**

    `Sequence` 构造将自身恢复到 1.4 系列之前的 DDL 行为，其中创建没有其他参数的 `Sequence` 将发出一个简单的 `CREATE SEQUENCE` 指令 **而不带有** 任何额外的“开始值”参数。对于大多数后端，这是以前的工作方式；**然而**，对于 MS SQL Server，默认值是 `-2**63`；为了防止此通常不切实际的默认值在 SQL Server 上生效，应该提供 `Sequence.start` 参数。由于多年来对 SQL Server 的标准化采用了`IDENTITY`，因此希望这种更改影响较小。

    另见

    序列构造不再具有任何显式默认的“开始”值；影响 MS SQL Server

    参考：[#7211](https://www.sqlalchemy.org/trac/ticket/7211)

### orm

+   **[orm] [bug]**

    移除了使用 ORM 启用的更新/删除时发出的警告，该警告最初在[#4073](https://www.sqlalchemy.org/trac/ticket/4073)中添加；这个警告实际上掩盖了一个场景，否则可能会根据实际列而为 ORM 映射的属性填充错误的 Python 值，因此此弃用的情况已被移除。在 2.0 中，ORM 启用的更新/删除使用“auto”作为“synchronize_session”，这应该自动处理任何给定的更新表达式。

    参考：[#8656](https://www.sqlalchemy.org/trac/ticket/8656)

### orm 声明

+   **[orm] [declarative] [usecase]**

    添加了对作为`Generic`子类的映射类的支持，可以在语句和对 `inspect()` 的调用中指定为`GenericAlias`对象（例如 `MyClass[str]`）。

    参考：[#8665](https://www.sqlalchemy.org/trac/ticket/8665)

+   **[orm] [declarative] [bug]**

    改进了 `DeclarativeBase` 类，使其与其他混合类（如 `MappedAsDataclass`）结合时，类的顺序可以是任意的。

    参考：[#8665](https://www.sqlalchemy.org/trac/ticket/8665)

+   **[orm] [declarative] [bug]**

    修复了新的 ORM 类型化声明映射中使用 `Optional[MyClass]` 或类似形式（如 `MyClass | None`）的类型注释来定义多对一关系时未实现的问题，导致错误。文档还为此用例添加了关系配置文档。

    参考：[#8668](https://www.sqlalchemy.org/trac/ticket/8668)

+   **[orm] [declarative] [bug]**

    修复了新数据类映射功能的问题，当处理覆盖`mapped_column()`声明的混合类时，有时会出现参数被错误排序的情况，导致初始化问题。

    参考：[#8688](https://www.sqlalchemy.org/trac/ticket/8688)

### sql

+   **[sql] [bug] [regression]**

    修复了新的“insertmanyvalues”功能中的 bug，在包含 `bindparam()` 的子查询的 INSERT 中，以“insertmanyvalues”格式正确呈现时会失败。这主要直接影响到 psycopg2，因为“insertmanyvalues”在此驱动程序中无条件使用。

    参考：[#8639](https://www.sqlalchemy.org/trac/ticket/8639)

### typing

+   **[typing] [bug]**

    修复了 pylance 严格模式报告“实例变量覆盖类变量”时的类型问题，该问题出现在使用方法定义`__tablename__`、`__mapper_args__`或`__table_args__`时。

    参考：[#8645](https://www.sqlalchemy.org/trac/ticket/8645)

+   **[typing] [bug]**

    修复了 pylance 严格模式报告 `mapped_column()` 构造的“部分未知”数据类型的类型问题。

    参考：[#8644](https://www.sqlalchemy.org/trac/ticket/8644)

### mssql

+   **[mssql] [bug]**

    由 SQL Server pyodbc 更改引起的回归已修复 [#8177](https://www.sqlalchemy.org/trac/ticket/8177) 现在默认使用 `setinputsizes()`；对于 VARCHAR，如果字符大小大于 4000（或 2000，取决于数据），则会失败，因为传入的数据类型是 NVARCHAR，其限制为 4000 个字符，尽管 VARCHAR 可以处理无限字符。当数据类型的大小 > 2000 个字符时，现在还会将额外的 pyodbc 特定的类型信息传递给 `setinputsizes()`。这一变化也适用于受此问题影响的大型 JSON 序列化的 `JSON` ���型。

    参考：[#8661](https://www.sqlalchemy.org/trac/ticket/8661)

+   **[mssql] [bug]**

    `Sequence` 构造在 1.4 系列之前恢复了其 DDL 行为，其中创建一个没有额外参数的 `Sequence` 将发出一个简单的 `CREATE SEQUENCE` 指令，**没有** 任何额外的“起始值”参数。对于大多数后端，无论如何，这就是以前的工作方式；**然而**，对于 MS SQL Server，该数据库上的默认值是 `-2**63`；为了防止这个通常不切实际的默认值在 SQL Server 上生效，应提供 `Sequence.start` 参数。由于在 SQL Server 中使用 `Sequence` 是不寻常的，多年来一直在 `IDENTITY` 上标准化，希望这一变化影响最小。

    参见

    Sequence 构造恢复为没有任何显式默认的“起始”值；影响 MS SQL Server

    参考：[#7211](https://www.sqlalchemy.org/trac/ticket/7211)

## 2.0.0b1

发布日期：2022 年 10 月 13 日

### 一般

+   **[general] [changed]**

    迁移代码库以删除所有在 2.0 版本中被标记为弃用并将被移除的预 2.0 行为和架构，包括但不限于：

    +   移除所有 Python 2 代码，最低版本现在是 Python 3.7

    +   `Engine` 和 `Connection` 现在使用新的 2.0 风格工作，其中包括 “autobegin”，库级别的 autocommit 已移除，子事务和 “branched” 连接已移除

    +   结果对象使用 2.0 风格行为；`Row` 完全是一个命名元组，没有“映射”行为，使用 `RowMapping` 用于“映射”行为

    +   SQLAlchemy 中的所有 Unicode 编码/解码架构已被移除。所有现代 DBAPI 实现都通过 Python 3 透明地支持 Unicode，因此已移除了`convert_unicode`功能以及在 DBAPI`cursor.description`中查找字节串等相关机制。

    +   从`MetaData`、`Table`以及以前可能引用“绑定引擎”的所有 DDL/DML/DQL 元素中删除了`.bind`属性和参数。

    +   独立的`sqlalchemy.orm.mapper()`函数已移除；所有经典映射应通过`registry`的`registry.map_imperatively()`方法完成。

    +   `Query.join()`方法不再接受关系名称的字符串；使用`Class.attrname`作为连接目标的长期记录的方法现在已成为标准。

    +   `Query.join()`不再接受“aliased”和“from_joinpoint”参数。

    +   `Query.join()`不再接受一次方法调用中的多个连接目标链。

    +   移除了`Query.from_self()`，`Query.select_entity_from()`和`Query.with_polymorphic()`。

    +   `relationship.cascade_backrefs`参数现在必须保持其新默认值为`False`；`save-update`级联不再沿着反向引用进行级联。

    +   参数`Session.future`必须始终设置为`True`。 `Session`的事务模式现在始终是 2.0 风格。

    +   加载器选项不再接受属性名称的字符串。使用`Class.attrname`作为加载器选项目标的长期记录的方法现在已经成为标准。

    +   移除了`select()`的旧形式，包括`select([cols])`，`some_table.select()`的“whereclause”和关键字参数。

    +   移除了`Select`上的旧“就地变更器”方法，如`append_whereclause()`、`append_order_by()`等。

    +   删除了非常古老的“dbapi_proxy”模块，这个模块在早期的 SQLAlchemy 版本中用于在原始 DBAPI 连接上提供透明的连接池。

    参考：[#7257](https://www.sqlalchemy.org/trac/ticket/7257)

+   **[通用] [更改]**

    `Query.instances()` 方法已弃用。该方法的行为约定，即它可以通过任意结果集迭代对象，早已过时且不再受测试。可以通过使用诸如 :meth`.Select.from_statement` 或 `aliased()` 等构造来使任意语句返回对象。

### 平台

+   **[平台] [功能]**

    SQLAlchemy 的 C 扩展现已全部替换为用 Cython 编写的全新实现。与之前的 C 扩展一样，在 pypi 上提供了广泛平台的预构建 wheel 文件，因此对于常见平台来说构建不是问题。对于自定义构建，`python setup.py build_ext` 与以前一样工作，只需要额外的 Cython 安装。`pyproject.toml` 现在也是源码的一部分，当使用 pip 时将建立正确的构建依赖关系。

    另请参阅

    C 扩展现已迁移到 Cython

    参考：[#7256](https://www.sqlalchemy.org/trac/ticket/7256)

+   **[平台] [变更]**

    SQLAlchemy 的源码构建和安装现在包括一个 `pyproject.toml` 文件，以完全支持 [**PEP 517**](https://peps.python.org/pep-0517/)。

    另请参阅

    安装现在完全支持 pep-517

    参考：[#7311](https://www.sqlalchemy.org/trac/ticket/7311)

### orm

+   **[orm] [功能] [sql]**

    为所有支持 RETURNING 的包含方言添加了一个名为“insertmanyvalues”的新功能。这是对首次在 1.4 版本中为 psycopg2 驱动程序引入的“fast executemany”功能的泛化，详情请参见 ORM Batch inserts with psycopg2 now batch statements with RETURNING in most cases，它允许 ORM 将 INSERT 语句批量处理为更高效的 SQL 结构，同时仍能够使用 RETURNING 获取新生成的主键和 SQL 默认值。

    现在该功能适用于许多支持 RETURNING 和多个 VALUES 构造用于 INSERT 的方言，包括所有 PostgreSQL 驱动程序、SQLite、MariaDB、MS SQL Server。另外，Oracle 方言也通过使用本机 cx_Oracle 或 OracleDB 功能获得了相同的能力。

    参考：[#6047](https://www.sqlalchemy.org/trac/ticket/6047)

+   **[orm] [功能]**

    新增了新参数`AttributeEvents.include_key`，将包括字典或列表操作的键，例如`__setitem__()`（例如`obj[key] = value`）和`__delitem__()`（例如`del obj[key]`），使用一个新的关键字参数“key”或“keys”，取决于事件，例如`AttributeEvents.append.key`，`AttributeEvents.bulk_replace.keys`。这允许事件处理程序考虑传递给操作的键，对于与`MappedCollection`一起工作的字典操作特别重要。

    参考：[#8375](https://www.sqlalchemy.org/trac/ticket/8375)

+   **[orm] [feature]**

    新增了新参数`Operators.op.python_impl`，可在`Operators.op()`以及直接使用`custom_op`构造函数时使用，允许提供一个在 Python 中进行评估的函数，以及自定义 SQL 操作符。当操作符对象在两侧使用普通 Python 对象作为操作数时，此评估函数将成为使用的实现，并且特别兼容与 ORM 启用的 INSERT、UPDATE 和 DELETE 语句一起使用的`synchronize_session='evaluate'`选项。

    参考：[#3162](https://www.sqlalchemy.org/trac/ticket/3162)

+   **[orm] [feature]**

    `Session`（以及扩展的`AsyncSession`）现在具有新的状态跟踪功能，将主动捕获在特定事务方法进行时发生的任何意外状态更改。这是为了允许`Session`在线程不安全的方式中使用时，事件钩子或类似的可能在操作中调用意外方法，以及在其他并发情况下（如 asyncio 或 gevent）在非法访问首次发生时引发信息性消息，而不是悄悄地传递导致由于`Session`处于无效状态而导致次要故障。

    另请参阅

    当检测到非法并发或可重入访问时，会主动引发会话错误

    参考：[#7433](https://www.sqlalchemy.org/trac/ticket/7433)

+   **[orm] [功能]**

    `composite()` 映射构造现在支持与 Python `dataclass` 结合使用时的值自动解析；不再需要实现 `__composite_values__()` 方法，因为该方法是从数据类的检查中派生出来的。

    此外，由 `composite` 映射的类现在支持排序比较操作，例如 `<`、`>=` 等。

    请参阅 复合列类型 的新文档以获取示例。

+   **[orm] [功能]**

    添加了非常实验性的功能到 `selectinload()` 和 `immediateload()` 加载器选项中，称为 `selectinload.recursion_depth` / `immediateload.recursion_depth`，它允许单个加载器选项自动递归到自引用关系。设置为指示深度的整数，并且也可以设置为 -1 以指示继续加载直到不再找到更深的级别为止。 `selectinload()` 和 `immediateload()` 的重大内部更改使此功能能够继续正确使用编译缓存，并且不使用任意递归，因此支持任何级别的深度（尽管会发出相应数量的查询）。对于必须完全急切加载的自引用结构，这可能很有用，例如在使用 asyncio 时。

    当在相关对象加载中检测到过度递归深度时（即，未使用新的 `recursion_depth` 选项连接加载器选项的任意长度），还会发出警告。此操作继续使用大量内存，并且性能极差；当检测到此条件时，将禁用缓存以防止缓存被任意语句淹没。

    参考：[#8126](https://www.sqlalchemy.org/trac/ticket/8126)

+   **[orm] [功能]**

    添加了新参数`Session.autobegin`，当设置为`False`时，将阻止`Session`隐式开始事务。必须首先显式调用`Session.begin()`方法才能继续操作，否则在任何操作本应自动开始事务时都会引发错误。此选项可用于创建一个“安全”的`Session`，不会隐式启动新事务。

    作为这一变化的一部分，还添加了一个新的状态变量`origin`，这对于事件处理代码了解特定`SessionTransaction`的来源可能很有用。

    参考：[#6928](https://www.sqlalchemy.org/trac/ticket/6928)

+   **[orm] [功能]**

    不再需要使用`declared_attr()`来实现此映射的声明性混合，其中使用包含`ForeignKey`引用的`Column`对象；当将列应用于声明的映射时，`ForeignKey`对象将与`Column`本身一起复制。

+   **[orm] [用例]**

    添加了`load_only.raiseload`参数到`load_only()`加载器选项中，以便未加载的属性可能具有“raise”行为而不是惰性加载。以前，使用`load_only()`选项没有直接的方法来实现这一点。

+   **[orm] [更改]**

    为了更好地适应显式类型化，一些通常在内部构造但有时也可见于消息传递和类型化的 ORM 构造的名称已更改为更简洁的名称，这些名称也与构造函数的名称（大小写不同）匹配，在所有情况下都保留了旧名称的别名以备将来使用：

    +   `RelationshipProperty`现在成为主要名称`Relationship`的别名，始终使用`relationship()`函数构建

    +   `SynonymProperty`现在成为主要名称`Synonym`的别名，始终使用`synonym()`函数构建

    +   `CompositeProperty`现在成为主要名称`Composite`的别名，始终使用`composite()`函数构建

+   **[orm] [change]**

    为了与突出的 ORM 概念`Mapped`保持一致，字典导向的集合名称，`attribute_mapped_collection()`、`column_mapped_collection()`和`MappedCollection`的名称被更改为`attribute_keyed_dict()`、`column_keyed_dict()`和`KeyFuncDict`，使用短语“dict”以减少与术语“mapped”造成的任何混淆。旧名称将无限期保留，没有移除计划。

    参考：[#8608](https://www.sqlalchemy.org/trac/ticket/8608)

+   **[orm] [bug]**

    所有`Result`对象现在在强制关闭后，无论是在调用“单行或值”方法（如`Result.first()`和`Result.scalar()`）后的“硬关闭”还是其他情况下，都将一致地引发`ResourceClosedError`。这已经是针对核心语句执行返回的最常见类别的结果对象的行为，即基于`CursorResult`的对象，因此这种行为并不新鲜。但是，该更改已经扩展到适当地适应使用 2.0 样式 ORM 查询时返回的 ORM“过滤”结果对象，以前这些对象可能会以返回空结果的“软关闭”方式运行，或者根本不会“软关闭”，并会继续从基础游标中产生结果。

    作为这一变化的一部分，还在基本的`Result`类中添加了`Result.close()`，并且对 ORM 使用的过滤结果实现进行了实现，这样就可以在使用`yield_per`执行选项关闭服务器端游标之前关闭基础的`CursorResult`并获取到剩余的 ORM 结果。这已经在核心结果集中可用，但是此更改还使得在 2.0 样式 ORM 结果中也可用。

    此更改也**回溯**到：1.4.27

    参考：[#7274](https://www.sqlalchemy.org/trac/ticket/7274)

+   **[orm] [bug]**

    修复了`registry.map_declaratively()`方法返回内部“映射器配置”对象而不是 API 文档中所述的`Mapper`对象的问题。

+   **[orm] [bug]**

    修复了至少在 1.3 版本中出现的性能回归（如果不是更早，在 1.0 版本之后的某个时候），在加载延迟列时出现，这些列明确地使用 `defer()` 进行映射，而不是已过期的未延迟列，从联合继承的子类。此时不会使用“优化”的查询，该查询仅查询包含未加载列的直接表，而是运行一个完整的 ORM 查询，该查询会为所有基本表发出 JOIN，当只从子类加载列时，这是不必要的。

    参考：[#7463](https://www.sqlalchemy.org/trac/ticket/7463)

+   **[orm] [bug]**

    `Load` 对象及相关加载器策略模式的内部大部分被重写，以利用现在仅支持属性绑定路径而不支持字符串的事实。这次重写希望更容易解决加载器策略系统中的新用例和细微问题。

    参考：[#6986](https://www.sqlalchemy.org/trac/ticket/6986)

+   **[orm] [bug]**

    对“deferred”/“load_only”一组策略选项进行了改进，在一个查询中从两个不同的逻辑路径加载某个对象时，至少有一个选项配置了要填充的属性将在所有情况下填充，即使该对象的其他加载路径没有设置此选项。之前，基于随机性决定哪个“路径”先处理该对象。

    参考：[#8166](https://www.sqlalchemy.org/trac/ticket/8166)

+   **[orm] [bug]**

    修复了 ORM 启用的 UPDATE 中的问题，当语句针对一个联合继承的子类创建时，只更新本地表列时，“fetch” 同步策略将不会为使用 RETURNING 进行抓取同步的数据库呈现正确的 RETURNING 子句。还调整了在 UPDATE FROM 和 DELETE FROM 语句中使用的 RETURNING 策略。

    参考：[#8344](https://www.sqlalchemy.org/trac/ticket/8344)

+   **[orm] [bug] [asyncio]**

    从 `begin` 和 `begin_nested` 中删除了未使用的 `**kw` 参数。这些 kw 没有被使用，看起来是错误地添加到 API 中的。

    参考：[#7703](https://www.sqlalchemy.org/trac/ticket/7703)

+   **[orm] [bug]**

    更改了由`attribute_mapped_collection()`和`column_mapped_collection()`使用的属性访问方法（现在称为`attribute_keyed_dict()`和`column_keyed_dict()`)，在填充字典时，会断言对象上的数据值实际上存在，而不是由于属性从未被实际分配而使用“None”。这用于在对象的“键”属性尚未分配时，通过反向引用进行分配时，防止将“None”错误地赋值为键。

    由于此处的故障模式是一个通常不会持久到数据库的暂时条件，并且很容易通过类的构造函数根据参数分配的顺序产生，因此很可能许多应用程序已经包含了这种行为，而这种行为是悄悄地被忽略的。为了适应现在引发此错误的应用程序，还添加了一个新参数`attribute_keyed_dict.ignore_unpopulated_attribute`到`attribute_keyed_dict()`和`column_keyed_dict()`中，它会导致错误的反向引用赋值被跳过。

    参考：[#8372](https://www.sqlalchemy.org/trac/ticket/8372)

+   **[orm] [bug]**

    添加了新参数`AbstractConcreteBase.strict_attrs`到`AbstractConcreteBase`声明混合类。此参数的影响是，子类上的属性范围正确地限制在声明每个属性的子类中，而不是之前的行为，其中整个层次结构的所有属性都应用于基本“抽象”类。这产生了一个更干净、更正确的映射，其中子类不再具有仅与兄弟类相关的非有用属性。此参数的默认值为 False，这保持了以前的行为不变；这是为了支持使用这些属性进行查询的现有代码。要迁移到更新的方法，根据需要将显式属性应用于抽象基类。

    参考：[#8403](https://www.sqlalchemy.org/trac/ticket/8403)

+   **[orm] [bug]**

    `defer()`关于主键和“多态识别器”列的行为已修订，使得这些列不再可延迟，无论是明确还是使用通配符如`defer('*')`。以前，通配符延迟不会加载 PK/多态列，这导致在所有情况下都会出错，因为 ORM 依赖于这些列来生成对象标识。主键列的显式延迟的行为不变，因为这些延迟已经被隐式忽略。

    参考：[#7495](https://www.sqlalchemy.org/trac/ticket/7495)

+   **[orm] [bug]**

    修复了`Mapper.eager_defaults`参数行为中的错误，以便在仅使用表定义中的客户端端 SQL 默认值或 onupdate 表达式时，当 ORM 为行发出 INSERT 或 UPDATE 时触发 RETURNING 或 SELECT 的抓取操作。以前，只有作为表 DDL 的一部分或服务器端 onupdate 表达式建立的服务器端默认值会触发此抓取，即使在呈现抓取时也会包含客户端端 SQL 表达式。

    参考：[#7438](https://www.sqlalchemy.org/trac/ticket/7438)

### engine

+   **[engine] [feature]**

    `DialectEvents.handle_error()` 事件现已从 `EngineEvents` 套件移至 `DialectEvents` 套件，并且现在参与连接池的“预 ping”事件，用于使用断开代码来检测数据库是否活动的方言。这允许最终用户代码更改“预 ping”的状态。请注意，这不包括包含原生“ping”方法的方言，如 psycopg2 或大多数 MySQL 方言。

    参考：[#5648](https://www.sqlalchemy.org/trac/ticket/5648)

+   **[engine] [feature]**

    `ConnectionEvents.set_connection_execution_options()` 和 `ConnectionEvents.set_engine_execution_options()` 事件挂钩现在允许就地修改给定的选项字典，新内容将作为最终执行选项接收。以前，不支持对字典进行就地修改。

+   **[engine] [usecase]**

    将 `create_engine.isolation_level` 参数泛化到基本方言，因此不再依赖于个别方言的存在。该参数设置“隔离级别”设置以在连接池创建新数据库连接时立即发生，然后值保持设置而不在每次检查时重置。

    `create_engine.isolation_level` 参数在功能上基本等同于通过 `Engine.execution_options.isolation_level` 参数使用 `Engine.execution_options()` 进行引擎范围设置。区别在于前者设置在创建连接时只分配一次隔离级别，而后者在每次连接检出时设置和重置给定级别。

    参考：[#6342](https://www.sqlalchemy.org/trac/ticket/6342)

+   **[engine] [change]**

    有关引擎和方言的一些小型 API 更改：

    +   `Dialect.set_isolation_level()`，`Dialect.get_isolation_level()`，:meth: 方言方法将始终传递原始的 DBAPI 连接

    +   `Connection`和`Engine`类不再共享基础的`Connectable`超类，该超类已被移除。

    +   添加了一个新的接口类`PoolProxiedConnection` - 这是熟悉的`_ConnectionFairy`类的公共接口，尽管它是一个私有类。

    参考：[#7122](https://www.sqlalchemy.org/trac/ticket/7122)

+   **[引擎] [错误] [回归]**

    修复了`CursorResult.fetchmany()`方法的回归，当结果完全耗尽时，服务器端游标（即在使用`stream_results`或`yield_per`时，无论是 Core 还是 ORM 导向的结果）将无法自动关闭。

    此更改也已**回溯**至：1.4.27

    参考：[#7274](https://www.sqlalchemy.org/trac/ticket/7274)

+   **[引擎] [错误]**

    修复了未来`Engine`中的问题，即调用`Engine.begin()`并进入上下文管理器，如果实际的 BEGIN 操作由于某种原因失败，例如事件处理程序引发异常，则连接不会关闭；这种用例未能为未来版本的引擎进行测试。请注意，“未来”上下文管理器处理 Core 和 ORM 中的`begin()`块，直到实际进入上下文管理器时才运行“BEGIN”操作。这与立即运行“BEGIN”操作的传统版本不同。

    此更改也已**回溯**至：1.4.27

    参考：[#7272](https://www.sqlalchemy.org/trac/ticket/7272)

+   **[引擎] [错误]**

    `QueuePool`现在在`pool_size=0`时忽略`max_overflow`，在所有情况下正确地使池无限制。

    参考：[#8523](https://www.sqlalchemy.org/trac/ticket/8523)

+   **[引擎] [错误]**

    为了提高安全性，当调用 `str(url)` 时，`URL` 对象现在默认使用密码混淆。要以明文密码字符串化 URL，可以使用 `URL.render_as_string()`，传递 `URL.render_as_string.hide_password` 参数为 `False`。感谢我们的贡献者提供此拉取请求。

    另请参阅

    str(engine.url) 默认情况下将混淆密码

    参考：[#8567](https://www.sqlalchemy.org/trac/ticket/8567)

+   **[引擎] [错误]**

    `Inspector.has_table()` 方法现在将一致性地检查给定名称的视图以及表。先前，这种行为取决于方言，PostgreSQL、MySQL/MariaDB 和 SQLite 支持它，而 Oracle 和 SQL Server 不支持它。第三方方言也应确保他们的 `Inspector.has_table()` 方法搜索给定名称的视图和表。

    参考：[#7161](https://www.sqlalchemy.org/trac/ticket/7161)

+   **[引擎] [错误]**

    修复了 `Result.columns()` 方法中的问题，在某些情况下，特别是 ORM 结果对象的情况下，调用带有单个索引的 `Result.columns()` 可能导致 `Result` 产生标量对象而不是 `Row` 对象，就像调用了 `Result.scalars()` 方法一样。在 SQLAlchemy 1.4 中，这种情况会发出警告，指出行为将在 SQLAlchemy 2.0 中更改。

    参考：[#7953](https://www.sqlalchemy.org/trac/ticket/7953)

+   **[引擎] [错误]**

    将`DefaultGenerator`对象（如`Sequence`）传递给`Connection.execute()`方法已弃用，因为该方法被定义为返回`CursorResult`对象，而不是普通标量值。应改用`Connection.scalar()`方法，该方法已重新设计为通过新的内部代码路径适用于调用 SELECT 以获取默认生成对象，而无需通过`Connection.execute()`方法。

+   **[engine] [已移除]**

    已从`create_engine()`中移除先前已弃用的`case_sensitive`参数，该参数仅影响 Core-only 结果集行中字符串列名的查找；它对 ORM 的行为没有影响。`case_sensitive`所指向的有效行为保持其默认值`True`，意味着在`row._mapping`中查找的字符串名称将区分大小写，就像任何其他 Python 映射一样。

    请注意，`case_sensitive`参数与一般的大小写敏感性控制、引用和 DDL 标识符名称的“名称规范化”（即转换为将所有大写单词视为不区分大小写的数据库）并无任何关系，后者仍然是 SQLAlchemy 的常规核心功能。

+   **[engine] [已移除]**

    已移除旧版和已弃用的包`sqlalchemy.databases`。请改用`sqlalchemy.dialects`。

    参考：[#7258](https://www.sqlalchemy.org/trac/ticket/7258)

+   **[engine] [弃用]**

    `create_engine.implicit_returning` 参数仅在 `create_engine()` 函数中已弃用；该参数仍然可用于 `Table` 对象。该参数最初旨在在 SQLAlchemy 首次开发时启用“隐式返回”功能，并且不是默认启用的。在现代用法中，没有理由禁用此参数，并且已观察到它会引起混乱，因为它会降低性能，并使 ORM 更难以检索最近插入的服务器默认值。该参数仍然可用于`Table`，以特别适应使 RETURNING 不可行的数据库级边缘情况，目前唯一的示例是 SQL Server 的限制，即不能在具有 INSERT 触发器的表上使用 INSERT RETURNING。

    参考：[#6962](https://www.sqlalchemy.org/trac/ticket/6962)

### sql

+   **[sql] [feature]**

    > 添加了长期请求的不区分大小写的字符串操作符 `ColumnOperators.icontains()`, `ColumnOperators.istartswith()`, `ColumnOperators.iendswith()`，它们生成不区分大小写的 LIKE 组合（在 PostgreSQL 上使用 ILIKE，在所有其他后端上使用 LOWER()函数），以补充现有的 LIKE 组合操作符 `ColumnOperators.contains()`, `ColumnOperators.startswith()` 等。特别感谢 Matias Martinez Rebori 对实现这些新方法的细致和完整的努力。

    参考：[#3482](https://www.sqlalchemy.org/trac/ticket/3482)

+   **[sql] [feature]**

    在所有`FromClause`对象的`FromClause.c`集合上添加了新的语法，允许将键的元组传递给`__getitem()`，以及支持`select()`构造来直接处理结果类似元组的集合，从而使得`select(table.c['a', 'b', 'c'])`这样的语法成为可能。返回的子集合本身是一个`ColumnCollection`，现在也可以直接被`select()`和类似的函数使用。

    另请参阅

    设置 COLUMNS 和 FROM 子句

    参考：[#8285](https://www.sqlalchemy.org/trac/ticket/8285)

+   **[sql] [特性]**

    从 PostgreSQL 方言泛化的新的与后端无关的`Uuid`数据类型现在成为核心类型，同时将`UUID`从 PostgreSQL 方言迁移过来。SQL Server 的`UNIQUEIDENTIFIER`数据类型也成为了一个处理 UUID 的数据类型。感谢 Trevor Gross 的帮助。

    参考：[#7212](https://www.sqlalchemy.org/trac/ticket/7212)

+   **[sql] [特性]**

    在基本的`sqlalchemy.`模块命名空间中添加了`Double`、`DOUBLE`、`DOUBLE_PRECISION`数据类型，用于明确使用双精度/双精度以及通用的“double”数据类型。使用`Double`来获得根据不同后端需要解析为 DOUBLE/DOUBLE PRECISION/FLOAT 的通用支持。

    参考：[#5465](https://www.sqlalchemy.org/trac/ticket/5465)

+   **[sql] [用例]**

    修改了`Insert`构造的编译机制，以便在特定后端的单行 INSERT 语句上获取“自增主键”列值，即使该值存在于参数集或`Insert.values()`方法中作为普通绑定值。这恢复了 1.3 系列中的行为，适用于单独参数集以及`Insert.values()`的用例。在 1.4 中，参数集行为无意中更改为不再执行此操作，但`Insert.values()`方法仍会获取自增值，直到 1.4.21，其中[#6770](https://www.sqlalchemy.org/trac/ticket/6770)再次无意中更改了行为，因为此用例从未被覆盖。

    现在定义的行为为“有效”，以适应 SQLite、MySQL 和 MariaDB 等数据库忽略显式 NULL 主键值并仍调用自增生成器的情况。

    参考：[#7998](https://www.sqlalchemy.org/trac/ticket/7998)

+   **[sql] [用例]**

    当使用`literal_binds`与由 PostgreSQL、MySQL、MariaDB、MSSQL、Oracle 方言提供的 SQL 编译器时，添加了修改后的 ISO-8601 渲染（即将 T 转换为空格的 ISO-8601）。对于 Oracle，ISO 格式被包装在适当的 TO_DATE()函数调用中。以前，此渲染未针对特定方言的编译实现。

    另请参阅

    DATE、TIME、DATETIME 数据类型现在在所有后端上支持文字渲染

    参考：[#5052](https://www.sqlalchemy.org/trac/ticket/5052)

+   **[sql] [用例]**

    添加了新参数`HasCTE.add_cte.nest_here`到`HasCTE.add_cte()`，该参数将在父语句级别“嵌套”给定的`CTE`。该参数等同于使用`HasCTE.cte.nesting`参数，但在某些情况下可能更直观，因为它允许同时设置嵌套属性和 CTE 的显式级别。

    `HasCTE.add_cte()` 方法也接受多个 CTE 对象。

    参考资料：[#7759](https://www.sqlalchemy.org/trac/ticket/7759)

+   **[sql] [bug]**

    当使用 `Select.select_from()` 方法在一个 `select()` 构造上建立 FROM 子句时，现在这些子句将首先在所渲染的 SELECT 的 FROM 子句中呈现，这有助于保持子句的顺序与传递给 `Select.select_from()` 方法本身时一致，而不受这些子句也被提及在查询的其他部分中的影响。如果 `Select` 的其他元素也生成 FROM 子句，比如 columns 子句或 WHERE 子句，那么这些子句将在由 `Select.select_from()` 提供的子句之后呈现，假设它们没有显式地传递给 `Select.select_from()`。这种改进在某些情况下很有用，其中特定的数据库基于 FROM 子句的特定顺序生成一个理想的查询计划，并允许完全控制 FROM 子句的顺序。

    参考资料：[#7888](https://www.sqlalchemy.org/trac/ticket/7888)

+   **[sql] [bug]**

    `Enum.length` 参数，用于为非本机枚举类型的 `VARCHAR` 列设置长度，现在无条件地在发出 `VARCHAR` 数据类型的 DDL 时使用，包括当目标后端继续使用 `VARCHAR` 时设置 `Enum.native_enum` 参数为 `True` 的情况。在这种情况下，以前会错误地忽略该参数。先前针对这种情况发出的警告现在已删除。

    参考资料：[#7791](https://www.sqlalchemy.org/trac/ticket/7791)

+   **[sql] [bug]**

    对于 Python 整数的就地类型检测，如`literal(25)`所示，现在还将应用基于值的适配以适应 Python 大整数，其中确定的数据类型将是`BigInteger`而不是`Integer`。这适用于像 asyncpg 这样的方言，它既向驱动程序发送隐式类型信息，又对数字规模敏感。

    参考：[#7909](https://www.sqlalchemy.org/trac/ticket/7909)

+   **[sql] [bug]**

    为所有“创建”/“删除”构造添加了`if_exists`和`if_not_exists`参数，包括`CreateSequence`、`DropSequence`、`CreateIndex`、`DropIndex`等，允许在 DDL 中呈现通用的“IF EXISTS”/“IF NOT EXISTS”短语。感谢 Jesse Bakker 的拉取请求。

    参考：[#7354](https://www.sqlalchemy.org/trac/ticket/7354)

+   **[sql] [bug]**

    改进了 SQL 二进制表达式的构造，以允许针对同一关联操作符的非常长的表达式进行处理，而无需特殊步骤以避免高内存使用和过度递归深度。现在，特定的二进制操作`A op B`可以与另一个元素`op C`连接，并且结果结构将被“扁平化”，以便表示以及 SQL 编译不需要递归。

    此更改的一个效果是使用 SQL 函数的字符串连接表达式变得“扁平”，例如 MySQL 现在将呈现`concat('x', 'y', 'z', ...)`而不是将两个元素函数嵌套在一起，如`concat(concat('x', 'y'), 'z')`。覆盖字符串连接运算符的第三方方言将需要实现一个新方法`def visit_concat_op_expression_clauselist()`来配合现有的`def visit_concat_op_binary()`方法。

    参考：[#7744](https://www.sqlalchemy.org/trac/ticket/7744)

+   **[sql] [bug]**

    使用“/”和“//”运算符完全支持“truediv”和“floordiv”。两个表达式之间的“truediv”操作现在被认为是`Integer`类型的结果，并且方言级别的编译将在方言特定基础上将右操作数转换为数值类型，以确保实现 truediv。对于 floordiv，还添加了转换，以解决那些不默认执行 floordiv 的数据库（MySQL、Oracle）以及在这种情况下呈现`FLOOR()`函数的情况，以及右操作数不是整数的情况（对于 PostgreSQL 等需要的情况）。

    此更改解决了不同后端上除法运算符的不一致行为以及修复了一个问题，即在 Oracle 上的整数除法无法获取结果，因为输出类型处理程序不合适。

    另请参阅

    Python 除法运算符对所有后端执行真除法；增加了地板除法

    参考文献：[#4926](https://www.sqlalchemy.org/trac/ticket/4926)

+   **[sql] [bug]**

    在编译器中添加了一个额外的查找步骤，将跟踪所有 FROM 子句，这些子句是表，可能在其中一个模式中共享相同的名称，其中一个模式是隐式的“默认”模式；在这种情况下，对该名称进行引用时不带模式限定符时，表名称将在编译器级别上使用匿名别名来渲染，以消除两个（或多个）名称的歧义。还考虑了使用服务器检测到的“默认模式名称”值对通常不带模式限定符的名称进行模式限定的方法，但是这种方法不适用于 Oracle，SQL Server 也不接受，也不适用于 PostgreSQL 搜索路径中的多个条目。此处解决的名称冲突问题已被确认至少影响到了 Oracle、PostgreSQL、SQL Server、MySQL 和 MariaDB。

    参考文献：[#7471](https://www.sqlalchemy.org/trac/ticket/7471)

+   **[sql] [bug]**

    当使用`literal()`时，从值的类型确定 SQL 类型的 Python 字符串值，现在将应用`String`类型，而不是`Unicode`数据类型，对于使用 Python `str.isascii()`测试为“仅 ascii”的 Python 字符串值。如果字符串不是`isascii()`，则将绑定`Unicode`数据类型，而以前所有字符串检测中都使用了它。此行为**仅适用于使用``literal()``或其他没有现有数据类型的上下文中的数据类型的就地检测**，这在正常`Column`比较操作下通常不是情况，在这种情况下，被比较的`Column`的类型始终优先。

    在后端如 SQL Server 上，使用`Unicode`数据类型可以确定文字字符串格式，其中文字值（即使用`literal_binds`）将呈现为`N'<value>'`而不是`'value'`。对于正常的绑定值处理，`Unicode`数据类型还可能对将值传递给 DBAPI 产生影响，再次以 SQL Server 为例，pyodbc 驱动程序支持使用 setinputsizes 模式，它将以不同方式处理`String`与`Unicode`。

    参考：[#7551](https://www.sqlalchemy.org/trac/ticket/7551)

+   **[sql] [错误]**

    `array_agg`现在将设置数组维度为 1。改进了对`None`值作为多维数组的值的`ARRAY`处理。

    参考：[#7083](https://www.sqlalchemy.org/trac/ticket/7083)

### 模式

+   **[模式] [特性]**

    扩展了由 `ExecutableDDLElement` 类实现的“条件 DDL”系统（从 `DDLElement` 改名而来），使其直接可用于 `SchemaItem` 构造，例如 `Index`、`ForeignKeyConstraint` 等，以便在默认的 DDL 发出过程中包含生成这些元素的条件逻辑。此系统也可以在将来的 Alembic 版本中进行调整，以支持在所有模式管理系统中支持条件 DDL 元素。

    另请参阅

    新的约束和索引条件 DDL

    参考：[#7631](https://www.sqlalchemy.org/trac/ticket/7631)

+   **[schema] [usecase]**

    向 `DropConstraint` 构造添加了参数 `DropConstraint.if_exists`，该参数导致“IF EXISTS” DDL 被添加到 DROP 语句中。这个短语并不被所有数据库接受，如果一个数据库不支持它，那么该操作将失败，因为在单个 DDL 语句的范围内没有类似兼容的回退。拉取请求由 Mike Fiedler 提供。

    参考：[#8141](https://www.sqlalchemy.org/trac/ticket/8141)

+   **[schema] [usecase]**

    实现了对所有包含单独 CREATE 或 DROP 步骤的 `SchemaItem` 对象的 DDL 事件钩子 `DDLEvents.before_create()`、`DDLEvents.after_create()`、`DDLEvents.before_drop()`、`DDLEvents.after_drop()`，当该步骤被作为单独的 SQL 语句调用时，包括对 `ForeignKeyConstraint`、`Sequence`、`Index` 和 PostgreSQL 的 `ENUM` 的支持。

    参考：[#8394](https://www.sqlalchemy.org/trac/ticket/8394)

+   **[模式] [性能]**

    重构了模式反射 API，以允许参与的方言使用高性能的批量查询来一次反映多个表的模式，通过数量级减少查询。新的性能特性首先针对 PostgreSQL 和 Oracle 后端，可以应用于任何使用对系统目录表进行 SELECT 查询以反映表的方言。该更改还包括对`Inspector`对象的新 API 特性和行为改进，包括一致的、缓存的方法行为，如`Inspector.has_table()`，`Inspector.get_table_names()`以及新方法`Inspector.has_schema()`和`Inspector.has_index()`。

    另请参阅

    数据库反射的主要架构、性能和 API 增强 - 完整背景

    参考：[#4379](https://www.sqlalchemy.org/trac/ticket/4379)

+   **[模式] [错误]**

    使用`Table.include_columns`参数排除然后发现属于这些约束的列而发出的索引或唯一约束的反射警告已被删除。使用`Table.include_columns`参数时，应该预期生成的`Table`构造不包括依赖于被省略列的约束。此更改是响应于[#8100](https://www.sqlalchemy.org/trac/ticket/8100)，该修复与依赖于省略列的外键约束一起使用`Table.include_columns`的用例变得清晰，省略此类约束应该是可以预期的。

    参考：[#8102](https://www.sqlalchemy.org/trac/ticket/8102)

+   **[模式] [postgresql]**

    添加对`Constraint`对象的注释支持，包括 DDL 和反射；字段添加到基本`Constraint`类和相应的构造函数中，但是目前只有 PostgreSQL 包含的后端支持此功能。请参阅参数，例如`ForeignKeyConstraint.comment`、`UniqueConstraint.comment`或`CheckConstraint.comment`。

    参考：[#5677](https://www.sqlalchemy.org/trac/ticket/5677)

+   **[模式] [mariadb] [mysql]**

    在 MySQL 和 MariaDB 中反映选项的支持分区和示例页面。选项存储在表方言选项字典中，因此以下关键字需要根据后端前缀为`mysql_`或`mariadb_`。支持的选项有：

    +   `stats_sample_pages`

    +   `partition_by`

    +   `partitions`

    +   `subpartition_by`

    从数据库加载表时也反映了这些选项，并将填充表`Table.dialect_options`。感谢 Ramon Will 的拉取请求。

    参考：[#4038](https://www.sqlalchemy.org/trac/ticket/4038)

### 打字

+   **[打字] [改进]**

    `TypeEngine.with_variant()`方法现在返回原始`TypeEngine`对象的副本，而不是将其包装在`Variant`类中，这实际上被移除了（导入符号仍保留，以便与可能正在测试此符号的代码向后兼容）。虽然以前的方法维护了 Python 中的行为，但维护原始类型可以实现更清晰的类型检查和调试。

    `TypeEngine.with_variant()`还接受每次调用多个方言名称，特别是这对于相关后端名称如`"mysql", "mariadb"`是有帮助的。

    另请参阅

    `with_variant()`克隆原始 TypeEngine 而不是更改类型

    参考：[#6980](https://www.sqlalchemy.org/trac/ticket/6980)

### postgresql

+   **[postgresql] [特性]**

    添加了一个新的 PostgreSQL `DOMAIN` 数据类型，其遵循与 PostgreSQL `ENUM` 相同的 CREATE TYPE / DROP TYPE 行为。非常感谢 David Baumgold 为此所做的努力。

    另请参阅

    `DOMAIN`

    参考：[#7316](https://www.sqlalchemy.org/trac/ticket/7316)

+   **[postgresql] [用例] [asyncpg]**

    添加了可重写方法`PGDialect_asyncpg.setup_asyncpg_json_codec`和`PGDialect_asyncpg.setup_asyncpg_jsonb_codec`编解码器，用于处理在使用 asyncpg 时为这些数据类型注册 JSON/JSONB 编解码器的必要任务。变化在于这些方法被拆分为单独的、可重写的方法，以支持需要修改或禁用这些特定编解码器设置方式的第三方方言。

    此更改也已**回溯**至：1.4.27

    参考：[#7284](https://www.sqlalchemy.org/trac/ticket/7284)

+   **[postgresql] [用例]**

    为`ARRAY`和`ARRAY`数据类型添加了字面类型渲染。通用的字符串化将使用方括号呈现，例如`[1, 2, 3]`，而特定于 PostgreSQL 的将使用 ARRAY 字面值，例如`ARRAY[1, 2, 3]`。还考虑了多维和引号。

    参考：[#8138](https://www.sqlalchemy.org/trac/ticket/8138)

+   **[postgresql] [用例]**

    增加了对 PostgreSQL 多范围类型的支持，引入于 PostgreSQL 14 中。现在已将对 PostgreSQL 范围和多范围的支持泛化到了 psycopg3、psycopg2 和 asyncpg 后端，还有进一步方言支持的空间，使用与先前使用的 psycopg2 对象兼容的后端不可知`Range`数据对象构造函数。查看新文档以获取使用模式。

    此外，增强了范围类型处理，使其自动呈现类型转换，因此对于不提供数据库任何上下文的语句的原地往返，不需要使用`cast()`构造明确告知数据库所需的类型（在[#8540](https://www.sqlalchemy.org/trac/ticket/8540)讨论）。

    非常感谢@zeeeeeb 为实现和测试新数据类型和 psycopg 支持��拉取请求。

    另请参阅

    PostgreSQL 后端的新 RANGE / MULTIRANGE 支持和更改

    范围和多范围类型

    参考：[#7156](https://www.sqlalchemy.org/trac/ticket/7156)，[#8540](https://www.sqlalchemy.org/trac/ticket/8540)

+   **[postgresql] [usecase]**

    针对 psycopg、asyncpg 和 pg8000 配置 `create_engine.pool_pre_ping` 时发出的“ping”查询，但不适用于 psycopg2，已更改为一个空查询 (`;`)，而不是 `SELECT 1`；此外，对于 asyncpg 驱动程序，已修复了此查询的不必要使用预处理语句。理由是消除 PostgreSQL 在发出 ping 时产生查询计划的需求。当前不支持此操作的 `psycopg2` 驱动程序仍然使用 `SELECT 1`。

    参考：[#8491](https://www.sqlalchemy.org/trac/ticket/8491)

+   **[postgresql] [change]**

    现在 SQLAlchemy 需要 PostgreSQL 版本为 9 或更高版本。在某些有限的用例中，旧版本可能仍然可用。

+   **[postgresql] [change] [mssql]**

    `UUID` 的参数 `UUID.as_uuid`，以前专门针对 PostgreSQL 方言，但现在已经为 Core 通用化（以及一个新的与后端无关的 `Uuid` 数据类型），现在默认为 `True`，表示此数据类型默认接受 Python `UUID` 对象。此外，SQL Server 的 `UNIQUEIDENTIFIER` 数据类型已转换为接收 UUID 类型；对于使用字符串值的旧代码中使用 `UNIQUEIDENTIFIER` 的情况，请将 `UNIQUEIDENTIFIER.as_uuid` 参数设置为 `False`。

    参考：[#7225](https://www.sqlalchemy.org/trac/ticket/7225)

+   **[postgresql] [change]**

    PostgreSQL 特定的 `ENUM` 数据类型的参数 `ENUM.name` 现在是一个必需的关键字参数。无论如何，“name”在任何情况下都是必需的，因为如果“name”不存在，那么在 SQL/DDL 渲染时会引发错误。 

+   **[postgresql] [change]**

    为了支持新的 PostgreSQL 功能，包括 psycopg3 方言以及扩展的“快速插入多个”支持，已重新设计了为绑定参数传递类型信息到 PostgreSQL 数据库的系统，使用 SQL 编译器发出的内联转换，并且现在应用于所有 PostgreSQL 方言。这与以前的方法相反，以前的方法依赖于正在使用的 DBAPI 自行呈现这些转换，例如 pg8000 和适应的 asyncpg 驱动程序，会使用 pep-249 的 `setinputsizes()` 方法，或者在大多数情况下会依赖于驱动程序本身，特殊情况除外，如 ARRAY。

    新方法现在通过编译器在所有 PostgreSQL 方言中呈现这些转换，使用 PostgreSQL 的双冒号样式，并且删除了对于 PostgreSQL 方言的 `setinputsizes()` 的使用，因为这在任何情况下通常不是这些 DBAPI 的一部分（pg8000 是唯一的例外，在 SQLAlchemy 开发人员的请求下添加了该方法）。

    这种方法的优点包括每个语句的性能，因为在执行时不需要对编译后的语句进行第二次遍历，对所有 DBAPI 的更好支持，因为现在有一个一致的应用类型信息的系统，以及改进的透明度，因为 SQL 记录输出，以及编译语句的字符串输出，将直接显示这些转换存在于语句中，而以前这些转换在记录输出中是不可见的，因为它们会在记录语句之后发生。

+   **[postgresql] [bug]**

    `Operators.match()` 操作现在在 PostgreSQL 全文搜索中使用 `plainto_tsquery()`，而不是 `to_tsquery()`。更改的原因是为了提供与其他数据库后端匹配的更好的交叉兼容性。通过与 `Operators.bool_op()` 结合使用 `func`（布尔运算符的改进版本）来获得所有 PostgreSQL 全文函数的完全支持（与 `Operators.op()` 相比）。

    另请参阅

    PostgreSQL 上的 match() 操作现在使用 plainto_tsquery()，而不是 to_tsquery()

    参考：[#7086](https://www.sqlalchemy.org/trac/ticket/7086)

+   **[postgresql] [已删除]**

    删除对多个不推荐使用的驱动程序的支持：

    > +   PostgreSQL 的 pypostgresql 驱动。可在 [`github.com/PyGreSQL`](https://github.com/PyGreSQL) 找到此外部驱动。
    > +   
    > +   PostgreSQL 的 pygresql 驱动。

    请切换到受支持的驱动程序之一，或者切换到相同驱动程序的外部版本。

    参考：[#7258](https://www.sqlalchemy.org/trac/ticket/7258)

+   **[postgresql] [方言]**

    添加对`psycopg`方言的支持，支持同步和异步执行。该方言可在`create_engine()`和`create_async_engine()`引擎创建函数中使用`postgresql+psycopg`名称。

    另请参阅

    对 psycopg 3（又名“psycopg”）的方言支持

    psycopg

    参考：[#6842](https://www.sqlalchemy.org/trac/ticket/6842)

+   **[postgresql] [psycopg2]**

    更新 psycopg2 方言以使用 DBAPI 接口执行两阶段事务。以前，SQL 命令用于处理此类事务。

    参考：[#7238](https://www.sqlalchemy.org/trac/ticket/7238)

+   **[postgresql] [schema]**

    引入了类型`JSONPATH`，可用于转换表达式。在使用 `jsonb_path_exists` 或 `jsonb_path_match` 等接受 `jsonpath` 作为输入的函数时，某些 PostgreSQL 方言需要此类型。

    另请参阅

    JSON 类型 - PostgreSQL JSON 类型。

    参考：[#8216](https://www.sqlalchemy.org/trac/ticket/8216)

+   **[postgresql] [reflection]**

    PostgreSQL 方言现在支持基于表达式的索引的反射。无论是使用 `Inspector.get_indexes()` 还是使用 `Table` 的 `Table.autoload_with` 反射表时，都支持反射。感谢 immerrr 和 Aidan Kane 在此票证上的帮助。

    参考：[#7442](https://www.sqlalchemy.org/trac/ticket/7442)

### mysql

+   **[mysql] [usecase] [mariadb]**

    `ROLLUP` 函数现在将在 MySql 和 MariaDB 上正确呈现 `WITH ROLLUP`，允许在这些后端使用 group by rollup。

    参考：[#8503](https://www.sqlalchemy.org/trac/ticket/8503)

+   **[mysql] [bug]**

    修复了 MySQL `Insert.on_duplicate_key_update()` 中的问题，当在 VALUES 表达式中使用表达式时，会导致错误的列名。感谢 Cristian Sabaila 提供的拉取请求。

    此更改也被**回溯**到：1.4.27

    参考：[#7281](https://www.sqlalchemy.org/trac/ticket/7281)

+   **[mysql] [removed]**

    移除对 MySQL 和 MariaDB 的 OurSQL 驱动程序的支持，因为该驱动程序似乎没有得到维护。

    参考：[#7258](https://www.sqlalchemy.org/trac/ticket/7258)

### mariadb

+   **[mariadb] [usecase]**

    添加了一个新的执行选项 `is_delete_using=True`，当使用 ORM 启用的 DELETE 语句与“fetch”同步策略一起使用时，ORM 会消耗此选项；此选项指示预期 DELETE 语句将使用多个表，在 MariaDB 上是 DELETE..USING 语法。然后，该选项指示不应该使用 RETURNING（在 SQLAlchemy 2.0 中为 MariaDB 新实现的选项）用于已知不支持“DELETE..USING..RETURNING”语法的数据库，即使它们支持“DELETE..USING”，这是 MariaDB 的当前功能。

    该选项的理由是，ORM 启用的 DELETE 在编译之前不知道是否针对多个表，直到编译发生，而编译在任何情况下都会被缓存，但是需要知道是否要事先发出 SELECT 来删除的行。为了避免为所有 DELETE 语句施加全面性能惩罚，因为需要预先检查它们是否都有这种相对不寻常的 SQL 模式，通过一个新的在编译步骤中引发的异常消息请求 `is_delete_using=True` 执行选项。该异常消息仅在以下情况下（并且仅在以下情况下）引发：语句是启用了 ORM 的 DELETE，其中请求了“fetch”同步策略；后端是 MariaDB 或其他具有此特定限制的后端；语句已在初始编译中检测到，否则它会发出“DELETE..USING..RETURNING”。通过应用执行选项，ORM 知道要先运行 SELECT。针对启用了 ORM 的 UPDATE 实现了类似的选项，但目前没有需要此选项的后端。

    参考：[#8344](https://www.sqlalchemy.org/trac/ticket/8344)

+   **[mariadb] [用例]**

    为 MariaDB 方言添加了 INSERT..RETURNING 和 DELETE..RETURNING 支持。UPDATE..RETURNING 目前还不被 MariaDB 支持。MariaDB 从 10.5.0 版本开始支持 INSERT..RETURNING，从 10.0.5 版本开始支持 DELETE..RETURNING。

    参考：[#7011](https://www.sqlalchemy.org/trac/ticket/7011)

### sqlite

+   **[sqlite] [用例]**

    为 SQLite 添加了反射方法的新参数 `sqlite_include_internal=True`；当省略时，不会包含以 `sqlite_` 前缀开头的本地表，这些表根据 SQLite 文档被标记为“内部模式”表，如为支持“AUTOINCREMENT”列而生成的 `sqlite_sequence` 表，在返回本地对象列表的反射方法中不会包含这些表。这可以避免例如在使用 Alembic autogenerate 时出现问题，之前会将这些由 SQLite 生成的表视为要从模型中删除的表。

    另请参阅

    反映内部模式表

    参考：[#8234](https://www.sqlalchemy.org/trac/ticket/8234)

+   **[sqlite] [用例]**

    为 SQLite 方言添加了 RETURNING 支持。自 SQLite 版本 3.35 起支持 RETURNING。

    参考：[#6195](https://www.sqlalchemy.org/trac/ticket/6195)

+   **[sqlite] [usecase]**

    SQLite 方言现在支持 UPDATE..FROM 语法，用于 UPDATE 语句可能在语句的 WHERE 条件中引用其他表而无需使用子查询。当使用`Update`构造时，如果使用了多个表或其他实体或可选择项，则会自动调用此语法。

    参考：[#7185](https://www.sqlalchemy.org/trac/ticket/7185)

+   **[sqlite] [performance] [bug]**

    SQLite 方言现在在使用基于文件的数据库时默认使用`QueuePool`。这是与将`check_same_thread`参数设置为`False`一起设置的。已经观察到，默认使用`NullPool`的先前方法，在释放数据库连接后不会保留连接，实际上会对性能产生可衡量的负面影响。如常，通过`create_engine.poolclass`参数可以自定义池类。

    另请参阅

    SQLite 方言为基于文件的数据库使用 QueuePool

    参考：[#7490](https://www.sqlalchemy.org/trac/ticket/7490)

+   **[sqlite] [performance] [usecase]**

    SQLite 的 datetime、date 和 time 数据类型现在使用 Python 标准库的`fromisoformat()`方法来解析传入的 datetime、date 和 time 字符串值。这比以前基于正则表达式的方法提高了性能，还自动适应包含六位“微秒”格式或三位“毫秒”格式的 datetime 和 time 格式。

    参考：[#7029](https://www.sqlalchemy.org/trac/ticket/7029)

+   **[sqlite] [bug]**

    移除了关于 `Numeric` 类型发出的警告，该警告指出 DBAPI 不原生支持 Decimal 值。这个警告是针对 SQLite 的，因为 SQLite 没有任何真正的方法来处理超过 15 个有效数字的精度数值，除非使用额外的扩展或解决方法，因为它只使用浮点数学来表示数字。由于这是 SQLite 本身已知且有文档记录的限制，并不是 pysqlite 驱动程序的怪癖，因此 SQLAlchemy 不需要为此发出警告。这个更改不会改变精度数值的处理方式。在使用 SQLite 时，值可以继续按照配置使用 `Decimal()` 或 `float()` 处理，只是无法保持超过 15 个有效数字的精度，除非使用字符串等其他表示形式。

    参考：[#7299](https://www.sqlalchemy.org/trac/ticket/7299)

### mssql

+   **[mssql] [用例]**

    实现了 SQL Server 方言的“聚集索引”标志 `mssql_clustered` 的反射。感谢 John Lennox 提交的拉取请求。

    参考：[#8288](https://www.sqlalchemy.org/trac/ticket/8288)

+   **[mssql] [用例]**

    在创建表时，为 MSSQL 添加了支持表和列注释。增加了在反射表时支持表注释的功能。感谢 Daniel Hall 在这个拉取请求中的帮助。

    参考：[#7844](https://www.sqlalchemy.org/trac/ticket/7844)

+   **[mssql] [错误]**

    `mssql+pyodbc` 方言的 `use_setinputsizes` 参数现在默认为 `True`；这样，非 Unicode 字符串比较将由 pyodbc 绑定到 pyodbc.SQL_VARCHAR 而不是 pyodbc.SQL_WVARCHAR，从而使得对 VARCHAR 列的索引生效。为了使 `fast_executemany=True` 参数继续正常工作，`use_setinputsizes` 模式现在在 `fast_executemany` 为 True 且使用的具体方法是 `cursor.executemany()` 时跳过 `cursor.setinputsizes()` 调用，因为该方法不支持 setinputsizes。该更改还为被标记为 `Unicode` 或 `UnicodeText` 的值添加了适当的 pyodbc DBAPI 类型，同时还修改了基本的 `JSON` 数据类型，将 JSON 字符串值视为 `Unicode` 而不是 `String`。

    参考：[#8177](https://www.sqlalchemy.org/trac/ticket/8177)

+   **[mssql] [已移除]**

    由于缺乏测试支持，删除了对 mxodbc 驱动程序的支持。ODBC 用户可以使用完全受支持的 pyodbc 方言。

    参考：[#7258](https://www.sqlalchemy.org/trac/ticket/7258)

### oracle

+   **[oracle] [feature]**

    添加对新的 oracle 驱动程序 `oracledb` 的支持。

    参见

    oracledb 的方言支持

    python-oracledb

    参考：[#8054](https://www.sqlalchemy.org/trac/ticket/8054)

+   **[oracle] [feature]**

    为包括显式“binary_precision”值的 `FLOAT` 数据类型实现了 DDL 和反射支持。使用 Oracle 特定的 `FLOAT` 数据类型，可以指定新参数 `FLOAT.binary_precision`，它将直接呈现浮点类型的 Oracle 精度。此值在反射期间解释。在反射回一个 `FLOAT` 数据类型时，返回的数据类型是 `DOUBLE_PRECISION`（对于精度为 126 的 `FLOAT` 来说，这也是 Oracle 的默认精度），对于精度为 63 的 `REAL`，以及对于自定义精度的 `FLOAT`，根据 Oracle 文档。

    作为这一变更的一部分，当为 Oracle 生成 DDL 时，明确拒绝使用通用的 `Float.precision` 值，因为这种精度无法准确转换为“二进制精度”；相反，错误消息鼓励使用 `TypeEngine.with_variant()`，以便精确选择 Oracle 的特定精度形式。这是一项与以前行为不兼容的更改，因为以前的“精度”值对于 Oracle 是被静默忽略的。

    参见

    具有二进制精度的新 Oracle FLOAT 类型；直接不接受十进制精度

    参考：[#5465](https://www.sqlalchemy.org/trac/ticket/5465)

+   **[oracle] [feature]**

    对 cx_Oracle 方言实现了完整的“RETURNING”支持，涵盖了两种单独的功能类型：

    +   实现了多行 RETURNING，意味着现在针对产生多于一行 RETURNING 的 DML 语句会接收到多个 RETURNING 行。

    +   当使用 `cursor.executemany()` 时，“executemany RETURNING” 也已实现 - 这允许 RETURNING 在每个语句中产生一行。这一功能的实现为 ORM 插入带来了显著的性能改进，就像 SQLAlchemy 1.4 中为 psycopg2 添加的 ORM Batch inserts with psycopg2 now batch statements with RETURNING in most cases 一样。

    参考：[#6245](https://www.sqlalchemy.org/trac/ticket/6245)

+   **[oracle] [用例]**

    Oracle 现在默认情况下将使用 FETCH FIRST N ROWS / OFFSET 语法来支持 Oracle 12c 及以上的 limit/offset。当直接使用 `Select.fetch()` 时，此语法已经可用，现在对于 `Select.limit()` 和 `Select.offset()` 也是如此。

    参考：[#8221](https://www.sqlalchemy.org/trac/ticket/8221)

+   **[oracle] [更改]**

    Oracle 上的物化视图现在被反映为视图。在之前的 SQLAlchemy 版本中，视图会在表名中返回，而不是在视图名中返回。作为这一变化的副作用，除非设置 `views=True`，否则默认情况下 `MetaData.reflect()` 不会反映它们。要获取物化视图列表，请使用新的检查方法 `Inspector.get_materialized_view_names()`。

+   **[oracle] [错误]**

    对 cx_Oracle 和 oracledb 方言中的 BLOB / CLOB / NCLOB 数据类型进行了调整，以改善性能，根据 Oracle 开发人员的建议。

    参考：[#7494](https://www.sqlalchemy.org/trac/ticket/7494)

+   **[oracle] [错误]**

    关于 `create_engine.implicit_returning` 的弃用，现在“implicit_returning” 功能在所有情况下都已为 Oracle 方言启用；以前，当检测到 Oracle 8/8i 版本时，该功能会被关闭，但在线文档显示这两个版本都支持与现代版本相同的 RETURNING 语法。

    参考：[#6962](https://www.sqlalchemy.org/trac/ticket/6962)

+   **[oracle]**

    cx_Oracle 7 现在是 cx_Oracle 的最低版本要求。

### 杂项

+   **[已移除] [sybase]**

    移除了在之前的 SQLAlchemy 版本中已弃用的“sybase”内部方言。第三方方言支持可用。

    另请参阅

    外部方言

    参考：[#7258](https://www.sqlalchemy.org/trac/ticket/7258)

+   **[已移除] [firebird]**

    移除了在之前的 SQLAlchemy 版本中已弃用的“firebird”内部方言。第三方方言支持可用。

    另请参阅

    外部方言

    参考：[#7258](https://www.sqlalchemy.org/trac/ticket/7258)

### 常规

+   **[常规] [更改]**

    将代码库迁移以删除所有之前在 2.0 版本中被标记为弃用并拟于删除的预 2.0 行为和架构，包括但不限于：

    +   移除了所有 Python 2 代码，最低版本现在是 Python 3.7

    +   `Engine` 和 `Connection` 现在使用新的 2.0 版本的工作方式，其中包括“autobegin”，移除了库级别的自动提交，移除了子事务和“分支”连接

    +   结果对象使用 2.0 版本的行为；`Row` 完全是一个具有命名元组特性的命名元组，不具有“映射”行为，使用 `RowMapping` 来实现“映射”行为。

    +   SQLAlchemy 中所有 Unicode 编码/解码架构已被移除。由于现代 DBAPI 实现支持 Python 3，因此已删除`convert_unicode`功能以及在 DBAPI `cursor.description` 等中查找字节字符串的相关机制。

    +   从 `MetaData`、`Table` 和所有之前可能引用“绑定引擎”的 DDL/DML/DQL 元素中移除了`.bind` 属性和参数

    +   独立的 `sqlalchemy.orm.mapper()` 函数已被移除；所有的经典映射应该通过 `registry.map_imperatively()` 方法来完成，`registry`。

    +   `Query.join()` 方法不再接受关系名称的字符串；使用 `Class.attrname` 作为联接目标的长期记录的方法现在是标准的。

    +   `Query.join()` 不再接受“别名”和“from_joinpoint”参数

    +   `Query.join()` 不再在一次方法调用中接受多个联接目标的链式调用。

    +   移除了 `Query.from_self()`、`Query.select_entity_from()` 和 `Query.with_polymorphic()`。

    +   `relationship.cascade_backrefs` 参数现在必须保持为其新默认值 `False`；`save-update` 级联不再沿着反向引用级联。

    +   `Session.future` 参数现在必须始终设置为 `True`。对于 `Session` 的 2.0 风格事务模式现在始终生效。

    +   现在加载器选项不再接受属性名称的字符串。长期以来一直使用的以 `Class.attrname` 形式为加载器选项目标的方法现在已经成为标准做法。

    +   移除了旧形式的 `select()`，包括 `select([cols])`，`some_table.select()` 的“whereclause”和关键字参数。

    +   移除了 `Select` 上的旧式“原地变异器”方法，如 `append_whereclause()`，`append_order_by()` 等。

    +   移除了非常古老的“dbapi_proxy”模块，该模块在早期的 SQLAlchemy 发布版中用于在原始 DBAPI 连接上提供透明的连接池。

    参考：[#7257](https://www.sqlalchemy.org/trac/ticket/7257)

+   **[general] [changed]**

    `Query.instances()` 方法已弃用。此方法的行为契约已经过时且不再受测试。可以通过诸如 :meth`.Select.from_statement` 或 `aliased()` 等构造来使任意语句返回对象。

### platform

+   **[platform] [feature]**

    SQLAlchemy C 扩展现已替换为全部新的 Cython 实现。与之前的 C 扩展一样，pypi 上提供了各种平台的预构建 wheel 文件，因此在常见平台上构建不是问题。对于自定义构建，`python setup.py build_ext` 仍然与以前一样工作，只需要额外的 Cython 安装。`pyproject.toml` 现在也是源码的一部分，使用 pip 时将建立正确的构建依赖关系。

    请参阅

    C 扩展现在改为使用 Cython

    参考：[#7256](https://www.sqlalchemy.org/trac/ticket/7256)

+   **[platform] [change]**

    SQLAlchemy 的源码构建和安装现在包括了完整的 [**PEP 517**](https://peps.python.org/pep-0517/) 支持。

    请参阅

    安装现在完全支持 pep-517

    参考：[#7311](https://www.sqlalchemy.org/trac/ticket/7311)

### orm

+   **[orm] [feature] [sql]**

    对所有支持 RETURNING 的包含方言添加了名为“insertmanyvalues”的新功能。这是对“快速执行多次”功能的泛化，该功能首次在 1.4 版本中为 psycopg2 驱动程序引入，详见使用 RETURNING 的 ORM 批量插入现在在大多数情况下批量返回语句，它允许 ORM 将 INSERT 语句批量处理为更高效的 SQL 结构，同时仍然能够使用 RETURNING 检索新生成的主键和 SQL 默认值。

    此功能现在适用于支持 RETURNING 的多个方言以及用于 INSERT 的多个 VALUES 构造，包括所有 PostgreSQL 驱动程序、SQLite、MariaDB、MS SQL Server。另外，Oracle 方言也使用本地 cx_Oracle 或 OracleDB 功能获得了相同的功能。

    参考：[#6047](https://www.sqlalchemy.org/trac/ticket/6047)

+   **[orm] [feature]**

    添加了新参数`AttributeEvents.include_key`，它将包括字典或列表操作的键，例如`__setitem__()`（例如`obj[key] = value`）和`__delitem__()`（例如`del obj[key]`），使用一个新的关键字参数“key”或“keys”，取决于事件，例如`AttributeEvents.append.key`，`AttributeEvents.bulk_replace.keys`。这允许事件处理程序考虑传递给操作的键，并且对于使用`MappedCollection`的字典操作非常重要。

    参考：[#8375](https://www.sqlalchemy.org/trac/ticket/8375)

+   **[orm] [feature]**

    添加了新参数`Operators.op.python_impl`，可从`Operators.op()`和直接使用`custom_op`构造函数时使用，它允许提供一个在 Python 中的评估函数以及自定义 SQL 运算符。当操作符对象在两侧都使用普通 Python 对象作为操作数时，此评估函数成为使用的实现，并且特别兼容于 启用 ORM 的 INSERT、UPDATE 和 DELETE 语句中使用的`synchronize_session='evaluate'`选项。

    参考：[#3162](https://www.sqlalchemy.org/trac/ticket/3162)

+   **[orm] [feature]**

    现在`Session`（以及间接地`AsyncSession`以线程不安全的方式使用的情况下，其中事件钩子或类似物可能在操作中调用意外的方法，以及潜在地在其他并发情况下，例如 asyncio 或 gevent 在首次发生非法访问时引发信息性消息，而不是无声地传递，从而导致由于`Session`处于无效状态而导致的次要故障。

    请参阅

    当检测到非法并发或重入访问时，会主动引发会话

    参考：[#7433](https://www.sqlalchemy.org/trac/ticket/7433)

+   **[orm] [feature]**

    当与 Python `dataclass`一起使用时，`composite()`映射构造现在支持值的自动解析；不再需要实现`__composite_values__()`方法，因为此方法是从数据类的检查中派生的。

    此外，由`composite`映射的类现在支持排序比较操作，例如 `<`，`>=`等。

    请参阅 Composite Column Types 的新文档以获取示例。

+   **[orm] [feature]**

    在`selectinload()`和`immediateload()`加载器选项中添加了非常实验性的功能，称为`selectinload.recursion_depth` / `immediateload.recursion_depth`，允许单个加载器选项自动递归到自引用关系中。设置为表示深度的整数，也可以设置为-1，表示继续加载直到不再找到更深层级。对`selectinload()`和`immediateload()`进行了重大内部更改，使此功能能够继续正确使用编译缓存，并且不使用任意递归，因此支持任何深度级别（尽管会发出相应数量的查询）。这对于必须完全急切加载的自引用结构可能很有用，例如在使用 asyncio 时。

    当检测到相关对象加载中存在过多递归深度时，连接在一起的加载器选项（即，未使用新的`recursion_depth`选项）时也会发出警告。此操作继续使用大量内存并性能极差；当检测到此条件时，缓存将被禁用，以防止缓存被任意语句淹没。

    参考：[#8126](https://www.sqlalchemy.org/trac/ticket/8126)

+   **[orm] [功能]**

    添加了新参数`Session.autobegin`，当设置为`False`时，将阻止`Session`隐式开始事务。必须首先显式调用`Session.begin()`方法才能继续操作，否则在任何操作本应自动开始时都会引发错误。此选项可用于创建一个“安全”的`Session`，不会隐式启动新事务。

    作为此更改的一部分，还添加了一个新的状态变量`origin`，这可能对事件处理代码有用，以便了解特定`SessionTransaction`的起源。

    参考文献：[#6928](https://www.sqlalchemy.org/trac/ticket/6928)

+   **[orm] [特性]**

    不再需要使用`declared_attr()`来实现此映射的使用`Column`对象的声明性混合对象，该对象包含`ForeignKey`引用；当列应用于声明的映射时，`ForeignKey`对象与`Column`本身一起被复制。

+   **[orm] [用例]**

    在`load_only()`加载器选项中添加了`load_only.raiseload`参数，使得未加载的属性可以具有“raise”行为而不是惰性加载。以前使用`load_only()`选项直接实现这一点的方法实际上并不存在。

+   **[orm] [更改]**

    为了更好地适应显式类型，一些通常在内部构建但有时也可见于消息以及类型化的 ORM 构造的名称已更改为更简洁的名称，这些名称还与其构造函数的名称（使用不同的大小写）匹配，所有情况下都保留了旧名称的别名以供可预见的未来使用：

    +   `RelationshipProperty`成为主要名称`Relationship`的别名，始终从`relationship()`函数构造

    +   `SynonymProperty`成为主要名称`Synonym`的别名，始终从`synonym()`函数构造

    +   `CompositeProperty`现在成为主要名称`Composite`的别名，始终由`composite()`函数构造

+   **[orm] [change]**

    为了与突出的 ORM 概念`Mapped`保持一致，字典导向的集合的名称，`attribute_mapped_collection()`，`column_mapped_collection()`和`MappedCollection`，被更改为`attribute_keyed_dict()`，`column_keyed_dict()`和`KeyFuncDict`，使用短语“dict”来最小化与术语“mapped”的任何混淆。旧名称将无限期保留，没有删除计划。

    参考：[#8608](https://www.sqlalchemy.org/trac/ticket/8608)

+   **[orm] [bug]**

    所有`Result`对象现在在硬关闭后使用时将一致引发`ResourceClosedError`，包括在调用“单行或值”方法后发生的“硬关闭”，例如`Result.first()`和`Result.scalar()`。这已经是基于`CursorResult`的最常见的 Core 语句执行返回的结果对象的行为，因此这种行为并非新鲜事。然而，这一变化已经扩展到适当地适应使用 2.0 风格 ORM 查询时返回的 ORM“过滤”结果对象，以前这些对象会以“软关闭”方式返回空结果，或者根本不会“软关闭”，而会继续从底层游标中产生。

    作为这一变更的一部分，还向基本`Result`类添加了`Result.close()`，并为 ORM 使用的过滤结果实现实现了它，因此在使用`yield_per`执行选项关闭服务器端游标之前，可以调用基础`CursorResult`上的`CursorResult.close()`方法，以关闭尚未获取的 ORM 结果。这对于 Core 结果集已经可用，但此更改也使其适用于 2.0 风格的 ORM 结果。

    此更改也**回溯**到：1.4.27

    参考：[#7274](https://www.sqlalchemy.org/trac/ticket/7274)

+   **[orm] [bug]**

    修复了`registry.map_declaratively()`方法返回内部“映射器配置”对象而不是 API 文档中所述的`Mapper`对象的问题。

+   **[orm] [bug]**

    修复了性能回退问题，该问题至少在 1.3 版本中出现，如果不是更早（在 1.0 之后的某个时候），在联接继承子类中加载延迟列时，那些明确映射为`defer()`的列，而不是已过期的非延迟列，不会使用“优化”查询，该查询仅查询包含未加载列的直接表，而是运行完整的 ORM 查询，该查询会为所有基本表发出 JOIN，当仅从子类加载列时，这是不必要的。

    参考：[#7463](https://www.sqlalchemy.org/trac/ticket/7463)

+   **[orm] [bug]**

    `Load`对象及相关加载器策略模式的内部大部分已经重写，以利用现在仅支持属性绑定路径而不是字符串的事实。重写希望更容易解决加载器策略系统中的新用例和微妙问题。

    参考：[#6986](https://www.sqlalchemy.org/trac/ticket/6986)

+   **[orm] [bug]**

    对“deferred”/“load_only”一组策略选项进行了改进，其中如果一个对象从一个查询中的两个不同的逻辑路径加载，则已由至少一个选项配置为填充的属性将在所有情况下填充，即使对于该对象的其他加载路径没有设置此选项。以前，基于随机性来确定哪个“路径”首先处理对象。

    参考资料：[#8166](https://www.sqlalchemy.org/trac/ticket/8166)

+   **[orm] [bug]**

    在启用 ORM 的情况下，修复了针对联合继承子类创建的 UPDATE 语句中的问题，仅更新本地表列的情况，在使用 RETURNING 进行获取同步的数据库中，如果使用“fetch”同步策略，则不会生成正确的 RETURNING 子句。还调整了 UPDATE FROM 和 DELETE FROM 语句中使用的 RETURNING 策略。

    参考资料：[#8344](https://www.sqlalchemy.org/trac/ticket/8344)

+   **[orm] [bug] [asyncio]**

    从`begin`和`begin_nested`中移除了未使用的 `**kw` 参数。这些参数未被使用，似乎是错误地添加到 API 中的。

    参考资料：[#7703](https://www.sqlalchemy.org/trac/ticket/7703)

+   **[orm] [bug]**

    更改了`attribute_mapped_collection()`和`column_mapped_collection()`的属性访问方法（现在称为`attribute_keyed_dict()`和`column_keyed_dict()`) ，用于断言要用作字典键的对象上的数据值实际上存在，并且不是由于属性从未实际分配而使用“None”。这用于防止在通过反向引用进行赋值时，对象上的“key”属性尚未被分配时，为键错误地赋予“None”。

    由于此处的故障模式是一个通常不会持久到数据库的瞬态条件，并且很容易通过类的构造函数根据分配参数的顺序产生，很可能许多应用程序已经包含了这种行为，这种行为是被默默地忽略的。为了适应这种现在引发错误的应用程序，还向 `attribute_keyed_dict()` 和 `column_keyed_dict()` 添加了一个新参数 `attribute_keyed_dict.ignore_unpopulated_attribute`，该参数使得错误的反向引用赋值被跳过。

    参考文献：[#8372](https://www.sqlalchemy.org/trac/ticket/8372)

+   **[orm] [bug]**

    向 `AbstractConcreteBase` 声明混合类添加了新的参数 `AbstractConcreteBase.strict_attrs`。该参数的效果是，子类上属性的范围正确限制为在每个属性声明的子类中，而不是以前的行为，在整个层次结构上应用于基本“抽象”类的所有属性。这产生了一个更干净、更正确的映射，其中子类不再具有仅与同级类相关的非有用属性。该参数的默认值为 False，这保留了先前的行为不变；这是为了支持在查询中显式使用这些属性的现有代码。要迁移到新方法，请根据需要将显式属性应用于抽象基类。

    参考文献：[#8403](https://www.sqlalchemy.org/trac/ticket/8403)

+   **[orm] [bug]**

    关于 `defer()` 的行为已经修订，以处理主键和“多态鉴别器”列，使得这些列不再是可延迟加载的，无论是显式地还是使用诸如 `defer('*')` 的通配符。以前，通配符延迟加载将不会加载主键/多态列，这导致在所有情况下都出现错误，因为 ORM 依赖于这些列来生成对象标识。对主键列的显式延迟加载的行为未更改，因为这些延迟加载已经被隐式忽略。

    参考文献：[#7495](https://www.sqlalchemy.org/trac/ticket/7495)

+   **[orm] [bug]**

    修复了`Mapper.eager_defaults`参数行为中的错误，以前只有表定义中的客户端端 SQL 默认值或 onupdate 表达式会在 ORM 为行执行 INSERT 或 UPDATE 时触发使用 RETURNING 或 SELECT 的提取操作。以前，只有作为表 DDL 的一部分和/或服务器端 onupdate 表达式建立的服务器端默认值会触发这个提取，尽管客户端端 SQL 表达式会在提取时被包括。

    引用：[#7438](https://www.sqlalchemy.org/trac/ticket/7438)

### 引擎

+   **[引擎] [特性]**

    `DialectEvents.handle_error()`事件现在已从 `EngineEvents` 套件移动到 `DialectEvents` 套件，并且现在参与连接池的“pre ping”事件，对于那些使用断开代码来检测数据库是否在线的方言。这允许最终用户代码修改“pre ping”的状态。请注意，这不包括包含本地“ping”方法的方言，例如 psycopg2 或大多数 MySQL 方言。

    引用：[#5648](https://www.sqlalchemy.org/trac/ticket/5648)

+   **[引擎] [特性]**

    `ConnectionEvents.set_connection_execution_options()` 和 `ConnectionEvents.set_engine_execution_options()` 事件挂钩现在允许对给定的选项字典进行就地修改，新内容将作为最终执行选项接收。以前，不支持对字典进行就地修改。

+   **[引擎] [用例]**

    将`create_engine.isolation_level`参数泛化到基础方言，以便它不再依赖于各个方言的存在。此参数设置“隔离级别”设置为一旦由连接池创建新的数据库连接，该值就保持设置而不会在每次签入时重置。

    `create_engine.isolation_level`参数在功能上基本等同于通过`Engine.execution_options.isolation_level`参数使用`Engine.execution_options()`进行引擎范围设置。区别在于前者设置隔离级别仅在创建连接时执行一次，后者在每次连接检出时设置和重置给定级别。

    参考：[#6342](https://www.sqlalchemy.org/trac/ticket/6342)

+   **[引擎] [更改]**

    关于引擎和方言的一些小的 API 更改：

    +   `Dialect.set_isolation_level()`，`Dialect.get_isolation_level()`，:meth: 方言方法将始终传递原始的 DBAPI 连接

    +   `Connection`和`Engine`类不再共享基础的`Connectable`超类，该超类已被移除。

    +   添加了一个新的接口类`PoolProxiedConnection` - 这是熟悉的`_ConnectionFairy`类的公共接口，尽管它是一个私有类。

    参考：[#7122](https://www.sqlalchemy.org/trac/ticket/7122)

+   **[引擎] [错误] [回归]**

    修复了一个问题，即`CursorResult.fetchmany()`方法在结果完全耗尽时无法自动关闭服务器端游标（即在使用`stream_results`或`yield_per`时，无论是核心还是 ORM 导向的结果）。

    此更改也**反向移植**到：1.4.27

    参考：[#7274](https://www.sqlalchemy.org/trac/ticket/7274)

+   **[引擎] [错误]**

    修复了未来版本的`Engine`中的问题，即在调用`Engine.begin()`并进入上下文管理器时，如果实际的 BEGIN 操作因某些原因失败，例如事件处理程序引发异常，则不会关闭连接；这种情况未对引擎的未来版本进行测试。请注意，“未来”上下文管理器处理 Core 和 ORM 中的`begin()`块，直到实际进入上下文管理器时才运行“BEGIN”操作。这与立即运行“BEGIN”操作的旧版本不同。

    此更改也已**回溯**至：1.4.27

    参考：[#7272](https://www.sqlalchemy.org/trac/ticket/7272)

+   **[engine] [bug]**

    当`pool_size=0`时，`QueuePool`现在会忽略`max_overflow`，在所有情况下正确地使池无限制。

    参考：[#8523](https://www.sqlalchemy.org/trac/ticket/8523)

+   **[engine] [bug]**

    为了提高安全性，当调用`str(url)`时，`URL`对象现在默认使用密码混淆。若要以明文密码字符串化 URL，则可以使用`URL.render_as_string()`，并将`URL.render_as_string.hide_password`参数设为`False`。感谢我们的贡献者提供此拉取请求。

    另请参阅

    默认情况下，`str(engine.url)`将混淆密码

    参考：[#8567](https://www.sqlalchemy.org/trac/ticket/8567)

+   **[engine] [bug]**

    `Inspector.has_table()`方法现在将一致性地检查给定名称的视图以及表。以前，此行为取决于方言，其中 PostgreSQL、MySQL/MariaDB 和 SQLite 支持它，而 Oracle 和 SQL Server 不支持它。第三方方言还应努力确保它们的`Inspector.has_table()`方法搜索给定名称的视图和表。

    参考：[#7161](https://www.sqlalchemy.org/trac/ticket/7161)

+   **[engine] [bug]**

    修复了 `Result.columns()` 方法中的问题，当使用单个索引调用 `Result.columns()` 时，在某些情况下，特别是 ORM 结果对象的情况下，可能导致 `Result` 产生标量对象，而不是 `Row` 对象，就好像已调用了 `Result.scalars()` 方法一样。在 SQLAlchemy 1.4 中，此场景会发出警告，指出行为将在 SQLAlchemy 2.0 中发生更改。

    参考：[#7953](https://www.sqlalchemy.org/trac/ticket/7953)

+   **[engine] [错误]**

    不推荐将 `DefaultGenerator` 对象（如 `Sequence`）传递给 `Connection.execute()` 方法，因为此方法被类型化为返回 `CursorResult` 对象，而不是普通的标量值。应改用 `Connection.scalar()` 方法，已重新设计此方法，具有新的内部代码路径，以适合调用不经过 `Connection.execute()` 方法即可进行选择默认生成对象。

+   **[engine] [已移除]**

    从 `create_engine()` 中移除了以前已弃用的 `case_sensitive` 参数，这只会影响仅在 Core 结果集行中查找字符串列名称；它对 ORM 的行为没有影响。`case_sensitive` 所指向的有效行为仍保持其默认值为 `True`，这意味着在 `row._mapping` 中查找的字符串名称将以区分大小写的方式匹配，就像任何其他 Python 映射一样。

    请注意 `case_sensitive` 参数与控制大小写敏感性、引用和“名称规范化”（即转换为将所有大写单词视为大小写不敏感的数据库）DDL 标识符名称的一般主题无关，后者仍然是 SQLAlchemy 的一个正常核心功能。

+   **[engine] [已移除]**

    删除了传统的已弃用包 `sqlalchemy.databases`。请改用 `sqlalchemy.dialects`。

    参考：[#7258](https://www.sqlalchemy.org/trac/ticket/7258)

+   **[engine] [已弃用]**

    `create_engine.implicit_returning` 参数仅在 `create_engine()` 函数上已废弃；该参数仍然在 `Table` 对象上可用。当 SQLAlchemy 最初开发时，此参数最初用于启用“隐式返回”功能，并且默认情况下未启用。在现代用法中，没有理由禁用此参数，因为已经观察到它会导致混淆，因为它会降低性能，并使 ORM 更难以检索最近插入的服务器默认值。此参数仍然在 `Table` 上可用于特定的数据库级别边缘情况，其中使 RETURNING 不可行，目前唯一的示例是 SQL Server 的限制，即不得在具有 INSERT 触发器的表上使用 INSERT RETURNING。

    参考: [#6962](https://www.sqlalchemy.org/trac/ticket/6962)

### sql

+   **[sql] [功能]**

    > 添加了长期要求的不区分大小写的字符串操作符 `ColumnOperators.icontains()`, `ColumnOperators.istartswith()`, `ColumnOperators.iendswith()`，这些操作符生成不区分大小写的 LIKE 组合（在 PostgreSQL 上使用 ILIKE，在其他所有后端上使用 LOWER() 函数），以补充现有的 LIKE 组合操作符 `ColumnOperators.contains()`, `ColumnOperators.startswith()`，等等。特别感谢 Matias Martinez Rebori 在实现这些新方法时的细致和完整的努力。

    参考: [#3482](https://www.sqlalchemy.org/trac/ticket/3482)

+   **[sql] [功能]**

    在所有`FromClause`对象的`FromClause.c`集合上添加了新的语法，允许将键的元组传递给`__getitem__()`，以及支持`select()`构造处理直接处理结果类似元组的集合，允许使用`select(table.c['a', 'b', 'c'])`这样的语法。返回的子集本身是一个`ColumnCollection`，现在也可以直接被`select()`和类似的函数消耗。

    另请参阅

    设置 COLUMNS 和 FROM 子句

    参考：[#8285](https://www.sqlalchemy.org/trac/ticket/8285)

+   **[sql] [feature]**

    新增了与后端无关的`Uuid`数据类型，从 PostgreSQL 方言泛化为核心类型，同时将`UUID`从 PostgreSQL 方言迁移过来。SQL Server 的`UNIQUEIDENTIFIER`数据类型也变成了一个处理 UUID 的数据类型。感谢 Trevor Gross 的帮助。

    参考：[#7212](https://www.sqlalchemy.org/trac/ticket/7212)

+   **[sql] [feature]**

    将`Double`、`DOUBLE`、`DOUBLE_PRECISION`数据类型添加到基本的`sqlalchemy.`模块命名空间，用于明确使用双精度/双精度以及通用“双精度”数据类型。使用`Double`进行通用支持，根据不同的后端需要解析为 DOUBLE/DOUBLE PRECISION/FLOAT。

    参考：[#5465](https://www.sqlalchemy.org/trac/ticket/5465)

+   **[sql] [usecase]**

    更改了`Insert`构造的编译机制，以便在已知会生成自动增量值的特定后端上，即使在参数集或`Insert.values()`方法中作为普通绑定值存在时，也将通过`cursor.lastrowid`或 RETURNING 获取“自动增量主键”列值，用于单行 INSERT 语句。这恢复了 1.3 系列中的行为，适用于单独参数集以及`Insert.values()`的用例。在 1.4 中，参数集行为无意中更改为不再执行此操作，但`Insert.values()`方法仍会获取自动增量值，直到 1.4.21，其中[#6770](https://www.sqlalchemy.org/trac/ticket/6770)再次无意中更改了行为，因为此用例从未被覆盖。

    现在定义的行为为“工作”，以适应 SQLite、MySQL 和 MariaDB 等数据库忽略显式 NULL 主键值并仍调用自动增量生成器的情况。

    参考：[#7998](https://www.sqlalchemy.org/trac/ticket/7998)

+   **[sql] [用例]**

    当使用 SQL 编译器提供的`literal_binds`与 PostgreSQL、MySQL、MariaDB、MSSQL、Oracle 方言时，添加了修改后的 ISO-8601 呈现（即将 T 转换为空格的 ISO-8601）。对于 Oracle，ISO 格式被包装在适当的 TO_DATE()函数调用内。以前，此呈现未针对特定方言的编译实现。

    另请参阅

    日期，时间，日期时间数据类型现在在所有后端上支持文字呈现

    参考：[#5052](https://www.sqlalchemy.org/trac/ticket/5052)

+   **[sql] [用例]**

    向`HasCTE.add_cte()`添加了新参数`HasCTE.add_cte.nest_here`，该参数将在父语句级别“嵌套”给定的`CTE`。该参数等效于使用`HasCTE.cte.nesting`参数，但在某些情况下可能更直观，因为它允许同时设置嵌套属性以及 CTE 的显式级别。

    `HasCTE.add_cte()` 方法还接受多个 CTE 对象。

    参考：[#7759](https://www.sqlalchemy.org/trac/ticket/7759)

+   **[sql] [bug]**

    在使用 `Select.select_from()` 方法时，对于 `select()` 构造的 FROM 子句现在将首先在呈现的 SELECT 的 FROM 子句中呈现，这有助于保持子句的顺序，就像它们传递给 `Select.select_from()` 方法本身一样，而不受这些子句也在查询的其他部分提及的影响。如果 `Select` 的其他元素也生成 FROM 子句，比如列子句或 WHERE 子句，这些将在 `Select.select_from()` 传递的子句之后呈现，假设它们未明确传递给 `Select.select_from()`。这种改进在某些情况下非常有用，特定数据库根据 FROM 子句的特定顺序生成理想的查询计划，并允许完全控制 FROM 子句的顺序。

    参考：[#7888](https://www.sqlalchemy.org/trac/ticket/7888)

+   **[sql] [bug]**

    `Enum.length` 参数用于非本地枚举类型的 `VARCHAR` 列的长度设置，在为 `VARCHAR` 数据类型发出 DDL 时无条件使用，包括当目标后端继续使用 `VARCHAR` 时设置 `Enum.native_enum` 参数为 `True` 的情况。在这种情况下，先前将会错误地忽略该参数。现在移除了此情况下先前发出的警告。

    参考：[#7791](https://www.sqlalchemy.org/trac/ticket/7791)

+   **[sql] [bug]**

    对于 Python 整数的就地类型检测，如表达式`literal(25)`，现在也将应用基于值的适应性，以适应 Python 大整数，其中确定的数据类型将是`BigInteger`而不是`Integer`。这适用于像 asyncpg 这样的方言，它既向驱动程序发送隐式类型信息，又对数字规模敏感。

    参考：[#7909](https://www.sqlalchemy.org/trac/ticket/7909)

+   **[sql] [bug]**

    为所有“Create”/“Drop”构造添加了`if_exists`和`if_not_exists`参数，包括`CreateSequence`、`DropSequence`、`CreateIndex`、`DropIndex`等，允许在 DDL 中呈现通用的“IF EXISTS”/“IF NOT EXISTS”短语。拉取请求由 Jesse Bakker 提供。

    参考：[#7354](https://www.sqlalchemy.org/trac/ticket/7354)

+   **[sql] [bug]**

    改进了 SQL 二进制表达式的构建，以允许非常长的相同关联运算符的表达式，而不需要特殊步骤来避免高内存使用和过多的递归深度。现在，特定的二元操作`A op B`可以与另一个元素`op C`连接，结果结构将被“展平”，以使表示以及 SQL 编译不需要递归。

    这一变化的一个影响是，使用 SQL 函数的字符串连接表达式会“展平”，例如，MySQL 现在会渲染`concat('x', 'y', 'z', ...)`而不是将两个元素函数像`concat(concat('x', 'y'), 'z')`一样嵌套在一起。覆盖字符串连接运算符的第三方方言将需要实现一个新方法`def visit_concat_op_expression_clauselist()`来配合现有的`def visit_concat_op_binary()`方法。

    参考：[#7744](https://www.sqlalchemy.org/trac/ticket/7744)

+   **[sql] [bug]**

    实现了对“truediv”和“floordiv”的全面支持，使用“/”和“//”运算符。现在，使用 `Integer` 的两个表达式之间的“truediv”操作将考虑结果为 `Numeric`，并且方言级编译将根据方言特定的基础将右操作数转换为数字类型以确保实现 truediv。对于 floordiv，还添加了转换，以用于那些默认情况下不执行 floordiv 的数据库（MySQL、Oracle），并且在这种情况下会呈现 `FLOOR()` 函数，以及对于右操作数不是整数的情况（需要用于 PostgreSQL 和其他数据库）。

    此更改解决了不同后端上除法运算符行为不一致的问题，还修复了在 Oracle 上的整数除法无法获取结果的问题，因为输出类型处理程序不当。

    另见

    Python 除法运算符对所有后端执行真除法；增加了地板除法

    参考：[#4926](https://www.sqlalchemy.org/trac/ticket/4926)

+   **[sql] [bug]**

    对编译器添加了额外的查找步骤，将跟踪所有 FROM 子句，这些子句是表，可能在多个模式中共享具有相同名称的模式之一，在这种情况下，在编译器级别引用该名称时不带模式限定符将以匿名别名名称呈现该表名，以澄清两者（或更多）的名称。还考虑了使用服务器检测到的“默认模式名称”值对通常未限定名称进行模式限定的方法，但此方法不适用于 Oracle，SQL Server 也不被接受，也不适用于 PostgreSQL 搜索路径中的多个条目。此处解决的名称冲突问题已确定至少影响 Oracle、PostgreSQL、SQL Server、MySQL 和 MariaDB。

    参考：[#7471](https://www.sqlalchemy.org/trac/ticket/7471)

+   **[sql] [bug]**

    从值的类型确定 SQL 类型的 Python 字符串值，主要是在使用`literal()`时，现在将应用`String`类型，而不是`Unicode`数据类型，对于使用 Python `str.isascii()`测试为“仅 ascii”的 Python 字符串值。如果字符串不是`isascii()`，则将绑定`Unicode`数据类型，而以前所有字符串检测中都使用了它。此行为**仅适用于使用``literal()``或其他没有现有数据类型的上下文中的数据类型的就地检测**，这在正常的`Column`比较操作下通常不是情况，在这种情况下，被比较的`Column`的类型始终优先。

    使用`Unicode`数据类型可以确定在诸如 SQL Server 之类的后端上的文字字符串格式，其中文字值（即使用`literal_binds`）将呈现为`N'<value>'`而不是`'value'`。对于正常的绑定值处理，`Unicode`数据类型在将值传递给 DBAPI 时也可能对其产生影响，同样在 SQL Server 的情况下，pyodbc 驱动程序支持使用 setinputsizes 模式，它将以不同的方式处理`String`与`Unicode`。

    参考：[#7551](https://www.sqlalchemy.org/trac/ticket/7551)

+   **[sql] [bug]**

    `array_agg`现在将数组维度设置为 1。改进了对`None`值作为多维数组值的处理。

    参考：[#7083](https://www.sqlalchemy.org/trac/ticket/7083)

### 模式

+   **[模式] [功能]**

    对由`ExecutableDDLElement`类（从`DDLElement`更名而来）实现的“条件 DDL”系统进行了扩展，以直接在`SchemaItem`构造上可用，例如`Index`，`ForeignKeyConstraint`等，使得用于生成这些元素的条件逻辑包含在默认的 DDL 发射过程中。这个系统也可以通过将来的 Alembic 版本来支持条件 DDL 元素，以支持所有模式管理系统中的条件 DDL 元素。

    亦参见

    约束和索引的新条件 DDL

    参考资料：[#7631](https://www.sqlalchemy.org/trac/ticket/7631)

+   **[schema] [usecase]**

    向`DropConstraint`构造添加了参数`DropConstraint.if_exists`，这将导致“IF EXISTS”DDL 被添加到 DROP 语句中。这个短语并不被所有数据库接受，如果数据库不支持这个短语，操作将失败，因为在单个 DDL 语句的范围内没有类似的兼容后备方案。感谢 Mike Fiedler 提供的拉取请求。

    参考资料：[#8141](https://www.sqlalchemy.org/trac/ticket/8141)

+   **[schema] [usecase]**

    实现了 DDL 事件钩子`DDLEvents.before_create()`，`DDLEvents.after_create()`，`DDLEvents.before_drop()`，`DDLEvents.after_drop()`，适用于包含不同的 CREATE 或 DROP 步骤的所有`SchemaItem`对象，当该步骤被作为单独的 SQL 语句调用时，包括`ForeignKeyConstraint`，`Sequence`，`Index`，以及 PostgreSQL 的`ENUM`。

    参考资料：[#8394](https://www.sqlalchemy.org/trac/ticket/8394)

+   **[模式] [性能]**

    重新设计了模式反射 API，允许参与的方言利用高性能的批量查询来一次性反映许多表的模式，查询次数减少了一个数量级。新的性能特性首先针对 PostgreSQL 和 Oracle 后端，可以应用于任何利用 SELECT 查询系统目录表反映表的方言。此更改还包括对`Inspector`对象的新 API 特性和行为改进，包括方法如`Inspector.has_table()`的一致、缓存行为，`Inspector.get_table_names()`以及新方法`Inspector.has_schema()`和`Inspector.has_index()`。

    参见

    数据库反射的重大架构、性能和 API 增强 - 完整背景

    参考：[#4379](https://www.sqlalchemy.org/trac/ticket/4379)

+   **[模式] [错误]**

    当使用`Table.include_columns`参数排除后发现是这些约束的一部分的列时，关于反射索引或唯一约束的警告已被移除。当使用`Table.include_columns`参数时，应该预期生成的`Table`构造将不包括依赖于被省略列的约束。这一更改是针对[#8100](https://www.sqlalchemy.org/trac/ticket/8100)做出的，该修复了`Table.include_columns`与依赖于被省略列的外键约束一起使用时，使用案例明确表明应该预期省略这些约束。

    参考：[#8102](https://www.sqlalchemy.org/trac/ticket/8102)

+   **[模式] [postgresql]**

    对`Constraint`对象的注释支持已添加，包括 DDL 和反射；该字段已添加到基本的`Constraint`类和相应的构造函数中，但目前只有 PostgreSQL 支持该功能。请参阅诸如`ForeignKeyConstraint.comment`、`UniqueConstraint.comment`或`CheckConstraint.comment`之类的参数。

    参考：[#5677](https://www.sqlalchemy.org/trac/ticket/5677)

+   **[schema] [mariadb] [mysql]**

    增加对 MySQL 和 MariaDB 反映选项的分区和示例页面的支持。选项存储在表方言选项字典中，因此以下关键字需要根据后端添加前缀`mysql_`或`mariadb_`。支持的选项包括：

    +   `stats_sample_pages`

    +   `partition_by`

    +   `partitions`

    +   `subpartition_by`

    这些选项在从数据库加载表时也会反映出来，并将填充表`Table.dialect_options`。感谢 Ramon Will 的拉取请求。

    参考：[#4038](https://www.sqlalchemy.org/trac/ticket/4038)

### typing

+   **[typing] [improvement]**

    `TypeEngine.with_variant()`方法现在返回原始`TypeEngine`对象的副本，而不是将其包装在`Variant`类中，该类已被实际删除（导入符号仍保留用于与可能正在测试此符号的代码向后兼容）。虽然先前的方法维护了 Python 中的行为，但保持原始类型可以更清晰地进行类型检查和调试。

    `TypeEngine.with_variant()`方法现在每次调用也接受多个方言名称，特别是对于相关的后端名称如`"mysql", "mariadb"`，这对于提高效率非常有帮助。

    亦可参考

    “with_variant()” clones the original TypeEngine rather than changing the type

    参考：[#6980](https://www.sqlalchemy.org/trac/ticket/6980)

### postgresql

+   **[postgresql] [feature]**

    添加了一个新的 PostgreSQL `DOMAIN` 数据类型，其遵循与 PostgreSQL `ENUM` 相同的 CREATE TYPE / DROP TYPE 行为。非常感谢 David Baumgold 对此的努力。

    另请参阅

    `DOMAIN`

    参考：[#7316](https://www.sqlalchemy.org/trac/ticket/7316)

+   **[postgresql] [usecase] [asyncpg]**

    添加了可覆盖的方法 `PGDialect_asyncpg.setup_asyncpg_json_codec` 和 `PGDialect_asyncpg.setup_asyncpg_jsonb_codec` 编解码器，当使用 asyncpg 时，它们处理注册 JSON/JSONB 编解码器的必要任务。该更改是将方法拆分为独立的、可覆盖的方法，以支持需要修改或禁用这些特定编解码器设置的第三方方言。

    此更改也被**回溯**至：1.4.27

    参考：[#7284](https://www.sqlalchemy.org/trac/ticket/7284)

+   **[postgresql] [usecase]**

    为 `ARRAY` 和 `ARRAY` 数据类型添加了字面类型渲染。通用字符串化将使用方括号进行渲染，例如 `[1, 2, 3]`，而 PostgreSQL 特定将使用 ARRAY 字面值，例如 `ARRAY[1, 2, 3]`。还考虑了多个维度和引号。

    参考：[#8138](https://www.sqlalchemy.org/trac/ticket/8138)

+   **[postgresql] [usecase]**

    为 PostgreSQL 多范围类型添加支持，引入自 PostgreSQL 14。现在，psycopg3、psycopg2 和 asyncpg 后端都已经通用化支持 PostgreSQL 范围和多范围，可以进一步支持不同的方言，使用与之前使用的 psycopg2 对象构造兼容的后端无关的 `Range` 数据对象。请参阅新的文档以了解使用模式。

    此外，范围类型处理已经增强，以便自动渲染类型转换，以便在不提供任何上下文给数据库的语句的情况下，无需对数据库明确指定所需的类型即可进行原地往返（讨论在 [#8540](https://www.sqlalchemy.org/trac/ticket/8540) 中）。

    非常感谢 @zeeeeeb 实施和测试新数据类型和 psycopg 支持的拉取请求。

    另请参阅

    PostgreSQL 后端的新 RANGE / MULTIRANGE 支持和更改

    范围和多范围类型

    参考资料：[#7156](https://www.sqlalchemy.org/trac/ticket/7156), [#8540](https://www.sqlalchemy.org/trac/ticket/8540)

+   **[postgresql] [用例]**

    配置`create_engine.pool_pre_ping`以对 psycopg、asyncpg 和 pg8000 进行预检时，发出的“ping”查询已更改为一个空查询 (`;`)，而不是 `SELECT 1`；此外，对于 asyncpg 驱动程序，已修复了此查询不必要地使用预准备语句的问题。理由是消除 PostgreSQL 在发出 ping 时产生查询计划的需要。当前不支持此操作的 `psycopg2` 驱动程序仍然使用 `SELECT 1`。

    参考资料：[#8491](https://www.sqlalchemy.org/trac/ticket/8491)

+   **[postgresql] [变更]**

    SQLAlchemy 现在需要 PostgreSQL 版本 9 或更高版本。在某些有限的使用情况下，旧版本可能仍然可用。

+   **[postgresql] [变更] [mssql]**

    `UUID`的参数 `UUID.as_uuid`，以前仅适用于 PostgreSQL 方言，现在已泛化为 Core（以及一个新的与后端无关的 `Uuid` 数据类型），现在默认值为 `True`，表示此数据类型默认接受 Python `UUID` 对象。此外，SQL Server 的 `UNIQUEIDENTIFIER` 数据类型已转换为接收 UUID 的类型；对于使用字符串值使用 `UNIQUEIDENTIFIER` 的旧代码，请将 `UNIQUEIDENTIFIER.as_uuid` 参数设置为 `False`。 

    参考资料：[#7225](https://www.sqlalchemy.org/trac/ticket/7225)

+   **[postgresql] [变更]**

    PostgreSQL 特有的 `ENUM` 数据类型的 `ENUM.name` 参数现在是一个必需的关键字参数。在任何情况下，“name”都是必需的，以便在 SQL/DDL 渲染时如果缺少“name”则会引发错误。

+   **[postgresql] [变更]**

    为了支持新的 PostgreSQL 功能，包括 psycopg3 方言以及扩展的“快速插入多个”支持，已重新设计了将绑定参数的类型信息传递给 PostgreSQL 数据库的系统，以使用 SQL 编译器发出的内联转换，并且现在适用于所有 PostgreSQL 方言。这与以前的方法形成对比，以前的方法依赖于使用的 DBAPI 自己呈现这些转换，例如在 pg8000 和改进的 asyncpg 驱动程序的情况下，会使用 pep-249 的`setinputsizes()`方法，或者使用 psycopg2 驱动程序在大多数情况下依赖于驱动程序本身，对于一些特殊的情况，例如 ARRAY。

    新方法现在通过编译器在需要时呈现所有 PostgreSQL 方言的这些转换，而对于 PostgreSQL 方言，已删除了`setinputsizes()`的使用，因为这在任何情况下通常不是这些 DBAPI 的一部分（pg8000 是唯一的例外，它在 SQLAlchemy 开发人员的要求下添加了该方法）。

    这种方法的优点包括每个语句的性能，因为在执行时不需要对编译后的语句进行第二次遍历，更好地支持所有 DBAPI，因为现在有一个一致的应用类型信息的系统，以及改进的透明度，因为 SQL 日志输出以及编译语句的字符串输出将直接显示语句中存在的这些转换，而以前这些转换在日志输出中是不可见的，因为它们会在语句记录后发生。

+   **[postgresql] [错误]**

    `Operators.match()`运算符现在使用`plainto_tsquery()`进行 PostgreSQL 全文搜索，而不是`to_tsquery()`。进行此更改的原因是提供与其他数据库后端上的 match 更好的跨兼容性。通过与`Operators.bool_op()`（布尔运算符的改进版本`Operators.op()`）结合使用`func`可以获得所有 PostgreSQL 全文函数的全面支持。

    另请参阅

    PostgreSQL 上的 match()运算符使用 plainto_tsquery()而不是 to_tsquery()

    参考：[#7086](https://www.sqlalchemy.org/trac/ticket/7086)

+   **[postgresql] [已移除]**

    移除对多个已弃用驱动程序的支持：

    > +   用于 PostgreSQL 的 pypostgresql。这可作为外部驱动程序在[`github.com/PyGreSQL`](https://github.com/PyGreSQL)上获得
    > +   
    > +   用于 PostgreSQL 的 pygresql。

    请切换到支持的驱动程序之一或同一驱动程序的外部版本。

    参考：[#7258](https://www.sqlalchemy.org/trac/ticket/7258)

+   **[postgresql] [方言]**

    增加了对`psycopg`方言的支持，支持同步和异步执行。该方言在`create_engine()`和`create_async_engine()`引擎创建函数下可用，名称为`postgresql+psycopg`。

    另请参阅

    Dialect support for psycopg 3 (a.k.a. “psycopg”)

    psycopg

    参考：[#6842](https://www.sqlalchemy.org/trac/ticket/6842)

+   **[postgresql] [psycopg2]**

    更新 psycopg2 方言以使用 DBAPI 接口执行两阶段事务。以前是执行 SQL 命令来处理此类事务。

    参考：[#7238](https://www.sqlalchemy.org/trac/ticket/7238)

+   **[postgresql] [schema]**

    引入了类型`JSONPATH`，可用于转换表达式中。在使用诸如`jsonb_path_exists`或`jsonb_path_match`等接受`jsonpath`作为输入的函数时，某些 PostgreSQL 方言需要此类型。

    另请参阅

    JSON 类型 - PostgreSQL JSON 类型。

    参考：[#8216](https://www.sqlalchemy.org/trac/ticket/8216)

+   **[postgresql] [reflection]**

    PostgreSQL 方言现在支持基于表达式的索引的反射。当使用`Inspector.get_indexes()`或使用`Table`的`Table.autoload_with`(../core/metadata.html#sqlalchemy.schema.Table.params.autoload_with "sqlalchemy.schema.Table")反射表时，支持反射。感谢 immerrr 和 Aidan Kane 在此票上的帮助。

    参考：[#7442](https://www.sqlalchemy.org/trac/ticket/7442)

### mysql

+   **[mysql] [usecase] [mariadb]**

    `ROLLUP`函数现在将在 MySql 和 MariaDB 上正确呈现`WITH ROLLUP`，允许在这些后端使用 group by rollup。

    参考：[#8503](https://www.sqlalchemy.org/trac/ticket/8503)

+   **[mysql] [bug]**

    修复了 MySQL 中`Insert.on_duplicate_key_update()`中的问题，当在 VALUES 表达式中使用表达式时，会呈现错误的列名。感谢 Cristian Sabaila 的拉取请求。

    此更改也**回溯**到：1.4.27

    参考：[#7281](https://www.sqlalchemy.org/trac/ticket/7281)

+   **[mysql] [removed]**

    移除了对 MySQL 和 MariaDB 的 OurSQL 驱动程序的支持，因为该驱动程序似乎没有得到维护。

    参考：[#7258](https://www.sqlalchemy.org/trac/ticket/7258)

### mariadb

+   **[mariadb] [usecase]**

    添加了一个新的执行选项 `is_delete_using=True`，当使用支持 ORM 的 DELETE 语句与“fetch”同步策略配合使用时，ORM 会消耗这个选项；此选项表示预计 DELETE 语句将使用多个表，在 MariaDB 上是 DELETE..USING 语法。然后，该选项指示对于已知不支持“DELETE..USING..RETURNING”语法但支持“DELETE..USING”语法的数据库，不应该使用在 SQLAlchemy 2.0 中新实现的用于 MariaDB 的 RETURNING（对于[#7011](https://www.sqlalchemy.org/trac/ticket/7011)）。

    这个选项的理由是，ORM 启用的 DELETE 当前不知道是否要删除多个表，直到编译发生，而无论如何，这都会被缓存，但需要知道这一点，以便提前发出用于将要删除的行的 SELECT。不是对所有 DELETE 语句都应用跨越式性能惩罚，以便针对这个相对不常见的 SQL 模式主动检查它们，而是通过在编译步骤中引发一个新的异常消息来请求 `is_delete_using=True` 执行选项。当满足以下条件时，这个异常消息特别（且仅仅）被引发：语句是启用 ORM 的 DELETE，请求了“fetch”同步策略；后端是 MariaDB 或其他具有这种特定限制的后端；在初始编译中检测到该语句否则将发出“DELETE..USING..RETURNING”。通过应用执行选项，ORM 知道要预先运行 SELECT。类似的选项也适用于启用 ORM 的 UPDATE，但目前还没有需要该选项的后端。

    引用：[#8344](https://www.sqlalchemy.org/trac/ticket/8344)

+   **[mariadb] [用例]**

    为 MariaDB 方言添加了 INSERT..RETURNING 和 DELETE..RETURNING 支持。MariaDB 尚不支持 UPDATE..RETURNING。MariaDB 从 10.5.0 开始支持 INSERT..RETURNING，从 10.0.5 开始支持 DELETE..RETURNING。

    引用：[#7011](https://www.sqlalchemy.org/trac/ticket/7011)

### sqlite

+   **[sqlite] [用例]**

    为 SQLite 的反射方法添加了一个新的参数 `sqlite_include_internal=True`；当省略时，以 `sqlite_` 为前缀的本地表（根据 SQLite 文档标记为“内部模式”表，例如为支持“AUTOINCREMENT”列而生成的`sqlite_sequence`表）将不会包括在返回本地对象列表的反射方法中。这样做可以防止在使用 Alembic 自动生成时出现问题，之前，Alembic 会将这些 SQLite 生成的表视为从模型中移除。

    另请参阅

    反映内部模式表

    引用：[#8234](https://www.sqlalchemy.org/trac/ticket/8234)

+   **[sqlite] [用例]**

    为 SQLite 方言添加了对 RETURNING 的支持。SQLite 自版本 3.35 起支持 RETURNING。

    参考：[#6195](https://www.sqlalchemy.org/trac/ticket/6195)

+   **[sqlite] [usecase]**

    SQLite 方言现在支持 UPDATE..FROM 语法，用于 UPDATE 语句可能在语句的 WHERE 条件中引用其他表而无需使用子查询的情况。当使用多个表或其他实体或可选择项时，使用 `Update` 构造时会自动调用此语法。

    参考：[#7185](https://www.sqlalchemy.org/trac/ticket/7185)

+   **[sqlite] [performance] [bug]**

    当使用基于文件的数据库时，SQLite 方言现在默认使用 `QueuePool`。这是通过将 `check_same_thread` 参数设置为 `False` 来完成的。观察到，之前默认使用 `NullPool` 的方法，该方法在释放连接后不会保留数据库连接，实际上会产生可测量的负面性能影响。与往常一样，可以通过 `create_engine.poolclass` 参数来自定义池类。

    参见

    SQLite 方言为基于文件的数据库使用 QueuePool

    参考：[#7490](https://www.sqlalchemy.org/trac/ticket/7490)

+   **[sqlite] [performance] [usecase]**

    SQLite 的 datetime、date 和 time 数据类型现在使用 Python 标准库的 `fromisoformat()` 方法来解析传入的 datetime、date 和 time 字符串值。这种方法相比之前基于正则表达式的方法提高了性能，并且自动适应包含六位数字“微秒”格式或三位数字“毫秒”格式的 datetime 和 time 格式。

    参考：[#7029](https://www.sqlalchemy.org/trac/ticket/7029)

+   **[sqlite] [bug]**

    移除了关于 `Numeric` 类型在 DBAPI 中不原生支持 Decimal 值的警告。该警告是针对 SQLite 的，因为 SQLite 没有任何真正的方法（除非使用额外的扩展或变通方法）来处理超过 15 个有效数字的精度数字值，因为它只使用浮点数学来表示数字。由于这是 SQLite 本身已知和记录的限制，而不是 pysqlite 驱动程序的怪癖，因此 SQLAlchemy 不需要为此发出警告。此更改不会修改精度数字的处理方式。在使用 SQLite 时，值可以继续按照配置的 `Numeric`、`Float` 和相关数据类型处理为 `Decimal()` 或 `float()`，只是不能在使用 SQLite 时保持超过 15 个有效数字的精度，除非使用其他表示形式，如字符串。

    参考：[#7299](https://www.sqlalchemy.org/trac/ticket/7299)

### mssql

+   **[mssql] [用例]**

    为 SQL Server 方言实现了“聚集索引”标志 `mssql_clustered` 的反映。拉取请求由 John Lennox 提供。

    参考：[#8288](https://www.sqlalchemy.org/trac/ticket/8288)

+   **[mssql] [用例]**

    当创建表时，添加了对 MSSQL 上表和列注释的支持。增加了对反映表注释的支持。感谢 Daniel Hall 在此拉取请求中的帮助。

    参考：[#7844](https://www.sqlalchemy.org/trac/ticket/7844)

+   **[mssql] [错误]**

    `mssql+pyodbc` 方言的 `use_setinputsizes` 参数现在默认为 `True`；这是为了使非 Unicode 字符串比较绑定到 pyodbc.SQL_VARCHAR 而不是 pyodbc.SQL_WVARCHAR，从而允许针对 VARCHAR 列的索引生效。为了使 `fast_executemany=True` 参数继续发挥作用，`use_setinputsizes` 模式现在在 `fast_executemany` 为 True 且正在使用的具体方法是 `cursor.executemany()` 时跳过了 `cursor.setinputsizes()` 调用，因为该方法不支持 setinputsizes。该更改还为被标记为 `Unicode` 或 `UnicodeText` 的值添加了适当的 pyodbc DBAPI 类型，以及将基本 `JSON` 数据类型更改为将 JSON 字符串值视为 `Unicode` 而不是 `String`。  

    参考：[#8177](https://www.sqlalchemy.org/trac/ticket/8177)

+   **[mssql] [已移除]**

    由于缺乏测试支持，移除了对 mxodbc 驱动程序的支持。ODBC 用户可以使用完全支持的 pyodbc 方言。

    参考：[#7258](https://www.sqlalchemy.org/trac/ticket/7258)

### oracle

+   **[oracle] [feature]**

    为新的 oracle 驱动程序 `oracledb` 添加支持。

    参见

    oracledb 的方言支持

    python-oracledb

    参考：[#8054](https://www.sqlalchemy.org/trac/ticket/8054)

+   **[oracle] [feature]**

    对包含显式“binary_precision”值的`FLOAT`数据类型实现了 DDL 和反射支持。使用 Oracle 特定的 `FLOAT` 数据类型，可以指定新参数 `FLOAT.binary_precision`，它将直接呈现 Oracle 的浮点类型的精度。此值在反射过程中解释。反射回 `FLOAT` 数据类型时，返回的数据类型之一是 `DOUBLE_PRECISION` 用于 126 的 `FLOAT` 的精度（这也是 Oracle 的 `FLOAT` 的默认精度），用于 63 的 `REAL`，以及用于自定义精度的 `FLOAT`，根据 Oracle 文档。

    作为此更改的一部分，当为 Oracle 生成 DDL 时，显式拒绝了通用 `Float.precision` 值，因为此精度无法准确转换为“二进制精度”；相反，错误消息鼓励使用 `TypeEngine.with_variant()`，以便可以精确选择 Oracle 的特定精度形式。这是一种不兼容的行为更改，因为以前的“精度”值对于 Oracle 是静默忽略的。

    参见

    新的 Oracle FLOAT 类型，具有二进制精度；不直接接受小数精度

    参考：[#5465](https://www.sqlalchemy.org/trac/ticket/5465)

+   **[oracle] [feature]**

    cx_Oracle 方言完全实现了“RETURNING”支持，涵盖了两种单独的功能类型：

    +   实现了多行 RETURNING，意味着现在对于生成多于一个 RETURNING 行的 DML 语句，将接收到多个 RETURNING 行。

    +   “executemany RETURNING” 也已实现 - 当使用 `cursor.executemany()` 时，这允许 RETURNING 在每个语句中产生一行。 此功能的实现为 ORM 插入带来了显著的性能改进，方式与 SQLAlchemy 1.4 更改 ORM 批量插入与 psycopg2 现在在大多数情况下批处理具有 RETURNING 的语句 中添加的方式相同。

    参考资料：[#6245](https://www.sqlalchemy.org/trac/ticket/6245)

+   **[oracle] [usecase]**

    Oracle 现在默认情况下将使用 FETCH FIRST N ROWS / OFFSET 语法来支持 Oracle 12c 及以上版本的 limit/offset。 当直接使用 `Select.fetch()` 时，此语法已经可用，现在也适用于 `Select.limit()` 和 `Select.offset()`。

    参考资料：[#8221](https://www.sqlalchemy.org/trac/ticket/8221)

+   **[oracle] [change]**

    在 Oracle 上的物化视图现在被反映为视图。 在 SQLAlchemy 的早期版本中，视图会在表名中返回，而不是在视图名中返回。 作为此更改的副作用，除非设置了 `views=True`，否则默认情况下它们不会被 `MetaData.reflect()` 反映。 要获取物化视图列表，请使用新的检查方法 `Inspector.get_materialized_view_names()`。

+   **[oracle] [bug]**

    调整了 cx_Oracle 和 oracledb 方言中的 BLOB / CLOB / NCLOB 数据类型，以根据 Oracle 开发人员的建议提高性能。

    参考资料：[#7494](https://www.sqlalchemy.org/trac/ticket/7494)

+   **[oracle] [bug]**

    关于 `create_engine.implicit_returning` 弃用的相关内容，现在“implicit_returning”功能在所有情况下都为 Oracle 方言启用； 以前，当检测到 Oracle 8/8i 版本时，该功能将被关闭，但在线文档表明这两个版本支持与现代版本相同的 RETURNING 语法。

    参考资料：[#6962](https://www.sqlalchemy.org/trac/ticket/6962)

+   **[oracle]**

    cx_Oracle 7 现在是 cx_Oracle 的最低版本。

### 杂项

+   **[removed] [sybase]**

    删除了在以前的 SQLAlchemy 版本中已弃用的“sybase”内部方言。 第三方方言支持可用。

    参见

    外部方言

    参考资料：[#7258](https://www.sqlalchemy.org/trac/ticket/7258)

+   **[removed] [firebird]**

    移除了在之前的 SQLAlchemy 版本中已弃用的“firebird”内部方言。第三方方言支持可用。

    另请参阅

    外部方言

    参考：[#7258](https://www.sqlalchemy.org/trac/ticket/7258)
