# 类型层次结构

> 原文：[`docs.sqlalchemy.org/en/20/core/type_basics.html`](https://docs.sqlalchemy.org/en/20/core/type_basics.html)

SQLAlchemy 提供了对大多数常见数据库数据类型的抽象，以及多种自定义数据类型的技术。

数据库类型使用 Python 类表示，所有这些类最终都是从名为`TypeEngine`的基本类型类扩展而来。有两种一般类别的数据类型，它们在类型层次结构中以不同的方式表达自己。根据两种不同的命名约定，即“驼峰命名法”和“大写字母”，可以识别个别数据类型类使用的类别。

另请参见

使用表对象设置 MetaData - 在 SQLAlchemy Unified Tutorial 中。以教程形式演示了使用`TypeEngine`类型对象定义`Table`元数据并介绍了类型对象的概念的最基本用法。

## “驼峰命名法”数据类型

初级类型的命名采用“驼峰命名法”，如`String`、`Numeric`、`Integer`和`DateTime`。所有`TypeEngine`的直接子类都是“驼峰命名法”类型。尽可能“驼峰命名法”类型是**与数据库无关的**，这意味着它们可以在任何数据库后端上使用，在这些后端上，它们将以适合该后端的方式行事，以产生所需的行为。

简单的“驼峰命名法”数据类型示例是`String`。在大多数后端上，将此数据类型用于 table specification 将对应于在目标后端上使用`VARCHAR`数据库类型，从而在数据库和数据库之间传递字符串值，如下例所示：

```py
from sqlalchemy import MetaData
from sqlalchemy import Table, Column, Integer, String

metadata_obj = MetaData()

user = Table(
    "user",
    metadata_obj,
    Column("user_name", String, primary_key=True),
    Column("email_address", String(60)),
)
```

在表定义或整体 SQL 表达式中使用特定的`TypeEngine`类时，如果不需要参数，则可以将其作为类本身传递，即不需要实例化它，例如上面的`"email_address"`列中的长度参数 60。如果需要参数，则可以将类型实例化。

另一个表达更具后端特定行为的“驼峰命名法”数据类型是`Boolean`数据类型。与`String`表示所有数据库都具有的字符串数据类型不同，不是每个后端都有真正的“布尔”数据类型；一些后端使用整数或比特值 0 和 1，一些具有布尔字面常量`true`和`false`，而另一些则没有。对于此数据类型，`Boolean`在后端（如 PostgreSQL）上可能会呈现为`BOOLEAN`，在 MySQL 后端上可能为`BIT`，在 Oracle 上可能为`SMALLINT`。当使用此类型发送和接收数据到数据库时，根据正在使用的方言，它可能会解释 Python 数字或布尔值。

典型的 SQLAlchemy 应用程序通常会在一般情况下主要使用“驼峰命名法”类型，因为它们通常提供最佳的基本行为，并且可以自动地在所有后端上移植。

通用“驼峰命名法”数据类型的参考资料请参见通用“驼峰命名法”类型。

## “大写字母”数据类型

与“驼峰命名法”类型相反的是“大写字母”数据类型。这些数据类型总是从特定的“驼峰命名法”数据类型继承，并且始终表示**确切**的数据类型。当使用“大写字母”数据类型时，类型的名称始终精确地呈现，而不考虑当前后端是否支持它。因此，在 SQLAlchemy 应用程序中使用“大写字母”类型表示需要特定数据类型，这随后意味着应用程序通常（如果没有采取额外步骤）会受限于那些完全使用给定类型的后端。大写字母类型的示例包括`VARCHAR`、`NUMERIC`、`INTEGER`和`TIMESTAMP`，它们直接继承自前面提到的“驼峰命名法”类型`String`、`Numeric`、`Integer`和`DateTime`。

作为`sqlalchemy.types`的一部分的“大写字母”数据类型是通用 SQL 类型，通常期望在至少两个后端上可用。

通用“大写字母”数据类型的参考资料请参见 SQL 标准和多供应商“大写字母”类型。

## 后端特定的“大写字母”数据类型

大多数数据库还具有完全特定于这些数据库的数据类型，或者添加了特定于这些数据库的附加参数。对于这些数据类型，特定的 SQLAlchemy 方言提供了**后端特定**的“大写”数据类型，用于在其他后端上没有类似物的 SQL 类型。后端特定大写数据类型的示例包括 PostgreSQL 的`JSONB`、SQL Server 的`IMAGE`和 MySQL 的`TINYTEXT`。

某些后端还可能包括“大写”数据类型，这些数据类型扩展了来自`sqlalchemy.types`模块中相同“大写”数据类型的参数。例如，当创建 MySQL 字符串数据类型时，可能希望指定 MySQL 特定参数，如`charset`或`national`，这些参数可以从 MySQL 版本的`VARCHAR`作为仅 MySQL 参数`VARCHAR.charset`和`VARCHAR.national`。

后端特定类型的 API 文档在方言特定文档中列出，详见方言。## 使用“大写”和后端特定类型用于多个后端

检查“大写”和“驼峰”类型的存在自然会引出如何在使用特定后端时利用“大写”数据类型的自然用例，但仅当该后端正在使用时。为了将数据库无关的“驼峰”和特定后端的“大写”系统联系在一起，可以使用`TypeEngine.with_variant()`方法将类型组合在一起，以在特定后端上使用特定行为。

例如，要使用`String`数据类型，但在运行时在 MySQL 上使用`VARCHAR.charset`参数的`VARCHAR`创建表时，可以使用`TypeEngine.with_variant()`如下所示：

```py
from sqlalchemy import MetaData
from sqlalchemy import Table, Column, Integer, String
from sqlalchemy.dialects.mysql import VARCHAR

metadata_obj = MetaData()

user = Table(
    "user",
    metadata_obj,
    Column("user_name", String(100), primary_key=True),
    Column(
        "bio",
        String(255).with_variant(VARCHAR(255, charset="utf8"), "mysql", "mariadb"),
    ),
)
```

在上面的表定义中，`"bio"`列将在所有后端上具有字符串行为。在大多数后端上，它将在 DDL 中呈现为`VARCHAR`。然而，在 MySQL 和 MariaDB（通过以`mysql`或`mariadb`开头的数据库 URL 表示）上，它将呈现为`VARCHAR(255) CHARACTER SET utf8`。

另请参阅

`TypeEngine.with_variant()` - 附加用法示例和注意事项  ## 通用“驼峰大小写”类型

通用类型指定一个列，该列可以读取、写入和存储特定类型的 Python 数据。当发出`CREATE TABLE`语句时，SQLAlchemy 将选择目标数据库上可用的最佳数据库列类型。对于完全控制在`CREATE TABLE`中发出的列类型，比如`VARCHAR`，请参见 SQL 标准和多个供应商的“大写”类型和本章的其他部分。

| 对象名称 | 描述 |
| --- | --- |
| BigInteger | 用于较大的`int`整数的类型。 |
| Boolean | 一个布尔数据类型。 |
| Date | 用于`datetime.date()`对象的类型。 |
| DateTime | 用于`datetime.datetime()`对象的类型。 |
| Double | 用于双精度`FLOAT`浮点类型的类型。 |
| Enum | 通用枚举类型。 |
| Float | 代表浮点类型，如`FLOAT`或`REAL`的类型。 |
| Integer | 一个`int`整数的类型。 |
| Interval | 用于`datetime.timedelta()`对象的类型。 |
| LargeBinary | 用于大型二进制字节数据的类型。 |
| MatchType | 指代 MATCH 运算符的返回类型。 |
| Numeric | 非整数数字类型的基类，如`NUMERIC`、`FLOAT`、`DECIMAL`和其他变体。 |
| PickleType | 包含使用 pickle 序列化的 Python 对象。 |
| SchemaType | 添加了允许将模式级 DDL 与类型关联的类型的功能。 |
| SmallInteger | 用于较小的`int`整数的类型。 |
| String | 所有字符串和字符类型的基类。 |
| Text | 可变大小的字符串类型。 |
| Time | 用于`datetime.time()`对象的类型。 |
| Unicode | 一个可变长度的 Unicode 字符串类型。 |
| UnicodeText | 一个无限长度的 Unicode 字符串类型。 |
| Uuid | 表示与数据库无关的 UUID 数据类型。 |

```py
class sqlalchemy.types.BigInteger
```

用于较大的`int`整数的类型。

在 DDL 中通常生成 `BIGINT`，在 Python 端则像正常的 `Integer` 一样操作。

**类签名**

class `sqlalchemy.types.BigInteger` (`sqlalchemy.types.Integer`)

```py
class sqlalchemy.types.Boolean
```

布尔数据类型。

`Boolean` 通常在 DDL 方面使用 BOOLEAN 或 SMALLINT，在 Python 方面处理 `True` 或 `False`。

`Boolean` 数据类型目前有两个断言级别，用于确保持久化的值是简单的 true/false 值。对于所有后端，仅接受 Python 值 `None`、`True`、`False`、`1` 或 `0` 作为参数值。对于不支持“本地布尔”数据类型的后端，还可以在目标列上创建 CHECK 约束的选项

1.2 版本中的变化：`Boolean` 数据类型现在断言传入的 Python 值已经是纯布尔形式。

**成员**

__init__(), bind_processor(), literal_processor(), python_type, result_processor()

**类签名**

class `sqlalchemy.types.Boolean` (`sqlalchemy.types.SchemaType`, `sqlalchemy.types.Emulated`, `sqlalchemy.types.TypeEngine`)

```py
method __init__(create_constraint: bool = False, name: str | None = None, _create_events: bool = True, _adapted_from: SchemaType | None = None)
```

构造一个布尔值。

参数：

+   `create_constraint` –

    默认为 False。如果布尔值生成为 int/smallint，还会在表上创建一个 CHECK 约束，以确保值为 1 或 0。

    注意

    强烈建议 CHECK 约束具有明确的名称，以支持模式管理方面的关注。可以通过设置 `Boolean.name` 参数或设置适当的命名约定来实现；有关背景信息，请参阅 配置约束命名约定。

    1.4 版本中的变化：- 此标志现在默认为 False，表示非本机枚举类型不会生成 CHECK 约束。

+   `name` – 如果生成了 CHECK 约束，请指定约束的名称。

```py
method bind_processor(dialect)
```

返回用于处理绑定值的转换函数。

返回一个可调用对象，该对象将接收一个绑定参数值作为唯一位置参数，并返回一个要发送到 DB-API 的值。

如果不需要处理，方法应返回 `None`。

注意

此方法仅相对于**方言特定类型对象**调用，该对象通常**是方言中使用的私有对象**，并且不是公共类型对象，这意味着无法通过子类化`TypeEngine`类来提供备用的`TypeEngine.bind_processor()`方法，除非明确地子类化`UserDefinedType`类。

为了为`TypeEngine.bind_processor()`提供备用行为，实现一个`TypeDecorator`类，并提供一个`TypeDecorator.process_bind_param()`的实现。

另见

增强现有类型

参数：

**dialect** – 正在使用的方言实例。

```py
method literal_processor(dialect)
```

返回一个转换函数，用于处理要直接呈现而不使用绑定的文字值。

当编译器使用“literal_binds”标志时，通常在 DDL 生成以及某些后端不接受绑定参数的情况下使用此函数。

返回一个可调用对象，它将接收一个字面 Python 值作为唯一的位置参数，并返回一个字符串表示以在 SQL 语句中呈现。

注意

此方法仅相对于**方言特定类型对象**调用，该对象通常**是方言中使用的私有对象**，并且不是公共类型对象，这意味着无法通过子类化`TypeEngine`类来提供备用的`TypeEngine.literal_processor()`方法，除非明确地子类化`UserDefinedType`类。

为了为`TypeEngine.literal_processor()`提供备用行为，实现一个`TypeDecorator`类，并提供一个`TypeDecorator.process_literal_param()`的实现。

另见

增强现有类型

```py
attribute python_type
```

```py
method result_processor(dialect, coltype)
```

返回一个用于处理结果行值的转换函数。

返回一个可调用对象，该对象将接收一个结果行列值作为唯一的位置参数，并将返回一个要返回给用户的值。

如果不需要处理，则方法应返回`None`。

注意

此方法仅相对于**特定方言类型对象**调用，该对象通常**私有于使用的方言**，并且不是与公共类型对象相同的类型对象，这意味着不可能通过继承`TypeEngine`类来提供替代的`TypeEngine.result_processor()`方法，除非显式地继承`UserDefinedType`类。

要为`TypeEngine.result_processor()`提供替代行为，需实现一个`TypeDecorator`类，并提供一个`TypeDecorator.process_result_value()`的实现。

另请参见

扩充现有类型

参数：

+   `dialect` – 正在使用的方言实例。

+   `coltype` – 在 cursor.description 中接收到的 DBAPI coltype 参数。

```py
class sqlalchemy.types.Date
```

用于`datetime.date()`对象的类型。

**成员**

get_dbapi_type()，literal_processor()，python_type

**类签名**

类`sqlalchemy.types.Date`（`sqlalchemy.types._RenderISO8601NoT`，`sqlalchemy.types.HasExpressionLookup`，`sqlalchemy.types.TypeEngine`）

```py
method get_dbapi_type(dbapi)
```

返回底层 DB-API 中相应的类型对象（如果有）。

例如，这对于调用`setinputsizes()`可能很有用。

```py
method literal_processor(dialect)
```

返回一个用于处理直接呈现而不使用绑定的文字值的转换函数。

当编译器使用“literal_binds”标志时，通常用于 DDL 生成以及某些情况下后端不接受绑定参数时，将使用此函数。

返回一个可调用对象，该对象将接收一个文字 Python 值作为唯一的位置参数，并将返回一个字符串表示以在 SQL 语句中呈现。

注意

这个方法仅在**特定方言的类型对象**相关时才被调用，这通常是**私有于正在使用的方言**的，并且不是公共类型对象，这意味着无法通过子类化`TypeEngine`类来提供替代的`TypeEngine.literal_processor()`方法，除非明确地子类化`UserDefinedType`类。

要为 `TypeEngine.literal_processor()` 提供替代行为，请实现一个 `TypeDecorator` 类，并提供 `TypeDecorator.process_literal_param()` 的实现。

另请参阅

扩展现有类型

```py
attribute python_type
```

```py
class sqlalchemy.types.DateTime
```

一个用于`datetime.datetime()`对象的类型。

日期和时间类型返回来自 Python `datetime` 模块的对象。大多数 DBAPI 都内置支持 datetime 模块，但 SQLite 是个例外。在 SQLite 的情况下，日期和时间类型存储为字符串，然后在返回行时将其转换回 datetime 对象。

在 datetime 类型内的时间表示中，一些后端包括其他选项，例如时区支持和分数秒支持。对于分数秒，使用特定于方言的数据类型，例如`TIME`。对于时区支持，至少使用`TIMESTAMP`数据类型，如果不是特定于方言的数据类型对象。

**成员**

__init__(), get_dbapi_type(), literal_processor(), python_type

**类签名**

类 `sqlalchemy.types.DateTime` (`sqlalchemy.types._RenderISO8601NoT`, `sqlalchemy.types.HasExpressionLookup`, `sqlalchemy.types.TypeEngine`)

```py
method __init__(timezone: bool = False)
```

构造一个新的 `DateTime`。

参数：

**timezone** – 布尔值。指示日期时间类型是否应启用时区支持，如果仅在**基本日期/时间保存类型上可用**。建议在使用此标志时直接使用`TIMESTAMP` 数据类型，因为一些数据库包括与支持时区的 TIMESTAMP 数据类型不同的单独的通用日期/时间保存类型，如 Oracle。

```py
method get_dbapi_type(dbapi)
```

如果有的话，返回底层 DB-API 的相应类型对象。

这对于调用`setinputsizes()`可能很有用。

```py
method literal_processor(dialect)
```

返回一个转换函数，用于处理直接呈现而不使用绑定的字面值。

当编译器使用“literal_binds”标志时，通常用于 DDL 生成以及在某些后端不接受绑定参数的情况下使用此函数。

返回一个可调用对象，该对象将接收一个字面的 Python 值作为唯一的位置参数，并返回一个字符串表示，以在 SQL 语句中呈现。

注意

此方法仅相对于**特定方言类型对象**调用，该对象通常**私有于正在使用的方言**，并且不是公共类型对象，这意味着不可通过子类化`TypeEngine` 类来提供替代的`TypeEngine.literal_processor()` 方法，除非显式地子类化`UserDefinedType` 类。

要为`TypeEngine.literal_processor()` 提供替代行为，请实现一个`TypeDecorator` 类，并提供一个`TypeDecorator.process_literal_param()` 的实现。

另请参阅

扩展现有类型

```py
attribute python_type
```

```py
class sqlalchemy.types.Enum
```

通用枚举类型。

`Enum` 类型提供了一组可能的字符串值，列受其约束。

如果可用，`Enum` 类型将使用后端的本机“ENUM”类型；否则，它使用 VARCHAR 数据类型。还存在一个选项，即在生成 VARCHAR（所谓的“非本机”）变体时自动生成 CHECK 约束；请参阅`Enum.create_constraint` 标志。

`Enum` 类型在 Python 中也提供了对字符串值进行读写操作期间的验证。从结果集中读取数据库中的值时，始终检查字符串值是否与可能值列表匹配，如果找不到匹配项，则引发 `LookupError`。在将值作为 SQL 语句中的纯字符串传递给数据库时，如果 `Enum.validate_strings` 参数设置为 True，则对于未位于给定可能值列表中的任何字符串值，都会引发 `LookupError`；请注意，这会影响到具有枚举值的 LIKE 表达式的使用（这是一个不寻常的用例）。

枚举值的来源可以是字符串值列表，或者是符合 PEP-435 的枚举类。对于 `Enum` 数据类型，此类只需要提供一个 `__members__` 方法即可。

当使用枚举类时，枚举对象用于输入和输出，而不是字符串，就像普通字符串枚举类型一样：

```py
import enum
from sqlalchemy import Enum

class MyEnum(enum.Enum):
    one = 1
    two = 2
    three = 3

t = Table(
    'data', MetaData(),
    Column('value', Enum(MyEnum))
)

connection.execute(t.insert(), {"value": MyEnum.two})
assert connection.scalar(t.select()) is MyEnum.two
```

在上面，每个元素的字符串名称（例如“one”、“two”、“three”）都会被持久化到数据库中；Python 枚举的值，这里表示为整数，**不会**被使用；因此，每个枚举的值可以是任何类型的 Python 对象，无论它是否可持久化。

为了持久化值而不是名称，可以使用 `Enum.values_callable` 参数。该参数的值是一个用户提供的可调用对象，旨在与符合 PEP-435 的枚举类一起使用，并返回要持久化的字符串值列表。对于使用字符串值的简单枚举，像 `lambda x: [e.value for e in x]` 这样的可调用对象就足够了。

另见

在类型映射中使用 Python 枚举或 pep-586 Literal 类型 - 使用 ORM 的 ORM 注释声明 功能时，关于在类型映射中使用 `Enum` 数据类型的背景信息。

`ENUM` - PostgreSQL 特定类型，具有额外的功能。

`ENUM` - MySQL 特定类型

**成员**

__init__(), create(), drop()

**类签名**

类`sqlalchemy.types.Enum`（`sqlalchemy.types.String`，`sqlalchemy.types.SchemaType`，`sqlalchemy.types.Emulated`，`sqlalchemy.types.TypeEngine`）

```py
method __init__(*enums: object, **kw: Any)
```

构造枚举。

不适用于特定后端的关键字参数将被该后端忽略。

参数：

+   `*enums` – 要么正好一个符合 PEP-435 标准的枚举类型，要么一个或多个字符串标签。

+   `create_constraint` –

    默认为 False。创建非本地枚举类型时，还在数据库上构建 CHECK 约束以针对有效值。

    注意

    强烈建议为 CHECK 约束指定显式名称，以支持模式管理方面的问题。这可以通过设置 `Enum.name` 参数或设置适当的命名约定来实现；有关背景，请参见配置约束命名约定。

    自版本 1.4 更改： - 此标志现在默认为 False，意味着对非本地枚举类型不会生成 CHECK 约束。

+   `metadata` –

    将此类型直接关联到 `MetaData` 对象。对于作为独立模式构造存在于目标数据库上的类型（如 PostgreSQL），此类型将在 `create_all()` 和 `drop_all()` 操作中创建和删除。如果该类型未与任何 `MetaData` 对象相关联，则它将与使用它的每个 `Table` 相关联，并且将在创建任何这些单独表时创建，并在检查其存在后创建。但是，仅在对该 `Table` 对象的元数据调用 `drop_all()` 时才会删除该类型。

    如果设置了`MetaData` 对象的 `MetaData.schema` 参数的值，并且未显式提供值，则将使用该对象上的 `Enum.schema` 的默认值。

    自版本 1.4.12 更改：`Enum` 如果传递使用`Enum.metadata` 参数时继承 `MetaData` 对象的`MetaData.schema` 参数（如果存在）。

+   `name` – 此类型的名称。这对于 PostgreSQL 和任何将来需要显式命名类型或显式命名约束以生成使用它的类型和/或表的支持数据库是必需的。如果使用了 PEP-435 枚举类，则默认情况下使用其名称（转换为小写）。

+   `native_enum` – 在可用时使用数据库的本机 ENUM 类型。默认为 True。当为 False 时，对所有后端使用 VARCHAR + 检查约束。当为 False 时，如果 native_enum=True，则“length”将被忽略。

+   `length` –

    允许在使用非本机枚举数据类型时为 VARCHAR 指定自定义长度。默认情况下，它使用最长值的长度。

    从版本 2.0.0 开始更改：无条件地使用 `Enum.length` 参数进行 `VARCHAR` 渲染，而不管 `Enum.native_enum` 参数的设置情况，对于那些使用 `VARCHAR` 作为枚举数据类型的后端。

+   `schema` –

    此类型的模式名称。对于作为独立模式构造存在于目标数据库上的类型（PostgreSQL），此参数指定了类型存在的命名模式。

    如果不存在，则在传递为 `Enum.metadata` 的 `MetaData` 中获取模式名称，对于包含 `MetaData.schema` 参数的 `MetaData`。

    从版本 1.4.12 开始更改：如果使用 `Enum.metadata` 参数传递，则 `Enum` 继承了 `MetaData` 对象的 `MetaData.schema` 参数（如果存在）。

    否则，如果`Enum.inherit_schema`标志设置为`True`，则模式将从相关的`Table`对象继承，如果有的话；当`Enum.inherit_schema`处于默认值`False`时，不使用所属表的模式。

+   `quote` – 为类型的名称设置明确的引用首选项。

+   `inherit_schema` – 当为 `True` 时，将从所属 `Table` 的`schema`复制到此 `Enum` 的`schema`属性中，替换传递给 `schema` 属性的任何值。这也会在使用 `Table.to_metadata()` 操作时生效。

+   `validate_strings` – 当为 True 时，将传递给 SQL 语句的字符串值将被检查是否有效。未识别的值将引发 `LookupError`。

+   `values_callable` –

    一个可调用对象，将传递符合 PEP-435 的枚举类型，然后应返回要持久化的字符串值列表。这允许替代用法，例如使用枚举的字符串值而不是其名称持久化到数据库中。可调用对象必须以与迭代枚举的 `__member__` 属性相同的顺序返回要持久化的值。例如 `lambda x: [i.value for i in x]`。

    从版本 1.2.3 起新增。

+   `sort_key_function` –

    可以作为 Python `sorted()` 内置函数中的“key”参数使用的 Python 可调用对象。SQLAlchemy ORM 要求映射的主键列必须以某种方式可排序。当使用不可排序的枚举对象，如 Python 3 的 `Enum` 对象时，可以使用此参数为对象设置默认的排序键函数。默认情况下，枚举的数据库值被用作排序函数。

    从版本 1.3.8 起新增。

+   `omit_aliases` –

    当为 true 时，将从 pep 435 枚举中删除别名的布尔值。默认为 `True`。

    自版本 2.0 起更改：此参数现在默认为 True。

```py
method create(bind, checkfirst=False)
```

*继承自* `SchemaType.create()` *方法的* `SchemaType`

如适用，为此类型发出 CREATE DDL。

```py
method drop(bind, checkfirst=False)
```

*继承自* `SchemaType.drop()` *方法的* `SchemaType`

如适用，为此类型发出 DROP DDL。

```py
class sqlalchemy.types.Double
```

用于双 `FLOAT` 浮点类型的类型。

通常在 DDL 中生成 `DOUBLE` 或 `DOUBLE_PRECISION`，否则在 Python 方面的行为类似于普通的 `Float`。

从版本 2.0 起新增。

**类签名**

类 `sqlalchemy.types.Double` (`sqlalchemy.types.Float`)

```py
class sqlalchemy.types.Float
```

表示浮点类型的类型，例如 `FLOAT` 或 `REAL`。

默认情况下，此类型返回 Python `float`对象，除非将`Float.asdecimal`标志设置为`True`，在这种情况下，它们将被强制转换为`decimal.Decimal`对象。

当`Float.precision`在`Float`类型中未提供时，某些后端可能将此类型编译为 8 字节/64 位浮点数据类型。要使用 4 字节/32 位浮点数据类型，通常可以提供精度<= 24，或者可以使用`REAL`类型。已知在将类型呈现为`FLOAT`的 PostgreSQL 和 MSSQL 方言中，这种情况是成立的，这两者都是`DOUBLE PRECISION`的别名。其他第三方方言可能具有类似的行为。

**成员**

__init__(), result_processor()

**类签名**

类`sqlalchemy.types.Float`（`sqlalchemy.types.Numeric`）

```py
method __init__(precision: int | None = None, asdecimal: bool = False, decimal_return_scale: int | None = None)
```

构造一个浮点数。

参数：

+   `precision` –

    用于在 DDL `CREATE TABLE`中使用的数字精度。后端**应该**尝试确保此精度指示了通用`Float`数据类型的数字位数。

    注意

    对于 Oracle 后端，在渲染 DDL 时，不接受`Float.precision`参数，因为 Oracle 不支持将浮点精度指定为小数位数。相反，请使用特定于 Oracle 的`FLOAT`数据类型，并指定`FLOAT.binary_precision`参数。这是 SQLAlchemy 2.0 版本中的新功能。

    要创建一个数据库通用的`Float`，并为 Oracle 单独指定二进制精度，请使用`TypeEngine.with_variant()`如下所示：

    ```py
    from sqlalchemy import Column
    from sqlalchemy import Float
    from sqlalchemy.dialects import oracle

    Column(
        "float_data",
        Float(5).with_variant(oracle.FLOAT(binary_precision=16), "oracle")
    )
    ```

+   `asdecimal` – 与`Numeric`相同的标志，但默认为`False`。请注意，将此标志设置为`True`会导致浮点数转换。

+   `decimal_return_scale` – 在将浮点数转换为 Python 小数时使用的默认精度。由于十进制的不准确性，浮点值通常会更长，而大多数浮点数据库类型没有“精度”的概念，因此默认情况下，当转换时，浮点类型将查找前十位小数。指定此值将覆盖该长度。请注意，如果未另行指定，则 MySQL 浮点类型将使用“精度”作为`decimal_return_scale`的默认值。

```py
method result_processor(dialect, coltype)
```

返回一个用于处理结果行值的转换函数。

返回一个可调用对象，该对象将接收一个结果行列值作为唯一的位置参数，并将返回一个要返回给用户的值。

如果不需要处理，该方法应返回`None`。

注意

此方法仅相对于一个**方言特定类型对象**调用，该对象通常是**正在使用的方言中的私有对象**，并且不是与公共对象相同的类型对象，这意味着无法子类化`TypeEngine`类以提供替代`TypeEngine.result_processor()`方法，除非明确子类化`UserDefinedType`类。

要为`TypeEngine.result_processor()`提供替代行为，请实现`TypeDecorator`类，并提供`TypeDecorator.process_result_value()`的实现。

另请参阅

扩充现有类型

参数：

+   `方言` – 正在使用的方言实例。

+   `coltype` – 在 cursor.description 中接收的 DBAPI coltype 参数。

```py
class sqlalchemy.types.Integer
```

一种适用于`int`整数的类型。

**成员**

get_dbapi_type()，literal_processor()，python_type

**类签名**

类`sqlalchemy.types.Integer` (`sqlalchemy.types.HasExpressionLookup`, `sqlalchemy.types.TypeEngine`)

```py
method get_dbapi_type(dbapi)
```

如果有的话，返回底层 DB-API 的相应类型对象。

例如，这对于调用`setinputsizes()`可能很有用。

```py
method literal_processor(dialect)
```

返回一个用于处理直接呈现的字面值的转换函数，而无需使用绑定。

此函数在编译器使用 “literal_binds” 标志时使用，通常用于 DDL 生成以及在某些后端不接受绑定参数的情况下。

返回一个可调用对象，该对象将接收一个字面的 Python 值作为唯一的位置参数，并返回一个字符串表示以在 SQL 语句中呈现。

注意

此方法仅相对于 **特定方言类型对象** 调用，该对象通常是 **特定方言私有的**，并且与公共类型对象不同，这意味着不可能通过子类化 `TypeEngine` 类来提供替代的 `TypeEngine.literal_processor()` 方法，除非显式子类化 `UserDefinedType` 类。

要为 `TypeEngine.literal_processor()` 提供替代行为，请实现一个 `TypeDecorator` 类，并提供 `TypeDecorator.process_literal_param()` 的实现。

另请参阅

扩展现有类型

```py
attribute python_type
```

```py
class sqlalchemy.types.Interval
```

用于 `datetime.timedelta()` 对象的类型。

Interval 类型处理 `datetime.timedelta` 对象。在 PostgreSQL 和 Oracle 中，使用原生的 `INTERVAL` 类型；对于其他数据库，该值存储为相对于“epoch”（1970 年 1 月 1 日）的日期。

请注意，`Interval` 类型当前在不原生支持间隔类型的平台上不提供日期算术操作。这些操作通常需要对表达式的两侧进行转换（例如，首先将两侧转换为整数时期值），目前这是一个手动过程（例如，通过 `expression.func`）。

**成员**

__init__(), adapt_to_emulated(), bind_processor(), cache_ok, coerce_compared_value(), comparator_factory, impl, python_type, result_processor()

**类签名**

类`sqlalchemy.types.Interval` (`sqlalchemy.types.Emulated`, `sqlalchemy.types._AbstractInterval`, `sqlalchemy.types.TypeDecorator`)

```py
class Comparator
```

**类签名**

类`sqlalchemy.types.Interval.Comparator` (`sqlalchemy.types.Comparator`, `sqlalchemy.types.Comparator`)

```py
method __init__(native: bool = True, second_precision: int | None = None, day_precision: int | None = None)
```

构造一个 Interval 对象。

参数：

+   `native` – 当为 True 时，如果支持（目前是 PostgreSQL、Oracle），则使用数据库提供的实际 INTERVAL 类型。否则，无论如何都将间隔数据表示为时代值。

+   `second_precision` – 对于支持“分秒精度”参数的本机间隔类型，例如 Oracle 和 PostgreSQL

+   `day_precision` – 对于支持“天精度”参数的本机间隔类型，例如 Oracle。

```py
method adapt_to_emulated(impltype, **kw)
```

给定一个 impl 类，将此类型适配到 impl，假设“模拟”。

impl 还应该是此类型的“模拟”版本，很可能是与此类型本身相同的类。

例如：sqltypes.Enum 适应于 Enum 类。

```py
method bind_processor(dialect: Dialect) → _BindProcessorType[dt.timedelta]
```

返回一个用于处理绑定值的转换函数。

返回一个可调用对象，该对象将接收绑定参数值作为唯一的位置参数，并返回要发送到 DB-API 的值。

如果不需要处理，则该方法应返回`None`。

注

此方法仅相对于**特定方言类型对象**调用，该对象通常**是正在使用的方言的私有对象**，并且不是公共类型对象，这意味着无法通过子类化`TypeEngine`类来提供替代的`TypeEngine.bind_processor()`方法，除非显式地对`UserDefinedType`类进行子类化。

要为`TypeEngine.bind_processor()`提供替代行为，请实现一个`TypeDecorator`类，并提供`TypeDecorator.process_bind_param()`的实现。

另请参见

扩展现有类型

参数：

**方言** – 正在使用的方言实例。

```py
attribute cache_ok: bool | None = True
```

指示使用此`ExternalType`的语句是否“安全缓存”。

默认值`None`会发出警告，然后不允许缓存包含此类型的语句。设置为`False`以禁用完全不带警告缓存使用此类型的语句。当设置为`True`时，将使用对象的类和其状态的选定元素作为缓存键的一部分。例如，使用`TypeDecorator`:

```py
class MyType(TypeDecorator):
    impl = String

    cache_ok = True

    def __init__(self, choices):
        self.choices = tuple(choices)
        self.internal_only = True
```

上述类型的缓存键将等同于：

```py
>>> MyType(["a", "b", "c"])._static_cache_key
(<class '__main__.MyType'>, ('choices', ('a', 'b', 'c')))
```

缓存方案将从类型中提取与`__init__()`方法中参数名称对应的属性。上面的“choices”属性成为缓存键的一部分，但“internal_only”不会，因为没有名为“internal_only”的参数。

可缓存元素的要求是它们是可哈希的，并且它们指示对于给定缓存值的表达式每次返回相同的 SQL 渲染。

为了适应引用不可哈希结构（如字典、集合和列表）的数据类型，这些对象可以通过将可哈希结构分配给与参数名称对应的属性来“可缓存”。例如，一个接受查找值字典的数据类型可以将其发布为排序后的元组序列。假设以前不可缓存的类型如下：

```py
class LookupType(UserDefinedType):
  '''a custom type that accepts a dictionary as a parameter.

 this is the non-cacheable version, as "self.lookup" is not
 hashable.

 '''

    def __init__(self, lookup):
        self.lookup = lookup

    def get_col_spec(self, **kw):
        return "VARCHAR(255)"

    def bind_processor(self, dialect):
        # ...  works with "self.lookup" ...
```

其中“lookup”是一个字典。该类型将无法生成缓存键：

```py
>>> type_ = LookupType({"a": 10, "b": 20})
>>> type_._static_cache_key
<stdin>:1: SAWarning: UserDefinedType LookupType({'a': 10, 'b': 20}) will not
produce a cache key because the ``cache_ok`` flag is not set to True.
Set this flag to True if this type object's state is safe to use
in a cache key, or False to disable this warning.
symbol('no_cache')
```

如果我们**确实**设置了这样一个缓存键，它将无法使用。我们将得到一个包含字典的元组结构，该字典本身不能用作“缓存字典”中的键，例如 SQLAlchemy 的语句缓存，因为 Python 字典不可哈希：

```py
>>> # set cache_ok = True
>>> type_.cache_ok = True

>>> # this is the cache key it would generate
>>> key = type_._static_cache_key
>>> key
(<class '__main__.LookupType'>, ('lookup', {'a': 10, 'b': 20}))

>>> # however this key is not hashable, will fail when used with
>>> # SQLAlchemy statement cache
>>> some_cache = {key: "some sql value"}
Traceback (most recent call last): File "<stdin>", line 1,
in <module> TypeError: unhashable type: 'dict'
```

通过将排序后的元组分配给“lookup”属性，可以使类型成为可缓存的：

```py
class LookupType(UserDefinedType):
  '''a custom type that accepts a dictionary as a parameter.

 The dictionary is stored both as itself in a private variable,
 and published in a public variable as a sorted tuple of tuples,
 which is hashable and will also return the same value for any
 two equivalent dictionaries.  Note it assumes the keys and
 values of the dictionary are themselves hashable.

 '''

    cache_ok = True

    def __init__(self, lookup):
        self._lookup = lookup

        # assume keys/values of "lookup" are hashable; otherwise
        # they would also need to be converted in some way here
        self.lookup = tuple(
            (key, lookup[key]) for key in sorted(lookup)
        )

    def get_col_spec(self, **kw):
        return "VARCHAR(255)"

    def bind_processor(self, dialect):
        # ...  works with "self._lookup" ...
```

在上述情况下，`LookupType({"a": 10, "b": 20})`的缓存键将是：

```py
>>> LookupType({"a": 10, "b": 20})._static_cache_key
(<class '__main__.LookupType'>, ('lookup', (('a', 10), ('b', 20))))
```

新版本 1.4.14 中新增：- 添加了 `cache_ok` 标志，以允许对 `TypeDecorator` 类进行某些缓存配置。

新版本 1.4.28 中新增：- 添加了 `ExternalType` 混合类，它将 `cache_ok` 标志推广到 `TypeDecorator` 和 `UserDefinedType` 类。

另请参阅

SQL 编译缓存

```py
method coerce_compared_value(op, value)
```

在表达式中为‘coerced’ Python 值建议一个类型。

给定一个运算符和值，使类型有机会返回一个值应该被强制转换为的类型。

这里的默认行为是保守的；如果右侧已根据其 Python 类型强制转换为 SQL 类型，则通常会保持不变。

此处的最终用户功能扩展通常应通过 `TypeDecorator` 实现，该实现具有更宽松的行为，因为它默认将表达式的另一侧强制转换为此类型，从而对除 DBAPI 需要的特殊 Python 转换之外的内容进行应用。它还提供了 `TypeDecorator.coerce_compared_value()` 的公共方法，该方法用于最终用户自定义此行为。

```py
attribute comparator_factory
```

别名 `Comparator`

```py
attribute impl
```

别名 `DateTime`

```py
attribute python_type
```

```py
method result_processor(dialect: Dialect, coltype: Any) → _ResultProcessorType[dt.timedelta]
```

返回一个用于处理结果行值的转换函数。

返回一个可调用对象，该对象将接收一个结果行列值作为唯一的位置参数，并将返回一个要返回给用户的值。

如果不需要处理，方法应返回`None`。

注

仅相对于**方言特定类型对象**调用此方法，该对象通常**私有于正在使用的方言**，并且与公共面向对象不同，这意味着无法通过子类化 `TypeEngine` 类来提供替代的 `TypeEngine.result_processor()` 方法，除非显式地子类化 `UserDefinedType` 类。

要为 `TypeEngine.result_processor()` 提供替代行为，实现一个 `TypeDecorator` 类并提供 `TypeDecorator.process_result_value()` 的实现。

另请参见

扩展现有类型

参数：

+   `dialect` – 正在使用的方言实例。

+   `coltype` – 在 cursor.description 中接收到的 DBAPI coltype 参数。

```py
class sqlalchemy.types.LargeBinary
```

用于大型二进制字节数据的类型。

`LargeBinary` 类型对应于目标平台的大型和/或无长度的二进制类型，例如 MySQL 上的 BLOB 和 PostgreSQL 上的 BYTEA。它还处理了 DBAPI 的必要转换。

**成员**

__init__()

**类签名**

类 `sqlalchemy.types.LargeBinary` (`sqlalchemy.types._Binary`)

```py
method __init__(length: int | None = None)
```

构造一个大型二进制类型。

参数：

**长度** – 可选，用于 DDL 语句的列长度，用于那些接受长度的二进制类型，例如 MySQL 的 BLOB 类型。

```py
class sqlalchemy.types.MatchType
```

指的是 MATCH 操作符的返回类型。

由于 `ColumnOperators.match()` 可能是 SQLAlchemy 核心中最开放的运算符，我们不能在 SQL 评估时假设返回类型，因为 MySQL 返回浮点数而不是布尔值，其他后端可能会执行不同的操作。因此，此类型充当占位符，目前是 `Boolean` 的子类。该类型允许方言在需要时注入结果处理功能，在 MySQL 上将返回浮点值。

**类签名**

类 `sqlalchemy.types.MatchType` (`sqlalchemy.types.Boolean`)

```py
class sqlalchemy.types.Numeric
```

非整数数字类型的基类，例如 `NUMERIC`、`FLOAT`、`DECIMAL` 和其他变体。

当直接使用 `Numeric` 数据类型时，如果可用，会呈现对应精度数字的 DDL，例如 `NUMERIC(precision, scale)`。当使用 `Float` 子类时，会尝试呈现浮点数据类型，例如 `FLOAT(precision)`。

`Numeric` 默认返回 Python `decimal.Decimal` 对象，基于 `Numeric.asdecimal` 参数的默认值 `True`。如果此参数设置为 False，则返回的值将被强制转换为 Python `float` 对象。

`Float` 子类型更具体于浮点数，默认情况下，`Float.asdecimal` 标志设置为 False，以便默认的 Python 数据类型为 `float`。

注意

当针对返回 Python 浮点值的数据库类型使用 `Numeric` 数据类型时，由 `Numeric.asdecimal` 指示的十进制转换的精度可能受到限制。具体数字/浮点数据类型的行为取决于正在使用的 SQL 数据类型、正在使用的 Python DBAPI 以及在使用的 SQLAlchemy 方言中可能存在的策略。鼓励需要特定精度/比例的用户尝试使用可用数据类型以确定最佳结果。

**成员**

__init__(), bind_processor(), get_dbapi_type(), literal_processor(), python_type, result_processor()

**类签名**

类 `sqlalchemy.types.Numeric` (`sqlalchemy.types.HasExpressionLookup`, `sqlalchemy.types.TypeEngine`)

```py
method __init__(precision: int | None = None, scale: int | None = None, decimal_return_scale: int | None = None, asdecimal: bool = True)
```

构造一个 Numeric。

参数：

+   `precision` – 用于 DDL `CREATE TABLE` 的数值精度。

+   `scale` – 用于 DDL `CREATE TABLE` 的数值比例。

+   `asdecimal` – 默认为 True。返回值是否应该作为 Python Decimal 对象发送，还是作为浮点数发送。不同的 DBAPI 根据数据类型发送其中之一 - Numeric 类型将确保跨 DBAPI 一致地返回值为其中之一。

+   `decimal_return_scale` – 在从浮点数转换为 Python 十进制数时使用的默认精度。由于十进制不精确，浮点值通常会更长，并且大多数浮点数据库类型都没有“精度”的概念，因此默认情况下，浮点类型在转换时会查找前十位小数。指定此值将覆盖该长度。包括显式“.scale”值的类型，例如基本 `Numeric` 和 MySQL 浮点类型，将使用“.scale”值作为默认的 decimal_return_scale，如果未另行指定。

使用 `Numeric` 类型时，应注意确保 asdecimal 设置适用于正在使用的 DBAPI - 当 Numeric 应用从 Decimal->float 或 float-> Decimal 的转换时，此转换会为接收到的所有结果列产生额外的性能开销。

返回 Decimal 值的 DBAPI（例如 psycopg2）将在设置为`True`时具有更好的精度和更高的性能，因为对 Decimal 的本机转换减少了浮点问题的发生，并且 Numeric 类型本身不需要进行任何进一步的转换。然而，另一个返回浮点数的 DBAPI 将会产生额外的转换开销，并且仍然可能发生浮点数据丢失 - 在这种情况下，`asdecimal=False` 至少会消除额外的转换开销。

```py
method bind_processor(dialect)
```

返回一个处理绑定值的转换函数。

返回一个可调用对象，该对象将接收一个绑定参数值作为唯一的位置参数，并返回要发送到 DB-API 的值。

如果不需要处理，则该方法应返回 `None`。

注意

此方法仅相对于**方言特定类型对象**调用，通常**私有于使用的方言**，并且不是与公共类型对象相同的类型对象，这意味着无法简单地通过子类化`TypeEngine`类来提供替代的`TypeEngine.bind_processor()`方法，除非显式地子类化`UserDefinedType`类。

要为`TypeEngine.bind_processor()`提供替代行为，需要实现一个`TypeDecorator`类，并提供`TypeDecorator.process_bind_param()`的实现。

另请参阅

扩展现有类型

参数：

**方言** – 正在使用的方言实例。

```py
method get_dbapi_type(dbapi)
```

如果有的话，从底层 DB-API 返回相应的类型对象。

例如，可以用于调用`setinputsizes()`。

```py
method literal_processor(dialect)
```

返回一个转换函数，用于处理直接渲染而不使用绑定的文字值。

当编译器使用“literal_binds”标志时，通常用于 DDL 生成以及某些后端不接受绑定参数的情况下，将使用此函数。

返回一个可调用对象，它将接收一个文字 Python 值作为唯一的位置参数，并返回一个字符串表示，以在 SQL 语句中呈现。

注意

此方法仅针对**方言特定类型对象**，通常**私有于使用的方言**，并且与公共类型对象不同，这意味着无法简单地通过子类化`TypeEngine`类来提供替代的`TypeEngine.literal_processor()`方法，除非显式地子类化`UserDefinedType`类。

要为 `TypeEngine.literal_processor()` 提供替代行为，需实现一个 `TypeDecorator` 类，并提供一个 `TypeDecorator.process_literal_param()` 的实现。

请参见

增强现有类型

```py
attribute python_type
```

```py
method result_processor(dialect, coltype)
```

返回一个转换函数，用于处理结果行值。

返回一个可调用对象，它将接收一个结果行列值作为唯一的位置参数，并将返回一个值以返回给用户。

如果不需要处理，该方法应返回 `None`。

注意

此方法仅在**特定方言类型对象**相对调用，该对象通常**是方言中私有的**，并且不是与公共面向用户的对象相同，这意味着无法子类化 `TypeEngine` 类以提供替代 `TypeEngine.result_processor()` 方法，除非显式子类化 `UserDefinedType` 类。

要为 `TypeEngine.result_processor()` 提供替代行为，需实现一个 `TypeDecorator` 类，并提供一个 `TypeDecorator.process_result_value()` 的实现。

请参见

增强现有类型

参数：

+   `dialect` – 正在使用的方言实例。

+   `coltype` – 在 cursor.description 中接收到的 DBAPI coltype 参数。

```py
class sqlalchemy.types.PickleType
```

持有由 pickle 序列化的 Python 对象。

PickleType 是建立在 Binary 类型之上的，它将 Python 的 `pickle.dumps()` 应用于传入的对象，并在传出时应用 `pickle.loads()`，允许任何可 pickle 的 Python 对象被存储为序列化的二进制字段。

若要允许与 `PickleType` 关联的元素的 ORM 更改事件传播，请参见 变异跟踪。

**成员**

__init__(), bind_processor(), cache_ok, compare_values(), impl, result_processor()

**类签名**

类`sqlalchemy.types.PickleType`（`sqlalchemy.types.TypeDecorator`）

```py
method __init__(protocol: int = 5, pickler: Any = None, comparator: Callable[[Any, Any], bool] | None = None, impl: _TypeEngineArgument[Any] | None = None)
```

构造一个 PickleType。

参数：

+   `protocol` – 默认为`pickle.HIGHEST_PROTOCOL`。

+   `pickler` – 默认为 pickle。可以是具有 pickle 兼容的`dumps`和`loads`方法的任何对象。

+   `comparator` – 用于比较此类型的值的二元调用谓词。如果保持为`None`，则使用 Python 的“equals”运算符来比较值。

+   `impl` –

    用于替代默认的`LargeBinary`的二进制存储`TypeEngine`类或实例。例如，在使用 MySQL 时，:class: _mysql.LONGBLOB 类可能更有效。

    新版本 1.4.20 中新增。

```py
method bind_processor(dialect)
```

为给定的`Dialect`提供绑定值处理函数。

这是实现绑定值转换的`TypeEngine`合约的方法，通常通过`TypeEngine.bind_processor()`方法实现。

注

用户定义的`TypeDecorator`子类**不应**实现此方法，而应实现`TypeDecorator.process_bind_param()`，以便保持实现类型提供的“内部”处理。

参数：

**方言** – 正在使用的方言实例。

```py
attribute cache_ok: bool | None = True
```

表明使用此`ExternalType`的语句是否“安全可缓存”。

默认值`None`将发出警告，然后不允许缓存包含此类型的语句。设置为`False`可完全禁用包含此类型的语句的缓存而不发出警告。设置为`True`时，对象的类和其状态的选定元素将用作缓存键的一部分。例如，使用`TypeDecorator`：

```py
class MyType(TypeDecorator):
    impl = String

    cache_ok = True

    def __init__(self, choices):
        self.choices = tuple(choices)
        self.internal_only = True
```

上述类型的缓存键相当于：

```py
>>> MyType(["a", "b", "c"])._static_cache_key
(<class '__main__.MyType'>, ('choices', ('a', 'b', 'c')))
```

缓存方案将从与`__init__()`方法中参数名称相对应的类型中提取属性。在上面的示例中，“choices”属性成为缓存键的一部分，但“internal_only”则不是，因为没有名为“internal_only”的参数。

可缓存元素的要求是它们是可哈希的，并且还要表明每次针对给定缓存值使用此类型的表达式时生成相同的 SQL 渲染。

为了适应引用不可哈希结构（如字典、集合和列表）的数据类型，可以通过将可哈希结构分配给与参数名称相对应的属性来使这些对象“可缓存”。例如，一个接受查找值字典的数据类型可以将其公开为排序后的元组序列。假设先前不可缓存的类型为：

```py
class LookupType(UserDefinedType):
  '''a custom type that accepts a dictionary as a parameter.

 this is the non-cacheable version, as "self.lookup" is not
 hashable.

 '''

    def __init__(self, lookup):
        self.lookup = lookup

    def get_col_spec(self, **kw):
        return "VARCHAR(255)"

    def bind_processor(self, dialect):
        # ...  works with "self.lookup" ...
```

这里的“lookup”是一个字典。该类型将无法生成缓存键：

```py
>>> type_ = LookupType({"a": 10, "b": 20})
>>> type_._static_cache_key
<stdin>:1: SAWarning: UserDefinedType LookupType({'a': 10, 'b': 20}) will not
produce a cache key because the ``cache_ok`` flag is not set to True.
Set this flag to True if this type object's state is safe to use
in a cache key, or False to disable this warning.
symbol('no_cache')
```

如果我们**设置了**这样一个缓存键，它是不能用的。我们会得到一个包含字典的元组结构，其中的字典本身不能作为“缓存字典”（例如 SQLAlchemy 的语句缓存）中的键，因为 Python 字典不可哈希：

```py
>>> # set cache_ok = True
>>> type_.cache_ok = True

>>> # this is the cache key it would generate
>>> key = type_._static_cache_key
>>> key
(<class '__main__.LookupType'>, ('lookup', {'a': 10, 'b': 20}))

>>> # however this key is not hashable, will fail when used with
>>> # SQLAlchemy statement cache
>>> some_cache = {key: "some sql value"}
Traceback (most recent call last): File "<stdin>", line 1,
in <module> TypeError: unhashable type: 'dict'
```

可以通过将排序后的元组元组分配给“lookup”属性来使类型可缓存：

```py
class LookupType(UserDefinedType):
  '''a custom type that accepts a dictionary as a parameter.

 The dictionary is stored both as itself in a private variable,
 and published in a public variable as a sorted tuple of tuples,
 which is hashable and will also return the same value for any
 two equivalent dictionaries.  Note it assumes the keys and
 values of the dictionary are themselves hashable.

 '''

    cache_ok = True

    def __init__(self, lookup):
        self._lookup = lookup

        # assume keys/values of "lookup" are hashable; otherwise
        # they would also need to be converted in some way here
        self.lookup = tuple(
            (key, lookup[key]) for key in sorted(lookup)
        )

    def get_col_spec(self, **kw):
        return "VARCHAR(255)"

    def bind_processor(self, dialect):
        # ...  works with "self._lookup" ...
```

在上述情况下，`LookupType({"a": 10, "b": 20})`的缓存键将是：

```py
>>> LookupType({"a": 10, "b": 20})._static_cache_key
(<class '__main__.LookupType'>, ('lookup', (('a', 10), ('b', 20))))
```

从版本 1.4.14 开始：- 添加了`cache_ok`标志，以允许对`TypeDecorator` 类进行某种缓存配置。

从版本 1.4.28 开始：- 添加了 `ExternalType` 混合类型，它将`cache_ok`标志推广到 `TypeDecorator` 和 `UserDefinedType` 类。

另请参见

SQL 编译缓存

```py
method compare_values(x, y)
```

给定两个值，比较它们是否相等。

默认情况下，这将调用底层“impl”的 `TypeEngine.compare_values()` 方法，而该方法通常使用 Python 等号运算符`==`。

此函数由 ORM 用于比较原始加载值与拦截的“更改”值，以确定是否发生了净变化。

```py
attribute impl
```

`LargeBinary` 的别名

```py
method result_processor(dialect, coltype)
```

为给定的`Dialect` 提供结果值处理函数。

这是通过 `TypeEngine.result_processor()` 方法正常发生的绑定值转换的方法，用于实现 `TypeEngine` 合约的方法。

注意

用户自定义的 `TypeDecorator` 的子类**不应该**实现这个方法，而应该实现 `TypeDecorator.process_result_value()` 方法，以便保持实现类型提供的“内部”处理。

参数：

+   `dialect` – 正在使用的方言实例。

+   `coltype` – 一个 SQLAlchemy 数据类型

```py
class sqlalchemy.types.SchemaType
```

为类型添加允许与类型关联的模式级 DDL 的功能。

支持必须显式创建/删除的类型（例如 PG ENUM 类型），以及受表或模式级约束、触发器和其他规则补充的类型。

`SchemaType` 类还可以成为 `DDLEvents.before_parent_attach()` 和 `DDLEvents.after_parent_attach()` 事件的目标，这些事件在类型对象与父 `Column` 关联时触发。 

另见

`Enum`

`Boolean`

**成员**

adapt(), copy(), create(), drop(), name

**类签名**

类 `sqlalchemy.types.SchemaType`（`sqlalchemy.sql.expression.SchemaEventTarget`, `sqlalchemy.types.TypeEngineMixin`）

```py
method adapt(cls: Type[TypeEngine | TypeEngineMixin], **kw: Any) → TypeEngine
```

```py
method copy(**kw)
```

```py
method create(bind, checkfirst=False)
```

如适用，请为此类型生成 CREATE DDL。

```py
method drop(bind, checkfirst=False)
```

如适用，请为此类型生成 DROP DDL。

```py
attribute name: str | None
```

```py
class sqlalchemy.types.SmallInteger
```

用于较小的 `int` 整数的类型。

通常在 DDL 中生成 `SMALLINT`，在 Python 端的行为与普通的 `Integer` 类型相似。

**类签名**

类 `sqlalchemy.types.SmallInteger`（`sqlalchemy.types.Integer`）

```py
class sqlalchemy.types.String
```

所有字符串和字符类型的基类。

在 SQL 中，对应于 VARCHAR。

当在 CREATE TABLE 语句中使用 String 类型时，通常需要长度字段，因为大多数数据库都要求 VARCHAR 指定长度。

**成员**

__init__(), bind_processor(), get_dbapi_type(), literal_processor(), python_type, result_processor()

**类签名**

类`sqlalchemy.types.String` (`sqlalchemy.types.Concatenable`, `sqlalchemy.types.TypeEngine`)

```py
method __init__(length: int | None = None, collation: str | None = None)
```

创建一个字符串持有类型。

参数：

+   `length` – 可选，列的长度，用于 DDL 和 CAST 表达式。如果不会发出`CREATE TABLE`，可以安全地省略。某些数据库可能需要在 DDL 中使用长度，并且如果包含没有长度的 VARCHAR，则在发出`CREATE TABLE` DDL 时将引发异常。值是以字节还是字符解释的是与数据库相关的。

+   `collation` –

    可选，用于 DDL 和 CAST 表达式的列级排序。使用 SQLite、MySQL 和 PostgreSQL 支持的 COLLATE 关键字进行渲染。例如：

    ```py
    >>> from sqlalchemy import cast, select, String
    >>> print(select(cast('some string', String(collation='utf8'))))
    SELECT  CAST(:param_1  AS  VARCHAR  COLLATE  utf8)  AS  anon_1 
    ```

    注意

    在大多数情况下，应该使用`Unicode`或`UnicodeText`数据类型来存储非 ASCII 数据的`Column`。这些数据类型将确保在数据库上使用正确的类型。

```py
method bind_processor(dialect)
```

返回一个用于处理绑定值的转换函数。

返回一个可调用对象，该对象将接收一个绑定参数值作为唯一的位置参数，并返回一个要发送到 DB-API 的值。

如果不需要处理，则方法应返回`None`。

注意

该方法仅相对于**方言特定类型对象**调用，该对象通常**是正在使用的方言的私有对象**，并且不是公共类型对象，这意味着无法通过子类化`TypeEngine`类来提供替代的`TypeEngine.bind_processor()`方法，除非显式子类化`UserDefinedType`类。

要为 `TypeEngine.bind_processor()` 提供替代行为，请实现一个 `TypeDecorator` 类，并提供 `TypeDecorator.process_bind_param()` 的实现。

另请参阅

扩展现有类型

参数：

**方言** – 使用的方言实例。

```py
method get_dbapi_type(dbapi)
```

如果有的话，从底层 DB-API 返回相应的类型对象。

例如，这对于调用 `setinputsizes()` 很有用。

```py
method literal_processor(dialect)
```

返回一个用于处理直接呈现而不使用绑定的文字值的转换函数。

当编译器使用“literal_binds”标志时，通常用于 DDL 生成以及在某些后端不接受绑定参数的情况下使用此函数。

返回一个可调用对象，它将接收一个原始的 Python 值作为唯一的位置参数，并返回一个字符串表示，用于在 SQL 语句中呈现。

注意

此方法仅针对**特定方言类型对象**调用，该对象通常是**使用的方言私有**，并且不是公共类型对象，这意味着无法子类化 `TypeEngine` 类以提供替代 `TypeEngine.literal_processor()` 方法，除非显式子类化 `UserDefinedType` 类。

要为 `TypeEngine.literal_processor()` 提供替代行为，请实现一个 `TypeDecorator` 类，并提供 `TypeDecorator.process_literal_param()` 的实现。

另请参阅

扩展现有类型

```py
attribute python_type
```

```py
method result_processor(dialect, coltype)
```

返回一个用于处理结果行值的转换函数。

返回一个可调用对象，它将接收一个结果行列值作为唯一的位置参数，并返回一个值以返回给用户。

如果不需要处理，该方法应返回 `None`。

注意

此方法仅相对于**特定于方言的类型对象**调用，该对象通常是特定方言中**私有的**，并且不是与公共接口相同的类型对象，这意味着不可通过子类化`TypeEngine`类来提供备用的`TypeEngine.result_processor()`方法，除非显式地子类化`UserDefinedType`类。

要为`TypeEngine.result_processor()`提供备用行为，实现一个`TypeDecorator`类，并提供一个`TypeDecorator.process_result_value()`的实现。

另请参阅

增强现有类型

参数：

+   `dialect` – 正在使用的方言实例。

+   `coltype` – 在 cursor.description 中收到的 DBAPI coltype 参数。

```py
class sqlalchemy.types.Text
```

可变大小的字符串类型。

在 SQL 中，通常对应于 CLOB 或 TEXT。一般而言，TEXT 对象没有长度；虽然某些数据库将在此处接受长度参数，但其他数据库将拒绝它。

**类签名**

类`sqlalchemy.types.Text`（`sqlalchemy.types.String`）

```py
class sqlalchemy.types.Time
```

用于`datetime.time()`对象的类型。

**成员**

get_dbapi_type()，literal_processor()，python_type

**类签名**

类`sqlalchemy.types.Time`（`sqlalchemy.types._RenderISO8601NoT`，`sqlalchemy.types.HasExpressionLookup`，`sqlalchemy.types.TypeEngine`）

```py
method get_dbapi_type(dbapi)
```

如果有的话，从底层 DB-API 返回相应的类型对象。

例如，这对于调用`setinputsizes()`可能很有用。

```py
method literal_processor(dialect)
```

返回一个用于处理字面值的转换函数，这些字面值将直接呈现，而不使用绑定。

当编译器使用“literal_binds”标志时使用此函数，通常在 DDL 生成以及某些后端不接受绑定参数的情况下使用。

返回一个可调用对象，该对象将接收一个字面的 Python 值作为唯一的位置参数，并返回一个要在 SQL 语句中呈现的字符串表示。

注意

该方法仅相对于**特定方言的类型对象**调用，该对象通常**私有于正在使用的方言**，并且与公共面向的类型对象不同，这意味着无法子类化`TypeEngine`类以提供替代的`TypeEngine.literal_processor()`方法，除非显式子类化`UserDefinedType`类。

要为`TypeEngine.literal_processor()`提供替代行为，请实现一个`TypeDecorator`类，并提供一个`TypeDecorator.process_literal_param()`的实现。

另请参阅

增强现有类型

```py
attribute python_type
```

```py
class sqlalchemy.types.Unicode
```

变长 Unicode 字符串类型。

`Unicode`类型是一个`String`子类，假定输入和输出字符串可能包含非 ASCII 字符，并且对于一些后端，暗示着明确支持非 ASCII 数据的底层列类型，比如在 Oracle 和 SQL Server 上的`NVARCHAR`。这将影响方言级别的`CREATE TABLE`语句和`CAST`函数的输出。

数据库中用于传输和接收数据的`Unicode`类型使用的字符编码通常由 DBAPI 本身确定。所有现代 DBAPI 都支持非 ASCII 字符串，但可能具有不同的管理数据库编码的方法；如有必要，应按照 Dialects 部分目标 DBAPI 的注意事项进行配置。

在现代 SQLAlchemy 中，使用`Unicode`数据类型不意味着 SQLAlchemy 本身具有任何编码/解码行为。在 Python 3 中，所有字符串对象都具有 Unicode 功能，并且 SQLAlchemy 不会生成字节字符串对象，也不会适应不返回 Python Unicode 对象作为字符串值结果集的 DBAPI。

警告

一些数据库后端，特别是使用 pyodbc 的 SQL Server，已知存在与被标记为 `NVARCHAR` 类型而不是 `VARCHAR` 类型的数据相关的不良行为，包括数据类型不匹配错误和不使用索引。 有关解决像 SQL Server 与 pyodbc 以及 cx_Oracle 这样的后端的 Unicode 字符问题的背景信息，请参阅关于 `DialectEvents.do_setinputsizes()` 的部分。

另请参阅

`UnicodeText` - 与 `Unicode` 相对应的无长度文本类型。

`DialectEvents.do_setinputsizes()`

**类签名**

类 `sqlalchemy.types.Unicode` (`sqlalchemy.types.String`)

```py
class sqlalchemy.types.UnicodeText
```

一个无界限长度的 Unicode 字符串类型。

参阅 `Unicode` 以了解此对象的 Unicode 行为详细信息。

与 `Unicode` 类似，使用 `UnicodeText` 类型意味着在后端使用 Unicode 能力类型，如 `NCLOB`、`NTEXT`。

**类签名**

类 `sqlalchemy.types.UnicodeText` (`sqlalchemy.types.Text`)

```py
class sqlalchemy.types.Uuid
```

表示数据库无关的 UUID 数据类型。

对于没有“本地”UUID 数据类型的后端，该值将使用 `CHAR(32)` 并将 UUID 作为 32 个字符的字母数字十六进制字符串进行存储。

对于已知直接支持 `UUID` 或类似的 uuid 存储数据类型（例如 SQL Server 的 `UNIQUEIDENTIFIER`）的后端，启用默认的“本地”模式将允许在这些后端使用这些类型。

在其默认使用模式下，`Uuid` 数据类型期望来自 Python [uuid](https://docs.python.org/3/library/uuid.html) 模块的**Python uuid 对象**：

```py
import uuid

from sqlalchemy import Uuid
from sqlalchemy import Table, Column, MetaData, String

metadata_obj = MetaData()

t = Table(
    "t",
    metadata_obj,
    Column('uuid_data', Uuid, primary_key=True),
    Column("other_data", String)
)

with engine.begin() as conn:
    conn.execute(
        t.insert(),
        {"uuid_data": uuid.uuid4(), "other_data", "some data"}
    )
```

要使 `Uuid` 数据类型与基于字符串的 Uuid（例如 32 字符十六进制字符串）配合使用，传递 `Uuid.as_uuid` 参数，并将值设为 `False`。

新版本 2.0 中新增。

另请参阅

`UUID` - 表示没有任何后端不可知行为的 `UUID` 数据类型。

**成员**

__init__(), bind_processor(), coerce_compared_value(), literal_processor(), python_type, result_processor()

**类签名**

类 `sqlalchemy.types.Uuid` (`sqlalchemy.types.Emulated`, `sqlalchemy.types.TypeEngine`)

```py
method __init__(as_uuid: bool = True, native_uuid: bool = True)
```

构造一个 `Uuid` 类型。

参数：

+   `as_uuid=True` –

    如果为 True，则将值解释为 Python uuid 对象，通过 DBAPI 转换为/从字符串。

+   `native_uuid=True` – 如果为 True，则支持直接的 `UUID` 数据类型或存储 UUID 值的后端（例如 SQL Server 的 `UNIQUEIDENTIFIER`）将使用这些后端。如果为 False，则对于所有后端都将使用 `CHAR(32)` 数据类型，而不管原生支持情况。

```py
method bind_processor(dialect)
```

返回一个转换函数，用于处理绑定值。

返回一个可调用对象，它将接收一个绑定参数值作为唯一的位置参数，并返回一个要发送到 DB-API 的值。

如果不需要处理，则该方法应返回 `None`。

注意

该方法仅相对于一个**特定方言的类型对象**调用，该对象通常**是正在使用的方言的私有对象**，并且不是与公共类型对象相同的类型对象，这意味着不可通过子类化`TypeEngine`类来提供备用 `TypeEngine.bind_processor()` 方法，除非明确子类化`UserDefinedType`类。

要为 `TypeEngine.bind_processor()` 提供替代行为，请实现 `TypeDecorator` 类并提供 `TypeDecorator.process_bind_param()` 的实现。

另请参见

扩展现有类型

参数：

**方言** – 正在使用的方言实例。

```py
method coerce_compared_value(op, value)
```

有关说明，请参阅 `TypeEngine.coerce_compared_value()`。

```py
method literal_processor(dialect)
```

返回一个转换函数，用于处理要直接渲染而不使用绑定的文本值。

当编译器使用“literal_binds”标志时，将使用此函数，通常在 DDL 生成以及后端不接受绑定参数的某些情况下使用。

返回一个可调用对象，该对象将接收一个字面 Python 值作为唯一位置参数，并返回要在 SQL 语句中呈现的字符串表示。

注意

此方法仅相对于**特定方言的类型对象**调用，该对象通常是**正在使用的方言的私有对象**，并且不是与公共类型对象相同的类型对象，这意味着无法通过子类化 `TypeEngine` 类来提供替代 `TypeEngine.literal_processor()` 方法，除非显式地子类化 `UserDefinedType` 类。

要为 `TypeEngine.literal_processor()` 提供替代行为，实现一个 `TypeDecorator` 类并提供 `TypeDecorator.process_literal_param()` 的实现。

另请参见

扩展现有类型

```py
attribute python_type
```

```py
method result_processor(dialect, coltype)
```

返回用于处理结果行值的转换函数。

返回一个可调用对象，该对象将接收结果行列值作为唯一位置参数，并将返回一个要返回给用户的值。

如果不需要处理，则该方法应返回`None`。

注意

此方法仅相对于**特定方言的类型对象**调用，该对象通常是**正在使用的方言的私有对象**，并且不是与公共类型对象相同的类型对象，这意味着无法通过子类化 `TypeEngine` 类来提供替代 `TypeEngine.result_processor()` 方法，除非显式地子类化 `UserDefinedType` 类。

要为 `TypeEngine.result_processor()` 提供替代行为，实现一个 `TypeDecorator` 类并提供 `TypeDecorator.process_result_value()` 的实现。

另请参阅

增强现有类型

参数：

+   `方言` – 正在使用的方言实例。

+   `coltype` – 在 cursor.description 中接收的 DBAPI coltype 参数。## SQL 标准和多厂商“大写”类型

这类类型指的是那些要么是 SQL 标准的一部分，要么可能在一些数据库后端的子集中找到的类型。与“通用”类型不同，SQL 标准/多厂商类型**没有**保证在所有后端上工作，并且只会在那些明确以名称支持它们的后端上工作。也就是说，当发出 `CREATE TABLE` 时，该类型将始终以其确切名称在 DDL 中发出。

| 对象名称 | 描述 |
| --- | --- |
| 数组 | 表示 SQL 数组类型。 |
| 大整数 | SQL BIGINT 类型。 |
| BINARY | SQL BINARY 类型。 |
| BLOB | SQL BLOB 类型。 |
| 布尔型 | SQL 布尔类型。 |
| CHAR | SQL CHAR 类型。 |
| CLOB | CLOB 类型。 |
| 日期 | SQL DATE 类型。 |
| 日期时间 | SQL DATETIME 类型。 |
| 十进制 | SQL DECIMAL 类型。 |
| 双精度 | SQL DOUBLE 类型。 |
| 双精度浮点数 | SQL DOUBLE PRECISION 类型。 |
| 浮点数 | SQL FLOAT 类型。 |
| 整数 | `INTEGER` 的别名 |
| 整数 | SQL INT 或 INTEGER 类型。 |
| JSON | 表示 SQL JSON 类型。 |
| NCHAR | SQL NCHAR 类型。 |
| 数值 | SQL NUMERIC 类型。 |
| NVARCHAR | SQL NVARCHAR 类型。 |
| 实数 | SQL REAL 类型。 |
| 小整数 | SQL SMALLINT 类型。 |
| 文本 | SQL TEXT 类型。 |
| 时间 | SQL TIME 类型。 |
| 时间戳 | SQL TIMESTAMP 类型。 |
| UUID | 表示 SQL UUID 类型。 |
| VARBINARY | SQL VARBINARY 类型。 |
| 可变长度字符串 | SQL VARCHAR 类型。 |

```py
class sqlalchemy.types.ARRAY
```

表示 SQL 数组类型。

注意

这种类型是所有 ARRAY 操作的基础。然而，目前**只有 PostgreSQL 后端支持 SQLAlchemy 中的 SQL 数组**。建议在与 PostgreSQL 使用 ARRAY 类型时直接使用 PostgreSQL 特定的`sqlalchemy.dialects.postgresql.ARRAY`类型，因为它提供了特定于该后端的附加运算符。

`ARRAY`是 Core 中支持各种 SQL 标准函数的一部分，例如`array_agg`，这些函数明确涉及数组；然而，除了 PostgreSQL 后端和可能一些第三方方言外，没有其他 SQLAlchemy 内置方言支持这种类型。

给定元素的“类型”，构造了一个`ARRAY`类型：

```py
mytable = Table("mytable", metadata,
        Column("data", ARRAY(Integer))
    )
```

上述类型表示一个 N 维数组，这意味着支持的后端（如 PostgreSQL）将自动解释具有任意维度数量的值。要生成传入整数一维数组的 INSERT 构造：

```py
connection.execute(
        mytable.insert(),
        {"data": [1,2,3]}
)
```

可以根据固定的维度数量构造`ARRAY`类型：

```py
mytable = Table("mytable", metadata,
        Column("data", ARRAY(Integer, dimensions=2))
    )
```

发送维度数量是可选的，但如果数据类型要表示多维数组，则建议这样做。这个数字被用于：

+   当将类型声明本身发送到数据库时，例如，`INTEGER[][]`

+   当将 Python 值转换为数据库值，反之亦然，例如，一个包含`Unicode`对象的数组使用这个数字来有效地访问数组结构内的字符串值，而不需要进行逐行类型检查

+   当与 Python 的`getitem`访问器一起使用时，维度数量用于定义`[]`运算符应返回的类型，例如，对于具有两个维度的整数数组：

    ```py
    >>> expr = table.c.column[5]  # returns ARRAY(Integer, dimensions=1)
    >>> expr = expr[6]  # returns Integer
    ```

对于一维数组，没有维度参数的`ARRAY`实例通常假定单维行为。

类型为`ARRAY`的 SQL 表达式支持“索引”和“切片”行为。`[]`运算符生成表达式构造，这些构造将为 SELECT 语句生成适当的 SQL：

```py
select(mytable.c.data[5], mytable.c.data[2:7])
```

以及在使用`Update.values()`方法时进行 UPDATE 语句：

```py
mytable.update().values({
    mytable.c.data[5]: 7,
    mytable.c.data[2:7]: [1, 2, 3]
})
```

默认情况下，索引访问是基于一的；要进行从零开始的索引转换，请设置`ARRAY.zero_indexes`。

`ARRAY` 类型还提供运算符 `Comparator.any()` 和 `Comparator.all()`。PostgreSQL 特定版本的 `ARRAY` 还提供了其他运算符。

**在使用 ORM 时检测 ARRAY 列的更改**

当与 SQLAlchemy ORM 一起使用时，`ARRAY` 类型不会检测对数组的原位突变。为了检测到这些变化，必须使用 `sqlalchemy.ext.mutable` 扩展，并使用 `MutableList` 类：

```py
from sqlalchemy import ARRAY
from sqlalchemy.ext.mutable import MutableList

class SomeOrmClass(Base):
    # ...

    data = Column(MutableList.as_mutable(ARRAY(Integer)))
```

此扩展将允许对数组进行“原位”更改，例如 `.append()` 以产生单位工作检测到的事件。请注意，对数组中的元素进行更改，包括原地突变的子数组，不会被检测到。

或者，将新的数组值分配给替换旧值的 ORM 元素将始终触发更改事件。

另请参阅

`sqlalchemy.dialects.postgresql.ARRAY`

**成员**

__init__(), contains(), any(), all()

**类签名**

类 `sqlalchemy.types.ARRAY` (`sqlalchemy.sql.expression.SchemaEventTarget`, `sqlalchemy.types.Indexable`, `sqlalchemy.types.Concatenable`, `sqlalchemy.types.TypeEngine`)

```py
method __init__(item_type: _TypeEngineArgument[Any], as_tuple: bool = False, dimensions: int | None = None, zero_indexes: bool = False)
```

构造一个 `ARRAY`。

例如：

```py
Column('myarray', ARRAY(Integer))
```

参数是：

参数：

+   `item_type` – 此数组项的数据类型。请注意，此处维度不相关，因此像 `INTEGER[][]` 这样的多维数组被构造为 `ARRAY(Integer)`，而不是 `ARRAY(ARRAY(Integer))` 或类似的。

+   `as_tuple=False` – 指定返回结果是否应从列表转换为元组。通常不需要此参数，因为 Python 列表很好地对应于 SQL 数组。

+   `dimensions` – 如果不是 None，则数组将假定具有固定数量的维度。这影响了数组在数据库上的声明方式，以及它如何解释 Python 和结果值，以及如何与“getitem”运算符结合使用时的表达式行为。有关更多详细信息，请参见`ARRAY`的描述。

+   `zero_indexes=False` – 当为 True 时，索引值将在 Python 零基础和 SQL 一基础索引之间转换，例如，在传递到数据库之前，所有索引值都将增加一个值。

```py
class Comparator
```

为`ARRAY`定义比较操作。

此类型的方言特定形式上还有更多的操作符。参见`Comparator`。

**类签名**

类`sqlalchemy.types.ARRAY.Comparator` (`sqlalchemy.types.Comparator`, `sqlalchemy.types.Comparator`)

```py
method contains(*arg, **kw)
```

`ARRAY.contains()` 对于基本的数组类型没有实现。请使用特定于方言的数组类型。

另请参阅

`ARRAY` - PostgreSQL 特定版本。

```py
method any(other, operator=None)
```

返回`other operator ANY (array)`子句。

传统功能

这个方法是一个`ARRAY` - 特定的构造，现在已经被`any_()`函数取代，其具有不同的调用风格。`any_()`函数也通过`ColumnOperators.any_()`方法在方法级别进行了镜像。

使用数组特定的`Comparator.any()`的用法如下：

```py
from sqlalchemy.sql import operators

conn.execute(
    select(table.c.data).where(
            table.c.data.any(7, operator=operators.lt)
        )
)
```

参数：

+   `other` – 要进行比较的表达式

+   `operator` – 来自`sqlalchemy.sql.operators`包的操作符对象，默认为`eq()`。

另请参阅

`any_()`

`Comparator.all()`

```py
method all(other, operator=None)
```

返回`other operator ALL (array)`子句。

传统功能

此方法是一个 `ARRAY` - 特定构造，现在已被 `all_()` 函数取代，具有不同的调用风格。`all_()` 函数也通过 `ColumnOperators.all_()` 方法在方法级别进行了镜像。

使用特定于数组的 `Comparator.all()` 如下：

```py
from sqlalchemy.sql import operators

conn.execute(
    select(table.c.data).where(
            table.c.data.all(7, operator=operators.lt)
        )
)
```

参数：

+   `other` – 待比较的表达式

+   `operator` – 来自 `sqlalchemy.sql.operators` 包的操作对象，默认为 `eq()`。

参见

`all_()`

`Comparator.any()`

```py
class sqlalchemy.types.BIGINT
```

SQL BIGINT 类型。

参见

`BigInteger` - 基本类型的文档。

**类签名**

类`sqlalchemy.types.BIGINT`（`sqlalchemy.types.BigInteger`）

```py
class sqlalchemy.types.BINARY
```

SQL BINARY 类型。

**类签名**

类`sqlalchemy.types.BINARY`（`sqlalchemy.types._Binary`）

```py
class sqlalchemy.types.BLOB
```

SQL BLOB 类型。

**成员**

__init__()

**类签名**

类`sqlalchemy.types.BLOB`（`sqlalchemy.types.LargeBinary`）

```py
method __init__(length: int | None = None)
```

*继承自* `LargeBinary` *的* `sqlalchemy.types.LargeBinary.__init__` *方法*

构造一个 LargeBinary 类型。

参数：

**length** – 可选，用于 DDL 语句中的列长度，适用于接受长度的二进制类型，如 MySQL BLOB 类型。

```py
class sqlalchemy.types.BOOLEAN
```

SQL BOOLEAN 类型。

**成员**

__init__()

**类签名**

类`sqlalchemy.types.BOOLEAN`（`sqlalchemy.types.Boolean`）

```py
method __init__(create_constraint: bool = False, name: str | None = None, _create_events: bool = True, _adapted_from: SchemaType | None = None)
```

*继承自* `Boolean` *的* `sqlalchemy.types.Boolean.__init__` *方法*

构造一个布尔值。

参数：

+   `create_constraint` –

    默认为 False。如果布尔值生成为 int/smallint，还会在表上创建一个 CHECK 约束，确保值为 1 或 0。

    注意

    强烈建议 CHECK 约束具有明确的名称，以支持模式管理问题。可以通过设置 `Boolean.name` 参数或设置适当的命名约定来建立这一点；参见配置约束命名约定以获取背景信息。

    在版本 1.4 中更改：-此标志现在默认为 False，表示对非本地枚举类型不会生成 CHECK 约束。

+   `name` – 如果生成 CHECK 约束，请指定约束的名称。

```py
class sqlalchemy.types.CHAR
```

SQL CHAR 类型。

**成员**

__init__()

**类签名**

类`sqlalchemy.types.CHAR`（`sqlalchemy.types.String`）

```py
method __init__(length: int | None = None, collation: str | None = None)
```

*继承自* `String` *的* `sqlalchemy.types.String.__init__` *方法*

创建一个字符串类型。

参数：

+   `length` – 可选的，用于 DDL 和 CAST 表达式中的列长度。如果不会发出`CREATE TABLE`，则可以安全地省略。某些数据库可能需要在 DDL 中使用长度，并且如果包含没有长度的`VARCHAR`，则在发出`CREATE TABLE` DDL 时会引发异常。该值是以字节还是字符解释的取决于数据库。

+   `collation` –

    可选的，用于 DDL 和 CAST 表达式中的列级排序。使用 SQLite、MySQL 和 PostgreSQL 支持的 COLLATE 关键字渲染。例如：

    ```py
    >>> from sqlalchemy import cast, select, String
    >>> print(select(cast('some string', String(collation='utf8'))))
    SELECT  CAST(:param_1  AS  VARCHAR  COLLATE  utf8)  AS  anon_1 
    ```

    注意

    在大多数情况下，应该使用`Unicode`或`UnicodeText`数据类型来存储非 ASCII 数据的`Column`。这些数据类型将确保在数据库上使用正确的类型。

```py
class sqlalchemy.types.CLOB
```

CLOB 类型。

此类型在 Oracle 和 Informix 中找到。

**成员**

__init__()

**类签名**

类`sqlalchemy.types.CLOB`（`sqlalchemy.types.Text`）

```py
method __init__(length: int | None = None, collation: str | None = None)
```

*继承自* `String` *的* `sqlalchemy.types.String.__init__` *方法*

创建一个字符串类型。

参数：

+   `length` – 可选的，用于 DDL 和 CAST 表达式中的列长度。如果不会发出`CREATE TABLE`，则可以安全地省略。某些数据库可能需要在 DDL 中使用长度，并且如果包含没有长度的`VARCHAR`，则在发出`CREATE TABLE` DDL 时会引发异常。该值是以字节还是字符解释的取决于数据库。

+   `collation` –

    可选，用于 DDL 和 CAST 表达式中的列级排序。使用 SQLite、MySQL 和 PostgreSQL 支持的 COLLATE 关键字呈现。例如：

    ```py
    >>> from sqlalchemy import cast, select, String
    >>> print(select(cast('some string', String(collation='utf8'))))
    SELECT  CAST(:param_1  AS  VARCHAR  COLLATE  utf8)  AS  anon_1 
    ```

    注意

    在大多数情况下，应使用 `Unicode` 或 `UnicodeText` 数据类型来存储非 ASCII 数据的 `Column`。这些数据类型将确保在数据库上使用正确的类型。

```py
class sqlalchemy.types.DATE
```

SQL DATE 类型。

**类签名**

class `sqlalchemy.types.DATE` (`sqlalchemy.types.Date`)

```py
class sqlalchemy.types.DATETIME
```

SQL DATETIME 类型。

**成员**

__init__()

**类签名**

class `sqlalchemy.types.DATETIME` (`sqlalchemy.types.DateTime`)

```py
method __init__(timezone: bool = False)
```

*继承自* `sqlalchemy.types.DateTime.__init__` *方法* `DateTime`

构建一个新的 `DateTime`。

参数：

**timezone** – boolean。指示日期时间类型是否应在**仅在基本日期/时间持有类型上**启用时区支持。建议在使用此标志时直接使用 `TIMESTAMP` 数据类型，因为一些数据库包括与时区功能的 TIMESTAMP 数据类型不同的独立通用日期/时间持有类型，例如 Oracle。

```py
class sqlalchemy.types.DECIMAL
```

SQL DECIMAL 类型。

另请参阅

`Numeric` - 基本类型的文档。

**成员**

__init__()

**类签名**

class `sqlalchemy.types.DECIMAL` (`sqlalchemy.types.Numeric`)

```py
method __init__(precision: int | None = None, scale: int | None = None, decimal_return_scale: int | None = None, asdecimal: bool = True)
```

*继承自* `sqlalchemy.types.Numeric.__init__` *方法* `Numeric`

构建一个 Numeric。

参数：

+   `precision` – 用于 DDL `CREATE TABLE` 的数字精度。

+   `scale` – 用于 DDL `CREATE TABLE` 的数字规模。

+   `asdecimal` – 默认为 True。返回值是否应作为 Python Decimal 对象发送，或作为浮点数发送。不同的 DBAPI 根据数据类型发送其中之一 - 数字类型将确保在 DBAPI 中一致地返回其中之一。

+   `decimal_return_scale` – 将浮点数转换为 Python 十进制时要使用的默认精度。由于十进制的不精确性，浮点值通常会更长，而大多数浮点数据库类型并没有“精度”的概念，因此默认情况下，浮点类型在转换时会查找前十位小数。指定此值将覆盖该长度。包含显式“.scale”值的类型，例如基本`Numeric`以及 MySQL 浮点类型，将使用“.scale”的值作为默认的 decimal_return_scale，如果未另行指定。

使用 `Numeric` 类型时，应注意确保 asdecimal 设置适用于正在使用的 DBAPI - 当 Numeric 应用从 Decimal->float 或 float-> Decimal 的转换时，此转换会为接收到的所有结果列增加额外的性能开销。

本地返回 Decimal 的 DBAPI（例如 psycopg2）将在设置为 `True` 时具有更好的准确性和更高的性能，因为对 Decimal 的本地转换减少了涉及的浮点问题的数量，并且 Numeric 类型本身不需要应用任何进一步的转换。但是，另一个本地返回浮点数的 DBAPI *将* 增加额外的转换开销，并且仍然可能存在浮点数据丢失 - 在这种情况下，`asdecimal=False` 将至少消除额外的转换开销。

```py
class sqlalchemy.types.DOUBLE
```

SQL DOUBLE 类型。

在 2.0 版本中新增。

另请参阅

`Double` - 基本类型的文档。

**成员**

__init__()

**类签名**

类`sqlalchemy.types.DOUBLE`（`sqlalchemy.types.Double`）。

```py
method __init__(precision: int | None = None, asdecimal: bool = False, decimal_return_scale: int | None = None)
```

*继承自* `Float` *的* `sqlalchemy.types.Float.__init__` *方法*

构造一个 Float。

参数：

+   `precision` –

    DDL `CREATE TABLE` 中用于使用的数字精度。后端**应该**尽量确保此精度表示通用`Float`数据类型的数字位数。

    注意

    对于 Oracle 后端，当渲染 DDL 时，不接受`Float.precision`参数，因为 Oracle 不支持指定为小数位数的浮点精度。而是使用 Oracle 特定的`FLOAT`数据类型并指定`FLOAT.binary_precision`参数。这是 SQLAlchemy 2.0 版本的新功能。

    要创建一个数据库不可知的`Float`，并为 Oracle 分别指定二进制精度，请使用`TypeEngine.with_variant()`如下：

    ```py
    from sqlalchemy import Column
    from sqlalchemy import Float
    from sqlalchemy.dialects import oracle

    Column(
        "float_data",
        Float(5).with_variant(oracle.FLOAT(binary_precision=16), "oracle")
    )
    ```

+   `asdecimal` – 与`Numeric`相同的标志，但默认为`False`。请注意，将此标志设置为`True`会导致浮点数转换。

+   `decimal_return_scale` – 在将浮点数转换为 Python 十进制数时使用的默认标度。由于十进制不准确性，浮点值通常会更长，并且大多数浮点数据库类型没有“标度”的概念，因此，默认情况下，浮点类型在转换时会查找前十个小数位。指定此值将覆盖该长度。请注意，如果未另行指定，包括“标度”的 MySQL 浮点类型将使用“标度”作为`decimal_return_scale`的默认值。

```py
class sqlalchemy.types.DOUBLE_PRECISION
```

SQL DOUBLE PRECISION 类型。

SQLAlchemy 2.0 中的新功能。

另请参阅

`Double` - 基本类型的文档。

**成员**

__init__()

**类签名**

类`sqlalchemy.types.DOUBLE_PRECISION` (`sqlalchemy.types.Double`)

```py
method __init__(precision: int | None = None, asdecimal: bool = False, decimal_return_scale: int | None = None)
```

*继承自* `Float` *的* `sqlalchemy.types.Float.__init__` *方法*

构造一个 Float。

参数：

+   `precision` –

    用于 DDL `CREATE TABLE`中的数字精度。后端**应该**尝试确保此精度表示通用`Float`数据类型的数字位数。

    注意

    对于 Oracle 后端，在渲染 DDL 时，不接受`Float.precision`参数，因为 Oracle 不支持将浮点精度指定为小数位数。相反，请使用特定于 Oracle 的`FLOAT`数据类型，并指定`FLOAT.binary_precision`参数。这是 SQLAlchemy 2.0 中的新功能。

    要创建一个数据库不可知的`Float`，并为 Oracle 分别指定二进制精度，请使用`TypeEngine.with_variant()`如下：

    ```py
    from sqlalchemy import Column
    from sqlalchemy import Float
    from sqlalchemy.dialects import oracle

    Column(
        "float_data",
        Float(5).with_variant(oracle.FLOAT(binary_precision=16), "oracle")
    )
    ```

+   `asdecimal` – 与`Numeric`相同的标志，但默认为`False`。请注意，将此标志设置为`True`会导致浮点数转换。

+   `decimal_return_scale` – 将浮点数转换为 Python 十进制数时要使用的默认标度。由于十进制不准确，浮点值通常会更长，并且大多数浮点数据库类型没有“标度”概念，因此默认情况下，浮点类型在转换时会查找前十个小数位。指定此值将覆盖该长度。请注意，MySQL 浮点类型包括“标度”，如果没有另外指定，将使用“标度”作为 `decimal_return_scale` 的默认值。

```py
class sqlalchemy.types.FLOAT
```

SQL FLOAT 类型。

另请参阅

`Float` - 基本类型的文档。

**成员**

__init__() 

**类签名**

类 `sqlalchemy.types.FLOAT` （`sqlalchemy.types.Float`）。

```py
method __init__(precision: int | None = None, asdecimal: bool = False, decimal_return_scale: int | None = None)
```

*继承自* `Float` 的 `sqlalchemy.types.Float.__init__` *方法*

构造一个 Float。

参数：

+   `precision` –

    用于 DDL `CREATE TABLE` 中的数字精度。后端**应该**尝试确保此精度指示了通用 `Float` 数据类型的位数。

    注意

    对于 Oracle 后端，在渲染 DDL 时，不接受 `Float.precision` 参数，因为 Oracle 不支持将浮点精度指定为小数位数。而是使用 Oracle 特定的 `FLOAT` 数据类型，并指定 `FLOAT.binary_precision` 参数。这是 SQLAlchemy 版本 2.0 中的新功能。

    要创建一个与数据库无关的 `Float`，并为 Oracle 分别指定二进制精度，请使用 `TypeEngine.with_variant()` 如下：

    ```py
    from sqlalchemy import Column
    from sqlalchemy import Float
    from sqlalchemy.dialects import oracle

    Column(
        "float_data",
        Float(5).with_variant(oracle.FLOAT(binary_precision=16), "oracle")
    )
    ```

+   `asdecimal` – 与 `Numeric` 相同的标志，但默认为 `False`。请注意，将此标志设置为 `True` 会导致浮点转换。

+   `decimal_return_scale` – 将浮点数转换为 Python 十进制数时要使用的默认标度。由于十进制不准确，浮点值通常会更长，并且大多数浮点数据库类型没有“标度”概念，因此默认情况下，浮点类型在转换时会查找前十个小数位。指定此值将覆盖该长度。请注意，MySQL 浮点类型包括“标度”，如果没有另外指定，将使用“标度”作为 `decimal_return_scale` 的默认值。

```py
attribute sqlalchemy.types.INT
```

`INTEGER` 的别名

```py
class sqlalchemy.types.JSON
```

表示 SQL JSON 类型。

注意

`JSON` 被提供为供应商特定的 JSON 类型的外观。由于它支持 JSON SQL 操作，因此它仅适用于具有实际 JSON 类型的后端，目前有:

+   PostgreSQL - 有关特定于后端的注意事项，请参阅 `sqlalchemy.dialects.postgresql.JSON` 和 `sqlalchemy.dialects.postgresql.JSONB`

+   MySQL - 有关特定于后端的注意事项，请参阅 `sqlalchemy.dialects.mysql.JSON`

+   SQLite 截至版本 3.9 - 有关特定于后端的注意事项，请参阅 `sqlalchemy.dialects.sqlite.JSON`

+   Microsoft SQL Server 2016 及更高版本 - 有关特定于后端的注意事项，请参阅 `sqlalchemy.dialects.mssql.JSON`

`JSON` 是核心的一部分，支持原生 JSON 数据类型日益增长的流行度。

`JSON` 类型存储任意的 JSON 格式数据，例如:

```py
data_table = Table('data_table', metadata,
    Column('id', Integer, primary_key=True),
    Column('data', JSON)
)

with engine.connect() as conn:
    conn.execute(
        data_table.insert(),
        {"data": {"key1": "value1", "key2": "value2"}}
    )
```

**JSON 特定的表达式运算符**

`JSON` 数据类型提供了这些额外的 SQL 操作:

+   键索引操作:

    ```py
    data_table.c.data['some key']
    ```

+   整数索引操作:

    ```py
    data_table.c.data[3]
    ```

+   路径索引操作:

    ```py
    data_table.c.data[('key_1', 'key_2', 5, ..., 'key_n')]
    ```

+   针对特定 JSON 元素类型的数据转换器，在调用索引或路径操作后:

    ```py
    data_table.c.data["some key"].as_integer()
    ```

    1.3.11 版中的新功能。

可能还可以从 `JSON` 的特定于方言的版本中获得其他操作，例如 `sqlalchemy.dialects.postgresql.JSON` 和 `sqlalchemy.dialects.postgresql.JSONB`，它们都提供了额外的特定于 PostgreSQL 的操作。

**将 JSON 元素转换为其他类型**

索引操作，即通过使用 Python 方括号操作符调用表达式，如 `some_column['some key']`，将返回一个类型默认为 `JSON` 的表达式对象，默认情况下，以便可以调用进一步的面向 JSON 的指令。然而，更常见的情况是希望索引操作返回特定的标量元素，例如字符串或整数。为了以与后端无关的方式提供对这些元素的访问，提供了一系列数据转换器:

+   `Comparator.as_string()` - 将元素作为字符串返回

+   `Comparator.as_boolean()` - 将元素返回为布尔值

+   `Comparator.as_float()` - 将元素返回为浮点数

+   `Comparator.as_integer()` - 将元素返回为整数

这些数据转换器由支持方言实现，以确保与上述类型的比较将按预期工作，例如：

```py
# integer comparison
data_table.c.data["some_integer_key"].as_integer() == 5

# boolean comparison
data_table.c.data["some_boolean"].as_boolean() == True
```

版本 1.3.11 中新增了基本 JSON 数据元素类型的特定类型转换器。

注意

数据转换器函数是 1.3.11 版本中的新功能，取代了以前记录的使用 CAST 的方法；作为参考，这看起来像是：

```py
from sqlalchemy import cast, type_coerce
from sqlalchemy import String, JSON
cast(
    data_table.c.data['some_key'], String
) == type_coerce(55, JSON)
```

上述情况现在直接有效为：

```py
data_table.c.data['some_key'].as_integer() == 5
```

有关 1.3.x 系列中先前比较方法的详细信息，请参阅 SQLAlchemy 1.2 的文档或版本分发的 doc/目录中包含的 HTML 文件。

**当使用 ORM 时检测 JSON 列中的更改**

当与 SQLAlchemy ORM 一起使用时，`JSON` 类型不会检测到对结构的原地突变。为了检测这些情况，必须使用`sqlalchemy.ext.mutable`扩展，通常使用`MutableDict`类。此扩展将允许对数据结构进行“原地”更改以产生事件，这些事件将被工作单元检测到。有关涉及字典的简单示例，请参见`HSTORE`中的示例。

或者，将 JSON 结构分配给替换旧结构的 ORM 元素将始终触发更改事件。

**支持 JSON null 与 SQL NULL**

在处理 NULL 值时，`JSON` 类型建议使用两个特定的常量来区分一个计算为 SQL NULL 的列，例如，没有值，与 JSON 编码的字符串`"null"`。要插入或选择 SQL NULL 的值，请使用常量`null()`。当使用包含特殊逻辑的`JSON`数据类型时，此符号可以作为参数值传递，该逻辑解释此符号表示列值应为 SQL NULL，而不是 JSON 的`"null"`：

```py
from sqlalchemy import null
conn.execute(table.insert(), {"json_value": null()})
```

要插入或选择 JSON 的值为`"null"`，请使用常量`JSON.NULL`：

```py
conn.execute(table.insert(), {"json_value": JSON.NULL})
```

`JSON` 类型支持一个标志 `JSON.none_as_null`，当设置为 True 时，Python 常量 `None` 将评估为 SQL NULL 的值，当设置为 False 时，Python 常量 `None` 将评估为 JSON `"null"` 的值。Python 值 `None` 可以与 `JSON.NULL` 和 `null()` 结合使用，以指示 NULL 值，但在这些情况下必须注意 `JSON.none_as_null` 的值。

**自定义 JSON 序列化器**

JSON 序列化器和反序列化器使用了`JSON`默认的 Python `json.dumps` 和 `json.loads` 函数；在 psycopg2 方言的情况下，psycopg2 可能会使用自己的自定义加载器函数。

为了影响序列化器/反序列化器，它们目前可以在 `create_engine()` 级别通过 `create_engine.json_serializer` 和 `create_engine.json_deserializer` 参数进行配置。例如，要关闭 `ensure_ascii`：

```py
engine = create_engine(
    "sqlite://",
    json_serializer=lambda obj: json.dumps(obj, ensure_ascii=False))
```

自版本 1.3.7 起更改：SQLite 方言的 `json_serializer` 和 `json_deserializer` 参数从 `_json_serializer` 和 `_json_deserializer` 重命名。

另请参阅

`sqlalchemy.dialects.postgresql.JSON`

`sqlalchemy.dialects.postgresql.JSONB`

`sqlalchemy.dialects.mysql.JSON`

`sqlalchemy.dialects.sqlite.JSON`

**成员**

as_boolean(), as_float(), as_integer(), as_json(), as_numeric(), as_string(), bind_processor(), literal_processor(), NULL, __init__(), bind_processor(), comparator_factory, hashable, python_type, result_processor(), should_evaluate_none

**类签名**

类 `sqlalchemy.types.JSON` (`sqlalchemy.types.Indexable`, `sqlalchemy.types.TypeEngine`)

```py
class Comparator
```

为 `JSON` 定义比较操作。

**类签名**

类 `sqlalchemy.types.JSON.Comparator` (`sqlalchemy.types.Comparator`, `sqlalchemy.types.Comparator`)

```py
method as_boolean()
```

将索引值转换为布尔值。

例如：

```py
stmt = select(
    mytable.c.json_column['some_data'].as_boolean()
).where(
    mytable.c.json_column['some_data'].as_boolean() == True
)
```

从 1.3.11 版本开始新增。

```py
method as_float()
```

将索引值转换为浮点数。

例如：

```py
stmt = select(
    mytable.c.json_column['some_data'].as_float()
).where(
    mytable.c.json_column['some_data'].as_float() == 29.75
)
```

从 1.3.11 版本开始新增。

```py
method as_integer()
```

将索引值转换为整数。

例如：

```py
stmt = select(
    mytable.c.json_column['some_data'].as_integer()
).where(
    mytable.c.json_column['some_data'].as_integer() == 5
)
```

从 1.3.11 版本开始新增。

```py
method as_json()
```

将索引值转换为 JSON。

例如：

```py
stmt = select(mytable.c.json_column['some_data'].as_json())
```

这通常是任何情况下索引元素的默认行为。

请注意，并非所有后端都支持对完整 JSON 结构的比较。

从 1.3.11 版本开始新增。

```py
method as_numeric(precision, scale, asdecimal=True)
```

将索引值转换为数值/小数。

例如：

```py
stmt = select(
    mytable.c.json_column['some_data'].as_numeric(10, 6)
).where(
    mytable.c.
    json_column['some_data'].as_numeric(10, 6) == 29.75
)
```

从 1.4.0b2 版本开始新增。

```py
method as_string()
```

将索引值转换为字符串。

例如：

```py
stmt = select(
    mytable.c.json_column['some_data'].as_string()
).where(
    mytable.c.json_column['some_data'].as_string() ==
    'some string'
)
```

从 1.3.11 版本开始新增。

```py
class JSONElementType
```

JSON 表达式中索引/路径元素的常见函数。

**类签名**

类 `sqlalchemy.types.JSON.JSONElementType` (`sqlalchemy.types.TypeEngine`)

```py
method bind_processor(dialect)
```

返回用于处理绑定值的转换函数。

返回一个可调用对象，该对象将接收一个绑定参数值作为唯一位置参数，并返回一个要发送到 DB-API 的值。

如果不需要处理，则方法应返回 `None`。

注意

此方法仅相对于**特定方言类型对象**调用，该对象通常是**私有于正在使用的方言**的，并且不是公共类型对象，这意味着不可通过子类化`TypeEngine`类来提供替代的`TypeEngine.bind_processor()`方法，除非显式子类化`UserDefinedType`类。

要为`TypeEngine.bind_processor()`提供替代行为，请实现一个`TypeDecorator`类，并提供一个`TypeDecorator.process_bind_param()`的实现。

另请参阅

扩展现有类型

参数：

**方言** - 正在使用的方言实例。

```py
method literal_processor(dialect)
```

返回一个用于处理直接呈现而不使用绑定的字面值的转换函数。

当编译器使用“literal_binds”标志时，通常在 DDL 生成以及某些后端不接受绑定参数的情况下使用此函数。

返回一个可调用对象，该对象将接收一个字面的 Python 值作为唯一的位置参数，并返回一个字符串表示，以在 SQL 语句中呈现。

注意

此方法仅相对于**特定方言类型对象**调用，该对象通常是**私有于正在使用的方言**的，并且不是公共类型对象，这意味着不可通过子类化`TypeEngine`类来提供替代的`TypeEngine.literal_processor()`方法，除非显式子类化`UserDefinedType`类。

要为`TypeEngine.literal_processor()`提供替代行为，请实现一个`TypeDecorator`类，并提供一个`TypeDecorator.process_literal_param()`的实现。

另请参阅

扩展现有类型

```py
class JSONIndexType
```

JSON 索引值的数据类型的占位符。

这允许对特殊语法的 JSON 索引值进行执行时处理。

**类签名**

类`sqlalchemy.types.JSON.JSONIndexType` (`sqlalchemy.types.JSONElementType`)

```py
class JSONIntIndexType
```

JSON 索引值的数据类型的占位符。

这允许对特殊语法的 JSON 索引值进行执行时处理。

**类签名**

类`sqlalchemy.types.JSON.JSONIntIndexType` (`sqlalchemy.types.JSONIndexType`)

```py
class JSONPathType
```

JSON 路径操作的占位符类型。

这允许对基于路径的索引值进行执行时处理，转换为特定的 SQL 语法。

**类签名**

类`sqlalchemy.types.JSON.JSONPathType` (`sqlalchemy.types.JSONElementType`)

```py
class JSONStrIndexType
```

JSON 索引值的数据类型的占位符。

这允许对特殊语法的 JSON 索引值进行执行时处理。

**类签名**

类`sqlalchemy.types.JSON.JSONStrIndexType` (`sqlalchemy.types.JSONIndexType`)

```py
attribute NULL = symbol('JSON_NULL')
```

描述 NULL 的 JSON 值。

此值用于强制使用 JSON 值`"null"`作为值。Python 的`None`值将根据`JSON.none_as_null` 标志的设置被识别为 SQL NULL 或 JSON`"null"`，常量`JSON.NULL` 可以始终解析为 JSON`"null"`，而不考虑此设置。这与总是解析为 SQL NULL 的`null()` 构造形成对比。例如：

```py
from sqlalchemy import null
from sqlalchemy.dialects.postgresql import JSON

# will *always* insert SQL NULL
obj1 = MyObject(json_value=null())

# will *always* insert JSON string "null"
obj2 = MyObject(json_value=JSON.NULL)

session.add_all([obj1, obj2])
session.commit()
```

为了将 JSON NULL 设置为列的默认值，最透明的方法是使用`text()`：

```py
Table(
    'my_table', metadata,
    Column('json_data', JSON, default=text("'null'"))
)
```

虽然在这种情况下可以使用`JSON.NULL`，但`JSON.NULL` 值将作为列的值返回，这在 ORM 或其他重新使用默认值的情况下可能不理想。使用 SQL 表达式意味着值将在检索生成的默认值的上下文中从数据库重新获取。

```py
method __init__(none_as_null: bool = False)
```

构造一个`JSON` 类型。

参数：

**none_as_null=False** –

如果为 True，则将值`None`持久化为 SQL NULL 值，而不是`null`的 JSON 编码。请注意，当此标志为 False 时，`null()` 构造仍然可以用于持久化 NULL 值，可以直接作为参数值传递，由`JSON` 类型特别解释为 SQL NULL：

```py
from sqlalchemy import null
conn.execute(table.insert(), {"data": null()})
```

注意

`JSON.none_as_null` 不适用于传递给 `Column.default` 和 `Column.server_default` 的值；对于这些参数传递的`None`值意味着“没有默认值”。

此外，在 SQL 比较表达式中使用时，Python 值`None`仍然指向 SQL null，而不是 JSON NULL。`JSON.none_as_null` 标志明确指的是值在 INSERT 或 UPDATE 语句中的**持久性**。应该使用`JSON.NULL`值用于希望与 JSON null 进���比较的 SQL 表达式。

参见

`JSON.NULL`

```py
method bind_processor(dialect)
```

返回一个用于处理绑定值的转换函数。

返回一个可调用对象，该对象将接收一个绑定参数值作为唯一的位置参数，并返回一个要发送到 DB-API 的值。

如果不需要处理，则该方法应返回`None`。

注意

此方法仅相对于**特定方言类型对象**调用，该对象通常**私有于正在使用的方言**，并且不是公共类型对象，这意味着无法通过子类化`TypeEngine`类来提供替代的`TypeEngine.bind_processor()`方法，除非显式子类化`UserDefinedType`类。

要为`TypeEngine.bind_processor()`提供替代行为，请实现一个`TypeDecorator`类，并提供一个`TypeDecorator.process_bind_param()`的实现。

参见

增强现有类型

参数：

**方言** – 正在使用的方言实例。

```py
attribute comparator_factory
```

`Comparator` 的别名。

```py
attribute hashable = False
```

标志，如果为 False，则表示该类型的值不可哈希。

在 ORM 中用于去重结果列表。

```py
attribute python_type
```

```py
method result_processor(dialect, coltype)
```

返回一个用于处理结果行值的转换函数。

返回一个可调用对象，该对象将接收结果行列值作为唯一的位置参数，并将返回一个要返回给用户的值。

如果不需要处理，则该方法应返回 `None`。

注意

该方法仅相对于**特定方言类型对象**调用，该对象通常是**正在使用的方言的私有对象**，并且不是公共类型对象，这意味着无法通过继承 `TypeEngine` 类来提供替代 `TypeEngine.result_processor()` 方法，除非显式地继承 `UserDefinedType` 类。

要为 `TypeEngine.result_processor()` 提供替代行为，请实现一个 `TypeDecorator` 类并提供 `TypeDecorator.process_result_value()` 的实现。

另请参阅

扩展现有类型

参数：

+   `dialect` – 正在使用的方言实例。

+   `coltype` – 在 cursor.description 中接收的 DBAPI coltype 参数。

```py
attribute should_evaluate_none: bool
```

如果为 True，则 Python 常量 `None` 被视为由此类型显式处理。

ORM 使用此标志指示在 INSERT 语句中传递了 None 的正值到列中，而不是省略了 INSERT 语句中的列，这会触发列级默认值。它还允许具有对 Python None 的特殊行为（例如 JSON 类型）的类型指示它们希望显式处理 None 值。

要在现有类型上设置此标志，请使用 `TypeEngine.evaluates_none()` 方法。

另请参阅

`TypeEngine.evaluates_none()` 方法。

```py
class sqlalchemy.types.INTEGER
```

SQL INT 或 INTEGER 类型。

另请参阅

`Integer` - 基本类型的文档。

**类签名**

类 `sqlalchemy.types.INTEGER`（`sqlalchemy.types.Integer`）

```py
class sqlalchemy.types.NCHAR
```

SQL NCHAR 类型。

**成员**

__init__()

**类签名**

类 `sqlalchemy.types.NCHAR` (`sqlalchemy.types.Unicode`)

```py
method __init__(length: int | None = None, collation: str | None = None)
```

*继承自* `String` 的 `sqlalchemy.types.String.__init__` *方法*

创建一个持有字符串的类型。

参数：

+   `length` – 可选，用于 DDL 和 CAST 表达式中的列的长度。如果不会发出 `CREATE TABLE`，则可以安全地省略。某些数据库可能需要在 DDL 中使用长度，并且如果包含了没有长度的 `VARCHAR`，则在发出 `CREATE TABLE` DDL 时会引发异常。值是以字节还是字符解释是与数据库特定的。

+   `collation` –

    可选，用于 DDL 和 CAST 表达式中的列级排序规则。在 SQLite、MySQL 和 PostgreSQL 中支持使用 COLLATE 关键字进行渲染。例如：

    ```py
    >>> from sqlalchemy import cast, select, String
    >>> print(select(cast('some string', String(collation='utf8'))))
    SELECT  CAST(:param_1  AS  VARCHAR  COLLATE  utf8)  AS  anon_1 
    ```

    注意

    在大多数情况下，`Unicode` 或 `UnicodeText` 数据类型应该用于预期存储非 ASCII 数据的 `Column`。这些数据类型将确保在数据库上使用正确的类型。

```py
class sqlalchemy.types.NVARCHAR
```

SQL NVARCHAR 类型。

**成员**

__init__()

**类签名**

类 `sqlalchemy.types.NVARCHAR` (`sqlalchemy.types.Unicode`)

```py
method __init__(length: int | None = None, collation: str | None = None)
```

*继承自* `String` 的 `sqlalchemy.types.String.__init__` *方法*

创建一个持有字符串的类型。

参数：

+   `length` – 可选，用于 DDL 和 CAST 表达式中的列的长度。如果不会发出 `CREATE TABLE`，则可以安全地省略。某些数据库可能需要在 DDL 中使用长度，并且如果包含了没有长度的 `VARCHAR`，则在发出 `CREATE TABLE` DDL 时会引发异常。值是以字节还是字符解释是与数据库特定的。

+   `collation` –

    可选，用于 DDL 和 CAST 表达式中的列级排序规则。在 SQLite、MySQL 和 PostgreSQL 中支持使用 COLLATE 关键字进行渲染。例如：

    ```py
    >>> from sqlalchemy import cast, select, String
    >>> print(select(cast('some string', String(collation='utf8'))))
    SELECT  CAST(:param_1  AS  VARCHAR  COLLATE  utf8)  AS  anon_1 
    ```

    注意

    在大多数情况下，`Unicode` 或 `UnicodeText` 数据类型应该用于预期存储非 ASCII 数据的 `Column`。这些数据类型将确保在数据库上使用正确的类型。

```py
class sqlalchemy.types.NUMERIC
```

SQL NUMERIC 类型。

另请参阅

`Numeric` - 基础类型的文档。

**成员**

__init__()

**类签名**

类 `sqlalchemy.types.NUMERIC`（`sqlalchemy.types.Numeric`）

```py
method __init__(precision: int | None = None, scale: int | None = None, decimal_return_scale: int | None = None, asdecimal: bool = True)
```

*继承自* `Numeric` *的* `sqlalchemy.types.Numeric.__init__` *方法*

构造一个 Numeric。

参数：

+   `precision` - 用于 DDL `CREATE TABLE` 中的数值精度。

+   `scale` - 用于 DDL `CREATE TABLE` 中的数值标度。

+   `asdecimal` - 默认为 True。返回值应该作为 Python Decimal 对象还是作为浮点数发送。根据数据类型不同，不同的 DBAPIs 会发送其中之一 - Numeric 类型将确保返回值在各个 DBAPIs 之间一致地是其中之一。

+   `decimal_return_scale` - 转换浮点数为 Python decimal 时使用的默认精度。由于浮点数的不准确性，浮点数值通常会更长，而大多数浮点数数据库类型没有“标度”概念，因此默认情况下，float 类型在转换时查找前十位小数点。指定此值将覆盖该长度。包含显式“.scale”值的类型，例如基本 `Numeric` 和 MySQL 浮点类型，如果未另行指定，将使用“.scale”的值作为 decimal_return_scale 的默认值。

使用 `Numeric` 类型时，应注意确保 asdecimal 设置适用于所使用的 DBAPI - 当 Numeric 应用从 Decimal->float 或 float-> Decimal 的转换时，此转换会为所有接收到的结果列增加额外的性能开销。

本地返回 Decimal 的 DBAPI（例如 psycopg2）将通过设置为 `True` 获得更好的准确性和更高的性能，因为本地转换为 Decimal 可降低浮点数问题的数量，并且 Numeric 类型本身不需要应用任何进一步的转换。但是，另一个本地返回浮点数的 DBAPI *将* 需要额外的转换开销，并且仍然可能存在浮点数据丢失 - 在这种情况下，`asdecimal=False` 将至少消除额外的转换开销。

```py
class sqlalchemy.types.REAL
```

SQL REAL 类型。

另请参阅

`Float` - 基本类型的文档。

**成员**

__init__()

**类签名**

类 `sqlalchemy.types.REAL`（`sqlalchemy.types.Float`）

```py
method __init__(precision: int | None = None, asdecimal: bool = False, decimal_return_scale: int | None = None)
```

*继承自* `Float` *的* `sqlalchemy.types.Float.__init__` *方法*

构造一个 Float。

参数：

+   `precision` -

    用于 DDL `CREATE TABLE` 中的数值精度。后端**应**尽量确保此精度指示了通用 `Float` 数据类型的数字位数。

    注意

    对于 Oracle 后端，在渲染 DDL 时不接受`Float.precision`参数，因为 Oracle 不支持以小数位数指定浮点精度。而是使用特定于 Oracle 的`FLOAT`数据类型，并指定`FLOAT.binary_precision`参数。这是 SQLAlchemy 2.0 版本的新功能。

    要创建一个与 Oracle 分开指定二进制精度的数据库不可知的`Float`，请使用`TypeEngine.with_variant()`如下所示：

    ```py
    from sqlalchemy import Column
    from sqlalchemy import Float
    from sqlalchemy.dialects import oracle

    Column(
        "float_data",
        Float(5).with_variant(oracle.FLOAT(binary_precision=16), "oracle")
    )
    ```

+   `asdecimal` – 与`Numeric`相同的标志，但默认为`False`。请注意，将此标志设置为`True`会导致浮点转换。

+   `decimal_return_scale` – 将浮点数转换为 Python 十进制数时使用的默认精度。由于十进制不准确，浮点值通常会更长，并且大多数浮点数据库类型都没有“精度”的概念，因此默认情况下，浮点类型在转换时会查找前十位小数。指定此值将覆盖该长度。请注意，如果未另行指定，则 MySQL 浮点类型将使用“精度”作为 decimal_return_scale 的默认值。

```py
class sqlalchemy.types.SMALLINT
```

SQL SMALLINT 类型。

另请参阅

`SmallInteger` - 基础类型的文档。

**类签名**

类`sqlalchemy.types.SMALLINT` (`sqlalchemy.types.SmallInteger`)

```py
class sqlalchemy.types.TEXT
```

SQL TEXT 类型。

**成员**

__init__()

**类签名**

类`sqlalchemy.types.TEXT` (`sqlalchemy.types.Text`)

```py
method __init__(length: int | None = None, collation: str | None = None)
```

*继承自* `String` *的* `sqlalchemy.types.String.__init__` *方法*

创建一个保存字符串的类型。

参数：

+   `length` – 可选项，在 DDL 和 CAST 表达式中用于列的长度。如果不会发出`CREATE TABLE`，可以安全地省略。某些数据库可能需要在 DDL 中使用长度，并且如果包括没有长度的`VARCHAR`，则在发出`CREATE TABLE` DDL 时会引发异常。值是按字节还是字符解释是特定于数据库的。

+   `collation` –

    可选项，在 DDL 和 CAST 表达式中使用的列级排序规则。使用 SQLite、MySQL 和 PostgreSQL 支持的 COLLATE 关键字呈现。例如：

    ```py
    >>> from sqlalchemy import cast, select, String
    >>> print(select(cast('some string', String(collation='utf8'))))
    SELECT  CAST(:param_1  AS  VARCHAR  COLLATE  utf8)  AS  anon_1 
    ```

    注意

    在大多数情况下，应该为预期存储非 ASCII 数据的 `Column` 使用 `Unicode` 或 `UnicodeText` 数据类型。这些数据类型将确保在数据库上使用正确的类型。

```py
class sqlalchemy.types.TIME
```

SQL TIME 类型。

**类签名**

类 `sqlalchemy.types.TIME`（`sqlalchemy.types.Time`）

```py
class sqlalchemy.types.TIMESTAMP
```

SQL TIMESTAMP 类型。

`TIMESTAMP` 数据类型在一些后端（如 PostgreSQL 和 Oracle）上支持时区存储。请使用 `TIMESTAMP.timezone` 参数以在这些后端上启用“TIMESTAMP WITH TIMEZONE”。

**成员**

__init__()，get_dbapi_type()

**类签名**

类 `sqlalchemy.types.TIMESTAMP`（`sqlalchemy.types.DateTime`）

```py
method __init__(timezone: bool = False)
```

构造一个新的`TIMESTAMP`。

参数：

**timezone** – 布尔值。指示 TIMESTAMP 类型应该在目标数据库上启用时区支持（如果可用）。在每个方言上类似于“TIMESTAMP WITH TIMEZONE”。如果目标数据库不支持时区，则此标志将被忽略。

```py
method get_dbapi_type(dbapi)
```

返回底层 DB-API 的相应类型对象（如果有）。

这对于调用 `setinputsizes()` 等操作可能很有用。

```py
class sqlalchemy.types.UUID
```

表示 SQL UUID 类型。

这是与以前的仅限于 PostgreSQL 版本的 `UUID` 向后兼容的 SQL 本地形式的 `Uuid` 数据库无关数据类型。

`UUID` 数据类型仅适用于具有名为 `UUID` 的 SQL 数据类型的数据库。对于没有这个确切命名类型的后端，包括 SQL Server，在其上不起作用。对于具有本机支持的与后端无关的 UUID 值，包括 SQL Server 的 `UNIQUEIDENTIFIER` 数据类型，请使用 `Uuid` 数据类型。

新版本 2.0 中的新增内容。

另请参见

`Uuid`

**成员**

__init__()

**类签名**

类 `sqlalchemy.types.UUID`（`sqlalchemy.types.Uuid`，`sqlalchemy.types.NativeForEmulated`）

```py
method __init__(as_uuid: bool = True)
```

构造一个 `UUID` 类型。

参数：

**as_uuid=True** –

如果为 True，则值将被解释为 Python uuid 对象，并通过 DBAPI 转换为/从字符串。

```py
class sqlalchemy.types.VARBINARY
```

SQL VARBINARY 类型。

**类签名**

类`sqlalchemy.types.VARBINARY`（`sqlalchemy.types._Binary`）

```py
class sqlalchemy.types.VARCHAR
```

SQL VARCHAR 类型。

**成员**

__init__()

**类签名**

类`sqlalchemy.types.VARCHAR`（`sqlalchemy.types.String`）

```py
method __init__(length: int | None = None, collation: str | None = None)
```

*从* `String` *的* `sqlalchemy.types.String.__init__` *方法继承*

创建一个持有字符串的类型。

参数：

+   `length` – 可选，用于 DDL 和 CAST 表达式中的列的长度。 如果不会发出`CREATE TABLE`，可以安全地省略。 某些数据库可能需要在 DDL 中使用`length`，如果包含没有长度的`VARCHAR`，则在发出`CREATE TABLE` DDL 时会引发异常。 值是以字节还是字符解释是与数据库特定的。

+   `collation` –

    可选，用于 DDL 和 CAST 表达式中的列级排序规则。 使用 SQLite、MySQL 和 PostgreSQL 支持的 COLLATE 关键字进行呈现。 例如：

    ```py
    >>> from sqlalchemy import cast, select, String
    >>> print(select(cast('some string', String(collation='utf8'))))
    SELECT  CAST(:param_1  AS  VARCHAR  COLLATE  utf8)  AS  anon_1 
    ```

    注意

    在大多数情况下，应该使用`Unicode`或`UnicodeText`数据类型来存储非 ASCII 数据的`Column`。 这些数据类型将确保在数据库上使用正确的类型。

## “驼峰大小写”数据类型

初级类型具有“驼峰大小写”名称，如`String`、`Numeric`、`Integer`和`DateTime`。 所有`TypeEngine`的直接子类都是“驼峰大小写”类型。 “驼峰大小写”类型在尽可能大的程度上是**与数据库无关的**，这意味着它们可以在任何数据库后端上使用，在那里它们将以适当的方式行事，以产生所需的行为。

一个简单的“驼峰大小写”数据类型的例子是`String`。 在大多数后端上，使用此数据类型在 table specification 中将对应于在目标后端上使用的`VARCHAR`数据库类型，将字符串值传递到数据库中，如下例所示：

```py
from sqlalchemy import MetaData
from sqlalchemy import Table, Column, Integer, String

metadata_obj = MetaData()

user = Table(
    "user",
    metadata_obj,
    Column("user_name", String, primary_key=True),
    Column("email_address", String(60)),
)
```

在`Table` 定义或整体 SQL 表达式中使用特定的 `TypeEngine` 类时，如果不需要参数，可以将其作为类本身传递，即，不用 `()` 实例化。如果需要参数，比如上面的 `"email_address"` 列中的长度参数 60，可以实例化该类型。

另一个表达更多后端特定行为的“CamelCase” 数据类型是 `Boolean` 数据类型。与 `String` 不同，后端并非每一个都有真正的“boolean”数据类型；有些使用整数或位值 0 和 1，有些具有布尔字面常量 `true` 和 `false`，而其他的则不具备。对于这种数据类型，`Boolean` 在后端（如 PostgreSQL）可能呈现为 `BOOLEAN`，在 MySQL 后端为 `BIT`，在 Oracle 中为 `SMALLINT`。根据使用的方言，作为使用该类型从数据库发送和接收数据的结果，它可能会解释 Python 数值或布尔值。

典型的 SQLAlchemy 应用程序可能会主要使用一般情况下的“CamelCase” 类型，因为它们通常提供最佳的基本行为，并且可以自动移植到所有后端。

通用的“CamelCase” 数据类型的参考在下面的 通用“CamelCase” 类型 处。

## “UPPERCASE” 数据类型

与“CamelCase” 类型相反的是“UPPERCASE” 数据类型。这些数据类型始终继承自特定的“CamelCase” 数据类型，并且始终表示**精确的**数据类型。在使用“UPPERCASE” 数据类型时，类型的名称始终如实呈现，不考虑当前后端是否支持它。因此，在 SQLAlchemy 应用程序中使用“UPPERCASE” 类型表明需要特定的数据类型，这意味着应用程序通常（在没有采取其他步骤的情况下）会受限于以给定类型精确使用的后端。UPPERCASE 类型的示例包括 `VARCHAR`、`NUMERIC`、`INTEGER` 和 `TIMESTAMP`，它们直接继承自先前提到的“CamelCase” 类型 `String`、`Numeric`、`Integer` 和 `DateTime`。

`sqlalchemy.types`中的“大写”数据类型是常见的 SQL 类型，通常希望至少在两个后端上可用，如果不是更多。

一般的“大写”数据类型的参考如下所示：SQL 标准和多供应商“大写”类型。

## 后端特定“大写”数据类型

大多数数据库还具有自己的数据类型，这些数据类型要么完全特定于这些数据库，要么添加了特定于这些数据库的附加参数。对于这些数据类型，特定的 SQLAlchemy 方言提供**特定于后端**的“大写”数据类型，用于没有在其他后端上有类似物的 SQL 类型。后端特定大写数据类型的示例包括 PostgreSQL 的`JSONB`、SQL Server 的`IMAGE`和 MySQL 的`TINYTEXT`。

特定后端可能还包括“大写”数据类型，这些数据类型扩展了`sqlalchemy.types`模块中同一“大写”数据类型可用的参数。例如，在创建 MySQL 字符串数据类型时，可能希望指定 MySQL 特定参数，如`charset`或`national`，这些参数可从 MySQL 版本的`VARCHAR`中获得，作为 MySQL 专用参数`VARCHAR.charset`和`VARCHAR.national`。

后端特定类型的 API 文档位于方言特定文档中，在 Dialects 列出。

## 对于多个后端使用“大写”和后端特定类型

查看“大写”和“CamelCase”类型的存在，自然会引出如何利用后端特定选项使用“大写”数据类型的用例，但仅当该后端正在使用时。为了将数据库不可知的“CamelCase”和后端特定的“大写”系统联系在一起，可以使用`TypeEngine.with_variant()`方法将类型组合在一起，以便在特定后端上使用特定行为。

例如，要使用 `String` 数据类型，但在 MySQL 上运行时要利用 `VARCHAR.charset` 参数的 `VARCHAR` 在创建表时在 MySQL 或 MariaDB 上使用时，可以使用 `TypeEngine.with_variant()` 如下所示：

```py
from sqlalchemy import MetaData
from sqlalchemy import Table, Column, Integer, String
from sqlalchemy.dialects.mysql import VARCHAR

metadata_obj = MetaData()

user = Table(
    "user",
    metadata_obj,
    Column("user_name", String(100), primary_key=True),
    Column(
        "bio",
        String(255).with_variant(VARCHAR(255, charset="utf8"), "mysql", "mariadb"),
    ),
)
```

在上述表定义中，`"bio"`列将在所有后端具有字符串行为。在大多数后端上，它将呈现为 `DDL`，例如 `VARCHAR`。然而，在 MySQL 和 MariaDB（通过以 `mysql` 或 `mariadb` 开头的数据库 URL 表示）上，它将呈现为 `VARCHAR(255) CHARACTER SET utf8`。

另请参见

`TypeEngine.with_variant()` - 附加使用示例和说明

## 通用“驼峰命名”类型

通用类型指定可以读取、写入和存储特定类型的 Python 数据的列。当发出 `CREATE TABLE` 语句时，SQLAlchemy 将选择目标数据库上可用的最佳数据库列类型。要完全控制在 `CREATE TABLE` 中发出的列类型，例如 `VARCHAR`，请参阅 SQL 标准和多供应商“大写”类型 和本章的其他部分。

| 对象名称 | 描述 |
| --- | --- |
| BigInteger | 用于更大的 `int` 整数的类型。 |
| Boolean | 布尔数据类型。 |
| Date | 用于 `datetime.date()` 对象的类型。 |
| DateTime | 用于 `datetime.datetime()` 对象的类型。 |
| Double | 用于双精度 `FLOAT` 浮点类型的类型。 |
| Enum | 通用枚举类型。 |
| Float | 表示浮点类型，例如 `FLOAT` 或 `REAL` 的类型。 |
| Integer | 用于 `int` 整数的类型。 |
| Interval | 用于 `datetime.timedelta()` 对象的类型。 |
| LargeBinary | 用于大型二进制字节数据的类型。 |
| MatchType | 指 MATCH 运算符的返回类型。 |
| Numeric | 非整数数值类型的基础，例如 `NUMERIC`、`FLOAT`、`DECIMAL` 和其他变体。 |
| PickleType | 存储使用 pickle 序列化的 Python 对象。 |
| SchemaType | 为类型添加功能，允许将模式级 DDL 与类型关联起来。 |
| SmallInteger | 用于较小的 `int` 整数的类型。 |
| String | 所有字符串和字符类型的基类。 |
| Text | 可变大小的字符串类型。 |
| Time | 用于 `datetime.time()` 对象的类型。 |
| Unicode | 一个可变长度的 Unicode 字符串类型。 |
| UnicodeText | 一个长度不受限制的 Unicode 字符串类型。 |
| Uuid | 表示数据库无关的 UUID 数据类型。 |

```py
class sqlalchemy.types.BigInteger
```

用于更大的 `int` 整数的类型。

通常在 DDL 中生成一个 `BIGINT`，在 Python 端表现得像一个普通的 `Integer`。

**类签名**

类 `sqlalchemy.types.BigInteger` (`sqlalchemy.types.Integer`)

```py
class sqlalchemy.types.Boolean
```

一个布尔值数据类型。

`Boolean` 通常在 DDL 端使用 BOOLEAN 或 SMALLINT，在 Python 端处理 `True` 或 `False`。

`Boolean` 数据类型当前有两个级别的断言，即持久化的值是简单的真/假值。对于所有后端，只接受 Python 值 `None`、`True`、`False`、`1` 或 `0` 作为参数值。对于那些不支持“原生布尔值”数据类型的后端，还可以选择在目标列上创建一个 CHECK 约束

从版本 1.2 开始：`Boolean` 数据类型现在断言传入的 Python 值已经是纯布尔形式。

**成员**

__init__(), bind_processor(), literal_processor(), python_type, result_processor()

**类签名**

类 `sqlalchemy.types.Boolean` (`sqlalchemy.types.SchemaType`, `sqlalchemy.types.Emulated`, `sqlalchemy.types.TypeEngine`)

```py
method __init__(create_constraint: bool = False, name: str | None = None, _create_events: bool = True, _adapted_from: SchemaType | None = None)
```

构造一个布尔值。

参数：

+   `create_constraint` –

    默认为 False。如果布尔值生成为 int/smallint，则在表上创建一个 CHECK 约束，确保值为 1 或 0。

    注意

    强烈建议 CHECK 约束具有明确的名称，以支持模式管理问题。这可以通过设置 `Boolean.name` 参数或设置适当的命名约定来实现；参见配置约束命名约定以获取背景信息。

    从版本 1.4 开始更改：- 此标志现在默认为 False，表示对于非本机枚举类型不生成 CHECK 约束。

+   `name` – 如果生成了 CHECK 约束，则指定约束的名称。

```py
method bind_processor(dialect)
```

返回一个用于处理绑定值的转换函数。

返回一个可调用对象，该对象将接收一个绑定参数值作为唯一的位置参数，并返回一个要发送到 DB-API 的值。

如果不需要处理，则方法应返回 `None`。

注意

仅相对于 **方言特定的类型对象** 调用此方法，该对象通常是 **当前使用的方言的私有对象**，并且不是公共面向的相同类型对象，这意味着不可通过子类化 `TypeEngine` 类来提供替代的 `TypeEngine.bind_processor()` 方法，除非显式地子类化 `UserDefinedType` 类。

要为 `TypeEngine.bind_processor()` 提供替代行为，需实现一个 `TypeDecorator` 类，并提供 `TypeDecorator.process_bind_param()` 的实现。

另请参阅

增强现有类型

参数：

**方言** – 当前使用的方言实例。

```py
method literal_processor(dialect)
```

返回一个用于处理直接渲染而不使用绑定的字面值的转换函数。

当编译器使用“literal_binds”标志时使用此函数，通常在 DDL 生成以及某些后端不接受绑定参数的情况下使用。

返回一个可调用对象，该对象将接收一个字面的 Python 值作为唯一的位置参数，并返回一个字符串表示以在 SQL 语句中渲染。

注意

仅相对于 **方言特定的类型对象** 调用此方法，该对象通常是 **当前使用的方言的私有对象**，并且不是公共面向的相同类型对象，这意味着不可通过子类化 `TypeEngine` 类来提供替代的 `TypeEngine.literal_processor()` 方法，除非显式地子类化 `UserDefinedType` 类。

要为 `TypeEngine.literal_processor()` 提供备用行为，实现一个 `TypeDecorator` 类，并提供一个 `TypeDecorator.process_literal_param()` 的实现。

另请参阅

增强现有类型

```py
attribute python_type
```

```py
method result_processor(dialect, coltype)
```

返回一个用于处理结果行值的转换函数。

返回一个可调用对象，该对象将接收一个结果行列值作为唯一的位置参数，并返回一个要返回给用户的值。

如果不需要处理，该方法应返回`None`。

注意

此方法仅相对于一个**方言特定的类型对象**调用，该对象通常是**私有于正在使用的方言**，并且不是公共面向的类型对象，这意味着不可能为了提供一个备用的`TypeEngine.result_processor()`方法而子类化一个`TypeEngine`类，除非显式地子类化`UserDefinedType`类。

要为 `TypeEngine.result_processor()` 提供备用行为，实现一个 `TypeDecorator` 类，并提供一个 `TypeDecorator.process_result_value()` 的实现。

另请参阅

增强现有类型

参数：

+   `dialect` – 正在使用的方言实例。

+   `coltype` – 在 cursor.description 中接收到的 DBAPI coltype 参数。

```py
class sqlalchemy.types.Date
```

一种用于`datetime.date()`对象的类型。

**成员**

get_dbapi_type()、literal_processor()、python_type

**类签名**

类`sqlalchemy.types.Date`（`sqlalchemy.types._RenderISO8601NoT`、`sqlalchemy.types.HasExpressionLookup`、`sqlalchemy.types.TypeEngine`）

```py
method get_dbapi_type(dbapi)
```

如果有的话，从底层的 DB-API 返回相应的类型对象。

例如，这对于调用`setinputsizes()`可能会有用。

```py
method literal_processor(dialect)
```

返回一个转换函数，用于处理直接渲染而不使用绑定的字面值。

当编译器使用“literal_binds”标志时使用此函数，通常用于 DDL 生成以及某些情况下后端不接受绑定参数的情况。

返回一个可调用对象，该对象将接收一个字面 Python 值作为唯一的位置参数，并返回一个字符串表示以在 SQL 语句中呈现。

注意

此方法仅相对于**特定方言类型对象**调用，该对象通常**私有于正在使用的方言**，并且不是公共类型对象，这意味着无法子类化`TypeEngine`类以提供替代的`TypeEngine.literal_processor()`方法，除非显式地子类化`UserDefinedType`类。

要为`TypeEngine.literal_processor()`提供替代行为，请实现一个`TypeDecorator`类，并提供一个`TypeDecorator.process_literal_param()`的实现。

另请参阅

增强现有类型

```py
attribute python_type
```

```py
class sqlalchemy.types.DateTime
```

用于`datetime.datetime()`对象的类型。

日期和时间类型返回来自 Python `datetime` 模块的对象。大多数 DBAPI 都内置支持 datetime 模块，但 SQLite 是个例外。在 SQLite 的情况下，日期和时间类型被存储为字符串，然后在返回行时转换回 datetime 对象。

对于 datetime 类型中的时间表示，某些后端包括其他选项，例如时区支持和分数秒支持。对于分数秒，使用方言特定的数据类型，例如`TIME`。对于时区支持，请至少使用`TIMESTAMP`数据类型，如果不是方言特定的数据类型对象的话。

**成员**

__init__(), get_dbapi_type(), literal_processor(), python_type

**类签名**

类`sqlalchemy.types.DateTime` (`sqlalchemy.types._RenderISO8601NoT`, `sqlalchemy.types.HasExpressionLookup`, `sqlalchemy.types.TypeEngine`)

```py
method __init__(timezone: bool = False)
```

构造一个新的`DateTime`。

参数：

**timezone** – boolean。指示日期时间类型是否应启用时区支持，仅适用于**基本日期/时间保持类型**。建议在使用此标志时直接使用`TIMESTAMP`数据类型，因为某些数据库包括与时区可用的 TIMESTAMP 数据类型不同的单独的通用日期/时间保持类型，如 Oracle。

```py
method get_dbapi_type(dbapi)
```

如果有的话，从底层 DB-API 返回相应的类型对象。

例如，这对调用`setinputsizes()`可能很有用。

```py
method literal_processor(dialect)
```

返回一个用于处理要直接呈现而不使用绑定的文字值的转换函数。

当编译器使用“literal_binds”标志时，将使用此函数，通常在 DDL 生成以及某些不接受绑定参数的后端场景中使用。

返回一个可调用对象，该对象将接收一个字面的 Python 值作为唯一的位置参数，并返回一个字符串表示，以在 SQL 语句中呈现。

注意

这种方法仅在**特定方言类型对象**的相对位置上调用，该对象通常**是使用的方言的私有部分**，并且与公共类型对象不同，这意味着无法子类化`TypeEngine`类以提供替代的`TypeEngine.literal_processor()`方法，除非显式地对`UserDefinedType`类进行子类化。

要为`TypeEngine.literal_processor()`提供替代行为，请实现一个`TypeDecorator`类，并提供`TypeDecorator.process_literal_param()`的实现。

另请参阅

增强现有类型

```py
attribute python_type
```

```py
class sqlalchemy.types.Enum
```

通用枚举类型。

`Enum`类型提供了一组可能的字符串值，该列受其约束。

如果可用，`Enum`类型将使用后端的本机“ENUM”类型；否则，它使用 VARCHAR 数据类型。当产生 VARCHAR（所谓的“非本机”）变体时，还存在一种自动生成 CHECK 约束的选项；请参阅`Enum.create_constraint`标志。

`Enum` 类型还提供了在 Python 中对字符串值进行读写操作时的验证。在结果集中从数据库中读取值时，始终会检查字符串值是否与可能值列表匹配，如果没有找到匹配项，则会引发 `LookupError`。在将值作为纯字符串传递给 SQL 语句中的数据库时，如果 `Enum.validate_strings` 参数设置为 True，则对于不在给定可能值列表中的任何字符串值都会引发 `LookupError`；请注意，这会影响使用枚举值的 LIKE 表达式的用法（一个不寻常的用例）。

枚举值的来源可以是字符串值列表，也可以是符合 PEP-435 的枚举类。对于 `Enum` 数据类型，此类只需要提供一个 `__members__` 方法。

当使用枚举类时，枚举对象用于输入和输出，而不是字符串，这与普通字符串枚举类型的情况不同：

```py
import enum
from sqlalchemy import Enum

class MyEnum(enum.Enum):
    one = 1
    two = 2
    three = 3

t = Table(
    'data', MetaData(),
    Column('value', Enum(MyEnum))
)

connection.execute(t.insert(), {"value": MyEnum.two})
assert connection.scalar(t.select()) is MyEnum.two
```

上面，每个元素的字符串名称，例如“one”、“two”、“three”，被持久化到数据库中；Python 枚举的值，这里表示为整数，**不会**被使用；因此，每个枚举的值可以是任何类型的 Python 对象，无论它是否可持久化。

为了持久化值而不是名称，可以使用 `Enum.values_callable` 参数。该参数的值是一个用户提供的可调用对象，用于与符合 PEP-435 的枚举类一起使用，并返回要持久化的字符串值列表。对于使用字符串值的简单枚举，诸如 `lambda x: [e.value for e in x]` 这样的可调用对象就足够了。

另请参阅

在类型映射中使用 Python 枚举或 pep-586 字面类型 - 关于在 ORM 的 ORM Annotated Declarative 特性中使用 `Enum` 数据类型的背景信息。

`ENUM` - PostgreSQL 特定类型，具有附加功能。

`ENUM` - MySQL 特定类型

**成员**

__init__(), create(), drop()

**类签名**

`sqlalchemy.types.Enum` 类（`sqlalchemy.types.String`、`sqlalchemy.types.SchemaType`、`sqlalchemy.types.Emulated`、`sqlalchemy.types.TypeEngine`）

```py
method __init__(*enums: object, **kw: Any)
```

构建一个枚举。

不适用于特定后端的关键字参数将被该后端忽略。

参数：

+   `*enums` – 要么是符合 PEP-435 的枚举类型，要么是一个或多个字符串标签。

+   `create_constraint` –

    默认为 False。当创建一个非本地枚举类型时，在数据库上还会构建一个 CHECK 约束，用于对有效值进行检查。

    注意

    强烈建议为 CHECK 约束指定一个显式名称，以支持模式管理方面的问题。这可以通过设置 `Enum.name` 参数或设置适当的命名约定来实现；请参阅 配置约束命名约定 以获取背景信息。

    在 1.4 版中更改：- 此标志现在默认为 False，意味着不会为非本地枚举类型生成 CHECK 约束。

+   `metadata` –

    将此类型直接与 `MetaData` 对象关联。对于在目标数据库上作为独立模式构造存在的类型（PostgreSQL），此类型将在 `create_all()` 和 `drop_all()` 操作中创建和删除。如果该类型未与任何 `MetaData` 对象关联，则它将与使用它的每个 `Table` 关联，并且将在创建任何这些单个表之一后创建，在检查其是否存在后。但是，仅在对该 `Table` 对象的元数据调用 `drop_all()` 时才会删除该类型。

    如果设置了 `MetaData.schema` 参数，则该参数的值将用作此对象上 `Enum.schema` 的默认值，如果没有显式指定值的话。

    在 1.4.12 版中更改：如果使用 `Enum.metadata` 参数传递，则 `Enum` 将继承 `MetaData` 对象的 `MetaData.schema` 参数（如果存在）。

+   `name` – 此类型的名称。对于需要显式命名的类型或显式命名约束以生成使用它的类型和/或表的未来支持的任何数据库，这是必需的。如果使用了 PEP-435 枚举类，则默认情况下使用其名称（转换为小写）。

+   `native_enum` – 在可用时使用数据库的原生 ENUM 类型。默认为 True。当为 False 时，对所有后端使用 VARCHAR + 检查约束。当为 False 时，可以使用`Enum.length`控制 VARCHAR 长度；当前如果 native_enum=True，则“length”被忽略。

+   `length` –

    允许在使用非本机枚举数据类型时为 VARCHAR 指定自定义长度。默认情况下，它使用最长值的长度。

    在 2.0.0 版中更改：`Enum.length`参数无条件地用于`VARCHAR`渲染，而不管`Enum.native_enum`参数，在其中`VARCHAR`用于枚举数据类型。

+   `schema` –

    此类型的模式名称。对于作为独立模式构造存在于目标数据库上的类型（PostgreSQL），此参数指定了类型存在的命名模式。

    如果不存在，则从`Enum.metadata`传递的`MetaData`集合中获取模式名称，用于包括`MetaData.schema`参数的`MetaData`。

    在 1.4.12 版中更改：当使用`Enum.metadata`参数传递时，`Enum`继承了`MetaData`对象的`MetaData.schema`参数。

    否则，如果`Enum.inherit_schema`标志设置为`True`，则架构将从关联的`Table`对象继承，如果有的话；当`Enum.inherit_schema`处于其默认值`False`时，不使用拥有表的架构。

+   `quote` – 为类型的名称设置显式引号首选项。

+   `inherit_schema` – 当为 `True` 时，将从拥有的 `Table` 复制`schema`到此 `Enum` 的`schema`属性，替换传递给 `schema` 属性的任何值。在使用 `Table.to_metadata()` 操作时也会生效。

+   `validate_strings` – 当为 True 时，将检查传递给 SQL 语句中的数据库的字符串值是否有效。未识别的值将导致引发 `LookupError`。

+   `values_callable` –

    一个可调用对象，将传递符合 PEP-435 规范的枚举类型，然后应返回要持久化的字符串值列表。这允许替代用法，例如将枚举的字符串值持久化到数据库中，而不是其名称。可调用对象必须以与遍历 Enum 的 `__member__` 属性相同的顺序返回要持久化的值。例如 `lambda x: [i.value for i in x]`。

    新版本 1.2.3 中新增。

+   `sort_key_function` –

    一个 Python 可调用对象，可用作 Python `sorted()` 内置函数中的“key”参数。SQLAlchemy ORM 要求映射的主键列必须以某种方式可排序。当使用不可排序的枚举对象，如 Python 3 的 `Enum` 对象时，可以使用此参数为对象设置默认排序键函数。默认情况下，枚举的数据库值用作排序函数。

    新版本 1.3.8 中新增。

+   `omit_aliases` –

    一个布尔值，当为 true 时将从 pep 435 枚举中删除别名。默认为 `True`。

    在版本 2.0 中更改：此参数现在默认为 True。

```py
method create(bind, checkfirst=False)
```

*继承自* `SchemaType.create()` *方法的* `SchemaType`

为此类型发出 CREATE DDL，如果适用。

```py
method drop(bind, checkfirst=False)
```

*继承自* `SchemaType.drop()` *方法的* `SchemaType`

为此类型发出 DROP DDL，如果适用。

```py
class sqlalchemy.types.Double
```

用于双 `FLOAT` 浮点类型的类型。

通常在 DDL 中生成 `DOUBLE` 或 `DOUBLE_PRECISION`，在 Python 端则像普通的 `Float` 一样运行。

新版本 2.0 中新增。

**类签名**

类 `sqlalchemy.types.Double` (`sqlalchemy.types.Float`)

```py
class sqlalchemy.types.Float
```

表示浮点类型的类型，如 `FLOAT` 或 `REAL`。

默认情况下，此类型返回 Python `float`对象，除非将`Float.asdecimal`标志设置为`True`，在这种情况下，它们将被强制转换为`decimal.Decimal`对象。

当在`Float`类型中未提供`Float.precision`时，某些后端可能将此类型编译为 8 字节/64 位浮点数据类型。要使用 4 字节/32 位浮点数据类型，通常可以提供精度<= 24，或者可以使用`REAL`类型。这在将类型呈现为`FLOAT`的 PostgreSQL 和 MSSQL 方言中是已知的，这两者都是`DOUBLE PRECISION`的别名。其他第三方方言可能具有类似的行为。

**成员**

__init__(), result_processor()

**类签名**

类`sqlalchemy.types.Float`（`sqlalchemy.types.Numeric`）

```py
method __init__(precision: int | None = None, asdecimal: bool = False, decimal_return_scale: int | None = None)
```

构造一个 Float。

参数：

+   `precision` –

    用于 DDL `CREATE TABLE`中的数字精度。后端**应该**尝试确保此精度指示通用`Float`数据类型的数字位数。

    注意

    对于 Oracle 后端，在渲染 DDL 时，不接受`Float.precision`参数，因为 Oracle 不支持将浮点精度指定为小数位数。相反，请使用特定于 Oracle 的`FLOAT`数据类型，并指定`FLOAT.binary_precision`参数。这是 SQLAlchemy 2.0 版本的新功能。

    要创建一个与数据库无关的`Float`，并为 Oracle 分别指定二进制精度，请使用`TypeEngine.with_variant()`如下：

    ```py
    from sqlalchemy import Column
    from sqlalchemy import Float
    from sqlalchemy.dialects import oracle

    Column(
        "float_data",
        Float(5).with_variant(oracle.FLOAT(binary_precision=16), "oracle")
    )
    ```

+   `asdecimal` – 与`Numeric`相同的标志，但默认为`False`。请注意，将此标志设置为`True`会导致浮点转换。

+   `decimal_return_scale` – 将浮点数转换为 Python 十进制数时使用的默认精度。由于浮点数的不精确性，浮点数值通常会更长，大多数浮点数数据库类型没有“精度”概念，因此默认情况下，float 类型在转换时会寻找前十个小数位。指定此值将覆盖该长度。请注意，如果未另行指定，具有“精度”的 MySQL 浮点数类型将使用“精度”作为 decimal_return_scale 的默认值。

```py
method result_processor(dialect, coltype)
```

返回一个用于处理结果行值的转换函数。

返回一个可调用函数，该函数将接收一个结果行列值作为唯一位置参数，并返回一个要返回给用户的值。

如果不需要处理，则该方法应返回`None`。

注意

仅相对于**特定方言类型对象**调用此方法，该对象通常是正在使用的方言的**私有类型**，并且不是公共类型对象，这意味着无法通过继承`TypeEngine`类来提供替代`TypeEngine.result_processor()`方法，除非明确地继承`UserDefinedType`类。

要为`TypeEngine.result_processor()`提供替代行为，请实现一个`TypeDecorator`类，并提供一个`TypeDecorator.process_result_value()`的实现。

另请参阅

扩展现有类型

参数：

+   `dialect` – 使用中的方言实例。

+   `coltype` – 在 cursor.description 中接收到的 DBAPI coltype 参数。

```py
class sqlalchemy.types.Integer
```

一种用于`int`整数的类型。

**成员**

get_dbapi_type(), literal_processor(), python_type

**类签名**

类 `sqlalchemy.types.Integer`（`sqlalchemy.types.HasExpressionLookup`，`sqlalchemy.types.TypeEngine`）

```py
method get_dbapi_type(dbapi)
```

返回底层 DB-API 的相应类型对象（如果有）。

例如，这对于调用 `setinputsizes()` 非常有用。

```py
method literal_processor(dialect)
```

返回一个用于处理要直接呈现而不使用绑定的文字值的转换函数。

当编译器使用“literal_binds”标志时，通常用于 DDL 生成以及某些后端不接受绑定参数的情况时，将使用此函数。

返回一个可调用对象，该对象将接收一个字面上的 Python 值作为唯一的位置参数，并返回一个字符串表示，用于在 SQL 语句中呈现。

注意

该方法仅相对于**特定方言类型对象**调用，该对象通常是**私有于正在使用的方言**，并且与公共面向的对象不是同一类型对象，这意味着无法通过子类化`TypeEngine`类来提供替代`TypeEngine.literal_processor()`方法，除非明确子类化`UserDefinedType`类。

要为`TypeEngine.literal_processor()`提供替代行为，请实现一个`TypeDecorator`类，并提供一个`TypeDecorator.process_literal_param()`的实现。

另请参阅

增强现有类型

```py
attribute python_type
```

```py
class sqlalchemy.types.Interval
```

用于`datetime.timedelta()`对象的类型。

间隔类型处理`datetime.timedelta`对象。在 PostgreSQL 和 Oracle 中，使用本地`INTERVAL`类型；对于其他数据库，该值存储为相对于“epoch”（1970 年 1 月 1 日）的日期。

请注意，`Interval`类型目前在不支持本地间隔类型的平台上不提供日期算术运算。此类操作通常需要转换表达式的两侧（例如，首先将两侧转换为整数 epoch 值），这目前是一个手动过程（例如，通过`expression.func`）。

**成员**

__init__(), adapt_to_emulated(), bind_processor(), cache_ok, coerce_compared_value(), comparator_factory, impl, python_type, result_processor()

**类签名**

类`sqlalchemy.types.Interval`（`sqlalchemy.types.Emulated`，`sqlalchemy.types._AbstractInterval`，`sqlalchemy.types.TypeDecorator`)

```py
class Comparator
```

**类签名**

类`sqlalchemy.types.Interval.Comparator`（`sqlalchemy.types.Comparator`，`sqlalchemy.types.Comparator`）

```py
method __init__(native: bool = True, second_precision: int | None = None, day_precision: int | None = None)
```

构造一个 Interval 对象。

参数：

+   `native` – 当为 True 时，如果支持（当前为 PostgreSQL、Oracle），则使用数据库提供的实际 INTERVAL 类型。否则，无论如何都将间隔数据表示为时代值。

+   `second_precision` – 用于支持“分秒精度”的本机间隔类型的参数，即 Oracle 和 PostgreSQL

+   `day_precision` – 对于支持“日精度”的本机间隔类型的参数，例如 Oracle。

```py
method adapt_to_emulated(impltype, **kw)
```

给定一个 impl 类，将此类型适配到 impl，假设“模拟”。

impl 也应是此类型的“模拟”版本，很可能是与此类型本身相同的类。

例如：sqltypes.Enum 适应于 Enum 类。

```py
method bind_processor(dialect: Dialect) → _BindProcessorType[dt.timedelta]
```

返回一个用于处理绑定值的转换函数。

返回一个可调用对象，它将接收一个绑定参数值作为唯一的位置参数，并返回要发送到 DB-API 的值。

如果不需要处理，则该方法应返回`None`。

注意

此方法仅相对于**特定方言的类型对象**调用，该类型对象通常是**正在使用的方言的私有对象**，并不是公共类型对象，因此无法通过对`TypeEngine`类进行子类化来提供替代的`TypeEngine.bind_processor()`方法，除非显式地对`UserDefinedType`类进行子类化。

要为`TypeEngine.bind_processor()`提供替代行为，请实现`TypeDecorator`类，并提供一个`TypeDecorator.process_bind_param()`的实现。

另请参阅

扩充现有类型

参数：

**dialect** – 正在使用的方言实例。

```py
attribute cache_ok: bool | None = True
```

指示使用此`ExternalType`的语句是否“可安全缓存”。

默认值`None`会发出警告，然后不允许缓存包含此类型的语句。将其设置为`False`可完全禁用使用此类型的语句进行缓存而不发出警告。当设置为`True`时，对象的类和其状态的选定元素将用作缓存键的一部分。例如，使用`TypeDecorator`:

```py
class MyType(TypeDecorator):
    impl = String

    cache_ok = True

    def __init__(self, choices):
        self.choices = tuple(choices)
        self.internal_only = True
```

上述类型的缓存键将等效于：

```py
>>> MyType(["a", "b", "c"])._static_cache_key
(<class '__main__.MyType'>, ('choices', ('a', 'b', 'c')))
```

缓存方案将从与`__init__()`方法中参数名称对应的类型中提取属性。在上面的例子中，“choices”属性成为缓存键的一部分，但“internal_only”不是，因为没有名为“internal_only”的参数。

可缓存元素的要求是它们是可哈希的，并且对于给定的缓存值，它们表示使用此类型的表达式的相同 SQL 每次都相同。

为了适应引用不可哈希结构的数据类型，例如字典、集合和列表，这些对象可以通过将可哈希结构分配给与参数名称对应的属性来“缓存”。例如，接受查找值字典的数据类型可以将其发布为排序后的元组系列。给定一个先前无法缓存的类型如下：

```py
class LookupType(UserDefinedType):
  '''a custom type that accepts a dictionary as a parameter.

 this is the non-cacheable version, as "self.lookup" is not
 hashable.

 '''

    def __init__(self, lookup):
        self.lookup = lookup

    def get_col_spec(self, **kw):
        return "VARCHAR(255)"

    def bind_processor(self, dialect):
        # ...  works with "self.lookup" ...
```

其中“lookup”是一个字典。该类型将无法生成缓存键：

```py
>>> type_ = LookupType({"a": 10, "b": 20})
>>> type_._static_cache_key
<stdin>:1: SAWarning: UserDefinedType LookupType({'a': 10, 'b': 20}) will not
produce a cache key because the ``cache_ok`` flag is not set to True.
Set this flag to True if this type object's state is safe to use
in a cache key, or False to disable this warning.
symbol('no_cache')
```

如果我们确实设置了这样的缓存键，它将无法使用。我们将获得一个包含其中的字典的元组结构，这个元组本身不能作为“缓存字典”中的键使用，例如 SQLAlchemy 的语句缓存，因为 Python 字典不可哈希：

```py
>>> # set cache_ok = True
>>> type_.cache_ok = True

>>> # this is the cache key it would generate
>>> key = type_._static_cache_key
>>> key
(<class '__main__.LookupType'>, ('lookup', {'a': 10, 'b': 20}))

>>> # however this key is not hashable, will fail when used with
>>> # SQLAlchemy statement cache
>>> some_cache = {key: "some sql value"}
Traceback (most recent call last): File "<stdin>", line 1,
in <module> TypeError: unhashable type: 'dict'
```

可以通过将排序后的元组分配给“.lookup”属性使该类型可缓存：

```py
class LookupType(UserDefinedType):
  '''a custom type that accepts a dictionary as a parameter.

 The dictionary is stored both as itself in a private variable,
 and published in a public variable as a sorted tuple of tuples,
 which is hashable and will also return the same value for any
 two equivalent dictionaries.  Note it assumes the keys and
 values of the dictionary are themselves hashable.

 '''

    cache_ok = True

    def __init__(self, lookup):
        self._lookup = lookup

        # assume keys/values of "lookup" are hashable; otherwise
        # they would also need to be converted in some way here
        self.lookup = tuple(
            (key, lookup[key]) for key in sorted(lookup)
        )

    def get_col_spec(self, **kw):
        return "VARCHAR(255)"

    def bind_processor(self, dialect):
        # ...  works with "self._lookup" ...
```

在上述情况中，`LookupType({"a": 10, "b": 20})`的缓存键将是：

```py
>>> LookupType({"a": 10, "b": 20})._static_cache_key
(<class '__main__.LookupType'>, ('lookup', (('a', 10), ('b', 20))))
```

1.4.14 版的新内容：- 添加了`cache_ok`标志，以允许对`TypeDecorator`类的缓存进行一些可配置性。

1.4.28 版的新内容：- 添加了`ExternalType`混合类型，它将`cache_ok`标志推广到`TypeDecorator`和`UserDefinedType`类。

另请参阅

SQL 编译缓存

```py
method coerce_compared_value(op, value)
```

在表达式中为“强制转换”Python 值建议一种类型。

给定一个运算符和值，让类型有机会返回一个应该将值强制转换为的类型。

此处的默认行为是保守的；如果右侧已经根据其 Python 类型强制转换为 SQL 类型，则通常会保持不变。

此处的最终用户功能扩展通常应通过 `TypeDecorator` 实现，它提供更自由的行为，因为它默认将表达式的另一侧强制转换为此类型，从而对两侧应用特殊的 Python 转换，除了 DBAPI 需要的转换之外。它还提供了公共方法 `TypeDecorator.coerce_compared_value()`，用于最终用户自定义此行为。

```py
attribute comparator_factory
```

`Comparator` 的别名

```py
attribute impl
```

`DateTime` 的别名

```py
attribute python_type
```

```py
method result_processor(dialect: Dialect, coltype: Any) → _ResultProcessorType[dt.timedelta]
```

返回用于处理结果行值的转换函数。

返回一个可调用对象，它将接收一个结果行列值作为唯一的位置参数，并将返回一个值以返回给用户。

如果不需要处理，该方法应返回 `None`。

注意

该方法仅相对于**特定方言类型对象**调用，这通常是**正在使用的方言的私有对象**，并且不是公共面向用户的对象，这意味着不可行地子类化 `TypeEngine` 类以提供替代 `TypeEngine.result_processor()` 方法，除非明确子类化 `UserDefinedType` 类。

要为 `TypeEngine.result_processor()` 提供替代行为，请实现一个 `TypeDecorator` 类并提供一个 `TypeDecorator.process_result_value()` 的实现。

另请参阅

增强现有类型

参数：

+   `dialect` – 正在使用的方言实例。

+   `coltype` – 在 cursor.description 中收到的 DBAPI coltype 参数。

```py
class sqlalchemy.types.LargeBinary
```

用于大型二进制字节数据的类型。

`LargeBinary` 类型对应于目标平台上的大型和/或无长度二进制类型，如 MySQL 上的 BLOB 和 PostgreSQL 上的 BYTEA。它还处理了 DBAPI 的必要转换。

**成员**

__init__()

**类签名**

类 `sqlalchemy.types.LargeBinary` (`sqlalchemy.types._Binary`)

```py
method __init__(length: int | None = None)
```

构造一个 LargeBinary 类型。

参数：

**length** – 可选，用于 DDL 语句中的列长度，用于那些接受长度的二进制类型，如 MySQL 的 BLOB 类型。

```py
class sqlalchemy.types.MatchType
```

指的是 MATCH 运算符的返回类型。

由于`ColumnOperators.match()`可能是通用 SQLAlchemy Core 中最开放的运算符，我们不能假设 SQL 评估时的返回类型，因为 MySQL 返回浮点数，而不是布尔值，其他后端可能会执行不同的操作。因此，此类型充当占位符，当前是`Boolean`的子类。该类型允许方言在需要时注入结果处理功能，并且在 MySQL 上将返回浮点值。

**类签名**

类`sqlalchemy.types.MatchType`（`sqlalchemy.types.Boolean`）

```py
class sqlalchemy.types.Numeric
```

非整数数值类型的基类，如`NUMERIC`、`FLOAT`、`DECIMAL`和其他变体。

直接使用`Numeric`数据类型时，如果可用，将呈现与精度数值对应的 DDL，例如`NUMERIC(precision, scale)`。`Float`子类将尝试呈现浮点数据类型，如`FLOAT(precision)`。

`Numeric`默认返回 Python `decimal.Decimal`对象，基于`Numeric.asdecimal`参数的默认值为`True`。如果将此参数设置为 False，则返回的值将被强制转换为 Python `float`对象。

`Float`子类型，更具体地针对浮点数，默认将`Float.asdecimal`标志设置为 False，以便默认 Python 数据类型为`float`。

注意

当使用`Numeric`数据类型与返回 Python 浮点值给驱动程序的数据库类型相对应时，由`Numeric.asdecimal`指示的十进制转换的准确性可能受到限制。特定数值/浮点数据类型的行为是使用的 SQL 数据类型、使用的 Python DBAPI 以及可能存在于使用的 SQLAlchemy 方言中的策略的产物。鼓励需要特定精度/比例的用户尝试使用可用的数据类型，以确定最佳结果。

**成员**

__init__(), bind_processor(), get_dbapi_type(), literal_processor(), python_type, result_processor()

**类签名**

类 `sqlalchemy.types.Numeric` (`sqlalchemy.types.HasExpressionLookup`, `sqlalchemy.types.TypeEngine`)

```py
method __init__(precision: int | None = None, scale: int | None = None, decimal_return_scale: int | None = None, asdecimal: bool = True)
```

构造一个 Numeric。

参数：

+   `precision` – 用于 DDL `CREATE TABLE` 中的数字精度。

+   `scale` – 用于 DDL `CREATE TABLE` 中的数字精度。

+   `asdecimal` – 默认为 True。返回值是否应该作为 Python 十进制对象发送，还是作为浮点数发送。不同的 DBAPI 基于数据类型发送其中之一 - Numeric 类型将确保返回值在不同的 DBAPI 中一致地是其中之一。

+   `decimal_return_scale` – 在从浮点数转换为 Python 十进制数时要使用的默认精度。由于十进制不精确性，浮点值通常会更长，而大多数浮点数据库类型都没有“精度”概念，因此默认情况下，浮点类型在转换时会查找前十位小数点。指定此值将覆盖该长度。包括显式“.scale”值的类型，如基础 `Numeric` 以及 MySQL 浮点类型，将使用“`.scale`”值作为 decimal_return_scale 的默认值，如果没有另外指定的话。

使用 `Numeric` 类型时，应注意确保 asdecimal 设置适用于正在使用的 DBAPI - 当 Numeric 应用从 Decimal->float 或 float-> Decimal 的转换时，此转换会为接收到的所有结果列增加额外的性能开销。

原生返回 Decimal 的 DBAPI（例如 psycopg2）在设置为 `True` 时将具有更好的精度和更高的性能，因为原生转换为 Decimal 可减少浮点问题的数量，并且 Numeric 类型本身不需要应用任何进一步的转换。然而，另一个原生返回浮点数的 DBAPI *将* 增加额外的转换开销，并且仍然受到浮点数据丢失的影响 - 在这种情况下，`asdecimal=False` 至少会消除额外的转换开销。

```py
method bind_processor(dialect)
```

返回一个用于处理绑定值的转换函数。

返回一个可调用对象，它将接收一个绑定参数值作为唯一的位置参数，并返回一个要发送到 DB-API 的值。

如果不需要处理，则该方法应返回 `None`。

注意

此方法仅相对于**特定方言类型对象**，该对象通常**是方言中私有的**，并且与公共对象不同，这意味着无法子类化`TypeEngine`类以提供替代的`TypeEngine.bind_processor()`方法，除非显式地子类化`UserDefinedType`类。

要为`TypeEngine.bind_processor()`提供替代行为，实现一个`TypeDecorator`类，并提供一个`TypeDecorator.process_bind_param()`的实现。

另请参阅

增强现有类型

参数:

**方言** - 使用中的方言实例。

```py
method get_dbapi_type(dbapi)
```

如果有的话，从底层 DB-API 返回相应的类型对象。

例如，这对于调用`setinputsizes()`可能很有用。

```py
method literal_processor(dialect)
```

返回一个转换函数，用于处理直接呈现而不使用绑定的文本值。

当编译器使用“literal_binds”标志时，将使用此函数，通常用于 DDL 生成以及在某些后端不接受绑定参数的情况下。

返回一个可调用对象，该对象将接收一个字面 Python 值作为唯一位置参数，并返回一个字符串表示以在 SQL 语句中呈现。

注意

此方法仅相对于**特定方言类型对象**，该对象通常**是方言中私有的**，并且与公共对象不同，这意味着无法子类化`TypeEngine`类以提供替代的`TypeEngine.literal_processor()`方法，除非显式地子类化`UserDefinedType`类。

若要为 `TypeEngine.literal_processor()` 提供替代行为，请实现一个 `TypeDecorator` 类，并提供一个 `TypeDecorator.process_literal_param()` 的实现。

另请参阅

扩充现有类型

```py
attribute python_type
```

```py
method result_processor(dialect, coltype)
```

返回一个转换函数，用于处理结果行值。

返回一个可调用对象，它将接收一个结果行列值作为唯一的位置参数，并返回一个值以返回给用户。

如果不需要处理，则该方法应返回 `None`。

注意

该方法仅针对 **特定方言类型对象** 调用，该对象通常 **私有于正在使用的方言**，并且不是与公共面向用户的类型对象相同的类型对象，这意味着不可能为了提供替代 `TypeEngine.result_processor()` 方法而对 `UserDefinedType` 类进行子类化，除非显式子类化 `UserDefinedType` 类。

若要为 `TypeEngine.result_processor()` 提供替代行为，请实现一个 `TypeDecorator` 类，并提供一个 `TypeDecorator.process_result_value()` 的实现。

另请参阅

扩充现有类型

参数:

+   `dialect` – 正在使用的 Dialect 实例。

+   `coltype` – 在`cursor.description`中接收的 DBAPI coltype 参数。

```py
class sqlalchemy.types.PickleType
```

持有使用 pickle 进行序列化的 Python 对象。

PickleType 在 Binary 类的基础上构建，将 Python 的 `pickle.dumps()` 应用于传入对象，并在传出时应用 `pickle.loads()`，从而允许将任何可 pickle 的 Python 对象存储为序列化的二进制字段。

若要允许与 `PickleType` 关联的元素的 ORM 更改事件传播，请参阅 Mutation Tracking。

**成员**

__init__(), bind_processor(), cache_ok, compare_values(), impl, result_processor()

**类签名**

类 `sqlalchemy.types.PickleType` (`sqlalchemy.types.TypeDecorator`)

```py
method __init__(protocol: int = 5, pickler: Any = None, comparator: Callable[[Any, Any], bool] | None = None, impl: _TypeEngineArgument[Any] | None = None)
```

构造一个 PickleType。

参数：

+   `protocol` – 默认为 `pickle.HIGHEST_PROTOCOL`。

+   `pickler` – 默认为 pickle。可以是具有 pickle 兼容的 `dumps` 和 `loads` 方法的任何对象。

+   `comparator` – 用于比较此类型值的 2-参数可调用谓词。如果保持为 `None`，则使用 Python 的“equals”运算符来比较值。

+   `impl` –

    用于替代默认的 `LargeBinary` 的二进制存储 `TypeEngine` 类或实例。例如，在使用 MySQL 时，可以更有效地使用 :class: _mysql.LONGBLOB 类。

    新版本 1.4.20 中新增。

```py
method bind_processor(dialect)
```

为给定的 `Dialect` 提供一个绑定值处理函数。

这是实现了 `TypeEngine` 合约的方法，用于绑定值转换，通常通过 `TypeEngine.bind_processor()` 方法实现。

注意

用户定义的 `TypeDecorator` 的子类**不应**实现此方法，而应该实现 `TypeDecorator.process_bind_param()` 方法，以保持实现类型提供的“内部”处理。

参数：

**dialect** – 正在使用的 Dialect 实例。

```py
attribute cache_ok: bool | None = True
```

使用此 `ExternalType` 表示的 if 语句是“可缓存的”。

默认值 `None` 将发出警告，然后不允许缓存包含此类型的语句。设置为 `False` 以完全禁用包含此类型的语句的缓存，而无需警告。当设置为 `True` 时，对象的类和其状态的选定元素将作为缓存键的一部分使用。例如，使用 `TypeDecorator`：

```py
class MyType(TypeDecorator):
    impl = String

    cache_ok = True

    def __init__(self, choices):
        self.choices = tuple(choices)
        self.internal_only = True
```

上述类型的缓存键将等价于：

```py
>>> MyType(["a", "b", "c"])._static_cache_key
(<class '__main__.MyType'>, ('choices', ('a', 'b', 'c')))
```

缓存方案将从与`__init__()`方法中的参数名称对应的类型中提取属性。上述，“choices”属性成为缓存键的一部分，但“internal_only”则不是，因为没有名为“internal_only”的参数。

可缓存元素的要求是它们是可散列的，并且它们指示对于给定缓存值，每次使用此类型的表达式渲染相同的 SQL。

为了适应引用诸如字典、集合和列表之类的不可散列结构的数据类型，这些对象可以通过将可散列结构赋值给与参数名称对应的属性来使其“可缓存”。例如，一个接受字典查找值的数据类型可以将其发布为一系列排序后的元组。给定先前不可缓存的类型如下：

```py
class LookupType(UserDefinedType):
  '''a custom type that accepts a dictionary as a parameter.

 this is the non-cacheable version, as "self.lookup" is not
 hashable.

 '''

    def __init__(self, lookup):
        self.lookup = lookup

    def get_col_spec(self, **kw):
        return "VARCHAR(255)"

    def bind_processor(self, dialect):
        # ...  works with "self.lookup" ...
```

当“lookup”是一个字典时，该类型将无法生成缓存键：

```py
>>> type_ = LookupType({"a": 10, "b": 20})
>>> type_._static_cache_key
<stdin>:1: SAWarning: UserDefinedType LookupType({'a': 10, 'b': 20}) will not
produce a cache key because the ``cache_ok`` flag is not set to True.
Set this flag to True if this type object's state is safe to use
in a cache key, or False to disable this warning.
symbol('no_cache')
```

如果我们**确实**设置了这样一个缓存键，它将无法使用。我们将得到一个包含字典的元组结构，其中字典本身无法作为“缓存字典”（如 SQLAlchemy 的语句缓存）中的键使用，因为 Python 字典不可散列：

```py
>>> # set cache_ok = True
>>> type_.cache_ok = True

>>> # this is the cache key it would generate
>>> key = type_._static_cache_key
>>> key
(<class '__main__.LookupType'>, ('lookup', {'a': 10, 'b': 20}))

>>> # however this key is not hashable, will fail when used with
>>> # SQLAlchemy statement cache
>>> some_cache = {key: "some sql value"}
Traceback (most recent call last): File "<stdin>", line 1,
in <module> TypeError: unhashable type: 'dict'
```

通过将一个排序后的元组的元组赋值给“.lookup”属性，可以使该类型可缓存：

```py
class LookupType(UserDefinedType):
  '''a custom type that accepts a dictionary as a parameter.

 The dictionary is stored both as itself in a private variable,
 and published in a public variable as a sorted tuple of tuples,
 which is hashable and will also return the same value for any
 two equivalent dictionaries.  Note it assumes the keys and
 values of the dictionary are themselves hashable.

 '''

    cache_ok = True

    def __init__(self, lookup):
        self._lookup = lookup

        # assume keys/values of "lookup" are hashable; otherwise
        # they would also need to be converted in some way here
        self.lookup = tuple(
            (key, lookup[key]) for key in sorted(lookup)
        )

    def get_col_spec(self, **kw):
        return "VARCHAR(255)"

    def bind_processor(self, dialect):
        # ...  works with "self._lookup" ...
```

在上述情况下，`LookupType({"a": 10, "b": 20})`的缓存键将为：

```py
>>> LookupType({"a": 10, "b": 20})._static_cache_key
(<class '__main__.LookupType'>, ('lookup', (('a', 10), ('b', 20))))
```

从版本 1.4.14 开始：- 添加了 `cache_ok` 标志，以允许对`TypeDecorator`类的缓存进行一些可配置化。

从版本 1.4.28 开始：- 添加了 `ExternalType` 混入，它将 `cache_ok` 标志推广到 `TypeDecorator` 和 `UserDefinedType` 类。

另请参阅

SQL 编译缓存

```py
method compare_values(x, y)
```

给定两个值，比较它们是否相等。

默认情况下，这将调用底层“impl”的`TypeEngine.compare_values()`，而这通常会使用 Python 的等号运算符`==`。

此函数由 ORM 用于比较原始加载的值与拦截的“更改”值，以确定是否发生了净变化。

```py
attribute impl
```

`LargeBinary` 的别名

```py
method result_processor(dialect, coltype)
```

为给定`Dialect`提供结果值处理函数。

这是满足绑定值转换的`TypeEngine`契约的方法，通常通过`TypeEngine.result_processor()`方法实现。

注意

用户定义的`TypeDecorator`的子类**不应**实现此方法，而应该实现`TypeDecorator.process_result_value()`，以便保持实现类型提供的“内部”处理。

参数：

+   `dialect` – 正在使用的方言实例。

+   `coltype` – 一个 SQLAlchemy 数据类型

```py
class sqlalchemy.types.SchemaType
```

为类型添加允许将模式级 DDL 与类型关联的功能。

支持必须显式创建/删除的类型（即 PG ENUM 类型），以及通过表或模式级约束、触发器和其他规则补充的类型。

`SchemaType` 类也可以成为`DDLEvents.before_parent_attach()`和`DDLEvents.after_parent_attach()`事件的目标，这些事件在类型对象与父`Column`关联时触发。

另请参阅

`Enum`

`Boolean`

**成员**

adapt(), copy(), create(), drop(), name

**类签名**

类`sqlalchemy.types.SchemaType`（`sqlalchemy.sql.expression.SchemaEventTarget`，`sqlalchemy.types.TypeEngineMixin`）

```py
method adapt(cls: Type[TypeEngine | TypeEngineMixin], **kw: Any) → TypeEngine
```

```py
method copy(**kw)
```

```py
method create(bind, checkfirst=False)
```

如适用，为此类型发出 CREATE DDL。

```py
method drop(bind, checkfirst=False)
```

如适用，为此类型发出 DROP DDL。

```py
attribute name: str | None
```

```py
class sqlalchemy.types.SmallInteger
```

用于较小的`int`整数的类型。

通常在 DDL 中生成`SMALLINT`，在 Python 端则像普通的`Integer`一样运行。

**类签名**

类`sqlalchemy.types.SmallInteger`（`sqlalchemy.types.Integer`）

```py
class sqlalchemy.types.String
```

所有字符串和字符类型的基础。

在 SQL 中对应于 VARCHAR。

当 String 类型在 CREATE TABLE 语句中使用时，通常需要长度字段，因为大多数数据库上的 VARCHAR 都需要长度。

**成员**

__init__(), bind_processor(), get_dbapi_type(), literal_processor(), python_type, result_processor()

**类签名**

类 `sqlalchemy.types.String` (`sqlalchemy.types.Concatenable`, `sqlalchemy.types.TypeEngine`)

```py
method __init__(length: int | None = None, collation: str | None = None)
```

创建一个持有字符串的类型。

参数:

+   `length` – 可选项，用于 DDL 和 CAST 表达式中列的长度。如果不会发出 `CREATE TABLE`，则可以安全地省略。某些数据库可能需要用于 DDL 的长度，并且在包含没有长度的 `VARCHAR` 的 `CREATE TABLE` DDL 时会引发异常。该值是以字节还是字符进行解释取决于数据库。

+   `collation` –

    可选项，用于 DDL 和 CAST 表达式的列级排序。使用 SQLite、MySQL 和 PostgreSQL 支持的 COLLATE 关键字进行渲染。例如：

    ```py
    >>> from sqlalchemy import cast, select, String
    >>> print(select(cast('some string', String(collation='utf8'))))
    SELECT  CAST(:param_1  AS  VARCHAR  COLLATE  utf8)  AS  anon_1 
    ```

    注意

    在大多数情况下，应该使用 `Unicode` 或 `UnicodeText` 数据类型来存储非 ASCII 数据的 `Column`。这些数据类型将确保在数据库上使用正确的类型。

```py
method bind_processor(dialect)
```

返回一个转换函数以处理绑定值。

返回一个可调用对象，该对象将接收一个绑定参数值作为唯一的位置参数，并返回要发送到 DB-API 的值。

如果不需要处理，则该方法应返回`None`。

注意

此方法仅相对于特定于方言的类型对象调用，该对象通常是正在使用的方言的私有类型对象，并且与公共类型对象不同，这意味着无法通过子类化 `TypeEngine` 类来提供替代 `TypeEngine.bind_processor()` 方法，除非显式子类化 `UserDefinedType` 类。

要为`TypeEngine.bind_processor()`提供替代行为，请实现一个`TypeDecorator`类，并提供一个`TypeDecorator.process_bind_param()`的实现。

另请参阅

扩展现有类型

参数：

**方言** – 正在使用的方言实例。

```py
method get_dbapi_type(dbapi)
```

返回底层 DB-API 中相应的类型对象（如果有）。

例如，这对调用`setinputsizes()`可能很有用。

```py
method literal_processor(dialect)
```

返回一个用于处理直接渲染而不使用绑定的字面值的转换函数。

当编译器使用“literal_binds”标志时使用此函数，通常用于 DDL 生成以及在某些后端不接受绑定参数的情况下。

返回一个可调用对象，该对象将接收一个字面 Python 值作为唯一的位置参数，并返回要在 SQL 语句中呈现的字符串表示。

注意

此方法仅相对于**特定方言类型对象**进行调用，该对象通常**私有于正在使用的方言**，并且不是公共类型对象，这意味着无法通过对`TypeEngine`类进行子类化来提供替代的`TypeEngine.literal_processor()`方法，除非显式地对`UserDefinedType`类进行子类化。

要为`TypeEngine.literal_processor()`提供替代行为，请实现一个`TypeDecorator`类，并提供一个`TypeDecorator.process_literal_param()`的实现。

另请参阅

扩展现有类型

```py
attribute python_type
```

```py
method result_processor(dialect, coltype)
```

返回一个用于处理结果行值的转换函数。

返回一个可调用对象，该对象将接收一个结果行列值作为唯一的位置参数，并返回一个值以返回给用户。

如果不需要处理，则该方法应返回`None`。

注意

该方法仅相对于**特定方言类型对象**调用，该对象通常**私有于使用的方言**，并且不是公共类型对象，这意味着不可通过子类化`TypeEngine`类来提供替代`TypeEngine.result_processor()`方法，除非显式子类化`UserDefinedType`类。

要为`TypeEngine.result_processor()`提供替代行为，实现一个`TypeDecorator`类，并提供一个`TypeDecorator.process_result_value()`的实现。

另请参见

增强现有类型

参数：

+   `dialect` – 使用中的方言实例。

+   `coltype` – 在 cursor.description 中接收的 DBAPI coltype 参数。

```py
class sqlalchemy.types.Text
```

可变大小的字符串类型。

在 SQL 中，通常对应于 CLOB 或 TEXT。一般来说，TEXT 对象没有长度；虽然一些数据库会接受这里的长度参数，但其他数据库会拒绝。

**类签名**

类`sqlalchemy.types.Text` (`sqlalchemy.types.String`)

```py
class sqlalchemy.types.Time
```

用于`datetime.time()`对象的类型。

**成员**

get_dbapi_type(), literal_processor(), python_type

**类签名**

类`sqlalchemy.types.Time` (`sqlalchemy.types._RenderISO8601NoT`, `sqlalchemy.types.HasExpressionLookup`, `sqlalchemy.types.TypeEngine`)

```py
method get_dbapi_type(dbapi)
```

如��有的话，从底层 DB-API 返回相应的类型对象。

这对于调用`setinputsizes()`可能很有用，例如。

```py
method literal_processor(dialect)
```

返回一个转换函数，用于处理直接呈现而不使用绑定的字面值。

当编译器使用“literal_binds”标志时，通常用于 DDL 生成以及某些后端不接受绑定参数的情况下使用此函数。

返回一个可调用对象，该对象将接收一个字面的 Python 值作为唯一的位置参数，并返回一个字符串表示以在 SQL 语句中呈现。

注意

此方法仅针对**特定方言类型对象**，通常**仅限于正在使用的方言私有**，并且与公共类型对象不同，这意味着无法通过子类化`TypeEngine`类来提供替代的`TypeEngine.literal_processor()`方法，除非显式地子类化`UserDefinedType`类。

要为`TypeEngine.literal_processor()`提供替代行为，实现一个`TypeDecorator`类，并提供`TypeDecorator.process_literal_param()`的实现。

另请参阅

增强现有类型

```py
attribute python_type
```

```py
class sqlalchemy.types.Unicode
```

可变长度 Unicode 字符串类型。

`Unicode`类型是一个`String`子类，假设输入和输出的字符串可能包含非 ASCII 字符，并且对于某些后端，暗示着明确支持非 ASCII 数据的底层列类型，比如在 Oracle 和 SQL Server 上的`NVARCHAR`。这将影响方言级别的`CREATE TABLE`语句和`CAST`函数的输出。

`Unicode`类型用于与数据库传输和接收数据的字符编码通常由 DBAPI 自身确定。所有现代 DBAPI 都支持非 ASCII 字符串，但可能有不同的管理数据库编码的方法；如果必要，应按照 Dialects 部分中目标 DBAPI 的说明进行配置。

在现代的 SQLAlchemy 中，使用`Unicode`数据类型不会暗示 SQLAlchemy 本身的任何编码/解码行为。在 Python 3 中，所有字符串对象都具有 Unicode 能力，SQLAlchemy 不会生成字节串对象，也不会适应 DBAPI 不返回 Python Unicode 对象作为字符串值结果集的情况。

警告

一些数据库后端，特别是使用 pyodbc 的 SQL Server，已知对被注明为`NVARCHAR`类型而不是`VARCHAR`类型的数据存在不良行为，包括数据类型不匹配错误和不使用索引。请参阅有关如何解决像 SQL Server 与 pyodbc 以及 cx_Oracle 这样的后端的 unicode 字符问题的背景信息的部分`DialectEvents.do_setinputsizes()`。

另请参阅

`UnicodeText` - 与`Unicode`的无长度文本对应。

`DialectEvents.do_setinputsizes()`

**类签名**

类`sqlalchemy.types.Unicode` (`sqlalchemy.types.String`)

```py
class sqlalchemy.types.UnicodeText
```

一个无界限长度的 Unicode 字符串类型。

有关此对象的 unicode 行为的详细信息，请参阅`Unicode`。

类似于`Unicode`，使用`UnicodeText`类型意味着在后端使用了一个支持 unicode 的类型，比如`NCLOB`，`NTEXT`。

**类签名**

类`sqlalchemy.types.UnicodeText` (`sqlalchemy.types.Text`)

```py
class sqlalchemy.types.Uuid
```

表示一个数据库通用的 UUID 数据类型。

对于没有“原生”UUID 数据类型的后端，该值将使用`CHAR(32)`并将 UUID 存储为 32 个字符的十六进制字符串。

对于已知直接支持`UUID`或类似的存储 uuid 的数据类型（例如 SQL Server 的`UNIQUEIDENTIFIER`）的后端，默认启用的“原生”模式将允许在这些后端上使用这些类型。

在其默认使用模式下，`Uuid`数据类型期望来自 Python [uuid](https://docs.python.org/3/library/uuid.html)模块的**Python uuid 对象**：

```py
import uuid

from sqlalchemy import Uuid
from sqlalchemy import Table, Column, MetaData, String

metadata_obj = MetaData()

t = Table(
    "t",
    metadata_obj,
    Column('uuid_data', Uuid, primary_key=True),
    Column("other_data", String)
)

with engine.begin() as conn:
    conn.execute(
        t.insert(),
        {"uuid_data": uuid.uuid4(), "other_data", "some data"}
    )
```

要使`Uuid`数据类型与基于字符串的 Uuids（例如 32 个字符的十六进制字符串）一起工作，请将`Uuid.as_uuid`参数传递值为`False`。

新版本 2.0 中添加。

另请参阅

`UUID` - 表示仅具有后端不可知行为的`UUID`数据类型。

**成员**

__init__(), bind_processor(), coerce_compared_value(), literal_processor(), python_type, result_processor()

**类签名**

类`sqlalchemy.types.Uuid` (`sqlalchemy.types.Emulated`, `sqlalchemy.types.TypeEngine`)

```py
method __init__(as_uuid: bool = True, native_uuid: bool = True)
```

构造一个`Uuid`类型。

参数：

+   `as_uuid=True` –

    如果为 True，则值将被解释为 Python uuid 对象，并通过 DBAPI 转换为/从字符串。

+   `native_uuid=True` – 如果为 True，则支持直接使用`UUID`数据类型或存储 UUID 值的后端（例如 SQL Server 的`UNIQUEIDENTIFIER`）将由这些后端使用。 如果为 False，则无论原生支持如何，所有后端都将使用`CHAR(32)`数据类型。

```py
method bind_processor(dialect)
```

返回一个用于处理绑定值的转换函数。

返回一个可调用对象，它将接收绑定参数值作为唯一的位置参数，并将返回要发送到 DB-API 的值。

如果不需要处理，则该方法应返回`None`。

注意

此方法仅相对于**特定方言的类型对象**调用，该对象通常是**正在使用的方言的私有对象**，并且不是与公共类型对象相同的类型对象，这意味着无法通过子类化`TypeEngine`类来提供备用`TypeEngine.bind_processor()`方法，除非明确地子类化`UserDefinedType`类。

要为`TypeEngine.bind_processor()`提供备用行为，实现一个`TypeDecorator`类，并提供一个`TypeDecorator.process_bind_param()`的实现。

另请参见

扩展现有类型

参数：

**方言** – 使用中的方言实例。

```py
method coerce_compared_value(op, value)
```

查看`TypeEngine.coerce_compared_value()`以获取描述信息。

```py
method literal_processor(dialect)
```

返回一个用于处理要直接呈现而不使用绑定的字面值的转换函数。

当编译器使用“literal_binds”标志时使用此函数，该标志通常用于 DDL 生成以及在某些情况下，后端不接受绑定参数的情况下使用。

返回一个可调用对象，该对象将接收一个字面上的 Python 值作为唯一的位置参数，并返回一个字符串表示，用于在 SQL 语句中呈现。

注意

此方法仅针对**特定方言的类型对象**调用，该对象通常**私有于正在使用的方言**，并且不是公共面向的类型对象，这意味着不可能通过子类化`TypeEngine`类来提供替代`TypeEngine.literal_processor()`方法，除非显式子类化`UserDefinedType`类。

要为`TypeEngine.literal_processor()`提供替代行为，实现一个`TypeDecorator`类，并提供一个`TypeDecorator.process_literal_param()`的实现。

另请参阅

扩展现有类型

```py
attribute python_type
```

```py
method result_processor(dialect, coltype)
```

返回用于处理结果行值的转换函数。

返回一个可调用对象，该对象将接收一个结果行列值作为唯一的位置参数，并返回一个要返回给用户的值。

如果不需要处理，该方法应返回`None`。

注意

此方法仅针对**特定方言的类型对象**调用，该对象通常**私有于正在使用的方言**，并且不是公共面向的类型对象，这意味着不可能通过子类化`TypeEngine`类来提供替代`TypeEngine.result_processor()`方法，除非显式子类化`UserDefinedType`类。

要为`TypeEngine.result_processor()`提供替代行为，实现一个`TypeDecorator`类，并提供`TypeDecorator.process_result_value()`的实现。

另请参阅

增强现有类型

参数：

+   `dialect` – 正在使用的方言实例。

+   `coltype` – 在 cursor.description 中收到的 DBAPI coltype 参数。

## SQL 标准和多供应商“大写”类型

此类类型指的是 SQL 标准的一部分，或者可能在一些数据库后端的子集中找到的类型。与“通用”类型不同，SQL 标准/多供应商类型**没有**保证在所有后端上工作，并且只会在明确通过名称支持它们的后端上工作。也就是说，当发出`CREATE TABLE`时，该类型将始终以其确切的名称在 DDL 中发出。

| 对象名称 | 描述 |
| --- | --- |
| ARRAY | 表示 SQL Array 类型。 |
| BIGINT | SQL BIGINT 类型。 |
| BINARY | SQL BINARY 类型。 |
| BLOB | SQL BLOB 类型。 |
| BOOLEAN | SQL BOOLEAN 类型。 |
| CHAR | SQL CHAR 类型。 |
| CLOB | CLOB 类型。 |
| DATE | SQL DATE 类型。 |
| DATETIME | SQL DATETIME 类型。 |
| DECIMAL | SQL DECIMAL 类型。 |
| DOUBLE | SQL DOUBLE 类型。 |
| DOUBLE_PRECISION | SQL DOUBLE PRECISION 类型。 |
| FLOAT | SQL FLOAT 类型。 |
| INT | `INTEGER` 的别名 |
| INTEGER | SQL INT 或 INTEGER 类型。 |
| JSON | 表示 SQL JSON 类型。 |
| NCHAR | SQL NCHAR 类型。 |
| NUMERIC | SQL NUMERIC 类型。 |
| NVARCHAR | SQL NVARCHAR 类型。 |
| REAL | SQL REAL 类型。 |
| SMALLINT | SQL SMALLINT 类型。 |
| TEXT | SQL TEXT 类型。 |
| TIME | SQL TIME 类型。 |
| TIMESTAMP | SQL TIMESTAMP 类型。 |
| UUID | 表示 SQL UUID 类型。 |
| VARBINARY | SQL VARBINARY 类型。 |
| VARCHAR | SQL VARCHAR 类型。 |

```py
class sqlalchemy.types.ARRAY
```

表示 SQL 数组类型。

注意

此类型为所有 ARRAY 操作提供了基础。然而，目前**只有 PostgreSQL 后端支持 SQL 数组在 SQLAlchemy 中**。建议在与 PostgreSQL 一起使用 ARRAY 类型时直接使用 PostgreSQL 特定的`sqlalchemy.dialects.postgresql.ARRAY`类型，因为它提供了特定于该后端的附加运算符。

`ARRAY`是核心的一部分，支持各种 SQL 标准函数，例如`array_agg`，明确涉及数组；但是，除了 PostgreSQL 后端和可能一些第三方方言外，没有其他 SQLAlchemy 内置方言支持此类型。

给定元素的“类型”，构造`ARRAY`类型：

```py
mytable = Table("mytable", metadata,
        Column("data", ARRAY(Integer))
    )
```

以上类型表示一个 N 维数组，意味着支持的后端（例如 PostgreSQL）将自动解释具有任意维数的值。要生成传递整数的一维数组的 INSERT 构造：

```py
connection.execute(
        mytable.insert(),
        {"data": [1,2,3]}
)
```

给定固定维数，可以构造`ARRAY`类型：

```py
mytable = Table("mytable", metadata,
        Column("data", ARRAY(Integer, dimensions=2))
    )
```

发送维度数量是可选的，但是如果数据类型要表示多于一个维度的数组，则建议使用。此数字用于：

+   在将类型声明本身发射到数据库时，例如`INTEGER[][]`

+   当将 Python 值翻译为数据库值，反之亦然时，例如，一个由`Unicode`对象组成的数组使用此数字来有效地访问数组结构内的字符串值，而不是使用每行类型检查。

+   当与 Python `getitem`访问器一起使用时，维数的数量用于定义`[]`操作符应返回的类型的种类，例如具有两个维度的 INTEGER 数组：

    ```py
    >>> expr = table.c.column[5]  # returns ARRAY(Integer, dimensions=1)
    >>> expr = expr[6]  # returns Integer
    ```

对于一维数组，通常假设没有维度参数的`ARRAY`实例将假定单维行为。

类型为`ARRAY`的 SQL 表达式支持“索引”和“切片”行为。`[]`操作符生成表达式构造，其将为 SELECT 语句生成适当的 SQL，例如：

```py
select(mytable.c.data[5], mytable.c.data[2:7])
```

以及当使用`Update.values()`方法时的 UPDATE 语句：

```py
mytable.update().values({
    mytable.c.data[5]: 7,
    mytable.c.data[2:7]: [1, 2, 3]
})
```

索引访问默认为基于一的；要进行从零开始的索引转换，请设置`ARRAY.zero_indexes`。

`ARRAY` 类型还提供了操作符 `Comparator.any()` 和 `Comparator.all()`。PostgreSQL 特定版本的 `ARRAY` 还提供了额外的操作符。

**在使用 ORM 时检测 ARRAY 列中的更改**

当与 SQLAlchemy ORM 一起使用时，`ARRAY` 类型不会检测对数组的原地突变。为了检测到这些变化，必须使用 `sqlalchemy.ext.mutable` 扩展，并使用 `MutableList` 类：

```py
from sqlalchemy import ARRAY
from sqlalchemy.ext.mutable import MutableList

class SomeOrmClass(Base):
    # ...

    data = Column(MutableList.as_mutable(ARRAY(Integer)))
```

此扩展将允许对数组进行“原地”更改，例如 `.append()` 会产生可以被工作单元检测到的事件。请注意，对数组内的元素的更改，包括原地突变的子数组，**不会**被检测到。

或者，将新的数组值分配给替换旧值的 ORM 元素将始终触发更改事件。

另请参阅

`sqlalchemy.dialects.postgresql.ARRAY`

**成员**

__init__(), contains(), any(), all()

**类签名**

类 `sqlalchemy.types.ARRAY` (`sqlalchemy.sql.expression.SchemaEventTarget`, `sqlalchemy.types.Indexable`, `sqlalchemy.types.Concatenable`, `sqlalchemy.types.TypeEngine`)

```py
method __init__(item_type: _TypeEngineArgument[Any], as_tuple: bool = False, dimensions: int | None = None, zero_indexes: bool = False)
```

构造一个 `ARRAY`。

例如：

```py
Column('myarray', ARRAY(Integer))
```

参数为：

参数：

+   `item_type` – 此数组项的数据类型。请注意，此处的维度是无关紧要的，因此像 `INTEGER[][]` 这样的多维数组，构造为 `ARRAY(Integer)`，而不是 `ARRAY(ARRAY(Integer))` 或类似的结构。

+   `as_tuple=False` – 指定返回结果是否应从列表转换为元组。通常不需要此参数，因为 Python 列表很好地对应于 SQL 数组。

+   `dimensions` – 如果非 None，则 ARRAY 将假定固定数量的维度。这会影响数组在数据库中的声明方式，以及它如何解释 Python 和结果值，以及与“getitem”运算符结合使用时表达式的行为。有关详细信息，请参阅 `ARRAY` 的描述。

+   `zero_indexes=False` – 当为 True 时，索引值将在 Python 以零为基础和 SQL 以一为基础的索引之间转换，例如，在传递给数据库之前，所有索引值都将增加一个值。

```py
class Comparator
```

为 `ARRAY` 定义比较操作。

此类型的特定于方言的形式上还提供了更多操作符。请参阅 `Comparator`。

**类签名**

class `sqlalchemy.types.ARRAY.Comparator` (`sqlalchemy.types.Comparator`, `sqlalchemy.types.Comparator`)

```py
method contains(*arg, **kw)
```

`ARRAY.contains()` 未为基本 ARRAY 类型实现。请使用特定于方言的 ARRAY 类型。

请参阅

`ARRAY` - PostgreSQL 特定版本。

```py
method any(other, operator=None)
```

返回 `other operator ANY (array)` 子句。

旧特性

此方法是一个 `ARRAY` - 特定的构造，现在已被 `any_()` 函数取代，后者具有不同的调用方式。 `any_()` 函数也通过 `ColumnOperators.any_()` 方法在方法级别进行了镜像。

使用数组特定的 `Comparator.any()` 如下所示：

```py
from sqlalchemy.sql import operators

conn.execute(
    select(table.c.data).where(
            table.c.data.any(7, operator=operators.lt)
        )
)
```

参数：

+   `other` – 待比较的表达式

+   `operator` – `sqlalchemy.sql.operators` 包中的操作符对象，默认为 `eq()`。

请参阅

`any_()`

`Comparator.all()`

```py
method all(other, operator=None)
```

返回 `other operator ALL (array)` 子句。

旧特性

该方法是一个特定的`ARRAY`构造，现在已被具有不同调用风格的`all_()`函数取代。`all_()`函数也通过`ColumnOperators.all_()`方法在方法级别进行了镜像。

使用特定于数组的`Comparator.all()`的方法如下：

```py
from sqlalchemy.sql import operators

conn.execute(
    select(table.c.data).where(
            table.c.data.all(7, operator=operators.lt)
        )
)
```

参数：

+   `other` – 要比较的表达式

+   `operator` – 来自`sqlalchemy.sql.operators`包的操作符对象，默认为`eq()`。

另请参阅

`all_()`

`Comparator.any()`

```py
class sqlalchemy.types.BIGINT
```

SQL BIGINT 类型。

另请参阅

`BigInteger` - 基本类型的文档。

**类签名**

类`sqlalchemy.types.BIGINT`（`sqlalchemy.types.BigInteger`）

```py
class sqlalchemy.types.BINARY
```

SQL BINARY 类型。

**类签名**

类`sqlalchemy.types.BINARY`（`sqlalchemy.types._Binary`）

```py
class sqlalchemy.types.BLOB
```

SQL BLOB 类型。

**成员**

__init__()

**类签名**

类`sqlalchemy.types.BLOB`（`sqlalchemy.types.LargeBinary`）

```py
method __init__(length: int | None = None)
```

*继承自* `LargeBinary` *的* `sqlalchemy.types.LargeBinary.__init__` *方法*

构造一个 LargeBinary 类型。

参数：

**length** – 可选，用于 DDL 语句中的列长度，用于那些接受长度的二进制类型，如 MySQL BLOB 类型。

```py
class sqlalchemy.types.BOOLEAN
```

SQL BOOLEAN 类型。

**成员**

__init__()

**类签名**

类`sqlalchemy.types.BOOLEAN`（`sqlalchemy.types.Boolean`）

```py
method __init__(create_constraint: bool = False, name: str | None = None, _create_events: bool = True, _adapted_from: SchemaType | None = None)
```

*继承自* `Boolean` *的* `sqlalchemy.types.Boolean.__init__` *方法*

构造一个 Boolean。

参数：

+   `create_constraint` –

    默认为 False。如果布尔值生成为 int/smallint，还要在表上创建一个 CHECK 约束，以确保值为 1 或 0。

    注意

    强烈建议 CHECK 约束具有明确的名称，以支持模式管理方面的考虑。可以通过设置`Boolean.name`参数或设置适当的命名约定来建立。有关背景，请参阅配置约束命名约定。

    从版本 1.4 开始更改： - 此标志现在默认为 False，表示为非本地枚举类型不生成 CHECK 约束。

+   `name` – 如果生成 CHECK 约束，请指定约束的名称。

```py
class sqlalchemy.types.CHAR
```

SQL CHAR 类型。

**成员**

__init__()

**类签名**

class `sqlalchemy.types.CHAR` (`sqlalchemy.types.String`)

```py
method __init__(length: int | None = None, collation: str | None = None)
```

*从* `String` *的* `sqlalchemy.types.String.__init__` *方法继承*

创建一个持有字符串的类型。

参数：

+   `length` – 可选项，用于 DDL 和 CAST 表达式中列的长度。如果不会发出`CREATE TABLE`，可以安全地省略。某些数据库可能需要 DDL 中的`length`，如果包括了没有长度的`VARCHAR`，则在发出`CREATE TABLE` DDL 时会引发异常。该值被解释为字节还是字符取决于数据库。

+   `collation` –

    可选项，用于 DDL 和 CAST 表达式中的列级排序。使用 SQLite、MySQL 和 PostgreSQL 支持的 COLLATE 关键字呈现。例如：

    ```py
    >>> from sqlalchemy import cast, select, String
    >>> print(select(cast('some string', String(collation='utf8'))))
    SELECT  CAST(:param_1  AS  VARCHAR  COLLATE  utf8)  AS  anon_1 
    ```

    注意

    在大多数情况下，应该使用`Unicode`或`UnicodeText`数据类型来存储非 ASCII 数据的`Column`。这些数据类型将确保在数据库上使用正确的类型。

```py
class sqlalchemy.types.CLOB
```

CLOB 类型。

此类型在 Oracle 和 Informix 中找到。

**成员**

__init__()

**类签名**

class `sqlalchemy.types.CLOB` (`sqlalchemy.types.Text`)

```py
method __init__(length: int | None = None, collation: str | None = None)
```

*从* `String` *的* `sqlalchemy.types.String.__init__` *方法继承*

创建一个持有字符串的类型。

参数：

+   `length` – 可选项，用于 DDL 和 CAST 表达式中列的长度。如果不会发出`CREATE TABLE`，可以安全地省略。某些数据库可能需要 DDL 中的`length`，如果包括了没有长度的`VARCHAR`，则在发出`CREATE TABLE` DDL 时会引发异常。该值被解释为字节还是字符取决于数据库。

+   `collation` –

    可选的，用于 DDL 和 CAST 表达式的列级校对。在 SQLite、MySQL 和 PostgreSQL 中使用 COLLATE 关键字呈现。例如：

    ```py
    >>> from sqlalchemy import cast, select, String
    >>> print(select(cast('some string', String(collation='utf8'))))
    SELECT  CAST(:param_1  AS  VARCHAR  COLLATE  utf8)  AS  anon_1 
    ```

    注意

    在大多数情况下，期望存储非 ASCII 数据的 `Column` 应使用 `Unicode` 或 `UnicodeText` 数据类型。这些数据类型将确保在数据库上使用正确的类型。

```py
class sqlalchemy.types.DATE
```

SQL DATE 类型。

**类签名**

类 `sqlalchemy.types.DATE`（`sqlalchemy.types.Date`的）

```py
class sqlalchemy.types.DATETIME
```

SQL DATETIME 类型。

**成员**

__init__()

**类签名**

类 `sqlalchemy.types.DATETIME`（`sqlalchemy.types.DateTime`的）

```py
method __init__(timezone: bool = False)
```

*继承自* `DateTime` *的* `sqlalchemy.types.DateTime.__init__` *方法*

构建一个新的 `DateTime`。

参数：

**timezone** – 布尔值。指示日期/时间类型应在仅在**基本日期/时间持有类型上可用**时启用时区支持。建议在使用此标志时直接使用 `TIMESTAMP` 数据类型，因为一些数据库包括与时区支持 TIMESTAMP 数据类型不同的单独的通用日期/时间持有类型，比如 Oracle。

```py
class sqlalchemy.types.DECIMAL
```

SQL DECIMAL 类型。

另请参阅

`Numeric` - 基本类型的文档。

**成员**

__init__()

**类签名**

类 `sqlalchemy.types.DECIMAL`（`sqlalchemy.types.Numeric`的）

```py
method __init__(precision: int | None = None, scale: int | None = None, decimal_return_scale: int | None = None, asdecimal: bool = True)
```

*继承自* `Numeric` *的* `sqlalchemy.types.Numeric.__init__` *方法*

构建一个 Numeric。

参数：

+   `precision` – 用于 DDL `CREATE TABLE` 的数值精度。

+   `scale` – 用于 DDL `CREATE TABLE` 的数值刻度。

+   `asdecimal` – 默认为 True。返回值是否应发送为 Python Decimal 对象，还是作为浮点数。不同的 DBAPI 基于数据类型发送其中之一 - Numeric 类型将确保跨 DBAPI 一致地返回值为其中之一。

+   `decimal_return_scale` – 默认精度，在将浮点数转换为 Python 十进制数时使用。由于浮点数的不准确性，浮点数值通常会更长，而大多数浮点数数据库类型没有“精度”的概念，因此默认情况下，浮点数类型在转换时会查找前十位小数。指定此值将覆盖该长度。包括显式“.scale”值的类型，例如基本的`Numeric`以及 MySQL 浮点类型，如果未另行指定，则将使用“.scale”的值作为 decimal_return_scale 的默认值。

当使用 `Numeric` 类型时，应注意确保 asdecimal 设置适用于正在使用的 DBAPI - 当 Numeric 应用从 Decimal->float 或 float-> Decimal 的转换时，此转换会为接收到的所有结果列增加额外的性能开销。

原生返回 Decimal 的 DBAPI（例如 psycopg2）将具有更好的准确性和更高的性能，设置为 `True`，因为原生翻译为 Decimal 可减少浮点数问题的数量，并且 Numeric 类型本身不需要应用任何进一步的转换。但是，另一个原生返回浮点数的 DBAPI *将*增加额外的转换开销，并且仍然会受到浮点数数据丢失的影响 - 在这种情况下，`asdecimal=False` 将至少消除额外的转换开销。

```py
class sqlalchemy.types.DOUBLE
```

SQL DOUBLE 类型。

在 2.0 版本中新增。

另请参阅

`Double` - 基本类型的文档。

**成员**

__init__()

**类签名**

类`sqlalchemy.types.DOUBLE` (`sqlalchemy.types.Double`)

```py
method __init__(precision: int | None = None, asdecimal: bool = False, decimal_return_scale: int | None = None)
```

*从* `Float` *的* `sqlalchemy.types.Float.__init__` *方法继承*

构造一个 Float。

参数：

+   `precision` –

    用于 DDL `CREATE TABLE` 中的数字精度。后端**应**尝试确保此精度指示通用`Float` 数据类型的数字位数。

    注意

    对于 Oracle 后端，在渲染 DDL 时，`Float.precision` 参数不被接受，因为 Oracle 不支持指定浮点精度为小数位数。相反，请使用 Oracle 特定的`FLOAT` 数据类型，并指定 `FLOAT.binary_precision` 参数。这是 SQLAlchemy 2.0 中的新功能。

    要创建一个数据库无关的`Float`，并为 Oracle 单独指定二进制精度，请使用 `TypeEngine.with_variant()` 如下所示：

    ```py
    from sqlalchemy import Column
    from sqlalchemy import Float
    from sqlalchemy.dialects import oracle

    Column(
        "float_data",
        Float(5).with_variant(oracle.FLOAT(binary_precision=16), "oracle")
    )
    ```

+   `asdecimal` – 与 `Numeric` 相同的标志，但默认值为 `False`。请注意，将此标志设置为 `True` 会导致浮点数转换。

+   `decimal_return_scale` – 在将浮点数转换为 Python 十进制数时使用的默认精度。由于十进制不准确性，浮点值通常会更长，并且大多数浮点数据库类型没有“精度”概念，因此默认情况下，浮点类型在转换时会查找前十位小数点。指定此值将覆盖该长度。请注意，如果未另行指定，包括“精度”的 MySQL 浮点类型将使用“精度”作为 decimal_return_scale 的默认值。

```py
class sqlalchemy.types.DOUBLE_PRECISION
```

SQL DOUBLE PRECISION 类型。

2.0 版本中的新功能。

另请参阅

`Double` - 基本类型的文档。

**成员**

__init__()

**类签名**

类 `sqlalchemy.types.DOUBLE_PRECISION`（`sqlalchemy.types.Double`）

```py
method __init__(precision: int | None = None, asdecimal: bool = False, decimal_return_scale: int | None = None)
```

*继承自* `Float` 的 `sqlalchemy.types.Float.__init__` *方法*

构造一个 Float。

参数：

+   `precision` –

    DDL `CREATE TABLE` 中用于使用的数字精度。后端**应**尝试确保此精度指示了用于通用 `Float` 数据类型的数字位数。

    注意

    对于 Oracle 后端，在渲染 DDL 时，不接受 `Float.precision` 参数，因为 Oracle 不支持以小数位数指定的浮点精度。而是使用特定于 Oracle 的 `FLOAT` 数据类型，并指定 `FLOAT.binary_precision` 参数。这是 SQLAlchemy 2.0 中的新功能。

    要创建一个数据库无关的`Float`，并为 Oracle 单独指定二进制精度，请使用 `TypeEngine.with_variant()` 如下所示：

    ```py
    from sqlalchemy import Column
    from sqlalchemy import Float
    from sqlalchemy.dialects import oracle

    Column(
        "float_data",
        Float(5).with_variant(oracle.FLOAT(binary_precision=16), "oracle")
    )
    ```

+   `asdecimal` – 与 `Numeric` 相同的标志，但默认值为 `False`。请注意，将此标志设置为 `True` 会导致浮点数转换。

+   `decimal_return_scale` – 在将浮点数转换为 Python 十进制数时使用的默认精度。由于十进制不准确性，浮点数值通常会更长，大多数浮点数据库类型没有“精度”概念，因此默认情况下，浮点类型在转换时会查找前十位小数点。指定此值将覆盖该长度。请注意，MySQL 浮点类型（包括“精度”）将使用“精度”作为 decimal_return_scale 的默认值，如果未另行指定。

```py
class sqlalchemy.types.FLOAT
```

SQL FLOAT 类型。

参见

`Float` - 基本类型的文档。

**成员**

__init__()

**类签名**

类 `sqlalchemy.types.FLOAT`（`sqlalchemy.types.Float`）

```py
method __init__(precision: int | None = None, asdecimal: bool = False, decimal_return_scale: int | None = None)
```

*继承自* `Float` *的* `sqlalchemy.types.Float.__init__` *方法*

构造一个 Float。

参数：

+   `precision` –

    用于 DDL `CREATE TABLE` 中的数字精度。后端 **应该** 尝试确保此精度指示通用 `Float` 数据类型的数字位数。

    注意

    对于 Oracle 后端，在渲染 DDL 时，不接受 `Float.precision` 参数，因为 Oracle 不支持将浮点精度指定为小数位数。相反，请使用特定于 Oracle 的 `FLOAT` 数据类型，并指定 `FLOAT.binary_precision` 参数。这是 SQLAlchemy 2.0 版本的新功能。

    要创建一个与数据库无关的 `Float`，并为 Oracle 单独指定二进制精度，请使用 `TypeEngine.with_variant()` 如下所示：

    ```py
    from sqlalchemy import Column
    from sqlalchemy import Float
    from sqlalchemy.dialects import oracle

    Column(
        "float_data",
        Float(5).with_variant(oracle.FLOAT(binary_precision=16), "oracle")
    )
    ```

+   `asdecimal` – 与 `Numeric` 相同的标志，但默认值为 `False`。请注意，将此标志设置为 `True` 会导致浮点数转换。

+   `decimal_return_scale` – 在将浮点数转换为 Python 十进制数时使用的默认精度。由于十进制不准确性，浮点数值通常会更长，大多数浮点数据库类型没有“精度”概念，因此默认情况下，浮点类型在转换时会查找前十位小数点。指定此值将覆盖该长度。请注意，MySQL 浮点类型（包括“精度”）将使用“精度”作为 decimal_return_scale 的默认值，如果未另行指定。

```py
attribute sqlalchemy.types.INT
```

`INTEGER` 的别名

```py
class sqlalchemy.types.JSON
```

表示 SQL JSON 类型。

注意

`JSON` 作为厂商特定 JSON 类型的门面提供。由于它支持 JSON SQL 操作，因此它仅适用于具有实际 JSON 类型的后端，目前包括：

+   PostgreSQL - 有关特定后端说明，请参阅 `sqlalchemy.dialects.postgresql.JSON` 和 `sqlalchemy.dialects.postgresql.JSONB`

+   MySQL - 有关特定后端说明，请参阅 `sqlalchemy.dialects.mysql.JSON`

+   自 SQLite 3.9 版本起 - 有关特定后端说明，请参阅 `sqlalchemy.dialects.sqlite.JSON`

+   Microsoft SQL Server 2016 及更高版本 - 有关特定后端说明，请参阅 `sqlalchemy.dialects.mssql.JSON`

`JSON` 是核心的一部分，支持本机 JSON 数据类型的日益流行。

`JSON` 类型可存储任意的 JSON 格式数据，例如：

```py
data_table = Table('data_table', metadata,
    Column('id', Integer, primary_key=True),
    Column('data', JSON)
)

with engine.connect() as conn:
    conn.execute(
        data_table.insert(),
        {"data": {"key1": "value1", "key2": "value2"}}
    )
```

**特定于 JSON 的表达式运算符**

`JSON` 数据类型提供以下附加 SQL 操作：

+   键索引操作：

    ```py
    data_table.c.data['some key']
    ```

+   整数索引操作：

    ```py
    data_table.c.data[3]
    ```

+   路径索引操作：

    ```py
    data_table.c.data[('key_1', 'key_2', 5, ..., 'key_n')]
    ```

+   针对特定 JSON 元素类型的数据转换器，在索引或路径操作被调用后：

    ```py
    data_table.c.data["some key"].as_integer()
    ```

    自版本 1.3.11 新增。

针对特定数据库方言的版本可能提供额外的操作，例如 `JSON` 的特定于数据库方言的版本，如 `sqlalchemy.dialects.postgresql.JSON` 和 `sqlalchemy.dialects.postgresql.JSONB` 都提供额外的 PostgreSQL 特定操作。

**将 JSON 元素转换为其他类型**

索引操作，即通过 Python 方括号操作符调用表达式的操作，例如 `some_column['some key']`，默认返回一个类型为 `JSON` 的表达式对象，以便对结果类型调用进一步的 JSON 导向指令。然而，更常见的情况可能是期望索引操作返回特定的标量元素，例如字符串或整数。为了以后端不可知的方式提供对这些元素的访问，提供了一系列的数据转换器：

+   `Comparator.as_string()` - 将元素作为字符串返回

+   `Comparator.as_boolean()` - 将元素作为布尔值返回

+   `Comparator.as_float()` - 将元素作为浮点数返回

+   `Comparator.as_integer()` - 将元素作为整数返回

这些数据转换器通过支持的方言实现，以确保对上述类型的比较能够正常工作，例如：

```py
# integer comparison
data_table.c.data["some_integer_key"].as_integer() == 5

# boolean comparison
data_table.c.data["some_boolean"].as_boolean() == True
```

1.3.11 版本新增：为基本 JSON 数据元素添加了类型特定的转换器。

注意

数据转换器函数是在 1.3.11 版本中新增的，取代了以前文档化的使用 CAST 的方法；作为参考，以前的做法如下所示：

```py
from sqlalchemy import cast, type_coerce
from sqlalchemy import String, JSON
cast(
    data_table.c.data['some_key'], String
) == type_coerce(55, JSON)
```

上述情况现在直接起作用：

```py
data_table.c.data['some_key'].as_integer() == 5
```

有关 1.3.x 系列中先前比较方法的详细信息，请参阅 SQLAlchemy 1.2 的文档或版本发行版的 doc/ 目录中包含的 HTML 文件。

**在使用 ORM 时检测 JSON 列中的更改**

当与 SQLAlchemy ORM 一起使用时，`JSON` 类型不会检测结构的原地变化。为了检测这些变化，必须使用 `sqlalchemy.ext.mutable` 扩展，最常见的是使用 `MutableDict` 类。此扩展将允许对数据结构的“原地”更改产生事件，这些事件将被工作单元检测到。有关涉及字典的简单示例的示例，请参阅 `HSTORE`。

或者，将 JSON 结构分配给替换旧结构的 ORM 元素将始终触发更改事件。

**支持 JSON null 与 SQL NULL**

处理 NULL 值时，`JSON` 类型建议使用两个特定的常量来区分一个评估为 SQL NULL 的列（例如，没有值），与 JSON 编码的字符串 `"null"`。要插入或选择一个 SQL NULL 值，请使用常量 `null()`。当使用包含特殊逻辑的 `JSON` 数据类型时，可以将此符号作为参数值传递，解释为列值应为 SQL NULL 而不是 JSON 的 `"null"`：

```py
from sqlalchemy import null
conn.execute(table.insert(), {"json_value": null()})
```

要插入或选择 JSON `"null"` 值，请使用常量 `JSON.NULL`：

```py
conn.execute(table.insert(), {"json_value": JSON.NULL})
```

`JSON`类型支持一个标志`JSON.none_as_null`，当设置为 True 时，Python 常量`None`将评估为 SQL NULL 的值，当设置为 False 时，Python 常量`None`将评估为 JSON 中的值`"null"`。在这些情况下，Python 值`None`可以与`JSON.NULL`和`null()`结合使用以指示 NULL 值，但必须注意在这些情况下`JSON.none_as_null`的值。

**自定义 JSON 序列化器**

`JSON`默认使用 Python 的`json.dumps`和`json.loads`函数作为 JSON 序列化器和反序列化器；在 psycopg2 方言的情况下，psycopg2 可能正在使用其自定义的加载器函数。

为了影响序列化器/反序列化器，它们目前可以在`create_engine()`级别通过`create_engine.json_serializer`和`create_engine.json_deserializer`参数进行配置。例如，要关闭`ensure_ascii`：

```py
engine = create_engine(
    "sqlite://",
    json_serializer=lambda obj: json.dumps(obj, ensure_ascii=False))
```

从版本 1.3.7 开始更改：SQLite 方言的`json_serializer`和`json_deserializer`参数从`_json_serializer`和`_json_deserializer`重命名。

另请参阅

`sqlalchemy.dialects.postgresql.JSON`

`sqlalchemy.dialects.postgresql.JSONB`

`sqlalchemy.dialects.mysql.JSON`

`sqlalchemy.dialects.sqlite.JSON`

**成员**

as_boolean(), as_float(), as_integer(), as_json(), as_numeric(), as_string(), bind_processor(), literal_processor(), NULL, __init__(), bind_processor(), comparator_factory, hashable, python_type, result_processor(), should_evaluate_none 

**类签名**

类 `sqlalchemy.types.JSON` (`sqlalchemy.types.Indexable`, `sqlalchemy.types.TypeEngine`) 

```py
class Comparator
```

为 `JSON` 定义比较操作。

**类签名**

类 `sqlalchemy.types.JSON.Comparator` (`sqlalchemy.types.Comparator`, `sqlalchemy.types.Comparator`) 

```py
method as_boolean()
```

将索引值转换为布尔值。

例如：

```py
stmt = select(
    mytable.c.json_column['some_data'].as_boolean()
).where(
    mytable.c.json_column['some_data'].as_boolean() == True
)
```

版本 1.3.11 中的新功能。

```py
method as_float()
```

将索引值转换为浮点数。

例如：

```py
stmt = select(
    mytable.c.json_column['some_data'].as_float()
).where(
    mytable.c.json_column['some_data'].as_float() == 29.75
)
```

版本 1.3.11 中的新功能。

```py
method as_integer()
```

将索引值转换为整数。

例如：

```py
stmt = select(
    mytable.c.json_column['some_data'].as_integer()
).where(
    mytable.c.json_column['some_data'].as_integer() == 5
)
```

版本 1.3.11 中的新功能。

```py
method as_json()
```

将索引值转换为 JSON。

例如：

```py
stmt = select(mytable.c.json_column['some_data'].as_json())
```

这通常是任何情况下索引元素的默认行为。

请注意，并非所有后端都支持完整 JSON 结构的比较。

版本 1.3.11 中的新功能。

```py
method as_numeric(precision, scale, asdecimal=True)
```

将索引值转换为数字/十进制数。

例如：

```py
stmt = select(
    mytable.c.json_column['some_data'].as_numeric(10, 6)
).where(
    mytable.c.
    json_column['some_data'].as_numeric(10, 6) == 29.75
)
```

版本 1.4.0b2 中的新功能。

```py
method as_string()
```

将索引值转换为字符串。

例如：

```py
stmt = select(
    mytable.c.json_column['some_data'].as_string()
).where(
    mytable.c.json_column['some_data'].as_string() ==
    'some string'
)
```

版本 1.3.11 中的新功能。

```py
class JSONElementType
```

JSON 表达式中索引 / 路径元素的常见函数。

**类签名**

类 `sqlalchemy.types.JSON.JSONElementType` (`sqlalchemy.types.TypeEngine`) 

```py
method bind_processor(dialect)
```

返回一个用于处理绑定值的转换函数。

返回一个可调用函数，该函数将接收绑定参数值作为唯一的位置参数，并返回要发送到 DB-API 的值。

如果不需要处理，则该方法应返回`None`。

注意

此方法仅与**方言特定类型对象**相关，该对象通常**私有于正在使用的方言**，并且与公共类型对象不同，这意味着无法通过子类化`TypeEngine`类来提供替代的`TypeEngine.bind_processor()`方法，除非显式地子类化`UserDefinedType`类。

要为`TypeEngine.bind_processor()`提供替代行为，实现一个`TypeDecorator`类并提供`TypeDecorator.process_bind_param()`的实现。

参见

扩展现有类型

参数：

**dialect** – 正在使用的方言实例。

```py
method literal_processor(dialect)
```

返回一个转换函数，用于处理直接呈现而不使用绑定的字面值。

当编译器使用“literal_binds”标志时，通常在 DDL 生成以及某些后端不接受绑定参数的情况下使用此函数。

返回一个可调用对象，该对象将接收一个字面 Python 值作为唯一的位置参数，并返回一个要在 SQL 语句中呈现的字符串表示。

注意

此方法仅与**方言特定类型对象**相关，该对象通常**私有于正在使用的方言**，并且与公共类型对象不同，这意味着无法通过子类化`TypeEngine`类来提供替代的`TypeEngine.literal_processor()`方法，除非显式地子类化`UserDefinedType`类。

要为`TypeEngine.literal_processor()`提供替代行为，实现一个`TypeDecorator`类并提供`TypeDecorator.process_literal_param()`的实现。

参见

扩展现有类型

```py
class JSONIndexType
```

JSON 索引值的数据类型占位符。

这允许对特殊语法的 JSON 索引值进行执行时处理。

**类签名**

类`sqlalchemy.types.JSON.JSONIndexType` (`sqlalchemy.types.JSONElementType`)

```py
class JSONIntIndexType
```

JSON 索引值的数据类型占位符。

这允许对特殊语法的 JSON 索引值进行执行时处理。

**类签名**

类`sqlalchemy.types.JSON.JSONIntIndexType` (`sqlalchemy.types.JSONIndexType`)

```py
class JSONPathType
```

JSON 路径操作的占位符类型。

这允许将基于路径的索引值在特定 SQL 语法中进行执行时处理。

**类签名**

类`sqlalchemy.types.JSON.JSONPathType` (`sqlalchemy.types.JSONElementType`)

```py
class JSONStrIndexType
```

JSON 索引值的数据类型占位符。

这允许对特殊语法的 JSON 索引值进行执行时处理。

**类签名**

类`sqlalchemy.types.JSON.JSONStrIndexType` (`sqlalchemy.types.JSONIndexType`)

```py
attribute NULL = symbol('JSON_NULL')
```

描述 NULL 的 JSON 值。

此值用于强制使用 JSON 值`"null"`作为值。Python 的`None`值将根据`JSON.none_as_null`标志的设置被识别为 SQL NULL 或 JSON`"null"`，常量`JSON.NULL`可用于始终解析为 JSON`"null"`，而不考虑此设置。这与始终解析为 SQL NULL 的`null()`构造形成对比。例如：

```py
from sqlalchemy import null
from sqlalchemy.dialects.postgresql import JSON

# will *always* insert SQL NULL
obj1 = MyObject(json_value=null())

# will *always* insert JSON string "null"
obj2 = MyObject(json_value=JSON.NULL)

session.add_all([obj1, obj2])
session.commit()
```

为了将 JSON NULL 设置为列的默认值，最透明的方法是使用`text()`：

```py
Table(
    'my_table', metadata,
    Column('json_data', JSON, default=text("'null'"))
)
```

虽然在这种情况下可以使用`JSON.NULL`，但`JSON.NULL`的值将作为列的值返回，这在 ORM 或其他默认值重新用途的情况下可能不理想。使用 SQL 表达式意味着该值将在检索生成的默认值的上下文中重新从数据库中获取。

```py
method __init__(none_as_null: bool = False)
```

构造一个`JSON`类型。

参数：

**none_as_null=False** –

如果为 True，则将值`None`持久化为 SQL 的 NULL 值，而不是`null`的 JSON 编码。请注意，当此标志为 False 时，`null()` 构造仍然可以用于持久化 NULL 值，可以直接作为参数值传递，该参数值会被`JSON` 类型解释为 SQL NULL：

```py
from sqlalchemy import null
conn.execute(table.insert(), {"data": null()})
```

注意

`JSON.none_as_null` 不适用于传递给`Column.default` 和 `Column.server_default`的值；对于这些参数传递的值为`None`意味着“没有默认值”。

此外，在 SQL 比较表达式中使用时，Python 值`None`仍然指代 SQL null，而不是 JSON NULL。`JSON.none_as_null` 标志明确指示了在 INSERT 或 UPDATE 语句中持久化该值时的情况。应使用`JSON.NULL`值来表示希望与 JSON null 进行比较的 SQL 表达式。

另请参阅

`JSON.NULL`

```py
method bind_processor(dialect)
```

返回一个用于处理绑定值的转换函数。

返回一个可调用对象，该对象将接收一个绑定参数值作为唯一的位置参数，并返回要发送到 DB-API 的值。

如果不需要处理，则该方法应返回`None`。

注意

该方法仅相对于**特定方言类型对象**调用，该对象通常**私有于正在使用的方言**，并且不是与公共面向的对象相同的类型对象，这意味着不可能通过子类化`TypeEngine` 类来提供备用的`TypeEngine.bind_processor()` 方法，除非明确子类化`UserDefinedType` 类。

要为`TypeEngine.bind_processor()`提供替代行为，实现一个`TypeDecorator`类，并提供一个实现`TypeDecorator.process_bind_param()`的实现。

另请参阅

增强现有类型

参数：

**dialect** – 正在使用的方言实例。

```py
attribute comparator_factory
```

`Comparator`的别名

```py
attribute hashable = False
```

如果为 False，则意味着此类型的值不可哈希。

当 ORM 对结果列表进行唯一化时使用。

```py
attribute python_type
```

```py
method result_processor(dialect, coltype)
```

返回一个用于处理结果行值的转换函数。

返回一个可调用对象，它将接收一个结果行列值作为唯一的位置参数，并返回一个要返回给用户的值。

如果不需要处理，则该方法应返回`None`。

注意

此方法仅相对于**特定方言类型对象**调用，该对象通常是正在使用的方言中的**私有对象**，并且不是公共类型对象的相同类型对象，这意味着无法通过子类化`TypeEngine`类来提供替代的`TypeEngine.result_processor()`方法，除非明确地子类化`UserDefinedType`类。

要为`TypeEngine.result_processor()`提供替代行为，请实现一个`TypeDecorator`类，并提供一个`TypeDecorator.process_result_value()`的实现。

参见

增强现有类型

参数：

+   `dialect` – 使用的方言实例。

+   `coltype` – 在 cursor.description 中接收的 DBAPI coltype 参数。

```py
attribute should_evaluate_none: bool
```

如果为 True，则认为 Python 常量`None`由此类型明确处理。

ORM 使用此标志指示在 INSERT 语句中将正值的`None`传递给列，而不是省略 INSERT 语句中的列，这会触发列级默认值。它还允许具有 Python None 的特殊行为的类型（例如 JSON 类型）指示它们希望明确处理 None 值。

要在现有类型上设置此标志，请使用`TypeEngine.evaluates_none()`方法。

参见

`TypeEngine.evaluates_none()`

```py
class sqlalchemy.types.INTEGER
```

SQL INT 或 INTEGER 类型。

参见

`Integer` - 基础类型的文档。

**类签名**

类`sqlalchemy.types.INTEGER` (`sqlalchemy.types.Integer`)

```py
class sqlalchemy.types.NCHAR
```

SQL NCHAR 类型。

**成员**

__init__()

**类签名**

类 `sqlalchemy.types.NCHAR`（`sqlalchemy.types.Unicode`）

```py
method __init__(length: int | None = None, collation: str | None = None)
```

*继承自* `String` *的* `sqlalchemy.types.String.__init__` *方法*

创建一个持有字符串的类型。

参数：

+   `length` – 可选的，用于 DDL 和 CAST 表达式中的列的长度。如果不会发出`CREATE TABLE`，则可以安全地省略。某些数据库可能需要在 DDL 中使用长度，并且如果包含没有长度的`VARCHAR`，则在发出`CREATE TABLE` DDL 时会引发异常。值是以字节还是字符解释是特定于数据库的。

+   `collation` –

    可选的，用于 DDL 和 CAST 表达式中的列级排序。使用 SQLite、MySQL 和 PostgreSQL 支持的 COLLATE 关键字呈现。例如：

    ```py
    >>> from sqlalchemy import cast, select, String
    >>> print(select(cast('some string', String(collation='utf8'))))
    SELECT  CAST(:param_1  AS  VARCHAR  COLLATE  utf8)  AS  anon_1 
    ```

    注意

    在大多数情况下，`Unicode` 或 `UnicodeText` 数据类型应该用于预期存储非 ASCII 数据的 `Column`。这些数据类型将确保在数据库上使用正确的类型。

```py
class sqlalchemy.types.NVARCHAR
```

SQL NVARCHAR 类型。

**成员**

__init__()

**类签名**

类 `sqlalchemy.types.NVARCHAR`（`sqlalchemy.types.Unicode`）

```py
method __init__(length: int | None = None, collation: str | None = None)
```

*继承自* `String` *的* `sqlalchemy.types.String.__init__` *方法*

创建一个持有字符串的类型。

参数：

+   `length` – 可选的，用于 DDL 和 CAST 表达式中的列的长度。如果不会发出`CREATE TABLE`，则可以安全地省略。某些数据库可能需要在 DDL 中使用长度，并且如果包含没有长度的`VARCHAR`，则在发出`CREATE TABLE` DDL 时会引发异常。值是以字节还是字符解释是特定于数据库的。

+   `collation` –

    可选的，用于 DDL 和 CAST 表达式中的列级排序。使用 SQLite、MySQL 和 PostgreSQL 支持的 COLLATE 关键字呈现。���如：

    ```py
    >>> from sqlalchemy import cast, select, String
    >>> print(select(cast('some string', String(collation='utf8'))))
    SELECT  CAST(:param_1  AS  VARCHAR  COLLATE  utf8)  AS  anon_1 
    ```

    注意

    在大多数情况下，`Unicode` 或 `UnicodeText` 数据类型应该用于预期存储非 ASCII 数据的 `Column`。这些数据类型将确保在数据库上使用正确的类型。

```py
class sqlalchemy.types.NUMERIC
```

SQL NUMERIC 类型。

另请参阅

`Numeric` - 基本类型的文档。

**成员**

__init__()

**类签名**

类`sqlalchemy.types.NUMERIC`（`sqlalchemy.types.Numeric`）。

```py
method __init__(precision: int | None = None, scale: int | None = None, decimal_return_scale: int | None = None, asdecimal: bool = True)
```

*继承自* `Numeric` *的* `sqlalchemy.types.Numeric.__init__` *方法。

构造一个数字。

参数：

+   `precision` – 用于 DDL `CREATE TABLE` 中的数字精度。

+   `scale` – 用于 DDL `CREATE TABLE` 中的数字比例。

+   `asdecimal` – 默认为 True。返回值是否应发送为 Python 十进制对象或浮点数。不同的 DBAPI 根据数据类型发送其中之一 - Numeric 类型将确保返回值在各个 DBAPI 中始终一致地是其中之一。

+   `decimal_return_scale` – 将浮点数转换为 Python 十进制数时要使用的默认精度。由于十进制的不准确性，浮点数值通常会更长，而大多数浮点数据库类型没有“精度”的概念，因此默认情况下，浮点类型在转换时会查找前十位小数。指定此值将覆盖该长度。具有显式“`.scale`”值的类型（例如基本`Numeric`以及 MySQL 浮点类型）将使用“`.scale`”值作为`decimal_return_scale`的默认值，如果未另行指定。

使用`Numeric`类型时，应注意确保`asdecimal`设置适用于正在使用的 DBAPI - 当 Numeric 应用 Decimal->float 或 float-> Decimal 的转换时，此转换会为接收到的所有结果列带来额外的性能开销。

本机返回 Decimal 的 DBAPI（例如 psycopg2）将具有更好的精度和更高的性能，设置为`True`，因为对 Decimal 的本机转换减少了涉及的浮点问题的数量，并且 Numeric 类型本身不需要应用任何进一步的转换。但是，另一个本机返回浮点数的 DBAPI *将*产生额外的转换开销，并且仍然受到浮点数据丢失的影响 - 在这种情况下，`asdecimal=False`至少会消除额外的转换开销。

```py
class sqlalchemy.types.REAL
```

SQL REAL 类型。

另请参阅

`Float` - 基本类型的文档。

**成员**

__init__()

**类签名**

类`sqlalchemy.types.REAL`（`sqlalchemy.types.Float`）。

```py
method __init__(precision: int | None = None, asdecimal: bool = False, decimal_return_scale: int | None = None)
```

*继承自* `Float` *的* `sqlalchemy.types.Float.__init__` *方法。

构造一个浮点数。

参数：

+   `precision` –

    用于 DDL `CREATE TABLE` 中的数字精度。后端**应**尝试确保此精度指示了用于通用`Float`数据类型的数字位数。

    注意

    对于 Oracle 后端，在渲染 DDL 时，不接受 `Float.precision` 参数，因为 Oracle 不支持将浮点精度指定为小数位数。相反，使用 Oracle 特定的 `FLOAT` 数据类型，并指定 `FLOAT.binary_precision` 参数。这是 SQLAlchemy 版本 2.0 中的新功能。

    要创建一个与数据库无关的 `Float`，可以分别为 Oracle 指定二进制精度，请使用 `TypeEngine.with_variant()` 如下所示：

    ```py
    from sqlalchemy import Column
    from sqlalchemy import Float
    from sqlalchemy.dialects import oracle

    Column(
        "float_data",
        Float(5).with_variant(oracle.FLOAT(binary_precision=16), "oracle")
    )
    ```

+   `asdecimal` – 与 `Numeric` 相同的标志，但默认为 `False`。请注意，将此标志设置为 `True` 将导致浮点转换。

+   `decimal_return_scale` – 在将浮点数转换为 Python 十进制数时要使用的默认比例。由于十进制的不精确性，浮点值通常会更长，而大多数浮点数据库类型都没有“比例”的概念，因此，默认情况下，浮点类型在转换时会寻找前十个小数位数。指定此值将覆盖该长度。请注意，MySQL 浮点类型包括“比例”，如果没有另外指定，则将使用“比例”作为 decimal_return_scale 的默认值。

```py
class sqlalchemy.types.SMALLINT
```

SQL SMALLINT 类型。

另请参阅

`SmallInteger` - 基本类型的文档。

**类签名**

类 `sqlalchemy.types.SMALLINT` (`sqlalchemy.types.SmallInteger`)

```py
class sqlalchemy.types.TEXT
```

SQL TEXT 类型。

**成员**

__init__()

**类签名**

类 `sqlalchemy.types.TEXT` (`sqlalchemy.types.Text`)

```py
method __init__(length: int | None = None, collation: str | None = None)
```

*继承自* `String` 的 `sqlalchemy.types.String.__init__` *方法*

创建一个字符串持有类型。

参数：

+   `length` – 可选，用于在 DDL 和 CAST 表达式中的列长度。如果不会发出 `CREATE TABLE`，则可以安全地省略。某些数据库可能需要在 DDL 中使用长度，并且如果包含了没有长度的 `VARCHAR`，则在发出 `CREATE TABLE` DDL 时将引发异常。值是按字节还是按字符解释是特定于数据库的。

+   `collation` –

    可选，用于在 DDL 和 CAST 表达式中的列级别排序。使用 SQLite、MySQL 和 PostgreSQL 支持的 COLLATE 关键字渲染。例如：

    ```py
    >>> from sqlalchemy import cast, select, String
    >>> print(select(cast('some string', String(collation='utf8'))))
    SELECT  CAST(:param_1  AS  VARCHAR  COLLATE  utf8)  AS  anon_1 
    ```

    注意

    在大多数情况下，`Unicode`或`UnicodeText`数据类型应该用于预期存储非 ASCII 数据的`Column`。这些数据类型将确保在数据库上使用正确的类型。

```py
class sqlalchemy.types.TIME
```

SQL 的时间类型。

**类签名**

class `sqlalchemy.types.TIME` (`sqlalchemy.types.Time`)

```py
class sqlalchemy.types.TIMESTAMP
```

SQL 的时间戳类型。

`TIMESTAMP`数据类型在一些后端（如 PostgreSQL 和 Oracle）上支持时区存储。使用`TIMESTAMP.timezone`参数以启用这些后端的“带时区的 TIMESTAMP”。

**成员**

__init__(), get_dbapi_type()

**类签名**

class `sqlalchemy.types.TIMESTAMP` (`sqlalchemy.types.DateTime`)

```py
method __init__(timezone: bool = False)
```

构造一个新的`TIMESTAMP`。

参数：

**时区** – 布尔值。表示如果目标数据库支持时区，则应该启用 TIMESTAMP 类型的时区支持。在每个方言基础上类似于“带时区的 TIMESTAMP”。如果目标数据库不支持时区，则忽略此标志。

```py
method get_dbapi_type(dbapi)
```

如果存在的话，从底层 DB-API 返回相应的类型对象。

这对于例如调用`setinputsizes()`是有用的。

```py
class sqlalchemy.types.UUID
```

表示 SQL UUID 类型。

这是`Uuid`数据库不可知数据类型的 SQL 本机形式，并且与以前的仅适用于 PostgreSQL 版本的 UUID 向后兼容。

`UUID`数据类型仅在具有名为 UUID 的 SQL 数据类型的数据库上工作。它不会对不具有此精确名称类型的后端（包括 SQL Server）产生影响。对于具有本地支持的后端不可知 UUID 值，包括对于 SQL Server 的`UNIQUEIDENTIFIER`数据类型，请使用`Uuid`数据类型。

2.0 版中的新功能。

另请参见

`Uuid`

**成员**

__init__()

**类签名**

class `sqlalchemy.types.UUID` (`sqlalchemy.types.Uuid`, `sqlalchemy.types.NativeForEmulated`)

```py
method __init__(as_uuid: bool = True)
```

构造一个`UUID`类型。

参数：

**as_uuid=True** –

如果为 True，则值将被解释为 Python uuid 对象，并通过 DBAPI 转换为/从字符串。

```py
class sqlalchemy.types.VARBINARY
```

SQL VARBINARY 类型。

**类签名**

类 `sqlalchemy.types.VARBINARY` (`sqlalchemy.types._Binary`)

```py
class sqlalchemy.types.VARCHAR
```

SQL VARCHAR 类型。

**成员**

__init__()

**类签名**

类 `sqlalchemy.types.VARCHAR` (`sqlalchemy.types.String`)

```py
method __init__(length: int | None = None, collation: str | None = None)
```

*继承自* `String` *的* `sqlalchemy.types.String.__init__` *方法*

创建一个字符串持有类型。

参数：

+   `length` – 可选，用于 DDL 和 CAST 表达式中的列长度。如果不会发出`CREATE TABLE`，则可以安全地省略。某些数据库可能需要 DDL 中的`length`，如果包含了没有长度的`VARCHAR`，则在发出`CREATE TABLE` DDL 时会引发异常。该值是以字节还是字符解释的，具体取决于数据库。

+   `collation` –

    可选，用于 DDL 和 CAST 表达式的列级排序规则。使用 SQLite、MySQL 和 PostgreSQL 支持的 COLLATE 关键字进行渲染。例如：

    ```py
    >>> from sqlalchemy import cast, select, String
    >>> print(select(cast('some string', String(collation='utf8'))))
    SELECT  CAST(:param_1  AS  VARCHAR  COLLATE  utf8)  AS  anon_1 
    ```

    注意

    在大多数情况下，`Unicode` 或 `UnicodeText` 数据类型应该被用于期望存储非 ASCII 数据的 `Column`。这些数据类型将确保在数据库上使用正确的类型。
