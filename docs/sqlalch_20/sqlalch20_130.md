# 1.1 更改日志

> 原文：[`docs.sqlalchemy.org/en/20/changelog/changelog_11.html`](https://docs.sqlalchemy.org/en/20/changelog/changelog_11.html)

## 1.1.18

发布日期：2018 年 3 月 6 日

### postgresql

+   **[postgresql] [bug] [py3k]**

    修复了在 PostgreSQL COLLATE / ARRAY 调整中首次引入的问题，在 Python 3.7 正则表达式的新行为导致修复失败的情况。首次引入：[#4006](https://www.sqlalchemy.org/trac/ticket/4006)。

    参考：[#4208](https://www.sqlalchemy.org/trac/ticket/4208)

### mysql

+   **[mysql] [bug]**

    MySQL 方言现在显式地使用 `SELECT @@version` 查询服务器版本以确保我们得到正确的版本信息。代理服务器（如 MaxScale）会干扰传递给 DBAPI 的连接.server_version 值，因此此信息现在不再可靠。

    参考：[#4205](https://www.sqlalchemy.org/trac/ticket/4205)

## 1.1.17

发布日期：2018 年 2 月 22 日

+   **[bug] [ext]**

    修复了在 1.2.3 和 1.1.16 中关于关联代理对象的回归，修订了计算关联代理的“拥有类”的方法，使其在计算关联代理的“拥有类”时默认选择当前类，如果代理对象与映射类没有直接关联，例如与混入类相似的情况。

    参考：[#4185](https://www.sqlalchemy.org/trac/ticket/4185)

## 1.1.16

发布日期：2018 年 2 月 16 日

### orm

+   **[orm] [bug]**

    修复了 post_update 功能中的问题，即当父对象已被删除但相关对象未被删除时，会发出 UPDATE。此问题已存在很长时间，但自 1.2 版本以来，为 post_update 断言行匹配，因此会引发错误。

    参考：[#4187](https://www.sqlalchemy.org/trac/ticket/4187)

+   **[orm] [bug]**

    由于修复问题 [#4116](https://www.sqlalchemy.org/trac/ticket/4116) 而引起的回归，影响到版本 1.2.2 以及 1.1.15，导致一些声明性混合/继承情况下的“拥有类”计算错误，以及如果关联代理从未映射的类中访问，则将关联代理错误地分配给了 `NoneType` 类。现在，“找到所有者”的逻辑已被一种深度搜索的方法取代，该方法通过搜索分配给类或子类的完整映射器层次结构来确定正确（我们希望如此）的匹配；如果未找到匹配项，则不会分配所有者。如果对未映射实例使用代理，则现在会引发异常。

    参考：[#4185](https://www.sqlalchemy.org/trac/ticket/4185)

+   **[orm] [bug]**

    修复了一个错误，即在回滚嵌套事务或子事务期间对已发生主键变异的对象进行删除时，会导致该对象未能正确从会话中移除，从而在使用会话时引发后续问题。

    参考：[#4151](https://www.sqlalchemy.org/trac/ticket/4151)

### sql

+   **[sql] [bug]**

    将`nullsfirst()`和`nullslast()`作为`sqlalchemy.`和`sqlalchemy.sql.`命名空间中的顶级导入添加。感谢 Lele Gaifax 的拉取请求。

+   **[sql] [bug]**

    修复了在`Insert.values()`中使用“multi-values”格式与`Column`对象作为键而不是字符串时会失败的 bug。感谢 Aubrey Stark-Toller 的拉取请求。

    参考：[#4162](https://www.sqlalchemy.org/trac/ticket/4162)

### postgresql

+   **[postgresql] [bug]**

    将“SSL SYSCALL error: Operation timed out”添加到触发 psycopg2 驱动程序“断开连接”场景的消息列表中。感谢 André Cruz 的拉取请求。

+   **[postgresql] [bug]**

    将“TRUNCATE”添加到 PostgreSQL 方言接受的关键字列表中，作为“autocommit”触发关键字。感谢 Jacob Hayes 的拉取请求。

### mysql

+   **[mysql] [bug]**

    修复了 MySQL 的“concat”和“match”运算符无法将 kwargs 传播到左右表达式，导致编译器选项（如“literal_binds”）失败的错误。

    参考：[#4136](https://www.sqlalchemy.org/trac/ticket/4136)

### misc

+   **[bug] [pool]**

    修复了一个相当严重的连接池 bug，即在用户定义的`DisconnectionError`或由于 1.2 版本发布的“pre_ping”功能导致刷新后获取的连接，如果连接由 weakref 清理（例如前端对象被垃圾回收）返回到池中，则不会正确重置连接；弱引用仍将指向先前失效的 DBAPI 连接，而该连接将错误地调用重置操作。这将导致日志中的堆栈跟踪和连接被检入池而未被重置，这可能导致锁定问题。

    参考：[#4184](https://www.sqlalchemy.org/trac/ticket/4184)

## 1.1.15

发布日期：2017 年 11 月 3 日

### orm

+   **[orm] [bug] [ext]**

    修复了当关联代理首先以`AliasedClass`作为父类调用时，会错误地将自身链接到`AliasedClass`对象的错误，导致后续使用时出现错误的问题。

    参考：[#4116](https://www.sqlalchemy.org/trac/ticket/4116)

+   **[orm] [bug]**

    修复了一个问题，在继承层次结构中的兄弟类中，ORM 关系会警告存在冲突的同步目标（例如，两个关系都将写入同一列），但实际上这两个关系在写入时永远不会发生冲突。

    参考：[#4078](https://www.sqlalchemy.org/trac/ticket/4078)

+   **[orm] [bug]**

    修复了一个问题，其中针对单表继承实体使用的相关选择在外部查询中无法正确呈现，因为不适当地调整了单继承鉴别器条件，错误地将条件重新应用到外部查询中。

    参考：[#4103](https://www.sqlalchemy.org/trac/ticket/4103)

### orm 声明式

+   **[orm] [声明式] [bug]**

    修复了一个问题，在刷新操作期间会引用描述符（即基于`AbstractConcreteBase`的其他地方的映射列或关系），导致错误，因为该属性未映射为映射器属性。如果类未在其映射器中包含“concrete=True”，那么其他属性（例如由`AbstractConcreteBase`添加的“type”列）可能会出现类似的问题，但此处的检查也应该防止该场景引发问题。

    参考：[#4124](https://www.sqlalchemy.org/trac/ticket/4124)

### sql

+   **[sql] [bug]**

    修复了一个问题，其中`ColumnDefault`的`__repr__`如果参数是元组，则会失败。 感谢 Nicolas Caniart 的拉取请求。

    参考：[#4126](https://www.sqlalchemy.org/trac/ticket/4126)

+   **[sql] [bug]**

    修复了一个问题，其中最近添加的`ColumnOperators.any_()`和`ColumnOperators.all_()`方法在作为方法调用时无法正常工作，而不是使用独立的函数`any_()`和`all_()`。此外，为这些相对不直观的 SQL 操作添加了文档示例。

    参考：[#4093](https://www.sqlalchemy.org/trac/ticket/4093)

### postgresql

+   **[postgresql] [bug]**

    进一步修复了与 COLLATE 结合使用的`ARRAY`类的问题，因为在[#4006](https://www.sqlalchemy.org/trac/ticket/4006)中进行的修复未能适应多维数组。

    参考：[#4006](https://www.sqlalchemy.org/trac/ticket/4006)

+   **[postgresql] [bug]**

    修复了`array_agg`函数中的错误，其中传递一个已经是`ARRAY`类型的参数，例如 PostgreSQL 中的`array`构造，会产生`ValueError`，因为函数尝试嵌套数组。

    参考：[#4107](https://www.sqlalchemy.org/trac/ticket/4107)

+   **[postgresql] [bug]**

    修复了 PostgreSQL 中`Insert.on_conflict_do_update()`中的错误，该错误会阻止插入语句被用作 CTE，例如通过`Insert.cte()`在另一个语句中使用。

    参考：[#4074](https://www.sqlalchemy.org/trac/ticket/4074)

### mysql

+   **[mysql] [bug]**

    当检测到 MariaDB 10.2.8 或更早版本的 10.2 系列时，会发出警告，因为这些版本中的 CHECK 约束存在重大问题，这些问题在 10.2.9 中已解决。

    请注意，此更改日志消息未随 SQLAlchemy 1.2.0b3 一起发布，而是事后添加的。

    参考：[#4097](https://www.sqlalchemy.org/trac/ticket/4097)

+   **[mysql] [bug]**

    MySQL 5.7.20 现在会警告使用@tx_isolation 变量；现在执行版本检查并使用@transaction_isolation 来防止此警告。

    参考：[#4120](https://www.sqlalchemy.org/trac/ticket/4120)

+   **[mysql] [bug]**

    修复了在 MariaDB 10.2 系列中 CURRENT_TIMESTAMP 由于语法更改而无法正确反映的问题，其中该函数现在表示为`current_timestamp()`。

    参考：[#4096](https://www.sqlalchemy.org/trac/ticket/4096)

+   **[mysql] [bug]**

    MariaDB 10.2 现在支持 CHECK 约束（警告：由于在[#4097](https://www.sqlalchemy.org/trac/ticket/4097)中指出的上游问题，请使用 10.2.9 或更高版本）。反射现在在存在 CHECK 约束时考虑这些 CHECK 约束，当它们出现在`SHOW CREATE TABLE`输出中时。

    参考：[#4098](https://www.sqlalchemy.org/trac/ticket/4098)

### sqlite

+   **[sqlite] [bug]**

    修复了 SQLite 中 CHECK 约束反射失败的错误，如果引用的表在远程模式下，例如在 SQLite 中由 ATTACH 引用的远程数据库。

    参考：[#4099](https://www.sqlalchemy.org/trac/ticket/4099)

### mssql

+   **[mssql] [bug]**

    为 SQL Server 的 PyODBC 方言添加了完整范围的“连接关闭”异常代码，包括‘08S01’、‘01002’、‘08003’、‘08007’、‘08S02’、‘08001’、‘HYT00’、‘HY010’。以前只覆盖了‘08S01’。

    参考：[#4095](https://www.sqlalchemy.org/trac/ticket/4095)

## 1.1.14

发布日期：2017 年 9 月 5 日

### orm

+   **[orm] [bug]**

    修复了`Session.merge()`中的 bug，与[#4030](https://www.sqlalchemy.org/trac/ticket/4030)类似，其中对于身份映射中的目标对象的内部检查，如果在合并过程实际检索对象之前立即被垃圾回收，可能会导致错误。

    参考：[#4069](https://www.sqlalchemy.org/trac/ticket/4069)

+   **[orm] [bug]**

    修复了一个 bug，当`undefer_group()`选项扩展自使用连接式急加载加载的关系时，该选项将不被识别。此外，由于该 bug 导致执行了过多的工作，Python 函数调用次数在结果集列的初始计算中也提高了 20%，这与[#3915](https://www.sqlalchemy.org/trac/ticket/3915)的连接急加载改进相辅相成。

    参考：[#4048](https://www.sqlalchemy.org/trac/ticket/4048)

+   **[orm] [bug]**

    修复了 ORM 身份映射中的竞争条件，导致在加载操作期间不适当地删除对象，从而导致重复对象标识的发生，特别是在涉及对象去重的连接急加载下。该问题特定于弱引用的垃圾回收，并且仅在 PyPy 解释器下观察到。

    参考：[#4068](https://www.sqlalchemy.org/trac/ticket/4068)

+   **[orm] [bug]**

    修复了`Session.merge()`中的 bug，其中集合中的对象的主键属性设置为`None`，对于通常是自动递增的键，会被视为内部去重过程中的数据库持久化键的一部分，导致实际上只有一个对象被插入到数据库中。

    参考：[#4056](https://www.sqlalchemy.org/trac/ticket/4056)

+   **[orm] [bug]**

    当对不是`MapperProperty`的属性（如关联代理）使用`synonym()`时，会引发`InvalidRequestError`。以前，尝试定位不存在的属性会导致递归溢出。

    参考：[#4067](https://www.sqlalchemy.org/trac/ticket/4067)

### sql

+   **[sql] [bug]**

    修改了窗口函数的范围规范，允许在范围中使用两个相同的 PRECEDING 或 FOLLOWING 关键字，通过允许范围的左侧为正数，右侧为负数，例如(1, 3)是“1 FOLLOWING AND 3 FOLLOWING”。

    参考：[#4053](https://www.sqlalchemy.org/trac/ticket/4053)

## 1.1.13

发布日期：2017 年 8 月 3 日

### oracle

+   **[oracle] [性能] [bug] [py2k]**

    由于修复[#3937](https://www.sqlalchemy.org/trac/ticket/3937)导致的性能回归，其中 cx_Oracle 自版本 5.3 起从其命名空间中删除了`.UNICODE`符号，这被解释为 cx_Oracle 的“WITH_UNICODE”模式被无条件打开，这会在 SQLAlchemy 端调用函数，无条件地将所有字符串转换为 unicode 并导致性能影响。实际上，根据 cx_Oracle 的作者，自 5.1 起，“WITH_UNICODE”模式已完全删除，因此不再需要昂贵的 unicode 转换函数，如果在 Python 2 下检测到 cx_Oracle 5.1 或更高版本，则会禁用它们。还恢复了在[#3937](https://www.sqlalchemy.org/trac/ticket/3937)中删除的针对“WITH_UNICODE”模式的警告。

    此更改也**回溯**到：1.0.19

    参考：[#4035](https://www.sqlalchemy.org/trac/ticket/4035)

## 1.1.12

发布日期：2017 年 7 月 24 日

### orm

+   **[orm] [bug]**

    修复了从 1.1.11 开始的回归，其中向包含具有子查询加载关系的实体的查询添加额外的非实体列会失败，原因是在 1.1.11 中添加的检查作为[#4011](https://www.sqlalchemy.org/trac/ticket/4011)的结果。

    参考：[#4033](https://www.sqlalchemy.org/trac/ticket/4033)

+   **[orm] [bug]**

    修复了在 1.1 中添加的涉及 JSON NULL 评估逻辑的错误，作为[#3514](https://www.sqlalchemy.org/trac/ticket/3514)的一部分，其中逻辑不会适应与映射的`Column`不同命名的 ORM 映射属性。

    参考：[#4031](https://www.sqlalchemy.org/trac/ticket/4031)

+   **[orm] [bug]**

    在`WeakInstanceDict`的所有方法中添加了`KeyError`检查，其中在检查`key in dict`之后立即对该键进行索引访问，以防止在负载下垃圾回收可能会将键从字典中移除，导致代码假定其存在后非常少见地引发`KeyError`。

    参考：[#4030](https://www.sqlalchemy.org/trac/ticket/4030)

### oracle

+   **[oracle] [功能] [postgresql]**

    向`Sequence`添加了新的关键字`Sequence.cache`和`Sequence.order`，以允许渲染 Oracle 和 PostgreSQL 理解的 CACHE 参数，以及 Oracle 理解的 ORDER 参数。感谢 David Moore 的拉取请求。

### 测试

+   **[测试] [bug] [py3k]**

    修复了与 Python 3.6.2 变更不兼容的测试固定装置中的问题。

    此更改也**回溯**到：1.0.18

    参考：[#4034](https://www.sqlalchemy.org/trac/ticket/4034)

## 1.1.11

发布日期：2017 年 6 月 19 日

### orm

+   **[orm] [bug]**

    修复了子查询预加载的问题，该问题是从[#2699](https://www.sqlalchemy.org/trac/ticket/2699)、[#3106](https://www.sqlalchemy.org/trac/ticket/3106)、[#3893](https://www.sqlalchemy.org/trac/ticket/3893)修复的一系列问题中延续而来，涉及到“子查询”在从连接的继承子类开始，然后对基类的关系进行子查询预加载时包含正确的 FROM 子句，同时查询还包括对子类的条件。之前票据中的修复未考虑到从第一级更深层次加载更多的 subqueryload 操作，因此修复已进一步泛化。

    参考：[#4011](https://www.sqlalchemy.org/trac/ticket/4011)

### sql

+   **[sql] [bug]**

    修复了在`WithinGroup`结构迭代期间可能发生的 AttributeError。

    参考：[#4012](https://www.sqlalchemy.org/trac/ticket/4012)

### postgresql

+   **[postgresql] [bug]**

    继续修复 1.1.8 中发布的正确处理 PostgreSQL 版本字符串“10devel”的问题，另外增加了一个正则表达式升级以处理形式为“10beta1”的版本字符串。虽然现在 PostgreSQL 提供了更好的获取此信息的方法，但至少在 1.1.x 版本中我们将继续使用正则表达式，以减少与旧版或替代 PostgreSQL 数据库兼容性的风险。

    参考：[#4005](https://www.sqlalchemy.org/trac/ticket/4005)

+   **[postgresql] [bug]**

    修复了在使用带有排序规则的字符串类型的`ARRAY`时，在 CREATE TABLE 中未能生成正确语法的错误。

    参考：[#4006](https://www.sqlalchemy.org/trac/ticket/4006)

### mysql

+   **[mysql] [bug]**

    MySQL 5.7 为“SHOW VARIABLES”命令引入了权限限制；MySQL 方言现在将处理当 SHOW 返回零行时的情况，特别是对于 SQL_MODE 的初始获取，并将发出警告，提示用户权限应该修改以允许该行存在。

    参考：[#4007](https://www.sqlalchemy.org/trac/ticket/4007)

### mssql

+   **[mssql] [bug]**

    修复了在使用 Azure 数据仓库时必须从不同视图获取 SQL Server 事务隔离的 bug，现在查询将尝试针对两个视图执行，如果持续失败，则无条件引发 NotImplemented，以提供对未来新 SQL Server 版本中任意 API 更改的最佳弹性。

    参考：[#3994](https://www.sqlalchemy.org/trac/ticket/3994)

+   **[mssql] [bug]**

    为 SQL Server 方言添加了一个占位符类型`XML`，以便包含此类型的反射表可以重新呈现为 CREATE TABLE。该类型没有特殊的往返行为，也不支持额外的限定参数。

    参考：[#3973](https://www.sqlalchemy.org/trac/ticket/3973)

### oracle

+   **[oracle] [bug]**

    当使用 cx_Oracle 的 6.0b1 或更高版本的 DBAPI 时，完全移除了对两阶段事务的支持。在任何情况下，两阶段特性在 cx_Oracle 5.x 下历史上从未可用，而 cx_Oracle 6.x 已经移除了这个特性依赖的连接级“twophase”标志。

    参考：[#3997](https://www.sqlalchemy.org/trac/ticket/3997)

## 1.1.10

发布日期：2017 年 5 月 19 日 星期五

### orm

+   **[orm] [bug]**

    修复了级联操作（如“delete-orphan”等）无法定位与继承关系中本身是子类的关系链接的对象的 bug，从而导致操作无法执行。

    参考：[#3986](https://www.sqlalchemy.org/trac/ticket/3986)

### schema

+   **[schema] [bug]**

    如果创建一个`ForeignKeyConstraint`对象时“local”和“remote”列的数量不匹配，现在会引发一个`ArgumentError`，否则会导致约束的内部状态不正确。请注意，这也会影响方言的反射过程产生的外键约束列集不匹配的情况。

    参考：[#3949](https://www.sqlalchemy.org/trac/ticket/3949)

### postgresql

+   **[postgresql] [bug]**

    为 GRANT、REVOKE 关键字添加了“autocommit”支持。感谢 Jacob Hayes 的 Pull 请求。

### mysql

+   **[mysql] [bug]**

    移除了对 UTC_TIMESTAMP MySQL 函数的古老且不必要的拦截，这个拦截妨碍了使用带参数的函数。

    参考：[#3966](https://www.sqlalchemy.org/trac/ticket/3966)

+   **[mysql] [bug]**

    修复了 MySQL 方言中有关在渲染 CREATE TABLE 时与 PARTITION 选项一起渲染表选项的 bug。PARTITION 相关选项需要跟随表选项，而以前这种顺序没有被强制执行。

    参考：[#3961](https://www.sqlalchemy.org/trac/ticket/3961)

### oracle

+   **[oracle] [bug]**

    修复了 cx_Oracle 方言中的 bug，其中由于“b”字符，对 cx_Oracle 版本 6.0b1 的版本字符串解析会失败。现在版本字符串解析通过正则表达式而不是简单的分割。

    参考：[#3975](https://www.sqlalchemy.org/trac/ticket/3975)

### misc

+   **[bug] [ext]**

    在声明类被垃圾回收并且新的 automap prepare()操作同时进行的情况下，防止测试“None”作为一个类，非常罕见地在 gc 后未完全处理的 weakref 上进行操作。

    参考：[#3980](https://www.sqlalchemy.org/trac/ticket/3980)

## 1.1.9

发布日期：2017 年 4 月 4 日

### sql

+   **[sql] [bug]**

    修复了在 1.1.5 版本中由于 [#3859](https://www.sqlalchemy.org/trac/ticket/3859) 导致的回归，根据 `Variant` 的“右侧”规则调整表达式的“右侧”评估，以遵守底层类型的“右侧”规则，导致在我们确实希望左侧类型直接传递到右侧，以便将绑定级规则应用于表达式参数时，`Variant` 类型不适当地丢失。

    参考：[#3952](https://www.sqlalchemy.org/trac/ticket/3952)

+   **[SQL] [错误] [PostgreSQL]**

    更改了`ResultProxy`的机制，无条件延迟“自动关闭”步骤，直到`Connection`完成对象的操作；在 PostgreSQL ON CONFLICT with RETURNING 返回零行的情况下，自动关闭会发生在这种以前不存在的用例中，导致以前在 INSERT/UPDATE/DELETE 上无条件发生的通常自动提交行为失败。

    参考：[#3955](https://www.sqlalchemy.org/trac/ticket/3955)

### 杂项

+   **[错误] [扩展]**

    修复了在 1.1.8 版本中由于 [#3950](https://www.sqlalchemy.org/trac/ticket/3950) 导致的回归，当映射还包含 `column_property` 时，在“模式类型”或 `TypeDecorator` 的情况下，对列类型的更深层搜索会导致属性错误。

    参考：[#3956](https://www.sqlalchemy.org/trac/ticket/3956)

## 1.1.8

发布日期：2017 年 3 月 31 日

### PostgreSQL

+   **[PostgreSQL] [错误]**

    增加了对解析 PostgreSQL 版本字符串的支持，例如“PostgreSQL 10devel”。感谢 Sean McCully 的拉取请求。

### 杂项

+   **[错误] [扩展]**

    修复了在`sqlalchemy.ext.mutable`中的 bug，在使用`TypeEngine.copy()`复制了类型后，`Mutable.as_mutable()`方法将不会跟踪该类型。这在 1.1 中与 1.0 相比变得更加退化，因为`TypeDecorator`类现在是`SchemaEventTarget`的子类，其中一项指示父`Column`应在复制`Column`时复制类型。在使用混合或抽象类的情况下，这些副本是常见的。

    参考：[#3950](https://www.sqlalchemy.org/trac/ticket/3950)

+   **[bug] [ext]**

    对`Result.count()`方法添加了对绑定参数的支持，例如通常通过`Query.params()`设置的参数。之前，对参数的支持被省略了。感谢 Pat Deegan 的拉取请求。

## 1.1.7

发布日期：2017 年 3 月 27 日

### orm

+   **[orm] [feature]**

    现在可以将`aliased()`构造传递给`Query.select_entity_from()`方法。实体将从由`aliased()`构造表示的可选择项中提取。这允许在与`Query.select_entity_from()`一起使用时使用`aliased()`的特殊选项，例如`aliased.adapt_on_names`。

    参考：[#3933](https://www.sqlalchemy.org/trac/ticket/3933)

+   **[orm] [bug]**

    修复了在多线程环境下可能发生的竞态条件，这是由于通过[#3915](https://www.sqlalchemy.org/trac/ticket/3915)添加的缓存引起的。内部`Column`对象的集合可能会不适当地在别名对象上重新生成，当尝试渲染 SQL 并收集结果时，会让连接的急加载器困惑，并导致属性错误。现在，在别名对象被缓存并在线程之间共享之前，该集合会提前生成。

    参考：[#3947](https://www.sqlalchemy.org/trac/ticket/3947)

### engine

+   **[engine] [bug]**

    添加了一个异常处理程序，当`Connection`的“autorollback”功能本身引发异常时，将会警告“cause”异常在 Py2K 上。在 Py3K 中，这两个异常自然由解释器报告为一个在处理另一个时发生。这是继续处理回滚失败处理的一系列更改的一部分，上次在 1.0.12 中作为[#2696](https://www.sqlalchemy.org/trac/ticket/2696)的一部分讨论过。

    参考：[#3946](https://www.sqlalchemy.org/trac/ticket/3946)

### sql

+   **[sql] [bug] [postgresql]**

    增加了对`Variant`和`SchemaType`对象相互兼容的支持。也就是说，可以针对像`Enum`这样的类型创建一个变体，并且创建约束和/或数据库特定类型对象的指令将根据变体的方言映射正确传播。

    参考：[#2892](https://www.sqlalchemy.org/trac/ticket/2892)

+   **[sql] [bug]**

    修复了编译器中的一个 bug，其中保存点的字符串标识符会被缓存在标识符引用字典中；由于这些标识符是任意的，如果一个`Connection`使用了无限数量的保存点，或者直接使用了无限数量的保存点子句构造，可能会发生小内存泄漏。这个内存泄漏**不会**影响绝大多数情况，因为通常会在每个事务或固定数量的事务基础上使用一个简单计数器从“1”开始渲染保存点名称的`Connection`在被丢弃之前使用。

    参考：[#3931](https://www.sqlalchemy.org/trac/ticket/3931)

+   **[sql] [bug]**

    修复了新“模式翻译”功能中的 bug，其中在与列表达式一起呈现时，翻译的模式名称将被调用为别名名称；仅当源翻译名称为“None”时才会发生。现在，“模式翻译”功能仅对`SchemaItem`和`SchemaType`子类生效，即对应于数据库中可创建 DDL 结构的对象。

    参考：[#3924](https://www.sqlalchemy.org/trac/ticket/3924)

### oracle

+   **[oracle] [bug]**

    修复了 cx_Oracle 的 WITH_UNICODE 模式，这是由于 cx_Oracle 5.3 现在似乎在构建中硬编码了此标志；使用此模式的内部方法未使用正确的签名。

    此更改也被**回溯**到：1.0.18

    参考：[#3937](https://www.sqlalchemy.org/trac/ticket/3937)

## 1.1.6

发布日期：2017 年 2 月 28 日

### orm

+   **[orm] [bug]**

    解决了自从早期版本以来积累的一些长期未解决的性能问题，这是由于增加的抽象而导致的连接式贪婪加载查询构造系统。每次查询使用临时`AliasedClass`对象进行列查找开销较大，现已更换为使用缓存方法，利用一小组在连接式贪婪加载调用之间重复使用的`AliasedClass`对象。还对涉及贪婪连接路径构造的某些机制进行了优化。最坏情况下的连接式加载场景的端到端查询构造+单行提取测试的调用次数与 1.1.5 相比减少了约 60%，与 0.8.6 相比减少了 42%。

    参考：[#3915](https://www.sqlalchemy.org/trac/ticket/3915)

+   **[orm] [bug]**

    修复了“eager_defaults”功能中的一个主要低效性，即当 ORM 明确插入 NULL 时，会为列值发出不必要的 SELECT，对应于对象上未设置但没有指定任何服务器默认值的属性，以及在更新时过期的属性，尽管没有设置服务器 onupdate。由于这些列不是 eager_defaults 尝试使用的 RETURNING 的一部分，因此也不应该被后置 SELECT。

    参考：[#3909](https://www.sqlalchemy.org/trac/ticket/3909)

+   **[orm] [bug]**

    修复了两个与映射器 eager_defaults 标志密切相关的错误，与单表继承结合使用；一个是 eager defaults 逻辑会在 eager defaults 获取期间无意中尝试访问映射器的“exclude_properties”列表中的列（由具有单表继承的 Declarative 使用），另一个是为了获取默认值而对行进行完整加载时，会失败地使用不正确的继承映射器。

    参考：[#3908](https://www.sqlalchemy.org/trac/ticket/3908)

+   **[orm] [bug]**

    修复了在 0.9.7 中首次引入的错误，由于[#3106](https://www.sqlalchemy.org/trac/ticket/3106)导致某些形式的多级子查询加载对别名实体产生错误查询，在最内层子查询中存在一个不必要的额外 FROM 实体。

    参考：[#3893](https://www.sqlalchemy.org/trac/ticket/3893)

### orm 声明式

+   **[orm] [declarative] [bug]**

    修复了一个 bug，即 declarative 的“自动排除”功能确保单个表继承子类的列不会出现在基类的其他派生类属性中，对于从基类多级子类化不会生效。

    参考：[#3895](https://www.sqlalchemy.org/trac/ticket/3895)

### sql

+   **[sql] [bug]**

    修复了一个 bug，即`DDLEvents.column_reflect()`事件不允许将非文本表达式作为新列的“default”值传递，例如`FetchedValue`对象表示通用触发默认值或`text()`构造。同时在文档中澄清了这一点。

    参考：[#3905](https://www.sqlalchemy.org/trac/ticket/3905)

### postgresql

+   **[postgresql] [bug]**

    为“IMPORT FOREIGN SCHEMA”、“REFRESH MATERIALIZED VIEW” PostgreSQL 语句添加了正则表达式，以便在通过连接或引擎调用时自动提交，而不需要显式事务。感谢 Frazer McLean 和 Paweł Stiasny 的拉取请求。

    参考：[#3804](https://www.sqlalchemy.org/trac/ticket/3804)

+   **[postgresql] [bug]**

    修复了 PostgreSQL `ExcludeConstraint`中的 bug，其中“whereclause”和“using”参数在像`Table.tometadata()`这样的操作期间不会被复制。

    参考：[#3900](https://www.sqlalchemy.org/trac/ticket/3900)

### mysql

+   **[mysql] [bug]**

    为了正确引用，将新的 MySQL 8.0 保留字添加到 MySQL 方言中。感谢 Hanno Schlichting 的拉取请求。

### mssql

+   **[mssql] [bug]**

    在“get_isolation_level”功能中添加了版本检查，该功能在首次连接时调用，因此对于 SQL Server 版本 2000，它会跳过，因为在 SQL Server 2005 之前不可用。

    参考：[#3898](https://www.sqlalchemy.org/trac/ticket/3898)

### misc

+   **[feature] [ext]**

    添加了`Result.scalar()`和`Result.count()`到“baked”查询系统。

    参考：[#3896](https://www.sqlalchemy.org/trac/ticket/3896)

+   **[bug] [ext]**

    修复了新的`sqlalchemy.ext.indexable`扩展中的 bug，其中设置一个属性，该属性本身引用另一个属性会失败。

    参考：[#3901](https://www.sqlalchemy.org/trac/ticket/3901)

## 1.1.5

发布日期：2017 年 1 月 17 日

### orm

+   **[orm] [bug]**

    修复了在多个实体上使用连接急加载时，当也使用多态继承时会抛出“'NoneType'对象没有'isa'属性”的错误。这个问题是由于修复[#3611](https://www.sqlalchemy.org/trac/ticket/3611)引入的。

    此更改也被**回溯**到：1.0.17

    参考：[#3884](https://www.sqlalchemy.org/trac/ticket/3884)

+   **[ORM] [错误]**

    修复了子查询加载中的错误，其中作为“现有”行遇到的对象，例如在同一查询中从不同路径加载的对象，不会调用指定此加载的未加载属性的子查询加载程序。这个问题与[#3431](https://www.sqlalchemy.org/trac/ticket/3431)、[#3811](https://www.sqlalchemy.org/trac/ticket/3811)中涉及的与连接加载类似的问题在同一领域。

    参考：[#3854](https://www.sqlalchemy.org/trac/ticket/3854)

+   **[ORM] [错误]**

    `Session.no_autoflush`上下文管理器现在确保在“finally”块中重置 autoflush 标志，以便如果在块内引发异常，则状态仍然适当地重置。感谢 Emin Arakelian 的拉取请求。

+   **[ORM] [错误]**

    修复了单表继承查询条件未插入查询中的错误，当`Bundle`构造用作选择条件时。

    参考：[#3874](https://www.sqlalchemy.org/trac/ticket/3874)

+   **[ORM] [错误]**

    修复了与[#3177](https://www.sqlalchemy.org/trac/ticket/3177)相关的错误，即`Query`发出的 UNION 或其他集合操作会将“单继承”条件应用于联合的外部（还引用了错误的可选择项），即使这些条件现在应该已经存在于内部子查询中。一旦对`Query`调用 union()或另一个集合操作，单继承条件现在会被省略，就像`Query.from_self()`一样。

    参考：[#3856](https://www.sqlalchemy.org/trac/ticket/3856)

### 示例

+   **[示例] [错误]**

    修复了版本历史示例中的两个问题，一个是历史表现在 autoincrement=False 以避免 1.1 版本中关于自动增量的新错误；另一个是现在使用 sqlite_autoincrement 标志以确保在 SQLite 上，即使删除了某些行，也会为表的生命周期使用唯一标识符。感谢 Carlos García Montoro 的拉取请求。

    参考：[#3872](https://www.sqlalchemy.org/trac/ticket/3872)

### 引擎

+   **[引擎] [错误]**

    `Table`反射的“extend_existing”选项会导致索引和约束在使用该参数与`MetaData.reflect()`（如 automap 扩展所做的）时重复，因为表既在外键路径中反射，也直接反射。在`MetaData.reflect()`序列中传递一个新的去重集合，以防止以这种方式重复反射。

    参考：[#3861](https://www.sqlalchemy.org/trac/ticket/3861)

### sql

+   **[sql] [bug]**

    修复了最初在 0.9 版本中引入的 bug，通过[#1068](https://www.sqlalchemy.org/trac/ticket/1068)，其中`order_by(<some Label()>)`将根据名称单独排序，即使标记的表达式与可选择中隐式或显式存在的表达式完全不同。现在，按标签排序的逻辑确保标记的表达式与解析为该名称的表达式相关联，然后再按标签名称排序；此外，名称必须在表达式中的其他位置明确解析为实际标签，而不仅仅是列名。这种逻辑与具有稍微不同目的的按文本名称排序功能明确分开。

    参考：[#3882](https://www.sqlalchemy.org/trac/ticket/3882)

+   **[sql] [bug]**

    修复了 1.1 版本中`import *`无法在 sqlalchemy.sql.expression 中工作的回归，因为`any_`和`all_`函数拼写错误。

    参考：[#3878](https://www.sqlalchemy.org/trac/ticket/3878)

+   **[sql] [bug]**

    在`MetaData.reflect()`中“无法反射”异常中嵌入的引擎 URL 现在隐藏了密码；此外，`TLEngine`的`__repr__`现在的行为类似于`Engine`，隐藏了 URL 密码。感谢 Valery Yundin 的拉取请求。

+   **[sql] [bug]**

    修复了 `Variant` 中的问题，其中“右手边强制转换”逻辑继承自 `TypeDecorator`，会将右侧强制转换为 `Variant` 本身，而不是 `Variant` 的默认类型。在 `Variant` 的情况下，我们希望类型大部分像基本类型一样工作，因此现在覆盖了 `TypeDecorator` 的默认逻辑，以回退到基础包装类型的逻辑。目前主要与 JSON 有关。

    参考文献：[#3859](https://www.sqlalchemy.org/trac/ticket/3859)

+   **[sql] [bg]**

    修复了当 `literal_binds` 编译器标志未被 `Insert` 构造的“多值”功能所尊重时的 bug；随后的值现在被渲染为字面值。

    参考文献：[#3880](https://www.sqlalchemy.org/trac/ticket/3880)

### postgresql

+   **[postgresql] [bug]**

    修复了新的“ON CONFLICT DO UPDATE”功能中的 bug，其中 UPDATE 子句的“set”值不会受到类型级别的处理，通常会生效以处理用户定义的类型级别转换以及方言所需的转换，例如 JSON 数据类型所需的转换。此外，澄清了 `set_` 字典中的键应与列的“键”匹配，如果与列名不同。对于不匹配列键的剩余列名发出警告；出于兼容性原因，这些警告与以前一样发出。

    参考文献：[#3888](https://www.sqlalchemy.org/trac/ticket/3888)

+   **[postgresql] [bug]**

    现在 `TIME` 和 `TIMESTAMP` 数据类型支持“精度”设置为零；之前零值会被忽略。感谢 Ionuț Ciocîrlan 的拉取请求。

### mysql

+   **[mysql] [feature]**

    添加了一个新的参数 `mysql_prefix`，由 `Index` 构造支持，允许指定 MySQL 特定的前缀，如 “FULLTEXT”。感谢 Joseph Schorr 的拉取请求。

+   **[mysql] [bug]**

    MySQL 方言现在不会在反射的列上有“COMMENT”关键字时发出警告，但请注意评论尚未反射；这在未来的发布计划中。感谢 Lele Long 的拉取请求。

    参考：[#3867](https://www.sqlalchemy.org/trac/ticket/3867)

### mssql

+   **[mssql] [bug]**

    修复了 SQL Server 方言尝试为 INSERT from SELECT 选择最后一行标识的 bug，在 SELECT 没有行的情况下会失败。对于这样的语句，内联标志设置为 True，表示不应获取最后一个主键。

    参考：[#3876](https://www.sqlalchemy.org/trac/ticket/3876)

### oracle

+   **[oracle] [bug] [postgresql]**

    修复了在源表包含自动递增序列的情况下，从 SELECT 进行插入时编译失败的 bug。

    参考：[#3877](https://www.sqlalchemy.org/trac/ticket/3877)

+   **[oracle] [bug]**

    修复了在 Oracle 9.2 上 ALL_TABLES 查询中使用“COMPRESSION”关键字的 bug；尽管 Oracle 文档说明表压缩是在 9i 中引入的，但实际列直到 10.1 才存在。

    参考：[#3875](https://www.sqlalchemy.org/trac/ticket/3875)

### misc

+   **[bug] [py3k]**

    修复了与没有 ‘r’ 修饰符的转义字符串相关的 Python 3.6 DeprecationWarnings，并为 Python 3.6 添加了测试覆盖。

    此更改也 **回溯** 至：1.0.17

    参考：[#3886](https://www.sqlalchemy.org/trac/ticket/3886)

+   **[bug] [firebird]**

    将对 Oracle 引号小写名称的修复移植到 Firebird，以便正确反映作为小写的表名，即使表名来自 get_table_names() 检查函数。

    参考：[#3548](https://www.sqlalchemy.org/trac/ticket/3548)

## 1.1.4

发布日期：2016 年 11 月 15 日

### orm

+   **[orm] [bug]**

    修复了 `Session.bulk_update_mappings()` 中的一个 bug，即备用命名的主键属性在更新语句中无法正确跟踪。

    此更改也 **回溯** 至：1.0.16

    参考：[#3849](https://www.sqlalchemy.org/trac/ticket/3849)

+   **[orm] [bug]**

    修复了 `Session.bulk_save()` 中的一个 bug，在实现版本 id 计数器的映射与 UPDATE 结合使用时无法正确运行。

    此更改也 **回溯** 至：1.0.16

    参考：[#3781](https://www.sqlalchemy.org/trac/ticket/3781)

+   **[orm] [bug]**

    修复了当首次调用 mapper 属性或其他 ORM 构造添加到映射器/类之后，`Mapper.attrs`、`Mapper.all_orm_descriptors` 和其他派生属性将无法刷新的 bug。

    此更改也 **回溯** 至：1.0.16

    参考：[#3778](https://www.sqlalchemy.org/trac/ticket/3778)

+   **[orm] [bug]**

    由于[#3457](https://www.sqlalchemy.org/trac/ticket/3457)中的反序列化在 pickle 或 deepcopy 期间失败，导致 ORM 集合的所有属性未能建立，进而导致进一步的变异操作失败，因此修复了集合中的回归。

    参考：[#3852](https://www.sqlalchemy.org/trac/ticket/3852)

+   **[orm] [bug]**

    修复了长期存在的 bug，即“noload”关系加载策略会导致忽略 backrefs 和/或 back_populates 选项。

    参考：[#3845](https://www.sqlalchemy.org/trac/ticket/3845)

### engine

+   **[engine] [bug]**

    从`Connection`中删除了长期存在的“default_schema_name()”方法。这个方法是从一个非常旧的版本遗留下来的，是不起作用的（例如会引发异常）。感谢 Benjamin Dopplinger 的拉取请求。

### sql

+   **[sql] [bug]**

    修复了一个 bug，在没有设置自动增量的情况下插入主键时，新添加的警告会失败地发出，详情请见[#3216](https://www.sqlalchemy.org/trac/ticket/3216)。

    参考：[#3842](https://www.sqlalchemy.org/trac/ticket/3842)

### postgresql

+   **[postgresql] [bug]**

    由于在[#3807](https://www.sqlalchemy.org/trac/ticket/3807)中修复的问题（版本 1.1.0），我们确保在 PostgreSQL 的 ON CONFLICT 的 DO UPDATE 部分的 WHERE 子句中限定了表名，但是实际上*不能*在 ON CONFLICT 本身的 WHERE 子句中放置表名。这是一个错误的假设，因此将回滚在[#3807](https://www.sqlalchemy.org/trac/ticket/3807)中的更改部分。

    参考：[#3807](https://www.sqlalchemy.org/trac/ticket/3807)，[#3846](https://www.sqlalchemy.org/trac/ticket/3846)

### mysql

+   **[mysql] [feature]**

    为 mysqlclient 和 pymysql 方言添加了对服务器端游标的支持。此功能可通过`Connection.execution_options.stream_results`标志以及相同方式中的`server_side_cursors=True`方言参数来使用，就像在 PostgreSQL 上的 psycopg2 一样。感谢 Roman Podoliaka 的拉��请求。

+   **[mysql] [bug]**

    MySQL 的原生 ENUM 类型支持发送任何非有效值，并且会返回一个空字符串。为了检查“是否返回空字符串”，在 ENUM 的 MySQL 实现中添加了一个硬编码规则，以便将此空字符串返回给应用程序，而不是将其拒绝为非有效值。请注意，如果您的 MySQL 枚举正在将值链接到对象，您仍将收到空字符串。

    参考：[#3841](https://www.sqlalchemy.org/trac/ticket/3841)

### sqlite

+   **[sqlite] [bug]**

    在 pysqlcipher 方言中为 PRAGMA 指令添加引号，以适当支持附加的密码参数。感谢 Kevin Jurczyk 的拉取请求。

+   **[sqlite] [bug] [py3k]**

    在使用 pysqlcipher 方言时，添加了对 pysqlcipher3 DBAPI 的可选导入。如果 Python-2 专用的 pysqlcipher DBAPI 不存在，则此软件包将尝试导入。感谢 Kevin Jurczyk 的拉取请求。

### mssql

+   **[mssql] [bug]**

    修复了 pyodbc 方言中的错误（以及大部分不起作用的 adodbapi 方言），其中密码或用户名字段中存在分号时可能被解释为另一个标记的分隔符；现在在存在分号时对值进行引用。

    此更改也**回溯**到：1.0.16

    参考：[#3762](https://www.sqlalchemy.org/trac/ticket/3762)

## 1.1.3

发布日期：2016 年 10 月 27 日

### orm

+   **[orm] [bug]**

    修复了由[#2677](https://www.sqlalchemy.org/trac/ticket/2677)引起的回归问题，即在对已在该会话中标记为已删除的对象调用`Session.delete()`时，会导致无法设置对象在标识映射中（或拒绝对象），从而导致刷新错误，因为对象处于工作单元不适应的状态。在这种情况下，恢复了 1.1 版本之前的行为，即将对象放回标识映射中，以便再次尝试 DELETE 语句，这会发出警告，指出未匹配预期行数（除非在会话外恢复了行）。

    参考：[#3839](https://www.sqlalchemy.org/trac/ticket/3839)

+   **[orm] [bug]**

    修复了一些`Query`方法（如`Query.update()`等）在针对一系列映射列而不是整个映射实体时会失败的回归问题。

    参考：[#3836](https://www.sqlalchemy.org/trac/ticket/3836)

### sql

+   **[sql] [bug]**

    修复了在`Enum`中使用新值翻译和验证功能时的错误，其中在字符串连接中使用枚举对象会将整个表达式的类型保持为`Enum`类型，导致查找缺失。现在，针对`Enum`类型的列进行字符串连接将使用`String`作为表达式本身的数据类型。

    参考：[#3833](https://www.sqlalchemy.org/trac/ticket/3833)

+   **[sql] [bug]**

    修复了一个回归问题，该问题是[#2919](https://www.sqlalchemy.org/trac/ticket/2919)的副作用，对于一个不太典型的情况，即用户定义的`TypeDecorator`同时也是`SchemaType`的实例（而不是实现是这样的），会导致列附加事件被跳过。

    参考：[#3832](https://www.sqlalchemy.org/trac/ticket/3832)

### postgresql

+   **[postgresql] [bug]**

    PostgreSQL 表反射将确保在反射不是`Integer`数据类型的主键列时，`Column.autoincrement`标志设置为 False，即使默认值与生成整数的序列相关。如果列被创建为 SERIAL 并且数据类型被更改，则 autoincrement 标志只能在 1.1 系列中的整数亲和性数据类型为 True 时才能为 True。

    参考：[#3835](https://www.sqlalchemy.org/trac/ticket/3835)

## 1.1.2

发布日期：2016 年 10 月 17 日

### orm

+   **[orm] [bug]**

    修复了一个 bug，涉及到在一个多对一懒加载器的另一侧禁用连接集合急加载器的规则，首次添加在[#1495](https://www.sqlalchemy.org/trac/ticket/1495)中，如果父对象有一些其他与其关联的懒加载器绑定的查询选项，该规则将失败。

    参考：[#3824](https://www.sqlalchemy.org/trac/ticket/3824)

+   **[orm] [bug]**

    以类似于[#3431](https://www.sqlalchemy.org/trac/ticket/3431)、[#3811](https://www.sqlalchemy.org/trac/ticket/3811)的方式修复了自引用实体、延迟列加载问题，其中一个实体由于自引用的急加载而在行中出现在多个位置；当延迟加载器仅适用于其中一个路径时，“存在”列加载器现在将覆盖该实体的延迟非加载，而不考虑行顺序。

    参考：[#3822](https://www.sqlalchemy.org/trac/ticket/3822)

### sql

+   **[sql] [bug]**

    修复了由于新增函数引起的回归问题，该函数执行 sql `DefaultGenerator`对象的“包装可调用”函数时，当默认可调用是`functools.partial`或其他没有`__module__`属性的对象时，会引发`__module__`属性错误。

    参考：[#3823](https://www.sqlalchemy.org/trac/ticket/3823)

+   **[sql] [bug] [postgresql]**

    修复了`Enum`类型中的回归，其中事件处理程序在复制类型对象的情况下未被传递，这是由于[#3250](https://www.sqlalchemy.org/trac/ticket/3250)中添加的冲突的 copy()方法。在复制列时，例如在 tometadata()中或在使用具有列的声明性 mixin 时，通常会发生此复制。事件处理程序的缺失会影响为非本地枚举类型创建的约束，但更为关键的是 PostgreSQL 后端的 ENUM 对象。

    参考：[#3827](https://www.sqlalchemy.org/trac/ticket/3827)

### postgresql

+   **[postgresql] [bug] [sql]**

    更改了生成多 VALUES 插入语句的绑定参数时使用的命名约定，以使编号参数名称不与现在在 PostgreSQL ON CONFLICT 构造中常见的 WHERE 子句的匿名参数发生冲突。

    参考：[#3828](https://www.sqlalchemy.org/trac/ticket/3828)

## 1.1.1

发布日期：2016 年 10 月 7 日

### mssql

+   **[mssql] [bug]**

    在[#3810](https://www.sqlalchemy.org/trac/ticket/3810)和[#3814](https://www.sqlalchemy.org/trac/ticket/3814)中添加的“SELECT SERVERPROPERTY”查询在未知的 Pyodbc 和 SQL Server 组合上失败。虽然预料到了此功能的失败，但异常捕获不够广泛，因此现在捕获所有形式的 pyodbc.Error。

    参考：[#3820](https://www.sqlalchemy.org/trac/ticket/3820)

### 杂项

+   **[bug] [core]**

    当检测到各种缺失主键情��时，将引发的 CompileError 更改为警告。语句再次传递到数据库，其中将失败并像往常一样引发 DBAPI 错误（通常是 IntegrityError）。

    另请参阅

    不再为复合主键列隐式启用.autoincrement 指令

    参考：[#3216](https://www.sqlalchemy.org/trac/ticket/3216)

## 1.1.0

发布日期：2016 年 10 月 5 日

### orm

+   **[orm] [feature]**

    增强了新的“raise”延迟加载策略，还包括一个“raise_on_sql”变体，可通过`relationship.lazy`和`raiseload()`使用。此变体仅在延迟加载实际发出 SQL 时才引发异常，而不是在调用延迟加载机制时引发异常。

    参考：[#3812](https://www.sqlalchemy.org/trac/ticket/3812)

+   **[orm] [feature]**

    如果传递了`None`参数，`Query.group_by()`方法现在会重置分组集合，就像`Query.order_by()`长期以来的工作方式一样。感谢 Iuri Diniz 的拉取请求。

+   **[orm] [change]**

    将 False 传递给`Query.order_by()`以取消所有排序已被弃用；现在调用此方法时传递 False 或 None 之间不再有任何区别。

+   **[orm] [bug]**

    修复了连接的急切加载对于多态加载的映射器会失败的错误，其中多态 _on 设置为未映射表达式，如 CASE 表达式。

    此更改也已**回溯**至：1.0.16

    参考：[#3800](https://www.sqlalchemy.org/trac/ticket/3800)

+   **[orm] [bug]**

    修复了当通过`Session.bind_mapper()`、`Session.bind_table()`或构造函数发送给 Session 的无效绑定时引发的 ArgumentError 无法正确引发的错误。

    此更改也已**回溯**至：1.0.16

    参考：[#3798](https://www.sqlalchemy.org/trac/ticket/3798)

+   **[orm] [bug]**

    修复了子查询急切加载中的错误，其中“of_type()”对象的子查询加载链接到第二个子查询加载的普通映射类，或者几个“of_type()”属性的更长链，将无法正确链接连接。

    此更改也已**回溯**至：1.0.15

    参考：[#3773](https://www.sqlalchemy.org/trac/ticket/3773), [#3774](https://www.sqlalchemy.org/trac/ticket/3774)

+   **[orm] [bug]**

    现在可以将 ORM 属性分配给具有`__clause_element__()`属性的任何对象，这将导致内联 SQL，就像任何`ClauseElement`类一样。这涵盖了其他映射属性，否则不会被进一步表达式构造转换。

    参考：[#3802](https://www.sqlalchemy.org/trac/ticket/3802)

+   **[orm] [bug]**

    对首次引入的[票号：3431]中的错误修复进行了调整，涉及到单个结果集中出现在多个上下文中的对象，因此会触发将相关对象值设置为 None 的急切加载器，从而满足该属性的加载。先前，该调整仅在次要行中急切加载属性的非 None 值到达时才受到尊重。

    参考：[#3811](https://www.sqlalchemy.org/trac/ticket/3811)

+   **[orm] [bug]**

    修复了新的`SessionEvents.persistent_to_deleted()`事件中的错误，其中目标对象可能在事件触发之前被垃圾回收。

    参考：[#3808](https://www.sqlalchemy.org/trac/ticket/3808)

+   **[orm] [bug]**

    `relationship()` 构造的 primaryjoin 现在可以包括一个包含可调用函数以生成值的 `bindparam()` 对象。以前，延迟加载策略与此用法不兼容，并且还会无法正确检测是否应该使用“use_get”条件，如果主键与绑定参数有关。

    参考：[#3767](https://www.sqlalchemy.org/trac/ticket/3767)

+   **[orm] [bug]**

    从 ORM 刷新过程中发出的 UPDATE 现在可以适应对象主键中的列的 SQL 表达式元素，如果目标数据库支持 RETURNING 以提供新值，或者如果将 PK 值设置为“自身”以用于触发其他触发器/列的 onupdate。

    参考：[#3801](https://www.sqlalchemy.org/trac/ticket/3801)

+   **[orm] [bug]**

    修复了一个 bug，即“简单的一对多”条件允许延迟加载使用来自标识映射的 get() 失败的情况，如果关系的 primaryjoin 具有由 AND 分隔的多个子句，并且这些子句的顺序与每个子句中比较的主键列的顺序不同。这种顺序差异发生在复合外键的情况下，其中引用方的表绑定列在 .c 集合中的顺序与被引用方的主键列不同……如果使用声明性混入和/或 declared_attr 来设置列，则会经常发生这种情况。

    参考：[#3788](https://www.sqlalchemy.org/trac/ticket/3788)

+   **[orm] [bug]**

    当映射上的两个 `@validates` 装饰器使用相同名称时，会引发异常。一次只支持一个特定名称的验证器，没有机制将它们链接在一起，因为在函数装饰器级别上验证器的顺序无法确定。

    另请参阅

    具有相同名称的 @validates 装饰器现在会引发异常

    参考：[#3776](https://www.sqlalchemy.org/trac/ticket/3776)

+   **[orm] [bug]**

    在 `configure_mappers()` 过程中引发的 Mapper 错误现在在异常消息中明确包含了源映射器的名称，以帮助处理那些被包装异常本身不包含源映射器的情况。感谢 John Perkins 的拉取请求。

### orm 声明性

+   **[orm] [declarative] [change]**

    继承另一个类的声明性基类也将继承其文档字符串。这意味着 `as_declarative()` 的行为更像是一个普通的类装饰器。

### sql

+   **[sql] [bug]**

    修复了`Table`中的内部方法`_reset_exported()`会破坏对象状态的错误。该方法旨在用于可选择对象，并在某些情况下被 ORM 调用；错误的映射配置可能导致 ORM 在`Table`对象上调用此方法。

    此更改也**回溯**到：1.0.15

    参考：[#3755](https://www.sqlalchemy.org/trac/ticket/3755)

+   **[sql] [bug]**

    现在可以在编译时从语句内部传播执行选项到最外层语句，因此，如果嵌入元素想要将“autocommit”设置为 True，它可以将此传播到封闭语句。目前，此功能仅适用于嵌入在 SELECT 语句中的面向 DML 的 CTE，例如在 SELECT 语句中的 INSERT/UPDATE/DELETE。

    参考：[#3805](https://www.sqlalchemy.org/trac/ticket/3805)

+   **[sql] [bug]**

    通过`Column.server_default`参数发送的作为列默认值的字符串现在已经为引号进行了转义。

    另请参阅

    String server_default now literal quoted

    参考：[#3809](https://www.sqlalchemy.org/trac/ticket/3809)

+   **[sql] [bug] [postgresql]**

    添加了编译器级别的标志，用于 PostgreSQL 在涉及 JSON、HSTORE 索引操作符以及它们的操作数时放置额外的括号，因为观察到 PostgreSQL 至少在 HSTORE 索引操作符的优先规则在 9.4 和 9.5 之间不一致。

    参考：[#3806](https://www.sqlalchemy.org/trac/ticket/3806)

+   **[sql] [bug] [mysql]**

    `BaseException`异常类现在被`Connection`的异常处理例程拦截，并包括`ConnectionEvents.handle_error()`事件的处理。在系统级别异常（不是`Exception`的子类，包括`KeyboardInterrupt`和 greenlet `GreenletExit`类）的情况下，默认情况下现在会使`Connection`**失效**，以防止在处于未知且可能已损坏状态的数据库连接上发生进一步操作。这个更改主要针对 MySQL 驱动程序，但是这个更改适用于所有 DBAPIs。

    另请参阅

    Engines now invalidate connections, run error handlers for BaseException

    参考：[#3803](https://www.sqlalchemy.org/trac/ticket/3803)

+   **[sql] [bug]**

    “eq” 和 “ne” 运算符不再是“关联”运算符列表的一部分，尽管它们仍被认为是“可交换的”。这允许像 `(x == y) == z` 这样的表达式在 SQL 级别保持括号。感谢 John Passaro 的拉取请求。

    参考：[#3799](https://www.sqlalchemy.org/trac/ticket/3799)

+   **[sql] [bug]**

    在表达式中使用未命名的 `Column` 对象的字符串化，如在许多情况下包括 ORM 错误报告中，现在将在字符串上下文中呈现名称为“<name unknown>”，而不是引发编译错误。

    参考：[#3789](https://www.sqlalchemy.org/trac/ticket/3789)

+   **[sql] [bug]**

    当 ClauseElement 或非 SQLAlchemy 对象被错误地传递给 `.execute()` 时，会引发更具描述性的异常/消息；在所有情况下都一致引发新异常 ObjectNotExecutableError。

    参考：[#3786](https://www.sqlalchemy.org/trac/ticket/3786)

+   **[sql] [bug] [mysql] [postgresql]**

    修复了 JSON 数据类型中的回归，其中 JSON 索引值的“literal processor”不会被调用。现在从 JSONIndexType 和 JSONPathType 中调用原生的 String 和 Integer 数据类型。这适用于通用、PostgreSQL 和 MySQL 的 JSON 类型，也依赖于 [#3766](https://www.sqlalchemy.org/trac/ticket/3766)。

    参考：[#3765](https://www.sqlalchemy.org/trac/ticket/3765)

+   **[sql] [bug]**

    修复了 `Index` 无法从包含在 ORM 风格 `__clause_element__()` 结构中的复合 SQL 表达式中提取列的 bug。这个 bug 在 1.0.x 中也存在，但在 1.1 中更为明显，因为 hybrid_property @expression 现在返回一个包装元素。

    参考：[#3763](https://www.sqlalchemy.org/trac/ticket/3763)

### postgresql

+   **[postgresql] [bug]**

    调整 ON CONFLICT，使“inserted_primary_key”逻辑能够适应没有 INSERT 或 UPDATE 且没有净变化的情况。在这种情况下，该值为 None，而不是引发异常。

    参考：[#3813](https://www.sqlalchemy.org/trac/ticket/3813)

+   **[postgresql] [bug]**

    > 修复了新的 PG “on conflict” 结构中的问题，其中包括“excluded”命名空间的列在语句的 WHERE 子句中不会被表限定。

    参考：[#3807](https://www.sqlalchemy.org/trac/ticket/3807)

### mysql

+   **[mysql] [bug]**

    在 URL 查询字符串中添加了对解析 MySQL/Connector 布尔值和整数参数的支持：connection_timeout、connect_timeout、pool_size、get_warnings、raise_on_warnings、raw、consume_results、ssl_verify_cert、force_ipv6、pool_reset_session、compress、allow_local_infile、use_pure。

    这个更改也被**回溯**到：1.0.15

    参考：[#3787](https://www.sqlalchemy.org/trac/ticket/3787)

+   **[mysql] [bug]**

    修复了在 MySQL 下设置单表 inh 子类的错误，该子类包括额外列会破坏映射表的外键集合，从而干扰关系的初始化。

    参考：[#3766](https://www.sqlalchemy.org/trac/ticket/3766)

### mssql

+   **[mssql] [bug]**

    更改了用于获取“默认模式名称”的查询，从查询数据库主体表的查询更改为使用“schema_name()”函数，因为已报告有关前一系统在 Azure Data Warehouse 版本上不可用的问题。希望这将最终在所有 SQL Server 版本和身份验证样式上正常工作。

    此更改也**回溯**到：1.0.16

    参考：[#3810](https://www.sqlalchemy.org/trac/ticket/3810)

+   **[mssql] [bug]**

    更新了 pyodbc 的服务器版本信息方案，使用 SQL Server SERVERPROPERTY()，而不是依赖于 pyodbc.SQL_DBMS_VER，后者仍然不可靠，特别是对于 FreeTDS。

    此更改也**回溯**到：1.0.16

    参考：[#3814](https://www.sqlalchemy.org/trac/ticket/3814)

+   **[mssql] [bug]**

    将错误代码 20017“服务器意外的 EOF”添加到导致连接池重置的断开异常列表中。感谢 Ken Robbins 的拉取请求。

    此更改也**回溯**到：1.0.16

    参考：[#3791](https://www.sqlalchemy.org/trac/ticket/3791)

### misc

+   **[bug] [orm.declarative]**

    修复了在设置一个单表 inh 子类的 bug，该子类包括一个额外列，会破坏映射表的外键集合，从而干扰关系的初始化。

    此更改也**回溯**到：1.0.16

    参考：[#3797](https://www.sqlalchemy.org/trac/ticket/3797)

## 1.1.0b3

发布日期：2016 年 7 月 26 日

### orm

+   **[orm] [change]**

    删除了一个警告，该警告可以追溯到 0.4 版本，当在通过连接或单表继承继承的两个映射器上放置同名关系时会发出。该警告不适用于当前的工作单元实现。

    另请参阅

    继承映射器上的同名关系不再发出警告

    参考：[#3749](https://www.sqlalchemy.org/trac/ticket/3749)

### sql

+   **[sql] [bug]**

    修复了新的 CTE 功能中的错误，用于 update/insert/delete，作为包含语句（通常是 SELECT）内的 CTE 陈述，其中 oninsert 和 onupdate 值未调用嵌入语句。 

    参考：[#3745](https://www.sqlalchemy.org/trac/ticket/3745)

+   **[sql] [bug]**

    修复了新的 CTE 功能中的错误，用于 update/insert/delete，其中围绕语句的匿名（例如，未传递名称）`CTE`构造将失败。

    参考：[#3744](https://www.sqlalchemy.org/trac/ticket/3744)

### postgresql

+   **[postgresql] [bug]**

    修复了 PostgreSQL 方言对于 `TypeDecorator` 和 `Variant` 类型检查不够深入的 bug，以确定是否应该渲染 SMALLSERIAL 或 BIGSERIAL 而不是 SERIAL。

    此更改也被**回溯**至：1.0.14

    参考资料：[#3739](https://www.sqlalchemy.org/trac/ticket/3739)

### oracle

+   **[oracle] [bug]**

    修复了 `Select.with_for_update.of` 中的 bug，其中 Oracle 的“rownum”方法限制/偏移失败，因为“OF”子句内的表达式没有适应，必须在最顶层引用子查询内的表达式。如果需要，现在将表达式添加到子查询中。

    此更改也被**回溯**至：1.0.14

    参考资料：[#3741](https://www.sqlalchemy.org/trac/ticket/3741)

### misc

+   **[feature] [ext]**

    为新的 sqlalchemy.ext.indexable 扩展添加了“default”参数。

+   **[bug] [ext]**

    修复了 `sqlalchemy.ext.baked` 中的 bug，在涉及多个子查询加载器时，子查询加载器查询的解除处理失败了，这是由于变量作用域问题导致的。感谢 Mark Hahnenberg 提供的拉取请求。

    此更改也被**回溯**至：1.0.15

    参考资料：[#3743](https://www.sqlalchemy.org/trac/ticket/3743)

+   **[bug] [ext]**

    sqlalchemy.ext.indexable 在引发 AttributeError 时也会拦截 IndexError 和 KeyError。

## 1.1.0b2

发布日期：2016 年 7 月 1 日

### sql

+   **[sql] [bug]**

    修复了 SQL 数学否定运算符的问题，其中表达式的类型将不再是原始的数字类型。这将导致类型确定结果集行为的问题。

    此更改也被**回溯**至：1.0.14

    参考资料：[#3735](https://www.sqlalchemy.org/trac/ticket/3735)

+   **[sql] [bug]**

    修复了 sqlalchemy.util.Properties 的 `__getstate__` / `__setstate__` 方法由于 1.0 系列转换到 `__slots__` 导致的无法工作的 bug。该问题可能会影响一些第三方应用程序。感谢 Pieter Mulder 提供的拉取请求。

    此更改也被**回溯**至：1.0.14

    参考资料：[#3728](https://www.sqlalchemy.org/trac/ticket/3728)

+   **[sql] [bug]**

    仅包含整数类型的后端的 `Boolean` 数据类型的处理在纯 Python 和 C 扩展版本之间已经保持一致，C 扩展版本将接受来自数据库的任何整数值作为布尔值，而不仅仅是零和一；此外，发送到数据库的非布尔整数值将被强制转换为精确的零或一，而不是作为原始整数值传递。

    参见

    非本地布尔整数值在所有情况下被强制转换为零/一/无

    参考：[#3730](https://www.sqlalchemy.org/trac/ticket/3730)

+   **[sql] [bug]**

    在`Enum`中稍微回滚了验证规则，以允许未知字符串值通过，除非向 Enum���递了标志`validate_string=True`；当然，任何其他类型的对象仍然被拒绝。虽然立即使用是为了允许与 LIKE 枚举进行比较，但存在这种用法表明可能存在更多未知字符串比较用例，这暗示可能还有一些未知字符串插入用例。

    参考：[#3725](https://www.sqlalchemy.org/trac/ticket/3725)

### postgresql

+   **[postgresql] [bug] [ext]**

    在`sqlalchemy.ext.compiler`扩展中进行了轻微的行为更改，如果已建立的构造的编译方案没有自己专用的`__visit_name__`，则会将其删除。这在 1.0 中很少发生，但在 1.1 中，`ARRAY`子类`ARRAY`并具有此行为。因此，为其他方言（如 SQLite）设置编译处理程序将使主`ARRAY`对象不再可编译。

    参考：[#3732](https://www.sqlalchemy.org/trac/ticket/3732)

### mysql

+   **[mysql] [bug]**

    在不再为具有 AUTO_INCREMENT 的复合主键生成隐式 KEY 中稍微减少了“按照自动增量排序主键列”的描述，因此如果`PrimaryKeyConstraint`被明确定义，列的顺序将被保持完全一致，允许在必要时控制此行为。

    参考：[#3726](https://www.sqlalchemy.org/trac/ticket/3726)

## 1.1.0b1

发布日期：2016 年 6 月 16 日

### orm

+   **[orm] [feature] [ext]**

    添加了一个新的 ORM 扩展 Indexable，允许构建引用“索引”结构的特定元素的 Python 属性，例如数组和 JSON 字段。感谢 Jeong YunWon 的拉取请求。

    另请参阅

    新的可索引 ORM 扩展

+   **[orm] [feature]**

    添加了新标志 `Session.bulk_insert_mappings.render_nulls`，它允许使用 NULL 值进行 ORM 批量 INSERT；这将绕过服务器端默认值，但允许使用相同的列集形成所有语句，从而使它们可以批处理。由 Tobias Sauerwein 提供的拉取请求。

+   **[orm] [功能]**

    添加了新事件 `AttributeEvents.init_scalar()`，以及一个新的示例套件，说明了其用法。此事件可用于在对象持久化之前为 Python 端的属性提供由 Core 生成的默认值。

    请参见

    新的 init_scalar() 事件拦截 ORM 级别的默认值

    参考：[#1311](https://www.sqlalchemy.org/trac/ticket/1311)

+   **[orm] [功能]**

    添加了 `AutomapBase.prepare.schema` 到 `AutomapBase.prepare()` 方法，以指示应从哪个模式中反映表，如果不是默认模式。由 Josh Marlow 提供的拉取请求。

+   **[orm] [功能]**

    添加了新参数 `mapper.passive_deletes` 到可用的映射器选项。这允许针对基表的联合表继承映射进行 DELETE 操作，同时允许 ON DELETE CASCADE 处理从子类表中删除行。

    请参见

    对于联合继承映射的被动删除功能

    参考：[#2349](https://www.sqlalchemy.org/trac/ticket/2349)

+   **[orm] [功能]**

    对核心 SQL 结构调用 str() 方法已经变得更加“友好”，当结构包含非标准 SQL 元素时，例如 RETURNING、数组索引操作或方言特定或自定义数据类型时。在这些情况下，将返回一个字符串，呈现出结构的近似值（通常是其 PostgreSQL 风格版本），而不是引发错误。

    请参见

    没有方言的核心 SQL 结构的“友好”字符串化

    参考：[#3631](https://www.sqlalchemy.org/trac/ticket/3631)

+   **[orm] [功能]**

    对于`Query`的`str()`调用现在将考虑`Session`绑定的`Engine`，在生成 SQL 的字符串形式时，以便显示将要发送到数据库的实际 SQL，如果可能的话。以前，只有与映射相关联的`MetaData`的引擎会被使用，如果存在的话。如果无法在`Session`或与映射相关联的`MetaData`上找到绑定，则使用“默认”方言来呈现 SQL，就像以前一样。

    另请参阅

    查询的字符串化将查询会话以获取正确的方言

    参考：[#3081](https://www.sqlalchemy.org/trac/ticket/3081)

+   **[orm] [功能]**

    `SessionEvents`套件现在包括事件，允许对所有对象生命周期状态转换进行明确跟踪，以`Session`本身为基础，例如挂起、瞬态、持久、分离。每个事件中对象的状态也被定义。

    另请参阅

    新会话生命周期事件

    参考：[#2677](https://www.sqlalchemy.org/trac/ticket/2677)

+   **[orm] [功能]**

    添加了一个新的会话生命周期状态 deleted。这个新状态表示一个已从 persistent 状态中删除的对象，一旦事务提交，将转移到 detached 状态。这解决了长期存在的问题，即被删除的对象存在于持久和分离之间的灰色区域。`InstanceState.persistent`访问器**不再**将已删除对象报告为持久；相反，这些对象的`InstanceState.deleted`访问器将为 True，直到它们变为分离状态。

    另请参阅

    新会话生命周期事件

    参考：[#2677](https://www.sqlalchemy.org/trac/ticket/2677)

+   **[orm] [功能]**

    添加了对将映射类或映射实例传递到被解释为 SQL 绑定参数的上下文的常见错误情况的新检查；为此引发了新异常。

    另请参阅

    为传递映射类、实例作为 SQL 文字添加了具体检查

    参考：[#3321](https://www.sqlalchemy.org/trac/ticket/3321)

+   **[orm] [功能]**

    添加了新的关系加载策略`raiseload()`（也可通过`lazy='raise'`访问）。这个策略几乎像`noload()`一样工作，但是它不是返回`None`而是引发一个 InvalidRequestError。拉取请求由 Adrian Moennich 提供。

    另请参阅

    新的“raise” / “raise_on_sql”加载器策略

    参考：[#3512](https://www.sqlalchemy.org/trac/ticket/3512)

+   **[orm] [变更]**

    `Mapper.order_by` 参数已被弃用。这是一个旧参数，与 SQLAlchemy 的工作方式不再相关，一旦引入了查询对象。通过弃用它，我们确立了我们不支持不起作用的用例，并鼓励应用程序摆脱使用该参数的做法。

    另请参阅

    Mapper.order_by 已弃用

    参考：[#3394](https://www.sqlalchemy.org/trac/ticket/3394)

+   **[orm] [变更]**

    `Session.weak_identity_map` 参数已被弃用。请参阅 Session Referencing Behavior 中的新方案，使用基于事件的方法来维护强身份映射行为。

    另请参阅

    新的会话生命周期事件

    参考：[#2677](https://www.sqlalchemy.org/trac/ticket/2677)

+   **[orm] [错误]**

    修复了一个问题，即将对象从一个父对象更改为另一个父对象的多对一更改可能与未刷新的外键属性的修改结合使用时工作不一致的问题。现在，属性移动考虑了外键的数据库提交值，以定位正在移动的对象的“以前”父对象。这样可以正确触发事件，包括反向引用事件。以前，这些事件并不总是会触发。可能依赖于先前损坏行为的应用可能会受到影响。

    另请参阅

    修复了涉及由用户启动的外键操纵的多对一对象移动的问题

    参考：[#3708](https://www.sqlalchemy.org/trac/ticket/3708)

+   **[orm] [错误]**

    修复了一个 bug，即当使用`session.merge(obj, load=False)`将对象合并到会话中时，延迟列可能会意外地设置为在下一个对象范围内取消过期时进行数据库加载。

    参考：[#3488](https://www.sqlalchemy.org/trac/ticket/3488)

+   **[orm] [错误] [mysql]**

    进一步延续了关于 MySQL 常见异常情况的讨论，即在[#2696](https://www.sqlalchemy.org/trac/ticket/2696)中首次涵盖的保存点首先被取消的情况，当 SAVEPOINT 在回滚之前消失时，`Session`所处的失败模式已经得到改进，以允许`Session`在该保存点之外继续运行。假定保存点操作失败并被取消。

    另请参阅

    数据库取消 SAVEPOINT 时改进的 Session 状态

    参考：[#3680](https://www.sqlalchemy.org/trac/ticket/3680)

+   **[orm] [bug]**

    修复了一个 bug，即新插入的实例如果被回滚，仍可能在下一个事务中引起持久性冲突，因为实例未被检查是否已过期。此修复将解决一大类错误地导致“具有标识 X 的新实例与持久实例 Y 冲突”的情况。

    另请参阅

    修复了错误的“新实例 X 与持久实例 Y 冲突”刷新错误

    参考：[#3677](https://www.sqlalchemy.org/trac/ticket/3677)

+   **[orm] [bug]**

    对`Query.correlate()`的工作方式进行了改进，以便在使用代表多个表直接连接的“多态”实体时，语句将确保连接中的所有表都是相关的。

    另请参阅

    改进了具有多态实体的 Query.correlate 方法

    参考：[#3662](https://www.sqlalchemy.org/trac/ticket/3662)

+   **[orm] [bug]**

    修复了一个 bug，该 bug 会导致一个急切加载的多对一属性不被加载，如果连接的急切加载来自一个行，其中同一实体多次出现，有些要求属性急切加载，而其他则不需要。这里的逻辑被修改为即使不同的加载路径已经处理了父实体，也要考虑属性。

    另请参阅

    在一行中同一实体多次出现时的连接急切加载

    参考：[#3431](https://www.sqlalchemy.org/trac/ticket/3431)

+   **[orm] [bug]**

    对于`Query.distinct()`与`Query.order_by()`组合时向生成的 SQL 添加列的逻辑进行了改进，使得已经存在的列不会被第二次添加，即使它们使用不同名称标记。尽管有这个改变，SQL 中添加的额外列从未在最终结果中返回，因此这个改变只影响语句的字符串形式以及在核心执行上下文中使用时的行为。此外，当使用 DISTINCT ON 格式时，不再添加列，前提是查询没有由于连接的急加载而包装在子查询中。

    另请参阅

    使用 DISTINCT + ORDER BY 不再冗余添加列

    参考：[#3641](https://www.sqlalchemy.org/trac/ticket/3641)

+   **[orm] [bug]**

    修复了两个同名关系引用基类和具体继承子类时，如果这些关系是使用“backref”设置的，那么会引发错误，而使用 relationship()设置相同配置的情况下，使用冲突名称将成功，就像在具体映射的情况下允许的那样。

    另请参阅

    应用于具体继承子类时，同名 backrefs 不会引发错误

    参考：[#3630](https://www.sqlalchemy.org/trac/ticket/3630)

+   **[orm] [bug]**

    `Session.merge()`方法现在在发出 INSERT 之前通过主键跟踪待处理对象，并在遇到具有重复主键的不同对象时将它们合并在一起，这在最好的情况下基本上是半确定性的。这种行为与持久对象已经发生的情况相匹配。

    另请参阅

    Session.merge 解决待处理冲突与持久对象相同

    参考：[#3601](https://www.sqlalchemy.org/trac/ticket/3601)

+   **[orm] [bug]**

    修复了在某些不适当情况下，例如在从单继承子类的 exists()查询时，会将“单表继承”标准添加到查询末尾的错误。

    另请参阅

    进一步修复单表继承查询

    参考：[#3582](https://www.sqlalchemy.org/trac/ticket/3582)

+   **[orm] [bug]**

    添加了一个新的类型级别修饰符`TypeEngine.evaluates_none()`，指示 ORM 应将一组正值的 None 持久化为值 NULL，而不是在 INSERT 语句中省略列。此功能既用作[#3514](https://www.sqlalchemy.org/trac/ticket/3514)的实现的一部分，也作为任何类型可用的独立功能。

    另请参阅

    允许显式持久化 NULL 覆盖默认值的新选项

    参考：[#3250](https://www.sqlalchemy.org/trac/ticket/3250)

+   **[orm] [bug]**

    `Session.bulk_save_objects()`和相关批量方法内部调用的“簿记”功能已经减少到目前未使用的程度，例如在 INSERT 或 UPDATE 语句之后获取列默认值的检查。

    参考：[#3526](https://www.sqlalchemy.org/trac/ticket/3526)

+   **[orm] [bug] [postgresql]**

    关于与 PostgreSQL `JSON` 类型一起使用值`None`的附加修复已经完成。当`JSON.none_as_null`标志保持默认值`False`时，ORM 现在将正确地在 ORM 对象上的值设置为`None`或在`Session.bulk_insert_mappings()`中使用值`None`时，将 JSON 的“'null'”字符串插入到列中，**包括**如果列上有默认值或服务器默认值。

    另请参阅

    JSON“null”在 ORM 操作中按预期插入，在不存在时被省略

    允许显式持久化 NULL 覆盖默认值的新选项

    参考：[#3514](https://www.sqlalchemy.org/trac/ticket/3514)

### engine

+   **[engine] [feature]**

    添加了连接池事件`ConnectionEvents.close()`，`ConnectionEvents.detach()`，`ConnectionEvents.close_detached()`。

+   **[engine] [feature]**

    用于日志记录、异常和`repr()`目的的所有绑定参数集和结果行的字符串格式化现在在每个集合中截断非常大的标量值，包括“N 个字符被截断”注释，类似于大型多参数集本身被截断的显示方式。

    另请参阅

    在日志和异常显示中截断大型参数和行值

    参考：[#2837](https://www.sqlalchemy.org/trac/ticket/2837)

+   **[engine] [feature]**

    为`Table`对象添加了多租户模式翻译。这支持应用程序在许多模式中使用相同的`Table`对象的用例，例如每个用户一个模式。添加了一个新的执行选项`Connection.execution_options.schema_translate_map`。

    另请参阅

    Table 对象的多租户模式翻译

    参考：[#2685](https://www.sqlalchemy.org/trac/ticket/2685)

+   **[engine] [feature]**

    为引擎添加了新的入口系统，允许在 URL 的查询字符串中声明“插件”。可以编写自定义插件，这些插件将有机会在前期修改和/或使用引擎的 URL 和关键字参数，然后在引擎创建时将获得引擎本身以允许进行额外的修改或事件注册。插件被编写为`CreateEnginePlugin`的子类；请参阅该类以获取详细信息。

    参考：[#3536](https://www.sqlalchemy.org/trac/ticket/3536)

### sql

+   **[sql] [feature]**

    通过新的`FromClause.tablesample()`方法和独立函数添加了 TABLESAMPLE 支持。感谢 Ilja Everilä的拉取请求。

    另请参阅

    支持 TABLESAMPLE

    参考：[#3718](https://www.sqlalchemy.org/trac/ticket/3718)

+   **[sql] [feature]**

    支持窗口函数中的范围，使用`over.range_`和`over.rows`参数。

    另请参阅

    支持窗口函数中的 RANGE 和 ROWS 规范

    参考：[#3049](https://www.sqlalchemy.org/trac/ticket/3049)

+   **[sql] [feature]**

    实现了 SQLite 和 PostgreSQL 的 CHECK 约束的反射。可以通过新的检查器方法`Inspector.get_check_constraints()`以及在反射`Table`对象时以`CheckConstraint`对象的形式存在于约束集合中。感谢 Alex Grönholm 的拉取请求。

+   **[sql] [feature]**

    新增`ColumnOperators.is_distinct_from()`和`ColumnOperators.isnot_distinct_from()`操作符；感谢 Sebastian Bank 的拉取请求。

    另请参阅

    IS DISTINCT FROM 和 IS NOT DISTINCT FROM 的支持

+   **[sql] [功能]**

    在`DDLCompiler.visit_create_table()`中增加了一个名为`DDLCompiler.create_table_suffix()`的钩子，允许自定义方言在“CREATE TABLE”子句之后添加关键字。感谢 Mark Sandan 的拉取请求。

+   **[sql] [功能]**

    负整数索引现在由`ResultProxy`返回的行支持。感谢 Emanuele Gaifas 的拉取请求。

    另请参阅

    核心结果行支持负整数索引

+   **[sql] [功能]**

    增加了`Select.lateral()`和相关构造，以允许使用 SQL 标准的 LATERAL 关键字，目前仅受 PostgreSQL 支持。

    另请参阅

    SQL LATERAL 关键字的支持

    参考：[#2857](https://www.sqlalchemy.org/trac/ticket/2857)

+   **[sql] [功能]**

    增加了对“FULL OUTER JOIN”在 Core 和 ORM 中的渲染支持。感谢 Stefan Urbanek 的拉取请求。

    另请参阅

    核心和 ORM 支持 FULL OUTER JOIN

    参考：[#1957](https://www.sqlalchemy.org/trac/ticket/1957)

+   **[sql] [功能]**

    CTE 功能已扩展到支持所有 DML，允许 INSERT、UPDATE 和 DELETE 语句指定自己的 WITH 子句，以及当这些语句包含 RETURNING 子句时，这些语句本身也可以是 CTE 表达式。

    另请参阅

    CTE 支持 INSERT、UPDATE、DELETE

    参考：[#2551](https://www.sqlalchemy.org/trac/ticket/2551)

+   **[sql] [功能]**

    增加了对 PEP-435 风格的枚举类的支持，即 Python 3 的`enum.Enum`类，但也包括兼容的枚举库，到`Enum`数据类型。`Enum`数据类型现在还在 Python 中验证传入值，并添加了一个选项来避免创建 CHECK 约束`Enum.create_constraint`。感谢 Alex Grönholm 的拉取请求。

    另请参阅

    支持 Python 的原生枚举类型和兼容形式

    Enum 类型现在在 Python 中验证值

    参考：[#3095](https://www.sqlalchemy.org/trac/ticket/3095), [#3292](https://www.sqlalchemy.org/trac/ticket/3292)

+   **[sql] [feature]**

    对最近添加的 `TextClause.columns()` 方法以及其与结果行处理的交互进行了深度改进，现在允许将传递给该方法的列与语句中的结果列进行位置匹配，而不仅仅是根据名称匹配。这样做的好处包括，当将文本 SQL 语句链接到 ORM 或核心表模型时，无需进行常见列名的标记或去重，这也意味着无需担心标签名称如何与 ORM 列匹配等等。此外，`ResultProxy` 在某些情况下进一步增强，以更精确地将列和字符串键映射到行。

    另请参见

    ResultSet 列匹配增强；文本 SQL 的位置列设置 - 特性概述

    TextClause.columns() 将按位置而不是按名称匹配列 - 向后兼容性备注

    参考：[#3501](https://www.sqlalchemy.org/trac/ticket/3501)

+   **[sql] [feature]**

    在核心中添加了一个新类型 `JSON`。这是 PostgreSQL 的基础 `JSON` 类型以及新 `JSON` 类型的基础，因此可以使用 PG/MySQL 不可知的 JSON 列。该类型具有基本的索引和路径搜索支持。

    另请参见

    Core 添加了对 JSON 的支持

    参考：[#3619](https://www.sqlalchemy.org/trac/ticket/3619)

+   **[sql] [feature]**

    添加了对形式为 `<function> WITHIN GROUP (ORDER BY <criteria>)` 的“set-aggregate”函数的支持，使用方法 `FunctionElement.within_group()`。已添加一系列常见的 set-aggregate 函数，其返回类型从集合派生。这包括像 `percentile_cont`、`dense_rank` 等函数。

    另请参见

    新的功能特性，“WITHIN GROUP”、“array_agg”和集合聚合函数

    参考：[#1370](https://www.sqlalchemy.org/trac/ticket/1370)

+   **[sql] [feature] [postgresql]**

    增加了对 SQL 标准函数 `array_agg` 的支持，该函数会自动返回正确类型的 `ARRAY` 并支持索引/切片操作，还有 `array_agg()`，它返回一个带有额外比较功能的 `ARRAY`。由于目前只有 PostgreSQL 支持数组，因此只在 PostgreSQL 上有效。还新增了一个支持 PG 的“ORDER BY”扩展的新构造 `aggregate_order_by`。

    另请参阅

    新函数功能，“WITHIN GROUP”，array_agg 和 set 聚合函数

    参考：[#3132](https://www.sqlalchemy.org/trac/ticket/3132)

+   **[sql] [feature]**

    在核心中新增了一个新类型 `ARRAY`。这是 PostgreSQL `ARRAY` 类型的基础，现在已经成为核心的一部分，以开始支持各种 SQL 标准的数组支持功能，包括一些函数和最终支持其他具有“数组”概念的数据库上的本地数组，如 DB2 或 Oracle。此外，还添加了新的运算符 `any_()` 和 `all_()`。这些不仅支持 PostgreSQL 上的数组构造，还支持可在 MySQL 上使用的子查询（但遗憾的是在 PostgreSQL 上不可用）。

    另请参阅

    核心添加了数组支持；新增了 ANY 和 ALL 运算符

    参考：[#3516](https://www.sqlalchemy.org/trac/ticket/3516)

+   **[sql] [change] [mysql]**

    `Column` 认为自己是“自动增量”列的系统已更改，因此不再为具有复合主键的 `Table` 隐式启用自动增量。为了能够为复合主键成员列启用自动增量，同时保持 SQLAlchemy 长期以来为单个整数主键启用隐式自动增量的行为，已将第三状态添加到 `Column.autoincrement` 参数中 `"auto"`，这现在是默认值。

    另请参阅

    不再为复合主键列隐式启用 .autoincrement 指令

    不再为具有 AUTO_INCREMENT 的复合主键生成隐式 KEY

    参考：[#3216](https://www.sqlalchemy.org/trac/ticket/3216)

+   **[sql] [bug]**

    `FromClause.count()` 已被弃用。该函数使用表中的任意列，并不可靠；对于 Core 使用，应优先使用 `func.count()`。

    参考：[#3724](https://www.sqlalchemy.org/trac/ticket/3724)

+   **[sql] [bug]**

    修复了一个断言，如果一个与小写-t `TableClause` 关联的 `Column` 关联了一个 `Index`，则会不太恰当地引发；为了将索引与 `Table` 关联起来，应忽略该关联。

    参考：[#3616](https://www.sqlalchemy.org/trac/ticket/3616)

+   **[sql] [bug]**

    `type_coerce()` 构造现在是一个完全成熟的 Core 表达式元素，在编译时进行延迟评估。以前，该函数只是一个转换函数，通过返回一个 `Label` 或基于列的表达式的副本的 `BindParameter` 处理不同的表达式输入，特别是当 ORM 级别的表达式转换将列转换为绑定参数时（例如用于延迟加载）时，该操作无法被逻辑地维护。

    另请参阅

    type_coerce 函数现在是一个持久的 SQL 元素

    引用：[#3531](https://www.sqlalchemy.org/trac/ticket/3531)

+   **[sql] [错误]**

    `TypeDecorator`类型扩展现在将与`SchemaType`实现一起工作，通常是`Enum`或`Boolean`，以确保从实现类型传播到外部类型的每个表事件。这些事件用于确保约束或 PostgreSQL 类型（例如 ENUM）与父表一起正确创建（可能删除）。

    另请参阅

    TypeDecorator 现在自动与 Enum、Boolean、“模式”类型配合工作

    引用：[#2919](https://www.sqlalchemy.org/trac/ticket/2919)

+   **[sql] [错误]**

    `union()`构造及相关构造如`Query.union()`现在处理嵌套的 SELECT 语句需要加括号的情况，因为它们包含 LIMIT、OFFSET 和/或 ORDER BY。这些查询**在 SQLite 上不起作用**，并且在该后端上会像以前一样失败，但现在应该在所有其他后端上起作用。

    另请参阅

    使用 LIMIT/OFFSET/ORDER BY 的 SELECT 的 UNION 或类似操作现在会给嵌套的 SELECT 加括号

    引用：[#2528](https://www.sqlalchemy.org/trac/ticket/2528)

### 模式

+   **[模式] [增强]**

    传递给`Column`对象的默认生成函数现在通过“update_wrapper”运行，或者如果传递了可调用的非函数，则通过等效函数运行，以便内省工具保留包装函数的名称和文档字符串。拉取请求由 hsum 提供。

### postgresql

+   **[postgresql] [功能]**

    添加了对 PostgreSQL 的 INSERT..ON CONFLICT 的支持，使用了一个新的 PostgreSQL 特定的`Insert`对象。由 Robin Thomas 在此处进行了拉取请求和大量工作。

    另请参阅

    支持 INSERT..ON CONFLICT (DO UPDATE | DO NOTHING)

    引用：[#3529](https://www.sqlalchemy.org/trac/ticket/3529)

+   **[postgresql] [功能]**

    如果在`Index`上设置了`postgresql_concurrently`标志，并且检测到正在使用的数据库为 PostgreSQL 版本 9.2 或更高版本，则 DROP INDEX 的 DDL 将发出“CONCURRENTLY”。对于 CREATE INDEX，还添加了数据库版本检测，如果 PG 版本低于 8.2，则将省略该子句。感谢 Iuri de Silvio 提供的拉取请求。

+   **[postgresql] [功能]**

    添加了新参数`PGInspector.get_view_names.include`，允许指定应返回哪种类型的视图。当前包括“plain”和“materialized”视图。感谢 Sebastian Bank 提供的拉取请求。

    参考：[#3588](https://www.sqlalchemy.org/trac/ticket/3588)

+   **[postgresql] [功能]**

    将`postgresql_tablespace`作为参数添加到`Index`中，以允许在 PostgreSQL 中为索引指定 TABLESPACE。与`Table`上的同名参数相辅相成。感谢 Benjamin Bertrand 提供的拉取请求。

    参考：[#3720](https://www.sqlalchemy.org/trac/ticket/3720)

+   **[postgresql] [功能]**

    添加了新参数`GenerativeSelect.with_for_update.key_share`，它将在 PostgreSQL 后端上呈现`FOR NO KEY UPDATE`版本的`FOR UPDATE`和`FOR KEY SHARE`，而不是`FOR SHARE`。感谢 Sergey Skopin 提供的拉取请求。

+   **[postgresql] [功能] [oracle]**

    添加了新参数`GenerativeSelect.with_for_update.skip_locked`，它将在 PostgreSQL 和 Oracle 后端上为`FOR UPDATE`或`FOR SHARE`锁呈现`SKIP LOCKED`短语。感谢 Jack Zhou 提供的拉取请求。

+   **[postgresql] [功能]**

    为 PyGreSQL PostgreSQL 方言添加了一个新的方言。感谢 Christoph Zwerschke 和 Kaolin Imago Fire 的努力。

+   **[postgresql] [功能]**

    添加了一个新常量`JSON.NULL`，表示应使用 JSON NULL 值作为值，而不考虑其他设置。

    另请参阅

    新增 JSON.NULL 常量

    参考：[#3514](https://www.sqlalchemy.org/trac/ticket/3514)

+   **[postgresql] [更改]**

    长期弃用的`sqlalchemy.dialects.postgres`模块已被移除；多年来一直发出警告，项目应该调用`sqlalchemy.dialects.postgresql`。形式为`postgres://`的引擎 URL 仍将继续运行。

+   **[postgresql] [错误]**

    增加了对将物化视图源反射到`Inspector.get_view_definition()`方法的 PostgreSQL 版本的支持。

    参考：[#3587](https://www.sqlalchemy.org/trac/ticket/3587)

+   **[postgresql] [bug]**

    引用了一个引用`Enum`或`ENUM`子类型的`ARRAY`对象现在在类型在“CREATE TABLE”或“DROP TABLE”中使用时会发出预期的“CREATE TYPE”和“DROP TYPE” DDL。

    另请参阅

    带有 ENUM 的 ARRAY 现在将为 ENUM 发出 CREATE TYPE

    参考：[#2729](https://www.sqlalchemy.org/trac/ticket/2729)

+   **[postgresql] [bug]**

    特殊数据类型如`ARRAY`、`JSON`和`HSTORE`上的“可哈希”标志现在设置为 False，这允许在包含行内实体的 ORM 查询中获取这些类型。

    另请参阅

    关于“不可哈希”类型的更改，影响 ORM 行的去重

    ARRAY 和 JSON 类型现在正确指定“不可哈希”

    参考：[#3499](https://www.sqlalchemy.org/trac/ticket/3499)

+   **[postgresql] [bug]**

    PostgreSQL `ARRAY` 类型现在支持多维索引访问，例如表达式`somecol[5][6]`，无需任何显式转换或类型强制转换，只要`ARRAY.dimensions` 参数设置为所需的维数即可。

    另请参阅

    正确的 SQL 类型来自于 ARRAY、JSON、HSTORE 的索引访问

    参考：[#3487](https://www.sqlalchemy.org/trac/ticket/3487)

+   **[postgresql] [bug]**

    当使用索引访问时，`JSON` 和 `JSONB` 的返回类型已经修复，使其像 PostgreSQL 本身一样工作，并返回一个表达式，该表达式本身是类型为 `JSON` 或 `JSONB` 的类型。之前，访问器会返回 `NullType`，这将禁止使用后续类似 JSON 的运算符。

    另请参阅

    从数组、JSON、HSTORE 的索引访问中建立正确的 SQL 类型

    参考：[#3503](https://www.sqlalchemy.org/trac/ticket/3503)

+   **[postgresql] [bug]**

    `JSON`、`JSONB` 和 `HSTORE` 数据类型现在允许完全控制从索引文本访问操作的返回类型，对于 JSON 类型使用 `column[someindex].astext`，对于 HSTORE 类型使用 `column[someindex]`，通过 `JSON.astext_type` 和 `HSTORE.text_type` 参数。

    另请参阅

    从数组、JSON、HSTORE 的索引访问中建立正确的 SQL 类型

    参考：[#3503](https://www.sqlalchemy.org/trac/ticket/3503)

+   **[postgresql] [bug]**

    `Comparator.astext` 修饰符不再隐式调用 `ColumnElement.cast()`，因为 PG 的 JSON/JSONB 类型允许彼此之间的交叉转换。在 JSON 索引访问上使用 `ColumnElement.cast()` 的代码，例如 `col[someindex].cast(Integer)`，需要改为显式调用 `Comparator.astext`。

    另请参阅

    现在需要显式调用 .astext 来执行 JSON 转换操作

    引用：[#3503](https://www.sqlalchemy.org/trac/ticket/3503)

### mysql

+   **[mysql] [功能]**

    为 MySQL 驱动程序添加了“自动提交”支持，通过 AUTOCOMMIT 隔离级别设置。 感谢 Roman Podoliaka 的拉取请求。

    另请参阅

    增加了对“自动提交”“隔离级别”的支持

    引用：[#3332](https://www.sqlalchemy.org/trac/ticket/3332)

+   **[mysql] [功能]**

    为 MySQL 5.7 添加了 `JSON`。 JSON 类型提供了在 MySQL 中持久化 JSON 值以及“getitem”和“getpath”基本运算符的支持，利用 `JSON_EXTRACT` 函数来引用 JSON 结构中的单个路径。

    另请参阅

    MySQL JSON 支持

    引用：[#3547](https://www.sqlalchemy.org/trac/ticket/3547)

+   **[mysql] [更改]**

    MySQL 方言不再在使用 InnoDB 的表的 CREATE TABLE DDL 生成额外的“KEY”指令，该表具有具有 AUTO_INCREMENT 的复合主键，而不是在第一列上； 为了克服这里 InnoDB 的限制，PRIMARY KEY 约束现在生成的 AUTO_INCREMENT 列放在列列表中的第一位。

    另请参阅

    不再为具有 AUTO_INCREMENT 的列的复合主键生成隐式 KEY

    对于复合主键列，不再隐式启用 .autoincrement 指令

    引用：[#3216](https://www.sqlalchemy.org/trac/ticket/3216)

### sqlite

+   **[sqlite] [功能]**

    SQLite 方言现在反映了外键约束中的 ON UPDATE 和 ON DELETE 词组。 感谢 Michal Petrucha 的拉取请求。

+   **[sqlite] [功能]**

    SQLite 方言现在反映了主键约束的名称。 感谢 Diana Clarke 的拉取请求。

    另请参阅

    主键约束名称的反射

    引用：[#3629](https://www.sqlalchemy.org/trac/ticket/3629)

+   **[sqlite] [更改]**

    为 SQLite 方言添加了对 `Inspector.get_schema_names()` 方法的支持，以与 SQLite 一起工作； 感谢 Brian Van Klaveren 的拉取请求。 修复了与模式有关的索引创建以及模式绑定表中外键约束的反射支持。

    另请参阅

    改进的远程模式支持

+   **[sqlite] [错误]**

    在 SQLite 上的右嵌套连接的解决方案，其中它们被重写为子查询以解决 SQLite 不支持此语法的问题，当检测到 SQLite 版本为 3.7.16 或更高版本时解除限制。

    另请参阅

    对 SQLite 版本 3.7.16 解除了右嵌套连接的限制

    参考：[#3634](https://www.sqlalchemy.org/trac/ticket/3634)

+   **[sqlite] [错误]**

    当检测到 SQLite 版本 3.10.0 或更高版本时，对于某些查询以 `tablename.columnname` 形式传递列名的 SQLite 的意外行为的解决方法现在已禁用。

    另请参阅

    针对 SQLite 版本 3.10.0 解除的点列名解决方法

    参考：[#3633](https://www.sqlalchemy.org/trac/ticket/3633)

### mssql

+   **[mssql] [功能]**

    `mssql_clustered` 标志现在默认为 `None`，可设置为 False，这将为主键渲染 NONCLUSTERED 关键字，允许使用不同的索引作为“clustered”。感谢 Saulius Žemaitaitis 的拉取请求。

+   **[mssql] [功能]**

    通过 `create_engine.isolation_level` 和 `Connection.execution_options.isolation_level` 参数，为 SQL Server 方言添加了基本的隔离级别支持。

    另请参阅

    为 SQL Server 添加事务隔离级别支持

    参考：[#3534](https://www.sqlalchemy.org/trac/ticket/3534)

+   **[mssql] [更改]**

    `legacy_schema_aliasing` 标志，作为版本 1.0.5 中的一部分引入，允许禁用 MSSQL 方言为模式限定表创建别名的尝试，现在默认为 False；除非显式打开，否则旧行为现已禁用。

    另请参阅

    legacy_schema_aliasing 标志现在设置为 False

    参考：[#3434](https://www.sqlalchemy.org/trac/ticket/3434)

+   **[mssql] [错误]**

    调整 mxODBC 方言以在适当情况下使用 `BinaryNull` 符号与 `VARBINARY` 数据类型一起。感谢 Sheila Allen 的拉取请求。

+   **[mssql] [错误]**

    修复了 SQL Server 方言在将字符串或其他可变长度列类型反映为无界长度时分配字符串的长度属性为 `"max"` 的问题。虽然 SQL Server 方言支持显式使用 `"max"` 标记，但它不是基本字符串类型的正常约定的一部分，而是长度应该保持为 None。方言现在在反射类型时将长度分配为 None，以便在其他上下文中正常使用该类型。

    另请参阅

    String / varlength 类型在反射时不再显式表示“max”

    参考：[#3504](https://www.sqlalchemy.org/trac/ticket/3504)

### 杂项

+   **[特性] [扩展]**

    向 Mutation Tracking 扩展添加了`MutableSet`和`MutableList`辅助类。感谢 Jeong YunWon 的拉取请求。

    参考：[#3297](https://www.sqlalchemy.org/trac/ticket/3297)

+   **[错误] [扩展]**

    现在在类级别尊重在混合属性或方法上指定的文档字符串，使其能够与 Sphinx autodoc 等工具一起使用。这里的机制必然涉及对混合属性进行一些包装，这可能会导致它们在内省时显示不同。

    另请参阅

    混合属性和方法现在也传播文档字符串以及.info

    参考：[#3653](https://www.sqlalchemy.org/trac/ticket/3653)

+   **[错误] [sybase]**

    不支持的 Sybase 方言现在在尝试编译包含“offset”的查询时引发`NotImplementedError`；Sybase 没有直接的“offset”功能。

    参考：[#2278](https://www.sqlalchemy.org/trac/ticket/2278)

## 1.1.18

发布日期：2018 年 3 月 6 日

### postgresql

+   **[postgresql] [错误] [py3k]**

    修复了在 PostgreSQL COLLATE / ARRAY 调整中首次引入的错误，最初在[#4006](https://www.sqlalchemy.org/trac/ticket/4006)中，Python 3.7 正则表达式的新行为导致修复失败。

    参考：[#4208](https://www.sqlalchemy.org/trac/ticket/4208)

### mysql

+   **[mysql] [错误]**

    MySQL 方言现在显式使用`SELECT @@version`查询服务器版本，以确保我们获得正确的版本信息。代理服务器如 MaxScale 干扰了传递给 DBAPI 的 connection.server_version 值，因此这不再可靠。

    参考：[#4205](https://www.sqlalchemy.org/trac/ticket/4205)

### postgresql

+   **[postgresql] [错误] [py3k]**

    修复了在 PostgreSQL COLLATE / ARRAY 调整中首次引入的错误，最初在[#4006](https://www.sqlalchemy.org/trac/ticket/4006)中，Python 3.7 正则表达式的新行为导致修复失败。

    参考：[#4208](https://www.sqlalchemy.org/trac/ticket/4208)

### mysql

+   **[mysql] [错误]**

    MySQL 方言现在显式使用`SELECT @@version`查询服务器版本，以确保我们获得正确的版本信息。代理服务器如 MaxScale 干扰了传递给 DBAPI 的 connection.server_version 值，因此这不再可靠。

    参考：[#4205](https://www.sqlalchemy.org/trac/ticket/4205)

## 1.1.17

发布日期：2018 年 2 月 22 日

+   **[错误] [扩展]**

    修复了在 1.2.3 和 1.1.16 中关于关联代理对象的回归，修订了在计算关联代理的“拥有类”时默认选择当前类的方法，如果代理对象与映射类没有直接关联，例如一个 mixin。

    参考：[#4185](https://www.sqlalchemy.org/trac/ticket/4185)

## 1.1.16

发布日期：2018 年 2 月 16 日

### orm

+   **[orm] [bug]**

    修复了在父对象已被删除但相关对象尚未删除时 post_update 功能会发出 UPDATE 的问题。这个问题已经存在很长时间，但自 1.2 版本开始为 post_update 断言匹配的行数，这导致了错误。

    参考：[#4187](https://www.sqlalchemy.org/trac/ticket/4187)

+   **[orm] [bug]**

    修复了由于修复问题 [#4116](https://www.sqlalchemy.org/trac/ticket/4116) 而引起的回归，影响版本 1.2.2 和 1.1.15，导致在某些声明性 mixin/继承情况下以及如果关联代理从未映射的类中访问时，`AssociationProxy` 的“拥有类”被错误地计算为 `NoneType` 类。现在，“找到所有者”逻辑已被替换为一个深入的程序，该程序搜索分配给类或子类的完整映射器层次结构，以确定正确（我们希望）的匹配；如果找不到匹配项，则不会分配所有者。如果代理用于未映射的实例，则现在会引发异常。

    参考：[#4185](https://www.sqlalchemy.org/trac/ticket/4185)

+   **[orm] [bug]**

    修复了在嵌套或子事务回滚期间被清除的对象，同时其主键被改变，会导致该对象未能正确从会话中移除的错误，从而导致后续使用会话时出现问题。

    参考：[#4151](https://www.sqlalchemy.org/trac/ticket/4151)

### sql

+   **[sql] [bug]**

    在 `sqlalchemy.` 和 `sqlalchemy.sql.` 命名空间中添加了 `nullsfirst()` 和 `nullslast()` 作为顶级导入。感谢 Lele Gaifax 提交的拉取请求。

+   **[sql] [bug]**

    修复了在 `Insert.values()` 中使用“multi-values”格式与 `Column` 对象作为键而不是字符串时会失败的 bug。感谢 Aubrey Stark-Toller 提交的拉取请求。

    参考：[#4162](https://www.sqlalchemy.org/trac/ticket/4162)

### postgresql

+   **[postgresql] [bug]**

    将“SSL SYSCALL error: Operation timed out”添加到触发 psycopg2 驱动程序“断开”场景的消息列表中。感谢 André Cruz 的拉取请求。

+   **[postgresql] [错误]**

    将“TRUNCATE”添加到 PostgreSQL 方言接受的关键字列表中，作为“自动提交”触发关键字。感谢 Jacob Hayes 的拉取请求。

### mysql

+   **[mysql] [错误]**

    修复了 MySQL 的“concat”和“match”运算符未能将 kwargs 传播到左右表达式的错误，导致编译器选项（如“literal_binds”）失败。

    参考：[#4136](https://www.sqlalchemy.org/trac/ticket/4136)

### 杂项

+   **[错误] [池]**

    修复了一个相当严重的连接池错误，即在由于用户定义的`DisconnectionError`或由于 1.2 版本发布的“pre_ping”功能导致刷新后获取的连接，如果连接由 weakref 清理（例如前端对象被垃圾回收）后返回到池中，该连接将不会被正确重置；弱引用仍将指向先前失效的 DBAPI 连接，而该连接将错误地调用重置操作。这将导致日志中的堆栈跟踪和连接被检入池中而未被重置，这可能导致锁定问题。

    参考：[#4184](https://www.sqlalchemy.org/trac/ticket/4184)

### orm

+   **[orm] [错误]**

    修复了在父对象已被删除但依赖对象未被删除时，post_update 功能会发出 UPDATE 的问题。这个问题已经存在很长时间，但自 1.2 版本开始，现在对于 post_update 会断言匹配的行数，这会引发错误。

    参考：[#4187](https://www.sqlalchemy.org/trac/ticket/4187)

+   **[orm] [错误]**

    修复了由于问题[#4116](https://www.sqlalchemy.org/trac/ticket/4116)的修复引起的回归，影响了 1.2.2 版本以及 1.1.15 版本，导致在某些声明性 mixin/继承情况下，`AssociationProxy`的“拥有类”被错误地计算为`NoneType`类，以及如果关联代理从未映射的类中访问。现在，“找到所有者”的逻辑已被替换为一个深入的例程，该例程通过搜索分配给类或子类的完整映射器层次结构来确定正确（我们希望）的匹配；如果找不到匹配项，则不会分配所有者。如果代理针对未映射的实例使用，则现在会引发异常。

    参考：[#4185](https://www.sqlalchemy.org/trac/ticket/4185)

+   **[orm] [错误]**

    修复了在嵌套或子事务回滚期间被清除的对象，同时其主键被突变，将不会被正确地从会话中移除的错误，导致在使用会话时出现后续问题。

    参考：[#4151](https://www.sqlalchemy.org/trac/ticket/4151)

### sql

+   **[sql] [bug]**

    将`nullsfirst()`和`nullslast()`作为`sqlalchemy.`和`sqlalchemy.sql.`命名空间中的顶级导入添加。感谢 Lele Gaifax 的拉取请求。

+   **[sql] [bug]**

    修复了在`Insert.values()`中使用“multi-values”格式与`Column`对象作为键而不是字符串时会失败的 bug。感谢 Aubrey Stark-Toller 的拉取请求。

    参考：[#4162](https://www.sqlalchemy.org/trac/ticket/4162)

### postgresql

+   **[postgresql] [bug]**

    将“SSL SYSCALL error: Operation timed out”添加到了 psycopg2 驱动程序的“断开连接”场景触发消息列表中。感谢 André Cruz 的拉取请求。

+   **[postgresql] [bug]**

    将“TRUNCATE”添加到了 PostgreSQL 方言接受的关键字列表中，作为“autocommit”触发关键字。感谢 Jacob Hayes 的拉取请求。

### mysql

+   **[mysql] [bug]**

    修复了 MySQL 的“concat”和“match”运算符未能将 kwargs 传播到左右表达式的 bug，导致编译器选项如“literal_binds”失败。

    参考：[#4136](https://www.sqlalchemy.org/trac/ticket/4136)

### 杂项

+   **[bug] [pool]**

    修复了一个相当严重的连接池 bug，当一个连接在由用户定义的`DisconnectionError`或由 1.2 版本发布的“pre_ping”功能刷新后被获取时，如果连接被弱引用清理（例如前端对象被垃圾回收），则连接不会被正确重置；弱引用仍然会指向先前失效的 DBAPI 连接，导致重置操作错误地被调用。这将导致日志中的堆栈跟踪和连接被检入池中而未被重置，可能会导致锁定问题。

    参考：[#4184](https://www.sqlalchemy.org/trac/ticket/4184)

## 1.1.15

发布日期：2017 年 11 月 3 日

### orm

+   **[orm] [bug] [ext]**

    修复了关联代理在首次调用时会不经意地将自身链接到一个`AliasedClass`对象的 bug，如果首次调用时将`AliasedClass`作为父级，会导致后续使用时出现错误。

    参考：[#4116](https://www.sqlalchemy.org/trac/ticket/4116)

+   **[orm] [bug]**

    修复了一个 bug，ORM 关系会警告存在冲突的同步目标（例如，两个关系都写入同一列）对于继承层次结构中的兄弟类，这两个关系实际上永远不会在写入时发生冲突。

    参考：[#4078](https://www.sqlalchemy.org/trac/ticket/4078)

+   **[orm] [bug]**

    修复了针对单表继承实体使用相关选择时，在外部查询中无法正确呈现的 bug，因为单一继承鉴别器标准的调整不当地重新应用于外部查询。

    参考：[#4103](https://www.sqlalchemy.org/trac/ticket/4103)

### orm declarative

+   **[orm] [declarative] [bug]**

    修复了一个 bug，其中描述符，即基于`AbstractConcreteBase`的映射列或关系在刷新操作期间被引用，导致错误，因为该属性未映射为映射器属性。如果类未在其映射器中包含“concrete=True”，则类似的问题也可能出现，例如`AbstractConcreteBase`添加的“type”列，但此处的检查也应防止该场景引起问题。

    参考：[#4124](https://www.sqlalchemy.org/trac/ticket/4124)

### sql

+   **[sql] [bug]**

    修复了一个 bug，当参数为元组时，`ColumnDefault`的`__repr__`会失败。感谢 Nicolas Caniart 的拉取请求。

    参考：[#4126](https://www.sqlalchemy.org/trac/ticket/4126)

+   **[sql] [bug]**

    修复了一个 bug，最近添加的`ColumnOperators.any_()`和`ColumnOperators.all_()`方法在作为方法调用时无法正常工作，而不是使用独立函数`any_()`和`all_()`。还为这些相对晦涩的 SQL 操作符添加了文档示例。

    参考：[#4093](https://www.sqlalchemy.org/trac/ticket/4093)

### postgresql

+   **[postgresql] [bug]**

    进一步修复了与 COLLATE 一起使用`ARRAY`类的问题，因为在[#4006](https://www.sqlalchemy.org/trac/ticket/4006)中进行的修复未能适应多维数组。

    引用：[#4006](https://www.sqlalchemy.org/trac/ticket/4006)

+   **[postgresql] [bug]**

    修复了 `array_agg` 函数中的错误，其中传递已经是 `ARRAY` 类型的参数，例如 PostgreSQL 的 `array` 构造，会产生 `ValueError`，因为函数尝试嵌套数组。

    引用：[#4107](https://www.sqlalchemy.org/trac/ticket/4107)

+   **[postgresql] [bug]**

    修复了 PostgreSQL `Insert.on_conflict_do_update()` 中的错误，该错误会阻止插入语句作为 CTE（例如通过 `Insert.cte()`）在另一个语句中使用。

    引用：[#4074](https://www.sqlalchemy.org/trac/ticket/4074)

### mysql

+   **[mysql] [bug]**

    当检测到 MariaDB 10.2.8 或更早版本的 10.2 系列时发出警告，因为这些版本中的 CHECK 约束存在重大问题，这些问题已在 10.2.9 中解决。

    请注意，此更改日志消息未随 SQLAlchemy 1.2.0b3 发布，而是事后添加的。

    引用：[#4097](https://www.sqlalchemy.org/trac/ticket/4097)

+   **[mysql] [bug]**

    MySQL 5.7.20 现在会警告使用 @tx_isolation 变量；现在执行版本检查并使用 @transaction_isolation 以防止此警告。

    引用：[#4120](https://www.sqlalchemy.org/trac/ticket/4120)

+   **[mysql] [bug]**

    修复了在 MariaDB 10.2 系列中 CURRENT_TIMESTAMP 由于语法更改而无法正确反映的问题，其中该函数现在表示为 `current_timestamp()`。

    引用：[#4096](https://www.sqlalchemy.org/trac/ticket/4096)

+   **[mysql] [bug]**

    MariaDB 10.2 现在支持 CHECK 约束（警告：由于上游问题，请使用版本 10.2.9 或更高版本）。反射现在在 `SHOW CREATE TABLE` 输出中考虑这些 CHECK 约束。

    引用：[#4098](https://www.sqlalchemy.org/trac/ticket/4098)

### sqlite

+   **[sqlite] [bug]**

    修复了 SQLite CHECK 约束反射失败的错误，如果引用的表位于远程模式中，例如在 SQLite 中由 ATTACH 引用的远程数据库。

    引用：[#4099](https://www.sqlalchemy.org/trac/ticket/4099)

### mssql

+   **[mssql] [bug]**

    为 SQL Server 的 PyODBC 方言添加了完整的“连接关闭”异常代码范围，包括 ‘08S01’、‘01002’、‘08003’、‘08007’、‘08S02’、‘08001’、‘HYT00’、‘HY010’。以前只覆盖了 ‘08S01’。

    引用：[#4095](https://www.sqlalchemy.org/trac/ticket/4095)

### orm

+   **[orm] [bug] [ext]**

    修复了当关联代理在首次调用时不经意地将自身链接到一个`AliasedClass`对象时，如果首次调用时`AliasedClass`作为父级，会在后续使用时导致错误的 bug。

    参考：[#4116](https://www.sqlalchemy.org/trac/ticket/4116)

+   **[orm] [bug]**

    修复了 ORM 关系在继承层次结构中的兄弟类中可能会发出警告，提示存在冲突的同步目标（例如，两个关系都写入同一列），而实际上这两个关系在写入时永远不会发生冲突。

    参考：[#4078](https://www.sqlalchemy.org/trac/ticket/4078)

+   **[orm] [bug]**

    修复了针对单表继承实体使用相关子查询会导致外部查询无法正确渲染的 bug，因为不恰当地重新应用了单一继承鉴别器条件以重新应用条件到外部查询。

    参考：[#4103](https://www.sqlalchemy.org/trac/ticket/4103)

### ORM 声明式

+   **[orm] [declarative] [bug]**

    修复了在刷新操作期间引用描述符（即基于`AbstractConcreteBase`的映射列或关系）会导致错误的 bug，因为该属性未映射为映射器属性。如果类未在其映射器中包含“concrete=True”，则类似的问题也可能出现在其他属性上，例如`AbstractConcreteBase`添加的“type”列，但此处的检查也应防止该场景引起问题。

    参考：[#4124](https://www.sqlalchemy.org/trac/ticket/4124)

### SQL

+   **[sql] [bug]**

    修复了当`ColumnDefault`的`__repr__`参数为元组时会失败的 bug。感谢 Nicolas Caniart 的拉取请求。

    参考：[#4126](https://www.sqlalchemy.org/trac/ticket/4126)

+   **[sql] [bug]**

    修复了最近添加的 `ColumnOperators.any_()` 和 `ColumnOperators.all_()` 方法在作为方法调用时无法正常工作的错误，而不是使用独立函数 `any_()` 和 `all_()`。还为这些相对晦涩的 SQL 操作符添加了文档示例。

    参考：[#4093](https://www.sqlalchemy.org/trac/ticket/4093)

### postgresql

+   **[postgresql] [bug]**

    进一步修复了与 COLLATE 结合使用的 `ARRAY` 类的问题，因为在 [#4006](https://www.sqlalchemy.org/trac/ticket/4006) 中进行的修复未能考虑到多维数组。

    参考：[#4006](https://www.sqlalchemy.org/trac/ticket/4006)

+   **[postgresql] [bug]**

    修复了`array_agg`函数中的错误，当传递一个已经是`ARRAY`类型的参数时，比如一个 PostgreSQL `array` 构造，会产生`ValueError`，因为函数尝试嵌套数组。

    参考：[#4107](https://www.sqlalchemy.org/trac/ticket/4107)

+   **[postgresql] [bug]**

    修复了 PostgreSQL `Insert.on_conflict_do_update()` 中的错误，该错误会阻止插入语句作为 CTE 使用，例如通过 `Insert.cte()` 在另一个语句中。

    参考：[#4074](https://www.sqlalchemy.org/trac/ticket/4074)

### mysql

+   **[mysql] [bug]**

    当检测到 MariaDB 10.2.8 或更早版本的 10.2 系列时发出警告，因为这些版本中的 CHECK 约束存在重大问题，这些问题在 10.2.9 中已解决。

    请注意，此更改日志消息未随 SQLAlchemy 1.2.0b3 发布，而是事后添加的。

    参考：[#4097](https://www.sqlalchemy.org/trac/ticket/4097)

+   **[mysql] [bug]**

    MySQL 5.7.20 现在警告使用 @tx_isolation 变量；现在执行版本检查并使用 @transaction_isolation 代替以防止此警告。

    参考：[#4120](https://www.sqlalchemy.org/trac/ticket/4120)

+   **[mysql] [bug]**

    修复了在 MariaDB 10.2 系列中，CURRENT_TIMESTAMP 无法正确反映的问题，因为语法发生了变化，现在该函数表示为 `current_timestamp()`。

    参考：[#4096](https://www.sqlalchemy.org/trac/ticket/4096)

+   **[mysql] [bug]**

    MariaDB 10.2 现在支持 CHECK 约束（警告：由于上游问题，请使用版本 10.2.9 或更高版本，详见[#4097](https://www.sqlalchemy.org/trac/ticket/4097)）。反射现在在存在这些 CHECK 约束时考虑这些 CHECK 约束，当它们出现在 `SHOW CREATE TABLE` 输出中时。

    参考：[#4098](https://www.sqlalchemy.org/trac/ticket/4098)

### sqlite

+   **[sqlite] [bug]**

    修复了 SQLite CHECK 约束反射的 bug，如果引用的表位于远程模式中，例如 SQLite 中由 ATTACH 引用的远程数据库，则会失败。

    参考：[#4099](https://www.sqlalchemy.org/trac/ticket/4099)

### mssql

+   **[mssql] [bug]**

    为 SQL Server 的 PyODBC 方言添加了一整套“连接关闭”异常代码，包括 ‘08S01’、‘01002’、‘08003’、‘08007’、‘08S02’、‘08001’、‘HYT00’、‘HY010’。之前只覆盖了 ‘08S01’。

    参考：[#4095](https://www.sqlalchemy.org/trac/ticket/4095)

## 1.1.14

发布日期：2017 年 9 月 5 日

### orm

+   **[orm] [bug]**

    在`Session.merge()`中修复了一个 bug，与[#4030](https://www.sqlalchemy.org/trac/ticket/4030)类似，其中对于身份映射中的目标对象的内部检查可能导致错误，如果在合并过程实际检索对象之前立即对其进行垃圾回收。

    参考：[#4069](https://www.sqlalchemy.org/trac/ticket/4069)

+   **[orm] [bug]**

    修复了当一个 `undefer_group()` 选项不会被识别，如果它延伸自使用连接式急加载加载的关系。此外，由于该 bug 导致执行了过多的工作，Python 函数调用数量在结果集列的初始计算中也提高了 20%，补充了 [#3915](https://www.sqlalchemy.org/trac/ticket/3915) 的连接式急加载改进。

    参考：[#4048](https://www.sqlalchemy.org/trac/ticket/4048)

+   **[orm] [bug]**

    在 ORM 身份映射中修复了竞争条件，该条件会导致对象在加载操作期间被不适当地移除，从而导致重复对象标识出现，特别是在涉及对象去重的连接急加载下。此问题特定于弱引用的垃圾收集，并且仅在 PyPy 解释器下观察到。

    参考：[#4068](https://www.sqlalchemy.org/trac/ticket/4068)

+   **[orm] [bug]**

    在`Session.merge()`中修复了一个 bug，其中集合中的对象的主键属性设置为 `None`，对于通常是自动增量的键，将被视为数据库持久化键的一部分，导致在内部去重处理过程中实际上只插入一个对象到数据库中。

    参考：[#4056](https://www.sqlalchemy.org/trac/ticket/4056)

+   **[orm] [bug]**

    当针对不是`MapperProperty`的属性（例如关联代理）使用`synonym()`时，会引发`InvalidRequestError`。以前，尝试定位不存在的属性会导致递归溢出。

    参考：[#4067](https://www.sqlalchemy.org/trac/ticket/4067)

### sql

+   **[sql] [bug]**

    修改了窗口函数的范围规范，允许范围中出现两个相同的 PRECEDING 或 FOLLOWING 关键字，通过允许范围的左侧为正数，右侧为负数，例如 (1, 3) 表示“1 FOLLOWING AND 3 FOLLOWING”。

    参考：[#4053](https://www.sqlalchemy.org/trac/ticket/4053)

### orm

+   **[orm] [bug]**

    修复了`Session.merge()`中的一个 bug，与[#4030](https://www.sqlalchemy.org/trac/ticket/4030)类似，其中对于身份映射中的目标对象的内部检查，如果在合并过程实际检索对象之前立即被垃圾回收，可能会导致错误。

    参考：[#4069](https://www.sqlalchemy.org/trac/ticket/4069)

+   **[orm] [bug]**

    修复了一个 bug，其中一个`undefer_group()`选项如果从使用连接式急加载加载的关系扩展，则不会被识别。此外，由于该 bug 导致执行过多的工作，因此在结果集列的初始计算中，Python 函数调用次数也提高了 20%，这与[#3915](https://www.sqlalchemy.org/trac/ticket/3915)的连接式急加载改进相辅相成。

    参考：[#4048](https://www.sqlalchemy.org/trac/ticket/4048)

+   **[orm] [bug]**

    修复了 ORM 身份映射中的竞争条件，导致在加载操作期间不适当地移除对象，从而导致重复的对象标识出现，特别是在涉及对象去重的连接式急加载下。该问题特定于弱引用的垃圾回收，并且仅在 PyPy 解释器下观察到。

    参考：[#4068](https://www.sqlalchemy.org/trac/ticket/4068)

+   **[orm] [bug]**

    修复了`Session.merge()`中的一个 bug，其中集合中的对象的主键属性设置为`None`，而该属性通常是自动递增的，会被视为数据库持久化键的一部分，导致内部去重过程中实际上只有一个对象被插入到数据库中。

    参考：[#4056](https://www.sqlalchemy.org/trac/ticket/4056)

+   **[orm] [bug]**

    当对不是`MapperProperty`的属性使用`synonym()`时，会引发`InvalidRequestError`。以前，尝试定位不存在的属性会导致递归溢出。

    参考：[#4067](https://www.sqlalchemy.org/trac/ticket/4067)

### sql

+   **[sql] [bug]**

    修改了窗口函数的范围规范，允许在范围内使用两个相同的 PRECEDING 或 FOLLOWING 关键字，通过允许范围的左侧为正数，右侧为负数，例如(1, 3)表示“1 FOLLOWING AND 3 FOLLOWING”。

    参考：[#4053](https://www.sqlalchemy.org/trac/ticket/4053)

## 1.1.13

发布日期：2017 年 8 月 3 日

### oracle

+   **[oracle] [performance] [bug] [py2k]**

    修复了由于修复[#3937](https://www.sqlalchemy.org/trac/ticket/3937)引起的性能回归，其中 cx_Oracle 版本 5.3 删除了其命名空间中的`.UNICODE`符号，这被解释为 cx_Oracle 的“WITH_UNICODE”模式被无条件地打开，从而在 SQLAlchemy 一侧调用函数，无条件地将所有字符串转换为 unicode，并导致性能影响。实际上，根据 cx_Oracle 的作者，“WITH_UNICODE”模式自 5.1 版本起已完全移除，因此昂贵的 unicode 转换函数不再必要，如果在 Python 2 下检测到 cx_Oracle 5.1 或更高版本，则会禁用这些函数。在[#3937](https://www.sqlalchemy.org/trac/ticket/3937)中删除的针对“WITH_UNICODE”模式的警告也已恢复。

    此更改也被**回溯**到：1.0.19

    参考：[#4035](https://www.sqlalchemy.org/trac/ticket/4035)

### oracle

+   **[oracle] [performance] [bug] [py2k]**

    修复了由于修复[#3937](https://www.sqlalchemy.org/trac/ticket/3937)引起的性能回归，其中 cx_Oracle 版本 5.3 删除了其命名空间中的`.UNICODE`符号，这被解释为 cx_Oracle 的“WITH_UNICODE”模式被无条件地打开，从而在 SQLAlchemy 一侧调用函数，无条件地将所有字符串转换为 unicode，并导致性能影响。实际上，根据 cx_Oracle 的作者，“WITH_UNICODE”模式自 5.1 版本起已完全移除，因此昂贵的 unicode 转换函数不再必要，如果在 Python 2 下检测到 cx_Oracle 5.1 或更高版本，则会禁用这些函数。在[#3937](https://www.sqlalchemy.org/trac/ticket/3937)中删除的针对“WITH_UNICODE”模式的警告也已恢复。

    此更改也被**回溯**到：1.0.19

    参考：[#4035](https://www.sqlalchemy.org/trac/ticket/4035)

## 1.1.12

发布日期：2017 年 7 月 24 日

### orm

+   **[orm] [bug]**

    修复了从 1.1.11 开始的回归，当向包含具有子查询加载关系的实体的查询中添加额外的非实体列时，由于 1.1.11 中添加的检查导致失败，这是由于[#4011](https://www.sqlalchemy.org/trac/ticket/4011)的结果。

    参考：[#4033](https://www.sqlalchemy.org/trac/ticket/4033)

+   **[orm] [错误]**

    修复了 1.1 版本中添加的涉及 JSON NULL 评估逻辑的错误，该逻辑作为[#3514](https://www.sqlalchemy.org/trac/ticket/3514)的一部分添加，其中逻辑不会适应与`Column`不同命名的 ORM 映射属性。

    参考：[#4031](https://www.sqlalchemy.org/trac/ticket/4031)

+   **[orm] [错误]**

    在`WeakInstanceDict`中的所有方法中添加了`KeyError`检查，其中在检查`key in dict`之后紧接着对该键进行索引访问，以防止在负载下可能会将键从字典中移除的垃圾回收竞争导致代码假定其存在后，非常少见地引发`KeyError`。

    参考：[#4030](https://www.sqlalchemy.org/trac/ticket/4030)

### oracle

+   **[oracle] [功能] [postgresql]**

    向`Sequence`添加了新关键字`Sequence.cache`和`Sequence.order`，以允许渲染 Oracle 和 PostgreSQL 理解的 CACHE 参数，以及 Oracle 理解的 ORDER 参数。感谢 David Moore 的拉取请求。

### 测试

+   **[测试] [错误] [py3k]**

    修复了与 Python 3.6.2 的更改不兼容的测试固定问题，涉及上下文管理器的更改。

    此更改也**回溯**到：1.0.18

    参考：[#4034](https://www.sqlalchemy.org/trac/ticket/4034)

### orm

+   **[orm] [错误]**

    修复了从 1.1.11 开始的回归，当向包含具有子查询加载关系的实体的查询中添加额外的非实体列时，由于 1.1.11 中添加的检查导致失败，这是由于[#4011](https://www.sqlalchemy.org/trac/ticket/4011)的结果。

    参考：[#4033](https://www.sqlalchemy.org/trac/ticket/4033)

+   **[orm] [错误]**

    修复了 1.1 版本中添加的涉及 JSON NULL 评估逻辑的错误，该逻辑作为[#3514](https://www.sqlalchemy.org/trac/ticket/3514)的一部分添加，其中逻辑不会适应与`Column`不同命名的 ORM 映射属性。

    参考：[#4031](https://www.sqlalchemy.org/trac/ticket/4031)

+   **[orm] [错误]**

    在`WeakInstanceDict`中的所有方法中添加了`KeyError`检查，其中在检查`key in dict`之后紧接着对该键进行索引访问，以防止在负载下可能会将键从字典中移除的垃圾回收竞争导致代码假定其存在后，非常少见地引发`KeyError`。

    参考：[#4030](https://www.sqlalchemy.org/trac/ticket/4030)

### oracle

+   **[oracle] [feature] [postgresql]**

    添加了新关键字`Sequence.cache`和`Sequence.order`到`Sequence`，以允许 Oracle 和 PostgreSQL 理解的 CACHE 参数的呈现，以及 Oracle 理解的 ORDER 参数。拉取请求由 David Moore 提供。

### tests

+   **[tests] [bug] [py3k]**

    修复了与 Python 3.6.2 的更改不兼容的测试固定装置中的问题，涉及上下文管理器。

    此更改也**回溯**到：1.0.18

    参考：[#4034](https://www.sqlalchemy.org/trac/ticket/4034)

## 1.1.11

发布日期：2017 年 6 月 19 日星期一

### orm

+   **[orm] [bug]**

    修复了子查询急加载中的问题，该问题继续自[#2699](https://www.sqlalchemy.org/trac/ticket/2699)、[#3106](https://www.sqlalchemy.org/trac/ticket/3106)、[#3893](https://www.sqlalchemy.org/trac/ticket/3893)修复的系列问题，涉及到“子查询”在从连接的继承子类开始，然后对基类的关系进行子查询急加载时包含正确的 FROM 子句，同时查询还包括对子类的条件。之前票证中的修复未考虑到从第一级更深层次加载更多的子查询操作，因此修复进一步泛化。

    参考：[#4011](https://www.sqlalchemy.org/trac/ticket/4011)

### sql

+   **[sql] [bug]**

    修复了在`WithinGroup`结构迭代期间可能发生的 AttributeError。

    参考：[#4012](https://www.sqlalchemy.org/trac/ticket/4012)

### postgresql

+   **[postgresql] [bug]**

    继续处理正确处理 1.1.8 中发布的 PostgreSQL 版本字符串“10devel”的修复，增加了一个正则表达式版本以处理形式为“10beta1”的版本字符串。虽然 PostgreSQL 现在提供更好的方法来获取这些信息，但至少在 1.1.x 中我们仍然坚持使用正则表达式，以最小化与旧版或替代 PostgreSQL 数据库的兼容性风险。

    参考：[#4005](https://www.sqlalchemy.org/trac/ticket/4005)

+   **[postgresql] [bug]**

    修复了使用带有排序的字符串类型的`ARRAY`在 CREATE TABLE 中未能生成正确语法的错误。

    参考：[#4006](https://www.sqlalchemy.org/trac/ticket/4006)

### mysql

+   **[mysql] [bug]**

    MySQL 5.7 引入了对“SHOW VARIABLES”命令的权限限制；MySQL 方言现在将处理当 SHOW 返回没有行时的情况，特别是对于 SQL_MODE 的初始获取，并将发出警告，提示用户权限应该被修改以允许该行存在。

    参考：[#4007](https://www.sqlalchemy.org/trac/ticket/4007)

### mssql

+   **[mssql] [bug]**

    修复了在使用 Azure 数据仓库时必须从不同视图获取 SQL Server 事务隔离的错误，现在查询将尝试针对两个视图，如果失败继续，则无条件地引发 NotImplemented，以提供对未来新 SQL Server 版本中任意 API 更改的最佳弹性。

    参考：[#3994](https://www.sqlalchemy.org/trac/ticket/3994)

+   **[mssql] [bug]**

    在 SQL Server 方言中添加了一个占位符类型`XML`，以便包含此类型的反射表可以重新呈现为 CREATE TABLE。该类型没有特殊的往返行为，也不支持当前支持额外的限定参数。

    参考：[#3973](https://www.sqlalchemy.org/trac/ticket/3973)

### oracle

+   **[oracle] [bug]**

    当使用 cx_Oracle 的版本 6.0b1 或更高版本时，完全删除了 cx_Oracle 的两阶段事务支持。在任何情况下，两阶段功能在 cx_Oracle 5.x 下历史上从未可用过，而 cx_Oracle 6.x 已经删除了此功能依赖的连接级“twophase”标志。

    参考：[#3997](https://www.sqlalchemy.org/trac/ticket/3997)

### orm

+   **[orm] [bug]**

    修复了子查询预加载的问题，这是继[#2699](https://www.sqlalchemy.org/trac/ticket/2699)、[#3106](https://www.sqlalchemy.org/trac/ticket/3106)、[#3893](https://www.sqlalchemy.org/trac/ticket/3893)修复的一系列问题的延续，涉及到当从一个连接的继承子类开始，然后对基类的关系进行子查询预加载时，“子查询”包含正确的 FROM 子句，同时查询还包括对子类的条件。之前票证中的修复没有考虑到从第一级更深层次加载更多的 subqueryload 操作，因此修复已进一步泛化。

    参考：[#4011](https://www.sqlalchemy.org/trac/ticket/4011)

### sql

+   **[sql] [bug]**

    修复了在`WithinGroup`结构迭代期间可能发生的 AttributeError。

    参考：[#4012](https://www.sqlalchemy.org/trac/ticket/4012)

### postgresql

+   **[postgresql] [bug]**

    继续修复正确处理 PostgreSQL 版本字符串“10devel”（在 1.1.8 中发布）的问题，增加了一个正则表达式来处理形式为“10beta1”的版本字符串。虽然 PostgreSQL 现在提供了更好的获取此信息的方法，但至少在 1.1.x 中，我们仍然坚持使用正则表达式，以最大程度地减少与旧版或其他 PostgreSQL 数据库的兼容性风险。

    参考：[#4005](https://www.sqlalchemy.org/trac/ticket/4005)

+   **[postgresql] [bug]**

    修复了一个 bug，在使用带有排序规则的字符串类型的 `ARRAY` 时，无法在 CREATE TABLE 中生成正确的语法。

    参考：[#4006](https://www.sqlalchemy.org/trac/ticket/4006)

### mysql

+   **[mysql] [bug]**

    MySQL 5.7 引入了对“SHOW VARIABLES”命令的权限限制；MySQL 方言现在将处理当 SHOW 不返回任何行时的情况，特别是对于 SQL_MODE 的初始获取，并将发出警告，提示用户权限应该被修改以允许该行存在。

    参考：[#4007](https://www.sqlalchemy.org/trac/ticket/4007)

### mssql

+   **[mssql] [bug]**

    修复了一个 bug，在使用 Azure 数据仓库时，SQL Server 事务隔离必须从不同的视图中获取，现在查询将尝试针对两个视图执行，如果失败继续提供最佳的弹性，以防止未来新 SQL Server 版本中的任意 API 更改引起的问题。

    参考：[#3994](https://www.sqlalchemy.org/trac/ticket/3994)

+   **[mssql] [bug]**

    向 SQL Server 方言添加了一个占位符类型 `XML`，以便包含此类型的反射表可以重新呈现为 CREATE TABLE。该类型没有特殊的往返行为，也不支持额外的限定参数。

    参考：[#3973](https://www.sqlalchemy.org/trac/ticket/3973)

### oracle

+   **[oracle] [bug]**

    当使用 cx_Oracle 版本 6.0b1 或更高版本的 DBAPI 时，完全移除了 cx_Oracle 的两阶段事务支持。历史上，在任何情况下，cx_Oracle 5.x 下的两阶段功能从未可用，而 cx_Oracle 6.x 已经移除了此功能所依赖的连接级“twophase”标志。

    参考：[#3997](https://www.sqlalchemy.org/trac/ticket/3997)

## 1.1.10

发布日期：2017 年 5 月 19 日星期五

### orm

+   **[orm] [bug]**

    修复了一个 bug，当级联操作（如“delete-orphan”等）无法定位到与继承关系中本身是子类本地关系的对象相关联的对象时，导致操作无法执行。

    参考：[#3986](https://www.sqlalchemy.org/trac/ticket/3986)

### schema

+   **[schema] [bug]**

    如果创建一个具有不匹配的“本地”和“远程”列数量的`ForeignKeyConstraint`对象，将引发`ArgumentError`，否则会导致约束的内部状态不正确。请注意，这也会影响方言的反射过程产生不匹配列集的情况。

    参考：[#3949](https://www.sqlalchemy.org/trac/ticket/3949)

### postgresql

+   **[postgresql] [bug]**

    为 GRANT、REVOKE 关键字添加了“autocommit”支持。感谢 Jacob Hayes 的拉取请求。

### mysql

+   **[mysql] [bug]**

    移除了对 UTC_TIMESTAMP MySQL 函数的古老且不必要的拦截，这会妨碍使用带参数的函数。

    参考：[#3966](https://www.sqlalchemy.org/trac/ticket/3966)

+   **[mysql] [bug]**

    修复了 MySQL 方言中关于在渲染 CREATE TABLE 时与 PARTITION 选项一起渲染表选项的错误。PARTITION 相关选项需要跟随表选项，而以前这种顺序没有得到执行。

    参考：[#3961](https://www.sqlalchemy.org/trac/ticket/3961)

### oracle

+   **[oracle] [bug]**

    修复了 cx_Oracle 方言中版本字符串解析对于 cx_Oracle 版本 6.0b1 会失败的错误，原因是“b”字符。现在版本字符串解析通过正则表达式而不是简单的拆分。

    参考：[#3975](https://www.sqlalchemy.org/trac/ticket/3975)

### misc

+   **[bug] [ext]**

    在声明类被垃圾回收并且新的 automap prepare()操作同时进行时，防止将“None”作为类进行测试，非常罕见地会在 gc 后未完全处理的弱引用上命中。

    参考：[#3980](https://www.sqlalchemy.org/trac/ticket/3980)

### orm

+   **[orm] [bug]**

    修复了一个错误，即“delete-orphan”（以及其他一些）级联操作无法定位到与继承关系中的子类本地关系链接的对象，从而导致操作无法执行。

    参考：[#3986](https://www.sqlalchemy.org/trac/ticket/3986)

### schema

+   **[schema] [bug]**

    如果创建一个具有不匹配的“本地”和“远程”列数量的`ForeignKeyConstraint`对象，将引发`ArgumentError`，否则会导致约束的内部状态不正确。请注意，这也会影响方言的反射过程产生不匹配列集的情况。

    参考：[#3949](https://www.sqlalchemy.org/trac/ticket/3949)

### postgresql

+   **[postgresql] [bug]**

    为 GRANT、REVOKE 关键字添加了“autocommit”支持。感谢 Jacob Hayes 的 Pull 请求。

### MySQL

+   **[mysql] [bug]**

    移除了对 UTC_TIMESTAMP MySQL 函数的古老且不必要的拦截，这妨碍了将其与参数一起使用。

    参考：[#3966](https://www.sqlalchemy.org/trac/ticket/3966)

+   **[mysql] [bug]**

    修复了 MySQL 方言中关于在渲染 CREATE TABLE 时与 PARTITION 选项一起渲染表选项的 bug。PARTITION 相关选项需要跟随表选项，而以前未强制执行此顺序。

    参考：[#3961](https://www.sqlalchemy.org/trac/ticket/3961)

### Oracle

+   **[oracle] [bug]**

    修复了 cx_Oracle 方言中版本字符串解析在 cx_Oracle 版本 6.0b1 中由于“b”字符而失败的 bug。现在版本字符串解析通过正则表达式而不是简单的分割。

    参考：[#3975](https://www.sqlalchemy.org/trac/ticket/3975)

### 杂项

+   **[bug] [ext]**

    防止在声明类被垃圾回收并发生新的 automap prepare()操作时，测试“None”作为类的情况，非常少见地会在 gc 后仍未完全处理的 weakref 上命中。

    参考：[#3980](https://www.sqlalchemy.org/trac/ticket/3980)

## 1.1.9

发布日期：2017 年 4 月 4 日

### SQL

+   **[sql] [bug]**

    由于[#3859](https://www.sqlalchemy.org/trac/ticket/3859)中发布的 1.1.5 中的回归，根据`Variant`的“右侧”规则调整表达式的“右侧”评估，以尊重底层类型的“右侧”规则，导致在我们*确实*希望左侧类型直接传递到右侧，以便将绑定级规则应用于表达式参数时，`Variant`类型被不适当地丢失。

    参考：[#3952](https://www.sqlalchemy.org/trac/ticket/3952)

+   **[sql] [bug] [postgresql]**

    更改了`ResultProxy`的机制，无条件地延迟“autoclose”步骤，直到`Connection`完成对象的操作；在 PostgreSQL ON CONFLICT 与 RETURNING 返回零行的情况下，之前不存在的用例中，autoclose 会发生，导致以前无条件发生的 INSERT/UPDATE/DELETE 上的通常自动提交行为失败。

    参考：[#3955](https://www.sqlalchemy.org/trac/ticket/3955)

### 杂项

+   **[bug] [ext]**

    修复了在 1.1.8 版本中由于[#3950](https://www.sqlalchemy.org/trac/ticket/3950)导致的回归问题，即在“模式类型”或`TypeDecorator`的情况下对列类型的更深层搜索会在映射中还包含`column_property`时产生属性错误。

    参考：[#3956](https://www.sqlalchemy.org/trac/ticket/3956)

### sql

+   **[sql] [bug]**

    修复了在 1.1.5 版本中由于[#3859](https://www.sqlalchemy.org/trac/ticket/3859)导致的回归问题，即基于`Variant`的表达式的“右侧”评估调整以遵守基础类型的“右侧”规则，导致`Variant`类型在我们*确实*希望左侧类型直接传递到右侧以便将绑定级规则应用于表达式参数时不适当丢失。

    参考：[#3952](https://www.sqlalchemy.org/trac/ticket/3952)

+   **[sql] [bug] [postgresql]**

    更改了`ResultProxy`的机制，无条件地延迟“自动关闭”步骤，直到`Connection`完成对象处理；在 PostgreSQL ON CONFLICT with RETURNING 返回零行的情况下，之前不存在的用例中，自动关闭会发生，导致以前在 INSERT/UPDATE/DELETE 上无条件发生的自动提交行为失败。

    参考：[#3955](https://www.sqlalchemy.org/trac/ticket/3955)

### misc

+   **[bug] [ext]**

    修复了在 1.1.8 版本中由于[#3950](https://www.sqlalchemy.org/trac/ticket/3950)导致的回归问题，即在“模式类型”或`TypeDecorator`的情况下对列类型的更深层搜索会在映射中还包含`column_property`时产生属性错误。

    参考：[#3956](https://www.sqlalchemy.org/trac/ticket/3956)

## 1.1.8

发布日期：2017 年 3 月 31 日

### postgresql

+   **[postgresql] [bug]**

    增加了对解析 PostgreSQL 版本字符串的支持，例如“PostgreSQL 10devel”。感谢 Sean McCully 的拉取请求。

### misc

+   **[bug] [ext]**

    修复了`sqlalchemy.ext.mutable`中的错误，其中`Mutable.as_mutable()`方法不会跟踪使用 `TypeEngine.copy()` 复制的类型。这在 1.1 版本相比于 1.0 版本更加的退化，因为 `TypeDecorator` 类现在是 `SchemaEventTarget` 的子类之一，它表明当 `Column` 被复制时，类型也应该被复制。在使用混合或抽象类的情况下，这些复制是常见的。

    参考：[#3950](https://www.sqlalchemy.org/trac/ticket/3950)

+   **[bug] [ext]**

    新增对绑定参数的支持，例如通常通过`Query.params()`设置的参数，到`Result.count()`方法。以前，对参数的支持被省略了。感谢 Pat Deegan 的拉取请求。

### postgresql

+   **[postgresql] [bug]**

    新增对解析 PostgreSQL 版本字符串的支持，例如开发版本的 “PostgreSQL 10devel”。感谢 Sean McCully 的拉取请求。

### 杂项

+   **[bug] [ext]**

    修复了`sqlalchemy.ext.mutable`中的错误，其中`Mutable.as_mutable()`方法不会跟踪使用 `TypeEngine.copy()` 复制的类型。这在 1.1 版本相比于 1.0 版本更加的退化，因为 `TypeDecorator` 类现在是 `SchemaEventTarget` 的子类之一，它表明当 `Column` 被复制时，类型也应该被复制。在使用混合或抽象类的情况下，这些复制是常见的。

    参考：[#3950](https://www.sqlalchemy.org/trac/ticket/3950)

+   **[bug] [ext]**

    新增对绑定参数的支持，例如通常通过`Query.params()`设置的参数，到`Result.count()`方法。以前，对参数的支持被省略了。感谢 Pat Deegan 的拉取请求。

## 1.1.7

发布日期：2017 年 3 月 27 日

### orm

+   **[orm] [feature]**

    现在可以将`aliased()`构造传递给`Query.select_entity_from()`方法。实体将从由`aliased()`构造表示的可选择项中提取。这允许与`Query.select_entity_from()`一起使用`aliased()`的特殊选项，例如`aliased.adapt_on_names`。

    参考：[#3933](https://www.sqlalchemy.org/trac/ticket/3933)

+   **[orm] [bug]**

    修复了一个在多线程环境下可能发生的竞争条件，这是通过 [#3915](https://www.sqlalchemy.org/trac/ticket/3915) 添加的缓存导致的。一个内部的`Column`对象集合可能会不适当地在别名对象上重新生成，当尝试渲染 SQL 并收集结果时，会混淆一个连接的急切加载器，导致属性错误。现在，在别名对象被缓存和在线程之间共享之前，集合会提前生成并共享。

    参考：[#3947](https://www.sqlalchemy.org/trac/ticket/3947)

### engine

+   **[engine] [bug]**

    添加了一个异常处理程序，当“autorollback”功能的`Connection`本身引发异常时，将会警告“cause���异常在 Py2K 上。在 Py3K 中，两个异常自然地由解释器报告为一个发生在处理另一个异常时。这是继续处理回滚失败处理的一系列更改之一，上次访问是在 1.0.12 中的[#2696](https://www.sqlalchemy.org/trac/ticket/2696)。

    参考：[#3946](https://www.sqlalchemy.org/trac/ticket/3946)

### sql

+   **[sql] [bug] [postgresql]**

    增加了对`Variant`和`SchemaType`对象相互兼容的支持。也就是说，可以针对像`Enum`这样的类型创建一个变体，并且创建约束和/或数据库特定类型对象的指令将根据变体的方言映射正确传播。

    参考：[#2892](https://www.sqlalchemy.org/trac/ticket/2892)

+   **[sql] [bug]**

    修复了编译器中的一个错误，其中保存点的字符串标识符会被缓存在标识符引用字典中；由于这些标识符是任意的，如果一个`Connection`有大量未限定数量的保存点使用，以及如果保存点子句构造直接使用了未限定数量的保存点名称，那么可能会发生小内存泄漏。这个内存泄漏**不会**影响绝大多数情况，因为通常是针对每个事务或固定数量的事务使用渲染保存点名称的`Connection`，它从“1”开始的简单计数器。

    参考：[#3931](https://www.sqlalchemy.org/trac/ticket/3931)

+   **[sql] [bug]**

    修复了新“模式翻译”功能中的错误，其中在与列表达式一起渲染时，翻译的模式名称会被调用为别名名称；仅当源翻译名称为“None”时才会发生。现在，“模式翻译”功能仅对`SchemaItem`和`SchemaType`子类产生影响，也就是说，对应于数据库中可 DDL 创建的结构的对象。

    参考：[#3924](https://www.sqlalchemy.org/trac/ticket/3924)

### oracle

+   **[oracle] [bug]**

    修复了 cx_Oracle 的 WITH_UNICODE 模式，这是因为 cx_Oracle 5.3 现在似乎在构建中硬编码了这个标志；使用此模式的内部方法没有使用正确的签名。

    此更改也**回溯**到：1.0.18

    参考：[#3937](https://www.sqlalchemy.org/trac/ticket/3937)

### orm

+   **[orm] [feature]**

    `aliased()` 构造现在可以传递给 `Query.select_entity_from()` 方法。实体将从由 `aliased()` 构造表示的可选择对象中拉取。这允许与 `Query.select_entity_from()` 结合使用 `aliased()` 的特殊选项，如 `aliased.adapt_on_names`。

    参考：[#3933](https://www.sqlalchemy.org/trac/ticket/3933)

+   **[orm] [bug]**

    修复了一个在多线程环境下可能发生的竞争条件，这是由于通过[#3915](https://www.sqlalchemy.org/trac/ticket/3915)添加的缓存引起的。一个内部的`Column`对象集合可能会在别名对象上不恰当地重新生成，当尝试渲染 SQL 并收集结果时，这会让一个连接的急切加载器感到困惑，并导致属性错误。现在，在别名对象被缓存和在线程之间共享之前，该集合会被提前生成并共享。

    参考：[#3947](https://www.sqlalchemy.org/trac/ticket/3947)

### 引擎

+   **[engine] [bug]**

    添加了一个异常处理程序，当`Connection`的“autorollback”功能本身引发异常时，将会对 Py2K 上的“cause”异常发出警告。在 Py3K 中，这两个异常会被解释器自然地报告为一个发生在处理另一个时。这是继续处理回滚失败处理的一系列更改之一，上次访问是在 1.0.12 中的[#2696](https://www.sqlalchemy.org/trac/ticket/2696)。

    参考：[#3946](https://www.sqlalchemy.org/trac/ticket/3946)

### sql

+   **[sql] [bug] [postgresql]**

    增加了对`Variant`和`SchemaType`对象相互兼容的支持。也就是说，一个变体可以针对像`Enum`这样的类型创建，并且创建约束和/或数据库特定类型对象的指令将根据变体的方言映射正确传播。

    参考：[#2892](https://www.sqlalchemy.org/trac/ticket/2892)

+   **[sql] [bug]**

    修复了编译器中的一个 bug，即保存点的字符串标识符会被缓存在标识符引用字典中；由于这些标识符是任意的，如果一个单独的`Connection`使用了无限数量的保存点，或者如果直接使用保存点子句构造了无限数量的保存点名称，就会发生一个小内存泄漏。这个内存泄漏**不会**影响绝大多数情况，因为通常会在每个事务或每个固定数量的事务基础上使用一个简单计数器从“1”开始渲染保存点名称的`Connection`，然后将其丢弃。

    参考：[#3931](https://www.sqlalchemy.org/trac/ticket/3931)

+   **[sql] [bug]**

    修复了新的“schema translate”功能中的错误，其中在与列表达式一起渲染时，翻译后的模式名称将以别名名称的形式调用；仅在源翻译名称为“None”时才会发生。现在，“schema translate”功能仅在 `SchemaItem` 和 `SchemaType` 子类上才生效，即对应于数据库中可 DDL 创建的结构的对象。

    参考：[#3924](https://www.sqlalchemy.org/trac/ticket/3924)

### oracle

+   **[oracle] [bug]**

    通过发现 cx_Oracle 5.3 现在似乎在构建时强制启用 WITH_UNICODE 模式的一个修复，揭示了 cx_Oracle 的 WITH_UNICODE 模式存在的问题；一个使用此模式的内部方法未使用正确的签名。

    此更改还 **向后移植** 到：1.0.18

    参考：[#3937](https://www.sqlalchemy.org/trac/ticket/3937)

## 1.1.6

发布日期：2017 年 2 月 28 日

### orm

+   **[orm] [bug]**

    解决了加入了 eager 加载器查询构造系统中长时间未解决的性能问题，这些问题是自早期版本以来由于增加的抽象而积累起来的。每次查询使用 ad-hoc `AliasedClass` 对象产生大量列查找开销，现已用一个缓存方法替换，该方法利用了一小部分在调用之间重复使用的 `AliasedClass` 对象池。还对一些涉及 eager 连接路径构造的机制进行了优化。与 1.1.5 相比，最坏情况下的 joined loader 场景的端到端查询构造 + 单行提取测试的调用次数减少了约 60%，与 0.8.6 相比减少了约 42%。

    参考：[#3915](https://www.sqlalchemy.org/trac/ticket/3915)

+   **[orm] [bug]**

    修复了“eager_defaults”功能中的一个主要效率问题，即对于 ORM 显式插入 NULL 的列值会发出不必要的 SELECT，这些列对应于对象上未设置的属性，但没有指定任何服务器默认值，以及在更新时已过期的属性，但仍然没有设置服务器的 onupdate。由于这些列不是 eager_defaults 尝试使用的 RETURNING 的一部分，因此也不应进行后续选择。

    参考：[#3909](https://www.sqlalchemy.org/trac/ticket/3909)

+   **[orm] [bug]**

    修复了两个密切相关的 bug，涉及映射器的 eager_defaults 标志与单表继承结合使用时的情况；一个是 eager defaults 逻辑会在 eager defaults 获取期间无意中尝试访问映射器的 “exclude_properties” 列表中的列（由单表继承使用）的 bug，另一个是为了获取默认值而对行进行完全加载时，使用错误的继承映射器的 bug。

    参考：[#3908](https://www.sqlalchemy.org/trac/ticket/3908)

+   **[orm] [bug]**

    修复了从 0.9.7 引入的 bug，由于 [#3106](https://www.sqlalchemy.org/trac/ticket/3106) 导致某些形式的多级子查询加载针对别名实体会出现不正确查询的 bug，在最内层子查询中会多出一个不必要的额外 FROM 实体。

    参考：[#3893](https://www.sqlalchemy.org/trac/ticket/3893)

### orm 声明

+   **[orm] [declarative] [bug]**

    修复了“自动排除”特性的 bug，在声明中确保单个表继承子类的列不会出现在基类的其他派生类属性中，对于从基类进行多级子类化时不会生效的 bug。

    参考：[#3895](https://www.sqlalchemy.org/trac/ticket/3895)

### sql

+   **[sql] [bug]**

    修复了 `DDLEvents.column_reflect()` 事件的 bug，在新列的 “default” 值中不允许传递非文本表达式的 bug，例如用于指示通用触发默认或 `text()` 构造的 `FetchedValue` 对象。同时在文档中澄清了这一点。

    参考：[#3905](https://www.sqlalchemy.org/trac/ticket/3905)

### postgresql

+   **[postgresql] [bug]**

    对“IMPORT FOREIGN SCHEMA”、“REFRESH MATERIALIZED VIEW” PostgreSQL 语句添加了正则表达式，使其在通过连接或引擎调用时自动提交，而无需显式事务。感谢 Frazer McLean 和 Paweł Stiasny 的拉取请求。

    参考：[#3804](https://www.sqlalchemy.org/trac/ticket/3804)

+   **[postgresql] [bug]**

    修复了 PostgreSQL 中 `ExcludeConstraint` 的 bug，在像 `Table.tometadata()` 这样的操作中，“whereclause” 和 “using” 参数将不会被复制。

    参考：[#3900](https://www.sqlalchemy.org/trac/ticket/3900)

### mysql

+   **[mysql] [bug]**

    为 MySQL 方言添加了新的 MySQL 8.0 保留字，以便正确引用。感谢 Hanno Schlichting 的拉取请求。

### mssql

+   **[mssql] [bug]**

    在“get_isolation_level”功能中添加了一个版本检查，该功能在首次连接时调用，因此对于 SQL Server 2000 版本，它会跳过，因为在 SQL Server 2005 之前不可用必要的系统视图。

    参考：[#3898](https://www.sqlalchemy.org/trac/ticket/3898)

### 杂项

+   **[feature] [ext]**

    已添加`Result.scalar()`和`Result.count()`到“烘焙”查询系统中。

    参考：[#3896](https://www.sqlalchemy.org/trac/ticket/3896)

+   **[bug] [ext]**

    修复了新的`sqlalchemy.ext.indexable`扩展中的错误，其中设置一个属性，该属性本身引用另一个属性将失败。

    参考：[#3901](https://www.sqlalchemy.org/trac/ticket/3901)

### orm

+   **[orm] [bug]**

    解决了在连接的急切加载器查询构造系统中长期存在的性能问题，这些问题自早期版本以来一直积累，这是由于抽象层次的增加所导致的。每次查询都使用临时`AliasedClass`对象，这会产生大量的列查找开销，现在已经被一个缓存方法取代，该方法利用一小组在连接的急切加载之间重复使用的`AliasedClass`对象。还对涉及急切连接路径构造的一些机制进行了优化。对于最坏情况下的连接加载器方案的端到端查询构造+单行获取测试的调用次数，与 1.1.5 相比减少了约 60%，与 0.8.6 相比减少了约 42%。

    参考：[#3915](https://www.sqlalchemy.org/trac/ticket/3915)

+   **[orm] [bug]**

    修复了“eager_defaults”功能中的一个主要低效性，即当 ORM 明确插入 NULL 对应于对象上未设置但没有指定任何服务器默认值的属性，以及在更新时过期的属性，尽管没有设置服务器 onupdate 时，将发出不必要的 SELECT。由于这些列不是 eager_defaults 尝试使用的 RETURNING 的一部分，它们也不应该被后续 SELECT。

    参考：[#3909](https://www.sqlalchemy.org/trac/ticket/3909)

+   **[orm] [bug]**

    修复了两个与映射器急切默认标志相关的紧密相关的错误，与单表继承结合使用；一个是急切默认逻辑会在急切默认获取期间无意中尝试访问映射器的“exclude_properties”列表中的列（由具有单表继承的 Declarative 使用），另一个是为了获取默认值而完全加载行时，会失败地使用不正确的继承映射器。

    参考：[#3908](https://www.sqlalchemy.org/trac/ticket/3908)

+   **[orm] [bug]**

    修复了在 0.9.7 中首次引入的 bug，由于 [#3106](https://www.sqlalchemy.org/trac/ticket/3106) 导致某些形式的多级子查询加载对别名实体的错误查询，在最内层子查询中多了一个不必要的额外 FROM 实体。

    参考：[#3893](https://www.sqlalchemy.org/trac/ticket/3893)

### orm 声明式

+   **[orm] [declarative] [bug]**

    修复了声明式的 “自动排除” 功能的 bug，该功能确保单个表继承子类的列不会出现在基类的其他派生类的属性中。

    参考：[#3895](https://www.sqlalchemy.org/trac/ticket/3895)

### sql

+   **[sql] [bug]**

    修复了 `DDLEvents.column_reflect()` 事件中的一个 bug，该事件不允许将非文本表达式作为新列的 “default” 值传递，例如 `FetchedValue` 对象表示通用触发默认值或 `text()` 构造。同时也在文档中对此进行了澄清。

    参考：[#3905](https://www.sqlalchemy.org/trac/ticket/3905)

### postgresql

+   **[postgresql] [bug]**

    为 “IMPORT FOREIGN SCHEMA”、“REFRESH MATERIALIZED VIEW” PostgreSQL 语句添加了正则表达式，以便在通过连接或引擎调用时自动提交，而无需显式事务。感谢 Frazer McLean 和 Paweł Stiasny 的拉取请求。

    参考：[#3804](https://www.sqlalchemy.org/trac/ticket/3804)

+   **[postgresql] [bug]**

    修复了 PostgreSQL `ExcludeConstraint` 中的一个 bug，在类似 `Table.tometadata()` 这样的操作中，“whereclause” 和 “using” 参数不会被复制。

    参考：[#3900](https://www.sqlalchemy.org/trac/ticket/3900)

### mysql

+   **[mysql] [bug]**

    为 MySQL 8.0 添加了新的保留字到 MySQL 方言以进行正确引用。感谢 Hanno Schlichting 的拉取请求。

### mssql

+   **[mssql] [bug]**

    为 “get_isolation_level” 功能添加了版本检查，该功能在首次连接时调用，以便在 SQL Server 版本 2000 中跳过���因为在 SQL Server 2005 之前不可用必要的系统视图。

    参考：[#3898](https://www.sqlalchemy.org/trac/ticket/3898)

### 杂项

+   **[feature] [ext]**

    为 “baked” 查询系统添加了 `Result.scalar()` 和 `Result.count()`。

    参考：[#3896](https://www.sqlalchemy.org/trac/ticket/3896)

+   **[bug] [ext]**

    修复了新的`sqlalchemy.ext.indexable`扩展中设置引用另一个属性的属性会失败的错误。

    参考：[#3901](https://www.sqlalchemy.org/trac/ticket/3901)

## 1.1.5

发布日期：2017 年 1 月 17 日

### orm

+   **[orm] [bug]**

    修复了涉及多个实体的连接急加载与多态继承同时使用时会抛出“‘NoneType’对象没有‘isa’属性”的错误的错误。此问题是由对[#3611](https://www.sqlalchemy.org/trac/ticket/3611)的修复引入的。

    此更改也**回溯**到：1.0.17

    参考：[#3884](https://www.sqlalchemy.org/trac/ticket/3884)

+   **[orm] [bug]**

    修复了子查询加载中的错误，其中作为“现有”行遇到的对象，例如在同一查询中从不同路径加载的对象，不会为指定了此加载的未加载属性调用子查询加载程序。此问题与[#3431](https://www.sqlalchemy.org/trac/ticket/3431)、[#3811](https://www.sqlalchemy.org/trac/ticket/3811)中涉及的与连接加载类似的问题在同一领域。

    参考：[#3854](https://www.sqlalchemy.org/trac/ticket/3854)

+   **[orm] [bug]**

    `Session.no_autoflush`上下文管理器现在确保在“finally”块内重置自动刷新标志，以便如果在块内引发异常，则状态仍然适当地重置。感谢 Emin Arakelian 提供的拉取请求。

+   **[orm] [bug]**

    修复了单表继承查询标准不会插入到查询中的错误，即如果`Bundle`构造用作选择标准。

    参考：[#3874](https://www.sqlalchemy.org/trac/ticket/3874)

+   **[orm] [bug]**

    修复了与[#3177](https://www.sqlalchemy.org/trac/ticket/3177)相关的错误，其中`Query`发出的 UNION 或其他集合操作会将“单继承”标准应用于联合的外部（还引用了错误的可选择项），尽管现在预期这些标准已经存在于内部子查询中。一旦针对`Query`调用 union()或另一个集合操作，单继承标准就会像`Query.from_self()`一样被省略。

    参考：[#3856](https://www.sqlalchemy.org/trac/ticket/3856)

### 例子

+   **[examples] [bug]**

    修复了版本历史示例中的两个问题，一个是历史表现在 autoincrement=False 以避免 1.1 版本中关于自动增量的复合主键的新错误；另一个是现在使用 sqlite_autoincrement 标志以确保在 SQLite 上，即使删除了一些行，表的生命周期中也使用唯一标识符。感谢 Carlos García Montoro 的拉取请求。

    参考：[#3872](https://www.sqlalchemy.org/trac/ticket/3872)

### 引擎

+   **[engine] [bug]**

    `Table`反射的“extend_existing”选项会导致索引和约束在使用`MetaData.reflect()`（如 automap 扩展所做的）参数时会重复出现，因为表既在外键路径中反射，也直接反射。在`MetaData.reflect()`序列中传递一个新的去重集合，以防止以这种方式重复反射。

    参考：[#3861](https://www.sqlalchemy.org/trac/ticket/3861)

### sql

+   **[sql] [bug]**

    修复了最初在 0.9 版本中引入的 bug，通过[#1068](https://www.sqlalchemy.org/trac/ticket/1068)，其中 order_by(<some Label()>)会根据名称单独对标签名称进行排序，即使标记的表达式与可选择的其他表达式完全不同，隐式或显式地存在。现在，按标签排序的逻辑确保标记的表达式与解析为该名称的表达式相关联，然后按标签名称排序；此外，名称必须在表达式中的其他地方明确解析为实际标签，而不仅仅是列名。这种逻辑与按(textual name)排序的功能严格分开，后者具有稍微不同的目的。

    参考：[#3882](https://www.sqlalchemy.org/trac/ticket/3882)

+   **[sql] [bug]**

    修复了 1.1 版本中`import *`无法在 sqlalchemy.sql.expression 中工作的回归问题，因为`any_`和`all_`函数拼写错误。

    参考：[#3878](https://www.sqlalchemy.org/trac/ticket/3878)

+   **[sql] [bug]**

    在`MetaData.reflect()`中“无法反射”异常中嵌入的引擎 URL 现在隐藏了密码；此外，`TLEngine`的`__repr__`现在的行为类似于`Engine`，隐藏了 URL 密码。感谢 Valery Yundin 的拉取请求。

+   **[sql] [bug]**

    修复了 `Variant` 中的问题，其中“right hand coercion”逻辑继承自 `TypeDecorator` ，会将右侧强制转换为 `Variant` 本身，而不是默认的 `Variant` 的默认类型。在 `Variant` 的情况下，我们希望类型大部分像基本类型一样运行，因此默认的 `TypeDecorator` 逻辑现在被重写为回退到基础包装类型的逻辑。目前主要与 JSON 相关。

    参考：[#3859](https://www.sqlalchemy.org/trac/ticket/3859)

+   **[sql] [bg]**

    修复了一个问题，即当“literal_binds”编译器标志对“multiple values”特性的 `Insert` 构造未被遵守时，随后的值现在将作为文字呈现。

    参考：[#3880](https://www.sqlalchemy.org/trac/ticket/3880)

### postgresql

+   **[postgresql] [bug]**

    修复了新的“ON CONFLICT DO UPDATE”功能中的错误，其中 UPDATE 子句的“set”值将不受类型级别处理的影响，通常会处理用户定义的类型级别转换以及方言要求的转换，例如 JSON 数据类型所需的转换。另外，澄清了 `set_` 字典中的键应该与列的“键”匹配，如果与列名不同，则应匹配。对于不匹配列名的剩余列名，将发出警告；出于兼容性原因，这些警告将像以前一样发出。

    参考：[#3888](https://www.sqlalchemy.org/trac/ticket/3888)

+   **[postgresql] [bug]**

    `TIME` 和 `TIMESTAMP` 数据类型现在支持“精度”设置为零；以前零会被忽略。感谢 Ionuț Ciocîrlan 提供的拉取请求。

### mysql

+   **[mysql] [feature]**

    添加了一个新的参数 `mysql_prefix`，由 `Index` 构造支持，允许指定 MySQL 特定的前缀，如“FULLTEXT”。感谢 Joseph Schorr 提供的拉取请求。

+   **[mysql] [bug]**

    MySQL 方言现在在反射的列上有“COMMENT”关键字时不会发出警告，但需要注意的是，评论尚未反映出来；这在未来的发布计划中。感谢 Lele Long 提供的拉取请求。

    参考：[#3867](https://www.sqlalchemy.org/trac/ticket/3867)

### mssql

+   **[mssql] [bug]**

    修复了 SQL Server 方言尝试从 SELECT 中选择最后一行标识符进行插入时出现 bug 的问题，在 SELECT 没有行的情况下会失败。对于这样的语句，内联标志被设置为 True，表示不应获取最后一个主键。

    参考：[#3876](https://www.sqlalchemy.org/trac/ticket/3876)

### oracle

+   **[oracle] [bug] [postgresql]**

    修复了一个 bug，在源表包含自动递增序列的情况下，从 SELECT 进行插入会无法正确编译。

    参考：[#3877](https://www.sqlalchemy.org/trac/ticket/3877)

+   **[oracle] [bug]**

    修复了在 Oracle 9.2 上在 ALL_TABLES 查询中使用“COMPRESSION”关键字的 bug；尽管 Oracle 文档指出表压缩是在 9i 中引入的，但实际列直到 10.1 才存在。

    参考：[#3875](https://www.sqlalchemy.org/trac/ticket/3875)

### misc

+   **[bug] [py3k]**

    修复了与未带‘r’修饰符的转义字符串相关的 Python 3.6 DeprecationWarnings，并为 Python 3.6 添加了测试覆盖。

    此更改也**回溯**到：1.0.17

    参考：[#3886](https://www.sqlalchemy.org/trac/ticket/3886)

+   **[bug] [firebird]**

    将用于 Oracle 引用小写名称的修复移植到 Firebird，以便可以正确反映以小写引用的表名，包括表名来自 get_table_names() 检查函数的情况。

    参考：[#3548](https://www.sqlalchemy.org/trac/ticket/3548)

### orm

+   **[orm] [bug]**

    修复了涉及多个实体的连接急加载以及同时使用多态继承时会抛出“'NoneType' object has no attribute 'isa'”错误的 bug。此问题是由于 [#3611](https://www.sqlalchemy.org/trac/ticket/3611) 的修复引入的。

    此更改也**回溯**到：1.0.17

    参考：[#3884](https://www.sqlalchemy.org/trac/ticket/3884)

+   **[orm] [bug]**

    修复了子查询加载中的 bug，当对象作为“现有”行遇到时，例如在同一查询中从不同路径加载的对象，不会为指定了此加载的未加载属性调用子查询加载器。这个问题与 [#3431](https://www.sqlalchemy.org/trac/ticket/3431)、[#3811](https://www.sqlalchemy.org/trac/ticket/3811) 中涉及的与连接加载类似的问题在同一领域。

    参考：[#3854](https://www.sqlalchemy.org/trac/ticket/3854)

+   **[orm] [bug]**

    `Session.no_autoflush` 上下文管理器现在确保在“finally”块中重置自动刷新标志，因此如果在块内引发异常，则状态仍会适当重置。感谢 Emin Arakelian 的拉取请求。

+   **[orm] [bug]**

    修复了单表继承查询条件不会插入到查询中的 bug，即在选择条件中使用 `Bundle` 构造时。

    参考：[#3874](https://www.sqlalchemy.org/trac/ticket/3874)

+   **[orm] [bug]**

    修复了与 [#3177](https://www.sqlalchemy.org/trac/ticket/3177) 相关的 bug，即 `Query` 发出的 UNION 或其他集合操作会将“单继承”条件应用于联合的外部（还引用了错误的可选择项），尽管现在预期这些条件已经存在于内部子查询中。一旦对 `Query` 调用 union() 或其他集合操作，单继承条件就会像 `Query.from_self()` 一样被省略。

    参考：[#3856](https://www.sqlalchemy.org/trac/ticket/3856)

### 示例

+   **[examples] [bug]**

    修复了版本化历史示例的两个问题，一个是历史表现在 autoincrement=False 以避免 1.1 版本关于具有自动增量的复合主键的新错误；另一个是现在使用 sqlite_autoincrement 标志以确保在 SQLite 上，即使删除了某些行，也会为表的生命周期使用唯一标识符。感谢 Carlos García Montoro 的拉取请求。

    参考：[#3872](https://www.sqlalchemy.org/trac/ticket/3872)

### 引擎

+   **[engine] [bug]**

    `Table` 反射的“extend_existing”选项会导致索引和约束在使用 `MetaData.reflect()`（如 automap 扩展所做的那样）参数时会重复出现，因为表会在外键路径和直接反射两次。在 `MetaData.reflect()` 序列中传递一个新的去重集合，以防止这种重复反射。

    参考：[#3861](https://www.sqlalchemy.org/trac/ticket/3861)

### sql

+   **[sql] [bug]**

    修复了最初在 0.9 版本中引入的 bug，通过 [#1068](https://www.sqlalchemy.org/trac/ticket/1068)，其中 order_by(<some Label()>) 会根据名称对标签进行排序，即使标记的表达式与可选择的其他表达式在其他地方明确或隐式地存在，也是如此。现在，按标签排序的逻辑确保标记的表达式与解析为该名称的表达式相关联，然后再按标签名称排序；此外，名称必须在表达式的其他地方明确表示为实际标签，而不仅仅是列名。这种逻辑与按照（文本名称）排序的功能严格分开，后者具有稍微不同的目的。

    参考：[#3882](https://www.sqlalchemy.org/trac/ticket/3882)

+   **[sql] [bug]**

    修复了 1.1 版本的一个回归问题，即 `import *` 对于 sqlalchemy.sql.expression 不起作用，因为 `any_` 和 `all_` 函数拼写错误。

    参考：[#3878](https://www.sqlalchemy.org/trac/ticket/3878)

+   **[sql] [bug]**

    在 `MetaData.reflect()` 中“无法反射”异常中嵌入的引擎 URL 现在隐藏了密码；此外，`TLEngine` 的 `__repr__` 现在的行为类似于 `Engine`，隐藏了 URL 密码。感谢 Valery Yundin 提交的拉取请求。

+   **[sql] [bug]**

    修复了 `Variant` 中的问题，其中“右手边强制转换”逻辑，继承自 `TypeDecorator`，会将右侧强制转换为 `Variant` 本身，而不是默认类型所做的。在 `Variant` 的情况下，我们希望该类型大部分像基本类型一样运行，因此现在覆盖了 `TypeDecorator` 的默认逻辑，以回退到基础包装类型的逻辑。目前主要与 JSON 相关。

    参考：[#3859](https://www.sqlalchemy.org/trac/ticket/3859)

+   **[sql] [bg]**

    修复了 literal_binds 编译器标志未被 `Insert` 构造的“多值”功能所遵守的 bug；随后的值现在被呈现为文字值。

    参考：[#3880](https://www.sqlalchemy.org/trac/ticket/3880)

### postgresql

+   **[postgresql] [bug]**

    修复了新的“ON CONFLICT DO UPDATE”功能中的 bug，即 UPDATE 子句的“set”值不会受到类型级别处理的影响，通常会处理用户定义的类型级别转换以及方言所需的转换，例如 JSON 数据类型所需的转换。此外，澄清了`set_`字典中的键应与列的“key”匹配，如果与列名不同。对于不匹配列键的剩余列名发出警告；出于兼容性原因，这些列名会像以前一样发出。

    参考：[#3888](https://www.sqlalchemy.org/trac/ticket/3888)

+   **[postgresql] [bug]**

    `TIME` 和 `TIMESTAMP` 数据类型现在支持“precision”设置为零；以前零会被忽略。感谢 Ionuț Ciocîrlan 的拉取请求。

### mysql

+   **[mysql] [feature]**

    添加了一个新参数`mysql_prefix`，由`Index`构造支持，允许指定 MySQL 特定的前缀，如“FULLTEXT”。感谢 Joseph Schorr 的拉取请求。

+   **[mysql] [bug]**

    MySQL 方言现在不会在反射列上有“COMMENT”关键字时发出警告，但请注意评论尚未反映；这在将来的版本中计划。感谢 Lele Long 的拉取请求。

    参考：[#3867](https://www.sqlalchemy.org/trac/ticket/3867)

### mssql

+   **[mssql] [bug]**

    修复了 SQL Server 方言尝试选择 INSERT from SELECT 的最后一行标识时失败的 bug，在 SELECT 没有行的情况下会失败。对于这样的语句，内联标志设置为 True，表示不应获取最后一个主键。

    参考：[#3876](https://www.sqlalchemy.org/trac/ticket/3876)

### oracle

+   **[oracle] [bug] [postgresql]**

    修复了在源表包含自动递增序列的情况下，从 SELECT 进行 INSERT 会导致编译错误的 bug。

    参考：[#3877](https://www.sqlalchemy.org/trac/ticket/3877)

+   **[oracle] [bug]**

    修复了在 Oracle 9.2 上在 ALL_TABLES 查询中使用“COMPRESSION”关键字的 bug；尽管 Oracle 文档中指出表压缩是在 9i 中引入的，但实际列直到 10.1 才存在。

    参考：[#3875](https://www.sqlalchemy.org/trac/ticket/3875)

### misc

+   **[bug] [py3k]**

    修复了与没有‘r’修饰符的转义字符串相关的 Python 3.6 DeprecationWarnings，并为 Python 3.6 添加了测试覆盖。

    此更改也已**回溯**到：1.0.17

    参考：[#3886](https://www.sqlalchemy.org/trac/ticket/3886)

+   **[bug] [firebird]**

    将解决 Oracle 引号小写名称的问题移植到 Firebird，以便可以正确反映作为小写引号的表名，包括表名来自 get_table_names() 检查函数时。

    参考：[#3548](https://www.sqlalchemy.org/trac/ticket/3548)

## 1.1.4

发布日期：2016 年 11 月 15 日

### orm

+   **[orm] [bug]**

    修复了 `Session.bulk_update_mappings()` 中的 bug，即备用命名的主键属性无法正确跟踪到 UPDATE 语句中的问题。

    此更改也**回溯**至：1.0.16

    参考：[#3849](https://www.sqlalchemy.org/trac/ticket/3849)

+   **[orm] [bug]**

    修复了 `Session.bulk_save()` 中的 bug，当与实现版本 id 计数器的映射配合使用时，UPDATE 将无法正确运行的问题。

    此更改也**回溯**至：1.0.16

    参考：[#3781](https://www.sqlalchemy.org/trac/ticket/3781)

+   **[orm] [bug]**

    修复了一个 bug，即当在第一次调用这些访问器之后向映射器/类添加映射器属性或其他 ORM 构造时，`Mapper.attrs`、`Mapper.all_orm_descriptors` 和其他派生属性将无法刷新的问题。

    此更改也**回溯**至：1.0.16

    参考：[#3778](https://www.sqlalchemy.org/trac/ticket/3778)

+   **[orm] [bug]**

    由于 [#3457](https://www.sqlalchemy.org/trac/ticket/3457) 导致的 collections 中的回归 bug 修复，即在 pickle 或 deepcopy 过程中反序列化将无法建立 ORM 集合的所有属性，导致进一步的变异操作失败。

    参考：[#3852](https://www.sqlalchemy.org/trac/ticket/3852)

+   **[orm] [bug]**

    修复了长期存在的 bug，即“noload”关系加载策略会导致忽略 backrefs 和/或 back_populates 选项的问题。

    参考：[#3845](https://www.sqlalchemy.org/trac/ticket/3845)

### engine

+   **[engine] [bug]**

    从 `Connection` 中删除了长期存在但已损坏的“default_schema_name()”方法。该方法是来自非常旧的版本，并且是不工作的（例如，将引发异常）。感谢 Benjamin Dopplinger 提供的拉取请求。

### sql

+   **[sql] [bug]**

    修复了新增的警告，用于没有设置自增设置的插入主键时会失败正确发出的问题（参见 [#3216](https://www.sqlalchemy.org/trac/ticket/3216)），当应用于小写 `table()` 构造时。 

    参考：[#3842](https://www.sqlalchemy.org/trac/ticket/3842)

### postgresql

+   **[postgresql] [bug]**

    由于修复[#3807](https://www.sqlalchemy.org/trac/ticket/3807)（版本 1.1.0）引起的回归错误，我们确保在 PostgreSQL 的 ON CONFLICT 的 DO UPDATE 部分的 WHERE 子句中限定了表名，然而你*不能*在实际的 ON CONFLICT 本身的 WHERE 子句中放置表名。这是一个错误的假设，因此在[#3807](https://www.sqlalchemy.org/trac/ticket/3807)中的这部分更改被撤销。

    参考：[#3807](https://www.sqlalchemy.org/trac/ticket/3807), [#3846](https://www.sqlalchemy.org/trac/ticket/3846)

### mysql

+   **[mysql] [feature]**

    为 mysqlclient 和 pymysql 方言添加了对服务器端游标的支持。这个功能可以通过`Connection.execution_options.stream_results`标志以及相同方式中的`server_side_cursors=True`方言参数来使用，就像在 PostgreSQL 上的 psycopg2 中一样。感谢 Roman Podoliaka 的拉取请求。

+   **[mysql] [bug]**

    MySQL 的原生 ENUM 类型支持发送任何非有效值，并且会返回一个空字符串。为了将这个空字符串返回给应用程序而不被拒绝为非有效值，MySQL 的 ENUM 实现中添加了一个硬编码规则来检查“是否返回空字符串”。请注意，如果你的 MySQL 枚举将值链接到对象，你仍然会得到空字符串返回。

    参考：[#3841](https://www.sqlalchemy.org/trac/ticket/3841)

### sqlite

+   **[sqlite] [bug]**

    在 pysqlcipher 方言的 PRAGMA 指令中添加了引号，以适当支持额外的密码参数。感谢 Kevin Jurczyk 的拉取请求。

+   **[sqlite] [bug] [py3k]**

    在使用 pysqlcipher 方言时，为 pysqlcipher3 DBAPI 添加了一个可选导入。如果 Python-2 专用的 pysqlcipher DBAPI 不存在，这个包将尝试被导入。感谢 Kevin Jurczyk 的拉取请求。

### mssql

+   **[mssql] [bug]**

    修复了 pyodbc 方言（以及大部分不工作的 adodbapi 方言）中的错误，其中密码或用户名字段中存在的分号可能被解释为另一个标记的分隔符；当存在分号时，现在这些值将被引用。

    此更改也**被回溯到**：1.0.16

    参考：[#3762](https://www.sqlalchemy.org/trac/ticket/3762)

### orm

+   **[orm] [bug]**

    修复了`Session.bulk_update_mappings()`中的错误，其中替代命名的主键属性无法正确跟踪到 UPDATE 语句中。

    此更改也**被回溯到**：1.0.16

    参考：[#3849](https://www.sqlalchemy.org/trac/ticket/3849)

+   **[orm] [bug]**

    修复了`Session.bulk_save()`中的错误，其中 UPDATE 与实现版本 id 计数器的映射无法正确运行。

    此更改也**回溯**到：1.0.16

    参考：[#3781](https://www.sqlalchemy.org/trac/ticket/3781)

+   **[ORM] [错误]**

    修复了当在首次调用这些访问器后向映射器/类添加映射器属性或其他 ORM 构造时，`Mapper.attrs`、`Mapper.all_orm_descriptors` 和其他派生属性无法刷新的 bug。

    此更改也**回溯**到：1.0.16

    参考：[#3778](https://www.sqlalchemy.org/trac/ticket/3778)

+   **[ORM] [错误]**

    由于 [#3457](https://www.sqlalchemy.org/trac/ticket/3457) 导致的集合中的反序列化回退问题已修复，pickle 或 deepcopy 过程中会导致 ORM 集合的所有属性无法建立，进而导致进一步的变异操作失败。

    参考：[#3852](https://www.sqlalchemy.org/trac/ticket/3852)

+   **[ORM] [错误]**

    修复了长期存在的 bug，即“noload”关系加载策略会导致 backrefs 和/或 back_populates 选项被忽略的问题。

    参考：[#3845](https://www.sqlalchemy.org/trac/ticket/3845)

### 引擎

+   **[引擎] [错误]**

    从`Connection`中移除了长期存在的破损的“default_schema_name()”方法。这个方法是从一个非常旧的版本遗留下来的，是无效的（例如，会引发异常）。感谢 Benjamin Dopplinger 的拉取请求。

### sql

+   **[sql] [错误]**

    修复了一个 bug，即在没有设置自动增量的情况下插入主键时新增的警告（参见 [#3216](https://www.sqlalchemy.org/trac/ticket/3216)）在调用小写 `table()` 构造时无法正确发出。

    参考：[#3842](https://www.sqlalchemy.org/trac/ticket/3842)

### postgresql

+   **[postgresql] [错误]**

    由于修复 [#3807](https://www.sqlalchemy.org/trac/ticket/3807)（版本 1.1.0）引起的回退，我们确保在 PostgreSQL 的 ON CONFLICT 的 DO UPDATE 部分的 WHERE 子句中限定了表名，然而实际上*不能*在 ON CONFLICT 本身的 WHERE 子句中放置表名。这是一个错误的假设，因此将回滚 [#3807](https://www.sqlalchemy.org/trac/ticket/3807) 中的这部分更改。

    参考：[#3807](https://www.sqlalchemy.org/trac/ticket/3807)，[#3846](https://www.sqlalchemy.org/trac/ticket/3846)

### mysql

+   **[mysql] [特性]**

    为 mysqlclient 和 pymysql 方言添加了对服务器端游标的支持。通过`Connection.execution_options.stream_results`标志以及相同方式的`server_side_cursors=True`方言参数可用此功能，就像在 PostgreSQL 上对 psycopg2 一样。感谢 Roman Podoliaka 的 Pull 请求。

+   **[mysql] [bug]**

    MySQL 的本机 ENUM 类型支持发送任何非有效值，并且响应将返回空字符串。已向 ENUM 的 MySQL 实现添加了一个硬编码规则，以检查“返回空字符串”，以便将此空字符串返回给应用程序而不被拒绝为非有效值。请注意，如果您的 MySQL 枚举将值链接到对象，则仍然会返回空字符串。

    引用：[#3841](https://www.sqlalchemy.org/trac/ticket/3841)

### sqlite

+   **[sqlite] [bug]**

    在 pysqlcipher 方言中的 PRAGMA 指令中添加了引号，以适当支持附加的密码参数。感谢 Kevin Jurczyk 的 Pull 请求。

+   **[sqlite] [bug] [py3k]**

    在使用 pysqlcipher 方言时，为 pysqlcipher3 DBAPI 添加了可选导入。如果 Python-2 仅 pysqlcipher DBAPI 不存在，则尝试导入此包。感谢 Kevin Jurczyk 的 Pull 请求。

### mssql

+   **[mssql] [bug]**

    修复了 pyodbc 方言中的错误（以及基本上不起作用的 adodbapi 方言中的错误），其中密码或用户名字段中存在分号时可能会将分号解释为另一个令牌的分隔符；现在在存在分号时对值进行引用。

    此更改还被**回溯到**：1.0.16

    引用：[#3762](https://www.sqlalchemy.org/trac/ticket/3762)

## 1.1.3

发布日期：2016 年 10 月 27 日

### orm

+   **[orm] [bug]**

    修复了由[#2677](https://www.sqlalchemy.org/trac/ticket/2677)引起的回归，即在该会话中已经刷新为删除状态的对象调用`Session.delete()`会导致对象未能在标识映射中设置（或拒绝该对象），从而导致刷新错误，因为该对象处于工作单元无法容纳的状态。在这种情况下，恢复了 1.1 版本之前的行为，即将对象放回标识映射中，以便再次尝试 DELETE 语句，这会发出警告，指出未匹配预期行数（除非在会话之外还原了该行）。

    引用：[#3839](https://www.sqlalchemy.org/trac/ticket/3839)

+   **[orm] [bug]**

    修复了一些`Query`方法（如`Query.update()`等）在针对一系列映射列而不是整个映射实体的`Query`时会失败的回归。

    参考：[#3836](https://www.sqlalchemy.org/trac/ticket/3836)

### sql

+   **[sql] [bug]**

    修复了在`Enum`中使用新值翻译和验证功能时的错误，其中在字符串连接中使用枚举对象会将整个表达式的类型保持为`Enum`类型，导致查找丢失。现在，针对`Enum`类型的列进行字符串连接将使用`String`作为表达式本身的数据类型。

    参考：[#3833](https://www.sqlalchemy.org/trac/ticket/3833)

+   **[sql] [bug]**

    修复了在用户定义的`TypeDecorator`同时也是`SchemaType`实例的情况下，由于[#2919](https://www.sqlalchemy.org/trac/ticket/2919)的副作用而导致列附加事件被跳过的回归。

    参考：[#3832](https://www.sqlalchemy.org/trac/ticket/3832)

### postgresql

+   **[postgresql] [bug]**

    PostgreSQL 表反射将确保在反射主键列时，如果列不是`Integer`数据类型，则`Column.autoincrement`标志设置为 False，即使默认值与生成整数的序列相关。如果列被创建为 SERIAL 并且数据类型已更改，则可能会发生这种情况。在 1.1 系列中，只有数据类型为整数亲和性时，autoincrement 标志才能为 True。

    参考：[#3835](https://www.sqlalchemy.org/trac/ticket/3835)

### orm

+   **[orm] [bug]**

    修复了由[#2677](https://www.sqlalchemy.org/trac/ticket/2677)引起的回归，其中在会话中已经刷新为删除状态的对象上调用`Session.delete()`将无法设置对象到标识映射（或拒绝对象），导致刷新错误，因为对象处于工作单元无法容纳的状态。 在这种情况下，已恢复了 1.1 之前的行为，即将对象放回到标识映射中，以便再次尝试 DELETE 语句，这会发出警告，指出未匹配预期行数（除非行在会话外被恢复）。

    参考：[#3839](https://www.sqlalchemy.org/trac/ticket/3839)

+   **[orm] [bug]**

    修复了某些`Query`方法（例如`Query.update()`等）失败的回归，如果`Query`针对一系列映射列而不是整个映射实体。

    参考：[#3836](https://www.sqlalchemy.org/trac/ticket/3836)

### sql

+   **[sql] [bug]**

    修复了在`Enum`的新值翻译和验证功能中涉及的错误，其中在字符串串联中使用枚举对象会将`Enum`类型保留为表达式的整体类型，导致查找丢失。 现在，对`Enum`类型的列进行字符串串联将使用`String`作为表达式本身的数据类型。

    参考：[#3833](https://www.sqlalchemy.org/trac/ticket/3833)

+   **[sql] [bug]**

    修复了由[#2919](https://www.sqlalchemy.org/trac/ticket/2919)的副作用导致的回归，即在用户定义的`TypeDecorator`也是`SchemaType`实例的情况下（而不是实现为此的情况）会导致跳过类型本身的列附加事件。

    参考：[#3832](https://www.sqlalchemy.org/trac/ticket/3832)

### postgresql

+   **[postgresql] [bug]**

    PostgreSQL 表反射将确保在反射主键列时，如果默认值与生成整数的序列相关联，将`Column.autoincrement`标志设置为 False，即使数据类型已更改为不是`Integer`数据类型。如果列被创建为 SERIAL 并且数据类型已更改，则 autoincrement 标志只能在 1.1 系列中的整数亲和性数据类型为 True。

    参考：[#3835](https://www.sqlalchemy.org/trac/ticket/3835)

## 1.1.2

发布日期：2016 年 10 月 17 日

### orm

+   **[orm] [bug]**

    修复了一个 bug，涉及到在一个多对一懒加载器的另一侧禁用连接集合急加载器的规则，首次添加在[#1495](https://www.sqlalchemy.org/trac/ticket/1495)中，如果父对象有一些其他与 lazyloader 绑定的查询选项，则规则会失败。

    参考：[#3824](https://www.sqlalchemy.org/trac/ticket/3824)

+   **[orm] [bug]**

    修复了自引用实体中延迟列加载问题，类似于[#3431](https://www.sqlalchemy.org/trac/ticket/3431)、[#3811](https://www.sqlalchemy.org/trac/ticket/3811)中的问题，其中由于自引用急加载而导致实体在行中出现多次；当延迟加载器仅适用于其中一条路径时，“存在”列加载器现在将覆盖该实体的延迟非加载，而不考虑行顺序。

    参考：[#3822](https://www.sqlalchemy.org/trac/ticket/3822)

### sql

+   **[sql] [bug]**

    修复了由于新增的执行 sql `DefaultGenerator`对象的“包装可调用”函数而引起的回归问题，在默认可调用对象是`functools.partial`或其他没有`__module__`属性的对象时，会引发`__module__`的属性错误。

    参考：[#3823](https://www.sqlalchemy.org/trac/ticket/3823)

+   **[sql] [bug] [postgresql]**

    修复了`Enum`类型中的回归问题，其中事件处理程序在类型对象被复制时未被传递，这是由于在[#3250](https://www.sqlalchemy.org/trac/ticket/3250)中添加了一个冲突的 copy()方法。在复制列时，例如在 tometadata()中或在使用带有列的声明性混合时，通常会发生此复制。事件处理程序的缺失会影响为非本地枚举类型创建的约束，但更为关键的是在 PostgreSQL 后端的 ENUM 对象上。

    参考：[#3827](https://www.sqlalchemy.org/trac/ticket/3827)

### postgresql

+   **[postgresql] [bug] [sql]**

    更改了生成多值插入语句的绑定参数时使用的命名约定，以便编号参数名称不会与 WHERE 子句中的匿名参数发生冲突，这在 PostgreSQL ON CONFLICT 结构中现在很常见。

    参考：[#3828](https://www.sqlalchemy.org/trac/ticket/3828)

### orm

+   **[orm] [bug]**

    修复了一个 bug，该 bug 涉及在许多对一懒加载器的另一侧禁用连接集合急加载器的规则，首次添加在[#1495](https://www.sqlalchemy.org/trac/ticket/1495)中，如果父对象有一些其他与之关联的懒加载器绑定的查询选项，该规则将失败。

    参考：[#3824](https://www.sqlalchemy.org/trac/ticket/3824)

+   **[orm] [bug]**

    修复了自引用实体中延迟列加载问题，类似于[#3431](https://www.sqlalchemy.org/trac/ticket/3431)、[#3811](https://www.sqlalchemy.org/trac/ticket/3811)中的问题，其中由于自引用的急加载，实体在行中出现在多个位置；当延迟加载器仅适用于其中一个路径时，“present”列加载器现在将覆盖该实体的延迟非加载，而不考虑行顺序。

    参考：[#3822](https://www.sqlalchemy.org/trac/ticket/3822)

### sql

+   **[sql] [bug]**

    修复了由于新增函数执行 sql `DefaultGenerator`对象的“wrap callable”函数而引起的回归，当默认可调用函数为`functools.partial`或其他没有`__module__`属性的对象时，会引发`__module__`属性错误。

    参考：[#3823](https://www.sqlalchemy.org/trac/ticket/3823)

+   **[sql] [bug] [postgresql]**

    修复了`Enum`类型中的回归，其中事件处理程序在类型对象被复制的情况下未传递，这是由于在[#3250](https://www.sqlalchemy.org/trac/ticket/3250)中添加的冲突的 copy()方法。在复制列时，例如在 tometadata()中或在使用具有列的声明性 mixin 时，会正常发生此复制。事件处理程序的缺失会影响为非本地枚举类型创建的约束，但更为关键的是 PostgreSQL 后端的 ENUM 对象。

    参考：[#3827](https://www.sqlalchemy.org/trac/ticket/3827)

### postgresql

+   **[postgresql] [bug] [sql]**

    更改了生成多值插入语句的绑定参数时使用的命名约定，以便编号参数名称不会与 WHERE 子句中的匿名参数发生冲突，这在 PostgreSQL ON CONFLICT 结构中现在很常见。

    参考：[#3828](https://www.sqlalchemy.org/trac/ticket/3828)

## 1.1.1

发布日期：2016 年 10 月 7 日

### mssql

+   **[mssql] [bug]**

    在[#3810](https://www.sqlalchemy.org/trac/ticket/3810)和[#3814](https://www.sqlalchemy.org/trac/ticket/3814)中添加的“SELECT SERVERPROPERTY”查询在未知的 Pyodbc 和 SQL Server 组合上失败。虽然预料到了这个函数的失败，但异常捕获不够广泛，因此现在捕获所有形式的 pyodbc.Error。

    参考资料：[#3820](https://www.sqlalchemy.org/trac/ticket/3820)

### 杂项

+   **[bug] [核心]**

    当检测到各种缺失主键的情况时，将引发的 CompileError 更改为警告。语句再次传递给数据库，将会失败，并像往常一样引发 DBAPI 错误（通常是 IntegrityError）。

    参见

    不再为复合主键列隐式启用.autoincrement 指令

    参考资料：[#3216](https://www.sqlalchemy.org/trac/ticket/3216)

### mssql

+   **[mssql] [bug]**

    在[#3810](https://www.sqlalchemy.org/trac/ticket/3810)和[#3814](https://www.sqlalchemy.org/trac/ticket/3814)中添加的“SELECT SERVERPROPERTY”查询在未知的 Pyodbc 和 SQL Server 组合上失败。虽然预料到了这个函数的失败，但异常捕获不够广泛，因此现在捕获所有形式的 pyodbc.Error。

    参考资料：[#3820](https://www.sqlalchemy.org/trac/ticket/3820)

### 杂项

+   **[bug] [核心]**

    当检测到各种缺失主键的情况时，将引发的 CompileError 更改为警告。语句再次传递给数据库，将会失败，并像往常一样引发 DBAPI 错误（通常是 IntegrityError）。

    参见

    不再为复合主键列隐式启用.autoincrement 指令

    参考资料：[#3216](https://www.sqlalchemy.org/trac/ticket/3216)

## 1.1.0

发布日期：2016 年 10 月 5 日

### orm

+   **[orm] [功能]**

    加强了新的“raise”懒加载策略，还包括一个“raise_on_sql”变体，可以通过`relationship.lazy`和`raiseload()`两种方式使用。这个变体只有在懒加载实际发出 SQL 时才会引发，而不是在任何时候调用懒加载机制时引发。

    参考资料：[#3812](https://www.sqlalchemy.org/trac/ticket/3812)

+   **[orm] [功能]**

    `Query.group_by()`方法现在如果传递`None`参数，则会重置分组集合，就像`Query.order_by()`长期以来的工作方式一样。感谢 Iuri Diniz 提供的拉取请求。

+   **[orm] [更改]**

    将 False 传递给`Query.order_by()` 以取消所有排序已弃用；使用 False 或 None 调用此方法不再有任何区别。

+   **[orm] [bug]**

    修复了一个错误，即连接的贪婪加载将无法针对多态加载的映射器进行加载，其中多态性在未映射的表达式上设置，例如 CASE 表达式。

    此更改也已**回溯**到：1.0.16

    引用：[#3800](https://www.sqlalchemy.org/trac/ticket/3800)

+   **[orm] [bug]**

    修复了一个错误，即对通过 `Session.bind_mapper()`、`Session.bind_table()` 或构造函数发送的无效绑定引发的 ArgumentError 未能被正确引发。

    此更改也已**回溯**到：1.0.16

    引用：[#3798](https://www.sqlalchemy.org/trac/ticket/3798)

+   **[orm] [bug]**

    修复了子查询贪婪加载中的错误，在这种情况下，“of_type()”对象的子查询加载链接到第二个子查询加载的普通映射类，或者是几个“of_type()”属性的较长链，将无法正确链接连接。 

    此更改也已**回溯**到：1.0.15

    引用：[#3773](https://www.sqlalchemy.org/trac/ticket/3773), [#3774](https://www.sqlalchemy.org/trac/ticket/3774)

+   **[orm] [bug]**

    ORM 属性现在可以分配任何具有 `__clause_element__()` 属性的对象，这将导致内联 SQL，就像任何 `ClauseElement` 类一样。这涵盖了其他通过进一步表达式构造未转换的映射属性。

    引用：[#3802](https://www.sqlalchemy.org/trac/ticket/3802)

+   **[orm] [bug]**

    对[票务：3431](https://www.sqlalchemy.org/trac/ticket/3431)中首次引入的错误修复进行了调整，涉及到在单个结果集中出现多个上下文中的对象的情况，这样一个贪婪的加载器将会将相关对象的值设置为 None，但仍然会触发加载该属性。之前，调整仅在次要行中贪婪加载属性时才尊重非 None 值的到来。

    引用：[#3811](https://www.sqlalchemy.org/trac/ticket/3811)

+   **[orm] [bug]**

    修复了新的 `SessionEvents.persistent_to_deleted()` 事件中的错误，其中目标对象可能在事件触发之前被垃圾回收。

    引用：[#3808](https://www.sqlalchemy.org/trac/ticket/3808)

+   **[orm] [bug]**

    `relationship()` 构造的 primaryjoin 现在可以包含一个 `bindparam()` 对象，该对象包含一个可调用函数来生成值。以前，延迟加载策略与此用法不兼容，并且还会无法正确检测是否应该使用“use_get”条件，如果主键与绑定参数有关。

    参考：[#3767](https://www.sqlalchemy.org/trac/ticket/3767)

+   **[orm] [bug]**

    从 ORM 刷新过程中发出的 UPDATE 现在可以适应对象主键中的列的 SQL 表达式元素，如果目标数据库支持 RETURNING 以提供新值，或者如果为了触发其他触发器/列的 onupdate 而将 PK 值设置为“自身”。

    参考：[#3801](https://www.sqlalchemy.org/trac/ticket/3801)

+   **[orm] [bug]**

    修复了一个 bug，即“简单的一对多”条件，允许延迟加载使用来自标识映射的 get() 失败的情况，如果关系的 primaryjoin 具有由 AND 分隔的多个子句，并且这些子句的顺序与每个子句中比较的主键列的顺序不同。这种顺序差异发生在复合外键的情况下，其中引用方的表绑定列在 .c 集合中的顺序与被引用方的主键列不同……如果使用声明性混入和/或 declared_attr 来设置列，则会经常发生这种情况。

    参考：[#3788](https://www.sqlalchemy.org/trac/ticket/3788)

+   **[orm] [bug]**

    当映射上的两个 `@validates` 装饰器使用相同名称时，会引发异常。一次只支持一个特定名称的验证器，没有机制将它们链接在一起，因为在函数装饰器级别上验证器的顺序无法确定。

    另请参阅

    相同名称的 @validates 装饰器现在会引发异常

    参考：[#3776](https://www.sqlalchemy.org/trac/ticket/3776)

+   **[orm] [bug]**

    在 `configure_mappers()` 过程中引发的映射器错误现在在异常消息中明确包含原始映射器的名称，以帮助处理那些包装异常本身不包含源映射器的情况。感谢 John Perkins 的拉取请求。

### orm 声明性

+   **[orm] [declarative] [change]**

    构建一个继承自另一个类的声明性基类也将继承其文档字符串。这意味着 `as_declarative()` 的行为更像一个普通的类装饰器。

### sql

+   **[sql] [bug]**

    修复了`Table`中的错误，其中内部方法`_reset_exported()`会破坏对象的状态。此方法用于可选择对象，并在某些情况下由 ORM 调用；错误的映射器配置可能导致 ORM 在`Table`对象上调用此方法。

    此更改也**回溯**到：1.0.15

    参考：[#3755](https://www.sqlalchemy.org/trac/ticket/3755)

+   **[sql] [bug]**

    执行选项现在可以在编译时从语句传播到最外层语句，因此，如果嵌入元素想要将“autocommit”设置为 True，例如，它可以将此传播到封闭语句。目前，此功能已启用用于嵌入在 SELECT 语句中的面向 DML 的 CTE，例如，在 SELECT 语句中的 INSERT/UPDATE/DELETE。

    参考：[#3805](https://www.sqlalchemy.org/trac/ticket/3805)

+   **[sql] [bug]**

    通过`Column.server_default`参数作为列默认值发送的字符串现在已经为引号进行了转义。

    参见

    String server_default 现在是文字引用

    参考：[#3809](https://www.sqlalchemy.org/trac/ticket/3809)

+   **[sql] [bug] [postgresql]**

    添加了由 PostgreSQL 使用的编译器级别标志，用于在涉及 JSON、HSTORE 索引运算符以及其操作数的操作中放置比通常由优先规则生成的额外括号，因为已经观察到 PostgreSQL 至少在 HSTORE 索引运算符之间的优先规则在 9.4 和 9.5 之间不一致。

    参考：[#3806](https://www.sqlalchemy.org/trac/ticket/3806)

+   **[sql] [bug] [mysql]**

    `BaseException`异常类现在被`Connection`的异常处理例程拦截，并包括`ConnectionEvents.handle_error()`事件的处理。在系统级别异常（不是`Exception`的子类，包括`KeyboardInterrupt`和 greenlet `GreenletExit`类）的情况下，默认情况下`Connection`现在被**作废**，以防止在处于未知且可能已损坏状态的数据库连接上发生进一步操作。MySQL 驱动程序是这种变化的主要目标，但这种变化适用于所有 DBAPIs。

    参见

    Engines 现在作废连接，运行 BaseException 的错误处理程序

    参考：[#3803](https://www.sqlalchemy.org/trac/ticket/3803)

+   **[sql] [bug]**

    “eq”和“ne”运算符不再是“关联”运算符列表的一部分，但它们仍然被认为是“可交换的”。这允许在 SQL 级别保持带有括号的表达式`(x == y) == z`。感谢 John Passaro 的拉取请求。

    参考：[#3799](https://www.sqlalchemy.org/trac/ticket/3799)

+   **[sql] [bug]**

    对于包含未命名`Column`对象的表达式的字符串化，例如 ORM 错误报告中经常发生的情况，现在在字符串上下文中呈现名称为“<name unknown>”，而不是引发编译错误。

    参考：[#3789](https://www.sqlalchemy.org/trac/ticket/3789)

+   **[sql] [bug]**

    当 ClauseElement 或非 SQLAlchemy 对象被错误地传递给`.execute()`时，抛出更具描述性的异常/消息；在所有情况下一致地引发新异常 ObjectNotExecutableError。

    参考：[#3786](https://www.sqlalchemy.org/trac/ticket/3786)

+   **[sql] [bug] [mysql] [postgresql]**

    修复了 JSON 数据类型中的回归，其中 JSON 索引值的“文字处理器”不会被调用。现在从 JSONIndexType 和 JSONPathType 内调用本机 String 和 Integer 数据类型。这适用于通用、PostgreSQL 和 MySQL JSON 类型，还依赖于[#3766](https://www.sqlalchemy.org/trac/ticket/3766)。

    参考：[#3765](https://www.sqlalchemy.org/trac/ticket/3765)

+   **[sql] [bug]**

    修复了当将 SQL 表达式包装在 ORM 风格的`__clause_element__()`构造内部时，`Index`无法从复合 SQL 表达式中提取列的错误。这个 bug 也存在于 1.0.x 中，但在 1.1 中更为明显，因为 hybrid_property @expression 现在返回一个包装元素。

    参考：[#3763](https://www.sqlalchemy.org/trac/ticket/3763)

### postgresql

+   **[postgresql] [bug]**

    调整 ON CONFLICT，使���inserted_primary_key”逻辑能够适应没有 INSERT 或 UPDATE 且没有净变化的情况。在这种情况下，值为 None，而不是在异常上失败。

    参考：[#3813](https://www.sqlalchemy.org/trac/ticket/3813)

+   **[postgresql] [bug]**

    > 修复了新的 PG“on conflict”构造中的问题，其中包括“excluded”命名空间的列在语句的 WHERE 子句中不会被表格限定。

    参考：[#3807](https://www.sqlalchemy.org/trac/ticket/3807)

### mysql

+   **[mysql] [bug]**

    增加了对解析 MySQL/Connector 布尔值和整数参数的支持，这些参数在 URL 查询字符串中：connection_timeout、connect_timeout、pool_size、get_warnings、raise_on_warnings、raw、consume_results、ssl_verify_cert、force_ipv6、pool_reset_session、compress、allow_local_infile、use_pure。

    这个更改也被**回溯**到：1.0.15

    参考：[#3787](https://www.sqlalchemy.org/trac/ticket/3787)

+   **[mysql] [bug]**

    修复了在 MySQL 下“literal_binds”标志不会传播到 CAST 表达式的错误。

    引用：[#3766](https://www.sqlalchemy.org/trac/ticket/3766)

### mssql

+   **[mssql] [bug]**

    将用于获取“默认模式名称”的查询更改为使用“schema_name()”函数，而不是查询数据库原则表的查询，因为已经有人报告说前一个系统在 Azure 数据仓库版本上不可用。希望这将最终在所有 SQL Server 版本和身份验证样式上工作。

    此更改也**回溯**至：1.0.16

    引用：[#3810](https://www.sqlalchemy.org/trac/ticket/3810)

+   **[mssql] [bug]**

    更新了 pyodbc 的服务器版本信息方案，使用 SQL Server 的 SERVERPROPERTY()，而不是依赖于 pyodbc.SQL_DBMS_VER，后者在特别是 FreeTDS 中仍然不可靠。

    此更改也**回溯**至：1.0.16

    引用：[#3814](https://www.sqlalchemy.org/trac/ticket/3814)

+   **[mssql] [bug]**

    将错误代码 20017“服务器意外结束”添加到断开连接异常列表中，导致连接池重置。感谢 Ken Robbins 提供的拉取请求。

    此更改也**回溯**至：1.0.16

    引用：[#3791](https://www.sqlalchemy.org/trac/ticket/3791)

### misc

+   **[bug] [orm.declarative]**

    修复了一个错误，设置一个单表继承子类的连接表子类，该连接表子类包含一个额外的列，会破坏映射表的外键集合，从而干扰关系的初始化。

    此更改也**回溯**至：1.0.16

    引用：[#3797](https://www.sqlalchemy.org/trac/ticket/3797)

### orm

+   **[orm] [feature]**

    增强了新的“raise”懒惰加载器策略，还包括一个“raise_on_sql”变体，通过`relationship.lazy`以及`raiseload()`都可以使用。此变体仅在懒加载实际上会发出 SQL 时才引发，而不是在调用懒加载机制时引发。

    引用：[#3812](https://www.sqlalchemy.org/trac/ticket/3812)

+   **[orm] [feature]**

    `Query.group_by()`方法现在如果传递`None`参数，则重置分组集合，就像`Query.order_by()`一直以来的工作方式一样。感谢 Iuri Diniz 提供的拉取请求。

+   **[orm] [change]**

    将 False 传递给`Query.order_by()`以取消所有排序已弃用；使用 False 或 None 调用此方法不再有区别。

+   **[orm] [bug]**

    修复了连接急切加载在多态加载的映射器中失败的错误，其中多态 _on 设置为未映射表达式，例如 CASE 表达式。

    此更改也**回溯**到：1.0.16

    参考：[#3800](https://www.sqlalchemy.org/trac/ticket/3800)

+   **[orm] [bug]**

    修复了对通过`Session.bind_mapper()`、`Session.bind_table()`或构造函数发送到会话的无效绑定引发的 ArgumentError 未能正确引发的错误。

    此更改也**回溯**到：1.0.16

    参考：[#3798](https://www.sqlalchemy.org/trac/ticket/3798)

+   **[orm] [bug]**

    修复了子查询急切加载中的错误，其中一个“of_type()”对象的子查询加载链接到第二个普通映射类的子查询加载，或者更长的几个“of_type()”属性链，将无法正确链接连接。

    此更改也**回溯**到：1.0.15

    参考：[#3773](https://www.sqlalchemy.org/trac/ticket/3773), [#3774](https://www.sqlalchemy.org/trac/ticket/3774)

+   **[orm] [bug]**

    现在可以将 ORM 属性分配给具有`__clause_element__()`属性的任何对象，这将导致内联 SQL 的生成方式与任何`ClauseElement`类一样。这涵盖了其他映射属性，否则不会被进一步的表达式构造转换。

    参考：[#3802](https://www.sqlalchemy.org/trac/ticket/3802)

+   **[orm] [bug]**

    对首次引入的[票号：3431]中的错误修复进行了调整，涉及到一个对象在单个结果集中出现在多个上下文中，因此会触发一个急切加载器，将相关对象值设置为 None，从而满足该属性的加载。之前，该调整只会在次要行中急切加载属性的非 None 值到达时才会生效。

    参考：[#3811](https://www.sqlalchemy.org/trac/ticket/3811)

+   **[orm] [bug]**

    修复了新的`SessionEvents.persistent_to_deleted()`事件中的错误，其中目标对象可能在事件触发之前被垃圾回收。

    参考：[#3808](https://www.sqlalchemy.org/trac/ticket/3808)

+   **[orm] [bug]**

    `relationship()`构造函数的 primaryjoin 现在可以包括一个包含可调用函数以生成值的`bindparam()`对象。以前，惰性加载策略与此用法不兼容，并且还会无法正确检测是否应该使用“use_get”标准，如果主键与绑定参数有关。

    参考：[#3767](https://www.sqlalchemy.org/trac/ticket/3767)

+   **[orm] [bug]**

    从 ORM 刷新过程中发出的 UPDATE 现在可以适应对象主键中的列的 SQL 表达式元素，如果目标数据库支持 RETURNING 以提供新值，或者如果将 PK 值设置为“自身”以用于触发其他触发器/列的 onupdate。

    参考：[#3801](https://www.sqlalchemy.org/trac/ticket/3801)

+   **[orm] [bug]**

    修复了一个 bug，即“简单的一对多”条件允许惰性加载使用来自标识映射的 get()时，如果关系的 primaryjoin 有多个由 AND 分隔的子句，并且这些子句的顺序与每个子句中比较的主键列的顺序不同，则无法调用。这种排序差异发生在复合外键的情况下，其中引用方的表绑定列在.c 集合中的顺序与被引用方的主键列不同……如果使用声明性 mixin 和/或 declared_attr 设置列，则会经常发生这种情况。

    参考：[#3788](https://www.sqlalchemy.org/trac/ticket/3788)

+   **[orm] [bug]**

    当映射上的两个`@validates`装饰器使用相同名称时，会引发异常。一次只支持一个特定名称的验证器，没有机制将它们链接在一起，因为在函数装饰器级别上验证器的顺序无法确定。

    另请参阅

    相同名称的@validates 装饰器现在会引发异常

    参考：[#3776](https://www.sqlalchemy.org/trac/ticket/3776)

+   **[orm] [bug]**

    在`configure_mappers()`期间引发的映射器错误现在在异常消息中明确包含源映射器的名称，以帮助处理那些包装异常本身不包含源映射器的情况。感谢 John Perkins 的拉取请求。

### orm declarative

+   **[orm] [declarative] [change]**

    构建一个继承自另一个类的声明性基类也会继承其文档字符串。这意味着`as_declarative()`的行为更像一个普通的类装饰器。

### sql

+   **[sql] [bug]**

    修复了`Table`中的错误，其中内部方法`_reset_exported()`会破坏对象的状态。此方法旨在用于可选择对象，并在某些情况下由 ORM 调用；错误的映射器配置可能导致 ORM 在`Table`对象上调用此方法。

    此更改也**回溯**到：1.0.15

    参考：[#3755](https://www.sqlalchemy.org/trac/ticket/3755)

+   **[sql] [bug]**

    执行选项现在可以在编译时从语句内传播到最外层语句，因此，如果嵌入元素想要将“autocommit”设置为 True，例如，它可以将此传播到封闭语句。目前，此功能已启用用于嵌入在 SELECT 语句内的面向 DML 的 CTE，例如，在 SELECT 内的 INSERT/UPDATE/DELETE。

    参考：[#3805](https://www.sqlalchemy.org/trac/ticket/3805)

+   **[sql] [bug]**

    通过`Column.server_default`参数作为列默认值发送的字符串现在已经为引号进行了转义。

    另请参见

    String server_default 现在是文字引用

    参考：[#3809](https://www.sqlalchemy.org/trac/ticket/3809)

+   **[sql] [bug] [postgresql]**

    添加了由 PostgreSQL 使用的编译器级别标志，用于在涉及 JSON、HSTORE 索引操作符以及它们的操作数时放置比通常由优先规则生成的括号更多的操作符，因为观察到 PostgreSQL 至少对于 HSTORE 索引操作符的优先规则在 9.4 和 9.5 之间不一致。

    参考：[#3806](https://www.sqlalchemy.org/trac/ticket/3806)

+   **[sql] [bug] [mysql]**

    现在`Connection`的异常处理例程拦截了`BaseException`异常类，并包括`ConnectionEvents.handle_error()`事件的处理。在系统级别异常（不是`Exception`的子类，包括`KeyboardInterrupt`和 greenlet `GreenletExit`类）的情况下，默认情况下现在`Connection`被**使无效**，以防止在处于未知且可能已损坏状态的数据库连接上发生进一步操作。MySQL 驱动程序是此更改的主要目标，但此更改适用于所有 DBAPIs。

    另请参见

    引擎现在使连接无效，运行 BaseException 的错误处理程序

    参考：[#3803](https://www.sqlalchemy.org/trac/ticket/3803)

+   **[sql] [bug]**

    “eq”和“ne”运算符不再是“关联”运算符列表的一部分，尽管它们仍被认为是“可交换的”。这允许像`(x == y) == z`这样的表达式在 SQL 级别上保持括号。感谢 John Passaro 提供的拉取请求。

    参考：[#3799](https://www.sqlalchemy.org/trac/ticket/3799)

+   **[sql] [bug]**

    在许多情况下包括 ORM 错误报告中出现的未命名`Column`对象的表达式字符串化，现在在字符串上下文中将名称呈现为“<name unknown>”��而不是引发编译错误。

    参考：[#3789](https://www.sqlalchemy.org/trac/ticket/3789)

+   **[sql] [bug]**

    当 ClauseElement 或非 SQLAlchemy 对象被错误地传递给`.execute()`时，引发一个更具描述性的异常/消息；在所有情况下一致地引发一个新的异常 ObjectNotExecutableError。

    参考：[#3786](https://www.sqlalchemy.org/trac/ticket/3786)

+   **[sql] [bug] [mysql] [postgresql]**

    修复了 JSON 数据类型中的回归，其中 JSON 索引值的“literal processor”不会被调用的问题。现在从 JSONIndexType 和 JSONPathType 内部调用原生的 String 和 Integer 数据类型。这适用于通用的、PostgreSQL 和 MySQL 的 JSON 类型，还依赖于[#3766](https://www.sqlalchemy.org/trac/ticket/3766)。

    参考：[#3765](https://www.sqlalchemy.org/trac/ticket/3765)

+   **[sql] [bug]**

    修复了`Index`无法从包含在 ORM 风格`__clause_element__()`构造内部的复合 SQL 表达式中提取列的错误。这个 bug 在 1.0.x 中也存在，但在 1.1 中更为明显，因为 hybrid_property @expression 现在返回一个包装元素。

    参考：[#3763](https://www.sqlalchemy.org/trac/ticket/3763)

### postgresql

+   **[postgresql] [bug]**

    调整了 ON CONFLICT，使“inserted_primary_key”逻辑能够适应没有 INSERT 或 UPDATE 且没有净变化的情况。在这种情况下，该值为 None，而不是在异常上失败。

    参考：[#3813](https://www.sqlalchemy.org/trac/ticket/3813)

+   **[postgresql] [bug]**

    > 修复了新的 PG“on conflict”构造中的问题，其中包括“排除”命名空间的列在语句的 WHERE 子句中不会被表格限定。

    参考：[#3807](https://www.sqlalchemy.org/trac/ticket/3807)

### mysql

+   **[mysql] [bug]**

    增加了对解析 MySQL/Connector 布尔值和整数参数的支持，这些参数在 URL 查询字符串中：connection_timeout, connect_timeout, pool_size, get_warnings, raise_on_warnings, raw, consume_results, ssl_verify_cert, force_ipv6, pool_reset_session, compress, allow_local_infile, use_pure。

    这个更改也被**回溯**到：1.0.15

    参考：[#3787](https://www.sqlalchemy.org/trac/ticket/3787)

+   **[mysql] [bug]**

    修复了在 MySQL 下“literal_binds”标志不会传播到 CAST 表达式的 bug。

    参考：[#3766](https://www.sqlalchemy.org/trac/ticket/3766)

### mssql

+   **[mssql] [bug]**

    更改了用于获取“默认模式名称”的查询，从查询数据库主体表的查询更改为使用“schema_name()”函数，因为有报道称前一系统在 Azure Data Warehouse 版本上不可用。希望这将最终在所有 SQL Server 版本和认证样式上都起作用。

    此更改也**回溯**到：1.0.16

    参考：[#3810](https://www.sqlalchemy.org/trac/ticket/3810)

+   **[mssql] [bug]**

    更新了用于 pyodbc 的服务器版本信息方案，使用 SQL Server SERVERPROPERTY()，而不是依赖于 pyodbc.SQL_DBMS_VER，后者在 FreeTDS 中仍然不可靠。

    此更改也**回溯**到：1.0.16

    参考：[#3814](https://www.sqlalchemy.org/trac/ticket/3814)

+   **[mssql] [bug]**

    将错误代码 20017“服务器意外 EOF”添加到导致连接池重置的断开异常列表中。感谢 Ken Robbins 的拉取请求。

    此更改也**回溯**到：1.0.16

    参考：[#3791](https://www.sqlalchemy.org/trac/ticket/3791)

### 杂项

+   **[bug] [orm.declarative]**

    修复了一个 bug，即设置一个包含额外列的联接表子类的单表继承子类会破坏映射表的外键集合，从而干扰关系的初始化。

    此更改也**回溯**到：1.0.16

    参考：[#3797](https://www.sqlalchemy.org/trac/ticket/3797)

## 1.1.0b3

发布日期：2016 年 7 月 26 日

### orm

+   **[orm] [change]**

    移除了一个警告，该警告可以追溯到 0.4 版本，当在通过联接或单表继承进行继承的两个映射器上放置同名关系时会发出警告。该警告不适用于当前的工作单元实现。

    另请参阅

    继承映射器上的同名关系不再警告

    参考：[#3749](https://www.sqlalchemy.org/trac/ticket/3749)

### sql

+   **[sql] [bug]**

    修复了一个新的 CTE 功能中的错误，用于在嵌套语句（通常是 SELECT）中作为 CTE 声明的 update/insert/delete 语句中，嵌入语句中的 oninsert 和 onupdate 值未被调用的问题。

    参考：[#3745](https://www.sqlalchemy.org/trac/ticket/3745)

+   **[sql] [bug]**

    修复了一个新的 CTE 功能中的错误，即在 update/insert/delete 中，围绕语句的匿名（例如未传递名称）`CTE`构造会失败的问题。

    参考：[#3744](https://www.sqlalchemy.org/trac/ticket/3744)

### postgresql

+   **[postgresql] [bug]**

    修复了`TypeDecorator`和`Variant`类型未被 PostgreSQL 方言深度检查以确定是否需要呈现 SMALLSERIAL 或 BIGSERIAL 而不是 SERIAL 的错误。

    这个更改也被**回溯**到：1.0.14

    参考：[#3739](https://www.sqlalchemy.org/trac/ticket/3739)

### oracle

+   **[oracle] [bug]**

    修复了`Select.with_for_update.of`中的错误，其中 Oracle 的“rownum”方法对 LIMIT/OFFSET 无法容纳“OF”子句内的表达式，这些表达式必须在顶层引用子查询中的表达式。如果需要，这些表达式现在将添加到子查询中。

    这个更改也被**回溯**到：1.0.14

    参考：[#3741](https://www.sqlalchemy.org/trac/ticket/3741)

### misc

+   **[feature] [ext]**

    为新的 sqlalchemy.ext.indexable 扩展添加了一个“default”参数。

+   **[bug] [ext]**

    修复了`sqlalchemy.ext.baked`中的错误，其中解除子查询急加载器查询的失败，由于变量作用域问题，当涉及多个子查询加载器时会失败。感谢 Mark Hahnenberg 的拉取请求。

    这个更改也被**回溯**到：1.0.15

    参考：[#3743](https://www.sqlalchemy.org/trac/ticket/3743)

+   **[bug] [ext]**

    sqlalchemy.ext.indexable 将在引发 AttributeError 时拦截 IndexError 和 KeyError。

### orm

+   **[orm] [change]**

    移除了一个警告，该警告可以追溯到 0.4 版本，当一个同名关系被放置在通过联接或单表继承继承的两个映射器上时会发出警告。该警告不适用于当前的工作单元实现。

    另请参阅

    继承映射器上的同名关系不再警告

    参考：[#3749](https://www.sqlalchemy.org/trac/ticket/3749)

### sql

+   **[sql] [bug]**

    修复了新的 CTE 功能中的错误，用于在嵌套语句（通常是 SELECT）中作为 CTE 声明的 update/insert/delete 功能，其中 oninsert 和 onupdate 值未在嵌入语句中调用。

    参考：[#3745](https://www.sqlalchemy.org/trac/ticket/3745)

+   **[sql] [bug]**

    修复了新的 CTE 功能中的错误，其中围绕语句的匿名（例如未传递名称）`CTE`构造将失败。

    参考：[#3744](https://www.sqlalchemy.org/trac/ticket/3744)

### postgresql

+   **[postgresql] [bug]**

    修复了`TypeDecorator`和`Variant`类型未被 PostgreSQL 方言深度检查以确定是否需要渲染 SMALLSERIAL 或 BIGSERIAL 而不是 SERIAL 的 bug。

    此更改也被**回溯**到：1.0.14

    参考：[#3739](https://www.sqlalchemy.org/trac/ticket/3739)

### oracle

+   **[oracle] [bug]**

    修复了`Select.with_for_update.of`中的 bug，在 Oracle 的“rownum”方法中，LIMIT/OFFSET 无法适应“OF”子句中的表达式，这些表达式必须在子查询中的最顶层引用表达式。如果需要，这些表达式现在将添加到子查询中。

    此更改也被**回溯**到：1.0.14

    参考：[#3741](https://www.sqlalchemy.org/trac/ticket/3741)

### 杂项

+   **[feature] [ext]**

    为新的 sqlalchemy.ext.indexable 扩展添加了一个“default”参数。

+   **[bug] [ext]**

    修复了`sqlalchemy.ext.baked`中的 bug，在涉及多个子查询加载器时，解除子查询贪婪加载器查询的问题由于变量作���域问题而失败。感谢 Mark Hahnenberg 的拉取请求。

    此更改也被**回溯**到：1.0.15

    参考：[#3743](https://www.sqlalchemy.org/trac/ticket/3743)

+   **[bug] [ext]**

    sqlalchemy.ext.indexable 在引发 AttributeError 时也会拦截 IndexError 和 KeyError。

## 1.1.0b2

发布日期：2016 年 7 月 1 日

### sql

+   **[sql] [bug]**

    修复了 SQL 数学否定运算符中的问题，其中表达式的类型将不再是原始的数值类型。这会导致确定结果集行为的问题。

    此更改也被**回溯**到：1.0.14

    参考：[#3735](https://www.sqlalchemy.org/trac/ticket/3735)

+   **[sql] [bug]**

    修复了 sqlalchemy.util.Properties 的`__getstate__` / `__setstate__`方法由于 1.0 系列向`__slots__`的过渡而无法工作的 bug。该问题可能会影响一些第三方应用程序。感谢 Pieter Mulder 的拉取请求。

    此更改也被**回溯**到：1.0.14

    参考：[#3728](https://www.sqlalchemy.org/trac/ticket/3728)

+   **[sql] [bug]**

    `Boolean`数据类型在仅具有整数类型的后端上的处理已经在纯 Python 和 C 扩展版本之间保持一致，C 扩展版本将接受数据库中的任何整数值作为布尔值，而不仅仅是零和一；此外，发送到数据库的非布尔整数值将被强制转换为零或一，而不是作为原始整数值传递。

    另请参阅

    非本地布尔整数值在所有情况下被强制转换为零/一/无

    引用：[#3730](https://www.sqlalchemy.org/trac/ticket/3730)

+   **[sql] [bug]**

    在 `Enum` 中将验证规则稍微回滚，以允许未知字符串值通过，除非将标志 `validate_string=True` 传递给 Enum；当然，任何其他类型的对象仍然被拒绝。虽然立即使用是允许与 LIKE 的枚举比较，但存在此使用表明可能有更多未知字符串比较用例，这暗示可能存在一些未知字符串插入用例。

    引用：[#3725](https://www.sqlalchemy.org/trac/ticket/3725)

### postgresql

+   **[postgresql] [bug] [ext]**

    在 `sqlalchemy.ext.compiler` 扩展中进行了轻微的行为更改，其中已建立的构造的现有编译方案将被移除，如果该构造本身尚未拥有专用的 `__visit_name__`。这在 1.0 中很少见，但是在 1.1 中，`ARRAY` 子类 `ARRAY` 并具有此行为。结果，为其他方言（例如 SQLite）设置编译处理程序将使主 `ARRAY` 对象不再可编译。

    引用：[#3732](https://www.sqlalchemy.org/trac/ticket/3732)

### mysql

+   **[mysql] [bug]**

    在 不再生成带 AUTO_INCREMENT 的复合主键的隐式 KEY 中略微调整了“按自动增量排序主键列”的描述，因此，如果 `PrimaryKeyConstraint` 被明确定义，列的顺序将完全保持不变，允许在必要时控制此行为。

    引用：[#3726](https://www.sqlalchemy.org/trac/ticket/3726)

### sql

+   **[sql] [bug]**

    修复了 SQL 数学否定运算符中的问题，其中表达式的类型将不再是原始表达式的数字类型。这将导致类型确定结果集行为的问题。

    此更改也 **回溯** 到：1.0.14

    引用：[#3735](https://www.sqlalchemy.org/trac/ticket/3735)

+   **[sql] [bug]**

    修复了 sqlalchemy.util.Properties 的 `__getstate__` / `__setstate__` 方法由于 1.0 系列向 `__slots__` 过渡而无法正常工作的错误。该问题可能会影响一些第三方应用程序。感谢 Pieter Mulder 的拉取请求。

    此更改也 **回溯** 到：1.0.14

    引用：[#3728](https://www.sqlalchemy.org/trac/ticket/3728)

+   **[sql] [bug]**

    仅具有整数类型的后端执行的`Boolean`数据类型的处理在纯 Python 和 C 扩展版本之间已经保持一致，即 C 扩展版本将接受数据库中的任何整数值作为布尔值，而不仅仅是零和一；此外，发送到数据库的非布尔整数值被强制转换为零或一，而不是作为原始整数值传递。

    另请参阅

    所有情况下将非本地布尔整数值强制转换为零/一/None

    参考：[#3730](https://www.sqlalchemy.org/trac/ticket/3730)

+   **[sql] [bug]**

    在`Enum`中稍微回退了验证规则，允许未知字符串值通过，除非向 Enum 传递了标志`validate_string=True`；当然，任何其他类型的对象仍然被拒绝。虽然立即使用是为了允许与 LIKE 一起比较枚举，但存在这种用法表明可能存在更多未知字符串比较用例，这暗示可能还有一些未知字符串插入用例。

    参考：[#3725](https://www.sqlalchemy.org/trac/ticket/3725)

### postgresql

+   **[postgresql] [bug] [ext]**

    在`sqlalchemy.ext.compiler`扩展中对行为进行了轻微更改，如果已建立的结构本身没有专用的`__visit_name__`，则会删除该结构的现有编译方案。这在 1.0 中很少见，但在 1.1 中`ARRAY`子类`ARRAY`并具有此行为。因此，为其他方言（如 SQLite）设置编译处理程序将导致主`ARRAY`对象不再可编译。

    参考：[#3732](https://www.sqlalchemy.org/trac/ticket/3732)

### mysql

+   **[mysql] [bug]**

    在不再为具有 AUTO_INCREMENT 的复合主键生成隐式 KEY 中稍微调整了“按自动增量排序主键列”的描述，因此如果明确定义了`PrimaryKeyConstraint`，则列的顺序将保持完全一致，允许在必要时控制此行为。

    参考：[#3726](https://www.sqlalchemy.org/trac/ticket/3726)

## 1.1.0b1

发布日期：2016 年 6 月 16 日

### orm

+   **[orm] [feature] [ext]**

    新增 ORM 扩展 Indexable，允许构建 Python 属性，这些属性引用“索引”结构的特定元素，如数组和 JSON 字段。感谢 Jeong YunWon 提交的拉取请求。

    另请参阅

    新的 Indexable ORM 扩展

+   **[orm] [功能]**

    新增标志 `Session.bulk_insert_mappings.render_nulls`，允许 ORM 批量插入时渲染 NULL 值；这将绕过服务器端默认值，但允许所有语句使用相同的列集合，从而可以进行批处理。感谢 Tobias Sauerwein 提交的拉取请求。

+   **[orm] [功能]**

    新增事件 `AttributeEvents.init_scalar()`，以及一个新的示例套件，说明其用法。此事件可用于在对象持久化之前为 Python 端属性提供 Core 生成的默认值。

    另请参阅

    ORM 层面新增 init_scalar() 事件拦截默认值

    参考：[#1311](https://www.sqlalchemy.org/trac/ticket/1311)

+   **[orm] [功能]**

    在 `AutomapBase.prepare()` 方法中新增 `AutomapBase.prepare.schema`，用于指示应从哪个模式反射表，如果不是默认模式。感谢 Josh Marlow 提交的拉取请求。

+   **[orm] [功能]**

    新增参数 `mapper.passive_deletes` 到可用的映射器选项中。这允许针对基表进行 DELETE 操作，而允许 ON DELETE CASCADE 处理从子类表中删除行。

    另请参阅

    joined-inheritance 映射的 passive_deletes 功能

    参考：[#2349](https://www.sqlalchemy.org/trac/ticket/2349)

+   **[orm] [功能]**

    当对核心 SQL 构造调用 str() 时，如果构造包含非标准 SQL 元素，如 RETURNING、数组索引操作或方言特定或自定义数据类型，则现在更加“友好”。在这些情况下，将返回一个字符串，呈现构造的近似值（通常是其 PostgreSQL 风格的版本），而不是引发错误。

    另请参阅

    不带方言的 Core SQL 构造的“友好”字符串化

    参考：[#3631](https://www.sqlalchemy.org/trac/ticket/3631)

+   **[orm] [功能]**

    对于`Query`的`str()`调用现在将考虑绑定到`Session`的`Engine`，在生成 SQL 的字符串形式时，如果可能的话，显示将要发送到数据库的实际 SQL。以前，只有与映射相关联的`MetaData`关联的引擎会被使用，如果存在的话。如果无法在`Session`或与映射相关联的`MetaData`上找到绑定，则将使用“默认”方言来呈现 SQL，就像以前一样。

    另请参阅

    查询的字符串化将查询会话的正确方言

    参考：[#3081](https://www.sqlalchemy.org/trac/ticket/3081)

+   **[orm] [feature]**

    `SessionEvents`套件现在包括事件，允许明确跟踪所有对象生命周期状态转换，例如`Session`本身，如挂起、瞬态、持久、分离。还定义了每个事件中对象的状态。

    另请参阅

    新的会话生命周期事件

    参考：[#2677](https://www.sqlalchemy.org/trac/ticket/2677)

+   **[orm] [feature]**

    添加了一个新的会话生命周期状态 deleted。这个新状态表示一个已经从 persistent 状态中删除的对象，一旦事务提交，它将转移到 detached 状态。这解决了长期存在的问题，即被删除的对象存在于持久和分离之间的灰色区域。`InstanceState.persistent`访问器将**不再**报告已删除对象为持久；相反，这些对象的`InstanceState.deleted`访问器将为 True，直到它们变为分离状态。

    另请参阅

    新的会话生命周期事件

    参考：[#2677](https://www.sqlalchemy.org/trac/ticket/2677)

+   **[orm] [feature]**

    添加了新的检查，用于常见错误情况，即将映射类或映射实例传递到被解释为 SQL 绑定参数的上下文中；为此将引发新异常。

    另请参阅

    为传递映射类和实例添加了特定检查作为 SQL 字面值

    参考：[#3321](https://www.sqlalchemy.org/trac/ticket/3321)

+   **[orm] [feature]**

    添加了新的关系加载策略`raiseload()`（也可通过`lazy='raise'`访问）。此策略的行为几乎与`noload()`相同，但不是返回`None`而是引发一个 InvalidRequestError。拉取请求由 Adrian Moennich 提供。

    另请参阅

    新的“raise”/“raise_on_sql”加载策略

    参考：[#3512](https://www.sqlalchemy.org/trac/ticket/3512)

+   **[orm] [change]**

    `Mapper.order_by`参数已弃用。这是一个旧参数，与 SQLAlchemy 的工作方式不再相关，一旦引入了 Query 对象。通过弃用它，我们确立了我们不支持不起作用的用例，并鼓励应用程序停止使用此参数。

    另请参阅

    Mapper.order_by 已弃用

    参考：[#3394](https://www.sqlalchemy.org/trac/ticket/3394)

+   **[orm] [change]**

    `Session.weak_identity_map`参数已弃用。请参阅 Session Referencing Behavior 中的新配方，了解基于事件的维护强引用映射行为的方法。

    另请参阅

    新的会话生命周期事件

    参考：[#2677](https://www.sqlalchemy.org/trac/ticket/2677)

+   **[orm] [bug]**

    修复了一个问题，即将对象从一个父对象改为另一个父对象的多对一更改在与未刷新的外键属性修改结合时可能工作不一致的情况。现在，属性移动考虑了外键的数据库提交值，以定位正在移动的对象的“先前”父对象。这样可以正确触发事件，包括反向引用事件。以前，这些事件并不总是触发。可能依赖于先前破损行为的应用可能会受到影响。

    另请参阅

    涉及用户启动的外键操作的多对一对象移动的修复

    参考：[#3708](https://www.sqlalchemy.org/trac/ticket/3708)

+   **[orm] [bug]**

    修复了延迟列在对象被合并到会话中时不经意地设置为数据库加载的错误，该对象在下一个对象范围的未过期时，通过`session.merge(obj, load=False)`合并到会话中。

    参考：[#3488](https://www.sqlalchemy.org/trac/ticket/3488)

+   **[orm] [bug] [mysql]**

    进一步延续了最初在[#2696](https://www.sqlalchemy.org/trac/ticket/2696)中涵盖的常见 MySQL 异常情况，即在回滚之前 SAVEPOINT 被取消的情况下，`Session`的故障模式已经改进，以允许 `Session` 仍然在那个 SAVEPOINT 之外正常运行。假设保存点操作失败并已取消。

    另请参阅

    取消数据库 SAVEPOINT 时改进的会话状态

    参考：[#3680](https://www.sqlalchemy.org/trac/ticket/3680)

+   **[orm] [bug]**

    修复了一个错误，即如果回滚了一个新插入的实例，由于实例未检查是否已过期，因此仍可能在下一个事务中导致持久性冲突。此修复将解决大量错误地导致“新实例 X 的标识与持久实例 Y 冲突”的错误的情况。

    另请参阅

    修复了错误的“新实例 X 与持久实例 Y 冲突”的 flush 错误

    参考：[#3677](https://www.sqlalchemy.org/trac/ticket/3677)

+   **[orm] [bug]**

    对`Query.correlate()`的改进，当使用代表多个表直接连接的“多态”实体时，语句将确保连接中的所有表都是相关的。

    另请参阅

    使用多态实体改进 Query.correlate 方法

    参考：[#3662](https://www.sqlalchemy.org/trac/ticket/3662)

+   **[orm] [bug]**

    修复了一个错误，该错误可能导致急切加载的多对一属性未加载，如果连接的急切加载来自同一个实体在多个位置出现的行，则一些位置需要急切加载属性，而其他位置不需要。这里的逻辑已经修改，以便即使不同的加载路径已经处理了父实体，也要接受该属性。

    另请参阅

    在一行中多次出现相同实体的连接急切加载

    参考：[#3431](https://www.sqlalchemy.org/trac/ticket/3431)

+   **[orm] [bug]**

    对于`Query.distinct()`与`Query.order_by()`组合时向生成的 SQL 添加列的逻辑进行了优化，以便已经存在的列不会被第二次添加，即使它们使用不同的名称标记。无论这种更改，SQL 中添加的额外列从未在最终结果中返回，因此这种更改仅影响语句的字符串形式以及在核心执行上下文中使用时的行为。此外，当使用 DISTINCT ON 格式时，如果查询由于连接的急加载而不是包装在子查询中，则不再添加列。

    另请参阅

    使用 DISTINCT + ORDER BY 时不再冗余添加列

    参考：[#3641](https://www.sqlalchemy.org/trac/ticket/3641)

+   **[orm] [bug]**

    修复了两个同名关系引用基类和具体继承子类时，如果这些关系是使用“backref”设置的，会引发错误的问题，而使用 relationship()设置相同配置并使用冲突名称则会成功，这在具体映射的情况下是允许的。

    另请参阅

    应用于具体继承子类时，同名 backrefs 不会引发错误

    参考：[#3630](https://www.sqlalchemy.org/trac/ticket/3630)

+   **[orm] [bug]**

    `Session.merge()` 方法现在在发出 INSERT 之前通过主键跟踪待处理对象，并在遇到重复主键的不同对象时将其合并在一起，这在最好的情况下基本上是半确定性的。这种行为与持久对象的情况已经匹配。

    另请参阅

    Session.merge 解决待处理冲突与持久对象相同

    参考：[#3601](https://www.sqlalchemy.org/trac/ticket/3601)

+   **[orm] [bug]**

    修复了在某些不适当情况下，例如从单继承子类的 exists()查询时，会将“单表继承”条件添加到查询末尾的错误。

    另请参阅

    对单表继承查询的进一步修复

    参考：[#3582](https://www.sqlalchemy.org/trac/ticket/3582)

+   **[orm] [bug]**

    添加了一个新的类型级修饰符`TypeEngine.evaluates_none()`，指示 ORM 应将一组正值的 None 持久化为值 NULL，而不是在 INSERT 语句中省略列。此功能既用作[#3514](https://www.sqlalchemy.org/trac/ticket/3514)的实现的一部分，也可作为任何类型上可用的独立功能。

    另请参阅

    新增选项允许显式持久化 NULL 而不是默认值

    参考：[#3250](https://www.sqlalchemy.org/trac/ticket/3250)

+   **[orm] [bug]**

    在`Session.bulk_save_objects()`和相关批量方法中的“簿记”函数的内部调用已经减少到目前未使用此功能的程度，例如在 INSERT 或 UPDATE 语句之后获取列默认值的检查。

    参考：[#3526](https://www.sqlalchemy.org/trac/ticket/3526)

+   **[orm] [bug] [postgresql]**

    关于 PostgreSQL `JSON` 类型中`None`值的额外修复已经完成。当`JSON.none_as_null`标志保持默认值`False`时，ORM 现在将正确地在 ORM 对象上的值设置为`None`或在使用`Session.bulk_insert_mappings()`时将 JSON“‘null’”字符串插入到列中，**包括**如果列上有默认值或服务器默认值。

    另请参阅

    ORM 操作中 JSON 的“null”如预期般插入，当不存在时被省略

    新增选项允许显式持久化 NULL 而不是默认值

    参考：[#3514](https://www.sqlalchemy.org/trac/ticket/3514)

### 引擎

+   **[engine] [feature]**

    添加了连接池事件`ConnectionEvents.close()`，`ConnectionEvents.detach()`，`ConnectionEvents.close_detached()`。

+   **[engine] [feature]**

    现在，用于日志记录、异常和`repr()`目的的所有绑定参数集和结果行的字符串格式化都会在每个集合中截断非常大的标量值，包括“N 个字符被截断”的标记，类似于大型多参数集的显示本身被截断的方式。

    另请参阅

    日志和异常显示中现在截断大型参数和行值

    参考：[#2837](https://www.sqlalchemy.org/trac/ticket/2837)

+   **[engine] [feature]**

    为 `Table` 对象添加了多租户模式翻译。这支持应用程序在许多模式中使用相同的 `Table` 对象的用例，例如每用户一个模式的模式。增加了一个新的执行选项 `Connection.execution_options.schema_translate_map`。

    另请参阅

    Table 对象的多租户模式翻译

    参考文献：[#2685](https://www.sqlalchemy.org/trac/ticket/2685)

+   **[engine] [特性]**

    在引擎中增加了一个新的入口系统，允许在 URL 的查询字符串中声明“插件”。可以编写自定义插件，这些插件将被提前给予修改和/或使用引擎的 URL 和关键字参数的机会，然后在引擎创建时将被给予引擎本身以允许额外的修改或事件注册。插件被编写为 `CreateEnginePlugin` 的子类；详见该类的详细信息。

    参考文献：[#3536](https://www.sqlalchemy.org/trac/ticket/3536)

### sql

+   **[sql] [特性]**

    通过新的 `FromClause.tablesample()` 方法和独立函数添加了 TABLESAMPLE 支持。感谢 Ilja Everilä 的 Pull 请求。

    另请参阅

    TABLESAMPLE 支持

    参考文献：[#3718](https://www.sqlalchemy.org/trac/ticket/3718)

+   **[sql] [特性]**

    增加了对窗口函数中范围的支持，使用 `over.range_` 和 `over.rows` 参数。

    另请参阅

    窗口函数内的 RANGE 和 ROWS 规范支持

    参考文献：[#3049](https://www.sqlalchemy.org/trac/ticket/3049)

+   **[sql] [特性]**

    实现了 SQLite 和 PostgreSQL 的 CHECK 约束反射。通过新的检查器方法 `Inspector.get_check_constraints()` 以及在反映 `Table` 对象时以 `CheckConstraint` 对象的形式存在于约束集合中。感谢 Alex Grönholm 的 Pull 请求。

+   **[sql] [特性]**

    新增 `ColumnOperators.is_distinct_from()` 和 `ColumnOperators.isnot_distinct_from()` 操作符；感谢 Sebastian Bank 提交的拉取请求。

    另请参阅

    IS DISTINCT FROM 和 IS NOT DISTINCT FROM 的支持

+   **[sql] [feature]**

    在 `DDLCompiler.visit_create_table()` 中添加了名为 `DDLCompiler.create_table_suffix()` 的钩子，允许自定义方言在 “CREATE TABLE” 子句之后添加关键字。感谢 Mark Sandan 提交的拉取请求。

+   **[sql] [feature]**

    `ResultProxy` 返回的行现在支持负整数索引。感谢 Emanuele Gaifas 提交的拉取请求。

    另请参阅

    核心结果行可容纳负整数索引

+   **[sql] [feature]**

    添加 `Select.lateral()` 和相关构造，以支持 SQL 标准的 LATERAL 关键字，目前仅受 PostgreSQL 支持。

    另请参阅

    SQL LATERAL 关键字的支持

    参考：[#2857](https://www.sqlalchemy.org/trac/ticket/2857)

+   **[sql] [feature]**

    添加对 “FULL OUTER JOIN” 的渲染支持，包括 Core 和 ORM。感谢 Stefan Urbanek 提交的拉取请求。

    另请参阅

    核心和 ORM 对 FULL OUTER JOIN 的支持

    参考：[#1957](https://www.sqlalchemy.org/trac/ticket/1957)

+   **[sql] [feature]**

    CTE 功能已扩展以支持所有 DML，允许 INSERT、UPDATE 和 DELETE 语句指定自己的 WITH 子句，以及在这些语句包含 RETURNING 子句时，这些语句本身也可以是 CTE 表达式。

    另请参阅

    CTE 支持 INSERT、UPDATE、DELETE

    参考：[#2551](https://www.sqlalchemy.org/trac/ticket/2551)

+   **[sql] [feature]**

    添加对 PEP-435 样式的枚举类的支持，包括 Python 3 的 `enum.Enum` 类，以及兼容的枚举库，到 `Enum` 数据类型中。`Enum` 数据类型现在还在 Python 中对传入的值进行验证，并添加了一个选项来跳过创建 CHECK 约束 `Enum.create_constraint`. 感谢 Alex Grönholm 提交的拉取请求。

    另请参阅

    对 Python 原生枚举类型及兼容形式的支持

    Enum 类型现在对值进行了在 Python 中的验证

    参考：[#3095](https://www.sqlalchemy.org/trac/ticket/3095), [#3292](https://www.sqlalchemy.org/trac/ticket/3292)

+   **[sql] [feature]**

    最近新增的 `TextClause.columns()` 方法及其与结果行处理的交互得到了深度改进，现在允许该方法传递的列与语句中的结果列按位置匹配，而不是仅匹配名称。这样做的好处包括，在将文本 SQL 语句链接到 ORM 或 Core 表模型时，无需进行常见列名称的标记或去重处理，这也意味着不需要担心标签名称与 ORM 列的匹配等问题。此外，`ResultProxy` 在某些情况下进一步增强，以更精确地将列和字符串键映射到行。

    另见

    ResultSet 列匹配增强；文本 SQL 的位置列设置 - 特性概述

    TextClause.columns() 现在会按位置匹配列，而不是按名称匹配 - 向后兼容说明

    参考：[#3501](https://www.sqlalchemy.org/trac/ticket/3501)

+   **[sql] [feature]**

    核心新增了一个新类型 `JSON`。这是 PostgreSQL `JSON` 类型以及新的 `JSON` 类型的基础，因此可以使用 PG/MySQL 通用的 JSON 列。该类型具有基本的索引和路径搜索支持。

    另见

    核心添加了 JSON 支持

    参考：[#3619](https://www.sqlalchemy.org/trac/ticket/3619)

+   **[sql] [feature]**

    添加了对“set-aggregate”函数的支持，格式为 `<function> WITHIN GROUP (ORDER BY <criteria>)`，使用 `FunctionElement.within_group()` 方法。已添加了一系列常见的 set-aggregate 函数，返回类型派生自集合。这包括诸如 `percentile_cont`、`dense_rank` 等函数。

    另见

    新增函数特性，“WITHIN GROUP”、“array_agg” 和集合聚合函数

    参考：[#1370](https://www.sqlalchemy.org/trac/ticket/1370)

+   **[sql] [feature] [postgresql]**

    添加了对 SQL 标准函数`array_agg`的支持，它会自动返回正确类型的`ARRAY`并支持索引/切片操作，以及`array_agg()`，它返回一个带有额外比较功能的`ARRAY`。由于目前只有 PostgreSQL 支持数组，因此只能在 PostgreSQL 上实际运行。还添加了一个新的构造`aggregate_order_by`以支持 PG 的“ORDER BY”扩展。

    另请参阅

    新功能特性，“WITHIN GROUP”，array_agg 和 set 聚合函数

    参考：[#3132](https://www.sqlalchemy.org/trac/ticket/3132)

+   **[sql] [feature]**

    在核心中添加了一个新类型`ARRAY`。这是 PostgreSQL `ARRAY` 类型的基础，现在已经成为核心的一部分，开始支持各种支持 SQL 标准数组的功能，包括一些函数以及对其他具有“数组”概念的数据库（如 DB2 或 Oracle）的原生数组的支持。此外，还添加了新的运算符`any_()`和`all_()`。这些不仅支持 PostgreSQL 上的数组构造，还支持可在 MySQL 上使用的子查询（但遗憾的是在 PostgreSQL 上不支持）。

    另请参阅

    核心添加了数组支持；新增 ANY 和 ALL 运算符

    参考：[#3516](https://www.sqlalchemy.org/trac/ticket/3516)

+   **[sql] [change] [mysql]**

    `Column` 认为自己是“自增”列的系统已更改，因此不再为具有复合主键的 `Table` 隐式启用 autoincrement。为了能够为复合主键成员列启用自增，同时保持 SQLAlchemy 长期以来启用单个整数主键的隐式自增的行为，已向 `Column.autoincrement` 参数添加了第三状态 `"auto"`，现在这是默认值。

    另请参阅

    不再为复合主键列隐式启用 .autoincrement 指令

    不再为具有 AUTO_INCREMENT 的复合主键生成隐式 KEY

    参考：[#3216](https://www.sqlalchemy.org/trac/ticket/3216)

+   **[sql] [bug]**

    `FromClause.count()` 已弃用。此函数使用表中的任意列，不可靠；对于核心使用，应优先使用 `func.count()`。

    参考：[#3724](https://www.sqlalchemy.org/trac/ticket/3724)

+   **[sql] [bug]**

    修复了一个断言，如果 `Index` 与一个小写的 `TableClause` 关联，而该 `Index` 又与一个 `Column` 关联，则会不太恰当地引发；为了将索引与 `Table` 关联起来，应忽略该关联。

    参考：[#3616](https://www.sqlalchemy.org/trac/ticket/3616)

+   **[sql] [bug]**

    `type_coerce()` 构造现在是一个完全成熟的核心表达式元素，在编译时进行延迟评估。先前，该函数只是一个转换函数，通过返回一个 `Label` 或列定向表达式的复制 `BindParameter` 对象，处理不同的表达式输入，特别是当 ORM 级别的表达式转换将列转换为绑定参数时（例如用于惰性加载），这会阻止该操作在逻辑上保持不变。

    另请参阅

    type_coerce 函数现在是持久的 SQL 元素

    参考：[#3531](https://www.sqlalchemy.org/trac/ticket/3531)

+   **[sql] [bug]**

    现在，`TypeDecorator` 类型扩展器将与 `SchemaType` 实现一起工作，通常是 `Enum` 或 `Boolean`，以确保从实现类型传播到外部类型的每个表事件。这些事件用于确保约束或 PostgreSQL 类型（例如 ENUM）与父表一起正确创建（可能删除）。

    亦参见

    TypeDecorator 现在自动与 Enum、Boolean、“schema”类型配合工作

    参考：[#2919](https://www.sqlalchemy.org/trac/ticket/2919)

+   **[sql] [bug]**

    `union()` 构造和相关构造的行为，例如 `Query.union()` 现在处理嵌入的 SELECT 语句需要括号的情况，因为它们包含 LIMIT、OFFSET 和/或 ORDER BY。这些查询在 SQLite 上**不适用**，并且在该后端上像以前一样失败，但现在应该在所有其他后端上工作。

    亦参见

    UNION 或类似的 SELECT 使用 LIMIT/OFFSET/ORDER BY 现在会给嵌入式 SELECT 添加括号

    参考：[#2528](https://www.sqlalchemy.org/trac/ticket/2528)

### 模式

+   **[schema] [enhancement]**

    现在，传递给 `Column` 对象的默认生成函数现在通过“update_wrapper”或等效的函数运行，如果传递的是一个可调用但不是函数的对象，以便内省工具保留包装函数的名称和文档字符串。拉取请求由 hsum 提供。

### postgresql

+   **[postgresql] [feature]**

    添加了对 PostgreSQL 的 INSERT..ON CONFLICT 的支持，使用了一个新的 PostgreSQL 特定的 `Insert` 对象。拉取请求和 Robin Thomas 的大量工作。

    亦参见

    支持 INSERT..ON CONFLICT (DO UPDATE | DO NOTHING)

    参考：[#3529](https://www.sqlalchemy.org/trac/ticket/3529)

+   **[postgresql] [feature]**

    如果在 `Index` 上设置了 `postgresql_concurrently` 标志，并且检测到正在使用的数据库为 PostgreSQL 版本 9.2 或更高，则 DROP INDEX 的 DDL 将发出 “CONCURRENTLY”。对于 CREATE INDEX，还添加了数据库版本检测，如果 PG 版本小于 8.2，则将省略该子句。感谢 Iuri de Silvio 的 Pull request。

+   **[postgresql] [功能]**

    添加了新参数 `PGInspector.get_view_names.include`，允许指定应返回哪种视图。目前包括 “plain” 和 “materialized” 视图。感谢 Sebastian Bank 的 Pull request。

    参考：[#3588](https://www.sqlalchemy.org/trac/ticket/3588)

+   **[postgresql] [功能]**

    添加了 `postgresql_tablespace` 作为 `Index` 的参数，允许在 PostgreSQL 中为索引指定 TABLESPACE。与 `Table` 上的同名参数相辅相成。感谢 Benjamin Bertrand 的 Pull request。

    参考：[#3720](https://www.sqlalchemy.org/trac/ticket/3720)

+   **[postgresql] [功能]**

    添加了新参数 `GenerativeSelect.with_for_update.key_share`，它将在 PostgreSQL 后端上呈现 `FOR NO KEY UPDATE` 版本的 `FOR UPDATE` 和 `FOR KEY SHARE`，而不是 `FOR SHARE`。感谢 Sergey Skopin 的 Pull request。

+   **[postgresql] [功能] [oracle]**

    添加了新参数 `GenerativeSelect.with_for_update.skip_locked`，它将在 PostgreSQL 和 Oracle 后端上为 `FOR UPDATE` 或 `FOR SHARE` 锁呈现 `SKIP LOCKED` 短语。感谢 Jack Zhou 的 Pull request。

+   **[postgresql] [功能]**

    添加了 PyGreSQL PostgreSQL 方言的新方言。感谢 Christoph Zwerschke 和 Kaolin Imago Fire 的努力。

+   **[postgresql] [功能]**

    添加了新的常量 `JSON.NULL`，表示应该使用 JSON NULL 值作为值，而不考虑其他设置。

    另请参阅

    新增 JSON.NULL 常量

    参考：[#3514](https://www.sqlalchemy.org/trac/ticket/3514)

+   **[postgresql] [变更]**

    `sqlalchemy.dialects.postgres` 模块，长期弃用，已被移除；多年来一直发出警告，项目应该调用 `sqlalchemy.dialects.postgresql`。然而，形式为 `postgres://` 的 Engine URLs 仍将继续运行。

+   **[postgresql] [错误]**

    增加了对将物化视图源反映到 PostgreSQL 版本的`Inspector.get_view_definition()`方法的支持。

    参考：[#3587](https://www.sqlalchemy.org/trac/ticket/3587)

+   **[postgresql] [bug]**

    使用引用`ARRAY`对象，指向`Enum`或`ENUM`子类型时，现在在类型在“CREATE TABLE”或“DROP TABLE”中使用时，将会发出预期的“CREATE TYPE”和“DROP TYPE” DDL。

    另请参阅

    ARRAY with ENUM will now emit CREATE TYPE for the ENUM

    参考：[#2729](https://www.sqlalchemy.org/trac/ticket/2729)

+   **[postgresql] [bug]**

    特殊数据类型（如`ARRAY`、`JSON`和`HSTORE`）上的“可哈希”标志现在设置为 False，这允许在包含行内实体的 ORM 查询中获取这些类型。

    另请参阅

    关于“不可哈希”类型的更改，影响 ORM 行的去重

    ARRAY and JSON types now correctly specify “unhashable”

    参考：[#3499](https://www.sqlalchemy.org/trac/ticket/3499)

+   **[postgresql] [bug]**

    PostgreSQL `ARRAY` 类型现在支持多维索引访问，例如表达式`somecol[5][6]`，无需进行显式转换或类型强制，只要`ARRAY.dimensions`参数设置为所需的维数即可。

    另请参阅

    Correct SQL Types are Established from Indexed Access of ARRAY, JSON, HSTORE

    参考：[#3487](https://www.sqlalchemy.org/trac/ticket/3487)

+   **[postgresql] [bug]**

    当使用索引访问时，`JSON` 和 `JSONB` 的返回类型已经修复，使其与 PostgreSQL 自身一样工作，并返回一个表达式，该表达式本身是 `JSON` 或 `JSONB` 类型。以前，访问器会返回 `NullType`，这会禁止后续使用类似 JSON 的操作符。

    另请参见

    已从数组、JSON、HSTORE 的索引访问中正确确定 SQL 类型

    参考文献：[#3503](https://www.sqlalchemy.org/trac/ticket/3503)

+   **[postgresql] [bug]**

    `JSON`、`JSONB` 和 `HSTORE` 数据类型现在允许通过 `JSON.astext_type` 和 `HSTORE.text_type` 参数对从索引文本访问操作的返回类型进行完全控制，例如 JSON 类型为 `column[someindex].astext` 或 HSTORE 类型为 `column[someindex]`。

    另请参见

    已从数组、JSON、HSTORE 的索引访问中正确确定 SQL 类型

    参考文献：[#3503](https://www.sqlalchemy.org/trac/ticket/3503)

+   **[postgresql] [bug]**

    `Comparator.astext` 修饰符不再隐式调用 `ColumnElement.cast()`，因为 PG 的 JSON/JSONB 类型允许彼此之间的交叉转换。对 JSON 索引访问上使用 `ColumnElement.cast()` 的代码，例如 `col[someindex].cast(Integer)`，需要显式调用 `Comparator.astext`。

    另请参见

    现在对 JSON cast()操作需要显式调用.astext

    参考：[#3503](https://www.sqlalchemy.org/trac/ticket/3503)

### mysql

+   **[mysql] [功能]**

    通过 AUTOCOMMIT 隔离级别设置，为 MySQL 驱动程序添加了对“autocommit”的支持。感谢 Roman Podoliaka 的拉取请求。

    另请参阅

    增加了对 AUTOCOMMIT“隔离级别”的支持

    参考：[#3332](https://www.sqlalchemy.org/trac/ticket/3332)

+   **[mysql] [功能]**

    为 MySQL 5.7 添加了`JSON`。JSON 类型在 MySQL 中提供了 JSON 值的持久性以及“getitem”和“getpath”的基本操作支持，利用`JSON_EXTRACT`函数来引用 JSON 结构中的单个路径。

    另请参阅

    MySQL JSON 支持

    参考：[#3547](https://www.sqlalchemy.org/trac/ticket/3547)

+   **[mysql] [更改]**

    当使用 InnoDB 在具有复合主键的表上生成 CREATE TABLE DDL 时，MySQL 方言不再生成额外的“KEY”指令；为了克服 InnoDB 在此处的限制，现在会在列的列表中首先放置 AUTO_INCREMENT 列来生成 PRIMARY KEY 约束。

    另请参阅

    不再为具有 AUTO_INCREMENT 的复合主键生成隐式 KEY

    不再为复合主键列隐式启用.autoincrement 指令

    参考：[#3216](https://www.sqlalchemy.org/trac/ticket/3216)

### sqlite

+   **[sqlite] [功能]**

    SQLite 方言现在反映了外键约束中的 ON UPDATE 和 ON DELETE 短语。感谢 Michal Petrucha 的拉取请求。

+   **[sqlite] [功能]**

    SQLite 方言现在反映了主键约束的名称。感谢 Diana Clarke 的拉取请求。

    另请参阅

    反映主键约束名称

    参考：[#3629](https://www.sqlalchemy.org/trac/ticket/3629)

+   **[sqlite] [更改]**

    为 SQLite 方言添加了对`Inspector.get_schema_names()`方法的支持，以便与 SQLite 一起使用；感谢 Brian Van Klaveren 的拉取请求。还修复了在具有模式的索引创建以及模式绑定表中外键约束的反射支持。

    另请参阅

    远程模式的改进支持

+   **[sqlite] [错误]**

    当检测到 SQLite 版本 3.7.16 或更高版本时，对 SQLite 中右嵌套连接的解决方法被取消。

    另请参阅

    SQLite 版本 3.7.16 解决了右嵌套连接的问题

    参考文献：[#3634](https://www.sqlalchemy.org/trac/ticket/3634)

+   **[sqlite] [错误]**

    当检测到 SQLite 版本为 3.10.0 或更高版本时，对于某些类型的查询，SQLite 意外地将列名作为 `tablename.columnname` 传送的解决方法现已禁用。

    另请参阅

    针对 SQLite 版本 3.10.0 解除了点分列名的解决方法

    参考文献：[#3633](https://www.sqlalchemy.org/trac/ticket/3633)

### mssql

+   **[mssql] [特性]**

    `UniqueConstraint`、`PrimaryKeyConstraint`、`Index` 上现有的 `mssql_clustered` 标志现在默认为 `None`，并且可以设置为 False，这将为主键特别渲染 NONCLUSTERED 关键字，允许使用不同的索引作为“clustered”。 拉取请求由 Saulius Žemaitaitis 提供。

+   **[mssql] [特性]**

    通过 `create_engine.isolation_level` 和 `Connection.execution_options.isolation_level` 参数，向 SQL Server 方言添加了基本的隔离级别支持。

    另请参阅

    为 SQL Server 添加了事务隔离级别支持

    参考文献：[#3534](https://www.sqlalchemy.org/trac/ticket/3534)

+   **[mssql] [更改]**

    `legacy_schema_aliasing` 标志，作为版本 1.0.5 的一部分引入，以允许禁用 MSSQL 方言尝试为模式合格的表创建别名，现在默认为 False； 旧的行为现在已禁用，除非显式打开。

    另请参阅

    legacy_schema_aliasing 标志现在设置为 False

    参考文献：[#3434](https://www.sqlalchemy.org/trac/ticket/3434)

+   **[mssql] [错误]**

    调整 mxODBC 方言以在适当情况下使用 `BinaryNull` 符号与 `VARBINARY` 数据类型配合使用。 拉取请求由 Sheila Allen 提供。

+   **[mssql] [错误]**

    修复了 SQL Server 方言反映字符串或其他可变长度列类型的问题，通过将 token `"max"` 分配给字符串的长度属性来分配无界长度。 虽然 SQL Server 方言显式支持使用 `"max"` token，但它不是基本字符串类型的正常约定的一部分，相反，长度应该保持为 None。 方言现在在类型反映时将长度分配为 None，以便该类型在其他上下文中正常工作。

    另请参阅

    字符串 / 可变长度类型不再在反射时明确表示“max”

    参考资料：[#3504](https://www.sqlalchemy.org/trac/ticket/3504)

### misc

+   **[feature] [ext]**

    添加了 `MutableSet` 和 `MutableList` 辅助类到 Mutation Tracking 扩展。拉取请求由 Jeong YunWon 提供。

    参考资料：[#3297](https://www.sqlalchemy.org/trac/ticket/3297)

+   **[bug] [ext]**

    现在在混合属性或方法上指定的文档字符串将在类级别上受到尊重，使其能够与 Sphinx autodoc 等工具一起使用。这里的机制必然涉及一些对混合属性进行包装的表达式，这可能会导致它们在内省时显示出不同的外观。

    另请参阅

    混合属性和方法现在也传播文档字符串以及.info

    参考资料：[#3653](https://www.sqlalchemy.org/trac/ticket/3653)

+   **[bug] [sybase]**

    不支持的 Sybase 方言现在在尝试编译包含“偏移”的查询时会引发 `NotImplementedError`；Sybase 没有直接的“偏移”功能。

    参考资料：[#2278](https://www.sqlalchemy.org/trac/ticket/2278)

### orm

+   **[orm] [feature] [ext]**

    添加了一个新的 ORM 扩展 Indexable，它允许构建指向“索引”结构特定元素的 Python 属性，如数组和 JSON 字段。拉取请求由 Jeong YunWon 提供。

    另请参阅

    新的可索引 ORM 扩展

+   **[orm] [feature]**

    添加了新的标志 `Session.bulk_insert_mappings.render_nulls`，允许 ORM 批量插入发生时渲染 NULL 值；这将绕过服务器端默认值，但允许所有语句使用相同的列集形成，从而使它们能够被批处理。拉取请求由 Tobias Sauerwein 提供。

+   **[orm] [feature]**

    添加了新的事件 `AttributeEvents.init_scalar()`，以及一个新的示例套件，说明了其用法。此事件可用于在对象持久化之前为 Python 端属性提供由 Core 生成的默认值。

    另请参阅

    新的 init_scalar() 事件拦截 ORM 级别的默认值

    参考资料：[#1311](https://www.sqlalchemy.org/trac/ticket/1311)

+   **[orm] [feature]**

    在 `AutomapBase.prepare()` 方法中新增了 `AutomapBase.prepare.schema` 参数，指示如果不是默认模式，则应该从哪个模式表反映。拉取请求由 Josh Marlow 提供。

+   **[orm] [功能]**

    在可用的映射器选项中新增了新参数 `mapper.passive_deletes`。这允许对基表进行联合表继承映射的 DELETE 操作，同时允许 ON DELETE CASCADE 处理从子类表删除行。

    请参阅

    联合继承映射的被动删除功能

    参考：[#2349](https://www.sqlalchemy.org/trac/ticket/2349)

+   **[orm] [功能]**

    对核心 SQL 构造调用 str() 方法变得更“友好”了，当构造包含非标准 SQL 元素（如 RETURNING、数组索引操作或方言特定或自定义数据类型）时。在这些情况下，将返回一个字符串，渲染构造的近似值（通常是其 PostgreSQL 风格版本），而不是引发错误。

    请参阅

    核心 SQL 构造的“友好”字符串化，无需方言

    参考：[#3631](https://www.sqlalchemy.org/trac/ticket/3631)

+   **[orm] [功能]**

    对于 `Query` 的 `str()` 调用现在将考虑到 `Session` 绑定的 `Engine`，在生成 SQL 的字符串形式时，以便显示将要发送到数据库的实际 SQL（如果可能的话）。以前，如果存在，只会使用与映射相关联的 `MetaData` 相关的引擎。如果无法在 `Session` 或映射相关联的 `MetaData` 上找到绑定，则使用“默认”方言来渲染 SQL，就像以前一样。

    请参阅

    查询的字符串化将参考 Session 来获取正确的方言

    参考：[#3081](https://www.sqlalchemy.org/trac/ticket/3081)

+   **[orm] [功能]**

    `SessionEvents` 套件现在包括事件，允许明确跟踪所有对象的生命周期状态转换，即 `Session` 本身，例如 pending、transient、persistent、detached。每个事件中的对象状态也已定义。

    另请参阅

    新的会话生命周期事件

    参考：[#2677](https://www.sqlalchemy.org/trac/ticket/2677)

+   **[orm] [特性]**

    添加了一个新的会话生命周期状态 deleted。这个新状态表示一个从 persistent 状态中删除的对象，并且一旦事务提交，将转移到 detached 状态。这解决了长期存在的问题，即已删除的对象存在于持久状态和已分离状态之间的灰色地带。`InstanceState.persistent` 访问器将**不再**报告已删除对象为持久；相反，对于这些对象，`InstanceState.deleted` 访问器将为 True，直到它们变为分离状态。

    另请参阅

    新的会话生命周期事件

    参考：[#2677](https://www.sqlalchemy.org/trac/ticket/2677)

+   **[orm] [特性]**

    添加了对将映射类或映射实例传递到将其解释为 SQL 绑定参数的上下文中的常见错误情况的新检查；为此引发了一个新的异常。

    另请参阅

    为将映射类、实例作为 SQL 字面量传递而添加的特定检查

    参考：[#3321](https://www.sqlalchemy.org/trac/ticket/3321)

+   **[orm] [特性]**

    添加了新的关系加载策略 `raiseload()`（也可通过 `lazy='raise'` 访问）。该策略几乎行为类似于 `noload()`，但不返回 `None`，而是引发一个 InvalidRequestError。此拉取请求由 Adrian Moennich 提供。

    另请参阅

    新的“raise” / “raise_on_sql” 加载策略

    参考：[#3512](https://www.sqlalchemy.org/trac/ticket/3512)

+   **[orm] [变更]**

    `Mapper.order_by` 参数已被弃用。这是一个旧参数，与 SQLAlchemy 的工作方式不再相关，一旦引入了查询对象。通过将其弃用，我们确定我们不支持不起作用的用例，并鼓励应用程序停止使用此参数。

    另请参阅

    Mapper.order_by 已弃用

    参考：[#3394](https://www.sqlalchemy.org/trac/ticket/3394)

+   **[orm] [change]**

    `Session.weak_identity_map`参数已弃用。请参阅 Session Referencing Behavior 中的新配方，以了解维护强身份映射行为的基于事件的方法。

    另请参阅

    新的 Session 生命周期事件

    参考：[#2677](https://www.sqlalchemy.org/trac/ticket/2677)

+   **[orm] [bug]**

    修复了一个问题，即将对象从一个父对象更改为另一个父对象的多对一更改在与外键属性的未刷新修改结合时可能工作不一致。现在，属性移动考虑了外键的数据库提交值，以定位正在移动的对象的“先前”父对象。这允许事件正确触发，包括反向引用事件。以前，这些事件并不总是触发。可能依赖于先前错误行为的应用程序可能会受到影响。

    另请参阅

    修复了涉及用户发起的外键操作的多对一对象移动的问题

    参考：[#3708](https://www.sqlalchemy.org/trac/ticket/3708)

+   **[orm] [bug]**

    修复了一个 bug，即当对象通过`session.merge(obj, load=False)`合并到会话中时，延迟列会在下一个对象级别的取消过期时意外地设置为数据库加载。

    参考：[#3488](https://www.sqlalchemy.org/trac/ticket/3488)

+   **[orm] [bug] [mysql]**

    进一步延续了常见的 MySQL 异常情况，即在[#2696](https://www.sqlalchemy.org/trac/ticket/2696)中首次涵盖的保存点被取消的情况，当 SAVEPOINT 在回滚之前消失时，`Session`的失败模式已经改进，以允许`Session`在该保存点之外继续运行。假定保存点操作失败并被取消。

    另请参阅

    数据库取消 SAVEPOINT 时改进的 Session 状态

    参考：[#3680](https://www.sqlalchemy.org/trac/ticket/3680)

+   **[orm] [bug]**

    修复了一个 bug，即在回滚的新插入实例仍可能在下一个事务中引起持久性冲突，因为实例未被检查为已过期。此修复将解决一大类错误地导致“具有标识 X 的新实例与持久实例 Y 冲突”的错误的情况。

    另请参阅

    修复了错误的“新实例 X 与持久实例 Y 冲突”的刷新错误

    参考：[#3677](https://www.sqlalchemy.org/trac/ticket/3677)

+   **[orm] [bug]**

    对`Query.correlate()`的工作方式进行了改进，以确保当使用代表几个表的直接连接的“多态”实体时，语句将确保连接中的所有表都是相关的。

    另请参阅

    对具有多态实体的 Query.correlate 方法进行改进

    参考：[#3662](https://www.sqlalchemy.org/trac/ticket/3662)

+   **[orm] [bug]**

    修复了一个 bug，该 bug 会导致一个被急加载的多对一属性无法加载，如果连接式急加载来自一个同一实体多次出现的行，有些要求属性被急加载，而其他则不是。这里的逻辑被修改为即使不同的加载路径已经处理了父实体，也要考虑属性。

    另请参阅

    在一行中多次出现相同实体的连接式预加载

    参考：[#3431](https://www.sqlalchemy.org/trac/ticket/3431)

+   **[orm] [bug]**

    对于`Query.distinct()`与`Query.order_by()`结合使用时，对结果 SQL 添加列的逻辑进行了改进，以确保已经存在的列不会被第二次添加，即使它们使用不同的名称标记。尽管有这个改变，SQL 中添加的额外列从未在最终结果中返回，因此这个改变只影响语句的字符串形式以及在核心执行上下文中使用时的行为。此外，当使用 DISTINCT ON 格式时，不再添加列，前提是查询不是由于连接式预加载而包装在子查询中。

    另请参阅

    使用 DISTINCT + ORDER BY 时不再冗余添加列

    参考：[#3641](https://www.sqlalchemy.org/trac/ticket/3641)

+   **[orm] [bug]**

    修复了一个问题，即两个同名关系引用一个基类和一个具体继承子类时，如果使用“backref”设置这些关系会引发错误，而使用 relationship()设置相同配置而使用冲突名称则会成功，这在具体映射的情况下是允许的。

    另请参阅

    应用于具体继承子类时，同名的 backrefs 不会引发错误

    参考：[#3630](https://www.sqlalchemy.org/trac/ticket/3630)

+   **[orm] [bug]**

    `Session.merge()`方法现在在发出 INSERT 之前通过主键跟踪挂起对象，并在遇到重复主键的不同对象时将它们合并在一起，这在最好的情况下基本上是半确定性的。这种行为与持久对象已经发生的情况相匹配。

    参见

    Session.merge 解决挂起冲突与持久性相同

    参考：[#3601](https://www.sqlalchemy.org/trac/ticket/3601)

+   **[orm] [bug]**

    修复了在某些不适当情况下将“单表继承”条件添加到查询末尾的错误，例如在从单继承子类的 exists()查询时。

    参见

    进一步修复单表继承查询

    参考：[#3582](https://www.sqlalchemy.org/trac/ticket/3582)

+   **[orm] [bug]**

    添加了一个新的类型级别修饰符`TypeEngine.evaluates_none()`，指示 ORM 应将一组正面的 None 持久化为值 NULL，而不是在 INSERT 语句中省略列。这个功能既作为[#3514](https://www.sqlalchemy.org/trac/ticket/3514)的实现的一部分，也作为任何类型可用的独立功能。

    参见

    允许显式持久化 NULL 覆盖默认值的新选项

    参考：[#3250](https://www.sqlalchemy.org/trac/ticket/3250)

+   **[orm] [bug]**

    `Session.bulk_save_objects()`内部调用的“簿记”功能以及相关的批量方法已经减少到目前未使用的程度，例如在 INSERT 或 UPDATE 语句之后获取列默认值的检查。

    参考：[#3526](https://www.sqlalchemy.org/trac/ticket/3526)

+   **[orm] [bug] [postgresql]**

    针对 PostgreSQL `JSON` 类型与`None`值的附加修复已经完成。当`JSON.none_as_null` 标志保持默认值`False`时，ORM 现在将正确地将 JSON “‘null’” 字符串插入到列中，无论是当 ORM 对象上的值设置为`None`时，还是当值`None`与`Session.bulk_insert_mappings()`一起使用时，**包括**列上有默认值或服务器默认值的情况。

    参见

    JSON “null” 在 ORM 操作中如预期般插入，当不存在时被省略

    新增选项允许显式持久化 NULL 覆盖默认值

    参考：[#3514](https://www.sqlalchemy.org/trac/ticket/3514)

### 引擎

+   **[引擎] [功能]**

    添加了连接池事件`ConnectionEvents.close()`，`ConnectionEvents.detach()`，`ConnectionEvents.close_detached()`。

+   **[引擎] [功能]**

    现在，用于日志记录、异常和`repr()`目的的所有绑定参数集和结果行的字符串格式化都会截断每个集合中非常大的标量值，包括“N 个字符被截断”说明，类似于对于大型多参数集合本身被截断的显示。

    另请参阅

    日志和异常显示中现在截断大型参数和行值

    参考：[#2837](https://www.sqlalchemy.org/trac/ticket/2837)

+   **[引擎] [功能]**

    为`Table`对象添加了多租户模式翻译。这支持应用程序在许多模式中使用相同的`Table`对象的用例，例如每个用户一个模式。添加了一个新的执行选项`Connection.execution_options.schema_translate_map`。

    另请参阅

    Table 对象的多租户模式翻译

    参考：[#2685](https://www.sqlalchemy.org/trac/ticket/2685)

+   **[引擎] [功能]**

    在引擎中添加了一个新的入口系统，允许在 URL 的查询字符串中声明“插件”。可以编写自定义插件，这些插件将有机会在最初修改和/或使用引擎的 URL 和关键字参数，然后在引擎创建时将获得引擎本身以允许额外的修改或事件注册。插件被编写为`CreateEnginePlugin`的子类；详细信息请参见该类。

    参考：[#3536](https://www.sqlalchemy.org/trac/ticket/3536)

### SQL

+   **[SQL] [功能]**

    通过新的`FromClause.tablesample()`方法和独立函数添加了 TABLESAMPLE 支持。感谢 Ilja Everilä 的拉取请求。

    另请参阅

    支持 TABLESAMPLE

    参考：[#3718](https://www.sqlalchemy.org/trac/ticket/3718)

+   **[SQL] [功能]**

    添加了对窗口函数中范围的支持，使用`over.range_`和`over.rows`参数。

    另请参阅

    窗口函数中的 RANGE 和 ROWS 规范支持

    参考：[#3049](https://www.sqlalchemy.org/trac/ticket/3049)

+   **[sql] [feature]**

    实现了 SQLite 和 PostgreSQL 的 CHECK 约束的反射。这可以通过新的检查器方法`Inspector.get_check_constraints()`以及在反射`Table`对象时以`CheckConstraint`对象的形式存在于约束集合中。感谢 Alex Grönholm 的拉取请求。

+   **[sql] [feature]**

    新的`ColumnOperators.is_distinct_from()`和`ColumnOperators.isnot_distinct_from()`操作符；感谢 Sebastian Bank 的拉取请求。

    另请参阅

    IS DISTINCT FROM 和 IS NOT DISTINCT FROM 的支持

+   **[sql] [feature]**

    在`DDLCompiler.visit_create_table()`中添加了一个名为`DDLCompiler.create_table_suffix()`的钩子，允许自定义方言在“CREATE TABLE”子句之后添加关键字。感谢 Mark Sandan 的拉取请求。

+   **[sql] [feature]**

    负整数索���现在可以被`ResultProxy`返回的行容纳。感谢 Emanuele Gaifas 的拉取请求。

    另请参阅

    Core 结果行中容纳负整数索引

+   **[sql] [feature]**

    添加了`Select.lateral()`和相关构造，以允许使用 SQL 标准的 LATERAL 关键字，目前仅受 PostgreSQL 支持。

    另请参阅

    SQL LATERAL 关键字的支持

    参考：[#2857](https://www.sqlalchemy.org/trac/ticket/2857)

+   **[sql] [feature]**

    增加了对“FULL OUTER JOIN”在 Core 和 ORM 中的渲染支持。感谢 Stefan Urbanek 的拉取请求。

    另请参阅

    Core 和 ORM 对 FULL OUTER JOIN 的支持

    参考：[#1957](https://www.sqlalchemy.org/trac/ticket/1957)

+   **[sql] [feature]**

    CTE 功能已扩展到支持所有 DML，允许 INSERT、UPDATE 和 DELETE 语句指定自己的 WITH 子句，以及当它们包含 RETURNING 子句时，这些语句本身也可以是 CTE 表达式。

    另请参阅

    CTE 支持 INSERT、UPDATE、DELETE

    参考：[#2551](https://www.sqlalchemy.org/trac/ticket/2551)

+   **[sql] [feature]**

    增加了对 PEP-435 风格的枚举类的支持，即 Python 3 的`enum.Enum`类，但也包括兼容的枚举库，到`Enum`数据类型。`Enum`数据类型现在还会对传入的值进行 Python 验证，并添加一个选项来避免创建 CHECK 约束`Enum.create_constraint`。感谢 Alex Grönholm 的拉取请求。

    另请参阅

    支持 Python 的原生枚举类型和兼容形式

    枚举类型现在在 Python 中对值进行验证（migration_11.html#change-3095）

    参考：[#3095](https://www.sqlalchemy.org/trac/ticket/3095)，[#3292](https://www.sqlalchemy.org/trac/ticket/3292)

+   **[sql] [feature]**

    对最近添加的`TextClause.columns()`方法进行了深度改进，以及它与结果行处理的交互，现在允许传递给该方法的列与语句中的结果列进行位置匹配，而不仅仅是按名称匹配。这样做的好处包括，当将文本 SQL 语句链接到 ORM 或 Core 表模型时，不需要进行常见列名的标记或去重系统，这也意味着不需要担心标签名称如何与 ORM 列匹配等等。此外，在某些情况下，`ResultProxy`已经进一步增强，以更精确地将列和字符串键映射到行。

    另请参阅

    ResultSet 列匹配增强；文本 SQL 的位置列设置 - 功能概述

    当按位置传递时，TextClause.columns()将按位置匹配列，而不是按名称匹配 - 向后兼容性说明

    参考：[#3501](https://www.sqlalchemy.org/trac/ticket/3501)

+   **[sql] [feature]**

    添加了核心的新类型`JSON`。这是 PostgreSQL `JSON`类型以及新的`JSON`类型的基础，因此可以使用一个与 PG/MySQL 无关的 JSON 列。该类型具有基本的索引和路径搜索支持。

    另请参阅

    JSON support added to Core

    参考：[#3619](https://www.sqlalchemy.org/trac/ticket/3619)

+   **[sql] [feature]**

    添加了对形式为`<function> WITHIN GROUP (ORDER BY <criteria>)`的“set-aggregate”函数的支持，使用方法`FunctionElement.within_group()`。已添加一系列常见的 set-aggregate 函数，其返回类型源自该集合。这包括函数如`percentile_cont`、`dense_rank`等。

    另请参阅

    New Function features, “WITHIN GROUP”, array_agg and set aggregate functions

    参考：[#1370](https://www.sqlalchemy.org/trac/ticket/1370)

+   **[sql] [feature] [postgresql]**

    添加了对 SQL 标准函数`array_agg`的支持，它会自动返回正确类型的`ARRAY`并支持索引/切片操作，还有`array_agg()`，它返回一个带有额外比较功能的`ARRAY`。由于目前只有 PostgreSQL 支持数组，因此只在 PostgreSQL 上有效。还添加了一个新的构造函数`aggregate_order_by`以支持 PG 的“ORDER BY”扩展。

    另请参阅

    New Function features, “WITHIN GROUP”, array_agg and set aggregate functions

    参考：[#3132](https://www.sqlalchemy.org/trac/ticket/3132)

+   **[sql] [feature]**

    在核心中添加了一个新类型`ARRAY`。这是 PostgreSQL `ARRAY` 类型的基础，并且现在已经成为核心的一部分，以开始支持各种支持 SQL 标准数组的功能，包括一些函数和最终支持其他具有“数组”概念的数据库上的本机数组，例如 DB2 或 Oracle。此外，还添加了新的运算符`any_()`和`all_()`。这些不仅支持 PostgreSQL 上的数组构造，还支持可在 MySQL 上使用的子查询（但遗憾的是在 PostgreSQL 上不支持）。

    另请参阅

    Core 添加了数组支持；新增 ANY 和 ALL 运算符

    参考：[#3516](https://www.sqlalchemy.org/trac/ticket/3516)

+   **[sql] [change] [mysql]**

    `Column`认为自己是“自动增量”列的系统已更改，因此不再为具有复合主键的`Table`隐式启用 autoincrement。为了能够为复合主键成员列启用 autoincrement，同时保持 SQLAlchemy 长期以来为单个整数主键启用隐式 autoincrement 的行为，已向`Column.autoincrement`参数添加了第三状态`"auto"`，这现在是默认值。

    另请参阅

    不再为复合主键列隐式启用.autoincrement 指令

    不再为具有 AUTO_INCREMENT 的复合主键生成隐式 KEY

    参考：[#3216](https://www.sqlalchemy.org/trac/ticket/3216)

+   **[sql] [bug]**

    `FromClause.count()`已被弃用。此函数使用表中的任意列，并不可靠；对于核心使用，应优先使用`func.count()`。

    参考：[#3724](https://www.sqlalchemy.org/trac/ticket/3724)

+   **[sql] [bug]**

    修复了一个断言，如果一个 `Index` 与一个小写的 `TableClause` 关联，那么它会相当不恰当地引发；为了将索引与 `Table` 关联，应忽略此关联。

    参考：[#3616](https://www.sqlalchemy.org/trac/ticket/3616)

+   **[sql] [bug]**

    `type_coerce()` 构造现在是一个完全成熟的 Core 表达式元素，在编译时进行延迟评估。之前，该函数只是一个转换函数，通过返回列导向表达式的 `Label` 或给定 `BindParameter` 对象的副本来处理不同的表达式输入，特别是在 ORM 级别的表达式转换将列转换为绑定参数（例如用于惰性加载）时，该操作不能被逻辑地维护。

    请参见

    type_coerce 函数现在是一个持久的 SQL 元素

    参考：[#3531](https://www.sqlalchemy.org/trac/ticket/3531)

+   **[sql] [bug]**

    `TypeDecorator` 类型扩展器现在将与 `SchemaType` 实现一起工作，通常是 `Enum` 或 `Boolean`，以确保表格事件从实现类型传播到外部类型。这些事件用于确保正确创建（以及可能删除）约束或 PostgreSQL 类型（例如 ENUM），以及与父表一起。

    请参见

    TypeDecorator 现在可以自动与 Enum、Boolean 和“schema” 类型配合使用

    参考：[#2919](https://www.sqlalchemy.org/trac/ticket/2919)

+   **[sql] [bug]**

    `union()`构造和相关构造的行为，如`Query.union()`现在处理嵌入的 SELECT 语句需要括号的情况，因为它们包括 LIMIT、OFFSET 和/或 ORDER BY。这些查询**在 SQLite 上不起作用**，并且在该后端上会像以前一样失败，但现在应该在所有其他后端上正常工作。

    另请参阅

    使用带有 LIMIT/OFFSET/ORDER BY 的 UNION 或类似的 SELECT 现在将嵌入的 SELECT 括在括号中

    参考：[#2528](https://www.sqlalchemy.org/trac/ticket/2528)

### schema

+   **[schema] [enhancement]**

    传递给`Column`对象的默认生成函数现在通过“update_wrapper”运行，或者如果传递了可调用的非函数，则通过等效函数运行，以便内省工具保留包装函数的名称和文档字符串。感谢 hsum 提交的拉取请求。

### postgresql

+   **[postgresql] [feature]**

    添加了对 PostgreSQL 的 INSERT..ON CONFLICT 的支持，使用了一个新的 PostgreSQL 特定的`Insert`对象。感谢 Robin Thomas 提交的拉取请求和大量工作。

    另请参阅

    支持 INSERT..ON CONFLICT (DO UPDATE | DO NOTHING)

    参考：[#3529](https://www.sqlalchemy.org/trac/ticket/3529)

+   **[postgresql] [feature]**

    如果在`Index`上设置了`postgresql_concurrently`标志，并且检测到正在使用的数据库是 PostgreSQL 版本 9.2 或更高，则 DROP INDEX 的 DDL 将发出“CONCURRENTLY”。对于 CREATE INDEX，还添加了数据库版本检测，如果 PG 版本低于 8.2，则将省略该子句。感谢 Iuri de Silvio 提交的拉取请求。

+   **[postgresql] [feature]**

    添加了新参数`PGInspector.get_view_names.include`，允许指定应返回哪种类型的视图。目前包括“plain”和“materialized”视图。感谢 Sebastian Bank 提交的拉取请求。

    参考：[#3588](https://www.sqlalchemy.org/trac/ticket/3588)

+   **[postgresql] [feature]**

    将`postgresql_tablespace`作为参数添加到`Index`中，以允许在 PostgreSQL 中为索引指定 TABLESPACE。与`Table`上的同名参数相辅相成。感谢 Benjamin Bertrand 提交的拉取请求。

    参考：[#3720](https://www.sqlalchemy.org/trac/ticket/3720)

+   **[postgresql] [feature]**

    新增了新参数`GenerativeSelect.with_for_update.key_share`，该参数将在 PostgreSQL 后端上呈现`FOR NO KEY UPDATE`版本的`FOR UPDATE`和`FOR KEY SHARE`，而不是`FOR SHARE`。感谢 Sergey Skopin 的拉取请求。

+   **[postgresql] [feature] [oracle]**

    新增了新参数`GenerativeSelect.with_for_update.skip_locked`，该参数将在 PostgreSQL 和 Oracle 后端上为`FOR UPDATE`或`FOR SHARE`锁呈现`SKIP LOCKED`短语。感谢 Jack Zhou 的拉取请求。

+   **[postgresql] [feature]**

    为 PyGreSQL PostgreSQL 方言新增了一个新的方言。感谢 Christoph Zwerschke 和 Kaolin Imago Fire 的努力。

+   **[postgresql] [feature]**

    新增了一个新的常量`JSON.NULL`，表示无论其他设置如何，都应使用 JSON NULL 值。

    参见

    新增 JSON.NULL 常量

    参考：[#3514](https://www.sqlalchemy.org/trac/ticket/3514)

+   **[postgresql] [change]**

    已移除`sqlalchemy.dialects.postgres`模块，该模块长期已过时；这已经发出了多年的警告，项目应该调用`sqlalchemy.dialects.postgresql`。形式为`postgres://`的引擎 URL 仍将继续使用。

+   **[postgresql] [bug]**

    新增了对在 PostgreSQL 版本的`Inspector.get_view_definition()`方法中反映材料化视图来源的支持。

    参考：[#3587](https://www.sqlalchemy.org/trac/ticket/3587)

+   **[postgresql] [bug]**

    当一个`ARRAY`对象引用一个`Enum`或`ENUM`子类型时，现在将会在该类型在“CREATE TABLE”或“DROP TABLE”中使用时发出预期的“CREATE TYPE”和“DROP TYPE” DDL。

    参见

    ARRAY with ENUM will now emit CREATE TYPE for the ENUM

    参考：[#2729](https://www.sqlalchemy.org/trac/ticket/2729)

+   **[postgresql] [bug]**

    特殊数据类型（如`ARRAY`、`JSON`和`HSTORE`）上的“可散列”标志现在设置为 False，这允许在包含行内实体的 ORM 查询中获取这些类型。

    另请参阅

    关于“不可散列”类型的更改，影响 ORM 行的去重

    数组和 JSON 类型现在正确指定为“不可散列”

    参考：[#3499](https://www.sqlalchemy.org/trac/ticket/3499)

+   **[postgresql] [bug]**

    PostgreSQL `ARRAY` 类型现在支持多维索引访问，例如`somecol[5][6]`这样的表达式，无需显式转换或类型强制转换，只要`ARRAY.dimensions`参数设置为所需的维数即可。

    另请参阅

    从数组、JSON、HSTORE 的索引访问中建立正确的 SQL 类型

    参考：[#3487](https://www.sqlalchemy.org/trac/ticket/3487)

+   **[postgresql] [bug]**

    当使用索引访问时，`JSON`和`JSONB`的返回类型已经修复，使其像 PostgreSQL 本身一样工作，并返回一个表达式，该表达式本身是类型`JSON`或`JSONB`。以前，访问器会返回`NullType`，这将禁止使用后续类似 JSON 的运算符。

    另请参阅

    从数组、JSON、HSTORE 的索引访问中建立正确的 SQL 类型

    参考：[#3503](https://www.sqlalchemy.org/trac/ticket/3503)

+   **[postgresql] [bug]**

    `JSON`、`JSONB` 和 `HSTORE` 数据类型现在允许完全控制从索引文本访问操作的返回类型，对于 JSON 类型是`column[someindex].astext`，对于 HSTORE 类型是`column[someindex]`，通过 `JSON.astext_type` 和 `HSTORE.text_type` 参数。

    另请参阅

    正确的 SQL 类型是通过对数组、JSON、HSTORE 进行索引访问来建立的

    参考：[#3503](https://www.sqlalchemy.org/trac/ticket/3503)

+   **[postgresql] [bug]**

    `Comparator.astext` 修改器不再隐式调用 `ColumnElement.cast()`，因为 PG 的 JSON/JSONB 类型允许彼此之间的交叉转换。在 JSON 索引访问上使用 `ColumnElement.cast()` 的代码，例如 `col[someindex].cast(Integer)`，现在需要显式调用 `Comparator.astext`。

    另请参阅

    JSON cast() 操作现在需要显式调用 .astext

    参考：[#3503](https://www.sqlalchemy.org/trac/ticket/3503)

### mysql

+   **[mysql] [feature]**

    增加了对 MySQL 驱���程序的“autocommit”支持，通过 AUTOCOMMIT 隔离级别设置。感谢 Roman Podoliaka 的拉取请求。

    另请参阅

    增加了对 AUTOCOMMIT “隔离级别”的支持

    参考：[#3332](https://www.sqlalchemy.org/trac/ticket/3332)

+   **[mysql] [feature]**

    为 MySQL 5.7 添加了`JSON`。JSON 类型在 MySQL 中提供了 JSON 值的持久性，以及基本的“getitem”和“getpath”操作符支持，利用`JSON_EXTRACT`函数来引用 JSON 结构中的单个路径。

    另请参阅

    MySQL JSON 支持

    参考：[#3547](https://www.sqlalchemy.org/trac/ticket/3547)

+   **[mysql] [change]**

    当使用 InnoDB 在列不是第一列上具有 AUTO_INCREMENT 的复合主键的表生成 CREATE TABLE DDL 时，MySQL 方言不再生成额外的“KEY”指令；为了克服 InnoDB 在这里的限制，现在主键约束将在列列表中的 AUTO_INCREMENT 列放在第一位。

    另请参阅

    不再为具有 AUTO_INCREMENT 的复合主键生成隐式 KEY

    不再为复合主键列隐式启用.autoincrement 指令

    参考：[#3216](https://www.sqlalchemy.org/trac/ticket/3216)

### sqlite

+   **[sqlite] [功能]**

    SQLite 方言现在反映了外键约束中的 ON UPDATE 和 ON DELETE 短语。感谢 Michal Petrucha 的拉取请求。

+   **[sqlite] [功能]**

    SQLite 方言现在反映了主键约束的名称。感谢 Diana Clarke 的拉取请求。

    另请参阅

    主键约束名称的反射

    参考：[#3629](https://www.sqlalchemy.org/trac/ticket/3629)

+   **[sqlite] [更改]**

    为 SQLite 方言添加了对`Inspector.get_schema_names()`方法的支持；感谢 Brian Van Klaveren 的拉取请求。还修复了在具有模式的索引创建以及模式绑定表中外键约束的反射支持。

    另请参阅

    远程模式的改进支持

+   **[sqlite] [错误]**

    当检测到 SQLite 版本为 3.7.16 或更高版本时，针对 SQLite 右嵌套连接的解决方法，其中它们被重写为子查询以解决 SQLite 不支持此语法的问题，将被取消。

    另请参阅

    SQLite 版本 3.7.16 取消右嵌套连接解决方法

    参考：[#3634](https://www.sqlalchemy.org/trac/ticket/3634)

+   **[sqlite] [错误]**

    当检测到 SQLite 版本为 3.10.0 或更高版本时，针对 SQLite 某些查询意外将列名传递为`tablename.columnname`的解决方法现已禁用。

    另请参阅

    取消 SQLite 版本 3.10.0 的点列名解决方法

    参考：[#3633](https://www.sqlalchemy.org/trac/ticket/3633)

### mssql

+   **[mssql] [功能]**

    可用于 `UniqueConstraint`、`PrimaryKeyConstraint`、`Index` 的 `mssql_clustered` 标志现在默认为 `None`，并且可以设置为 False，这将特别为主键渲染 NONCLUSTERED 关键字，允许使用不同的索引作为“聚集”。感谢 Saulius Žemaitaitis 的拉取请求。

+   **[mssql] [feature]**

    通过`create_engine.isolation_level`和`Connection.execution_options.isolation_level`参数，为 SQL Server 方言添加了基本的隔离级别支持。

    另请参阅

    为 SQL Server 添加了事务隔离级别支持

    参考：[#3534](https://www.sqlalchemy.org/trac/ticket/3534)

+   **[mssql] [change]**

    `legacy_schema_aliasing` 标志在 1.0.5 版本中引入，作为 [#3424](https://www.sqlalchemy.org/trac/ticket/3424) 的一部分，允许禁用 MSSQL 方言尝试为模式合格的表创建别名。现在默认为 False；除非显式启用，否则将禁用旧行为。

    另请参阅

    legacy_schema_aliasing 标志现在设置为 False

    参考：[#3434](https://www.sqlalchemy.org/trac/ticket/3434)

+   **[mssql] [bug]**

    对 mxODBC 方言进行了调整，以便在适当情况下与 `VARBINARY` 数据类型一起使用 `BinaryNull` 符号。感谢 Sheila Allen 的拉取请求。

+   **[mssql] [bug]**

    修复了 SQL Server 方言将字符串或其他可变长度列类型的无限长度反映为将标记`"max"`分配给字符串的长度属性的问题。虽然 SQL Server 方言明确支持使用`"max"`标记，但它并不是基本字符串类型的正常约定的一部分，相反，长度应该只留为空。方言现在在反射类型时将长度分配为 None，以便类型在其他情境中表现正常。

    另请参阅

    在反射时，字符串/可变长度类型不再明确表示“max”。

    参考：[#3504](https://www.sqlalchemy.org/trac/ticket/3504)

### 杂项

+   **[feature] [ext]**

    将 `MutableSet` 和 `MutableList` 辅助类添加到 Mutation Tracking 扩展中。感谢 Jeong YunWon 的拉取请求。

    参考：[#3297](https://www.sqlalchemy.org/trac/ticket/3297)

+   **[bug] [ext]**

    对混合属性或方法指定的文档字符串现在在类级别上得到了尊重，使其能够与 Sphinx autodoc 等工具一起使用。这里的机制必然涉及对混合属性进行一些表达式包装，这可能会导致它们在使用内省时显示出不同的外观。

    另请参阅：

    混合属性和方法现在传播文档字符串以及 .info

    参考：[#3653](https://www.sqlalchemy.org/trac/ticket/3653)

+   **[bug] [sybase]**

    不支持的 Sybase 方言现在在尝试编译包含“offset”的查询时引发 `NotImplementedError`；Sybase 没有直接的“offset”功能。

    参考：[#2278](https://www.sqlalchemy.org/trac/ticket/2278)
