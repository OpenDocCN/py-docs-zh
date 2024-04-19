# 使用声明性进行表格配置

> 原文：[`docs.sqlalchemy.org/en/20/orm/declarative_tables.html`](https://docs.sqlalchemy.org/en/20/orm/declarative_tables.html)

如 声明性映射 中所介绍的，声明性样式包括生成一个映射的 `Table` 对象的能力，或者直接适应一个 `Table` 或其他 `FromClause` 对象。

以下示例假定有一个声明性基类如下：

```py
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    pass
```

接下来的所有示例都演示了从上述 `Base` 继承的类。在 使用装饰器的声明性映射（无声明性基类） 中介绍的装饰器样式以及通过 `declarative_base()` 生成的基类的遗留形式都得到了全面支持。

## 使用 `mapped_column()` 的声明性表格

在使用声明性时，大多数情况下要映射的类的主体包括一个名为 `__tablename__` 的属性，该属性指示应与映射一起生成的 `Table` 的字符串名称。然后在类主体中使用 `mapped_column()` 构造，该构造具有额外的 ORM 特定配置功能，在普通的 `Column` 类中不存在，以指示表中的列。下面的示例说明了在声明性映射中使用此构造的最基本用法：

```py
from sqlalchemy import Integer, String
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import mapped_column

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "user"

    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50), nullable=False)
    fullname = mapped_column(String)
    nickname = mapped_column(String(30))
```

在上面的示例中，`mapped_column()` 构造函数被放置在类定义中作为类级别属性。在声明类的时候，声明性映射过程将根据与声明性 `Base` 关联的 `MetaData` 集合生成一个新的 `Table` 对象；然后，每个 `mapped_column()` 的实例将在此过程中用于生成一个 `Column` 对象，该对象将成为此 `Table` 对象的 `Table.columns` 集合的一部分。

在上面的示例中，声明性将构建一个等同于以下内容的 `Table` 结构：

```py
# equivalent Table object produced
user_table = Table(
    "user",
    Base.metadata,
    Column("id", Integer, primary_key=True),
    Column("name", String(50)),
    Column("fullname", String()),
    Column("nickname", String(30)),
)
```

在上面的 `User` 类被映射之后，可以通过 `__table__` 属性直接访问这个 `Table` 对象；详细内容请参阅 访问表和元数据。

`mapped_column()` 构造函数接受所有被 `Column` 构造函数接受的参数，以及额外的 ORM 特定参数。通常会省略 `mapped_column.__name` 字段，指示数据库列的名称，因为声明性过程将使用给定构造函数的属性名称，并将其分配为列的名称（在上面的示例中，这指的是 `id`、`name`、`fullname`、`nickname` 的名称）。也可以分配一个替代的 `mapped_column.__name`，在这种情况下，生成的 `Column` 将在 SQL 和 DDL 语句中使用给定的名称，而 `User` 映射类将继续允许使用给定的属性名称访问属性，而不管列本身的名称如何（更多内容请参阅 明确命名声明式映射列）。

提示

`mapped_column()` 结构**仅在 Declarative 类映射内有效**。 在使用 Core 构造`Table`对象以及在使用 imperative table 配置时，仍然需要`Column`结构以指示数据库列的存在。

另请参阅

映射表列 - 包含有关影响`Mapper`解释传入的`Column`对象的附加说明。

### 使用带注释的声明表（`mapped_column()`的类型注释形式）

`mapped_column()` 结构能够从与 Declarative 映射类中声明的属性相关联的[**PEP 484**](https://peps.python.org/pep-0484/)类型注释中派生其列配置信息。 如果使用了这些类型注释，则**必须**存在于称为`Mapped`的特殊 SQLAlchemy 类型中，该类型然后表示其中的特定 Python 类型。

以下说明了从前一节开始的映射，增加了对`Mapped`的使用：

```py
from typing import Optional

from sqlalchemy import String
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "user"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(50))
    fullname: Mapped[Optional[str]]
    nickname: Mapped[Optional[str]] = mapped_column(String(30))
```

在上述情况下，当 Declarative 处理每个类属性时，如果存在，每个`mapped_column()`将从左侧相应的`Mapped`类型注释派生出其他参数。 此外，当遇到没有分配给属性的`Mapped`类型注释时（这种形式受到了 Python [dataclasses](https://docs.python.org/3/library/dataclasses.html)中使用的类似样式的启发）Declarative 将隐式生成一个空的`mapped_column()`指令；此`mapped_column()`结构继续从存在的`Mapped`注释派生其配置。

#### `mapped_column()`从`Mapped`注释中派生出数据类型和可空性。

`mapped_column()` 从 `Mapped` 注解中派生的两个特性是：

+   **datatype** - 在 `Mapped` 中给定的 Python 类型，如果存在于 `typing.Optional` 结构中，则与 `TypeEngine` 的子类相关联，例如 `Integer`、`String`、`DateTime` 或 `Uuid` 等常见类型。

    数据类型是基于 Python 类型到 SQLAlchemy 数据类型的字典确定的。这个字典是完全可定制的，如下一节 自定义类型映射 中详细说明。默认类型映射的实现如下面的代码示例所示：

    ```py
    from typing import Any
    from typing import Dict
    from typing import Type

    import datetime
    import decimal
    import uuid

    from sqlalchemy import types

    # default type mapping, deriving the type for mapped_column()
    # from a Mapped[] annotation
    type_map: Dict[Type[Any], TypeEngine[Any]] = {
        bool: types.Boolean(),
        bytes: types.LargeBinary(),
        datetime.date: types.Date(),
        datetime.datetime: types.DateTime(),
        datetime.time: types.Time(),
        datetime.timedelta: types.Interval(),
        decimal.Decimal: types.Numeric(),
        float: types.Float(),
        int: types.Integer(),
        str: types.String(),
        uuid.UUID: types.Uuid(),
    }
    ```

    如果 `mapped_column()` 构造指示传递给 `mapped_column.__type` 参数的显式类型，则给定的 Python 类型将被忽略。

+   **nullability** - `mapped_column()` 构造将首先通过 `mapped_column.nullable` 参数的存在与设置为 `True` 或 `False` 来指示其 `Column` 为 `NULL` 或 `NOT NULL`。此外，如果 `mapped_column.primary_key` 参数存在并设置为 `True`，那也意味着该列应为 `NOT NULL`。

    在**两者**参数均不存在的情况下，`Mapped`类型注释中存在`typing.Optional[]`将用于确定可空性，其中`typing.Optional[]`表示`NULL`，而没有`typing.Optional[]`则表示`NOT NULL`。如果根本没有`Mapped[]`注释，并且没有`mapped_column.nullable`或`mapped_column.primary_key`参数，则 SQLAlchemy 对于`Column`的通常默认值`NULL`将被使用。

    在下面的示例中，`id`和`data`列将是`NOT NULL`，而`additional_info`列将是`NULL`：

    ```py
    from typing import Optional

    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column

    class Base(DeclarativeBase):
        pass

    class SomeClass(Base):
        __tablename__ = "some_table"

        # primary_key=True, therefore will be NOT NULL
        id: Mapped[int] = mapped_column(primary_key=True)

        # not Optional[], therefore will be NOT NULL
        data: Mapped[str]

        # Optional[], therefore will be NULL
        additional_info: Mapped[Optional[str]]
    ```

    也完全可以有一个其可空性与注释所暗示的不同的`mapped_column()`。例如，ORM 映射的属性可以在创建和填充对象时被注释为允许在 Python 代码中使用`None`，但是该值最终将被写入一个`NOT NULL`的数据库列。当存在时，`mapped_column.nullable`参数始终优先：

    ```py
    class SomeClass(Base):
        # ...

        # will be String() NOT NULL, but can be None in Python
        data: Mapped[Optional[str]] = mapped_column(nullable=False)
    ```

    类似地，写入数据库列的非`None`属性，由于某种原因需要在模式级别为 NULL，`mapped_column.nullable`可以设置为`True`：

    ```py
    class SomeClass(Base):
        # ...

        # will be String() NULL, but type checker will not expect
        # the attribute to be None
        data: Mapped[str] = mapped_column(nullable=True)
    ```  #### 自定义类型映射

在前一节中描述的 Python 类型到 SQLAlchemy `TypeEngine` 类型的映射默认为硬编码的字典，位于`sqlalchemy.sql.sqltypes`模块中。然而，协调声明映射过程的`registry`对象将首先查阅本地用户定义的类型字典，该字典可以在构造`registry`时作为参数`registry.type_annotation_map`传递，并且在首次使用时可以与`DeclarativeBase`超类相关联。

例如，如果我们希望使用`BIGINT`数据类型来表示`int`，带有`timezone=True`的`TIMESTAMP`数据类型表示`datetime.datetime`，然后仅在 Microsoft SQL Server 上使用`NVARCHAR`数据类型表示 Python 的`str`，则注册表和 Declarative base 可以配置为：

```py
import datetime

from sqlalchemy import BIGINT, Integer, NVARCHAR, String, TIMESTAMP
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped, mapped_column, registry

class Base(DeclarativeBase):
    type_annotation_map = {
        int: BIGINT,
        datetime.datetime: TIMESTAMP(timezone=True),
        str: String().with_variant(NVARCHAR, "mssql"),
    }

class SomeClass(Base):
    __tablename__ = "some_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    date: Mapped[datetime.datetime]
    status: Mapped[str]
```

下面说明了为上述映射生成的 CREATE TABLE 语句，首先是在 Microsoft SQL Server 后端，说明了`NVARCHAR`数据类型：

```py
>>> from sqlalchemy.schema import CreateTable
>>> from sqlalchemy.dialects import mssql, postgresql
>>> print(CreateTable(SomeClass.__table__).compile(dialect=mssql.dialect()))
CREATE  TABLE  some_table  (
  id  BIGINT  NOT  NULL  IDENTITY,
  date  TIMESTAMP  NOT  NULL,
  status  NVARCHAR(max)  NOT  NULL,
  PRIMARY  KEY  (id)
) 
```

然后在 PostgreSQL 后端，说明了`TIMESTAMP WITH TIME ZONE`：

```py
>>> print(CreateTable(SomeClass.__table__).compile(dialect=postgresql.dialect()))
CREATE  TABLE  some_table  (
  id  BIGSERIAL  NOT  NULL,
  date  TIMESTAMP  WITH  TIME  ZONE  NOT  NULL,
  status  VARCHAR  NOT  NULL,
  PRIMARY  KEY  (id)
) 
```

通过利用诸如`TypeEngine.with_variant()`之类的方法，我们能够建立一个类型映射，该映射根据不同的后端定制我们所需的内容，同时仍然能够使用简洁的仅注释的`mapped_column()`配置。除此之外，还有两个级别的 Python 类型可配置性，分别在下面的两个章节中描述。#### 将多种类型配置映射到 Python 类型

由于个别 Python 类型可能与任何类型的`TypeEngine`配置相关联，通过使用`registry.type_annotation_map`参数，额外的能力是能够将单个 Python 类型与基于额外类型限定符的 SQL 类型的不同变体相关联。其中一个典型示例是将 Python 的`str`数据类型映射到不同长度的`VARCHAR` SQL 类型。另一个是将不同种类的`decimal.Decimal`映射到不同大小的`NUMERIC`列。

Python 的类型系统提供了一种很好的方法来为 Python 类型添加额外的元数据，即使用[**PEP 593**](https://peps.python.org/pep-0593/) `Annotated`泛型类型，它允许将额外的信息与 Python 类型捆绑在一起。`mapped_column()` 构造将正确解释`Annotated`对象的身份，当在`registry.type_annotation_map`中解析它时，就像下面的示例中声明两个变体`String`和`Numeric`一样：

```py
from decimal import Decimal

from typing_extensions import Annotated

from sqlalchemy import Numeric
from sqlalchemy import String
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import registry

str_30 = Annotated[str, 30]
str_50 = Annotated[str, 50]
num_12_4 = Annotated[Decimal, 12]
num_6_2 = Annotated[Decimal, 6]

class Base(DeclarativeBase):
    registry = registry(
        type_annotation_map={
            str_30: String(30),
            str_50: String(50),
            num_12_4: Numeric(12, 4),
            num_6_2: Numeric(6, 2),
        }
    )
```

传递给`Annotated`容器的 Python 类型，在上述示例中为`str`和`Decimal`类型，仅对于类型工具的好处而重要；就`mapped_column()`构造而言，它只需要在`registry.type_annotation_map`字典中查找每个类型对象，而不实际查看`Annotated`对象的内部，至少在这个特定的上下文中是如此。类似地，传递给`Annotated`的参数超出了基础 Python 类型本身也不重要，只是`Annotated`构造必须存在至少一个参数才有效。然后，我们可以直接在映射中使用这些增强型类型，它们将与更具体的类型构造相匹配，如以下示例所示：

```py
class SomeClass(Base):
    __tablename__ = "some_table"

    short_name: Mapped[str_30] = mapped_column(primary_key=True)
    long_name: Mapped[str_50]
    num_value: Mapped[num_12_4]
    short_num_value: Mapped[num_6_2]
```

上述映射的 CREATE TABLE 将说明我们配置的不同变体的`VARCHAR`和`NUMERIC`，如下所示：

```py
>>> from sqlalchemy.schema import CreateTable
>>> print(CreateTable(SomeClass.__table__))
CREATE  TABLE  some_table  (
  short_name  VARCHAR(30)  NOT  NULL,
  long_name  VARCHAR(50)  NOT  NULL,
  num_value  NUMERIC(12,  4)  NOT  NULL,
  short_num_value  NUMERIC(6,  2)  NOT  NULL,
  PRIMARY  KEY  (short_name)
) 
```

尽管将`Annotated`类型与不同的 SQL 类型进行链接提供了广泛的灵活性，但下一节说明了`Annotated`可能与声明性一起使用的第二种方式，这种方式更加开放。#### 将整个列声明映射到 Python 类型

前一节说明了使用[**PEP 593**](https://peps.python.org/pep-0593/) `Annotated`类型实例作为`registry.type_annotation_map`字典中的键。在这种形式中，`mapped_column()`构造实际上不会查看`Annotated`对象本身，而是仅用作字典键。然而，声明性还具有直接从`Annotated`对象中提取整个预先建立的`mapped_column()`构造的能力。使用这种形式，我们不仅可以定义与 Python 类型链接的不同种类的 SQL 数据类型，而无需使用`registry.type_annotation_map`字典，还可以以可重用的方式设置任意数量的参数，例如可为空性、列默认值和约束。

一组 ORM 模型通常会具有一种对所有映射类都通用的主键风格。还可能存在一些常见的列配置，例如带有默认值的时间戳和其他预先设置大小和配置的字段。我们可以将这些配置组合成`mapped_column()`实例，然后直接将其捆绑到`Annotated`实例中，然后在任意数量的类声明中重复使用。当以这种方式提供`Annotated`对象时，Declarative 将解包该对象，跳过不适用于 SQLAlchemy 的任何其他指令，并仅搜索 SQLAlchemy ORM 构造。

下面的示例说明了以这种方式使用的各种预配置字段类型，其中我们定义了代表`Integer`主键列的`intpk`，代表将使用`CURRENT_TIMESTAMP`作为 DDL 级别列默认值的`DateTime`类型的`timestamp`，以及`required_name`，它是一个长度为 30 的`String`，不可为空：

```py
import datetime

from typing_extensions import Annotated

from sqlalchemy import func
from sqlalchemy import String
from sqlalchemy.orm import mapped_column

intpk = Annotated[int, mapped_column(primary_key=True)]
timestamp = Annotated[
    datetime.datetime,
    mapped_column(nullable=False, server_default=func.CURRENT_TIMESTAMP()),
]
required_name = Annotated[str, mapped_column(String(30), nullable=False)]
```

上述的`Annotated`对象然后可以直接在`Mapped`中使用，在那里，预配置的`mapped_column()`构造将被提取并复制到一个新实例中，该实例将特定于每个属性：

```py
class Base(DeclarativeBase):
    pass

class SomeClass(Base):
    __tablename__ = "some_table"

    id: Mapped[intpk]
    name: Mapped[required_name]
    created_at: Mapped[timestamp]
```

我们上面的映射的`CREATE TABLE`看起来像这样：

```py
>>> from sqlalchemy.schema import CreateTable
>>> print(CreateTable(SomeClass.__table__))
CREATE  TABLE  some_table  (
  id  INTEGER  NOT  NULL,
  name  VARCHAR(30)  NOT  NULL,
  created_at  DATETIME  DEFAULT  CURRENT_TIMESTAMP  NOT  NULL,
  PRIMARY  KEY  (id)
) 
```

当以这种方式使用`Annotated`类型时，类型的配置也可能会受到每个属性的影响。对于上面示例中明确使用了`mapped_column.nullable`的类型，我们可以将`Optional[]`泛型修饰符应用于我们的任何类型，以使字段在 Python 级别是可选的或不可选的，这将独立于在数据库中发生的`NULL` / `NOT NULL`设置：

```py
from typing_extensions import Annotated

import datetime
from typing import Optional

from sqlalchemy.orm import DeclarativeBase

timestamp = Annotated[
    datetime.datetime,
    mapped_column(nullable=False),
]

class Base(DeclarativeBase):
    pass

class SomeClass(Base):
    # ...

    # pep-484 type will be Optional, but column will be
    # NOT NULL
    created_at: Mapped[Optional[timestamp]]
```

`mapped_column()`构造也与显式传递的`mapped_column()`构造进行了调和，其参数将优先于`Annotated`构造的参数。下面我们向整数主键添加了一个`ForeignKey`约束，并且还为`created_at`列使用了一个替代的服务器默认值：

```py
import datetime

from typing_extensions import Annotated

from sqlalchemy import ForeignKey
from sqlalchemy import func
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.schema import CreateTable

intpk = Annotated[int, mapped_column(primary_key=True)]
timestamp = Annotated[
    datetime.datetime,
    mapped_column(nullable=False, server_default=func.CURRENT_TIMESTAMP()),
]

class Base(DeclarativeBase):
    pass

class Parent(Base):
    __tablename__ = "parent"

    id: Mapped[intpk]

class SomeClass(Base):
    __tablename__ = "some_table"

    # add ForeignKey to mapped_column(Integer, primary_key=True)
    id: Mapped[intpk] = mapped_column(ForeignKey("parent.id"))

    # change server default from CURRENT_TIMESTAMP to UTC_TIMESTAMP
    created_at: Mapped[timestamp] = mapped_column(server_default=func.UTC_TIMESTAMP())
```

创建表语句说明了这些属性设置，添加了一个`FOREIGN KEY`约束，并将`UTC_TIMESTAMP`替换为`CURRENT_TIMESTAMP`：

```py
>>> from sqlalchemy.schema import CreateTable
>>> print(CreateTable(SomeClass.__table__))
CREATE  TABLE  some_table  (
  id  INTEGER  NOT  NULL,
  created_at  DATETIME  DEFAULT  UTC_TIMESTAMP()  NOT  NULL,
  PRIMARY  KEY  (id),
  FOREIGN  KEY(id)  REFERENCES  parent  (id)
) 
```

注意

刚刚描述的`mapped_column()`功能，其中可以使用包含“模板”`mapped_column()`对象的[**PEP 593**](https://peps.python.org/pep-0593/) `Annotated`对象来指示完全构建的列参数集，目前尚未实现用于其他 ORM 构造，如`relationship()`和`composite()`。虽然这种功能在理论上是可能的，但目前尝试使用`Annotated`来指示`relationship()`等的进一步参数将在运行时引发`NotImplementedError`异常，但可能会在未来的版本中实现。#### 在类型映射中使用 Python `Enum` 或 pep-586 `Literal` 类型

新版本 2.0.0b4 中新增：- 添加了`Enum`支持

新版本 2.0.1 中新增：- 添加了`Literal`支持

用户定义的 Python 类型，这些类型派生自 Python 内置的`enum.Enum`以及`typing.Literal`类，在 ORM 声明映射中使用时会自动链接到 SQLAlchemy `Enum` 数据类型。下面的示例在`Mapped[]`构造函数中使用了自定义的`enum.Enum`：

```py
import enum

from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column

class Base(DeclarativeBase):
    pass

class Status(enum.Enum):
    PENDING = "pending"
    RECEIVED = "received"
    COMPLETED = "completed"

class SomeClass(Base):
    __tablename__ = "some_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    status: Mapped[Status]
```

在上面的示例中，映射属性`SomeClass.status`将链接到一个`Column`，其数据类型为`Enum(Status)`。我们可以在 PostgreSQL 数据库的 CREATE TABLE 输出中看到这一点：

```py
CREATE  TYPE  status  AS  ENUM  ('PENDING',  'RECEIVED',  'COMPLETED')

CREATE  TABLE  some_table  (
  id  SERIAL  NOT  NULL,
  status  status  NOT  NULL,
  PRIMARY  KEY  (id)
)
```

类似地，可以使用`typing.Literal`，使用由所有字符串组成的`typing.Literal`：

```py
from typing import Literal

from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column

class Base(DeclarativeBase):
    pass

Status = Literal["pending", "received", "completed"]

class SomeClass(Base):
    __tablename__ = "some_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    status: Mapped[Status]
```

在`registry.type_annotation_map`中使用的条目将基本的`enum.Enum` Python 类型以及`typing.Literal`类型链接到 SQLAlchemy `Enum` SQL 类型，使用一种特殊形式，该形式指示`Enum`数据类型应自动针对任意枚举类型进行配置。这种默认情况下隐式的配置将被明确指示为：

```py
import enum
import typing

import sqlalchemy
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    type_annotation_map = {
        enum.Enum: sqlalchemy.Enum(enum.Enum),
        typing.Literal: sqlalchemy.Enum(enum.Enum),
    }
```

在 Declarative 内部的解析逻辑能够将`enum.Enum`的子类以及`typing.Literal`的实例解析为与`registry.type_annotation_map`字典中的`enum.Enum`或`typing.Literal`条目匹配的类型注释。然后，`Enum` SQL 类型知道如何生成具有适当设置的已配置版本，包括默认字符串长度。如果传递的 `typing.Literal` 不仅包含字符串值，则会引发具有信息的错误。

##### 本机枚举和命名

`Enum.native_enum`参数指的是`Enum`数据类型是否应创建所谓的“本机”枚举，这在 MySQL/MariaDB 上是 `ENUM` 数据类型，在 PostgreSQL 上是由 `CREATE TYPE` 创建的新 `TYPE` 对象，或者是“非本机”枚举，这意味着将使用`VARCHAR`来创建数据类型。对于除 MySQL/MariaDB 或 PostgreSQL 外的后端，`VARCHAR` 在所有情况下都被使用（第三方方言可能有自己的行为）。

因为 PostgreSQL 的 `CREATE TYPE` 要求为要创建的类型指定一个显式名称，所以当使用隐式生成的`Enum`时，如果没有在映射中指定显式的 `Enum` 数据类型，就会存在特殊的回退逻辑：

1.  如果`Enum`链接到`enum.Enum`对象，则`Enum.native_enum`参数默认为`True`，并且枚举的名称将从`enum.Enum`数据类型的名称中获取。PostgreSQL 后端将假设使用此名称创建 `CREATE TYPE`。

1.  如果`Enum`链接到`typing.Literal`对象，则`Enum.native_enum`参数默认为`False`；不会生成名称，并且假设为`VARCHAR`。

要在 PostgreSQL 的 `CREATE TYPE` 类型中使用 `typing.Literal`，必须使用显式的`Enum`，可以在类型映射中：

```py
import enum
import typing

import sqlalchemy
from sqlalchemy.orm import DeclarativeBase

Status = Literal["pending", "received", "completed"]

class Base(DeclarativeBase):
    type_annotation_map = {
        Status: sqlalchemy.Enum("pending", "received", "completed", name="status_enum"),
    }
```

或者在`mapped_column()`内部：

```py
import enum
import typing

import sqlalchemy
from sqlalchemy.orm import DeclarativeBase

Status = Literal["pending", "received", "completed"]

class Base(DeclarativeBase):
    pass

class SomeClass(Base):
    __tablename__ = "some_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    status: Mapped[Status] = mapped_column(
        sqlalchemy.Enum("pending", "received", "completed", name="status_enum")
    )
```

##### 修改默认枚举的配置

为了修改隐式生成的`Enum`数据类型的固定配置，请在`registry.type_annotation_map`中指定新条目，表示附加参数。例如，要无条件使用“非本地枚举”，可以为所有类型将`Enum.native_enum`参数设置为 False：

```py
import enum
import typing
import sqlalchemy
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    type_annotation_map = {
        enum.Enum: sqlalchemy.Enum(enum.Enum, native_enum=False),
        typing.Literal: sqlalchemy.Enum(enum.Enum, native_enum=False),
    }
```

自 2.0.1 版本更改：在建立`registry.type_annotation_map`时，实现了对`Enum.native_enum`等参数进行覆盖的支持。以前，此功能未能正常工作。

要为特定的`enum.Enum`子类型使用特定的配置，例如在使用示例`Status`数据类型时将字符串长度设置为 50：

```py
import enum
import sqlalchemy
from sqlalchemy.orm import DeclarativeBase

class Status(enum.Enum):
    PENDING = "pending"
    RECEIVED = "received"
    COMPLETED = "completed"

class Base(DeclarativeBase):
    type_annotation_map = {
        Status: sqlalchemy.Enum(Status, length=50, native_enum=False)
    }
```

默认情况下，自动生成的`Enum`不与`Base`使用的`MetaData`实例关联，因此如果元数据定义了模式，则不会自动与枚举关联。要自动将枚举与元数据或其所属的表的模式关联起来，可以设置`Enum.inherit_schema`：

```py
from enum import Enum
import sqlalchemy as sa
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    metadata = sa.MetaData(schema="my_schema")
    type_annotation_map = {Enum: sa.Enum(Enum, inherit_schema=True)}
```

##### 将特定的`enum.Enum`或`typing.Literal`链接到其他数据类型

上述示例演示了使用自动配置自身到`enum.Enum`或`typing.Literal`类型对象上的参数/属性的`Enum`。对于特定种类的`enum.Enum`或`typing.Literal`应链接到其他类型的用例，这些特定类型也可以放置在类型映射中。在下面的示例中，包含非字符串类型的`Literal[]`条目与`JSON`数据类型相关联：

```py
from typing import Literal

from sqlalchemy import JSON
from sqlalchemy.orm import DeclarativeBase

my_literal = Literal[0, 1, True, False, "true", "false"]

class Base(DeclarativeBase):
    type_annotation_map = {my_literal: JSON}
```

在上述配置中，`my_literal`数据类型将解析为一个`JSON`实例。其他`Literal`变体将继续解析为`Enum`数据类型。

#### `mapped_column()`中的数据类功能

`mapped_column()`构造与 SQLAlchemy 的“原生数据类”功能集成，详见声明性数据类映射。查看该部分以获取关于`mapped_column()`支持的其他指令的当前背景。  ### 访问表和元数据

声明性映射类将始终包括一个名为`__table__`的属性；当使用上述使用`__tablename__`的配置完成时，声明过程会通过`__table__`属性使`Table`可用：

```py
# access the Table
user_table = User.__table__
```

上述表格最终与`Mapper.local_table`属性相对应，我们可以通过运行时检查系统看到它：

```py
from sqlalchemy import inspect

user_table = inspect(User).local_table
```

与声明性`registry`以及基类关联的`MetaData`集合通常是必要的，以便运行 DDL 操作，如 CREATE，以及与迁移工具（例如 Alembic）一起使用。此对象可通过`registry`以及声明性基类的`.metadata`属性获得。下面，对于一个小脚本，我们可能希望针对 SQLite 数据库发出所有表的 CREATE：

```py
engine = create_engine("sqlite://")

Base.metadata.create_all(engine)
```  ### 声明性表配置

使用带有`__tablename__`声明类属性的声明性表配置时，应使用`__table_args__`声明类属性提供要提供给`Table`构造函数的附加参数。

此属性既支持位置参数，也支持通常发送到`Table`构造函数的关键字参数。该属性可以用两种形式指定。一种是作为一个字典：

```py
class MyClass(Base):
    __tablename__ = "sometable"
    __table_args__ = {"mysql_engine": "InnoDB"}
```

另一个是一个元组，其中每个参数都是位置参数（通常是约束条件）：

```py
class MyClass(Base):
    __tablename__ = "sometable"
    __table_args__ = (
        ForeignKeyConstraint(["id"], ["remote_table.id"]),
        UniqueConstraint("foo"),
    )
```

关键字参数可以通过在上述形式中将最后一个参数指定为字典来指定：

```py
class MyClass(Base):
    __tablename__ = "sometable"
    __table_args__ = (
        ForeignKeyConstraint(["id"], ["remote_table.id"]),
        UniqueConstraint("foo"),
        {"autoload": True},
    )
```

类还可以使用`declared_attr()`方法装饰器以动态方式指定`__table_args__`声明属性，以及`__tablename__`属性。有关背景，请参阅使用混合组合映射层次结构。  ### 声明性表的显式架构名称

有关 `Table` 的模式名称，请参阅 指定模式名称，将模式名称应用于单个 `Table` 使用 `Table.schema` 参数。在使用声明性表时，此选项像其他任何选项一样传递给 `__table_args__` 字典：

```py
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    pass

class MyClass(Base):
    __tablename__ = "sometable"
    __table_args__ = {"schema": "some_schema"}
```

模式名称也可以通过在 指定默认模式名称与 MetaData 中使用的 `MetaData.schema` 参数应用于全局所有 `Table` 对象。 `MetaData` 对象可以单独构造，并通过直接赋值给 `metadata` 属性与 `DeclarativeBase` 子类关联起来：

```py
from sqlalchemy import MetaData
from sqlalchemy.orm import DeclarativeBase

metadata_obj = MetaData(schema="some_schema")

class Base(DeclarativeBase):
    metadata = metadata_obj

class MyClass(Base):
    # will use "some_schema" by default
    __tablename__ = "sometable"
```

另请参阅

指定模式名称 - 在 使用 MetaData 描述数据库 文档中。  ### 设置声明性映射列的加载和持久化选项

`mapped_column()` 构造函数接受其他影响生成的 `Column` 映射的 ORM 特定参数，影响其加载和持久化行为。常用的选项包括：

+   **延迟列加载** - `mapped_column.deferred` 布尔值默认情况下建立 `Column` 使用 延迟列加载。在下面的示例中，`User.bio` 列不会默认加载，而是在访问时加载：

    ```py
    class User(Base):
        __tablename__ = "user"

        id: Mapped[int] = mapped_column(primary_key=True)
        name: Mapped[str]
        bio: Mapped[str] = mapped_column(Text, deferred=True)
    ```

    另请参阅

    限制列的加载方式与列延迟 - 延迟列加载的完整描述

+   **活动历史** - `mapped_column.active_history` 确保在更改属性的值时，先前的值将已加载并作为属性历史的一部分放入 `AttributeState.history` 集合中，当检查属性的历史时，可能会产生额外的 SQL 语句：

    ```py
    class User(Base):
        __tablename__ = "user"

        id: Mapped[int] = mapped_column(primary_key=True)
        important_identifier: Mapped[str] = mapped_column(active_history=True)
    ```

请参阅 `mapped_column()` 的文档字符串以获取支持的参数列表。

另请参阅

对声明式表列应用加载、持久化和映射选项 - 描述了在 Imperative Table 配置中使用 `column_property()` 和 `deferred()` 的方法  ### 显式命名声明式映射列

到目前为止，所有的示例都是使用了 `mapped_column()` 构造与一个 ORM 映射的属性关联起来，其中给定给 `mapped_column()` 的 Python 属性名称也是 CREATE TABLE 语句以及查询中所见的列的名称。在 SQL 中表示列名可以通过将字符串位置参数 `mapped_column.__name` 作为第一个位置参数来指定。在下面的示例中，`User` 类被映射到了列本身的备用名称：

```py
class User(Base):
    __tablename__ = "user"

    id: Mapped[int] = mapped_column("user_id", primary_key=True)
    name: Mapped[str] = mapped_column("user_name")
```

`User.id` 对应的是一个名为 `user_id` 的列，而 `User.name` 对应的是一个名为 `user_name` 的列。我们可以使用我们的 Python 属性名称编写一个 `select()` 语句，使用的是我们的 Python 属性名称，我们将看到生成的 SQL 名称：

```py
>>> from sqlalchemy import select
>>> print(select(User.id, User.name).where(User.name == "x"))
SELECT  "user".user_id,  "user".user_name
FROM  "user"
WHERE  "user".user_name  =  :user_name_1 
```

另请参阅

映射表列的备用属性名称 - 适用于声明式表  ### 向现有的声明式映射类添加附加列

声明式表配置允许在已经生成了 `Table` 元数据之后，向现有映射中添加新的 `Column` 对象。

对于使用声明式基类声明的声明式类，底层元类`DeclarativeMeta`包括一个`__setattr__()`方法，该方法将拦截额外的`mapped_column()`或 Core `Column`对象，并将它们添加到`Table`和现有的`Mapper`中，分别使用`Table.append_column()`和`Mapper.add_property()`：

```py
MyClass.some_new_column = mapped_column(String)
```

使用核心`Column`：

```py
MyClass.some_new_column = Column(String)
```

支持所有参数，包括备用名称，例如`MyClass.some_new_column = mapped_column("some_name", String)`。但是，必须显式传递 SQL 类型给`mapped_column()`或`Column`对象，就像上面的例子中传递`String`类型一样。`Mapped`注解类型无法参与此操作。

在使用单表继承的特定情况下，也可以向映射添加额外的`Column`对象，在这种情况下，映射的子类上存在额外的列，但它们没有自己的`Table`。这在单表继承部分有所说明。

另请参阅

在声明后向映射类添加关系 - 类似的例子可以参考`relationship()`

注意

将映射属性分配给已经映射的类，只有在使用“声明性基类”，即用户定义的`DeclarativeBase`的子类，或者由`declarative_base()`或者`registry.generate_base()`返回的动态生成的类时，才能正确运行。这个“基类”包括一个 Python 元类，它实现了一个特殊的`__setattr__()`方法来拦截这些操作。

使用类映射属性对映射类进行运行时分配，如果使用装饰器，如`registry.mapped()`或者像`registry.map_imperatively()`这样的命令式函数进行类映射，则**无法**正常工作。## 声明式与命令式表格（又名混合声明式）

声明式映射也可以使用预先存在的`Table`对象，或者其他任意的`FromClause`构造（例如`Join`或`Subquery`），它是单独构造的。

这被称为“混合声明式”映射，因为类使用声明式风格进行所有涉及映射器配置的操作，然而映射的`Table`对象是单独生成的，并直接传递给声明性流程：

```py
from sqlalchemy import Column, ForeignKey, Integer, String
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    pass

# construct a Table directly.  The Base.metadata collection is
# usually a good choice for MetaData but any MetaData
# collection may be used.

user_table = Table(
    "user",
    Base.metadata,
    Column("id", Integer, primary_key=True),
    Column("name", String),
    Column("fullname", String),
    Column("nickname", String),
)

# construct the User class using this table.
class User(Base):
    __table__ = user_table
```

在上面的例子中，使用在用 MetaData 描述数据库中描述的方法构造了一个`Table`对象。然后可以直接将其应用于声明性映射的类。在这种形式中，不使用`__tablename__`和`__table_args__`声明性类属性。上述配置通常更易读，作为内联定义：

```py
class User(Base):
    __table__ = Table(
        "user",
        Base.metadata,
        Column("id", Integer, primary_key=True),
        Column("name", String),
        Column("fullname", String),
        Column("nickname", String),
    )
```

上述风格的自然结果是`__table__`属性本身在类定义块中被定义。因此，它可以立即在后续属性中被引用，例如下面的例子，说明在多态映射器配置中引用`type`列：

```py
class Person(Base):
    __table__ = Table(
        "person",
        Base.metadata,
        Column("id", Integer, primary_key=True),
        Column("name", String(50)),
        Column("type", String(50)),
    )

    __mapper_args__ = {
        "polymorphic_on": __table__.c.type,
        "polymorhpic_identity": "person",
    }
```

当要映射非`Table`构造时，例如`Join`或`Subquery`对象时，也使用“命令式表”形式。以下是一个示例：

```py
from sqlalchemy import func, select

subq = (
    select(
        func.count(orders.c.id).label("order_count"),
        func.max(orders.c.price).label("highest_order"),
        orders.c.customer_id,
    )
    .group_by(orders.c.customer_id)
    .subquery()
)

customer_select = (
    select(customers, subq)
    .join_from(customers, subq, customers.c.id == subq.c.customer_id)
    .subquery()
)

class Customer(Base):
    __table__ = customer_select
```

有关映射到非`Table`构造的背景，请参阅将类映射到多个表和将类映射到任意子查询一节。

当类本身使用替代形式的属性声明时，例如 Python 数据类时，“命令式表”形式特别有用。详见将 ORM 映射应用于现有数据类（传统数据类用法）一节。

另请参见

使用 MetaData 描述数据库

将 ORM 映射应用于现有数据类（传统数据类用法）

### 映射表列的替代属性名称

显式命名声明式映射的列一节说明了如何使用`mapped_column()`为生成的`Column`对象提供一个特定名称，与其映射的属性名称分开。

当使用命令式表配置时，我们已经有`Column`对象存在。为了将它们映射到替代名称，我们可以直接将`Column`分配给所需的属性：

```py
user_table = Table(
    "user",
    Base.metadata,
    Column("user_id", Integer, primary_key=True),
    Column("user_name", String),
)

class User(Base):
    __table__ = user_table

    id = user_table.c.user_id
    name = user_table.c.user_name
```

上面的`User`映射将通过`User.id`和`User.name`属性引用`"user_id"`和`"user_name"`列，方式与显式命名声明式映射的列一节所示相同。

对上述映射的一个注意事项是，当使用 [**PEP 484**](https://peps.python.org/pep-0484/) 类型工具时，对 `Column` 的直接内联链接将不会被正确类型化。解决此问题的一种策略是在 `column_property()` 函数中应用 `Column` 对象；虽然 `Mapper` 已经自动为其内部使用生成了此属性对象，但通过在类声明中命名它，类型工具将能够将属性与 `Mapped` 注释匹配起来：

```py
from sqlalchemy.orm import column_property
from sqlalchemy.orm import Mapped

class User(Base):
    __table__ = user_table

    id: Mapped[int] = column_property(user_table.c.user_id)
    name: Mapped[str] = column_property(user_table.c.user_name)
```

请参阅

显式命名声明式映射列 - 适用于声明式表  ### 对命令式表列应用加载、持久化和映射选项

在为声明式映射列设置加载和持久化选项一节中，讲述了如何在使用声明式表配置时设置加载和持久化选项时，使用 `mapped_column()` 构造。在使用命令式表配置时，我们已经有了现有的与之映射的 `Column` 对象。为了映射这些与额外参数一起的 `Column` 对象，这些参数特定于 ORM 映射，我们可以使用 `column_property()` 和 `deferred()` 构造以将额外参数与列关联起来。选项包括：

+   **推迟加载列** - `deferred()` 函数是使用 `column_property.deferred` 参数设置为 `True` 调用 `column_property()` 的速记方式；此构造默认使用 推迟加载列 来建立 `Column`。在下面的示例中，`User.bio` 列将不会默认加载，而只在访问时加载：

    ```py
    from sqlalchemy.orm import deferred

    user_table = Table(
        "user",
        Base.metadata,
        Column("id", Integer, primary_key=True),
        Column("name", String),
        Column("bio", Text),
    )

    class User(Base):
        __table__ = user_table

        bio = deferred(user_table.c.bio)
    ```

> 请参阅
> 
> 限制加载列与列推迟加载 - 推迟列加载的完整描述

+   **活动历史** - `column_property.active_history` 确保在属性值更改时，之前的值将已加载并成为检查属性历史时的`AttributeState.history`集合的一部分。这可能会产生额外的 SQL 语句：

    ```py
    from sqlalchemy.orm import deferred

    user_table = Table(
        "user",
        Base.metadata,
        Column("id", Integer, primary_key=True),
        Column("important_identifier", String),
    )

    class User(Base):
        __table__ = user_table

        important_identifier = column_property(
            user_table.c.important_identifier, active_history=True
        )
    ```

另请参阅

`column_property()` 构造对于类被映射到替代的 FROM 子句（例如连接和选择）的情况也很重要。有关这些情况的更多背景信息请参阅：

+   将类映射到多个表

+   SQL 表达式作为映射属性

对于使用`mapped_column()`进行声明式表配置，大多数选项都是直接可用的；请参阅设置声明式映射列的加载和持久化选项一节的示例。  ## 使用反射表声明式映射

有几种可用的模式，用于根据从数据库反射的一系列`Table`对象生成映射类，使用的是在反映数据库对象中描述的反射过程。

从数据库反射到表的简单方法是使用声明式混合映射，将`Table.autoload_with`参数传递给`Table`的构造函数：

```py
from sqlalchemy import create_engine
from sqlalchemy import Table
from sqlalchemy.orm import DeclarativeBase

engine = create_engine("postgresql+psycopg2://user:pass@hostname/my_existing_database")

class Base(DeclarativeBase):
    pass

class MyClass(Base):
    __table__ = Table(
        "mytable",
        Base.metadata,
        autoload_with=engine,
    )
```

上述模式的变体适用于许多表的情况，可以使用`MetaData.reflect()`方法一次反射完整的`Table`对象集合，然后从`MetaData`中引用它们：

```py
from sqlalchemy import create_engine
from sqlalchemy import Table
from sqlalchemy.orm import DeclarativeBase

engine = create_engine("postgresql+psycopg2://user:pass@hostname/my_existing_database")

class Base(DeclarativeBase):
    pass

Base.metadata.reflect(engine)

class MyClass(Base):
    __table__ = Base.metadata.tables["mytable"]
```

使用 `__table__` 方法的一个注意事项是，映射的类不能声明，直到表被反射，这需要数据库连接源在声明应用程序类时存在；典型情况下，类是在应用程序模块被导入时声明的，但是数据库连接直到应用程序开始运行代码时才可用，以便它可以使用配置信息并创建引擎。目前有两种解决此问题的方法，描述在下面的两个部分中。

### 使用 DeferredReflection

为了适应声明映射类的用例，可以稍后对表元数据进行反射的情况，提供了一个简单的扩展，称为 `DeferredReflection` mixin，它改变了声明映射过程，延迟到特殊的类级 `DeferredReflection.prepare()` 方法被调用，该方法将根据目标数据库执行反射过程，并将结果与声明表映射过程集成，即使用 `__tablename__` 属性的类：

```py
from sqlalchemy.ext.declarative import DeferredReflection
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    pass

class Reflected(DeferredReflection):
    __abstract__ = True

class Foo(Reflected, Base):
    __tablename__ = "foo"
    bars = relationship("Bar")

class Bar(Reflected, Base):
    __tablename__ = "bar"

    foo_id = mapped_column(Integer, ForeignKey("foo.id"))
```

在上述代码中，我们创建了一个名为 `Reflected` 的混合类，该类将作为声明性层次结构中的类的基础，当调用 `Reflected.prepare` 方法时应该被映射。在进行此映射之前，以上映射是不完整的，给定一个 `Engine`：

```py
engine = create_engine("postgresql+psycopg2://user:pass@hostname/my_existing_database")
Reflected.prepare(engine)
```

`Reflected` 类的目的是定义类应该被反射映射的范围。插件将在调用 `.prepare()` 的目标的子类树中搜索，并反射所有由声明类命名的表；目标数据库中不属于映射的表，且不通过外键约束与目标表相关的表将不会被反射。

### 使用自动映射

一个更自动化的解决方案是使用 自动映射 扩展来映射现有数据库，其中使用表反射。该扩展将从数据库模式生成整个映射的类，包括基于观察到的外键约束的类之间的关系。虽然它包含用于定制的挂钩，例如允许自定义类命名和关系命名方案的挂钩，但自动映射是面向迅速零配置的工作风格。如果应用程序希望具有完全明确的模型，并使用表反射，那么 DeferredReflection 类可能更可取，因为它的方法较少自动化。

另请参阅

自动映射

### 自动从反射表中命名列方案

当使用任何以前的反射技术时，我们有选择通过列映射的命名方案。 `Column` 对象包括一个参数 `Column.key`，它是一个字符串名称，确定此 `Column` 将以何种名称独立于列的 SQL 名称出现在 `Table.c` 集合中。如果未通过其他方式提供，例如在 映射表列的备用属性名称 中说明的那样，此键也将由 `Mapper` 用作将 `Column` 映射到的属性名称。

在使用表反射时，我们可以拦截将作为参数接收到的 `Column` 的参数，并应用我们需要的任何更改，包括 `.key` 属性，以及诸如数据类型之类的内容。

事件挂钩最容易与正在使用的 `MetaData` 对象相关联，如下所示：

```py
from sqlalchemy import event
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    pass

@event.listens_for(Base.metadata, "column_reflect")
def column_reflect(inspector, table, column_info):
    # set column.key = "attr_<lower_case_name>"
    column_info["key"] = "attr_%s" % column_info["name"].lower()
```

有了上述事件，`Column` 对象的反射将被我们添加新的“.key”元素的事件拦截，例如在下面的映射中：

```py
class MyClass(Base):
    __table__ = Table("some_table", Base.metadata, autoload_with=some_engine)
```

这种方法也适用于 `DeferredReflection` 基类以及 Automap 扩展。特别是对于 automap，请参阅部分 拦截列定义 以了解背景信息。

另请参阅

使用反射表声明式映射

`DDLEvents.column_reflect()`

拦截列定义 - 在 Automap 文档中  ### 映射到明确的主键列集

为了成功映射一个表，`Mapper` 构造始终要求至少标识一个列为该可选择项的“主键”。这样，当加载或持久化 ORM 对象时，它可以被放置在具有适当 标识键 的标识映射中。

在那些被映射的反射表不包含主键约束的情况下，以及在针对任意可选择项进行映射的一般情况下，可能不存在主键列的情况下，提供了 `Mapper.primary_key` 参数，以便可以将任何一组列配置为表的“主键”，就 ORM 映射而言。

给定一个针对现有 `Table` 对象进行命令式表映射的示例，其中该表没有声明的主键（可能在反射情景中出现），我们可以将这样的表映射如下示例所示：

```py
from sqlalchemy import Column
from sqlalchemy import MetaData
from sqlalchemy import String
from sqlalchemy import Table
from sqlalchemy import UniqueConstraint
from sqlalchemy.orm import DeclarativeBase

metadata = MetaData()
group_users = Table(
    "group_users",
    metadata,
    Column("user_id", String(40), nullable=False),
    Column("group_id", String(40), nullable=False),
    UniqueConstraint("user_id", "group_id"),
)

class Base(DeclarativeBase):
    pass

class GroupUsers(Base):
    __table__ = group_users
    __mapper_args__ = {"primary_key": [group_users.c.user_id, group_users.c.group_id]}
```

上文中，`group_users` 表是一种关联表，具有字符串列 `user_id` 和 `group_id`，但未设置主键；相反，只建立了一个 `UniqueConstraint` 来确保这两列表示唯一键。`Mapper` 不会自动检查唯一约束以用作主键；而是我们利用 `Mapper.primary_key` 参数，传递了一个 `[group_users.c.user_id, group_users.c.group_id]` 的集合，指示应使用这两列来构建 `GroupUsers` 类实例的标识键。### 映射表列的子集

有时，表反射可能会提供一个 `Table`，其中包含许多对我们的需求不重要且可以安全忽略的列。对于这样一个具有许多不需要在应用程序中引用的列的表，`Mapper.include_properties` 或 `Mapper.exclude_properties` 参数可以指示要映射的列的子集，其中目标 `Table` 中的其他列不会以任何方式被 ORM 考虑。示例：

```py
class User(Base):
    __table__ = user_table
    __mapper_args__ = {"include_properties": ["user_id", "user_name"]}
```

在上面的示例中，`User` 类将映射到 `user_table` 表，只包括 `user_id` 和 `user_name` 列 - 其余列不被引用。

同样地：

```py
class Address(Base):
    __table__ = address_table
    __mapper_args__ = {"exclude_properties": ["street", "city", "state", "zip"]}
```

将 `Address` 类映射到 `address_table` 表，包括除了 `street`、`city`、`state` 和 `zip` 之外的所有列。

如两个示例所示，列可以通过字符串名称或直接引用 `Column` 对象来引用。直接引用对象可能对明确性和解决映射到具有重复名称的多表构造时的歧义很有用：

```py
class User(Base):
    __table__ = user_table
    __mapper_args__ = {
        "include_properties": [user_table.c.user_id, user_table.c.user_name]
    }
```

当列未包含在映射中时，在执行 `select()` 或传统的 `Query` 对象时，这些列不会被引用在任何 SELECT 语句中，映射类中也不会有任何代表该列的映射属性；给定该名称的属性赋值将不会产生除普通 Python 属性赋值以外的效果。

然而，重要的是要注意，**模式级别的列默认值仍然会生效**，对于那些包含这些默认值的 `Column` 对象，即使它们被排除在 ORM 映射之外。

“模式级列默认值”指的是在列插入/更新默认值中描述的默认值，包括通过`Column.default`、`Column.onupdate`、`Column.server_default`和`Column.server_onupdate`参数配置的那些。这些构造仍然具有正常的效果，因为在`Column.default`和`Column.onupdate`的情况下，`Column`对象仍然存在于底层的`Table`上，因此当 ORM 发出 INSERT 或 UPDATE 时，允许默认函数发生作用，并且在`Column.server_default`和`Column.server_onupdate`的情况下，关系数据库本身会作为服务器端行为发出这些默认值。## 带有`mapped_column()`的声明性表格

在使用声明性时，要映射的类的主体在大多数情况下包括一个`__tablename__`属性，该属性指示应与映射一起生成的`Table`的字符串名称。然后，在类主体中使用了`mapped_column()`构造，该构造具有额外的 ORM 特定配置功能，这些功能不在普通的`Column`类中存在，以指示表中的列。下面的示例说明了在声明性映射中使用此构造的最基本用法：

```py
from sqlalchemy import Integer, String
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import mapped_column

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "user"

    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50), nullable=False)
    fullname = mapped_column(String)
    nickname = mapped_column(String(30))
```

在上面的示例中，`mapped_column()`构造作为类级别属性内联放置在类定义中。在声明类时，声明性映射过程将针对与声明性`Base`相关联的`MetaData`集合生成一个新的`Table`对象；然后每个`mapped_column()`的实例将用于在此过程中生成一个`Column`对象，该对象将成为此`Table`对象的`Table.columns`集合的一部分。

在上面的示例中，声明性将构建一个等效于以下内容的`Table`构造：

```py
# equivalent Table object produced
user_table = Table(
    "user",
    Base.metadata,
    Column("id", Integer, primary_key=True),
    Column("name", String(50)),
    Column("fullname", String()),
    Column("nickname", String(30)),
)
```

当上面的`User`类被映射时，可以直接通过`__table__`属性访问此`Table`对象；这在访问表和元数据中进一步描述。

`mapped_column()`构造接受`Column`构造接受的所有参数，以及额外的 ORM 特定参数。通常省略了`mapped_column.__name`字段，该字段指示数据库列的名称，因为声明性过程将使用赋予构造的属性名称，并将其分配为列的名称（在上面的示例中，这指的是名称`id`、`name`、`fullname`、`nickname`）。分配替代的`mapped_column.__name`也是有效的，在这种情况下，生成的`Column`将在 SQL 和 DDL 语句中使用给定的名称，而`User`映射类将继续允许使用给定的属性名称访问属性，独立于列本身的名称（有关此处更多信息，请参阅显式命名声明性映射列）。

提示

`mapped_column()` 构造仅在声明式类映射内有效。在使用 Core 构造 `Table` 对象以及使用命令式表配置时，仍然需要 `Column` 构造来指示数据库列的存在。

参见

映射表列 - 包含了关于影响 `Mapper` 解释传入的 `Column` 对象的附加说明。

### 使用带注释的声明式表（`mapped_column()`的类型注释形式）

`mapped_column()` 构造能够从与声明式映射类中声明的属性关联的 [**PEP 484**](https://peps.python.org/pep-0484/) 类型注释中导出其列配置信息。如果使用了这些类型注释，则**必须**存在于一个称为`Mapped`的特殊 SQLAlchemy 类型中，这是一个[泛型](https://peps.python.org/pep-0484/#generics)类型，然后在其中指示一个特定的 Python 类型。

下面说明了从前一节映射的映射，增加了使用 `Mapped` 的情况：

```py
from typing import Optional

from sqlalchemy import String
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "user"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(50))
    fullname: Mapped[Optional[str]]
    nickname: Mapped[Optional[str]] = mapped_column(String(30))
```

在上面，当声明式处理每个类属性时，如果存在的话，每个 `mapped_column()` 将从左侧对应的 `Mapped` 类型注释派生出额外的参数。此外，当遇到一个没有为属性分配值的 `Mapped` 类型注释时（此形式受到 Python [dataclasses](https://docs.python.org/3/library/dataclasses.html) 中使用的相似风格的启发），声明式将隐式地生成一个空的 `mapped_column()` 指令；此 `mapped_column()` 构造随后从存在的 `Mapped` 注解中派生其配置。

#### `mapped_column()` 从 `Mapped` 注解中派生数据类型和可空性

`mapped_column()` 从 `Mapped` 注释中派生的两个特性是：

+   **数据类型** - 在 `Mapped` 中给出的 Python 类型，如果存在 `typing.Optional` 构造，则与 `TypeEngine` 的子类关联，例如 `Integer`、`String`、`DateTime` 或 `Uuid` 等常见类型。

    数据类型是根据 Python 类型到 SQLAlchemy 数据类型的字典确定的。这个字典是完全可定制的，如下一节自定义类型映射中所详细描述的。默认类型映射实现如下面的代码示例所示：

    ```py
    from typing import Any
    from typing import Dict
    from typing import Type

    import datetime
    import decimal
    import uuid

    from sqlalchemy import types

    # default type mapping, deriving the type for mapped_column()
    # from a Mapped[] annotation
    type_map: Dict[Type[Any], TypeEngine[Any]] = {
        bool: types.Boolean(),
        bytes: types.LargeBinary(),
        datetime.date: types.Date(),
        datetime.datetime: types.DateTime(),
        datetime.time: types.Time(),
        datetime.timedelta: types.Interval(),
        decimal.Decimal: types.Numeric(),
        float: types.Float(),
        int: types.Integer(),
        str: types.String(),
        uuid.UUID: types.Uuid(),
    }
    ```

    如果 `mapped_column()` 构造指示了作为参数传递给 `mapped_column.__type` 的显式类型，则给定的 Python 类型将被忽略。

+   **可空性** - `mapped_column()` 构造将通过存在的 `mapped_column.nullable` 参数首先和主要指示其 `Column` 是否为 `NULL` 或 `NOT NULL`，可以传递为 `True` 或 `False`。此外，如果存在并且将 `mapped_column.primary_key` 参数设置为 `True`，那么这也将意味着该列应为 `NOT NULL`。

    如果这两个参数都不存在，那么 `Mapped` 类型注释中的 `typing.Optional[]` 的存在将用于确定可空性，其中 `typing.Optional[]` 表示 `NULL`，而不存在 `typing.Optional[]` 则表示 `NOT NULL`。如果根本没有存在 `Mapped[]` 注释，并且也没有 `mapped_column.nullable` 或 `mapped_column.primary_key` 参数，则 SQLAlchemy 在 `Column` 的默认值 `NULL` 被使用。

    在下面的示例中，`id` 和 `data` 列将是 `NOT NULL`，而 `additional_info` 列将是 `NULL`：

    ```py
    from typing import Optional

    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column

    class Base(DeclarativeBase):
        pass

    class SomeClass(Base):
        __tablename__ = "some_table"

        # primary_key=True, therefore will be NOT NULL
        id: Mapped[int] = mapped_column(primary_key=True)

        # not Optional[], therefore will be NOT NULL
        data: Mapped[str]

        # Optional[], therefore will be NULL
        additional_info: Mapped[Optional[str]]
    ```

    一个 `mapped_column()` 也可以存在，其空值属性与注释所暗示的不同。例如，ORM 映射属性在 Python 代码中被注释为允许 `None`，该代码在对象首次创建和填充时使用，但最终的值将写入一个 `NOT NULL` 的数据库列。当存在 `mapped_column.nullable` 参数时，该参数始终优先：

    ```py
    class SomeClass(Base):
        # ...

        # will be String() NOT NULL, but can be None in Python
        data: Mapped[Optional[str]] = mapped_column(nullable=False)
    ```

    同样，写入数据库列的非 `None` 属性，如果出于某种原因需要在架构级别为 `NULL`，则可以将 `mapped_column.nullable` 设置为 `True`：

    ```py
    class SomeClass(Base):
        # ...

        # will be String() NULL, but type checker will not expect
        # the attribute to be None
        data: Mapped[str] = mapped_column(nullable=True)
    ```  #### 自定义类型映射

在上一节描述的 Python 类型到 SQLAlchemy `TypeEngine` 类型的映射中，默认为 `sqlalchemy.sql.sqltypes` 模块中的硬编码字典。然而，协调 Declarative 映射过程的 `registry` 对象首先将会查看一个本地的、用户定义的类型字典，该字典可以在构造 `registry` 时作为 `registry.type_annotation_map` 参数传递，该参数可以与首次使用时与 `DeclarativeBase` 超类相关联。

举例来说，如果我们希望对`int`使用`BIGINT`数据类型，对`datetime.datetime`使用带有`timezone=True`的`TIMESTAMP`数据类型，然后只在 Microsoft SQL Server 上当 Python `str`被使用时我们希望使用`NVARCHAR`数据类型，那么注册表和声明基础可以配置为：

```py
import datetime

from sqlalchemy import BIGINT, Integer, NVARCHAR, String, TIMESTAMP
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped, mapped_column, registry

class Base(DeclarativeBase):
    type_annotation_map = {
        int: BIGINT,
        datetime.datetime: TIMESTAMP(timezone=True),
        str: String().with_variant(NVARCHAR, "mssql"),
    }

class SomeClass(Base):
    __tablename__ = "some_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    date: Mapped[datetime.datetime]
    status: Mapped[str]
```

下面说明了为上述映射生成的`CREATE TABLE`语句，在 Microsoft SQL Server 后端首先展示了`NVARCHAR`数据类型：

```py
>>> from sqlalchemy.schema import CreateTable
>>> from sqlalchemy.dialects import mssql, postgresql
>>> print(CreateTable(SomeClass.__table__).compile(dialect=mssql.dialect()))
CREATE  TABLE  some_table  (
  id  BIGINT  NOT  NULL  IDENTITY,
  date  TIMESTAMP  NOT  NULL,
  status  NVARCHAR(max)  NOT  NULL,
  PRIMARY  KEY  (id)
) 
```

然后在 PostgreSQL 后端，展示了`TIMESTAMP WITH TIME ZONE`：

```py
>>> print(CreateTable(SomeClass.__table__).compile(dialect=postgresql.dialect()))
CREATE  TABLE  some_table  (
  id  BIGSERIAL  NOT  NULL,
  date  TIMESTAMP  WITH  TIME  ZONE  NOT  NULL,
  status  VARCHAR  NOT  NULL,
  PRIMARY  KEY  (id)
) 
```

通过使用诸如`TypeEngine.with_variant()`之类的方法，我们能够构建一个针对不同后端定制的类型映射，同时仍然能够使用简洁的注解方式配置`mapped_column()`。除此之外，还有两个级别的 Python 类型可配置性，将在接下来的两个部分中描述。#### 将多个类型配置映射到 Python 类型

由于个别 Python 类型可以通过使用`registry.type_annotation_map`参数与任何类型的`TypeEngine`配置相关联，另一个功能是根据附加类型限定符将单个 Python 类型与 SQL 类型的不同变体关联起来的能力。其中一个典型示例是将 Python `str`数据类型映射到不同长度的`VARCHAR` SQL 类型。另一个示例是将不同种类的`decimal.Decimal`映射到不同大小的`NUMERIC`列。

Python 的类型系统提供了一种很好的方式来为 Python 类型添加附加元数据，即使用[**PEP 593**](https://peps.python.org/pep-0593/) `Annotated`泛型类型，它允许将附加信息与 Python 类型捆绑在一起。当在`registry.type_annotation_map`中解析时，`mapped_column()`构造将正确地识别`Annotated`对象的标识，就像下面的示例中声明两个`String`和`Numeric`变体时一样：

```py
from decimal import Decimal

from typing_extensions import Annotated

from sqlalchemy import Numeric
from sqlalchemy import String
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import registry

str_30 = Annotated[str, 30]
str_50 = Annotated[str, 50]
num_12_4 = Annotated[Decimal, 12]
num_6_2 = Annotated[Decimal, 6]

class Base(DeclarativeBase):
    registry = registry(
        type_annotation_map={
            str_30: String(30),
            str_50: String(50),
            num_12_4: Numeric(12, 4),
            num_6_2: Numeric(6, 2),
        }
    )
```

传递给`Annotated`容器的 Python 类型，在上面的示例中是`str`和`Decimal`类型，仅对于类型工具的好处是重要的；就`mapped_column()`结构而言，它只需要在`registry.type_annotation_map`字典中查找每个类型对象，而不实际查看`Annotated`对象的内部，至少在这种特定上下文中是如此。类似地，传递给`Annotated`的除了底层 Python 类型之外的参数也不重要，只是至少必须存在一个参数才能使`Annotated`构造有效。然后，我们可以直接在我们的映射中使用这些增强型类型，它们将与更具体的类型构造匹配，就像以下示例中所示：

```py
class SomeClass(Base):
    __tablename__ = "some_table"

    short_name: Mapped[str_30] = mapped_column(primary_key=True)
    long_name: Mapped[str_50]
    num_value: Mapped[num_12_4]
    short_num_value: Mapped[num_6_2]
```

上述映射的 CREATE TABLE 将说明我们配置的不同变体的`VARCHAR`和`NUMERIC`，并且如下所示：

```py
>>> from sqlalchemy.schema import CreateTable
>>> print(CreateTable(SomeClass.__table__))
CREATE  TABLE  some_table  (
  short_name  VARCHAR(30)  NOT  NULL,
  long_name  VARCHAR(50)  NOT  NULL,
  num_value  NUMERIC(12,  4)  NOT  NULL,
  short_num_value  NUMERIC(6,  2)  NOT  NULL,
  PRIMARY  KEY  (short_name)
) 
```

虽然将`Annotated`类型与不同的 SQL 类型进行链接的多样性为我们提供了广泛的灵活性，但下一节将说明另一种使用 `Annotated` 与声明式的方式，这种方式甚至更加开放。#### 将整个列声明映射到 Python 类型

上一节详细介绍了如何使用[**PEP 593**](https://peps.python.org/pep-0593/)中的`Annotated`类型实例作为`registry.type_annotation_map`字典中的键。在这种形式下，`mapped_column()`结构实际上并不查看`Annotated`对象本身，而是仅用作字典键。然而，声明式还具有直接从`Annotated`对象中提取整个预先建立的`mapped_column()`结构的能力。使用这种形式，我们不仅可以定义与 Python 类型链接的不同类型的 SQL 数据类型，而无需使用`registry.type_annotation_map`字典，还可以以可重用的方式设置任意数量的参数，例如可空性、列默认值和约束。

一组 ORM 模型通常会有一种对所有映射类都通用的主键样式。还可能有常见的列配置，例如具有默认值的时间戳和其他预先设置大小和配置的字段。我们可以将这些配置组合成`mapped_column()`实例，然后直接捆绑到`Annotated`的实例中，然后在任意数量的类声明中重复使用。当以这种方式提供`Annotated`对象时，Declarative 将解开一个`Annotated`对象，跳过任何不适用于 SQLAlchemy 的其他指令，并仅搜索 SQLAlchemy ORM 构造。

下面的示例说明了以这种方式使用的各种预配置字段类型，我们在这里定义了`intpk`，表示一个`Integer`主键列，`timestamp`表示一个`DateTime`类型，它将使用`CURRENT_TIMESTAMP`作为 DDL 级别列默认值，并且 `required_name` 是一个长度为 30 的`String`，它是 `NOT NULL` 的：

```py
import datetime

from typing_extensions import Annotated

from sqlalchemy import func
from sqlalchemy import String
from sqlalchemy.orm import mapped_column

intpk = Annotated[int, mapped_column(primary_key=True)]
timestamp = Annotated[
    datetime.datetime,
    mapped_column(nullable=False, server_default=func.CURRENT_TIMESTAMP()),
]
required_name = Annotated[str, mapped_column(String(30), nullable=False)]
```

上述的`Annotated`对象可以直接在`Mapped`内使用，在这里，预先配置的`mapped_column()`构造将被提取并复制到一个新的实例中，该实例将针对每个属性具体化：

```py
class Base(DeclarativeBase):
    pass

class SomeClass(Base):
    __tablename__ = "some_table"

    id: Mapped[intpk]
    name: Mapped[required_name]
    created_at: Mapped[timestamp]
```

我们上面映射的`CREATE TABLE`看起来像：

```py
>>> from sqlalchemy.schema import CreateTable
>>> print(CreateTable(SomeClass.__table__))
CREATE  TABLE  some_table  (
  id  INTEGER  NOT  NULL,
  name  VARCHAR(30)  NOT  NULL,
  created_at  DATETIME  DEFAULT  CURRENT_TIMESTAMP  NOT  NULL,
  PRIMARY  KEY  (id)
) 
```

当以这种方式使用`Annotated`类型时，类型的配置也可能会受到每个属性的影响。对于上面示例中显式使用`mapped_column.nullable`的类型，我们可以对任何类型应用`Optional[]`通用修饰符，以使该字段在 Python 级别是可选的还是不可选的，这将独立于在数据库中发生的`NULL` / `NOT NULL`设置：

```py
from typing_extensions import Annotated

import datetime
from typing import Optional

from sqlalchemy.orm import DeclarativeBase

timestamp = Annotated[
    datetime.datetime,
    mapped_column(nullable=False),
]

class Base(DeclarativeBase):
    pass

class SomeClass(Base):
    # ...

    # pep-484 type will be Optional, but column will be
    # NOT NULL
    created_at: Mapped[Optional[timestamp]]
```

`mapped_column()`构造也与显式传递的`mapped_column()`构造相调和，其参数将优先于`Annotated`构造的参数。下面我们给我们的整数主键添加了一个`ForeignKey`约束，并且还为`created_at`列使用了一个备用服务器默认值：

```py
import datetime

from typing_extensions import Annotated

from sqlalchemy import ForeignKey
from sqlalchemy import func
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.schema import CreateTable

intpk = Annotated[int, mapped_column(primary_key=True)]
timestamp = Annotated[
    datetime.datetime,
    mapped_column(nullable=False, server_default=func.CURRENT_TIMESTAMP()),
]

class Base(DeclarativeBase):
    pass

class Parent(Base):
    __tablename__ = "parent"

    id: Mapped[intpk]

class SomeClass(Base):
    __tablename__ = "some_table"

    # add ForeignKey to mapped_column(Integer, primary_key=True)
    id: Mapped[intpk] = mapped_column(ForeignKey("parent.id"))

    # change server default from CURRENT_TIMESTAMP to UTC_TIMESTAMP
    created_at: Mapped[timestamp] = mapped_column(server_default=func.UTC_TIMESTAMP())
```

`CREATE TABLE`语句说明了这些每个属性的设置，除此之外，还添加了一个`FOREIGN KEY`约束，并将`UTC_TIMESTAMP`替换为`CURRENT_TIMESTAMP`：

```py
>>> from sqlalchemy.schema import CreateTable
>>> print(CreateTable(SomeClass.__table__))
CREATE  TABLE  some_table  (
  id  INTEGER  NOT  NULL,
  created_at  DATETIME  DEFAULT  UTC_TIMESTAMP()  NOT  NULL,
  PRIMARY  KEY  (id),
  FOREIGN  KEY(id)  REFERENCES  parent  (id)
) 
```

注意

刚才描述的 `mapped_column()` 功能，其中可以使用 [**PEP 593**](https://peps.python.org/pep-0593/) 的 `Annotated` 对象指示一组完整构造的列参数，该对象包含一个“模板” `mapped_column()` 对象，将被复制到属性中，目前尚未针对其他 ORM 构造（例如 `relationship()` 和 `composite()`）实现。虽然这种功能理论上是可能的，但目前尝试使用 `Annotated` 来指示对 `relationship()` 等的进一步参数将在运行时引发 `NotImplementedError` 异常，但可能会在未来的版本中实现。  #### 在类型映射中使用 Python `Enum` 或 pep-586 `Literal` 类型

新版本 2.0.0b4 中新增：- 添加了 `Enum` 支持

新版本 2.0.1 中新增：- 添加了 `Literal` 支持

当在 ORM 声明映射中使用时，用户定义的 Python 类型，其派生自 Python 内置的 `enum.Enum` 类以及 `typing.Literal` 类，将自动链接到 SQLAlchemy `Enum` 数据类型。下面的示例在 `Mapped[]` 构造函数中使用了自定义的 `enum.Enum`：

```py
import enum

from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column

class Base(DeclarativeBase):
    pass

class Status(enum.Enum):
    PENDING = "pending"
    RECEIVED = "received"
    COMPLETED = "completed"

class SomeClass(Base):
    __tablename__ = "some_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    status: Mapped[Status]
```

在上面的示例中，映射属性 `SomeClass.status` 将链接到一个 `Column`，其数据类型为 `Enum(Status)`。我们可以在 PostgreSQL 数据库的 CREATE TABLE 输出中看到这一点：

```py
CREATE  TYPE  status  AS  ENUM  ('PENDING',  'RECEIVED',  'COMPLETED')

CREATE  TABLE  some_table  (
  id  SERIAL  NOT  NULL,
  status  status  NOT  NULL,
  PRIMARY  KEY  (id)
)
```

类似地，也可以使用 `typing.Literal`，其中包含了所有字符串：

```py
from typing import Literal

from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column

class Base(DeclarativeBase):
    pass

Status = Literal["pending", "received", "completed"]

class SomeClass(Base):
    __tablename__ = "some_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    status: Mapped[Status]
```

在 `registry.type_annotation_map` 中使用的条目将基本的 `enum.Enum` Python 类型以及 `typing.Literal` 类型链接到 SQLAlchemy `Enum` SQL 类型，使用的是一种特殊的形式，指示 `Enum` 数据类型应自动配置自己以适应任意枚举类型。默认情况下，这种配置是隐式的，但可以显式指示如下：

```py
import enum
import typing

import sqlalchemy
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    type_annotation_map = {
        enum.Enum: sqlalchemy.Enum(enum.Enum),
        typing.Literal: sqlalchemy.Enum(enum.Enum),
    }
```

在 Declarative 内部的解析逻辑能够解析 `enum.Enum` 的子类以及 `typing.Literal` 的实例，以匹配 `registry.type_annotation_map` 字典中的 `enum.Enum` 或 `typing.Literal` 条目。然后，`Enum` SQL 类型知道如何生成具有适当设置的配置版本，包括默认字符串长度。如果传递了不仅由字符串值组成的 `typing.Literal`，则会引发详细的错误。

##### 本地枚举和命名

`Enum.native_enum` 参数指的是 `Enum` 数据类型是否应该创建所谓的“本机”枚举，在 MySQL/MariaDB 上是 `ENUM` 数据类型，在 PostgreSQL 上是通过 `CREATE TYPE` 创建的新 `TYPE` 对象，或者“非本机”枚举，这意味着将使用 `VARCHAR` 创建数据类型。对于除 MySQL/MariaDB 或 PostgreSQL 之外的后端，无论如何都使用 `VARCHAR`（第三方方言可能具有自己的行为）。

因为 PostgreSQL 的 `CREATE TYPE` 要求必须为要创建的类型指定显式名称，所以在使用隐式生成的 `Enum` 而没有在映射中指定显式的 `Enum` 数据类型时存在特殊的回退逻辑：

1.  如果 `Enum` 链接到 `enum.Enum` 对象，则 `Enum.native_enum` 参数默认为 `True`，枚举的名称将从 `enum.Enum` 数据类型的名称中取。PostgreSQL 后端将假定使用此名称创建 `CREATE TYPE`。

1.  如果 `Enum` 链接到 `typing.Literal` 对象，则 `Enum.native_enum` 参数默认为 `False`；不生成名称，并且假定为 `VARCHAR`。

要在 PostgreSQL 的 `CREATE TYPE` 类型中使用 `typing.Literal`，必须使用显式的 `Enum`，可以在类型映射中：

```py
import enum
import typing

import sqlalchemy
from sqlalchemy.orm import DeclarativeBase

Status = Literal["pending", "received", "completed"]

class Base(DeclarativeBase):
    type_annotation_map = {
        Status: sqlalchemy.Enum("pending", "received", "completed", name="status_enum"),
    }
```

或者在 `mapped_column()` 中另外使用：

```py
import enum
import typing

import sqlalchemy
from sqlalchemy.orm import DeclarativeBase

Status = Literal["pending", "received", "completed"]

class Base(DeclarativeBase):
    pass

class SomeClass(Base):
    __tablename__ = "some_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    status: Mapped[Status] = mapped_column(
        sqlalchemy.Enum("pending", "received", "completed", name="status_enum")
    )
```

##### 修改默认枚举的配置

为了修改隐式生成的`Enum`数据类型的固定配置，可以在`registry.type_annotation_map`中指定新条目，指示附加参数。例如，要无条件使用“非原生枚举”，可以为所有类型设置`Enum.native_enum`参数为 False：

```py
import enum
import typing
import sqlalchemy
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    type_annotation_map = {
        enum.Enum: sqlalchemy.Enum(enum.Enum, native_enum=False),
        typing.Literal: sqlalchemy.Enum(enum.Enum, native_enum=False),
    }
```

在 2.0.1 版本中更改：在建立`Enum`数据类型时，实现了覆盖参数（如`Enum.native_enum`）的支持在`registry.type_annotation_map`中。先前，此功能无效。

要为特定的`enum.Enum`子类型使用特定的配置，例如在使用示例`Status`数据类型时将字符串长度设置为 50：

```py
import enum
import sqlalchemy
from sqlalchemy.orm import DeclarativeBase

class Status(enum.Enum):
    PENDING = "pending"
    RECEIVED = "received"
    COMPLETED = "completed"

class Base(DeclarativeBase):
    type_annotation_map = {
        Status: sqlalchemy.Enum(Status, length=50, native_enum=False)
    }
```

默认情况下，自动生成的`Enum`不与`Base`使用的`MetaData`实例关联，因此，如果元数据定义了模式，它将不会自动与枚举关联。要自动将枚举与元数据中的模式或表关联起来，可以设置`Enum.inherit_schema`：

```py
from enum import Enum
import sqlalchemy as sa
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    metadata = sa.MetaData(schema="my_schema")
    type_annotation_map = {Enum: sa.Enum(Enum, inherit_schema=True)}
```

##### 将特定的`enum.Enum`或`typing.Literal`链接到其他数据类型

上述示例展示了自动配置自身到`enum.Enum`或`typing.Literal`类型对象的`Enum`的使用。对于特定种类的`enum.Enum`或`typing.Literal`应链接到其他类型的用例，也可以将这些特定类型放置在类型映射中。在下面的示例中，一个包含非字符串类型的`Literal[]`条目链接到了`JSON`数据类型：

```py
from typing import Literal

from sqlalchemy import JSON
from sqlalchemy.orm import DeclarativeBase

my_literal = Literal[0, 1, True, False, "true", "false"]

class Base(DeclarativeBase):
    type_annotation_map = {my_literal: JSON}
```

在上述配置中，`my_literal`数据类型将解析为一个`JSON`实例。其他`Literal`变体将继续解析为`Enum`数据类型。

#### 在`mapped_column()`中的 Dataclass 特性

`mapped_column()`结构与 SQLAlchemy 的“本地数据类”功能集成，该功能在声明性数据类映射中讨论。有关`mapped_column()`支持的其他指令的当前背景，请参阅该部分。 ### 访问表和元数据

一个声明性映射的类始终会包含一个名为`__table__`的属性；当上述配置使用`__tablename__`完成时，声明性过程通过`__table__`属性使`Table`可用：

```py
# access the Table
user_table = User.__table__
```

上述表最终是与`Mapper.local_table`属性相对应的表，我们可以通过运行时检查系统看到这一点：

```py
from sqlalchemy import inspect

user_table = inspect(User).local_table
```

与声明性`registry`以及基类关联的`MetaData`集合通常是运行 DDL 操作（例如 CREATE）以及与诸如 Alembic 之类的迁移工具一起使用的必要对象。此对象可通过`registry`以及声明性基类的`.metadata`属性获得。下面，对于一个小脚本，我们可能希望针对 SQLite 数据库发出所有表的 CREATE：

```py
engine = create_engine("sqlite://")

Base.metadata.create_all(engine)
```  ### 声明性表配置

当使用`__tablename__`声明性类属性进行声明性表配置时，应使用`__table_args__`声明性类属性提供额外的参数供`Table`构造函数使用。

此属性可容纳通常发送到`Table`构造函数的位置参数和关键字参数。该属性可以以两种形式之一指定。一种是作为字典：

```py
class MyClass(Base):
    __tablename__ = "sometable"
    __table_args__ = {"mysql_engine": "InnoDB"}
```

另一种是元组，其中每个参数是位置参数（通常是约束）：

```py
class MyClass(Base):
    __tablename__ = "sometable"
    __table_args__ = (
        ForeignKeyConstraint(["id"], ["remote_table.id"]),
        UniqueConstraint("foo"),
    )
```

通过将最后一个参数指定为字典，可以使用上述形式指定关键字参数：

```py
class MyClass(Base):
    __tablename__ = "sometable"
    __table_args__ = (
        ForeignKeyConstraint(["id"], ["remote_table.id"]),
        UniqueConstraint("foo"),
        {"autoload": True},
    )
```

类还可以使用`declared_attr()`方法装饰器以动态方式指定`__table_args__`声明属性和`__tablename__`属性。有关背景，请参见使用 Mixin 构建映射层级 ### 使用声明性表的显式模式名称

如文档中所述，`Table`的模式名称应用于单个`Table`，使用`Table.schema`参数。在使用声明式表时，此选项像任何其他选项一样传递给`__table_args__`字典：

```py
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    pass

class MyClass(Base):
    __tablename__ = "sometable"
    __table_args__ = {"schema": "some_schema"}
```

模式名称也可以通过使用文档化的`MetaData.schema`参数全局应用于所有`Table`对象。`MetaData`对象可以单独构建，并通过直接赋值给`metadata`属性与`DeclarativeBase`子类关联：

```py
from sqlalchemy import MetaData
from sqlalchemy.orm import DeclarativeBase

metadata_obj = MetaData(schema="some_schema")

class Base(DeclarativeBase):
    metadata = metadata_obj

class MyClass(Base):
    # will use "some_schema" by default
    __tablename__ = "sometable"
```

另请参见

指定模式名称 - 在使用 MetaData 描述数据库文档中。### 设置声明式映射列的加载和持久化选项

`mapped_column()` 构造函数接受额外的与 ORM 相关的参数，影响生成的`Column`的映射方式，影响其加载和持久化行为。常用的选项包括：

+   **延迟列加载** - `mapped_column.deferred` 布尔值默认使用延迟列加载来建立`Column`。在下面的示例中，`User.bio`列默认不会被加载，只有在访问时才会加载：

    ```py
    class User(Base):
        __tablename__ = "user"

        id: Mapped[int] = mapped_column(primary_key=True)
        name: Mapped[str]
        bio: Mapped[str] = mapped_column(Text, deferred=True)
    ```

    另请参见

    限制哪些列与列延迟加载 - 延迟列加载的完整描述

+   **活动历史** - `mapped_column.active_history` 确保在属性值更改时，先前的值已被加载，并在检查属性历史时成为`AttributeState.history`集合的一部分。这可能会导致额外的 SQL 语句：

    ```py
    class User(Base):
        __tablename__ = "user"

        id: Mapped[int] = mapped_column(primary_key=True)
        important_identifier: Mapped[str] = mapped_column(active_history=True)
    ```

参见`mapped_column()`的文档字符串以获取支持的参数列表。

另请参见

应用于命令式表列的加载、持久化和映射选项 - 描述了使用`column_property()`和`deferred()`与命令式表配置一起使用  ### 明确命名声明式映射列

到目前为止，所有的例子都以 ORM 映射属性链接到`mapped_column()`构造为特色，其中 Python 属性名称赋予了`mapped_column()`，正如我们在 CREATE TABLE 语句和查询中看到的那样。在 SQL 中表示列的名称可以通过将字符串位置参数`mapped_column.__name`传递为第一个位置参数来指示。在下面的示例中，`User`类被映射到了给定列的备用名称：

```py
class User(Base):
    __tablename__ = "user"

    id: Mapped[int] = mapped_column("user_id", primary_key=True)
    name: Mapped[str] = mapped_column("user_name")
```

在上述例子中，`User.id`解析为名为`user_id`的列，而`User.name`解析为名为`user_name`的列。我们可以使用 Python 属性名称编写一个`select()`语句，然后会看到生成的 SQL 名称：

```py
>>> from sqlalchemy import select
>>> print(select(User.id, User.name).where(User.name == "x"))
SELECT  "user".user_id,  "user".user_name
FROM  "user"
WHERE  "user".user_name  =  :user_name_1 
```

另请参见

映射表列的备用属性名称 - 适用于命令式表  ### 向现有声明式映射类追加额外的列

声明式表配置允许在已生成`Table`元数据之后向现有映射添加新的`Column`对象。

对于使用声明基类声明的声明类，底层元类`DeclarativeMeta`包括一个`__setattr__()`方法，将拦截附加的`mapped_column()`或核心`Column`对象，并将它们添加到`Table`使用`Table.append_column()`以及现有的`Mapper`使用`Mapper.add_property()`：

```py
MyClass.some_new_column = mapped_column(String)
```

使用核心`Column`：

```py
MyClass.some_new_column = Column(String)
```

所有参数都受支持，包括替代名称，例如`MyClass.some_new_column = mapped_column("some_name", String)`。然而，SQL 类型必须显式地传递给`mapped_column()`或`Column`对象，就像上面的示例中传递了`String`类型一样。`Mapped`注释类型无法参与操作。

在使用单表继承的特定情况下，还可以向映射添加其他`Column`对象，在此情况下，映射的子类上存在其他列，这些列没有自己的`Table`。这在单表继承部分进行了说明。

另请参阅

在声明后向映射类添加关系 - `relationship()`的类似示例

注意

将映射属性分配给已映射类只有在使用“声明基类”时才能正常运行，这意味着用户定义的`DeclarativeBase`子类或由`declarative_base()`或`registry.generate_base()`返回的动态生成类。这个“基类”包括一个实现特殊`__setattr__()`方法的 Python 元类，用于拦截这些操作。

将类映射属性运行时分配给映射类，如果使用装饰器（`registry.mapped()`）或命令式函数（`registry.map_imperatively()`）来映射类，则**不会**起作用。### 使用注释声明表（`mapped_column()`的类型注释形式）

`mapped_column()`构造能够从声明式映射类中声明的与属性关联的[**PEP 484**](https://peps.python.org/pep-0484/)类型注释中派生其列配置信息。如果使用了这些类型注释，则**必须**存在于一个名为`Mapped`的特殊 SQLAlchemy 类型中，这是一个[泛型](https://peps.python.org/pep-0484/#generics)类型，然后在其中���示一个特定的 Python 类型。

下面说明了前一节的映射，添加了对`Mapped`的使用：

```py
from typing import Optional

from sqlalchemy import String
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "user"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(50))
    fullname: Mapped[Optional[str]]
    nickname: Mapped[Optional[str]] = mapped_column(String(30))
```

在上述情况下，当声明式处理每个类属性时，如果存在的话，每个`mapped_column()`将从左侧对应的`Mapped`类型注释中派生出额外的参数。此外，当遇到没有为属性分配值的`Mapped`类型注释时（这种形式受到 Python [dataclasses](https://docs.python.org/3/library/dataclasses.html)中使用的类似风格的启发），声明式将隐式生成一个空的`mapped_column()`指令；这个`mapped_column()`构造将从存在的`Mapped`注释中派生其配置。

#### `mapped_column()` 从 `Mapped` 注释中派生数据类型和可空性

`mapped_column()` 从 `Mapped` 注释派生的两个特点是：

+   **datatype** - 给定在 `Mapped` 中的 Python 类型，如果存在，则与 `TypeEngine` 的子类相关联，例如 `Integer`、`String`、`DateTime` 或 `Uuid`，等等常见类型。

    数据类型是基于 Python 类型到 SQLAlchemy 数据类型的字典确定的。如下一节 自定义类型映射 中详细说明的那样，该字典是完全可定制的。默认类型映射的实现如下面的代码示例所示：

    ```py
    from typing import Any
    from typing import Dict
    from typing import Type

    import datetime
    import decimal
    import uuid

    from sqlalchemy import types

    # default type mapping, deriving the type for mapped_column()
    # from a Mapped[] annotation
    type_map: Dict[Type[Any], TypeEngine[Any]] = {
        bool: types.Boolean(),
        bytes: types.LargeBinary(),
        datetime.date: types.Date(),
        datetime.datetime: types.DateTime(),
        datetime.time: types.Time(),
        datetime.timedelta: types.Interval(),
        decimal.Decimal: types.Numeric(),
        float: types.Float(),
        int: types.Integer(),
        str: types.String(),
        uuid.UUID: types.Uuid(),
    }
    ```

    如果 `mapped_column()` 构造指示明确的类型，作为传递给 `mapped_column.__type` 参数，则给定的 Python 类型将被忽略。

+   **可空性** - `mapped_column()` 构造将首先通过 `mapped_column.nullable` 参数的存在与否来指示其 `Column` 是 `NULL` 还是 `NOT NULL`，可以传递为 `True` 或 `False`。此外，如果存在 `mapped_column.primary_key` 参数并设置为 `True`，那么这也将意味着该列应该是 `NOT NULL`。

    如果这两个参数都不存在，则在`Mapped`类型注释中存在`typing.Optional[]`将用于确定可为空性，其中`typing.Optional[]`表示`NULL`，而没有`typing.Optional[]`表示`NOT NULL`。如果根本没有`Mapped[]`注释，并且没有`mapped_column.nullable`或`mapped_column.primary_key`参数，则使用 SQLAlchemy 对`Column`的通常默认值`NULL`。

    在下面的示例中，`id`和`data`列将是`NOT NULL`，而`additional_info`列将是`NULL`：

    ```py
    from typing import Optional

    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column

    class Base(DeclarativeBase):
        pass

    class SomeClass(Base):
        __tablename__ = "some_table"

        # primary_key=True, therefore will be NOT NULL
        id: Mapped[int] = mapped_column(primary_key=True)

        # not Optional[], therefore will be NOT NULL
        data: Mapped[str]

        # Optional[], therefore will be NULL
        additional_info: Mapped[Optional[str]]
    ```

    从注释中可以推断出的`mapped_column()`的可为 null 性与注释所暗示的可为 null 性不同是完全有效的。例如，在使用对象进行首次创建和填充的 Python 代码中，ORM 映射的属性可能被注释为允许`None`，但最终该值将被写入到一个`NOT NULL`的数据库列中。当存在时，`mapped_column.nullable`参数将始终优先考虑：

    ```py
    class SomeClass(Base):
        # ...

        # will be String() NOT NULL, but can be None in Python
        data: Mapped[Optional[str]] = mapped_column(nullable=False)
    ```

    类似地，需要在模式级别为某些原因需要为 NULL 的数据库列写入的非 None 属性，可以将`mapped_column.nullable`设置为`True`：

    ```py
    class SomeClass(Base):
        # ...

        # will be String() NULL, but type checker will not expect
        # the attribute to be None
        data: Mapped[str] = mapped_column(nullable=True)
    ```  #### 自定义类型映射

在前一节描述的 Python 类型到 SQLAlchemy `TypeEngine`类型的映射默认为硬编码字典，位于`sqlalchemy.sql.sqltypes`模块中。然而，协调 Declarative 映射过程的`registry`对象将首先查询一个本地的、用户定义的类型字典，该字典可以在构造`registry`时作为`registry.type_annotation_map`参数传递，并且在首次使用时可能与`DeclarativeBase`超类相关联。

作为一个示例，如果我们希望使用`BIGINT`数据类型代表`int`，在`datetime.datetime`上使用带有`timezone=True`的`TIMESTAMP`数据类型，并且仅在 Microsoft SQL Server 上使用`NVARCHAR`数据类型时，Python `str` 被使用，那么注册表和 Declarative base 可以被配置为：

```py
import datetime

from sqlalchemy import BIGINT, Integer, NVARCHAR, String, TIMESTAMP
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped, mapped_column, registry

class Base(DeclarativeBase):
    type_annotation_map = {
        int: BIGINT,
        datetime.datetime: TIMESTAMP(timezone=True),
        str: String().with_variant(NVARCHAR, "mssql"),
    }

class SomeClass(Base):
    __tablename__ = "some_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    date: Mapped[datetime.datetime]
    status: Mapped[str]
```

下面演示了针对上述映射生成的 CREATE TABLE 语句，首先在 Microsoft SQL Server 后端上，说明了 `NVARCHAR` 数据类型：

```py
>>> from sqlalchemy.schema import CreateTable
>>> from sqlalchemy.dialects import mssql, postgresql
>>> print(CreateTable(SomeClass.__table__).compile(dialect=mssql.dialect()))
CREATE  TABLE  some_table  (
  id  BIGINT  NOT  NULL  IDENTITY,
  date  TIMESTAMP  NOT  NULL,
  status  NVARCHAR(max)  NOT  NULL,
  PRIMARY  KEY  (id)
) 
```

接着在 PostgreSQL 后端上，说明 `TIMESTAMP WITH TIME ZONE`：

```py
>>> print(CreateTable(SomeClass.__table__).compile(dialect=postgresql.dialect()))
CREATE  TABLE  some_table  (
  id  BIGSERIAL  NOT  NULL,
  date  TIMESTAMP  WITH  TIME  ZONE  NOT  NULL,
  status  VARCHAR  NOT  NULL,
  PRIMARY  KEY  (id)
) 
```

通过使用`TypeEngine.with_variant()`等方法，我们能够构建一个针对不同后端的定制类型映射，同时仍然能够使用简洁的基于注释的 `mapped_column()` 配置。在此之上还有两个级别的 Python 类型可配置性可用，将在接下来的两个部分中描述。  #### 将多个类型配置映射到 Python 类型

由于个别 Python 类型可以通过使用`registry.type_annotation_map`参数与任何类型的`TypeEngine`配置相关联，因此另一个能力是能够将单个 Python 类型与基于额外类型限定符的 SQL 类型的不同变体相关联。其中一个典型的例子是将 Python `str` 数据类型映射到不同长度的 `VARCHAR` SQL 类型。另一个例子是将不同种类的 `decimal.Decimal` 映射到不同大小的 `NUMERIC` 列。

Python 的类型系统提供了一种很好的方法，可以为 Python 类型添加附加的元数据，即使用[**PEP 593**](https://peps.python.org/pep-0593/) `Annotated` 泛型类型，它允许将附加信息捆绑到 Python 类型上。`mapped_column()` 构造将正确地解释 `Annotated` 对象的身份，当在`registry.type_annotation_map`中解析它时，就像下面的示例中我们声明 `String` 和 `Numeric` 的两个变体一样：

```py
from decimal import Decimal

from typing_extensions import Annotated

from sqlalchemy import Numeric
from sqlalchemy import String
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import registry

str_30 = Annotated[str, 30]
str_50 = Annotated[str, 50]
num_12_4 = Annotated[Decimal, 12]
num_6_2 = Annotated[Decimal, 6]

class Base(DeclarativeBase):
    registry = registry(
        type_annotation_map={
            str_30: String(30),
            str_50: String(50),
            num_12_4: Numeric(12, 4),
            num_6_2: Numeric(6, 2),
        }
    )
```

传递给 `Annotated` 容器的 Python 类型，在上面的示例中是 `str` 和 `Decimal` 类型，仅对于类型工具而言是重要的；就 `mapped_column()` 构造而言，它只需要在 `registry.type_annotation_map` 字典中查找每个类型对象，而不实际查看 `Annotated` 对象的内部，至少在这种特定上下文中是如此。同样，传递给 `Annotated` 的参数除了底层的 Python 类型本身之外也并不重要，只是至少必须存在一个参数才能使 `Annotated` 构造有效。然后我们可以直接在我们的映射中使用这些增强型类型，它们将与更具体的类型构造匹配，就像以下示例中一样：

```py
class SomeClass(Base):
    __tablename__ = "some_table"

    short_name: Mapped[str_30] = mapped_column(primary_key=True)
    long_name: Mapped[str_50]
    num_value: Mapped[num_12_4]
    short_num_value: Mapped[num_6_2]
```

上述映射的 CREATE TABLE 将演示我们配置的不同变体的 `VARCHAR` 和 `NUMERIC`，并且看起来如下：

```py
>>> from sqlalchemy.schema import CreateTable
>>> print(CreateTable(SomeClass.__table__))
CREATE  TABLE  some_table  (
  short_name  VARCHAR(30)  NOT  NULL,
  long_name  VARCHAR(50)  NOT  NULL,
  num_value  NUMERIC(12,  4)  NOT  NULL,
  short_num_value  NUMERIC(6,  2)  NOT  NULL,
  PRIMARY  KEY  (short_name)
) 
```

将 `Annotated` 类型与不同的 SQL 类型进行链接的多样性赋予了我们广泛的灵活性，下一节将演示 `Annotated` 的第二种更加开放的用法。#### 将整个列声明映射到 Python 类型

前一节演示了使用 [**PEP 593**](https://peps.python.org/pep-0593/) `Annotated` 类型实例作为`registry.type_annotation_map` 字典中的键。在这种形式中，`mapped_column()` 构造实际上并不查看 `Annotated` 对象本身，它只被用作字典键。然而，Declarative 还具有直接从 `Annotated` 对象中提取整个预先建立的 `mapped_column()` 构造的能力。使用这种形式，我们不仅可以定义不同种类的 SQL 数据类型与 Python 类型的链接，而且可以以可重用的方式设置任意数量的参数，例如可为空性、列默认值和约束。

一组 ORM 模型通常会有一种对所有映射类都通用的主键样式。还可能有一些常见的列配置，例如带有默认值的时间戳和其他预先设定大小和配置的字段。我们可以将这些配置组合成`mapped_column()`实例，然后直接捆绑到`Annotated`的实例中，然后在任意数量的类声明中重新使用它们。当以这种方式提供时，声明式将解开一个`Annotated`对象，跳过任何不适用于 SQLAlchemy 的其他指令，仅搜索 SQLAlchemy ORM 构造。

下面的示例演示了以这种方式使用的各种预配置字段类型，我们在其中定义了`intpk`表示一个`Integer`主键列，`timestamp`表示一个`DateTime`类型，它将使用`CURRENT_TIMESTAMP`作为 DDL 级别列默认值，并且`required_name`是一个长度为 30 的`String`，`NOT NULL`：

```py
import datetime

from typing_extensions import Annotated

from sqlalchemy import func
from sqlalchemy import String
from sqlalchemy.orm import mapped_column

intpk = Annotated[int, mapped_column(primary_key=True)]
timestamp = Annotated[
    datetime.datetime,
    mapped_column(nullable=False, server_default=func.CURRENT_TIMESTAMP()),
]
required_name = Annotated[str, mapped_column(String(30), nullable=False)]
```

上述的`Annotated`对象然后可以直接在`Mapped`中使用，在那里预先配置的`mapped_column()`构造将被提取并复制到一个新实例中，该实例将针对每个属性具体化：

```py
class Base(DeclarativeBase):
    pass

class SomeClass(Base):
    __tablename__ = "some_table"

    id: Mapped[intpk]
    name: Mapped[required_name]
    created_at: Mapped[timestamp]
```

我们上面映射的`CREATE TABLE`如下所示：

```py
>>> from sqlalchemy.schema import CreateTable
>>> print(CreateTable(SomeClass.__table__))
CREATE  TABLE  some_table  (
  id  INTEGER  NOT  NULL,
  name  VARCHAR(30)  NOT  NULL,
  created_at  DATETIME  DEFAULT  CURRENT_TIMESTAMP  NOT  NULL,
  PRIMARY  KEY  (id)
) 
```

当以这种方式使用`Annotated`类型时，类型的配置也可能会受到每个属性的影响。对于上面示例中显式使用`mapped_column.nullable`的类型，我们可以将`Optional[]`泛型修饰符应用于我们的任何类型，以便该字段在 Python 级别上是可选的或非可选的，这将独立于数据库中发生的`NULL` / `NOT NULL`设置：

```py
from typing_extensions import Annotated

import datetime
from typing import Optional

from sqlalchemy.orm import DeclarativeBase

timestamp = Annotated[
    datetime.datetime,
    mapped_column(nullable=False),
]

class Base(DeclarativeBase):
    pass

class SomeClass(Base):
    # ...

    # pep-484 type will be Optional, but column will be
    # NOT NULL
    created_at: Mapped[Optional[timestamp]]
```

`mapped_column()`构造也与显式传递的`mapped_column()`构造协调，其参数将优先于`Annotated`构造的参数。下面我们向整数主键添加一个`ForeignKey`约束，并为`created_at`列使用另一个替代的服务器默认值：

```py
import datetime

from typing_extensions import Annotated

from sqlalchemy import ForeignKey
from sqlalchemy import func
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.schema import CreateTable

intpk = Annotated[int, mapped_column(primary_key=True)]
timestamp = Annotated[
    datetime.datetime,
    mapped_column(nullable=False, server_default=func.CURRENT_TIMESTAMP()),
]

class Base(DeclarativeBase):
    pass

class Parent(Base):
    __tablename__ = "parent"

    id: Mapped[intpk]

class SomeClass(Base):
    __tablename__ = "some_table"

    # add ForeignKey to mapped_column(Integer, primary_key=True)
    id: Mapped[intpk] = mapped_column(ForeignKey("parent.id"))

    # change server default from CURRENT_TIMESTAMP to UTC_TIMESTAMP
    created_at: Mapped[timestamp] = mapped_column(server_default=func.UTC_TIMESTAMP())
```

`CREATE TABLE`语句说明了这些每个属性的设置，还添加了一个`FOREIGN KEY`约束，并将`UTC_TIMESTAMP`替换为`CURRENT_TIMESTAMP`：

```py
>>> from sqlalchemy.schema import CreateTable
>>> print(CreateTable(SomeClass.__table__))
CREATE  TABLE  some_table  (
  id  INTEGER  NOT  NULL,
  created_at  DATETIME  DEFAULT  UTC_TIMESTAMP()  NOT  NULL,
  PRIMARY  KEY  (id),
  FOREIGN  KEY(id)  REFERENCES  parent  (id)
) 
```

注意

刚刚描述的`mapped_column()`特性，可以使用[**PEP 593**](https://peps.python.org/pep-0593/)中包含一个“模板”`mapped_column()`对象的`Annotated`对象来指示一组完整构造的列参数，这些参数将被复制到属性中，目前还没有实现到其他 ORM 构造中，例如`relationship()`和`composite()`。虽然理论上可以实现这个功能，但当前尝试使用`Annotated`来指示对`relationship()`和类似方法的更多参数将在运行时引发`NotImplementedError`异常，但可能在未来版本中实现。  #### 在类型映射中使用 Python `Enum`或 pep-586 `Literal`类型

在 2.0.0b4 版本中新增：- 添加了`Enum`支持

在 2.0.1 版本中新增：- 添加了`Literal`支持

当在 ORM 声明式映射中使用时，从 Python 内置的`enum.Enum`以及`typing.Literal`类派生的用户定义的 Python 类型将自动链接到 SQLAlchemy 的`Enum`数据类型。下面的示例在`Mapped[]`构造函数中使用了自定义的`enum.Enum`：

```py
import enum

from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column

class Base(DeclarativeBase):
    pass

class Status(enum.Enum):
    PENDING = "pending"
    RECEIVED = "received"
    COMPLETED = "completed"

class SomeClass(Base):
    __tablename__ = "some_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    status: Mapped[Status]
```

在上面的示例中，映射属性`SomeClass.status`将链接到一个具有`Enum(Status)`数据类型的`Column`。我们可以在 PostgreSQL 数据库的 CREATE TABLE 输出中看到这一点：

```py
CREATE  TYPE  status  AS  ENUM  ('PENDING',  'RECEIVED',  'COMPLETED')

CREATE  TABLE  some_table  (
  id  SERIAL  NOT  NULL,
  status  status  NOT  NULL,
  PRIMARY  KEY  (id)
)
```

类似地，可以使用`typing.Literal`，使用一个由所有字符串组成的`typing.Literal`：

```py
from typing import Literal

from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column

class Base(DeclarativeBase):
    pass

Status = Literal["pending", "received", "completed"]

class SomeClass(Base):
    __tablename__ = "some_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    status: Mapped[Status]
```

在`registry.type_annotation_map`中使用的条目将基本的`enum.Enum` Python 类型以及`typing.Literal`类型链接到 SQLAlchemy 的`Enum` SQL 类型，使用一种特殊形式，指示`Enum`数据类型应自动配置自己以适应任意枚举类型。这个默认情况下隐含的配置将明确表示为：

```py
import enum
import typing

import sqlalchemy
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    type_annotation_map = {
        enum.Enum: sqlalchemy.Enum(enum.Enum),
        typing.Literal: sqlalchemy.Enum(enum.Enum),
    }
```

在声明式中的解析逻辑能够解析 `enum.Enum` 的子类以及 `typing.Literal` 的实例，以匹配 `registry.type_annotation_map` 字典中的 `enum.Enum` 或 `typing.Literal` 条目。然后，`Enum` SQL 类型知道如何生成一个带有适当设置的配置版本，包括默认字符串长度。如果传递的 `typing.Literal` 不仅包含字符串值，则会引发一个信息性错误。

##### 本地枚举和命名

`Enum.native_enum` 参数是指 `Enum` 数据类型是否应创建所谓的“本地”枚举，在 MySQL/MariaDB 上是 `ENUM` 数据类型，在 PostgreSQL 上是由 `CREATE TYPE` 创建的新 `TYPE` 对象，或者是“非本地”枚举，这意味着将使用 `VARCHAR` 来创建数据类型。对于除 MySQL/MariaDB 或 PostgreSQL 外的后端，无论何种情况都使用 `VARCHAR`（第三方方言可能有其自己的行为）。

因为 PostgreSQL 的 `CREATE TYPE` 要求为要创建的类型有一个显式的名称，所以在处理隐式生成的 `Enum` 而没有在映射中指定显式的 `Enum` 数据类型时，存在特殊的回退逻辑：

1.  如果 `Enum` 被链接到一个 `enum.Enum` 对象，那么 `Enum.native_enum` 参数默认为 `True`，并且枚举的名称将从 `enum.Enum` 数据类型的名称中获取。在 PostgreSQL 后端，将假定使用此名称创建 `CREATE TYPE`。

1.  如果 `Enum` 被链接到一个 `typing.Literal` 对象，则 `Enum.native_enum` 参数默认为 `False`；不生成名称，并假定为 `VARCHAR`。

要在 PostgreSQL `CREATE TYPE` 类型中使用 `typing.Literal`，必须使用显式的 `Enum`，可以在类型映射中使用：

```py
import enum
import typing

import sqlalchemy
from sqlalchemy.orm import DeclarativeBase

Status = Literal["pending", "received", "completed"]

class Base(DeclarativeBase):
    type_annotation_map = {
        Status: sqlalchemy.Enum("pending", "received", "completed", name="status_enum"),
    }
```

或者也可以在 `mapped_column()` 中使用：

```py
import enum
import typing

import sqlalchemy
from sqlalchemy.orm import DeclarativeBase

Status = Literal["pending", "received", "completed"]

class Base(DeclarativeBase):
    pass

class SomeClass(Base):
    __tablename__ = "some_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    status: Mapped[Status] = mapped_column(
        sqlalchemy.Enum("pending", "received", "completed", name="status_enum")
    )
```

##### 更改默认枚举的配置

要修改隐式生成的`Enum`数据类型的固定配置，需在`registry.type_annotation_map`中指定新条目，表示额外的参数。例如，要无条件使用“非本地枚举”，可以为所有类型设置`Enum.native_enum`参数为 False：

```py
import enum
import typing
import sqlalchemy
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    type_annotation_map = {
        enum.Enum: sqlalchemy.Enum(enum.Enum, native_enum=False),
        typing.Literal: sqlalchemy.Enum(enum.Enum, native_enum=False),
    }
```

在 2.0.1 版本中更改：实现了在建立`registry.type_annotation_map`时覆盖参数（如`Enum.native_enum`）的支持。先前，此功能不起作用。

要为特定的`enum.Enum`子类型使用特定配置，例如在使用示例`Status`数据类型时将字符串长度设置为 50：

```py
import enum
import sqlalchemy
from sqlalchemy.orm import DeclarativeBase

class Status(enum.Enum):
    PENDING = "pending"
    RECEIVED = "received"
    COMPLETED = "completed"

class Base(DeclarativeBase):
    type_annotation_map = {
        Status: sqlalchemy.Enum(Status, length=50, native_enum=False)
    }
```

默认情况下，自动生成的`Enum`不与`Base`使用的`MetaData`实例关联，因此如果元数据定义了模式，它将不会自动与枚举关联。要自动将枚举与元数据中的模式或表关联起来，可以设置`Enum.inherit_schema`：

```py
from enum import Enum
import sqlalchemy as sa
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    metadata = sa.MetaData(schema="my_schema")
    type_annotation_map = {Enum: sa.Enum(Enum, inherit_schema=True)}
```

##### 将特定的`enum.Enum`或`typing.Literal`链接到其他数据类型

上述示例展示了一个自动配置自身到`enum.Enum`或`typing.Literal`类型对象上存在的参数/属性的`Enum`的使用。对于特定种类的`enum.Enum`或`typing.Literal`应链接到其他类型的用例，这些特定类型也可以放置在类型映射中。在下面的示例中，一个包含非字符串类型的`Literal[]`条目链接到`JSON`数据类型：

```py
from typing import Literal

from sqlalchemy import JSON
from sqlalchemy.orm import DeclarativeBase

my_literal = Literal[0, 1, True, False, "true", "false"]

class Base(DeclarativeBase):
    type_annotation_map = {my_literal: JSON}
```

在上述配置中，`my_literal`数据类型将解析为一个`JSON`实例。其他`Literal`变体将继续解析为`Enum`数据类型。

#### `mapped_column()`中的数据类特性

`mapped_column()` 构造与 SQLAlchemy 的“原生数据类”功能集成，详见 声明性数据类映射。请参阅该部分了解 `mapped_column()` 支持的额外指令的当前背景。

#### `mapped_column()` 从 `Mapped` 注释中派生数据类型和可空性

`mapped_column()` 从 `Mapped` 注释中派生的两个特性是：

+   **数据类型** - 在 `Mapped` 中给出的 Python 类型，如果存在，则与 `TypeEngine` 的子类关联，例如 `Integer`、`String`、`DateTime` 或 `Uuid` 等常见类型。

    数据类型是基于 Python 类型到 SQLAlchemy 数据类型的字典确定的。这个字典是完全可定制的，如下一节 自定义类型映射 中所述。默认的类型映射实现如下面的代码示例所示：

    ```py
    from typing import Any
    from typing import Dict
    from typing import Type

    import datetime
    import decimal
    import uuid

    from sqlalchemy import types

    # default type mapping, deriving the type for mapped_column()
    # from a Mapped[] annotation
    type_map: Dict[Type[Any], TypeEngine[Any]] = {
        bool: types.Boolean(),
        bytes: types.LargeBinary(),
        datetime.date: types.Date(),
        datetime.datetime: types.DateTime(),
        datetime.time: types.Time(),
        datetime.timedelta: types.Interval(),
        decimal.Decimal: types.Numeric(),
        float: types.Float(),
        int: types.Integer(),
        str: types.String(),
        uuid.UUID: types.Uuid(),
    }
    ```

    如果 `mapped_column()` 构造指示明确的类型，如传递给 `mapped_column.__type` 参数，则给定的 Python 类型将被忽略。

+   **可空性** - `mapped_column()` 构造将通过 `mapped_column.nullable` 参数的存在来首先指示其 `Column` 为 `NULL` 或 `NOT NULL`，该参数传递为 `True` 或 `False`。此外，如果 `mapped_column.primary_key` 参数存在并设置为 `True`，那么也会暗示该列应该是 `NOT NULL`。

    在**这两个**参数都不存在的情况下，`Mapped`类型注释中的`typing.Optional[]`的存在将用于确定空值性，其中`typing.Optional[]`表示`NULL`，而`typing.Optional[]`的缺失表示`NOT NULL`。如果根本没有`Mapped[]`注释存在，并且没有`mapped_column.nullable`或`mapped_column.primary_key`参数，则 SQLAlchemy 对于`Column`的通常默认值为`NULL`。

    在下面的示例中，`id`和`data`列将是`NOT NULL`，而`additional_info`列将是`NULL`：

    ```py
    from typing import Optional

    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column

    class Base(DeclarativeBase):
        pass

    class SomeClass(Base):
        __tablename__ = "some_table"

        # primary_key=True, therefore will be NOT NULL
        id: Mapped[int] = mapped_column(primary_key=True)

        # not Optional[], therefore will be NOT NULL
        data: Mapped[str]

        # Optional[], therefore will be NULL
        additional_info: Mapped[Optional[str]]
    ```

    具有`mapped_column()`的空值属性与注释所暗示的**不同**是完全有效的。例如，一个 ORM 映射的属性可能在 Python 代码中被注释为允许`None`，这段代码在对象首次创建和填充时使用，然而最终该值将被写入一个`NOT NULL`的数据库列。当存在`mapped_column.nullable`参数时，该参数将始终优先考虑：

    ```py
    class SomeClass(Base):
        # ...

        # will be String() NOT NULL, but can be None in Python
        data: Mapped[Optional[str]] = mapped_column(nullable=False)
    ```

    同样，一个非空属性写入到一个数据库列，由于某种原因需要在模式级别为 NULL，`mapped_column.nullable`可以设置为`True`：

    ```py
    class SomeClass(Base):
        # ...

        # will be String() NULL, but type checker will not expect
        # the attribute to be None
        data: Mapped[str] = mapped_column(nullable=True)
    ```

#### 自定义类型映射

Python 类型到 SQLAlchemy `TypeEngine`类型的映射在前一节中描述的默认为`sqlalchemy.sql.sqltypes`模块中的硬编码字典。然而，协调 Declarative 映射过程的`registry`对象将首先查阅一个本地的、用户定义的类型字典，该字典可以在构建`registry`时作为`registry.type_annotation_map`参数传递，当首次使用时可能与`DeclarativeBase`超类相关联。

举例来说，如果我们希望使用`BIGINT`数据类型来表示`int`，使用带有`timezone=True`的`TIMESTAMP`数据类型来表示`datetime.datetime`，然后仅在 Microsoft SQL Server 上使用`NVARCHAR`数据类型来表示 Python 的`str`，则可以配置注册表和 Declarative 基类如下：

```py
import datetime

from sqlalchemy import BIGINT, Integer, NVARCHAR, String, TIMESTAMP
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped, mapped_column, registry

class Base(DeclarativeBase):
    type_annotation_map = {
        int: BIGINT,
        datetime.datetime: TIMESTAMP(timezone=True),
        str: String().with_variant(NVARCHAR, "mssql"),
    }

class SomeClass(Base):
    __tablename__ = "some_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    date: Mapped[datetime.datetime]
    status: Mapped[str]
```

下面展示了为上述映射生成的 CREATE TABLE 语句，在 Microsoft SQL Server 后端首先展示了`NVARCHAR`数据类型：

```py
>>> from sqlalchemy.schema import CreateTable
>>> from sqlalchemy.dialects import mssql, postgresql
>>> print(CreateTable(SomeClass.__table__).compile(dialect=mssql.dialect()))
CREATE  TABLE  some_table  (
  id  BIGINT  NOT  NULL  IDENTITY,
  date  TIMESTAMP  NOT  NULL,
  status  NVARCHAR(max)  NOT  NULL,
  PRIMARY  KEY  (id)
) 
```

然后在 PostgreSQL 后端，展示了`TIMESTAMP WITH TIME ZONE`：

```py
>>> print(CreateTable(SomeClass.__table__).compile(dialect=postgresql.dialect()))
CREATE  TABLE  some_table  (
  id  BIGSERIAL  NOT  NULL,
  date  TIMESTAMP  WITH  TIME  ZONE  NOT  NULL,
  status  VARCHAR  NOT  NULL,
  PRIMARY  KEY  (id)
) 
```

通过使用诸如`TypeEngine.with_variant()`之类的方法，我们能够构建一个针对不同后端定制的类型映射，同时仍然能够使用简洁的仅注释的`mapped_column()`配置。在此之上还有两个级别的 Python 类型可配置性，将在接下来的两个部分中描述。

#### 将多种类型配置映射到 Python 类型

由于个别 Python 类型可以通过使用`TypeEngine`配置的任何类型与`registry.type_annotation_map`参数相关联，另一个功能是能够将单个 Python 类型与基于附加类型限定符的 SQL 类型的不同变体关联起来。一个典型的例子是将 Python 的`str`数据类型映射到不同长度的`VARCHAR` SQL 类型。另一个是将不同种类的`decimal.Decimal`映射到不同大小的`NUMERIC`列。

Python 的类型系统提供了一种很好的方式来为 Python 类型添加额外的元数据，即使用[**PEP 593**](https://peps.python.org/pep-0593/) `Annotated`泛型类型，它允许将附加信息与 Python 类型捆绑在一起。`mapped_column()`构造将正确解释`Annotated`对象的身份，当在`registry.type_annotation_map`中解析它时，就像下面的示例中声明两个`String`和`Numeric`变体一样：

```py
from decimal import Decimal

from typing_extensions import Annotated

from sqlalchemy import Numeric
from sqlalchemy import String
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import registry

str_30 = Annotated[str, 30]
str_50 = Annotated[str, 50]
num_12_4 = Annotated[Decimal, 12]
num_6_2 = Annotated[Decimal, 6]

class Base(DeclarativeBase):
    registry = registry(
        type_annotation_map={
            str_30: String(30),
            str_50: String(50),
            num_12_4: Numeric(12, 4),
            num_6_2: Numeric(6, 2),
        }
    )
```

在上述示例中传递给`Annotated`容器的 Python 类型，例如`str`和`Decimal`类型，仅对于类型工具的好处是重要的；就`mapped_column()`构造而言，在这个特定的上下文中，它只需要在`registry.type_annotation_map`字典中查找每个类型对象，而不实际查看`Annotated`对象的内部。类似地，传递给`Annotated`的参数超出底层 Python 类型本身也不重要，只是必须至少存在一个参数，以使`Annotated`构造有效。然后，我们可以直接在我们的映射中使用这些增强类型，它们将与更具体的类型构造相匹配，就像以下示例中一样：

```py
class SomeClass(Base):
    __tablename__ = "some_table"

    short_name: Mapped[str_30] = mapped_column(primary_key=True)
    long_name: Mapped[str_50]
    num_value: Mapped[num_12_4]
    short_num_value: Mapped[num_6_2]
```

上述映射的 CREATE TABLE 将说明我们配置的不同变体的`VARCHAR`和`NUMERIC`，并且看起来像是：

```py
>>> from sqlalchemy.schema import CreateTable
>>> print(CreateTable(SomeClass.__table__))
CREATE  TABLE  some_table  (
  short_name  VARCHAR(30)  NOT  NULL,
  long_name  VARCHAR(50)  NOT  NULL,
  num_value  NUMERIC(12,  4)  NOT  NULL,
  short_num_value  NUMERIC(6,  2)  NOT  NULL,
  PRIMARY  KEY  (short_name)
) 
```

虽然将`Annotated`类型与不同的 SQL 类型链接的多样性为我们提供了广泛的灵活性，但下一节说明了一种使用`Annotated`与 Declarative 结合使用的第二种更加开放的方式。

#### 将整个列声明映射到 Python 类型

前面的章节说明了使用[**PEP 593**](https://peps.python.org/pep-0593/) `Annotated`类型实例作为`registry.type_annotation_map`字典中的键。在这种形式中，`mapped_column()`构造实际上并不查看`Annotated`对象本身，而是仅用作字典键。然而，Declarative 还具有直接从`Annotated`对象中提取整个预先建立的`mapped_column()`构造的能力。使用这种形式，我们不仅可以定义与 Python 类型相关联的不同种类的 SQL 数据类型，而且还可以以可重用的方式设置任意数量的参数，例如可为空性、列默认值和约束。

一组 ORM 模型通常会有一种对所有映射类都通用的主键样式。还可能有常见的列配置，例如具有默认值的时间戳和其他预先确定大小和配置的字段。我们可以将这些配置组合成`mapped_column()`实例，然后直接捆绑到`Annotated`的实例中，然后在任意数量的类声明中重复使用。当以这种方式提供时，声明性将解开`Annotated`对象，跳过任何不适用于 SQLAlchemy 的其他指令，并仅搜索 SQLAlchemy ORM 构造。

下面的示例说明了以这种方式使用的各种预配置字段类型，其中我们定义了`intpk`代表一个`整型`主键列，`timestamp`代表一个`日期时间`类型，它将使用`CURRENT_TIMESTAMP`作为 DDL 级别的列默认值，并且`required_name`是一个长度为 30 的`字符串`，它是`NOT NULL`的：

```py
import datetime

from typing_extensions import Annotated

from sqlalchemy import func
from sqlalchemy import String
from sqlalchemy.orm import mapped_column

intpk = Annotated[int, mapped_column(primary_key=True)]
timestamp = Annotated[
    datetime.datetime,
    mapped_column(nullable=False, server_default=func.CURRENT_TIMESTAMP()),
]
required_name = Annotated[str, mapped_column(String(30), nullable=False)]
```

上述的`Annotated`对象然后可以直接在`Mapped`中使用，预配置的`mapped_column()`构造将被提取并复制到一个新实例中，该实例将针对每个属性具体：

```py
class Base(DeclarativeBase):
    pass

class SomeClass(Base):
    __tablename__ = "some_table"

    id: Mapped[intpk]
    name: Mapped[required_name]
    created_at: Mapped[timestamp]
```

我们上面的映射的`CREATE TABLE`如下所示：

```py
>>> from sqlalchemy.schema import CreateTable
>>> print(CreateTable(SomeClass.__table__))
CREATE  TABLE  some_table  (
  id  INTEGER  NOT  NULL,
  name  VARCHAR(30)  NOT  NULL,
  created_at  DATETIME  DEFAULT  CURRENT_TIMESTAMP  NOT  NULL,
  PRIMARY  KEY  (id)
) 
```

当以这种方式使用`Annotated`类型时，类型的配置也可能会受到每个属性基础的影响。对于上面示例中显式使用了`mapped_column.nullable`的类型，我们可以对我们的任何类型应用`Optional[]`泛型修饰符，以便在 Python 级别该字段是可选的还是不可选的，这将独立于数据库中发生的`NULL` / `NOT NULL`设置：

```py
from typing_extensions import Annotated

import datetime
from typing import Optional

from sqlalchemy.orm import DeclarativeBase

timestamp = Annotated[
    datetime.datetime,
    mapped_column(nullable=False),
]

class Base(DeclarativeBase):
    pass

class SomeClass(Base):
    # ...

    # pep-484 type will be Optional, but column will be
    # NOT NULL
    created_at: Mapped[Optional[timestamp]]
```

`mapped_column()`构造还与显式传递的`mapped_column()`构造协调一致，其参数将优先于`Annotated`构造的参数。下面我们向我们的整型主键添加了一个`外键`约束，并且还为`created_at`列使用了备用服务器默认值：

```py
import datetime

from typing_extensions import Annotated

from sqlalchemy import ForeignKey
from sqlalchemy import func
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.schema import CreateTable

intpk = Annotated[int, mapped_column(primary_key=True)]
timestamp = Annotated[
    datetime.datetime,
    mapped_column(nullable=False, server_default=func.CURRENT_TIMESTAMP()),
]

class Base(DeclarativeBase):
    pass

class Parent(Base):
    __tablename__ = "parent"

    id: Mapped[intpk]

class SomeClass(Base):
    __tablename__ = "some_table"

    # add ForeignKey to mapped_column(Integer, primary_key=True)
    id: Mapped[intpk] = mapped_column(ForeignKey("parent.id"))

    # change server default from CURRENT_TIMESTAMP to UTC_TIMESTAMP
    created_at: Mapped[timestamp] = mapped_column(server_default=func.UTC_TIMESTAMP())
```

`CREATE TABLE`语句说明了这些每个属性的设置，添加了`FOREIGN KEY`约束，并且将`UTC_TIMESTAMP`替换为`CURRENT_TIMESTAMP`：

```py
>>> from sqlalchemy.schema import CreateTable
>>> print(CreateTable(SomeClass.__table__))
CREATE  TABLE  some_table  (
  id  INTEGER  NOT  NULL,
  created_at  DATETIME  DEFAULT  UTC_TIMESTAMP()  NOT  NULL,
  PRIMARY  KEY  (id),
  FOREIGN  KEY(id)  REFERENCES  parent  (id)
) 
```

注意

刚刚描述的 `mapped_column()` 特性，其中可以使用 [**PEP 593**](https://peps.python.org/pep-0593/) `Annotated` 对象指示一组完全构造的列参数，这些列参数包含一个“模板” `mapped_column()` 对象，将被复制到属性中，目前尚未实现为其他 ORM 构造（如 `relationship()` 和 `composite()`）。虽然理论上可能存在这种功能，但目前尝试使用 `Annotated` 来指示 `relationship()` 等的进一步参数将在运行时引发 `NotImplementedError` 异常，但可能会在将来的版本中实现。

#### 在类型映射中使用 Python `Enum` 或 pep-586 `Literal` 类型

新版本 2.0.0b4 中新增：- 添加了 `Enum` 支持

新版本 2.0.1 中新增：- 添加了 `Literal` 支持

当在 ORM 声明性映射中使用用户定义的 Python 类型时，这些类型派生自 Python 内置的 `enum.Enum` 类以及 `typing.Literal` 类时，它们会自动链接到 SQLAlchemy `Enum` 数据类型。下面的示例在 `Mapped[]` 构造函数中使用了自定义的 `enum.Enum`：

```py
import enum

from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column

class Base(DeclarativeBase):
    pass

class Status(enum.Enum):
    PENDING = "pending"
    RECEIVED = "received"
    COMPLETED = "completed"

class SomeClass(Base):
    __tablename__ = "some_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    status: Mapped[Status]
```

在上面的示例中，映射的属性 `SomeClass.status` 将链接到一个 `Column`，其数据类型为 `Enum(Status)`。我们可以在 PostgreSQL 数据库的 CREATE TABLE 输出中看到这一点：

```py
CREATE  TYPE  status  AS  ENUM  ('PENDING',  'RECEIVED',  'COMPLETED')

CREATE  TABLE  some_table  (
  id  SERIAL  NOT  NULL,
  status  status  NOT  NULL,
  PRIMARY  KEY  (id)
)
```

类似地，也可以使用 `typing.Literal`，使用包含所有字符串的 `typing.Literal`：

```py
from typing import Literal

from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column

class Base(DeclarativeBase):
    pass

Status = Literal["pending", "received", "completed"]

class SomeClass(Base):
    __tablename__ = "some_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    status: Mapped[Status]
```

在 `registry.type_annotation_map` 中使用的条目将基本的 `enum.Enum` Python 类型以及 `typing.Literal` 类型链接到 SQLAlchemy `Enum` SQL 类型，使用特殊形式来指示 `Enum` 数据类型应自动配置自己以针对任意枚举类型。默认情况下，此配置是隐式的，但可以显式指示为：

```py
import enum
import typing

import sqlalchemy
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    type_annotation_map = {
        enum.Enum: sqlalchemy.Enum(enum.Enum),
        typing.Literal: sqlalchemy.Enum(enum.Enum),
    }
```

在 Declarative 内部的解析逻辑能够解析`enum.Enum`的子类以及`typing.Literal`的实例，以匹配`registry.type_annotation_map`字典中的`enum.Enum`或`typing.Literal`条目。然后，`Enum` SQL 类型知道如何生成具有适当设置的已配置版本，包括默认字符串长度。如果传递的`typing.Literal`不仅由字符串值组成，则会引发信息性错误。

##### 本地枚举和命名

`Enum.native_enum`参数是指`Enum`数据类型是否应创建所谓的“本地”枚举，在 MySQL/MariaDB 上是`ENUM`数据类型，在 PostgreSQL 上是通过`CREATE TYPE`创建的新`TYPE`对象，或者是“非本地”枚举，这意味着将使用`VARCHAR`创建数据类型。对于 MySQL/MariaDB 或 PostgreSQL 以外的后端，在所有情况下都使用`VARCHAR`（第三方方言可能具有自己的行为）。

因为 PostgreSQL 的`CREATE TYPE`要求为要创建的类型指定显式名称，所以在处理未显式指定显式`Enum`数据类型的情况下，特殊的后备逻辑存在于隐式生成的`Enum`时：

1.  如果`Enum`链接到一个`enum.Enum`对象，则`Enum.native_enum`参数默认为`True`，并且枚举的名称将取自`enum.Enum`数据类型的名称。 PostgreSQL 后端将假定使用此名称创建`CREATE TYPE`。

1.  如果`Enum`链接到一个`typing.Literal`对象，则`Enum.native_enum`参数默认为`False`；不会生成名称，并且假定为`VARCHAR`。

要在 PostgreSQL 的`CREATE TYPE`类型中使用`typing.Literal`，必须使用显式的`Enum`，要么在类型映射中：

```py
import enum
import typing

import sqlalchemy
from sqlalchemy.orm import DeclarativeBase

Status = Literal["pending", "received", "completed"]

class Base(DeclarativeBase):
    type_annotation_map = {
        Status: sqlalchemy.Enum("pending", "received", "completed", name="status_enum"),
    }
```

或者在 `mapped_column()` 内部：

```py
import enum
import typing

import sqlalchemy
from sqlalchemy.orm import DeclarativeBase

Status = Literal["pending", "received", "completed"]

class Base(DeclarativeBase):
    pass

class SomeClass(Base):
    __tablename__ = "some_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    status: Mapped[Status] = mapped_column(
        sqlalchemy.Enum("pending", "received", "completed", name="status_enum")
    )
```

##### 修改默认枚举的配置

为了修改隐式生成的`Enum`数据类型的固定配置，指定在`registry.type_annotation_map`中添加新条目，表示额外参数。例如，要无条件使用“非原生枚举”，可以为所有类型设置`Enum.native_enum`参数为 False：

```py
import enum
import typing
import sqlalchemy
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    type_annotation_map = {
        enum.Enum: sqlalchemy.Enum(enum.Enum, native_enum=False),
        typing.Literal: sqlalchemy.Enum(enum.Enum, native_enum=False),
    }
```

从 2.0.1 版本开始更改：实现了在建立`registry.type_annotation_map`时覆盖参数的支持，例如`Enum.native_enum`参数。以前，此功能未能正常工作。

要为特定的`enum.Enum`子类型使用特定的配置，例如在使用示例`Status`数据类型时将字符串长度设置为 50：

```py
import enum
import sqlalchemy
from sqlalchemy.orm import DeclarativeBase

class Status(enum.Enum):
    PENDING = "pending"
    RECEIVED = "received"
    COMPLETED = "completed"

class Base(DeclarativeBase):
    type_annotation_map = {
        Status: sqlalchemy.Enum(Status, length=50, native_enum=False)
    }
```

默认情况下，自动生成的`Enum`不与由`Base`使用的`MetaData`实例关联，因此，如果元数据定义了模式，它将不会自动与枚举关联。要将枚举自动与元数据或表中的模式关联起来，可以设置`Enum.inherit_schema`：

```py
from enum import Enum
import sqlalchemy as sa
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    metadata = sa.MetaData(schema="my_schema")
    type_annotation_map = {Enum: sa.Enum(Enum, inherit_schema=True)}
```

##### 将特定的`enum.Enum`或`typing.Literal`链接到其他数据类型

以上示例展示了一个`Enum`自动配置自身到一个`enum.Enum`或`typing.Literal`类型对象上存在的参数/属性的情况。对于应用场景，特定类型的`enum.Enum`或`typing.Literal`应链接到其他类型的情况，这些特定类型也可以放置在类型映射中。在下面的示例中，一个包含非字符串类型的`Literal[]`条目被链接到`JSON`数据类型：

```py
from typing import Literal

from sqlalchemy import JSON
from sqlalchemy.orm import DeclarativeBase

my_literal = Literal[0, 1, True, False, "true", "false"]

class Base(DeclarativeBase):
    type_annotation_map = {my_literal: JSON}
```

在上述配置中，`my_literal`数据类型将解析为一个`JSON`实例。其他`Literal`变体将继续解析为`Enum`数据类型。

##### 原生枚举和命名

`Enum.native_enum` 参数指的是 `Enum` 数据类型是否应该创建所谓的“本地”枚举，在 MySQL/MariaDB 中是 `ENUM` 数据类型，在 PostgreSQL 中是由 `CREATE TYPE` 创建的新 `TYPE` 对象，或者是“非本地”枚举，这意味着将使用 `VARCHAR` 来创建数据类型。对于不是 MySQL/MariaDB 或 PostgreSQL 的后端，`VARCHAR` 在所有情况下都会被使用（第三方方言可能具有自己的行为）。

因为 PostgreSQL 的 `CREATE TYPE` 要求有一个明确的类型名称要被创建，所以在使用隐式生成的 `Enum` 时，当没有指定显式的 `Enum` 数据类型时，存在特殊的回退逻辑：

1.  如果 `Enum` 与 `enum.Enum` 对象关联，则 `Enum.native_enum` 参数默认为 `True`，并且枚举的名称将从 `enum.Enum` 数据类型的名称中获取。PostgreSQL 后端将假定使用此名称创建 `CREATE TYPE`。

1.  如果 `Enum` 与 `typing.Literal` 对象关联，则 `Enum.native_enum` 参数默认为 `False`；不会生成名称，假定为 `VARCHAR`。

要在 PostgreSQL 的 `CREATE TYPE` 类型中使用 `typing.Literal`，必须使用显式的 `Enum`，要么在类型映射中：

```py
import enum
import typing

import sqlalchemy
from sqlalchemy.orm import DeclarativeBase

Status = Literal["pending", "received", "completed"]

class Base(DeclarativeBase):
    type_annotation_map = {
        Status: sqlalchemy.Enum("pending", "received", "completed", name="status_enum"),
    }
```

或者在 `mapped_column()` 中：

```py
import enum
import typing

import sqlalchemy
from sqlalchemy.orm import DeclarativeBase

Status = Literal["pending", "received", "completed"]

class Base(DeclarativeBase):
    pass

class SomeClass(Base):
    __tablename__ = "some_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    status: Mapped[Status] = mapped_column(
        sqlalchemy.Enum("pending", "received", "completed", name="status_enum")
    )
```

##### 修改默认枚举的配置

为了修改隐式生成的 `Enum` 数据类型的固定配置，可以在 `registry.type_annotation_map` 中指定新的条目，指示额外的参数。例如，要无条件地使用“非本地枚举”，可以为所有类型将 `Enum.native_enum` 参数设置为 False：

```py
import enum
import typing
import sqlalchemy
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    type_annotation_map = {
        enum.Enum: sqlalchemy.Enum(enum.Enum, native_enum=False),
        typing.Literal: sqlalchemy.Enum(enum.Enum, native_enum=False),
    }
```

在 2.0.1 版本中更改：实现了在建立 `registry.type_annotation_map` 时重写参数的支持，例如 `Enum.native_enum` 中的参数。之前，此功能未正常工作。

若要针对特定的 `enum.Enum` 子类型使用特定配置，例如在使用示例 `Status` 数据类型时将字符串长度设置为 50：

```py
import enum
import sqlalchemy
from sqlalchemy.orm import DeclarativeBase

class Status(enum.Enum):
    PENDING = "pending"
    RECEIVED = "received"
    COMPLETED = "completed"

class Base(DeclarativeBase):
    type_annotation_map = {
        Status: sqlalchemy.Enum(Status, length=50, native_enum=False)
    }
```

默认情况下，自动生成的 `Enum` 与 `Base` 使用的 `MetaData` 实例不关联，因此如果元数据定义了模式，则不会自动将其与枚举关联起来。要将枚举自动关联到元数据或表中的模式，可以设置 `Enum.inherit_schema`：

```py
from enum import Enum
import sqlalchemy as sa
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    metadata = sa.MetaData(schema="my_schema")
    type_annotation_map = {Enum: sa.Enum(Enum, inherit_schema=True)}
```

##### 将特定的 `enum.Enum` 或 `typing.Literal` 链接到其他数据类型

上述示例展示了自动将 `Enum` 配置到 `enum.Enum` 或 `typing.Literal` 类型对象上的用法。对于应该与其他类型链接的特定种类的 `enum.Enum` 或 `typing.Literal` 的用例，也可以将这些特定类型放入类型映射中。在下面的示例中，包含非字符串类型的 `Literal[]` 条目被链接到 `JSON` 数据类型：

```py
from typing import Literal

from sqlalchemy import JSON
from sqlalchemy.orm import DeclarativeBase

my_literal = Literal[0, 1, True, False, "true", "false"]

class Base(DeclarativeBase):
    type_annotation_map = {my_literal: JSON}
```

在上述配置中，`my_literal` 数据类型将解析为 `JSON` 实例。其他 `Literal` 变体将继续解析为 `Enum` 数据类型。

#### `mapped_column()` 中的数据类特性

`mapped_column()` 构造与 SQLAlchemy 的“原生数据类”功能集成，该功能在声明式数据类映射中讨论。请参阅该部分了解关于 `mapped_column()` 支持的其他指令的当前背景。

### 访问表和元数据

声明式映射的类将始终包括一个名为 `__table__` 的属性；当使用 `__tablename__` 进行上述配置时，声明过程通过 `__table__` 属性使 `Table` 可用：

```py
# access the Table
user_table = User.__table__
```

上述表最终与`Mapper.local_table`属性相同，我们可以通过运行时检查系统来查看它：

```py
from sqlalchemy import inspect

user_table = inspect(User).local_table
```

与声明式`registry`以及基类关联的`MetaData`集合通常是必要的，以便执行诸如 CREATE 之类的 DDL 操作，以及与诸如 Alembic 之类的迁移工具一起使用。该对象可通过`registry`以及声明式基类的`.metadata`属性获得。下面，对于一个小脚本，我们可能希望针对 SQLite 数据库发出所有表的 CREATE：

```py
engine = create_engine("sqlite://")

Base.metadata.create_all(engine)
```

### 声明式表配置

在使用具有`__tablename__`声明类属性的声明式表配置时，应该使用`__table_args__`声明类属性提供额外的参数供`Table`构造函数使用。

此属性同时适用于通常发送到`Table`构造函数的位置参数和关键字参数。该属性可以以两种形式之一指定。一种是作为字典：

```py
class MyClass(Base):
    __tablename__ = "sometable"
    __table_args__ = {"mysql_engine": "InnoDB"}
```

另一种是元组，其中每个参数都是位置参数（通常是约束）：

```py
class MyClass(Base):
    __tablename__ = "sometable"
    __table_args__ = (
        ForeignKeyConstraint(["id"], ["remote_table.id"]),
        UniqueConstraint("foo"),
    )
```

关键字参数可以通过指定最后一个参数为字典的形式来指定：

```py
class MyClass(Base):
    __tablename__ = "sometable"
    __table_args__ = (
        ForeignKeyConstraint(["id"], ["remote_table.id"]),
        UniqueConstraint("foo"),
        {"autoload": True},
    )
```

类还可以使用`declared_attr()`方法装饰器以动态方式指定`__table_args__`声明属性以及`__tablename__`属性。有关背景信息，请参阅使用混合组合映射层次结构。

### 使用声明式表的显式模式名称

如指定模式名称文档化的`Table`的模式名称应用于单个`Table`，使用`Table.schema`参数。在使用声明式表时，此选项像任何其他选项一样传递到`__table_args__`字典中：

```py
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    pass

class MyClass(Base):
    __tablename__ = "sometable"
    __table_args__ = {"schema": "some_schema"}
```

模式名称也可以通过在指定 MetaData 的默认模式名称文档中记录的`MetaData.schema`参数全局应用于所有`Table`对象。`MetaData`对象可以单独构造，并通过直接赋值给`metadata`属性与`DeclarativeBase`子类关联：

```py
from sqlalchemy import MetaData
from sqlalchemy.orm import DeclarativeBase

metadata_obj = MetaData(schema="some_schema")

class Base(DeclarativeBase):
    metadata = metadata_obj

class MyClass(Base):
    # will use "some_schema" by default
    __tablename__ = "sometable"
```

另请参阅

指定模式名称 - 在使用 MetaData 描述数据库文档中。

### 为声明式映射列设置加载和持久化选项

`mapped_column()`构造接受影响生成的`Column`映射的额外 ORM 特定参数，影响其加载和持久化行为。常用选项包括：

+   **延迟列加载** - `mapped_column.deferred`布尔值默认使用延迟列加载来建立`Column`。在下面的示例中，`User.bio`列默认不会被加载，只有在访问时才会加载：

    ```py
    class User(Base):
        __tablename__ = "user"

        id: Mapped[int] = mapped_column(primary_key=True)
        name: Mapped[str]
        bio: Mapped[str] = mapped_column(Text, deferred=True)
    ```

    另请参阅

    限制哪些列使用列延迟加载 - 延迟列加载的完整描述

+   **活动历史** - `mapped_column.active_history`确保在属性值更改时，之前的值已加载并作为`AttributeState.history`集合的一部分。检查属性的历史记录时，可能会产生额外的 SQL 语句：

    ```py
    class User(Base):
        __tablename__ = "user"

        id: Mapped[int] = mapped_column(primary_key=True)
        important_identifier: Mapped[str] = mapped_column(active_history=True)
    ```

查看`mapped_column()`的文档字符串，以获取支持的参数列表。

另请参阅

为命令式表列应用加载、持久化和映射选项 - 描述了与命令式表配置一起使用`column_property()`和`deferred()`的用法

### 显式命名声明式映射列

所有到目前为止的示例都是与 ORM 映射属性相关的`mapped_column()`构造相关联的，其中给定给`mapped_column()`的 Python 属性名称也是我们在 CREATE TABLE 语句以及查询中看到的列的名称。在 SQL 中表达的列名称可以通过将字符串位置参数`mapped_column.__name`作为第一个位置参数传递来指示。在下面的示例中，`User`类与给定给列本身的备用名称进行了映射：

```py
class User(Base):
    __tablename__ = "user"

    id: Mapped[int] = mapped_column("user_id", primary_key=True)
    name: Mapped[str] = mapped_column("user_name")
```

在上面的例子中，`User.id`解析为一个名为`user_id`的列，`User.name`解析为一个名为`user_name`的列。我们可以使用我们的 Python 属性名称编写一个`select()`语句，并将看到生成的 SQL 名称：

```py
>>> from sqlalchemy import select
>>> print(select(User.id, User.name).where(User.name == "x"))
SELECT  "user".user_id,  "user".user_name
FROM  "user"
WHERE  "user".user_name  =  :user_name_1 
```

另请参阅

映射表列的备用属性名称 - 适用于命令式表

### 将额外的列附加到现有的声明式映射类

在现有的映射生成`Table`元数据之后，声明性表配置允许向其添加新的`Column`对象。

对于使用声明性基类声明的声明性类，底层元类`DeclarativeMeta`包括一个`__setattr__()`方法，该方法将拦截额外的`mapped_column()`或 Core `Column`对象，并将它们添加到使用`Table.append_column()`的`Table`以及到现有的`Mapper`中使用`Mapper.add_property()`：

```py
MyClass.some_new_column = mapped_column(String)
```

使用核心`Column`：

```py
MyClass.some_new_column = Column(String)
```

所有参数都受支持，包括备用名称，例如 `MyClass.some_new_column = mapped_column("some_name", String)`。但是，SQL 类型必须明确地传递给 `mapped_column()` 或 `Column` 对象，就像上面的示例中传递 `String` 类型一样。`Mapped` 注释类型无法参与操作。

在使用单表继承的特定情况下，还可以将其他 `Column` 对象添加到映射中，其中在映射的子类上存在其他列，这些列没有自己的 `Table`。这在 单表继承 部分有说明。

另请参见

在声明后向映射类添加关系 - `relationship()` 的类似示例

注

对已映射类进行映射属性的赋值仅在使用“声明基类”类时才能正常工作，这意味着使用用户定义的 `DeclarativeBase` 子类或 `declarative_base()` 或 `registry.generate_base()` 返回的动态生成类。这个“基”类包括一个 Python 元类，它实现了一个特殊的 `__setattr__()` 方法，拦截这些操作。

如果类使用诸如 `registry.mapped()` 或命令式函数诸如 `registry.map_imperatively()` 的装饰器进行映射，则将类映射到映射类的类映射属性的运行时赋值 **不会** 生效。

## 声明式与命令式表（又名混合声明式）

也可以为声明式映射提供预先存在的`Table`对象，或者其他任意`FromClause`构造（例如`Join`或`Subquery`）单独构造。

这被称为“混合声明式”映射，因为类使用声明式样式进行映射配置的所有内容，但映射的`Table`对象是单独生成的，并直接传递给声明过程：

```py
from sqlalchemy import Column, ForeignKey, Integer, String
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    pass

# construct a Table directly.  The Base.metadata collection is
# usually a good choice for MetaData but any MetaData
# collection may be used.

user_table = Table(
    "user",
    Base.metadata,
    Column("id", Integer, primary_key=True),
    Column("name", String),
    Column("fullname", String),
    Column("nickname", String),
)

# construct the User class using this table.
class User(Base):
    __table__ = user_table
```

在上述内容中，使用了一种通过用 MetaData 描述数据库的方法构建了一个`Table`对象。然后可以直接将其应用于声明式映射的类。在此形式中不使用`__tablename__`和`__table_args__`声明式类属性。以上配置通常更易读，可作为内联定义：

```py
class User(Base):
    __table__ = Table(
        "user",
        Base.metadata,
        Column("id", Integer, primary_key=True),
        Column("name", String),
        Column("fullname", String),
        Column("nickname", String),
    )
```

上述样式的一个自然效果是，`__table__`属性本身在类定义块内定义。因此，它可以立即在后续属性中引用，例如下面的示例，演示了在多态映射配置中引用`type`列：

```py
class Person(Base):
    __table__ = Table(
        "person",
        Base.metadata,
        Column("id", Integer, primary_key=True),
        Column("name", String(50)),
        Column("type", String(50)),
    )

    __mapper_args__ = {
        "polymorphic_on": __table__.c.type,
        "polymorhpic_identity": "person",
    }
```

当要映射非`Table`构造时，也使用“命令式表格”形式，例如`Join`或`Subquery`对象。以下是一个示例：

```py
from sqlalchemy import func, select

subq = (
    select(
        func.count(orders.c.id).label("order_count"),
        func.max(orders.c.price).label("highest_order"),
        orders.c.customer_id,
    )
    .group_by(orders.c.customer_id)
    .subquery()
)

customer_select = (
    select(customers, subq)
    .join_from(customers, subq, customers.c.id == subq.c.customer_id)
    .subquery()
)

class Customer(Base):
    __table__ = customer_select
```

关于映射到非`Table`构造的背景信息，请参阅将类映射到多个表和将类映射到任意子查询部分。

当类本身使用另一种属性声明形式（例如 Python 数据类）时，“命令式表格”形式尤其有用。详情请参见将 ORM 映射应用于现有数据类（传统数据类用法）部分。

请参阅

用 MetaData 描述数据库

将 ORM 映射应用于现有数据类（传统数据类使用）

### 映射表列的备用属性名称

明确命名声明式映射列说明了如何使用`mapped_column()`为生成的`Column`对象提供一个与其映射的属性名称不同的特定名称。

在使用命令式表配置时，我们已经有`Column`对象存在。要将这些映射到备用名称，我们可以直接将`Column`分配给所需的属性：

```py
user_table = Table(
    "user",
    Base.metadata,
    Column("user_id", Integer, primary_key=True),
    Column("user_name", String),
)

class User(Base):
    __table__ = user_table

    id = user_table.c.user_id
    name = user_table.c.user_name
```

上面的`User`映射将通过`User.id`和`User.name`属性引用`"user_id"`和`"user_name"`列，就像在 Naming Declarative Mapped Columns Explicitly 中演示的那样。

以上映射的一个注意事项是，当使用[**PEP 484**](https://peps.python.org/pep-0484/)类型工具时，对`Column`的直接内联链接将无法正确键入。解决此问题的一种策略是在`column_property()`函数内应用`Column`对象；虽然`Mapper`已经自动生成了此属性对象供其内部使用，但通过在类声明中命名它，类型工具将能够将属性与`Mapped`注释匹配起来：

```py
from sqlalchemy.orm import column_property
from sqlalchemy.orm import Mapped

class User(Base):
    __table__ = user_table

    id: Mapped[int] = column_property(user_table.c.user_id)
    name: Mapped[str] = column_property(user_table.c.user_name)
```

另请参阅

明确命名声明式映射列 - 适用于声明式表  ### 将加载、持久化和映射选项应用于命令式表列

在使用 `mapped_column()` 构造与 Declarative Table 配置时，本节回顾了如何设置加载和持久性选项。在使用 Imperative Table 配置时，我们已经有现有的 `Column` 对象被映射。为了将这些 `Column` 对象与 ORM 映射特定的附加参数一起映射，我们可以使用 `column_property()` 和 `deferred()` 构造来将附加参数与列关联起来。选项包括：

+   **延迟列加载** - `deferred()` 函数是使用 `column_property()` 与 `column_property.deferred` 参数设置为 `True` 的简写；这个构造默认使用 延迟列加载。在下面的示例中，`User.bio` 列默认不会被加载，只有在访问时才会加载：

    ```py
    from sqlalchemy.orm import deferred

    user_table = Table(
        "user",
        Base.metadata,
        Column("id", Integer, primary_key=True),
        Column("name", String),
        Column("bio", Text),
    )

    class User(Base):
        __table__ = user_table

        bio = deferred(user_table.c.bio)
    ```

> 请参阅
> 
> 限制哪些列与列延迟加载 - 延迟列加载的完整描述

+   **活动历史** - `column_property.active_history` 确保在属性值更改时，之前的值将被加载并作为 `AttributeState.history` 集合的一部分，当检查属性历史时。这可能会产生额外的 SQL 语句：

    ```py
    from sqlalchemy.orm import deferred

    user_table = Table(
        "user",
        Base.metadata,
        Column("id", Integer, primary_key=True),
        Column("important_identifier", String),
    )

    class User(Base):
        __table__ = user_table

        important_identifier = column_property(
            user_table.c.important_identifier, active_history=True
        )
    ```

请参阅

`column_property()` 构造在类被映射到其他 FROM 子句（如连接和选择）的情况下也很重要。关于这些情况的更多背景信息在：

+   将类映射到多个表

+   SQL 表达式作为映射属性

对于使用`mapped_column()`进行声明式表配置的情况，大多数选项都可以直接使用；请参阅为声明式映射列设置加载和持久性选项部分的示例。  ### 映射表列的备用属性名称

命名声明式映射列 部分演示了如何使用`mapped_column()`为生成的`Column`对象提供一个与其映射的属性名称分离的特定名称。

在使用命令式表配置时，我们已经有了`Column`对象。要将这些映射到备用名称，我们可以直接将`Column`分配给所需的属性：

```py
user_table = Table(
    "user",
    Base.metadata,
    Column("user_id", Integer, primary_key=True),
    Column("user_name", String),
)

class User(Base):
    __table__ = user_table

    id = user_table.c.user_id
    name = user_table.c.user_name
```

上述`User`映射将通过`User.id`和`User.name`属性引用`"user_id"`和`"user_name"`列，方式与 Naming Declarative Mapped Columns Explicitly 中演示的相同。

上述映射的一个警告是，使用[**PEP 484**](https://peps.python.org/pep-0484/)类型工具时，直接内联到`Column`的链接将不会正确类型化。解决此问题的策略是在`column_property()`函数中应用`Column`对象；虽然`Mapper`已经自动生成了用于内部使用的此属性对象，通过在类声明中命名它，类型工具将能够将属性与`Mapped`注释匹配：

```py
from sqlalchemy.orm import column_property
from sqlalchemy.orm import Mapped

class User(Base):
    __table__ = user_table

    id: Mapped[int] = column_property(user_table.c.user_id)
    name: Mapped[str] = column_property(user_table.c.user_name)
```

另请参阅

命名声明式映射列 - 适用于声明式表

### 为命令式表列应用加载、持久性和映射选项

在设置声明性映射列的加载和持久化选项一节中，我们讨论了在使用声明性表配置时如何设置加载和持久化选项。当使用命令式表配置时，我们已经有了现有的`Column`对象进行映射。为了将这些`Column`对象与与 ORM 映射相关的其他参数一起映射，我们可以使用`column_property()`和`deferred()`构造来将其他参数与列相关联。选项包括：

+   **延迟列加载** - `deferred()`函数是调用具有`column_property.deferred`参数设置为`True`的`column_property()`的速记方式；此构造默认情况下使用延迟列加载来建立`Column`。在下面的示例中，`User.bio`列不会默认加载，而是在访问时加载：

    ```py
    from sqlalchemy.orm import deferred

    user_table = Table(
        "user",
        Base.metadata,
        Column("id", Integer, primary_key=True),
        Column("name", String),
        Column("bio", Text),
    )

    class User(Base):
        __table__ = user_table

        bio = deferred(user_table.c.bio)
    ```

> 另请参阅
> 
> 限制加载列的范围与列延迟加载 - 列延迟加载的完整说明

+   **活跃历史** - `column_property.active_history`确保在属性值更改时，先前的值将已加载并作为`AttributeState.history`集合的一部分，以便在检查属性历史时。这可能会引起额外的 SQL 语句：

    ```py
    from sqlalchemy.orm import deferred

    user_table = Table(
        "user",
        Base.metadata,
        Column("id", Integer, primary_key=True),
        Column("important_identifier", String),
    )

    class User(Base):
        __table__ = user_table

        important_identifier = column_property(
            user_table.c.important_identifier, active_history=True
        )
    ```

另请参阅

`column_property()`构造在将类映射到替代 FROM 子句（如连接和选择）的情况下也很重要。有关这些情况的更多背景信息，请参阅：

+   将类映射到多个表

+   SQL 表达式作为映射属性

对于具有 `mapped_column()` 的声明式表配置，大多数选项都是直接可用的；参见 为声明式映射列设置加载和持久性选项 部分中的示例。

## 使用反射表声明性地映射

有几种可用的模式，可以根据从数据库中内省的一系列 `Table` 对象生成映射的类，使用在 反射数据库对象 中描述的反射过程。

从数据库中反映一个表并将类映射到它的简单方法是使用声明式混合映射，将 `Table.autoload_with` 参数传递给 `Table` 的构造函数：

```py
from sqlalchemy import create_engine
from sqlalchemy import Table
from sqlalchemy.orm import DeclarativeBase

engine = create_engine("postgresql+psycopg2://user:pass@hostname/my_existing_database")

class Base(DeclarativeBase):
    pass

class MyClass(Base):
    __table__ = Table(
        "mytable",
        Base.metadata,
        autoload_with=engine,
    )
```

以上模式的一个变种，可以针对许多表扩展，即使用 `MetaData.reflect()` 方法一次反射一组完整的 `Table` 对象，然后从 `MetaData` 中引用它们：

```py
from sqlalchemy import create_engine
from sqlalchemy import Table
from sqlalchemy.orm import DeclarativeBase

engine = create_engine("postgresql+psycopg2://user:pass@hostname/my_existing_database")

class Base(DeclarativeBase):
    pass

Base.metadata.reflect(engine)

class MyClass(Base):
    __table__ = Base.metadata.tables["mytable"]
```

使用 `__table__` 方法的一个注意事项是，映射的类不能在表被反射之前声明，这需要数据库连接源在声明应用程序类时存在；通常情况下，类是在应用程序模块被导入时声明的，但是数据库连接在应用程序启动运行代码时才可用，以便它可以使用配置信息并创建引擎。目前有两种方法可以解决这个问题，分别在接下来的两个部分中描述。

### 使用 DeferredReflection

为了适应在反射表元数据之后声明映射类的用例，有一个简单的扩展称为 `DeferredReflection` mixin 可用，它改变了声明性映射过程，延迟直到调用一个特殊的类级 `DeferredReflection.prepare()` 方法，该方法将针对目标数据库执行反射过程，并将结果与声明性表映射过程集成，也就是说，使用 `__tablename__` 属性的类：

```py
from sqlalchemy.ext.declarative import DeferredReflection
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    pass

class Reflected(DeferredReflection):
    __abstract__ = True

class Foo(Reflected, Base):
    __tablename__ = "foo"
    bars = relationship("Bar")

class Bar(Reflected, Base):
    __tablename__ = "bar"

    foo_id = mapped_column(Integer, ForeignKey("foo.id"))
```

在上面，我们创建了一个混合类`Reflected`，它将作为我们的声明性层次结构中的类的基类，在调用`Reflected.prepare`方法时应该成为映射。在我们这样做之前，上述映射是不完整的，给定一个`Engine`：

```py
engine = create_engine("postgresql+psycopg2://user:pass@hostname/my_existing_database")
Reflected.prepare(engine)
```

`Reflected` 类的目的是定义应该在哪个范围内进行反射映射。插件将在调用`.prepare()`的目标的子类树中搜索，并反射所有由声明的类命名的表；不是映射的目标数据库中的表，也不是通过外键约束与目标表相关联的表将不被反射。

### 使用 Automap

对于映射到现有数据库并使用表反射的自动化解决方案是使用 Automap 扩展。此扩展将从数据库模式生成完整的映射类，包括根据观察到的外键约束之间的关系的类。虽然它包括用于定制的钩子，例如允许自定义类命名和关系命名方案的钩子，但 automap 面向的是一种迅速的零配置工作风格。如果应用程序希望拥有一个完全明确的模型，该模型利用了表反射，那么 DeferredReflection 类可能更可取，因为它的自动化程度较低。

另请参见

Automap

### 从反射表自动化列命名方案

当使用任何前述的反射技术时，我们可以选择更改映射列的命名方案。`Column` 对象包括一个参数`Column.key`，它是一个字符串名称，确定这个 `Column` 在 `Table.c` 集合中的名称，在不考虑列的 SQL 名称的情况下。如果没有通过其他手段提供，如在 映射表列的备用属性名称 中所示，此键也被 `Mapper` 用作将 `Column` 映射到的属性名称。

当使用表反射时，我们可以拦截 `Column` 将要使用的参数，使用 `DDLEvents.column_reflect()` 事件接收它们，并应用我们需要的任何更改，包括 `.key` 属性以及数据类型等。

事件钩子最容易与正在使用的 `MetaData` 对象相关联，如下所示：

```py
from sqlalchemy import event
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    pass

@event.listens_for(Base.metadata, "column_reflect")
def column_reflect(inspector, table, column_info):
    # set column.key = "attr_<lower_case_name>"
    column_info["key"] = "attr_%s" % column_info["name"].lower()
```

使用上述事件，`Column` 对象的反射将被我们的事件拦截，该事件添加了一个新的“.key”元素，如下面的映射所示：

```py
class MyClass(Base):
    __table__ = Table("some_table", Base.metadata, autoload_with=some_engine)
```

该方法还适用于 `DeferredReflection` 基类以及 Automap 扩展。对于 automap，特别是请参阅 Intercepting Column Definitions 部分了解背景信息。

另请参阅

使用反射表声明式映射

`DDLEvents.column_reflect()`

拦截列定义 - 在 Automap 文档中  ### 映射到一组显式主键列

为了成功映射一个表，`Mapper` 结构始终要求至少有一个列被标识为该可选择内容的“主键”。这样，当加载或持久化 ORM 对象时，它可以被放置在标识映射中，并具有适当的标识键。

在需要映射的反射表不包含主键约束的情况下，以及在映射对任意可选择内容的情况下，主键列可能不存在的一般情况下，提供了 `Mapper.primary_key` 参数，以便将任何一组列配置为表的“主键”，就 ORM 映射而言。

给定以下 Imperative Table 映射的示例，针对一个现有的 `Table` 对象，该表没有声明任何主键（这可能在反射场景中发生），我们可以像以下示例那样映射这样的表：

```py
from sqlalchemy import Column
from sqlalchemy import MetaData
from sqlalchemy import String
from sqlalchemy import Table
from sqlalchemy import UniqueConstraint
from sqlalchemy.orm import DeclarativeBase

metadata = MetaData()
group_users = Table(
    "group_users",
    metadata,
    Column("user_id", String(40), nullable=False),
    Column("group_id", String(40), nullable=False),
    UniqueConstraint("user_id", "group_id"),
)

class Base(DeclarativeBase):
    pass

class GroupUsers(Base):
    __table__ = group_users
    __mapper_args__ = {"primary_key": [group_users.c.user_id, group_users.c.group_id]}
```

在上面的例子中，`group_users`表是某种类型的关联表，具有字符串列`user_id`和`group_id`，但没有设置主键；相反，只有一个`UniqueConstraint` 建立了这两列表示唯一键的约束。`Mapper` 不会自动检查主键的唯一约束；相反，我们利用`Mapper.primary_key` 参数，传递一个集合`[group_users.c.user_id, group_users.c.group_id]`，表示这两列应该用于构造`GroupUsers`类的实例的标识键。### 映射表列的子集

有时，表反射可能会提供一个`Table`，其中有许多对我们的需求不重要且可以安全忽略的列。对于这样一个表，其中有许多列不需要在应用程序中引用，`Mapper.include_properties` 或 `Mapper.exclude_properties` 参数可以指示要映射的列的子集，其中来自目标`Table` 的其他列不会被 ORM 以任何方式考虑。例如：

```py
class User(Base):
    __table__ = user_table
    __mapper_args__ = {"include_properties": ["user_id", "user_name"]}
```

在上面的示例中，`User`类将映射到`user_table`表，只包括`user_id`和`user_name`列 - 其余列不会被引用。

同样地：

```py
class Address(Base):
    __table__ = address_table
    __mapper_args__ = {"exclude_properties": ["street", "city", "state", "zip"]}
```

将`Address`类映射到`address_table`表，包括除了`street`、`city`、`state`和`zip`之外的所有列。

如两个示例所示，列可以通过字符串名称或直接引用`Column`对象来引用。直接引用对象可能对明确性有用，也可用于解决映射到可能具有重复名称的多表构造时的歧义：

```py
class User(Base):
    __table__ = user_table
    __mapper_args__ = {
        "include_properties": [user_table.c.user_id, user_table.c.user_name]
    }
```

当列未包含在映射中时，在执行`select()` 或传统的 `Query` 对象时，这些列将不会在任何 SELECT 语句中引用，映射类中也不会有任何表示该列的映射属性；将其名称分配为属性将不会产生其他效果，仅仅与普通的 Python 属性赋值相同。

但是，需要注意的是**模式级列默认值仍然有效**，对于那些包括它们的`Column`对象，即使它们可能被排除在 ORM 映射之外。

“模式级列默认值”指的是在列插入/更新默认值中描述的默认值，包括由`Column.default`、`Column.onupdate`、`Column.server_default`和`Column.server_onupdate`参数配置的默认值。这些结构继续具有正常的效果，因为在`Column.default`和`Column.onupdate`的情况下，`Column`对象仍然存在于底层的`Table`上，因此当 ORM 发出 INSERT 或 UPDATE 时，可以发生默认函数，并且在`Column.server_default`和`Column.server_onupdate`的情况下，关系数据库本身会作为服务器端行为发出这些默认值。### 使用 DeferredReflection

为了适应声明映射类的用例，在此之后可以发生表元数据的反射，提供了一个简单的扩展叫做`DeferredReflection` mixin，它修改了声明性映射过程，延迟到调用一个特殊的类级`DeferredReflection.prepare()`方法，该方法将执行对目标数据库的反射过程，并将结果与声明性表映射过程集成，即使用`__tablename__`属性的类：

```py
from sqlalchemy.ext.declarative import DeferredReflection
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    pass

class Reflected(DeferredReflection):
    __abstract__ = True

class Foo(Reflected, Base):
    __tablename__ = "foo"
    bars = relationship("Bar")

class Bar(Reflected, Base):
    __tablename__ = "bar"

    foo_id = mapped_column(Integer, ForeignKey("foo.id"))
```

在上面，我们创建了一个名为 `Reflected` 的混合类，它将作为我们声明性层次结构中的类的基类，当调用 `Reflected.prepare` 方法时，这些类应该成为映射。在我们这样做之前，上述映射是不完整的，给定一个 `Engine`：

```py
engine = create_engine("postgresql+psycopg2://user:pass@hostname/my_existing_database")
Reflected.prepare(engine)
```

`Reflected` 类的目的是定义应在其中反射映射类的范围。插件将针对调用 `.prepare()` 的目标的子类树中搜索，并反射所有由声明类命名的表；不属于映射的目标数据库中的表，也不通过外键约束与目标表相关联的表将不被反射。

### 使用自动映射

映射到现有数据库并使用表反射的更自动化的解决方案是使用 自动映射 扩展。该扩展将从数据库架构中生成完整的映射类，包括基于观察到的外键约束的类之间的关系。虽然它包括用于自定义的钩子，例如允许自定义类命名和关系命名方案的钩子，但自动映射是面向一种便捷的零配置工作风格。如果应用程序希望拥有完全明确的模型，该模型利用表反射，则可能更喜欢 延迟反射 类，因为它的方法较不自动化。

另请参阅

自动映射

### 从反射表自动命名列方案

在使用任何先前的反射技术时，我们可以选择更改列映射的命名方案。`Column` 对象包括一个参数 `Column.key`，它是一个字符串名称，确定了这个 `Column` 将以什么名称出现在 `Table.c` 集合中，与列的 SQL 名称无关。如果没有通过其他方式提供，比如在 用于映射表列的替代属性名称 中所示的方式，该键也将被 `Mapper` 用作属性名称，该属性将被映射到 `Column`。

在使用表反射时，我们可以拦截将用于`Column`的参数，因为它们是使用`DDLEvents.column_reflect()`事件接收的，并应用我们需要的任何更改，包括`.key`属性，以及诸如数据类型之类的内容。

事件挂钩最容易与正在使用的`MetaData`对象相关联，如下所示：

```py
from sqlalchemy import event
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    pass

@event.listens_for(Base.metadata, "column_reflect")
def column_reflect(inspector, table, column_info):
    # set column.key = "attr_<lower_case_name>"
    column_info["key"] = "attr_%s" % column_info["name"].lower()
```

有了上述事件，`Column`对象的反射将被我们的事件拦截，该事件添加了一个新的“.key”元素，例如在下面的映射中：

```py
class MyClass(Base):
    __table__ = Table("some_table", Base.metadata, autoload_with=some_engine)
```

该方法还适用于`DeferredReflection`基类以及 Automap 扩展。对于自动映射，具体请参阅背景信息的拦截列定义部分。

另见

使用反射表声明式映射

`DDLEvents.column_reflect()`

拦截列定义 - 在 Automap 文档中

### 映射到显式主键列集合

为了成功映射表，`Mapper` 构造始终要求至少有一个列被标识为该可选择的“主键”。这样，当加载或持久化 ORM 对象时，它可以被放置在身份映射中，具有适当的身份键。

在那些要映射的反射表中不包括主键约束的情况下，以及在映射到任意选择项的一般情况下，其中可能不存在主键列的情况下，提供了 `Mapper.primary_key` 参数，以便任何一组列可以被配置为表的“主键”，就 ORM 映射而言。

给出了一个关于现有 `Table` 对象的命令式表映射的示例，在该表中没有声明任何主键（在反射场景中可能会发生），我们可以将这样的表映射为以下示例中的方式：

```py
from sqlalchemy import Column
from sqlalchemy import MetaData
from sqlalchemy import String
from sqlalchemy import Table
from sqlalchemy import UniqueConstraint
from sqlalchemy.orm import DeclarativeBase

metadata = MetaData()
group_users = Table(
    "group_users",
    metadata,
    Column("user_id", String(40), nullable=False),
    Column("group_id", String(40), nullable=False),
    UniqueConstraint("user_id", "group_id"),
)

class Base(DeclarativeBase):
    pass

class GroupUsers(Base):
    __table__ = group_users
    __mapper_args__ = {"primary_key": [group_users.c.user_id, group_users.c.group_id]}
```

上述的 `group_users` 表是某种关联表，具有字符串列 `user_id` 和 `group_id`，但没有设置主键；相反，只有一个 `UniqueConstraint` 建立了这两列代表唯一键的关系。`Mapper` 不会自动检查唯一约束以获取主键；而是我们利用 `Mapper.primary_key` 参数，传递一个集合 `[group_users.c.user_id, group_users.c.group_id]`，表示这两列应该用于构建 `GroupUsers` 类的标识键。

### 映射表列的子集

有时，表反射可能提供具有许多对我们的需求不重要且可以安全忽略的列的 `Table`。对于这样一个具有许多不需要在应用程序中引用的列的表，`Mapper.include_properties` 或 `Mapper.exclude_properties` 参数可以指示要映射的列的子集，其中来自目标 `Table` 的其他列将不会以任何方式被 ORM 考虑。示例：

```py
class User(Base):
    __table__ = user_table
    __mapper_args__ = {"include_properties": ["user_id", "user_name"]}
```

在上述示例中，`User` 类将映射到 `user_table` 表，只包括 `user_id` 和 `user_name` 列 - 其余列不会被引用。

类似地：

```py
class Address(Base):
    __table__ = address_table
    __mapper_args__ = {"exclude_properties": ["street", "city", "state", "zip"]}
```

将 `Address` 类映射到 `address_table` 表，包括除 `street`、`city`、`state` 和 `zip` 之外的所有列。

如两个示例所示，列可以通过字符串名称或直接引用 `Column` 对象来引用。直接引用对象可能有助于明确以及解决映射到可能具有重复名称的多表结构时的歧义：

```py
class User(Base):
    __table__ = user_table
    __mapper_args__ = {
        "include_properties": [user_table.c.user_id, user_table.c.user_name]
    }
```

当列没有包含在映射中时，在执行 `select()` 或传统的 `Query` 对象时，这些列将不会被引用到任何 SELECT 语句中，并且映射类上不会有任何代表该列的映射属性；给定该名称的属性赋值将不会产生任何效果，仅仅是普通的 Python 属性赋值。

然而，需要注意的是**模式级列默认值仍然会生效**于那些包含它们的`Column`对象，即使它们可能被排除在 ORM 映射之外。

“模式级列默认值”指的是在列插入/更新默认值中描述的默认值，包括由`Column.default`、`Column.onupdate`、`Column.server_default`和`Column.server_onupdate`参数配置的默认值。这些构造继续具有正常的效果，因为在`Column.default`和`Column.onupdate`的情况下，`Column`对象仍然存在于底层的`Table`上，因此在 ORM 发出 INSERT 或 UPDATE 时允许默认函数发生作用，而在`Column.server_default`和`Column.server_onupdate`的情况下，关系数据库本身将这些默认值作为服务器端行为发出。
