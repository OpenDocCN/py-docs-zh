# 核心内部

> 原文：[`docs.sqlalchemy.org/en/20/core/internals.html`](https://docs.sqlalchemy.org/en/20/core/internals.html)

这里列出了一些关键的内部结构。

| 对象名称 | 描述 |
| --- | --- |
| AdaptedConnection | 支持 DBAPI 协议的适配连接对象的接口。 |
| BindTyping | 定义了在语句中传递绑定参数的不同方法以传递到数据库驱动程序。 |
| Compiled | 表示编译的 SQL 或 DDL 表达式。 |
| DBAPIConnection | 表示 [**PEP 249**](https://peps.python.org/pep-0249/) 数据库连接的协议。 |
| DBAPICursor | 表示 [**PEP 249**](https://peps.python.org/pep-0249/) 数据库游标的协议。 |
| DBAPIType | 表示 [**PEP 249**](https://peps.python.org/pep-0249/) 数据库类型的协议。 |
| DDLCompiler |  |
| DefaultDialect | Dialect 的默认实现 |
| DefaultExecutionContext |  |
| Dialect | 定义特定数据库和 DB-API 组合的行为。 |
| ExecutionContext | 用于对应于单个执行的 Dialect 的信使对象。 |
| ExpandedState | 表示在为语句生成“扩展”和“编译后”绑定参数时要使用的状态。 |
| GenericTypeCompiler |  |
| Identified |  |
| IdentifierPreparer | 根据选项处理标识符的引用和大小写折叠。 |
| SQLCompiler | `Compiled` 的默认实现。 |
| StrSQLCompiler | `SQLCompiler` 的子类，允许将一小部分非标准 SQL 特性渲染为字符串值。 |

```py
class sqlalchemy.engine.BindTyping
```

定义了在语句中传递绑定参数的不同方法以传递到数据库驱动程序。

从版本 2.0 开始。

**成员**

NONE, RENDER_CASTS, SETINPUTSIZES

**类签名**

类`sqlalchemy.engine.BindTyping` (`enum.Enum`) 

```py
attribute NONE = 1
```

没有采取任何步骤将类型信息传递给数据库驱动程序。

这是 SQLite、MySQL/MariaDB、SQL Server 等数据库的默认行为。

```py
attribute RENDER_CASTS = 3
```

在 SQL 字符串中渲染转换或其他指令。

此方法适用于所有 PostgreSQL 方言，包括 asyncpg、pg8000、psycopg、psycopg2。实现此方法的方言可以选择在 SQL 语句中明确转换哪些类型的数据类型，哪些不转换。

使用 RENDER_CASTS 时，编译器将为渲染的字符串表示形式中的每个 `BindParameter` 对象调用 `SQLCompiler.render_bind_cast()` 方法，该对象的方言级类型设置了 `TypeEngine.render_bind_cast` 属性。

`SQLCompiler.render_bind_cast()` 也用于渲染一种形式的 “insertmanyvalues” 查询的转换，当同时设置了 `InsertmanyvaluesSentinelOpts.USE_INSERT_FROM_SELECT` 和 `InsertmanyvaluesSentinelOpts.RENDER_SELECT_COL_CASTS` 时，转换应用于中间列，例如 “INSERT INTO t (a, b, c) SELECT p0::TYP, p1::TYP, p2::TYP ” “FROM (VALUES (?, ?), (?, ?), …)”。

2.0.10 版本中的新内容：- 现在在 “insertmanyvalues” 实现的某些部分中使用了 `SQLCompiler.render_bind_cast()`。

```py
attribute SETINPUTSIZES = 2
```

使用 pep-249 的 setinputsizes 方法。

仅对支持此方法的 DBAPIs 实现了此功能，并且对于 SQLAlchemy 方言已经设置了适当的基础架构的情况下才实现了。当前的方言包括 cx_Oracle，以及对使用 pyodbc 的 SQL Server 的可选支持。

使用 setinputsizes 时，方言还可以通过包含/排除列表的方式只对某些数据类型使用该方法。

使用 SETINPUTSIZES 时，对于每个执行的语句，具有绑定参数的语句都会调用 `Dialect.do_set_input_sizes()` 方法。

```py
class sqlalchemy.engine.Compiled
```

表示已编译的 SQL 或 DDL 表达式。

**成员**

__init__(), cache_key, compile_state, construct_params(), dml_compile_state, execution_options, params, sql_compiler, state, statement, string

`Compiled`对象的`__str__`方法应生成语句的实际文本。`Compiled`对象特定于其底层数据库方言，并且可能特定于在特定绑定参数集中引用的列。在任何情况下，`Compiled`对象都不应依赖于这些绑定参数的实际值，即使它可能引用这些值作为默认值。

```py
method __init__(dialect: Dialect, statement: ClauseElement | None, schema_translate_map: SchemaTranslateMapType | None = None, render_schema_translate: bool = False, compile_kwargs: Mapping[str, Any] = {})
```

构造一个新的`Compiled`对象。

参数：

+   `dialect` – 编译的`Dialect`。

+   `statement` – 要编译的`ClauseElement`。

+   `schema_translate_map` –

    模式名称的字典，用于在形成结果 SQL 时进行翻译

    另请参阅

    模式名称的翻译

+   `compile_kwargs` – 将传递给初始调用`Compiled.process()`的额外 kwargs。

```py
attribute cache_key: CacheKey | None = None
```

创建此`Compiled`对象之前生成的`CacheKey`。

用于需要访问在首次缓存`Compiled`实例时生成的原始`CacheKey`实例的例程，通常是为了将原始的`BindParameter`对象列表与在每次调用时生成的每个语句列表进行对比。

```py
attribute compile_state: CompileState | None = None
```

可选的`CompileState`对象，用于维护编译器使用的其他状态。

主要的可执行对象，例如`Insert`、`Update`、`Delete`、`Select`在编译时会生成此状态，以计算关于对象的其他信息。对于要执行的顶级对象，可以将状态存储在此处，也可以适用于结果集处理。

版本 1.4 中的新功能。

```py
method construct_params(params: _CoreSingleExecuteParams | None = None, extracted_parameters: Sequence[BindParameter[Any]] | None = None, escape_names: bool = True) → _MutableCoreSingleExecuteParams | None
```

返回此编译对象的绑定参数。

参数：

**params** – 一个字符串/对象对的字典，其值将覆盖编译到语句中的绑定值。

```py
attribute dml_compile_state: CompileState | None = None
```

可选的`CompileState`在分配`.isinsert`、`.isupdate`或`.isdelete`时分配。

这通常将是与`.compile_state`相同的对象，但有一种例外情况，例如`ORMFromStatementCompileState`对象。

版本 1.4.40 中的新功能。

```py
attribute execution_options: _ExecuteOptions = {}
```

从语句传播的执行选项。在某些情况下，语句的子元素可以修改这些选项。

```py
attribute params
```

返回此已编译对象的绑定参数。

```py
attribute sql_compiler
```

返回一个能够处理 SQL 表达式的已编译对象。

如果此编译器是一个，那么它很可能只返回‘self’。

```py
attribute state: CompilerState
```

编译器状态的描述

```py
attribute statement: ClauseElement | None = None
```

要编译的语句。

```py
attribute string: str = ''
```

`statement`的字符串表示

```py
class sqlalchemy.engine.interfaces.DBAPIConnection
```

表示[**PEP 249**](https://peps.python.org/pep-0249/)数据库连接的协议。

版本 2.0 中的新内容。

另请参见

[连接对象](https://www.python.org/dev/peps/pep-0249/#connection-objects) - 在[**PEP 249**](https://peps.python.org/pep-0249/)中

**成员**

自动提交，close()，commit()，cursor()，rollback()

**类签名**

类`sqlalchemy.engine.interfaces.DBAPIConnection`（`typing_extensions.Protocol`）

```py
attribute autocommit: bool
```

```py
method close() → None
```

```py
method commit() → None
```

```py
method cursor() → DBAPICursor
```

```py
method rollback() → None
```

```py
class sqlalchemy.engine.interfaces.DBAPICursor
```

表示[**PEP 249**](https://peps.python.org/pep-0249/)数据库游标的协议。

版本 2.0 中的新内容。

另请参见

[游标对象](https://www.python.org/dev/peps/pep-0249/#cursor-objects) - 在[**PEP 249**](https://peps.python.org/pep-0249/)中

**成员**

数组大小，callproc()，close()，description，execute()，executemany()，fetchall()，fetchmany()，fetchone()，lastrowid，nextset()，rowcount，setinputsizes()，setoutputsize()

**类签名**

类`sqlalchemy.engine.interfaces.DBAPICursor`（`typing_extensions.Protocol`）

```py
attribute arraysize: int
```

```py
method callproc(procname: str, parameters: Sequence[Any] = Ellipsis) → Any
```

```py
method close() → None
```

```py
attribute description
```

游标的描述属性。

另请参见

[cursor.description](https://www.python.org/dev/peps/pep-0249/#description) - 在[**PEP 249**](https://peps.python.org/pep-0249/)中

```py
method execute(operation: Any, parameters: Sequence[Any] | Mapping[str, Any] | None = None) → Any
```

```py
method executemany(operation: Any, parameters: Sequence[Sequence[Any]] | Sequence[Mapping[str, Any]]) → Any
```

```py
method fetchall() → Sequence[Any]
```

```py
method fetchmany(size: int = Ellipsis) → Sequence[Any]
```

```py
method fetchone() → Any | None
```

```py
attribute lastrowid: int
```

```py
method nextset() → bool | None
```

```py
attribute rowcount
```

```py
method setinputsizes(sizes: Sequence[Any]) → None
```

```py
method setoutputsize(size: Any, column: Any) → None
```

```py
class sqlalchemy.engine.interfaces.DBAPIType
```

表示[**PEP 249**](https://peps.python.org/pep-0249/)数据库类型的协议。

版本 2.0 中的新内容。

另请参见

[类型对象](https://www.python.org/dev/peps/pep-0249/#type-objects) - 在 [**PEP 249**](https://peps.python.org/pep-0249/)

**类签名**

类 `sqlalchemy.engine.interfaces.DBAPIType` (`typing_extensions.Protocol`) 的定义

```py
class sqlalchemy.sql.compiler.DDLCompiler
```

**成员**

`__init__()`, cache_key, compile_state, `construct_params()`, define_constraint_remote_table(), dml_compile_state, execution_options, params, sql_compiler, state, statement, string

**类签名**

类 `sqlalchemy.sql.compiler.DDLCompiler`（`sqlalchemy.sql.compiler.Compiled`的一个子类）

```py
method __init__(dialect: Dialect, statement: ClauseElement | None, schema_translate_map: SchemaTranslateMapType | None = None, render_schema_translate: bool = False, compile_kwargs: Mapping[str, Any] = {})
```

*继承自* `Compiled` *的* `sqlalchemy.sql.compiler.Compiled.__init__` *方法*

构造一个新的 `Compiled` 对象。

参数:

+   `dialect` – 要编译的 `Dialect`。

+   `statement` – 待编译的 `ClauseElement`。

+   `schema_translate_map` –

    用于形成生成 SQL 时要翻译的模式名称字典

    另请参见

    模式名称的翻译

+   `compile_kwargs` – 将传递给初始调用 `Compiled.process()` 的额外 kwargs。

```py
attribute cache_key: CacheKey | None = None
```

*继承自* `Compiled` *的* `Compiled.cache_key` *属性*

生成此 `Compiled` 对象之前生成的 `CacheKey`。

这用于需要访问在首次缓存`Compiled`实例时生成的原始`CacheKey`实例的例程，通常是为了调和原始的`BindParameter`对象列表与每次调用时生成的每个语句列表。

```py
attribute compile_state: CompileState | None = None
```

*继承自* `Compiled` 的 `Compiled.compile_state` *属性*

可选的 `CompileState` 对象，用于维护编译器使用的其他状态。

主要的可执行对象，如`Insert`、`Update`、`Delete`、`Select`，在编译时会生成此状态，以计算有关对象的其他信息。对于要执行的顶级对象，该状态可以存储在这里，也可以适用于结果集处理。

1.4 版中的新功能。

```py
method construct_params(params: _CoreSingleExecuteParams | None = None, extracted_parameters: Sequence[BindParameter[Any]] | None = None, escape_names: bool = True) → _MutableCoreSingleExecuteParams | None
```

返回此编译对象的绑定参数。

参数：

**params** – 一个字符串/对象对的字典，其值将覆盖编译到语句中的绑定值。

```py
method define_constraint_remote_table(constraint, table, preparer)
```

格式化 CREATE CONSTRAINT 语句的远程表子句。

```py
attribute dml_compile_state: CompileState | None = None
```

*继承自* `Compiled` 的 `Compiled.dml_compile_state` *属性*

在分配 `.isinsert`、`.isupdate` 或 `.isdelete` 的相同点分配的可选的 `CompileState`。

这通常是与 `.compile_state` 相同的对象，但有一些例外情况，比如 `ORMFromStatementCompileState` 对象。

1.4.40 版中的新功能。

```py
attribute execution_options: _ExecuteOptions = {}
```

*继承自* `Compiled` 的 `Compiled.execution_options` *属性*

语句中传播的执行选项。在某些情况下，语句的子元素可以修改这些选项。

```py
attribute params
```

*继承自* `Compiled` 的 `Compiled.params` *属性*

返回此编译对象的绑定参数。

```py
attribute sql_compiler
```

```py
attribute state: CompilerState
```

编译器状态的描述

```py
attribute statement: ClauseElement | None = None
```

*继承自* `Compiled` 的 `Compiled.statement` *属性*

要编译的语句。

```py
attribute string: str = ''
```

*继承自* `Compiled` 的 `Compiled.string` *属性*

`statement` 的字符串表示

```py
class sqlalchemy.engine.default.DefaultDialect
```

Dialect 的默认实现

**成员**

bind_typing, colspecs, connect(), construct_arguments, create_connect_args(), create_xid(), cte_follows_insert, dbapi, dbapi_exception_translation_map, ddl_compiler, default_isolation_level, default_metavalue_token, default_schema_name, default_sequence_base, delete_executemany_returning, delete_returning, delete_returning_multifrom, denormalize_name(), div_is_floordiv, do_begin(), do_begin_twophase(), do_close(), do_commit(), do_commit_twophase(), do_execute(), do_execute_no_params(), do_executemany(), do_ping(), do_prepare_twophase(), do_recover_twophase(), do_release_savepoint(), do_rollback(), do_rollback_to_savepoint(), do_rollback_twophase(), do_savepoint(), do_set_input_sizes(), do_terminate(), driver, engine_config_types, engine_created(), exclude_set_input_sizes, execute_sequence_format, execution_ctx_cls, favor_returning_over_lastrowid, full_returning, get_async_dialect_cls(), get_check_constraints(), get_columns(), get_default_isolation_level(), get_dialect_cls(), get_dialect_pool_class(), get_driver_connection(), get_foreign_keys(), get_indexes(), get_isolation_level(), get_isolation_level_values(), get_materialized_view_names(), get_multi_check_constraints(), get_multi_columns(), get_multi_foreign_keys(), get_multi_indexes(), get_multi_pk_constraint(), get_multi_table_comment(), get_multi_table_options(), get_multi_unique_constraints(), get_pk_constraint(), get_schema_names(), get_sequence_names(), get_table_comment(), get_table_names(), get_table_options(), get_temp_table_names(), get_temp_view_names(), get_unique_constraints(), get_view_definition(), get_view_names(), has_index(), has_schema(), has_sequence(), has_table(), has_terminate, identifier_preparer, import_dbapi(), include_set_input_sizes, initialize(), inline_comments, insert_executemany_returning, insert_executemany_returning_sort_by_parameter_order, insert_returning, insertmanyvalues_implicit_sentinel, insertmanyvalues_max_parameters, insertmanyvalues_page_size, is_async, is_disconnect(), label_length, load_provisioning(), loaded_dbapi, max_identifier_length, name, normalize_name(), on_connect(), on_connect_url(), paramstyle, positional, preexecute_autoincrement_sequences, preparer, reflection_options, reset_isolation_level(), returns_native_bytes, sequences_optional, server_side_cursors, server_version_info, set_connection_execution_options(), set_engine_execution_options(), set_isolation_level(), statement_compiler, supports_alter, supports_comments, supports_constraint_comments, supports_default_metavalue, supports_default_values, supports_empty_insert, supports_identity_columns, supports_multivalues_insert, supports_native_boolean, supports_native_decimal, supports_native_enum, supports_native_uuid, supports_sane_multi_rowcount, supports_sane_rowcount, supports_sane_rowcount_returning, supports_sequences, supports_server_side_cursors, supports_simple_order_by_label, supports_statement_cache, tuple_in_values, type_compiler, type_compiler_cls, type_compiler_instance, type_descriptor(), update_executemany_returning, update_returning, update_returning_multifrom, use_insertmanyvalues, use_insertmanyvalues_wo_returning

**类签名**

`sqlalchemy.engine.default.DefaultDialect` 类 (`sqlalchemy.engine.interfaces.Dialect`)

```py
attribute bind_typing = 1
```

定义将类型信息传递给绑定参数的数据库和/或驱动程序的方法。

有关值，请参阅 `BindTyping`。

2.0 版本中新增。

```py
attribute colspecs: MutableMapping[Type[TypeEngine[Any]], Type[TypeEngine[Any]]] = {}
```

从 sqlalchemy.types 映射到特定于方言类的子类的 TypeEngine 类字典。该字典仅适用于类级别，并且不是从方言实例本身访问的。

```py
method connect(*cargs, **cparams)
```

使用此方言的 DBAPI 建立连接。

此方法的默认实现为：

```py
def connect(self, *cargs, **cparams):
    return self.dbapi.connect(*cargs, **cparams)
```

`*cargs, **cparams` 参数直接从此方言的 `Dialect.create_connect_args()` 方法生成。

此方法可用于需要在从 DBAPI 获得新连接时执行程序化的每个连接步骤的方言。

参数：

+   `*cargs` – 从 `Dialect.create_connect_args()` 方法返回的位置参数

+   `**cparams` – 从 `Dialect.create_connect_args()` 方法返回的关键字参数。

返回：

一个 DBAPI 连接，通常来自于 [**PEP 249**](https://peps.python.org/pep-0249/) 模块级别的 `.connect()` 函数。

另请参阅

`Dialect.create_connect_args()`

`Dialect.on_connect()`

```py
attribute construct_arguments: List[Tuple[Type[SchemaItem | ClauseElement], Mapping[str, Any]]] | None = None
```

*继承自* `Dialect` 的 `Dialect.construct_arguments` *属性*

各种 SQLAlchemy 构造的可选参数说明，通常是模式项。

要实现，将其建立为元组的系列，如下所示：

```py
construct_arguments = [
    (schema.Index, {
        "using": False,
        "where": None,
        "ops": None
    })
]
```

如果以上结构在 PostgreSQL 方言上建立，则 `Index` 结构现在将接受关键字参数 `postgresql_using`、`postgresql_where` 和 `postgresql_ops`。构造函数中以 `postgresql_` 为前缀的任何其他参数都将引发 `ArgumentError`。

不包括`construct_arguments`成员的方言将不参与参数验证系统。对于这样的方言，任何参数名称都被所有参与的构造接受，在以该方言名称为前缀的参数命名空间内。这里的原理是，尚未实现此功能的第三方方言将继续以旧方式运行。

另请参阅

`DialectKWArgs` - 实现基类，它消耗`DefaultDialect.construct_arguments`

```py
method create_connect_args(url)
```

构建 DB-API 兼容的连接参数。

给定一个`URL`对象，返回一个包含`(*args, **kwargs)`的元组，适合直接发送到 dbapi 的 connect 函数。这些参数被发送到`Dialect.connect()`方法，然后运行 DBAPI 级别的`connect()`函数。

该方法通常利用`URL.translate_connect_args()`方法生成一个选项字典。

默认实现为：

```py
def create_connect_args(self, url):
    opts = url.translate_connect_args()
    opts.update(url.query)
    return ([], opts)
```

参数：

**url** – 一个`URL`对象

返回：

一个将传递给`Dialect.connect()`方法的`(*args, **kwargs)`元组。

另请参阅

`URL.translate_connect_args()`

```py
method create_xid()
```

创建一个随机的两阶段事务 ID。

此 ID 将传递给 do_begin_twophase()、do_rollback_twophase()、do_commit_twophase()。其格式未指定。

```py
attribute cte_follows_insert: bool = False
```

给定 CTE 和 INSERT 语句时的目标数据库，需要 CTE 在 INSERT 语句之下。

```py
attribute dbapi: ModuleType | None
```

DBAPI 模块对象本身的引用。

SQLAlchemy 方言使用 classmethod `Dialect.import_dbapi()` 导入 DBAPI 模块。其原因是任何方言模块都可以被导入和用于生成 SQL 语句，而无需安装实际的 DBAPI 驱动程序。只有在使用`create_engine()`构造`Engine`时，DBAPI 才会被导入；此时，创建过程将把 DBAPI 模块分配给此属性。

因此，方言应实现 `Dialect.import_dbapi()`，它将导入必要的模块并返回它，然后在方言代码中引用 `self.dbapi` 以引用 DBAPI 模块内容。

版本更改：`Dialect.dbapi` 属性专门用作每个`Dialect` 实例对 DBAPI 模块的引用。以前未完全记录的 `.Dialect.dbapi()` 类方法已被弃用，并由 `Dialect.import_dbapi()` 替换。

```py
attribute dbapi_exception_translation_map: Mapping[str, str] = {}
```

*继承自* `Dialect` 的 `Dialect.dbapi_exception_translation_map` *属性*

一个名称字典，其值将包含作为值的 pep-249 异常的名称（“IntegrityError”、“OperationalError” 等），键入为替代类名，以支持 DBAPI 具有不以它们所引用的方式命名的异常类的情况（例如 IntegrityError = MyException）。在绝大多数情况下，此字典为空。

```py
attribute ddl_compiler
```

`DDLCompiler` 的别名

```py
attribute default_isolation_level: IsolationLevel | None
```

在新连接上隐含存在的隔离级别

```py
attribute default_metavalue_token: str = 'DEFAULT'
```

对于 INSERT… VALUES (DEFAULT) 语法，括号中放置的令牌。

```py
attribute default_schema_name: str | None = None
```

默认模式的名称。此值仅适用于支持的方言，并且通常在与数据库的初始连接期间填充。

```py
attribute default_sequence_base: int = 1
```

将呈现为 CREATE SEQUENCE DDL 语句的“START WITH” 部分的默认值。

```py
attribute delete_executemany_returning: bool = False
```

方言支持具有 executemany 的 DELETE..RETURNING。

```py
attribute delete_returning: bool = False
```

如果方言支持带有 DELETE 的 RETURNING

版本 2.0 中的新功能。

```py
attribute delete_returning_multifrom: bool = False
```

如果方言支持带有 DELETE..FROM 的 RETURNING

版本 2.0 中的新功能。

```py
method denormalize_name(name)
```

如果给定的名称是全小写名称，则将其转换为后端的不区分大小写标识符。

此方法仅在方言定义 `requires_name_normalize=True` 时使用。

```py
attribute div_is_floordiv: bool = True
```

目标数据库将 / 除法运算符视为“地板除法”

```py
method do_begin(dbapi_connection)
```

提供给定 DB-API 连接的 `connection.begin()` 实现。

DBAPI 没有专用的“开始”方法，预期事务是隐式的。此挂钩是为了那些可能需要在此领域提供额外帮助的 DBAPI 而提供的。

参数：

**dbapi_connection** – 一个 DBAPI 连接，通常在 `ConnectionFairy` 中被代理。

```py
method do_begin_twophase(connection: Connection, xid: Any) → None
```

*继承自* `Dialect` 的 `Dialect.do_begin_twophase()` *方法*

在给定连接上开始两阶段事务。

参数：

+   `connection` – 一个 `Connection`。

+   `xid` – xid

```py
method do_close(dbapi_connection)
```

提供给定 DBAPI 连接的 `connection.close()` 实现。

当连接从池中分离或被返回超出池的正常容量时，将调用此钩子。

```py
method do_commit(dbapi_connection)
```

提供 `connection.commit()` 的实现，给定一个 DB-API 连接。

参数：

**dbapi_connection** – 一个 DBAPI 连接，通常在 `ConnectionFairy` 中代理。

```py
method do_commit_twophase(connection: Connection, xid: Any, is_prepared: bool = True, recover: bool = False) → None
```

*继承自* `Dialect` *方法的* `Dialect.do_commit_twophase()`。

在给定连接上提交一个两阶段事务。

参数：

+   `connection` – 一个 `Connection`。

+   `xid` – xid

+   `is_prepared` – 是否调用了 `TwoPhaseTransaction.prepare()`。

+   `recover` – 如果传递了恢复标志。

```py
method do_execute(cursor, statement, parameters, context=None)
```

提供 `cursor.execute(statement, parameters)` 的实现。

```py
method do_execute_no_params(cursor, statement, context=None)
```

提供 `cursor.execute(statement)` 的实现。

不应发送参数集合。

```py
method do_executemany(cursor, statement, parameters, context=None)
```

提供 `cursor.executemany(statement, parameters)` 的实现。

```py
method do_ping(dbapi_connection: DBAPIConnection) → bool
```

检查 DBAPI 连接并在连接可用时返回 True。

```py
method do_prepare_twophase(connection: Connection, xid: Any) → None
```

*继承自* `Dialect` *方法的* `Dialect.do_prepare_twophase()`。

在给定连接上准备一个两阶段事务。

参数：

+   `connection` – 一个 `Connection`。

+   `xid` – xid

```py
method do_recover_twophase(connection: Connection) → List[Any]
```

*继承自* `Dialect` *方法的* `Dialect.do_recover_twophase()`。

在给定连接上恢复未提交的准备好的两阶段事务标识符列表。

参数：

**connection** – 一个 `Connection`。

```py
method do_release_savepoint(connection, name)
```

在连接上释放命名保存点。

参数：

+   `connection` – 一个 `Connection`。

+   `name` – 保存点名称。

```py
method do_rollback(dbapi_connection)
```

提供 `connection.rollback()` 的实现，给定一个 DB-API 连接。

参数：

**dbapi_connection** – 一个 DBAPI 连接，通常在 `ConnectionFairy` 中代理。

```py
method do_rollback_to_savepoint(connection, name)
```

将连接回滚到命名保存点。

参数：

+   `connection` – 一个 `Connection`。

+   `name` – 保存点名称。

```py
method do_rollback_twophase(connection: Connection, xid: Any, is_prepared: bool = True, recover: bool = False) → None
```

*继承自* `Dialect` *方法的* `Dialect.do_rollback_twophase()`。

在给定连接上回滚两阶段事务。

参数：

+   `connection` – 一个 `Connection`。

+   `xid` – xid

+   `is_prepared` – 是否调用了`TwoPhaseTransaction.prepare()`。

+   `recover` – 如果传递了 recover 标志。

```py
method do_savepoint(connection, name)
```

使用给定名称创建一个保存点。

参数：

+   `connection` – 一个`Connection`。

+   `name` – 保存点名称。

```py
method do_set_input_sizes(cursor: DBAPICursor, list_of_tuples: _GenericSetInputSizesType, context: ExecutionContext) → Any
```

*继承自* `Dialect` *的* `Dialect.do_set_input_sizes()` *方法。

使用适当的参数调用`cursor.setinputsizes()`方法。

如果`Dialect.bind_typing`属性设置为`BindTyping.SETINPUTSIZES`值，则调用此钩子。参数数据以元组列表的形式传递（paramname，dbtype，sqltype），其中`paramname`是语句中参数的键，`dbtype`是 DBAPI 数据类型，`sqltype`是 SQLAlchemy 类型。元组的顺序是正确的参数顺序。

自 1.4 版本新推出。

在 2.0 版本中更改：- 通过将`Dialect.bind_typing`设置为`BindTyping.SETINPUTSIZES`来启用 setinputsizes 模式。接受`use_setinputsizes`参数的方言应适当设置此值。

```py
method do_terminate(dbapi_connection)
```

提供一个实现了尽可能不阻塞的`connection.close()`的钩子，给定一个 DBAPI 连接。

在绝大多数情况下，这只是调用`.close()`，但是对于某些 asyncio 方言可能调用不同的 API 功能。

当连接被回收或无效时，此钩子由`Pool`调用。

自 1.4.41 版本新推出。

```py
attribute driver: str
```

方言的 DBAPI 的标识名称。

```py
attribute engine_config_types: Mapping[str, Any] = {'echo': <function bool_or_str.<locals>.bool_or_value>, 'echo_pool': <function bool_or_str.<locals>.bool_or_value>, 'future': <function asbool>, 'max_overflow': <function asint>, 'pool_recycle': <function asint>, 'pool_size': <function asint>, 'pool_timeout': <function asint>}
```

一个字符串键的映射，可以是引擎配置中的类型转换函数。

```py
classmethod engine_created(engine: Engine) → None
```

*继承自* `Dialect` *的* `Dialect.engine_created()` *方法。

一个方便的钩子，在返回最终`Engine`之前调用。

如果方言从`get_dialect_cls()`方法返回了不同的类，则该钩子将在两个类上调用，首先在由`get_dialect_cls()`方法返回的方言类上调用，然后在调用方法的类上调用。

钩子应由方言和/或包装器用于将特殊事件应用于引擎或其组件。特别是，它允许方言包装类应用方言级事件。

```py
attribute exclude_set_input_sizes: Set[Any] | None = None
```

设置应在自动 cursor.setinputsizes() 调用中排除的 DBAPI 类型对象集合。

仅当 bind_typing 为 BindTyping.SET_INPUT_SIZES 时才使用此选项。

```py
attribute execute_sequence_format
```

`tuple` 的别名

```py
attribute execution_ctx_cls
```

`DefaultExecutionContext` 的别名

```py
attribute favor_returning_over_lastrowid: bool = False
```

对于支持 lastrowid 和 RETURNING 插入策略的后端，请优先使用 RETURNING 进行简单的单整数 pk 插入。

在大多数后端上，cursor.lastrowid 往往更具性能。

```py
attribute full_returning
```

自版本 2.0 弃用：full_returning 已弃用，请改用 insert_returning、update_returning、delete_returning

```py
classmethod get_async_dialect_cls(url: URL) → Type[Dialect]
```

*继承自* `Dialect` 的 `Dialect.get_async_dialect_cls()` 方法

给定一个 URL，返回将由异步引擎使用的 `Dialect`。

默认情况下，这是 `Dialect.get_dialect_cls()` 的别名，只返回 cls。如果一个方言在同一名称下提供了同步和异步版本，则可以使用它，例如 `psycopg` 驱动程序。

版本 2 中的新功能。

另请参见

`Dialect.get_dialect_cls()`

```py
method get_check_constraints(connection: Connection, table_name: str, schema: str | None = None, **kw: Any) → List[ReflectedCheckConstraint]
```

*继承自* `Dialect` 的 `Dialect.get_check_constraints()` 方法

返回 `table_name` 中检查约束的信息。

给定一个字符串 `table_name` 和一个可选的字符串 `schema`，将检查约束信息作为与 `ReflectedCheckConstraint` 字典相对应的字典列表返回。

这是一个内部方言方法。应用程序应使用 `Inspector.get_check_constraints()`。

```py
method get_columns(connection: Connection, table_name: str, schema: str | None = None, **kw: Any) → List[ReflectedColumn]
```

*继承自* `Dialect` 的 `Dialect.get_columns()` 方法

返回关于 `table_name` 中列的信息。

给定一个 `Connection`、一个字符串 `table_name` 和一个可选的字符串 `schema`，将列信息作为与 `ReflectedColumn` 字典相对应的字典列表返回。

这是一个内部方言方法。 应用程序应使用`Inspector.get_columns()`。

```py
method get_default_isolation_level(dbapi_conn)
```

给定一个 DBAPI 连接，返回其隔离级别，或者如果无法检索到隔离级别，则返回默认隔离级别。

可以由子类覆盖，以提供对于无法可靠检索实际隔离级别的数据库的“回退”隔离级别。

默认情况下，调用`Interfaces.get_isolation_level()`方法，传播引发的任何异常。

新版本中新增 1.3.22。

```py
classmethod get_dialect_cls(url: URL) → Type[Dialect]
```

*继承自* `方言` *的* `Dialect.get_dialect_cls()` *方法*

给定一个 URL，返回将要使用的`方言`。

这是一个挂钩，允许外部插件围绕现有方言提供功能，通过允许从基于入口点的 URL 加载插件，然后插件返回要使用的实际方言。

默认情况下，这只是返回 cls。

```py
method get_dialect_pool_class(url: URL) → Type[Pool]
```

返回用于给定 URL 的 Pool 类。

```py
method get_driver_connection(connection)
```

返回由外部驱动程序包返回的连接对象。

对于使用 DBAPI 兼容驱动程序的普通方言，此调用将只返回作为参数传递的`connection`。 对于改用非 DBAPI 兼容驱动程序进行适配的方言，例如在适配异步驱动程序时，此调用将返回驱动程序返回的类似连接的对象。

新版本中新增 1.4.24。

```py
method get_foreign_keys(connection: Connection, table_name: str, schema: str | None = None, **kw: Any) → List[ReflectedForeignKeyConstraint]
```

*继承自* `方言` *的* `Dialect.get_foreign_keys()` *方法*

返回关于`table_name`中外键的信息。

给定一个`连接`，一个字符串`table_name`，和一个可选字符串`schema`，返回作为与`ReflectedForeignKeyConstraint`字典对应的字典列表的外键信息。

这是一个内部方言方法。 应用程序应使用`Inspector.get_foreign_keys()`。

```py
method get_indexes(connection: Connection, table_name: str, schema: str | None = None, **kw: Any) → List[ReflectedIndex]
```

*继承自* `方言` *的* `Dialect.get_indexes()` *方法*

返回关于`table_name`中索引的信息。

给定一个`连接`，一个字符串`table_name`和一个可选字符串`schema`，返回作为与`ReflectedIndex`字典对应的字典列表的索引信息。

这是一个内部方言方法。应用程序应使用 `Inspector.get_indexes()`。 

```py
method get_isolation_level(dbapi_connection: DBAPIConnection) → Literal['SERIALIZABLE', 'REPEATABLE READ', 'READ COMMITTED', 'READ UNCOMMITTED', 'AUTOCOMMIT']
```

*继承自* `Dialect` 的 `Dialect.get_isolation_level()` *方法*

给定一个 DBAPI 连接，返回其隔离级别。

当使用 `Connection` 对象时，可以使用 `Connection.connection` 访问器获取相应的 DBAPI 连接。

请注意，这是一个方言级方法，用作 `Connection` 和 `Engine` 隔离级别功能实现的一部分；对于大多数典型用例，应优先使用这些 API。

另请参阅

`Connection.get_isolation_level()` - 查看当前级别

`Connection.default_isolation_level` - 查看默认级别

`Connection.execution_options.isolation_level` - 设置每个 `Connection` 的隔离级别

`create_engine.isolation_level` - 设置每个 `Engine` 的隔离级别

```py
method get_isolation_level_values(dbapi_conn: DBAPIConnection) → List[Literal['SERIALIZABLE', 'REPEATABLE READ', 'READ COMMITTED', 'READ UNCOMMITTED', 'AUTOCOMMIT']]
```

*继承自* `Dialect` 的 `Dialect.get_isolation_level_values()` *方法*

返回一个字符串隔离级别名称序列，该序列被此方言接受。

可用名称应使用以下约定：

+   使用大写命名。隔离级别方法将接受小写名称，但在传递给方言之前会将其标准化为大写。

+   单词之间应该用空格分隔，而不是下划线，例如 `REPEATABLE READ`。在传递给方言之前，隔离级别名称将下划线转换为空格。

+   对于标准隔离名称，如果后端支持，应为 `READ UNCOMMITTED`、`READ COMMITTED`、`REPEATABLE READ`、`SERIALIZABLE`

+   如果方言支持自动提交选项，则应使用隔离级别名称 `AUTOCOMMIT`。

+   其他隔离模式也可能存在，只要它们以大写形式命名并使用空格而不是下划线。

此函数用于默认方言检查给定的隔离级别参数是否有效，否则会引发`ArgumentError`。

在可能的情况下，方法会传递一个 DBAPI 连接，以便方言自身需要查询连接本身以确定此列表，但是大多数后端都会返回一个硬编码的值列表。如果方言支持“AUTOCOMMIT”，那么该值也应该存在于返回的序列中。

该方法默认引发`NotImplementedError`。如果方言未实现此方法，则默认方言将不会在将其传递给`Dialect.set_isolation_level()`方法之前对给定的隔离级别值执行任何检查。这是为了与尚未实现此方法的第三方方言保持向后兼容。

版本 2.0 新增。

```py
method get_materialized_view_names(connection: Connection, schema: str | None = None, **kw: Any) → List[str]
```

*继承自* `Dialect` *的* `Dialect.get_materialized_view_names()` *方法*

返回数据库中所有可用物化视图名称的列表。

这是一个内部方言方法。应用程序应该使用`Inspector.get_materialized_view_names()`。

参数：

**模式** –

查询的模式名称，如果不是默认模式。

版本 2.0 新增。

```py
method get_multi_check_constraints(connection, **kw)
```

返回给定`schema`中所有表的检查约束信息。

这是一个内部方言方法。应用程序应该使用`Inspector.get_multi_check_constraints()`。

注意

`DefaultDialect`提供了一个默认实现，将针对`Dialect.get_table_names()`、`Dialect.get_view_names()`或`Dialect.get_materialized_view_names()`返回的每个对象调用单表方法，具体取决于提供的`kind`。希望支持更快实现的方言应该实现此方法。

版本 2.0 新增。

```py
method get_multi_columns(connection, **kw)
```

返回给定`schema`中所有表的列信息。

这是一个内部方言方法。应用程序应该使用`Inspector.get_multi_columns()`。

注意

`DefaultDialect`提供了一个默认实现，将针对`Dialect.get_table_names()`、`Dialect.get_view_names()`或`Dialect.get_materialized_view_names()`返回的每个对象调用单表方法，具体取决于提供的`kind`。希望支持更快实现的方言应该实现此方法。

版本 2.0 新增。

```py
method get_multi_foreign_keys(connection, **kw)
```

返回给定 `schema` 中所有表的外键信息。

这是一个内部方言方法。应用程序应该使用 `Inspector.get_multi_foreign_keys()`。

注意

`DefaultDialect` 提供了一个默认实现，它会根据提供的 `kind` 对 `Dialect.get_table_names()`、`Dialect.get_view_names()` 或 `Dialect.get_materialized_view_names()` 返回的每个对象调用单个表方法。希望支持更快实现的方言应该实现这个方法。

2.0 版本中新增。

```py
method get_multi_indexes(connection, **kw)
```

返回给定 `schema` 中所有表的索引信息。

这是一个内部方言方法。应用程序应该使用 `Inspector.get_multi_indexes()`。

注意

`DefaultDialect` 提供了一个默认实现，它会根据提供的 `kind` 对 `Dialect.get_table_names()`、`Dialect.get_view_names()` 或 `Dialect.get_materialized_view_names()` 返回的每个对象调用单个表方法。希望支持更快实现的方言应该实现这个方法。

2.0 版本中新增。

```py
method get_multi_pk_constraint(connection, **kw)
```

返回给定 `schema` 中所有表的主键约束信息。

这是一个内部方言方法。应用程序应该使用 `Inspector.get_multi_pk_constraint()`。

注意

`DefaultDialect` 提供了一个默认实现，它会根据提供的 `kind` 对 `Dialect.get_table_names()`、`Dialect.get_view_names()` 或 `Dialect.get_materialized_view_names()` 返回的每个对象调用单个表方法。希望支持更快实现的方言应该实现这个方法。

2.0 版本中新增。

```py
method get_multi_table_comment(connection, **kw)
```

返回给定 `schema` 中所有表的表注释信息。

这是一个内部方言方法。应用程序应该使用 `Inspector.get_multi_table_comment()`。

注意

`DefaultDialect` 提供了一个默认实现，它会根据提供的 `kind` 对 `Dialect.get_table_names()`、`Dialect.get_view_names()` 或 `Dialect.get_materialized_view_names()` 返回的每个对象调用单个表方法。希望支持更快实现的方言应该实现这个方法。

2.0 版本中新增。

```py
method get_multi_table_options(connection, **kw)
```

返回给定 `schema` 中表创建时指定的选项字典。

这是一个内部方言方法。应用程序应该使用 `Inspector.get_multi_table_options()`。

注意

`DefaultDialect` 提供了一个默认实现，它会根据提供的 `kind` 对 `Dialect.get_table_names()`、`Dialect.get_view_names()` 或 `Dialect.get_materialized_view_names()` 返回的每个对象调用单个表方法。希望支持更快实现的方言应该实现这个方法。

2.0 版本中新增。

```py
method get_multi_unique_constraints(connection, **kw)
```

返回给定`schema`中所有表的唯一约束信息。

这是一个内部方言方法。应用程序应该使用`Inspector.get_multi_unique_constraints()`。

注意

`DefaultDialect`提供了一个默认实现，将根据提供的`kind`调用`Dialect.get_table_names()`、`Dialect.get_view_names()`或`Dialect.get_materialized_view_names()`返回的每个对象的单表方法。希望支持更快实现的方言应该实现这个方法。

新版本 2.0。

```py
method get_pk_constraint(connection: Connection, table_name: str, schema: str | None = None, **kw: Any) → ReflectedPrimaryKeyConstraint
```

*继承自* `Dialect` *的* `Dialect.get_pk_constraint()` *方法。

返回`table_name`上主键约束的信息。

给定`Connection`、字符串`table_name`和可选字符串`schema`，返回与`ReflectedPrimaryKeyConstraint`字典对应的主键信息字典。

这是一个内部方言方法。应用程序应该使用`Inspector.get_pk_constraint()`。

```py
method get_schema_names(connection: Connection, **kw: Any) → List[str]
```

*继承自* `Dialect` *的* `Dialect.get_schema_names()` *方法。

返回数据库中所有模式名称的列表。

这是一个内部方言方法。应用程序应该使用`Inspector.get_schema_names()`。

```py
method get_sequence_names(connection: Connection, schema: str | None = None, **kw: Any) → List[str]
```

*继承自* `Dialect` *的* `Dialect.get_sequence_names()` *方法。

返回数据库中所有序列名称的列表。

这是一个内部方言方法。应用程序应该使用`Inspector.get_sequence_names()`。

参数：

**schema** – 要查询的模式名称，如果不是默认模式。

新版本 1.4。

```py
method get_table_comment(connection: Connection, table_name: str, schema: str | None = None, **kw: Any) → ReflectedTableComment
```

*继承自* `Dialect` *的* `Dialect.get_table_comment()` *方法。

返回由`table_name`标识的表的“注释”。

给定字符串`table_name`和可选字符串`schema`，返回与`ReflectedTableComment`字典对应的表注释信息字典。

这是一个内部方言方法。应用程序应该使用`Inspector.get_table_comment()`。

抛出：

对于不支持注释的方言，抛出 `NotImplementedError`。

版本 1.2 中的新功能。

```py
method get_table_names(connection: Connection, schema: str | None = None, **kw: Any) → List[str]
```

*继承自* `Dialect` *的* `Dialect.get_table_names()` *方法。*

返回 `schema` 的表名列表。

这是一个内部方言方法。应用程序应使用 `Inspector.get_table_names()`。

```py
method get_table_options(connection: Connection, table_name: str, schema: str | None = None, **kw: Any) → Dict[str, Any]
```

*继承自* `Dialect` *的* `Dialect.get_table_options()` *方法。*

返回创建 `table_name` 时指定的选项字典。

这是一个内部方言方法。应用程序应使用 `Inspector.get_table_options()`。

```py
method get_temp_table_names(connection: Connection, schema: str | None = None, **kw: Any) → List[str]
```

*继承自* `Dialect` *的* `Dialect.get_temp_table_names()` *方法。*

返回给定连接上的临时表名列表，如果底层后端支持。

这是一个内部方言方法。应用程序应使用 `Inspector.get_temp_table_names()`。

```py
method get_temp_view_names(connection: Connection, schema: str | None = None, **kw: Any) → List[str]
```

*继承自* `Dialect` *的* `Dialect.get_temp_view_names()` *方法。*

返回给定连接上的临时视图名列表，如果底层后端支持。

这是一个内部方言方法。应用程序应使用 `Inspector.get_temp_view_names()`。

```py
method get_unique_constraints(connection: Connection, table_name: str, schema: str | None = None, **kw: Any) → List[ReflectedUniqueConstraint]
```

*继承自* `Dialect` *的* `Dialect.get_unique_constraints()` *方法。*

返回 `table_name` 中唯一约束的信息。

给定一个字符串 `table_name` 和一个可选的字符串 `schema`，返回唯一约束信息，作为与 `ReflectedUniqueConstraint` 字典相对应的字典列表。

这是一个内部方言方法。应用程序应使用 `Inspector.get_unique_constraints()`。

```py
method get_view_definition(connection: Connection, view_name: str, schema: str | None = None, **kw: Any) → str
```

*继承自* `Dialect` *的* `Dialect.get_view_definition()` *方法。*

返回普通或材料化视图定义。

这是一个内部方言方法。应用程序应使用 `Inspector.get_view_definition()`。

给定一个 `Connection`，一个字符串 `view_name`，和一个可选的字符串 `schema`，返回视图定义。

```py
method get_view_names(connection: Connection, schema: str | None = None, **kw: Any) → List[str]
```

*继承自* `Dialect` *的* `Dialect.get_view_names()` *方法。*

返回数据库中所有非材料化视图名称的列表。

这是一个内部方言方法。应用程序应使用 `Inspector.get_view_names()`。

参数：

**schema** – 要查询的模式名称，如果不是默认模式。

```py
method has_index(connection, table_name, index_name, schema=None, **kw)
```

检查数据库中特定索引名称的存在性。

给定一个 `Connection` 对象，一个字符串 `table_name` 和字符串索引名称，如果给定表上存在给定名称的索引，则返回 `True`，否则返回 `False`。

`DefaultDialect` 在 `Dialect.has_table()` 和 `Dialect.get_indexes()` 方法方面实现了这一点，但是方言可以实现更高效的版本。

这是一个内部方言方法。应用程序应使用 `Inspector.has_index()`。

版本 1.4 中的新功能。

```py
method has_schema(connection: Connection, schema_name: str, **kw: Any) → bool
```

检查数据库中特定模式名称的存在性。

给定一个 `Connection` 对象，一个字符串 `schema_name`，如果存在给定的模式，则返回 `True`，否则返回 `False`。

`DefaultDialect` 通过检查 `Dialect.get_schema_names()` 返回的模式中是否存在 `schema_name` 来实现这一点，但是方言可以实现更高效的版本。

这是一个内部方言方法。应用程序应使用 `Inspector.has_schema()`。

版本 2.0 中的新功能。

```py
method has_sequence(connection: Connection, sequence_name: str, schema: str | None = None, **kw: Any) → bool
```

*从* `Dialect` *的* `Dialect.has_sequence()` *方法继承*

检查数据库中特定序列的存在性。

给定一个 `Connection` 对象和一个字符串 sequence_name，如果数据库中存在给定的序列，则返回 `True`，否则返回 `False`。

这是一个内部方言方法。应用程序应使用 `Inspector.has_sequence()`。

```py
method has_table(connection: Connection, table_name: str, schema: str | None = None, **kw: Any) → bool
```

*从* `Dialect` *的* `Dialect.has_table()` *方法继承*

对于内部方言使用，检查数据库中特定表或视图的存在性。

给定一个 `Connection` 对象，一个字符串 table_name 和可选的模式名称，如果数据库中存在给定的表，则返回 True，否则返回 False。

此方法作为公共面向的 `Inspector.has_table()` 方法的底层实现，并且在内部用于实现方法如 `Table.create()` 和 `MetaData.create_all()` 的“checkfirst”行为。

注意

此方法由 SQLAlchemy 内部使用，并发布以便第三方方言可以提供实现。这**不是**用于检查表存在性的公共 API。请使用 `Inspector.has_table()` 方法。

2.0 版本更改::: `Dialect.has_table()` 现在正式支持检查其他类似表的对象：

+   任何类型的视图（普通或材料化）

+   任何类型的临时表

以前，这两个检查没有正式规定，不同的方言在行为上会有所不同。方言测试套件现在包括对所有这些对象类型的测试，支持视图或临时表的程度应该寻求支持定位这些对象以实现完全的兼容性。

```py
attribute has_terminate: bool = False
```

此方言是否具有单独的“终止”实现，不会阻塞或需要等待。

```py
attribute identifier_preparer: IdentifierPreparer
```

一旦构造了 `DefaultDialect` 的实例，此元素将引用一个 `IdentifierPreparer` 的实例。

```py
classmethod import_dbapi() → module
```

*继承自* `Dialect` *的* `Dialect.import_dbapi()` *方法。

导入此方言使用的 DBAPI 模块。

此处返回的 Python 模块对象将被分配为构造的方言的一个实例变量，名称为 `.dbapi`。

2.0 版本更改：`Dialect.import_dbapi()` 类方法从先前的方法 `.Dialect.dbapi()` 重命名，该方法将在方言实例化时由 DBAPI 模块本身替换，因此以两种不同方式使用相同的名称。如果第三方方言上存在 `.Dialect.dbapi()` 类方法，将使用它并���出弃用警告。

```py
attribute include_set_input_sizes: Set[Any] | None = None
```

应包含在自动 cursor.setinputsizes() 调用中的一组 DBAPI 类型对象。

仅在 bind_typing 为 BindTyping.SET_INPUT_SIZES 时使用。

```py
method initialize(connection)
```

在使用连接创建方言时调用。

允许方言根据服务器版本信息或其他属性配置选项。

此处传递的连接是一个具有完整功能的 SQLAlchemy 连接对象。

应通过 super()调用基础方言的 initialize()方法。

注意

从 SQLAlchemy 1.4 开始，在调用任何`Dialect.on_connect()`钩子之前调用此方法。

```py
attribute inline_comments: bool = False
```

指示方言是否支持与表或列定义内联的注释 DDL。如果为 False，则意味着必须使用 ALTER 来设置表和列的注释。

```py
attribute insert_executemany_returning: bool
```

当 dialect.do_executemany()被使用时，方言/驱动程序/数据库支持某些方法来提供 INSERT…RETURNING 支持。

```py
attribute insert_executemany_returning_sort_by_parameter_order: bool
```

当 dialect.do_executemany()与设置了`Insert.returning.sort_by_parameter_order`参数一起使用时，方言/驱动程序/数据库支持某些方法来提供 INSERT…RETURNING 支持。

```py
attribute insert_returning: bool = False
```

如果方言支持 INSERT 与 RETURNING

2.0 版本中的新功能。

```py
attribute insertmanyvalues_implicit_sentinel: InsertmanyvaluesSentinelOpts = symbol('NOT_SUPPORTED')
```

指示数据库是否支持一种形式的批量插入，其中自增整数主键可以可靠地用作 INSERT 的行的排序。

2.0.10 版本中的新功能。

另请参见

将返回的行与参数集相关联

```py
attribute insertmanyvalues_max_parameters: int = 32700
```

与 insertmanyvalues_page_size 替代，还将基于语句中的参数总数限制页面大小。

```py
attribute insertmanyvalues_page_size: int = 1000
```

每个`ExecuteStyle.INSERTMANYVALUES`执行中渲染为单个 INSERT..VALUES()语句的行数。

默认方言将此默认设置为 1000。

2.0 版本中的新功能。

另请参见

`Connection.execution_options.insertmanyvalues_page_size` - `Connection`上可用的执行选项，语句

```py
attribute is_async: bool = False
```

此方言是否用于 asyncio 使用。

```py
method is_disconnect(e, connection, cursor)
```

如果给定的 DB-API 错误表示无效连接，则返回 True

```py
attribute label_length: int | None
```

SQL 标签的可选用户定义的最大长度

```py
classmethod load_provisioning()
```

为此方言设置 provision.py 模块。

对于包含一个设置供应跟随者的 provision.py 模块的方言，此方法应启动该过程。

典型实现可能是：

```py
@classmethod
def load_provisioning(cls):
    __import__("mydialect.provision")
```

默认方法假定当前方言的所有者包中有一个名为`provision.py`的模块，基于`__module__`属性：

```py
@classmethod
def load_provisioning(cls):
    package = ".".join(cls.__module__.split(".")[0:-1])
    try:
        __import__(package + ".provision")
    except ImportError:
        pass
```

1.3.14 版本中的新功能。

```py
attribute loaded_dbapi
```

```py
attribute max_identifier_length: int = 9999
```

标识符名称的最大长度。

```py
attribute name: str = 'default'
```

方言的从 DBAPI 中立点来看的标识名称（即“sqlite”）

```py
method normalize_name(name)
```

如果检测到名称不区分大小写，则将给定名称转换为小写。

仅当方言定义 requires_name_normalize=True 时才使用此方法。

```py
method on_connect()
```

返回一个可调用对象，该对象设置新创建的 DBAPI 连接。

这个可调用函数应该接受一个名为“conn”的参数，它就是 DBAPI 连接本身。内部可调用函数没有返回值。

例如：

```py
class MyDialect(default.DefaultDialect):
    # ...

    def on_connect(self):
        def do_on_connect(connection):
            connection.execute("SET SPECIAL FLAGS etc")

        return do_on_connect
```

用于设置方言范围内的每个连接的选项，例如隔离模式、Unicode 模式等。

“do_on_connect” 可调用函数是通过使用 `PoolEvents.connect()` 事件钩子来调用的，然后解包 DBAPI 连接并将其传递给可调用函数。

在 1.4 版本中更改：不再为方言的第一个连接调用两次 on_connect 钩子。然而，在 `Dialect.initialize()` 方法之前仍会调用 on_connect 钩子。

在 1.4.3 版本中更改：从一个新方法 on_connect_url 调用 on_connect 钩子，该方法传递用于创建连接参数的 URL。如果方言需要使用用于连接的 URL 对象来获取附加上下文，则可以实现 on_connect_url 而不是 on_connect。

如果返回 None，则不生成事件监听器。

返回：

一个可调用的函数，接受一个单一的 DBAPI 连接作为参数，或者为 None。

另请参阅

`Dialect.connect()` - 允许控制 DBAPI `connect()` 序列本身。

`Dialect.on_connect_url()` - 取代了 `Dialect.on_connect()` 来接收上下文中的 `URL` 对象。

```py
method on_connect_url(url: URL) → Callable[[Any], Any] | None
```

*继承自* `Dialect` *的* `Dialect.on_connect_url()` *方法。

返回一个可调用的函数，用于设置新创建的 DBAPI 连接。

当由方言实现时，这个方法是一个新的钩子，它取代了 `Dialect.on_connect()` 方法。当方言没有实现时，它直接调用 `Dialect.on_connect()` 方法，以保持与现有方言的兼容性。不会为 `Dialect.on_connect()` 预期进行弃用。

这个可调用函数应该接受一个名为“conn”的参数，它就是 DBAPI 连接本身。内部可调用函数没有返回值。

例如：

```py
class MyDialect(default.DefaultDialect):
    # ...

    def on_connect_url(self, url):
        def do_on_connect(connection):
            connection.execute("SET SPECIAL FLAGS etc")

        return do_on_connect
```

用于设置方言范围内的每个连接的选项，例如隔离模式、Unicode 模式等。

这个方法与 `Dialect.on_connect()` 不同之处在于，它接收与连接参数相关的 `URL` 对象。通常，从 `Dialect.on_connect()` 钩子获取这个对象的唯一方式是查看 `Engine` 本身，但是这个 URL 对象可能已经被插件替换。

注意

`Dialect.on_connect_url()` 的默认实现是调用 `Dialect.on_connect()` 方法。因此，如果一个方言实现了这个方法，那么除非覆盖方言直接从这里调用，否则不会调用 `Dialect.on_connect()` 方法。

版本 1.4.3 中的新内容：增加了 `Dialect.on_connect_url()`，它通常调用 `Dialect.on_connect()`。

参数：

**url** – 表示传递给 `Dialect.create_connect_args()` 方法的 `URL` 的 `URL` 对象。

返回：

一个可接受单个 DBAPI 连接作为参数的可调用对象，或者为 None。

另请参见

`Dialect.on_connect()`

```py
attribute paramstyle: str
```

将要使用的参数风格（某些 DB-API 支持多个参数风格）。

```py
attribute positional: bool
```

如果这个 Dialect 的参数风格是按位置的，则为 True。

```py
attribute preexecute_autoincrement_sequences: bool = False
```

如果没有使用 RETURNING，则‘implicit’ 主键函数必须分别执行以获取它们的值，则为 True。

当在 `Table` 对象上使用 `implicit_returning=False` 参数时，这目前是针对 PostgreSQL 的。

```py
attribute preparer
```

`IdentifierPreparer` 的别名

```py
attribute reflection_options: Sequence[str] = ()
```

*继承自* `Dialect` *的* `Dialect.reflection_options` *属性*

一个字符串名称序列，指示可以在使用 `Table.autoload_with` 时作为“反射选项”传递的关键字参数，这些参数将在 `Table` 对象上设置。

当前示例是 Oracle 方言中的“oracle_resolve_synonyms”。

```py
method reset_isolation_level(dbapi_conn)
```

给定一个 DBAPI 连接，将其隔离恢复为默认值。

请注意，这是一个方言级方法，用作`Connection`和`Engine`隔离级别设施的实现的一部分；这些 API 应该优先用于大多数典型用例。

另请参阅

`Connection.get_isolation_level()` - 查看当前级别

`Connection.default_isolation_level` - 查看默认级别

`Connection.execution_options.isolation_level` - 设置每个`Connection`的隔离级别

`create_engine.isolation_level` - 设置每个`Engine`的隔离级别

```py
attribute returns_native_bytes: bool = False
```

指示 Python bytes()对象是否由驱动程序原生返回 SQL“binary”数据类型。

新版本 2.0.11 中提供。

```py
attribute sequences_optional: bool = False
```

如果为 True，则表示`Sequence.optional`参数在`Sequence`构造上是否应该发出信号以不生成 CREATE SEQUENCE。仅适用于支持序列的方言。目前仅用于允许在指定 Sequence()用于其他后端的列上使用 PostgreSQL SERIAL。

```py
attribute server_side_cursors: bool = False
```

已弃用；指示方言是否应尝试默认使用服务器端游标

```py
attribute server_version_info: Tuple[Any, ...] | None = None
```

包含正在使用的 DB 后端的版本号的元组。

此值仅适用于支持的方言，并且通常在初始连接到数据库时填充。

```py
method set_connection_execution_options(connection: Connection, opts: Mapping[str, Any]) → None
```

为给定连接建立执行选项。

这由`DefaultDialect`实现，以实现`Connection.execution_options.isolation_level`执行选项。方言可以拦截各种执行选项，这些选项可能需要修改特定 DBAPI 连接上的状态。

新版本 1.4 中提供。

```py
method set_engine_execution_options(engine: Engine, opts: Mapping[str, Any]) → None
```

为给定引擎建立执行选项。

这由`DefaultDialect`实现，用于为给定的`Engine`创建的新`Connection`实例建立事件钩子，然后调用该连接的`Dialect.set_connection_execution_options()`方法。

```py
method set_isolation_level(dbapi_connection: DBAPIConnection, level: Literal['SERIALIZABLE', 'REPEATABLE READ', 'READ COMMITTED', 'READ UNCOMMITTED', 'AUTOCOMMIT']) → None
```

*继承自* `Dialect` 的 `Dialect.set_isolation_level()` *方法*

给定一个 DBAPI 连接，设置其隔离级别。

注意，这是一个方言级方法，用作`Connection`和`Engine`隔离级别功能的实现的一部分；对于大多数典型用例，应优先使用这些 API。

如果方言还实现了`Dialect.get_isolation_level_values()`方法，则给定的级别将保证是该序列中的字符串名称之一，且该方法不需要预先考虑查找失败。

另请参阅

`Connection.get_isolation_level()` - 查看当前级别

`Connection.default_isolation_level` - 查看默认级别

`Connection.execution_options.isolation_level` - 设置每个`Connection`的隔离级别

`create_engine.isolation_level` - 设置每个`Engine`的隔离级别

```py
attribute statement_compiler
```

`SQLCompiler`的别名

```py
attribute supports_alter: bool = True
```

如果数据库支持`ALTER TABLE`，则为`True` - 仅在某些情况下用于生成外键约束

```py
attribute supports_comments: bool = False
```

表明方言支持对表和列进行评论的 DDL。

```py
attribute supports_constraint_comments: bool = False
```

表示方言是否支持对约束进行评论的 DDL。

```py
attribute supports_default_metavalue: bool = False
```

方言支持 INSERT… VALUES (DEFAULT)语法

```py
attribute supports_default_values: bool = False
```

方言支持 INSERT… DEFAULT VALUES 语法

```py
attribute supports_empty_insert: bool = True
```

方言支持 INSERT () VALUES ()

```py
attribute supports_identity_columns: bool = False
```

目标数据库支持 IDENTITY

```py
attribute supports_multivalues_insert: bool = False
```

目标数据库支持使用多个值集合的 INSERT…VALUES，即 INSERT INTO table (cols) VALUES (…), (…), (…), …

```py
attribute supports_native_boolean: bool = False
```

指示方言是否支持本地布尔构造。这将阻止 `Boolean` 在使用该类型时生成 CHECK 约束。

```py
attribute supports_native_decimal: bool = False
```

指示 Decimal 对象是否被处理并返回为精度数字类型，或者返回浮点数。

```py
attribute supports_native_enum: bool = False
```

指示方言是否支持本地 ENUM 构造。这将阻止 `Enum` 在“本地”模式下使用时生成 CHECK 约束。

```py
attribute supports_native_uuid: bool = False
```

指示 Python UUID() 对象是否由驱动程序原生处理 SQL UUID 数据类型。

新于版本 2.0。

```py
attribute supports_sane_multi_rowcount: bool = True
```

指示方言是否正确地实现了通过 executemany 执行的 `UPDATE` 和 `DELETE` 语句的 rowcount。

```py
attribute supports_sane_rowcount: bool = True
```

指示方言是否正确地实现了 `UPDATE` 和 `DELETE` 语句的 rowcount。

```py
attribute supports_sane_rowcount_returning
```

如果此方言支持使用 RETURNING，则即使使用 RETURNING，也支持合理的行数。

对于不支持 RETURNING 的方言，这与 `supports_sane_rowcount` 是同义词。

```py
attribute supports_sequences: bool = False
```

指示方言是否支持 CREATE SEQUENCE 或类似操作。

```py
attribute supports_server_side_cursors: bool = False
```

指示方言是否支持服务器端游标。

```py
attribute supports_simple_order_by_label: bool = True
```

目标数据库支持 ORDER BY <labelname>，其中 <labelname> 是 SELECT 的列子句中的标签。

```py
attribute supports_statement_cache: bool = True
```

指示此方言是否支持缓存。

所有与语句缓存兼容的方言都应该在每个支持它的方言类和子类上直接设置此标志为 True。SQLAlchemy 在使用语句缓存之前会测试每个方言子类上是否本地存在此标志。这是为了提供对尚未完全测试以符合 SQL 语句缓存的旧版或新版方言的安全性。

新于版本 1.4.5。

另请参阅

第三方方言的缓存

```py
attribute tuple_in_values: bool = False
```

目标数据库支持元组 IN，即 (x, y) IN ((q, p), (r, z))

```py
attribute type_compiler: Any
```

传统的；这是一个在类级别的 TypeCompiler 类，在实例级别的 TypeCompiler 实例。

参考 type_compiler_instance。

```py
attribute type_compiler_cls
```

`GenericTypeCompiler` 的别名。

```py
attribute type_compiler_instance: TypeCompiler
```

用于编译 SQL 类型对象的 `Compiled` 类的实例。

新于版本 2.0。

```py
method type_descriptor(typeobj)
```

提供一个特定于数据库的 `TypeEngine` 对象，给定来自 types 模块的通用对象。

此方法寻找一个名为 `colspecs` 的字典，作为类或实例级变量，并传递给 `adapt_type()`。

```py
attribute update_executemany_returning: bool = False
```

方言是否支持 UPDATE..RETURNING 与 executemany。

```py
attribute update_returning: bool = False
```

如果方言支持使用 RETURNING 的 UPDATE

新于版本 2.0。

```py
attribute update_returning_multifrom: bool = False
```

如果方言支持使用 RETURNING 的 UPDATE..FROM

新于版本 2.0。

```py
attribute use_insertmanyvalues: bool = False
```

如果为 True，则表示应使用“insertmanyvalues”功能，以允许进行`insert_executemany_returning`行为，如果可能的话。

在实践中，将其设置为 True 意味着：

如果`supports_multivalues_insert`、`insert_returning`和`use_insertmanyvalues`都为 True，则 SQL 编译器将生成一个 INSERT，由`DefaultDialect`解释为`ExecuteStyle.INSERTMANYVALUES`执行，允许通过将单行 INSERT 语句重写为具有多个 VALUES 子句来 INSERT 多行并在给定大量行时多次执行语句以进行一系列批处理。

对于默认方言，该参数为 False，并且对于 SQLAlchemy 内部方言 SQLite、MySQL/MariaDB、PostgreSQL、SQL Server，该参数设置为 True。对于提供本地“带 RETURNING 的 executemany”支持且不支持`supports_multivalues_insert`的 Oracle，该参数保持为 False。对于不支持 RETURNING 的 MySQL/MariaDB，这些 MySQL 方言将不会报告`insert_executemany_returning`为 True。

版本 2.0 中的新功能。

另请参见

“插入多个值”行为适用于 INSERT 语句

```py
attribute use_insertmanyvalues_wo_returning: bool = False
```

如果为 True，并且 use_insertmanyvalues 也为 True，则不包括 RETURNING 的 INSERT 语句也将使用“insertmanyvalues”。

版本 2.0 中的新功能。

另请参见

“插入多个值”行为适用于 INSERT 语句

```py
class sqlalchemy.engine.Dialect
```

定义特定数据库和 DB-API 组合的行为。

元数据定义、SQL 查询生成、执行、结果集处理或其他在数据库之间变化的任何方面都在方言的一般类别下定义。方言充当其他特定于数据库的对象实现的工厂，包括 ExecutionContext、Compiled、DefaultGenerator 和 TypeEngine。

注意

第三方方言不应直接子类化`Dialect`。而应该子类化`DefaultDialect`或后代类。

**成员**

bind_typing, colspecs, connect(), construct_arguments, create_connect_args(), create_xid(), cte_follows_insert, dbapi, dbapi_exception_translation_map, ddl_compiler, default_isolation_level, default_metavalue_token, default_schema_name, default_sequence_base, delete_executemany_returning, delete_returning, delete_returning_multifrom, denormalize_name(), div_is_floordiv, do_begin(), do_begin_twophase(), do_close(), do_commit(), do_commit_twophase(), do_execute(), do_execute_no_params(), do_executemany(), do_ping(), do_prepare_twophase(), do_recover_twophase(), do_release_savepoint(), do_rollback(), do_rollback_to_savepoint(), do_rollback_twophase(), do_savepoint(), do_set_input_sizes(), do_terminate(), driver, engine_config_types, engine_created(), exclude_set_input_sizes, execute_sequence_format, execution_ctx_cls, favor_returning_over_lastrowid, get_async_dialect_cls(), get_check_constraints(), get_columns(), get_default_isolation_level(), get_dialect_cls(), get_dialect_pool_class(), get_driver_connection(), get_foreign_keys(), get_indexes(), get_isolation_level(), get_isolation_level_values(), get_materialized_view_names(), get_multi_check_constraints(), get_multi_columns(), get_multi_foreign_keys(), get_multi_indexes(), get_multi_pk_constraint(), get_multi_table_comment(), get_multi_table_options(), get_multi_unique_constraints(), get_pk_constraint(), get_schema_names(), get_sequence_names(), get_table_comment(), get_table_names(), get_table_options(), get_temp_table_names(), get_temp_view_names(), get_unique_constraints(), get_view_definition(), get_view_names(), has_index(), has_schema(), has_sequence(), has_table(), has_terminate, identifier_preparer, import_dbapi(), include_set_input_sizes, initialize(), inline_comments, insert_executemany_returning, insert_executemany_returning_sort_by_parameter_order, insert_returning, insertmanyvalues_implicit_sentinel, insertmanyvalues_max_parameters, insertmanyvalues_page_size, is_async, is_disconnect(), label_length, load_provisioning(), loaded_dbapi, max_identifier_length, name, normalize_name(), on_connect(), on_connect_url(), paramstyle, positional, preexecute_autoincrement_sequences, preparer, reflection_options, reset_isolation_level(), returns_native_bytes, sequences_optional, server_side_cursors, server_version_info, set_connection_execution_options(), set_engine_execution_options(), set_isolation_level(), statement_compiler, supports_alter, supports_comments, supports_constraint_comments, supports_default_metavalue, supports_default_values, supports_empty_insert, supports_identity_columns, supports_multivalues_insert, supports_native_boolean, supports_native_decimal, supports_native_enum, supports_native_uuid, supports_sane_multi_rowcount, supports_sane_rowcount, supports_sequences, supports_server_side_cursors, supports_simple_order_by_label, supports_statement_cache, tuple_in_values, type_compiler, type_compiler_cls, type_compiler_instance, type_descriptor(), update_executemany_returning, update_returning, update_returning_multifrom, use_insertmanyvalues, use_insertmanyvalues_wo_returning

**类签名**

类`sqlalchemy.engine.Dialect`（`sqlalchemy.event.registry.EventTarget`）

```py
attribute bind_typing = 1
```

定义一种将类型信息传递给数据库和/或驱动程序以用于绑定参数的方法。

请参阅`BindTyping`以获取值。

2.0 版本中的新功能。

```py
attribute colspecs: MutableMapping[Type[TypeEngine[Any]], Type[TypeEngine[Any]]]
```

从 sqlalchemy.types 映射到特定于方言类的子类的 TypeEngine 类的字典。此字典仅在类级别上存在，不从方言实例本身访问。

```py
method connect(*cargs: Any, **cparams: Any) → DBAPIConnection
```

使用此方言的 DBAPI 建立连接。

此方法的默认实现是：

```py
def connect(self, *cargs, **cparams):
    return self.dbapi.connect(*cargs, **cparams)
```

`*cargs, **cparams`参数直接从此方言的`Dialect.create_connect_args()`方法生成。

当需要在从 DBAPI 获取新连接时执行程序化的每个连接步骤时，可以使用此方法进行方言处理。

参数：

+   `*cargs` – 从`Dialect.create_connect_args()`方法返回的位置参数

+   `**cparams` – 从`Dialect.create_connect_args()`方法返回的关键字参数。

返回：

一个 DBAPI 连接，通常来自[**PEP 249**](https://peps.python.org/pep-0249/)模块级别的`.connect()`函数。

另请参阅

`Dialect.create_connect_args()`

`Dialect.on_connect()`

```py
attribute construct_arguments: List[Tuple[Type[SchemaItem | ClauseElement], Mapping[str, Any]]] | None = None
```

各种 SQLAlchemy 构造的可选参数说明，通常是模式项。

要实现，建立为一系列元组，如下所示：

```py
construct_arguments = [
    (schema.Index, {
        "using": False,
        "where": None,
        "ops": None
    })
]
```

如果上述构造在 PostgreSQL 方言上建立，则`Index`构造现在将接受关键字参数`postgresql_using`、`postgresql_where`和`postgresql_ops`。任何其他指定给`Index`构造函数的以`postgresql_`为前缀的参数都将引发`ArgumentError`。

不包括`construct_arguments`成员的方言将不参与参数验证系统。对于这样的方言，所有参与构造的命名空间中以该方言名称为前缀的参数名称都被所有参与构造接受。这里的理由是，尚未实现此功能的第三方方言将继续以旧方式运行。

另请参阅

`DialectKWArgs` - 实现基类，消耗`DefaultDialect.construct_arguments`

```py
method create_connect_args(url: URL) → ConnectArgsType
```

构建与 DB-API 兼容的连接参数。

给定一个`URL`对象，返回一个元组，包含一个适合直接发送到 dbapi 的 connect 函数的`(*args, **kwargs)`。这些参数将发送到`Dialect.connect()`方法，然后运行 DBAPI 级别的`connect()`函数。

该方法通常使用`URL.translate_connect_args()`方法来生成一个选项字典。

默认实现为：

```py
def create_connect_args(self, url):
    opts = url.translate_connect_args()
    opts.update(url.query)
    return ([], opts)
```

参数：

**url** – 一个`URL`对象

返回：

一个`(*args, **kwargs)`元组，将传递给`Dialect.connect()`方法。

另请参阅

`URL.translate_connect_args()`

```py
method create_xid() → Any
```

创建一个两阶段事务 ID。

此 ID 将传递给 do_begin_twophase()、do_rollback_twophase()、do_commit_twophase()。其格式未指定。

```py
attribute cte_follows_insert: bool
```

目标数据库，在给定带有 INSERT 语句的 CTE 时，需要 CTE 位于 INSERT 语句下方。

```py
attribute dbapi: ModuleType | None
```

对 DBAPI 模块对象本身的引用。

SQLAlchemy 方言使用类方法`Dialect.import_dbapi()`导入 DBAPI 模块。其原理是任何方言模块都可以被导入并用于生成 SQL 语句，而无需安装实际的 DBAPI 驱动程序。只有在使用`create_engine()`构造`Engine`时才会导入 DBAPI；在那时，创建过程将把 DBAPI 模块分配给此属性。

因此，方言应该实现`Dialect.import_dbapi()`方法，该方法将导入必要的模块并返回它，然后在方言代码中引用`self.dbapi`以引用 DBAPI 模块内容。

自版本更改：`Dialect.dbapi`属性是作为每个`Dialect`实例对 DBAPI 模块的引用。以前未完全记录的`.Dialect.dbapi()`类方法已被弃用，并由`Dialect.import_dbapi()`替换。

```py
attribute dbapi_exception_translation_map: Mapping[str, str] = {}
```

一个名称字典，其值将包含 pep-249 异常的名称（“IntegrityError”，“OperationalError”等），键为备用类名，以支持 DBAPI 具有未按其引用命名的异常类的情况（例如，`IntegrityError = MyException`）。在绝大多数情况下，此字典为空。

```py
attribute ddl_compiler: Type[DDLCompiler]
```

用于编译 DDL 语句的`Compiled`类。

```py
attribute default_isolation_level: IsolationLevel | None
```

在新连接上隐式存在的隔离性。

```py
attribute default_metavalue_token: str = 'DEFAULT'
```

对于 INSERT… VALUES (DEFAULT)语法，放在括号中的标记。

例如，对于 SQLite，这是关键字“NULL”。

```py
attribute default_schema_name: str | None
```

默认模式的名称。此值仅适用于支持方言，并且通常在与数据库的初始连接期间填充。

```py
attribute default_sequence_base: int
```

将默认值转换为 CREATE SEQUENCE DDL 语句的“START WITH”部分。

```py
attribute delete_executemany_returning: bool
```

支持使用`executemany`的`DELETE..RETURNING`方言。

```py
attribute delete_returning: bool
```

如果方言支持带`DELETE`的`RETURNING`。

自版本 2.0 开始新增。

```py
attribute delete_returning_multifrom: bool
```

如果方言支持`DELETE..FROM`带`RETURNING`。

自版本 2.0 开始新增。

```py
method denormalize_name(name: str) → str
```

如果名称为全小写，则将给定名称转换为后端的不区分大小写的标识符。

仅当方言定义了`requires_name_normalize=True`时才会使用此方法。

```py
attribute div_is_floordiv: bool
```

目标数据库将`/`运算符视为“floor division”。

```py
method do_begin(dbapi_connection: PoolProxiedConnection) → None
```

提供给定 DB-API 连接的`connection.begin()`的实现。

DBAPI 没有专用的“begin”方法，并且预计事务是隐式的。为那些可能需要在此区域提供额外帮助的 DBAPI 提供此挂钩。

参数：

**dbapi_connection** – 一个 DBAPI 连接，通常在`ConnectionFairy`内代理。

```py
method do_begin_twophase(connection: Connection, xid: Any) → None
```

在给定连接上开始一个两阶段事务。

参数：

+   `connection` – 一个`Connection`。

+   `xid` – xid

```py
method do_close(dbapi_connection: DBAPIConnection) → None
```

提供给定 DBAPI 连接的`connection.close()`的实现。

当连接已从池中分离或超出池的正常容量返回时，`Pool`将调用此挂钩。

```py
method do_commit(dbapi_connection: PoolProxiedConnection) → None
```

提供给定 DB-API 连接的`connection.commit()`的实现。

参数：

**dbapi_connection** – 一个 DBAPI 连接，通常在`ConnectionFairy`内代理。

```py
method do_commit_twophase(connection: Connection, xid: Any, is_prepared: bool = True, recover: bool = False) → None
```

在给定连接上提交一个两阶段事务。

参数：

+   `connection` – 一个`Connection`。

+   `xid` – xid

+   `is_prepared` – 是否调用了`TwoPhaseTransaction.prepare()`。

+   `recover` – 如果传递了 recover 标志。

```py
method do_execute(cursor: DBAPICursor, statement: str, parameters: Sequence[Any] | Mapping[str, Any] | None, context: ExecutionContext | None = None) → None
```

提供 `cursor.execute(statement, parameters)` 的实现。

```py
method do_execute_no_params(cursor: DBAPICursor, statement: str, context: ExecutionContext | None = None) → None
```

提供 `cursor.execute(statement)` 的实现。

不应发送参数集合。

```py
method do_executemany(cursor: DBAPICursor, statement: str, parameters: Sequence[Sequence[Any]] | Sequence[Mapping[str, Any]], context: ExecutionContext | None = None) → None
```

提供 `cursor.executemany(statement, parameters)` 的实现。

```py
method do_ping(dbapi_connection: DBAPIConnection) → bool
```

检查 DBAPI 连接并在连接可用时返回 True。

```py
method do_prepare_twophase(connection: Connection, xid: Any) → None
```

在给定的连接上准备两阶段事务。

参数：

+   `connection` – 一个`Connection`。

+   `xid` – xid

```py
method do_recover_twophase(connection: Connection) → List[Any]
```

恢复在给定连接上未提交的准备好的两阶段事务标识符列表。

参数：

**connection** – 一个`Connection`。

```py
method do_release_savepoint(connection: Connection, name: str) → None
```

在连接上释放指定的保存点。

参数：

+   `connection` – 一个`Connection`。

+   `name` – 保存点名称。

```py
method do_rollback(dbapi_connection: PoolProxiedConnection) → None
```

给定 DB-API 连接，提供 `connection.rollback()` 的实现。

参数：

**dbapi_connection** – 一个 DBAPI 连接，通常在 `ConnectionFairy` 中被代理。

```py
method do_rollback_to_savepoint(connection: Connection, name: str) → None
```

将连接回滚到指定的保存点。

参数：

+   `connection` – 一个`Connection`。

+   `name` – 保存点名称。

```py
method do_rollback_twophase(connection: Connection, xid: Any, is_prepared: bool = True, recover: bool = False) → None
```

在给定连接上回滚两阶段事务。

参数：

+   `connection` – 一个`Connection`。

+   `xid` – xid

+   `is_prepared` – 是否调用了`TwoPhaseTransaction.prepare()`。

+   `recover` – 如果传递了 recover 标志。

```py
method do_savepoint(connection: Connection, name: str) → None
```

使用给定名称创建保存点。

参数：

+   `connection` – 一个`Connection`。

+   `name` – 保存点名称。

```py
method do_set_input_sizes(cursor: DBAPICursor, list_of_tuples: _GenericSetInputSizesType, context: ExecutionContext) → Any
```

调用 cursor.setinputsizes() 方法并传递适当的参数。

如果 `Dialect.bind_typing` 属性设置为 `BindTyping.SETINPUTSIZES` 值，则调用此挂钩。参数数据以元组列表形式传递（paramname、dbtype、sqltype），其中 `paramname` 是语句中参数的键，`dbtype` 是 DBAPI 数据类型，`sqltype` 是 SQLAlchemy 类型。元组的顺序是正确的参数顺序。

新版本 1.4 中新增。

2.0 版本中的更改：- 通过将 `Dialect.bind_typing` 设置为 `BindTyping.SETINPUTSIZES` 来启用 setinputsizes 模式。接受 `use_setinputsizes` 参数的方言应适当设置此值。

```py
method do_terminate(dbapi_connection: DBAPIConnection) → None
```

提供了一个实现 `connection.close()` 的方法，尽可能地避免阻塞，给定一个 DBAPI 连接。

在绝大多数情况下，这只是调用了 .close()，但对于一些 asyncio 方言可能调用了不同的 API 特性。

当连接正在被回收利用或已被废弃时，该钩子由 `Pool` 调用。

在版本 1.4.41 中新增。

```py
attribute driver: str
```

方言的 DBAPI 的标识名称

```py
attribute engine_config_types: Mapping[str, Any]
```

一个字符串键的映射，可以在引擎配置中链接到类型转换函数。

```py
classmethod engine_created(engine: Engine) → None
```

在返回最终 `Engine` 之前调用的一个方便的钩子。

如果方言从 `get_dialect_cls()` 方法返回了一个不同的类，则该钩子将在两个类上调用，首先在由 `get_dialect_cls()` 方法返回的方言类上调用，然后在调用该方法的类上调用。

该钩子应该由方言和/或包装器使用，以应用特殊事件到引擎或其组件。特别是，它允许方言包装类应用方言级别的事件。

```py
attribute exclude_set_input_sizes: Set[Any] | None
```

在自动 cursor.setinputsizes() 调用中应该排除的一组 DBAPI 类型对象。

仅在 bind_typing 为 BindTyping.SET_INPUT_SIZES 时使用。

```py
attribute execute_sequence_format: Type[Tuple[Any, ...]] | Type[Tuple[List[Any]]]
```

要么是 ‘tuple’ 类型，要么是 ‘list’ 类型，取决于 cursor.execute() 对于第二个参数接受的是什么（它们会变化）。

```py
attribute execution_ctx_cls: Type[ExecutionContext]
```

用于处理语句执行的 `ExecutionContext` 类。

```py
attribute favor_returning_over_lastrowid: bool
```

对于同时支持 lastrowid 和 RETURNING 插入策略的后端，优先考虑 RETURNING 用于简单的单整数主键插入。

cursor.lastrowid 在大多数后端上性能更好。

```py
classmethod get_async_dialect_cls(url: URL) → Type[Dialect]
```

给定一个 URL，返回将由异步引擎使用的 `Dialect`。

默认情况下，这是 `Dialect.get_dialect_cls()` 的别名，并且只是返回 cls。如果方言提供了同名的同步和异步版本，如 `psycopg` 驱动程序，则可能会使用它。

在版本 2 中新增。

另请参阅

`Dialect.get_dialect_cls()`

```py
method get_check_constraints(connection: Connection, table_name: str, schema: str | None = None, **kw: Any) → List[ReflectedCheckConstraint]
```

返回 `table_name` 中检查约束的信息。

给定一个字符串`table_name`和一个可选的字符串`schema`，返回检查约束信息，作为与`ReflectedCheckConstraint`字典对应的字典列表。

这是一种内部方言方法。应用程序应该使用`Inspector.get_check_constraints()`。

```py
method get_columns(connection: Connection, table_name: str, schema: str | None = None, **kw: Any) → List[ReflectedColumn]
```

返回`table_name`中的列信息。

给定一个`Connection`，一个字符串`table_name`和一个可选的字符串`schema`，返回列信息，作为与`ReflectedColumn`字典对应的字典列表。

这是一种内部方言方法。应用程序应该使用`Inspector.get_columns()`。

```py
method get_default_isolation_level(dbapi_conn: DBAPIConnection) → Literal['SERIALIZABLE', 'REPEATABLE READ', 'READ COMMITTED', 'READ UNCOMMITTED', 'AUTOCOMMIT']
```

给定一个 DBAPI 连接，返回其隔离级别，或者如果无法检索到隔离级别，则返回默认隔离级别。

此方法只能引发 NotImplementedError，并且**不能引发任何其他异常**，因为它在第一次连接时隐式使用。

该方法**必须返回一个值**，以便于支持隔离级别设置的方言，因为这个级别是在进行每个连接隔离级别更改时将要回滚到的级别。

该方法默认使用`Dialect.get_isolation_level()`方法，除非被方言覆盖。

在 1.3.22 版本中新增。

```py
classmethod get_dialect_cls(url: URL) → Type[Dialect]
```

给定一个 URL，返回将要使用的`Dialect`。

这是一个钩子，允许外部插件围绕现有方言提供功能，通过允许根据入口点从 URL 加载插件，然后插件返回实际要使用的方言。

默认情况下，这只是返回 cls。

```py
method get_dialect_pool_class(url: URL) → Type[Pool]
```

返回用于给定 URL 的 Pool 类。

```py
method get_driver_connection(connection: DBAPIConnection) → Any
```

返回由外部驱动程序包返回的连接对象。

对于使用 DBAPI 兼容驱动程序的普通方言，此调用将只返回作为参数传递的`connection`。 对于代替适配非 DBAPI 兼容驱动程序的方言，例如在适配异步驱动程序时，此调用将返回驱动程序返回的类似连接的对象。

在 1.4.24 版本中新增。

```py
method get_foreign_keys(connection: Connection, table_name: str, schema: str | None = None, **kw: Any) → List[ReflectedForeignKeyConstraint]
```

返回`table_name`中的 foreign_keys 信息。

给定一个`Connection`、一个字符串`table_name`和一个可选的字符串`schema`，返回外键信息，作为与`ReflectedForeignKeyConstraint`字典对应的字典列表。

这是一个内部方言方法。应用程序应该使用`Inspector.get_foreign_keys()`。

```py
method get_indexes(connection: Connection, table_name: str, schema: str | None = None, **kw: Any) → List[ReflectedIndex]
```

返回`table_name`中索引的信息。

给定一个`Connection`、一个字符串`table_name`和一个可选的字符串`schema`，返回索引信息，作为与`ReflectedIndex`字典对应的字典列表。

这是一个内部方言方法。应用程序应该使用`Inspector.get_indexes()`。

```py
method get_isolation_level(dbapi_connection: DBAPIConnection) → Literal['SERIALIZABLE', 'REPEATABLE READ', 'READ COMMITTED', 'READ UNCOMMITTED', 'AUTOCOMMIT']
```

给定一个 DBAPI 连接，返回其隔离级别。

在使用`Connection`对象时，可以使用`Connection.connection`访问器获取相应的 DBAPI 连接。

请注意，这是一个方言级别的方法，用作`Connection`和`Engine`隔离级别功能实现的一部分；这些 API 应该优先用于大多数典型用例。

另请参阅

`Connection.get_isolation_level()` - 查看当前级别

`Connection.default_isolation_level` - 查看默认级别

`Connection.execution_options.isolation_level` - 设置每个`Connection`的隔离级别

`create_engine.isolation_level` - 设置每个`Engine`的隔离级别

```py
method get_isolation_level_values(dbapi_conn: DBAPIConnection) → List[Literal['SERIALIZABLE', 'REPEATABLE READ', 'READ COMMITTED', 'READ UNCOMMITTED', 'AUTOCOMMIT']]
```

返回一个由此方言接受的字符串隔离级别名称序列。

可用的名称应遵循以下约定：

+   使用大写名称。隔离级别方法将接受小写名称，但在传递给方言之前将其规范化为大写。

+   应将分隔的单词用空格分隔，而不是下划线，例如`REPEATABLE READ`。隔离级别名称在传递给方言之前将下划线转换为空格。

+   四个标准隔离级别名称的名称，以它们受到后端支持的程度来说应该是`READ UNCOMMITTED` `READ COMMITTED`、`REPEATABLE READ`、`SERIALIZABLE`

+   如果方言支持自动提交选项，则应使用隔离级别名称`AUTOCOMMIT`。

+   其他隔离模式也可能存在，只要它们以大写命名，并使用空格而不是下划线。

此函数用于默认方言检查给定的隔离级别参数是否有效，否则会引发`ArgumentError`。

一个 DBAPI 连接被传递给该方法，在极少情况下，方言需要询问连接本身以确定此列表，但预计大多数后端将返回一个硬编码的值列表。如果方言支持“AUTOCOMMIT”，则该值也应该出现在返回的序列中。

该方法默认引发`NotImplementedError`。如果方言未实现此方法，则默认方言将不会在传递给`Dialect.set_isolation_level()`方法之前对给定的隔离级别值执行任何检查。这是为了与可能尚未实现此方法的第三方方言保持向后兼容性。

新版本 2.0 中。

```py
method get_materialized_view_names(connection: Connection, schema: str | None = None, **kw: Any) → List[str]
```

返回数据库中所有可用的物化视图名称列表。

这是一个内部方言方法。应用程序应使用`Inspector.get_materialized_view_names()`。

参数：

**模式** -

要查询的模式名称，如果不是默认模式。

新版本 2.0 中。

```py
method get_multi_check_constraints(connection: Connection, schema: str | None = None, filter_names: Collection[str] | None = None, **kw: Any) → Iterable[Tuple[TableKey, List[ReflectedCheckConstraint]]]
```

返回给定`schema`中所有表的检查约束的信息。

这是一个内部方言方法。应用程序应使用`Inspector.get_multi_check_constraints()`。

注意

`DefaultDialect` 提供了一个默认实现，将对由 `Dialect.get_table_names()`、`Dialect.get_view_names()` 或 `Dialect.get_materialized_view_names()` 返回的每个对象调用单个表方法，取决于提供的 `kind`。希望支持更快实现的方言应该实现此方法。

从版本 2.0 开始。

```py
method get_multi_columns(connection: Connection, schema: str | None = None, filter_names: Collection[str] | None = None, **kw: Any) → Iterable[Tuple[TableKey, List[ReflectedColumn]]]
```

返回给定 `schema` 中所有表中列的信息。

这是一个内部方言方法。应用程序应该使用 `Inspector.get_multi_columns()`。

注意

`DefaultDialect` 提供了一个默认实现，将对由 `Dialect.get_table_names()`、`Dialect.get_view_names()` 或 `Dialect.get_materialized_view_names()` 返回的每个对象调用单个表方法，取决于提供的 `kind`。希望支持更快实现的方言应该实现此方法。

从版本 2.0 开始。

```py
method get_multi_foreign_keys(connection: Connection, schema: str | None = None, filter_names: Collection[str] | None = None, **kw: Any) → Iterable[Tuple[TableKey, List[ReflectedForeignKeyConstraint]]]
```

返回给定 `schema` 中所有表中外键的信息。

这是一个内部方言方法。应用程序应该使用 `Inspector.get_multi_foreign_keys()`。

注意

`DefaultDialect` 提供了一个默认实现，将对由 `Dialect.get_table_names()`、`Dialect.get_view_names()` 或 `Dialect.get_materialized_view_names()` 返回的每个对象调用单个表方法，取决于提供的 `kind`。希望支持更快实现的方言应该实现此方法。

从版本 2.0 开始。

```py
method get_multi_indexes(connection: Connection, schema: str | None = None, filter_names: Collection[str] | None = None, **kw: Any) → Iterable[Tuple[TableKey, List[ReflectedIndex]]]
```

返回给定 `schema` 中所有表中索引的信息。

这是一个内部方言方法。应用程序应该使用 `Inspector.get_multi_indexes()`。

注意

`DefaultDialect`提供了一个默认实现，将根据提供的`kind`调用`Dialect.get_table_names()`、`Dialect.get_view_names()`或`Dialect.get_materialized_view_names()`中的每个对象的单表方法。希望支持更快实现的方言应该实现这个方法。

版本 2.0 中新增。

```py
method get_multi_pk_constraint(connection: Connection, schema: str | None = None, filter_names: Collection[str] | None = None, **kw: Any) → Iterable[Tuple[TableKey, ReflectedPrimaryKeyConstraint]]
```

返回给定`schema`中所有表的主键约束信息。

这是一个内部方言方法。应用程序应该使用`Inspector.get_multi_pk_constraint()`。

注意

`DefaultDialect`提供了一个默认实现，将根据提供的`kind`调用`Dialect.get_table_names()`、`Dialect.get_view_names()`或`Dialect.get_materialized_view_names()`中的每个对象的单表方法。希望支持更快实现的方言应该实现这个方法。

版本 2.0 中新增。

```py
method get_multi_table_comment(connection: Connection, schema: str | None = None, filter_names: Collection[str] | None = None, **kw: Any) → Iterable[Tuple[TableKey, ReflectedTableComment]]
```

返回给定`schema`中所有表的表注释信息。

这是一个内部方言方法。应用程序应该使用`Inspector.get_multi_table_comment()`。

注意

`DefaultDialect`提供了一个默认实现，将根据提供的`kind`调用`Dialect.get_table_names()`、`Dialect.get_view_names()`或`Dialect.get_materialized_view_names()`中的每个对象的单表方法。希望支持更快实现的方言应该实现这个方法。

版本 2.0 中新增。

```py
method get_multi_table_options(connection: Connection, schema: str | None = None, filter_names: Collection[str] | None = None, **kw: Any) → Iterable[Tuple[TableKey, Dict[str, Any]]]
```

返回在创建给定`schema`中的表时指定的选项字典。

这是一个内部方言方法。应用程序应该使用`Inspector.get_multi_table_options()`。

注意

`DefaultDialect`提供了一个默认实现，将根据提供的`kind`调用每个对象的单表方法`Dialect.get_table_names()`、`Dialect.get_view_names()`或`Dialect.get_materialized_view_names()`。想要支持更快实现的方言应该实现这个方法。

新版本 2.0 中引入。

```py
method get_multi_unique_constraints(connection: Connection, schema: str | None = None, filter_names: Collection[str] | None = None, **kw: Any) → Iterable[Tuple[TableKey, List[ReflectedUniqueConstraint]]]
```

返回给定`schema`中所有表的唯一约束信息。

这是一个内部方言方法。应用程序应该使用`Inspector.get_multi_unique_constraints()`。

注意

`DefaultDialect`提供了一个默认实现，将根据提供的`kind`调用每个对象的单表方法`Dialect.get_table_names()`、`Dialect.get_view_names()`或`Dialect.get_materialized_view_names()`。想要支持更快实现的方言应该实现这个方法。

新版本 2.0 中引入。

```py
method get_pk_constraint(connection: Connection, table_name: str, schema: str | None = None, **kw: Any) → ReflectedPrimaryKeyConstraint
```

返回`table_name`表上主键约束的信息。

给定一个`Connection`、一个字符串`table_name`和一个可选的字符串`schema`，返回作为字典对应于`ReflectedPrimaryKeyConstraint`字典的主键信息。

这是一个内部方言方法。应用程序应该使用`Inspector.get_pk_constraint()`。

```py
method get_schema_names(connection: Connection, **kw: Any) → List[str]
```

返回数据库中所有模式名称的列表。

这是一个内部方言方法。应用程序应该使用`Inspector.get_schema_names()`。

```py
method get_sequence_names(connection: Connection, schema: str | None = None, **kw: Any) → List[str]
```

返回数据库中所有序列名称的列表。

这是一个内部方言方法。应用程序应该使用`Inspector.get_sequence_names()`。

参数：

**schema** - 要查询的模式名称，如果不是默认模式。

新版本 1.4 中引入。

```py
method get_table_comment(connection: Connection, table_name: str, schema: str | None = None, **kw: Any) → ReflectedTableComment
```

返回由`table_name`标识的表的“注释”。

给定一个字符串`table_name`和一个可选的字符串`schema`，返回一个对应于`ReflectedTableComment`字典的表注释信息字典。

这是一个内部方言方法。应用程序应使用`Inspector.get_table_comment()`。

引发：

对于不支持注释的方言，抛出`NotImplementedError`。

版本 1.2 中的新功能。

```py
method get_table_names(connection: Connection, schema: str | None = None, **kw: Any) → List[str]
```

返回`schema`的表名称列表。

这是一个内部方言方法。应用程序应使用`Inspector.get_table_names()`。

```py
method get_table_options(connection: Connection, table_name: str, schema: str | None = None, **kw: Any) → Dict[str, Any]
```

返回创建`table_name`时指定的选项字典。

这是一个内部方言方法。应用程序应使用`Inspector.get_table_options()`。

```py
method get_temp_table_names(connection: Connection, schema: str | None = None, **kw: Any) → List[str]
```

返回给定连接上的临时表名称列表（如果底层后端支持）。

这是一个内部方言方法。应用程序应使用`Inspector.get_temp_table_names()`。

```py
method get_temp_view_names(connection: Connection, schema: str | None = None, **kw: Any) → List[str]
```

返回给定连接上的临时视图名称列表（如果底层后端支持）。

这是一个内部方言方法。应用程序应使用`Inspector.get_temp_view_names()`。

```py
method get_unique_constraints(connection: Connection, table_name: str, schema: str | None = None, **kw: Any) → List[ReflectedUniqueConstraint]
```

返回`table_name`中关于唯一约束的信息。

给定一个字符串`table_name`和一个可选的字符串`schema`，返回一个对应于`ReflectedUniqueConstraint`字典的唯一约束信息列表。

这是一个内部方言方法。应用程序应使用`Inspector.get_unique_constraints()`。

```py
method get_view_definition(connection: Connection, view_name: str, schema: str | None = None, **kw: Any) → str
```

返回普通或物化视图定义。

这是一个内部方言方法。应用程序应使用`Inspector.get_view_definition()`。

给定一个`Connection`对象，一个字符串`view_name`和一个可选的字符串`schema`，返回视图定义。

```py
method get_view_names(connection: Connection, schema: str | None = None, **kw: Any) → List[str]
```

返回数据库中所有非物化视图名称的列表。

这是一个内部方言方法。应用程序应使用`Inspector.get_view_names()`。

参数：

**schema** – 查询的模式名称，如果不是默认模式。

```py
method has_index(connection: Connection, table_name: str, index_name: str, schema: str | None = None, **kw: Any) → bool
```

检查数据库中特定索引名称的存在。

给定一个`Connection`对象，一个字符串`table_name`和字符串索引名称，如果给定表上存在给定名称的索引，则返回`True`，否则返回`False`。

`DefaultDialect`根据`Dialect.has_table()`和`Dialect.get_indexes()`方法实现此功能，但是方言可以实现更高效的版本。

这是一个内部方言方法。应用程序应该使用`Inspector.has_index()`。

新版本 1.4 中新增。

```py
method has_schema(connection: Connection, schema_name: str, **kw: Any) → bool
```

检查数据库中特定模式名称的存在。

给定一个`Connection`对象和一个字符串`schema_name`，如果给定的架构存在，则返回`True`，否则返回`False`。

`DefaultDialect`通过检查`Dialect.get_schema_names()`返回的模式中是否存在`schema_name`来实现此功能，但是方言可以实现更高效的版本。

这是一个内部方言方法。应用程序应该使用`Inspector.has_schema()`。

新版本 2.0 中新增。

```py
method has_sequence(connection: Connection, sequence_name: str, schema: str | None = None, **kw: Any) → bool
```

检查数据库中特定序列的存在。

给定一个`Connection`对象和一个字符串`sequence_name`，如果数据库中存在给定的序列，则返回`True`，否则返回`False`。

这是一个内部方言方法。应用程序应该使用`Inspector.has_sequence()`。

```py
method has_table(connection: Connection, table_name: str, schema: str | None = None, **kw: Any) → bool
```

对内部方言使用，检查数据库中特定表或视图的存在。

给定一个`Connection`对象，一个字符串`table_name`和可选的模式名称，如果数据库中存在给定的表，则返回`True`，否则返回`False`。

此方法作为公共面向用户的`Inspector.has_table()`方法的基础实现，并且还在内部用于实现像`Table.create()`和`MetaData.create_all()`等方法的“checkfirst”行为。

注意

此方法在 SQLAlchemy 内部使用，并公开以便第三方方言可以提供实现。这**不是**用于检查表存在的公共 API。请使用`Inspector.has_table()`方法。

从版本 2.0 开始更改：`Dialect.has_table()`现在正式支持检查其他类似表的对象：

+   任何类型的视图（普通或材料化）

+   任何类型的临时表

以前，这两个检查没有正式规定，不同的方言在行为上会有所不同。方言测试套件现在包括对所有这些对象类型的测试，以及在支持视图或临时表的程度上应该寻求支持定位这些对象以实现完全的兼容性。

```py
attribute has_terminate: bool
```

此方言是否具有单独的“终止”实现，不会阻塞或需要等待。

```py
attribute identifier_preparer: IdentifierPreparer
```

一旦构建了`DefaultDialect`的实例，此元素将引用一个`IdentifierPreparer`的实例。

```py
classmethod import_dbapi() → module
```

导入此方言使用的 DBAPI 模块。

此处返回的 Python 模块对象将分配为构建的方言的一个实例变量，名称为`.dbapi`。

从版本 2.0 开始更改：`Dialect.import_dbapi()`类方法从先前的方法`.Dialect.dbapi()`重命名，该方法将在方言实例化时由 DBAPI 模块本身替换，因此以两种不同的方式使用相同的名称。如果第三方方言上存在`.Dialect.dbapi()`类方法，将使用它并发出弃用警告。

```py
attribute include_set_input_sizes: Set[Any] | None
```

应包括在自动 cursor.setinputsizes()调用中的一组���该包含的 DBAPI 类型对象。

仅在 bind_typing 为 BindTyping.SET_INPUT_SIZES 时使用。

```py
method initialize(connection: Connection) → None
```

在连接的策略化创建期间调用与连接一起的方言。

允许方言根据服务器版本信息或其他属性配置选项。

此处传递的连接是一个完全具备功能的 SQLAlchemy Connection 对象。

应通过 super()调用基本方言的 initialize()方法。

注意

从 SQLAlchemy 1.4 开始，此方法在调用任何`Dialect.on_connect()`钩子之前被调用。

```py
attribute inline_comments: bool
```

表示方言支持与表或列的定义相一致的内联注释 DDL。如果为 False，则意味着必须使用 ALTER 来设置表和列的注释。

```py
attribute insert_executemany_returning: bool
```

当使用`dialect.do_executemany()`时，方言/驱动程序/数据库支持某种方式提供 INSERT…RETURNING 支持。

```py
attribute insert_executemany_returning_sort_by_parameter_order: bool
```

当使用`Insert.returning.sort_by_parameter_order`参数设置时，方言/驱动程序/数据库支持某种方式提供 INSERT…RETURNING 支持，同时使用`dialect.do_executemany()`。

```py
attribute insert_returning: bool
```

如果方言支持 INSERT 时的 RETURNING

2.0 版本中的新功能。

```py
attribute insertmanyvalues_implicit_sentinel: InsertmanyvaluesSentinelOpts
```

表示数据库支持一种形式的批量插入，其中自增整数主键可可靠用作插入行的排序。

2.0.10 版本中的新功能。

另请参阅

将 RETURNING 行与参数集相关联

```py
attribute insertmanyvalues_max_parameters: int
```

insertmanyvalues_page_size 的替代方案，还将基于语句中参数的总数限制页面大小。

```py
attribute insertmanyvalues_page_size: int
```

每个`ExecuteStyle.INSERTMANYVALUES`执行中渲染为单独的 INSERT..VALUES()语句的行数。

默认方言将此默认值设置为 1000。

2.0 版本中的新功能。

另请参阅

`Connection.execution_options.insertmanyvalues_page_size` - 在`Connection`上可用的执行选项，语句

```py
attribute is_async: bool
```

此方言是否用于 asyncio 使用。

```py
method is_disconnect(e: Exception, connection: PoolProxiedConnection | DBAPIConnection | None, cursor: DBAPICursor | None) → bool
```

如果给定的 DB-API 错误指示无效连接，则返回 True

```py
attribute label_length: int | None
```

SQL 标签的可选用户定义的最大长度

```py
classmethod load_provisioning() → None
```

为此方言设置 provision.py 模块。

对于包含设置提供程序跟随者的 provision.py 模块的方言，此方法应启动该过程。

典型的实现方式可能是：

```py
@classmethod
def load_provisioning(cls):
    __import__("mydialect.provision")
```

默认方法假定当前方言所属包内有一个名为`provision.py`的模块，基于`__module__`属性：

```py
@classmethod
def load_provisioning(cls):
    package = ".".join(cls.__module__.split(".")[0:-1])
    try:
        __import__(package + ".provision")
    except ImportError:
        pass
```

1.3.14 版本中的新功能。

```py
attribute loaded_dbapi
```

与.dbapi 相同，但永远不会是 None；如果没有设置 DBAPI，将会引发错误。

2.0 版本中的新功能。

```py
attribute max_identifier_length: int
```

标识符名称的最大长度。

```py
attribute name: str
```

从 DBAPI 中立的角度确定方言的标识名称（即‘sqlite’）

```py
method normalize_name(name: str) → str
```

如果检测到名称不区分大小写，则将给定名称转换为小写。

仅当方言定义 requires_name_normalize=True 时才使用此方法。

```py
method on_connect() → Callable[[Any], Any] | None
```

返回一个可调用对象，用于设置新创建的 DBAPI 连接。

可调用对象应接受一个名为“conn”的参数，即 DBAPI 连接本身。内部可调用对象没有返回值。

例如：

```py
class MyDialect(default.DefaultDialect):
    # ...

    def on_connect(self):
        def do_on_connect(connection):
            connection.execute("SET SPECIAL FLAGS etc")

        return do_on_connect
```

用于设置方言范围内的每个连接选项，如隔离模式、Unicode 模式等。

通过使用`PoolEvents.connect()`事件钩子调用“do_on_connect”可调用对象，然后解包 DBAPI 连接并将其传递给可调用对象。

1.4 版本中的更改：对于方言的第一个连接，不再两次调用 on_connect 挂钩。但是，在调用`Dialect.initialize()`方法之前仍会调用 on_connect 挂钩。

在版本 1.4.3 中更改：从一个新方法 on_connect_url 调用 on_connect 挂钩，传递用于创建连接参数的 URL。如果方言需要用于连接的 URL 对象以获取其他上下文，则方言可以实现 on_connect_url 而不是 on_connect。

如果返回 None，则不会生成事件监听器。

返回：

一个可接受单个 DBAPI 连接作为参数的可调用对象，或者为 None。

另请参阅

`Dialect.connect()` - 允许控制 DBAPI `connect()`序列本身。

`Dialect.on_connect_url()` - 取代`Dialect.on_connect()`以在上下文中还接收`URL`对象。

```py
method on_connect_url(url: URL) → Callable[[Any], Any] | None
```

返回一个可调用对象，该对象设置新创建的 DBAPI 连接。

这种方法是一种新的钩子，它在方言实现时取代了`Dialect.on_connect()`方法。当方言没有实现时，它会直接调用`Dialect.on_connect()`方法，以保持与现有方言的兼容性。不会有对`Dialect.on_connect()`的弃用预期。

可调用对象应接受一个名为“conn”的参数，该参数是 DBAPI 连接本身。内部可调用对象没有返回值。

例如：

```py
class MyDialect(default.DefaultDialect):
    # ...

    def on_connect_url(self, url):
        def do_on_connect(connection):
            connection.execute("SET SPECIAL FLAGS etc")

        return do_on_connect
```

此方法用于设置方言范围内的每个连接选项，如隔离模式、Unicode 模式等。

此方法与`Dialect.on_connect()`不同之处在于，它接收与连接参数相关的`URL`对象。通常从`Dialect.on_connect()`挂钩获取此对象的唯一方法是查看`Engine`本身，但此 URL 对象可能已被插件替换。

注意

`Dialect.on_connect_url()`的默认实现是调用`Dialect.on_connect()`方法。因此，如果一个方言实现了这个方法，`Dialect.on_connect()`方法 **将不会被调用**，除非覆盖方言从此处直接调用它。

版本 1.4.3 中新添加的 `Dialect.on_connect_url()` 通常调用 `Dialect.on_connect()`。

参数：

**url** – 一个表示传递给 `Dialect.create_connect_args()` 方法的 `URL` 对象。

返回：

一个接受单个 DBAPI 连接作为参数的可调用对象，或者为 None。

参见

`Dialect.on_connect()`

```py
attribute paramstyle: str
```

要使用的 paramstyle（一些 DB-API 支持多种 paramstyles）。

```py
attribute positional: bool
```

如果此 Dialect 的 paramstyle 是按位置的，则为 True。

```py
attribute preexecute_autoincrement_sequences: bool
```

如果‘implicit’主键函数必须单独执行以获取它们的值，如果未使用 RETURNING，则为真。

当在 `implicit_returning=False` 参数用于 `Table` 对象时，当前面向 PostgreSQL。

```py
attribute preparer: Type[IdentifierPreparer]
```

一个用于引用标识符的 `IdentifierPreparer` 类。

```py
attribute reflection_options: Sequence[str] = ()
```

表示可以在使用 `Table.autoload_with` 时作为“反射选项”传递给 `Table` 对象的关键字参数名称的字符串名称序列。

当在 Oracle dialect 中使用“oracle_resolve_synonyms”时，当前示例为真。

```py
method reset_isolation_level(dbapi_connection: DBAPIConnection) → None
```

给定一个 DBAPI 连接，将其隔离恢复为默认值。

请注意，这是一种方言级别的方法，作为 `Connection` 和 `Engine` 隔离级别功能实现的一部分使用；对于大多数典型用例，应优先使用这些 API。

参见

`Connection.get_isolation_level()` - 查看当前级别

`Connection.default_isolation_level` - 查看默认级别

`Connection.execution_options.isolation_level` - 设置每个 `Connection` 的隔离级别

`create_engine.isolation_level` - 设置每个`Engine`的隔离级别

```py
attribute returns_native_bytes: bool
```

指示 Python 的 bytes()对象是否由驱动程序原生返回 SQL“binary”数据类型。

版本 2.0.11 中的新功能。

```py
attribute sequences_optional: bool
```

如果为 True，则指示`Sequence`构造中的`Sequence`参数是否应该表示不生成 CREATE SEQUENCE。仅适用于支持序列的方言。目前仅用于允许在指定 Sequence()用于其他后端的列上使用 PostgreSQL SERIAL。

```py
attribute server_side_cursors: bool
```

已弃用；指示方言是否应尝试默认使用服务器端游标

```py
attribute server_version_info: Tuple[Any, ...] | None
```

包含正在使用的 DB 后端的版本号的元组。

此值仅适用于支持的方言，并且通常在与数据库的初始连接期间填充。

```py
method set_connection_execution_options(connection: Connection, opts: CoreExecuteOptionsParameter) → None
```

为给定连接建立执行选项。

这是由`DefaultDialect`实现的，以实现`Connection.execution_options.isolation_level`执行选项。方言可以拦截各种执行选项，这些选项可能需要修改特定 DBAPI 连接上的状态。

版本 1.4 中的新功能。

```py
method set_engine_execution_options(engine: Engine, opts: CoreExecuteOptionsParameter) → None
```

为给定引擎建立执行选项。

这是由`DefaultDialect`实现的，用于为给定`Engine`创建的新`Connection`实例建立事件钩子，然后将为该连接调用`Dialect.set_connection_execution_options()`方法。

```py
method set_isolation_level(dbapi_connection: DBAPIConnection, level: Literal['SERIALIZABLE', 'REPEATABLE READ', 'READ COMMITTED', 'READ UNCOMMITTED', 'AUTOCOMMIT']) → None
```

给定一个 DBAPI 连接，设置其隔离级别。

请注意，这是一个方言级方法，用作`Connection`和`Engine`隔离级别功能的实现的一部分；这些 API 应该优先用于大多数典型用例。

如果方言还实现了`Dialect.get_isolation_level_values()`方法，则给定级别将保证是该序列中的字符串名称之一，并且该方法不需要预期查找失败。

另请参见

`Connection.get_isolation_level()` - 查看当前级别

`Connection.default_isolation_level` - 查看默认级别

`Connection.execution_options.isolation_level` - 设置每个`Connection`的隔离级别

`create_engine.isolation_level` - 设置每个`Engine`的隔离级别

```py
attribute statement_compiler: Type[SQLCompiler]
```

用于编译 SQL 语句的`Compiled`类

```py
attribute supports_alter: bool
```

如果数据库支持`ALTER TABLE`，则为`True` - 仅在某些情况下用于生成外键约束

```py
attribute supports_comments: bool
```

表示方言是否支持对表和列的 DDL 注释。

```py
attribute supports_constraint_comments: bool
```

表示方言是否支持对约束的 DDL 注释。

```py
attribute supports_default_metavalue: bool
```

方言支持 INSERT…(col) VALUES (DEFAULT)语法。

大多数数据库都以某种方式支持此功能，例如 SQLite 使用`VALUES (NULL)`支持它。 MS SQL Server 也支持该语法，但它是唯一一个包含在内的方言，我们在其中禁用了此功能，因为 MSSQL 不支持 IDENTITY 列的字段，而通常我们喜欢利用该功能。

```py
attribute supports_default_values: bool
```

方言支持 INSERT… DEFAULT VALUES 语法。

```py
attribute supports_empty_insert: bool
```

方言支持 INSERT () VALUES ()，即没有列的普通 INSERT。

这通常不受支持；“空”插入通常使用“INSERT..DEFAULT VALUES”或“INSERT … (col) VALUES (DEFAULT)”来适用。

```py
attribute supports_identity_columns: bool
```

目标数据库支持 IDENTITY

```py
attribute supports_multivalues_insert: bool
```

目标数据库支持带有多个值集的 INSERT…VALUES，即 INSERT INTO table (cols) VALUES (…), (…), (…), …

```py
attribute supports_native_boolean: bool
```

表示方言是否支持原生布尔值构造。当使用该类型时，这将防止`Boolean`生成 CHECK 约束。

```py
attribute supports_native_decimal: bool
```

表示是否处理和返回十进制对象以获取精度数值类型，或者是否返回浮点数

```py
attribute supports_native_enum: bool
```

表示方言是否支持原生 ENUM 构造。当以“本地”模式使用该类型时，这将防止`Enum`生成 CHECK 约束。

```py
attribute supports_native_uuid: bool
```

指示 Python 的 UUID()对象是否由驱动程序本地处理以用于 SQL UUID 数据类型。

新版本 2.0 中新增。

```py
attribute supports_sane_multi_rowcount: bool
```

指示方言在通过 executemany 执行 UPDATE 和 DELETE 语句时是否正确实现了 rowcount。

```py
attribute supports_sane_rowcount: bool
```

指示方言在执行`UPDATE`和`DELETE`语句时是否正确实现了 rowcount。

```py
attribute supports_sequences: bool
```

表示方言是否支持 CREATE SEQUENCE 或类似功能。

```py
attribute supports_server_side_cursors: bool
```

表示方言是否支持服务器端游标。

```py
attribute supports_simple_order_by_label: bool
```

目标数据库支持 ORDER BY <labelname>，其中<labelname>指的是 SELECT 中列子句中的标签。

```py
attribute supports_statement_cache: bool = True
```

表示此方言是否支持缓存。

所有兼容语句缓存的方言都应直接在每个支持的方言类和子类上将此标志设置为 True。SQLAlchemy 在使用语句缓存之前会测试每个方言子类上是否存在此标志。这是为了对尚未完全测试以符合 SQL 语句缓存的旧版或新版方言提供安全性。

新版本 1.4.5 中新增。

另见

第三方方言的缓存

```py
attribute tuple_in_values: bool
```

目标数据库是否支持元组 IN，即(x, y) IN ((q, p), (r, z))。

```py
attribute type_compiler: Any
```

传统的；这是一个 TypeCompiler 类在类级别，一个 TypeCompiler 实例在实例级别。

请参阅 type_compiler_instance。

```py
attribute type_compiler_cls: ClassVar[Type[TypeCompiler]]
```

用于编译 SQL 类型对象的`Compiled`类。

新版本 2.0 中新增。

```py
attribute type_compiler_instance: TypeCompiler
```

`Compiled`类的实例用于编译 SQL 类型对象。

新版本 2.0 中新增。

```py
classmethod type_descriptor(typeobj: TypeEngine[_T]) → TypeEngine[_T]
```

将通用类型转换为特定于方言的类型。

方言类通常会使用类型模块中的`adapt_type()`函数来完成此操作。

返回的结果被缓存*每个方言类*，因此可能不包含方言实例状态。

```py
attribute update_executemany_returning: bool
```

方言支持具有`executemany`的 UPDATE..RETURNING。

```py
attribute update_returning: bool
```

如果方言支持带有 UPDATE 的 RETURNING。

新版本 2.0 中新增。

```py
attribute update_returning_multifrom: bool
```

如果方言支持带有 UPDATE..FROM 的 RETURNING。

新版本 2.0 中新增。

```py
attribute use_insertmanyvalues: bool
```

如果为 True，则表示应使用“insertmanyvalues”功能，以允许`insert_executemany_returning`行为，如果可能的话。

实际上，将此设置为 True 意味着：

如果`supports_multivalues_insert`、`insert_returning`和`use_insertmanyvalues`都为 True，则 SQL 编译器将生成一个 INSERT，该 INSERT 将由`DefaultDialect`解释为`ExecuteStyle.INSERTMANYVALUES`执行，通过重新编写单行 INSERT 语句以具有多个 VALUES 子句，当给定大量行时，还会多次执行该语句以进行一系列批处理。

该参数对于默认方言为 False，并且对于 SQLAlchemy 内部方言 SQLite、MySQL/MariaDB、PostgreSQL、SQL Server 设置为 True。对于提供原生“带 RETURNING 的 executemany”支持且不支持`supports_multivalues_insert`的 Oracle，它保持为 False。对于 MySQL/MariaDB，那些不支持 RETURNING 的 MySQL 方言将不会将`insert_executemany_returning`报告为 True。

版本 2.0 中的新功能。

另请参阅

INSERT 语句的“插入多个值”行为

```py
attribute use_insertmanyvalues_wo_returning: bool
```

如果为 True，并且 use_insertmanyvalues 也为 True，则不包括 RETURNING 的 INSERT 语句也将使用“insertmanyvalues”。

版本 2.0 中的新功能。

另请参阅

INSERT 语句的“插入多个值”行为

```py
class sqlalchemy.engine.default.DefaultExecutionContext
```

**成员**

编译, 连接, create_cursor(), current_parameters, cursor, dialect, engine, execute_style, executemany, execution_options, fetchall_for_returning(), get_current_parameters(), get_lastrowid(), get_out_parameter_values(), get_result_processor(), handle_dbapi_exception(), invoked_statement, isinsert, isupdate, lastrow_has_defaults(), no_parameters, parameters, post_exec(), postfetch_cols, pre_exec(), prefetch_cols, root_connection

**类签名**

类`sqlalchemy.engine.default.DefaultExecutionContext` (`sqlalchemy.engine.interfaces.ExecutionContext`)

```py
attribute compiled: Compiled | None = None
```

如果传递给构造函数，表示正在执行的 sqlalchemy.engine.base.Compiled 对象

```py
attribute connection: Connection
```

可以被默认值生成器自由使用以执行 SQL 的连接对象。此连接应引用与 root_connection 相同的底层连接/事务资源。

```py
method create_cursor()
```

从此 ExecutionContext 的连接生成一个新的游��。

一些方言可能希望更改 connection.cursor() 的行为，例如 postgresql 可能会返回一个 PG 的“服务器端”游标。

```py
attribute current_parameters: _CoreSingleExecuteParams | None = None
```

应用于当前行的参数字典。

此属性仅在用户定义的默认生成函数的上下文中可用，例如在 上下文敏感的默认函数 中描述的那样。它由一个字典组成，该字典包含要包含在 INSERT 或 UPDATE 语句中的每个列/值对的条目。字典的键将是每个 `Column` 的键值，这通常与名称同义。

请注意，`DefaultExecutionContext.current_parameters` 属性不适用于 `Insert.values()` 方法的“多值”特性。应优先使用 `DefaultExecutionContext.get_current_parameters()` 方法。

另请参阅

`DefaultExecutionContext.get_current_parameters()`

上下文敏感的默认函数

```py
attribute cursor: DBAPICursor
```

从连接获取的 DB-API 游标

```py
attribute dialect: Dialect
```

创建此执行上下文的方言。

```py
attribute engine: Engine
```

与连接关联的引擎

```py
attribute execute_style: ExecuteStyle = 0
```

将用于执行语句的 DBAPI 游标方法的风格。

新版本 2.0 中新增。

```py
attribute executemany: bool
```

如果上下文有多个参数集的列表，则为 True。

从历史上看，此属性链接到是否将使用 `cursor.execute()` 还是 `cursor.executemany()`。现在它还可以表示“insertmanyvalues”可能被使用，这表示一个或多个 `cursor.execute()` 调用。

```py
attribute execution_options: _ExecuteOptions = {}
```

与当前语句执行关联的执行选项

```py
method fetchall_for_returning(cursor)
```

对于 RETURNING 结果，请从 DBAPI 游标传递 `cursor.fetchall()`。

这是一个方言特定的钩子，用于调用“RETURNING”语句交付的行时有特殊考虑的方言。默认实现是 `cursor.fetchall()`。

此钩子目前仅由 insertmanyvalues 功能使用。不设置 `use_insertmanyvalues=True` 的方言不需要考虑此钩子。

新版本 2.0.10 中新增。

```py
method get_current_parameters(isolate_multiinsert_groups=True)
```

返回应用于当前行的参数字典。

该方法只能在用户定义的默认生成函数的上下文中使用，例如在 上下文敏感的默认函数 中描述的方式。调用时，将返回一个字典，该字典包含 INSERT 或 UPDATE 语句的每个列/值对的条目。字典的键将是每个 `Column` 的键值，通常与名称同义。

参数：

**isolate_multiinsert_groups=True** – 表示应通过仅返回与当前列默认调用相关的参数子集来处理使用 `Insert.values()` 创建的多值 INSERT 构造。当为 `False` 时，返回语句的原始参数，包括在多值 INSERT 情况下使用的命名约定。

版本 1.2 中新增了 `DefaultExecutionContext.get_current_parameters()` 方法，提供了比现有 `DefaultExecutionContext.current_parameters` 属性更多的功能。

另请参阅

`DefaultExecutionContext.current_parameters`

上下文敏感的默认函数

```py
method get_lastrowid()
```

在 INSERT 之后返回 self.cursor.lastrowid 或其等价值。

这可能涉及调用特殊的游标函数，在游标上发出新的 SELECT（或新的 SELECT），或者返回在 post_exec() 内计算的存储值。

该函数仅对支持“隐式”主键生成的方言调用，并且保持 preexecute_autoincrement_sequences 设置为 False，并且没有将显式 id 值绑定到语句时才会被调用。

对于那些使用 lastrowid 概念的方言，此函数在需要返回最后插入的主键的 INSERT 语句中被调用一次。在这些情况下，它会在 `ExecutionContext.post_exec()` 之后直接被调用。

```py
method get_out_parameter_values(names)
```

从游标返回 OUT 参数值的序列。

对于支持 OUT 参数的方言，当存在一个`SQLCompiler`对象，并且该对象的`SQLCompiler.has_out_parameters`标志被设置时，将调用此方法。反过来，如果语句本身具有`BindParameter`对象，并且这些对象的`.isoutparam`标志被`SQLCompiler.visit_bindparam()`方法使用，则此标志将设置为 True。如果方言编译器生成具有`.isoutparam`设置的`BindParameter`对象，但`SQLCompiler.visit_bindparam()`未处理，它应显式设置此标志。

为每个绑定参数呈现的名称列表传递给该方法。然后，该方法应返回与参数对象列表对应的值序列。与以前的 SQLAlchemy 版本不同，这些值可以是来自 DBAPI 的**原始值**；执行上下文将根据 self.compiled.binds 中的内容应用适当的类型处理程序并更新值。处理后的字典将通过结果对象上的`.out_parameters`集合提供。请注意，SQLAlchemy 1.4 作为 2.0 过渡的一部分具有多种结果对象。

版本 1.4 中新增：- 添加`ExecutionContext.get_out_parameter_values()`，当存在具有`.isoutparam`标志的`BindParameter`对象时，将自动由`DefaultExecutionContext`调用。这取代了在现在已删除的`get_result_proxy()`方法中设置输出参数的做法。

```py
method get_result_processor(type_, colname, coltype)
```

为游标描述中存在的给定类型返回一个‘结果处理器’。

这有一个默认实现，方言可以为上下文敏感的结果类型处理覆盖。

```py
method handle_dbapi_exception(e)
```

接收在执��、结果获取等过程中发生的 DBAPI 异常。

```py
attribute invoked_statement: Executable | None = None
```

最初给定的可执行语句对象。

结构上等同于 compiled.statement，但在缓存场景中，编译形式可能不是同一个对象。

```py
attribute isinsert: bool = False
```

如果语句是一个 INSERT，则返回 True。

```py
attribute isupdate: bool = False
```

如果语句是一个 UPDATE，则返回 True。

```py
method lastrow_has_defaults()
```

如果最后一个 INSERT 或 UPDATE 行包含内联或数据库端默认值，则返回 True。

```py
attribute no_parameters: bool
```

如果执行方式不使用参数，则返回 True。

```py
attribute parameters: _DBAPIMultiExecuteParams
```

传递给 execute()或 exec_driver_sql()方法的绑定参数。

这些始终被存储为参数条目的列表。单个元素列表对应于`cursor.execute()`调用，多个元素列表对应于`cursor.executemany()`，除非在`ExecuteStyle.INSERTMANYVALUES`的情况下将使用一个或多个`cursor.execute()`。

```py
method post_exec()
```

在编译语句执行后调用。

如果已编译的语句被传递给此 ExecutionContext，则在此方法完成后应该可以使用 last_insert_ids、last_inserted_params 等数据成员。

```py
attribute postfetch_cols: util.generic_fn_descriptor[Sequence[Column[Any]] | None]
```

列表，其中包含为其触发了服务器端默认值或内联 SQL 表达式值的 Column 对象。适用于插入和更新。

```py
method pre_exec()
```

在编译语句执行前调用。

如果已编译的语句被传递给此 ExecutionContext，则在此语句完成后必须初始化语句和参数数据成员。

```py
attribute prefetch_cols: util.generic_fn_descriptor[Sequence[Column[Any]] | None]
```

一个 Column 对象列表，其中为其触发了客户端默认值。适用于插入和更新。

```py
attribute root_connection: Connection
```

是此 ExecutionContext 的源的 Connection 对象。

```py
class sqlalchemy.engine.ExecutionContext
```

**成员**

compiled, connection, create_cursor(), cursor, dialect, engine, execute_style, executemany, execution_options, fetchall_for_returning(), fire_sequence(), get_out_parameter_values(), get_rowcount(), handle_dbapi_exception(), invoked_statement, isinsert, isupdate, lastrow_has_defaults(), no_parameters, parameters, post_exec(), postfetch_cols, pre_exec(), prefetch_cols, root_connection, statement

与单个执行对应的 Dialect 的信使对象。

```py
attribute compiled: Compiled | None
```

如果传递给构造函数，则执行中的 sqlalchemy.engine.base.Compiled 对象

```py
attribute connection: Connection
```

连接对象，可由默认值生成器自由使用以执行 SQL。此连接应引用与 root_connection 相同的基础连接/事务资源。

```py
method create_cursor() → DBAPICursor
```

从此 ExecutionContext 的连接生成新游标。

一些方言可能希望更改 connection.cursor() 的行为，例如可能返回 PG “服务器端”游标的 postgresql。

```py
attribute cursor: DBAPICursor
```

从连接获取的 DB-API 游标

```py
attribute dialect: Dialect
```

创建此 ExecutionContext 的方言。

```py
attribute engine: Engine
```

与 Connection 关联的引擎

```py
attribute execute_style: ExecuteStyle
```

将用于执行语句的 DBAPI 游标方法的样式。

版本 2.0 中的新功能。

```py
attribute executemany: bool
```

如果上下文具有超过一个参数集的列表，则为 True。

历史上，此属性与是否将使用 `cursor.execute()` 或 `cursor.executemany()` 相关联。现在它也可能意味着“insertmanyvalues”可能被使用，这表明一个或多个 `cursor.execute()` 调用。

```py
attribute execution_options: _ExecuteOptions
```

与当前语句执行相关联的执行选项

```py
method fetchall_for_returning(cursor: DBAPICursor) → Sequence[Any]
```

对于 RETURNING 结果，从 DBAPI 游标传递 cursor.fetchall()。

这是特定于方言的钩子，用于在调用“RETURNING”语句的交付行时具有特殊考虑因素的方言。默认实现是 `cursor.fetchall()`。

此钩子目前仅由 insertmanyvalues 功能使用。不设置 `use_insertmanyvalues=True` 的方言不需要考虑此钩子。

版本 2.0.10 中的新功能。

```py
method fire_sequence(seq: Sequence_SchemaItem, type_: Integer) → int
```

给定一个 `Sequence`，调用它并返回下一个 int 值。

```py
method get_out_parameter_values(out_param_names: Sequence[str]) → Sequence[Any]
```

从游标返回一系列 OUT 参数值。

对于支持 OUT 参数的方言，当存在具有设置了 `SQLCompiler` 对象的 `SQLCompiler.has_out_parameters` 标志的 `SQLCompiler` 对象时，将调用此方法。反过来，如果语句本身具有 `.isoutparam` 标志设置的 `BindParameter` 对象，并且这些对象由 `SQLCompiler.visit_bindparam()` 方法消耗，则将设置此标志为 True。如果方言编译器生成具有 `.isoutparam` 设置的 `BindParameter` 对象，并且不由 `SQLCompiler.visit_bindparam()` 处理，则应显式设置此标志。

将每个绑定参数的渲染名称列表传递给该方法。然后该方法应返回与参数对象列表对应的值序列。与之前的 SQLAlchemy 版本不同，这些值可以是来自 DBAPI 的**原始值**；执行上下文将根据 self.compiled.binds 中存在的内容应用适当的类型处理程序，并更新这些值。然后处理后的字典将通过结果对象上的`.out_parameters`集合提供。请注意，SQLAlchemy 1.4 在 2.0 过渡的一部分中有多种结果对象。

新版本 1.4 中新增：- 添加了 `ExecutionContext.get_out_parameter_values()`，当存在设置了`.isoutparam`标志的 `BindParameter` 对象时，将自动调用该方法。这取代了在现已删除的`get_result_proxy()`方法中设置输出参数的做法。

```py
method get_rowcount() → int | None
```

返回 DBAPI `cursor.rowcount` 值，或在某些情况下返回解释值。

有关此内容的详细信息，请参阅 `CursorResult.rowcount`。

```py
method handle_dbapi_exception(e: BaseException) → None
```

接收在执行、结果获取等过程中发生的 DBAPI 异常。

```py
attribute invoked_statement: Executable | None
```

首次给定的可执行语句对象。

这应该在结构上等同于 compiled.statement，但在缓存场景中不一定是相同的对象，因为编译形式将从缓存中提取出来。

```py
attribute isinsert: bool
```

如果语句是 INSERT，则返回 True。

```py
attribute isupdate: bool
```

如果语句是 UPDATE，则返回 True。

```py
method lastrow_has_defaults() → bool
```

如果最后一个 INSERT 或 UPDATE 行包含内联或数据库端默认值，则返回 True。

```py
attribute no_parameters: bool
```

如果执行样式不使用参数，则返回 True。

```py
attribute parameters: _AnyMultiExecuteParams
```

传递给 execute() 或 exec_driver_sql() 方法的绑定参数。

这些始终存储为参数条目的列表。单个元素列表对应于 `cursor.execute()` 调用，多个元素列表对应于 `cursor.executemany()`，除了在使用 `ExecuteStyle.INSERTMANYVALUES` 的情况下，它将一次或多次使用 `cursor.execute()`。

```py
method post_exec() → None
```

在编译语句执行后调用。

如果已将编译的语句传递给此执行上下文，则在此方法完成后，last_insert_ids、last_inserted_params 等数据成员应该可用。

```py
attribute postfetch_cols: util.generic_fn_descriptor[Sequence[Column[Any]] | None]
```

一组列对象，为其触发了服务器端默认值或内联 SQL 表达式值。适用于插入和更新操作。

```py
method pre_exec() → None
```

在编译语句执行之前调用。

如果已将编译的语句传递给此执行上下文，则在此语句完成后，必须初始化语句和参数数据成员。

```py
attribute prefetch_cols: util.generic_fn_descriptor[Sequence[Column[Any]] | None]
```

为客户端端触发了默认值的 Column 对象列表。适用于插入和更新。

```py
attribute root_connection: Connection
```

是此 ExecutionContext 来源的 Connection 对象。

```py
attribute statement: str
```

要执行的语句的字符串版本。要么传递给构造函数，要么必须在 pre_exec()完成时从 sql.Compiled 对象创建。

```py
class sqlalchemy.sql.compiler.ExpandedState
```

表示在为语句生成“扩展”和“编译后”绑定参数时使用的状态。

“扩展”参数是在语句执行时生成的参数，以适应传递的参数数量，最突出的例子是 IN 表达式中的各个元素。

“编译后”参数是在执行时将 SQL 文本值呈现到 SQL 语句中，而不是作为单独的参数传递给驱动程序的参数。

要创建一个`ExpandedState`实例，请在任何`SQLCompiler`实例上使用`SQLCompiler.construct_expanded_state()`方法。

**成员**

additional_parameters, parameter_expansion, parameters, positional_parameters, positiontup, processors, statement

**类签名**

类`sqlalchemy.sql.compiler.ExpandedState`（`builtins.tuple`）

```py
attribute additional_parameters
```

`ExpandedState.parameters`的同义词。

```py
attribute parameter_expansion: Mapping[str, List[str]]
```

表示从原始参数名称到“扩展”参数名称列表的中间链接的映射，用于那些已扩展的参数。

```py
attribute parameters: _CoreSingleExecuteParams
```

参数字典，参数已完全展开。

对于使用命名参数的语句，此字典将与语句中的名称完全匹配。对于使用位置参数的语句，`ExpandedState.positional_parameters`将生成一个具有位置参数集的元组。

```py
attribute positional_parameters
```

用于使用位置参数样式编译的语句的位置参数元组。

```py
attribute positiontup: Sequence[str] | None
```

一个指示位置参数顺序的字符串名称序列

```py
attribute processors: Mapping[str, _BindProcessorType[Any]]
```

绑定值处理器的映射

```py
attribute statement: str
```

字符串 SQL 语句，参数已完全展开

```py
class sqlalchemy.sql.compiler.GenericTypeCompiler
```

**成员**

ensure_kwarg

**类签名**

类`sqlalchemy.sql.compiler.GenericTypeCompiler`（`sqlalchemy.sql.compiler.TypeCompiler`）

```py
attribute ensure_kwarg: str = 'visit_\\w+'
```

*继承自* `TypeCompiler` *的* `TypeCompiler.ensure_kwarg` *属性*

一个指示方法名称的正则表达式，该方法应接受`**kw`参数。

该类将扫描与名称模板匹配的方法，并在必要时装饰它们，以确保接受`**kw`参数。

```py
class sqlalchemy.log.Identified
```

```py
class sqlalchemy.sql.compiler.IdentifierPreparer
```

**成员**

__init__(), format_column(), format_label_name(), format_schema(), format_table(), format_table_seq(), quote(), quote_identifier(), quote_schema(), schema_for_object, unformat_identifiers(), validate_sql_phrase()

根据选项处理标识符的引用和大小写折叠。

```py
method __init__(dialect, initial_quote='"', final_quote=None, escape_quote='"', quote_case_sensitive_collations=True, omit_schema=False)
```

构造一个新的`IdentifierPreparer`对象。

初始引号

开始定界标识符的字符。

最终引号

结束定界标识符的字符。默认为初始引号。

省略模式

防止添加模式名。适用于不支持模式的数据库。

```py
method format_column(column, use_table=False, name=None, table_name=None, use_schema=False, anon_map=None)
```

准备一个带引号的列名。

```py
method format_label_name(name, anon_map=None)
```

准备一个带引号的列名。

```py
method format_schema(name)
```

准备一个带引号的模式名。

```py
method format_table(table, use_schema=True, name=None)
```

准备一个带引号的表名和模式名。

```py
method format_table_seq(table, use_schema=True)
```

将表名和模式格式化为元组。

```py
method quote(ident: str, force: Any | None = None) → str
```

有条件地引用标识符。

如果标识符是保留字、包含引号必要字符或是一个包含`quote`设置为`True`的`quoted_name`的实例，则对标识符进行引用。

子类可以重写此方法，为标识符名称提供依赖于数据库的引用行为。

参数：

+   `ident` – 字符串标识符

+   `force` –

    未使用

    从版本 0.9 开始弃用：`IdentifierPreparer.quote.force` 参数已弃用，并将在将来的版本中移除。此标志对`IdentifierPreparer.quote()`方法的行为没有影响；请参考`quoted_name`。

```py
method quote_identifier(value: str) → str
```

引用标识符。

子类应重写此方法以提供特定于数据库的引号行为。

```py
method quote_schema(schema: str, force: Any | None = None) → str
```

有条件地引用模式名称。

如果名称是保留字，包含引号必要字符，或是包含`quote`设置为`True`的`quoted_name`的实例，则对名称进行引用。

子类可以重写此方法以提供特定于数据库的模式名称引用行为。

参数：

+   `schema` – 字符串模式名称

+   `force` –

    未使用

    从版本 0.9 开始弃用：`IdentifierPreparer.quote_schema.force` 参数已弃用，并将在将来的版本中移除。此标志对`IdentifierPreparer.quote()`方法的行为没有影响；请参考`quoted_name`。

```py
attribute schema_for_object: _SchemaForObjectCallable = operator.attrgetter('schema')
```

返回对象的.schema 属性。

对于默认的 IdentifierPreparer，对象的模式始终是“.schema”属性的值。如果准备器被替换为具有非空 schema_translate_map 的准备器，则“.schema”属性的值将被呈现为一个符号，该符号将在编译后从映射转换为真实模式名称。

```py
method unformat_identifiers(identifiers)
```

将类似‘schema.table.column’的字符串解包成组件。

```py
method validate_sql_phrase(element, reg)
```

关键字序列过滤器。

用于表示关键字序列的元素的过滤器，例如“INITIALLY”，“INITIALLY DEFERRED”等。不应包含任何特殊字符。

版本 1.3 中的新功能。

```py
class sqlalchemy.sql.compiler.SQLCompiler
```

`Compiled`的默认实现。

将`ClauseElement`对象编译成 SQL 字符串。

**成员**

__init__(), ansi_bind_rules, bind_names, bindname_escape_characters, binds, bindtemplate, compilation_bindtemplate, construct_expanded_state(), construct_params(), current_executable, default_from(), delete_extra_from_clause(), effective_returning, escaped_bind_names, get_select_precolumns(), group_by_clause(), has_out_parameters, implicit_returning, insert_prefetch, insert_single_values_expr, isupdate, literal_execute_params, order_by_clause(), params, positiontup, post_compile_params, postfetch, postfetch_lastrowid, render_literal_value(), render_table_with_column_in_update_from, returning, returning_precedes_values, sql_compiler, stack, translate_select_structure, update_from_clause(), update_limit_clause(), update_prefetch, update_tables_clause(), visit_override_binds()

**类签名**

类`sqlalchemy.sql.compiler.SQLCompiler`（`sqlalchemy.sql.compiler.Compiled`）

```py
method __init__(dialect: Dialect, statement: ClauseElement | None, cache_key: CacheKey | None = None, column_keys: Sequence[str] | None = None, for_executemany: bool = False, linting: Linting = Linting.NO_LINTING, _supporting_against: SQLCompiler | None = None, **kwargs: Any)
```

构造一个新的`SQLCompiler`对象。

参数：

+   `dialect` – 要使用的`Dialect`

+   `statement` – 要编译的`ClauseElement`

+   `column_keys` – 一个列名列表，将被编译成一个 INSERT 或 UPDATE 语句。

+   `for_executemany` – INSERT / UPDATE 语句是否应该期望以“executemany”样式调用，这可能会影响语句如何期望返回默认值和自增/序列等的值。根据使用的后端和驱动程序，检索这些值的支持可能已禁用，这意味着 SQL 表达式可能会被内联渲染，RETURNING 可能不会被渲染等。

+   `kwargs` – 要被超类消耗的额外关键字参数。

```py
attribute ansi_bind_rules: bool = False
```

SQL 92 不允许在 SELECT 的列子句中使用绑定参数，也不允许模糊表达式如“? = ?”。如果目标驱动程序/数据库强制执行此规则，编译器子类可以将此标志设置为 False

```py
attribute bind_names: Dict[BindParameter[Any], str]
```

一个包含“编译”名称的 BindParameter 实例字典，这些名称实际上出现在生成的 SQL 中

```py
attribute bindname_escape_characters: ClassVar[Mapping[str, str]] = {' ': '_', '%': 'P', '(': 'A', ')': 'Z', '.': '_', ':': 'C', '[': '_', ']': '_'}
```

一个映射（例如字典或类似结构），包含一个字符查找表，键为替换字符，这些字符将被应用于所有 SQL 语句中使用的“绑定名称”作为一种“转义”；当在 SQL 语句中呈现时，给定字符将完全替换为“替换”字符，并且对传递给`Connection.execute()`等方法的参数字典中使用的传入名称执行类似的转换。

这允许在`bindparam()`和其他构造中使用的绑定参数名称具有任意字符，而不必担心目标数据库上根本不允许的字符。

第三方方言可以在此处建立自己的字典以替换默认映射，这将确保映射中的特定字符永远不会出现在绑定参数名称中。

字典在**类创建时**进行评估，因此不能在运行时修改；在类首次声明时，必须存在于类上。

请注意，对于具有额外绑定参数规则的方言，例如对前导字符有额外限制的方言，可能需要增强`SQLCompiler.bindparam_string()`方法。请参考 cx_Oracle 编译器的示例。

在版本 2.0.0rc1 中新增。

```py
attribute binds: Dict[str, BindParameter[Any]]
```

一个将绑定参数键与 BindParameter 实例关联的字典。

```py
attribute bindtemplate: str
```

根据 paramstyle 渲染绑定参数的模板。

```py
attribute compilation_bindtemplate: str
```

编译器在应用位置参数样式之前渲染参数时使用的模板。

```py
method construct_expanded_state(params: _CoreSingleExecuteParams | None = None, escape_names: bool = True) → ExpandedState
```

为给定参数集返回一个新的`ExpandedState`。

对于使用“扩展”或其他晚期渲染参数的查询，此方法将提供特定参数集的最终 SQL 字符串以及将用于该特定参数集的参数。

自版本 2.0.0rc1 起新增。

```py
method construct_params(params: _CoreSingleExecuteParams | None = None, extracted_parameters: Sequence[BindParameter[Any]] | None = None, escape_names: bool = True, _group_number: int | None = None, _check: bool = True, _no_postcompile: bool = False) → _MutableCoreSingleExecuteParams
```

返回一个绑定参数键和值的字典。

```py
attribute current_executable
```

返回当前正在编译的“可执行”。

当前正在编译的是`Select`、`Insert`、`Update`、`Delete`、`CompoundSelect`对象。具体来说，它被分配给`self.stack`元素列表。

当编译类似上述语句时，通常也会分配给`Compiler`对象的`.statement`属性。然而，所有 SQL 构造最终都是可嵌套的，`visit_`方法永远不应该查询此属性，因为不能保证已分配，也不能保证与当前正在编译的语句对应。

自版本 1.3.21 起新增：为了与旧版本兼容，请使用以下方法：

```py
statement = getattr(self, "current_executable", False)
if statement is False:
    statement = self.stack[-1]["selectable"]
```

对于 1.4 及以上版本，请确保仅使用`.current_executable`；“self.stack”的格式可能会更改。

```py
method default_from()
```

当 SELECT 语句没有 froms，并且不需要附加 FROM 子句时调用。

给 Oracle 一个机会将`FROM DUAL`附加到字符串输出中。

```py
method delete_extra_from_clause(update_stmt, from_table, extra_froms, from_hints, **kw)
```

提供一个钩子来覆盖生成 DELETE..FROM 子句。

这可以用来实现 DELETE..USING 等。

MySQL 和 MSSQL 会覆盖此方法。

```py
attribute effective_returning
```

INSERT、UPDATE 或 DELETE 的有效“返回”列。

这是所谓的“隐式返回”列，编译器根据需要在运行时计算的列，或者基于`self.statement._returning`中存在的列（使用`._all_selected_columns`属性展开为单独的列），即使用`UpdateBase.returning()`方法明确设置的列。

自版本 2.0 起新增。

```py
attribute escaped_bind_names: util.immutabledict[str, str] = {}
```

晚期转义绑定参数名称，必须在查看参数字典时转换为原始名称。

```py
method get_select_precolumns(select, **kw)
```

在构建`SELECT`语句时调用，位置位于列列表之前。

```py
method group_by_clause(select, **kw)
```

允许方言自定义如何渲染 GROUP BY。

```py
attribute has_out_parameters = False
```

如果为 True，则存在具有 isoutparam 标志设置的 bindparam() 对象。

```py
attribute implicit_returning: Sequence[ColumnElement[Any]] | None = None
```

用于顶级 INSERT 或 UPDATE 语句的“隐式”返回列的列表，用于接收新生成的列值。

从版本 2.0 开始：`implicit_returning` 替换了先前的 `returning` 集合，后者不是一个通用的 RETURNING 集合，而实际上是特定于“隐式返回”功能的。

```py
attribute insert_prefetch: Sequence[Column[Any]] = ()
```

列表，应在执行 INSERT 之前评估默认值的列。

```py
attribute insert_single_values_expr
```

当 INSERT 与一个参数集合编译到一个 VALUES 表达式内时，该字符串被分配在此处，可以在插入批处理方案中用于重写 VALUES 表达式。

从版本 1.3.8 开始。

从版本 2.0 开始更改：此集合不再由 SQLAlchemy 内置方言使用，而是使用仅由 `SQLCompiler` 内部当前使用的 `_insertmanyvalues` 集合。

```py
attribute isupdate: bool = False
```

类级别的默认值，可以在实例级别设置以定义此 Compiled 实例是否表示 INSERT/UPDATE/DELETE。

```py
attribute literal_execute_params: FrozenSet[BindParameter[Any]] = frozenset({})
```

在语句执行时将渲染为文字值的 bindparameter 对象。

```py
method order_by_clause(select, **kw)
```

允许方言定制如何呈现 ORDER BY。

```py
attribute params
```

返回嵌入到此编译对象中的绑定参数字典，用于那些存在的值。

另请参阅

如何将 SQL 表达式呈现为字符串，可能包含内联的绑定参数？ - 包含用于调试用例的用法示例。

```py
attribute positiontup: List[str] | None = None
```

对于使用位置参数风格的已编译构造，将是一个字符串序列，指示按顺序绑定参数的名称。

用于以正确顺序呈现绑定参数，并与 `Compiled.params` 字典结合使用以呈现参数。

此序列始终包含参数的未转义名称。

另请参阅

如何将 SQL 表达式呈现为字符串，可能包含内联的绑定参数？ - 包含用于调试用例的用法示例。

```py
attribute post_compile_params: FrozenSet[BindParameter[Any]] = frozenset({})
```

在语句执行时将呈现为绑定参数占位符的 bindparameter 对象。

```py
attribute postfetch: List[Column[Any]] | None
```

列表，可在 INSERT 或 UPDATE 后进行后提取以接收服务器更新的值。

```py
attribute postfetch_lastrowid = False
```

如果为 True，并且这是插入操作，则使用 cursor.lastrowid 来填充 result.inserted_primary_key。

```py
method render_literal_value(value, type_)
```

将绑定参数的值呈现为带引号的文字。

这用于目标驱动程序/数据库上不接受绑定参数的语句部分。

应由子类使用 DBAPI 的引用服务来实现此功能。

```py
attribute render_table_with_column_in_update_from: bool = False
```

设置为 True 可以类别地表示多表 UPDATE 语句中的 SET 子句应该使用表名限定列（即仅适用于 MySQL）。

```py
attribute returning
```

向后兼容性；返回有效的返回集合。

```py
attribute returning_precedes_values: bool = False
```

设置为 True 可以类别地在 VALUES 或 WHERE 子句之前生成 RETURNING 子句（即 MSSQL）。

```py
attribute sql_compiler
```

```py
attribute stack: List[_CompilerStackEntry]
```

如 SELECT、INSERT、UPDATE、DELETE 等主要语句使用条目格式在此堆栈中进行跟踪。

```py
attribute translate_select_structure: Any = None
```

如果不是 `None`，应该是一个可调用对象，接受 `(select_stmt, **kw)` 并返回一个 select 对象。这主要用于结构性变更，主要是为了适应 LIMIT/OFFSET 方案。

```py
method update_from_clause(update_stmt, from_table, extra_froms, from_hints, **kw)
```

提供一个钩子来覆盖生成 UPDATE..FROM 子句。

MySQL 和 MSSQL 覆盖此项。

```py
method update_limit_clause(update_stmt)
```

为 MySQL 提供一个钩子以在 UPDATE 中添加 LIMIT

```py
attribute update_prefetch: Sequence[Column[Any]] = ()
```

在 UPDATE 发生之前应评估 onupdate 默认值的列列表

```py
method update_tables_clause(update_stmt, from_table, extra_froms, **kw)
```

提供一个钩子来覆盖 UPDATE 语句中的初始表子句。

MySQL 覆盖此项。

```py
method visit_override_binds(override_binds, **kw)
```

SQL 编译 OverrideBinds 的嵌套元素，并交换绑定参数。

OverrideBinds 通常不会被编译；它的使用是指当已经缓存的语句要被使用时，编译已经执行过，只需在执行时交换绑定参数。

然而，有测试用例会使用这个对象，而且 ORM 子查询加载器已知会在新查询中添加包含此结构的表达式（在 #11173 中发现），所以它也必须在编译时做正确的事情。

```py
class sqlalchemy.sql.compiler.StrSQLCompiler
```

`SQLCompiler` 的一个子类，允许一小部分非标准 SQL 功能渲染为字符串值。

当 Core 表达式元素直接字符串化而不调用 `ClauseElement.compile()` 方法时，将调用 `StrSQLCompiler`。它可以渲染一组有限的非标准 SQL 构造以协助基本字符串化，但是对于更重要的自定义或方言特定的 SQL 构造，需要直接使用 `ClauseElement.compile()`。

另请参阅

如何将 SQL 表达式呈现为字符串，可能还包含内联的绑定参数？

**成员**

delete_extra_from_clause()，update_from_clause()

**类签名**

类 `sqlalchemy.sql.compiler.StrSQLCompiler`（`sqlalchemy.sql.compiler.SQLCompiler`）。

```py
method delete_extra_from_clause(update_stmt, from_table, extra_froms, from_hints, **kw)
```

提供一个钩子来覆盖生成 DELETE..FROM 子句。

这可用于实现 DELETE..USING 等。

MySQL 和 MSSQL 覆盖此项。

```py
method update_from_clause(update_stmt, from_table, extra_froms, from_hints, **kw)
```

提供一个钩子来覆盖生成 UPDATE..FROM 子句。

MySQL 和 MSSQL 覆盖此项。

```py
class sqlalchemy.engine.AdaptedConnection
```

支持 DBAPI 协议的适配连接对象的接口。

用于 asyncio 方言，以在驱动程序提供的 asyncio 连接/游标 API 之上提供同步风格的 pep-249 门面。

**成员**

driver_connection, run_async()

版本 1.4.24 中的新功能。

```py
attribute driver_connection
```

连接对象是驱动程序在连接后返回的对象。

```py
method run_async(fn: Callable[[Any], Awaitable[_T]]) → _T
```

运行给定函数返回的可等待对象，该函数接收原始的 asyncio 驱动程序连接。

用于在“同步”方法的上下文中调用驱动程序连接上的仅可等待方法，例如连接池事件处理程序。

例如：

```py
engine = create_async_engine(...)

@event.listens_for(engine.sync_engine, "connect")
def register_custom_types(dbapi_connection, ...):
    dbapi_connection.run_async(
        lambda connection: connection.set_type_codec(
            'MyCustomType', encoder, decoder, ...
        )
    )
```

版本 1.4.30 中的新功能。

另请参见

在连接池和其他事件中使用仅可等待的驱动程序方法
