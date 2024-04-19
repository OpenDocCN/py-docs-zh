# Mypy / Pep-484 对 ORM 映射的支持

> 原文：[`docs.sqlalchemy.org/en/20/orm/extensions/mypy.html`](https://docs.sqlalchemy.org/en/20/orm/extensions/mypy.html)

当使用直接引用 `Column` 对象而不是 SQLAlchemy 2.0 中引入的 `mapped_column()` 构造时，支持 [**PEP 484**](https://peps.python.org/pep-0484/) 类型注释以及 [MyPy](https://mypy.readthedocs.io/) 类型检查工具。

自 2.0 版开始已被弃用：**SQLAlchemy Mypy 插件已弃用，并且可能在 SQLAlchemy 2.1 发布时被移除。我们建议用户尽快迁移。**

无法跨不断变化的 mypy 发布维护此插件，未来的稳定性不能保证。

现代 SQLAlchemy 现在提供了 完全符合 pep-484 的映射语法；请参阅链接的部分以获取迁移详情。

## 安装

仅适用于 **SQLAlchemy 2.0**：不应安装存根，并且应完全卸载诸如 [sqlalchemy-stubs](https://github.com/dropbox/sqlalchemy-stubs) 和 [sqlalchemy2-stubs](https://github.com/sqlalchemy/sqlalchemy2-stubs) 等软件包。

[Mypy](https://mypy.readthedocs.io/) 包本身是一个依赖项。

可以使用 pip 使用“mypy”额外钩子安装 Mypy：

```py
pip install sqlalchemy[mypy]
```

插件本身如 [Configuring mypy to use Plugins](https://mypy.readthedocs.io/en/latest/extending_mypy.html#configuring-mypy-to-use-plugins) 中描述的那样配置，使用 `sqlalchemy.ext.mypy.plugin` 模块名，例如在 `setup.cfg` 中：

```py
[mypy]
plugins = sqlalchemy.ext.mypy.plugin
```

## 插件功能

Mypy 插件的主要目的是拦截并修改 SQLAlchemy 声明性映射 的静态定义，使其与它们在被其 `Mapper` 对象 instrumented 后的结构相匹配。这允许类结构本身以及使用类的代码对 Mypy 工具有意义，否则基于当前声明性映射的功能，这是不可能的。该插件类似于需要为类似 [dataclasses](https://docs.python.org/3/library/dataclasses.html) 这样的库修改类的动态插件。

为了涵盖这种情况经常发生的主要区域，考虑以下 ORM 映射，使用 `User` 类的典型示例：

```py
from sqlalchemy import Column, Integer, String, select
from sqlalchemy.orm import declarative_base

# "Base" is a class that is created dynamically from the
# declarative_base() function
Base = declarative_base()

class User(Base):
    __tablename__ = "user"

    id = Column(Integer, primary_key=True)
    name = Column(String)

# "some_user" is an instance of the User class, which
# accepts "id" and "name" kwargs based on the mapping
some_user = User(id=5, name="user")

# it has an attribute called .name that's a string
print(f"Username: {some_user.name}")

# a select() construct makes use of SQL expressions derived from the
# User class itself
select_stmt = select(User).where(User.id.in_([3, 4, 5])).where(User.name.contains("s"))
```

上述，Mypy 扩展可以执行的步骤包括：

+   解释由 `declarative_base()` 生成的 `Base` 动态类，以便从中继承的类被认为是映射的。它还可以适应在使用装饰器进行声明式映射（无声明式基类）中描述的类装饰器方法。

+   对在声明式“内联”样式中定义的 ORM 映射属性进行类型推断，例如上面示例中 `User` 类的 `id` 和 `name` 属性。这包括 `User` 的实例将使用 `int` 类型的 `id` 和 `str` 类型的 `name`。还包括当访问 `User.id` 和 `User.name` 类级属性时，如上面的 `select()` 语句中所示，它们与 SQL 表达式行为兼容，这是从 `InstrumentedAttribute` 属性描述符类派生的。

+   将 `__init__()` 方法应用于尚未包含显式构造函数的映射类，该构造函数接受检测到的所有映射属性的特定类型的关键字参数。

当 Mypy 插件处理上述文件时，传递给 Mypy 工具的结果静态类定义和 Python 代码等效于以下内容：

```py
from sqlalchemy import Column, Integer, String, select
from sqlalchemy.orm import Mapped
from sqlalchemy.orm.decl_api import DeclarativeMeta

class Base(metaclass=DeclarativeMeta):
    __abstract__ = True

class User(Base):
    __tablename__ = "user"

    id: Mapped[Optional[int]] = Mapped._special_method(
        Column(Integer, primary_key=True)
    )
    name: Mapped[Optional[str]] = Mapped._special_method(Column(String))

    def __init__(self, id: Optional[int] = ..., name: Optional[str] = ...) -> None: ...

some_user = User(id=5, name="user")

print(f"Username: {some_user.name}")

select_stmt = select(User).where(User.id.in_([3, 4, 5])).where(User.name.contains("s"))
```

上述已经采取的关键步骤包括：

+   `Base` 类现在明确地是基于 `DeclarativeMeta` 类定义的，而不再是一个动态类。

+   `id` 和 `name` 属性是基于 `Mapped` 类定义的，该类代表一个在类和实例级别表现出不同行为的 Python 描述符。`Mapped` 类现在是用于所有 ORM 映射属性的 `InstrumentedAttribute` 类的基类。

    `Mapped` 被定义为一个针对任意 Python 类型的通用类，这意味着特定的 `Mapped` 实例与特定的 Python 类型相关联，例如上面的 `Mapped[Optional[int]]` 和 `Mapped[Optional[str]`。

+   声明性映射属性赋值的右侧被**移除**，因为这类似于`Mapper`类通常要执行的操作，即它将用`InstrumentedAttribute`](../internals.html#sqlalchemy.orm.InstrumentedAttribute "sqlalchemy.orm.InstrumentedAttribute")的特定实例替换这些属性。原始表达式移动到一个函数调用中，这样可以仍然进行类型检查而不与表达式的左侧冲突。对于 Mypy 来说，左侧的类型注释足以理解属性的行为。

+   添加了`User.__init__()`方法的类型存根，其中包括了正确的关键字和数据类型。

## 用法

以下各小节将讨论到目前为止已经考虑到的符合 PEP-484 的各种使用情况。

### 基于 TypeEngine 的列的内省

对于包含显式数据类型的映射列，当它们被映射为内联属性时，映射类型将被自动内省：

```py
class MyClass(Base):
    # ...

    id = Column(Integer, primary_key=True)
    name = Column("employee_name", String(50), nullable=False)
    other_name = Column(String(50))
```

上述，`id`，`name`和`other_name`的最终类级数据类型将被内省为`Mapped[Optional[int]]`，`Mapped[Optional[str]]`和`Mapped[Optional[str]]`。这些类型默认始终被认为是`Optional`，即使对于主键和非空列也是如此。原因是因为虽然数据库列`id`和`name`不能为 NULL，但 Python 属性`id`和`name`很可能是`None`，而不需要显式的构造函数：

```py
>>> m1 = MyClass()
>>> m1.id
None
```

上述列的类型可以被**显式地**声明，提供了更清晰的自我文档化以及能够控制哪些类型是可选的两个优点：

```py
class MyClass(Base):
    # ...

    id: int = Column(Integer, primary_key=True)
    name: str = Column("employee_name", String(50), nullable=False)
    other_name: Optional[str] = Column(String(50))
```

Mypy 插件将接受上述`int`，`str`和`Optional[str]`并将它们转换为包含在其周围的`Mapped[]`类型。`Mapped[]`构造也可以被显式使用：

```py
from sqlalchemy.orm import Mapped

class MyClass(Base):
    # ...

    id: Mapped[int] = Column(Integer, primary_key=True)
    name: Mapped[str] = Column("employee_name", String(50), nullable=False)
    other_name: Mapped[Optional[str]] = Column(String(50))
```

当类型是非可选时，这意味着从`MyClass`的实例中访问的属性将被认为是非`None`的：

```py
mc = MyClass(...)

# will pass mypy --strict
name: str = mc.name
```

对于可选属性，Mypy 认为类型必须包含 None，否则就是`Optional`：

```py
mc = MyClass(...)

# will pass mypy --strict
other_name: Optional[str] = mc.name
```

无论映射的属性是否被标记为`Optional`，`__init__()`方法的生成都**仍然认为所有关键字都是可选的**。这再次与 SQLAlchemy ORM 在创建构造函数时实际执行的操作相匹配，不应与诸如 Python `dataclasses`之类的验证系统的行为混淆，后者将生成一个根据注释匹配的构造函数，包括可选和必需的属性。

### 没有明确类型的列

包含`ForeignKey`修饰符的列在 SQLAlchemy 声明映射中不需要指定数据类型。对于这种类型的属性，Mypy 插件将通知用户需要发送明确的类型：

```py
# .. other imports
from sqlalchemy.sql.schema import ForeignKey

Base = declarative_base()

class User(Base):
    __tablename__ = "user"

    id = Column(Integer, primary_key=True)
    name = Column(String)

class Address(Base):
    __tablename__ = "address"

    id = Column(Integer, primary_key=True)
    user_id = Column(ForeignKey("user.id"))
```

插件将按以下方式传递消息：

```py
$ mypy test3.py --strict
test3.py:20: error: [SQLAlchemy Mypy plugin] Can't infer type from
ORM mapped expression assigned to attribute 'user_id'; please specify a
Python type or Mapped[<python type>] on the left hand side.
Found 1 error in 1 file (checked 1 source file)
```

要解决问题，请对`Address.user_id`列应用明确的类型注释：

```py
class Address(Base):
    __tablename__ = "address"

    id = Column(Integer, primary_key=True)
    user_id: int = Column(ForeignKey("user.id"))
```

### 使用命令式表映射列

在命令式表样式中，`Column`定义位于与映射属性本身分开的`Table`构造内。Mypy 插件不考虑这个`Table`，而是支持可以明确声明属性，并且**必须**使用`Mapped`类将其标识为映射属性：

```py
class MyClass(Base):
    __table__ = Table(
        "mytable",
        Base.metadata,
        Column(Integer, primary_key=True),
        Column("employee_name", String(50), nullable=False),
        Column(String(50)),
    )

    id: Mapped[int]
    name: Mapped[str]
    other_name: Mapped[Optional[str]]
```

上述`Mapped`注释被视为映射列，并将包含在默认构造函数中，同时为`MyClass`在类级别和实例级别提供正确的类型配置文件。

### 映射关系

该插件对使用类型推断来检测关系类型有限支持。对于所有无法检测类型的情况，它将发出信息丰富的错误消息，并且在所有情况下，可以明确提供适当的类型，要么使用`Mapped`类，要么选择在内联声明中省略它。插件还需要确定关系是指向集合还是标量，并且为此依赖于`relationship.uselist`和/或`relationship.collection_class`参数的显式值。如果这些参数都不存在，则需要明确的类型，以及如果`relationship()`的目标类型是字符串或可调用对象，而不是类：

```py
class User(Base):
    __tablename__ = "user"

    id = Column(Integer, primary_key=True)
    name = Column(String)

class Address(Base):
    __tablename__ = "address"

    id = Column(Integer, primary_key=True)
    user_id: int = Column(ForeignKey("user.id"))

    user = relationship(User)
```

上述映射将产生以下错误：

```py
test3.py:22: error: [SQLAlchemy Mypy plugin] Can't infer scalar or
collection for ORM mapped expression assigned to attribute 'user'
if both 'uselist' and 'collection_class' arguments are absent from the
relationship(); please specify a type annotation on the left hand side.
Found 1 error in 1 file (checked 1 source file)
```

可以通过使用`relationship(User, uselist=False)`或提供类型来解决错误，在这种情况下是标量`User`对象：

```py
class Address(Base):
    __tablename__ = "address"

    id = Column(Integer, primary_key=True)
    user_id: int = Column(ForeignKey("user.id"))

    user: User = relationship(User)
```

对于集合，类似的模式也适用，即在没有`uselist=True`或`relationship.collection_class`的情况下，可以使用诸如`List`之类的集合注释。还可以完全适当地使用类的字符串名称进行注释，如 pep-484 所支持，确保根据需要在[TYPE_CHECKING 块](https://www.python.org/dev/peps/pep-0484/#runtime-or-type-checking)中导入类：

```py
from typing import TYPE_CHECKING, List

from .mymodel import Base

if TYPE_CHECKING:
    # if the target of the relationship is in another module
    # that cannot normally be imported at runtime
    from .myaddressmodel import Address

class User(Base):
    __tablename__ = "user"

    id = Column(Integer, primary_key=True)
    name = Column(String)
    addresses: List["Address"] = relationship("Address")
```

与列一样，`Mapped` 类也可以显式应用：

```py
class User(Base):
    __tablename__ = "user"

    id = Column(Integer, primary_key=True)
    name = Column(String)

    addresses: Mapped[List["Address"]] = relationship("Address", back_populates="user")

class Address(Base):
    __tablename__ = "address"

    id = Column(Integer, primary_key=True)
    user_id: int = Column(ForeignKey("user.id"))

    user: Mapped[User] = relationship(User, back_populates="addresses")
```

### 使用 @declared_attr 和声明性混合类

`declared_attr` 类允许在类级别函数中声明声明性映射的属性，并且在使用声明性混合类时特别有用。对于这些函数，函数的返回类型应使用`Mapped[]`构造或指示函数返回的确切对象类型进行注释。此外，“mixin”类（即不以`declarative_base()`类扩展，也不使用诸如`registry.mapped()`之类的方法映射的类）应该被装饰上 `declarative_mixin()` 装饰器，这为 Mypy 插件提供了一个提示，指明特定的类意图充当声明性混合类：

```py
from sqlalchemy.orm import declarative_mixin, declared_attr

@declarative_mixin
class HasUpdatedAt:
    @declared_attr
    def updated_at(cls) -> Column[DateTime]:  # uses Column
        return Column(DateTime)

@declarative_mixin
class HasCompany:
    @declared_attr
    def company_id(cls) -> Mapped[int]:  # uses Mapped
        return Column(ForeignKey("company.id"))

    @declared_attr
    def company(cls) -> Mapped["Company"]:
        return relationship("Company")

class Employee(HasUpdatedAt, HasCompany, Base):
    __tablename__ = "employee"

    id = Column(Integer, primary_key=True)
    name = Column(String)
```

注意方法`HasCompany.company`的实际返回类型与注释之间的不匹配。Mypy 插件将所有`@declared_attr`函数转换为简单的带注释的属性，以避免这种复杂性：

```py
# what Mypy sees
class HasCompany:
    company_id: Mapped[int]
    company: Mapped["Company"]
```

### 与数据类或其他类型敏感的属性系统相结合

在 将 ORM 映射应用到现有数据类（遗留数据类用法） 中的 Python 数据类集成示例存在一个问题；Python 数据类期望明确的类型，它将用于构建类，并且每个赋值语句中给定的值都是重要的。也就是说，必须确切地声明以下类才能被数据类接受：

```py
mapper_registry: registry = registry()

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
        "properties": {"addresses": relationship("Address")}
    }
```

我们不能将我们的`Mapped[]`类型应用于属性`id`、`name`等，因为它们会被`@dataclass`装饰器拒绝。此外，Mypy 还有另一个专门用于数据类的插件，这也可能妨碍我们的操作。

上述类实际上会顺利通过 Mypy 的类型检查；我们唯一缺少的是在`User`上的属性可用于 SQL 表达式，例如：

```py
stmt = select(User.name).where(User.id.in_([1, 2, 3]))
```

为了提供一个解决方法，Mypy 插件具有一个额外的功能，我们可以指定一个额外的属性 `_mypy_mapped_attrs`，它是一个包含类级对象或它们的字符串名称的列表。这个属性可以在 `TYPE_CHECKING` 变量内部是条件性的：

```py
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
    fullname: Optional[str]
    nickname: Optional[str]
    addresses: List[Address] = field(default_factory=list)

    if TYPE_CHECKING:
        _mypy_mapped_attrs = [id, name, "fullname", "nickname", addresses]

    __mapper_args__ = {  # type: ignore
        "properties": {"addresses": relationship("Address")}
    }
```

使用上述方法，列在 `_mypy_mapped_attrs` 中的属性将应用 `Mapped` 类型信息，以便在类绑定上下文中使用 `User` 类时，它将表现为一个 SQLAlchemy 映射类。

## 安装

对于 **仅适用于 SQLAlchemy 2.0**：不应安装存根，而应完全卸载像 [sqlalchemy-stubs](https://github.com/dropbox/sqlalchemy-stubs) 和 [sqlalchemy2-stubs](https://github.com/sqlalchemy/sqlalchemy2-stubs) 这样的包。

[Mypy](https://mypy.readthedocs.io/) 包本身是一个依赖项。

可以使用 pip 使用 “mypy” extras 钩子安装 Mypy：

```py
pip install sqlalchemy[mypy]
```

插件本身配置如 [配置 mypy 使用插件](https://mypy.readthedocs.io/en/latest/extending_mypy.html#configuring-mypy-to-use-plugins) 中所述，使用 `sqlalchemy.ext.mypy.plugin` 模块名称，例如在 `setup.cfg` 中：

```py
[mypy]
plugins = sqlalchemy.ext.mypy.plugin
```

## 插件的功能

Mypy 插件的主要目的是拦截和修改 SQLAlchemy 声明式映射 的静态定义，以使其与它们在被 `Mapper` 对象 仪器化 后的结构相匹配。这使得类结构本身以及使用类的代码对 Mypy 工具有意义，否则根据当前声明式映射的功能，这将不是情况。该插件类似于为像 [dataclasses](https://docs.python.org/3/library/dataclasses.html) 这样的库所需的类似插件，这些插件在运行时动态地修改类。

要涵盖此类情况经常发生的主要领域，请考虑以下 ORM 映射，使用 `User` 类的典型示例：

```py
from sqlalchemy import Column, Integer, String, select
from sqlalchemy.orm import declarative_base

# "Base" is a class that is created dynamically from the
# declarative_base() function
Base = declarative_base()

class User(Base):
    __tablename__ = "user"

    id = Column(Integer, primary_key=True)
    name = Column(String)

# "some_user" is an instance of the User class, which
# accepts "id" and "name" kwargs based on the mapping
some_user = User(id=5, name="user")

# it has an attribute called .name that's a string
print(f"Username: {some_user.name}")

# a select() construct makes use of SQL expressions derived from the
# User class itself
select_stmt = select(User).where(User.id.in_([3, 4, 5])).where(User.name.contains("s"))
```

上面，Mypy 扩展可以执行的步骤包括：

+   对由 `declarative_base()` 生成的 `Base` 动态类进行解释，以便继承它的类被知道是已映射的。它还可以适应 使用装饰器进行声明式映射（无声明式基类） 中描述的类装饰器方法。

+   对于在声明式“内联”样式中定义的 ORM 映射属性的类型推断，例如上面示例中 `User` 类的 `id` 和 `name` 属性。这包括 `User` 实例将使用 `int` 类型的 `id` 和 `str` 类型的 `name`。它还包括当访问 `User.id` 和 `User.name` 类级属性时，正如它们在上面的 `select()` 语句中那样，它们与 SQL 表达式行为兼容，这是从 `InstrumentedAttribute` 属性描述符类派生的。

+   将 `__init__()` 方法应用于尚未包含显式构造函数的映射类，该构造函数接受特定类型的关键字参数，用于检测到的所有映射属性。

当 Mypy 插件处理上述文件时，结果的静态类定义和传递给 Mypy 工具的 Python 代码等效于以下内容：

```py
from sqlalchemy import Column, Integer, String, select
from sqlalchemy.orm import Mapped
from sqlalchemy.orm.decl_api import DeclarativeMeta

class Base(metaclass=DeclarativeMeta):
    __abstract__ = True

class User(Base):
    __tablename__ = "user"

    id: Mapped[Optional[int]] = Mapped._special_method(
        Column(Integer, primary_key=True)
    )
    name: Mapped[Optional[str]] = Mapped._special_method(Column(String))

    def __init__(self, id: Optional[int] = ..., name: Optional[str] = ...) -> None: ...

some_user = User(id=5, name="user")

print(f"Username: {some_user.name}")

select_stmt = select(User).where(User.id.in_([3, 4, 5])).where(User.name.contains("s"))
```

以上已经采取的关键步骤包括：

+   `Base` 类现在明确地以 `DeclarativeMeta` 类的形式定义，而不是动态类。

+   `id` 和 `name` 属性是以 `Mapped` 类的术语定义的，该类表示在类与实例级别上表现出不同行为的 Python 描述符。`Mapped` 类现在是用于所有 ORM 映射属性的 `InstrumentedAttribute` 类的基类。

    `Mapped` 被定义为针对任意 Python 类型的通用类，这意味着 `Mapped` 的特定出现与特定的 Python 类型相关联，例如上面的 `Mapped[Optional[int]]` 和 `Mapped[Optional[str]]`。

+   声明式映射属性分配的右侧 **已移除**，因为这类似于 `Mapper` 类通常会执行的操作，即它将这些属性替换为 `InstrumentedAttribute` 的具体实例。原始表达式移到一个函数调用中，这将允许它仍然被类型检查而不与表达式的左侧发生冲突。对于 Mypy 来说，左侧的类型注释足以理解属性的行为。

+   为 `User.__init__()` 方法添加了类型存根，其中包括正确的关键字和数据类型。

## 使用方法

以下各小节将讨论迄今为止已考虑到的个别用例的 pep-484 符合性。

### 基于 TypeEngine 的列的自省

对于包含显式数据类型的映射列，当它们作为内联属性映射时，映射类型将被自动解析：

```py
class MyClass(Base):
    # ...

    id = Column(Integer, primary_key=True)
    name = Column("employee_name", String(50), nullable=False)
    other_name = Column(String(50))
```

在上面，`id`、`name` 和 `other_name` 这些最终的类级别数据类型将被解析为 `Mapped[Optional[int]]`、`Mapped[Optional[str]]` 和 `Mapped[Optional[str]]`。这些类型默认情况下**总是**被认为是 `Optional` 的，即使对于主键和非空列也是如此。原因是因为虽然数据库列`id`和`name`不能为 NULL，但 Python 属性 `id` 和 `name` 在没有显式构造函数的情况下肯定可以是 `None`：

```py
>>> m1 = MyClass()
>>> m1.id
None
```

上述列的类型可以被**显式地**声明，提供了更清晰的自我文档说明以及能够控制哪些类型是可选的两个优点：

```py
class MyClass(Base):
    # ...

    id: int = Column(Integer, primary_key=True)
    name: str = Column("employee_name", String(50), nullable=False)
    other_name: Optional[str] = Column(String(50))
```

Mypy 插件将接受上述的 `int`、`str` 和 `Optional[str]`，并将它们转换为包含在 `Mapped[]` 类型周围的类型。`Mapped[]` 结构也可以被显式使用：

```py
from sqlalchemy.orm import Mapped

class MyClass(Base):
    # ...

    id: Mapped[int] = Column(Integer, primary_key=True)
    name: Mapped[str] = Column("employee_name", String(50), nullable=False)
    other_name: Mapped[Optional[str]] = Column(String(50))
```

当类型是非可选时，这意味着从 `MyClass` 实例访问的属性将被视为非 None：

```py
mc = MyClass(...)

# will pass mypy --strict
name: str = mc.name
```

对于可选属性，Mypy 认为类型必须包括 None，否则为 `Optional`：

```py
mc = MyClass(...)

# will pass mypy --strict
other_name: Optional[str] = mc.name
```

无论映射属性是否被标记为 `Optional`，生成的 `__init__()` 方法仍然**将所有关键字视为可选的**。这再次与 SQLAlchemy ORM 实际创建构造函数时的行为相匹配，不应与诸如 Python `dataclasses` 之类的验证系统的行为混淆，后者将生成一个与注释匹配的构造函数，以确定可选 vs. 必需属性的注解。 

### 没有明确类型的列

包含 `ForeignKey` 修改器的列在 SQLAlchemy 声明式映射中不需要指定数据类型。对于这种类型的属性，Mypy 插件将通知用户需要发送一个显式类型：

```py
# .. other imports
from sqlalchemy.sql.schema import ForeignKey

Base = declarative_base()

class User(Base):
    __tablename__ = "user"

    id = Column(Integer, primary_key=True)
    name = Column(String)

class Address(Base):
    __tablename__ = "address"

    id = Column(Integer, primary_key=True)
    user_id = Column(ForeignKey("user.id"))
```

插件将如下传递消息：

```py
$ mypy test3.py --strict
test3.py:20: error: [SQLAlchemy Mypy plugin] Can't infer type from
ORM mapped expression assigned to attribute 'user_id'; please specify a
Python type or Mapped[<python type>] on the left hand side.
Found 1 error in 1 file (checked 1 source file)
```

要解决此问题，请为 `Address.user_id` 列应用显式类型注释：

```py
class Address(Base):
    __tablename__ = "address"

    id = Column(Integer, primary_key=True)
    user_id: int = Column(ForeignKey("user.id"))
```

### 使用命令式表格映射列

在命令式表格风格中，`Column` 定义位于一个与映射属性本身分离的 `Table` 结构内。Mypy 插件不考虑这个 `Table`，而是支持可以显式声明属性，必须使用 `Mapped` 类来标识它们为映射属性：

```py
class MyClass(Base):
    __table__ = Table(
        "mytable",
        Base.metadata,
        Column(Integer, primary_key=True),
        Column("employee_name", String(50), nullable=False),
        Column(String(50)),
    )

    id: Mapped[int]
    name: Mapped[str]
    other_name: Mapped[Optional[str]]
```

上述`Mapped`注释被视为映射列，并将包含在默认构造函数中，以及在类级别和实例级别为`MyClass`提供正确的类型配置文件。

### 映射关系

该插件对使用类型推断来检测关系的类型有限支持。对于所有这些无法检测到类型的情况，它都将发出一个信息丰富的错误消息，在所有情况下，可以明确提供适当的类型，要么使用`Mapped`类，要么选择在内联声明中省略它。该插件还需要确定关系是指向集合还是标量，并且为此依赖于`relationship.uselist`和/或`relationship.collection_class`参数的显式值。如果这些参数都不存在，则需要明确的类型，以及如果`relationship()`的目标类型是字符串或可调用对象而不是类，则也需要明确的类型：

```py
class User(Base):
    __tablename__ = "user"

    id = Column(Integer, primary_key=True)
    name = Column(String)

class Address(Base):
    __tablename__ = "address"

    id = Column(Integer, primary_key=True)
    user_id: int = Column(ForeignKey("user.id"))

    user = relationship(User)
```

上述映射将产生以下错误：

```py
test3.py:22: error: [SQLAlchemy Mypy plugin] Can't infer scalar or
collection for ORM mapped expression assigned to attribute 'user'
if both 'uselist' and 'collection_class' arguments are absent from the
relationship(); please specify a type annotation on the left hand side.
Found 1 error in 1 file (checked 1 source file)
```

错误可以通过使用`relationship(User, uselist=False)`或者提供类型来解决，例如标量`User`对象：

```py
class Address(Base):
    __tablename__ = "address"

    id = Column(Integer, primary_key=True)
    user_id: int = Column(ForeignKey("user.id"))

    user: User = relationship(User)
```

对于集合，类似的模式也适用，如果没有`uselist=True`或者`relationship.collection_class`，可以使用集合注释，如`List`。在注释中使用类的字符串名称也是完全合适的，这是由 pep-484 支持的，确保在适当的时候在[TYPE_CHECKING 块](https://www.python.org/dev/peps/pep-0484/#runtime-or-type-checking)中导入该类：

```py
from typing import TYPE_CHECKING, List

from .mymodel import Base

if TYPE_CHECKING:
    # if the target of the relationship is in another module
    # that cannot normally be imported at runtime
    from .myaddressmodel import Address

class User(Base):
    __tablename__ = "user"

    id = Column(Integer, primary_key=True)
    name = Column(String)
    addresses: List["Address"] = relationship("Address")
```

与列相似，`Mapped`类也可以显式地应用：

```py
class User(Base):
    __tablename__ = "user"

    id = Column(Integer, primary_key=True)
    name = Column(String)

    addresses: Mapped[List["Address"]] = relationship("Address", back_populates="user")

class Address(Base):
    __tablename__ = "address"

    id = Column(Integer, primary_key=True)
    user_id: int = Column(ForeignKey("user.id"))

    user: Mapped[User] = relationship(User, back_populates="addresses")
```

### 使用 @declared_attr 和声明性混合

`declared_attr`类允许在类级函数中声明 Declarative 映射属性，并且在使用声明性混入时特别有用。对于这些函数，函数的返回类型应该使用`Mapped[]`构造进行注释，或者指示函数返回的对象的确切类型。此外，未被映射的“混入”类（即不从`declarative_base()`类继承，也不使用诸如`registry.mapped()`之类的方法进行映射）应该用`declarative_mixin()`装饰器进行修饰，这为 Mypy 插件提供了一个提示，表明特定类意图作为声明性混入：

```py
from sqlalchemy.orm import declarative_mixin, declared_attr

@declarative_mixin
class HasUpdatedAt:
    @declared_attr
    def updated_at(cls) -> Column[DateTime]:  # uses Column
        return Column(DateTime)

@declarative_mixin
class HasCompany:
    @declared_attr
    def company_id(cls) -> Mapped[int]:  # uses Mapped
        return Column(ForeignKey("company.id"))

    @declared_attr
    def company(cls) -> Mapped["Company"]:
        return relationship("Company")

class Employee(HasUpdatedAt, HasCompany, Base):
    __tablename__ = "employee"

    id = Column(Integer, primary_key=True)
    name = Column(String)
```

注意像`HasCompany.company`这样的方法的实际返回类型与注释的不匹配。Mypy 插件将所有`@declared_attr`函数转换为简单的注释属性，以避免这种复杂性：

```py
# what Mypy sees
class HasCompany:
    company_id: Mapped[int]
    company: Mapped["Company"]
```

### 与 Dataclasses 或其他类型敏感的属性系统结合

Python dataclasses 集成的示例在将 ORM 映射应用于现有数据类（传统数据类用法）中提出了一个问题；Python dataclasses 期望一个明确的类型，它将用于构建类，并且每个赋值语句中给定的值是重要的。也就是说，一个如下所示的类必须要准确地声明才能被 dataclasses 接受：

```py
mapper_registry: registry = registry()

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
        "properties": {"addresses": relationship("Address")}
    }
```

我们无法将我们的`Mapped[]`类型应用于属性`id`、`name`等，因为它们将被`@dataclass`装饰器拒绝。此外，Mypy 还有另一个专门用于 dataclasses 的插件，这也可能影响我们的操作。

上述类实际上会通过 Mypy 的类型检查而没有问题；我们唯一缺少的是`User`上的属性能够在 SQL 表达式中使用，比如：

```py
stmt = select(User.name).where(User.id.in_([1, 2, 3]))
```

为了解决这个问题，Mypy 插件有一个额外的功能，我们可以指定一个额外的属性`_mypy_mapped_attrs`，这是一个包含类级对象或它们的字符串名称的列表。这个属性可以在`TYPE_CHECKING`变量内部进行条件判断：

```py
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
    fullname: Optional[str]
    nickname: Optional[str]
    addresses: List[Address] = field(default_factory=list)

    if TYPE_CHECKING:
        _mypy_mapped_attrs = [id, name, "fullname", "nickname", addresses]

    __mapper_args__ = {  # type: ignore
        "properties": {"addresses": relationship("Address")}
    }
```

使用上述方法，列在`_mypy_mapped_attrs`中列出的属性将应用于`Mapped`类型信息，以便在类绑定上下文中使用`User`类时，它将表现为一个 SQLAlchemy 映射类。

### 基于 TypeEngine 的列的内省

对于包含显式数据类型的映射列，当它们被映射为内联属性时，映射类型将自动进行内省：

```py
class MyClass(Base):
    # ...

    id = Column(Integer, primary_key=True)
    name = Column("employee_name", String(50), nullable=False)
    other_name = Column(String(50))
```

在上面，`id`、`name`和`other_name`的最终类级数据类型将被内省为`Mapped[Optional[int]]`、`Mapped[Optional[str]]`和`Mapped[Optional[str]]`。类型默认始终被视为**可选**，即使对于主键和非空列也是如此。原因是因为虽然数据库列`id`和`name`不能为 NULL，但 Python 属性`id`和`name`可以毫无疑问地是`None`，而不需要显式构造函数：

```py
>>> m1 = MyClass()
>>> m1.id
None
```

上述列的类型可以**明确**声明，提供两个优势，即更清晰的自我文档化以及能够控制哪些类型是可选的：

```py
class MyClass(Base):
    # ...

    id: int = Column(Integer, primary_key=True)
    name: str = Column("employee_name", String(50), nullable=False)
    other_name: Optional[str] = Column(String(50))
```

Mypy 插件将接受上述`int`、`str`和`Optional[str]`，并将它们转换为包围它们的`Mapped[]`类型。`Mapped[]`结构也可以明确使用：

```py
from sqlalchemy.orm import Mapped

class MyClass(Base):
    # ...

    id: Mapped[int] = Column(Integer, primary_key=True)
    name: Mapped[str] = Column("employee_name", String(50), nullable=False)
    other_name: Mapped[Optional[str]] = Column(String(50))
```

当类型为非可选时，这意味着从`MyClass`实例中访问的属性将被视为非`None`：

```py
mc = MyClass(...)

# will pass mypy --strict
name: str = mc.name
```

对于可选属性，Mypy 认为类型必须包含 None，否则为`Optional`：

```py
mc = MyClass(...)

# will pass mypy --strict
other_name: Optional[str] = mc.name
```

无论映射属性是否被标记为`Optional`，`__init__()`方法的生成**仍然考虑所有关键字都是可选的**。这再次与 SQLAlchemy ORM 实际创建构造函数时的行为相匹配，不应与验证系统（如 Python `dataclasses`）的行为混淆，后者将根据注释生成与可选与必需属性相匹配的构造函数。

### 不具有显式类型的列

包含 `ForeignKey` 修改器的列在 SQLAlchemy 声明性映射中不需要指定数据类型。对于这种类型的属性，Mypy 插件将通知用户需要发送显式类型：

```py
# .. other imports
from sqlalchemy.sql.schema import ForeignKey

Base = declarative_base()

class User(Base):
    __tablename__ = "user"

    id = Column(Integer, primary_key=True)
    name = Column(String)

class Address(Base):
    __tablename__ = "address"

    id = Column(Integer, primary_key=True)
    user_id = Column(ForeignKey("user.id"))
```

插件将以以下方式发送消息：

```py
$ mypy test3.py --strict
test3.py:20: error: [SQLAlchemy Mypy plugin] Can't infer type from
ORM mapped expression assigned to attribute 'user_id'; please specify a
Python type or Mapped[<python type>] on the left hand side.
Found 1 error in 1 file (checked 1 source file)
```

要解决此问题，请对`Address.user_id`列应用显式类型注释：

```py
class Address(Base):
    __tablename__ = "address"

    id = Column(Integer, primary_key=True)
    user_id: int = Column(ForeignKey("user.id"))
```

### 使用命令式表映射列

在 命令式表风格 中，`Column` 定义放在一个独立于映射属性本身的 `Table` 结构中。Mypy 插件不考虑这个`Table`，而是支持可以明确声明属性，并且**必须**使用 `Mapped` 类来标识它们为映射属性：

```py
class MyClass(Base):
    __table__ = Table(
        "mytable",
        Base.metadata,
        Column(Integer, primary_key=True),
        Column("employee_name", String(50), nullable=False),
        Column(String(50)),
    )

    id: Mapped[int]
    name: Mapped[str]
    other_name: Mapped[Optional[str]]
```

上述`Mapped`注释被视为映射列，并将包含在默认构造函数中，同时为`MyClass`提供正确的类型配置文件，无论是在类级别还是实例级别。

### 映射关系

该插件对使用类型推断来检测关系类型有限支持。对于所有无法检测类型的情况，它将发出信息丰富的错误消息，并且在所有情况下，可以明确提供适当的类型，可以使用`Mapped`类或选择性地省略内联声明。插件还需要确定关系是引用集合还是标量，为此依赖于`relationship.uselist`和/或`relationship.collection_class`参数的显式值。如果这些参数都不存在，则需要明确类型，以及如果`relationship()`的目标类型是字符串或可调用的，而不是类：

```py
class User(Base):
    __tablename__ = "user"

    id = Column(Integer, primary_key=True)
    name = Column(String)

class Address(Base):
    __tablename__ = "address"

    id = Column(Integer, primary_key=True)
    user_id: int = Column(ForeignKey("user.id"))

    user = relationship(User)
```

上述映射将产生以下错误：

```py
test3.py:22: error: [SQLAlchemy Mypy plugin] Can't infer scalar or
collection for ORM mapped expression assigned to attribute 'user'
if both 'uselist' and 'collection_class' arguments are absent from the
relationship(); please specify a type annotation on the left hand side.
Found 1 error in 1 file (checked 1 source file)
```

可以通过使用`relationship(User, uselist=False)`或提供类型来解决错误，在这种情况下是标量`User`对象：

```py
class Address(Base):
    __tablename__ = "address"

    id = Column(Integer, primary_key=True)
    user_id: int = Column(ForeignKey("user.id"))

    user: User = relationship(User)
```

对于集合，类似的模式适用，如果没有`uselist=True`或`relationship.collection_class`，可以使用`List`等集合注释。在注释中使用类的字符串名称也是完全适当的，支持 pep-484，确保类在[TYPE_CHECKING block](https://www.python.org/dev/peps/pep-0484/#runtime-or-type-checking)中适当导入：

```py
from typing import TYPE_CHECKING, List

from .mymodel import Base

if TYPE_CHECKING:
    # if the target of the relationship is in another module
    # that cannot normally be imported at runtime
    from .myaddressmodel import Address

class User(Base):
    __tablename__ = "user"

    id = Column(Integer, primary_key=True)
    name = Column(String)
    addresses: List["Address"] = relationship("Address")
```

与列一样，`Mapped`类也可以显式应用：

```py
class User(Base):
    __tablename__ = "user"

    id = Column(Integer, primary_key=True)
    name = Column(String)

    addresses: Mapped[List["Address"]] = relationship("Address", back_populates="user")

class Address(Base):
    __tablename__ = "address"

    id = Column(Integer, primary_key=True)
    user_id: int = Column(ForeignKey("user.id"))

    user: Mapped[User] = relationship(User, back_populates="addresses")
```

### 使用@declared_attr 和声明性混合

`declared_attr` 类允许在类级函数中声明声明性映射属性，并且在使用声明性混合时特别有用。对于这些函数，函数的返回类型应该使用`Mapped[]`构造进行注释，或者指示函数返回的确切对象类型。另外，未以其他方式映射的“混合”类（即不从`declarative_base()`类扩展，也不使用诸如`registry.mapped()`之类的方法进行映射）应该用`declarative_mixin()`装饰器进行装饰，这为 Mypy 插件提供了一个提示，表明特定的类打算作为声明性混合使用：

```py
from sqlalchemy.orm import declarative_mixin, declared_attr

@declarative_mixin
class HasUpdatedAt:
    @declared_attr
    def updated_at(cls) -> Column[DateTime]:  # uses Column
        return Column(DateTime)

@declarative_mixin
class HasCompany:
    @declared_attr
    def company_id(cls) -> Mapped[int]:  # uses Mapped
        return Column(ForeignKey("company.id"))

    @declared_attr
    def company(cls) -> Mapped["Company"]:
        return relationship("Company")

class Employee(HasUpdatedAt, HasCompany, Base):
    __tablename__ = "employee"

    id = Column(Integer, primary_key=True)
    name = Column(String)
```

请注意，像`HasCompany.company`这样的方法的实际返回类型与其注释之间存在不匹配。Mypy 插件将所有`@declared_attr`函数转换为简单的注释属性，以避免这种复杂性：

```py
# what Mypy sees
class HasCompany:
    company_id: Mapped[int]
    company: Mapped["Company"]
```

### 与数据类或其他类型敏感的属性系统结合

Python 数据类集成示例中的将 ORM 映射应用到现有数据类（旧数据类使用）存在一个问题；Python 数据类期望一个明确的类型，它将用于构建类，并且在每个赋值语句中给定的值是重要的。也就是说，必须准确地声明如下的类才能被数据类接受：

```py
mapper_registry: registry = registry()

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
        "properties": {"addresses": relationship("Address")}
    }
```

我们无法将我们的`Mapped[]`类型应用于属性`id`、`name`等，因为它们将被`@dataclass`装饰器拒绝。另外，Mypy 还有另一个专门针对数据类的插件，这也可能妨碍我们的操作。

上述类实际上将无障碍地通过 Mypy 的类型检查；我们唯一缺少的是`User`上属性被用于 SQL 表达式的能力，例如：

```py
stmt = select(User.name).where(User.id.in_([1, 2, 3]))
```

为此提供一种解决方案，Mypy 插件具有一个额外的功能，我们可以指定一个额外的属性`_mypy_mapped_attrs`，它是一个包含类级对象或它们的字符串名称的列表。该属性可以在`TYPE_CHECKING`变量中条件化：

```py
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
    fullname: Optional[str]
    nickname: Optional[str]
    addresses: List[Address] = field(default_factory=list)

    if TYPE_CHECKING:
        _mypy_mapped_attrs = [id, name, "fullname", "nickname", addresses]

    __mapper_args__ = {  # type: ignore
        "properties": {"addresses": relationship("Address")}
    }
```

使用上述方法，将在`_mypy_mapped_attrs`中列出的属性应用`Mapped`类型信息，以便在类绑定上下文中使用`User`类时，它将表现为 SQLAlchemy 映射类。
