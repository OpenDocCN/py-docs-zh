# Session API

> 原文：[`docs.sqlalchemy.org/en/20/orm/session_api.html`](https://docs.sqlalchemy.org/en/20/orm/session_api.html)

## Session 和 sessionmaker()

| 对象名称 | 描述 |
| --- | --- |
| ORMExecuteState | 表示对`Session.execute()`方法的调用，作为传递给`SessionEvents.do_orm_execute()`事件钩子的参数。 |
| Session | 管理 ORM 映射对象的持久化操作。 |
| sessionmaker | 可配置的`Session`工厂。 |
| SessionTransaction | 一个`Session`级别的事务。 |
| SessionTransactionOrigin | 表示`SessionTransaction`的来源。 |

```py
class sqlalchemy.orm.sessionmaker
```

可配置的`Session`工厂。

`sessionmaker`工厂在调用时生成新的`Session`对象，在此处建立的配置参数的基础上创建它们。

例如：

```py
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

# an Engine, which the Session will use for connection
# resources
engine = create_engine('postgresql+psycopg2://scott:tiger@localhost/')

Session = sessionmaker(engine)

with Session() as session:
    session.add(some_object)
    session.add(some_other_object)
    session.commit()
```

上下文管理器的使用是可选的；否则，通过`Session.close()`方法可以显式关闭返回的`Session`对象。使用`try:/finally:`块是可选的，但是会确保即使存在数据库错误，关闭也会发生：

```py
session = Session()
try:
    session.add(some_object)
    session.add(some_other_object)
    session.commit()
finally:
    session.close()
```

`sessionmaker`充当`Engine`充当`Connection`对象的工厂的工厂。以这种方式，它还包括一个`sessionmaker.begin()`方法，提供一个上下文管理器，该管理器既开始又提交事务，完成后关闭`Session`，如果出现任何错误，则回滚事务：

```py
Session = sessionmaker(engine)

with Session.begin() as session:
    session.add(some_object)
    session.add(some_other_object)
# commits transaction, closes session
```

新版本 1.4 中新增。

当调用`sessionmaker`来构造一个`Session`时，也可以传递关键字参数给方法；这些参数将覆盖全局配置的参数。下面我们使用一个绑定到某个`Engine`的`sessionmaker`来生成一个`Session`，而该`Session`则绑定到从该引擎获取的特定`Connection`上：

```py
Session = sessionmaker(engine)

# bind an individual session to a connection

with engine.connect() as connection:
    with Session(bind=connection) as session:
        # work with session
```

该类还包括一个方法`sessionmaker.configure()`，用于指定工厂的其他关键字参数，这些参数将对生成的后续`Session`对象生效。通常用于在首次使用之前将一个或多个`Engine`对象与现有的`sessionmaker`工厂关联起来：

```py
# application starts, sessionmaker does not have
# an engine bound yet
Session = sessionmaker()

# ... later, when an engine URL is read from a configuration
# file or other events allow the engine to be created
engine = create_engine('sqlite:///foo.db')
Session.configure(bind=engine)

sess = Session()
# work with session
```

另请参阅

打开和关闭会话 - 关于使用`sessionmaker`创建会话的介绍性文本。

**成员**

__call__(), __init__(), begin(), close_all(), configure(), identity_key(), object_session()

**类签名**

类`sqlalchemy.orm.sessionmaker`（`sqlalchemy.orm.session._SessionClassMethods`, `typing.Generic`）

```py
method __call__(**local_kw: Any) → _S
```

使用在这个`sessionmaker`中建立的配置生成一个新的`Session`对象。

在 Python 中，当对象“被调用”时，会调用`__call__`方法，其方式与函数相同：

```py
Session = sessionmaker(some_engine)
session = Session()  # invokes sessionmaker.__call__()
```

```py
method __init__(bind: Optional[_SessionBind] = None, *, class_: Type[_S] = <class 'sqlalchemy.orm.session.Session'>, autoflush: bool = True, expire_on_commit: bool = True, info: Optional[_InfoType] = None, **kw: Any)
```

构造一个新的`sessionmaker`。

这里的所有参数，除了`class_`之外，都与`Session`直接接受的参数相对应。有关参数的更多详细信息，请参阅`Session.__init__()`文档字符串。

参数：

+   `bind` – 一个`Engine`或其他`Connectable`，新创建的`Session`对象将与之关联。

+   `class_` – 用于创建新的`Session`对象的类。默认为`Session`。

+   `autoflush` –

    用于新创建的`Session`对象的自动刷新设置。

    另请参阅

    刷新 - 关于自动刷新的额外背景信息

+   `expire_on_commit=True` – 用于新创建的`Session`对象的`Session.expire_on_commit`设置。

+   `info` – 可选信息字典，将通过`Session.info`可用。请注意，当指定`info`参数进行特定`Session`构造操作时，此字典将被*更新*，而不是替换。

+   `**kw` – 所有其他关键字参数都传递给新创建的`Session`对象的构造函数。

```py
method begin() → AbstractContextManager[_S]
```

生成一个上下文管理器，既提供一个新的`Session`，又提供一个提交的事务。

例如：

```py
Session = sessionmaker(some_engine)

with Session.begin() as session:
    session.add(some_object)

# commits transaction, closes session
```

版本 1.4 中的新功能。

```py
classmethod close_all() → None
```

*继承自* `sqlalchemy.orm.session._SessionClassMethods.close_all` *方法的* `sqlalchemy.orm.session._SessionClassMethods`

关闭内存中的*所有*会话。

自版本 1.3 起已弃用：`Session.close_all()`方法已弃用，并将在将来的版本中删除。请参考`close_all_sessions()`。

```py
method configure(**new_kw: Any) → None
```

(重新)配置此 sessionmaker 的参数。

例如：

```py
Session = sessionmaker()

Session.configure(bind=create_engine('sqlite://'))
```

```py
classmethod identity_key(class_: Type[Any] | None = None, ident: Any | Tuple[Any, ...] = None, *, instance: Any | None = None, row: Row[Any] | RowMapping | None = None, identity_token: Any | None = None) → _IdentityKeyType[Any]
```

*继承自* `sqlalchemy.orm.session._SessionClassMethods.identity_key` *方法的* `sqlalchemy.orm.session._SessionClassMethods`

返回一个标识键。

这是`identity_key()`的别名。

```py
classmethod object_session(instance: object) → Session | None
```

*继承自* `sqlalchemy.orm.session._SessionClassMethods.object_session` *方法的* `sqlalchemy.orm.session._SessionClassMethods`

返回一个对象所属的`Session`。

这是`object_session()`的别名。

```py
class sqlalchemy.orm.ORMExecuteState
```

表示对`Session.execute()`方法的调用，作为传递给`SessionEvents.do_orm_execute()`事件钩子的参数。

版本 1.4 中的新功能。

另请参阅

执行事件 - 如何使用`SessionEvents.do_orm_execute()`的顶级文档。

**成员**

__init__(), all_mappers, bind_arguments, bind_mapper, execution_options, invoke_statement(), is_column_load, is_delete, is_executemany, is_from_statement, is_insert, is_orm_statement, is_relationship_load, is_select, is_update, lazy_loaded_from, load_options, loader_strategy_path, local_execution_options, parameters, session, statement, update_delete_options, update_execution_options(), user_defined_options

**类签名**

类`sqlalchemy.orm.ORMExecuteState` (`sqlalchemy.util.langhelpers.MemoizedSlots`)

```py
method __init__(session: Session, statement: Executable, parameters: _CoreAnyExecuteParams | None, execution_options: _ExecuteOptions, bind_arguments: _BindArguments, compile_state_cls: Type[ORMCompileState] | None, events_todo: List[_InstanceLevelDispatch[Session]])
```

构造一个新的`ORMExecuteState`。

此对象是在内部构造的。

```py
attribute all_mappers
```

返回此语句顶层涉及的所有`Mapper`对象的序列。

“顶级”指的是那些在`select()`查询的结果集行中表示的`Mapper`对象，或者在`update()`或`delete()`查询中，是 UPDATE 或 DELETE 的主体。

版本 1.4.0b2 中的新内容。

另见

`ORMExecuteState.bind_mapper`

```py
attribute bind_arguments: _BindArguments
```

字典作为 `Session.execute.bind_arguments` 字典传递。

此字典可由扩展用于 `Session` 以传递将有助于确定一组数据库连接中的哪一个应该用于调用此语句的参数。

```py
attribute bind_mapper
```

返回是主“绑定”映射器的 `Mapper`。

对于调用 ORM 语句的 `ORMExecuteState` 对象，即 `ORMExecuteState.is_orm_statement` 属性为 `True` 的情况，此属性将返回被视为语句的“主”映射器。术语“绑定映射器”指的是 `Session` 对象可能“绑定”到多个映射类键入的多个 `Engine` 对象，并且“绑定映射器”确定将选择哪个 `Engine` 对象。

对于针对单个映射类调用的语句，`ORMExecuteState.bind_mapper` 旨在是获取此映射器的可靠方法。

版本 1.4.0b2 中的新功能。

另请参阅

`ORMExecuteState.all_mappers`

```py
attribute execution_options: _ExecuteOptions
```

当前执行选项的完整字典。

这是语句级选项与本地传递的执行选项的合并。

另请参阅

`ORMExecuteState.local_execution_options`

`Executable.execution_options()`

ORM 执行选项

```py
method invoke_statement(statement: Executable | None = None, params: _CoreAnyExecuteParams | None = None, execution_options: OrmExecuteOptionsParameter | None = None, bind_arguments: _BindArguments | None = None) → Result[Any]
```

执行由此 `ORMExecuteState` 表示的语句，而不重新调用已经进行过的事件。

此方法本质上执行当前语句的可重入执行，即当前调用 `SessionEvents.do_orm_execute()` 事件。这样做的用例是为了事件处理程序想要重写如何返回最终 `Result` 对象，比如从离线缓存检索结果或者将结果从多次执行中连接起来的方案。

当实际处理程序函数在 `SessionEvents.do_orm_execute()` 中返回 `Result` 对象，并且传播到调用 `Session.execute()` 方法的地方时，`Session.execute()` 方法的其余部分将被抢占，并且 `Result` 对象将立即返回给 `Session.execute()` 的调用者。

参数：

+   `statement` – 可选的语句，用于代替当前由 `ORMExecuteState.statement` 表示的语句。

+   `params` –

    可选的参数字典或参数列表将合并到此 `ORMExecuteState` 的现有 `ORMExecuteState.parameters` 中。

    2.0 版更改：接受参数字典列表进行 executemany 执行。

+   `execution_options` – 可选的执行选项字典将合并到此 `ORMExecuteState` 的现有 `ORMExecuteState.execution_options` 中。

+   `bind_arguments` – 可选的 bind_arguments 字典将在此 `ORMExecuteState` 的当前 `ORMExecuteState.bind_arguments` 中合并。

返回：

一个带有 ORM 级结果的 `Result` 对象。

另见

重新执行语句 - 关于 `ORMExecuteState.invoke_statement()` 的适当用法的背景和示例。

```py
attribute is_column_load
```

如果操作是刷新现有 ORM 对象上的基于列的属性，则返回 True。

在诸如 `Session.refresh()` 的操作期间发生，以及当由 `defer()` 推迟的属性正在加载时，或者由 `Session.expire()` 直接或通过提交操作而过期的属性正在加载时。

处理程序在进行此类操作时很可能不希望向查询添加任何选项，因为查询应该是直接的主键获取，不应该有任何额外的 WHERE 条件，并且实例旅行的加载器选项已经添加到查询中。

版本 1.4.0b2 中的新功能。

另请参阅

`ORMExecuteState.is_relationship_load`

```py
attribute is_delete
```

如果这是一个 DELETE 操作，则返回 True。

2.0.30 版本中的更改：- 该属性对 `Select.from_statement()` 构造也是真的，该构造本身针对 `Delete` 构造，例如 `select(Entity).from_statement(delete(..))`

```py
attribute is_executemany
```

如果参数是一个包含多个字典且字典数量大于一个的列表，则返回 True。

版本 2.0 中的新功能。

```py
attribute is_from_statement
```

如果此操作是 `Select.from_statement()` 操作，则返回 True。

这与 `ORMExecuteState.is_select` 独立，因为 `select().from_statement()` 构造也可以与 INSERT/UPDATE/DELETE RETURNING 类型的语句一起使用。`ORMExecuteState.is_select` 仅在 `Select.from_statement()` 本身针对 `Select` 构造时设置。

版本 2.0.30 中的新功能。

```py
attribute is_insert
```

如果这是一个 INSERT 操作，则返回 True。

在 2.0.30 版本中更改： - 该属性对`Select.from_statement()`构造也为 True，该构造本身针对`Insert`构造，例如`select(Entity).from_statement(insert(..))`

```py
attribute is_orm_statement
```

如果操作是 ORM 语句，则返回 True。

这表示所调用的 select()、insert()、update()或 delete()包含 ORM 实体作为主体。对于没有 ORM 实体，而只引用`Table`元数据的语句，它被调用为核心 SQL 语句，并且不发生 ORM 级别的自动化。

```py
attribute is_relationship_load
```

如果此加载正在代表关系加载对象，则返回 True。

这意味着，加载程序实际上是一个 LazyLoader、SelectInLoader、SubqueryLoader 或类似的加载程序，并且整个发出的 SELECT 语句都是代表关系加载的。

处理程序很可能不希望在发生此类操作时向查询添加任何选项，因为加载程序选项已经能够传播到关系加载程序，并且应已存在。

另请参阅

`ORMExecuteState.is_column_load`

```py
attribute is_select
```

如果这是一个 SELECT 操作，则返回 True。

在 2.0.30 版本中更改： - 该属性对`Select.from_statement()`构造也为 True，该构造本身针对`Select`构造，例如`select(Entity).from_statement(select(..))`

```py
attribute is_update
```

如果这是一个 UPDATE 操作，则返回 True。

在 2.0.30 版本中更改： - 该属性对`Select.from_statement()`构造也为 True，该构造本身针对`Update`构造，例如`select(Entity).from_statement(update(..))`

```py
attribute lazy_loaded_from
```

正在使用此语句执行进行延迟加载操作的`InstanceState`。

此属性的主要理由是支持水平分片扩展，在此扩展创建的特定查询执行时间钩子中可用。为此，该属性仅打算在**查询执行时间**具有意义，而且重要的是不是在此之前的任何时间，包括查询编译时间。

```py
attribute load_options
```

返回将用于此执行的`load_options`。

```py
attribute loader_strategy_path
```

返回当前加载路径的`PathRegistry`。

此对象表示查询中关系的“路径”，当加载特定对象或集合时。

```py
attribute local_execution_options: _ExecuteOptions
```

`Session.execute()` 方法传递的执行选项的字典视图。

这不包括与被调用语句相关的选项。

另请参阅

`ORMExecuteState.execution_options`

```py
attribute parameters: _CoreAnyExecuteParams | None
```

传递给 `Session.execute()` 方法的参数字典。

```py
attribute session: Session
```

正在使用的 `Session`。

```py
attribute statement: Executable
```

被调用的 SQL 语句。

对于像从 `Query` 检索到的 ORM 选择，这是从 ORM 查询生成的 `select` 的一个实例。

```py
attribute update_delete_options
```

返回将用于此执行的 update_delete_options。

```py
method update_execution_options(**opts: Any) → None
```

使用新值更新本地执行选项。

```py
attribute user_defined_options
```

与被调用语句关联的 `UserDefinedOptions` 序列。

```py
class sqlalchemy.orm.Session
```

管理 ORM 映射对象的持久化操作。

`Session` **不适用于并发线程**。有关详情，请参阅会话是否线程安全？ AsyncSession 是否安全可在并发任务中共享？。

关于 Session 的使用范例请参见使用 Session。

**成员**

__init__(), add(), add_all(), begin(), begin_nested(), bind_mapper(), bind_table(), bulk_insert_mappings(), bulk_save_objects(), bulk_update_mappings(), close(), close_all(), commit(), connection(), delete(), deleted, dirty, enable_relationship_loading(), execute(), expire(), expire_all(), expunge(), expunge_all(), flush(), get(), get_bind(), get_nested_transaction(), get_one(), get_transaction(), identity_key(), identity_map, in_nested_transaction(), in_transaction(), info, invalidate(), is_active, is_modified(), merge(), new, no_autoflush, object_session(), prepare(), query(), refresh(), reset(), rollback(), scalar(), scalars()

**类签名**

类 `sqlalchemy.orm.Session` (`sqlalchemy.orm.session._SessionClassMethods`, `sqlalchemy.event.registry.EventTarget`)

```py
method __init__(bind: _SessionBind | None = None, *, autoflush: bool = True, future: Literal[True] = True, expire_on_commit: bool = True, autobegin: bool = True, twophase: bool = False, binds: Dict[_SessionBindKey, _SessionBind] | None = None, enable_baked_queries: bool = True, info: _InfoType | None = None, query_cls: Type[Query[Any]] | None = None, autocommit: Literal[False] = False, join_transaction_mode: JoinTransactionMode = 'conditional_savepoint', close_resets_only: bool | _NoArg = _NoArg.NO_ARG)
```

构建一个新的 `Session`。

还请参阅 `sessionmaker` 函数，该函数用于生成具有给定参数集的产生 `Session` 的可调用对象。

参数：

+   `autoflush` –

    当为`True`时，所有查询操作将在继续之前对此`Session`发出一个`Session.flush()`调用。这是一个方便的功能，使得不需要重复调用`Session.flush()`以便数据库查询检索结果。

    另请参阅

    刷新 - 自动刷新的额外背景

+   `autobegin` –

    当请求由操作请求数据库访问时，自动启动事务（即相当于调用`Session.begin()`）。默认为`True`。将其设置为`False`以防止`Session`在构造后隐式开始事务，以及在调用任何`Session.rollback()`、`Session.commit()`或`Session.close()`方法后隐式开始事务。

    2.0 版中的新功能。

    另请参阅

    禁用自动启动以防止隐式事务

+   `bind` – 此`Session`应绑定到的可选`Engine`或`Connection`。当指定时，此会话执行的所有 SQL 操作都将通过此连接执行。

+   `binds` –

    一个字典，可以指定任意数量的`Engine`或`Connection`对象作为每个实体连接的源。字典的键由任何一系列映射类、任意的用作映射类基础的 Python 类、`Table`对象和`Mapper`对象组成。然后字典的值是`Engine`或较少常见的`Connection`对象的实例。针对特定映射类进行的操作将查询此字典，以确定用于特定 SQL 操作的最接近匹配实体为何。解析的完整启发式方法在`Session.get_bind()`中描述。用法如下：

    ```py
    Session = sessionmaker(binds={
        SomeMappedClass: create_engine('postgresql+psycopg2://engine1'),
        SomeDeclarativeBase: create_engine('postgresql+psycopg2://engine2'),
        some_mapper: create_engine('postgresql+psycopg2://engine3'),
        some_table: create_engine('postgresql+psycopg2://engine4'),
        })
    ```

    另请参阅

    分区策略（例如每个会话的多个数据库后端）

    `Session.bind_mapper()`

    `Session.bind_table()`

    `Session.get_bind()`

+   `class_` – 指定除了 `sqlalchemy.orm.session.Session` 之外的另一个类，该类应该由返回的类使用。 这是唯一一个本地于 `sessionmaker` 函数的参数，并且不直接发送到 `Session` 的构造函数。

+   `enable_baked_queries` –

    经典； 默认为 `True`。 由 `sqlalchemy.ext.baked` 扩展消耗的一个参数，用于确定是否应该缓存“烘焙查询”，正如该扩展的正常操作一样。 当设置为 `False` 时，该特定扩展所使用的缓存被禁用。

    从版本 1.4 开始更改： `sqlalchemy.ext.baked` 扩展是遗留的，不被 SQLAlchemy 的任何内部使用。 因此，该标志仅影响在其自己的代码中明确使用此扩展的应用程序。

+   `expire_on_commit` –

    默认为 `True`。 当为 `True` 时，每次 `commit()` 后所有实例都将完全过期，以便在完成事务后的所有属性/对象访问从最新的数据库状态加载。

    > 另请参阅
    > 
    > 提交

+   `future` –

    已弃用； 此标志始终为 True。

    另请参阅

    SQLAlchemy 2.0 - 主要迁移指南

+   `info` – 可选的与此 `Session` 关联的任意数据的字典。 可通过 `Session.info` 属性访问。 请注意，该字典在构造时被复制，因此对每个 `Session` 字典的修改将局限于该 `Session`。

+   `query_cls` – 应该用于创建新的查询对象的类，由 `Session.query()` 方法返回。 默认为 `Query`。

+   `twophase` – 当为`True`时，所有事务都将作为“两阶段”事务启动，即使用正在使用的数据库的“两阶段”语义以及 XID。在每个附加数据库上发出了所有附加数据库的`flush()`之后，在`commit()`期间，将调用每个数据库的`TwoPhaseTransaction`的`TwoPhaseTransaction.prepare()`方法。这允许每个数据库在提交每个事务之前回滚整个事务。

+   `autocommit` – `autocommit`关键字出现是为了向后兼容，但必须保持其默认值为`False`。

+   `join_transaction_mode` –

    描述了在给定绑定是已经在此`Session`范围之外开始事务的`Connection`时要采取的事务行为；换句话说，`Connection.in_transaction()`方法返回 True。

    以下行为仅在`Session`**实际使用给定的连接**时才生效；也就是说，诸如`Session.execute()`、`Session.connection()`等方法实际上被调用：

    +   `"conditional_savepoint"` - 这是默认值。如果给定的`Connection`在事务中开始但没有保存点，则使用`"rollback_only"`。如果`Connection`此外还在保存点中，换句话说，`Connection.in_nested_transaction()`方法返回 True，则使用`"create_savepoint"`。

        `"conditional_savepoint"` 行为试图利用 SAVEPOINT 以保持现有事务的状态不变，但只有在已经存在 SAVEPOINT 的情况下才会这样做；否则，不假定正在使用的后端具有足够的 SAVEPOINT 支持，因为此功能的可用性会有所变化。 `"conditional_savepoint"` 还试图与之前的 `Session` 行为建立大致的向后兼容性，适用于未设置特定模式的应用程序。建议使用其中一种显式设置。

    +   `"create_savepoint"` - `Session` 将在所有情况下使用 `Connection.begin_nested()` 来创建自己的事务。该事务本质上“在顶部”上使用给定 `Connection` 上已打开的任何现有事务；如果底层数据库和正在使用的驱动程序完全且不破损地支持 SAVEPOINT，则外部事务将在 `Session` 的生命周期内保持不受影响。

        `"create_savepoint"` 模式对于将 `Session` 集成到测试套件中并使外部启动的事务保持不受影响是最有用的；但是，它依赖于底层驱动程序和数据库对 SAVEPOINT 的正确支持。

        提示

        当使用 SQLite 时，Python 3.11 中包含的 SQLite 驱动程序在某些情况下未正确处理 SAVEPOINTs，需要使用解决方法。详见 Serializable isolation / Savepoints / Transactional DDL 和 Serializable isolation / Savepoints / Transactional DDL (asyncio version) 部分。

    +   `"control_fully"` - `Session` 将控制给定事务作为自己的事务； `Session.commit()` 将在事务上调用`.commit()`， `Session.rollback()` 将在事务上调用`.rollback()`， `Session.close()` 将调用事务上的`.rollback`。

        提示

        此使用模式等同于 SQLAlchemy 1.4 处理具有现有 SAVEPOINT 的`Connection`的方式（即`Connection.begin_nested()`）; `Session`将完全控制现有的 SAVEPOINT。

    +   `"rollback_only"` - `Session`仅会接管给定事务的`.rollback()`调用；`.commit()`调用不会传播到给定事务。`.close()`调用不会对给定事务产生影响。

        提示

        此使用模式等同于 SQLAlchemy 1.4 处理具有现有常规数据库事务的`Connection`的方式（即`Connection.begin()`）；`Session`将传播`Session.rollback()`调用到底层事务，但不会传播`Session.commit()`或`Session.close()`调用。

    自 2.0.0rc1 版本新增。

+   `close_resets_only` -

    默认为`True`。确定会话在调用`.close()`后是否应该重置自身，还是应该处于不再可用的状态，禁用重用。

    自 2.0.22 版本新增：添加标志`close_resets_only`。未来的 SQLAlchemy 版本可能会将此标志的默认值更改为`False`。

    另请参阅

    关闭 - 关于`Session.close()`和`Session.reset()`语义的详细信息。

```py
method add(instance: object, _warn: bool = True) → None
```

将一个对象放入此`Session`中。

传递给`Session.add()`方法时处于瞬时状态的对象将移动到挂起状态，直到下一次刷新，然后它们将转移到持久化状态。

传递给`Session.add()`方法时处于分离状态的对象将直接转移到持久化状态。

如果由`Session`使用的事务被回滚，那些在传递给`Session.add()`时处于瞬态的对象将会被移回到瞬态状态，并且不再存在于这个`Session`中。

另请参见

`Session.add_all()`

添加新项目或现有项目 - 在使用会话的基础知识

```py
method add_all(instances: Iterable[object]) → None
```

将给定的实例集合添加到此`Session`中。

请参阅`Session.add()`的文档以获取一般行为描述。

另请参见

`Session.add()`

添加新项目或现有项目 - 在使用会话的基础知识

```py
method begin(nested: bool = False) → SessionTransaction
```

在此`Session`上开始事务或嵌套事务（如果尚未开始）。

`Session`对象具有**autobegin**行为，因此通常不需要显式调用`Session.begin()`方法。但是，它可以用来控制何时开始事务状态的范围。

当用于开始最外层事务时，如果此`Session`已经处于事务中，则会引发错误。

参数：

**nested** – 如果为 True，则开始一个 SAVEPOINT 事务，并等效于调用`Session.begin_nested()`。有关 SAVEPOINT 事务的文档，请参阅使用 SAVEPOINT。

返回：

`SessionTransaction`对象。注意`SessionTransaction`充当 Python 上下文管理器，允许在“with”块中使用`Session.begin()`。请参阅显式开始以获取示例。

另请参见

自动开始

管理事务

`Session.begin_nested()`

```py
method begin_nested() → SessionTransaction
```

在此`Session`上开始“嵌套”事务，例如 SAVEPOINT。

目标数据库和相关驱动程序必须支持 SQL SAVEPOINT，以使此方法正常工作。

有关 SAVEPOINT 事务的文档，请参阅 使用 SAVEPOINT。

返回：

`SessionTransaction` 对象。请注意，`SessionTransaction` 作为上下文管理器，允许在“with”块中使用 `Session.begin_nested()`。请参阅 使用 SAVEPOINT 获取使用示例。

另请参阅

使用 SAVEPOINT

可串行化隔离 / Savepoints / 事务性 DDL - 使用 SQLite 驱动程序时，为使 SAVEPOINT 正常工作需要特殊的解决方法。对于 asyncio 案例，请参阅 可串行化隔离 / Savepoints / 事务性 DDL（asyncio 版本） 部分。

```py
method bind_mapper(mapper: _EntityBindKey[_O], bind: _SessionBind) → None
```

将 `Mapper` 或任意 Python 类与“bind”关联，例如一个 `Engine` 或 `Connection`。

给定的实体被添加到 `Session.get_bind()` 方法使用的查找中。

参数：

+   `mapper` – 一个 `Mapper` 对象，或者一个映射类的实例，或者任何作为一组映射类基础的 Python 类。

+   `bind` – 一个 `Engine` 或 `Connection` 对象。

另请参阅

分区策略（例如，每个 Session 多个数据库后端）

`Session.binds`

`Session.bind_table()`

```py
method bind_table(table: TableClause, bind: Engine | Connection) → None
```

将 `Table` 与“bind”关联，例如一个 `Engine` 或 `Connection`。

给定的 `Table` 被添加到 `Session.get_bind()` 方法使用的查找中。

参数：

+   `table` – 一个 `Table` 对象，通常是 ORM 映射的目标，或者存在于可选择的映射中。

+   `bind` - 一个`Engine`或`Connection`对象。

另请参阅

分区策略（例如每个 Session 的多个数据库后端）

`Session.binds`

`Session.bind_mapper()`

```py
method bulk_insert_mappings(mapper: Mapper[Any], mappings: Iterable[Dict[str, Any]], return_defaults: bool = False, render_nulls: bool = False) → None
```

执行给定映射字典列表的批量插入。

传统特性

该方法是 SQLAlchemy 2.0 系列的一个传统特性。对于现代的批量插入和更新，请参见 ORM 批量插入语句和 ORM 按主键批量更新部分。2.0 API 共享了与该方法的实现细节，并添加了新的特性。

参数：

+   `mapper` - 一个映射类，或者实际的`Mapper`对象，表示映射列表中所代表的单一对象种类。

+   `mappings` - 一系列字典，每个字典包含要插入的映射行的状态，以映射类上的属性名称表示。如果映射涉及多个表，例如连接继承映射，则每个字典必须包含要填充到所有表中的所有键。

+   `return_defaults` -

    当为 True 时，INSERT 过程将被修改以确保新生成的主键值将被获取。该参数的理由通常是为了使连接表继承映射能够批量插入。

    注意

    对于不支持 RETURNING 的后端，`Session.bulk_insert_mappings.return_defaults`参数可能会显著降低性能，因为 INSERT 语句无法再批量处理。有关受影响的后端的背景信息，请参阅 INSERT 语句的“插入多个值”行为。

+   `render_nulls` -

    当为 True 时，`None`值将导致在 INSERT 语句中包含一个 NULL 值，而不是将列从 INSERT 中省略。这允许所有被 INSERT 的行具有相同的列集，从而允许将完整的行集批量发送到 DBAPI。通常，每个包含与上一行不同的 NULL 值组合的列集必须从渲染的 INSERT 语句中省略不同的列系列，这意味着必须作为单独的语句发出。通过传递此标志，可以确保将完整的行集批量处理为一个批次；但成本是，通过省略列调用的服务器端默认值将被跳过，因此必须注意确保这些不是必需的。

    警告

    当设置此标志时，**不会调用服务器端默认的 SQL 值**，对于那些作为 NULL 插入的列；NULL 值将被显式发送。必须注意确保整个操作不需要调用服务器端默认函数。

另请参阅

启用 ORM 的 INSERT、UPDATE 和 DELETE 语句

`Session.bulk_save_objects()`

`Session.bulk_update_mappings()`

```py
method bulk_save_objects(objects: Iterable[object], return_defaults: bool = False, update_changed_only: bool = True, preserve_order: bool = True) → None
```

对给定对象列表执行批量保存。

传统功能

该方法是 SQLAlchemy 2.0 系列的传统功能。对于现代的批量 INSERT 和 UPDATE，请参见 ORM 批量 INSERT 语句和 ORM 按主键批量 UPDATE 部分。

对于一般的 INSERT 和 UPDATE 现有 ORM 映射对象，建议使用标准的工作单元数据管理模式，介绍在 SQLAlchemy 统一教程中的 ORM 数据操作。SQLAlchemy 2.0 现在使用现代方言的“Insert Many Values”行为用于 INSERT 语句，解决了以前批量 INSERT 速度慢的问题。

参数：

+   `objects` –

    映射对象实例的序列。映射对象按原样持久化，并且之后**不**与`Session`关联。

    对于每个对象，无论对象是作为 INSERT 还是 UPDATE 发送的，都取决于传统操作中`Session`使用的相同规则；如果对象具有`InstanceState.key`属性设置，则假定对象为“分离”，将导致 UPDATE。否则，将使用 INSERT。

    在 UPDATE 的情况下，语句根据哪些属性已更改而分组，并因此成为每个 SET 子句的主题。如果`update_changed_only`为 False，则每个对象中存在的所有属性都将应用于 UPDATE 语句，这有助于将语句组合成更大的 executemany()，并且还将减少检查属性历史的开销。

+   `return_defaults` - 当为 True 时，缺少生成默认值的值的行，即整数主键默认值和序列，将逐个插入，以便主键值可用。特别是，这将允许加入继承和其他多表映射正确插入，而无需提前提供主键值；但是，`Session.bulk_save_objects.return_defaults`会大大降低该方法的性能增益。强烈建议使用标准的`Session.add_all()`方法。

+   `update_changed_only` - 当为 True 时，根据每个状态中已记录更改的属性来渲染 UPDATE 语句。当为 False 时，所有存在的属性都将渲染到 SET 子句中，除了主键属性。

+   `preserve_order` - 当为 True 时，插入和更新的顺序与给定对象的顺序完全匹配。当为 False 时，常见类型的对象被分组为插入和更新，以便提供更多的批处理机会。

另请参阅

ORM 启用的 INSERT、UPDATE 和 DELETE 语句

`Session.bulk_insert_mappings()`

`Session.bulk_update_mappings()`

```py
method bulk_update_mappings(mapper: Mapper[Any], mappings: Iterable[Dict[str, Any]]) → None
```

对给定的映射字典列表执行批量更新。

传统特性

该方法是 SQLAlchemy 2.0 系列的一个传统特性。对于现代的批量 INSERT 和 UPDATE，请参见 ORM 批量 INSERT 语句和 ORM 按主键批量 UPDATE 部分。2.0 API 与此方法共享实现细节，并添加了新功能。

参数：

+   `mapper` - 一个映射类，或者实际的`Mapper`对象，表示映射列表中表示的单一对象类型。

+   `mappings` - 一个字典序列，每个字典包含要更新的映射行的状态，以映射类上的属性名称表示。如果映射涉及多个表，比如联接继承映射，每个字典可能包含与所有表对应的键。所有存在且不是主键的键将应用于 UPDATE 语句的 SET 子句；需要的主键值将应用于 WHERE 子句。

另请参见

ORM-Enabled INSERT、UPDATE 和 DELETE 语句

`Session.bulk_insert_mappings()`

`Session.bulk_save_objects()`

```py
method close() → None
```

结束由此 `Session` 使用的事务资源和 ORM 对象。

这会清除与此 `Session` 关联的所有 ORM 对象，结束任何正在进行的事务，并释放此 `Session` 从相关 `Engine` 对象中检出的任何 `Connection` 对象。然后，该操作将 `Session` 置于可以再次使用的状态。

提示

在默认运行模式下，`Session.close()` 方法**不会阻止该 Session 再次使用**。 `Session` 本身实际上并没有一个独立的“关闭”状态；它只是意味着 `Session` 将释放所有数据库连接和 ORM 对象。

将参数 `Session.close_resets_only` 设置为 `False` 将使 `close` 最终化，这意味着会禁止对会话的任何进一步操作。

从 1.4 版本开始变更：`Session.close()` 方法不会立即创建一个新的 `SessionTransaction` 对象；相反，只有在再次使用 `Session` 进行数据库操作时才会创建新的 `SessionTransaction`。

另请参见

关闭 - 关于`Session.close()`和`Session.reset()`语义的详细信息。

`Session.reset()` - 一种类似的方法，其行为类似于`close()`，参数为`Session.close_resets_only`设置为`True`。

```py
classmethod close_all() → None
```

*继承自* `sqlalchemy.orm.session._SessionClassMethods.close_all` *方法的* `sqlalchemy.orm.session._SessionClassMethods`

关闭*所有*内存中的会话。

自版本 1.3 起已弃用：`Session.close_all()`方法已弃用，将在将来的版本中删除。请参考`close_all_sessions()`。

```py
method commit() → None
```

刷新待处理的更改并提交当前事务。

当 COMMIT 操作完成时，所有对象都完全过期，擦除其内部内容，当下次访问这些对象时，将自动重新加载。在此期间，这些对象处于过期状态，并且如果从`Session`中分离出来，则不会起作用。此外，当使用基于 asyncio 的 API 时，不支持此重新加载操作。`Session.expire_on_commit`参数可用于禁用此行为。

当`Session`中没有事务时，表示自上次调用`Session.commit()`以来没有对此`Session`执行任何操作时，该方法将开始并提交一个仅内部的“逻辑”事务，通常不会影响数据库，除非检测到有待提交的刷新更改，但仍然会调用事件处理程序和对象过期规则。

最外层的数据库事务会无条件提交，自动释放任何生效的 SAVEPOINT。

另请参阅

提交

管理事务

在使用 AsyncSession 时避免隐式 IO

```py
method connection(bind_arguments: _BindArguments | None = None, execution_options: CoreExecuteOptionsParameter | None = None) → Connection
```

返回一个与该`Session`对象的事务状态对应的`Connection`对象。

返回当前事务对应的 `Connection`，或者如果没有进行中的事务，则开始新的事务并返回 `Connection`（请注意，在第一个 SQL 语句被发出之前，与 DBAPI 之间不会建立事务状态）。

多重绑定或未绑定的 `Session` 对象中的歧义可以通过任何可选关键字参数解决。最终将使用 `get_bind()` 方法进行解决。

参数：

+   `bind_arguments` – 绑定参数字典。可能包括“mapper”，“bind”，“clause”等其他传递给 `Session.get_bind()` 的自定义参数。

+   `execution_options` –

    一个执行选项字典，当首次获得连接时将传递给 `Connection.execution_options()`。如果连接已经存在于 `Session` 中，将发出警告并忽略参数。

    另请参阅

    设置事务隔离级别 / DBAPI AUTOCOMMIT

```py
method delete(instance: object) → None
```

将实例标记为已删除。

假设传入的对象在调用该方法后处于 持久化 或 分离 状态；在此方法被调用后，对象将保持 持久化 状态，直到下一次刷新操作进行。在此期间，该对象也将是 `Session.deleted` 集合的成员。

当下一次刷新操作进行时，对象将移动到 已删除 状态，表示在当前事务中为其行发出了 `DELETE` 语句。当事务成功提交时，已删除对象将移动到 分离 状态，并且不再存在于此 `Session` 中。

另请参阅

删除 - 在 使用会话的基础知识

```py
attribute deleted
```

所有在此 `Session` 中标记为 ‘deleted’ 的实例集合。

```py
attribute dirty
```

所有被认为是脏的持久化实例集合。

例如：

```py
some_mapped_object in session.dirty
```

当实例被修改但未删除时被视为脏。

请注意，此“脏”计算是“乐观”的；大多数属性设置或集合修改操作都将将实例标记为“脏”，并将其放入此集合中，即使属性的值没有净变化。在刷新时，将每个属性的值与其先前保存的值进行比较，如果没有净变化，则不会执行任何 SQL 操作（这是一项更昂贵的操作，因此仅在刷新时执行）。

要检查实例是否有可操作的净变化来修改其属性，请使用`Session.is_modified()`方法。

```py
method enable_relationship_loading(obj: object) → None
```

将对象与此`Session`关联以进行相关对象加载。

警告

`enable_relationship_loading()`存在以服务于特殊用例，并不建议用于一般用途。

使用`relationship()`映射的属性的访问将尝试使用此`Session`作为连接源从数据库加载值。值将根据此对象上存在的外键和主键值加载 - 如果不存在，则这些关系将不可用。

对象将附加到此会话，但**不会**参与任何持久性操作；它的状态对于几乎所有目的都将保持“瞬态”或“分离”，除了关系加载的情况。

还要注意，反向引用通常不会按预期工作。如果目标对象上的关系绑定属性发生更改，则可能不会触发反向引用事件，如果有效值已从保存外键值的值中加载，则不会触发事件。

`Session.enable_relationship_loading()`方法类似于`relationship()`上的`load_on_pending`标志。与该标志不同，`Session.enable_relationship_loading()`允许对象保持瞬态，同时仍然能够加载相关项目。

要使一个临时对象与`Session`相关联，可以通过`Session.enable_relationship_loading()`将其添加到`Session`中。如果该对象代表数据库中现有的标识，则应使用`Session.merge()`进行合并。

当 ORM 正常使用时，`Session.enable_relationship_loading()`不会改善行为 - 对象引用应该在对象级别而不是在外键级别构建，以便它们在 flush()继续之前以普通方式存在。此方法不适用于一般用途。

另请参见

`relationship.load_on_pending` - 此标志允许在待处理项目上对多对一进行每关系加载。

`make_transient_to_detached()` - 允许将对象添加到`Session`中而不发出 SQL，然后在访��时取消过期属性。

```py
method execute(statement: Executable, params: _CoreAnyExecuteParams | None = None, *, execution_options: OrmExecuteOptionsParameter = {}, bind_arguments: _BindArguments | None = None, _parent_execute_state: Any | None = None, _add_event: Any | None = None) → Result[Any]
```

执行一个 SQL 表达式构造。

返回一个代表语句执行结果的`Result`对象。

例如：

```py
from sqlalchemy import select
result = session.execute(
    select(User).where(User.id == 5)
)
```

`Session.execute()`的 API 契约类似于`Connection.execute()`的契约，是`Connection`的 2.0 风格版本。

从版本 1.4 开始更改：当使用 2.0 风格的 ORM 用法时，`Session.execute()`方法现在是 ORM 语句执行的主要点。

参数：

+   `statement` – 一个可执行的语句（即一个`Executable`表达式，如`select()`）。

+   `params` – 可选字典，或包含绑定参数值的字典列表。如果是单个字典，则执行单行; 如果是字典列表，则将调用“executemany”。每个字典中的键必须与语句中存在的参数名称相对应。

+   `execution_options` –

    可选的执行选项字典，将与语句执行相关联。此字典可以提供`Connection.execution_options()`接受的选项的子集，并且还可以提供仅在 ORM 上下文中理解的其他选项。

    另请参阅

    ORM 执行选项 - ORM 特定的执行选项

+   `bind_arguments` – 用于确定绑定的其他参数字典。可以包括“mapper”，“bind”或其他自定义参数。此字典的内容将传递给`Session.get_bind()`方法。

返回：

一个`Result`对象。

```py
method expire(instance: object, attribute_names: Iterable[str] | None = None) → None
```

使实例上的属性过期。

标记实例的属性为过期。下次访问过期属性时，将向`Session`对象的当前事务上下文发出查询，以加载给定实例的所有过期属性。请注意，高度隔离的事务将返回与该事务中先前读取的相同值，而不管该事务外的数据库状态发生了什么变化。

要同时使`Session`中的所有对象过期，请使用`Session.expire_all()`。

当调用`Session.rollback()`或`Session.commit()`方法时，`Session`对象的默认行为是使所有状态过期，以便为新事务加载新状态。因此，仅在当前事务中发出非 ORM SQL 语句的特定情况下调用`Session.expire()`才有意义。

参数：

+   `instance` – 要刷新的实例。

+   `attribute_names` – 可选的字符串属性名称列表，指示要过期的属性子集。

另请参阅

刷新/过期 - 入门材料

`Session.expire()`

`Session.refresh()`

`Query.populate_existing()`

```py
method expire_all() → None
```

使此会话中的所有持久实例过期。

下次访问持久实例上的任何属性时，将使用`Session`对象的当前事务上下文发出查询，以加载给定实例的所有过期属性。请注意，高度隔离的事务将返回与之前在同一事务中读取的相同值，而不考虑该事务之外的数据库状态的更改。

要使个别对象和这些对象上的个别属性过期，请使用`Session.expire()`。

当调用`Session.rollback()`或`Session.commit()`方法时，默认情况下，`Session`对象会将所有状态过期，以便为新的事务加载新的状态。因此，通常不需要调用`Session.expire_all()`，假设事务是隔离的。

另请参阅

刷新 / 过期 - 入门材料

`Session.expire()`

`Session.refresh()`

`Query.populate_existing()`

```py
method expunge(instance: object) → None
```

从这个`Session`中移除实例。

这将释放对该实例的所有内部引用。根据*expunge*级联规则将应用级联。

```py
method expunge_all() → None
```

从这个`Session`中移除所有对象实例。

这相当于在这个`Session`中调用`expunge(obj)`来移除所有对象。

```py
method flush(objects: Sequence[Any] | None = None) → None
```

刷新所有对象更改到数据库。

将所有待定的对象创建、删除和修改写入数据库，作为 INSERT、DELETE、UPDATE 等操作。操作会自动按照会话的工作单元依赖解决器进行排序。

数据库操作将在当前事务上下文中发出，并且不会影响事务的状态，除非发生错误，此时整个事务都会回滚。在事务中随时可以刷新（flush()）以将更改从 Python 移动到数据库的事务缓冲区。

参数：

**对象** –

可选；限制刷新操作仅对给定集合中存在的元素进行操作。

这个功能适用于极少数情况，特定对象可能需要在完全刷新（flush()）之前进行操作。它不适用于一般用途。

```py
method get(entity: _EntityBindKey[_O], ident: _PKIdentityArgument, *, options: Sequence[ORMOption] | None = None, populate_existing: bool = False, with_for_update: ForUpdateParameter = None, identity_token: Any | None = None, execution_options: OrmExecuteOptionsParameter = {}, bind_arguments: _BindArguments | None = None) → _O | None
```

返回基于给定主键标识符的实例，如果找不到则返回`None`。

例如：

```py
my_user = session.get(User, 5)

some_object = session.get(VersionedFoo, (5, 10))

some_object = session.get(
    VersionedFoo,
    {"id": 5, "version_id": 10}
)
```

从版本 1.4 开始：添加了`Session.get()`，该方法已从现在过时的`Query.get()`方法中移动。

`Session.get()`是特殊的，因为它直接提供了对`Session`的标识映射的访问。如果给定的主键标识符存在于本地标识映射中，则直接从该集合返回对象，而不会发出任何 SQL，除非对象已被标记为完全过期。如果不存在，则执行 SELECT 以定位对象。

`Session.get()`还将执行检查，如果对象存在于标识映射中并标记为过期，则发出 SELECT 以刷新对象以及确保行仍然存在。如果不存在，则引发`ObjectDeletedError`。

参数：

+   `entity` – 表示要加载的实体类型的映射类或`Mapper`。

+   `ident` –

    表示主键的标量、元组或字典。对于复合（例如，多列）主键，应传递元组或字典。

    对于单列主键，标量调用形式通常是最方便的。如果行的主键是值“5”，调用如下所示：

    ```py
    my_object = session.get(SomeClass, 5)
    ```

    元组形式包含主键值，通常按照它们与映射的`Table`对象的主键列对应的顺序排列，或者如果使用了`Mapper.primary_key`配置参数，则按照该参数的顺序排列。例如，如果行的主键由整数数字“5，10”表示，则调用如下所示：

    ```py
    my_object = session.get(SomeClass, (5, 10))
    ```

    字典形式应包含作为主键每个元素相应的映射属性名称的键。如果映射类具有存储对象主键值的属性`id`、`version_id`，则调用如下所示：

    ```py
    my_object = session.get(SomeClass, {"id": 5, "version_id": 10})
    ```

+   `options` – 可选的加载器选项序列，将应用于查询（如果有的话）。

+   `populate_existing` – 导致该方法无条件地发出 SQL 查询，并使用新加载的数据刷新对象，无论对象是否已存在。

+   `with_for_update` – 可选的布尔值`True`，表示应使用 FOR UPDATE，或者可以是一个包含用于指示 SELECT 的更具体的一组 FOR UPDATE 标志的字典；标志应与`Query.with_for_update()`参数的参数相匹配。取代`Session.refresh.lockmode`参数。

+   `execution_options` –

    可选的执行选项字典，如果发出了查询执行，则将其与查询执行相关联。该字典可以提供`Connection.execution_options()`接受的选项的子集，并且还可以提供只有在 ORM 上下文中理解的其他选项。

    1.4.29 版中新增。

    另请参阅

    ORM 执行选项 - ORM 特定的执行选项

+   `bind_arguments` –

    用于确定绑定的其他参数的字典。可能包括“mapper”、“bind”或其他自定义参数。该字典的内容将传递给`Session.get_bind()`方法。

返回：

对象实例，或`None`。

```py
method get_bind(mapper: _EntityBindKey[_O] | None = None, *, clause: ClauseElement | None = None, bind: _SessionBind | None = None, _sa_skip_events: bool | None = None, _sa_skip_for_implicit_returning: bool = False, **kw: Any) → Engine | Connection
```

返回此`Session`绑定到的“bind”。

“bind”通常是`Engine`的实例，除非`Session`已被明确地直接绑定到`Connection`的情况。

对于多次绑定或未绑定的`Session`，使用`mapper`或`clause`参数来确定返回的适当绑定。

注意当通过 ORM 操作调用`Session.get_bind()`时通常存在“mapper”参数，例如`Session.query()`中的每个个体 INSERT/UPDATE/DELETE 操作，`Session.flush()`调用等。

解析顺序为：

1.  如果给定了`mapper`并且`Session.binds`存在，则首先基于正在使用的映射器，然后基于正在使用的映射类，然后基于映射类的`__mro__`中存在的任何基类，从更具体的超类到更一般的类来定位一个绑定。

1.  如果给定了条件并且`Session.binds`存在，则基于`Session.binds`中的给定条件中找到的`Table`对象定位绑定。

1.  如果`Session.binds`存在，则返回该绑定。

1.  如果给定了条件，则尝试返回与该条件最终关联的`MetaData`绑定。

1.  如果提供了映射器，尝试返回与最终与该映射器映射的`Table`或其他可选择对象关联的`MetaData`绑定。

1.  如果找不到绑定，则会引发`UnboundExecutionError`。

请注意，`Session.get_bind()`方法可以在`Session`的用户定义子类上被重写，以提供任何类型的绑定解析方案。请参阅自定义垂直分区中的示例。

参数：

+   `mapper` – 可选的映射类或相应的`Mapper`实例。绑定可以首先从此`Session`关联的“绑定”映射中派生，其次是从该`Mapper`映射到的`Table`关联的`MetaData`中派生。

+   `clause` – 一个`ClauseElement`（即`select()`，`text()`等）。如果`mapper`参数不存在或无法生成绑定，则将搜索给定的表达式构造，通常是与绑定的`MetaData`关联的`Table`。

另请参阅

分区策略（例如每个会话的多个数据库后端）

`Session.binds`

`Session.bind_mapper()`

`Session.bind_table()`

```py
method get_nested_transaction() → SessionTransaction | None
```

返回正在进行的当前嵌套事务（如果有）。

新版本 1.4 中新增。

```py
method get_one(entity: _EntityBindKey[_O], ident: _PKIdentityArgument, *, options: Sequence[ORMOption] | None = None, populate_existing: bool = False, with_for_update: ForUpdateParameter = None, identity_token: Any | None = None, execution_options: OrmExecuteOptionsParameter = {}, bind_arguments: _BindArguments | None = None) → _O
```

根据给定的主键标识符返回精确的一个实例，如果未找到则引发异常。

如果查询未选择任何行，则引发`sqlalchemy.orm.exc.NoResultFound`。

有关参数的详细文档，请参阅方法`Session.get()`。

新版本 2.0.22 中新增。

返回：

对象实例。

另请参见

`Session.get()` - 相当的方法，代替

如果未找到具有提供的主键的行，则返回`None`。

```py
method get_transaction() → SessionTransaction | None
```

返回正在进行的当前根事务（如果有）。

新版本 1.4 中新增。

```py
classmethod identity_key(class_: Type[Any] | None = None, ident: Any | Tuple[Any, ...] = None, *, instance: Any | None = None, row: Row[Any] | RowMapping | None = None, identity_token: Any | None = None) → _IdentityKeyType[Any]
```

*继承自* `sqlalchemy.orm.session._SessionClassMethods.identity_key` *的方法* `sqlalchemy.orm.session._SessionClassMethods`

返回一个标识键。

这是 `identity_key()` 的别名。

```py
attribute identity_map: IdentityMap
```

对象标识到对象本身的映射。

通过`Session.identity_map.values()`迭代提供对当前会话中当前持久对象（即具有行标识的对象）的完整集合的访问。

另请参见

`identity_key()` - 辅助函数，用于生成此字典中使用的键。

```py
method in_nested_transaction() → bool
```

如果此 `Session` 已开始嵌套事务（例如，SAVEPOINT），则返回 True。

新版本 1.4 中新增。

```py
method in_transaction() → bool
```

如果此 `Session` 已开始事务，则返回 True。

新版本 1.4 中新增。

另请参见

`Session.is_active`

```py
attribute info
```

一个用户可修改的字典。

此字典的初始值可以使用 `info` 参数来填充到 `Session` 构造函数或 `sessionmaker` 构造函数或工厂方法中。此处的字典始终局限于此 `Session` 并且可以独立于所有其他 `Session` 对象进行修改。

```py
method invalidate() → None
```

关闭此会话，使用连接失效。

这是 `Session.close()` 的变体，还将确保对每个当前用于事务的 `Connection` 对象调用 `Connection.invalidate()` 方法（通常只有一个连接，除非 `Session` 与多个引擎一起使用）。

当已知数据库处于不再安全使用连接的状态时，可以调用此方法。

以下示例说明了在使用 [gevent](https://www.gevent.org/) 时可能出现的可导致底层连接应被丢弃的 `Timeout` 异常情况：

```py
import gevent

try:
    sess = Session()
    sess.add(User())
    sess.commit()
except gevent.Timeout:
    sess.invalidate()
    raise
except:
    sess.rollback()
    raise
```

此方法还会执行 `Session.close()` 所做的所有操作，包括清除所有 ORM 对象。

```py
attribute is_active
```

如果此 `Session` 未处于“部分回滚”状态，则返回 True。

版本 1.4 中的更改：`Session` 不再立即开始新事务，因此当首次实例化 `Session` 时，此属性将为 False。

“部分回滚”状态通常表示 `Session` 的刷新过程失败，并且必须发出 `Session.rollback()` 方法以完全回滚事务。

如果此 `Session` 未处于事务中，则当首次使用时，`Session` 将自动开始，因此在这种情况下，`Session.is_active` 将返回 True。

否则，如果此 `Session` 处于事务中，并且该事务尚未在内部回滚，则 `Session.is_active` 也将返回 True。

另请参阅

“由于刷新期间的先前异常，此会话的事务已回滚。”（或类似内容）

`Session.in_transaction()`

```py
method is_modified(instance: object, include_collections: bool = True) → bool
```

如果给定实例具有本地修改的属性，则返回 `True`。

此方法检索实例上每个受仪器化的属性的历史记录，并将当前值与其先前提交的值进行比较（如果有）。

实际上，这是检查 `Session.dirty` 集合中给定实例的更昂贵和准确的版本；对每个属性的净“脏”状态进行全面测试。

例如：

```py
return session.is_modified(someobject)
```

这种方法有一些注意事项：

+   在 `Session.dirty` 集合中存在的实例在使用此方法进行测试时可能会报告 `False`。这是因为对象可能通过属性突变接收到更改事件，从而将其置于 `Session.dirty` 中，但最终状态与从数据库加载的状态相同，在这里没有净更改。

+   当新值被应用时，标量属性可能没有记录先前设置的值，如果在接收新值时该属性未加载或已过期，则假定该属性有一个更改，即使最终对其数据库值没有净更改也是如此。在这些情况下，即使最终没有针对数据库值的净更改，也假定该属性有一个更改。在大多数情况下，当发生 set 事件时，SQLAlchemy 不需要“旧”值，因此如果旧值不存在，则跳过 SQL 调用的开销，这是基于标量值通常需要 UPDATE 的假设，并且在那几种情况下它不需要的情况下，平均比发出防御性的 SELECT 更便宜。

    仅当属性容器的 `active_history` 标志设置为 `True` 时，才无条件地在 set 时获取“旧”值。这个标志通常设置为主键属性和不是简单多对一的标量对象引用。要为任意映射列设置此标志，请使用 `column_property()` 的 `active_history` 参数。

参数：

+   `instance` – 被测试的映射实例。

+   `include_collections` – 表示是否应包含多值集合在操作中。将其设置为 `False` 是一种检测仅基于本地列的属性（即标量列或多对一外键），这些属性会导致此实例在刷新时进行 UPDATE 的方法。

```py
method merge(instance: _O, *, load: bool = True, options: Sequence[ORMOption] | None = None) → _O
```

将给定实例的状态复制到此 `Session` 中的相应实例。

`Session.merge()`检查源实例的主键属性，并尝试将其与会话中具有相同主键的实例进行协调。如果在本地找不到，则尝试根据主键从数据库加载对象，如果找不到，则创建一个新实例。然后将源实例上的每个属性的状态复制到目标实例。然后该方法返回生成的目标实例；原始源实例保持不变，并且如果尚未与`Session`相关联，则不与之相关联。

如果关联使用`cascade="merge"`进行映射，则此操作会级联到关联的实例。

有关合并的详细讨论，请参见合并。

参数：

+   `instance` – 要合并的实例。

+   `load` –

    布尔值，当为 False 时，`merge()`切换到“高性能”模式，使其放弃发出历史事件以及所有数据库访问。此标志用于诸如将对象图传输到从第二级缓存中的`Session`中，或者将刚加载的对象传输到由工作线程或进程拥有的`Session`中，而无需重新查询数据库的情况。

    `load=False`用例增加了这样一个警告，即给定对象必须处于“干净”状态，也就是说，没有要刷新的挂起更改-即使传入对象已从任何`Session`分离。这样，当合并操作填充本地属性并级联到相关对象和集合时，值可以原样“打印”到目标对象上，而不会生成任何历史记录或属性事件，并且无需将传入数据与可能未加载的任何现有相关对象或集合进行协调。来自`load=False`的结果对象始终生成为“干净”，因此只有给定对象也应该是“干净”的，否则这表明方法的错误使用。

+   `options` –

    在合并操作从数据库加载对象的现有版本时，会将一系列可选的加载器选项应用于`Session.get()`方法。

    版本 1.4.24 中的新功能。

另请参见

`make_transient_to_detached()` - 提供了一种将单个对象“合并”到`Session`中的替代方法

```py
attribute new
```

在此`Session`中标记为“new”的所有实例的集合。

```py
attribute no_autoflush
```

返回一个禁用自动刷新的上下文管理器。

例如：

```py
with session.no_autoflush:

    some_object = SomeClass()
    session.add(some_object)
    # won't autoflush
    some_object.related_thing = session.query(SomeRelated).first()
```

在 `with:` 块内进行的操作不会受到在查询访问时发生的刷新的影响。当初始化一系列涉及现有数据库查询的对象时，尚未完成的对象不应立即被刷新时，这是有用的。

```py
classmethod object_session(instance: object) → Session | None
```

*继承自* `sqlalchemy.orm.session._SessionClassMethods.object_session` *方法的* `sqlalchemy.orm.session._SessionClassMethods`

返回一个对象所属的 `Session`。

这是 `object_session()` 的别名。

```py
method prepare() → None
```

准备当前进行中的事务以进行两阶段提交。

如果没有进行事务，则此方法会引发一个 `InvalidRequestError`。

仅两阶段会话的根事务可以被准备。如果当前事务不是这样的事务，则会引发一个 `InvalidRequestError`。

```py
method query(*entities: _ColumnsClauseArgument[Any], **kwargs: Any) → Query[Any]
```

返回一个与此 `Session` 对应的新 `Query` 对象。

请注意，`Query` 对象自 SQLAlchemy 2.0 起已被视为遗留；现在使用 `select()` 构造 ORM 查询。

另请参阅

SQLAlchemy 统一教程

ORM 查询指南

遗留查询 API - 遗留 API 文档

```py
method refresh(instance: object, attribute_names: Iterable[str] | None = None, with_for_update: ForUpdateParameter = None) → None
```

使给定实例的属性过期并刷新。

首先将选定的属性作为当使用 `Session.expire()` 时会过期的那样进行过期处理；然后将发出 SELECT 语句到数据库，以使用当前事务中可用的当前值刷新基于列的属性。

如果对象已经被急切加载，那么 `relationship()` 定向属性也将立即被加载，使用它们最初加载时使用的急切加载策略。

版本 1.4 中的新功能：- `Session.refresh()` 方法也可以刷新急切加载的属性。

`relationship()` 导向属性通常会使用 `select`（或“延迟加载”）加载策略，如果它们在 `attribute_names` 集合中被明确命名，也会加载**，使用 `immediate` 加载策略对属性发出 SELECT 语句。如果惰性加载的关系未在`Session.refresh.attribute_names`中命名，则它们保持为“惰性加载”属性，不会被隐式刷新。

在版本 2.0.4 中更改：`Session.refresh()` 方法现在将刷新那些在`Session.refresh.attribute_names`集合中明确命名的惰性加载的`relationship()` 导向属性。

提示

虽然 `Session.refresh()` 方法能够刷新列和关系导向的属性，但其主要重点是在单个实例上刷新本地列导向的属性。要获得更开放的“刷新”功能，包括能够同时刷新许多对象的属性并明确控制关系加载策略，请使用 populate existing 功能。

请注意，高度隔离的事务将返回与在同一事务中先前读取的值相同的值，而不管该事务之外的数据库状态是否发生了变化。通常仅在事务开始时，尚未访问数据库行时刷新属性才有意义。

参数：

+   `attribute_names` – 可选。一个字符串属性名称的可迭代集合，指示要刷新的属性的子集。

+   `with_for_update` – 可选布尔值 `True`，指示是否应使用 FOR UPDATE，或者可能是一个包含标志的字典，用于指示更具体的 FOR UPDATE 标志集合用于 SELECT；标志应该与 `Query.with_for_update()` 的参数匹配。取代`Session.refresh.lockmode` 参数。

另请参阅

刷新/过期 - 初级材料

`Session.expire()`

`Session.expire_all()`

填充现有对象 - 允许任何 ORM 查询刷新对象，就像它们通常加载的那样。

```py
method reset() → None
```

关闭此`Session`使用的事务资源和 ORM 对象，将会话重置为其初始状态。

此方法提供了与`Session.close()`方法在历史上提供的相同的“仅重置”行为，其中`Session`的状态被重置，就像对象是全新的一样，准备好再次使用。然后，此方法可能对将`Session.close_resets_only`设置为`False`的`Session`对象有用，以便仍然可以使用“仅重置”行为。

新版本 2.0.22。

另请参阅

关闭 - 详细介绍了 `Session.close()` 和 `Session.reset()` 的语义。

`Session.close()` - 当参数 `Session.close_resets_only` 设置为 `False` 时，此类方法还将阻止会话的重复使用。

```py
method rollback() → None
```

回滚当前进行中的事务。

如果没有进行中的事务，则此方法是一个直通方法。

该方法始终回滚最顶层的数据库事务，丢弃可能正在进行的任何嵌套事务。

另请参阅

回滚

事务管理

```py
method scalar(statement: Executable, params: _CoreSingleExecuteParams | None = None, *, execution_options: OrmExecuteOptionsParameter = {}, bind_arguments: _BindArguments | None = None, **kw: Any) → Any
```

执行一条语句并返回一个标量结果。

用法和参数与`Session.execute()`相同；返回结果是一个标量 Python 值。

```py
method scalars(statement: Executable, params: _CoreAnyExecuteParams | None = None, *, execution_options: OrmExecuteOptionsParameter = {}, bind_arguments: _BindArguments | None = None, **kw: Any) → ScalarResult[Any]
```

执行一条语句并将结果作为标量返回。

用法和参数与`Session.execute()`相同；返回结果是一个`ScalarResult`过滤对象，它将返回单个元素而不是`Row`对象。

返回：

一个`ScalarResult`对象

新版本 1.4.24 中添加：`Session.scalars()`

新版本 1.4.26 中添加：`scoped_session.scalars()`

另请参阅

选择 ORM 实体 - 将 `Session.execute()` 的行为与 `Session.scalars()` 进行对比

```py
class sqlalchemy.orm.SessionTransaction
```

一个 `Session` 级别的事务。

`SessionTransaction` 是由 `Session.begin()` 和 `Session.begin_nested()` 方法生成的。它在现代用法中主要是一个内部对象，为会话事务提供上下文管理器。

与 `SessionTransaction` 交互的文档位于：管理事务。

自版本 1.4 更改：直接处理 `SessionTransaction` 对象的范围和 API 方法已经简化。

另请参阅

管理事务

`Session.begin()`

`Session.begin_nested()`

`Session.rollback()`

`Session.commit()`

`Session.in_transaction()`

`Session.in_nested_transaction()`

`Session.get_transaction()`

`Session.get_nested_transaction()`

**成员**

嵌套, 原始, 父级

**类签名**

类 `sqlalchemy.orm.SessionTransaction` (`sqlalchemy.orm.state_changes._StateChange`, `sqlalchemy.engine.util.TransactionalContext`)

```py
attribute nested: bool = False
```

指示这是否为嵌套或 SAVEPOINT 事务。

当 `SessionTransaction.nested` 为 True 时，预计 `SessionTransaction.parent` 也会出现，链接到封闭的 `SessionTransaction`。

另请参见

`SessionTransaction.origin`

```py
attribute origin: SessionTransactionOrigin
```

此 `SessionTransaction` 的来源。

指的是一个 `SessionTransactionOrigin` 实例，该实例是指导致构造此 `SessionTransaction` 的源事件的枚举。

新版本 2.0 中添加。

```py
attribute parent
```

此 `SessionTransaction` 的父级 `SessionTransaction`。

如果此属性为 `None`，表示此 `SessionTransaction` 位于堆栈顶部，并且对应于真实的 “COMMIT”/”ROLLBACK” 块。如果非 `None`，则这是一个 “子事务”（由刷新进程使用的内部标记对象）或 “嵌套” / SAVEPOINT 事务。如果 `SessionTransaction.nested` 属性为 `True`，则这是一个 SAVEPOINT，如果为 `False`，表示这是一个子事务。

```py
class sqlalchemy.orm.SessionTransactionOrigin
```

表示`SessionTransaction`的来源。

此枚举存在于任何 `SessionTransaction` 对象的 `SessionTransaction.origin` 属性上。

新版本 2.0 中添加。

**成员**

AUTOBEGIN, BEGIN, BEGIN_NESTED, SUBTRANSACTION

**类签名**

类`sqlalchemy.orm.SessionTransactionOrigin` (`enum.Enum`)

```py
attribute AUTOBEGIN = 0
```

通过自动开始开始的事务

```py
attribute BEGIN = 1
```

通过调用 `Session.begin()` 开始事务。

```py
attribute BEGIN_NESTED = 2
```

通过 `Session.begin_nested()` 开始事务。

```py
attribute SUBTRANSACTION = 3
```

事务是内部的 “子事务”

## 会话工具

| 对象名称 | 描述 |
| --- | --- |
| close_all_sessions() | 关闭内存中的所有会话。 |
| make_transient(instance) | 修改给定实例的状态，使其为瞬态。 |
| make_transient_to_detached(instance) | 使给定的瞬态实例成为脱离的。 |
| object_session(instance) | 返回给定实例所属的 `Session`。 |
| was_deleted(object_) | 如果给定对象在会话刷新中被删除，则返回 True。 |

```py
function sqlalchemy.orm.close_all_sessions() → None
```

关闭内存中的所有会话。

此函数会查询所有 `Session` 对象的全局注册表，并对它们调用 `Session.close()` ，将它们重置为干净的状态。

此函数不适用于一般用途，但在拆卸方案中的测试套件中可能有用。

版本 1.3 中的新功能。

```py
function sqlalchemy.orm.make_transient(instance: object) → None
```

改变给定实例的状态，使其成为瞬态的。

注意

`make_transient()` 是仅用于高级用例的特殊情况函数。

假定给定的映射实例处于持久的或脱离的状态。 该函数将删除其与任何 `Session` 的关联以及其 `InstanceState.identity`。 其效果是对象将表现得像它是新构造的，除了保留在调用时加载的任何属性/集合值。 如果此对象已因使用 `Session.delete()` 而被删除，则还将重置 `InstanceState.deleted` 标志。

警告

`make_transient()` **不**“取消过期”或以其他方式急切加载目前未加载的 ORM 映射属性。 这包括：

+   通过 `Session.expire()` 过期

+   由于提交会话事务的自然效果，会话过期，例如 `Session.commit()`

+   通常是惰性加载，但目前未加载

+   是“延迟加载”（参见 限制哪些列与列延迟加载）并且尚未加载

+   在加载此对象的查询中不存在，例如，在连接表继承和其他场景中常见的情况下。

在调用`make_transient()`之后，通常会解析为`None`的未加载属性，例如上面的属性，或者是针对集合导向属性的空集合。 由于对象是临时的，并且与任何数据库标识都没有关联，因此将不再检索这些值。

另请参阅

`make_transient_to_detached()`

```py
function sqlalchemy.orm.make_transient_to_detached(instance: object) → None
```

使给定的临时实例脱离。

注意

`make_transient_to_detached()`是一个仅用于高级用例的特殊情况函数。

给定实例的所有属性历史记录都将被重置，就好像该实例是从查询中新加载的一样。 缺少的属性将被标记为过期。 对象的主键属性将被制成实例的“键”，这些主键属性是必需的。

然后可以将对象添加到会话中，或者可能与 load=False 标志合并，此时它看起来就像是以这种方式加载的，而不会发出 SQL。

这是一个特殊的用例函数，它与对`Session.merge()`的普通调用不同，因为可以在不进行任何 SQL 调用的情况下制造给定的持久状态。

另请参阅

`make_transient()`

`Session.enable_relationship_loading()`

```py
function sqlalchemy.orm.object_session(instance: object) → Session | None
```

返回给定实例所属的`Session`。

这基本上与`InstanceState.session`访问器相同。 有关详细信息，请参阅该属性。

```py
function sqlalchemy.orm.util.was_deleted(object_: object) → bool
```

如果给定对象在会话刷新时被删除，则返回 True。

无论对象是否持久还是分离，都是如此。

另请参阅

`InstanceState.was_deleted`

## 属性和状态管理工具

这些函数由 SQLAlchemy 属性检测 API 提供，用于提供详细的接口来处理实例、属性值和历史记录。 在构建事件监听器函数时，其中一些函数非常有用，例如在 ORM 事件中描述的函数。

| 对象名称 | 描述 |
| --- | --- |
| del_attribute(instance, key) | 删除属性的值，并触发历史事件。 |
| flag_dirty(instance) | 标记一个实例为“脏”，而不具体提到任何属性。 |
| flag_modified(instance, key) | 将实例上的属性标记为“修改”。 |
| get_attribute(instance, key) | 获取属性的值，触发任何所需的可调用对象。 |
| get_history(obj, key[, passive]) | 为给定对象和属性键返回一个`History` 记录。 |
| History | 添加、未更改和已删除值的三元组，表示在工具化属性上发生的更改。 |
| init_collection(obj, key) | 初始化集合属性并返回集合适配器。 |
| instance_state | 返回给定映射对象的`InstanceState`。 |
| is_instrumented(instance, key) | 如果给定实例的给定属性由 attributes 包进行了工具化，则返回 True。 |
| object_state(instance) | 给定一个对象，返回与该对象关联的`InstanceState`。 |
| set_attribute(instance, key, value[, initiator]) | 设置属性的值，触发历史事件。 |
| set_committed_value(instance, key, value) | 设置属性的值，没有历史事件。 |

```py
function sqlalchemy.orm.util.object_state(instance: _T) → InstanceState[_T]
```

给定一个对象，返回与该对象关联的`InstanceState`。

如果没有配置映射，则会引发`sqlalchemy.orm.exc.UnmappedInstanceError`。

通过 `inspect()` 函数可以获得等效功能，如下所示：

```py
inspect(instance)
```

使用检查系统将会在实例不属于映射的情况下引发`sqlalchemy.exc.NoInspectionAvailable`。

```py
function sqlalchemy.orm.attributes.del_attribute(instance: object, key: str) → None
```

删除属性的值，触发历史事件。

无论直接应用于类的工具化如何，都可以使用此函数，即不需要描述符。自定义属性管理方案将需要使用此方法来建立由 SQLAlchemy 理解的属性状态。

```py
function sqlalchemy.orm.attributes.get_attribute(instance: object, key: str) → Any
```

获取属性的值，触发任何所需的可调用对象。

无论直接应用于类的仪器是什么，都可以使用此函数，即不需要描述符。自定义属性管理方案将需要使用此方法来使用 SQLAlchemy 理解的属性状态。

```py
function sqlalchemy.orm.attributes.get_history(obj: object, key: str, passive: PassiveFlag = symbol('PASSIVE_OFF')) → History
```

返回给定对象和属性键的`History`记录。

这是给定属性的**预刷新**历史记录，每次`Session`刷新对当前数据库事务进行更改时都会重置。

注意

优先使用`AttributeState.history`和`AttributeState.load_history()`访问器来检索实例属性的`History`。

参数：

+   `obj` - 其类由属性包装的对象。

+   `key` - 字符串属性名称。

+   `passive` - 如果值尚不存在，则指示属性的加载行为。这是一个比特标志属性，默认为符号`PASSIVE_OFF`，表示应发出所有必要的 SQL。

另请参阅

`AttributeState.history`

`AttributeState.load_history()` - 如果值未在本地存在，则使用加载器可调用检索历史记录。

```py
function sqlalchemy.orm.attributes.init_collection(obj: object, key: str) → CollectionAdapter
```

初始化集合属性并返回集合适配器。

此函数用于为先前未加载的属性提供直接访问集合内部。例如：

```py
collection_adapter = init_collection(someobject, 'elements')
for elem in values:
    collection_adapter.append_without_event(elem)
```

要更轻松地执行上述操作，请参见`set_committed_value()`。

参数：

+   `obj` - 一个映射对象

+   `key` - 集合所在的字符串属性名称。

```py
function sqlalchemy.orm.attributes.flag_modified(instance: object, key: str) → None
```

将实例上的属性标记为“修改”。

这将在实例上设置“修改”标志，并为给定属性建立一个无条件的更改事件。属性必须有一个值存在，否则会引发`InvalidRequestError`。

要将对象标记为“脏”，而不引用任何特定属性，以便在刷新时将其视为“脏”，请使用`flag_dirty()`调用。

另请参阅

`flag_dirty()`

```py
function sqlalchemy.orm.attributes.flag_dirty(instance: object) → None
```

将实例标记为“脏”，而不提及任何特定属性。

这是一个特殊操作，将允许对象通过刷新过程，以便被`SessionEvents.before_flush()`等事件拦截。请注意，对于没有更改的对象，在刷新过程中不会发出任何 SQL，即使通过此方法标记为脏。但是，`SessionEvents.before_flush()`处理程序将能够在`Session.dirty`集合中看到对象，并可能对其进行更改，然后将其包含在发出的 SQL 中。

版本 1.2 中的新功能。

另请参阅

`flag_modified()`

```py
function sqlalchemy.orm.attributes.instance_state()
```

返回给定映射对象的`InstanceState`。

此函数是`object_state()`的内部版本。在这里，建议使用`object_state()`和/或`inspect()`函数，因为它们会在给定对象未映射时分别发出信息性异常。

```py
function sqlalchemy.orm.instrumentation.is_instrumented(instance, key)
```

如果给定实例上的给定属性由属性包进行了仪器化，则返回 True。

无论直接应用于类的仪器化如何，都可以使用此函数，即不需要描述符。

```py
function sqlalchemy.orm.attributes.set_attribute(instance: object, key: str, value: Any, initiator: AttributeEventToken | None = None) → None
```

设置属性的值，触发历史事件。

无论直接应用于类的仪器化如何，都可以使用此函数，即不需要描述符。自定义属性管理方案将需要使用此方法来建立 SQLAlchemy 理解的属性状态。

参数：

+   `instance` – 将被修改的对象

+   `key` – 属性的字符串名称

+   `value` – 要分配的值

+   `initiator` –

    一个`Event`的实例，可能已从先前的事件侦听器传播。当在现有事件侦听函数中使用`set_attribute()`函数时，会使用此参数；该对象可用于跟踪事件链的起源。

    版本 1.2.3 中的新功能。

```py
function sqlalchemy.orm.attributes.set_committed_value(instance, key, value)
```

设置没有历史事件的属性的值。

取消任何先前存在的历史。对于持有标量属性的属性，值应为标量值，对于任何持有集合属性的属性，值应为可迭代对象。

当惰性加载程序触发并从数据库加载附加数据时，使用的是相同的基础方法。特别是，此方法可被应用代码使用，通过单独的查询加载了额外的属性或集合，然后将其附加到实例，就好像它是其原始加载状态的一部分。

```py
class sqlalchemy.orm.attributes.History
```

一个由添加、未更改和删除值组成的 3 元组，表示在受监控属性上发生的更改。

获取对象上特定属性的`History`对象的最简单方法是使用`inspect()`函数：

```py
from sqlalchemy import inspect

hist = inspect(myobject).attrs.myattribute.history
```

每个元组成员都是一个可迭代序列：

+   `added` - 添加到属性中的项目集合（第一个元组元素）。

+   `unchanged` - 未更改属性上的项目集合（第二个元组元素）。

+   `deleted` - 从属性中删除的项目集合（第三个元组元素）。

**成员**

added, deleted, empty(), has_changes(), non_added(), non_deleted(), sum(), unchanged

**类签名**

类`sqlalchemy.orm.History`（`builtins.tuple`）

```py
attribute added: Tuple[()] | List[Any]
```

字段 0 的别名

```py
attribute deleted: Tuple[()] | List[Any]
```

字段 2 的别名

```py
method empty() → bool
```

如果此`History`没有更改和现有的未更改状态，则返回 True。

```py
method has_changes() → bool
```

如果此`History`有更改，则返回 True。

```py
method non_added() → Sequence[Any]
```

返回一个未更改+删除的集合。

```py
method non_deleted() → Sequence[Any]
```

返回一个添加+未更改的集合。

```py
method sum() → Sequence[Any]
```

返回一个添加+未更改+删除的集合。

```py
attribute unchanged: Tuple[()] | List[Any]
```

字段 1 的别名

## Session 和 sessionmaker()

| 对象名称 | 描述 |
| --- | --- |
| ORMExecuteState | 表示对`Session.execute()`方法的调用，作为传递给`SessionEvents.do_orm_execute()`事件钩子。 |
| Session | 管理 ORM 映射对象的持久性操作。 |
| sessionmaker | 一个可配置的`Session`工厂。 |
| SessionTransaction | 一个`Session`级事务。 |
| SessionTransactionOrigin | 表示`SessionTransaction`的来源。 |

```py
class sqlalchemy.orm.sessionmaker
```

可配置的`Session`工厂。

当调用`sessionmaker`工厂时，会根据此处设定的配置参数生成新的`Session`对象。

例如：

```py
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

# an Engine, which the Session will use for connection
# resources
engine = create_engine('postgresql+psycopg2://scott:tiger@localhost/')

Session = sessionmaker(engine)

with Session() as session:
    session.add(some_object)
    session.add(some_other_object)
    session.commit()
```

上下文管理器的使用是可选的；否则，返回的`Session`对象可以通过`Session.close()`方法显式关闭。使用`try:/finally:`块是可选的，但是会确保即使出现数据库错误也会执行关闭操作：

```py
session = Session()
try:
    session.add(some_object)
    session.add(some_other_object)
    session.commit()
finally:
    session.close()
```

`sessionmaker`的作用类似于`Engine`对`Connection`对象的工厂。通过这种方式，它还包括一个`sessionmaker.begin()`方法，提供一个上下文管理器，既开始又提交事务，并在完成时关闭`Session`，如果发生任何错误则回滚事务：

```py
Session = sessionmaker(engine)

with Session.begin() as session:
    session.add(some_object)
    session.add(some_other_object)
# commits transaction, closes session
```

1.4 版本中新增。

当调用`sessionmaker`构造`Session`时，也可以传递关键字参数给该方法；这些参数将覆盖全局配置的参数。下面我们使用一个绑定到特定`Engine`的`sessionmaker`来生成一个与该引擎提供的特定`Connection`绑定的`Session`：

```py
Session = sessionmaker(engine)

# bind an individual session to a connection

with engine.connect() as connection:
    with Session(bind=connection) as session:
        # work with session
```

该类还包括一个`sessionmaker.configure()` 方法，该方法可用于指定工厂的额外关键字参数，这些参数将对后续生成的`Session` 对象生效。通常用于在首次使用之前将一个或多个`Engine` 对象与现有的`sessionmaker` 工厂关联起来：

```py
# application starts, sessionmaker does not have
# an engine bound yet
Session = sessionmaker()

# ... later, when an engine URL is read from a configuration
# file or other events allow the engine to be created
engine = create_engine('sqlite:///foo.db')
Session.configure(bind=engine)

sess = Session()
# work with session
```

另请参阅

打开和关闭会话 - 使用`sessionmaker` 创建会话的简介文本。

**成员**

__call__(), __init__(), begin(), close_all(), configure(), identity_key(), object_session()

**类签名**

类`sqlalchemy.orm.sessionmaker` (`sqlalchemy.orm.session._SessionClassMethods`, `typing.Generic`)

```py
method __call__(**local_kw: Any) → _S
```

使用此`sessionmaker` 中建立的配置生成新的`Session` 对象。

在 Python 中，当对象“调用”方式与函数相同时，将调用对象上的`__call__`方法：

```py
Session = sessionmaker(some_engine)
session = Session()  # invokes sessionmaker.__call__()
```

```py
method __init__(bind: Optional[_SessionBind] = None, *, class_: Type[_S] = <class 'sqlalchemy.orm.session.Session'>, autoflush: bool = True, expire_on_commit: bool = True, info: Optional[_InfoType] = None, **kw: Any)
```

构造一个新的`sessionmaker`。

这里的所有参数，除了`class_`，都对应于直接由`Session` 接受的参数。有关参数的更多详细信息，请参阅`Session.__init__()` 文档字符串。

参数：

+   `bind` – 与新创建的`Session` 对象关联的`Engine` 或其他 `Connectable`。

+   `class_` – 用于创建新的`Session` 对象的类。默认为`Session`。

+   `autoflush` –

    与新创建的`Session` 对象一起使用的自动刷新设置。

    另请参阅

    刷新 - 关于自动刷新的额外背景信息

+   `expire_on_commit=True` – 与新创建的`Session` 对象一起使用的`Session.expire_on_commit` 设置。

+   `info` – 可选字典，可以通过`Session.info`访问。请注意，当`info`参数被指定给特定的`Session`构造操作时，此字典会被*更新*而不是被替换。

+   `**kw` – 所有其他关键字参数都将传递给新创建的`Session`对象的构造函数。

```py
method begin() → AbstractContextManager[_S]
```

生成一个上下文管理器，既提供一个新的`Session`又提供一个提交的事务。

例如：

```py
Session = sessionmaker(some_engine)

with Session.begin() as session:
    session.add(some_object)

# commits transaction, closes session
```

自版本 1.4 起新增。

```py
classmethod close_all() → None
```

*继承自* `sqlalchemy.orm.session._SessionClassMethods.close_all` *方法的* `sqlalchemy.orm.session._SessionClassMethods`

关闭*所有*内存中的会话。

自版本 1.3 起弃用：`Session.close_all()`方法已弃用，将在将来的版本中删除。请参考`close_all_sessions()`。

```py
method configure(**new_kw: Any) → None
```

(重新)配置此 sessionmaker 的参数。

例如：

```py
Session = sessionmaker()

Session.configure(bind=create_engine('sqlite://'))
```

```py
classmethod identity_key(class_: Type[Any] | None = None, ident: Any | Tuple[Any, ...] = None, *, instance: Any | None = None, row: Row[Any] | RowMapping | None = None, identity_token: Any | None = None) → _IdentityKeyType[Any]
```

*继承自* `sqlalchemy.orm.session._SessionClassMethods.identity_key` *方法的* `sqlalchemy.orm.session._SessionClassMethods`

返回一个标识键。

这是`identity_key()`的别名。

```py
classmethod object_session(instance: object) → Session | None
```

*继承自* `sqlalchemy.orm.session._SessionClassMethods.object_session` *方法的* `sqlalchemy.orm.session._SessionClassMethods`

返回对象所属的`Session`。

这是`object_session()`的别名。

```py
class sqlalchemy.orm.ORMExecuteState
```

表示对`Session.execute()`方法的调用，如传递给`SessionEvents.do_orm_execute()`事件挂钩。

自版本 1.4 起新增。

另请参见

执行事件 - 如何使用`SessionEvents.do_orm_execute()`的顶级文档

**成员**

__init__(), all_mappers, bind_arguments, bind_mapper, execution_options, invoke_statement(), is_column_load, is_delete, is_executemany, is_from_statement, is_insert, is_orm_statement, is_relationship_load, is_select, is_update, lazy_loaded_from, load_options, loader_strategy_path, local_execution_options, parameters, session, statement, update_delete_options, update_execution_options(), user_defined_options

**类签名**

类`sqlalchemy.orm.ORMExecuteState` (`sqlalchemy.util.langhelpers.MemoizedSlots`)

```py
method __init__(session: Session, statement: Executable, parameters: _CoreAnyExecuteParams | None, execution_options: _ExecuteOptions, bind_arguments: _BindArguments, compile_state_cls: Type[ORMCompileState] | None, events_todo: List[_InstanceLevelDispatch[Session]])
```

构造一个新的`ORMExecuteState`。

此对象在内部构造。

```py
attribute all_mappers
```

返回一个包含此语句顶层涉及的所有`Mapper`对象的序列。

所谓“顶层”是指那些在`select()`查询的结果集行中表示的`Mapper`对象，或者对于`update()`或`delete()`查询，即 UPDATE 或 DELETE 的主要主题的映射器。

版本 1.4.0b2 中的新内容。

另请参阅

`ORMExecuteState.bind_mapper`

```py
attribute bind_arguments: _BindArguments
```

作为`Session.execute.bind_arguments`字典传递的字典。

此字典可供`Session`的扩展使用，以传递将帮助确定在一组数据库连接中应使用哪一个来调用此语句的参数。

```py
attribute bind_mapper
```

返回主要的“bind”映射器`Mapper`。

对于调用 ORM 语句的`ORMExecuteState`对象，即`ORMExecuteState.is_orm_statement`属性为`True`，此属性将返回被认为是语句的“主要”映射器的`Mapper`。术语“bind mapper”指的是`Session`对象可能被“绑定”到多个映射类键到映射类的多个`Engine`对象，并且“bind mapper”确定哪些`Engine`对象将被选择。

对于针对单个映射类调用的语句，`ORMExecuteState.bind_mapper` 旨在是一种可靠的方式来获取此映射器。

版本 1.4.0b2 中的新功能。

另请参阅

`ORMExecuteState.all_mappers`

```py
attribute execution_options: _ExecuteOptions
```

当前执行选项的完整字典。

这是语句级选项与本地传递的执行选项的合并。

另请参阅

`ORMExecuteState.local_execution_options`

`Executable.execution_options()`

ORM 执行选项

```py
method invoke_statement(statement: Executable | None = None, params: _CoreAnyExecuteParams | None = None, execution_options: OrmExecuteOptionsParameter | None = None, bind_arguments: _BindArguments | None = None) → Result[Any]
```

执行由此`ORMExecuteState`表示的语句，而不重新调用已经进行的事件。

该方法实质上执行当前语句的可重入执行，当前正在调用`SessionEvents.do_orm_execute()`事件。这样做的用例是为了事件处理程序想要覆盖最终`Result`对象返回方式，比如从离线缓存中检索结果或者从多次执行中连接结果的方案。

当实际处理程序函数在`SessionEvents.do_orm_execute()`中返回`Result`对象，并传播到调用的`Session.execute()`方法时，`Session.execute()`方法的其余部分被抢占，`Result`对象立即返回给`Session.execute()`的调用者。

参数：

+   `statement` – 可选的要调用的语句，代替当前由`ORMExecuteState.statement`表示的语句。

+   `params` –

    可选的参数字典或参数列表，将合并到此`ORMExecuteState`的现有`ORMExecuteState.parameters`中。

    从版本 2.0 开始更改：接受参数字典列表进行 executemany 执行。

+   `execution_options` – 可选的执行选项字典将合并到此`ORMExecuteState`的现有`ORMExecuteState.execution_options`中。

+   `bind_arguments` – 可选的绑定参数字典，将在此`ORMExecuteState`的当前`ORMExecuteState.bind_arguments`中合并。

返回值：

一个带有 ORM 级别结果的`Result`对象。

另请参阅

重新执行语句 - 关于适当使用 `ORMExecuteState.invoke_statement()` 的背景和示例。

```py
attribute is_column_load
```

如果操作是刷新现有 ORM 对象上的面向列的属性，则返回 True。

在诸如 `Session.refresh()` 之类的操作期间发生，以及在加载由 `defer()` 推迟的属性时，或者正在加载已由 `Session.expire()` 直接或通过提交操作过期的属性时。

当发生此类操作时，处理程序很可能不希望向查询添加任何选项，因为查询应该是一个直接的主键获取，不应该有任何额外的 WHERE 条件，并且随实例传递的加载器选项已经添加到查询中。

新版本 1.4.0b2 中新增。

另请参阅

`ORMExecuteState.is_relationship_load`

```py
attribute is_delete
```

如果这是一个 DELETE 操作，则返回 True。

在版本 2.0.30 中更改：- 该属性对于自身针对 `Delete` 构造的 `Select.from_statement()` 构造也为 True，例如 `select(Entity).from_statement(delete(..))`

```py
attribute is_executemany
```

如果参数是一个多元素字典列表，并且有多个字典，则返回 True。

新版本 2.0 中新增。

```py
attribute is_from_statement
```

��果这是一个 `Select.from_statement()` 操作，则返回 True。

这与 `ORMExecuteState.is_select` 独立，因为 `select().from_statement()` 构造也可以与 INSERT/UPDATE/DELETE RETURNING 类型的语句一起使用。只有在 `Select.from_statement()` 本身针对 `Select` 构造时，才会设置 `ORMExecuteState.is_select`。

新版本 2.0.30 中新增。

```py
attribute is_insert
```

如果这是一个 INSERT 操作，则返回 True。

在版本 2.0.30 中更改：- 对于`Select.from_statement()`构造本身针对`Insert`构造，例如`select(Entity).from_statement(insert(..))`，该属性也为 True

```py
attribute is_orm_statement
```

如果操作是 ORM 语句，则返回 True。

这表明调用 select()、insert()、update()或 delete()包含 ORM 实体作为主题。对于不包含 ORM 实体而仅引用`Table`元数据的语句，它被调用为核心 SQL 语句，并且不会发生 ORM 级别的自动化。

```py
attribute is_relationship_load
```

如果此加载正在代表关系加载对象，则返回 True。

这意味着，实际上加载程序是一个 LazyLoader、SelectInLoader、SubqueryLoader 或类似的加载程序，并且整个 SELECT 语句是代表一个关系加载发出的。

当发生此类操作时，处理程序很可能不希望向查询添加任何选项，因为加载程序选项已经能够传播到关系加载程序并且应该已经存在。

另请参阅

`ORMExecuteState.is_column_load`

```py
attribute is_select
```

如果这是一个选择操作，则返回 True。

在版本 2.0.30 中更改：- 对于`Select.from_statement()`构造本身针对`Select`构造，例如`select(Entity).from_statement(select(..))`，该属性也为 True

```py
attribute is_update
```

如果这是一个更新操作，则返回 True。

在版本 2.0.30 中更改：- 对于`Select.from_statement()`构造本身针对`Update`构造，例如`select(Entity).from_statement(update(..))`，该属性也为 True

```py
attribute lazy_loaded_from
```

使用此语句执行进行惰性加载操作的`InstanceState`。

此属性的主要原因是支持水平分片扩展，该扩展在此扩展创建的特定查询执行时间挂钩中可用。为此，该属性仅打算在**查询执行时间**有意义，并且重要的是不包括任何之前的时间，包括查询编译时间。

```py
attribute load_options
```

返回将用于此执行的 load_options。

```py
attribute loader_strategy_path
```

返回当前加载路径的`PathRegistry`。

此对象表示在查询中沿着关系的“路径”时，加载特定对象或集合的情况。

```py
attribute local_execution_options: _ExecuteOptions
```

传递给`Session.execute()`方法的执行选项的字典视图。

此处不包括与所调用语句相关的选项。

另请参阅

`ORMExecuteState.execution_options`

```py
attribute parameters: _CoreAnyExecuteParams | None
```

传递给`Session.execute()`的参数字典。

```py
attribute session: Session
```

正在使用的`Session`。

```py
attribute statement: Executable
```

所调用的 SQL 语句。

对于从`Query`检索的 ORM 选择，这是从 ORM 查询生成的`select`的一个实例。

```py
attribute update_delete_options
```

返回将用于此执行的 update_delete_options。

```py
method update_execution_options(**opts: Any) → None
```

使用新值更新本地执行选项。

```py
attribute user_defined_options
```

已与所调用语句相关联的`UserDefinedOptions`序列。

```py
class sqlalchemy.orm.Session
```

管理 ORM 映射对象的持久性操作。

`Session`**不适合在并发线程中使用**。请参阅 Session 线程安全吗？AsyncSession 在并发任务中安全共享吗？了解背景信息。

关于 Session 的使用范例，请参阅使用 Session。

**成员**

__init__(), add(), add_all(), begin(), begin_nested(), bind_mapper(), bind_table(), bulk_insert_mappings(), bulk_save_objects(), bulk_update_mappings(), close(), close_all(), commit(), connection(), delete(), deleted, dirty, enable_relationship_loading(), execute(), expire(), expire_all(), expunge(), expunge_all(), flush(), get(), get_bind(), get_nested_transaction(), get_one(), get_transaction(), identity_key(), identity_map, in_nested_transaction(), in_transaction(), info, invalidate(), is_active, is_modified(), merge(), new, no_autoflush, object_session(), prepare(), query(), refresh(), reset(), rollback(), scalar(), scalars()

**类签名**

类`sqlalchemy.orm.Session` (`sqlalchemy.orm.session._SessionClassMethods`, `sqlalchemy.event.registry.EventTarget`)

```py
method __init__(bind: _SessionBind | None = None, *, autoflush: bool = True, future: Literal[True] = True, expire_on_commit: bool = True, autobegin: bool = True, twophase: bool = False, binds: Dict[_SessionBindKey, _SessionBind] | None = None, enable_baked_queries: bool = True, info: _InfoType | None = None, query_cls: Type[Query[Any]] | None = None, autocommit: Literal[False] = False, join_transaction_mode: JoinTransactionMode = 'conditional_savepoint', close_resets_only: bool | _NoArg = _NoArg.NO_ARG)
```

构造一个新的`Session`。

请参见`sessionmaker`函数，该函数用于生成带有给定参数集的`Session`产生的可调用对象。

参数：

+   `autoflush` –

    当为`True`时，所有查询操作将在继续之前对此`Session`执行`Session.flush()`调用。这是一个方便的特性，使得不需要反复调用`Session.flush()`以便数据库查询检索结果。

    请参见

    刷新 - 关于自动刷新的额外背景信息

+   `autobegin` –

    在请求操作时自动启动事务（即等同于调用`Session.begin()`） 。默认为`True`。设置为`False`以防止`Session`在构造之后以及在调用任何 `Session.rollback()`、`Session.commit()` 或 `Session.close()` 方法之后隐式开始事务。

    从 2.0 版本开始新增。

    请参见

    禁用 Autobegin 以防止隐式事务

+   `bind` – 可选的`Engine`或`Connection`，应该绑定到此`Session`。指定时，此会话执行的所有 SQL 操作将通过此可连接对象执行。

+   `binds` –

    一个字典，可能指定任意数量的`Engine`或`Connection`对象作为 SQL 操作的连接源，以实体为单位。字典的键由任何一系列映射类、任意 Python 类（作为映射类的基类）、`Table`对象和`Mapper`对象组成。然后，字典的值是`Engine`的实例，或者较少见的是`Connection`对象。针对特定映射类进行的操作将查阅此字典，以确定哪个`Engine`应该用于特定的 SQL 操作。解析的完整启发式描述在`Session.get_bind()`中。用法示例如下：

    ```py
    Session = sessionmaker(binds={
        SomeMappedClass: create_engine('postgresql+psycopg2://engine1'),
        SomeDeclarativeBase: create_engine('postgresql+psycopg2://engine2'),
        some_mapper: create_engine('postgresql+psycopg2://engine3'),
        some_table: create_engine('postgresql+psycopg2://engine4'),
        })
    ```

    另请参见

    分区策略（例如，每个会话多个数据库后端）

    `Session.bind_mapper()`

    `Session.bind_table()`

    `Session.get_bind()`

+   `class_` – 指定除了 `sqlalchemy.orm.session.Session` 外应该由返回的类使用的替代类。这是唯一作为 `sessionmaker` 函数本地的参数，并且不直接发送到 `Session` 的构造函数。

+   `enable_baked_queries` –

    遗留；默认为 `True`。由 `sqlalchemy.ext.baked` 扩展使用的参数，用于确定是否应缓存“烘焙查询”，如此扩展的正常操作所用。当设置为 `False` 时，此特定扩展使用的缓存被禁用。

    从版本 1.4 起更改：`sqlalchemy.ext.baked` 扩展是遗留的，并且没有被 SQLAlchemy 的任何内部使用。因此，此标志仅影响明确在其自己的代码中使用此扩展的应用程序。

+   `expire_on_commit` –

    默认为 `True`。当为 `True` 时，每次 `commit()` 后所有实例都将完全过期，以便在完成事务后的所有属性/对象访问加载最新的数据库状态。

    > 另请参见
    > 
    > 提交

+   `future` –

    已弃用；此标志始终为 True。

    另请参见

    SQLAlchemy 2.0 - 主要迁移指南

+   `info` – 可选字典，与此 `Session` 关联的任意数据。可通过 `Session.info` 属性访问。请注意，字典在构造时被复制，因此对每个 `Session` 字典的修改将局限于该 `Session`。

+   `query_cls` – 应该用于创建新的查询对象的类，由 `Session.query()` 方法返回。默认为 `Query`。

+   `twophase` – 当设置为`True`时，所有事务都将作为“两阶段”事务启动，即使用正在使用的数据库的“两阶段”语义以及一个 XID。在`commit()`之后，对所有已附加数据库发出`flush()`后，将调用每个数据库的`TwoPhaseTransaction.prepare()`方法的`TwoPhaseTransaction`。这允许每个数据库在提交每个事务之前回滚整个事务。

+   `autocommit` – `autocommit`关键字出现是为了向后兼容，但必须保持其默认值为`False`。

+   `join_transaction_mode` –

    描述了在给定绑定是`Session`之外已经开始事务的`Connection`时应采取的事务行为；换句话说，`Connection.in_transaction()`方法返回 True。

    以下行为仅在`Session` **实际使用给定的连接**时才生效；也就是说，诸如`Session.execute()`、`Session.connection()`等方法实际上被调用：

    +   `"conditional_savepoint"` - 这是默认行为。如果给定的`Connection`在事务中开始但没有 SAVEPOINT，则使用`"rollback_only"`。如果`Connection`此外还在 SAVEPOINT 中，换句话说，`Connection.in_nested_transaction()`方法返回 True，则使用`"create_savepoint"`。

        `"conditional_savepoint"` 行为试图利用保存点来保持现有事务的状态不变，但仅在已经存在保存点的情况下；否则，不假设所使用的后端具有足够的 SAVEPOINT 支持，因为该功能的可用性有所不同。 `"conditional_savepoint"` 还试图与先前 `Session` 行为建立大致的向后兼容性，用于未设置特定模式的应用程序。建议使用其中一个明确的设置。

    +   `"create_savepoint"` - `Session` 将在所有情况下使用 `Connection.begin_nested()` 来创建自己的事务。这个事务本质上是“在顶部”的任何存在的事务之上打开的；如果底层数据库和正在使用的驱动程序完全支持 SAVEPOINT，那么外部事务将在 `Session` 生命周期内保持不受影响。

        `"create_savepoint"` 模式对于将 `Session` 集成到测试套件中并保持外部启动的事务不受影响最为有用；然而，它依赖于底层驱动程序和数据库的正确 SAVEPOINT 支持。

        提示

        使用 SQLite 时，Python 3.11 中包含的 SQLite 驱动在某些情况下不能正确处理 SAVEPOINTs，需要通过一些变通方法。请参阅 Serializable isolation / Savepoints / Transactional DDL 和 Serializable isolation / Savepoints / Transactional DDL（asyncio 版本）部分，了解当前的解决方法详情。

    +   `"control_fully"` - `Session` 将接管给定的事务作为自己的事务；`Session.commit()` 将在事务上调用 `.commit()`，`Session.rollback()` 将在事务上调用 `.rollback()`，`Session.close()` 将调用事务的 `.rollback`。

        提示

        此使用模式相当于 SQLAlchemy 1.4 如何处理具有现有 SAVEPOINT 的`Connection`（即`Connection.begin_nested()`）; `Session`将完全控制现有 SAVEPOINT。

    +   `"rollback_only"` - `Session`将仅控制给定事务进行`.rollback()`调用；`.commit()`调用不会传播到给定事务。`.close()`调用对给定事务没有影响。

        提示

        此使用模式相当于 SQLAlchemy 1.4 如何处理具有现有常规数据库事务的`Connection`（即`Connection.begin()`）; `Session`将传播`Session.rollback()`调用到底层事务，但不会传播`Session.commit()`或`Session.close()`调用。

    在版本 2.0.0rc1 中新增。

+   `close_resets_only` –

    默认为`True`。确定在调用`.close()`后会话是否应重置自身，或者应传入不再可用的状态，禁用重用。

    新版本 2.0.22 中新增标志`close_resets_only`。将来的 SQLAlchemy 版本可能会将此标志的默认值更改为`False`。

    另请参阅

    关闭 - 有关`Session.close()`和`Session.reset()`语义的详细信息。

```py
method add(instance: object, _warn: bool = True) → None
```

将对象放入此`Session`。

当传递给`Session.add()`方法时处于瞬态状态的对象将转移到挂起状态，直到下一次刷新，然后它们将转移到持久化状态。

当传递给`Session.add()`方法时处于分离状态的对象将直接转移到持久化状态。

如果`Session`使用的事务被回滚，则在传递给`Session.add()`时是瞬时的对象将被移回瞬时状态，并且将不再存在于此`Session`中。

另请参见

`Session.add_all()`

添加新项目或现有项目 - 在使用会话的基础知识中

```py
method add_all(instances: Iterable[object]) → None
```

将给定的实例集合添加到此`Session`中。

请参阅`Session.add()`的文档以获取一般行为描述。

另请参见

`Session.add()`

添加新项目或现有项目 - 在使用会话的基础知识中

```py
method begin(nested: bool = False) → SessionTransaction
```

在此`Session`上开始事务或嵌套事务（如果尚未开始）。

`Session`对象具有**自动开始**行为，因此通常不需要显式调用`Session.begin()`方法。但是，可以用来控制事务状态开始的范围。

当用于开始最外层事务时，如果此`Session`已经在事务中，则会引发错误。

参数：

**嵌套** - 如果为 True，则开始一个 SAVEPOINT 事务，等效于调用`Session.begin_nested()`。有关 SAVEPOINT 事务的文档，请参见使用 SAVEPOINT。

返回：

`SessionTransaction`对象。请注意，`SessionTransaction`充当 Python 上下文管理器，允许在“with”块中使用`Session.begin()`。请参见显式开始以获取示例。

另请参见

自动开始

管理事务

`Session.begin_nested()`

```py
method begin_nested() → SessionTransaction
```

在此会话上开始“嵌套”事务，例如 SAVEPOINT。

目标数据库和相关驱动程序必须支持 SQL SAVEPOINT 才能使此方法正常运行。

有关 SAVEPOINT 事务的文档，请参阅使用 SAVEPOINT。

返回：

`SessionTransaction`对象。请注意，`SessionTransaction`作为上下文管理器，允许在“with”块中使用`Session.begin_nested()`。请参阅使用 SAVEPOINT 获取用法示例。

另请参阅

使用 SAVEPOINT

可序列化隔离/保存点/事务性 DDL - 针对 SQLite 驱动程序需要特殊的解决方案，以使 SAVEPOINT 正常工作。对于 asyncio 用例，请参阅可序列化隔离/保存点/事务性 DDL（asyncio 版本）部分。

```py
method bind_mapper(mapper: _EntityBindKey[_O], bind: _SessionBind) → None
```

将`Mapper`或任意 Python 类与“bind”相关联，例如`Engine`或`Connection`。

给定的实体被添加到`Session.get_bind()`方法使用的查找中。

参数：

+   `mapper` – 一个`Mapper`对象，或者是一个映射类的实例，或者是任何作为一组映射类基类的 Python 类。

+   `bind` – 一个`Engine`或`Connection`对象。

另请参阅

分区策略（例如每个会话多个数据库后端）

`Session.binds`

`Session.bind_table()`

```py
method bind_table(table: TableClause, bind: Engine | Connection) → None
```

将`Table`与“bind”相关联，例如`Engine`或`Connection`。

给定的`Table`被添加到`Session.get_bind()`方法使用的查找中。

参数：

+   `table` – 一个`Table`对象，通常是 ORM 映射的目标，或者存在于被映射的可选择性内。

+   `bind` – 一个`Engine`或`Connection`对象。

另请参阅

分区策略（例如每个 Session 多个数据库后端）

`Session.binds`

`Session.bind_mapper()`

```py
method bulk_insert_mappings(mapper: Mapper[Any], mappings: Iterable[Dict[str, Any]], return_defaults: bool = False, render_nulls: bool = False) → None
```

对给定的映射字典列表执行批量插入。

旧特性

该方法是 SQLAlchemy 2.0 系列的旧特性。对于现代的批量 INSERT 和 UPDATE，请参阅 ORM 批量 INSERT 语句和 ORM 按主键批量 UPDATE 部分。2.0 API 与此方法共享实现细节，并添加了新功能。

参数：

+   `mapper` – 一个映射类，或者实际的`Mapper`对象，表示映射列表中表示的单一对象类型。

+   `mappings` – 一个字典序列，每个字典包含要插入的映射行的状态，以映射类上的属性名称为准。如果映射涉及多个表，比如联合继承映射，每个字典必须包含要填充到所有表中的所有键。

+   `return_defaults` –

    当为 True 时，INSERT 过程将被修改以确保新生成的主键值将被获取。通常此参数的理由是为了使联合表继承映射能够进行批量插入。

    注意

    对于不支持 RETURNING 的后端，`Session.bulk_insert_mappings.return_defaults`参数可以显著降低性能，因为 INSERT 语句不能再批量处理。请参阅“插入多个值”行为对 INSERT 语句的影响以了解哪些后端受到影响。

+   `render_nulls` –

    当为 True 时，`None`值将导致在 INSERT 语句中包含一个 NULL 值，而不是将列从 INSERT 中省略。这允许所有被 INSERT 的行具有相同的列集，从而允许将完整的行集批量发送到 DBAPI。通常，每个包含与前一行不同的 NULL 值组合的列集必须从渲染的 INSERT 语句中省略不同的列系列，这意味着必须将其作为单独的语句发出。通过传递此标志，可以确保将完整的行集批量处理为一个批次；但成本是，通过省略的列调用的服务器端默认值将被跳过，因此必须确保这些不是必需的。

    警告

    当设置了此标志时，**服务器端默认的 SQL 值不会被调用**，对于那些以 NULL 插入的列；NULL 值将被显式发送。必须确保整个操作不需要调用任何服务器端默认函数。

另请参阅

启用 ORM 的 INSERT、UPDATE 和 DELETE 语句

`Session.bulk_save_objects()`

`Session.bulk_update_mappings()`

```py
method bulk_save_objects(objects: Iterable[object], return_defaults: bool = False, update_changed_only: bool = True, preserve_order: bool = True) → None
```

对给定对象列表执行批量保存。

旧特性

该方法是 SQLAlchemy 2.0 系列的一个旧特性。对于现代的批量 INSERT 和 UPDATE，请参阅 ORM 批量 INSERT 语句和按主键批量 UPDATE 部分。

对于一般的 INSERT 和更新现有 ORM 映射对象，建议使用标准的工作单元数据管理模式，介绍在 SQLAlchemy 统一教程中的 ORM 数据操作。SQLAlchemy 2.0 现在使用现代方言的“插入多个值”行为用于 INSERT 语句，解决了以前批量 INSERT 速度慢的问题。

参数：

+   `objects` –

    一系列映射对象实例。映射对象将按原样持久化，并且之后**不**与`Session`关联。

    对于每个对象，对象是作为 INSERT 还是 UPDATE 发送取决于`Session`在传统操作中使用的相同规则；如果对象具有`InstanceState.key`属性设置，则假定对象是“分离的”并将导致 UPDATE。否则，将使用 INSERT。

    在 UPDATE 的情况下，根据哪些属性已更改，语句将被分组，因此将成为每个 SET 子句的主题。如果`update_changed_only`为 False，则将每个对象中存在的所有属性应用于 UPDATE 语句，这可能有助于将语句组合成更大的 executemany()，并且还将减少检查属性历史记录的开销。

+   `return_defaults` – 当为 True 时，缺少生成默认值的值的行，即整数主键默认值和序列，将逐个插入，以便主键值可用。特别是，这将允许联合继承和其他多表映射正确插入，而无需提前提供主键值；但是，`Session.bulk_save_objects.return_defaults` `极大地降低了方法的性能收益`。强烈建议使用标准的`Session.add_all()`方法。

+   `update_changed_only` – 当为 True 时，根据每个状态中记录的更改的属性生成 UPDATE 语句。当为 False 时，除了主键属性之外，所有存在的属性都将生成到 SET 子句中。

+   `preserve_order` – 当为 True 时，插入和更新的顺序与给定对象的顺序完全匹配。当为 False 时，常见类型的对象将分组为插入和更新，以允许更多的批处理机会。

另请参阅

ORM 启用的 INSERT、UPDATE 和 DELETE 语句

`Session.bulk_insert_mappings()`

`Session.bulk_update_mappings()`

```py
method bulk_update_mappings(mapper: Mapper[Any], mappings: Iterable[Dict[str, Any]]) → None
```

对给定的映射字典列表执行批量更新。

遗留特性

该方法是 SQLAlchemy 2.0 系列的遗留特性。有关现代批量 INSERT 和 UPDATE，请参见 ORM 批量 INSERT 语句和 ORM 按主键批量 UPDATE 章节。2.0 API 与该方法共享实现细节，并添加了新功能。

参数：

+   `mapper` – 映射类，或者表示映射列表中所表示的单个对象的实际`Mapper`对象。

+   `mappings` - 一个字典序列，每个字典包含要更新的映射行的状态，以映射类上的属性名称表示。如果映射涉及多个表，例如连接继承映射，每个字典可能包含与所有表对应的键。所有那些存在且不是主键的键将应用于 UPDATE 语句的 SET 子句；必需的主键值将应用于 WHERE 子句。

另请参阅

启用 ORM 的 INSERT、UPDATE 和 DELETE 语句

`Session.bulk_insert_mappings()`

`Session.bulk_save_objects()`

```py
method close() → None
```

关闭此`Session`使用的事务资源和 ORM 对象。

这将清除与此`Session`关联的所有 ORM 对象，结束任何正在进行的事务，并释放此`Session`自身从关联的`Engine`对象中签出的任何`Connection`对象。然后，该操作将使`Session`处于可以再次使用的状态。

提示

在默认运行模式下，`Session.close()`方法**不会阻止再次使用 Session**。`Session`本身实际上没有明确的“关闭”状态；它仅意味着`Session`将释放所有数据库连接和 ORM 对象。

将参数`Session.close_resets_only`设置为`False`将使`close`变为最终状态，这意味着对会话的任何进一步操作都将被禁止。

从版本 1.4 开始更改：`Session.close()`方法不会立即创建新的`SessionTransaction`对象；相反，只有在再次为数据库操作使用`Session`时才会创建新的`SessionTransaction`。

另请参阅

关闭会话 - 关于`Session.close()`和`Session.reset()`语义的详细信息。

`Session.reset()` - 与参数`Session.close_resets_only`设置为`True`的`close()`类似的方法。

```py
classmethod close_all() → None
```

*从* `sqlalchemy.orm.session._SessionClassMethods.close_all` *方法继承*

关闭内存中的*所有*会话。

从版本 1.3 开始已废弃：`Session.close_all()` 方法已被废弃，并将在将来的版本中删除。请参考 `close_all_sessions()`。

```py
method commit() → None
```

刷新待定更改并提交当前事务。

当 COMMIT 操作完成时，所有对象都将被完全过期，擦除其内部内容，下次访问这些对象时将自动重新加载。在此期间，这些对象处于过期状态，如果从`Session`中分离，则无法正常使用。此外，在使用基于 asyncio 的 API 时不支持此重新加载操作。可以使用`Session.expire_on_commit`参数来禁用此行为。

当`Session`中没有事务时，表示自上次调用`Session.commit()`以来没有在此`Session`上调用任何操作，则该方法将开始并提交一个仅内部使用的“逻辑”事务，通常不会影响数据库，除非检测到待定刷新更改，但仍会调用事件处理程序和对象过期规则。

最外层的数据库事务会无条件提交，自动释放任何当前有效的 SAVEPOINT。

另请参阅

提交

事务管理

在使用 AsyncSession 时避免隐式 IO

```py
method connection(bind_arguments: _BindArguments | None = None, execution_options: CoreExecuteOptionsParameter | None = None) → Connection
```

返回与此`Session`对象的事务状态对应的`Connection`对象。

返回与当前事务对应的`Connection`，或者如果没有进行中的事务，则开始一个新事务并返回`Connection`（请注意，在发出第一条 SQL 语句之前，不会与 DBAPI 建立事务状态）。

多绑定或未绑定的`Session`对象中的歧义可以通过任何可选关键字参数解决。最终，使用`get_bind()`方法进行解析。

参数：

+   `bind_arguments` – 绑定参数字典。可能包括“mapper”、“bind”、“clause”等传递给`Session.get_bind()`的其他自定义参数。

+   `execution_options` –

    一个执行选项字典，将仅在首次获取连接时传递给`Connection.execution_options()`。如果连接已经存在于`Session`中，则会发出警告并忽略参数。

    另请参见

    设置事务隔离级别 / DBAPI AUTOCOMMIT

```py
method delete(instance: object) → None
```

将实例标记为已删除。

假定传递的对象在调用方法后将保持在 persistent 或 detached 状态；在下次刷新之前，对象将保持在 persistent 状态。在此期间，对象还将是`Session.deleted`集合的成员。

下次刷新时，对象将转移到 deleted 状态，表示在当前事务中为其行发出了`DELETE`语句。当事务成功提交时，已删除的对象将移至 detached 状态，并不再存在于此`Session`中。

另请参见

删除 - 在使用会话的基础知识

```py
attribute deleted
```

在此`Session`中标记为‘deleted’的所有实例的集合

```py
attribute dirty
```

所有被视为脏数据的持久实例的集合。

例如：

```py
some_mapped_object in session.dirty
```

当实例被修改但未被删除时，被视为脏数据。

请注意，这种‘脏’计算是‘乐观的’；大多数属性设置或集合修改操作都会将实例标记为‘脏’并将其放入此集合中，即使属性的值没有净变化也是如此。在刷新时，将每个属性的值与其先前保存的值进行比较，如果没有净变化，则不会发生 SQL 操作（这是一种更昂贵的操作，因此只在刷新时执行）。

要检查实例的属性是否具有可执行的净变化，请使用`Session.is_modified()`方法。

```py
method enable_relationship_loading(obj: object) → None
```

将对象与此`Session`关联以加载相关对象。

警告

`enable_relationship_loading()`存在于服务于特殊用例，并不建议一般使用。

通过`relationship()`映射的属性访问将尝试使用此`Session`作为连接的源来从数据库加载值。这些值将根据此对象上存在的外键和主键值进行加载 - 如果不存在，则这些关系将不可用。

对象将附加到此会话，但将**不会**参与任何持久化操作；对于几乎所有目的，其状态仍将保持“瞬态”或“分离”，除了关系加载的情况。

还请注意，反向引用通常不会按预期工作。在目标对象上修改与关系绑定的属性可能不会触发反向引用事件，如果有效值已从保存外键值中加载，则是如此。

`Session.enable_relationship_loading()`方法类似于`relationship()`上的`load_on_pending`标志。与该标志不同，`Session.enable_relationship_loading()`允许对象保持瞬态状态，同时仍然能够加载相关项目。

要使一个临时对象与`Session`相关联，可以通过`Session.enable_relationship_loading()`将其添加到`Session`中。如果该对象代表数据库中现有的标识，则应使用`Session.merge()`进行合并。

`Session.enable_relationship_loading()`在正常使用 ORM 时不会改善行为 - 对象引用应该在对象级别构建，而不是在外键级别构建，以便它们在 flush()继续之前以普通方式存在。此方法不适用于一般用途。

另请参见

`relationship.load_on_pending` - 此标志允许在待处理项目上对多对一关系进行逐个加载。

`make_transient_to_detached()` - 允许将对象添加到`Session`中而不发出 SQL，然后在访问时取消过期属性。

```py
method execute(statement: Executable, params: _CoreAnyExecuteParams | None = None, *, execution_options: OrmExecuteOptionsParameter = {}, bind_arguments: _BindArguments | None = None, _parent_execute_state: Any | None = None, _add_event: Any | None = None) → Result[Any]
```

执行 SQL 表达式构造。

返回代表语句执行结果的`Result`对象。

例如：

```py
from sqlalchemy import select
result = session.execute(
    select(User).where(User.id == 5)
)
```

`Session.execute()`的 API 契约类似于`Connection.execute()`的契约，是`Connection`的 2.0 风格版本。

从版本 1.4 开始更改：当使用 2.0 风格的 ORM 用法时，`Session.execute()`方法现在是 ORM 语句执行的主要点。

参数：

+   `statement` – 可执行语句（即`Executable`表达式，如`select()`）。

+   `params` – 可选字典，或包含绑定参数值的字典列表。如果是单个字典，则执行单行；如果是字典列表，则将调用“executemany”。每个字典中的键必须对应于语句中存在的参数名称。

+   `execution_options` –

    可选的执行选项字典，将与语句执行关联起来。此字典可以提供`Connection.execution_options()`所接受的选项子集，也可以提供仅在 ORM 上下文中理解的附加选项。

    另请参见

    ORM 执行选项 - ORM 特定的执行选项

+   `bind_arguments` – 附加参数字典，用于确定绑定。可能包括“mapper”、“bind”或其他自定义参数。此字典的内容传递给`Session.get_bind()`方法。

返回：

一个`Result`对象。

```py
method expire(instance: object, attribute_names: Iterable[str] | None = None) → None
```

使实例上的属性过期。

将实例的属性标记为过时。下次访问过期属性时，将向`Session`对象的当前事务上下文发出查询，以便为给定实例加载所有过期属性。请注意，高度隔离的事务将返回与之前在同一事务中读取的相同值，而不管该事务之外的数据库状态如何更改。

要同时使`Session`中的所有对象过期，请使用`Session.expire_all()`。

`Session`对象的默认行为是在调用`Session.rollback()`或`Session.commit()`方法时使所有状态过期，以便为新事务加载新状态。因此，仅在当前事务中发出非 ORM SQL 语句的情况下调用`Session.expire()`才有意义。

参数：

+   `instance` – 要刷新的实例。

+   `attribute_names` – 可选的字符串属性名称列表，指示要过期的属性子集。

另请参见

刷新/过期 - 介绍性材料

`Session.expire()`

`Session.refresh()`

`Query.populate_existing()`

```py
method expire_all() → None
```

使此会话中的所有持久实例过期。

当对持久实例上的任何属性进行下一次访问时，将使用`Session`对象的当前事务上下文发出查询，以加载给定实例的所有过期属性。请注意，高度隔离的事务将返回与之前在同一事务中读取的相同值，而不管事务外数据库状态的更改如何。

要使单个对象及其上的单个属性过期，请使用`Session.expire()`。

当调用`Session.rollback()`或`Session.commit()`方法时，`Session`对象的默认行为是使所有状态过期，以便为新事务加载新状态。因此，通常不需要调用`Session.expire_all()`，假设事务是隔离的。

另请参阅

刷新/过期 - 简介材料

`Session.expire()`

`Session.refresh()`

`Query.populate_existing()`

```py
method expunge(instance: object) → None
```

从此`Session`中删除实例。

这将释放所有对实例的内部引用。级联将根据*expunge*级联规则应用。

```py
method expunge_all() → None
```

从此`Session`中删除所有对象实例。

这相当于在此`Session`中调用`expunge(obj)`以将所有对象从中清除。

```py
method flush(objects: Sequence[Any] | None = None) → None
```

将所有对象更改刷新到数据库。

将所有待处理的对象创建、删除和修改操作写入数据库，作为 INSERTs、DELETEs、UPDATEs 等。操作会自动按照会话的工作单元依赖解析器的顺序进行排序。

数据库操作将在当前事务上下文中发出，并且不会影响事务的状态，除非发生错误，在这种情况下将回滚整个事务。您可以在事务中随意刷新()，以将更改从 Python 移动到数据库的事务缓冲区。

参数：

**objects** –

可选；仅将刷新操作限制为仅操作给定集合中的元素。

此功能适用于极其狭窄的一组用例，其中特定对象可能需要在完全执行 flush()之前操作。不适用于一般用途。

```py
method get(entity: _EntityBindKey[_O], ident: _PKIdentityArgument, *, options: Sequence[ORMOption] | None = None, populate_existing: bool = False, with_for_update: ForUpdateParameter = None, identity_token: Any | None = None, execution_options: OrmExecuteOptionsParameter = {}, bind_arguments: _BindArguments | None = None) → _O | None
```

根据给定的主键标识符返回一个实例，如果找不到则返回`None`。

示例：

```py
my_user = session.get(User, 5)

some_object = session.get(VersionedFoo, (5, 10))

some_object = session.get(
    VersionedFoo,
    {"id": 5, "version_id": 10}
)
```

版本 1.4 中的新功能：添加了`Session.get()`，该方法已从现已过时的`Query.get()`方法中移除。

`Session.get()` 是特殊的，因为它直接提供对`Session`的标识映射的访问。如果给定的主键标识符存在于本地标识映射中，则直接从该集合返回对象，并且不会发出 SQL，除非对象已被标记为完全过期。如果不存在，则执行 SELECT 来定位对象。

`Session.get()` 还会检查对象是否存在于标识映射中并标记为过期 - 会发出 SELECT 来刷新对象，并确保行仍然存在。如果不是，则引发 `ObjectDeletedError`。

参数：

+   `entity` – 表示要加载的实体类型的映射类或`Mapper`。

+   `ident` –

    代表主键的标量、元组或字典。对于复合（例如多列）主键，应传递元组或字典。

    对于单列主键，标量调用形式通常是最快捷的。如果行的主键值是“5”，则调用如下：

    ```py
    my_object = session.get(SomeClass, 5)
    ```

    元组形式通常按照它们与映射的`Table`对象的主键列对应的顺序排列，或者如果使用了`Mapper.primary_key`配置参数，则按照该参数的使用顺序排列。例如，如果一行的主键由整数数字“5, 10”表示，则调用将如下所示：

    ```py
    my_object = session.get(SomeClass, (5, 10))
    ```

    字典形式应该将每个主键元素对应的映射属性名称作为键。如果映射类具有存储对象主键值的属性`id`、`version_id`，则调用将如下所示：

    ```py
    my_object = session.get(SomeClass, {"id": 5, "version_id": 10})
    ```

+   `options` – 可选的加载器选项序列，将应用于查询（如果发出了查询）。

+   `populate_existing` – 导致该方法无条件地发出 SQL 查询并使用新加载的数据刷新对象，无论对象是否已存在。

+   `with_for_update` – 可选的布尔值`True`表示应该使用 FOR UPDATE，或者可以是一个包含标志的字典，表示一个更具体的用于 SELECT 的 FOR UPDATE 标志集合；标志应该与`Query.with_for_update()`的参数匹配。覆盖`Session.refresh.lockmode`参数。

+   `execution_options` –

    可选的执行选项字典，如果发出了查询执行，则将与之相关联。此字典可以提供由`Connection.execution_options()`接受的选项的子集，并且还可以提供仅在 ORM 上下文中理解的其他选项。

    新版本中的 1.4.29。

    另请参见

    ORM 执行选项 - ORM 特定的执行选项

+   `bind_arguments` –

    用于确定绑定的其他参数的字典。可能包括“mapper”，“bind”或其他自定义参数。此字典的内容将传递给`Session.get_bind()`方法。

返回：

对象实例，或者`None`。

```py
method get_bind(mapper: _EntityBindKey[_O] | None = None, *, clause: ClauseElement | None = None, bind: _SessionBind | None = None, _sa_skip_events: bool | None = None, _sa_skip_for_implicit_returning: bool = False, **kw: Any) → Engine | Connection
```

返回此`Session`所绑定到的“绑定”。

“绑定”通常是`Engine`的一个实例，除非`Session`已经被明确地直接绑定到一个`Connection`的情况除外。

对于多次绑定或未绑定的`Session`，使用`mapper`或`clause`参数来确定返回的适当绑定。

请注意，当通过 ORM 操作（例如`Session.query()`、`Session.flush()`调用等）调用`Session.get_bind()`时，“mapper”参数通常存在。

解析的顺序是：

1.  如果提供了 mapper 并且`Session.binds`存在，则根据使用的 mapper，然后根据使用的 mapped 类，然后根据 mapped 类的`__mro__`中存在的任何基类，从更具体的超类到更一般的超类来定位一个绑定。

1.  如果给定了子句并且存在`Session.binds`，则基于`Session.binds`中存在的给定子句中找到的`Table`对象定位一个绑定。

1.  如果存在`Session.binds`，则返回该绑定。

1.  如果给定了子句，则尝试返回与最终与子句关联的`MetaData`相关联的绑定。

1.  如果给定了映射器，则尝试返回与最终与映射器映射到的`Table`或其他可选择的绑定相关联的`MetaData`。

1.  找不到绑定时，将引发`UnboundExecutionError`。

注意，`Session.get_bind()` 方法可以在用户定义的`Session`子类上被重写，以提供任何类型的绑定解析方案。请参见自定义垂直分区中的示例。

参数：

+   `mapper` – 可选的映射类或对应的`Mapper`实例。绑定可以首先从与此`Session`关联的“binds”映射中派生`Mapper`，其次通过查看与`Mapper`映射到的`Table`相关联的`MetaData`来获取。

+   `clause` – 一个`ClauseElement`（即`select()`，`text()`等）。如果`mapper`参数不存在或无法生成绑定，则将搜索给定表达式构造的绑定元素，通常是与绑定的`MetaData`相关联的`Table`。

另请参阅

分区策略（例如每个 Session 多个数据库后���）

`Session.binds` 

`Session.bind_mapper()`

`Session.bind_table()`

```py
method get_nested_transaction() → SessionTransaction | None
```

返回当前正在进行的嵌套事务，如果有的话。

版本 1.4 中的新功能。

```py
method get_one(entity: _EntityBindKey[_O], ident: _PKIdentityArgument, *, options: Sequence[ORMOption] | None = None, populate_existing: bool = False, with_for_update: ForUpdateParameter = None, identity_token: Any | None = None, execution_options: OrmExecuteOptionsParameter = {}, bind_arguments: _BindArguments | None = None) → _O
```

根据给定的主键标识符返回一个实例，如果未找到则引发异常。

如果查询未选择任何行，则引发`sqlalchemy.orm.exc.NoResultFound`。

有关参数的详细文档，请参见方法`Session.get()`。

版本 2.0.22 中的新功能。

返回：

对象实例。

另请参见

`Session.get()` - 相当的方法，而不是

如果未找到具有提供的主键的行，则返回`None`。

```py
method get_transaction() → SessionTransaction | None
```

返回当前正在进行的根事务，如果有的话。

版本 1.4 中的新功能。

```py
classmethod identity_key(class_: Type[Any] | None = None, ident: Any | Tuple[Any, ...] = None, *, instance: Any | None = None, row: Row[Any] | RowMapping | None = None, identity_token: Any | None = None) → _IdentityKeyType[Any]
```

*继承自* `sqlalchemy.orm.session._SessionClassMethods.identity_key` *方法的* `sqlalchemy.orm.session._SessionClassMethods`

返回一个标识键。

这是`identity_key()`的别名。

```py
attribute identity_map: IdentityMap
```

对象标识与对象本身的映射。

通过遍历`Session.identity_map.values()`可以访问当前会话中的所有持久对象的完整集合（即具有行标识的对象）。

另请参见

`identity_key()` - 用于生成此字典中使用的键的辅助函数。

```py
method in_nested_transaction() → bool
```

如果此`Session`已开始嵌套事务（例如 SAVEPOINT），则返回 True。

版本 1.4 中的新功能。

```py
method in_transaction() → bool
```

如果此`Session`已开始事务，则返回 True。

版本 1.4 中的新功能。

另请参见

`Session.is_active`

```py
attribute info
```

用户可修改的字典。

此字典的初始值可以使用`Session`构造函数或`sessionmaker`构造函数或工厂方法的`info`参数进行填充。此处的字典始终局限于此`Session`，可以独立于所有其他`Session`对象进行修改。

```py
method invalidate() → None
```

使用连接失效关闭此会话。

这是`Session.close()`的变体，还将确保在当前用于事务的每个`Connection`对象上调用`Connection.invalidate()`方法（通常只有一个连接，除非`Session`与多个引擎一起使用）。

当已知数据库处于不再安全使用连接的状态时，可以调用此方法。

下面说明了使用[gevent](https://www.gevent.org/)时可能出现的`Timeout`异常的情况，这可能意味着应丢弃底层连接：

```py
import gevent

try:
    sess = Session()
    sess.add(User())
    sess.commit()
except gevent.Timeout:
    sess.invalidate()
    raise
except:
    sess.rollback()
    raise
```

该方法还执行`Session.close()`执行的所有操作，包括清除所有 ORM 对象。

```py
attribute is_active
```

如果此`Session`不处于“部分回滚”状态，则为 True。

在版本 1.4 中更改：`Session` 不再立即开始新事务，因此当首次实例化`Session`时，此属性将为 False。

“部分回滚”状态通常表示`Session`的刷新过程失败，并且必须发出`Session.rollback()`方法以完全回滚事务。

如果此`Session`根本不处于事务中，则在首次使用时`Session`将自动开始，因此在这种情况下`Session.is_active`将返回 True。

否则，如果此`Session`在事务内，并且该事务尚未在内部回滚，则`Session.is_active`也将返回 True。

另请参见

“由于在刷新期间发生的先前异常，此会话的事务已回滚。”（或类似内容）

`Session.in_transaction()`

```py
method is_modified(instance: object, include_collections: bool = True) → bool
```

返回 `True` 如果给定的实例具有本地修改的属性。

此方法检索实例上每个受监视属性的历史记录，并将当前值与先前提交的值进行比较（如果有的话）。

实际上，这是一种更昂贵和准确的版本，用于检查给定实例是否存在于 `Session.dirty` 集合中；对每个属性的净“脏”状态进行了完整测试。

例如：

```py
return session.is_modified(someobject)
```

这种方法有一些注意事项：

+   当测试使用此方法时，存在于 `Session.dirty` 集合中的实例可能会报告 `False`。这是因为对象可能已通过属性变化接收到更改事件，从而将其放置在 `Session.dirty` 中，但最终状态与从数据库加载的状态相同，在这里没有净变化。

+   当新值被应用时，标量属性可能没有记录先前设置的值，如果属性在接收到新值时没有被加载或已过期，则假定属性发生了变化，即使最终与其数据库值相比没有净变化，在大多数情况下，当设置事件发生时，SQLAlchemy 不需要“旧”值，因此，如果旧值不存在，则跳过发出 SQL 调用的开销，这是基于这样一个假设：通常需要更新标量值，并且在那些极少数情况下，其中不需要，平均而言，这比发出防御性 SELECT 更便宜。

    只有当属性容器的 `active_history` 标志设置为 `True` 时，才无条件地在设置时获取“旧”值。此标志通常设置为主键属性和非简单一对多的标量对象引用。要为任意映射列设置此标志，请使用 `column_property()` 中的 `active_history` 参数。

参数：

+   `instance` – 要测试是否存在待处理更改的映射实例。

+   `include_collections` – 指示是否应该在操作中包含多值集合。将其设置为 `False` 是一种检测仅基于本地列的属性（即标量列或一对多外键），这将导致此实例在刷新时进行更新。

```py
method merge(instance: _O, *, load: bool = True, options: Sequence[ORMOption] | None = None) → _O
```

将给定实例的状态复制到此 `Session` 中对应的实例。

`Session.merge()` 检查源实例的主键属性，并尝试将其与会话中具有相同主键的实例进行协调。如果在本地找不到，则尝试根据主键从数据库加载对象，如果找不到，则创建一个新实例。然后将源实例上的每个属性的状态复制到目标实例。然后，该方法返回结果目标实例；原始源实例保持不变，并且如果尚未与`Session` 关联，则不与其关联。

此操作级联到相关实例，如果关联使用 `cascade="merge"` 进行映射。

有关合并的详细讨论，请参见合并。

参数：

+   `instance` – 要合并的实例。

+   `load` –

    布尔值，当为 False 时，`merge()` 切换到“高性能”模式，导致它放弃发出历史事件以及所有数据库访问。此标志用于将对象图传输到从第二级缓存中的`Session` 中，或者将刚加载的对象传输到由工作线程或进程拥有的`Session` 中，而无需重新查询数据库。

    `load=False` 的使用情况添加了一个警告，即给定对象必须处于“干净”的状态，即没有任何待冲刷的更改 - 即使传入的对象是从任何一个`Session` 分离出来的。这样，当合并操作填充本地属性并级联到相关对象和集合时，值可以“按原样”放置到目标对象上，而不会生成任何历史或属性事件，并且无需将传入的数据与可能未加载的任何现有相关对象或集合进行协调。`load=False` 生成的结果对象始终为“干净”，因此只有给定对象也应为“干净”，否则这表明方法的错误使用。

+   `options` –

    可选的加载器选项序列，在合并操作从数据库加载对象的现有版本时将应用于`Session.get()` 方法。

    新版本 1.4.24 中新增。

另请参阅

`make_transient_to_detached()` - 提供了将单个对象“合并”到`Session` 的另一种方法。

```py
attribute new
```

在此`Session`中标记为“新”的所有实例的集合。

```py
attribute no_autoflush
```

返回一个禁用自动冲刷的上下文管理器。

例如：

```py
with session.no_autoflush:

    some_object = SomeClass()
    session.add(some_object)
    # won't autoflush
    some_object.related_thing = session.query(SomeRelated).first()
```

在 `with:` 块内进行的操作不会受到查询访问时发生的 flush 的影响。这在初始化一系列涉及现有数据库查询的对象时很有用，此时尚未完成的对象不应立即被 flush。

```py
classmethod object_session(instance: object) → Session | None
```

*继承自* `sqlalchemy.orm.session._SessionClassMethods` *的* `sqlalchemy.orm.session._SessionClassMethods.object_session` *方法*

返回一个对象所属的`Session`。

这是 `object_session()` 方法的别名。

```py
method prepare() → None
```

准备进行中的当前事务以进行两阶段提交。

如果没有进行中的事务，则此方法会引发一个`InvalidRequestError`。

仅两阶段会话的根事务才能被准备。如果当前事务不是这样的事务，则会引发 `InvalidRequestError`。

```py
method query(*entities: _ColumnsClauseArgument[Any], **kwargs: Any) → Query[Any]
```

返回一个与此 `Session` 对应的新 `Query` 对象。

请注意，`Query` 对象在 SQLAlchemy 2.0 中已经是遗留的；现在使用 `select()` 构造 ORM 查询。

另请参阅

SQLAlchemy 统一教程

ORM 查询指南

旧版查询 API - 旧版 API 文档

```py
method refresh(instance: object, attribute_names: Iterable[str] | None = None, with_for_update: ForUpdateParameter = None) → None
```

过期并刷新给定实例上的属性。

选定的属性将首先被过期，就像使用 `Session.expire()` 时一样；然后会向数据库发出 SELECT 语句，以当前事务中可用的当前值刷新基于列的属性。

如果对象已经急加载，那么以`relationship()`为导向的属性也将立即加载，使用最初加载时的相同的急加载策略。

1.4 版本新增：- `Session.refresh()` 方法还可以立即刷新急加载的属性。

`relationship()`定向属性通常使用`select`（或“lazy”）加载器策略将在属性名称集合中明确命名时也会加载**，使用`immediate`加载器策略发出用于属性的 SELECT 语句。如果惰性加载的关系未在`Session.refresh.attribute_names`中命名，则它们将保持为“惰性加载”属性，并且不会被隐式刷新。

从版本 2.0.4 开始更改：`Session.refresh()`方法现在将刷新在`Session.refresh.attribute_names`集合中明确命名的惰性加载的`relationship()`定向属性。

提示

虽然`Session.refresh()`方法能够刷新列和关系定向属性，但其主要重点是刷新单个实例上的本地列定向属性。对于更开放的“刷新”功能，包括能够同时刷新多个对象的属性并明确控制关系加载器策略，请改用填充现有功能。

请注意，高度隔离的事务将返回与先前在该事务中读取的相同值，而不考虑该事务之外数据库状态的更改。通常只在事务开始时数据库行尚未被访问时刷新属性才有意义。

参数：

+   `attribute_names` – 可选。一个包含字符串属性名称的可迭代集合，指示要刷新的属性子集。

+   `with_for_update` – 可选布尔值`True`表示应使用 FOR UPDATE，或者可以是一个包含标志的字典，指示用于 SELECT 的更具体的 FOR UPDATE 标志集；标志应与`Query.with_for_update()`参数匹配。取代`Session.refresh.lockmode`参数。

另请参阅

刷新/过期 - 入门材料

`Session.expire()`

`Session.expire_all()`

填充现有对象 - 允许任何 ORM 查询刷新对象，就像它们通常加载的那样。

```py
method reset() → None
```

关闭事务资源和此`Session`使用的 ORM 对象，将会重置会话到其初始状态。

此方法提供了与`Session.close()`方法历史上提供的相同“仅重置”行为，其中`Session`的状态被重置，就像对象是全新的一样，并且可以再次使用。然后，此方法可能对将`Session.close_resets_only`设置为`False`的`Session`对象有用，以便仍然可以使用“仅重置”行为。

新版本 2.0.22 中新增。

另请参阅

关闭操作 - 关于`Session.close()`和`Session.reset()`语义的详细信息。

`Session.close()` - 当参数`Session.close_resets_only`设置为`False`时，类似的方法还会阻止重复使用`Session`。

```py
method rollback() → None
```

回滚当前进行中的事务。

如果没有进行中的事务，则此方法是一个传递方法。

该方法始终回滚最顶层的数据库事务，丢弃可能正在进行的任何嵌套事务。

另请参阅

回滚操作

事务管理

```py
method scalar(statement: Executable, params: _CoreSingleExecuteParams | None = None, *, execution_options: OrmExecuteOptionsParameter = {}, bind_arguments: _BindArguments | None = None, **kw: Any) → Any
```

执行语句并返回标量结果。

使用和参数与`Session.execute()`相同；返回结果是一个标量 Python 值。

```py
method scalars(statement: Executable, params: _CoreAnyExecuteParams | None = None, *, execution_options: OrmExecuteOptionsParameter = {}, bind_arguments: _BindArguments | None = None, **kw: Any) → ScalarResult[Any]
```

执行语句并将结果作为标量返回。

使用和参数与`Session.execute()`相同；返回结果是一个过滤对象`ScalarResult`，它将返回单个元素而不是`Row`对象。

返回：

一个`ScalarResult`对象

新版本 1.4.24 中新增：添加`Session.scalars()`

新版本 1.4.26 中新增：添加`scoped_session.scalars()`

另请参阅

选择 ORM 实体 - 对比 `Session.execute()` 和 `Session.scalars()` 的行为

```py
class sqlalchemy.orm.SessionTransaction
```

一个 `Session` 级别的事务。

`SessionTransaction` 是从 `Session.begin()` 和 `Session.begin_nested()` 方法中生成的。它在现代用法中主要是一个为会话事务提供上下文管理器的内部对象。

与 `SessionTransaction` 交互的文档位于：管理事务。

在版本 1.4 中更改：与 `SessionTransaction` 对象直接交互的作用域和 API 方法已经简化。

另请参阅

管理事务

`Session.begin()`

`Session.begin_nested()`

`Session.rollback()`

`Session.commit()`

`Session.in_transaction()`

`Session.in_nested_transaction()`

`Session.get_transaction()`

`Session.get_nested_transaction()`

**成员**

嵌套, origin, 父级

**类签名**

类 `sqlalchemy.orm.SessionTransaction` (`sqlalchemy.orm.state_changes._StateChange`, `sqlalchemy.engine.util.TransactionalContext`)

```py
attribute nested: bool = False
```

指示此事务是否为嵌套或 SAVEPOINT 事务。

当 `SessionTransaction.nested` 为 True 时，预期 `SessionTransaction.parent` 也将出现，并链接到封闭的 `SessionTransaction`。

另请参见

`SessionTransaction.origin`

```py
attribute origin: SessionTransactionOrigin
```

此 `SessionTransaction` 的来源。

指的是一个 `SessionTransactionOrigin` 实例，它是一个枚举，指示导致构建此 `SessionTransaction` 的源事件。

新版本 2.0 中新增。

```py
attribute parent
```

此 `SessionTransaction` 的父 `SessionTransaction`。

如果此属性为 `None`，表示此 `SessionTransaction` 处于堆栈顶部，并对应于真实的“COMMIT”/“ROLLBACK”块。 如果非 `None`，则这是一个“子事务”（刷新过程中使用的内部标记对象）或“嵌套”/保存点事��。 如果 `SessionTransaction.nested` 属性为 `True`，则这是一个保存点，如果为 `False`，则表示这是一个子事务。

```py
class sqlalchemy.orm.SessionTransactionOrigin
```

指示 `SessionTransaction` 的来源。

此枚举存在于任何 `SessionTransaction` 对象的 `SessionTransaction.origin` 属性上。

新版本 2.0 中新增。

**成员**

AUTOBEGIN, BEGIN, BEGIN_NESTED, SUBTRANSACTION

**类签名**

类 `sqlalchemy.orm.SessionTransactionOrigin` (`enum.Enum`)

```py
attribute AUTOBEGIN = 0
```

事务是由 autobegin 启动的。

```py
attribute BEGIN = 1
```

事务是通过调用 `Session.begin()` 开始的。

```py
attribute BEGIN_NESTED = 2
```

事务是通过 `Session.begin_nested()` 开始的。

```py
attribute SUBTRANSACTION = 3
```

事务是一个内部的“子事务”。

## 会话工具

| 对象名称 | 描述 |
| --- | --- |
| close_all_sessions() | 关闭内存中的所有会话。 |
| make_transient(instance) | 更改给定实例的状态，使其成为瞬态。 |
| make_transient_to_detached(instance) | 使给定的瞬态实例分离。 |
| object_session(instance) | 返回给定实例所属的`Session`。 |
| was_deleted(object_) | 如果给定对象在会话刷新中被删除，则返回 True。 |

```py
function sqlalchemy.orm.close_all_sessions() → None
```

关闭内存中的所有会话。

此函数查询所有`Session`对象的全局注册表，并调用`Session.close()`关闭它们，将它们重置为干净状态。

此函数不适用于一般用途，但可能对拆卸方案中的测试套件有用。

版本 1.3 中的新功能。

```py
function sqlalchemy.orm.make_transient(instance: object) → None
```

更改给定实例的状态，使其为瞬态。

注意

`make_transient()`是仅适用于高级用例的特殊函数。

假定给定的映射实例处于持久或分离状态。该函数将删除其与任何`Session`的关联以及其`InstanceState.identity`。其效果是对象将表现得像是新构造的，但保留在调用时加载的任何属性/集合值。如果此对象曾因使用`Session.delete()`而被删除，则`InstanceState.deleted`标志也将被重置。

警告

`make_transient()` **不** “取消过期”或以其他方式急切加载在调用函数时尚未加载的 ORM 映射属性。这包括：

+   通过`Session.expire()`过期

+   作为提交会话事务的自然效果而过期，例如`Session.commit()`

+   通常是延迟加载，但目前尚未加载

+   被“延迟加载”（参见限制加载的列与列延迟）且尚未加载

+   在加载此对象的查询中不存在，例如在联接表继承和其他场景中常见的情况。

调用 `make_transient()` 后，像上面这样未加载的属性通常在访问时将解析为值 `None`，或者对于集合定向属性为一个空集合。由于对象是瞬态的，并且未关联任何数据库标识，因此它将不再检索这些值。

另请参阅

`make_transient_to_detached()`

```py
function sqlalchemy.orm.make_transient_to_detached(instance: object) → None
```

使给定的瞬态实例 分离。

注意

`make_transient_to_detached()` 是一个仅限于高级用例的特殊情况函数。

在给定实例上的所有属性历史都将被重置，就像实例是从查询中新加载的一样。丢失的属性将被标记为过期。对象的主键属性将被制作为实例的“键”，这些主键属性是必需的。

然后对象可以被添加到一个会话中，或者可能与 load=False 标志合并，此时它将看起来像是以这种方式加载，而不发出 SQL。

这是一个特殊的用例函数，与对 `Session.merge()` 的正常调用不同，因为可以制造给定的持久状态而不进行任何 SQL 调用。

另请参阅

`make_transient()`

`Session.enable_relationship_loading()`

```py
function sqlalchemy.orm.object_session(instance: object) → Session | None
```

返回给定实例所属的 `Session`。

这与 `InstanceState.session` 访问器本质上是相同的。有关详细信息，请参阅该属性。

```py
function sqlalchemy.orm.util.was_deleted(object_: object) → bool
```

返回 True，如果给定的对象在会话刷新内被删除。

不管对象是持久的还是分离的，都是如此。

另请参阅

`InstanceState.was_deleted`

## 属性和状态管理实用程序

这些函数由 SQLAlchemy 属性调制 API 提供，以提供一个详细的接口来处理实例、属性值和历史。当构造事件监听函数时，其中一些函数是有用的，比如 ORM 事件 中描述的那些函数。

| 对象名称 | 描述 |
| --- | --- |
| del_attribute(instance, key) | 删除属性的值，触发历史事件。 |
| flag_dirty(instance) | 标记一个实例为“脏”状态，而不需要特定的属性名称。 |
| flag_modified(instance, key) | 将实例上的属性标记为“已修改”。 |
| get_attribute(instance, key) | 获取属性的值，触发任何所需的可调用函数。 |
| get_history(obj, key[, passive]) | 返回给定对象和属性键的`History`记录。 |
| History | 已添加、未更改和已删除值的 3 元组，表示在受监控属性上发生的更改。 |
| init_collection(obj, key) | 初始化一个集合属性并返回集合适配器。 |
| instance_state | 返回给定映射对象的`InstanceState`。 |
| is_instrumented(instance, key) | 如果给定实例上的给定属性由 attributes 包进行了插装，则返回 True。 |
| object_state(instance) | 给定一个对象，返回与该对象关联的`InstanceState`。 |
| set_attribute(instance, key, value[, initiator]) | 设置属性的值，触发历史事件。 |
| set_committed_value(instance, key, value) | 设置属性的值，不触发历史事件。 |

```py
function sqlalchemy.orm.util.object_state(instance: _T) → InstanceState[_T]
```

给定一个对象，返回与该对象关联的`InstanceState`。

如果未配置映射，则引发`sqlalchemy.orm.exc.UnmappedInstanceError`。

相同的功能可以通过`inspect()`函数获得，如下所示：

```py
inspect(instance)
```

使用检查系统将在实例不属于映射的情况下引发`sqlalchemy.exc.NoInspectionAvailable`。

```py
function sqlalchemy.orm.attributes.del_attribute(instance: object, key: str) → None
```

删除属性的值，触发历史事件。

无论直接应用于类的插装如何，都可以使用此函数，即不需要描述符。自定义属性管理方案将需要使用此方法来建立由 SQLAlchemy 理解的属性状态。

```py
function sqlalchemy.orm.attributes.get_attribute(instance: object, key: str) → Any
```

获取属性的值，触发任何所需的可调用函数。

无论直接应用于类的仪器，都可以使用此功能，即不需要描述符。自定义属性管理方案将需要使用此方法来使用 SQLAlchemy 所理解的属性状态。

```py
function sqlalchemy.orm.attributes.get_history(obj: object, key: str, passive: PassiveFlag = symbol('PASSIVE_OFF')) → History
```

返回给定对象和属性键的`History`记录。

这是给定属性的**预刷新**历史记录，每次`Session`刷新更改到当前数据库事务时都会重置它。

注意

首选使用`AttributeState.history`和`AttributeState.load_history()`访问器来检索实例属性的`History`。

参数：

+   `obj` – 一个其类由属性包仪器化的对象。

+   `key` – 字符串属性名称。

+   `passive` – 如果值尚不存在，则指示属性的加载行为。这是一个位标志属性，默认为`PASSIVE_OFF`，表示应发出所有必要的 SQL。

另请参见

`AttributeState.history`

`AttributeState.load_history()` - 如果值在本地不存在，则使用加载器可调用检索历史。

```py
function sqlalchemy.orm.attributes.init_collection(obj: object, key: str) → CollectionAdapter
```

初始化一个集合属性并返回集合适配器。

此函数用于为先前未加载的属性提供直接访问集合内部。例如：

```py
collection_adapter = init_collection(someobject, 'elements')
for elem in values:
    collection_adapter.append_without_event(elem)
```

对于执行上述操作的更简单方法，请参见`set_committed_value()`。

参数：

+   `obj` – 一个映射对象

+   `key` – 集合所在的字符串属性名称。

```py
function sqlalchemy.orm.attributes.flag_modified(instance: object, key: str) → None
```

将实例上的属性标记为“已修改”。

这会在实例上设置“已修改”标志，并为给定属性建立一个无条件的更改事件。属性必须具有值，否则会引发`InvalidRequestError`。

要标记一个对象为“脏”，而不引用任何特定属性，以便在刷新时考虑到它，使用`flag_dirty()`调用。

另请参见

`flag_dirty()`

```py
function sqlalchemy.orm.attributes.flag_dirty(instance: object) → None
```

将实例标记为“脏”，而不提及任何特定属性。

这是一个特殊操作，允许对象通过刷新流程进行拦截，例如 `SessionEvents.before_flush()`。请注意，对于没有更改的对象，在刷新过程中不会发出任何 SQL，即使通过此方法标记为脏。但是，`SessionEvents.before_flush()` 处理程序将能够在 `Session.dirty` 集合中看到对象，并可能对其进行更改，然后这些更改将包含在发出的 SQL 中。

自 1.2 版新功能。

另请参阅

`flag_modified()`

```py
function sqlalchemy.orm.attributes.instance_state()
```

返回给定映射对象的 `InstanceState`。

此函数是 `object_state()` 的内部版本。此处推荐使用 `object_state()` 和/或 `inspect()` 函数，因为它们会在给定对象未映射时各自发出信息丰富的异常。

```py
function sqlalchemy.orm.instrumentation.is_instrumented(instance, key)
```

如果给定实例上的给定属性由 attributes 包进行仪器化，则返回 True。

无论直接应用于类的仪器如何，都可以使用此函数，即不需要描述符。

```py
function sqlalchemy.orm.attributes.set_attribute(instance: object, key: str, value: Any, initiator: AttributeEventToken | None = None) → None
```

设置属性的值，并触发历史事件。

无论直接应用于类的仪器如何，都可以使用此函数，即不需要描述符。自定义属性管理方案将需要使用此方法来建立由 SQLAlchemy 理解的属性状态。

参数：

+   `instance` – 将要修改的对象

+   `key` – 属性的字符串名称

+   `value` – 要分配的值

+   `initiator` –

    一个 `Event` 的实例，可能已从前一个事件侦听器传播。当在现有事件侦听器函数中使用 `set_attribute()` 函数时，其中提供了一个 `Event` 对象；该对象可用于跟踪事件链的来源。

    自 1.2.3 版新功能。

```py
function sqlalchemy.orm.attributes.set_committed_value(instance, key, value)
```

设置没有历史事件的属性值。

取消任何先前存在的历史。值应为标量值（对于保存标量的属性）或可迭代对象（对于任何保存集合的属性）。

当惰性加载器触发并从数据库加载附加数据时，使用的是相同的基础方法。特别是，该方法可被应用代码使用，通过单独的查询加载了额外的属性或集合，然后可以将其附加到实例上，就像它是其原始加载状态的一部分一样。

```py
class sqlalchemy.orm.attributes.History
```

一个由添加、未更改和已删除值组成的 3 元组，表示在一个被检测的属性上发生的变化。

获取对象上特定属性的`History`对象的最简单方法是使用`inspect()`函数：

```py
from sqlalchemy import inspect

hist = inspect(myobject).attrs.myattribute.history
```

每个元组成员都是一个可迭代序列：

+   `added` - 添加到属性的项目的集合（第一个元组元素）。

+   `unchanged` - 在属性上没有更改的项目的集合（第二个元组元素）。

+   `deleted` - 从属性中删除的项目的集合（第三个元组元素）。

**成员**

added, deleted, empty(), has_changes(), non_added(), non_deleted(), sum(), unchanged

**类签名**

类`sqlalchemy.orm.History`（`builtins.tuple`）

```py
attribute added: Tuple[()] | List[Any]
```

字段编号 0 的别名

```py
attribute deleted: Tuple[()] | List[Any]
```

字段编号 2 的别名

```py
method empty() → bool
```

如果这个`History`没有更改并且没有现有的未更改状态，则返回 True。

```py
method has_changes() → bool
```

如果这个`History`有更改，则返回 True。

```py
method non_added() → Sequence[Any]
```

返回未更改 + 已删除的集合。

```py
method non_deleted() → Sequence[Any]
```

返回已添加 + 未更改的集合。

```py
method sum() → Sequence[Any]
```

返回已添加 + 未更改 + 已删除的集合。

```py
attribute unchanged: Tuple[()] | List[Any]
```

字段编号 1 的别名
