# 附加持久化技术

> 原文：[`docs.sqlalchemy.org/en/20/orm/persistence_techniques.html`](https://docs.sqlalchemy.org/en/20/orm/persistence_techniques.html)

## 将 SQL 插入/更新表达式嵌入到刷新中

此功能允许将数据库列的值设置为 SQL 表达式而不是文字值。这对于原子更新、调用存储过程等特别有用。您所要做的就是将表达式分配给属性：

```py
class SomeClass(Base):
    __tablename__ = "some_table"

    # ...

    value = mapped_column(Integer)

someobject = session.get(SomeClass, 5)

# set 'value' attribute to a SQL expression adding one
someobject.value = SomeClass.value + 1

# issues "UPDATE some_table SET value=value+1"
session.commit()
```

这种技术对于 INSERT 和 UPDATE 语句都适用。在刷新/提交操作之后，上述`someobject`的`value`属性将会过期，因此在下次访问时，新生成的值将从数据库加载。

该功能还具有条件支持，可以与主键列一起使用。对于支持 RETURNING 的后端（包括 Oracle、SQL Server、MariaDB 10.5、SQLite 3.35），还可以将 SQL 表达式分配给主键列。这允许对 SQL 表达式进行评估，并允许在 INSERT 时修改主键值的服务器端触发器成功地由 ORM 作为对象的主键的一部分检索：

```py
class Foo(Base):
    __tablename__ = "foo"
    pk = mapped_column(Integer, primary_key=True)
    bar = mapped_column(Integer)

e = create_engine("postgresql+psycopg2://scott:tiger@localhost/test", echo=True)
Base.metadata.create_all(e)

session = Session(e)

foo = Foo(pk=sql.select(sql.func.coalesce(sql.func.max(Foo.pk) + 1, 1)))
session.add(foo)
session.commit()
```

在 PostgreSQL 上，上述`Session`将发出以下 INSERT：

```py
INSERT  INTO  foo  (foopk,  bar)  VALUES
((SELECT  coalesce(max(foo.foopk)  +  %(max_1)s,  %(coalesce_2)s)  AS  coalesce_1
FROM  foo),  %(bar)s)  RETURNING  foo.foopk
```

新版本 1.3 中：现在可以在 ORM 刷新期间将 SQL 表达式传递给主键列；如果数据库支持 RETURNING，或者正在使用 pysqlite，ORM 将能够将服务器生成的值作为主键属性的值检索出来。  ## 在会话中使用 SQL 表达式

SQL 表达式和字符串可以通过`Session`在其事务上下文中执行。最简单的方法是使用`Session.execute()`方法，它以与`Engine`或`Connection`相同的方式返回一个`CursorResult`：

```py
Session = sessionmaker(bind=engine)
session = Session()

# execute a string statement
result = session.execute(text("select * from table where id=:id"), {"id": 7})

# execute a SQL expression construct
result = session.execute(select(mytable).where(mytable.c.id == 7))
```

由`Session`持有的当前`Connection`可通过`Session.connection()`方法访问：

```py
connection = session.connection()
```

上述示例涉及绑定到单个`Engine`或`Connection`的`Session`。要使用绑定到多个引擎或根本没有绑定（即依赖于绑定元数据）的`Session`执行语句，`Session.execute()`和`Session.connection()`都接受一个绑定参数字典`Session.execute.bind_arguments`，其中可能包括“mapper”，该参数传递了一个映射类或`Mapper`实例，用于定位所需引擎的适当上下文：

```py
Session = sessionmaker()
session = Session()

# need to specify mapper or class when executing
result = session.execute(
    text("select * from table where id=:id"),
    {"id": 7},
    bind_arguments={"mapper": MyMappedClass},
)

result = session.execute(
    select(mytable).where(mytable.c.id == 7), bind_arguments={"mapper": MyMappedClass}
)

connection = session.connection(MyMappedClass)
```

从 1.4 版本开始更改：`Session.execute()`中的`mapper`和`clause`参数现在作为发送给`Session.execute.bind_arguments`参数的字典的一部分传递。之前的参数仍然被接受，但此用法已被弃用。  ## 强制在具有默认值的列上使用 NULL

ORM 将任何从未在对象上设置过的属性视为“默认”情况；该属性将在 INSERT 语句中被省略：

```py
class MyObject(Base):
    __tablename__ = "my_table"
    id = mapped_column(Integer, primary_key=True)
    data = mapped_column(String(50), nullable=True)

obj = MyObject(id=1)
session.add(obj)
session.commit()  # INSERT with the 'data' column omitted; the database
# itself will persist this as the NULL value
```

在 INSERT 中省略一列意味着该列将设置为 NULL 值，*除非*该列设置了默认值，此时默认值将被保留。这既适用于纯 SQL 角度的服务器端默认值，也适用于 SQLAlchemy 在客户端和服务器端默认值下的插入行为：

```py
class MyObject(Base):
    __tablename__ = "my_table"
    id = mapped_column(Integer, primary_key=True)
    data = mapped_column(String(50), nullable=True, server_default="default")

obj = MyObject(id=1)
session.add(obj)
session.commit()  # INSERT with the 'data' column omitted; the database
# itself will persist this as the value 'default'
```

然而，在 ORM 中，即使将 Python 值`None`明确地分配给对象，这与从未分配值一样处理：

```py
class MyObject(Base):
    __tablename__ = "my_table"
    id = mapped_column(Integer, primary_key=True)
    data = mapped_column(String(50), nullable=True, server_default="default")

obj = MyObject(id=1, data=None)
session.add(obj)
session.commit()  # INSERT with the 'data' column explicitly set to None;
# the ORM still omits it from the statement and the
# database will still persist this as the value 'default'
```

上述操作将将服务器默认值`"default"`持久化到`data`列中，而不是 SQL NULL，即使传递了`None`；这是 ORM 的一个长期行为，许多应用程序将其视为一种假设。

那么如果我们想要实际将 NULL 放入这一列中，即使该列有默认值呢？有两种方法。一种是在每个实例级别上，我们使用`null` SQL 构造来分配属性：

```py
from sqlalchemy import null

obj = MyObject(id=1, data=null())
session.add(obj)
session.commit()  # INSERT with the 'data' column explicitly set as null();
# the ORM uses this directly, bypassing all client-
# and server-side defaults, and the database will
# persist this as the NULL value
```

`null` SQL 构造总是直接在目标 INSERT 语句中转换为 SQL NULL 值。

如果我们希望能够使用 Python 值 `None` 并且尽管存在列默认值，也希望将其持久化为 NULL，则可以在 ORM 中使用一个 Core 级别的修改器 `TypeEngine.evaluates_none()` 进行配置，该修改器指示 ORM 应该将值 `None` 与任何其他值一样对待，并将其传递，而不是将其视为“缺失”的值而省略它：

```py
class MyObject(Base):
    __tablename__ = "my_table"
    id = mapped_column(Integer, primary_key=True)
    data = mapped_column(
        String(50).evaluates_none(),  # indicate that None should always be passed
        nullable=True,
        server_default="default",
    )

obj = MyObject(id=1, data=None)
session.add(obj)
session.commit()  # INSERT with the 'data' column explicitly set to None;
# the ORM uses this directly, bypassing all client-
# and server-side defaults, and the database will
# persist this as the NULL value
```  ## 获取服务器生成的默认值

如在服务器调用的 DDL-显式默认表达式和标记隐式生成的值、时间戳和触发列章节中介绍的，Core 支持数据库列的概念，即数据库自身在 INSERT 语句中生成一个值，以及在较少见的情况下，在 UPDATE 语句中生成一个值。ORM 功能支持此类列，以便在刷新时能够获取这些新生成的值。在服务器生成的主键列的情况下，这种行为是必需的，因为一旦对象被持久化，ORM 就必须知道对象的主键。

在绝大多数情况下，由数据库自动生成值的主键列是简单的整数列，数据库实现为所谓的“自增”列，或者从与列关联的序列中生成。SQLAlchemy Core 中的每个数据库方言都支持一种检索这些主键值的方法，这种方法通常是 Python DBAPI 本地的，并且一般情况下这个过程是自动的。关于这一点还有更多的文档资料在 `Column.autoincrement`。

对于不是主键列或不是简单自增整数列的服务器生成列，ORM 要求这些列用适当的 `server_default` 指令标记，以允许 ORM 检索此值。然而，并非所有方法都支持所有后端，因此必须小心使用适当的方法。要回答的两个问题是，1\. 这一列是否是主键，2\. 数据库是否支持 RETURNING 或等效方法，如“OUTPUT inserted”；这些是在调用 INSERT 或 UPDATE 语句时同时返回服务器生成的值的 SQL 短语。RETURNING 目前由 PostgreSQL、Oracle、MariaDB 10.5、SQLite 3.35 和 SQL Server 支持。

### 情况 1：非主键，支持 RETURNING 或等效方法

在这种情况下，列应标记为 `FetchedValue` 或具有显式的 `Column.server_default`。当执行 INSERT 语句时，ORM 将自动将这些列添加到 RETURNING 子句中，假设 `Mapper.eager_defaults` 参数设置为 `True`，或者如果将其默认设置为 `"auto"`，对于同时支持 RETURNING 和 insertmanyvalues 的方言：

```py
class MyModel(Base):
    __tablename__ = "my_table"

    id = mapped_column(Integer, primary_key=True)

    # server-side SQL date function generates a new timestamp
    timestamp = mapped_column(DateTime(), server_default=func.now())

    # some other server-side function not named here, such as a trigger,
    # populates a value into this column during INSERT
    special_identifier = mapped_column(String(50), server_default=FetchedValue())

    # set eager defaults to True.  This is usually optional, as if the
    # backend supports RETURNING + insertmanyvalues, eager defaults
    # will take place regardless on INSERT
    __mapper_args__ = {"eager_defaults": True}
```

上面的 INSERT 语句未为“timestamp”或“special_identifier”指定客户端端显式值时，将在 RETURNING 子句中包括“timestamp”和“special_identifier”列，以便立即可用。在 PostgreSQL 数据库中，上述表的 INSERT 如下所示：

```py
INSERT  INTO  my_table  DEFAULT  VALUES  RETURNING  my_table.id,  my_table.timestamp,  my_table.special_identifier
```

从版本 2.0.0rc1 更改：`Mapper.eager_defaults` 参数现在默认为新设置 `"auto"`，它将自动使用 RETURNING 在 INSERT 时获取服务器生成的默认值，如果后备数据库同时支持 RETURNING 和 insertmanyvalues。

注意

`Mapper.eager_defaults` 的 `"auto"` 值仅适用于 INSERT 语句。即使可用，UPDATE 语句也不会使用 RETURNING，除非 `Mapper.eager_defaults` 设置为 `True`。这是因为 UPDATE 没有等价的“insertmanyvalues”功能，因此 UPDATE RETURNING 将要求为每个要 UPDATE 的行单独发出 UPDATE 语句。

### 情况 2：表中包含触发器生成的值，这些值与 RETURNING 不兼容

`Mapper.eager_defaults` 的 `"auto"` 设置意味着支持 RETURNING 的后端通常会在 INSERT 语句中使用 RETURNING 来检索新生成的默认值。但是，存在使用触发器生成的服务器生成值的限制，无法使用 RETURNING：

+   SQL Server 不允许在 INSERT 语句中使用 RETURNING 来检索触发器生成的值；该语句将失败。

+   SQLite 在将 RETURNING 与触发器结合使用时存在限制，因此 RETURNING 子句将不会具有 INSERTed 值可用

+   其他后端可能在与触发器一起使用 RETURNING 或其他类型的服务器生成的值时存在限制。

要禁用 RETURNING 用于这些值的使用，不仅包括服务器生成的默认值，还要确保 ORM 永远不会对特定表使用 RETURNING，请为映射的`Table`指定`Table.implicit_returning`参数为`False`。使用声明性映射，看起来像这样：

```py
class MyModel(Base):
    __tablename__ = "my_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    data: Mapped[str] = mapped_column(String(50))

    # assume a database trigger populates a value into this column
    # during INSERT
    special_identifier = mapped_column(String(50), server_default=FetchedValue())

    # disable all use of RETURNING for the table
    __table_args__ = {"implicit_returning": False}
```

在使用 pyodbc 驱动程序的 SQL Server 上，上述表的 INSERT 不会使用 RETURNING，并将使用 SQL Server `scope_identity()`函数检索新生成的主键值：

```py
INSERT  INTO  my_table  (data)  VALUES  (?);  select  scope_identity()
```

另请参阅

INSERT 行为 - SQL Server 方言获取新生成的主键值的方法的背景

### 情况 3：非主键、不支持或不需要 RETURNING 或等效功能

这种情况与上述情况 1 相同，除了我们通常不想使用`Mapper.eager_defaults`，因为在没有支持 RETURNING 的情况下，它的当前实现是发出每行一个 SELECT，这是不高效的。因此，在下面的映射中省略了该参数：

```py
class MyModel(Base):
    __tablename__ = "my_table"

    id = mapped_column(Integer, primary_key=True)
    timestamp = mapped_column(DateTime(), server_default=func.now())

    # assume a database trigger populates a value into this column
    # during INSERT
    special_identifier = mapped_column(String(50), server_default=FetchedValue())
```

在使用上述映射的记录在不包括 RETURNING 或“insertmanyvalues”支持的后端上 INSERT 之后， “timestamp”和“special_identifier”列将保持为空，并且将在刷新后首次访问时通过第二个 SELECT 语句获取，例如它们被标记为“过期”时。

如果`Mapper.eager_defaults`显式提供了一个值为`True`，并且后端数据库不支持 RETURNING 或等效功能，则 ORM 将在 INSERT 语句之后立即发出一个 SELECT 语句以获取新生成的值；如果没有可用的 RETURNING，则 ORM 目前无法批量选择许多新插入的行。这通常是不希望的，因为它会在刷新过程中添加额外的 SELECT 语句，这些语句可能是不必要的。将上述映射与`Mapper.eager_defaults`标志设置为 True 一起针对 MySQL（不是 MariaDB）使用会在刷新时产生如下 SQL：

```py
INSERT  INTO  my_table  ()  VALUES  ()

-- when eager_defaults **is** used, but RETURNING is not supported
SELECT  my_table.timestamp  AS  my_table_timestamp,  my_table.special_identifier  AS  my_table_special_identifier
FROM  my_table  WHERE  my_table.id  =  %s
```

未来的 SQLAlchemy 版本可能会试图改进在没有 RETURNING 的情况下的急切默认值的效率，以便在单个 SELECT 语句中批量处理多行。

### 情况 4：支持主键、RETURNING 或等效功能

具有服务器生成值的主键列必须在 INSERT 后立即获取；ORM 只能访问具有主键值的行，因此如果主键由服务器生成，则 ORM 需要一种在 INSERT 后立即检索该新值的方法。

如上所述，对于整数的“自增”列，以及标记为`Identity`的列和诸如 PostgreSQL SERIAL 等特殊构造，这些类型由 Core 自动处理；数据库包括用于获取“最后插入的 id”的函数，在不支持 RETURNING 的情况下，以及在支持 RETURNING 的情况下 SQLAlchemy 将使用它。

例如，在 Oracle 中使用标记为`Identity`的列，RETURNING 会自动用于获取新的主键值：

```py
class MyOracleModel(Base):
    __tablename__ = "my_table"

    id: Mapped[int] = mapped_column(Identity(), primary_key=True)
    data: Mapped[str] = mapped_column(String(50))
```

上述模型在 Oracle 上的 INSERT 如下所示：

```py
INSERT  INTO  my_table  (data)  VALUES  (:data)  RETURNING  my_table.id  INTO  :ret_0
```

SQLAlchemy 为“data”字段渲染了一个 INSERT，但只在 RETURNING 子句中包括了“id”，以便进行“id”的服务器端生成，并立即返回新值。

对于由服务器端函数或触发器生成的非整数值，以及来自表本身之外的构造的整数值，包括显式序列和触发器，必须在表元数据中标记服务器默认生成。再次以 Oracle 为例，我们可以用上面类似的表来说明使用 `Sequence` 构造命名的显式序列：

```py
class MyOracleModel(Base):
    __tablename__ = "my_table"

    id: Mapped[int] = mapped_column(Sequence("my_oracle_seq"), primary_key=True)
    data: Mapped[str] = mapped_column(String(50))
```

在 Oracle 上，这个版本的模型的 INSERT 如下所示：

```py
INSERT  INTO  my_table  (id,  data)  VALUES  (my_oracle_seq.nextval,  :data)  RETURNING  my_table.id  INTO  :ret_0
```

在上面的例子中，SQLAlchemy 为主键列渲染了 `my_sequence.nextval`，以便用于新的主键生成，并且还使用 RETURNING 立即获取新值。

如果数据源不是由简单的 SQL 函数或 `Sequence` 表示，例如在使用触发器或产生新值的数据库特定数据类型时，可以通过在列定义中使用 `FetchedValue` 来指示存在生成值的默认值。以下是一个使用 SQL Server TIMESTAMP 列作为主键的模型；在 SQL Server 上，这种数据类型会自动生成新值，因此在表元数据中通过为 `Column.server_default` 参数指示 `FetchedValue` 来指示：

```py
class MySQLServerModel(Base):
    __tablename__ = "my_table"

    timestamp: Mapped[datetime.datetime] = mapped_column(
        TIMESTAMP(), server_default=FetchedValue(), primary_key=True
    )
    data: Mapped[str] = mapped_column(String(50))
```

上述表在 SQL Server 上的 INSERT 如下所示：

```py
INSERT  INTO  my_table  (data)  OUTPUT  inserted.timestamp  VALUES  (?)
```

### 情况 5：主键，不支持 RETURNING 或等价的功能

在这个区域，我们正在为像 MySQL 这样的数据库生成行，其中一些生成默认值的方法是在服务器上进行的，但超出了数据库通常的自动增量程序。在这种情况下，我们必须确保 SQLAlchemy 可以“预执行”默认值，这意味着它必须是一个显式的 SQL 表达式。

注意

本节将以 MySQL 的 datetime 值为例，说明多个配方，因为此后端的 datetime 数据类型具有额外的特殊要求，这些要求对于说明很有用。但请记住，MySQL 对于用作主键的任何自动生成的数据类型都需要一个显式的“预执行”默认生成器，而不是通常的单列自增整数值。

#### 具有 DateTime 主键的 MySQL

使用 MySQL 的`DateTime`列的例子，我们使用“NOW()”SQL 函数添加了一个显式的预执行支持的默认值：

```py
class MyModel(Base):
    __tablename__ = "my_table"

    timestamp = mapped_column(DateTime(), default=func.now(), primary_key=True)
```

在上述例子中，我们选择“NOW()”函数来向列传递日期时间值。由上述生成的 SQL 如下：

```py
SELECT  now()  AS  anon_1
INSERT  INTO  my_table  (timestamp)  VALUES  (%s)
('2018-08-09 13:08:46',)
```

#### 具有 TIMESTAMP 主键的 MySQL

当在 MySQL 中使用`TIMESTAMP`数据类型时，MySQL 通常会自动将服务器端默认值与此数据类型关联起来。但是，当我们将其用作主键时，核心无法检索新生成的值，除非我们自己执行函数。由于 MySQL 上的`TIMESTAMP`实际上存储了一个二进制值，因此我们需要在使用“NOW()”时添加额外的“CAST”，以便检索到可以持久化到列中的二进制值： 

```py
from sqlalchemy import cast, Binary

class MyModel(Base):
    __tablename__ = "my_table"

    timestamp = mapped_column(
        TIMESTAMP(), default=cast(func.now(), Binary), primary_key=True
    )
```

除了选择“NOW()”函数之外，在上面的例子中，我们还额外使用`Binary`数据类型结合`cast()`，以便返回的值是二进制的。从上面插入的 SQL 看起来像：

```py
SELECT  CAST(now()  AS  BINARY)  AS  anon_1
INSERT  INTO  my_table  (timestamp)  VALUES  (%s)
(b'2018-08-09 13:08:46',)
```

另请参阅

列插入/更新默认值

### 关于急切获取客户端调用的用于 INSERT 或 UPDATE 的 SQL 表达式的注意事项

前面的例子表明了使用`Column.server_default`创建包含默认生成函数的表格的 DDL。

SQLAlchemy 还支持非 DDL 服务器端默认值，如文档中所述客户端调用的 SQL 表达式; 这些“客户端调用的 SQL 表达式”是使用`Column.default`和`Column.onupdate`参数设置的。

当 `Mapper.eager_defaults` 设置为 `"auto"` 或 `True` 时，这些 SQL 表达式目前受到 ORM 中与真正的服务器端默认值相同的限制；除非将 `FetchedValue` 指令与 `Column` 关联，否则当使用 RETURNING 时，它们不会被自动获取，尽管这些表达式不是 DDL 服务器默认值，而是由 SQLAlchemy 本身积极渲染的。这个限制可能在未来的 SQLAlchemy 版本中得到解决。

`FetchedValue` 结构可以同时应用于 `Column.server_default` 或 `Column.server_onupdate`，以及与 `Column.default` 和 `Column.onupdate` 一起使用的 SQL 表达式，例如下面的示例，其中 `func.now()` 结构用作客户端调用的 SQL 表达式，用于 `Column.default` 和 `Column.onupdate`。为了让 `Mapper.eager_defaults` 的行为包括使用 RETURNING 时获取这些值，需要使用 `FetchedValue` 来确保进行获取：

```py
class MyModel(Base):
    __tablename__ = "my_table"

    id = mapped_column(Integer, primary_key=True)

    created = mapped_column(
        DateTime(), default=func.now(), server_default=FetchedValue()
    )
    updated = mapped_column(
        DateTime(),
        onupdate=func.now(),
        server_default=FetchedValue(),
        server_onupdate=FetchedValue(),
    )

    __mapper_args__ = {"eager_defaults": True}
```

与上述类似的映射，ORM 渲染的 INSERT 和 UPDATE 的 SQL 将在 RETURNING 子句中包含 `created` 和 `updated`：

```py
INSERT  INTO  my_table  (created)  VALUES  (now())  RETURNING  my_table.id,  my_table.created,  my_table.updated

UPDATE  my_table  SET  updated=now()  WHERE  my_table.id  =  %(my_table_id)s  RETURNING  my_table.updated
```  ## 使用 INSERT、UPDATE 和 ON CONFLICT（即 upsert）返回 ORM 对象

SQLAlchemy 2.0 包括增强功能，用于发出几种类型的 ORM-enabled INSERT、UPDATE 和 upsert 语句。请查看文档 ORM-Enabled INSERT, UPDATE, and DELETE statements。有关 upsert，请参见 ORM “upsert” Statements。

### 使用 PostgreSQL ON CONFLICT 与 RETURNING 返回 upserted ORM 对象

本节已移至 ORM "upsert"语句。## 分区策略（例如，每个会话多个数据库后端）

### 简单的垂直分区

垂直分区将不同的类、类层次结构或映射表配置到多个数据库中，方法是通过使用`Session`的`Session.binds`参数。该参数接收一个包含任意 ORM 映射类、映射层次结构中的任意类（如声明基类或混合类）、`Table`对象和`Mapper`对象的字典作为键，然后通常引用`Engine`或不太常见的是`Connection`对象作为目标。每当`Session`需要代表特定类型的映射类发出 SQL 以定位适当的数据库连接源时，都会查询该字典：

```py
engine1 = create_engine("postgresql+psycopg2://db1")
engine2 = create_engine("postgresql+psycopg2://db2")

Session = sessionmaker()

# bind User operations to engine 1, Account operations to engine 2
Session.configure(binds={User: engine1, Account: engine2})

session = Session()
```

在上述情况下，针对任何一个类的 SQL 操作都将使用与该类相关联的`Engine`。该功能在读取和写入操作中都是全面的；针对映射到`engine1`的实体的`Query`（通过查看请求项列表中的第一个实体来确定）将使用`engine1`来运行查询。刷新操作将基于每个类使用**两个**引擎，因为它会刷新`User`和`Account`类型的对象。

在更常见的情况下，通常有基类或混合类可用于区分目的地不同数据库连接的操作。`Session.binds`参数可以容纳任何任意的 Python 类作为键，如果发现它在特定映射类的`__mro__`（Python 方法解析顺序）中，则将使用它。假设有两个声明基类代表两个不同的数据库连接：

```py
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Session

class BaseA(DeclarativeBase):
    pass

class BaseB(DeclarativeBase):
    pass

class User(BaseA): ...

class Address(BaseA): ...

class GameInfo(BaseB): ...

class GameStats(BaseB): ...

Session = sessionmaker()

# all User/Address operations will be on engine 1, all
# Game operations will be on engine 2
Session.configure(binds={BaseA: engine1, BaseB: engine2})
```

以上，从`BaseA`和`BaseB`继承的类将根据它们是否从任何一个超类继承来路由它们的 SQL 操作到两个引擎之一。在一个类从多个“绑定”超类继承的情况下，将选择目标类层次结构中最高的超类来表示应该使用哪个引擎。

另请参见

`Session.binds`

### 多引擎会话的事务协调

使用多个绑定引擎时的一个注意事项是，当提交操作在一个后端成功提交后，可能在另一个后端失败。这是一个不一致性问题，在关系型数据库中通过使用“两阶段事务”解决，它在提交序列中增加了一个额外的“准备”步骤，允许多个数据库在实际完成事务之前同意提交。

由于 DBAPI 中的支持有限，SQLAlchemy 在跨后端的两阶段事务方面的支持也有限。通常，它被认为与 PostgreSQL 后端很好地配合使用，而与 MySQL 后端配合使用的程度较低。但是，当后端支持时，`Session` 完全能够利用两阶段事务功能，方法是在 `sessionmaker` 或 `Session` 中设置 `Session.use_twophase` 标志。请参阅启用两阶段提交以获取示例。

### 自定义垂直分区

通过重写 `Session.get_bind()` 方法可以构建更全面的基于规则的类级别分区。下面我们展示一个自定义的 `Session` ，它提供以下规则：

1.  刷新操作以及批量的“更新”和“删除”操作都发送到名为 `leader` 的引擎。

1.  所有子类为 `MyOtherClass` 的对象操作都发生在 `other` 引擎上。

1.  所有其他类的读操作都发生在 `follower1` 或 `follower2` 数据库的随机选择上。

```py
engines = {
    "leader": create_engine("sqlite:///leader.db"),
    "other": create_engine("sqlite:///other.db"),
    "follower1": create_engine("sqlite:///follower1.db"),
    "follower2": create_engine("sqlite:///follower2.db"),
}

from sqlalchemy.sql import Update, Delete
from sqlalchemy.orm import Session, sessionmaker
import random

class RoutingSession(Session):
    def get_bind(self, mapper=None, clause=None):
        if mapper and issubclass(mapper.class_, MyOtherClass):
            return engines["other"]
        elif self._flushing or isinstance(clause, (Update, Delete)):
            # NOTE: this is for example, however in practice reader/writer
            # splits are likely more straightforward by using two distinct
            # Sessions at the top of a "reader" or "writer" operation.
            # See note below
            return engines["leader"]
        else:
            return engines[random.choice(["follower1", "follower2"])]
```

上述的 `Session` 类通过 `class_` 参数插入到 `sessionmaker` 中：

```py
Session = sessionmaker(class_=RoutingSession)
```

这种方法可以与多个 `MetaData` 对象结合使用，使用类似使用声明性 `__abstract__` 关键字的方法，详细描述在 __abstract__。

注意

虽然上面的示例说明了将特定的 SQL 语句路由到所谓的“领导者”或“跟随者”数据库的方法，这可能不是一个实际的方法，因为它导致了在同一操作中读写之间的不协调事务行为。实际上，最好根据正在进行的整体操作/事务，提前将 `Session` 构造为“读取器”或“写入器”会话。这样，将要写入数据的操作也将在同一事务范围内发出其读取查询。参见 为 Sessionmaker / Engine 设置隔离级别 中的示例，该示例设置了一个用于使用自动提交连接的“只读”操作的 `sessionmaker` ，另一个用于“写入”操作，其中包括 DML / COMMIT。

另请参阅

[在 SQLAlchemy 中实现 Django 风格的数据库路由器](https://techspot.zzzeek.org/2012/01/11/django-style-database-routers-in-sqlalchemy/) - 关于更全面的`Session.get_bind()`的博客文章。

### 水平分区

水平分区将单个表（或一组表）的行分布到多个数据库中。SQLAlchemy `Session` 包含对这个概念的支持，但要完全使用它，需要使用 `Session` 和 `Query` 的子类。这些子类的基本版本可在 水平分片 ORM 扩展中找到。一个使用示例在这里：水平分片。  ## 批量操作

传统特性

SQLAlchemy 2.0 已将“批量插入”和“批量更新”功能集成到 2.0 风格的 `Session.execute()` 方法中，直接使用了 `Insert` 和 `Update` 构造。参见文档 ORM-Enabled INSERT, UPDATE, and DELETE statements，包括传统的会话批量 INSERT 方法 ，该文档说明了从旧方法迁移到新方法的示例。  ## 将 SQL 插入/更新表达式嵌入到刷新中

此功能允许将数据库列的值设置为 SQL 表达式，而不是文字值。对于原子更新、调用存储过程等特别有用。您所做的一切就是将表达式分配给属性：

```py
class SomeClass(Base):
    __tablename__ = "some_table"

    # ...

    value = mapped_column(Integer)

someobject = session.get(SomeClass, 5)

# set 'value' attribute to a SQL expression adding one
someobject.value = SomeClass.value + 1

# issues "UPDATE some_table SET value=value+1"
session.commit()
```

此技术对于 INSERT 和 UPDATE 语句均有效。在 flush/commit 操作之后，上述 `someobject` 上的 `value` 属性将过期，因此在下次访问时，新生成的值将从数据库加载。

该功能还具有条件支持，可与主键列一起使用。对于支持 RETURNING 的后端（包括 Oracle、SQL Server、MariaDB 10.5、SQLite 3.35），还可以将 SQL 表达式分配给主键列。这不仅允许评估 SQL 表达式，还允许检索任何在插入时修改主键值的服务器端触发器作为对象主键的一部分成功地检索到 ORM：

```py
class Foo(Base):
    __tablename__ = "foo"
    pk = mapped_column(Integer, primary_key=True)
    bar = mapped_column(Integer)

e = create_engine("postgresql+psycopg2://scott:tiger@localhost/test", echo=True)
Base.metadata.create_all(e)

session = Session(e)

foo = Foo(pk=sql.select(sql.func.coalesce(sql.func.max(Foo.pk) + 1, 1)))
session.add(foo)
session.commit()
```

在 PostgreSQL 上，上述 `Session` 将发出以下 INSERT：

```py
INSERT  INTO  foo  (foopk,  bar)  VALUES
((SELECT  coalesce(max(foo.foopk)  +  %(max_1)s,  %(coalesce_2)s)  AS  coalesce_1
FROM  foo),  %(bar)s)  RETURNING  foo.foopk
```

从版本 1.3 新增：在 ORM flush 期间，SQL 表达式现在可以传递到主键列；如果数据库支持 RETURNING，或者正在使用 pysqlite，则 ORM 将能够将服务器生成的值检索为主键属性的值。

## 使用 SQL 表达式与会话

SQL 表达式和字符串可以通过其事务上下文在 `Session` 中执行。这最容易通过 `Session.execute()` 方法来实现，该方法以与 `Engine` 或 `Connection` 相同的方式返回一个 `CursorResult`：

```py
Session = sessionmaker(bind=engine)
session = Session()

# execute a string statement
result = session.execute(text("select * from table where id=:id"), {"id": 7})

# execute a SQL expression construct
result = session.execute(select(mytable).where(mytable.c.id == 7))
```

`Session` 当前持有的 `Connection` 可以通过 `Session.connection()` 方法访问：

```py
connection = session.connection()
```

上面的示例涉及到绑定到单个 `Engine` 或 `Connection` 的 `Session`。要使用绑定到多个引擎或根本没有绑定到引擎的 `Session` 执行语句，`Session.execute()` 和 `Session.connection()` 都接受一个绑定参数字典 `Session.execute.bind_arguments`，其中可能包括 “mapper”，该参数传递了一个映射类或 `Mapper` 实例，用于定位所需引擎的正确上下文：

```py
Session = sessionmaker()
session = Session()

# need to specify mapper or class when executing
result = session.execute(
    text("select * from table where id=:id"),
    {"id": 7},
    bind_arguments={"mapper": MyMappedClass},
)

result = session.execute(
    select(mytable).where(mytable.c.id == 7), bind_arguments={"mapper": MyMappedClass}
)

connection = session.connection(MyMappedClass)
```

从版本 1.4 开始变更：`Session.execute()` 的 `mapper` 和 `clause` 参数现在作为字典的一部分发送，作为 `Session.execute.bind_arguments` 参数。以前的参数仍然被接受，但此用法已被弃用。

## 强制将 NULL 值放入具有默认值的列

ORM 认为对象上从未设置的任何属性都是“默认”情况；该属性将从 INSERT 语句中省略：

```py
class MyObject(Base):
    __tablename__ = "my_table"
    id = mapped_column(Integer, primary_key=True)
    data = mapped_column(String(50), nullable=True)

obj = MyObject(id=1)
session.add(obj)
session.commit()  # INSERT with the 'data' column omitted; the database
# itself will persist this as the NULL value
```

如果在 INSERT 中省略了某列，则该列将被设置为 NULL 值，*除非*该列设置了默认值，在这种情况下，默认值将被保留。这适用于纯 SQL 视角下具有服务器端默认值的情况，也适用于 SQLAlchemy 的插入行为，无论是客户端默认值还是服务器端默认值：

```py
class MyObject(Base):
    __tablename__ = "my_table"
    id = mapped_column(Integer, primary_key=True)
    data = mapped_column(String(50), nullable=True, server_default="default")

obj = MyObject(id=1)
session.add(obj)
session.commit()  # INSERT with the 'data' column omitted; the database
# itself will persist this as the value 'default'
```

然而，在 ORM 中，即使将 Python 值 `None` 显式地分配给对象，这也被视为**相同**，就像从未分配过值一样：

```py
class MyObject(Base):
    __tablename__ = "my_table"
    id = mapped_column(Integer, primary_key=True)
    data = mapped_column(String(50), nullable=True, server_default="default")

obj = MyObject(id=1, data=None)
session.add(obj)
session.commit()  # INSERT with the 'data' column explicitly set to None;
# the ORM still omits it from the statement and the
# database will still persist this as the value 'default'
```

上述操作将持久化到 `data` 列的服务器默认值为 `"default"`，而不是 SQL NULL，即使传递了 `None`；这是 ORM 的长期行为，许多应用程序都将其视为假设。

那么，如果我们想要在这列中实际放入 NULL 值，即使该列有默认值呢？有两种方法。一种是在每个实例级别上，我们使用 `null` SQL 构造分配属性：

```py
from sqlalchemy import null

obj = MyObject(id=1, data=null())
session.add(obj)
session.commit()  # INSERT with the 'data' column explicitly set as null();
# the ORM uses this directly, bypassing all client-
# and server-side defaults, and the database will
# persist this as the NULL value
```

`null` SQL 结构总是将 SQL NULL 值直接包含在目标 INSERT 语句中。

如果我们希望能够使用 Python 值 `None` 并且将其作为 NULL 持久化，尽管存在列默认值，我们可以在 ORM 中使用 Core 级别的修饰符 `TypeEngine.evaluates_none()` 进行配置，该修饰符指示 ORM 应该将值 `None` 与任何其他值一样对待并将其传递，而不是将其省略为“丢失”的值：

```py
class MyObject(Base):
    __tablename__ = "my_table"
    id = mapped_column(Integer, primary_key=True)
    data = mapped_column(
        String(50).evaluates_none(),  # indicate that None should always be passed
        nullable=True,
        server_default="default",
    )

obj = MyObject(id=1, data=None)
session.add(obj)
session.commit()  # INSERT with the 'data' column explicitly set to None;
# the ORM uses this directly, bypassing all client-
# and server-side defaults, and the database will
# persist this as the NULL value
```

## 获取服务器生成的默认值

正如在章节 Server-invoked DDL-Explicit Default Expressions 和 Marking Implicitly Generated Values, timestamps, and Triggered Columns 中介绍的，Core 支持数据库列的概念，其中数据库本身在 INSERT 时生成值，在不太常见的情况下，在 UPDATE 语句中生成值。ORM 功能支持这些列，以便能够在刷新时获取这些新生成的值。在服务器生成的主键列的情况下，由于 ORM 必须在对象持久化后知道其主键，因此需要这种行为。

在绝大多数情况下，由数据库自动生成值的主键列都是简单的整数列，这些列由数据库实现为所谓的“自增”列，或者是与列关联的序列。SQLAlchemy Core 中的每个数据库方言都支持一种检索这些主键值的方法，通常是原生于 Python DBAPI，并且通常这个过程是自动的。关于这一点，有更多的文档说明在 `Column.autoincrement` 中。

对于不是主键列或不是简单自增整数列的服务器生成列，ORM 要求这些列使用适当的 `server_default` 指令标记，以允许 ORM 检索此值。然而，并不是所有方法都受到所有后端的支持，因此必须注意使用适当的方法。要回答的两个问题是，1\. 此列是否是主键列，2\. 数据库是否支持 RETURNING 或等效操作，如 “OUTPUT inserted”；这些是 SQL 短语，它们在调用 INSERT 或 UPDATE 语句时同时返回服务器生成的值。RETURNING 目前由 PostgreSQL、Oracle、MariaDB 10.5、SQLite 3.35 和 SQL Server 支持。

### 情况 1：非主键，支持 RETURNING 或等效操作

在这种情况下，列应标记为`FetchedValue`，或者用显式的`Column.server_default`。ORM 在执行 INSERT 语句时将自动将这些列添加到 RETURNING 子句中，假设 `Mapper.eager_defaults` 参数设置为 `True`，或者对于同时支持 RETURNING 和 insertmanyvalues 的方言，保持其默认设置为 `"auto"`：

```py
class MyModel(Base):
    __tablename__ = "my_table"

    id = mapped_column(Integer, primary_key=True)

    # server-side SQL date function generates a new timestamp
    timestamp = mapped_column(DateTime(), server_default=func.now())

    # some other server-side function not named here, such as a trigger,
    # populates a value into this column during INSERT
    special_identifier = mapped_column(String(50), server_default=FetchedValue())

    # set eager defaults to True.  This is usually optional, as if the
    # backend supports RETURNING + insertmanyvalues, eager defaults
    # will take place regardless on INSERT
    __mapper_args__ = {"eager_defaults": True}
```

在上面的示例中，如果客户端未为“timestamp”或“special_identifier”指定显式值，则 INSERT 语句将在 RETURNING 子句中包含“timestamp”和“special_identifier”列，以便立即使用。在 PostgreSQL 数据库中，上述表的 INSERT 如下所示：

```py
INSERT  INTO  my_table  DEFAULT  VALUES  RETURNING  my_table.id,  my_table.timestamp,  my_table.special_identifier
```

从版本 2.0.0rc1 起更改：`Mapper.eager_defaults` 参数现在默认为新设置 `"auto"`，如果后端数据库同时支持 RETURNING 和 insertmanyvalues，将自动使用 RETURNING 获取 INSERT 上生成的默认值。

注意

`Mapper.eager_defaults` 的 `"auto"` 值仅适用于 INSERT 语句。即使可用，UPDATE 语句也不会使用 RETURNING，除非 `Mapper.eager_defaults` 设置为 `True`。这是因为 UPDATE 没有等效的“insertmanyvalues”特性，因此 UPDATE RETURNING 将要求对每个要更新的行分别发出 UPDATE 语句。

### 情况 2：表包含不兼容于 RETURNING 的触发器生成的值

`Mapper.eager_defaults` 的 `"auto"` 设置意味着支持 RETURNING 的后端通常会在 INSERT 语句中使用 RETURNING 以检索新生成的默认值。但是，使用触发器生成的服务器值存在限制，使得无法使用 RETURNING：

+   SQL Server 不允许在 INSERT 语句中使用 RETURNING 来检索触发器生成的值；该语句将失败。

+   SQLite 在将 RETURNING 与触发器组合使用时存在限制，因此 RETURNING 子句将不会包含插入的值

+   其他后端可能在与触发器一起使用 RETURNING，或者其他类型的服务器生成值时存在限制。

要禁用对这些值的 RETURNING 使用，不仅包括服务器生成的默认值，还要确保 ORM 永远不会与特定表使用 RETURNING，请为映射的 `Table` 指定 `Table.implicit_returning` 为 `False`。使用声明性映射如下所示：

```py
class MyModel(Base):
    __tablename__ = "my_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    data: Mapped[str] = mapped_column(String(50))

    # assume a database trigger populates a value into this column
    # during INSERT
    special_identifier = mapped_column(String(50), server_default=FetchedValue())

    # disable all use of RETURNING for the table
    __table_args__ = {"implicit_returning": False}
```

在使用 pyodbc 驱动程序的 SQL Server 上，对上述表的 INSERT 不会使用 RETURNING，并将使用 SQL Server 的 `scope_identity()` 函数来检索新生成的主键值：

```py
INSERT  INTO  my_table  (data)  VALUES  (?);  select  scope_identity()
```

另请参阅

INSERT 行为 - 关于 SQL Server 方言获取新生成主键值的方法的背景信息

### 情况 3：非主键，不支持或不需要 RETURNING 或等效功能

该情况与上述情况 1 相同，但通常我们不希望使用 `Mapper.eager_defaults`，因为在没有 RETURNING 支持的情况下，其当前实现是为每行发出一个 SELECT，这不是高效的。因此，在下面的映射中省略了该参数：

```py
class MyModel(Base):
    __tablename__ = "my_table"

    id = mapped_column(Integer, primary_key=True)
    timestamp = mapped_column(DateTime(), server_default=func.now())

    # assume a database trigger populates a value into this column
    # during INSERT
    special_identifier = mapped_column(String(50), server_default=FetchedValue())
```

在不包含 RETURNING 或“insertmanyvalues”支持的后端上插入具有上述映射的记录后，"timestamp" 和 "special_identifier" 列将保持为空，并且在刷新后首次访问时，例如标记为“过期”时，将通过第二个 SELECT 语句获取。

如果 `Mapper.eager_defaults` 明确提供了值 `True`，并且后端数据库不支持 RETURNING 或等效功能，则 ORM 将在 INSERT 语句后立即发出 SELECT 语句以获取新生成的值；如果没有 RETURNING 可用，ORM 目前无法批量选择许多新插入的行。这通常是不希望的，因为它会向刷新过程添加额外的 SELECT 语句，这些语句可能是不必要的。在 MySQL（而不是 MariaDB）上使用上述映射与将 `Mapper.eager_defaults` 标志设置为 True 会导致刷新时生成以下 SQL：

```py
INSERT  INTO  my_table  ()  VALUES  ()

-- when eager_defaults **is** used, but RETURNING is not supported
SELECT  my_table.timestamp  AS  my_table_timestamp,  my_table.special_identifier  AS  my_table_special_identifier
FROM  my_table  WHERE  my_table.id  =  %s
```

未来的 SQLAlchemy 版本可能会在没有 RETURNING 的情况下，通过批量处理单个 SELECT 语句中的多行来提高急切默认值的效率。

### 情况 4：主键，支持 RETURNING 或等效功能

具有服务器生成值的主键列必须在 INSERT 后立即获取；ORM 只能访问具有主键值的行，因此如果主键由服务器生成，则 ORM 需要一种在 INSERT 后立即检索该新值的方法。

如上所述，对于整数“自动增量”列，以及标记有 `Identity` 和特殊构造（如 PostgreSQL SERIAL）的列，Core 会自动处理这些类型；数据库包括用于获取“最后插入 id”的函数，在不支持 RETURNING 的情况下，以及支持 RETURNING 的情况下 SQLAlchemy 将使用该函数。

例如，在 Oracle 中，如果将列标记为 `Identity`，则自动使用 RETURNING 获取新的主键值：

```py
class MyOracleModel(Base):
    __tablename__ = "my_table"

    id: Mapped[int] = mapped_column(Identity(), primary_key=True)
    data: Mapped[str] = mapped_column(String(50))
```

如上所述，上述模型在 Oracle 上的 INSERT 如下所示：

```py
INSERT  INTO  my_table  (data)  VALUES  (:data)  RETURNING  my_table.id  INTO  :ret_0
```

SQLAlchemy 为“data”字段渲染了一个 INSERT，但在 RETURNING 子句中仅包含了“id”，以便在服务器端生成“id”，并立即返回新值。

对于由服务器端函数或触发器生成的非整数值，以及来自表格本身之外的结构（包括显式序列和触发器）的整数值，必须在表格元数据中标记服务器默认生成。再次以 Oracle 为例，我们可以举例说明一个类似上述的表格，使用 `Sequence` 构造命名一个显式序列：

```py
class MyOracleModel(Base):
    __tablename__ = "my_table"

    id: Mapped[int] = mapped_column(Sequence("my_oracle_seq"), primary_key=True)
    data: Mapped[str] = mapped_column(String(50))
```

在 Oracle 上，对于此模型的 INSERT 如下所示：

```py
INSERT  INTO  my_table  (id,  data)  VALUES  (my_oracle_seq.nextval,  :data)  RETURNING  my_table.id  INTO  :ret_0
```

在上述情况中，SQLAlchemy 为主键列渲染了 `my_sequence.nextval`，以便用于新的主键生成，并且还使用 RETURNING 立即获取新值。

如果数据源不是由简单的 SQL 函数或 `Sequence` 表示，例如在使用触发器或生成新值的数据库特定数据类型时，可以通过在列定义中使用 `FetchedValue` 来指示值生成默认值的存在。下面是一个使用 SQL Server TIMESTAMP 列作为主键的模型；在 SQL Server 上，此数据类型会自动生成新值，因此在表格元数据中通过为 `Column.server_default` 参数指示 `FetchedValue` 来表示这一点：

```py
class MySQLServerModel(Base):
    __tablename__ = "my_table"

    timestamp: Mapped[datetime.datetime] = mapped_column(
        TIMESTAMP(), server_default=FetchedValue(), primary_key=True
    )
    data: Mapped[str] = mapped_column(String(50))
```

在 SQL Server 上，对于上述表格的 INSERT 如下所示：

```py
INSERT  INTO  my_table  (data)  OUTPUT  inserted.timestamp  VALUES  (?)
```

### 情况 5：不支持主键、RETURNING 或等效功能。

在此领域，我们正在为 MySQL 等数据库生成行，其中服务器上正在发生一些默认生成的手段，但这些手段不在数据库的通常自增例程中。在这种情况下，我们必须确保 SQLAlchemy 可以“预先执行”默认值，这意味着它必须是一个明确的 SQL 表达式。

注意

本节将说明涉及 MySQL 日期时间值的多个配方，因为该后端的日期时间数据类型具有额外的特殊要求，这些要求对于说明非常有用。但是请注意，除了通常的单列自增整数值之外，MySQL 需要为*任何*用作主键的自动生成数据类型显式的“预执行”默认生成器。

#### 具有 DateTime 主键的 MySQL

以 MySQL 的`DateTime`列为例，我们使用“NOW()”SQL 函数添加了一个明确的预执行支持的默认值：

```py
class MyModel(Base):
    __tablename__ = "my_table"

    timestamp = mapped_column(DateTime(), default=func.now(), primary_key=True)
```

在上述情况下，我们选择“NOW()”函数以向列传递日期时间值。上述生成的 SQL 是：

```py
SELECT  now()  AS  anon_1
INSERT  INTO  my_table  (timestamp)  VALUES  (%s)
('2018-08-09 13:08:46',)
```

#### 具有 TIMESTAMP 主键的 MySQL

当使用 MySQL 的`TIMESTAMP`数据类型时，MySQL 通常会自动将服务器端默认与该数据类型关联起来。但是，当我们将其用作主键时，Core 无法检索新生成的值，除非我们自己执行该函数。由于 MySQL 上的`TIMESTAMP`实际上存储了一个二进制值，因此我们需要在“NOW()”的使用中添加一个额外的“CAST”，以便检索到可以持久化到列中的二进制值：

```py
from sqlalchemy import cast, Binary

class MyModel(Base):
    __tablename__ = "my_table"

    timestamp = mapped_column(
        TIMESTAMP(), default=cast(func.now(), Binary), primary_key=True
    )
```

以上，在选择“NOW()”函数的同时，我们还使用了`Binary`数据类型结合`cast()`，以便返回的值是二进制的。在 INSERT 中从上述渲染的 SQL 如下所示：

```py
SELECT  CAST(now()  AS  BINARY)  AS  anon_1
INSERT  INTO  my_table  (timestamp)  VALUES  (%s)
(b'2018-08-09 13:08:46',)
```

另见

列插入/更新默认值

### 注意事项：对于用于 INSERT 或 UPDATE 的急切提取客户端调用的 SQL 表达式

上述示例指示了使用`Column.server_default`创建包含其 DDL 中的默认生成函数的表。

SQLAlchemy 也支持非 DDL 服务器端的默认设置，如客户端调用的 SQL 表达式文档中所述；这些“客户端调用的 SQL 表达式”是使用`Column.default`和`Column.onupdate`参数设置的。

目前，ORM 中的这些 SQL 表达式受到与真正的服务器端默认值相同的限制；当 `Mapper.eager_defaults` 设置为 `"auto"` 或 `True` 时，除非 `FetchedValue` 指令与 `Column` 相关联，否则它们不会被 RETURNING 急切地获取，尽管这些表达式不是 DDL 服务器默认值，并且由 SQLAlchemy 本身主动渲染。这个限制可能在未来的 SQLAlchemy 版本中得到解决。

`FetchedValue` 构造可以同时应用于 `Column.server_default` 或 `Column.server_onupdate`，就像下面的例子中使用 `func.now()` 构造作为客户端调用的 SQL 表达式用于 `Column.default` 和 `Column.onupdate` 一样。为了使 `Mapper.eager_defaults` 的行为包括在可用时使用 RETURNING 获取这些值，`Column.server_default` 和 `Column.server_onupdate` 与 `FetchedValue` 一起使用以确保获取发生：

```py
class MyModel(Base):
    __tablename__ = "my_table"

    id = mapped_column(Integer, primary_key=True)

    created = mapped_column(
        DateTime(), default=func.now(), server_default=FetchedValue()
    )
    updated = mapped_column(
        DateTime(),
        onupdate=func.now(),
        server_default=FetchedValue(),
        server_onupdate=FetchedValue(),
    )

    __mapper_args__ = {"eager_defaults": True}
```

使用类似上面的映射，ORM 渲染的 INSERT 和 UPDATE 的 SQL 将在 RETURNING 子句中包括`created`和`updated`：

```py
INSERT  INTO  my_table  (created)  VALUES  (now())  RETURNING  my_table.id,  my_table.created,  my_table.updated

UPDATE  my_table  SET  updated=now()  WHERE  my_table.id  =  %(my_table_id)s  RETURNING  my_table.updated
```

### 情况 1：非主键，支持 RETURNING 或等效功能

在这种情况下，应将列标记为`FetchedValue`或具有显式的`Column.server_default`。如果`Mapper.eager_defaults`参数设置为`True`，或者对于支持 RETURNING 以及 insertmanyvalues 的方言，默认设置为`"auto"`，ORM 将在执行 INSERT 语句时自动将这些列添加到 RETURNING 子句中：

```py
class MyModel(Base):
    __tablename__ = "my_table"

    id = mapped_column(Integer, primary_key=True)

    # server-side SQL date function generates a new timestamp
    timestamp = mapped_column(DateTime(), server_default=func.now())

    # some other server-side function not named here, such as a trigger,
    # populates a value into this column during INSERT
    special_identifier = mapped_column(String(50), server_default=FetchedValue())

    # set eager defaults to True.  This is usually optional, as if the
    # backend supports RETURNING + insertmanyvalues, eager defaults
    # will take place regardless on INSERT
    __mapper_args__ = {"eager_defaults": True}
```

在上述情况下，未在客户端指定“timestamp”或“special_identifier”的显式值的 INSERT 语句将包括“timestamp”和“special_identifier”列在 RETURNING 子句中，以便立即使用。在 PostgreSQL 数据库上，上述表的 INSERT 将如下所示：

```py
INSERT  INTO  my_table  DEFAULT  VALUES  RETURNING  my_table.id,  my_table.timestamp,  my_table.special_identifier
```

从版本 2.0.0rc1 开始更改：`Mapper.eager_defaults`参数现在默认为新设置`"auto"`，如果支持 RETURNING 以及 insertmanyvalues 的后端数据库，则会自动使用 RETURNING 来获取 INSERT 时的服务器生成默认值。

注意

`Mapper.eager_defaults`的`"auto"`值仅适用于 INSERT 语句。即使可用，UPDATE 语句也不会使用 RETURNING，除非将`Mapper.eager_defaults`设置为`True`。这是因为 UPDATE 没有等效的“insertmanyvalues”特性，因此 UPDATE RETURNING 将要求为每个被 UPDATE 的行分别发出 UPDATE 语句。

### 情况 2：表包含与 RETURNING 不兼容的触发器生成的值

`Mapper.eager_defaults`的`"auto"`设置意味着支持 RETURNING 的后端通常会在 INSERT 语句中使用 RETURNING 来检索新生成的默认值。但是，存在使用触发器生成的服务器生成值的限制，因此不能使用 RETURNING：

+   SQL Server 不允许在 INSERT 语句中使用 RETURNING 来检索触发器生成的值；该语句将失败。

+   SQLite 在与触发器结合使用 RETURNING 时存在限制，因此 RETURNING 子句将无法获取已插入的值。

+   其他后端可能在与触发器或其他类型的服务器生成值结合使用 RETURNING 时存在限制。

要禁用用于此类值的 RETURNING 的使用，包括不仅用于服务器生成的默认值而且确保 ORM 永远不会使用 RETURNING 与特定表，指定 `Table.implicit_returning` 为 `False` 对于映射的 `Table`。使用声明性映射，看起来像这样：

```py
class MyModel(Base):
    __tablename__ = "my_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    data: Mapped[str] = mapped_column(String(50))

    # assume a database trigger populates a value into this column
    # during INSERT
    special_identifier = mapped_column(String(50), server_default=FetchedValue())

    # disable all use of RETURNING for the table
    __table_args__ = {"implicit_returning": False}
```

在使用 pyodbc 驱动程序的 SQL Server 上，对于上述表的 INSERT 不会使用 RETURNING，并且将使用 SQL Server `scope_identity()` 函数来检索新生成的主键值：

```py
INSERT  INTO  my_table  (data)  VALUES  (?);  select  scope_identity()
```

另请参阅

INSERT 行为 - 关于 SQL Server 方言获取新生成的主键值的方法的背景

### 情况 3：非主键，不支持或不需要 RETURNING 或等效功能

该情况与上面的情况 1 相同，只是我们通常不想使用 `Mapper.eager_defaults`，因为在没有 RETURNING 支持的情况下，其当前实现是发出每行一个 SELECT，这是不高效的。因此，在下面的映射中省略了该参数：

```py
class MyModel(Base):
    __tablename__ = "my_table"

    id = mapped_column(Integer, primary_key=True)
    timestamp = mapped_column(DateTime(), server_default=func.now())

    # assume a database trigger populates a value into this column
    # during INSERT
    special_identifier = mapped_column(String(50), server_default=FetchedValue())
```

在上述映射中插入记录后，在不包括 RETURNING 或“insertmanyvalues”支持的后端上，“timestamp” 和 “special_identifier” 列将保持为空，并且在刷新后首次访问时将通过第二个 SELECT 语句获取，例如，它们被标记为“过期”时。

如果 `Mapper.eager_defaults` 明确提供了值 `True`，并且后端数据库不支持 RETURNING 或等效功能，则 ORM 将在 INSERT 语句后立即发出 SELECT 语句，以获取新生成的值；如果没有可用的 RETURNING，ORM 目前无法批量选择许多新插入的行。这通常是不可取的，因为它会向刷新过程添加额外的 SELECT 语句，这些语句可能是不需要的。使用上述映射，针对 MySQL（不是 MariaDB）将 `Mapper.eager_defaults` 标志设置为 True 在刷新时会产生类似以下的 SQL：

```py
INSERT  INTO  my_table  ()  VALUES  ()

-- when eager_defaults **is** used, but RETURNING is not supported
SELECT  my_table.timestamp  AS  my_table_timestamp,  my_table.special_identifier  AS  my_table_special_identifier
FROM  my_table  WHERE  my_table.id  =  %s
```

未来的 SQLAlchemy 版本可能会在没有 RETURNING 的情况下寻求改进急切默认值的效率，以在单个 SELECT 语句中批量处理多行。

### 情况 4：主键，支持 RETURNING 或等效功能

具有服务器生成值的主键列必须在 INSERT 后立即获取；ORM 只能访问具有主键值的行，因此如果主键由服务器生成，则 ORM 需要一种在 INSERT 后立即检索该新值的方法。

如上所述，对于整数“自增”列，以及标记为`Identity`的列和特殊构造，例如 PostgreSQL 的 SERIAL，这些类型将由核心自动处理；数据库包括获取“最后插入的 id”函数，其中不支持 RETURNING，而在支持 RETURNING 的情况下，SQLAlchemy 将使用它。

例如，使用 Oracle 并将列标记为`Identity`，RETURNING 将自动用于获取新的主键值：

```py
class MyOracleModel(Base):
    __tablename__ = "my_table"

    id: Mapped[int] = mapped_column(Identity(), primary_key=True)
    data: Mapped[str] = mapped_column(String(50))
```

如上模型在 Oracle 上的 INSERT 如下所示：

```py
INSERT  INTO  my_table  (data)  VALUES  (:data)  RETURNING  my_table.id  INTO  :ret_0
```

SQLAlchemy 渲染“data”字段的 INSERT，但仅在 RETURNING 子句中包含“id”，以便在服务器端生成“id”并立即返回新值。

对于由服务器端函数或触发器生成的非整数值，以及来自表本身之外的构造的整数值，包括显式序列和触发器，必须在表元数据中标记服务器默认生成。再次以 Oracle 为例，我们可以用`Sequence`构造说明一个类似的表：

```py
class MyOracleModel(Base):
    __tablename__ = "my_table"

    id: Mapped[int] = mapped_column(Sequence("my_oracle_seq"), primary_key=True)
    data: Mapped[str] = mapped_column(String(50))
```

Oracle 上此模型的 INSERT 如下所示：

```py
INSERT  INTO  my_table  (id,  data)  VALUES  (my_oracle_seq.nextval,  :data)  RETURNING  my_table.id  INTO  :ret_0
```

在上述情况下，SQLAlchemy 渲染`my_sequence.nextval`用于主键列的新主键生成，并且还使用 RETURNING 立即获取新值。

如果数据源不是由简单的 SQL 函数或`Sequence`表示，例如使用触发器或生成新值的数据库特定数据类型，可以通过在列定义中使用`FetchedValue`来指示值生成的默认值。下面是一个使用 SQL Server TIMESTAMP 列作为主键的模型；在 SQL Server 上，此数据类型会自动生成新值，因此在表元数据中通过为`Column.server_default`参数指定`FetchedValue`来指示：

```py
class MySQLServerModel(Base):
    __tablename__ = "my_table"

    timestamp: Mapped[datetime.datetime] = mapped_column(
        TIMESTAMP(), server_default=FetchedValue(), primary_key=True
    )
    data: Mapped[str] = mapped_column(String(50))
```

SQL Server 上上述表的 INSERT 如下所示：

```py
INSERT  INTO  my_table  (data)  OUTPUT  inserted.timestamp  VALUES  (?)
```

### 情况 5：不支持主键、RETURNING 或等效项。

在这个领域，我们为像 MySQL 这样的数据库生成行，其中服务器上正在发生某种默认生成的方法，但是超出了数据库的通常自动增量例程。在这种情况下，我们必须确保 SQLAlchemy 可以“预执行”默认值，这意味着它必须是一个显式的 SQL 表达式。

注

本节将说明 MySQL 中涉及日期时间值的多个示例，因为此后端的日期时间数据类型具有有用的额外特殊要求。但是请记住，MySQL 对于用作主键的任何自动生成的数据类型都需要明确的“预执行”默认生成器，除了通常的单列自增整数值。

#### MySQL 使用 DateTime 主键

使用 MySQL 的`DateTime`列作为例子，我们使用“NOW()”SQL 函数添加一个明确的预执行支持的默认值：

```py
class MyModel(Base):
    __tablename__ = "my_table"

    timestamp = mapped_column(DateTime(), default=func.now(), primary_key=True)
```

在上面的例子中，我们选择“NOW()”函数将日期时间值传递给列。由上述生成的 SQL 是：

```py
SELECT  now()  AS  anon_1
INSERT  INTO  my_table  (timestamp)  VALUES  (%s)
('2018-08-09 13:08:46',)
```

#### MySQL 使用 TIMESTAMP 主键

当在 MySQL 中使用`TIMESTAMP`数据类型时，MySQL 通常会自动将服务器端默认值与此数据类型关联起来。但是，当我们将其用作主键时，Core 无法检索到新生成的值，除非我们自己执行该函数。由于 MySQL 上的`TIMESTAMP`实际上存储的是二进制值，因此我们需要在“NOW()”的使用中添加额外的“CAST”，以便检索到可持久化到列中的二进制值：

```py
from sqlalchemy import cast, Binary

class MyModel(Base):
    __tablename__ = "my_table"

    timestamp = mapped_column(
        TIMESTAMP(), default=cast(func.now(), Binary), primary_key=True
    )
```

上述，除了选择“NOW()”函数外，我们还额外利用`Binary`数据类型与`cast()`结合使用，以便返回值是二进制的。在 INSERT 中生成的 SQL 如下所示：

```py
SELECT  CAST(now()  AS  BINARY)  AS  anon_1
INSERT  INTO  my_table  (timestamp)  VALUES  (%s)
(b'2018-08-09 13:08:46',)
```

另请参阅

列的 INSERT/UPDATE 默认值

#### MySQL 使用 DateTime 主键

使用 MySQL 的`DateTime`列作为例子，我们使用“NOW()”SQL 函数添加一个明确的预执行支持的默认值：

```py
class MyModel(Base):
    __tablename__ = "my_table"

    timestamp = mapped_column(DateTime(), default=func.now(), primary_key=True)
```

在上面的例子中，我们选择“NOW()”函数将日期时间值传递给列。由上述生成的 SQL 是：

```py
SELECT  now()  AS  anon_1
INSERT  INTO  my_table  (timestamp)  VALUES  (%s)
('2018-08-09 13:08:46',)
```

#### MySQL 使用 TIMESTAMP 主键

当在 MySQL 中使用`TIMESTAMP`数据类型时，MySQL 通常会自动将服务器端默认值与此数据类型关联起来。但是，当我们将其用作主键时，Core 无法检索到新生成的值，除非我们自己执行该函数。由于 MySQL 上的`TIMESTAMP`实际上存储的是二进制值，因此我们需要在“NOW()”的使用中添加额外的“CAST”，以便检索到可持久化到列中的二进制值：

```py
from sqlalchemy import cast, Binary

class MyModel(Base):
    __tablename__ = "my_table"

    timestamp = mapped_column(
        TIMESTAMP(), default=cast(func.now(), Binary), primary_key=True
    )
```

在上面的示例中，除了选择“NOW()”函数外，我们还使用`Binary`数据类型结合`cast()`，以使返回的值是二进制的。在 INSERT 中从上面渲染的 SQL 如下所示：

```py
SELECT  CAST(now()  AS  BINARY)  AS  anon_1
INSERT  INTO  my_table  (timestamp)  VALUES  (%s)
(b'2018-08-09 13:08:46',)
```

另请参阅

列插入/更新默认值

### 关于急切获取用于 INSERT 或 UPDATE 的客户端调用的 SQL 表达式的注意事项

前面的示例表明了使用`Column.server_default`创建包含默认生成函数的表的方法。

SQLAlchemy 也支持非 DDL 服务器端默认值，如客户端调用的 SQL 表达式文档所述；这些“客户端调用的 SQL 表达式”是使用`Column.default`和`Column.onupdate`参数设置的。

这些 SQL 表达式目前受 ORM 中与真正的服务器端默认值发生的相同限制的约束；当`Mapper.eager_defaults`设置为`"auto"`或`True`时，它们不会被急切地获取到 RETURNING 中，除非`FetchedValue`指令与`Column`关联，即使这些表达式不是 DDL 服务器默认值，而是由 SQLAlchemy 本身主动渲染的。这个限制可能在未来的 SQLAlchemy 版本中得到解决。

`FetchedValue` 构造可以同时应用于 `Column.server_default` 或 `Column.server_onupdate`，与 `Column.default` 和 `Column.onupdate` 一起使用 SQL 表达式，例如下面的示例中，`func.now()` 构造被用作客户端调用的 SQL 表达式，用于 `Column.default` 和 `Column.onupdate`。为了使 `Mapper.eager_defaults` 的行为包括使用 RETURNING 在可用时获取这些值，需要使用 `Column.server_default` 和 `Column.server_onupdate` 与 `FetchedValue` 以确保获取发生：

```py
class MyModel(Base):
    __tablename__ = "my_table"

    id = mapped_column(Integer, primary_key=True)

    created = mapped_column(
        DateTime(), default=func.now(), server_default=FetchedValue()
    )
    updated = mapped_column(
        DateTime(),
        onupdate=func.now(),
        server_default=FetchedValue(),
        server_onupdate=FetchedValue(),
    )

    __mapper_args__ = {"eager_defaults": True}
```

与上述类似的映射，ORM 渲染的 INSERT 和 UPDATE 的 SQL 将在 RETURNING 子句中包括 `created` 和 `updated`：

```py
INSERT  INTO  my_table  (created)  VALUES  (now())  RETURNING  my_table.id,  my_table.created,  my_table.updated

UPDATE  my_table  SET  updated=now()  WHERE  my_table.id  =  %(my_table_id)s  RETURNING  my_table.updated
```

## 使用 INSERT、UPDATE 和 ON CONFLICT（即 upsert）返回 ORM 对象

SQLAlchemy 2.0 包括增强功能，用于发出几种类型的启用 ORM 的 INSERT、UPDATE 和 upsert 语句。查看文档 ORM-Enabled INSERT, UPDATE, and DELETE statements 以获取文档。有关 upsert，请参见 ORM “upsert” Statements。

### 使用 PostgreSQL ON CONFLICT 与 RETURNING 返回 upserted ORM 对象

本节已移至 ORM “upsert” Statements。

### 使用 PostgreSQL ON CONFLICT 与 RETURNING 返回 upserted ORM 对象

本节已移至 ORM “upsert” Statements。

## 分区策略（例如，每个会话使用多个数据库后端）

### 简单的垂直分区

垂直分区通过配置`Session`的`Session.binds` 参数，将不同的类、类层次结构或映射表放置在多个数据库中。此参数接收一个包含任意组合的 ORM 映射类、映射层次结构中的任意类（如声明性基类或混合类）、`Table` 对象和`Mapper` 对象作为键的字典，然后通常引用`Engine` 或较不常见的 `Connection` 对象作为目标。每当`Session`需要代表特定类型的映射类发出 SQL 以定位适当的数据库连接源时，就会查询该字典：

```py
engine1 = create_engine("postgresql+psycopg2://db1")
engine2 = create_engine("postgresql+psycopg2://db2")

Session = sessionmaker()

# bind User operations to engine 1, Account operations to engine 2
Session.configure(binds={User: engine1, Account: engine2})

session = Session()
```

在上面，针对任一类的 SQL 操作都将使用与该类链接的`Engine`。该功能涵盖了读写操作；针对映射到`engine1`的实体的 `Query`（通过查看请求的项目列表中的第一个实体确定）将使用`engine1`来运行查询。刷新操作将根据每个类使用**两个**引擎，因为它刷新了`User`和`Account`类型的对象。

在更常见的情况下，通常有基础类或混合类可用于区分不同数据库连接的操作。`Session.binds` 参数可以容纳任何任意的 Python 类作为键，如果发现它在特定映射类的`__mro__`（Python 方法解析顺序）中，则会使用该键。假设两个声明性基类分别表示两个不同的数据库连接：

```py
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Session

class BaseA(DeclarativeBase):
    pass

class BaseB(DeclarativeBase):
    pass

class User(BaseA): ...

class Address(BaseA): ...

class GameInfo(BaseB): ...

class GameStats(BaseB): ...

Session = sessionmaker()

# all User/Address operations will be on engine 1, all
# Game operations will be on engine 2
Session.configure(binds={BaseA: engine1, BaseB: engine2})
```

在上面，从`BaseA`和`BaseB`继承的类将根据它们是否继承自任何超类来将它们的 SQL 操作路由到两个引擎中的一个。对于从多个“绑定”超类继承的类的情况，将选择目标类层次结构中最高的超类来表示应使用哪个引擎。

另请参阅

`Session.binds`

### 为多引擎会话协调事务

使用多个绑定引擎的一个注意事项是，如果提交操作在一个后端成功提交后在另一个后端失败，则可能会出现问题。这是一个一致性问题，在关系数据库中通过“两阶段事务”解决，它在提交序列中添加了一个额外的“准备”步骤，允许多个数据库在实际完成事务之前同意提交。

由于 DBAPI 中的支持有限，SQLAlchemy 对跨后端的两阶段事务的支持也有限。通常来说，它已知与 PostgreSQL 后端一起工作良好，与 MySQL 后端一起工作效果较差。然而，当后端支持时，`Session` 完全能够利用两阶段事务功能，方法是在 `sessionmaker` 或 `Session` 中设置 `Session.use_twophase` 标志。参见启用两阶段提交以获取示例。

### 自定义垂直分区

更全面的基于规则的类级分区可以通过覆盖 `Session.get_bind()` 方法来构建。下面我们演示了一个自定义的 `Session`，它提供以下规则：

1.  刷新操作以及批量的“更新”和“删除”操作都传递到名为 `leader` 的引擎。

1.  所有子类为 `MyOtherClass` 的对象的操作都发生在 `other` 引擎上。

1.  对于所有其他类的读取操作都发生在随机选择的 `follower1` 或 `follower2` 数据库上。

```py
engines = {
    "leader": create_engine("sqlite:///leader.db"),
    "other": create_engine("sqlite:///other.db"),
    "follower1": create_engine("sqlite:///follower1.db"),
    "follower2": create_engine("sqlite:///follower2.db"),
}

from sqlalchemy.sql import Update, Delete
from sqlalchemy.orm import Session, sessionmaker
import random

class RoutingSession(Session):
    def get_bind(self, mapper=None, clause=None):
        if mapper and issubclass(mapper.class_, MyOtherClass):
            return engines["other"]
        elif self._flushing or isinstance(clause, (Update, Delete)):
            # NOTE: this is for example, however in practice reader/writer
            # splits are likely more straightforward by using two distinct
            # Sessions at the top of a "reader" or "writer" operation.
            # See note below
            return engines["leader"]
        else:
            return engines[random.choice(["follower1", "follower2"])]
```

上述`Session` 类是通过 `sessionmaker` 的 `class_` 参数来插入的：

```py
Session = sessionmaker(class_=RoutingSession)
```

这种方法可以与多个 `MetaData` 对象结合使用，使用类似于使用声明性 `__abstract__` 关键字的方法，如 __abstract__ 中所述。

注意

上述示例说明了根据 SQL 语句是否期望写入数据将特定 SQL 语句路由到所谓的“主”或“从”数据库，但这可能不是一个实用的方法，因为它会导致在同一操作中读取和写入之间存在不协调的事务行为。实践中，最好在整个操作/事务进行的基础上，提前构建`Session`作为“读取者”或“写入者”会话。这样，将要写入数据的操作也会在同一个事务范围内发出其读取查询。有关在`sessionmaker`中设置“只读”操作的配方，使用自动提交连接，以及用于包含 DML/COMMIT 的“写入”操作的另一个配方，请参阅为 Sessionmaker / Engine 设置隔离级别的示例。

另请参阅

[SQLAlchemy 中的 Django 风格数据库路由器](https://techspot.zzzeek.org/2012/01/11/django-style-database-routers-in-sqlalchemy/) - 关于`Session.get_bind()`更全面示例的博文

### 水平分区

水平分区将单个表（或一组表）的行分区到多个数据库中。SQLAlchemy 的`Session`包含对这个概念的支持，但要完全使用它，需要使用`Session`和`Query`子类。这些子类的基本版本可在水平分区 ORM 扩展中找到。一个使用示例位于：水平分区。

### 简单的垂直分区

垂直分区将不同的类、类层次结构或映射表配置到多个数据库中，通过配置`Session`的`Session.binds`参数。该参数接收一个字典，其中包含任意组合的 ORM 映射类、映射层次结构内的任意类（例如声明基类或混合类）、`Table`对象和`Mapper`对象作为键，这些键通常引用`Engine`或更少见的情况下引用`Connection`对象作为目标。每当`Session`需要代表特定类型的映射类发出 SQL 以定位数据库连接的适当源时，就会查询该字典：

```py
engine1 = create_engine("postgresql+psycopg2://db1")
engine2 = create_engine("postgresql+psycopg2://db2")

Session = sessionmaker()

# bind User operations to engine 1, Account operations to engine 2
Session.configure(binds={User: engine1, Account: engine2})

session = Session()
```

在上述情况下，针对任一类的 SQL 操作将使用与该类链接的`Engine`。该功能在读写操作中都是全面的；针对映射到`engine1`的实体的`Query`（通过查看请求的项目列表中的第一个实体来确定）将使用`engine1`来运行查询。刷新操作将基于每个类使用**两个**引擎，因为它会刷新`User`和`Account`类型的对象。

在更常见的情况下，通常有基类或混合类可用于区分命令操作的目标数据库连接。`Session.binds`参数可以接受任何任意的 Python 类作为键，如果在特定映射类的`__mro__`（Python 方法解析顺序）中找到，则会使用该键。假设有两个声明基类代表两个不同的数据库连接：

```py
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Session

class BaseA(DeclarativeBase):
    pass

class BaseB(DeclarativeBase):
    pass

class User(BaseA): ...

class Address(BaseA): ...

class GameInfo(BaseB): ...

class GameStats(BaseB): ...

Session = sessionmaker()

# all User/Address operations will be on engine 1, all
# Game operations will be on engine 2
Session.configure(binds={BaseA: engine1, BaseB: engine2})
```

在上述情况下，从`BaseA`和`BaseB`继承的类将根据它们是否继承自其中任何一个超类而将其 SQL 操作路由到两个引擎中的一个。对于从多个“绑定”超类继承的类，将选择目标类层次结构中最高的超类来表示应该使用哪个引擎。

另请参阅

`Session.binds`

### 多引擎会话的事务协调

在使用多个绑定引擎的情况下，有一个需要注意的地方是，在一个提交操作在一个后端成功提交后，另一个后端可能失败。这是一个一致性问题，在关系型数据库中通过“两阶段事务”解决，该事务将一个额外的“准备”步骤添加到提交序列中，允许多个数据库在实际完成事务之前同意提交。

由于 DBAPI 的支持有限，SQLAlchemy 对跨后端的两阶段事务的支持也有限。最典型的是，它在 PostgreSQL 后端上运行良好，并且在 MySQL 后端上的支持较少。但是，当后端支持时，`Session`完全能够利用两阶段事务功能，方法是在`sessionmaker`或`Session`中设置`Session.use_twophase`标志。参见启用两阶段提交以获取示例。

### 自定义垂直分区

可以通过重写`Session.get_bind()`方法来构建更全面的基于规则的类级分区。以下是一个自定义`Session`的示例，提供以下规则：

1.  刷新操作以及批量“更新”和“删除”操作将传送到名为`leader`的引擎。

1.  所有子类化`MyOtherClass`的对象操作都发生在`other`引擎上。

1.  所有其他类的读取操作都在`follower1`或`follower2`数据库的随机选择上进行。

```py
engines = {
    "leader": create_engine("sqlite:///leader.db"),
    "other": create_engine("sqlite:///other.db"),
    "follower1": create_engine("sqlite:///follower1.db"),
    "follower2": create_engine("sqlite:///follower2.db"),
}

from sqlalchemy.sql import Update, Delete
from sqlalchemy.orm import Session, sessionmaker
import random

class RoutingSession(Session):
    def get_bind(self, mapper=None, clause=None):
        if mapper and issubclass(mapper.class_, MyOtherClass):
            return engines["other"]
        elif self._flushing or isinstance(clause, (Update, Delete)):
            # NOTE: this is for example, however in practice reader/writer
            # splits are likely more straightforward by using two distinct
            # Sessions at the top of a "reader" or "writer" operation.
            # See note below
            return engines["leader"]
        else:
            return engines[random.choice(["follower1", "follower2"])]
```

上述`Session`类是通过向`sessionmaker`传递`class_`参数来插入的：

```py
Session = sessionmaker(class_=RoutingSession)
```

这种方法可以与多个`MetaData`对象结合使用，例如使用声明性的`__abstract__`关键字的方法，如在 __abstract__ 中所述。

注意

尽管上面的示例说明了将特定的 SQL 语句路由到基于语句是否期望写入数据的所谓 “leader” 或 “follower” 数据库，但这可能不是一种实际的方法，因为它导致在同一操作中读取和写入之间的不协调事务行为。实际上，最好是根据正在进行的整体操作 / 事务，提前将 `Session` 构造为 “读取者” 或 “写入者” 会话。这样，将要写入数据的操作也会在同一个事务范围内发出其读取查询。请参阅 为 Sessionmaker / Engine 设置隔离 中的示例，该示例设置了一个用于 “只读” 操作的 `sessionmaker` ，使用自动提交连接，另一个用于包含 DML / COMMIT 的 “写入” 操作。

另请参阅

[SQLAlchemy 中的 Django 风格数据库路由器](https://techspot.zzzeek.org/2012/01/11/django-style-database-routers-in-sqlalchemy/) - 有关 `Session.get_bind()` 的更全面示例的博客文章

### 水平分区

水平分区将单个表（或一组表）的行跨多个数据库进行分区。SQLAlchemy `Session` 包含对此概念的支持，但要充分利用它，需要使用 `Session` 和 `Query` 的子类。这些子类的基本版本在 水平分片 ORM 扩展中可用。使用示例位于：水平分片。

## 批量操作

遗留特性

SQLAlchemy 2.0 将 `Session` 的“批量插入”和“批量更新”功能集成到了 2.0 风格的 `Session.execute()` 方法中，直接使用了 `Insert` 和 `Update` 构造。请参阅 ORM 启用的 INSERT、UPDATE 和 DELETE 语句 文档，包括 遗留 Session 批量 INSERT 方法 ，其中说明了从旧方法迁移到新方法的示例。
