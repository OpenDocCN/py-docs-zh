# 自定义 DDL

> 原文：[`docs.sqlalchemy.org/en/20/core/ddl.html`](https://docs.sqlalchemy.org/en/20/core/ddl.html)

在前面的章节中，我们讨论了各种模式构造，包括 `Table`、`ForeignKeyConstraint`、`CheckConstraint` 和 `Sequence`。在整个过程中，我们依赖于 `Table` 和 `MetaData` 的 `create()` 和 `create_all()` 方法来为所有构造发出数据定义语言 (DDL)。当发出时，会调用预先确定的操作顺序，并且无条件地创建用于创建每个表的 DDL，包括与其关联的所有约束和其他对象。对于需要数据库特定 DDL 的更复杂的场景，SQLAlchemy 提供了两种技术，可以根据任何条件添加任何 DDL，既可以附带标准生成表格，也可以单独使用。

## 自定义 DDL

自定义 DDL 短语最容易通过 `DDL` 结构实现。这个结构的工作方式与所有其他 DDL 元素相同，除了它接受一个字符串作为要发出的文本：

```py
event.listen(
    metadata,
    "after_create",
    DDL(
        "ALTER TABLE users ADD CONSTRAINT "
        "cst_user_name_length "
        " CHECK (length(user_name) >= 8)"
    ),
)
```

创建 DDL 构造库的更全面的方法是使用自定义编译 - 有关详细信息，请参阅 自定义 SQL 构造和编译扩展。

## 控制 DDL 序列

之前介绍的 `DDL` 结构还具有根据对数据库的检查有条件地调用的能力。可以使用 `ExecutableDDLElement.execute_if()` 方法实现此功能。例如，如果我们想要创建一个触发器，但只在 PostgreSQL 后端上，我们可以这样调用：

```py
mytable = Table(
    "mytable",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("data", String(50)),
)

func = DDL(
    "CREATE FUNCTION my_func() "
    "RETURNS TRIGGER AS $$ "
    "BEGIN "
    "NEW.data := 'ins'; "
    "RETURN NEW; "
    "END; $$ LANGUAGE PLPGSQL"
)

trigger = DDL(
    "CREATE TRIGGER dt_ins BEFORE INSERT ON mytable "
    "FOR EACH ROW EXECUTE PROCEDURE my_func();"
)

event.listen(mytable, "after_create", func.execute_if(dialect="postgresql"))

event.listen(mytable, "after_create", trigger.execute_if(dialect="postgresql"))
```

`ExecutableDDLElement.execute_if.dialect` 关键字还接受一个字符串方言名称的元组：

```py
event.listen(
    mytable, "after_create", trigger.execute_if(dialect=("postgresql", "mysql"))
)
event.listen(
    mytable, "before_drop", trigger.execute_if(dialect=("postgresql", "mysql"))
)
```

`ExecutableDDLElement.execute_if()` 方法也可以针对一个可调用函数进行操作，该函数将接收正在使用的数据库连接。在下面的示例中，我们使用这个方法来有条件地创建一个 CHECK 约束，首先在 PostgreSQL 目录中查看是否存在：

```py
def should_create(ddl, target, connection, **kw):
    row = connection.execute(
        "select conname from pg_constraint where conname='%s'" % ddl.element.name
    ).scalar()
    return not bool(row)

def should_drop(ddl, target, connection, **kw):
    return not should_create(ddl, target, connection, **kw)

event.listen(
    users,
    "after_create",
    DDL(
        "ALTER TABLE users ADD CONSTRAINT "
        "cst_user_name_length CHECK (length(user_name) >= 8)"
    ).execute_if(callable_=should_create),
)
event.listen(
    users,
    "before_drop",
    DDL("ALTER TABLE users DROP CONSTRAINT cst_user_name_length").execute_if(
        callable_=should_drop
    ),
)

users.create(engine)
CREATE  TABLE  users  (
  user_id  SERIAL  NOT  NULL,
  user_name  VARCHAR(40)  NOT  NULL,
  PRIMARY  KEY  (user_id)
)

SELECT  conname  FROM  pg_constraint  WHERE  conname='cst_user_name_length'
ALTER  TABLE  users  ADD  CONSTRAINT  cst_user_name_length  CHECK  (length(user_name)  >=  8)
users.drop(engine)
SELECT  conname  FROM  pg_constraint  WHERE  conname='cst_user_name_length'
ALTER  TABLE  users  DROP  CONSTRAINT  cst_user_name_length
DROP  TABLE  users 
```

## 使用内置 DDLElement 类

`sqlalchemy.schema` 包含提供 DDL 表达式的 SQL 表达式构造，所有这些构造都扩展自公共基类 `ExecutableDDLElement`。例如，要生成 `CREATE TABLE` 语句，可以使用 `CreateTable` 构造：

```py
from sqlalchemy.schema import CreateTable

with engine.connect() as conn:
    conn.execute(CreateTable(mytable))
CREATE  TABLE  mytable  (
  col1  INTEGER,
  col2  INTEGER,
  col3  INTEGER,
  col4  INTEGER,
  col5  INTEGER,
  col6  INTEGER
) 
```

在上述内容中，`CreateTable` 构造与任何其他表达式构造（例如 `select()`、`table.insert()` 等）一样工作。SQLAlchemy 所有面向 DDL 的构造都是 `ExecutableDDLElement` 基类的子类；这是 SQLAlchemy 中的所有对象对应 CREATE 和 DROP 以及 ALTER 的基础，不仅在 SQLAlchemy 中，在 Alembic 迁移中也是如此。可用构造的完整参考在 DDL 表达式构造 API 中。

用户定义的 DDL 构造也可以作为 `ExecutableDDLElement` 的子类而创建。在自定义 SQL 构造和编译扩展文档中有几个示例。

## 控制约束和索引的 DDL 生成

新版本 2.0 中的新增内容。

虽然前面提到的 `ExecutableDDLElement.execute_if()` 方法对于需要有条件地调用的自定义 `DDL` 类很有用，但通常也有一种常见需求，即通常与特定 `Table` 相关的元素，例如约束和索引，也要受到“条件”规则的约束，例如一个索引包含特定于特定后端（如 PostgreSQL 或 SQL Server）的功能。对于这种用例，可以针对诸如 `CheckConstraint`、`UniqueConstraint` 和 `Index` 等构造使用 `Constraint.ddl_if()` 和 `Index.ddl_if()` 方法，接受与 `ExecutableDDLElement.execute_if()` 方法相同的参数，以控制它们的 DDL 是否会在其父 `Table` 对象的情况下发出。这些方法可以在创建 `Table` 的定义时内联使用（或类似地，在 ORM 声明映射中使用 `__table_args__` 集合时），例如：

```py
from sqlalchemy import CheckConstraint, Index
from sqlalchemy import MetaData, Table, Column
from sqlalchemy import Integer, String

meta = MetaData()

my_table = Table(
    "my_table",
    meta,
    Column("id", Integer, primary_key=True),
    Column("num", Integer),
    Column("data", String),
    Index("my_pg_index", "data").ddl_if(dialect="postgresql"),
    CheckConstraint("num > 5").ddl_if(dialect="postgresql"),
)
```

在上面的例子中，`Table` 构造同时指代 `Index` 和 `CheckConstraint` 构造，它们都表明 `.ddl_if(dialect="postgresql")`，这意味着这些元素只会在针对 PostgreSQL 方言时包含在 CREATE TABLE 序列中。例如，如果我们针对 SQLite 方言运行 `meta.create_all()`，那么这两个构造都不会被包含：

```py
>>> from sqlalchemy import create_engine
>>> sqlite_engine = create_engine("sqlite+pysqlite://", echo=True)
>>> meta.create_all(sqlite_engine)
BEGIN  (implicit)
PRAGMA  main.table_info("my_table")
[raw  sql]  ()
PRAGMA  temp.table_info("my_table")
[raw  sql]  ()

CREATE  TABLE  my_table  (
  id  INTEGER  NOT  NULL,
  num  INTEGER,
  data  VARCHAR,
  PRIMARY  KEY  (id)
) 
```

然而，如果我们针对 PostgreSQL 数据库运行相同的命令，我们将看到 CHECK 约束的内联 DDL 以及为索引发出的单独的 CREATE 语句：

```py
>>> from sqlalchemy import create_engine
>>> postgresql_engine = create_engine(
...     "postgresql+psycopg2://scott:tiger@localhost/test", echo=True
... )
>>> meta.create_all(postgresql_engine)
BEGIN  (implicit)
select  relname  from  pg_class  c  join  pg_namespace  n  on  n.oid=c.relnamespace  where  pg_catalog.pg_table_is_visible(c.oid)  and  relname=%(name)s
[generated  in  0.00009s]  {'name':  'my_table'}

CREATE  TABLE  my_table  (
  id  SERIAL  NOT  NULL,
  num  INTEGER,
  data  VARCHAR,
  PRIMARY  KEY  (id),
  CHECK  (num  >  5)
)
[no  key  0.00007s]  {}
CREATE  INDEX  my_pg_index  ON  my_table  (data)
[no  key  0.00013s]  {}
COMMIT 
```

`Constraint.ddl_if()`和`Index.ddl_if()`方法创建了一个事件钩子，该事件钩子不仅可以在 DDL 执行时进行查询，就像`ExecutableDDLElement.execute_if()`的行为一样，还可以在`CreateTable`对象的 SQL 编译阶段内进行查询，该对象负责在 CREATE TABLE 语句中内联渲染`CHECK (num > 5)` DDL。因此，`ddl_if.callable_()`参数接收到的事件钩子具有更丰富的参数集，包括传递了一个`dialect`关键字参数，以及通过`compiler`关键字参数传递给“内联渲染”部分的`DDLCompiler`的实例。当事件在`DDLCompiler`序列中触发时，`bind`参数**不**存在，因此，现代事件钩子如果希望检查数据库版本信息，则最好使用给定的`Dialect`对象，例如测试 PostgreSQL 版本：

```py
def only_pg_14(ddl_element, target, bind, dialect, **kw):
    return dialect.name == "postgresql" and dialect.server_version_info >= (14,)

my_table = Table(
    "my_table",
    meta,
    Column("id", Integer, primary_key=True),
    Column("num", Integer),
    Column("data", String),
    Index("my_pg_index", "data").ddl_if(callable_=only_pg_14),
)
```

另见

`Constraint.ddl_if()`

`Index.ddl_if()`  ## DDL 表达式构造 API

| 对象名称 | 描述 |
| --- | --- |
| _CreateDropBase | 用于表示 CREATE 和 DROP 或等效语句的 DDL 构造的基类。 |
| AddConstraint | 代表一个 ALTER TABLE ADD CONSTRAINT 语句。 |
| BaseDDLElement | DDL 构造的根，包括那些在“创建表”和其他过程中的子元素。 |
| CreateColumn | 在 CREATE TABLE 语句中呈现为`Column`的表示，通过`CreateTable`构造。 |
| CreateIndex | 代表一个 CREATE INDEX 语句。 |
| CreateSchema | 代表一个 CREATE SCHEMA 语句。 |
| CreateSequence | 代表一个 CREATE SEQUENCE 语句。 |
| CreateTable | 代表一个 CREATE TABLE 语句。 |
| DDL | 一个字面的 DDL 语句。 |
| DropConstraint | 表示 ALTER TABLE DROP CONSTRAINT 语句。 |
| DropIndex | 表示一个 DROP INDEX 语句。 |
| DropSchema | 表示一个 DROP SCHEMA 语句。 |
| DropSequence | 表示一个 DROP SEQUENCE 语句。 |
| DropTable | 表示一个 DROP TABLE 语句。 |
| ExecutableDDLElement | 独立可执行的 DDL 表达式构造的基类。 |
| sort_tables(tables[, skip_fn, extra_dependencies]) | 根据依赖关系对一组`Table`对象进行排序。 |
| sort_tables_and_constraints(tables[, filter_fn, extra_dependencies, _warn_for_cycles]) | 对一组`Table` / `ForeignKeyConstraint`对象进行排序。 |

```py
function sqlalchemy.schema.sort_tables(tables: Iterable[TableClause], skip_fn: Callable[[ForeignKeyConstraint], bool] | None = None, extra_dependencies: typing_Sequence[Tuple[TableClause, TableClause]] | None = None) → List[Table]
```

根据依赖关系对一组`Table`对象进行排序。

这是一个依赖顺序排序，将发出`Table`对象，使其跟随其依赖的`Table`对象。表是根据存在的`ForeignKeyConstraint`对象以及由`Table.add_is_dependent_on()`添加的显式依赖关系而相互依赖的。

警告

`sort_tables()`函数本身无法处理表之间的依赖循环，这些循环通常是由相互依赖的外键约束引起的。当检测到这些循环时，这些表的外键将被从排序中排除。当发生此情况时会发出警告，这将在将来的版本中引发异常。不属于循环的表仍将按依赖顺序返回。

为解决这些循环，可以将 `ForeignKeyConstraint.use_alter` 参数应用于创建循环的约束。或者，当检测到循环时，`sort_tables_and_constraints()` 函数将自动返回外键约束的单独集合，以便可以将其分别应用于模式。

在版本 1.3.17 中更改：- 当`sort_tables()` 由于循环依赖关系无法执行正确排序时，会发出警告。在将来的版本中，这将成为一个异常。此外，排序将继续返回其他未涉及到的表，这些表的排序不是之前的那种依赖顺序。

参数:

+   `tables` – 一个`Table` 对象的序列。

+   `skip_fn` – 可选的可调用对象，将传递一个`ForeignKeyConstraint` 对象；如果返回 True，则此约束将不被视为依赖项。请注意，这与`sort_tables_and_constraints()` 中的相同参数不同，后者实际上是传递了拥有者`ForeignKeyConstraint` 对象。

+   `extra_dependencies` – 2-元组序列，这些表也将被视为彼此相关。

另请参见

`sort_tables_and_constraints()`

`MetaData.sorted_tables` - 使用此函数进行排序

```py
function sqlalchemy.schema.sort_tables_and_constraints(tables, filter_fn=None, extra_dependencies=None, _warn_for_cycles=False)
```

对`Table` / `ForeignKeyConstraint` 对象进行排序。

这是一个依赖项排序，将发出元组 `(Table, [ForeignKeyConstraint, ...])`，以便每个`Table` 后跟其依赖的 `Table` 对象。由于排序未满足依赖关系规则而分离的其余`ForeignKeyConstraint` 对象稍后作为 `(None, [ForeignKeyConstraint ...])` 发出。

表格依赖于另一个基于存在`ForeignKeyConstraint`对象，通过`Table.add_is_dependent_on()`添加的显式依赖关系，以及在此处使用`sort_tables_and_constraints.skip_fn`和/或`sort_tables_and_constraints.extra_dependencies`参数指定的依赖关系。

参数：

+   `tables` – `Table`对象的序列。

+   `filter_fn` – 可选的可调用对象，将传递给`ForeignKeyConstraint`对象，并根据此约束是否应作为内联约束绝对包含或排除的值返回一个值，或者两者都不是。如果返回 False，则该约束肯定会被包含为无法进行 ALTER 的依赖项；如果返回 True，则它将仅作为 ALTER 结果在最后包含。返回 None 意味着该约束将包含在基于表的结果中，除非它被检测为依赖循环的一部分。

+   `extra_dependencies` – 2 元组序列，其中的表也将被视为相互依赖。

另请参阅

`sort_tables()`

```py
class sqlalchemy.schema.BaseDDLElement
```

DDL 构造的根，包括那些在“create table”和其他进程中作为子元素的元素。

版本 2.0 中的新功能。

**类签名**

类`sqlalchemy.schema.BaseDDLElement`（`sqlalchemy.sql.expression.ClauseElement`）

```py
class sqlalchemy.schema.ExecutableDDLElement
```

独立可执行 DDL 表达式构造的基类。

此类是通用目的`DDL`类的基类，以及各种创建/删除子句构造，如`CreateTable`、`DropTable`、`AddConstraint`等。

在版本 2.0 中更改：`ExecutableDDLElement` 从 `DDLElement` 重命名，后者仍然存在以保持向后兼容性。

`ExecutableDDLElement` 与 SQLAlchemy 事件紧密集成，在 事件 中介绍。其实例本身就是一个接收事件的可调用对象：

```py
event.listen(
    users,
    'after_create',
    AddConstraint(constraint).execute_if(dialect='postgresql')
)
```

另请参阅

`DDL`

`DDLEvents`

事件

控制 DDL 序列

**成员**

__call__(), against(), execute_if()

**类签名**

类 `sqlalchemy.schema.ExecutableDDLElement` (`sqlalchemy.sql.roles.DDLRole`, `sqlalchemy.sql.expression.Executable`, `sqlalchemy.schema.BaseDDLElement`)

```py
method __call__(target, bind, **kw)
```

将 DDL 作为 ddl_listener 执行。

```py
method against(target: SchemaItem) → Self
```

返回一个包含给定目标的 `ExecutableDDLElement` 的副本。

这基本上是将给定项应用于返回的 `ExecutableDDLElement` 对象的`.target` 属性。然后，此目标可由事件处理程序和编译例程使用，以提供诸如按特定 `Table` 来标记 DDL 字符串等服务。

当 `ExecutableDDLElement` 对象被建立为 `DDLEvents.before_create()` 或 `DDLEvents.after_create()` 事件的事件处理程序，并且事件然后发生在给定目标（如 `Constraint` 或 `Table`）上时，该目标将使用此方法与 `ExecutableDDLElement` 对象的副本建立，并继续调用 `ExecutableDDLElement.execute()` 方法以调用实际的 DDL 指令。

参数:

**target** – 将进行 DDL 操作的 `SchemaItem`。

返回:

将此`ExecutableDDLElement`的副本与`.target`属性分配给给定的`SchemaItem`。

另请参见

`DDL` - 在处理 DDL 字符串时针对“目标”使用标记化。

```py
method execute_if(dialect: str | None = None, callable_: DDLIfCallable | None = None, state: Any | None = None) → Self
```

返回一个可调用对象，在事件处理程序中有条件地执行此`ExecutableDDLElement`。

用于提供事件监听的包装器：

```py
event.listen(
            metadata,
            'before_create',
            DDL("my_ddl").execute_if(dialect='postgresql')
        )
```

参数：

+   `方言` –

    可能是字符串或字符串元组。如果是字符串，则将与执行数据库方言的名称进行比较：

    ```py
    DDL('something').execute_if(dialect='postgresql')
    ```

    如果是元组，则指定多个方言名称：

    ```py
    DDL('something').execute_if(dialect=('postgresql', 'mysql'))
    ```

+   `callable_` –

    一个可调用对象，将以三个位置参数以及可选关键字参数的形式调用：

    > ddl： 
    > 
    > 此 DDL 元素。
    > 
    > 目标：
    > 
    > 此事件的目标是`Table`或`MetaData`对象。如果显式执行 DDL，则可能为 None。
    > 
    > 绑定：
    > 
    > 用于执行 DDL 的`Connection`。如果此构造在表内创建，则可能为 None，在这种情况下，`编译器`将存在。
    > 
    > 表：
    > 
    > 可选关键字参数 - 要在 MetaData.create_all()或 drop_all()方法调用中创建/删除的 Table 对象列表。
    > 
    > 方言：
    > 
    > 关键字参数，但始终存在 - 涉及操作的`Dialect`。
    > 
    > 编译器：
    > 
    > 关键字参数。对于引擎级别的 DDL 调用，将为`None`，但如果此 DDL 元素在表内创建时，将引用`DDLCompiler`。
    > 
    > 状态：
    > 
    > 可选关键字参数 - 将是传递给此函数的`state`参数。
    > 
    > checkfirst：
    > 
    > 关键字参数，如果在调用`create()`、`create_all()`、`drop()`、`drop_all()`时设置了‘checkfirst’标志，则为 True。

    如果可调用函数返回 True 值，则将执行 DDL 语句。

+   `状态` – 任何值，将作为`state`关键字参数传递给`callable_`。

另请参见

`SchemaItem.ddl_if()`

`DDLEvents`

事件

```py
class sqlalchemy.schema.DDL
```

一个字面 DDL 语句。

指定要由数据库执行的文字 SQL DDL。DDL 对象作为 DDL 事件监听器，可以订阅那些在`DDLEvents`中列出的事件，使用`Table`或`MetaData`对象作为目标。基本的模板支持允许单个 DDL 实例处理多个表的重复任务。

例子：

```py
from sqlalchemy import event, DDL

tbl = Table('users', metadata, Column('uid', Integer))
event.listen(tbl, 'before_create', DDL('DROP TRIGGER users_trigger'))

spow = DDL('ALTER TABLE %(table)s SET secretpowers TRUE')
event.listen(tbl, 'after_create', spow.execute_if(dialect='somedb'))

drop_spow = DDL('ALTER TABLE users SET secretpowers FALSE')
connection.execute(drop_spow)
```

在操作 Table 事件时，以下`statement`字符串替换可用：

```py
%(table)s  - the Table name, with any required quoting applied
%(schema)s - the schema name, with any required quoting applied
%(fullname)s - the Table name including schema, quoted if needed
```

如果有的话，DDL 的“context”将与上述标准替换组合在一起。上下文中存在的键将覆盖标准替换。

**成员**

__init__()

**类签名**

类`sqlalchemy.schema.DDL` (`sqlalchemy.schema.ExecutableDDLElement`)

```py
method __init__(statement, context=None)
```

创建一个 DDL 语句。

参数：

+   `statement` –

    要执行的字符串或 unicode 字符串。语句将使用 Python 的字符串格式化操作符以及由可选的`DDL.context`参数提供的一组固定字符串替换处理。

    语句中的字面‘%’必须被转义为‘%%’。

    DDL 语句中不可用 SQL 绑定参数。

+   `context` – 可选字典，默认为 None。这些值将可用于对 DDL 语句进行字符串替换。

另请参阅

`DDLEvents`

事件

```py
class sqlalchemy.schema._CreateDropBase
```

用于表示 CREATE 和 DROP 或等效操作的 DDL 构造的基类。

_CreateDropBase 的共同主题是一个`element`属性，它指的是要创建或删除的元素。

**类签名**

类 `sqlalchemy.schema._CreateDropBase` (`sqlalchemy.schema.ExecutableDDLElement`)

```py
class sqlalchemy.schema.CreateTable
```

表示一个 CREATE TABLE 语句。

**成员**

__init__()

**类签名**

类`sqlalchemy.schema.CreateTable` (`sqlalchemy.schema._CreateBase`)

```py
method __init__(element: Table, include_foreign_key_constraints: typing_Sequence[ForeignKeyConstraint] | None = None, if_not_exists: bool = False)
```

创建一个`CreateTable` 构造。

参数：

+   `element` – 一个`Table` ，是 CREATE 的主题

+   `on` – 参见`DDL`中`on`的描述。

+   `include_foreign_key_constraints` – 可选的`ForeignKeyConstraint`对象序列，将内联包含在 CREATE 构造中；如果省略，那么所有没有指定 use_alter=True 的外键约束都将被包含。

+   `if_not_exists` –

    如果为 True，则将应用 IF NOT EXISTS 操作符到构造中。

    新版本 1.4.0b2 中提供。

```py
class sqlalchemy.schema.DropTable
```

代表一个 DROP TABLE 语句。

**成员**

__init__()

**类签名**

类`sqlalchemy.schema.DropTable`(`sqlalchemy.schema._DropBase`)

```py
method __init__(element: Table, if_exists: bool = False)
```

创建一个`DropTable`构造。

参数：

+   `element` – 一个`Table`，是 DROP 的主题。

+   `on` – 查看`DDL`中关于`on`的描述。

+   `if_exists` –

    如果为 True，则将应用 IF EXISTS 操作符到构造中。

    新版本 1.4.0b2 中提供。

```py
class sqlalchemy.schema.CreateColumn
```

代表一个`Column`在 CREATE TABLE 语句中的呈现，通过`CreateTable`构造。

这是为了在生成 CREATE TABLE 语句时支持自定义列 DDL，通过使用在自定义 SQL 构造和编译扩展中记录的编译器扩展来扩展`CreateColumn`。

典型的集成是检查传入的`Column`对象，并在找到特定标志或条件时重定向编译：

```py
from sqlalchemy import schema
from sqlalchemy.ext.compiler import compiles

@compiles(schema.CreateColumn)
def compile(element, compiler, **kw):
    column = element.element

    if "special" not in column.info:
        return compiler.visit_create_column(element, **kw)

    text = "%s SPECIAL DIRECTIVE %s" % (
            column.name,
            compiler.type_compiler.process(column.type)
        )
    default = compiler.get_column_default_string(column)
    if default is not None:
        text += " DEFAULT " + default

    if not column.nullable:
        text += " NOT NULL"

    if column.constraints:
        text += " ".join(
                    compiler.process(const)
                    for const in column.constraints)
    return text
```

上述构造可以应用到一个`Table`中，如下所示：

```py
from sqlalchemy import Table, Metadata, Column, Integer, String
from sqlalchemy import schema

metadata = MetaData()

table = Table('mytable', MetaData(),
        Column('x', Integer, info={"special":True}, primary_key=True),
        Column('y', String(50)),
        Column('z', String(20), info={"special":True})
    )

metadata.create_all(conn)
```

上述，我们添加到`Column.info`集合的指令将被我们的自定义编译方案检测到：

```py
CREATE TABLE mytable (
        x SPECIAL DIRECTIVE INTEGER NOT NULL,
        y VARCHAR(50),
        z SPECIAL DIRECTIVE VARCHAR(20),
    PRIMARY KEY (x)
)
```

`CreateColumn`构造也可以用于在生成`CREATE TABLE`时跳过某些列。这是通过创建一个有条件地返回`None`的编译规则来实现的。这本质上就是如何产生与在`Column`上使用`system=True`参数相同的效果，这个参数将列标记为隐式存在的“系统”列。

例如，假设我们希望生成一个`Table`，它在 PostgreSQL 后端跳过渲染 PostgreSQL `xmin` 列，但在其他后端会渲染它，以预期触发规则。条件编译规则只会在 PostgreSQL 上跳过此名称：

```py
from sqlalchemy.schema import CreateColumn

@compiles(CreateColumn, "postgresql")
def skip_xmin(element, compiler, **kw):
    if element.element.name == 'xmin':
        return None
    else:
        return compiler.visit_create_column(element, **kw)

my_table = Table('mytable', metadata,
            Column('id', Integer, primary_key=True),
            Column('xmin', Integer)
        )
```

上述的`CreateTable` 结构将生成一个 `CREATE TABLE`，其中字符串只包含 `id` 列；`xmin` 列将被省略，但仅针对 PostgreSQL 后端。

**类签名**

类 `sqlalchemy.schema.CreateColumn`（`sqlalchemy.schema.BaseDDLElement`）

```py
class sqlalchemy.schema.CreateSequence
```

代表一个 CREATE SEQUENCE 语句。

**类签名**

类 `sqlalchemy.schema.CreateSequence`（`sqlalchemy.schema._CreateBase`）

```py
class sqlalchemy.schema.DropSequence
```

代表一个 DROP SEQUENCE 语句。

**类签名**

类 `sqlalchemy.schema.DropSequence`（`sqlalchemy.schema._DropBase`）

```py
class sqlalchemy.schema.CreateIndex
```

代表一个 CREATE INDEX 语句。

**成员**

__init__() 

**类签名**

类 `sqlalchemy.schema.CreateIndex`（`sqlalchemy.schema._CreateBase`）

```py
method __init__(element, if_not_exists=False)
```

创建一个 `Createindex` 结构。

参数：

+   `element` – 一个 `Index`，是 CREATE 的主题。

+   `if_not_exists` –

    若为真，则会对结构应用 IF NOT EXISTS 操作符。

    从版本 1.4.0b2 开始的新功能。

```py
class sqlalchemy.schema.DropIndex
```

代表一个 DROP INDEX 语句。

**成员**

__init__()

**类签名**

类 `sqlalchemy.schema.DropIndex`（`sqlalchemy.schema._DropBase`）

```py
method __init__(element, if_exists=False)
```

创建一个 `DropIndex` 结构。

参数：

+   `element` – 一个 `Index`，是 DROP 的主题。

+   `if_exists` –

    若为真，则会对结构应用 IF EXISTS 操作符。

    从版本 1.4.0b2 开始的新功能。

```py
class sqlalchemy.schema.AddConstraint
```

代表一个 ALTER TABLE ADD CONSTRAINT 语句。

**类签名**

类 `sqlalchemy.schema.AddConstraint`（`sqlalchemy.schema._CreateBase`）

```py
class sqlalchemy.schema.DropConstraint
```

代表一个 ALTER TABLE DROP CONSTRAINT 语句。

**类签名**

类 `sqlalchemy.schema.DropConstraint`（`sqlalchemy.schema._DropBase`）

```py
class sqlalchemy.schema.CreateSchema
```

代表一个 CREATE SCHEMA 语句。

此处的参数是模式的字符串名称。

**成员**

__init__()

**类签名**

类 `sqlalchemy.schema.CreateSchema` (`sqlalchemy.schema._CreateBase`)

```py
method __init__(name, if_not_exists=False)
```

创建一个新的 `CreateSchema` 构造。

```py
class sqlalchemy.schema.DropSchema
```

表示一个 DROP SCHEMA 语句。

这里的参数是模式的字符串名称。

**成员**

__init__()

**类签名**

类 `sqlalchemy.schema.DropSchema` (`sqlalchemy.schema._DropBase`)

```py
method __init__(name, cascade=False, if_exists=False)
```

创建一个新的 `DropSchema` 构造。

## 自定义 DDL

使用 `DDL` 构造最容易实现自定义 DDL 短语。此构造与所有其他 DDL 元素的工作方式相同，只是它接受一个文本字符串作为要发出的文本：

```py
event.listen(
    metadata,
    "after_create",
    DDL(
        "ALTER TABLE users ADD CONSTRAINT "
        "cst_user_name_length "
        " CHECK (length(user_name) >= 8)"
    ),
)
```

创建 DDL 构造库的更全面的方法是使用自定义编译 - 有关详细信息，请参阅 自定义 SQL 构造和编译扩展。

## 控制 DDL 序列

之前介绍的 `DDL` 构造也具有根据对数据库的检查有条件地调用的功能。使用 `ExecutableDDLElement.execute_if()` 方法可以使用此功能。例如，如果我们想要创建一个触发器，但仅在 PostgreSQL 后端上，我们可以这样调用：

```py
mytable = Table(
    "mytable",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("data", String(50)),
)

func = DDL(
    "CREATE FUNCTION my_func() "
    "RETURNS TRIGGER AS $$ "
    "BEGIN "
    "NEW.data := 'ins'; "
    "RETURN NEW; "
    "END; $$ LANGUAGE PLPGSQL"
)

trigger = DDL(
    "CREATE TRIGGER dt_ins BEFORE INSERT ON mytable "
    "FOR EACH ROW EXECUTE PROCEDURE my_func();"
)

event.listen(mytable, "after_create", func.execute_if(dialect="postgresql"))

event.listen(mytable, "after_create", trigger.execute_if(dialect="postgresql"))
```

`ExecutableDDLElement.execute_if.dialect` 关键字还接受一组字符串方言名称：

```py
event.listen(
    mytable, "after_create", trigger.execute_if(dialect=("postgresql", "mysql"))
)
event.listen(
    mytable, "before_drop", trigger.execute_if(dialect=("postgresql", "mysql"))
)
```

`ExecutableDDLElement.execute_if()` 方法还可以针对一个可调用函数进行操作，该函数将接收正在使用的数据库连接。在下面的示例中，我们使用这个方法来有条件地创建一个 CHECK 约束，首先在 PostgreSQL 目录中查看它是否存在：

```py
def should_create(ddl, target, connection, **kw):
    row = connection.execute(
        "select conname from pg_constraint where conname='%s'" % ddl.element.name
    ).scalar()
    return not bool(row)

def should_drop(ddl, target, connection, **kw):
    return not should_create(ddl, target, connection, **kw)

event.listen(
    users,
    "after_create",
    DDL(
        "ALTER TABLE users ADD CONSTRAINT "
        "cst_user_name_length CHECK (length(user_name) >= 8)"
    ).execute_if(callable_=should_create),
)
event.listen(
    users,
    "before_drop",
    DDL("ALTER TABLE users DROP CONSTRAINT cst_user_name_length").execute_if(
        callable_=should_drop
    ),
)

users.create(engine)
CREATE  TABLE  users  (
  user_id  SERIAL  NOT  NULL,
  user_name  VARCHAR(40)  NOT  NULL,
  PRIMARY  KEY  (user_id)
)

SELECT  conname  FROM  pg_constraint  WHERE  conname='cst_user_name_length'
ALTER  TABLE  users  ADD  CONSTRAINT  cst_user_name_length  CHECK  (length(user_name)  >=  8)
users.drop(engine)
SELECT  conname  FROM  pg_constraint  WHERE  conname='cst_user_name_length'
ALTER  TABLE  users  DROP  CONSTRAINT  cst_user_name_length
DROP  TABLE  users 
```

## 使用内置的 DDLElement 类

`sqlalchemy.schema` 包含提供 DDL 表达式的 SQL 表达式构造，所有这些构造都从共同基类 `ExecutableDDLElement` 扩展。例如，要生成一个 `CREATE TABLE` 语句，可以使用 `CreateTable` 构造：

```py
from sqlalchemy.schema import CreateTable

with engine.connect() as conn:
    conn.execute(CreateTable(mytable))
CREATE  TABLE  mytable  (
  col1  INTEGER,
  col2  INTEGER,
  col3  INTEGER,
  col4  INTEGER,
  col5  INTEGER,
  col6  INTEGER
) 
```

在上面的例子中，`CreateTable` 构造像任何其他表达式构造一样工作（例如 `select()`、`table.insert()` 等）。所有的 SQLAlchemy 的 DDL 相关构造都是 `ExecutableDDLElement` 基类的子类；这是所有与 CREATE、DROP 以及 ALTER 相关的对象的基类，不仅仅在 SQLAlchemy 中，在 Alembic 迁移中也是如此。可用构造的完整参考在 DDL 表达式构造 API 中。

用户定义的 DDL 构造也可以作为 `ExecutableDDLElement` 的子类创建。自定义 SQL 构造和编译扩展 的文档中有几个示例。

## 控制约束和索引的 DDL 生成

新版本 2.0 中新增功能。

虽然之前提到的 `ExecutableDDLElement.execute_if()` 方法对于需要有条件地调用的自定义 `DDL` 类很有用，但通常还有一个常见的需求，即通常与特定 `Table` 相关的元素，如约束和索引，也要受到“条件”规则的约束，比如一个包含特定于特定后端（如 PostgreSQL 或 SQL Server）的特性的索引。对于这种用例，可以针对诸如 `CheckConstraint`、`UniqueConstraint` 和 `Index` 这样的构造使用 `Constraint.ddl_if()` 和 `Index.ddl_if()` 方法，接受与 `ExecutableDDLElement.execute_if()` 方法相同的参数，以控制它们的 DDL 是否将以其父 `Table` 对象的形式发出。在创建 `Table` 的定义时（或类似地，在使用 ORM 声明性映射的 `__table_args__` 集合时），可以内联使用这些方法，例如：

```py
from sqlalchemy import CheckConstraint, Index
from sqlalchemy import MetaData, Table, Column
from sqlalchemy import Integer, String

meta = MetaData()

my_table = Table(
    "my_table",
    meta,
    Column("id", Integer, primary_key=True),
    Column("num", Integer),
    Column("data", String),
    Index("my_pg_index", "data").ddl_if(dialect="postgresql"),
    CheckConstraint("num > 5").ddl_if(dialect="postgresql"),
)
```

在上面的示例中，`Table` 构造同时指代 `Index` 和 `CheckConstraint` 构造，均表明 `.ddl_if(dialect="postgresql")`，这表示这些元素仅针对 PostgreSQL 方言将包括在 CREATE TABLE 序列中。例如，如果我们针对 SQLite 方言运行 `meta.create_all()`，则不会包括任何构造：

```py
>>> from sqlalchemy import create_engine
>>> sqlite_engine = create_engine("sqlite+pysqlite://", echo=True)
>>> meta.create_all(sqlite_engine)
BEGIN  (implicit)
PRAGMA  main.table_info("my_table")
[raw  sql]  ()
PRAGMA  temp.table_info("my_table")
[raw  sql]  ()

CREATE  TABLE  my_table  (
  id  INTEGER  NOT  NULL,
  num  INTEGER,
  data  VARCHAR,
  PRIMARY  KEY  (id)
) 
```

但是，如果我们对 PostgreSQL 数据库运行相同的命令，我们将看到内联的 DDL 用于 CHECK 约束，以及为索引发出的单独的 CREATE 语句：

```py
>>> from sqlalchemy import create_engine
>>> postgresql_engine = create_engine(
...     "postgresql+psycopg2://scott:tiger@localhost/test", echo=True
... )
>>> meta.create_all(postgresql_engine)
BEGIN  (implicit)
select  relname  from  pg_class  c  join  pg_namespace  n  on  n.oid=c.relnamespace  where  pg_catalog.pg_table_is_visible(c.oid)  and  relname=%(name)s
[generated  in  0.00009s]  {'name':  'my_table'}

CREATE  TABLE  my_table  (
  id  SERIAL  NOT  NULL,
  num  INTEGER,
  data  VARCHAR,
  PRIMARY  KEY  (id),
  CHECK  (num  >  5)
)
[no  key  0.00007s]  {}
CREATE  INDEX  my_pg_index  ON  my_table  (data)
[no  key  0.00013s]  {}
COMMIT 
```

`Constraint.ddl_if()` 和 `Index.ddl_if()` 方法创建一个事件挂钩，该挂钩不仅可以在 DDL 执行时查询，如 `ExecutableDDLElement.execute_if()` 的行为，而且还可以在 `CreateTable` 对象的 SQL 编译阶段内查询，该阶段负责在 CREATE TABLE 语句中内联渲染 `CHECK (num > 5)` DDL。因此，通过 `ddl_if.callable_()` 参数接收到的事件挂钩具有更丰富的参数集，包括传递的 `dialect` 关键字参数，以及通过 `compiler` 关键字参数传递的 `DDLCompiler` 的实例，用于序列的“内联渲染”部分。当事件在 `DDLCompiler` 序列内触发时，`bind` 参数 **不存在**，因此，希望检查数据库版本信息的现代事件挂钩最好使用给定的 `Dialect` 对象，例如测试 PostgreSQL 版本：

```py
def only_pg_14(ddl_element, target, bind, dialect, **kw):
    return dialect.name == "postgresql" and dialect.server_version_info >= (14,)

my_table = Table(
    "my_table",
    meta,
    Column("id", Integer, primary_key=True),
    Column("num", Integer),
    Column("data", String),
    Index("my_pg_index", "data").ddl_if(callable_=only_pg_14),
)
```

另请参阅

`Constraint.ddl_if()`

`Index.ddl_if()`

## DDL 表达式构造 API

| 对象名称 | 描述 |
| --- | --- |
| _CreateDropBase | 用于表示 CREATE 和 DROP 或等效操作的 DDL 构造的基类。 |
| AddConstraint | 代表 ALTER TABLE ADD CONSTRAINT 语句。 |
| BaseDDLElement | DDL 构造的根，包括“创建表”和其他流程中的子元素。 |
| CreateColumn | 以在 CREATE TABLE 语句中呈现的方式表示`Column`，通过`CreateTable`构造。 |
| CreateIndex | 代表一个 CREATE INDEX 语句。 |
| CreateSchema | 代表一个 CREATE SCHEMA 语句。 |
| CreateSequence | 代表一个 CREATE SEQUENCE 语句。 |
| CreateTable | 代表一个 CREATE TABLE 语句。 |
| DDL | 字面的 DDL 语句。 |
| DropConstraint | 代表一个 ALTER TABLE DROP CONSTRAINT 语句。 |
| DropIndex | 代表一个 DROP INDEX 语句。 |
| DropSchema | 代表一个 DROP SCHEMA 语句。 |
| DropSequence | 代表一个 DROP SEQUENCE 语句。 |
| DropTable | 代表一个 DROP TABLE 语句。 |
| ExecutableDDLElement | 独立可执行的 DDL 表达式构造的基类。 |
| sort_tables(tables[, skip_fn, extra_dependencies]) | 基于依赖关系对一组`Table`对象进行排序。 |
| sort_tables_and_constraints(tables[, filter_fn, extra_dependencies, _warn_for_cycles]) | 对一组`Table` / `ForeignKeyConstraint` 对象进行排序。 |

```py
function sqlalchemy.schema.sort_tables(tables: Iterable[TableClause], skip_fn: Callable[[ForeignKeyConstraint], bool] | None = None, extra_dependencies: typing_Sequence[Tuple[TableClause, TableClause]] | None = None) → List[Table]
```

根据依赖关系对一组`Table`对象进行排序。

这是一个依赖排序，将发出`Table`对象，以便它们将遵循其依赖的`Table`对象。表依赖于另一个表，根据`ForeignKeyConstraint`对象的存在以及由`Table.add_is_dependent_on()`添加的显式依赖关系。 |

警告

`sort_tables()`函数本身无法自动解决表之间的依赖循环，这些循环通常是由相互依赖的外键约束引起的。当检测到这些循环时，这些表的外键将被从排序考虑中省略。当出现此条件时会发出警告，在未来的版本中将引发异常。不属于循环的表仍将按照依赖顺序返回。

为了解决这些循环，可以将`ForeignKeyConstraint.use_alter`参数应用于创建循环的约束。另外，当检测到循环时，`sort_tables_and_constraints()`函数将自动将外键约束返回到一个单独的集合中，以便可以单独应用到模式中。

在版本 1.3.17 中更改：当`sort_tables()`由于循环依赖而无法执行适当排序时，会发出警告。这将在未来的版本中成为异常。此外，排序将继续返回其他未涉及循环的表以依赖顺序，而这在以前的情况下并非如此。

参数：

+   `tables` – 一系列`Table`对象。

+   `skip_fn` – 可选的可调用对象，将传递一个`ForeignKeyConstraint`对象；如果返回 True，则不会考虑此约束作为依赖项。请注意，这与`sort_tables_and_constraints()`中相同参数的不同之处，该参数将传递给拥有的`ForeignKeyConstraint`对象。

+   `extra_dependencies` – 一个包含表之间互相依赖的 2 元组序列。

另请参阅

`sort_tables_and_constraints()`函数用于排序表和约束。

`MetaData.sorted_tables` - 使用此函数进行排序。

```py
function sqlalchemy.schema.sort_tables_and_constraints(tables, filter_fn=None, extra_dependencies=None, _warn_for_cycles=False)
```

对一组`Table` / `ForeignKeyConstraint`对象进行排序。

这是一个依赖顺序排序，将发出`(Table, [ForeignKeyConstraint, ...])`元组，以便每个`Table`都跟随其依赖的`Table`对象。由于排序未满足的依赖规则而分开的其余`ForeignKeyConstraint`对象将作为`(None, [ForeignKeyConstraint ...])`之后发出。

表依赖于另一个表，基于存在的`ForeignKeyConstraint`对象，由`Table.add_is_dependent_on()`添加的显式依赖关系，以及在此处使用`sort_tables_and_constraints.skip_fn`和/或`sort_tables_and_constraints.extra_dependencies`参数声明的依赖关系。

参数:

+   `tables` – 一个包含`Table`对象的序列。

+   `filter_fn` – 可选的可调用对象，将传递一个`ForeignKeyConstraint`对象，并根据此约束是否应明确包含或排除为内联约束返回一个值，或者两者都不是。如果返回 False，则该约束将明确包含为不能受 ALTER 影响的依赖项；如果为 True，则它将`仅`作为最终的 ALTER 结果包含。返回 None 意味着该约束将包含在基于表的结果中，除非它被检测为依赖循环的一部分。

+   `extra_dependencies` – 一个包含两个表的 2 元组序列，这两个表也将被视为相互依赖。

另请参阅

`sort_tables()`

```py
class sqlalchemy.schema.BaseDDLElement
```

DDL 构造的根，包括那些作为“create table”和其他过程中的子元素的构造。

版本 2.0 中的新功能。

**类签名**

类`sqlalchemy.schema.BaseDDLElement`（`sqlalchemy.sql.expression.ClauseElement`）。

```py
class sqlalchemy.schema.ExecutableDDLElement
```

独立可执行的 DDL 表达式构造的基类。

此类是通用目的 `DDL` 类的基类，以及各种创建/删除子句构造，例如 `CreateTable`、`DropTable`、`AddConstraint` 等等。

从版本 2.0 起更改：`ExecutableDDLElement` 从 `DDLElement` 重命名，`DDLElement` 仍然存在以保持向后兼容性。

`ExecutableDDLElement` 与 SQLAlchemy 事件密切集成，介绍在 Events 中。一个实例本身就是一个事件接收器：

```py
event.listen(
    users,
    'after_create',
    AddConstraint(constraint).execute_if(dialect='postgresql')
)
```

另请参见

`DDL`

`DDLEvents`

Events

控制 DDL 序列

**成员**

__call__(), against(), execute_if()

**类签名**

类 `sqlalchemy.schema.ExecutableDDLElement` (`sqlalchemy.sql.roles.DDLRole`, `sqlalchemy.sql.expression.Executable`, `sqlalchemy.schema.BaseDDLElement`)

```py
method __call__(target, bind, **kw)
```

将 DDL 作为 ddl_listener 执行。

```py
method against(target: SchemaItem) → Self
```

返回此 `ExecutableDDLElement` 的副本，其中将包含给定的目标。

这实质上将给定项应用于返回的 `ExecutableDDLElement` 对象的 `.target` 属性。然后可以由事件处理程序和编译例程使用此目标，以提供诸如基于特定 `Table` 的 DDL 字符串的标记化之类的服务。

当将`ExecutableDDLElement`对象作为`DDLEvents.before_create()`或`DDLEvents.after_create()`事件的事件处理程序进行建立，并且事件随后发生在诸如`Constraint`或`Table`之类的给定目标上时，使用此方法使用`ExecutableDDLElement`对象的副本来为该目标建立，然后进入`ExecutableDDLElement.execute()`方法以调用实际的 DDL 指令。

参数：

**target** – 一个`SchemaItem`，将成为 DDL 操作的主体。

返回：

此`ExecutableDDLElement`的副本，其`.target`属性分配给给定的`SchemaItem`。

另请参阅

`DDL` - 对处理 DDL 字符串时针对“target”进行标记化。

```py
method execute_if(dialect: str | None = None, callable_: DDLIfCallable | None = None, state: Any | None = None) → Self
```

返回一个可调用对象，该对象将在事件处理程序中有条件地执行此`ExecutableDDLElement`。

用于提供事件监听的包装器：

```py
event.listen(
            metadata,
            'before_create',
            DDL("my_ddl").execute_if(dialect='postgresql')
        )
```

参数：

+   `dialect` –

    可以是字符串或字符串元组。如果是字符串，则将其与执行数据库方言的名称进行比较：

    ```py
    DDL('something').execute_if(dialect='postgresql')
    ```

    如果是元组，则指定多个方言名称：

    ```py
    DDL('something').execute_if(dialect=('postgresql', 'mysql'))
    ```

+   `callable_` –

    一个可调用对象，将使用三个位置参数以及可选的关键字参数进行调用：

    > ddl：
    > 
    > 此 DDL 元素。
    > 
    > target：
    > 
    > 此事件的目标`Table`或`MetaData`对象。如果 DDL 是显式执行的，则可以为 None。
    > 
    > bind：
    > 
    > 用于执行 DDL 的`Connection`。如果此构造是在表内创建的，则可以为 None，在这种情况下将存在`compiler`。
    > 
    > tables：
    > 
    > 可选关键字参数 - 一个 Table 对象列表，这些对象将在 MetaData.create_all() 或 drop_all() 方法调用时被创建/删除。
    > 
    > dialect：
    > 
    > 关键字参数，但始终存在 - 涉及操作的`Dialect`。
    > 
    > compiler：
    > 
    > 关键字参数。对于引擎级别的 DDL 调用，将为`None`，但如果此 DDL 元素在表内联中创建，则将引用`DDLCompiler`。
    > 
    > state:
    > 
    > 可选关键字参数 - 将作为此函数传递的`state`参数。
    > 
    > checkfirst：
    > 
    > 关键字参数，在调用`create()`、`create_all()`、`drop()`、`drop_all()`时，如果设置了`checkfirst`标志，则为 True。

    如果可调用函数返回 True 值，则将执行 DDL 语句。

+   `state` - 任何将作为`state`关键字参数传递给可调用函数的值。

另请参阅

`SchemaItem.ddl_if()`

`DDLEvents`

Events

```py
class sqlalchemy.schema.DDL
```

字面 DDL 语句。

指定要由数据库执行的字面 SQL DDL。DDL 对象充当 DDL 事件侦听器，并可以订阅`DDLEvents`中列出的事件，使用`Table`或`MetaData`对象作为目标。基本的模板支持允许单个 DDL 实例处理多个表的重复任务。

示例：

```py
from sqlalchemy import event, DDL

tbl = Table('users', metadata, Column('uid', Integer))
event.listen(tbl, 'before_create', DDL('DROP TRIGGER users_trigger'))

spow = DDL('ALTER TABLE %(table)s SET secretpowers TRUE')
event.listen(tbl, 'after_create', spow.execute_if(dialect='somedb'))

drop_spow = DDL('ALTER TABLE users SET secretpowers FALSE')
connection.execute(drop_spow)
```

在操作表事件时，以下`statement`字符串替换可用：

```py
%(table)s  - the Table name, with any required quoting applied
%(schema)s - the schema name, with any required quoting applied
%(fullname)s - the Table name including schema, quoted if needed
```

DDL 的“上下文”，如果有的话，将与上述标准替换组合。上下文中存在的键将覆盖标准替换。

**成员**

__init__()

**类签名**

类`sqlalchemy.schema.DDL`（`sqlalchemy.schema.ExecutableDDLElement`）

```py
method __init__(statement, context=None)
```

创建 DDL 语句。

参数：

+   `statement` -

    要执行的字符串或 Unicode 字符串。语句将使用 Python 的字符串格式化运算符处理，使用一组固定的字符串替换，以及可选的`DDL.context`参数提供的其他替换。

    语句中的字面‘%’必须转义为‘%%’。

    DDL 语句中不可用 SQL 绑定参数。

+   `context` - 可选字典，默认为 None。这些值将可用于 DDL 语句中的字符串替换。

另请参阅

`DDLEvents`

Events

```py
class sqlalchemy.schema._CreateDropBase
```

用于表示 CREATE 和 DROP 或等效操作的 DDL 构造的基类。

_CreateDropBase 的共同主题是一个`element`属性，指向要创建或删除的元素。

**类签名**

类`sqlalchemy.schema._CreateDropBase`（`sqlalchemy.schema.ExecutableDDLElement`）

```py
class sqlalchemy.schema.CreateTable
```

表示 CREATE TABLE 语句。

**成员**

__init__()

**类签名**

类`sqlalchemy.schema.CreateTable`（`sqlalchemy.schema._CreateBase`）

```py
method __init__(element: Table, include_foreign_key_constraints: typing_Sequence[ForeignKeyConstraint] | None = None, if_not_exists: bool = False)
```

创建一个`CreateTable`构造。

参数：

+   `element` – 是创建的主题`Table`。

+   `on` – 请参阅`DDL`中关于`on`的描述。

+   `include_foreign_key_constraints` – 将作为内联包含在 CREATE 构造中的可选的`ForeignKeyConstraint`对象序列；如果省略，则包括所有未指定 use_alter=True 的外键约束。

+   `if_not_exists` –

    如果为 True，则将应用 IF NOT EXISTS 操作符到构造中。

    新版本 1.4.0b2 中新增。

```py
class sqlalchemy.schema.DropTable
```

表示 DROP TABLE 语句。

**成员**

__init__()

**类签名**

类`sqlalchemy.schema.DropTable`（`sqlalchemy.schema._DropBase`）

```py
method __init__(element: Table, if_exists: bool = False)
```

创建一个`DropTable`构造。

参数：

+   `element` – 是 DROP 的主题`Table`。

+   `on` – 请参阅`DDL`中关于`on`的描述。

+   `if_exists` –

    如果为 True，则将应用 IF EXISTS 操作符到构造中。

    新版本 1.4.0b2 中新增。

```py
class sqlalchemy.schema.CreateColumn
```

将`Column`表示为通过`CreateTable`构造在 CREATE TABLE 语句中呈现的形式。

这是为了支持在生成 CREATE TABLE 语句时使用 Custom SQL Constructs and Compilation Extension 中记录的编译器扩展来扩展`CreateColumn`以支持自定义列 DDL 而提供的。

典型的集成是检查传入的`Column`对象，并且在找到特定标志或条件时重定向编译：

```py
from sqlalchemy import schema
from sqlalchemy.ext.compiler import compiles

@compiles(schema.CreateColumn)
def compile(element, compiler, **kw):
    column = element.element

    if "special" not in column.info:
        return compiler.visit_create_column(element, **kw)

    text = "%s SPECIAL DIRECTIVE %s" % (
            column.name,
            compiler.type_compiler.process(column.type)
        )
    default = compiler.get_column_default_string(column)
    if default is not None:
        text += " DEFAULT " + default

    if not column.nullable:
        text += " NOT NULL"

    if column.constraints:
        text += " ".join(
                    compiler.process(const)
                    for const in column.constraints)
    return text
```

上述构造可以应用于`Table`如下：

```py
from sqlalchemy import Table, Metadata, Column, Integer, String
from sqlalchemy import schema

metadata = MetaData()

table = Table('mytable', MetaData(),
        Column('x', Integer, info={"special":True}, primary_key=True),
        Column('y', String(50)),
        Column('z', String(20), info={"special":True})
    )

metadata.create_all(conn)
```

以上，我们添加到`Column.info`集合中的指令将被我们自定义的编译方案检测到：

```py
CREATE TABLE mytable (
        x SPECIAL DIRECTIVE INTEGER NOT NULL,
        y VARCHAR(50),
        z SPECIAL DIRECTIVE VARCHAR(20),
    PRIMARY KEY (x)
)
```

`CreateColumn`构造也可以用于在生成`CREATE TABLE`时跳过某些列。这是通过创建一个有条件返回`None`的编译规则来实现的。这实质上就是如何产生与在`Column`上使用`system=True`参数相同的效果，该参数将列标记为隐含的“系统”列。

例如，假设我们希望生成一个`Table`，在 PostgreSQL 后端跳过渲染 PostgreSQL `xmin`列，但在其他后端进行渲染，以预期触发规则。一个条件编译规则可以仅在 PostgreSQL 上跳过此名称：

```py
from sqlalchemy.schema import CreateColumn

@compiles(CreateColumn, "postgresql")
def skip_xmin(element, compiler, **kw):
    if element.element.name == 'xmin':
        return None
    else:
        return compiler.visit_create_column(element, **kw)

my_table = Table('mytable', metadata,
            Column('id', Integer, primary_key=True),
            Column('xmin', Integer)
        )
```

上面，一个`CreateTable`结构将仅在字符串中包含`id`列；`xmin`列将被省略，但仅针对 PostgreSQL 后端。

**类签名**

类`sqlalchemy.schema.CreateColumn`（`sqlalchemy.schema.BaseDDLElement`）

```py
class sqlalchemy.schema.CreateSequence
```

表示一个 CREATE SEQUENCE 语句。

**类签名**

类`sqlalchemy.schema.CreateSequence`（`sqlalchemy.schema._CreateBase`）

```py
class sqlalchemy.schema.DropSequence
```

表示一个 DROP SEQUENCE 语句。

**类签名**

类`sqlalchemy.schema.DropSequence`（`sqlalchemy.schema._DropBase`）

```py
class sqlalchemy.schema.CreateIndex
```

表示一个 CREATE INDEX 语句。

**成员**

__init__()

**类签名**

类`sqlalchemy.schema.CreateIndex`（`sqlalchemy.schema._CreateBase`）

```py
method __init__(element, if_not_exists=False)
```

创建一个`Createindex`结构。

参数：

+   `element` – 一个`Index`，是 CREATE 的主题。

+   `if_not_exists` –

    如果为 True，则将 IF NOT EXISTS 运算符应用于该结构。

    新版本 1.4.0b2 中新增。

```py
class sqlalchemy.schema.DropIndex
```

表示一个 DROP INDEX 语句。

**成员**

__init__()

**类签名**

类`sqlalchemy.schema.DropIndex`（`sqlalchemy.schema._DropBase`）

```py
method __init__(element, if_exists=False)
```

创建一个`DropIndex`结构。

参数：

+   `element` – 一个`Index`，是 DROP 的主题。

+   `if_exists` –

    如果为 True，则将 IF EXISTS 运算符应用于该结构。

    新版本 1.4.0b2 中新增。

```py
class sqlalchemy.schema.AddConstraint
```

表示一个 ALTER TABLE ADD CONSTRAINT 语句。

**类签名**

类 `sqlalchemy.schema.AddConstraint` (`sqlalchemy.schema._CreateBase`)

```py
class sqlalchemy.schema.DropConstraint
```

表示一个 ALTER TABLE DROP CONSTRAINT 语句。

**类签名**

类 `sqlalchemy.schema.DropConstraint` (`sqlalchemy.schema._DropBase`)

```py
class sqlalchemy.schema.CreateSchema
```

表示一个 CREATE SCHEMA 语句。

这里的参数是模式的字符串名称。

**成员**

__init__()

**类签名**

类 `sqlalchemy.schema.CreateSchema` (`sqlalchemy.schema._CreateBase`)

```py
method __init__(name, if_not_exists=False)
```

创建一个新的 `CreateSchema` 构造。

```py
class sqlalchemy.schema.DropSchema
```

表示一个 DROP SCHEMA 语句。

这里的参数是模式的字符串名称。

**成员**

__init__()

**类签名**

类 `sqlalchemy.schema.DropSchema` (`sqlalchemy.schema._DropBase`)

```py
method __init__(name, cascade=False, if_exists=False)
```

创建一个新的 `DropSchema` 构造。
