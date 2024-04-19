# 反射数据库对象

> 原文：[`docs.sqlalchemy.org/en/20/core/reflection.html`](https://docs.sqlalchemy.org/en/20/core/reflection.html)

可以命令`Table`对象从数据库中已经存在的相应数据库架构对象中加载关于自身的信息。这个过程称为*反射*。在最简单的情况下，您只需要指定表名、一个`MetaData`对象和`autoload_with`参数：

```py
>>> messages = Table("messages", metadata_obj, autoload_with=engine)
>>> [c.name for c in messages.columns]
['message_id', 'message_name', 'date']
```

上述操作将使用给定的引擎来查询有关`messages`表格的数据库信息，然后将生成`Column`、`ForeignKey`和其他对象，这些对象对应于此信息，就像`Table`对象在 Python 中手工构造一样。

当表格被反射时，如果给定的表格通过外键引用另一个表格，那么在表示连接的`MetaData`对象中将创建第二个 `Table`对象。下面假设表格`shopping_cart_items`引用了一个名为`shopping_carts`的表格。反射`shopping_cart_items`表格的效果是`shopping_carts`表格也将被加载：

```py
>>> shopping_cart_items = Table("shopping_cart_items", metadata_obj, autoload_with=engine)
>>> "shopping_carts" in metadata_obj.tables
True
```

`MetaData`具有一种有趣的“类单例”行为，即如果您单独请求了两个表格，`MetaData`将确保为每个不同的表名创建一个 `Table`对象。如果具有给定名称的表格已经存在，则`Table`构造函数实际上会将已经存在的`Table`对象返回给您。例如，我们可以通过以下方式访问已经生成的`shopping_carts`表格：

```py
shopping_carts = Table("shopping_carts", metadata_obj)
```

当然，无论如何，最好在上述表格中使用`autoload_with=engine`。这样，如果尚未加载表格的属性，它们将被加载。只有在尚未加载表格的情况下才会自动加载表格；一旦加载，对于具有相同名称的新调用`Table`将不会重新发出任何反射查询。

## 覆盖反射的列

当反映表格时，可以通过显式值覆盖单个列；这对于指定自定义数据类型、数据库中可能未配置的主键等约束非常方便：

```py
>>> mytable = Table(
...     "mytable",
...     metadata_obj,
...     Column(
...         "id", Integer, primary_key=True
...     ),  # override reflected 'id' to have primary key
...     Column("mydata", Unicode(50)),  # override reflected 'mydata' to be Unicode
...     # additional Column objects which require no change are reflected normally
...     autoload_with=some_engine,
... )
```

另请参阅

使用自定义类型和反射 - 说明了上述列覆盖技术如何应用于使用自定义数据类型进行表反射。

## 反射视图

反射系统也可以反射视图。基本用法与表的用法相同：

```py
my_view = Table("some_view", metadata, autoload_with=engine)
```

在上面，`my_view` 是一个具有 `Column` 对象的 `Table` 对象，表示视图“some_view”中每列的名称和类型。

通常，在反映视图时，至少希望有一个主键约束，如果可能的话，也有外键。视图反射不会推断这些约束。

使用“覆盖”技术，明确指定那些是主键的列或具有外键约束的列：

```py
my_view = Table(
    "some_view",
    metadata,
    Column("view_id", Integer, primary_key=True),
    Column("related_thing", Integer, ForeignKey("othertable.thing_id")),
    autoload_with=engine,
)
```

## 一次性反射所有表格

`MetaData` 对象还可以获取表的列表并反映全部。这通过使用 `reflect()` 方法实现。调用后，所有定位的表格都存在于 `MetaData` 对象的表字典中：

```py
metadata_obj = MetaData()
metadata_obj.reflect(bind=someengine)
users_table = metadata_obj.tables["users"]
addresses_table = metadata_obj.tables["addresses"]
```

`metadata.reflect()` 还提供了一种方便的方式来清除或删除数据库中的所有行：

```py
metadata_obj = MetaData()
metadata_obj.reflect(bind=someengine)
with someengine.begin() as conn:
    for table in reversed(metadata_obj.sorted_tables):
        conn.execute(table.delete())
```

## 从其他架构中反射表格

章节 指定架构名称 介绍了表架构的概念，这是数据库中包含表和其他对象的命名空间，可以明确指定。`Table` 对象的“架构”，以及视图、索引和序列等其他对象的“架构”，可以使用 `Table.schema` 参数进行设置，也可以使用 `MetaData.schema` 参数作为 `MetaData` 对象的默认架构。

此架构参数的使用直接影响表反射功能在被要求反射对象时的搜索位置。例如，给定通过其 `MetaData.schema` 参数配置了默认架构名称“project”的 `MetaData` 对象：

```py
>>> metadata_obj = MetaData(schema="project")
```

然后，`MetaData.reflect()`将利用配置的`.schema`进行反射：

```py
>>> # uses `schema` configured in metadata_obj
>>> metadata_obj.reflect(someengine)
```

最终结果是，来自“project”模式的`Table`对象将被反映出来，并且它们将以该名称作为模式限定进行填充：

```py
>>> metadata_obj.tables["project.messages"]
Table('messages', MetaData(), Column('message_id', INTEGER(), table=<messages>), schema='project')
```

类似地，包括`Table.schema`参数的单个`Table`对象也将从该数据库模式反映出来，覆盖可能已经在拥有的`MetaData`集合上配置的任何默认模式：

```py
>>> messages = Table("messages", metadata_obj, schema="project", autoload_with=someengine)
>>> messages
Table('messages', MetaData(), Column('message_id', INTEGER(), table=<messages>), schema='project')
```

最后，`MetaData.reflect()`方法本身也允许传递一个`MetaData.reflect.schema`参数，因此我们也可以为默认配置的`MetaData`对象从“project”模式加载表：

```py
>>> metadata_obj = MetaData()
>>> metadata_obj.reflect(someengine, schema="project")
```

我们可以使用不同的`MetaData.schema`参数（或者完全不使用）任意多次调用`MetaData.reflect()`，以继续用更多对象填充`MetaData`对象：

```py
>>> # add tables from the "customer" schema
>>> metadata_obj.reflect(someengine, schema="customer")
>>> # add tables from the default schema
>>> metadata_obj.reflect(someengine)
```

### 与默认模式交互的模式限定反射

最佳实践总结部分

在本节中，我们讨论了 SQLAlchemy 关于数据库会话中“默认模式”可见的表的反射行为，以及这些如何与明确包含模式的 SQLAlchemy 指令相互作用。作为最佳实践，请确保数据库的“默认”模式只是一个单一名称，而不是名称列表；对于属于此“默认”模式并且可以在 DDL 和 SQL 中不带模式限定命名的表，请将相应的`Table.schema`和类似的模式参数设置为它们的默认值`None`。

如 在 MetaData 中指定默认模式名称 中所述，具有模式概念的数据库通常还包括“默认”模式的概念。这自然是因为当引用没有模式的表对象时（这是常见的情况），支持模式的数据库仍然会认为该表在某处存在“模式”。一些数据库，如 PostgreSQL，将这个概念进一步扩展为 [模式搜索路径](https://www.postgresql.org/docs/current/static/ddl-schemas.html#DDL-SCHEMAS-PATH)，在特定数据库会话中可以考虑多个模式名称为“隐式”；引用其中任何一个模式中的表名都不需要存在模式名称（与此同时，如果模式名称存在，则也是完全可以的）。

由于大多数关系型数据库都有特定的表对象概念，可以以模式限定的方式引用，也可以以“隐式”方式引用，即没有模式存在，这给 SQLAlchemy 的反射特性带来了复杂性。以模式限定方式反射表将始终填充其 `Table.schema` 属性，并且会影响此 `Table` 如何组织到 `MetaData.tables` 集合中，也就是以模式限定方式。相反，以非模式限定方式反射 **同样的** 表将使其以非模式限定方式组织到 `MetaData.tables` 集合中。最终的结果是，单个 `MetaData` 集合中将存在两个独立的表示实际数据库中同一表的 `Table` 对象。

为了说明这个问题的影响，考虑前面示例中“project”模式中的表，并假设“project”模式也是我们数据库连接的默认模式，或者如果使用 PostgreSQL 等数据库，则假设“project”模式设置在 PostgreSQL 的 `search_path` 中。这意味着数据库接受以下两个 SQL 语句作为等价：

```py
-- schema qualified
SELECT  message_id  FROM  project.messages

-- non-schema qualified
SELECT  message_id  FROM  messages
```

在 SQLAlchemy 中，这并不是一个问题，因为可以以两种方式找到表。但是，在 SQLAlchemy 中，是`Table`对象的**标识**决定了它在 SQL 语句中的语义角色。基于 SQLAlchemy 当前的决策，这意味着如果我们以模式限定的方式和非模式限定的方式反射相同的“messages”表，我们将得到**两个**不会被视为语义等同的`Table`对象：

```py
>>> # reflect in non-schema qualified fashion
>>> messages_table_1 = Table("messages", metadata_obj, autoload_with=someengine)
>>> # reflect in schema qualified fashion
>>> messages_table_2 = Table(
...     "messages", metadata_obj, schema="project", autoload_with=someengine
... )
>>> # two different objects
>>> messages_table_1 is messages_table_2
False
>>> # stored in two different ways
>>> metadata.tables["messages"] is messages_table_1
True
>>> metadata.tables["project.messages"] is messages_table_2
True
```

当被反映的表包含对其他表的外键引用时，上述问题变得更加复杂。假设“messages”有一个“project_id”列，它引用另一个模式本地表“projects”的行，这意味着“messages”表的定义中有一个`ForeignKeyConstraint`对象。

我们可能会发现自己处于一个情况下，其中一个`MetaData`集合可能包含表示这两个数据库表的四个`Table`对象，其中一个或两个附加表是由反射过程生成的；这是因为当反射过程遇到要反射的表上的外键约束时，它会分支出去反射该引用表。它用于为这个引用表分配模式的决策是，如果拥有的`Table`也省略了其模式名称，并且这两个对象位于相同的模式中，则 SQLAlchemy 将**省略默认模式**从反射的`ForeignKeyConstraint`对象中，但如果没有省略，则**包括**它。

常见的情况是以模式限定的方式反射表，然后以模式限定的方式加载一个相关表：

```py
>>> # reflect "messages" in a schema qualified fashion
>>> messages_table_1 = Table(
...     "messages", metadata_obj, schema="project", autoload_with=someengine
... )
```

上述`messages_table_1`也将以模式限定的方式引用`projects`。这个`projects`表将自动反映出“messages”引用它的事实：

```py
>>> messages_table_1.c.project_id
Column('project_id', INTEGER(), ForeignKey('project.projects.project_id'), table=<messages>)
```

如果代码的其他部分以非模式限定的方式反映“projects”，那么现在有两个不同的项目表：

```py
>>> # reflect "projects" in a non-schema qualified fashion
>>> projects_table_1 = Table("projects", metadata_obj, autoload_with=someengine)
```

```py
>>> # messages does not refer to projects_table_1 above
>>> messages_table_1.c.project_id.references(projects_table_1.c.project_id)
False
```

```py
>>> # it refers to this one
>>> projects_table_2 = metadata_obj.tables["project.projects"]
>>> messages_table_1.c.project_id.references(projects_table_2.c.project_id)
True
```

```py
>>> # they're different, as one non-schema qualified and the other one is
>>> projects_table_1 is projects_table_2
False
```

上述混淆可能会在使用表反射加载应用程序级`Table`对象的应用程序中以及在迁移场景中（尤其是使用 Alembic Migrations 检测新表和外键约束时）引起问题。

以上行为可以通过坚持一项简单的做法来纠正：

+   对于期望位于数据库**默认**模式中的任何`Table`，不要包含`Table.schema`参数。

对于支持模式“搜索”路径的 PostgreSQL 和其他数据库，添加以下额外做法：

+   将“搜索路径”限定为**仅一个模式，即默认模式**。

另请参阅

远程模式表反射和 PostgreSQL 搜索路径 - 关于 PostgreSQL 数据库的此行为的附加细节。## 使用检查器进行精细化反射

还提供了一个低级接口，它提供了一种与后端无关的从给定数据库加载模式、表、列和约束描述列表的系统。这被称为“检查器”：

```py
from sqlalchemy import create_engine
from sqlalchemy import inspect

engine = create_engine("...")
insp = inspect(engine)
print(insp.get_table_names())
```

| 对象名称 | 描述 |
| --- | --- |
| Inspector | 执行数据库模式检查。 |
| ReflectedCheckConstraint | 表示与`CheckConstraint`对应的反射元素的字典。 |
| ReflectedColumn | 表示与`Column`对象对应的反射元素的字典。 |
| ReflectedComputed | 表示计算列的反射元素，对应于`Computed`构造。 |
| ReflectedForeignKeyConstraint | 表示与`ForeignKeyConstraint`对应的反射元素的字典。 |
| ReflectedIdentity | 表示列的反射 IDENTITY 结构，对应于`Identity`构造。 |
| ReflectedIndex | 表示与`Index`对应的反射元素的字典。 |
| ReflectedPrimaryKeyConstraint | 表示与`PrimaryKeyConstraint`对应的反射元素的字典。 |
| ReflectedTableComment | 表示对应于`Table.comment`属性的反射注释的字典。 |
| ReflectedUniqueConstraint | 表示对应于`UniqueConstraint`的反射元素的字典。 |

```py
class sqlalchemy.engine.reflection.Inspector
```

执行数据库模式检查。

Inspector 充当`Dialect`的反射方法的代理，提供一致的接口以及对先前获取的元数据的缓存支持。

`Inspector` 对象通常通过`inspect()`函数创建，该函数可以传递一个`Engine`或一个`Connection`：

```py
from sqlalchemy import inspect, create_engine
engine = create_engine('...')
insp = inspect(engine)
```

在上述情况中，与引擎关联的`Dialect`可能选择返回一个提供了特定于方言目标数据库的额外方法的`Inspector`子类。

**成员**

__init__(), bind, clear_cache(), default_schema_name, dialect, engine, from_engine(), get_check_constraints(), get_columns(), get_foreign_keys(), get_indexes(), get_materialized_view_names(), get_multi_check_constraints(), get_multi_columns(), get_multi_foreign_keys(), get_multi_indexes(), get_multi_pk_constraint(), get_multi_table_comment(), get_multi_table_options(), get_multi_unique_constraints(), get_pk_constraint(), get_schema_names(), get_sequence_names(), get_sorted_table_and_fkc_names(), get_table_comment(), get_table_names(), get_table_options(), get_temp_table_names(), get_temp_view_names(), get_unique_constraints(), get_view_definition(), get_view_names(), has_index(), has_schema(), has_sequence(), has_table(), info_cache, reflect_table(), sort_tables_on_foreign_key_dependency()

**类签名**

类`sqlalchemy.engine.reflection.Inspector`（`sqlalchemy.inspection.Inspectable`）

```py
method __init__(bind: Engine | Connection)
```

初始化一个新的`Inspector`。

自版本 1.4 起已弃用：`Inspector`上的 __init__()方法已弃用，并将在将来的版本中删除。请使用`Engine`或`Connection`上的`inspect()`函数来获取`Inspector`。

参数：

**bind** – 一个`Connection`，通常是`Engine`或`Connection`的实例。

对于`Inspector`的方言特定实例，请参阅`Inspector.from_engine()`。

```py
attribute bind: Engine | Connection
```

```py
method clear_cache() → None
```

重置此`Inspector`的缓存。

具有数据缓存的检查方法在下次调用以获取新数据时将发出 SQL 查询。

版本 2.0 中的新功能。

```py
attribute default_schema_name
```

返回当前引擎数据库用户的方言呈现的默认模式名称。

例如，对于 PostgreSQL 通常为`public`，对于 SQL Server 为`dbo`。

```py
attribute dialect: Dialect
```

```py
attribute engine: Engine
```

```py
classmethod from_engine(bind: Engine) → Inspector
```

从给定的引擎或连接构造一个新的特定于方言的 Inspector 对象。

自版本 1.4 起已弃用：`Inspector`上的 from_engine()方法已弃用，并将在将来的版本中删除。请使用`Engine`或`Connection`上的`inspect()`函数来获取`Inspector`。

参数：

**bind** – 一个`Connection`或`Engine`。

该方法与直接构造函数调用`Inspector`不同，因为`Dialect`有机会提供特定于方言的`Inspector`实例，该实例可能提供其他方法。

请参阅`Inspector`的示例。

```py
method get_check_constraints(table_name: str, schema: str | None = None, **kw: Any) → List[ReflectedCheckConstraint]
```

返回`table_name`中关于检查约束的信息。

给定一个字符串`table_name`和一个可选的字符串模式，将检查约束信息作为`ReflectedCheckConstraint`的列表返回。

参数：

+   `table_name` – 表的名称字符串。对于特殊引用，请使用`quoted_name`。

+   `schema` – 字符串模式名称；如果省略，则使用数据库连接的默认模式。对于特殊引用，请使用`quoted_name`。

+   `**kw` – 传递给特定方言实现的其他关键字参数。有关更多信息，请参阅正在使用的方言的文档。

返回：

一个字典列表，每个字典代表检查约束的定义。

另见

`Inspector.get_multi_check_constraints()`

```py
method get_columns(table_name: str, schema: str | None = None, **kw: Any) → List[ReflectedColumn]
```

返回`table_name`中关于列的信息。

给定一个字符串`table_name`和一个可选的字符串`schema`，返回列信息作为`ReflectedColumn`的列表。

参数：

+   `table_name` – 表的名称字符串。对于特殊引用，请使用`quoted_name`。

+   `schema` – 字符串模式名称；如果省略，则使用数据库连接的默认模式。对于特殊引用，请使用`quoted_name`。

+   `**kw` – 传递给特定方言实现的其他关键字参数。有关更多信息，请参阅正在使用的方言的文档。

返回：

一个字典列表，每个字典代表数据库列的定义。

另见

`Inspector.get_multi_columns()`.

```py
method get_foreign_keys(table_name: str, schema: str | None = None, **kw: Any) → List[ReflectedForeignKeyConstraint]
```

返回`table_name`中的外键信息。

给定一个字符串`table_name`和一个可选的字符串模式，返回外键信息作为`ReflectedForeignKeyConstraint`的列表。

参数：

+   `table_name` – 表的字符串名称。对于特殊引用，请使用`quoted_name`。

+   `schema` – 字符串模式名称；如果省略，则使用数据库连接的默认模式。对于特殊引用，请使用`quoted_name`。

+   `**kw` – 传递给特定方言实现的额外关键字参数。有关更多信息，请参阅正在使用的方言的文档。

返回：

一个字典列表，每个代表一个外键定义。

另请参阅

`Inspector.get_multi_foreign_keys()`

```py
method get_indexes(table_name: str, schema: str | None = None, **kw: Any) → List[ReflectedIndex]
```

返回表`table_name`中索引的信息。

给定一个字符串`table_name`和一个可选的字符串模式，返回索引信息作为`ReflectedIndex`的列表。

参数：

+   `table_name` – 表的字符串名称。对于特殊引用，请使用`quoted_name`。

+   `schema` – 字符串模式名称；如果省略，则使用数据库连接的默认模式。对于特殊引用，请使用`quoted_name`。

+   `**kw` – 传递给特定方言实现的额外关键字参数。有关更多信息，请参阅正在使用的方言的文档。

返回：

一个字典列表，每个代表一个索引的定义。

另请参阅

`Inspector.get_multi_indexes()`

```py
method get_materialized_view_names(schema: str | None = None, **kw: Any) → List[str]
```

返回模式中所有物化视图的名称。

参数：

+   `schema` – 可选，从非默认模式中检索名称。对于特殊引用，请���用`quoted_name`。

+   `**kw` – 传递给特定方言实现的额外关键字参数。有关更多信息，请参阅正在使用的方言的文档。

2.0 版本中的新功能。

另请参阅

`Inspector.get_view_names()`

```py
method get_multi_check_constraints(schema: str | None = None, filter_names: Sequence[str] | None = None, kind: ObjectKind = ObjectKind.TABLE, scope: ObjectScope = ObjectScope.DEFAULT, **kw: Any) → Dict[TableKey, List[ReflectedCheckConstraint]]
```

返回给定模式中所有表中检查约束的信息。

可通过将要用于`filter_names`的名称传递来过滤表。

对于每个表，值是一个`ReflectedCheckConstraint`列表。

参数：

+   `schema` – 字符串模式名称；如果省略，则使用数据库连接的默认模式。要进行特殊引用，请使用`quoted_name`。

+   `filter_names` – 可选择性地仅返回列出的对象的信息。

+   `kind` – 指定要反映的对象类型的`ObjectKind`。默认为`ObjectKind.TABLE`。

+   `scope` – 指定要反映默认、临时或任何表的约束的`ObjectScope`。默认为`ObjectScope.DEFAULT`。

+   `**kw` – 传递给特定方言实现的额外关键字参数。有关更多信息，请参阅正在使用的方言的文档。

返回：

一个字典，其中键是两元组模式、表名，值是字典列表，每个字典表示检查约束的定义。如果未提供模式，则模式为`None`。

新版本 2.0 中新增。

另请参见

`Inspector.get_check_constraints()`

```py
method get_multi_columns(schema: str | None = None, filter_names: Sequence[str] | None = None, kind: ObjectKind = ObjectKind.TABLE, scope: ObjectScope = ObjectScope.DEFAULT, **kw: Any) → Dict[TableKey, List[ReflectedColumn]]
```

返回给定模式中所有对象中列的信息。

可通过将要使用的名称传递给`filter_names`来过滤对象。

对于每个表，值是一个`ReflectedColumn`列表。

参数：

+   `schema` – 字符串模式名称；如果省略，则使用数据库连接的默认模式。要进行特殊引用，请使用`quoted_name`。

+   `filter_names` – 可选择性地仅返回列出的对象的信息。

+   `kind` – 指定要反映的对象类型的`ObjectKind`。默认为`ObjectKind.TABLE`。

+   `scope` – 指定要反映默认、临时或任何表的列的`ObjectScope`。默认为`ObjectScope.DEFAULT`。

+   `**kw` – 传递给特定方言实现的额外关键字参数。有关更多信息，请参阅正在使用的方言的文档。

返回：

一个字典，其中键是两元组模式、表名，值是字典列表，每个字典表示数据库列的定义。如果未提供模式，则模式为`None`。

新版本 2.0 中新增。

另请参见

`Inspector.get_columns()`

```py
method get_multi_foreign_keys(schema: str | None = None, filter_names: Sequence[str] | None = None, kind: ObjectKind = ObjectKind.TABLE, scope: ObjectScope = ObjectScope.DEFAULT, **kw: Any) → Dict[TableKey, List[ReflectedForeignKeyConstraint]]
```

返回给定模式中所有表中外键的信息。

可通过将要使用的名称传递给`filter_names`来过滤表。

对于每个表，该值是一个 `ReflectedForeignKeyConstraint` 列表。

参数：

+   `schema` – 字符串模式名称；如果省略，将使用数据库连接的默认模式。对于特殊引用，请使用 `quoted_name`。

+   `filter_names` – 可选择仅返回此处列出的对象信息。

+   `kind` – 一个指定要反映的对象类型的 `ObjectKind`。默认为 `ObjectKind.TABLE`。

+   `scope` – 一个指定是否应反映默认、临时或任何表的外键的 `ObjectScope`。默认为 `ObjectScope.DEFAULT`。

+   `**kw` – 传递给特定方言实现的其他关键字参数。有关更多信息，请参阅正在使用的方言的文档。

返回：

一个字典，其中键是两元组模式、表名，值是字典列表，每个表示外键定义。如果未提供模式，则模式为 `None`。

2.0 版中的新功能。

另请参阅

`Inspector.get_foreign_keys()`

```py
method get_multi_indexes(schema: str | None = None, filter_names: Sequence[str] | None = None, kind: ObjectKind = ObjectKind.TABLE, scope: ObjectScope = ObjectScope.DEFAULT, **kw: Any) → Dict[TableKey, List[ReflectedIndex]]
```

返回给定模式中所有对象中索引的信息。

通过将要使用的名称传递给 `filter_names` 来过滤对象。

对于每个表，该值是一个 `ReflectedIndex` 列表。

参数：

+   `schema` – 字符串模式名称；如果省略，将使用数据库连接的默认模式。对于特殊引用，请使用 `quoted_name`。

+   `filter_names` – 可选择仅返回此处列出的对象信息。

+   `kind` – 一个指定要反映的对象类型的 `ObjectKind`。默认为 `ObjectKind.TABLE`。

+   `scope` – 一个指定是否应反映默认、临时或任何表的索引的 `ObjectScope`。默认为 `ObjectScope.DEFAULT`。

+   `**kw` – 传递给特定方言实现的其他关键字参数。有关更多信息，请参阅正在使用的方言的文档。

返回：

一个字典，其中键是两元组模式、表名，值是字典列表，每个表示索引的定义。如果未提供模式，则模式为 `None`。

2.0 版中的新功能。

另请参阅

`Inspector.get_indexes()`

```py
method get_multi_pk_constraint(schema: str | None = None, filter_names: Sequence[str] | None = None, kind: ObjectKind = ObjectKind.TABLE, scope: ObjectScope = ObjectScope.DEFAULT, **kw: Any) → Dict[TableKey, ReflectedPrimaryKeyConstraint]
```

返回给定模式中所有表的主键约束的信息。

通过将要使用的名称传递给 `filter_names` 来过滤表格。

对于每个表，该值为 `ReflectedPrimaryKeyConstraint`。

参数：

+   `schema` – 字符串模式名称；如果省略，则使用数据库连接的默认模式。要进行特殊引用，请使用 `quoted_name`。

+   `filter_names` – 可选择仅返回此处列出的对象的信息。

+   `kind` – 一个指定要反映的对象类型的 `ObjectKind`。默认为 `ObjectKind.TABLE`。

+   `scope` – 一个指定应反映默认、临时或任何表的主键的 `ObjectScope`。默认为 `ObjectScope.DEFAULT`。

+   `**kw` – 要传递给特定方言实现的其他关键字参数。有关更多信息，请参阅正在使用的方言的文档。

返回：

一个字典，其键为二元组模式、表名，值为每个表示主键约束的定义的字典。如果未提供模式，则模式为 `None`。

2.0 版中的新内容。

另请参见

`Inspector.get_pk_constraint()`

```py
method get_multi_table_comment(schema: str | None = None, filter_names: Sequence[str] | None = None, kind: ObjectKind = ObjectKind.TABLE, scope: ObjectScope = ObjectScope.DEFAULT, **kw: Any) → Dict[TableKey, ReflectedTableComment]
```

返回给定模式中所有对象中表注释的信息。

可通过传递用于 `filter_names` 的名称来过滤对象。

对于每个表，该值为 `ReflectedTableComment`。

对不支持注释的方言引发 `NotImplementedError`。

参数：

+   `schema` – 字符串模式名称；如果省略，则使用数据库连接的默认模式。要进行特殊引用，请使用 `quoted_name`。

+   `filter_names` – 可选择仅返回此处列出的对象的信息。

+   `kind` – 一个指定要反映的对象类型的 `ObjectKind`。默认为 `ObjectKind.TABLE`。

+   `scope` – 一个指定应反映默认、临时或任何表的注释的 `ObjectScope`。默认为 `ObjectScope.DEFAULT`。

+   `**kw` – 要传递给特定方言实现的其他关键字参数。有关更多信息，请参阅正在使用的方言的文档。

返回：

一个字典，其键为二元组模式、表名，值为表示表注释的字典。如果未提供模式，则模式为 `None`。

2.0 版中的新内容。

另请参见

`Inspector.get_table_comment()`

```py
method get_multi_table_options(schema: str | None = None, filter_names: Sequence[str] | None = None, kind: ObjectKind = ObjectKind.TABLE, scope: ObjectScope = ObjectScope.DEFAULT, **kw: Any) → Dict[TableKey, Dict[str, Any]]
```

返回指定了在给定模式中创建表时指定的选项的字典。

可通过传递用于 `filter_names` 的名称来过滤表。

目前包括适用于 MySQL 和 Oracle 表的一些选项。

参数：

+   `schema` – 字符串模式名称；如果省略，使用数据库连接的默认模式。对于特殊引用，请使用`quoted_name`。

+   `filter_names` – 可选择仅返回列出的对象的信息。

+   `kind` – 一个`ObjectKind`，指定要反映的对象类型。默认为`ObjectKind.TABLE`。

+   `scope` – 一个`ObjectScope`，指定是否应反映默认、临时或任何表的选项。默认为`ObjectScope.DEFAULT`。

+   `**kw` – 传递给特定方言实现的额外关键字参数。有关更多信息，请参阅正在使用的方言的文档。

返回：

一个字典，其中键是两元组模式、表名，值是具有表选项的字典。每个字典中返回的键取决于正在使用的方言。每个键都以方言名称为前缀。如果未提供模式，则模式为`None`。

版本 2.0 中的新功能。

另请参见

`Inspector.get_table_options()`

```py
method get_multi_unique_constraints(schema: str | None = None, filter_names: Sequence[str] | None = None, kind: ObjectKind = ObjectKind.TABLE, scope: ObjectScope = ObjectScope.DEFAULT, **kw: Any) → Dict[TableKey, List[ReflectedUniqueConstraint]]
```

返回给定模式中所有表中唯一约束的信息。

可通过传递要用于`filter_names`的名称来过滤表。

对于每个表，值是一个`ReflectedUniqueConstraint`列表。

参数：

+   `schema` – 字符串模式名称；如果省略，使用数据库连接的默认模式。对于特殊引用，请使用`quoted_name`。

+   `filter_names` – 可选择仅返回列出的对象的信息。

+   `kind` – 一个`ObjectKind`，指定要反映的对象类型。默认为`ObjectKind.TABLE`。

+   `scope` – 一个`ObjectScope`，指定是否应反映默认、临时或任何表的约束。默认为`ObjectScope.DEFAULT`。

+   `**kw` – 传递给特定方言实现的额外关键字参数。有关更多信息，请参阅正在使用的方言的文档。

返回：

一个字典，其中键是两元组模式、表名，值是字典列表，每个表示唯一约束的定义。如果未提供模式，则模式为`None`。

版本 2.0 中的新功能。

另请参见

`Inspector.get_unique_constraints()`

```py
method get_pk_constraint(table_name: str, schema: str | None = None, **kw: Any) → ReflectedPrimaryKeyConstraint
```

返回有关`table_name`中主键约束的信息。

给定字符串 `table_name`，和可选的字符串模式，作为 `ReflectedPrimaryKeyConstraint` 返回主键信息。

参数：

+   `table_name` – 表的字符串名称。对于特殊引用，请使用`quoted_name`。

+   `schema` – 字符串模式名称；如果省略，则使用数据库连接的默认模式。对于特殊引用，请使用`quoted_name`。

+   `**kw` – 传递给特定方言实现的附加关键字参数。有关更多信息，请参阅使用的方言的文档。

返回：

代表主键约束定义的字典。

另请参阅

`Inspector.get_multi_pk_constraint()`

```py
method get_schema_names(**kw: Any) → List[str]
```

返回所有模式名称。

参数：

****kw** – 传递给特定方言实现的附加关键字参数。有关更多信息，请参阅使用的方言的文档。

```py
method get_sequence_names(schema: str | None = None, **kw: Any) → List[str]
```

返回模式中的所有序列名称。

参数：

+   `schema` – 可选，从非默认模式检索名称。对于特殊引用，请使用`quoted_name`。

+   `**kw` – 传递给特定方言实现的附加关键字参数。有关更多信息，请参阅使用的方言的文档。

```py
method get_sorted_table_and_fkc_names(schema: str | None = None, **kw: Any) → List[Tuple[str | None, List[Tuple[str, str | None]]]]
```

返回特定模式中引用的表和外键约束名称的依赖排序。

这将生成 `(tablename, [(tname, fkname), (tname, fkname), ...])` 的 2 元组，其中包含按创建顺序分组的表名和未被检测为属于循环的外键约束名称。最后一个元素将是 `(None, [(tname, fkname), (tname, fkname), ..])`，其中包含剩余的外键约束名称，这些名称需要根据表之间的依赖关系在事后进行单独的创建步骤。

参数：

+   `schema` – 要查询的模式名称，如果不是默认模式。

+   `**kw` – 传递给特定方言实现的附加关键字参数。有关更多信息，请参阅使用的方言的文档。

另请参阅

`Inspector.get_table_names()`

`sort_tables_and_constraints()` - 与已给定的 `MetaData` 类似的方法。

```py
method get_table_comment(table_name: str, schema: str | None = None, **kw: Any) → ReflectedTableComment
```

返回 `table_name` 的表注释信息。

给定字符串`table_name`和可选字符串`schema`，将表注释信息作为`ReflectedTableComment`返回。

对于不支持注释的方言，引发`NotImplementedError`。 

参数：

+   `table_name` – 表的字符串名称。对于特殊引用，请使用`quoted_name`。

+   `schema` – 字符串模式名称；如果省略，则使用数据库连接的默认模式。对于特殊引用，请使用`quoted_name`。

+   `**kw` – 传递给特定方言实现的额外关键字参数。有关更多信息，请参阅正在使用的方言的文档。

返回：

一个字典，带有表注释。

自版本 1.2 新增。

另请参阅

`Inspector.get_multi_table_comment()`

```py
method get_table_names(schema: str | None = None, **kw: Any) → List[str]
```

返回特定模式内的所有表名。

名称预期仅为实际表，而不是视图。视图使用`Inspector.get_view_names()`和/或`Inspector.get_materialized_view_names()`方法返回。

参数：

+   `schema` – 模式名称。如果`schema`为`None`，则使用数据库的默认模式，否则搜索命名模式。如果数据库不支持命名模式，则如果未将`schema`作为`None`传递，则行为未定义。对于特殊引用，请使用`quoted_name`。

+   `**kw` – 传递给特定方言实现的额外关键字参数。有关更多信息，请参阅正在使用的方言的文档。

另请参阅

`Inspector.get_sorted_table_and_fkc_names()`

`MetaData.sorted_tables`

```py
method get_table_options(table_name: str, schema: str | None = None, **kw: Any) → Dict[str, Any]
```

返回给定名称的表创建时指定的选项的字典。

目前包括适用于 MySQL 和 Oracle 表的某些选项。

参数：

+   `table_name` – 表的字符串名称。对于特殊引用，请使用`quoted_name`。

+   `schema` – 字符串模式名称；如果省略，则使用数据库连接的默认模式。要进行特殊引用，请使用`quoted_name`。

+   `**kw` – 传递给方言特定实现的附加关键字参数。有关更多信息，请参阅正在使用的方言的文档。

返回：

一个包含表选项的字典。返回的键取决于使用的方言。每个键都以方言名称为前缀。

另请参阅

`Inspector.get_multi_table_options()`

```py
method get_temp_table_names(**kw: Any) → List[str]
```

返回当前绑定的临时表名称列表。

大多数方言都不支持此方法；目前只有 Oracle、PostgreSQL 和 SQLite 实现了它。

参数：

****kw** – 传递给方言特定实现的附加关键字参数。有关更多信息，请参阅正在使用的方言的文档。

```py
method get_temp_view_names(**kw: Any) → List[str]
```

返回当前绑定的临时视图名称列表。

大多数方言都不支持此方法；目前只有 PostgreSQL 和 SQLite 实现了它。

参数：

****kw** – 传递给方言特定实现的附加关键字参数。有关更多信息，请参阅正在使用的方言的文档。

```py
method get_unique_constraints(table_name: str, schema: str | None = None, **kw: Any) → List[ReflectedUniqueConstraint]
```

返回`table_name`中唯一约束的信息。

给定一个字符串`table_name`和一个可选的字符串模式，将唯一约束信息返回为一个`ReflectedUniqueConstraint`的列表。

参数：

+   `table_name` – 表名称字符串。要进行特殊引用，请使用`quoted_name`。

+   `schema` – 字符串模式名称；如果省略，则使用数据库连接的默认模式。要进行特殊引用，请使用`quoted_name`。

+   `**kw` – 传递给方言特定实现的附加关键字参数。有关更多信息，请参阅正在使用的方言的文档。

返回：

一个字典列表，每个都代表唯一约束的定义。

另请参阅

`Inspector.get_multi_unique_constraints()`

```py
method get_view_definition(view_name: str, schema: str | None = None, **kw: Any) → str
```

返回名为`view_name`的普通或物化视图的定义。

参数：

+   `view_name` – 视图的名称。

+   `schema` – 可选，从非默认模式检索名称。要进行特殊引用，请使用`quoted_name`。

+   `**kw` – 传递给方言特定实现的附加关键字参数。有关更多信息，请参阅正在使用的方言的文档。

```py
method get_view_names(schema: str | None = None, **kw: Any) → List[str]
```

返回模式中的所有非材料化视图名称。

参数：

+   `schema` – 可选，从非默认模式中检索名称。对于特殊引用，请使用 `quoted_name`。

+   `**kw` – 传递给方言特定实现的其他关键字参数。有关更多信息，请参阅正在使用的方言的文档。

从版本 2.0 开始更改：对于以前在此列表中包括材料化视图名称的方言（当前为 PostgreSQL），此方法不再返回材料化视图的名称。应改为使用 `Inspector.get_materialized_view_names()` 方法。

另请参阅

`Inspector.get_materialized_view_names()`

```py
method has_index(table_name: str, index_name: str, schema: str | None = None, **kw: Any) → bool
```

检查数据库中特定索引名称的存在。

参数：

+   `table_name` – 索引所属表的名称

+   `index_name` – 要检查的索引名称

+   `schema` – 查询的模式名称，如果不是默认模式。

+   `**kw` – 传递给方言特定实现的其他关键字参数。有关更多信息，请参阅正在使用的方言的文档。

版本 2.0 中的新功能。

```py
method has_schema(schema_name: str, **kw: Any) → bool
```

如果后端具有给定名称的模式，则返回 True。

参数：

+   `schema_name` – 要检查的模式名称

+   `**kw` – 传递给方言特定实现的其他关键字参数。有关更多信息，请参阅正在使用的方言的文档。

版本 2.0 中的新功能。

```py
method has_sequence(sequence_name: str, schema: str | None = None, **kw: Any) → bool
```

如果后端具有给定名称的序列，则返回 True。

参数：

+   `sequence_name` – 要检查的序列名称

+   `schema` – 查询的模式名称，如果不是默认模式。

+   `**kw` – 传递给方言特定实现的其他关键字参数。有关更多信息，请参阅正在使用的方言的文档。

版本 1.4 中的新功能。

```py
method has_table(table_name: str, schema: str | None = None, **kw: Any) → bool
```

如果后端具有给定名称的表、视图或临时表，则返回 True。

参数：

+   `table_name` – 要检查的表名称

+   `schema` – 查询的模式名称，如果不是默认模式。

+   `**kw` – 传递给方言特定实现的其他关键字参数。有关更多信息，请参阅正在使用的方言的文档。

从版本 1.4 开始：- `Inspector.has_table()` 方法替换了 `Engine.has_table()` 方法。

从版本 2.0 开始更改：`Inspector.has_table()` 现在正式支持检查额外的类似表的对象：

+   任何类型的视图（普通或材料化）

+   任何类型的临时表

以前，这两个检查没有正式指定，并且不同的方言在行为上会有所不同。方言测试套件现在包括所有这些对象类型的测试，并且应该由所有包含 SQLAlchemy 的方言支持。但是，第三方方言中的支持可能滞后。

```py
attribute info_cache: Dict[Any, Any]
```

```py
method reflect_table(table: Table, include_columns: Collection[str] | None, exclude_columns: Collection[str] = (), resolve_fks: bool = True, _extend_on: Set[Table] | None = None, _reflect_info: _ReflectionInfo | None = None) → None
```

给定一个 `Table` 对象，根据内省加载其内部构造。

这是大多数方言用于生成表反射的底层方法。直接用法如下：

```py
from sqlalchemy import create_engine, MetaData, Table
from sqlalchemy import inspect

engine = create_engine('...')
meta = MetaData()
user_table = Table('user', meta)
insp = inspect(engine)
insp.reflect_table(user_table, None)
```

从版本 1.4 开始更改：从 `reflecttable` 更名为 `reflect_table`

参数：

+   `table` – 一个 `Table` 实例。

+   `include_columns` – 要包含在反射过程中的字符串列名列表。如果为 `None`，则反射所有列。

```py
method sort_tables_on_foreign_key_dependency(consider_schemas: Collection[str | None] = (None,), **kw: Any) → List[Tuple[Tuple[str | None, str] | None, List[Tuple[Tuple[str | None, str], str | None]]]]
```

返回在多个模式中引用的依赖项排序的表和外键约束名称。

此方法可以与 `Inspector.get_sorted_table_and_fkc_names()` 进行比较，后者一次只能处理一个模式；在这里，该方法是一个通用化的方法，一次可以考虑多个模式，包括解决跨模式外键的问题。

版本 2.0 中新增。

```py
class sqlalchemy.engine.interfaces.ReflectedColumn
```

表示与 `Column` 对象对应的反射元素的字典。

`ReflectedColumn` 结构是由 `get_columns` 方法返回的。

**成员**

autoincrement, comment, computed, default, dialect_options, identity, name, nullable, type

**类签名**

类 `sqlalchemy.engine.interfaces.ReflectedColumn` (`builtins.dict`)

```py
attribute autoincrement: NotRequired[bool]
```

数据库相关的自增标志。

此标志指示列是否具有某种数据库端的 “autoincrement” 标志。在 SQLAlchemy 中，其他类型的列也可以充当 “autoincrement” 列，而不一定在它们身上具有这样的标志。

有关 “autoincrement” 的更多背景信息，请参阅 `Column.autoincrement`。

```py
attribute comment: NotRequired[str | None]
```

如果存在，为列添加注释。只有一些方言会返回此键。

```py
attribute computed: NotRequired[ReflectedComputed]
```

指示此列由数据库计算。只有一些方言会返回此键。

版本 1.3.16 中新增：- 增加对计算反射的支持。

```py
attribute default: str | None
```

列默认表达式作为 SQL 字符串

```py
attribute dialect_options: NotRequired[Dict[str, Any]]
```

检测到此反射对象的附加方言特定选项

```py
attribute identity: NotRequired[ReflectedIdentity]
```

指示此列为 IDENTITY 列。只有一些方言会返回此键。

版本 1.4 中新增：- 增加对标识列反射的支持。

```py
attribute name: str
```

列名

```py
attribute nullable: bool
```

如果列为 NULL 或 NOT NULL，则为布尔标志

```py
attribute type: TypeEngine[Any]
```

列类型表示为`TypeEngine` 实例。

```py
class sqlalchemy.engine.interfaces.ReflectedComputed
```

表示计算列的反射元素，对应于`Computed` 结构。

`ReflectedComputed` 结构是 `ReflectedColumn` 结构的一部分，由 `Inspector.get_columns()` 方法返回。

**成员**

persisted，sqltext

**类签名**

类`sqlalchemy.engine.interfaces.ReflectedComputed` (`builtins.dict`)

```py
attribute persisted: NotRequired[bool]
```

指示值是存储在表中还是按需计算

```py
attribute sqltext: str
```

用于生成此列的表达式，返回为字符串 SQL 表达式

```py
class sqlalchemy.engine.interfaces.ReflectedCheckConstraint
```

字典表示反射元素，对应于`CheckConstraint`。

`ReflectedCheckConstraint` 结构由 `Inspector.get_check_constraints()` 方法返回。

**成员**

dialect_options，sqltext

**类签名**

类`sqlalchemy.engine.interfaces.ReflectedCheckConstraint` (`builtins.dict`)

```py
attribute dialect_options: NotRequired[Dict[str, Any]]
```

检测到此检查约束的附加方言特定选项

版本 1.3.8 中新增。

```py
attribute sqltext: str
```

检查约束的 SQL 表达式

```py
class sqlalchemy.engine.interfaces.ReflectedForeignKeyConstraint
```

字典表示对应于 `ForeignKeyConstraint` 的反射元素。

`ReflectedForeignKeyConstraint` 结构是由 `Inspector.get_foreign_keys()` 方法返回的。

**成员**

constrained_columns, options, referred_columns, referred_schema, referred_table

**类签名**

类`sqlalchemy.engine.interfaces.ReflectedForeignKeyConstraint` (`builtins.dict`)

```py
attribute constrained_columns: List[str]
```

构成外键的本地列名称

```py
attribute options: NotRequired[Dict[str, Any]]
```

这个外键约束检测到了额外的选项

```py
attribute referred_columns: List[str]
```

对应于 `constrained_columns` 的被引用列名称

```py
attribute referred_schema: str | None
```

被引用表的架构名称

```py
attribute referred_table: str
```

被引用表的名称

```py
class sqlalchemy.engine.interfaces.ReflectedIdentity
```

表示列的反射 IDENTITY 结构，对应于 `Identity` 结构。

`ReflectedIdentity` 结构是 `ReflectedColumn` 结构的一部分，由 `Inspector.get_columns()` 方法返回。

**成员**

always, cache, cycle, increment, maxvalue, minvalue, nomaxvalue, nominvalue, on_null, order, start

**类签名**

类`sqlalchemy.engine.interfaces.ReflectedIdentity`（`builtins.dict`）

```py
attribute always: bool
```

身份列的类型

```py
attribute cache: int | None
```

预先计算的序列中的未来值的数量。

```py
attribute cycle: bool
```

允许序列在达到最大值或最小值时环绕。

```py
attribute increment: int
```

序列的增量值

```py
attribute maxvalue: int
```

序列的最大值。

```py
attribute minvalue: int
```

序列的最小值。

```py
attribute nomaxvalue: bool
```

没有序列的最大值。

```py
attribute nominvalue: bool
```

没有序列的最小值。

```py
attribute on_null: bool
```

表示 ON NULL

```py
attribute order: bool
```

如果为真，则渲染 ORDER 关键字。

```py
attribute start: int
```

序列的起始索引

```py
class sqlalchemy.engine.interfaces.ReflectedIndex
```

表示与`Index`对应的反射元素的字典。

`ReflectedIndex` 结构由 `Inspector.get_indexes()` 方法返回。

**成员**

column_names、column_sorting、dialect_options、duplicates_constraint、expressions、include_columns、name、unique

**类签名**

类`sqlalchemy.engine.interfaces.ReflectedIndex`（`builtins.dict`）

```py
attribute column_names: List[str | None]
```

索引引用的列名。如果列表中的元素是表达式，则为`None`，并在`expressions`列表中返回。

```py
attribute column_sorting: NotRequired[Dict[str, Tuple[str]]]
```

可选字典，将列名或表达式映射到排序关键字元组，可能包括`asc`、`desc`、`nulls_first`、`nulls_last`。

新版本 1.3.5 中的内容。

```py
attribute dialect_options: NotRequired[Dict[str, Any]]
```

检测到的此索引的附加方言特定选项

```py
attribute duplicates_constraint: NotRequired[str | None]
```

指示此索引是否反映了具有此名称的约束

```py
attribute expressions: NotRequired[List[str]]
```

组成索引的表达式。当存在时，此列表包含普通列名（也在`column_names`中）和表达式（在`column_names`中为`None`）。

```py
attribute include_columns: NotRequired[List[str]]
```

在支持的数据库中包含在 INCLUDE 子句中的列。

自版本 2.0 起已弃用：遗留值，将被替换为`index_dict["dialect_options"]["<dialect name>_include"]`

```py
attribute name: str | None
```

索引名称

```py
attribute unique: bool
```

索引是否具有唯一标志

```py
class sqlalchemy.engine.interfaces.ReflectedPrimaryKeyConstraint
```

表示与`PrimaryKeyConstraint`对应的反射元素的字典。

`ReflectedPrimaryKeyConstraint` 结构由 `Inspector.get_pk_constraint()` 方法返回。

**成员**

constrained_columns, dialect_options

**类签名**

类 `sqlalchemy.engine.interfaces.ReflectedPrimaryKeyConstraint` (`builtins.dict`)

```py
attribute constrained_columns: List[str]
```

组成主键的列名

```py
attribute dialect_options: NotRequired[Dict[str, Any]]
```

检测到针对此主键的其他方言特定选项

```py
class sqlalchemy.engine.interfaces.ReflectedUniqueConstraint
```

表示与 `UniqueConstraint` 对应的反映元素的字典。

`ReflectedUniqueConstraint` 结构由 `Inspector.get_unique_constraints()` 方法返回。

**成员**

column_names, dialect_options, duplicates_index

**类签名**

类 `sqlalchemy.engine.interfaces.ReflectedUniqueConstraint` (`builtins.dict`)

```py
attribute column_names: List[str]
```

组成唯一约束的列名

```py
attribute dialect_options: NotRequired[Dict[str, Any]]
```

检测到针对此唯一约束的其他方言特定选项

```py
attribute duplicates_index: NotRequired[str | None]
```

指示此唯一约束是否重复了具有此名称的索引

```py
class sqlalchemy.engine.interfaces.ReflectedTableComment
```

表示与 `Table.comment` 属性对应的反映注释的字典。

`ReflectedTableComment` 结构由 `Inspector.get_table_comment()` 方法返回。

**成员**

text

**类签名**

类 `sqlalchemy.engine.interfaces.ReflectedTableComment` (`builtins.dict`)

```py
attribute text: str | None
```

注释的文本  ## 使用与数据库无关的类型反射

当表的列被反映时，可以使用 `Table.autoload_with` 参数或 `Inspector.get_columns()` 方法，通过 `Table` 或 `Inspector`，数据类型将尽可能与目标数据库特定。这意味着，如果从 MySQL 数据库反映出一个“integer”数据类型，则该类型将由 `sqlalchemy.dialects.mysql.INTEGER` 类表示，其中包括 MySQL 特定属性，如“display_width”。或者在 PostgreSQL 上，可能返回 PostgreSQL 特定的数据类型，如 `sqlalchemy.dialects.postgresql.INTERVAL` 或 `sqlalchemy.dialects.postgresql.ENUM`。

反映的一个使用案例是将给定的 `Table` 转移到不同的供应商数据库。为了适应这种使用情况，有一种技术，可以将这些供应商特定的数据类型即时转换为 SQLAlchemy 后端不可知数据类型的实例，例如上面的类型，如 `Integer`、`Interval` 和 `Enum`。这可以通过拦截列反映并使用 `DDLEvents.column_reflect()` 事件与 `TypeEngine.as_generic()` 方法来实现。

给定 MySQL 中的一个表（选择 MySQL 是因为 MySQL 有很多特定于供应商的数据类型和选项）：

```py
CREATE  TABLE  IF  NOT  EXISTS  my_table  (
  id  INTEGER  PRIMARY  KEY  AUTO_INCREMENT,
  data1  VARCHAR(50)  CHARACTER  SET  latin1,
  data2  MEDIUMINT(4),
  data3  TINYINT(2)
)
```

上述表包括仅限于 MySQL 的整数类型 `MEDIUMINT` 和 `TINYINT`，以及一个包含 MySQL 专有 `CHARACTER SET` 选项的 `VARCHAR`。如果我们正常反映这个表，它将生成一个包含那些 MySQL 特定数据类型和选项的 `Table` 对象。

```py
>>> from sqlalchemy import MetaData, Table, create_engine
>>> mysql_engine = create_engine("mysql+mysqldb://scott:tiger@localhost/test")
>>> metadata_obj = MetaData()
>>> my_mysql_table = Table("my_table", metadata_obj, autoload_with=mysql_engine)
```

上述示例将上述表模式反映到一个新的 `Table` 对象中。然后，我们可以出于演示目的，使用 `CreateTable` 构造打印出特定于 MySQL 的“CREATE TABLE”语句：

```py
>>> from sqlalchemy.schema import CreateTable
>>> print(CreateTable(my_mysql_table).compile(mysql_engine))
CREATE  TABLE  my_table  (
id  INTEGER(11)  NOT  NULL  AUTO_INCREMENT,
data1  VARCHAR(50)  CHARACTER  SET  latin1,
data2  MEDIUMINT(4),
data3  TINYINT(2),
PRIMARY  KEY  (id)
)ENGINE=InnoDB  DEFAULT  CHARSET=utf8mb4 
```

在上面的例子中，保留了特定于 MySQL 的数据类型和选项。如果我们想要一个能够干净地转移到另一个数据库供应商的 `Table`，并且用 `Integer` 替换特殊数据类型 `sqlalchemy.dialects.mysql.MEDIUMINT` 和 `sqlalchemy.dialects.mysql.TINYINT`，我们可以选择在此表上“泛型化”数据类型，或以任何我们喜欢的方式进行更改，通过使用 `DDLEvents.column_reflect()` 事件建立一个处理程序。自定义处理程序将使用 `TypeEngine.as_generic()` 方法将上述 MySQL 特定类型对象转换为通用类型，方法是通过将传递给事件处理程序的列字典条目中的 `"type"` 条目替换为泛型。此字典的格式在 `Inspector.get_columns()` 中描述：

```py
>>> from sqlalchemy import event
>>> metadata_obj = MetaData()

>>> @event.listens_for(metadata_obj, "column_reflect")
... def genericize_datatypes(inspector, tablename, column_dict):
...     column_dict["type"] = column_dict["type"].as_generic()

>>> my_generic_table = Table("my_table", metadata_obj, autoload_with=mysql_engine)
```

现在我们得到了一个新的通用 `Table` 并使用 `Integer` 作为那些数据类型。我们现在可以在 PostgreSQL 数据库上发出一个“CREATE TABLE”语句，例如：

```py
>>> pg_engine = create_engine("postgresql+psycopg2://scott:tiger@localhost/test", echo=True)
>>> my_generic_table.create(pg_engine)
CREATE  TABLE  my_table  (
  id  SERIAL  NOT  NULL,
  data1  VARCHAR(50),
  data2  INTEGER,
  data3  INTEGER,
  PRIMARY  KEY  (id)
) 
```

还需要注意的是，SQLAlchemy 通常会对其他行为做出合理的猜测，例如，MySQL 的 `AUTO_INCREMENT` 指令在 PostgreSQL 中最接近地使用 `SERIAL` 自增数据类型表示。

版本 1.4 新增了 `TypeEngine.as_generic()` 方法，并进一步改进了 `DDLEvents.column_reflect()` 事件的使用，以便方便地应用于 `MetaData` 对象。

## 反射的局限性

需要注意的是，反射过程仅使用在关系数据库中表示的信息重新创建 `Table` 元数据。根据定义，这个过程无法恢复数据库中实际未存储的模式方面。反射无法获取的状态包括但不限于：

+   客户端默认值，即使用 `Column` 的 `default` 关键字定义的 Python 函数或 SQL 表达式（请注意，这与通过反射获得的 `server_default` 是分开的）。

+   列信息，例如可能放入 `Column.info` 字典中的数据

+   `.quote` 设置对于 `Column` 或 `Table` 的价值。

+   特定 `Sequence` 与给定 `Column` 的关联

在许多情况下，关系数据库报告的表元数据格式与 SQLAlchemy 中指定的格式不同。从反射返回的 `Table` 对象不能始终依赖于生成与原始 Python 定义的 `Table` 对象相同的 DDL。发生这种情况的地方包括服务器默认值、与列关联的序列以及有关约束和数据类型的各种特殊情况。服务器端默认值可能会带有转换指令（通常 PostgreSQL 将包括一个 `::<type>` 转换）或不同于最初指定的引号模式。

另一类限制包括反射仅部分或尚未定义的模式结构。最近对反射的改进允许反映视图、索引和外键选项等内容。截至本文撰写时，像 CHECK 约束、表注释和触发器等结构并未反映。

## 覆盖反射列

在反射表时，可以使用显式值覆盖单个列；这对于指定自定义数据类型、在数据库中未配置的主键等约束非常方便：

```py
>>> mytable = Table(
...     "mytable",
...     metadata_obj,
...     Column(
...         "id", Integer, primary_key=True
...     ),  # override reflected 'id' to have primary key
...     Column("mydata", Unicode(50)),  # override reflected 'mydata' to be Unicode
...     # additional Column objects which require no change are reflected normally
...     autoload_with=some_engine,
... )
```

另请参阅

使用自定义类型和反射 - 演示了上述列覆盖技术如何应用于使用自定义数据类型进行表反射。

## 反射视图

反射系统也可以反映视图。基本用法与表相同：

```py
my_view = Table("some_view", metadata, autoload_with=engine)
```

在上面，`my_view`是一个`Table`对象，其中包含代表视图“some_view”中每个列的名称和类型的`Column`对象。

通常，在反射视图时，至少希望有一个主键约束，如果可能的话还有外键。视图反射不会推断这些约束。

使用“override”技术，明确指定那些是主键或具有外键约束的列：

```py
my_view = Table(
    "some_view",
    metadata,
    Column("view_id", Integer, primary_key=True),
    Column("related_thing", Integer, ForeignKey("othertable.thing_id")),
    autoload_with=engine,
)
```

## 一次性反射所有表

`MetaData`对象还可以获取表列表并反射完整集合。通过使用`reflect()`方法实现。调用后，所有定位的表都存在于`MetaData`对象的表字典中：

```py
metadata_obj = MetaData()
metadata_obj.reflect(bind=someengine)
users_table = metadata_obj.tables["users"]
addresses_table = metadata_obj.tables["addresses"]
```

`metadata.reflect()`还提供了一种方便的方法来清除或删除数据库中的所有行：

```py
metadata_obj = MetaData()
metadata_obj.reflect(bind=someengine)
with someengine.begin() as conn:
    for table in reversed(metadata_obj.sorted_tables):
        conn.execute(table.delete())
```

## 从其他模式反射表

章节指定模式名称介绍了表模式的概念，这是数据库中包含表和其他对象的命名空间，并且可以明确指定。可以使用`Table.schema`参数为`Table`对象以及其他对象如视图、索引和序列设置“模式”，还可以使用`MetaData.schema`参数为`MetaData`对象设置默认模式。

此模式参数的使用直接影响表反射功能在被要求反射对象时查找的位置。例如，给定一个通过其`MetaData.schema`参数配置了默认模式名称“project”的`MetaData`对象：

```py
>>> metadata_obj = MetaData(schema="project")
```

`MetaData.reflect()`然后将利用配置的`.schema`进行反射：

```py
>>> # uses `schema` configured in metadata_obj
>>> metadata_obj.reflect(someengine)
```

最终结果是，“project”模式中的`Table`对象将被反射，并且它们将以该名称的模式限定形式填充：

```py
>>> metadata_obj.tables["project.messages"]
Table('messages', MetaData(), Column('message_id', INTEGER(), table=<messages>), schema='project')
```

同样，如果 `Table` 对象中包含了 `Table.schema` 参数，那么该表也将从该数据库模式中反映出来，覆盖了可能已在拥有的 `MetaData` 集合上配置的任何默认模式：

```py
>>> messages = Table("messages", metadata_obj, schema="project", autoload_with=someengine)
>>> messages
Table('messages', MetaData(), Column('message_id', INTEGER(), table=<messages>), schema='project')
```

最后，`MetaData.reflect()` 方法本身也允许传递一个 `MetaData.reflect.schema` 参数，因此我们也可以为默认配置的 `MetaData` 对象从“project”模式加载表：

```py
>>> metadata_obj = MetaData()
>>> metadata_obj.reflect(someengine, schema="project")
```

我们可以使用不同的 `MetaData.schema` 参数（或者不使用任何参数）多次调用 `MetaData.reflect()` 方法，以便继续向 `MetaData` 对象中添加更多对象：

```py
>>> # add tables from the "customer" schema
>>> metadata_obj.reflect(someengine, schema="customer")
>>> # add tables from the default schema
>>> metadata_obj.reflect(someengine)
```

### 带有默认模式的模式限定反射的交互

最佳实践总结部分

在本节中，我们讨论了 SQLAlchemy 关于数据库会话中“默认模式”中可见表的反射行为，以及这些与显式包含模式的 SQLAlchemy 指令的交互方式。 作为最佳实践，请确保数据库的“默认”模式只是一个单一名称，而不是名称列表; 对于属于此“默认”模式并且可以在 DDL 和 SQL 中无需模式限定名称的表，将相应的 `Table.schema` 和类似的模式参数设置为其默认值 `None`。 

如 使用 MetaData 指定默认模式名称 中所述，具有模式概念的数据库通常也包括“默认”模式的概念。 这自然是因为，当一个通常的表对象没有模式时，具有模式的数据库仍然会认为该表在某处的“模式”中。 一些数据库（如 PostgreSQL）进一步将此概念扩展为“模式搜索路径”的概念，其中可以在特定数据库会话中将 *多个* 模式名称视为“隐式”; 指的是任何这些模式中的表名称将不需要模式名称存在（同时，如果模式名称存在，也是完全可以的）。

由于大多数关系数据库都有一个特定的表对象的概念，可以以模式限定的方式引用它，以及一个“隐式”的方式，其中没有模式存在，这为 SQLAlchemy 的反射特性带来了复杂性。以模式限定的方式反映表将始终填充其`Table.schema`属性，并且还会影响如何将此`Table`组织到`MetaData.tables`集合中，即以模式限定的方式。相反，以非模式限定的方式反映**相同的**表将在不模式限定的情况下将其组织到`MetaData.tables`集合中。最终的结果是，在实际数据库中，单一的`MetaData`集合中会有两个单独的`Table`对象，表示相同的表。

为了说明这个问题的影响，考虑上一个示例中来自“project”模式的表，并假设“project”模式是我们数据库连接的默认模式，或者如果使用诸如 PostgreSQL 之类的数据库，则假设“project”模式在 PostgreSQL 中设置了`search_path`。这意味着数据库接受以下两个 SQL 语句是等价的：

```py
-- schema qualified
SELECT  message_id  FROM  project.messages

-- non-schema qualified
SELECT  message_id  FROM  messages
```

这不是一个问题，因为可以双向找到表。但是在 SQLAlchemy 中，是`Table`对象的**标识**决定了它在 SQL 语句中的语义角色。根据 SQLAlchemy 当前的决定，这意味着如果我们以模式限定和非模式限定的方式同时反映同一个“messages”表，我们会得到**两个**`Table`对象，它们**不会**被视为语义上等价：

```py
>>> # reflect in non-schema qualified fashion
>>> messages_table_1 = Table("messages", metadata_obj, autoload_with=someengine)
>>> # reflect in schema qualified fashion
>>> messages_table_2 = Table(
...     "messages", metadata_obj, schema="project", autoload_with=someengine
... )
>>> # two different objects
>>> messages_table_1 is messages_table_2
False
>>> # stored in two different ways
>>> metadata.tables["messages"] is messages_table_1
True
>>> metadata.tables["project.messages"] is messages_table_2
True
```

上述问题在反映的表包含对其他表的外键引用时变得更加复杂。假设“messages”有一个“project_id”列，它引用另一个模式本地表“projects”的行，这意味着“messages”表定义的一部分是一个`ForeignKeyConstraint`对象。

我们可能会发现自己处于这样一种情况：一个`MetaData`集合可能包含多达四个`Table`对象，代表这两个数据库表，其中一个或两个附加表是由反射过程生成的；这是因为当反射过程遇到一个正在被反射的表上的外键约束时，它会分支出去反射那个被引用的表。它用于为这个被引用的表分配模式的决策是，如果拥有的`Table`也省略了其模式名称，并且这两个对象位于同一模式中，那么 SQLAlchemy 将**省略默认模式**的反射`ForeignKeyConstraint`对象，但如果没有省略，则**包括**它。

常见情况是以模式合格的方式反映表，然后以同样的方式加载相关表：

```py
>>> # reflect "messages" in a schema qualified fashion
>>> messages_table_1 = Table(
...     "messages", metadata_obj, schema="project", autoload_with=someengine
... )
```

上述的 `messages_table_1` 也会以模式合格的方式引用 `projects`。这个 `projects` 表会自动反射，因为 “messages” 引用了它：

```py
>>> messages_table_1.c.project_id
Column('project_id', INTEGER(), ForeignKey('project.projects.project_id'), table=<messages>)
```

如果代码的其他部分以非模式合格的方式反映“projects”，现在就有了两个不同的 projects 表：

```py
>>> # reflect "projects" in a non-schema qualified fashion
>>> projects_table_1 = Table("projects", metadata_obj, autoload_with=someengine)
```

```py
>>> # messages does not refer to projects_table_1 above
>>> messages_table_1.c.project_id.references(projects_table_1.c.project_id)
False
```

```py
>>> # it refers to this one
>>> projects_table_2 = metadata_obj.tables["project.projects"]
>>> messages_table_1.c.project_id.references(projects_table_2.c.project_id)
True
```

```py
>>> # they're different, as one non-schema qualified and the other one is
>>> projects_table_1 is projects_table_2
False
```

上述混淆可能会在使用表反射加载应用程序级别`Table`对象的应用程序中造成问题，以及在迁移场景中，特别是在使用 Alembic 迁移检测新表和外键约束时。

可以通过坚持一个简单的做法来纠正上述行为：

+   对于任何期望位于数据库的**默认**模式中的`Table`，不要包含`Table.schema`参数。

对于支持模式的“搜索”路径的 PostgreSQL 和其他数据库，请添加以下附加做法：

+   将“搜索路径”限制为**一个模式，即默认模式**。

另请参阅

远程模式表反射和 PostgreSQL search_path - 关于 PostgreSQL 数据库的此行为的附加详细信息。### 模式合格反射与默认模式的交互

最佳实践概述部分

在本节中，我们将讨论 SQLAlchemy 在数据库会话的“默认模式”中可见的表的反射行为，以及这些表如何与显式包含模式的 SQLAlchemy 指令进行交互。作为最佳实践，请确保数据库的“默认”模式只是一个单一的名称，而不是名称列表；对于属于此“默认”模式且可以在 DDL 和 SQL 中不带模式限定命名的表，将相应的 `Table.schema` 和类似的模式参数设置为它们的默认值 `None`。

如在使用 MetaData 指定默认模式名称中描述的那样，具有模式概念的数据库通常还包括“默认”模式的概念。这自然是因为当人们引用常见的无模式表对象时，具有模式功能的数据库仍会认为该表位于某个“模式”中。一些数据库，如 PostgreSQL，将这个概念进一步发展成为[模式搜索路径](https://www.postgresql.org/docs/current/static/ddl-schemas.html#DDL-SCHEMAS-PATH)的概念，其中一个特定数据库会话中可以考虑*多个*模式名称为“隐式”；引用任何这些模式中的表名都不需要模式名（同时如果模式名*存在*也完全可以）。

因此，由于大多数关系数据库都有一种特定的表对象的概念，既可以以模式限定的方式引用，也可以以“隐式”方式引用，其中不需要模式，这给 SQLAlchemy 的反射特性带来了复杂性。以模式限定的方式反映表将始终填充其 `Table.schema` 属性，并且另外影响到这个 `Table` 如何以模式限定的方式组织到 `MetaData.tables` 集合中。相反，以非模式限定的方式反映**相同的**表将以不带模式的方式组织到 `MetaData.tables` 集合中。最终结果是，在实际数据库中表示同一张表的单个 `MetaData` 集合中将有两个单独的 `Table` 对象。

为了说明这个问题的后果，考虑前面示例中“project”模式中的表，并假设“project”模式是我们数据库连接的默认模式，或者如果使用像 PostgreSQL 这样的数据库，假设“project”模式设置在 PostgreSQL 的`search_path`中。这意味着数据库接受以下两个 SQL 语句作为等价：

```py
-- schema qualified
SELECT  message_id  FROM  project.messages

-- non-schema qualified
SELECT  message_id  FROM  messages
```

这并不是一个问题，因为表可以以两种方式找到。然而，在 SQLAlchemy 中，是`Table`对象的**标识**决定了它在 SQL 语句中的语义角色。根据 SQLAlchemy 当前的决策，这意味着如果我们以模式限定和非模式限定的方式反射相同的“messages”表，我们会得到**两个**不会被视为语义等价的`Table`对象：

```py
>>> # reflect in non-schema qualified fashion
>>> messages_table_1 = Table("messages", metadata_obj, autoload_with=someengine)
>>> # reflect in schema qualified fashion
>>> messages_table_2 = Table(
...     "messages", metadata_obj, schema="project", autoload_with=someengine
... )
>>> # two different objects
>>> messages_table_1 is messages_table_2
False
>>> # stored in two different ways
>>> metadata.tables["messages"] is messages_table_1
True
>>> metadata.tables["project.messages"] is messages_table_2
True
```

当被反射的表包含对其他表的外键引用时，上述问题变得更加复杂。假设“messages”有一个“project_id”列，它引用另一个模式本地表“projects”，这意味着“messages”表的定义中包含一个`ForeignKeyConstraint`对象。

我们可能会发现自己处于这样一种情况，一个`MetaData`集合可能包含代表这两个数据库表的四个`Table`对象，其中一个或两个额外的表是由反射过程生成的；这是因为当反射过程遇到被反射表上的外键约束时，它会分支出去反射该引用表。它用于为这个引用表分配模式的决策是，如果拥有的`Table`也省略了它的模式名称，那么 SQLAlchemy 将**省略默认模式**从反射的`ForeignKeyConstraint`对象中，如果这两个对象在同一个模式中，则**包括**它，但如果没有被省略的话。

常见情况是以模式限定方式反射表，然后以模式限定方式加载相关表：

```py
>>> # reflect "messages" in a schema qualified fashion
>>> messages_table_1 = Table(
...     "messages", metadata_obj, schema="project", autoload_with=someengine
... )
```

上述`messages_table_1`也将以模式限定方式引用`projects`。这个`projects`表将被自动反射，因为“messages”引用了它：

```py
>>> messages_table_1.c.project_id
Column('project_id', INTEGER(), ForeignKey('project.projects.project_id'), table=<messages>)
```

如果代码的其他部分以非模式限定方式反射“projects”，那么现在有两个不同的 projects 表：

```py
>>> # reflect "projects" in a non-schema qualified fashion
>>> projects_table_1 = Table("projects", metadata_obj, autoload_with=someengine)
```

```py
>>> # messages does not refer to projects_table_1 above
>>> messages_table_1.c.project_id.references(projects_table_1.c.project_id)
False
```

```py
>>> # it refers to this one
>>> projects_table_2 = metadata_obj.tables["project.projects"]
>>> messages_table_1.c.project_id.references(projects_table_2.c.project_id)
True
```

```py
>>> # they're different, as one non-schema qualified and the other one is
>>> projects_table_1 is projects_table_2
False
```

上述混淆可能会在使用表反射加载应用级别`Table`对象的应用程序内以及在迁移方案中引起问题，特别是在使用 Alembic Migrations 检测新表和外键约束时。

以上行为可以通过坚持一个简单的做法来纠正：

+   不要为任何期望位于数据库**默认模式**中的`Table`包括`Table.schema`参数。

对于支持“搜索”模式的 PostgreSQL 和其他数据库，添加以下额外的做法：

+   将“搜索路径”限制为**仅一个模式，即默认模式**。

另请参阅

远程模式表内省和 PostgreSQL search_path - 关于 PostgreSQL 数据库的此行为的附加细节。

## 使用检查员进行细粒度反射

也提供了低级接口，它提供了一个与后端无关的系统，用于从给定数据库加载模式、表、列和约束描述的列表。这被称为“检查员”：

```py
from sqlalchemy import create_engine
from sqlalchemy import inspect

engine = create_engine("...")
insp = inspect(engine)
print(insp.get_table_names())
```

| 对象名称 | 描述 |
| --- | --- |
| 检查员 | 执行数据库模式检查。 |
| ReflectedCheckConstraint | 表示反射元素的字典，对应于`CheckConstraint`。 |
| ReflectedColumn | 表示反射元素的字典，对应于`Column`对象。 |
| ReflectedComputed | 表示计算列的反射元素，对应于`Computed`构造。 |
| ReflectedForeignKeyConstraint | 表示反射元素的字典，对应于`ForeignKeyConstraint`。 |
| ReflectedIdentity | 表示列的反射身份结构，对应于`Identity`构造。 |
| ReflectedIndex | 表示反射元素的字典，对应于`Index`。 |
| 反射主键约束 | 表示对应于`PrimaryKeyConstraint`的反射元素的字典。 |
| 反射表注释 | 表示对应于`Table.comment`属性的反射注释的字典。 |
| 反射唯一约束 | 表示对应于`UniqueConstraint`的反射元素的字典。 |

```py
class sqlalchemy.engine.reflection.Inspector
```

执行数据库模式检查。

Inspector 充当`Dialect`的反射方法的代理，提供一致的接口以及对先前获取的元数据的缓存支持。

通常通过`inspect()`函数创建`Inspector`对象，可以传递一个`Engine`或一个`Connection`：

```py
from sqlalchemy import inspect, create_engine
engine = create_engine('...')
insp = inspect(engine)
```

在上述情况下，与引擎相关联的`Dialect`可能选择返回一个提供了特定于该方言目标数据库的附加方法的`Inspector`子类。

**成员**

__init__(), bind, clear_cache(), default_schema_name, dialect, engine, from_engine(), get_check_constraints(), get_columns(), get_foreign_keys(), get_indexes(), get_materialized_view_names(), get_multi_check_constraints(), get_multi_columns(), get_multi_foreign_keys(), get_multi_indexes(), get_multi_pk_constraint(), get_multi_table_comment(), get_multi_table_options(), get_multi_unique_constraints(), get_pk_constraint(), get_schema_names(), get_sequence_names(), get_sorted_table_and_fkc_names(), get_table_comment(), get_table_names(), get_table_options(), get_temp_table_names(), get_temp_view_names(), get_unique_constraints(), get_view_definition(), get_view_names(), has_index(), has_schema(), has_sequence(), has_table(), info_cache, reflect_table(), sort_tables_on_foreign_key_dependency()

**类签名**

类`sqlalchemy.engine.reflection.Inspector`（`sqlalchemy.inspection.Inspectable`）

```py
method __init__(bind: Engine | Connection)
```

初始化一个新的`Inspector`。

自版本 1.4 弃用：`Inspector` 上的 __init__() 方法已弃用，并将在将来的版本中移除。请使用 `Engine` 或 `Connection` 上的 `inspect()` 函数以获取 `Inspector`。

参数：

**bind** – 一个`Connection`，通常是`Engine`或`Connection`的实例。

对于特定于方言的 `Inspector` 实例，请参阅 `Inspector.from_engine()`。

```py
attribute bind: Engine | Connection
```

```py
method clear_cache() → None
```

重置此`Inspector`的缓存。

当检查方法有缓存数据时，在下次调用以获取新数据时会发出 SQL 查询。

从版本 2.0 开始。

```py
attribute default_schema_name
```

返回当前引擎的数据库用户的方言提供的默认模式名称。

例如，对于 PostgreSQL 通常是 `public`，对于 SQL Server 是 `dbo`。

```py
attribute dialect: Dialect
```

```py
attribute engine: Engine
```

```py
classmethod from_engine(bind: Engine) → Inspector
```

从给定的引擎或连接构造一个新的特定于方言的 Inspector 对象。

自版本 1.4 弃用：`Inspector` 上的 from_engine() 方法已弃用，并将在将来的版本中移除。请使用 `Engine` 或 `Connection` 上的 `inspect()` 函数以获取 `Inspector`。

参数：

**bind** – 一个`Connection`或者`Engine`。

该方法与直接构造函数调用`Inspector`不同，在此，`Dialect`有机会提供特定于方言的`Inspector`实例，该实例可能提供附加方法。

请参阅`Inspector`的示例。

```py
method get_check_constraints(table_name: str, schema: str | None = None, **kw: Any) → List[ReflectedCheckConstraint]
```

返回`table_name`中的检查约束信息。

给定字符串`table_name`和可选字符串模式，将检查约束信息作为`ReflectedCheckConstraint`列表返回。

参数：

+   `table_name` – 表的字符串名称。对于特殊引用，请使用`quoted_name`。

+   `schema` – 字符串模式名称；如果省略，则使用数据库连接的默认模式。对于特殊引用，请使用`quoted_name`。

+   `**kw` – 要传递给特定方言实现的附加关键字参数。有关更多信息，请参阅使用中的方言的文档。

返回：

字典列表，每个表示检查约束的定义。

另请参阅

`Inspector.get_multi_check_constraints()`

```py
method get_columns(table_name: str, schema: str | None = None, **kw: Any) → List[ReflectedColumn]
```

返回`table_name`中的列信息。

给定字符串`table_name`和可选字符串`schema`，将列信息作为`ReflectedColumn`列表返回。

参数：

+   `table_name` – 表的字符串名称。对于特殊引用，请使用`quoted_name`。

+   `schema` – 字符串模式名称；如果省略，则使用数据库连接的默认模式。对于特殊引用，请使用`quoted_name`。

+   `**kw` – 要传递给特定方言实现的附加关键字参数。有关更多信息，请参阅使用中的方言的文档。

返回：

字典列表，每个表示数据库列的定义。

另请参阅

`Inspector.get_multi_columns()`。

```py
method get_foreign_keys(table_name: str, schema: str | None = None, **kw: Any) → List[ReflectedForeignKeyConstraint]
```

返回`table_name`中的外键信息。

给定字符串`table_name`，以及可选的字符串模式，将外键信息作为`ReflectedForeignKeyConstraint`的列表返回。

参数：

+   `table_name` – 表格的字符串名称。对于特殊引用，请使用`quoted_name`。

+   `schema` – 字符串模式名称；如果省略，则使用数据库连接的默认模式。对于特殊引用，请使用`quoted_name`。

+   `**kw` – 附加的关键字参数，传递给特定方言的实现。有关更多信息，请参阅正在使用的方言的文档。

返回：

一个字典列表，每个字典表示一个外键定义。

另请参阅

`Inspector.get_multi_foreign_keys()`

```py
method get_indexes(table_name: str, schema: str | None = None, **kw: Any) → List[ReflectedIndex]
```

返回有关`table_name`中索引的信息。

给定字符串`table_name`和可选的字符串模式，将索引信息作为`ReflectedIndex`的列表返回。

参数：

+   `table_name` – 表格的字符串名称。对于特殊引用，请使用`quoted_name`。

+   `schema` – 字符串模式名称；如果省略，则使用数据库连接的默认模式。对于特殊引用，请使用`quoted_name`。

+   `**kw` – 附加的关键字参数，传递给特定方言的实现。有关更多信息，请参阅正在使用的方言的文档。

返回：

一个字典列表，每个字典表示一个索引的定义。

另请参阅

`Inspector.get_multi_indexes()`

```py
method get_materialized_view_names(schema: str | None = None, **kw: Any) → List[str]
```

返回模式中的所有物化视图名称。

参数：

+   `schema` – 可选，从非默认模式中检索名称。对于特殊引用，请使用`quoted_name`。

+   `**kw` – 附加的关键字参数，传递给特定方言的实现。有关更多信息，请参阅正在使用的方言的文档。

版本 2.0 中的新功能。

另请参阅

`Inspector.get_view_names()`

```py
method get_multi_check_constraints(schema: str | None = None, filter_names: Sequence[str] | None = None, kind: ObjectKind = ObjectKind.TABLE, scope: ObjectScope = ObjectScope.DEFAULT, **kw: Any) → Dict[TableKey, List[ReflectedCheckConstraint]]
```

返回给定模式中所有表格中检查约束的信息。

可以通过将要使用的名称传递给`filter_names`来过滤表格。

对于每个表，值是`ReflectedCheckConstraint`的列表。

参数：

+   `schema` – 字符串模式名称；如果省略，则使用数据库连接的默认模式。对于特殊引用，请使用`quoted_name`。

+   `filter_names` – 可选地仅返回此处列出的对象的信息。

+   `kind` – 指定要反映的对象类型的`ObjectKind`。默认为`ObjectKind.TABLE`。

+   `scope` – 指定应反映默认、临时或任何表的约束的`ObjectScope`。默认为`ObjectScope.DEFAULT`。

+   `**kw` – 传递给特定方言实现的额外关键字参数。有关更多信息，请参阅正在使用的方言的文档。

返回：

一个字典，其中键是两元组模式、表名，值是字典列表，每个表示检查约束的定义。如果未提供模式，则模式为`None`。

新版本 2.0 中新增。

另请参阅

`Inspector.get_check_constraints()`

```py
method get_multi_columns(schema: str | None = None, filter_names: Sequence[str] | None = None, kind: ObjectKind = ObjectKind.TABLE, scope: ObjectScope = ObjectScope.DEFAULT, **kw: Any) → Dict[TableKey, List[ReflectedColumn]]
```

返回给定模式中所有对象中列的信息。

可通过将要用于`filter_names`的名称传递来过滤对象。

对于每个表，值是`ReflectedColumn`的列表。

参数：

+   `schema` – 字符串模式名称；如果省略，则使用数据库连接的默认模式。对于特殊引用，请使用`quoted_name`。

+   `filter_names` – 可选地仅返回此处列出的对象的信息。

+   `kind` – 指定要反映的对象类型的`ObjectKind`。默认为`ObjectKind.TABLE`。

+   `scope` – 指定应反映默认、临时或任何表的列的`ObjectScope`。默认为`ObjectScope.DEFAULT`。

+   `**kw` – 传递给特定方言实现的额外关键字参数。有关更多信息，请参阅正在使用的方言的文档。

返回：

一个字典，其中键是两元组模式、表名，值是字典列表，每个表示数据库列的定义。如果未提供模式，则模式为`None`。

新版本 2.0 中新增。

另请参阅

`Inspector.get_columns()`

```py
method get_multi_foreign_keys(schema: str | None = None, filter_names: Sequence[str] | None = None, kind: ObjectKind = ObjectKind.TABLE, scope: ObjectScope = ObjectScope.DEFAULT, **kw: Any) → Dict[TableKey, List[ReflectedForeignKeyConstraint]]
```

返回给定模式中所有表中外键的信息。

可通过将要用于`filter_names`的名称传递来过滤表。

对于每个表，值是一个 `ReflectedForeignKeyConstraint` 列表。

参数：

+   `schema` – 字符串模式名称；如果省略，则使用数据库连接的默认模式。要进行特殊引用，请使用 `quoted_name`。

+   `filter_names` – 可选择仅返回此处列出的对象的信息。

+   `kind` – 一个指定要反射的对象类型的 `ObjectKind`。默认为 `ObjectKind.TABLE`。

+   `scope` – 一个指定要反射的默认、临时或任何表的外键的 `ObjectScope`。默认为 `ObjectScope.DEFAULT`。

+   `**kw` – 传递给特定方言实现的额外关键字参数。有关更多信息，请参阅正在使用的方言的文档。

返回：

一个字典，其中键是二元组模式、表名，值是字典列表，每个表示外键定义。如果未提供模式，则模式为 `None`。

新版本 2.0 中新增。

另请参阅

`Inspector.get_foreign_keys()`

```py
method get_multi_indexes(schema: str | None = None, filter_names: Sequence[str] | None = None, kind: ObjectKind = ObjectKind.TABLE, scope: ObjectScope = ObjectScope.DEFAULT, **kw: Any) → Dict[TableKey, List[ReflectedIndex]]
```

返回给定模式中所有对象中的索引的信息。

可通过将名称传递给 `filter_names` 来过滤对象。

对于每个表，值是一个 `ReflectedIndex` 列表。

参数：

+   `schema` – 字符串模式名称；如果省略，则使用数据库连接的默认模式。要进行特殊引用，请使用 `quoted_name`。

+   `filter_names` – 可选择仅返回此处列出的对象的信息。

+   `kind` – 一个指定要反射的对象类型的 `ObjectKind`。默认为 `ObjectKind.TABLE`。

+   `scope` – 一个指定要反射的默认、临时或任何表的索引的 `ObjectScope`。默认为 `ObjectScope.DEFAULT`。

+   `**kw` – 传递给特定方言实现的额外关键字参数。有关更多信息，请参阅正在使用的方言的文档。

返回：

一个字典，其中键是二元组模式、表名，值是字典列表，每个表示索引的定义。如果未提供模式，则模式为 `None`。

新版本 2.0 中新增。

另请参阅

`Inspector.get_indexes()`

```py
method get_multi_pk_constraint(schema: str | None = None, filter_names: Sequence[str] | None = None, kind: ObjectKind = ObjectKind.TABLE, scope: ObjectScope = ObjectScope.DEFAULT, **kw: Any) → Dict[TableKey, ReflectedPrimaryKeyConstraint]
```

返回给定模式中所有表中主键约束的信息。

可通过将名称传递给 `filter_names` 来过滤表。

对于每个表，值是 `ReflectedPrimaryKeyConstraint`。

参数：

+   `schema` – 字符串模式名称；如果省略，则使用数据库连接的默认模式。要进行特殊引用，请使用 `quoted_name`。

+   `filter_names` – 可选地仅返回此处列出的对象的信息。

+   `kind` – 一个 `ObjectKind`，指定要反映的对象类型。默认为 `ObjectKind.TABLE`。

+   `scope` – 一个 `ObjectScope`，指定应反映默认、临时或任何表的主键。默认为 `ObjectScope.DEFAULT`。

+   `**kw` – 传递给方言特定实现的额外关键字参数。有关更多信息，请参阅所使用的方言的文档。

返回：

一个字典，其中键是二元组 schema,table-name，值是字典，每个表示主键约束的定义。如果未提供模式，则模式为 `None`。

2.0 版新功能。

另请参阅

`Inspector.get_pk_constraint()`

```py
method get_multi_table_comment(schema: str | None = None, filter_names: Sequence[str] | None = None, kind: ObjectKind = ObjectKind.TABLE, scope: ObjectScope = ObjectScope.DEFAULT, **kw: Any) → Dict[TableKey, ReflectedTableComment]
```

返回给定模式中所有对象的表注释信息。

可通过将要使用的名称传递给 `filter_names` 进行过滤对象。

对于每个表，值是 `ReflectedTableComment`。

对于不支持注释的方言，引发 `NotImplementedError` 异常。

参数：

+   `schema` – 字符串模式名称；如果省略，则使用数据库连接的默认模式。要进行特殊引用，请使用 `quoted_name`。

+   `filter_names` – 可选地仅返回此处列出的对象的信息。

+   `kind` – 一个 `ObjectKind`，指定要反映的对象类型。默认为 `ObjectKind.TABLE`。

+   `scope` – 一个 `ObjectScope`，指定应反映默认、临时或任何表的注释。默认为 `ObjectScope.DEFAULT`。

+   `**kw` – 传递给方言特定实现的额外关键字参数。有关更多信息，请参阅所使用的方言的文档。

返回：

一个字典，其中键是二元组 schema,table-name，值是字典，表示表注释。如果未提供模式，则模式为 `None`。

2.0 版新功能。

另请参阅

`Inspector.get_table_comment()`

```py
method get_multi_table_options(schema: str | None = None, filter_names: Sequence[str] | None = None, kind: ObjectKind = ObjectKind.TABLE, scope: ObjectScope = ObjectScope.DEFAULT, **kw: Any) → Dict[TableKey, Dict[str, Any]]
```

返回指定模式中的表创建时指定的选项的字典。

表格可以通过将要使用的名称传递给 `filter_names` 进行过滤。

目前包括一些适用于 MySQL 和 Oracle 表的选项。

参数：

+   `schema` – 字符串模式名称；如果省略，则使用数据库连接的默认模式。对于特殊引用，请使用`quoted_name`。

+   `filter_names` – 可选择仅返回此处列出的对象的信息。

+   `kind` – 一个`ObjectKind`，指定要反映的对象类型。默认为`ObjectKind.TABLE`。

+   `scope` – 一个`ObjectScope`，指定应该反映哪些选项的范围，默认为`ObjectScope.DEFAULT`。

+   `**kw` – 传递给方言特定实现的附加关键字参数。有关更多信息，请参阅所使用方言的文档。

返回值：

一个字典，其中键是两元组 schema,table-name，值是具有表选项的字典。每个字典中返回的键取决于所使用的方言。每个键都以方言名称为前缀。如果未提供模式，则模式为`None`。

新版本功能 2.0。

另请参阅

`Inspector.get_table_options()`

```py
method get_multi_unique_constraints(schema: str | None = None, filter_names: Sequence[str] | None = None, kind: ObjectKind = ObjectKind.TABLE, scope: ObjectScope = ObjectScope.DEFAULT, **kw: Any) → Dict[TableKey, List[ReflectedUniqueConstraint]]
```

返回有关给定模式中所有表的唯一约束的信息。

表格可以通过将要使用的名称传递给`filter_names`来进行过滤。

对于每个表，值是一个`ReflectedUniqueConstraint`的列表。

参数：

+   `schema` – 字符串模式名称；如果省略，则使用数据库连接的默认模式。对于特殊引用，请使用`quoted_name`。

+   `filter_names` – 可选择仅返回此处列出的对象的信息。

+   `kind` – 一个`ObjectKind`，指定要反映的对象类型。默认为`ObjectKind.TABLE`。

+   `scope` – 一个`ObjectScope`，指定应该反映哪些约束的范围，默认为`ObjectScope.DEFAULT`。

+   `**kw` – 传递给方言特定实现的附加关键字参数。有关更多信息，请参阅所使用方言的文档。

返回值：

一个字典，其中键是两元组 schema,table-name，值是表示唯一约束定义的字典列表。如果未提供模式，则模式为`None`。

新版本功能 2.0。

另请参阅

`Inspector.get_unique_constraints()`

```py
method get_pk_constraint(table_name: str, schema: str | None = None, **kw: Any) → ReflectedPrimaryKeyConstraint
```

返回有关`table_name`中主键约束的信息。

给定字符串`table_name`，以及一个可选的字符串模式，返回主键信息作为`ReflectedPrimaryKeyConstraint`。

参数：

+   `table_name` - 表的字符串名称。要进行特殊引用，请使用`quoted_name`。

+   `schema` - 字符串模式名称；如果省略，则使用数据库连接的默认模式。要进行特殊引用，请使用`quoted_name`。

+   `**kw` - 传递给特定方言实现的附加关键字参数。有关更多信息，请参阅正在使用的方言的文档。

返回：

一个表示主键约束定义的字典。

另请参阅

`Inspector.get_multi_pk_constraint()`

```py
method get_schema_names(**kw: Any) → List[str]
```

返回所有模式名称。

参数：

****kw** - 传递给特定方言实现的附加关键字参数。有关更多信息，请参阅正在使用的方言的文档。

```py
method get_sequence_names(schema: str | None = None, **kw: Any) → List[str]
```

返回模式中所有序列名称。

参数：

+   `schema` - 可选，从非默认模式中检索名称。要进行特殊引用，请使用`quoted_name`。

+   `**kw` - 传递给特定方言实现的附加关键字参数。有关更多信息，请参阅正在使用的方言的文档。

```py
method get_sorted_table_and_fkc_names(schema: str | None = None, **kw: Any) → List[Tuple[str | None, List[Tuple[str, str | None]]]]
```

返回特定模式中所引用的表和外键约束名的依赖排序。

这将产生 2 元组`(tablename, [(tname, fkname), (tname, fkname), ...])`，其中包含按 CREATE 顺序分组的表名与未被检测为属于循环的外键约束名。最后一个元素将是`(None, [(tname, fkname), (tname, fkname), ..])`，其中包含剩余的外键约束名，这些名字需要在事后单独进行 CREATE 步骤，基于表之间的依赖关系。

参数：

+   `schema` - 要查询的模式名称，如果不是默认模式。

+   `**kw` - 传递给特定方言实现的附加关键字参数。有关更多信息，请参阅正在使用的方言的文档。

另请参阅

`Inspector.get_table_names()`

`sort_tables_and_constraints()` - 与已给定的`MetaData`类似的方法。

```py
method get_table_comment(table_name: str, schema: str | None = None, **kw: Any) → ReflectedTableComment
```

返回关于`table_name`的表注释的信息。

给定字符串 `table_name` 和可选字符串 `schema`，将表注释信息返回为 `ReflectedTableComment`。

对于不支持注释的方言，引发 `NotImplementedError`。

参数：

+   `table_name` – 表的字符串名称。要进行特殊引用，请使用 `quoted_name`。

+   `schema` – 模式名称的字符串；如果省略，将使用数据库连接的默认模式。要进行特殊引用，请使用 `quoted_name`。

+   `**kw` – 传递给特定方言实现的额外关键字参数。有关更多信息，请参阅正在使用的方言的文档。

返回：

一个字典，包含表的注释。

自 1.2 版开始新增。

另请参阅

`Inspector.get_multi_table_comment()`

```py
method get_table_names(schema: str | None = None, **kw: Any) → List[str]
```

返回特定模式内的所有表名称。

名称预期只是实际表，而不是视图。视图使用 `Inspector.get_view_names()` 和/或 `Inspector.get_materialized_view_names()` 方法返回。 

参数：

+   `schema` – 模式名称。如果将 `schema` 留在 `None`，则使用数据库的默认模式，否则搜索命名模式。如果数据库不支持命名模式，则如果不将 `schema` 传递为 `None`，则行为未定义。要进行特殊引用，请使用 `quoted_name`。

+   `**kw` – 传递给特定方言实现的额外关键字参数。有关更多信息，请参阅正在使用的方言的文档。

另请参阅

`Inspector.get_sorted_table_and_fkc_names()`

`MetaData.sorted_tables`

```py
method get_table_options(table_name: str, schema: str | None = None, **kw: Any) → Dict[str, Any]
```

返回在创建给定名称的表时指定的选项字典。

目前包括一些适用于 MySQL 和 Oracle 表的选项。

参数：

+   `table_name` – 表的字符串名称。要进行特殊引用，请使用 `quoted_name`。

+   `schema` – 字符串模式名称；如果省略，则使用数据库连接的默认模式。对于特殊引用，请使用`quoted_name`。

+   `**kw` – 传递给方言特定实现的额外关键字参数。有关更多信息，请参阅正在使用的方言的文档。

返回：

一个带有表选项的字典。返回的键取决于正在使用的方言。每个键都以方言名称为前缀。

另请参阅

`Inspector.get_multi_table_options()`

```py
method get_temp_table_names(**kw: Any) → List[str]
```

返回当前绑定的临时表名称列表。

大多数方言不支持此方法；目前只有 Oracle、PostgreSQL 和 SQLite 实现了它。

参数：

****kw** – 传递给方言特定实现的额外关键字参数。有关更多信息，请参阅正在使用的方言的文档。

```py
method get_temp_view_names(**kw: Any) → List[str]
```

返回当前绑定的临时视图名称列表。

大多数方言不支持此方法；目前只有 PostgreSQL 和 SQLite 实现了它。

参数：

****kw** – 传递给方言特定实现的额外关键字参数。有关更多信息，请参阅正在使用的方言的文档。

```py
method get_unique_constraints(table_name: str, schema: str | None = None, **kw: Any) → List[ReflectedUniqueConstraint]
```

返回`table_name`中唯一约束的信息。

给定一个字符串`table_name`和一个可选的字符串模式，返回`ReflectedUniqueConstraint`的唯一约束信息列表。

参数：

+   `table_name` – 表的字符串名称。对于特殊引用，请使用`quoted_name`。

+   `schema` – 字符串模式名称；如果省略，则使用数据库连接的默认模式。对于特殊引用，请使用`quoted_name`。

+   `**kw` – 传递给方言特定实现的额外关键字参数。有关更多信息，请参阅正在使用的方言的文档。

返回：

一个字典列表，每个代表一个唯一约束的定义。

另请参阅

`Inspector.get_multi_unique_constraints()`

```py
method get_view_definition(view_name: str, schema: str | None = None, **kw: Any) → str
```

返回名为`view_name`的普通或材料化视图的定义。

参数：

+   `view_name` – 视图的名称。

+   `schema` – 可选，从非默认模式中检索名称。对于特殊引用，请使用`quoted_name`。

+   `**kw` – 传递给方言特定实现的额外关键字参数。有关更多信息，请参阅正在使用的方言的文档。

```py
method get_view_names(schema: str | None = None, **kw: Any) → List[str]
```

返回模式中所有非材料化视图名称。

参数：

+   `schema` – 可选，从非默认模式中检索名称。要进行特殊引用，请使用`quoted_name`。

+   `**kw` – 额外的关键字参数，传递给特定方言实现。有关更多信息，请参阅正在使用的方言的文档。

自版本 2.0 起更改：对于以前在此列表中包括材料化视图名称的方言（目前为 PostgreSQL），此方法不再返回材料化视图的名称。应改用`Inspector.get_materialized_view_names()`方法。

另请参见

`Inspector.get_materialized_view_names()`

```py
method has_index(table_name: str, index_name: str, schema: str | None = None, **kw: Any) → bool
```

检查数据库中特定索引名称的存在。

参数：

+   `table_name` – 索引所属的表的名称。

+   `index_name` – 要检查的索引的名称。

+   `schema` – 如果不是默认模式，则要查询的模式名称。

+   `**kw` – 额外的关键字参数，传递给特定方言实现。有关更多信息，请参阅正在使用的方言的文档。

自版本 2.0 起新增。

```py
method has_schema(schema_name: str, **kw: Any) → bool
```

如果后端具有给定名称的模式，则返回 True。

参数：

+   `schema_name` – 要检查的模式的名称。

+   `**kw` – 额外的关键字参数，传递给特定方言实现。有关更多信息，请参阅正在使用的方言的文档。

自版本 2.0 起新增。

```py
method has_sequence(sequence_name: str, schema: str | None = None, **kw: Any) → bool
```

如果后端具有给定名称的序列，则返回 True。

参数：

+   `sequence_name` – 序列的名称。

+   `schema` – 如果不是默认模式，则要查询的模式名称。

+   `**kw` – 额外的关键字参数，传递给特定方言实现。有关更多信息，请参阅正在使用的方言的文档。

自版本 1.4 起新增。

```py
method has_table(table_name: str, schema: str | None = None, **kw: Any) → bool
```

如果后端具有给定名称的表、视图或临时表，则返回 True。

参数：

+   `table_name` – 要检查的表的名称。

+   `schema` – 如果不是默认模式，则要查询的模式名称。

+   `**kw` – 额外的关键字参数，传递给特定方言实现。有关更多信息，请参阅正在使用的方言的文档。

自版本 1.4 起新增：- `Inspector.has_table()` 方法替换了 `Engine.has_table()` 方法。

自版本 2.0 起更改：`Inspector.has_table()` 现在正式支持检查额外的类似表的对象：

+   任何类型的视图（普通或材料化）

+   任何类型的临时表

以前，这两个检查没有正式指定，不同的方言在行为上会有所不同。方言测试套件现在包括所有这些对象类型的测试，并应该受到所有包含在 SQLAlchemy 中的方言的支持。然而，第三方方言中的支持可能滞后。

```py
attribute info_cache: Dict[Any, Any]
```

```py
method reflect_table(table: Table, include_columns: Collection[str] | None, exclude_columns: Collection[str] = (), resolve_fks: bool = True, _extend_on: Set[Table] | None = None, _reflect_info: _ReflectionInfo | None = None) → None
```

给定一个`Table`对象，根据内省加载其内部结构。

这是大多数方言用于生成表反射的基础方法。直接使用方式如下：

```py
from sqlalchemy import create_engine, MetaData, Table
from sqlalchemy import inspect

engine = create_engine('...')
meta = MetaData()
user_table = Table('user', meta)
insp = inspect(engine)
insp.reflect_table(user_table, None)
```

从版本 1.4 开始更改：从`reflecttable`改名为`reflect_table`

参数：

+   `table` – 一个`Table`实例。

+   `include_columns` – 一个包含在反射过程中的字符串列名列表。如果为`None`，则反射所有列。

```py
method sort_tables_on_foreign_key_dependency(consider_schemas: Collection[str | None] = (None,), **kw: Any) → List[Tuple[Tuple[str | None, str] | None, List[Tuple[Tuple[str | None, str], str | None]]]]
```

返回在多个模式中引用的表和外键约束名称的依赖排序。

此方法可以与`Inspector.get_sorted_table_and_fkc_names()`进行比较，后者一次只处理一个模式；在这里，该方法是一个通用方法，将同时考虑多个模式，包括解决跨模式外键。

2.0 版本中的新功能。

```py
class sqlalchemy.engine.interfaces.ReflectedColumn
```

表示与`Column`对象对应的反射元素的字典。

`ReflectedColumn`结构由`get_columns`方法返回。

**成员**

autoincrement, comment, computed, default, dialect_options, identity, name, nullable, type

**类签名**

类`sqlalchemy.engine.interfaces.ReflectedColumn` (`builtins.dict`)

```py
attribute autoincrement: NotRequired[bool]
```

依赖于数据库的自动增量标志。

此标志指示列是否具有某种数据库端的“自动增量”标志。在 SQLAlchemy 中，其他类型的列也可能充当“自动增量”列，而不一定在其上具有这样的标志。

有关“自动增量”的更多背景信息，请参见`Column.autoincrement`。

```py
attribute comment: NotRequired[str | None]
```

如果存在，则为列的注释。只有一些方言返回此键

```py
attribute computed: NotRequired[ReflectedComputed]
```

指示此列是由数据库计算的。只有一些方言返回此键。

版本 1.3.16 中的新功能：- 添加了对计算反射的支持。

```py
attribute default: str | None
```

列的默认表达式作为 SQL 字符串

```py
attribute dialect_options: NotRequired[Dict[str, Any]]
```

检测到的此反射对象的额外方言特定选项

```py
attribute identity: NotRequired[ReflectedIdentity]
```

表示此列是一个 IDENTITY 列。只有一些方言返回此键。

版本 1.4 中的新功能：- 添加了对标识列反射的支持。

```py
attribute name: str
```

列名

```py
attribute nullable: bool
```

列的布尔标志，如果列是 NULL 或 NOT NULL。

```py
attribute type: TypeEngine[Any]
```

作为`TypeEngine`实例表示的列类型。

```py
class sqlalchemy.engine.interfaces.ReflectedComputed
```

表示计算列的反射元素，对应于`Computed`构造。

`ReflectedComputed`结构是`ReflectedColumn`结构的一部分，由`Inspector.get_columns()`方法返回。

**成员**

持久化，sqltext

**类签名**

类`sqlalchemy.engine.interfaces.ReflectedComputed` (`builtins.dict`)

```py
attribute persisted: NotRequired[bool]
```

指示值是存储在表中还是按需计算的

```py
attribute sqltext: str
```

用于生成此列的表达式，以字符串 SQL 表达式返回

```py
class sqlalchemy.engine.interfaces.ReflectedCheckConstraint
```

表示反映与`CheckConstraint`对应的元素的字典。

`ReflectedCheckConstraint`结构由`Inspector.get_check_constraints()`方法返回。

**成员**

方言选项，sqltext

**类签名**

类`sqlalchemy.engine.interfaces.ReflectedCheckConstraint` (`builtins.dict`)

```py
attribute dialect_options: NotRequired[Dict[str, Any]]
```

检测到的此检查约束的额外方言特定选项

版本 1.3.8 中的新功能。

```py
attribute sqltext: str
```

检查约束的 SQL 表达式

```py
class sqlalchemy.engine.interfaces.ReflectedForeignKeyConstraint
```

表示反映的元素的字典，对应于`ForeignKeyConstraint`。

`ReflectedForeignKeyConstraint` 结构由 `Inspector.get_foreign_keys()` 方法返回。

**成员**

constrained_columns，options，referred_columns，referred_schema，referred_table

**类签名**

类`sqlalchemy.engine.interfaces.ReflectedForeignKeyConstraint` (`builtins.dict`)

```py
attribute constrained_columns: List[str]
```

组成外键的本地列名

```py
attribute options: NotRequired[Dict[str, Any]]
```

检测到此外键约束的附加选项

```py
attribute referred_columns: List[str]
```

引用的列名对应于`constrained_columns`。

```py
attribute referred_schema: str | None
```

被引用的表的架构名称

```py
attribute referred_table: str
```

被引用的表的名称

```py
class sqlalchemy.engine.interfaces.ReflectedIdentity
```

表示对应于 `Identity` 构造的反映的 IDENTITY 结构的列。

`ReflectedIdentity` 结构是 `ReflectedColumn` 结构的一部分，由 `Inspector.get_columns()` 方法返回。

**成员**

always，cache，cycle，increment，maxvalue，minvalue，nomaxvalue，nominvalue，on_null，order，start

**类签名**

类`sqlalchemy.engine.interfaces.ReflectedIdentity`（`builtins.dict`）

```py
attribute always: bool
```

标识列的类型

```py
attribute cache: int | None
```

提前计算的序列中的未来值的数量。

```py
attribute cycle: bool
```

允许在达到最大值或最小值时循环。

```py
attribute increment: int
```

序列的增量值

```py
attribute maxvalue: int
```

序列的最大值。

```py
attribute minvalue: int
```

序列的最小值。

```py
attribute nomaxvalue: bool
```

序列的最大值。

```py
attribute nominvalue: bool
```

序列的最小值。

```py
attribute on_null: bool
```

指示 ON NULL

```py
attribute order: bool
```

如果为 true，则呈现 ORDER 关键字。

```py
attribute start: int
```

序列的起始索引

```py
class sqlalchemy.engine.interfaces.ReflectedIndex
```

表示与`Index`相对应的反射元素的字典。

`ReflectedIndex`结构由`Inspector.get_indexes()`方法返回。

**成员**

列名、列排序、方言选项、重复约束、表达式、包含列、名称、唯一

**类签名**

类`sqlalchemy.engine.interfaces.ReflectedIndex`（`builtins.dict`）

```py
attribute column_names: List[str | None]
```

索引引用的列名。此列表的元素如果是表达式，则为`None`，并在`expressions`列表中返回。

```py
attribute column_sorting: NotRequired[Dict[str, Tuple[str]]]
```

可选字典，将列名或表达式映射到排序关键字元组，其中可能包括`asc`、`desc`、`nulls_first`、`nulls_last`。

新版本 1.3.5 中新增。

```py
attribute dialect_options: NotRequired[Dict[str, Any]]
```

此索引的附加特定方言选项

```py
attribute duplicates_constraint: NotRequired[str | None]
```

表示此索引是否镜像了此名称的约束

```py
attribute expressions: NotRequired[List[str]]
```

构成索引的表达式。此列表（当存在时）包含普通列名（也在`column_names`中）和表达式（在`column_names`中为`None`）。

```py
attribute include_columns: NotRequired[List[str]]
```

包含在支持数据库的 INCLUDE 子句中的列。

自版本 2.0 开始弃用：遗留值，将被`index_dict["dialect_options"]["<dialect name>_include"]`替换。

```py
attribute name: str | None
```

索引名称

```py
attribute unique: bool
```

索引是否具有唯一标志

```py
class sqlalchemy.engine.interfaces.ReflectedPrimaryKeyConstraint
```

表示与`PrimaryKeyConstraint`相对应的反射元素的字典。

`ReflectedPrimaryKeyConstraint` 结构由 `Inspector.get_pk_constraint()` 方法返回。

**成员**

constrained_columns, dialect_options

**类签名**

类`sqlalchemy.engine.interfaces.ReflectedPrimaryKeyConstraint` (`builtins.dict`)

```py
attribute constrained_columns: List[str]
```

组成主键的列名

```py
attribute dialect_options: NotRequired[Dict[str, Any]]
```

检测到的此主键的附加特定方言选项

```py
class sqlalchemy.engine.interfaces.ReflectedUniqueConstraint
```

字典表示对应于 `UniqueConstraint` 的反射元素。

`ReflectedUniqueConstraint` 结构由 `Inspector.get_unique_constraints()` 方法返回。

**成员**

column_names, dialect_options, duplicates_index

**类签名**

类`sqlalchemy.engine.interfaces.ReflectedUniqueConstraint` (`builtins.dict`)

```py
attribute column_names: List[str]
```

组成唯一约束的列名

```py
attribute dialect_options: NotRequired[Dict[str, Any]]
```

检测到的此唯一约束的附加特定方言选项

```py
attribute duplicates_index: NotRequired[str | None]
```

指示此唯一约束是否重复使用此名称的索引

```py
class sqlalchemy.engine.interfaces.ReflectedTableComment
```

字典表示对应于 `Table.comment` 属性的反射注释。

`ReflectedTableComment` 结构由 `Inspector.get_table_comment()` 方法返回。

**成员**

text

**类签名**

类`sqlalchemy.engine.interfaces.ReflectedTableComment` (`builtins.dict`)

```py
attribute text: str | None
```

注释文本

## 使用数据库通用类型反射

当反射表的列时，无论是使用`Table`的`Table.autoload_with`参数，还是使用`Inspector`的`Inspector.get_columns()`方法，数据类型都将尽可能地特定于目标数据库。这意味着，如果从 MySQL 数据库中反射出一个“整数”数据类型，该类型将由`sqlalchemy.dialects.mysql.INTEGER`类表示，其中包括 MySQL 特定的属性，如“display_width”。或者在 PostgreSQL 上，可能会返回 PostgreSQL 特定的数据类型，如`sqlalchemy.dialects.postgresql.INTERVAL`或`sqlalchemy.dialects.postgresql.ENUM`。

有一个反射的用例，即给定一个`Table`要转移到另一个供应商数据库。为了适应这个用例，有一种技术，可以将这些供应商特定的数据类型即时转换为 SQLAlchemy 后端不可知的数据类型，例如上面的示例中的`Integer`、`Interval`和`Enum`。这可以通过拦截列反射并结合`DDLEvents.column_reflect()`事件和`TypeEngine.as_generic()`方法来实现。

给定一个 MySQL 表（选择 MySQL 是因为 MySQL 具有许多供应商特定的数据类型和选项）：

```py
CREATE  TABLE  IF  NOT  EXISTS  my_table  (
  id  INTEGER  PRIMARY  KEY  AUTO_INCREMENT,
  data1  VARCHAR(50)  CHARACTER  SET  latin1,
  data2  MEDIUMINT(4),
  data3  TINYINT(2)
)
```

上述表包括 MySQL 专用的整数类型`MEDIUMINT`和`TINYINT`，以及一个包含 MySQL 专用`CHARACTER SET`选项的`VARCHAR`。如果我们正常地反映这个表，它将产生一个包含这些 MySQL 特定数据类型和选项的`Table`对象：

```py
>>> from sqlalchemy import MetaData, Table, create_engine
>>> mysql_engine = create_engine("mysql+mysqldb://scott:tiger@localhost/test")
>>> metadata_obj = MetaData()
>>> my_mysql_table = Table("my_table", metadata_obj, autoload_with=mysql_engine)
```

上面的示例将上述表模式反映到一个新的 `Table` 对象中。然后，为了演示目的，我们可以使用 `CreateTable` 构造打印出特定于 MySQL 的“CREATE TABLE”语句：

```py
>>> from sqlalchemy.schema import CreateTable
>>> print(CreateTable(my_mysql_table).compile(mysql_engine))
CREATE  TABLE  my_table  (
id  INTEGER(11)  NOT  NULL  AUTO_INCREMENT,
data1  VARCHAR(50)  CHARACTER  SET  latin1,
data2  MEDIUMINT(4),
data3  TINYINT(2),
PRIMARY  KEY  (id)
)ENGINE=InnoDB  DEFAULT  CHARSET=utf8mb4 
```

在上面的例子中，保留了特定于 MySQL 的数据类型和选项。如果我们想要一个可以干净地转移到另一个数据库供应商的 `Table`，并用 `sqlalchemy.dialects.mysql.MEDIUMINT` 和 `sqlalchemy.dialects.mysql.TINYINT` 替换特殊数据类型，则可以选择在此表上“泛型化”数据类型，或以任何我们喜欢的方式更改它们，方法是使用 `DDLEvents.column_reflect()` 事件建立一个处理程序。自定义处理程序将使用 `TypeEngine.as_generic()` 方法，通过替换传递给事件处理程序的列字典条目中的 `"type"` 条目来将上述特定于 MySQL 的类型对象转换为通用类型。此字典的格式在 `Inspector.get_columns()` 中描述：

```py
>>> from sqlalchemy import event
>>> metadata_obj = MetaData()

>>> @event.listens_for(metadata_obj, "column_reflect")
... def genericize_datatypes(inspector, tablename, column_dict):
...     column_dict["type"] = column_dict["type"].as_generic()

>>> my_generic_table = Table("my_table", metadata_obj, autoload_with=mysql_engine)
```

现在我们得到了一个新的泛型 `Table` 并且使用 `Integer` 作为这些数据类型。例如，我们现在可以在 PostgreSQL 数据库上发出“CREATE TABLE”语句：

```py
>>> pg_engine = create_engine("postgresql+psycopg2://scott:tiger@localhost/test", echo=True)
>>> my_generic_table.create(pg_engine)
CREATE  TABLE  my_table  (
  id  SERIAL  NOT  NULL,
  data1  VARCHAR(50),
  data2  INTEGER,
  data3  INTEGER,
  PRIMARY  KEY  (id)
) 
```

还要注意，SQLAlchemy 通常会对其他行为做出合理的猜测，例如，MySQL 的 `AUTO_INCREMENT` 指令在 PostgreSQL 中最接近地表示为 `SERIAL` 自动增量数据类型。

1.4 版本新增了 `TypeEngine.as_generic()` 方法，并进一步改进了 `DDLEvents.column_reflect()` 事件的使用，以便可以方便地将其应用于 `MetaData` 对象。

## 反射的局限性

需要注意的是，反射过程仅使用在关系数据库中表示的信息重建`Table`元数据。按照定义，此过程无法恢复数据库中实际未存储的模式的方面。无法从反射中获得的状态包括但不限于：

+   客户端默认值，可以是使用`Column`的`default`关键字定义的 Python 函数或 SQL 表达式（注意，这与`server_default`是分开的，后者是通过反射获得的）。

+   列信息，例如可能已放置在`Column.info`字典中的数据。

+   对于`Column`或`Table`的`.quote`设置的值。

+   将特定的`Sequence`与给定的`Column`相关联。

在许多情况下，关系数据库报告的表元数据格式与 SQLAlchemy 中指定的格式不同。从反射返回的`Table`对象不能始终依赖于产生与原始 Python 定义的`Table`对象相同的 DDL。发生这种情况的领域包括服务器默认值、与列相关联的序列以及关于约束和数据类型的各种特殊情况。服务器端默认值可能会以转换指令返回（通常情况下，PostgreSQL 会包含一个`::<type>`转换）或与最初指定的不同的引用模式。

另一类限制包括仅部分或尚未定义反射的模式结构。最近对反射进行的改进允许反射诸如视图、索引和外键选项之类的内容。截至撰写本文时，像检查约束、表注释和触发器之类的结构并未反射。
