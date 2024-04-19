# 列的插入/更新默认值

> 原文：[`docs.sqlalchemy.org/en/20/core/defaults.html`](https://docs.sqlalchemy.org/en/20/core/defaults.html)

列的插入和更新默认值是指在针对该行进行插入或更新语句时，为该列创建**默认值**的函数，前提是对该列的插入或更新语句未提供任何值。也就是说，如果一个表有一个名为“timestamp”的列，并且进行了不包含该列值的插入语句，那么插入默认值将创建一个新值，例如当前时间，该值将用作要插入到“timestamp”列的值。如果语句*包含*该列的值，则默认值*不*会发生。

列默认值可以是服务器端函数或与数据库中的架构一起定义的常量值，在 DDL 中，或者作为 SQLAlchemy 直接在 INSERT 或 UPDATE 语句中呈现的 SQL 表达式；它们也可以是由 SQLAlchemy 在将数据传递到数据库之前调用的客户端 Python 函数或常量值。

注意

列默认处理程序不应与拦截和修改传递给语句的插入和更新语句中的值的构造混淆。这称为数据编组，在这里，在将列值发送到数据库之前，应用程序以某种方式修改列值。SQLAlchemy 提供了几种实现这一点的方法，包括使用自定义数据类型、SQL 执行事件以及 ORM 中的自定义验证器以及属性事件。列默认值仅在 SQL DML 语句中的某一列没有**值时**调用。

SQLAlchemy 提供了一系列关于在插入和更新语句中针对不存在的值进行默认生成函数的特性。选项包括：

+   插入和更新操作中用作默认值的标量值

+   在插入和更新操作中执行的 Python 函数

+   嵌入到插入语句中的 SQL 表达式（或在某些情况下提前执行的表达式）

+   嵌入到更新语句中的 SQL 表达式

+   插入时使用的服务器端默认值

+   用于更新时的服务器端触发器的标记

所有插入/更新默认值的一般规则是，只有当某一列的值未作为`execute()`参数传递时，它们才会生效；否则，将使用给定的值。

## 标量默认值

最简单的默认值是用作列的默认值的标量值：

```py
Table("mytable", metadata_obj, Column("somecolumn", Integer, default=12))
```

在上述情况下，如果没有提供其他值，则“12”将绑定为插入时的列值。

标量值也可以与 UPDATE 语句关联，但这不是很常见（因为 UPDATE 语句通常正在寻找动态默认值）：

```py
Table("mytable", metadata_obj, Column("somecolumn", Integer, onupdate=25))
```

## Python 执行的函数

`Column.default` 和 `Column.onupdate` 关键字参数也接受 Python 函数。如果没有为该列提供其他值，则在插入或更新时调用这些函数，并且返回的值将用于列的值。下面说明了一个简单的“序列”，它将递增的计数器分配给主键列：

```py
# a function which counts upwards
i = 0

def mydefault():
    global i
    i += 1
    return i

t = Table(
    "mytable",
    metadata_obj,
    Column("id", Integer, primary_key=True, default=mydefault),
)
```

应该注意，对于真正的“递增序列”行为，通常应使用数据库的内置功能，这可能包括序列对象或其他自动增量功能。对于主键列，SQLAlchemy 在大多数情况下将自动使用这些功能。请参阅关于 `Column` 的 API 文档，包括 `Column.autoincrement` 标志，以及本章后面关于 `Sequence` 的部分，了解标准主键生成技术的背景。

为了说明 `onupdate`，我们将 Python 的 `datetime` 函数 `now` 赋值给 `Column.onupdate` 属性：

```py
import datetime

t = Table(
    "mytable",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    # define 'last_updated' to be populated with datetime.now()
    Column("last_updated", DateTime, onupdate=datetime.datetime.now),
)
```

当执行更新语句并且没有为 `last_updated` 传递值时，将执行 `datetime.datetime.now()` Python 函数，并将其返回值用作 `last_updated` 的值。请注意，我们将 `now` 提供为函数本身而不是调用它（即后面没有括号） - SQLAlchemy 将在语句执行时执行该函数。

### 上下文敏感的默认函数

`Column.default` 和 `Column.onupdate` 使用的 Python 函数也可以利用当前语句的上下文来确定一个值。语句的上下文是一个内部的 SQLAlchemy 对象，它包含有关正在执行的语句的所有信息，包括其源表达式、与之关联的参数和游标。与默认生成相关的上下文的典型用例是访问正在插入或更新的行上的其他值。要访问上下文，请提供一个接受单个 `context` 参数的函数：

```py
def mydefault(context):
    return context.get_current_parameters()["counter"] + 12

t = Table(
    "mytable",
    metadata_obj,
    Column("counter", Integer),
    Column("counter_plus_twelve", Integer, default=mydefault, onupdate=mydefault),
)
```

上述默认生成函数适用于所有 INSERT 和 UPDATE 语句，其中未提供 `counter_plus_twelve` 的值，其值将是执行 `counter` 列的值加上数字 12。

对于使用“executemany”样式执行的单个语句，例如传递给 `Connection.execute()` 的多个参数集，用户定义的函数将为每组参数集调用一次。对于多值 `Insert` 构造的用例（例如通过 `Insert.values()` 方法设置了多个 VALUES 子句），用户定义的函数也将为每组参数集调用一次。

当调用该函数时，上下文对象（`DefaultExecutionContext` 的子类）中可用特殊方法 `DefaultExecutionContext.get_current_parameters()`。该方法返回一个列键到值的字典，表示 INSERT 或 UPDATE 语句的完整值集。在多值 INSERT 构造的情况下，与单个 VALUES 子句对应的参数子集被从完整参数字典中隔离并单独返回。

新版本 1.2 中：添加了 `DefaultExecutionContext.get_current_parameters()` 方法，它通过提供将多个 VALUES 子句组织成单独参数字典的服务，改进了仍然存在的 `DefaultExecutionContext.current_parameters` 属性。## 客户端调用的 SQL 表达式

`Column.default` 和 `Column.onupdate` 关键字还可以传递 SQL 表达式，大多数情况下，这些表达式将在 INSERT 或 UPDATE 语句中直接呈现。

```py
t = Table(
    "mytable",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    # define 'create_date' to default to now()
    Column("create_date", DateTime, default=func.now()),
    # define 'key' to pull its default from the 'keyvalues' table
    Column(
        "key",
        String(20),
        default=select(keyvalues.c.key).where(keyvalues.c.type="type1"),
    ),
    # define 'last_modified' to use the current_timestamp SQL function on update
    Column("last_modified", DateTime, onupdate=func.utc_timestamp()),
)
```

在上面的例子中，`create_date` 列将在插入语句中使用 `now()` SQL 函数的结果填充（根据后端的不同，在大多数情况下编译为 `NOW()` 或 `CURRENT_TIMESTAMP`），而 `key` 列将使用另一个表的 SELECT 子查询的结果填充。当为该表发出更新语句时，`last_modified` 列将填充为 SQL `UTC_TIMESTAMP()` MySQL 函数的值。

注意

在使用 `func` 构造与 SQL 函数时，我们“调用”命名函数，例如在 `func.now()` 中使用括号。这与当我们将 Python 可调用对象指定为默认值时不同，例如 `datetime.datetime`，在这种情况下，我们传递函数本身，但我们自己不调用它。对于 SQL 函数，调用 `func.now()` 返回将“NOW”函数渲染到正在发射的 SQL 中的 SQL 表达式对象。

`Column.default` 和 `Column.onupdate` 指定的默认值和更新 SQL 表达式在插入或更新语句发生时由 SQLAlchemy 明确调用，通常在 DML 语句中内联渲染，除了下面列出的特定情况。这与“服务器端”默认值不同，后者是表的 DDL 定义的一部分，例如作为“CREATE TABLE”语句的一部分，这种情况可能更常见。对于服务器端默认值，请参阅下一节 Server-invoked DDL-Explicit Default Expressions。

当通过 `Column.default` 指示的 SQL 表达式与主键列一起使用时，存在一些情况下，SQLAlchemy 必须“预执行”默认生成 SQL 函数，这意味着它在单独的 SELECT 语句中被调用，并且生成的值作为参数传递给 INSERT。这仅发生在要求返回此主键值的 INSERT 语句的主键列上，其中不能使用 RETURNING 或 `cursor.lastrowid`。指定了 `insert.inline` 标志的 `Insert` 构造将始终内联渲染默认表达式。

当语句使用单个参数集执行时（即不是“executemany”样式的执行），返回的`CursorResult`将包含通过`CursorResult.postfetch_cols()`可访问的集合，该集合包含所有具有内联执行默认值的`Column`对象的列表。同样，语句绑定的所有参数，包括预先执行的所有 Python 和 SQL 表达式，都存在于`CursorResult.last_inserted_params()`或`CursorResult.last_updated_params()`集合中的`CursorResult`上。`CursorResult.inserted_primary_key`集合包含插入行的主键值列表（列表格式，以便单列和复合列主键以相同的格式表示）。  ## 服务器调用的 DDL-显式默认表达式

SQL 表达式默认值的变体是`Column.server_default`，在`Table.create()`操作期间会放置在 CREATE TABLE 语句中：

```py
t = Table(
    "test",
    metadata_obj,
    Column("abc", String(20), server_default="abc"),
    Column("created_at", DateTime, server_default=func.sysdate()),
    Column("index_value", Integer, server_default=text("0")),
)
```

对于上述表格的创建调用将产生：

```py
CREATE  TABLE  test  (
  abc  varchar(20)  default  'abc',
  created_at  datetime  default  sysdate,
  index_value  integer  default  0
)
```

上面的示例说明了`Column.server_default`的两种典型用例，即 SQL 函数（在上面的示例中为 SYSDATE）以及服务器端常量值（在上面的示例中为整数“0”）。建议对任何文字 SQL 值使用`text()`构造，而不是传递原始值，因为 SQLAlchemy 通常不会对这些值执行任何引号添加或转义。

与客户端生成的表达式类似，`Column.server_default`可以适应一般的 SQL 表达式，但通常预期这些将是简单的函数和表达式，而不是像嵌入式 SELECT 这样更复杂的情况。  ## 标记隐式生成的值、时间戳和触发列

列在插入或更新时基于其他服务器端数据库机制生成新值，例如某些平台上的时间戳列所见的数据库特定的自动生成行为，以及在插入或更新时调用的自定义触发器生成新值，可以使用`FetchedValue`作为标记：

```py
from sqlalchemy.schema import FetchedValue

t = Table(
    "test",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    Column("abc", TIMESTAMP, server_default=FetchedValue()),
    Column("def", String(20), server_onupdate=FetchedValue()),
)
```

`FetchedValue` 指示符不会影响 CREATE TABLE 的渲染 DDL。相反，它标记列将在 INSERT 或 UPDATE 语句的过程中由数据库填充新值，并且对于支持的数据库，可以用于指示该列应该是 RETURNING 或 OUTPUT 子句的一部分。诸如 SQLAlchemy ORM 之类的工具随后利用此标记以了解如何在此类操作之后获取列的值。特别是，`ValuesBase.return_defaults()` 方法可与`Insert` 或`Update` 构造一起使用，以指示应返回这些值。

有关在 ORM 中使用`FetchedValue` 的详细信息，请参阅获取服务器生成的默认值。

警告

`Column.server_onupdate` 指令**目前不会**生成 MySQL 的“ON UPDATE CURRENT_TIMESTAMP()”子句。请参阅为 MySQL / MariaDB 的 explicit_defaults_for_timestamp 渲染 ON UPDATE CURRENT TIMESTAMP 了解如何生成此子句的背景信息。

另请参阅

获取服务器生成的默认值  ## 定义序列

SQLAlchemy 使用`Sequence` 对象表示数据库序列，被视为“列默认值”的特例。它仅对显式支持序列的数据库产生影响，其中包括 PostgreSQL、Oracle、MS SQL Server 和 MariaDB 在内的 SQLAlchemy 包含的方言。否则，`Sequence` 对象将被忽略。

提示

在较新的数据库引擎中，应该优先使用`Identity` 构造而不是`Sequence` 生成整数主键值。请参阅 Identity Columns (GENERATED { ALWAYS | BY DEFAULT } AS IDENTITY) 了解此构造的背景信息。

`Sequence`可以放置在任何列上作为“默认”生成器，在 INSERT 操作期间使用，并且如果需要，还可以配置为在 UPDATE 操作期间触发。它最常与单个整数主键列一起使用：

```py
table = Table(
    "cartitems",
    metadata_obj,
    Column(
        "cart_id",
        Integer,
        Sequence("cart_id_seq", start=1),
        primary_key=True,
    ),
    Column("description", String(40)),
    Column("createdate", DateTime()),
)
```

在上述情况中，表`cartitems`与名为`cart_id_seq`的序列相关联。对上述表执行`MetaData.create_all()`将包括：

```py
CREATE  SEQUENCE  cart_id_seq  START  WITH  1

CREATE  TABLE  cartitems  (
  cart_id  INTEGER  NOT  NULL,
  description  VARCHAR(40),
  createdate  TIMESTAMP  WITHOUT  TIME  ZONE,
  PRIMARY  KEY  (cart_id)
)
```

提示

当使用具有显式模式名称的表时（详见指定模式名称），`Table`的配置模式**不会**自动与嵌入的`Sequence`共享，而是需要指定`Sequence.schema`：

```py
Sequence("cart_id_seq", start=1, schema="some_schema")
```

`Sequence`还可以自动使用正在使用的`MetaData`中的`MetaData.schema`设置；有关背景，请参阅将序列与 MetaData 关联。

当针对`cartitems`表调用`Insert` DML 构造时，如果未传递`cart_id`列的显式值，则将使用`cart_id_seq`序列在参与的后端生成值。通常，序列函数被嵌入到 INSERT 语句中，与 RETURNING 结合使用，以便将新生成的值返回给 Python 进程：

```py
INSERT  INTO  cartitems  (cart_id,  description,  createdate)
VALUES  (next_val(cart_id_seq),  'some description',  '2015-10-15 12:00:15')
RETURNING  cart_id
```

当使用`Connection.execute()`调用`Insert`构造时，包括但不限于使用`Sequence`生成的新生成的主键标识符，可通过`CursorResult`构造的`CursorResult.inserted_primary_key`属性获取。

当`Sequence`与`Column`作为其**Python 端**默认生成器关联时，`Sequence`也将受到“CREATE SEQUENCE”和“DROP SEQUENCE” DDL 的影响，当为拥有的`Table`发出类似 DDL 时，比如使用`MetaData.create_all()`为一系列表生成 DDL 时。

`Sequence`也可以直接与`MetaData`构造关联。这允许`Sequence`同时用于多个`Table`，并且还允许继承`MetaData.schema`参数。请参见将序列与 MetaData 关联部分了解背景信息。

### 在 SERIAL 列上关联序列

PostgreSQL 的 SERIAL 数据类型是一个自增类型，意味着在发出 CREATE TABLE 时隐式创建一个 PostgreSQL 序列。当为`Column`指定`Sequence`构造时，可以通过为`Sequence.optional`参数指定`True`的值来指示在这种特定情况下不应使用它。这允许给定的`Sequence`用于没有其他主键生成系统的后端，但在后端（如 PostgreSQL）中会自动生成特定列的序列时忽略它：

```py
table = Table(
    "cartitems",
    metadata_obj,
    Column(
        "cart_id",
        Integer,
        # use an explicit Sequence where available, but not on
        # PostgreSQL where SERIAL will be used
        Sequence("cart_id_seq", start=1, optional=True),
        primary_key=True,
    ),
    Column("description", String(40)),
    Column("createdate", DateTime()),
)
```

在上面的示例中，对于 PostgreSQL 的`CREATE TABLE`将使用`SERIAL`数据类型来创建`cart_id`列，而`cart_id_seq`序列将被忽略。然而在 Oracle 上，`cart_id_seq`序列将被显式创建。

提示

SERIAL 和 SEQUENCE 的这种特定交互相当传统，就像其他情况一样，使用`Identity`将简化操作，只需在所有支持的后端上使用`IDENTITY`即可。

### 单独执行序列

SEQUENCE 是 SQL 中的一类一流模式对象，可用于独立生成数据库中的值。如果有一个`Sequence`对象，可以通过直接将其传递给 SQL 执行方法来调用其“next value”指令：

```py
with my_engine.connect() as conn:
    seq = Sequence("some_sequence", start=1)
    nextid = conn.execute(seq)
```

为了将`Sequence`的“next value”函数嵌入 SQL 语句（如 SELECT 或 INSERT）中，使用`Sequence.next_value()`方法，在语句编译时会生成适用于目标后端的 SQL 函数：

```py
>>> my_seq = Sequence("some_sequence", start=1)
>>> stmt = select(my_seq.next_value())
>>> print(stmt.compile(dialect=postgresql.dialect()))
SELECT  nextval('some_sequence')  AS  next_value_1 
```

### 将序列与 MetaData 关联

对于要与任意`Table`对象关联的`Sequence`，可以使用`Sequence.metadata`参数将`Sequence`与特定的`MetaData`关联：

```py
seq = Sequence("my_general_seq", metadata=metadata_obj, start=1)
```

这样的序列可以按照通常的方式与列关联：

```py
table = Table(
    "cartitems",
    metadata_obj,
    seq,
    Column("description", String(40)),
    Column("createdate", DateTime()),
)
```

在上面的示例中，`Sequence`对象被视为独立的模式构造，可以独立存在或在表之间共享。

明确将`Sequence`与`MetaData`关联允许以下行为：

+   `Sequence`将继承目标`MetaData`指定的`MetaData.schema`参数，这会影响 CREATE / DROP DDL 的生成以及`Sequence.next_value()`函数在 SQL 语句中的呈现方式。

+   `MetaData.create_all()` 和 `MetaData.drop_all()` 方法将发出 CREATE / DROP 命令，即使此 `Sequence` 没有与任何属于此 `MetaData` 的 `Table` / `Column` 关联。

### 将序列关联为服务器端默认值

注意

以下技术仅已知适用于 PostgreSQL 数据库。它不适用于 Oracle。

上文说明了如何将 `Sequence` 关联到 `Column` 作为 **Python 端的默认生成器**：

```py
Column(
    "cart_id",
    Integer,
    Sequence("cart_id_seq", metadata=metadata_obj, start=1),
    primary_key=True,
)
```

在上述情况下，当相关的 `Table` 被创建 / 删除时，`Sequence` 将自动受到 CREATE SEQUENCE / DROP SEQUENCE DDL 的影响。然而，当发出 CREATE TABLE 时，该序列不会作为列的服务器端默认值存在。

如果我们希望序列被用作服务器端默认值，即使我们从 SQL 命令行向表发出 INSERT 命令，它也会生效，我们可以使用 `Column.server_default` 参数与序列的值生成函数一起使用，该函数可从 `Sequence.next_value()` 方法中获得。下面我们展示了相同的 `Sequence` 与 `Column` 关联，既作为 Python 端的默认生成器，也作为服务器端的默认生成器：

```py
cart_id_seq = Sequence("cart_id_seq", metadata=metadata_obj, start=1)
table = Table(
    "cartitems",
    metadata_obj,
    Column(
        "cart_id",
        Integer,
        cart_id_seq,
        server_default=cart_id_seq.next_value(),
        primary_key=True,
    ),
    Column("description", String(40)),
    Column("createdate", DateTime()),
)
```

或者使用 ORM：

```py
class CartItem(Base):
    __tablename__ = "cartitems"

    cart_id_seq = Sequence("cart_id_seq", metadata=Base.metadata, start=1)
    cart_id = Column(
        Integer, cart_id_seq, server_default=cart_id_seq.next_value(), primary_key=True
    )
    description = Column(String(40))
    createdate = Column(DateTime)
```

当发出“CREATE TABLE”语句时，在 PostgreSQL 上，它将被表述为：

```py
CREATE  TABLE  cartitems  (
  cart_id  INTEGER  DEFAULT  nextval('cart_id_seq')  NOT  NULL,
  description  VARCHAR(40),
  createdate  TIMESTAMP  WITHOUT  TIME  ZONE,
  PRIMARY  KEY  (cart_id)
)
```

在 Python 端和服务器端默认生成上下文中放置`Sequence`可确保“主键获取”逻辑在所有情况下都能正常工作。通常，支持序列的数据库也支持 INSERT 语句的 RETURNING，当发出此语句时，SQLAlchemy 会自动使用。但是，如果对于特定的插入操作不使用 RETURNING，则 SQLAlchemy 更倾向于在 INSERT 语句之外“预执行”序列，这仅在将序列包含为 Python 端默认生成函数时才有效。

该示例还直接将`Sequence`与封闭的`MetaData`关联，这再次确保`Sequence`与`MetaData`集合的参数完全关联，包括默认模式（如果有）。

另请参阅

Sequences/SERIAL/IDENTITY - 在 PostgreSQL 方言文档中

RETURNING 支持 - 在 Oracle 方言文档中 ## 计算列 (GENERATED ALWAYS AS)

版本 1.3.11 中的新功能。

`Computed` 构造允许在 DDL 中声明一个“GENERATED ALWAYS AS”列的`Column`，即一个由数据库服务器计算值的列。该构造接受一个 SQL 表达式，通常使用字符串或`text()`构造进行文本声明，类似于`CheckConstraint`的方式。然后数据库服务器解释 SQL 表达式以确定行内列的值。

示例：

```py
from sqlalchemy import Table, Column, MetaData, Integer, Computed

metadata_obj = MetaData()

square = Table(
    "square",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    Column("side", Integer),
    Column("area", Integer, Computed("side * side")),
    Column("perimeter", Integer, Computed("4 * side")),
)
```

在 PostgreSQL 12 后端运行时，`square` 表的 DDL 如下所示：

```py
CREATE  TABLE  square  (
  id  SERIAL  NOT  NULL,
  side  INTEGER,
  area  INTEGER  GENERATED  ALWAYS  AS  (side  *  side)  STORED,
  perimeter  INTEGER  GENERATED  ALWAYS  AS  (4  *  side)  STORED,
  PRIMARY  KEY  (id)
)
```

值在 INSERT 和 UPDATE 时是否持久化，或者在获取时是否计算，是数据库的实现细节；前者称为“stored”，后者称为“virtual”。一些数据库实现支持两者，但有些只支持其中一个。可以指定可选的`Computed.persisted`标志为 `True` 或 `False`，以指示在 DDL 中是否应该呈现“STORED”或“VIRTUAL”关键字，但是如果目标后端不支持该关键字，则会引发错误；如果不设置，将为目标后端使用一个有效的默认值。

`Computed` 构造是 `FetchedValue` 对象的子类，将自己设置为目标 `Column` 的“服务器默认值”和“服务器更新”生成器，这意味着在生成 INSERT 和 UPDATE 语句时，它将被视为默认生成列，并且在使用 ORM 时将被作为生成列获取。这包括在数据库支持 RETURNING 且要急切地获取生成值时，它将成为数据库的 RETURNING 子句的一部分。

注意

使用 `Computed` 构造定义的 `Column` 可能不会存储除服务器应用于其外的任何值； 当为此类列传递值以在 INSERT 或 UPDATE 中写入时，SQLAlchemy 的行为目前是忽略该值。

“GENERATED ALWAYS AS” 目前已知受到支持的数据库有：

+   MySQL 版本 5.7 及更高版本

+   MariaDB 10.x 系列及更高版本

+   PostgreSQL 版本 12 及更高版本

+   Oracle - 值得注意的是，RETURNING 在 UPDATE 中不正确地工作（当包含计算列的 UPDATE..RETURNING 被渲染时，会发出此警告）

+   Microsoft SQL Server

+   SQLite 版本 3.31

当 `Computed` 与不受支持的后端一起使用时，如果目标方言不支持它，则在尝试渲染构造时会引发 `CompileError`。否则，如果方言支持它但正在使用的特定数据库服务器版本不支持它，则在将 DDL 发送到数据库时会引发 `DBAPIError` 的子类，通常是 `OperationalError`。

另请参阅

`Computed`  ## 标识列（GENERATED { ALWAYS | BY DEFAULT } AS IDENTITY）

版本 1.4 中新增。

`Identity` 构造允许将 `Column` 声明为标识列，并在 DDL 中渲染为 “GENERATED { ALWAYS | BY DEFAULT } AS IDENTITY”。标识列的值是由数据库服务器自动使用递增（或递减）序列生成的。该构造与 `Sequence` 共享控制数据库行为的大多数选项。

示例：

```py
from sqlalchemy import Table, Column, MetaData, Integer, Identity, String

metadata_obj = MetaData()

data = Table(
    "data",
    metadata_obj,
    Column("id", Integer, Identity(start=42, cycle=True), primary_key=True),
    Column("data", String),
)
```

在 PostgreSQL 12 后端运行时，`data` 表的 DDL 如下所示：

```py
CREATE  TABLE  data  (
  id  INTEGER  GENERATED  BY  DEFAULT  AS  IDENTITY  (START  WITH  42  CYCLE)  NOT  NULL,
  data  VARCHAR,
  PRIMARY  KEY  (id)
)
```

数据库将在插入时为 `id` 列生成值，从 `42` 开始，如果语句中尚未包含 `id` 列的值。标识列还可以要求数据库生成列的值，忽略语句中传递的值或引发错误，具体取决于后端。要激活此模式，请在 `Identity` 构造中将参数 `Identity.always` 设置为 `True`。更新前面的示例以包含此参数将生成以下 DDL：

```py
CREATE  TABLE  data  (
  id  INTEGER  GENERATED  ALWAYS  AS  IDENTITY  (START  WITH  42  CYCLE)  NOT  NULL,
  data  VARCHAR,
  PRIMARY  KEY  (id)
)
```

`Identity` 构造是 `FetchedValue` 对象的子类，并且将自身设置为目标 `Column` 的“服务器默认”生成器，这意味着当生成 INSERT 语句时，它将被视为默认生成列，以及在使用 ORM 时，它将被作为生成列提取。这包括对于支持 RETURNING 的数据库，它将成为 RETURNING 子句的一部分，并且生成的值将被急切地提取。

目前已知支持 `Identity` 构造的数据库有：

+   PostgreSQL 版本为 10。

+   Oracle 版本为 12。 还支持传递 `always=None` 以启用默认生成模式，并传递参数 `on_null=True` 以指定在使用 “BY DEFAULT” 标识列时“ON NULL”。

+   Microsoft SQL Server。 MSSQL 使用自定义语法，仅支持 `start` 和 `increment` 参数，并忽略所有其他参数。

当与不受支持的后端一起使用时，`Identity` 将被忽略，并且将使用默认的 SQLAlchemy 逻辑自动递增列。

当 `Column` 同时指定 `Identity` 并将 `Column.autoincrement` 设置为 `False` 时，会引发错误。

另请参阅

`Identity`

## 默认对象 API

| 对象名称 | 描述 |
| --- | --- |
| ColumnDefault | 列的普通默认值。 |
| Computed | 定义生成列，即“GENERATED ALWAYS AS”语法。 |
| DefaultClause | 由 DDL 指定的默认列值。 |
| DefaultGenerator | 列 *默认* 值的基类。 |
| FetchedValue | 用于透明数据库端默认值的标记。 |
| Identity | 定义标识列，即“GENERATED { ALWAYS &#124; BY DEFAULT } AS IDENTITY”语法。 |
| Sequence | 表示命名的数据库序列。 |

```py
class sqlalchemy.schema.Computed
```

定义一个生成列，即“GENERATED ALWAYS AS”语法。

`Computed` 构造是一个内联构造，添加到 `Column` 对象的参数列表中：

```py
from sqlalchemy import Computed

Table('square', metadata_obj,
    Column('side', Float, nullable=False),
    Column('area', Float, Computed('side * side'))
)
```

详细信息请参阅下面链接的文档。

新于版本 1.3.11。

请参阅

计算列（GENERATED ALWAYS AS）

**成员**

__init__(), copy()

**类签名**

类`sqlalchemy.schema.Computed`（`sqlalchemy.schema.FetchedValue`，`sqlalchemy.schema.SchemaItem`）

```py
method __init__(sqltext: _DDLColumnArgument, persisted: bool | None = None) → None
```

构造一个 GENERATED ALWAYS AS DDL 构造，以配合 `Column`。

参数：

+   `sqltext` –

    包含列生成表达式的字符串，该表达式将直接使用，或者是一个 SQL 表达式构造，比如`text()`对象。如果以字符串形式给出，则将该对象转换为`text()`对象。

    警告

    传递给`Computed`的`Computed.sqltext`参数可以作为 Python 字符串参数传递，它将被视为**受信任的 SQL 文本**并按给定方式呈现。**请勿将不受信任的输入传递给此参数**。

+   `persisted` –

    可选，控制该列在数据库中的持久化方式。可能的值包括：

    +   `None`，默认值，它将使用数据库定义的默认持久化方式。

    +   `True`，将呈现 `GENERATED ALWAYS AS ... STORED`，或者如果目标数据库支持的话，将呈现等效值。

    +   `False`，将呈现 `GENERATED ALWAYS AS ... VIRTUAL`，或者如果目标数据库支持的话，将呈现等效值。

    如果数据库不支持该持久化选项，则指定 `True` 或 `False` 可能会在将 DDL 发出到目标数据库时引发错误。将此参数保留为其默认值`None` 保证在所有支持 `GENERATED ALWAYS AS` 的数据库上都可以成功。

```py
method copy(*, target_table: Table | None = None, **kw: Any) → Computed
```

自版本 1.4 起已弃用：`Computed.copy()` 方法已弃用，并将在将来的版本中删除。

```py
class sqlalchemy.schema.ColumnDefault
```

列上的纯默认值。

这可能对应于一个常量、一个可调用函数或一个 SQL 子句。

`ColumnDefault` 在 `Column` 的 `default`、`onupdate` 参数被使用时会自动生成。一个 `ColumnDefault` 也可以作为位置参数传递。

例如，以下内容：

```py
Column('foo', Integer, default=50)
```

等价于：

```py
Column('foo', Integer, ColumnDefault(50))
```

**类签名**

类 `sqlalchemy.schema.ColumnDefault`（`sqlalchemy.schema.DefaultGenerator`，`abc.ABC`）

```py
class sqlalchemy.schema.DefaultClause
```

DDL 指定的 DEFAULT 列值。

`DefaultClause` 是一个 `FetchedValue`，当发出 “CREATE TABLE” 时也会生成一个“DEFAULT”子句。

当 `Column` 的 `server_default`、`server_onupdate` 参数被使用时，`DefaultClause` 会自动生成。一个 `DefaultClause` 也可以作为位置参数传递。

例如，以下内容：

```py
Column('foo', Integer, server_default="50")
```

等价于：

```py
Column('foo', Integer, DefaultClause("50"))
```

**类签名**

类 `sqlalchemy.schema.DefaultClause`（`sqlalchemy.schema.FetchedValue`）

```py
class sqlalchemy.schema.DefaultGenerator
```

列默认值的基类。

此对象仅出现在 column.default 或 column.onupdate。它不能作为服务器默认值有效。

**类签名**

类 `sqlalchemy.schema.DefaultGenerator`（`sqlalchemy.sql.expression.Executable`，`sqlalchemy.schema.SchemaItem`）

```py
class sqlalchemy.schema.FetchedValue
```

用于透明数据库端默认值的标记。

当数据库配置为为列提供一些自动默认值时，请使用 `FetchedValue`。

例如：

```py
Column('foo', Integer, FetchedValue())
```

将指示某个触发器或默认生成器在 INSERT 期间为 `foo` 列创建一个新值。

另请参阅

标记隐式生成的值、时间戳和触发列

**类签名**

类 `sqlalchemy.schema.FetchedValue`（`sqlalchemy.sql.expression.SchemaEventTarget`）

```py
class sqlalchemy.schema.Sequence
```

表示一个命名的数据库序列。

`Sequence` 对象表示数据库序列的名称和配置参数。它还表示一个可以被 SQLAlchemy `Engine` 或 `Connection` “执行”的结构，为目标数据库渲染适当的“下一个值”函数并返回结果。

`Sequence` 通常与主键列关联：

```py
some_table = Table(
    'some_table', metadata,
    Column('id', Integer, Sequence('some_table_seq', start=1),
    primary_key=True)
)
```

当为上述 `Table` 发射 CREATE TABLE 时，如果目标平台支持序列，则也会发射 CREATE SEQUENCE 语句。对于不支持序列的平台，将忽略 `Sequence` 构造。

另请参阅

定义序列

`CreateSequence`

`DropSequence`

**成员**

__init__(), create(), drop(), next_value()

**类签名**

类 `sqlalchemy.schema.Sequence` (`sqlalchemy.schema.HasSchemaAttr`, `sqlalchemy.schema.IdentityOptions`, `sqlalchemy.schema.DefaultGenerator`)

```py
method __init__(name: str, start: int | None = None, increment: int | None = None, minvalue: int | None = None, maxvalue: int | None = None, nominvalue: bool | None = None, nomaxvalue: bool | None = None, cycle: bool | None = None, schema: str | Literal[SchemaConst.BLANK_SCHEMA] | None = None, cache: int | None = None, order: bool | None = None, data_type: _TypeEngineArgument[int] | None = None, optional: bool = False, quote: bool | None = None, metadata: MetaData | None = None, quote_schema: bool | None = None, for_update: bool = False) → None
```

构建一个 `Sequence` 对象。

参数：

+   `name` – 序列的名称。

+   `start` – 开始索引。

    序列的起始索引。在将 CREATE SEQUENCE 命令发射到数据库时，此值被用作“START WITH”子句的值。如果为 `None`，则省略该子句，大多数平台上表示起始值为 1。

    从版本 2.0 开始更改：要求 `Sequence.start` 参数以发射 DDL “START WITH”。这是在版本 1.4 中进行的一项更改的反转，该更改如果未包括 `Sequence.start` 将隐式渲染“START WITH 1”。有关更多详细信息，请参阅 序列结构还原为没有任何显式默认“开始”值；影响 MS SQL Server。

+   `increment` – 序列的增量值。在将 CREATE SEQUENCE 命令发射到数据库时，此值被用作“INCREMENT BY”子句的值。如果为 `None`，则省略该子句，大多数平台上表示增量为 1。

+   `minvalue` – 序列的最小值。当向数据库发出 CREATE SEQUENCE 命令时，此值用作“MINVALUE”子句的值。如果为 `None`，则省略该子句，在大多数平台上表示升序和降序序列的最小值分别为 1 和 -2⁶³-1。

+   `maxvalue` – 序列的最大值。当向数据库发出 CREATE SEQUENCE 命令时，此值用作“MAXVALUE”子句的值。如果为 `None`，则省略该子句，在大多数平台上表示升序和降序序列的最大值分别为 2⁶³-1 和 -1。

+   `nominvalue` – 序列没有最小值。当向数据库发出 CREATE SEQUENCE 命令时，此值用作“NO MINVALUE”子句的值。如果为 `None`，则省略该子句，在大多数平台上表示升序和降序序列的最小值分别为 1 和 -2⁶³-1。

+   `nomaxvalue` – 序列没有最大值。当向数据库发出 CREATE SEQUENCE 命令时，此值用作“NO MAXVALUE”子句的值。如果为 `None`，则省略该子句，在大多数平台上表示升序和降序序列的最大值分别为 2⁶³-1 和 -1。

+   `cycle` – 当升序或降序序列达到最大值或最小值时，允许序列环绕。当向数据库发出 CREATE SEQUENCE 命令时，此值用作“CYCLE”子句。如果达到限制，则下一个生成的数字将是最小值或最大值，分别是升序或降序。如果 cycle=False（默认值），在序列达到其最大值后，任何对 nextval 的调用都将返回错误。

+   `schema` – 序列的可选模式名称，如果位于默认模式之外的模式中。当 `MetaData` 也存在时，选择模式名称的规则与 `Table.schema` 相同。

+   `cache` – 可选整数值；提前计算序列中的未来值的数量。呈现 Oracle 和 PostgreSQL 可理解的 CACHE 关键字。

+   `order` – 可选布尔值；如果为 `True`，则呈现 ORDER 关键字，Oracle 可理解，表示序列是有序的。可能需要使用 Oracle RAC 提供确定性排序。

+   `data_type` –

    序列返回的类型，适用于允许我们在 INTEGER、BIGINT 等之间进行选择的方言（例如，mssql）。

    新版本 1.4.0 中新增。

+   `optional` – 布尔值，当为 `True` 时，表示此 `Sequence` 对象只需在没有其他方法生成主键标识符的后端上显式生成。目前，这实际上意味着，“在 PostgreSQL 后端上不要创建此序列，因为 SERIAL 关键字会自动为我们创建序列”。

+   `quote` – 布尔值，当为 `True` 或 `False` 时，明确地强制引用 `Sequence.name` 的名称，打开或关闭。当保留其默认值 `None` 时，基于大小写和保留字的常规引用规则生效。

+   `quote_schema` – 设置对 `schema` 名称的引用首选项。

+   `metadata` –

    可选的 `MetaData` 对象，此 `Sequence` 将与之关联。与 `MetaData` 关联的 `Sequence` 具有以下功能：

    +   `Sequence` 将继承指定给目标 `MetaData` 的 `MetaData.schema` 参数，这会影响 CREATE / DROP DDL 的生成（如果有）。

    +   `Sequence.create()` 和 `Sequence.drop()` 方法会自动使用绑定到 `MetaData` 对象的引擎（如果有）。

    +   `MetaData.create_all()` 和 `MetaData.drop_all()` 方法将为此 `Sequence` 发出 CREATE / DROP，即使此 `Sequence` 未与此 `MetaData` 的任何成员 `Table` / `Column` 关联。

    上述行为只有在通过此参数将 `Sequence` 明确关联到 `MetaData` 时才会发生。

    另请参阅

    将序列与 MetaData 关联 - 关于`Sequence.metadata`参数的详细讨论。

+   `for_update` – 表示当与`Column`关联时，此`Sequence`应在该列的表上进行 UPDATE 语句调用，而不是在该语句中否则在该列中没有值。

```py
method create(bind: _CreateDropBind, checkfirst: bool = True) → None
```

在数据库中创建此序列。

```py
method drop(bind: _CreateDropBind, checkfirst: bool = True) → None
```

从数据库中删除此序列。

```py
method next_value() → Function[int]
```

返回一个`next_value`函数元素，该函数将为此`Sequence`在任何 SQL 表达式中呈现适当的增量函数。

```py
class sqlalchemy.schema.Identity
```

定义一个标识列，即“GENERATED { ALWAYS | BY DEFAULT } AS IDENTITY”语法。

`Identity`结构是添加到`Column`对象的参数列表中的内联结构：

```py
from sqlalchemy import Identity

Table('foo', metadata_obj,
    Column('id', Integer, Identity())
    Column('description', Text),
)
```

请参阅下面链接的文档以获取完整详情。

从版本 1.4 开始新增。

另请参阅

标识列（GENERATED { ALWAYS | BY DEFAULT } AS IDENTITY）

**成员**

__init__(), copy()

**类签名**

类`sqlalchemy.schema.Identity`（`sqlalchemy.schema.IdentityOptions`，`sqlalchemy.schema.FetchedValue`，`sqlalchemy.schema.SchemaItem`）

```py
method __init__(always: bool = False, on_null: bool | None = None, start: int | None = None, increment: int | None = None, minvalue: int | None = None, maxvalue: int | None = None, nominvalue: bool | None = None, nomaxvalue: bool | None = None, cycle: bool | None = None, cache: int | None = None, order: bool | None = None) → None
```

创建一个 DDL 结构`GENERATED { ALWAYS | BY DEFAULT } AS IDENTITY`，用于配合`Column`。

请参阅`Sequence`文档，了解大多数参数的完整描述。

注意

MSSQL 支持此结构作为在列上生成 IDENTITY 的首选替代方案，但它使用的非标准语法仅支持`Identity.start`和`Identity.increment`。所有其他参数都会被忽略。

参数：

+   `always` – 一个布尔值，表示标识列的类型。如果指定了`False`，则默认情况下用户指定的值优先。如果指定了`True`，则不接受用户指定的值（在某些后端，如 PostgreSQL，可以在插入时指定 OVERRIDING SYSTEM VALUE 或类似语句以覆盖序列值）。某些后端也对此参数有默认值，`None`可以用于在 DDL 中省略渲染此部分。如果后端没有默认值，则将其视为`False`。

+   `on_null` – 设置为`True`以与`always=False`标识列一起指定 ON NULL。此选项仅在某些后端（如 Oracle）上受支持。

+   `start` – 序列的起始索引。

+   `increment` – 序列的增量值。

+   `minvalue` – 序列的最小值。

+   `maxvalue` – 序列的最大值。

+   `nominvalue` – 序列的最小值不存在。

+   `nomaxvalue` – 序列的最大值不存在。

+   `cycle` – 当达到最大值或最小值时允许序列循环。

+   `cache` – 可选的整数值；提前计算的序列中未来值的数量。

+   `order` – 可选的布尔值；如果为 true，则呈现 ORDER 关键字。

```py
method copy(**kw: Any) → Identity
```

自版本 1.4 起已弃用：`Identity.copy()`方法已弃用，将在将来的版本中移除。

## 标量默认值

最简单的默认值是作为列默认值使用的标量值：

```py
Table("mytable", metadata_obj, Column("somecolumn", Integer, default=12))
```

上述，如果未提供其他值，则值“12”将在插入时绑定为列值。

标量值也可能与 UPDATE 语句关联，尽管这不是很常见（因为 UPDATE 语句通常寻找动态默认值）：

```py
Table("mytable", metadata_obj, Column("somecolumn", Integer, onupdate=25))
```

## Python 执行的函数

`Column.default`和`Column.onupdate`关键字参数也接受 Python 函数。如果未提供该列的其他值，则在插入或更新时调用这些函数，并使用返回的值作为列的值。以下示例说明了一个粗糙的“序列”，将递增计数器分配给主键列：

```py
# a function which counts upwards
i = 0

def mydefault():
    global i
    i += 1
    return i

t = Table(
    "mytable",
    metadata_obj,
    Column("id", Integer, primary_key=True, default=mydefault),
)
```

应该注意，对于真正的“增量序列”行为，通常应该使用数据库的内置功能，这可能包括序列对象或其他自增能力。对于主键列，SQLAlchemy 在大多数情况下将自动使用这些功能。请参阅`Column`的 API 文档，包括`Column.autoincrement`标志，以及本章后面关于`Sequence`的部分，了解标准主键生成技术的背景。

为了说明 onupdate，我们将 Python 的`datetime`函数`now`赋值给`Column.onupdate`属性：

```py
import datetime

t = Table(
    "mytable",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    # define 'last_updated' to be populated with datetime.now()
    Column("last_updated", DateTime, onupdate=datetime.datetime.now),
)
```

当执行更新语句并且未传递`last_updated`值时，将执行`datetime.datetime.now()` Python 函数，并将其返回值用作`last_updated`的值。请注意，我们将`now`作为函数本身提供，而不调用它（即后面没有括号）- SQLAlchemy 将在执行语句时执行该函数。

### 上下文敏感的默认函数

`Column.default`和`Column.onupdate`使用的 Python 函数也可以利用当前语句上下文来确定值。语句的上下文是一个内部的 SQLAlchemy 对象，包含有关正在执行的语句的所有信息，包括其源表达式、与之关联的参数和游标。在默认生成的上下文中，典型的用例是访问正在插入或更新的行上的其他值。要访问上下文，请提供一个接受单个`context`参数的函数：

```py
def mydefault(context):
    return context.get_current_parameters()["counter"] + 12

t = Table(
    "mytable",
    metadata_obj,
    Column("counter", Integer),
    Column("counter_plus_twelve", Integer, default=mydefault, onupdate=mydefault),
)
```

上述默认生成函数是这样应用的：它将用于所有未提供`counter_plus_twelve`值的 INSERT 和 UPDATE 语句，并且该值将是执行`counter`列时的任何值加上 12 的值。

对于使用“executemany”样式执行的单个语句，例如通过`Connection.execute()`传递多个参数集的情况，用户定义的函数将为每组参数集调用一次。对于多值`Insert`结构的用例（例如通过`Insert.values()`方法设置了多个 VALUES 子句），用户定义的函数也将为每组参数集调用一次。

当函数被调用时，上下文对象（`DefaultExecutionContext`的一个子类）中可用的特殊方法`DefaultExecutionContext.get_current_parameters()`。此方法返回一个字典，其中键-值对表示 INSERT 或 UPDATE 语句的完整值集。在多值 INSERT 结构的情况下，与单个 VALUES 子句对应的参数子集将从完整参数字典中隔离并单独返回。

1.2 版中的新功能：增加了`DefaultExecutionContext.get_current_parameters()`方法，它改进了仍然存在的`DefaultExecutionContext.current_parameters`属性，通过提供将多个 VALUES 子句组织成单独参数字典的服务。### 上下文敏感的默认函数

由`Column.default`和`Column.onupdate`使用的 Python 函数还可以利用当前语句的上下文来确定一个值。语句的上下文是一个内部的 SQLAlchemy 对象，其中包含关于正在执行的语句的所有信息，包括其源表达式、与之关联的参数和游标。与默认生成相关的此上下文的典型用例是访问要插入或更新的行上的其他值。要访问上下文，请提供一个接受单个`context`参数的函数：

```py
def mydefault(context):
    return context.get_current_parameters()["counter"] + 12

t = Table(
    "mytable",
    metadata_obj,
    Column("counter", Integer),
    Column("counter_plus_twelve", Integer, default=mydefault, onupdate=mydefault),
)
```

上述默认生成函数被应用于所有 INSERT 和 UPDATE 语句，其中未提供`counter_plus_twelve`的值，并且该值将为执行`counter`列时存在的任何值加上 12。

对于使用“executemany”风格执行的单个语句，例如向 `Connection.execute()` 传递多个参数集的情况，用户定义的函数会为每个参数集调用一次。对于多值 `Insert` 结构的用例（例如通过 `Insert.values()` 方法设置了多个 VALUES 子句），用户定义的函数也会为每个参数集调用一次。

当函数被调用时，上下文对象（`DefaultExecutionContext`的子类）中可用特殊方法`DefaultExecutionContext.get_current_parameters()`。该方法返回一个字典，列键到值的映射，表示 INSERT 或 UPDATE 语句的完整值集。对于多值 INSERT 结构的情况，与单个 VALUES 子句对应的参数子集被从完整参数字典中隔离并单独返回。

1.2 版本中新增：增加了 `DefaultExecutionContext.get_current_parameters()` 方法，它改进了仍然存在的 `DefaultExecutionContext.current_parameters` 属性，提供了将多个 VALUES 子句组织成单独参数字典的服务。

## 客户端调用的 SQL 表达式

也可以传递 SQL 表达式给 `Column.default` 和 `Column.onupdate` 关键字，这在大多数情况下会在 INSERT 或 UPDATE 语句中内联呈现：

```py
t = Table(
    "mytable",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    # define 'create_date' to default to now()
    Column("create_date", DateTime, default=func.now()),
    # define 'key' to pull its default from the 'keyvalues' table
    Column(
        "key",
        String(20),
        default=select(keyvalues.c.key).where(keyvalues.c.type="type1"),
    ),
    # define 'last_modified' to use the current_timestamp SQL function on update
    Column("last_modified", DateTime, onupdate=func.utc_timestamp()),
)
```

在上述情况中，`create_date`列将在 INSERT 语句期间填充 `now()` SQL 函数的结果（在大多数情况下，根据后端编译为 `NOW()` 或 `CURRENT_TIMESTAMP`），而 `key` 列将填充另一张表的 SELECT 子查询的结果。当为此表发出 UPDATE 语句时，`last_modified` 列将填充 SQL `UTC_TIMESTAMP()` MySQL 函数的值。

注意

使用`func`构造和 SQL 函数时，我们会“调用”命名函数，例如在 `func.now()` 中使用括号。这与当我们将 Python 可调用对象（例如 `datetime.datetime`）指定为默认值时不同，其中我们传递函数本身，但我们不自己调用它。对于 SQL 函数，调用`func.now()`将返回将“NOW”函数呈现为正在发出的 SQL 中的 SQL 表达式对象。

当发生 INSERT 或 UPDATE 语句时，SQLAlchemy 在 `Column.default` 和 `Column.onupdate` 指定的默认和更新 SQL 表达式明确调用它们，通常在 DML 语句中内联渲染，除了下面列出的某些情况。这与“服务器端”默认值不同，后者是表的 DDL 定义的一部分，例如作为“CREATE TABLE”语句的一部分，这可能更常见。有关服务器端默认值，请参阅下一节服务器调用 DDL-显式默认表达式。

当由`Column.default`指示的 SQL 表达式与主键列一起使用时，有些情况下 SQLAlchemy 必须“预先执行”默认生成的 SQL 函数，这意味着它在单独的 SELECT 语句中被调用，并且生成的值作为参数传递给 INSERT。这仅发生在主键列为 INSERT 语句被要求返回该主键值的情况下，其中不能使用 RETURNING 或 `cursor.lastrowid`。指定了`insert.inline`标志的 `Insert` 构造将始终将默认表达式呈现为内联。

当执行语句使用单一参数集合（即不是“executemany”风格执行）时，返回的`CursorResult`将包含一个可通过`CursorResult.postfetch_cols()`访问的集合，其中包含所有具有内联执行默认值的`Column`对象的列表。同样，绑定到语句的所有参数，包括所有预先执行的 Python 和 SQL 表达式，都存在于`CursorResult.last_inserted_params()`或`CursorResult.last_updated_params()`集合中的`CursorResult`。`CursorResult.inserted_primary_key`集合包含插入的行的主键值列表（列表使得单列和复合列主键以相同格式表示）。

## 服务器调用的 DDL 显式默认表达式

SQL 表达式默认的一种变体是`Column.server_default`，在`Table.create()`操作期间会被放置在 CREATE TABLE 语句中：

```py
t = Table(
    "test",
    metadata_obj,
    Column("abc", String(20), server_default="abc"),
    Column("created_at", DateTime, server_default=func.sysdate()),
    Column("index_value", Integer, server_default=text("0")),
)
```

对上述表的创建调用将产生：

```py
CREATE  TABLE  test  (
  abc  varchar(20)  default  'abc',
  created_at  datetime  default  sysdate,
  index_value  integer  default  0
)
```

上述示例说明了`Column.server_default`的两种典型用法，即 SQL 函数（上述示例中的 SYSDATE）以及服务器端常量值（上述示例中的整数“0”）。建议对任何文字 SQL 值使用`text()`构造，而不是传递原始值，因为 SQLAlchemy 通常不对这些值执行任何引用或转义。

与客户端生成的表达式类似，`Column.server_default`可以容纳一般的 SQL 表达式，但是预期这些通常会是简单的函数和表达式，而不是更复杂的情况，比如嵌套的 SELECT。

## 标记隐式生成的值、时间戳和触发列

当插入或更新时，基于其他服务器端数据库机制生成新值的列，例如在某些平台上与时间戳列一起看到的数据库特定的自动生成行为，以及在插入或更新时调用的自定义触发器以生成新值，可以使用`FetchedValue`作为标记：

```py
from sqlalchemy.schema import FetchedValue

t = Table(
    "test",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    Column("abc", TIMESTAMP, server_default=FetchedValue()),
    Column("def", String(20), server_onupdate=FetchedValue()),
)
```

`FetchedValue` 指示器不会影响 CREATE TABLE 的渲染 DDL。相反，它标记了在 INSERT 或 UPDATE 语句的过程中由数据库填充新值的列，并且对于支持的数据库，可能会用于指示该列应该是 RETURNING 或 OUTPUT 子句的一部分。然后，诸如 SQLAlchemy ORM 之类的工具使用此标记来了解如何获取此类操作后列的值。特别是，可以使用`ValuesBase.return_defaults()`方法与`Insert`或`Update`构造来指示应返回这些值。

有关使用 ORM 中的`FetchedValue`的详细信息，请参阅获取服务器生成的默认值。

警告

`Column.server_onupdate` 指令目前**不会**生成 MySQL 的“ON UPDATE CURRENT_TIMESTAMP()”子句。有关如何生成此子句的背景信息，请参阅为 MySQL / MariaDB 的 explicit_defaults_for_timestamp 渲染 ON UPDATE CURRENT TIMESTAMP。

另请参阅

获取服务器生成的默认值

## 定义序列

SQLAlchemy 使用`Sequence`对象表示数据库序列，这被认为是“列默认值”的特殊情况。它仅对具有对序列的明确支持的数据库产生影响，其中包括 SQLAlchemy 包含的方言中的 PostgreSQL、Oracle、MS SQL Server 和 MariaDB。否则，`Sequence`对象将被忽略。

提示

在较新的数据库引擎中，应该优先使用`Identity`构造生成整数主键值，而不是`Sequence`。有关此构造的背景信息，请参阅 Identity Columns (GENERATED { ALWAYS | BY DEFAULT } AS IDENTITY)。

`Sequence`可以放置在任何列上作为“默认”生成器，在 INSERT 操作期间使用，并且如果需要，还可以配置在 UPDATE 操作期间触发。它通常与单个整数主键列一起使用：

```py
table = Table(
    "cartitems",
    metadata_obj,
    Column(
        "cart_id",
        Integer,
        Sequence("cart_id_seq", start=1),
        primary_key=True,
    ),
    Column("description", String(40)),
    Column("createdate", DateTime()),
)
```

在上述情况下，表`cartitems`与名为`cart_id_seq`的序列相关联。发出上述表的`MetaData.create_all()`将包括：

```py
CREATE  SEQUENCE  cart_id_seq  START  WITH  1

CREATE  TABLE  cartitems  (
  cart_id  INTEGER  NOT  NULL,
  description  VARCHAR(40),
  createdate  TIMESTAMP  WITHOUT  TIME  ZONE,
  PRIMARY  KEY  (cart_id)
)
```

提示

当使用具有显式模式名称的表（详细信息请参阅指定模式名称）时，`Table`的配置模式**不会**自动由嵌入的`Sequence`共享，而是需要指定`Sequence.schema`:

```py
Sequence("cart_id_seq", start=1, schema="some_schema")
```

`Sequence`也可以自动使用正在使用的`MetaData`的设置中的`MetaData.schema`；有关背景信息，请参阅将序列与 MetaData 关联。

当针对`cartitems`表调用`Insert` DML 构造时，如果没有为`cart_id`列传递显式值，则`cart_id_seq`序列将用于在参与的后端生成一个值。通常，序列函数嵌入在 INSERT 语句中，与 RETURNING 结合在一起，以便新生成的值可以返回给 Python 进程：

```py
INSERT  INTO  cartitems  (cart_id,  description,  createdate)
VALUES  (next_val(cart_id_seq),  'some description',  '2015-10-15 12:00:15')
RETURNING  cart_id
```

当使用`Connection.execute()`来调用`Insert`构造时，包括但不限于使用`Sequence`生成的新生成的主键标识符，可以通过`CursorResult`构造使用`CursorResult.inserted_primary_key`属性获得。

当将 `Sequence` 关联到 `Column` 作为其 **Python-side** 默认生成器时，当为拥有 `Table` 发出类似 DDL 的情况下，例如使用 `MetaData.create_all()` 为一系列表生成 DDL 时，该 `Sequence` 也将受到 “CREATE SEQUENCE” 和 “DROP SEQUENCE” DDL 的约束。

`Sequence` 也可以直接与 `MetaData` 构造关联。这允许 `Sequence` 在多个 `Table` 中同时使用，并且还允许继承 `MetaData.schema` 参数。有关详情，请参见 将序列与元数据关联 部分。

### 在 SERIAL 列上关联一个序列

PostgreSQL 的 SERIAL 数据类型是一种自增类型，意味着在发出 CREATE TABLE 命令时隐式创建了一个 PostgreSQL 序列。当为 `Column` 指定了 `Sequence` 构造时，可以通过将 `Sequence.optional` 参数的值设置为 `True` 来指示在这种特定情况下不应使用它。这允许给定的 `Sequence` 用于没有其他替代主键生成系统的后端，但对于自动为特定列生成序列的后端（例如 PostgreSQL），可以忽略它：

```py
table = Table(
    "cartitems",
    metadata_obj,
    Column(
        "cart_id",
        Integer,
        # use an explicit Sequence where available, but not on
        # PostgreSQL where SERIAL will be used
        Sequence("cart_id_seq", start=1, optional=True),
        primary_key=True,
    ),
    Column("description", String(40)),
    Column("createdate", DateTime()),
)
```

在上面的示例中，对于 PostgreSQL，`CREATE TABLE` 将使用 `SERIAL` 数据类型来创建 `cart_id` 列，并且 `cart_id_seq` 序列将被忽略。然而在 Oracle 中，`cart_id_seq` 序列将被显式创建。

提示

SERIAL 和 SEQUENCE 的这种特定交互在相当程度上是遗留的，在其他情况下，使用 `Identity` 将简化操作，只需在所有支持的后端上使用 `IDENTITY` 即可。

### 执行一个独立的序列

SEQUENCE 是 SQL 中的一种一流模式对象，可用于在数据库中独立生成值。如果有 `Sequence` 对象，可以通过直接将其传递给 SQL 执行方法来调用其“next value”指令：

```py
with my_engine.connect() as conn:
    seq = Sequence("some_sequence", start=1)
    nextid = conn.execute(seq)
```

为了将 `Sequence` 的“next value”函数嵌入到类似于 SELECT 或 INSERT 的 SQL 语句中，可以使用 `Sequence.next_value()` 方法，该方法将在语句编译时生成适用于目标后端的 SQL 函数：

```py
>>> my_seq = Sequence("some_sequence", start=1)
>>> stmt = select(my_seq.next_value())
>>> print(stmt.compile(dialect=postgresql.dialect()))
SELECT  nextval('some_sequence')  AS  next_value_1 
```

### 将序列与 MetaData 关联

对于将与任意 `Table` 对象关联的 `Sequence`，可以使用 `Sequence.metadata` 参数将 `Sequence` 关联到特定的 `MetaData`：

```py
seq = Sequence("my_general_seq", metadata=metadata_obj, start=1)
```

然后可以以通常的方式将这样的序列与列关联起来：

```py
table = Table(
    "cartitems",
    metadata_obj,
    seq,
    Column("description", String(40)),
    Column("createdate", DateTime()),
)
```

在上面的示例中，`Sequence` 对象被视为独立的模式构造，可以独立存在或在表之间共享。

明确将 `Sequence` 与 `MetaData` 关联允许以下行为：

+   `Sequence` 将继承指定给目标 `MetaData` 的 `MetaData.schema` 参数，这影响了 CREATE / DROP DDL 的生成以及 SQL 语句中 `Sequence.next_value()` 函数的呈现方式。

+   `MetaData.create_all()` 和 `MetaData.drop_all()` 方法将发出 CREATE / DROP 用于此 `Sequence`，即使该 `Sequence` 未与任何此 `MetaData` 的成员 `Table` / `Column` 相关联。

### 将序列关联为服务器端默认值

注意

以下技术仅在 PostgreSQL 数据库中有效，不适用于 Oracle。

前面的部分说明了如何将 `Sequence` 与 `Column` 关联为**Python 端默认生成器**：

```py
Column(
    "cart_id",
    Integer,
    Sequence("cart_id_seq", metadata=metadata_obj, start=1),
    primary_key=True,
)
```

在上述情况下，当相关的 `Table` 被 CREATE / DROP 时，`Sequence` 将自动受到 CREATE SEQUENCE / DROP SEQUENCE DDL 的影响。但是，当发出 CREATE TABLE 时，该序列不会作为列的服务器端默认值存在。

如果我们希望序列被用作服务器端默认值，即使我们从 SQL 命令行向表中发出 INSERT 命令，我们可以使用 `Column.server_default` 参数与序列的值生成函数一起使用，该函数可以从 `Sequence.next_value()` 方法中获取。下面我们将演示相同的 `Sequence` 作为 Python 端默认生成器以及服务器端默认生成器与 `Column` 相关联的情况：

```py
cart_id_seq = Sequence("cart_id_seq", metadata=metadata_obj, start=1)
table = Table(
    "cartitems",
    metadata_obj,
    Column(
        "cart_id",
        Integer,
        cart_id_seq,
        server_default=cart_id_seq.next_value(),
        primary_key=True,
    ),
    Column("description", String(40)),
    Column("createdate", DateTime()),
)
```

或者使用 ORM：

```py
class CartItem(Base):
    __tablename__ = "cartitems"

    cart_id_seq = Sequence("cart_id_seq", metadata=Base.metadata, start=1)
    cart_id = Column(
        Integer, cart_id_seq, server_default=cart_id_seq.next_value(), primary_key=True
    )
    description = Column(String(40))
    createdate = Column(DateTime)
```

当发出“CREATE TABLE”语句时，在 PostgreSQL 上，它将被生成为：

```py
CREATE  TABLE  cartitems  (
  cart_id  INTEGER  DEFAULT  nextval('cart_id_seq')  NOT  NULL,
  description  VARCHAR(40),
  createdate  TIMESTAMP  WITHOUT  TIME  ZONE,
  PRIMARY  KEY  (cart_id)
)
```

在 Python 端和服务器端默认生成上下文中放置 `Sequence` 确保“主键获取”逻辑在所有情况下都能正常工作。通常，启用序列的数据库还支持 INSERT 语句的 RETURNING，当发出此语句时，SQLAlchemy 会自动使用它。但是，如果对于特定的插入未使用 RETURNING，则 SQLAlchemy 更愿意在 INSERT 语句本身之外“预先执行”序列，只有在将序列作为 Python 端默认生成函数包含时才能正常工作。

该示例还将 `Sequence` 直接与封闭的 `MetaData` 关联起来，这再次确保 `Sequence` 完全与 `MetaData` 集合的参数相关联，包括默认模式（如果有）。

另请参阅

序列/SERIAL/IDENTITY - 在 PostgreSQL 方言文档中

返回支持 - 在 Oracle 方言文档中

### 将序列关联到 SERIAL 列

PostgreSQL 的 SERIAL 数据类型是一种自增类型，它意味着在发出 CREATE TABLE 时会隐式创建一个 PostgreSQL 序列。当为 `Column` 指定 `Sequence` 构造时，可以通过为 `Sequence.optional` 参数指定 `True` 值来表明在这种特定情况下不应使用它。这允许给定的 `Sequence` 用于没有其他替代主键生成系统的后端，但对于诸如 PostgreSQL 之类的后端，它会自动生成一个特定列的序列：

```py
table = Table(
    "cartitems",
    metadata_obj,
    Column(
        "cart_id",
        Integer,
        # use an explicit Sequence where available, but not on
        # PostgreSQL where SERIAL will be used
        Sequence("cart_id_seq", start=1, optional=True),
        primary_key=True,
    ),
    Column("description", String(40)),
    Column("createdate", DateTime()),
)
```

在上面的例子中，对于 PostgreSQL 的 `CREATE TABLE` 将使用 `SERIAL` 数据类型来创建 `cart_id` 列，而 `cart_id_seq` 序列将被忽略。然而，在 Oracle 中，`cart_id_seq` 序列将被显式创建。

提示

SERIAL 和 SEQUENCE 的这种特定交互相当古老，与其他情况一样，改用 `Identity` 将简化操作，只需在所有支持的后端上使用 `IDENTITY` 即可。

### 单独执行序列

SEQUENCE 是 SQL 中的一级模式对象，并且可以在数据库中独立生成值。如果你有一个 `Sequence` 对象，可以直接将其传递给 SQL 执行方法，通过其“next value”指令来调用它：

```py
with my_engine.connect() as conn:
    seq = Sequence("some_sequence", start=1)
    nextid = conn.execute(seq)
```

为了将 `Sequence` 的“next value”函数嵌入到类似 SELECT 或 INSERT 的 SQL 语句中，可以使用 `Sequence.next_value()` 方法，该方法将在语句编译时生成适合目标后端的 SQL 函数：

```py
>>> my_seq = Sequence("some_sequence", start=1)
>>> stmt = select(my_seq.next_value())
>>> print(stmt.compile(dialect=postgresql.dialect()))
SELECT  nextval('some_sequence')  AS  next_value_1 
```

### 将 Sequence 与 MetaData 关联起来

对于要与任意 `Table` 对象关联的 `Sequence`，可以使用 `Sequence.metadata` 参数将 `Sequence` 关联到特定的 `MetaData`：

```py
seq = Sequence("my_general_seq", metadata=metadata_obj, start=1)
```

这样的序列可以按照通常的方式与列关联起来：

```py
table = Table(
    "cartitems",
    metadata_obj,
    seq,
    Column("description", String(40)),
    Column("createdate", DateTime()),
)
```

在上面的例子中，`Sequence` 对象被视为一个独立的模式构造，可以独立存在或在表之间共享。

显式地将 `Sequence` 与 `MetaData` 关联起来，可以实现以下行为：

+   `Sequence` 会继承指定给目标 `MetaData` 的 `MetaData.schema` 参数，这会影响 CREATE / DROP DDL 的生成，以及 `Sequence.next_value()` 函数在 SQL 语句中的呈现方式。

+   `MetaData.create_all()` 和 `MetaData.drop_all()` 方法将为此 `Sequence` 发出 CREATE / DROP，即使该 `Sequence` 未与此 `MetaData` 的任何成员 `Table` / `Column` 关联。

### 将 Sequence 关联为服务器端默认

注意

以下技术仅在 PostgreSQL 数据库中可用。它在 Oracle 中不起作用。

前面的部分演示了如何将 `Sequence` 关联到 `Column` 作为**Python 端默认生成器**：

```py
Column(
    "cart_id",
    Integer,
    Sequence("cart_id_seq", metadata=metadata_obj, start=1),
    primary_key=True,
)
```

在上述情况下，当相关的 `Table` 要被创建 / 删除时，`Sequence` 将自动受到 CREATE SEQUENCE / DROP SEQUENCE DDL 的影响。但是，在发出 CREATE TABLE 时，该序列不会出现为该列的服务器端默认。

如果我们希望序列被用作服务器端默认，即使我们从 SQL 命令行向表中发出 INSERT 命令，我们可以使用 `Column.server_default` 参数，与序列的值生成函数一起使用，该函数可以从 `Sequence.next_value()` 方法获得。下面我们演示了相同的 `Sequence` 同时关联到 `Column`，既作为 Python 端的默认生成器，又作为服务器端的默认生成器：

```py
cart_id_seq = Sequence("cart_id_seq", metadata=metadata_obj, start=1)
table = Table(
    "cartitems",
    metadata_obj,
    Column(
        "cart_id",
        Integer,
        cart_id_seq,
        server_default=cart_id_seq.next_value(),
        primary_key=True,
    ),
    Column("description", String(40)),
    Column("createdate", DateTime()),
)
```

或使用 ORM：

```py
class CartItem(Base):
    __tablename__ = "cartitems"

    cart_id_seq = Sequence("cart_id_seq", metadata=Base.metadata, start=1)
    cart_id = Column(
        Integer, cart_id_seq, server_default=cart_id_seq.next_value(), primary_key=True
    )
    description = Column(String(40))
    createdate = Column(DateTime)
```

在发出“CREATE TABLE”语句时，在 PostgreSQL 上会发出：

```py
CREATE  TABLE  cartitems  (
  cart_id  INTEGER  DEFAULT  nextval('cart_id_seq')  NOT  NULL,
  description  VARCHAR(40),
  createdate  TIMESTAMP  WITHOUT  TIME  ZONE,
  PRIMARY  KEY  (cart_id)
)
```

在 Python 端和服务器端默认生成上下文中放置`Sequence`可以确保“主键提取”逻辑在所有情况下都有效。 通常，启用序列的数据库也支持对 INSERT 语句使用 RETURNING，当发出此语句时，SQLAlchemy 会自动使用它。 但是，如果对特定插入未使用 RETURNING，则 SQLAlchemy 更愿意在 INSERT 语句本身之外“预执行”序列，这仅在序列作为 Python 端默认生成器函数时有效。

该示例还将`Sequence`直接与封闭的`MetaData`关联起来，这再次确保了`Sequence`与`MetaData`集合的参数完全关联，包括默认模式（如果有）。

另请参阅

序列/SERIAL/IDENTITY - 在 PostgreSQL 方言文档中

RETURNING 支持 - 在 Oracle 方言文档中

## 计算列（GENERATED ALWAYS AS）

1.3.11 版中新增。

`Computed` 构造允许将 `Column` 声明为“GENERATED ALWAYS AS”列，在 DDL 中，即由数据库服务器计算值的列。 该构造接受一个 SQL 表达式，通常使用字符串或 `text()` 构造进行文本声明，类似于 `CheckConstraint` 的方式。 然后，数据库服务器会解释该 SQL 表达式，以确定行内列的值。

示例:

```py
from sqlalchemy import Table, Column, MetaData, Integer, Computed

metadata_obj = MetaData()

square = Table(
    "square",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    Column("side", Integer),
    Column("area", Integer, Computed("side * side")),
    Column("perimeter", Integer, Computed("4 * side")),
)
```

在 PostgreSQL 12 后端上运行 `square` 表的 DDL 如下所示：

```py
CREATE  TABLE  square  (
  id  SERIAL  NOT  NULL,
  side  INTEGER,
  area  INTEGER  GENERATED  ALWAYS  AS  (side  *  side)  STORED,
  perimeter  INTEGER  GENERATED  ALWAYS  AS  (4  *  side)  STORED,
  PRIMARY  KEY  (id)
)
```

值是在 INSERT 和 UPDATE 时持久化，还是在获取时计算，是数据库的实现细节；前者称为“存储”，后者称为“虚拟”。 一些数据库实现支持两者，但有些只支持其中一个。 可以指定可选的 `Computed.persisted` 标志为 `True` 或 `False`，以指示是否在 DDL 中渲染“STORED”或“VIRTUAL”关键字，但是如果目标后端不支持该关键字，则会引发错误； 如果将其设置为未设置，则将使用目标后端的有效默认值。

`计算列` 构造是 `FetchedValue` 对象的子类，并且会自行设置为目标 `列` 的“服务器默认值”和“服务器更新时生成器”，这意味着当生成 INSERT 和 UPDATE 语句时，它将被视为默认生成的列，以及当使用 ORM 时，它将被视为生成的列被获取。这包括它将作为数据库的 RETURNING 子句的一部分，对于支持 RETURNING 并且生成的值需要被急切地获取的数据库。

注

使用 `计算列` 构造定义的 `列` 可能不会存储除服务器应用之外的任何值；当尝试写入 INSERT 或 UPDATE 时，SQLAlchemy 目前的行为是将忽略该值。

“GENERATED ALWAYS AS” 目前已知受支持的数据库有：

+   MySQL 版本 5.7 及以上

+   MariaDB 10.x 系列及以上

+   PostgreSQL 版本 12 及以上

+   Oracle - 注意 RETURNING 在 UPDATE 中无法正常工作（在包含计算列的 UPDATE..RETURNING 被呈现时会发出警告）

+   Microsoft SQL Server

+   SQLite 版本 3.31 及以上

当 `计算列` 与不受支持的后端一起使用时，如果目标方言不支持它，则在尝试呈现构造时会引发 `CompileError`。否则，如果方言支持它但使用的特定数据库服务器版本不支持它，则在将 DDL 发送到数据库时会引发 `DBAPIError` 的子类，通常是 `OperationalError`。

请参见

`计算列`

## 自增列（GENERATED { ALWAYS | BY DEFAULT } AS IDENTITY）

1.4 版本中新增。

`Identity` 构造允许将 `列` 声明为自增列，并在 DDL 中呈现为“GENERATED { ALWAYS | BY DEFAULT } AS IDENTITY”。自增列的值由数据库服务器自动生成，使用增量（或减量）序列。该构造与 `Sequence` 共享大部分用于控制数据库行为的选项。

示例：

```py
from sqlalchemy import Table, Column, MetaData, Integer, Identity, String

metadata_obj = MetaData()

data = Table(
    "data",
    metadata_obj,
    Column("id", Integer, Identity(start=42, cycle=True), primary_key=True),
    Column("data", String),
)
```

在 PostgreSQL 12 后端上运行时，`data` 表的 DDL 如下所示：

```py
CREATE  TABLE  data  (
  id  INTEGER  GENERATED  BY  DEFAULT  AS  IDENTITY  (START  WITH  42  CYCLE)  NOT  NULL,
  data  VARCHAR,
  PRIMARY  KEY  (id)
)
```

数据库将在插入时为 `id` 列生成一个值，从 `42` 开始，如果语句尚未包含 `id` 列的值。身份列也可以要求数据库生成列的值，忽略语句中传递的值或者根据后端引发错误。要激活此模式，请在 `Identity` 构造函数中将参数 `Identity.always` 设置为 `True`。将上一个示例更新以包含此参数将生成以下 DDL：

```py
CREATE  TABLE  data  (
  id  INTEGER  GENERATED  ALWAYS  AS  IDENTITY  (START  WITH  42  CYCLE)  NOT  NULL,
  data  VARCHAR,
  PRIMARY  KEY  (id)
)
```

`Identity` 构造函数是 `FetchedValue` 对象的子类，并将自己设置为目标 `Column` 的“服务器默认”生成器，这意味着当生成 INSERT 语句时，它将被视为默认生成列，以及在使用 ORM 时，它将被视为生成列。这包括它将作为数据库的 RETURNING 子句的一部分，对于支持 RETURNING 并且要急切获取生成的值的数据库来说，它将被提前获取。

当前已知支持 `Identity` 构造函数的包括：

+   PostgreSQL 版本为 10。

+   Oracle 版本为 12\. 还支持传递 `always=None` 以启用默认生成模式，以及传递参数 `on_null=True` 以指定“ON NULL”与“BY DEFAULT”身份列一起使用。

+   Microsoft SQL Server. MSSQL 使用一种自定义语法，仅支持 `start` 和 `increment` 参数，而忽略所有其他参数。

当 `Identity` 与不受支持的后端一起使用时，它会被忽略，并且会使用默认的 SQLAlchemy 自增列逻辑。

当 `Column` 同时指定 `Identity` 并将 `Column.autoincrement` 设置为 `False` 时，将引发错误。

另请参阅

`Identity`

## 默认对象 API

| 对象名称 | 描述 |
| --- | --- |
| ColumnDefault | 列的普通默认值。 |
| Computed | 定义了一个生成的列，即“GENERATED ALWAYS AS”语法。 |
| DefaultClause | 由 DDL 指定的 DEFAULT 列值。 |
| DefaultGenerator | 用于列默认值的基类。 |
| FetchedValue | 透明数据库端默认值的标记。 |
| Identity | 定义标识列，即“GENERATED { ALWAYS &#124; BY DEFAULT } AS IDENTITY”语法。 |
| Sequence | 表示命名的数据库序列。 |

```py
class sqlalchemy.schema.Computed
```

定义一个生成列，即“GENERATED ALWAYS AS”语法。

`Computed`构造是添加到`Column`对象的参数列表中的内联构造：

```py
from sqlalchemy import Computed

Table('square', metadata_obj,
    Column('side', Float, nullable=False),
    Column('area', Float, Computed('side * side'))
)
```

请参阅下面链接的文档以获取完整详细信息。

版本 1.3.11 中的新功能。

另请参阅

计算列（GENERATED ALWAYS AS）

**成员**

__init__(), copy()

**类签名**

类`sqlalchemy.schema.Computed`（`sqlalchemy.schema.FetchedValue`，`sqlalchemy.schema.SchemaItem`)

```py
method __init__(sqltext: _DDLColumnArgument, persisted: bool | None = None) → None
```

构造一个生成的 DDL 构造，以配合`Column`。

参数：

+   `sqltext` –

    包含列生成表达式的字符串，该表达式将逐字使用，或者 SQL 表达式构造，例如`text()`对象。 如果以字符串形式给出，则将对象转换为`text()`对象。

    警告

    `Computed.sqltext`参数可以作为 Python 字符串参数传递给`Computed`，它将被视为**受信任的 SQL 文本**并按照给定的方式呈现。 **不要将不受信任的输入传递给此参数**。

+   `persisted` –

    可选，控制数据库如何持久化此列。 可能的值为：

    +   `None`，默认值，将使用数据库定义的默认持久性。

    +   `True`，将呈现`GENERATED ALWAYS AS ... STORED`，或者如果支持的话，将呈现目标数据库的等效值。

    +   `False`，将呈现`GENERATED ALWAYS AS ... VIRTUAL`，或者如果支持的话，将呈现目标数据库的等效值。

    当 DDL 发出到目标数据库时，如果数据库不支持持久性选项，则指定`True`或`False`可能会引发错误。 将此参数保留在其默认值`None`上可确保对所有支持`GENERATED ALWAYS AS`的数据库都能成功。

```py
method copy(*, target_table: Table | None = None, **kw: Any) → Computed
```

自版本 1.4 起已弃用：`Computed.copy()`方法已弃用，并将在将来的版本中删除。

```py
class sqlalchemy.schema.ColumnDefault
```

列上的普通默认值。

这可能对应于一个常量，一个可调用函数，或者一个 SQL 子句。

每当使用 `Column` 的 `default`、`onupdate` 参数时，都会自动生成 `ColumnDefault`。`ColumnDefault` 也可以按位置传递。

例如，以下内容：

```py
Column('foo', Integer, default=50)
```

等同于：

```py
Column('foo', Integer, ColumnDefault(50))
```

**类签名**

class `sqlalchemy.schema.ColumnDefault` (`sqlalchemy.schema.DefaultGenerator`, `abc.ABC`)

```py
class sqlalchemy.schema.DefaultClause
```

由 DDL 指定的 DEFAULT 列值。

`DefaultClause` 是一个 `FetchedValue`，在发出“CREATE TABLE”时也会生成一个“DEFAULT”子句。

每当使用 `Column` 的 `server_default`、`server_onupdate` 参数时，都会自动生成 `DefaultClause`。`DefaultClause` 也可以按位置传递。

例如，以下内容：

```py
Column('foo', Integer, server_default="50")
```

等同于：

```py
Column('foo', Integer, DefaultClause("50"))
```

**类签名**

class `sqlalchemy.schema.DefaultClause` (`sqlalchemy.schema.FetchedValue`)

```py
class sqlalchemy.schema.DefaultGenerator
```

列默认值的基类。

此对象仅存在于 column.default 或 column.onupdate。它不作为服务器默认值有效。

**类签名**

class `sqlalchemy.schema.DefaultGenerator` (`sqlalchemy.sql.expression.Executable`, `sqlalchemy.schema.SchemaItem`)

```py
class sqlalchemy.schema.FetchedValue
```

一个用于透明数据库端默认值的标记。

当数据库配置为为列提供一些自动默认值时，请使用 `FetchedValue`。

例如：

```py
Column('foo', Integer, FetchedValue())
```

将指示某个触发器或默认生成器在插入期间为 `foo` 列创建一个新值。

另请参阅

标记隐式生成的值、时间戳和触发列

**类签名**

class `sqlalchemy.schema.FetchedValue` (`sqlalchemy.sql.expression.SchemaEventTarget`)

```py
class sqlalchemy.schema.Sequence
```

表示一个命名的数据库序列。

`Sequence` 对象表示数据库序列的名称和配置参数。它还表示可以由 SQLAlchemy `Engine` 或 `Connection` “执行”的构造，为目标数据库渲染适当的 “下一个值” 函数并返回结果。

`Sequence` 通常与主键列相关联：

```py
some_table = Table(
    'some_table', metadata,
    Column('id', Integer, Sequence('some_table_seq', start=1),
    primary_key=True)
)
```

当为上述 `Table` 发出 CREATE TABLE 时，如果目标平台支持序列，则还将发出 CREATE SEQUENCE 语句。对于不支持序列的平台，将忽略 `Sequence` 构造。

另请参阅

定义序列

`CreateSequence`

`DropSequence`

**成员**

__init__(), create(), drop(), next_value()

**类签名**

类 `sqlalchemy.schema.Sequence` (`sqlalchemy.schema.HasSchemaAttr`, `sqlalchemy.schema.IdentityOptions`, `sqlalchemy.schema.DefaultGenerator`)

```py
method __init__(name: str, start: int | None = None, increment: int | None = None, minvalue: int | None = None, maxvalue: int | None = None, nominvalue: bool | None = None, nomaxvalue: bool | None = None, cycle: bool | None = None, schema: str | Literal[SchemaConst.BLANK_SCHEMA] | None = None, cache: int | None = None, order: bool | None = None, data_type: _TypeEngineArgument[int] | None = None, optional: bool = False, quote: bool | None = None, metadata: MetaData | None = None, quote_schema: bool | None = None, for_update: bool = False) → None
```

构造一个 `Sequence` 对象。

参数：

+   `name` – 序列的名称。

+   `start` –

    序列的起始索引。当向数据库发出 CREATE SEQUENCE 命令时，此值用作 “START WITH” 子句的值。如果为 `None`，则省略子句，大多数平台上表示起始值为 1。

    从版本 2.0 开始：要使 DDL 发出 “START WITH” 命令，`Sequence.start` 参数是必需的。这是对版本 1.4 中所做更改的逆转，该更改如果未包括 `Sequence.start` 将隐式地渲染 “START WITH 1”。有关更多详细信息，请参见序列构造将恢复为没有任何显式默认的“start”值；影响 MS SQL Server。

+   `increment` – 序列的增量值。当向数据库发出 CREATE SEQUENCE 命令时，此值用作 “INCREMENT BY” 子句的值。如果为 `None`，则省略子句，大多数平台上表示增量为 1。

+   `minvalue` – 序列的最小值。当将 CREATE SEQUENCE 命令发送到数据库时，此值用作“MINVALUE”子句的值。如果为`None`，则省略该子句，在大多数平台上表示升序和降序序列的最小值分别为 1 和-2⁶³-1。

+   `maxvalue` – 序列的最大值。当将 CREATE SEQUENCE 命令发送到数据库时，此值用作“MAXVALUE”子句的值。如果为`None`，则省略该子句，在大多数平台上表示升序和降序序列的最大值分别为 2⁶³-1 和-1。

+   `nominvalue` – 序列的无最小值。当将 CREATE SEQUENCE 命令发送到数据库时，此值用作“NO MINVALUE”子句的值。如果为`None`，则省略该子句，在大多数平台上表示升序和降序序列的最小值分别为 1 和-2⁶³-1。

+   `nomaxvalue` – 序列的无最大值。当将 CREATE SEQUENCE 命令发送到数据库时，此值用作“NO MAXVALUE”子句的值。如果为`None`，则省略该子句，在大多数平台上表示升序和降序序列的最大值分别为 2⁶³-1 和-1。

+   `cycle` – 允许序列在达到最大值或最小值时循环。当升序或降序序列分别达到最大值或最小值时，此值用于在将 CREATE SEQUENCE 命令发送到数据库时作为“CYCLE”子句。如果达到限制，则生成的下一个数字将分别是最小值或最大值。如果 cycle=False（默认值），则在序列达到其最大值后调用 nextval 将返回错误。

+   `schema` – 序列的可选模式名称，如果位于除默认模式之外的模式中。当`MetaData`也存在时，选择模式名称的规则与`Table.schema`的规则相同。

+   `cache` – 可选的整数值；提前计算序列中未来值的数量。渲染 Oracle 和 PostgreSQL 理解的 CACHE 关键字。

+   `order` – 可选的布尔值；如果为`True`，则渲染 ORDER 关键字，Oracle 可理解此关键字，表示序列是有确定顺序的。可能需要使用 Oracle RAC 提供确定性排序。

+   `data_type` –

    序列要返回的类型，对于允许我们在 INTEGER、BIGINT 等之间选择的方言（例如 mssql）。

    自 1.4.0 版本新增。

+   `optional` – 布尔值，当为`True`时，表示这个`Sequence`对象只需在不提供其他方法生成主键标识符的后端上显式生成。目前，它基本上意味着“在 PostgreSQL 后端上不要创建这个序列，在那里，SERIAL 关键字会自动为我们创建一个序列”。

+   `quote` – 布尔值，当为`True`或`False`时，显式地强制对 `Sequence.name` 进行引用或取消引用。当保持其默认值`None`时，将根据大小写和保留字规则进行正常的引用。

+   `quote_schema` – 设置对`schema`名称的引用偏好。

+   `metadata` –

    可选的 `MetaData` 对象，这个`Sequence`将与之关联。与 `MetaData` 关联的 `Sequence` 将获得以下功能：

    +   `Sequence`将继承指定给目标`MetaData`的`MetaData.schema`参数，这会影响创建/删除 DDL 的生成，如果有的话。

    +   `Sequence.create()` 和 `Sequence.drop()` 方法会自动使用绑定到 `MetaData` 对象的引擎（如果有的话）。

    +   `MetaData.create_all()` 和 `MetaData.drop_all()` 方法将为这个`Sequence`发出 CREATE / DROP，即使这个`Sequence`没有与任何属于这个`MetaData`的`Table` / `Column`相关联也是如此。

    上述行为只有在通过此参数将 `Sequence` 显式关联到 `MetaData` 时才会发生。

    另请参见

    将 Sequence 与 MetaData 关联 - 对`Sequence.metadata`参数的完整讨论。

+   `for_update` – 当与`Column`相关联时，表示应该在该列的表上对 UPDATE 语句调用此`Sequence`，而不是在 INSERT 语句中，当该列在语句中没有其他值时。

```py
method create(bind: _CreateDropBind, checkfirst: bool = True) → None
```

在数据库中创建此序列。

```py
method drop(bind: _CreateDropBind, checkfirst: bool = True) → None
```

从数据库中删除此序列。

```py
method next_value() → Function[int]
```

返回一个`next_value`函数元素，该函数将在任何 SQL 表达式中为此`Sequence`呈现适当的增量函数。

```py
class sqlalchemy.schema.Identity
```

定义一个 identity 列，即“GENERATED { ALWAYS | BY DEFAULT } AS IDENTITY”语法。

`Identity` 构造是一个内联构造，添加到`Column`对象的参数列表中：

```py
from sqlalchemy import Identity

Table('foo', metadata_obj,
    Column('id', Integer, Identity())
    Column('description', Text),
)
```

详细信息请参见下面的链接文档。

版本 1.4 中新增。

另请参阅

Identity 列（GENERATED { ALWAYS | BY DEFAULT } AS IDENTITY）

**成员**

__init__(), copy()

**类签名**

class `sqlalchemy.schema.Identity` (`sqlalchemy.schema.IdentityOptions`, `sqlalchemy.schema.FetchedValue`, `sqlalchemy.schema.SchemaItem`)

```py
method __init__(always: bool = False, on_null: bool | None = None, start: int | None = None, increment: int | None = None, minvalue: int | None = None, maxvalue: int | None = None, nominvalue: bool | None = None, nomaxvalue: bool | None = None, cycle: bool | None = None, cache: int | None = None, order: bool | None = None) → None
```

构造一个 GENERATED { ALWAYS | BY DEFAULT } AS IDENTITY 的 DDL 构造，以配合一个`Column`。

有关大多数参数的完整描述，请参阅`Sequence`文档。

注意

MSSQL 支持此构造作为在列上生成 IDENTITY 的首选替代方法，但它使用的非标准语法仅支持`Identity.start`和`Identity.increment`。所有其他参数都将被忽略。

参数：

+   `always` – 一个布尔值，表示身份列的类型。如果指定为`False`，则用户指定的值优先。如果指定为`True`，则不接受用户指定的值（在某些后端，如 PostgreSQL，可以在 INSERT 中指定 OVERRIDING SYSTEM VALUE 或类似的内容来覆盖序列值）。一些后端也对此参数有一个默认值，`None` 可以用来省略在 DDL 中呈现这部分。如果后端没有默认值，则将其视为`False`。

+   `on_null` – 设置为`True` 以指定在`always=False`身份列中与`ON NULL`一起使用。此选项仅在某些后端（如 Oracle）上受支持。

+   `start` – 序列的起始索引。

+   `increment` – 序列的增量值。

+   `minvalue` – 序列的最小值。

+   `maxvalue` – 序列的最大值。

+   `nominvalue` – 序列没有最小值。

+   `nomaxvalue` – 序列没有最大值。

+   `cycle` – 允许序列在达到`maxvalue`或`minvalue`时循环。

+   `cache` – 可选整数值；提前计算的序列中未来值的数量。

+   `order` – 可选布尔值；如果为真，则呈现 ORDER 关键字。

```py
method copy(**kw: Any) → Identity
```

自版本 1.4 弃用：`Identity.copy()` 方法已弃用，并将在将来的版本中移除。
