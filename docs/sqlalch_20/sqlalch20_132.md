# 0.9 变更日志

> 原文：[`docs.sqlalchemy.org/en/20/changelog/changelog_09.html`](https://docs.sqlalchemy.org/en/20/changelog/changelog_09.html)

## 0.9.10

发布日期：2015 年 7 月 22���

### orm

+   **[orm] [feature]**

    在`Query.column_descriptions`返回的字典中添加了一个新条目`"entity"`。这指的是由表达式引用的主 ORM 映射类或别名类。与现有的`"type"`条目相比，它将始终是一个映射实体，即使从列表达式中提取，或者如果给定的表达式是纯核心表达式，则为`None`。另请参见[#3403](https://www.sqlalchemy.org/trac/ticket/3403)，该修复了此功能中的一个回归，该回归在 0.9.10 中未发布，但在 1.0 版本中发布。

    参考：[#3320](https://www.sqlalchemy.org/trac/ticket/3320)

+   **[orm] [bug]**

    当使用`Query.update()`或`Query.delete()`方法时，`Query`不支持连接、子查询或特殊的 FROM 子句；如果已调用类似`Query.join()`或`Query.select_from()`的方法，而不是静默地忽略这些字段，将发出警告。从 1.0.0b5 开始，这将引发错误。

    参考：[#3349](https://www.sqlalchemy.org/trac/ticket/3349)

+   **[orm] [bug]**

    修复了在多个嵌套的`Session.begin_nested()`操作中状态跟踪失败的 bug，导致在内部保存点中更新的对象的“脏”标志无法传播，因此如果外部保存点被回滚，该对象将不会成为过期状态的一部分，因此将恢复到其数据库状态。

    参考：[#3352](https://www.sqlalchemy.org/trac/ticket/3352)

### engine

+   **[engine] [bug]**

    将字符串值`"none"`添加到`Pool.reset_on_return`参数中，作为`None`的同义词，以便所有设置都可以使用字符串值，从而允许像`engine_from_config()`这样的实用程序可以无问题地使用。

    参考：[#3375](https://www.sqlalchemy.org/trac/ticket/3375)

### sql

+   **[sql] [feature]**

    官方支持了`Insert.from_select()`内部 SELECT 中使用的 CTE。此行为在 0.9.9 版本之前意外生效，当时由于与 [#3248](https://www.sqlalchemy.org/trac/ticket/3248) 的无关更改而不再生效。请注意，这是 INSERT 之后、SELECT 之前 WITH 子句的呈现方式；在 INSERT、UPDATE、DELETE 的顶层呈现 CTE 的完整功能是一个后续版本的新功能。

    参考：[#3418](https://www.sqlalchemy.org/trac/ticket/3418)

+   **[sql] [bug]**

    修复了一个问题，即使用命名约定的 `MetaData` 对象将无法正确与 pickle 一起工作。由于跳过了该属性，如果使用反序列化的 `MetaData` 对象来基于其他表，则会导致不一致和失败。

    参考：[#3362](https://www.sqlalchemy.org/trac/ticket/3362)

### postgresql

+   **[postgresql] [bug]**

    修复了长期存在的错误，即在 psycopg2 方言中与非 ASCII 值和 `native_enum=False` 结合使用时，`Enum` 类型无法正确解码返回结果。这源于 PG `ENUM` 类型曾经是一个独立的类型，没有“非本机”选项。

    参考：[#3354](https://www.sqlalchemy.org/trac/ticket/3354)

### mysql

+   **[mysql] [bug] [pymysql]**

    当使用“executemany”操作并带有 unicode 参数时，修复了 PyMySQL 的 Unicode 支持。SQLAlchemy 现在将语句和绑定参数都作为 Unicode 对象传递，因为 PyMySQL 通常在内部使用字符串插值来生成最终语句，在 executemany 情况下仅对最终语句执行“encode”步骤。

    参考：[#3337](https://www.sqlalchemy.org/trac/ticket/3337)

+   **[mysql] [bug] [py3k]**

    修复了在 Py3K 上未正确使用 `ord()` 函数的 `BIT` 类型。感谢 David Marin 的拉取请求。

    参考：[#3333](https://www.sqlalchemy.org/trac/ticket/3333)

### sqlite

+   **[sqlite] [bug]**

    修复了 SQLite 方言中的一个错误，即不会反映包含非字母字符（如点或空格）的唯一约束的反射问题。

    参考：[#3495](https://www.sqlalchemy.org/trac/ticket/3495)

### 测试

+   **[tests] [bug] [pypy]**

    修复了一个导入问题，阻止了“pypy setup.py test”的正确工作。

    参考：[#3406](https://www.sqlalchemy.org/trac/ticket/3406)

### 杂项

+   **[bug] [ext]**

    修复了在使用扩展属性仪器系统时的错误，当使用 `class_mapper()` 调用无效输入（同时也不是弱引用可引用的）时，不会引发正确的异常，例如整数。

    参考：[#3408](https://www.sqlalchemy.org/trac/ticket/3408)

+   **[bug] [ext]**

    修复了从 0.9.9 开始的回归，其中 `as_declarative()` 符号从 `sqlalchemy.ext.declarative` 命名空间中移除。

    参考：[#3324](https://www.sqlalchemy.org/trac/ticket/3324)

## 0.9.9

发布日期：2015 年 3 月 10 日

### orm

+   **[orm] [feature]**

    添加了新的参数 `Session.connection.execution_options`，可以在事务开始之前，当 `Connection` 第一次被检出时设置执行选项。这用于在事务开始之前在连接上设置诸如隔离级别等选项。

    另请参阅

    设置事务隔离级别 / DBAPI AUTOCOMMIT - 新文档部分，详细介绍了使用会话设置事务隔离的最佳实践。

    参考：[#3296](https://www.sqlalchemy.org/trac/ticket/3296)

+   **[orm] [feature]**

    添加了新的方法 `Session.invalidate()`，功能类似于 `Session.close()`，但还会在所有连接上调用 `Connection.invalidate()`，确保它们不会返回到连接池。在某些情况下很有用，比如处理 gevent 超时时，不能进一步使用连接，即使是回滚也不安全。

+   **[orm] [bug]**

    修复了 ORM 对象比较中的错误，当比较多对一 `!= None` 时，如果源是别名类，或者查询需要对表达式应用特殊别名处理，例如由于别名连接或多态查询导致，比较将失败；还修复了当将多对一与对象状态进行比较时，如果查询需要对别名连接或多态查询应用特殊别名处理，比较将失败的情况。

    参考：[#3310](https://www.sqlalchemy.org/trac/ticket/3310)

+   **[orm] [bug]**

    修复了一个 bug，在`Session`的`after_rollback()`处理程序错误地在处理程序内向该`Session`添加状态时，内部断言将失败，并且尝试继续的任务警告和删除此状态（由[#2389](https://www.sqlalchemy.org/trac/ticket/2389)建立）。

    参考：[#3309](https://www.sqlalchemy.org/trac/ticket/3309)

+   **[orm] [bug]**

    修复了一个 bug，当使用未知的 kw 参数调用`Query.join()`时会引发 TypeError，由于格式错误导致其自身引发 TypeError。感谢 Malthe Borch 的拉取请求。

+   **[orm] [bug]**

    修复了懒加载 SQL 构造中的 bug，其中一个复杂的 primaryjoin 在“指向自身的列”的自引用连接样式中多次引用相同的“本地”列时，在所有情况下都不会被替换。这里确定替换的逻辑已经重新制定为更加开放式。

    参考：[#3300](https://www.sqlalchemy.org/trac/ticket/3300)

+   **[orm] [bug]**

    “通配符”加载器选项，特别是由`load_only()`选项设置的选项，以覆盖未明确提及的所有属性，现在考虑到给定实体的超类，如果该实体使用继承映射进行映射，则超类中的属性名称也将从加载中省略。此外，多态鉴别器列无条件地包含在列表中，就像主键列一样，因此即使设置了 load_only()，子类型的多态加载仍将正常工作。

    参考：[#3287](https://www.sqlalchemy.org/trac/ticket/3287)

+   **[orm] [bug] [pypy]**

    修复了一个 bug，在`Query`在获取结果之前抛出异常时，特别是当无法形成行处理器时，游标会保持打开状态，结果仍在等待中，实际上并未关闭。这通常只在像 PyPy 这样的解释器上出现问题，其中游标不会立即被 GC 回收，并且在某些情况下可能导致事务/锁定的持续时间超过所需时间。

    参考：[#3285](https://www.sqlalchemy.org/trac/ticket/3285)

+   **[orm] [bug]**

    修复了一个泄漏问题，该问题会在不支持且极不推荐的情况下发生，即多次替换固定映射类上的关系，引用任意增长的目标映射器数量。当替换旧关系时会发出警告，但是如果映射已用于查询，则旧关系仍将在某些注册表中被引用。

    参考：[#3251](https://www.sqlalchemy.org/trac/ticket/3251)

+   **[orm] [bug] [sqlite]**

    修复了表达式变异可能表现为在使用`Query`从多个匿名列实体中选择时出现“无法找到列”错误的 bug，当针对 SQLite 查询时，作为 SQLite 方言使用的“join rewriting”功能的副作用。

    参考：[#3241](https://www.sqlalchemy.org/trac/ticket/3241)

+   **[orm] [bug]**

    修复了当使用`of_type()`将单继承子类连接到`Query.join()`和`Query.outerjoin()`的 ON 子句时，如果设置了`from_joinpoint=True`标志，则不会在 ON 子句中呈现“单表条件”的 bug。

    参考：[#3232](https://www.sqlalchemy.org/trac/ticket/3232)

### examples

+   **[examples] [bug]**

    更新了带有历史表的版本控制示例，使映射列重新映射以匹配列名称以及列的分组；特别是，这允许在同一列命名的连接继承场景中明确分组的列在历史映射中以相同的方式映射，避免了 0.9 系列中关于此模式的警告，并允许属��键的相同视图。

+   **[examples] [bug]**

    修复了示例/generic_associations/discriminator_on_association.py 中的 bug，在这里，AddressAssociation 的子类未被映射为“单表继承”，导致在尝试进一步使用映射时出现问题。

### engine

+   **[engine] [feature]**

    添加了用于查看事务隔离级别的新用户空间访问器；`Connection.get_isolation_level()`，`Connection.default_isolation_level`。

+   **[engine] [bug]**

    修复了`Connection`和池中的 bug，当使用`isolation_level`参数与`Connection.execution_options()`一起使用时，`Connection.invalidate()`方法或由于数据库断开连接而导致的失效会失败；重置隔离级别的“finalizer”将在不再打开的连接上调用。

    参考：[#3302](https://www.sqlalchemy.org/trac/ticket/3302)

+   **[engine] [bug]**

    当 `Connection.execution_options()` 在使用 `Transaction` 时，如果使用了 `isolation_level` 参数，则会发出警告；DBAPIs 和/或 SQLAlchemy 方言（如 psycopg2、MySQLdb）可能会隐式回滚或提交事务，或者在下一个事务中不更改设置，因此这是不安全的。

    参考：[#3296](https://www.sqlalchemy.org/trac/ticket/3296)

### sql

+   **[sql] [bug]**

    在 `Enum` 的 `__repr__()` 输出中添加了 `native_enum` 标志，当与 Alembic autogenerate 一起使用时，这主要是重要的。贡献者：Dimitris Theodorou。

+   **[sql] [bug]**

    修复了使用实现了也是 `TypeDecorator` 的 `TypeDecorator` 会在针对使用此类型的对象进行任何类型的 SQL 比较表达式时失败的问题，Python 会报错“Cannot create a consistent method resolution order (MRO)”。

    参考：[#3278](https://www.sqlalchemy.org/trac/ticket/3278)

+   **[sql] [bug]**

    修复了在 INSERT 中嵌入 SELECT 时的问题，无论是通过 values 子句还是作为“from select”，当两个语句共享相同名称的列时，将污染由 RETURNING 子句生成的结果集中使用的列类型，导致检索返回行时可能出现错误或错误适应。

    参考：[#3248](https://www.sqlalchemy.org/trac/ticket/3248)

### schema

+   **[schema] [bug]**

    修复了 0.9 版本中外键设置系统中的错误，即当外键与父级链接时，如果外键在稍后才会接收到其父级列，例如在反射 + “useexisting” 场景中，如果目标列实际上具有不同于其名称的键值，则使用了“link_to_name=True”的外键可能会失败，因为在反射中如果使用了列反射事件来更改反映的 `Column` 对象的 .key，使 link_to_name 变得重要。同样，在目标列具有不同键并且使用 link_to_name 引用时，也以类似的方式修复了通过 FK 传输列类型的支持。

    参考：[#1765](https://www.sqlalchemy.org/trac/ticket/1765), [#3298](https://www.sqlalchemy.org/trac/ticket/3298)

### postgresql

+   **[postgresql] [feature]**

    添加了对 PostgreSQL 索引使用 `CONCURRENTLY` 关键字的支持，使用 `postgresql_concurrently` 建立。感谢 Iuri de Silvio 的拉取请求。

    另请参见

    并发索引

+   **[postgresql] [bug]**

    修复了在使用 psycopg2 时支持 PostgreSQL UUID 类型与 ARRAY 类型的问题。现在 psycopg2 方言使用 psycopg2.extras.register_uuid() 钩子，以便始终将 UUID 值作为 UUID() 对象传递到/从 DBAPI。`UUID.as_uuid` 标志仍然受到尊重，但在 psycopg2 中，当禁用此标志时，我们需要将返回的 UUID 对象转换回字符串。

    参考：[#2940](https://www.sqlalchemy.org/trac/ticket/2940)

+   **[postgresql] [bug]**

    添加了对 psycopg2 2.5.4 或更高版本使用 `postgresql.JSONB` 数据类型的支持，该版本具有 JSONB 数据的本机转换，因此必须禁用 SQLAlchemy 的转换器；此外，新添加的 psycopg2 扩展 `extras.register_default_jsonb` 用于通过 `json_deserializer` 参数传递给方言的 JSON 反序列化器。还修复了实际上未循环传输 JSONB 类型而不是 JSON 类型的 PostgreSQL 集成测试。感谢 Mateusz Susik 的拉取请求。

+   **[postgresql] [bug]**

    修复了在使用旧版本 psycopg2 < 2.4.3 时注册 HSTORE 类型时使用“array_oid”标志的问题，该版本不支持此标志，以及在使用 psycopg2 版本 < 2.5 时使用本机 json 序列化器钩子“register_default_json”与用户定义的`json_deserializer`时，该版本不包括本机 json。

+   **[postgresql] [bug]**

    修复了 PostgreSQL 方言在 `Index` 中未能呈现表绑定列的表达式的 bug；通常当一个 `text()` 构造是索引中的表达式之一时；或者如果其中一个或多个是这样的表达式，则可能会误解表达式列表。

    参考：[#3174](https://www.sqlalchemy.org/trac/ticket/3174)

### mysql

+   **[mysql] [change]**

    `gaerdbms` 方言不再必要，并发出弃用警告。Google 现在建议直接使用 MySQLdb 方言。

    参考：[#3275](https://www.sqlalchemy.org/trac/ticket/3275)

+   **[mysql] [bug]**

    在 MySQLdb 方言周围添加了一个版本检查，用于检查‘utf8_bin’排序规则，因为这在 MySQL 服务器 < 5.0 上失败。

    参考：[#3274](https://www.sqlalchemy.org/trac/ticket/3274)

### sqlite

+   **[sqlite] [feature]**

    添加了对 SQLite 上部分索引（例如带有 WHERE 子句）的支持。感谢 Kai Groner 的拉取请求。

    另请参见

    部分索引

+   **[sqlite] [feature]**

    添加了一个新的 SQLite 后端用于 SQLCipher 后端。该后端使用 pysqlcipher Python 驱动程序提供加密的 SQLite 数据库，该驱动程序与 pysqlite 驱动程序非常相似。

    另请参阅

    `pysqlcipher`

### 杂项

+   **[bug] [ext] [py3k]**

    修复了在 Py3K 下关联代理列表类无法正确解释切片的 bug。感谢 Gilles Dartiguelongue 提供的拉取请求。

## 0.9.8

发布日期：2014 年 10 月 13 日

### orm

+   **[orm] [bug] [engine]**

    修复了一个影响与[#3199](https://www.sqlalchemy.org/trac/ticket/3199)相同类别事件的 bug，当使用`named=True`参数时会出现问题。一些事件无法注册，其他事件无法正确调用事件参数，通常在事件被“包装”以适应其他方式时。已重新排列“named”机制，以不干扰内部包装函数期望的参数签名。

    参考：[#3197](https://www.sqlalchemy.org/trac/ticket/3197)

+   **[orm] [bug]**

    修复了一个影响许多事件类的 bug，特别是 ORM 事件，但也包括引擎事件，在这些事件中，“去重复”一个冗余调用`listen()`的常规逻辑失败，对于那些监听器函数被包装的事件。在 registry.py 中会触发一个断言。现在这个断言已经整合到去重复检查中，另外还有一个更简单的检查去重复的方法。

    参考：[#3199](https://www.sqlalchemy.org/trac/ticket/3199)

+   **[orm] [bug]**

    修复了一个警告，当一个复杂的自引用主连接包含函数时会发出警告，同时指定了 remote_side；警告会建议设置“remote side”。现在只有在 remote_side 不存在时才会发出警告。

    参考：[#3194](https://www.sqlalchemy.org/trac/ticket/3194)

### orm 声明式

+   **[orm] [declarative] [bug]**

    在使用`AbstractConcreteBase`与声明`__abstract__`的子类时修复了“‘NoneType’ object has no attribute ‘concrete’”错误。

    参考：[#3185](https://www.sqlalchemy.org/trac/ticket/3185)

### 引擎

+   **[engine] [bug]**

    通过`create_engine.execution_options`或`Engine.update_execution_options()`传递给`Engine`的执行选项不会传递给在“第一次连接”事件中初始化方言的特殊`Connection`；方言通常会在此阶段执行自己的查询，并且不应应用任何当前可用的选项。特别是，“autocommit”选项导致在此初始连接中尝试自动提交，由于`Connection`的非标准状态而导致 AttributeError 失败。

    参考：[#3200](https://www.sqlalchemy.org/trac/ticket/3200)

+   **[引擎] [错误]**

    用于确定 INSERT 或 UPDATE 影响的列的字符串键现在在它们对“compiled cache”缓存键的贡献时排序。这些键之前没有确定性地排序，这意味着相同的语句可能会基于等效键多次被缓存，这既会在内存方面带来成本，也会在性能方面带来成本。

    参考：[#3165](https://www.sqlalchemy.org/trac/ticket/3165)

### sql

+   **[sql] [错误]**

    修复了在 sql 包中有相当数量的 SQL 元素无法成功执行`__repr__()`的 bug，因为缺少`description`属性，这将导致一个内部 AttributeError 再次调用`__repr__()`时递归溢出。

    参考：[#3195](https://www.sqlalchemy.org/trac/ticket/3195)

+   **[sql] [错误]**

    调整表/索引反射，如果索引报告的列在表中未找到，则发出警告并跳过该列。这可能会发生在一些特殊的系统列情况下，如在 Oracle 中观察到的情况。

    参考：[#3180](https://www.sqlalchemy.org/trac/ticket/3180)

+   **[sql] [错误]**

    修复了 CTE 中的 bug，其中当一个 CTE 引用语句中的另一个别名 CTE 时，`literal_binds`编译器参数将不会始终正确传播。

    参考：[#3154](https://www.sqlalchemy.org/trac/ticket/3154)

+   **[sql] [错误]**

    修复了 0.9.7 版本的退化，由[#3067](https://www.sqlalchemy.org/trac/ticket/3067)引起，结合一个命名错误的单元测试，以至于所谓的“schema”类型如`Boolean`和`Enum`无法再被 pickle 化。

    参考：[#3067](https://www.sqlalchemy.org/trac/ticket/3067)，[#3144](https://www.sqlalchemy.org/trac/ticket/3144)

### postgresql

+   **[postgresql] [feature] [pg8000]**

    使用 pg8000 驱动程序添加了对“合理的多行计数”的支持，这主要适用于在 ORM 中使用版本控制时。该功能基于使用 pg8000 1.9.14 或更高版本进行版本检测。感谢 Tony Locke 的拉取请求。

+   **[postgresql] [bug]**

    重新访问了首次在 0.9.5 中修补的此问题，显然 psycopg2 的`.closed`访问器并不像我们假设的那样可靠，因此我们已添加了对异常消息“SSL SYSCALL error: Bad file descriptor”和“SSL SYSCALL error: EOF detected”进行显式检查，以检测到断开连接的情况。我们将继续将 psycopg2 的 connection.closed 作为首次检查。

    参考：[#3021](https://www.sqlalchemy.org/trac/ticket/3021)

+   **[postgresql] [bug]**

    修复了 PostgreSQL JSON 类型无法持久化或以其他方式呈现 SQL NULL 列值，而不是 JSON 编码的`'null'`的错误。为支持此情况，更改如下：

    +   现在可以指定值`null()`，这将始终导致结果语句中的 NULL 值。

    +   添加了一个新参数`JSON.none_as_null`，当为 True 时表示 Python 的`None`值应该持久化为 SQL NULL，而不是 JSON 编码的`'null'`。

    对于除了 psycopg2 之外的其他 DBAPI，如 pg8000，将 NULL 检索为 None 也已修复。

    参考：[#3159](https://www.sqlalchemy.org/trac/ticket/3159)

+   **[postgresql] [bug]**

    现在，DBAPI 错误的异常包装系统可以适应非标准的 DBAPI 异常，例如 psycopg2 的 TransactionRollbackError。这些异常现在将使用`sqlalchemy.exc`中最接近的可用子类引发，在 TransactionRollbackError 的情况下，使用`sqlalchemy.exc.OperationalError`。

    参考：[#3075](https://www.sqlalchemy.org/trac/ticket/3075)

+   **[postgresql] [bug]**

    修复了`array`对象中的错误，其中与普通的 Python 列表进行比较会导致使用不正确的数组构造函数。感谢 Andrew 的拉取请求。

    参考：[#3141](https://www.sqlalchemy.org/trac/ticket/3141)

+   **[postgresql] [bug]**

    为函数添加了支持的`FunctionElement.alias()`方法，例如`func`构造。先前，此方法的行为是未定义的。当前行为模仿了 0.9.4 之前的行为，即将函数转换为具有给定别名的单列 FROM 子句，其中列本身是匿名命名的。

    参考：[#3137](https://www.sqlalchemy.org/trac/ticket/3137)

### mysql

+   **[mysql] [bug] [mysqlconnector]**

    从版本 2.0 开始的 Mysqlconnector，可能是由于 Python 3 合并的副作用，现在不再期望百分号（例如作为模数运算符等）被加倍，即使使用“pyformat”绑定参数格式（Mysqlconnector 没有记录此更改）。 方言现在在检测模数运算符是否应该呈现为`%%`或`%`时，会检查 py2k 和 mysqlconnector 小于版本 2.0。

+   **[mysql] [bug] [mysqlconnector]**

    MySQLconnector 版本 2.0 及以上现在会传递 Unicode SQL；对于 Py2k 和 MySQL < 2.0，字符串会被编码。

### sqlite

+   **[sqlite] [bug]**

    当使用附加数据库文件从 UNION 进行选择时，pysqlite 驱动器在 cursor.description 中报告列名为‘dbname.tablename.colname’，而不是正常情况下的‘tablename.colname’（请注意，对于 UNION，它应该是‘colname’，但我们对此进行了处理）。 此处的列翻译逻辑已调整为检索最右侧的标记，而不是第二个标记，因此在两种情况下都有效。 解决方法由 Tony Roberts 提供。

    参考：[#3211](https://www.sqlalchemy.org/trac/ticket/3211)

### mssql

+   **[mssql] [bug]**

    修复了 pymssql 方言中版本字符串检测的问题，以便与 Microsoft SQL Azure 一起工作，后者将“SQL Server”更改为“SQL Azure”。

    参考：[#3151](https://www.sqlalchemy.org/trac/ticket/3151)

### oracle

+   **[oracle] [bug]**

    修复了 Oracle 方言中长期存在的 bug，即以数字开头的绑定参数名不会被引用，因为 Oracle 不喜欢绑定参数名中的数字。

    参考：[#2138](https://www.sqlalchemy.org/trac/ticket/2138)

### misc

+   **[bug] [declarative]**

    修复了在一些特殊的终端用户设置中观察到的不太可能的竞争条件，其中在声明性检查“重复类名”时会遇到未完全清理的弱引用，与其他被移除的类相关联；此处的检查现在确保在进一步调用之前弱引用仍然引用对象。

    参考：[#3208](https://www.sqlalchemy.org/trac/ticket/3208)

+   **[bug] [ext]**

    修复了排序列表中的 bug，在集合替换事件期间，如果 reorder_on_append 标志设置为 True，则项目的顺序会被打乱。 修复确保排序列表仅影响显式与对象相关联的列表。

    参考：[#3191](https://www.sqlalchemy.org/trac/ticket/3191)

+   **[bug] [ext]**

    修复了`MutableDict`中未能实现`update()`字典方法的错误，因此未捕获更改。 拉请求由 Matt Chisholm 提供。

+   **[bug] [ext]**

    修复了一个错误，即`MutableDict`的自定义子类在“强制”操作中不会显示，并且会返回一个普通的`MutableDict`。感谢 Matt Chisholm 的拉取请求。

+   **[bug] [pool]**

    修复了连接池日志记录中的错误，即如果使用`logging.setLevel()`设置日志记录而不是使用`echo_pool`标志，则“连接已检出”调试日志消息将不会发出。已添加用于断言此日志记录的测试。这是在 0.9.0 中引入的回归。

    参考：[#3168](https://www.sqlalchemy.org/trac/ticket/3168)

## 0.9.7

发布日期：2014 年 7 月 22 日

### orm

+   **[orm] [bug] [eagerloading]**

    修复了由 0.9.4 发布的[#2976](https://www.sqlalchemy.org/trac/ticket/2976)引起的回归，其中沿着一系列连接的急加载的“外连接”传播会错误地将兄弟连接路径上的“内连接”也转换为外连接，当只有后代路径应该接收“外连接”传播时；另外，修复了相关问题，即“嵌套”连接传播会不适当地发生在两个兄弟连接路径之间。

    参考：[#3131](https://www.sqlalchemy.org/trac/ticket/3131)

+   **[orm] [bug]**

    由于[#2736](https://www.sqlalchemy.org/trac/ticket/2736)导致的 0.9.0 的回归已修复，`Query.select_from()`方法不再正确设置`Query`对象的“from entity”，因此随后的`Query.filter_by()`或`Query.join()`调用将无法在按字符串名称搜索属性时检查适当的“from”实体。

    参考：[#2736](https://www.sqlalchemy.org/trac/ticket/2736)，[#3083](https://www.sqlalchemy.org/trac/ticket/3083)

+   **[orm] [bug]**

    对于 query.update()/delete()的“评估器”不适用于多表更新，并且需要设置为 synchronize_session=False 或 synchronize_session='fetch'；现在会发出警告。在 1.0 版本中，这将升级为完整的异常。

    参考：[#3117](https://www.sqlalchemy.org/trac/ticket/3117)

+   **[orm] [bug]**

    修复了在保存点块内持久化、删除或主键更改的项目在外部事务回滚后不参与恢复到其先前状态（不在会话中，在会话中，先前的 PK）的错误。

    参考：[#3108](https://www.sqlalchemy.org/trac/ticket/3108)

+   **[orm] [bug]**

    修复了子查询急加载与`with_polymorphic()`一起使用时的错误，子查询加载中实体和列的定位对于这种类型的实体和其他实体更加准确。

    参考：[#3106](https://www.sqlalchemy.org/trac/ticket/3106)

+   **[orm] [bug]**

    修复了涉及动态属性的错误，这是从版本 0.9.5 中的[#3060](https://www.sqlalchemy.org/trac/ticket/3060)再次出现的回归。具有`lazy='dynamic'`的自引用关系在刷新操作中会引发 TypeError。

    参考：[#3099](https://www.sqlalchemy.org/trac/ticket/3099)

### engine

+   **[engine] [feature]**

    添加了新的事件`ConnectionEvents.handle_error()`，这是`ConnectionEvents.dbapi_error()`的更全面和全面的替代品。

    参考：[#3076](https://www.sqlalchemy.org/trac/ticket/3076)

### sql

+   **[sql] [bug]**

    修复了`Enum`和其他`SchemaType`子类中的错误，直接将类型与`MetaData`关联会导致在`MetaData`上发���事件（如创建事件）时挂起。

    这个更改也被**回溯**到：0.8.7

    参考：[#3124](https://www.sqlalchemy.org/trac/ticket/3124)

+   **[sql] [bug]**

    修复了自定义操作符加`TypeEngine.with_variant()`系统中的错误，当与变体一起使用`TypeDecorator`时，使用比较运算符会导致 MRO 错误。

    这个更改也被**回溯**到：0.8.7

    参考：[#3102](https://www.sqlalchemy.org/trac/ticket/3102)

+   **[sql] [bug]**

    修复了命名约定功能中的错误，其中使用包含`constraint_name`的检查约定会强制所有`Boolean`和`Enum`类型也需要名称，因为这些隐式创建约束，即使最终目标后端不需要生成约束，比如 PostgreSQL。这些特定约束的命名约定机制已经重新组织，使得命名确定在 DDL 编译时完成，而不是在约束/表构建时完成。

    参考：[#3067](https://www.sqlalchemy.org/trac/ticket/3067)

+   **[sql] [bug]**

    修复了通用表达式中的错误，当 CTE 在某些方式中嵌套时，位置绑定参数可能以错误的最终顺序表达。

    参考：[#3090](https://www.sqlalchemy.org/trac/ticket/3090)

+   **[sql] [bug]**

    修复了多值`Insert`构造中的错误，使得对于字面 SQL 表达式给定的第一个值之外的后续值条目未能检查。

    参考：[#3069](https://www.sqlalchemy.org/trac/ticket/3069)

+   **[sql] [bug]**

    在 Python 版本< 2.6.5 中，为 dialect_kwargs 迭代添加了一个“str()”步骤，解决了“无 unicode 关键字参数”错误，因为这些参数在某些反射过程中作为关键字参数传递。

    参考：[#3123](https://www.sqlalchemy.org/trac/ticket/3123)

+   **[sql] [bug]**

    `TypeEngine.with_variant()`方��现在将接受一个类型类作为参数，内部将其转换为实例，使用了其他构造（如`Column`）长期建立的相同约定。

    参考：[#3122](https://www.sqlalchemy.org/trac/ticket/3122)

### postgresql

+   **[postgresql] [feature]**

    添加了`postgresql_regconfig`参数到`ColumnOperators.match()`操作符，允许指定“reg config”参数到`to_tsquery()`函数中。感谢 Jonathan Vanasco 的拉取请求。

    参考：[#3078](https://www.sqlalchemy.org/trac/ticket/3078)

+   **[postgresql] [feature]**

    通过`JSONB`添加了对 PostgreSQL JSONB 的支持。感谢 Damian Dimmich 的拉取请求。

+   **[postgresql] [bug] [pg8000]**

    修复了 0.9.5 版本中由新的 pg8000 隔离级别功能引入的错误，其中引擎级别的隔离级别参数在连接时会引发错误。

    参考：[#3134](https://www.sqlalchemy.org/trac/ticket/3134)

### mysql

+   **[mysql] [bug]**

    MySQL 错误 2014“commands out of sync”似乎在现代 MySQL-Python 版本中被提升为 ProgrammingError，而不是 OperationalError；现在所有被测试为“is disconnect”的 MySQL 错误代码都在 OperationalError 和 ProgrammingError 中进行检查。

    此更改也被**回溯**到：0.8.7

    参考：[#3101](https://www.sqlalchemy.org/trac/ticket/3101)

### sqlite

+   **[sqlite] [bug]**

    修复了 SQLite 连接重写问题，其中作为标量子查询嵌入的子查询（例如在 IN 中）会从包含查询中接收不适当的替换，如果相同的表在子查询中存在，并且在包含查询中也存在，例如在连接继承场景中。

    参考：[#3130](https://www.sqlalchemy.org/trac/ticket/3130)

### mssql

+   **[mssql] [功能]**

    为 SQL Server 2008 启用了“多值插入”。感谢 Albert Cervin 的拉取请求。还扩展了“IDENTITY INSERT”模式的检查，以包括当标识键出现在语句的 VALUEs 子句中时。

+   **[mssql] [错误]**

    将语句编码添加到“SET IDENTITY_INSERT”语句中，当显式插入插入到 IDENTITY 列时，以支持像 pyodbc + unix + py2k 这样不支持 unicode 语句的驱动程序上的非 ascii 表标识符。

    此更改也**回溯**到：0.8.7

+   **[mssql] [错误]**

    在 SQL Server pyodbc 方言中，修复了`description_encoding`方言参数的实现，当未明确设置时，会导致无法正确解析包含不同编码名称的结果集的 cursor.description。未来不应该需要此参数。

    此更改也**回溯**到：0.8.7

    参考：[#3091](https://www.sqlalchemy.org/trac/ticket/3091)

+   **[mssql] [错误]**

    修复了从 0.9.5 引起的回归，由[#3025](https://www.sqlalchemy.org/trac/ticket/3025)引起，其中用于确定“默认模式”的查询在 SQL Server 2000 中无效。对于 SQL Server 2000，我们回到默认的“模式名称”参数，该参数是可配置的，但默认为'dbo'。

    参考：[#3025](https://www.sqlalchemy.org/trac/ticket/3025)

### oracle

+   **[oracle] [错误] [测试]**

    修复了 oracle 方言测试套件中的错误，其中在一个测试中，假定‘用户名’在数据库 URL 中，即使这可能并非事实。 

    参考：[#3128](https://www.sqlalchemy.org/trac/ticket/3128)

### 测试

+   **[测试] [错误]**

    修复了“python setup.py test”未正确调用 distutils 的错误，导致在测试套件结束时会发出错误。

### 杂项

+   **[错误] [声明性]**

    修复了当声明性`__abstract__`标志未被区分为实际值`False`时的错误。`__abstract__`标志需要在被测试的级别上实际评估为 True 值。

    参考：[#3097](https://www.sqlalchemy.org/trac/ticket/3097)

## 0.9.6

发布日期：2014 年 6 月 23 日

### orm

+   **[orm] [错误]**

    撤销了[#3060](https://www.sqlalchemy.org/trac/ticket/3060)的更改 - 这是一个在 1.0 中更全面更新的工作单元修复，通过[#3061](https://www.sqlalchemy.org/trac/ticket/3061)。[#3060](https://www.sqlalchemy.org/trac/ticket/3060)中的修复不幸地产生了一个新问题，即对多对一属性的急加载可能会产生被解释为属性更改的事件。

    参考：[#3060](https://www.sqlalchemy.org/trac/ticket/3060)

## 0.9.5

发布日期：2014 年 6 月 23 日

### orm

+   **[orm] [功能]**

    “primaryjoin”模型已经进一步扩展，允许严格从单个列到自身的连接条件，通过某种 SQL 函数或表达式进行转换。这有点实验性质，但第一个概念验证是“材料化路径”连接条件，其中路径字符串使用“like”与自身进行比较。`ColumnOperators.like()` 操作符也已添加到可在 primaryjoin 条件中使用的有效操作符列表中。

    参考：[#3029](https://www.sqlalchemy.org/trac/ticket/3029)

+   **[orm] [feature]**

    添加了新的实用函数`make_transient_to_detached()`，可用于制造行为就像它们从会话中加载然后分离的对象。不存在的属性被标记为过期，并且对象可以添加到一个会话中，它将表现得像一个持久对象。

    参考：[#3017](https://www.sqlalchemy.org/trac/ticket/3017)

+   **[orm] [bug]**

    修复了子查询急加载中的错误，当跨多态子类边界的长链急加载与多态加载一起使用时，会无法定位链中的子类链接，导致在`AliasedClass`上出现缺少属性名称的错误。

    此更改也已**回溯**至：0.8.7

    参考：[#3055](https://www.sqlalchemy.org/trac/ticket/3055)

+   **[orm] [bug]**

    修复了 ORM 中的 bug，`class_mapper()` 函数会掩盖应该在映射器配置期间由于用户错误而引发的 AttributeErrors 或 KeyErrors。对于属性/键错误的捕获已经更具体，不包括配置步骤。

    此更改也已**回溯**至：0.8.7

    参考：[#3047](https://www.sqlalchemy.org/trac/ticket/3047)

+   **[orm] [bug]**

    已添加额外的检查，用于处理继承映射器隐式组合其基于列的属性之一与父级属性的情况，其中这些列通常不一定共享相同的值。这是通过[#1892](https://www.sqlalchemy.org/trac/ticket/1892)添加的现有检查的扩展；然而，这个新检查只发出警告，而不是异常，以允许依赖于现有行为的应用程序。

    参见

    我收到关于“隐式组合列 X 在属性 Y 下”的警告或错误

    参考：[#3042](https://www.sqlalchemy.org/trac/ticket/3042)

+   **[orm] [bug]**

    修改了 `load_only()` 的行为，使得主键列始终被添加到“未延迟加载”列的列表中；否则，ORM 无法加载行的标识。显然，可以延迟映射的主键，ORM 将失败，这一点没有改变。但是，由于 load_only 本质上是说“除了 X 之外都延迟加载”，因此 PK 列不参与此延迟加载更为关键。

    引用：[#3080](https://www.sqlalchemy.org/trac/ticket/3080)

+   **[orm] [bug]**

    修复了在所谓的“行切换”场景中出现的一些边缘情况，其中 INSERT/DELETE 可以转换为 UPDATE。在这种情况下，将多对一关系设置为 None，或在某些情况下将标量属性设置为 None，可能不会被检测为值的净变化，因此 UPDATE 不会重置前一行上的内容。这是由于属性历史的一些尚未解决的副作用，这些副作用涉及隐式假定 None 对于先前未设置的属性实际上不是“变化”。另请参见 [#3061](https://www.sqlalchemy.org/trac/ticket/3061)。

    注意

    此更改已在 0.9.6 版本中**撤销**。完整的修复将在 SQLAlchemy 的 1.0 版本中实现。

    引用：[#3060](https://www.sqlalchemy.org/trac/ticket/3060)

+   **[orm] [bug]**

    关联到 [#3060](https://www.sqlalchemy.org/trac/ticket/3060)，对工作单元进行了调整，以便在要删除的自引用对象图中，加载相关的多对一对象更为积极；加载相关对象有助于确定删除顺序的正确顺序，如果未设置 passive_deletes。

+   **[orm] [bug]**

    修复了 SQLite 连接重写中的错误，其中由于重复导致的匿名列名在子查询中不会被正确重写。这会影响带有任何类型子查询 + 连接的 SELECT 查询。

    引用：[#3057](https://www.sqlalchemy.org/trac/ticket/3057)

+   **[orm] [bug] [sql]**

    修复了 [#2804](https://www.sqlalchemy.org/trac/ticket/2804) 中新增的布尔强制转换的问题，新规则对于“where”和“having”在“whereclause”和“having” kw 参数上不会生效，这也是 `select()` 构造函数使用的内容，因此在 ORM 中也无法正常工作。

    引用：[#3013](https://www.sqlalchemy.org/trac/ticket/3013)

### 示例

+   **[examples] [feature]**

    添加了一个新示例，演示了使用最新关系特性的物化路径。示例由 Jack Zhou 提供。

### 引擎

+   **[engine] [bug]**

    修复了一个 bug，当引擎首次连接并进行初始检查时发生 DBAPI 异常，异常不是断开连接异常，但是当我们尝试关闭光标时光标引发错误。在这种情况下，真正的异常将被压制，因为我们试图通过连接池记录光标关闭异常并失败，因为我们试图以不适合这种非常特定情况的方式访问池的记录器。

    参考：[#3063](https://www.sqlalchemy.org/trac/ticket/3063)

+   **[engine] [bug]**

    检测到一些“双重无效”情况，其中连接无效可能发生在已经关键的部分内，比如连接关闭(); 最终，这些条件是由于[#2907](https://www.sqlalchemy.org/trac/ticket/2907)中的更改引起的，因为“返回时重置”功能调用 Connection/Transaction 来处理它，其中可能会捕获“断开检测”。然而，最近在[#2985](https://www.sqlalchemy.org/trac/ticket/2985)中的更改使得这种情况更有可能被视为“连接无效”操作更快，因为在 0.9.4 上更容易复现这个问题，而在 0.9.3 上更难。

    现在在可能发生无效的任何部分都添加了检查，以阻止在无效连接上发生进一步的不允许操作。这包括两个修复，一个在引擎级别，一个在池级别。虽然这个问题在高并发的 gevent 情况下被观察到，但理论上在任何发生断开连接的情况下都可能发生，这发生在连接关闭操作中。

    参考：[#3043](https://www.sqlalchemy.org/trac/ticket/3043)

### sql

+   **[sql] [feature]**

    在`Index`的合同中稍微放宽了一点，你可以指定一个`text()`表达式作为目标；如果要手动将索引添加到表中，那么索引不再需要存在绑定表列，可以通过内联声明或通过`Table.append_constraint()`添加。

    参考：[#3028](https://www.sqlalchemy.org/trac/ticket/3028)

+   **[sql] [feature]**

    添加了新标志`between.symmetric`，当设置为 True 时呈现“BETWEEN SYMMETRIC”。还添加了一个新的否定运算符“notbetween_op”，现在允许像`~col.between(x, y)`这样的表达式呈现为“col NOT BETWEEN x AND y”，而不是一个带括号的 NOT 字符串。

    参考：[#2990](https://www.sqlalchemy.org/trac/ticket/2990)

+   **[sql] [bug]**

    修复了在 INSERT..FROM SELECT 结构中的 bug，在从 UNION 中选择时，会将 UNION 包装在一个匿名（例如未标记）子查询中。

    这个更改也**回溯**到：0.8.7

    参考：[#3044](https://www.sqlalchemy.org/trac/ticket/3044)

+   **[sql] [bug]**

    修复了当应用空的`and_()`或`or_()`或其他空表达式时，`Table.update()`和`Table.delete()`会生成空的 WHERE 子句的错误。现在这与`select()`的行为一致。

    此更改也已**回溯**到：0.8.7

    参考：[#3045](https://www.sqlalchemy.org/trac/ticket/3045)

+   **[sql] [bug]**

    当在表的显式`PrimaryKeyConstraint`中引用该`Column`时，`Column.nullable`标志隐式设置为`False`。此行为现在与当`Column`本身的`Column.primary_key`标志设置为`True`时的行为相匹配，这意味着这是一个完全等价的情况。

    参考：[#3023](https://www.sqlalchemy.org/trac/ticket/3023)

+   **[sql] [bug]**

    修复了`Operators.__and__()`、`Operators.__or__()`和`Operators.__invert__()`运算符重载方法无法在自定义`Comparator`实现中被覆盖的错误。

    参考：[#3012](https://www.sqlalchemy.org/trac/ticket/3012)

+   **[sql] [bug]**

    修复了新的`DialectKWArgs.argument_for()`方法中的错误，其中为以前未包含任何特殊参数的构造添加参数将失败。

    参考：[#3024](https://www.sqlalchemy.org/trac/ticket/3024)

+   **[sql] [bug]**

    修复了 0.9 版本引入的回归问题，即新的“ORDER BY <labelname>”功能从[#1068](https://www.sqlalchemy.org/trac/ticket/1068)中不会对标签名称应用引用规则，如在 ORDER BY 中呈现的那样。

    参考：[#1068](https://www.sqlalchemy.org/trac/ticket/1068), [#3020](https://www.sqlalchemy.org/trac/ticket/3020)

+   **[sql] [bug]**

    恢复了`Function`的导入到`sqlalchemy.sql.expression`导入命名空间，这在 0.9 版本开始时被移除。

### postgresql

+   **[postgresql] [feature]**

    在使用 pg8000 DBAPI 时，添加了对 AUTOCOMMIT 隔离级别的支持。拉取请求由 Tony Locke 提供。

+   **[postgresql] [feature]**

    向 PostgreSQL `ARRAY` 类型添加了一个新标志`ARRAY.zero_indexes`。当设置为`True`时，将在传递给数据库之前将所有数组索引值加一，从而在 Python 风格的零基索引和 PostgreSQL 基索引之间实现更好的互操作性。拉取请求由 Alexey Terentev 提供。

    参考：[#2785](https://www.sqlalchemy.org/trac/ticket/2785)

+   **[postgresql] [bug]**

    在 PG `HSTORE` 类型中添加了`hashable=False`标志，这是为了允许 ORM 在请求混合列/实体列表时跳过尝试对 ORM 映射的 HSTORE 列进行“哈希”操作。补丁由 Gunnlaugur Þór Briem 提供。

    这个更改也被**回溯**到：0.8.7

    参考：[#3053](https://www.sqlalchemy.org/trac/ticket/3053)

+   **[postgresql] [bug]**

    添加了一个新的“断开连接”消息“连接意外关闭”。这似乎与较新版本的 SSL 有关。拉取请求由 Antti Haapala 提供。

    这个更改也被**回溯**到：0.8.7

+   **[postgresql] [bug]**

    现在在确定异常是否为“断开连接”错误时会咨询 psycopg2 的`.closed`访问器；理想情况下，这应该消除对异常消息的任何其他检查以检测断开连接的需要，但我们将保留这些现有消息作为备用。这应该能够处理新的情况，如“SSL EOF”条件。拉取请求由 Dirk Mueller 提供。

    参考：[#3021](https://www.sqlalchemy.org/trac/ticket/3021)

+   **[postgresql] [enhancement]**

    向 PostgreSQL 方言添加了一个新类型`OID`。虽然“oid”通常是 PG 内部的私有类型，在现代版本中不会公开，但在一些 PG 用例中（如大对象支持）可能会公开这些类型，以及在一些用户报告的模式反射用例中可能会公开。

    参考：[#3002](https://www.sqlalchemy.org/trac/ticket/3002)

### mysql

+   **[mysql] [bug]**

    修复了在索引的`mysql_length`参数上添加列名时，需要对带引号的名称使用相同的引号才能被识别的错误。修复使引号变为可选，但也为那些使用解决方法的人提供了旧的行为，以便与向后兼容。

    此更改也**回溯**到：0.8.7

    参考：[#3085](https://www.sqlalchemy.org/trac/ticket/3085)

+   **[mysql] [bug]**

    添加了支持，以使用等号包含 KEY_BLOCK_SIZE 的索引来反映表。Pull 请求由 Sean McGivern 提供。

    此更改也**回溯**到：0.8.7

### mssql

+   **[mssql] [bug]**

    修订了用于确定当前默认模式名称的查询，使用`database_principal_id()`函数与`sys.database_principals`视图结合使用，以便我们可以独立于正在进行的登录类型（例如，SQL Server，Windows 等）确定默认模式。 

    参考：[#3025](https://www.sqlalchemy.org/trac/ticket/3025)

### 测试

+   **[tests] [bug] [py3k]**

    修正了运行测试时涉及`imp`模块和 Python 3.3 或更高版本的一些弃用警告。Pull 请求由 Matt Chisholm 提供。

    参考：[#2830](https://www.sqlalchemy.org/trac/ticket/2830)

### 杂项

+   **[bug] [declarative]**

    当访问时，`__mapper_args__`字典从声明性 mixin 或抽象类中复制，以便声明性本身对此字典所做的修改不会与其他映射发生冲突。该字典在`version_id_col`和`polymorphic_on`参数方面进行修改，用本地类/表正式映射到的列替换其中的列。

    此更改也**回溯**到：0.8.7

    参考：[#3062](https://www.sqlalchemy.org/trac/ticket/3062)

+   **[bug] [ext]**

    修复了可变扩展中的错误，其中`MutableDict`未对`setdefault()`字典操作报告更改事件。

    此更改也**回溯**到：0.8.7

    参考：[#3051](https://www.sqlalchemy.org/trac/ticket/3051), [#3093](https://www.sqlalchemy.org/trac/ticket/3093)

+   **[bug] [ext]**

    修复了`MutableDict.setdefault()`未返回现有值或新值的错误（此错误未在任何 0.8 版本中发布）。Pull 请求由 Thomas Hervé提供。

    此更改也**回溯**到：0.8.7

    参考：[#3051](https://www.sqlalchemy.org/trac/ticket/3051), [#3093](https://www.sqlalchemy.org/trac/ticket/3093)

+   **[bug] [testsuite]**

    在公共测试套件中，从不太受支持的`Text`改为使用`String(40)`在`StringTest.test_literal_backslashes`中。Pullreq 由 Jan 提供。

+   **[bug] [firebird]**

    修复了一个错误，即在绑定参数（只有 firebird 同时具有“limit”渲染为“SELECT FIRST n ROWS”的情况）中“limit”与列级子查询组合的情况下，“limit”以及“位置”绑定参数（例如 qmark 样式），将错误地在外围 SELECT 之前分配子查询级别的位置，从而返回顺序不正确的参数。

    参考：[#3038](https://www.sqlalchemy.org/trac/ticket/3038)

## 0.9.4

发布日期：2014 年 3 月 28 日

### general

+   **[general] [feature]**

    添加了对 pytest 的支持以运行测试。该运行器当前在 nose 之外也得到了支持，并且将来可能会更倾向于使用 pytest。SQLAlchemy 使用的 nose 插件系统已拆分出来，以便它也可以在 pytest 下运行。目前没有计划放弃对 nose 的支持，我们希望测试套件本身可以继续保持尽可能与测试平台无关。有关使用 pytest 运行测试的更新信息，请参阅 README.unittests.rst 文件。

    还增强了测试插件系统，以支持一次针对多个数据库 URL 运行测试，方法是多次指定`--db`和/或`--dburi`标志。这不会对每个数据库运行整个测试套件，而是允许特定于某些后端的测试用例在运行测试时使用该后端。当将 pytest 作为测试运行器时，该系统还将多次运行特定的测试套件，每个数据库运行一次，特别是“dialect suite”中的那些测试。计划增强的系统还将被 Alembic 使用，并允许 Alembic 在一次运行中针对多个后端运行迁移操作测试，包括 Alembic 本身未包含的第三方后端。还鼓励第三方方言和扩展标准化为 SQLAlchemy 的测试套件作为基础；请参阅 README.dialects.rst 文件，了解从 SQLAlchemy 的测试平台构建的背景。

+   **[general] [bug]**

    调整了`setup.py`文件，以支持将来可能从 setuptools 中移除`setuptools.Feature`扩展。如果不存在此关键字，则设置仍将成功使用 setuptools 而不是回退到 distutils。现在还可以通过设置 DISABLE_SQLALCHEMY_CEXT 环境变量来禁用 C 扩展构建。无论 setuptools 是否可用，此变量都起作用。

    此更改还**被迁移到了**：0.8.6

    参考：[#2986](https://www.sqlalchemy.org/trac/ticket/2986)

+   **[general] [bug]**

    修复了一些在 Python 3.4 中发生的测试/功能失败，特别是用于包装“column default”可调用对象的逻辑不会对 Python 内置函数起作用。

    参考：[#2979](https://www.sqlalchemy.org/trac/ticket/2979)

### orm

+   **[orm] [feature]**

    添加了新参数`mapper.confirm_deleted_rows`。默认为 True，表示一系列 DELETE 语句应确认游标行数与应匹配的主键数量相匹配；这种行为在大多数情况下已被取消（除非使用 version_id），以支持自引用 ON DELETE CASCADE 的不寻常边缘情况；为了适应这一点，消息现在只是一个警告，而不是异常，并且可以使用该标志指示期望这种自引用级联删除的映射。另请参见[#2403](https://www.sqlalchemy.org/trac/ticket/2403)以了解原始更改的背景。

    参考：[#3007](https://www.sqlalchemy.org/trac/ticket/3007)

+   **[orm] [功能]**

    如果将`MapperEvents.before_configured()`或`MapperEvents.after_configured()`事件应用于特定的映射器或映射类，则会发出警告，因为这些事件仅对一般级别的`Mapper`目标调用。

+   **[orm] [功能]**

    添加了一个新的关键字参数`once=True`到`listen()`和`listens_for()`。这是一个方便的功能，它将包装给定的监听器，以便只调用一次。

+   **[orm] [功能]**

    添加了一个新选项到`relationship.innerjoin`，即指定字符串`"nested"`。当设置为`"nested"`而不是`True`时，连接的“链式”将在现有外连接的右侧括号内连接，而不是作为一系列外连接的字符串连接。当 0.9 发布时，这可能应该是默认行为，因为我们在 ORM 中引入了右嵌套连接的功能，但是为了避免进一步的意外，我们目前将其保留为非默认行为。

    另请参见

    右嵌套内连接可用于连接的急加载

    参考：[#2976](https://www.sqlalchemy.org/trac/ticket/2976)

+   **[orm] [错误]**

    修复了 ORM 中的一个 bug，即更改对象的主键，然后将其标记为 DELETE 将无法定位正确的 DELETE 行。

    此更改也**回溯**到：0.8.6

    参考：[#3006](https://www.sqlalchemy.org/trac/ticket/3006)

+   **[orm] [错误]**

    修复了从 0.8.3 回归的错误，作为结果[#2818](https://www.sqlalchemy.org/trac/ticket/2818)，其中`Query.exists()`不会在仅具有`Query.select_from()`条目但没有其他实体的查询上工作。

    这个更改也被**回溯**到：0.8.6

    参考：[#2995](https://www.sqlalchemy.org/trac/ticket/2995)

+   **[orm] [bug]**

    改进了一个错误消息，如果对非可选择性（例如`literal_column()`）进行了查询()，然后尝试使用`Query.join()`使得“左”侧被确定为`None`，然后失败。现在明确检测到这种情况。

    这个更改也被**回溯**到：0.8.6

+   **[orm] [bug]**

    从`sqlalchemy.orm.interfaces.__all__`中删除陈旧的名称，并用当前名称刷新，以便再次从该模块进行`import *`。

    这个更改也被**回溯**到：0.8.6

    参考：[#2975](https://www.sqlalchemy.org/trac/ticket/2975)

+   **[orm] [bug]**

    修复了一个非常古老的行为，其中对于一对多的惰性加载可能不当地拉入父表，并且基于父表中的内容返回不一致的结果，当 primaryjoin 包含针对父表的某种鉴别器时，例如`and_(parent.id == child.parent_id, parent.deleted == False)`。虽然这种 primaryjoin 对于一对多来说没有太多意义，但当应用于多对一的一侧时略微更常见，并且一对多作为 backref 的结果而出现。在这种情况下加载来自`child`的行将保持查询中的`parent.deleted == False`，从而将其拉入 FROM 子句并执行笛卡尔积。新的行为现在将适当地替换本地“parent.deleted”的值为该参数。尽管通常，真实世界的应用程序可能希望在任何情况下都为 o2m 侧使用不同的 primaryjoin。

    参考：[#2948](https://www.sqlalchemy.org/trac/ticket/2948)

+   **[orm] [bug]**

    改进了“如何从 A 连接到 B”的检查，以便当一个表具有多个，针对父表的复合外键时，`relationship.foreign_keys`参数将被正确解释以解决歧义; 以前，当事实上 foreign_keys 参数应该确定哪一个是预期的时，此条件将引发存在多个 FK 路径。

    参考：[#2965](https://www.sqlalchemy.org/trac/ticket/2965)

+   **[orm] [bug]**

    添加了对尚未完全记录的 `insert=True` 标志的支持，以便与 mapper / 实例事件一起使用 `listen()`。

+   **[ORM] [bug] [引擎]**

    修复了一个 bug，即设置在类级别监听事件（例如在 `Mapper` 或 `ClassManager` 等级上，而不是在单个映射类上，还有在 `Connection` 上，同时还使用了内部参数转换（在这些类别中大多数情况下）的事件将无法被移除。

    参考：[#2973](https://www.sqlalchemy.org/trac/ticket/2973)

+   **[ORM] [bug]**

    修复了从 0.8 版本中的一个回归 bug，即在使用类似 `lazyload()` 这样的选项与“通配符”表达式，例如 `"*"`，在查询中不包含任何实际实体时会引发断言错误的情况。此断言是为其他情况而设计的，却无意中捕捉到了这种情况。

+   **[ORM] [bug] [sqlite]**

    对 SQLite 的“连接重写”进行了更多修复；在 0.9.3 版本发布之前实施的修复影响了 UNION 中包含嵌套连接的情况。“连接重写”是一个具有广泛可能性的功能，并且是多年来我们引入的第一个复杂的“SQL 重写”功能，因此我们正在进行许多迭代（就像在 0.2/0.3 系列中的急加载，0.4/0.5 中的多态加载）。我们很快就会完成，感谢您的耐心等待 :).

    参考：[#2969](https://www.sqlalchemy.org/trac/ticket/2969)

### 示例

+   **[示例] [bug]**

    修复了版本化历史示例中的一个 bug，其中列级的 INSERT 默认值会阻止写入 NULL 的历史值。

### 引擎

+   **[引擎] [功能]**

    为方言级事件添加了一些新的事件机制；初始实现允许事件处理程序重新定义任意方言在 DBAPI 游标上调用 execute() 或 executemany() 的具体机制。这些新事件，目前是半公开和实验性的，是为了支持即将推出的一些与事务相关的扩展。

+   **[引擎] [功能]**

    现在可以将事件监听器与`Engine`关联，之后一个或多个`Connection`对象已创建（例如通过 orm `Session`或通过显式连接），监听器将从这些连接中接收事件。以前，性能问题导致事件传输从`Engine`仅在初始化时转移到`Connection`，但我们内联了一堆条件检查，使得这种操作在没有任何额外函数调用的情况下成为可能。

    参考：[#2978](https://www.sqlalchemy.org/trac/ticket/2978)

+   **[engine] [bug]**

    对`Engine`在检测到“断开”条件时重新使用连接池的机制进行了重大改进；现在不再丢弃池并显式关闭连接，而是保留池并更新“生成”时间戳以反映当前时间，从而导致在下次检出时重新使用所有现有连接。这极大地简化了回收过程，消除了等待旧池的“唤醒”连接尝试的需要，并消除了在回收操作期间可能创建许多立即丢弃的“池”对象的竞争条件。

    参考：[#2985](https://www.sqlalchemy.org/trac/ticket/2985)

+   **[engine] [bug]**

    `ConnectionEvents.after_cursor_execute()` 事件现在针对`Connection`的“_cursor_execute()”方法发出；这是用于诸如在 INSERT 语句之前执行序列等快速执行器，以及用于方言启动检查（如 unicode 返回、字符集等）的事件。`ConnectionEvents.before_cursor_execute()` 事件已在此处调用。此处“executemany”标志现在始终设置为 False，因为此事件始终对应于单个执行。以前，如果我们代表 executemany INSERT 语句执行操作，该标志可能为 True。

### sql

+   **[sql] [feature]**

    增加了对布尔值的文字呈现支持，例如“true” / “false”或“1” / “0”。

+   **[sql] [feature]**

    添加了一个新功能`conv()`，其目的是标记约束名称已经应用了命名约定。这个标记将在 Alembic 0.6.4 中被 Alembic 迁移使用，以便在迁移脚本中呈现已经标记为已经应用了命名约定的约束。

+   **[sql] [feature]**

    为了帮助依赖于向构造添加临时关键字参数的现有方案，已经增强了模式级构造的新方言级关键字参数系统。

    例如，类似`Index`这样的构造在构建后将再次接受`Index.kwargs`集合中的临时关键字参数：

    ```py
    idx = Index("a", "b")
    idx.kwargs["mysql_someargument"] = True
    ```

    为了适应允许在构造时使用自定义参数的用例，`DialectKWArgs.argument_for()`方法现在允许这种注册：

    ```py
    Index.argument_for("mysql", "someargument", False)

    idx = Index("a", "b", mysql_someargument=True)
    ```

    另请参阅

    `DialectKWArgs.argument_for()`

    参考：[#2866](https://www.sqlalchemy.org/trac/ticket/2866)，[#2962](https://www.sqlalchemy.org/trac/ticket/2962)

+   **[sql] [bug]**

    修复了`tuple_()`构造中的错误，其中实质上第一个 SQL 表达式的“类型”会被应用为与比较的元组值的“比较类型”；在某些情况下，这会导致不恰当的“类型强制转换”发生，例如当一个包含字符串和二进制值混合的元组错误地将目标值强制转换为二进制，即使左侧并不是二进制。`tuple_()`现在期望其值列表中存在异构类型。

    这个更改也被**回溯**到：0.8.6

    参考：[#2977](https://www.sqlalchemy.org/trac/ticket/2977)

+   **[sql] [bug]**

    修复了 0.9 中的一个回归问题，即未能正确反映的`Table`不会从父`MetaData`中移除，即使处于无效状态。感谢 Roman Podoliaka 的 Pullreq。

    参考：[#2988](https://www.sqlalchemy.org/trac/ticket/2988)

+   **[sql] [bug]**

    `MetaData.naming_convention` 功能现在也将应用于直接与`Column`关联的`CheckConstraint`对象，而不仅仅是`Table`上的。

+   **[sql] [bug]**

    修复了新的`MetaData.naming_convention`功能中的错误，其中使用“%(constraint_name)s”标记的检查约束的名称会在布尔或枚举类型生成的约束中重复，而整体重复事件会导致“%(constraint_name)s”标记不断累积。

    参考：[#2991](https://www.sqlalchemy.org/trac/ticket/2991)

+   **[sql] [bug]**

    调整了将名称应用于 .c 集合时的逻辑，当接收到无名称的`BindParameter`时，例如通过`literal()`或类似方式；绑定参数的“key”被用作 .c 中的键，而不是渲染名称。由于这些绑定在任何情况下都具有“匿名”名称，这允许单独的绑定参数在可选择的情况下具有自己的名称，如果它们没有被标记。

    参考：[#2974](https://www.sqlalchemy.org/trac/ticket/2974)

+   **[sql] [bug]**

    对`FromClause.c`集合的行为进行了一些更改，当遇到重复的列时。发出警告并替换具有相同名称的旧列的行为仍然在某种程度上保留；特别是替换是为了保持向后兼容性。但是，替换的列现在仍然与`c`集合关联在一起，现在在集合`._all_columns`中（用于处理诸如别名和联合之类的结构中的`c`列集合中的列集合），以更多地处理`c`中的列列表而不是唯一的键名称集合。这有助于处理在联合等情况下使用具有相同命名列的 SELECT 语句的情况，以便联合可以按位置匹配列，并且在这里仍然有一些机会使用`FromClause.corresponding_column()`（它现在可以返回仅在 selectable.c._all_columns 中而不以其他方式命名的列）。新集合是有下划线的，因为我们仍然需要决定这个列表最终可能会到哪里。从理论上讲，它将成为 iter(selectable.c)的结果，但是这意味着迭代的长度将不再与 keys()的长度匹配，需要检查该行为。

    参考：[#2974](https://www.sqlalchemy.org/trac/ticket/2974)

+   **[sql] [bug]**

    修复了新的`TextClause.columns()`方法中的问题，其中给定位置的列的顺序不会被保留。这可能会在诸如将生成的`TextAsFrom`对象应用于联合等位置情况下产生潜在影响。

### postgresql

+   **[postgresql] [feature]**

    为 psycopg2 DBAPI 启用了“理智的多行计数”检查，因为截至 psycopg2 2.0.9 似乎支持这一点。

    此更改还被**回溯到**：0.8.6

+   **[postgresql] [bug]**

    由版本 0.8.5 / 0.9.3 的兼容性增强引起的回归错误已修复，在只针对 8.1、8.2 系列的 PostgreSQL 版本上反射索引时再次出现了问题，涉及到永远问题的 int2vector 类型。虽然 int2vector 从 8.1 开始支持数组操作，但显然它从 8.3 开始只支持 CAST 到 varchar。

    此更改还被**回溯到**：0.8.6

    参考：[#3000](https://www.sqlalchemy.org/trac/ticket/3000)

### mysql

+   **[mysql] [bug]**

    调整了 mysql-connector-python 的设置；在 Py2K 中，“supports unicode statements”标志现在为 False，因此 SQLAlchemy 将在发送到数据库之前将*SQL 字符串*（注意：*不是*参数）编码为字节。这似乎允许 mysql-connector 的所有与 Unicode 相关的测试通过，包括那些使用非 ASCII 表/列名称以及使用 unicode 在 cursor.executemany()下的 TEXT 类型的一些测试。

### oracle

+   **[oracle] [feature]**

    在 cx_Oracle 方言中添加了一个新的引擎选项 `coerce_to_unicode=True`，这恢复了在 Python 2 中 cx_Oracle 输出类型处理方法到 Python Unicode 转换的方式，该方法在 0.9.2 中因 [#2911](https://www.sqlalchemy.org/trac/ticket/2911) 被移除。一些使用情况可能会希望对所有字符串值进行 unicode 强制转换，尽管存在性能问题。补丁由 Christoph Zwerschke 提供。

    引用：[#2911](https://www.sqlalchemy.org/trac/ticket/2911)

+   **[oracle] [bug]**

    添加了新的数据类型 `DATE`，它是 `DateTime` 的子类。由于 Oracle 没有“datetime”类型，而只有 `DATE`，因此在 Oracle 方言中，`DATE` 类型作为 `DateTime` 的实例是合适的。这个问题并不改变类型的行为，因为数据转换无论如何都是由 DBAPI 处理的，但改进的子类布局将有助于跨数据库兼容性的检查类型的使用情况。此外，从 Oracle 方言中删除了大写的 `DATETIME`，因为这种类型在该上下文中无法使用。

    引用：[#2987](https://www.sqlalchemy.org/trac/ticket/2987)

### 测试

+   **[测试] [bug]**

    修复了一些在 Py3.2 中会导致测试失败的 errant `u''` 字符串。补丁由 Arfrever Frehtes Taifersar Arahesis 提供。

    引用：[#2980](https://www.sqlalchemy.org/trac/ticket/2980)

### 杂项

+   **[bug] [ext]**

    修复了 mutable extension 和 `flag_modified()` 中的一个 bug，即如果属性被重新分配为自身，则更改事件将不会传播。

    这个变更也被**回溯**到：0.8.6

    引用：[#2997](https://www.sqlalchemy.org/trac/ticket/2997)

+   **[bug] [automap] [ext]**

    为 automap 添加了对两个处于联合继承关系的类之间不应创建关系的支持，用于将子类链接回父类的那些外键。

    引用：[#3004](https://www.sqlalchemy.org/trac/ticket/3004)

+   **[bug] [池]**

    修复了 `SingletonThreadPool` 中的一个小问题，即在“清理”过程中可能会无意间清除要返回的当前连接。补丁由 jd23 提供。

+   **[bug] [扩展] [py3k]**

    修复了关联代理中的一个 bug，在 Py3k 上，分配一个空切片（例如 `x[:] = [...]`）将会失败。

+   **[bug] [ext]**

    修复了关联代理中由[#2810](https://www.sqlalchemy.org/trac/ticket/2810)引起的回归问题，该问题导致当从不存在的目标中获取标量值时，用户提供的“getter”不再接收`None`值。此更改引入的 None 检查现在已移至默认 getter 中，因此用户提供的 getter 也将再次接收到 None 值。

    参考：[#2810](https://www.sqlalchemy.org/trac/ticket/2810)

## 0.9.3

发布日期：2014 年 2 月 19 日

### orm

+   **[orm] [feature]**

    添加了新的`MapperEvents.before_configured()`事件，允许在`configure_mappers()`开始时触发事件，以及在声明式中配合`__declare_last__()`的`__declare_first__()`钩子。

+   **[orm] [bug]**

    修复了当`Query.get()`在具有现有条件的查询上调用时，如果给定的标识已经存在于标识映射中，则无法始终引发`InvalidRequestError`的错误的 bug。

    这个变更也被**回移**到了：0.8.5

    参考：[#2951](https://www.sqlalchemy.org/trac/ticket/2951)

+   **[orm] [bug] [sqlite]**

    修复了 SQLite “join rewriting” 中的一个 bug，即使用 exists() 构造时无法正确重写的问题，例如，当 exists 被映射到一个复杂的嵌套连接场景中的 column_property 时。还修复了一个有些相关的问题，即当目标是别名表而不是单独的别名列时，如果 SELECT 语句的 columns 子句的目标是别名表，则连接重写将在失败。

    参考：[#2967](https://www.sqlalchemy.org/trac/ticket/2967)

+   **[orm] [bug]**

    修复了一个 0.9 版本的回归问题，即应用于基类（例如使用了传播标志的声明式基类）的 ORM 实例或映射器事件将无法应用于还使用了继承的现有映射类的情况，因为存在一个断言。另外，修复了在移除这种事件时可能出现的属性错误，具体取决于首次分配的方式。

    参考：[#2949](https://www.sqlalchemy.org/trac/ticket/2949)

+   **[orm] [bug]**

    改进了复合属性的初始化逻辑，使得调用`MyClass.attribute`不再需要配置映射器步骤，例如，它将在不抛出任何错误的情况下正常工作。

    参考：[#2935](https://www.sqlalchemy.org/trac/ticket/2935)

+   **[orm] [bug]**

    更多关于[ticket:2932]的问题首次在 0.9.2 中解决，其中使用形式为`<tablename>_<columnname>`的列键与文本中的别名列仍然不匹配，这最终是由于核心列匹配问题。已添加额外规则，以便在使用`TextAsFrom`构造或文字列时考虑列`_label`。

    参考：[#2932](https://www.sqlalchemy.org/trac/ticket/2932)

### orm 声明性

+   **[orm] [declarative] [bug]**

    修复了一个 bug，`AbstractConcreteBase`在声明性关系配置中无法完全使用，因为其字符串类名在映射器配置时不可用。该类现在明确将自己添加到类注册表中，并且`AbstractConcreteBase`以及`ConcreteBase`在映射器配置之前*之前*设置自己，在`configure_mappers()`设置中，使用新的`MapperEvents.before_configured()`事件。

    参考：[#2950](https://www.sqlalchemy.org/trac/ticket/2950)

### examples

+   **[examples] [feature]**

    在版本化行示例中添加了可选的“changed”列，以及当版本化的`Table`具有显式的`Table.schema`参数时的支持。感谢 jplaverdure 的拉取请求。

### engine

+   **[engine] [bug] [pool]**

    修复了由[#2880](https://www.sqlalchemy.org/trac/ticket/2880)引起的关键回归，其中新的并发能力从池中返回连接意味着“first_connect”事件现在也不再同步，从而在即使是最小并发情况下也导致方言配置错误。

    此更改也**回溯**到：0.8.5

    参考：[#2880](https://www.sqlalchemy.org/trac/ticket/2880)，[#2964](https://www.sqlalchemy.org/trac/ticket/2964)

### sql

+   **[sql] [bug]**

    修复了调用`Insert.values()`时使用空列表或元组会引发 IndexError 的 bug。现在它会产生一个空的插入构造，就像使用空字典的情况一样。

    此更改也**回溯**到：0.8.5

    参考：[#2944](https://www.sqlalchemy.org/trac/ticket/2944)

+   **[sql] [bug]**

    修复了如果错误地传递了包含`__getitem __()`方法的列表达式的比较器的`ColumnOperators.in_()`会进入无限循环的错误，例如使用`ARRAY`类型的列。

    此更改也已经**反向移植**到：0.8.5

    参考：[#2957](https://www.sqlalchemy.org/trac/ticket/2957)

+   **[sql] [bug]**

    修复了新“命名约定”功能中的回归，如果外键中的引用表包含模式名称，则约定会失败。Pull request 来自 Thomas Farvour。

+   **[sql] [bug]**

    修复了所谓的“字面渲染”`bindparam()`构造的 bug，如果绑定是用可调用的方式构造而不是直接值，则会失败。这阻止了使用“literal_binds”编译器标志渲染 ORM 表达式。

### postgresql

+   **[postgresql] [feature]**

    在`ARRAY`类型上添加了`TypeEngine.python_type`的便利访问器。Pull request 来自 Alexey Terentev。

+   **[postgresql] [bug]**

    向 psycopg2 断开连接检测添加了一个额外的消息，“无法将数据发送到服务器”，这补充了现有的“无法从服务器接收数据”，并已被用户观察到。

    此更改也已经**反向移植**到：0.8.5

    参考：[#2936](https://www.sqlalchemy.org/trac/ticket/2936)

+   **[postgresql] [bug]**

    > 对于非常旧的（8.1 之前）版本的 PostgreSQL 以及可能的其他 PG 引擎（假设 Redshift 报告版本小于 8.1），已改进了对 PostgreSQL 反射行为的支持。 “索引”和“主键”的查询依赖于检查所谓的“int2vector”数据类型，该数据类型在 8.1 之前的版本中拒绝转换为数组，导致查询中使用的“ANY（）”运算符失败。通过广泛的搜索找到了非常巧妙但由 PG 核心开发人员推荐使用的查询，用于在使用 PG 版本<8.1 时，索引和主键约束反射现在可以在这些版本上工作。

    此更改也已经**反向移植**到：0.8.5

+   **[postgresql] [bug]**

    修正了这个非常老的问题，其中 PostgreSQL 的“获取主键”反射查询已更新以考虑已重命名的主键约束；新的查询在旧版本的 PostgreSQL 上失败，如版本 7，因此在检测到 server_version_info <(8, 0)的情况下，在这些情况下恢复旧的查询。

    此更改也已经**反向移植**到：0.8.5

    参考：[#2291](https://www.sqlalchemy.org/trac/ticket/2291)

+   **[postgresql] [错误]**

    在新添加的方言启动查询中添加了服务器版本检测，用于“show standard_conforming_strings”; 由于此变量是从 PG 8.2 开始添加的，我们会跳过对报告早于该版本的 PG 版本的查询。

    参考：[#2946](https://www.sqlalchemy.org/trac/ticket/2946)

### mysql

+   **[mysql] [功能]**

    添加了新的 MySQL-specific `DATETIME`，其中包括分数秒支持；还为 `TIMESTAMP` 添加了分数秒支持。 DBAPI 支持有限，尽管 MySQL Connector/Python 已知支持分数秒。 补丁由 Geert JM Vanderkelen 提供。

    此更改也**回溯**到：0.8.5

    参考：[#2941](https://www.sqlalchemy.org/trac/ticket/2941)

+   **[mysql] [错误]**

    添加了对 `PARTITION BY` 和 `PARTITIONS` MySQL 表关键字的支持，指定为 `mysql_partition_by='value'` 和 `mysql_partitions='value'` 到 `Table`。 拉取请求由 Marcus McCurdy 提供。

    此更改也**回溯**到：0.8.5

    参考：[#2966](https://www.sqlalchemy.org/trac/ticket/2966)

+   **[mysql] [错误]**

    修复了一个 bug，该 bug 阻止了基于 MySQLdb 的方言（例如 pymysql）在 Py3K 中工作，因为“connection charset”的检查会由于 Py3K 更严格的值比较规则而失败。 在任何情况下，该调用都没有考虑数据库版本，因为服务器版本在那时仍然为 None，因此该方法已经简化为依赖于 connection.character_set_name()。

    此更改也**回溯**到：0.8.5

    参考：[#2933](https://www.sqlalchemy.org/trac/ticket/2933)

+   **[mysql] [错误] [cymysql]**

    修复了 cymysql 方言中的一个 bug，其中类似 `'33a-MariaDB'` 的版本字符串无法正确解析。 拉取请求由 Matt Schmidt 提供。

    参考：[#2934](https://www.sqlalchemy.org/trac/ticket/2934)

### sqlite

+   **[sqlite] [错误]**

    SQLite 方言现在在反射类型时会跳过不支持的参数；例如，如果遇到类似 `INTEGER(5)` 的字符串，将实例化 `INTEGER` 类型时不包括“5”，基于在第一次尝试时检测到 `TypeError`。

+   **[sqlite] [错误]**

    已向 SQLite 类型反射添加了对完全支持“类型亲和性”契约的支持，该契约在 [`www.sqlite.org/datatype3.html`](https://www.sqlite.org/datatype3.html) 中指定。 在此方案中，类型名称中的关键字如 `INT`、`CHAR`、`BLOB` 或 `REAL` 通常将类型与五种亲和性之一关联起来。 拉取请求由 Erich Blume 提供。

    另请参阅

    类型反射

### 杂项

+   **[bug] [ext]**

    修复了当类在单个或潜在的联合继承模式中预先排列时，新的自动映射扩展的 `AutomapBase` 类会失败的 bug。修复的联合继承问题也可能适用于使用 `DeferredReflection` 时。

## 0.9.2

发布日期：2014 年 2 月 2 日

### orm

+   **[orm] [feature]**

    添加了一个新的参数 `Operators.op.is_comparison`。此标志允许将来自 `Operators.op()` 的自定义操作视为“比较”操作符，因此可用于自定义 `relationship.primaryjoin` 条件。

    另见

    在连接条件中使用自定义操作符

+   **[orm] [feature]**

    对于以 `join()` 构造作为 `relationship.secondary` 目标以创建非常复杂的 `relationship()` 连接条件的目的，支持得到了改善。此更改包括查询连接、连接的及时加载以不生成 SELECT 子查询、延迟加载的更改以使“secondary”目标正确包含在 SELECT 中，以及声明性的更改以更好地支持将 join() 对象与类作为目标的指定。

    新的用例有些实验性，但已添加了一个新的文档部分。

    另见

    组合“Secondary”连接

+   **[orm] [bug]**

    修复了当将迭代器对象传递给 `class_mapper()` 或类似函数时的错误消息，其中错误会在字符串格式化时失败的 bug。来自 Kyle Stark 的 Pullreq。

    此更改还被 **回溯** 到：0.8.5

+   **[orm] [bug]**

    修复了新的`TextAsFrom`构造中的错误，在此构造中，`Column`-定向行查找未能与`TextAsFrom`生成的临时`ColumnClause`对象匹配，因此不能在`Query.from_statement()`中使用。还修复了`Query.from_statement()`的机制，以免误将`TextAsFrom`误认为是`Select`构造。该错误也是 0.9 的退化，因为调用了`TextClause.columns()`方法以适应`text.typemap`参数。

    参考：[#2932](https://www.sqlalchemy.org/trac/ticket/2932)

+   **[orm] [bug]**

    在属性“set”操作的范围内添加了一个新的指令，用于禁用自动刷新，在属性需要延迟加载“旧”值的情况下使用，例如在替换一对一值或某些类型的一对多值时。否则，在属性为 None 时会发生刷新，并可能导致 NULL 违规。

    参考：[#2921](https://www.sqlalchemy.org/trac/ticket/2921)

+   **[orm] [bug]**

    修复了一个 0.9 退化，即当由`Query`自动应用别名时，以及在其他情况下选择或连接被别名化时（例如联接表继承），如果在表达式中使用了用户定义的`Column`子类，则可能失败。在这种情况下，子类将无法传播 ORM 特定的“注解”，而这些注解对适应是必需的。已根据这种情况纠正了“表达式注解”系统。

    参考：[#2918](https://www.sqlalchemy.org/trac/ticket/2918)

+   **[orm] [bug]**

    修复了一个 bug，涉及与新的扁平化 JOIN 结构一起使用的`joinedload()`（从而导致连接式贪婪加载的回归）以及与`flat=True`标志和连接表继承一起使用的`aliased()`；基本上，通过不同路径跨越“父 JOIN 子”实体进行多个连接以到达目标类的情况下，不会形成正确的 ON 条件。 在处理一个别名的，连接继承类的“左侧”加入的机制中进行的调整/简化修复了该问题。

    参考: [#2908](https://www.sqlalchemy.org/trac/ticket/2908)

### 示例

+   **[示例] [错误修复]**

    在“history_meta”示例中添加了一个调整，其中对于关系绑定属性上的“history”检查如果关系未加载，则不再发出任何 SQL。

### 引擎

+   **[引擎] [功能] [池]**

    添加了一个新的池事件`PoolEvents.invalidate()`。 当要将 DBAPI 连接标记为“无效”并从池中丢弃时调用。

### sql

+   **[sql] [功能]**

    添加了 `MetaData.reflect.dialect_kwargs` 以支持对所有`Table`对象进行反射的方言级反射选项。

+   **[sql] [功能]**

    添加了一个新功能，允许将自动命名约定应用于`Constraint`和`Index`对象。 基于 wiki 中的一个配方，新功能使用模式事件来设置各个模式对象关联时的名称。 然后，事件通过一个新的参数`MetaData.naming_convention`暴露了一个配置系统。 该系统允许在每个`MetaData`基础上生成简单和自定义的约束和索引命名方案。

    另请参阅

    配置约束命名约定

    参考: [#2923](https://www.sqlalchemy.org/trac/ticket/2923)

+   **[sql] [功能]**

    现在可以在不涉及表中的列规范的情况下独立于`primary_key=True`标志指定`PrimaryKeyConstraint`对象上的选项；使用一个不包含列的`PrimaryKeyConstraint`对象来实现此结果。

    之前，显式的`PrimaryKeyConstraint`会导致那些标记为`primary_key=True`的列被忽略；由于现在不再是这样，`PrimaryKeyConstraint`现在将确保使用一种风格或另一种风格来指定列，或者如果两者都存在，则列列表完全匹配。如果在`PrimaryKeyConstraint`中存在不一致的列集，并且在`Table`内标记为`primary_key=True`的列，则会发出警告，并且列的列表仅来自先前发布中的`PrimaryKeyConstraint`本身，就像以前的情况一样。

    另请参阅

    `PrimaryKeyConstraint`

    参考：[#2910](https://www.sqlalchemy.org/trac/ticket/2910)

+   **[sql] [特性]**

    架构构造和某些 SQL 构造接受方言特定关键字参数的系统已得到加强。该系统通常包括`Table`和`Index`构造，它们接受各种各样的方言特定参数，如`mysql_engine`和`postgresql_where`，以及构造`PrimaryKeyConstraint`、`UniqueConstraint`、`Update`、`Insert` 和 `Delete`，并且还新增了对`ForeignKeyConstraint`和`ForeignKey`的关键字参数能力。变化在于参与的方言现在可以指定这些构造的可接受参数列表，允许在为特定方言指定无效关键字时引发参数错误。如果关键字的方言部分未被识别，仅发出警告；虽然系统实际上将使用 setuptools 入口点来定位非本地方言，但在某些方言特定参数在未安装第三方方言的环境中使用的用例仍然受支持。方言还必须显式地选择加入此系统，以便未使用此系统的外部方言不受影响。

    参考：[#2866](https://www.sqlalchemy.org/trac/ticket/2866)

+   **[sql] [bug]**

    `Table.tometadata()` 方法的行为已经调整，使得`ForeignKey` 的模式目标不会改变，除非该模式与父表的模式匹配。也就是说，如果一个表“schema_a.user”有一个指向“schema_b.order.id”的外键，无论是否将“schema”参数传递给`Table.tometadata()`，都将保持“schema_b”目标不变。然而，如果一个表“schema_a.user”引用“schema_a.order.id”，则“schema_a” 的存在将在父表和被引用的表上更新。这是一个行为变更，因此不太可能被回溯到 0.8 版本；假设以前的行为相当错误，而且很少有人依赖它。

    另外，新增了一个参数`Table.tometadata.referred_schema_fn`。这是指一个可调用函数，用于确定在 tometadata 操作中遇到的任何`ForeignKeyConstraint` 的新被引用模式。这个可调用函数可以用于恢复到以前的行为或自定义如何处理每个约束的被引用模式。

    参考：[#2913](https://www.sqlalchemy.org/trac/ticket/2913)

+   **[sql] [bug]**

    修复了一个 bug，即在某些情况下，如果与“test”方言一起使用二进制类型，例如 DefaultDialect 或其他没有 DBAPI 的方言，二进制类型会失败。

+   **[sql] [bug] [py3k]**

    修复了“literal binds”无法与绑定参数为二进制类型的 bug。0.8 版本中修复了一个类似但不同的问题。

+   **[sql] [bug]**

    修复了回归问题，即 ORM 使用的“注释”系统泄漏到`sqlalchemy.sql.functions` 中标准函数的名称中，例如`func.coalesce()` 和 `func.max()`。在 ORM 属性中使用这些函数，从而生成它们的带注释版本，可能会破坏在 SQL 中呈现的实际函数名称。

    参考：[#2927](https://www.sqlalchemy.org/trac/ticket/2927)

+   **[sql] [bug]**

    修复了 0.9 版本中的回归问题，即`RowProxy` 的新可排序支持在与非元组类型进行比较时会导致`TypeError`，因为它试图无条件地应用 tuple()到“other”对象。现在在`RowProxy` 上实现了完整的 Python 比较运算符范围，使用一种保证等效于元组的比较系统的方法，只有在“other”对象是 RowProxy 的实例时才会强制转换该对象。

    参考：[#2848](https://www.sqlalchemy.org/trac/ticket/2848), [#2924](https://www.sqlalchemy.org/trac/ticket/2924)

+   **[sql] [bug]**

    在没有列的 `Table` 内联创建的 `UniqueConstraint` 将被跳过。感谢 Derek Harland 提供了 Pullreq。

+   **[sql] [bug] [orm]**

    修复了多表“UPDATE..FROM”构造，在 MySQL 上可用，以正确渲染 SET 子句中跨表具有相同名称的多列。这还更改了仅针对非主表的 SET 子句中使用的绑定参数的名称为“<tablename>_<colname>”；由于此参数通常是直接使用 `Column` 对象指定的，这不应对应用程序产生影响。该修复对 ORM 中的 `Table.update()` 和 `Query.update()` 均生效。

    引用：[#2912](https://www.sqlalchemy.org/trac/ticket/2912)

### schema

+   **[schema] [bug]**

    将 `sqlalchemy.schema.SchemaVisitor` 恢复到 `.schema` 模块中。感谢 Sean Dague 提供了 Pullreq。

### postgresql

+   **[postgresql] [feature]**

    添加了一个新的方言级参数 `postgresql_ignore_search_path`；此参数被 `Table` 构造函数和 `MetaData.reflect()` 方法接受。在对 PostgreSQL 使用时，指定了远程模式名称的外键引用表将保留该模式名称，即使该名称存在于 `search_path` 中；自 0.7.3 版以来的默认行为是，`search_path` 中存在的模式不会复制到反射的 `ForeignKey` 对象中。文档已更新以详细描述 `pg_get_constraintdef()` 函数的行为，以及 `postgresql_ignore_search_path` 功能如何确定我们是否将遵守此函数报告的模式限定。

    另请参阅

    远程模式表内省和 PostgreSQL search_path

    引用：[#2922](https://www.sqlalchemy.org/trac/ticket/2922)

### mysql

+   **[mysql] [bug]**

    为 cymysql 方言添加了一些缺失的方法，包括 _get_server_version_info() 和 _detect_charset()。感谢 Hajime Nakagami 提供了 Pullreq。

    此更改也被**回溯**到：0.8.5

+   **[mysql] [bug] [sql]**

    为所谓的 SQL 类型“向下适应”添加了新的测试覆盖，其中更具体的类型被适应为更通用的类型 - 这种用例被一些第三方工具（如`sqlacodegen`）所需。在此测试套件中需要修复的特定情况包括将`ENUM`降级为`Enum`，以及将 SQLite 日期类型转换为通用日期类型。`adapt()`方法在这里需要变得更具体，以抵消在 0.9 中删除的基本`TypeEngine`类上的“捕获所有”`**kwargs`集合的移除。

    参考：[#2917](https://www.sqlalchemy.org/trac/ticket/2917)

+   **[mysql] [bug]**

    现在 MySQL CAST 编译考虑了字符串类型的“字符集”和“排序”。虽然 MySQL 希望所有基于字符的 CAST 调用都使用 CHAR 类型，但我们现在在 CAST 时创建一个真正的 CHAR 对象，并复制它具有的所有参数，因此像`cast(x, mysql.TEXT(charset='utf8'))`这样的表达式将呈现为`CAST(t.col AS CHAR CHARACTER SET utf8)`。

+   **[mysql] [bug]**

    对 MySQL 方言和默认方言系统整体添加了新的“unicode 返回”检测，以便任何方言都可以在首次连接时添加额外的“测试”来检测 DBAPI 是否直接返回 unicode。在这种情况下，我们特别针对“utf8”编码添加了一个检查，使用显式的“utf8_bin”排序类型（在检查此排序是否可用后），以测试观察到的 MySQLdb 版本 1.2.3 存在的一些错误的 unicode 行为。虽然 MySQLdb 在 1.2.4 中已解决了此问题，但此处的检查应该防止出现退化。此更改还允许“unicode”检查记录在引擎日志中，这以前是不可能的。

    参考：[#2906](https://www.sqlalchemy.org/trac/ticket/2906)

+   **[mysql] [bug] [engine] [pool]**

    `Connection` 现在将一个新的 `RootTransaction` 或 `TwoPhaseTransaction` 与其直接的 `_ConnectionFairy` 关联为该事务的“重置处理程序”，在该事务的范围内接管调用 commit() 或 rollback() 以实现“返回时重置”的行为，如果事务未被完成。这解决了当连接在没有显式回滚或提交的情况下关闭时，像 MySQL 两阶段那样挑剔的事务将被正确关闭的问题（例如，在此情况下不再引发“XAER_RMFAIL” - 请注意，此异常仅在日志中显示，因为异常未在池重置中传播）。例如，当使用设置了 `twophase` 的 orm `Session`，然后在没有显式回滚或提交的情况下调用 `Session.close()` 时，将出现此问题。此更改还具有效果，即在非自动提交模式下使用 `Session` 对象时，无论如何丢弃该会话，您现在都将在日志中看到明确的“ROLLBACK”。感谢 Jeff Dairiki 和 Laurence Rowe 在此处隔离问题。

    参考：[#2907](https://www.sqlalchemy.org/trac/ticket/2907)

### sqlite

+   **[sqlite] [bug]**

    修复了 SQLite 编译器未能将编译器参数（如“literal binds”）传播到 CAST 表达式中的 bug。

### mssql

+   **[mssql] [feature]**

    在`UniqueConstraint`和`PrimaryKeyConstraint`构造中添加了一个选项`mssql_clustered`；在 SQL Server 上，这将在 DDL 中的约束构造中添加`CLUSTERED`关键字。感谢 Derek Harland 提交的 Pullreq。

### oracle

+   **[oracle] [bug]**

    已经观察到，在 Python 2.xx 中使用 cx_Oracle 的“outputtypehandler”来将字符串值强制转换为 Unicode 是非常昂贵的；即使 cx_Oracle 是用 C 编写的，当你将 Python `unicode` 原语传递给 cursor.var() 并与输出处理程序关联时，该库会将每次转换都计为 Python 函数调用，并记录所有必要的开销；尽管在 Python 3 中运行时，所有字符串也都被无条件地转换为 Unicode，但它不会产生这种开销，这意味着 cx_Oracle 在 Py2K 中未能使用高效的技术。由于 SQLAlchemy 不能轻松地针对每列选择此类类型处理程序，因此处理程序被无条件地组装，从而将开销添加到所有字符串访问中。

    因此，这个逻辑已经被 SQLAlchemy 自己的 Unicode 转换系统取代，在 Py2K 中仅对请求为 Unicode 的列生效。当使用 C 扩展时，SQLAlchemy 的系统似乎比 cx_Oracle 的系统快 2-3 倍。此外，SQLAlchemy 的 Unicode 转换已经得到增强，以便在需要“条件”转换器（现在在 Oracle 后端需要）时，在 C 中执行“已经是 Unicode”的检查，不再引入显着的开销。

    这个变化对 cx_Oracle 后端有两个影响。一个是在 Py2K 中，未明确请求 Unicode 类型或 convert_unicode=True 的字符串值现在将返回`str`，而不是`unicode` - 这种行为类似于诸如 MySQL 的后端。此外，当使用 cx_Oracle 后端请求 Unicode 值时，如果未使用 C 扩展，现在每列都有一个 isinstance() 检查的额外开销。这种权衡已经被做出，因为可以解决它，并且不再对可能是非 Unicode 字符串的大多数 Oracle 结果列施加性能负担。

    参考：[#2911](https://www.sqlalchemy.org/trac/ticket/2911)

### misc

+   **[bug] [pool]**

    `PoolEvents.reset()` 事件的参数名已更改为 `dbapi_connection` 和 `connection_record`，以保持与所有其他池事件的一致性。预计任何现有的侦听器都在任何情况下都使用位置样式来接收参数。

+   **[bug] [cextensions] [py3k]**

    修复了在 Py3K 中 C 扩展使用错误的 API 来指定顶级模块函数的问题，这在 Python 3.4b2 中会出错。Py3.4b2 将 PyMODINIT_FUNC 更改为返回“void”而不是`PyObject *`，因此现在我们确保直接使用“PyMODINIT_FUNC”而不是`PyObject *`。拉取请求由 cgohlke 提供。

## 0.9.1

发布日期：2014 年 1 月 5 日

### orm

+   **[orm] [bug] [events]**

    修复了一个回归问题，使用`functools.partial()`与事件系统会因为在其上使用 inspect.getargspec()来检测某些事件的传统调用签名而导致递归溢出，显然无法在部分对象上执行此操作。我们现在跳过传统检查，假设采用现代风格；现在检查仅在 SessionEvents.after_bulk_update 和 SessionEvents.after_bulk_delete 事件中发生。如果分配给“partial”事件侦听器，则这两个事件将需要新的签名样式。

    参考：[#2905](https://www.sqlalchemy.org/trac/ticket/2905)

+   **[orm] [bug]**

    修复了一个 bug，使用新的`Session.info`属性会失败，如果`.info`参数仅传递给`sessionmaker`创建调用，而不传递给对象本身。感谢 Robin Schoonover。

+   **[orm] [bug]**

    修复了一个回归问题，当根据名称设置基于名称的 backref 时，我们不检查给定名称与正确的字符串类是否匹配，因此导致错误“too many values to unpack”。这与 Py3k 转换有关。

    参考：[#2901](https://www.sqlalchemy.org/trac/ticket/2901)

+   **[orm] [bug]**

    修复了一个回归问题，当说 query(B).join(B.cs)时，我们显然仍然创建一个隐式别名，其中“C”是一个连接的继承类；然而，这个隐式别名仅考虑了直接左侧，而没有考虑同一基类的不同连接继承子类的更长链的连接。只要我们在这种情况下仍然隐式别名，行为就会有所减弱，以便在更广泛的情况下别名右侧。

    参考：[#2903](https://www.sqlalchemy.org/trac/ticket/2903)

### orm 声明

+   **[orm] [declarative] [bug]**

    修复了一个极不可能的内存问题，当使用`DeferredReflection`定义待反射的类时，如果在调用`DeferredReflection.prepare()`方法之前丢弃了其中一些类的子集，那么在映射和映射类时，将保留对类的强引用。这个内部的“要映射的类”集合现在使用对类本身的弱引用。

+   **[orm] [declarative] [bug]**

    在 0.8 版本中，似乎可以在声明性上设置一个类级属性，直接引用超类或类本身的`InstrumentedAttribute`，它更多地充当同义词；在 0.9 版本中，这种设置未能设置足够的记录来跟上更自由化的反向引用逻辑，来源于[#2789](https://www.sqlalchemy.org/trac/ticket/2789)。即使这种用例从未直接考虑过，现在在“setattr()”级别以及设置子类时，声明性也会检测到，并且镜像/重命名属性现在设置为`synonym()`。

    参考：[#2900](https://www.sqlalchemy.org/trac/ticket/2900)

### orm 扩展

+   **[orm] [扩展] [特性]**

    添加了一个新的、**实验性的**扩展`sqlalchemy.ext.automap`。该扩展扩展了声明性的功能，以及`DeferredReflection`类，生成一个基类，根据表元数据自动生成映射类和*关系*。 

    另请参阅

    自动映射扩展

    自动映射

### sql

+   **[sql] [特性]**

    像`and_()`和`or_()`这样的连接词现在可以接受 Python 生成器作为单个参数，例如：

    ```py
    and_(x == y for x, y in tuples)
    ```

    这里的逻辑查找一个单个参数`*args`，其中第一个元素是`types.GeneratorType`的实例。

### 模式

+   **[模式] [特性]**

    `Table.extend_existing`和`Table.autoload_replace`参数现在可用于`MetaData.reflect()`方法。

## 0.9.0

发布日期：2013 年 12 月 30 日

### orm

+   **[orm] [特性]**

    `StatementError`或与 DBAPI 相关的子类现在可以容纳有关异常“原因”的其他信息；当异常发生在自动刷新中时，`Session`现在会为其添加一些细节。采用这种方法是为了与 Python 3 风格的“链接异常”方法相反，以保持与 Py2K 代码以及已经捕获`IntegrityError`或类似异常的代码的兼容性。

+   **[orm] [feature] [backrefs]**

    向`validates()`函数添加了新参数`include_backrefs=True`；当设置为 False 时，如果事件是从另一侧的属性操作的反向引用发起的，则不会触发验证事件。

    请参见

    @validates 的 include_backrefs=False 选项

    参考：[#1535](https://www.sqlalchemy.org/trac/ticket/1535)

+   **[orm] [feature]**

    添加了用于指定`SELECT`的`FOR UPDATE`子句的新 API，通过新的`Query.with_for_update()`方法，以补充新的`GenerativeSelect.with_for_update()`方法。感谢 Mario Lassnig 的拉取请求。

    请参见

    在 select()，Query()上新增 FOR UPDATE 支持

+   **[orm] [bug]**

    对`subqueryload()`策略进行了调整，确保查询在加载过程开始后运行；这样，subqueryload 优先于其他加载器运行，这些加载器可能由于其他错误的时机导致命中相同属性。

    此更改也已**回溯**至：0.8.5

    参考：[#2887](https://www.sqlalchemy.org/trac/ticket/2887)

+   **[orm] [bug]**

    修复了从表到基表的联接表继承时，PK 列名称不同的 bug；持久性系统将无法将主键值从基表复制到继承表中进行 INSERT。

    此更改也已**回溯**至：0.8.5

    参考：[#2885](https://www.sqlalchemy.org/trac/ticket/2885)

+   **[orm] [bug]**

    当传递的列/属性（名称）不解析为列或映射属性（例如错误的元组）时，`composite()`将引发一个信息性错误消息；以前引发一个未绑定的本地错误。

    此更改也**回溯到**：0.8.5

    参考：[#2889](https://www.sqlalchemy.org/trac/ticket/2889)

+   **[orm] [bug]**

    修复了由[#2818](https://www.sqlalchemy.org/trac/ticket/2818)引入的回归，其中生成的 EXISTS 查询会为具有两个同名列的语句产生“正在替换列”的警告，因为内部 SELECT 不会设置 use_labels。

    此更改也**回溯到**：0.8.4

    参考：[#2818](https://www.sqlalchemy.org/trac/ticket/2818)

+   **[orm] [bug] [collections] [py3k]**

    在 ORM 集合仪器化系统中添加了对 Python 3 方法`list.clear()`的支持；感谢 Eduardo Schettino 的拉取请求。

+   **[orm] [bug]**

    对于`AliasedClass`构造进行了一些改进，涉及到描述符，如混合体、同义词、复合体、用户定义的描述符等。进行的属性适配更加健壮，这样，如果一个描述符返回另一个被仪器化的属性，而不是一个复合 SQL 表达式元素，则操作仍将进行。此外，“适配”的运算符将保留其类；以前，从`InstrumentedAttribute`到`QueryableAttribute`（一个超类）的类的更改会与 Python 的运算符系统进行交互，例如`aliased(MyClass.x) > MyClass.x`表达式将反转为`myclass.x < myclass_1.x`。适配的属性还将引用新的`AliasedClass`作为其父类，而以前并不总是这样。

    参考：[#2872](https://www.sqlalchemy.org/trac/ticket/2872)

+   **[orm] [bug]**

    `relationship()`上的`viewonly`标志现在将防止为目标属性代表写入属性历史记录。这导致在对象发生变化时不会将其写入到 Session.dirty 列表中。以前，对象会出现在 Session.dirty 中，但在 flush 期间不会代表修改的属性进行任何更改。属性仍然会发出事件，如反向引用事件和用户定义的事件，并且仍将从反向引用接收变化。

    另见

    在关系（relationship()）上使用`viewonly=True`会防止历史记录生效 migration_09.html#migration-2833。

    参考：[#2833](https://www.sqlalchemy.org/trac/ticket/2833)

+   **[orm] [bug]**

    为`scoped_session`添加了对新`Session.info`属性的支持。

+   **[orm] [bug]**

    修复了使用新的`Bundle`对象将导致`Query.column_descriptions`属性失败的错误。

+   **[orm] [bug] [sql] [sqlite]**

    修复了由[#2369](https://www.sqlalchemy.org/trac/ticket/2369)和[#2587](https://www.sqlalchemy.org/trac/ticket/2587)的连接重写功能引入的回归，其中一个嵌套连接的一侧已经是一个别名选择，将无法正确地转换外部的 ON 子句；在 ORM 中，当使用 SELECT 语句作为“次要”表时，可以看到这种情况。

    参考：[#2858](https://www.sqlalchemy.org/trac/ticket/2858)

### orm 声明性

+   **[orm] [declarative] [bug]**

    Declarative 额外进行了检查，以检测是否将相同的`Column`多次映射到不同的属性下（通常应该是一个`synonym()`），或者是否给两个或更多`Column`对象赋予了相同的名称，在检测到这种情况时会引发警告。

    参考：[#2828](https://www.sqlalchemy.org/trac/ticket/2828)

+   **[orm] [declarative] [bug]**

    `DeferredReflection`类已增强，以提供对由`relationship()`引用的“次要”表的自动反射支持。“次要”，当指定为字符串表名或只有名称和`MetaData`对象的`Table`对象时，在调用`DeferredReflection.prepare()`时，也将包括在反射过程中。

    参考：[#2865](https://www.sqlalchemy.org/trac/ticket/2865)

+   **[orm] [declarative] [bug]**

    修复了 Py2K 中不接受 Unicode 字面量作为声明中使用`relationship()`的类或其他参数的字符串名称的 bug。

### 示例

+   **[examples] [bug]**

    修复了阻止 history_meta 配方与超过一级深度的联合继承方案一起使用的错误。

### 引擎

+   **[engine] [feature]**

    `engine_from_config()`函数已经改进，以便我们能够从字符串配置字典中解析特定于方言的参数。方言类现在可以提供自己的参数类型列表和字符串转换例程。然而，这个功能目前尚未被内置方言使用。

    参考：[#2875](https://www.sqlalchemy.org/trac/ticket/2875)

+   **[engine] [bug]**

    当`connect()`方法引发一个不是`dbapi.Error`子类（比如`TypeError`、`NotImplementedError`等）的错误时，DBAPI 将不加修改地传播异常。之前，`connect()`方法特定的错误处理会不恰当地通过方言的`Dialect.is_disconnect()`方法运行异常，并将其包装在一个`sqlalchemy.exc.DBAPIError`中。现在，异常将以与执行过程中相同的方式不加修改地传播。

    此更改也被**回溯**到：0.8.4

    参考：[#2881](https://www.sqlalchemy.org/trac/ticket/2881)

+   **[engine] [bug] [pool]**

    `QueuePool`已经得到增强，当现有连接尝试阻塞时，不会阻止新的连接尝试。之前，新连接的生成在监视溢出的块内串行化；现在，溢出计数器在连接过程之外的自己的关键部分中被改变。

    此更改也被**回溯**到：0.8.4

    参考：[#2880](https://www.sqlalchemy.org/trac/ticket/2880)

+   **[engine] [bug] [pool]**

    对等待可用连接的逻辑进行了轻微调整，对于未指定超时的连接池，每隔半秒就会中断等待以检查所谓的“中止”标志，这允许等待者在整个连接池被释放的情况下中断；通常情况下，等待者应该由`notify_all()`中断，但在极少数情况下可能会错过这个`notify_all()`。这是在 0.8.0 中首次引入的逻辑的扩展，该问题只在压力测试中偶尔观察到。

    此更改也被**回溯**到：0.8.4

    参考：[#2522](https://www.sqlalchemy.org/trac/ticket/2522)

+   **[engine] [bug]**

    修复了一个 bug，在预 DBAPI `StatementError` 在 `Connection.execute()` 中引发时，SQL 语句会被错误地 ASCII 编码，导致非 ASCII 语句的编码错误。现在字符串化仍然保持在 Python unicode 中，从而避免编码错误。

    此更改也已**回溯**至：0.8.4

    参考：[#2871](https://www.sqlalchemy.org/trac/ticket/2871)

+   **[engine] [bug]**

    `create_engine()` 例程和相关的 `make_url()` 函数不再将 `+` 符号视为密码字段中的空格。在这个区域的解析已经调整得更接近 RFC 1738 处理这些标记的方式，即 `username` 和 `password` 都只期望 `:`、`@` 和 `/` 被编码。

    参见

    `create_engine()` 的“密码”部分不再将 + 号视为编码空格

    参考：[#2873](https://www.sqlalchemy.org/trac/ticket/2873)

+   **[engine] [bug]**

    `RowProxy` 对象现在在 Python 中可排序，就像普通元组一样；这是通过在 `__eq__()` 方法中确保在两侧都进行 tuple() 转换以及添加 `__lt__()` 方法来实现的。

    参见

    RowProxy 现在具有元组排序行为

    参考：[#2848](https://www.sqlalchemy.org/trac/ticket/2848)

### sql

+   **[sql] [feature]**

    `text()` 构造的新改进，包括更灵活的设置绑定参数和返回类型的方式；特别是，现在可以将 `text()` 转换为完整的 FROM 对象，在其他语句中作为别名或 CTE 嵌入，使用新方法 `TextClause.columns()`。当构造在“文字绑定”上下文中编译时，`text()` 构造也可以呈现“内联”绑定参数。

    参见

    新的 text() 功能

    参考：[#2877](https://www.sqlalchemy.org/trac/ticket/2877), [#2882](https://www.sqlalchemy.org/trac/ticket/2882)

+   **[sql] [feature]**

    使用新的`GenerativeSelect.with_for_update()`方法添加了指定`SELECT`的`FOR UPDATE`子句的新 API。与`select()`的`for_update`关键字参数相比，该方法支持更直观的设置方言特定选项的系统，并且还包括对 SQL 标准`FOR UPDATE OF`子句的支持。ORM 还包括一个新的对应方法`Query.with_for_update()`。感谢 Mario Lassnig 的拉取请求。

    另请参阅

    在 select()、Query()上新增 FOR UPDATE 支持

+   **[sql] [功能]**

    当通过字符串将返回的浮点数值强制转换为 Python `Decimal` 时使用的精度现在是可配置的。标志`decimal_return_scale`现在由所有`Numeric`和`Float`类型支持，它将确保从原生浮点值中取出这么多位数字时，它被转换为字符串。如果不存在，类型将使用`.scale`的值（如果类型支持此设置且不为 None）。否则，将使用原始默认长度 10。

    另请参阅

    浮点数字符串转换精度可配置为原生浮点数类型

    参考：[#2867](https://www.sqlalchemy.org/trac/ticket/2867)

+   **[sql] [错误]**

    修复了一个问题：如果主键列上有一个序列，但列不是“自动增量”列，可能是因为它有一个外键约束或设置了`autoincrement=False`，则对于不支持序列的后端，在插入缺少主键值的 INSERT 时，会尝试触发序列。这将在非序列后端（如 SQLite、MySQL）上发生。

    此更改也**回溯**到：0.8.5

    参考：[#2896](https://www.sqlalchemy.org/trac/ticket/2896)

+   **[sql] [错误]**

    修复了`Insert.from_select()`方法的错误，其中给定名称的顺序在生成 INSERT 语句时不会被考虑，因此与给定 SELECT 语句中的列名不匹配。还注意到，`Insert.from_select()`暗示不能使用 Python 端的插入默认值，因为语句没有 VALUES 子句。

    此更改也**回溯**到：0.8.5

    参考：[#2895](https://www.sqlalchemy.org/trac/ticket/2895)

+   **[sql] [bug]**

    当给定普通文字值时，`cast()` 函数现在将根据给定的类型将给定的文字值应用于绑定参数侧，与`type_coerce()` 函数的方式相同。然而，与`type_coerce()` 不同的是，只有在将非 clauseelement 值传递给`cast()` 时才会生效；现有的已经有类型的构造将保留其类型。

+   **[sql] [bug]**

    `ForeignKey` 类更积极地检查给定的列参数。如果不是字符串，则检查对象至少是一个`ColumnClause`，或者一个解析为一个的对象，如果`.table` 属性存在，则它引用一个`TableClause` 或子类，而不是像`Alias` 这样的东西。否则，将引发一个`ArgumentError`。

    参考：[#2883](https://www.sqlalchemy.org/trac/ticket/2883)

+   **[sql] [bug]**

    `ColumnOperators.collate()` 操作符的优先规则已修改，使得 COLLATE 操作符现在比比较操作符的优先级低。这样做的效果是，应用于比较的 COLLATE 不会在比较周围生成括号，这对于 MSSQL 等后端不解析括号的情况是不兼容的。对于那些通过将 `Operators.collate()` 应用于比较表达式的单个元素而不是整个比较表达式来解决该问题的设置而言，该更改是向后不兼容的。

    请参阅

    COLLATE 的优先规则已更改

    参考：[#2879](https://www.sqlalchemy.org/trac/ticket/2879)

+   **[sql] [enhancement]**

    当编译的语句中存在一个`BindParameter` 而没有值时引发的异常现在在错误消息中包括绑定参数的键名。

    这个改变也被**反向移植**到：0.8.5

### 模式

+   **[模式] [错误]**

    由 [#2812](https://www.sqlalchemy.org/trac/ticket/2812) 引起的回归 bug，如果名称包含非 ASCII 字符，则表名和列名的 repr() 将失败。

    参考：[#2868](https://www.sqlalchemy.org/trac/ticket/2868)

### postgresql

+   **[postgresql] [功能]**

    添加了对 PostgreSQL JSON 的支持，使用新的 `JSON` 类型。非常感谢 Nathan Rice 实现和测试。

    参考：[#2581](https://www.sqlalchemy.org/trac/ticket/2581)

+   **[postgresql] [功能]**

    通过 `TSVECTOR` 类型添加了对 PostgreSQL TSVECTOR 的支持。感谢 Noufal Ibrahim 的拉取请求。

+   **[postgresql] [错误]**

    修复了使用 pypostgresql 适配器时索引反射会错误解释 indkey 值的 bug，该适配器将这些值作为列表返回，而不是 psycopg2 的字符串返回类型。

    此更改也**回溯**到：0.8.4

    参考：[#2855](https://www.sqlalchemy.org/trac/ticket/2855)

+   **[postgresql] [错误]**

    现在使用 psycopg2 UNICODEARRAY 扩展来处理带有 psycopg2 + 普通“本地 unicode”模式的 unicode 数组，与使用 UNICODE 扩展的方式相同。

+   **[postgresql] [错误]**

    修复了 ENUM 中的值未对单引号符号进行转义的 bug。请注意，对于手动转义单引号的现有解决方法来说，这是不兼容的。

    另请参阅

    PostgreSQL CREATE TYPE <x> AS ENUM 现在对值应用引号

    参考：[#2878](https://www.sqlalchemy.org/trac/ticket/2878)

### mysql

+   **[mysql] [错误]**

    改进了 SQL 类型在 `__repr__()` 中生成的系统，特别是关于 MySQL 整数/数字/字符类型，这些类型具有各种关键字参数。`__repr__()` 对于 Alembic autogenerate 很重要，用于在迁移脚本中呈现 Python 代码时使用。

    参考：[#2893](https://www.sqlalchemy.org/trac/ticket/2893)

### mssql

+   **[mssql] [错误] [firebird]**

    与 `Float` 类型一起使用的“asdecimal”标志现在也适用于 Firebird 以及 mssql+pyodbc 方言；以前未发生十进制转换。

    此更改也**回溯**到：0.8.5

+   **[mssql] [错误] [pymssql]**

    将“Net-Lib error during Connection reset by peer”消息添加到 pymssql 方言中检查的“断开”消息列表中。感谢 John Anderson。

    此更改也**回溯**到：0.8.5

+   **[mssql] [错误]**

    修复了在 0.8.0 中引入的 bug，如果索引位于备用模式中，则 MSSQL 中的 `DROP INDEX` 语句会错误渲染；模式名/表名将被颠倒。格式也已经修订以匹配当前的 MSSQL 文档。感谢 Derek Harland。

    此更改也被**回溯**到：0.8.4

### oracle

+   **[oracle] [错误]**

    将 ORA-02396 “最大空闲时间”错误代码添加到具有 cx_oracle 的“is disconnect”代码列表中。

    此更改也被**回溯**到：0.8.4

    参考：[#2864](https://www.sqlalchemy.org/trac/ticket/2864)

+   **[oracle] [错误]**

    修复了 Oracle 中给定没有长度的 `VARCHAR` 类型（例如用于 `CAST` 或类似操作）会错误地呈现为 `None CHAR` 或类似情况的 bug。

    此更改也被**回溯**到：0.8.4

    参考：[#2870](https://www.sqlalchemy.org/trac/ticket/2870)

### 杂项

+   **[错误] [firebird]**

    firebird 方言将引用以下划线开头的标识符。感谢 Treeve Jelbert。

    此更改也被**回溯**到：0.8.5

    参考：[#2897](https://www.sqlalchemy.org/trac/ticket/2897)

+   **[错误] [firebird]**

    修复了 Firebird 索引反射中列在索引内部没有正确排序的 bug；现在按照 RDB$FIELD_POSITION 的顺序排序。

    此更改也被**回溯**到：0.8.5

+   **[错误] [declarative]**

    当发送给 `relationship()` 的字符串参数不能解析为类或映射器时，错误消息已更正为与接收非字符串参数时相同的方式，指示配置错误的关系名称。

    此更改也被**回溯**到：0.8.5

    参考：[#2888](https://www.sqlalchemy.org/trac/ticket/2888)

+   **[错误] [扩展]**

    修复了 `serializer` 扩展无法正确处理包含非 ASCII 字符的表或列名称的 bug。

    此更改也被**回溯**到：0.8.4

    参考：[#2869](https://www.sqlalchemy.org/trac/ticket/2869)

+   **[错误] [firebird]**

    更改了 Firebird 用于列出表和视图名称的查询，现在从 `rdb$relations` 视图而不是 `rdb$relation_fields` 和 `rdb$view_relations` 视图查询。许多 FAQ 和博客中提到了新旧查询的变体，但新查询直接来自“Firebird FAQ”，这似乎是最官方的信息来源。

    参考：[#2898](https://www.sqlalchemy.org/trac/ticket/2898)

+   **[已移除]**

    “informix” 和 “informixdb” 方言已被移除；该代码现在作为一个独立的存储库在 Bitbucket 上提供。自从第一次添加 informixdb 方言以来，IBM-DB 项目提供了生产级的 Informix 支持。

## 0.9.0b1

发布日期：2013 年 10 月 26 日

### 通用

+   **[通用] [特性] [py3k]**

    C 扩展已移植到 Python 3，并将在任何支持的 CPython 2 或 3 环境下构建。

    参考：[#2161](https://www.sqlalchemy.org/trac/ticket/2161)

+   **[通用] [特性] [py3k]**

    代码现在对 Python 2 和 3 “原地”运行，不再需要运行 2to3。兼容性现在针对 Python 2.6 及更高版本。

    参考：[#2671](https://www.sqlalchemy.org/trac/ticket/2671)

+   **[通用]**

    大规模的包重构已经重新组织了许多核心模块的导入结构以及一些 ORM 模块的方面。特别是`sqlalchemy.sql`已经被拆分成比以前更多的模块，以便将`sqlalchemy.sql.expression`的非常庞大的大小削减下来。这一努力主要集中在大幅减少导入循环上。此外，`sqlalchemy.sql.expression`和`sqlalchemy.orm`中的 API 函数系统已经重新组织，以消除函数与它们产生的对象之间文档中的冗余。

### orm

+   **[orm] [feature]**

    添加了新选项`relationship()` `distinct_target_key`。这使得子查询预加载策略可以对最内部的 SELECT 子查询应用 DISTINCT，以帮助解决内部查询生成重复行的情况（尚无解决子查询预加载中重复行问题的通用解决方案，但是当最内部子查询之外的连接产生重复行时）。当标志设置为`True`时，DISTINCT 被无条件渲染，当设置为`None`时，如果最内部关系目标列不包括完整的主键，则渲染 DISTINCT。该选项在 0.8 中默认为 False（例如，在所有情况下默认关闭），在 0.9 中默认为 None（例如，默认情况下自动）。感谢 Alexander Koval 对此的帮助。

    参见

    子查询预加载将对某些查询的最内部 SELECT 应用 DISTINCT

    此更改也被**回溯**到：0.8.3

    参考：[#2836](https://www.sqlalchemy.org/trac/ticket/2836)

+   **[orm] [feature]**

    现在，当从标量关系中获取标量属性时，关联代理会返回`None`，而不是引发`AttributeError`。

    参见

    关联代理缺失标量返回 None

    参考：[#2810](https://www.sqlalchemy.org/trac/ticket/2810)

+   **[orm] [feature]**

    添加了新方法`AttributeState.load_history()`，类似于`AttributeState.history`但也触发加载器可调用。

    参见

    attributes.get_history()如果值不存在将默认从数据库查询

    参考：[#2787](https://www.sqlalchemy.org/trac/ticket/2787)

+   **[orm] [feature]**

    添加了一个新的加载选项`load_only()`。这允许指定一系列列名仅加载这些属性，推迟其余属性的加载。

    参考：[#1418](https://www.sqlalchemy.org/trac/ticket/1418)

+   **[orm] [feature]**

    现在加载器选项系统已完全重构，建立在更加全面的基础上，即`Load`对象。该基础允许任何常见的加载器选项，如`joinedload()`、`defer()`等，以“链式”样式用于沿着路径指定选项，例如`joinedload("foo").subqueryload("bar")`。新系统取代了点分隔路径名的使用、选项中的多个属性以及使用`_all()`选项的用法。

    另请参阅

    新查询选项 API；load_only() 选项

    参考：[#1418](https://www.sqlalchemy.org/trac/ticket/1418)

+   **[orm] [功能]**

    当在基于列的`Query`中使用时，`composite()`构造现在会保持返回对象，而不是展开为单独的列。这利用了内部的新`Bundle`功能。此行为是向后不兼容的；要从一个会展开的组合列中进行选择，请使用`MyClass.some_composite.clauses`。

    另请参阅

    组合属性现在在按属性查询时作为它们的对象形式返回

    参考：[#2824](https://www.sqlalchemy.org/trac/ticket/2824)

+   **[orm] [功能]**

    添加了一个新的构造`Bundle`，它允许对`Query`构造中的一组列表达式进行规范。默认情况下，这组列作为单个元组返回。然而，可以覆盖`Bundle`的行为，以提供对返回行进行任何类型的结果处理。当组合属性在基于列的`Query`中使用时，`Bundle`的行为也已嵌入其中。

    另请参阅

    ORM 查询的列束

    组合属性现在在按属性查询时作为它们的对象形式返回

    参考：[#2824](https://www.sqlalchemy.org/trac/ticket/2824)

+   **[orm] [功能]**

    `Mapper`的`version_id_generator`参数现在可以指定依赖于服务器生成的版本标识符，使用触发器或其他数据库提供的版本控制功能，或通过设置`version_id_generator=False`来使用可选的程序值。当使用服务器生成的版本标识符时，ORM 将在可用时使用 RETURNING 立即加载新的版本值，否则将发出第二个 SELECT。

    参考：[#2793](https://www.sqlalchemy.org/trac/ticket/2793)

+   **[orm] [feature]**

    `Mapper`的`eager_defaults`标志现在将允许使用内联 RETURNING 子句获取新���成的默认值，而不是使用第二个 SELECT 语句，对于支持 RETURNING 的后端。

    参考：[#2793](https://www.sqlalchemy.org/trac/ticket/2793)

+   **[orm] [feature]**

    添加了一个新的属性`Session.info`到`Session`；这是一个字典，应用程序可以将任意数据存储在与`Session`相关的本地数据中。`Session.info`的内容也可以使用`Session`或`sessionmaker`的`info`参数进行初始化。

+   **[orm] [feature]**

    现在已实现事件监听器的移除。该功能通过`remove()`函数提供。

    另请参阅

    事件移除 API

    参考：[#2268](https://www.sqlalchemy.org/trac/ticket/2268)

+   **[orm] [feature]**

    属性事件通过`AttributeImpl`传递的机制已更改为一个“initiator”标记，现在对象是一个名为`Event`的特定于事件的对象。此外，属性系统不再根据匹配的“initiator”标记停止事件；这一逻辑已移至特定于 ORM 反向引用事件处理程序的地方，这些处理程序是重新传播属性事件到后续附加/设置/移除操作的典型来源。模拟反向引用处理程序行为的最终用户代码现在必须确保递归事件传播方案被停止，如果该方案不使用反向引用处理程序。使用这个新系统，当对象被附加到集合中，与新的多对一关联，与之前的多对一解除关联，然后从之前的集合中移除时，反向引用处理程序现在可以执行“两跳”操作。在这个改变之前，从之前的集合中移除的最后一步将不会发生。

    另请参阅

    反向引用处理程序现在可以传播多个级别

    参考：[#2789](https://www.sqlalchemy.org/trac/ticket/2789)

+   **[orm] [feature]**

    关于 ORM 如何构建右侧本身是 JOIN 或 LEFT OUTER JOIN 的连接的重大变更。ORM 现在配置为允许形式为`a JOIN (b JOIN c ON b.id=c.id) ON a.id=b.id`的连接的简单嵌套，而不是强制右侧成为`SELECT`子查询。这应该允许大多数后端实现显着的性能改进，尤其是 MySQL。多年来一直阻碍此更改的一个数据库后端，SQLite，现在通过将`SELECT`子查询的生成从 ORM 移动到 SQL 编译器来解决；因此，在 SQLite 上的右侧嵌套连接最终仍将以`SELECT`呈现，而所有其他后端不再受此解决方法的影响。

    作为这一变更的一部分，`aliased()`、`Join.alias()`和`with_polymorphic()`函数中添加了一个新参数`flat=True`，允许生成一个 JOIN 的“别名”，该别名对加入的每个组件表应用匿名别名，而不是生成子查询。

    另请参阅

    许多 JOIN 和 LEFT OUTER JOIN 表达式将不再被包装在(SELECT * FROM ..) AS ANON_1 中

    参考：[#2587](https://www.sqlalchemy.org/trac/ticket/2587)

+   **[orm] [bug]**

    修复了列表插入操作`insert(0, item)`无法正确表示`[0:0]`的切片的错误，特别是在使用关联代理时可能发生。由于 Python 集合中的某些怪癖，该问题在 Python 3 中更有可能发生。

    此更改还**回溯**到：0.8.3, 0.7.11

    参考：[#2807](https://www.sqlalchemy.org/trac/ticket/2807)

+   **[orm] [bug]**

    修复了在使用`remote()`或`foreign()`等注释在与父`Table`关联之前可能会产生与父表在连接中未呈现相关的问题的错误，这是由注释执行的固有复制操作引起的。

    此更改还**回溯**到：0.8.3

    参考：[#2813](https://www.sqlalchemy.org/trac/ticket/2813)

+   **[orm] [bug]**

    修复了`Query.exists()`在没有任何 WHERE 条件的情况下无法正常工作的错误。感谢 Vladimir Magamedov。

    此更改还**回溯**到：0.8.3

    参考：[#2818](https://www.sqlalchemy.org/trac/ticket/2818)

+   **[orm] [bug]**

    修复了 ORM 使用的有序序列实现中的潜在问题，该问题用于迭代映射器层次结构；在 Jython 解释器下，即使 cPython 和 PyPy 保持了排序，该实现也没有排序。

    此更改还 **被后移** 到：0.8.3

    参考：[#2794](https://www.sqlalchemy.org/trac/ticket/2794)

+   **[orm] [bug]**

    修复了 ORM 级别事件注册中的错误，其中在某些“未映射的基类”配置中，“原始”或“传播”标志可能会被错误配置。

    此更改还 **被后移** 到：0.8.3

    参考：[#2786](https://www.sqlalchemy.org/trac/ticket/2786)

+   **[orm] [bug]**

    与加载映射实体时使用 `defer()` 选项相关的性能修复。在加载时将每个对象的延迟可调用函数应用到实例的函数开销明显高于仅从行中加载数据的函数开销（请注意，`defer()` 旨在减少 DB/网络开销，而不一定是函数调用计数）；在所有情况下，函数调用开销现在都小于从列加载数据的开销。每次加载从 N（结果中的总延迟值）减少到 1（延迟列的总数）的“惰性可调用”对象的数量也有所减少。

    此更改还 **被后移** 到：0.8.3

    参考：[#2778](https://www.sqlalchemy.org/trac/ticket/2778)

+   **[orm] [bug]**

    修复了属性历史函数的错误，当我们使用 `make_transient()` 函数将对象从“持久”移动到“待定”时，涉及基于集合的反向引用的操作会失败。

    此更改还 **被后移** 到：0.8.3

    参考：[#2773](https://www.sqlalchemy.org/trac/ticket/2773)

+   **[orm] [bug]**

    尝试刷新继承类的对象时，如果多态识别器已分配给对该类无效的值，则会发出警告。

    此更改还 **被后移** 到：0.8.2

    参考：[#2750](https://www.sqlalchemy.org/trac/ticket/2750)

+   **[orm] [bug]**

    修复了多态 SQL 生成中的错误，在针对相同基类的多个连接继承实体相互连接的情况下，如果连接字符串超过两个实体，则不会独立跟踪基表上的列。

    此更改还 **被后移** 到：0.8.2

    参考：[#2759](https://www.sqlalchemy.org/trac/ticket/2759)

+   **[orm] [bug]**

    修复了将复合属性发送到 `Query.order_by()` 会产生一些数据库不接受的带括号的表达式的错误。

    此更改还 **被后移** 到：0.8.2

    参考：[#2754](https://www.sqlalchemy.org/trac/ticket/2754)

+   **[orm] [bug]**

    修复了复合属性与`aliased()`函数之间的交互。以前，在应用别名时，复合属性在比较操作中无法正常工作。

    此更改也**回溯**到：0.8.2

    参考：[#2755](https://www.sqlalchemy.org/trac/ticket/2755)

+   **[orm] [bug] [ext]**

    修复了`MutableDict`在调用`clear()`时未报告更改事件的错误。

    此更改也**回溯**到：0.8.2

    参考：[#2730](https://www.sqlalchemy.org/trac/ticket/2730)

+   **[orm] [bug]**

    当与标量列映射属性一起使用时，`get_history()`现在将遵守传递给它的“被动”标志；由于默认为`PASSIVE_OFF`，如果值不存在，默认情况下该函数将查询数据库。这与 0.8 相比是一种行为变化。

    另请参阅

    attributes.get_history()将默认从数据库查询值不存在的情况

    参考：[#2787](https://www.sqlalchemy.org/trac/ticket/2787)

+   **[orm] [bug] [associationproxy]**

    对于与标量值一起使用的==，！=比较器，添加了额外的条件，用于将与 None 进行比较的情况考虑到也要考虑到关联记录本身不存在，除了现有的测试关联记录上的标量端点是否为 NULL。以前，比较`Cls.scalar == None`将返回`Cls.associated`存在且`Cls.associated.scalar`为 None 的记录，但不会返回`Cls.associated`不存在的行。更重要的是，相反的操作`Cls.scalar != None` *会*返回`Cls`行，其中`Cls.associated`不存在。

    `Cls.scalar != 'somevalue'`的情况也修改为更像直接的 SQL 比较；只返回`Cls.associated`存在且`Associated.scalar`非 NULL 且不等于`'somevalue'`的行。以前，这将是一个简单的`NOT EXISTS`。

    还添加了一个特殊用例，当`Cls.scalar`是基于列的值时，您可以调用`Cls.scalar.has()`而不带参数 - 这将返回`Cls.associated`是否有任何行存在，而不管`Cls.associated.scalar`是否为 NULL。

    另请参阅

    关联代理 SQL 表达式改进和修复

    参考：[#2751](https://www.sqlalchemy.org/trac/ticket/2751)

+   **[orm] [bug]**

    修复了一个晦涩的错误，当跨越一个具有特定鉴别器值的单表继承子类的多对多关系进行连接/联接加载时，错误的结果会被获取，这是由于“secondary”行的返回。现在，在所有 ORM 在多对多关系上的连接中，“secondary”和右侧表现在括号内进行内部连接，以便左->右连接可以准确过滤。这个改变是通过最终解决了在 [#2587](https://www.sqlalchemy.org/trac/ticket/2587) 中概述的右嵌套连接问题而实现的。

    另请参阅

    许多 JOIN 和 LEFT OUTER JOIN 表达式将不再被包装在 (SELECT * FROM ..) AS ANON_1 中

    参考：[#2369](https://www.sqlalchemy.org/trac/ticket/2369)

+   **[orm] [错误]**

    `Query.select_from()` 方法的“自动别名”行为已被关闭。现在特定行为可以通过新方法 `Query.select_entity_from()` 获得。这里的自动别名行为从未有很好的文档记录，并且通常不是所期望的，因为 `Query.select_from()` 已经更加倾向于控制 JOIN 的渲染方式。`Query.select_entity_from()` 也将在 0.8 中提供，以便依赖于自动别名的应用程序可以转向使用这种方法。

    另请参阅

    _query.Query.select_from() 不再将子句应用于相应的实体

    参考：[#2736](https://www.sqlalchemy.org/trac/ticket/2736)

### orm declarative

+   **[orm] [declarative] [feature]**

    添加了一个方便的类装饰器 `as_declarative()`，它是 `declarative_base()` 的包装器，允许使用一种巧妙的类装饰方法应用现有的基类。

    这个改变也被**回溯**到：0.8.3

+   **[orm] [declarative] [feature]**

    ORM 描述符，如混合属性，现在可以通过名称在与 `order_by`、`primaryjoin` 或类似的在 `relationship()` 中使用的字符串参数中引用，除了列绑定属性。

    这个改变也被**回溯**到：0.8.2

    参考：[#2761](https://www.sqlalchemy.org/trac/ticket/2761)

### 例子

+   **[例子] [功能]**

    改进了`examples/generic_associations`中的示例，包括`discriminator_on_association.py`利用单表继承来处理“鉴别器”的工作。还添加了一个真正的“通用外键”示例，它类似于其他流行框架，使用开放的整数指向任何其他表，放弃传统的引用完整性。虽然我们不推荐这种模式，但信息想要自由。

    此更改也**回溯**到：0.8.3

+   **[examples] [bug]**

    在版本控制示例中创建的历史表中添加了“autoincrement=False”，因为这个表在任何情况下都不应该具有自增。由 Patrick Schmid 提供。

    此更改也**回溯**到：0.8.3

+   **[examples] [bug]**

    修复了“版本控制”配方中的一个问题，即当存在反向引用时，多对一引用可能会为目标产生一个无意义的版本，即使它没有更改。补丁由 Matt Chisholm 提供。

    此更改也**回溯**到：0.8.2

### engine

+   **[engine] [feature]**

    对`Engine`的`URL`的`repr()`现在将使用星号隐藏密码。由 Gunnlaugur Þór Briem 提供。

    此更改也**回溯**到：0.8.3

    参考：[#2821](https://www.sqlalchemy.org/trac/ticket/2821)

+   **[engine] [feature]**

    添加到`ConnectionEvents`的新事件：

    +   `ConnectionEvents.engine_connect()`

    +   `ConnectionEvents.set_connection_execution_options()`

    +   `ConnectionEvents.set_engine_execution_options()`

    参考：[#2770](https://www.sqlalchemy.org/trac/ticket/2770)

+   **[engine] [bug]**

    `make_url()`函数使用的正则表达式现在解析 ipv6 地址，例如用括号括起来。

    此更改也**回溯**到：0.8.3, 0.7.11

    参考：[#2851](https://www.sqlalchemy.org/trac/ticket/2851)

+   **[engine] [bug] [oracle]**

    如果重新创建一个`Engine`，则不会再次调用 Dialect.initialize()，这是由于断开连接错误。这修复了 Oracle 8 方言中的一个特定问题，但通常情况下，dialect.initialize()阶段应该只执行一次。

    此更改也**回溯**到：0.8.3

    参考：[#2776](https://www.sqlalchemy.org/trac/ticket/2776)

+   **[engine] [bug] [pool]**

    修复了一个 bug，即`QueuePool`在现有池化连接在无效或重新生成事件后无法重新连接时会丢失正确的已检出计数。

    这个更改也被**回溯**到：0.8.3

    参考：[#2772](https://www.sqlalchemy.org/trac/ticket/2772)

+   **[engine] [bug]**

    修复了一个 bug，即各种`Pool`实现中的`reset_on_return`参数在重新生成池时不会传播。感谢 Eevee。

    这个更改也被**回溯**到：0.8.2

+   **[engine] [bug]**

    `Dialect.reflecttable()`的方法签名，在所有已知情况下由`DefaultDialect`提供，已经调整为期望`include_columns`和`exclude_columns`参数，不再有任何 kw 选项，减少了歧义 - 之前缺少了`exclude_columns`。

    参考：[#2748](https://www.sqlalchemy.org/trac/ticket/2748)

### sql

+   **[sql] [feature]**

    增加了对“唯一约束”反射的支持，通过`Inspector.get_unique_constraints()`方法。感谢 Roman Podolyaka 的补丁。

    这个更改也被**回溯**到：0.8.4

    参考：[#1443](https://www.sqlalchemy.org/trac/ticket/1443)

+   **[sql] [feature]**

    `update()`、`insert()`和`delete()`构造现在将 ORM 实体解释为要操作的目标表，例如：

    ```py
    from sqlalchemy import insert, update, delete

    ins = insert(SomeMappedClass).values(x=5)

    del_ = delete(SomeMappedClass).where(SomeMappedClass.id == 5)

    upd = update(SomeMappedClass).where(SomeMappedClass.id == 5).values(name="ed")
    ```

    这个更改也被**回溯**到：0.8.3

+   **[sql] [feature] [mysql] [postgresql]**

    PostgreSQL 和 MySQL 方言现在支持外键选项的反射/检查，包括 ON UPDATE、ON DELETE。PostgreSQL 还反映了 MATCH、DEFERRABLE 和 INITIALLY。感谢 ijl。

    参考：[#2183](https://www.sqlalchemy.org/trac/ticket/2183)

+   **[sql] [feature]**

    现在，当在类型化表达式中使用具有“null”类型（例如，未指定类型）的`bindparam()`构造时，会复制该构造，并将新副本分配给比较列的实际类型。以前，此逻辑会在给定的`bindparam()`中发生。此外，现在对于传递给`ValuesBase.values()`用于`Insert`或`Update`构造的`bindparam()`构造，在构造的编译阶段中也会发生类似的过程。

    这两者都是一些微妙的行为变化，可能会影响某些用法。

    请参见

    当可用类型时，没有类型的`bindparam()`构造通过复制升级

    参考：[#2850](https://www.sqlalchemy.org/trac/ticket/2850)

+   **[sql] [feature]**

    对于特殊符号的表达式处理进行了全面改进，特别是连接词，例如`None` `null()` `true()` `false()`，包括在连接中呈现 NULL 的一致性，包含布尔常量的`and_()`和`or_()`表达式的“短路”，以及对布尔常量和表达式的呈现与不支持`true`/`false`常量的后端相比较为“1”或“0”。

    请参见

    改进的布尔常量、NULL 常量、连接的呈现

    参考：[#2734](https://www.sqlalchemy.org/trac/ticket/2734), [#2804](https://www.sqlalchemy.org/trac/ticket/2804), [#2823](https://www.sqlalchemy.org/trac/ticket/2823)

+   **[sql] [feature]**

    类型系统现在处理渲染“字面绑定”值的任务，例如通常绑定参数但由于上下文必须呈现为字符串的值，通常在 DDL 结构中，如 CHECK 约束和索引中（注意，随着 [#2742](https://www.sqlalchemy.org/trac/ticket/2742)，“字面绑定”值将成为 DDL 使用的）。新方法 `TypeEngine.literal_processor()` 作为基础，添加了 `TypeDecorator.process_literal_param()` 以允许包装本地字面渲染方法。

    另见

    现在，类型系统处理渲染“字面绑定”值的任务了。（见 “literal bind” 值的渲染）

    参考：[#2838](https://www.sqlalchemy.org/trac/ticket/2838)

+   **[sql] [feature]**

    `Table.tometadata()` 方法现在会复制结构中所有 `SchemaItem` 对象的所有 `SchemaItem.info` 字典，包括列、约束、外键等。由于这些字典是副本，它们与原始字典独立。以前，此操作仅传输了 `Column` 的 `.info` 字典，并且它仅链接在原地，而不是复制。

    参考：[#2716](https://www.sqlalchemy.org/trac/ticket/2716)

+   **[sql] [feature]**

    `Column` 的 `default` 参数现在接受类或对象方法作为参数，除了独立函数之外；将正确检测是否接受“上下文”参数。

+   **[sql] [feature]**

    `insert()` 构造中添加了新方法 `Insert.from_select()`。给定一个列列表和一个可选择项，渲染 `INSERT INTO (table) (columns) SELECT ..`。虽然此功能作为 0.9 的一部分突出显示，但也已经回溯到了 0.8.3。

    另见

    从 SELECT 插入

    参考：[#722](https://www.sqlalchemy.org/trac/ticket/722)

+   **[sql] [feature]**

    为`TypeDecorator`提供了一个名为`TypeDecorator.coerce_to_is_types`的新属性，以便更容易控制使用`==`或`!=`与`None`和布尔类型进行比较时如何生成`IS`表达式，或者带有绑定参数的普通相等表达式。

    参考：[#2734](https://www.sqlalchemy.org/trac/ticket/2734), [#2744](https://www.sqlalchemy.org/trac/ticket/2744)

+   **[sql] [feature]**

    在`ORDER BY`子句中，如果`label()`构造也在 select 的列子句中引用了该标签，则现在将仅以其名称单独呈现，而不是重写完整表达式。这使得数据库有更好的机会优化在两个不同上下文中评估相同表达式。

    另请参阅

    标签构造现在可以在 ORDER BY 中仅呈现其名称

    参考：[#1068](https://www.sqlalchemy.org/trac/ticket/1068)

+   **[sql] [bug]**

    修复了自 0.7.9 以来的回归，即如果在多个 FROM 子句中引用了 CTE 的名称，则可能无法正确引用。

    此更改也**回溯**到：0.8.3, 0.7.11

    参考：[#2801](https://www.sqlalchemy.org/trac/ticket/2801)

+   **[sql] [bug] [cte]**

    修复了公共表达式系统中的错误，如果 CTE 仅用作`alias()`构造，则不会使用 WITH 关键字呈现。

    此更改也**回溯**到：0.8.3, 0.7.11

    参考：[#2783](https://www.sqlalchemy.org/trac/ticket/2783)

+   **[sql] [bug]**

    修复了`CheckConstraint` DDL 中的错误，其中来自`Column`对象的“quote”标志不会传播。

    此更改也**回溯**到：0.8.3, 0.7.11

    参考：[#2784](https://www.sqlalchemy.org/trac/ticket/2784)

+   **[sql] [bug]**

    修复了`type_coerce()`在解释具有`__clause_element__()`方法的 ORM 元素时无法正确处理的错误。

    此更改也**回溯**到：0.8.3

    参考：[#2849](https://www.sqlalchemy.org/trac/ticket/2849)

+   **[sql] [bug]**

    当为“非本地”类型生成 CHECK 约束时，`Enum`和`Boolean`类型现在会绕过任何自定义（例如 TypeDecorator）类型。这样，自定义类型不会参与 CHECK 中的表达式，因为此表达式针对“impl”值而不是“decorated”值。

    这个更改也被**回溯**到：0.8.3

    参考：[#2842](https://www.sqlalchemy.org/trac/ticket/2842)

+   **[sql] [bug]**

    如果从未指定`unique`（默认为`None`）的`Column`生成了`Index`上的`.unique`标志，它可能会被生成为`None`。现在该标志将始终为`True`或`False`。

    这个更改也被**回溯**到：0.8.3

    参考：[#2825](https://www.sqlalchemy.org/trac/ticket/2825)

+   **[sql] [bug]**

    修复了默认编译器以及 postgresql、mysql 和 mssql 的 bug，以确保任何字面 SQL 表达式值在 CREATE INDEX 语句中直接呈现为字面值，而不是作为绑定参数。这也改变了其他 DDL 的呈现方案，如约束。

    这个更改也被**回溯**到：0.8.3

    参考：[#2742](https://www.sqlalchemy.org/trac/ticket/2742)

+   **[sql] [bug]**

    在其 FROM 子句中引用自身的`select()`，通常通过就地突变，将引发一个信息性错误消息，而不是导致递归溢出。

    这个更改也被**回溯**到：0.8.3

    参考：[#2815](https://www.sqlalchemy.org/trac/ticket/2815)

+   **[sql] [bug]**

    修复了一个 bug，使用`column_reflect`事件来更改传入的`Column`的`.key`会阻止主键约束、索引和外键约束被正确反映。

    这个更改也被**回溯**到：0.8.3

    参考：[#2811](https://www.sqlalchemy.org/trac/ticket/2811)

+   **[sql] [bug]**

    在 0.8 版本中添加的`ColumnOperators.notin_()`运算符现在在针对空集合使用时正确地产生表达式“IN”的否定返回。

    这个更改也被**回溯**到：0.8.3

+   **[sql] [bug] [postgresql]**

    修复了表达式系统依赖于某些表达式的`str()`形式，当引用`select()`构造上的`.c`集合时，但是`str()`形式不可用，因为元素依赖于特定于方言的编译构造，特别是与 PostgreSQL 的`ARRAY`元素一起使用的`__getitem__()`运算符。该修复还添加了一个新的异常类`UnsupportedCompilationError`，在编译器被要求编译它不知道如何处理的内容时引发。

    此更改也被**回溯**到：0.8.3

    参考：[#2780](https://www.sqlalchemy.org/trac/ticket/2780)

+   **[sql] [bug]**

    对`Select`构造的相关性行为进行了多次修复，首次引入于 0.8.0：

    +   为满足 FROM 条目应向外相关到包含另一个 SELECT 的 SELECT 的用例，然后包含此 SELECT，当通过`Select.correlate()`建立显式相关性时，现在当目标 SELECT 在链中的某处被包含在 WHERE/ORDER BY/columns 子句中时，相关性现在可以跨多个级别工作，而不仅仅是嵌套的 FROM 子句。这使得`Select.correlate()`再次更兼容于 0.7，同时仍保持新的“智能”相关性。

    +   当未使用显式相关性时，通常的“隐式”相关性将其行为限制在仅限于直接封闭的 SELECT，以最大限度地提高与 0.7 应用程序的兼容性，并且在这种情况下阻止跨嵌套 FROM 的相关性，保持与 0.8.0/0.8.1 的兼容性。

    +   `Select.correlate_except()`方法未能在所有情况下阻止给定的 FROM 子句相关性，并且还会导致 FROM 子句被错误地完全省略（更像是 0.7 会做的），这已经修复。

    +   调用 select.correlate_except(None)将按预期将所有 FROM 子句输入到相关性中。

    此更改也被**回溯**到：0.8.2

    参考：[#2668](https://www.sqlalchemy.org/trac/ticket/2668), [#2746](https://www.sqlalchemy.org/trac/ticket/2746)

+   **[sql] [bug]**

    修复了一个错误，即将一个表“A”的 select()与多个外键路径连接到表“B”，到表“B”，如果直接将表“A”连接到“B”会导致“模糊连接条件”错误，但是它会产生具有多个条件的连接条件。

    此更改也被**回溯**到：0.8.2

    参考：[#2738](https://www.sqlalchemy.org/trac/ticket/2738)

+   **[sql] [bug] [reflection]**

    修复了一个错误，即在跨远程模式和本地模式使用`MetaData.reflect()`可能会在两个模式都有相同名称的表时产生错误的结果。

    此更改也被**回溯**到：0.8.2

    参考：[#2728](https://www.sqlalchemy.org/trac/ticket/2728)

+   **[sql] [bug]**

    从基础`ColumnOperators`类中删除了“not implemented” `__iter__()`调用，虽然在 0.8.0 版本中引入此功能是为了防止在自定义操作符上实现`__getitem__()`方法并在该对象上错误地调用`list()`时出现无限增长的内存循环，但这会导致列元素报告它们实际上是可迭代类型，然后在尝试迭代时抛出错误。在这里没有真正的解决办法，所以我们遵循 Python 的最佳实践。在自定义操作符上实现`__getitem__()`时要小心！

    此更改也被**回溯**到：0.8.2

    参考：[#2726](https://www.sqlalchemy.org/trac/ticket/2726)

+   **[sql] [bug]**

    在调用“attach”事件之前，将“name”属性设置在`Index`上，以便可以使用附加事件根据父表和/或列动态生成索引名称。

    参考：[#2835](https://www.sqlalchemy.org/trac/ticket/2835)

+   **[sql] [bug]**

    错误的关键字参数“schema”已从`ForeignKey`对象中移除。这是一个意外提交，没有任何作用；在使用此关键字参数时，0.8.3 版本会发出警告。

    参考：[#2831](https://www.sqlalchemy.org/trac/ticket/2831)

+   **[sql] [bug]**

    对处理“引号”标识符的方式进行了重新设计，不再依赖于传递各种`quote=True`标志，而是将这些标志转换为包含引号信息的丰富字符串对象，这些信息包含在它们被传递给像`Table`、`Column`等常见模式构造时。这解决了各种方法不正确地遵守“引号”标志的问题，例如`Engine.has_table()`和相关方法。`quoted_name`对象是一个字符串子类，如果需要的话也可以显式使用；该对象将保留传递的引号偏好，并且还将绕过标准化为大写符号的方言执行的“名称规范化”。Oracle、Firebird 和 DB2 等标准化为大写符号的后端现在可以使用强制引号名称，例如小写引号名称和新的保留字。

    另请参阅

    模式标识符现在携带自己的引号信息

    参考：[#2812](https://www.sqlalchemy.org/trac/ticket/2812)

+   **[sql] [bug]**

    对`ForeignKey`对象到其目标`Column`的解析已经重新设计，以尽可能立即地进行，基于目标`Column`与与此`ForeignKey`关联的相同`MetaData`的时刻，而不是等待构建第一次连接或类似的时刻。这与其他改进一起，允许更早地检测到一些外键配置问题。此外，这里还包括了对类型传播系统的重新设计，因此现在应该可以可靠地在任何通过`ForeignKey`引用另一个列的`Column`上将类型设置为`None` - 该类型将在另一个列关联时立即从目标列复制，并且现在也适用于复合外键。

    另请参阅

    列现在可以可靠地从通过外键引用的列获取其类型

    参考：[#1765](https://www.sqlalchemy.org/trac/ticket/1765)

### postgresql

+   **[postgresql] [feature]**

    添加了对 PostgreSQL 9.2 范围类型的支持。目前，不提供类型转换，因此目前直接使用字符串或 psycopg2 2.5 范围扩展类型。补丁由 Chris Withers 提供。

    此更改也已**回溯**至：0.8.2

+   **[postgresql] [feature]**

    在使用 psycopg2 DBAPI 时，添加了对“AUTOCOMMIT”隔离的支持。该关键字可通过 `isolation_level` 执行选项使用。补丁由 Roman Podolyaka 提供。

    此更改也已**回溯**至：0.8.2

    参考：[#2072](https://www.sqlalchemy.org/trac/ticket/2072)

+   **[postgresql] [feature]**

    当在主键自增列上使用 `SmallInteger` 类型时，基于 PostgreSQL 版本 9.2 或更高版本的服务器版本检测，添加了对 `SMALLSERIAL` 的渲染支持。

    参考：[#2840](https://www.sqlalchemy.org/trac/ticket/2840)

+   **[postgresql] [bug]**

    删除了对列的服务器默认值的 128 字符截断；此代码最初来自于 PG 系统视图，用于截断字符串以便阅读。

    此更改也已**回溯**至：0.8.3

    参考：[#2844](https://www.sqlalchemy.org/trac/ticket/2844)

+   **[postgresql] [bug]**

    括号将应用于复合 SQL 表达式，如在 CREATE INDEX 语句的列列表中呈现。

    此更改也已**回溯**至：0.8.3

    参考：[#2742](https://www.sqlalchemy.org/trac/ticket/2742)

+   **[postgresql] [bug]**

    修复了一个 bug，其中在“PostgreSQL”或“EnterpriseDB”之前带有前缀的 PostgreSQL 版本字符串无法解析。感谢 Scott Schaefer。

    此更改也已**回溯**至：0.8.3

    参考：[#2819](https://www.sqlalchemy.org/trac/ticket/2819)

+   **[postgresql] [bug]**

    在 PostgreSQL 方言上，`extract()` 的行为已经简化，不再向给定表达式注入硬编码的 `::timestamp` 或类似的转换，因为这会干扰诸如时区感知日期时间之类的类型，但在现代版本的 psycopg2 中似乎也不必要。

    此更改也已**回溯**至：0.8.2

    参考：[#2740](https://www.sqlalchemy.org/trac/ticket/2740)

+   **[postgresql] [bug]**

    修复了 HSTORE 类型中包含反斜杠引号的键/值在使用“非本机”（即非-psycopg2）手段转换 HSTORE 数据时无法正确转义的 bug。补丁由 Ryan Kelly 提供。

    此更改也已**回溯**至：0.8.2

    参考：[#2766](https://www.sqlalchemy.org/trac/ticket/2766)

+   **[postgresql] [bug]**

    修复了一个 bug，其中多列 PostgreSQL 索引中列的顺序会以错误的顺序反映出来。感谢 Roman Podolyaka。

    此更改也已**回溯**至：0.8.2

    参考：[#2767](https://www.sqlalchemy.org/trac/ticket/2767)

### mysql

+   **[mysql] [feature]**

    与`Index`一起使用的`mysql_length`参数现在可以作为列名/长度的字典传递，用于复合索引。非常感谢 Roman Podolyaka 的补丁。

    此更改也已**回溯**至：0.8.2

    参考：[#2704](https://www.sqlalchemy.org/trac/ticket/2704)

+   **[mysql] [feature]**

    MySQL `SET`类型现在具有与`ENUM`相同的自动引号行为。设置值时不需要引号，但存在的引号将被自动检测并发出警告。这也有助于 Alembic，其中 SET 类型不带引号。

    参考：[#2817](https://www.sqlalchemy.org/trac/ticket/2817)

+   **[mysql] [bug]**

    更新 MySQL 保留字版本 5.5、5.6，感谢 Hanno Schlichting。

    此更改也已**回溯**至：0.8.3, 0.7.11

    参考：[#2791](https://www.sqlalchemy.org/trac/ticket/2791)

+   **[mysql] [bug]**

    [#2721](https://www.sqlalchemy.org/trac/ticket/2721)中的更改是，`ForeignKeyConstraint`的`deferrable`关键字在 MySQL 后端上被静默忽略，将在 0.9 版本中恢复；此关键字现在将再次呈现，在 MySQL 上引发错误，因为它不被理解 - 相同的行为也将适用于`initially`关键字。在 0.8 中，这些关键字将继续被忽略，但会发出警告。此外，`match`关键字现在在 0.9 上引发`CompileError`，在 0.8 上发出警告；这个关键字不仅被 MySQL 静默忽略，还会破坏 ON UPDATE/ON DELETE 选项。

    要使用在 MySQL 上不呈现或呈现不同的`ForeignKeyConstraint`，请使用自定义编译选项。文档中已添加了此用法示例，请参阅 MySQL / MariaDB Foreign Keys。

    此更改也已**回溯**至：0.8.3

    参考：[#2721](https://www.sqlalchemy.org/trac/ticket/2721), [#2839](https://www.sqlalchemy.org/trac/ticket/2839)

+   **[mysql] [bug]**

    MySQL 连接器方言现在允许在 create_engine 查询字符串中设置选项来覆盖连接中设置的默认值，包括“buffered”和“raise_on_warnings”。

    此更改也已**回溯**至：0.8.3

    参考：[#2515](https://www.sqlalchemy.org/trac/ticket/2515)

+   **[mysql] [bug]**

    修复了在使用多表 UPDATE 时的 bug，其中一个补充表是带有自己绑定参数的 SELECT，绑定参数的位置与使用 MySQL 的特殊语法时语句本身的位置相反。

    此更改也被**回溯**到：0.8.2

    参考：[#2768](https://www.sqlalchemy.org/trac/ticket/2768)

+   **[mysql] [bug]**

    在`mysql+gaerdbms`方言中添加了另一个条件，以检测所谓的“开发”模式，在这种模式下，我们应该使用`rdbms_mysqldb` DBAPI。修补程序由 Brett Slatkin 提供。

    此更改也被**回溯**到：0.8.2

    参考：[#2715](https://www.sqlalchemy.org/trac/ticket/2715)

+   **[mysql] [bug]**

    在`ForeignKey`和`ForeignKeyConstraint`上的`deferrable`关键字参数不会在 MySQL 方言上呈现`DEFERRABLE`关键字。很长一段时间以来，我们一直保留这个设置，因为一个非延迟的外键会与一个延迟的外键表现得非常不同，但一些环境只是在 MySQL 上禁用 FKs，所以我们在这里会少些主观意见。

    此更改也被**回溯**到：0.8.2

    参考：[#2721](https://www.sqlalchemy.org/trac/ticket/2721)

+   **[mysql] [bug]**

    修复并测试在反射中解析 MySQL 外键选项的问题；这补充了[#2183](https://www.sqlalchemy.org/trac/ticket/2183)中的工作，我们开始支持外键选项的反射，如 ON UPDATE/ON DELETE cascade。

    参考：[#2839](https://www.sqlalchemy.org/trac/ticket/2839)

+   **[mysql] [bug]**

    改进了对 cymysql 驱动程序的支持，支持版本 0.6.5，由 Hajime Nakagami 提供。

### sqlite

+   **[sqlite] [bug]**

    新增的 SQLite DATETIME 参数 storage_format 和 regexp 显然没有完全正确实现；虽然参数被接受，但实际上它们没有任何效果；这个问题已经修复。

    此更改也被**回溯**到：0.8.3

    参考：[#2781](https://www.sqlalchemy.org/trac/ticket/2781)

+   **[sqlite] [bug]**

    将`sqlalchemy.types.BIGINT`添加到 SQLite 方言可以反映的类型名称列表中；由 Russell Stuart 提供。

    此更改也被**回溯**到：0.8.2

    参考：[#2764](https://www.sqlalchemy.org/trac/ticket/2764)

### mssql

+   **[mssql] [bug]**

    在查询 SQL Server 2000 上的信息模式时，删除了在 0.8.1 中添加的一个 CAST 调用，以帮助处理驱动程序问题，显然在 2000 上不兼容。CAST 保留在 SQL Server 2005 及更高版本中。

    此更改也被**回溯**到：0.8.2

    参考：[#2747](https://www.sqlalchemy.org/trac/ticket/2747)

+   **[mssql] [bug] [pyodbc]**

    修复了使用 Python 3 + pyodbc 的 MSSQL，包括正确传递语句。

    参考：[#2355](https://www.sqlalchemy.org/trac/ticket/2355)

### oracle

+   **[oracle] [功能] [py3k]**

    使用 cx_oracle 的 Oracle 单元测试现在在 Python 3 下完全通过。

+   **[oracle] [错误]**

    修复了使用同义词进行 Oracle 表反射时出现的错误，如果同义词和表位于不同的远程模式中，则会失败。修复补丁由 Kyle Derr 提供。

    此更改也**回溯**到：0.8.3

    参考：[#2853](https://www.sqlalchemy.org/trac/ticket/2853)

### 杂项

+   **[功能]**

    添加了一个新标志`system=True`到`Column`，将列标记为数据库自动添加的“系统”列（例如 PostgreSQL 的 `oid` 或 `xmin`）。该列将在`CREATE TABLE`语句中被省略，但仍可用于查询。此外，`CreateColumn` 构造可以应用于自定义编译规则，允许跳过列，通过生成返回`None`的规则。

    此更改也**回溯**到：0.8.3

+   **[功能] [firebird]**

    向 kinterbasdb 和 fdb 方言添加了新标志`retaining=True`。这控制发送到 DBAPI 连接的`commit()`和`rollback()`方法的`retaining`标志的值。由于历史原因，此标志在 0.8.2 中默认为`True`，但在 0.9.0b1 中此标志默认为`False`。

    此更改也**回溯**到：0.8.2

    参考：[#2763](https://www.sqlalchemy.org/trac/ticket/2763)

+   **[功能] [核心]**

    添加了一个新的变体`UpdateBase.returning()`，称为`ValuesBase.return_defaults()`；这允许将任意列添加到语句的 RETURNING 子句中，而不会干扰编译器通常的“隐式返回”功能，该功能用于高效地获取新生成的主键值。对于支持的后端，所有获取的值的字典都存在于`ResultProxy.returned_defaults`中。

    参考：[#2793](https://www.sqlalchemy.org/trac/ticket/2793)

+   **[功能] [池]**

    为“rollback-on-return”和较少使用的“commit-on-return”添加了池记录。这与池“debug”记录一起启用。

    参考：[#2752](https://www.sqlalchemy.org/trac/ticket/2752)

+   **[功能] [firebird]**

    当未指定方言修饰符时，`fdb` 方言现在是默认方言，即 `firebird://`，因为 Firebird 项目将 `fdb` 发布为其官方 Python 驱动程序。

    参考：[#2504](https://www.sqlalchemy.org/trac/ticket/2504)

+   **[错误] [firebird]**

    在反射 Firebird 类型 LONG 和 INT64 时，��型查找已修复，以便将 LONG 视为 INTEGER，INT64 视为 BIGINT，除非类型具有“精度”，在这种情况下，它将被视为 NUMERIC。修复补丁由 Russell Stuart 提供。

    此更改也**回溯**到：0.8.2

    参考：[#2757](https://www.sqlalchemy.org/trac/ticket/2757)

+   **[bug] [ext]**

    修复了一个错误，即如果使用函数而不是类设置复合类型，则当尝试检查该列是否为`MutableComposite`时，可变扩展会出错（它不是）。感谢 asldevi。

    此更改也**回溯**到：0.8.2

+   **[requirements]**

    Python [mock](https://pypi.org/project/mock) 库现在是运行单元测试套件所必需的。虽然作为 Python 3.3 的一部分，但之前的 Python 安装需要安装此库才能运行单元测试或使用`sqlalchemy.testing`包来处理外部方言。

    此更改也**回溯**到：0.8.2

## 0.9.10

发布日期：2015 年 7 月 22 日

### orm

+   **[orm] [feature]**

    在`Query.column_descriptions`返回的字典中添加了一个新条目`"entity"`。这指的是由表达式引用的主 ORM 映射类或别名类。与现有的`"type"`条目相比，它始终是一个映射实体，即使从列表达式中提取，或者如果给定表达式是纯核心表达式，则为`None`。另请参见[#3403](https://www.sqlalchemy.org/trac/ticket/3403)，该修复了此功能中的一个回归，该回归在 0.9.10 中未发布，但在 1.0 版本中发布。

    参考：[#3320](https://www.sqlalchemy.org/trac/ticket/3320)

+   **[orm] [bug]**

    当使用`Query.update()`或`Query.delete()`方法时，`Query`不支持连接、子选择或特殊的 FROM 子句；如果调用了像`Query.join()`或`Query.select_from()`这样的方法，而不是静默地忽略这些字段，会发出警告。从 1.0.0b5 开始，这将引发错误。

    参考：[#3349](https://www.sqlalchemy.org/trac/ticket/3349)

+   **[orm] [bug]**

    修复了在多个嵌套的`Session.begin_nested()`操作中，状态跟踪失败传播“脏”标志的错误，导致在内部保存点中更新过的对象在外部保存点回滚时，该对象不会成为过期状态的一部分，因此不会恢复到其数据库状态。

    参考：[#3352](https://www.sqlalchemy.org/trac/ticket/3352)

### engine

+   **[engine] [bug]**

    将字符串值`"none"`添加到`Pool.reset_on_return` 参数中，作为 `None` 的同义词，以便所有设置都可以使用字符串值，允许像`engine_from_config()` 这样的实用程序可以无问题地使用。

    参考：[#3375](https://www.sqlalchemy.org/trac/ticket/3375)

### sql

+   **[sql] [feature]**

    正式支持在 `Insert.from_select()` 中使用的 SELECT 中存在的 CTE。这种行为在 0.9.9 版本之前偶然起作用，之后由于与 [#3248](https://www.sqlalchemy.org/trac/ticket/3248) 中的不相关更改而不再起作用。请注意，这是在 INSERT 之后、SELECT 之前渲染 WITH 子句的行为；在后续版本中，将针对 INSERT、UPDATE、DELETE 的顶层渲染 CTE 的完整功能作为新功能发布。

    参考：[#3418](https://www.sqlalchemy.org/trac/ticket/3418)

+   **[sql] [bug]**

    修复了一个问题，即使用命名约定的 `MetaData` 对象在 pickle 中无法正常工作。属性被跳过，导致如果从未拣选的 `MetaData` 对象基础上创建其他表，则会出现不一致和失败。

    参考：[#3362](https://www.sqlalchemy.org/trac/ticket/3362)

### postgresql

+   **[postgresql] [bug]**

    修复了一个长期存在的 bug，即在 psycopg2 方言中与非 ascii 值一起使用时，`Enum` 类型与 `native_enum=False` 一起使用时无法正确解码返回结果。这源于 PG `ENUM` 类型曾经是一个独立的类型，没有“非本地”选项。

    参考：[#3354](https://www.sqlalchemy.org/trac/ticket/3354)

### mysql

+   **[mysql] [bug] [pymysql]**

    修复了 PyMySQL 在使用“executemany”操作时对 unicode 参数的支持。现在，SQLAlchemy 将语句和绑定参数都作为 unicode 对象传递，因为 PyMySQL 通常在内部使用字符串插值来生成最终语句，并且在 executemany 情况下仅对最终语句执行“encode”步骤。

    参考：[#3337](https://www.sqlalchemy.org/trac/ticket/3337)

+   **[mysql] [bug] [py3k]**

    修复了 Py3K 上未正确使用 `ord()` 函数的 `BIT` 类型。感谢 David Marin 提交的拉取请求。

    参考：[#3333](https://www.sqlalchemy.org/trac/ticket/3333)

### sqlite

+   **[SQLite] [错误]**

    修复了 SQLite 方言中的错误，其中包含非字母字符（如点或空格）的唯一约束的反射不会反射其名称。

    参考：[#3495](https://www.sqlalchemy.org/trac/ticket/3495)

### 测试

+   **[测试] [错误] [pypy]**

    修复了一个导入问题，阻止了 “pypy setup.py test” 的正确工作。

    参考：[#3406](https://www.sqlalchemy.org/trac/ticket/3406)

### 杂项

+   **[错误] [扩展]**

    修复了使用扩展属性仪器系统时的错误，在调用 `class_mapper()` 时，当使用无效输入（也恰好不是弱引用）时，不会引发正确的异常，例如整数。

    参考：[#3408](https://www.sqlalchemy.org/trac/ticket/3408)

+   **[错误] [扩展]**

    从 0.9.9 版中回归的错误，其中 `as_declarative()` 符号已从 `sqlalchemy.ext.declarative` 命名空间中删除。

    参考：[#3324](https://www.sqlalchemy.org/trac/ticket/3324)

### ORM

+   **[ORM] [特性]**

    向 `Query.column_descriptions` 返回的字典中添加了一个新条目 `"entity"`。它指的是由表达式引用的主要 ORM 映射类或别名类。与现有的 `"type"` 条目相比，它始终是一个映射实体，即使从列表达式中提取，或者如果给定的表达式是一个纯核心表达式，则为 None。另请参阅 [#3403](https://www.sqlalchemy.org/trac/ticket/3403)，修复了该特性的一个回归，该特性未在 0.9.10 版中发布，但在 1.0 版本中发布了。

    参考：[#3320](https://www.sqlalchemy.org/trac/ticket/3320)

+   **[ORM] [错误]**

    `Query` 在使用 `Query.update()` 或 `Query.delete()` 方法时，不支持连接、子查询或特殊的 FROM 子句；如果已调用像 `Query.join()` 或 `Query.select_from()` 这样的方法，而这些字段又不被静默忽略，那么将会发出警告。自 1.0.0b5 版本起，这将会引发错误。

    参考：[#3349](https://www.sqlalchemy.org/trac/ticket/3349)

+   **[ORM] [错误]**

    修复了在多个嵌套的`Session.begin_nested()`操作中状态跟踪失败的 bug，当对象在内部保存点中被更新时，未能传播“脏”标志，因此如果外部保存点被回滚，则对象不会成为已过期状态的一部分，因此不会恢复到其数据库状态。

    参考：[#3352](https://www.sqlalchemy.org/trac/ticket/3352)

### engine

+   **[engine] [bug]**

    将字符串值`"none"`添加到`Pool.reset_on_return`参数接受的值中，作为`None`的同义词，以便所有设置都可以使用字符串值，允许像`engine_from_config()`这样的实用程序可以无问题地使用。

    参考：[#3375](https://www.sqlalchemy.org/trac/ticket/3375)

### sql

+   **[sql] [feature]**

    官方支持在`Insert.from_select()`内部使用的 SELECT 中使用 CTE。此行为在 0.9.9 之前意外工作，当时由于与[#3248](https://www.sqlalchemy.org/trac/ticket/3248)的不相关更改而不再工作。请注意，这是在 INSERT 之后，SELECT 之前呈现 WITH 子句的功能；在 INSERT、UPDATE、DELETE 的顶层呈现 CTE 的完整功能是一个针对以后版本的新功能。

    参考：[#3418](https://www.sqlalchemy.org/trac/ticket/3418)

+   **[sql] [bug]**

    修复了使用命名约定的`MetaData`对象在与 pickle 一起使用时无法正常工作的问题。属性被跳过，导致不一致和失败，如果反序列化的`MetaData`对象用于基于其他表的附加表，则会出现问题。

    参考：[#3362](https://www.sqlalchemy.org/trac/ticket/3362)

### postgresql

+   **[postgresql] [bug]**

    修复了长期存在的 bug，即在与 psycopg2 方言一起使用时，`Enum`类型与非 ascii 值和`native_enum=False`一起使用时无法正确解码返回结果。这源自于 PG `ENUM`类型曾经是一个独立的类型，没有“非本地”选项。

    参考：[#3354](https://www.sqlalchemy.org/trac/ticket/3354)

### mysql

+   **[mysql] [bug] [pymysql]**

    修复了在使用带有 Unicode 参数的“executemany”操作时 PyMySQL 的 Unicode 支持问题。SQLAlchemy 现在将语句和绑定参数都作为 Unicode 对象传递，因为 PyMySQL 通常在内部使用字符串插值来生成最终语句，并且在 executemany 情况下仅在最终语句上执行“encode”步骤。

    参考：[#3337](https://www.sqlalchemy.org/trac/ticket/3337)

+   **[mysql] [bug] [py3k]**

    修复了在 Py3K 上未正确使用 `ord()` 函数的 `BIT` 类型。感谢 David Marin 的拉取请求。

    参考：[#3333](https://www.sqlalchemy.org/trac/ticket/3333)

### sqlite

+   **[sqlite] [bug]**

    修复了 SQLite 方言中的一个 bug，即反射包含非字母字符（如点或空格）的 UNIQUE 约束的名称时，名称不会被反映出来。

    参考：[#3495](https://www.sqlalchemy.org/trac/ticket/3495)

### 测试

+   **[tests] [bug] [pypy]**

    修复了一个导入问题，导致“pypy setup.py test”无法正常工作。

    参考：[#3406](https://www.sqlalchemy.org/trac/ticket/3406)

### 杂项

+   **[bug] [ext]**

    修复了使用扩展属性仪器系统时的 bug，当使用 `class_mapper()` 调用无效输入（也恰好不是弱引用）时，不会引发正确的异常，例如整数。

    参考：[#3408](https://www.sqlalchemy.org/trac/ticket/3408)

+   **[bug] [ext]**

    修复了从 0.9.9 版本开始的回归问题，其中 `as_declarative()` 符号从 `sqlalchemy.ext.declarative` 命名空间中移除。

    参考：[#3324](https://www.sqlalchemy.org/trac/ticket/3324)

## 0.9.9

发布日期：2015 年 3 月 10 日

### orm

+   **[orm] [feature]**

    添加了新参数 `Session.connection.execution_options`，可用于在首次检出连接时设置 `Connection` 的执行选项，即在事务开始之前。这用于在事务开始之前在连接上设置选项，如隔离级别。

    另请参阅

    设置事务隔离级别 / DBAPI AUTOCOMMIT - 新的文档部分详细介绍了使用会话设置事务隔离的最佳实践。

    参考：[#3296](https://www.sqlalchemy.org/trac/ticket/3296)

+   **[orm] [feature]**

    添加了新的方法 `Session.invalidate()`，与 `Session.close()` 的功能类似，但还会调用所有连接的 `Connection.invalidate()` 方法，保证它们不会被返回到连接池中。这在处理 gevent 超时等情况时非常有用，当不安全继续使用连接时，甚至用于回滚时。

+   **[orm] [bug]**

    修复了 ORM 对象比较中的 bug，在比较多对一 `!= None` 时，如果源是一个别名类，或者如果查询需要对表达式应用特殊的别名处理，因为别名连接或多态查询; 还修复了在将多对一与对象状态进行比较时，如果查询需要对别名连接或多态查询应用特殊的别名处理，将失败的情况。

    参考：[#3310](https://www.sqlalchemy.org/trac/ticket/3310)

+   **[orm] [bug]**

    修复了一个 bug，在 `Session` 的 `after_rollback()` 处理程序错误地在处理程序内部向 `Session` 添加状态时，内部断言会失败，而警告和删除此状态的任务（由 [#2389](https://www.sqlalchemy.org/trac/ticket/2389) 设立）试图继续进行时。

    参考：[#3309](https://www.sqlalchemy.org/trac/ticket/3309)

+   **[orm] [bug]**

    修复了一个 bug，在使用未知的 kw 参数调用 `Query.join()` 时，由于格式错误而导致的 TypeError 异常会引发其自身的 TypeError。感谢 Malthe Borch 提供的拉取请求。

+   **[orm] [bug]**

    修复了懒加载 SQL 构造中的 bug，其中一个复杂的主键关联在“指向自身的列”的自引用连接中多次引用相同的“本地”列时，在所有情况下都不会被替换。这里确定替换的逻辑已经重新设计，变得更加开放。

    参考：[#3300](https://www.sqlalchemy.org/trac/ticket/3300)

+   **[orm] [bug]**

    “通配符”加载器选项，特别是由`load_only()`选项设置的选项，用于覆盖未明确提及的所有属性，现在考虑了给定实体的超类，如果该实体使用继承映射进行映射，则超类中的属性名称也将从加载中省略。此外，多态鉴别器列无条件地包含在列表中，就像主键列一样，因此即使设置了 load_only()，子类型的多态加载仍将正常工作。

    参考：[#3287](https://www.sqlalchemy.org/trac/ticket/3287)

+   **[orm] [bug] [pypy]**

    修复了一个错误，即如果在获取结果之前`Query`的开始处抛出异常，特别是当无法形成行处理器时，游标将保持打开状态，结果仍在等待中，并且实际上不会关闭。这通常只在像 PyPy 这样的解释器上出现问题，其中游标不会立即被 GC 回收，并且在某些情况下可能导致事务/锁打开的时间比预期的长。

    参考：[#3285](https://www.sqlalchemy.org/trac/ticket/3285)

+   **[orm] [bug]**

    修复了一个泄漏问题，该问题会在不支持且极不推荐的用例中发生，即在固定映射类上多次替换关系，引用任意增长的目标映射器数量。当替换旧关系时会发出警告，但是如果映射已用于查询，则旧关系仍将在某些注册表中被引用。

    参考：[#3251](https://www.sqlalchemy.org/trac/ticket/3251)

+   **[orm] [bug] [sqlite]**

    修复了关于表达式突变的错误，当使用`Query`从 SQLite 中选择多个匿名列实体进行查询时，可能会表现为“找不到列”错误，这是由 SQLite 方言使用的“连接重写”功能的副作用。

    参考：[#3241](https://www.sqlalchemy.org/trac/ticket/3241)

+   **[orm] [bug]**

    修复了一个错误，即当使用`of_type()`将`Query.join()`和`Query.outerjoin()`连接到单一继承子类时，如果设置了`from_joinpoint=True`标志，则 ON 子句中不会呈现“单表条件”。

    参考：[#3232](https://www.sqlalchemy.org/trac/ticket/3232)

### 示例

+   **[examples] [bug]**

    更新了使用历史表进行版本控制示例，使映射的列重新映射以匹配列名以及列的分组；特别是，这允许在相同列名的联合继承场景中明确分组的列在历史映射中以相同的方式映射，避免了在 0.9 系列中添加的关于此模式的警告，并允许相同的属性键视图。

+   **[examples] [bug]**

    修复了示例中的一个错误，即在 examples/generic_associations/discriminator_on_association.py 示例中，AddressAssociation 的子类未被映射为“单表继承”，导致在尝试进一步使用映射时出现问题。

### engine

+   **[engine] [feature]**

    添加了新的用户空间访问器，用于查看事务隔离级别；`Connection.get_isolation_level()`，`Connection.default_isolation_level`。

+   **[engine] [bug]**

    修复了`Connection`和池中的错误，即如果使用了`Connection.execution_options()`的`isolation_level`参数，则`Connection.invalidate()`方法，或由于数据库断开连接而导致的失效会失败；将在不再打开的连接上调用重置隔离级别的“finalizer”。

    参考：[#3302](https://www.sqlalchemy.org/trac/ticket/3302)

+   **[engine] [bug]**

    如果在播放`Transaction`时使用`isolation_level`参数与`Connection.execution_options()`，则会发出警告；DBAPIs 和/或 SQLAlchemy 方言（如 psycopg2、MySQLdb）可能会隐式回滚或提交事务，或者在下一个事务中不更改设置，因此这永远不安全。

    参考：[#3296](https://www.sqlalchemy.org/trac/ticket/3296)

### sql

+   **[sql] [bug]**

    在`Enum`的`__repr__()`输出中添加了`native_enum`标志，当与 Alembic autogenerate 一起使用时，这一点大多很重要。感谢 Dimitris Theodorou 的拉取请求。

+   **[sql] [bug]**

    修复了一个 bug，当使用实现了也是`TypeDecorator`的类型时，当对使用此类型的对象进行任何类型的 SQL 比较表达式时，会出现 Python 的“无法创建一致的方法解析顺序（MRO）”错误。

    参考：[#3278](https://www.sqlalchemy.org/trac/ticket/3278)

+   **[sql] [bug]**

    修复了一个问题，即当从 SELECT 嵌入到 INSERT 中，无论是通过 values 子句还是作为“from select”，当两个语句的列共享相同的名称时，返回的行中产生的结果集中使用的列类型会受到污染，导致在检索返回的行时可能出现错误或错误适应。

    参考：[#3248](https://www.sqlalchemy.org/trac/ticket/3248)

### schema

+   **[schema] [bug]**

    修复了 0.9 版本的外键设置系统中的一个 bug，即当外键与其父对象进行关联的逻辑在目标`Table`稍后才能接收其父列时，如果目标列实际上具有不同于其名称的键值，例如，在反射中使用列反射事件来更改反射的`Column`对象的.key，以便 link_to_name 变得重要。还修复了当目标列具有不同键并且使用 link_to_name 引用时通过 FK 传输列类型的支持。

    参考：[#1765](https://www.sqlalchemy.org/trac/ticket/1765), [#3298](https://www.sqlalchemy.org/trac/ticket/3298)

### postgresql

+   **[postgresql] [feature]**

    添加了对 PostgreSQL 索引的`CONCURRENTLY`关键字的支持，使用`postgresql_concurrently`建立。感谢 Iuri de Silvio 的拉取请求。

    请参见

    使用 CONCURRENTLY 的索引

+   **[postgresql] [bug]**

    修复了在使用 psycopg2 时，PostgreSQL UUID 类型与 ARRAY 类型一起使用时的支持。psycopg2 方言现在使用 psycopg2.extras.register_uuid()钩子，以便始终将 UUID 值传递到/从 DBAPI 作为 UUID()对象。`UUID.as_uuid`标志仍然受到尊重，除非在 psycopg2 中我们需要在禁用时将返回的 UUID 对象转换回字符串。

    参考：[#2940](https://www.sqlalchemy.org/trac/ticket/2940)

+   **[postgresql] [bug]**

    在使用 psycopg2 2.5.4 或更高版本时，添加了对`postgresql.JSONB`数据类型的支持，该版本具有原生的 JSONB 数据转换，因此必须禁用 SQLAlchemy 的转换器；此外，新增的 psycopg2 扩展`extras.register_default_jsonb`用于通过`json_deserializer`参数传递给方言的 JSON 反序列化器。还修复了实际上未循环传递 JSONB 类型而不是 JSON 类型的 PostgreSQL 集成测试的 bug。感谢 Mateusz Susik 的拉取请求。

+   **[postgresql] [bug]**

    修复了在使用旧版本 psycopg2 < 2.4.3 注册 HSTORE 类型时使用“array_oid”标志的问题，该版本不支持此标志，以及在 psycopg2 版本< 2.5 上使用本机 json 序列化器钩子“register_default_json”与用户定义的`json_deserializer`时的问题，该版本不包括本机 json。

+   **[postgresql] [bug]**

    修复了 PostgreSQL 方言在`Index`中无法呈现表绑定列的表达式的 bug；通常当`text()`构造是索引中的表达式之一时，或者如果其中一个或多个是这样的表达式，则可能会误解表达式列表。

    参考：[#3174](https://www.sqlalchemy.org/trac/ticket/3174)

### mysql

+   **[mysql] [change]**

    `gaerdbms`方言不再必要，并发出弃用警告。Google 现在建议直接使用 MySQLdb 方言。

    参考：[#3275](https://www.sqlalchemy.org/trac/ticket/3275)

+   **[mysql] [bug]**

    在 MySQLdb 方言周围添加了版本检查，用于检查‘utf8_bin’排序规则，因为在 MySQL 服务器< 5.0 上会失败。

    参考：[#3274](https://www.sqlalchemy.org/trac/ticket/3274)

### sqlite

+   **[sqlite] [feature]**

    在 SQLite 上添加了对部分索引（例如带有 WHERE 子句）的支持。感谢 Kai Groner 的拉取请求。

    另请参阅

    部分索引

+   **[sqlite] [feature]**

    添加了一个新的 SQLite 后端，用于 SQLCipher 后端。该后端提供了使用 pysqlcipher Python 驱动程序的加密 SQLite 数据库，该驱动程序与 pysqlite 驱动程序非常相似。

    另请参阅

    `pysqlcipher`

### 杂项

+   **[bug] [ext] [py3k]**

    修复了在 Py3K 下关联代理列表类无法正确解释切片的错误。感谢 Gilles Dartiguelongue 的拉取请求。

### orm

+   **[orm] [feature]**

    添加了新参数 `Session.connection.execution_options`，可用于在首次检出连接时设置连接上的执行选项，事务开始之前。用于在事务开始之前设置连接的选项，如隔离级别。

    另请参阅

    设置事务隔离级别 / DBAPI AUTOCOMMIT - 新的文档部分详细介绍了使用会话设置事务隔离的最佳实践。

    参考：[#3296](https://www.sqlalchemy.org/trac/ticket/3296)

+   **[orm] [feature]**

    添加了新方法 `Session.invalidate()`，类似于 `Session.close()`，但还会在所有连接上调用 `Connection.invalidate()`，确保它们不会返回到连接池。在处理 gevent 超时等情况时非常有用，此时不安全继续使用连接，即使是用于回滚。

+   **[orm] [bug]**

    修复了 ORM 对象比较中的 bug，当比较多对一 `!= None` 时，如果源是一个别名类，或者查询需要对表达式应用特殊别名处理，由于别名连接或多态查询而失败；还修复了比较多对一与对象状态时的 bug，如果查询需要应用特殊别名处理，由于别名连接或多态查询而失败。

    参考：[#3310](https://www.sqlalchemy.org/trac/ticket/3310)

+   **[orm] [bug]**

    修复了一个 bug，在一个 `after_rollback()` 处理程序为一个 `Session` 错误地在处理程序内部向该 `Session` 添加状态时，内部断言会失败的情况下，任务警告并移除此状态（由 [#2389](https://www.sqlalchemy.org/trac/ticket/2389) 确定）尝试继续进行。

    参考：[#3309](https://www.sqlalchemy.org/trac/ticket/3309)

+   **[orm] [bug]**

    修复了一个 bug，当调用 `Query.join()` 时带有未知的关键字参数会引发 TypeError，由于格式错误会导致自身的 TypeError。感谢 Malthe Borch 提交的拉取请求。

+   **[orm] [bug]**

    修复了懒加载 SQL 构造中的 bug，其中复杂的主连接在自引用连接的“指向自身的列”样式中多次引用相同的“本地”列时，在所有情况下都不会被替换。这里确定替换的逻辑已经重新设计为更加开放式。

    参考：[#3300](https://www.sqlalchemy.org/trac/ticket/3300)

+   **[orm] [bug]**

    “通配符”加载器选项，特别是由`load_only()`选项设置的选项，用于覆盖未明确提及的所有属性，现在考虑到给定实体的超类，如果该实体使用继承映射进行映射，则超类中的属性名称也将从加载中省略。此外，多态鉴别器列无条件地包含在列表中，就像主键列一样，因此即使设置了 load_only()，子类型的多态加载仍将正常运行。

    参考：[#3287](https://www.sqlalchemy.org/trac/ticket/3287)

+   **[orm] [bug] [pypy]**

    修复了一个 bug，即如果在`Query`开始获取结果之前抛出异常，特别是当无法形成行处理器时，游标将保持打开状态，结果仍在等待中，实际上并未关闭。这通常只在像 PyPy 这样的解释器上出现问题，其中游标不会立即被 GC 回收，并且在某些情况下可能导致事务/锁定打开时间过长。

    参考：[#3285](https://www.sqlalchemy.org/trac/ticket/3285)

+   **[orm] [bug]**

    修复了在不支持且极不推荐的情况下多次替换固定映射类上的关系时可能发生的泄漏，这些关系指向任意增长的目标映射器数量。在替换旧关系时会发出警告，但如果映射已用于查询，则旧关系仍将在某些注册表中被引用。

    参考：[#3251](https://www.sqlalchemy.org/trac/ticket/3251)

+   **[orm] [bug] [sqlite]**

    修复了关于表达式变异的错误，当使用`Query`从 SQLite 中选择多个匿名列实体进行查询时，可能会表现为“找不到列”错误，这是由 SQLite 方言使用的“联接重写”功能的副作用。

    参考：[#3241](https://www.sqlalchemy.org/trac/ticket/3241)

+   **[orm] [bug]**

    修复了一个 bug，当使用 `of_type()` 将 `Query.join()` 和 `Query.outerjoin()` 到单一继承子类时，如果设置了 `from_joinpoint=True` 标志，则 ON 子句中的“单表条件”不会被渲染。

    参考：[#3232](https://www.sqlalchemy.org/trac/ticket/3232)

### 例子

+   **[例子] [bug]**

    更新了 带有历史表的版本控制 示例，使映射列重新映射以匹配列名以及列的分组；特别是，这允许在同名列的联合继承场景中明确分组的列在历史映射中以相同的方式映射，避免了在 0.9 系列中关于此模式添加的警告，并允许属性键的相同视图。

+   **[例子] [bug]**

    修复了 examples/generic_associations/discriminator_on_association.py 示例中的 bug，其中 AddressAssociation 的子类未被映射为“单表继承”，导致在进一步使用映射时出现问题。

### 引擎

+   **[引擎] [功能]**

    添加了用于查看事务隔离级别的新用户空间访问器；`Connection.get_isolation_level()`，`Connection.default_isolation_level`。

+   **[引擎] [bug]**

    修复了 `Connection` 和池中的 bug，当使用 `Connection.invalidate()` 方法，或由于数据库断开连接而导致失效时，如果使用了 `isolation_level` 参数与 `Connection.execution_options()` 一起使用，则会失败；重置隔离级别的“finalizer”将在不再打开的连接上调用。

    参考：[#3302](https://www.sqlalchemy.org/trac/ticket/3302)

+   **[引擎] [bug]**

    如果在使用 `Transaction` 时使用了 `isolation_level` 参数与 `Connection.execution_options()`，则会发出警告；DBAPIs 和/或 SQLAlchemy 方言（如 psycopg2、MySQLdb）可能会隐式回滚或提交事务，或者在下一个事务中不更改设置，因此这永远不安全。

    参考：[#3296](https://www.sqlalchemy.org/trac/ticket/3296)

### sql

+   **[sql] [错误]**

    在`Enum`的`__repr__()`输出中添加了`native_enum`标志，当与 Alembic autogenerate 一起使用时，这在大多数情况下是重要的。感谢 Dimitris Theodorou 的拉取请求。

+   **[sql] [错误]**

    修复了使用实现了也是`TypeDecorator`的类型的`TypeDecorator`会在针对使用此类型的对象使用任何类型的 SQL 比较表达式时导致 Python 的“无法创建一致的方法解析顺序（MRO）”错误的错误。

    参考：[#3278](https://www.sqlalchemy.org/trac/ticket/3278)

+   **[sql] [错误]**

    修复了在 INSERT 语句中嵌套的 SELECT 中的列，无论是通过 values 子句还是作为“from select”，都会污染由 RETURNING 子句生成的结果集中使用的列类型的问题，当两个语句的列共享相同名称时，可能导致检索返回行时出现潜在错误或适应不良。

    参考：[#3248](https://www.sqlalchemy.org/trac/ticket/3248)

### 模式

+   **[模式] [错误]**

    修复了 0.9 版本中外键设置系统中的错误，即当外键与其父表使用“link_to_name=True”关联时，如果目标表直到后来才接收到其父列，例如在反射+“useexisting”场景中，如果目标列实际上具有与其名称不同的键值，那么用于将`ForeignKey`链接到其父表的逻辑可能会失败，例如在反射中使用列反射事件来更改反射的`Column`对象的.key，以便 link_to_name 变得重要。还修复了当目标列具有不同键并且使用 link_to_name 引用时，通过 FK 传输支持列类型的方式。

    参考：[#1765](https://www.sqlalchemy.org/trac/ticket/1765), [#3298](https://www.sqlalchemy.org/trac/ticket/3298)

### postgresql

+   **[postgresql] [功能]**

    添加了对使用`CONCURRENTLY`关键字在 PostgreSQL 索引上建立的支持，使用`postgresql_concurrently`进行建立。感谢 Iuri de Silvio 的拉取请求。

    另请参阅

    并发索引

+   **[postgresql] [错误]**

    修复了在使用 psycopg2 时与 ARRAY 类型一起支持 PostgreSQL UUID 类型的问题。现在，psycopg2 方言使用 psycopg2.extras.register_uuid() 钩子，以便始终将 UUID 值作为 UUID() 对象传递给/从 DBAPI。`UUID.as_uuid` 标志仍然受到尊重，但是对于 psycopg2，当禁用此标志时，我们需要将返回的 UUID 对象转换回字符串。

    参考：[#2940](https://www.sqlalchemy.org/trac/ticket/2940)

+   **[postgresql] [bug]**

    在使用 psycopg2 2.5.4 或更高版本时，为 `postgresql.JSONB` 数据类型添加了支持，该支持具有原生的 JSONB 数据转换，因此必须禁用 SQLAlchemy 的转换器；此外，新添加的 psycopg2 扩展 `extras.register_default_jsonb` 用于建立传递给方言的 JSON 反序列化器，通过 `json_deserializer` 参数。还修复了 PostgreSQL 集成测试，这些测试实际上并没有循环传输 JSONB 类型，而不是 JSON 类型。感谢 Mateusz Susik 提交的拉取请求。

+   **[postgresql] [bug]**

    修复了在使用旧版本 psycopg2 < 2.4.3 时注册 HSTORE 类型时使用 “array_oid” 标志的问题，该标志不支持此标志，以及在使用旧版本 psycopg2 < 2.5 时使用本机 json 序列化器钩子 “register_default_json” 与用户定义的 `json_deserializer` 时的问题，该版本不包括本机 json。

+   **[postgresql] [bug]**

    修复了 PostgreSQL 方言在 `Index` 中无法正确渲染表绑定列之外的表达式的 bug；通常情况下，当 `text()` 构造是索引中的表达式之一时；或者如果其中一个或多个表达式是这样的表达式，则可能会误解表达式列表。

    参考：[#3174](https://www.sqlalchemy.org/trac/ticket/3174)

### mysql

+   **[mysql] [change]**

    `gaerdbms` 方言不再必要，并发出弃用警告。Google 现在建议直接使用 MySQLdb 方言。

    参考：[#3275](https://www.sqlalchemy.org/trac/ticket/3275)

+   **[mysql] [bug]**

    在 MySQLdb 方言周围添加了一个版本检查，用于检查 ‘utf8_bin’ 校对，因为这在 MySQL 服务器 < 5.0 上失败。

    参考：[#3274](https://www.sqlalchemy.org/trac/ticket/3274)

### sqlite

+   **[sqlite] [feature]**

    在 SQLite 上添加了对部分索引（例如带有 WHERE 子句）的支持。感谢 Kai Groner 提交的拉取请求。

    另请参阅

    部分索引

+   **[sqlite] [feature]**

    添加了一个新的 SQLite 后端用于 SQLCipher 后端。该后端使用 pysqlcipher Python 驱动程序提供加密的 SQLite 数据库，该驱动程序与 pysqlite 驱动程序非常相似。

    另请参阅

    `pysqlcipher`

### 杂项

+   **[bug] [ext] [py3k]**

    修复了在 Py3K 下关联代理列表类无法正确解释切片的 bug。感谢 Gilles Dartiguelongue 提供的拉取请求。

## 0.9.8

发布日期：2014 年 10 月 13 日

### orm

+   **[orm] [bug] [engine]**

    修复了一个 bug，通常会影响与[#3199](https://www.sqlalchemy.org/trac/ticket/3199)相同类型的事件，当`named=True`参数被使用时。一些事件将无法注册，其他事件将不正确地调用事件参数，通常是在事件被“包装”以适应其他方式时。已重新排列“命名”机制，以不干扰内部包装函数预期的参数签名。

    参考：[#3197](https://www.sqlalchemy.org/trac/ticket/3197)

+   **[orm] [bug]**

    修复了一个 bug，影响了许多事件类，特别是 ORM 事件，但也包括引擎事件，其中对于通常的“去重”一个冗余调用`listen()`与相同参数的逻辑失败，对于那些被包装的监听函数会命中 registry.py 中的一个断言。现在，这个断言已经被集成到去重检查中，附加了一个检查去重的更简单的方式。

    参考：[#3199](https://www.sqlalchemy.org/trac/ticket/3199)

+   **[orm] [bug]**

    修复了一个警告，当一个复杂的自引用主要连接包含函数时，同时指定了 remote_side，该警告将被发出；警告将建议设置“远程方”。现在，只有在 remote_side 不存在时才发出。

    参考：[#3194](https://www.sqlalchemy.org/trac/ticket/3194)

### orm 声明式

+   **[orm] [declarative] [bug]**

    使用`AbstractConcreteBase`与声明`__abstract__`的子类时，修复了“‘NoneType’ object has no attribute ‘concrete’”错误。

    参考：[#3185](https://www.sqlalchemy.org/trac/ticket/3185)

### 引擎

+   **[engine] [bug]**

    通过`create_engine.execution_options`或`Engine.update_execution_options()`传递给`Engine`的执行选项不会传递给用于在“第一次连接”事件中初始化方言的特殊`Connection`；方言通常会在此阶段执行自己的查询，并且当前可用的选项都不应该应用在这里。特别是，“autocommit”选项导致在这个初始连接中尝试自动提交，这将由于`Connection`的非标准状态而导致 AttributeError。

    参考：[#3200](https://www.sqlalchemy.org/trac/ticket/3200)

+   **[engine] [bug]**

    用于确定 INSERT 或 UPDATE 受影响列的字符串键现在在对“编译缓存”缓存键的贡献时进行排序。这些键以前没有确定性地排序，这意味着相同的语句可能根据等效键被多次缓存，这既在内存方面��在性能方面造成了损失。

    参考：[#3165](https://www.sqlalchemy.org/trac/ticket/3165)

### sql

+   **[sql] [bug]**

    修复了 sql 包中许多 SQL 元素无法成功`__repr__()`的错误，由于缺少`description`属性，导致内部 AttributeError 再次调用`__repr__()`时递归溢出。

    参考：[#3195](https://www.sqlalchemy.org/trac/ticket/3195)

+   **[sql] [bug]**

    调整表/索引反射，如果索引报告一个在表中找不到的列，将发出警告并跳过该列。这可能发生在一些特殊的系统列情况下，正如在 Oracle 中观察到的情况。

    参考：[#3180](https://www.sqlalchemy.org/trac/ticket/3180)

+   **[sql] [bug]**

    修复了 CTE 中的错误，其中当一个 CTE 引用语句中的另一个别名 CTE 时，`literal_binds`编译器参数不会始终正确传播。

    参考：[#3154](https://www.sqlalchemy.org/trac/ticket/3154)

+   **[sql] [bug]**

    由于[#3067](https://www.sqlalchemy.org/trac/ticket/3067)引起的 0.9.7 版本回归问题，再加上一个命名错误的单元测试，导致所谓的“模式”类型如`Boolean`和`Enum`无法再被序列化。

    参考：[#3067](https://www.sqlalchemy.org/trac/ticket/3067)，[#3144](https://www.sqlalchemy.org/trac/ticket/3144)

### postgresql

+   **[postgresql] [feature] [pg8000]**

    添加了对 pg8000 驱动程序的“合理的多行计数”支持，这主要适用于在 ORM 中使用版本控制的情况。该功能是基于使用 pg8000 1.9.14 或更高版本的版本进行检测的。Pull request 由 Tony Locke 提供。

+   **[postgresql] [bug]**

    对于在 0.9.5 中首次修补的此问题进行了重新审视，显然 psycopg2 的`.closed`访问器不像我们假设的那样可靠，因此我们已经添加了对异常消息“SSL SYSCALL error: Bad file descriptor”和“SSL SYSCALL error: EOF detected”的显式检查，以检测到断开连接的情况。我们将继续首先检查 psycopg2 的 connection.closed。

    参考文献：[#3021](https://www.sqlalchemy.org/trac/ticket/3021)

+   **[postgresql] [bug]**

    修复了 PostgreSQL JSON 类型无法持久化或以其他方式呈现 SQL NULL 列值的错误，而不是 JSON 编码的`'null'`。为支持此案例，更改如下：

    +   现在可以指定值 `null()`，这将始终导致语句中的 NULL 值。

    +   添加了一个新参数 `JSON.none_as_null`，当为 True 时表示 Python 的`None`值应该被持久化为 SQL NULL，而不是 JSON 编码的`'null'`。

    对于除了 psycopg2 外的其他 DBAPI，即 pg8000，检索 NULL 作为 None 也已修复。

    参考文献：[#3159](https://www.sqlalchemy.org/trac/ticket/3159)

+   **[postgresql] [bug]**

    现在，DBAPI 错误的异常包装系统可以容纳非标准的 DBAPI 异常，例如 psycopg2 的 TransactionRollbackError。这些异常现在将使用`sqlalchemy.exc`中最接近的可用子类引发，在 TransactionRollbackError 的情况下，是`sqlalchemy.exc.OperationalError`。

    参考文献：[#3075](https://www.sqlalchemy.org/trac/ticket/3075)

+   **[postgresql] [bug]**

    修复了 `array` 对象中的错误，其中与普通的 Python 列表进行比较将无法使用正确的数组构造函数。Pull request 由 Andrew 提供。

    参考文献：[#3141](https://www.sqlalchemy.org/trac/ticket/3141)

+   **[postgresql] [bug]**

    添加了对函数的支持 `FunctionElement.alias()` 方法，例如，`func` 构造。之前，此方法的行为是未定义的。当前行为模仿了 0.9.4 之前的行为，即将函数转换为具有给定别名的单列 FROM 子句，其中列本身被匿名命名。

    参考文献：[#3137](https://www.sqlalchemy.org/trac/ticket/3137)

### mysql

+   **[mysql] [bug] [mysqlconnector]**

    从版本 2.0 开始的 Mysqlconnector，可能是由于 Python 3 合并的副作用，现在不再期望百分号（例如用作模运算符和其他操作符）加倍，即使使用“pyformat”绑定参数格式（Mysqlconnector 未记录此更改）。方言现在在检测模运算符应该呈现为`%%`还是`%`时，检查 py2k 和 mysqlconnector 小于版本 2.0。

+   **[mysql] [bug] [mysqlconnector]**

    现在对于 MySQLconnector 版本 2.0 及以上，Unicode SQL 已传递；对于 Py2k 和 MySQL < 2.0，字符串被编码。

### sqlite

+   **[sqlite] [bug]**

    当使用附加数据库文件从 UNION 中进行选择时，pysqlite 驱动程序将列名在 cursor.description 中报告为‘dbname.tablename.colname’，而不是正常情况下的‘tablename.colname’（请注意，对于 UNION，它应该只是‘colname’，但我们对此进行了处理）。此处的列翻译逻辑已调整为检索最右边的标记，而不是第二个标记，因此在两种情况下都有效。感谢 Tony Roberts 的解决方法。

    参考：[#3211](https://www.sqlalchemy.org/trac/ticket/3211)

### mssql

+   **[mssql] [bug]**

    修复了在 pymssql 方言中检测版本字符串时与 Microsoft SQL Azure 一起使用的问题，后者将“SQL Server”更改为“SQL Azure”。

    参考：[#3151](https://www.sqlalchemy.org/trac/ticket/3151)

### oracle

+   **[oracle] [bug]**

    修复了 Oracle 方言中长期存在的 bug，即以数字开头的绑定参数名称不会被引用，因为 Oracle 不喜欢绑定参数名称中的数字。

    参考：[#2138](https://www.sqlalchemy.org/trac/ticket/2138)

### 杂项

+   **[bug] [declarative]**

    修复了在一些奇特的最终用户设置中观察到的不太可能的竞争条件，在这种情况下，尝试在声明中检查“重复类名”会遇到与另一个被移除的类相关的未完全清理的弱引用；此处的检查现在确保弱引用在进一步调用之前仍然引用一个对象。

    参考：[#3208](https://www.sqlalchemy.org/trac/ticket/3208)

+   **[bug] [ext]**

    修复了在集合替换事件期间，如果 reorder_on_append 标志设置为 True，则项目顺序会被打乱的排序列表中的 bug。修复确保排序列表仅影响与对象明确关联的列表。

    参考：[#3191](https://www.sqlalchemy.org/trac/ticket/3191)

+   **[bug] [ext]**

    修复了`MutableDict`未实现`update()`字典方法的 bug，因此无法捕捉更改。感谢 Matt Chisholm 的拉取请求。

+   **[bug] [ext]**

    修复了一个 bug，在其中一个自定义 `MutableDict` 的子类在“强制转换”操作中不会显示，并且会返回一个普通的 `MutableDict`。感谢 Matt Chisholm 提供的拉取请求。

+   **[错误] [池]**

    修复了连接池日志记录中的一个 bug，即“连接检出”调试日志消息如果使用 `logging.setLevel()` 设置日志记录而不是使用 `echo_pool` 标志则不会发出。已添加用于断言此日志记录的测试。这是在 0.9.0 中引入的一个回归。

    参考：[#3168](https://www.sqlalchemy.org/trac/ticket/3168)

### orm

+   **[orm] [错误] [引擎]**

    修复了一个影响基本与 [#3199](https://www.sqlalchemy.org/trac/ticket/3199) 相同的事件类别的 bug，当使用 `named=True` 参数时。一些事件将无法注册，其他事件将不会正确调用事件参数，通常在事件被“包装”以适应其他方式时。已重新排列“命名”机制，以不干扰内部包装函数期望的参数签名。

    参考：[#3197](https://www.sqlalchemy.org/trac/ticket/3197)

+   **[orm] [错误]**

    修复了一个影响多种事件类别的 bug，特别是 ORM 事件，但也包括引擎事件，即使用相同参数对 `listen()` 进行“去重”的常规逻辑失败的情况，对于那些监听器函数被包装的事件。在 registry.py 中会触发一个断言。现在，这个断言已经集成到去重检查中，并额外增加了一种更简单的检查去重的方法。

    参考：[#3199](https://www.sqlalchemy.org/trac/ticket/3199)

+   **[orm] [错误]**

    修复了一个警告，当一个复杂的自引用 primaryjoin 包含函数时，并且同时指定了 remote_side 时，警告会建议设置“remote side”。现在只有在 remote_side 不存在时才会发出。

    参考：[#3194](https://www.sqlalchemy.org/trac/ticket/3194)

### orm 声明

+   **[orm] [声明] [错误]**

    修复了在与声明 `__abstract__` 的子类一起使用 `AbstractConcreteBase` 时出现“‘NoneType’ 对象没有 ‘concrete’ 属性”的错误。

    参考：[#3185](https://www.sqlalchemy.org/trac/ticket/3185)

### 引擎

+   **[引擎] [错误]**

    通过`create_engine.execution_options`或`Engine.update_execution_options()`传递给`Engine`的执行选项不会传递给用于在“第一次连接”事件中初始化方言的特殊`Connection`；方言通常会在此阶段执行自己的查询，并且当前可用的选项都不应该应用于此处。特别是，“autocommit”选项导致在此初始连接中尝试自动提交，这将由于`Connection`的非标准状态而导致 AttributeError 而失败。

    参考：[#3200](https://www.sqlalchemy.org/trac/ticket/3200)

+   **[engine] [bug]**

    用于确定 INSERT 或 UPDATE 受影响列的字符串键现在在对“编译缓存”缓存键做出贡献时进行排序。这些键以前没有确定性地排序，这意味着相同的语句可能会根据等效键被多次缓存，这既会在内存方面也会在性能方面造成损失。

    参考：[#3165](https://www.sqlalchemy.org/trac/ticket/3165)

### sql

+   **[sql] [bug]**

    修复了 sql 包中许多 SQL 元素无法成功`__repr__()`的错误，这是由于缺少`description`属性导致的，然后在内部 AttributeError 时再次调用`__repr__()`会引发递归溢出。

    参考：[#3195](https://www.sqlalchemy.org/trac/ticket/3195)

+   **[sql] [bug]**

    调整表/索引反射，如果索引报告一个在表中找不到的列，则会发出警告并跳过该列。这可能发生在一些特殊的系统列情况下，正如在 Oracle 中观察到的情况。

    参考：[#3180](https://www.sqlalchemy.org/trac/ticket/3180)

+   **[sql] [bug]**

    修复了 CTE 中的错误，其中当一个 CTE 引用语另一个别名 CTE 时，`literal_binds`编译器参数不会始终正确传播。

    参考：[#3154](https://www.sqlalchemy.org/trac/ticket/3154)

+   **[sql] [bug]**

    修复了 0.9.7 中由[#3067](https://www.sqlalchemy.org/trac/ticket/3067)引起的回归，与一个命名错误的单元测试一起导致所谓的“模式”类型如`Boolean`和`Enum`无法再被 pickle。

    参考：[#3067](https://www.sqlalchemy.org/trac/ticket/3067)，[#3144](https://www.sqlalchemy.org/trac/ticket/3144)

### postgresql

+   **[postgresql] [feature] [pg8000]**

    使用 pg8000 驱动程序添加了“sane multi row count”支持，这主要适用于在 ORM 中使用版本控制时。该功能基于使用 pg8000 1.9.14 或更高版本进行版本检测。拉取请求由 Tony Locke 提供。

+   **[postgresql] [bug]**

    重新访问了在 0.9.5 中首次修补的问题，显然 psycopg2 的`.closed`访问器并不像我们所假设的那样可靠，因此我们已经添加了一个显式检查异常消息“SSL SYSCALL error: Bad file descriptor”和“SSL SYSCALL error: EOF detected”以检测断开连接的情况。我们将继续将 psycopg2 的 connection.closed 作为第一次检查。

    参考：[#3021](https://www.sqlalchemy.org/trac/ticket/3021)

+   **[postgresql] [bug]**

    修复了 PostgreSQL JSON 类型无法持久化或以其他方式呈现 SQL NULL 列值而不是 JSON 编码的`'null'`的错误。为支持此情况，更改如下：

    +   现在可以指定值`null()`，这将始终导致语句中的 NULL 值。

    +   新参数`JSON.none_as_null`被添加，当为 True 时表示 Python 的`None`值应该被持久化为 SQL NULL，而不是 JSON 编码的`'null'`。

    对于除了 psycopg2 之外的其他 DBAPI，如 pg8000，将 NULL 检索为 None 也已修复。

    参考：[#3159](https://www.sqlalchemy.org/trac/ticket/3159)

+   **[postgresql] [bug]**

    DBAPI 错误的异常包装系统现在可以容纳非标准的 DBAPI 异常，例如 psycopg2 的 TransactionRollbackError。这些异常现在将使用`sqlalchemy.exc`中最接近的可用子类引发，在 TransactionRollbackError 的情况下，是`sqlalchemy.exc.OperationalError`。

    参考：[#3075](https://www.sqlalchemy.org/trac/ticket/3075)

+   **[postgresql] [bug]**

    修复了`array`对象中的��误，其中与普通 Python 列表的比较将无法使用正确的数组构造函数。拉取请求由 Andrew 提供。

    参考：[#3141](https://www.sqlalchemy.org/trac/ticket/3141)

+   **[postgresql] [bug]**

    为函数添加了支持的`FunctionElement.alias()`方法，例如`func`构造。先前，此方法的行为未定义。当前行为模仿了 0.9.4 之前的行为，即将函数转换为具有给定别名的单列 FROM 子句，其中列本身是匿名命名的。

    参考：[#3137](https://www.sqlalchemy.org/trac/ticket/3137)

### mysql

+   **[mysql] [bug] [mysqlconnector]**

    Mysqlconnector 从版本 2.0 开始，可能是由于 Python 3 合并的副作用，现在不再期望百分号（例如用作模运算符和其他操作符）被加倍，即使使用“pyformat”绑定参数格式（这一变化未在 Mysqlconnector 中记录）。方言现在在检测模运算符应该呈现为`%%`还是`%`时检查 py2k 和 mysqlconnector 小于版本 2.0。

+   **[mysql] [bug] [mysqlconnector]**

    现在对于 MySQLconnector 版本 2.0 及以上版本传递 Unicode SQL；对于 Py2k 和 MySQL < 2.0，字符串被编码。

### sqlite

+   **[sqlite] [bug]**

    当从使用附加数据库文件的 UNION 中进行选择时，pysqlite 驱动程序将列名报告为‘dbname.tablename.colname’，而不是正常情况下对于 UNION 的‘tablename.colname’（请注意，对于两者都应该只是‘colname’，但我们对此进行了处理）。这里的列翻译逻辑已经调整为检索最右边的标记，而不是第二个标记，因此在两种情况下都有效。感谢 Tony Roberts 的解决方法。

    参考：[#3211](https://www.sqlalchemy.org/trac/ticket/3211)

### mssql

+   **[mssql] [bug]**

    修复了 pymssql 方言中版本字符串检测的 bug，以便与 Microsoft SQL Azure 一起使用，该数据库将“SQL Server”更改为“SQL Azure”。

    参考：[#3151](https://www.sqlalchemy.org/trac/ticket/3151)

### oracle

+   **[oracle] [bug]**

    修复了 Oracle 方言中长期存在的 bug，即以数字开头的绑定参数名称不会被引用，因为 Oracle 不喜欢绑定参数名称中的数字。

    参考：[#2138](https://www.sqlalchemy.org/trac/ticket/2138)

### 杂项

+   **[bug] [declarative]**

    修复了在一些奇特的最终用户设置中观察到的不太可能的竞争条件 bug，在这种情况下，在声明中检查“重复类名”时会遇到一个与其他被移除的类相关的未完全清理的弱引用；这里的检查现在确保在进一步调用之前弱引用仍然引用一个对象。

    参考：[#3208](https://www.sqlalchemy.org/trac/ticket/3208)

+   **[bug] [ext]**

    修复了在排序列表中的 bug，如果将 reorder_on_append 标志设置为 True，则在集合替换事件期间项目的顺序会被打乱。修复确保排序列表仅影响与对象明确关联的列表。

    参考：[#3191](https://www.sqlalchemy.org/trac/ticket/3191)

+   **[bug] [ext]**

    修复了`MutableDict`在实现`update()`字典方法时失败的 bug，因此无法捕捉更改。感谢 Matt Chisholm 的拉取请求。

+   **[bug] [ext]**

    修复了自定义`MutableDict`的子类不会在“强制转换”操作中显示，并且会返回一个普通的`MutableDict`的错误。感谢 Matt Chisholm 提供的拉取请求。

+   **[错误] [连接池]**

    修复了连接池日志中的错误，即“连接已检出”调试日志消息如果使用`logging.setLevel()`设置日志记录，而不是使用`echo_pool`标志，则不会发出。已添加了用于断言此日志记录的测试。这是在 0.9.0 中引入的退化。

    参考：[#3168](https://www.sqlalchemy.org/trac/ticket/3168)

## 0.9.7

发布日期：2014 年 7 月 22 日

### orm

+   **[orm] [错误] [贪婪加载]**

    由 0.9.4 中释放的[#2976](https://www.sqlalchemy.org/trac/ticket/2976)引起的退化修复，在一系列加入的贪婪加载中，“外部连接”传播会错误地将兄弟加入路径上的“内部连接”也转换为外部连接，当只有后代路径应该接收“外部连接”传播时；另外，修复了“嵌套”加入传播在两个兄弟加入路径之间不适当地进行的相关问题。

    参考：[#3131](https://www.sqlalchemy.org/trac/ticket/3131)

+   **[orm] [错误]**

    由于 0.9.0 中的[#2736](https://www.sqlalchemy.org/trac/ticket/2736)引起的退化修复，`Query.select_from()`方法不再正确设置`Query`对象的“from entity”，因此后续的`Query.filter_by()`或`Query.join()`调用将无法在按字符串名称搜索属性时检查适当的“from”实体而失败。

    参考：[#2736](https://www.sqlalchemy.org/trac/ticket/2736)，[#3083](https://www.sqlalchemy.org/trac/ticket/3083)

+   **[orm] [错误]**

    查询.update()/delete()的“评估器”在多表更新中不起作用，并且需要设置为 synchronize_session=False 或 synchronize_session=’fetch’；现在会发出警告。在 1.0 中，这将升级为完整的异常。

    参考：[#3117](https://www.sqlalchemy.org/trac/ticket/3117)

+   **[orm] [错误]**

    修复了在保存点块内持久化、删除或主键更改的项目在外部事务回滚后无法参与恢复到其先前状态（不在会话中，在会话中，先前的 PK）的错误。

    参考：[#3108](https://www.sqlalchemy.org/trac/ticket/3108)

+   **[orm] [错误]**

    修复了在与`with_polymorphic()`一起使用子查询预加载时的 bug，对子查询加载中实体和列的定位已经针对这种类型的实体和其他实体更加准确。

    参考：[#3106](https://www.sqlalchemy.org/trac/ticket/3106)

+   **[orm] [bug]**

    修复了动态属性中的 bug，这又是版本 0.9.5 的 [#3060](https://www.sqlalchemy.org/trac/ticket/3060) 的一个回归。具有 lazy=’dynamic’ 的自引用关系会在刷新操作中引发 TypeError。

    参考：[#3099](https://www.sqlalchemy.org/trac/ticket/3099)

### 引擎

+   **[引擎] [特性]**

    添加了新的事件`ConnectionEvents.handle_error()`，这是对`ConnectionEvents.dbapi_error()`的更全面和全面的替代。

    参考：[#3076](https://www.sqlalchemy.org/trac/ticket/3076)

### sql

+   **[sql] [bug]**

    修复了在`Enum`和其他`SchemaType`子类中的 bug，在这些类型直接关联到`MetaData`时，当在`MetaData`上发出事件（如创建事件）时会导致 hang。

    这个变更也被**回溯**到了：0.8.7

    参考：[#3124](https://www.sqlalchemy.org/trac/ticket/3124)

+   **[sql] [bug]**

    修复了在自定义操作符加法`TypeEngine.with_variant()`系统中的 bug，当与 variant 一起使用 `TypeDecorator`时，在使用比较运算符时会导致 MRO 错误。

    这个变更也被**回溯**到了：0.8.7

    参考：[#3102](https://www.sqlalchemy.org/trac/ticket/3102)

+   **[sql] [bug]**

    修复了命名约定特性中的 bug，在使用包含 `constraint_name` 的检查约定时，将强制所有`Boolean`和`Enum`类型也需要名称，因为这些隐式创建约束，即使最终目标后端不需要生成约束，比如 PostgreSQL。这些特定约束的命名约定机制已经重新组织，使得命名确定在 DDL 编译时进行，而不是在约束/表构造时进行。

    参考：[#3067](https://www.sqlalchemy.org/trac/ticket/3067)

+   **[sql] [bug]**

    修复了公共表达式中的错误，当 CTE 嵌套在某些方式中时，位置绑定参数可能以错误的最终顺序表示。

    参考：[#3090](https://www.sqlalchemy.org/trac/ticket/3090)

+   **[sql] [错误]**

    修复了多值 `Insert` 构造中的错误，该构造在字面 SQL 表达式的第一个给定值之后未能检查后续值条目。

    参考：[#3069](https://www.sqlalchemy.org/trac/ticket/3069)

+   **[sql] [错误]**

    在 Python 版本 < 2.6.5 中为 dialect_kwargs 迭代添加了一个“str()”步骤，解决了“无 unicode 关键字参数”错误，因为这些参数在某些反射过程中作为关键字参数传递。

    参考：[#3123](https://www.sqlalchemy.org/trac/ticket/3123)

+   **[sql] [错误]**

    `TypeEngine.with_variant()` 方法现在将接受一个类型类作为参数，该参数在内部转换为一个实例，使用了其他构造（如 `Column`）长期建立的相同约定。

    参考：[#3122](https://www.sqlalchemy.org/trac/ticket/3122)

### postgresql

+   **[postgresql] [功能]**

    在 `ColumnOperators.match()` 操作符中添加了 kw 参数 `postgresql_regconfig`，允许指定“reg config”参数以发出到 `to_tsquery()` 函数。感谢 Jonathan Vanasco 的拉取请求。

    参考：[#3078](https://www.sqlalchemy.org/trac/ticket/3078)

+   **[postgresql] [功能]**

    通过 `JSONB` 添加了对 PostgreSQL JSONB 的支持。感谢 Damian Dimmich 的拉取请求。

+   **[postgresql] [错误] [pg8000]**

    修复了 0.9.5 版本中由新的 pg8000 隔离级别功能引入的错误，其中引擎级别的隔离级别参数在连接时会引发错误。

    参考：[#3134](https://www.sqlalchemy.org/trac/ticket/3134)

### mysql

+   **[mysql] [错误]**

    MySQL 错误 2014 “commands out of sync” 似乎在现代 MySQL-Python 版本中被提升为 ProgrammingError，而不是 OperationalError；现在在 OperationalError 和 ProgrammingError 中检查了所有被测试为“is disconnect”的 MySQL 错误代码。

    此更改也被**回溯**到：0.8.7

    参考：[#3101](https://www.sqlalchemy.org/trac/ticket/3101)

### sqlite

+   **[sqlite] [错误]**

    修复了 SQLite 连接重写问题，其中作为标量子查询嵌入的子查询（例如在 IN 中）会从包含查询中接收不适当的替换，如果相同的表在子查询中存在，并且在包含查询中也存在，例如在连接继承场景中。 

    参考：[#3130](https://www.sqlalchemy.org/trac/ticket/3130)

### mssql

+   **[mssql] [feature]**

    为 SQL Server 2008 启用了“多值插入”。感谢 Albert Cervin 的贡献。还扩展了“IDENTITY INSERT”模式的检查，以包括当标识键出现在语句的 VALUEs 子句中时。

+   **[mssql] [bug]**

    将语句编码添加到“SET IDENTITY_INSERT”语句中，当在 IDENTITY 列中插入显式 INSERT 时，以支持在不支持 unicode 语句的驱动程序（如 pyodbc + unix + py2k）上使用非 ASCII 表标识符。

    此更改也被**回溯**到：0.8.7

+   **[mssql] [bug]**

    在 SQL Server pyodbc 方言中，修复了 `description_encoding` 方言参数的实现，当未显式设置时，会导致无法正确解析包含其他编码名称的结果集的 cursor.description。未来不应该需要此参数。

    此更改也被**回溯**到：0.8.7

    参考：[#3091](https://www.sqlalchemy.org/trac/ticket/3091)

+   **[mssql] [bug]**

    修复了从 0.9.5 中由 [#3025](https://www.sqlalchemy.org/trac/ticket/3025) 引起的回归，其中用于确定“默认模式”的查询在 SQL Server 2000 中无效。对于 SQL Server 2000，我们回到了默认为 ‘dbo’ 的“模式名称”参数。可配置但默认为 ‘dbo’。 

    参考：[#3025](https://www.sqlalchemy.org/trac/ticket/3025)

### oracle

+   **[oracle] [bug] [tests]**

    修复了 oracle 方言测试套件中的 bug，在一个测试中，假定 ‘username’ 在数据库 URL 中，即使这可能并非事实。

    参考：[#3128](https://www.sqlalchemy.org/trac/ticket/3128)

### tests

+   **[tests] [bug]**

    修复了“python setup.py test”未适当调用 distutils，导致在测试套件结束时会发出错误。

### misc

+   **[bug] [declarative]**

    修复了当声明性 `__abstract__` 标志未被区分为实际值为 `False` 时的 bug。`__abstract__` 标志需要在被测试的级别实际评估为 True 值。

    参考：[#3097](https://www.sqlalchemy.org/trac/ticket/3097)

### orm

+   **[orm] [bug] [eagerloading]**

    修复了在 0.9.4 中由 [#2976](https://www.sqlalchemy.org/trac/ticket/2976) 引起的回归，其中沿着一系列连接的“外连接”传播会错误地将一个“内连接”转换为外连接，当只有后代路径应该接收“外连接”传播时；此外，修复了相关问题，即“嵌套”连接传播会不适当地发生在两个兄弟连接路径之间。

    参考：[#3131](https://www.sqlalchemy.org/trac/ticket/3131)

+   **[orm] [bug]**

    由于 [#2736](https://www.sqlalchemy.org/trac/ticket/2736) 导致的从 0.9.0 开始的回归修复了一个 bug，`Query.select_from()` 方法不再正确设置 `Query` 对象的“from 实体”，因此，后续的 `Query.filter_by()` 或 `Query.join()` 调用将无法在按字符串名称搜索属性时检查适当的“from”实体。

    参考：[#2736](https://www.sqlalchemy.org/trac/ticket/2736)，[#3083](https://www.sqlalchemy.org/trac/ticket/3083)

+   **[orm] [bug]**

    用于 query.update()/delete() 的“evaluator”在多表更新中不起作用，需要设置为 synchronize_session=False 或 synchronize_session='fetch'；现在会发出警告。在 1.0 版本中，这将提升为完整的异常。

    参考：[#3117](https://www.sqlalchemy.org/trac/ticket/3117)

+   **[orm] [bug]**

    修复了在 savepoint 块中持久化、删除或主键更改的项目在外部事务回滚后无法参与恢复到其先前状态（不在会话中，在会话中，先前的 PK）的 bug。

    参考：[#3108](https://www.sqlalchemy.org/trac/ticket/3108)

+   **[orm] [bug]**

    在与 `with_polymorphic()` 结合使用的子查询提前加载中修复了 bug，加载子查询中实体和列的目标对于此类实体和其他实体更加准确。

    参考：[#3106](https://www.sqlalchemy.org/trac/ticket/3106)

+   **[orm] [bug]**

    修复了涉及动态属性的 bug，这是从版本 0.9.5 重新出现的 [#3060](https://www.sqlalchemy.org/trac/ticket/3060) 的一个回归。带有 lazy='dynamic' 的自引用关系在刷新操作中会引发 TypeError。

    参考：[#3099](https://www.sqlalchemy.org/trac/ticket/3099)

### engine

+   **[engine] [feature]**

    添加了新的事件 `ConnectionEvents.handle_error()`，这是 `ConnectionEvents.dbapi_error()` 的更全面和全面的替代品。

    参考：[#3076](https://www.sqlalchemy.org/trac/ticket/3076)

### sql

+   **[sql] [bug]**

    修复了 `Enum` 和其他 `SchemaType` 子类中的 bug，在直接将类型与 `MetaData` 关联时，当在 `MetaData` 上发出事件（如创建事件）时，会导致挂起。

    此更改还**回溯到**：0.8.7

    引用：[#3124](https://www.sqlalchemy.org/trac/ticket/3124)

+   **[sql] [bug]**

    修复了在自定义运算符加法 `TypeEngine.with_variant()` 系统中的一个错误，当与变体一起使用时，使用 `TypeDecorator` 会在使用比较运算符时出现 MRO 错误。

    此更改还**回溯到**：0.8.7

    引用：[#3102](https://www.sqlalchemy.org/trac/ticket/3102)

+   **[sql] [bug]**

    修复了命名约定功能中的错误，在使用包含 `constraint_name` 的检查约定时，会强制所有 `Boolean` 和 `Enum` 类型也需要名称，因为这些隐式地创建了约束，即使最终的目标后端不需要生成约束，比如 PostgreSQL。这些特定约束的命名约定机制已经重新组织，以便在 DDL 编译时进行命名确定，而不是在约束/表构造时。

    引用：[#3067](https://www.sqlalchemy.org/trac/ticket/3067)

+   **[sql] [bug]**

    修复了通用表达式中的错误，在某些方式下嵌套 CTEs 时，位置绑定参数可能以错误的最终顺序表示。

    引用：[#3090](https://www.sqlalchemy.org/trac/ticket/3090)

+   **[sql] [bug]**

    修复了多值 `Insert` 构造中的错误，使其在给定 SQL 表达式的第一个值之外检查后续值条目时不会失败。

    引用：[#3069](https://www.sqlalchemy.org/trac/ticket/3069)

+   **[sql] [bug]**

    为 Python 版本 < 2.6.5 的 dialect_kwargs 迭代添加了一个“str()”步骤，解决了“无 Unicode 关键字参数”错误，因为这些参数会在某些反射过程中作为关键字参数传递。

    引用：[#3123](https://www.sqlalchemy.org/trac/ticket/3123)

+   **[sql] [bug]**

    `TypeEngine.with_variant()` 方法现在将接受一个类型类作为参数，该参数在内部转换为一个实例，使用了其他构造（例如 `Column`）长期建立的相同约定。

    引用：[#3122](https://www.sqlalchemy.org/trac/ticket/3122)

### postgresql

+   **[postgresql] [feature]**

    将 kw 参数 `postgresql_regconfig` 添加到 `ColumnOperators.match()` 操作符，允许指定“reg config”参数以发出到 `to_tsquery()` 函数。拉取请求由 Jonathan Vanasco 提供。

    参考：[#3078](https://www.sqlalchemy.org/trac/ticket/3078)

+   **[postgresql] [feature]**

    通过 `JSONB` 添加了对 PostgreSQL JSONB 的支持。拉取请求由 Damian Dimmich 提供。

+   **[postgresql] [bug] [pg8000]**

    修复了在 0.9.5 中由新的 pg8000 隔离级别功能引入的错误，其中引擎级别的隔离级别参数在连接时会引发错误。

    参考：[#3134](https://www.sqlalchemy.org/trac/ticket/3134)

### mysql

+   **[mysql] [bug]**

    MySQL 错误 2014 “commands out of sync” 似乎在现代 MySQL-Python 版本中被提升为 ProgrammingError，而不是 OperationalError；现在在 OperationalError 和 ProgrammingError 中都检查了所有被测试为“is disconnect”的 MySQL 错误代码。

    此更改也**回溯**到：0.8.7

    参考：[#3101](https://www.sqlalchemy.org/trac/ticket/3101)

### sqlite

+   **[sqlite] [bug]**

    修复了 SQLite 连接重写问题，其中嵌入作为标量子查询的子查询（例如在 IN 中）会从包含查询中接收不适当的替换，如果相同的表在子查询中存在，并且在包含查询中也存在，例如在连接继承场景中。

    参考：[#3130](https://www.sqlalchemy.org/trac/ticket/3130)

### mssql

+   **[mssql] [feature]**

    为 SQL Server 2008 启用了“multivalues insert”。拉取请求由 Albert Cervin 提供。还扩展了“IDENTITY INSERT”模式的检查，以包括当标识键出现在语句的 VALUEs 子句中时。

+   **[mssql] [bug]**

    将语句编码添加到“SET IDENTITY_INSERT”语句中，当在 IDENTITY 列中插入显式 INSERT 时进行操作，以支持在不支持 unicode 语句的驱动程序（如 pyodbc + unix + py2k）上的非 ascii 表标识符。

    此更改也**回溯**到：0.8.7

+   **[mssql] [bug]**

    在 SQL Server pyodbc 方言中，修复了 `description_encoding` 方言参数的实现，当未明确设置时，会阻止正确解析包含不同编码名称的结果集的 cursor.description。未来不应该需要此参数。

    此更改也**回溯**到：0.8.7

    参考：[#3091](https://www.sqlalchemy.org/trac/ticket/3091)

+   **[mssql] [bug]**

    由[#3025](https://www.sqlalchemy.org/trac/ticket/3025)引起的 0.9.5 中的回归错误已修复，用于确定“默认模式”的查询在 SQL Server 2000 中无效。对于 SQL Server 2000，我们回到默认为‘dbo’的 dialect 的“模式名称”参数。

    参考：[#3025](https://www.sqlalchemy.org/trac/ticket/3025)

### oracle

+   **[oracle] [bug] [tests]**

    修复了 oracle 方言测试套件中的一个 bug，在一个测试中，假定‘username’在数据库 URL 中，尽管这可能并非事实。

    参考：[#3128](https://www.sqlalchemy.org/trac/ticket/3128)

### 测试

+   **[tests] [bug]**

    修复了“python setup.py test”未适当调用 distutils 的 bug，测试套件结束时会发出错误。

### 杂项

+   **[bug] [declarative]**

    修复了当声明式`__abstract__`标志未被区分为实际值`False`时的 bug。`__abstract__`标志需要在被测试的级别上实际评估为 True 值。

    参考：[#3097](https://www.sqlalchemy.org/trac/ticket/3097)

## 0.9.6

发布日期：2014 年 6 月 23 日

### orm

+   **[orm] [bug]**

    撤销了[#3060](https://www.sqlalchemy.org/trac/ticket/3060)的更改 - 这是一个工作单元修复，在 1.0 中更全面地更新为[#3061](https://www.sqlalchemy.org/trac/ticket/3061)。[#3060](https://www.sqlalchemy.org/trac/ticket/3060)中的修复不幸地产生了一个新问题，即一个多对一属性的急加载可能会产生一个被解释为属性更改的事件。

    参考：[#3060](https://www.sqlalchemy.org/trac/ticket/3060)

### orm

+   **[orm] [bug]**

    撤销了[#3060](https://www.sqlalchemy.org/trac/ticket/3060)的更改 - 这是一个工作单元修复，在 1.0 中更全面地更新为[#3061](https://www.sqlalchemy.org/trac/ticket/3061)。[#3060](https://www.sqlalchemy.org/trac/ticket/3060)中的修复不幸地产生了一个新问题，即一个多对一属性的急加载可能会产生一个被解释为属性更改的事件。

    参考：[#3060](https://www.sqlalchemy.org/trac/ticket/3060)

## 0.9.5

发布日期：2014 年 6 月 23 日

### orm

+   **[orm] [feature]**

    “primaryjoin”模型已经进一步扩展，允许一个连接条件严格地从单个列到自身，通过某种 SQL 函数或表达式进行转换。这有点实验性质，但第一个概念验证是一个“材料化路径”连接条件，其中一个路径字符串与自身使用“like”进行比较。`ColumnOperators.like()` 操作符也已添加到可在 primaryjoin 条件中使用的有效操作符列表中。

    参考：[#3029](https://www.sqlalchemy.org/trac/ticket/3029)

+   **[orm] [feature]**

    添加了新的实用函数`make_transient_to_detached()`，可用于制造行为就像它们从会话中加载然后分离的对象。不存在的属性被标记为过期，并且对象可以添加到一个会话中，其中它将表现得像一个持久对象。

    参考：[#3017](https://www.sqlalchemy.org/trac/ticket/3017)

+   **[orm] [bug]**

    修复了子查询急加载中的一个 bug，在多态子类边界上的一长串急加载与多态加载一起会无法定位链中的子类链接，导致在`AliasedClass`上出现缺少属性名称的错误。

    此更改也**回溯**到：0.8.7

    参考：[#3055](https://www.sqlalchemy.org/trac/ticket/3055)

+   **[orm] [bug]**

    修复了 ORM 中的一个 bug，`class_mapper()`函数会掩盖应该在映射器配置期间由于用户错误而引发的 AttributeErrors 或 KeyErrors。对于属性/键错误的捕获已经更具体，不包括配置步骤。

    此更改也**回溯**到：0.8.7

    参考：[#3047](https://www.sqlalchemy.org/trac/ticket/3047)

+   **[orm] [bug]**

    对于继承映射器隐式组合其基于列的属性之一与父级的情况，已添加了额外的检查，其中这些列通常不一定共享相同的值。这是通过[#1892](https://www.sqlalchemy.org/trac/ticket/1892)添加的现有检查的扩展；然而，这个新检查只发出警告，而不是异常，以允许依赖于现有行为的应用程序。

    另请参见

    我收到关于“隐式组合列 X 在属性 Y 下”的警告或错误

    参考：[#3042](https://www.sqlalchemy.org/trac/ticket/3042)

+   **[orm] [bug]**

    修改了`load_only()`的行为，使得主键列始终添加到“未延迟加载”列的列表中；否则，ORM 无法加载行的标识。显然，可以延迟映射的主键，ORM 将失败，这一点没有改变。但是，由于 load_only 本质上是说“除了 X 之外都延迟加载”，因此 PK 列不参与此延迟加载更为关键。

    参考：[#3080](https://www.sqlalchemy.org/trac/ticket/3080)

+   **[orm] [bug]**

    修复了在所谓的“行切换”场景中出现的一些边缘情况，其中 INSERT/DELETE 可以转换为 UPDATE。在这种情况下，将一对多关系设置为 None，或在某些情况下将标量属性设置为 None，可能不会被检测为值的净变化，因此 UPDATE 不会重置前一行上的内容。这是由于属性历史的一些尚未解决的副作用，这些副作用在隐式假定 None 对于先前未设置的属性实际上不是一个“变化”。另请参见[#3061](https://www.sqlalchemy.org/trac/ticket/3061)。

    注意

    这个更改已在 0.9.6 版本中**撤销**。完整修复将在 SQLAlchemy 的 1.0 版本中实现。

    参考：[#3060](https://www.sqlalchemy.org/trac/ticket/3060)

+   **[orm] [bug]**

    与[#3060](https://www.sqlalchemy.org/trac/ticket/3060)相关，对工作单元进行了调整，以便对相关的一对多对象进行加载时更加积极，例如在要删除的自引用对象图中；加载相关对象有助于确定删除顺序的正确顺序，如果未设置`passive_deletes`。

+   **[orm] [bug]**

    修复了 SQLite 连接重写中的错误，由于重复导致的匿名列名不会在子查询中正确重写。这会影响带有任何类型子查询 + 连接的 SELECT 查询。

    参考：[#3057](https://www.sqlalchemy.org/trac/ticket/3057)

+   **[orm] [bug] [sql]**

    修复了在[#2804](https://www.sqlalchemy.org/trac/ticket/2804)中新增的布尔强制转换中的问题，新规则对“where”和“having”不会对`select()`构造函数的“whereclause”和“having”关键字参数生效，这也是`Query`使用的内容，所以在 ORM 中也无法正常工作。

    参考：[#3013](https://www.sqlalchemy.org/trac/ticket/3013)

### 例子

+   **[examples] [feature]**

    添加了一个新示例，演示了使用最新关系特性的物化路径。示例由 Jack Zhou 提供。

### 引擎

+   **[engine] [bug]**

    修复了一个错误，当引擎首次连接并进行初始检查时发生 DBAPI 异常，并且异常不是断开连接异常，但是当我们尝试关闭游标时游标引发错误。在这种情况下，由于我们试图通过连接池记录游标关闭异常并失败，因为我们试图以不适当的方式访问池的记录器来记录真正的异常，所以真正的异常会被压制。

    参考：[#3063](https://www.sqlalchemy.org/trac/ticket/3063)

+   **[engine] [bug]**

    修复了一些“双失效”情况，检测到连接失效可能发生在已经是关键部分的情况下，比如连接关闭(); 最终，这些条件是由 [#2907](https://www.sqlalchemy.org/trac/ticket/2907) 中的变化引起的，在这个变化中，“返回时重置”功能调用连接/事务来处理它，在那里可能会被“断开检测”捕获。然而，可能最近在 [#2985](https://www.sqlalchemy.org/trac/ticket/2985) 中的更改使得这更有可能被视为“连接失效”操作更快，因为问题在 0.9.4 上更容易重现，而不是 0.9.3。

    现在在可能发生失效的任何部分内都添加了检查，以阻止在失效的连接上进一步的不允许的操作。这包括两个修复，一个在引擎级别，一个在池级别。虽然这个问题是在高并发的 gevent 情况下观察到的，但理论上它可能发生在任何断开连接的情况下。

    参考：[#3043](https://www.sqlalchemy.org/trac/ticket/3043)

### sql

+   **[sql] [功能]**

    在 `Index` 的合同中稍微放宽了一点，如果索引将手动添加到表中，则可以将 `text()` 表达式指定为目标；索引不再需要有表绑定的列存在。

    参考：[#3028](https://www.sqlalchemy.org/trac/ticket/3028)

+   **[sql] [功能]**

    添加了新的标志 `between.symmetric`，当设置为 True 时，渲染为“BETWEEN SYMMETRIC”。还添加了一个新的否定操作符“notbetween_op”，现在允许一个表达式像 `~col.between(x, y)` 这样渲染为“col NOT BETWEEN x AND y”，而不是带括号的 NOT 字符串。

    参考：[#2990](https://www.sqlalchemy.org/trac/ticket/2990)

+   **[sql] [bug]**

    修复了 INSERT..FROM SELECT 构造中的错误，其中从 UNION 中选择会将联合包装在一个匿名的子查询中。

    此更改也被**回溯**到：0.8.7

    参考：[#3044](https://www.sqlalchemy.org/trac/ticket/3044)

+   **[sql] [bug]**

    修复了当应用空的 `and_()` 或 `or_()` 或其他空白表达式时，`Table.update()` 和 `Table.delete()` 会生成一个空的 WHERE 子句的 bug。现在这与 `select()` 的行为一致。

    此更改也被 **回溯** 到：0.8.7

    参考：[#3045](https://www.sqlalchemy.org/trac/ticket/3045)

+   **[sql] [bug]**

    当该表中的 `Column` 在显式的 `PrimaryKeyConstraint` 中被引用时，`Column.nullable` 标志会被隐式设置为 `False`。这种行为现在与 `Column` 本身具有 `Column.primary_key` 标志设置为 `True` 时的行为相匹配，这意味着这是一个完全等价的情况。

    参考：[#3023](https://www.sqlalchemy.org/trac/ticket/3023)

+   **[sql] [bug]**

    修复了一个 bug，即在自定义 `Comparator` 实现中无法重写 `Operators.__and__()`、`Operators.__or__()` 和 `Operators.__invert__()` 运算符重载方法。

    参考：[#3012](https://www.sqlalchemy.org/trac/ticket/3012)

+   **[sql] [bug]**

    修复了新的 `DialectKWArgs.argument_for()` 方法中的 bug，其中为以前未包含任何特殊参数的构造添加参数将失败。

    参考：[#3024](https://www.sqlalchemy.org/trac/ticket/3024)

+   **[sql] [bug]**

    修复了在 0.9 版本中引入的回归，新的“ORDER BY <labelname>”功能从 [#1068](https://www.sqlalchemy.org/trac/ticket/1068) 中不会将标签名称作为 ORDER BY 中呈现的引用规则应用。

    参考：[#1068](https://www.sqlalchemy.org/trac/ticket/1068)，[#3020](https://www.sqlalchemy.org/trac/ticket/3020)

+   **[sql] [bug]**

    恢复了`Function`的导入到`sqlalchemy.sql.expression`导入命名空间，该导入在 0.9 版本初期被移除。

### postgresql

+   **[postgresql] [feature]**

    在使用 pg8000 DBAPI 时，添加了对 AUTOCOMMIT 隔离级别的支持。感谢 Tony Locke 提交的拉取请求。

+   **[postgresql] [feature]**

    为 PostgreSQL `ARRAY` 类型添加了一个新标志`ARRAY.zero_indexes`。当设置为`True`时，在传递到数据库之前，将为所有数组索引值添加一个值，从而更好地实现 Python 风格的从零开始索引和 PostgreSQL 从一开始索引之间的互操作性。感谢 Alexey Terentev 提交的拉取请求。

    参考：[#2785](https://www.sqlalchemy.org/trac/ticket/2785)

+   **[postgresql] [bug]**

    为 PG `HSTORE` 类型添加了`hashable=False`标志，这是为了允许 ORM 在请求混合列/实体列表中的 ORM 映射的 HSTORE 列时跳过尝试“哈希” ORM 的需求。感谢 Gunnlaugur Þór Briem 提交的补丁。

    此更改也**回溯**到：0.8.7

    参考：[#3053](https://www.sqlalchemy.org/trac/ticket/3053)

+   **[postgresql] [bug]**

    添加了一个新的“断开连接”消息“连接意外关闭”。这似乎与较新版本的 SSL 有关。感谢 Antti Haapala 提交的拉取请求。

    此更改也**回溯**到：0.8.7

+   **[postgresql] [bug]**

    当确定异常是否为“断开连接”错误时，现在会查看 psycopg2 的`.closed`访问器；理想情况下，这应该消除对异常消息的任何其他检查来检测断开连接的需要，但我们将保留这些现有消息作为备用。这应该能够处理新的情况，如“SSL EOF”条件。感谢 Dirk Mueller 提交的拉取请求。

    参考：[#3021](https://www.sqlalchemy.org/trac/ticket/3021)

+   **[postgresql] [enhancement]**

    向 PostgreSQL 方言添加了一个新类型`OID`。虽然“oid”通常是 PG 中的一个私有类型，在现代版本中不会暴露，但在一些 PG 使用情况下，如大对象支持，这些类型可能会被暴露，以及在一些用户报告的模式反射使用情况中。

    参考：[#3002](https://www.sqlalchemy.org/trac/ticket/3002)

### mysql

+   **[mysql] [bug]**

    修复了一个错误，在索引上将列名添加到`mysql_length`参数时，需要对引号名进行相同的引号以识别。修复使引号变为可选，但也提供了旧行为以向后兼容使用该解决方法的人。

    此更改也**被回溯**至：0.8.7

    参考：[#3085](https://www.sqlalchemy.org/trac/ticket/3085)

+   **[mysql] [bug]**

    添加了对反映包含 KEY_BLOCK_SIZE 的索引的表的支持，使用等号。感谢 Sean McGivern 提供的 Pull 请求。

    此更改也**被回溯**至：0.8.7

### mssql

+   **[mssql] [bug]**

    修正了用于确定当前默认模式名称的查询，以使用`database_principal_id()`函数与`sys.database_principals`视图相结合，以便我们可以独立于正在进行的登录类型（例如 SQL Server，Windows 等）确定默认模式。

    参考：[#3025](https://www.sqlalchemy.org/trac/ticket/3025)

### 测试

+   **[tests] [bug] [py3k]**

    修正了一些关于在运行测试时涉及`imp`模块和 Python 3.3 或更高版本的弃用警告。感谢 Matt Chisholm 提供的 Pull 请求。

    参考：[#2830](https://www.sqlalchemy.org/trac/ticket/2830)

### 杂项

+   **[bug] [declarative]**

    在访问时，从声明混合或抽象类复制`__mapper_args__`字典，以便声明本身对该字典进行的修改不会与其他映射冲突。该字典在`version_id_col`和`polymorphic_on`参数方面进行修改，将其内的列替换为正式映射到本地类/表的列。

    此更改也**被回溯**至：0.8.7

    参考：[#3062](https://www.sqlalchemy.org/trac/ticket/3062)

+   **[bug] [ext]**

    修复了一个在可变扩展中的错误，其中`MutableDict`未对`setdefault()`字典操作报告更改事件。

    此更改也**被回溯**至：0.8.7

    参考：[#3051](https://www.sqlalchemy.org/trac/ticket/3051)，[#3093](https://www.sqlalchemy.org/trac/ticket/3093)

+   **[bug] [ext]**

    修复了一个错误，其中`MutableDict.setdefault()`未返回现有值或新值（此错误未在任何 0.8 版本中发布）。感谢 Thomas Hervé提供的 Pull 请求。

    此更改也**被回溯**至：0.8.7

    参考：[#3051](https://www.sqlalchemy.org/trac/ticket/3051)，[#3093](https://www.sqlalchemy.org/trac/ticket/3093)

+   **[bug] [testsuite]**

    在公共测试套件中，将`StringTest.test_literal_backslashes`中不受支持的`Text`更改为使用`String(40)`。感谢 Jan 提供的 Pull 请求。

+   **[bug] [firebird]**

    修复了一个 bug，即“limit”渲染为“SELECT FIRST n ROWS”并使用绑定参数（只有 firebird 同时具有两者），与列级子查询结合使用，该子查询也具有“limit”以及“位置”绑定参数（例如，qmark 样式），会错误地在封闭 SELECT 之前分配子查询级别的位置，从而返回顺序错误的参数。

    参考：[#3038](https://www.sqlalchemy.org/trac/ticket/3038)

### orm

+   **[orm] [feature]**

    “primaryjoin”模型已进一步扩展，以允许严格从单个列到自身的连接条件，通过某种 SQL 函数或表达式进行转换。这有点实验性，但第一个概念验证是“materialized path”连接条件，其中路径字符串与自身进行“like”比较。`ColumnOperators.like()`操作符也已添加到可在 primaryjoin 条件中使用的有效操作符列表中。

    参考：[#3029](https://www.sqlalchemy.org/trac/ticket/3029)

+   **[orm] [feature]**

    添加了新的实用函数`make_transient_to_detached()`，可用于制造行为像从会话加载然后分离的对象。不存在的属性被标记为过期，对象可以添加到会话中，其中它将像持久对象一样工作。

    参考：[#3017](https://www.sqlalchemy.org/trac/ticket/3017)

+   **[orm] [bug]**

    修复了子查询贪婪加载中的一个 bug，其中在与多态子类边界上的长贪婪加载链与多态加载相结合时，将无法定位链中的子类链接，导致在`AliasedClass`上的丢失属性名称错误。

    此更改也已**回溯**到：0.8.7

    参考：[#3055](https://www.sqlalchemy.org/trac/ticket/3055)

+   **[orm] [bug]**

    修复了 ORM 中的一个 bug，即`class_mapper()`函数会掩盖应该在映射器配置期间引发的 AttributeErrors 或 KeyErrors，原因是用户错误。针对属性/关键错误的捕获已经更具体，不包括配置步骤。

    此更改也已**回溯**到：0.8.7

    参考：[#3047](https://www.sqlalchemy.org/trac/ticket/3047)

+   **[orm] [bug]**

    已添加额外的检查，用于处理继承映射器隐式组合其基于列的属性之一与父级属性的情况，其中这些列通常不一定共享相同的值。这是通过[#1892](https://www.sqlalchemy.org/trac/ticket/1892)添加的现有检查的扩展；但是，这个新检查只发出警告，而不是异常，以允许可能依赖现有行为的应用程序。

    另请参见

    我收到关于“在属性 Y 下隐式组合列 X”警告或错误

    参考：[#3042](https://www.sqlalchemy.org/trac/ticket/3042)

+   **[orm] [bug]**

    修改了`load_only()`的行为，使得主键列始终添加到“未延迟加载”列的列表中；否则，ORM 无法加载行的标识。显然，可以延迟映射的主键，ORM 将失败，这一点没有改变。但是，由于 load_only 本质上是说“除了 X 之外都延迟加载”，因此 PK 列不参与此延迟更为关键。

    参考：[#3080](https://www.sqlalchemy.org/trac/ticket/3080)

+   **[orm] [bug]**

    修复了在所谓的“行切换”场景中出现的一些边缘情况，其中 INSERT/DELETE 可以转换为 UPDATE。在这种情况下，将一个多对一关系设置为 None，或者在某些情况下将标量属性设置为 None，可能不会被检测为值的净变化，因此 UPDATE 不会重置前一行上的内容。这是由于属性历史的一些尚未解决的副作用，这些副作用假定 None 对于先前未设置的属性实际上不是真正的“变化”。另请参见[#3061](https://www.sqlalchemy.org/trac/ticket/3061)。

    注意

    这个更改已在 0.9.6 版本中**撤销**。完整的修复将在 SQLAlchemy 的 1.0 版本中实现。

    参考：[#3060](https://www.sqlalchemy.org/trac/ticket/3060)

+   **[orm] [bug]**

    与[#3060](https://www.sqlalchemy.org/trac/ticket/3060)相关，对工作单元进行了调整，以便在要删除的自引用对象图的情况下，与相关的多对一对象的加载更为积极；相关对象的加载有助于确定删除顺序的正确顺序，如果未设置 passive_deletes。

+   **[orm] [bug]**

    修复了 SQLite 连接重写中的错误，其中由于重复导致的匿名列名不会在子查询中正确重写。这将影响带有任何类型的子查询 + 连接的 SELECT 查询。

    参考：[#3057](https://www.sqlalchemy.org/trac/ticket/3057)

+   **[orm] [bug] [sql]**

    修复了[#2804](https://www.sqlalchemy.org/trac/ticket/2804)中新增的布尔强制转换，在“where”和“having”上的新规则不会对`select()`构造函数的“whereclause”和“having”关键字参数生效，这也是`Query`使用的内容，因此在 ORM 中也无法正常工作。

    参考：[#3013](https://www.sqlalchemy.org/trac/ticket/3013)

### 示例

+   **[示例] [特性]**

    添加了一个新的示例，演示了使用最新的关系特性的物化路径。示例由 Jack Zhou 提供。

### 引擎

+   **[引擎] [错误]**

    修复了一个 bug，当引擎首次连接并进行初始检查时发生 DBAPI 异常，且异常不是断开连接异常，但当我们尝试关闭游标时，游标引发错误。在这种情况下，真正的异常会被压制，因为我们尝试通过连接池记录游标关闭异常并失败，因为我们试图以不适合在这种非常特定情况下访问池的记录器的方式。

    参考：[#3063](https://www.sqlalchemy.org/trac/ticket/3063)

+   **[引擎] [错误]**

    修复了一些“双重无效”情况，其中连接无效可能发生在已经处于关键部分的情况下，比如连接关闭(); 最终，这些条件是由于[#2907](https://www.sqlalchemy.org/trac/ticket/2907)中的更改引起的，因为“返回时重置”功能调用 Connection/Transaction 来处理它，其中可能会被捕获“断开连接检测”。然而，最近在[#2985](https://www.sqlalchemy.org/trac/ticket/2985)中的更改可能使这种情况更容易被视为“连接无效”操作更快，因为在 0.9.4 上更容易复现这个问题，而在 0.9.3 上不太容易。

    现在在可能发生无效操作的任何部分中添加检查以阻止对无效连接进行进一步的不允许操作。这包括引擎级别和池级别的两个修复。虽然这个问题在高度并发的 gevent 情况下被观察到，但理论上在任何发生连接关闭操作中断开连接的情况下都可能发生。

    参考：[#3043](https://www.sqlalchemy.org/trac/ticket/3043)

### sql

+   **[sql] [特性]**

    在`Index`的合同中稍微放宽了一下，您可以将`text()`表达式指定为目标；如果要手动将索引添加到表中，则索引不再需要存在表绑定列，无论是通过内联声明还是通过`Table.append_constraint()`。

    参考：[#3028](https://www.sqlalchemy.org/trac/ticket/3028)

+   **[sql] [功能]**

    添加了新的标志`between.symmetric`，当设置为 True 时，渲染为 “BETWEEN SYMMETRIC”。还添加了一个新的否定运算符“notbetween_op”，现在允许表达式如`~col.between(x, y)`渲染为 “col NOT BETWEEN x AND y”，而不是一个带括号的 NOT 字符串。

    参考：[#2990](https://www.sqlalchemy.org/trac/ticket/2990)

+   **[sql] [错误]**

    修复了在 INSERT..FROM SELECT 结构中的 bug，在此结构中从 UNION 中进行选择会将 UNION 包装在一个匿名（例如无标签）子查询中。

    这个更改也被**回溯**到：0.8.7

    参考：[#3044](https://www.sqlalchemy.org/trac/ticket/3044)

+   **[sql] [错误]**

    修复了`Table.update()`和`Table.delete()`在应用空的`and_()`、`or_()`或其他空表达式时产生空的 WHERE 子句的 bug。现在这与`select()`的行为一致了。

    这个更改也被**回溯**到：0.8.7

    参考：[#3045](https://www.sqlalchemy.org/trac/ticket/3045)

+   **[sql] [错误]**

    当在表的明确`PrimaryKeyConstraint`中引用 `Column` 时，`Column.nullable` 标志隐式设置为 `False`。这个行为现在与 `Column` 本身具有 `Column.primary_key` 标志设置为 `True` 时的行为相匹配，这是一个完全等效的情况。

    参考：[#3023](https://www.sqlalchemy.org/trac/ticket/3023)

+   **[sql] [错误]**

    修复了一个 bug，即`Operators.__and__()`、`Operators.__or__()`和`Operators.__invert__()`操作符重载方法无法在自定义`Comparator`实现中被覆盖。

    引用：[#3012](https://www.sqlalchemy.org/trac/ticket/3012)

+   **[sql] [bug]**

    修复了新的`DialectKWArgs.argument_for()`方法中的 bug，其中为以前未包含任何特殊参数的构造添加参数将失败。

    引用：[#3024](https://www.sqlalchemy.org/trac/ticket/3024)

+   **[sql] [bug]**

    修复了在 0.9 版本中引入的回归问题，即新的“ORDER BY <labelname>”功能从[#1068](https://www.sqlalchemy.org/trac/ticket/1068)中不会将标签名称的引用规则应用于在 ORDER BY 中呈现的标签名称。

    引用：[#1068](https://www.sqlalchemy.org/trac/ticket/1068), [#3020](https://www.sqlalchemy.org/trac/ticket/3020)

+   **[sql] [bug]**

    恢复了`Function`的导入到`sqlalchemy.sql.expression`导入命名空间，该导入在 0.9 开始时被移除。

### postgresql

+   **[postgresql] [feature]**

    在使用 pg8000 DBAPI 时，添加了对 AUTOCOMMIT 隔离级别的支持。感谢 Tony Locke 的拉取请求。

+   **[postgresql] [feature]**

    为 PostgreSQL `ARRAY` 类型添加了一个新标志`ARRAY.zero_indexes`。当设置为`True`时，在传递给数据库之前，将在所有数组索引值上添加一个值，从而允许 Python 风格的从零开始索引和 PostgreSQL 从一开始索引之间更好地进行互操作。感谢 Alexey Terentev 的拉取请求。

    引用：[#2785](https://www.sqlalchemy.org/trac/ticket/2785)

+   **[postgresql] [bug]**

    为 PG `HSTORE` 类型添加了`hashable=False`标志，这是为了允许 ORM 在请求混合列/实体列表中的 ORM 映射的 HSTORE 列时跳过尝试“哈希”ORM 的必要性。感谢 Gunnlaugur Þór Briem 的补丁。

    此更改也**回溯**到：0.8.7

    引用：[#3053](https://www.sqlalchemy.org/trac/ticket/3053)

+   **[postgresql] [bug]**

    添加了一个新的“断开连接”消息“连接意外关闭”。这似乎与较新版本的 SSL 有关。感谢 Antti Haapala 的拉取请求。

    这个更改也**回溯**到：0.8.7

+   **[postgresql] [bug]**

    当确定异常是否为“断开连接”错误时，现在会查看 psycopg2 的`.closed`访问器；理想情况下，这应该消除对异常消息的其他检查的需求，但我们将保留这些现有消息作为备用。这应该能够处理新的情况，如“SSL EOF”条件。感谢 Dirk Mueller 的拉取请求。

    参考：[#3021](https://www.sqlalchemy.org/trac/ticket/3021)

+   **[postgresql] [enhancement]**

    在 PostgreSQL 方言中添加了一个新类型`OID`。虽然“oid”通常是 PG 中的私有类型，在现代版本中不会公开，但在某些 PG 使用情况下，如大对象支持，这些类型可能会被公开，以及在一些用户报告的模式反射使用情况中。

    参考：[#3002](https://www.sqlalchemy.org/trac/ticket/3002)

### mysql

+   **[mysql] [bug]**

    修复了在索引的`mysql_length`参数中添加的列名需要具有相同引号才能被识别的错误。修复使引号变为可选，但也为那些使用解决方法的人提供了旧的行为以实现向后兼容。

    这个更改也**回溯**到：0.8.7

    参考：[#3085](https://www.sqlalchemy.org/trac/ticket/3085)

+   **[mysql] [bug]**

    添加了对包含 KEY_BLOCK_SIZE 的索引的表进行反射的支持，使用等号。感谢 Sean McGivern 的拉取请求。

    这个更改也**回溯**到：0.8.7

### mssql

+   **[mssql] [bug]**

    修改了用于确定当前默认模式名称的查询，使用`database_principal_id()`函数与`sys.database_principals`视图结合使用，以便我们可以独立于正在进行的登录类型（例如 SQL Server、Windows 等）确定默认模式。 

    参考：[#3025](https://www.sqlalchemy.org/trac/ticket/3025)

### 测试

+   **[tests] [bug] [py3k]**

    修正了运行测试时涉及`imp`模块和 Python 3.3 或更高版本的一些弃用警告。感谢 Matt Chisholm 的拉取请求。

    参考：[#2830](https://www.sqlalchemy.org/trac/ticket/2830)

### 杂项

+   **[bug] [declarative]**

    当访问`__mapper_args__`字典时，会从声明性混合或抽象类中复制，以便 declarative 自身对此字典所做的修改不会与其他映射发生冲突。关于`version_id_col`和`polymorphic_on`参数，字典会进行修改，用本地类/表正式映射到的列替换其中的列。

    这个更改也**回溯**到：0.8.7

    参考：[#3062](https://www.sqlalchemy.org/trac/ticket/3062)

+   **[bug] [ext]**

    修复了可变扩展中的 bug，即`MutableDict`未报告`setdefault()`字典操作的更改事件。

    此更改也**回溯**到：0.8.7

    参考：[#3051](https://www.sqlalchemy.org/trac/ticket/3051)，[#3093](https://www.sqlalchemy.org/trac/ticket/3093)

+   **[bug] [ext]**

    修复了`MutableDict.setdefault()`没有返回现有值或新值的 bug（此 bug 未在任何 0.8 版本中发布）。感谢 Thomas Hervé提供的拉取请求。

    此更改也**回溯**到：0.8.7

    参考：[#3051](https://www.sqlalchemy.org/trac/ticket/3051)，[#3093](https://www.sqlalchemy.org/trac/ticket/3093)

+   **[bug] [testsuite]**

    在公共测试套件中，从不太受支持的`Text`更改为使用`String(40)`在`StringTest.test_literal_backslashes`中。感谢 Jan 提供的拉取请求。

+   **[bug] [firebird]**

    修复了一个 bug，即“limit”渲染为“SELECT FIRST n ROWS”并使用绑定参数（只有 firebird 同时具有这两个特性）的组合，再加上具有“limit”和“位置”绑定参数（例如 qmark 样式）的列级子查询，会错误地将子查询级别的位置分配在封闭 SELECT 之前，从而返回顺序错误的参数。

    参考：[#3038](https://www.sqlalchemy.org/trac/ticket/3038)

## 0.9.4

发布日期：2014 年 3 月 28 日

### 一般

+   **[general] [feature]**

    已添加对 pytest 的支持以运行测试。目前，此运行程序除了 nose 外还得到支持，并且将来可能更倾向于使用 pytest。SQLAlchemy 使用的 nose 插件系统已被拆分，以便在 pytest 下也能正常工作。目前没有计划放弃对 nose 的支持，我们希望测试套件本身可以继续保持对测试平台的中立性。请查看 README.unittests.rst 文件以获取有关使用 pytest 运行测试的更新信息。

    测试插件系统还增强了对一次针对多个数据库 URL 运行测试的支持，通过多次指定`--db`和/或`--dburi`标志。这不会为每个数据库运行整个测试套件，而是允许特定于某些后端的测试用例在运行测试时使用该后端。当使用 pytest 作为测试运行器时，系统还将多次运行特定的测试套件，每个数据库运行一次，特别是那些在“方言套件”中的测试。计划是增强系统还将被 Alembic 使用，并允许 Alembic 在一次运行中针对多个后端运行迁移操作测试，包括 Alembic 本身不包含的第三方后端。还鼓励第三方方言和扩展标准化使用 SQLAlchemy 的测试套件作为基础；请参阅 README.dialects.rst 文件以了解从 SQLAlchemy 的测试平台构建的背景。

+   **[general] [bug]**

    调整了`setup.py`文件以支持将来可能从 setuptools 中删除`setuptools.Feature`扩展。如果不存在此关键字，设置将仍然成功使用 setuptools 而不是退回到 distutils。现在还可以通过设置 DISABLE_SQLALCHEMY_CEXT 环境变量来禁用 C 扩展构建。无论 setuptools 是否可用，此变量都有效。

    此更改也**回溯**到：0.8.6

    参考：[#2986](https://www.sqlalchemy.org/trac/ticket/2986)

+   **[general] [bug]**

    修复了在 Python 3.4 中发生的一些测试/功能失败，特别是用于包装“列默认”可调用对象的逻辑对于 Python 内置函数不起作用。

    参考：[#2979](https://www.sqlalchemy.org/trac/ticket/2979)

### orm

+   **[orm] [feature]**

    新增参数`mapper.confirm_deleted_rows`。默认为 True，表示一系列 DELETE 语句应确认游标行数与应匹配的主键数量相匹配；这种行为在大多数情况下已被取消（除非使用 version_id），以支持自引用 ON DELETE CASCADE 的不寻常边缘情况；为了适应这一点，消息现在只是一个警告，而不是异常，并且可以使用该标志指示期望自引用级联删除的映射。另请参见[#2403](https://www.sqlalchemy.org/trac/ticket/2403)以了解有关原始更改的背景。

    参考：[#3007](https://www.sqlalchemy.org/trac/ticket/3007)

+   **[orm] [feature]**

    如果将`MapperEvents.before_configured()`或`MapperEvents.after_configured()`事件应用于特定的映射器或映射类，则会发出警告，因为这些事件仅在通用级别上为`Mapper`目标调用。

+   **[orm] [feature]**

    添加了一个新的关键字参数`once=True`到`listen()`和`listens_for()`。这是一个方便的功能，它将包装给定的监听器，使其只调用一次。

+   **[orm] [feature]**

    添加了一个新选项到`relationship.innerjoin`，即指定字符串`"nested"`。当设置为`"nested"`时，与`True`相反，连接的“链”将在现有外连接的右侧括起内连接，而不是将其链接为一串外连接。当 0.9 发布时，这可能本应该是默认行为，因为我们在 ORM 中引入了右嵌套连接的功能，但是目前我们将其保留为非默认行为，以避免进一步的意外。

    请参阅

    在联接的急切加载中可用的右嵌套内连接

    参考：[#2976](https://www.sqlalchemy.org/trac/ticket/2976)

+   **[orm] [bug]**

    修复了 ORM 中的一个 bug，即更改对象的主键，然后将其标记为 DELETE 会失败，无法定位正确的行进行 DELETE 操作。

    此更改还被**回溯**到：0.8.6

    参考：[#3006](https://www.sqlalchemy.org/trac/ticket/3006)

+   **[orm] [bug]**

    修复了从 0.8.3 中的回归，因为[#2818](https://www.sqlalchemy.org/trac/ticket/2818)的结果是`Query.exists()`在只有一个`Query.select_from()`条目但没有其他实体的查询上不起作用。

    此更改还被**回溯**到：0.8.6

    参考：[#2995](https://www.sqlalchemy.org/trac/ticket/2995)

+   **[orm] [bug]**

    改进了一个错误消息，如果对非可选择的内容进行了查询（例如`literal_column()`），然后尝试使用`Query.join()`使“左”侧确定为`None`，然后失败。现在明确检测到了这种情况。

    此更改也**回溯**到：0.8.6

+   **[orm] [bug]**

    从`sqlalchemy.orm.interfaces.__all__`中删除了过时的名称，并使用当前名称进行刷新，以便再次从此模块进行`import *`操作。

    此更改也**回溯**到：0.8.6

    参考：[#2975](https://www.sqlalchemy.org/trac/ticket/2975)

+   **[orm] [bug]**

    修复了一个非常古老的行为，即一对多的延迟加载可能不适当地拉入父表，并且根据父表中的内容返回不一致的结果，当主连接包含某种针对父表的鉴别器时，例如`and_(parent.id == child.parent_id, parent.deleted == False)`。虽然这种主连接对于一对多来说并没有太多意义，但当应用于多对一方时稍微更常见，并且一对多是由反向引用产生的。在这种情况下加载`child`的行将保持查询中的`parent.deleted == False`不变，从而将其拉入 FROM 子句并执行笛卡尔积。新行为现在将适当地替换本地“parent.deleted”的值以用于该参数。尽管通常，一个真实的应用程序可能希望在任何情况下为 o2m 方面使用不同的主连接。

    参考：[#2948](https://www.sqlalchemy.org/trac/ticket/2948)

+   **[orm] [bug]**

    改进了“如何从 A 连接到 B”的检查，以便当一个表具有多个、复合的外键指向一个父表时，`relationship.foreign_keys`参数将被正确解释以解决歧义；以前，这种情况会引发存在多个 FK 路径，而实际上 foreign_keys 参数应该确定哪一个是预期的。

    参考：[#2965](https://www.sqlalchemy.org/trac/ticket/2965)

+   **[orm] [bug]**

    为尚未完全记录的`insert=True`标志添加了对`listen()`与 mapper / instance 事件一起工作的支持。

+   **[orm] [bug] [engine]**

    修复了一个 bug，即在类级别设置监听事件（例如在`Mapper`或`ClassManager`级别，而不是在单个映射类上，以及在`Connection`上，同时还使用了内部参数转换（在这些类别中大多数情况下）会导致无法移除。

    参考：[#2973](https://www.sqlalchemy.org/trac/ticket/2973)

+   **[orm] [bug]**

    修复了从 0.8 版本开始的回归问题，即在使用像`lazyload()`这样的选项与“通配符”表达式（例如，`"*"`）时，在查询中没有包含任何实际实体的情况下，会触发断言错误。这个断言是为其他情况准备的，并且意外地捕获了这种情况。

+   **[orm] [bug] [sqlite]**

    对 SQLite“联接重写”进行了更多修复；在 0.9.3 版本发布前实施的来自[#2967](https://www.sqlalchemy.org/trac/ticket/2967)的修复影响了 UNION 包含嵌套联接的情况。 “联接重写”是一个具有广泛可能性的功能，并且是我们多年来引入的第一个复杂的“SQL 重写”功能，因此我们正在进行许多迭代（不像 0.2/0.3 系列中的急切加载，0.4/0.5 中的多态加载）。我们应该很快就会到达目的地，所以感谢您的耐心等待：）。

    引用：[#2969](https://www.sqlalchemy.org/trac/ticket/2969)

### 例子

+   **[examples] [bug]**

    修复了版本化历史示例中的错误，其中列级别的 INSERT 默认值会阻止写入 NULL 的历史值。

### engine

+   **[engine] [feature]**

    为方言级事件添加了一些新的事件机制；最初的实现允许事件处理程序重新定义任意方言调用 DBAPI 游标的 execute()或 executemany()的具体机制。此时的新事件在某种程度上是半公开和试验性的，支持即将推出的一些与事务相关的扩展。

+   **[engine] [feature]**

    现在可以在一个或多个`Connection`对象已创建后（例如通过 orm `Session`或通过显式连接）与`Engine`相关联事件监听器，并且该监听器将捕获这些连接的事件。以前，出于性能考虑，事件传输从`Engine`到`Connection`仅在初始化时进行，但是我们内联了一堆条件检查以使此成为可能，而不需要任何额外的函数调用。

    引用：[#2978](https://www.sqlalchemy.org/trac/ticket/2978)

+   **[engine] [bug]**

    对于`Engine`在检测到“断开”条件时重新使用连接池的机制进行了重大改进；不再丢弃池并显式关闭连接，而是保留池并更新“生成”时间戳以反映当前时间，从而在下次检出时重新使用所有现有连接。这极大地简化了回收过程，消除了等待旧池的“唤醒”连接尝试的需要，并消除了在回收操作期间可能创建的许多立即丢弃的“池”对象的竞争条件。

    参考：[#2985](https://www.sqlalchemy.org/trac/ticket/2985)

+   **[引擎] [错误]**

    `ConnectionEvents.after_cursor_execute()` 事件现在被发射到`Connection`的“_cursor_execute()”方法中；这是用于诸如在 INSERT 语句之前执行序列等快速执行器，以及用于方言启动检查（如 unicode 返回、字符集等）的“快速”执行器。`ConnectionEvents.before_cursor_execute()` 事件已经在这里被调用。这里的“executemany”标志现在总是设置为 False，因为此事件始终对应单个执行。以前，如果我们代表 executemany INSERT 语句执行操作，该标志可能为 True。

### sql

+   **[sql] [特性]**

    增加了对布尔值的文字渲染支持，例如“true” / “false”或“1” / “0”。

+   **[sql] [特性]**

    添加了一个新功能`conv()`，其目的是将约束名称标记为已经应用了命名约定。从 Alembic 0.6.4 开始，Alembic 迁移将使用此令牌，以便在迁移脚本中呈现已标记为已经受到命名约定影响的约束。

+   **[sql] [特性]**

    为了帮助依赖于向构造添加临时关键字参数的现有方案，已增强了模式级构造的方言级关键字参数系统。

    例如，`Index` 这样的构造将再次在构造后接受`Index.kwargs` 集合中的临时关键字参数：

    ```py
    idx = Index("a", "b")
    idx.kwargs["mysql_someargument"] = True
    ```

    为了适应允许在构建时使用自定义参数的用例，`DialectKWArgs.argument_for()` 方法现在允许进行此注册：

    ```py
    Index.argument_for("mysql", "someargument", False)

    idx = Index("a", "b", mysql_someargument=True)
    ```

    参见

    `DialectKWArgs.argument_for()`

    参考：[#2866](https://www.sqlalchemy.org/trac/ticket/2866), [#2962](https://www.sqlalchemy.org/trac/ticket/2962)

+   **[sql] [bug]**

    修复了 `tuple_()` 构造中的错误，其中基本上第一个 SQL 表达式的“类型”将被应用为比较元组值的“比较类型”；这在某些情况下会导致不适当的“类型转换”发生，例如当元组中混合了字符串和二进制值时，错误地将目标值转换为二进制，即使左侧的值不是二进制。 `tuple_()` 现在在其值列表中期望异构类型。

    此更改也 **回溯** 至：0.8.6

    参考：[#2977](https://www.sqlalchemy.org/trac/ticket/2977)

+   **[sql] [bug]**

    修复了 0.9 版本中的一个回归，即 `Table` 如果无法正确反映则不会从父 `MetaData` 中删除，即使处于无效状态。 Pullreq 由 Roman Podoliaka 提供。

    参考：[#2988](https://www.sqlalchemy.org/trac/ticket/2988)

+   **[sql] [bug]**

    `MetaData.naming_convention` 特性现在还将应用于与 `Column` 直接关联的 `CheckConstraint` 对象，而不仅仅是 `Table`。

+   **[sql] [bug]**

    修复了新的 `MetaData.naming_convention` 特性中的错误，在使用“%(constraint_name)s”标记的检查约束的名称会为布尔或枚举类型生成的约束而重复，而总体重复的事件将导致“%(constraint_name)s”标记不断增加。

    参考：[#2991](https://www.sqlalchemy.org/trac/ticket/2991)

+   **[sql] [bug]**

    当接收到没有名称的`BindParameter`时，调整了将名称应用于 .c 集合的逻辑，例如通过`literal()`或类似方式；绑定参数的“键”被用作 .c 内的键，而不是呈现的名称。由于这些绑定在任何情况下都具有“匿名”名称，这使得在可选地中，如果它们未被标记，那么单个绑定参数可以拥有自己的名称。

    参考：[#2974](https://www.sqlalchemy.org/trac/ticket/2974)

+   **[sql] [bug]**

    当呈现具有重复列的`FromClause.c`集合时，对其行为进行了一些更改。仍然存在发出警告并用相同名称替换旧列的行为，以某种程度上保持向后兼容性。然而，替换的列现在仍然与`c`集合相关联，现在在一个集合`._all_columns`中，这由诸如别名和联合之类的结构使用，以处理`c`中的列集合更接近于实际列列表而不是唯一的键名称集合。这有助于处理在联合等中使用具有相同命名列的 SELECT 语句的情况，以便联合可以按位置匹配列，并且在这里仍然有一些使用`FromClause.corresponding_column()`的机会（它现在可以返回一个仅在 selectable.c._all_columns 中而不是以其他方式命名的列）。新集合加下划线是因为我们仍然需要决定此列表可能会结束的位置。从理论上讲，它将成为 iter(selectable.c)的结果，但这意味着迭代的长度将不再匹配 keys()的长度，需要检查该行为。

    参考：[#2974](https://www.sqlalchemy.org/trac/ticket/2974)

+   **[sql] [bug]**

    修复了新的`TextClause.columns()`方法中列的顺序位置上不会被保留的问题。这可能会在类似于将生成的`TextAsFrom`对象应用于联合的位置情况中产生潜在影响。

### postgresql

+   **[postgresql] [feature]**

    对于 psycopg2 DBAPI，启用了“合理的多行计数”检查，因为这似乎是作为 psycopg2 2.0.9 的支持。

    此更改也被**反向移植**到：0.8.6

+   **[postgresql] [bug]**

    修复了由发布 0.8.5 / 0.9.3 的兼容性增强引起的回归，其中针对仅���用于 8.1、8.2 系列的 PostgreSQL 版本的索引反射再次出现问题，围绕着一直存在问题的 int2vector 类型。虽然从 8.1 开始，int2vector 支持数组操作，但显然从 8.3 开始才支持将其转换为 varchar。 

    这个更改也被**回溯**到了：0.8.6

    参考：[#3000](https://www.sqlalchemy.org/trac/ticket/3000)

### mysql

+   **[mysql] [bug]**

    调整了 mysql-connector-python 的设置；在 Py2K 中，“supports unicode statements”标志现在为 False，因此 SQLAlchemy 将在发送到数据库之前将*SQL 字符串*（注意：*不是*参数）编码为字节。这似乎允许所有与 unicode 相关的测试通过 mysql-connector，包括那些使用非 ascii 表/列名称以及使用 unicode 在 cursor.executemany()下的 TEXT 类型的一些测试。

### oracle

+   **[oracle] [feature]**

    向 cx_Oracle 方言添加了一个新的引擎选项`coerce_to_unicode=True`，它将 cx_Oracle 的输出类型处理程序方法恢复到 Python 2 中的 Unicode 转换，这是因为在 0.9.2 中由于[#2911](https://www.sqlalchemy.org/trac/ticket/2911)而被移除。一些用例希望对所有字符串值进行 Unicode 强制转换，尽管存在性能问题。拉取请求由 Christoph Zwerschke 提供。

    参考：[#2911](https://www.sqlalchemy.org/trac/ticket/2911)

+   **[oracle] [bug]**

    添加了新的数据类型`DATE`，它是`DateTime`的子类。由于 Oracle 本身没有“datetime”类型，而只有`DATE`，因此在 Oracle 方言中，将`DATE`类型作为`DateTime`的实例是合适的。这个问题并不会改变类型的行为，因为数据转换无论如何都是由 DBAPI 处理的，但是改进的子类布局将有助于检查类型以实现跨数据库兼容性。此外，从 Oracle 方言中删除了大写的`DATETIME`，因为在该上下文中这种类型是无效的。

    参考：[#2987](https://www.sqlalchemy.org/trac/ticket/2987)

### 测试

+   **[tests] [bug]**

    修复了一些错误的`u''`字符串，这些字符串会导致在 Py3.2 中无法通过测试。补丁由 Arfrever Frehtes Taifersar Arahesis 提供。

    参考：[#2980](https://www.sqlalchemy.org/trac/ticket/2980)

### 杂项

+   **[bug] [ext]**

    修复了可变扩展中的错误以及`flag_modified()`中的 bug，在这种情况下，如果属性已重新分配给自身，则更改事件将不会传播。

    这个更改也被**回溯**到了：0.8.6

    参考：[#2997](https://www.sqlalchemy.org/trac/ticket/2997)

+   **[错误] [自动映射] [扩展]**

    为自动映射添加了支持，当关系不应该在两个类之间创建时，这两个类处于联合继承关系中，针对将子类链接回超类的外键。

    参考：[#3004](https://www.sqlalchemy.org/trac/ticket/3004)

+   **[错误] [池]**

    在`SingletonThreadPool`中修复了一个小问题，即在“清理”过程中当前要返回的连接可能会在无意中被清除。修补程序提供 jd23。

+   **[错误] [扩展] [Py3k]**

    在关联代理中修复了一个错误，在该错误中，赋予一个空切片（例如`x[:] = [...]`）在 Py3k 上会失败。

+   **[错误] [扩展]**

    修复了一个与 [#2810](https://www.sqlalchemy.org/trac/ticket/2810) 相关的关联代理中的回归，该回归导致一个用户提供的“getter”在从非存在目标获取标量值时不再接收到`None`值。此更改引入的 None 检查现在移至默认 getter 中，因此用户提供的 getter 也将再次接收到 None 值。

    参考：[#2810](https://www.sqlalchemy.org/trac/ticket/2810)

### 通用

+   **[通用] [特性]**

    添加了对 pytest 运行测试的支持。此运行器目前正在额外支持 nose，并且将来可能更倾向于使用 pytest。SQLAlchemy 使用的 nose 插件系统已分离出来，以便它也能在 pytest 下工作。目前没有计划放弃对 nose 的支持，我们希望测试套件本身可以继续尽可能保持与测试平台的不可知性。请参阅 README.unittests.rst 文件，了解使用 pytest 运行测试的更新信息。

    测试插件系统还增强了，支持一次性针对多个数据库 URL 运行测试，通过多次指定`--db`和/或`--dburi`标志。这不会针对每个数据库运行整个测试套件，而是允许特定于某些后端的测试用例在运行测试时使用该后端。当将 pytest 作为测试运行器时，系统还将多次运行特定的测试套件，每个数据库运行一次，特别是“方言套件”中的测试。计划增强的系统也将由 Alembic 使用，并允许 Alembic 在一个运行中针对多个后端运行迁移操作测试，包括 Alembic 本身未包含的第三方后端。还鼓励第三方方言和扩展标准化为 SQLAlchemy 的测试套件作为基础；请参阅 README.dialects.rst 文件，了解从 SQLAlchemy 的测试平台构建的背景信息。

+   **[通用] [错误]**

    调整了`setup.py`文件，以支持将来可能从 setuptools 中删除`setuptools.Feature`扩展。如果不存在此关键字，则设置仍将成功使用 setuptools 而不是退回到 distutils。现在还可以通过设置 DISABLE_SQLALCHEMY_CEXT 环境变量来禁用 C 扩展构建。无论 setuptools 是否可用，此变量都有效。 

    此更改也被**回溯**到：0.8.6

    参考：[#2986](https://www.sqlalchemy.org/trac/ticket/2986)

+   **[通用] [错误]**

    修复了在 Python 3.4 中发生的一些测试/特性失败，特别是用于包装“列默认值”可调用对象的逻辑对于 Python 内置函数不起作用。

    参考：[#2979](https://www.sqlalchemy.org/trac/ticket/2979)

### orm

+   **[ORM] [特性]**

    添加了新参数`mapper.confirm_deleted_rows`。默认为 True，表示一系列 DELETE 语句应确认游标行数与应匹配的主键数相匹配；这种行为在大多数情况下已被取消（除非使用 version_id），以支持自引用 ON DELETE CASCADE 的不寻常边缘情况；为了适应这一点，消息现在只是一个警告，而不是异常，并且该标志可用于指示期望此类自引用级联删除的映射。有关原始更改的背景，请参见[#2403](https://www.sqlalchemy.org/trac/ticket/2403)。

    参考：[#3007](https://www.sqlalchemy.org/trac/ticket/3007)

+   **[ORM] [特性]**

    如果将`MapperEvents.before_configured()`或`MapperEvents.after_configured()`事件应用于特定的映射器或映射类，则会发出警告，因为这些事件仅在通用级别的`Mapper`目标上调用。

+   **[ORM] [特性]**

    向`listen()`和`listens_for()`添加了一个新的关键字参数`once=True`。这是一个方便的功能，它将包装给定的监听器，以便只调用一次。

+   **[ORM] [特性]**

    在`relationship.innerjoin`中添加了一个新选项，即指定字符串`"nested"`。当设置为`"nested"`而不是`True`时，连接的“链”将在现有外连接的右侧括号内连接内连接，而不是作为一系列外连接的字符串连接。当 0.9 发布时，这可能应该是默认行为，因为我们在 ORM 中引入了右嵌套连接的功能，但是为了避免进一步的意外，我们目前将其保留为非默认行为。

    另请参见

    右嵌套内连接可用于连接的预加载

    参考：[#2976](https://www.sqlalchemy.org/trac/ticket/2976)

+   **[orm] [bug]**

    修复了 ORM 中的一个 bug，即更改对象的主键，然后将其标记为 DELETE 将无法定位正确的行进行 DELETE。

    此更改也**回溯**到：0.8.6

    参考：[#3006](https://www.sqlalchemy.org/trac/ticket/3006)

+   **[orm] [bug]**

    修复了从 0.8.3 版本开始的回归问题，这是由于[#2818](https://www.sqlalchemy.org/trac/ticket/2818)导致的，`Query.exists()`在只有一个`Query.select_from()`条目但没有其他实体的查询上无法工作。

    此更改也**回溯**到：0.8.6

    参考：[#2995](https://www.sqlalchemy.org/trac/ticket/2995)

+   **[orm] [bug]**

    改进了一个错误消息，如果对非可选择的内容进行了查询（例如`literal_column()`），然后尝试使用`Query.join()`进行连接，使得“左”侧被确定为`None`，然后失败。现在明确检测到这种情况。

    此更改也**回溯**到：0.8.6

+   **[orm] [bug]**

    从`sqlalchemy.orm.interfaces.__all__`中删除了过时的名称，并刷新为当前名称，以便再次从该模块进行`import *`操作。

    此更改也**回溯**到：0.8.6

    参考：[#2975](https://www.sqlalchemy.org/trac/ticket/2975)

+   **[orm] [bug]**

    修复了一个非常古老的行为，即懒加载在一对多关系中可能会不适当地拉入父表，并且基于父表中的内容返回不一致的结果，当 `primaryjoin` 包含针对父表的某种类型的鉴别器时，例如 `and_(parent.id == child.parent_id, parent.deleted == False)`。虽然这种 `primaryjoin` 对于一对多关系并没有太多意义，但当应用于多对一的一侧时，这种情况稍微更为常见，并且一对多是作为反向引用的结果。在这种情况下加载 `child` 行会在查询中保留 `parent.deleted == False`，从而将其拉入 `FROM` 子句并执行笛卡尔积。新行为现在将适当地替换本地的 “parent.deleted” 的值为该参数。尽管通常，一个真实的应用程序可能想在任何情况下都使用不同的 `primaryjoin`。

    参考：[#2948](https://www.sqlalchemy.org/trac/ticket/2948)

+   **[orm] [bug]**

    改进了“如何从 A 连接到 B”的检查，使得当一个表有多个复合外键指向父表时，`relationship.foreign_keys` 参数将被正确解释以解决歧义；以前，此条件会引发存在多个 FK 路径的错误，而事实上，`foreign_keys` 参数应该确定哪个路径是预期的。

    参考：[#2965](https://www.sqlalchemy.org/trac/ticket/2965)

+   **[orm] [bug]**

    添加了对尚未完全文档化的 `insert=True` 标志与 mapper / 实例事件一起使用的支持。

+   **[orm] [bug] [engine]**

    修复了一个 bug，在类级别设置监听事件（例如在 `Mapper` 或 `ClassManager` 级别，而不是在单个映射类上，并且还在 `Connection` 上，也使用了内部参数转换（这在这些类别中是大多数情况）将无法移除。

    参考：[#2973](https://www.sqlalchemy.org/trac/ticket/2973)

+   **[orm] [bug]**

    修复了 0.8 中的回归，当使用像 `lazyload()` 这样的选项与“通配符”表达式，例如 `"*"`，时，在查询中不包含任何实际实体时会引发断言错误。此断言是针对其他情况的，而无意中捕获了这种情况。

+   **[orm] [bug] [sqlite]**

    对 SQLite“连接重写”进行了更多修复；在 0.9.3 版本发布之前实施的[#2967](https://www.sqlalchemy.org/trac/ticket/2967)修复影响了 UNION 中包含嵌套连接的情况。“连接重写”是一个具有广泛可能性的功能，是多年来我们引入的第一个复杂的“SQL 重写”功能，因此我们正在进行许多迭代（有点类似于 0.2/0.3 系列中的急加载，0.4/0.5 中的多态加载）。我们应该很快就会完成，所以感谢您的耐心等待 :).

    参考：[#2969](https://www.sqlalchemy.org/trac/ticket/2969)

### 示例

+   **[示例] [错误]**

    修复了 versioned_history 示例中的 bug，其中列级别的 INSERT 默认值会阻止写入 NULL 的历史值。

### 引擎

+   **[engine] [功能]**

    为方言级事件添加了一些新的事件机制；初始实现允许事件处理程序重新定义任意方言在 DBAPI 游标上调用 execute()或 executemany()的具体机制。这些新事件，目前是半公开和实验性的，是为了支持即将推出的一些与事务相关的扩展。

+   **[engine] [功能]**

    现在可以在创建一个或多个`Connection`对象（例如通过 orm `Session`或通过显式连接）之后，将事件监听器与`Engine`关联，监听器将从这些连接中接收事件。以前，性能问题导致事件传输从`Engine`到`Connection`仅在初始化时进行，但我们内联了一堆条件检查，使得这一点成为可能，而无需进行任何额外的函数调用。

    参考：[#2978](https://www.sqlalchemy.org/trac/ticket/2978)

+   **[engine] [错误]**

    对`Engine`在检测到“断开”条件时回收连接池的机制进行了重大改进；不再丢弃池并显式关闭连接，而是保留池并更新“生成”时间戳以反映当前时间，从而在下次检出时回收所有现有连接。这极大地简化了回收过程，消除了等待旧池的“唤醒”连接尝试的需要，并消除了在回收操作期间可能创建许多立即丢弃的“池”对象的竞争条件。

    参考：[#2985](https://www.sqlalchemy.org/trac/ticket/2985)

+   **[engine] [错误]**

    `ConnectionEvents.after_cursor_execute()` 事件现在针对`Connection`的“_cursor_execute()”方法发出；这是用于诸如在 INSERT 语句之前执行序列等快速执行器，以及用于方言启动检查（如 unicode 返回、字符集等）的事件。`ConnectionEvents.before_cursor_execute()` 事件已在此处调用。此处的“executemany”标志现在始终设置为 False，因为此事件始终对应单个执行。以前，如果我们代表 executemany INSERT 语句执行操作，标志可能为 True。

### sql

+   **[sql] [功能]**

    增加了对布尔值的文字呈现支持，例如“true” / “false”或“1” / “0”。

+   **[sql] [功能]**

    增加了一个新功能`conv()`，其目的是将约束名称标记为已经应用了命名约定。从 Alembic 0.6.4 开始，Alembic 迁移将使用此标记，在迁移脚本中呈现已经标记为已经受到命名约定影响的约束。

+   **[sql] [功能]**

    已增强新的方言级关键字参数系统，以协助依赖于向结构添加临时关键字参数的现有方案。

    例如，`Index` 这样的结构在构造后将再次接受`Index.kwargs` 集合中的临时关键字参数：

    ```py
    idx = Index("a", "b")
    idx.kwargs["mysql_someargument"] = True
    ```

    为了适应允许在构造时使用自定义参数的用例，`DialectKWArgs.argument_for()` 方法现在允许进行此注册：

    ```py
    Index.argument_for("mysql", "someargument", False)

    idx = Index("a", "b", mysql_someargument=True)
    ```

    另请参阅

    `DialectKWArgs.argument_for()`

    参考：[#2866](https://www.sqlalchemy.org/trac/ticket/2866), [#2962](https://www.sqlalchemy.org/trac/ticket/2962)

+   **[sql] [错误]**

    修复了`tuple_()`构造中的错误，其中实质上第一个 SQL 表达式的“类型”会被应用为与比较的元组值的“比较类型”；在某些情况下，这会导致不恰当的“类型强制转换”发生，例如当一个混合了 String 和 Binary 值的元组错误地将目标值强制转换为 Binary，即使左侧的值并非如此。`tuple_()`现在期望其值列表中存在异构类型。

    此更改也已**回溯**至：0.8.6

    参考：[#2977](https://www.sqlalchemy.org/trac/ticket/2977)

+   **[sql] [bug]**

    修复了 0.9 版本中的一个回归问题，即`Table`未能正确反映时，即使处于无效状态，也不会从父`MetaData`中移除。感谢 Roman Podoliaka 的 Pullreq。

    参考：[#2988](https://www.sqlalchemy.org/trac/ticket/2988)

+   **[sql] [bug]**

    `MetaData.naming_convention`功能现在还将应用于直接与`Column`关联的`CheckConstraint`对象，而不仅仅是在`Table`上。

+   **[sql] [bug]**

    修复了新的`MetaData.naming_convention`功能中的错误，其中使用“%(constraint_name)s”标记的检查约束的名称会在布尔或枚举类型生成的约束中重复，整体重复事件会导致“%(constraint_name)s”标记不断累积。

    参考：[#2991](https://www.sqlalchemy.org/trac/ticket/2991)

+   **[sql] [bug]**

    调整了将名称应用于.c 集合时的逻辑，当接收到无名称的`BindParameter`时，例如通过`literal()`或类似方式；绑定参数的“键”被用作.c 中的键，而不是渲染的名称。由于这些绑定在任何情况下都具有“匿名”名称，因此这允许单独的绑定参数在可选择的情况下具有自己的名称，如果它们没有被标记。

    参考：[#2974](https://www.sqlalchemy.org/trac/ticket/2974)

+   **[sql] [bug]**

    对`FromClause.c`集合的行为进行了一些更改，当出现重复列时。仍然存在发出警告并替换旧列的行为；特别是替换是为了保持向后兼容性。但是，替换的列现在仍然与`c`集合关联在一个名为`._all_columns`的集合中，该集合由诸如别名和联合之类的结构使用，以处理`c`中的列集更加接近实际列列表而不是唯一的键名称。这有助于处理在联合等中使用具有相同名称的 SELECT 语句的情况，以便联合可以按位置匹配列，并且在这里仍然有一些机会使用`FromClause.corresponding_column()`（现在它可以返回仅在 selectable.c._all_columns 中的列，而不是其他命名）。新集合下划线的原因是我们仍然需要决定这个列表可能会在哪里结束。理论上它将成为 iter(selectable.c)的结果，但是这意味着迭代的长度将不再与 keys()的长度匹配，需要检查该行为。

    参考：[#2974](https://www.sqlalchemy.org/trac/ticket/2974)

+   **[sql] [错误]**

    修复了新的`TextClause.columns()`方法中的问题，其中按位置给定的列的排序不会被保留。这可能会在位置相关的情况下产生潜在影响，比如将生成的`TextAsFrom`对象应用到联合操作中。

### postgresql

+   **[postgresql] [功能]**

    为 psycopg2 DBAPI 启用了“合理的多行计数”检查，因为这似乎在 psycopg2 2.0.9 中得到了支持。

    此更改还**回溯**到：0.8.6

+   **[postgresql] [错误]**

    修复了由发布 0.8.5 / 0.9.3 的兼容性增强引起的回归，其中索引反射在仅适用于 8.1、8.2 系列的 PostgreSQL 版本上再次中断，周围围绕着永远有问题的 int2vector 类型。虽然 int2vector 自 8.1 起支持数组操作，但显然它只支持从 8.3 开始的 varchar 的 CAST。

    此更改还**回溯**到：0.8.6

    参考：[#3000](https://www.sqlalchemy.org/trac/ticket/3000)

### mysql

+   **[mysql] [错误]**

    调整了 mysql-connector-python 的设置；在 Py2K 中，“supports unicode statements”标志现在为 False，因此 SQLAlchemy 将在发送到数据库之前将*SQL 字符串*（注意：*不是*参数）编码为字节。这似乎允许所有与 mysql-connector 相关的 unicode 测试通过，包括那些使用非 ASCII 表/列名称以及使用 cursor.executemany()下的 unicode 的 TEXT 类型的一些测试。

### oracle

+   **[oracle] [功能]**

    为 cx_Oracle 方言添加了一个新的引擎选项`coerce_to_unicode=True`，该选项恢复了在 Python 2 下的 cx_Oracle 输出类型处理程序方法进行 Unicode 转换的功能，该功能在 0.9.2 中被删除，原因是[#2911](https://www.sqlalchemy.org/trac/ticket/2911)。一些用例希望对所有字符串值进行 Unicode 强制转换，尽管存在性能问题。拉取请求由 Christoph Zwerschke 提供。

    参考：[#2911](https://www.sqlalchemy.org/trac/ticket/2911)

+   **[oracle] [bug]**

    添加了新的数据类型`DATE`，它是`DateTime`的子类。由于 Oracle 没有“datetime”类型，而只有`DATE`，因此在这里，Oracle 方言中的`DATE`类型作为`DateTime`的实例是合适的。这个问题并不会改变类型的行为，因为数据转换无论如何都由 DBAPI 处理，但是改进的子类布局将有助于检查类型以实现跨数据库兼容性。此外，从 Oracle 方言中删除了大写的`DATETIME`，因为这种类型在该上下文中不起作用。

    参考：[#2987](https://www.sqlalchemy.org/trac/ticket/2987)

### 测试

+   **[tests] [bug]**

    修复了一些错误的`u''`字符串，这些字符串会导致在 Py3.2 中无法通过测试。补丁由 Arfrever Frehtes Taifersar Arahesis 提供。

    参考：[#2980](https://www.sqlalchemy.org/trac/ticket/2980)

### 杂项

+   **[bug] [ext]**

    修复了可变扩展中的错误以及`flag_modified()`中的错误，如果属性已重新分配给自身，则更改事件将不会传播。

    此更改也**回溯**到：0.8.6

    参考：[#2997](https://www.sqlalchemy.org/trac/ticket/2997)

+   **[bug] [automap] [ext]**

    为自动映射添加了支持，用于处理在联合继承关系中不应该创建两个类之间关系的情况，对于那些将子类链接回超类的外键。

    参考：[#3004](https://www.sqlalchemy.org/trac/ticket/3004)

+   **[bug] [pool]**

    修复了`SingletonThreadPool`中的一个小问题，在“清理”过程中，当前要返回的连接可能会在无意中被清除。补丁由 jd23 提供。

+   **[bug] [ext] [py3k]**

    修复了关联代理中的错误，其中分配一个空切片（例如`x[:] = [...]`）将在 Py3k 上失败。

+   **[bug] [ext]**

    修复了由[#2810](https://www.sqlalchemy.org/trac/ticket/2810)引起的关联代理中的回归问题，导致当从不存在的目标获取标量值时，用户提供的“getter”不再接收`None`值。此更改引入的 None 检查现在移至默认 getter 中，因此用户提供的 getter 将再次接收 None 值。

    参考：[#2810](https://www.sqlalchemy.org/trac/ticket/2810)

## 0.9.3

发布日期：2014 年 2 月 19 日

### orm

+   **[orm] [功能]**

    添加了新的`MapperEvents.before_configured()`事件，允许在`configure_mappers()`开始时触发事件，以及在声明中的`__declare_first__()`钩子以补充`__declare_last__()`。

+   **[orm] [错误]**

    修复了`Query.get()`在查询存在条件时未能一致引发`InvalidRequestError`的错误，当给定的标识已经存在于标识映射中时。

    此更改也**回溯**到：0.8.5

    参考：[#2951](https://www.sqlalchemy.org/trac/ticket/2951)

+   **[orm] [错误] [sqlite]**

    修复了 SQLite“连接重写”中的错误，其中 exists()构造的使用未能正确重写，例如当 exists 映射到复杂嵌套连接场景中的 column_property 时。还修复了一个有些相关的问题，即当目标是别名表而不是单独的别名列时，连接重写会在 SELECT 语句的 columns 子句上失败。

    参考：[#2967](https://www.sqlalchemy.org/trac/ticket/2967)

+   **[orm] [错误]**

    修复了 0.9 版本中的一个回归问题，即应用于基类（例如具有 propagate=True 标志的声明基类）的 ORM 实例或映射器事件未能应用于由于断言而失败的现有映射类。此外，修复了在删除此类事件时可能发生的属性错误，具体取决于首次分配方式。

    参考：[#2949](https://www.sqlalchemy.org/trac/ticket/2949)

+   **[orm] [错误]**

    改进了复合属性的初始化逻辑，使得调用`MyClass.attribute`不需要配置映射器步骤已发生，例如，它将正常工作而不会引发任何错误。

    参考：[#2935](https://www.sqlalchemy.org/trac/ticket/2935)

+   **[orm] [错误]**

    更多关于[ticket:2932]的问题首次在 0.9.2 中解决，其中使用形式为`<tablename>_<columnname>`的列键与文本中别名列的匹配仍然无法在 ORM 级别匹配，这最终是由于核心列匹配问题。已添加额外规则，以便在使用`TextAsFrom`构造或文字列时考虑列`_label`。

    参考：[#2932](https://www.sqlalchemy.org/trac/ticket/2932)

### ORM 声明

+   **[ORM] [声明] [错误]**

    修复了`AbstractConcreteBase`在声明关系配置中无法完全可用的错误，因为其字符串类名在映射器配置时不可用。该类现在显式将自己添加到类注册表中，并且`AbstractConcreteBase`以及`ConcreteBase`都在`configure_mappers()`设置中在配置映射器之前使用新的`MapperEvents.before_configured()`事件设置自己。

    参考：[#2950](https://www.sqlalchemy.org/trac/ticket/2950)

### 示例

+   **[示例] [特性]**

    在版本化行示例中添加了可选的“changed”列，以及当版本化的`Table`具有显式`Table.schema`参数时的支持。感谢 jplaverdure 的拉取请求。

### 引擎

+   **[引擎] [错误] [池]**

    修复了由[#2880](https://www.sqlalchemy.org/trac/ticket/2880)引起的关键回归，其中新的并发能力从池中返回连接意味着“first_connect”事件现在也不再同步，从而在即使在最小并发情况下也会导致方言配置错误。

    此更改也**回溯**到：0.8.5

    参考：[#2880](https://www.sqlalchemy.org/trac/ticket/2880), [#2964](https://www.sqlalchemy.org/trac/ticket/2964)

### SQL

+   **[SQL] [错误]**

    修复了调用`Insert.values()`时传入空列表或元组会引发 IndexError 的错误。现在会生成一个空的插入构造，就像使用空字典一样。

    此更改也**回溯**到：0.8.5

    参考文献：[#2944](https://www.sqlalchemy.org/trac/ticket/2944)

+   **[sql] [bug]**

    修复了 `ColumnOperators.in_()` 的 bug，如果错误地传递了包含 `__getitem__()` 方法的列表达式的比较器，例如使用 `ARRAY` 类型的列，将进入无限循环。

    这个改变也**被回溯到**：0.8.5

    参考文献：[#2957](https://www.sqlalchemy.org/trac/ticket/2957)

+   **[sql] [bug]**

    修复了新“命名约定”功能中的回归，在外键中的引用表包含模式名称时会导致约定失败的问题。感谢 Thomas Farvour 提供的拉取请求。

+   **[sql] [bug]**

    修复了所谓的“字面渲染” `bindparam()` 构造的 bug，如果绑定是用可调用对象而不是直接值构造的，则会失败。这阻止了 ORM 表达式使用“literal_binds”编译器标志进行渲染。

### postgresql

+   **[postgresql] [feature]**

    在 `ARRAY` 类型上添加了 `TypeEngine.python_type` 方便访问器。感谢 Alexey Terentev 提供的拉取请求。

+   **[postgresql] [bug]**

    向 psycopg2 断开连接检测添加了一个额外的消息，“无法发送数据到服务器”，这补充了现有的“无法从服务器接收数据”的消息，已被用户观察到。

    这个改变也**被回溯到**：0.8.5

    参考文献：[#2936](https://www.sqlalchemy.org/trac/ticket/2936)

+   **[postgresql] [bug]**

    > 改进了对非常旧（8.1 版本之前）版本的 PostgreSQL 以及可能的其他 PG 引擎（假设 Redshift 报告的版本 < 8.1）的 PostgreSQL 反射行为的支持。关于“索引”以及“主键”的查询依赖于检查所谓的“int2vector”数据类型，该类型在 8.1 之前拒绝强制转换为数组，导致查询中使用的“ANY()”运算符失败。通过广泛搜索，找到了一个非常糟糕但是被 PG 核心开发者推荐使用的查询，用于当 PG 版本 < 8.1 时，索引和主键约束反射现在可以在这些版本上正常工作。

    这个改变也**被回溯到**：0.8.5

+   **[postgresql] [bug]**

    修订了这个非常古老的问题，其中 PostgreSQL 的“获取主键”反射查询已更新，以考虑重命名的主键约束；新的查询在非常旧的 PostgreSQL 版本（如版本 7）上失败，因此在检测到 server_version_info < (8, 0) 的情况下，在这些情况下恢复旧的查询。

    这个改变也**被回溯到**：0.8.5

    参考：[#2291](https://www.sqlalchemy.org/trac/ticket/2291)

+   **[postgresql] [bug]**

    对于新添加的方言启动查询“show standard_conforming_strings”添加了服务器版本检测；由于此变量是从 PG 8.2 版本开始添加的，我们会跳过对于报告早于该版本的 PG 版本的查询。

    参考：[#2946](https://www.sqlalchemy.org/trac/ticket/2946)

### mysql

+   **[mysql] [feature]**

    添加了新的 MySQL 特定的 `DATETIME`，其中包括分数秒的支持；还为 `TIMESTAMP` 添加了分数秒的支持。尽管 DBAPI 支持有限，但 MySQL Connector/Python 已知支持分数秒。补丁由 Geert JM Vanderkelen 提供。

    此更改也被**回溯**到：0.8.5

    参考：[#2941](https://www.sqlalchemy.org/trac/ticket/2941)

+   **[mysql] [bug]**

    添加了对 `PARTITION BY` 和 `PARTITIONS` MySQL 表关键字的支持，指定为 `mysql_partition_by='value'` 和 `mysql_partitions='value'` 到 `Table`。感谢 Marcus McCurdy 的拉取请求。

    此更改也被**回溯**到：0.8.5

    参考：[#2966](https://www.sqlalchemy.org/trac/ticket/2966)

+   **[mysql] [bug]**

    修复了一个 bug，该 bug 阻止了基于 MySQLdb 的方言（例如 pymysql）在 Py3K 中工作，因为“connection charset”的检查会由于 Py3K 更严格的值比较规则而失败。在这种情况下，所调用的方法根本没有考虑数据库版本，因为服务器版本在那时仍然为 None，因此该方法已经简化为依赖于 connection.character_set_name()。

    此更改也被**回溯**到：0.8.5

    参考：[#2933](https://www.sqlalchemy.org/trac/ticket/2933)

+   **[mysql] [bug] [cymysql]**

    修复了 cymysql 方言中的一个 bug，即像 `'33a-MariaDB'` 这样的版本字符串无法正确解析。感谢 Matt Schmidt 的拉取请求。

    参考：[#2934](https://www.sqlalchemy.org/trac/ticket/2934)

### sqlite

+   **[sqlite] [bug]**

    SQLite 方言现在在反射类型时将跳过不支持的参数；例如，如果遇到像 `INTEGER(5)` 这样的字符串，将实例化 `INTEGER` 类型而不包含“5”，基于在第一次尝试时检测到 `TypeError`。

+   **[sqlite] [bug]**

    对 SQLite 类型反射添加了对完全支持在 [`www.sqlite.org/datatype3.html`](https://www.sqlite.org/datatype3.html) 指定的“类型亲和性”协议的支持。在这个方案中，类型名称中的关键字如 `INT`、`CHAR`、`BLOB` 或 `REAL` 通常将类型与五种亲和性之一关联起来。感谢 Erich Blume 的拉取请求。

    另请参阅

    类型反射

### 杂项

+   **[bug] [ext]**

    修复了新 automap 扩展的`AutomapBase`类在类被预先排列在单个或潜在的连接继承模式中时会失败的错误。修复的连接继承问题也可能在使用`DeferredReflection`时应用。

### orm

+   **[orm] [feature]**

    添加了新的`MapperEvents.before_configured()`事件，允许在`configure_mappers()`开始时触发事件，以及在声明性中补充`__declare_last__()`的`__declare_first__()`钩子。

+   **[orm] [bug]**

    修复了`Query.get()`在查询中存在现有条件时无法一致引发`InvalidRequestError`的错误，当给定的标识已经存在于标识映射中时。

    此更改也**回溯**到：0.8.5

    参考：[#2951](https://www.sqlalchemy.org/trac/ticket/2951)

+   **[orm] [bug] [sqlite]**

    修复了 SQLite“连接重写”中的错误，其中 exists()构造的使用未能正确重写，例如，当 exists 映射到复杂嵌套连接场景中的 column_property 时。还修复了一个有些相关的问题，即当目标是别名表而不是单独的别名列时，连接重写会在 SELECT 语句的 columns 子句上失败。

    参考：[#2967](https://www.sqlalchemy.org/trac/ticket/2967)

+   **[orm] [bug]**

    修复了 0.9 版本中的一个回归，即应用于基类（例如具有 propagate=True 标志的声明性基类）的 ORM 实例或映射器事件将无法应用于已存在的使用继承的映射类，因为断言失败。此外，修复了在删除此类事件时可能发生的属性错误，具体取决于首次分配方式。

    参考：[#2949](https://www.sqlalchemy.org/trac/ticket/2949)

+   **[orm] [bug]**

    改进了复合属性的初始化逻辑，使调用`MyClass.attribute`不需要配置映射器步骤，例如，它将正常工作而不会抛出任何错误。

    参考：[#2935](https://www.sqlalchemy.org/trac/ticket/2935)

+   **[orm] [bug]**

    在 0.9.2 中首次解决的更多问题，其中使用形式为`<tablename>_<columnname>`的列键与文本中的别名列匹配仍然不会在 ORM 级别匹配，这最终是由于核心列匹配问题。已添加额外规则，以便在使用`TextAsFrom`构造或文字列时考虑列`_label`。

    参考：[#2932](https://www.sqlalchemy.org/trac/ticket/2932)

### orm 声明式

+   **[orm] [declarative] [bug]**

    修复了`AbstractConcreteBase`在声明关系配置中无法完全可用的错误，因为其字符串类名在映射器配置时不可用。该类现在明确将自己添加到类注册表中，并且`AbstractConcreteBase`以及`ConcreteBase`在映射器配置之前*之前*设置自己，在`configure_mappers()`设置中，使用新的`MapperEvents.before_configured()`事件。

    参考：[#2950](https://www.sqlalchemy.org/trac/ticket/2950)

### 示例

+   **[examples] [feature]**

    在版本化行示例中添加了可选的“changed”列，以及当版本化`Table`具有显式`Table.schema`参数时的支持。拉取��求由 jplaverdure 提供。

### 引擎

+   **[engine] [bug] [pool]**

    修复了由[#2880](https://www.sqlalchemy.org/trac/ticket/2880)引起的关键回归，其中新的并发能力从池中返回连接意味着“first_connect”事件现在也不再同步，从而在即使是最小并发情况下也会导致方言配置错误。

    此更改也**回溯**到：0.8.5

    参考：[#2880](https://www.sqlalchemy.org/trac/ticket/2880)，[#2964](https://www.sqlalchemy.org/trac/ticket/2964)

### sql

+   **[sql] [bug]**

    修复了调用带有空列表或元组的`Insert.values()`会引发 IndexError 的错误。现在它会生成一个空的插入构造，就像使用空字典的情况一样。

    此更改也**回溯**到：0.8.5

    参考：[#2944](https://www.sqlalchemy.org/trac/ticket/2944)

+   **[sql] [bug]**

    修复了`ColumnOperators.in_()`会陷入无限循环的错误，如果错误地传递了一个包含`__getitem__()`方法的列表达式的比较器，比如一个使用`ARRAY`类型的列。

    这个变更也被**回溯**到：0.8.5

    参考：[#2957](https://www.sqlalchemy.org/trac/ticket/2957)

+   **[sql] [bug]**

    修复了新“命名约定”功能中的回归错误，在外键中引用的表包含模式名称时，约定会失败。感谢 Thomas Farvour 的拉取请求。

+   **[sql] [bug]**

    修复了所谓的`bindparam()`构造的“字面渲染”失败的错误，如果绑定是用可调用方式构造的，而不是直接值。这会阻止 ORM 表达式使用“literal_binds”编译器标志进行渲染。

### postgresql

+   **[postgresql] [feature]**

    在`ARRAY`类型上添加了`TypeEngine.python_type`的便利访问器。感谢 Alexey Terentev 的拉取请求。

+   **[postgresql] [bug]**

    添加了对 psycopg2 断开连接检测的额外消息，“无法发送数据到服务器”，这补充了现有的“无法从服务器接收数据”并被用户观察到。

    这个变更也被**回溯**到：0.8.5

    参考：[#2936](https://www.sqlalchemy.org/trac/ticket/2936)

+   **[postgresql] [bug]**

    > 对于非常旧的（8.1 之前）PostgreSQL 版本以及可能的其他 PG 引擎（假设 Redshift 报告的版本为 < 8.1），改进了对 PostgreSQL 反射行为的支持。关于“索引”和“主键”的查询依赖于检查所谓的“int2vector”数据类型，它在 8.1 之前拒绝强制转换为数组，导致查询中使用的“ANY()”运算符失败。通过广泛的搜索，找到了非常笨拙但由 PG 核心开发人员推荐使用的查询，用于当 PG 版本 < 8.1 时，索引和主键约束反射现在在这些版本上工作。

    这个变更也被**回溯**到：0.8.5

+   **[postgresql] [bug]**

    修正了这个非常古老的问题，即 PostgreSQL “获取主键”反射查询已更新以考虑已重命名的主键约束；新的查询在旧版本的 PostgreSQL 中（如版本 7）失败，因此在检测到 server_version_info < (8, 0) 时，恢复了旧的查询。

    这个变更也被**回溯**到：0.8.5

    参考：[#2291](https://www.sqlalchemy.org/trac/ticket/2291)

+   **[postgresql] [bug]**

    在新添加的方言启动查询中添加了服务器版本检测，用于“show standard_conforming_strings”；由于此变量是从 PG 8.2 开始添加的，我们会跳过报告版本字符串早于此的 PG 版本的查询。

    参考：[#2946](https://www.sqlalchemy.org/trac/ticket/2946)

### mysql

+   **[mysql] [feature]**

    添加了新的 MySQL 特定的 `DATETIME`，其中包括分数秒支持；还为 `TIMESTAMP` 添加了分数秒支持。尽管 DBAPI 支持有限，但分数秒已知受到 MySQL Connector/Python 的支持。补丁由 Geert JM Vanderkelen 提供。

    此更改也被**回溯**到：0.8.5

    参考：[#2941](https://www.sqlalchemy.org/trac/ticket/2941)

+   **[mysql] [bug]**

    添加了对 `PARTITION BY` 和 `PARTITIONS` MySQL 表关键字的支持，指定为 `mysql_partition_by='value'` 和 `mysql_partitions='value'` 到 `Table`。拉取请求由 Marcus McCurdy 提供。

    此更改也被**回溯**到：0.8.5

    参考：[#2966](https://www.sqlalchemy.org/trac/ticket/2966)

+   **[mysql] [bug]**

    修复了阻止基于 MySQLdb 的方言（例如 pymysql）在 Py3K 中工作的错误，其中“连接字符集”检查会由于 Py3K 更严格的值比较规则而失败。在任何情况下，所涉及的调用都没有考虑数据库版本，因为服务器版本在那时仍然为 None，因此该方法已经简化为依赖于 connection.character_set_name()。

    此更改也被**回溯**到：0.8.5

    参考：[#2933](https://www.sqlalchemy.org/trac/ticket/2933)

+   **[mysql] [bug] [cymysql]**

    修复了 cymysql 方言中的错误，其中版本字符串如 `'33a-MariaDB'` 无法��确解析。拉取请求由 Matt Schmidt 提供。

    参考：[#2934](https://www.sqlalchemy.org/trac/ticket/2934)

### sqlite

+   **[sqlite] [bug]**

    SQLite 方言现在在反射类型时会跳过不支持的参数；例如，如果遇到类似 `INTEGER(5)` 的字符串，`INTEGER` 类型将在实例化时不包含“5”，基于在第一次尝试时检测到 `TypeError`。

+   **[sqlite] [bug]**

    已添加对 SQLite 类型反射的支持，以完全支持在 [`www.sqlite.org/datatype3.html`](https://www.sqlite.org/datatype3.html) 中指定的“类型亲和性”协议。在此方案中，类型名称中的关键字如 `INT`、`CHAR`、`BLOB` 或 `REAL` 通常将类型与五种亲和性之一关联起来。拉取请求由 Erich Blume 提供。

    另请参阅

    类型反射

### 杂项

+   **[bug] [ext]**

    修复了新自动映射扩展的`AutomapBase`类在类被预先排列在单个或潜在的联合继承模式中时会失败的错误。修复的联合继承问题也可能适用于使用`DeferredReflection`时。

## 0.9.2

发布日期：2014 年 2 月 2 日

### orm

+   **[orm] [feature]**

    添加了一个新参数`Operators.op.is_comparison`。此标志允许来自`Operators.op()`的自定义操作被视为“比较”运算符，因此可用于自定义`relationship.primaryjoin`条件。

    另请参阅

    在连接条件中使用自定义操作符

+   **[orm] [feature]**

    改进了为`relationship.secondary`的目标提供`join()`构造的支持，以便创建非常复杂的`relationship()`连接条件。此更改包括对查询连接、连接的急加载进行调整，以避免渲染 SELECT 子查询，对延迟加载进行更改，以便正确包含“次要”目标在 SELECT 中，并对声明性进行更改以更好地支持将 join()对象与类作为目标的规范。

    新用例有些实验性，但已添加了新的文档部分。

    另请参阅

    复合“次要”连接

+   **[orm] [bug]**

    修复了当将迭代器对象传递给`class_mapper()`或类似函数时出现的错误消息，该错误会导致字符串格式化时无法呈现错误。感谢 Kyle Stark 提交的 Pullreq。

    此更改也**回溯**至：0.8.5

+   **[orm] [bug]**

    在新的`TextAsFrom`结构中修复了一个错误，其中`Column`导向的行查找与`TextAsFrom`生成的临时`ColumnClause`对象不匹配，因此在`Query.from_statement()`中无法使用。还修复了`Query.from_statement()`的机制，以防将`TextAsFrom`错误地误认为是`Select`构造。此错误也是 0.9 的回归，因为需要调用`TextClause.columns()`方法来适应`text.typemap`参数。

    引用：[#2932](https://www.sqlalchemy.org/trac/ticket/2932)

+   **[orm] [bug]**

    在属性“set”操作范围内添加了一个新指令，用于禁用自动刷新，在属性需要延迟加载“旧”值的情况下使用，例如替换一对一值或某些类型的一对多值。在此时刷新否则会在属性为 None 时发生，并可能导致 NULL 违规。

    引用：[#2921](https://www.sqlalchemy.org/trac/ticket/2921)

+   **[orm] [bug]**

    修复了 0.9 中的一个回归，即`Query`自动别名和在其他情况下别名（例如联接表继承）可能会失败，如果表达式中使用了用户定义的`Column`子类。在这种情况下，子类将无法传播 ORM 特定的“注解”，这些注解由适配器所需。已对“表达式注解”系统进行了修正，以解决此问题。

    引用：[#2918](https://www.sqlalchemy.org/trac/ticket/2918)

+   **[orm] [bug]**

    修复了一个关于新的扁平化 JOIN 结构的错误，该结构与`joinedload()`一起使用（从而导致连接贪婪加载中的回归），以及与`flat=True`标志和联接表继承一起使用的`aliased()`；基本上，通过不同路径跨越“父 JOIN 子”实体进行多次连接以到达目标类的多个连接不会形成正确的 ON 条件。在处理 aliased、joined-inh 类的“左侧”连接的机制中进行了调整/简化，修复了此问题。

    参考：[#2908](https://www.sqlalchemy.org/trac/ticket/2908)

### 示例

+   **[示例] [错误]**

    对“history_meta”示例进行了微调，现在在关系绑定属性上检查“history”时，如果关系未加载，则不会再发出任何 SQL。

### 引擎

+   **[引擎] [特性] [池]**

    添加了一个新的池事件`PoolEvents.invalidate()`。当要将 DBAPI 连接标记为“无效”并从池中丢弃时调用。

### sql

+   **[sql] [特性]**

    添加了`MetaData.reflect.dialect_kwargs`，以支持为所有反射的`Table`对象设置方言级别的反射选项。

+   **[sql] [特性]**

    添加了一个新功能，允许自动命名约定应用于`Constraint`和`Index`对象。基于维基中的一个示例，新功能使用模式事件来设置名称，因为各种模式对象相互关联。然后，事件通过一个新参数`MetaData.naming_convention`公开一个配置系统。该系统允许在每个`MetaData`基础上生成简单和自定义的约束和索引命名方案。

    另请参阅

    配置约束命名约定

    参考：[#2923](https://www.sqlalchemy.org/trac/ticket/2923)

+   **[sql] [特性]**

    现在可以在`PrimaryKeyConstraint`对象上独立于使用`primary_key=True`标志在表中指定列的情况下指定选项；使用一个不包含列的`PrimaryKeyConstraint`对象即可实现此结果。

    以前，显式的`PrimaryKeyConstraint`会导致将标记为`primary_key=True`的列被忽略；由于不再这样，`PrimaryKeyConstraint`现在将断言使用其中一种样式来指定列，或者如果两者都存在，则列列表必须完全匹配。如果在`PrimaryKeyConstraint`和在`Table`中标记为`primary_key=True`的不一致列集存在，则会发出警告，并且列列表仅从`PrimaryKeyConstraint`中获取，就像在先前的版本中一样。

    另请参阅

    `PrimaryKeyConstraint` 

    参考：[#2910](https://www.sqlalchemy.org/trac/ticket/2910)

+   **[sql] [feature]**

    架构构造和某些 SQL 构造接受特定方言关键字参数的系统已得到增强。这个系统通常包括`Table`和`Index`构造，它们接受各种各样的特定方言参数，如`mysql_engine`和`postgresql_where`，以及构造`PrimaryKeyConstraint`、`UniqueConstraint`、`Update`、`Insert`和`Delete`，还新增了对`ForeignKeyConstraint`和`ForeignKey`的关键字参数功能。改变在于参与的方言现在可以为这些构造指定可接受的参数列表，允许在为特定方言指定无效关键字时引发参数错误。如果关键字的方言部分未被识别，只会发出警告；虽然系统实际上会利用 setuptools 入口点来定位非本地方言，但在未安装第三方方言的环境中使用某些特定方言参数的用例仍然受支持。方言还必须明确选择加入这个系统，因此不使用这个系统的外部方言将不受影响。

    参考：[#2866](https://www.sqlalchemy.org/trac/ticket/2866)

+   **[sql] [bug]**

    `Table.tometadata()` 的行为已经调整，以便当`ForeignKey`的模式目标不匹配父表的模式时，不会更改。也就是说，如果一个表“schema_a.user”有一个指向“schema_b.order.id”的外键，无论是否将“schema”参数传递给`Table.tometadata()`，都将保持“schema_b”目标。但是，如果一个表“schema_a.user”引用“schema_a.order.id”，则“schema_a”将在父表和被引用表上更新。这是一个行为变更，因此不太可能被回溯到 0.8；可以假设以前的行为相当错误，而且很少有人依赖它。

    另外，新增了一个参数`Table.tometadata.referred_schema_fn`。这是一个可调用函数，用于确定在 tometadata 操作中遇到的任何`ForeignKeyConstraint`的新引用模式。这个可调用函数可以用于恢复到以前的行为，或者根据每个约束自定义如何处理引用模式。

    参考：[#2913](https://www.sqlalchemy.org/trac/ticket/2913)

+   **[sql] [bug]**

    修复了在某些情况下，如果与“test”方言（例如 DefaultDialect 或其他没有 DBAPI 的方言）一起使用二进制类型会失败的 bug。

+   **[sql] [bug] [py3k]**

    修复了“literal binds”无法与绑定参数是二进制类型的 bug。在 0.8 中修复了类似但不同的问题。

+   **[sql] [bug]**

    修复了 ORM 使用的“注释”系统泄漏到`sqlalchemy.sql.functions`中标准函数（如`func.coalesce()`和`func.max()`）使用的名称中的回归问题。在 ORM 属性中使用这些函数，从而生成它们的带注释版本，可能会破坏在 SQL 中呈现的实际函数名称。

    参考：[#2927](https://www.sqlalchemy.org/trac/ticket/2927)

+   **[sql] [bug]**

    修复了 0.9 版本中新的`RowProxy`可排序支持导致与非元组类型比较时会导致`TypeError`的回归问题，因为它试图无条件地应用 tuple()到“other”对象。现在在`RowProxy`上实现了完整的 Python 比较运算符范围，使用一种保证等效于元组的比较系统的方法，只有在“other”对象是 RowProxy 的实例时才会强制转换。

    参考：[#2848](https://www.sqlalchemy.org/trac/ticket/2848), [#2924](https://www.sqlalchemy.org/trac/ticket/2924)

+   **[sql] [bug]**

    与一个没有列的 `Table` 内联创建的 `UniqueConstraint` 将被跳过。Pullreq 由 Derek Harland 提供。

+   **[sql] [bug] [orm]**

    修复了多表“UPDATE..FROM”构造，在 MySQL 上可用，以正确地在跨表中的多个相同名称的列之间渲染 SET 子句。这也将非主表中用于 SET 子句中的绑定参数的名称更改为“<tablename>_<colname>”；由于通常直接使用 `Column` 对象指定此参数，这不应对应用程序产生影响。此修复对 ORM 中的 `Table.update()` 和 `Query.update()` 都生效。

    参考：[#2912](https://www.sqlalchemy.org/trac/ticket/2912)

### schema

+   **[schema] [bug]**

    将 `sqlalchemy.schema.SchemaVisitor` 恢复到 `.schema` 模块中。Pullreq 由 Sean Dague 提供。

### postgresql

+   **[postgresql] [feature]**

    添加了一个新的方言级参数 `postgresql_ignore_search_path`；该参数被 `Table` 构造函数和 `MetaData.reflect()` 方法都接受。当用于 PostgreSQL 时，指定了远程模式名称的外键引用表将保留该模式名称，即使该名称存在于 `search_path` 中；自 0.7.3 以来的默认行为是，`search_path` 中存在的模式不会复制到反射的 `ForeignKey` 对象中。文档已更新以详细描述 `pg_get_constraintdef()` 函数的行为以及 `postgresql_ignore_search_path` 特性如何实质上决定我们是否会尊重此函数报告的模式资格。

    另见

    远程模式表内省和 PostgreSQL search_path

    参考：[#2922](https://www.sqlalchemy.org/trac/ticket/2922)

### mysql

+   **[mysql] [bug]**

    为 cymysql 方言添加了一些缺失的方法，包括 _get_server_version_info() 和 _detect_charset()。Pullreq 由 Hajime Nakagami 提供。

    此更改也被 **回溯** 到：0.8.5

+   **[mysql] [bug] [sql]**

    为所谓的 SQL 类型的“向下适应”添加了新的测试覆盖，其中更具体的类型被适应为更通用的类型 - 这种用例被一些第三方工具（如`sqlacodegen`）所需。在此测试套件中需要修复的特定情况是将`ENUM`降级为`Enum`，以及将 SQLite 日期类型转换为通用日期类型。`adapt()`方法需要在这里变得更具体，以抵消在 0.9 中删除的基本`TypeEngine`类上的“捕获所有”`**kwargs`集合的移除。

    参考：[#2917](https://www.sqlalchemy.org/trac/ticket/2917)

+   **[mysql] [bug]**

    现在 MySQL CAST 编译考虑了字符串类型的“字符集”和“排序”。虽然 MySQL 希望所有基于字符的 CAST 调用都使用 CHAR 类型，但我们现在在 CAST 时创建一个真正的 CHAR 对象，并复制它具有的所有参数，因此像`cast(x, mysql.TEXT(charset='utf8'))`这样的表达式将呈现为`CAST(t.col AS CHAR CHARACTER SET utf8)`。

+   **[mysql] [bug]**

    对 MySQL 方言和整体默认方言系统添加了新的“unicode 返回”检测，以便任何方言都可以在首次连接时添加额外的“测试”来检测 DBAPI 是否直接返回 unicode。在这种情况下，我们特别针对“utf8”编码添加了一个检查，使用显式的“utf8_bin”排序类型（在检查此排序是否可用后）来测试观察到的 MySQLdb 版本 1.2.3 存在的一些错误的 unicode 行为。虽然 MySQLdb 在 1.2.4 中已解决了此问题，但此处的检查应该防止出现退化。此更改还允许“unicode”检查记录在引擎日志中，这以前是不可能的。

    参考：[#2906](https://www.sqlalchemy.org/trac/ticket/2906)

+   **[mysql] [bug] [engine] [pool]**

    `Connection` 现在将一个新的 `RootTransaction` 或 `TwoPhaseTransaction` 与其立即的 `_ConnectionFairy` 关联为该事务的“复位处理程序”，在该事务的范围内接管调用 commit() 或 rollback() 来处理 `Pool` 的“返回时重置”行为，如果事务没有被其他方式完成。这解决了当连接关闭时，像 MySQL 两阶段那样挑剔的事务将被正确关闭的问题，而不需要显式的回滚或提交（例如，在此情况下不再引发“XAER_RMFAIL” - 请注意，这仅显示在日志中，因为异常未在池重置中传播）。例如，当使用带有 `twophase` 设置的 orm `Session`，然后在没有显式回滚或提交的情况下调用 `Session.close()` 时，会出现此问题。该更改还具有的效果是，现在在非自动提交模式下使用 `Session` 对象时，无论该会话是如何被丢弃的，都会在日志中看到显式的“ROLLBACK”。感谢 Jeff Dairiki 和 Laurence Rowe 在此处隔离问题。

    参考资料：[#2907](https://www.sqlalchemy.org/trac/ticket/2907)

### sqlite

+   **[sqlite] [bug]**

    修复了一个错误，即 SQLite 编译器无法将编译器参数（例如“literal binds”）传播到 CAST 表达式中的情况。

### mssql

+   **[mssql] [feature]**

    向 `UniqueConstraint` 和 `PrimaryKeyConstraint` 构造添加了一个选项 `mssql_clustered`；在 SQL Server 上，这将在 DDL 中将 `CLUSTERED` 关键字添加到约束构造中。感谢 Derek Harland 提供了 Pullreq。

### oracle

+   **[oracle] [bug]**

    已经观察到，在 Python 2.xx 中使用 cx_Oracle 的 “outputtypehandler” 来将字符串值强制转换为 Unicode 是非常昂贵的；即使 cx_Oracle 是用 C 编写的，当你将 Python `unicode` 原语传递给 cursor.var() 并与输出处理程序关联时，库会将每次转换都记录为 Python 函数调用，带有所有必要的开销；尽管在 Python 3 中运行时，所有字符串也被无条件地强制转换为 unicode，但它不会产生这种开销，这意味着 cx_Oracle 在 Py2K 中未能使用高效的技术。由于 SQLAlchemy 无法轻松地按列选择这种类型处理程序，因此处理程序是无条件地组装的，从而为所有字符串访问添加了开销。

    因此，这个逻辑已被替换为 SQLAlchemy 自己的 unicode 转换系统，现在只在请求为 unicode 的列中对 Py2K 生效。当使用 C 扩展时，SQLAlchemy 的系统似乎比 cx_Oracle 的快 2-3 倍。此外，SQLAlchemy 的 unicode 转换已经得到增强，以便在需要“条件”转换器（现在需要用于 Oracle 后端）时，在 C 中执行“已经是 unicode”的检查，不再引入显著的开销。

    这个变化对 cx_Oracle 后端有两个影响。一个是在 Py2K 中，未明确请求 Unicode 类型或 convert_unicode=True 的字符串值现在将返回为 `str`，而不是 `unicode` - 这种行为类似于像 MySQL 这样的后端。此外，当使用 cx_Oracle 后端请求 unicode 值时，如果不使用 C 扩展，则现在每列都会有一个 isinstance() 检查的额外开销。这种权衡已经做出，因为它可以被解决，并且不再对大多数非 unicode 字符串的 Oracle 结果列产生性能负担。

    参考：[#2911](https://www.sqlalchemy.org/trac/ticket/2911)

### 杂项

+   **[bug] [pool]**

    `PoolEvents.reset()` 事件的参数名称已更改为 `dbapi_connection` 和 `connection_record`，以保持与所有其他池事件的一致性。预计任何现有监听器对于这个相对较新且很少使用的事件都是使用位置样式来接收参数。

+   **[bug] [cextensions] [py3k]**

    修复了一个问题，即 Py3K 中的 C 扩展正在使用错误的 API 来指定顶级模块函数，这在 Python 3.4b2 中会出错。Py3.4b2 将 PyMODINIT_FUNC 更改为返回“void”，而不是 `PyObject *`，因此我们现在确保直接使用“PyMODINIT_FUNC”而不是 `PyObject *`。感谢 cgohlke 提交的拉取请求。

### orm

+   **[orm] [feature]**

    添加了一个新参数`Operators.op.is_comparison`。此标志允许从`Operators.op()`中选择自定义运算符作为“比较”运算符，因此可用于自定义`relationship.primaryjoin`条件。

    另请参阅

    在连接条件中使用自定义运算符

+   **[orm] [feature]**

    改进了将`join()`构造作为`relationship.secondary`的目标以创建非常复杂的`relationship()`连接条件的支持。该更改包括对查询连接、连接的急加载进行调整，以避免渲染 SELECT 子查询，对延迟加载进行更改，以便正确地在 SELECT 中包含“次要”目标，并对声明性进行更改以更好地支持使用类作为目标的 join() 对象的规范。

    新的用例有些实验性，但已添加了新的文档部分。

    另请参阅

    复合“次要”连接

+   **[orm] [bug]**

    修复了当将迭代器对象传递给`class_mapper()`或类似方法时出现的错误消息，该错误会导致字符串格式化时无法呈现。感谢 Kyle Stark 提交的 Pullreq。

    此更改也已**回溯**至：0.8.5

+   **[orm] [bug]**

    修复了新的`TextAsFrom`构造中的错误，其中`Column` - 导向行查找未与`TextAsFrom`生成的临时`ColumnClause`对象相匹配，因此不能在`Query.from_statement()`中使用。同时修复了`Query.from_statement()`机制，以免将`TextAsFrom`错误地视为`Select`构造。该错误也是 0.9 的一个回归，因为要适应`text.typemap`参数，`TextClause.columns()`方法被调用。

    参考：[#2932](https://www.sqlalchemy.org/trac/ticket/2932)

+   **[orm] [bug]**

    添加了一个新的指令，在属性“设置”操作的范围内使用，以禁用自动刷新，在属性需要惰性加载“旧”值的情况下使用，例如替换一对一值或某些类型的一对多。此时刷新通常在属性为 None 时发生，并且可能会导致 NULL 违规。

    参考：[#2921](https://www.sqlalchemy.org/trac/ticket/2921)

+   **[orm] [bug]**

    修复了 0.9 版本中的一个回归，即`Query`自动别名化以及在其他情况下选择或连接被别名化（例如连接表继承）可能会失败，如果表达式中使用了用户定义的`Column`子类。在这种情况下，子类将无法传播需要适应的 ORM 特定的“注解”。已经修正了“表达式注解”系统以解决此问题。

    参考：[#2918](https://www.sqlalchemy.org/trac/ticket/2918)

+   **[orm] [bug]**

    修复了一个关于新的扁平化 JOIN 结构的错误，该结构与`joinedload()`（从而导致连接贪婪加载的回归）以及与`flat=True`标志和联接表继承一起使用的`aliased()`相关的错误；基本上，通过不同路径跨越“父 JOIN 子”实体进行多次连接以到达目标类的多个连接不会形成正确的 ON 条件。在处理 aliased、joined-inh 类的“左侧”连接的机制中进行了调整/简化，修复了这个问题。

    参考：[#2908](https://www.sqlalchemy.org/trac/ticket/2908)

### 例子

+   **[examples] [bug]**

    对“history_meta”示例进行了调整，当关系绑定属性上的“history”检查不再发出任何 SQL 时，如果关系未加载。

### engine

+   **[engine] [feature] [pool]**

    添加了一个新的池事件`PoolEvents.invalidate()`。当一个 DBAPI 连接被标记为“无效”并且从池中丢弃时调用。

### sql

+   **[sql] [feature]**

    添加了`MetaData.reflect.dialect_kwargs`以支持为所有反射的`Table`对象设置方言级别的反射选项。

+   **[sql] [feature]**

    添加了一个新功能，允许自动命名约定应用于`Constraint`和`Index`对象。基于维基百科中的一个配方，新功能使用模式事件来设置名称，当各种模式对象相互关联时。然后，事件通过一个新参数`MetaData.naming_convention`暴露一个配置系统。该系统允许在每个`MetaData`基础上生成简单和自定义的约束和索引命名方案。

    另请参阅

    配置约束命名约定

    参考：[#2923](https://www.sqlalchemy.org/trac/ticket/2923)

+   **[sql] [feature]**

    现在可以在`PrimaryKeyConstraint`对象上独立指定选项，而不需要在表中的列上使用`primary_key=True`标志；使用一个不包含列的`PrimaryKeyConstraint`对象即可实现此目的。

    以前，显式的`PrimaryKeyConstraint`会导致那些标记为`primary_key=True`的列被忽略；由于现在不再这样，`PrimaryKeyConstraint`现在会断言要么使用一种风格指定列，要么使用另一种风格，或者如果两者都存在，则列列表必须完全匹配。如果在`PrimaryKeyConstraint`和在`Table`中标记为`primary_key=True`的不一致列集存在，则会发出警告，并且列列表仅从`PrimaryKeyConstraint`中获取，就像在先前的版本中一样。

    另请参阅

    `PrimaryKeyConstraint`

    参考：[#2910](https://www.sqlalchemy.org/trac/ticket/2910)

+   **[sql] [feature]**

    架构构造和某些 SQL 构造接受特定方言的关键字参数的系统已经得到增强。这个系统通常包括`Table`和`Index`构造，它们接受各种特定方言的参数，比如`mysql_engine`和`postgresql_where`，以及`PrimaryKeyConstraint`、`UniqueConstraint`、`Update`、`Insert`和`Delete`等构造，还新增了对`ForeignKeyConstraint`和`ForeignKey`的关键字参数功能。改变之处在于参与的方言现在可以为这些构造指定可接受的参数列表，允许在为特定方言指定无效关键字时引发参数错误。如果关键字的方言部分未被识别，只会发出警告；虽然系统实际上会利用 setuptools 的 entrypoints 来定位非本地方言，但在未安装第三方方言的环境中使用某些特定方言参数的用例仍然受支持。方言还必须明确选择加入这个系统，因此不使用这个系统的外部方言将不受影响。

    参考：[#2866](https://www.sqlalchemy.org/trac/ticket/2866)

+   **[sql] [bug]**

    `Table.tometadata()`的行为已经调整，以便在不匹配父表模式的情况下不会更改`ForeignKey`的模式目标。也就是说，如果一个表“schema_a.user”有一个指向“schema_b.order.id”的外键，无论是否将“schema”参数传递给`Table.tometadata()`，都将保持“schema_b”目标。但是，如果一个表“schema_a.user”引用“schema_a.order.id”，则“schema_a”将在父表和引用表上更新。这是一个行为变更，因此不太可能被回溯到 0.8；可以假设以前的行为相当错误，而且很少有人依赖它。

    另外，新增了一个参数`Table.tometadata.referred_schema_fn`。这是一个可调用函数，用于确定在 tometadata 操作中遇到的任何`ForeignKeyConstraint`的新引用模式。这个可调用函数可以用于恢复到以前的行为或者自定义如何处理每个约束的引用模式。

    参考：[#2913](https://www.sqlalchemy.org/trac/ticket/2913)

+   **[sql] [bug]**

    修复了二进制类型在某些情况下与“test”方言（例如 DefaultDialect 或其他没有 DBAPI 的方言）一起使用时会失败的 bug。

+   **[sql] [bug] [py3k]**

    修复了“literal binds”无法与绑定参数为二进制类型的 bug。0.8 中修复了类似但不同的问题。

+   **[sql] [bug]**

    修复了 ORM 使用的“注释”系统泄漏到`sqlalchemy.sql.functions`中标准函数（如`func.coalesce()`和`func.max()`）使用的名称中的回归问题。在 ORM 属性中使用这些函数，从而生成它们的带注释版本，可能会破坏在 SQL 中呈现的实际函数名称。

    参考：[#2927](https://www.sqlalchemy.org/trac/ticket/2927)

+   **[sql] [bug]**

    修复了 0.9 中的回归问题，新的`RowProxy`可排序支持会导致与非元组类型比较时引发`TypeError`，因为它试图无条件地应用 tuple()到��other”对象。现在在`RowProxy`上实现了完整的 Python 比较运算符范围，使用一种保证等效于元组的比较系统的方法，只有当“other”对象是`RowProxy`的实例时才会强制转换。

    参考：[#2848](https://www.sqlalchemy.org/trac/ticket/2848), [#2924](https://www.sqlalchemy.org/trac/ticket/2924)

+   **[sql] [bug]**

    在没有列的`Table`中内联创建的`UniqueConstraint`将被跳过。Pullreq 感谢 Derek Harland。

+   **[sql] [bug] [orm]**

    修复了多表“UPDATE..FROM”结构，仅适用于 MySQL，在多个表中正确呈现 SET 子句的问题。这也更改了 SET 子句中用于非主表的绑定参数的名称为“<tablename>_<colname>”；由于该参数通常是直接使用`Column`对象指定的，因此这不会对应用程序产生影响。此修复对 ORM 中的`Table.update()`和`Query.update()`都生效。

    参考：[#2912](https://www.sqlalchemy.org/trac/ticket/2912)

### schema

+   **[schema] [bug]**

    恢复了`sqlalchemy.schema.SchemaVisitor`到`.schema`模块。Pullreq 感谢 Sean Dague。

### postgresql

+   **[postgresql] [feature]**

    新增了一个方言级别的参数`postgresql_ignore_search_path`；该参数被`Table`构造函数和`MetaData.reflect()`方法同时接受。在针对 PostgreSQL 使用时，如果外键引用的表指定了远程模式名称，即使该名称存在于`search_path`中，也会保留该模式名称；自 0.7.3 以来的默认行为是，`search_path`中存在的模式不会被复制到反映的`ForeignKey`对象中。文档已经更新，详细描述了`pg_get_constraintdef()`函数的行为以及`postgresql_ignore_search_path`特性如何确定我们是否会尊重此函数报告的模式限定。

    另请参阅

    远程模式表内省和 PostgreSQL search_path

    参考：[#2922](https://www.sqlalchemy.org/trac/ticket/2922)

### mysql

+   **[mysql] [bug]**

    向 cymysql 方言添加了一些缺失的方法，包括 _get_server_version_info() 和 _detect_charset()。Pullreq 感谢 Hajime Nakagami。

    此更改也已**回溯**至：0.8.5

+   **[mysql] [bug] [sql]**

    为所谓的 SQL 类型的“向下适配”添加了新的测试覆盖，其中更具体的类型被适配为更通用的类型 - 这种用例被一些第三方工具（如 `sqlacodegen`）所需。在这个测试套件中需要修复的具体情况包括将 `ENUM` 转换为 `Enum`，以及将 SQLite 日期类型转换为通用日期类型。`adapt()` 方法在这里需要变得更具体，以抵消在 0.9 版本中从基本 `TypeEngine` 类中删除的“捕获所有”`**kwargs` 集合。 

    参考：[#2917](https://www.sqlalchemy.org/trac/ticket/2917)

+   **[mysql] [bug]**

    MySQL CAST 编译现在考虑了字符串类型的“字符集”和“校对规则”等方面。虽然 MySQL 希望所有基于字符的 CAST 调用都使用 CHAR 类型，但我们现在在 CAST 时创建一个真正的 CHAR 对象，并复制它所有的参数，因此像 `cast(x, mysql.TEXT(charset='utf8'))` 这样的表达式将会渲染为 `CAST(t.col AS CHAR CHARACTER SET utf8)`。

+   **[mysql] [bug]**

    向 MySQL 方言和默认方言系统整体添加了新的“unicode 返回”检测，以便任何方言都可以向首次连接时添加额外的“测试”来检测“这个 DBAPI 是否直接返回 unicode”。在这种情况下，我们特别针对“utf8”编码添加了一个检查，带有显式的“utf8_bin”校对类型（在检查此校对是否可用后），以测试观察到的 MySQLdb 版本 1.2.3 存在的一些错误的 unicode 行为。虽然 MySQLdb 已在 1.2.4 版本中解决了这个问题，但这里的检查应该防范回归。此更改还允许“unicode”检查记录在引擎日志中，这在以前是不可能的。

    参考：[#2906](https://www.sqlalchemy.org/trac/ticket/2906)

+   **[mysql] [bug] [engine] [pool]**

    `Connection` 现在将一个新的 `RootTransaction` 或 `TwoPhaseTransaction` 与其直接的 `_ConnectionFairy` 关联为该事务的“重置处理程序”，负责在事务期间调用 commit() 或 rollback() 以实现 `Pool` 的“返回时重置”行为，如果事务未被其他方式完成，则接管该任务。当连接关闭而没有显式的回滚或提交时，解决了像 MySQL 两阶段这样挑剔的事务将被正确关闭的问题（例如，在此情况下不再引发“XAER_RMFAIL” - 请注意，这只会显示在日志中，因为异常不会在池重置中传播）。当使用设置了 `twophase` 的 orm `Session`，然后在没有显式的回滚或提交的情况下调用 `Session.close()` 时，会出现此问题。该更改还具有以下效果：无论会话如何被丢弃，在非自动提交模式下使用 `Session` 对象时，您现在都将在日志中看到显式的“ROLLBACK”。感谢 Jeff Dairiki 和 Laurence Rowe 在此处分离问题。

    参考：[#2907](https://www.sqlalchemy.org/trac/ticket/2907)

### sqlite

+   **[sqlite] [bug]**

    修复了 SQLite 编译器无法将诸如“literal binds”之类的编译器参数传递到 CAST 表达式中的错误。

### mssql

+   **[mssql] [feature]**

    为 `UniqueConstraint` 和 `PrimaryKeyConstraint` 构造添加了一个 `mssql_clustered` 选项；在 SQL Server 上，这将在 DDL 中的约束结构中添加 `CLUSTERED` 关键字。感谢 Derek Harland 的 Pullreq。

### oracle

+   **[oracle] [bug]**

    已经观察到在 Python 2.xx 中使用 cx_Oracle 的“outputtypehandler”来将字符串值强制转换为 Unicode 是非常昂贵的；即使 cx_Oracle 是用 C 写的，当你将 Python `unicode` 原语传递给 cursor.var() 并与输出处理程序关联时，库会将每次转换都记录为 Python 函数调用，带有所有必要的开销；尽管在 Python 3 中运行时，所有字符串也被无条件地转换为 Unicode，但它不会产生这种开销，这意味着 cx_Oracle 在 Py2K 中未能使用高性能技术。由于 SQLAlchemy 不能轻松地按列选择这种类型处理程序的方式，处理程序被无条件地组装，从而为所有字符串访问增加了开销。

    因此，这个逻辑已被 SQLAlchemy 自己的 Unicode 转换系统取代，现在只在请求为 Unicode 的列中在 Py2K 中生效。当使用 C 扩展时，SQLAlchemy 的系统似乎比 cx_Oracle 的快 2-3 倍。此外，SQLAlchemy 的 Unicode 转换已经得到增强，以便当需要“条件”转换器（现在需要用于 Oracle 后端）时，在 C 中执行“已经是 Unicode”的检查，不再引入显著的开销。

    这个改变对 cx_Oracle 后端有两个影响。一个是在 Py2K 中，如果没有明确请求 Unicode 类型或 convert_unicode=True，那么字符串值现在将返回为`str`，而不是`unicode` - 这种行为类似于 MySQL 这样的后端。此外，在请求带有 cx_Oracle 后端的 Unicode 值时，如果没有使用 C 扩展，现在每列都会有一个 isinstance() 检查的额外开销。这种权衡是因为它可以被解决，并且不再对大多数非 Unicode 字符串的 Oracle 结果列造成性能负担。

    参考：[#2911](https://www.sqlalchemy.org/trac/ticket/2911)

### 杂项

+   **[bug] [pool]**

    `PoolEvents.reset()` 事件的参数名称已更改为 `dbapi_connection` 和 `connection_record`，以保持与所有其他池事件的一致性。预计任何现有监听器对于这个相对新的且很少使用的事件都在任何情况下使用位置样式接收参数。

+   **[bug] [cextensions] [py3k]**

    修复了一个问题，即 Py3K 中的 C 扩展正在使用错误的 API 来指定顶层模块函数，这在 Python 3.4b2 中会出错。Py3.4b2 将 PyMODINIT_FUNC 更改为返回“void”而不是`PyObject *`，因此我们现在确保使用“PyMODINIT_FUNC”而不是直接使用`PyObject *`。感谢 cgohlke 提交的拉取请求。

## 0.9.1

发布日期：2014 年 1 月 5 日

### orm

+   **[orm] [bug] [events]**

    修复了回归问题，当使用`functools.partial()`与事件系统时，由于在其中使用 inspect.getargspec()来检测某些事件的传统调用签名，因此会导致递归溢出，而显然没有办法在 partial 对象上执行此操作。相反，我们跳过传统检查，并假设现代样式；该检查现在仅在 SessionEvents.after_bulk_update 和 SessionEvents.after_bulk_delete 事件中发生。如果将这两个事件分配给“partial”事件侦听器，则这两个事件将需要新的签名样式。

    参考：[#2905](https://www.sqlalchemy.org/trac/ticket/2905)

+   **[orm] [错误]**

    修复了一个 bug，即如果`.info`参数仅传递给`sessionmaker`创建调用而不传递给对象本身，则使用新的`Session.info`属性将失败。感谢 Robin Schoonover。

+   **[orm] [错误]**

    修复了回归问题，即在设置基于名称的 backref 时，我们不会根据正确的字符串类检查给定的名称，因此会导致错误“值太多而无法解压缩”。这与 Py3k 转换有关。

    参考：[#2901](https://www.sqlalchemy.org/trac/ticket/2901)

+   **[orm] [错误]**

    修复了回归问题，当我们显然仍然在说查询（B）。join（B.cs）时会创建一个隐式别名，其中“C”是一个联接继承类；然而，这个隐式别名只考虑了直接左侧，而没有考虑到同一基类的不同联接继承子类的一系列联接。只要我们在这种情况下仍然隐式别名，行为就会稍微减弱，以便在更广泛的情况下为右侧别名。

    参考：[#2903](https://www.sqlalchemy.org/trac/ticket/2903)

### orm 声明

+   **[orm] [声明] [错误]**

    修复了一个极不可能发生的内存问题，即当使用`DeferredReflection`来定义待反射的类时，如果在调用`DeferredReflection.prepare()`方法之前丢弃了其中某些类的子集，则在映射和反映类时，对类的强引用将仍然保持在声明性内部。这个内部的“要映射的类”集合现在使用弱引用来引用这些类本身。

+   **[orm] [声明] [错误]**

    一种准回归，显然在 0.8 中，您可以在声明性上设置一个类级属性，直接引用超类或类本身上的`InstrumentedAttribute`，它更多或更少地像一个同义词；在 0.9 中，这未能设置足够的记录来跟上从[#2789](https://www.sqlalchemy.org/trac/ticket/2789)中更自由化的 backref 逻辑。即使这种用例从未直接考虑过，现在在声明性的“setattr()”级别以及在设置子类时也会检测到，并且镜像/重命名属性现在设置为`synonym()`。

    参考：[#2900](https://www.sqlalchemy.org/trac/ticket/2900)

### orm 扩展

+   **[orm] [extensions] [feature]**

    添加了一个新的，**实验性的**扩展`sqlalchemy.ext.automap`。该扩展扩展了声明性的功能以及`DeferredReflection`类，以生成一个基类，该基类根据表元数据自动生成映射类和*关系*。

    参见

    Automap Extension

    Automap

### sql

+   **[sql] [feature]**

    像`and_()`和`or_()`这样的连接词现在可以接受 Python 生成器作为单个参数，例如：

    ```py
    and_(x == y for x, y in tuples)
    ```

    这里的逻辑寻找一个单一参数`*args`，其中第一个元素是`types.GeneratorType`的实例。

### schema

+   **[schema] [feature]**

    `Table.extend_existing`和`Table.autoload_replace`参数现在在`MetaData.reflect()`方法中可用。

### orm

+   **[orm] [bug] [events]**

    修复了一个回归问题，即在事件系统中使用`functools.partial()`会因为在其上使用 inspect.getargspec()来检测某些事件的传统调用签名而导致递归溢出，显然无法通过部分对象来实现这一点。相反，我们跳过传统检查并假定现代风格；现在检查本身仅在 SessionEvents.after_bulk_update 和 SessionEvents.after_bulk_delete 事件中发生。如果将这两个事件分配给“partial”事件侦听器，则这两个事件将需要新的签名样式。

    参考：[#2905](https://www.sqlalchemy.org/trac/ticket/2905)

+   **[orm] [bug]**

    修复了一个错误，在使用新的`Session.info`属性时，如果`.info`参数只传递给了`sessionmaker`创建调用，而没有传递给对象本身，则会失败。Robin Schoonover 礼貌提供。

+   **[orm] [bug]**

    修复了一个回归，当设置基于名称的反向引用时，我们不会检查给定名称与正确的字符串类是否匹配，因此会导致错误“要解包的值太多”。这与 Py3k 转换有关。

    参考：[#2901](https://www.sqlalchemy.org/trac/ticket/2901)

+   **[orm] [bug]**

    修复了一个回归，当我们显然仍然在说查询(B).join(B.cs)时创建了一个隐式别名，其中“C”是一个已连接的继承类；但是，此隐式别名仅考虑了立即左侧，而不是沿着相同基类的不同已连接继承子类的一长串连接。只要我们在这种情况下仍然隐式地为别名赋值，行为就会稍微减弱，以便在更广泛的情况下为右侧创建别名。

    参考：[#2903](https://www.sqlalchemy.org/trac/ticket/2903)

### orm 声明性

+   **[orm] [declarative] [bug]**

    修复了一个极不可能的内存问题，即当使用`DeferredReflection`来定义待反射的类时，如果在调用`DeferredReflection.prepare()`方法之前丢弃了其中某些类的子集，那么在反射和映射类时将会保留对该类的强引用在声明性内部。这个内部的“要映射的类”集合现在对类本身使用弱引用。

+   **[orm] [declarative] [bug]**

    几乎是一个回归，显然在 0.8 版本中，你可以在声明性上设置一个类级别的属性，直接引用超类上或类本身上的`InstrumentedAttribute`，并且它更或多少地充当同义词；在 0.9 版本中，这无法设置足够的记录以跟上从 [#2789](https://www.sqlalchemy.org/trac/ticket/2789) 中放宽的反向引用逻辑。即使从未直接考虑过这种用例，现在也会在声明性“setattr()”级别以及在设置子类时检测到，同时，镜像/重命名的属性现在被设置为`synonym()`。

    参考：[#2900](https://www.sqlalchemy.org/trac/ticket/2900)

### orm 扩展

+   **[orm] [extensions] [feature]**

    添加了一个新的、**实验性的**扩展 `sqlalchemy.ext.automap`。该扩展扩展了 Declarative 的功能以及 `DeferredReflection` 类，以生成一个基类，该基类根据表元数据自动生成映射类和*关系*。

    另请参阅

    Automap Extension

    Automap

### sql

+   **[sql] [feature]**

    像 `and_()` 和 `or_()` 这样的连接词现在可以接受 Python 生成器作为单个参数，例如：

    ```py
    and_(x == y for x, y in tuples)
    ```

    此处的逻辑查找一个单一参数 `*args`，其中第一个元素是 `types.GeneratorType` 的实例。

### schema

+   **[schema] [feature]**

    `Table.extend_existing` 和 `Table.autoload_replace` 参数现在可以在 `MetaData.reflect()` 方法中使用。

## 版本：0.9.0

发布日期：2013 年 12 月 30 日

### orm

+   **[orm] [feature]**

    `StatementError` 或与 DBAPI 相关的子类现在可以容纳有关异常“原因”的附加信息；当异常发生在自动刷新中时，`Session` 现在会添加一些详细信息。这种方法是为了与 Py2K 代码以及已经捕获 `IntegrityError` 或类似异常的代码保持兼容，而不是将 `FlushError` 与 Python 3 风格的“链接异常”方法相结合。

+   **[orm] [feature] [backrefs]**

    向 `validates()` 函数添加了新参数 `include_backrefs=True`；当设置为 False 时，如果事件是从另一侧的属性操作的反向引用发起的，则不会触发验证事件。

    另请参阅

    include_backrefs=False option for @validates

    参考：[#1535](https://www.sqlalchemy.org/trac/ticket/1535)

+   **[orm] [feature]**

    通过新的`Query.with_for_update()`方法添加了用于指定`SELECT`的`FOR UPDATE`子句的新 API，以补充新的`GenerativeSelect.with_for_update()`方法。感谢 Mario Lassnig 的拉取请求。

    另请参阅

    对 select()，Query()的新 FOR UPDATE 支持

+   **[orm] [bug]**

    对`subqueryload()`策略进行了调整，确保查询在加载过程开始后运行；这样 subqueryload 就优先于其他加载器，这些加载器可能由于其他错误的时机导致命中相同属性。

    这个更改也被**回溯**到：0.8.5

    参考：[#2887](https://www.sqlalchemy.org/trac/ticket/2887)

+   **[orm] [bug]**

    修复了使用从表到基表的联接表继承时的 bug，其中 PK 列也不是同名的情况；持久性系统在 INSERT 时无法将主键值从基表复制到继承表中。

    这个更改也被**回溯**到：0.8.5

    参考：[#2885](https://www.sqlalchemy.org/trac/ticket/2885)

+   **[orm] [bug]**

    当传递的列/属性（名称）不能解析为列或映射属性（例如错误的元组）时，`composite()`将引发一个信息性错误消息；之前引发一个未绑定的本地错误。

    这个更改也被**回溯**到：0.8.5

    参考：[#2889](https://www.sqlalchemy.org/trac/ticket/2889)

+   **[orm] [bug]**

    修复了由[#2818](https://www.sqlalchemy.org/trac/ticket/2818)引入的回归，其中生成的 EXISTS 查询会为具有两个同名列的语句产生“正在替换列”警告，因为内部 SELECT 没有设置 use_labels。

    这个更改也被**回溯**到：0.8.4

    参考：[#2818](https://www.sqlalchemy.org/trac/ticket/2818)

+   **[orm] [bug] [collections] [py3k]**

    在 ORM 集合仪器系统中添加了对 Python 3 方法`list.clear()`的支持；感谢 Eduardo Schettino 的拉取请求。

+   **[orm] [bug]**

    对于`AliasedClass`构造进行了一些细化，涉及到描述符，如混合体、同义词、复合体、用户定义的描述符等。进行的属性适应变得更加健壮，因此，如果描述符返回另一个被检测的属性，而不是一个复合的 SQL 表达式元素，操作仍将继续。此外，“适应”的操作符将保留其类；以前，从`InstrumentedAttribute`到`QueryableAttribute`（一个超类）的类变化会与 Python 的操作符系统交互，使得像`aliased(MyClass.x) > MyClass.x`这样的表达式会反转为`myclass.x < myclass_1.x`。适应的属性还将引用新的`AliasedClass`作为其父类，这在以前并不总是如此。

    参考：[#2872](https://www.sqlalchemy.org/trac/ticket/2872)

+   **[orm] [bug]**

    `relationship()`上的`viewonly`标志现在将阻止为目标属性代表写入属性历史记录。这将导致对象在被改变时不会被写入到 Session.dirty 列表中。以前，对象会出现在 Session.dirty 中，但在 flush 期间不会代表修改的属性发生任何变化。该属性仍会发出事件，如反向引用事件和用户定义事件，并且仍会接收来自反向引用的变化。

    另请参阅

    在 relationship() 上使用 viewonly=True 阻止历史记录生效

    参考：[#2833](https://www.sqlalchemy.org/trac/ticket/2833)

+   **[orm] [bug]**

    增加了对`scoped_session`的新`Session.info`属性的支持。

+   **[orm] [bug]**

    修复了使用新的`Bundle`对象会导致`Query.column_descriptions`属性失败的 bug。

+   **[orm] [bug] [sql] [sqlite]**

    修复了由于[#2369](https://www.sqlalchemy.org/trac/ticket/2369)和[#2587](https://www.sqlalchemy.org/trac/ticket/2587)的连接重写功能引入的回归，其中一个嵌套连接的一侧已经是别名选择时，无法正确翻译外部的 ON 子句的 bug；在 ORM 中，当使用 SELECT 语句作为“secondary”表时，可以看到这种情况。

    参考：[#2858](https://www.sqlalchemy.org/trac/ticket/2858)

### orm 声明式

+   **[orm] [declarative] [bug]**

    声明式进行额外检查，以检测是否将相同的`Column`映射到不同的属性下（通常应该是`synonym()`），或者是否给两个或更多的`Column`对象赋予相同的名称，如果检测到这种情况，则会发出警告。

    参考：[#2828](https://www.sqlalchemy.org/trac/ticket/2828)

+   **[orm] [声明式] [错误]**

    `DeferredReflection` 类已经增强，为`relationship()`引用的“次要”表提供自动反射支持。当指定“次要”表时，无论是作为字符串表名还是作为仅具有名称和`MetaData`对象的`Table`对象，当调用`DeferredReflection.prepare()`时，也将包括在反射过程中。

    参考：[#2865](https://www.sqlalchemy.org/trac/ticket/2865)

+   **[orm] [声明式] [错误]**

    修复了一个 bug，在 Py2K 中，unicode 文字不能作为声明式中的类或其他参数的字符串名称被`relationship()`接受的 bug。

### 示例

+   **[示例] [错误]**

    修复了一个 bug，该 bug 阻止了 history_meta 配方与多级联继承方案一起工作。

### 引擎

+   **[引擎] [特性]**

    `engine_from_config()` 函数已经改进，以便我们能够从字符串配置字典中解析特定方言的参数。方言类现在可以提供自己的参数类型列表和字符串转换例程。然而，该功能目前尚未被内置方言使用。

    参考：[#2875](https://www.sqlalchemy.org/trac/ticket/2875)

+   **[引擎] [错误]**

    当`connect()`调用引发的错误不是`dbapi.Error`的子类（如`TypeError`、`NotImplementedError`等）时，将原样传播该异常。以前，针对`connect()`例程的特定错误处理既会通过方言的`Dialect.is_disconnect()`例程不适当地运行异常，也会将其包装在一个`sqlalchemy.exc.DBAPIError`中。现在，它将以与执行过程中相同的方式原样传播。

    此更改也已**回溯**至：0.8.4

    参考：[#2881](https://www.sqlalchemy.org/trac/ticket/2881)

+   **[engine] [bug] [pool]**

    `QueuePool`已经进行了增强，当现有连接尝试阻塞时，不会阻止新的连接尝试。以前，新连接的生成是在监视溢出的块内串行进行的；现在，溢出计数器在连接过程本身之外的自己的临界区内被修改。

    此更改也已**回溯**至：0.8.4

    参考：[#2880](https://www.sqlalchemy.org/trac/ticket/2880)

+   **[engine] [bug] [pool]**

    对等待池化连接可用的逻辑进行了微调，这样对于未指定超时的连接池，每半秒钟都会中断等待，以检查所谓的“中止”标志，该标志允许等待者在整个连接池被卸载的情况下中断；通常，等待者应该因为 notify_all()而中断，但在极少数情况下可能会错过这个 notify_all()。这是从 0.8.0 首次引入的逻辑的扩展，该问题只在压力测试中偶尔观察到。

    此更改也已**回溯**至：0.8.4

    参考：[#2522](https://www.sqlalchemy.org/trac/ticket/2522)

+   **[engine] [bug]**

    修复了在`Connection.execute()`中引发了一个预 DBAPI `StatementError`时 SQL 语句会不正确进行 ASCII 编码的错误，从而导致非 ASCII 语句的编码错误。现在，字符串化仍然在 Python Unicode 内部，因此避免了编码错误。

    此更改也已**回溯**至：0.8.4

    参考：[#2871](https://www.sqlalchemy.org/trac/ticket/2871)

+   **[engine] [bug]**

    `create_engine()`例程和相关的`make_url()`函数不再将`+`号视为密码字段中的空格。该区域的解析已经调整，以更接近 RFC 1738 处理这些标记的方式，即`username`和`password`都只期望`:`、`@`和`/`被编码。

    参见

    create_engine()的“密码”部分不再将+号视为编码空格

    参考：[#2873](https://www.sqlalchemy.org/trac/ticket/2873)

+   **[engine] [bug]**

    `RowProxy`对象现在在 Python 中可排序，就像普通元组一样；这是通过在`__eq__()`方法中确保在两侧都进行 tuple()转换以及添加`__lt__()`方法来实现的。

    参见

    RowProxy 现在具有元组排序行为

    参考：[#2848](https://www.sqlalchemy.org/trac/ticket/2848)

### sql

+   **[sql] [feature]**

    `text()`构造的新改进，包括更灵活的设置绑定参数和返回类型的方式；特别是，现在可以将`text()`转换为完整的 FROM 对象，在其他语句中作为别名或 CTE 嵌入，使用新方法`TextClause.columns()`。当构造在“文字绑定”上下文中编译时，`text()`构造还可以呈现“内联”绑定参数。

    参见

    text()的新功能

    参考：[#2877](https://www.sqlalchemy.org/trac/ticket/2877), [#2882](https://www.sqlalchemy.org/trac/ticket/2882)

+   **[sql] [feature]**

    新增了一种用于指定`SELECT`的`FOR UPDATE`子句的 API，使用新的`GenerativeSelect.with_for_update()`方法。该方法支持一种更直接的设置方言特定选项的系统，与`select()`的`for_update`关键字参数相比，还支持 SQL 标准的`FOR UPDATE OF`子句。ORM 还包括一个新的相应方法`Query.with_for_update()`。感谢 Mario Lassnig 的拉取请求。

    参见

    select()、Query() 新的 FOR UPDATE 支持

+   **[sql] [feature]**

    当将返回的浮点值强制转换为 Python `Decimal` 时，现在可以配置使用的精度。标志 `decimal_return_scale` 现在由所有 `Numeric` 和 `Float` 类型支持，这将确保从本机浮点值中取出这么多位数字，当它被转换为字符串时。如果不存在，则类型将使用`.scale` 的值，如果类型支持此设置且它不为 None。否则将使用原始默认长度 10。

    另请参见

    本机浮点类型的浮点字符串转换精度可配置

    参考：[#2867](https://www.sqlalchemy.org/trac/ticket/2867)

+   **[sql] [bug]**

    修复了一个问题，即具有 Sequence 的主键列，但该列不是“自动增量”列，可能是因为具有外键约束或设置了 `autoincrement=False`，在没有支持序列的后端上，当插入缺少主键值的 INSERT 时，会尝试触发 Sequence。这将发生在像 SQLite、MySQL 这样的非序列后端上。

    此更改也已**回溯**至：0.8.5

    参考：[#2896](https://www.sqlalchemy.org/trac/ticket/2896)

+   **[sql] [bug]**

    修复了 `Insert.from_select()` 方法的 bug，其中给定名称的顺序在生成 INSERT 语句时不会被考虑，因此与给定 SELECT 语句中的列名不匹配。还注意到 `Insert.from_select()` 暗示着不能使用 Python 端的插入默认值，因为该语句没有 VALUES 子句。

    此更改也已**回溯**至：0.8.5

    参考：[#2895](https://www.sqlalchemy.org/trac/ticket/2895)

+   **[sql] [bug]**

    当给定一个普通文本值时，`cast()`函数现在会根据给定的类型将给定的文本值应用到绑定参数侧，方式与`type_coerce()`函数相同。然而，与`type_coerce()`不同的是，只有当传递给`cast()`的不是 clauseelement 值时，才会生效；现有的类型构造将保留其类型。

+   **[sql] [错误]**

    `ForeignKey`类更积极地检查给定的列参数。如果不是字符串，则检查对象至少是`ColumnClause`或解析为其中一个的对象，并且`.table`属性（如果存在）指向`TableClause`或其子类，而不是类似`Alias`的东西。否则，将引发`ArgumentError`。

    参考：[#2883](https://www.sqlalchemy.org/trac/ticket/2883)

+   **[sql] [错误]**

    `ColumnOperators.collate()`运算符的优先规则已经修改，使得 COLLATE 运算符现在比比较运算符的优先级低。这样做的效果是，应用于比较的 COLLATE 不会在比较周围添加括号，这样的括号在诸如 MSSQL 等后端不会被解析。这个改变对那些通过将`Operators.collate()`应用于比较表达式的单个元素而绕过问题的设置是不兼容的，而不是整个比较表达式。

    另请参阅

    COLLATE 的优先规则已更改

    参考：[#2879](https://www.sqlalchemy.org/trac/ticket/2879)

+   **[sql] [增强]**

    当在编译的语句中存在一个未提供值的`BindParameter`时引发的异常现在在错误消息中包含了绑定参数的键名。

    此更改也**回溯**到：0.8.5

### 模式

+   **[模式] [错误]**

    修复了由 [#2812](https://www.sqlalchemy.org/trac/ticket/2812) 引起的回归，其中表和列名的 repr() 如果包含非 ASCII 字符则会失败。

    参考：[#2868](https://www.sqlalchemy.org/trac/ticket/2868)

### postgresql

+   **[postgresql] [feature]**

    添加了对 PostgreSQL JSON 的支持，使用新的 `JSON` 类型。非常感谢 Nathan Rice 实施和测试。

    参考：[#2581](https://www.sqlalchemy.org/trac/ticket/2581)

+   **[postgresql] [feature]**

    通过 `TSVECTOR` 类型添加了对 PostgreSQL TSVECTOR 的支持。拉取请求由 Noufal Ibrahim 提供。

+   **[postgresql] [bug]**

    修复了一个 bug，当使用 pypostgresql 适配器时，索引反射会错误地解释 indkey 值，该适配器将这些值作为列表返回，而不是 psycopg2 返回的字符串类型。

    此更改也被**回溯**到：0.8.4

    参考：[#2855](https://www.sqlalchemy.org/trac/ticket/2855)

+   **[postgresql] [bug]**

    现在使用 psycopg2 的 UNICODEARRAY 扩展来处理带有 psycopg2 + 普通“本地 unicode”模式的 unicode 数组，与 UNICODE 扩展的使用方式相同。

+   **[postgresql] [bug]**

    修复了 ENUM 中的值未对单引号进行转义的 bug。请注意，对于手动转义单引号的现有解决方法来说，这是不兼容��。

    另请参阅

    PostgreSQL CREATE TYPE <x> AS ENUM 现在对值应用引号

    参考：[#2878](https://www.sqlalchemy.org/trac/ticket/2878)

### mysql

+   **[mysql] [bug]**

    改进了 SQL 类型在 `__repr__()` 中生成的系统，特别是关于 MySQL 整数/数字/字符类型，这些类型具有各种关键字参数。`__repr__()` 对于 Alembic autogenerate 很重要，用于在迁移脚本中呈现 Python 代码时使用。

    参考：[#2893](https://www.sqlalchemy.org/trac/ticket/2893)

### mssql

+   **[mssql] [bug] [firebird]**

    使用 `Float` 类型时，与“asdecimal”标志一起使用将适用于 Firebird 以及 mssql+pyodbc 方言；以前未发生十进制转换。

    此更改也被**回溯**到：0.8.5

+   **[mssql] [bug] [pymssql]**

    将“Net-Lib error during Connection reset by peer”消息添加到 pymssql 方言中检查“disconnect”消息的列表中。感谢 John Anderson。

    此更改也被**回溯**到：0.8.5

+   **[mssql] [bug]**

    修复了 0.8.0 中引入的 bug，当 MSSQL 中的索引位于替代模式中时，`DROP INDEX` 语句会渲染错误；模式名/表名会被颠倒。格式也已经修订以匹配当前的 MSSQL 文档。感谢 Derek Harland。

    此更改也被**回溯**到：0.8.4

### oracle

+   **[oracle] [bug]**

    将 ORA-02396“最大空闲时间”错误代码添加到与 cx_oracle 一起的“is disconnect”代码列表中。

    此更改也已**回溯**至：0.8.4

    参考：[#2864](https://www.sqlalchemy.org/trac/ticket/2864)

+   **[oracle] [bug]**

    修复了 Oracle 中未指定长度的`VARCHAR`类型（例如用于`CAST`或类似操作）会错误地呈现为`None CHAR`或类似情况的错误。

    此更改也已**回溯**至：0.8.4

    参考：[#2870](https://www.sqlalchemy.org/trac/ticket/2870)

### 杂项

+   **[bug] [firebird]**

    Firebird 方言将引用以下划线开头的标识符。感谢 Treeve Jelbert。

    此更改也已**回溯**至：0.8.5

    参考：[#2897](https://www.sqlalchemy.org/trac/ticket/2897)

+   **[bug] [firebird]**

    修复了 Firebird 索引反射中列未正确排序的错误；现在它们按照 RDB$FIELD_POSITION 的顺序排序。

    此更改也已**回溯**至：0.8.5

+   **[bug] [declarative]**

    当将字符串参数发送到`relationship()`时，如果未解析为类或映射器，则错误消息已更正为与接收非字符串参数时相同的方式，该方式指示配置错误的关系名称。

    此更改也已**回溯**至：0.8.5

    参考：[#2888](https://www.sqlalchemy.org/trac/ticket/2888)

+   **[bug] [ext]**

    修复了一个错误，该错误导致`serializer`扩展无法正确处理包含非 ASCII 字符的表或列名称。

    此更改也已**回溯**至：0.8.4

    参考：[#2869](https://www.sqlalchemy.org/trac/ticket/2869)

+   **[bug] [firebird]**

    更改了 Firebird 用于列出表和视图名称的查询，从`rdb$relations`视图查询，而不是从`rdb$relation_fields`和`rdb$view_relations`视图查询。许多 FAQ 和博客中提到了新旧查询的变体，但新查询直接来自“Firebird FAQ”，这似乎是最官方的信息来源。

    参考：[#2898](https://www.sqlalchemy.org/trac/ticket/2898)

+   **[removed]**

    “informix”和“informixdb”方言已被移除；该代码现在作为一个独立的存储库在 Bitbucket 上提供。自从添加 informixdb 方言以来，IBM-DB 项目提供了生产级 Informix 支持。

### orm

+   **[orm] [feature]**

    `StatementError`或 DBAPI 相关的子类现在可以容纳有关异常“原因”的其他信息；当异常发生在自动刷新中时，`Session`现在会在异常发生时添加一些详细信息。与将`FlushError`与 Python 3 风格的“链接异常”方法相结合以保持与 Py2K 代码以及已经捕获`IntegrityError`或类似异常的代码的兼容性相比，采取了这种方法。

+   **[orm] [feature] [backrefs]**

    向`validates()`函数添加了新参数`include_backrefs=True`；当设置为 False 时，如果事件是从另一侧的属性操作的反向引用发起的，则不会触发验证事件。

    另请参阅

    include_backrefs=False option for @validates

    参考：[#1535](https://www.sqlalchemy.org/trac/ticket/1535)

+   **[orm] [feature]**

    添加了一个新的 API 来指定`SELECT`的`FOR UPDATE`子句，使用新的`Query.with_for_update()`方法，以补充新的`GenerativeSelect.with_for_update()`方法。感谢 Mario Lassnig 的拉取请求。

    另请参阅

    New FOR UPDATE support on select(), Query()

+   **[orm] [bug]**

    对`subqueryload()`策略进行了调整，确保查询在加载过程开始后运行；这样，subqueryload 优先于其他加载器运行，这些加载器可能由于其他错误的时机导致了其他急切/无加载情况。

    此更改也**回溯**到：0.8.5

    参考：[#2887](https://www.sqlalchemy.org/trac/ticket/2887)

+   **[orm] [bug]**

    修复了使用从表到基表的连接表继承时的错误，其中主键列也不同名的 bug；持久性系统在 INSERT 时无法将主键值从基表复制到继承表。

    此更改也**回溯**到：0.8.5

    参考：[#2885](https://www.sqlalchemy.org/trac/ticket/2885)

+   **[orm] [bug]**

    当传递的列/属性（名称）不能解析为列或映射属性（例如错误的元组）时，`composite()`将引发一个信息性错误消息；以前引发一个未绑定的本地错误。

    此更改也已**回溯**至：0.8.5

    参考：[#2889](https://www.sqlalchemy.org/trac/ticket/2889)

+   **[orm] [bug]**

    修复了由[#2818](https://www.sqlalchemy.org/trac/ticket/2818)引入的回归，其中生成的 EXISTS 查询会对具有两个同名列的语句产生“正在替换列”的警告，因为内部 SELECT 不会设置 use_labels。

    此更改也已**回溯**至：0.8.4

    参考：[#2818](https://www.sqlalchemy.org/trac/ticket/2818)

+   **[orm] [bug] [collections] [py3k]**

    在 ORM 集合仪器系统中添加了对 Python 3 方法`list.clear()`的支持；拉取请求由 Eduardo Schettino 提供。

+   **[orm] [bug]**

    对于`AliasedClass`构造进行了一些优化，涉及到描述符，如混合体、同义词、复合体、用户定义的描述符等。进行的属性适应性更加健壮，因此如果描述符返回另一个受检测的属性，而不是一个复合 SQL 表达式元素，则操作仍将继续。此外，“适应”运算符将保留其类；以前，从`InstrumentedAttribute`到`QueryableAttribute`（超类）的类变化会与 Python 的运算符系统交互，使得像`aliased(MyClass.x) > MyClass.x`这样的表达式会反转为`myclass.x < myclass_1.x`。适应的属性还将引用新的`AliasedClass`作为其父类，这在以前并不总是如此。

    参考：[#2872](https://www.sqlalchemy.org/trac/ticket/2872)

+   **[orm] [bug]**

    对于`relationship()`上的`viewonly`标志现在会阻止代表目标属性写入属性历史。这使得如果对象被改变，它不会被写入到 Session.dirty 列表中。以前，对象会出现在 Session.dirty 中，但在刷新期间不会代表修改的属性进行任何更改。该属性仍然会发出事件，例如反向引用事件和用户定义的事件，并且仍将从反向引用接收到变化。

    另请参见

    在 relationship()上设置 viewonly=True 可以阻止历史记录生效

    参考：[#2833](https://www.sqlalchemy.org/trac/ticket/2833)

+   **[orm] [bug]**

    为新的`Session.info`属性添加了对`scoped_session`的支持。

+   **[orm] [bug]**

    修复了使用新的 `Bundle` 对象会导致 `Query.column_descriptions` 属性失败的 bug。

+   **[ORM] [错误] [SQL] [SQLite]**

    修复了由 [#2369](https://www.sqlalchemy.org/trac/ticket/2369) 和 [#2587](https://www.sqlalchemy.org/trac/ticket/2587) 的连接重写特性引入的回归，其中一个嵌套连接的一侧已经是别名选择时，外部的 ON 子句无法正确转换的问题；在 ORM 中，当使用 SELECT 语句作为“次要”表时，可能会出现这种情况。

    参考：[#2858](https://www.sqlalchemy.org/trac/ticket/2858)

### ORM 声明式

+   **[ORM] [声明式] [错误]**

    Declarative 做了额外的检查，以检测同一个 `Column` 是否在不同属性下被映射多次（通常应该是 `synonym()`），或者是否有两个或更多的 `Column` 对象具有相同的名称，如果检测到这种情况，则会发出警告。

    参考：[#2828](https://www.sqlalchemy.org/trac/ticket/2828)

+   **[ORM] [声明式] [错误]**

    `DeferredReflection` 类已经增强，以提供对由 `relationship()` 引用的“次要”表的自动反射支持。当指定“次要”表时，无论是作为字符串表名，还是作为仅具有名称和 `MetaData` 对象的 `Table` 对象，当调用 `DeferredReflection.prepare()` 时，也将包括在反射过程中。

    参考：[#2865](https://www.sqlalchemy.org/trac/ticket/2865)

+   **[ORM] [声明式] [错误]**

    修复了一个 bug，在 Py2K 中，unicode 文本不能被接受作为声明式中 `relationship()` 内的类或其他参数的字符串名称。

### 示例

+   **[示例] [错误]**

    修复了一个 bug，该 bug 阻止了 history_meta 配方与超过一级深度的连接继承方案一起工作。

### 引擎

+   **[引擎] [特性]**

    `engine_from_config()`函数已经改进，以便我们能够从字符串配置字典中解析特定于方言的参数。方言类现在可以提供自己的参数类型列表和字符串转换例程。然而，该功能尚未被内置方言使用。

    参考：[#2875](https://www.sqlalchemy.org/trac/ticket/2875)

+   **[引擎] [错误]**

    引发`connect()`上错误的 DBAPI 如果不是 dbapi.Error 的子类（如`TypeError`、`NotImplementedError`等），则会传播未改变的异常。以前，`connect()`例程的特定错误处理既不恰当地通过方言的`Dialect.is_disconnect()`例程运行异常，也将其包装在`sqlalchemy.exc.DBAPIError`中。现在，它与执行过程中发生的方式相同，未经改变地传播。

    此更改还**被回溯到**: 0.8.4

    参考：[#2881](https://www.sqlalchemy.org/trac/ticket/2881)

+   **[引擎] [错误] [池]**

    `QueuePool`已经增强，以便在现有连接尝试阻塞时不会阻塞新的连接尝试。以前，新连接的生成在监视溢出的块内序列化；现在，在连接过程本身之外，溢出计数器在其自己的关键部分中进行了更改。

    此更改还**被回溯到**: 0.8.4

    参考：[#2880](https://www.sqlalchemy.org/trac/ticket/2880)

+   **[引擎] [错误] [池]**

    对等待池连接可用性的逻辑进行了轻微调整，以便于对于未指定超时的连接池，每隔半秒钟就会中断等待以检查所谓的“中止”标志，这允许等待者在整个连接池被抛弃的情况下中断；通常情况下，等待者应该因为 notify_all()而中断，但在极少数情况下可能会漏掉这个 notify_all()。这是逻辑在 0.8.0 中首次引入的扩展，该问题只在压力测试中偶尔观察到。

    此更改还**被回溯到**: 0.8.4

    参考：[#2522](https://www.sqlalchemy.org/trac/ticket/2522)

+   **[引擎] [错误]**

    修复了一个 bug，当在`Connection.execute()`中引发一个预先 DBAPI `StatementError`时，SQL 语句会被错误地 ASCII 编码，导致非 ASCII 语句出现编码错误。现在字符串化保持在 Python unicode 中，从而避免编码错误。

    此更改也被**回溯**到：0.8.4

    参考：[#2871](https://www.sqlalchemy.org/trac/ticket/2871)

+   **[engine] [bug]**

    `create_engine()`例程和相关的`make_url()`函数不再将`+`号视为密码字段中的空格。该区域的解析已经调整，以更接近 RFC 1738 处理这些标记的方式，即`username`和`password`都只期望`:`, `@`, 和`/`被编码。

    另请参阅

    create_engine()中的“密码”部分不再将+号视为编码空格

    参考：[#2873](https://www.sqlalchemy.org/trac/ticket/2873)

+   **[engine] [bug]**

    `RowProxy`对象现在在 Python 中可以像常规元组一样进行排序；这是通过在`__eq__()`方法中确保双方都进行了 tuple()转换以及添加了`__lt__()`方法来实现的。

    另请参阅

    RowProxy 现在具有元组排序行为

    参考：[#2848](https://www.sqlalchemy.org/trac/ticket/2848)

### sql

+   **[sql] [feature]**

    对`text()`构造进行了新的改进，包括更灵活地设置绑定参数和返回类型的方式；特别是，现在可以将`text()`转换为完整的 FROM 对象，在其他语句中作为别名或 CTE 嵌入，使用新方法`TextClause.columns()`。当构造在“literal bound”上下文中编译时，`text()`构造还可以呈现“内联”绑定参数。

    另请参阅

    新的 text()功能

    参考：[#2877](https://www.sqlalchemy.org/trac/ticket/2877), [#2882](https://www.sqlalchemy.org/trac/ticket/2882)

+   **[sql] [feature]**

    添加了用于指定`SELECT`的`FOR UPDATE`子句的新 API，使用新的`GenerativeSelect.with_for_update()`方法。该方法支持一种更直接的设置方言特定选项的系统，与`select()`的`for_update`关键字参数相比，还包括对 SQL 标准`FOR UPDATE OF`子句的支持。ORM 还包括一个新的相应方法`Query.with_for_update()`。感谢 Mario Lassnig 的拉取请求。

    另请参阅

    在 select()，Query()上支持新的 FOR UPDATE

+   **[sql] [feature]**

    当将返回的浮点值强制转换为 Python `Decimal`时使用的精度现在是可配置的。现在，所有`Numeric`和`Float`类型都支持标志`decimal_return_scale`，这将确保从本机浮点值转换为字符串时取出这么多位数。如果不存在，则类型将使用`.scale`的值，如果类型支持此设置且不为`None`。否则将使用原始默认长度为 10。

    另请参阅

    本地浮点类型的可配置浮点字符串转换精度

    参考：[#2867](https://www.sqlalchemy.org/trac/ticket/2867)

+   **[sql] [bug]**

    修复了主键列具有 Sequence 但列不是“自动增量”列的问题，要么因为它有外键约束，要么设置了`autoincrement=False`，当在没有主键值的情况下提供 INSERT 时，会尝试在不支持序列的后端上触发 Sequence。这将发生在像 SQLite，MySQL 这样的非序列后端上。

    此更改也**回溯**到：0.8.5

    参考：[#2896](https://www.sqlalchemy.org/trac/ticket/2896)

+   **[sql] [bug]**

    修复了`Insert.from_select()`方法的错误，其中给定名称的顺序在生成 INSERT 语句时不会被考虑，因此与给定 SELECT 语句中的列名不匹配。还指出`Insert.from_select()`暗示不能使用 Python 端的插入默认值，因为该语句没有 VALUES 子句。

    此更改也**回溯**到：0.8.5

    参考：[#2895](https://www.sqlalchemy.org/trac/ticket/2895)

+   **[sql] [错误]**

    当`cast()`函数给定普通文字值时，现在将根据给定的类型将给定的文字值应用于绑定参数侧，方式与`type_coerce()`函数相同。但与`type_coerce()`不同，只有在传递非 clauseelement 值给`cast()`时才会生效；现有的类型构造将保留其类型。

+   **[sql] [错误]**

    `ForeignKey`类更积极地检查给定的列参数。如果不是字符串，则检查对象至少是`ColumnClause`，或者解析为其中一个的对象，并且`.table`属性（如果存在）指向`TableClause`或其子类，而不是像`Alias`之类的东西。否则，将引发`ArgumentError`。

    参考：[#2883](https://www.sqlalchemy.org/trac/ticket/2883)

+   **[sql] [错误]**

    `ColumnOperators.collate()`运算符的优先规则已经修改，使得 COLLATE 运算符现在比比较运算符的优先级低。这样做的效果是，应用于比较的 COLLATE 不会在比较周围添加括号，这在后端如 MSSQL 中不会被解析。对于那些通过将`Operators.collate()`应用于比较表达式的单个元素而不是整个比较表达式来解决问题的设置，此更改是不兼容的。

    另请参阅

    COLLATE 的优先规则已更改

    参考：[#2879](https://www.sqlalchemy.org/trac/ticket/2879)

+   **[sql] [增强]**

    当编译语句中存在`BindParameter`但没有值时，现在引发的异常在错误消息中包含绑定参数的键名。

    此更改也**回溯**到：0.8.5

### 模式

+   **[模式] [错误]**

    由 [#2812](https://www.sqlalchemy.org/trac/ticket/2812) 引起的回归已修复，如果名称包含非 ASCII 字符，则表和列名称的 repr() 将失败。

    参考：[#2868](https://www.sqlalchemy.org/trac/ticket/2868)

### postgresql

+   **[postgresql] [功能]**

    添加了对 PostgreSQL JSON 的支持，使用新的 `JSON` 类型。非常感谢 Nathan Rice 实施和测试。

    参考：[#2581](https://www.sqlalchemy.org/trac/ticket/2581)

+   **[postgresql] [功能]**

    通过 `TSVECTOR` 类型添加了对 PostgreSQL TSVECTOR 的支持。感谢 Noufal Ibrahim 提交的拉取请求。

+   **[postgresql] [错误]**

    修复了使用 pypostgresql 适配器时，索引反射会错误解释 indkey 值的 bug，该适配器将这些值返回为列表，而 psycopg2 返回字符串。

    此更改也被**回溯**到：0.8.4

    参考：[#2855](https://www.sqlalchemy.org/trac/ticket/2855)

+   **[postgresql] [错误]**

    现在使用 psycopg2 UNICODEARRAY 扩展来处理带有 psycopg2 + 普通 “本地 unicode” 模式的 unicode 数组，与使用 UNICODE 扩展的方式相同。

+   **[postgresql] [错误]**

    修复了 ENUM 中的值未对单引号进行转义的 bug。请注意，对于手动转义单引号的现有解决方法，这是不兼容的。

    另请参阅

    PostgreSQL CREATE TYPE <x> AS ENUM 现在对值应用引号

    参考：[#2878](https://www.sqlalchemy.org/trac/ticket/2878)

### mysql

+   **[mysql] [错误]**

    对 SQL 类型在 `__repr__()` 中生成的系统进行了改进，特别是关于 MySQL 的整数/数字/字符类型，这些类型具有各种关键字参数。`__repr__()` 对于 Alembic autogenerate 非常重要，用于在迁移脚本中呈现 Python 代码时。

    参考：[#2893](https://www.sqlalchemy.org/trac/ticket/2893)

### mssql

+   **[mssql] [错误] [firebird]**

    与 `Float` 类型一起使用的 “asdecimal” 标志现在也适用于 Firebird 以及 mssql+pyodbc 方言；以前未发生十进制转换。

    此更改也被**回溯**到：0.8.5

+   **[mssql] [错误] [pymssql]**

    将 “Net-Lib error during Connection reset by peer” 消息添加到 pymssql 方言中检查的 “disconnect” 消息列表中。感谢 John Anderson。

    此更改也被**回溯**到：0.8.5

+   **[mssql] [错误]**

    修复了在 0.8.0 中引入的 bug，如果 MSSQL 中的索引位于替代模式中，则 `DROP INDEX` 语句将呈现错误；模式名/表名将被颠倒。格式也已经修订以匹配当前的 MSSQL 文档。感谢 Derek Harland。

    此更改也**回溯**至：0.8.4

### oracle

+   **[oracle] [bug]**

    将 ORA-02396“最大空闲时间”错误代码添加到与 cx_oracle 一起“断开连接”代码列表中。

    此更改也**回溯**至：0.8.4

    参考：[#2864](https://www.sqlalchemy.org/trac/ticket/2864)

+   **[oracle] [bug]**

    修复了 Oracle 中未给出长度的`VARCHAR`类型（例如用于`CAST`或类似操作）会错误地呈现`None CHAR`或类似情况的错误。

    此更改也**回溯**至：0.8.4

    参考：[#2870](https://www.sqlalchemy.org/trac/ticket/2870)

### 杂项

+   **[bug] [firebird]**

    火鸟方言将引用以下划线开头的标识符。Treeve Jelbert 提供。

    此更改也**回溯**至：0.8.5

    参考：[#2897](https://www.sqlalchemy.org/trac/ticket/2897)

+   **[bug] [firebird]**

    修复了 Firebird 索引反射中列未正确排序的错误；它们现在按照 RDB$FIELD_POSITION 的顺序排序。

    此更改也**回溯**至：0.8.5

+   **[bug] [declarative]**

    当发送给`relationship()`的字符串参数无法解析为类或映射器时，错误消息已更正，以与接收非字符串参数时相同的方式工作，指示配置错误的关系名称。

    此更改也**回溯**至：0.8.5

    参考：[#2888](https://www.sqlalchemy.org/trac/ticket/2888)

+   **[bug] [ext]**

    修复了阻止`serializer`扩展与包含非 ASCII 字符的表或列名正常工作的错误。

    此更改也**回溯**至：0.8.4

    参考：[#2869](https://www.sqlalchemy.org/trac/ticket/2869)

+   **[bug] [firebird]**

    更改了 Firebird 用于列出表和视图名称的查询，从`rdb$relations`视图而不是`rdb$relation_fields`和`rdb$view_relations`视图查询。许多 FAQ 和博客中提到了新旧查询的变体，但新查询直接来自“Firebird FAQ”，似乎是最官方的信息来源。

    参考：[#2898](https://www.sqlalchemy.org/trac/ticket/2898)

+   **[removed]**

    “informix”和“informixdb”方言已被移除；代码现在作为一个独立的存储库在 Bitbucket 上提供���自从添加 informixdb 方言以来，IBM-DB 项目提供了生产级 Informix 支持。

## 0.9.0b1

发布日期：2013 年 10 月 26 日

### general

+   **[general] [feature] [py3k]**

    C 扩展已移植到 Python 3，并将在任何支持的 CPython 2 或 3 环境下构建。

    参考：[#2161](https://www.sqlalchemy.org/trac/ticket/2161)

+   **[general] [feature] [py3k]**

    代码库现在针对 Python 2 和 3“原地”，不再需要运行 2to3。兼容性现在针对 Python 2.6 及更高版本。

    参考：[#2671](https://www.sqlalchemy.org/trac/ticket/2671)

+   **[general]**

    对许多 Core 模块以及 ORM 模块的导入结构进行了大规模重构。特别是，`sqlalchemy.sql` 被分解成比以前更多的模块，因此`sqlalchemy.sql.expression` 的庞大大小现在已经减小。该工作集中在大幅减少导入循环上。此外，`sqlalchemy.sql.expression` 和 `sqlalchemy.orm` 中的 API 函数系统已被重新组织，以消除函数与它们产生的对象之间的文档中的冗余。

### orm

+   **[orm] [feature]**

    添加了`relationship()`的新选项`distinct_target_key`。这使得子查询急切加载策略能够对最内层的 SELECT 子查询应用 DISTINCT，在最内层查询生成与此关系对应的重复行的情况下提供帮助（对于在最内层子查询之外的连接生成重复行的问题尚无普遍解决方案）。当标志设置为`True`时，无条件渲染 DISTINCT，当设置为`None`时，如果最内层关系目标列不构成完整的主键，则渲染 DISTINCT。该选项在 0.8 中默认为 False（即在所有情况下默认关闭），在 0.9 中默认为 None（即默认情况下自动开启）。感谢 Alexander Koval 提供帮助。

    另请参阅

    子查询急切加载将对某些查询的最内层 SELECT 应用 DISTINCT

    此更改也已**回溯**至：0.8.3

    参考：[#2836](https://www.sqlalchemy.org/trac/ticket/2836)

+   **[orm] [feature]**

    当从标量关系中获取标量属性时，关联代理现在将返回`None`，如果标量关系本身指向`None`，而不是引发`AttributeError`。

    另请参阅

    关联代理缺失标量返回 None

    参考：[#2810](https://www.sqlalchemy.org/trac/ticket/2810)

+   **[orm] [feature]**

    添加了新方法`AttributeState.load_history()`，功能类似于`AttributeState.history`，但还会触发加载器可调用。

    另请参阅

    attributes.get_history() 将默认从数据库查询，如果值不存在

    参考：[#2787](https://www.sqlalchemy.org/trac/ticket/2787)

+   **[orm] [feature]**

    添加了新的加载选项`load_only()`。这允许指定一系列列名仅加载这些属性，推迟其余属性的加载。

    参考：[#1418](https://www.sqlalchemy.org/trac/ticket/1418)

+   **[orm] [feature]**

    加载器选项系统已完全重构，以构建在一个更全面的基础上，即 `Load` 对象。这个基础允许使用任何常见的加载器选项，如 `joinedload()` ，`defer()` 等，在“链接”风格中用于指定沿路径的选项，比如 `joinedload("foo").subqueryload("bar")`。新系统取代了点分隔路径名称的使用，选项中的多个属性以及 `_all()` 选项的使用。

    另请参阅

    新查询选项 API；load_only() 选项

    参考：[#1418](https://www.sqlalchemy.org/trac/ticket/1418)

+   **[orm] [feature]**

    现在，当在基于列的 `Query` 中使用 `composite()` 构造时，会维持返回对象，而不会扩展为单独的列。这是利用了内部的新 `Bundle` 特性。这种行为是不向后兼容的；要从一个将要展开的复合列中选择，请使用 `MyClass.some_composite.clauses`。

    另请参阅

    当按属性基础查询时，复合属性现在作为其对象形式返回

    参考：[#2824](https://www.sqlalchemy.org/trac/ticket/2824)

+   **[orm] [feature]**

    新增了一个构造 `Bundle` ，允许对列表达式的组进行指定，并作为单个元组返回默认。然而，可以重写 `Bundle` 的行为，以对返回的行进行任何形式的结果处理。现在，在使用基于列的 `Query` 时，`Bundle` 的行为也嵌入到复合属性中。

    另请参阅

    ORM 查询的列绑定

    当按属性基础查询时，复合属性现在作为其对象形式返回

    参考：[#2824](https://www.sqlalchemy.org/trac/ticket/2824)

+   **[orm] [feature]**

    `Mapper`的`version_id_generator`参数现在可以指定依赖于服务器生成的版本标识符，使用触发器或其他数据库提供的版本控制功能，或通过设置`version_id_generator=False`来使用可选的程序值。当使用服务器生成的版本标识符时，ORM 将在可用时使用 RETURNING 立即加载新版本值，否则将发出第二个 SELECT。

    参考：[#2793](https://www.sqlalchemy.org/trac/ticket/2793)

+   **[orm] [feature]**

    `Mapper`的`eager_defaults`标志现在将允许使用内联 RETURNING 子句获取新生成的默认值，而不是使用第二个 SELECT 语句，适用于支持 RETURNING 的后端。

    参考：[#2793](https://www.sqlalchemy.org/trac/ticket/2793)

+   **[orm] [feature]**

    向`Session`添加了一个新属性`Session.info`；这是一个字典，应用程序可以将任意数据存储在`Session`本地。`Session.info`的内容也可以使用`Session`或`sessionmaker`的`info`参数进行初始化。

+   **[orm] [feature]**

    现在已实现事件监听器的移除。该功能通过`remove()`函数提供。

    另请参阅

    事件移除 API

    参考：[#2268](https://www.sqlalchemy.org/trac/ticket/2268)

+   **[orm] [feature]**

    属性事件传递的机制已更改，现在将`AttributeImpl`作为“initiator”令牌传递; 对象现在是一个称为`Event`的特定于事件的对象。此外，属性系统不再根据匹配的“initiator”令牌停止事件; 此逻辑已移至特定于 ORM 反向引用事件处理程序的地方，这些处理程序是将属性事件重新传播到后续附加/设置/删除操作的典型来源。模拟反向引用行为的最终用户代码现在必须确保递归事件传播方案被停止，如果该方案不使用反向引用处理程序。使用这个新系统，当对象附加到集合时，与新的多对一关联，与先前的多对一解除关联，然后从先前的集合中移除时，反向引用处理程序现在可以执行“两跳”操作。在此更改之前，从先前集合中删除的最后一步将不会发生。

    另请参阅

    反向引用处理程序现在可以传播多个级别

    参考：[#2789](https://www.sqlalchemy.org/trac/ticket/2789)

+   **[orm] [功能]**

    关于 ORM 如何构建右侧是自身连接或左外连接的连接的重大变化。现在，ORM 已配置为允许形式为`a JOIN (b JOIN c ON b.id=c.id) ON a.id=b.id`的连接的简单嵌套，而不是强制右侧成为`SELECT`子查询。这应该允许大多数后端数据库获得显着的性能改进，尤其是 MySQL。多年来一直阻碍此更改的一个数据库后端，SQLite，现在通过将`SELECT`子查询的生成从 ORM 移动到 SQL 编译器来解决；因此，在 SQLite 上的右侧连接仍最终以`SELECT`呈现，而所有其他后端不再受此解决方法的影响。

    作为此更改的一部分，`aliased()`、`Join.alias()`和`with_polymorphic()`函数中添加了一个新参数`flat=True`，允许生成一个 JOIN 的“别名”，该别名对加入的每个组件表应用匿名别名，而不是生成一个子查询。

    另请参阅

    许多 JOIN 和 LEFT OUTER JOIN 表达式将不再被包装在(SELECT * FROM ..) AS ANON_1 中

    参考：[#2587](https://www.sqlalchemy.org/trac/ticket/2587)

+   **[orm] [bug]**

    修复了列表插入操作`insert(0, item)`时，列表仪器化无法正确表示`[0:0]`的 bug，特别是在使用关联代理时可能发生。由于 Python 集合中的某些怪癖，该问题在 Python 3 中比在 Python 2 中更有可能发生。

    此更改也已**回溯**至：0.8.3, 0.7.11

    参考：[#2807](https://www.sqlalchemy.org/trac/ticket/2807)

+   **[orm] [bug]**

    修复了在与父`Table`关联之前使用`remote()`或`foreign()`等注释时可能导致父表未在连接中呈现的问题的 bug，这是由注释执行的固有复制操作引起的。

    此更改也已**回溯**至：0.8.3

    参考：[#2813](https://www.sqlalchemy.org/trac/ticket/2813)

+   **[orm] [bug]**

    修复了`Query.exists()`在没有任何 WHERE 条件的情况下无法正常工作的 bug。感谢 Vladimir Magamedov。

    此更改也已**回溯**至：0.8.3

    参考：[#2818](https://www.sqlalchemy.org/trac/ticket/2818)

+   **[orm] [bug]**

    修复了 ORM 用于迭代映射器层次结构的有序序列实现中的潜在问题；在 Jython 解释器下，这个实现没有排序，尽管 cPython 和 PyPy 保持了排序。

    此更改也被**回溯**到：0.8.3

    参考：[#2794](https://www.sqlalchemy.org/trac/ticket/2794)

+   **[orm] [bug]**

    修复了 ORM 级别事件注册中的 bug，在某些“未映射基类”配置中，“原始”或“传播”标志可能被错误配置。

    此更改也被**回溯**到：0.8.3

    参考：[#2786](https://www.sqlalchemy.org/trac/ticket/2786)

+   **[orm] [bug]**

    修复了在加载映射实体时使用 `defer()` 选项时与性能相关的问题。在加载时将每个对象的延迟可调用函数应用到实例的函数开销明显高于仅从行加载数据的开销（请注意，`defer()` 旨在减少数据库/网络开销，而不一定是函数调用次数）；在所有情况下，函数调用开销现在小于从列加载数据的开销。每次加载从 N（结果中的总延迟值）减少到 1（延迟列的总数）的“延迟可调用”对象的数量也有所减少。

    此更改也被**回溯**到：0.8.3

    参考：[#2778](https://www.sqlalchemy.org/trac/ticket/2778)

+   **[orm] [bug]**

    修复了一个 bug，当使用 `make_transient()` 函数将对象从“持久”状态移动到“挂起”状态时，涉及基于集合的反向引用的操作会导致属性历史函数失败。

    此更改也被**回溯**到：0.8.3

    参考：[#2773](https://www.sqlalchemy.org/trac/ticket/2773)

+   **[orm] [bug]**

    当尝试刷新已分配给类别无效值的继承类对象时，会发出警告。

    此更改也被**回溯**到：0.8.2

    参考：[#2750](https://www.sqlalchemy.org/trac/ticket/2750)

+   **[orm] [bug]**

    修复了多个加入继承实体针对同一基类相互加入时，在连接字符串超过两个实体时，不会独立跟踪基表上的列的多态 SQL 生成中的 bug。

    此更改也被**回溯**到：0.8.2

    参考：[#2759](https://www.sqlalchemy.org/trac/ticket/2759)

+   **[orm] [bug]**

    修复了将复合属性传递到 `Query.order_by()` 会产生一种某些数据库不接受的带括号表达式的 bug。

    此更改也被**回溯**到：0.8.2

    参考：[#2754](https://www.sqlalchemy.org/trac/ticket/2754)

+   **[orm] [bug]**

    修复了复合属性与`aliased()`函数之间的交互。以前，在应用别名时，复合属性在比较操作中不会正常工作。

    此更改也**回溯**到：0.8.2

    参考：[#2755](https://www.sqlalchemy.org/trac/ticket/2755)

+   **[orm] [bug] [ext]**

    修复了当调用`clear()`时`MutableDict`未报告更改事件的错误。

    此更改也**回溯**到：0.8.2

    参考：[#2730](https://www.sqlalchemy.org/trac/ticket/2730)

+   **[orm] [bug]**

    当与标量列映射属性一起使用时，`get_history()`现在将遵守传递给它的“被动”标志；由于默认为`PASSIVE_OFF`，如果值不存在，默认情况下该函数将查询数据库。这与 0.8 相比是一种行为变化。

    另请参阅

    attributes.get_history()将默认从数据库查询值不存在的情况

    参考：[#2787](https://www.sqlalchemy.org/trac/ticket/2787)

+   **[orm] [bug] [associationproxy]**

    对于与 None 进行比较的==，！=比较器，用于标量值，还添加了额外的条件，以考虑到关联记录本身不存在，除了现有的对关联记录上的标量端点为 NULL 的测试。以前，比较`Cls.scalar == None`将返回`Cls.associated`存在且`Cls.associated.scalar`为 None 的记录，但不会返回`Cls.associated`不存在的行。更重要的是，相反的操作`Cls.scalar != None` *会*返回`Cls`行，其中`Cls.associated`不存在。

    对于`Cls.scalar != 'somevalue'`的情况也进行了修改，以更像直接的 SQL 比较；只有`Cls.associated`存在且`Associated.scalar`非 NULL 且不等于`'somevalue'`的行才会被返回。以前，这将是一个简单的`NOT EXISTS`。

    还添加了一个特殊用例，您可以在`Cls.scalar`是基于列的值时调用`Cls.scalar.has()`而不带参数 - 这将返回`Cls.associated`是否有任何行存在的信息，而不管`Cls.associated.scalar`是否为 NULL。

    另请参阅

    关联代理 SQL 表达式改进和修复

    参考：[#2751](https://www.sqlalchemy.org/trac/ticket/2751)

+   **[orm] [bug]**

    修复了一个晦涩的错误，当跨越一个具有特定鉴别器值的单表继承子类的多对多关系进行连接/联接加载时，错误的结果会被获取，这是由于返回的“secondary”行。现在，在所有 ORM 多对多关系的 JOIN 中，“secondary”和右侧表现在括号内进行内连接，以便左->右连接可以准确过滤。这个改变是通过最终解决了在[#2587](https://www.sqlalchemy.org/trac/ticket/2587)中概述的右嵌套连接问题而实现的。

    另请参阅

    许多 JOIN 和 LEFT OUTER JOIN 表达式将不再被包装在(SELECT * FROM ..) AS ANON_1 中

    参考：[#2369](https://www.sqlalchemy.org/trac/ticket/2369)

+   **[orm] [bug]**

    `Query.select_from()`方法的“自动别名”行为已被关闭。现在可以通过新方法`Query.select_entity_from()`获得特定行为。这里的自动别名行为从未有很好的文档记录，并且通常不是所需的行为，因为`Query.select_from()`更多地用于控制如何呈现 JOIN。`Query.select_entity_from()`也将在 0.8 中提供，以便依赖自动别名的应用程序可以转向使用此方法。

    另请参阅

    _query.Query.select_from()不再将子句应用于相应的实体

    参考：[#2736](https://www.sqlalchemy.org/trac/ticket/2736)

### orm 声明

+   **[orm] [bug]**

    添加了一个方便的类装饰器`as_declarative()`，它是对`declarative_base()`的包装，允许使用一种巧妙的类装饰方法应用现有的基类。

    此更改也**回溯**到：0.8.3

+   **[orm] [bug]**

    现在可以在与`order_by`、`primaryjoin`或类似的用法中使用字符串参数引用 ORM 描述符，例如混合属性，以及列绑定属性，用于`relationship()`中。

    此更改也**回溯**到：0.8.2

    参考：[#2761](https://www.sqlalchemy.org/trac/ticket/2761)

### 示例

+   **[示例] [功能]**

    改进了`examples/generic_associations`中的示例，包括`discriminator_on_association.py`使用单表继承来处理“鉴别器”。还添加了一个真正的“通用外键”示例，它类似于其他流行框架，使用���个开放的整数指向任何其他表，放弃了传统的参照完整性。虽然我们不推荐这种模式，但信息想要自由。

    这个更改也被**回溯**到：0.8.3

+   **[examples] [bug]**

    在版本示例中添加了“autoincrement=False”到历史表中，因为这个表在任何情况下都不应该有自增。感谢 Patrick Schmid。

    这个更改也被**回溯**到：0.8.3

+   **[examples] [bug]**

    修复了“版本控制”配方中的一个问题，即当存在反向引用时，一个多对一引用可能会为目标产生一个无意义的版本，即使它没有被更改。补丁由 Matt Chisholm 提供。

    这个更改也被**回溯**到：0.8.2

### engine

+   **[engine] [feature]**

    对于`Engine`的`URL`的`repr()`现在将使用星号隐藏密码。感谢 Gunnlaugur Þór Briem。

    这个更改也被**回溯**到：0.8.3

    参考：[#2821](https://www.sqlalchemy.org/trac/ticket/2821)

+   **[engine] [feature]**

    在`ConnectionEvents`中添加了新事件：

    +   `ConnectionEvents.engine_connect()`

    +   `ConnectionEvents.set_connection_execution_options()`

    +   `ConnectionEvents.set_engine_execution_options()`

    参考：[#2770](https://www.sqlalchemy.org/trac/ticket/2770)

+   **[engine] [bug]**

    `make_url()`函数使用的正则表达式现在解析 ipv6 地址，例如用括号括起来。

    这个更改也被**回溯**到：0.8.3, 0.7.11

    参考：[#2851](https://www.sqlalchemy.org/trac/ticket/2851)

+   **[engine] [bug] [oracle]**

    如果重新创建一个`Engine`，则不会第二次调用 Dialect.initialize()，因为出现了断开错误。这修复了 Oracle 8 方言中的一个特定问题，但通常情况下，Dialect.initialize()阶段应该只执行一次。

    这个更改也被**回溯**到：0.8.3

    参考：[#2776](https://www.sqlalchemy.org/trac/ticket/2776)

+   **[engine] [bug] [pool]**

    修复了一个 bug，当现有的池化连接在失效或重新生成事件后未能重新连接时，`QueuePool`会丢失正确的已检出计数。

    这个更改也被**回溯**到：0.8.3

    参考：[#2772](https://www.sqlalchemy.org/trac/ticket/2772)

+   **[引擎] [错误]**

    修复了一个 bug，当`Pool`的各种实现中的`reset_on_return`参数在重新生成池时未被传播时。感谢 Eevee。

    这个更改也被**回溯**到：0.8.2

+   **[引擎] [错误]**

    `Dialect.reflecttable()`的方法签名，所有已知情况下都由`DefaultDialect`提供，已经被调整为期望`include_columns`和`exclude_columns`参数，不带任何 kw 选项，减少了歧义 - 以前缺少了`exclude_columns`。

    参考：[#2748](https://www.sqlalchemy.org/trac/ticket/2748)

### sql

+   **[sql] [特性]**

    添加了对“唯一约束”反射的支持，通过`Inspector.get_unique_constraints()`方法。感谢 Roman Podolyaka 的补丁。

    这个更改也被**回溯**到：0.8.4

    参考：[#1443](https://www.sqlalchemy.org/trac/ticket/1443)

+   **[sql] [特性]**

    `update()`、`insert()`和`delete()`构造现在将 ORM 实体解释为要操作的目标表，例如：

    ```py
    from sqlalchemy import insert, update, delete

    ins = insert(SomeMappedClass).values(x=5)

    del_ = delete(SomeMappedClass).where(SomeMappedClass.id == 5)

    upd = update(SomeMappedClass).where(SomeMappedClass.id == 5).values(name="ed")
    ```

    这个更改也被**回溯**到：0.8.3

+   **[sql] [特性] [mysql] [postgresql]**

    PostgreSQL 和 MySQL 方言现在支持外键选项的反射/检查，包括 ON UPDATE、ON DELETE。PostgreSQL 还反映了 MATCH、DEFERRABLE 和 INITIALLY。感谢 ijl。

    参考：[#2183](https://www.sqlalchemy.org/trac/ticket/2183)

+   **[sql] [特性]**

    一个带有“null”类型（例如没有指定类型）的`bindparam()`构造现在在用于有类型表达式时会被复制，并且新的副本会被分配给比较列的实际类型。以前，这个逻辑会在给定的`bindparam()`上发生。此外，类似的过程现在也会发生在传递给`ValuesBase.values()`用于`Insert`或`Update`构造的`bindparam()`构造中，在构造的编译阶段。

    这两者都是一些微妙的行为变化，可能会影响一些用法。

    参见

    一个没有类型的`bindparam()`构造在有类型时通过复制升级

    参考：[#2850](https://www.sqlalchemy.org/trac/ticket/2850)

+   **[sql] [特性]**

    对特殊符号的表达式处理进行了彻底改革，特别是连接词，例如`None` `null()` `true()` `false()`，包括在连接词中渲染 NULL 的一致性，“短路”`and_()`和`or_()`表达式中包含布尔常量，并且将布尔常量和表达式渲染为与不支持`true`/`false`常量的后端相比的“1”或“0”。

    参见

    改进的布尔常量、NULL 常量、连接词的渲染

    参考：[#2734](https://www.sqlalchemy.org/trac/ticket/2734), [#2804](https://www.sqlalchemy.org/trac/ticket/2804), [#2823](https://www.sqlalchemy.org/trac/ticket/2823)

+   **[sql] [特性]**

    现在，键入系统处理呈现“文字绑定”值的任务，例如通常是绑定参数但由于上下文必须呈现为字符串的值，通常在 DDL 构造中，例如 CHECK 约束和索引中（请注意，“文字绑定”值从[#2742](https://www.sqlalchemy.org/trac/ticket/2742)开始被 DDL 使用）。一个新方法`TypeEngine.literal_processor()`作为基础，添加了`TypeDecorator.process_literal_param()`以允许包装本机文字呈现方法。

    另请参阅

    键入系统现在处理呈现“文字绑定”值的任务。

    参考：[#2838](https://www.sqlalchemy.org/trac/ticket/2838)

+   **[sql] [功能]**

    `Table.tometadata()`方法现在会复制所有结构中所有`SchemaItem`对象的所有`SchemaItem.info`字典，包括列、约束、外键等。由于这些字典是副本，它们独立于原始字典。以前，此操作仅传输`Column`的`.info`字典，并且仅在原地链接，而不是复制。

    参考：[#2716](https://www.sqlalchemy.org/trac/ticket/2716)

+   **[sql] [功能]**

    `Column`的`default`参数现在接受类或对象方法作为参数，除了独立函数；将正确检测是否接受“上下文”参数。

+   **[sql] [功能]**

    添加了新方法到`insert()`构造`Insert.from_select()`。给定列的列表和可选择的内容，呈现`INSERT INTO (table) (columns) SELECT ..`。虽然此功能作为 0.9 的一部分而突出显示，但也已回溯到 0.8.3。

    另请参阅

    从 SELECT 插入

    参考：[#722](https://www.sqlalchemy.org/trac/ticket/722)

+   **[sql] [功能]**

    为`TypeDecorator`提供了一个名为`TypeDecorator.coerce_to_is_types`的新属性，以便更容易控制使用`==`或`!=`与`None`和布尔类型进行比较时如何生成`IS`表达式，或者与绑定参数一起生成普通的相等表达式。

    参考：[#2734](https://www.sqlalchemy.org/trac/ticket/2734), [#2744](https://www.sqlalchemy.org/trac/ticket/2744)

+   **[sql] [feature]**

    如果`label()`构造也在 select 的列子句中引用了该标签，则该`label()`构造现在将仅在`ORDER BY`子句中呈现其名称，而不是重新编写完整表达式。这使得数据库有更好的机会优化在两个不同上下文中评估相同表达式。

    另请参阅

    标签构造现在可以在 ORDER BY 中仅呈现其名称

    参考：[#1068](https://www.sqlalchemy.org/trac/ticket/1068)

+   **[sql] [bug]**

    修复了自 0.7.9 以来的回归，即如果在多个 FROM 子句中引用了 CTE 的名称，则可能无法正确引用 CTE 的名称。

    此更改也被**回溯**到：0.8.3, 0.7.11

    参考：[#2801](https://www.sqlalchemy.org/trac/ticket/2801)

+   **[sql] [bug] [cte]**

    修复了通用表达式系统中的错误，如果 CTE 仅被用作`alias()`构造，则不会使用 WITH 关键字呈现。

    此更改也被**回溯**到：0.8.3, 0.7.11

    参考：[#2783](https://www.sqlalchemy.org/trac/ticket/2783)

+   **[sql] [bug]**

    修复了`CheckConstraint` DDL 中的错误，其中来自`Column`对象的“quote”标志不会传播。

    此更改也被**回溯**到：0.8.3, 0.7.11

    参考：[#2784](https://www.sqlalchemy.org/trac/ticket/2784)

+   **[sql] [bug]**

    修复了`type_coerce()`无法正确解释具有`__clause_element__()`方法的 ORM 元素的错误。

    此更改也被**回溯**到：0.8.3

    参考：[#2849](https://www.sqlalchemy.org/trac/ticket/2849)

+   **[sql] [bug]**

    当生成“非本地”类型的 CHECK 约束时，`Enum`和`Boolean`类型现在会绕过任何自定义（例如 TypeDecorator）类型的使用。这样，自定义类型不会参与 CHECK 中的表达式，因为此表达式针对“impl”值而不是“decorated”值。

    此更改也被**回溯**到：0.8.3

    参考：[#2842](https://www.sqlalchemy.org/trac/ticket/2842)

+   **[sql] [bug]**

    如果从未指定`unique`（默认为`None`）的`Column`生成了`Index`，则`.unique`标志可能会生成为`None`。现在该标志将始终为`True`或`False`。

    此更改也被**回溯**到：0.8.3

    参考：[#2825](https://www.sqlalchemy.org/trac/ticket/2825)

+   **[sql] [bug]**

    修复了默认编译器以及 postgresql、mysql 和 mssql 的 bug，以确保任何字面 SQL 表达式值在 CREATE INDEX 语句中直接呈现为字面值，而不是作为绑定参数。这也改变了其他 DDL（如约束）的呈现方案。

    此更改也被**回溯**到：0.8.3

    参考：[#2742](https://www.sqlalchemy.org/trac/ticket/2742)

+   **[sql] [bug]**

    一个`select()`在其 FROM 子句中引用自身，通常通过就地突变，将引发信息性错误消息，而不是导致递归溢出。

    此更改也被**回溯**到：0.8.3

    参考：[#2815](https://www.sqlalchemy.org/trac/ticket/2815)

+   **[sql] [bug]**

    修复了使用`column_reflect`事件更改传入`Column`的`.key`会阻止正确反映主键约束、索引和外键约束的 bug。

    此更改也被**回溯**到：0.8.3

    参考：[#2811](https://www.sqlalchemy.org/trac/ticket/2811)

+   **[sql] [bug]**

    0.8 中添加的`ColumnOperators.notin_()`运算符现在正确地生成了对空集合使用时“IN”返回的否定。

    此更改也被**回溯**到：0.8.3

+   **[sql] [bug] [postgresql]**

    修复了表达式系统依赖于`select()`构造中的`.c`集合的`str()`形式的一些表达式的 bug，但由于元素依赖于方言特定的编译构造，特别是与 PostgreSQL `ARRAY` 元素一起使用的`__getitem__()`运算符，因此`str()`形式不可用。 修复还添加了一个新的异常类`UnsupportedCompilationError`，在编译器被要求编译它不知道如何处理的内容时引发该异常。

    此更改也**回溯**到：0.8.3

    参考：[#2780](https://www.sqlalchemy.org/trac/ticket/2780)

+   **[sql] [bug]**

    对`Select` 构造的关联行为进行了多次修复，这是在 0.8.0 中首次引入的：

    +   为满足 FROM 条目应该向外关联到包含另一个 SELECT 的 SELECT，然后再包含此 SELECT 的用例，现在当通过`Select.correlate()`建立显式关联时，关联现在可以跨多个级别工作，前提是目标 select 在由 WHERE/ORDER BY/columns 子句包含的链中的某处，而不仅仅是嵌套的 FROM 子句。 这使得`Select.correlate()`的行为再次更加兼容于 0.7，同时仍保持新的“智能”关联。

    +   当未使用显式关联时，通常的“隐式”关联将其行为限制在仅限于直接封闭的 SELECT 中，以最大限度地提高与 0.7 应用程序的兼容性，并且在这种情况下还防止跨嵌套 FROM 的关联，以保持与 0.8.0/0.8.1 的兼容性。

    +   `Select.correlate_except()` 方法未在所有情况下阻止给定的 FROM 子句进行关联，并且还会导致 FROM 子句被错误地完全省略（更像是 0.7 会做的），这已经修复。

    +   调用`select.correlate_except(None)`将使所有 FROM 子句进入关联，正如预期的那样。

    此更改也**回溯**到：0.8.2

    参考：[#2668](https://www.sqlalchemy.org/trac/ticket/2668)，[#2746](https://www.sqlalchemy.org/trac/ticket/2746)

+   **[sql] [bug]**

    修复了一个 bug，即将表“A”的多个外键路径的`select()`与表“B”连接到表“B”时，如果直接将表“A”与“B”连接，则不会产生“模糊的连接条件”错误，而是会产生具有多个条件的连接条件。

    此更改也**回溯**到：0.8.2

    参考：[#2738](https://www.sqlalchemy.org/trac/ticket/2738)

+   **[sql] [bug] [reflection]**

    修复了一个 bug，即在跨远程模式和本地模式使用 `MetaData.reflect()` 可能会在两个模式都有相同名称的表的情况下产生错误结果。

    此更改也**回溯**到：0.8.2

    参考：[#2728](https://www.sqlalchemy.org/trac/ticket/2728)

+   **[sql] [bug]**

    从基础 `ColumnOperators` 类中删除了“not implemented” `__iter__()` 调用，虽然这在 0.8.0 版本中引入是为了防止在自定义运算符上实现 `__getitem__()` 方法并在该对象上错误调用 `list()` 时出现无限增长的内存循环，但它导致列元素报告它们实际上是可迭代类型，然后在尝试迭代时抛出错误。在这里没有真正的办法同时实现两边，所以我们坚持使用 Python 最佳实践。在自定义运算符上实现 `__getitem__()` 时要小心！

    此更改也**回溯**到：0.8.2

    参考：[#2726](https://www.sqlalchemy.org/trac/ticket/2726)

+   **[sql] [bug]**

    在调用“attach”事件之前，`Index` 上设置了“name”属性，以便可以使用附加事件根据父表和/或列动态生成索引名称。

    参考：[#2835](https://www.sqlalchemy.org/trac/ticket/2835)

+   **[sql] [bug]**

    `ForeignKey` 对象中的错误 kw 参数“schema”已被移除。这是一个意外提交，没有任何作用；在使用此 kw 参数时，0.8.3 版本会发出警告。

    参考：[#2831](https://www.sqlalchemy.org/trac/ticket/2831)

+   **[sql] [bug]**

    对“引号”标识符处理方式进行了重新设计，不再依赖于传递各种`quote=True`标志，而是将这些标志转换为包含引号信息的丰富字符串对象，并在它们传递给像`Table`、`Column`等常见模式构造时包含这些信息。这解决了各种方法不正确遵守“quote”标志的问题，例如`Engine.has_table()`和相关方法。`quoted_name`对象是一个字符串子类，如果需要，也可以明确使用；该对象将保留传递的引号首选项，并且还将绕过标准化为大写符号的方言执行的“名称规范化”。结果是，“大写”后端现在可以使用强制引号名称，例如小写引号名称和新的保留字。

    另请参阅

    模式标识符现在携带自己的引号信息

    参考：[#2812](https://www.sqlalchemy.org/trac/ticket/2812)

+   **[sql] [bug]**

    `ForeignKey`对象解析为其目标`Column`的分辨率已经重新设计，以尽可能立即进行，基于目标`Column`与此`ForeignKey`关联的相同`MetaData`的时刻，而不是等待构建联接的第一次或类似情况。这与其他改进一起，允许更早地检测到一些外键配置问题。此外，这里还包括对类型传播系统的重新设计，因此现在应该可以可靠地在通过`ForeignKey`引用另一个`Column`的任何`Column`上将类型设置为`None` - 该类型将在另一个列关联时立即从目标列复制，并且现在也适用于复合外键。

    另请参阅

    列可以可靠地从通过 ForeignKey 引用的列获取其类型

    参考：[#1765](https://www.sqlalchemy.org/trac/ticket/1765)

### postgresql

+   **[postgresql] [feature]**

    支持 PostgreSQL 9.2 范围类型已添加。目前尚未提供类型转换，因此暂时直接使用字符串或 psycopg2 2.5 范围扩展类型。补丁由 Chris Withers 提供。

    此更改也**已回溯**至：0.8.2

+   **[postgresql] [功能]**

    在使用 psycopg2 DBAPI 时，添加了对“AUTOCOMMIT”隔离的支持。关键字可通过`isolation_level`执行选项使用。补丁由 Roman Podolyaka 提供。

    此更改也**已回溯**至：0.8.2

    参考：[#2072](https://www.sqlalchemy.org/trac/ticket/2072)

+   **[postgresql] [功能]**

    当在主键自增列上使用 `SmallInteger` 类型时，根据 PostgreSQL 版本检测，添加了对 `SMALLSERIAL` 的呈现支持。

    参考：[#2840](https://www.sqlalchemy.org/trac/ticket/2840)

+   **[postgresql] [错误]**

    删除了从列的服务器默认反射中的 128 字符截断；此代码最初来自 PG 系统视图，用于可读性截断字符串。

    此更改也**已回溯**至：0.8.3

    参考：[#2844](https://www.sqlalchemy.org/trac/ticket/2844)

+   **[postgresql] [错误]**

    在 CREATE INDEX 语句的列列表中，将会对复合 SQL 表达式应用括号。

    此更改也**已回溯**至：0.8.3

    参考：[#2742](https://www.sqlalchemy.org/trac/ticket/2742)

+   **[postgresql] [错误]**

    修复了 PostgreSQL 版本字符串前缀为“PostgreSQL”或“EnterpriseDB”之前的字符串不会解析的错误。由 Scott Schaefer 提供。

    此更改也**已回溯**至：0.8.3

    参考：[#2819](https://www.sqlalchemy.org/trac/ticket/2819)

+   **[postgresql] [错误]**

    在 PostgreSQL 方言上，`extract()`的行为已简化，不再将硬编码的`::timestamp`或类似的转换注入到给定的表达式中，因为这会干扰诸如时区感知日期时间之类的类型，但是对于现代版本的 psycopg2 似乎也不是必需的。

    此更改也**已回溯**至：0.8.2

    参考：[#2740](https://www.sqlalchemy.org/trac/ticket/2740)

+   **[postgresql] [错误]**

    修复了 HSTORE 类型中包含反斜杠引号的键/值在使用“非本机”（即非-psycopg2）方式转换 HSTORE 数据时不会正确转义的错误。补丁由 Ryan Kelly 提供。

    此更改也**已回溯**至：0.8.2

    参考：[#2766](https://www.sqlalchemy.org/trac/ticket/2766)

+   **[postgresql] [错误]**

    修复了多列 PostgreSQL 索引中列顺序反映错误的错误。由 Roman Podolyaka 提供。

    此更改也**已回溯**至：0.8.2

    参考：[#2767](https://www.sqlalchemy.org/trac/ticket/2767)

### mysql

+   **[mysql] [功能]**

    与`Index`一起使用的`mysql_length`参数现在可以作为列名/长度的字典传递，用于复合索引。非常感谢 Roman Podolyaka 的补丁。

    此更改也已**回溯**至：0.8.2

    参考：[#2704](https://www.sqlalchemy.org/trac/ticket/2704)

+   **[mysql] [feature]**

    MySQL `SET` 类型现在具有与`ENUM`相同的自动引号行为。在设置值时不需要引号，但存在的引号将被自动检测并发出警告。这也有助于 Alembic，在那里 SET 类型不带引号。

    参考：[#2817](https://www.sqlalchemy.org/trac/ticket/2817)

+   **[mysql] [bug]**

    更新 MySQL 保留字版本 5.5、5.6，感谢 Hanno Schlichting。

    此更改也已**回溯**至：0.8.3, 0.7.11

    参考：[#2791](https://www.sqlalchemy.org/trac/ticket/2791)

+   **[mysql] [bug]**

    [#2721](https://www.sqlalchemy.org/trac/ticket/2721)中的更改是，MySQL 后端对`ForeignKeyConstraint`的`deferrable`关键字被静默忽略，将在 0.9 版本中恢复；此关键字现在将再次呈现，在 MySQL 上引发错误，因为它不被理解 - 相同的行为也将适用于`initially`关键字。在 0.8 版本中，这些关键字将继续被忽略，但会发出警告。此外，`match`关键字现在在 0.9 上引发`CompileError`，在 0.8 上发出警告；这个关键字不仅被 MySQL 静默忽略，还会破坏 ON UPDATE/ON DELETE 选项。

    要使用在 MySQL 上不呈现或呈现不同的`ForeignKeyConstraint`，请使用自定义编译选项。文档中已添加了此用法示例，请参阅 MySQL / MariaDB Foreign Keys。

    此更改也已**回溯**至：0.8.3

    参考：[#2721](https://www.sqlalchemy.org/trac/ticket/2721), [#2839](https://www.sqlalchemy.org/trac/ticket/2839)

+   **[mysql] [bug]**

    MySQL-connector 方言现在允许在 create_engine 查询字符串中使用选项来覆盖在连接中设置的默认值，包括“buffered”和“raise_on_warnings”。

    此更改也已**回溯**至：0.8.3

    参考：[#2515](https://www.sqlalchemy.org/trac/ticket/2515)

+   **[mysql] [bug]**

    修复了在使用多表 UPDATE 时出现的 bug，其中一个补充表是带有自己绑定参数的 SELECT，绑定参数的位置与使用 MySQL 的特殊语法时语句本身相反。

    此更改也**回溯**到：0.8.2

    参考：[#2768](https://www.sqlalchemy.org/trac/ticket/2768)

+   **[mysql] [bug]**

    在`mysql+gaerdbms`方言中添加了另一个条件来检测所谓的“开发”模式，其中我们应该使用`rdbms_mysqldb` DBAPI。补丁由 Brett Slatkin 提供。

    此更改也**回溯**到：0.8.2

    参考：[#2715](https://www.sqlalchemy.org/trac/ticket/2715)

+   **[mysql] [bug]**

    在`ForeignKey`和`ForeignKeyConstraint`上的`deferrable`关键字参数不会在 MySQL 方言上呈现`DEFERRABLE`关键字。很长一段时间以来，我们一直保留这个设置，因为一个非延迟的外键与一个延迟的外键的行为非常不同，但一些环境只是在 MySQL 上禁用 FKs，所以我们在这里会少些主观意见。

    此更改也**回溯**到：0.8.2

    参考：[#2721](https://www.sqlalchemy.org/trac/ticket/2721)

+   **[mysql] [bug]**

    修复并测试 MySQL 外键选项在反射中的解析；这是对[#2183](https://www.sqlalchemy.org/trac/ticket/2183)中的工作的补充，我们开始支持外键选项的反射，如 ON UPDATE/ON DELETE cascade。

    参考：[#2839](https://www.sqlalchemy.org/trac/ticket/2839)

+   **[mysql] [bug]**

    改进了对 cymysql 驱动程序的支持，支持版本 0.6.5，感谢 Hajime Nakagami。

### sqlite

+   **[sqlite] [bug]**

    新增的 SQLite DATETIME 参数 storage_format 和 regexp 显然没有完全正确实现；虽然参数被接受，但实际上它们没有任何效果；这个问题已经修复。

    此更改也**回溯**到：0.8.3

    参考：[#2781](https://www.sqlalchemy.org/trac/ticket/2781)

+   **[sqlite] [bug]**

    将`sqlalchemy.types.BIGINT`添加到可以由 SQLite 方言反射的类型名称列表中；感谢 Russell Stuart。

    此更改也**回溯**到：0.8.2

    参考：[#2764](https://www.sqlalchemy.org/trac/ticket/2764)

### mssql

+   **[mssql] [bug]**

    在 SQL Server 2000 上查询信息模式时，删除了在 0.8.1 中添加的 CAST 调用，以帮助处理驱动程序问题，显然在 2000 上不兼容。CAST 保留在 SQL Server 2005 及更高版本中。

    此更改也**回溯**到：0.8.2

    参考：[#2747](https://www.sqlalchemy.org/trac/ticket/2747)

+   **[mssql] [bug] [pyodbc]**

    修复了 Python 3 + pyodbc 中的 MSSQL 问题，包括正确传递语句。

    参考：[#2355](https://www.sqlalchemy.org/trac/ticket/2355)

### oracle

+   **[oracle] [特性] [py3k]**

    使用 cx_oracle 进行的 Oracle 单元测试现在完全在 Python 3 下通过。

+   **[oracle] [错误]**

    修复了使用同义词反射 Oracle 表时，如果同义词和表位于不同的远程模式中，则会失败的错误。感谢 Kyle Derr 提供的修补程序。

    此更改还**反向移植**到：0.8.3

    参考：[#2853](https://www.sqlalchemy.org/trac/ticket/2853)

### 杂项

+   **[特性]**

    向 `Column` 添加了一个新标志 `system=True`，它将列标记为数据库自动添加的“系统”列（例如 PostgreSQL 的 `oid` 或 `xmin`）。该列将被省略在 `CREATE TABLE` 语句中，但仍可用于查询。此外，`CreateColumn` 构造可以应用于自定义编译规则，以允许跳过列，方法是生成一个返回 `None` 的规则。

    此更改还**反向移植**到：0.8.3

+   **[特性] [firebird]**

    向 kinterbasdb 和 fdb 方言添加了新标志 `retaining=True`。这控制发送到 DBAPI 连接的 `commit()` 和 `rollback()` 方法的 `retaining` 标志的值。由于历史原因，此标志在 0.8.2 中默认为 `True`，但是在 0.9.0b1 中，此标志默认为 `False`。

    此更改还**反向移植**到：0.8.2

    参考：[#2763](https://www.sqlalchemy.org/trac/ticket/2763)

+   **[特性] [核心]**

    添加了 `UpdateBase.returning()` 的新变体，称为 `ValuesBase.return_defaults()`；这允许将任意列添加到语句的 RETURNING 子句中，而不会干扰编译器通常的“隐式返回”特性，该特性用于有效地获取新生成的主键值。对于支持的后端，所有获取的值的字典都存在于 `ResultProxy.returned_defaults` 中。

    参考：[#2793](https://www.sqlalchemy.org/trac/ticket/2793)

+   **[特性] [池]**

    添加了“回滚后返回”和较少使用的“提交后返回”的池记录。这与池“调试”日志一起启用。

    参考：[#2752](https://www.sqlalchemy.org/trac/ticket/2752)

+   **[特性] [firebird]**

    当未指定方言限定符时，`fdb` 方言现在是默认方言，即 `firebird://`，根据 Firebird 项目发布 `fdb` 作为其官方 Python 驱动程序。

    参考：[#2504](https://www.sqlalchemy.org/trac/ticket/2504)

+   **[错误] [firebird]**

    在反射 Firebird 类型 LONG 和 INT64 时，类型查找已修复，使得 LONG 被视为 INTEGER，INT64 被视为 BIGINT，除非类型具有“精度”，在这种情况下，它将被视为 NUMERIC。补丁由 Russell Stuart 提供。

    此更改还**反向移植**到：0.8.2

    参考：[#2757](https://www.sqlalchemy.org/trac/ticket/2757)

+   **[错误] [扩展]**

    修复了一个错误，即如果使用函数而不是类设置组合类型，则当尝试检查该列是否为 `MutableComposite` 时，可变扩展会失败（它不是）。感谢 asldevi 提供。

    此更改也已**回溯**至：0.8.2

+   **[要求]**

    Python [模拟](https://pypi.org/project/mock) 库现在是运行单元测试套件所必需的。虽然从 Python 3.3 开始是标准库的一部分，但之前的 Python 安装需要安装它才能运行单元测试或者使用 `sqlalchemy.testing` 包来处理外部方言。

    此更改也已**回溯**至：0.8.2

### 一般

+   **[一般] [功能] [py3k]**

    C 扩展已迁移到 Python 3，并且将在任何支持的 CPython 2 或 3 环境下构建。

    参考：[#2161](https://www.sqlalchemy.org/trac/ticket/2161)

+   **[一般] [功能] [py3k]**

    代码库现在对 Python 2 和 3 进行了“原地”修改，不再需要运行 2to3。兼容性现在针对 Python 2.6 向前。

    参考：[#2671](https://www.sqlalchemy.org/trac/ticket/2671)

+   **[一般]**

    对许多核心模块以及 ORM 模块的导入结构进行了大规模的重构。特别是，`sqlalchemy.sql` 已经被分解成比以前更多的模块，因此非常大的 `sqlalchemy.sql.expression` 现在已经被削减了。该工作的重点是大幅减少导入循环。此外，`sqlalchemy.sql.expression` 和 `sqlalchemy.orm` 中的 API 函数系统已重新组织，以消除函数与它们生成的对象之间文档中的冗余。

### orm

+   **[orm] [功能]**

    添加了新选项到 `relationship()` `distinct_target_key`。这使子查询急加载策略能够对最内部的 SELECT 子查询应用 DISTINCT，以协助解决由最内部查询生成重复行的情况（尚无解决子查询急加载中重复行问题的一般解决方案，但是，当最内部子查询之外的连接生成重复时）。当标志设置为 `True` 时，将无条件地呈现 DISTINCT，当设置为 `None` 时，如果最内部关系的目标列不包含完整的主键，则呈现 DISTINCT。该选项在 0.8 中默认为 False（例如，在所有情况下默认关闭），在 0.9 中默认为 None（例如，默认情况下自动执行）。感谢 Alexander Koval 的帮助。

    另请参见

    子查询急加载将对某些查询的最内部 SELECT 应用 DISTINCT

    此更改也已**回溯**至：0.8.3

    参考：[#2836](https://www.sqlalchemy.org/trac/ticket/2836)

+   **[orm] [功能]**

    当从标量关系中获取标量属性时，关联代理现在会返回`None`，而不是引发`AttributeError`，其中标量关系本身指向`None`。

    请参阅

    关联代理缺失标量返回 None

    参考：[#2810](https://www.sqlalchemy.org/trac/ticket/2810)

+   **[orm] [功能]**

    添加了新方法`AttributeState.load_history()`，类似于`AttributeState.history`，但也触发加载器可调用。

    请参阅

    attributes.get_history()现在默认情况下会从数据库查询，如果值不存在

    参考：[#2787](https://www.sqlalchemy.org/trac/ticket/2787)

+   **[orm] [功能]**

    添加了一个新的加载选项`load_only()`。这允许指定一系列列名作为“仅”加载这些属性，推迟其余属性的加载。

    参考：[#1418](https://www.sqlalchemy.org/trac/ticket/1418)

+   **[orm] [功能]**

    加载器选项系统已经完全重新设计，建立在更全面的基础上，即`Load`对象。这个基础允许任何常见的加载器选项，如`joinedload()`、`defer()`等，以“链式”方式用于指定路径下的选项，例如`joinedload("foo").subqueryload("bar")`。新系统取代了点分隔路径名称、选项中的多个属性以及使用`_all()`选项的用法。

    请参阅

    新的查询选项 API；load_only() 选项

    参考：[#1418](https://www.sqlalchemy.org/trac/ticket/1418)

+   **[orm] [功能]**

    当在基于列的`Query`中使用`composite()`构造时，现在会保留返回对象，而不是展开为单独的列。这在内部使用了新的`Bundle`功能。这种行为是不兼容的；要从一个会展开的复合列中选择，使用`MyClass.some_composite.clauses`。

    请参阅

    复合属性现在在按属性查询时以对象形式返回

    参考：[#2824](https://www.sqlalchemy.org/trac/ticket/2824)

+   **[orm] [功能]**

    添加了一个新的构造`Bundle`，允许将列表达式组指定给`Query`构造。默认情况下，列组将作为单个元组返回。但是，可以覆盖`Bundle`的行为，以提供对返回行的任何类型的结果处理。当在基于列的`Query`中使用时，现在复合属性的行为也嵌入到`Bundle`中。

    另请参阅

    ORM 查询的列捆绑

    当按属性基础查询时，现在复合属性将以对象形式返回

    参考：[#2824](https://www.sqlalchemy.org/trac/ticket/2824)

+   **[orm] [feature]**

    `Mapper`的`version_id_generator`参数现在可以指定依赖于服务器生成的版本标识符，使用触发器或其他数据库提供的版本控制功能，或通过设置`version_id_generator=False`来使用可选的程序值。当使用服务器生成的版本标识符时，ORM 将在可用时使用 RETURNING 立即加载新的版本值，否则将发出第二个 SELECT。

    参考：[#2793](https://www.sqlalchemy.org/trac/ticket/2793)

+   **[orm] [feature]**

    `Mapper`的`eager_defaults`标志现在将允许使用内联 RETURNING 子句获取新生成的默认值，而不是使用第二个 SELECT 语句，适用于支持 RETURNING 的后端。

    参考：[#2793](https://www.sqlalchemy.org/trac/ticket/2793)

+   **[orm] [feature]**

    添加了一个新属性`Session.info`到`Session`；这是一个字典，应用程序可以将任意数据存储在与`Session`相关的本地数据中。`Session.info`的内容也可以使用`Session`或`sessionmaker`的`info`参数进行初始化。

+   **[orm] [feature]**

    现在已实现事件监听器的移除。该功能通过`remove()`函数提供。

    另请参阅

    事件移除 API

    参考：[#2268](https://www.sqlalchemy.org/trac/ticket/2268)

+   **[orm] [feature]**

    属性事件传递 `AttributeImpl` 作为“发起者”令牌的机制已更改；对象现在是一个名为 `Event` 的特定事件对象。此外，属性系统不再根据匹配的“发起者”令牌停止事件；此逻辑已移至特定于 ORM 反向引用事件处理程序的地方，这些处理程序是属性事件重新传播到后续附加/设置/移除操作的典型来源。模拟反向引用处理程序行为的最终用户代码现在必须确保递归事件传播方案被停止，如果该方案不使用反向引用处理程序。使用这个新系统，当对象附加到集合中，与新的多对一关联，与先前的多对一解除关联，然后从先前的集合中移除时，反向引用处理程序现在可以执行“两跳”操作。在此更改之前，从先前集合中移除的最后一步操作将不会发生。

    另请参阅

    Backref 处理程序现在可以传播多个级别

    参考：[#2789](https://www.sqlalchemy.org/trac/ticket/2789)

+   **[orm] [feature]**

    关于 ORM 构建右侧为 JOIN 或 LEFT OUTER JOIN 的连接的重大变更。现在 ORM 配置为允许形式为 `a JOIN (b JOIN c ON b.id=c.id) ON a.id=b.id` 的连接简单嵌套，而不是强制右侧成为 `SELECT` 子查询。这应该可以在大多数后端上实现显著的性能改进，尤其是 MySQL。多年来一直阻碍此变更的一个数据库后端，SQLite，现在通过将 `SELECT` 子查询的生成从 ORM 移动到 SQL 编译器来解决；因此，在 SQLite 上的右侧嵌套连接最终仍将呈现为 `SELECT`，而所有其他后端不再受此解决方法的影响。

    作为这一变更的一部分，`aliased()`、`Join.alias()` 和 `with_polymorphic()` 函数现在添加了一个新参数 `flat=True`，允许生成一个 JOIN 的“别名”，该别名对加入的每个组件表应用一个匿名别名，而不是生成一个子查询。

    另请参阅

    许多 JOIN 和 LEFT OUTER JOIN 表达式将不再被包裹在 (SELECT * FROM ..) AS ANON_1 中

    参考：[#2587](https://www.sqlalchemy.org/trac/ticket/2587)

+   **[orm] [bug]**

    修复了列表插入操作`insert(0, item)`时，列表仪器化未能正确表示`[0:0]`的切片设置，特别是在使用关联代理时可能发生的情况。由于 Python 集合中的一些怪癖，该问题在 Python 3 中比在 Python 2 中更有可能发生。

    此更改也**回溯**到：0.8.3, 0.7.11

    参考：[#2807](https://www.sqlalchemy.org/trac/ticket/2807)

+   **[orm] [bug]**

    修复了一个 bug，即在与父 `Table` 关联之前在 `Column` 上使用 `remote()` 或 `foreign()` 等注释可能会导致与父表相关的问题，因为注释���行的固有复制操作。

    此更改也**回溯**到：0.8.3

    参考：[#2813](https://www.sqlalchemy.org/trac/ticket/2813)

+   **[orm] [bug]**

    修复了一个 bug，即 `Query.exists()` 在没有任何 WHERE 条件的情况下无法正常工作。感谢 Vladimir Magamedov。

    此更改也**回溯**到：0.8.3

    参考：[#2818](https://www.sqlalchemy.org/trac/ticket/2818)

+   **[orm] [bug]**

    修复了 ORM 用于迭代映射器层次结构的有序序列实现中的潜在问题；在 Jython 解释器下，这个实现没有排序，尽管 cPython 和 PyPy 保持了排序。

    此更改也**回溯**到：0.8.3

    参考：[#2794](https://www.sqlalchemy.org/trac/ticket/2794)

+   **[orm] [bug]**

    修复了 ORM 级别事件注册中“原始”或“传播”标志在某些“未映射基类”配置中可能被错误配置的 bug。

    此更改也**回溯**到：0.8.3

    参考：[#2786](https://www.sqlalchemy.org/trac/ticket/2786)

+   **[orm] [bug]**

    与加载映射实体时使用 `defer()` 选项相关的性能修复。在加载时间将一个每个对象延迟可调用应用到实例的函数开销明显高于仅从行加载数据的开销（请注意，`defer()` 旨在减少数据库/网络开销，而不一定是函数调用次数）；在所有情况下，函数调用开销现在小于从列加载数据的开销。每次从 N（结果中的总延迟值）加载的“延迟可调用”对象数量也从 N 减少到 1（延迟列的总数）。

    此更改也**回溯**到：0.8.3

    参考：[#2778](https://www.sqlalchemy.org/trac/ticket/2778)

+   **[orm] [bug]**

    修复了一个 bug，即当我们使用 `make_transient()` 函数将对象从“持久”移动到“挂起”时，涉及基于集合的反向引用的操作中，属性历史函数会失败。

    此更改也**回溯**到：0.8.3

    参考：[#2773](https://www.sqlalchemy.org/trac/ticket/2773)

+   **[orm] [bug]**

    当尝试刷新一个继承类的对象时，如果多态鉴别器已分配给对该类无效的值，则会发出警告。

    这个更改也被**回溯**到：0.8.2

    参考：[#2750](https://www.sqlalchemy.org/trac/ticket/2750)

+   **[orm] [bug]**

    修复了多态 SQL 生成中的错误，当多个继承实体针对相同的基类相互连接时，如果连接字符串超过两个实体，则基表上的列不会独立跟踪彼此。

    这个更改也被**回溯**到：0.8.2

    参考：[#2759](https://www.sqlalchemy.org/trac/ticket/2759)

+   **[orm] [bug]**

    修复了将复合属性发送到`Query.order_by()`会产生一些数据库不接受的括号表达式的错误。

    这个更改也被**回溯**到：0.8.2

    参考：[#2754](https://www.sqlalchemy.org/trac/ticket/2754)

+   **[orm] [bug]**

    修复了复合属性与`aliased()`函数之间的交互。以前，在应用别名时，复合属性在比较操作中不会正确工作。

    这个更改也被**回溯**到：0.8.2

    参考：[#2755](https://www.sqlalchemy.org/trac/ticket/2755)

+   **[orm] [bug] [ext]**

    修复了`MutableDict`在调用`clear()`时未报告更改事件的错误。

    这个更改也被**回溯**到：0.8.2

    参考：[#2730](https://www.sqlalchemy.org/trac/ticket/2730)

+   **[orm] [bug]**

    当与标量列映射属性一起使用时，`get_history()`现在将遵守传递给它的“被动”标志；由于这默认为`PASSIVE_OFF`，如果值不存在，默认情况下该函数将查询数据库。这与 0.8 版本相比是一种行为变化。

    另请参阅

    attributes.get_history()将默认从数据库查询，如果值不存在

    参考：[#2787](https://www.sqlalchemy.org/trac/ticket/2787)

+   **[orm] [bug] [associationproxy]**

    对于与标量值一起使用的 ==、!= 比较器添加了额外的标准，以便比较 None 时也考虑到关联记录本身不存在，除了现有的对关联记录的标量端点为 NULL 的测试。之前，比较 `Cls.scalar == None` 将返回 `Cls.associated` 存在且 `Cls.associated.scalar` 为 None 的记录，但不会返回 `Cls.associated` 不存在的行。更重要的是，反向操作 `Cls.scalar != None` *将* 返回 `Cls` 行，其中 `Cls.associated` 不存在。

    `Cls.scalar != 'somevalue'` 的情况也被修改，更像直接的 SQL 比较；只有当 `Cls.associated` 存在且 `Associated.scalar` 非 NULL 且不等于 `'somevalue'` 时才返回行。之前，这将是一个简单的 `NOT EXISTS`。

    还添加了一个特殊用例，当 `Cls.scalar` 是基于列的值时，可以调用 `Cls.scalar.has()` 而不带参数，这将返回 `Cls.associated` 是否有任何行存在，而不管 `Cls.associated.scalar` 是否为 NULL。

    另请参阅

    关联代理 SQL 表达式改进和修复

    参考：[#2751](https://www.sqlalchemy.org/trac/ticket/2751)

+   **[orm] [bug]**

    修复了一个晦涩的 bug，在跨多对多关系连接/联接加载到具有特定鉴别器值的单表继承子类时，错误的结果将被获取，这是由于返回的“secondary”行。现在，在所有 ORM 关联的多对多关系中，“secondary” 和右侧表现在括号内进行内部连接，以便左->右连接可以准确过滤。这一变化得以实现，最终解决了关于右侧嵌套连接的问题，详见 [#2587](https://www.sqlalchemy.org/trac/ticket/2587)。

    另请参阅

    许多 JOIN 和 LEFT OUTER JOIN 表达式将不再被包装在 (SELECT * FROM ..) AS ANON_1 中

    参考：[#2369](https://www.sqlalchemy.org/trac/ticket/2369)

+   **[orm] [bug]**

    `Query.select_from()` 方法的“自动别名”行为已关闭。现在特定行为可以通过新方法 `Query.select_entity_from()` 获得。这里的自动别名行为从未有很好的文档记录，通常也不是所需的，因为 `Query.select_from()` 更多地用于控制 JOIN 的呈现方式。`Query.select_entity_from()` 也将在 0.8 版本中提供，以便依赖自动别名的应用程序可以转向使用此方法。

    另请参阅

    _query.Query.select_from() 不再将子句应用于相应的实体

    参考：[#2736](https://www.sqlalchemy.org/trac/ticket/2736)

### orm declarative

+   **[orm] [declarative] [feature]**

    添加了一个方便的类装饰器`as_declarative()`，它是对`declarative_base()` 的包装，允许使用一种巧妙的类装饰方法应用现有的基类。

    此更改也已**反向移植**到：0.8.3

+   **[orm] [declarative] [feature]**

    现在可以在字符串参数中使用`order_by`、`primaryjoin`或类似参数中使用 ORM 描述符，如混合属性，来引用 ORM 描述符，如混合属性，在`relationship()` 中，除了列绑定属性之外。

    此更改也已**反向移植**到：0.8.2

    参考：[#2761](https://www.sqlalchemy.org/trac/ticket/2761)

### 例子

+   **[examples] [feature]**

    改进了`examples/generic_associations`中的示例，包括`discriminator_on_association.py`利用单表继承来处理“鉴别器”的工作。还添加了一个真正的“通用外键”示例，其工作方式类似于其他流行框架，它使用开放式整数指向任何其他表，放弃了传统的引用完整性。虽然我们不建议使用这种模式，但信息渴望自由。

    此更改也已**反向移植**到：0.8.3

+   **[examples] [bug]**

    在版本控制示例中创建的历史表中添加了“autoincrement=False”，因为该表在任何情况下都不应具有自动增量，由 Patrick Schmid 提供。

    此更改也已**反向移植**到：0.8.3

+   **[examples] [bug]**

    修复了“版本控制”配方中的一个问题，即当存在反向引用时，多对一引用可能会为目标产生一个无意义的版本，即使它没有被更改。修补来自 Matt Chisholm。

    此更改也已**反向移植**到：0.8.2

### 引擎

+   **[engine] [feature]**

    `Engine` 的`URL` 的`repr()`现在将使用星号隐藏密码。由 Gunnlaugur Þór Briem 提供。

    此更改也已**反向移植**到：0.8.3

    参考：[#2821](https://www.sqlalchemy.org/trac/ticket/2821)

+   **[engine] [feature]**

    `ConnectionEvents` 中添加了新的事件：

    +   `ConnectionEvents.engine_connect()`

    +   `ConnectionEvents.set_connection_execution_options()`

    +   `ConnectionEvents.set_engine_execution_options()`

    参考：[#2770](https://www.sqlalchemy.org/trac/ticket/2770)

+   **[engine] [bug]**

    `make_url()`函数使用的正则表达式现在解析 ipv6 地址，例如用方括号括起来。

    此更改也**回溯**到：0.8.3, 0.7.11

    参考：[#2851](https://www.sqlalchemy.org/trac/ticket/2851)

+   **[engine] [bug] [oracle]**

    如果重新创建`Engine`时出现断开错误，将不会再第二次调用 Dialect.initialize()。这修复了 Oracle 8 方言中的一个特定问题，但通常情况下，dialect.initialize()阶段应该只执行一次。

    此更改也**回溯**到：0.8.3

    参考：[#2776](https://www.sqlalchemy.org/trac/ticket/2776)

+   **[engine] [bug] [pool]**

    修复了`QueuePool`在现有池化连接在无效或重新生成事件后未能重新连接时会丢失正确的已检出计数的错误。

    此更改也**回溯**到：0.8.3

    参考：[#2772](https://www.sqlalchemy.org/trac/ticket/2772)

+   **[engine] [bug]**

    修复了各种`Pool`实现中`reset_on_return`参数在重新生成池时不会传播的错误。感谢 Eevee。

    此更改也**回溯**到：0.8.2

+   **[engine] [bug]**

    `Dialect.reflecttable()`方法的方法签名已经更改，所有已知情况下由`DefaultDialect`提供，现在期望`include_columns`和`exclude_columns`参数没有任何 kw 选项，减少了歧义 - 以前缺少了`exclude_columns`。

    参考：[#2748](https://www.sqlalchemy.org/trac/ticket/2748)

### sql

+   **[sql] [feature]**

    添加了对“唯一约束”反射的支持，通过`Inspector.get_unique_constraints()`方法。感谢 Roman Podolyaka 的补丁。

    此更改也**回溯**到：0.8.4

    参考：[#1443](https://www.sqlalchemy.org/trac/ticket/1443)

+   **[sql] [feature]**

    `update()`、`insert()`和`delete()`构造现在将 ORM 实体解释为要操作的目标表，例如：

    ```py
    from sqlalchemy import insert, update, delete

    ins = insert(SomeMappedClass).values(x=5)

    del_ = delete(SomeMappedClass).where(SomeMappedClass.id == 5)

    upd = update(SomeMappedClass).where(SomeMappedClass.id == 5).values(name="ed")
    ```

    此更改也**回溯**到：0.8.3

+   **[sql] [feature] [mysql] [postgresql]**

    PostgreSQL 和 MySQL 方言现在支持反射/检查外键选项，包括 ON UPDATE、ON DELETE。PostgreSQL 还反映了 MATCH、DEFERRABLE 和 INITIALLY。感谢 ijl。

    参考：[#2183](https://www.sqlalchemy.org/trac/ticket/2183)

+   **[sql] [feature]**

    当在类型表达式中使用具有“null”类型（例如未指定类型）的 `bindparam()` 构造时，现在会复制该构造，并将新副本分配给比较列的实际类型。以前，这种逻辑会在给定的 `bindparam()` 中发生。此外，在构造的编译阶段，现在对传递给 `ValuesBase.values()` 用于 `Insert` 或 `Update` 构造的 `bindparam()` 构造也会发生类似的过程。

    这些都是一些微妙的行为变化，可能会影响某些用法。

    另请参阅

    当类型可用时，没有类型的 bindparam() 构造通过复制升级

    参考：[#2850](https://www.sqlalchemy.org/trac/ticket/2850)

+   **[sql] [feature]**

    对特殊符号的表达式处理进行了彻底改革，特别是连接词，例如 `None` `null()` `true()` `false()`，包括在连接中呈现 NULL 的一致性，包含布尔常量的 `and_()` 和 `or_()` 表达式的“短路”，以及对布尔常量和表达式的呈现与不支持 `true`/`false` 常量的后端相比较为“1”或“0”。

    另请参阅

    改进的布尔常量、NULL 常量、连接的呈现

    参考：[#2734](https://www.sqlalchemy.org/trac/ticket/2734)，[#2804](https://www.sqlalchemy.org/trac/ticket/2804)，[#2823](https://www.sqlalchemy.org/trac/ticket/2823)

+   **[sql] [feature]**

    现在，打字系统处理呈现“文字绑定”值的任务，例如通常绑定参数但由于上下文必须呈现为字符串的值，通常在 DDL 构造中，例如 CHECK 约束和索引（请注意，“文字绑定”值从 DDL 中使用，如[#2742](https://www.sqlalchemy.org/trac/ticket/2742)）。一个新方法`TypeEngine.literal_processor()`作为基础，添加了`TypeDecorator.process_literal_param()`以允许包装本机文字呈现方法。

    另请参阅

    打字系统现在处理呈现“文字绑定”值的任务

    参考：[#2838](https://www.sqlalchemy.org/trac/ticket/2838)

+   **[sql] [feature]**

    `Table.tometadata()`方法现在会生成所有`SchemaItem`对象中的所有`SchemaItem.info`字典的副本，包括列、约束、外键等。由于这些字典是副本，它们独立于原始字典。以前，仅传输了此操作中`Column`的`.info`字典，并且仅在原地链接，而不是复制。

    参考：[#2716](https://www.sqlalchemy.org/trac/ticket/2716)

+   **[sql] [feature]**

    `Column`的`default`参数现在接受类或对象方法作为参数，除了独立函数；将正确检测是否接受“上下文”参数。

+   **[sql] [feature]**

    添加了新方法到`insert()`构造`Insert.from_select()`。给定列的列表和可选择的，呈现`INSERT INTO (table) (columns) SELECT ..`。虽然此功能作为 0.9 的一部分突出显示，但也已回溯到 0.8.3。

    另请参阅

    从 SELECT 插入

    参考：[#722](https://www.sqlalchemy.org/trac/ticket/722)

+   **[sql] [feature]**

    为`TypeDecorator`提供了一个名为`TypeDecorator.coerce_to_is_types`的新属性，以便更容易控制使用`==`或`!=`与`None`和布尔类型进行比较时如何生成`IS`表达式，或者带有绑定参数的普通相等表达式。

    参考：[#2734](https://www.sqlalchemy.org/trac/ticket/2734), [#2744](https://www.sqlalchemy.org/trac/ticket/2744)

+   **[sql] [feature]**

    如果在`SELECT`的列子句中也引用了`label()`构造，那么`label()`构造现在将仅在`ORDER BY`子句中呈现其名称，而不是重写完整表达式。这使得数据库有更好的机会优化在两个不同上下文中评估相同表达式。

    另请参见

    标签构造现在可以仅在 ORDER BY 中呈现其名称

    参考：[#1068](https://www.sqlalchemy.org/trac/ticket/1068)

+   **[sql] [bug]**

    修复了自 0.7.9 以来的回归，即如果在多个 FROM 子句中引用了 CTE 的名称，则可能无法正确引用。

    此更改还**回溯**到：0.8.3, 0.7.11

    参考：[#2801](https://www.sqlalchemy.org/trac/ticket/2801)

+   **[sql] [bug] [cte]**

    修复了通用表达式系统中的一个错误，如果 CTE 仅被用作`alias()`构造，则不会使用 WITH 关键字呈现。

    此更改还**回溯**到：0.8.3, 0.7.11

    参考：[#2783](https://www.sqlalchemy.org/trac/ticket/2783)

+   **[sql] [bug]**

    修复了`CheckConstraint` DDL 中的错误，其中来自`Column`对象的“quote”标志不会传播。

    此更改还**回溯**到：0.8.3, 0.7.11

    参考：[#2784](https://www.sqlalchemy.org/trac/ticket/2784)

+   **[sql] [bug]**

    修复了`type_coerce()`无法正确解释具有`__clause_element__()`方法的 ORM 元素的错误。

    此更改还**回溯**到：0.8.3

    参考：[#2849](https://www.sqlalchemy.org/trac/ticket/2849)

+   **[sql] [bug]**

    当为“非本地”类型生成 CHECK 约束时，`Enum`和`Boolean`类型现在会绕过任何自定义（例如 TypeDecorator）类型。这样，自定义类型不会参与 CHECK 中的表达式，因为此表达式针对“impl”值而不是“decorated”值。

    这个更改也被**回溯**到：0.8.3

    参考：[#2842](https://www.sqlalchemy.org/trac/ticket/2842)

+   **[sql] [bug]**

    如果从未指定`unique`（默认为`None`）的`Column`生成了`Index`上的`.unique`标志，那么该标志现在将始终为`True`或`False`。

    这个更改也被**回溯**到：0.8.3

    参考：[#2825](https://www.sqlalchemy.org/trac/ticket/2825)

+   **[sql] [bug]**

    修复了默认编译器以及 postgresql、mysql 和 mssql 的 bug，以确保在 CREATE INDEX 语句中任何文本 SQL 表达式值都直接呈现为文本，而不是作为绑定参数。这也改变了其他 DDL（如约束）的呈现方案。

    这个更改也被**回溯**到：0.8.3

    参考：[#2742](https://www.sqlalchemy.org/trac/ticket/2742)

+   **[sql] [bug]**

    一个在其 FROM 子句中引用自身的`select()`，通常通过就地突变，将引发一个信息性错误消息，而不是导致递归溢出。

    这个更改也被**回溯**到：0.8.3

    参考：[#2815](https://www.sqlalchemy.org/trac/ticket/2815)

+   **[sql] [bug]**

    修复了一个 bug，即使用`column_reflect`事件来更改传入的`Column`的`.key`会阻止主键约束、索引和外键约束被正确反映的问题。

    这个更改也被**回溯**到：0.8.3

    参考：[#2811](https://www.sqlalchemy.org/trac/ticket/2811)

+   **[sql] [bug]**

    在 0.8 版本中添加的`ColumnOperators.notin_()`运算符现在在针对空集合时正确地产生“IN”表达式的否定。

    这个更改也被**回溯**到：0.8.3

+   **[sql] [bug] [postgresql]**

    修复了表达式系统在引用`select()`构造中的`.c`集合时依赖于一些表达式的`str()`形式的 bug，但是由于元素依赖于方言特定的编译构造，特别是与 PostgreSQL 的`ARRAY`元素一起使用的`__getitem__()`运算符，因此`str()`形式不可用。修复还添加了一个新的异常类`UnsupportedCompilationError`，在编译器被要求编译它不知道如何处理的内容时引发该异常。

    这个改变也被**回溯**到：0.8.3 版本

    引用：[#2780](https://www.sqlalchemy.org/trac/ticket/2780)

+   **[sql] [bug]**

    对`Select`构造中首次引入的 0.8.0 版本的关联行为进行了多次修��：

    +   为了满足 FROM 条目应该向外关联到包含另一个 SELECT 的 SELECT 的用例，然后再包含这个 SELECT 的情况，当通过`Select.correlate()`建立显式关联时，关联现在可以跨多个级别进行，前提是目标 SELECT 在由 WHERE/ORDER BY/columns 子句包含的链中的某个位置，而不仅仅是嵌套的 FROM 子句。这使得`Select.correlate()`再次更加兼容 0.7 版本，同时仍然保持新的“智能”关联。

    +   当没有使用显式关联时，“隐式”关联将其行为限制在仅仅是直接封闭的 SELECT 中，以最大程度地兼容 0.7 版本的应用程序，并且在这种情况下还防止了跨嵌套 FROM 子句的关联，保持了与 0.8.0/0.8.1 版本的兼容性。

    +   `Select.correlate_except()` 方法在某些情况下未能阻止给定的 FROM 子句进行关联，并且还会导致 FROM 子句被错误地完全省略（更像是 0.7 版本的行为），这个问题已经修复。

    +   调用 select.correlate_except(None)将使所有 FROM 子句进入关联，就像预期的那样。

    这个改变也被**回溯**到：0.8.2 版本

    引用：[#2668](https://www.sqlalchemy.org/trac/ticket/2668)，[#2746](https://www.sqlalchemy.org/trac/ticket/2746)

+   **[sql] [bug]**

    修复了一个 bug，即将一个具有多个外键路径到表“B”的表“A”的 select()与表“B”连接时，与直接将表“A”连接到“B”时报告的“模糊连接条件”错误不同，它会产生具有多个条件的连接条件。

    这个改变也被**回溯**到：0.8.2 版本

    参考：[#2738](https://www.sqlalchemy.org/trac/ticket/2738)

+   **[sql] [bug] [reflection]**

    修复了在跨远程模式和本地模式使用`MetaData.reflect()`可能会产生错误结果的 bug，在这种情况下，两个模式都有相同名称的表。 

    此更改也**回溯**到：0.8.2

    参考：[#2728](https://www.sqlalchemy.org/trac/ticket/2728)

+   **[sql] [bug]**

    从基本`ColumnOperators`类中删除了“not implemented” `__iter__()`调用，尽管这是在 0.8.0 中引入的，以防止在自定义运算符上实现`__getitem__()`方法并在该对象上错误调用`list()`时出现无限增长的内存循环，但它导致列元素报告它们实际上是可迭代类型，然后在尝试迭代时抛出错误。在这里没有真正的方法来同时拥有两边，所以我们坚持使用 Python 最佳实践。在自定义运算符上实现`__getitem__()`时要小心！

    此更改也**回溯**到：0.8.2

    参考：[#2726](https://www.sqlalchemy.org/trac/ticket/2726)

+   **[sql] [bug]**

    在调用“attach”事件之前，`Index`上设置了“name”属性，以便可以使用附加事件根据父表和/或列动态生成索引名称。

    参考：[#2835](https://www.sqlalchemy.org/trac/ticket/2835)

+   **[sql] [bug]**

    已从`ForeignKey`对象中删除了错误的 kw 参数“schema”。这是一个意外提交，什么也没做；当使用此 kw 参数时，0.8.3 会发出警告。

    参考：[#2831](https://www.sqlalchemy.org/trac/ticket/2831)

+   **[sql] [bug]**

    对“引号”标识符处理方式进行了重新设计，不再依赖于传递各种`quote=True`标志，而是将这些标志转换为包含引号信息的丰富字符串对象，并在传递给像`Table`、`Column`等常见模式构造时包含引号信息。这解决了各种方法未正确遵守“quote”标志的问题，例如`Engine.has_table()`和相关方法。`quoted_name`对象是一个字符串子类，如果需要，也可以显式使用；该对象将保留传递的引号偏好，并且还将绕过标准化为大写符号的方言执行的“名称规范化”。这样，“大写”后端现在可以使用强制引号名称，例如小写引号名称和新保留字。

    另请参阅

    模式标识符现在携带自己的引号信息

    参考：[#2812](https://www.sqlalchemy.org/trac/ticket/2812)

+   **[sql] [bug]**

    `ForeignKey`对象解析为其目标`Column`的过程已经重新设计，尽可能立即进行，基于目标`Column`与与此`ForeignKey`关联的相同`MetaData`关联的时刻，而不是等待构建联接或类似的第一次。这与其他改进一起，允许更早地检测到一些外键配置问题。此外，还对类型传播系统进行了重新设计，因此现在应该可以可靠地在通过`ForeignKey`引用另一个`Column`的任何`Column`上将类型设置为`None` - 该类型将在另一个列关联时立即从目标列复制，并且现在也适用于复合外键。

    另请参阅

    列可以可靠地从通过 ForeignKey 引用的列获取其类型

    参考：[#1765](https://www.sqlalchemy.org/trac/ticket/1765)

### postgresql

+   **[postgresql] [feature]**

    添加了对 PostgreSQL 9.2 范围类型的支持。目前，不提供类型转换，因此目前直接使用字符串或 psycopg2 2.5 范围扩展类型。补丁感谢 Chris Withers。

    此更改也被**回溯**到：0.8.2

+   **[postgresql] [feature]**

    在使用 psycopg2 DBAPI 时，添加了对“AUTOCOMMIT”隔离的支持。该关键字可通过 `isolation_level` 执行选项使用。补丁感谢 Roman Podolyaka。

    此更改也被**回溯**到：0.8.2

    参考：[#2072](https://www.sqlalchemy.org/trac/ticket/2072)

+   **[postgresql] [feature]**

    当在主键自增列上使用 `SmallInteger` 类型时，基于 PostgreSQL 版本 9.2 或更高版本的服务器版本检测，添加了对 `SMALLSERIAL` 的渲染支持。

    参考：[#2840](https://www.sqlalchemy.org/trac/ticket/2840)

+   **[postgresql] [bug]**

    从列的服务器默认反射中删除了 128 个字符的截断；此代码最初来自于 PG 系统视图，用于可读性截断字符串。

    此更改也被**回溯**到：0.8.3

    参考：[#2844](https://www.sqlalchemy.org/trac/ticket/2844)

+   **[postgresql] [bug]**

    在 CREATE INDEX 语句的列列表中，将对复合 SQL 表达式应用括号。

    此更改也被**回溯**到：0.8.3

    参考：[#2742](https://www.sqlalchemy.org/trac/ticket/2742)

+   **[postgresql] [bug]**

    修复了一个 bug，其中在 PostgreSQL 版本字符串中，单词“PostgreSQL”或“EnterpriseDB”之前有前缀时，无法解析。感谢 Scott Schaefer。

    此更改也被**回溯**到：0.8.3

    参考：[#2819](https://www.sqlalchemy.org/trac/ticket/2819)

+   **[postgresql] [bug]**

    在 PostgreSQL 方言上，`extract()` 的行为已经简化，不再向给定表达式中注入硬编码的 `::timestamp` 或类似的转换，因为这会干扰诸如时区感知日期时间之类的类型，但在现代版本的 psycopg2 中似乎也完全不必要。

    此更改也被**回溯**到：0.8.2

    参考：[#2740](https://www.sqlalchemy.org/trac/ticket/2740)

+   **[postgresql] [bug]**

    修复了 HSTORE 类型中包含反斜杠引号的键/值在使用“非本地”（即非-psycopg2）手段转换 HSTORE 数据时无法正确转义的 bug。补丁感谢 Ryan Kelly。

    此更改也被**回溯**到：0.8.2

    参考：[#2766](https://www.sqlalchemy.org/trac/ticket/2766)

+   **[postgresql] [bug]**

    修复了多列 PostgreSQL 索引中列的顺序会反映错误顺序的 bug。感谢 Roman Podolyaka。

    此更改也被**回溯**到：0.8.2

    参考：[#2767](https://www.sqlalchemy.org/trac/ticket/2767)

### mysql

+   **[mysql] [feature]**

    与 `Index` 一起使用的 `mysql_length` 参数现在可以作为列名/长度的字典传递，用于复合索引。非常感谢 Roman Podolyaka 提供的补丁。

    此更改也已**反向移植**到：0.8.2

    参考：[#2704](https://www.sqlalchemy.org/trac/ticket/2704)

+   **[mysql] [功能]**

    MySQL 的 `SET` 类型现在具有与 `ENUM` 相同的自动引用行为。在设置值时不需要引号，但存在的引号将与警告一起自动检测。这也有助于 Alembic，其中 SET 类型不会呈现带引号。

    参考：[#2817](https://www.sqlalchemy.org/trac/ticket/2817)

+   **[mysql] [错误]**

    更新 MySQL 版本 5.5、5.6 的保留字，感谢 Hanno Schlichting。

    此更改也已**反向移植**到：0.8.3，0.7.11

    参考：[#2791](https://www.sqlalchemy.org/trac/ticket/2791)

+   **[mysql] [错误]**

    在 [#2721](https://www.sqlalchemy.org/trac/ticket/2721) 中的更改是，`ForeignKeyConstraint` 的 `deferrable` 关键字在 MySQL 后端被悄悄地忽略，将在 0.9 版本中还原；此关键字现在将再次呈现，并在 MySQL 上引发错误，因为它不被理解 - 相同的行为也将适用于 `initially` 关键字。在 0.8 中，关键字将继续被忽略，但会发出警告。此外，`match` 关键字现在在 0.9 上引发 `CompileError`，在 0.8 上发出警告；这个关键字不仅在 MySQL 上被悄悄地忽略，而且还破坏了 ON UPDATE/ON DELETE 选项。

    若要使用在 MySQL 上不呈现或以不同方式呈现的 `ForeignKeyConstraint`，请使用自定义编译选项。文档中已添加了此用法示例，请参阅 MySQL / MariaDB 外键。

    此更改也已**反向移植**到：0.8.3

    参考：[#2721](https://www.sqlalchemy.org/trac/ticket/2721)，[#2839](https://www.sqlalchemy.org/trac/ticket/2839)

+   **[mysql] [错误]**

    MySQL-connector 方言现在允许在 create_engine 查询字符串中使用选项来覆盖连接中设置的默认值，包括“buffered”和“raise_on_warnings”。

    此更改也已**反向移植**到：0.8.3

    参考：[#2515](https://www.sqlalchemy.org/trac/ticket/2515)

+   **[mysql] [错误]**

    修复了在使用多表 UPDATE 时出现的 bug，其中一个补充表是带有自己绑定参数的 SELECT，当使用 MySQL 的特殊语法时，绑定参数的位置与语句本身相反。

    此更改也**反向移植**到：0.8.2

    参考：[#2768](https://www.sqlalchemy.org/trac/ticket/2768)

+   **[mysql] [bug]**

    在`mysql+gaerdbms`方言中添加了另一个条件，用于检测所谓的“开发”模式，在这种模式下，我们应该使用`rdbms_mysqldb` DBAPI。修补程序由 Brett Slatkin 提供。

    此更改也**反向移植**到：0.8.2

    参考：[#2715](https://www.sqlalchemy.org/trac/ticket/2715)

+   **[mysql] [bug]**

    在`ForeignKey`和`ForeignKeyConstraint`上的`deferrable`关键字参数将不会在 MySQL 方言上呈现`DEFERRABLE`关键字。很长一段时间以来，我们一直保留这个设置，因为一个非延迟的外键与一个延迟的外键会有非常不同的行为，但是一些环境只是在 MySQL 上禁用 FKs，所以我们在这里会少些主观意见。

    此更改也**反向移植**到：0.8.2

    参考：[#2721](https://www.sqlalchemy.org/trac/ticket/2721)

+   **[mysql] [bug]**

    修复并测试了 MySQL 反射中外键选项的解析；这是对[#2183](https://www.sqlalchemy.org/trac/ticket/2183)中的工作的补充，我们开始支持外键选项的反射，例如 ON UPDATE/ON DELETE 级联。

    参考：[#2839](https://www.sqlalchemy.org/trac/ticket/2839)

+   **[mysql] [bug]**

    改进了对 cymysql 驱动程序的支持，支持版本 0.6.5，由 Hajime Nakagami 提供。

### sqlite

+   **[sqlite] [bug]**

    新添加的 SQLite DATETIME 参数 storage_format 和 regexp 显然没有完全正确实现；虽然参数被接受，但实际上它们没有任何效果；这个问题已经修复。

    此更改也**反向移植**到：0.8.3

    参考：[#2781](https://www.sqlalchemy.org/trac/ticket/2781)

+   **[sqlite] [bug]**

    将`sqlalchemy.types.BIGINT`添加到 SQLite 方言可以反射的类型名称列表中；由 Russell Stuart 提供。

    此更改也**反向移植**到：0.8.2

    参考：[#2764](https://www.sqlalchemy.org/trac/ticket/2764)

### mssql

+   **[mssql] [bug]**

    在 SQL Server 2000 上查询信息模式时，删除了在 0.8.1 中添加的 CAST 调用，以帮助处理驱动程序问题，显然这在 2000 年不兼容。CAST 保留在 SQL Server 2005 及更高版本中。

    此更改也**反向移植**到：0.8.2

    参考：[#2747](https://www.sqlalchemy.org/trac/ticket/2747)

+   **[mssql] [bug] [pyodbc]**

    修复了 MSSQL 与 Python 3 + pyodbc 的问题，包括正确传递语句。

    参考：[#2355](https://www.sqlalchemy.org/trac/ticket/2355)

### oracle

+   **[oracle] [功能] [py3k]**

    使用 cx_oracle 的 Oracle 单元测试现在在 Python 3 下完全通过。

+   **[oracle] [错误]**

    修复了使用同义词反射 Oracle 表时的 bug，如果同义词和表位于不同的远程模式中，则会失败。修复补丁由 Kyle Derr 提供。

    此更改也已**回溯**至：0.8.3

    参考：[#2853](https://www.sqlalchemy.org/trac/ticket/2853)

### 杂项

+   **[功能]**

    为 `Column` 添加了一个新标志 `system=True`，将列标记为“系统”列，该列由数据库自动添加（例如 PostgreSQL 的 `oid` 或 `xmin`）。该列将在 `CREATE TABLE` 语句中被省略，但在其他情况下可用于查询。此外，`CreateColumn` 构造可以应用于自定义编译���则，允许跳过列，通过生成返回 `None` 的规则。

    此更改也已**回溯**至：0.8.3

+   **[功能] [firebird]**

    为 kinterbasdb 和 fdb 方言添加了新标志 `retaining=True`。这控制发送到 DBAPI 连接的 `commit()` 和 `rollback()` 方法的 `retaining` 标志的值。由于历史原因，此标志在 0.8.2 中默认为 `True`，但在 0.9.0b1 中此标志默认为 `False`。

    此更改也已**回溯**至：0.8.2

    参考：[#2763](https://www.sqlalchemy.org/trac/ticket/2763)

+   **[功能] [核心]**

    在 `UpdateBase.returning()` 中添加了一个新的变体，称为 `ValuesBase.return_defaults()`；这允许向语句的 RETURNING 子句中添加任意列，而不会干扰编译器通常的“隐式返回”功能，该功能用于高效地获取新生成的主键值。对于支持的后端，所有获取的值的字典都存在于 `ResultProxy.returned_defaults` 中。

    参考：[#2793](https://www.sqlalchemy.org/trac/ticket/2793)

+   **[功能] [池]**

    为“rollback-on-return”和较少使用的“commit-on-return”添加了池记录。这与池“debug”记录一起启用。

    参考：[#2752](https://www.sqlalchemy.org/trac/ticket/2752)

+   **[功能] [firebird]**

    当未指定方言限定符时，`fdb` 方言现在是默认方言，即 `firebird://`，因为 Firebird 项目将 `fdb` 发布为其官方 Python 驱动程序。

    参考：[#2504](https://www.sqlalchemy.org/trac/ticket/2504)

+   **[错误] [firebird]**

    当反射 Firebird 类型 LONG 和 INT64 时，类型查找已修复，使得 LONG 被视为 INTEGER，INT64 被视为 BIGINT，除非该类型具有“精度”，在这种情况下，它被视为 NUMERIC。补丁由 Russell Stuart 提供。

    此更改也已**回溯**至：0.8.2

    参考：[#2757](https://www.sqlalchemy.org/trac/ticket/2757)

+   **[错误] [扩展]**

    修复了一个错误，即如果使用函数而不是类设置复合类型，则当尝试检查该列是否为`MutableComposite`时，可变扩展会出错（实际上不是）。感谢 asldevi。

    此更改也**回溯**到：0.8.2

+   **[要求]**

    现在运行单元测试套件需要 Python [mock](https://pypi.org/project/mock) 库。虽然作为 Python 3.3 的一部分，但之前的 Python 安装需要安装此库才能运行单元测试或使用`sqlalchemy.testing`包来处理外部方言。

    此更改也**回溯**到：0.8.2
