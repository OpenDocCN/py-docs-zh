# 引擎配置

> 原文：[`docs.sqlalchemy.org/en/20/core/engines.html`](https://docs.sqlalchemy.org/en/20/core/engines.html)

`Engine` 是任何 SQLAlchemy 应用的起点。它是实际数据库和通过连接池和 `Dialect` 传递给 SQLAlchemy 应用程序的 DBAPI 的“主基地”，`Dialect` 描述了如何与特定类型的数据库/DBAPI 组合进行通信。

通用结构可如下所示：

![../_images/sqla_engine_arch.png](img/f551a4dda3e52a3c0fd8a1de0517174e.png)

在上述中，`Engine` 同时引用了 `Dialect` 和 `Pool`，它们一起解释了 DBAPI 的模块函数以及数据库的行为。

创建引擎只需发出一个单独的调用，`create_engine()`：

```py
from sqlalchemy import create_engine

engine = create_engine("postgresql+psycopg2://scott:tiger@localhost:5432/mydatabase")
```

上述引擎创建了一个针对 PostgreSQL 定制的 `Dialect` 对象，以及一个 `Pool` 对象，当首次收到连接请求时，将在 `localhost:5432` 建立 DBAPI 连接。请注意，`Engine` 及其底层的 `Pool` 在首次调用 `Engine.connect()` 或 `Engine.begin()` 方法之前都不会建立第一个实际的 DBAPI 连接。当其他 SQLAlchemy `Engine` 依赖对象（例如首次需要数据库连接时的 ORM `Session` 对象）首次需要数据库连接时，也可以调用这些方法中的任何一个。通过这种方式，`Engine` 和 `Pool` 可以说具有*延迟初始化*行为。

一旦创建了 `Engine`，它可以直接用于与数据库交互，也可以传递给一个 `Session` 对象以与 ORM 一起使用。本节介绍了配置 `Engine` 的详细信息。下一节，使用 Engines 和 Connections，将详细介绍 `Engine` 和类似对象的使用 API，通常用于非 ORM 应用程序。

## 支持的数据库

SQLAlchemy 包含许多不同后端的 `Dialect` 实现。SQLAlchemy 包含最常见数据库的方言；另外一些需要额外安装单独的方言。

参见 方言 部分了解各种可用后端的信息。## 数据库 URL

`create_engine()` 函数基于 URL 生成一个 `Engine` 对象。URL 的格式通常遵循 [RFC-1738](https://www.ietf.org/rfc/rfc1738.txt) 的规范，但也有一些例外，包括“scheme”部分中接受下划线而不是破折号或句点。URL 通常包括用户名、密码、主机名、数据库名称字段，以及用于额外配置的可选关键字参数。在某些情况下，可以接受文件路径，而在其他情况下，“数据源名称”取代了“主机”和“数据库”部分。数据库 URL 的典型形式是：

```py
dialect+driver://username:password@host:port/database
```

方言名称包括 SQLAlchemy 方言的标识名称，例如 `sqlite`、`mysql`、`postgresql`、`oracle` 或 `mssql`。驱动名称是要使用的 DBAPI 的名称，全部使用小写字母连接到数据库。如果未指定，将导入“默认”DBAPI（如果可用）- 该默认值通常是该后端可用的最广为人知的驱动程序。

### 转义特殊字符，例如密码中的 @ 符号

构建一个完整的 URL 字符串以传递给 `create_engine()` 时，**特殊字符（如用户和密码中可能使用的字符）需要进行 URL 编码才能正确解析。**。**这包括 @ 符号**。

下面是一个包含密码 `"kx@jj5/g"` 的 URL 示例，其中“at”符号和斜杠字符分别表示为 `%40` 和 `%2F`：

```py
postgresql+pg8000://dbuser:kx%40jj5%2Fg@pghost10/appdb
```

上述密码的编码可以使用 [urllib.parse](https://docs.python.org/3/library/urllib.parse.html) 生成：

```py
>>> import urllib.parse
>>> urllib.parse.quote_plus("kx@jj5/g")
'kx%40jj5%2Fg'
```

然后可以将 URL 作为字符串传递给 `create_engine()`：

```py
from sqlalchemy import create_engine

engine = create_engine("postgresql+pg8000://dbuser:kx%40jj5%2Fg@pghost10/appdb")
```

作为在创建完整的 URL 字符串时转义特殊字符的替代方法，可以将传递给 `create_engine()` 的对象替换为 `URL` 对象，它可以直接绕过解析阶段并直接处理未转义的字符串。请参阅下一节的示例。

在 1.4 版本中更改：修复了主机名和数据库名中 `@` 符号的支持。由于此修复的副作用，密码中的 `@` 符号必须进行转义。

### 程序化创建 URL

传递给 `create_engine()` 的值可以是 `URL` 的一个实例，而不是一个普通字符串，这样可以绕过需要使用字符串解析的步骤，因此不需要提供转义的 URL 字符串。

使用 `URL.create()` 构造方法创建 `URL` 对象，通过逐个传递所有字段。密码中的特殊字符可以不做任何修改地传递：

```py
from sqlalchemy import URL

url_object = URL.create(
    "postgresql+pg8000",
    username="dbuser",
    password="kx@jj5/g",  # plain (unescaped) text
    host="pghost10",
    database="appdb",
)
```

构造的 `URL` 对象然后可以直接传递给 `create_engine()`，而不是一个字符串参数：

```py
from sqlalchemy import create_engine

engine = create_engine(url_object)
```

另见

`URL`

`URL.create()`

### 后端特定的 URLs

下面是常见连接样式的示例。要获取所有包含的方言的详细信息以及第三方方言的链接，请参阅 Dialects。

#### PostgreSQL

PostgreSQL 方言使用 psycopg2 作为默认的 DBAPI。其他 PostgreSQL DBAPI 包括 pg8000 和 asyncpg：

```py
# default
engine = create_engine("postgresql://scott:tiger@localhost/mydatabase")

# psycopg2
engine = create_engine("postgresql+psycopg2://scott:tiger@localhost/mydatabase")

# pg8000
engine = create_engine("postgresql+pg8000://scott:tiger@localhost/mydatabase")
```

连接到 PostgreSQL 的更多注意事项请参考 PostgreSQL。

#### MySQL

MySQL 方言使用 mysqlclient 作为默认的 DBAPI。还有其他可用的 MySQL DBAPI，包括 PyMySQL：

```py
# default
engine = create_engine("mysql://scott:tiger@localhost/foo")

# mysqlclient (a maintained fork of MySQL-Python)
engine = create_engine("mysql+mysqldb://scott:tiger@localhost/foo")

# PyMySQL
engine = create_engine("mysql+pymysql://scott:tiger@localhost/foo")
```

连接到 MySQL 的更多注意事项请参考 MySQL 和 MariaDB。

#### Oracle

Oracle 方言使用 cx_oracle 作为默认的 DBAPI：

```py
engine = create_engine("oracle://scott:tiger@127.0.0.1:1521/sidname")

engine = create_engine("oracle+cx_oracle://scott:tiger@tnsname")
```

连接到 Oracle 的更多注意事项请参考 Oracle。

#### Microsoft SQL Server

SQL Server 方言使用 pyodbc 作为默认的 DBAPI。也可以使用 pymssql：

```py
# pyodbc
engine = create_engine("mssql+pyodbc://scott:tiger@mydsn")

# pymssql
engine = create_engine("mssql+pymssql://scott:tiger@hostname:port/dbname")
```

连接到 SQL Server 的更多注意事项请参考 Microsoft SQL Server。

#### SQLite

SQLite 连接到基于文件的数据库，默认情况下使用 Python 内置模块 `sqlite3`。

由于 SQLite 连接到本地文件，URL 格式略有不同。URL 的 “file” 部分是数据库的文件名。对于相对文件路径，这需要三个斜杠：

```py
# sqlite://<nohostname>/<path>
# where <path> is relative:
engine = create_engine("sqlite:///foo.db")
```

对于绝对文件路径，三个斜杠后面跟着绝对路径：

```py
# Unix/Mac - 4 initial slashes in total
engine = create_engine("sqlite:////absolute/path/to/foo.db")

# Windows
engine = create_engine("sqlite:///C:\\path\\to\\foo.db")

# Windows alternative using raw string
engine = create_engine(r"sqlite:///C:\path\to\foo.db")
```

要使用 SQLite 的 `:memory:` 数据库，请指定一个空的 URL：

```py
engine = create_engine("sqlite://")
```

关于连接到 SQLite 的更多注释请参见 SQLite。

#### 其他

请查看方言，其中包含所有额外方言文档的顶级页面。## Engine Creation API

| 对象名称 | 描述 |
| --- | --- |
| create_engine(url, **kwargs) | 创建一个新的 `Engine` 实例。 |
| create_mock_engine(url, executor, **kw) | 创建一个用于回显 DDL 的“模拟”引擎。 |
| create_pool_from_url(url, **kwargs) | 从给定的 URL 创建一个池实例。 |
| engine_from_config(configuration[, prefix], **kwargs) | 使用配置字典创建一个新的 Engine 实例。 |
| make_url(name_or_url) | 给定一个字符串，生成一个新的 URL 实例。 |
| URL | 表示用于连接到数据库的 URL 的组件。 |

```py
function sqlalchemy.create_engine(url: str | _url.URL, **kwargs: Any) → Engine
```

创建一个新的 `Engine` 实例。

标准调用形式是将 URL 作为第一个位置参数发送，通常是一个指示数据库方言和连接参数的字符串：

```py
engine = create_engine("postgresql+psycopg2://scott:tiger@localhost/test")
```

注

请查阅数据库 URL 以获取编写 URL 字符串的一般指南。特别是，诸如密码中经常出现的特殊字符必须进行 URL 编码才能正确解析。

然后可以跟随它的附加关键字参数，这些参数在结果 `Engine` 及其底层 `Dialect` 和 `Pool` 构造上建立各种选项：

```py
engine = create_engine("mysql+mysqldb://scott:tiger@hostname/dbname",
                            pool_recycle=3600, echo=True)
```

URL 的字符串形式是 `dialect[+driver]://user:password@host/dbname[?key=value..]`，其中 `dialect` 是数据库名称，例如 `mysql`、`oracle`、`postgresql` 等，而 `driver` 是 DBAPI 的名称，例如 `psycopg2`、`pyodbc`、`cx_oracle` 等。或者，URL 可以是 `URL` 的实例。

`**kwargs` 接受各种选项，这些选项将被路由到其相应的组件。参数可能是特定于 `Engine`、底层 `Dialect`，以及 `Pool` 的。特定的方言也接受特定于该方言的关键字参数。在这里，我们描述了大多数 `create_engine()` 使用的参数。

一旦建立，新生成的`Engine`在调用`Engine.connect()`或依赖于它的方法（如`Engine.execute()`）时，将从底层的`Pool`请求连接。当收到此请求时，`Pool`将建立第一个实际的 DBAPI 连接。`create_engine()`调用本身**不会**直接建立任何实际的 DBAPI 连接。

另请参阅

引擎配置

方言

使用引擎和连接

参数：

+   `connect_args` – 一个字典选项，将作为额外的关键字参数直接传递给 DBAPI 的`connect()`方法。请参见自定义 DBAPI connect()参数 / on-connect 例程中的示例。

+   `creator` –

    一个可调用的函数，返回一个 DBAPI 连接。此创建函数将传递给底层连接池，并将用于创建所有新的数据库连接。使用此函数会绕过 URL 参数中指定的连接参数。

    这个钩子不像较新的`DialectEvents.do_connect()`钩子那样灵活，后者允许完全控制如何连接到数据库，考虑到之前的 URL 参数和状态。

    另请参阅

    `DialectEvents.do_connect()` - 允许完全控制 DBAPI 连接机制的事件钩子。

    自定义 DBAPI connect()参数 / on-connect 例程

+   `echo=False` –

    如果为 True，则 Engine 将记录所有语句以及它们参数列表的`repr()`到默认的日志处理程序，该处理程序默认为`sys.stdout`。如果设置为字符串`"debug"`，结果行也将打印到标准输出。`Engine`的`echo`属性可以随时修改以打开和关闭日志记录；还可以使用标准的 Python `logging`模块直接控制日志记录。

    另请参阅

    配置日志记录 - 关于如何配置日志记录的更多详细信息。

+   `echo_pool=False` –

    如果为 True，则连接池将记录信息性输出，例如当连接无效时以及当连接被回收时，将输出到默认的日志处理程序，该处理程序默认为`sys.stdout`。如果设置为字符串`"debug"`，日志记录将包括池的检出和检入。还可以使用标准的 Python `logging`模块直接控制日志记录。

    另请参阅

    配置日志记录 - 关于如何配置日志记录的更多详细信息。

+   `empty_in_strategy` –

    不再使用；SQLAlchemy 现在在所有情况下都使用“空集合”行为来处理 IN。

    自版本 1.4 弃用：`create_engine.empty_in_strategy` 关键字已弃用，不再起作用。所有 IN 表达式现在都使用“扩展参数”策略进行渲染，该策略在语句执行时会渲染一组绑定的表达式，或者一个“空集合”SELECT。

+   `enable_from_linting` –

    默认为 True。如果发现给定的 SELECT 语句具有未链接的 FROM 元素，会发出警告，这些元素会导致笛卡尔积。

    新版本 1.4 中新增。

    另请参阅

    内置的 FROM linting 将为 SELECT 语句中的任何潜在笛卡尔积发出警告

+   `execution_options` – 将应用于所有连接的字典执行选项。参见 `Connection.execution_options()`

+   `future` –

    使用 2.0 风格的 `Engine` 和 `Connection` API。

    从 SQLAlchemy 2.0 开始，此参数仅用于向后兼容，必须保持默认值为 `True`。

    `create_engine.future` 参数将在随后的 2.x 版本中弃用，并最终被移除。

    新版本 1.4 中新增。

    从版本 2.0 开始更改：所有 `Engine` 对象都是“未来”风格的引擎，不再有 `future=False` 的操作模式。

    另请参阅

    SQLAlchemy 2.0 - 主要迁移指南

+   `hide_parameters` –

    布尔值，当设置为 True 时，SQL 语句参数将不会显示在 INFO 日志中，也不会被格式化为 `StatementError` 对象的字符串表示形式。

    新版本 1.3.8 中新增。

    另请参阅

    配置日志记录 - 关于如何配置日志记录的更多详细信息。

+   `implicit_returning=True` – 只能设置为 True 的旧参数。在 SQLAlchemy 2.0 中，此参数无效。为了禁用 ORM 调用的语句的“隐式返回”，请在每个表上使用 `Table.implicit_returning` 参数进行配置。

+   `insertmanyvalues_page_size` –

    当语句使用“insertmanyvalues”模式时，格式化为 INSERT 语句的行数。这是一种分页形式的批量插入，通常与 executemany 执行一起使用，与 RETURNING 一起使用。默认为 1000，但也可能受方言特定的限制因素的影响，这些因素可能会覆盖此值的每个语句的基础。

    2.0 版本中的新内容。

    另请参见

    关于 INSERT 语句的“插入多个值”行为

    控制批量大小

    `Connection.execution_options.insertmanyvalues_page_size`

+   `isolation_level` –

    无条件地在所有新连接上设置的隔离级别的可选字符串名称。隔离级别通常是字符串名称的一些子集`"SERIALIZABLE"`、`"REPEATABLE READ"`、`"READ COMMITTED"`、`"READ UNCOMMITTED"`和`"AUTOCOMMIT"`根据后端。

    `create_engine.isolation_level`参数与`Connection.execution_options.isolation_level`执行选项形成对比，后者可以在单个`Connection`上设置，以及传递给`Engine.execution_options()`的相同参数，它可以用于创建具有不同隔离级别的多个引擎，这些引擎共享一个公共的连接池和方言。

    从版本 2.0 开始更改：`create_engine.isolation_level`参数已被概括为适用于所有支持隔离级别概念的方言，并提供为更简洁、更前置的配置开关，与执行选项形成对比，后者更多是一种临时的编程选项。

    另请参见

    设置事务隔离级别，包括 DBAPI 自动提交

+   `json_deserializer` –

    对于支持`JSON`数据类型的方言，这是一个将 JSON 字符串转换为 Python 对象的 Python 可调用函数。默认情况下，使用 Python 的`json.loads`函数。

    从版本 1.3.7 开始更改：SQLite 方言将其从`_json_deserializer`改名为。

+   `json_serializer` –

    对支持`JSON`数据类型的方言来说，这是一个 Python 可调用函数，用于将给定对象呈现为 JSON。默认情况下，使用 Python 的`json.dumps`函数。

    在 1.3.7 版本中更改：SQLite 方言将其从`_json_serializer`重命名为其他内容。

+   `label_length=None` –

    可选整数值，限制动态生成的列标签的大小到该字符数。如果小于 6，则标签生成为“_(counter)”。如果为`None`，则使用`create_engine.max_identifier_length`参数受影响的`dialect.max_identifier_length`的值。`create_engine.label_length`的值不能大于`create_engine.max_identfier_length`的值。

    另请参见

    `create_engine.max_identifier_length`

+   `logging_name` –

    用于在“sqlalchemy.engine”记录器中生成的日志记录的“name”字段内使用的字符串标识符。默认为对象的 id 的十六进制字符串。

    另请参见

    配置日志 - 关于如何配置日志的进一步详细信息。

    `Connection.execution_options.logging_token`

+   `max_identifier_length` –

    整数；覆盖方言确定的`max_identifier_length`。如果为`None`或零，则无效果。这是数据库配置的最大字符数，可以在 SQL 标识符（如表名、列名或标签名）中使用。所有方言都会自动确定此值，但是如果数据库版本更改了此值但是 SQLAlchemy 的方言没有进行调整，可以在此处传递此值。

    1.3.9 版本中的新内容。

    另请参见

    `create_engine.label_length`

+   `max_overflow=10` – 允许在连接池“溢出”中打开的连接数，即在池大小设置之上可以打开的连接数，默认为五。仅与`QueuePool`一起使用。

+   `module=None` – Python 模块对象的引用（模块本身，而不是其字符串名称）。指定引擎方言要使用的替代 DBAPI 模块。每个子方言引用一个特定的 DBAPI，将在首次连接之前导入。此参数导致导入被绕过，并且使用给定模块代替。可以用于测试 DBAPI 以及将“模拟”DBAPI 实现注入到`Engine`中。

+   `paramstyle=None` – 渲染绑定参数时要使用的[paramstyle](https://legacy.python.org/dev/peps/pep-0249/#paramstyle)。此样式默认为 DBAPI 本身推荐的样式，从 DBAPI 的`.paramstyle`属性中检索。然而，大多数 DBAPI 接受多种 paramstyle，特别是可能希望将“命名”paramstyle 转换为“位置”paramstyle，反之亦然。当传递此属性时，它应该是值之一`"qmark"`、`"numeric"`、`"named"`、`"format"`或`"pyformat"`，并且应该对应于 DBAPI 已知支持的参数样式。

+   `pool=None` – 已构造的`Pool`实例，例如`QueuePool`实例。如果非 None，则此池将直接用作引擎的底层连接池，绕过 URL 参数中存在的任何连接参数。有关手动构建连接池的信息，请参见连接池。

+   `poolclass=None` – `Pool`子类，将使用 URL 中给定的连接参数创建连接池实例。请注意，这与`pool`不同，因为在这种情况下您实际上不会实例化池，只是指示要使用哪种类型的池。

+   `pool_logging_name` –

    将在“sqlalchemy.pool”记录生成的“name”字段中使用的字符串标识符。默认为对象的 id 的十六进制字符串。

    另请参见

    配置日志记录 - 关于如何配置日志记录的更多细节。

+   `pool_pre_ping` –

    布尔值，如果为 True，则启用连接池的“预检”功能，即在每次检出时测试连接的活动性。

    1.2 版中的新功能。

    另请参见

    断开处理 - 悲观

+   `pool_size=5` – 在连接池中保持打开的连接数。这与`QueuePool`以及`SingletonThreadPool`一起使用。对于`QueuePool`，设置`pool_size`为 0 表示没有限制；要禁用池化，请将`poolclass`设置为`NullPool`。

+   `pool_recycle=-1` –

    此设置在给定的秒数过去后导致池回收连接。默认为 -1，或无超时。例如，将其设置为 3600 表示一小时后将回收连接。请注意，特别是 MySQL 将在连接上没有检测到活动时自动断开连接八小时（虽然这可以通过 MySQLDB 连接本身和服务器配置进行配置）。

    另请参阅

    设置池回收

+   `pool_reset_on_return='rollback'` –

    设置底层`Pool`对象的`Pool.reset_on_return`参数，可以设置为值`"rollback"`、`"commit"`或`None`。

    另请参阅

    归还时重置

+   `pool_timeout=30` –

    从池中获取连接之前等待的秒数。仅与`QueuePool`一起使用。可以是浮点数，但受 Python 时间函数的限制，可能不可靠达到几十毫秒。

+   `pool_use_lifo=False` –

    从`QueuePool`检索连接时使用 LIFO（后进先出）而不是 FIFO（先进先出）。使用 LIFO，服务器端超时方案可以在非高峰使用期间减少使用的连接数。在规划服务器端超时时，请确保使用回收或预先 ping 策略优雅地处理过时的连接。

    > 自版本 1.3 新增。
    > 
    > 另请参阅
    > 
    > 使用 FIFO vs. LIFO
    > 
    > 处理断开连接

+   `plugins` –

    要加载的插件名称字符串列表。请参阅`CreateEnginePlugin`获取背景信息。

    自版本 1.2.3 新增。

+   `query_cache_size` –

    用于缓存查询的 SQL 字符串形式的缓存大小。设置为零以禁用缓存。

    当缓存大小达到 N * 1.5 时，将剪辑其最近最少使用的项。默认为 500，这意味着当填充时，缓存将始终存储至少 500 条 SQL 语句，并且将增长到 750 项，然后通过删除最近最少使用的 250 项将其剪裁回 500。

    缓存是通过生成代表语句结构的缓存键来实现的，然后仅当该键不存在于缓存中时，才为当前方言生成字符串 SQL。所有语句都支持缓存，但是某些功能，例如具有大量参数的 INSERT 操作，将有意绕过缓存。SQL 记录将指示每个语句的统计信息，无论是否从缓存中获取。

    注意

    一些与工作单元持久性相关的 ORM 函数以及一些属性加载策略将使用主缓存之外的每个映射器的个别缓存。

    另请参阅

    SQL 编译缓存

    1.4 版中的新功能。

+   `use_insertmanyvalues` –

    默认为 True，对于 INSERT..RETURNING 语句，默认使用“insertmanyvalues”执行样式。

    2.0 版中的新功能。

    另请参阅

    关于 INSERT 语句的“Insert Many Values”行为

```py
function sqlalchemy.engine_from_config(configuration: Dict[str, Any], prefix: str = 'sqlalchemy.', **kwargs: Any) → Engine
```

使用配置字典创建一个新的 Engine 实例。

该字典通常由配置文件生成。

对于`engine_from_config()`感兴趣的键应添加前缀，例如`sqlalchemy.url`，`sqlalchemy.echo`等。 ‘prefix’参数表示要搜索的前缀。每个匹配的键（在剥离前缀后）都被视为对`create_engine()`调用的相应关键字参数。

唯一必需的键是（假设默认前缀为）`sqlalchemy.url`，它提供了数据库 URL。

一组关键字参数将基于字符串值“强制转换”为其预期类型。每个方言可以使用`engine_config_types`访问器进行参数集的可扩展。

参数：

+   `configuration` – 一个字典（通常从配置文件生成，但这不是必需的）。其键以“prefix”值开头的项目将剥离该前缀，然后将传递给`create_engine()`。

+   `prefix` – 匹配并从‘configuration’中的键中剥离的前缀。

+   `kwargs` – `engine_from_config()`的每个关键字参数本身都会覆盖从‘configuration’字典中获取的相应项目。关键字参数不应添加前缀。

```py
function sqlalchemy.create_mock_engine(url: str | URL, executor: Any, **kw: Any) → MockConnection
```

创建用于回显 DDL 的“模拟”引擎。

这是一个用于调试或存储由`MetaData.create_all()`及其相关方法生成的 DDL 序列输出的实用函数。

该函数接受一个 URL，仅用于确定要使用的方言类型，以及一个“executor”可调用函数，该函数将接收 SQL 表达式对象和参数，然后可以回显或以其他方式打印。执行者的返回值不受处理，引擎也不允许调用常规字符串语句，因此仅在将 DDL 发送到数据库而不接收任何结果时有用。

例如：

```py
from sqlalchemy import create_mock_engine

def dump(sql, *multiparams, **params):
    print(sql.compile(dialect=engine.dialect))

engine = create_mock_engine('postgresql+psycopg2://', dump)
metadata.create_all(engine, checkfirst=False)
```

参数：

+   `url` - 一个字符串 URL，通常只需包含数据库后端名称。

+   `executor` - 一个可调用对象，接收参数`sql`，`*multiparams`和`**params`。`sql`参数通常是`ExecutableDDLElement`的实例，可以使用`ExecutableDDLElement.compile()`编译成字符串。

版本 1.4 中新增：- `create_mock_engine()`函数替换了以前与`create_engine()`一起使用的“模拟”引擎策略。

另请参阅

如何将 CREATE TABLE / DROP TABLE 输出为字符串？

```py
function sqlalchemy.engine.make_url(name_or_url: str | URL) → URL
```

给定一个字符串，生成一个新的 URL 实例。

URL 的格式通常遵循[RFC-1738](https://www.ietf.org/rfc/rfc1738.txt)，但也有一些例外，包括“scheme”部分接受下划线而不是破折号或句点。

如果传递了`URL`对象，则原样返回。

另请参阅

数据库 URL

```py
function sqlalchemy.create_pool_from_url(url: str | URL, **kwargs: Any) → Pool
```

从给定的 url 创建一个池实例。

如果未提供`poolclass`，则使用 URL 中指定的方言选择要使用的池类。

传递给`create_pool_from_url()`的参数与传递给`create_engine()`函数的池参数相同。

版本 2.0.10 中新增。

```py
class sqlalchemy.engine.URL
```

表示用于连接到数据库的 URL 的各个组件。

URL 通常是从完整格式的 URL 字符串构建的，其中`make_url()`函数在内部由`create_engine()`函数使用，以将 URL 字符串解析为其各个组件，然后用于构建新的`URL`对象。当从格式化的 URL 字符串解析时，解析格式通常遵循[RFC-1738](https://www.ietf.org/rfc/rfc1738.txt)，但也有一些例外情况。

可以直接生成 `URL` 对象，方法是使用带有完整形式 URL 字符串的 `make_url()` 函数，或者使用 `URL.create()` 构造函数以编程方式给出单个字段来构造 `URL`。生成的 `URL` 对象可以直接传递给 `create_engine()` 以替代字符串参数，这将绕过引擎创建过程中 `make_url()` 的使用。

从版本 1.4 开始更改：`URL` 对象现在是不可变的对象。要创建 URL，请使用 `make_url()` 或 `URL.create()` 函数/方法。要修改 `URL`，请使用类似 `URL.set()` 和 `URL.update_query_dict()` 的方法返回带有修改的新 `URL` 对象。有关此更改的注释，请参阅 URL 对象现在是不可变的。

另请参阅

数据库 URL

`URL` 包含以下属性：

+   `URL.drivername`: 数据库后端和驱动程序名称，例如 `postgresql+psycopg2`

+   `URL.username`: 用户名字符串

+   `URL.password`: 密码字符串

+   `URL.host`: 字符串主机名

+   `URL.port`: 整数端口号

+   `URL.database`: 字符串数据库名称

+   `URL.query`: 表示查询字符串的不可变映射。包含键的字符串和值的字符串或字符串元组。

**成员**

create(), database, difference_update_query(), drivername, get_backend_name(), get_dialect(), get_driver_name(), host, normalized_query, password, port, query, render_as_string(), set(), translate_connect_args(), update_query_dict(), update_query_pairs(), update_query_string(), username

**类签名**

类`sqlalchemy.engine.URL` (`builtins.tuple`)

```py
classmethod create(drivername: str, username: str | None = None, password: str | None = None, host: str | None = None, port: int | None = None, database: str | None = None, query: Mapping[str, Sequence[str] | str] = {}) → URL
```

创建一个新的`URL`对象。

另请参见

数据库 URL

参数：

+   `drivername` – 数据库后端的名称。该名称将对应于 sqlalchemy/databases 模块中的一个模块或第三方插件。

+   `username` – 用户名。

+   `password` –

    数据库密码。通常是一个字符串，但也可以是可以用`str()`字符串化的对象。

    注意

    当作为参数传递给`URL.create()`时，密码字符串不应该进行 URL 编码；该字符串应该包含与用户输入一致的密码字符。

    注意

    每个`Engine`对象只会对密码生成对象进行一次字符串化。对于每次连接的动态密码生成，请参见生成动态认证令牌。

+   `host` – 主机名。

+   `port` – 端口号。

+   `database` – 数据库名称。

+   `query` – 一个将要传递给方言和/或 DBAPI 的字符串键到字符串值的字典。要直接指定非字符串参数给 Python DBAPI，使用`create_engine.connect_args`参数传递给`create_engine()`。另请参见`URL.normalized_query`，用于一个一致为字符串->字符串列表的字典。

返回：

新的`URL`对象。

版本 1.4 中的新功能：`URL`对象现在是**不可变命名元组**。此外，`query`字典也是不可变的。要创建 URL，请使用`make_url()`或`URL.create()`函数/方法。要修改`URL`，请使用`URL.set()`和`URL.update_query()`方法。

```py
attribute database: str | None
```

数据库名称

```py
method difference_update_query(names: Iterable[str]) → URL
```

从`URL.query`字典中删除给定名称，返回新的`URL`。

例如：

```py
url = url.difference_update_query(['foo', 'bar'])
```

等同于使用`URL.set()`如下：

```py
url = url.set(
    query={
        key: url.query[key]
        for key in set(url.query).difference(['foo', 'bar'])
    }
)
```

版本 1.4 中的新功能。

另请参见

`URL.query`

`URL.update_query_dict()`

`URL.set()`

```py
attribute drivername: str
```

数据库后端和驱动程序名称，如`postgresql+psycopg2`

```py
method get_backend_name() → str
```

返回后端名称。

这是与使用的数据库后端对应的名称，是`URL.drivername`中加号左侧的部分。

```py
method get_dialect(_is_async: bool = False) → Type[Dialect]
```

返回与此 URL 的驱动程序名称对应的 SQLAlchemy `Dialect`类。

```py
method get_driver_name() → str
```

返回后端名称。

这是与使用的 DBAPI 驱动程序对应的名称，是`URL.drivername`中加号右侧的部分。

如果`URL.drivername`不包含加号，则导入此`URL`的默认`Dialect`以获取驱动程序名称。

```py
attribute host: str | None
```

主机名或 IP 地址。对于某些驱动程序，也可以是数据源名称。

```py
attribute normalized_query
```

返回带有值规范化为序列的`URL.query`字典。

由于`URL.query`字典可能包含字符串值或字符串值序列，以区分在查询字符串中多次指定的参数，需要通用处理多个参数的代码将希望使用此属性，以便所有存在的参数都呈现为序列。灵感来自 Python 的`urllib.parse.parse_qs`函数。例如：

```py
>>> from sqlalchemy.engine import make_url
>>> url = make_url("postgresql+psycopg2://user:pass@host/dbname?alt_host=host1&alt_host=host2&ssl_cipher=%2Fpath%2Fto%2Fcrt")
>>> url.query
immutabledict({'alt_host': ('host1', 'host2'), 'ssl_cipher': '/path/to/crt'})
>>> url.normalized_query
immutabledict({'alt_host': ('host1', 'host2'), 'ssl_cipher': ('/path/to/crt',)})
```

```py
attribute password: str | None
```

密码，通常是字符串，但也可以是具有`__str__()`方法的任何对象。

```py
attribute port: int | None
```

整数端口号

```py
attribute query: immutabledict[str, Tuple[str, ...] | str]
```

表示查询字符串的不可变映射。键包含字符串，值为字符串或字符串元组，例如：

```py
 >>> from sqlalchemy.engine import make_url
   >>> url = make_url("postgresql+psycopg2://user:pass@host/dbname?alt_host=host1&alt_host=host2&ssl_cipher=%2Fpath%2Fto%2Fcrt")
   >>> url.query
   immutabledict({'alt_host': ('host1', 'host2'), 'ssl_cipher': '/path/to/crt'})

To create a mutable copy of this mapping, use the ``dict`` constructor::

   mutable_query_opts = dict(url.query)
```

另请参阅

`URL.normalized_query` - 将所有值标准化为序列以进行一致处理。

更改 `URL.query` 内容的方法：

`URL.update_query_dict()`

`URL.update_query_string()`

`URL.update_query_pairs()` 

`URL.difference_update_query()`

```py
method render_as_string(hide_password: bool = True) → str
```

渲染此 `URL` 对象为字符串。

此方法在使用 `__str__()` 或 `__repr__()` 方法时使用。该方法直接包括附加选项。

参数：

**hide_password** – 默认为 True。除非设置为 False，否则不显示密码。

```py
method set(drivername: str | None = None, username: str | None = None, password: str | None = None, host: str | None = None, port: int | None = None, database: str | None = None, query: Mapping[str, Sequence[str] | str] | None = None) → URL
```

返回具有修改的新 `URL` 对象。

如果非空则使用值。要明确将值设置为 `None`，请使用从 `namedtuple` 转换的 `URL._replace()` 方法。

参数：

+   `drivername` – 新驱动名称

+   `username` – 新的用户名

+   `password` – 新密码

+   `host` – 新的主机名

+   `port` – 新端口

+   `query` – 新的查询参数，传递一个字符串键的字典，引用字符串或字符串值的序列。完全替换之前的参数列表。

返回：

新 `URL` 对象。

版本 1.4 中新增。

另请参阅

`URL.update_query_dict()`

```py
method translate_connect_args(names: List[str] | None = None, **kw: Any) → Dict[str, Any]
```

将 url 属性转换为连接参数的字典。

将此 url 的属性（主机、数据库、用户名、密码、端口）作为普通字典返回。默认情况下，属性名称用作键。未设置或为假的属性将从最终字典中省略。

参数：

+   `**kw` – 可选，用于 url 属性的替代键名。

+   `names` – 已弃用。与基于关键字的替代名称具有相同目的，但将名称与原始位置相关联。

```py
method update_query_dict(query_parameters: Mapping[str, str | List[str]], append: bool = False) → URL
```

返回一个新的 `URL` 对象，其 `URL.query` 参数字典由给定字典更新。

字典通常包含字符串键和字符串值。为了表示多次表达的查询参数，请传递字符串值序列。

例如：

```py
>>> from sqlalchemy.engine import make_url
>>> url = make_url("postgresql+psycopg2://user:pass@host/dbname")
>>> url = url.update_query_dict({"alt_host": ["host1", "host2"], "ssl_cipher": "/path/to/crt"})
>>> str(url)
'postgresql+psycopg2://user:pass@host/dbname?alt_host=host1&alt_host=host2&ssl_cipher=%2Fpath%2Fto%2Fcrt'
```

参数：

+   `query_parameters` – 具有字符串键和字符串或字符串序列值的字典。

+   `append` – 如果为 True，则现有查询字符串中的参数不会被删除；新参数将添加到已有参数之后。如果保持默认值 False，则给定查询参数中的键将替换现有查询字符串中的键。

新版本 1.4 中添加。

另请参阅

`URL.query`

`URL.update_query_string()`

`URL.update_query_pairs()`

`URL.difference_update_query()`

`URL.set()`

```py
method update_query_pairs(key_value_pairs: Iterable[Tuple[str, str | List[str]]], append: bool = False) → URL
```

返回一个新的 `URL` 对象，其 `URL.query` 参数字典由给定的键值对序列更新

例如：

```py
>>> from sqlalchemy.engine import make_url
>>> url = make_url("postgresql+psycopg2://user:pass@host/dbname")
>>> url = url.update_query_pairs([("alt_host", "host1"), ("alt_host", "host2"), ("ssl_cipher", "/path/to/crt")])
>>> str(url)
'postgresql+psycopg2://user:pass@host/dbname?alt_host=host1&alt_host=host2&ssl_cipher=%2Fpath%2Fto%2Fcrt'
```

参数：

+   `key_value_pairs` – 包含两个字符串的元组序列。

+   `append` – 如果为 True，则现有查询字符串中的参数不会被删除；新参数将添加到已有参数之后。如果保持默认值 False，则给定查询参数中的键将替换现有查询字符串中的键。

新版本 1.4 中添加。

另请参阅

`URL.query`

`URL.difference_update_query()`

`URL.set()`

```py
method update_query_string(query_string: str, append: bool = False) → URL
```

返回一个新的 `URL` 对象，其 `URL.query` 参数字典由给定的查询字符串更新。

例如：

```py
>>> from sqlalchemy.engine import make_url
>>> url = make_url("postgresql+psycopg2://user:pass@host/dbname")
>>> url = url.update_query_string("alt_host=host1&alt_host=host2&ssl_cipher=%2Fpath%2Fto%2Fcrt")
>>> str(url)
'postgresql+psycopg2://user:pass@host/dbname?alt_host=host1&alt_host=host2&ssl_cipher=%2Fpath%2Fto%2Fcrt'
```

参数：

+   `query_string` – 一个 URL 编码的查询字符串，不包括问号。

+   `append` – 如果为 True，则现有查询字符串中的参数不会被删除；新参数将添加到已有参数之后。如果保持默认值 False，则给定查询参数中的键将替换现有查询字符串中的键。

新版本 1.4 中添加。

另请参阅

`URL.query`

`URL.update_query_dict()`

```py
attribute username: str | None
```

用户名 字符串

## 连接池

当调用 `connect()` 或 `execute()` 方法时，`Engine` 将向连接池请求连接。默认的连接池 `QueuePool` 将根据需要打开到数据库的连接。随着并发语句的执行，`QueuePool` 将增加其连接池的大小，默认为五个，并允许默认的 "溢出" 十个。由于 `Engine` 本质上是连接池的“主基地”，因此在应用程序中应该为每个数据库保留一个单独的 `Engine`，而不是为每个连接创建一个新的。

注意

默认情况下，SQLite 引擎不使用 `QueuePool`。有关 SQLite 连接池使用的详细信息，请参阅 SQLite。

有关连接池的更多信息，请参阅 连接池。

自定义 DBAPI connect() 参数 / 连接时例程

对于需要特殊连接方法的情况，在绝大多数情况下，最合适的方法是在 `create_engine()` 级别使用多个钩子来自定义此过程。这些在以下子部分中描述。

### 通过 dbapi.connect() 传递的特殊关键字参数

所有的 Python DBAPI 都接受除了基本连接之外的额外参数。常见参数包括用于指定字符集编码和超时值的参数；更复杂的数据包括特殊的 DBAPI 常量和对象以及 SSL 子参数。有两种简单的方式可以传递这些参数而不复杂化。

#### 将参数添加到 URL 查询字符串中

简单的字符串值，以及一些数字值和布尔标志，通常可以直接在 URL 的查询字符串中指定。一个常见的例子是接受字符编码参数 `encoding` 的 DBAPI，例如大多数 MySQL DBAPI：

```py
engine = create_engine("mysql+pymysql://user:pass@host/test?charset=utf8mb4")
```

使用查询字符串的优点在于可以在配置文件中指定其他的 DBAPI 选项，这样做的方式在 URL 中指定的 DBAPI 方式是可移植的。在此级别传递的具体参数因 SQLAlchemy 方言而异。某些方言将所有参数都作为字符串传递，而其他方言将解析特定的数据类型并将参数移动到不同的位置，例如到驱动程序级别的 DSN 和连接字符串中。由于此领域中方言的行为目前存在差异，因此应该查阅特定方言的文档以查看是否支持在此级别上支持特定参数。

提示

对于给定 URL 显示传递给 DBAPI 的确切参数的一般技术可以直接使用`Dialect.create_connect_args()`方法进行如下操作：

```py
>>> from sqlalchemy import create_engine
>>> engine = create_engine(
...     "mysql+pymysql://some_user:some_pass@some_host/test?charset=utf8mb4"
... )
>>> args, kwargs = engine.dialect.create_connect_args(engine.url)
>>> args, kwargs
([], {'host': 'some_host', 'database': 'test', 'user': 'some_user', 'password': 'some_pass', 'charset': 'utf8mb4', 'client_flag': 2})
```

上述的`args, kwargs`对通常作为`dbapi.connect(*args, **kwargs)`传递给 DBAPI。

#### 使用 connect_args 字典参数

将任何参数传递给保证在任何时候传递所有参数的`dbapi.connect()`函数的更通用的系统是`create_engine.connect_args`字典参数。这可用于否则不被方言处理的参数添加到查询字符串时，以及当需要将特殊子结构或对象传递给 DBAPI 时。有时只是需要将特定标志发送为`True`符号，而 SQLAlchemy 方言并不知道如何将其从 URL 中呈现的字符串形式强制为此关键字参数。下面说明了使用取代连接的基础实现的 psycopg2“连接工厂”的用法：

```py
engine = create_engine(
    "postgresql+psycopg2://user:pass@hostname/dbname",
    connect_args={"connection_factory": MyConnectionFactory},
)
```

另一个示例是 pyodbc 的“timeout”参数：

```py
engine = create_engine(
    "mssql+pyodbc://user:pass@sqlsrvr?driver=ODBC+Driver+13+for+SQL+Server",
    connect_args={"timeout": 30},
)
```

上述示例还说明了 URL“查询字符串”参数以及`create_engine.connect_args`都可以同时使用；在 pyodbc 的情况下，“driver”关键字在 URL 中具有特殊含义。

### 控制参数传递给 DBAPI connect()函数的方式

除了操作传递给`connect()`的参数之外，我们还可以使用`DialectEvents.do_connect()`事件挂钩进一步定制如何调用 DBAPI `connect()`函数本身。此挂钩将传递完整的`*args, **kwargs`，方言将发送到`connect()`。然后，可以在原地修改这些集合以更改它们的使用方式：

```py
from sqlalchemy import event

engine = create_engine("postgresql+psycopg2://user:pass@hostname/dbname")

@event.listens_for(engine, "do_connect")
def receive_do_connect(dialect, conn_rec, cargs, cparams):
    cparams["connection_factory"] = MyConnectionFactory
```

#### 生成动态认证令牌

`DialectEvents.do_connect()`也是一个理想的方法，可以动态插入可能在`Engine`生命周期内更改的认证令牌。例如，如果令牌由`get_authentication_token()`生成并作为`token`参数传递给 DBAPI，则可以实现如下：

```py
from sqlalchemy import event

engine = create_engine("postgresql+psycopg2://user@hostname/dbname")

@event.listens_for(engine, "do_connect")
def provide_token(dialect, conn_rec, cargs, cparams):
    cparams["token"] = get_authentication_token()
```

另请参见

使用访问令牌连接到数据库 - 一个涉及 SQL Server 的更具体的示例

### 修改连接后的 DBAPI 连接或在连接后运行命令

对于 SQLAlchemy 创建的 DBAPI 连接，没有问题，但我们希望在实际使用之前修改完成的连接，例如设置特殊标志或运行某些命令，`PoolEvents.connect()` 事件钩子是最合适的钩子。这个钩子在每次创建新连接时都会被调用，在 SQLAlchemy 使用之前：

```py
from sqlalchemy import event

engine = create_engine("postgresql+psycopg2://user:pass@hostname/dbname")

@event.listens_for(engine, "connect")
def connect(dbapi_connection, connection_record):
    cursor_obj = dbapi_connection.cursor()
    cursor_obj.execute("SET some session variables")
    cursor_obj.close()
```

### 完全替换 DBAPI 的 `connect()` 函数

最后，`DialectEvents.do_connect()` 事件钩子也可以允许我们完全接管连接过程，建立连接并返回它：

```py
from sqlalchemy import event

engine = create_engine("postgresql+psycopg2://user:pass@hostname/dbname")

@event.listens_for(engine, "do_connect")
def receive_do_connect(dialect, conn_rec, cargs, cparams):
    # return the new DBAPI connection with whatever we'd like to
    # do
    return psycopg2.connect(*cargs, **cparams)
```

`DialectEvents.do_connect()` 钩子取代了以前的 `create_engine.creator` 钩子，但前者仍然可用。`DialectEvents.do_connect()` 具有一个明显的优势，就是解析自 URL 的完整参数也会传递给用户定义的函数，而这在 `create_engine.creator` 中不是这样的。 ## 配置日志记录

Python 的标准 [logging](https://docs.python.org/library/logging.html) 模块用于实现 SQLAlchemy 的信息和调试日志输出。这使得 SQLAlchemy 的日志记录可以以标准方式与其他应用程序和库集成。`create_engine()` 还有两个参数 `create_engine.echo` 和 `create_engine.echo_pool`，允许立即将日志记录到 `sys.stdout` 以便进行本地开发；这些参数最终会与下面描述的常规 Python 记录器交互。

本节假设您熟悉上面链接的日志记录模块。SQLAlchemy 所执行的所有日志记录都存在于 `sqlalchemy` 命名空间下，就像 `logging.getLogger('sqlalchemy')` 一样。当配置了日志记录（例如通过 `logging.basicConfig()`），可以打开的 SA 日志记录器的通用命名空间如下：

+   `sqlalchemy.engine` - 控制 SQL 回显。设置为 `logging.INFO` 以输出 SQL 查询，设置为 `logging.DEBUG` 以输出查询 + 结果集。这些设置等同于 `create_engine.echo` 上的 `echo=True` 和 `echo="debug"`。

+   `sqlalchemy.pool` - 控制连接池日志记录。设置为 `logging.INFO` 以记录连接失效和重用事件；设置为 `logging.DEBUG` 以另外记录所有池的签入和签出。这些设置等同于在 `create_engine.echo_pool` 上分别设置 `pool_echo=True` 和 `pool_echo="debug"`。

+   `sqlalchemy.dialects` - 控制用于 SQL 方言的自定义日志记录，日志记录程度在特定方言中使用的情况下通常很少。

+   `sqlalchemy.orm` - 控制在 ORM 中使用日志记录的各种 ORM 函数的日志记录程度，通常很少。设置为 `logging.INFO` 以记录一些关于映射器配置的顶级信息。

例如，要使用 Python 日志记录而不是 `echo=True` 标志来记录 SQL 查询：

```py
import logging

logging.basicConfig()
logging.getLogger("sqlalchemy.engine").setLevel(logging.INFO)
```

默认情况下，整个 `sqlalchemy` 命名空间中的日志级别设置为 `logging.WARN`，以便即使在已启用日志记录的应用程序中，也不会发生任何日志操作。

注意

SQLAlchemy `Engine` 通过仅在检测到当前日志级别为 `logging.INFO` 或 `logging.DEBUG` 时发出日志语句来节省 Python 函数调用开销。它仅在从连接池获取新连接时检查此级别。因此，在已经运行的应用程序中更改日志配置时，任何当前活动的 `Connection`（通常更常见的是活动事务中的 `Session` 对象）将根据新配置不会记录任何 SQL，直到获取新的 `Connection`（对于 `Session`，这是在当前事务结束并开始新事务之后）。

### 关于回显标志的更多信息

如前所述，`create_engine.echo` 和 `create_engine.echo_pool` 参数是立即记录到 `sys.stdout` 的快捷方式：

```py
>>> from sqlalchemy import create_engine, text
>>> e = create_engine("sqlite://", echo=True, echo_pool="debug")
>>> with e.connect() as conn:
...     print(conn.scalar(text("select 'hi'")))
2020-10-24 12:54:57,701 DEBUG sqlalchemy.pool.impl.SingletonThreadPool Created new connection <sqlite3.Connection object at 0x7f287819ac60>
2020-10-24 12:54:57,701 DEBUG sqlalchemy.pool.impl.SingletonThreadPool Connection <sqlite3.Connection object at 0x7f287819ac60> checked out from pool
2020-10-24 12:54:57,702 INFO sqlalchemy.engine.Engine select 'hi'
2020-10-24 12:54:57,702 INFO sqlalchemy.engine.Engine ()
hi
2020-10-24 12:54:57,703 DEBUG sqlalchemy.pool.impl.SingletonThreadPool Connection <sqlite3.Connection object at 0x7f287819ac60> being returned to pool
2020-10-24 12:54:57,704 DEBUG sqlalchemy.pool.impl.SingletonThreadPool Connection <sqlite3.Connection object at 0x7f287819ac60> rollback-on-return
```

使用这些标志大致相当于：

```py
import logging

logging.basicConfig()
logging.getLogger("sqlalchemy.engine").setLevel(logging.INFO)
logging.getLogger("sqlalchemy.pool").setLevel(logging.DEBUG)
```

需要注意的是，这两个标志**独立**于任何现有的日志配置，并且将无条件使用 `logging.basicConfig()`。这会在任何现有的记录器配置之外**额外**进行配置。因此，**在明确配置日志记录时，请始终确保所有回显标志始终设置为 False**，以避免获得重复的日志行。

### 设置日志名称

实例的记录器名称，例如 `Engine` 或 `Pool` 的默认值为使用截断的十六进制标识符字符串。要将其设置为特定名称，请使用 `create_engine.logging_name` 和 `create_engine.pool_logging_name` 与 `sqlalchemy.create_engine()`；名称将附加到记录名称 `sqlalchemy.engine.Engine`：

```py
>>> import logging
>>> from sqlalchemy import create_engine
>>> from sqlalchemy import text
>>> logging.basicConfig()
>>> logging.getLogger("sqlalchemy.engine.Engine.myengine").setLevel(logging.INFO)
>>> e = create_engine("sqlite://", logging_name="myengine")
>>> with e.connect() as conn:
...     conn.execute(text("select 'hi'"))
2020-10-24 12:47:04,291 INFO sqlalchemy.engine.Engine.myengine select 'hi'
2020-10-24 12:47:04,292 INFO sqlalchemy.engine.Engine.myengine ()
```

提示

`create_engine.logging_name` 和 `create_engine.pool_logging_name` 参数也可以与 `create_engine.echo` 和 `create_engine.echo_pool` 一起使用。但是，如果其他引擎的回声标志设置为 True，而没有记录名称，则将发生不可避免的双重记录条件。这是因为将自动为 `sqlalchemy.engine.Engine` 添加一个处理程序，该处理程序将同时记录无名称引擎和具有记录名称的引擎的消息。例如：

```py
from sqlalchemy import create_engine, text

e1 = create_engine("sqlite://", echo=True, logging_name="myname")
with e1.begin() as conn:
    conn.execute(text("SELECT 1"))

e2 = create_engine("sqlite://", echo=True)
with e2.begin() as conn:
    conn.execute(text("SELECT 2"))

with e1.begin() as conn:
    conn.execute(text("SELECT 3"))
```

上述场景将双重记录 `SELECT 3`。要解决此问题，请确保所有引擎都设置了 `logging_name`，或者使用显式记录器/处理程序设置，而不使用 `create_engine.echo` 和 `create_engine.echo_pool`。

### 设置每个连接/子引擎令牌

1.4.0b2 版本中的新功能。

当记录名称适合于在长时间存在的 `Engine` 对象上建立时，它并不灵活到足以容纳任意大的名称列表，用于跟踪日志消息中的单个连接和/或事务的情况。

对于这种用例，由 `Connection` 和 `Result` 对象生成的日志消息本身可以使用其他令牌进行增强，例如事务或请求标识符。 `Connection.execution_options.logging_token` 参数接受一个字符串参数，该参数可用于建立每个连接的跟踪令牌：

```py
>>> from sqlalchemy import create_engine
>>> e = create_engine("sqlite://", echo="debug")
>>> with e.connect().execution_options(logging_token="track1") as conn:
...     conn.execute(text("select 1")).all()
2021-02-03 11:48:45,754 INFO sqlalchemy.engine.Engine [track1] select 1
2021-02-03 11:48:45,754 INFO sqlalchemy.engine.Engine [track1] [raw sql] ()
2021-02-03 11:48:45,754 DEBUG sqlalchemy.engine.Engine [track1] Col ('1',)
2021-02-03 11:48:45,755 DEBUG sqlalchemy.engine.Engine [track1] Row (1,)
```

`Connection.execution_options.logging_token`参数也可以通过`create_engine.execution_options`或`Engine.execution_options()`在引擎或子引擎上建立。这可能对应用程序的不同组件应用不同的日志令牌而无需创建新引擎很有用：

```py
>>> from sqlalchemy import create_engine
>>> e = create_engine("sqlite://", echo="debug")
>>> e1 = e.execution_options(logging_token="track1")
>>> e2 = e.execution_options(logging_token="track2")
>>> with e1.connect() as conn:
...     conn.execute(text("select 1")).all()
2021-02-03 11:51:08,960 INFO sqlalchemy.engine.Engine [track1] select 1
2021-02-03 11:51:08,960 INFO sqlalchemy.engine.Engine [track1] [raw sql] ()
2021-02-03 11:51:08,960 DEBUG sqlalchemy.engine.Engine [track1] Col ('1',)
2021-02-03 11:51:08,961 DEBUG sqlalchemy.engine.Engine [track1] Row (1,)

>>> with e2.connect() as conn:
...     conn.execute(text("select 2")).all()
2021-02-03 11:52:05,518 INFO sqlalchemy.engine.Engine [track2] Select 1
2021-02-03 11:52:05,519 INFO sqlalchemy.engine.Engine [track2] [raw sql] ()
2021-02-03 11:52:05,520 DEBUG sqlalchemy.engine.Engine [track2] Col ('1',)
2021-02-03 11:52:05,520 DEBUG sqlalchemy.engine.Engine [track2] Row (1,)
```

### 隐藏参数

`Engine`发出的日志还指示了特定语句中存在的 SQL 参数的摘录。为了防止出于隐私目的记录这些参数，启用`create_engine.hide_parameters`标志：

```py
>>> e = create_engine("sqlite://", echo=True, hide_parameters=True)
>>> with e.connect() as conn:
...     conn.execute(text("select :some_private_name"), {"some_private_name": "pii"})
2020-10-24 12:48:32,808 INFO sqlalchemy.engine.Engine select ?
2020-10-24 12:48:32,808 INFO sqlalchemy.engine.Engine [SQL parameters hidden due to hide_parameters=True]
```  ## 支持的数据库

SQLAlchemy 包括许多用于各种后端的`Dialect`实现。SQLAlchemy 包含最常见数据库的方言；另外一些需要额外安装单独的方言。

查看 Dialects 部分，了解可用的各种后端信息。

## 数据库 URL

`create_engine()`函数基于 URL 生成一个`Engine`对象。URL 的格式通常遵循[RFC-1738](https://www.ietf.org/rfc/rfc1738.txt)，但也有一些例外，包括“scheme”部分接受下划线而不是破折号或句点。URL 通常包括用户名、密码、主机名、数据库名字段，以及用于额外配置的可选关键字参数。在某些情况下，接受文件路径，而在其他情况下，“数据源名称”取代“主机”和“数据库”部分。数据库 URL 的典型形式为：

```py
dialect+driver://username:password@host:port/database
```

方言名称包括 SQLAlchemy 方言的标识名称，如`sqlite`、`mysql`、`postgresql`、`oracle`或`mssql`。驱动程序名称是用于使用所有小写字母连接到数据库的 DBAPI 的名称。如果未指定，将导入“默认”DBAPI（如果可用）- 这个默认通常是该后端可用的最广为人知的驱动程序。

### 转义特殊字符，如密码中的@符号

在构建完整的 URL 字符串以传递给`create_engine()`时，**特殊字符（如用户和密码中可能使用的字符）需要进行 URL 编码以正确解析。**。**这包括@符号**。

以下是包含密码 `"kx@jj5/g"` 的 URL 的示例，其中“at” 符号和斜杠字符分别表示为 `%40` 和 `%2F`：

```py
postgresql+pg8000://dbuser:kx%40jj5%2Fg@pghost10/appdb
```

上述密码的编码可以使用 [urllib.parse](https://docs.python.org/3/library/urllib.parse.html) 生成：

```py
>>> import urllib.parse
>>> urllib.parse.quote_plus("kx@jj5/g")
'kx%40jj5%2Fg'
```

然后 URL 可以作为字符串传递给 `create_engine()`：

```py
from sqlalchemy import create_engine

engine = create_engine("postgresql+pg8000://dbuser:kx%40jj5%2Fg@pghost10/appdb")
```

作为在创建完整 URL 字符串时替代转义特殊字符的选择，传递给 `create_engine()` 的对象可以是 `URL` 对象的实例，它绕过了解析阶段，可以直接容纳未转义的字符串。查看下一节以获取示例。

1.4 版本中的变化：已修复主机名和数据库名中 `@` 符号的支持。作为此修复的副作用，密码中的 `@` 符号必须转义。

### 以编程方式创建 URL

传递给 `create_engine()` 的值可以是 `URL` 的实例，而不是简单的字符串，这样可以绕过使用字符串解析的需要，因此不需要提供转义的 URL 字符串。

使用 `URL.create()` 构造方法创建 `URL` 对象，逐个传递所有字段。密码中的特殊字符可以不作任何修改地传递：

```py
from sqlalchemy import URL

url_object = URL.create(
    "postgresql+pg8000",
    username="dbuser",
    password="kx@jj5/g",  # plain (unescaped) text
    host="pghost10",
    database="appdb",
)
```

构造的 `URL` 对象然后可以直接传递给 `create_engine()` 代替字符串参数：

```py
from sqlalchemy import create_engine

engine = create_engine(url_object)
```

另请参阅

`URL`

`URL.create()`

### 特定后端的 URL

下面是常见连接样式的示例。有关所有包含方言的详细信息以及第三方方言的链接的完整索引，请参见 方言。

#### PostgreSQL

PostgreSQL 方言使用 psycopg2 作为默认的 DBAPI。其他 PostgreSQL DBAPI 包括 pg8000 和 asyncpg：

```py
# default
engine = create_engine("postgresql://scott:tiger@localhost/mydatabase")

# psycopg2
engine = create_engine("postgresql+psycopg2://scott:tiger@localhost/mydatabase")

# pg8000
engine = create_engine("postgresql+pg8000://scott:tiger@localhost/mydatabase")
```

关于连接到 PostgreSQL 的更多说明请参见 PostgreSQL。

#### MySQL

MySQL 方言使用 mysqlclient 作为默认的 DBAPI。还有其他可用的 MySQL DBAPI，包括 PyMySQL：

```py
# default
engine = create_engine("mysql://scott:tiger@localhost/foo")

# mysqlclient (a maintained fork of MySQL-Python)
engine = create_engine("mysql+mysqldb://scott:tiger@localhost/foo")

# PyMySQL
engine = create_engine("mysql+pymysql://scott:tiger@localhost/foo")
```

关于连接到 MySQL 的更多说明请参见 MySQL 和 MariaDB。

#### Oracle

Oracle 方言使用 cx_oracle 作为默认的 DBAPI：

```py
engine = create_engine("oracle://scott:tiger@127.0.0.1:1521/sidname")

engine = create_engine("oracle+cx_oracle://scott:tiger@tnsname")
```

关于连接到 Oracle 的更多说明请参见 Oracle。

#### Microsoft SQL Server

SQL Server 方言使用 pyodbc 作为默认的 DBAPI。pymssql 也可用：

```py
# pyodbc
engine = create_engine("mssql+pyodbc://scott:tiger@mydsn")

# pymssql
engine = create_engine("mssql+pymssql://scott:tiger@hostname:port/dbname")
```

关于连接到 SQL Server 的更多说明请参见 Microsoft SQL Server。

#### SQLite

SQLite 连接到基于文件的数据库，默认使用 Python 内置模块`sqlite3`。

由于 SQLite 连接到本地文件，URL 格式略有不同。URL 的“file”部分是数据库的文件名。对于相对文件路径，这需要三个斜杠：

```py
# sqlite://<nohostname>/<path>
# where <path> is relative:
engine = create_engine("sqlite:///foo.db")
```

对于绝对文件路径，三个斜杠后跟绝对路径：

```py
# Unix/Mac - 4 initial slashes in total
engine = create_engine("sqlite:////absolute/path/to/foo.db")

# Windows
engine = create_engine("sqlite:///C:\\path\\to\\foo.db")

# Windows alternative using raw string
engine = create_engine(r"sqlite:///C:\path\to\foo.db")
```

要使用 SQLite `:memory:`数据库，请指定空 URL：

```py
engine = create_engine("sqlite://")
```

关于连接到 SQLite 的更多说明请参见 SQLite。

#### 其他

请参阅 Dialects，所有其他方言文档的顶级页面。

### 转义密码中的特殊字符，例如@符号

在构造完整的 URL 字符串以传递给`create_engine()`时，**需要对特殊字符进行 URL 编码才能正确解析**。**这包括@符号**。

下面是一个包含密码`"kx@jj5/g"`的 URL 示例，其中“at”符号和斜杠字符分别表示为`%40`和`%2F`：

```py
postgresql+pg8000://dbuser:kx%40jj5%2Fg@pghost10/appdb
```

上述密码的编码可以使用[urllib.parse](https://docs.python.org/3/library/urllib.parse.html)生成：

```py
>>> import urllib.parse
>>> urllib.parse.quote_plus("kx@jj5/g")
'kx%40jj5%2Fg'
```

然后 URL 可以作为字符串传递给`create_engine()`：

```py
from sqlalchemy import create_engine

engine = create_engine("postgresql+pg8000://dbuser:kx%40jj5%2Fg@pghost10/appdb")
```

作为在创建完整 URL 字符串时转义特殊字符的替代方法，传递给`create_engine()`的对象可以是`URL`对象的实例，它会绕过解析阶段并可以直接适应未转义的字符串。请参阅下一节的示例。

从版本 1.4 开始更改：修复了对主机名和数据库名中的`@`符号的支持。由于此修复的副作用，密码中的`@`符号必须转义。

### 以编程方式创建 URL

传递给`create_engine()`的值可以是`URL`的实例，而不是简单的字符串，这将绕过使用字符串解析的需要，因此不需要提供已转义的 URL 字符串。

`URL`对象是使用`URL.create()`构造方法创建的，分别传递所有字段。例如密码中的特殊字符可以直接传递而无需任何修改：

```py
from sqlalchemy import URL

url_object = URL.create(
    "postgresql+pg8000",
    username="dbuser",
    password="kx@jj5/g",  # plain (unescaped) text
    host="pghost10",
    database="appdb",
)
```

然后构造的`URL`对象可以直接传递给`create_engine()`以替代字符串参数：

```py
from sqlalchemy import create_engine

engine = create_engine(url_object)
```

另请参阅

`URL`

`URL.create()`

### 后端特定的 URL

下面是常见连接样式的示例。有关所有包含方言的详细信息以及链接到第三方方言的链接的完整索引，请参阅方言。

#### PostgreSQL

PostgreSQL 方言默认使用 psycopg2 作为默认的 DBAPI。其他 PostgreSQL DBAPI 包括 pg8000 和 asyncpg：

```py
# default
engine = create_engine("postgresql://scott:tiger@localhost/mydatabase")

# psycopg2
engine = create_engine("postgresql+psycopg2://scott:tiger@localhost/mydatabase")

# pg8000
engine = create_engine("postgresql+pg8000://scott:tiger@localhost/mydatabase")
```

在 PostgreSQL 中有关连接到 PostgreSQL 的更多注意事项。

#### MySQL

MySQL 方言默认使用 mysqlclient 作为默认的 DBAPI。还有其他可用的 MySQL DBAPI，包括 PyMySQL：

```py
# default
engine = create_engine("mysql://scott:tiger@localhost/foo")

# mysqlclient (a maintained fork of MySQL-Python)
engine = create_engine("mysql+mysqldb://scott:tiger@localhost/foo")

# PyMySQL
engine = create_engine("mysql+pymysql://scott:tiger@localhost/foo")
```

在 MySQL 和 MariaDB 中有关连接到 MySQL 的更多注意事项。

#### Oracle

Oracle 方言默认使用 cx_oracle 作为默认的 DBAPI：

```py
engine = create_engine("oracle://scott:tiger@127.0.0.1:1521/sidname")

engine = create_engine("oracle+cx_oracle://scott:tiger@tnsname")
```

在 Oracle 中有关连接到 Oracle 的更多注意事项。

#### Microsoft SQL Server

SQL Server 方言默认使用 pyodbc 作为默认的 DBAPI。pymssql 也可用：

```py
# pyodbc
engine = create_engine("mssql+pyodbc://scott:tiger@mydsn")

# pymssql
engine = create_engine("mssql+pymssql://scott:tiger@hostname:port/dbname")
```

在 Microsoft SQL Server 中有关连接到 SQL Server 的更多注意事项。

#### SQLite

SQLite 连接到基于文件的数据库，默认情况下使用 Python 内置模块`sqlite3`。

由于 SQLite 连接到本地文件，URL 格式略有不同。 URL 的“file”部分是数据库的文件名。对于相对文件路径，这需要三个斜杠：

```py
# sqlite://<nohostname>/<path>
# where <path> is relative:
engine = create_engine("sqlite:///foo.db")
```

对于绝对文件路径，三个斜杠后跟绝对路径：

```py
# Unix/Mac - 4 initial slashes in total
engine = create_engine("sqlite:////absolute/path/to/foo.db")

# Windows
engine = create_engine("sqlite:///C:\\path\\to\\foo.db")

# Windows alternative using raw string
engine = create_engine(r"sqlite:///C:\path\to\foo.db")
```

要使用 SQLite `:memory:` 数据库，请指定一个空 URL：

```py
engine = create_engine("sqlite://")
```

在 SQLite 中有关连接到 SQLite 的更多注意事项。

#### 其他

请参阅方言，这是所有额外方言文档的顶级页面。

#### PostgreSQL

PostgreSQL 方言默认使用 psycopg2 作为默认的 DBAPI。其他 PostgreSQL DBAPI 包括 pg8000 和 asyncpg：

```py
# default
engine = create_engine("postgresql://scott:tiger@localhost/mydatabase")

# psycopg2
engine = create_engine("postgresql+psycopg2://scott:tiger@localhost/mydatabase")

# pg8000
engine = create_engine("postgresql+pg8000://scott:tiger@localhost/mydatabase")
```

在 PostgreSQL 中有关连接到 PostgreSQL 的更多注意事项。

#### MySQL

MySQL 方言默认使用 mysqlclient 作为默认的 DBAPI。还有其他可用的 MySQL DBAPI，包括 PyMySQL：

```py
# default
engine = create_engine("mysql://scott:tiger@localhost/foo")

# mysqlclient (a maintained fork of MySQL-Python)
engine = create_engine("mysql+mysqldb://scott:tiger@localhost/foo")

# PyMySQL
engine = create_engine("mysql+pymysql://scott:tiger@localhost/foo")
```

在 MySQL 和 MariaDB 中有关连接到 MySQL 的更多注意事项。

#### Oracle

Oracle 方言默认使用 cx_oracle 作为默认的 DBAPI：

```py
engine = create_engine("oracle://scott:tiger@127.0.0.1:1521/sidname")

engine = create_engine("oracle+cx_oracle://scott:tiger@tnsname")
```

在 Oracle 中有关连接到 Oracle 的更多注意事项。

#### Microsoft SQL Server

SQL Server 方言默认使用 pyodbc 作为默认的 DBAPI。pymssql 也可用：

```py
# pyodbc
engine = create_engine("mssql+pyodbc://scott:tiger@mydsn")

# pymssql
engine = create_engine("mssql+pymssql://scott:tiger@hostname:port/dbname")
```

在 Microsoft SQL Server 中有关连接到 SQL Server 的更多注意事项。

#### SQLite

SQLite 连接到基于文件的数据库，默认情况下使用 Python 内置模块`sqlite3`。

由于 SQLite 连接到本地文件，URL 格式略有不同。 URL 的“file”部分是数据库的文件名。对于相对文件路径，这需要三个斜杠：

```py
# sqlite://<nohostname>/<path>
# where <path> is relative:
engine = create_engine("sqlite:///foo.db")
```

对于绝对文件路径，三个斜杠后跟绝对路径：

```py
# Unix/Mac - 4 initial slashes in total
engine = create_engine("sqlite:////absolute/path/to/foo.db")

# Windows
engine = create_engine("sqlite:///C:\\path\\to\\foo.db")

# Windows alternative using raw string
engine = create_engine(r"sqlite:///C:\path\to\foo.db")
```

要使用 SQLite `:memory:` 数据库，请指定一个空 URL：

```py
engine = create_engine("sqlite://")
```

在 SQLite 中有关连接到 SQLite 的更多注意事项。

#### 其他

请参阅方言，这是所有额外方言文档的顶级页面。

## 引擎创建 API

| 对象名称 | 描述 |
| --- | --- |
| create_engine(url, **kwargs) | 创建一个新的 `Engine` 实例。 |
| create_mock_engine(url, executor, **kw) | 创建用于回显 DDL 的“模拟”引擎。 |
| create_pool_from_url(url, **kwargs) | 根据给定的 URL 创建一个池实例。 |
| engine_from_config(configuration[, prefix], **kwargs) | 使用配置字典创建一个新的 Engine 实例。 |
| make_url(name_or_url) | 给定一个字符串，生成一个新的 URL 实例。 |
| URL | 表示用于连接到数据库的 URL 的组件。 |

```py
function sqlalchemy.create_engine(url: str | _url.URL, **kwargs: Any) → Engine
```

创建一个新的 `Engine` 实例。

标准的调用形式是将 URL 作为第一个位置参数发送，通常是一个指示数据库方言和连接参数的字符串：

```py
engine = create_engine("postgresql+psycopg2://scott:tiger@localhost/test")
```

注意

请参阅数据库 URL 以获取编写 URL 字符串的一般准则。特别是，特殊字符（例如密码中经常包含的字符）必须进行 URL 编码才能正确解析。

随后可以跟随其他关键字参数，这些参数将在生成的 `Engine` 及其基础的 `Dialect` 和 `Pool` 构造上建立各种选项：

```py
engine = create_engine("mysql+mysqldb://scott:tiger@hostname/dbname",
                            pool_recycle=3600, echo=True)
```

URL 的字符串形式为 `dialect[+driver]://user:password@host/dbname[?key=value..]`，其中 `dialect` 是数据库名称，例如 `mysql`、`oracle`、`postgresql` 等，而 `driver` 是 DBAPI 的名称，例如 `psycopg2`、`pyodbc`、`cx_oracle` 等。另外，URL 还可以是 `URL` 的实例。

`**kwargs` 接受各种路由到其适当组件的选项。参数可能特定于 `Engine`、底层 `Dialect`，以及 `Pool`。特定方言还接受特定于该方言的关键字参数。在这里，我们描述了大多数 `create_engine()` 使用的通用参数。

一旦建立，新生成的 `Engine` 将在调用 `Engine.connect()` 或依赖于它的方法，例如调用 `Engine.execute()` 时从底层 `Pool` 请求连接。当收到此请求时，`Pool` 反过来将建立第一个实际的 DBAPI 连接。 `create_engine()` 调用本身**不会**直接建立任何实际的 DBAPI 连接。

另请参阅

引擎配置

方言

使用引擎和连接

参数：

+   `connect_args` – 一个选项字典，将作为额外的关键字参数直接传递给 DBAPI 的 `connect()` 方法。请参阅自定义 DBAPI connect() 参数 / on-connect routines 的示例。

+   `creator` –

    返回 DBAPI 连接的可调用对象。此创建函数将传递给底层连接池，并将用于创建所有新的数据库连接。使用此函数会使 URL 参数中指定的连接参数被绕过。

    此钩子不如较新的 `DialectEvents.do_connect()` 钩子灵活，后者允许完全控制在数据库中如何建立连接，给定全套 URL 参数和先前的状态。

    另请参阅

    `DialectEvents.do_connect()` - 事件钩子，允许完全控制 DBAPI 连接机制。

    自定义 DBAPI connect() 参数 / on-connect routines

+   `echo=False` –

    如果为 True，则引擎将记录所有语句以及它们参数列表的`repr()`到默认的日志处理程序，默认情况下为`sys.stdout`输出。如果设置为字符串`"debug"`，则结果行也将打印到标准输出。`Engine`的`echo`属性可以随时修改以打开和关闭日志记录；也可以使用标准的 Python `logging` 模块直接控制日志记录。

    另请参阅

    配置日志记录 - 如何配置日志记录的更多细节。

+   `echo_pool=False` –

    如果为 True，则连接池将记录信息输出，例如当连接无效时以及当连接被回收到默认日志处理程序时，该默认日志处理程序默认为`sys.stdout`用于输出。如果设置为字符串`"debug"`，则日志将包括池检出和签入。也可以使用标准的 Python `logging` 模块进行日志记录。

    另请参阅

    配置日志记录 - 关于如何配置日志记录的更多详细信息。

+   `empty_in_strategy` –

    不再使用；SQLAlchemy 现在在所有情况下都使用“空集”行为来处理 IN。

    自 1.4 版本起弃用：`create_engine.empty_in_strategy`关键字已弃用，不再起作用。现在所有 IN 表达式都使用“扩展参数”策略进行渲染，在语句执行时渲染一组 boundexpressions，或者一个“空集”SELECT。

+   `enable_from_linting` –

    默认为 True。如果发现给定的 SELECT 语句具有未链接的 FROM 元素，可能导致笛卡尔积，则会发出警告。

    1.4 版本中的新功能。

    另请参阅

    内置的 FROM linting 将警告任何 SELECT 语句中可能存在的笛卡尔积

+   `execution_options` – 将应用于所有连接的字典执行选项。参见`Connection.execution_options()`

+   `future` –

    使用 2.0 风格的`Engine`和`Connection` API。

    从 SQLAlchemy 2.0 开始，此���数仅用于向后兼容，必须保持其默认值为`True`。

    `create_engine.future`参数将在随后的 2.x 版本中被弃用，并最终移除。

    1.4 版本中的新功能。

    2.0 版本中的更改：所有`Engine`对象都是“future”风格的引擎，不再有`future=False`操作模式。

    另请参阅

    SQLAlchemy 2.0 - 主要迁移指南

+   `hide_parameters` –

    布尔值，当设置为 True 时，SQL 语句参数将不会显示在 INFO 日志中，也不会被格式化为`StatementError`对象的字符串表示。

    1.3.8 版本中的新功能。

    另请参阅

    配置日志记录 - 关于如何配置日志记录的更多详细信息。

+   `implicit_returning=True` – 旧参数，只能设置为 True。在 SQLAlchemy 2.0 中，此参数不起作用。为了禁用 ORM 调用的语句的“隐式返回”，请在每个表上使用`Table.implicit_returning`参数进行配置。

+   `insertmanyvalues_page_size` –

    在使用“insertmanyvalues”模式的语句格式化为 INSERT 语句时，插入的行数。该模式是用于许多后端的分页形式的批量插入，在使用 executemany 执行时通常与 RETURNING 一起使用。默认为 1000，但也可能受到方言特定的限制因素的影响，这些因素可能会在每个语句的基础上覆盖此值。

    自 2.0 版本起新增。

    另请参见

    “插入多个值”INSERT 语句行为

    控制批量大小

    `Connection.execution_options.insertmanyvalues_page_size`

+   `isolation_level` –

    必要时，所有新连接上都将设置的隔离级别的可选字符串名称。隔离级别通常是基于后端的字符串名称`"SERIALIZABLE"`、`"REPEATABLE READ"`、`"READ COMMITTED"`、`"READ UNCOMMITTED"`和`"AUTOCOMMIT"`的某些子集。

    `create_engine.isolation_level`参数与`Connection.execution_options.isolation_level`执行选项形成对比，后者可在单个`Connection`上设置，以及传递给`Engine.execution_options()`的相同参数，在那里可以用来创建具有不同隔离级别的多个引擎，这些引擎共享一个公共连接池和方言。

    从 2.0 版本开始更改：`create_engine.isolation_level`参数已泛化，适用于所有支持隔离级别概念的方言，并提供作为更简洁、直接的配置开关，与执行选项形成对比，后者更像是一种临时编程选项。

    另请参见

    设置事务隔离级别，包括 DBAPI 自动提交

+   `json_deserializer` –

    对于支持`JSON`数据类型的方言，这是一个 Python 可调用对象，它将 JSON 字符串转换为 Python 对象。默认情况下，使用 Python 的`json.loads`函数。

    从 1.3.7 版本开始更改：SQLite 方言将其从`_json_deserializer`改名为此。

+   `json_serializer` –

    对于支持 `JSON` 数据类型的方言，这是一个 Python 可调用函数，它将给定对象呈现为 JSON。默认情况下，使用 Python 的 `json.dumps` 函数。

    从版本 `1.3.7` 开始更改：SQLite 方言将其从 `_json_serializer` 重命名为 `_json_serializer`。

+   `label_length=None` -

    可选整数值，将动态生成的列标签的大小限制为该数目的字符数。如果小于 6，则标签将生成为“_(计数器)”。如果为 `None`，则使用 `dialect.max_identifier_length` 的值，可以通过 `create_engine.max_identifier_length` 参数进行设置。`create_engine.label_length` 的值不能大于 `create_engine.max_identfier_length` 的值。

    另请参阅

    `create_engine.max_identifier_length`

+   `logging_name` -

    用于在 “sqlalchemy.engine” 记录器中生成的日志记录的 “name” 字段中使用的字符串标识符。默认为对象 id 的十六进制字符串。

    另请参阅

    配置日志记录 - 关于如何配置日志记录的更多详细信息。

    `Connection.execution_options.logging_token`

+   `max_identifier_length` -

    整数；覆盖由方言确定的 max_identifier_length。如果为 `None` 或零，则无效。这是数据库配置的最大字符数，可用于 SQL 标识符（如表名、列名或标签名）。所有方言都会自动确定此值，但是在新数据库版本中，此值已更改但未调整 SQLAlchemy 方言的情况下，可以将值传递到此处。

    从版本 `1.3.9` 新增。

    另请参阅

    `create_engine.label_length`

+   `max_overflow=10` - 允许在连接池 “溢出” 中打开的连接数，即可以超过 pool_size 设置的连接数，后者默认为五。仅在 `QueuePool` 中使用。

+   `module=None` – 指向 Python 模块对象的引用（模块本身，而不是其字符串名称）。指定引擎方言要使用的备用 DBAPI 模块。每个子方言引用一个特定的 DBAPI，该 DBAPI 将在第一次连接之前导入。此参数导致跳过导入，并改用给定的模块。不仅可用于 DBAPI 的测试，还可用于将“模拟”DBAPI 实现注入到`Engine`中。

+   `paramstyle=None` – 渲染绑定参数时使用的[paramstyle](https://legacy.python.org/dev/peps/pep-0249/#paramstyle)。此样式默认为 DBAPI 本身推荐的样式，该样式从 DBAPI 的`.paramstyle`属性中获取。然而，大多数 DBAPI 都接受多种 paramstyle，特别是将“命名”paramstyle 转换为“位置”paramstyle，或者反之，可能是可取的。当传递此属性时，它应该是以下值之一：`"qmark"`、`"numeric"`、`"named"`、`"format"`或`"pyformat"`，并且应该对应于所使用的 DBAPI 已知支持的参数样式。

+   `pool=None` – `Pool`的一个已构造实例，例如`QueuePool`实例。如果非 None，则此池将直接用作引擎的底层连接池，跳过 URL 参数中存在的任何连接参数。有关手动构造连接池的信息，请参阅连接池。

+   `poolclass=None` – 一个`Pool`子类，将使用 URL 中给定的连接参数来创建连接池实例。请注意，这与`pool`不同，因为在这种情况下，您实际上并不实例化池，而是指示要使用何种类型的池。

+   `pool_logging_name` –

    将在“sqlalchemy.pool”记录器中生成的日志记录的“name”字段内使用的字符串标识符。默认为对象 id 的十六进制字符串。

    另请参阅

    配置日志记录 - 关于如何配置日志记录的更多详细信息。

+   `pool_pre_ping` –

    如果为 True，则启用连接池“预检”功能，该功能在每次检出时测试连接的活动性。

    自版本 1.2 起新增。

    另请参阅

    断开处理 - 悲观

+   `pool_size=5` – 在连接池中保持打开的连接数。与 `QueuePool` 以及 `SingletonThreadPool` 一起使用。对于 `QueuePool`，`pool_size` 设置为 0 表示没有限制；要禁用池化，请将 `poolclass` 设置为 `NullPool`。

+   `pool_recycle=-1` –

    此设置会导致池在经过给定的秒数后重新利用连接。默认值为-1，或者没有超时。例如，将其设置为 3600 意味着连接将在一小时后重新利用。请注意，特别是 MySQL 会在连接上检测不到任何活动时自动断开连接，时间为八小时（尽管可以通过 MySQLDB 连接本身和服务器配置进行配置）。

    另请参阅

    设置池回收

+   `pool_reset_on_return='rollback'` –

    设置底层 `Pool` 对象的 `Pool.reset_on_return` 参数，该参数可以设置为值 `"rollback"`、`"commit"` 或 `None`。

    另请参阅

    返回时重置

+   `pool_timeout=30` –

    从池中获取连接之前等待的秒数。仅与 `QueuePool` 一起使用。这可以是一个浮点数，但受 Python 时间函数的限制，可能不可靠在十几毫秒的范围内。

+   `pool_use_lifo=False` –

    当从 `QueuePool` 检索连接时使用 LIFO（后进先出）而不是 FIFO（先进先出）。使用 LIFO，服务器端的超时方案可以在非高峰使用期间减少使用的连接数。在规划服务器端超时时，请确保使用回收或预先 ping 策略以优雅地处理陈旧的连接。

    > 版本 1.3 中的新功能。
    > 
    > 另请参阅
    > 
    > 使用 FIFO vs. LIFO
    > 
    > 处理断开连接

+   `plugins` –

    要加载的插件名称的字符串列表。有关背景，请参阅 `CreateEnginePlugin`。

    版本 1.2.3 中的新功能。

+   `query_cache_size` –

    用于缓存查询的 SQL 字符串形式的缓存大小。将其设置为零以禁用缓存。

    当缓存大小达到 N * 1.5 时，最近最少使用的项目将被清理。默认为 500，意味着缓存填满时始终至少存储 500 条 SQL 语句，并且会增长到 750 个项目，然后通过删除最近最少使用的 250 个项目将其减少到 500。

    缓存是在每个语句的基础上完成的，方法是生成一个表示语句结构的缓存键，然后仅当该键不在缓存中时才为当前方言生成字符串 SQL。所有语句都支持缓存，但是一些特性，如具有大量参数的 INSERT，将有意绕过缓存。SQL 日志将指示每个语句的统计信息，无论是否从缓存中拉取。

    注意

    一些与工作单元持久性相关的 ORM 函数以及一些属性加载策略将使用主缓存之外的每个映射器缓存。

    另请参阅

    SQL 编译缓存

    版本 1.4 中的新功能。

+   `use_insertmanyvalues` –

    默认为 True，对于 INSERT..RETURNING 语句，默认使用“insertmanyvalues”执行样式。

    版本 2.0 中的新功能。

    另请参阅

    INSERT 语句的“插入多个值”行为

```py
function sqlalchemy.engine_from_config(configuration: Dict[str, Any], prefix: str = 'sqlalchemy.', **kwargs: Any) → Engine
```

使用配置字典创建一个新的 Engine 实例。

该字典通常由配置文件生成。

对于 `engine_from_config()` 感兴趣的键应该有前缀，例如 `sqlalchemy.url`，`sqlalchemy.echo` 等。‘prefix’参数指示要搜索的前缀。每个匹配键（前缀被剥离后）都被视为对 `create_engine()` 调用的相应关键字参数。

唯一必需的键是（假设默认前缀为）`sqlalchemy.url`，它提供了数据库 URL。

一组选择的关键字参数将根据字符串值“强制转换”为其预期类型。使用 `engine_config_types` 访问器可针对每个方言扩展参数集。

参数：

+   `configuration` – 一个字典（通常由配置文件生成，但这不是必需的）。键以“prefix”值开头的项目将删除该前缀，然后传递给 `create_engine()`。

+   `prefix` – 用于在“configuration”中匹配并剥离键的前缀。

+   `kwargs` – 传递给 `engine_from_config()` 的每个关键字参数本身都会覆盖从“configuration”字典中获取的相应项目。关键字参数不应有前缀。

```py
function sqlalchemy.create_mock_engine(url: str | URL, executor: Any, **kw: Any) → MockConnection
```

创建用于回显 DDL 的“模拟”引擎。

这是一个用于调试或存储由 `MetaData.create_all()` 和相关方法生成的 DDL 序列输出的实用函数。

该函数接受一个 URL 参数，仅用于确定要使用的方言类型，以及一个“执行器”可调用函数，该函数将接收一个 SQL 表达式对象和参数，然后可以将其回显或以其他方式打印。执行器的返回值不受处理，也不允许调用常规字符串语句，因此仅对发送到数据库的 DDL 有用而不接收任何结果。

例如：

```py
from sqlalchemy import create_mock_engine

def dump(sql, *multiparams, **params):
    print(sql.compile(dialect=engine.dialect))

engine = create_mock_engine('postgresql+psycopg2://', dump)
metadata.create_all(engine, checkfirst=False)
```

参数：

+   `url` – 一个字符串 URL，通常只需包含数据库后端名称。

+   `executor` – 一个可调用的函数，接收参数 `sql`、`*multiparams` 和 `**params`。`sql` 参数通常是 `ExecutableDDLElement` 的实例，然后可以使用 `ExecutableDDLElement.compile()` 将其编译为字符串。

从版本 1.4 开始新增：- `create_mock_engine()` 函数取代了以前与 `create_engine()` 一起使用的“模拟”引擎策略。

另请参阅

如何将 CREATE TABLE/ DROP TABLE 输出作为字符串获取？

```py
function sqlalchemy.engine.make_url(name_or_url: str | URL) → URL
```

给定一个字符串，生成一个新的 URL 实例。

URL 的格式通常遵循 [RFC-1738](https://www.ietf.org/rfc/rfc1738.txt)，但也有一些例外情况，包括在“scheme”部分接受下划线而不是破折号或句点。

如果传递了 `URL` 对象，则返回原样。

另请参阅

数据库 URL

```py
function sqlalchemy.create_pool_from_url(url: str | URL, **kwargs: Any) → Pool
```

从给定的 URL 创建一个池实例。

如果未提供 `poolclass` 参数，则使用 URL 中指定的方言选择池类。

传递给 `create_pool_from_url()` 的参数与传递给 `create_engine()` 函数的池参数相同。

从版本 2.0.10 开始新增。

```py
class sqlalchemy.engine.URL
```

表示用于连接到数据库的 URL 的各个组件。

URL 通常是从完全格式化的 URL 字符串构建的，其中 `make_url()` 函数在内部由 `create_engine()` 函数使用，以将 URL 字符串解析为其各个组件，然后用于构造一个新的 `URL` 对象。在解析格式化的 URL 字符串时，解析格式通常遵循 [RFC-1738](https://www.ietf.org/rfc/rfc1738.txt)，但也有一些例外情况。

一个 `URL` 对象也可以直接生成，可以使用具有完整形式的 URL 字符串的 `make_url()` 函数，也可以使用 `URL.create()` 构造函数以编程方式给出单个字段来构造一个 `URL`。生成的 `URL` 对象可以直接传递给 `create_engine()` 以替换字符串参数的使用，这将在引擎的创建过程中绕过 `make_url()` 的使用。

从版本 1.4 开始更改：`URL` 对象现在是一个不可变对象。要创建 URL，请使用 `make_url()` 或 `URL.create()` 函数/方法。要修改 `URL`，请使用诸如 `URL.set()` 和 `URL.update_query_dict()` 等方法返回具有修改的新 `URL` 对象。有关此更改的说明，请参见 URL 对象现在是不可变的。

另请参见

数据库 URL

`URL` 包含以下属性：

+   `URL.drivername`: 数据库后端和驱动程序名称，例如`postgresql+psycopg2`

+   `URL.username`: 用户名字符串

+   `URL.password`: 密码字符串

+   `URL.host`: 主机名字符串

+   `URL.port`: 整数端口号

+   `URL.database`: 数据库名称字符串

+   `URL.query`: 一个表示查询字符串的不可变映射。包含键的字符串和值的字符串或字符串元组。

**成员**

create(), database, difference_update_query(), drivername, get_backend_name(), get_dialect(), get_driver_name(), host, normalized_query, password, port, query, render_as_string(), set(), translate_connect_args(), update_query_dict(), update_query_pairs(), update_query_string(), username

**类签名**

类`sqlalchemy.engine.URL` (`builtins.tuple`)

```py
classmethod create(drivername: str, username: str | None = None, password: str | None = None, host: str | None = None, port: int | None = None, database: str | None = None, query: Mapping[str, Sequence[str] | str] = {}) → URL
```

创建一个新的`URL`对象。

另请参阅

数据库 URL

参数：

+   `drivername` – 数据库后端的名称。该名称将对应于 sqlalchemy/databases 中的一个模块或第三方插件。

+   `username` – 用户名。

+   `password` –

    数据库密码。通常为字符串，但也可以是可以使用`str()`进行字符串化的对象。

    注意

    当作为参数传递给`URL.create()`时，密码字符串不应进行 URL 编码；该字符串应该包含与键入的密码字符完全相同。

    注意

    对于每个`Engine`对象，一个生成密码的对象只会被字符串化**一次**。对于每次连接的动态密码生成，请参阅生成动态认证令牌。

+   `host` – 主机名。

+   `port` – 端口号。

+   `database` – 数据库名称。

+   `query` – 一个字符串键到字符串值的字典，将在连接时传递给方言和/或 DBAPI。要直接指定 Python DBAPI 的非字符串参数，请使用`create_engine.connect_args`参数来调用`create_engine()`。另请参阅`URL.normalized_query`，用于一个一致的字符串->字符串列表字典。

返回：

新的`URL`对象。

从版本 1.4 开始：`URL`对象现在是一个**不可变的命名元组**。此外，`query`字典也是不可变的。要创建 URL，请使用`make_url()`或`URL.create()`函数/方法。要修改`URL`，请使用`URL.set()`和`URL.update_query()`方法。

```py
attribute database: str | None
```

数据库名称

```py
method difference_update_query(names: Iterable[str]) → URL
```

从`URL.query`字典中删除给定的名称，返回新的`URL`。

例如：

```py
url = url.difference_update_query(['foo', 'bar'])
```

等同于以下使用`URL.set()`：

```py
url = url.set(
    query={
        key: url.query[key]
        for key in set(url.query).difference(['foo', 'bar'])
    }
)
```

从版本 1.4 开始。

参见

`URL.query`

`URL.update_query_dict()`

`URL.set()`

```py
attribute drivername: str
```

数据库后端和驱动程序名称，如`postgresql+psycopg2`

```py
method get_backend_name() → str
```

返回后端名称。

这是与正在使用的数据库后端相对应的名称，并且是`URL.drivername`左侧的部分。

```py
method get_dialect(_is_async: bool = False) → Type[Dialect]
```

返回与此 URL 的驱动程序名称对应的 SQLAlchemy `Dialect`类。

```py
method get_driver_name() → str
```

返回后端名称。

这是与正在使用的 DBAPI 驱动程序相对应的名称，并且是`URL.drivername`右侧的部分。

如果`URL.drivername`中不包含加号，则导入此`URL`的默认`Dialect`以获取驱动程序名称。

```py
attribute host: str | None
```

主机名或 IP 地址。对于某些驱动程序，也可以是数据源名称。

```py
attribute normalized_query
```

将值标准化为序列后返回`URL.query`字典。

由于`URL.query`字典可能包含字符串值或字符串值序列以区分在查询字符串中多次指定的参数，因此需要通用处理多个参数的代码将希望使用此属性以便将所有出现的参数表示为序列。灵感来自 Python 的`urllib.parse.parse_qs`函数。例如：

```py
>>> from sqlalchemy.engine import make_url
>>> url = make_url("postgresql+psycopg2://user:pass@host/dbname?alt_host=host1&alt_host=host2&ssl_cipher=%2Fpath%2Fto%2Fcrt")
>>> url.query
immutabledict({'alt_host': ('host1', 'host2'), 'ssl_cipher': '/path/to/crt'})
>>> url.normalized_query
immutabledict({'alt_host': ('host1', 'host2'), 'ssl_cipher': ('/path/to/crt',)})
```

```py
attribute password: str | None
```

密码，通常是一个字符串，但也可能是具有`__str__()`方法的任何对象。

```py
attribute port: int | None
```

整数端口号

```py
attribute query: immutabledict[str, Tuple[str, ...] | str]
```

代表查询字符串的不可变映射。包含键的字符串和值的字符串或字符串元组，例如：

```py
 >>> from sqlalchemy.engine import make_url
   >>> url = make_url("postgresql+psycopg2://user:pass@host/dbname?alt_host=host1&alt_host=host2&ssl_cipher=%2Fpath%2Fto%2Fcrt")
   >>> url.query
   immutabledict({'alt_host': ('host1', 'host2'), 'ssl_cipher': '/path/to/crt'})

To create a mutable copy of this mapping, use the ``dict`` constructor::

   mutable_query_opts = dict(url.query)
```

另请参阅

`URL.normalized_query` - 将所有值标准化为序列以进行一致处理

更改`URL.query`内容的方法：

`URL.update_query_dict()`

`URL.update_query_string()`

`URL.update_query_pairs()`

`URL.difference_update_query()`

```py
method render_as_string(hide_password: bool = True) → str
```

将此`URL`对象呈现为字符串。

当使用`__str__()`或`__repr__()`方法时使用此方法。该方法直接包括附加选项。

参数：

**hide_password** – 默认为 True。除非设置为 False，否则密码不会显示在字符串中。

```py
method set(drivername: str | None = None, username: str | None = None, password: str | None = None, host: str | None = None, port: int | None = None, database: str | None = None, query: Mapping[str, Sequence[str] | str] | None = None) → URL
```

返回一个带有修改的`URL`对象。

如果值为非 None，则使用该值。要将值显式设置为`None`，请使用从`namedtuple`调整的`URL._replace()`方法。

参数：

+   `drivername` – 新的驱动程序名称

+   `username` – 新用户名

+   `password` – 新密码

+   `host` – 新主机名

+   `port` – 新端口

+   `query` – 新的查询参数，传递一个字符串键的字典，指向字符串或字符串值序列。完全替换了以前的参数列表。

返回：

新的`URL`对象。

版本 1.4 中的新功能。

另请参阅

`URL.update_query_dict()`

```py
method translate_connect_args(names: List[str] | None = None, **kw: Any) → Dict[str, Any]
```

将 url 属性翻译为连接参数的字典。

将此 url 的属性（主机、数据库、用户名、密码、端口）作为普通字典返回。默认情况下，属性名称用作键。未设置或为 false 的属性将从最终字典中省略。

参数：

+   `**kw` – 可选，url 属性的替代键名。

+   `names` – 已弃用。与基于关键字的替代名称具有相同目的，但将名称与原始位置相关联。

```py
method update_query_dict(query_parameters: Mapping[str, str | List[str]], append: bool = False) → URL
```

返回一个由给定字典更新的`URL`对象的新对象。

字典通常包含字符串键和字符串值。为了表示表达多次的查询参数，传递一个字符串值序列。

例如：

```py
>>> from sqlalchemy.engine import make_url
>>> url = make_url("postgresql+psycopg2://user:pass@host/dbname")
>>> url = url.update_query_dict({"alt_host": ["host1", "host2"], "ssl_cipher": "/path/to/crt"})
>>> str(url)
'postgresql+psycopg2://user:pass@host/dbname?alt_host=host1&alt_host=host2&ssl_cipher=%2Fpath%2Fto%2Fcrt'
```

参数：

+   `query_parameters` – 具有字符串键和值的字典，值为字符串或字符串序列。

+   `append` – 如果为 True，则现有查询字符串中的参数不会被删除；新参数将添加到已有参数之后。如果保持其默认值 False，则给定查询参数中的键将替换现有查询字符串中的键。

1.4 版本中的新内容。

另请参阅

`URL.query`

`URL.update_query_string()`

`URL.update_query_pairs()`

`URL.difference_update_query()`

`URL.set()`

```py
method update_query_pairs(key_value_pairs: Iterable[Tuple[str, str | List[str]]], append: bool = False) → URL
```

返回一个新的 `URL` 对象，其参数字典由给定的键/值对序列更新。

例如：

```py
>>> from sqlalchemy.engine import make_url
>>> url = make_url("postgresql+psycopg2://user:pass@host/dbname")
>>> url = url.update_query_pairs([("alt_host", "host1"), ("alt_host", "host2"), ("ssl_cipher", "/path/to/crt")])
>>> str(url)
'postgresql+psycopg2://user:pass@host/dbname?alt_host=host1&alt_host=host2&ssl_cipher=%2Fpath%2Fto%2Fcrt'
```

参数：

+   `key_value_pairs` – 包含两个字符串的元组序列。

+   `append` – 如果为 True，则现有查询字符串中的参数不会被删除；新参数将添加到已有参数之后。如果保持其默认值 False，则给定查询参数中的键将替换现有查询字符串中的键。

1.4 版本中的新内容。

另请参阅

`URL.query`

`URL.difference_update_query()`

`URL.set()`

```py
method update_query_string(query_string: str, append: bool = False) → URL
```

返回一个新的 `URL` 对象，其参数字典由给定的查询字符串更新。

例如：

```py
>>> from sqlalchemy.engine import make_url
>>> url = make_url("postgresql+psycopg2://user:pass@host/dbname")
>>> url = url.update_query_string("alt_host=host1&alt_host=host2&ssl_cipher=%2Fpath%2Fto%2Fcrt")
>>> str(url)
'postgresql+psycopg2://user:pass@host/dbname?alt_host=host1&alt_host=host2&ssl_cipher=%2Fpath%2Fto%2Fcrt'
```

参数：

+   `query_string` – 一个 URL 转义的查询字符串，不包括问号。

+   `append` – 如果为 True，则现有查询字符串中的参数不会被删除；新参数将添加到已有参数之后。如果保持其默认值 False，则给定查询参数中的键将替换现有查询字符串中的键。

1.4 版本中的新内容。

另请参阅

`URL.query`

`URL.update_query_dict()`

```py
attribute username: str | None
```

用户名 字符串

## 池化

当调用 `connect()` 或 `execute()` 方法时，`Engine` 将向连接池请求连接。默认连接池 `QueuePool` 将根据需要打开与数据库的连接。随着并发语句的执行，`QueuePool` 将增加其连接池的大小至默认大小为五，并允许默认的“溢出”为十。由于 `Engine` 本质上是连接池的“基地”，因此在应用程序中，应该保持单个 `Engine` 与数据库建立连接，而不是为每个连接创建新的。

注意

默认情况下，SQLite 引擎不使用 `QueuePool`。有关 SQLite 连接池使用的详细信息，请参阅 SQLite。

欲了解更多有关连接池的信息，请参阅连接池。

## 自定义 DBAPI connect() 参数 / 连接时例程

对于需要特殊连接方法的情况，在绝大多数情况下，最适合的方法是使用 `create_engine()` 层级中的一些钩子来自定义此过程。这些在以下子章节中描述。

### 传递给 `dbapi.connect()` 的特殊关键字参数

所有 Python DBAPI 都接受除连接基础之外的附加参数。常见的参数包括用于指定字符集编码和超时值的参数；更复杂的数据包括特殊的 DBAPI 常量和对象以及 SSL 子参数。有两种基本的方式可以在不复杂的情况下传递这些参数。

#### 将参数添加到 URL 查询字符串

简单的字符串值，以及一些数值和布尔标志，可以直接在 URL 的查询字符串中指定。其中一个常见的例子是接受字符编码参数 `encoding` 的 DBAPI，例如大多数 MySQL DBAPI：

```py
engine = create_engine("mysql+pymysql://user:pass@host/test?charset=utf8mb4")
```

使用查询字符串的优点是可以在配置文件中指定附加的 DBAPI 选项，以便与 URL 中指定的 DBAPI 兼容。在这个级别传递的具体参数因 SQLAlchemy 方言而异。一些方言将所有参数都作为字符串传递，而另一些方言将解析特定的数据类型并将参数移到不同的位置，比如驱动程序级别的 DSN 和连接字符串。由于此领域的每个方言的行为当前存在差异，因此应该查阅用于特定方言的方言文档，以查看在此级别是否支持特定参数。

提示

使用 `Dialect.create_connect_args()` 方法可以执行以下操作，显示给定 URL 的 DBAPI 传递的确切参数：

```py
>>> from sqlalchemy import create_engine
>>> engine = create_engine(
...     "mysql+pymysql://some_user:some_pass@some_host/test?charset=utf8mb4"
... )
>>> args, kwargs = engine.dialect.create_connect_args(engine.url)
>>> args, kwargs
([], {'host': 'some_host', 'database': 'test', 'user': 'some_user', 'password': 'some_pass', 'charset': 'utf8mb4', 'client_flag': 2})
```

上述 `args, kwargs` 对通常作为 `dbapi.connect(*args, **kwargs)` 传递给 DBAPI。

#### 使用 `connect_args` 字典参数

一个更一般的系统，用于将任何参数传递给 `dbapi.connect()` 函数，保证始终传递所有参数，是 `create_engine.connect_args` 字典参数。这可用于否则未在查询字符串中处理的参数，以及在必须将特殊子结构或对象传递给 DBAPI 时。有时，只需将特定标志作为 `True` 符号发送，而 SQLAlchemy 方言并不知道如何从 URL 中的字符串形式强制转换此关键字参数。下面示例说明了使用 psycopg2 “连接工厂” 替换连接的基础实现：

```py
engine = create_engine(
    "postgresql+psycopg2://user:pass@hostname/dbname",
    connect_args={"connection_factory": MyConnectionFactory},
)
```

另一个示例是 pyodbc 的 “timeout” 参数：

```py
engine = create_engine(
    "mssql+pyodbc://user:pass@sqlsrvr?driver=ODBC+Driver+13+for+SQL+Server",
    connect_args={"timeout": 30},
)
```

上述示例还说明了 URL “查询字符串”参数以及 `create_engine.connect_args` 可以同时使用；在 pyodbc 的情况下，“driver” 关键字在 URL 中具有特殊含义。

### 控制参数如何传递给 DBAPI 的 `connect()` 函数

除了操作传递给 `connect()` 的参数外，我们还可以使用 `DialectEvents.do_connect()` 事件挂钩进一步定制 DBAPI `connect()` 函数本身的调用方式。此挂钩传递了方言将发送给 `connect()` 的完整 `*args, **kwargs`。然后，这些集合可以就地修改以更改它们的使用方式：

```py
from sqlalchemy import event

engine = create_engine("postgresql+psycopg2://user:pass@hostname/dbname")

@event.listens_for(engine, "do_connect")
def receive_do_connect(dialect, conn_rec, cargs, cparams):
    cparams["connection_factory"] = MyConnectionFactory
```

#### 生成动态认证令牌

`DialectEvents.do_connect()` 也是动态插入认证令牌的理想方式，该令牌可能在 `Engine` 的生命周期内发生变化。例如，如果令牌由 `get_authentication_token()` 生成并在 `token` 参数中传递给 DBAPI，则可以实现如下：

```py
from sqlalchemy import event

engine = create_engine("postgresql+psycopg2://user@hostname/dbname")

@event.listens_for(engine, "do_connect")
def provide_token(dialect, conn_rec, cargs, cparams):
    cparams["token"] = get_authentication_token()
```

另请参阅

使用访问令牌连接数据库 - 涉及 SQL Server 的更具体示例

### 修改连接后的 DBAPI 连接，或在连接后运行命令

对于 SQLAlchemy 创建的 DBAPI 连接，如果我们想在实际使用之前修改已完成的连接，比如设置特殊标志或运行某些命令，`PoolEvents.connect()`事件钩子是最合适的钩子。这个钩子在每个新连接创建时调用，然后被 SQLAlchemy 使用：

```py
from sqlalchemy import event

engine = create_engine("postgresql+psycopg2://user:pass@hostname/dbname")

@event.listens_for(engine, "connect")
def connect(dbapi_connection, connection_record):
    cursor_obj = dbapi_connection.cursor()
    cursor_obj.execute("SET some session variables")
    cursor_obj.close()
```

### 完全替换 DBAPI 的`connect()`函数

最后，`DialectEvents.do_connect()`事件钩子也可以让我们完全接管连接过程，建立连接并返回它：

```py
from sqlalchemy import event

engine = create_engine("postgresql+psycopg2://user:pass@hostname/dbname")

@event.listens_for(engine, "do_connect")
def receive_do_connect(dialect, conn_rec, cargs, cparams):
    # return the new DBAPI connection with whatever we'd like to
    # do
    return psycopg2.connect(*cargs, **cparams)
```

`DialectEvents.do_connect()`钩子取代了以前的`create_engine.creator`钩子，该钩子仍然可用。`DialectEvents.do_connect()`具有一个明显的优势，即从 URL 解析的完整参数也传递给用户定义的函数，而这在`create_engine.creator`中并非如此。

### 传递给`dbapi.connect()`的特殊关键字参数

所有 Python DBAPI 都接受除了基本连接之外的额外参数。常见参数包括用于指定字符集编码和超时值的参数；更复杂的数据包括特殊的 DBAPI 常量和对象以及 SSL 子参数。有两种基本的方法可以传递这些参数而不增加复杂性。

#### 将参数添加到 URL 查询字符串中

简单的字符串值，以及一些数值和布尔标志，通常可以直接在 URL 的查询字符串中指定。一个常见的例子是接受`encoding`参数用于字符编码的 DBAPIs，比如大多数 MySQL DBAPIs：

```py
engine = create_engine("mysql+pymysql://user:pass@host/test?charset=utf8mb4")
```

使用查询字符串的优势在于可以在配置文件中指定额外的 DBAPI 选项，以一种对 URL 中指定的 DBAPI 可移植的方式。在这个级别传递的具体参数因 SQLAlchemy 方言而异。一些方言将所有参数都作为字符串传递，而其他方言将解析特定数据类型并将参数移动到不同位置，比如驱动程序级别的 DSN 和连接字符串。由于当前在这个领域的方言行为各不相同，应该查阅特定方言的文档，以查看在这个级别是否支持特定参数。

提示

一种显示给定 URL 的 DBAPI 所传递的精确参数的一般技术可以直接使用 `Dialect.create_connect_args()` 方法，如下所示：

```py
>>> from sqlalchemy import create_engine
>>> engine = create_engine(
...     "mysql+pymysql://some_user:some_pass@some_host/test?charset=utf8mb4"
... )
>>> args, kwargs = engine.dialect.create_connect_args(engine.url)
>>> args, kwargs
([], {'host': 'some_host', 'database': 'test', 'user': 'some_user', 'password': 'some_pass', 'charset': 'utf8mb4', 'client_flag': 2})
```

上述 `args, kwargs` 对通常作为 `dbapi.connect(*args, **kwargs)` 传递给 DBAPI。

#### 使用 `connect_args` 字典参数

一种更通用的系统，用于向 `dbapi.connect()` 函数传递任何参数，并保证始终传递所有参数，是 `create_engine.connect_args` 字典参数。这可用于那些否则不被方言处理的参数，当它们添加到查询字符串时，以及当必须将特殊子结构或对象传递给 DBAPI 时。有时，只需将特定标志发送为 `True` 符号，而 SQLAlchemy 方言不知道此关键字参数以将其从其 URL 中呈现的字符串形式强制转换。下面说明了使用 psycopg2 “连接工厂” 的示例，用它替换了连接的底层实现：

```py
engine = create_engine(
    "postgresql+psycopg2://user:pass@hostname/dbname",
    connect_args={"connection_factory": MyConnectionFactory},
)
```

另一个示例是 pyodbc 的 “timeout” 参数：

```py
engine = create_engine(
    "mssql+pyodbc://user:pass@sqlsrvr?driver=ODBC+Driver+13+for+SQL+Server",
    connect_args={"timeout": 30},
)
```

上面的示例还说明了 URL “查询字符串”参数以及 `create_engine.connect_args` 可以同时使用；在 pyodbc 的情况下，“driver” 关键字在 URL 中具有特殊意义。

#### 向 URL 查询字符串添加参数

简单的字符串值，以及一些数值和布尔标志，通常可以直接在 URL 的查询字符串中指定。这种情况的常见例子是接受字符编码参数 `encoding` 的 DBAPI，例如大多数 MySQL DBAPI：

```py
engine = create_engine("mysql+pymysql://user:pass@host/test?charset=utf8mb4")
```

使用查询字符串的优点是可以在配置文件中指定其他 DBAPI 选项，这种方式对于在 URL 中指定的 DBAPI 是可移植的。通过此级别传递的特定参数因 SQLAlchemy 方言而异。一些方言将所有参数作为字符串传递，而其他方言将解析特定的数据类型并将参数移到不同的位置，例如到驱动程序级别的 DSN 和连接字符串。由于此领域的方言行为目前有所变化，因此应查阅使用的特定方言的方言文档，以查看在此级别是否支持特定参数。

提示

一种显示给定 URL 的 DBAPI 所传递的精确参数的一般技术可以直接使用 `Dialect.create_connect_args()` 方法，如下所示：

```py
>>> from sqlalchemy import create_engine
>>> engine = create_engine(
...     "mysql+pymysql://some_user:some_pass@some_host/test?charset=utf8mb4"
... )
>>> args, kwargs = engine.dialect.create_connect_args(engine.url)
>>> args, kwargs
([], {'host': 'some_host', 'database': 'test', 'user': 'some_user', 'password': 'some_pass', 'charset': 'utf8mb4', 'client_flag': 2})
```

上述 `args, kwargs` 对通常作为 `dbapi.connect(*args, **kwargs)` 传递给 DBAPI。

#### 使用 `connect_args` 字典参数

将任何参数传递给`dbapi.connect()`函数的更通用的系统，保证始终传递所有参数的是`create_engine.connect_args`字典参数。这可以用于否则不会由方言处理的添加到查询字符串的参数，以及当必须将特殊子结构或对象传递给 DBAPI 时。有时只是一个特定标志必须作为`True`符号发送，并且 SQLAlchemy 方言不知道此关键字参数以将其从 URL 中呈现的字符串形式强制转换为其。下面说明了使用 psycopg2 的“connection factory”替换连接的底层实现的示例：

```py
engine = create_engine(
    "postgresql+psycopg2://user:pass@hostname/dbname",
    connect_args={"connection_factory": MyConnectionFactory},
)
```

另一个例子是 pyodbc 的“timeout”参数：

```py
engine = create_engine(
    "mssql+pyodbc://user:pass@sqlsrvr?driver=ODBC+Driver+13+for+SQL+Server",
    connect_args={"timeout": 30},
)
```

上述示例还说明了 URL“查询字符串”参数以及`create_engine.connect_args`都可以同时使用；在 pyodbc 的情况下，“driver”关键字在 URL 中具有特殊含义。

### 控制参数传递到 DBAPI connect() 函数的方式

除了操纵传递给 `connect()` 的参数之外，我们还可以通过`DialectEvents.do_connect()`事件钩子进一步自定义 DBAPI `connect()` 函数本身的调用方式。此钩子传递了方言将发送给 `connect()` 的完整的 `*args, **kwargs`。然后可以修改这些集合以更改它们的使用方式：

```py
from sqlalchemy import event

engine = create_engine("postgresql+psycopg2://user:pass@hostname/dbname")

@event.listens_for(engine, "do_connect")
def receive_do_connect(dialect, conn_rec, cargs, cparams):
    cparams["connection_factory"] = MyConnectionFactory
```

#### 生成动态认证令牌

`DialectEvents.do_connect()`也是一种理想的方法，可以动态插入可能会随着`Engine`寿命而变化的认证令牌。例如，如果令牌由`get_authentication_token()`生成并在`token`参数中传递给 DBAPI，则可以实现为：

```py
from sqlalchemy import event

engine = create_engine("postgresql+psycopg2://user@hostname/dbname")

@event.listens_for(engine, "do_connect")
def provide_token(dialect, conn_rec, cargs, cparams):
    cparams["token"] = get_authentication_token()
```

请参阅

使用访问令牌连接数据库 - 一个更具体的例子涉及 SQL Server  #### 生成动态认证令牌

`DialectEvents.do_connect()`也是一种理想的方法，可以动态插入可能会随着`Engine`寿命而变化的认证令牌。例如，如果令牌由`get_authentication_token()`生成并在`token`参数中传递给 DBAPI，则可以实现为：

```py
from sqlalchemy import event

engine = create_engine("postgresql+psycopg2://user@hostname/dbname")

@event.listens_for(engine, "do_connect")
def provide_token(dialect, conn_rec, cargs, cparams):
    cparams["token"] = get_authentication_token()
```

请参阅

使用访问令牌连接数据库 - 一个更具体的涉及 SQL Server 的例子

### 在连接后修改 DBAPI 连接，或在连接后运行命令

对于 SQLAlchemy 创建的 DBAPI 连接，虽然没有问题，但我们想要在实际使用之前修改完成的连接的情况，例如设置特殊标志或运行某些命令，`PoolEvents.connect()`事件钩子是最合适的钩子。这个钩子在每次创建新连接时调用，然后由 SQLAlchemy 使用：

```py
from sqlalchemy import event

engine = create_engine("postgresql+psycopg2://user:pass@hostname/dbname")

@event.listens_for(engine, "connect")
def connect(dbapi_connection, connection_record):
    cursor_obj = dbapi_connection.cursor()
    cursor_obj.execute("SET some session variables")
    cursor_obj.close()
```

### 完全替换 DBAPI 的`connect()`函数

最后，`DialectEvents.do_connect()`事件钩子也可以允许我们完全接管连接过程，建立连接并返回它：

```py
from sqlalchemy import event

engine = create_engine("postgresql+psycopg2://user:pass@hostname/dbname")

@event.listens_for(engine, "do_connect")
def receive_do_connect(dialect, conn_rec, cargs, cparams):
    # return the new DBAPI connection with whatever we'd like to
    # do
    return psycopg2.connect(*cargs, **cparams)
```

`DialectEvents.do_connect()`钩子取代了以前的`create_engine.creator`钩子，后者仍然可用。`DialectEvents.do_connect()`的明显优势是解析自 URL 的完整参数也传递给用户定义的函数，而对于`create_engine.creator`不是这样。

## 配置日志记录

Python 的标准[logging](https://docs.python.org/library/logging.html)模块用于实现 SQLAlchemy 的信息和调试日志输出。这使得 SQLAlchemy 的日志记录可以以标准方式与其他应用程序和库集成。`create_engine()`上还有两个参数`create_engine.echo`和`create_engine.echo_pool`，它们允许立即将日志记录到`sys.stdout`以进行本地开发；这些参数最终与下面描述的常规 Python 记录器进行交互。

这一节假设您熟悉上面链接的日志记录模块。SQLAlchemy 执行的所有日志记录都存在于`sqlalchemy`命名空间下，就像`logging.getLogger('sqlalchemy')`所使用的那样。当日志记录已经配置好（例如通过`logging.basicConfig()`），可以打开的 SA 日志记录器的一般命名空间如下所示：

+   `sqlalchemy.engine` - 控制 SQL 回显。设置为`logging.INFO`以输出 SQL 查询结果，设置为`logging.DEBUG`以输出查询结果集。这些设置相当于`create_engine.echo` 上的`echo=True`和`echo="debug"`。

+   `sqlalchemy.pool` - 控制连接池日志记录。设置为`logging.INFO`以记录连接失效和回收事件；设置为`logging.DEBUG`以额外记录所有连接池的签入和签出。这些设置相当于`create_engine.echo_pool` 上的`pool_echo=True`和`pool_echo="debug"`。

+   `sqlalchemy.dialects` - 控制 SQL 方言的自定义日志记录，以特定方言内部使用日志的程度为准，通常很少。

+   `sqlalchemy.orm` - 控制各种 ORM 函数的日志记录，以 ORM 内部使用日志的程度为准，通常很少。设置为`logging.INFO`以记录一些有关映射器配置的顶级信息。

例如，要使用 Python 日志记录而不是`echo=True`标志记录 SQL 查询：

```py
import logging

logging.basicConfig()
logging.getLogger("sqlalchemy.engine").setLevel(logging.INFO)
```

默认情况下，整个 `sqlalchemy` 命名空间的日志级别设置为`logging.WARN`，因此即使在已启用日志记录的应用程序中也不会发生任何日志操作。

注意

SQLAlchemy `Engine` 通过仅在检测到当前日志级别为`logging.INFO`或`logging.DEBUG`时发出日志语句来节省 Python 函数调用开销。仅在从连接池获取新连接时才检查此级别。因此，在更改已运行应用程序的日志配置时，任何当前活动的 `Connection`（更常见的是处于事务中的活动 `Session` 对象）都不会根据新配置记录任何 SQL，直到获取新的 `Connection`（在 `Session` 的情况下，这是在当前事务结束并开始新事务之后）。

### 更多关于 Echo 标志

如前所述，`create_engine.echo` 和 `create_engine.echo_pool` 参数是直接将日志记录到 `sys.stdout` 的快捷方式：

```py
>>> from sqlalchemy import create_engine, text
>>> e = create_engine("sqlite://", echo=True, echo_pool="debug")
>>> with e.connect() as conn:
...     print(conn.scalar(text("select 'hi'")))
2020-10-24 12:54:57,701 DEBUG sqlalchemy.pool.impl.SingletonThreadPool Created new connection <sqlite3.Connection object at 0x7f287819ac60>
2020-10-24 12:54:57,701 DEBUG sqlalchemy.pool.impl.SingletonThreadPool Connection <sqlite3.Connection object at 0x7f287819ac60> checked out from pool
2020-10-24 12:54:57,702 INFO sqlalchemy.engine.Engine select 'hi'
2020-10-24 12:54:57,702 INFO sqlalchemy.engine.Engine ()
hi
2020-10-24 12:54:57,703 DEBUG sqlalchemy.pool.impl.SingletonThreadPool Connection <sqlite3.Connection object at 0x7f287819ac60> being returned to pool
2020-10-24 12:54:57,704 DEBUG sqlalchemy.pool.impl.SingletonThreadPool Connection <sqlite3.Connection object at 0x7f287819ac60> rollback-on-return
```

使用这些标志大致相当于：

```py
import logging

logging.basicConfig()
logging.getLogger("sqlalchemy.engine").setLevel(logging.INFO)
logging.getLogger("sqlalchemy.pool").setLevel(logging.DEBUG)
```

需要注意的是，这两个标志**独立**于任何现有的日志配置工作，并且将无条件地使用 `logging.basicConfig()`。这将在**除了**任何现有记录器配置之外**额外**配置。因此，在显式配置日志时，请确保始终将所有回显标志设置为 False，以避免获取重复的日志行。

### 设置日志名称

诸如 `Engine` 或 `Pool` 的实例的记录器名称默认为使用截断的十六进制标识符字符串。要将其设置为特定名称，请使用 `create_engine.logging_name` 和 `create_engine.pool_logging_name` 与 `sqlalchemy.create_engine()`；名称将附加到日志名称 `sqlalchemy.engine.Engine`：

```py
>>> import logging
>>> from sqlalchemy import create_engine
>>> from sqlalchemy import text
>>> logging.basicConfig()
>>> logging.getLogger("sqlalchemy.engine.Engine.myengine").setLevel(logging.INFO)
>>> e = create_engine("sqlite://", logging_name="myengine")
>>> with e.connect() as conn:
...     conn.execute(text("select 'hi'"))
2020-10-24 12:47:04,291 INFO sqlalchemy.engine.Engine.myengine select 'hi'
2020-10-24 12:47:04,292 INFO sqlalchemy.engine.Engine.myengine ()
```

提示

`create_engine.logging_name` 和 `create_engine.pool_logging_name` 参数也可与 `create_engine.echo` 和 `create_engine.echo_pool` 结合使用。然而，如果其他引擎创建时将回显标志设置为 True 而没有日志名称，则将会发生无法避免的双重记录条件。这是因为将自动为 `sqlalchemy.engine.Engine` 添加处理程序，它将同时记录无名称的引擎和具有日志名称的引擎的消息。例如：

```py
from sqlalchemy import create_engine, text

e1 = create_engine("sqlite://", echo=True, logging_name="myname")
with e1.begin() as conn:
    conn.execute(text("SELECT 1"))

e2 = create_engine("sqlite://", echo=True)
with e2.begin() as conn:
    conn.execute(text("SELECT 2"))

with e1.begin() as conn:
    conn.execute(text("SELECT 3"))
```

上述情况将会双重记录 `SELECT 3`。为解决此问题，请确保所有引擎都设置了 `logging_name`，或者使用显式的记录器/处理器设置，而不使用`create_engine.echo` 和 `create_engine.echo_pool`。

### 设置每个连接/子引擎令牌

自 1.4.0b2 版本新增。

虽然在长期存在的`Engine`对象上建立日志名称是适当的，但它不够灵活，无法容纳任意长的名称列表，以跟踪日志消息中的单个连接和/或事务的情况。

对于此用例，由 `Connection` 和 `Result` 对象生成的日志消息本身可以通过附加诸如事务或请求标识符之类的其他标记来增强。 `Connection.execution_options.logging_token` 参数接受一个字符串参数，该参数可用于建立每个连接的跟踪标记：

```py
>>> from sqlalchemy import create_engine
>>> e = create_engine("sqlite://", echo="debug")
>>> with e.connect().execution_options(logging_token="track1") as conn:
...     conn.execute(text("select 1")).all()
2021-02-03 11:48:45,754 INFO sqlalchemy.engine.Engine [track1] select 1
2021-02-03 11:48:45,754 INFO sqlalchemy.engine.Engine [track1] [raw sql] ()
2021-02-03 11:48:45,754 DEBUG sqlalchemy.engine.Engine [track1] Col ('1',)
2021-02-03 11:48:45,755 DEBUG sqlalchemy.engine.Engine [track1] Row (1,)
```

`Connection.execution_options.logging_token` 参数也可以通过 `create_engine.execution_options` 或 `Engine.execution_options()` 在引擎或子引擎上建立。这可能对将不同的日志标记应用于应用程序的不同组件而不创建新引擎非常有用：

```py
>>> from sqlalchemy import create_engine
>>> e = create_engine("sqlite://", echo="debug")
>>> e1 = e.execution_options(logging_token="track1")
>>> e2 = e.execution_options(logging_token="track2")
>>> with e1.connect() as conn:
...     conn.execute(text("select 1")).all()
2021-02-03 11:51:08,960 INFO sqlalchemy.engine.Engine [track1] select 1
2021-02-03 11:51:08,960 INFO sqlalchemy.engine.Engine [track1] [raw sql] ()
2021-02-03 11:51:08,960 DEBUG sqlalchemy.engine.Engine [track1] Col ('1',)
2021-02-03 11:51:08,961 DEBUG sqlalchemy.engine.Engine [track1] Row (1,)

>>> with e2.connect() as conn:
...     conn.execute(text("select 2")).all()
2021-02-03 11:52:05,518 INFO sqlalchemy.engine.Engine [track2] Select 1
2021-02-03 11:52:05,519 INFO sqlalchemy.engine.Engine [track2] [raw sql] ()
2021-02-03 11:52:05,520 DEBUG sqlalchemy.engine.Engine [track2] Col ('1',)
2021-02-03 11:52:05,520 DEBUG sqlalchemy.engine.Engine [track2] Row (1,)
```

### 隐藏参数

`Engine` 发出的日志还显示了特定语句中存在的 SQL 参数摘录。为了防止出于隐私目的记录这些参数，请启用 `create_engine.hide_parameters` 标志：

```py
>>> e = create_engine("sqlite://", echo=True, hide_parameters=True)
>>> with e.connect() as conn:
...     conn.execute(text("select :some_private_name"), {"some_private_name": "pii"})
2020-10-24 12:48:32,808 INFO sqlalchemy.engine.Engine select ?
2020-10-24 12:48:32,808 INFO sqlalchemy.engine.Engine [SQL parameters hidden due to hide_parameters=True]
```

### 回声标记的更多信息

正如前面提到的，`create_engine.echo` 和 `create_engine.echo_pool` 参数是立即记录到 `sys.stdout` 的快捷方式：

```py
>>> from sqlalchemy import create_engine, text
>>> e = create_engine("sqlite://", echo=True, echo_pool="debug")
>>> with e.connect() as conn:
...     print(conn.scalar(text("select 'hi'")))
2020-10-24 12:54:57,701 DEBUG sqlalchemy.pool.impl.SingletonThreadPool Created new connection <sqlite3.Connection object at 0x7f287819ac60>
2020-10-24 12:54:57,701 DEBUG sqlalchemy.pool.impl.SingletonThreadPool Connection <sqlite3.Connection object at 0x7f287819ac60> checked out from pool
2020-10-24 12:54:57,702 INFO sqlalchemy.engine.Engine select 'hi'
2020-10-24 12:54:57,702 INFO sqlalchemy.engine.Engine ()
hi
2020-10-24 12:54:57,703 DEBUG sqlalchemy.pool.impl.SingletonThreadPool Connection <sqlite3.Connection object at 0x7f287819ac60> being returned to pool
2020-10-24 12:54:57,704 DEBUG sqlalchemy.pool.impl.SingletonThreadPool Connection <sqlite3.Connection object at 0x7f287819ac60> rollback-on-return
```

使用这些标志大致相当于：

```py
import logging

logging.basicConfig()
logging.getLogger("sqlalchemy.engine").setLevel(logging.INFO)
logging.getLogger("sqlalchemy.pool").setLevel(logging.DEBUG)
```

需要注意的是，这两个标志**独立**于任何现有的日志配置工作，并且无条件使用 `logging.basicConfig()`。这会使其效果**叠加**到任何现有的记录器配置中。因此，**在明确配置日志时，请始终确保所有回声标志都设置为 False**，以避免获得重复的日志行。

### 设置日志名称

实例的记录器名称（如`Engine`或`Pool`）默认为使用截断的十六进制标识符字符串。要将其设置为特定名称，请使用`create_engine.logging_name`和`create_engine.pool_logging_name`与`sqlalchemy.create_engine()`；该名称将附加到日志名称`sqlalchemy.engine.Engine`：

```py
>>> import logging
>>> from sqlalchemy import create_engine
>>> from sqlalchemy import text
>>> logging.basicConfig()
>>> logging.getLogger("sqlalchemy.engine.Engine.myengine").setLevel(logging.INFO)
>>> e = create_engine("sqlite://", logging_name="myengine")
>>> with e.connect() as conn:
...     conn.execute(text("select 'hi'"))
2020-10-24 12:47:04,291 INFO sqlalchemy.engine.Engine.myengine select 'hi'
2020-10-24 12:47:04,292 INFO sqlalchemy.engine.Engine.myengine ()
```

小贴士

`create_engine.logging_name`和`create_engine.pool_logging_name`参数也可以与`create_engine.echo`和`create_engine.echo_pool`一起使用。但是，如果其他引擎设置了 echo 标志为 True 而没有设置日志名称，则将会发生不可避免的双重记录条件。这是因为将自动为`sqlalchemy.engine.Engine`添加处理程序，该处理程序将同时为无名称的引擎和具有日志名称的引擎记录消息。例如：

```py
from sqlalchemy import create_engine, text

e1 = create_engine("sqlite://", echo=True, logging_name="myname")
with e1.begin() as conn:
    conn.execute(text("SELECT 1"))

e2 = create_engine("sqlite://", echo=True)
with e2.begin() as conn:
    conn.execute(text("SELECT 2"))

with e1.begin() as conn:
    conn.execute(text("SELECT 3"))
```

上述情景将对`SELECT 3`进行双重记录。为了解决此问题，请确保所有引擎都设置了`logging_name`，或者在不使用`create_engine.echo`和`create_engine.echo_pool`的情况下，使用显式的记录器/处理程序设置。

### 设置每个连接/子引擎令牌

1.4.0b2 版本中的新功能。

虽然在长期存在的`Engine`对象上建立记录名称是合适的，但是对于跟踪日志消息中的单个连接和/或事务的情况，它的灵活性不够。 

对于这种用例，由`Connection`和`Result`对象生成的日志消息本身可能会使用附加令牌（例如事务或请求标识符）进行扩充。`Connection.execution_options.logging_token`参数接受一个字符串参数，该参数可用于建立每个连接的跟踪令牌：

```py
>>> from sqlalchemy import create_engine
>>> e = create_engine("sqlite://", echo="debug")
>>> with e.connect().execution_options(logging_token="track1") as conn:
...     conn.execute(text("select 1")).all()
2021-02-03 11:48:45,754 INFO sqlalchemy.engine.Engine [track1] select 1
2021-02-03 11:48:45,754 INFO sqlalchemy.engine.Engine [track1] [raw sql] ()
2021-02-03 11:48:45,754 DEBUG sqlalchemy.engine.Engine [track1] Col ('1',)
2021-02-03 11:48:45,755 DEBUG sqlalchemy.engine.Engine [track1] Row (1,)
```

`Connection.execution_options.logging_token` 参数也可以通过 `create_engine.execution_options` 或 `Engine.execution_options()` 在引擎或子引擎上建立。这可能对于在不创建新引擎的情况下将不同的日志令牌应用于应用程序的不同组件很有用：

```py
>>> from sqlalchemy import create_engine
>>> e = create_engine("sqlite://", echo="debug")
>>> e1 = e.execution_options(logging_token="track1")
>>> e2 = e.execution_options(logging_token="track2")
>>> with e1.connect() as conn:
...     conn.execute(text("select 1")).all()
2021-02-03 11:51:08,960 INFO sqlalchemy.engine.Engine [track1] select 1
2021-02-03 11:51:08,960 INFO sqlalchemy.engine.Engine [track1] [raw sql] ()
2021-02-03 11:51:08,960 DEBUG sqlalchemy.engine.Engine [track1] Col ('1',)
2021-02-03 11:51:08,961 DEBUG sqlalchemy.engine.Engine [track1] Row (1,)

>>> with e2.connect() as conn:
...     conn.execute(text("select 2")).all()
2021-02-03 11:52:05,518 INFO sqlalchemy.engine.Engine [track2] Select 1
2021-02-03 11:52:05,519 INFO sqlalchemy.engine.Engine [track2] [raw sql] ()
2021-02-03 11:52:05,520 DEBUG sqlalchemy.engine.Engine [track2] Col ('1',)
2021-02-03 11:52:05,520 DEBUG sqlalchemy.engine.Engine [track2] Row (1,)
```

### 隐藏参数

`Engine` 发出的日志还指示了对于特定语句存在的 SQL 参数的摘录。为了防止出于隐私目的记录这些参数，请启用 `create_engine.hide_parameters` 标志：

```py
>>> e = create_engine("sqlite://", echo=True, hide_parameters=True)
>>> with e.connect() as conn:
...     conn.execute(text("select :some_private_name"), {"some_private_name": "pii"})
2020-10-24 12:48:32,808 INFO sqlalchemy.engine.Engine select ?
2020-10-24 12:48:32,808 INFO sqlalchemy.engine.Engine [SQL parameters hidden due to hide_parameters=True]
```
