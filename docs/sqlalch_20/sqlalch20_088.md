# 描述数据库与元数据

> 原文：[`docs.sqlalchemy.org/en/20/core/metadata.html`](https://docs.sqlalchemy.org/en/20/core/metadata.html)

本节讨论了基本的`Table`、`Column`和`MetaData`对象。

请参阅

使用数据库元数据 - SQLAlchemy 的数据库元数据概念入门教程，位于 SQLAlchemy 统一教程中

一个元数据实体的集合存储在一个名为`MetaData`的对象中：

```py
from sqlalchemy import MetaData

metadata_obj = MetaData()
```

`MetaData`是一个容器对象，它将数据库（或多个数据库）的许多不同特征组合在一起进行描述。

要表示一个表，请使用`Table`类。它的两个主要参数是表名称，然后是它将关联的`MetaData`对象。其余的位置参数大多是描述每列的`Column`对象：

```py
from sqlalchemy import Table, Column, Integer, String

user = Table(
    "user",
    metadata_obj,
    Column("user_id", Integer, primary_key=True),
    Column("user_name", String(16), nullable=False),
    Column("email_address", String(60)),
    Column("nickname", String(50), nullable=False),
)
```

上面描述了一个名为`user`的表，其中包含四列。表的主键由`user_id`列组成。可以将多个列分配`primary_key=True`标志，表示多列主键，称为*复合*主键。

还要注意，每个列使用与通用化类型对应的对象来描述其数据类型，例如`Integer`和`String`。SQLAlchemy 具有几十种不同级别的类型以及创建自定义类型的能力。有关类型系统的文档可以在 SQL 数据类型对象中找到。

## 访问表和列

`MetaData`对象包含了我们与其关联的所有模式构造。它支持几种访问这些表对象的方法，例如`sorted_tables`访问器，它以外键依赖顺序返回每个`Table`对象的列表（也就是说，每个表都在其引用的所有表之前）：

```py
>>> for t in metadata_obj.sorted_tables:
...     print(t.name)
user
user_preference
invoice
invoice_item
```

在大多数情况下，单个`Table`对象已经被明确声明，并且这些对象通常直接作为应用程序中的模块级变量访问。 一旦定义了一个`Table`，它就有了一整套访问器，允许检查其属性。 给定以下`Table`定义：

```py
employees = Table(
    "employees",
    metadata_obj,
    Column("employee_id", Integer, primary_key=True),
    Column("employee_name", String(60), nullable=False),
    Column("employee_dept", Integer, ForeignKey("departments.department_id")),
)
```

请注意在这个表中使用的`ForeignKey`对象 - 这个结构定义了对远程表的引用，并在定义外键中进行了全面描述。 访问关于这个表的信息的方法包括：

```py
# access the column "employee_id":
employees.columns.employee_id

# or just
employees.c.employee_id

# via string
employees.c["employee_id"]

# a tuple of columns may be returned using multiple strings
# (new in 2.0)
emp_id, name, type = employees.c["employee_id", "name", "type"]

# iterate through all columns
for c in employees.c:
    print(c)

# get the table's primary key columns
for primary_key in employees.primary_key:
    print(primary_key)

# get the table's foreign key objects:
for fkey in employees.foreign_keys:
    print(fkey)

# access the table's MetaData:
employees.metadata

# access a column's name, type, nullable, primary key, foreign key
employees.c.employee_id.name
employees.c.employee_id.type
employees.c.employee_id.nullable
employees.c.employee_id.primary_key
employees.c.employee_dept.foreign_keys

# get the "key" of a column, which defaults to its name, but can
# be any user-defined string:
employees.c.employee_name.key

# access a column's table:
employees.c.employee_id.table is employees

# get the table related by a foreign key
list(employees.c.employee_dept.foreign_keys)[0].column.table
```

提示

`FromClause.c`集合，与`FromClause.columns`集合同义，是`ColumnCollection`的一个实例，它提供了类似于字典的接口来访问列集合。 名称通常像属性名称那样访问，例如 `employees.c.employee_name`。 但是，对于具有空格的特殊名称或与字典方法名称匹配的名称，例如`ColumnCollection.keys()`或`ColumnCollection.values()`，必须使用索引访问，例如 `employees.c['values']` 或 `employees.c["some column"]`。 有关更多信息，请参阅`ColumnCollection`。

## 创建和删除数据库表

一旦您定义了一些`Table`对象，假设您正在使用全新的数据库，您可能想要做的一件事是为这些表及其相关结构发出 CREATE 语句（顺便说一句，如果您已经有了一些首选的方法，比如与数据库一起提供的工具或现有的脚本系统 - 如果是这种情况，请随意跳过此部分 - SQLAlchemy 不要求使用它来创建您的表）。

发出 CREATE 的常规方式是在`MetaData`对象上使用`create_all()`。此方法将发出查询，首先检查每个单独表的存在性，如果未找到，则发出 CREATE 语句：

```py
engine = create_engine("sqlite:///:memory:")

metadata_obj = MetaData()

user = Table(
    "user",
    metadata_obj,
    Column("user_id", Integer, primary_key=True),
    Column("user_name", String(16), nullable=False),
    Column("email_address", String(60), key="email"),
    Column("nickname", String(50), nullable=False),
)

user_prefs = Table(
    "user_prefs",
    metadata_obj,
    Column("pref_id", Integer, primary_key=True),
    Column("user_id", Integer, ForeignKey("user.user_id"), nullable=False),
    Column("pref_name", String(40), nullable=False),
    Column("pref_value", String(100)),
)

metadata_obj.create_all(engine)
PRAGMA  table_info(user){}
CREATE  TABLE  user(
  user_id  INTEGER  NOT  NULL  PRIMARY  KEY,
  user_name  VARCHAR(16)  NOT  NULL,
  email_address  VARCHAR(60),
  nickname  VARCHAR(50)  NOT  NULL
)
PRAGMA  table_info(user_prefs){}
CREATE  TABLE  user_prefs(
  pref_id  INTEGER  NOT  NULL  PRIMARY  KEY,
  user_id  INTEGER  NOT  NULL  REFERENCES  user(user_id),
  pref_name  VARCHAR(40)  NOT  NULL,
  pref_value  VARCHAR(100)
) 
```

`create_all()`通常在表定义本身内联创建表之间的外键约束，并且出于这个原因，它也按照它们的依赖顺序生成表。有选项可以更改此行为，使其使用`ALTER TABLE`。

类似地，使用`drop_all()`方法可以删除所有表。此方法与`create_all()`完全相反-首先检查每个表的存在性，然后按依赖关系的相反顺序删除表。

创建和删除单个表可以通过`Table`的`create()`和`drop()`方法来完成。这些方法默认情况下会发出 CREATE 或 DROP 命令，而不管表是否存在：

```py
engine = create_engine("sqlite:///:memory:")

metadata_obj = MetaData()

employees = Table(
    "employees",
    metadata_obj,
    Column("employee_id", Integer, primary_key=True),
    Column("employee_name", String(60), nullable=False, key="name"),
    Column("employee_dept", Integer, ForeignKey("departments.department_id")),
)
employees.create(engine)
CREATE  TABLE  employees(
  employee_id  SERIAL  NOT  NULL  PRIMARY  KEY,
  employee_name  VARCHAR(60)  NOT  NULL,
  employee_dept  INTEGER  REFERENCES  departments(department_id)
)
{} 
```

`drop()`方法：

```py
employees.drop(engine)
DROP  TABLE  employees
{} 
```

要启用“首先检查表是否存在”的逻辑，请在`create()`或`drop()`中添加`checkfirst=True`参数：

```py
employees.create(engine, checkfirst=True)
employees.drop(engine, checkfirst=False)
```

## 通过迁移修改数据库对象

虽然 SQLAlchemy 直接支持为模式构造发出 CREATE 和 DROP 语句，但是修改这些构造的能力，通常通过 ALTER 语句以及其他特定于数据库的构造，超出了 SQLAlchemy 本身的范围。虽然手动发出 ALTER 语句等很容易，例如通过将`text()`构造传递给`Connection.execute()`或使用`DDL`构造，但通常的做法是使用模式迁移工具自动化维护数据库模式与应用程序代码的关系。

SQLAlchemy 项目为此提供了[Alembic](https://alembic.sqlalchemy.org)迁移工具。Alembic 具有高度可定制的环境和极简的使用模式，支持诸如事务 DDL、自动生成“候选”迁移、生成 SQL 脚本的“脱机”模式以及分支解析支持等功能。

Alembic 取代了[SQLAlchemy-Migrate](https://github.com/openstack/sqlalchemy-migrate)项目，这是 SQLAlchemy 的原始迁移工具，现在被视为遗留工具。## 指定模式名称

大多数数据库支持多个“模式”（schemas）的概念 - 指代备选表格和其他结构的命名空间。一个“模式”的服务器端几何形状有多种形式，包括特定数据库范围内的“模式”名称（例如 PostgreSQL 模式）、命名的同级数据库（例如 MySQL / MariaDB 访问同一服务器上的其他数据库）、以及其他概念，比如其他用户名拥有的表格（Oracle、SQL Server）甚至是指代备选数据库文件（SQLite ATTACH）或远程服务器（Oracle DBLINK with synonyms）的名称。

所有上述方法（大多数）共同之处在于有一种引用这个备选表格集的方式，使用一个字符串名称。SQLAlchemy 将这个名称称为**模式名称**。在 SQLAlchemy 中，这只是一个与`Table`对象关联的字符串名称，然后以适合目标数据库的方式渲染成 SQL 语句，以便表格在其远程“模式”中被引用，无论目标数据库上的机制是什么。

“模式”名称可以直接与`Table`关联，使用`Table.schema`参数；当使用 ORM 与声明式表格配置时，参数通过`__table_args__`参数字典传递。

“模式”名称也可以与`MetaData`对象关联，在这种情况下，它将自动对所有与该`MetaData`关联的未另行指定名称的`Table`对象生效。最后，SQLAlchemy 还支持一种“动态”模式名称系统，通常用于多租户应用程序，以便单个`Table`元数据集可以根据每个连接或每个语句的基础动态配置的模式名称集。

另请参阅

使用声明式表格的显式模式名称 - 在使用 ORM 声明式表格配置时指定模式名称

最基本的例子是使用 Core `Table` 对象的`Table.schema`参数，如下所示：

```py
metadata_obj = MetaData()

financial_info = Table(
    "financial_info",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    Column("value", String(100), nullable=False),
    schema="remote_banks",
)
```

使用这个`Table`渲染的 SQL，比如下面的 SELECT 语句，将会明确使用`remote_banks`模式名称来限定表格名称`financial_info`：

```py
>>> print(select(financial_info))
SELECT  remote_banks.financial_info.id,  remote_banks.financial_info.value
FROM  remote_banks.financial_info 
```

当使用显式模式名称声明`Table`对象时，它将使用模式和表名的组合存储在内部`MetaData`命名空间中。我们可以通过搜索键`'remote_banks.financial_info'`在`MetaData.tables`集合中查看此内容：

```py
>>> metadata_obj.tables["remote_banks.financial_info"]
Table('financial_info', MetaData(),
Column('id', Integer(), table=<financial_info>, primary_key=True, nullable=False),
Column('value', String(length=100), table=<financial_info>, nullable=False),
schema='remote_banks')
```

此点名也是在引用用于与`ForeignKey`或`ForeignKeyConstraint`对象一起使用的表时必须使用的内容，即使引用表也在同一模式中：

```py
customer = Table(
    "customer",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    Column("financial_info_id", ForeignKey("remote_banks.financial_info.id")),
    schema="remote_banks",
)
```

参数`Table.schema`也可以与某些方言一起使用，以指示到特定表的多个标记路径（例如，点分）。

```py
schema = "dbo.scott"
```

另请参阅

多部分模式名称 - 描述了在 SQL Server 方言中使用点分模式名称的情况。

从其他模式反射表

### 使用 MetaData 指定默认模式名称

`MetaData`对象还可以通过将`MetaData.schema`参数传递给顶层`MetaData`构造来为所有`Table.schema`参数设置一个明确的默认选项：

```py
metadata_obj = MetaData(schema="remote_banks")

financial_info = Table(
    "financial_info",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    Column("value", String(100), nullable=False),
)
```

以上，对于任何（直接与`MetaData`相关联的`Table`对象或`Sequence`对象）将`Table.schema`参数保持在其默认值`None`的情况，将会作为参数设置为值`"remote_banks"`。这包括使用模式限定名称在`MetaData`中对`Table`进行目录化，即：

```py
metadata_obj.tables["remote_banks.financial_info"]
```

当使用`ForeignKey`或`ForeignKeyConstraint`对象引用此表时，可以使用模式限定名称或非模式限定名称来引用`remote_banks.financial_info`表：

```py
# either will work:

refers_to_financial_info = Table(
    "refers_to_financial_info",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    Column("fiid", ForeignKey("financial_info.id")),
)

# or

refers_to_financial_info = Table(
    "refers_to_financial_info",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    Column("fiid", ForeignKey("remote_banks.financial_info.id")),
)
```

当使用设置了`MetaData.schema`的`MetaData`对象时，希望指定不应限定模式的`Table`可以使用特殊符号`BLANK_SCHEMA`：

```py
from sqlalchemy import BLANK_SCHEMA

metadata_obj = MetaData(schema="remote_banks")

financial_info = Table(
    "financial_info",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    Column("value", String(100), nullable=False),
    schema=BLANK_SCHEMA,  # will not use "remote_banks"
)
```

另见

`MetaData.schema`  ### 应用动态模式命名约定

`Table.schema`参数使用的名称也可以应用于对每个连接或每个执行动态的查找，因此例如在多租户情况下，每个事务或语句可能针对一组不同的模式名称进行定位。本节模式名称的翻译描述了如何使用此功能。

另见

模式名称的翻译  ### 为新连接设置默认模式

上述方法都涉及在 SQL 语句中包含显式模式名称的方法。数据库连接实际上具有“默认”模式的概念，这是一个“模式”（或数据库、所有者等）的名称，如果表名没有显式地限定模式，则会发生。这些名称通常在登录级别配置，例如连接到 PostgreSQL 数据库时，默认的“模式”称为“public”。

通常情况下，默认的“模式”无法通过登录本身设置，而是在每次建立连接时有用地配置，使用诸如在 PostgreSQL 上使用“SET SEARCH_PATH”或在 Oracle 上使用“ALTER SESSION”的语句。这些方法可以通过使用`PoolEvents.connect()`事件来实现，该事件允许在首次创建 DBAPI 连接时访问该连接。例如，要将 Oracle 的 CURRENT_SCHEMA 变量设置为替代名称：

```py
from sqlalchemy import event
from sqlalchemy import create_engine

engine = create_engine("oracle+cx_oracle://scott:tiger@tsn_name")

@event.listens_for(engine, "connect", insert=True)
def set_current_schema(dbapi_connection, connection_record):
    cursor_obj = dbapi_connection.cursor()
    cursor_obj.execute("ALTER SESSION SET CURRENT_SCHEMA=%s" % schema_name)
    cursor_obj.close()
```

在上述情况中，当上述`Engine`首次连接时，`set_current_schema()`事件处理程序将立即发生；由于该事件被“插入”到处理程序列表的开头，因此它将在方言自己的事件处理程序运行之前发生，特别是包括确定连接的“默认模式”的事件处理程序。

对于其他数据库，请查阅特定信息的数据库和/或方言文档，了解如何配置默认模式的详细信息。

在版本 1.4.0b2 中更改：上述方法现在无需建立额外的事件处理程序即可运行。

另请参见

在 Connect 上设置备用搜索路径 - 在 PostgreSQL 方言文档中。

### 模式和反射

SQLAlchemy 的模式功能与在 反射数据库对象 中介绍的表反射功能相互作用。请参阅 从其他模式反射表 部分，了解此功能的更多详细信息。

## 后端特定选项

`Table` 支持特定于数据库的选项。例如，MySQL 有不同的表后端类型，包括“MyISAM”和“InnoDB”。这可以通过 `Table` 使用 `mysql_engine` 表达：

```py
addresses = Table(
    "engine_email_addresses",
    metadata_obj,
    Column("address_id", Integer, primary_key=True),
    Column("remote_user_id", Integer, ForeignKey(users.c.user_id)),
    Column("email_address", String(20)),
    mysql_engine="InnoDB",
)
```

其他后端可能也支持表级选项 - 这些将在每个方言的个别文档部分中描述。

## 列、表、MetaData API

| 对象名称 | 描述 |
| --- | --- |
| 列 | 代表数据库表中的列。 |
| insert_sentinel([name, type_], *, [default, omit_from_statements]) | 提供一个虚拟的 `Column`，将作为专用的插入 sentinel 列，允许对没有其他符合主键配置的表进行高效的批量插入，并具有确定性的 RETURNING 排序。 |
| MetaData | 一组 `Table` 对象及其关联的模式构造。 |
| SchemaConst | 一个枚举。 |
| SchemaItem | 用于定义数据库模式的项目的基类。 |
| 表 | 代表数据库中的表。 |

```py
attribute sqlalchemy.schema.BLANK_SCHEMA
```

指的是 `SchemaConst.BLANK_SCHEMA`。

```py
attribute sqlalchemy.schema.RETAIN_SCHEMA
```

指的是 `SchemaConst.RETAIN_SCHEMA`

```py
class sqlalchemy.schema.Column
```

代表数据库表中的列。

**成员**

__eq__(), __init__(), __le__(), __lt__(), __ne__(), all_(), anon_key_label, anon_label, any_(), argument_for(), asc(), between(), bitwise_and(), bitwise_lshift(), bitwise_not(), bitwise_or(), bitwise_rshift(), bitwise_xor(), bool_op(), cast(), collate(), compare(), compile(), concat(), contains(), copy(), desc(), dialect_kwargs, dialect_options, distinct(), endswith(), expression, foreign_keys, get_children(), icontains(), iendswith(), ilike(), in_(), index, info, inherit_cache, is_(), is_distinct_from(), is_not(), is_not_distinct_from(), isnot(), isnot_distinct_from(), istartswith(), key, kwargs, label(), like(), match(), not_ilike(), not_in(), not_like(), notilike(), notin_(), notlike(), nulls_first(), nulls_last(), nullsfirst(), nullslast(), op(), operate(), params(), proxy_set, references(), regexp_match(), regexp_replace(), reverse_operate(), self_group(), shares_lineage(), startswith(), timetuple, unique, unique_params()

**类签名**

类`sqlalchemy.schema.Column`（`sqlalchemy.sql.base.DialectKWArgs`，`sqlalchemy.schema.SchemaItem`，`sqlalchemy.sql.expression.ColumnClause`）

```py
method __eq__(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators` *的* `sqlalchemy.sql.expression.ColumnOperators.__eq__` *方法*

实现`==`运算符。

在列上下文中，生成子句`a = b`。如果目标是`None`，则生成`a IS NULL`。

```py
method __init__(_Column__name_pos: str | _TypeEngineArgument[_T] | SchemaEventTarget | None = None, _Column__type_pos: _TypeEngineArgument[_T] | SchemaEventTarget | None = None, *args: SchemaEventTarget, name: str | None = None, type_: _TypeEngineArgument[_T] | None = None, autoincrement: _AutoIncrementType = 'auto', default: Any | None = None, doc: str | None = None, key: str | None = None, index: bool | None = None, unique: bool | None = None, info: _InfoType | None = None, nullable: bool | Literal[SchemaConst.NULL_UNSPECIFIED] | None = SchemaConst.NULL_UNSPECIFIED, onupdate: Any | None = None, primary_key: bool = False, server_default: _ServerDefaultArgument | None = None, server_onupdate: FetchedValue | None = None, quote: bool | None = None, system: bool = False, comment: str | None = None, insert_sentinel: bool = False, _omit_from_statements: bool = False, _proxies: Any | None = None, **dialect_kwargs: Any)
```

构造一个新的`Column`对象。

参数：

+   `name` –

    在数据库中表示此列的名称。此参数可以是第一个位置参数，也可以通过关键字指定。

    不包含大写字符的名称将被视为大小写不敏感的名称，并且除非它们是保留字，否则不会被引用。包含任意数量大写字符的名称将被引用并发送完全相同。请注意，即使对于标准化大写名称为大小写不敏感的数据库（例如 Oracle），此行为也适用。

    名称字段可以在构建时省略，并在与`Table`关联之前的任何时候应用。这是为了支持在`declarative`扩展中的方便使用。

+   `type_` –

    列的类型，使用一个继承自`TypeEngine`的实例来表示。如果类型不需要参数，则也可以发送类型的类，例如：

    ```py
    # use a type with arguments
    Column('data', String(50))

    # use no arguments
    Column('level', Integer)
    ```

    `type`参数可以是第二个位置参数或通过关键字指定。

    如果`type`为`None`或省略，则首先默认为特殊类型`NullType`。如果并且当此`Column`被指定为引用另一列时，使用`ForeignKey`和/或`ForeignKeyConstraint`，远程引用列的类型也将被复制到此列中，在解析外键与该远程`Column`对象相匹配的时刻。

+   `*args` – 附加的位置参数包括各种派生自`SchemaItem`的构造，这些构造将作为选项应用于列。这些包括`Constraint`、`ForeignKey`、`ColumnDefault`、`Sequence`、`Computed`和`Identity`的实例。在某些情况下，可能会提供等效的关键字参数，例如`server_default`、`default`和`unique`。

+   `autoincrement` –

    为**没有外键依赖的整数主键列设置“自动递增”语义**（有关更具体的定义，请参见本文档字符串后面）。这可能会影响在创建表期间为此列发出的 DDL，以及在编译和执行 INSERT 语句时如何考虑该列。

    默认值是字符串`"auto"`，表示应自动为单列（即非复合）主键提供自动递增语义，该主键为 INTEGER 类型且没有其他客户端或服务器端默认构造指示。其他值包括`True`（强制此列具有自动递增语义以供复合主键使用）、`False`（此列永远不应具有自动递增语义）和字符串`"ignore_fk"`（外键列的特殊情况，请参见下文）。

    “自动递增语义”一词既指在 CREATE TABLE 语句中为列发出的 DDL 的类型，也指在调用诸如`MetaData.create_all()`和`Table.create()`等方法时，以及在编译和发出 INSERT 语句到数据库时如何考虑该列：

    +   `DDL 渲染`（即，`MetaData.create_all()`，`Table.create()`）：当用于没有其他默认生成构造与之关联的 `Column`（例如 `Sequence` 或 `Identity` 构造）时，该参数将暗示应该渲染数据库特定的关键字，如 PostgreSQL 中的 `SERIAL`，MySQL 中的 `AUTO_INCREMENT`，或 SQL Server 中的 `IDENTITY`。并非每个数据库后端都有“隐式”的默认生成器可用；例如，Oracle 后端总是需要一个显式的构造，例如 `Identity`，以便在渲染的 DDL 中包括自动生成的构造，也在数据库中生成。

    +   `INSERT` 语义（即，当 `insert()` 构造编译成 SQL 字符串并使用 `Connection.execute()` 或等效方法在数据库上执行时）：单行 INSERT 语句将自动为此列生成一个新的整数主键值，该值在语句被调用后可通过 `Result` 对象上的 `CursorResult.inserted_primary_key` 属性访问。当使用 ORM 持久化 ORM 映射对象到数据库时，这也适用，表示一个新的整数主键将可用作该对象的 identity key 的一部分。无论与 `Column` 关联的 DDL 构造是什么，此行为都会发生，并且与上面讨论的“DDL 渲染”行为无关。

    该参数可以设置为 `True`，以指示作为复合（即多列）主键的列应具有自动递增语义，但请注意，主键中仅有一个列可以具有此设置。它还可以设置为 `True`，以指示在具有客户端或服务器端默认配置的列上具有自动递增语义，但请注意，并非所有方言都可以适应所有样式的默认值作为“自动递增”。它还可以在具有 INTEGER 数据类型的单列主键上设置为 `False`，以禁用该列的自动递增语义。

    *仅*对以下列有效：

    +   整数衍生（即 INT、SMALLINT、BIGINT）。

    +   主键的一部分

    +   不通过`ForeignKey`来引用另一列，除非该值被指定为`'ignore_fk'`：

        ```py
        # turn on autoincrement for this column despite
        # the ForeignKey()
        Column('id', ForeignKey('other.id'),
                    primary_key=True, autoincrement='ignore_fk')
        ```

    通常不希望启用“自动递增”功能于通过外键引用另一列的情况，因为这样的列必须引用源自其他地方的值。

    该设置对满足上述条件的列有以下效果：

    +   如果列尚未包括由后端支持的默认生成结构（如 `Identity`），则为该列发出的 DDL 将包含特定于数据库的关键字，用于表示该列为特定后端的“自动递增”列。主要 SQLAlchemy 方言的行为包括：

        +   MySQL 和 MariaDB 上的 AUTO INCREMENT

        +   在 PostgreSQL 上的 SERIAL

        +   MS-SQL 上的 IDENTITY - 即使没有 `Identity` 结构，这也会发生，因为 `Column.autoincrement` 参数早于此结构存在。

        +   SQLite - SQLite 整数主键列隐式为“自动递增”，不会渲染任何附加关键字；不包括特殊的 SQLite 关键字 `AUTOINCREMENT`，因为这是不必要的，也不被数据库供应商推荐。有关更多背景信息，请参阅 SQLite 自动递增行为部分。

        +   Oracle - Oracle 方言目前没有默认的“自动递增”功能可用，因此建议使用 `Identity` 结构来实现此目的（也可以使用 `Sequence` 结构）。

        +   第三方方言 - 请参阅这些方言的文档，了解其特定行为的详情。

    +   当编译并执行单行`insert()`构造时，该构造未设置`Insert.inline()`修饰符，并且使用的数据库驱动程序不会自动在语句执行时检索新生成的主键值时，将自动检索此列的新生成的主键值：

        +   MySQL、SQLite - 调用 `cursor.lastrowid()`（请参阅[`www.python.org/dev/peps/pep-0249/#lastrowid`](https://www.python.org/dev/peps/pep-0249/#lastrowid)）

        +   PostgreSQL、SQL Server、Oracle - 在渲染 INSERT 语句时使用 RETURNING 或等效结构，并在执行后检索新生成的主键值。

        +   对于将 `Table.implicit_returning` 设置为 False 的`Table`对象的 PostgreSQL、Oracle - 仅对于 `Sequence`，在执行 INSERT 语句之前显式调用 `Sequence`，以便新生成的主键值对客户端可用。

        +   对于将 `Table.implicit_returning` 设置为 False 的`Table`对象的 SQL Server - 在调用 INSERT 语句后使用 `SELECT scope_identity()` 构造来检索新生成的主键值。

        +   第三方方言 - 请查阅这些方言的文档，了解其特定行为的详细信息。

    +   对于使用参数列表（即“executemany”语义）调用的多行`insert()`构造，通常会禁用主键检索行为，但是可能有特殊的 API 可以用于检索“executemany”中新主键值的列表，例如 psycopg2 的“fast insertmany”功能。此类功能非常新，可能尚未在文档中充分介绍。

+   `default` –

    表示此列的*默认值*的标量、Python 可调用对象或`ColumnElement`表达式，如果此列在 INSERT 的 VALUES 子句中未指定，则将在插入时调用此值。这是使用`ColumnDefault`作为位置参数的一种快捷方式；请参阅该类以获取有关参数结构的完整详细信息。

    将此参数与`Column.server_default`进行对比，后者在数据库端创建默认生成器。

    请参阅

    列插入/更新默认值

+   `doc` – 可选的字符串，可被 ORM 或类似的程序用于在 Python 端记录属性。此属性不会渲染 SQL 注释；为此目的，请使用`Column.comment`参数。

+   `key` – 可选的字符串标识符，将用于标识此`Column`对象在`Table`上。提供关键字时，这是应用程序中引用`Column`的唯一标识符，包括 ORM 属性映射；`name`字段仅在渲染 SQL 时使用。

+   `index` –

    当为`True`时，表示将为此`Column`自动生成一个`Index`构造，这将导致在调用 DDL 创建操作时为`Table`发出“CREATE INDEX”语句。

    使用此标志等效于在`Table`构造本身的层次上显式使用`Index`构造：

    ```py
    Table(
        "some_table",
        metadata,
        Column("x", Integer),
        Index("ix_some_table_x", "x")
    )
    ```

    若要将`Index.unique`标志添加到`Index`中，请同时将`Column.unique`和`Column.index`标志设置为 True，这将导致发出“CREATE UNIQUE INDEX”DDL 指令而不是“CREATE INDEX”。

    索引的名称使用默认命名约定生成，对于`Index`构造，其形式为`ix_<tablename>_<columnname>`。

    由于此标志仅旨在为常见情况（向表定义添加单列默认配置的索引）提供便利，因此大多数情况下应首选显式使用`Index`构造，包括跨越多个列的复合索引，具有 SQL 表达式或排序的索引，后端特定的索引配置选项以及使用特定名称的索引。

    注意

    `Column.index`属性在`Column`上**并不表示**此列是否已建立索引，只表示此标志是否在此处明确设置。要查看列上的索引，请查看`Table.indexes`集合或使用`Inspector.get_indexes()`。

    另请参阅

    索引

    配置约束命名约定

    `Column.unique`

+   `info` – 可选数据字典，将填充到此对象的`SchemaItem.info`属性中。

+   `nullable` –

    当设置为`False`时，在生成列的 DDL 时将添加“NOT NULL”短语。当设置为`True`时，通常不会生成任何内容（在 SQL 中默认为“NULL”），除非在一些非常特定的后端特定边缘情况下，“NULL”可能会显式呈现。除非`Column.primary_key`也为`True`或列指定为`Identity`，否则默认为`True`。此参数仅在发出 CREATE TABLE 语句时使用。

    注意

    当列指定为`Identity`时，DDL 编译器通常会忽略此参数。PostgreSQL 数据库允许通过将此参数显式设置为`True`来创建可空的标识列。

+   `onupdate` –

    一个标量、Python 可调用对象或`ClauseElement`，表示要应用于列的默认值，在 UPDATE 语句中将在更新时调用，如果此列不在 UPDATE 语句的 SET 子句中，则将被应用。这是使用`ColumnDefault`作为位置参数与`for_update=True`的快捷方式。

    另请参阅

    列插入/更新默认值 - 完整讨论 onupdate

+   `primary_key` – 如果为`True`，将此列标记为主键列。可以将多个列设置此标志以指定复合主键。作为替代，可以通过显式的`PrimaryKeyConstraint`对象来指定`Table`的主键。

+   `server_default` –

    一个`FetchedValue`实例、str、Unicode 或`text()`构造，表示列的 DDL DEFAULT 值。

    字符串类型将原样输出，用单引号括起来：

    ```py
    Column('x', Text, server_default="val")

    x TEXT DEFAULT 'val'
    ```

    `text()`表达式将原样呈现，不带引号：

    ```py
    Column('y', DateTime, server_default=text('NOW()'))

    y DATETIME DEFAULT NOW()
    ```

    字符串和 text()将在初始化时转换为`DefaultClause`对象。

    该参数还可以接受上下文中有效的 SQLAlchemy 表达式或构造的复杂组合：

    ```py
    from sqlalchemy import create_engine
    from sqlalchemy import Table, Column, MetaData, ARRAY, Text
    from sqlalchemy.dialects.postgresql import array

    engine = create_engine(
        'postgresql+psycopg2://scott:tiger@localhost/mydatabase'
    )
    metadata_obj = MetaData()
    tbl = Table(
            "foo",
            metadata_obj,
            Column("bar",
                   ARRAY(Text),
                   server_default=array(["biz", "bang", "bash"])
                   )
    )
    metadata_obj.create_all(engine)
    ```

    以上结果将创建一个带有以下 SQL 的表：

    ```py
    CREATE TABLE foo (
        bar TEXT[] DEFAULT ARRAY['biz', 'bang', 'bash']
    )
    ```

    使用`FetchedValue`表示数据库端已存在的列将生成一个默认值，在插入后可由 SQLAlchemy 进行后获取。这种构造不指定任何 DDL，实现留给数据库处理，比如通过触发器。

    请参见

    服务器调用的 DDL-显式默认表达式 - 服务器端默认值的完整讨论

+   `server_onupdate` –

    `FetchedValue`实例表示数据库端的默认生成函数，比如触发器。这表明对 SQLAlchemy 而言，新生成的值将在更新后可用。这种构造实际上并不在数据库内部实现任何生成函数，而是必须单独指定。

    警告

    此指令**当前不会**生成 MySQL 的“ON UPDATE CURRENT_TIMESTAMP()”子句。有关如何生成此子句的背景，请参阅为 MySQL / MariaDB 的 explicit_defaults_for_timestamp 渲染 ON UPDATE CURRENT TIMESTAMP。

    请参见

    标记隐式生成的值、时间戳和触发列

+   `quote` – 强制打开或关闭对此列名称的引用，对应`True`或`False`。当保持默认值`None`时，根据列标识符是否区分大小写（至少有一个大写字符的标识符被视为区分大小写），或者是否是保留字来引用列标识符。只有在需要强制引用 SQLAlchemy 方言未知的保留字时才需要此标志。

+   `unique` –

    当值为`True`时，且`Column.index`参数保持默认值`False`，表示将自动生成一个`UniqueConstraint`构造用于此`Column`，这将导致在调用`Table`对象的 DDL 创建操作时，在`CREATE TABLE`语句中包含引用此列的“UNIQUE CONSTRAINT”子句。

    当此标志为`True`而同时将`Column.index`参数设置为`True`时，效果实际上是生成一个包含`Index.unique`参数设置为`True`的`Index`构造。有关详细信息，请参阅`Column.index`的文档。

    使用此标志等效于在`Table`构造本身的级别上显式使用`UniqueConstraint`构造：

    ```py
    Table(
        "some_table",
        metadata,
        Column("x", Integer),
        UniqueConstraint("x")
    )
    ```

    唯一约束对象的`UniqueConstraint.name`参数保持其默认值为`None`；在缺少封闭的`MetaData`的 naming convention 时，唯一约束构造将被发出为未命名的，这通常会调用数据库特定的命名约定来发生。

    由于此标志仅用作在表定义中添加单列，默认配置的唯一约束的便利，因此在大多数用例中应优先使用`UniqueConstraint`构造的显式使用，包括涵盖多个列的复合约束、特定于后端的索引配置选项以及使用特定名称的约束。

    注意

    `Column`上的`Column.unique`属性**并不表示**此列是否具有唯一约束，只表示此标志是否在此处明确设置了。要查看涉及此列的索引和唯一约束，请查看`Table.indexes`和/或`Table.constraints`集合，或使用`Inspector.get_indexes()`和/或`Inspector.get_unique_constraints()`

    另请参阅

    唯一约束

    配置约束命名约定

    `Column.index`

+   `system` –

    当为`True`时，表示这是一个“系统”列，即数据库自动提供的列，并且不应包含在`CREATE TABLE`语句的列列表中。

    对于更复杂的场景，其中列应在不同的后端上以不同的条件呈现，请考虑`CreateColumn`的自定义编译规则。

+   `comment` –

    在表创建时可选的字符串，将在 SQL 注释中显示。

    新功能在版本 1.2 中：将`Column.comment`参数添加到`Column`。

+   `insert_sentinel` –

    将此`Column`标记为插入标记，用于优化对于没有其他限定主键配置的表的 insertmanyvalues 功能的性能。

    新功能在版本 2.0.10 中引入。

    另请参阅

    `insert_sentinel()` - 一站式助手，用于声明哨兵列

    “插入多个值”INSERT 语句行为

    配置 Sentinel 列

```py
method __le__(other: Any) → ColumnOperators
```

*从* `ColumnOperators` *的* `sqlalchemy.sql.expression.ColumnOperators.__le__` *方法继承*

实现`<=`运算符。

在列上下文中，生成子句`a <= b`。

```py
method __lt__(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators` *的* `sqlalchemy.sql.expression.ColumnOperators.__lt__` *方法。

实现`<`运算符。

在列上下文中，生成子句`a < b`。

```py
method __ne__(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators` *的* `sqlalchemy.sql.expression.ColumnOperators.__ne__` *方法。

实现`!=`运算符。

在列上下文中，生成子句`a != b`。如果目标是`None`，则生成`a IS NOT NULL`。

```py
method all_() → ColumnOperators
```

*继承自* `ColumnOperators` *的* `ColumnOperators.all_()` *方法。

生成针对父对象的`all_()`子句。

查看`all_()`的文档以获取示例。

注意

请确保不要混淆较新的`ColumnOperators.all_()`方法与此方法的**旧版**版本，即专用于`ARRAY`的`Comparator.all()`方法，其使用不同的调用风格。

```py
attribute anon_key_label
```

*继承自* `ColumnElement` *的* `ColumnElement.anon_key_label` *属性。

自版本 1.4 起弃用：`ColumnElement` *的* `ColumnElement.anon_key_label` *属性现在是私有的，公共访问器已弃用。

```py
attribute anon_label
```

*继承自* `ColumnElement` *的* `ColumnElement.anon_label` *属性。

自版本 1.4 起弃用：`ColumnElement` *的* `ColumnElement.anon_label` *属性现在是私有的，公共访问器已弃用。

```py
method any_() → ColumnOperators
```

*继承自* `ColumnOperators.any_()` *方法的* `ColumnOperators` *。

对父对象生成一个 `any_()` *子句*。

有关示例，请参阅 `any_()` *的文档*。

注意

请务必不要混淆较新的 `ColumnOperators.any_()` *方法与该方法的* **旧版**，`Comparator.any()` *，该方法专用于* `ARRAY` *，其调用方式不同。

```py
classmethod argument_for(dialect_name, argument_name, default)
```

*继承自* `DialectKWArgs.argument_for()` *方法的* `DialectKWArgs` *。

为这个类添加一种新的特定于方言的关键字参数。

例如：

```py
Index.argument_for("mydialect", "length", None)

some_index = Index('a', 'b', mydialect_length=5)
```

`DialectKWArgs.argument_for()` *方法是一种逐个参数的方式向* `DefaultDialect.construct_arguments` *字典添加额外参数。此字典为方言代表的各种架构级别构造提供了一组接受的参数名称。

新的方言通常应将此字典一次性指定为方言类的数据成员。通常，用于添加额外参数名称的即席用例是对自定义编译方案也使用额外参数的最终用户代码。

参数：

+   `dialect_name` – 方言的名称。方言必须是可定位的，否则会引发`NoSuchModuleError`。方言还必须包括一个现有的`DefaultDialect.construct_arguments`集合，表示它参与关键字参数验证和默认系统，否则会引发`ArgumentError`。如果方言不包括此集合，则已经可以代表该方言指定任何关键字参数。SQLAlchemy 内置的所有方言都包括此集合，但对于第三方方言，支持可能有所不同。

+   `argument_name` – 参数的名称。

+   `default` – 参数的默认值。

```py
method asc() → ColumnOperators
```

*继承自* `ColumnOperators.asc()` *方法的* `ColumnOperators`

根据父对象生成一个`asc()`子句。

```py
method between(cleft: Any, cright: Any, symmetric: bool = False) → ColumnOperators
```

*继承自* `ColumnOperators.between()` *方法的* `ColumnOperators`

根据下限和上限生成一个`between()`子句。

```py
method bitwise_and(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_and()` *方法的* `ColumnOperators`

执行位与操作，通常使用`&`运算符。

在 2.0.2 版本中新增。

另请参见

位运算符

```py
method bitwise_lshift(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_lshift()` *方法的* `ColumnOperators`

执行位左移操作，通常使用`<<`运算符。

在 2.0.2 版本中新增。

另请参见

位运算符

```py
method bitwise_not() → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_not()` *方法的* `ColumnOperators`

执行位非操作，通常通过`~`运算符实现。

版本 2.0.2 中新增。

另请参阅

位运算符

```py
method bitwise_or(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_or()` *方法的* `ColumnOperators`

执行位或操作，通常通过`|`运算符实现。

版本 2.0.2 中新增。

另请参阅

位运算符

```py
method bitwise_rshift(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_rshift()` *方法的* `ColumnOperators`

执行位右移操作，通常通过`>>`运算符实现。

版本 2.0.2 中新增。

另请参阅

位运算符

```py
method bitwise_xor(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_xor()` *方法的* `ColumnOperators`

执行位异或操作，通常通过`^`运算符实现，或者对于 PostgreSQL 使用`#`。

版本 2.0.2 中新增。

另请参阅

位运算符

```py
method bool_op(opstring: str, precedence: int = 0, python_impl: Callable[[...], Any] | None = None) → Callable[[Any], Operators]
```

*继承自* `Operators.bool_op()` *方法的* `Operators`

返回自定义布尔运算符。

此方法是调用 `Operators.op()` 并传递 `Operators.op.is_comparison` 标志为 True 的简写。 使用 `Operators.bool_op()` 的一个关键优势是，在使用列构造时，返回表达式的“布尔”性质将出现在 [**PEP 484**](https://peps.python.org/pep-0484/) 目的中。

另请参阅

`Operators.op()`

```py
method cast(type_: _TypeEngineArgument[_OPT]) → Cast[_OPT]
```

*继承自* `ColumnElement.cast()` *方法，来自于* `ColumnElement`

生成类型转换，即 `CAST(<expression> AS <type>)`。

这是一个快捷方式到 `cast()` 函数。

请参阅

数据转换和类型强制转换

`cast()`

`type_coerce()`

```py
method collate(collation: str) → ColumnOperators
```

*继承自* `ColumnOperators.collate()` *方法，来自于* `ColumnOperators`

生成一个针对父对象的 `collate()` 子句，给定排序规则字符串。

请参阅

`collate()`

```py
method compare(other: ClauseElement, **kw: Any) → bool
```

*继承自* `ClauseElement.compare()` *方法，来自于* `ClauseElement`

将此 `ClauseElement` 与给定的 `ClauseElement` 进行比较。

子类应该覆盖默认行为，即直接的标识比较。

**kw 是子类 `compare()` 方法消耗的参数，可能用于修改比较的条件（请参阅 `ColumnElement`）。

```py
method compile(bind: _HasDialect | None = None, dialect: Dialect | None = None, **kw: Any) → Compiled
```

*继承自* `CompilerElement.compile()` *方法，来自于* `CompilerElement`

编译这个 SQL 表达式。

返回值是一个 `Compiled` 对象。调用返回值的 `str()` 或 `unicode()` 方法将得到结果的字符串表示。`Compiled` 对象还可以使用 `params` 访问器返回一个绑定参数名称和值的字典。

参数:

+   `bind` – 一个`Connection`或`Engine`，它可以提供一个`Dialect`以生成一个`Compiled`对象。如果省略了`bind`和`dialect`参数，则使用默认的 SQL 编译器。

+   `column_keys` – 用于 INSERT 和 UPDATE 语句，列名列表，应出现在编译语句的 VALUES 子句中。如果为`None`，则从目标表对象中呈现所有列。

+   `dialect` – 一个`Dialect`实例，可以生成一个`Compiled`对象。此参数优先于`bind`参数。

+   `compile_kwargs` –

    可选的额外参数字典，将在所有“visit”方法中传递给编译器。这允许将任何自定义标志传递给自定义编译结构，例如。也用于通过`literal_binds`标志传递的情况：

    ```py
    from sqlalchemy.sql import table, column, select

    t = table('t', column('x'))

    s = select(t).where(t.c.x == 5)

    print(s.compile(compile_kwargs={"literal_binds": True}))
    ```

另请参阅

如何将 SQL 表达式渲染为字符串，可能带有内联的绑定参数？

```py
method concat(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.concat()` *方法的* `ColumnOperators`

实现‘concat’运算符。

在列上下文中，生成子句`a || b`，或在 MySQL 上使用`concat()`运算符。

```py
method contains(other: Any, **kw: Any) → ColumnOperators
```

*继承自* `ColumnOperators.contains()` *方法的* `ColumnOperators`

实现‘contains’运算符。

生成一个 LIKE 表达式，测试字符串值的中间匹配：

```py
column LIKE '%' || <other> || '%'
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.contains("foobar"))
```

由于操作符使用`LIKE`，存在于<other>表达式中的通配符字符`"%"`和`"_"`也将像通配符一样工作。对于字面字符串值，可以设置`ColumnOperators.contains.autoescape`标志为`True`，以对字符串值中这些字符的出现应用转义，使它们匹配为自身而不是通配符字符。或者，`ColumnOperators.contains.escape`参数将建立一个给定字符作为转义字符，当目标表达式不是字面字符串时，这可能会有所帮助。

参数：

+   `other` – 要比较的表达式。这通常是一个普通的字符串值，但也可以是任意的 SQL 表达式。除非设置了`ColumnOperators.contains.autoescape`标志为 True，否则 LIKE 通配符字符`%`和`_`不会被默认转义。

+   `autoescape` –

    布尔值；当为 True 时，在 LIKE 表达式中建立一个转义字符，然后将其应用于比较值中所有出现的`"%"`、`"_"`和转义字符本身，假定比较值是字面字符串而不是 SQL 表达式。

    例如：

    ```py
    somecolumn.contains("foo%bar", autoescape=True)
    ```

    将渲染为：

    ```py
    somecolumn LIKE '%' || :param || '%' ESCAPE '/'
    ```

    其值为`:param`时，将呈现为`"foo/%bar"`。

+   `escape` –

    一个字符，当给定时将使用`ESCAPE`关键字将其建立为转义字符。然后，可以将此字符放在`%`和`_`的出现之前，以允许它们作为自身而不是通配符字符。

    例如：

    ```py
    somecolumn.contains("foo/%bar", escape="^")
    ```

    将渲染为：

    ```py
    somecolumn LIKE '%' || :param || '%' ESCAPE '^'
    ```

    该参数还可以与`ColumnOperators.contains.autoescape`结合使用：

    ```py
    somecolumn.contains("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上述情况中，给定的字面参数将在传递到数据库之前转换为`"foo^%bar^^bat"`。

另请参阅

`ColumnOperators.startswith()`

`ColumnOperators.endswith()`

`ColumnOperators.like()`

```py
method copy(**kw: Any) → Column[Any]
```

自版本 1.4 起已弃用：`Column.copy()`方法已弃用，将在将来的版本中删除。

```py
method desc() → ColumnOperators
```

*继承自* `ColumnOperators.desc()` *方法的* `ColumnOperators`

对父对象生成 `desc()` 子句。

```py
attribute dialect_kwargs
```

*继承自* `DialectKWArgs.dialect_kwargs` *属性的* `DialectKWArgs`

作为方言特定选项指定的关键字参数的集合。

这里的参数以其原始的 `<dialect>_<kwarg>` 格式存在。只包括实际传递的参数；不同于 `DialectKWArgs.dialect_options` 集合，该集合包含了此方言已知的所有选项，包括默认值。

该集合也是可写的；接受形式为`<dialect>_<kwarg>`的键，其值将被组装到选项列表中。

另请参阅

`DialectKWArgs.dialect_options` - 嵌套字典形式

```py
attribute dialect_options
```

*继承自* `DialectKWArgs.dialect_options` *属性的* `DialectKWArgs`

作为方言特定选项指定的关键字参数的集合。

这是一个两级嵌套的注册表，键入为 `<dialect_name>` 和 `<argument_name>`。例如，`postgresql_where` 参数可以定位为：

```py
arg = my_object.dialect_options['postgresql']['where']
```

在版本 0.9.2 中新增。

另请参阅

`DialectKWArgs.dialect_kwargs` - 平面字典形式

```py
method distinct() → ColumnOperators
```

*继承自* `ColumnOperators.distinct()` *方法的* `ColumnOperators`

对父对象生成 `distinct()` 子句。

```py
method endswith(other: Any, escape: str | None = None, autoescape: bool = False) → ColumnOperators
```

*继承自* `ColumnOperators.endswith()` *方法的* `ColumnOperators`

实现 'endswith' 操作符。

产生一个 LIKE 表达式，用于测试字符串值的末尾是否匹配：

```py
column LIKE '%' || <other>
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.endswith("foobar"))
```

由于该操作符使用了 `LIKE`，因此存在于 <other> 表达式内部的通配符字符 `"%"` 和 `"_"` 也将像通配符一样行事。对于文本字符串值，可以将 `ColumnOperators.endswith.autoescape` 标志设置为 `True`，以对字符串值中这些字符的出现进行转义，使它们与自身匹配而不是作为通配符字符。或者，`ColumnOperators.endswith.escape` 参数将确立给定字符作为转义字符，当目标表达式不是文本字符串时可能会有用。

参数：

+   `other` – 要比较的表达式。通常是一个普通字符串值，但也可以是任意的 SQL 表达式。除非将 `ColumnOperators.endswith.autoescape` 标志设置为 True，否则不会默认转义 LIKE 通配符字符 `%` 和 `_`。

+   `autoescape` –

    布尔值；当为 True 时，在 LIKE 表达式中建立一个转义字符，然后将其应用于比较值中所有的 `"%"`、`"_"` 和转义字符本身，假设比较值是一个文本字符串而不是 SQL 表达式。

    诸如：

    ```py
    somecolumn.endswith("foo%bar", autoescape=True)
    ```

    渲染为：

    ```py
    somecolumn LIKE '%' || :param ESCAPE '/'
    ```

    以 `:param` 的值为 `"foo/%bar"`。

+   `escape` –

    一个字符，当提供时将以 `ESCAPE` 关键字呈现，以建立该字符作为转义字符。然后，此字符可以放置在 `%` 和 `_` 前面，使它们可以作为自己而不是通配符字符。

    诸如：

    ```py
    somecolumn.endswith("foo/%bar", escape="^")
    ```

    渲染为：

    ```py
    somecolumn LIKE '%' || :param ESCAPE '^'
    ```

    参数也可以与 `ColumnOperators.endswith.autoescape` 结合使用：

    ```py
    somecolumn.endswith("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上述示例中，给定的文本参数将在传递到数据库之前转换为 `"foo^%bar^^bat"`。

另请参见：

`ColumnOperators.startswith()`

`ColumnOperators.contains()`

`ColumnOperators.like()`

```py
attribute expression
```

*继承自* `ColumnElement.expression` *属性的* `ColumnElement`

返回列表达式。

检查接口的一部分；返回 self。

```py
attribute foreign_keys: Set[ForeignKey] = frozenset({})
```

*继承自* `ColumnElement.foreign_keys` *属性的* `ColumnElement`

与此 `Column` 关联的所有 `ForeignKey` 标记对象的集合。

每个对象都是 `Table` 范围内的 `ForeignKeyConstraint` 的成员。

另请参见

`Table.foreign_keys`

```py
method get_children(*, column_tables=False, **kw)
```

*继承自* `ColumnClause.get_children()` *方法的* `ColumnClause`

返回此 `HasTraverseInternals` 的即时子元素。

这用于访问遍历。

**kw 可能包含更改返回的集合的标志，例如返回子项的子集以减少更大的遍历，或者从不同的上下文（例如模式级别的集合而不是从子句级别）返回子项。

```py
method icontains(other: Any, **kw: Any) → ColumnOperators
```

*继承自* `ColumnOperators.icontains()` *方法的* `ColumnOperators`

实现 `icontains` 运算符，例如 `ColumnOperators.contains()` 的不区分大小写版本。

生成一个 LIKE 表达式，用于对字符串值的中间进行不区分大小写的匹配：

```py
lower(column) LIKE '%' || lower(<other>) || '%'
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.icontains("foobar"))
```

由于该操作符使用了`LIKE`，所以在<other>表达式内部存在的通配符字符`"%"`和`"_"`也将像通配符一样起作用。对于字面字符串值，可以设置`ColumnOperators.icontains.autoescape`标志为`True`，对字符串值内这些字符的出现应用转义，使它们匹配为它们自身而不是通配符字符。或者，`ColumnOperators.icontains.escape`参数将建立一个给定的字符作为转义字符，当目标表达式不是字面字符串时，这可能会有所帮助。

参数：

+   `other` – 要比较的表达式。这通常是一个普通字符串值，但也可以是任意的 SQL 表达式。除非将`ColumnOperators.icontains.autoescape`标志设置为`True`，否则`LIKE`通配符字符`%`和`_`默认不会转义。

+   `autoescape` –

    `boolean`；当为`True`时，在`LIKE`表达式中建立一个转义字符，然后将其应用于比较值内所有出现的`"%"`、`"_"`和转义字符本身，这里假定比较值是一个字面字符串而不是一个 SQL 表达式。

    一个类似如下的表达式：

    ```py
    somecolumn.icontains("foo%bar", autoescape=True)
    ```

    将渲染为：

    ```py
    lower(somecolumn) LIKE '%' || lower(:param) || '%' ESCAPE '/'
    ```

    以`:param`的值为`"foo/%bar"`。

+   `escape` –

    当给定一个字符时，将以`ESCAPE`关键字渲染该字符以将其建立为转义字符。然后，可以将该字符放在`%`和`_`的前面以允许它们作为它们自身而不是通配符字符。

    一个类似如下的表达式：

    ```py
    somecolumn.icontains("foo/%bar", escape="^")
    ```

    将渲染为：

    ```py
    lower(somecolumn) LIKE '%' || lower(:param) || '%' ESCAPE '^'
    ```

    该参数也可以与`ColumnOperators.contains.autoescape`结合使用：

    ```py
    somecolumn.icontains("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上述情况下，给定的字面参数将在传递到数据库之前转换为`"foo^%bar^^bat"`。

另见

`ColumnOperators.contains()`

```py
method iendswith(other: Any, escape: str | None = None, autoescape: bool = False) → ColumnOperators
```

*继承自* `ColumnOperators.iendswith()` *方法的* `ColumnOperators`

实现`iendswith`操作符，例如，`ColumnOperators.endswith()`的不区分大小写版本。

产生一个 LIKE 表达式，用于对字符串值的结尾进行不区分大小写匹配：

```py
lower(column) LIKE '%' || lower(<other>)
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.iendswith("foobar"))
```

由于运算符使用`LIKE`，所以存在于<other>表达式中的通配符字符`"%"`和`"_"`也会像通配符一样起作用。对于字面字符串值，可以将标志`ColumnOperators.iendswith.autoescape`设置为`True`，以对字符串值中这些字符的出现进行转义，使它们作为自身而不是通配符字符进行匹配。或者，参数`ColumnOperators.iendswith.escape`将建立一个给定的字符作为转义字符，当目标表达式不是字面字符串时可以派上用场。

参数：

+   `other` – 要进行比较的表达式。这通常是一个简单的字符串值，但也可以是任意的 SQL 表达式。除非将标志`ColumnOperators.iendswith.autoescape`设置为 True，否则不会转义 LIKE 通配符`%`和`_`。

+   `autoescape` –

    布尔值；当为 True 时，在 LIKE 表达式中建立一个转义字符，然后将其应用于比较值中的所有`"%"`、`"_"`和转义字符本身的出现，假设比较值是一个字面字符串而不是 SQL 表达式。

    诸如：

    ```py
    somecolumn.iendswith("foo%bar", autoescape=True)
    ```

    将呈现为：

    ```py
    lower(somecolumn) LIKE '%' || lower(:param) ESCAPE '/'
    ```

    其中，`:param`的值为`"foo/%bar"`。

+   `escape` –

    一个字符，当给定时，将使用`ESCAPE`关键字将该字符建立为转义字符。然后，该字符可以放在`%`和`_`之前，以允许它们作为自身而不是通配符字符。

    诸如：

    ```py
    somecolumn.iendswith("foo/%bar", escape="^")
    ```

    将呈现为：

    ```py
    lower(somecolumn) LIKE '%' || lower(:param) ESCAPE '^'
    ```

    参数也可以与`ColumnOperators.iendswith.autoescape`组合：

    ```py
    somecolumn.endswith("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上面的例子中，给定的字面参数将在传递到数据库之前被转换为`"foo^%bar^^bat"`。

另请参阅

`ColumnOperators.endswith()`

```py
method ilike(other: Any, escape: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.ilike()` *的方法* `ColumnOperators`

实现`ilike`运算符，例如不区分大小写的 LIKE。

在列上下文中，产生形式为：

```py
lower(a) LIKE lower(other)
```

或者在支持 ILIKE 运算符的后端上：

```py
a ILIKE other
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.ilike("%foobar%"))
```

参数：

+   `other` – 要比较的表达式

+   `escape` –

    可选的转义字符，渲染 `ESCAPE` 关键字，例如：

    ```py
    somecolumn.ilike("foo/%bar", escape="/")
    ```

参见

`ColumnOperators.like()`

```py
method in_(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.in_()` *方法的* `ColumnOperators`

实现 `in` 运算符。

在列上下文中，生成子句 `column IN <other>`。

给定的参数 `other` 可能是：

+   一个字面值列表，例如：

    ```py
    stmt.where(column.in_([1, 2, 3]))
    ```

    在这种调用形式中，项目列表转换为与给定列表长度相同的一组绑定参数：

    ```py
    WHERE COL IN (?, ?, ?)
    ```

+   如果比较是针对包含多个表达式的 `tuple_()`，则可以提供元组列表：

    ```py
    from sqlalchemy import tuple_
    stmt.where(tuple_(col1, col2).in_([(1, 10), (2, 20), (3, 30)]))
    ```

+   一个空列表，例如：

    ```py
    stmt.where(column.in_([]))
    ```

    在这种调用形式中，表达式渲染为“空集”表达式。这些表达式针对各个后端进行了定制，通常试图将空的 SELECT 语句作为子查询。例如在 SQLite 上，表达式是：

    ```py
    WHERE col IN (SELECT 1 FROM (SELECT 1) WHERE 1!=1)
    ```

    从版本 1.4 开始：在所有情况下，空的 IN 表达式现在使用执行时生成的 SELECT 子查询。

+   可以使用绑定参数，例如 `bindparam()`，如果它包含 `bindparam.expanding` 标志：

    ```py
    stmt.where(column.in_(bindparam('value', expanding=True)))
    ```

    在这种调用形式中，表达式渲染为一个特殊的非 SQL 占位符表达式，看起来像：

    ```py
    WHERE COL IN ([EXPANDING_value])
    ```

    这个占位符表达式在语句执行时被拦截，转换为前面所示的可变数量的绑定参数形式。如果执行语句如下：

    ```py
    connection.execute(stmt, {"value": [1, 2, 3]})
    ```

    数据库将为每个值传递一个绑定参数：

    ```py
    WHERE COL IN (?, ?, ?)
    ```

    新版本 1.2 中添加了“扩展”绑定参数

    如果传递了空列表，则渲染一个特殊的“空列表”表达式，该表达式特定于正在使用的数据库。在 SQLite 上，这将是：

    ```py
    WHERE COL IN (SELECT 1 FROM (SELECT 1) WHERE 1!=1)
    ```

    新版本 1.3 中：现在“扩展”绑定参数支持空列表

+   一个 `select()` 构造，通常是一个相关的标量选择：

    ```py
    stmt.where(
        column.in_(
            select(othertable.c.y).
            where(table.c.x == othertable.c.x)
        )
    )
    ```

    在这种调用形式中，`ColumnOperators.in_()` 渲染如下：

    ```py
    WHERE COL IN (SELECT othertable.y
    FROM othertable WHERE othertable.x = table.x)
    ```

参数：

**other** – 一个字面量列表，一个 `select()` 构造，或者一个包含 `bindparam.expanding` 标志设置为 True 的 `bindparam()` 构造。

```py
attribute index: bool | None
```

`Column.index` 参数的值。

不指示此 `Column` 是否实际上已经索引化；请使用 `Table.indexes`。

另请参阅

`Table.indexes`

```py
attribute info
```

*继承自* `SchemaItem.info` *属性的* `SchemaItem`。

与对象相关联的信息字典，允许将用户定义的数据与此 `SchemaItem` 关联。

字典在第一次访问时会自动生成。它还可以在某些对象的构造函数中指定，例如 `Table` 和 `Column`。

```py
attribute inherit_cache: bool | None = True
```

指示此 `HasCacheKey` 实例是否应该使用其直接超类使用的缓存键生成方案。

此属性默认为 `None`，表示构造尚未考虑其是否适合参与缓存；这在功能上等同于将值设置为 `False`，除了还会发出警告。

如果与此类局部而非其超类相关的属性不会更改与对象对应的 SQL，则可以在特定类上将此标志设置为 `True`。

另请参阅

为自定义构造启用缓存支持 - 有关设置特定类的 `HasCacheKey.inherit_cache` 属性以供第三方或用户定义的 SQL 构造使用的一般指南。

```py
method is_(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.is_()` *方法的* `ColumnOperators`。

实现 `IS` 操作符。

通常情况下，与`None`值比较时会自动生成`IS`，其解析为`NULL`。然而，在某些平台上，如果要与布尔值进行比较，则可能需要显式使用`IS`。

另请参阅

`ColumnOperators.is_not()`

```py
method is_distinct_from(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.is_distinct_from()` *方法的* `ColumnOperators`

实现`IS DISTINCT FROM`操作符。

在大多数平台上呈现为“a IS DISTINCT FROM b”；在某些平台上，如 SQLite 可能呈现为“a IS NOT b”。

```py
method is_not(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.is_not()` *方法的* `ColumnOperators`

实现`IS NOT`操作符。

通常情况下，与`None`值比较时会自动生成`IS NOT`，其解析为`NULL`。然而，在某些平台上，如果要与布尔值进行比较，则可能需要显式使用`IS NOT`。

1.4 版本更改：`is_not()`操作符从先前版本的`isnot()`重新命名。以前的名称仍然可用于向后兼容。

另请参阅

`ColumnOperators.is_()`

```py
method is_not_distinct_from(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.is_not_distinct_from()` *方法的* `ColumnOperators`

实现`IS NOT DISTINCT FROM`操作符。

在大多数平台上呈现为“a IS NOT DISTINCT FROM b”；在某些平台上，如 SQLite 可能呈现为“a IS b”。

1.4 版本更改：`is_not_distinct_from()`操作符从先前版本的`isnot_distinct_from()`重新命名。以前的名称仍然可用于向后兼容。

```py
method isnot(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.isnot()` *方法的* `ColumnOperators`

实现`IS NOT`操作符。

通常情况下，与`None`值比较时会自动生成`IS NOT`，其解析为`NULL`。然而，在某些平台上，如果要与布尔值进行比较，则可能需要显式使用`IS NOT`。

从版本 1.4 开始更改：`is_not()` 操作符在之前的版本中从`isnot()`重命名。以确保向后兼容性仍可使用之前的名称。

另请参阅

`ColumnOperators.is_()`

```py
method isnot_distinct_from(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.isnot_distinct_from()` *的方法* `ColumnOperators`

实现 `IS NOT DISTINCT FROM` 操作符。

在大多数平台上呈现“a IS NOT DISTINCT FROM b”；在某些平台上，例如 SQLite 可能会呈现“a IS b”。

从版本 1.4 开始更改：`is_not_distinct_from()` 操作符在之前的版本中从 `isnot_distinct_from()` 重命名。以确保向后兼容性仍可使用之前的名称。

```py
method istartswith(other: Any, escape: str | None = None, autoescape: bool = False) → ColumnOperators
```

*继承自* `ColumnOperators.istartswith()` *的方法* `ColumnOperators`

实现 `istartswith` 操作符，例如 `ColumnOperators.startswith()` 的不区分大小写版本。

生成一个 LIKE 表达式，用于对字符串值的开头进行不区分大小写的匹配：

```py
lower(column) LIKE lower(<other>) || '%'
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.istartswith("foobar"))
```

由于操作符使用`LIKE`，存在于<other>表达式内部的通配符字符`"%"`和`"_"`也会像通配符一样起作用。对于文字字符串值，可以将`ColumnOperators.istartswith.autoescape`标志设置为`True`，以对字符串值内的这些字符应用转义，使它们匹配为自己而不是通配符字符。或者，`ColumnOperators.istartswith.escape`参数将建立一个给定的字符作为转义字符，当目标表达式不是文字字符串时可以派上用场。

参数：

+   `other` – 要比较的表达式。通常这是一个纯文本字符串值，但也可以是任意的 SQL 表达式。LIKE 通配符字符 `%` 和 `_` 默认情况下不会被转义，除非设置了`ColumnOperators.istartswith.autoescape`标志为 True。

+   `autoescape` –

    布尔值；当为 True 时，在 LIKE 表达式中建立一个转义字符，然后将其应用于比较值中的所有`"%"`、`"_"`和转义字符本身的出现，假定比较值为文字字符串而不是 SQL 表达式。

    例如：

    ```py
    somecolumn.istartswith("foo%bar", autoescape=True)
    ```

    将呈现为：

    ```py
    lower(somecolumn) LIKE lower(:param) || '%' ESCAPE '/'
    ```

    具有值`:param`为`"foo/%bar"`。

+   `escape` –

    一个字符，当给定时，将使用`ESCAPE`关键字将其呈现为转义字符。然后可以将此字符放在`%`和`_`之前，以使它们作为自身而不是通配符字符。

    例如：

    ```py
    somecolumn.istartswith("foo/%bar", escape="^")
    ```

    将呈现为：

    ```py
    lower(somecolumn) LIKE lower(:param) || '%' ESCAPE '^'
    ```

    参数也可以与`ColumnOperators.istartswith.autoescape`结合使用：

    ```py
    somecolumn.istartswith("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上述情况下，给定的文字参数将在传递到数据库之前转换为`"foo^%bar^^bat"`。

另请参见

`ColumnOperators.startswith()`

```py
attribute key: str = None
```

*继承自* `ColumnElement.key` *属性的* `ColumnElement`

在某些情况下指代此对象在 Python 命名空间中的‘key’。

这通常指的是在可选择的`.c`集合中表示的列的“键”，例如`somekey`的`somekey`将返回一个`Column`，其`.key`为“somekey”。

```py
attribute kwargs
```

*继承自* `DialectKWArgs.kwargs` *属性的* `DialectKWArgs`

`DialectKWArgs.dialect_kwargs`的同义词。

```py
method label(name: str | None) → Label[_T]
```

*继承自* `ColumnElement.label()` *方法的* `ColumnElement`

生成一个列标签，即`<columnname> AS <name>`。

这是`label()`函数的快捷方式。

如果‘name’为`None`，将生成一个匿名标签名称。

```py
method like(other: Any, escape: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.like()` *的* `ColumnOperators` *方法*

实现 `like` 运算符。

在列上下文中，生成表达式：

```py
a LIKE other
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.like("%foobar%"))
```

参数：

+   `other` – 要比较的表达式

+   `escape` –

    可选的转义字符，呈现`ESCAPE`关键字，例如：

    ```py
    somecolumn.like("foo/%bar", escape="/")
    ```

另请参阅

`ColumnOperators.ilike()`

```py
method match(other: Any, **kwargs: Any) → ColumnOperators
```

*继承自* `ColumnOperators.match()` *的* `ColumnOperators`

实现特定于数据库的 'match' 运算符。

`ColumnOperators.match()` 尝试解析由后端提供的类似 MATCH 的函数或运算符。示例包括：

+   PostgreSQL - 呈现 `x @@ plainto_tsquery(y)`

    > 版本 2.0 中的更改：现在对于 PostgreSQL，使用 `plainto_tsquery()` 而不是 `to_tsquery()`；为了与其他形式兼容，请参阅 全文搜索。

+   MySQL - 呈现 `MATCH (x) AGAINST (y IN BOOLEAN MODE)`

    另请参阅

    `match` - 具有附加功能的 MySQL 特定构造。

+   Oracle - 呈现 `CONTAINS(x, y)`

+   其他后端可能提供特殊实现。

+   没有任何特殊实现的后端将发出操作符为“MATCH”。例如，这与 SQLite 兼容。

```py
method not_ilike(other: Any, escape: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.not_ilike()` *的* `ColumnOperators`

实现 `NOT ILIKE` 运算符。

这相当于使用 `ColumnOperators.ilike()` 的否定，即 `~x.ilike(y)`。

版本 1.4 中的更改：`not_ilike()` 运算符从以前的版本中的 `notilike()` 重命名。以前的名称仍可用于向后兼容。

另请参阅

`ColumnOperators.ilike()`

```py
method not_in(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.not_in()` *的方法* `ColumnOperators`

实现 `NOT IN` 运算符。

这相当于对 `ColumnOperators.in_()` 进行取反操作，即 `~x.in_(y)`。

如果 `other` 是一个空序列，则编译器会生成一个“空的不在”表达式。 默认情况下，这会转换为表达式 “1 = 1” 以在所有情况下产生 true。 可以使用 `create_engine.empty_in_strategy` 来更改此行为。

在 1.4 版本中有所变动：`not_in()` 运算符从先前的 `notin_()` 改名为 `not_in()`。 先前的名称仍可用于向后兼容。

在 1.2 版本中有所变动：`ColumnOperators.in_()` 和 `ColumnOperators.not_in()` 运算符现在默认生成一个空的 IN 序列的“静态”表达式。

另请参阅

`ColumnOperators.in_()`

```py
method not_like(other: Any, escape: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.not_like()` *的方法* `ColumnOperators`

实现 `NOT LIKE` 运算符。

这相当于对 `ColumnOperators.like()` 进行取反操作，即 `~x.like(y)`。

在 1.4 版本中有所变动：`not_like()` 运算符从先前的 `notlike()` 改名为 `not_like()`。 先前的名称仍可用于向后兼容。

另请参阅

`ColumnOperators.like()`

```py
method notilike(other: Any, escape: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.notilike()` *的方法* `ColumnOperators`

实现 `NOT ILIKE` 运算符。

这相当于对 `ColumnOperators.ilike()` 进行取反操作，即 `~x.ilike(y)`。

在 1.4 版本中更改：`not_ilike()`运算符在之前的版本中从`notilike()`重命名。以前的名称仍然可用于向后兼容。

参见

`ColumnOperators.ilike()`

```py
method notin_(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.notin_()` *方法的* `ColumnOperators`

实现`NOT IN`运算符。

这相当于对 `ColumnOperators.in_()` 使用否定，即 `~x.in_(y)`。

如果`other`是空序列，则编译器生成一个“空的不在其中”表达式。默认情况下，这默认为表达式“1 = 1”，在所有情况下都产生 true。`create_engine.empty_in_strategy` 可用于更改此行为。

在 1.4 版本中更改：`not_in()`运算符在之前的版本中从`notin_()`重命名。以前的名称仍然可用于向后兼容。

在 1.2 版本中更改：`ColumnOperators.in_()` 和 `ColumnOperators.not_in()` 运算符现在默认情况下生成一个“静态”表达式以表示空的 IN 序列。

参见

`ColumnOperators.in_()`

```py
method notlike(other: Any, escape: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.notlike()` *方法的* `ColumnOperators`

实现`NOT LIKE`运算符。

这相当于对 `ColumnOperators.like()` 使用否定，即 `~x.like(y)`。

在 1.4 版本中更改：`not_like()`运算符在之前的版本中从`notlike()`重命名。以前的名称仍然可用于向后兼容。

参见

`ColumnOperators.like()`

```py
method nulls_first() → ColumnOperators
```

*继承自* `ColumnOperators.nulls_first()` *方法的* `ColumnOperators`

对父对象生成一个`nulls_first()`子句。

在 1.4 版本中更改：`nulls_first()`运算符在以前的版本中从`nullsfirst()`改名。 以前的名称仍然可用于向后兼容。

```py
method nulls_last() → ColumnOperators
```

*继承自* `ColumnOperators.nulls_last()` *方法的* `ColumnOperators`

对父对象生成一个`nulls_last()`子句。

在 1.4 版本中更改：`nulls_last()`运算符在以前的版本中从`nullslast()`改名。 以前的名称仍然可用于向后兼容。

```py
method nullsfirst() → ColumnOperators
```

*继承自* `ColumnOperators.nullsfirst()` *方法的* `ColumnOperators`

对父对象生成一个`nulls_first()`子句。

在 1.4 版本中更改：`nulls_first()`运算符在以前的版本中从`nullsfirst()`改名。 以前的名称仍然可用于向后兼容。

```py
method nullslast() → ColumnOperators
```

*继承自* `ColumnOperators.nullslast()` *方法的* `ColumnOperators`

对父对象生成一个`nulls_last()`子句。

在 1.4 版本中更改：`nulls_last()`运算符在以前的版本中从`nullslast()`改名。 以前的名称仍然可用于向后兼容。

```py
method op(opstring: str, precedence: int = 0, is_comparison: bool = False, return_type: Type[TypeEngine[Any]] | TypeEngine[Any] | None = None, python_impl: Callable[..., Any] | None = None) → Callable[[Any], Operators]
```

*继承自* `Operators.op()` *方法的* `Operators`

生成一个通用操作函数。

例如：

```py
somecolumn.op("*")(5)
```

产生：

```py
somecolumn * 5
```

此函数还可用于使按位运算符明确。 例如：

```py
somecolumn.op('&')(0xff)
```

是`somecolumn`中值的按位与。

参数：

+   `opstring` – 将作为此元素与传递给生成函数的表达式之间的中缀操作符输出的字符串。

+   `precedence` –

    数据库预期应用于 SQL 表达式中操作符的优先级。这个整数值作为 SQL 编译器的提示，用于确定何时应该在特定操作周围呈现显式括号。较低的数字将导致在与具有较高优先级的其他操作符应用时对表达式进行括号化。默认值`0`低于所有操作符，除了逗号（`,`）和`AS`操作符。值为 100 时，将高于或等于所有操作符，而-100 将低于或等于所有操作符。

    另请参阅

    我正在使用 op() 生成自定义操作符，但我的括号不正确 - SQLAlchemy SQL 编译器呈现括号的详细描述

+   `is_comparison` –

    传统；如果为 True，则将该操作符视为“比较”运算符，即评估为布尔值的运算符，如 `==`、`>` 等。提供此标志是为了使 ORM 关系可以在自定义连接条件中使用操作符时，建立该操作符是比较运算符的关系。

    使用`is_comparison`参数已被使用`Operators.bool_op()`方法取代；此更简洁的运算符会自动设置此参数，但也提供了正确的[**PEP 484**](https://peps.python.org/pep-0484/)类型支持，因为返回的对象将表示“布尔”数据类型，即 `BinaryExpression[bool]`。

+   `return_type` – 一个`TypeEngine`类或对象，将强制由此操作符产生的表达式的返回类型为该类型。默认情况下，指定`Operators.op.is_comparison`的运算符将解析为`Boolean`，而未指定的将与左操作数的类型相同。

+   `python_impl` –

    可选的 Python 函数，可以以与在数据库服务器上运行此操作符时相同的方式评估两个 Python 值。对于在 Python 中的 SQL 表达式评估函数非常有用，例如用于 ORM 混合属性的函数，以及用于匹配多行更新或删除后会话中的对象的 ORM“评估器”。

    例如：

    ```py
    >>> expr = column('x').op('+', python_impl=lambda a, b: a + b)('y')
    ```

    上述表达式的运算符也将适用于非 SQL 左和右对象：

    ```py
    >>> expr.operator(5, 10)
    15
    ```

    2.0 版本中的新功能。

另请参阅

`Operators.bool_op()`

重新定义和创建新的操作符

在连接条件中使用自定义操作符

```py
method operate(op: OperatorType, *other: Any, **kwargs: Any) → ColumnElement[Any]
```

*继承自* `ColumnElement.operate()` *方法的* `ColumnElement`

对参数进行操作。

这是操作的最低级别，默认情况下引发 `NotImplementedError`。

在子类上覆盖这个方法可以使常见行为应用于所有操作。例如，覆盖 `ColumnOperators` 以将 `func.lower()` 应用于左右两侧：

```py
class MyComparator(ColumnOperators):
    def operate(self, op, other, **kwargs):
        return op(func.lower(self), func.lower(other), **kwargs)
```

参数：

+   `op` – 操作符可调用。

+   `*other` – 操作的‘另一’侧。对于大多数操作，将是单个标量。

+   `**kwargs` – 修饰符。这些可能由特殊操作符（如 `ColumnOperators.contains()`）传递。

```py
method params(*optionaldict, **kwargs)
```

*继承自* `Immutable.params()` *方法的* `Immutable`

返回一个替换了 `bindparam()` 元素的副本。

返回此 ClauseElement 的副本，其中的 `bindparam()` 元素替换为给定字典中的值：

```py
>>> clause = column('x') + bindparam('foo')
>>> print(clause.compile().params)
{'foo':None}
>>> print(clause.params({'foo':7}).compile().params)
{'foo':7}
```

```py
attribute proxy_set: util.generic_fn_descriptor[FrozenSet[Any]]
```

*继承自* `ColumnElement.proxy_set` *属性的* `ColumnElement`

我们正在代理的所有列的集合

从 2.0 版本开始，这些列明确地被取消注释。以前它们实际上是取消注释的列，但没有强制执行。如果可能的话，注释列基本上不应该进入集合，因为它们的哈希行为非常低效。

```py
method references(column: Column[Any]) → bool
```

如果此列通过外键引用给定列，则返回 True。

```py
method regexp_match(pattern: Any, flags: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.regexp_match()` *方法的* `ColumnOperators`

实现特定于数据库的‘regexp match’运算符。

例如：

```py
stmt = select(table.c.some_column).where(
    table.c.some_column.regexp_match('^(b|c)')
)
```

`ColumnOperators.regexp_match()` 尝试解析为后端提供的类似 REGEXP 的函数或操作符，但特定的正则表达式语法和可用标志**不是后端通用的**。

示例包括：

+   PostgreSQL - 在否定时渲染 `x ~ y` 或 `x !~ y`。

+   Oracle - 渲染 `REGEXP_LIKE(x, y)`

+   SQLite - 使用 SQLite 的 `REGEXP` 占位符操作符，并调用 Python 的 `re.match()` 内置函数。

+   其他后端可能提供特殊的实现。

+   没有任何特殊实现的后端将生成操作符“REGEXP”或“NOT REGEXP”。例如，这与 SQLite 和 MySQL 兼容。

目前为止，Oracle、PostgreSQL、MySQL 和 MariaDB 实现了正则表达式支持。SQLite 提供了部分支持。第三方方言的支持可能有所不同。

参数：

+   `pattern` – 正则表达式模式字符串或列子句。

+   `flags` – 要应用的任何正则表达式字符串标志，仅作为纯 Python 字符串传递。这些标志是后端特定的。某些后端，例如 PostgreSQL 和 MariaDB，可能会将标志作为模式的一部分指定。在 PostgreSQL 中使用忽略大小写标志 'i' 时，将使用忽略大小写正则表达式匹配运算符 `~*` 或 `!~*`。

新版本 1.4 中新增。

在版本 1.4.48 中更改，: 2.0.18 请注意，由于实现错误，先前的“flags”参数接受了 SQL 表达式对象，例如列表达式，而不仅仅是纯 Python 字符串。这个实现与缓存一起使用时不会正常工作，并且已被删除；只应传递字符串给“flags”参数，因为这些标志会作为 SQL 表达式中的文字内联值呈现。

另请参阅

`ColumnOperators.regexp_replace()`

```py
method regexp_replace(pattern: Any, replacement: Any, flags: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.regexp_replace()` *方法的* `ColumnOperators`

实现了特定于数据库的“regexp replace”运算符。

例如：

```py
stmt = select(
    table.c.some_column.regexp_replace(
        'b(..)',
        'XY',
        flags='g'
    )
)
```

`ColumnOperators.regexp_replace()` 尝试解析后端提供的类似 REGEXP_REPLACE 的函数，通常会生成函数 `REGEXP_REPLACE()`。然而，特定的正则表达式语法和可用标志**不是跨后端通用的**。

目前仅为 Oracle、PostgreSQL、MySQL 8 或更高版本和 MariaDB 实现了正则表达式替换支持。第三方方言的支持可能有所不同。

参数：

+   `pattern` – 正则表达式模式字符串或列子句。

+   `pattern` – 替换字符串或列子句。

+   `flags` – 要应用的任何正则表达式字符串标志，仅作为纯 Python 字符串传递。这些标志是后端特定的。某些后端，例如 PostgreSQL 和 MariaDB，可能会将标志作为模式的一部分指定。

新版本 1.4 中新增。

在版本 1.4.48 更改为：2.0.18 请注意，由于实现错误，以前“flags”参数接受 SQL 表达式对象，例如列表达式，而不仅仅是普通的 Python 字符串。 这种实现与缓存不正确，因此已删除；应该仅传递字符串作为“flags”参数，因为这些标志将作为 SQL 表达式中的文字内联值呈现。

另请参阅

`ColumnOperators.regexp_match()`

```py
method reverse_operate(op: OperatorType, other: Any, **kwargs: Any) → ColumnElement[Any]
```

*继承自* `ColumnElement.reverse_operate()` *方法的* `ColumnElement`

对参数进行反向操作。

使用方式与`operate()`相同。

```py
method self_group(against: OperatorType | None = None) → ColumnElement[Any]
```

*继承自* `ColumnElement.self_group()` *方法的* `ColumnElement`

对这个`ClauseElement`应用一个“分组”。

子类重写此方法以返回一个“分组”构造，即括号。特别是当“二进制”表达式被放置到更大的表达式中时，它们会提供一个围绕自身的分组，以及当`select()`构造被放置到另一个`select()`的 FROM 子句中时。 （请注意，子查询通常应使用`Select.alias()`方法创建，因为许多平台要求嵌套的 SELECT 语句必须被命名）。

当表达式组合在一起时，`self_group()`的应用是自动的 - 最终用户代码不应直接使用此方法。 请注意，SQLAlchemy 的子句构造会考虑运算符优先级 - ���此在表达式中可能不需要括号，例如，`x OR (y AND z)` - AND 优先于 OR。

`ClauseElement`的基本`self_group()`方法只返回自身。

```py
method shares_lineage(othercolumn: ColumnElement[Any]) → bool
```

*继承自* `ColumnElement.shares_lineage()` *方法的* `ColumnElement`

如果给定的`ColumnElement`与此`ColumnElement`具有共同的祖先，则返回 True。

```py
method startswith(other: Any, escape: str | None = None, autoescape: bool = False) → ColumnOperators
```

*继承自* `ColumnOperators.startswith()` *方法的* `ColumnOperators`

实现`startswith`运算符。

生成一个 LIKE 表达式，用于测试字符串值的开头是否匹配：

```py
column LIKE <other> || '%'
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.startswith("foobar"))
```

由于该运算符使用`LIKE`，存在于<other>表达式中的通配符字符`%`和`_`也将像通配符一样起作用。对于字面字符串值，可以将`ColumnOperators.startswith.autoescape`标志设置为`True`，以对字符串值中这些字符的出现应用转义，使它们匹配为自身而不是通配符字符。或者，`ColumnOperators.startswith.escape`参数将建立一个给定字符作为转义字符，当目标表达式不是字面字符串时可能会有用。

参数：

+   `other` – 要比较的表达式。通常是一个普通字符串值，但也可以是任意的 SQL 表达式。除非设置`ColumnOperators.startswith.autoescape`标志为 True，否则 LIKE 通配符字符`%`和`_`默认不会被转义。

+   `autoescape` –

    布尔值；当为 True 时，在 LIKE 表达式中建立一个转义字符，然后将其应用于比较值中的所有出现的`"%"`、`"_"`和转义字符本身，假定比较值是一个字面字符串而不是 SQL 表达式。

    一个表达式如下：

    ```py
    somecolumn.startswith("foo%bar", autoescape=True)
    ```

    将呈现为：

    ```py
    somecolumn LIKE :param || '%' ESCAPE '/'
    ```

    具有值`:param`为`"foo/%bar"`。

+   `escape` –

    一个字符，当给定时将使用`ESCAPE`关键字来建立该字符作为转义字符。然后可以将该字符放在`%`和`_`之前，以允许它们作为自身而不是通配符字符。

    一个表达式如下：

    ```py
    somecolumn.startswith("foo/%bar", escape="^")
    ```

    将呈现为：

    ```py
    somecolumn LIKE :param || '%' ESCAPE '^'
    ```

    参数也可以与`ColumnOperators.startswith.autoescape`结合使用：

    ```py
    somecolumn.startswith("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上述情况下，给定的文字参数将在传递到数据库之前转换为`"foo^%bar^^bat"`。

另请参见

`ColumnOperators.endswith()`

`ColumnOperators.contains()`

`ColumnOperators.like()`

```py
attribute timetuple: Literal[None] = None
```

*继承自* `ColumnOperators` *的* `ColumnOperators.timetuple` *属性*

Hack，允许在左侧比较 datetime 对象。

```py
attribute unique: bool | None
```

`Column.unique`参数的值。

不指示此`Column`是否实际上受到唯一约束的影响；请使用`Table.indexes`和`Table.constraints`。

另请参见

`Table.indexes`

`Table.constraints`。

```py
method unique_params(*optionaldict, **kwargs)
```

*继承自* `Immutable` *的* `Immutable.unique_params()` *方法*

返回一个副本，其中`bindparam()`元素被替换。

与`ClauseElement.params()`具有相同功能，只是对受影响的绑定参数添加了 unique=True，以便可以使用多个语句。

```py
class sqlalchemy.schema.MetaData
```

一组`Table`对象及其关联的模式构造。

包含一组`Table`对象以及一个可选的绑定到`Engine`或`Connection`的集合。如果绑定，集合中的`Table`对象及其列可以参与隐式 SQL 执行。

`Table` 对象本身存储在`MetaData.tables` 字典中。

`MetaData` 是一个线程安全的对象，用于读取操作。在单个`MetaData` 对象内构建新表，无论是显式地还是通过反射，可能并不完全线程安全。

另请参阅

使用 MetaData 描述数据库 - 介绍数据库元数据

**成员**

__init__(), clear(), create_all(), drop_all(), reflect(), remove(), sorted_tables, tables

**类签名**

类`sqlalchemy.schema.MetaData` (`sqlalchemy.schema.HasSchemaAttr`)

```py
method __init__(schema: str | None = None, quote_schema: bool | None = None, naming_convention: _NamingSchemaParameter | None = None, info: _InfoType | None = None) → None
```

创建一个新的 MetaData 对象。

参数：

+   `schema` –

    默认要用于`Table`、`Sequence`和可能与此`MetaData`关联的其他对象的模式。默认为 `None`。

    另请参阅

    使用 MetaData 指定默认模式名称 - 有关如何使用`MetaData.schema`参数的详细信息。

    `Table.schema`

    `Sequence.schema`

+   `quote_schema` – 为那些使用本地 `schema` 名称的`Table`、`Sequence`和其他对象设置 `quote_schema` 标志。

+   `info` – 可选数据字典，将填充到此对象的`SchemaItem.info`属性中。

+   `naming_convention` –

    一个字典，指向将为未明确命名的`Constraint`和`Index`对象建立默认命名约定的值。

    此字典的键可能是：

    +   约束或索引类，例如`UniqueConstraint`类、`ForeignKeyConstraint`类、`Index`类

    +   已知约束类别之一的字符串助记符；分别为外键（"fk"）、主键（"pk"）、索引（"ix"）、检查（"ck"）和唯一约束（"uq"）。

    +   用户定义的“token”的字符串名称，可用于定义新的命名标记。

    与每个“约束类”或“约束助记符”键关联的值是命名模板字符串，例如`"uq_%(table_name)s_%(column_0_name)s"`，描述了名称应该如何组成。与用户定义的“token”键关联的值应该是形式为`fn(constraint, table)`的可调用对象，接受约束/索引对象和`Table`作为参数，返回一个字符串结果。

    内置名称如下，其中一些可能仅适用于某些类型的约束：

    > +   `%(table_name)s` - 与约束相关联的`Table`对象的名称。
    > +   
    > +   `%(referred_table_name)s` - 与`ForeignKeyConstraint`的引用目标相关联的`Table`对象的名称。
    > +   
    > +   `%(column_0_name)s` - 约束内索引位置“0”处的`Column`的名称。
    > +   
    > +   `%(column_0N_name)s` - 约束内所有`Column`对象的名称，按顺序不使用分隔符连接。
    > +   
    > +   `%(column_0_N_name)s` - 约束内所有`Column`对象的名称，按顺序用下划线分隔。
    > +   
    > +   `%(column_0_label)s`、`%(column_0N_label)s`、`%(column_0_N_label)s` - 零号`Column`或所有`Columns`的标签，用下划线分隔或不分隔。
    > +   
    > +   `%(column_0_key)s`、`%(column_0N_key)s`、`%(column_0_N_key)s` - 零号`Column`或所有`Columns`的键，用下划线分隔或不分隔。
    > +   
    > +   `%(referred_column_0_name)s`、`%(referred_column_0N_name)s`、`%(referred_column_0_N_name)s`、`%(referred_column_0_key)s`、`%(referred_column_0N_key)s`、…渲染由`ForeignKeyConstraint`引用的列的名称/键/标签的列标记。
    > +   
    > +   `%(constraint_name)s` - 一个特殊键，指代约束已存在的名称。当存在此键时，`Constraint` 对象的现有名称将被替换为使用此标记的模板字符串组成的名称。当存在此标记时，要求在此之前明确给出 `Constraint` 的名称。
    > +   
    > +   用户定义：通过将其与 `fn(constraint, table)` 可调用对象一起传递给 naming_convention 字典，可以实现任何额外的标记。

    新版本 1.3.0 中新增：- 添加了新的 `%(column_0N_name)s`、`%(column_0_N_name)s` 等相关标记，用于生成由给定约束引用的所有列的名称、键或标签的连接。

    另请参阅

    配置约束命名约定 - 详细的使用示例。

```py
method clear() → None
```

从此 MetaData 中清除所有 Table 对象。

```py
method create_all(bind: _CreateDropBind, tables: _typing_Sequence[Table] | None = None, checkfirst: bool = True) → None
```

创建存储在此元数据中的所有表。

默认为条件性操作，不会尝试重新创建已经存在于目标数据库中的表。

参数：

+   `bind` – 用于访问数据库的 `Connection` 或 `Engine`。 

+   `tables` – 可选的 `Table` 对象列表，是 `MetaData` 中总表的子集（其他表将被忽略）。

+   `checkfirst` – 默认为 True，不会为已经存在于目标数据库中的表发出 CREATE 语句。

```py
method drop_all(bind: _CreateDropBind, tables: _typing_Sequence[Table] | None = None, checkfirst: bool = True) → None
```

删除存储在此元数据中的所有表。

默认为条件性操作，不会尝试删除目标数据库中不存在的表。

参数：

+   `bind` – 用于访问数据库的 `Connection` 或 `Engine`。

+   `tables` – 可选的 `Table` 对象列表，是 `MetaData` 中总表的子集（其他表将被忽略）。

+   `checkfirst` – 默认为 True，仅对确认存在于目标数据库中的表发出 DROP 语句。

```py
method reflect(bind: Engine | Connection, schema: str | None = None, views: bool = False, only: _typing_Sequence[str] | Callable[[str, MetaData], bool] | None = None, extend_existing: bool = False, autoload_replace: bool = True, resolve_fks: bool = True, **dialect_kwargs: Any) → None
```

从数据库加载所有可用的表定义。

自动在此 `MetaData` 中为数据库中任何尚未存在于 `MetaData` 中的表创建 `Table` 条目。可以多次调用以获取最近添加到数据库中的表，但是如果 `MetaData` 中的表在数据库中不再存在，则不会采取任何特殊操作。

参数：

+   `bind` – 用于访问数据库的 `Connection` 或 `Engine`。

+   `schema` – 可选，从替代模式查询和反映表。如果为 None，则使用与此`MetaData`关联的模式（如果有）。

+   `views` – 如果为 True，则还反映视图（物化和普通）。

+   `only` –

    可选。仅加载可用命名表的子集。可以指定为名称序列或可调用对象。

    如果提供了一系列名称，则只会反映这些表。如果请求了一个表但该表不存在，则会引发错误。已经存在于此`MetaData`中的命名表将被忽略。

    如果提供了可调用对象，则将其用作布尔谓词，以过滤潜在表名称列表。可调用对象将以表名称和此`MetaData`实例作为位置参数调用，并应为任何要反映的表返回一个真值。

+   `extend_existing` – 传递给每个`Table`作为`Table.extend_existing`。

+   `autoload_replace` – 传递给每个`Table`作为`Table.autoload_replace`。

+   `resolve_fks` –

    如果为 True，则反映`ForeignKey`对象位于每个`Table`中的`Table`对象。对于`MetaData.reflect()`，这将导致反映可能不在要反映的表列表中的相关表，例如，如果引用的表位于不同模式中或通过`MetaData.reflect.only`参数省略。当为 False 时，不会跟随`ForeignKey`对象到它们链接的`Table`，但是如果相关表也是无论如何将被反映的表列表的一部分，则在`MetaData.reflect()`操作完成后，`ForeignKey`对象仍将解析为其相关的`Table`。默认为 True。

    版本 1.3.0 中的新功能。

    另请参阅

    `Table.resolve_fks`

+   `**dialect_kwargs` – 上述未提及的额外关键字参数是特定于方言的，并以`<dialectname>_<argname>`的形式传递。有关各个方言的文档参数的详细信息，请参阅 Dialects。

另请参阅

反射数据库对象

`DDLEvents.column_reflect()` - 用于自定义反射列的事件。通常用于使用`TypeEngine.as_generic()`泛化类型。

使用与数据库无关的类型进行反射 - 描述如何使用通用类型反射表。

```py
method remove(table: Table) → None
```

从此 MetaData 中删除给定的 Table 对象。

```py
attribute sorted_tables
```

返回一个按外键依赖顺序排序的`Table`对象列表。

排序将会先放置具有依赖关系的`Table`对象，然后才是依赖对象本身，代表着它们可以被创建的顺序。要获取表被删除的顺序，请使用`reversed()` Python 内置函数。

警告

`MetaData.sorted_tables`属性本身无法自动解决表之间的依赖关系循环，这通常是由相互依赖的外键约束引起的。当检测到这些循环时，这些表的外键将被从排序考虑中省略。当这种情况发生时会发出警告，这将在未来版本中引发异常。不属于循环的表仍将按照依赖关系顺序返回。

要解决这些循环依赖，可以将`ForeignKeyConstraint.use_alter`参数应用于创建循环的约束。或者，当检测到循环时，`sort_tables_and_constraints()`函数将自动将外键约束返回到一个单独的集合中，以便可以单独应用到架构中。

从版本 1.3.17 开始更改：当由于循环依赖关系而无法进行适当排序时，将发出警告`MetaData.sorted_tables`。这将在未来版本中成为异常。此外，排序将继续返回未涉及循环的其他表，其顺序为依赖顺序，这在以前不是这样的。

另请参阅

`sort_tables()`

`sort_tables_and_constraints()`

`MetaData.tables`

`Inspector.get_table_names()`

`Inspector.get_sorted_table_and_fkc_names()`

```py
attribute tables: util.FacadeDict[str, Table]
```

一个由`Table`对象组成的字典，按它们的名称或“表键”键控。

确定的键由`Table.key`属性决定；对于没有`Table.schema`属性的表，这与`Table.name`相同。对于具有模式的表，它通常采用`schemaname.tablename`的形式。

另请参阅

`MetaData.sorted_tables`

```py
class sqlalchemy.schema.SchemaConst
```

一个枚举。

**成员**

BLANK_SCHEMA, NULL_UNSPECIFIED, RETAIN_SCHEMA

**类签名**

类`sqlalchemy.schema.SchemaConst` (`enum.Enum`)

```py
attribute BLANK_SCHEMA = 2
```

表示一个`Table`或`Sequence`应该具有“None”作为其模式，即使父`MetaData`已指定了一个模式。

另请参阅

`MetaData.schema`

`Table.schema`

`Sequence.schema`

```py
attribute NULL_UNSPECIFIED = 3
```

表示“nullable”关键字未传递给 Column 的符号。

这用于区分向`Column`传递`nullable=None`的用例，这在某些后端（如 SQL Server）上具有特殊含义。

```py
attribute RETAIN_SCHEMA = 1
```

表示`Table`、`Sequence`或在某些情况下表示`ForeignKey`对象的符号，在进行`Table.to_metadata()`操作时，应保留其已有的模式名称。

```py
class sqlalchemy.schema.SchemaItem
```

定义数据库模式的项目的基类。

**成员**

info

**类签名**

类`sqlalchemy.schema.SchemaItem`（`sqlalchemy.sql.expression.SchemaEventTarget`，`sqlalchemy.sql.visitors.Visitable`）

```py
attribute info
```

与对象关联的信息字典，允许将用户定义的数据与此`SchemaItem`关联。

字典在首次访问时会自动生成。它也可以在某些对象的构造函数中指定，例如`Table`和`Column`。

```py
function sqlalchemy.schema.insert_sentinel(name: str | None = None, type_: _TypeEngineArgument[_T] | None = None, *, default: Any | None = None, omit_from_statements: bool = True) → Column[Any]
```

提供一个代理`Column`，将充当专用的插入哨兵列，允许对不具有相应主键配置的表进行有效的批量插入，并确保按顺序进行 RETURNING 排序。

将此列添加到`Table`对象需要确保相应的数据库表实际上包含此列，因此如果将其添加到现有模型中，则需要对现有数据库表进行迁移（例如使用 ALTER TABLE 或类似的操作）以包含此列。

有关此对象的使用背景，请参阅部分配置哨兵列，作为部分“插入多个值”行为的 INSERT 语句的一部分。

返回的`Column`默认将是可空的整数列，并且仅在“insertmanyvalues”操作中使用哨兵特定的默认生成器。

另请参阅

`orm_insert_sentinel()`

`Column.insert_sentinel`

“插入多个值”行为的 INSERT 语句

配置哨兵列

新版本 2.0.10 中新增。

```py
class sqlalchemy.schema.Table
```

表示数据库中的表。

例如：

```py
mytable = Table(
    "mytable", metadata,
    Column('mytable_id', Integer, primary_key=True),
    Column('value', String(50))
)
```

`Table` 对象根据其名称和可选模式名称在给定的 `MetaData` 对象中构建一个独特的实例。使用相同的名称和相同的 `MetaData` 参数再次调用 `Table` 构造函数将返回*相同*的 `Table` 对象 - 这样，`Table` 构造函数就像一个注册函数。

另请参阅

使用 MetaData 描述数据库 - 数据库元数据介绍

**成员**

__init__(), add_is_dependent_on(), alias(), append_column(), append_constraint(), argument_for(), autoincrement_column, c, columns, compare(), compile(), constraints, corresponding_column(), create(), delete(), description, dialect_kwargs, dialect_options, drop(), entity_namespace, exported_columns, foreign_key_constraints, foreign_keys, get_children(), implicit_returning, indexes, info, inherit_cache, insert(), is_derived_from(), join(), key, kwargs, lateral(), outerjoin(), params(), primary_key, replace_selectable(), schema, select(), self_group(), table_valued(), tablesample(), to_metadata(), tometadata(), unique_params(), update()

**类签名**

类 `sqlalchemy.schema.Table` (`sqlalchemy.sql.base.DialectKWArgs`, `sqlalchemy.schema.HasSchemaAttr`, `sqlalchemy.sql.expression.TableClause`, `sqlalchemy.inspection.Inspectable`)

```py
method __init__(name: str, metadata: MetaData, *args: SchemaItem, schema: str | Literal[SchemaConst.BLANK_SCHEMA] | None = None, quote: bool | None = None, quote_schema: bool | None = None, autoload_with: Engine | Connection | None = None, autoload_replace: bool = True, keep_existing: bool = False, extend_existing: bool = False, resolve_fks: bool = True, include_columns: Collection[str] | None = None, implicit_returning: bool = True, comment: str | None = None, info: Dict[Any, Any] | None = None, listeners: _typing_Sequence[Tuple[str, Callable[..., Any]]] | None = None, prefixes: _typing_Sequence[str] | None = None, _extend_on: Set[Table] | None = None, _no_init: bool = True, **kw: Any) → None
```

`Table`的构造函数。

参数：

+   `name` –

    此表在数据库中表示的名称。

    表名，以及`schema`参数的值，形成一个键，唯一标识拥有的`MetaData`集合中的此`Table`。对具有相同名称、元数据和模式名称的`Table`进行的其他调用将返回相同的`Table`对象。

    不包含大写字符的名称将被视为不区分大小写的名称，并且除非它们是保留字或包含特殊字符，否则不会被引用。包含任何数量大写字符的名称被视为区分大小写的名称，并将被发送为引用。

    要为表名启用无条件引用，请在构造函数中指定标志`quote=True`，或使用`quoted_name`构造来指定名称。

+   `metadata` – 一个包含此表的`MetaData`对象。元数据用作将此表与通过外键引用的其他表关联的关联点。它还可以用于将此表与特定的`Connection`或`Engine`关联起来。

+   `*args` – 附加的位置参数主要用于添加包含在此表中的`Column`对象的列表。类似于 CREATE TABLE 语句的风格，其他`SchemaItem`构造可以在此处添加，包括`PrimaryKeyConstraint`和`ForeignKeyConstraint`。

+   `autoload_replace` –

    默认为`True`；当与`Table.extend_existing`一起使用`Table.autoload_with`时，指示应该用从 autoload 过程中检索到的相同名称的列替换已存在的`Table`对象中存在的`Column`对象。当为`False`时，已存在的列将被省略不包括在反射过程中。

    请注意，此设置不会影响通过调用`Table`程序指定的`Column`对象；当`Table.extend_existing`为`True`时，这些`Column`对象将始终替换同名的现有列。

    另请参阅

    `Table.autoload_with`

    `Table.extend_existing`

+   `autoload_with` –

    一个`Engine`或`Connection`对象，或由`inspect()`针对其中一个返回的`Inspector`对象，其将反映此`Table`对象。当设置为非`None`值时，autoload 过程将在此表针对给定的引擎或连接上进行。

    另请参阅

    反映数据库对象

    `DDLEvents.column_reflect()`

    使用与数据库无关的类型进行反射

+   `extend_existing` –

    当为`True`时，表示如果此`Table`已经存在于给定的`MetaData`中，则将构造函数中的进一步参数应用于现有的`Table`。

    如果未设置`Table.extend_existing`或`Table.keep_existing`，并且新`Table`的给定名称指的是目标`MetaData`集合中已经存在的`Table`，并且此`Table`指定了额外的列或其他构造或修改表状态的标志，将引发错误。这两个互斥标志的目的是指定在指定匹配现有`Table`但指定了其他构造的情况下应采取什么操作。

    `Table.extend_existing`也将与`Table.autoload_with`一起工作，针对数据库运行新的反射操作，即使目标`MetaData`中已经存在同名的`Table`；新反射的`Column`对象和其他选项将被添加到`Table`的状态中，可能会覆盖同名的现有列和选项。

    与`Table.autoload_with`一直如此，`Column`对象可以在同一`Table`构造函数中指定，这将优先考虑。下面，现有表`mytable`将被增加，其中包括从数据库反射的`Column`对象，以及给定的名为“y”的`Column`：

    ```py
    Table("mytable", metadata,
                Column('y', Integer),
                extend_existing=True,
                autoload_with=engine
            )
    ```

    另请参阅

    `Table.autoload_with`

    `Table.autoload_replace`

    `Table.keep_existing`

+   `implicit_returning` –

    默认为 True - 表示可以使用 RETURNING，通常由 ORM 使用，以获取服务器生成的值，如主键值和服务器端默认值，在支持 RETURNING 的后端上。

    在现代 SQLAlchemy 中，通常没有理由更改此设置，除非是一些特定于后端的情况（请参阅 SQL Server 方言文档中的 Triggers 以获取一个示例）。

+   `include_columns` – 一个字符串列表，指示通过`autoload`操作加载的列的子集；不在此列表中的表列将不会在生成的`Table`对象上表示。默认为`None`，表示应反映所有列。

+   `resolve_fks` –

    是否反映与此对象相关的 `Table` 对象，通过 `ForeignKey` 对象，当指定 `Table.autoload_with` 时。默认为 True。设置为 False 以禁用通过 `ForeignKey` 对象反映相关表；可以用于节省 SQL 调用或避免无法访问的相关表的问题。请注意，如果相关表已经存在于 `MetaData` 集合中，或者稍后存在，与此 `Table` 关联的 `ForeignKey` 对象将正常解析为该表。

    1.3 版本中的新功能。

    另见

    `MetaData.reflect.resolve_fks`

+   `info` – 将填充到此对象的 `SchemaItem.info` 属性中的可选数据字典。

+   `keep_existing` –

    当设置为 `True` 时，表示如果这个 `Table` 已经存在于给定的 `MetaData` 中，则忽略构造函数内部的进一步参数，并将 `Table` 对象返回为最初创建的对象。这是为了允许一个希望在第一次调用时定义一个新的 `Table` 的函数，但在后续调用中将返回相同的 `Table`，而不会应用任何声明（特别是约束）第二次。

    如果未设置`Table.extend_existing`或`Table.keep_existing`，并且新`Table`的给定名称引用的`Table`已经存在于目标`MetaData`集合中，并且这个`Table`指定了额外的列或其他构造或修改表状态的标志，将引发错误。这两个互斥标志的目的是指定当指定一个与现有的`Table`匹配的`Table`时应该采取什么操作，而又指定了额外的构造。

    参见

    `Table.extend_existing`

+   `listeners` –

    一个形如 `(<eventname>, <fn>)` 的元组列表，将在构造时传递给`listen()`。这个替代钩子用于在“自动加载”过程开始之前针对这个`Table`建立特定的监听函数。历史上，这被用于与`DDLEvents.column_reflect()`事件一起使用，但请注意，现在这个事件钩子可以直接与`MetaData`对象关联：

    ```py
    def listen_for_reflect(table, column_info):
        "handle the column reflection event"
        # ...

    t = Table(
        'sometable',
        autoload_with=engine,
        listeners=[
            ('column_reflect', listen_for_reflect)
        ])
    ```

    参见

    `DDLEvents.column_reflect()`

+   `must_exist` – 当为`True`时，表示这个 Table 必须已经存在于给定的`MetaData`集合中，否则会引发异常。

+   `prefixes` – 在 CREATE TABLE 语句中在 CREATE 之后插入的字符串列表。它们将用空格分隔。

+   `quote` –

    强制对这个表的名称进行引用，对应为`True`或`False`。当保持默认值`None`时，根据名称是否区分大小写（至少有一个大写字符的标识符被视为区分大小写），或者是否为保留字来引用列标识符。这个标志只需要强制引用一个 SQLAlchemy 方言不知道的保留字。

    注意

    将此标志设置为`False`将不会为表反射提供不区分大小写的行为；表反射将始终以区分大小写的方式搜索混合大小写名称。 在 SQLAlchemy 中，仅通过使用所有小写字符的名称来指定不区分大小写的名称。

+   `quote_schema` – 与‘quote’相同，但适用于模式标识符。

+   `schema` –

    此表的模式名称，如果该表位于引擎的数据库连接的默认选择模式之外的模式中，则此名称是必需的。 默认为`None`。

    如果此`Table`的所有者`MetaData`指定了自己的`MetaData.schema`参数，则如果此处的模式参数设置为`None`，则该模式名称将应用于此`Table`。 要在否则将使用所设置的模式的`MetaData`上指定空白模式名称的`Table`，请指定特殊符号`BLANK_SCHEMA`。

    模式名称的引号规则与`name`参数的引号规则相同，即针对保留字或区分大小写的名称应用引号； 要为模式名称启用无条件引号，请将`quote_schema=True`标志传递给构造函数，或使用`quoted_name`构造来指定名称。

+   `comment` –

    可选字符串，将在表创建时渲染 SQL 注释。

    自版本 1.2 新增：在`Table`中添加了`Table.comment`参数。

+   `**kw` – 未提及的其他关键字参数是特定于方言的，并以`<dialectname>_<argname>`的形式传递。 有关详细信息，请参阅个别方言的文档 Dialects。

```py
method add_is_dependent_on(table: Table) → None
```

为此表添加一个“dependency”。

这是另一个必须在此表之前创建或在此表之后删除的 Table 对象。

通常，表之间的依赖关系是通过 ForeignKey 对象确定的。 但是，对于创建在外键之外的依赖关系的其他情况（规则，继承），此方法可以手动建立这样的链接。

```py
method alias(name: str | None = None, flat: bool = False) → NamedFromClause
```

*继承自* `FromClause.alias()` *方法的* `FromClause`

返回此 `FromClause` 的别名。

例如：

```py
a2 = some_table.alias('a2')
```

上述代码创建了一个 `Alias` 对象，可以在任何 SELECT 语句中作为 FROM 子句使用。

另请参阅

使用别名

`alias()`

```py
method append_column(column: ColumnClause[Any], replace_existing: bool = False) → None
```

向此 `Table` 添加一个 `Column`。

新添加的 `Column` 的“键”，即其 `.key` 属性的值，将在此 `Table` 的 `.c` 集合中可用，并且该列定义将包含在从此 `Table` 构造生成的任何 CREATE TABLE、SELECT、UPDATE 等语句中。

注意，这 **不会** 更改表的定义，因为它存在于任何底层数据库中，假设该表已经在数据库中创建。关系数据库支持向现有表添加列，使用 SQL ALTER 命令即可，对于已存在但不包含新增列的表，需要发出此命令。

参数：

**replace_existing** –

当为 `True` 时，允许替换现有列。当为 `False`（默认）时，如果具有相同 `.key` 的列已存在，则会发出警告。未来版本的 SQLAlchemy 将会发出警告。

在 1.4.0 版中新增。

```py
method append_constraint(constraint: Index | Constraint) → None
```

向此 `Table` 添加一个 `Constraint`。

这将使约束包含在任何将来的 CREATE TABLE 语句中，假设没有将特定的 DDL 创建事件与给定的 `Constraint` 对象关联。

注意，这 **不会** 自动在关系数据库中生成约束，对于已经存在于数据库中的表。要向现有的关系数据库表添加约束，必须使用 SQL ALTER 命令。SQLAlchemy 还提供了 `AddConstraint` 结构，当作为可执行子句调用时，可以生成此 SQL。

```py
classmethod argument_for(dialect_name, argument_name, default)
```

*继承自* `DialectKWArgs` *类的* `DialectKWArgs.argument_for()` *方法*

为此类添加一种新的特定于方言的关键字参数。

例如：

```py
Index.argument_for("mydialect", "length", None)

some_index = Index('a', 'b', mydialect_length=5)
```

`DialectKWArgs.argument_for()` 方法是一种通过将额外参数添加到`DefaultDialect.construct_arguments` 字典中的一种方法来为每个参数添加额外参数的方法。此字典为代表方言的各种模式级别构造提供了接受的参数名称列表。

新的方言通常应该一次性指定该字典，作为方言类的数据成员。通常情况下，用于额外添加参数名称的用例是针对同时使用自定义编译方案的最终用户代码，该编译方案使用额外的参数。

参数：

+   `dialect_name` – 方言的名称。方言必须是可定位的，否则会引发`NoSuchModuleError`。方言还必须包含现有的`DefaultDialect.construct_arguments`集合，表示它参与关键字参数验证和默认系统，否则会引发`ArgumentError`。如果方言不包含此集合，则可以为此方言已经指定任何关键字参数。SQLAlchemy 中的所有打包的方言都包括此集合，但是对于第三方方言，支持可能有所不同。

+   `argument_name` – 参数的名称。

+   `default` – 参数的默认值。

```py
attribute autoincrement_column
```

返回当前表示“自动增量”列的`Column`对象，如果没有，则返回 None。

这基于由`Column.autoincrement`参数定义的`Column`的规则，通常意味着不受外键约束的单个整数列主键约束中的列。如果表没有这样的主键约束，则没有“自动增量”列。一个`Table`只能定义一个列作为“自动增量”列。

从版本 2.0.4 开始新的。

另请参见

`Column.autoincrement`

```py
attribute c
```

*继承自* `FromClause` *属性的* `FromClause.c`

`FromClause.columns` 的同义词

返回：

一个 `ColumnCollection`

```py
attribute columns
```

*继承自* `FromClause.columns` *属性的* `FromClause`

由此 `FromClause` 维护的基于名称的 `ColumnElement` 对象集合。

`columns` 或 `c` 集合，是使用绑定到表或其他可选择的列构建 SQL 表达式的入口点：

```py
select(mytable).where(mytable.c.somecolumn == 5)
```

返回：

一个 `ColumnCollection` 对象。

```py
method compare(other: ClauseElement, **kw: Any) → bool
```

*继承自* `ClauseElement.compare()` *方法的* `ClauseElement`

将此 `ClauseElement` 与给定的 `ClauseElement` 进行比较。

子类应该覆盖默认行为，即直接标识比较。

**kw 是子类 `compare()` 方法消耗的参数，可以用于修改比较的标准（参见 `ColumnElement`）。

```py
method compile(bind: _HasDialect | None = None, dialect: Dialect | None = None, **kw: Any) → Compiled
```

*继承自* `CompilerElement.compile()` *方法的* `CompilerElement`

编译此 SQL 表达式。

返回值是一个 `Compiled` 对象。对返回值调用 `str()` 或 `unicode()` 将产生结果的字符串表示。`Compiled` 对象还可以使用 `params` 访问器返回绑定参数名称和值的字典。

参数：

+   `bind` – 一个可以提供`Dialect`以生成`Compiled`对象的`Connection`或`Engine`。如果`bind`和`dialect`参数都被省略，将使用默认的 SQL 编译器。

+   `column_keys` – 用于 INSERT 和 UPDATE 语句，应在编译语句的 VALUES 子句中出现的列名列表。如果为`None`，则渲染目标表对象的所有列。

+   `dialect` – 一个可以生成`Compiled`对象的`Dialect`实例。此参数优先于`bind`参数。

+   `compile_kwargs` –

    可选的附加参数字典，将在所有“visit”方法中传递给编译器。这允许将任何自定义标志传递给自定义编译结构，例如。它还用于通过`literal_binds`标志传递的情况：

    ```py
    from sqlalchemy.sql import table, column, select

    t = table('t', column('x'))

    s = select(t).where(t.c.x == 5)

    print(s.compile(compile_kwargs={"literal_binds": True}))
    ```

另请参阅

如何将 SQL 表达式呈现为字符串，可能包含内联的绑定参数？

```py
attribute constraints: Set[Constraint]
```

与此`Table`相关联的所有`Constraint`对象的集合。

包括`PrimaryKeyConstraint`、`ForeignKeyConstraint`、`UniqueConstraint`、`CheckConstraint`。一个单独的集合`Table.foreign_key_constraints`指的是所有与该`Table`相关联的`ForeignKeyConstraint`对象的集合，而`Table.primary_key`属性指的是与该`Table`相关联的单个`PrimaryKeyConstraint`。

另请参阅

`Table.constraints`

`Table.primary_key`

`Table.foreign_key_constraints`

`Table.indexes`

`Inspector`

```py
method corresponding_column(column: KeyedColumnElement[Any], require_embedded: bool = False) → KeyedColumnElement[Any] | None
```

*继承自* `Selectable.corresponding_column()` *方法的* `Selectable`

给定一个`ColumnElement`，从此`Selectable`的`Selectable.exported_columns`集合中返回与该原始`ColumnElement`通过共同祖先列对应的导出`ColumnElement`对象。

参数：

+   `column` – 要匹配的目标`ColumnElement`。

+   `require_embedded` – 只返回给定`ColumnElement`对应的列，如果给定的`ColumnElement`实际上存在于此`Selectable`的子元素中。通常，如果列仅与此`Selectable`的导出列之一共享共同的祖先，则列将匹配。

另请参阅

`Selectable.exported_columns` - 用于操作的`ColumnCollection`。

`ColumnCollection.corresponding_column()` - 实现方法。

```py
method create(bind: _CreateDropBind, checkfirst: bool = False) → None
```

为此 `Table` 发出一个 `CREATE` 语句，使用给定的 `Connection` 或 `Engine` 进行连接。

请参阅

`MetaData.create_all()`.

```py
method delete() → Delete
```

*继承自* `TableClause.delete()` *方法的* `TableClause`

针对此 `TableClause` 生成一个 `delete()` 构造。

例如：

```py
table.delete().where(table.c.id==7)
```

有关参数和用法信息，请参阅`delete()`。

```py
attribute description
```

*继承自* `TableClause.description` *属性的* `TableClause`

```py
attribute dialect_kwargs
```

*继承自* `DialectKWArgs.dialect_kwargs` *属性的* `DialectKWArgs`

作为方言特定选项指定的关键字参数集合。

这里的参数以其原始的 `<dialect>_<kwarg>` 格式呈现。只包括实际传递的参数；不像 `DialectKWArgs.dialect_options` 集合那样，其中包含了此方言的所有已知选项，包括默认值。

此集合也是可写的；键采用 `<dialect>_<kwarg>` 形式，其中的值将被组装到选项列表中。

请参阅

`DialectKWArgs.dialect_options` - 嵌套字典形式

```py
attribute dialect_options
```

*继承自* `DialectKWArgs.dialect_options` *属性的* `DialectKWArgs`

作为方言特定选项指定的关键字参数集合。

这是一个两级嵌套注册表，键为 `<dialect_name>` 和 `<argument_name>`。例如，`postgresql_where` 参数可以定位为：

```py
arg = my_object.dialect_options['postgresql']['where']
```

自版本 0.9.2 新增。

另请参阅

`DialectKWArgs.dialect_kwargs` - 平面字典形式

```py
method drop(bind: _CreateDropBind, checkfirst: bool = False) → None
```

使用给定的 `Connection` 或 `Engine` 发出此 `Table` 的`DROP`语句以进行连接。

另请参阅

`MetaData.drop_all()`.

```py
attribute entity_namespace
```

*继承自* `FromClause.entity_namespace` *属性的* `FromClause`

返回用于 SQL 表达式中基于名称访问的命名空间。

这是用于解析“filter_by()”类型表达式的命名空间，例如：

```py
stmt.filter_by(address='some address')
```

默认为`.c`集合，但在内部可以使用“entity_namespace”注解进行覆盖以提供替代结果。

```py
attribute exported_columns
```

*继承自* `FromClause.exported_columns` *属性的* `FromClause`

代表此 `Selectable` 的“导出”列的 `ColumnCollection`。

`FromClause` 对象的“导出”列与 `FromClause.columns` 集合是同义词。

自版本 1.4 新增。

另请参阅

`Selectable.exported_columns`

`SelectBase.exported_columns`

```py
attribute foreign_key_constraints
```

被此 `Table` 引用的 `ForeignKeyConstraint` 对象。

此列表是当前关联的 `ForeignKey` 对象集合生成的。

另请参阅

`Table.constraints`

`Table.foreign_keys`

`Table.indexes`

```py
attribute foreign_keys
```

*继承自* `FromClause.foreign_keys` *属性的* `FromClause`

返回此 FromClause 引用的所有`ForeignKey`标记对象的集合。

每个`ForeignKey`都是`Table`范围内的`ForeignKeyConstraint`的成员。

另请参阅

`Table.foreign_key_constraints`

```py
method get_children(*, omit_attrs: Tuple[str, ...] = (), **kw: Any) → Iterable[HasTraverseInternals]
```

*继承自* `HasTraverseInternals.get_children()` *方法的* `HasTraverseInternals`

返回此`HasTraverseInternals`的直接子级`HasTraverseInternals`元素。

这用于访问遍历。

**kw 可能包含更改返回的集合的标志，例如返回子集以减少较大遍历的项，或者从不同上下文（例如模式级集合而不是子句级）返回子项。

```py
attribute implicit_returning = False
```

*继承自* `TableClause.implicit_returning` *属性的* `TableClause`

`TableClause`不支持具有主键或列级默认值，因此隐式返回不适用。

```py
attribute indexes: Set[Index]
```

与此`Table`相关联的所有`Index`对象的集合。

另请参阅

`Inspector.get_indexes()`

```py
attribute info
```

*继承自* `SchemaItem.info` *属性的* `SchemaItem`

与对象关联的信息字典，允许将用户定义的数据与此`SchemaItem`关联。

当首次访问时，该字典会自动生成。也可以在某些对象的构造函数中指定，例如 `Table` 和 `Column`。

```py
attribute inherit_cache: bool | None = None
```

*继承自* `HasCacheKey.inherit_cache` *属性的* `HasCacheKey` *对象*

指示此 `HasCacheKey` 实例是否应使用其直接超类使用的缓存密钥生成方案。

该属性默认为`None`，表示构造尚未考虑是否适合参与缓存；这在功能上等同于将值设置为`False`，只是还会发出警告。

如果与此类本地属性而不是其超类相关的属性不会更改与对象对应的 SQL，则可以将此标志设置为 `True`。

另请参阅

为自定义构造启用缓存支持 - 设置第三方或用户定义的 SQL 构造的`HasCacheKey.inherit_cache` 属性的一般指南。

```py
method insert() → Insert
```

*继承自* `TableClause.insert()` *方法的* `TableClause` *对象*

针对此 `TableClause` 生成一个 `Insert` 构造。

例如：

```py
table.insert().values(name='foo')
```

有关参数和用法信息，请参阅 `insert()`。

```py
method is_derived_from(fromclause: FromClause | None) → bool
```

*继承自* `FromClause.is_derived_from()` *方法的* `FromClause` *对象*

如果此 `FromClause` 从给定的 `FromClause`‘派生’，则返回`True`。

一个例子就是一个表的别名是从该表派生出来的。

```py
method join(right: _FromClauseArgument, onclause: _ColumnExpressionArgument[bool] | None = None, isouter: bool = False, full: bool = False) → Join
```

*继承自* `FromClause.join()` *方法的* `FromClause` *对象*

从此`FromClause`返回到另一个`FromClause`的`Join`。

例如：

```py
from sqlalchemy import join

j = user_table.join(address_table,
                user_table.c.id == address_table.c.user_id)
stmt = select(user_table).select_from(j)
```

会生成类似以下的 SQL：

```py
SELECT user.id, user.name FROM user
JOIN address ON user.id = address.user_id
```

参数：

+   `right` – 连接的右侧；这是任何`FromClause`对象，如`Table`对象，也可以是可选择兼容的对象，如 ORM 映射类。

+   `onclause` – 一个代表连接的 ON 子句的 SQL 表达式。如果保持为`None`，`FromClause.join()` 将尝试基于外键关系连接这两个表。

+   `isouter` – 如果为 True，则渲染一个 LEFT OUTER JOIN，而不是 JOIN。

+   `full` – 如果为 True，则渲染一个 FULL OUTER JOIN，而不是 LEFT OUTER JOIN。暗示`FromClause.join.isouter`。

另请参见

`join()` - 独立函数

`Join` - 生成的对象类型

```py
attribute key
```

返回此`Table`的‘key’。

此值用作`MetaData.tables`集合中的字典键。对于没有设置`Table.schema`的表，通常与`Table.name`相同；否则，通常为`schemaname.tablename`形式。

```py
attribute kwargs
```

*继承自* `DialectKWArgs.kwargs` *属性的* `DialectKWArgs`

`DialectKWArgs.dialect_kwargs`的同义词。

```py
method lateral(name: str | None = None) → LateralFromClause
```

*继承自* `Selectable.lateral()` *方法的* `Selectable`

返回此`Selectable`的 LATERAL 别名。

返回值是`Lateral`构造，也由顶级`lateral()`函数提供。

另请参阅

LATERAL correlation - 使用概述。

```py
method outerjoin(right: _FromClauseArgument, onclause: _ColumnExpressionArgument[bool] | None = None, full: bool = False) → Join
```

*继承自* `FromClause.outerjoin()` *方法的* `FromClause`

从这个`FromClause`返回到另一个 `FromClause` 的 `Join`，将“isouter”标志设置为 True。

例如：

```py
from sqlalchemy import outerjoin

j = user_table.outerjoin(address_table,
                user_table.c.id == address_table.c.user_id)
```

上述等价于：

```py
j = user_table.join(
    address_table,
    user_table.c.id == address_table.c.user_id,
    isouter=True)
```

参数：

+   `right` – 连接的右侧；这是任何 `FromClause` 对象，如 `Table` 对象，也可以是一个可选择兼容对象，如 ORM 映射类。

+   `onclause` – 表示连接的 ON 子句的 SQL 表达式。如果保持为 `None`，`FromClause.join()` 将尝试基于外键关系连接两个表。

+   `full` – 如果为 True，则渲染一个 FULL OUTER JOIN，而不是 LEFT OUTER JOIN。

另请参阅

`FromClause.join()`

`Join`

```py
method params(*optionaldict, **kwargs)
```

*继承自* `Immutable.params()` *方法的* `Immutable`

返回带有 `bindparam()` 元素替换的副本。

返回此 ClauseElement 的副本，其中的 `bindparam()` 元素替换为给定字典中取出的值：

```py
>>> clause = column('x') + bindparam('foo')
>>> print(clause.compile().params)
{'foo':None}
>>> print(clause.params({'foo':7}).compile().params)
{'foo':7}
```

```py
attribute primary_key
```

*继承自* `FromClause.primary_key` *属性的* `FromClause`

返回这个 `_selectable.FromClause` 的主键组成的 `Column` 对象的可迭代集合。

对于`Table`对象，此集合由`PrimaryKeyConstraint`表示，它本身是一个`Column`对象的可迭代集合。

```py
method replace_selectable(old: FromClause, alias: Alias) → Self
```

*继承自* `Selectable.replace_selectable()` *方法的* `Selectable`

使用给定的`Alias`对象替换所有出现的`FromClause` ‘old’，返回此`FromClause`的副本。

自版本 1.4 起已弃用：`Selectable.replace_selectable()`方法已弃用，并将在将来的发布中删除。类似的功能可通过 sqlalchemy.sql.visitors 模块获得。

```py
attribute schema: str | None = None
```

*继承自* `FromClause.schema` *属性的* `FromClause`

为此`FromClause`定义‘schema’属性。

对于大多数对象，这通常是`None`，除了`Table`的情况，其中它被视为`Table.schema`参数的值。

```py
method select() → Select
```

*继承自* `FromClause.select()` *方法的* `FromClause`

返回此`FromClause`的 SELECT。

例如：

```py
stmt = some_table.select().where(some_table.c.id == 5)
```

另请参阅

`select()` - 允许任意列列表的通用方法。

```py
method self_group(against: OperatorType | None = None) → ClauseElement
```

*继承自* `ClauseElement.self_group()` *方法的* `ClauseElement`

对这个 `ClauseElement` 进行“分组”。

子类将重写此方法以返回一个“分组”构造，即括号。特别是它被“二进制”表达式用于在放置到更大表达式中时提供自身周围的分组，以及被放置到另一个 `select()` 的 FROM 子句中的 `select()` 构造使用。（请注意，通常应使用 `Select.alias()` 方法创建子查询，因为许多平台要求嵌套的 SELECT 语句有名称）。

随着表达式的组合，`self_group()` 的应用是自动的 - 最终用户代码不应直接使用此方法。请注意，SQLAlchemy 的子句构造会考虑运算符优先级 - 因此，例如在表达式 `x OR (y AND z)` 中可能不需要括号 - AND 的优先级高于 OR。

`ClauseElement` 的基本方法 `self_group()` 只是返回自身。

```py
method table_valued() → TableValuedColumn[Any]
```

*继承自* `NamedFromClause.table_valued()` *方法的* `NamedFromClause`

为这个 `FromClause` 返回一个 `TableValuedColumn` 对象。

`TableValuedColumn` 是代表表中完整行的 `ColumnElement`。对于这个构造的支持取决于后端，各后端以不同形式支持，如 PostgreSQL、Oracle 和 SQL Server。

例如：

```py
>>> from sqlalchemy import select, column, func, table
>>> a = table("a", column("id"), column("x"), column("y"))
>>> stmt = select(func.row_to_json(a.table_valued()))
>>> print(stmt)
SELECT  row_to_json(a)  AS  row_to_json_1
FROM  a 
```

版本 1.4.0b2 中的新功能。

另请参阅

使用 SQL 函数 - 在 SQLAlchemy 统一教程 中

```py
method tablesample(sampling: float | Function[Any], name: str | None = None, seed: roles.ExpressionElementRole[Any] | None = None) → TableSample
```

*继承自* `FromClause.tablesample()` *方法的* `FromClause`

返回这个 `FromClause` 的 TABLESAMPLE 别名。

返回值是由顶级`tablesample()`函数提供的`TableSample`构造。

另请参阅

`tablesample()` - 用法指南和参数

```py
method to_metadata(metadata: MetaData, schema: str | Literal[SchemaConst.RETAIN_SCHEMA] = SchemaConst.RETAIN_SCHEMA, referred_schema_fn: Callable[[Table, str | None, ForeignKeyConstraint, str | None], str | None] | None = None, name: str | None = None) → Table
```

返回与不同`MetaData`相关联的此`Table`的副本。

例如：

```py
m1 = MetaData()

user = Table('user', m1, Column('id', Integer, primary_key=True))

m2 = MetaData()
user_copy = user.to_metadata(m2)
```

从版本 1.4 开始更改：`Table.to_metadata()`函数的名称已从`Table.tometadata()`更改。

参数：

+   `metadata` – 目标`MetaData`对象，新的`Table`对象将被创建到其中。

+   `schema` –

    可选的字符串名称，指示目标模式。默认为特殊符号`RETAIN_SCHEMA`，表示在新的`Table`中不应更改模式名称。如果设置为字符串名称，则新的`Table`将具有此新名称作为`.schema`。如果设置为`None`，则模式将设置为在目标`MetaData`上设置的模式，该模式通常也为`None`，除非明确设置：

    ```py
    m2 = MetaData(schema='newschema')

    # user_copy_one will have "newschema" as the schema name
    user_copy_one = user.to_metadata(m2, schema=None)

    m3 = MetaData()  # schema defaults to None

    # user_copy_two will have None as the schema name
    user_copy_two = user.to_metadata(m3, schema=None)
    ```

+   `referred_schema_fn` –

    可选的可调用函数，用于提供应分配给`ForeignKeyConstraint`的引用表的模式名称。该可调用函数接受此父`Table`、我们要更改的目标模式、`ForeignKeyConstraint`对象和该约束的现有“目标模式”。该函数应返回应应用的字符串模式名称。要重置模式为“none”，请返回符号`BLANK_SCHEMA`。要不进行更改，请返回`None`或`RETAIN_SCHEMA`。

    从版本 1.4.33 开始更改：`referred_schema_fn`函数可能返回`BLANK_SCHEMA`或`RETAIN_SCHEMA`符号。

    例如：

    ```py
    def referred_schema_fn(table, to_schema,
                                    constraint, referred_schema):
        if referred_schema == 'base_tables':
            return referred_schema
        else:
            return to_schema

    new_table = table.to_metadata(m2, schema="alt_schema",
                            referred_schema_fn=referred_schema_fn)
    ```

+   `name` – 可选的字符串名称，指示目标表名称。如果未指定或为 None，则保留表名称。这允许将 `Table` 复制到具有新名称的相同 `MetaData` 目标。

```py
method tometadata(metadata: MetaData, schema: str | Literal[SchemaConst.RETAIN_SCHEMA] = SchemaConst.RETAIN_SCHEMA, referred_schema_fn: Callable[[Table, str | None, ForeignKeyConstraint, str | None], str | None] | None = None, name: str | None = None) → Table
```

返回与不同 `MetaData` 关联的此 `Table` 的副本。

自版本 1.4 弃用： `Table.tometadata()` 已重命名为 `Table.to_metadata()`

查看 `Table.to_metadata()` 获取完整描述。

```py
method unique_params(*optionaldict, **kwargs)
```

*继承自* `Immutable.unique_params()` *方法的* `Immutable`

返回一个副本，其中 `bindparam()` 元素已替换。

与 `ClauseElement.params()` 相同的功能，只是将 unique=True 添加到受影响的绑定参数，以便可以使用多个语句。

```py
method update() → Update
```

*继承自* `TableClause.update()` *方法的* `TableClause`

对此 `TableClause` 生成一个 `update()` 构造。

例如：

```py
table.update().where(table.c.id==7).values(name='foo')
```

查看 `update()` 获取参数和使用信息。

## 访问表和列

`MetaData` 对象包含我们与之关联的所有模式构造。它支持几种访问这些表对象的方法，例如 `sorted_tables` 访问器，它以外键依赖关系的顺序返回每个 `Table` 对象的列表（也就是说，每个表之前都有它引用的所有表）：

```py
>>> for t in metadata_obj.sorted_tables:
...     print(t.name)
user
user_preference
invoice
invoice_item
```

在大多数情况下，单独的 `Table` 对象已被明确声明，并且这些对象通常作为应用程序中的模块级变量直接访问。一旦定义了 `Table`，它就有了一整套访问器，允许检查其属性。给定以下 `Table` 定义：

```py
employees = Table(
    "employees",
    metadata_obj,
    Column("employee_id", Integer, primary_key=True),
    Column("employee_name", String(60), nullable=False),
    Column("employee_dept", Integer, ForeignKey("departments.department_id")),
)
```

注意此表中使用的 `ForeignKey` 对象 - 此构造定义了对远程表的引用，在定义外键中完全描述了方法访问关于此表的信息包括：

```py
# access the column "employee_id":
employees.columns.employee_id

# or just
employees.c.employee_id

# via string
employees.c["employee_id"]

# a tuple of columns may be returned using multiple strings
# (new in 2.0)
emp_id, name, type = employees.c["employee_id", "name", "type"]

# iterate through all columns
for c in employees.c:
    print(c)

# get the table's primary key columns
for primary_key in employees.primary_key:
    print(primary_key)

# get the table's foreign key objects:
for fkey in employees.foreign_keys:
    print(fkey)

# access the table's MetaData:
employees.metadata

# access a column's name, type, nullable, primary key, foreign key
employees.c.employee_id.name
employees.c.employee_id.type
employees.c.employee_id.nullable
employees.c.employee_id.primary_key
employees.c.employee_dept.foreign_keys

# get the "key" of a column, which defaults to its name, but can
# be any user-defined string:
employees.c.employee_name.key

# access a column's table:
employees.c.employee_id.table is employees

# get the table related by a foreign key
list(employees.c.employee_dept.foreign_keys)[0].column.table
```

提示

`FromClause.c` 集合，与`FromClause.columns` 集合同义，是`ColumnCollection` 的一个实例，它提供了**类似字典的接口**来访问列集合。通常可以像访问属性名一样访问名称，例如`employees.c.employee_name`。但是对于具有空格的特殊名称或与字典方法名称匹配的名称，例如`ColumnCollection.keys()` 或 `ColumnCollection.values()`，必须使用索引访问，例如`employees.c['values']` 或 `employees.c["some column"]`。有关详细信息，请参阅`ColumnCollection`。

## 创建和删除数据库表

一旦您定义了一些 `Table` 对象，假设您正在使用全新的数据库，您可能希望为这些表及其相关构造发出 CREATE 语句（作为一种附带说明，如果您已经有一些首选方法，例如数据库中包含的工具或现有的脚本系统 - 如果是这种情况，请随时跳过此部分 - SQLAlchemy 不要求使用它来创建您的表）。

发出 CREATE 语句的常规方式是在 `MetaData` 对象上使用 `create_all()` 方法。此方法将发出查询，首先检查每个单独表的存在，如果未找到将发出 CREATE 语句：

```py
engine = create_engine("sqlite:///:memory:")

metadata_obj = MetaData()

user = Table(
    "user",
    metadata_obj,
    Column("user_id", Integer, primary_key=True),
    Column("user_name", String(16), nullable=False),
    Column("email_address", String(60), key="email"),
    Column("nickname", String(50), nullable=False),
)

user_prefs = Table(
    "user_prefs",
    metadata_obj,
    Column("pref_id", Integer, primary_key=True),
    Column("user_id", Integer, ForeignKey("user.user_id"), nullable=False),
    Column("pref_name", String(40), nullable=False),
    Column("pref_value", String(100)),
)

metadata_obj.create_all(engine)
PRAGMA  table_info(user){}
CREATE  TABLE  user(
  user_id  INTEGER  NOT  NULL  PRIMARY  KEY,
  user_name  VARCHAR(16)  NOT  NULL,
  email_address  VARCHAR(60),
  nickname  VARCHAR(50)  NOT  NULL
)
PRAGMA  table_info(user_prefs){}
CREATE  TABLE  user_prefs(
  pref_id  INTEGER  NOT  NULL  PRIMARY  KEY,
  user_id  INTEGER  NOT  NULL  REFERENCES  user(user_id),
  pref_name  VARCHAR(40)  NOT  NULL,
  pref_value  VARCHAR(100)
) 
```

`create_all()` 在表定义本身之间创建外键约束，通常会生成表的依赖顺序。有更改此行为的选项，使其使用`ALTER TABLE`。

类似地，可以使用`drop_all()`方法来删除所有表。此方法与`create_all()`完全相反-首先检查每个表的存在，并按依赖关系的相反顺序删除表。

可以通过`Table`的`create()`和`drop()`方法来创建和删除单个表。这些方法默认情况下会无论表是否存在都发出 CREATE 或 DROP：

```py
engine = create_engine("sqlite:///:memory:")

metadata_obj = MetaData()

employees = Table(
    "employees",
    metadata_obj,
    Column("employee_id", Integer, primary_key=True),
    Column("employee_name", String(60), nullable=False, key="name"),
    Column("employee_dept", Integer, ForeignKey("departments.department_id")),
)
employees.create(engine)
CREATE  TABLE  employees(
  employee_id  SERIAL  NOT  NULL  PRIMARY  KEY,
  employee_name  VARCHAR(60)  NOT  NULL,
  employee_dept  INTEGER  REFERENCES  departments(department_id)
)
{} 
```

`drop()`方法：

```py
employees.drop(engine)
DROP  TABLE  employees
{} 
```

要启用“首先检查表是否存在”的逻辑，需要在`create()`或`drop()`中添加`checkfirst=True`参数：

```py
employees.create(engine, checkfirst=True)
employees.drop(engine, checkfirst=False)
```

## 通过迁移修改数据库对象

虽然 SQLAlchemy 直接支持发出用于模式构造的 CREATE 和 DROP 语句，但是通过 ALTER 语句以及其他特定于数据库的构造修改这些构造的能力通常不在 SQLAlchemy 本身的范围之内。尽管手工发出 ALTER 语句等很容易，比如通过将`text()`构造传递给`Connection.execute()`或使用`DDL`构造，但是使用模式迁移工具自动维护与应用程序代码相关的数据库模式是一种常见做法。

SQLAlchemy 项目提供了[迁移工具 Alembic](https://alembic.sqlalchemy.org)。Alembic 具有高度可定制的环境和简约的使用模式，支持诸如事务性 DDL、自动生成“候选”迁移、生成 SQL 脚本的“离线”模式以及分支解决支持等功能。

Alembic 取代了[SQLAlchemy-Migrate](https://github.com/openstack/sqlalchemy-migrate)项目，后者是 SQLAlchemy 的最初迁移工具，现在已被视为过时。

## 指定模式名称

大多数数据库支持多个“模式”的概念-指代替代表集和其他构造的命名空间。 “模式”的服务器端几何形状采用多种形式，包括特定数据库范围内的“模式”名称（例如，PostgreSQL 模式），命名的同级数据库（例如，MySQL / MariaDB 访问同一服务器上的其他数据库），以及其他概念，如由其他用户名拥有的表（Oracle，SQL Server）甚至是指代替代数据库文件（SQLite ATTACH）或远程服务器（带有同义词的 Oracle DBLINK）的名称。

上述所有方法（大多数）的共同之处是，有一种引用此备选表集的方式，使用字符串名称。SQLAlchemy 将此名称称为**模式名称**。在 SQLAlchemy 中，这只是一个与`Table`对象关联的字符串名称，然后以适合于目标数据库的方式呈现为 SQL 语句，从而在目标数据库上引用表时使用其远程“模式”。

“模式”名称可以直接与`Table`关联，使用`Table.schema`参数；当使用 ORM 进行声明性表配置时，该参数将通过`__table_args__`参数字典传递。

“模式”名称也可以与`MetaData`对象关联，在此情况下，它将自动影响所有与该`MetaData`关联的`Table`对象，这些对象不会另外指定自己的名称。最后，SQLAlchemy 还支持一个“动态”模式名称系统，通常用于多租户应用程序，以便单个`Table`元数据集可以引用每个连接或语句基础上动态配置的模式名称集。

另请参阅

使用声明性表的显式模式名称 - 使用 ORM 声明性表配置时的模式名称规范

最基本的示例是使用 Core `Table`对象的`Table.schema`参数，如下所示：

```py
metadata_obj = MetaData()

financial_info = Table(
    "financial_info",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    Column("value", String(100), nullable=False),
    schema="remote_banks",
)
```

使用此`Table`渲染的 SQL，比如下面的 SELECT 语句，将明确限定表名`financial_info`与`remote_banks`模式名一起使用：

```py
>>> print(select(financial_info))
SELECT  remote_banks.financial_info.id,  remote_banks.financial_info.value
FROM  remote_banks.financial_info 
```

当使用显式模式名称声明`Table`对象时，它将使用模式和表名称的组合存储在内部`MetaData`命名空间中。我们可以在`MetaData.tables`集合中查找键`'remote_banks.financial_info'`以查看这一点：

```py
>>> metadata_obj.tables["remote_banks.financial_info"]
Table('financial_info', MetaData(),
Column('id', Integer(), table=<financial_info>, primary_key=True, nullable=False),
Column('value', String(length=100), table=<financial_info>, nullable=False),
schema='remote_banks')
```

即使引用表时也必须使用此点名，以便与 `ForeignKey` 或 `ForeignKeyConstraint` 对象一起使用，即使引用表也在同一个模式中：

```py
customer = Table(
    "customer",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    Column("financial_info_id", ForeignKey("remote_banks.financial_info.id")),
    schema="remote_banks",
)
```

在某些方言中，也可以使用 `Table.schema` 参数指定到达特定表的多令牌（例如，点分）路径。这在诸如 Microsoft SQL Server 这样的数据库上特别重要，因为通常会有点分的 “数据库/所有者” 令牌。可以一次直接将令牌放在名称中，例如：

```py
schema = "dbo.scott"
```

另请参阅

多部分模式名称 - 描述了在 SQL Server 方言中使用点分模式名称的情况。

从其他模式反映表

### 使用 MetaData 指定默认模式名称

`MetaData` 对象也可以通过将 `MetaData.schema` 参数传递给顶级 `MetaData` 结构来设置所有 `Table.schema` 参数的显式默认选项：

```py
metadata_obj = MetaData(schema="remote_banks")

financial_info = Table(
    "financial_info",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    Column("value", String(100), nullable=False),
)
```

以上，对于任何将 `Table.schema` 参数保留在其默认值 `None` 的 `Table` 对象（或直接与 `MetaData` 关联的 `Sequence` 对象），将会像参数设置为值 `"remote_banks"` 一样。包括，`Table` 在 `MetaData` 中以模式限定名称进行分类，即：

```py
metadata_obj.tables["remote_banks.financial_info"]
```

当使用 `ForeignKey` 或 `ForeignKeyConstraint` 对象引用此表时，可以使用模式限定名称或非模式限定名称来引用 `remote_banks.financial_info` 表：

```py
# either will work:

refers_to_financial_info = Table(
    "refers_to_financial_info",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    Column("fiid", ForeignKey("financial_info.id")),
)

# or

refers_to_financial_info = Table(
    "refers_to_financial_info",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    Column("fiid", ForeignKey("remote_banks.financial_info.id")),
)
```

当使用设置了 `MetaData.schema` 的 `MetaData` 对象时，希望指定不应该被模式限定的 `Table` 可以使用特殊符号 `BLANK_SCHEMA`：

```py
from sqlalchemy import BLANK_SCHEMA

metadata_obj = MetaData(schema="remote_banks")

financial_info = Table(
    "financial_info",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    Column("value", String(100), nullable=False),
    schema=BLANK_SCHEMA,  # will not use "remote_banks"
)
```

另请参阅

`MetaData.schema`  ### 应用动态模式命名约定

`Table.schema` 参数使用的名称也可以针对每个连接或每个执行基础上的动态查找应用，因此例如在多租户情况下，每个事务或语句可以针对一组动态变化的模式名称。模式名称的翻译 部分描述了此功能的使用方式。

另请参阅

模式名称的翻译  ### 为新连接设置默认模式

上述方法都是指在 SQL 语句中包含显式模式名称的方法。实际上，数据库连接具有“默认”模式的概念，这是在表名未明确指定模式的情况下发生的“模式”（或数据库，所有者等）的名称。这些名称通常在登录级别配置，例如，连接到 PostgreSQL 数据库时，默认的“模式”称为 “public”。

通常存在无法通过登录本身设置默认 “模式” 的情况，而是在每次建立连接时有用地配置的情况，例如在 PostgreSQL 上使用类似于 “SET SEARCH_PATH” 的语句或在 Oracle 上使用 “ALTER SESSION”。可以通过使用 `PoolEvents.connect()` 事件来实现这些方法，该事件允许在首次创建时访问 DBAPI 连接。例如，将 Oracle CURRENT_SCHEMA 变量设置为备用名称：

```py
from sqlalchemy import event
from sqlalchemy import create_engine

engine = create_engine("oracle+cx_oracle://scott:tiger@tsn_name")

@event.listens_for(engine, "connect", insert=True)
def set_current_schema(dbapi_connection, connection_record):
    cursor_obj = dbapi_connection.cursor()
    cursor_obj.execute("ALTER SESSION SET CURRENT_SCHEMA=%s" % schema_name)
    cursor_obj.close()
```

上述 `set_current_schema()` 事件处理程序将在上述 `Engine` 首次连接时立即发生；由于该事件被 “插入” 到处理程序列表的开头，因此它将在方言自身的事件处理程序运行之前发生，特别是包括确定连接的 “默认模式” 的处理程序。

对于其他数据库，请参阅数据库和/或方言文档，以获取有关如何配置默认模式的具体信息。

从版本 1.4.0b2 开始更改：上述方法现在无需建立额外的事件处理程序即可工作。

另请参阅

在连接时设置替代搜索路径 - 在 PostgreSQL 方言文档中。

### 模式和反射

SQLAlchemy 的模式特性与 反射数据库对象 中介绍的表反射特性相互作用。有关此工作原理的详细信息，请参阅 从其他模式反射表 部分。

### 使用 MetaData 指定默认模式名称

`MetaData`对象还可以通过将`MetaData.schema`参数传递给顶层`MetaData`构造来为所有`Table.schema`参数设置显式默认选项：

```py
metadata_obj = MetaData(schema="remote_banks")

financial_info = Table(
    "financial_info",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    Column("value", String(100), nullable=False),
)
```

对于任何`Table`对象（或直接与`MetaData`相关联的`Sequence`对象），如果将`Table.schema`参数保留在默认值`None`，则会自动将参数视为值`"remote_banks"`。这包括`Table`在`MetaData`中以模式限定名称进行目录化，即：

```py
metadata_obj.tables["remote_banks.financial_info"]
```

当使用`ForeignKey`或`ForeignKeyConstraint`对象引用此表时，可以使用模式限定名称或非模式限定名称来引用`remote_banks.financial_info`表：

```py
# either will work:

refers_to_financial_info = Table(
    "refers_to_financial_info",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    Column("fiid", ForeignKey("financial_info.id")),
)

# or

refers_to_financial_info = Table(
    "refers_to_financial_info",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    Column("fiid", ForeignKey("remote_banks.financial_info.id")),
)
```

当使用设置了`MetaData.schema`的`MetaData`对象时，希望指定不应以模式限定方式命名的`Table`可以使用特殊符号`BLANK_SCHEMA`：

```py
from sqlalchemy import BLANK_SCHEMA

metadata_obj = MetaData(schema="remote_banks")

financial_info = Table(
    "financial_info",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    Column("value", String(100), nullable=False),
    schema=BLANK_SCHEMA,  # will not use "remote_banks"
)
```

另请参阅

`MetaData.schema`

### 应用动态模式命名约定

`Table.schema`参数使用的名称也可以根据每个连接或每次执行动态应用于查找，因此例如在多租户情况下，每个事务或语句可以针对一组不断变化的模式名称。章节模式名称的翻译描述了如何使用此功能。

另请参阅

模式名称的翻译

### 为新连接设置默认模式

上述方法都涉及在 SQL 语句中包含显式模式名称的方法。数据库连接实际上具有“默认”模式的概念，这是如果表名未显式指定模式限定符，则会发生的“模式”（或数据库、所有者等）的名称。这些名称通常在登录级别配置，例如，连接到 PostgreSQL 数据库时，默认的“模式”称为“public”。

通常存在无法通过登录本身设置默认“模式”的情况，而应在每次连接时有用地进行配置，例如，在 PostgreSQL 上使用类似 “SET SEARCH_PATH” 的语句或在 Oracle 上使用 “ALTER SESSION”。这些方法可以通过使用 `PoolEvents.connect()` 事件来实现，该事件允许在首次创建 DBAPI 连接时访问它。例如，将 Oracle 的 CURRENT_SCHEMA 变量设置为替代名称：

```py
from sqlalchemy import event
from sqlalchemy import create_engine

engine = create_engine("oracle+cx_oracle://scott:tiger@tsn_name")

@event.listens_for(engine, "connect", insert=True)
def set_current_schema(dbapi_connection, connection_record):
    cursor_obj = dbapi_connection.cursor()
    cursor_obj.execute("ALTER SESSION SET CURRENT_SCHEMA=%s" % schema_name)
    cursor_obj.close()
```

上述 `set_current_schema()` 事件处理程序将在上述 `Engine` 首次连接时立即发生；由于该事件被“插入”到处理程序列表的开头，因此它也将在方言自己的事件处理程序之前发生，特别是包括将为连接确定“默认模式”的事件处理程序。

对于其他数据库，请查阅数据库和/或方言文档，以获取有关如何配置默认模式的具体信息。

在版本 1.4.0b2 中更改：上述配方现在无需建立额外的事件处理程序即可工作。

另请参阅

在连接时设置替代搜索路径 - 参见 PostgreSQL 方言文档。

### 模式和反射

SQLAlchemy 的模式特性与引入的表反射特性交互 Reflecting Database Objects。有关此工作原理的其他详细信息，请参阅 Reflecting Tables from Other Schemas 部分。

## 特定于后端的选项

`Table` 支持特定于数据库的选项。例如，MySQL 有不同的表后端类型，包括“MyISAM”和“InnoDB”。这可以通过 `Table` 使用 `mysql_engine` 来表示：

```py
addresses = Table(
    "engine_email_addresses",
    metadata_obj,
    Column("address_id", Integer, primary_key=True),
    Column("remote_user_id", Integer, ForeignKey(users.c.user_id)),
    Column("email_address", String(20)),
    mysql_engine="InnoDB",
)
```

其他后端可能也支持表级选项 - 这些将在每个方言的单独文档部分中描述。

## 列、表、MetaData API

| 对象名称 | 描述 |
| --- | --- |
| Column | 表示数据库表中的列。 |
| insert_sentinel([name, type_], *, [default, omit_from_statements]) | 提供一个代理 `Column`，它将作为专用的插入 sentinel 列，允许对没有合格的主键配置的表进行高效的批量插入，并且对返回排序具有确定性。 |
| MetaData | 一组`Table`对象及其相关的模式构造。 |
| SchemaConst | 一个枚举。 |
| SchemaItem | 定义数据库模式的项目的基类。 |
| Table | 在数据库中表示一个表。 |

```py
attribute sqlalchemy.schema.BLANK_SCHEMA
```

指的是 `SchemaConst.BLANK_SCHEMA`.

```py
attribute sqlalchemy.schema.RETAIN_SCHEMA
```

指的是 `SchemaConst.RETAIN_SCHEMA`

```py
class sqlalchemy.schema.Column
```

表示数据库表中的列。

**成员**

__eq__(), __init__(), __le__(), __lt__(), __ne__(), all_(), anon_key_label, anon_label, any_(), argument_for(), asc(), between(), bitwise_and(), bitwise_lshift(), bitwise_not(), bitwise_or(), bitwise_rshift(), bitwise_xor(), bool_op(), cast(), collate(), compare(), compile(), concat(), contains(), copy(), desc(), dialect_kwargs, dialect_options, distinct(), endswith(), expression, foreign_keys, get_children(), icontains(), iendswith(), ilike(), in_(), index, info, inherit_cache, is_(), is_distinct_from(), is_not(), is_not_distinct_from(), isnot(), isnot_distinct_from(), istartswith(), key, kwargs, label(), like(), match(), not_ilike(), not_in(), not_like(), notilike(), notin_(), notlike(), nulls_first(), nulls_last(), nullsfirst(), nullslast(), op(), operate(), params(), proxy_set, references(), regexp_match(), regexp_replace(), reverse_operate(), self_group(), shares_lineage(), startswith(), timetuple, unique, unique_params()

**类签名**

类`sqlalchemy.schema.Column` (`sqlalchemy.sql.base.DialectKWArgs`, `sqlalchemy.schema.SchemaItem`, `sqlalchemy.sql.expression.ColumnClause`)

```py
method __eq__(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators` *类的* `sqlalchemy.sql.expression.ColumnOperators.__eq__` *方法*

实现`==`运算符。

在列上下文中，生成子句`a = b`。如果目标为`None`，则生成`a IS NULL`。

```py
method __init__(_Column__name_pos: str | _TypeEngineArgument[_T] | SchemaEventTarget | None = None, _Column__type_pos: _TypeEngineArgument[_T] | SchemaEventTarget | None = None, *args: SchemaEventTarget, name: str | None = None, type_: _TypeEngineArgument[_T] | None = None, autoincrement: _AutoIncrementType = 'auto', default: Any | None = None, doc: str | None = None, key: str | None = None, index: bool | None = None, unique: bool | None = None, info: _InfoType | None = None, nullable: bool | Literal[SchemaConst.NULL_UNSPECIFIED] | None = SchemaConst.NULL_UNSPECIFIED, onupdate: Any | None = None, primary_key: bool = False, server_default: _ServerDefaultArgument | None = None, server_onupdate: FetchedValue | None = None, quote: bool | None = None, system: bool = False, comment: str | None = None, insert_sentinel: bool = False, _omit_from_statements: bool = False, _proxies: Any | None = None, **dialect_kwargs: Any)
```

构造一个新的`Column`对象。

参数：

+   `name` –

    此列在数据库中表示的名称。此参数可以是第一个位置参数，也可以通过关键字指定。

    不包含大写字符的名称将被视为不区分大小写的名称，并且除非它们是保留字，否则不会被引用。包含任意数量大写字符的名称将被引用并且原样发送。请注意，即使对于标准化大写名称为不区分大小写的数据库（如 Oracle）也适用此行为。

    可以在构造时省略名称字段，并在任何时候在列与`Table`关联之前应用。这是为了支持在`declarative`扩展中方便的使用。

+   `type_` –

    列的类型，使用一个子类化了`TypeEngine`的实例指示。如果类型不需要任何参数，则也可以发送类型的类，例如：

    ```py
    # use a type with arguments
    Column('data', String(50))

    # use no arguments
    Column('level', Integer)
    ```

    `type`参数可以是第二个位置参数，也可以通过关键字指定。

    如果`type`为`None`或被省略，它将首先默认为特殊类型`NullType`。如果此`Column`通过`ForeignKey`和/或`ForeignKeyConstraint`参考到另一个列，并且在该外键被解析为远程`Column`对象的时刻，远程引用列的类型也将被复制到此列。

+   `*args` – 附加的位置参数包括各种派生自`SchemaItem`的构造，这些构造将被应用为列的选项。这些包括`Constraint`、`ForeignKey`、`ColumnDefault`、`Sequence`、`Computed`和`Identity`的实例。在某些情况下，还可以使用等效的关键字参数，如`server_default`、`default`和`unique`。

+   `autoincrement` –

    设置“自动递增”语义，用于**没有外键依赖的整数主键列**（详见本文档字符串后面的更具体定义）。这可能会影响在创建表时为该列发出的 DDL，以及编译和执行 INSERT 语句时该列的考虑方式。

    默认值为字符串`"auto"`，表示应自动为具有整数类型且没有其他客户端或服务器端默认构造的单列（即非复合）主键接收自动递增语义。其他值包括`True`（强制此列对于复合主键也具有自动递增语义），`False`（此列不应具有自动递增语义），以及字符串`"ignore_fk"`（外键列的特殊情况，请参见下文）。

    术语“自动递增语义”既涉及在 CREATE TABLE 语句中为列发出的 DDL 的类型，当调用诸如`MetaData.create_all()`和`Table.create()`之类的方法时，也涉及编译和发出 INSERT 语句到数据库时该列的考虑方式：

    +   `DDL 渲染`（即 `MetaData.create_all()`、`Table.create()` 等）：当应用于没有与之关联的其他默认生成构造（例如 `Sequence` 或 `Identity` 构造）的 `Column` 时，该参数将暗示应该还呈现数据库特定关键字，例如 PostgreSQL 的 `SERIAL`、MySQL 的 `AUTO_INCREMENT` 或 SQL Server 上的 `IDENTITY`。并非每个数据库后端都有“暗示”的默认生成器可用；例如 Oracle 后端总是需要在 `Column` 中包含一个明确的构造（如 `Identity`）才能使 DDL 渲染中包括自动生成构造也被生成到数据库中。

    +   `INSERT` 语义（即当 `insert()` 构造编译为 SQL 字符串并使用 `Connection.execute()` 或等效方法在数据库上执行时）：单行 `INSERT` 语句将会自动为该列生成一个新的整数主键值，该值可在语句调用后通过 `Result` 对象上的 `CursorResult.inserted_primary_key` 属性访问。当 ORM 将 ORM 映射对象持久化到数据库时，该行为也适用，表明一个新的整数主键将可用于成为该对象的 标识键 的一部分。此行为与 `Column` 关联的 DDL 构造无关，且独立于上述前一注中讨论的“DDL 渲染”行为。

    可以将参数设置为 `True`，表示复合（即多列）主键的一部分的列应具有自动增量语义，但请注意，主键中只有一列可以具有此设置。也可以将其设置为 `True`，表示在客户端或服务器端配置了默认值的列应具有自动增量语义，但请注意，并非所有方言都能适应所有风格的默认值作为“自增”。也可以在数据类型为 INTEGER 的单列主键上将其设置为 `False`，以禁用该列的自动增量语义。

    *仅仅*对以下列有效：

    +   衍生整数（即 INT、SMALLINT、BIGINT）。

    +   是主键的一部分

    +   不通过 `ForeignKey` 引用另一列，除非指定值为 `'ignore_fk'`：

        ```py
        # turn on autoincrement for this column despite
        # the ForeignKey()
        Column('id', ForeignKey('other.id'),
                    primary_key=True, autoincrement='ignore_fk')
        ```

    在引用其他列的列上启用“自增”通常是不可取的，因为这样的列需要引用来自其他地方的值。

    对满足上述条件的列有以下影响：

    +   对于列发出 DDL，如果列尚未包含后端支持的默认生成结构，如 `Identity`，则会包含特定于数据库的关键字，以表示此列为特定后端的“自增”列。主要 SQLAlchemy 方言的行为包括：

        +   在 MySQL 和 MariaDB 上的 AUTO INCREMENT

        +   在 PostgreSQL 上的 SERIAL

        +   在 MS-SQL 上的 IDENTITY - 这甚至在没有 `Identity` 结构的情况下也会发生，因为 `Column.autoincrement` 参数早于此结构。

        +   SQLite - SQLite 整数主键列隐式“自动增长”，不需要添加额外的关键词；不包括特殊的 SQLite 关键词 `AUTOINCREMENT`，因为这是不必要的，也不被数据库厂商推荐。更多背景信息请参阅 SQLite Auto Incrementing Behavior 章节。

        +   Oracle - 目前 Oracle 方言没有默认的“自增”功能可用，推荐使用 `Identity` 结构来实现此功能（也可以使用 `Sequence` 结构）。

        +   第三方方言 - 请查阅这些方言的文档，了解其特定行为。

    +   当编译和执行单行`insert()`构造时，如果没有设置`Insert.inline()`修饰符，此列的新生成的主键值将在语句执行时自动通过特定于正在使用的数据库驱动程序的方法检索：

        +   MySQL，SQLite - 调用`cursor.lastrowid()`（参见[`www.python.org/dev/peps/pep-0249/#lastrowid`](https://www.python.org/dev/peps/pep-0249/#lastrowid)）

        +   PostgreSQL，SQL Server，Oracle - 在渲染 INSERT 语句时使用 RETURNING 或等效构造，然后在执行后检索新生成的主键值

        +   对于将`Table`对象的`Table.implicit_returning`设置为 False 的 PostgreSQL，Oracle - 仅对于`Sequence`，在执行 INSERT 语句之前显式调用`Sequence`，以便新生成的主键值可供客户端使用

        +   对于将`Table`对象的`Table.implicit_returning`设置为 False 的 SQL Server - 在调用 INSERT 语句后使用`SELECT scope_identity()`构造来检索新生成的主键值。

        +   第三方方言 - 请查阅这些方言的文档，了解它们特定行为的详细信息。

    +   对于使用参数列表（即“executemany”语义）调用的多行`insert()`构造，通常会禁用主键检索行为，但可能有特殊的 API 可用于检索“executemany”的新主键值列表，例如 psycopg2 的“fast insertmany”功能。这些功能非常新，可能尚未在文档中充分介绍。

+   `default` -

    表示此列的*默认值*的标量、Python 可调用对象或`ColumnElement`表达式，如果此列在插入的 VALUES 子句中未指定，则将在插入时调用。这是使用`ColumnDefault`作为位置参数的快捷方式；请参阅该类以获取有关参数结构的完整详细信息。

    与`Column.server_default`相对的是在数据库端创建默认生成器的默认值。

    另请参阅

    列 INSERT/UPDATE 默认值

+   `doc` – 可选字符串，可由 ORM 或类似的东西用于文档化 Python 端的属性。此属性不会渲染 SQL 注释；用于此目的的是`Column.comment`参数。

+   `key` – 一个可选的字符串标识符，将在`Table`上识别此`Column`对象。当提供了一个 key 时，这是应用程序中唯一引用`Column`的标识符，包括 ORM 属性映射；`name`字段仅在渲染 SQL 时使用。

+   `index` –

    当`True`时，表示将为此`Column`自动生成一个`Index`构造，这将导致在调用 DDL 创建操作时为`Table`发出“CREATE INDEX”语句。

    使用此标志等同于在`Table`构造本身的级别上显式使用`Index`构造：

    ```py
    Table(
        "some_table",
        metadata,
        Column("x", Integer),
        Index("ix_some_table_x", "x")
    )
    ```

    要将`Index.unique`标志添加到`Index`中，同时将`Column.unique`和`Column.index`标志设置为 True，这将导致渲染“CREATE UNIQUE INDEX”DDL 指令而不是“CREATE INDEX”。

    索引的名称使用默认命名约定，对于`Index`构造，其形式为`ix_<tablename>_<columnname>`。

    由于此标志仅用作向表定义添加单列默认配置索引的常见情况的便利性，因此对于大多数用例，包括跨多列的复合索引、具有 SQL 表达式或排序的索引、特定于后端的索引配置选项以及使用特定名称的索引，应首选显式使用`Index`构造。

    注意

    `Column.index` 属性在 `Column` 上 **并不表示** 此列是否已建立索引，只表示是否在此处显式设置了此标志。要查看列上的索引，请查看 `Table.indexes` 集合或使用 `Inspector.get_indexes()`。

    另请参阅

    索引

    配置约束命名规范

    `Column.unique` 

+   `info` – 可选数据字典，将填充到此对象的 `SchemaItem.info` 属性中。

+   `nullable` –

    当设置为 `False` 时，将在生成列的 DDL 时添加“NOT NULL”短语。当设置为 `True` 时，通常不生成任何内容（在 SQL 中默认为“NULL”），除非在一些非常特定的后端特定情况下，“NULL”可能会被显式渲染。默认为 `True`，除非 `Column.primary_key` 也为 `True` 或列指定了 `Identity`，在这种情况下默认为 `False`。此参数仅在发出 CREATE TABLE 语句时使用。

    注意

    当列指定了 `Identity` 时，DDL 编译器通常会忽略此参数。PostgreSQL 数据库允许通过将此参数显式设置为 `True` 来设置可空标识列。

+   `onupdate` –

    表示要应用于 UPDATE 语句中的列的默认值的标量、Python 可调用对象或 `ClauseElement`。如果该列在 UPDATE 的 SET 子句中不存在，将在更新时调用此默认值。这是使用 `ColumnDefault` 作为 `for_update=True` 的位置参数的一种捷径。

    另请参阅

    列的 INSERT/UPDATE 默认值 - 对`onupdate`的完整讨论

+   `primary_key` – 如果设置为`True`，将该列标记为主键列。可以设置多个列具有此标志以指定复合主键。作为替代，可以通过显式的 `PrimaryKeyConstraint` 对象来指定 `Table` 的主键。

+   `server_default` –

    一个`FetchedValue`实例，str，Unicode 或`text()`构造，表示列的 DDL DEFAULT 值。

    字符串类型将按原样输出，用单引号括起来：

    ```py
    Column('x', Text, server_default="val")

    x TEXT DEFAULT 'val'
    ```

    `text()`表达式将按原样呈现，不带引号：

    ```py
    Column('y', DateTime, server_default=text('NOW()'))

    y DATETIME DEFAULT NOW()
    ```

    字符串和 text()将在初始化时转换为`DefaultClause`对象。

    此参数还可以接受上下文有效的 SQLAlchemy 表达式或构造的复杂组合：

    ```py
    from sqlalchemy import create_engine
    from sqlalchemy import Table, Column, MetaData, ARRAY, Text
    from sqlalchemy.dialects.postgresql import array

    engine = create_engine(
        'postgresql+psycopg2://scott:tiger@localhost/mydatabase'
    )
    metadata_obj = MetaData()
    tbl = Table(
            "foo",
            metadata_obj,
            Column("bar",
                   ARRAY(Text),
                   server_default=array(["biz", "bang", "bash"])
                   )
    )
    metadata_obj.create_all(engine)
    ```

    以上结果将创建一个使用以下 SQL 创建的表：

    ```py
    CREATE TABLE foo (
        bar TEXT[] DEFAULT ARRAY['biz', 'bang', 'bash']
    )
    ```

    使用`FetchedValue`表示已经存在的列将在数据库端生成默认值，该值将在插入后可供 SQLAlchemy 后获取。此构造不指定任何 DDL，实现留给数据库，例如通过触发器。

    另请参见

    服务器调用的 DDL-显式默认表达式 - 有关服务器端默认值的完整讨论

+   `server_onupdate` –

    一个`FetchedValue`实例，表示数据库端的默认生成函数，例如触发器。这告诉 SQLAlchemy 在更新后将可用新生成的值。此构造实际上不在数据库中实现任何生成函数，而必须单独指定。

    警告

    此指令**目前不会**生成 MySQL 的“ON UPDATE CURRENT_TIMESTAMP()”子句。请参阅为 MySQL / MariaDB 的 explicit_defaults_for_timestamp 呈现 ON UPDATE CURRENT TIMESTAMP 以了解如何生成此子句的背景信息。

    另请参见

    标记隐式生成的值、时间戳和触发列

+   `quote` – 强制引用此列名，对应`True`或`False`。当保持默认值`None`时，列标识符将根据名称是否区分大小写（至少有一个大写字符的标识符被视为区分大小写），或者是否为保留字来引用。只有在需要强制引用 SQLAlchemy 方言不知道的保留字时才需要此标志。

+   `unique` –

    当为`True`时，并且`Column.index`参数保持其默认值为`False`，表示将为此`Column`自动生成一个`UniqueConstraint`构造，这将导致在调用`Table`对象的 DDL 创建操作时，包含引用此列的“UNIQUE CONSTRAINT”子句的`CREATE TABLE`语句被发出。

    当此标志设置为`True`时，同时`Column.index`参数也设置为`True`，则效果是生成一个包含`Index.unique`参数设置为`True`的`Index`构造。有关更多详细信息，请参阅`Column.index`的文档。

    使用此标志等效于在`Table`构造本身的级别上显式使用`UniqueConstraint`构造：

    ```py
    Table(
        "some_table",
        metadata,
        Column("x", Integer),
        UniqueConstraint("x")
    )
    ```

    唯一约束对象的`UniqueConstraint.name`参数保持其默认值为`None`；在缺乏对封闭的`MetaData`的命名约定的情况下，唯一约束构造将被发出为未命名的，这通常会触发特定于数据库的命名约定。

    由于此标志仅用作向表定义添加单列、默认配置的唯一约束的常见情况的便利性，因此在大多数用例中，应优先使用显式使用`UniqueConstraint`构造，包括涵盖多个列的复合约束、特定于后端的索引配置选项以及使用特定名称的约束。

    注意

    `Column.unique` 属性在 `Column` 上**不表示**此列是否具有唯一约束，只表示此标志是否在此处被显式设置。要查看可能涉及此列的索引和唯一约束，请查看 `Table.indexes` 和/或 `Table.constraints` 集合，或使用 `Inspector.get_indexes()` 和/或 `Inspector.get_unique_constraints()`

    另请参阅

    唯一约束

    配置约束命名规范

    `Column.index`

+   `system` –

    当为 `True` 时，表示这是一个“系统”列，即数据库自动提供的列，不应包含在 `CREATE TABLE` 语句的列列表中。

    对于更加复杂的情景，在不同的后端条件下应该以不同方式渲染列的情况，考虑为`CreateColumn`自定义编译规则。

+   `comment` –

    可选字符串，将在表创建时渲染为 SQL 注释。

    版本 1.2 中的新增内容：增加了 `Column.comment` 参数到 `Column`。

+   `insert_sentinel` –

    将此 `Column` 标记为用于优化对于没有其他符合条件的主键配置的表的 insertmanyvalues 功能性能的插入标志。

    版本 2.0.10 中的新增内容。

    另请参阅

    `insert_sentinel()` - 一站式助手，用于声明标志列

    对于 INSERT 语句的“插入多个值”行为

    配置标志列

```py
method __le__(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators` *的* `sqlalchemy.sql.expression.ColumnOperators.__le__` *方法*

实现 `<=` 运算符。

在列上下文中，生成子句 `a <= b`。

```py
method __lt__(other: Any) → ColumnOperators
```

*继承自* `sqlalchemy.sql.expression.ColumnOperators.__lt__` *方法的* `ColumnOperators`

实现 `<` 运算符。

在列的上下文中，生成子句 `a < b`。

```py
method __ne__(other: Any) → ColumnOperators
```

*继承自* `sqlalchemy.sql.expression.ColumnOperators.__ne__` *方法的* `ColumnOperators`

实现 `!=` 运算符。

在列的上下文中，生成子句 `a != b`。如果目标是 `None`，则生成 `a IS NOT NULL`。

```py
method all_() → ColumnOperators
```

*继承自* `ColumnOperators.all_()` *方法的* `ColumnOperators` *属性*

针对父对象生成一个 `all_()` 子句。

请参阅 `all_()` 的文档以获取示例。

注意

请确保不要将较新的 `ColumnOperators.all_()` 方法与此方法的 **传统** 版本混淆，即专用于 `ARRAY` 的 `Comparator.all()` 方法，其使用不同的调用风格。

```py
attribute anon_key_label
```

*继承自* `ColumnElement.anon_key_label` *属性的* `ColumnElement`

自版本 1.4 弃用：`ColumnElement.anon_key_label` 属性现已私有化，公共访问器已弃用。

```py
attribute anon_label
```

*继承自* `ColumnElement.anon_label` *属性的* `ColumnElement`

自版本 1.4 弃用：`ColumnElement.anon_label` 属性现已私有化，公共访问器已弃用。

```py
method any_() → ColumnOperators
```

*继承自* `ColumnOperators.any_()` *方法的* `ColumnOperators`

生成针对父对象的 `any_()` 子句。

请参阅 `any_()` 的文档以获取示例。

注意

一定不要将新版本的 `ColumnOperators.any_()` 方法与**旧版**方法混淆，旧版方法是专用于 `ARRAY` 的 `Comparator.any()` 方法，其调用风格不同。

```py
classmethod argument_for(dialect_name, argument_name, default)
```

*继承自* `DialectKWArgs.argument_for()` *方法的* `DialectKWArgs`

为这个类添加一种新的方言特定的关键字参数。

例如：

```py
Index.argument_for("mydialect", "length", None)

some_index = Index('a', 'b', mydialect_length=5)
```

`DialectKWArgs.argument_for()` 方法是一种逐个添加额外参数到 `DefaultDialect.construct_arguments` 字典的方式。该字典为方言代表提供了一组被各种模式级构造接受的参数名称。

新的方言通常应该一次性将此字典作为方言类的数据成员来指定。临时添加参数名称的用例通常是针对终端用户代码的，该代码还使用了消耗附加参数的自定义编译方案。

参数：

+   `dialect_name` – 方言的名称。方言必须是可定位的，否则会引发 `NoSuchModuleError`。方言还必须包括一个现有的 `DefaultDialect.construct_arguments` 集合，表示它参与关键字参数验证和默认系统，否则会引发 `ArgumentError`。如果方言不包括此集合，则可以代表该方言已经指定任何关键字参数。SQLAlchemy 内置的所有方言都包含此集合，但是对于第三方方言，支持可能有所不同。

+   `argument_name` – 参数的名称。

+   `default` – 参数的默认值。

```py
method asc() → ColumnOperators
```

*继承自* `ColumnOperators.asc()` *方法的* `ColumnOperators`

生成一个针对父对象的 `asc()` 子句。

```py
method between(cleft: Any, cright: Any, symmetric: bool = False) → ColumnOperators
```

*继承自* `ColumnOperators.between()` *方法的* `ColumnOperators`

生成一个针对父对象的 `between()` 子句，给定下限和上限。

```py
method bitwise_and(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_and()` *方法的* `ColumnOperators`

执行按位 AND 操作，通常通过 `&` 运算符。

新版本 2.0.2 中新增。

另请参阅

位运算符

```py
method bitwise_lshift(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_lshift()` *方法的* `ColumnOperators`

生成一个按位 LSHIFT 操作，通常通过 `<<` 运算符。

新版本 2.0.2 中新增。

另请参阅

位运算符

```py
method bitwise_not() → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_not()` *方法的* `ColumnOperators`

执行按位非操作，通常通过 `~` 运算符。

版本 2.0.2 中的新功能。

另请参阅

位运算符

```py
method bitwise_or(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_or()` *方法的* `ColumnOperators`

执行按位或操作，通常通过 `|` 运算符。

版本 2.0.2 中的新功能。

另请参阅

位运算符

```py
method bitwise_rshift(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_rshift()` *方法的* `ColumnOperators`

执行按位右移操作，通常通过 `>>` 运算符。

版本 2.0.2 中的新功能。

另请参阅

位运算符

```py
method bitwise_xor(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_xor()` *方法的* `ColumnOperators`

执行按位异或操作，通常通过 `^` 运算符，或 PostgreSQL 中的 `#` 运算符。

版本 2.0.2 中的新功能。

另请参阅

位运算符

```py
method bool_op(opstring: str, precedence: int = 0, python_impl: Callable[[...], Any] | None = None) → Callable[[Any], Operators]
```

*继承自* `Operators.bool_op()` *方法的* `Operators`

返回自定义布尔运算符。

此方法是调用 `Operators.op()` 并传递 `Operators.op.is_comparison` 标志为 True 的简写。 使用 `Operators.bool_op()` 的一个关键优势是，在使用列构造时，返回表达式的“布尔”性质将出现在 [**PEP 484**](https://peps.python.org/pep-0484/) 目的中。

另请参阅

`Operators.op()`

```py
method cast(type_: _TypeEngineArgument[_OPT]) → Cast[_OPT]
```

*继承自* `ColumnElement.cast()` *方法的* `ColumnElement`

生成一个类型转换，即`CAST(<expression> AS <type>)`。

这是`cast()`函数的快捷方式。

另请参阅

数据转换和类型强制转换

`cast()`

`type_coerce()`

```py
method collate(collation: str) → ColumnOperators
```

*继承自* `ColumnOperators.collate()` *方法的* `ColumnOperators`

对父对象产生一个`collate()`子句，给定排序规则字符串。

另请参阅

`collate()`

```py
method compare(other: ClauseElement, **kw: Any) → bool
```

*继承自* `ClauseElement.compare()` *方法的* `ClauseElement`

将此`ClauseElement`与给定的`ClauseElement`进行比较。

子类应该重写默认行为，这是一个简单的身份比较。

**kw 是子类 `compare()` 方法消耗的参数，可以用来修改比较的标准（参见`ColumnElement`）。

```py
method compile(bind: _HasDialect | None = None, dialect: Dialect | None = None, **kw: Any) → Compiled
```

*继承自* `CompilerElement.compile()` *方法的* `CompilerElement`

编译这个 SQL 表达式。

返回值是一个`Compiled`对象。对返回值调用`str()`或`unicode()`将产生结果的字符串表示。`Compiled`对象还可以使用`params`访问器返回绑定参数名称和值的字典。

参数：

+   `bind` – 一个`Connection`或`Engine`，可以提供一个`Dialect`以生成一个`Compiled`对象。如果`bind`和`dialect`参数都被省略，将使用默认的 SQL 编译器。

+   `column_keys` – 用于 INSERT 和 UPDATE 语句，应该出现在编译语句的 VALUES 子句中的列名列表。如果为`None`，则从目标表对象中渲染所有列。

+   `dialect` – 一个`Dialect`实例，可以生成一个`Compiled`对象。此参数优先于`bind`参数。

+   `compile_kwargs` –

    可选的字典，其中包含将传递给所有“visit”方法中的编译器的附加参数。这允许将任何自定义标志传递给自定义编译结构，例如。它还用于通过传递`literal_binds`标志的情况：

    ```py
    from sqlalchemy.sql import table, column, select

    t = table('t', column('x'))

    s = select(t).where(t.c.x == 5)

    print(s.compile(compile_kwargs={"literal_binds": True}))
    ```

另请参见

如何将 SQL 表达式呈现为字符串，可能包含内联的绑定参数？

```py
method concat(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.concat()` *方法的* `ColumnOperators`

实现‘concat’操作符。

在列上下文中，生成子句`a || b`，或者在 MySQL 上使用`concat()`操作符。

```py
method contains(other: Any, **kw: Any) → ColumnOperators
```

*继承自* `ColumnOperators.contains()` *方法的* `ColumnOperators`

实现‘contains’操作符。

生成一个 LIKE 表达式，测试字符串值的中间匹配：

```py
column LIKE '%' || <other> || '%'
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.contains("foobar"))
```

由于操作符使用了`LIKE`，因此存在于<other>表达式中的通配符字符`"%"`和`"_"`也将像通配符一样起作用。对于字面字符串值，可以设置`ColumnOperators.contains.autoescape`标志为`True`，以对字符串值内的这些字符出现进行转义，使它们作为自身而不是通配符字符匹配。或者，`ColumnOperators.contains.escape`参数将建立给定字符作为转义字符，当目标表达式不是字面字符串时可以派上用场。

参数：

+   `other` – 要比较的表达式。通常这是一个简单的字符串值，但也可以是任意的 SQL 表达式。默认情况下，不对 LIKE 通配符字符`%`和`_`进行转义，除非设置了`ColumnOperators.contains.autoescape`标志为 True。

+   `autoescape` –

    布尔值；当为 True 时，在 LIKE 表达式中建立一个转义字符，然后将其应用于比较值内的所有`"%"`、`"_"`和转义字符本身的出现，假定比较值为字面字符串而不是 SQL 表达式。

    如下表达式：

    ```py
    somecolumn.contains("foo%bar", autoescape=True)
    ```

    渲染为：

    ```py
    somecolumn LIKE '%' || :param || '%' ESCAPE '/'
    ```

    以值`：param`为`"foo/%bar"`。

+   `escape` –

    一个字符，当给出时将使用`ESCAPE`关键字以建立该字符作为转义字符。然后，可以将此字符放置在`%`和`_`的前面，以允许它们作为自身而不是通配符字符。

    如下表达式：

    ```py
    somecolumn.contains("foo/%bar", escape="^")
    ```

    渲染为：

    ```py
    somecolumn LIKE '%' || :param || '%' ESCAPE '^'
    ```

    该参数也可以与`ColumnOperators.contains.autoescape`组合使用：

    ```py
    somecolumn.contains("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上述情况下，给定的字面参数将在传递给数据库之前转换为`"foo^%bar^^bat"`。

另请参阅

`ColumnOperators.startswith()`

`ColumnOperators.endswith()`

`ColumnOperators.like()`

```py
method copy(**kw: Any) → Column[Any]
```

自版本 1.4 起已弃用：`Column.copy()`方法已弃用，将在将来的版本中删除。

```py
method desc() → ColumnOperators
```

继承自 `ColumnOperators.desc()` *方法的* `ColumnOperators`

生成针对父对象的 `desc()` 子句。

```py
attribute dialect_kwargs
```

继承自 `DialectKWArgs.dialect_kwargs` *属性的* `DialectKWArgs`

作为特定于方言的选项指定的关键字参数的集合，用于此结构。

这里的参数以其原始 `<dialect>_<kwarg>` 格式呈现。仅包括实际传递的参数；不像 `DialectKWArgs.dialect_options` 集合，该集合包含此方言已知的所有选项，包括默认值。

集合也是可写的；接受形式为 `<dialect>_<kwarg>` 的键，其值将被组装到选项列表中。

另请参阅

`DialectKWArgs.dialect_options` - 嵌套字典形式

```py
attribute dialect_options
```

继承自 `DialectKWArgs.dialect_options` *属性的* `DialectKWArgs`

作为特定于方言的选项指定的关键字参数的集合，用于此结构。

这是一个两级嵌套的注册表，键入 `<dialect_name>` 和 `<argument_name>`。例如，`postgresql_where` 参数可以定位为：

```py
arg = my_object.dialect_options['postgresql']['where']
```

从版本 0.9.2 开始新增。

另请参阅

`DialectKWArgs.dialect_kwargs` - 平面字典形式

```py
method distinct() → ColumnOperators
```

继承自 `ColumnOperators.distinct()` *方法的* `ColumnOperators`

生成针对父对象的 `distinct()` 子句。

```py
method endswith(other: Any, escape: str | None = None, autoescape: bool = False) → ColumnOperators
```

*从* `ColumnOperators.endswith()` *方法继承的* `ColumnOperators`

实现“endswith”操作符。

生成一个 LIKE 表达式，用于测试字符串值的结尾是否匹配：

```py
column LIKE '%' || <other>
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.endswith("foobar"))
```

由于操作符使用`LIKE`，因此存在于<other>表达式内部的通配符字符`"%"`和`"_"`也将像通配符一样起作用。对于文字字符串值，可以将`ColumnOperators.endswith.autoescape`标志设置为`True`，以将这些字符在字符串值内的出现进行转义，使它们匹配为自己而不是通配符字符。或者，`ColumnOperators.endswith.escape`参数将建立一个给定的字符作为转义字符，这在目标表达式不是文字字符串时可能会有用。

参数：

+   `other` – 要比较的表达式。这通常是一个纯文字字符串值，但也可以是任意的 SQL 表达式。默认情况下，LIKE 通配符字符`%`和`_`不会被转义，除非将`ColumnOperators.endswith.autoescape`标志设置为 True。

+   `autoescape` –

    boolean；当为 True 时，在 LIKE 表达式中建立一个转义字符，然后将其应用于比较值内所有出现的`"%"`、`"_"`和转义字符本身，假定该值为文字字符串而不是 SQL 表达式。

    一个表达式，例如：

    ```py
    somecolumn.endswith("foo%bar", autoescape=True)
    ```

    将呈现为：

    ```py
    somecolumn LIKE '%' || :param ESCAPE '/'
    ```

    使用`param`的值为`"foo/%bar"`。

+   `escape` –

    一个字符，当给定时，将使用`ESCAPE`关键字将该字符确定为转义字符。然后可以将此字符放在`%`和`_`的前面，以允许它们作为自己而不是通配符字符。

    一个表达式，例如：

    ```py
    somecolumn.endswith("foo/%bar", escape="^")
    ```

    将呈现为：

    ```py
    somecolumn LIKE '%' || :param ESCAPE '^'
    ```

    参数也可以与`ColumnOperators.endswith.autoescape`组合使用：

    ```py
    somecolumn.endswith("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上述示例中，给定的文字参数在传递到数据库之前将转换为`"foo^%bar^^bat"`。

另请参阅

`ColumnOperators.startswith()`

`ColumnOperators.contains()`

`ColumnOperators.like()`

```py
attribute expression
```

*从* `ColumnElement.expression` *属性继承* `ColumnElement`

返回列表达式。

检查接口的一部分；返回自身。

```py
attribute foreign_keys: Set[ForeignKey] = frozenset({})
```

*从* `ColumnElement.foreign_keys` *属性继承* `ColumnElement`

所有与此 `Column` 关联的 `ForeignKey` 标记对象的集合。

每个对象都是 `Table` 范围内的 `ForeignKeyConstraint` 的成员。

另请参阅

`Table.foreign_keys`

```py
method get_children(*, column_tables=False, **kw)
```

*从* `ColumnClause.get_children()` *方法继承* `ColumnClause`

返回此 `HasTraverseInternals` 的直接子元素 `HasTraverseInternals`。

这用于访问遍历。

**kw 可以包含更改返回的集合的标志，例如以便返回子项的子集以减少较大的遍历量，或者从不同上下文（例如模式级集合而不是子句级别）返回子项。

```py
method icontains(other: Any, **kw: Any) → ColumnOperators
```

*从* `ColumnOperators.icontains()` *方法继承* `ColumnOperators`

实现 `icontains` 运算符，例如 `ColumnOperators.contains()` 的不区分大小写版本。

生成一个 LIKE 表达式，用于测试字符串值中间的不区分大小写的匹配：

```py
lower(column) LIKE '%' || lower(<other>) || '%'
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.icontains("foobar"))
```

由于该操作符使用`LIKE`，因此在<other>表达式内部存在的通配符字符`"%"`和`"_"`也将像通配符一样运行。对于字面字符串值，可以将`ColumnOperators.icontains.autoescape`标志设置为 True，以对字符串值内部这些字符的出现应用转义，使它们匹配为自身而不是通配符字符。或者，`ColumnOperators.icontains.escape`参数将建立给定字符作为转义字符，当目标表达式不是字面字符串时这可能会有用。

参数：

+   `other` – 要比较的表达式。这通常是一个简单的字符串值，但也可以是任意的 SQL 表达式。LIKE 通配符字符`%`和`_`默认情况下不会被转义，除非将`ColumnOperators.icontains.autoescape`标志设置为 True。

+   `autoescape` –

    boolean；当为 True 时，在 LIKE 表达式中建立转义字符，然后将其应用于比较值中的所有出现的`"%"`、`"_"`和转义字符本身，假定比较值是字面字符串而不是 SQL 表达式。

    例如表达式：

    ```py
    somecolumn.icontains("foo%bar", autoescape=True)
    ```

    将呈现为：

    ```py
    lower(somecolumn) LIKE '%' || lower(:param) || '%' ESCAPE '/'
    ```

    将`:param`的值设为`"foo/%bar"`。

+   `escape` –

    一个字符，当给定时，将使用`ESCAPE`关键字来建立该字符作为转义字符。然后，可以将该字符放在`%`和`_`之前，以允许它们作为自身而不是通配符字符。

    例如表达式：

    ```py
    somecolumn.icontains("foo/%bar", escape="^")
    ```

    将呈现为：

    ```py
    lower(somecolumn) LIKE '%' || lower(:param) || '%' ESCAPE '^'
    ```

    参数也可以与`ColumnOperators.contains.autoescape`结合使用：

    ```py
    somecolumn.icontains("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上述情况下，给定的字面参数将在传递到数据库之前被转换为`"foo^%bar^^bat"`。

另请参阅

`ColumnOperators.contains()`

```py
method iendswith(other: Any, escape: str | None = None, autoescape: bool = False) → ColumnOperators
```

*继承自* `ColumnOperators.iendswith()` *方法的* `ColumnOperators`

实现`iendswith`操作符，例如，`ColumnOperators.endswith()`的不区分大小写版本。

生成一个 LIKE 表达式，用于对字符串值的结尾进行大小写不敏感匹配：

```py
lower(column) LIKE '%' || lower(<other>)
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.iendswith("foobar"))
```

由于该运算符使用 `LIKE`，所以存在于 <other> 表达式内部的通配符字符 `"%"` 和 `"_"` 也会像通配符一样行为。对于字面字符串值，可以将 `ColumnOperators.iendswith.autoescape` 标志设置为 `True`，以对字符串值内部的这些字符进行转义，使它们不会被识别为通配符字符而是作为自身匹配。或者，`ColumnOperators.iendswith.escape` 参数将建立一个给定的字符作为转义字符，当目标表达式不是字面字符串时会有用。

参数：

+   `other` – 表达式用于比较。通常是一个普通字符串值，但也可以是任意的 SQL 表达式。LIKE 通配符字符 `%` 和 `_` 默认情况下不会被转义，除非 `ColumnOperators.iendswith.autoescape` 标志设置为 True。

+   `autoescape` –

    布尔值；当为 True 时，在 LIKE 表达式中建立一个转义字符，然后将其应用于比较值中的所有 `"%"`、`"_"` 和转义字符本身的出现次数，假定比较值是一个字面字符串而不是 SQL 表达式。

    一个如下的表达式：

    ```py
    somecolumn.iendswith("foo%bar", autoescape=True)
    ```

    渲染为：

    ```py
    lower(somecolumn) LIKE '%' || lower(:param) ESCAPE '/'
    ```

    使用 `:param` 参数值为 `"foo/%bar"`。

+   `escape` –

    给定一个字符，将会在 `ESCAPE` 关键字后面呈现，将该字符作为转义字符。然后可以将此字符放在 `%` 和 `_` 的前面，以允许它们作为自身而不是通配符字符。

    一个如下的表达式：

    ```py
    somecolumn.iendswith("foo/%bar", escape="^")
    ```

    渲染为：

    ```py
    lower(somecolumn) LIKE '%' || lower(:param) ESCAPE '^'
    ```

    该参数还可以与 `ColumnOperators.iendswith.autoescape` 结合使用：

    ```py
    somecolumn.endswith("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上述情况下，给定的字面参数将在传递到数据库之前转换为 `"foo^%bar^^bat"`。

另请参阅

`ColumnOperators.endswith()`

```py
method ilike(other: Any, escape: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.ilike()` *方法的* `ColumnOperators`

实现 `ilike` 运算符，例如大小写不敏感的 LIKE。

在列上下文中，生成一个形如：

```py
lower(a) LIKE lower(other)
```

或在支持 ILIKE 运算符的后端上：

```py
a ILIKE other
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.ilike("%foobar%"))
```

参数：

+   `other` – 要比较的表达式

+   `escape` –

    可选的转义字符，呈现`ESCAPE`关键字，例如：

    ```py
    somecolumn.ilike("foo/%bar", escape="/")
    ```

另请参见

`ColumnOperators.like()`

```py
method in_(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.in_()` *的* `ColumnOperators` *方法*

实现 `in` 运算符。

在列上下文中，生成子句`column IN <other>`。

给定参数`other`可以是：

+   一个字面值列表，例如：

    ```py
    stmt.where(column.in_([1, 2, 3]))
    ```

    在此调用形式中，项目列表将转换为与给定列表长度相同的一组绑定参数：

    ```py
    WHERE COL IN (?, ?, ?)
    ```

+   如果比较是针对包含多个表达式的 `tuple_()`，则可以提供元组列表：

    ```py
    from sqlalchemy import tuple_
    stmt.where(tuple_(col1, col2).in_([(1, 10), (2, 20), (3, 30)]))
    ```

+   一个空列表，例如：

    ```py
    stmt.where(column.in_([]))
    ```

    在此调用形式中，该表达式呈现一个“空集”表达式。这些表达式针对各个后端进行了定制，并且通常试图将一个空的 SELECT 语句作为子查询。例如在 SQLite 上，该表达式是：

    ```py
    WHERE col IN (SELECT 1 FROM (SELECT 1) WHERE 1!=1)
    ```

    自 1.4 版本更改：在所有情况下，空 IN 表达式现在都使用运行时生成的 SELECT 子查询。

+   如果包括 `bindparam.expanding` 标志，则可以使用一个绑定参数，例如 `bindparam()`：

    ```py
    stmt.where(column.in_(bindparam('value', expanding=True)))
    ```

    在此调用形式中，该表达式呈现一个特殊的非 SQL 占位符表达式，看起来像：

    ```py
    WHERE COL IN ([EXPANDING_value])
    ```

    此占位符表达式在语句执行时被拦截，以转换为前面所示的变量数目的绑定参数形式。如果执行语句为：

    ```py
    connection.execute(stmt, {"value": [1, 2, 3]})
    ```

    数据库将传递一个绑定参数给每个值：

    ```py
    WHERE COL IN (?, ?, ?)
    ```

    自 1.2 版本新功能：添加了“扩展”绑定参数

    如果传递了一个空列表，则呈现一个特殊的“空列表”表达式，该表达式特定于正在使用的数据库。在 SQLite 上，将会是：

    ```py
    WHERE COL IN (SELECT 1 FROM (SELECT 1) WHERE 1!=1)
    ```

    自 1.3 版本新功能：“扩展”绑定参数现在支持空列表

+   一个 `select()` 构造，通常是一个相关的标量选择：

    ```py
    stmt.where(
        column.in_(
            select(othertable.c.y).
            where(table.c.x == othertable.c.x)
        )
    )
    ```

    在此调用形式中，`ColumnOperators.in_()` 的呈现如下：

    ```py
    WHERE COL IN (SELECT othertable.y
    FROM othertable WHERE othertable.x = table.x)
    ```

参数：

**其他** – 一系列文字、`select()`构造，或包括`bindparam()`构造，其中`bindparam.expanding`标志设置为 True。

```py
attribute index: bool | None
```

`Column.index`参数的值。

不指示此`Column`是否实际上已被索引；请使用`Table.indexes`。

另请参见

`Table.indexes`的值。

```py
attribute info
```

*继承自* `SchemaItem.info` *属性的* `SchemaItem`

与该`SchemaItem`关联的信息字典，允许将用户定义的数据与此关联。

当首次访问时，字典会自动生成。也可以在某些对象的构造函数中指定，例如`Table`和`Column`。

```py
attribute inherit_cache: bool | None = True
```

指示此`HasCacheKey`实例是否应使用其直接超类使用的缓存键生成方案。

该属性默认为`None`，表示构造尚未考虑是否适合参与缓存；这在功能上等同于将值设置为`False`，只是还会发出警告。

如果与此类本地相关而不是其超类的属性不会更改与对象对应的 SQL，则可以在特定类上将此标志设置为`True`。

另请参见

为自定义构造启用缓存支持 - 设置第三方或用户定义的 SQL 构造的`HasCacheKey.inherit_cache`属性的一般指南。

```py
method is_(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.is_()` *方法的* `ColumnOperators`

实现`IS`运算符。

通常，当与`None`的值进行比较时，`IS`会自动生成，其解析为`NULL`。然而，在某些平台上，如果与布尔值进行比较，则可能希望显式使用`IS`。

另请参阅

`ColumnOperators.is_not()`

```py
method is_distinct_from(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.is_distinct_from()` *方法的* `ColumnOperators`

实现`IS DISTINCT FROM`运算符。

在大多数平台上渲染为“a IS DISTINCT FROM b”；在某些平台上，例如 SQLite，可能会渲染为“a IS NOT b”。

```py
method is_not(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.is_not()` *方法的* `ColumnOperators`

实现`IS NOT`运算符。

通常，当与`None`的值进行比较时，`IS NOT`会自动生成，其解析为`NULL`。然而，在某些平台上，如果与布尔值进行比较，则可能希望显式使用`IS NOT`。

从版本 1.4 开始：`is_not()`运算符从先前版本的`isnot()`重命名。先前的名称仍然可用于向后兼容。

另请参阅

`ColumnOperators.is_()`

```py
method is_not_distinct_from(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.is_not_distinct_from()` *方法的* `ColumnOperators`

实现`IS NOT DISTINCT FROM`运算符。

在大多数平台上渲染为“a IS NOT DISTINCT FROM b”；在某些平台上，例如 SQLite，可能会渲染为“a IS b”。

从版本 1.4 开始：`is_not_distinct_from()`运算符从先前版本的`isnot_distinct_from()`重命名。先前的名称仍然可用于向后兼容。

```py
method isnot(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.isnot()` *方法的* `ColumnOperators`

实现`IS NOT`运算符。

通常，当与`None`的值进行比较时，`IS NOT`会自动生成，其解析为`NULL`。然而，在某些平台上，如果与布尔值进行比较，则可能希望显式使用`IS NOT`。

在 1.4 版本中更改：`is_not()`运算符从之前的版本中的`isnot()`更名。之前的名称仍可用于向后兼容。

另请参阅

`ColumnOperators.is_()`

```py
method isnot_distinct_from(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.isnot_distinct_from()` *方法的* `ColumnOperators`

实现`IS NOT DISTINCT FROM`运算符。

在大多数平台上渲染为“a IS NOT DISTINCT FROM b”；在某些平台上如 SQLite 可以渲染为“a IS b”。

在 1.4 版本中更改：`is_not_distinct_from()`运算符从之前的版本中的`isnot_distinct_from()`更名。之前的名称仍可用于向后兼容。

```py
method istartswith(other: Any, escape: str | None = None, autoescape: bool = False) → ColumnOperators
```

*继承自* `ColumnOperators.istartswith()` *方法的* `ColumnOperators`

实现`istartswith`运算符，例如 `ColumnOperators.startswith()` 的不区分大小写版本。

生成一个 LIKE 表达式，用于对字符串值的起始部分进行不区分大小写的匹配：

```py
lower(column) LIKE lower(<other>) || '%'
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.istartswith("foobar"))
```

由于该运算符使用了 `LIKE`，所以在 <other> 表达式中存在的通配符字符 `"%"` 和 `"_"` 也会像通配符一样运行。对于文字字符串值，可以将 `ColumnOperators.istartswith.autoescape` 标志设置为 `True`，以对字符串值中这些字符的出现应用转义，以便它们与它们自身匹配，而不是作为通配符字符。或者，`ColumnOperators.istartswith.escape` 参数将建立一个给定字符作为转义字符，当目标表达式不是文字字符串时可以派上用场。

参数：

+   `other` – 要进行比较的表达式。通常这是一个纯字符串值，但也可以是任意的 SQL 表达式。除非设置了 `ColumnOperators.istartswith.autoescape` 标志为 True，否则默认情况下不会转义 LIKE 通配符字符 `%` 和 `_`。

+   `autoescape` –

    布尔值；当为 True 时，在 LIKE 表达式中建立转义字符，然后将其应用于比较值中的所有`"%"`、`"_"`和转义字符本身的出现，假定比较值为文字字符串而不是 SQL 表达式。

    例如：

    ```py
    somecolumn.istartswith("foo%bar", autoescape=True)
    ```

    渲染为：

    ```py
    lower(somecolumn) LIKE lower(:param) || '%' ESCAPE '/'
    ```

    值为`:param`，表示为`"foo/%bar"`。

+   `escape` –

    一个字符，当给出时，将使用`ESCAPE`关键字来将该字符作为转义字符。然后可以将此字符放置在`%`和`_`之前的位置，以允许它们作为自身而不是通配符字符。

    例如：

    ```py
    somecolumn.istartswith("foo/%bar", escape="^")
    ```

    渲染为：

    ```py
    lower(somecolumn) LIKE lower(:param) || '%' ESCAPE '^'
    ```

    该参数也可以与`ColumnOperators.istartswith.autoescape`结合使用：

    ```py
    somecolumn.istartswith("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上述情况下，给定的字面参数将在传递到数据库之前转换为`"foo^%bar^^bat"`。

另请参见

`ColumnOperators.startswith()`

```py
attribute key: str = None
```

*继承自* `ColumnElement.key` *属性的* `ColumnElement`

在某些情况下指代 Python 命名空间中的该对象的‘key’。

这通常指的是在可选择的`.c`集合中作为列的“key”存在，例如，`sometable.c["somekey"]`将返回一个具有“somekey”作为`.key`的`Column`。

```py
attribute kwargs
```

*继承自* `DialectKWArgs.kwargs` *属性的* `DialectKWArgs`

`DialectKWArgs.dialect_kwargs`的同义词。

```py
method label(name: str | None) → Label[_T]
```

*继承自* `ColumnElement.label()` *方法的* `ColumnElement`

生成一个列标签，即`<columnname> AS <name>`。

这是`label()`函数的快捷方式。

如果‘name’为`None`，将生成一个匿名标签名称。

```py
method like(other: Any, escape: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.like()` *方法的* `ColumnOperators`

实现 `like` 操作符。

在列上下文中，产生表达式：

```py
a LIKE other
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.like("%foobar%"))
```

参数：

+   `other` – 要比较的表达式

+   `escape` –

    可选转义字符，渲染 `ESCAPE` 关键字，例如：

    ```py
    somecolumn.like("foo/%bar", escape="/")
    ```

另请参见

`ColumnOperators.ilike()`

```py
method match(other: Any, **kwargs: Any) → ColumnOperators
```

*继承自* `ColumnOperators.match()` *方法的* `ColumnOperators`

实现特定于数据库的‘匹配’操作符。

`ColumnOperators.match()` 尝试解析为后端提供的 MATCH 类似函数或操作符。示例包括：

+   PostgreSQL - 渲染 `x @@ plainto_tsquery(y)`

    > 版本 2.0 中的更改：现在 PostgreSQL 使用 `plainto_tsquery()` 代替 `to_tsquery()`；有关其他形式的兼容性，请参阅全文搜索。

+   MySQL - 渲染 `MATCH (x) AGAINST (y IN BOOLEAN MODE)`

    另请参见

    `match` - 具有额外功能的 MySQL 特定构造。

+   Oracle - 渲染 `CONTAINS(x, y)`

+   其他后端可能提供特殊实现。

+   没有任何特殊实现的后端将作为“MATCH”发出操作符。例如，这与 SQLite 兼容。

```py
method not_ilike(other: Any, escape: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.not_ilike()` *方法的* `ColumnOperators`

实现 `NOT ILIKE` 操作符。

这等效于使用否定与 `ColumnOperators.ilike()`，即 `~x.ilike(y)`。

版本 1.4 中的更改：`not_ilike()` 操作符在以前的版本中从 `notilike()` 重命名。以前的名称仍可用于向后兼容。

另请参见

`ColumnOperators.ilike()`

```py
method not_in(other: Any) → ColumnOperators
```

*从* `ColumnOperators` *的* `ColumnOperators.not_in()` *方法继承*

实现`NOT IN`操作符。

这等同于在`ColumnOperators.in_()`中使用否定，即`~x.in_(y)`。

当`other`是一个空序列时，编译器会生成一个“空不在”表达式。这默认为表达式“1 = 1”，在所有情况下产生 true。`create_engine.empty_in_strategy` 可用于修改此行为。

在 1.4 版本中更改：`not_in()`操作符从先前版本的`notin_()`中改名。先前的名称仍可用于向后兼容性。

在 1.2 版本中更改：`ColumnOperators.in_()` 和 `ColumnOperators.not_in()` 操作现在默认情况下为一个空的 IN 序列生成“静态”表达式。

另请参阅

`ColumnOperators.in_()`

```py
method not_like(other: Any, escape: str | None = None) → ColumnOperators
```

*从* `ColumnOperators` *的* `ColumnOperators.not_like()` *方法继承*

实现`NOT LIKE`操作符。

这等同于在`ColumnOperators.like()`中使用否定，即`~x.like(y)`。

在 1.4 版本中更改：`not_like()`操作符从先前版本的`notlike()`中改名。先前的名称仍可用于向后兼容性。

另请参阅

`ColumnOperators.like()`

```py
method notilike(other: Any, escape: str | None = None) → ColumnOperators
```

*从* `ColumnOperators` *的* `ColumnOperators.notilike()` *方法继承*

实现`NOT ILIKE`操作符。

这等同于在`ColumnOperators.ilike()`中使用否定，即`~x.ilike(y)`。

从 1.4 版本开始更改：`not_ilike()` 操作符从先前版本的 `notilike()` 重命名为 `not_ilike()`。先前的名称仍可用于向后兼容。

另请参阅

`ColumnOperators.ilike()`

```py
method notin_(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.notin_()` *方法的* `ColumnOperators`

实现 `NOT IN` 操作符。

这相当于使用 `ColumnOperators.in_()` 时使用否定，即 `~x.in_(y)`。

在 `other` 为空序列的情况下，编译器会生成一个“空 not in” 表达式。默认情况下，这默认为表达式“1 = 1”，以在所有情况下产生 true。可以使用 `create_engine.empty_in_strategy` 来更改此行为。

从 1.4 版本开始更改：`not_in()` 操作符从先前版本的 `notin_()` 重命名为 `not_in()`。先前的名称仍可用于向后兼容。

从 1.2 版本开始更改：`ColumnOperators.in_()` 和 `ColumnOperators.not_in()` 操作符现在默认情况下为空 IN 序列生成“静态”表达式。

另请参阅

`ColumnOperators.in_()`

```py
method notlike(other: Any, escape: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.notlike()` *方法的* `ColumnOperators`

实现 `NOT LIKE` 操作符。

这相当于使用 `ColumnOperators.like()` 时使用否定，即 `~x.like(y)`。

从 1.4 版本开始更改：`not_like()` 操作符从先前版本的 `notlike()` 重命名为 `not_like()`。先前的名称仍可用于向后兼容。

另请参阅

`ColumnOperators.like()`

```py
method nulls_first() → ColumnOperators
```

*继承自* `ColumnOperators.nulls_first()` *方法的* `ColumnOperators`

对父对象生成一个 `nulls_first()` 子句。

在 1.4 版本中更改：`nulls_first()` 操作符在先前版本中的名称已更改为 `nullsfirst()`。以前的名称仍可用于向后兼容。

```py
method nulls_last() → ColumnOperators
```

*继承自* `ColumnOperators.nulls_last()` *方法的* `ColumnOperators`

对父对象生成一个 `nulls_last()` 子句。

在 1.4 版本中更改：`nulls_last()` 操作符在先前版本中的名称已更改为 `nullslast()`。以前的名称仍可用于向后兼容。

```py
method nullsfirst() → ColumnOperators
```

*继承自* `ColumnOperators.nullsfirst()` *方法的* `ColumnOperators`

对父对象生成一个 `nulls_first()` 子句。

在 1.4 版本中更改：`nulls_first()` 操作符在先前版本中的名称已更改为 `nullsfirst()`。以前的名称仍可用于向后兼容。

```py
method nullslast() → ColumnOperators
```

*继承自* `ColumnOperators.nullslast()` *方法的* `ColumnOperators`

对父对象生成一个 `nulls_last()` 子句。

在 1.4 版本中更改：`nulls_last()` 操作符在先前版本中的名称已更改为 `nullslast()`。以前的名称仍可用于向后兼容。

```py
method op(opstring: str, precedence: int = 0, is_comparison: bool = False, return_type: Type[TypeEngine[Any]] | TypeEngine[Any] | None = None, python_impl: Callable[..., Any] | None = None) → Callable[[Any], Operators]
```

*继承自* `Operators.op()` *方法的* `Operators`

产生一个通用的运算符函数。

例如：

```py
somecolumn.op("*")(5)
```

生成：

```py
somecolumn * 5
```

这个函数也可以用来明确位运算符。例如：

```py
somecolumn.op('&')(0xff)
```

是对`somecolumn`值的位与操作。

参数：

+   `opstring` – 将作为该元素与传递给生成函数的表达式之间的中缀运算符输出的字符串。

+   `precedence` –

    数据库预计在 SQL 表达式中应用于运算符的优先级。这个整数值充当 SQL 编译器的提示，以便知道何时应该在特定操作周围呈现显式括号。较低的数字将导致表达式在应用于具有更高优先级的另一个运算符时被加括号。默认值为`0`，低于所有运算符，除了逗号（`,`）和`AS`运算符。值为 100 将高于或等于所有运算符，而-100 将低于或等于所有运算符。

    另请参阅

    我正在使用 op()生成自定义运算符，但是我的括号没有正确显示 - SQLAlchemy SQL 编译器如何呈现括号的详细描述

+   `is_comparison` –

    legacy；如果为 True，则将运算符视为“比较”运算符，即评估为布尔值真/假的运算符，如`==`，`>`等。提供此标志是为了让 ORM 关系能够在自定义连接条件中确定该运算符是比较运算符。

    使用`is_comparison`参数被使用`Operators.bool_op()`方法替代；这个更简洁的运算符会自动设置此参数，但也会提供正确的[**PEP 484**](https://peps.python.org/pep-0484/)类型支持，因为返回的对象将表示“布尔”数据类型，即`BinaryExpression[bool]`。

+   `return_type` – 一个`TypeEngine`类或对象，将强制由此运算符产生的表达式的返回类型为该类型。默认情况下，指定了`Operators.op.is_comparison`的运算符将解析为`Boolean`，而没有指定的将与左操作数的类型相同。

+   `python_impl` –

    一个可选的 Python 函数，可以在数据库服务器上运行时以与该运算符相同的方式评估两个 Python 值。用于在 Python 中进行 SQL 表达式评估函数，例如用于 ORM 混合属性的，以及 ORM“评估器”用于在多行更新或删除后匹配会话中的对象。

    例如：

    ```py
    >>> expr = column('x').op('+', python_impl=lambda a, b: a + b)('y')
    ```

    上述表达式的运算符也将适用于非 SQL 左和右对象：

    ```py
    >>> expr.operator(5, 10)
    15
    ```

    2.0 版本中的新内容。

另请参阅

`Operators.bool_op()`

重新定义和创建新运算符

在连接条件中使用自定义运算符

```py
method operate(op: OperatorType, *other: Any, **kwargs: Any) → ColumnElement[Any]
```

*继承自* `ColumnElement.operate()` *方法的* `ColumnElement`

对参数进行操作。

这是操作的最低级别，默认情况下引发 `NotImplementedError`。

在子类上覆盖此方法可以允许将通用行为应用于所有操作。例如，覆盖 `ColumnOperators` 以将 `func.lower()` 应用于左侧和右侧：

```py
class MyComparator(ColumnOperators):
    def operate(self, op, other, **kwargs):
        return op(func.lower(self), func.lower(other), **kwargs)
```

参数：

+   `op` – 运算符可调用。

+   `*other` – 操作的“其他”一侧。对于大多数操作，将是单个标量。

+   `**kwargs` – 修饰符。这些可能由特殊运算符（如 `ColumnOperators.contains()`）传递。

```py
method params(*optionaldict, **kwargs)
```

*继承自* `Immutable.params()` *方法的* `Immutable`

返回一个副本，其中包含替换了 `bindparam()` 元素的内容。

返回此 ClauseElement 的副本，其中包含从给定字典中获取的值替换了 `bindparam()` 元素：

```py
>>> clause = column('x') + bindparam('foo')
>>> print(clause.compile().params)
{'foo':None}
>>> print(clause.params({'foo':7}).compile().params)
{'foo':7}
```

```py
attribute proxy_set: util.generic_fn_descriptor[FrozenSet[Any]]
```

*继承自* `ColumnElement.proxy_set` *属性的* `ColumnElement`

我们正在代理的所有列的集合

从 2.0 开始，这些是明确取消注释的列。以前是有效取消注释的列，但没有强制执行。如果可能的话，注释的列基本上不应该进入集合，因为它们的哈希行为非常低效。

```py
method references(column: Column[Any]) → bool
```

如果此列通过外键引用给定列，则返回 True。

```py
method regexp_match(pattern: Any, flags: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.regexp_match()` *方法的* `ColumnOperators`

实现了一个特定于数据库的‘正则匹配’操作符。

例如：

```py
stmt = select(table.c.some_column).where(
    table.c.some_column.regexp_match('^(b|c)')
)
```

`ColumnOperators.regexp_match()` 试图解析为后端提供的类似于 REGEXP 的函数或运算符，但可用的特定正则表达式语法和标志 **不是后端通用**。

示例包括：

+   PostgreSQL - 当取反时渲染 `x ~ y` 或 `x !~ y`。

+   Oracle - 渲染 `REGEXP_LIKE(x, y)`

+   SQLite - 使用 SQLite 的 `REGEXP` 占位符操作符，并调用 Python 的 `re.match()` 内置函数。

+   其他后端可能提供特殊实现。

+   没有任何特殊实现的后端将将操作符输出为“REGEXP”或“NOT REGEXP”。例如，这与 SQLite 和 MySQL 兼容。

正则表达式支持目前已在 Oracle、PostgreSQL、MySQL 和 MariaDB 中实现。对于 SQLite，部分支持可用。第三方方言中的支持可能有所不同。

参数：

+   `pattern` – 正则表达式模式字符串或列子句。

+   `flags` – 要应用的任何正则表达式字符串标志，仅作为普通 Python 字符串传递。这些标志是特定于后端的。一些后端，如 PostgreSQL 和 MariaDB，可能会将标志作为模式的一部分指定。在 PostgreSQL 中使用忽略大小写标志‘i’ 时，将使用忽略大小写的正则表达式匹配操作符 `~*` 或 `!~*`。

版本 1.4 中的新功能。

从版本 1.4.48 更改为：2.0.18 请注意，由于实现错误，“flags”参数先前接受了 SQL 表达式对象，例如列表达式，而不仅仅是普通的 Python 字符串。这种实现与缓存不兼容，并已移除；应该仅传递字符串作为“flags”参数，因为这些标志将作为 SQL 表达式中的文字内联值呈现。

另请参见

`ColumnOperators.regexp_replace()`

```py
method regexp_replace(pattern: Any, replacement: Any, flags: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.regexp_replace()` *方法的* `ColumnOperators`

实现了特定于数据库的‘regexp replace’操作符。

例如：

```py
stmt = select(
    table.c.some_column.regexp_replace(
        'b(..)',
        'XY',
        flags='g'
    )
)
```

`ColumnOperators.regexp_replace()` 尝试解析为后端提供的类似 REGEXP_REPLACE 的函数，通常会输出函数 `REGEXP_REPLACE()`。然而，特定的正则表达式语法和可用标志**并非后端通用**。

正则表达式替换支持目前已在 Oracle、PostgreSQL、MySQL 8 或更高版本和 MariaDB 中实现。第三方方言中的支持可能有所不同。

参数：

+   `pattern` – 正则表达式模式字符串或列子句。

+   `pattern` – 替换字符串或列子句。

+   `flags` – 要应用的任何正则表达式字符串标志，仅作为普通 Python 字符串传递。这些标志是特定于后端的。一些后端，如 PostgreSQL 和 MariaDB，可能会将标志作为模式的一部分指定。

版本 1.4 中的新���能。

在版本 1.4.48 中更改：2.0.18 请注意，由于实现错误，以前的“flags”参数接受 SQL 表达式对象，例如列表达式，而不仅仅是普通的 Python 字符串。此实现与缓存一起使用时不起作用，并已删除；只应传递字符串作为“flags”参数，因为这些标志作为 SQL 表达式中的字面内联值呈现。

另请参阅

`ColumnOperators.regexp_match()`

```py
method reverse_operate(op: OperatorType, other: Any, **kwargs: Any) → ColumnElement[Any]
```

继承自 `ColumnElement.reverse_operate()` *方法的* `ColumnElement`

对参数进行反向操作。

用法与 `operate()` 相同。

```py
method self_group(against: OperatorType | None = None) → ColumnElement[Any]
```

继承自 `ColumnElement.self_group()` *方法的* `ColumnElement`

将“分组”应用于此 `ClauseElement`。

子类重写此方法以返回“分组”构造，即括号。特别是当“二进制”表达式放置到较大表达式中时，它们用于提供对自身的分组，以及当 `select()` 构造放置到另一个 `select()` 的 FROM 子句中时。 （请注意，通常应使用 `Select.alias()` 方法创建子查询，因为许多平台要求嵌套的 SELECT 语句具有名称）。

随着表达式的组合，`self_group()` 的应用是自动的 - 最终用户代码不应直接使用此方法。请注意，SQLAlchemy 的子句构造考虑了操作符优先级 - 因此，例如，在表达式 `x OR (y AND z)` 中可能不需要括号 - AND 优先于 OR。

`ClauseElement`的基本 `self_group()` 方法只返回自身。

```py
method shares_lineage(othercolumn: ColumnElement[Any]) → bool
```

*继承自* `ColumnElement.shares_lineage()` *方法的* `ColumnElement`

如果给定的 `ColumnElement` 具有与此 `ColumnElement` 的共同祖先，则返回 True。

```py
method startswith(other: Any, escape: str | None = None, autoescape: bool = False) → ColumnOperators
```

*继承自* `ColumnOperators.startswith()` *方法的* `ColumnOperators`

实现`startswith`运算符。

产生一个 LIKE 表达式，用于测试字符串值的开头匹配：

```py
column LIKE <other> || '%'
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.startswith("foobar"))
```

由于该运算符使用 `LIKE`，因此存在于 <other> 表达式内部的通配符字符 `"%"` 和 `"_"` 也将像通配符一样起作用。对于字面字符串值，可以将 `ColumnOperators.startswith.autoescape` 标志设置为 `True`，以对字符串值内部这些字符的出现应用转义，使它们作为自身而不是通配符字符进行匹配。或者，`ColumnOperators.startswith.escape` 参数将建立一个给定的字符作为转义字符，当目标表达式不是字面字符串时可能会有用。

参数：

+   `other` – 要比较的表达式。通常这是一个简单的字符串值，但也可以是一个任意的 SQL 表达式。除非将 `ColumnOperators.startswith.autoescape` 标志设置为 True，否则 LIKE 通配符字符 `%` 和 `_` 默认不会被转义。

+   `autoescape` –

    布尔值；当为 True 时，在 LIKE 表达式中建立一个转义字符，然后将其应用于比较值中所有出现的`"%"`、`"_"`和转义字符本身，假定比较值是一个字面字符串而不是一个 SQL 表达式。

    一个如下的表达式：

    ```py
    somecolumn.startswith("foo%bar", autoescape=True)
    ```

    将呈现为：

    ```py
    somecolumn LIKE :param || '%' ESCAPE '/'
    ```

    使用`:param`的值为`"foo/%bar"`。

+   `escape` –

    一个字符，当给定时，将以 `ESCAPE` 关键字呈现以将该字符作为转义字符。然后可以在 `%` 和 `_` 的前面放置该字符，以允许它们作为它们自己而不是通配符字符。

    一个如下的表达式：

    ```py
    somecolumn.startswith("foo/%bar", escape="^")
    ```

    将呈现为：

    ```py
    somecolumn LIKE :param || '%' ESCAPE '^'
    ```

    参数也可以与`ColumnOperators.startswith.autoescape`结合使用：

    ```py
    somecolumn.startswith("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上述情况中，给定的字面参数将在传递到数据库之前转换为`"foo^%bar^^bat"`。

另见

`ColumnOperators.endswith()`

`ColumnOperators.contains()`

`ColumnOperators.like()`

```py
attribute timetuple: Literal[None] = None
```

*继承自* `ColumnOperators.timetuple` *属性的* `ColumnOperators`

Hack，允许在 LHS 上比较 datetime 对象。

```py
attribute unique: bool | None
```

`Column.unique`参数的值。

不表示此`Column`实际上是否受唯一约束的约束; 使用`Table.indexes`和`Table.constraints`。

另见

`Table.indexes`

`Table.constraints`。

```py
method unique_params(*optionaldict, **kwargs)
```

*继承自* `Immutable.unique_params()` *方法的* `Immutable`

返回一个副本，其中`bindparam()`元素被替换。

与`ClauseElement.params()`具有相同的功能，除了将 unique=True 添加到受影响的绑定参数，以便可以使用多个语句。

```py
class sqlalchemy.schema.MetaData
```

一个包含`Table`对象及其关联模式结构的集合。

包含一个`Table`对象的集合，以及一个可选的绑定到`Engine`或`Connection`的绑定。如果绑定，则集合中的`Table`对象及其列可能会参与隐式 SQL 执行。

`Table` 对象本身存储在 `MetaData.tables` 字典中。

`MetaData` 是一个线程安全的对象，用于读操作。在单个 `MetaData` 对象内构造新表，无论是显式地还是通过反射，可能不完全是线程安全的。

请参见

使用 MetaData 描述数据库 - 数据库元数据介绍

**成员**

__init__(), clear(), create_all(), drop_all(), reflect(), remove(), sorted_tables, tables

**类签名**

类 `sqlalchemy.schema.MetaData` (`sqlalchemy.schema.HasSchemaAttr`)

```py
method __init__(schema: str | None = None, quote_schema: bool | None = None, naming_convention: _NamingSchemaParameter | None = None, info: _InfoType | None = None) → None
```

创建一个新的 MetaData 对象。

参数：

+   `schema` –

    用于 `Table`、`Sequence` 和可能与此 `MetaData` 关联的其他对象的默认模式。默认为 `None`。

    请参见

    使用 MetaData 指定默认模式名称 - 关于如何使用 `MetaData.schema` 参数的详细信息。

    `Table.schema`

    `Sequence.schema` 

+   `quote_schema` – 为使用本地 `schema` 名称的那些 `Table`、`Sequence` 和其他对象设置 `quote_schema` 标志。

+   `info` – 可选数据字典，将填充到此对象的 `SchemaItem.info` 属性中。

+   `naming_convention` –

    一个字典，指向将为那些未明确给出名称的 `Constraint` 和 `Index` 对象建立默认命名约定的值。

    此字典的键可以是：

    +   一个约束或索引类，例如`UniqueConstraint`类，`ForeignKeyConstraint`类，`Index`类

    +   已知约束类之一的字符串助记符；分别为外键（"fk"），主键（"pk"），索引（"ix"），检查（"ck"），唯一约束（"uq"）。

    +   用户定义的可用于定义新命名标记的“令牌”的字符串名称。

    与每个“约束类”或“约束助记符”键关联的值是字符串命名模板，例如`"uq_%(table_name)s_%(column_0_name)s"`，描述了名称应如何组成。与用户定义的“令牌”键关联的值应该是形式为`fn(constraint, table)`的可调用对象，接受约束/索引对象和`Table`作为参数，返回一个字符串结果。

    内置名称如下，其中一些可能仅适用于某些类型的约束：

    > +   `%(table_name)s` - 与约束相关联的`Table`对象的名称。
    > +   
    > +   `%(referred_table_name)s` - 与`ForeignKeyConstraint`的引用目标相关联的`Table`对象的名称。
    > +   
    > +   `%(column_0_name)s` - 约束内索引位置“0”的`Column`的名称。
    > +   
    > +   `%(column_0N_name)s` - 约束内所有`Column`对象的名称按顺序连接而成，不带分隔符。
    > +   
    > +   `%(column_0_N_name)s` - 约束内所有`Column`对象的名称按顺序连接，用下划线作为分隔符。
    > +   
    > +   `%(column_0_label)s`，`%(column_0N_label)s`，`%(column_0_N_label)s` - 第零个`Column`或所有`Columns`的标签，用或不用下划线分隔
    > +   
    > +   `%(column_0_key)s`，`%(column_0N_key)s`，`%(column_0_N_key)s` - 第零个`Column`或所有`Columns`的键，用或不用下划线分隔。
    > +   
    > +   `%(referred_column_0_name)s`，`%(referred_column_0N_name)s`，`%(referred_column_0_N_name)s`，`%(referred_column_0_key)s`，`%(referred_column_0N_key)s`，…渲染由`ForeignKeyConstraint`引用的列的名称/键/标签的列标记。
    > +   
    > +   `%(constraint_name)s` - 一个特殊的键，引用了约束的现有名称。当存在这个键时，`Constraint` 对象的现有名称将被替换为使用此标记的模板字符串组成的名称。当存在此标记时，必须提前为 `Constraint` 指定一个显式名称。
    > +   
    > +   用户自定义：可以通过将其与 `fn(constraint, table)` 可调用对象一起传递到 naming_convention 字典中来实现任何其他标记。

    1.3.0 版新增内容：- 添加了新的 `%(column_0N_name)s`、`%(column_0_N_name)s` 和相关标记，这些标记生成给定约束引用的所有列的名称、键或标签的串联。

    另请参阅

    配置约束命名约定 - 详细的用法示例。

```py
method clear() → None
```

从此 MetaData 中清除所有 Table 对象。

```py
method create_all(bind: _CreateDropBind, tables: _typing_Sequence[Table] | None = None, checkfirst: bool = True) → None
```

创建存储在此元数据中的所有表。

默认情况下是有条件的，不会尝试重新创建已经存在于目标数据库中的表。

参数：

+   `bind` – 用于访问数据库的 `Connection` 或 `Engine`。

+   `tables` – `Table` 对象的可选列表，这是 `MetaData` 中的总表的子集（其他表被忽略）。

+   `checkfirst` – 默认为 True，不会为已经存在于目标数据库中的表发出 CREATE 操作。

```py
method drop_all(bind: _CreateDropBind, tables: _typing_Sequence[Table] | None = None, checkfirst: bool = True) → None
```

删除存储在此元数据中的所有表。

默认情况下是有条件的，不会尝试删除目标数据库中不存在的表。

参数：

+   `bind` – 用于访问数据库的 `Connection` 或 `Engine`。

+   `tables` – `Table` 对象的可选列表，这是 `MetaData` 中的总表的子集（其他表被忽略）。

+   `checkfirst` – 默认为 True，仅对确认存在于目标数据库中的表发出 DROP 操作。

```py
method reflect(bind: Engine | Connection, schema: str | None = None, views: bool = False, only: _typing_Sequence[str] | Callable[[str, MetaData], bool] | None = None, extend_existing: bool = False, autoload_replace: bool = True, resolve_fks: bool = True, **dialect_kwargs: Any) → None
```

从数据库加载所有可用的表定义。

自动在此 MetaData 中为数据库中尚未存在的任何表创建 `Table` 条目。可以多次调用以获取最近添加到数据库中的表，但如果此 `MetaData` 中的表在数据库中不再存在，则不会采取任何特殊操作。

参数：

+   `bind` – 用于访问数据库的 `Connection` 或 `Engine`。

+   `schema` – 可选，从替代模式查询和反映表。如果为 None，则使用与此`MetaData`关联的模式（如果有）。

+   `views` – 如果为 True，则还反映视图（物化和普通）。

+   `only` –

    可选。仅加载可用命名表的子集。可以指定为名称序列或可调用对象。

    如果提供了名称序列，则只会反映这些表。如果请求了一个表但该表不可用，则会引发错误。已经存在于此`MetaData`中的命名表将被忽略。

    如果提供了可调用对象，则将用作布尔谓词来过滤潜在的表名称列表。可调用对象以表名称和此`MetaData`实例作为位置参数调用，并应为任何要反映的表返回一个真值。

+   `extend_existing` – 传递给每个`Table`作为`Table.extend_existing`。

+   `autoload_replace` – 传递给每个`Table`作为`Table.autoload_replace`。

+   `resolve_fks` –

    如果为 True，则反映与每个`Table`中的`ForeignKey`对象链接的`Table`。对于`MetaData.reflect()`，这将导致反映可能不在要反映的表列表中的相关表，例如，如果引用的表位于不同的模式中或通过`MetaData.reflect.only`参数被省略。当为 False 时，不会跟随`ForeignKey`对象到它们链接的`Table`，但是如果相关表也是反映表列表的一部分，那么在`MetaData.reflect()`操作完成后，`ForeignKey`对象仍将解析为其相关的`Table`。默认为 True。

    1.3.0 版本中的新功能。

    另请参阅

    `Table.resolve_fks`

+   `**dialect_kwargs` – 未在上述提及的其他关键字参数是方言特定的，并以`<方言名称>_<参数名称>`的形式传递。有关详细记录参数的文档，请参阅 Dialects 中有关单个方言的文档。

另请参阅

反映数据库对象

`DDLEvents.column_reflect()` - 用于自定义反映列的事件。通常用于使用`TypeEngine.as_generic()`泛化类型。

使用与数据库无关的类型反映 - 描述如何使用通用类型反映表格。

```py
method remove(table: Table) → None
```

从此 MetaData 中删除给定的 Table 对象。

```py
attribute sorted_tables
```

返回一个按外键依赖顺序排序的`Table`对象列表。

排序将首先将具有依赖关系的`Table`对象放置在依赖关系本身之前，表示它们可以被创建的顺序。要获取表格将被删除的顺序，请使用 Python 内置的`reversed()`。

警告

单独的`MetaData.sorted_tables`属性本身不能自动解决表格之间的依赖循环，这些循环通常是由相互依赖的外键约束引起的。当检测到这些循环时，这些表的外键将被忽略在排序中考虑。当此条件发生时，会发出警告，这将在未来的版本中引发异常。仍将以依赖顺序返回不属于循环的表。

要解决这些循环，可以将`ForeignKeyConstraint.use_alter`参数应用于创建循环的那些约束。或者，当检测到循环时，`sort_tables_and_constraints()`函数将自动将外键约束返回到单独的集合中，以便可以将其应用于模式。

从 1.3.17 版本开始更改：当由于循环依赖关系而无法对`MetaData.sorted_tables`进行适当排序时，会发出警告。这将在未来的版本中引发异常。此外，排序将继续以先前不属于循环的其他表的依赖顺序返回其他表，这不是以前的情况。

另请参阅

`sort_tables()`

`sort_tables_and_constraints()`

`MetaData.tables`

`Inspector.get_table_names()`

`Inspector.get_sorted_table_and_fkc_names()`

```py
attribute tables: util.FacadeDict[str, Table]
```

一个以其名称或“表键”为键的`Table`对象字典。

具体键是由`Table.key`属性确定的；对于没有`Table.schema`属性的表，这与`Table.name`相同。对于具有模式的表，通常是`schemaname.tablename`的形式。

另请参阅

`MetaData.sorted_tables`

```py
class sqlalchemy.schema.SchemaConst
```

一个枚举。

**成员**

BLANK_SCHEMA，NULL_UNSPECIFIED，RETAIN_SCHEMA

**类签名**

类`sqlalchemy.schema.SchemaConst` (`enum.Enum`)

```py
attribute BLANK_SCHEMA = 2
```

表示在某些情况下，即使父 `MetaData` 指定了模式，`Table` 或 `Sequence` 应该具有“None”作为其模式。

另请参阅

`MetaData.schema`

`Table.schema`

`Sequence.schema`。

```py
attribute NULL_UNSPECIFIED = 3
```

表示“nullable”关键字未传递给列的符号。

这用于区分将`nullable=None`传递给`Column`的用例，这在某些后端（如 SQL Server）中具有特殊含义。

```py
attribute RETAIN_SCHEMA = 1
```

表示在某些情况下，对于正在复制的 `Table`、`Sequence` 或者 `ForeignKey` 对象，应该保留其已经具有的模式名称。

```py
class sqlalchemy.schema.SchemaItem
```

数据库模式定义项的基类。

**成员**

info

**类签名**

类 `sqlalchemy.schema.SchemaItem` (`sqlalchemy.sql.expression.SchemaEventTarget`, `sqlalchemy.sql.visitors.Visitable`)

```py
attribute info
```

与对象关联的信息字典，允许将用户定义的数据与此 `SchemaItem` 关联。

字典在首次访问时会自动生成。它也可以在某些对象的构造函数中指定，例如 `Table` 和 `Column`。

```py
function sqlalchemy.schema.insert_sentinel(name: str | None = None, type_: _TypeEngineArgument[_T] | None = None, *, default: Any | None = None, omit_from_statements: bool = True) → Column[Any]
```

提供一个代理 `Column` ，它将充当专用的插入 sentinel 列，允许对没有其他合格的主键配置的表进行高效的批量插入，并实现确定性的 RETURNING 排序。

将此列添加到 `Table` 对象中需要确保相应的数据库表实际上具有此列，因此如果将其添加到现有模型中，则现有的数据库表需要进行迁移（例如使用 ALTER TABLE 或类似操作）以包含此列。

关于此对象的使用背景，请参阅部分 配置 Sentinel 列 作为部分 “插入多个值” INSERT 语句的行为。

默认情况下，返回的 `Column` 将是可空的整数列，并且仅在“insertmanyvalues”操作中使用特定于 sentinel 的默认生成器。

另请参阅

`orm_insert_sentinel()`

`Column.insert_sentinel`

“插入多个值” INSERT 语句的行为

配置 Sentinel 列

新版本中新增功能 2.0.10。

```py
class sqlalchemy.schema.Table
```

在数据库中表示一个表。

例如：

```py
mytable = Table(
    "mytable", metadata,
    Column('mytable_id', Integer, primary_key=True),
    Column('value', String(50))
)
```

`Table` 对象根据其名称和可选模式名称在给定的 `MetaData` 对象内构造唯一的实例。使用相同的名称和相同的 `MetaData` 参数再次调用 `Table` 构造函数将返回相同的 `Table` 对象 - 这样 `Table` 构造函数充当注册函数。

请参阅

描述数据库的元数据 - 数据库元数据介绍

**成员**

__init__()，add_is_dependent_on()，alias()，append_column()，append_constraint()，argument_for()，autoincrement_column，c，columns，compare()，compile()，constraints，corresponding_column()，create()，delete()，description，dialect_kwargs，dialect_options，drop()，entity_namespace，exported_columns，foreign_key_constraints，foreign_keys，get_children()，implicit_returning，indexes，info，inherit_cache，insert()，is_derived_from()，join()，key，kwargs，lateral()，outerjoin()，params()，primary_key，replace_selectable()，schema，select()，self_group()，table_valued()，tablesample()，to_metadata()，tometadata()，unique_params()，update()

**类签名**

类`sqlalchemy.schema.Table`（`sqlalchemy.sql.base.DialectKWArgs`，`sqlalchemy.schema.HasSchemaAttr`，`sqlalchemy.sql.expression.TableClause`，`sqlalchemy.inspection.Inspectable`）

```py
method __init__(name: str, metadata: MetaData, *args: SchemaItem, schema: str | Literal[SchemaConst.BLANK_SCHEMA] | None = None, quote: bool | None = None, quote_schema: bool | None = None, autoload_with: Engine | Connection | None = None, autoload_replace: bool = True, keep_existing: bool = False, extend_existing: bool = False, resolve_fks: bool = True, include_columns: Collection[str] | None = None, implicit_returning: bool = True, comment: str | None = None, info: Dict[Any, Any] | None = None, listeners: _typing_Sequence[Tuple[str, Callable[..., Any]]] | None = None, prefixes: _typing_Sequence[str] | None = None, _extend_on: Set[Table] | None = None, _no_init: bool = True, **kw: Any) → None
```

`Table` 的构造函数。

参数：

+   `name` –

    数据库中表示此表的名称。

    表名与 `schema` 参数的值一起形成一个键，唯一标识此 `Table` 在所属的 `MetaData` 集合中。对具有相同名称、元数据和模式名称的 `Table` 的其他调用将返回相同的 `Table` 对象。

    不含大写字符的名称将被视为大小写不敏感的名称，并且除非它们是保留字或包含特殊字符，否则不会被引用。任何数量的大写字符被视为区分大小写的名称，并将作为引号发送。

    要为表名启用无条件引用，请在构造函数中指定标志 `quote=True`，或使用 `quoted_name` 构造指定名称。

+   `metadata` – 一个 `MetaData` 对象，将包含此表。元数据用作将此表与其他通过外键引用的表关联的点。它也可以用于将此表与特定的 `Connection` 或 `Engine` 关联起来。

+   `*args` – 主要用于添加此表中包含的 `Column` 对象列表的其他位置参数。类似于 CREATE TABLE 语句的风格，其他 `SchemaItem` 构造可以在此处添加，包括 `PrimaryKeyConstraint` 和 `ForeignKeyConstraint`。

+   `autoload_replace` –

    默认为 `True`；与 `Table.extend_existing` 结合使用 `Table.autoload_with` 时，指示已存在的 `Table` 对象中的 `Column` 对象应该被从 autoload 过程中检索到的同名列替换。当 `False` 时，已存在的列将被从反射过程中省略。

    请注意，此设置不会影响程序化指定的 `Column` 对象，在自动加载时也会替换具有相同名称的现有列，当 `Table.extend_existing` 为 `True` 时。

    另请参见

    `Table.autoload_with`

    `Table.extend_existing`

+   `autoload_with` –

    一个 `Engine` 或 `Connection` 对象，或者通过 `inspect()` 对其进行检查后返回的 `Inspector` 对象，其中此 `Table` 对象将被反射。当设置为非 None 值时，将对此表针对给定引擎或连接进行自动加载。

    另请参见

    反射数据库对象

    `DDLEvents.column_reflect()`

    使用数据库无关类型进行反射

+   `extend_existing` –

    当设为`True`时，表示如果此 `Table` 已经存在于给定的 `MetaData` 中，则将构造函数中的进一步参数应用于现有的 `Table`。

    如果未设置 `Table.extend_existing` 或 `Table.keep_existing`，并且新 `Table` 的给定名称引用的是已经存在于目标 `MetaData` 集合中的 `Table`，并且此 `Table` 指定了额外的列或其他结构或修改表状态的标志，那么将引发错误。这两个互斥标志的目的是指定当指定与现有 `Table` 匹配的 `Table` 时应采取的操作，但指定了其他构造。

    `Table.extend_existing` - `Table.extend_existing`属性也将与`Table.autoload_with`结合使用，针对数据库运行新的反射操作，即使目标`MetaData`中已经存在同名的`Table`；新反射的`Column`对象和其他选项将被添加到`Table`的状态中，可能会覆盖同名的现有列和选项。

    与`Table.autoload_with`一样，`Column`对象可以在同一`Table`构造函数中指定，这将优先考虑。下面，现有表`mytable`将被用从数据库反射出的`Column`对象以及给定的名为“y”的`Column`对象增加：

    ```py
    Table("mytable", metadata,
                Column('y', Integer),
                extend_existing=True,
                autoload_with=engine
            )
    ```

    另请参阅

    `Table.autoload_with` - `Table.autoload_with`属性

    `Table.autoload_replace` - `Table.autoload_replace`属性

    `Table.keep_existing` - `Table.keep_existing`属性

+   `implicit_returning` – `implicit_returning`属性

    默认为 True - 表示返回值可以使用，通常由 ORM 使用，以便在支持 RETURNING 的后端上获取服务器生成的值，例如主键值和服务器端默认值。

    在现代 SQLAlchemy 中，通常没有理由修改此设置，除了一些特定于后端的情况（有关一个这样的示例，请参见 SQL Server 方言文档中的 Triggers）。

+   `include_columns` – `include_columns`属性 - 一个字符串列表，指示要通过`autoload`操作加载的列的子集；不在此列表中的表列将不会在生成的`Table`对象上表示。默认为`None`，表示应反映所有列。

+   `resolve_fks` – `resolve_fks`属性

    是否反映与此对象相关的`Table`对象，当指定`Table.autoload_with`时。默认为 True。设置为 False 以禁用遇到的相关表的反射作为`ForeignKey`对象；可以用于节省 SQL 调用或避免无法访问的相关表的问题。请注意，如果相关表已经存在于`MetaData`集合中，或稍后出现，与此`Table`关联的`ForeignKey`对象将正常解析到该表。

    版本 1.3 中的新功能。

    另请参阅

    `MetaData.reflect.resolve_fks`

+   `info` – 将填充到此对象的`SchemaItem.info`属性中的可选数据字典。

+   `keep_existing` –

    当为`True`时，表示如果此 Table 已经存在于给定的`MetaData`中，则忽略构造函数中现有`Table`的进一步参数，并将`Table`对象返回为最初创建的对象。这是为了允许希望在第一次调用时定义新`Table`的函数，在后续调用中将返回相同的`Table`，而不会再次应用任何声明（特别是约束）。

    如果未设置`Table.extend_existing`或`Table.keep_existing`，并且新`Table`的给定名称指的是目标`MetaData`集合中已经存在的一个`Table`，并且这个`Table`指定了额外的列或其他构造或修改表状态的标志，将会引发错误。这两个互斥的标志的目的是指定当指定一个与现有`Table`匹配的`Table`时应采取的操作，但指定了额外的构造。

    另请参阅

    `Table.extend_existing`

+   `listeners` –

    一个形如`(<eventname>, <fn>)`的元组列表，将在构建时传递给`listen()`。这个替代的钩子用于在“autoload”过程开始之前建立一个特定于这个`Table`的监听器函数。历史上，这是用于与`DDLEvents.column_reflect()`事件一起使用的，但请注意，现在可以直接将此事件钩子与`MetaData`对象关联起来：

    ```py
    def listen_for_reflect(table, column_info):
        "handle the column reflection event"
        # ...

    t = Table(
        'sometable',
        autoload_with=engine,
        listeners=[
            ('column_reflect', listen_for_reflect)
        ])
    ```

    另请参阅

    `DDLEvents.column_reflect()`

+   `must_exist` – 当为`True`时，表示这个 Table 必须已经存在于给定的`MetaData`集合中，否则将引发异常。

+   `prefixes` – 一个字符串列表，插入在 CREATE TABLE 语句中 CREATE 后面。它们将用空格分隔。

+   `quote` –

    强制引用此表的名称打开或关闭，对应为`True`或`False`。当保持其默认值`None`时，根据名称是否区分大小写（至少有一个大写字符的标识符被视为区分大小写），或者它是否是保留字来引用列标识符。只有在需要强制引用 SQLAlchemy 方言不知道的保留字时才需要此标志。

    注意

    将此标志设置为 `False` 将不会为表反射提供不区分大小写的行为；表反射将始终以区分大小写的方式搜索混合大小写名称。 SQLAlchemy 中仅通过使用所有小写字符的名称来指定不区分大小写的名称。

+   `quote_schema` - 与 ‘quote’ 相同，但适用于模式标识符。

+   `schema` -

    此表的模式名称，如果表位于引擎的数据库连接的默认选定模式之外的模式中，则需要。 默认为 `None`。

    如果此 `Table` 的拥有者 `MetaData` 指定了自己的 `MetaData.schema` 参数，则如果此处的模式参数设置为 `None`，则将该模式名称应用于此 `Table`。 要在否则使用所设置的模式的拥有者 `MetaData` 的 `Table` 上设置空白模式名称，请指定特殊符号 `BLANK_SCHEMA`。

    模式名称的引用规则与 `name` 参数的规则相同，即对保留字或区分大小写的名称应用引用； 要为模式名称启用无条件引用，请在构造函数中指定标志 `quote_schema=True`，或使用 `quoted_name` 结构来指定名称。

+   `comment` -

    可选字符串，将在表创建时呈现 SQL 注释。

    新版本 1.2 中：添加了 `Table.comment` 参数到 `Table`。

+   `**kw` - 上面未提及的附加关键字参数是特定于方言的，并以 `<dialectname>_<argname>` 的形式传递。 有关有关个别方言的文档中记录的参数的详细信息，请参阅 方言 中的个别方言的文档。

```py
method add_is_dependent_on(table: Table) → None
```

为此表添加一个‘依赖’。

这是另一个必须在此之前创建的 Table 对象，或者在此之后删除的对象。

通常，表之间的依赖关系是通过外键对象确定的。然而，对于创建外键以外的其他情况（规则、继承），可以手动建立这样的链接。

```py
method alias(name: str | None = None, flat: bool = False) → NamedFromClause
```

*继承自* `FromClause.alias()` *方法的* `FromClause`

返回此 `FromClause` 的别名。

例如：

```py
a2 = some_table.alias('a2')
```

上述代码创建了一个 `Alias` 对象，可在任何 SELECT 语句中用作 FROM 子句。

另见

使用别名

`alias()` 

```py
method append_column(column: ColumnClause[Any], replace_existing: bool = False) → None
```

向此 `Table` 添加一个 `Column`。

新添加的 `Column` 的“键”，即其`.key`属性的值，将在此 `Table` 的`.c`集合中可用，并且列定义将包含在从此 `Table` 构造生成的任何 CREATE TABLE、SELECT、UPDATE 等语句中。

请注意，这不会更改表的定义，因为它存在于任何底层数据库中，假设该表已经在数据库中创建。关系数据库支持使用 SQL ALTER 命令向现有表添加列，这将需要对于已经存在但不包含新添加列的表发出。 

参数：

**replace_existing** –

当为`True`时，允许替换现有列。当为`False`时，将会发出警告，如果具有相同`.key`的列已经存在。SQLAlchemy 的将来版本将改为提出警告。

1.4.0 版本中的新功能。

```py
method append_constraint(constraint: Index | Constraint) → None
```

向此 `Table` 添加一个 `Constraint`。

这样做会使约束包含在任何未来的 CREATE TABLE 语句中，假设特定的 DDL 创建事件尚未与给定的`Constraint`对象关联。

请注意，这不会自动在关系数据库中生成约束，对于已经存在于数据库中的表。要向现有的关系数据库表添加约束，必须使用 SQL ALTER 命令。SQLAlchemy 还提供了当调用时可以生成此 SQL 的`AddConstraint`构造。

```py
classmethod argument_for(dialect_name, argument_name, default)
```

*继承自* `DialectKWArgs.argument_for()` *方法的* `DialectKWArgs` 

为此类添加一种新的特定于方言的关键字参数。

例如：

```py
Index.argument_for("mydialect", "length", None)

some_index = Index('a', 'b', mydialect_length=5)
```

`DialectKWArgs.argument_for()` 方法是向 `DefaultDialect.construct_arguments` 字典添加额外参数的逐参数方式。该字典提供了一组由方言代表的各种模式级构造所接受的参数名称。

新方言通常应一次性将此字典指定为方言类的数据成员。通常情况下，对于临时添加参数名称的用例，是针对同时使用自定义编译方案的最终用户代码，该编译方案会使用额外的参数。

参数：

+   `dialect_name` – 方言名称。必须能够找到该方言，否则会引发`NoSuchModuleError`。方言还必须包括现有的`DefaultDialect.construct_arguments`集合，表明它参与关键字参数验证和默认系统，否则会引发`ArgumentError`。如果方言不包括此集合，则可以已经代表该方言指定任何关键字参数。SQLAlchemy 中打包的所有方言都包括此集合，但是对于第三方方言，支持可能有所不同。

+   `argument_name` – 参数名称。

+   `default` – 参数的默认值。

```py
attribute autoincrement_column
```

返回当前表示“自动增量”列（如果有）的`Column`对象；否则返回 None。

这基于`Column`的规则，由`Column.autoincrement`参数定义，通常意味着在不受外键约束的单整数列主键约束内的列。如果表格没有这样的主键约束，那么就没有“自动增量”列。`Table`可能只有一个列被定义为“自动增量”列。

版本 2.0.4 中的新功能。

另请参阅

`Column.autoincrement`

```py
attribute c
```

*继承自* `FromClause` *的* `FromClause.c` *属性*

`FromClause.columns` 的同义词

返回：

一个 `ColumnCollection`

```py
attribute columns
```

*继承自* `FromClause.columns` *属性的* `FromClause`

由此 `FromClause` 维护的基于名称的 `ColumnElement` 对象集合。

`columns` 或 `c` 集合是使用表绑定或其他可选择绑定列构建 SQL 表达式的门户：

```py
select(mytable).where(mytable.c.somecolumn == 5)
```

返回：

一个 `ColumnCollection` 对象。

```py
method compare(other: ClauseElement, **kw: Any) → bool
```

*继承自* `ClauseElement.compare()` *方法的* `ClauseElement`

将此 `ClauseElement` 与给定的 `ClauseElement` 进行比较。

子类应重写默认行为，即直接标识比较。

**kw 是子类 `compare()` 方法消耗的参数，可以用于修改比较条件（参见 `ColumnElement`）。

```py
method compile(bind: _HasDialect | None = None, dialect: Dialect | None = None, **kw: Any) → Compiled
```

*继承自* `CompilerElement.compile()` *方法的* `CompilerElement`

编译此 SQL 表达式。

返回值是一个 `Compiled` 对象。调用返回值的 `str()` 或 `unicode()` 方法将产生结果的字符串表示。`Compiled` 对象还可以使用 `params` 访问器返回绑定参数名称和值的字典。

参数：

+   `bind` – 一个`Connection` 或 `Engine`，可以提供一个`Dialect`以生成一个`Compiled`对象。如果 `bind` 和 `dialect` 参数都被省略，则使用默认的 SQL 编译器。

+   `column_keys` – 用于 INSERT 和 UPDATE 语句，一个列名列表，应该出现在编译后的语句的 VALUES 子句中。如果为 `None`，则渲染目标表对象的所有列。

+   `dialect` – 一个`Dialect`实例，可以生成一个`Compiled`对象。此参数优先于`bind`参数。

+   `compile_kwargs` –

    附加字典，其中包含将传递到所有“visit”方法中的其他参数。这允许将任何自定义标志传递给自定义编译构造，例如。它还用于传递 `literal_binds` 标志的情况：

    ```py
    from sqlalchemy.sql import table, column, select

    t = table('t', column('x'))

    s = select(t).where(t.c.x == 5)

    print(s.compile(compile_kwargs={"literal_binds": True}))
    ```

另请参阅

我如何将 SQL 表达式渲染为字符串，可能会内联绑定的参数？

```py
attribute constraints: Set[Constraint]
```

与此 `Table` 关联的所有 `Constraint` 对象的集合。

包括`PrimaryKeyConstraint`、`ForeignKeyConstraint`、`UniqueConstraint`、`CheckConstraint`。一个单独的集合`Table.foreign_key_constraints` 指的是所有 `ForeignKeyConstraint` 对象的集合，而 `Table.primary_key` 属性指的是与 `Table` 关联的单个 `PrimaryKeyConstraint`。

另请参阅

`Table.constraints`

`Table.primary_key`

`Table.foreign_key_constraints`

`Table.indexes`

`Inspector`

```py
method corresponding_column(column: KeyedColumnElement[Any], require_embedded: bool = False) → KeyedColumnElement[Any] | None
```

*继承自* `Selectable.corresponding_column()` *方法的* `Selectable`

给定一个`ColumnElement`，从此`Selectable`的`Selectable.exported_columns`集合中返回与原始`ColumnElement`通过共同祖先列对应的导出`ColumnElement`对象。

参数：

+   `column` – 要匹配的目标`ColumnElement`。

+   `require_embedded` – 只返回给定`ColumnElement`对应的列，如果给定的`ColumnElement`实际上存在于此`Selectable`的子元素中。通常，如果列仅与此`Selectable`的导出列之一共享共同的祖先，则列将匹配。

另请参阅

`Selectable.exported_columns` - 用于操作的`ColumnCollection`。

`ColumnCollection.corresponding_column()` - 实现方法。

```py
method create(bind: _CreateDropBind, checkfirst: bool = False) → None
```

使用给定的`Connection`或`Engine`进行连接，为此`Table`发出`CREATE`语句。

请参阅

`MetaData.create_all()`。

```py
method delete() → Delete
```

*继承自* `TableClause.delete()` *方法的* `TableClause`

生成针对此`TableClause`的`delete()`构造。

例如：

```py
table.delete().where(table.c.id==7)
```

参见`delete()`获取参数和使用信息。

```py
attribute description
```

*继承自* `TableClause.description` *属性的* `TableClause`

```py
attribute dialect_kwargs
```

*继承自* `DialectKWArgs.dialect_kwargs` *属性的* `DialectKWArgs`

作为方言特定选项指定的关键字参数的集合。

这里的参数以其原始的`<dialect>_<kwarg>`格式呈现。只包括实际传递的参数；不像`DialectKWArgs.dialect_options`集合，该集合包含此方言的所有已知选项，包括默认值。

该集合也是可写的；接受形式为`<dialect>_<kwarg>`的键，其中值将被组装成选项列表。

请参阅

`DialectKWArgs.dialect_options` - 嵌套字典形式

```py
attribute dialect_options
```

*继承自* `DialectKWArgs.dialect_options` *属性的* `DialectKWArgs`

作为方言特定选项指定的关键字参数的集合。

这是一个两级嵌套注册表，键为`<dialect_name>`和`<argument_name>`。例如，`postgresql_where`参数可以定位为：

```py
arg = my_object.dialect_options['postgresql']['where']
```

版本 0.9.2 中的新功能。

请参见

`DialectKWArgs.dialect_kwargs` - 平面字典形式

```py
method drop(bind: _CreateDropBind, checkfirst: bool = False) → None
```

使用给定的 `Connection` 或 `Engine` 发出此 `Table` 的 `DROP` 语句以进行连接。

请参见

`MetaData.drop_all()`.

```py
attribute entity_namespace
```

*继承自* `FromClause.entity_namespace` *属性的* `FromClause`

返回用于在 SQL 表达式中基于名称的访问的命名空间。

这是用于解析“filter_by()”类型表达式的命名空间，例如：

```py
stmt.filter_by(address='some address')
```

它默认为 `.c` 集合，但是内部可以使用“entity_namespace”注解进行重写以提供替代结果。

```py
attribute exported_columns
```

*继承自* `FromClause.exported_columns` *属性的* `FromClause`

代表此 `Selectable` 的“导出”列的 `ColumnCollection`。

`FromClause` 对象的“导出”列与 `FromClause.columns` 集合是同义词。

版本 1.4 中的新功能。

请参见

`Selectable.exported_columns`

`SelectBase.exported_columns`

```py
attribute foreign_key_constraints
```

`ForeignKeyConstraint` 对象是由这个 `Table` 引用的。

此列表是从当前关联的 `ForeignKey` 对象集合生成的。

请参见

`Table.constraints`

`Table.foreign_keys`

`Table.indexes`

```py
attribute foreign_keys
```

*继承自* `FromClause.foreign_keys` *属性的* `FromClause`

返回此 FromClause 引用的所有 `ForeignKey` 标记对象的集合。

每个`ForeignKey`都是`Table`范围内的`ForeignKeyConstraint`的成员。

另请参阅

`Table.foreign_key_constraints`

```py
method get_children(*, omit_attrs: Tuple[str, ...] = (), **kw: Any) → Iterable[HasTraverseInternals]
```

*继承自* `HasTraverseInternals.get_children()` *方法的* `HasTraverseInternals`

返回此 `HasTraverseInternals` 的直接子 `HasTraverseInternals` 元素。

这用于访问遍历。

**kw 可能包含更改返回集合的标志，例如返回子集以减少更大的遍历，或者返回来自不同上下文（例如模式级集合而不是子句级集合）的子项。**

```py
attribute implicit_returning = False
```

*继承自* `TableClause.implicit_returning` *属性的* `TableClause`

`TableClause` 不支持具有主键或列级默认值，因此隐式返回不适用。

```py
attribute indexes: Set[Index]
```

与此 `Table` 关联的所有 `Index` 对象的集合。

另请参阅

`Inspector.get_indexes()`

```py
attribute info
```

*继承自* `SchemaItem.info` *属性的* `SchemaItem`

与对象关联的信息字典，允许将用户定义的数据与此 `SchemaItem` 关联。

字典在首次访问时会自动生成。它也可以在某些对象的构造函数中指定，例如`Table`和`Column`。

```py
attribute inherit_cache: bool | None = None
```

*继承自* `HasCacheKey.inherit_cache` *属性的* `HasCacheKey`

指示此`HasCacheKey`实例是否应使用其直接超类使用的缓存键生成方案。

该属性默认为`None`，表示一个结构尚未考虑其是否适合参与缓存；这在功能上等同于将值设置为`False`，只是还会发出警告。

这个标志可以在特定类上设置为`True`，如果对应于对象的 SQL 不基于本类的局部属性而变化，而不是基于其超类。

请参阅

为自定义结构启用缓存支持 - 设置第三方或用户定义的 SQL 结构的`HasCacheKey.inherit_cache`属性的一般指南。

```py
method insert() → Insert
```

*继承自* `TableClause.insert()` *方法的* `TableClause`

生成针对此`TableClause`的`Insert`构造。

例如：

```py
table.insert().values(name='foo')
```

有关参数和用法信息，请参阅`insert()`。

```py
method is_derived_from(fromclause: FromClause | None) → bool
```

*继承自* `FromClause.is_derived_from()` *方法的* `FromClause`

如果此`FromClause`是从给定的`FromClause`‘派生’，则返回`True`。

一个示例是，表的别名派生自该表。

```py
method join(right: _FromClauseArgument, onclause: _ColumnExpressionArgument[bool] | None = None, isouter: bool = False, full: bool = False) → Join
```

*继承自* `FromClause.join()` *方法的* `FromClause`

从此`FromClause`返回到另一个`FromClause`的`Join`。

例如：

```py
from sqlalchemy import join

j = user_table.join(address_table,
                user_table.c.id == address_table.c.user_id)
stmt = select(user_table).select_from(j)
```

会生成类似以下的 SQL：

```py
SELECT user.id, user.name FROM user
JOIN address ON user.id = address.user_id
```

参数：

+   `right` – 连接的右侧；这是任何 `FromClause` 对象，如 `Table` 对象，也可以是可选兼容对象，例如 ORM 映射的类。

+   `onclause` – 表示连接的 ON 子句的 SQL 表达式。如果保持为 `None`，`FromClause.join()` 将尝试根据外键关系连接两个表。

+   `isouter` – 如果为 True，则渲染一个 LEFT OUTER JOIN，而不是 JOIN。

+   `full` – 如果为 True，则渲染一个 FULL OUTER JOIN，而不是 LEFT OUTER JOIN。暗示 `FromClause.join.isouter`。

另请参阅

`join()` - 独立函数

`Join` - 产生的对象类型

```py
attribute key
```

返回此`Table`的 'key'。

此值用作 `MetaData.tables` 集合中的字典键。对于未设置 `Table.schema` 的表，它通常与 `Table.name` 相同；否则，通常是 `schemaname.tablename` 的形式。

```py
attribute kwargs
```

*继承自* `DialectKWArgs.kwargs` *属性的* `DialectKWArgs`

`DialectKWArgs.dialect_kwargs` 的同义词。

```py
method lateral(name: str | None = None) → LateralFromClause
```

*继承自* `Selectable.lateral()` *方法的* `Selectable`

返回此`Selectable`的 LATERAL 别名。

返回值也是顶级 `lateral()` 函数提供的 `Lateral` 构造。

另请参阅

LATERAL 关联 - 用法概述。

```py
method outerjoin(right: _FromClauseArgument, onclause: _ColumnExpressionArgument[bool] | None = None, full: bool = False) → Join
```

*继承自* `FromClause.outerjoin()` *方法的* `FromClause`

从此 `FromClause` 返回一个 `Join` 到另一个 `FromClause`，并将 “isouter” 标志设置为 True。

例如：

```py
from sqlalchemy import outerjoin

j = user_table.outerjoin(address_table,
                user_table.c.id == address_table.c.user_id)
```

上述相当于：

```py
j = user_table.join(
    address_table,
    user_table.c.id == address_table.c.user_id,
    isouter=True)
```

参数：

+   `right` – 连接的右侧；这是任何`FromClause` 对象，如`Table` 对象，也可以是可选择兼容的对象，如 ORM 映射类。

+   `onclause` – 表示连接的 ON 子句的 SQL 表达式。如果保留为 `None`，`FromClause.join()` 将尝试基于外键关系连接两个表。

+   `full` – 如果为 True，则渲染 FULL OUTER JOIN，而不是 LEFT OUTER JOIN。

另请参阅

`FromClause.join()`

`Join`

```py
method params(*optionaldict, **kwargs)
```

*继承自* `Immutable` 的 `Immutable.params()` *方法*

返回一个副本，并用 `bindparam()` 元素替换。

返回此 ClauseElement 的副本，并用给定字典中的值替换`bindparam()` 元素：

```py
>>> clause = column('x') + bindparam('foo')
>>> print(clause.compile().params)
{'foo':None}
>>> print(clause.params({'foo':7}).compile().params)
{'foo':7}
```

```py
attribute primary_key
```

*继承自* `FromClause.primary_key` *属性的* `FromClause`

返回此 `_selectable.FromClause` 的主键组成的可迭代列 `Column` 对象集合。

对于 `Table` 对象，此集合由 `PrimaryKeyConstraint` 表示，它本身是一个可迭代的 `Column` 对象集合。

```py
method replace_selectable(old: FromClause, alias: Alias) → Self
```

*继承自* `Selectable.replace_selectable()` *方法的* `Selectable`

将所有 `FromClause` 中的 ‘old’ 替换为给定的 `Alias` 对象，返回此 `FromClause` 的副本。

自 1.4 版本起已弃用：`Selectable.replace_selectable()` 方法已弃用，并将在将来的发布中删除。类似功能可通过 sqlalchemy.sql.visitors 模块获得。

```py
attribute schema: str | None = None
```

*继承自* `FromClause.schema` *属性的* `FromClause`

为此 `FromClause` 定义 ‘schema’ 属性。

对于大多数对象而言，这通常为 `None`，除了 `Table` 对象，其中它被视为 `Table.schema` 参数的值。

```py
method select() → Select
```

*继承自* `FromClause.select()` *方法的* `FromClause`

返回此 `FromClause` 的 SELECT。

例如：

```py
stmt = some_table.select().where(some_table.c.id == 5)
```

另请参阅

`select()` - 允许任意列列表的通用方法。

```py
method self_group(against: OperatorType | None = None) → ClauseElement
```

*继承自* `ClauseElement.self_group()` *方法的* `ClauseElement`

对这个`ClauseElement`应用一个“分组”。

子类会重写这个方法以返回一个“分组”结构，即括号。特别地，它被“二进制”表达式用来在放置到更大的表达式中时提供一个围绕自己的分组，以及被`select()`构造用来放置到另一个`select()`的 FROM 子句中时。(注意，子查询通常应该使用`Select.alias()`方法创建，因为许多平台要求嵌套的 SELECT 语句必须具名)。

当表达式被组合在一起时，`self_group()`的应用是自动的 - 最终用户代码不应直接使用此方法。注意，SQLAlchemy 的子句构造会考虑运算符优先级 - 因此括号可能不是必需的，例如在`x OR (y AND z)`这样的表达式中 - AND 优先于 OR。

`ClauseElement`的基本`self_group()`方法只返回自身。

```py
method table_valued() → TableValuedColumn[Any]
```

*继承自* `NamedFromClause` 的 `NamedFromClause.table_valued()` *方法*

返回这个`FromClause`的`TableValuedColumn`对象。

`TableValuedColumn`是一个代表表中完整行的`ColumnElement`。对于这个构造的支持依赖于后端，而且由后端以不同形式支持，例如 PostgreSQL、Oracle 和 SQL Server。

例如：

```py
>>> from sqlalchemy import select, column, func, table
>>> a = table("a", column("id"), column("x"), column("y"))
>>> stmt = select(func.row_to_json(a.table_valued()))
>>> print(stmt)
SELECT  row_to_json(a)  AS  row_to_json_1
FROM  a 
```

新版本 1.4.0b2 中新增。

另见

与 SQL 函数一起工作 - 在 SQLAlchemy 统一教程中

```py
method tablesample(sampling: float | Function[Any], name: str | None = None, seed: roles.ExpressionElementRole[Any] | None = None) → TableSample
```

*继承自* `FromClause.tablesample()` *方法*

返回这个`FromClause`的 TABLESAMPLE 别名。

返回值是由顶层`tablesample()`函数提供的`TableSample`构造。

另请参阅

`tablesample()` - 用法指南和参数

```py
method to_metadata(metadata: MetaData, schema: str | Literal[SchemaConst.RETAIN_SCHEMA] = SchemaConst.RETAIN_SCHEMA, referred_schema_fn: Callable[[Table, str | None, ForeignKeyConstraint, str | None], str | None] | None = None, name: str | None = None) → Table
```

返回与不同的`MetaData`相关联的此`Table`的副本。

例如：

```py
m1 = MetaData()

user = Table('user', m1, Column('id', Integer, primary_key=True))

m2 = MetaData()
user_copy = user.to_metadata(m2)
```

从 1.4 版本开始更改：`Table.to_metadata()`函数的名称已从`Table.tometadata()`更改。

参数：

+   `metadata` – 目标`MetaData`对象，将在其中创建新的`Table`对象。

+   `schema` –

    可选字符串名称，指示目标模式。默认为特殊符号`RETAIN_SCHEMA`，表示在新的`Table`中不应更改模式名称。如果设置为字符串名称，则新的`Table`将具有此新名称作为`.schema`。如果设置为`None`，则模式将设置为在目标`MetaData`上设置的模式，通常也是`None`，除非明确设置：

    ```py
    m2 = MetaData(schema='newschema')

    # user_copy_one will have "newschema" as the schema name
    user_copy_one = user.to_metadata(m2, schema=None)

    m3 = MetaData()  # schema defaults to None

    # user_copy_two will have None as the schema name
    user_copy_two = user.to_metadata(m3, schema=None)
    ```

+   `referred_schema_fn` –

    可选的可调用对象，可以提供应分配给`ForeignKeyConstraint`的引用表的模式名称。可调用对象接受此父`Table`、我们正在更改的目标模式、`ForeignKeyConstraint`对象以及该约束的现有“目标模式”。该函数应返回应用的字符串模式名称。要将模式重置为“无”，请返回符号`BLANK_SCHEMA`。要不进行更改，请返回`None`或`RETAIN_SCHEMA`。

    从 1.4.33 版本开始更改：`referred_schema_fn`函数可以返回`BLANK_SCHEMA`或`RETAIN_SCHEMA`符号。

    例如：

    ```py
    def referred_schema_fn(table, to_schema,
                                    constraint, referred_schema):
        if referred_schema == 'base_tables':
            return referred_schema
        else:
            return to_schema

    new_table = table.to_metadata(m2, schema="alt_schema",
                            referred_schema_fn=referred_schema_fn)
    ```

+   `name` – 可选字符串名称，指示目标表名称。如果未指定或为 None，则保留表名称。这允许将`Table`复制到具有新名称的相同`MetaData`目标。

```py
method tometadata(metadata: MetaData, schema: str | Literal[SchemaConst.RETAIN_SCHEMA] = SchemaConst.RETAIN_SCHEMA, referred_schema_fn: Callable[[Table, str | None, ForeignKeyConstraint, str | None], str | None] | None = None, name: str | None = None) → Table
```

返回与不同`MetaData`关联的此`Table`的副本。

从版本 1.4 开始弃用：`Table.tometadata()`已更名为`Table.to_metadata()`

请参阅`Table.to_metadata()`获取完整描述。

```py
method unique_params(*optionaldict, **kwargs)
```

*继承自* `Immutable.unique_params()` *方法的* `Immutable` *对象*

返回一个副本，其中包含替换为`bindparam()`元素的内容。

与`ClauseElement.params()`具有相同的功能，只是对受影响的绑定参数添加了 unique=True，以便可以使用多个语句。

```py
method update() → Update
```

*继承自* `TableClause.update()` *方法的* `TableClause` *对象*

针对此`TableClause`生成一个`update()`构造。

例如：

```py
table.update().where(table.c.id==7).values(name='foo')
```

请参阅`update()`获取参数和用法信息。
