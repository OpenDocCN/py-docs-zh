# 与 dataclasses 和 attrs 集成

> 原文：[`docs.sqlalchemy.org/en/20/orm/dataclasses.html`](https://docs.sqlalchemy.org/en/20/orm/dataclasses.html)

SQLAlchemy 从 2.0 版本开始具有“本地数据类”集成，在带注释的声明表映射中，可以通过向映射类添加单个 mixin 或装饰器将其转换为 Python [dataclass](https://docs.python.org/3/library/dataclasses.html)。

2.0 版本中的新功能：将数据类创建与 ORM 声明类集成

还有一些可用的模式，允许将现有的数据类映射，以及映射由第三方集成库[attrs](https://pypi.org/project/attrs/)仪表化的类。

## 声明式数据类映射

SQLAlchemy 带注释的声明表映射可以通过附加的 mixin 类或装饰器指令进行扩展，在映射完成后将映射类**原地**转换为 Python [dataclass](https://docs.python.org/3/library/dataclasses.html)，然后完成应用 ORM 特定的仪表化到类的映射过程。这提供的最突出的行为增加是生成具有对位置和关键字参数具有或不具有默认值的精细控制的`__init__()`方法，以及生成诸如`__repr__()`和`__eq__()`等方法。

从[**PEP 484**](https://peps.python.org/pep-0484/)的类型化角度来看，该类被认为具有特定于 Dataclass 的行为，最重要的是通过利用[**PEP 681**](https://peps.python.org/pep-0681/)“数据类转换”，使类型工具可以将该类视为明确使用了`@dataclasses.dataclass`装饰器装饰的类。

注意

**2023 年 4 月 4 日**之后，对于[**PEP 681**](https://peps.python.org/pep-0681/)在类型工具中的支持是有限的，目前已知由[Pyright](https://github.com/microsoft/pyright)以及**1.2 版**的[Mypy](https://mypy.readthedocs.io/en/stable/)支持。请注意，Mypy 1.1.1 引入了[**PEP 681**](https://peps.python.org/pep-0681/)支持，但未正确适配 Python 描述符，这将导致在使用 SQLAlchemy 的 ORM 映射方案时出现错误。

另请参阅

[`peps.python.org/pep-0681/#the-dataclass-transform-decorator`](https://peps.python.org/pep-0681/#the-dataclass-transform-decorator) - 有关像 SQLAlchemy 这样的库如何实现[**PEP 681**](https://peps.python.org/pep-0681/)支持的背景信息

数据类转换可以通过将`MappedAsDataclass`混入添加到`DeclarativeBase`类层次结构中的任何声明性类，或通过使用`registry.mapped_as_dataclass()`类装饰器进行装饰器映射来添加。

`MappedAsDataclass`混入可以应用于声明性`Base`类或任何超类，如下例所示：

```py
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import MappedAsDataclass

class Base(MappedAsDataclass, DeclarativeBase):
  """subclasses will be converted to dataclasses"""

class User(Base):
    __tablename__ = "user_account"

    id: Mapped[int] = mapped_column(init=False, primary_key=True)
    name: Mapped[str]
```

或者直接应用于从声明性基类扩展的类：

```py
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import MappedAsDataclass

class Base(DeclarativeBase):
    pass

class User(MappedAsDataclass, Base):
  """User class will be converted to a dataclass"""

    __tablename__ = "user_account"

    id: Mapped[int] = mapped_column(init=False, primary_key=True)
    name: Mapped[str]
```

当使用装饰器形式时，仅支持`registry.mapped_as_dataclass()`装饰器：

```py
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import registry

reg = registry()

@reg.mapped_as_dataclass
class User:
    __tablename__ = "user_account"

    id: Mapped[int] = mapped_column(init=False, primary_key=True)
    name: Mapped[str]
```

### 类级特性配置

对于数据类功能的支持是部分的。目前**支持**的功能有`init`、`repr`、`eq`、`order`和`unsafe_hash`，`match_args`和`kw_only`在 Python 3.10+上支持。目前**不支持**的功能有`frozen`和`slots`。

当使用混入类形式的`MappedAsDataclass`时，类配置参数作为类级参数传递：

```py
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import MappedAsDataclass

class Base(DeclarativeBase):
    pass

class User(MappedAsDataclass, Base, repr=False, unsafe_hash=True):
  """User class will be converted to a dataclass"""

    __tablename__ = "user_account"

    id: Mapped[int] = mapped_column(init=False, primary_key=True)
    name: Mapped[str]
```

当使用具有`registry.mapped_as_dataclass()`的装饰器形式时，类配置参数直接传递给装饰器：

```py
from sqlalchemy.orm import registry
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column

reg = registry()

@reg.mapped_as_dataclass(unsafe_hash=True)
class User:
  """User class will be converted to a dataclass"""

    __tablename__ = "user_account"

    id: Mapped[int] = mapped_column(init=False, primary_key=True)
    name: Mapped[str]
```

关于数据类类选项的背景，请参阅[@dataclasses.dataclass](https://docs.python.org/3/library/dataclasses.html#dataclasses.dataclass)中的[dataclasses](https://docs.python.org/3/library/dataclasses.html)文档。

### 属性配置

SQLAlchemy 原生数据类与普通数据类的不同之处在于，要映射的属性在所有情况下都是使用`Mapped`泛型注解容器来描述的。映射遵循与带有 mapped_column()的声明性表中记录的相同形式，`mapped_column()`和`Mapped`的所有功能都得到支持。

另外，ORM 属性配置结构，包括`mapped_column()`，`relationship()`和`composite()`支持**每个属性字段选项**，包括`init`，`default`，`default_factory`和`repr`。这些参数的名称是按照[**PEP 681**](https://peps.python.org/pep-0681/)中指定的固定的。功能与 dataclasses 相同：

+   `init`，如`mapped_column.init`，`relationship.init`，如果为 False，则表示该字段不应该是`__init__()`方法的一部分

+   `default`，比如`mapped_column.default`，`relationship.default`，表示字段的默认值，可以在`__init__()`方法中作为关键字参数给出。

+   `default_factory`，如`mapped_column.default_factory`，`relationship.default_factory`，表示一个可调用的函数，如果未显式传递给`__init__()`方法，则将调用该函数生成参数的新默认值。

+   `repr`默认为 True，表示该字段应该是生成的`__repr__()`方法的一部分

与 dataclasses 的另一个关键区别是，属性的默认值**必须**使用 ORM 结构的`default`参数进行配置，例如`mapped_column(default=None)`。不支持类似于 dataclass 语法的语法，该语法接受简单的 Python 值作为默认值，而无需使用`@dataclasses.field()`。

以`mapped_column()`为例，下面的映射将产生一个`__init__()`方法，该方法仅接受`name`和`fullname`字段，其中`name`是必需的，可以作为位置参数传递，而`fullname`是可选的。我们预期`id`字段是由数据库生成的，根本不是构造函数的一部分：

```py
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import registry

reg = registry()

@reg.mapped_as_dataclass
class User:
    __tablename__ = "user_account"

    id: Mapped[int] = mapped_column(init=False, primary_key=True)
    name: Mapped[str]
    fullname: Mapped[str] = mapped_column(default=None)

# 'fullname' is optional keyword argument
u1 = User("name")
```

#### 列默认值

为了适应 `default` 参数与 `Column.default` 构造函数中现有的参数重叠，`mapped_column()` 构造函数通过添加一个新的参数 `mapped_column.insert_default` 来消除这两个名称的歧义，该参数将直接填充到 `Column.default` 参数中，独立于 `mapped_column.default` 的设置，后者始终用于数据类配置。例如，要配置一个日期时间列，其 `Column.default` 设置为 `func.utc_timestamp()` SQL 函数，但构造函数中该参数是可选的：

```py
from datetime import datetime

from sqlalchemy import func
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import registry

reg = registry()

@reg.mapped_as_dataclass
class User:
    __tablename__ = "user_account"

    id: Mapped[int] = mapped_column(init=False, primary_key=True)
    created_at: Mapped[datetime] = mapped_column(
        insert_default=func.utc_timestamp(), default=None
    )
```

使用上述映射，对于未传递 `created_at` 参数的新 `User` 对象的 `INSERT` 将如下进行：

```py
>>> with Session(e) as session:
...     session.add(User())
...     session.commit()
BEGIN  (implicit)
INSERT  INTO  user_account  (created_at)  VALUES  (utc_timestamp())
[generated  in  0.00010s]  ()
COMMIT 
```

#### 与 Annotated 的集成

将整个列声明映射到 Python 类型 中介绍的方法演示了如何使用 [**PEP 593**](https://peps.python.org/pep-0593/) 的 `Annotated` 对象将整个 `mapped_column()` 构造函数打包以供重用。此功能受数据类功能支持。但是，该功能的一个方面在使用类型工具时需要一个解决方法，即 [**PEP 681**](https://peps.python.org/pep-0681/) 特定参数 `init`、`default`、`repr` 和 `default_factory` **必须** 包含在右侧，并打包到显式的 `mapped_column()` 构造函数中，以便类型工具正确解释属性。例如，下面的方法在运行时完全正常工作，但是类型工具将认为 `User()` 构造函数是无效的，因为它们看不到 `init=False` 参数存在：

```py
from typing import Annotated

from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import registry

# typing tools will ignore init=False here
intpk = Annotated[int, mapped_column(init=False, primary_key=True)]

reg = registry()

@reg.mapped_as_dataclass
class User:
    __tablename__ = "user_account"
    id: Mapped[intpk]

# typing error: Argument missing for parameter "id"
u1 = User()
```

相反，`mapped_column()` 必须同时出现在右侧，并对 `mapped_column.init` 进行显式设置；其他参数可以保留在 `Annotated` 构造中：

```py
from typing import Annotated

from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import registry

intpk = Annotated[int, mapped_column(primary_key=True)]

reg = registry()

@reg.mapped_as_dataclass
class User:
    __tablename__ = "user_account"

    # init=False and other pep-681 arguments must be inline
    id: Mapped[intpk] = mapped_column(init=False)

u1 = User()
```

### 使用混合和抽象超类

在`MappedAsDataclass`映射类中使用的任何混合类或基类，其中包括`Mapped`属性，必须本身是`MappedAsDataclass`层次结构的一部分，例如，在下面的示例中使用混合类：

```py
class Mixin(MappedAsDataclass):
    create_user: Mapped[int] = mapped_column()
    update_user: Mapped[Optional[int]] = mapped_column(default=None, init=False)

class Base(DeclarativeBase, MappedAsDataclass):
    pass

class User(Base, Mixin):
    __tablename__ = "sys_user"

    uid: Mapped[str] = mapped_column(
        String(50), init=False, default_factory=uuid4, primary_key=True
    )
    username: Mapped[str] = mapped_column()
    email: Mapped[str] = mapped_column()
```

否则，支持[**PEP 681**](https://peps.python.org/pep-0681/)的 Python 类型检查器将不考虑来自非数据类混合类的属性作为数据类的一部分。

自 2.0.8 版已弃用：在`MappedAsDataclass`或`registry.mapped_as_dataclass()`层次结构中使用混合类和抽象基类，这些类本身不是数据类，因此这些字段不受[**PEP 681**](https://peps.python.org/pep-0681/)支持。对于此情况，将发出警告，以后将是一个错误。

另请参阅

将<cls>转换为数据类时，属性来自不是数据类的超类<cls>。 - 关于原因的背景

### 关系配置

当与`relationship()`结合使用时，`Mapped`注释的使用方式与基本关系模式中描述的方式相同。在指定基于集合的`relationship()`作为可选关键字参数时，必须传递`relationship.default_factory`参数，并且它必须指向要使用的集合类。如果默认值为`None`，则多对一和标量对象引用可以使用`relationship.default`：

```py
from typing import List

from sqlalchemy import ForeignKey
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import registry
from sqlalchemy.orm import relationship

reg = registry()

@reg.mapped_as_dataclass
class Parent:
    __tablename__ = "parent"
    id: Mapped[int] = mapped_column(primary_key=True)
    children: Mapped[List["Child"]] = relationship(
        default_factory=list, back_populates="parent"
    )

@reg.mapped_as_dataclass
class Child:
    __tablename__ = "child"
    id: Mapped[int] = mapped_column(primary_key=True)
    parent_id: Mapped[int] = mapped_column(ForeignKey("parent.id"))
    parent: Mapped["Parent"] = relationship(default=None)
```

当构造新的`Parent()`对象而不传递`children`时，上述映射将为`Parent.children`生成一个空列表，类似地，当构造新的`Child()`对象而不传递`parent`时，将为`Child.parent`生成一个`None`值。

虽然 `relationship.default_factory` 可以从 `relationship()` 自身的给定集合类自动推导出来，但这会破坏与 dataclasses 的兼容性，因为 `relationship.default_factory` 或 `relationship.default` 的存在决定了参数在转换为 `__init__()` 方法时是必需的还是可选的。

### 使用非映射数据类字段

当使用声明式数据类时，类上也可以使用非映射字段，这些字段将成为数据类构造过程的一部分，但不会被映射。任何不使用 `Mapped` 的字段都将被映射过程忽略。在下面的示例中，字段 `ctrl_one` 和 `ctrl_two` 将成为对象的实例级状态的一部分，但不会被 ORM 持久化：

```py
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import registry

reg = registry()

@reg.mapped_as_dataclass
class Data:
    __tablename__ = "data"

    id: Mapped[int] = mapped_column(init=False, primary_key=True)
    status: Mapped[str]

    ctrl_one: Optional[str] = None
    ctrl_two: Optional[str] = None
```

上面的 `Data` 实例可以通过以下方式创建：

```py
d1 = Data(status="s1", ctrl_one="ctrl1", ctrl_two="ctrl2")
```

更现实的例子可能是结合使用 Dataclasses 的 `InitVar` 特性和 `__post_init__()` 特性来接收只初始化的字段，这些字段可以用来组成持久化数据。在下面的示例中，`User` 类使用 `id`、`name` 和 `password_hash` 作为映射特性，但使用只初始化的 `password` 和 `repeat_password` 字段来表示用户创建过程（注意：在运行此示例时，请将函数 `your_crypt_function_here()` 替换为第三方加密函数，例如 [bcrypt](https://pypi.org/project/bcrypt/) 或 [argon2-cffi](https://pypi.org/project/argon2-cffi/)）：

```py
from dataclasses import InitVar
from typing import Optional

from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import registry

reg = registry()

@reg.mapped_as_dataclass
class User:
    __tablename__ = "user_account"

    id: Mapped[int] = mapped_column(init=False, primary_key=True)
    name: Mapped[str]

    password: InitVar[str]
    repeat_password: InitVar[str]

    password_hash: Mapped[str] = mapped_column(init=False, nullable=False)

    def __post_init__(self, password: str, repeat_password: str):
        if password != repeat_password:
            raise ValueError("passwords do not match")

        self.password_hash = your_crypt_function_here(password)
```

上述对象使用了 `password` 和 `repeat_password` 参数，这些参数被提前使用，以便生成 `password_hash` 变量：

```py
>>> u1 = User(name="some_user", password="xyz", repeat_password="xyz")
>>> u1.password_hash
'$6$9ppc... (example crypted string....)'
```

从版本 2.0.0rc1 起发生了变化：当使用 `registry.mapped_as_dataclass()` 或 `MappedAsDataclass` 时，不包含 `Mapped` 注解的字段可能会被包括在内，这些字段将被视为结果数据类的一部分，但不会被映射，无需指定 `__allow_unmapped__` 类属性也可以。以前的 2.0 beta 版本需要显式地包含此属性，即使此属性的目的仅是为了使传统的 ORM 类型映射继续正常工作。  ### 与 Pydantic 等备选数据类提供程序集成

警告

Pydantic 的数据类层与 SQLAlchemy 的类仪器不完全兼容，除非进行额外的内部更改，否则许多功能，如相关集合，可能无法正常工作。

为了与 Pydantic 兼容，请考虑使用基于 SQLAlchemy ORM 构建的[SQLModel](https://sqlmodel.tiangolo.com) ORM，该 ORM 包含专门解决这些不兼容性的实现细节。

SQLAlchemy 的`MappedAsDataclass`类和`registry.mapped_as_dataclass()`方法直接调用 Python 标准库的`dataclasses.dataclass`类装饰器，此操作在将声明性映射过程应用于类之后进行。此函数调用可以通过`MappedAsDataclass`作为类关键字参数以及`registry.mapped_as_dataclass()`接受的`dataclass_callable`参数进行替换，以使用 Pydantic 等替代数据类提供程序：

```py
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import MappedAsDataclass
from sqlalchemy.orm import registry

class Base(
    MappedAsDataclass,
    DeclarativeBase,
    dataclass_callable=pydantic.dataclasses.dataclass,
):
    pass

class User(Base):
    __tablename__ = "user"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
```

上述的`User`类将被应用为数据类，使用 Pydantic 的`pydantic.dataclasses.dataclasses`可调用。该过程既适用于映射类，也适用于从`MappedAsDataclass`扩展或直接应用`registry.mapped_as_dataclass()`的混入。

新功能版本 2.0.4：为`MappedAsDataclass`和`registry.mapped_as_dataclass()`添加了`dataclass_callable`类和方法参数，并调整了一些数据类内部，以适应更严格的数据类功能，例如 Pydantic 的功能。## 将 ORM 映射应用于现有数据类（旧版数据类用法）

遗留特性

此处描述的方法已被 2.0 系列 SQLAlchemy 中的声明性数据类映射功能取代。这一更新版本的功能是建立在首次添加到 1.4 版本的数据类支持之上的，该支持在本节中进行了描述。

要映射现有的数据类，不能直接使用 SQLAlchemy 的“内联”声明性指令；ORM 指令通过以下三种技术之一分配：

+   使用“具有命令式表”的方法，要映射的表/列是使用分配给类的`__table__`属性的`Table`对象来定义的；关系在`__mapper_args__`字典中定义。使用`registry.mapped()`装饰器对类进行映射。以下是一个示例，位于使用具有命令式表的预先存在的数据类进行映射。

+   使用完全“声明式”方法，Declarative 解释的指令，如`Column`，`relationship()`被添加到`dataclasses.field()`构造函数的`.metadata`字典中，它们被声明性过程使用。再次使用`registry.mapped()`装饰器对类进行映射。请参见下面的示例，在使用声明式样式字段映射预先存在的数据类。

+   可以使用`registry.map_imperatively()`方法将“命令式”映射应用到现有的数据类上，以完全相同的方式生成映射，就像在命令式映射中描述的那样。下面在使用命令式映射映射预先存在的数据类中进行了说明。

SQLAlchemy 将映射应用到数据类的一般过程与普通类的过程相同，但还包括 SQLAlchemy 将检测到的类级属性，这些属性是数据类声明过程的一部分，并在运行时用通常的 SQLAlchemy ORM 映射属性替换它们。由数据类生成的`__init__`方法保持不变，以及数据类生成的所有其他方法，如`__eq__()`，`__repr__()`等。

### 使用具有命令式表的预先存在的数据类进行映射

下面是使用带命令式表的声明式（即混合声明）的`@dataclass`进行映射的示例。一个完整的`Table`对象被显式地构建并分配给`__table__`属性。使用普通数据类语法定义实例字段。其他`MapperProperty`定义，如`relationship()`，放置在类级别的字典 __mapper_args__ 中，位于`properties`键下，对应于`Mapper.properties`参数：

```py
from __future__ import annotations

from dataclasses import dataclass, field
from typing import List, Optional

from sqlalchemy import Column, ForeignKey, Integer, String, Table
from sqlalchemy.orm import registry, relationship

mapper_registry = registry()

@mapper_registry.mapped
@dataclass
class User:
    __table__ = Table(
        "user",
        mapper_registry.metadata,
        Column("id", Integer, primary_key=True),
        Column("name", String(50)),
        Column("fullname", String(50)),
        Column("nickname", String(12)),
    )
    id: int = field(init=False)
    name: Optional[str] = None
    fullname: Optional[str] = None
    nickname: Optional[str] = None
    addresses: List[Address] = field(default_factory=list)

    __mapper_args__ = {  # type: ignore
        "properties": {
            "addresses": relationship("Address"),
        }
    }

@mapper_registry.mapped
@dataclass
class Address:
    __table__ = Table(
        "address",
        mapper_registry.metadata,
        Column("id", Integer, primary_key=True),
        Column("user_id", Integer, ForeignKey("user.id")),
        Column("email_address", String(50)),
    )
    id: int = field(init=False)
    user_id: int = field(init=False)
    email_address: Optional[str] = None
```

在上面的示例中，`User.id`、`Address.id`和`Address.user_id`属性被定义为`field(init=False)`。这意味着这些参数不会被添加到`__init__()`方法中，但`Session`仍然可以在获取它们的值后通过自动增量或其他默认值生成器进行刷新时设置它们。要允许在构造函数中显式指定它们，它们将被赋予`None`的默认值。

要单独声明一个`relationship()`，需要直接在`Mapper.properties`字典中指定它，该字典本身是在`__mapper_args__`字典中指定的，以便将其传递给`Mapper`的构造函数。这种方法的另一种选择在下一个示例中。

警告

使用`default`设置一个`default`与`init=False`的数据类字段()将不像预期的那样与完全普通的数据类一起工作，因为 SQLAlchemy 类工具将替换数据类创建过程中在类上设置的默认值。而是使用`default_factory`。当使用声明性数据类映射时，此适应过程会自动完成。### 使用声明式字段映射现有数据类

遗留功能

使用数据类进行声明性映射的这种方法应被视为遗留。它将继续受支持，但不太可能提供任何优势，与声明性数据类映射中详细描述的新方法相比。

请注意，**不支持使用 mapped_column()**; 应继续使用`Column`构造在`dataclasses.field()`的`metadata`字段中声明表元数据。

完全的声明性方法要求`Column`对象被声明为类属性，而在使用数据类时会与数据类级别的属性冲突。将这些内容结合在一起的一种方法是利用`dataclass.field`对象上的`metadata`属性，其中可以提供特定于 SQLAlchemy 的映射信息。当类指定属性`__sa_dataclass_metadata_key__`时，声明性支持提取这些参数。这也提供了一种更简洁的方法来指示`relationship()`关联：

```py
from __future__ import annotations

from dataclasses import dataclass, field
from typing import List

from sqlalchemy import Column, ForeignKey, Integer, String
from sqlalchemy.orm import registry, relationship

mapper_registry = registry()

@mapper_registry.mapped
@dataclass
class User:
    __tablename__ = "user"

    __sa_dataclass_metadata_key__ = "sa"
    id: int = field(init=False, metadata={"sa": Column(Integer, primary_key=True)})
    name: str = field(default=None, metadata={"sa": Column(String(50))})
    fullname: str = field(default=None, metadata={"sa": Column(String(50))})
    nickname: str = field(default=None, metadata={"sa": Column(String(12))})
    addresses: List[Address] = field(
        default_factory=list, metadata={"sa": relationship("Address")}
    )

@mapper_registry.mapped
@dataclass
class Address:
    __tablename__ = "address"
    __sa_dataclass_metadata_key__ = "sa"
    id: int = field(init=False, metadata={"sa": Column(Integer, primary_key=True)})
    user_id: int = field(init=False, metadata={"sa": Column(ForeignKey("user.id"))})
    email_address: str = field(default=None, metadata={"sa": Column(String(50))})
```

#### 使用声明性混合类型与预先存在的数据类

在使用混合类型构成映射层次结构部分中，引入了声明性混合类型类。声明性混合类型的一个要求是，某些不能轻易复制的构造必须作为可调用对象给出，使用`declared_attr`装饰器，例如在混合关系示例中：

```py
class RefTargetMixin:
    @declared_attr
    def target_id(cls):
        return Column("target_id", ForeignKey("target.id"))

    @declared_attr
    def target(cls):
        return relationship("Target")
```

在数据类`field()`对象中，通过使用 lambda 表示 SQLAlchemy 构造来支持此形式。使用`declared_attr()`将 lambda 包围起来是可选的。如果我们想要生成上面的`User`类，其中 ORM 字段来自一个本身就是数据类的 mixin，形式将是：

```py
@dataclass
class UserMixin:
    __tablename__ = "user"

    __sa_dataclass_metadata_key__ = "sa"

    id: int = field(init=False, metadata={"sa": Column(Integer, primary_key=True)})

    addresses: List[Address] = field(
        default_factory=list, metadata={"sa": lambda: relationship("Address")}
    )

@dataclass
class AddressMixin:
    __tablename__ = "address"
    __sa_dataclass_metadata_key__ = "sa"
    id: int = field(init=False, metadata={"sa": Column(Integer, primary_key=True)})
    user_id: int = field(
        init=False, metadata={"sa": lambda: Column(ForeignKey("user.id"))}
    )
    email_address: str = field(default=None, metadata={"sa": Column(String(50))})

@mapper_registry.mapped
class User(UserMixin):
    pass

@mapper_registry.mapped
class Address(AddressMixin):
    pass
```

新版本 1.4.2 中：为“声明属性”样式的 mixin 属性增加了支持，即`relationship()`构造以及带有外键声明的`Column`对象，用于在“带有声明性表格的数据类”样式映射中使用。### 使用命令式映射映射预先存在的数据类

如前所述，使用 `@dataclass` 装饰器设置为 dataclass 的类可以进一步使用 `registry.mapped()` 装饰器进行装饰，以便对类应用声明式样式的映射。作为使用 `registry.mapped()` 装饰器的替代方案，我们也可以将类通过 `registry.map_imperatively()` 方法传递，这样我们就可以将所有 `Table` 和 `Mapper` 的配置以命令方式传递给函数，而不是将它们定义为类变量：

```py
from __future__ import annotations

from dataclasses import dataclass
from dataclasses import field
from typing import List

from sqlalchemy import Column
from sqlalchemy import ForeignKey
from sqlalchemy import Integer
from sqlalchemy import MetaData
from sqlalchemy import String
from sqlalchemy import Table
from sqlalchemy.orm import registry
from sqlalchemy.orm import relationship

mapper_registry = registry()

@dataclass
class User:
    id: int = field(init=False)
    name: str = None
    fullname: str = None
    nickname: str = None
    addresses: List[Address] = field(default_factory=list)

@dataclass
class Address:
    id: int = field(init=False)
    user_id: int = field(init=False)
    email_address: str = None

metadata_obj = MetaData()

user = Table(
    "user",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    Column("name", String(50)),
    Column("fullname", String(50)),
    Column("nickname", String(12)),
)

address = Table(
    "address",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    Column("user_id", Integer, ForeignKey("user.id")),
    Column("email_address", String(50)),
)

mapper_registry.map_imperatively(
    User,
    user,
    properties={
        "addresses": relationship(Address, backref="user", order_by=address.c.id),
    },
)

mapper_registry.map_imperatively(Address, address)
```

使用此映射样式时，请注意 Mapping pre-existing dataclasses using Declarative With Imperative Table 中提到的相同警告。## 对现有 attrs 类应用 ORM 映射

[attrs](https://pypi.org/project/attrs/) 库是一个流行的第三方库，提供了类似 dataclasses 的功能，并提供了许多普通 dataclasses 中没有的附加功能。

使用 [attrs](https://pypi.org/project/attrs/) 扩展的类使用 `@define` 装饰器。该装饰器启动一个过程来扫描类以定义类的行为的属性，然后用于生成方法、文档和注释。

SQLAlchemy ORM 支持使用 **Declarative with Imperative Table** 或 **Imperative** 映射来映射 [attrs](https://pypi.org/project/attrs/) 类。这两种样式的一般形式与用于 dataclasses 的 Mapping pre-existing dataclasses using Declarative-style fields 和 Mapping pre-existing dataclasses using Declarative With Imperative Table 映射形式完全等效，其中 dataclasses 或 attrs 使用的内联属性指令保持不变，并且 SQLAlchemy 的面向表的仪器化在运行时应用。

[attrs](https://pypi.org/project/attrs/) 的 `@define` 装饰器默认用基于新 __slots__ 的类替换带注释的类，这是不支持的。当使用旧样式注释 `@attr.s` 或使用 `define(slots=False)` 时，类不会被替换。此外，attrs 在装饰器运行后移除其自己的类绑定属性，以便 SQLAlchemy 的映射过程接管这些属性而不会出现任何问题。`@attr.s` 和 `@define(slots=False)` 这两个装饰器都与 SQLAlchemy 兼容。

### 使用 Declarative “Imperative Table” 映射 attrs

在“声明式与命令式表”风格中，`Table` 对象与声明式类内联声明。首先将 `@define` 装饰器应用于类，然后将`registry.mapped()` 装饰器应用于类：

```py
from __future__ import annotations

from typing import List
from typing import Optional

from attrs import define
from sqlalchemy import Column
from sqlalchemy import ForeignKey
from sqlalchemy import Integer
from sqlalchemy import MetaData
from sqlalchemy import String
from sqlalchemy import Table
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import registry
from sqlalchemy.orm import relationship

mapper_registry = registry()

@mapper_registry.mapped
@define(slots=False)
class User:
    __table__ = Table(
        "user",
        mapper_registry.metadata,
        Column("id", Integer, primary_key=True),
        Column("name", String(50)),
        Column("FullName", String(50), key="fullname"),
        Column("nickname", String(12)),
    )
    id: Mapped[int]
    name: Mapped[str]
    fullname: Mapped[str]
    nickname: Mapped[str]
    addresses: Mapped[List[Address]]

    __mapper_args__ = {  # type: ignore
        "properties": {
            "addresses": relationship("Address"),
        }
    }

@mapper_registry.mapped
@define(slots=False)
class Address:
    __table__ = Table(
        "address",
        mapper_registry.metadata,
        Column("id", Integer, primary_key=True),
        Column("user_id", Integer, ForeignKey("user.id")),
        Column("email_address", String(50)),
    )
    id: Mapped[int]
    user_id: Mapped[int]
    email_address: Mapped[Optional[str]]
```

注意

`attrs`的`slots=True`选项，该选项在映射类上启用`__slots__`，不能与 SQLAlchemy 映射一起使用，除非完全实现了替代的属性仪器化，因为映射类通常依赖于对`__dict__`的直接访问来存储状态。当此选项存在时，行为是未定义的。

### 使用命令式映射映射 attrs

就像对待 dataclasses 一样，我们可以利用`registry.map_imperatively()`来将现有的 `attrs` 类映射为：

```py
from __future__ import annotations

from typing import List

from attrs import define
from sqlalchemy import Column
from sqlalchemy import ForeignKey
from sqlalchemy import Integer
from sqlalchemy import MetaData
from sqlalchemy import String
from sqlalchemy import Table
from sqlalchemy.orm import registry
from sqlalchemy.orm import relationship

mapper_registry = registry()

@define(slots=False)
class User:
    id: int
    name: str
    fullname: str
    nickname: str
    addresses: List[Address]

@define(slots=False)
class Address:
    id: int
    user_id: int
    email_address: Optional[str]

metadata_obj = MetaData()

user = Table(
    "user",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    Column("name", String(50)),
    Column("fullname", String(50)),
    Column("nickname", String(12)),
)

address = Table(
    "address",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    Column("user_id", Integer, ForeignKey("user.id")),
    Column("email_address", String(50)),
)

mapper_registry.map_imperatively(
    User,
    user,
    properties={
        "addresses": relationship(Address, backref="user", order_by=address.c.id),
    },
)

mapper_registry.map_imperatively(Address, address)
```

上述形式等同于之前使用声明式与命令式表的示例。## 声明式数据类映射

SQLAlchemy 带注释的声明表映射可以通过附加的 mixin 类或装饰器指令来增强，这将在映射完成后的声明过程中添加一个额外的步骤，将映射的类**原地**转换为 Python [dataclass](https://docs.python.org/3/library/dataclasses.html)，然后完成应用于类的 ORM 特定仪器化过程。这提供的最突出的行为增强是生成具有对位置和关键字参数的细粒度控制的`__init__()`方法，有或没有默认值，以及生成诸如`__repr__()`和`__eq__()`之类的方法。

从[**PEP 484**](https://peps.python.org/pep-0484/)的类型学角度来看，该类被认为具有 Dataclass 特定的行为，最值得注意的是通过利用[**PEP 681**](https://peps.python.org/pep-0681/)“Dataclass Transforms”，这允许类型工具将该类视为明确使用`@dataclasses.dataclass`装饰器装饰的类。

注意

截至**2023 年 4 月 4 日**，在类型工具中支持[**PEP 681**](https://peps.python.org/pep-0681/)的情况是有限的，并且目前已知被[Pyright](https://github.com/microsoft/pyright)以及**1.2 版**的[Mypy](https://mypy.readthedocs.io/en/stable/)支持。请注意，Mypy 1.1.1 引入了[**PEP 681**](https://peps.python.org/pep-0681/)支持，但未正确适应 Python 描述符，这将导致在使用 SQLAlchemy 的 ORM 映射方案时出现错误。

另请参阅

[`peps.python.org/pep-0681/#the-dataclass-transform-decorator`](https://peps.python.org/pep-0681/#the-dataclass-transform-decorator) - 背景介绍了像 SQLAlchemy 这样的库如何启用 [**PEP 681**](https://peps.python.org/pep-0681/) 支持

数据类转换可以通过将 `MappedAsDataclass` mixin 添加到 `DeclarativeBase` 类层次结构中的任何声明性类来实现，或者通过使用 `registry.mapped_as_dataclass()` 类装饰器来进行装饰映射。

`MappedAsDataclass` mixin 可以应用于声明性的 `Base` 类或任何超类，如下例所示：

```py
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import MappedAsDataclass

class Base(MappedAsDataclass, DeclarativeBase):
  """subclasses will be converted to dataclasses"""

class User(Base):
    __tablename__ = "user_account"

    id: Mapped[int] = mapped_column(init=False, primary_key=True)
    name: Mapped[str]
```

也可以直接应用于从声明性基类扩展的类：

```py
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import MappedAsDataclass

class Base(DeclarativeBase):
    pass

class User(MappedAsDataclass, Base):
  """User class will be converted to a dataclass"""

    __tablename__ = "user_account"

    id: Mapped[int] = mapped_column(init=False, primary_key=True)
    name: Mapped[str]
```

使用装饰器形式时，仅支持 `registry.mapped_as_dataclass()` 装饰器：

```py
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import registry

reg = registry()

@reg.mapped_as_dataclass
class User:
    __tablename__ = "user_account"

    id: Mapped[int] = mapped_column(init=False, primary_key=True)
    name: Mapped[str]
```

### 类级别的功能配置

数据类功能的支持是部分的。目前支持的是`init`、`repr`、`eq`、`order`和`unsafe_hash`功能，`match_args` 和 `kw_only`在 Python 3.10+ 上支持。目前不支持的是 `frozen` 和 `slots` 功能。

使用带有 `MappedAsDataclass` 的 mixin 类形式时，类配置参数作为类级别参数传递：

```py
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import MappedAsDataclass

class Base(DeclarativeBase):
    pass

class User(MappedAsDataclass, Base, repr=False, unsafe_hash=True):
  """User class will be converted to a dataclass"""

    __tablename__ = "user_account"

    id: Mapped[int] = mapped_column(init=False, primary_key=True)
    name: Mapped[str]
```

使用带有 `registry.mapped_as_dataclass()` 的装饰器形式时，类配置参数直接传递给装饰器：

```py
from sqlalchemy.orm import registry
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column

reg = registry()

@reg.mapped_as_dataclass(unsafe_hash=True)
class User:
  """User class will be converted to a dataclass"""

    __tablename__ = "user_account"

    id: Mapped[int] = mapped_column(init=False, primary_key=True)
    name: Mapped[str]
```

有关数据类选项的背景，请参阅 [dataclasses](https://docs.python.org/3/library/dataclasses.html) 文档中的 [@dataclasses.dataclass](https://docs.python.org/3/library/dataclasses.html#dataclasses.dataclass).

### 属性配置

SQLAlchemy 原生数据类与普通数据类不同之处在于，要映射的属性在所有情况下都是使用 `Mapped` 泛型注释容器描述的。映射遵循与 声明性表格与 mapped_column() 中记录的相同形式，所有 `mapped_column()` 和 `Mapped` 的功能都得到支持。

此外，ORM 属性配置构造，包括`mapped_column()`，`relationship()`和`composite()` 支持**每个属性字段选项**，包括`init`，`default`，`default_factory`和`repr`。这些参数的名称固定为[**PEP 681**](https://peps.python.org/pep-0681/)中指定的名称。功能等同于 dataclasses：

+   `init`，如`mapped_column.init`，`relationship.init`，如果为 False，则表示该字段不应该是`__init__()`方法的一部分

+   `default`，如`mapped_column.default`，`relationship.default`，表示字段的默认值，可以作为关键字参数在`__init__()`方法中传递。

+   `default_factory`，如`mapped_column.default_factory`，`relationship.default_factory`，表示一个可调用函数，如果未明确传递给`__init__()`方法，则会调用它生成新的默认值。

+   `repr` 默认为 True，表示该字段应该是生成的`__repr__()`方法的一部分

与 dataclasses 的另一个关键区别是，属性的默认值**必须**使用 ORM 构造函数的`default`参数进行配置，例如`mapped_column(default=None)`。不支持类似 dataclass 语法的语法，该语法接受简单的 Python 值作为默认值，而不使用`@dataclases.field()`。

作为使用`mapped_column()`的示例，下面的映射将生成一个仅接受字段`name`和`fullname`的`__init__()`方法，其中`name`是必需的，可以按位置传递，而`fullname`是可选的。我们预期会从数据库生成`id`字段，根本不是构造函数的一部分：

```py
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import registry

reg = registry()

@reg.mapped_as_dataclass
class User:
    __tablename__ = "user_account"

    id: Mapped[int] = mapped_column(init=False, primary_key=True)
    name: Mapped[str]
    fullname: Mapped[str] = mapped_column(default=None)

# 'fullname' is optional keyword argument
u1 = User("name")
```

#### 列默认值

为了适应 `default` 参数与现有 `Column.default` 参数之间的名称重叠，`mapped_column()` 构造通过添加一个新参数 `mapped_column.insert_default` 来消除这两个名称的歧义，该参数将直接填充到 `Column.default` 参数中，并且独立于在 `mapped_column.default` 上设置的内容，后者始终用于数据类配置。例如，配置一个带有 `func.utc_timestamp()` SQL 函数作为 `Column.default` 的日期时间列，但在构造函数中该参数是可选的：

```py
from datetime import datetime

from sqlalchemy import func
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import registry

reg = registry()

@reg.mapped_as_dataclass
class User:
    __tablename__ = "user_account"

    id: Mapped[int] = mapped_column(init=False, primary_key=True)
    created_at: Mapped[datetime] = mapped_column(
        insert_default=func.utc_timestamp(), default=None
    )
```

使用上述映射，对于未传递任何 `created_at` 参数的新 `User` 对象的 `INSERT` 如下进行：

```py
>>> with Session(e) as session:
...     session.add(User())
...     session.commit()
BEGIN  (implicit)
INSERT  INTO  user_account  (created_at)  VALUES  (utc_timestamp())
[generated  in  0.00010s]  ()
COMMIT 
```

#### 与 Annotated 的集成

将整个列声明映射到 Python 类型 中介绍的方法演示了如何使用 [**PEP 593**](https://peps.python.org/pep-0593/) 的 `Annotated` 对象来打包整个 `mapped_column()` 构造以便重用。该功能支持使用数据类特性。然而，该特性的一个方面在与类型工具一起使用时需要一种解决方法，即必须将 [**PEP 681**](https://peps.python.org/pep-0681/) 中的参数 `init`、`default`、`repr` 和 `default_factory` **必须** 放在右侧，并打包到一个明确的 `mapped_column()` 构造中，以便类型工具正确解释属性。例如，下面的方法在运行时完全正常，但是类型工具将认为 `User()` 构造无效，因为它们看不到 `init=False` 参数：

```py
from typing import Annotated

from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import registry

# typing tools will ignore init=False here
intpk = Annotated[int, mapped_column(init=False, primary_key=True)]

reg = registry()

@reg.mapped_as_dataclass
class User:
    __tablename__ = "user_account"
    id: Mapped[intpk]

# typing error: Argument missing for parameter "id"
u1 = User()
```

相反，`mapped_column()` 必须出现在右侧，并且使用明确设置的 `mapped_column.init`；其他参数可以保留在 `Annotated` 结构内：

```py
from typing import Annotated

from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import registry

intpk = Annotated[int, mapped_column(primary_key=True)]

reg = registry()

@reg.mapped_as_dataclass
class User:
    __tablename__ = "user_account"

    # init=False and other pep-681 arguments must be inline
    id: Mapped[intpk] = mapped_column(init=False)

u1 = User()
```

### 使用混入和抽象超类

在`MappedAsDataclass`映射类中使用的任何混入或基类，其中包括`Mapped`属性，必须本身是 `MappedAsDataclass`层次结构的一部分，例如下面的示例使用混入：

```py
class Mixin(MappedAsDataclass):
    create_user: Mapped[int] = mapped_column()
    update_user: Mapped[Optional[int]] = mapped_column(default=None, init=False)

class Base(DeclarativeBase, MappedAsDataclass):
    pass

class User(Base, Mixin):
    __tablename__ = "sys_user"

    uid: Mapped[str] = mapped_column(
        String(50), init=False, default_factory=uuid4, primary_key=True
    )
    username: Mapped[str] = mapped_column()
    email: Mapped[str] = mapped_column()
```

支持 [**PEP 681**](https://peps.python.org/pep-0681/) 的 Python 类型检查器将否则不会认为来自非数据类混入的属性属于数据类的一部分。

自 2.0.8 版开始已弃用：在`MappedAsDataclass`或`registry.mapped_as_dataclass()`层次结构中使用混入和抽象基类，这些结构本身不是数据类已被弃用，因为这些字段不被 [**PEP 681**](https://peps.python.org/pep-0681/) 视为数据类的一部分。对于这种情况会发出警告，稍后将成为错误。

另请参见

将<cls>转换为数据类时，属性来自非数据类的超类<cls>。 - 关于原因的背景

### 关系配置

`Mapped` 注释与 `relationship()` 结合使用的方式与基本关系模式中描述的方式相同。当指定基于集合的 `relationship()` 作为可选关键字参数时，必须传递 `relationship.default_factory` 参数，并且它必须引用要使用的集合类。如果默认值为 `None`，则多对一和标量对象引用可以使用 `relationship.default`:

```py
from typing import List

from sqlalchemy import ForeignKey
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import registry
from sqlalchemy.orm import relationship

reg = registry()

@reg.mapped_as_dataclass
class Parent:
    __tablename__ = "parent"
    id: Mapped[int] = mapped_column(primary_key=True)
    children: Mapped[List["Child"]] = relationship(
        default_factory=list, back_populates="parent"
    )

@reg.mapped_as_dataclass
class Child:
    __tablename__ = "child"
    id: Mapped[int] = mapped_column(primary_key=True)
    parent_id: Mapped[int] = mapped_column(ForeignKey("parent.id"))
    parent: Mapped["Parent"] = relationship(default=None)
```

上述映射将在构造新的`Parent()`对象时，如果没有传递`children`参数，则为`Parent.children`生成一个空列表，并且在构造新的`Child()`对象时，如果没有传递`parent`参数，则为`Child.parent`生成一个`None`值。

虽然`relationship.default_factory`可以从`relationship()`自身的给定集合类中自动推导出来，但这将与数据类的兼容性破坏，因为`relationship.default_factory`或`relationship.default`的存在决定了参数在渲染到`__init__()`方法时是必需还是可选。

### 使用非映射数据类字段

当使用声明性数据类时，类上也可以使用非映射字段，这些字段将成为数据类构造过程的一部分，但不会被映射。任何未使用`Mapped`的字段都将被映射过程忽略。在下面的示例中，字段`ctrl_one`和`ctrl_two`将成为对象的实例级状态的一部分，但不会由 ORM 持久化：

```py
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import registry

reg = registry()

@reg.mapped_as_dataclass
class Data:
    __tablename__ = "data"

    id: Mapped[int] = mapped_column(init=False, primary_key=True)
    status: Mapped[str]

    ctrl_one: Optional[str] = None
    ctrl_two: Optional[str] = None
```

上面的`Data`实例可以创建为：

```py
d1 = Data(status="s1", ctrl_one="ctrl1", ctrl_two="ctrl2")
```

一个更实际的例子可能是结合数据类的`InitVar`特性和`__post_init__()`特性来接收仅初始化字段，这些字段可用于组成持久化数据。在下面的示例中，`User`类使用`id`、`name`和`password_hash`作为映射特性，但使用仅初始化的`password`和`repeat_password`字段表示用户创建过程（注意：要运行此示例，请将函数`your_crypt_function_here()`替换为第三方加密函数，如[bcrypt](https://pypi.org/project/bcrypt/)或[argon2-cffi](https://pypi.org/project/argon2-cffi/)）：

```py
from dataclasses import InitVar
from typing import Optional

from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import registry

reg = registry()

@reg.mapped_as_dataclass
class User:
    __tablename__ = "user_account"

    id: Mapped[int] = mapped_column(init=False, primary_key=True)
    name: Mapped[str]

    password: InitVar[str]
    repeat_password: InitVar[str]

    password_hash: Mapped[str] = mapped_column(init=False, nullable=False)

    def __post_init__(self, password: str, repeat_password: str):
        if password != repeat_password:
            raise ValueError("passwords do not match")

        self.password_hash = your_crypt_function_here(password)
```

上述对象使用参数`password`和`repeat_password`创建，这些参数被立即消耗，以便生成`password_hash`变量：

```py
>>> u1 = User(name="some_user", password="xyz", repeat_password="xyz")
>>> u1.password_hash
'$6$9ppc... (example crypted string....)'
```

从版本 2.0.0rc1 开始更改：当使用`registry.mapped_as_dataclass()`或`MappedAsDataclass`时，可以包括不包括`Mapped`注释的字段，这些字段将被视为生成的数据类的一部分，但不会被映射，无需指定`__allow_unmapped__`类属性。以前的 2.0 beta 版本将要求明确存在此属性，即使此属性的目的仅是允许旧版 ORM 类型映射继续运行。### 与诸如 Pydantic 等替代数据类提供者集成

警告

Pydantic 的 dataclass 层与 SQLAlchemy 的类仪器化不完全兼容，需要额外的内部更改，许多功能，例如相关集合，可能无法正常工作。

为了与 Pydantic 兼容，请考虑使用[SQLModel](https://sqlmodel.tiangolo.com) ORM，该 ORM 基于 SQLAlchemy ORM 构建，使用 Pydantic，其中包括明确解决这些不兼容性的特殊实现细节。

SQLAlchemy 的`MappedAsDataclass`类和`registry.mapped_as_dataclass()`方法调用直接进入 Python 标准库 `dataclasses.dataclass` 类装饰器，经过类的声明性映射处理后。此函数调用可以通过`MappedAsDataclass`作为类关键字参数以及`registry.mapped_as_dataclass()`接受的`dataclass_callable`参数交换为备用 dataclasses 提供程序，例如 Pydantic 的提供程序：

```py
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import MappedAsDataclass
from sqlalchemy.orm import registry

class Base(
    MappedAsDataclass,
    DeclarativeBase,
    dataclass_callable=pydantic.dataclasses.dataclass,
):
    pass

class User(Base):
    __tablename__ = "user"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
```

上述 `User` 类将被应用为一个数据类，使用 Pydantic 的 `pydantic.dataclasses.dataclasses` 可调用。该过程对映射类以及从`MappedAsDataclass`扩展的混合类或直接应用了`registry.mapped_as_dataclass()`的类都可用。

新版本 2.0.4 中：为`MappedAsDataclass`和`registry.mapped_as_dataclass()`添加了 `dataclass_callable` 类和方法参数，并调整了一些数据类内部，以适应更严格的数据类函数，例如 Pydantic 的函数。

### 类级特性配置

对 dataclasses 特性的支持是部分的。当前支持的特性包括 `init`、`repr`、`eq`、`order` 和 `unsafe_hash` 特性，`match_args` 和 `kw_only` 在 Python 3.10+ 上受支持。当前不支持的特性包括 `frozen` 和 `slots` 特性。

当使用与`MappedAsDataclass`配合使用的混合类形式时，类配置参数被作为类级参数传递：

```py
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import MappedAsDataclass

class Base(DeclarativeBase):
    pass

class User(MappedAsDataclass, Base, repr=False, unsafe_hash=True):
  """User class will be converted to a dataclass"""

    __tablename__ = "user_account"

    id: Mapped[int] = mapped_column(init=False, primary_key=True)
    name: Mapped[str]
```

当使用装饰器形式与`registry.mapped_as_dataclass()`一起使用时，类配置参数直接传递给装饰器：

```py
from sqlalchemy.orm import registry
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column

reg = registry()

@reg.mapped_as_dataclass(unsafe_hash=True)
class User:
  """User class will be converted to a dataclass"""

    __tablename__ = "user_account"

    id: Mapped[int] = mapped_column(init=False, primary_key=True)
    name: Mapped[str]
```

有关 dataclass 类选项的背景，请参阅[dataclasses](https://docs.python.org/3/library/dataclasses.html)文档中的[@dataclasses.dataclass](https://docs.python.org/3/library/dataclasses.html#dataclasses.dataclass)。

### 属性配置

SQLAlchemy 本地 dataclasses 与普通 dataclasses 不同之处在于，要映射的属性在所有情况下都使用`Mapped`通用注释容器描述。映射遵循与使用 mapped_column() 声明的声明性表格文档中记录的相同形式，支持所有`mapped_column()`和`Mapped`的功能。

此外，ORM 属性配置构造包括`mapped_column()`，`relationship()`和`composite()`支持**每个属性字段选项**，包括`init`，`default`，`default_factory`和`repr`。这些参数的名称如[**PEP 681**](https://peps.python.org/pep-0681/)中指定的那样固定。功能与 dataclasses 等效：

+   `init`，比如`mapped_column.init`，`relationship.init`，如果为 False，表示该字段不应该是`__init__()`方法的一部分。

+   `default`，比如`mapped_column.default`，`relationship.default`，表示字段的默认值，作为`__init__()`方法的关键字参数传递。

+   `default_factory`，比如`mapped_column.default_factory`，`relationship.default_factory`，表示一个可调用函数，将被调用以生成一个新的默认值，如果参数未明确传递给`__init__()`方法。

+   `repr` 默认为 True，表示该字段应该是生成的`__repr__()`方法的一部分。

与 dataclasses 的另一个关键区别是，属性的默认值**必须**使用 ORM 构造函数的 `default` 参数进行配置，例如 `mapped_column(default=None)`。不支持接受简单 Python 值作为默认值的类似 dataclass 语法，而不使用 `@dataclases.field()`。

作为使用 `mapped_column()` 的示例，下面的映射将生成一个 `__init__()` 方法，该方法仅接受字段 `name` 和 `fullname`，其中 `name` 是必需的，并且可以按位置传递，而 `fullname` 是可选的。我们预期的由数据库生成的 `id` 字段根本不是构造函数的一部分：

```py
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import registry

reg = registry()

@reg.mapped_as_dataclass
class User:
    __tablename__ = "user_account"

    id: Mapped[int] = mapped_column(init=False, primary_key=True)
    name: Mapped[str]
    fullname: Mapped[str] = mapped_column(default=None)

# 'fullname' is optional keyword argument
u1 = User("name")
```

#### 列默认值

为了适应 `default` 参数与 `Column.default` 构造函数现有参数的名称重叠，`mapped_column()` 构造函数通过添加一个新参数 `mapped_column.insert_default` 来消除两个名称之间的歧义，该参数将直接填充到 `Column.default` 参数中，独立于 `mapped_column.default` 上的设置，后者始终用于数据类配置。例如，要配置一个 datetime 列，其中 `Column.default` 设置为 `func.utc_timestamp()` SQL 函数，但构造函数中该参数是可选的：

```py
from datetime import datetime

from sqlalchemy import func
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import registry

reg = registry()

@reg.mapped_as_dataclass
class User:
    __tablename__ = "user_account"

    id: Mapped[int] = mapped_column(init=False, primary_key=True)
    created_at: Mapped[datetime] = mapped_column(
        insert_default=func.utc_timestamp(), default=None
    )
```

使用上述映射，对于一个新的 `User` 对象的 `INSERT`，如果没有传递 `created_at` 的参数，操作将如下进行：

```py
>>> with Session(e) as session:
...     session.add(User())
...     session.commit()
BEGIN  (implicit)
INSERT  INTO  user_account  (created_at)  VALUES  (utc_timestamp())
[generated  in  0.00010s]  ()
COMMIT 
```

#### 与 Annotated 集成

在将整个列声明映射到 Python 类型介绍的方法说明了如何使用 [**PEP 593**](https://peps.python.org/pep-0593/) 中的 `Annotated` 对象将整个 `mapped_column()` 构建打包以供重复使用。此功能支持 dataclasses 功能。然而，该功能的一个方面在使用类型工具时需要一个解决方法，即 [**PEP 681**](https://peps.python.org/pep-0681/) 特定参数 `init`、`default`、`repr` 和 `default_factory` **必须** 在右侧，被打包到显式的 `mapped_column()` 构建中，以便类型工具正确解释属性。例如，下面的方法在运行时将完美地工作，但是类型工具会认为 `User()` 构造无效，因为它们没有看到 `init=False` 参数：

```py
from typing import Annotated

from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import registry

# typing tools will ignore init=False here
intpk = Annotated[int, mapped_column(init=False, primary_key=True)]

reg = registry()

@reg.mapped_as_dataclass
class User:
    __tablename__ = "user_account"
    id: Mapped[intpk]

# typing error: Argument missing for parameter "id"
u1 = User()
```

相反，`mapped_column()` 必须在右侧也存在，并且必须显式设置 `mapped_column.init`；其他参数可以保留在 `Annotated` 结构中：

```py
from typing import Annotated

from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import registry

intpk = Annotated[int, mapped_column(primary_key=True)]

reg = registry()

@reg.mapped_as_dataclass
class User:
    __tablename__ = "user_account"

    # init=False and other pep-681 arguments must be inline
    id: Mapped[intpk] = mapped_column(init=False)

u1 = User()
```

#### 列默认值

为了适应 `default` 参数与 `Column.default` 构造函数的现有参数的名称重叠，`mapped_column()` 构造函数通过添加一个新参数 `mapped_column.insert_default` 来消除这两个名称之间的歧义，该参数将直接填充到 `Column.default` 参数中，而与 `mapped_column.default` 设置无关，后者始终用于 dataclasses 配置。例如，要配置一个 datetime 列，并将 `Column.default` 设置为 `func.utc_timestamp()` SQL 函数，但构造函数中该参数是可选的：

```py
from datetime import datetime

from sqlalchemy import func
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import registry

reg = registry()

@reg.mapped_as_dataclass
class User:
    __tablename__ = "user_account"

    id: Mapped[int] = mapped_column(init=False, primary_key=True)
    created_at: Mapped[datetime] = mapped_column(
        insert_default=func.utc_timestamp(), default=None
    )
```

在上述映射中，当未传递 `created_at` 参数的情况下，对新的 `User` 对象进行 `INSERT` 操作如下进行：

```py
>>> with Session(e) as session:
...     session.add(User())
...     session.commit()
BEGIN  (implicit)
INSERT  INTO  user_account  (created_at)  VALUES  (utc_timestamp())
[generated  in  0.00010s]  ()
COMMIT 
```

#### 与 `Annotated` 的集成

在将整个列声明映射到 Python 类型中介绍的方法说明了如何使用[**PEP 593**](https://peps.python.org/pep-0593/) `Annotated`对象来打包整个`mapped_column()`结构以供重用。该功能与数据类功能一起使用。然而，该功能的一个方面在使用类型工具时需要一个解决方法，即[**PEP 681**](https://peps.python.org/pep-0681/)特定的参数`init`、`default`、`repr`和`default_factory` **必须** 在右侧，打包到显式的`mapped_column()`构造中，以便类型工具正确解释属性。例如，下面的方法在运行时将完美运行，但是类型工具将认为`User()`构造无效，因为它们看不到`init=False`参数的存在：

```py
from typing import Annotated

from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import registry

# typing tools will ignore init=False here
intpk = Annotated[int, mapped_column(init=False, primary_key=True)]

reg = registry()

@reg.mapped_as_dataclass
class User:
    __tablename__ = "user_account"
    id: Mapped[intpk]

# typing error: Argument missing for parameter "id"
u1 = User()
```

相反，`mapped_column()`也必须出现在右侧，并在`Annotated`结构中包含对`mapped_column.init`的显式设置；其他参数可以保留在`Annotated`结构中：

```py
from typing import Annotated

from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import registry

intpk = Annotated[int, mapped_column(primary_key=True)]

reg = registry()

@reg.mapped_as_dataclass
class User:
    __tablename__ = "user_account"

    # init=False and other pep-681 arguments must be inline
    id: Mapped[intpk] = mapped_column(init=False)

u1 = User()
```

### 使用混入和抽象超类

在`MappedAsDataclass`映射类中使用的任何混入或基类，其中包括`Mapped`属性，必须本身是`MappedAsDataclass`层次结构的一部分，例如，在下面的示例中使用混入：

```py
class Mixin(MappedAsDataclass):
    create_user: Mapped[int] = mapped_column()
    update_user: Mapped[Optional[int]] = mapped_column(default=None, init=False)

class Base(DeclarativeBase, MappedAsDataclass):
    pass

class User(Base, Mixin):
    __tablename__ = "sys_user"

    uid: Mapped[str] = mapped_column(
        String(50), init=False, default_factory=uuid4, primary_key=True
    )
    username: Mapped[str] = mapped_column()
    email: Mapped[str] = mapped_column()
```

Python 类型检查器，支持[**PEP 681**](https://peps.python.org/pep-0681/)，否则将不考虑非数据类混入的属性作为数据类的一部分。

从版本 2.0.8 开始已弃用：在`MappedAsDataclass`或`registry.mapped_as_dataclass()`层次结构中使用混入和抽象基类，这些类本身不是数据类，这是不推荐的，因为这些字段不被[**PEP 681**](https://peps.python.org/pep-0681/)支持为数据类的一部分。对于这种情况会发出警告，以后会成为错误。

另请参阅

当将<cls>转换为数据类时，属性源自不是数据类的超类<cls>。 - 关于原因的背景

### 关系配置

`Mapped`注解与`relationship()`结合使用的方式与基本关系模式中描述的相同。当将基于集合的`relationship()`指定为可选关键字参数时，必须传递`relationship.default_factory`参数，并且它必须引用要使用的集合类。如果默认值是`None`，则多对一和标量对象引用可以使用`relationship.default`:

```py
from typing import List

from sqlalchemy import ForeignKey
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import registry
from sqlalchemy.orm import relationship

reg = registry()

@reg.mapped_as_dataclass
class Parent:
    __tablename__ = "parent"
    id: Mapped[int] = mapped_column(primary_key=True)
    children: Mapped[List["Child"]] = relationship(
        default_factory=list, back_populates="parent"
    )

@reg.mapped_as_dataclass
class Child:
    __tablename__ = "child"
    id: Mapped[int] = mapped_column(primary_key=True)
    parent_id: Mapped[int] = mapped_column(ForeignKey("parent.id"))
    parent: Mapped["Parent"] = relationship(default=None)
```

上述映射将在构造新的`Parent()`对象时为`Parent.children`生成一个空列表，当构造新的`Child()`对象时，如果不传递`parent`，则`Child.parent`将生成一个`None`值。

虽然`relationship.default_factory`可以自动从`relationship()`本身的给定集合类中派生，但这会与数据类兼容性破坏，因为`relationship.default_factory`或`relationship.default`的存在决定了参数在渲染为`__init__()`方法时是必需还是可选。

### 使用非映射数据类字段

当使用声明性数据类时，也可以在类上使用非映射字段，这些字段将成为数据类构造过程的一部分，但不会被映射。任何不使用`Mapped`的字段都将被映射过程忽略。在下面的示例中，字段`ctrl_one`和`ctrl_two`将成为对象的实例级状态的一部分，但不会被 ORM 持久化：

```py
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import registry

reg = registry()

@reg.mapped_as_dataclass
class Data:
    __tablename__ = "data"

    id: Mapped[int] = mapped_column(init=False, primary_key=True)
    status: Mapped[str]

    ctrl_one: Optional[str] = None
    ctrl_two: Optional[str] = None
```

上面的`Data`实例可以创建为：

```py
d1 = Data(status="s1", ctrl_one="ctrl1", ctrl_two="ctrl2")
```

更真实的示例可能是结合 Dataclasses 的`InitVar`特性和`__post_init__()`特性来使用仅初始化的字段，这些字段可以用于组合持久化数据。在下面的示例中，`User`类使用`id`、`name`和`password_hash`作为映射特性声明，但使用了仅初始化的`password`和`repeat_password`字段来表示用户创建过程（注意：要运行此示例，请将函数`your_crypt_function_here()`替换为第三方加密函数，如[bcrypt](https://pypi.org/project/bcrypt/)或[argon2-cffi](https://pypi.org/project/argon2-cffi/)）：

```py
from dataclasses import InitVar
from typing import Optional

from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import registry

reg = registry()

@reg.mapped_as_dataclass
class User:
    __tablename__ = "user_account"

    id: Mapped[int] = mapped_column(init=False, primary_key=True)
    name: Mapped[str]

    password: InitVar[str]
    repeat_password: InitVar[str]

    password_hash: Mapped[str] = mapped_column(init=False, nullable=False)

    def __post_init__(self, password: str, repeat_password: str):
        if password != repeat_password:
            raise ValueError("passwords do not match")

        self.password_hash = your_crypt_function_here(password)
```

上述对象使用了参数`password`和`repeat_password`，这些参数首先被使用，以便生成`password_hash`变量：

```py
>>> u1 = User(name="some_user", password="xyz", repeat_password="xyz")
>>> u1.password_hash
'$6$9ppc... (example crypted string....)'
```

从版本 2.0.0rc1 开始更改：当使用`registry.mapped_as_dataclass()`或`MappedAsDataclass`时，可以包括不包含`Mapped`注释的字段，这些字段将被视为生成的 dataclass 的一部分，但不会被映射，无需另外指示`__allow_unmapped__`类属性。先前的 2.0 beta 版本将要求明确包含此属性，即使此属性的目的仅是允许旧的 ORM 类型映射继续工作。

### 与 Pydantic 等替代 Dataclass 提供程序集成

警告

Pydantic 的 dataclass 层与 SQLAlchemy 的类仪器化**不完全兼容**，需要额外的内部更改，许多功能，如相关集合，可能无法正常工作。

为了与 Pydantic 兼容，请考虑使用[SQLModel](https://sqlmodel.tiangolo.com) ORM，它是在 SQLAlchemy ORM 的基础上构建的 Pydantic，其中包含了专门解决这些不兼容性的实现细节。

SQLAlchemy 的`MappedAsDataclass`类和`registry.mapped_as_dataclass()`方法调用直接进入 Python 标准库的`dataclasses.dataclass`类装饰器中，声明性映射过程应用到类之后。此函数调用可以用 Pydantic 等替代数据类提供程序替换，使用`MappedAsDataclass`作为类关键字参数接受的`dataclass_callable`参数，以及`registry.mapped_as_dataclass()`同样接受的参数：

```py
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import MappedAsDataclass
from sqlalchemy.orm import registry

class Base(
    MappedAsDataclass,
    DeclarativeBase,
    dataclass_callable=pydantic.dataclasses.dataclass,
):
    pass

class User(Base):
    __tablename__ = "user"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
```

上述的`User`类将被应用为一个数据类，使用 Pydantic 的`pydantic.dataclasses.dataclasses`可调用。此过程对于映射类和扩展自`MappedAsDataclass`或直接应用了`registry.mapped_as_dataclass()`的混入类都可用。

2.0.4 版本中的新内容：为`MappedAsDataclass`和`registry.mapped_as_dataclass()`添加了`dataclass_callable`类和方法参数，并调整了一些数据类内部，以适应更严格的数据类函数，例如 Pydantic 的函数。

## 将 ORM 映射应用于现有数据类（旧数据类使用）

遗留特性

这里描述的方法已被 SQLAlchemy 2.0 系列中的声明性数据类映射特性取代。该特性的新版本建立在首次在 1.4 版本中添加的数据类支持之上，该支持在本节中描述。

要映射现有的数据类，不能直接使用 SQLAlchemy 的“内联”声明性指令；ORM 指令是使用以下三种技术之一分配的：

+   使用“带命令式表”的方法，要映射的表/列是使用分配给类的`__table__`属性的`Table`对象定义的；关系在`__mapper_args__`字典中定义。使用`registry.mapped()`装饰器映射类。下面是一个示例，在使用带命令式表的方式映射预先存在的数据类中。

+   使用完整的“声明式”，将 Declarative 解释的指令（例如`Column`、`relationship()`）添加到`dataclasses.field()`构造函数的`.metadata`字典中，它们将被声明性过程使用。再次使用`registry.mapped()`装饰器映射类。请参见下面的示例，在使用声明式字段映射预先存在的数据类中。

+   可以使用 `registry.map_imperatively()` 方法将“命令式”映射应用于现有数据类，以完全相同的方式生成映射，如命令式映射中所述。下面在 使用命令式映射预存在的数据类中说明。

通过 SQLAlchemy 将映射应用于数据类的一般过程与普通类的过程相同，但还包括 SQLAlchemy 将检测到的类级别属性，这些属性是数据类声明过程的一部分，并在运行时用通常的 SQLAlchemy ORM 映射属性替换它们。数据类生成的 `__init__` 方法保持不变，所有其他由数据类生成的方法，例如 `__eq__()`、`__repr__()` 等也是如此。

### 使用声明式与命令式表（即混合式声明式）映射预存在的数据类

以下是使用 `@dataclass` 和 使用声明式与命令式表（又名混合式声明式） 进行映射的示例。完整的 `Table` 对象是明确构造的，并分配给 `__table__` 属性。实例字段使用正常的数据类语法进行定义。其他 `MapperProperty` 定义，如 `relationship()`，放置在 __mapper_args__ 类级别字典中，位于 `properties` 键下，与 `Mapper.properties` 参数对应：

```py
from __future__ import annotations

from dataclasses import dataclass, field
from typing import List, Optional

from sqlalchemy import Column, ForeignKey, Integer, String, Table
from sqlalchemy.orm import registry, relationship

mapper_registry = registry()

@mapper_registry.mapped
@dataclass
class User:
    __table__ = Table(
        "user",
        mapper_registry.metadata,
        Column("id", Integer, primary_key=True),
        Column("name", String(50)),
        Column("fullname", String(50)),
        Column("nickname", String(12)),
    )
    id: int = field(init=False)
    name: Optional[str] = None
    fullname: Optional[str] = None
    nickname: Optional[str] = None
    addresses: List[Address] = field(default_factory=list)

    __mapper_args__ = {  # type: ignore
        "properties": {
            "addresses": relationship("Address"),
        }
    }

@mapper_registry.mapped
@dataclass
class Address:
    __table__ = Table(
        "address",
        mapper_registry.metadata,
        Column("id", Integer, primary_key=True),
        Column("user_id", Integer, ForeignKey("user.id")),
        Column("email_address", String(50)),
    )
    id: int = field(init=False)
    user_id: int = field(init=False)
    email_address: Optional[str] = None
```

在上述示例中，`User.id`、`Address.id` 和 `Address.user_id` 属性被定义为 `field(init=False)`。这意味着这些参数不会被添加到 `__init__()` 方法中，但`Session` 仍然能够在从自增或其他默认值生成器刷新时获取它们的值并设置它们。为了允许在构造函数中明确指定它们，它们将被赋予 `None` 的默认值。

要单独声明一个 `relationship()`，需要将其直接指定在 `Mapper.properties` 字典中，该字典本身在 `__mapper_args__` 字典中指定，以便将其传递给 `Mapper` 的构造函数。另一种方法在下一个示例中。

警告

声明一个 dataclass `field()` 设置一个 `default` 与 `init=False` 一起使用不会像完全普通的 dataclass 那样起作用，因为 SQLAlchemy 类的内部机制会用数据类创建过程中设置的默认值替换类上的默认值。使用 `default_factory` 代替。当使用 声明式 Dataclass 映射 时，此适应将自动完成。### 使用声明式字段映射现有数据类

旧版特性

应将此数据类与声明式映射一起使用的方法视为旧版。它将继续受到支持，但是不太可能提供与 声明式 Dataclass 映射 中详细说明的新方法相比的任何优势。

注意**`mapped_column()`在这种用法下不受支持**；应继续使用`Column` 构造函数来声明 `metadata` 字段中的表元数据。

完全声明式方法要求将 `Column` 对象声明为类属性，在使用 dataclasses 时会与 dataclass 级别的属性冲突。结合这些的一种方法是利用 `dataclass.field` 对象上的 `metadata` 属性，在那里可以提供 SQLAlchemy 特定的映射信息。声明式支持在类指定属性 `__sa_dataclass_metadata_key__` 时提取这些参数。这也提供了一种更简洁的方法来指示 `relationship()` 关联：

```py
from __future__ import annotations

from dataclasses import dataclass, field
from typing import List

from sqlalchemy import Column, ForeignKey, Integer, String
from sqlalchemy.orm import registry, relationship

mapper_registry = registry()

@mapper_registry.mapped
@dataclass
class User:
    __tablename__ = "user"

    __sa_dataclass_metadata_key__ = "sa"
    id: int = field(init=False, metadata={"sa": Column(Integer, primary_key=True)})
    name: str = field(default=None, metadata={"sa": Column(String(50))})
    fullname: str = field(default=None, metadata={"sa": Column(String(50))})
    nickname: str = field(default=None, metadata={"sa": Column(String(12))})
    addresses: List[Address] = field(
        default_factory=list, metadata={"sa": relationship("Address")}
    )

@mapper_registry.mapped
@dataclass
class Address:
    __tablename__ = "address"
    __sa_dataclass_metadata_key__ = "sa"
    id: int = field(init=False, metadata={"sa": Column(Integer, primary_key=True)})
    user_id: int = field(init=False, metadata={"sa": Column(ForeignKey("user.id"))})
    email_address: str = field(default=None, metadata={"sa": Column(String(50))})
```

#### 使用声明式 Mixin 与现有数据类

在 使用 Mixins 构建映射层次结构 部分介绍了声明式 Mixin 类。声明式 Mixin 的一个要求是，某些无法轻松复制的构造必须作为可调用对象给出，使用 `declared_attr` 装饰器，例如在 混合关系 中的示例：

```py
class RefTargetMixin:
    @declared_attr
    def target_id(cls):
        return Column("target_id", ForeignKey("target.id"))

    @declared_attr
    def target(cls):
        return relationship("Target")
```

通过在数据类`field()`对象中使用 lambda 来表示`field()`内的 SQLAlchemy 构造支持此形式。使用`declared_attr()`将 lambda 包围起来是可选的。如果我们想要生成上述的`User`类，其中 ORM 字段来自于一个自身是数据类的混合类，形式将是：

```py
@dataclass
class UserMixin:
    __tablename__ = "user"

    __sa_dataclass_metadata_key__ = "sa"

    id: int = field(init=False, metadata={"sa": Column(Integer, primary_key=True)})

    addresses: List[Address] = field(
        default_factory=list, metadata={"sa": lambda: relationship("Address")}
    )

@dataclass
class AddressMixin:
    __tablename__ = "address"
    __sa_dataclass_metadata_key__ = "sa"
    id: int = field(init=False, metadata={"sa": Column(Integer, primary_key=True)})
    user_id: int = field(
        init=False, metadata={"sa": lambda: Column(ForeignKey("user.id"))}
    )
    email_address: str = field(default=None, metadata={"sa": Column(String(50))})

@mapper_registry.mapped
class User(UserMixin):
    pass

@mapper_registry.mapped
class Address(AddressMixin):
    pass
```

新功能在版本 1.4.2 中新增：对“已声明属性”样式的混合属性提供支持，即`relationship()`构造以及带有外键声明的`Column`对象，可用于“声明式表的数据类”样式的映射中。### 使用命令式映射映射预先存在的数据类

如前所述，使用`@dataclass`装饰器设置为数据类的类，然后可以进一步使用`registry.mapped()`装饰器来将声明式样式的映射应用于类。作为使用`registry.mapped()`装饰器的替代方案，我们也可以将类通过`registry.map_imperatively()`方法传递，以便我们可以将所有`Table`和`Mapper`配置命令式地传递给函数，而不是将它们定义为类本身的类变量：

```py
from __future__ import annotations

from dataclasses import dataclass
from dataclasses import field
from typing import List

from sqlalchemy import Column
from sqlalchemy import ForeignKey
from sqlalchemy import Integer
from sqlalchemy import MetaData
from sqlalchemy import String
from sqlalchemy import Table
from sqlalchemy.orm import registry
from sqlalchemy.orm import relationship

mapper_registry = registry()

@dataclass
class User:
    id: int = field(init=False)
    name: str = None
    fullname: str = None
    nickname: str = None
    addresses: List[Address] = field(default_factory=list)

@dataclass
class Address:
    id: int = field(init=False)
    user_id: int = field(init=False)
    email_address: str = None

metadata_obj = MetaData()

user = Table(
    "user",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    Column("name", String(50)),
    Column("fullname", String(50)),
    Column("nickname", String(12)),
)

address = Table(
    "address",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    Column("user_id", Integer, ForeignKey("user.id")),
    Column("email_address", String(50)),
)

mapper_registry.map_imperatively(
    User,
    user,
    properties={
        "addresses": relationship(Address, backref="user", order_by=address.c.id),
    },
)

mapper_registry.map_imperatively(Address, address)
```

使用此映射样式时，请注意使用声明式带命令式表映射预先存在的数据类中提到的相同警告。### 使用声明式带命令式表映射预先存在的数据类

下面是使用 声明式和命令式表格（也称为混合声明式） 的 `@dataclass` 进行映射的示例。一个完整的 `Table` 对象被明确构建并分配给 `__table__` 属性。使用普通数据类语法定义实例字段。其他 `MapperProperty` 定义，如 `relationship()`，被放置在 __mapper_args__ 类级别的字典中，位于 `properties` 键下，对应于 `Mapper.properties` 参数：

```py
from __future__ import annotations

from dataclasses import dataclass, field
from typing import List, Optional

from sqlalchemy import Column, ForeignKey, Integer, String, Table
from sqlalchemy.orm import registry, relationship

mapper_registry = registry()

@mapper_registry.mapped
@dataclass
class User:
    __table__ = Table(
        "user",
        mapper_registry.metadata,
        Column("id", Integer, primary_key=True),
        Column("name", String(50)),
        Column("fullname", String(50)),
        Column("nickname", String(12)),
    )
    id: int = field(init=False)
    name: Optional[str] = None
    fullname: Optional[str] = None
    nickname: Optional[str] = None
    addresses: List[Address] = field(default_factory=list)

    __mapper_args__ = {  # type: ignore
        "properties": {
            "addresses": relationship("Address"),
        }
    }

@mapper_registry.mapped
@dataclass
class Address:
    __table__ = Table(
        "address",
        mapper_registry.metadata,
        Column("id", Integer, primary_key=True),
        Column("user_id", Integer, ForeignKey("user.id")),
        Column("email_address", String(50)),
    )
    id: int = field(init=False)
    user_id: int = field(init=False)
    email_address: Optional[str] = None
```

在上述示例中，`User.id`、`Address.id` 和 `Address.user_id` 属性被定义为 `field(init=False)`。这意味着这些属性的参数不会被添加到 `__init__()` 方法中，但`Session` 仍然可以在 flush 期间从自增或其他默认值生成器获取它们的值后设置它们。为了允许在构造函数中显式指定它们，它们将被给定一个默认值`None`。

要单独声明一个 `relationship()`，需要直接在 `Mapper.properties` 字典内指定它，该字典本身在 `__mapper_args__` 字典内指定，以便将其传递给 `Mapper` 的构造函数。这种方法的替代方案在下一个示例中。

警告

使用 `field()` 声明一个数据类并设置 `default` 以及 `init=False` 不会像在完全普通的数据类中预期的那样起作用，因为 SQLAlchemy 类的装饰会替换数据类创建过程中在类上设置的默认值。使用 `default_factory` 代替。当使用声明式数据类映射时，这个适配会自动完成。

### 使用声明式字段映射现有数据类

遗留特性

这种使用数据类进行声明式映射的方法应被视为遗留。它仍然会得到支持，但不太可能在声明式数据类映射的新方法面前提供任何优势。

请注意 **mapped_column() 不支持此用法**；应继续使用 `Column` 构造在 `dataclasses.field()` 的 `metadata` 字段内声明表元数据。

完全声明式的方法要求 `Column` 对象被声明为类属性，当使用数据类时，这将与数据类级别的属性冲突。结合这些的一种方法是利用 `dataclass.field` 对象上的 `metadata` 属性，其中可以提供特定于 SQLAlchemy 的映射信息。当类指定属性 `__sa_dataclass_metadata_key__` 时，声明式支持提取这些参数。这也提供了一种更简洁的方法来指示 `relationship()` 关联：

```py
from __future__ import annotations

from dataclasses import dataclass, field
from typing import List

from sqlalchemy import Column, ForeignKey, Integer, String
from sqlalchemy.orm import registry, relationship

mapper_registry = registry()

@mapper_registry.mapped
@dataclass
class User:
    __tablename__ = "user"

    __sa_dataclass_metadata_key__ = "sa"
    id: int = field(init=False, metadata={"sa": Column(Integer, primary_key=True)})
    name: str = field(default=None, metadata={"sa": Column(String(50))})
    fullname: str = field(default=None, metadata={"sa": Column(String(50))})
    nickname: str = field(default=None, metadata={"sa": Column(String(12))})
    addresses: List[Address] = field(
        default_factory=list, metadata={"sa": relationship("Address")}
    )

@mapper_registry.mapped
@dataclass
class Address:
    __tablename__ = "address"
    __sa_dataclass_metadata_key__ = "sa"
    id: int = field(init=False, metadata={"sa": Column(Integer, primary_key=True)})
    user_id: int = field(init=False, metadata={"sa": Column(ForeignKey("user.id"))})
    email_address: str = field(default=None, metadata={"sa": Column(String(50))})
```

#### 使用具有预先存在的数据类的声明式混合

在 使用混合组合映射层次结构 部分介绍了声明式 Mixin 类。声明式 mixins 的一个要求是，某些无法轻松复制的构造必须以可调用的形式给出，使用 `declared_attr` 装饰器，例如在 混合关系 中的示例：

```py
class RefTargetMixin:
    @declared_attr
    def target_id(cls):
        return Column("target_id", ForeignKey("target.id"))

    @declared_attr
    def target(cls):
        return relationship("Target")
```

此形式在 Dataclasses 的 `field()` 对象中得到支持，通过使用 lambda 来指示 `field()` 内部的 SQLAlchemy 构造。在 lambda 周围使用 `declared_attr()` 是可选的。如果我们想要生成我们上面的 `User` 类，其中 ORM 字段来自一个本身就是数据类的 mixin，形式将是：

```py
@dataclass
class UserMixin:
    __tablename__ = "user"

    __sa_dataclass_metadata_key__ = "sa"

    id: int = field(init=False, metadata={"sa": Column(Integer, primary_key=True)})

    addresses: List[Address] = field(
        default_factory=list, metadata={"sa": lambda: relationship("Address")}
    )

@dataclass
class AddressMixin:
    __tablename__ = "address"
    __sa_dataclass_metadata_key__ = "sa"
    id: int = field(init=False, metadata={"sa": Column(Integer, primary_key=True)})
    user_id: int = field(
        init=False, metadata={"sa": lambda: Column(ForeignKey("user.id"))}
    )
    email_address: str = field(default=None, metadata={"sa": Column(String(50))})

@mapper_registry.mapped
class User(UserMixin):
    pass

@mapper_registry.mapped
class Address(AddressMixin):
    pass
```

新版本 1.4.2 中新增：为“声明式表中的数据类”样式映射添加了对“已声明属性”样式 mixin 属性的支持，即用于“使用声明式混合具有预先存在的数据类”样式映射中的 `relationship()` 结构以及具有外键声明的 `Column` 对象。  #### 使用具有预先存在的数据类的声明式混合

在 使用混合组合映射层次结构 部分介绍了声明式 Mixin 类。声明式 mixins 的一个要求是，某些无法轻松复制的构造必须以可调用的形式给出，使用 `declared_attr` 装饰器，例如在 混合关系 中的示例：

```py
class RefTargetMixin:
    @declared_attr
    def target_id(cls):
        return Column("target_id", ForeignKey("target.id"))

    @declared_attr
    def target(cls):
        return relationship("Target")
```

在 Dataclasses `field()` 对象中支持此形式，通过使用 lambda 表示 SQLAlchemy 构造在 `field()` 内部。使用 `declared_attr()` 将 lambda 包围起来是可选的。如果我们想要生成上述的 `User` 类，其中 ORM 字段来自于一个自身是 dataclass 的 mixin，那么形式将是：

```py
@dataclass
class UserMixin:
    __tablename__ = "user"

    __sa_dataclass_metadata_key__ = "sa"

    id: int = field(init=False, metadata={"sa": Column(Integer, primary_key=True)})

    addresses: List[Address] = field(
        default_factory=list, metadata={"sa": lambda: relationship("Address")}
    )

@dataclass
class AddressMixin:
    __tablename__ = "address"
    __sa_dataclass_metadata_key__ = "sa"
    id: int = field(init=False, metadata={"sa": Column(Integer, primary_key=True)})
    user_id: int = field(
        init=False, metadata={"sa": lambda: Column(ForeignKey("user.id"))}
    )
    email_address: str = field(default=None, metadata={"sa": Column(String(50))})

@mapper_registry.mapped
class User(UserMixin):
    pass

@mapper_registry.mapped
class Address(AddressMixin):
    pass
```

版本 1.4.2 中的新内容：添加了对“声明属性”风格 mixin 属性的支持，即 `relationship()` 构造以及带有外键声明的 `Column` 对象，用于在“具有声明性表”的样式映射中使用。

### 使用命令式映射映射现有 dataclasses

如前所述，通过使用 `@dataclass` 装饰器设置为 dataclass 的类，然后可以进一步使用 `registry.mapped()` 装饰器装饰该类，以将声明性映射应用到类。作为使用 `registry.mapped()` 装饰器的替代方案，我们也可以将类传递给 `registry.map_imperatively()` 方法，这样我们就可以将所有 `Table` 和 `Mapper` 配置命令性地传递给函数，而不是将它们定义为类变量：

```py
from __future__ import annotations

from dataclasses import dataclass
from dataclasses import field
from typing import List

from sqlalchemy import Column
from sqlalchemy import ForeignKey
from sqlalchemy import Integer
from sqlalchemy import MetaData
from sqlalchemy import String
from sqlalchemy import Table
from sqlalchemy.orm import registry
from sqlalchemy.orm import relationship

mapper_registry = registry()

@dataclass
class User:
    id: int = field(init=False)
    name: str = None
    fullname: str = None
    nickname: str = None
    addresses: List[Address] = field(default_factory=list)

@dataclass
class Address:
    id: int = field(init=False)
    user_id: int = field(init=False)
    email_address: str = None

metadata_obj = MetaData()

user = Table(
    "user",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    Column("name", String(50)),
    Column("fullname", String(50)),
    Column("nickname", String(12)),
)

address = Table(
    "address",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    Column("user_id", Integer, ForeignKey("user.id")),
    Column("email_address", String(50)),
)

mapper_registry.map_imperatively(
    User,
    user,
    properties={
        "addresses": relationship(Address, backref="user", order_by=address.c.id),
    },
)

mapper_registry.map_imperatively(Address, address)
```

在使用此映射样式时，与使用声明性与命令性表映射现有数据类中提到的相同警告适用。

## 将 ORM 映射应用到现有 attrs 类

[attrs](https://pypi.org/project/attrs/) 库是一个流行的第三方库，提供了类似于 dataclasses 的功能，同时提供了许多在普通 dataclasses 中找不到的附加功能。

使用 [attrs](https://pypi.org/project/attrs/) 增强的类使用 `@define` 装饰器。此装饰器启动一个过程，用于扫描类以查找定义类行为的属性，然后使用这些属性生成方法、文档和注释。

SQLAlchemy ORM 支持使用**声明式与命令式表**或**命令式**映射来映射 [attrs](https://pypi.org/project/attrs/) 类。这两种风格的一般形式与数据类一起使用的使用声明式字段映射预先存在的数据类和使用声明式与命令式表映射预先存在的数据类的映射形式完全相同，其中数据类或 attrs 使用的内联属性指令保持不变，并且 SQLAlchemy 的面向表的仪器化在运行时应用。  

[attrs](https://pypi.org/project/attrs/) 的 `@define` 装饰器默认用新的基于 __slots__ 的类替换注释类，这是不受支持的。当使用旧式注释 `@attr.s` 或使用 `define(slots=False)` 时，类不会被替换。此外，attrs 在装饰器运行后移除了自己的类绑定属性，以便 SQLAlchemy 的映射过程接管这些属性而不会出现任何问题。`@attr.s` 和 `@define(slots=False)` 两个装饰器都适用于 SQLAlchemy。

### 使用声明式“命令式表”映射属性

在“声明式与命令式表”风格中，`Table` 对象与声明式类内联声明。首先将 `@define` 装饰器应用于类，然后将 `registry.mapped()` 装饰器应用于第二个位置：

```py
from __future__ import annotations

from typing import List
from typing import Optional

from attrs import define
from sqlalchemy import Column
from sqlalchemy import ForeignKey
from sqlalchemy import Integer
from sqlalchemy import MetaData
from sqlalchemy import String
from sqlalchemy import Table
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import registry
from sqlalchemy.orm import relationship

mapper_registry = registry()

@mapper_registry.mapped
@define(slots=False)
class User:
    __table__ = Table(
        "user",
        mapper_registry.metadata,
        Column("id", Integer, primary_key=True),
        Column("name", String(50)),
        Column("FullName", String(50), key="fullname"),
        Column("nickname", String(12)),
    )
    id: Mapped[int]
    name: Mapped[str]
    fullname: Mapped[str]
    nickname: Mapped[str]
    addresses: Mapped[List[Address]]

    __mapper_args__ = {  # type: ignore
        "properties": {
            "addresses": relationship("Address"),
        }
    }

@mapper_registry.mapped
@define(slots=False)
class Address:
    __table__ = Table(
        "address",
        mapper_registry.metadata,
        Column("id", Integer, primary_key=True),
        Column("user_id", Integer, ForeignKey("user.id")),
        Column("email_address", String(50)),
    )
    id: Mapped[int]
    user_id: Mapped[int]
    email_address: Mapped[Optional[str]]
```

注意

`attrs` 的 `slots=True` 选项，在映射类上启用了 `__slots__`，不能与未完全实现备用 属性仪器化 的 SQLAlchemy 映射一起使用，因为映射类通常依赖于直接访问 `__dict__` 进行状态存储。当此选项存在时，行为是未定义的。

### 使用命令式映射映射属性

正如对于数据类一样，我们可以使用 `registry.map_imperatively()` 来映射现有的 `attrs` 类：

```py
from __future__ import annotations

from typing import List

from attrs import define
from sqlalchemy import Column
from sqlalchemy import ForeignKey
from sqlalchemy import Integer
from sqlalchemy import MetaData
from sqlalchemy import String
from sqlalchemy import Table
from sqlalchemy.orm import registry
from sqlalchemy.orm import relationship

mapper_registry = registry()

@define(slots=False)
class User:
    id: int
    name: str
    fullname: str
    nickname: str
    addresses: List[Address]

@define(slots=False)
class Address:
    id: int
    user_id: int
    email_address: Optional[str]

metadata_obj = MetaData()

user = Table(
    "user",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    Column("name", String(50)),
    Column("fullname", String(50)),
    Column("nickname", String(12)),
)

address = Table(
    "address",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    Column("user_id", Integer, ForeignKey("user.id")),
    Column("email_address", String(50)),
)

mapper_registry.map_imperatively(
    User,
    user,
    properties={
        "addresses": relationship(Address, backref="user", order_by=address.c.id),
    },
)

mapper_registry.map_imperatively(Address, address)
```

上述形式等同于先前使用声明式与命令式表的示例。

### 使用声明式“命令式表”映射属性

在“声明式与命令式表”风格中，`Table` 对象与声明式类内联声明。首先将 `@define` 装饰器应用于类，然后将 `registry.mapped()` 装饰器应用于第二个位置：

```py
from __future__ import annotations

from typing import List
from typing import Optional

from attrs import define
from sqlalchemy import Column
from sqlalchemy import ForeignKey
from sqlalchemy import Integer
from sqlalchemy import MetaData
from sqlalchemy import String
from sqlalchemy import Table
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import registry
from sqlalchemy.orm import relationship

mapper_registry = registry()

@mapper_registry.mapped
@define(slots=False)
class User:
    __table__ = Table(
        "user",
        mapper_registry.metadata,
        Column("id", Integer, primary_key=True),
        Column("name", String(50)),
        Column("FullName", String(50), key="fullname"),
        Column("nickname", String(12)),
    )
    id: Mapped[int]
    name: Mapped[str]
    fullname: Mapped[str]
    nickname: Mapped[str]
    addresses: Mapped[List[Address]]

    __mapper_args__ = {  # type: ignore
        "properties": {
            "addresses": relationship("Address"),
        }
    }

@mapper_registry.mapped
@define(slots=False)
class Address:
    __table__ = Table(
        "address",
        mapper_registry.metadata,
        Column("id", Integer, primary_key=True),
        Column("user_id", Integer, ForeignKey("user.id")),
        Column("email_address", String(50)),
    )
    id: Mapped[int]
    user_id: Mapped[int]
    email_address: Mapped[Optional[str]]
```

注意

`attrs` 的 `slots=True` 选项可以在映射类上启用 `__slots__`，但在不完全实现替代的属性调试时，无法与 SQLAlchemy 映射一起使用，因为映射类通常依赖于直接访问 `__dict__` 来存储状态。当存在此选项时，行为是未定义的。

### 使用命令式映射（Imperative Mapping）进行属性映射

就像在数据类中一样，我们可以利用`registry.map_imperatively()`来映射现有的 `attrs` 类：

```py
from __future__ import annotations

from typing import List

from attrs import define
from sqlalchemy import Column
from sqlalchemy import ForeignKey
from sqlalchemy import Integer
from sqlalchemy import MetaData
from sqlalchemy import String
from sqlalchemy import Table
from sqlalchemy.orm import registry
from sqlalchemy.orm import relationship

mapper_registry = registry()

@define(slots=False)
class User:
    id: int
    name: str
    fullname: str
    nickname: str
    addresses: List[Address]

@define(slots=False)
class Address:
    id: int
    user_id: int
    email_address: Optional[str]

metadata_obj = MetaData()

user = Table(
    "user",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    Column("name", String(50)),
    Column("fullname", String(50)),
    Column("nickname", String(12)),
)

address = Table(
    "address",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    Column("user_id", Integer, ForeignKey("user.id")),
    Column("email_address", String(50)),
)

mapper_registry.map_imperatively(
    User,
    user,
    properties={
        "addresses": relationship(Address, backref="user", order_by=address.c.id),
    },
)

mapper_registry.map_imperatively(Address, address)
```

上述形式等同于使用命令式表格进行声明的上一个示例。
