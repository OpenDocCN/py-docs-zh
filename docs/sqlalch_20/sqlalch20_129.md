# 1.2 更新日志

> 原文：[`docs.sqlalchemy.org/en/20/changelog/changelog_12.html`](https://docs.sqlalchemy.org/en/20/changelog/changelog_12.html)

## 1.2.19

发布日期：2019 年 4 月 15 日

### orm

+   **[orm] [bug]**

    修复了 1.2 版本中由于为关系懒加载器引入烘焙查询而导致的回归，其中在生成“懒惰子句”时创建了竞争条件，该条件发生在一个被记忆的属性内。如果两个线程同时初始化被记忆的属性，则烘焙查询可能会生成带有绑定参数键的查询，然后在下一次运行时用新键替换，导致懒加载查询将相关条件指定为`None`。修复确保在生成新子句和参数对象之前固定参数名称，以便每次名称都相同。

    参考：[#4507](https://www.sqlalchemy.org/trac/ticket/4507)

### 示例

+   **[示例] [bug]**

    修复了大结果集示例中的 bug，由于代码重排导致“id”变量重新命名，导致测试失败。感谢 Matt Schuchhardt 提供的拉取请求。

    参考：[#4528](https://www.sqlalchemy.org/trac/ticket/4528)

### engine

+   **[engine] [bug]**

    使用`__eq__()`比较两个`URL`对象时未考虑端口号，只有端口号不同的两个对象被视为相等。现在在`URL`的`__eq__()`方法中添加了端口比较，端口号不同的对象现在不相等。此外，`URL`未实现`__ne__()`，导致在 Python2 中使用`!=`时出现意外结果，因为在 Python2 中比较运算符之间没有暗示的关系。

    参考：[#4406](https://www.sqlalchemy.org/trac/ticket/4406)

### mssql

+   **[mssql] [bug]**

    在将隔离级别更改为 SNAPSHOT 后会发出一个 commit()，因为 pyodbc 和 pymssql 都会打开一个隐式事务，这会阻止当前事务中发出后续的 SQL。

    参考：[#4536](https://www.sqlalchemy.org/trac/ticket/4536)

### oracle

+   **[oracle] [bug]**

    增加了对 Oracle 方言反射`NCHAR`数据类型的支持，并将`NCHAR`添加到 Oracle 方言导出的类型列表中。

    参考：[#4506](https://www.sqlalchemy.org/trac/ticket/4506)

## 1.2.18

发布日期：2019 年 2 月 15 日

### orm

+   **[orm] [bug]**

    修复了 1.2 版本中的一个回归问题，即通配符/load_only 加载器选项在加载路径中使用 of_type()限制到特定子类时无法正常工作。修复目前仅适用于简单子类的 of_type()，而不适用于将在单独问题中解决的 with_polymorphic 实体；这种后一种情况以前可能不起作用。

    参考：[#4468](https://www.sqlalchemy.org/trac/ticket/4468)

+   **[orm] [bug]**

    修复了一个相当简单但关键的问题，即`SessionEvents.pending_to_persistent()`事件不仅在对象从待定转为持久时被调用，而且在它们已经是持久的并且正在更新时也被调用，从而导致事件在每次更新时为所有对象调用。

    参考：[#4489](https://www.sqlalchemy.org/trac/ticket/4489)

### sql

+   **[sql] [bug]**

    修复了`JSON`类型具有只读`JSON.should_evaluate_none`属性的问题，这会导致在与此类型一起使用`TypeEngine.evaluates_none()`方法时出现故障。感谢 Sanjana S 提交的拉取请求。

    参考：[#4485](https://www.sqlalchemy.org/trac/ticket/4485)

### mysql

+   **[mysql] [bug]**

    修复了由[#4344](https://www.sqlalchemy.org/trac/ticket/4344)引起的第二个回归问题（第一个是[#4361](https://www.sqlalchemy.org/trac/ticket/4361)），这是针对 MySQL 问题 88718 的解决方案，其中使用的小写函数对于具有 Python 2 的 OSX/Windows 大小写约定不正确，这将引发`TypeError`。已对此逻辑添加了完整覆盖，以便以模拟样式在所有 Python 版本的所有三种大小写约定上执行每个代码路径。与此同时，MySQL 8.0 已经修复了问题 88718，因此这个解决方案仅适用于特定范围的 MySQL 8.0 版本。

    参考：[#4492](https://www.sqlalchemy.org/trac/ticket/4492)

### sqlite

+   **[sqlite] [bug]**

    修复了在 SQLite DDL 中的一个 bug，其中将表达式用作服务器端默认值需要将其包含在括号中才能被 sqlite 解析器接受。感谢 Bartlomiej Biernacki 提交的拉取请求。

    参考：[#4474](https://www.sqlalchemy.org/trac/ticket/4474)

### mssql

+   **[mssql] [bug]**

    修复了一个错误，即 SQL Server 中允许在 IDENTITY 列上使用明确值进行插入的“IDENTITY_INSERT”逻辑未检测到使用字典的情况，该字典包含`Insert.values()`作为键和 SQL 表达式作为值的`Column`。

    参考：[#4499](https://www.sqlalchemy.org/trac/ticket/4499)

## 1.2.17

发布日期：2019 年 1 月 25 日

### orm

+   **[orm] [feature]**

    添加了新的事件钩子`QueryEvents.before_compile_update()`和`QueryEvents.before_compile_delete()`，它们在`Query.update()`和`Query.delete()`方法的情况下补充了`QueryEvents.before_compile()`。

    参考：[#4461](https://www.sqlalchemy.org/trac/ticket/4461)

+   **[orm] [bug]**

    修复了单表继承与使用“with polymorphic”加载的连接继承层次结构一起使用时，“单表条件”可能会被混淆为同一查询中使用的同一层次结构的其他实体的问题。将“单表条件”的适应性更具体地改为目标实体，以避免它意外地适应为查询中的其他表。

    参考：[#4454](https://www.sqlalchemy.org/trac/ticket/4454)

### postgresql

+   **[postgresql] [bug]**

    修订了在反射 CHECK 约束时使用的查询，以利用`pg_get_constraintdef`函数，因为`consrc`列在 PG 12 中已被弃用。感谢约翰·A·斯蒂文森的提示。

    参考：[#4463](https://www.sqlalchemy.org/trac/ticket/4463)

### oracle

+   **[oracle] [bug]**

    由于 1.2 版本中 cx_Oracle 方言的重构导致整数精度逻辑的回归。现在我们不再将 cx_Oracle.NATIVE_INT 类型应用于发送整数值的结果列（检测到为正精度且比例为 0），这会导致超出 32 位边界的值发生整数溢出问题。相反，输出变量保持未分类，以便 cx_Oracle 可以选择最佳选项。

    参考：[#4457](https://www.sqlalchemy.org/trac/ticket/4457)

## 1.2.16

发布日期：2019 年 1 月 11 日

### 引擎

+   **[engine] [bug]**

    修复了在版本 1.2 中引入的回归，其中对`SQLAlchemyError`基本异常类的重构引入了一个不当的强制转换，将纯字符串消息转换为 Unicode 在 python 2k 下，这不被 Python 解释器处理，因为平台的编码（通常是 ascii）之外的字符。`SQLAlchemyError`类现在在 Py2K 下通过`__str__()`传递一个字节串，这是 Py2K 下异常对象的一般行为，对于`__unicode__()`进行安全的 utf-8 强制转换。对于 Py3K，消息通常已经是 unicode，但如果不是，`__str__()`方法会再次进行安全的 utf-8 强制转换。

    参考：[#4429](https://www.sqlalchemy.org/trac/ticket/4429)

### sql

+   **[sql] [bug] [mysql] [oracle]**

    修复了为`DropTableComment`发出的 DDL 对于即将用于 Alembic 的新版本的 MySQL 和 Oracle 数据库是不正确的问题。

    参考：[#4436](https://www.sqlalchemy.org/trac/ticket/4436)

### postgresql

+   **[postgresql] [bug]**

    修复了在远程模式中存在的`ENUM`或自定义域在列反射中无法识别的问题，如果枚举/域的名称或模式的名称需要引号。现在，新的解析方案完全解析带引号或不带引号的标记，包括支持 SQL 转义引号。

    参考：[#4416](https://www.sqlalchemy.org/trac/ticket/4416)

+   **[postgresql] [bug]**

    修复了同一`MetaData`对象引用的多个`ENUM`对象在具有相同名称但不同模式名称的情况下无法创建的问题。PostgreSQL 方言在 DDL 创建序列期间用于跟踪是否在数据库中创建了特定`ENUM`的内部记忆现在考虑了模式名称。

### sqlite

+   **[sqlite] [bug]**

    基于 SQL 表达式的索引的反射现在会跳过并发出警告，与 Postgresql 方言的方式相同，我们目前不支持反映其中包含 SQL 表达式的索引。以前，会生成具有 None 列的索引，这会破坏像 Alembic 这样的工具。

    参考：[#4431](https://www.sqlalchemy.org/trac/ticket/4431)

### 杂项

+   **[no_tags]**

    修复了“扩展 IN”功能中的问题，即在查询中多次使用相同的绑定参数名称会导致在重写查询中的参数时出现 KeyError 的问题。

    参考：[#4394](https://www.sqlalchemy.org/trac/ticket/4394)

## 1.2.15

发布日期：2018 年 12 月 11 日

### orm

+   **[orm] [bug]**

    修复了当在声明映射中使用`ForeignKey(SomeClass.id)`模式时，ORM 注释可能不正确的 bug。这种模式会将不需要的注释泄漏到加入条件中，这可能会破坏`Query`中进行的别名操作，这些操作不应影响该加入条件中的元素。如果存在这些注释，现在会立即将其删除。

    参考：[#4367](https://www.sqlalchemy.org/trac/ticket/4367)

+   **[orm] [bug]**

    继续解决与最近[#4349](https://www.sqlalchemy.org/trac/ticket/4349)类似主题的问题，修复了`Comparator.any()`和`Comparator.has()`中的问题，其中“secondary”可选择需要明确作为 FROM 子查询的一部分，以适应“secondary”是`Join`对象的情况。

    参考：[#4366](https://www.sqlalchemy.org/trac/ticket/4366)

+   **[orm] [bug]**

    修复了由[#4349](https://www.sqlalchemy.org/trac/ticket/4349)引起的回归问题，即将“secondary”表添加到动态加载器的 FROM 子句中会影响`Query`对另一个实体进行后续连接的能力。修复方法是将主实体添加为 FROM 列表的第一个元素，因为`Query.join()`希望从那里跳转。版本 1.3 还将对此问题提供更全面的解决方案（[#4365](https://www.sqlalchemy.org/trac/ticket/4365)）。

    参考：[#4363](https://www.sqlalchemy.org/trac/ticket/4363)

+   **[orm] [bug]**

    修复了一个 bug，即在使用`RelationshipProperty.of_type()`链接映射器选项时，与仅通过字符串引用属性名称的链接选项一起使用时，会无法定位属性的问题。

    参考：[#4400](https://www.sqlalchemy.org/trac/ticket/4400)

### orm declarative

+   **[orm] [declarative] [bug]**

    在将`column()`对象应用于声明类的情况下，会发出警告，因为这似乎是打算将其作为`Column`对象。

    参考：[#4374](https://www.sqlalchemy.org/trac/ticket/4374)

### 杂项

+   **[no_tags]**

    添加了对`write_timeout`标志的支持，该标志被 mysqlclient 和 pymysql 接受并传递到 URL 字符串中。

    参考：[#4381](https://www.sqlalchemy.org/trac/ticket/4381)

+   **[no_tags]**

    修复了一个问题，即无法识别表示为数组的 PostgreSQL 域的反射。感谢 Jakub Synowiec 提供的拉取请求。

    参考：[#4377](https://www.sqlalchemy.org/trac/ticket/4377), [#4380](https://www.sqlalchemy.org/trac/ticket/4380)

## 1.2.14

发布日期：2018 年 11 月 10 日

### orm

+   **[orm] [bug]**

    修复了`Session.bulk_update_mappings()`中的 bug，其中替代映射属性名称会导致 UPDATE 语句的主键列包含在 SET 子句中，以及 WHERE 子句中；虽然通常无害，但对于 SQL Server，这可能会由于 IDENTITY 列而引发错误。这是在[#3849](https://www.sqlalchemy.org/trac/ticket/3849)中修复的相同 bug 的延续，测试不足以捕捉到这个额外的缺陷。

    参考：[#4357](https://www.sqlalchemy.org/trac/ticket/4357)

+   **[orm] [bug]**

    修复了一个轻微的性能问题，可能会在某些情况下增加不必要的开销，涉及在查询中同时使用 ORM 列和包含这些列的实体时。问题涉及在不同方式引用列时的哈希/相等开销。

    参考：[#4347](https://www.sqlalchemy.org/trac/ticket/4347)

### mysql

+   **[mysql] [bug]**

    修复了 1.2.13 中发布的[#4344](https://www.sqlalchemy.org/trac/ticket/4344)引起的回归问题，其中解决了 MySQL 8.0 在反射外键引用列名称时的大小写敏感性问题，使用`information_schema.columns`视图绕过。在 OSX / `lower_case_table_names=2`上，这种解决方法会失败，因为`information_schema.columns`的大小写与`SHOW CREATE TABLE`不匹配，因此在不区分大小写的 SQL 模式下现在使用不区分大小写的匹配。

    参考：[#4361](https://www.sqlalchemy.org/trac/ticket/4361)

## 1.2.13

发布日期：2018 年 10 月 31 日

### orm

+   **[orm] [bug]**

    修复了“动态”加载器需要在查询的 FROM 子句中显式设置“secondary”表的 bug，以适应次要表是一个联接对象，否则仅从其列中提取无法将其拉入查询的情况。

    参考：[#4349](https://www.sqlalchemy.org/trac/ticket/4349)

### orm 声明式

+   **[orm] [declarative] [bug]**

    修复了版本 1.2.12 中由[#4326](https://www.sqlalchemy.org/trac/ticket/4326)引起的回归，其中在使用`declared_attr`与`synonym()`混合使用时，会导致无法正确将同义词映射到继承的子类。

    参考：[#4350](https://www.sqlalchemy.org/trac/ticket/4350)

+   **[orm] [declarative] [bug]**

    现在，讨论的列冲突解决技术使用`use_existing_column`解决列冲突对于同时是主键列的`Column`现在可用。以前，在允许列复制通过之前，会先检查单一继承子类上声明的主键列。

    参考：[#4352](https://www.sqlalchemy.org/trac/ticket/4352)

### sql

+   **[sql] [feature]**

    重构`SQLCompiler`以公开类似于`SQLCompiler.order_by_clause()`和`SQLCompiler.limit_clause()`方法的`SQLCompiler.group_by_clause()`方法，可以被方言重写以自定义 GROUP BY 的呈现方式。感谢 Samuel Chou 的拉取请求。

+   **[sql] [bug]**

    修复了`Enum`数据类型上的`Enum.create_constraint`标志不会传播到类型的副本的错误，这会影响到声明性混合和抽象基类等用例。

    参考：[#4341](https://www.sqlalchemy.org/trac/ticket/4341)

### postgresql

+   **[postgresql] [bug]**

    增加了对`aggregate_order_by`函数接收多个 ORDER BY 元素的支持，之前只接受单个元素。

    参考：[#4337](https://www.sqlalchemy.org/trac/ticket/4337)

### mysql

+   **[mysql] [bug]**

    将`function`一词添加到 MySQL 的保留字列表中，现在在 MySQL 8.0 中是一个关键字。

    参考：[#4348](https://www.sqlalchemy.org/trac/ticket/4348)

+   **[mysql] [bug]**

    为 MySQL 8.0 系列中引入的一个 bug #88718 添加了一个解决方法，其中外键约束的反射未报告所引用列的正确大小写敏感性，导致在使用反射约束时出现错误，例如在使用 automap 扩展时。解决方法通过向 information_schema 表发出额外查询以检索正确的大小写敏感名称。

    参考：[#4344](https://www.sqlalchemy.org/trac/ticket/4344)

### 杂项

+   **[杂项] [错误]**

    修复了实用语言助手内部的一部分错误，该错误将错误类型的参数传递给 Python `__import__`内置函数作为要导入的模块列表。该问题在核心库中没有产生任何症状，但可能会导致重新定义`__import__`内置函数或以其他方式对其进行调整的外部应用程序出现问题。感谢 Joe Urciuoli 的拉取请求。

+   **[杂项] [错误] [py3k]**

    修复了由于 Python 3.7 中 Python `collections`和`collections.abc`包组织变化而产生的额外警告。之前版本 1.2.11 中已修复了`collections`的警告。感谢 xtreak 的拉取请求。

    参考：[#4339](https://www.sqlalchemy.org/trac/ticket/4339)

+   **[错误] [扩展]**

    在关联代理扩展中的基于列表的关联集合中添加了缺失的`.index()`方法。

## 1.2.12

发布日期：2018 年 9 月 19 日

### ORM

+   **[ORM] [错误]**

    在弱引用清理中添加了一个检查，用于检查`InstanceState`对象中是否存在`dict`内置对象，以减少在解释器关闭时发生这些清理时生成的错误消息。感谢 Romuald Brunet 的拉取请求。

+   **[ORM] [错误]**

    修复了在与`Query.join()`以及`Query.select_entity_from()`一起使用`Lateral`构造时，不会将子句适应到连接的右侧的 bug。 “lateral”引入了连接的右侧可以相关的用例。以前，未考虑适应此子句。请注意，在 1.2 版本中，由`Query.subquery()`引入的可选择项仍未适应，原因是[#4304](https://www.sqlalchemy.org/trac/ticket/4304)；可选择项需要由`select()`函数生成，以成为“lateral”连接的右侧。

    参考：[#4334](https://www.sqlalchemy.org/trac/ticket/4334)

+   **[ORM] [错误]**

    修复了 1.2 版本中由 [#3472](https://www.sqlalchemy.org/trac/ticket/3472) 引起的回归问题，其中在后续更新操作的上下文中处理“updated_at”样式列也会发生在更新后要删除的行上，这意味着具有 Python 端值生成器的列将显示在更新之前发出的现已删除值（这不是以前的行为），以及 SQL 发出的值生成器将使属性过期，这意味着由于行已被删除并且对象已从会话中分离，无法访问以前的值。作为 [#3472](https://www.sqlalchemy.org/trac/ticket/3472) 的一部分添加的“postfetch”逻辑现在完全跳过将最终被删除的对象。

    参考：[#4327](https://www.sqlalchemy.org/trac/ticket/4327)

### orm 声明式

+   **[orm] [declarative] [bug]**

    修复了一个 bug，其中对已映射类的子类上使用 `@declared_attr` 可调用时，属性的声明性扫描会收到由混合属性在类级别传递的表达式代理，而不是混合属性本身。这将导致在 `Mapper.all_orm_descriptors` 中查看时，未报告自身为混合的属性。

    参考：[#4326](https://www.sqlalchemy.org/trac/ticket/4326)

### postgresql

+   **[postgresql] [bug]**

    修复了 PostgreSQL 方言中的一个 bug，其中编译器关键字参数（如 `literal_binds=True`）未被传播到 DISTINCT ON 表达式。

    参考：[#4325](https://www.sqlalchemy.org/trac/ticket/4325)

+   **[postgresql] [bug]**

    修复了 `array_agg()` 函数，这是通常 `array_agg()` 函数的略微改变版本，也接受一个传入的“type”参数，而不需要强制在其周围使用 ARRAY，本质上与 1.1 版本中修复的通用函数相同 [#4107](https://www.sqlalchemy.org/trac/ticket/4107)。

    参考：[#4324](https://www.sqlalchemy.org/trac/ticket/4324)

+   **[postgresql] [bug]**

    修复了 PostgreSQL 枚举反射中的 bug，其中包含引号的区分大小写名称将由查询报告，这些引号在表反射期间不会与目标列匹配，因为需要去除引号。

    参考：[#4323](https://www.sqlalchemy.org/trac/ticket/4323)

### oracle

+   **[oracle] [bug]**

    为 cx_Oracle 7.0 修复了一个问题，其中 Oracle param.getvalue() 的行为现在返回一个列表，而不是单个标量值，从而在 Core 和 ORM 中破坏了自增逻辑。 dml_ret_array_val 兼容性标志用于 cx_Oracle 6.3 和 6.4，以建立与 7.0 及更高版本的兼容性行为，对于 cx_Oracle 6.2.1 及之前的版本号检查，将退回到旧逻辑。

    参考：[#4335](https://www.sqlalchemy.org/trac/ticket/4335)

### 其他

+   **[bug] [ext]**

    修复了`BakedQuery` 没有将 `Session` 使用的具体查询类作为缓存键的一部分，导致在使用自定义查询类时出现不兼容性，特别是 `ShardedQuery` 其具有一些不同的参数签名。

    参考：[#4328](https://www.sqlalchemy.org/trac/ticket/4328)

## 1.2.11

发布日期：2018 年 8 月 20 日

### orm 声明式

+   **[orm] [declarative] [bug]**

    修复了先前未经测试的用例中的问题，允许声明式映射的类继承自声明式基类之外的经典映射类，包括它适应于未映射的中间类的情况。 未映射的中间类可以指定 `__abstract__`，现在将正确解释，或者中间类可以保持未标记状态，并且基类将在层次结构中被正确检测到。 为了预期可能将经典映射混合到现有声明式层次结构中的现有场景，如果检测到给定类的多个映射基类，现在将引发错误。

    参考：[#4321](https://www.sqlalchemy.org/trac/ticket/4321)

### sql

+   **[sql] [bug]**

    修复了一个与 [#3639](https://www.sqlalchemy.org/trac/ticket/3639) 密切相关的问题，即在非原生布尔后端的布尔上下文中呈现的表达式将与 1/0 进行比较，即使它已经是一个隐式布尔表达式，当使用 `ColumnElement.self_group()` 时。虽然这不影响用户友好的后端（MySQL，SQLite），但 Oracle（可能还包括 SQL Server）没有处理它。现在，任何数据库上的表达式是否隐式布尔都将被提前确定为附加检查，以在语句的编译中不生成整数比较。

    参考：[#4320](https://www.sqlalchemy.org/trac/ticket/4320)

+   **[sql] [bug]**

    添加了缺失的窗口函数参数`WithinGroup.over.range_`和`WithinGroup.over.rows`参数到`WithinGroup.over()`和`FunctionFilter.over()`方法，以对应于版本 1.1 中作为 SQL 函数“over”方法的一部分添加的 range/rows 功能。

    参考：[#4322](https://www.sqlalchemy.org/trac/ticket/4322)

+   **[sql] [bug]**

    修复了 UPDATE 和 DELETE 语句的多表支持未将额外的 FROM 元素视为相关目标的 bug，当相关的 SELECT 也与语句组合时。此更改现在包括了 WHERE 子句中的 SELECT 语句将尝试自动关联回这些额外的表到父 UPDATE/DELETE 中，或者如果使用了`Select.correlate()`，则无条件关联。请注意，如果 SELECT 语句由于父 UPDATE/DELETE 在其额外的表集中指定相同的表而没有 FROM 子句，则自动关联会引发错误；请显式指定`Select.correlate()`以解决。

    参考：[#4313](https://www.sqlalchemy.org/trac/ticket/4313)

### oracle

+   **[oracle] [bug]**

    对于 cx_Oracle，整数数据类型现在将绑定到“int”，根据 cx_Oracle 开发人员的建议。以前，在 cx_Oracle 6.x 系列中使用 cx_Oracle.NUMBER 会导致精度丢失。

    参考：[#4309](https://www.sqlalchemy.org/trac/ticket/4309)

### 杂项

+   **[bug] [py3k]**

    开始在 Python 3.3 及更高版本中从“collections.abc”导入“collections”，以实现 Python 3.8 的兼容性。感谢 Nathaniel Knight 的拉取请求。

+   **[no_tags]**

    修复了在表反射中用于 SQLite 数据库的“schema”名称不会正确引用模式名称的问题。感谢 Phillip Cloud 的拉取请求。

## 1.2.10

发布日期：2018 年 7 月 13 日

### orm

+   **[orm] [bug]**

    修复了`Bundle`构造中的错误，当放置两个同名列时会被去重，当`Bundle`被用作渲染的 SQL 的一部分，比如在语句的 ORDER BY 或 GROUP BY 中。

    参考：[#4295](https://www.sqlalchemy.org/trac/ticket/4295)

+   **[orm] [bug]**

    由于[#4287](https://www.sqlalchemy.org/trac/ticket/4287)导致 1.2.9 中的回归，使用`Load`选项与字符串通配符结合使用会导致 TypeError。

    参考：[#4298](https://www.sqlalchemy.org/trac/ticket/4298)

### sql

+   **[sql] [bug]**

    修复了在任何引用它的`Table`之前显式删除`Sequence`的错误，当序列还涉及到该表的服务器端默认值时，当使用`MetaData.drop_all()`时会中断。现在，在删除表本身之后调用处理要通过非服务器端列默认函数删除的序列的步骤。

    参考：[#4300](https://www.sqlalchemy.org/trac/ticket/4300)

## 1.2.9

发布日期：2018 年 6 月 29 日

### orm

+   **[orm] [bug]**

    修复了在`Query.join()`中链接多个连接元素可能无法正确适应先前的左侧的问题，当链接共享相同基类的连接继承类时。

    参考：[#3505](https://www.sqlalchemy.org/trac/ticket/3505)

+   **[orm] [bug]**

    修复了为烘焙查询生成缓存键时的错误，可能导致生成的缓存键过短，对于跨子类的急加载情况。这可能会导致急加载查询被缓存，而不是非急加载查询，或者反之，对于多态的“selectin”加载，或者可能也适用于延迟加载或 selectin 加载。

    参考：[#4287](https://www.sqlalchemy.org/trac/ticket/4287)

+   **[orm] [bug]**

    修复了新的多态 selectin 加载中的错误，其中内部使用的 BakedQuery 会被给定的加载器选项改变，这既会不适当地改变子类查询，也会将效果传递给后续查询。

    参考：[#4286](https://www.sqlalchemy.org/trac/ticket/4286)

+   **[orm] [bug]**

    修复了由[#4256](https://www.sqlalchemy.org/trac/ticket/4256)引起的回归（本身是对[#4228](https://www.sqlalchemy.org/trac/ticket/4228)的回归修复），它破坏了一个未记录的行为，将直接传递给`Query`构造函数的实体的非序列转换为单个元素序列。虽然这种行为从未得到支持或记录，但已经在使用中，因此已被添加为`Query`的行为契约。

    参考：[#4269](https://www.sqlalchemy.org/trac/ticket/4269)

+   **[orm] [bug]**

    修复了一个性能回归问题，1.2 版本中的一个不正确结果涉及“baked”懒惰加载器，涉及从原始`Query`对象的加载器选项生成缓存键的问题。如果加载器选项是以“分支”样式构建的，其中使用了多个选项的公共基本元素，那么相同的选项将被重复渲染到缓存键中，这将导致性能问题以及生成错误的缓存键。修复了此问题，同时通过`Query.options()`应用这种“分支”选项时进行了性能改进，以防止重复应用相同的选项对象。

    参考：[#4270](https://www.sqlalchemy.org/trac/ticket/4270)

### sql

+   **[sql] [bug]**

    修复了 1.2 版本中的回归问题，该问题是由于[#4147](https://www.sqlalchemy.org/trac/ticket/4147)导致的，其中一个`Table`的部分索引列被重新定义为新列，这会在反射期间覆盖列或使用`Table.extend_existing`时发生，因此当尝试复制这些索引时，`Table.tometadata()`方法会失败，因为它们仍然指向被替换的列。现在的复制逻辑已经适应了这种情况。

    参考：[#4279](https://www.sqlalchemy.org/trac/ticket/4279)

### mysql

+   **[mysql] [bug]**

    修复了 mysql-connector-python 方言中的百分号双倍问题，该问题不需要去除百分号的双倍。此外，mysql-connector-python 驱动程序在传递列名到 cursor.description 时是不一致的，因此已添加了一个解码器的解决方法，条件地将这些随机-有时是字节的值解码为 Unicode，仅在需要时解码。还改进了 mysql-connector-python 的测试支持，但应注意，该驱动程序仍然存在与 Unicode 相关的问题，目前尚未解决。

+   **[mysql] [bug]**

    修复了索引反射中的 bug，在 MySQL 8.0 上，包含 ASC 或 DESC 的索引列规范的索引将不会被正确反映，因为 MySQL 8.0 引入了在表定义字符串中返回此信息的支持。

    参考：[#4293](https://www.sqlalchemy.org/trac/ticket/4293)

+   **[mysql] [bug]**

    修复了 MySQLdb 方言和 PyMySQL 等变体中的 bug，其中连接时对“unicode 返回”进行额外检查，明确使用“utf8”字符集，而在 MySQL 8.0 中会发出警告，建议使用 utf8mb4。现在已经用 utf8mb4 替换了这个。MySQL 方言的文档也已更新，以在所有示例中指定 utf8mb4。还对测试套件进行了其他更改，以使用 utf8mb3 字符集和数据库（在某些极端情况下，utf8mb4 存在排序问题），以及支持 MySQL 8.0 中的配置默认更改，例如 explicit_defaults_for_timestamp 以及为无效 MyISAM 索引引发的新错误。

    参考：[#4283](https://www.sqlalchemy.org/trac/ticket/4283)

+   **[mysql] [bug]**

    `Update` 结构现在支持 `Join` 对象，这是 MySQL 对 UPDATE..FROM 支持的方式。由于该结构已经接受了一个别名对象用于类似的目的，因此已经隐含了对非表进行 UPDATE 的功能，因此现在已经添加了此功能。

    参考：[#3645](https://www.sqlalchemy.org/trac/ticket/3645)

### sqlite

+   **[sqlite] [bug]**

    修复了测试套件中的问题，在 SQLite 3.24 中添加了一个新的保留字，与 TypeReflectionTest 中的使用冲突。Pull request 由 Nils Philippsen 提供。

### mssql

+   **[mssql] [bug]**

    修复了 MSSQL 反射中的 bug，在不同模式中具有相同名称的两个表具有相同名称的主键约束时，引用其中一个表的外键约束的列将会重复，导致错误。Pull request 由 Sean Dunn 提供。

    参考：[#4288](https://www.sqlalchemy.org/trac/ticket/4288)

+   **[mssql] [bug] [py3k]**

    修复了在 Python 3 下运行时针对非标准 SQL Server 数据库的问题，在这种数据库中不包含“sys.dm_exec_sessions”或“sys.dm_pdw_nodes_exec_sessions”视图，导致无法获取隔离级别时，由于 UnboundLocalError 而失败。

    参考：[#4273](https://www.sqlalchemy.org/trac/ticket/4273)

### oracle

+   **[oracle] [feature]**

    添加了一个新的事件，目前只有 cx_Oracle 方言使用，`DialectEvents.setiputsizes()`。该事件将 `BindParameter` 对象的字典传递给特定于 DBAPI 的类型对象，这些对象将被传递到 cx_Oracle `cursor.setinputsizes()` 方法中。这允许对 setinputsizes 过程进行可见性，并能够修改传递给此方法的数据类型的行为。

    另请参阅

    使用 setinputsizes 对 cx_Oracle 数据绑定性能进行精细控制

    参考：[#4290](https://www.sqlalchemy.org/trac/ticket/4290)

+   **[oracle] [错误] [mysql]**

    修复了 Oracle 和 MySQL 方言的 INSERT FROM SELECT 与 CTEs 一起使用时的问题，其中 CTE 被放置在整个语句的上方，这与其他数据库的典型做法相同，但是 Oracle 和 MariaDB 10.2 希望 CTE 位于“INSERT”段的下方。请注意，当将 CTE 应用于 UPDATE 或 DELETE 语句内部的子查询时，Oracle 和 MySQL 方言尚不起作用，因为 CTE 仍然应用于顶部而不是内部子查询。

    参考：[#4275](https://www.sqlalchemy.org/trac/ticket/4275)

### 杂项

+   **[功能] [扩展]**

    添加了新属性`Query.lazy_loaded_from`，其中填充了一个使用此`Query`来延迟加载关系的`InstanceState`。这样做的理由是它作为水平分片功能的提示，使得状态的标识令牌可以作为查询中 id_chooser()的默认标识令牌使用。

    参考：[#4243](https://www.sqlalchemy.org/trac/ticket/4243)

+   **[错误] [py3k]**

    用从 Python 标准库复制的一个供应版本替换了 inspect.formatargspec()的使用，因为 inspect.formatargspec()已被弃用，并且从 Python 3.7.0 开始发出警告。

    参考：[#4291](https://www.sqlalchemy.org/trac/ticket/4291)

## 1.2.8

发布日期：2018 年 5 月 28 日

### orm

+   **[orm] [错误]**

    由于[#4228](https://www.sqlalchemy.org/trac/ticket/4228)引起的 1.2.7 中的回归已修复，该回归本身是修复 1.2 级别回归的，其中假定传递给`Session`的`query_cls`可调用对象是`Query`的子类，并具有类方法可用性，而不是任意可调用对象。特别是，dogpile 缓存示例说明了`query_cls`作为函数而不是`Query`子类。

    参考：[#4256](https://www.sqlalchemy.org/trac/ticket/4256)

+   **[orm] [错误]**

    修复了 1.0 版本中长期存在的回归问题，该问题阻止了使用自定义`MapperOption`来改变`Query`对象的 _params 以进行延迟加载，因为延迟加载器本身会覆盖这些参数。这适用于维基上的“时间范围”示例。但请注意，现在需要使用`Query.populate_existing()`方法来重写与已加载到标识映射中的对象相关联的映射器选项。

    作为这一变更的一部分，现在自定义定义的`MapperOption`将导致与目标对象相关的延迟加载器默认使用非烘焙查询，除非实现了`MapperOption._generate_cache_key()`方法。特别是，这修复了一个回归，当使用 dogpile.cache 的“高级”示例时发生了问题，该示例由于与烘焙查询加载器不兼容而未返回缓存结果，而是发出 SQL；通过这一变更，dogpile 示例中包含的多个版本的`RelationshipCache`选项将完全禁用“烘焙”查询。请注意，dogpile 示例也经过现代化处理，以避免这些问题，作为问题[#4258](https://www.sqlalchemy.org/trac/ticket/4258)的一部分。

    参考：[#4128](https://www.sqlalchemy.org/trac/ticket/4128)

+   **[orm] [bug]**

    修复了新的`Result.with_post_criteria()`方法与子查询急切加载器无法正确交互的错误，即“后置条件”不会应用于嵌入式子查询急切加载器。这与[#4128](https://www.sqlalchemy.org/trac/ticket/4128)相关，因为现在延迟加载器使用了后置条件功能。

+   **[orm] [bug]**

    更新了 dogpile.caching 示例，包括适应“烘焙”查询系统的新结构，该系统默认在延迟加载器和一些急切关系加载器中使用。dogpile.caching 的“relationship_caching”和“advanced”示例也由于[#4256](https://www.sqlalchemy.org/trac/ticket/4256)而中断。这里的问题也通过[#4128](https://www.sqlalchemy.org/trac/ticket/4128)中的修复来解决。

    参考：[#4258](https://www.sqlalchemy.org/trac/ticket/4258)

### 引擎

+   **[引擎] [bug]**

    修复了连接池问题，即如果在连接池的“返回时重置”序列中引发了断开连接错误，并且针对封闭的`Connection`对象打开了显式事务（例如从调用`Session.close()`而没有回滚或提交，或者在没有首先关闭使用`Connection.begin()`声明的事务的情况下调用`Connection.close()`），将导致双重签入，这可能会导致对同一连接的并发签出。现在通过断言总体上防止了双重签入条件，同时还修复了特定的双重签入场景。

    参考：[#4252](https://www.sqlalchemy.org/trac/ticket/4252)

+   **[engine] [bug]**

    修复了引用泄漏问题，即在语句执行中使用的参数字典的值仍然被“编译缓存”引用，因为存储了 Python 3 字典 keys() 使用的键视图。感谢 Olivier Grisel 提交的拉取请求。

### sql

+   **[sql] [bug]**

    修复了在将文字值解释为 SQL 表达式值时遇到元组值时使用的“模棱两可的文字”错误消息，并且未能正确格式化消息的问题。感谢 Miguel Ventura 提交的拉取请求。

### mssql

+   **[mssql] [bug]**

    修复了由[#4061](https://www.sqlalchemy.org/trac/ticket/4061)引起的 1.2 版本回归问题，其中 SQL Server 的“BIT”类型被认为是“本地布尔”。这里的目标是避免在列上创建 CHECK 约束，然而更大的问题是 BIT 值不像真/假常量那样行为，并且不能被解释为独立表达式，例如“WHERE <column>”。SQL Server 方言现在回到非本地布尔，但增加了一个额外的标志，仍然避免创建 CHECK 约束。

    参考：[#4250](https://www.sqlalchemy.org/trac/ticket/4250)

### oracle

+   **[oracle] [bug]**

    Oracle 的 BINARY_FLOAT 和 BINARY_DOUBLE 数据类型现在在 cx_Oracle.setinputsizes()中参与，传递 NATIVE_FLOAT，以支持 NaN 值。此外，`BINARY_FLOAT`、`BINARY_DOUBLE`和`DOUBLE_PRECISION`现在是`Float`的子类，因为这些是浮点数据类型，而不是十进制数据类型。这些数据类型已经默认将`Float.asdecimal`标志设置为 False，以与`Float`的行为保持一致。

    参考：[#4264](https://www.sqlalchemy.org/trac/ticket/4264)

+   **[oracle] [bug]**

    为`BINARY_FLOAT`、`BINARY_DOUBLE`数据类型添加了反射功能。

+   **[oracle] [bug]**

    更改了 Oracle 方言，以便在使用`Integer`类型时，设置 cx_Oracle.NUMERIC 类型的 setinputsizes()。在 SQLAlchemy 1.1 及更早版本中，cx_Oracle.NUMERIC 无条件地传递给所有数值类型，并且在 1.2 中已删除，以允许更好的数值精度。但是，对于整数，一些数据库/客户端设置将无法将布尔值 True/False 强制转换为整数，这在使用 SQLAlchemy 1.2 时引入了回归行为。总的来说，setinputsizes 逻辑似乎需要更多的灵活性，这是一个开始。

    参考：[#4259](https://www.sqlalchemy.org/trac/ticket/4259)

### 测试

+   **[tests] [bug]**

    修复了测试套件中的一个 bug，如果外部方言返回`None`作为`server_version_info`，排除逻辑将引发`AttributeError`。

    参考：[#4249](https://www.sqlalchemy.org/trac/ticket/4249)

### 杂项

+   **[bug] [ext]**

    水平分片扩展现在利用了作为[#4137](https://www.sqlalchemy.org/trac/ticket/4137)的一部分添加到 ORM 身份密钥中的身份令牌，当对象刷新或基于列的延迟加载或取消过期操作发生时。由于我们知道对象的“分片”来自哪里，所以在刷新时我们利用了这个值，从而避免针对其他不匹配此对象身份的分片的查询。

    参考：[#4247](https://www.sqlalchemy.org/trac/ticket/4247)

+   **[bug] [ext]**

    修复了一个竞争条件，可能会在多线程环境中使用 automap `AutomapBase.prepare()` 与其他可能调用 `configure_mappers()` 的线程同时进行时发生。 automap 的未完成映射工作特别容易被 `configure_mappers()` 步骤所引入，导致错误。

    参考：[#4266](https://www.sqlalchemy.org/trac/ticket/4266)

## 1.2.7

发布日期：2018 年 4 月 20 日

### orm

+   **[orm] [bug]**

    修复了 1.2 版本中分片查询功能中的回归问题，其中新的“identity_token”元素在搜索相关的多对一元素时，在延迟加载操作的范围内未被正确考虑。新的行为将允许利用“id_chooser”来确定从身份映射中检索的最佳身份键。为了实现这一点，对 1.2 的“identity_token”方法进行了一些重构，对`ShardedQuery`的实现进行了一些细微更改，其他派生类应该注意这些更改。

    参考：[#4228](https://www.sqlalchemy.org/trac/ticket/4228)

+   **[orm] [bug]**

    修复了单继承加载中的问题，其中在使用`Query.select_from()`方法针对单继承子类使用别名实体时，会导致 SQL 呈现为未别名化的表混入查询，导致笛卡尔积。特别是当针对单继承子类使用新的“selectin”加载器时，会受到影响。

    参考：[#4241](https://www.sqlalchemy.org/trac/ticket/4241)

### sql

+   **[sql] [bug]**

    修复了使用“literal_binds”选项编译带有显式序列和“inline”生成的 INSERT 语句时，例如在 PostgreSQL 和 Oracle 上，会在序列处理过程中无法容纳额外关键字参数的问题。

    参考：[#4231](https://www.sqlalchemy.org/trac/ticket/4231)

### postgresql

+   **[postgresql] [feature]**

    添加了新的 PG 类型`REGCLASS`，有助于将表名转换为 OID 值。感谢 Sebastian Bank 的拉取请求。

    参考：[#4160](https://www.sqlalchemy.org/trac/ticket/4160)

+   **[postgresql] [bug]**

    修复了对于 PostgreSQL“range”数据类型（如 DATERANGE）的特殊“不等于”运算符与 Python 的`None`值进行比较时，无法呈现“IS NOT NULL”的错误。

    参考：[#4229](https://www.sqlalchemy.org/trac/ticket/4229)

### mssql

+   **[mssql] [bug]**

    修复了由 [#4060](https://www.sqlalchemy.org/trac/ticket/4060) 引起的 1.2 版本回归，其中用于反映 SQL Server 跨模式外键的查询错误地限制了条件。

    参考：[#4234](https://www.sqlalchemy.org/trac/ticket/4234)

### oracle

+   **[oracle] [错误]**

    如果精度为 NULL 并且比例为零，则 Oracle NUMBER 数据类型将反映为 INTEGER，因为这是从 Oracle 表中反映 INTEGER 值时的方式。感谢 Kent Bower 的拉取请求。

## 1.2.6

发布日期：2018 年 3 月 30 日

### ORM

+   **[ORM] [错误]**

    修复了一个 bug，当与具有用不同名称属性设置的非主映射器的类一起使用 `Mutable.associate_with()` 或 `Mutable.as_mutable()` 时，会产生属性错误。由于非主映射器不用于持久性，mutable 扩展现在将非主映射器排除在其仪器化步骤之外。

    参考：[#4215](https://www.sqlalchemy.org/trac/ticket/4215)

### 引擎

+   **[引擎] [错误]**

    修复了连接池中的 bug，如果先前的“connect”处理程序抛出异常，则连接可能存在于池中，而没有调用所有“connect”事件处理程序；请注意，方言本身具有发出 SQL 的 connect 处理程序，例如设置事务隔离的处理程序，如果数据库处于不可用状态，则可能失败，但仍允许连接。如果任何连接处理程序失败，首先使连接无效。

    参考：[#4225](https://www.sqlalchemy.org/trac/ticket/4225)

### SQL

+   **[SQL] [错误]**

    修复了在 1.2.5 版本中从先前对 [#4204](https://www.sqlalchemy.org/trac/ticket/4204) 的修复中发生的回归，其中在调用 `CTE.alias()` 方法后引用自身的 CTE 将无法正确引用自身。

    参考：[#4204](https://www.sqlalchemy.org/trac/ticket/4204)

### PostgreSQL

+   **[postgresql] [特性]**

    添加了对 PostgreSQL 表定义中“PARTITION BY”的支持，使用“postgresql_partition_by”。感谢 Vsevolod Solovyov 的拉取请求。

### MSSQL

+   **[mssql] [错误]**

    调整了对于 pyodbc 的 SQL Server 版本检测，只允许数字标记，过滤掉非整数，因为该方言使用元组-数字比较这个值。这在所有已知的 SQL Server / pyodbc 驱动程序中通常都是正确的。

    参考：[#4227](https://www.sqlalchemy.org/trac/ticket/4227)

### oracle

+   **[oracle] [错误]**

    支持的最低 cx_Oracle 版本为 5.2（2015 年 6 月）。以前，该方言对版本 5.0 进行了断言，但从 1.2.2 开始，我们使用了一些直到 5.2 才出现的符号。

    参考：[#4211](https://www.sqlalchemy.org/trac/ticket/4211)

### 杂项

+   **[bug] [declarative]**

    移除了在调用`__table_args__`、`__mapper_args__`时会发出警告的问题，这些方法是通过`@declared_attr`方法命名的，在非映射的声明性混合类中调用时。在映射类上重写这些方法时，直接调用它们是文档中记录的方法。对于常规属性名称，警告仍会发出。

    参考：[#4221](https://www.sqlalchemy.org/trac/ticket/4221)

## 1.2.5

发布日期：2018 年 3 月 6 日

### orm

+   **[orm] [feature]**

    添加了新功能`Query.only_return_tuples()`。导致`Query`对象无条件地返回键值元组对象，即使查询针对单个实体。感谢 Eric Atkin 的 Pull 请求。

+   **[orm] [bug]**

    修复了新的“多态 selectin”加载中的错误，当从关系懒加载器部分加载多态对象的选择时，会导致加载中出现“空 IN”条件，在“IN”的“内联”形式中引发错误。

    参考：[#4199](https://www.sqlalchemy.org/trac/ticket/4199)

+   **[orm] [bug]**

    修复了 1.2 版本中的一个回归问题，其中包含一个`AliasedClass`对象的映射选项，在使用`QueryableAttribute.of_type()`方法时，无法被 pickle 化。1.1 版本的行为是从路径中省略别名类对象，因此恢复了这种行为。

    参考：[#4209](https://www.sqlalchemy.org/trac/ticket/4209)

### sql

+   **[sql] [bug]**

    修复了:class:.`CTE`构造中的错误，与[#4204](https://www.sqlalchemy.org/trac/ticket/4204)中的问题类似，其中一个被别名的`CTE`在“克隆”操作期间无法正确复制自身，这在 ORM 中经常发生，也在使用`ClauseElement.params()`方法时发生。

    参考：[#4210](https://www.sqlalchemy.org/trac/ticket/4210)

+   **[sql] [bug]**

    修复了 CTE 渲染中的一个错误，当一个`CTE`也被转换为一个`Alias`时，如果在 FROM 子句中有多个对 CTE 的引用，则其“ctename AS aliasname”子句不会被适当地渲染。

    参考：[#4204](https://www.sqlalchemy.org/trac/ticket/4204)

+   **[sql] [bug]**

    修复了新“扩展 IN 参数”功能中绑定参数处理器的错误，其中值根本无法工作，测试未覆盖这个非常基本的情况，其中包括 ENUM 值无法工作。

    参考：[#4198](https://www.sqlalchemy.org/trac/ticket/4198)

### postgresql

+   **[postgresql] [错误] [py3k]**

    修复了在 PostgreSQL COLLATE / ARRAY 调整中首次引入的错误，最初在[#4006](https://www.sqlalchemy.org/trac/ticket/4006)中，Python 3.7 正则表达式的新行为导致修复失败。

    此更改也**回溯**到：1.1.18

    参考：[#4208](https://www.sqlalchemy.org/trac/ticket/4208)

### mysql

+   **[mysql] [错误]**

    MySQL 方言现在明确使用`SELECT @@version`查询服务器版本，以确保我们获得正确的版本信息。代理服务器如 MaxScale 会干扰传递给 DBAPI 的 connection.server_version 值，因此这不再可靠。

    此更改也**回溯**到：1.1.18

    参考：[#4205](https://www.sqlalchemy.org/trac/ticket/4205)

## 1.2.4

发布日期：2018 年 2 月 22 日

### orm

+   **[orm] [错误]**

    修复了 ORM 版本控制功能中的 1.2 回归错误，其中针对`select()`或`alias()`的映射，还使用了对基础表的版本控制列，由于添加的检查部分[#3673](https://www.sqlalchemy.org/trac/ticket/3673)而失败。

    参考：[#4193](https://www.sqlalchemy.org/trac/ticket/4193)

### 引擎

+   **[引擎] [错误]**

    由于从[#4181](https://www.sqlalchemy.org/trac/ticket/4181)修复引起的 1.2.3 版本的回归错误，事件系统中涉及`Engine`和`OptionEngine`的更改未考虑到事件的移除，当在类级别调用时会引发`AttributeError`。

    参考：[#4190](https://www.sqlalchemy.org/trac/ticket/4190)

### sql

+   **[sql] [错误]**

    修复了 CTE 表达式在给定名称区分大小写或以其他方式需要引号时，其名称或别名未被引用的错误。感谢 Eric Atkin 提供的拉取请求。

    参考：[#4197](https://www.sqlalchemy.org/trac/ticket/4197)

## 1.2.3

发布日期：2018 年 2 月 16 日

### orm

+   **[orm] [功能]**

    向`set_attribute()`函数添加了新参数`set_attribute.inititator`，允许从监听器函数接收的事件令牌传播到后续设置事件。

+   **[orm] [错误]**

    修复了 post_update 功能中的问题，在父对象已删除但相关对象尚未删除时，会发出 UPDATE。这个问题已经存在很长时间，然而自从 1.2 版本现在为 post_update 断言匹配的行，这会引发错误。

    这个改变也 **回溯** 到：1.1.16

    参考：[#4187](https://www.sqlalchemy.org/trac/ticket/4187)

+   **[orm] [bug]**

    由于针对版本 1.2.2 以及 1.1.15 的问题 [#4116](https://www.sqlalchemy.org/trac/ticket/4116) 的修复导致的回归已经修复，该问题导致在一些声明混合/继承情况下以及如果访问未映射类的关联代理时，误计算 `AssociationProxy` 的 “拥有类” 为 `NoneType` 类。现在，“找出所有者”的逻辑已被一个深度程序替换，该程序搜索分配给类或子类的完整映射器层次结构，以确定正确（我们希望）的匹配；如果找不到匹配项，则不会分配所有者。如果代理针对未映射实例使用，现在会引发异常。

    这个改变也 **回溯** 到：1.1.16

    参考：[#4185](https://www.sqlalchemy.org/trac/ticket/4185)

+   **[orm] [bug]**

    修复了一个 bug，即 `Bundle` 对象没有正确报告由 bundle 表示的主 `Mapper` 对象，如果有的话。这个问题的直接副作用是新的 selectinload 加载策略无法与水平分片扩展一起工作。

    参考：[#4175](https://www.sqlalchemy.org/trac/ticket/4175)

+   **[orm] [bug]**

    修复了具体继承映射中的 bug，即用户定义的属性（例如与兄弟类的映射属性同名的混合属性）将被映射器覆盖为在实例级别不可访问。此外，确保在映射器配置阶段不会隐式调用用户绑定的描述符。

    参考：[#4188](https://www.sqlalchemy.org/trac/ticket/4188)

+   **[orm] [bug]**

    修复了一个 bug，即 `reconstructor()` 事件助手如果应用于映射类的 `__init__()` 方法，则不会被识别。

    参考：[#4178](https://www.sqlalchemy.org/trac/ticket/4178)

### engine

+   **[engine] [bug]**

    修复了与类级别的 `Engine` 关联的事件在使用 `Engine.execution_options()` 方法时会重复的 bug。为了实现这一点，半私有类 `OptionEngine` 不再直接在类级别接受事件，并将引发错误；该类仅从其父 `Engine` 传播类级别事件。实例级别事件继续像以前一样工作。

    参考：[#4181](https://www.sqlalchemy.org/trac/ticket/4181)

+   **[engine] [bug]**

    `URL` 对象现在允许多次指定查询键，其值将被连接成列表。这是为了支持插件功能，文档记录在 `CreateEnginePlugin` 中，文档指出“plugin”可以多次传递。此外，插件名称可以通过新的 `create_engine.plugins` 参数在 URL 之外传递给 `create_engine()`。

    参考：[#4170](https://www.sqlalchemy.org/trac/ticket/4170)

### sql

+   **[sql] [feature]**

    添加了对 `Enum` 的支持，以持久化枚举的值，而不是键，当使用 Python pep-435 风格的枚举对象时。用户提供一个可调用函数，该函数将返回要持久化的字符串值。这允许对非字符串值的枚举也可以进行值持久化。感谢 Jon Snyder 提交的拉取请求。

    参考：[#3906](https://www.sqlalchemy.org/trac/ticket/3906)

+   **[sql] [bug]**

    修复了`Enum`类型无法正确处理枚举“别名”的 bug，当多个键引用相同值时。感谢 Daniel Knell 提交的拉取请求。

    参考：[#4180](https://www.sqlalchemy.org/trac/ticket/4180)

### postgresql

+   **[postgresql] [bug]**

    将“SSL SYSCALL error: Operation timed out”添加到触发 psycopg2 驱动程序“断开连接”场景的消息列表中。感谢 André Cruz 提交的���取请求。

    此更改也**回溯**至：1.1.16

+   **[postgresql] [bug]**

    将“TRUNCATE”添加到 PostgreSQL 方言接受的关键字列表中，作为“autocommit”触发关键字。感谢 Jacob Hayes 提交的拉取请求。

    此更改也**回溯**至：1.1.16

### sqlite

+   **[sqlite] [bug]**

    修复了当平台既没有安装 pysqlite2 也没有安装 sqlite3 时引发的导入错误，使得引发与 sqlite3 相关的导入错误，而不是实际的失败模式 pysqlite2。感谢 Robin 的拉取请求。

### oracle

+   **[oracle] [feature]**

    外键的 ON DELETE 选项现在是 Oracle 反射的一部分。Oracle 不支持 ON UPDATE 级联。感谢 Miroslav Shubernetskiy 的拉取请求。

+   **[oracle] [bug]**

    修复了 cx_Oracle 断开连接检测中的 bug，该 bug 用于 pre_ping 和其他功能，之前可能会引发一个包含数字错误代码的 DatabaseError；之前我们没有在这种情况下检查断开连接代码。

    参考：[#4182](https://www.sqlalchemy.org/trac/ticket/4182)

### tests

+   **[tests] [bug]**

    在 1.2 版本中添加的一个测试，旨在确认 Python 2.7 行为，结果只确认了 Python 2.7.8 的行为。Python bug #8743 仍然影响 Python 2.7.7 及更早版本中的集合比较，因此涉及 AssociationSet 的测试不再适用于这些较旧的 Python 2.7 版本。

    参考：[#3265](https://www.sqlalchemy.org/trac/ticket/3265)

### misc

+   **[bug] [pool]**

    修复了一个相当严重的连接池 bug，当一个连接在由用户定义的 `DisconnectionError` 或由 1.2 版本发布的“pre_ping”功能刷新后被获取时，如果连接由 weakref 清理返回到池中（例如前端对象被垃圾回收），则如果 weakref 仍然指向先前失效的 DBAPI 连接，那么将不会正确重置连接；这将导致日志中的堆栈跟踪和一个连接被检入池中而未被重置，这可能导致锁定问题。

    此更改也被**回溯**到：1.1.16

    参考：[#4184](https://www.sqlalchemy.org/trac/ticket/4184)

## 1.2.2

发布日期：2018 年 1 月 24 日

### orm

+   **[orm] [bug]**

    修复了 1.2 版本中关于新 bulk_replace 事件的回归，其中一个反向引用在批量赋值将对象分配给新所有者时，未能从先前所有者中删除对象。

    参考：[#4171](https://www.sqlalchemy.org/trac/ticket/4171)

### mysql

+   **[mysql] [bug]**

    为了引用目的，向 MySQL 方言添加了更多 MySQL 8.0 保留字。感谢 Riccardo Magliocchetti 的拉取请求。

### mssql

+   **[mssql] [bug]**

    将 ODBC 错误代码 10054 添加到作为 ODBC / MSSQL 服务器断开连接的错误代码列表中。

    参考：[#4164](https://www.sqlalchemy.org/trac/ticket/4164)

### oracle

+   **[oracle] [bug]**

    cx_Oracle 方言现在在使用 NVARCHAR2 数据类型时，无条件地调用 setinputsizes()，其中 SQLAlchemy 中对应的是 sqltypes.Unicode()。根据 cx_Oracle 的作者，这样可以在 Oracle 客户端内正确进行转换，而不受 NLS_NCHAR_CHARACTERSET 设置的影响。

    参考：[#4163](https://www.sqlalchemy.org/trac/ticket/4163)

## 1.2.1

发布日期：2018 年 1 月 15 日

### orm

+   **[orm] [bug]**

    修复了在嵌套或子事务回滚期间从会话中正确移除的对象的回归，该对象的主键也已发生突变，从而导致使用会话时出现后续问题的错误。

    此更改也**被回溯**到：1.1.16

    参考：[#4151](https://www.sqlalchemy.org/trac/ticket/4151)

+   **[orm] [bug]**

    修复了 pickle 格式的 Load / _UnboundLoad 对象（例如加载器选项）的回归，即使尝试这样做，从旧格式接收到的对象`__setstate__()`也会因为 UnboundLocalError 而引发异常。现在添加了测试以确保此功能正常工作。

    参考：[#4159](https://www.sqlalchemy.org/trac/ticket/4159)

+   **[orm] [bug]**

    修复了[#3954](https://www.sqlalchemy.org/trac/ticket/3954)中引入的回归，新的 lazyload 缓存方案导致具有 of_type 的加载器选项的查询会导致无关路径的惰性加载失败，从而导致 TypeError。

    参考：[#4153](https://www.sqlalchemy.org/trac/ticket/4153)

+   **[orm] [bug]**

    修复了新的“selectin”关系加载器中的错误，当加载多态对象的集合时，加载器可能会尝试加载不存在的关系，其中只有一些映射器包括该关系，通常在使用`PropComparator.of_type()`时。

    参考：[#4156](https://www.sqlalchemy.org/trac/ticket/4156)

### sql

+   **[sql] [bug]**

    修复了`Insert.values()`中的错误，其中使用“多值”格式与`Column`对象作为键而不是字符串会失败。拉请求由 Aubrey Stark-Toller 提供。

    此更改也**被回溯**到：1.1.16

    参考：[#4162](https://www.sqlalchemy.org/trac/ticket/4162)

### mssql

+   **[mssql] [bug]**

    修复了 1.2 中新修复的引用中对排序名称的引用在[#3785](https://www.sqlalchemy.org/trac/ticket/3785)中破坏了 SQL Server 的问题，SQL Server 明确不理解排序名称的引用。现在，是否引用混合大小写排序名称已延迟到方言级别的决定，以便每个方言可以直接准备这些标识符。

    参考：[#4154](https://www.sqlalchemy.org/trac/ticket/4154)

### oracle

+   **[oracle] [bug]**

    修复了从 cx_Oracle 方言中删除大多数 setinputsizes 规则导致 TIMESTAMP 数据类型无法检索小数秒的问题。

    参考：[#4157](https://www.sqlalchemy.org/trac/ticket/4157)

+   **[oracle] [bug]**

    修复了 Oracle 导入中的回归，其中缺少逗号导致出现未定义的符号。拉请求由 Miroslav Shubernetskiy 提供。

### 测试

+   **[测试] [bug]**

    从公共测试套件中删除了一个特定于 Oracle 的要求规则，该规则干扰了第三方方言套件。

+   **[测试] [bug]**

    添加了一个新的排除规则组 `group_by_complex_expression`，禁用了使用“GROUP BY <expr>”的测试，这似乎对至少两个第三方方言不可行。

### 杂项

+   **[bug] [扩展]**

    修复了联合代理中的回归问题，原因是 [#3769](https://www.sqlalchemy.org/trac/ticket/3769)（允许链式的 any() / has()）其中一个调用了一个针对联合代理的 contains() 错误链接形式（o2m 关系，联合代理（m2o 关系，m2o 关系）），将在链的最终链接上重新应用 contains() 时会引发错误。

    参考：[#4150](https://www.sqlalchemy.org/trac/ticket/4150)

## 1.2.0

发布日期：2017 年 12 月 27 日

### orm

+   **[orm] [功能]**

    在 ORM 的标识映射中添加了一个新的数据成员，称为“identity_token”。此标记默认为 None，但可以被数据库分片方案用来区分来自不同数据库的具有相同主键的内存对象。水平分片扩展将此标记应用到 shard 标识符上，从而允许主键在水平分片后端之间重复。

    另见

    标识键增强以支持分片

    参考：[#4137](https://www.sqlalchemy.org/trac/ticket/4137)

+   **[orm] [bug] [扩展]**

    修复了联合代理中的错误，如果先使用 `AliasedClass` 作为父类调用联合代理，那么联合代理会意外地将自身链接到 `AliasedClass` 对象上，在后续使用时会引发错误。

    此更改也被 **回溯** 至：1.1.15

    参考：[#4116](https://www.sqlalchemy.org/trac/ticket/4116)

+   **[orm] [bug]**

    修复了 `contains_eager()` 查询选项中的错误，使用跨越多个连接级别引用子类的路径会要求“别名”参数也提供相同的子类型，以避免向查询添加不必要的 FROM 子句；另外，使用 `contains_eager()` 跨越使用子类的 `aliased()` 对象作为 `PropComparator.of_type()` 参数的子类也会正确渲染。

    参考：[#4130](https://www.sqlalchemy.org/trac/ticket/4130)

+   **[orm] [bug]**

    `Query.exists()`方法现在将在查询呈现时禁用急加载器。以前，连接急加载连接将被不必要地呈现，以及子查询急加载查询将被不必要地生成。新行为与`Query.subquery()`方法相匹配。

    参考：[#4032](https://www.sqlalchemy.org/trac/ticket/4032)

### orm 声明

+   **[orm] [declarative] [bug]**

    修复了一个错误，其中描述符（即基于`AbstractConcreteBase`的层次结构中的映射列或关系）在刷新操作期间被引用，导致错误，因为该属性未映射为映射器属性。如果类未在其映射器中包含“concrete=True”，则其他属性（如`AbstractConcreteBase`添加的“type”列）可能出现类似问题，但此处的检查也应防止该情况引起问题。

    此更改也**回溯**到：1.1.15

    参考：[#4124](https://www.sqlalchemy.org/trac/ticket/4124)

### engine

+   **[engine] [feature]**

    `URL`对象的“password”属性现在可以是任何用户定义或用户子类化的字符串对象，该对象响应 Python 的`str()`内置函数。传递的对象将保持为数据成员`URL.password_original`，并且在读取`URL.password`属性以生成字符串值时将进行查询。

    参考：[#4089](https://www.sqlalchemy.org/trac/ticket/4089)

### sql

+   **[sql] [bug]**

    修复了一个错误，其中`ColumnDefault`的`__repr__`如果参数是元组，则会失败。感谢 Nicolas Caniart 的拉取请求。

    此更改也**回溯**到：1.1.15

    参考：[#4126](https://www.sqlalchemy.org/trac/ticket/4126)

+   **[sql] [bug]**

    重新设计了在 1.2.0b2 中引入的新的“autoescape”选项用于 startswith()，endswith()的“autoescape”功能，使其完全自动化；转义字符现在默认为斜杠`"/"`，并应用于百分号、下划线以及转义字符本身，以实现完全自动转义。也可以使用“escape”参数更改字符。

    请参阅

    新的“autoescape”选项用于 startswith()，endswith()

    参考：[#2694](https://www.sqlalchemy.org/trac/ticket/2694)

+   **[sql] [bug]**

    修复了`Table.tometadata()`方法无法正确适应不仅由简单列表达式组成的`Index`对象的问题，例如针对`text()`构造的索引，使用 SQL 表达式或`func`的索引等。现在该例程会完全复制表达式到一个新的`Index`对象，同时将所有与目标表的列绑定的`Column`对象替换为目标表的列。

    参考：[#4147](https://www.sqlalchemy.org/trac/ticket/4147)

+   **[sql] [错误]**

    将`ColumnElement`的“访问名称”从“column”更改为“column_element”，这样当此元素被用作用户定义的 SQL 元素的基础时，它不会被假定为在被 ORM 常用的各种 SQL 遍历工具处理时表现得像一个绑定到表的`ColumnClause`。

    参考：[#4142](https://www.sqlalchemy.org/trac/ticket/4142)

+   **[sql] [错误] [扩展]**

    修复了`ARRAY`数据类型中的问题，本质上与[#3832](https://www.sqlalchemy.org/trac/ticket/3832)的问题相同，只是不是一个回归问题，`ARRAY`上的列附加事件不会正确触发，从而干扰依赖于此的系统。这个问题破坏的一个关键用例是使用混入来声明使用`MutableList.as_mutable()`的列。

    参考：[#4141](https://www.sqlalchemy.org/trac/ticket/4141)

+   **[sql] [错误]**

    修复了新的“扩展绑定参数”功能中的错误，即如果一个语句中使用了多个参数，则正则表达式将无法正确匹配参数名。

    参考：[#4140](https://www.sqlalchemy.org/trac/ticket/4140)

+   **[sql] [增强]**

    为 PostgreSQL、MySQL、MS SQL Server（以及不支持的 Sybase 方言）实现了“DELETE..FROM”语法，类似于“UPDATE..FROM”的工作方式。引用了 Pieter Mulder 的拉取请求。

    另请参阅

    支持多表条件的 DELETE

    参考：[#959](https://www.sqlalchemy.org/trac/ticket/959)

### postgresql

+   **[postgresql] [特性]**

    添加了新的`MONEY`数据类型。感谢 Cleber J Santos 的拉取请求。

### mysql

+   **[mysql] [bug]**

    MySQL 5.7.20 现在警告使用@tx_isolation 变量；现在执行版本检查并使用@transaction_isolation 来代替以防止此警告。

    此更改也**回溯**到：1.1.15

    参考：[#4120](https://www.sqlalchemy.org/trac/ticket/4120)

+   **[mysql] [bug]**

    修复了从问题 1.2.0b3 中的回归，其中“MariaDB”版本比较在 Python 3 下可能会失败，对于某些特定的 MariaDB 版本字符串。

    参考：[#4115](https://www.sqlalchemy.org/trac/ticket/4115)

### mssql

+   **[mssql] [bug]**

    修复了 sqltypes.BINARY 和 sqltypes.VARBINARY 数据类型不包含正确的绑定值处理程序以用于 pyodbc 的错误，这允许传递帮助 FreeTDS 的 pyodbc.NullParam 值。

    参考：[#4121](https://www.sqlalchemy.org/trac/ticket/4121)

### oracle

+   **[oracle] [bug]**

    添加了一些额外规则，以完全处理`Decimal('Infinity')`，`Decimal('-Infinity')`值与 cx_Oracle 数字时使用`asdecimal=True`。

    参考：[#4064](https://www.sqlalchemy.org/trac/ticket/4064)

### misc

+   **[misc] [feature]**

    在文档中添加了一个新的错误部分，介绍常见错误消息的背景。SQLAlchemy 中的选定异常将在其字符串输出中包含指向此页面相关部分的链接。

+   **[enhancement] [ext]**

    在烘焙查询系统中添加了新方法`Result.with_post_criteria()`，允许在查询从缓存中拉取后进行非 SQL 修改转换。此方法可以与`ShardedQuery`一起使用，以设置分片标识符。`ShardedQuery`也已经修改，使其`ShardedQuery.get()`方法与`Result`的方法正确交互。

    参考：[#4135](https://www.sqlalchemy.org/trac/ticket/4135)

## 1.2.0b3

发布日期：2017 年 10 月 13 日

### orm

+   **[orm] [bug]**

    修复了 ORM 关系会警告针对在继承层次结构中的兄弟类中的冲突同步目标（例如，两个关系都将写入同一列）的错误，其中两个关系实际上永远不会在写入时发生冲突。

    此更改也**回溯**到：1.1.15

    参考：[#4078](https://www.sqlalchemy.org/trac/ticket/4078)

+   **[orm] [bug]**

    修复了针对单表继承实体使用相关选择时，在外部查询中无法正确呈现的错误，因为单一继承鉴别器标准的调整不当地重新应用于外部查询。

    此更改也**回溯**到：1.1.15

    参考：[#4103](https://www.sqlalchemy.org/trac/ticket/4103)

+   **[orm] [bug]**

    修复了`Session.merge()`中的错误，遵循与[#4030](https://www.sqlalchemy.org/trac/ticket/4030)类似的线路，其中对于标识映射中的目标对象的内部检查可能会导致错误，如果在合并程序实际检索对象之前立即对其进行垃圾回收。

    此更改也**回溯**到：1.1.14

    参考：[#4069](https://www.sqlalchemy.org/trac/ticket/4069)

+   **[orm] [bug]**

    修复了一个错误，即如果从使用连接式急加载加载的关系扩展，则不会识别`undefer_group()`选项。此外，由于该错误导致执行过多的工作，因此在结果集列的初始计算中，Python 函数调用次数也提高了 20%，这与[#3915](https://www.sqlalchemy.org/trac/ticket/3915)的连接急加载改进相辅相成。

    此更改也**回溯**到：1.1.14

    参考：[#4048](https://www.sqlalchemy.org/trac/ticket/4048)

+   **[orm] [bug]**

    修复了`Session.merge()`中的错误，其中集合中的对象的主键属性设置为`None`，对于通常是自动递增的键，将被视为数据库持久化键的一部分，导致在内部去重过程中实际上只插入一个对象到数据库中。

    此更改也**回溯**到：1.1.14

    参考：[#4056](https://www.sqlalchemy.org/trac/ticket/4056)

+   **[orm] [bug]**

    当针对不是针对`MapperProperty`的属性（例如关联代理）使用`synonym()`时，会引发`InvalidRequestError`。以前，尝试定位不存在的属性会导致递归溢出。

    此更改也**回溯**到：1.1.14

    参考：[#4067](https://www.sqlalchemy.org/trac/ticket/4067)

+   **[orm] [bug]**

    由于[#3934](https://www.sqlalchemy.org/trac/ticket/3934)引入的 1.2.0b1 版本中的回归问题已修复，`Session`在回滚失败时未能“停用”事务（目标问题是当 MySQL 丢失 SAVEPOINT 时）。 这将导致随后调用`Session.rollback()`再次引发错误，而不是完成并将`Session`恢复为活动状态。

    参考：[#4050](https://www.sqlalchemy.org/trac/ticket/4050)

+   **[orm] [bug]**

    修复了`make_transient_to_detached()`函数会使目标对象上的所有属性过期的问题，包括“延迟加载”属性，这会导致属性在下一次刷新时被取消延迟加载，从而导致属性意外加载。

    参考：[#4084](https://www.sqlalchemy.org/trac/ticket/4084)

+   **[orm] [bug]**

    修复了涉及删除孤立级联的错误，其中在父对象成为会话的一部分之前成为孤立项的相关项仍被跟踪为移动到孤立状态，这导致它被从会话中删除而不是被刷新。

    注意

    这个修复在 1.2.0b3 版本发布期间被错误地合并，并且**未被添加到更改日志**中。此更改日志注释已作为 1.2.13 版本的一部分追加到发布中。

    参考：[#4040](https://www.sqlalchemy.org/trac/ticket/4040)

+   **[orm] [bug]**

    修复了“selectin”多态加载，使用单独的 IN 查询加载子类中的错误，该错误阻止了多级类层次结构中“selectin”和“inline”设置按预期交互。

    参考：[#4026](https://www.sqlalchemy.org/trac/ticket/4026)

+   **[orm] [bug]**

    移除了当映射器和加载策略使用的 LRU 缓存达到阈值时发出的警告；最初这个警告的目的是防止生成过多的缓存键，但后来基本上成为“创建许多引擎”反模式的检查。虽然这仍然是一个反模式，但测试套件既为每个测试创建一个引擎又在所有警告上引发的存在将是一个不便；对于这个警告，这样的测试套件改变其架构并不是必要的（尽管每个测试一个引擎的套件总是更好）。

    参考：[#4071](https://www.sqlalchemy.org/trac/ticket/4071)

+   **[orm] [bug]**

    修复了在 1.2 版本中作为[#3954](https://www.sqlalchemy.org/trac/ticket/3954)的一部分添加的 SQL 缓存键生成中的错误，导致在与延迟加载关系选项一起使用`undefer_group()`选项���会导致属性错误。

    参考：[#4049](https://www.sqlalchemy.org/trac/ticket/4049)

+   **[orm] [bug]**

    修改了在[#3366](https://www.sqlalchemy.org/trac/ticket/3366)中对 ORM 更新/删除评估器所做的更改，如果更新或删除中存在未映射的列表达式，并且评估器可以将其名称与目标类的映射列匹配，将发出警告，而不是引发 UnevaluatableError。这本质上是 1.2 版本之前的行为，目的是允许依赖于此模式的应用程序进行迁移。但是，如果给定的属性名称无法与映射器的列匹配，仍会引发 UnevaluatableError，这是在[#3366](https://www.sqlalchemy.org/trac/ticket/3366)中修复的问题。

    参考：[#4073](https://www.sqlalchemy.org/trac/ticket/4073)

### orm declarative

+   **[orm] [declarative] [bug]**

    如果子类尝试覆盖在父类上使用`@declared_attr.cascading`声明的属性，则会发出警告，覆盖的属性将被忽略。这种用例无法在更复杂的开发工作下完全支持到更进一步的子类，因此为了一致性，“级联”将一直被遵守，无论覆盖属性如何。

    参考：[#4091](https://www.sqlalchemy.org/trac/ticket/4091)

+   **[orm] [declarative] [bug]**

    如果使用`@declared_attr.cascading`属性与特殊的声明名称（如`__tablename__`）一起使用，将发出警告，因为这没有任何效果。

    参考：[#4092](https://www.sqlalchemy.org/trac/ticket/4092)

### engine

+   **[engine] [feature]**

    向`ResultProxy`添加了`__next__()`和`next()`方法，以便`next()`内置函数直接在对象上起作用。`ResultProxy`长期以来已经有一个`__iter__()`方法，允许它响应`iter()`内置函数。`__iter__()`的实现保持不变，因为性能测试表明，使用带有`StopIteration`的`__next__()`方法进行迭代在 Python 2.7 和 3.6 中都要慢大约 20%。

    参考：[#4077](https://www.sqlalchemy.org/trac/ticket/4077)

+   **[engine] [bug]**

    对`Pool`和`Connection`进行了一些调整，使得在`pool.Empty`、`AttributeError`异常捕获中不再运行恢复逻辑，因为当恢复操作本身失败时，Python 3 会创建一个误导性的堆栈跟踪，将`Empty` / `AttributeError`误认为是原因，而实际上这些异常捕获是控制流的一部分。

    参考：[#4028](https://www.sqlalchemy.org/trac/ticket/4028)

### sql

+   **[sql] [bug]**

    修复了最近添加的 `ColumnOperators.any_()` 和 `ColumnOperators.all_()` 方法在作为方法调用时无法正常工作的 bug，而不是使用独立函数 `any_()` 和 `all_()`。还为这些相对晦涩的 SQL 操作符添加了文档示例。

    此更改也已**回溯**至：1.1.15

    参考：[#4093](https://www.sqlalchemy.org/trac/ticket/4093)

+   **[sql] [bug]**

    添加了一个新方法 `DefaultExecutionContext.get_current_parameters()`，该方法在基于函数的默认值生成器中使用，以检索传递给语句的当前参数。这个新函数与 `DefaultExecutionContext.current_parameters` 属性不同，它还提供了对应于多值“插入”结构的参数的可选分组。以前无法识别与函数调用相关的参数子集。

    另请参阅

    用于具有上下文默认生成器的多值插入的参数助手

    上下文敏感的默认函数

    参考：[#4075](https://www.sqlalchemy.org/trac/ticket/4075)

+   **[sql] [bug]**

    修复了新的 SQL 注释功能中的 bug，使用 `Table.tometadata()` 时，表和列的注释不会被复制。

    参考：[#4087](https://www.sqlalchemy.org/trac/ticket/4087)

+   **[sql] [bug]**

    在版本 1.1 中，`Boolean` 类型存在问题，即通过 `bool()` 进行布尔强制转换会发生在不支持“本地布尔”功能的后端，但不会发生在本地布尔后端，这意味着字符串 `"0"` 现在表现不一致。经过一次投票，达成共识，即非布尔值应该引发错误，特别是���字符串 `"0"` 的模棱两可情况下；因此，如果传入值不在范围 `None, True, False, 1, 0` 内，`Boolean` 数据类型现在将引发 `ValueError`。

    另请参阅

    布尔数据类型现在强制 True/False/None 值

    参考：[#4102](https://www.sqlalchemy.org/trac/ticket/4102)

+   **[sql] [bug]**

    优化了`Operators.op()`的行为，使得在所有情况下，如果`Operators.op.is_comparison`标志设置为 True，则结果表达式的返回类型将是`Boolean`，如果标志为 False，则结果表达式的返回类型将与左侧表达式的类型相同，这是其他运算符的典型默认行为。还添加了一个新参数`Operators.op.return_type`以及一个辅助方法`Operators.bool_op()`。

    另请参阅

    自定义运算符的类型行为已经保持一致

    参考：[#4063](https://www.sqlalchemy.org/trac/ticket/4063)

+   **[sql] [bug]**

    对`Enum`、`Interval`和`Boolean`类型进行了内部优化，现在这些类型都扩展了一个通用的 mixin `Emulated`，表示提供了对数据库本地类型的 Python 端模拟，在使用支持的后端时切换到数据库本地类型。直接使用 PostgreSQL `INTERVAL` 类型现在将包括正确的类型强制转换规则，这些规则也适用于 `Interval` 的 SQL 表达式（例如，将日期添加到间隔会产生日期时间）。

    引用：[#4088](https://www.sqlalchemy.org/trac/ticket/4088)

### postgresql

+   **[postgresql] [feature]**

    向 psycopg2 方言添加了一个新的标志 `use_batch_mode`。当 `Engine` 调用 `cursor.executemany()` 时，此标志允许使用 psycopg2 的 `psycopg2.extras.execute_batch` 扩展。这个扩展在批量运行 INSERT 语句时提供了至关重要的性能提升，增加了一个数量级。该标志默认为 False，因为目前被认为是实验性的。

    请参见

    支持批处理模式 / 快速执行助手

    引用：[#4109](https://www.sqlalchemy.org/trac/ticket/4109)

+   **[postgresql] [bug]**

    与 COLLATE 结合使用进一步修复了 `ARRAY` 类，因为在 [#4006](https://www.sqlalchemy.org/trac/ticket/4006) 中进行的修复未能适应多维数组。

    这个变更也被**回溯到**：1.1.15

    引用：[#4006](https://www.sqlalchemy.org/trac/ticket/4006)

+   **[postgresql] [bug]**

    修复了 `array_agg` 函数中的错误，当传递一个已经是 `ARRAY` 类型的参数时，比如 PostgreSQL 的 `array` 结构，会产生一个 `ValueError`，因为该函数试图嵌套数组。

    这个变更也被**回溯到**：1.1.15

    引用：[#4107](https://www.sqlalchemy.org/trac/ticket/4107)

+   **[postgresql] [bug]**

    修复了 PostgreSQL 中 `Insert.on_conflict_do_update()` 的错误，该错误将阻止将插入语句用作 CTE，例如通过 `Insert.cte()` 在另一个语句中。

    这个变更也被**回溯到**：1.1.15

    引用：[#4074](https://www.sqlalchemy.org/trac/ticket/4074)

+   **[postgresql] [bug]**

    修复了 pg8000 驱动器的错误，如果使用带有模式名称的 `MetaData.reflect()`，则会失败，因为模式名称将作为一个“quoted_name”对象发送，这是一个字符串子类，pg8000 不认识。连接时将 quoted_name 类型添加到 pg8000 的 py_types 集合中。

    引用：[#4041](https://www.sqlalchemy.org/trac/ticket/4041)

+   **[postgresql] [bug]**

    为 pg8000 驱动器启用了 UUID 支持，这支持本机 Python uuid 往返于此数据类型。然而，仍然不支持 UUID 数组。

    引用：[#4016](https://www.sqlalchemy.org/trac/ticket/4016)

### mysql

+   **[mysql] [bug]**

    当检测到 MariaDB 10.2.8 或更早版本的 10.2 系列时，会发出警告，因为这些版本中的 CHECK 约束存在重大问题，这些问题已在 10.2.9 解决。

    请注意，此更改日志消息未随 SQLAlchemy 1.2.0b3 一起发布，而是事后添加的。

    此更改也**回溯**到：1.1.15

    参考：[#4097](https://www.sqlalchemy.org/trac/ticket/4097)

+   **[mysql] [bug]**

    修复了在 MariaDB 10.2 系列中`CURRENT_TIMESTAMP`由于语法更改而无法正确反映的问题，现在该函数表示为`current_timestamp()`。

    此更改也**回溯**到：1.1.15

    参考：[#4096](https://www.sqlalchemy.org/trac/ticket/4096)

+   **[mysql] [bug]**

    MariaDB 10.2 现在支持 CHECK 约束（警告：由于[#4097](https://www.sqlalchemy.org/trac/ticket/4097)中指出的上游问题，请��用 10.2.9 或更高版本）。反射现在在`SHOW CREATE TABLE`输出中考虑这些 CHECK 约束。

    此更改也**回溯**到：1.1.15

    参考：[#4098](https://www.sqlalchemy.org/trac/ticket/4098)

+   **[mysql] [bug]**

    将新的 MySQL INSERT..ON DUPLICATE KEY UPDATE 结构的`.values`属性更名为`.inserted`，因为`Insert`已经有一个名为`Insert.values()`的方法。`.inserted`属性最终呈现了 MySQL 的`VALUES()`函数。

    参考：[#4072](https://www.sqlalchemy.org/trac/ticket/4072)

### sqlite

+   **[sqlite] [bug]**

    修复了 SQLite CHECK 约束反射失败的 bug，如果引用的表在远程模式下，例如在 SQLite 中由 ATTACH 引用的远程数据库。

    此更改也**回溯**到：1.1.15

    参考：[#4099](https://www.sqlalchemy.org/trac/ticket/4099)

### mssql

+   **[mssql] [feature]**

    添加了一个新的`TIMESTAMP`数据类型，它在 SQL Server 中正确地像二进制数据类型一样工作，而不是 datetime 类型，因为 SQL Server 在这里违反了 SQL 标准。还添加了`ROWVERSION`，因为 SQL Server 中的`TIMESTAMP`类型已被弃用，改用 ROWVERSION。

    参考：[#4086](https://www.sqlalchemy.org/trac/ticket/4086)

+   **[mssql] [feature]**

    为 PyODBC 和 pymssql 方言添加了对“AUTOCOMMIT”隔离级别的支持，通过`Connection.execution_options()`来建立，这个隔离级别在底层连接对象上设置了适当的 DBAPI 特定标志。

    参考：[#4058](https://www.sqlalchemy.org/trac/ticket/4058)

+   **[mssql] [bug]**

    在 PyODBC 方言中为 SQL Server 添加了一整套“连接关闭”异常代码，包括‘08S01’、‘01002’、‘08003’、‘08007’、‘08S02’、‘08001’、‘HYT00’、‘HY010’。之前只覆盖了‘08S01’。

    此更改也已**回溯**至：1.1.15

    参考：[#4095](https://www.sqlalchemy.org/trac/ticket/4095)

+   **[mssql] [bug]**

    SQL Server 支持 SQLAlchemy 所称的“本地布尔”与其 BIT 类型，因为该类型只接受 0 或 1，而 DBAPI 返回其值为 True/False。因此，SQL Server 方言现在启用了“本地布尔”支持，即 `Boolean` 数据类型不会生成 CHECK 约束。与其他本地布尔的唯一区别在于这里仍然不生成“true” / “false”常量，因此“1”和“0”仍然在这里呈现。

    参考：[#4061](https://www.sqlalchemy.org/trac/ticket/4061)

+   **[mssql] [bug]**

    修复了 pymssql 方言中 SQL 文本中的百分号，例如在模数表达式或文字值中使用的百分号，不会被加倍，这似乎是 pymssql 期望的。尽管 pymssql DBAPI 使用“pyformat”参数样式，它认为百分号本身是重要的。

    参考：[#4057](https://www.sqlalchemy.org/trac/ticket/4057)

+   **[mssql] [bug]**

    修复了 SQL Server 方言在反射自引用外键约束时可能从多个模式中提取列的错误，如果多个模式包含相同名称的约束针对相同名称的表。

    参考：[#4060](https://www.sqlalchemy.org/trac/ticket/4060)

+   **[mssql] [bug] [orm]**

    为方言添加了一个新的“行计数支持”类，用于在“RETURNING”时特定于方言的情况下，对于 SQL Server 看起来像“OUTPUT inserted”的情况，因为 PyODBC 后端在 OUTPUT 生效时无法给我们 UPDATE 或 DELETE 语句的行计数。这主要影响 ORM，当刷新正在更新包含服务器计算值的行时，如果后端没有返回预期的行数，将会引发错误。PyODBC 现在声明它支持行数计数，除非存在 OUTPUT.inserted，ORM 在刷新时会考虑到这一点，以确定是否查找行数计数。

    参考：[#4062](https://www.sqlalchemy.org/trac/ticket/4062)

+   **[mssql] [bug] [orm]**

    启用了 pymssql 方言的“sane_rowcount”标志，指示 DBAPI 现在报告了 UPDATE 或 DELETE 语句受影响的正确行数。这主要影响 ORM 版本功能，因为现在它可以验证目标版本上受影响的行数。

+   **[mssql] [bug]**

    添加了一个规则到 SQL Server 索引反射中，忽略所谓的隐式存在于未指定聚集索引的表上的“堆”索引。

    参考：[#4059](https://www.sqlalchemy.org/trac/ticket/4059)

### oracle

+   **[oracle] [performance] [bug] [py2k]**

    修复了由于对 [#3937](https://www.sqlalchemy.org/trac/ticket/3937) 的修复引起的性能回退，其中 cx_Oracle 版本 5.3 删除了其命名空间中的 `.UNICODE` 符号，这被解释为 cx_Oracle 的“WITH_UNICODE”模式被无条件地打开，从而在 SQLAlchemy 侧调用函数时无条件地将所有字符串转换为 unicode 并引起性能影响。实际上，根据 cx_Oracle 的作者，自 5.1 版本以来，“WITH_UNICODE”模式已经完全删除，因此不再需要昂贵的 unicode 转换函数，并且如果在 Python 2 下检测到 cx_Oracle 5.1 或更高版本，则会被禁用。还恢复了在 [#3937](https://www.sqlalchemy.org/trac/ticket/3937) 中删除的针对“WITH_UNICODE”模式的警告。

    此更改也**已回溯**至：1.1.13、1.0.19

    参考：[#4035](https://www.sqlalchemy.org/trac/ticket/4035)

+   **[oracle] [bug]**

    使用 cx_Oracle 部分支持持久化和检索 Oracle 值“infinity”，只使用 Python 浮点值，例如 `float("inf")`。目前 cx_Oracle DBAPI 驱动程序尚未实现对十进制的支持。

    参考：[#4064](https://www.sqlalchemy.org/trac/ticket/4064)

+   **[oracle] [bug]**

    cx_Oracle 方言已经进行了重组和现代化，以利用在旧的 4.x 系列 cx_Oracle 中不存在的新模式。这包括 cx_Oracle 的最低版本是 5.x 系列，而且 cx_Oracle 6.x 现在已经完全测试过了。最重要的变化涉及类型转换，主要是关于数值 / 浮点和 LOB 数据类型，更有效地利用了 cx_Oracle 类型处理钩子，简化了绑定参数和结果数据的处理方式。

    另请参阅

    cx_Oracle 方言，类型系统主要重构

+   **[oracle] [bug]**

    对于所有版本的 cx_Oracle，两阶段支持已完全移除，而在 1.2.0b1 中，这个变化仅对 cx_Oracle 6.x 系列生效。这个特性在任何版本的 cx_Oracle 中从未正常工作过，并且在 cx_Oracle 6.x 中，SQLAlchemy 依赖的 API 已被移除。

    另请参阅

    cx_Oracle 方言，类型系统主要重构

    参考：[#3997](https://www.sqlalchemy.org/trac/ticket/3997)

+   **[oracle] [bug]**

    在使用 cx_Oracle 后端的 `Insert.returning()` 返回结果集时，现在使用正确的列 / 标签名称，就像其他所有方言一样。以前，这些结果会出现为 `ret_nnn`。

    另请参阅

    cx_Oracle 方言，类型系统主要重构

+   **[oracle] [bug]**

    对 cx_Oracle 方言的几个参数现在已经不推荐使用，并且不会产生任何效果：`auto_setinputsizes`、`exclude_setinputsizes`、`allow_twophase`。

    另请参阅

    cx_Oracle 方言、类型系统的重大重构

+   **[oracle] [bug]**

    修复了一个错误，其中在 Oracle 下反映出的索引，例如“column DESC”，如果表也没有主键，则不会返回，这是由于尝试将 Oracle 隐式添加到主键列上的索引的逻辑导致的。

    参考：[#4042](https://www.sqlalchemy.org/trac/ticket/4042)

+   **[oracle] [bug]**

    由于 cx_Oracle 6.0 引起的更多回归已修复；目前，用户的唯一行为变化是断开检测现在除了 cx_Oracle.InterfaceError 外还检测 cx_Oracle.DatabaseError，因为这种行为似乎已经改变。 关于数字精度和无法关闭连接的其他问题仍在上游 cx_Oracle 问题跟踪器中挂起。

    参考：[#4045](https://www.sqlalchemy.org/trac/ticket/4045)

+   **[oracle] [bug]**

    修复了 Oracle 8 的“非 ansi”连接模式不会向使用其他运算符而不是`=`运算符的表达式添加`(+)`运算符的错误。 `(+)`需要出现在右侧的所有列上。

    参考：[#4076](https://www.sqlalchemy.org/trac/ticket/4076)

## 1.2.0b2

发布日期：2017 年 7 月 24 日

### orm

+   **[orm] [bug]**

    修复了从 1.1.11 中添加附加非实体列到包含子查询加载关系的实体的查询失败的回归，由于在 1.1.11 中添加了的检查作为[#4011](https://www.sqlalchemy.org/trac/ticket/4011)的结果。

    此更改也**回溯到**：1.1.12

    参考：[#4033](https://www.sqlalchemy.org/trac/ticket/4033)

+   **[orm] [bug]**

    修复了 1.1 中作为 [#3514](https://www.sqlalchemy.org/trac/ticket/3514) 的一部分添加的 JSON NULL 评估逻辑中的错误，其中逻辑不会适应 ORM 映射的属性与`Column`不同的名称。

    此更改也**回溯到**：1.1.12

    参考：[#4031](https://www.sqlalchemy.org/trac/ticket/4031)

+   **[orm] [bug]**

    在 `WeakInstanceDict` 内的所有方法中添加了 `KeyError` 检查，其中 `key in dict` 的检查后跟着对该键的索引访问，以防止在负载下垃圾收集可能会将该键从字典中移除的情况下，代码假设其存在，导致非常不频繁的 `KeyError` 引发。

    此更改也**回溯到**：1.1.12

    参考：[#4030](https://www.sqlalchemy.org/trac/ticket/4030)

### 测试

+   **[tests] [bug] [py3k]**

    修复了与 Python 3.6.2 作为不兼容的更改的测试夹具中的问题，涉及上下文管理器。

    此更改也**回溯到**：1.1.12, 1.0.18

    参考：[#4034](https://www.sqlalchemy.org/trac/ticket/4034)

## 1.2.0b1

发布日期：2017 年 7 月 10 日

### orm

+   **[orm] [feature]**

    现在可以将一个`aliased()`构造传递给`Query.select_entity_from()`方法。实体将从由`aliased()`构造表示的可选择项中提取。这允许与`Query.select_entity_from()`一起使用`aliased()`的特殊选项，如`aliased.adapt_on_names`。

    此更改也**回溯**到：1.1.7

    参考：[#3933](https://www.sqlalchemy.org/trac/ticket/3933)

+   **[orm] [feature]**

    向`scoped_session`添加了`.autocommit`属性，代理了当前分配给线程的底层`Session`的`.autocommit`属性。感谢 Ben Fagin 的拉取请求。

+   **[orm] [feature]**

    添加了一个新功能`with_expression()`，允许在结果时间将一个临时 SQL 表达式添加到查询中的特定实体。这是 SQL 表达式作为结果元组中的一个单独元素传递的替代方法。

    另请参阅

    可以接收临时 SQL 表达式的 ORM 属性

    参考：[#3058](https://www.sqlalchemy.org/trac/ticket/3058)

+   **[orm] [feature]**

    添加了一种新的映射器级继承加载样式“多态 selectin”。这种加载方式在加载基本对象类型后，为继承层次结构中的每个子类发出查询，使用 IN 来指定所需的主键值。

    另请参阅

    “selectin”多态加载，使用单独的 IN 查询加载子类

    参考：[#3948](https://www.sqlalchemy.org/trac/ticket/3948)

+   **[orm] [feature]**

    添加了一种新的急切加载方式称为“selectin”加载。这种加载方式与“subquery”急切加载非常相似，只是它使用了一个 IN 表达式，给出了加载的父对象的主键值列表，而不是重新陈述原始查询。这产生了一个更有效的查询，是“烘烤的”（例如，SQL 字符串被缓存），并且也适用于`Query.yield_per()`的上下文中。

    另请参阅

    新的“selectin”急切加载，使用 IN 一次加载所有集合

    参考：[#3944](https://www.sqlalchemy.org/trac/ticket/3944)

+   **[orm] [feature]**

    `lazy="select"` 加载策略现在在所有情况下都使用 `BakedQuery` 查询缓存系统。这消除了生成 `Query` 对象并将其运行到 `select()` 中，然后从懒加载相关集合和对象的过程中生成字符串 SQL 语句的大部分开销。 “烘焙”懒加载器也已经改进，现在在大多数情况下可以缓存使用查询加载选项的情况。

    另请参阅

    “烘焙”加载现在是懒加载的默认值

    参考：[#3954](https://www.sqlalchemy.org/trac/ticket/3954)

+   **[orm] [feature] [ext]**

    `Query.update()` 方法现在可以同时适应混合属性和复合属性作为放置在 SET 子句中的键的来源。对于混合属性，用户提供一个返回元组的函数，还提供了额外的装饰器 `hybrid_property.update_expression()`。

    另请参阅

    支持混合属性、复合属性的批量更新

    参考：[#3229](https://www.sqlalchemy.org/trac/ticket/3229)

+   **[orm] [feature]**

    添加了新的属性事件 `AttributeEvents.bulk_replace()`。当将集合分配给关系时，触发此事件，在将传入集合与现有集合进行比较之前。这个早期事件还允许转换传入的非 ORM 对象。该事件与 `@validates` 装饰器集成。

    另请参阅

    新的 bulk_replace 事件

    参考：[#3896](https://www.sqlalchemy.org/trac/ticket/3896)

+   **[orm] [feature]**

    添加了新的事件处理程序 `AttributeEvents.modified()`，当调用 func:.attributes.flag_modified 函数时触发，这在使用 `sqlalchemy.ext.mutable` 扩展模块时很常见。

    另请参阅

    sqlalchemy.ext.mutable 的新“modified”事件处理程序

    参考：[#3303](https://www.sqlalchemy.org/trac/ticket/3303)

+   **[orm] [bug]**

    修复了与子查询急切加载相关的问题，这是从[#2699](https://www.sqlalchemy.org/trac/ticket/2699)、[#3106](https://www.sqlalchemy.org/trac/ticket/3106)、[#3893](https://www.sqlalchemy.org/trac/ticket/3893)修复的一系列问题中继续的。涉及到“子查询”包含正确的 FROM 子句，当从一个连接的继承子类开始，然后对基类的关系进行子查询急切加载时，同时查询还包括针对子类的条件。之前票据中的修复没有考虑到加载更深层次的额外子查询加载操作，因此修复已进一步泛化。

    此更改也被**回溯**到：1.1.11

    参考：[#4011](https://www.sqlalchemy.org/trac/ticket/4011)

+   **[orm] [bug]**

    修复了一个错误，即像“delete-orphan”（以及其他一些）这样的级联操作将无法定位到与继承关系中的子类本地关系相链接的对象，从而导致操作无法执行。

    此更改也被**回溯**到：1.1.10

    参考：[#3986](https://www.sqlalchemy.org/trac/ticket/3986)

+   **[orm] [bug]**

    修复了在多线程环境下可能发生的竞争条件，这是通过[#3915](https://www.sqlalchemy.org/trac/ticket/3915)添加的缓存导致的。内部的`Column`对象集合可能会在别名对象上不当地重新生成，使得连接的急切加载器在尝试渲染 SQL 并收集结果时混淆，并导致属性错误。现在在别名对象被缓存和在线程之间共享之前，集合会提前生成。

    此更改也被**回溯**到：1.1.7

    参考：[#3947](https://www.sqlalchemy.org/trac/ticket/3947)

+   **[orm] [bug]**

    作为`relationship.post_update`功能的结果发出的 UPDATE 现在将与版本控制功能集成，以提升行的版本 ID，并断言现有版本号已匹配。

    另请参阅

    post_update 与 ORM 版本控制集成

    参考：[#3496](https://www.sqlalchemy.org/trac/ticket/3496)

+   **[orm] [bug]**

    修复了几个情况，这些情况涉及到与具有“onupdate”值的列一起使用时的`relationship.post_update`功能。当 UPDATE 发出时，相应的对象属性现在会过期或刷新，以便新生成的“onupdate”值可以填充到对象中；以前的陈旧值将保留。另外，如果 Python 中设置了对象的 INSERT 的目标属性，则该值现在将在 UPDATE 期间重新发送，以便“onupdate”不会覆盖它（请注意，这对于服务器生成的 onupdates 同样有效）。最后，在刷新时，这些属性现在会发出`SessionEvents.refresh_flush()`事件。

    另请参阅

    改进与 onupdate 结合使用的 post_update

    参考：[#3471](https://www.sqlalchemy.org/trac/ticket/3471)，[#3472](https://www.sqlalchemy.org/trac/ticket/3472)

+   **[orm] [bug]**

    修复了一个错误，即在程序化版本 ID 计数器与连接表继承结合使用时，如果版本 ID 计数器实际上未增加，并且基表上没有修改其他值，则 UPDATE 将具有空的 SET 子句时会失败。由于程序化版本 ID 在版本计数器未增加时是一个记录的用例，因此现在会检测到这种特定条件，并且 UPDATE 现在将版本 ID 值设置为自身，以便仍然进行并发检查。

    参考：[#3996](https://www.sqlalchemy.org/trac/ticket/3996)

+   **[orm] [bug]**

    版本功能不支持版本计数器为 NULL。如果版本 ID 是程序化的，并且在 UPDATE 中将其设置为 NULL，则现在会引发异常。拉取请求由 Diana Clarke 提供。

    参考：[#3673](https://www.sqlalchemy.org/trac/ticket/3673)

+   **[orm] [bug]**

    从`scoped_session`中删除了一个非常古老的关键字参数，称为`scope`。这个关键字从未被记录，并且是早期尝试允许变量范围的一种尝试。

    另请参阅

    从 scoped_session 中删除“scope”关键字

    参考：[#3796](https://www.sqlalchemy.org/trac/ticket/3796)

+   **[orm] [bug]**

    修复了一个错误，即将“with_polymorphic”加载与指定了 innerjoin=True 的子类链接关系结合使用时，将无法将这些“innerjoins”降级为“outerjoins”以适应不支持该关系的其他多态类。这适用于单个和连接继承多态加载。

    参考：[#3988](https://www.sqlalchemy.org/trac/ticket/3988)

+   **[orm] [bug]**

    为`Session.refresh()`方法添加了新参数`with_for_update`。当`Query.with_lockmode()`方法被弃用，改用`Query.with_for_update()`时，`Session.refresh()`方法从未更新以反映新选项。

    参见

    为`Session.refresh`添加了“for update”参数

    参考：[#3991](https://www.sqlalchemy.org/trac/ticket/3991)

+   **[orm] [bug]**

    修复了一个 bug，即一个同时标记为“延迟加载”的`column_property()`在刷新时会被标记为“过期”，导致它与常规属性一起加载，即使从未访问过该属性。

    参考：[#3984](https://www.sqlalchemy.org/trac/ticket/3984)

+   **[orm] [bug]**

    修复了子查询急加载中的错误，其中自引用关系的“join_depth”参数不会被正确遵守，而是加载所有可用的深度，而不是正确计算急加载的指定级别数。

    参考：[#3967](https://www.sqlalchemy.org/trac/ticket/3967)

+   **[orm] [bug]**

    在`Mapper`中添加了对 LRU“编译缓存”的警告（最终也会用于其他基于 ORM 的 LRU 缓存），当缓存开始达到大小限制时，应用程序将发出警告，指出这是一个可能需要关注的性能下降情况。LRU 缓存主要会达到大小限制，如果应用程序使用了无限数量的`Engine`对象，这是一个反模式。否则，这可能表明存在一个问题，应该引起 SQLAlchemy 开发人员的注意。

+   **[orm] [bug]**

    修复了一个 bug，以提高在延迟加载相关实体后生效的加载器选项的特异性，使得加载器选项将更具体地匹配到一个别名或非别名实体，如果这些选项包括实体信息。

    参考：[#3963](https://www.sqlalchemy.org/trac/ticket/3963)

+   **[orm] [bug]**

    `flag_modified()` 函数现在在对象中未找到指定属性键时会引发 `InvalidRequestError`，因为在刷新过程中假定该属性键是存在的。要在不涉及任何特定属性的情况下标记对象为“脏”以进行刷新，可以使用 `flag_dirty()` 函数。

    另请参阅

    使用 flag_dirty() 将对象标记为“脏”而不更改任何属性

    参考：[#3753](https://www.sqlalchemy.org/trac/ticket/3753)

+   **[orm] [bug]**

    `Query.update()` 和 `Query.delete()` 使用的“evaluate”策略现在可以在主键/外键列的属性名称与实际列名称不匹配时，从多对一关系到实例进行简单对象比较。以前，这将进行简单的基于名称的匹配，并因 AttributeError 失败。

    参考：[#3366](https://www.sqlalchemy.org/trac/ticket/3366)

+   **[orm] [bug]**

    `@validates` 装饰器现在允许装饰的方法接收尚未与现有集合进行比较的“批量集合设置”操作中的对象。这允许传入值转换为兼容的 ORM 对象，就像从“追加”事件中已经允许的那样。请注意，这意味着在集合分配期间 **所有** 值都会调用 `@validates` 方法，而不仅仅是新值。

    另请参阅

    在比较之前，@validates 方法在批量集合设置上接收所有值

    参考：[#3896](https://www.sqlalchemy.org/trac/ticket/3896)

+   **[orm] [bug]**

    修复了单表继承中 select_from() 参数在限制行到子类时不会被考虑的 bug。以前，只有请求的列中的表达式会被考虑。

    另请参阅

    修复了与 select_from() 一起使用的单表继承的问题

    参考：[#3891](https://www.sqlalchemy.org/trac/ticket/3891)

+   **[orm] [bug]**

    当将集合分配给由关系映射的属性时，以前的集合不再发生变化。以前，旧集合会随着触发的“项删除”事件而清空；现在事件会触发，但不会影响旧集合。

    另请参阅

    替换时不再改变先前的集合

    参考：[#3913](https://www.sqlalchemy.org/trac/ticket/3913)

+   **[orm] [bug]**

    当`SessionEvents.after_rollback()`事件被触发时，`Session`的状态现在是存在的，即对象在过期之前的属性状态。这与`SessionEvents.after_commit()`事件的行为一致，该事件在对象的属性状态过期之前也会被触发。

    另请参阅

    在对象过期之前，after_rollback() Session 事件现在会被触发

    参考：[#3934](https://www.sqlalchemy.org/trac/ticket/3934)

+   **[orm] [错误]**

    修复了一个 bug，`Query.with_parent()`在针对`aliased()`构造而不是常规映射类时无法正常工作。还为独立的`with_parent()`函数以及`Query.with_parent()`添加了一个新参数`with_parent.from_entity`。

    参考：[#3607](https://www.sqlalchemy.org/trac/ticket/3607)

### 声明式 orm

+   **[orm] [声明式] [错误]**

    修复了一个 bug，在`AbstractConcreteBase`上使用`declared_attr`时，如果特定返回值是一些非映射符号，包括`None`，会导致属性只被硬评估一次并将值存储到对象字典中，不允许它为子类调用。当`declared_attr`在映射类上时，这种行为是正常的，在混合类或抽象类上不会发生。由于`AbstractConcreteBase`既是“抽象”的又是实际“映射”的，因此在这里做了一个特殊的例外情况，以便“抽象”行为优先于`declared_attr`。

    参考：[#3848](https://www.sqlalchemy.org/trac/ticket/3848)

### 引擎

+   **[引擎] [特性]**

    在`Pool`对象中添加了本机“悲观断开”处理。新参数`Pool.pre_ping`，可从引擎中作为`create_engine.pool_pre_ping`使用，应用了池文档中特色的“预检测”配方的高效形式，每次连接检出时，发出一个简单的语句，通常是“SELECT 1”，以测试连接的活动性。如果现有连接不再能响应命令，则连接将被透明地回收，并且在当前时间戳之前进行的所有其他连接将被作废。

    另请参阅

    断开处理 - 悲观

    连接池中添加了悲观断开检测

    参考：[#3919](https://www.sqlalchemy.org/trac/ticket/3919)

+   **[engine] [bug]**

    添加了一个异常处理程序，当`Connection`的“autorollback”功能本身引发异常时，将对 Py2K 上的“cause”异常发出警告。在 Py3K 中，两个异常自然地由解释器报告为一个发生在处理另一个时。这是继续处理回滚失败处理的一系列更改的一部分，上次在 1.0.12 中作为[#2696](https://www.sqlalchemy.org/trac/ticket/2696)的一部分访问。

    此更改也**回溯**到：1.1.7

    参考：[#3946](https://www.sqlalchemy.org/trac/ticket/3946)

+   **[engine] [bug]**

    修复了一个错误，即在将`Compiled`对象直接传递给`Connection.execute()`的不寻常情况下，生成`Compiled`对象的方言未被咨询以获取字符串语句的 paramstyle，而是假定它将匹配方言级别的 paramstyle，导致不匹配发生。

    参考：[#3938](https://www.sqlalchemy.org/trac/ticket/3938)

### sql

+   **[sql] [feature]**

    添加了一种名为“expanding”的新类型`bindparam()`。这用于在`IN`表达式中，元素列表在语句执行时被渲染为单独的绑定参数，而不是在语句编译时。这允许将单个绑定参数名称链接到包含多个元素的 IN 表达式，同时还允许在 IN 表达式中使用查询缓存。这一新功能允许“select in”加载和“polymorphic in”加载使用烘焙查询扩展来减少调用开销。这一功能应被视为**实验性**的 1.2 版本。

    另请参阅

    延迟扩展的 IN 参数集允许使用缓存语句的 IN 表达式

    参考：[#3953](https://www.sqlalchemy.org/trac/ticket/3953)

+   **[sql] [feature] [mysql] [oracle] [postgresql]**

    对`Table`和`Column`对象添加了对 SQL 注释的支持，通过新的`Table.comment`和`Column.comment`参数。这些注释将作为表创建时的 DDL 的一部分包含在内，可以通过内联或适当的 ALTER 语句进行反映，并且也会在表反射中反映出来，以及通过`Inspector`进行反映。目前支持的后端包括 MySQL、PostgreSQL 和 Oracle。非常感谢 Frazer McLean 在这方面付出的大量努力。

    另请参阅

    对 Table、Column 支持 SQL 注释，包括 DDL、反射

    参考：[#1546](https://www.sqlalchemy.org/trac/ticket/1546)

+   **[sql] [feature]**

    `ColumnOperators.in_()` 和 `ColumnOperators.notin_()` 操作符的长期行为已经修订，当右侧条件为空序列时，会发出警告；现在默认情况下会呈现一个简单的“静态”表达式“1 != 1”或“1 = 1”，而不是引入原始的左侧表达式。这导致对空集合进行 NULL 列比较的结果从 NULL 更改为 true/false。此行为是可配置的，旧行为可以通过 `create_engine()` 的 `create_engine.empty_in_strategy` 参数启用。

    另请参阅

    IN / NOT IN 运算符的空集合行为现在可配置；默认表达式简化

    参考：[#3907](https://www.sqlalchemy.org/trac/ticket/3907)

+   **[sql] [feature]**

    为比较器的“startswith”和“endswith”类添加了一个新选项`autoescape`；这个选项提供了一个转义字符，并自动将其应用于所有通配符“%”和“_”的出现。感谢 Diana Clarke 的拉取请求。

    注意

    该功能已于 1.2.0 版本中从其在 1.2.0b2 中的初始实现中更改，现在 autoescape 被传递为布尔值，而不是用作转义字符的特定字符。

    另请参阅

    新的“autoescape”选项用于 startswith()，endswith()

    参考：[#2694](https://www.sqlalchemy.org/trac/ticket/2694)

+   **[sql] [bug]**

    修复了在结构迭代期间可能发生的 `WithinGroup` 构造中的 AttributeError。

    此更改也**回溯**到：1.1.11

    参考：[#4012](https://www.sqlalchemy.org/trac/ticket/4012)

+   **[sql] [bug]**

    修复了在 1.1.5 版本中由于 [#3859](https://www.sqlalchemy.org/trac/ticket/3859) 导致的回归问题，其中基于 `Variant` 的“右侧”表达式评估的调整，以遵守底层类型的“右侧”规则，导致在那些我们*确实*希望左侧类型直接传递到右侧，以便将绑定级规则应用于表达式参数时，`Variant` 类型被不适当地丢失。

    此更改也**回溯**到：1.1.9

    参考：[#3952](https://www.sqlalchemy.org/trac/ticket/3952)

+   **[sql] [bug] [postgresql]**

    更改了`ResultProxy`的机制，无条件延迟“autoclose”步骤，直到`Connection`完成对象的使用；在 PostgreSQL ON CONFLICT 与 RETURNING 返回零行的情况下，autoclose 会发生在这种以前不存在的用例中，导致在 INSERT/UPDATE/DELETE 上无条件发生的通常自动提交行为失败。

    此更改也**回溯**到：1.1.9

    参考：[#3955](https://www.sqlalchemy.org/trac/ticket/3955)

+   **[sql] [bug]**

    现在，`Numeric`、`Integer`和与日期相关的类型之间的类型强制转换规则现在包括额外的逻辑，将尝试保留“resolved”类型的传入类型的设置。目前，这个目标是`asdecimal`标志，因此在`Numeric`或`Float`与`Integer`之间的数学操作将保留`asdecimal`标志，以及类型是否应该是`Float`子类。

    另请参阅

    对“float”数据类型进行更强的类型转换

    参考：[#4018](https://www.sqlalchemy.org/trac/ticket/4018)

+   **[sql] [bug] [mysql]**

    现在，`Float`类型的结果处理器现在无条件地通过`float()`处理器运行值，如果方言指定它也支持“本地十进制”模式。虽然大多数后端将为浮点数据类型提供 Python `float`对象，但 MySQL 后端在某些情况下缺乏类型信息以提供此功能，并且除非进行浮点转换，否则将返回`Decimal`。

    另请参阅

    对“float”数据类型进行更强的类型转换

    参考：[#4020](https://www.sqlalchemy.org/trac/ticket/4020)

+   **[sql] [bug]**

    对于传递给 SQL 语句的 Python“float”值，增加了一些额外的严格性。现在，“float”值将与`Float`数据类型相关联，而不是以前的 Decimal 强制转换`Numeric`数据类型，消除了在 SQLite 上发出的令人困惑的警告以及对 Decimal 的不必要强制转换。

    另请参阅

    对“float”数据类型进行更强的类型转换

    参考：[#4017](https://www.sqlalchemy.org/trac/ticket/4017)

+   **[sql] [bug]**

    所有比较运算符（如 LIKE、IS、IN、MATCH、等于、大于、小于等）的操作优先级已经合并为一个级别，因此对这些运算符进行相互比较的表达式将在它们之间产生括号。这适用于像 Oracle、MySQL 等数据库的操作符优先级，这些数据库将所有这些运算符视为相等优先级，以及 PostgreSQL 9.5 版本之后也已经扁平化了其操作符优先级。

    另请参阅

    比较运算符的扁平化操作优先级

    参考：[#3999](https://www.sqlalchemy.org/trac/ticket/3999)

+   **[sql] [bug]**

    修复了使用`ColumnOperators.is_()`或类似操作符的表达式类型不会是“boolean”类型的问题，而是类型将是“nulltype”，以及当使用自定义比较运算符对未类型化表达式进行比较时。这种类型化可能会影响表达式在更大上下文中以及在结果行处理中的行为。

    参考：[#3873](https://www.sqlalchemy.org/trac/ticket/3873)

+   **[sql] [bug]**

    修复了对`Label`构造的否定，使得当应用`not_()`修饰符到标记表达式时，内部元素被正确否定。

    参考：[#3969](https://www.sqlalchemy.org/trac/ticket/3969)

+   **[sql] [bug]**

    SQL 语句中百分号“双倍”用于转义目的的系统已经得到改进。与`literal_column`构造以及`ColumnOperators.contains()`等操作符相关的百分号“双倍”现在基于正在使用的 DBAPI 的声明的参数样式而发生；对于像 PostgreSQL 和 MySQL 驱动程序常见的百分号敏感参数样式，将会发生双倍，对于 SQLite 等其他参数样式则不会。这使得更多数据库通用的使用`literal_column`构造成为可能。

    另请参阅

    literal_column()中的百分号现在有条件转义

    参考：[#3740](https://www.sqlalchemy.org/trac/ticket/3740)

+   **[sql] [bug]**

    修复了一个 bug，在这个 bug 中，列级`CheckConstraint`在编译使用底层方言编译器的 SQL 表达式时会失败，并且无法应用正确的标志以生成内联的字面值，如果 sqltext 是一个 Core 表达式而不仅仅是一个普通字符串。这在 0.9 版本中作为[#2742](https://www.sqlalchemy.org/trac/ticket/2742)的一部分长时间以来已经修复，更常见的是在表级别的检查约束中使用 Core SQL 表达式而不是普通字符串表达式。

    参考：[#3957](https://www.sqlalchemy.org/trac/ticket/3957)

+   **[sql] [bug]**

    修复了一个 bug，在这个 bug 中，SQL 导向的 Python 端列默认值在“pre-execute”代码路径中可能无法在 INSERT 时正确执行，如果 SQL 本身是一个未类型化的表达式，比如纯文本。然而，“pre-execute”代码路径相当不常见，但当不使用 RETURNING 时，它可以应用于具有 SQL 默认值的非整数主键列。

    参考：[#3923](https://www.sqlalchemy.org/trac/ticket/3923)

+   **[sql] [bug]**

    通过列级`collate()`和`ColumnOperators.collate()`渲染的用于 COLLATE 的表达式现在在名称区分大小写时被引用为标识符。请注意，这不会影响类型级别的排序规则，因为它已经被引用。

    另请参见

    列级 COLLATE 关键字现在引用排序规则名称

    参考：[#3785](https://www.sqlalchemy.org/trac/ticket/3785)

+   **[sql] [bug]**

    修复了在列上下文中使用`Alias`对象会在尝试将自身分组到括号表达式中时引发参数错误的 bug。目前还不完全支持以这种方式使用`Alias`的 API，但它适用于一些最终用户的示例，并且可能在支持某些未来 PostgreSQL 功能时发挥更重要的作用。

    参考：[#3939](https://www.sqlalchemy.org/trac/ticket/3939)

### 模式

+   **[schema] [bug]**

    如果创建一个具有不匹配的“本地”和“远程”列数量的`ForeignKeyConstraint`对象，则现在会引发一个`ArgumentError`，否则会导致约束的内部状态不正确。请注意，这也会影响方言的反射过程产生不匹配的列集合用于外键约束的情况。

    此更改也被**回溯**到：1.1.10

    参考：[#3949](https://www.sqlalchemy.org/trac/ticket/3949)

### postgresql

+   **[postgresql] [bug]**

    继续修复正确处理 PostgreSQL 版本字符串“10devel”的问题，该问题在 1.1.8 中发布，另外增加了一个正则表达式版本以处理形式为“10beta1”的版本字符串。虽然 PostgreSQL 现在提供了更好的获取此信息的方法，但我们至少在 1.1.x 中仍然坚持使用正则表达式，以减少与旧版或替代 PostgreSQL 数据库的兼容性风险。

    此更改也被**回溯**到：1.1.11

    参考：[#4005](https://www.sqlalchemy.org/trac/ticket/4005)

+   **[postgresql] [bug]**

    修复了使用带有排序规则的字符串类型的 `ARRAY` 会在 CREATE TABLE 中产生正确语法的错误。

    此更改也被**回溯**到：1.1.11

    参考：[#4006](https://www.sqlalchemy.org/trac/ticket/4006)

+   **[postgresql] [bug]**

    为 GRANT、REVOKE 关键字增加了“autocommit”支持。感谢 Jacob Hayes 提交的拉取请求。

    此更改也被**回溯**到：1.1.10

+   **[postgresql] [bug]**

    增加了解析 PostgreSQL 版本字符串的支持，例如“PostgreSQL 10devel”。感谢 Sean McCully 提交的拉取请求。

    此更改也被**回溯**到：1.1.8

+   **[postgresql] [bug]**

    修复了基本 `ARRAY` 数据类型不会调用 `ARRAY` 的绑定/结果处理器的错误。

    参考：[#3964](https://www.sqlalchemy.org/trac/ticket/3964)

+   **[postgresql] [bug]**

    在反射 PostgreSQL `INTERVAL` 数据类型时，现在支持所有可能的“fields”标识符，例如“YEAR”、“MONTH”、“DAY TO MINUTE”等。此外，`INTERVAL` 数据类型本身现在包括一个新参数 `INTERVAL.fields`，可以在其中指定这些限定符；在反射/检查时，限定符也会反映到生成的数据类型中。

    另请参阅

    支持 INTERVAL 中的字段规范，包括完整反射

    参考：[#3959](https://www.sqlalchemy.org/trac/ticket/3959)

### mysql

+   **[mysql] [feature]**

    增加了对 MySQL 的 ON DUPLICATE KEY UPDATE MySQL 特定 `Insert` 对象的支持。感谢 Michael Doronin 提交的拉取请求。

    另请参阅

    支持 INSERT..ON DUPLICATE KEY UPDATE

    参考：[#4009](https://www.sqlalchemy.org/trac/ticket/4009)

+   **[mysql] [bug]**

    MySQL 5.7 引入了对“SHOW VARIABLES”命令的权限限制；MySQL 方言现在将处理 SHOW 返回没有行的情况，特别是对于 SQL_MODE 的初始获取，并发出警告，指出用户权限应该被修改以允许行存在。

    这个更改也**回溯**到：1.1.11

    参考：[#4007](https://www.sqlalchemy.org/trac/ticket/4007)

+   **[mysql] [bug]**

    移除了对 UTC_TIMESTAMP MySQL 函数的古老且不必要的拦截，这个拦截影响了使用带参数的情况。

    这个更改也**回溯**到：1.1.10

    参考：[#3966](https://www.sqlalchemy.org/trac/ticket/3966)

+   **[mysql] [bug]**

    修复了 MySQL 方言中关于在渲染 CREATE TABLE 时渲染表选项与 PARTITION 选项同时出现时表选项的渲染问题。PARTITION 相关的选项需要跟在表选项后面，而以前没有强制执行这个顺序。

    这个更改也**回溯**到：1.1.10

    参考：[#3961](https://www.sqlalchemy.org/trac/ticket/3961)

+   **[mysql] [bug]**

    添加了对由于陈旧的表定义而无法反射的视图的支持，在调用`MetaData.reflect()`时，对于无法响应`DESCRIBE`的表，会发出警告，但操作会成功。

    参考：[#3871](https://www.sqlalchemy.org/trac/ticket/3871)

### mssql

+   **[mssql] [bug]**

    修复了在使用 Azure 数据仓库时必须从不同视图中获取 SQL Server 事务隔离的错误，现在会尝试针对两个视图执行查询，然后如果持续失败将无条件地引发一个 NotImplemented，以提供对未来新的 SQL Server 版本中任意 API 更改的最佳恢复能力。

    这个更改也**回溯**到：1.1.11

    参考：[#3994](https://www.sqlalchemy.org/trac/ticket/3994)

+   **[mssql] [bug]**

    添加了一个占位符类型`XML`到 SQL Server 方言中，以便包含此类型的反射表可以重新渲染为 CREATE TABLE。该类型没有特殊的往返行为，也不支持额外的限定参数。

    这个更改也**回溯**到：1.1.11

    参考：[#3973](https://www.sqlalchemy.org/trac/ticket/3973)

+   **[mssql] [bug]**

    现在，SQL Server 方言允许在字符串中明确使用括号来表示带有点的数据库和/或所有者名称，围绕所有者和可选的数据库名称。此外，发送`quoted_name`构造用于模式名称将不会在点上拆分，并将提供完整字符串作为“所有者”。`quoted_name`现在也可以从`sqlalchemy.sql`导入空间中使用。

    另请参阅

    支持带有嵌入点的 SQL Server 模式名称

    参考：[#2626](https://www.sqlalchemy.org/trac/ticket/2626)

### oracle

+   **[oracle] [feature] [postgresql]**

    向`Sequence`添加了新关键字`Sequence.cache`和`Sequence.order`，以允许渲染 Oracle 和 PostgreSQL 理解的 CACHE 参数以及 Oracle 理解的 ORDER 参数。拉取请求由 David Moore 提供。

    此更改也**回溯**到：1.1.12

+   **[oracle] [feature]**

    当使用`Inspector.get_unique_constraints()`、`Inspector.get_check_constraints()`时，Oracle 方言现在会检查唯一约束和检查约束。由于 Oracle 没有与唯一`Index`分开的唯一约束，因此反射的`Table`仍将继续不具有与之关联的`UniqueConstraint`对象。拉取请求由 Eloy Felix 提供。

    另请参阅

    Oracle 唯一约束，检查约束现在反映

    参考：[#4003](https://www.sqlalchemy.org/trac/ticket/4003)

+   **[oracle] [bug]**

    当使用版本为 6.0b1 或更高版本的 DBAPI 时，cx_Oracle 完全删除了对两阶段事务的支持。在任何情况下，cx_Oracle 5.x 历史上从未能够使用两阶段功能，而 cx_Oracle 6.x 已经删除了此功能所依赖的连接级“twophase”标志。

    此更改也**回溯**到：1.1.11

    参考：[#3997](https://www.sqlalchemy.org/trac/ticket/3997)

+   **[oracle] [bug]**

    修复了 cx_Oracle 方言中版本字符串解析失败的 bug，因为 cx_Oracle 版本 6.0b1 中的“b”字符。现在版本字符串解析使用正则表达式而不是简单的分割。

    此更改也**回溯到**：1.1.10

    参考：[#3975](https://www.sqlalchemy.org/trac/ticket/3975)

+   **[oracle] [错误]**

    cx_Oracle 方言现在支持“合理的多行数”，即，当通过 DBAPI `cursor.executemany()` 执行一系列参数集时，我们可以利用 `cursor.rowcount` 来验证匹配的行数。这在 ORM 中检测并发修改场景时产生影响，即使 ORM 正在批量处理语句时，也可以检测到一些简单条件，以及在使用更严格的版本功能时，ORM 仍然可以使用语句批处理。该标志针对 cx_Oracle 默认为至少版本 5.0，这在现在已经很普遍。

    参考：[#3932](https://www.sqlalchemy.org/trac/ticket/3932)

+   **[oracle] [错误]** 

    Oracle 反射现在“标准化”了外键约束的名称，即，返回不区分大小写的名称。这已经是对索引和主键约束以及所有表和列名称的行为。当 Alembic 自动生成脚本比较和渲染外键约束名称时，初始指定为不区分大小写时，这将允许正确地进行比较。

    另请参阅

    Oracle 外键约束名称现在“名字标准化”了

    参考：[#3276](https://www.sqlalchemy.org/trac/ticket/3276)

### 杂项

+   **[特性] [扩展]**

    添加了新标志`Session.enable_baked_queries`到`Session`，允许全局禁用烘焙查询，从而减少内存使用。还添加了新的`Bakery`包装器，以便检查由`BakedQuery.bakery`返回的烘焙。

+   **[错误] [扩展]**

    在自动映射准备()操作同时进行的情况下，保护了测试“None”作为一个类的情况，非常少数情况下可能会在垃圾回收时清除声明性类，并且新的 automap 准备()操作正在进行。

    此更改也**回溯到**：1.1.10

    参考：[#3980](https://www.sqlalchemy.org/trac/ticket/3980)

+   **[错误] [扩展]**

    修复了 `sqlalchemy.ext.mutable` 中的错误，在使用 `TypeEngine.copy()` 复制类型后，`Mutable.as_mutable()` 方法将不会跟踪已复制的类型。这在 1.1 版本相对于 1.0 变得更加退化，因为 `TypeDecorator` 类现在是 `SchemaEventTarget` 的子类，其中之一的功能是指示父 `Column` 在 `Column` 被复制时应该被复制。在使用具有混合类或抽象类的声明时，这些复制是常见的。

    此更改也被**回溯**到：1.1.8

    参考：[#3950](https://www.sqlalchemy.org/trac/ticket/3950)

+   **[错误] [扩展]**

    添加了对绑定参数的支持，例如通常通过 `Query.params()` 设置的参数，到 `Result.count()` 方法。之前省略了对参数的支持。感谢 Pat Deegan 提供的拉取请求。

    此更改也被**回溯**到：1.1.8

+   **[错误] [扩展]**

    现在 `AssociationProxy.any()`, `AssociationProxy.has()` 和 `AssociationProxy.contains()` 比较方法支持链接到一个属性，该属性本身也是一个`AssociationProxy`，递归地。

    另请参阅

    关联代理 any(), has(), contains() 可与链式关联代理一起使用

    参考：[#3769](https://www.sqlalchemy.org/trac/ticket/3769)

+   **[错误] [扩展]**

    实现了原地变异操作符 `__ior__`, `__iand__`, `__ixor__` 和 `__isub__`，用于 `MutableSet`，以及 `__iadd__` 用于 `MutableList`，这样当使用这些变异方法修改集合时会触发变更事件。

    另请参阅

    原地变异操作符对 MutableSet, MutableList 有效

    参考：[#3853](https://www.sqlalchemy.org/trac/ticket/3853)

+   **[错误] [声明]**

    如果使用`declared_attr.cascading`修饰符与一个声明属性，而该属性本身是在要映射的类上声明的，而不是在混合类或`__abstract__`类上声明，则会发出警告。`declared_attr.cascading`修饰符目前仅适用于混合/抽象类。

    参考：[#3847](https://www.sqlalchemy.org/trac/ticket/3847)

+   **[bug] [ext]**

    改进了关联代理列表集合，以防止针对新创建的关联对象进行过早的自动刷新，如果使用`list.append()`，并且当关联代理访问端点集合时会调用惰性加载。现在首先访问端点集合，然后调用创建者以生成关联对象。

    参考：[#3941](https://www.sqlalchemy.org/trac/ticket/3941)

+   **[bug] [ext]**

    `sqlalchemy.ext.hybrid.hybrid_property`类现在支持在子类中多次调用诸如`@setter`、`@expression`等的 mutators，并且现在提供了一个`@getter` mutator，以便特定的混合可以在子类或其他类中重新使用。这现在与标准 Python 中的`@property`的行为相匹配。

    另请参阅

    混合属性支持在子类之间重用，重新定义@getter

    参考：[#3911](https://www.sqlalchemy.org/trac/ticket/3911), [#3912](https://www.sqlalchemy.org/trac/ticket/3912)

+   **[bug] [ext]**

    修复了`sqlalchemy.ext.serializer`扩展中的一个 bug，即“注释”SQL 元素（由 ORM 为许多类型的 SQL 表达式生成）无法可靠地序列化。还将序列化器的默认 pickle 级别提升到“HIGHEST_PROTOCOL”。

    参考：[#3918](https://www.sqlalchemy.org/trac/ticket/3918)

## 1.2.19

发布日期：2019 年 4 月 15 日

### orm

+   **[orm] [bug]**

    由于引入了用于关系惰性加载器的烘焙查询，导致 1.2 版本中的一个回归，生成“惰性子句”时会出现竞争条件，该条件发生在一个被记忆的属性内。如果两个线程同时初始化被记忆的属性，则烘焙查询可能会生成带有绑定参数键的查询，然后在下一次运行时用新键替换，导致惰性加载查询将相关条件指定为`None`。修复确保在生成新子句和参数对象之前固定参数名称，以便每次名称都相同。

    参考：[#4507](https://www.sqlalchemy.org/trac/ticket/4507)

### 示例

+   **[示例] [bug]**

    修复了大结果集示例中的一个 bug，由于代码重排导致“id”变量被重新命名，导致测试失败。感谢 Matt Schuchhardt 提供的拉取请求。

    参考：[#4528](https://www.sqlalchemy.org/trac/ticket/4528)

### engine

+   **[engine] [bug]**

    使用`__eq__()`比较两个`URL`对象时没有考虑端口号，只有端口号不同的两个对象被认为是相等的。现在在`URL`的`__eq__()`方法中添加了端口比较，端口号不同的对象现在不相等。此外，在 Python2 中使用`!=`时，`URL`没有实现`__ne__()`，这导致了意外的结果，因为在 Python2 中比较运算符之间没有暗示的关系。

    参考：[#4406](https://www.sqlalchemy.org/trac/ticket/4406)

### mssql

+   **[mssql] [bug]**

    在将隔离级别更改为 SNAPSHOT 后会发出一个 commit()，因为 pyodbc 和 pymssql 都会打开一个隐式事务，这会阻止当前事务中的后续 SQL 被发出。

    参考：[#4536](https://www.sqlalchemy.org/trac/ticket/4536)

### oracle

+   **[oracle] [bug]**

    为 Oracle 方言添加了对`NCHAR`数据类型的反射支持，并将`NCHAR`添加到 Oracle 方言导出的类型列表中。

    参考：[#4506](https://www.sqlalchemy.org/trac/ticket/4506)

### orm

+   **[orm] [bug]**

    由于为关系惰性加载器引入了烘焙查询，导致 1.2 中出现了一个回归问题，其中在生成“惰性子句”期间创建了一个竞争条件，该条件发生在一个被备忘的属性内。如果两个线程同时初始化备忘属性，则烘焙查询可能会生成带有绑定参数键的查询，然后在下一次运行时用新键替换，导致惰性加载查询将相关条件指定为`None`。修复确保在生成新子句和参数对象之前固定参数名称，以便每次名称都相同。

    参考：[#4507](https://www.sqlalchemy.org/trac/ticket/4507)

### 示例

+   **[examples] [bug]**

    修复了大结果集示例中的一个 bug，由于代码重排导致“id”变量被重新命名，导致测试失败。感谢 Matt Schuchhardt 提供的拉取请求。

    参考：[#4528](https://www.sqlalchemy.org/trac/ticket/4528)

### engine

+   **[engine] [bug]**

    使用 `__eq__()` 比较两个`URL`对象时没有考虑端口号，仅通过端口号不同的两个对象被视为相等。现在，在`URL`的 `__eq__()` 方法中添加了端口比较，端口号不同的对象现在不相等。另外，`URL`中也没有实现 `__ne__()`，这导致在 Python2 中使用 `!=` 时出现意外结果，因为在 Python2 中比较运算符之间没有暗含的关系。

    引用：[#4406](https://www.sqlalchemy.org/trac/ticket/4406)

### mssql

+   **[mssql] [bug]**

    在将隔离级别更改为 SNAPSHOT 时发出 commit()，因为 pyodbc 和 pymssql 都会打开一个隐式事务，这会阻止当前事务中发出后续的 SQL。

    引用：[#4536](https://www.sqlalchemy.org/trac/ticket/4536)

### oracle

+   **[oracle] [bug]**

    增加了对 Oracle 方言的`NCHAR`数据类型的反射支持，并将`NCHAR`添加到 Oracle 方言导出的类型列表中。

    引用：[#4506](https://www.sqlalchemy.org/trac/ticket/4506)

## 1.2.18

发布日期：2019 年 2 月 15 日

### orm

+   **[orm] [bug]**

    修复了 1.2 版本中的一个回归问题，其中通配符/load_only 加载器选项对使用 of_type() 限制到特定子类的加载器路径不起作用。修复目前仅适用于简单子类的 of_type()，而不是使用多态实体的 with_polymorphic，后者将在单独的问题中处理；以前很可能不起作用。

    引用：[#4468](https://www.sqlalchemy.org/trac/ticket/4468)

+   **[orm] [bug]**

    修复了一个相当简单但关键的问题，即当`SessionEvents.pending_to_persistent()`事件不仅在对象从挂起转为持久时触发时，而且在对象已经是持久的并且仅在更新时触发时，导致该事件被触发用于每次更新的所有对象。

    引用：[#4489](https://www.sqlalchemy.org/trac/ticket/4489)

### sql

+   **[sql] [bug]**

    修复了`JSON`类型具有只读的`JSON.should_evaluate_none`属性的问题，当与此类型一起使用`TypeEngine.evaluates_none()`方法时会导致失败。拉取请求感谢 Sanjana S。

    参考：[#4485](https://www.sqlalchemy.org/trac/ticket/4485)

### mysql

+   **[mysql] [bug]**

    修复了由[#4344](https://www.sqlalchemy.org/trac/ticket/4344)引起的第二个回归问题（第一个是[#4361](https://www.sqlalchemy.org/trac/ticket/4361)），该问题解决了 MySQL 问题 88718，其中使用的小写函数对于 Python 2 与 OSX/Windows 大小写约定不正确，然后会引发`TypeError`。现在已经为此逻辑添加了完整的覆盖，以便以模拟样式执行所有三种大小写约定在所有 Python 版本上的所有代码路径。与此同时，MySQL 8.0 已经修复了问题 88718，因此这个解决方法仅适用于特定范围的 MySQL 8.0 版本。

    参考：[#4492](https://www.sqlalchemy.org/trac/ticket/4492)

### sqlite

+   **[sqlite] [bug]**

    修复了在 SQLite DDL 中的一个 bug，其中在服务器端默认值使用表达式时需要将其包含在括号中才能被 SQLite 解析器接受。感谢 Bartlomiej Biernacki 提供的拉取请求。

    参考：[#4474](https://www.sqlalchemy.org/trac/ticket/4474)

### mssql

+   **[mssql] [bug]**

    修复了一个 bug，即 SQL Server 的“IDENTITY_INSERT”逻辑允许在 IDENTITY 列上使用显式值进行 INSERT 时未检测到`Insert.values()`与包含`Column`作为键和 SQL 表达式作为值的字典一起使用的情况。

    参考：[#4499](https://www.sqlalchemy.org/trac/ticket/4499)

### orm

+   **[orm] [bug]**

    修复了 1.2 版本中的一个回归问题，其中通配符/load_only 加载器选项对于使用 of_type()限制到特定子类的加载器路径不会正确工作。修复目前仅适用于简单子类的 of_type()，而不适用于将在单独的问题中解决的 with_polymorphic 实体；这后一种情况以前可能不起作用。

    参考：[#4468](https://www.sqlalchemy.org/trac/ticket/4468)

+   **[orm] [bug]**

    修复了一个相当简单但关键的问题，即`SessionEvents.pending_to_persistent()`事件不仅在对象从挂起状态转为持久状态时被调用，而且在对象已经是持久状态并且只是被更新时也会被调用，从而导致事件在每次更新时为所有对象被调用。

    参考：[#4489](https://www.sqlalchemy.org/trac/ticket/4489)

### sql

+   **[sql] [bug]**

    修复了`JSON`类型具有只读`JSON.should_evaluate_none`属性的问题，这会导致在使用`TypeEngine.evaluates_none()`方法与此类型一起使用时出现故障。感谢 Sanjana S 提供的拉取请求。

    参考：[#4485](https://www.sqlalchemy.org/trac/ticket/4485)

### mysql

+   **[mysql] [bug]**

    修复了由[#4344](https://www.sqlalchemy.org/trac/ticket/4344)引起的第二个回归（第一个是[#4361](https://www.sqlalchemy.org/trac/ticket/4361)），该回归解决了 MySQL 问题 88718，其中使用的小写函数对于具有 OSX/Windows 大小写约定的 Python 2 来说不正确，然后会引发`TypeError`。已对此逻辑进行了全面覆盖，以便在所有 Python 版本的所有三种大小写约定上以模拟样式执行每个代码路径。与此同时，MySQL 8.0 已经修复了问题 88718，因此这个解决方法仅适用于特定范围的 MySQL 8.0 版本。

    参考：[#4492](https://www.sqlalchemy.org/trac/ticket/4492)

### sqlite

+   **[sqlite] [bug]**

    修复了 SQLite DDL 中的一个 bug，即在将表达式作为服务器端默认值时，必须将其包含在括号中才能被 sqlite 解析器接受。感谢 Bartlomiej Biernacki 提供的拉取请求。

    参考：[#4474](https://www.sqlalchemy.org/trac/ticket/4474)

### mssql

+   **[mssql] [bug]**

    修复了一个 bug，即 SQL Server 的“IDENTITY_INSERT”逻辑允许在 IDENTITY 列上使用显式值进行 INSERT 时，未检测到使用字典的情况，该字典将`Column`作为键，SQL 表达式作为值。

    参考：[#4499](https://www.sqlalchemy.org/trac/ticket/4499)

## 1.2.17

发布日期：2019 年 1 月 25 日

### orm

+   **[orm] [功能]**

    添加了新的事件钩子`QueryEvents.before_compile_update()`和`QueryEvents.before_compile_delete()`，这些事件钩子在`Query.update()`和`Query.delete()`方法的情况下补充了`QueryEvents.before_compile()`。

    参考：[#4461](https://www.sqlalchemy.org/trac/ticket/4461)

+   **[orm] [bug]**

    修复了在使用单表继承与使用“with polymorphic”加载的连接继承层次结构时，该单表实体的“单表条件”可能会与在同一查询中使用的同一层次结构的其他实体的条件混淆的问题。对“单表条件”的调整更具体地针对目标实体，以避免它意外地适应查询中的其他表。

    参考：[#4454](https://www.sqlalchemy.org/trac/ticket/4454)

### **postgresql**

+   **[postgresql] [bug]**

    修订了在反射 CHECK 约束时使用的查询，以利用`pg_get_constraintdef`函数，因为`consrc`列在 PG 12 中已被弃用。感谢 John A Stevenson 的提示。

    参考：[#4463](https://www.sqlalchemy.org/trac/ticket/4463)

### **oracle**

+   **[oracle] [bug]**

    由于在 1.2 版本中重构了 cx_Oracle 方言，修复了整数精度逻辑的回归。我们现在不再将 cx_Oracle.NATIVE_INT 类型应用于发送整数值的结果列（检测为具有正精度和 scale == 0 的值），这会导致超出 32 位边界的值发生整数溢出问题。相反，输出变量保持未命名，以便 cx_Oracle 可以选择最佳选项。

    参考：[#4457](https://www.sqlalchemy.org/trac/ticket/4457)

### **orm**

+   **[orm] [feature]**

    添加了新的事件钩子`QueryEvents.before_compile_update()`和`QueryEvents.before_compile_delete()`，这与`Query.update()`和`Query.delete()`方法相辅相成。

    参考：[#4461](https://www.sqlalchemy.org/trac/ticket/4461)

+   **[orm] [bug]**

    修复了在使用单表继承与使用“with polymorphic”加载的连接继承层次结构时，该单表实体的“单表条件”可能会与在同一查询中使用的同一层次结构的其他实体的条件混淆的问题。对“单表条件”的调整更具体地针对目标实体，以避免它意外地适应查询中的其他表。

    参考：[#4454](https://www.sqlalchemy.org/trac/ticket/4454)

### **postgresql**

+   **[postgresql] [bug]**

    对于反射 CHECK 约束时使用的查询进行了修订，利用了 `pg_get_constraintdef` 函数，因为 `consrc` 列在 PG 12 中被弃用。感谢 John A Stevenson 的建议。

    参考：[#4463](https://www.sqlalchemy.org/trac/ticket/4463)

### oracle

+   **[oracle] [bug]**

    由于在 1.2 版中重构了 cx_Oracle 方言，导致整数精度逻辑出现回归。我们现在不再将 cx_Oracle.NATIVE_INT 类型应用于发送整数值的结果列（检测为具有正精度和 scale == 0 的值），因为这会导致超出 32 位边界的值发生整数溢出问题。相反，输出变量保持未命名，以便 cx_Oracle 可以选择最佳选项。

    参考：[#4457](https://www.sqlalchemy.org/trac/ticket/4457)

## 1.2.16

发布日期：2019 年 1 月 11 日

### engine

+   **[engine] [bug]**

    在版本 1.2 中引入的回归已修复，该版本中对 `SQLAlchemyError` 基本异常类进行了重构，将纯字符串消息不适当地强制转换为 Unicode（在 Python 2k 下，这由 Python 解释器处理，不处理平台编码之外的字符（通常是 ascii）。 `SQLAlchemyError` 类现在在 Py2K 下通过 `__str__()` 传递字节串，这是 Py2K 下异常对象的一般行为，对于 `__unicode__()` 进行安全的 utf-8 强制转换。对于 Py3K，消息通常已经是 Unicode，但如果不是，则再次进行安全的 utf-8 强制转换以备用于 `__str__()` 方法。

    参考：[#4429](https://www.sqlalchemy.org/trac/ticket/4429)

### sql

+   **[sql] [bug] [mysql] [oracle]**

    修复了一个问题，即针对 MySQL 和 Oracle 数据库，将用于即将推出的 Alembic 版本的 `DropTableComment` 发出的 DDL 不正确。

    参考：[#4436](https://www.sqlalchemy.org/trac/ticket/4436)

### postgresql

+   **[postgresql] [bug]**

    修复了一个问题，即在远程模式下存在的 `ENUM` 或自定义域，如果枚举/域的名称或模式的名称需要引用，则在列反射中将无法识别。现在，一个新的解析方案完全解析出带引号或不带引号的标记，包括支持 SQL 转义引号。

    参考：[#4416](https://www.sqlalchemy.org/trac/ticket/4416)

+   **[postgresql] [bug]**

    修复了一个问题，即如果多个由相同的 `MetaData` 对象引用的多个 `ENUM` 对象在不同的模式名称下具有相同的名称，则在创建期间将无法创建。PostgreSQL 方言使用的内部记忆化现在会考虑模式名称，在 DDL 创建序列期间跟踪它是否已在数据库中创建了特定的 `ENUM`。

### sqlite

+   **[sqlite] [bug]**

    基于 SQL 表达式的索引反射现在会跳过，并显示警告，方式与 Postgresql 方言相同，在那里我们目前不支持反映具有其中的 SQL 表达式的索引。以前，会生成具有列为 None 的索引，这会破坏像 Alembic 这样的工具。

    参考资料：[#4431](https://www.sqlalchemy.org/trac/ticket/4431)

### 杂项

+   **[no_tags]**

    在“expanding IN”功能中修复了问题，其中在查询中多次使用相同的绑定参数名称会导致查询中的参数重写过程中出现 KeyError。

    参考资料：[#4394](https://www.sqlalchemy.org/trac/ticket/4394)

### engine

+   **[engine] [bug]**

    修复了 1.2 版本中引入的退化问题，其中 `SQLAlchemyError` 基本异常类的重构引入了对纯字符串消息的不适当强制转换为 Unicode 在 python 2k 下，在平台的编码之外（通常是 ascii）不由 Python 解释器处理的字符。`SQLAlchemyError` 类现在在 Py2K 下将字节串通过 `__str__()` 传递，这是 Py2K 下异常对象的一般行为，对于 `__unicode__()` 进行了安全的 utf-8 强制转换并回退到反斜杠。对于 Py3K，消息通常已经是 unicode，但如果不是，则再次进行 utf-8 的安全强制转换，并为 `__str__()` 方法进行反斜杠回退。

    参考资料：[#4429](https://www.sqlalchemy.org/trac/ticket/4429)

### sql

+   **[sql] [bug] [mysql] [oracle]**

    修复了为 MySQL 和 Oracle 数据库准备的将用于即将推出的版本的 Alembic 的 `DropTableComment` 发出的 DDL 错误的问题。

    参考资料：[#4436](https://www.sqlalchemy.org/trac/ticket/4436)

### postgresql

+   **[postgresql] [bug]**

    修复了在列反射中无法识别远程模式中存在的`ENUM`或自定义域的问题，如果枚举/域的名称或模式的名称需要引号。现在的新解析方案完全解析出带引号或不带引号的标记，包括对 SQL 转义引号的支持。

    参考：[#4416](https://www.sqlalchemy.org/trac/ticket/4416)

+   **[postgresql] [bug]**

    修复了多个由相同`MetaData`对象引用的`ENUM`对象在具有不同模式名称的相同名称的多个对象时无法创建的问题。PostgreSQL 方言在 DDL 创建序列期间用于跟踪是否在数据库中创建了特定`ENUM`的内部记忆现在考虑模式名称。

### sqlite

+   **[sqlite] [bug]**

    基于 SQL 表达式的索引的反射现在会跳过并发出警告，与 Postgresql 方言的方式相同，我们目前不支持反射具有 SQL 表达式的索引。以前，会生成具有 None 列的索引，这会破坏像 Alembic 这样的工具。

    参考：[#4431](https://www.sqlalchemy.org/trac/ticket/4431)

### 杂项

+   **[no_tags]**

    修复了“扩展 IN”功能中在查询中多次使用相同绑定参数名称会导致在重写查询中的参数时出现 KeyError 的问题。

    参考：[#4394](https://www.sqlalchemy.org/trac/ticket/4394)

## 1.2.15

发布日期：2018 年 12 月 11 日

### orm

+   **[orm] [bug]**

    修复了在声明性映射中使用模式`ForeignKey(SomeClass.id)`时，ORM 注释可能不正确的 bug，这种模式会将不需要的注释泄漏到连接条件中，这可能会破坏`Query`中进行的不应影响该连接条件中的元素的别名操作。如果存在这些注释，现在会提前将其删除。

    参考：[#4367](https://www.sqlalchemy.org/trac/ticket/4367)

+   **[orm] [bug]**

    继续与最近的[#4349](https://www.sqlalchemy.org/trac/ticket/4349)类似的主题，修复了`Comparator.any()`和`Comparator.has()`的问题，其中“secondary”可选择性地需要明确作为 FROM 子句的一部分存在于 EXISTS 子查询中，以适应“secondary”是`Join`对象的情况。

    参考：[#4366](https://www.sqlalchemy.org/trac/ticket/4366)

+   **[orm] [bug]**

    修复了由[#4349](https://www.sqlalchemy.org/trac/ticket/4349)引起的回归，其中将“secondary”表添加到动态加载器的 FROM 子句会影响`Query`后续连接到另一个实体的能力。修复将主实体添加为 FROM 列表的第一个元素，因为`Query.join()`希望从那里跳转。版本 1.3 也将对这个问题有一个更全面的解决方案（[#4365](https://www.sqlalchemy.org/trac/ticket/4365)）。

    参考：[#4363](https://www.sqlalchemy.org/trac/ticket/4363)

+   **[orm] [bug]**

    修复了使用`RelationshipProperty.of_type()`链式映射选项的 bug，与仅通过字符串引用属性名称的链式选项一起使用时，无法定位属性的问题。

    参考：[#4400](https://www.sqlalchemy.org/trac/ticket/4400)

### orm declarative

+   **[orm] [declarative] [bug]**

    在将`column()`对象应用于声明类的情况下，会发出警告，因为这似乎是想要一个`Column`对象。

    参考：[#4374](https://www.sqlalchemy.org/trac/ticket/4374)

### 杂项

+   **[no_tags]**

    增加了对 mysqlclient 和 pymysql 接受的`write_timeout`标志在 URL 字符串中传递的支持。

    参考：[#4381](https://www.sqlalchemy.org/trac/ticket/4381)

+   **[no_tags]**

    修复了无法识别表达为数组的 PostgreSQL 域的反射问题。感谢 Jakub Synowiec 的拉取请求。

    参考：[#4377](https://www.sqlalchemy.org/trac/ticket/4377)，[#4380](https://www.sqlalchemy.org/trac/ticket/4380)

### orm

+   **[orm] [bug]**

    修复了一个错误，即如果在声明映射中使用模式`ForeignKey(SomeClass.id)`，则关系的 primaryjoin/secondaryjoin 的 ORM 注释可能不正确。这种模式会将不需要的注释泄漏到加入条件中，这可能会破坏`Query`中进行的别名操作，这些别名操作不应影响该加入条件中的元素。如果存在这些注释，现在会立即删除。

    参考：[#4367](https://www.sqlalchemy.org/trac/ticket/4367)

+   **[orm] [bug]**

    继续与最近的[#4349](https://www.sqlalchemy.org/trac/ticket/4349)类似的主题，修复了`Comparator.any()`和`Comparator.has()`的问题，其中“secondary”可选择性地需要明确作为 FROM 子句的一部分存在于 EXISTS 子查询中，以适应“secondary”是`Join`对象的情况。

    参考：[#4366](https://www.sqlalchemy.org/trac/ticket/4366)

+   **[orm] [bug]**

    由[#4349](https://www.sqlalchemy.org/trac/ticket/4349)引起的回归错误已修复，其中将“secondary”表添加到动态加载器的 FROM 子句会影响`Query`后续加入到另一个实体的能力。修复方法是将主实体作为 FROM 列表的第一个元素，因为`Query.join()`希望从那里跳转。版本 1.3 还将���此问题提供更全面的解决方案（[#4365](https://www.sqlalchemy.org/trac/ticket/4365))。

    参考：[#4363](https://www.sqlalchemy.org/trac/ticket/4363)

+   **[orm] [bug]**

    修复了使用`RelationshipProperty.of_type()`链式映射器选项与仅通过字符串引用属性名称的链式选项进行链接时无法定位属性的错误。

    参考：[#4400](https://www.sqlalchemy.org/trac/ticket/4400)

### orm 声明式

+   **[orm] [declarative] [bug]**

    当将`column()`对象应用于声明类时，会发出警告，因为这似乎是打算将其作为`Column`对象。

    参考：[#4374](https://www.sqlalchemy.org/trac/ticket/4374)

### 杂项

+   **[no_tags]**

    增加了对`write_timeout`标志的支持，该标志被 mysqlclient 和 pymysql 接受并传递到 URL 字符串中。

    参考：[#4381](https://www.sqlalchemy.org/trac/ticket/4381)

+   **[no_tags]**

    修复了一个问题，即将表示为数组的 PostgreSQL 域的反射会失败无法被识别。感谢 Jakub Synowiec 提交的拉取请求。

    参考：[#4377](https://www.sqlalchemy.org/trac/ticket/4377), [#4380](https://www.sqlalchemy.org/trac/ticket/4380)

## 1.2.14

发布日期：2018 年 11 月 10 日

### orm

+   **[orm] [bug]**

    修复了 `Session.bulk_update_mappings()` 中的错误，在其中，备用映射属性名称会导致 UPDATE 语句的主键列包含在 SET 子句中，以及在 WHERE 子句中；虽然通常是无害的，但对于 SQL Server，这可能由于 IDENTITY 列而引发错误。这是与 [#3849](https://www.sqlalchemy.org/trac/ticket/3849) 中修复的同一错误的延续，其中测试不足以捕捉到这个额外的缺陷。

    参考：[#4357](https://www.sqlalchemy.org/trac/ticket/4357)

+   **[orm] [bug]**

    修复了一个小的性能问题，可能会在某些情况下给结果获取增加不必要的开销，这涉及到在查询中同时使用 ORM 列和包含这些相同列的实体。这个问题与在不同方式中引用列时的哈希 / 相等性开销有关。

    参考：[#4347](https://www.sqlalchemy.org/trac/ticket/4347)

### mysql

+   **[mysql] [bug]**

    修复了 1.2.13 中发布的 [#4344](https://www.sqlalchemy.org/trac/ticket/4344) 引起的回归问题，在此版本中，对于 MySQL 8.0 在反射外键引用时处理列名大小写敏感性问题的修复是通过使用 `information_schema.columns` 视图来解决的。这个解决方法在 OSX / `lower_case_table_names=2` 上失败了，这会导致 `information_schema.columns` 与 `SHOW CREATE TABLE` 的大小写不匹配，因此在不区分大小写的 SQL 模式下，现在使用不区分大小写的匹配。

    参考：[#4361](https://www.sqlalchemy.org/trac/ticket/4361)

### orm

+   **[orm] [bug]**

    修复了 `Session.bulk_update_mappings()` 中的错误，在其中，备用映射属性名称会导致 UPDATE 语句的主键列包含在 SET 子句中，以及在 WHERE 子句中；虽然通常是无害的，但对于 SQL Server，这可能由于 IDENTITY 列而引发错误。这是与 [#3849](https://www.sqlalchemy.org/trac/ticket/3849) 中修复的同一错误的延续，其中测试不足以捕捉到这个额外的缺陷。

    参考：[#4357](https://www.sqlalchemy.org/trac/ticket/4357)

+   **[orm] [bug]**

    修复了一个小的性能问题，可能会在某些情况下给结果获取增加不必要的开销，这涉及到在查询中同时使用 ORM 列和包含这些相同列的实体。这个问题与在不同方式中引用列时的哈希 / 相等性开销有关。

    参考：[#4347](https://www.sqlalchemy.org/trac/ticket/4347)

### mysql

+   **[mysql] [bug]**

    修复了 1.2.13 中由[#4344](https://www.sqlalchemy.org/trac/ticket/4344)引起的回归问题，其中针对 MySQL 8.0 在反射外键引用列名称时的大小写敏感性问题的修复是通过使用`information_schema.columns`视图来解决的。这种解决方法在 OSX / `lower_case_table_names=2`上失败，因为`information_schema.columns`与`SHOW CREATE TABLE`的大小写不匹配，因此在不区分大小写的 SQL 模式下现在使用不区分大小写的匹配。

    参考：[#4361](https://www.sqlalchemy.org/trac/ticket/4361)

## 1.2.13

发布日期：2018 年 10 月 31 日

### orm

+   **[orm] [bug]**

    修复了“动态”加载器需要在查询的 FROM 子句中显式设置“secondary”表的 bug，以适应次要表是连接对象的情况，否则仅从其列中提取的查询中不会包含该表。

    参考：[#4349](https://www.sqlalchemy.org/trac/ticket/4349)

### orm declarative

+   **[orm] [declarative] [bug]**

    修复了 1.2.12 版本中由[#4326](https://www.sqlalchemy.org/trac/ticket/4326)引起的回归问题，当与`synonym()`一起在 mixin 中使用`declared_attr`时，无法正确将同义词映射到继承的子类。

    参考：[#4350](https://www.sqlalchemy.org/trac/ticket/4350)

+   **[orm] [declarative] [bug]**

    在使用 use_existing_column 解决列冲突中讨论的列冲突解决技术现在对于`Column`也是主键列的情况下可用。以前，在单继承子类上声明主键列时，会在允许列复制通过之前进行主键列检查。

    参考：[#4352](https://www.sqlalchemy.org/trac/ticket/4352)

### sql

+   **[sql] [feature]**

    重构了`SQLCompiler`以公开类似于`SQLCompiler.order_by_clause()`和`SQLCompiler.limit_clause()`方法的`SQLCompiler.group_by_clause()`方法，可以被方言重写以自定义 GROUP BY 的呈现方式。感谢 Samuel Chou 的拉取请求。

+   **[sql] [bug]**

    修复了`Enum.create_constraint` 标志在 `Enum` 数据类型的副本中未传播的 bug，这影响了声明性混合和抽象基类等用例。

    参考：[#4341](https://www.sqlalchemy.org/trac/ticket/4341)

### postgresql

+   **[postgresql] [bug]**

    添加了对`aggregate_order_by` 函数的支持，以接收多个 ORDER BY 元素，先前只接受单个元素。

    参考：[#4337](https://www.sqlalchemy.org/trac/ticket/4337)

### mysql

+   **[mysql] [bug]**

    将单词 `function` 添加到 MySQL 的保留字列表中，现在是 MySQL 8.0 中的关键字。

    参考：[#4348](https://www.sqlalchemy.org/trac/ticket/4348)

+   **[mysql] [bug]**

    为 MySQL 8.0 系列引入的一个 bug #88718 添加了一个解决方法，其中外键约束的反射未报告引用列的正确大小写敏感性，导致在使用反射约束时出现错误，例如在使用 automap 扩展时。解决方法通过向 information_schema 表发出额外的查询来检索正确的大小写敏感名称。

    参考：[#4344](https://www.sqlalchemy.org/trac/ticket/4344)

### 杂项

+   **[杂项] [bug]**

    修复了实用语言助手内部的一部分错误，该错误将错误类型的参数传递给 Python `__import__` 内置函数作为要导入的模块列表。该问题在核心库中没有产生任何症状，但可能会导致重新定义 `__import__` 内置函数或以其他方式对其进行检测的外部应用程序出现问题。感谢 Joe Urciuoli 提交的拉取请求。

+   **[杂项] [bug] [py3k]**

    修复了由于 Python 3.7 中 Python `collections` 和 `collections.abc` 包组织变化而生成的额外警告。之前的 `collections` 警告在版本 1.2.11 中已修复。感谢 xtreak 提交的拉取请求。

    参考：[#4339](https://www.sqlalchemy.org/trac/ticket/4339)

+   **[bug] [ext]**

    在关联代理扩展的基于列表的关联集合中添加了缺失的`.index()`方法。

### orm

+   **[orm] [bug]**

    修复了“动态”加载器需要显式设置查询的 FROM 子句中的“secondary”表的 bug，以适应次要表是一个联接对象，否则仅从其列中拉入查询会导致问题的情况。

    参考：[#4349](https://www.sqlalchemy.org/trac/ticket/4349)

### orm 声明性

+   **[orm] [declarative] [bug]**

    修复了版本 1.2.12 中由[#4326](https://www.sqlalchemy.org/trac/ticket/4326)引起的回归，使用`declared_attr`与 mixin 结合使用`synonym()`会导致将同义词正确映射到继承子类失败。

    参考：[#4350](https://www.sqlalchemy.org/trac/ticket/4350)

+   **[orm] [declarative] [bug]**

    现在，讨论的列冲突解决技术使用`use_existing_column`解决列冲突对于`Column`也是主键列的功能正常运行。以前，在单继承子类上声明主键列之前，会发生主键列检查，然后才允许列复制通过。

    参考：[#4352](https://www.sqlalchemy.org/trac/ticket/4352)

### sql

+   **[sql] [feature]**

    重构了`SQLCompiler`以公开一个类似于`SQLCompiler.order_by_clause()`和`SQLCompiler.limit_clause()`方法的`SQLCompiler.group_by_clause()`方法，可以被方言重写以自定义 GROUP BY 的呈现方式。感谢 Samuel Chou 的拉取请求。

+   **[sql] [bug]**

    修复了`Enum.create_constraint`标志在`Enum`数据类型的副本中不会传播的错误，这会影响到声明性 mixin 和抽象基类等用例。

    参考：[#4341](https://www.sqlalchemy.org/trac/ticket/4341)

### postgresql

+   **[postgresql] [bug]**

    添加了对`aggregate_order_by`函数接收多个 ORDER BY 元素的支持，以前只接受单个元素。

    参考：[#4337](https://www.sqlalchemy.org/trac/ticket/4337)

### mysql

+   **[mysql] [bug]**

    将`function`一词添加到 MySQL 的保留字列表中，现在在 MySQL 8.0 中是一个关键字。

    参考：[#4348](https://www.sqlalchemy.org/trac/ticket/4348)

+   **[mysql] [bug]**

    添加了针对 MySQL 8.0 系列引入的 bug #88718 的解决方法，其中外键约束的反射未报告所引用列的正确大小写敏感性，导致在使用反射约束时出现错误，例如在使用 automap 扩展时。解决方法通过向 information_schema 表发出额外查询来检索正确的大小写敏感名称。

    参考：[#4344](https://www.sqlalchemy.org/trac/ticket/4344)

### misc

+   **[misc] [bug]**

    修复了实用语言助手内部的一部分问题，该问题将错误类型的参数传递给 Python `__import__`内置函数作为要导入的模块列表。该问题在核心库中没有产生任何症状，但可能会导致重新定义`__import__`内置函数或以其他方式对其进行调试的外部应用程序出现问题。感谢 Joe Urciuoli 提供的拉取请求。

+   **[misc] [bug] [py3k]**

    修复了 Python 3.7 由于 Python `collections`和`collections.abc`包组织结构变化而生成的额外警告。之前的`collections`警告在 1.2.11 版本中已修复。感谢 xtreak 提供的拉取请求。

    参考：[#4339](https://www.sqlalchemy.org/trac/ticket/4339)

+   **[bug] [ext]**

    在关联代理扩展中添加了缺失的`.index()`方法到基于列表的关联集合。

## 版本号：1.2.12

发布日期：2018 年 9 月 19 日

### orm

+   **[orm] [bug]**

    在弱引用清理中添加了一个检查，用于检查`InstanceState`对象是否存在`dict`内置对象，以减少在解释器关闭期间发生这些清理时生成的错误消息。感谢 Romuald Brunet 提供的拉取请求。

+   **[orm] [bug]**

    修复了一个 bug，即在与`Query.join()`以及`Query.select_entity_from()`结合使用`Lateral`构造时，不会将适配器应用于连接的右侧。 “lateral”引入了连接右侧可关联的用例。先前，未考虑适配此子句。请注意，在 1.2 版本中，由`Query.subquery()`引入的可选择项仍未适配，原因是[#4304](https://www.sqlalchemy.org/trac/ticket/4304)；可选择项需要由`select()`函数生成，以成为“lateral”连接的右侧。

    参考：[#4334](https://www.sqlalchemy.org/trac/ticket/4334)

+   **[orm] [bug]**

    修复了 1.2 版本中由 [#3472](https://www.sqlalchemy.org/trac/ticket/3472) 引起的回归问题，即在后续更新操作的上下文中处理“updated_at”风格列时，也会发生在更新后要删除的行上，这意味着具有 Python 端值生成器的列将显示在 UPDATE 之前发出的现在已删除的值（这不是以前的行为），以及 SQL 发出的值生成器将使属性过期，这意味着由于行已被删除且对象已从会话中分离，因此无法访问以前的值。对于最终将被删除的对象，完全跳过了作为 [#3472](https://www.sqlalchemy.org/trac/ticket/3472) 的一部分添加的“postfetch”逻辑。

    参考：[#4327](https://www.sqlalchemy.org/trac/ticket/4327)

### orm declarative

+   **[orm] [declarative] [bug]**

    修复了一个 bug，即在已映射类的子类上通过 `@declared_attr` 可调用获取描述符时，声明性扫描属性会收到混合属性提供的表达式代理，而不是混合属性本身。这会导致在 `Mapper.all_orm_descriptors` 中查看时，属性不会报告自身为混合属性。

    参考：[#4326](https://www.sqlalchemy.org/trac/ticket/4326)

### postgresql

+   **[postgresql] [bug]**

    修复了 PostgreSQL 方言中的一个 bug，即编译器关键字参数（如 `literal_binds=True`）未传播到 DISTINCT ON 表达式。

    参考：[#4325](https://www.sqlalchemy.org/trac/ticket/4325)

+   **[postgresql] [bug]**

    修复了 `array_agg()` 函数，这是通常 `array_agg()` 函数的略有改变版本，也接受传入的“type”参数，而不强制将其包装在 ARRAY 中，本质上与 1.1 中为通用函数修复的相同问题 [#4107](https://www.sqlalchemy.org/trac/ticket/4107)。

    参考：[#4324](https://www.sqlalchemy.org/trac/ticket/4324)

+   **[postgresql] [bug]**

    修复了 PostgreSQL ENUM 反射中的 bug，即查询中会报告包含引号的区分大小写的名称，这在表反射期间不会与目标列匹配，因为需要去掉引号。 

    参考：[#4323](https://www.sqlalchemy.org/trac/ticket/4323)

### oracle

+   **[oracle] [bug]**

    修复了针对 cx_Oracle 7.0 的问题，其中 Oracle param.getvalue() 的行为现在返回一个列表，而不是单个标量值，这破坏了 Core 和 ORM 中的自增逻辑。对于 cx_Oracle 6.3 和 6.4，使用了 dml_ret_array_val 兼容标志来建立与 7.0 及更高版本的兼容行为，对于 cx_Oracle 6.2.1 及更早版本，版本号检查退回到旧逻辑。

    参考：[#4335](https://www.sqlalchemy.org/trac/ticket/4335)

### 杂项

+   **[bug] [ext]**

    修复了`BakedQuery`未包含会话（`Session`）使用的特定查询类作为缓存键的问题，导致在使用自定义查询类时不兼容，特别是`ShardedQuery`，它具有一些不同的参数签名。

    参考：[#4328](https://www.sqlalchemy.org/trac/ticket/4328)

### orm

+   **[orm] [bug]**

    在弱引用清理中增加了一个检查，检查`InstanceState` 对象中是否存在 `dict` 内置对象，以减少在解释器关闭时发生清理时生成的错误消息。感谢 Romuald Brunet 的拉取请求。

+   **[orm] [bug]**

    修复了在使用`Query.join()`以及`Query.select_entity_from()`与`Lateral`构造结合使用时，不会将子句适配应用于 join 的右侧的 bug。“lateral”引入了右侧 join 可相关的用例。以前，没有考虑到适配此子句。请注意，在 1.2 版本中，由 `Query.subquery()` 引入的可选择项仍未适配，因为[#4304](https://www.sqlalchemy.org/trac/ticket/4304)；可选择项需要由 `select()` 函数生成以成为“lateral”连接的右侧。

    参考：[#4334](https://www.sqlalchemy.org/trac/ticket/4334)

+   **[orm] [bug]**

    修复了由[#3472](https://www.sqlalchemy.org/trac/ticket/3472)引起的 1.2 回归，即在后续更新操作的上下文中处理“updated_at”样式列时，也会发生对于随后将被删除的行的更新，这意味着具有 Python 端值生成器的列将显示在 UPDATE 之前发出的现在已删除的值（这不是以前的行为），以及 SQL 发出的值生成器将使属性过期，这意味着由于行已被删除且对象已从会话中分离，因此无法访问先前的值。对于最终将被删除的对象，完全跳过了作为[#3472](https://www.sqlalchemy.org/trac/ticket/3472)的一部分添加的“postfetch”逻辑。

    参考：[#4327](https://www.sqlalchemy.org/trac/ticket/4327)

### orm 声明式

+   **[orm] [declarative] [bug]**

    修复了一个 bug，即在已映射类的子类上通过`@declared_attr`可调用获取描述符时，声明扫描属性会收到混合属性提供的表达式代理，而不是混合属性本身。这会导致在`Mapper.all_orm_descriptors`中查看时，该属性不会报告自身为混合属性。

    参考：[#4326](https://www.sqlalchemy.org/trac/ticket/4326)

### postgresql

+   **[postgresql] [bug]**

    修复了 PostgreSQL 方言中的 bug，即编译器关键字参数（如`literal_binds=True`）未传播到 DISTINCT ON 表达式。

    参考：[#4325](https://www.sqlalchemy.org/trac/ticket/4325)

+   **[postgresql] [bug]**

    修复了`array_agg()`函数，这是通常`array_agg()`函数的略有改变版本，也接受传入的“type”参数，而不强制在其周围添加一个 ARRAY，这与 1.1 中为通用函数修复的内容相同，见[#4107](https://www.sqlalchemy.org/trac/ticket/4107)。

    参考：[#4324](https://www.sqlalchemy.org/trac/ticket/4324)

+   **[postgresql] [bug]**

    修复了 PostgreSQL ENUM 反射中的 bug，即查询中包含带引号的区分大小写名称，这些引号在表反射期间不会匹配目标列，因为需要去掉引号。

    参考：[#4323](https://www.sqlalchemy.org/trac/ticket/4323)

### oracle

+   **[oracle] [bug]**

    修复了 cx_Oracle 7.0 中 Oracle param.getvalue() 的行为，现在返回一个列表，而不是单个标量值，这会破坏 Core 和 ORM 中的自增逻辑。对于 cx_Oracle 6.3 和 6.4，使用 dml_ret_array_val 兼容性标志来建立与 7.0 及以后版本的兼容性行为，对于 cx_Oracle 6.2.1 及之前的版本，版本号检查会回退到旧逻辑。

    参考文献：[#4335](https://www.sqlalchemy.org/trac/ticket/4335)

### misc

+   **[bug] [ext]**

    修复了 `BakedQuery` 不包含由 `Session` 使用的特定查询类作为缓存键的一部分的问题，导致在使用自定义查询类时不兼容，特别是 `ShardedQuery` 具有一些不同的参数签名。

    参考文献：[#4328](https://www.sqlalchemy.org/trac/ticket/4328)

## 1.2.11

发布日期：2018 年 8 月 20 日

### orm declarative

+   **[orm] [declarative] [bug]**

    修复了先前未经测试的用例中的问题，允许声明式映射类继承自声明基类之外的经典映射类，其中包括它适应未映射的中间类。未映射的中间类可以指定 `__abstract__`，现在可以正确解释，或者中间类可以保持未标记状态，并且经典映射的基类将在层次结构中被检测到。为了预期可能正在将经典映射混合到现有声明层次结构中的现有情景，如果检测到给定类的多个映射基类，则现在会引发错误。

    参考文献：[#4321](https://www.sqlalchemy.org/trac/ticket/4321)

### sql

+   **[sql] [bug]**

    修复了一个与 [#3639](https://www.sqlalchemy.org/trac/ticket/3639) 密切相关的问题，即在非本地布尔后端的布尔上下文中呈现的表达式将与 1/0 进行比较，即使它已经是隐式布尔表达式，当使用 `ColumnElement.self_group()` 时也是如此。虽然这不影响用户友好的后端（MySQL、SQLite），但 Oracle（可能还有 SQL Server）没有处理它。现在，任何数据库上是否隐式布尔表达式现在都会被事先确定为在语句的编译中不生成整数比较的附加检查。

    参考文献：[#4320](https://www.sqlalchemy.org/trac/ticket/4320)

+   **[sql] [bug]**

    将缺失的窗口函数参数`WithinGroup.over.range_`和`WithinGroup.over.rows`参数添加到`WithinGroup.over()`和`FunctionFilter.over()`方法中，以对应于 SQL 函数的“over”方法中在版本 1.1 中添加的 range/rows 功能。

    参考：[#4322](https://www.sqlalchemy.org/trac/ticket/4322)

+   **[sql] [bug]**

    修复了 UPDATE 和 DELETE 语句的多表支持未将额外的 FROM 元素视为与语句结合时的相关目标的 bug，当相关的 SELECT 也与语句结合时。此更改现在包括在 WHERE 子句中的 SELECT 语句将尝试自动关联回父 UPDATE/DELETE 中的这些额外表，或者如果使用 `Select.correlate()`，则无条件关联。请注意，如果 SELECT 语句的结果没有 FROM 子句，则自动关联会引发错误，这种情况现在可能发生，如果父 UPDATE/DELETE 在其额外的表集中指定相同的表，请显式指定 `Select.correlate()` 以解决。

    参考：[#4313](https://www.sqlalchemy.org/trac/ticket/4313)

### oracle

+   **[oracle] [bug]**

    对于 cx_Oracle，整数数据类型现在将绑定到“int”，根据 cx_Oracle 开发人员的建议。在 cx_Oracle 6.x 系列中，以前使用 cx_Oracle.NUMBER 会导致精度丢失。

    参考：[#4309](https://www.sqlalchemy.org/trac/ticket/4309)

### 杂项

+   **[bug] [py3k]**

    开始在 Python 3.3 及更高版本中从“collections.abc”导入“collections”以实现 Python 3.8 的兼容性。感谢 Nathaniel Knight 的拉取请求。

+   **[no_tags]**

    修复了在表反射中用于 SQLite 数据库的“schema”名称未正确引用模式名称的问题。感谢 Phillip Cloud 的拉取请求。

### orm 声明式

+   **[orm] [declarative] [bug]**

    修复了以前未经测试的用例中的问题，允许声明映射类从声明基类之外的经典映射类继承，包括适应未映射的中间类。未映射的中间类可以指定`__abstract__`，现在将正确解释，或者中间类可以保持未标记，而经典映射的基类将在层次结构中被检测到。为了预期可能将经典映射混合到现有声明层次结构中的现有场景，如果检测到给定类的多个映射基类，则现在会引发错误。

    参考：[#4321](https://www.sqlalchemy.org/trac/ticket/4321)

### sql

+   **[sql] [bug]**

    修复了一个与[#3639](https://www.sqlalchemy.org/trac/ticket/3639)密切相关的问题，即在非本地布尔后端上渲染为布尔上下文的表达式将与 1/0 进行比较，即使它已经是一个隐式布尔表达式，当使用`ColumnElement.self_group()`时。虽然这不会影响用户友好的后端（MySQL，SQLite），但 Oracle（可能还有 SQL Server）没有处理。现在，无论在任何数据库上表达式是否隐式布尔都将在编译语句时提前确定，以避免生成整数比较。

    参考：[#4320](https://www.sqlalchemy.org/trac/ticket/4320)

+   **[sql] [bug]**

    将缺失的窗口函数参数`WithinGroup.over.range_`和`WithinGroup.over.rows`参数添加到`WithinGroup.over()`和`FunctionFilter.over()`方法中，以对应于版本 1.1 中作为 SQL 函数“over”方法的一部分添加的 range/rows 功能的特性[#3049](https://www.sqlalchemy.org/trac/ticket/3049)。

    参考：[#4322](https://www.sqlalchemy.org/trac/ticket/4322)

+   **[sql] [bug]**

    修复了一个错误，在联合 SELECT 与语句结合时，UPDATE 和 DELETE 语句的多表支持没有将额外的 FROM 元素视为相关目标，此时一个相关的 SELECT 也与该语句结合。此更改现在包括了在这样一个语句的 WHERE 子句中的 SELECT 语句将尝试自动关联到父 UPDATE/DELETE 中的这些额外表，或者如果使用 `Select.correlate()` 则无条件关联。请注意，如果 SELECT 语句的结果没有 FROM 子句，则自动关联会引发错误，这现在可能会发生，如果父 UPDATE/DELETE 指定了相同的表在其额外的表集中；请显式指定 `Select.correlate()` 以解决此问题。

    参考：[#4313](https://www.sqlalchemy.org/trac/ticket/4313)

### oracle

+   **[oracle] [bug]**

    对于 cx_Oracle，Integer 数据类型现在将绑定到“int”，根据 cx_Oracle 开发人员的建议。以前，在 cx_Oracle 6.x 系列中使用 cx_Oracle.NUMBER 会导致精度丢失。

    参考：[#4309](https://www.sqlalchemy.org/trac/ticket/4309)

### 杂项

+   **[bug] [py3k]**

    开始在 Python 3.3 及更高版本中从“collections”导入“collections.abc”，以实现与 Python 3.8 的兼容性。由 Nathaniel Knight 提供的 Pull 请求。

+   **[no_tags]**

    修复了在表反射中使用的 SQLite 数据库的“schema”名称无法正确引用模式名称的问题。由 Phillip Cloud 提供的 Pull 请求。

## 1.2.10

发布日期：2018 年 7 月 13 日

### orm

+   **[orm] [bug]**

    在`Bundle`构造中修复了一个错误，当放置两个同名列时，它们会被去重，当`Bundle`作为渲染的 SQL 的一部分使用时，比如在语句的 ORDER BY 或 GROUP BY 中。

    参考：[#4295](https://www.sqlalchemy.org/trac/ticket/4295)

+   **[orm] [bug]**

    由于 [#4287](https://www.sqlalchemy.org/trac/ticket/4287) 导致 1.2.9 中的回归错误，使用 `Load` 选项与字符串通配符结合使用会导致 TypeError。

    参考：[#4298](https://www.sqlalchemy.org/trac/ticket/4298)

### sql

+   **[sql] [bug]**

    修复了一个 bug，在任何引用它的`Table`之前，会明确删除`Sequence`，这会在序列还涉及该表的服务器端默认值时出现问题，当使用`MetaData.drop_all()`时。现在，在删除表本身之后，会调用处理要通过非服务器端列默认函数删除的序列的步骤。

    参考：[#4300](https://www.sqlalchemy.org/trac/ticket/4300)

### orm

+   **[orm] [bug]**

    修复了在`Bundle`构造中的 bug，当将两个同名列放置在一起时，会被去重，当`Bundle`作为渲染 SQL 的一部分使用时，例如在语句的 ORDER BY 或 GROUP BY 中。

    参考：[#4295](https://www.sqlalchemy.org/trac/ticket/4295)

+   **[orm] [bug]**

    由于[#4287](https://www.sqlalchemy.org/trac/ticket/4287)，在 1.2.9 中出现了回归，使用`Load`选项与��符串通配符一起使用会导致 TypeError。

    参考：[#4298](https://www.sqlalchemy.org/trac/ticket/4298)

### sql

+   **[sql] [bug]**

    修复了一个 bug，在任何引用它的`Table`之前，会明确删除`Sequence`，这会在序列还涉及该表的服务器端默认值时出现问题，当使用`MetaData.drop_all()`时。现在，在删除表本身之后，会调用处理要通过非服务器端列默认函数删除的序列的步骤。

    参考：[#4300](https://www.sqlalchemy.org/trac/ticket/4300)

## 1.2.9

发布日期：2018 年 6 月 29 日

### orm

+   **[orm] [bug]**

    修复了在`Query.join()`中链接多个连接元素可能无法正确适应先前左侧的问题，当链接共享相同基类的连接继承类时。

    参考：[#3505](https://www.sqlalchemy.org/trac/ticket/3505)

+   **[orm] [bug]**

    修复了为烘焙查询生成缓存键时的 bug，这可能导致为跨子类的贪婪加载生成一个过短的缓存键。这反过来可能导致贪婪加载查询被缓存，而不是非贪婪加载查询，或者反之，对于多态“selectin”加载，或者可能对于延迟加载或 selectin 加载也是如此。

    参考文献：[#4287](https://www.sqlalchemy.org/trac/ticket/4287)

+   **[orm] [bug]**

    修复了新多态选择加载中的错误，其中内部使用的 BakedQuery 会被给定的加载器选项所改变，这既会不适当地改变子类查询，也会将效果带到后续查询中。

    参考文献：[#4286](https://www.sqlalchemy.org/trac/ticket/4286)

+   **[orm] [bug]**

    由于 [#4256](https://www.sqlalchemy.org/trac/ticket/4256) 引起的回归问题已修复（本身是对 [#4228](https://www.sqlalchemy.org/trac/ticket/4228) 的回归修复），它中断了一项未记录的行为，即将直接传递给 `Query` 构造函数的非实体序列转换为单一元素序列。虽然这种行为从未得到支持或记录，但已经在使用中，因此已将其添加为 `Query` 的行为约定。

    参考文献：[#4269](https://www.sqlalchemy.org/trac/ticket/4269)

+   **[orm] [bug]**

    修复了在 1.2 版本中的性能回归和“烘焙”懒加载器中的错误结果，涉及从原始 `Query` 对象的加载器选项生成缓存键。如果加载器选项是以“分支”样式构建的，使用相同的基本元素来构建多个选项，那么相同的选项将被重复渲染到缓存键中，导致性能问题以及生成错误的缓存键。已修复此问题，并通过在应用 `Query.options()` 时防止重复应用相同选项对象来提高性能，以防止重复应用“分支”选项。

    参考文献：[#4270](https://www.sqlalchemy.org/trac/ticket/4270)

### sql

+   **[sql] [bug]**

    由于 [#4147](https://www.sqlalchemy.org/trac/ticket/4147) 引起的 1.2 版本中的回归问题已修复，其中一个 `Table` 在反射期间重写其部分索引列或在使用 `Table.extend_existing` 时重新定义其部分索引列时，`Table.tometadata()` 方法在尝试复制这些索引时将失败，因为它们仍然引用替换的列。现在的复制逻辑已适应了这种情况。

    参考文献：[#4279](https://www.sqlalchemy.org/trac/ticket/4279)

### mysql

+   **[mysql] [bug]**

    修复了 mysql-connector-python 方言中百分号重复的问题，不需要去除百分号的重复。此外，mysql-connector-python 驱动在传递游标描述中的列名时不一致，因此添加了一个解码器来有条件地将这些随机的字节值解码为 Unicode，仅在需要时进行解码。同时改进了 mysql-connector-python 的测试支持，但需要注意的是，该驱动程序仍然存在与 Unicode 相关的问题，目前尚未解决。

+   **[mysql] [bug]**

    修复了索引反射中的 bug，在 MySQL 8.0 上，包含 ASC 或 DESC 在索引列规范中的索引将不会被正确反映，因为 MySQL 8.0 引入了在表定义字符串中返回此信息的支持。

    参考：[#4293](https://www.sqlalchemy.org/trac/ticket/4293)

+   **[mysql] [bug]**

    修复了 MySQLdb 方言和 PyMySQL 等变体中的 bug，在连接时对“unicode 返回”进行额外检查，明确使用“utf8”字符集，在 MySQL 8.0 中会发出警告，建议使用 utf8mb4。现在已经替换为 utf8mb4 等效。还更新了 MySQL 方言的文档，以在所有示例中指定 utf8mb4。对测试套件进行了额外的更改，以使用 utf8mb3 字符集和数据库（在某些边缘情况下 utf8mb4 存在排序问题），并支持 MySQL 8.0 中进行的配置默认更改，如 explicit_defaults_for_timestamp 以及为无效的 MyISAM 索引引发的新错误。

    参考：[#4283](https://www.sqlalchemy.org/trac/ticket/4283)

+   **[mysql] [bug]**

    `Update`构造现在支持`Join`对象，这是 MySQL 对 UPDATE..FROM 支持的一部分。由于该构造已经接受了一个别名对象用于类似的目的，因此已经暗示了针对非表的 UPDATE 功能，因此已经添加了这个功能。

    参考：[#3645](https://www.sqlalchemy.org/trac/ticket/3645)

### sqlite

+   **[sqlite] [bug]**

    修复了测试套件中的问题，SQLite 3.24 添加了一个新的保留字，与 TypeReflectionTest 中的使用发生冲突。感谢 Nils Philippsen 提供的拉取请求。

### mssql

+   **[mssql] [bug]**

    修复了 MSSQL 反射中的 bug，在不同模式中有两个同名表具有同名主键约束时，引用其中一个表的外键约束的列会被加倍，导致错误。感谢 Sean Dunn 提供的拉取请求。

    参考：[#4288](https://www.sqlalchemy.org/trac/ticket/4288)

+   **[mssql] [bug] [py3k]**

    修复了在 Python 3 下 SQL Server 方言中的问题，当针对不包含“sys.dm_exec_sessions”或“sys.dm_pdw_nodes_exec_sessions”视图的非标准 SQL Server 数据库运行时，导致无法获取隔离级别，由于 UnboundLocalError 导致错误引发失败。

    参考：[#4273](https://www.sqlalchemy.org/trac/ticket/4273)

### oracle

+   **[oracle] [feature]**

    添加了一个新事件，目前仅由 cx_Oracle 方言使用，`DialectEvents.setiputsizes()`。该事件传递了一个包含 `BindParameter` 对象的字典，这些对象将被传递给 DBAPI 特定类型的对象，然后转换为参数名称，传递给 cx_Oracle `cursor.setinputsizes()` 方法。这既允许查看 setinputsizes 过程，也允许更改传递给此方法的数据类型的行为。

    另请参阅

    使用 setinputsizes 对 cx_Oracle 数据绑定性能进行细粒度控制

    参考：[#4290](https://www.sqlalchemy.org/trac/ticket/4290)

+   **[oracle] [bug] [mysql]**

    修复了在 Oracle 和 MySQL 方言中使用 CTEs 进行 INSERT FROM SELECT 时的问题，其中 CTE 被放置在整个语句之上，这在其他数据库中是典型的，但是 Oracle 和 MariaDB 10.2 希望 CTE 出现在“INSERT”段的下方。请注意，当将 CTE 应用于 UPDATE 或 DELETE 语句内的子查询时，Oracle 和 MySQL 方言尚不起作用，因为 CTE 仍然应用于顶部而不是子查询内部。

    参考：[#4275](https://www.sqlalchemy.org/trac/ticket/4275)

### 杂项

+   **[feature] [ext]**

    添加了新属性 `Query.lazy_loaded_from`，其中填充了使用此 `Query` 来延迟加载关系的 `InstanceState`。这样做的理由是它作为水平分片功能的提示，以便使用状态的标识令牌作为查询中的默认标识令牌在 id_chooser() 中使用。

    参考：[#4243](https://www.sqlalchemy.org/trac/ticket/4243)

+   **[bug] [py3k]**

    用从 Python 标准库复制的 vendored 版本替换了 inspect.formatargspec() 的使用，因为 inspect.formatargspec() 已被弃用，并且从 Python 3.7.0 开始发出警告。

    参考：[#4291](https://www.sqlalchemy.org/trac/ticket/4291)

### orm

+   **[orm] [bug]**

    修复了在`Query.join()`中链接多个连接元素时可能无法正确适应先前左侧的问题，当链接共享相同基类的继承类时。

    参考：[#3505](https://www.sqlalchemy.org/trac/ticket/3505)

+   **[orm] [bug]**

    修复了为烘焙查询生成缓存键时可能导致生成的缓存键过短的问题，对于跨子类的急加载。这可能会导致急加载查询被缓存代替非急加载查询，反之亦然，对于多态“selectin”加载，或者可能对于延迟加载或 selectin 加载也是如此。

    参考：[#4287](https://www.sqlalchemy.org/trac/ticket/4287)

+   **[orm] [bug]**

    修复了新多态 selectin 加载中的错误，其中内部使用的 BakedQuery 会被给定的加载器选项改变，这既会不当地改变子类查询，也会将效果传递给后续查询。

    参考：[#4286](https://www.sqlalchemy.org/trac/ticket/4286)

+   **[orm] [bug]**

    修复了由[#4256](https://www.sqlalchemy.org/trac/ticket/4256)引起的回归（本身是对[#4228](https://www.sqlalchemy.org/trac/ticket/4228)的回归修复），它破坏了一个未记录的行为，即将直接传递给`Query`构造函数的非实体序列转换为单个元素序列。虽然这种行为从未得到支持或记录，但已经在使用中，因此已被添加为`Query`的行为契约。

    参考：[#4269](https://www.sqlalchemy.org/trac/ticket/4269)

+   **[orm] [bug]**

    修复了 1.2 版本中的性能回归问题以及关于“烘焙”延迟加载器的错误结果，涉及从原始`Query`对象的加载器选项生成缓存键。如果加载器选项是以“分支”样式构建的，使用共同的基本元素为多个选项，那么相同的选项将被重复渲染到缓存键中，导致性能问题以及生成错误的缓存键。通过`Query.options()`应用这种“分支”选项时，已修复此问题，并防止重复应用相同的选项对象以提高性能。

    参考：[#4270](https://www.sqlalchemy.org/trac/ticket/4270)

### sql

+   **[sql] [bug]**

    修复了 1.2 版本中的回归问题，由于 [#4147](https://www.sqlalchemy.org/trac/ticket/4147) 导致一个 `Table` 的一些索引列被重新定义为新列，这种情况会在反射期间覆盖列或使用 `Table.extend_existing` 时发生，因此当尝试复制这些索引时，`Table.tometadata()` 方法会失败，因为它们仍然指向被替换的列。现在的复制逻辑已经适应了这种情况。

    参考：[#4279](https://www.sqlalchemy.org/trac/ticket/4279)

### mysql

+   **[mysql] [bug]**

    修复了 mysql-connector-python 方言中百分号双倍的问题，该方言不需要去除百分号的双倍。此外，mysql-connector-python 驱动程序在传递列名时不一致，因此添加了一个解码器来有条件地将这些随机-有时是字节的值解码为 unicode，仅在需要时。还改进了对 mysql-connector-python 的测试支持，但应注意该驱动程序仍存在与 unicode 相关的问题，目前尚未解决。

+   **[mysql] [bug]**

    修复了在 MySQL 8.0 中索引反射中的一个 bug，其中在索引列规范中包含 ASC 或 DESC 的索引不会被正确反映，因为 MySQL 8.0 引入了在表定义字符串中返回此信息的支持。

    参考：[#4293](https://www.sqlalchemy.org/trac/ticket/4293)

+   **[mysql] [bug]**

    修复了 MySQLdb 方言和 PyMySQL 等变体中的一个 bug，在连接时对“unicode 返回”进行额外检查时，显式使用“utf8”字符集，而在 MySQL 8.0 中会发出警告，建议使用 utf8mb4。现在已经用 utf8mb4 等效替换。还更新了 MySQL 方言的文档，以在所有示例中指定 utf8mb4。对测试套件进行了额外更改，以使用 utf8mb3 字符集和数据库（在某些边缘情况下 utf8mb4 存在排序问题），并支持 MySQL 8.0 中进行的配置默认更改，如 explicit_defaults_for_timestamp 以及对无效 MyISAM 索引引发的新错误。

    参考：[#4283](https://www.sqlalchemy.org/trac/ticket/4283)

+   **[mysql] [bug]**

    `Update` 构造现在支持将 `Join` 对象作为 MySQL 中 UPDATE..FROM 支持的对象。由于该构造已经接受了一个别名对象用于类似的目的，因此已经暗示了针对非表的 UPDATE 功能，因此这一功能已经被添加。

    参考：[#3645](https://www.sqlalchemy.org/trac/ticket/3645)

### sqlite

+   **[sqlite] [bug]**

    修复了测试套件中的问题，SQLite 3.24 添加了一个新的保留字与 TypeReflectionTest 中的用法冲突。感谢 Nils Philippsen 的拉取请求。

### mssql

+   **[mssql] [bug]**

    修复了 MSSQL 反射中的错误，当不同模式中有两个同名表具有相同名字的主键约束时，引用其中一个表的外键约束的列会加倍，导致错误。感谢 Sean Dunn 的拉取请求。

    参考：[#4288](https://www.sqlalchemy.org/trac/ticket/4288)

+   **[mssql] [bug] [py3k]**

    修复了在 Python 3 下的 SQL Server 方言中的问题，在运行针对不包含“sys.dm_exec_sessions”或“sys.dm_pdw_nodes_exec_sessions”视图的非标准 SQL Server 数据库时，导致无法获取隔离级别，由于 UnboundLocalError 导致错误提出失败。

    参考：[#4273](https://www.sqlalchemy.org/trac/ticket/4273)

### oracle

+   **[oracle] [feature]**

    添加了一个新的事件，目前仅由 cx_Oracle 方言使用，`DialectEvents.setiputsizes()`。该事件传递了一个 `BindParameter` 对象字典给将要传递给 cx_Oracle `cursor.setinputsizes()` 方法的 DBAPI 特定类型对象。这允许查看 setinputsizes 过程并能够改变传递给此方法的数据类型的行为。

    另见

    通过 setinputsizes 实现对 cx_Oracle 数据绑定性能的细粒度控制

    参考：[#4290](https://www.sqlalchemy.org/trac/ticket/4290)

+   **[oracle] [bug] [mysql]**

    修复了在 Oracle 和 MySQL 方言中使用 CTE 进行 INSERT FROM SELECT 时的问题，其中 CTE 被放置在整个语句上方，与其他数据库一样，但是 Oracle 和 MariaDB 10.2 要求 CTE 放在“INSERT”段下方。请注意，当将 CTE 应用于 UPDATE 或 DELETE 语句内部的子查询时，Oracle 和 MySQL 方言尚不起作用，因为 CTE 仍然应用于顶部而不是内部子查询。

    参考：[#4275](https://www.sqlalchemy.org/trac/ticket/4275)

### 杂项

+   **[feature] [ext]**

    添加了新属性 `Query.lazy_loaded_from` ，其中填充了一个使用此 `Query` 来延迟加载关系的 `InstanceState`。这样做的理由是它作为水平分片功能的提示，以便可以使用状态的标识令牌作为查询中要使用的默认标识令牌。

    参考：[#4243](https://www.sqlalchemy.org/trac/ticket/4243)

+   **[bug] [py3k]**

    用从 Python 标准库复制的 vendored 版本替换了 inspect.formatargspec()的使用，因为 inspect.formatargspec()已被弃用，并且从 Python 3.7.0 开始发出警告。

    参考：[#4291](https://www.sqlalchemy.org/trac/ticket/4291)

## 1.2.8

发布日期：2018 年 5 月 28 日

### orm

+   **[orm] [bug]**

    修复了 1.2.7 中由[#4228](https://www.sqlalchemy.org/trac/ticket/4228)引起的回归，该回归本身修复了一个 1.2 级别的回归，其中传递给`Session`的`query_cls`可调用被假定为具有类方法可用性的`Query`子类，而不是任意可调用对象。特别是，dogpile 缓存示例说明了`query_cls`是一个函数而不是`Query`子类。

    参考：[#4256](https://www.sqlalchemy.org/trac/ticket/4256)

+   **[orm] [bug]**

    修复了 1.0 版本中长期存在的回归，该回归阻止了使用自定义`MapperOption`来修改`Query`对象的 _params 以进行延迟加载，因为延迟加载器本身会覆盖这些参数。这适用于维基上的“时间范围”示例。但请注意，现在需要使用`Query.populate_existing()`方法来重写与已加载到标识映射中的对象相关联的映射器选项。

    作为这一变更的一部分，现在自定义定义的`MapperOption`将导致与目标对象相关的延迟加载器默认使用非烘焙查询，除非实现了`MapperOption._generate_cache_key()`方法。特别是，修复了一个回归，该回归发生在使用 dogpile.cache“高级”示例时，该示例由于与烘焙查询加载器不兼容而未返回缓存结果，而是发出 SQL；通过这一变更，dogpile 示例中包含的多个版本的`RelationshipCache`选项将完全禁用“烘焙”查询。请注意，dogpile 示例也经过现代化处理，以避免这两个问题，作为问题[#4258](https://www.sqlalchemy.org/trac/ticket/4258)的一部分。

    参考：[#4128](https://www.sqlalchemy.org/trac/ticket/4128)

+   **[orm] [bug]**

    修复了新的`Result.with_post_criteria()`方法与子查询急加载器交互不正确的 bug，即“后置条件”不会应用于嵌入式子查询急加载器。这与[#4128](https://www.sqlalchemy.org/trac/ticket/4128)有关，因为后置条件功能现在被延迟加载器使用。

+   **[orm] [bug]**

    更新了 dogpile.caching 示例，包括适应 “baked” 查询系统的新结构，该系统默认在延迟加载器和一些急切关系加载器中使用。由于 [#4256](https://www.sqlalchemy.org/trac/ticket/4256)，dogpile.caching 的 “relationship_caching” 和 “advanced” 示例也被破坏。这里的问题也通过 [#4128](https://www.sqlalchemy.org/trac/ticket/4128) 中的修复解决了。

    引用：[#4258](https://www.sqlalchemy.org/trac/ticket/4258)

### engine

+   **[engine] [bug]**

    修复了连接池问题，即如果在连接池的 “返回时重置” 序列期间引发了断开连接错误，并且针对外部 `Connection` 对象打开了显式事务（例如从调用 `Session.close()` 而不是回滚或提交，或者在调用 `Connection.close()` 之前没有关闭使用 `Connection.begin()` 声明的事务），将导致二次签入，这可能会导致同一连接的并发签出。现在通过断言来防止整体的二次签入条件，以及特定的二次签入场景已被修复。

    引用：[#4252](https://www.sqlalchemy.org/trac/ticket/4252)

+   **[engine] [bug]**

    修复了一个引用泄漏问题，其中在语句执行中使用的参数字典的值将继续被 “编译缓存” 引用，因为存储了 Python 3 字典 keys() 使用的键视图。感谢 Olivier Grisel 提交的拉取请求。

### sql

+   **[sql] [bug]**

    修复了在将文本值解释为 SQL 表达式值时遇到元组值并且无法正确格式化消息的“模糊文本”错误消息问题。感谢 Miguel Ventura 提交的拉取请求。

### mssql

+   **[mssql] [bug]**

    修复了由 [#4061](https://www.sqlalchemy.org/trac/ticket/4061) 引起的 1.2 版本回归，其中 SQL Server 的 “BIT” 类型将被视为 “本机布尔” 的问题。这里的目标是避免在列上创建 CHECK 约束，然而更大的问题是 BIT 值不像一个真/假常量那样工作，不能被解释为一个独立的表达式，例如 “WHERE <column>”。SQL Server 方言现在回到了非本机布尔，但增加了一个额外的标志，仍然避免创建 CHECK 约束。

    引用：[#4250](https://www.sqlalchemy.org/trac/ticket/4250)

### oracle

+   **[oracle] [bug]**

    Oracle 的 BINARY_FLOAT 和 BINARY_DOUBLE 数据类型现在参与 cx_Oracle.setinputsizes()，传递 NATIVE_FLOAT，以支持 NaN 值。此外，`BINARY_FLOAT`、`BINARY_DOUBLE` 和 `DOUBLE_PRECISION` 现在都是 `Float` 的子类，因为这些是浮点数据类型，而不是十进制。这些数据类型已经将 `Float.asdecimal` 标志默认设置为 False，与 `Float` 的行为一致。

    参考：[#4264](https://www.sqlalchemy.org/trac/ticket/4264)

+   **[oracle] [bug]**

    为 `BINARY_FLOAT`、`BINARY_DOUBLE` 数据类型添加了反��功能。

+   **[oracle] [bug]**

    修改了 Oracle 方言，当使用 `Integer` 类型时，设置 cx_Oracle.NUMERIC 类型用于 setinputsizes()。在 SQLAlchemy 1.1 及更早版本中，cx_Oracle.NUMERIC 无条件地传递给所有数值类型，而在 1.2 中，为了获得更好的数值精度，这一点被移除。然而，对于整数，一些数据库/客户端设置将无法将布尔值 True/False 强制转换为整数，这在使用 SQLAlchemy 1.2 时引入了退化行为。总体而言，setinputsizes 逻辑似乎需要更多的灵活性，因此这是一个开始。

    参考：[#4259](https://www.sqlalchemy.org/trac/ticket/4259)

### 测试

+   **[tests] [bug]**

    修复了测试套件中的一个 bug，如果外部方言对 `server_version_info` 返回 `None`，排除逻辑会引发 `AttributeError`。

    参考：[#4249](https://www.sqlalchemy.org/trac/ticket/4249)

### 杂项

+   **[bug] [ext]**

    水平分片扩展现在利用了作为 [#4137](https://www.sqlalchemy.org/trac/ticket/4137) 的一部分添加到 ORM 身份键中的标识令牌，当对象刷新或基于列的延迟加载或取消过期操作发生时。由于我们知道对象的“分片”来自哪里，因此在刷新时利用这个值，从而避免针对与此对象身份不匹配的其他分片进行查询。

    参考：[#4247](https://www.sqlalchemy.org/trac/ticket/4247)

+   **[bug] [ext]**

    修复了一种竞争条件，该条件可能会在多线程环境中使用 automap `AutomapBase.prepare()`与可能调用`configure_mappers()`的其他线程相互作用时发生，因为使用其他映射器会导致未完成的 automap 映射工作被`configure_mappers()`步骤拉入，从而导致错误。

    参考：[#4266](https://www.sqlalchemy.org/trac/ticket/4266)

### orm

+   **[orm] [bug]**

    修复了 1.2.7 版本中由[#4228](https://www.sqlalchemy.org/trac/ticket/4228)引起的回归问题，该问题本身修复了一个 1.2 级别的回归问题，即假定传递给`Session`的`query_cls`可调用对象是`Query`的子类，并具有类方法可用性，而不是任意可调用对象。特别是，dogpile 缓存示例说明了`query_cls`是一个函数而不是`Query`的子类。

    参考：[#4256](https://www.sqlalchemy.org/trac/ticket/4256)

+   **[orm] [bug]**

    修复了 1.0 版本中长期存在的回归问题，该问题阻止了自定义`MapperOption`用于更改`Query`对象的 _params 以进行延迟加载，因为延迟加载器本身会覆盖这些参数。这适用于维基上的“时间范围”示例。但请注意，现在需要使用`Query.populate_existing()`方法来重写与已加载到标识映射中的对象相关联的映射器选项。

    作为这一变更的一部分，现在自定义定义的`MapperOption`将导致与目标对象相关的延迟加载器默认使用非烘焙查询，除非实现了`MapperOption._generate_cache_key()`方法。特别是，这修复了一个回归问题，即在使用 dogpile.cache 的“高级”示例时发生的问题，该问题未返回缓存结果，而是由于与烘焙查询加载器不兼容而发出 SQL；通过更改，dogpile 示例中包含的`RelationshipCache`选项将完全禁用“烘焙”查询。请注意，作为问题[#4258](https://www.sqlalchemy.org/trac/ticket/4258)的一部分，dogpile 示例也现代化以避免这两个问题。

    参考：[#4128](https://www.sqlalchemy.org/trac/ticket/4128)

+   **[orm] [bug]**

    修复了新的`Result.with_post_criteria()`方法与子查询急加载器的交互不正确的错误，即“后置条件”不会应用于嵌入式子查询急加载器。这与[#4128](https://www.sqlalchemy.org/trac/ticket/4128)有关，因为现在懒加载器使用了后置条件功能。

+   **[orm] [bug]**

    更新了 dogpile.caching 示例，包括适用于“baked”查询系统的新结构，默认情况下在懒加载器和一些急加载关系加载器中使用。dogpile.caching 的“relationship_caching”和“advanced”示例也由于[#4256](https://www.sqlalchemy.org/trac/ticket/4256)而中断。这里的问题也通过[#4128](https://www.sqlalchemy.org/trac/ticket/4128)中的修复来解决。

    参考：[#4258](https://www.sqlalchemy.org/trac/ticket/4258)

### engine

+   **[engine] [bug]**

    修复了连接池问题，即如果在连接池的“重置返回时”序列中引发了断开连接错误，并且同时针对封闭的`Connection`对象打开了显式事务（例如从调用`Session.close()`而没有回滚或提交，或者在调用`Connection.close()`之前没有关闭使用`Connection.begin()`声明的事务），将导致双重签入，然后可能导致对同一连接的并发签出。通过断言来防止整体双重签入条件，以及特定的双重签入场景已得到修复。

    参考：[#4252](https://www.sqlalchemy.org/trac/ticket/4252)

+   **[engine] [bug]**

    修复了引用泄漏问题，即在语句执行中使用的参数字典的值将继续被“编译缓存”引用，因为存储了 Python 3 字典 keys()使用的键视图。感谢 Olivier Grisel 的拉取请求。

### sql

+   **[sql] [bug]**

    修复了在将文字值解释为 SQL 表达式值时遇到元组值时使用的“模棱两可的文字”错误消息，并未正确格式化消息的问题。感谢 Miguel Ventura 的拉取请求。

### mssql

+   **[mssql] [bug]**

    修复了由[#4061](https://www.sqlalchemy.org/trac/ticket/4061)引起的 1.2 版本回归问题，其中 SQL Server 的“BIT”类型被认为是“本地布尔”。这里的目标是避免在列上创建 CHECK 约束，但更大的问题是 BIT 值不像真/假常量那样行为，并且不能被解释为独立表达式，例如“WHERE <column>”。SQL Server 方言现在回到了非本地布尔，但增加了一个额外的标志，仍然避免创建 CHECK 约束。

    参考：[#4250](https://www.sqlalchemy.org/trac/ticket/4250)

### Oracle

+   **[oracle] [bug]**

    Oracle 的 BINARY_FLOAT 和 BINARY_DOUBLE 数据类型现在参与 cx_Oracle.setinputsizes()，传递 NATIVE_FLOAT，以支持 NaN 值。此外，`BINARY_FLOAT`、`BINARY_DOUBLE`和`DOUBLE_PRECISION`现在都是`Float`的子类，因为这些是浮点数据类型，而不是十进制。这些数据类型已经将`Float.asdecimal`标志默认设置为 False，与`Float`的行为一致。

    参考：[#4264](https://www.sqlalchemy.org/trac/ticket/4264)

+   **[oracle] [bug]**

    为`BINARY_FLOAT`、`BINARY_DOUBLE`数据类型添加了反射功能。

+   **[oracle] [bug]**

    修改了 Oracle 方言，当使用`Integer`类型时，设置了 cx_Oracle.NUMERIC 类型用于 setinputsizes()。在 SQLAlchemy 1.1 及更早版本中，cx_Oracle.NUMERIC 无条件地传递给所有数值类型，而在 1.2 中，这被移除以允许更好的数值精度。然而，对于整数，一些数据库/客户端设置将无法将布尔值 True/False 强制转换为整数，这在使用 SQLAlchemy 1.2 时引入了回归行为。总体而言，setinputsizes 逻辑似乎需要更多的灵活性，因此这是一个开始。

    参考：[#4259](https://www.sqlalchemy.org/trac/ticket/4259)

### 测试

+   **[测试] [bug]**

    修复了测试套件中的一个错误，如果外部方言对`server_version_info`返回`None`，则排除逻辑将引发`AttributeError`。

    参考：[#4249](https://www.sqlalchemy.org/trac/ticket/4249)

### 杂项

+   **[bug] [ext]**

    水平分片扩展现在利用了作为[#4137](https://www.sqlalchemy.org/trac/ticket/4137)的一部分添加到 ORM 标识键中的标识令牌，当对象刷新或基于列的延迟加载或取消过期操作发生时。由于我们知道对象来自的“分片”，因此在刷新时我们利用此值，从而避免针对与此对象标识不匹配的其他分片进行查询。

    参考：[#4247](https://www.sqlalchemy.org/trac/ticket/4247)

+   **[bug] [ext]**

    修复了可能发生的竞争条件，如果在多线程上下文中使用 automap `AutomapBase.prepare()`与其他可能调用`configure_mappers()`的线程一起使用其他映射器。 automap 的未完成映射工作对于被`configure_mappers()`步骤拉入尤为敏感，导致错误。

    参考：[#4266](https://www.sqlalchemy.org/trac/ticket/4266)

## 1.2.7

发布日期：2018 年 4 月 20 日

### orm

+   **[orm] [bug]**

    修复了 1.2 版本中分片查询功能中的回归，其中新的“identity_token”元素在懒加载操作范围内未被正确考虑，当在身份映射中搜索相关的多对一元素时。新行为将允许利用“id_chooser”来确定从身份映射中检索的最佳标识键。为了实现这一点，对 1.2 的“identity_token”方法进行了一些重构，对`ShardedQuery`的实现进行了一些细微更改，其他派生类应该注意这些更改。

    参考：[#4228](https://www.sqlalchemy.org/trac/ticket/4228)

+   **[orm] [bug]**

    修复了单继承加载中的问题，其中使用别名实体与单继承子类结合使用`Query.select_from()`方法会导致 SQL 呈现为未别名化的表混入查询中，导致笛卡尔积。特别是当针对单继承子类使用新的“selectin”加载器时，会受到影响。

    参考：[#4241](https://www.sqlalchemy.org/trac/ticket/4241)

### sql

+   **[sql] [bug]**

    修复了使用“literal_binds”选项编译 INSERT 语句时的问题，该语句还使用了显式序列和“inline”生成，如在 PostgreSQL 和 Oracle 上，会导致序列处理程序无法容纳额外的关键字参数。

    参考：[#4231](https://www.sqlalchemy.org/trac/ticket/4231)

### postgresql

+   **[postgresql] [feature]**

    添加了新的 PG 类型 `REGCLASS`，有助于将表名转换为 OID 值。感谢 Sebastian Bank 的拉取请求。

    参考：[#4160](https://www.sqlalchemy.org/trac/ticket/4160)

+   **[postgresql] [bug]**

    修复了 PostgreSQL “range” 数据类型（如 DATERANGE）的特殊“不等于”运算符与 Python `None` 值比较时无法渲染“IS NOT NULL”的 bug。

    参考：[#4229](https://www.sqlalchemy.org/trac/ticket/4229)

### mssql

+   **[mssql] [bug]**

    修复了由 [#4060](https://www.sqlalchemy.org/trac/ticket/4060) 引起的 1.2 版本回归 bug，导致用于反映 SQL Server 跨模式外键的查询错误地限制了条件。

    参考：[#4234](https://www.sqlalchemy.org/trac/ticket/4234)

### oracle

+   **[oracle] [bug]**

    如果精度为 NULL 且比例为零，则 Oracle NUMBER 数据类型将反映为 INTEGER，因为这是从 Oracle 表反映出来的 INTEGER 值。感谢 Kent Bower 的拉取请求。

### orm

+   **[orm] [bug]**

    修复了 1.2 版本中分片查询功能中的回归 bug，其中新的“identity_token”元素在搜索相关的一对多元素时未被正确考虑在延迟加载操作范围内。新的行为将允许利用“id_chooser”来确定从身份映射中检索的最佳身份键。为了实现这一点，对 1.2 版本的“identity_token”方法进行了一些重构，对于此类的其他派生应该注意到一些对 `ShardedQuery` 实现的轻微更改。

    参考：[#4228](https://www.sqlalchemy.org/trac/ticket/4228)

+   **[orm] [bug]**

    修复了单继承加载中的问题，其中在使用 `Query.select_from()` 方法时，针对单继承子类使用别名实体会导致 SQL 渲染时未经别名处理的表混入查询，导致笛卡尔积。特别是当针对单继承子类使用新的“selectin”加载器时会受到影响。

    参考：[#4241](https://www.sqlalchemy.org/trac/ticket/4241)

### sql

+   **[sql] [bug]**

    修复了使用“literal_binds”选项编译 INSERT 语句时，同时使用显式序列和“inline”生成（如在 PostgreSQL 和 Oracle 上），无法在序列处理过程中适应额外关键字参数的问题。

    参考：[#4231](https://www.sqlalchemy.org/trac/ticket/4231)

### postgresql

+   **[postgresql] [feature]**

    添加了新的 PG 类型 `REGCLASS`，有助于将表名转换为 OID 值。感谢 Sebastian Bank 的拉取请求。

    参考：[#4160](https://www.sqlalchemy.org/trac/ticket/4160)

+   **[PostgreSQL] [错误]**

    修复了针对 PostgreSQL“range”数据类型（如 DATERANGE）的特殊“不等于”运算符与 Python `None`值比较时无法呈现“IS NOT NULL”的错误。

    参考：[#4229](https://www.sqlalchemy.org/trac/ticket/4229)

### MSSQL

+   **[MSSQL] [错误]**

    修复了由[#4060](https://www.sqlalchemy.org/trac/ticket/4060)引起的 1.2 回归，其中用于反映 SQL Server 跨模式外键的查询错误地限制了条件。

    参考：[#4234](https://www.sqlalchemy.org/trac/ticket/4234)

### Oracle

+   **[Oracle] [错误]**

    如果精度为 NULL 且比例为零，则 Oracle NUMBER 数据类型将反映为 INTEGER，因为这是从 Oracle 表反映出来的 INTEGER 值的方式。拉取请求由 Kent Bower 提供。

## 1.2.6

发布日期：2018 年 3 月 30 日

### ORM

+   **[ORM] [错误]**

    修复了在与具有替代命名属性设置的非主映射器一起使用`Mutable.associate_with()`或`Mutable.as_mutable()`时会产生属性错误的错误。由于非主映射器不用于持久性，因此可变扩展现在将非主映射器排除在其仪器步骤之外。

    参考：[#4215](https://www.sqlalchemy.org/trac/ticket/4215)

### 引擎

+   **[引擎] [错误]**

    修复了连接池中可能存在连接但未调用所有“connect”事件处理程序的错误，如果先前的“connect”处理程序抛出异常；请注意，方言本身具有发出 SQL 的连接处理程序，例如设置事务隔离级别的处理程序，如果数据库处于不可用状态，则可能失败，但仍允许连接。如果任何连接处理程序失败，首先使连接无效。

    参考：[#4225](https://www.sqlalchemy.org/trac/ticket/4225)

### SQL

+   **[SQL] [错误]**

    修复了在版本 1.2.5 中从先前修复的[#4204](https://www.sqlalchemy.org/trac/ticket/4204)中发生的回归，调用`CTE.alias()`方法后引用自身的 CTE 将无法正确引用自身的问题。

    参考：[#4204](https://www.sqlalchemy.org/trac/ticket/4204)

### PostgreSQL

+   **[PostgreSQL] [功能]**

    添加了对在 PostgreSQL 表定义中使用“PARTITION BY”的支持，使用“postgresql_partition_by”。拉取请求由 Vsevolod Solovyov 提供。

### MSSQL

+   **[MSSQL] [错误]**

    调整了对 pyodbc 的 SQL Server 版本检测，只允许数字标记，过滤掉非整数，因为该方言使用此值进行元组-数字比较。无论如何，这通常对所有已知的 SQL Server/pyodbc 驱动程序都是正确的。

    参考：[#4227](https://www.sqlalchemy.org/trac/ticket/4227)

### oracle

+   **[oracle] [bug]**

    支持的最低 cx_Oracle 版本为 5.2（2015 年 6 月）。以前，该方言对版本 5.0 进行了断言，但从 1.2.2 开始，我们使用了一些直到 5.2 才出现的符号。

    参考：[#4211](https://www.sqlalchemy.org/trac/ticket/4211)

### misc

+   **[bug] [declarative]**

    移除了在调用`__table_args__`、`__mapper_args__`时发出的警告，这些方法被命名为`@declared_attr`方法，当从非映射的声明性 mixin 中调用时。直接调用这些方法是文档中指定的在映射类上覆盖这些方法时要使用的方法。对于常规属性名称，警告仍会发出。

    参考：[#4221](https://www.sqlalchemy.org/trac/ticket/4221)

### orm

+   **[orm] [bug]**

    修复了在与具有替代命名属性设置的非主要映射器一起使用`Mutable.associate_with()`或`Mutable.as_mutable()`时会产生属性错误的错误。由于非主要映射器不用于持久性，因此可变扩展现在将非主要映射器排除在其检测步骤之外。

    参考：[#4215](https://www.sqlalchemy.org/trac/ticket/4215)

### engine

+   **[engine] [bug]**

    修复了连接池中的错误，其中如果先前的“connect”处理程序抛出异常，则可能存在连接而未调用所有“connect”事件处理程序；请注意，方言本身具有发出 SQL 的连接处理程序，例如设置事务隔离的处理程序，如果数据库处于不可用状态，则可能失败，但仍允许连接。如果任何连接处理程序失败，现在首先使连接无效。

    参考：[#4225](https://www.sqlalchemy.org/trac/ticket/4225)

### sql

+   **[sql] [bug]**

    修复了在 1.2.5 版本中从先前对[#4204](https://www.sqlalchemy.org/trac/ticket/4204)的修复中发生的回归，其中在调用`CTE.alias()`方法后引用自身的 CTE 将无法正确引用自身。

    参考：[#4204](https://www.sqlalchemy.org/trac/ticket/4204)

### postgresql

+   **[postgresql] [feature]**

    在 PostgreSQL 表定义中添加了对“PARTITION BY”的支持，使用“postgresql_partition_by”。感谢 Vsevolod Solovyov 的拉取请求。

### mssql

+   **[mssql] [bug]**

    调整了对于 pyodbc 的 SQL Server 版本检测，只允许数字标记，过滤掉非整数，因为该方言对于此值进行元组数值比较。在任何情况下，这通常对所有已知的 SQL Server/pyodbc 驱动程序都是正确的。

    参考：[#4227](https://www.sqlalchemy.org/trac/ticket/4227)

### oracle

+   **[oracle] [bug]**

    支持的最低 cx_Oracle 版本是 5.2（2015 年 6 月）。之前，该方言对版本 5.0 进行了断言，但从 1.2.2 版本开始，我们使用了一些直到 5.2 版本才出现的符号。

    参考：[#4211](https://www.sqlalchemy.org/trac/ticket/4211)

### misc

+   **[bug] [declarative]**

    移除了一个警告，当调用`@declared_attr`方法命名为`__table_args__`，`__mapper_args__`时，当从非映射的声明性混合物中调用时会发出警告。直接调用这些方法是文档中建议的方法，当一个映射类覆盖其中一个这些方法时使用。对于常规属性名称仍然会发出警告。

    参考：[#4221](https://www.sqlalchemy.org/trac/ticket/4221)

## 1.2.5

发布日期：2018 年 3 月 6 日

### orm

+   **[orm] [feature]**

    添加了新功能`Query.only_return_tuples()`。导致`Query`对象无条件地返回键控元组对象，即使查询针对的是单个实体。拉取请求由 Eric Atkin 提供。

+   **[orm] [bug]**

    在新的“多态 selectin”加载中修复了一个错误，当从关系延迟加载器部分加载多态对象的选择时，导致加载中出现“空 IN”条件，在“IN”的“内联”形式中引发错误。

    参考：[#4199](https://www.sqlalchemy.org/trac/ticket/4199)

+   **[orm] [bug]**

    修复了 1.2 版本的回归问题，当一个包含`AliasedClass`对象的映射器选项，通常在使用`QueryableAttribute.of_type()`方法时，无法被 pickle。1.1 版本的行为是从路径中省略别名类对象，因此恢复了这种行为。

    参考：[#4209](https://www.sqlalchemy.org/trac/ticket/4209)

### sql

+   **[sql] [bug]**

    修复了:class:.`CTE`构造中的错误，与[#4204](https://www.sqlalchemy.org/trac/ticket/4204)类似，其中一个被别名化的`CTE`在“克隆”操作期间无法正确复制自身，这在 ORM 中经常发生，并且在使用`ClauseElement.params()`方法时也会发生。

    参考：[#4210](https://www.sqlalchemy.org/trac/ticket/4210)

+   **[sql] [bug]**

    修复了 CTE 渲染中的一个 bug，即将`CTE`转换为`Alias`时，如果在 FROM 子句中多次引用 CTE，则其“ctename AS aliasname”子句不会适当地渲染。

    参考：[#4204](https://www.sqlalchemy.org/trac/ticket/4204)

+   **[sql] [bug]**

    修复了新的“扩展 IN 参数”功能中的一个 bug，即值的绑定参数处理器根本不起作用，测试未能覆盖这个相当基本的情况，其中包括 ENUM 值无法工作。

    参考：[#4198](https://www.sqlalchemy.org/trac/ticket/4198)

### postgresql

+   **[postgresql] [bug] [py3k]**

    修复了 PostgreSQL COLLATE / ARRAY 调整中的一个 bug，首次引入于[#4006](https://www.sqlalchemy.org/trac/ticket/4006)，其中 Python 3.7 正则表达式中的新行为导致修复失败。

    此更改也已**回溯**至：1.1.18

    参考：[#4208](https://www.sqlalchemy.org/trac/ticket/4208)

### mysql

+   **[mysql] [bug]**

    MySQL 方言现在明确使用`SELECT @@version`向服务器查询版本，以确保我们正确获取版本信息。代理服务器如 MaxScale 干扰了传递给 DBAPI 的连接服务器版本值，因此这不再可靠。

    此更改也已**回溯**至：1.1.18

    参考：[#4205](https://www.sqlalchemy.org/trac/ticket/4205)

### orm

+   **[orm] [feature]**

    添加了新功能`Query.only_return_tuples()`。导致`Query`对象无条件返回键值元组对象，即使查询针对单个实体。感谢 Eric Atkin 提交的拉取请求。

+   **[orm] [bug]**

    修复了新的“多态选择”加载中的一个 bug，当从关系懒加载器部分加载多态对象的选择时，导致加载中出现“空 IN”条件，在“IN”的“内联”形式中引发错误。

    参考：[#4199](https://www.sqlalchemy.org/trac/ticket/4199)

+   **[orm] [bug]**

    修复了 1.2 版本中的一个问题，即包含`AliasedClass`对象的映射器选项无法被 pickle 化。1.1 版本的行为是从路径中省略别名类对象，因此恢复了这种行为。

    参考：[#4209](https://www.sqlalchemy.org/trac/ticket/4209)

### sql

+   **[sql] [bug]**

    修复了:class:.`CTE`构造中的错误，与[#4204](https://www.sqlalchemy.org/trac/ticket/4204)类似，其中一个被别名的`CTE`在“克隆”操作期间无法正确复制自身，这在 ORM 中经常发生，也在使用`ClauseElement.params()`方法时发生。

    参考：[#4210](https://www.sqlalchemy.org/trac/ticket/4210)

+   **[sql] [bug]**

    修复了在 CTE 渲染中的一个错误，其中一个`CTE`也被转换为一个`Alias`，如果在 FROM 子句中对 CTE 有多个引用，则其“ctename AS aliasname”子句不会被正确渲染。

    参考：[#4204](https://www.sqlalchemy.org/trac/ticket/4204)

+   **[sql] [bug]**

    修复了新的“扩展 IN 参数”功能中的错误，其中值的绑定参数处理器根本不起作用，测试未能覆盖这个非常基本的情况，其中包括 ENUM 值无法工作。

    参考：[#4198](https://www.sqlalchemy.org/trac/ticket/4198)

### postgresql

+   **[postgresql] [bug] [py3k]**

    修复了在 PostgreSQL COLLATE / ARRAY 调整中首次引入的错误，其中 Python 3.7 正则表达式中的新行为导致修复失败。

    此更改也被**回溯**到：1.1.18

    参考：[#4208](https://www.sqlalchemy.org/trac/ticket/4208)

### mysql

+   **[mysql] [bug]**

    MySQL 方言现在明确使用`SELECT @@version`向服务器查询服务器版本，以确保我们获得正确的版本信息。代理服务器如 MaxScale 干扰传递给 DBAPI 的 connection.server_version 值，因此这不再可靠。

    此更改也被**回溯**到：1.1.18

    参考：[#4205](https://www.sqlalchemy.org/trac/ticket/4205)

## 1.2.4

发布日期：2018 年 2 月 22 日

### orm

+   **[orm] [bug]**

    修复了 ORM 版本控制功能中的 1.2 回归错误，其中针对一个`select()`或`alias()`的映射，还使用了针对底层表的版本控制列，由于添加的检查导致失败，这是作为[#3673](https://www.sqlalchemy.org/trac/ticket/3673)的一部分。

    参考：[#4193](https://www.sqlalchemy.org/trac/ticket/4193)

### engine

+   **[engine] [bug]**

    由于来自[#4181](https://www.sqlalchemy.org/trac/ticket/4181)的修复导致的 1.2.3 中的回归错误，涉及`Engine`和`OptionEngine`的事件系统的更改没有考虑到事件的移除，当在类级别调用时会引发`AttributeError`。

    参考：[#4190](https://www.sqlalchemy.org/trac/ticket/4190)

### sql

+   **[sql] [bug]**

    修复了 CTE 表达式的 bug，当给定名称区分大小写或需要引号时，它们的名称或别名名称不会被引用。感谢 Eric Atkin 的拉取请求。

    参考：[#4197](https://www.sqlalchemy.org/trac/ticket/4197)

### orm

+   **[orm] [bug]**

    修复了 1.2 版本中 ORM 版本控制功能的回归错误，其中针对`select()`或`alias()`的映射，同时还使用了对基础表的版本控制列，由于[#3673](https://www.sqlalchemy.org/trac/ticket/3673)中添加的检查而失败。

    参考：[#4193](https://www.sqlalchemy.org/trac/ticket/4193)

### engine

+   **[engine] [bug]**

    由于来自[#4181](https://www.sqlalchemy.org/trac/ticket/4181)的修复导致的 1.2.3 中的回归错误，涉及`Engine`和`OptionEngine`的事件系统的更改没有考虑到事件的移除，当在类级别调用时会引发`AttributeError`。

    参考：[#4190](https://www.sqlalchemy.org/trac/ticket/4190)

### sql

+   **[sql] [bug]**

    修复了 CTE 表达式的 bug，当给定名称区分大小写或需要引号时，它们的名称或别名名称不会被引用。感谢 Eric Atkin 的拉取请求。

    参考：[#4197](https://www.sqlalchemy.org/trac/ticket/4197)

## 1.2.3

发布日期：2018 年 2 月 16 日

### orm

+   **[orm] [feature]**

    向`set_attribute()`函数添加了新参数`set_attribute.inititator`，允许从监听器函数接收的事件令牌传播到后续的设置事件。

+   **[orm] [bug]**

    修复了后更新功能中的问题，当父对象已被删除但相关对象尚未删除时会发出 UPDATE。这个问题已经存在很长时间，但自 1.2 版本开始，现在对后更新断言匹配的行数，这会引发错误。

    这个更改也被**回溯**到：1.1.16

    参考：[#4187](https://www.sqlalchemy.org/trac/ticket/4187)

+   **[orm] [bug]**

    修复了由于问题[#4116](https://www.sqlalchemy.org/trac/ticket/4116)修复引起的回归，影响版本 1.2.2 以及 1.1.15，导致在某些声明性混合/继承情况下错误计算`AssociationProxy`的“拥有类”为`NoneType`类，以及如果关联代理从未映射的类中访问。“找到所有者”的逻辑已被替换为一个深入的例程，通过搜索分配给类或子类的完整映射器层次结构来确定正确（我们希望）的匹配；如果找不到匹配项，则不会分配所有者。如果对未映射的实例使用代理，则现在会引发异常。

    此更改也**回溯**到：1.1.16

    参考：[#4185](https://www.sqlalchemy.org/trac/ticket/4185)

+   **[ORM] [错误]**

    修复了`Bundle`对象未正确报告由 bundle 表示的主要`Mapper`对象的 bug，如果有的话。这个问题的一个直接副作用是，新的 selectinload 加载器策略无法与水平分片扩展一起工作。

    参考：[#4175](https://www.sqlalchemy.org/trac/ticket/4175)

+   **[ORM] [错误]**

    修复了具体继承映射中的 bug，其中用户定义的属性（如镜像来自兄弟类的映射属性的混合属性）会被映射器在实例级别被覆盖为不可访问。此外，确保在映射器配置阶段不会在类级别隐式调用用户绑定的描述符。

    参考：[#4188](https://www.sqlalchemy.org/trac/ticket/4188)

+   **[ORM] [错误]**

    修复了`reconstructor()`事件助手应用于映射类的`__init__()`方法时不会被识别的 bug。

    参考：[#4178](https://www.sqlalchemy.org/trac/ticket/4178)

### 引擎

+   **[引擎] [错误]**

    修复了一个 bug，当使用`Engine.execution_options()`方法时，与类级别的`Engine`关联的事件会在类级别时被重复。为了实现这一点，半私有类`OptionEngine`不再直接接受类级别的事件，并将引发错误；该类仅从其父类`Engine`传播类级别的事件。实例级别的事件继续像以前一样工作。

    参考：[#4181](https://www.sqlalchemy.org/trac/ticket/4181)

+   **[引擎] [错误]**

    `URL`对象现在允许多次指定查询键，它们的值将被连接成一个列表。这是为了支持插件功能的特性，该功能在`CreateEnginePlugin`中有文档记录，文档中指出“plugin”可以多次传递。此外，插件名称可以通过新的`create_engine.plugins`参数在 URL 之外传递给`create_engine()`。

    参考：[#4170](https://www.sqlalchemy.org/trac/ticket/4170)

### sql

+   **[sql] [feature]**

    为`Enum`添加了对持久化枚举值的支持，而不是键，当使用 Python pep-435 风格的枚举对象时。用户提供一个可调用函数，该函数将返回要持久化的字符串值。这允许对非字符串值的枚举进行值持久化。感谢 Jon Snyder 提供的拉取请求。

    参考：[#3906](https://www.sqlalchemy.org/trac/ticket/3906)

+   **[sql] [bug]**

    修复了`Enum`类型在多个键引用相同值时无法正确处理枚举“别名”的错误，感谢 Daniel Knell 提供的拉取请求。

    参考：[#4180](https://www.sqlalchemy.org/trac/ticket/4180)

### postgresql

+   **[postgresql] [bug]**

    将“SSL SYSCALL error: Operation timed out”添加到了触发 psycopg2 驱动程序“断开连接”场景的消息列表中。感谢 André Cruz 提供的拉取请求。

    此更改也**回溯**到：1.1.16

+   **[postgresql] [bug]**

    将“TRUNCATE”添加到了 PostgreSQL 方言接受的关键字列表中，作为一个“autocommit”触发关键字。感谢 Jacob Hayes 提供的拉取请求。

    此更改也**回��**到：1.1.16

### sqlite

+   **[sqlite] [bug]**

    修复了当平台上既没有安装 pysqlite2 也没有安装 sqlite3 时引发的导入错误，使得引发 sqlite3 相关的导入错误，而不是实际的失败模式 pysqlite2。感谢 Robin 提供的拉取请求。

### oracle

+   **[oracle] [feature]**

    外键的 ON DELETE 选项现在是 Oracle 反射的一部分。Oracle 不支持 ON UPDATE 级联。感谢 Miroslav Shubernetskiy 提供的拉取请求。

+   **[oracle] [bug]**

    修复了 cx_Oracle 断开连接检测中的错误，用于 pre_ping 和其他功能，其中可能会引发一个包含数字错误代码的 DatabaseError; 以前我们在这种情况下没有检查断开连接代码。

    参考：[#4182](https://www.sqlalchemy.org/trac/ticket/4182)

### tests

+   **[tests] [bug]**

    在 1.2 版本中添加的一个测试，旨在确认 Python 2.7 的行为，结果只确认了 Python 2.7.8 的行为。Python bug＃8743 仍然影响 Python 2.7.7 及更早版本中的集合比较，因此涉及 AssociationSet 的相关测试不再适用于这些较旧的 Python 2.7 版本。

    参考：[#3265](https://www.sqlalchemy.org/trac/ticket/3265)

### 杂项

+   **[bug] [pool]**

    修复了一个相当严重的连接池错误，即在由于用户定义的`DisconnectionError`或由于 1.2 版本发布的“pre_ping”功能导致刷新后获取的连接，如果连接通过弱引用清理（例如前端对象被垃圾回收）返回到池中，则不会正确重置；弱引用仍将指向先前失效的 DBAPI 连接，而将重置操作错误地调用。这将导致日志中的堆栈跟踪以及将连接检入池中而不进行重置，这可能会导致锁定问题。

    此更改也已**回溯**至：1.1.16

    参考：[#4184](https://www.sqlalchemy.org/trac/ticket/4184)

### orm

+   **[orm] [feature]**

    在`set_attribute()`函数中添加了新参数`set_attribute.inititator`，允许从监听器函数接收到的事件令牌传播到后续的设置事件。

+   **[orm] [bug]**

    修复了后更新功能中的问题，当父对象已被删除但相关对象尚未删除时，会发出 UPDATE。这个问题已经存在很长时间，但由于 1.2 现在断言后更新匹配的行数，因此会引发错误。

    此更改也已**回溯**至：1.1.16

    参考：[#4187](https://www.sqlalchemy.org/trac/ticket/4187)

+   **[orm] [bug]**

    由于修复问题[#4116](https://www.sqlalchemy.org/trac/ticket/4116)导致的回归，影响版本 1.2.2 以及 1.1.15，导致在某些声明性混合/继承情况下以及如果关联代理从未映射的类中访问时，错误地计算`AssociationProxy`的“拥有类”为`NoneType`类。现在，“找到所有者”的逻辑已被一个深入的例程所取代，该例程通过搜索分配给类或子类的完整映射器层次结构来确定正确（我们希望）的匹配；如果找不到匹配项，则不会分配所有者。如果代理用于未映射的实例，则现在会引发异常。

    此更改也已**回溯**至：1.1.16

    参考：[#4185](https://www.sqlalchemy.org/trac/ticket/4185)

+   **[orm] [bug]**

    修复了 `Bundle` 对象未正确报告由 bundle 表示的主要 `Mapper` 对象（如果有）的 bug。此问题的直接副作用是新的 selectinload 加载器策略无法与水平分片扩展一起使用。

    参考：[#4175](https://www.sqlalchemy.org/trac/ticket/4175)

+   **[orm] [bug]**

    修复了具体继承映射中的 bug，其中用户定义的属性（例如反映与同级类的映射属性相同名称的混合属性）将被映射器覆盖为在实例级别不可访问。此外，确保在映射器配置阶段不会隐式调用用户绑定的描述符。

    参考：[#4188](https://www.sqlalchemy.org/trac/ticket/4188)

+   **[orm] [bug]**

    修复了当 `reconstructor()` 事件助手应用于映射类的 `__init__()` 方法时，它不会被识别的 bug。

    参考：[#4178](https://www.sqlalchemy.org/trac/ticket/4178)

### engine

+   **[engine] [bug]**

    修复了当使用 `Engine.execution_options()` 方法时与 `Engine` 关联的事件在类级别上会重复的 bug。为了实现这一点，半私有类 `OptionEngine` 不再直接接受类级别的事件，并将引发错误；该类仅从其父类 `Engine` 传播类级别的事件。实例级别的事件继续像以前一样工作。

    参考：[#4181](https://www.sqlalchemy.org/trac/ticket/4181)

+   **[engine] [bug]**

    `URL` 对象现在允许指定查询键多次，其值将被连接成一个列表。这是为了支持插件功能的特性所做的修改，该特性在 `CreateEnginePlugin` 中有详细说明，该文档指出“plugin”可以多次传递。此外，插件名称可以在使用新的 `create_engine.plugins` 参数之外通过 URL 传递给 `create_engine()`。

    参考：[#4170](https://www.sqlalchemy.org/trac/ticket/4170)

### sql

+   **[sql] [feature]**

    为`Enum`添加了支持，以持久化枚举的值，而不是键，当使用 Python pep-435 风格的枚举对象时。用户提供一个可调用函数，该函数将返回要持久化的字符串值。这允许对非字符串值的枚举也可以进行值持久化。感谢 Jon Snyder 的拉取请求。

    参考：[#3906](https://www.sqlalchemy.org/trac/ticket/3906)

+   **[sql] [bug]**

    修复了`Enum`类型无法正确处理枚举“别名”的 bug，当多个键引用相同值时。感谢 Daniel Knell 的拉取请求。

    参考：[#4180](https://www.sqlalchemy.org/trac/ticket/4180)

### postgresql

+   **[postgresql] [bug]**

    将“SSL SYSCALL error: Operation timed out”添加到触发 psycopg2 驱动程序“断开连接”场景的消息列表中。感谢 André Cruz 的拉取请求。

    此更改也**回溯**到：1.1.16

+   **[postgresql] [bug]**

    将“TRUNCATE”添加到 PostgreSQL 方言接受的关键字列表中，作为“自动提交”触发关键字。感谢 Jacob Hayes 的拉取请求。

    此更改也**回溯**到：1.1.16

### sqlite

+   **[sqlite] [bug]**

    修复了当平台既没有安装 pysqlite2 也没有安装 sqlite3 时引发的导入错误，使得引发与 sqlite3 相关的导入错误，而不是实际的失败模式 pysqlite2。感谢 Robin 的拉取请求。

### oracle

+   **[oracle] [feature]**

    外键的 ON DELETE 选项现在是 Oracle 反射的一部分。Oracle 不支持 ON UPDATE 级联。感谢 Miroslav Shubernetskiy 的拉取请求。

+   **[oracle] [bug]**

    修复了 cx_Oracle 断开连接检测中的错误，该错误由 pre_ping 和其他功能使用，可能会引发一个包含数字错误代码的 DatabaseError；以前我们在这种情况下没有检查断开连接代码。

    参考：[#4182](https://www.sqlalchemy.org/trac/ticket/4182)

### tests

+   **[tests] [bug]**

    在 1.2 中添加的一个测试，旨在确认 Python 2.7 行为，结果只确认了 Python 2.7.8 的行为。Python bug #8743 仍然影响 Python 2.7.7 及更早版本中的集合比较，因此涉及 AssociationSet 的相关测试不再适用于这些较旧的 Python 2.7 版本。

    参考：[#3265](https://www.sqlalchemy.org/trac/ticket/3265)

### misc

+   **[bug] [pool]**

    修复了一个相当严重的连接池错误，即在用户定义的`DisconnectionError`或由于 1.2 版本发布的“pre_ping”功能导致刷新后获取的连接，如果连接通过弱引用清理（例如，前端对象被垃圾回收）返回到池中，则不会正确重置；弱引用仍将指向先前失效的 DBAPI 连接，而将重置操作错误地调用在其上。这将导致日志中的堆栈跟踪和连接被检入池中而未被重置，这可能导致锁定问题。

    此更改也**回溯**到：1.1.16

    参考：[#4184](https://www.sqlalchemy.org/trac/ticket/4184)

## 1.2.2

发布日期：2018 年 1 月 24 日

### orm

+   **[orm] [bug]**

    修复了 1.2 版本关于新的 bulk_replace 事件的回归，其中当批量赋值将对象分配给新所有者时，反向引用将无法从先前所有者中删除对象。

    参考：[#4171](https://www.sqlalchemy.org/trac/ticket/4171)

### mysql

+   **[mysql] [bug]**

    为引用目的向 MySQL 方言添加了更多 MySQL 8.0 保留字。感谢 Riccardo Magliocchetti 的拉取请求。

### mssql

+   **[mssql] [bug]**

    将 ODBC 错误代码 10054 添加到作为 ODBC / MSSQL 服务器断开连接的错误代码列表中。

    参考：[#4164](https://www.sqlalchemy.org/trac/ticket/4164)

### oracle

+   **[oracle] [bug]**

    cx_Oracle 方言现在无条件地在使用 NVARCHAR2 数据类型时调用 setinputsizes()，在 SQLAlchemy 中对应于 sqltypes.Unicode()。根据 cx_Oracle 的作者，这允许在 Oracle 客户端中发生正确的转换，而不管 NLS_NCHAR_CHARACTERSET 的设置如何。

    参考：[#4163](https://www.sqlalchemy.org/trac/ticket/4163)

### orm

+   **[orm] [bug]**

    修复了 1.2 版本关于新的 bulk_replace 事件的回归，其中当批量赋值将对象分配给新所有者时，反向引用将无法从先前所有者中删除对象。

    参考：[#4171](https://www.sqlalchemy.org/trac/ticket/4171)

### mysql

+   **[mysql] [bug]**

    为引用目的向 MySQL 方言添加了更多 MySQL 8.0 保留字。感谢 Riccardo Magliocchetti 的拉取请求。

### mssql

+   **[mssql] [bug]**

    将 ODBC 错误代码 10054 添加到作为 ODBC / MSSQL 服务器断开连接的错误代码列表中。

    参考：[#4164](https://www.sqlalchemy.org/trac/ticket/4164)

### oracle

+   **[oracle] [bug]**

    cx_Oracle 方言现在无条件地在使用 NVARCHAR2 数据类型时调用 setinputsizes()，在 SQLAlchemy 中对应于 sqltypes.Unicode()。根据 cx_Oracle 的作者，这允许在 Oracle 客户端中发生正确的转换，而不管 NLS_NCHAR_CHARACTERSET 的设置如何。

    参考：[#4163](https://www.sqlalchemy.org/trac/ticket/4163)

## 1.2.1

发布日期：2018 年 1 月 15 日

### orm

+   **[orm] [bug]**

    修复了在嵌套或子事务回滚期间被清除的对象，该对象还在其主键发生变化时不会被正确地从会话中移除的错误，导致在使用会话时出现后续问题。

    这个更改也被**回溯**到：1.1.16

    参考：[#4151](https://www.sqlalchemy.org/trac/ticket/4151)

+   **[orm] [bug]**

    修复了 pickle 格式的 Load / _UnboundLoad 对象（例如加载器选项）的回归，其中`__setstate__()`为从旧格式接收的对象引发 UnboundLocalError，尽管尝试这样做。现在添加了测试以确保这样可以正常工作。

    参考：[#4159](https://www.sqlalchemy.org/trac/ticket/4159)

+   **[orm] [bug]**

    由于[#3954](https://www.sqlalchemy.org/trac/ticket/3954)中的新 lazyload 缓存方案引起的回归，使用 of_type 的加载器选项的查询将导致与 TypeError 一起失败的不相关路径的延迟加载。

    参考：[#4153](https://www.sqlalchemy.org/trac/ticket/4153)

+   **[orm] [bug]**

    修复了新的“selectin”关系加载程序中的错误，其中加载程序在加载多态对象集合时可能尝试加载不存在的关系，其中只有一些映射器包含该关系，通常在使用`PropComparator.of_type()`时。

    参考：[#4156](https://www.sqlalchemy.org/trac/ticket/4156)

### sql

+   **[sql] [bug]**

    修复了`Insert.values()`中的错误，其中在与`Column`对象一起使用“多值”格式作为键而不是字符串时会失败。感谢 Aubrey Stark-Toller 提供的拉取请求。

    这个更改也被**回溯**到：1.1.16

    参考：[#4162](https://www.sqlalchemy.org/trac/ticket/4162)

### mssql

+   **[mssql] [bug]**

    修复了 1.2 中修复的引用名称引用在[#3785](https://www.sqlalchemy.org/trac/ticket/3785)中破坏了 SQL Server 的回归，该引用明确不理解引号引用的排序名称。现在，混合大小写排序名称是否被引用或不被引用现在被推迟到方言级别的决定，以便每个方言可以直接准备这些标识符。

    参考：[#4154](https://www.sqlalchemy.org/trac/ticket/4154)

### oracle

+   **[oracle] [bug]**

    修复了从 cx_Oracle 方言中删除大多数 setinputsizes 规则对 TIMESTAMP 数据类型检索分数秒的影响的回归。

    参考：[#4157](https://www.sqlalchemy.org/trac/ticket/4157)

+   **[oracle] [bug]**

    修复了 Oracle 导入中的回归，其中缺少逗号导致未定义的符号存在。感谢 Miroslav Shubernetskiy 提供的拉取请求。

### 测试

+   **[tests] [bug]**

    从公共测试套件中删除了一个干扰第三方方言套件的特定于 Oracle 的要求规则。

+   **[tests] [bug]**

    添加了一个新的排除规则组 group_by_complex_expression，禁用使用“GROUP BY <expr>”的测试，这在至少两个第三方方言中似乎不可行。

### misc

+   **[bug] [ext]**

    修复了关联代理中的回归问题，原因是 [#3769](https://www.sqlalchemy.org/trac/ticket/3769)（允许链式 any() / has()）导致的，其中针对关联代理进行 contains() 操作链式化的形式（o2m 关系，关联代理(m2o 关系，m2o 关系)）会导致在链的最后一环重新应用 contains() 时引发错误。

    参考：[#4150](https://www.sqlalchemy.org/trac/ticket/4150)

### orm

+   **[orm] [bug]**

    修复了在嵌套或子事务回滚期间从会话中正确移除在其主键发生变化的对象时，对象不会被正确移除而导致后续使用会话时出现问题的错误。

    此更改也已**回溯**至：1.1.16

    参考：[#4151](https://www.sqlalchemy.org/trac/ticket/4151)

+   **[orm] [bug]**

    修复了 pickle 格式的 Load / _UnboundLoad 对象（例如加载器选项）的回归问题，其中 pickle 格式发生变化，而 `__setstate__()` 对于从旧格式接收的对象引发 UnboundLocalError，尽管尝试这样做。现在添加了测试以确保其正常工作。

    参考：[#4159](https://www.sqlalchemy.org/trac/ticket/4159)

+   **[orm] [bug]**

    修复了由 [#3954](https://www.sqlalchemy.org/trac/ticket/3954) 中新的延迟加载缓存方案引起的回归问题，其中使用带有 of_type 的 loader 选项的查询会导致与无关路径的延迟加载失败并引发 TypeError。

    参考：[#4153](https://www.sqlalchemy.org/trac/ticket/4153)

+   **[orm] [bug]**

    修复了新的“selectin”关系加载器中的错误，其中加载器在加载多态对象集合时可能尝试加载不存在的关系，通常在使用`PropComparator.of_type()`时会出现这种情况。

    参考：[#4156](https://www.sqlalchemy.org/trac/ticket/4156)

### sql

+   **[sql] [bug]**

    修复了`Insert.values()`中的错误，其中在使用“多值”格式与`Column`对象作为键而不是字符串时会失败。感谢 Aubrey Stark-Toller 提交的拉取请求。

    此更改也已**回溯**至：1.1.16

    参考：[#4162](https://www.sqlalchemy.org/trac/ticket/4162)

### mssql

+   **[mssql] [bug]**

    修复了 1.2 版本中的回归问题，即在[#3785](https://www.sqlalchemy.org/trac/ticket/3785)中修复的排序名称引号在 SQL Server 中出现问题，因为 SQL Server 明确不理解带引号的排序名称。现在，混合大小写排序名称是否带引号取决于方言级别的决定，以便每个方言可以直接准备这些标识符。

    参考：[#4154](https://www.sqlalchemy.org/trac/ticket/4154)

### Oracle

+   **[Oracle] [错误]**

    修复了一个回归问题，即从 cx_Oracle 方言中删除大多数 setinputsizes 规则影响了 TIMESTAMP 数据类型检索分数秒的能力。

    参考：[#4157](https://www.sqlalchemy.org/trac/ticket/4157)

+   **[Oracle] [错误]**

    修复了 Oracle 导入中的回归问题，其中缺少逗号导致出现未定义的符号。感谢 Miroslav Shubernetskiy 的拉取请求。

### 测试

+   **[测试] [错误]**

    从公共测试套件中删除了一个特定于 Oracle 的要求规则，该规则干扰了第三方方言套件。

+   **[测试] [错误]**

    添加了一个新的排除规则组 group_by_complex_expression，禁用了使用“GROUP BY <expr>”的测试，这似乎对至少两个第三方方言不可行。

### 杂项

+   **[错误] [扩展]**

    修复了关联代理中的回归问题，由于[#3769](https://www.sqlalchemy.org/trac/ticket/3769)（允许链式 any() / has()）导致的问题，其中对关联代理进行 contains()操作链式调用形式（o2m 关系，associationproxy(m2o 关系，m2o 关系)）会导致关于在链的最终链接上重新应用 contains()的错误。

    参考：[#4150](https://www.sqlalchemy.org/trac/ticket/4150)

## 1.2.0

发布日期：2017 年 12 月 27 日

### ORM

+   **[ORM] [特性]**

    向 ORM 的标识键元组添加了一个新的数据成员，称为“identity_token”。此令牌默认为 None，但可以被数据库分片方案用来区分内存中具有相同主键但来自不同数据库的对象。水平分片扩展将此令牌与分片标识符应用，从而允许主键在水平分片后端之间重复。

    另请参阅

    支持分片的标识键增强

    参考：[#4137](https://www.sqlalchemy.org/trac/ticket/4137)

+   **[ORM] [错误] [扩展]**

    修复了关联代理无意中将自身链接到`AliasedClass`对象的错误，如果首先使用`AliasedClass`作为父级调用，会导致后续使用时出错。

    此更改也**回溯**到���1.1.15

    参考：[#4116](https://www.sqlalchemy.org/trac/ticket/4116)

+   **[ORM] [错误]**

    修复了在`contains_eager()`查询选项中的错误，其中使用 `PropComparator.of_type()` 引用跨多个连接级别到子类的路径还需要提供“别名”参数，并且需要与同一子类型一起使用 `aliased()` 对象；此外，对于使用 `aliased()` 对象作为 `PropComparator.of_type()` 参数的子类，使用 `contains_eager()` 也将正确渲染。

    参考：[#4130](https://www.sqlalchemy.org/trac/ticket/4130)

+   **[orm] [bug]**

    `Query.exists()` 方法现在将在查询被渲染时禁用急加载器。先前，连接的急加载连接将被不必要地渲染，以及不必要地生成子查询急加载查询。新行为与 `Query.subquery()` 方法相匹配。

    参考：[#4032](https://www.sqlalchemy.org/trac/ticket/4032)

### orm 声明式

+   **[orm] [声明式] [bug]**

    修复了在刷新操作期间引用描述符时的错误，该描述符是基于 `AbstractConcreteBase` 的层次结构中的映射列或其他位置的关系。由于该属性未映射为映射器属性，因此会导致错误。如果类未在其映射器中包含“concrete=True”，则还可能出现类似的问题，例如 `AbstractConcreteBase` 添加的“type”列，但是此处的检查也应防止该方案引起问题。

    此更改也**回溯**到：1.1.15

    参考：[#4124](https://www.sqlalchemy.org/trac/ticket/4124)

### 引擎

+   **[engine] [feature]**

    `URL` 对象的“password”属性现在可以是任何用户定义或用户子类化的字符串对象，该对象对 Python `str()` 内置函数做出响应。传递的对象将作为数据成员 `URL.password_original` 维护，并在读取 `URL.password` 属性以生成字符串值时进行查询。

    参考：[#4089](https://www.sqlalchemy.org/trac/ticket/4089)

### sql

+   **[sql] [bug]**

    修复了如果参数是元组，则`ColumnDefault`的`__repr__`会失败的错误。感谢 Nicolas Caniart 提供的拉取请求。

    此更改也**回溯**到：1.1.15

    参考：[#4126](https://www.sqlalchemy.org/trac/ticket/4126)

+   **[sql] [bug]**

    重新设计了在 1.2.0b2 中引入的 startswith()，endswith()的新“autoescape”选项中的新“autoescape”功能，使其完全自动化；转义字符现在默认为斜杠`"/"`，并应用于百分号、下划线以及转义字符本身，以实现完全自动转义。也可以使用“escape”参数更改字符。

    另请参阅

    startswith()，endswith()的新“autoescape”选项

    参考：[#2694](https://www.sqlalchemy.org/trac/ticket/2694)

+   **[sql] [bug]**

    修复了`Table.tometadata()`方法无法正确适应不由简单列表达式组成的`Index`对象的错误，例如针对`text()`构造的索引，使用 SQL 表达式或`func`的索引等。现在该程序将表达式完全复制到新的`Index`对象中，同时将所有绑定到目标表的`Column`对象替换为目标表的对象。

    参考：[#4147](https://www.sqlalchemy.org/trac/ticket/4147)

+   **[sql] [bug]**

    将`ColumnElement`的“visit name”从“column”更改为“column_element”，这样当此元素被用作用户定义的 SQL 元素的基础时，不会被各种 SQL 遍历工具（通常由 ORM 使用）处理时假定其行为类似于绑定到表的`ColumnClause`。

    参考：[#4142](https://www.sqlalchemy.org/trac/ticket/4142)

+   **[sql] [bug] [ext]**

    修复了 `ARRAY` 数据类型中的问题，本质上与 [#3832](https://www.sqlalchemy.org/trac/ticket/3832) 的问题相同，只是不是一个回归，即在 `ARRAY` 上的列附加事件不会正确触发，从而干扰依赖于此的系统。这个问题破坏的一个关键用例是使用 mixins 声明使用 `MutableList.as_mutable()` 的列。

    参考：[#4141](https://www.sqlalchemy.org/trac/ticket/4141)

+   **[sql] [bug]**

    修复了新的“扩展绑定参数”功能中的 bug，即如果一个语句中使用了多个参数，则正则表达式将无法正确匹配参数名。

    参考：[#4140](https://www.sqlalchemy.org/trac/ticket/4140)

+   **[sql] [enhancement]**

    在 PostgreSQL、MySQL、MS SQL Server（以及不支持的 Sybase 方言中）实现了“DELETE..FROM”语法，类似于“UPDATE..FROM”的工作方式。引用多个表的 DELETE 语句将切换到“多表”模式，并呈现数据库理解的适当的“USING”或多表“FROM”子句。感谢 Pieter Mulder 的拉取请求。

    另请参阅

    DELETE 的多表条件支持

    参考：[#959](https://www.sqlalchemy.org/trac/ticket/959)

### postgresql

+   **[postgresql] [feature]**

    添加了新的`MONEY` 数据类型。感谢 Cleber J Santos 的拉取请求。

### mysql

+   **[mysql] [bug]**

    MySQL 5.7.20 现在警告使用 @tx_isolation 变量；现在执行版本检查并使用 @transaction_isolation 代替以防止此警告。

    此更改也**回溯**到：1.1.15

    参考：[#4120](https://www.sqlalchemy.org/trac/ticket/4120)

+   **[mysql] [bug]**

    修复了从问题 1.2.0b3 中的回归，其中“MariaDB”版本比较可能在某些特定的 MariaDB 版本字符串下在 Python 3 中失败。

    参考：[#4115](https://www.sqlalchemy.org/trac/ticket/4115)

### mssql

+   **[mssql] [bug]**

    修复了在 pyodbc 中 sqltypes.BINARY 和 sqltypes.VARBINARY 数据类型不包含正确的 bound-value 处理程序的 bug，这允许传递 pyodbc.NullParam 值以帮助 FreeTDS。

    参考：[#4121](https://www.sqlalchemy.org/trac/ticket/4121)

### oracle

+   **[oracle] [bug]**

    添加了一些额外规则，以完全处理使用 `asdecimal=True` 时，cx_Oracle 数值中的 `Decimal('Infinity')`、`Decimal('-Infinity')` 值。

    参考：[#4064](https://www.sqlalchemy.org/trac/ticket/4064)

### 杂项

+   **[misc] [feature]**

    在文档中添加了一个新的错误部分，其中包含关于常见错误消息的背景信息。SQLAlchemy 中的选定异常将在其字符串输出中包含指向此页面相关部分的链接。

+   **[enhancement] [ext]**

    添加了新方法`Result.with_post_criteria()`到烘焙查询系统，允许在从缓存中提取查询后进行非 SQL 修改转换。此方法可以与`ShardedQuery`一起使用，以设置分片标识符。`ShardedQuery`也已经修改，使其`ShardedQuery.get()`方法与`Result`的方法正确交互。

    参考：[#4135](https://www.sqlalchemy.org/trac/ticket/4135)

### orm

+   **[orm] [功能]**

    向 ORM 的身份映射中使用的身份键元组添加了一个名为“identity_token”的新数据成员。此令牌默认为 None，但可以被数据库分片方案用来区分来自不同数据库的具有相同主键的内存对象。水平分片扩展将此令牌与分片标识符结合起来，从而允许主键在水平分片后端之间重复。

    另请参见

    支持分片的身份键增强

    参考：[#4137](https://www.sqlalchemy.org/trac/ticket/4137)

+   **[orm] [错误] [扩展]**

    修复了关联代理在首次使用时会错误地将自身链接到`AliasedClass`对象的错误，如果首先将`AliasedClass`作为父级调用，将导致后续使用时出现错误。

    此更改也已**回溯**到：1.1.15

    参考：[#4116](https://www.sqlalchemy.org/trac/ticket/4116)

+   **[orm] [错误]**

    修复了`contains_eager()`查询选项中的错误，其中使用路径使用`PropComparator.of_type()`引用跨越多个级别的连接到子类的情况还需要提供“alias”参数，以避免向查询添加不需要的 FROM 子句；此外，跨子类使用`aliased()`对象作为`PropComparator.of_type()`参数使用`contains_eager()`也将正确呈现。

    参考：[#4130](https://www.sqlalchemy.org/trac/ticket/4130)

+   **[orm] [错误]**

    `Query.exists()` 方法现在在查询被渲染时会禁用急加载器。以前，连接急加载连接会被不必要地渲染，以及子查询急加载查询也会被不必要地生成。新行为与 `Query.subquery()` 方法相匹配。

    参考：[#4032](https://www.sqlalchemy.org/trac/ticket/4032)

### orm declarative

+   **[orm] [declarative] [bug]**

    修复了一个 bug，其中描述符，在基于 `AbstractConcreteBase` 的层次结构中的映射列或关系，在刷新操作期间会被引用，导致错误，因为该属性未映射为映射器属性。如果类未在其映射器中包含“concrete=True”，则类似问题也可能出现在其他属性上，比如 `AbstractConcreteBase` 添加的“type” 列，但此处的检查也应防止该场景引起问题。

    此更改也被 **回溯** 至：1.1.15

    参考：[#4124](https://www.sqlalchemy.org/trac/ticket/4124)

### engine

+   **[engine] [feature]**

    `URL` 对象的“password”属性现在可以是任何用户定义或用户子类化的字符串对象，该对象响应于 Python 的 `str()` 内置函数。传递的对象将保持为数据成员 `URL.password_original`，并在读取 `URL.password` 属性时进行查询以生成字符串值。

    参考：[#4089](https://www.sqlalchemy.org/trac/ticket/4089)

### sql

+   **[sql] [bug]**

    修复了 `__repr__` 的 `ColumnDefault` 在参数为元组时会失败的错误。感谢 Nicolas Caniart 提交的拉取请求。

    此更改也被 **回溯** 至：1.1.15

    参考：[#4126](https://www.sqlalchemy.org/trac/ticket/4126)

+   **[sql] [bug]**

    重新设计了在 1.2.0b2 中引入的 New “autoescape” option for startswith(), endswith() 中的新“autoescape”功能，现在完全自动化；转义字符现在默认为正斜杠 `"/"`，并应用于百分号、下划线，以及转义字符本身，实现完全自动转义。该字符也可以使用“escape”参数进行更改。

    另请参阅

    New “autoescape” option for startswith(), endswith()

    参考：[#2694](https://www.sqlalchemy.org/trac/ticket/2694)

+   **[sql] [bug]**

    修复了`Table.tometadata()`方法无法正确适应不仅由简单列表达式组成的`Index`对象，例如针对`text()`构造的索引，使用 SQL 表达式或 `func` 的索引等。现在，该例程将完全复制表达式到新的`Index`对象，同时将所有绑定到目标表的`Column`对象替换为目标表的对象。

    参考：[#4147](https://www.sqlalchemy.org/trac/ticket/4147)

+   **[sql] [bug]**

    将`ColumnElement`的“visit name”从“column”更改为“column_element”，这样当此元素用作用户定义的 SQL 元素的基础时，不会被假定为在被各种 SQL 遍历工具处理时表现得像绑定到表的`ColumnClause`，这些工具通常被 ORM 使用。

    参考：[#4142](https://www.sqlalchemy.org/trac/ticket/4142)

+   **[sql] [bug] [ext]**

    修复了 `ARRAY` 数据类型中的问题，本质上与 [#3832](https://www.sqlalchemy.org/trac/ticket/3832) 的问题相同，只是不是一个回归，即在 `ARRAY` 上的列附加事件不会正确触发，从而干扰依赖此功能的系统。这一问题破坏的一个关键用例是使用 mixins 声明使用 `MutableList.as_mutable()` 的列。

    参考：[#4141](https://www.sqlalchemy.org/trac/ticket/4141)

+   **[sql] [bug]**

    修复了新的“扩展绑定参数”功能中的 bug，即如果一个语句中使用了多个参数，则正则表达式将无法正确匹配参数名称。

    参考：[#4140](https://www.sqlalchemy.org/trac/ticket/4140)

+   **[sql] [enhancement]**

    实现了针对 PostgreSQL、MySQL、MS SQL Server（以及不支持的 Sybase 方言）的“DELETE..FROM”语法，类似于“UPDATE..FROM”工作方式。引用多个表的 DELETE 语句将切换到“多表”模式，并根据数据库理解的方式生成适当的“USING”或多表“FROM”子句。感谢 Pieter Mulder 的拉取请求。

    另请参阅

    支持多表条件的 DELETE

    参考：[#959](https://www.sqlalchemy.org/trac/ticket/959)

### postgresql

+   **[postgresql] [feature]**

    添加了新的`MONEY`数据类型。感谢 Cleber J Santos 的拉取请求。

### mysql

+   **[mysql] [bug]**

    MySQL 5.7.20 现在警告使用@tx_isolation 变量；现在执行版本检查并使用@transaction_isolation 代替以防止此警告。

    此更改也**回溯**到：1.1.15

    参考：[#4120](https://www.sqlalchemy.org/trac/ticket/4120)

+   **[mysql] [bug]**

    修复了从问题 1.2.0b3 中的回归，其中“MariaDB”版本比较可能在某些特定 MariaDB 版本字符串下在 Python 3 下失败。

    参考：[#4115](https://www.sqlalchemy.org/trac/ticket/4115)

### mssql

+   **[mssql] [bug]**

    修复了一个 bug，其中 sqltypes.BINARY 和 sqltypes.VARBINARY 数据类型不会为 pyodbc 包括正确的绑定值处理程序，这允许传递 pyodbc.NullParam 值，有助于 FreeTDS。

    参考：[#4121](https://www.sqlalchemy.org/trac/ticket/4121)

### oracle

+   **[oracle] [bug]**

    添加了一些额外规则，以完全处理`Decimal('Infinity')`，`Decimal('-Infinity')`值与使用`asdecimal=True`时的 cx_Oracle 数值。

    参考：[#4064](https://www.sqlalchemy.org/trac/ticket/4064)

### misc

+   **[misc] [feature]**

    在文档中添加了一个新的错误部分，介绍常见错误消息的背景。SQLAlchemy 中的选定异常将在其字符串输出中包含指向此页面相关部分的链接。

+   **[enhancement] [ext]**

    添加了新方法`Result.with_post_criteria()`到烘焙查询系统，允许在查询从缓存中拉取后进行非 SQL 修改转换。除其他外，此方法可与`ShardedQuery`一起使用以设置分片标识符。`ShardedQuery`也已修改，使其`ShardedQuery.get()`方法与`Result`的方法正确交互。

    参考：[#4135](https://www.sqlalchemy.org/trac/ticket/4135)

## 1.2.0b3

发布日期：2017 年 10 月 13 日

### orm

+   **[orm] [bug]**

    修复了一个 bug，其中 ORM 关系会警告存在冲突的同步目标（例如，两个关系都将写入同一列）对于继承层次结构中的兄弟类，在这种情况下，两个关系实际上永远不会在写入时发生冲突。

    此更改也**回溯**到：1.1.15

    参考：[#4078](https://www.sqlalchemy.org/trac/ticket/4078)

+   **[orm] [bug]**

    修复了一个 bug，其中针对单表继承实体使用相关选择会导致外部查询无法正确呈现，因为调整单一继承鉴别器条件不适当地重新应用于外部查询。

    此��改也**回溯**到：1.1.15

    参考：[#4103](https://www.sqlalchemy.org/trac/ticket/4103)

+   **[orm] [bug]**

    在`Session.merge()`中修复了一个错误，与[#4030](https://www.sqlalchemy.org/trac/ticket/4030)类似，其中对于标识映射中的目标对象的内部检查，如果在合并过程实际检索对象之前立即被垃圾回收，可能会导致错误。

    此更改也**回溯**到：1.1.14

    参考：[#4069](https://www.sqlalchemy.org/trac/ticket/4069)

+   **[orm] [bug]**

    修复了一个错误，即如果从使用连接式急加载加载的关系扩展，则不会识别`undefer_group()`选项。此外，由于该错误导致执行过多的工作，因此在结果集列的初始计算中，Python 函数调用次数也提高了 20%，这与[#3915](https://www.sqlalchemy.org/trac/ticket/3915)的连接急加载改进相辅相成。

    此更改也**回溯**到：1.1.14

    参考：[#4048](https://www.sqlalchemy.org/trac/ticket/4048)

+   **[orm] [bug]**

    修复了一个错误，在`Session.merge()`中，如果集合中的对象的主键属性设置为`None`，而该属性通常是自动递增的键，则在内部去重过程的一部分中，这些对象将被视为数据库持久化键，导致实际上只有一个对象被插入到数据库中。

    此更改也**回溯**到：1.1.14

    参考：[#4056](https://www.sqlalchemy.org/trac/ticket/4056)

+   **[orm] [bug]**

    当针对不是针对`MapperProperty`的属性（如关联代理）使用`synonym()`时，会引发`InvalidRequestError`。以前，尝试定位不存在的属性会导致递归溢出。

    此更改也**回溯**到：1.1.14

    参考：[#4067](https://www.sqlalchemy.org/trac/ticket/4067)

+   **[orm] [bug]**

    修复了 1.2.0b1 中引入的回归问题，由于[#3934](https://www.sqlalchemy.org/trac/ticket/3934)，如果回滚失败（目标问题是当 MySQL 丢失 SAVEPOINT 时），`Session`将无法“停用”事务。这将导致随后对`Session.rollback()`的调用再次引发错误，而不是完成并将`Session`恢复为活动状态。

    参考：[#4050](https://www.sqlalchemy.org/trac/ticket/4050)

+   **[orm] [bug]**

    修复了`make_transient_to_detached()`函数会使目标对象上的所有属性过期的问题，包括“延迟加载”属性，这会导致下一次刷新时属性被取消延迟加载，从而导致属性意外加载。

    参考：[#4084](https://www.sqlalchemy.org/trac/ticket/4084)

+   **[orm] [bug]**

    修复了涉及 delete-orphan 级联的 bug，其中相关项目在父对象成为会话的一部分之前成为孤儿，仍然被跟踪为进入孤儿状态，导致其从会话中被清除而不是被刷新。

    注意

    这个修复在 1.2.0b3 发布期间被错误地合并，并且**没有被添加到更改日志**中。这个更改日志注释是作为版本 1.2.13 的一部分事后添加的。

    参考：[#4040](https://www.sqlalchemy.org/trac/ticket/4040)

+   **[orm] [bug]**

    修复了“selectin”多态加载，使用单独的 IN 查询加载子类中的错误，该错误阻止了多级类层次结构中“selectin”和“inline”设置按预期交互。

    参考：[#4026](https://www.sqlalchemy.org/trac/ticket/4026)

+   **[orm] [bug]**

    移除了当映射器和加载策略使用的 LRU 缓存达到阈值时发出的警告；最初这个警告的目的是防止生成过多的缓存键，但后来基本上成为“创建许多引擎”反模式的检查。虽然这仍然是一个反模式，但测试套件中既为每个测试创建一个引擎又在所有警告上引发的存在将是一个不便；对于这个警告，这些测试套件改变其架构并不是必要的（尽管每个测试一个引擎的套件总是更好）。

    参考：[#4071](https://www.sqlalchemy.org/trac/ticket/4071)

+   **[orm] [bug]**

    修复了一个回归问题，即在与延迟加载关系选项一起使用`undefer_group()`选项时，由于 1.2 版本中作为[#3954](https://www.sqlalchemy.org/trac/ticket/3954)的一部分添加的 SQL 缓存键生成中的错误，会导致属性错误。

    参考：[#4049](https://www.sqlalchemy.org/trac/ticket/4049)

+   **[orm] [bug]**

    修改了在[#3366](https://www.sqlalchemy.org/trac/ticket/3366)中对 ORM 更新/删除评估器所做的更改，如果更新或删除中存在未映射的列表达式，并且评估器可以将其名称与目标类的映射列匹配，将发出警告，而不是引发 UnevaluatableError。这本质上是 1.2 版本之前的行为，目的是允许正在依赖此模式的应用程序进行迁移。但是，如果给定的属性名称无法与映射器的列匹配，仍会引发 UnevaluatableError，这是在[#3366](https://www.sqlalchemy.org/trac/ticket/3366)中修复的问题。

    参考：[#4073](https://www.sqlalchemy.org/trac/ticket/4073)

### orm declarative

+   **[orm] [declarative] [bug]**

    如果子类尝试覆盖在父类上声明的属性，并使用`@declared_attr.cascading`，则会发出警告，指出覆盖的属性将被忽略。这种用例无法在更深层次的子类中得到完全支持，因此为了一致性，无论覆盖属性如何，“级联”都会一直被遵守。

    参考：[#4091](https://www.sqlalchemy.org/trac/ticket/4091)

+   **[orm] [declarative] [bug]**

    如果使用`@declared_attr.cascading`属性与特殊的声明名称（如`__tablename__`）一起使用，则会发出警告，因为这没有效果。

    参考：[#4092](https://www.sqlalchemy.org/trac/ticket/4092)

### engine

+   **[engine] [feature]**

    向`ResultProxy`添加了`__next__()`和`next()`方法，以便直接在对象上使用`next()`内置函数。`ResultProxy`长期以来已经有一个`__iter__()`方法，允许它响应`iter()`内置函数。`__iter__()`的实现未更改，因为性能测试表明，使用带有`StopIteration`的`__next__()`方法进行迭代在 Python 2.7 和 3.6 中都要慢大约 20%。

    参考：[#4077](https://www.sqlalchemy.org/trac/ticket/4077)

+   **[engine] [bug]**

    对`Pool`和`Connection`进行了一些调整，使得在`pool.Empty`、`AttributeError`异常捕获下不会运行恢复逻辑，因为当恢复操作本身失败时，Python 3 会创建一个误导性的堆栈跟踪，将`Empty` / `AttributeError`作为原因，而实际上这些异常捕获是控制流的一部分。

    参考文献：[#4028](https://www.sqlalchemy.org/trac/ticket/4028)

### sql

+   **[sql] [bug]**

    修复了最近添加的 `ColumnOperators.any_()` 和 `ColumnOperators.all_()` 方法在作为方法调用时不起作用的错误，与使用独立函数 `any_()` 和 `all_()` 相对，还为这些相对不直观的 SQL 运算符添加了文档示例。

    此更改还 **反向移植** 至：1.1.15

    参考文献：[#4093](https://www.sqlalchemy.org/trac/ticket/4093)

+   **[sql] [bug]**

    添加了一个新方法 `DefaultExecutionContext.get_current_parameters()`，该方法在函数型默认值生成器中使用，以便检索传递给语句的当前参数。新函数与 `DefaultExecutionContext.current_parameters` 属性不同之处在于，它还提供了参数的可选分组，这些参数对应于多值“插入”构造。以前无法识别与函数调用相关的参数子集。

    请参阅

    用于具有上下文默认生成器的多值插入的参数辅助程序

    上下文敏感的默认函数

    参考文献：[#4075](https://www.sqlalchemy.org/trac/ticket/4075)

+   **[sql] [bug]**

    修复了新的 SQL 注释功能中的错误，当使用 `Table.tometadata()` 时，表和列注释不会被复制。

    参考文献：[#4087](https://www.sqlalchemy.org/trac/ticket/4087)

+   **[sql] [bug]**

    在版本 1.1 中，`Boolean` 类型存在问题，即通过 `bool()` 进行布尔强制转换会发生在不支持“原生布尔”的后端，但不会发生在原生布尔后端，这意味着字符串 `"0"` 现在表现不一致。经过一次投票，达成共识，即非布尔值应该引发错误，特别是在字符串 `"0"` 的模棱两可情况下；因此，如果传入值不在范围 `None, True, False, 1, 0` 内，`Boolean` 数据类型现在将引发 `ValueError`。

    参见

    布尔数据类型现在强制执行严格的 True/False/None 值

    参考：[#4102](https://www.sqlalchemy.org/trac/ticket/4102)

+   **[sql] [bug]**

    优化了`Operators.op()`的行为，使得在所有情况下，如果`Operators.op.is_comparison`标志设置为 True，则生成表达式的返回类型将是`Boolean`，如果标志为 False，则生成表达式的返回类型将与左侧表达式的类型相同，这是其他运算符的典型默认行为。还添加了一个新参数`Operators.op.return_type`以及一个辅助方法`Operators.bool_op()`。

    参见

    自定义运算符的类型行为已经保持一致

    参考：[#4063](https://www.sqlalchemy.org/trac/ticket/4063)

+   **[sql] [bug]**

    对`Enum`、`Interval`和`Boolean`类型进行了内部优化，现在它们都扩展了一个通用的 mixin `Emulated`，表示提供了对数据库本地类型的 Python 端模拟，在使用支持的后端时切换到数据库本地类型。直接使用 PostgreSQL `INTERVAL` 类型现在将包括正确的类型强制转换规则，对于也适用于 `Interval` 的 SQL 表达式（例如将日期添加到间隔会产生日期时间）。

    参考：[#4088](https://www.sqlalchemy.org/trac/ticket/4088)

### postgresql

+   **[postgresql] [feature]**

    向 psycopg2 方言添加了一个新标志`use_batch_mode`。当`Engine`调用`cursor.executemany()`时，此标志启用了 psycopg2 的`psycopg2.extras.execute_batch`扩展。此扩展在批量运行 INSERT 语句时提供了关键的性能提升，性能提升超过一个数量级。该标志默认为 False，因为目前被认为是实验性的。

    另请参阅

    批处理模式/快速执行助手的支持

    参考：[#4109](https://www.sqlalchemy.org/trac/ticket/4109)

+   **[postgresql] [bug]**

    针对与 COLLATE 一起的`ARRAY`类进行了进一步修复，因为在[#4006](https://www.sqlalchemy.org/trac/ticket/4006)中进行的修复未能适应多维数组。

    此更改也**回溯**至：1.1.15

    参考：[#4006](https://www.sqlalchemy.org/trac/ticket/4006)

+   **[postgresql] [bug]**

    修复了`array_agg`函数中的错误，其中传递一个已经是`ARRAY`类型的参数，例如 PostgreSQL `array`构造，将产生`ValueError`，因为函数尝试嵌套数组。

    此更改也**回溯**至：1.1.15

    参考：[#4107](https://www.sqlalchemy.org/trac/ticket/4107)

+   **[postgresql] [bug]**

    修复了 PostgreSQL `Insert.on_conflict_do_update()`中的错误，该错误将阻止将插入语句用作 CTE，例如通过`Insert.cte()`在另一个语句中使用。

    此更改也**回溯**至：1.1.15

    参考：[#4074](https://www.sqlalchemy.org/trac/ticket/4074)

+   **[postgresql] [bug]**

    修复了 pg8000 驱动程序在使用带有模式名称的`MetaData.reflect()`时会失败的错误，因为模式名称将作为“quoted_name”对象发送，该对象是一个字符串子类，pg8000 不识别。连接时将 quoted_name 类型添加到 pg8000 的 py_types 集合中。

    参考：[#4041](https://www.sqlalchemy.org/trac/ticket/4041)

+   **[postgresql] [bug]**

    为 pg8000 驱动程序启用了 UUID 支持，支持此数据类型的本机 Python uuid 往返。但是仍不���持 UUID 数组。

    参考：[#4016](https://www.sqlalchemy.org/trac/ticket/4016)

### mysql

+   **[mysql] [bug]**

    当检测到 MariaDB 10.2.8 或更早版本的 10.2 系列时，会发出警告，因为这些版本中的 CHECK 约束存在重大问题，这些问题在 10.2.9 中已解决。

    请注意，此更改日志消息并未随 SQLAlchemy 1.2.0b3 发布，而是事后添加的。

    此更改也被**回溯**到：1.1.15

    参考：[#4097](https://www.sqlalchemy.org/trac/ticket/4097)

+   **[mysql] [bug]**

    修复了在 MariaDB 10.2 系列中，由于语法更改，导致 CURRENT_TIMESTAMP 无法正确反映的问题，现在该函数表示为 `current_timestamp()`。

    此更改也被**回溯**到：1.1.15

    参考：[#4096](https://www.sqlalchemy.org/trac/ticket/4096)

+   **[mysql] [bug]**

    MariaDB 10.2 现在支持 CHECK 约束（警告：由于上游问题，请使用版本 10.2.9 或更高版本，详见 [#4097](https://www.sqlalchemy.org/trac/ticket/4097)）。反射现在在 `SHOW CREATE TABLE` 输出中考虑这些 CHECK 约束。

    此更改也被**回溯**到：1.1.15

    参考：[#4098](https://www.sqlalchemy.org/trac/ticket/4098)

+   **[mysql] [bug]**

    将新的 MySQL INSERT..ON DUPLICATE KEY UPDATE 结构的 `.values` 属性更名为 `.inserted`，因为 `Insert` 已经有一个名为 `Insert.values()` 的方法。`.inserted` 属性最终呈现 MySQL 的 `VALUES()` 函数。

    参考：[#4072](https://www.sqlalchemy.org/trac/ticket/4072)

### sqlite

+   **[sqlite] [bug]**

    修复了 SQLite CHECK 约束反射失败的 bug，如果引用的表位于远程模式中，例如 SQLite 中由 ATTACH 引用的远程数据库。

    此更改也被**回溯**到：1.1.15

    参考：[#4099](https://www.sqlalchemy.org/trac/ticket/4099)

### mssql

+   **[mssql] [feature]**

    添加了一个新的 `TIMESTAMP` 数据类型，对 SQL Server 而言，它正确地像二进制数据类型而不是 datetime 类型，因为 SQL Server 在这里违反了 SQL 标准。还添加了 `ROWVERSION`，因为 SQL Server 中的`TIMESTAMP`类型已被弃用，改用 ROWVERSION。

    参考：[#4086](https://www.sqlalchemy.org/trac/ticket/4086)

+   **[mssql] [feature]**

    为 PyODBC 和 pymssql 方言添加了对“AUTOCOMMIT”隔离级别的支持，通过 `Connection.execution_options()` 来建立，这个隔离级别在底层连接对象上设置适当的 DBAPI 特定标志。

    参考：[#4058](https://www.sqlalchemy.org/trac/ticket/4058)

+   **[mssql] [bug]**

    为 SQL Server 的 PyODBC 方言添加了一整套“连接关闭”异常代码，包括‘08S01’、‘01002’、‘08003’、‘08007’、‘08S02’、‘08001’、‘HYT00’、‘HY010’。以前只覆盖了‘08S01’。

    此更改也被**回溯**到：1.1.15

    参考：[#4095](https://www.sqlalchemy.org/trac/ticket/4095)

+   **[mssql] [bug]**

    SQL Server 支持 SQLAlchemy 称之为“本地布尔”的 BIT 类型，因为此类型仅接受 0 或 1，而 DBAPI 将其值返回为 True/False。因此，SQL Server 方言现在启用了“本地布尔”支持，即不为`Boolean`数据类型生成 CHECK 约束。与其他本地布尔的唯一区别是没有“true” / “false”常量，因此这里仍然呈现为“1”和“0”。

    参考：[#4061](https://www.sqlalchemy.org/trac/ticket/4061)

+   **[mssql] [bug]**

    修复了 pymssql 方言中 SQL 文本中的百分号，例如在模数表达式或文字值中使用的情况，不会加倍，这似乎是 pymssql 所期望的。尽管 pymssql DBAPI 使用“pyformat”参数样式，该样式认为百分号是重要的。

    参考：[#4057](https://www.sqlalchemy.org/trac/ticket/4057)

+   **[mssql] [bug]**

    修复了 SQL Server 方言在反射自引用外键约束时可能从多个模式中提取列的错误，如果多个模式包含相同名称的约束针对相同名称的表。

    参考：[#4060](https://www.sqlalchemy.org/trac/ticket/4060)

+   **[mssql] [bug] [orm]**

    对于特定于“RETURNING”的方言的“rowcount 支持”添加了一个新类，当在使用时，例如在 SQL Server 上看起来像“OUTPUT inserted”时，PyODBC 后端无法在 OUTPUT 生效时给我们提供 UPDATE 或 DELETE 语句的 rowcount。这主要影响 ORM，当刷新正在更新包含服务器计算值的行时，如果后端未返回预期的行数，则会引发错误。PyODBC 现在声明支持 rowcount，除非存在 OUTPUT.inserted，ORM 在刷新期间会考虑是否寻找 rowcount。

    参考：[#4062](https://www.sqlalchemy.org/trac/ticket/4062)

+   **[mssql] [bug] [orm]**

    为 pymssql 方言启用了“sane_rowcount”标志，表示 DBAPI 现在从 UPDATE 或 DELETE 语句中报告受影响的行数。这主要影响 ORM 版本功能，因为现在它可以验证目标版本上受影响的行数。

+   **[mssql] [bug]**

    添加了一个规则到 SQL Server 索引反射中，忽略所谓的在未指定聚集索引的表上隐式存在的“堆”索引。

    参考：[#4059](https://www.sqlalchemy.org/trac/ticket/4059)

### oracle

+   **[oracle] [性能] [bug] [py2k]**

    由于修复[#3937](https://www.sqlalchemy.org/trac/ticket/3937)导致的性能回归已修复，因为 cx_Oracle 从版本 5.3 开始从其命名空间中删除了`.UNICODE`符号，这被解释为 cx_Oracle 的“WITH_UNICODE”模式被无条件打开，从而在 SQLAlchemy 端调用函数将所有字符串无条件转换为 unicode 并导致性能影响。实际上，根据 cx_Oracle 的作者，“WITH_UNICODE”模式自 5.1 起已被完全移除，因此如果在 Python 2 下检测到 cx_Oracle 5.1 或更高版本，则不再需要昂贵的 unicode 转换函数，并且如果检测到 cx_Oracle 5.1 或更高版本，则会禁用这些函数。在[#3937](https://www.sqlalchemy.org/trac/ticket/3937)中删除的针对“WITH_UNICODE”模式的警告也已恢复。

    此更改也**回溯**到：1.1.13，1.0.19

    参考：[#4035](https://www.sqlalchemy.org/trac/ticket/4035)

+   **[oracle] [bug]**

    使用 cx_Oracle 实现了对 Oracle 值“无穷大”的部分支持，仅使用 Python 浮点值，例如`float("inf")`。目前，cx_Oracle DBAPI 驱动程序尚未实现对 Decimal 的��持。

    参考：[#4064](https://www.sqlalchemy.org/trac/ticket/4064)

+   **[oracle] [bug]**

    cx_Oracle 方言已经重构和现代化，以利用在旧的 4.x 系列 cx_Oracle 中不存在的新模式。其中最重要的变化涉及类型转换，主要是关于数字/浮点和 LOB 数据类型，更有效地利用 cx_Oracle 类型处理挂钩简化了绑定参数和结果数据的处理方式。

    另请参阅

    cx_Oracle 方言，类型系统的重大重构

+   **[oracle] [bug]**

    对于所有版本的 cx_Oracle，cx_Oracle 的两阶段支持已完全移除，而在 1.2.0b1 中，此更改仅对 cx_Oracle 的 6.x 系列生效。这个功能在任何版本的 cx_Oracle 中都从未正常工作过，在 cx_Oracle 6.x 中，SQLAlchemy 依赖的 API 被移除。

    另请参阅

    cx_Oracle 方言，类型系统的重大重构

    参考：[#3997](https://www.sqlalchemy.org/trac/ticket/3997)

+   **[oracle] [bug]**

    当使用`Insert.returning()`与 cx_Oracle 后端时，结果集中的列键现在使用正确的列名/标签，与所有其他方言一样。以前，这些列名为`ret_nnn`。

    另请参阅

    cx_Oracle 方言，类型系统的重大重构

+   **[oracle] [bug]**

    cx_Oracle 方言的几个参数现在已被弃用且不会产生任何效果：`auto_setinputsizes`，`exclude_setinputsizes`，`allow_twophase`。

    另请参阅

    对 cx_Oracle 方言、类型系统进行了重大重构

+   **[oracle] [bug]**

    修复了在 Oracle 下反映出的带有“column DESC”表达式的索引不会返回的 bug，如果表也没有主键，这是由于逻辑尝试过滤掉 Oracle 隐式添加到主键列上的索引所导致的。

    参考：[#4042](https://www.sqlalchemy.org/trac/ticket/4042)

+   **[oracle] [bug]**

    修复了由 cx_Oracle 6.0 引起的更多回归问题；目前，用户唯一的行为变化是断开连接检测现在除了检测 cx_Oracle.InterfaceError 外还检测 cx_Oracle.DatabaseError，因为这种行为似乎已经改变。关于数值精度和无法关闭连接的其他问题仍在上游 cx_Oracle 问题跟踪器中等待处理。

    参考：[#4045](https://www.sqlalchemy.org/trac/ticket/4045)

+   **[oracle] [bug]**

    修复了 Oracle 8 中“非 ANSI”连接模式不会向使用`=`运算符以外的运算符的表达式添加`(+)`运算符的 bug。`(+)`需要添加到右侧的所有列。

    参考：[#4076](https://www.sqlalchemy.org/trac/ticket/4076)

### orm

+   **[orm] [bug]**

    修复了 ORM 关系在继承层次结构中的兄弟类中可能会发出警告，提示存在同步目标冲突的 bug（例如，两个关系都写入同一列），而这两个关系实际上在写入时永远不会发生冲突。

    此更改也**回溯**到：1.1.15

    参考：[#4078](https://www.sqlalchemy.org/trac/ticket/4078)

+   **[orm] [bug]**

    修复了针对单表继承实体使用的相关查询在外部查询中无法正确呈现的 bug，因为单一继承鉴别器条件的调整不当地重新应用到外部查询中。

    此更改也**回溯**到：1.1.15

    参考：[#4103](https://www.sqlalchemy.org/trac/ticket/4103)

+   **[orm] [bug]**

    修复了`Session.merge()`中的 bug，与[#4030](https://www.sqlalchemy.org/trac/ticket/4030)类似，其中对于标识映射中的目标对象的内部检查，如果在合并过程实际检索对象之前立即被垃圾回收，可能会导致错误。

    此更改也**回溯**到：1.1.14

    参考：[#4069](https://www.sqlalchemy.org/trac/ticket/4069)

+   **[orm] [bug]**

    修复了一个 bug，当`undefer_group()`选项不被识别时，如果它是从使用联合急加载加载的关系扩展的。此外，由于该错误导致额外的工作被执行，Python 函数调用计数也在结果集列的初始计算中提高了 20%，这与[#3915](https://www.sqlalchemy.org/trac/ticket/3915)的联合急加载改进相辅相成。

    这个改变也被**回溯到**了：1.1.14

    参考文献：[#4048](https://www.sqlalchemy.org/trac/ticket/4048)

+   **[orm] [bug]**

    修复了`Session.merge()`中的一个 bug，当集合中的对象的主键属性设置为`None`时，通常是自动增量的键，则会将其视为数据库持久化键的一部分，用于内部去重处理过程，导致只有一个对象实际上被插入数据库。

    这个改变也被**回溯到**了：1.1.14

    参考文献：[#4056](https://www.sqlalchemy.org/trac/ticket/4056)

+   **[orm] [bug]**

    当对一个不是针对`MapperProperty`的属性（例如关联代理）使用`synonym()`时，会引发`InvalidRequestError`。以前，尝试定位不存在的属性时会导致递归溢出。

    这个改变也被**回溯到**了：1.1.14

    参考文献：[#4067](https://www.sqlalchemy.org/trac/ticket/4067)

+   **[orm] [bug]**

    由于[#3934](https://www.sqlalchemy.org/trac/ticket/3934)引入的 1.2.0b1 中的回归已修复，即使回滚失败（目标问题是当 MySQL 丢失 SAVEPOINT 时），`Session`也不会“停用”事务。这将导致对`Session.rollback()`的后续调用再次引发错误，而不是完成并将`Session`带回 ACTIVE 状态。

    参考文献：[#4050](https://www.sqlalchemy.org/trac/ticket/4050)

+   **[orm] [bug]**

    修复了`make_transient_to_detached()`函数的问题，该函数会使目标对象上的所有属性过期，包括“延迟加载”属性，导致属性在下一次刷新时被取消延迟加载，从而导致属性意外加载。

    参考文献：[#4084](https://www.sqlalchemy.org/trac/ticket/4084)

+   **[orm] [bug]**

    修复了涉及删除孤立级联的错误，其中相关项目在父对象成为会话的一部分之前变为孤立状态，但仍然被跟踪为进入孤立状态，结果是它被从会话中删除而不是被刷新。

    注意

    这个修复在 1.2.0b3 版本中被无意中合并，并且在那个时候**没有被添加到更改日志**中。这个更改日志记录是作为 1.2.13 版本的一部分进行了补充的。

    参考：[#4040](https://www.sqlalchemy.org/trac/ticket/4040)

+   **[orm] [bug]**

    修复了“selectin”多态加载，使用单独的 IN 查询加载子类中的错误，该错误阻止了多级类层次结构中的“selectin”和“inline”设置按预期进行交互。

    参考：[#4026](https://www.sqlalchemy.org/trac/ticket/4026)

+   **[orm] [bug]**

    移除了当映射器和加载器策略所使用的 LRU 缓存达到阈值时发出的警告；最初该警告的目的是防止生成过多的缓存键，但后来基本上成为了对“创建许多引擎”反模式的检查。尽管这仍然是一个反模式，但存在同时为每个测试创建一个引擎并且在所有警告上引发的测试套件将是一个不便；这种测试套件不应该因为这个警告而改变它们的架构（尽管每个测试一个引擎的套件总是更好）。

    参考：[#4071](https://www.sqlalchemy.org/trac/ticket/4071)

+   **[orm] [bug]**

    修复了一个回归，其中在与延迟加载关系选项结合使用`undefer_group()`选项时，由于在 1.2 中作为[#3954](https://www.sqlalchemy.org/trac/ticket/3954)的一部分添加的 SQL 缓存键生成中存在错误，导致属性错误。

    参考：[#4049](https://www.sqlalchemy.org/trac/ticket/4049)

+   **[orm] [bug]**

    修改了在[#3366](https://www.sqlalchemy.org/trac/ticket/3366)中对 ORM 更新/删除评估器所做的更改，以便如果更新或删除中存在未映射的列表达式，并且评估器可以将其名称与目标类的映射列相匹配，则发出警告，而不是引发 UnevaluatableError。这基本上是 1.2 版本之前的行为，目的是允许正在依赖此模式的应用程序进行迁移。然而，如果给定的属性名称无法与映射器的列匹配，则仍然会引发 UnevaluatableError，这是在[#3366](https://www.sqlalchemy.org/trac/ticket/3366)中修复的问题。

    参考：[#4073](https://www.sqlalchemy.org/trac/ticket/4073)

### orm 声明性

+   **[orm] [declarative] [bug]**

    如果子类试图覆盖在父类上使用 `@declared_attr.cascading` 声明的属性，则会发出警告，表明覆盖的属性将被忽略。这种用法不能完全支持到更进一步的子类，需要更复杂的开发工作，因此为了一致性，无论覆盖属性如何，都会一直遵循“级联”一直下去。

    参考：[#4091](https://www.sqlalchemy.org/trac/ticket/4091)

+   **[orm] [declarative] [bug]**

    如果 `@declared_attr.cascading` 属性与特殊的声明性名称（例如 `__tablename__`）一起使用，则会发出警告，因为这没有效果。

    参考：[#4092](https://www.sqlalchemy.org/trac/ticket/4092)

### engine

+   **[engine] [feature]**

    为`ResultProxy`添加了`__next__()`和`next()`方法，以便直接在对象上使用`next()`内置函数。`ResultProxy`长期以来已经有了`__iter__()`方法，它已经允许它响应`iter()`内置函数。`__iter__()`的实现没有改变，因为性能测试表明，使用带有`StopIteration`的`__next__()`方法进行迭代在 Python 2.7 和 3.6 中都慢约 20%。

    参考：[#4077](https://www.sqlalchemy.org/trac/ticket/4077)

+   **[engine] [bug]**

    对`Pool`和`Connection`进行了一些调整，以便在 `pool.Empty`、`AttributeError` 的异常捕获下不运行恢复逻辑，因为当恢复操作本身失败时，Python 3 会创建一个误导性的堆栈跟踪，将 `Empty` / `AttributeError` 误认为是原因，而实际上这些异常捕获是控制流的一部分。

    参考：[#4028](https://www.sqlalchemy.org/trac/ticket/4028)

### sql

+   **[sql] [bug]**

    修复了最近添加的`ColumnOperators.any_()`和`ColumnOperators.all_()`方法在被调用为方法时不起作用的错误，与使用独立函数`any_()`和`all_()`相反。还为这些相对不直观的 SQL 操作添加了文档示例。

    这个更改也**回溯**到：1.1.15

    参考：[#4093](https://www.sqlalchemy.org/trac/ticket/4093)

+   **[sql] [bug]**

    添加了一个新的方法 `DefaultExecutionContext.get_current_parameters()`，用于在基于函数的默认值生成器中检索传递给语句的当前参数。新函数与 `DefaultExecutionContext.current_parameters` 属性不同之处在于，它还提供了对与多值“插入”构造相对应的参数的可选分组。以前不可能确定与函数调用相关的参数子集。

    另请参阅

    多值插入的参数辅助工具带有上下文默认生成器

    上下文敏感的默认函数

    参考：[#4075](https://www.sqlalchemy.org/trac/ticket/4075)

+   **[sql] [bug]**

    修复了新 SQL 注释功能中的错误，其中在使用 `Table.tometadata()` 时，表和列注释不会被复制。

    参考：[#4087](https://www.sqlalchemy.org/trac/ticket/4087)

+   **[sql] [bug]**

    在 1.1 版本中，`Boolean` 类型存在问题，即在没有“本地布尔值”的后端中，通过 `bool()` 进行布尔强制转换，但在本地布尔后端中不会发生，这意味着字符串 `"0"` 现在的行为不一致。经过投票，达成共识，即非布尔值应该引发错误，特别是在字符串 `"0"` 的模糊情况下；因此，如果传入值不在 `None, True, False, 1, 0` 范围内，`Boolean` 数据类型现在会引发 `ValueError`。

    另请参阅

    布尔数据类型现在强制使用严格的 True/False/None 值

    参考：[#4102](https://www.sqlalchemy.org/trac/ticket/4102)

+   **[sql] [bug]**

    优化了`Operators.op()`的行为，使得在所有情况下，如果`Operators.op.is_comparison`标志设置为 True，则结果表达式的返回类型将是`Boolean`，如果标志为 False，则结果表达式的返回类型将与左侧表达式的类型相同，这是其他运算符的典型默认行为。还添加了一个新参数`Operators.op.return_type`以及一个辅助方法`Operators.bool_op()`。

    参见

    自定义运算符的类型行为已经变得一致

    参考：[#4063](https://www.sqlalchemy.org/trac/ticket/4063)

+   **[sql] [错误]**

    对`Enum`、`Interval`和`Boolean`类型进行内部优化，现在它们都扩展了一个名为`Emulated`的通用混合类型，表示提供了对数据库原生类型的 Python 端模拟，在使用支持的后端时切换到数据库原生类型。直接使用 PostgreSQL 的`INTERVAL`类型现在将包括正确的类型强制转换规则，这些规则也适用于`Interval`（例如将日期添加到间隔会产生日期时间）。

    参考：[#4088](https://www.sqlalchemy.org/trac/ticket/4088)

### postgresql

+   **[postgresql] [功能]**

    向 psycopg2 方言添加了一个新标志`use_batch_mode`。当`Engine`调用`cursor.executemany()`时，此标志启用了 psycopg2 的`psycopg2.extras.execute_batch`扩展。这个扩展在批量运行 INSERT 语句时提供了关键的性能提升，性能提升超过一个数量级。该标志默认为 False，因为目前被认为是实验性的。

    参见

    批处理模式/快速执行助手的支持

    参考：[#4109](https://www.sqlalchemy.org/trac/ticket/4109)

+   **[postgresql] [错误]**

    进一步修复了与 COLLATE 结合使用的 `ARRAY` 类中的问题，因为在 [#4006](https://www.sqlalchemy.org/trac/ticket/4006) 中进行的修复未能适应多维数组。

    这个更改也被**回溯**到：1.1.15

    参考：[#4006](https://www.sqlalchemy.org/trac/ticket/4006)

+   **[postgresql] [bug]**

    修复了`array_agg`函数中的错误，其中传递一个已经是`ARRAY`类型的参数，例如 PostgreSQL `array` 构造，会产生`ValueError`，因为函数尝试嵌套数组。

    这个更改也被**回溯**到：1.1.15

    参考：[#4107](https://www.sqlalchemy.org/trac/ticket/4107)

+   **[postgresql] [bug]**

    修复了 PostgreSQL `Insert.on_conflict_do_update()` 中的错误，该错误将阻止将插入语句用作 CTE，例如通过 `Insert.cte()`，在另一个语句中。

    这个更改也被**回溯**到：1.1.15

    参考：[#4074](https://www.sqlalchemy.org/trac/ticket/4074)

+   **[postgresql] [bug]**

    修复了 pg8000 驱动程序在使用带有模式名称的 `MetaData.reflect()` 时会失败的错误，因为模式名称将作为“quoted_name”对象发送，该对象是一个字符串子类，pg8000 不识别。在连接时，quoted_name 类型被添加到 pg8000 的 py_types 集合中。

    参考：[#4041](https://www.sqlalchemy.org/trac/ticket/4041)

+   **[postgresql] [bug]**

    为 pg8000 驱动程序启用了 UUID 支持，支持此数据类型的本机 Python uuid 往返。但是仍然不支持 UUID 数组。

    参考：[#4016](https://www.sqlalchemy.org/trac/ticket/4016)

### mysql

+   **[mysql] [bug]**

    当检测到 MariaDB 10.2.8 或更早版本的 10.2 系列时发出警告，因为这些版本中的 CHECK 约束存在重大问题，这些问题在 10.2.9 中已解决。

    请注意，此更改日志消息未随 SQLAlchemy 1.2.0b3 一起发布，而是事后添加的。

    这个更改也被**回溯**到：1.1.15

    参考：[#4097](https://www.sqlalchemy.org/trac/ticket/4097)

+   **[mysql] [bug]**

    修复了在 MariaDB 10.2 系列中 CURRENT_TIMESTAMP 由于语法更改而无法正确反映的问题，其中该函数现在表示为 `current_timestamp()`。

    这个更改也被**回溯**到：1.1.15

    参考：[#4096](https://www.sqlalchemy.org/trac/ticket/4096)

+   **[mysql] [bug]**

    MariaDB 10.2 现在支持 CHECK 约束（警告：由于上游问题，请使用版本 10.2.9 或更高版本，详见[#4097](https://www.sqlalchemy.org/trac/ticket/4097)）。反射现在在存在时考虑这些 CHECK 约束，当它们出现在`SHOW CREATE TABLE`输出中时。

    此更改也被**回溯**到：1.1.15

    参考：[#4098](https://www.sqlalchemy.org/trac/ticket/4098)

+   **[mysql] [bug]**

    将新的 MySQL INSERT..ON DUPLICATE KEY UPDATE 构造的`.values`属性的名称更改为`.inserted`，因为`Insert`已经有一个名为`Insert.values()`的方法。`.inserted`属性最终呈现 MySQL 的`VALUES()`函数。

    参考：[#4072](https://www.sqlalchemy.org/trac/ticket/4072)

### sqlite

+   **[sqlite] [bug]**

    修复了一个 bug，当 SQLite 的 CHECK 约束反射失败时，如果引用的表在远程模式下，例如在 SQLite 中由 ATTACH 引用的远程数据库，则会失败。

    此更改也被**回溯**到：1.1.15

    参考：[#4099](https://www.sqlalchemy.org/trac/ticket/4099)

### mssql

+   **[mssql] [feature]**

    添加了一个新的`TIMESTAMP`数据类型，它在 SQL Server 中正确地像二进制数据类型一样工作，而不是一个 datetime 类型，因为 SQL Server 在这里违反了 SQL 标准。还添加了`ROWVERSION`，因为 SQL Server 中的`TIMESTAMP`类型已被弃用，改用 ROWVERSION。

    参考：[#4086](https://www.sqlalchemy.org/trac/ticket/4086)

+   **[mssql] [feature]**

    添加了对“AUTOCOMMIT”隔离级别的支持，通过`Connection.execution_options()`在 PyODBC 和 pymssql 方言中建立。此隔离级别在底层连接对象上设置适当的 DBAPI 特定标志。

    参考：[#4058](https://www.sqlalchemy.org/trac/ticket/4058)

+   **[mssql] [bug]**

    为 SQL Server 的 PyODBC 方言添加了完整的“连接关闭”异常代码范围，包括‘08S01’、‘01002’、‘08003’、‘08007’、‘08S02’、‘08001’、‘HYT00’、‘HY010’。以前只覆盖了‘08S01’。

    此更改也被**回溯**到：1.1.15

    参考：[#4095](https://www.sqlalchemy.org/trac/ticket/4095)

+   **[mssql] [bug]**

    SQL Server 支持 SQLAlchemy 称为“本地布尔”的 BIT 类型，因为该类型只接受 0 或 1，而 DBAPI 将其值返回为 True/False。因此，SQL Server 方言现在启用了“本地布尔”支持，即不为`Boolean`数据类型生成 CHECK 约束。与其他本地布尔的唯一区别是没有“true” / “false”常量，因此这里仍然呈现为“1”和“0”。

    参考：[#4061](https://www.sqlalchemy.org/trac/ticket/4061)

+   **[mssql] [bug]**

    修复了 pymssql 方言中 SQL 文本中的百分号，例如在模数表达式或文字值中使用的情况，不会加倍，这似乎是 pymssql 所期望的。尽管 pymssql DBAPI 使用“pyformat”参数样式，该样式认为百分号是重要的。

    参考：[#4057](https://www.sqlalchemy.org/trac/ticket/4057)

+   **[mssql] [bug]**

    修复了 SQL Server 方言在反射自引用外键约束时可能从多个模式中提取列的错误，如果多个模式包含相同名称的约束针对相同名称的表。

    参考：[#4060](https://www.sqlalchemy.org/trac/ticket/4060)

+   **[mssql] [bug] [orm]**

    为特定于“RETURNING”的方言添加了一个新类别的“rowcount 支持”，在 SQL Server 上看起来像“OUTPUT inserted”，因为 PyODBC 后端在 OUTPUT 生效时无法给我们 UPDATE 或 DELETE 语句的 rowcount。这主要影响 ORM，当 flush 更新包含服务器计算值的行时，如果后端没有返回预期的行数，会引发错误。PyODBC 现在声明支持 rowcount，除非存在 OUTPUT.inserted，ORM 在 flush 期间会考虑是否寻找 rowcount。

    参考：[#4062](https://www.sqlalchemy.org/trac/ticket/4062)

+   **[mssql] [bug] [orm]**

    为 pymssql 方言启用了“sane_rowcount”标志，表示 DBAPI 现在报告 UPDATE 或 DELETE 语句受影响的正确行数。这主要影响 ORM 版本功能，因为它现在可以验证目标版本受影响的行数。

+   **[mssql] [bug]**

    添加了一个规则到 SQL Server 索引反射中，忽略了在未指定聚集索引的表上隐式存在的所谓“堆”索引。

    参考：[#4059](https://www.sqlalchemy.org/trac/ticket/4059)

### oracle

+   **[oracle] [performance] [bug] [py2k]**

    由于修复[#3937](https://www.sqlalchemy.org/trac/ticket/3937)导致的性能回归已经修复，其中 cx_Oracle 自版本 5.3 起从其命名空间中删除了`.UNICODE`符号，这被解释为 cx_Oracle 的“WITH_UNICODE”模式被无条件地打开，从而在 SQLAlchemy 端调用函数将所有字符串无条件地转换为 unicode 并导致性能影响。实际上，根据 cx_Oracle 的作者，“WITH_UNICODE”模式自 5.1 起已被完全移除，因此昂贵的 unicode 转换函数不再必要，如果在 Python 2 下检测到 cx_Oracle 5.1 或更高版本，则会被禁用。已恢复在[#3937](https://www.sqlalchemy.org/trac/ticket/3937)中删除的“WITH_UNICODE”模式警告。

    此更改也被**回溯**到：1.1.13，1.0.19

    参考：[#4035](https://www.sqlalchemy.org/trac/ticket/4035)

+   **[oracle] [bug]**

    使用 cx_Oracle 实现对 Oracle 值“infinity”的部分支持，仅使用 Python 浮点值，例如`float("inf")`。cx_Oracle DBAPI 驱动程序尚未实现十进制支持。

    参考：[#4064](https://www.sqlalchemy.org/trac/ticket/4064)

+   **[oracle] [bug]**

    cx_Oracle 方言已经重新设计和现代化，以利用旧的 4.x 系列 cx_Oracle 中不存在的新模式。其中包括最小的 cx_Oracle 版本是 5.x 系列，cx_Oracle 6.x 现在已经完全测试。最重要的变化涉及类型转换，主要是关于数字/浮点和 LOB 数据类型，更有效地利用 cx_Oracle 类型处理挂钩简化了绑定参数和结果数据的处理方式。

    参见

    cx_Oracle 方言，类型系统的重大重构

+   **[oracle] [bug]**

    对于所有版本的 cx_Oracle，两阶段支持已完全移除，而在 1.2.0b1 中，此更改仅对 cx_Oracle 的 6.x 系列生效。这个功能在任何版本的 cx_Oracle 中都从未正确工作，在 cx_Oracle 6.x 中，SQLAlchemy 依赖的 API 已被移除。

    参见

    cx_Oracle 方言，类型系统的重大重构

    参考：[#3997](https://www.sqlalchemy.org/trac/ticket/3997)

+   **[oracle] [bug]**

    当使用 cx_Oracle 后端的`Insert.returning()`时，结果集中的列键现在使用正确的列/标签名称，与所有其他方言一样。以前，这些列键会显示为`ret_nnn`。

    参见

    cx_Oracle 方言，类型系统的重大重构

+   **[oracle] [bug]**

    cx_Oracle 方言的几个参数现在已被弃用且不再起作用：`auto_setinputsizes`，`exclude_setinputsizes`，`allow_twophase`。

    参见

    对 cx_Oracle 方言、类型系统的重大重构

+   **[oracle] [bug]**

    修复了在 Oracle 下反映出一个类似“column DESC”的表达式的索引不会被返回的错误，如果表也没有主键，这是由于尝试过滤掉 Oracle 隐式添加到主键列上的索引的逻辑导致的。

    参考：[#4042](https://www.sqlalchemy.org/trac/ticket/4042)

+   **[oracle] [bug]**

    修复了由 cx_Oracle 6.0 引起的更多回归问题；目前，用户唯一的行为变化是断开检测现在除了 cx_Oracle.InterfaceError 外还检测 cx_Oracle.DatabaseError，因为这种行为似乎已经改变。其他关于数字精度和无法关闭连接的问题仍在上游 cx_Oracle 问题跟踪器中挂起。

    参考：[#4045](https://www.sqlalchemy.org/trac/ticket/4045)

+   **[oracle] [bug]**

    修复了 Oracle 8“非 ansi”连接模式不会向使用与`=`操作符不同的操作符的表达式添加`(+)`运算符的错误。`(+)`需要添加到右侧的所有列。

    参考：[#4076](https://www.sqlalchemy.org/trac/ticket/4076)

## 1.2.0b2

发布日期：2017 年 7 月 24 日

### orm

+   **[orm] [bug]**

    修复了从 1.1.11 开始的回归，其中向包含具有子查询加载关系的实体的查询添加额外的非实体列会失败，这是由于 1.1.11 中添加的检查导致的，这是由于[#4011](https://www.sqlalchemy.org/trac/ticket/4011)。

    这个更改也被**回溯**到：1.1.12

    参考：[#4033](https://www.sqlalchemy.org/trac/ticket/4033)

+   **[orm] [bug]**

    修复了在 1.1 中添加的涉及 JSON NULL 评估逻辑的错误，其中逻辑不会适应与`Column`不同命名的 ORM 映射属性。

    这个更改也被**回溯**到：1.1.12

    参考：[#4031](https://www.sqlalchemy.org/trac/ticket/4031)

+   **[orm] [bug]**

    在`WeakInstanceDict`中的所有方法中添加了`KeyError`检查，其中在检查`key in dict`后紧接着对该键进行索引访问，以防止在负载下垃圾收集导致的竞争中，代码假定键存在后，键从字典中被移除，导致非常罕见的`KeyError`引发。

    这个更改也被**回溯**到：1.1.12

    参考：[#4030](https://www.sqlalchemy.org/trac/ticket/4030)

### 测试

+   **[tests] [bug] [py3k]**

    修复了与 Python 3.6.2 的更改不兼容的测试固定装置中的问题，涉及上下文管理器。

    这个更改也被**回溯**到：1.1.12, 1.0.18

    参考：[#4034](https://www.sqlalchemy.org/trac/ticket/4034)

### orm

+   **[orm] [bug]**

    修复了从 1.1.11 开始的回归，其中向包含具有子查询加载关系的实体的查询添加额外的非实体列将失败，原因是 1.1.11 中添加的检查作为[#4011](https://www.sqlalchemy.org/trac/ticket/4011)的结果。

    此更改还**回溯**到：1.1.12

    参考：[#4033](https://www.sqlalchemy.org/trac/ticket/4033)

+   **[orm] [错误]**

    修复了在 1.1 中添加的涉及 JSON NULL 评估逻辑的错误，该逻辑不会适应与`Column`不同命名的 ORM 映射属性的情况。

    此更改还**回溯**到：1.1.12

    参考：[#4031](https://www.sqlalchemy.org/trac/ticket/4031)

+   **[orm] [错误]**

    在`WeakInstanceDict`中的所有方法中添加了`KeyError`检查，其中检查`key in dict`后跟随对该键的索引访问，以防止在负载下垃圾收集导致的竞争中，代码假设其存在后，将键从字典中移除，导致非常罕见的`KeyError`引发。

    此更改还**回溯**到：1.1.12

    参考：[#4030](https://www.sqlalchemy.org/trac/ticket/4030)

### 测试

+   **[测试] [错误] [py3k]**

    修复了与 Python 3.6.2 的更改不兼容的测试固定装置中的问题，涉及上下文管理器的更改。

    此更改还**回溯**到：1.1.12, 1.0.18

    参考：[#4034](https://www.sqlalchemy.org/trac/ticket/4034)

## 1.2.0b1

发布日期：2017 年 7 月 10 日

### orm

+   **[orm] [功能]**

    `aliased()`构造现在可以传递给`Query.select_entity_from()`方法。实体将从由`aliased()`构造表示的可选择项中提取。这允许与`Query.select_entity_from()`一起使用`aliased()`的特殊选项，例如`aliased.adapt_on_names`。

    此更改还**回溯**到：1.1.7

    参考：[#3933](https://www.sqlalchemy.org/trac/ticket/3933)

+   **[orm] [功能]**

    向`scoped_session`添加了`.autocommit`属性，代理当前分配给线程的底层`Session`的`.autocommit`属性。感谢 Ben Fagin 的拉取请求。

+   **[orm] [功能]**

    添加了一个新特性`with_expression()`，允许在结果时间将一个临时 SQL 表达式添加到查询中的特定实体。这是 SQL 表达式作为结果元组中的一个单独元素传递的替代方法。

    参见

    ORM 属性可以接收临时 SQL 表达式

    参考：[#3058](https://www.sqlalchemy.org/trac/ticket/3058)

+   **[orm] [feature]**

    添加了一种新的映射器级继承加载方式“多态 selectin”。这种加载方式在加载基本对象类型后，为继承层次结构中的每个子类发出查询，使用 IN 来指定所需的主键值。

    参见

    “selectin”多态加载，使用单独的 IN 查询加载子类

    参考：[#3948](https://www.sqlalchemy.org/trac/ticket/3948)

+   **[orm] [feature]**

    添加了一种新的急切加载方式称为“selectin”加载。这种加载方式与“subquery”急切加载非常相似，只是它使用了一个 IN 表达式，给出了加载的父对象的主键值列表，而不是重新陈述原始查询。这产生了一个更有效的查询，是“烘焙”（例如，SQL 字符串被缓存），并且也适用于`Query.yield_per()`的上下文中。

    参见

    新的“selectin”急切加载，使用 IN 一次加载所有集合

    参考：[#3944](https://www.sqlalchemy.org/trac/ticket/3944)

+   **[orm] [feature]**

    `lazy="select"`加载策略现在在所有情况下都使用`BakedQuery`查询缓存系统。这消除了生成`Query`对象并将其运行到`select()`和然后从懒加载相关集合和对象的过程中生成字符串 SQL 语句的大部分开销。 “烘焙”懒加载器也得到了改进，以便在大多数情况下可以缓存使用查询加载选项的情况。

    参见

    “烘焙”加载现在是延迟加载的默认方式

    参考：[#3954](https://www.sqlalchemy.org/trac/ticket/3954)

+   **[orm] [feature] [ext]**

    `Query.update()`方法现在可以容纳混合属性和复合属性作为放置在 SET 子句中的键的来源。对于混合属性，还提供了一个额外的装饰器`hybrid_property.update_expression()`，用户提供一个返回元组的函数。

    另请参阅

    支持混合、复合的批量更新

    参考：[#3229](https://www.sqlalchemy.org/trac/ticket/3229)

+   **[orm] [feature]**

    添加了新的属性事件`AttributeEvents.bulk_replace()`。当将集合分配给关系时触发此事件，在将传入集合与现有集合进行比较之前。这个早期事件还允许转换传入的非 ORM 对象。该事件与`@validates`装饰器集成。

    另请参阅

    新的 bulk_replace 事件

    参考：[#3896](https://www.sqlalchemy.org/trac/ticket/3896)

+   **[orm] [feature]**

    添加了新的事件处理程序`AttributeEvents.modified()`，当调用 func:.attributes.flag_modified 函数时触发，这在使用`sqlalchemy.ext.mutable`扩展模块时很常见。

    另请参阅

    SQLAlchemy.ext.mutable 中的新“修改”事件处理程序

    参考：[#3303](https://www.sqlalchemy.org/trac/ticket/3303)

+   **[orm] [bug]**

    修复了子查询急加载的问题，这个问题延续自[#2699](https://www.sqlalchemy.org/trac/ticket/2699)、[#3106](https://www.sqlalchemy.org/trac/ticket/3106)、[#3893](https://www.sqlalchemy.org/trac/ticket/3893)中修复的一系列问题，涉及到“子查询”在从连接的继承子类开始，然后对基类的关系进行子查询急加载时，包含正确的 FROM 子句，同时查询还包括对子类的条件。之前票证中的修复没有考虑到从第一级更深层次加载更多的 subqueryload 操作，因此修复进一步泛化。

    此更改也**回溯**到：1.1.11

    参考：[#4011](https://www.sqlalchemy.org/trac/ticket/4011)

+   **[orm] [bug]**

    修复了级联操作（如“delete-orphan”等）无法定位与继承关系中本地关系相连的对象的 bug，从而导致操作无法执行。

    此更改也**回溯**到：1.1.10

    参考：[#3986](https://www.sqlalchemy.org/trac/ticket/3986)

+   **[orm] [bug]**

    修复了一个在多线程环境下可能发生的竞争条件，这是由于通过[#3915](https://www.sqlalchemy.org/trac/ticket/3915)添加的缓存引起的。一个`Column`对象的内部集合可能会在别名对象上不恰当地重新生成，当尝试渲染 SQL 并收集结果时，会混淆一个连接的急切加载器，导致属性错误。现在，在别名对象被缓存和在线程之间共享之前，集合会提前生成。

    此更改也**回溯**到：1.1.7

    参考：[#3947](https://www.sqlalchemy.org/trac/ticket/3947)

+   **[orm] [bug]**

    由于`relationship.post_update`功能发出的 UPDATE 现在将与版本控制功能集成，以增加行的版本 id 并断言现有版本号是否匹配。

    另请参阅

    post_update 与 ORM 版本控制集成

    参考：[#3496](https://www.sqlalchemy.org/trac/ticket/3496)

+   **[orm] [bug]**

    修复了几个使用`relationship.post_update`功能的用例，当与具有“onupdate”值的列一起使用时。当 UPDATE 发出时，相应的对象属性现在会过期或刷新，以便新生成的“onupdate”值可以填充到对象中；以前的过时值将保留。此外，如果目标属性在 Python 中设置为对象的 INSERT，那么在 UPDATE 期间该值现在将被重新发送，以便“onupdate”不会覆盖它（请注意，这对于服务器生成的 onupdates 同样有效）。最后，在刷新时，当在 flush 中刷新这些属性时，现在会发出`SessionEvents.refresh_flush()`事件。

    另请参阅

    与 onupdate 一起的 post_update 的改进

    参考：[#3471](https://www.sqlalchemy.org/trac/ticket/3471), [#3472](https://www.sqlalchemy.org/trac/ticket/3472)

+   **[orm] [bug]**

    修复了一个 bug，即在程序化版本 _id 计数器与连接表继承结合使用时，如果版本 _id 计数器实际上没有增加，并且基表上没有修改其他值，那么 UPDATE 将具有一个空的 SET 子句，会失败。由于程序化版本 _id 在版本计数器未增加的情况下是一个已记录的用例，现在会检测到这种特定情况，并且 UPDATE 现在将版本 _id 值设置为自身，以便并发检查仍然进行。

    参考：[#3996](https://www.sqlalchemy.org/trac/ticket/3996)

+   **[orm] [bug]**

    版本控制功能不支持版本计数器的 NULL 值。如果版本 id 是程序化的，并且在 UPDATE 时设置为 NULL，则现在会引发异常。感谢 Diana Clarke 提交的拉取请求。

    参考资料：[#3673](https://www.sqlalchemy.org/trac/ticket/3673)

+   **[orm] [bug]**

    从 `scoped_session` 中移除了一个非常古老的关键字参数 `scope`。这个关键字从未被文档化，并且是早期尝试允许变量作用域的一部分。

    另请参阅

    从 scoped_session 中移除了“scope”关键字

    参考资料：[#3796](https://www.sqlalchemy.org/trac/ticket/3796)

+   **[orm] [bug]**

    修复了一个 bug，在其中结合“with_polymorphic”加载与指定了 innerjoin=True 的子类链接关系的 joinedload 结合使用时，会导致无法将这些“innerjoins”降级为“outerjoins”，以适应不支持该关系的其他多态类。这适用于单个继承和联合继承多态加载。

    参考资料：[#3988](https://www.sqlalchemy.org/trac/ticket/3988)

+   **[orm] [bug]**

    向 `Session.refresh()` 方法添加了新参数 `with_for_update`。当 `Query.with_lockmode()` 方法被弃用，改用 `Query.with_for_update()` 时，`Session.refresh()` 方法从未被更新以反映新选项。

    另请参阅

    向 Session.refresh 添加了“for update”参数

    参考资料：[#3991](https://www.sqlalchemy.org/trac/ticket/3991)

+   **[orm] [bug]**

    修复了一个 bug，在其中将 `column_property()` 标记为“deferred”后，在 flush 期间会将其标记为“expired”，导致它与常规属性的未过期加载一起被加载，即使从未访问过该属性也是如此。

    参考资料：[#3984](https://www.sqlalchemy.org/trac/ticket/3984)

+   **[orm] [bug]**

    修复了子查询提前加载中的错误，其中自引用关系的“join_depth”参数不会被正确地识别，而是加载所有可用的深度，而不是正确地计算提前加载的指定级别数。

    参考资料：[#3967](https://www.sqlalchemy.org/trac/ticket/3967)

+   **[orm] [bug]**

    向 `Mapper`（最终也将用于其他基于 ORM 的 LRU 缓存）的 LRU “编译缓存”添加了警告，以便当缓存开始达到其大小限制时，应用程序会发出警告，指出这是一个可能需要关注的性能下降情况。LRU 缓存主要会达到其大小限制，如果应用程序使用了无限数量的 `Engine` 对象，这是一个反模式。否则，这可能表明存在一个应该引起 SQLAlchemy 开发人员注意的问题。

+   **[orm] [bug]**

    修复了一个 bug，以改进在延迟加载相关实体后生效的加载器选项的特异性，以便如果这些选项包括实体信息，则加载器选项将更具体地匹配到别名或非别名实体。

    参考：[#3963](https://www.sqlalchemy.org/trac/ticket/3963)

+   **[orm] [bug]**

    `flag_modified()` 函数现在会在对象中未找到指定属性键时引发 `InvalidRequestError`，因为这在刷新过程中被假定为存在。要在不涉及任何特定属性的情况下标记对象为“脏”以进行刷新，可以使用 `flag_dirty()` 函数。

    另请参阅

    使用 flag_dirty() 将对象标记为“脏”，而不更改任何属性

    参考：[#3753](https://www.sqlalchemy.org/trac/ticket/3753)

+   **[orm] [bug]**

    `Query.update()` 和 `Query.delete()` 使用的“evaluate”策略现在可以在主键/外键列的属性名称与实际列名称不匹配时，从多对一关系到实例进行简单对象比较。以前，这将进行简单的基于名称的匹配，并在 AttributeError 失败。

    参考：[#3366](https://www.sqlalchemy.org/trac/ticket/3366)

+   **[orm] [bug]**

    `@validates` 装饰器现在允许装饰的方法接收尚未与现有集合进行比较的“批量集合设置”操作中的对象。这允许传入值转换为兼容的 ORM 对象，就像从“追加”事件中已经允许的那样。请注意，这意味着 `@validates` 方法在集合分配期间对**所有**值进行调用，而不仅仅是新值。

    另请参阅

    在比较之前，@validates 方法在批量集合设置时接收所有值

    参考：[#3896](https://www.sqlalchemy.org/trac/ticket/3896)

+   **[orm] [bug]**

    修复了单表继承中的 bug，当将行限制为子类时，select_from()参数不会被考虑。以前，只有请求的列中的表达式会被考虑。

    另请参阅

    修复了与 select_from()一起使用的单表继承的问题

    参考：[#3891](https://www.sqlalchemy.org/trac/ticket/3891)

+   **[orm] [bug]**

    当将集合分配给由关系映射的属性时，先前的集合不再被改变。以前，旧集合会在“项移除”事件触发时被清空；现在事件会触发而不影响旧集合。

    另请参阅

    替换时不再改变先前的集合

    参考：[#3913](https://www.sqlalchemy.org/trac/ticket/3913)

+   **[orm] [bug]**

    当`SessionEvents.after_rollback()`事件被触发时，`Session`的状态现在是存在的，即对象在过期之前的属性状态。这与`SessionEvents.after_commit()`事件的行为一致，该事件在对象的属性状态过期之前也会被触发。

    另请参阅

    在对象过期之前，after_rollback() Session 事件现在会被触发

    参考：[#3934](https://www.sqlalchemy.org/trac/ticket/3934)

+   **[orm] [bug]**

    修复了一个 bug，`Query.with_parent()`在针对`aliased()`构造而不是常规映射类时无法工作。还为独立的`with_parent()`函数以及`Query.with_parent()`添加了一个新参数`with_parent.from_entity`。

    参考：[#3607](https://www.sqlalchemy.org/trac/ticket/3607)

### orm 声明

+   **[orm] [declarative] [bug]**

    修复了在`AbstractConcreteBase`上使用`declared_attr`时的错误，其中特定返回值为一些非映射符号，包括`None`，会导致属性只被强制评估一次并将值存储到对象字典中，不允许其为子类调用。当`declared_attr`在映射类上时，这种行为是正常的，并且不会发生在混入类或抽象类上。由于`AbstractConcreteBase`既是“抽象”的又实际上是“映射”的，因此在这里特殊的例外情况是“抽象”行为优先于`declared_attr`。

    参考：[#3848](https://www.sqlalchemy.org/trac/ticket/3848)

### 引擎

+   **[引擎] [功能]**

    向`Pool`对象添加了本机“悲观断开”处理。新参数`Pool.pre_ping`，可从引擎中作为`create_engine.pool_pre_ping`获得，应用了池文档中特色的“预先 ping”配方的高效形式，每次连接检出时，发出一个简单的语句，通常是“SELECT 1”，以测试连接的活动性。如果现有连接不再能响应命令，则连接将被透明地回收，并且在当前时间戳之前进行的所有其他连接将被作废。

    另请参阅

    断开处理 - 悲观

    连接池中添加了悲观断开检测

    参考：[#3919](https://www.sqlalchemy.org/trac/ticket/3919)

+   **[引擎] [错误]**

    添加了一个异常处理程序，当`Connection`的“autorollback”功能本身引发异常时，将会警告“cause”异常在 Py2K 上。在 Py3K 中，这两个异常自然地由解释器报告为一个发生在处理另一个异常时。这是继续处理回滚失��处理的一系列更改的一部分，上次在 1.0.12 中作为[#2696](https://www.sqlalchemy.org/trac/ticket/2696)的一部分访问的。

    此更改也**回溯**到：1.1.7

    参考：[#3946](https://www.sqlalchemy.org/trac/ticket/3946)

+   **[引擎] [错误]**

    修复了一个 bug，在罕见情况下直接将`Compiled`对象传递给`Connection.execute()`时，生成`Compiled`对象的方言未被咨询以获取字符串语句的 paramstyle，而是假定它将匹配方言级别的 paramstyle，导致不匹配发生。

    参考：[#3938](https://www.sqlalchemy.org/trac/ticket/3938)

### sql

+   **[sql] [feature]**

    增加了一种名为“expanding”的新类型的`bindparam()`。这用于在`IN`表达式中，元素列表在语句执行时被渲染为单独的绑定参数，而不是在语句编译时。这允许将单个绑定参数名称链接到多个元素的 IN 表达式，同时允许使用查询缓存与 IN 表达式。这一新功能允许“select in”加载和“polymorphic in”加载使用烘焙查询扩展以减少调用开销。这一功能应被视为**实验性**的 1.2 版本。

    另请参阅

    延迟扩展的 IN 参数集允许具有缓存语句的 IN 表达式

    参考：[#3953](https://www.sqlalchemy.org/trac/ticket/3953)

+   **[sql] [feature] [mysql] [oracle] [postgresql]**

    增加了对`Table`和`Column`对象上的 SQL 注释的支持，通过新的`Table.comment`和`Column.comment`参数。这些注释作为 DDL 的一部分包含在表创建中，可以内联或通过适当的 ALTER 语句反映回来，并且也通过表反射以及通过`Inspector`反映回来。目前支持的后端包括 MySQL、PostgreSQL 和 Oracle。非常感谢 Frazer McLean 在这方面的大量努力。

    另请参阅

    支持在 Table、Column 上的 SQL 注释，包括 DDL、反射

    参考：[#1546](https://www.sqlalchemy.org/trac/ticket/1546)

+   **[sql] [feature]**

    `ColumnOperators.in_()`和`ColumnOperators.notin_()`操作符的长期行为已经修订，当右侧条件为空序列时，会发出警告；现在，默认情况下会呈现一个简单的“静态”表达式“1 != 1”或“1 = 1”，而不是引入原始的左侧表达式。这导致对空集合进行 NULL 列比较的结果从 NULL 更改为 true/false。该行为是可配置的，并且可以使用`create_engine.empty_in_strategy`参数来启用`create_engine()`中的旧行为。

    另请参阅

    IN / NOT IN 操作符的空集合行为现在是可配置的；默认表达式已简化

    参考：[#3907](https://www.sqlalchemy.org/trac/ticket/3907)

+   **[sql] [feature]**

    为“startswith”和“endswith”比较器类添加了一个新选项`autoescape`；这将提供一个转义���符，并自动将其应用于所有通配符字符“%”和“_”。感谢戴安娜·克拉克提供的拉取请求。

    注意

    该功能已于 1.2.0 中更改，从其在 1.2.0b2 中的初始实现中，现在 autoescape 作为布尔值传递，而不是作为转义字符使用。

    另请参阅

    新的“autoescape”选项用于 startswith()，endswith()

    参考：[#2694](https://www.sqlalchemy.org/trac/ticket/2694)

+   **[sql] [bug]**

    修复了在对结构进行迭代期间可能发生的`WithinGroup`构造中出现的 AttributeError。

    此更改也已**回溯**到：1.1.11

    参考：[#4012](https://www.sqlalchemy.org/trac/ticket/4012)

+   **[sql] [bug]**

    修复了 1.1.5 版本中由于[#3859](https://www.sqlalchemy.org/trac/ticket/3859)导致的回归问题，其中基于`Variant`的“右侧”表达式评估的调整，以遵守基础类型的“右侧”规则，导致`Variant`类型在那些情况下被不当地丢失，当我们确实希望左侧类型直接转移到右侧，以便将绑定级规则应用于表达式的参数时。

    此更改也已**回溯**到：1.1.9

    参考：[#3952](https://www.sqlalchemy.org/trac/ticket/3952)

+   **[sql] [bug] [postgresql]**

    更改了 `ResultProxy` 的机制，无条件地延迟“自动关闭”步骤，直到 `Connection` 完成对象的操作；在 PostgreSQL ON CONFLICT with RETURNING 返回零行的情况下，之前不存在的用例中发生了自动关闭，导致在 INSERT/UPDATE/DELETE 时无条件发生的通常自动提交行为失败。

    这个���改也被**回溯**到：1.1.9

    参考：[#3955](https://www.sqlalchemy.org/trac/ticket/3955)

+   **[sql] [bug]**

    `Numeric`、`Integer` 和与日期相关的类型之间的类型强制转换规则现在包括额外的逻辑，将尝试保留“resolved”类型的传入类型的设置。目前，这个目标是 `asdecimal` 标志，因此在 `Numeric` 或 `Float` 与 `Integer` 之间的数学操作将保留`asdecimal`标志，以及类型是否应该是 `Float` 子类。

    另请参阅

    “float” 数据类型增加了更强的类型检查

    参考：[#4018](https://www.sqlalchemy.org/trac/ticket/4018)

+   **[sql] [bug] [mysql]**

    `Float` 类型的结果处理器现在无条件地通过 `float()` 处理器运行值，如果方言指定它也支持“本地十进制”模式。虽然大多数后端会为浮点数据类型提供 Python `float` 对象，但在某些情况下，MySQL 后端缺乏类型信息，除非进行浮点转换，否则会返回 `Decimal`。

    另请参阅

    “float” 数据类型增加了更强的类型检查

    参考：[#4020](https://www.sqlalchemy.org/trac/ticket/4020)

+   **[sql] [bug]**

    对传递给 SQL 语句的 Python “float” 值的处理增加了一些额外的严格性。一个“float” 值将与 `Float` 数据类型关联，而不是以前的 Decimal-coercing `Numeric` 数据类型，消除了在 SQLite 上发出的令人困惑的警告以及不必要的强制转换为 Decimal。

    另请参阅

    “float” 数据类型增加了更强的类型检查

    参考：[#4017](https://www.sqlalchemy.org/trac/ticket/4017)

+   **[sql] [bug]**

    所有比较运算符（如 LIKE、IS、IN、MATCH、等于、大于、小于等）的操作符优先级已经合并为一个级别，因此使用这些运算符相互比较的表达式将在它们之间产生括号。这适用于像 Oracle、MySQL 等数据库的操作符优先级，这些数据库将所有这些运算符视为相等优先级，以及 PostgreSQL 9.5 版本之后也已经将其操作符优先级展平。

    另请参阅

    比较运算符的操作符优先级已展平

    参考：[#3999](https://www.sqlalchemy.org/trac/ticket/3999)

+   **[sql] [bug]**

    修复了使用`ColumnOperators.is_()`或类似操作符的表达式的类型不是“boolean”类型的问题，而是“nulltype”类型，以及当使用自定义比较运算符与未类型化表达式相对比时也会出现这种情况。这种类型化可能会影响表达式在更大上下文中以及在结果行处理中的行为。

    参考：[#3873](https://www.sqlalchemy.org/trac/ticket/3873)

+   **[sql] [bug]**

    修复了对`Label`构造的否定，以便在应用`not_()`修饰符时正确否定内部元素。

    参考：[#3969](https://www.sqlalchemy.org/trac/ticket/3969)

+   **[sql] [bug]**

    SQL 语句中百分号“加倍”以进行转义目的的系统已经得到改进。与`literal_column`构造以及像`ColumnOperators.contains()`这样的操作符密切相关的百分号“加倍”现在基于正在使用的 DBAPI 的指定 paramstyle 进行；对于像 PostgreSQL 和 MySQL 驱动程序常见的百分号敏感 paramstyles，将会发生加倍，对于 SQLite 等其他驱动程序则不会。这使得更多数据库无关的使用`literal_column`构造成为可能。

    另请参阅

    在`literal_column()`中条件性转义百分号

    参考：[#3740](https://www.sqlalchemy.org/trac/ticket/3740)

+   **[sql] [bug]**

    修复了一个 bug，其中列级 `CheckConstraint` 无法使用基础方言编译器编译 SQL 表达式，并且在 sqltext 是 Core 表达式而不仅仅是普通字符串的情况下应用正确的标志以生成内联字面值。长期以来，这在 0.9 中作为 [#2742](https://www.sqlalchemy.org/trac/ticket/2742) 的一部分已经修复了，这更常见地使用 Core SQL 表达式而不是普通字符串表达式的表级检查约束。

    参考：[#3957](https://www.sqlalchemy.org/trac/ticket/3957)

+   **[sql] [bug]**

    修复了一个 bug，在“预执行”代码路径中，如果 SQL 本身是一个未分类的表达式（比如纯文本），则 SQL 导向的 Python 侧列默认值可能在 INSERT 时无法正确执行。然而，“预执行”代码路径相当不常见，但是在不使用 RETURNING 时，它可以应用于具有 SQL 默认值的非整数主键列。

    参考：[#3923](https://www.sqlalchemy.org/trac/ticket/3923)

+   **[sql] [bug]**

    当列级 `collate()` 和 `ColumnOperators.collate()` 用于 COLLATE 的表达式在名称区分大小写时，现在会被引用为标识符，例如包含大写字符。请注意，这不影响类型级排序，因为它已经被引用。

    另见

    列级 COLLATE 关键字现在引用排序名称

    参考：[#3785](https://www.sqlalchemy.org/trac/ticket/3785)

+   **[sql] [bug]**

    修复了在列上下文中使用 `Alias` 对象会在尝试将自身分组到括号表达式中时引发参数错误的 bug。然而，以这种方式使用 `Alias` 还不是一个完全支持的 API，但它适用于一些最终用户的配方，并且可能在支持某些未来 PostgreSQL 特性时扮演更重要的角色。

    参考：[#3939](https://www.sqlalchemy.org/trac/ticket/3939)

### 模式

+   **[schema] [bug]**

    如果 `ForeignKeyConstraint` 对象创建时“本地”和“远程”列数量不匹配，则现在会引发 `ArgumentError`，否则会导致约束的内部状态不正确。请注意，这也影响方言的反射过程产生的外键约束的列集不匹配的情况。

    此更改也**回溯**到：1.1.10

    参考：[#3949](https://www.sqlalchemy.org/trac/ticket/3949)

### postgresql

+   **[postgresql] [错误]**

    继续修复正确处理 PostgreSQL 版本字符串“10devel”的问题，该问题在 1.1.8 中发布，另外增加了一个正则表达式来处理形式为“10beta1”的版本字符串。虽然 PostgreSQL 现在提供了更好的获取此信息的方法，但我们至少在 1.1.x 中仍然坚持使用正则表达式，以减少与旧版或替代 PostgreSQL 数据库的兼容性风险。

    此更改也**回溯**到：1.1.11

    参考：[#4005](https://www.sqlalchemy.org/trac/ticket/4005)

+   **[postgresql] [错误]**

    修复了在使用带有排序规则的字符串类型的 `ARRAY` 时，在 CREATE TABLE 中无法生成正确语法的错误。

    此更改也**回溯**到：1.1.11

    参考：[#4006](https://www.sqlalchemy.org/trac/ticket/4006)

+   **[postgresql] [错误]**

    为 GRANT、REVOKE 关键字添加了“autocommit”支持。感谢 Jacob Hayes 的拉取请求。

    此更改也**回溯**到：1.1.10

+   **[postgresql] [错误]**

    增加了解析 PostgreSQL 版本字符串的支持，例如“PostgreSQL 10devel”这样的开发版本。感谢 Sean McCully 的拉取请求。

    此更改也**回溯**到：1.1.8

+   **[postgresql] [错误]**

    修复了基本 `ARRAY` 数据类型不会调用 `ARRAY` 的绑定/结果处理器的错误。

    参考：[#3964](https://www.sqlalchemy.org/trac/ticket/3964)

+   **[postgresql] [错误]**

    在反射 PostgreSQL `INTERVAL` 数据类型时，增加了对所有可能的“字段”标识符的支持，例如“YEAR”、“MONTH”、“DAY TO MINUTE”等。此外，`INTERVAL` 数据类型本身现在包括一个新参数 `INTERVAL.fields`，可以在其中指定这些限定符；在反射/检查时，限定符也会反映到结果数据类型中。

    另请参阅

    INTERVAL 中字段规范的支持，包括完整反射

    参考：[#3959](https://www.sqlalchemy.org/trac/ticket/3959)

### mysql

+   **[mysql] [功能]**

    增加了对 MySQL 的 ON DUPLICATE KEY UPDATE MySQL 特定的 `Insert` 对象的支持。感谢 Michael Doronin 的拉取请求。

    另请参阅

    INSERT..ON DUPLICATE KEY UPDATE 的支持

    参考：[#4009](https://www.sqlalchemy.org/trac/ticket/4009)

+   **[mysql] [错误]**

    MySQL 5.7 引入了对“SHOW VARIABLES”命令的权限限制；MySQL 方言现在将处理当 SHOW 返回零行时，特别是对于 SQL_MODE 的初始获取，并将发出警告，指出用户权限应修改以允许该行存在。

    此更改也**回溯**到：1.1.11

    参考：[#4007](https://www.sqlalchemy.org/trac/ticket/4007)

+   **[mysql] [bug]**

    移除了对 UTC_TIMESTAMP MySQL 函数的古老且不必要的拦截，这妨碍了使用带参数的函数。

    此更改也**回溯**到：1.1.10

    参考：[#3966](https://www.sqlalchemy.org/trac/ticket/3966)

+   **[mysql] [bug]**

    修复了 MySQL 方言在渲染 CREATE TABLE 时与 PARTITION 选项一起渲染表选项时的错误。PARTITION 相关选项需要遵循表选项，而以前未强制执行此顺序。

    此更改也**回溯**到：1.1.10

    参考：[#3961](https://www.sqlalchemy.org/trac/ticket/3961)

+   **[mysql] [bug]**

    添加了对由于过时表定义而无法反射的视图的支持，当调用`MetaData.reflect()`时，对于无法响应`DESCRIBE`的表会发出警告，但操作成功。

    参考：[#3871](https://www.sqlalchemy.org/trac/ticket/3871)

### mssql

+   **[mssql] [bug]**

    修复了在使用 Azure 数据仓库时必须从不同视图获取 SQL Server 事务隔离的错误，现在查询将尝试针对两个视图，如果失败继续提供最佳的抗未来任意 API 更改的弹性，将无条件引发 NotImplemented。

    此更改也**回溯**到：1.1.11

    参考：[#3994](https://www.sqlalchemy.org/trac/ticket/3994)

+   **[mssql] [bug]**

    向 SQL Server 方言添加了一个占位符类型`XML`，以便包含此类型的反射表可以重新渲染为 CREATE TABLE。该类型没有特殊的往返行为，也不支持额外的限定参数。

    此更改也**回溯**到：1.1.11

    参考：[#3973](https://www.sqlalchemy.org/trac/ticket/3973)

+   **[mssql] [bug]**

    SQL Server 方言现在允许在其中带有点的数据库和/或所有者名称，使用字符串中明确在所有者周围以及可选地在数据库名称周围的括号。此外，将`quoted_name`构造发送到模式名称将不会在点上拆分，并将提供完整字符串作为“所有者”。`quoted_name`现在也可以从`sqlalchemy.sql`导入空间中使用。

    另请参阅

    支持带有嵌入点的 SQL Server 模式名称

    参考：[#2626](https://www.sqlalchemy.org/trac/ticket/2626)

### oracle

+   **[oracle] [feature] [postgresql]**

    添加了新关键字`Sequence.cache`和`Sequence.order`到`Sequence`，以允许呈现 Oracle 和 PostgreSQL 理解的 CACHE 参数以及 Oracle 理解的 ORDER 参数。感谢 David Moore 的拉取请求。

    此更改也**回溯**到：1.1.12

+   **[oracle] [feature]**

    当使用`Inspector.get_unique_constraints()`、`Inspector.get_check_constraints()`时，Oracle 方言现在检查唯一和检查约束。由于 Oracle 没有与唯一`Index`分开的唯一约束，反映的`Table`仍将继续不具有与之关联的`UniqueConstraint`对象。拉取请求由 Eloy Felix 提供。

    另请参阅

    Oracle 唯一性、检查约束现已反映

    参考：[#4003](https://www.sqlalchemy.org/trac/ticket/4003)

+   **[oracle] [bug]**

    当使用版本 6.0b1 或更高版本的 DBAPI 时，cx_Oracle 完全删除了对两阶段事务的支持。在任何情况下，cx_Oracle 5.x 历史上从未能够使用两阶段功能，而 cx_Oracle 6.x 已删除了此功能依赖的连接级“twophase”标志。

    此更改也**回溯**到：1.1.11

    参考：[#3997](https://www.sqlalchemy.org/trac/ticket/3997)

+   **[oracle] [bug]**

    修复了 cx_Oracle 方言中的错误，其中版本字符串解析会由于“b”字符而在 cx_Oracle 版本 6.0b1 中失败。现在版本字符串解析是通过正则表达式而不是简单的拆分。

    此更改还**回溯**到：1.1.10

    参考：[#3975](https://www.sqlalchemy.org/trac/ticket/3975)

+   **[Oracle] [错误]**

    cx_Oracle 方言现在支持“合理的多行计数”，即当通过 DBAPI `cursor.executemany()` 执行一系列参数集时，我们可以利用 `cursor.rowcount` 来验证匹配的行数。这在 ORM 中检测并发修改场景时有影响，因为现在即使 ORM 批处理语句，一些简单条件也可以被检测到，而且当使用更严格的版本控制功能时，ORM 仍然可以使用语句批处理。假定至少为版本 5.0，这个标志对 cx_Oracle 启用，这现在是很普遍的。

    参考：[#3932](https://www.sqlalchemy.org/trac/ticket/3932)

+   **[Oracle] [错误]**

    Oracle 反射现在“规范化”了外键约束的名称，即对于大小写不敏感的名称，将其全部转换为小写。对于索引和主键约束以及所有表和列名称，这已经是行为。这将允许 Alembic 自动生成脚本在最初指定为大小写不敏感时正确比较和呈现外键约束名称。

    另见

    Oracle 外键约束名称现在是“名称规范化的”

    参考：[#3276](https://www.sqlalchemy.org/trac/ticket/3276)

### 杂项

+   **[特性] [扩展]**

    添加了新标志`Session.enable_baked_queries`到`Session`以允许在会话范围内禁用烘焙查询，减少内存使用。还添加了新的`Bakery`包装器，以便通过`BakedQuery.bakery`返回的烘焙可以进行检查。

+   **[错误] [扩展]**

    在声明类被垃圾回收并且新的 automap prepare() 操作同时发生的情况下，保护不会将“None”作为类进行测试，非常少地在 gc 后没有完全处理 weakref，从而非常不频繁地命中。

    此更改还**回溯**到：1.1.10

    参考：[#3980](https://www.sqlalchemy.org/trac/ticket/3980)

+   **[错误] [扩展]**

    修复了`sqlalchemy.ext.mutable`中的一个错误，其中`Mutable.as_mutable()`方法不会跟踪使用`TypeEngine.copy()`复制的类型。这在 1.1 版本相对于 1.0 版本来说更像是一个回退，因为`TypeDecorator`类现在是`SchemaEventTarget`的子类之一，其中之一的功能是告诉父`Column`当`Column`被复制时，类型也应该被复制。在使用混入或抽象类的声明性时，这些副本很常见。

    此更改也已**回溯**到：1.1.8

    参考：[#3950](https://www.sqlalchemy.org/trac/ticket/3950)

+   **[bug] [ext]**

    对`Result.count()`方法添加了对绑定参数的支持，例如通常通过`Query.params()`设置的参数。先前，对参数的支持被省略了。感谢 Pat Deegan 提交的拉取请求。

    此更改也已**回溯**到：1.1.8

+   **[bug] [ext]**

    `AssociationProxy.any()`，`AssociationProxy.has()`和`AssociationProxy.contains()`比较方法现在支持链接到一个属性，该属性本身也是`AssociationProxy`，递归地。

    另请参阅

    AssociationProxy any()，has()，contains()可与链式关联代理一起使用

    参考：[#3769](https://www.sqlalchemy.org/trac/ticket/3769)

+   **[bug] [ext]**

    为`MutableSet`实现了就地突变运算符`__ior__`，`__iand__`，`__ixor__`和`__isub__`，以及为`MutableList`实现了`__iadd__`，这样当使用这些变异器方法来改变集合时就会触发变更事件。

    另请参阅

    就地突变运算符对 MutableSet，MutableList 起作用

    参考：[#3853](https://www.sqlalchemy.org/trac/ticket/3853)

+   **[bug] [declarative]**

    如果在要映射的类上使用了 `declared_attr.cascading` 修饰符，而不是在声明的 mixin 类或 `__abstract__` 类上使用，则会发出警告。 `declared_attr.cascading` 修饰符目前仅适用于 mixin/abstract 类。

    参考：[#3847](https://www.sqlalchemy.org/trac/ticket/3847)

+   **[bug] [ext]**

    改进了关联代理列表集合，以便在使用`list.append()`时防止对新创建的关联对象进行过早的自动刷新，并且在关联代理访问端点集合时将调用延迟加载。现在首先访问端点集合，然后再调用创建者以产生关联对象。

    参考：[#3941](https://www.sqlalchemy.org/trac/ticket/3941)

+   **[bug] [ext]**

    `sqlalchemy.ext.hybrid.hybrid_property` 类现在支持多次调用诸如 `@setter`、`@expression` 等 mutator，并且现在提供了一个 `@getter` mutator，以便可以在子类或其他类中重新使用特定的混合属性。这现在与标准 Python 中的 `@property` 的行为相匹配。

    另请参阅

    混合属性支持在子类之间重用，重新定义 @getter

    参考：[#3911](https://www.sqlalchemy.org/trac/ticket/3911), [#3912](https://www.sqlalchemy.org/trac/ticket/3912)

+   **[bug] [ext]**

    修复了`sqlalchemy.ext.serializer`扩展中的一个 bug，该 bug 导致“已注释”的 SQL 元素（由 ORM 为许多类型的 SQL 表达式生成）无法可靠地序列化。还将序列化器的默认 pickle 级别提升为“HIGHEST_PROTOCOL”。

    参考：[#3918](https://www.sqlalchemy.org/trac/ticket/3918)

### orm

+   **[orm] [功能]**

    `aliased()` 构造现在可以传递给 `Query.select_entity_from()` 方法。实体将从由 `aliased()` 构造表示的可选择项中提取。这允许与 `Query.select_entity_from()` 结合使用 `aliased()` 的特殊选项，例如 `aliased.adapt_on_names`。

    此更改也 **回溯** 到：1.1.7

    参考：[#3933](https://www.sqlalchemy.org/trac/ticket/3933)

+   **[orm] [功能]**

    向`scoped_session`添加了`.autocommit`属性，代理当前分配给线程的底层`Session`的`.autocommit`属性。感谢 Ben Fagin 的 Pull 请求。

+   **[orm] [feature]**

    添加了一个新功能`with_expression()`，允许在查询结果时向特定实体添加一个临时 SQL 表达式。这是将 SQL 表达式作为结果元组中的单独元素传递的替代方法。

    另请参阅

    可以接收临时 SQL 表达式的 ORM 属性

    参考：[#3058](https://www.sqlalchemy.org/trac/ticket/3058)

+   **[orm] [feature]**

    添加了一种新的映射器级继承加载方式“多态 selectin”。这种加载方式在加载基本对象类型后，为继承��次结构中的每个子类发出查询，使用 IN 指定所需的主键值。

    另请参阅

    “selectin”多态加载，使用单独的 IN 查询加载子类

    参考：[#3948](https://www.sqlalchemy.org/trac/ticket/3948)

+   **[orm] [feature]**

    添加了一种新的急切加载方式，称为“selectin”加载。这种加载方式与“subquery”急切加载非常相似，只是它使用了一个 IN 表达式，给出了加载的父对象的主键值列表，而不是重新声明原始查询。这产生了一个更有效的查询，是“烘焙”（例如，SQL 字符串被缓存），并且也适用于`Query.yield_per()`的上下文。

    另请参阅

    新的“selectin”急切加载，使用 IN 一次加载所有集合

    参考：[#3944](https://www.sqlalchemy.org/trac/ticket/3944)

+   **[orm] [feature]**

    `lazy="select"`加载策略现在在所有情况下都使用`BakedQuery`查询缓存系统。这消除了生成`Query`对象并将其运行到`select()`和然后从懒加载相关集合和对象的过程中生成字符串 SQL 语句的大部分开销。 “烘焙”懒加载器也已经改进，以便在大多数情况下可以缓存查询加载选项的情况。

    另请参阅

    “烘焙”加载现在是延迟加载的默认设置

    参考：[#3954](https://www.sqlalchemy.org/trac/ticket/3954)

+   **[orm] [feature] [ext]**

    `Query.update()`方法现在可以同时处理混合属性和复合属性作为放置在 SET 子句中的键的来源。对于混合属性，还提供了额外的装饰器`hybrid_property.update_expression()`，用户提供一个返回元组的函数。

    另请参阅

    支持混合、复合类型的批量更新

    参考：[#3229](https://www.sqlalchemy.org/trac/ticket/3229)

+   **[orm] [feature]**

    添加了新的属性事件`AttributeEvents.bulk_replace()`。当将集合分配给关系时，触发此事件，在比较传入集合与现有集合之前触发。此早期事件还允许转换传入的非 ORM 对象。该事件与`@validates`装饰器集成。

    另请参阅

    新的 bulk_replace 事件

    参考：[#3896](https://www.sqlalchemy.org/trac/ticket/3896)

+   **[orm] [feature]**

    添加了新的事件处理程序`AttributeEvents.modified()`，当调用`func:.attributes.flag_modified`函数时触发，使用`sqlalchemy.ext.mutable`扩展模块时通常会触发。

    另请参阅

    sqlalchemy.ext.mutable 的新“modified”事件处理程序

    参考：[#3303](https://www.sqlalchemy.org/trac/ticket/3303)

+   **[orm] [bug]**

    修复了子查询急加载的问题，该问题继续于[#2699](https://www.sqlalchemy.org/trac/ticket/2699)、[#3106](https://www.sqlalchemy.org/trac/ticket/3106)、[#3893](https://www.sqlalchemy.org/trac/ticket/3893)修复的系列问题中，涉及“子查询”包含正确的 FROM 子句，当从连接的继承子类开始，然后从基类的关系上进行子查询急加载，并且查询还包括对子类的条件。之前票据中的修复未适应从第一级更深层次加载更多的子查询操作，因此修复已进一步概括。

    此更改也**被移植**到：1.1.11

    参考：[#4011](https://www.sqlalchemy.org/trac/ticket/4011)

+   **[orm] [bug]**

    修复了级联操作（例如“delete-orphan”等）无法定位到链接到继承关系中的子类的关系的对象的错误，从而导致操作未执行的错误。

    此更改也**被移植**到：1.1.10

    参考文献：[#3986](https://www.sqlalchemy.org/trac/ticket/3986)

+   **[orm] [bug]**

    修复了在由 [#3915](https://www.sqlalchemy.org/trac/ticket/3915) 添加的缓存造成的线程环境下可能发生的竞态条件。一个内部的 `Column` 对象集合可能会在错误的时候在别名对象上重新生成，当尝试渲染 SQL 并收集结果时，会混淆连接的 eager loader，从而导致属性错误。现在，在别名对象被缓存和在线程之间共享之前，该集合现在会被提前生成。

    此更改也已**回溯**到：1.1.7

    参考文献：[#3947](https://www.sqlalchemy.org/trac/ticket/3947)

+   **[orm] [bug]**

    作为 `relationship.post_update` 功能的结果发出的 UPDATE 现在将与版本控制功能集成，既会提升行的版本号，也会断言现有版本号已匹配。

    另见

    post_update 与 ORM 版本控制集成

    参考文献：[#3496](https://www.sqlalchemy.org/trac/ticket/3496)

+   **[orm] [bug]**

    修复了几个用例，涉及 `relationship.post_update` 功能与具有“onupdate”值的列结合使用时。当 UPDATE 发出时，相应的对象属性现在将被过期或刷新，以便新生成的“onupdate”值可以填充到对象中；以前，旧值将保留。此外，如果在对象的 INSERT 中设置了目标属性，则在 UPDATE 期间现在会重新发送该值，以便“onupdate”不会覆盖它（请注意，这对于服务器生成的 onupdates 也同样有效）。最后，在刷新期间刷新这些属性时，现在会触发 `SessionEvents.refresh_flush()` 事件。

    另见

    与 onupdate 结合使用的 post_update 的细化

    参考文献：[#3471](https://www.sqlalchemy.org/trac/ticket/3471)，[#3472](https://www.sqlalchemy.org/trac/ticket/3472)

+   **[orm] [bug]**

    修复了一个 bug，在程序化版本 _id 计数器与联合表继承结合使用时，如果版本 _id 计数器实际上未递增且未修改基表上的任何其他值，则会失败，因为 UPDATE 将具有一个空的 SET 子句。由于程序化版本 _id 的版本计数器未递增是一个记录的用例，因此现在会检测到此特定条件，并且 UPDATE 现在将版本 _id 值设置为自身，以便仍然进行并发检查。

    参考文献：[#3996](https://www.sqlalchemy.org/trac/ticket/3996)

+   **[orm] [bug]**

    版本功能不支持版本计数器为 NULL。如果版本 id 是程序化的，并且在 UPDATE 时设置为 NULL，则现在会引发异常。感谢 Diana Clarke 的 Pull 请求。

    参考：[#3673](https://www.sqlalchemy.org/trac/ticket/3673)

+   **[orm] [bug]**

    从`scoped_session`中移除了一个非常古老的关键字参数`scope`。这个关键字从未被记录在文档中，是早期允许变量作用域的尝试。

    参见

    从 scoped_session 中移除“scope”关键字

    参考：[#3796](https://www.sqlalchemy.org/trac/ticket/3796)

+   **[orm] [bug]**

    修复了一个 bug，即在与指定了 innerjoin=True 的 subclass-linked 关系一起组合“with_polymorphic”加载时，会失败将这些“innerjoins”降级为“outerjoins”以适应不支持该关系的其他多态类。这适用于单个和连接继承多态加载。

    参考：[#3988](https://www.sqlalchemy.org/trac/ticket/3988)

+   **[orm] [bug]**

    为`Session.refresh()`方法添加了新参数`with_for_update`。当`Query.with_lockmode()`方法被弃用，改用`Query.with_for_update()`时，`Session.refresh()`方法从未更新以反映新选项。

    参见

    为 Session.refresh 添加“for update”参数

    参考：[#3991](https://www.sqlalchemy.org/trac/ticket/3991)

+   **[orm] [bug]**

    修复了一个 bug，即`column_property()`同时标记为“deferred”时，在刷新期间会被标记为“expired”，导致它与常规属性一起加载，即使从未访问过该属性。

    参考：[#3984](https://www.sqlalchemy.org/trac/ticket/3984)

+   **[orm] [bug]**

    修复了子查询急加载中的一个 bug，即对于自引用关系，“join_depth”参数不会被正确遵守，会加载所有可用的深度，而不是正确计算急加载的指定级别数。

    参考：[#3967](https://www.sqlalchemy.org/trac/ticket/3967)

+   **[orm] [bug]**

    已向`Mapper`（最终也将用于其他基于 ORM 的 LRU 缓存）的 LRU“编译缓存”添加了警告，以便当缓存开始达到其大小限制时，应用程序会发出警告，指出这是一种可能需要关注的降低性能的情况。如果应用程序正在使用无限数量的`Engine`对象，则 LRU 缓存主要可以达到其大小限制，这是一种反模式。否则，这可能表明存在应该引起 SQLAlchemy 开发人员注意的问题。

+   **[orm] [bug]**

    修复了 bug，以改善对在延迟加载相关实体后生效的加载器选项的特异性，以便如果这些选项包括实体信息，则加载器选项将更具体地匹配到别名或非别名实体。

    参考：[#3963](https://www.sqlalchemy.org/trac/ticket/3963)

+   **[orm] [bug]**

    函数`flag_modified()`现在如果对象中不存在指定的属性键，则会引发`InvalidRequestError`异常，因为在刷新过程中假定该键已存在。要在不引用任何特定属性的情况下标记对象为“脏”以进行刷新，可以使用函数`flag_dirty()`。

    请参见

    使用 flag_dirty()将对象标记为“脏”而不更改任何属性

    参考：[#3753](https://www.sqlalchemy.org/trac/ticket/3753)

+   **[orm] [bug]**

    由`Query.update()`和`Query.delete()`使用的“评估”策略现在可以容纳从多对一关系到实例的简单对象比较，当主键/外键列的属性名称与列的实际名称不匹配时。以前，这将进行简单的基于名称的匹配，并在 AttributeError 时失败。

    参考：[#3366](https://www.sqlalchemy.org/trac/ticket/3366)

+   **[orm] [bug]**

    `@validates`装饰器现在允许装饰的方法接收来自“批量集合设置”操作的对象，这些对象尚未与现有集合进行比较。这允许将传入的值转换为兼容的 ORM 对象，就像从“追加”事件中已经允许的那样。请注意，这意味着在集合分配期间，将调用**所有**值的`@validates`方法，而不仅仅是新的值。

    请参见

    在比较之前，@validates 方法会在批量集合设置时接收所有值

    参考：[#3896](https://www.sqlalchemy.org/trac/ticket/3896)

+   **[orm] [bug]**

    修复了单表继承中的 bug，当限制行到子类时，`select_from()` 参数将不会被考虑。以前，只有请求的列中的表达式会被考虑。

    请参阅

    修复了与 select_from() 一起使用的单表继承问题

    参考：[#3891](https://www.sqlalchemy.org/trac/ticket/3891)

+   **[orm] [bug]**

    当将集合分配给由关系映射的属性时，以前的集合不再发生变化。以前，旧集合会随着“项删除”事件的触发而清空；现在事件会在不影响旧集合的情况下触发。

    请参阅

    替换时以前的集合不再发生变化

    参考：[#3913](https://www.sqlalchemy.org/trac/ticket/3913)

+   **[orm] [bug]**

    当`Session`的状态在发出`SessionEvents.after_rollback()`事件时现在存在，也就是说，在对象过期之前的属性状态。这与发出`SessionEvents.after_commit()`事件的行为一致，该事件也会在对象的属性状态过期之前发出。

    请参阅

    在对象过期之前，`after_rollback()` 会话事件现在会发出

    参考：[#3934](https://www.sqlalchemy.org/trac/ticket/3934)

+   **[orm] [bug]**

    修复了`Query.with_parent()`在`Query`针对`aliased()`构造而不是常规映射类时无法工作的 bug。还为独立的`with_parent()`函数以及`Query.with_parent()`添加了一个新参数`with_parent.from_entity`。

    参考：[#3607](https://www.sqlalchemy.org/trac/ticket/3607)

### orm 声明

+   **[orm] [declarative] [bug]**

    修复了一个 bug，在 `AbstractConcreteBase` 上使用 `declared_attr`，其中特定的返回值是一些非映射符号，包括 `None`，会导致属性只会硬评估一次并将值存储到对象字典中，不允许它为子类调用。当 `declared_attr` 在一个映射类上时，这种行为是正常的，在混合类或抽象类上不会发生。由于 `AbstractConcreteBase` 既是“抽象的”又实际上是“映射的”，所以这里特殊地做了一个异常情况，以便“抽象”的行为优先于 `declared_attr`。

    参考：[#3848](https://www.sqlalchemy.org/trac/ticket/3848)

### 引擎

+   **[引擎] [功能]**

    向 `Pool` 对象添加了本机“悲观断开连接”处理。新参数 `Pool.pre_ping`，可从引擎中作为 `create_engine.pool_pre_ping` 获得，应用了在连接池文档中特色的“预先 ping”配方的有效形式，每次连接检出时，都会发出一个简单的语句，通常是“SELECT 1”，以测试连接的活动性。如果现有连接不再能够响应命令，则连接将被透明地回收，并且在当前时间戳之前进行的所有其他连接都将被作废。

    另请参阅

    断开连接处理 - 悲观

    对连接池添加了悲观断开连接检测

    参考：[#3919](https://www.sqlalchemy.org/trac/ticket/3919)

+   **[引擎] [bug]**

    添加了一个异常处理程序，当 `Connection` 的“autorollback”功能本身引发异常时，将会警告“原因”异常。在 Py3K 中，两个异常自然由解释器报告为一个在处理另一个时发生。这是继续上次在 1.0.12 中访问的回滚失败处理的系列更改的一部分。

    此更改也**回溯**到：1.1.7

    参考：[#3946](https://www.sqlalchemy.org/trac/ticket/3946)

+   **[引擎] [bug]**

    修复了一个 bug，在罕见情况下，直接将`Compiled`对象传递给`Connection.execute()`时，生成`Compiled`对象的方言未被咨询以获取字符串语句的 paramstyle，而是假设它将匹配方言级别的 paramstyle，导致不匹配发生。

    参考：[#3938](https://www.sqlalchemy.org/trac/ticket/3938)

### sql

+   **[sql] [feature]**

    添加了一种名为“expanding”的新类型的`bindparam()`。这是用于`IN`表达式中的，其中元素列表在语句执行时被渲染为单独的绑定参数，而不是在语句编译时。这允许将单个绑定参数名称链接到多个元素的 IN 表达式，同时也允许在 IN 表达式中使用查询缓存。这一新功能允许“select in”加载和“polymorphic in”加载相关功能利用烘焙查询扩展以减少调用开销。这一功能应被视为**实验性**的 1.2 版本。

    另请参阅

    延迟扩展的 IN 参数集允许具有缓存语句的 IN 表达式

    参考：[#3953](https://www.sqlalchemy.org/trac/ticket/3953)

+   **[sql] [feature] [mysql] [oracle] [postgresql]**

    添加了对`Table`和`Column`对象上的 SQL 注释的支持，通过新的`Table.comment`和`Column.comment`参数。这些注释包含在表创建的 DDL 中，可以是内联的，也可以通过适当的 ALTER 语句，同时也会在表反射中反映回来，以及通过`Inspector`。当前支持的后端包括 MySQL、PostgreSQL 和 Oracle。非常感谢 Frazer McLean 在这方面的大量努力。

    另请参阅

    支持在表、列上添加 SQL 注释，包括 DDL、反射

    参考：[#1546](https://www.sqlalchemy.org/trac/ticket/1546)

+   **[sql] [feature]**

    当右侧条件为空序列时，`ColumnOperators.in_()`和`ColumnOperators.notin_()`操作符发出警告的长期行为已经修订；现在默认情况下会呈现一个简单的“静态”表达式“1 != 1”或“1 = 1”，而不是引入原始的左侧表达式。这导致对空集合进行 NULL 列比较的结果从 NULL 更改为 true/false。该行为是可配置的，并且可以使用`create_engine()`的`create_engine.empty_in_strategy`参数启用旧行为。

    参见

    IN / NOT IN 运算符的空集合行为现在是可配置的；默认表达式简化

    参考：[#3907](https://www.sqlalchemy.org/trac/ticket/3907)

+   **[sql] [feature]**

    为“startswith”和“endswith”比较器的类添加了一个新选项`autoescape`；这提供了一个转义字符，同时自动应用于所有通配符字符“%”和“_”的所有出现。感谢戴安娜·克拉克的拉取请求。

    注意

    该功能已从其在 1.2.0b2 中的初始实现中更改为 1.2.0，使得 autoescape 现在作为布尔值传递，而不是作为要用作转义字符的特定字符。

    参见

    新的“autoescape”选项用于 startswith()，endswith()

    参考：[#2694](https://www.sqlalchemy.org/trac/ticket/2694)

+   **[sql] [bug]**

    修复了在`WithinGroup`结构迭代期间可能发生的 AttributeError。

    此更改也被**回溯**到：1.1.11

    参考：[#4012](https://www.sqlalchemy.org/trac/ticket/4012)

+   **[sql] [bug]**

    由于[#3859](https://www.sqlalchemy.org/trac/ticket/3859)导致的 1.1.5 中发布的回归已修复，根据`Variant`对表达式的“右侧”评估进行调整，以遵守底层类型的“右侧”规则，导致`Variant`类型在那些我们*确实*希望左侧类型直接传递到右侧以便将绑定级规则应用于表达式参数的情况下不当地丢失。

    此更改也被**回溯**到：1.1.9

    参考：[#3952](https://www.sqlalchemy.org/trac/ticket/3952)

+   **[sql] [bug] [postgresql]**

    改变了`ResultProxy`的机制，无条件地推迟“autoclose”步骤，直到`Connection`完成对象处理；在 PostgreSQL ON CONFLICT with RETURNING 返回零行的情况下，以前不存在的使用情况中，会发生自动关闭，导致在以前不存在的情况下进行的 INSERT/UPDATE/DELETE 操作的通常自动提交行为失败。

    此更改也**回溯到**：1.1.9

    参考：[#3955](https://www.sqlalchemy.org/trac/ticket/3955)

+   **[sql] [bug]**

    现在关于`Numeric`、`Integer`和与日期相关的类型之间的类型强制转换规则现在包括额外的逻辑，该逻辑将尝试保留“resolved”类型的传入类型的设置。当前，这个目标是`asdecimal`标志，因此，`Numeric`或`Float`与`Integer`之间的数学运算将保留`asdecimal`标志以及类型是否应该是`Float`子类。

    另见

    强化了“浮点数”数据类型的类型强度

    参考：[#4018](https://www.sqlalchemy.org/trac/ticket/4018)

+   **[sql] [bug] [mysql]**

    现在，`Float`类型的结果处理器现在无条件地通过`float()`处理器运行值，如果方言还支持“本地十进制”模式的话。虽然大多数后端会为浮点数据类型提供 Python `float`对象，但是 MySQL 后端在某些情况下缺少类型信息以提供此类信息，并且除非进行浮点转换，否则返回`Decimal`。

    另见

    强化了“浮点数”数据类型的类型强度

    参考：[#4020](https://www.sqlalchemy.org/trac/ticket/4020)

+   **[sql] [bug]**

    对传递给 SQL 语句的 Python“浮点数”值的处理增加了一些额外的严格性。一个“浮点数”值将与`Float`数据类型关联，而不是以前的 Decimal-coercing `Numeric`数据类型，这消除了 SQLite 上发出的令人困惑的警告以及不必要的转换为 Decimal 的情况。

    另见

    强化了“浮点数”数据类型的类型强度

    参考：[#4017](https://www.sqlalchemy.org/trac/ticket/4017)

+   **[sql] [bug]**

    所有比较操作符（如 LIKE、IS、IN、MATCH、等于、大于、小于等）的运算符优先级已经合并为一个级别，因此使用这些操作符相互比较的表达式将在它们之间产生括号。这适用于像 Oracle、MySQL 等数据库的声明的运算符优先级，这些数据库将所有这些操作符视为相等优先级，以及 PostgreSQL 截至 9.5 版本也已经将其运算符优先级扁平化。

    另请参阅

    比较操作符的扁平化运算符优先级

    参考：[#3999](https://www.sqlalchemy.org/trac/ticket/3999)

+   **[sql] [bug]**

    修复了使用`ColumnOperators.is_()`或类似操作符的表达式的类型不会是“boolean”类型的问题，而是类型将是“nulltype”，以及当使用自定义比较操作符对未类型化表达式进行比较时。这种类型化可能会影响表达式在更大上下文中以及在结果行处理中的行为。

    参考：[#3873](https://www.sqlalchemy.org/trac/ticket/3873)

+   **[sql] [bug]**

    修复了对`Label`构造的否定，使得当应用`not_()`修饰符到标记表达式时，内部元素被正确否定。

    参考：[#3969](https://www.sqlalchemy.org/trac/ticket/3969)

+   **[sql] [bug]**

    SQL 语句中百分号“加倍”以进行转义目的的系统已经得到改进。百分号“加倍”主要与`literal_column`构造以及`ColumnOperators.contains()`等操作符相关联，现在根据正在使用的 DBAPI 的声明的 paramstyle 进行；对于像 PostgreSQL 和 MySQL 驱动程序常见的百分号敏感 paramstyles，将会发生加倍，对于 SQLite 等其他驱动程序则不会。这使得更多与数据库无关的使用`literal_column`构造成为可能。

    另请参阅

    在 literal_column()中的百分号现在有条件地转义

    参考：[#3740](https://www.sqlalchemy.org/trac/ticket/3740)

+   **[sql] [bug]**

    修复了一个 bug，在列级`CheckConstraint`中，如果 sqltext 是一个 Core 表达式而不仅仅是一个普通字符串，将无法使用底层方言编译器编译 SQL 表达式并应用适当的标志以生成内联的字面值。这在 0.9 版本中已经很久以前修复了表级别的检查约束，作为[#2742](https://www.sqlalchemy.org/trac/ticket/2742)的一部分，更常见的是使用 Core SQL 表达式而不是普通字符串表达式。

    参考：[#3957](https://www.sqlalchemy.org/trac/ticket/3957)

+   **[SQL] [错误]**

    修复了一个 bug，在“预执行”代码路径中，如果 SQL 本身是一个未分类的表达式，比如纯文本，那么 SQL 导向的 Python 端列默认值可能在 INSERT 时无法正确执行。然而，“预执行”代码路径相当罕见，但在不使用 RETURNING 时，可以应用于具有 SQL 默认值的非整数主键列。

    参考：[#3923](https://www.sqlalchemy.org/trac/ticket/3923)

+   **[SQL] [错误]**

    由列级`collate()`和`ColumnOperators.collate()`呈现的用于 COLLATE 的表达式现在在名称区分大小写时被引用为标识符，例如包含大写字符。请注意，这不会影响类型级别的排序，因为它已经被引用。

    另请参见

    列级别的 COLLATE 关键字现在引用排序名称

    参考：[#3785](https://www.sqlalchemy.org/trac/ticket/3785)

+   **[SQL] [错误]**

    修复了一个 bug，在列上下文中使用`Alias`对象会在尝试将自身分组到括号表达式中时引发参数错误。目前还不是完全支持的 API，但这种方式的使用适用于一些最终用户的示例，并且可能在支持一些未来的 PostgreSQL 功能时发挥更重要的作用。

    参考：[#3939](https://www.sqlalchemy.org/trac/ticket/3939)

### 模式

+   **[模式] [错误]**

    如果使用不匹配的“本地”和“远程”列创建了一个`ForeignKeyConstraint`对象，现在会引发一个`ArgumentError`，否则会导致约束的内部状态不正确。请注意，这也会影响方言的反射过程产生的外键约束的列集不匹配的情况。

    此更改还**回溯**到：1.1.10

    参考文献：[#3949](https://www.sqlalchemy.org/trac/ticket/3949)

### postgresql

+   **[postgresql] [bug]**

    继续修复正确处理 PostgreSQL 版本字符串“10devel”，在 1.1.8 中发布，进一步提升正则表达式以处理形式为“10beta1”的版本字符串。虽然 PostgreSQL 现在提供了更好的获取此信息的方法，但我们至少在 1.1.x 中仍然坚持使用正则表达式，以便与较旧或替代的 PostgreSQL 数据库的兼容性风险最小化。

    此更改还**回溯**到：1.1.11

    参考文献：[#4005](https://www.sqlalchemy.org/trac/ticket/4005)

+   **[postgresql] [bug]**

    修复了使用带有排序规则的字符串类型的 `ARRAY` 在 CREATE TABLE 中无法产生正确语法的错误。

    此更改还**回溯**到：1.1.11

    参考文献：[#4006](https://www.sqlalchemy.org/trac/ticket/4006)

+   **[postgresql] [bug]**

    为 GRANT、REVOKE 关键字添加“autocommit”支持。拉取请求由 Jacob Hayes 提供。

    此更改还**回溯**到：1.1.10

+   **[postgresql] [bug]**

    添加了解析 PostgreSQL 版本字符串的支持，例如“PostgreSQL 10devel”这样的开发版本。拉取请求由 Sean McCully 提供。

    此更改还**回溯**到：1.1.8

+   **[postgresql] [bug]**

    修复了基本 `ARRAY` 数据类型不会调用 `ARRAY` 的绑定/结果处理器的错误。

    参考文献：[#3964](https://www.sqlalchemy.org/trac/ticket/3964)

+   **[postgresql] [bug]**

    添加对 PostgreSQL `INTERVAL` 数据类型的所有可能“fields”标识符的支持，例如“YEAR”、“MONTH”、“DAY TO MINUTE”等。此外，`INTERVAL` 数据类型本身现在包含一个新参数 `INTERVAL.fields`，可以在其中指定这些限定符；限定符也会在反射/检查后反映到结果数据类型中。

    参见

    对 INTERVAL 中字段规范的支持，包括完全反射

    参考文献：[#3959](https://www.sqlalchemy.org/trac/ticket/3959)

### mysql

+   **[mysql] [feature]**

    添加了对 MySQL 的 ON DUPLICATE KEY UPDATE MySQL 特定的 `Insert` 对象的支持。拉取请求由 Michael Doronin 提供。

    参见

    对 INSERT..ON DUPLICATE KEY UPDATE 的支持

    参考文献：[#4009](https://www.sqlalchemy.org/trac/ticket/4009)

+   **[mysql] [bug]**

    MySQL 5.7 引入了对“SHOW VARIABLES”命令的权限限制；MySQL 方言现在将处理当 SHOW 返回没有行时的情况，特别是对于 SQL_MODE 的初始获取，并将发出警告，提示用户权限应该被修改以允许该行存在。

    此更改也被**回溯**到：1.1.11

    参考：[#4007](https://www.sqlalchemy.org/trac/ticket/4007)

+   **[mysql] [bug]**

    移除了对 UTC_TIMESTAMP MySQL 函数的古老且不必要的拦截，这妨碍了使用带参数的函数。

    此更改也被**回溯**到：1.1.10

    参考：[#3966](https://www.sqlalchemy.org/trac/ticket/3966)

+   **[mysql] [bug]**

    修复了 MySQL 方言中关于在渲染 CREATE TABLE 时与 PARTITION 选项一起渲染表选项的错误。PARTITION 相关选项需要跟随表选项，而以前这种顺序没有被强制执行。

    此更改也被**回溯**到：1.1.10

    参考：[#3961](https://www.sqlalchemy.org/trac/ticket/3961)

+   **[mysql] [bug]**

    添加了对由于过时表定义而无法反射的视图的支持，当调用`MetaData.reflect()`时，对于无法响应`DESCRIBE`的表会发出警告，但操作成功。

    参考：[#3871](https://www.sqlalchemy.org/trac/ticket/3871)

### mssql

+   **[mssql] [bug]**

    修复了在使用 Azure 数据仓库时必须从不同视图中获取 SQL Server 事务隔离的错误，现在尝试针对两个视图执行查询，如果失败继续提供对未来新 SQL Server 版本中任意 API 更改的最佳弹性，则无条件引发 NotImplemented。

    此更改也被**回溯**到：1.1.11

    参考：[#3994](https://www.sqlalchemy.org/trac/ticket/3994)

+   **[mssql] [bug]**

    在 SQL Server 方言中添加了一个占位符类型`XML`，以便包含此类型的反射表可以重新呈现为 CREATE TABLE。该类型没有特殊的往返行为，也不支持额外的限定参数。

    此更改也被**回溯**到：1.1.11

    参考：[#3973](https://www.sqlalchemy.org/trac/ticket/3973)

+   **[mssql] [bug]**

    SQL Server 方言现在允许在其中使用点的数据库和/或所有者名称，使用字符串中明确的括号围绕所有者和可选的数据库名称。此外，将`quoted_name`构造发送到模式名称将不会在点上拆分，并将提供完整字符串作为“所有者”。`quoted_name`现在也可以从`sqlalchemy.sql`导入空间中使用。

    另请参阅

    支持带有嵌入点的 SQL Server 模式名称

    参考：[#2626](https://www.sqlalchemy.org/trac/ticket/2626)

### oracle

+   **[oracle] [feature] [postgresql]**

    向`Sequence`添加了新关键字`Sequence.cache`和`Sequence.order`，以允许渲染 Oracle 和 PostgreSQL 理解的 CACHE 参数，以及 Oracle 理解的 ORDER 参数。感谢 David Moore 的拉取请求。

    此更改也**回溯**到：1.1.12

+   **[oracle] [feature]**

    当使用`Inspector.get_unique_constraints()`、`Inspector.get_check_constraints()`时，Oracle 方言现在会检查唯一约束和检查约束。由于 Oracle 没有与唯一`Index`分开的唯一约束，因此反射的`Table`仍将继续不与`UniqueConstraint`对象关联。感谢 Eloy Felix 的拉取请求。

    另请参阅

    Oracle 唯一约束，检查约束现在反映

    参考：[#4003](https://www.sqlalchemy.org/trac/ticket/4003)

+   **[oracle] [bug]**

    当使用版本为 6.0b1 或更高的 DBAPI 时，cx_Oracle 完全删除了对两阶段事务的支持。在任何情况下，cx_Oracle 5.x 历史上从未能够使用两阶段功能，而 cx_Oracle 6.x 已经删除了此功能所依赖的连接级“twophase”标志。

    此更改也**回溯**到：1.1.11

    参考：[#3997](https://www.sqlalchemy.org/trac/ticket/3997)

+   **[oracle] [bug]**

    修复了 cx_Oracle 方言中的错误，版本字符串解析会因为 cx_Oracle 版本 6.0b1 中的“b”字符而失败。现在版本字符串解析使用正则表达式而不是简单的分割。

    此更改也被**回溯**到：1.1.10

    参考文献：[#3975](https://www.sqlalchemy.org/trac/ticket/3975)

+   **[oracle] [错误]**

    cx_Oracle 方言现在支持“合理的多行计数”，即，当通过 DBAPI `cursor.executemany()` 执行一系列参数集时，我们可以利用 `cursor.rowcount` 来验证匹配的行数。这在 ORM 中对于检测并发修改方案有影响，因为在批处理语句时，一些简单的条件现在可以被检测到，同时当使用更严格的版本功能时，ORM 仍然可以使用语句批处理。假设至少版本 5.0，现在 cx_Oracle 默认启用该标志，这已经是司空见惯的了。

    参考文献：[#3932](https://www.sqlalchemy.org/trac/ticket/3932)

+   **[oracle] [错误]**

    Oracle 反射现在“规范化”了给定的外键约束名称，即，对于大小写不敏感的名称，将其返回为全部小写。这已经是索引和主键约束以及所有表和列名称的行为了。这将允许 Alembic 自动生成的脚本在最初指定为大小写不敏感时正确比较和渲染外键约束名称。

    请参见

    Oracle 外键约束名称现在已“规范化”

    参考文献：[#3276](https://www.sqlalchemy.org/trac/ticket/3276)

### 杂项

+   **[特性] [扩展]**

    添加了新标志 `Session.enable_baked_queries` 到 `Session` 中，以允许全局禁用烘焙查询，减少内存使用。还添加了新的 `Bakery` 包装器，以便可以检查由 `BakedQuery.bakery` 返回的烘焙器。

+   **[错误] [扩展]**

    在声明性类被垃圾收集并且新的 automap prepare() 操作同时进行的情况下，防止测试“None”作为一个类，非常不经常地命中一个在垃圾回收后尚未完全处理的 weakref。

    此更改也被**回溯**到：1.1.10

    参考文献：[#3980](https://www.sqlalchemy.org/trac/ticket/3980)

+   **[错误] [扩展]**

    修复了`sqlalchemy.ext.mutable`中的错误，在使用`TypeEngine.copy()`复制类型时，`Mutable.as_mutable()`方法不会跟踪已复制的类型。在 1.1 中相比于 1.0，这变得更加退化，因为`TypeDecorator`类现在是`SchemaEventTarget`的子类，其中之一是当`Column`被复制时指示类型也应该被复制的内容。在使用混合或抽象类与声明式一起使用时，这些副本很常见。

    此更改也已**回溯**至：1.1.8

    参考：[#3950](https://www.sqlalchemy.org/trac/ticket/3950)

+   **[错误] [扩展]**

    添加了对绑定参数的支持，例如通常通过`Query.params()`设置的参数，到`Result.count()`方法中。先前，参数的支持被省略了。感谢 Pat Deegan 提供的拉取请求。

    此更改也已**回溯**至：1.1.8

+   **[错误] [扩展]**

    `AssociationProxy.any()`、`AssociationProxy.has()`和`AssociationProxy.contains()`比较方法现在支持链接到本身也是`AssociationProxy`的属性，递归地。

    另请参阅

    `AssociationProxy.any()`、`has()`、`contains()`与链接的关联代理一起工作

    参考：[#3769](https://www.sqlalchemy.org/trac/ticket/3769)

+   **[错误] [扩展]**

    对于`MutableSet`实施就地变异运算符`__ior__`、`__iand__`、`__ixor__`和`__isub__`，以及对`MutableList`实施`__iadd__`，以便在使用这些变异器方法修改集合时触发更改事件。

    另请参阅

    就地变异运算符对 MutableSet、MutableList 有效

    参考：[#3853](https://www.sqlalchemy.org/trac/ticket/3853)

+   **[错误] [声明式]**

    当使用`declared_attr.cascading`修饰符与一个在要映射的类上自身声明的声明性属性一起使用时，会发出警告，而不是在声明性混入类或`__abstract__`类上声明的属性。`declared_attr.cascading`修饰符目前仅适用于混入/抽象类。

    参考：[#3847](https://www.sqlalchemy.org/trac/ticket/3847)

+   **[bug] [ext]**

    改进了关联代理列表集合，以防止对新创建的关联对象进行过早的自动刷新，当使用`list.append()`时，当关联代理访问端点集合时会调用延迟加载。现在在调用创建者以生成关联对象之前首先访问端点集合。

    参考：[#3941](https://www.sqlalchemy.org/trac/ticket/3941)

+   **[bug] [ext]**

    `sqlalchemy.ext.hybrid.hybrid_property`类现在支持多次调用像`@setter`，`@expression`等的变异器，以及现在提供了一个`@getter`变异器，这样一个特定的混合属性可以在子类或其他类之间重新使用。这现在与标准 Python 中`@property`的行为相匹配。

    另请参见

    混合属性支持在子类之间重用，重新定义@getter

    参考：[#3911](https://www.sqlalchemy.org/trac/ticket/3911)，[#3912](https://www.sqlalchemy.org/trac/ticket/3912)

+   **[bug] [ext]**

    修复了`sqlalchemy.ext.serializer`扩展中的一个 bug，即“注释”SQL 元素（由 ORM 为许多类型的 SQL 表达式生成）无法可靠地序列化。还将序列化器的默认 pickle 级别提升为“HIGHEST_PROTOCOL”。

    参考：[#3918](https://www.sqlalchemy.org/trac/ticket/3918)
