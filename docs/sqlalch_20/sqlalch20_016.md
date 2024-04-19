# ORM 映射类概述

> 原文：[`docs.sqlalchemy.org/en/20/orm/mapping_styles.html`](https://docs.sqlalchemy.org/en/20/orm/mapping_styles.html)

ORM 类映射配置概述。

对于对 SQLAlchemy ORM 和/或对 Python 比较新的读者来说，建议浏览 ORM 快速入门，最好是通过 SQLAlchemy 统一教程进行学习，其中首次介绍了 ORM 配置，即使用 ORM 声明形式定义表元数据。

## ORM 映射风格

SQLAlchemy 具有两种不同的映射器配置风格，然后具有更多的子选项来设置它们。映射器风格的可变性存在是为了适应各种开发人员偏好的列表，包括用户定义的类与如何映射到关系模式表和列之间的抽象程度，正在使用的类层次结构的种类，包括是否存在自定义元类方案，最后，是否同时存在其他类实例化方法，例如是否同时使用 Python [dataclasses](https://docs.python.org/3/library/dataclasses.html)。

在现代 SQLAlchemy 中，这些风格之间的差异基本上是表面的；当使用特定的 SQLAlchemy 配置风格来表达映射类的意图时，映射类的内部映射过程大部分都是相同的，最终的结果始终是一个用户定义的类，其配置了针对可选择单元的`Mapper`，通常由`Table`对象表示，并且该类本身已经被 instrumented 以包括与关系操作相关的行为，无论是在类的级别还是在该类的实例上。由于过程在所有情况下基本上都是相同的，因此从不同风格映射的类始终是完全可互操作的。协议`MappedClassProtocol`可用于在使用诸如 mypy 等类型检查器时指示映射类。

原始的映射 API 通常被称为“经典”风格，而更自动化的映射风格称为“声明”风格。SQLAlchemy 现在将这两种映射风格称为**命令式映射**和**声明式映射**。

无论使用何种映射样式，截至 SQLAlchemy 1.4 版本，所有 ORM 映射都源自一个名为`registry`的单个对象，它是映射类的注册表。使用此注册表，一组映射器配置可以作为一个组进行最终确定，并且在特定注册表内的类可以在配置过程中相互通过名称引用。

自 1.4 版本更改：声明式和经典映射现在被称为“声明式”和“命令式”映射，并在内部统一，都源自代表一组相关映射的`registry` 构造。

### 声明式映射

**声明式映射**是现代 SQLAlchemy 中构建映射的典型方式。最常见的模式是首先使用`DeclarativeBase` 超类构建一个基类。生成的基类，当被子类化时，将对从它派生的所有子类应用声明式映射过程，相对于默认情况下新基类的本地`registry`。下面的示例演示了使用声明基类，然后在声明表映射中使用它：

```py
from sqlalchemy import Integer, String, ForeignKey
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column

# declarative base class
class Base(DeclarativeBase):
    pass

# an example mapping using the base
class User(Base):
    __tablename__ = "user"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    fullname: Mapped[str] = mapped_column(String(30))
    nickname: Mapped[Optional[str]]
```

在上面的示例中，`DeclarativeBase` 类用于生成一个新的基类（在 SQLAlchemy 文档中通常被称为 `Base`，但可以有任何所需名称），从中新类可以继承映射，如上所示，构建了一个新的映射类 `User`。

自 2.0 版本更改：`DeclarativeBase` 超类取代了`declarative_base()` 函数和`registry.generate_base()` 方法的使用；超类方法与[**PEP 484**](https://peps.python.org/pep-0484/) 工具集成，无需使用插件。有关迁移说明，请参阅 ORM 声明模型。

基类指的是维护一组相关映射类的`registry`对象，以及保留映射到类的一组`Table`对象的`MetaData`对象。

主要的声明式映射样式在以下各节中进一步详细说明：

+   使用声明基类 - 使用基类进行声明式映射。

+   使用装饰器的声明式映射（无声明基类） - 使用装饰器进行声明式映射，而不是使用基类。

在声明式映射类的范围内，`Table` 元数据的声明方式也有两种变体。包括：

+   使用 mapped_column() 的声明式表 - 在映射类内联声明表列，使用 `mapped_column()` 指令（或在传统形式中，直接使用 `Column` 对象）。`mapped_column()` 指令也可以选择性地与使用 `Mapped` 类的类型注解结合，该类可以直接提供有关映射列的一些详细信息。列指令，结合 `__tablename__` 和可选的 `__table_args__` 类级指令，将允许声明式映射过程构建要映射的 `Table` 对象。

+   具有命令式表的声明式（又名混合声明式） - 不是单独指定表名和属性，而是将显式构建的 `Table` 对象与以其他方式进行声明式映射的类关联。这种映射风格是“声明式”和“命令式”映射的混合，并适用于将类映射到反射的 `Table` 对象，以及将类映射到现有的 Core 构造，如连接和子查询。

声明式映射的文档继续在 使用声明式映射映射类 中。### 命令式映射

**命令式**或**经典**映射是指使用 `registry.map_imperatively()` 方法配置映射类的情况，其中目标类不包含任何声明类属性。

提示

命令式映射形式是 SQLAlchemy 最早发布的版本中源自的较少使用的一种映射形式。它本质上是一种绕过声明式系统提供更“基础”的映射系统的方法，并且不提供像[**PEP 484**](https://peps.python.org/pep-0484/)支持这样的现代特性。因此，大多数文档示例都使用声明式形式，并建议新用户从声明式表配置开始。

在 2.0 版本中更改：现在使用`registry.map_imperatively()`方法来创建经典映射。`sqlalchemy.orm.mapper()`独立函数已被有效移除。

在“经典”形式中，表的元数据是分别用`Table`构造创建的，然后通过`registry.map_imperatively()`方法与`User`类关联，在建立`registry`实例后。通常，一个`registry`的单个实例被共享给所有彼此相关的映射类：

```py
from sqlalchemy import Table, Column, Integer, String, ForeignKey
from sqlalchemy.orm import registry

mapper_registry = registry()

user_table = Table(
    "user",
    mapper_registry.metadata,
    Column("id", Integer, primary_key=True),
    Column("name", String(50)),
    Column("fullname", String(50)),
    Column("nickname", String(12)),
)

class User:
    pass

mapper_registry.map_imperatively(User, user_table)
```

提供关于映射属性的信息，比如与其他类的关系，通过`properties`字典提供。下面的示例说明了第二个`Table`对象，映射到一个名为`Address`的类，然后通过`relationship()`与`User`关联：

```py
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
        "addresses": relationship(Address, backref="user", order_by=address.c.id)
    },
)

mapper_registry.map_imperatively(Address, address)
```

注意，使用命令式方法映射的类与使用声明式方法映射的类**完全可以互换**。这两种系统最终都会创建相同的配置，包括一个`Table`、用户定义的类，以及一个`Mapper`对象。当我们谈论“`Mapper`的行为”时，这也包括了使用声明式系统时——它仍然被使用，只是在幕后进行。

对于所有的映射形式，可以通过传递构造参数来配置类的映射，这些构造参数最终成为`Mapper`对象的一部分，通过它的构造函数传递。传递给`Mapper`的参数来自给定的映射形式，包括传递给 Imperative 映射的`registry.map_imperatively()`的参数，或者在使用声明式系统时，来自被映射的表列、SQL 表达式和关系以及 __mapper_args__ 等属性的组合。

`Mapper`类寻找的配置信息大致可以分为四类：

### 要映射的类

这是我们应用程序中构建的类。通常情况下，这个类的结构没有限制。[[1]](#id4) 当映射一个 Python 类时，对于这个类只能有**一个**`Mapper`对象。[[2]](#id5)

当使用声明式映射样式进行映射时，要映射的类要么是声明基类的子类，要么由装饰器或函数（如`registry.mapped()`）处理。

当使用命令式映射样式进行映射时，类直接作为`map_imperatively.class_`参数传递。

### 表或其他来自子句对象

在绝大多数常见情况下，这是`Table`的实例。对于更高级的用例，它也可以指任何一种`FromClause`对象，最常见的替代对象是`Subquery`和`Join`对象。

当使用声明式映射样式时，主题表格是根据`__tablename__`属性和所提供的`Column`对象，由声明式系统生成的，或者是通过`__table__`属性建立的。这两种配置样式分别在使用 mapped_column()定义声明式表格和命令式表格与声明式（又名混合声明式）中介绍。

当使用命令式样式进行映射时，主题表格作为`map_imperatively.local_table`参数位置传递。

与映射类“每个类一个映射器”的要求相反，用于映射的`Table`或其他`FromClause`对象可以与任意数量的映射关联。`Mapper`直接对用户定义的类应用修改，但不以任何方式修改给定的`Table`或其他`FromClause`。

### 属性字典

这是与映射类关联的所有属性的字典。默认情况下，`Mapper`根据给定的`Table`从中派生出这个字典的条目，以`ColumnProperty`对象的形式表示，每个对象引用映射表的单个`Column`。属性字典还将包含所有其他类型的要配置的`MapperProperty`对象，最常见的是由`relationship()`构造生成的实例。

当使用声明式映射样式进行映射时，属性字典由声明式系统通过扫描要映射的类生成。有关此过程的说明，请参阅使用声明式定义映射属性部分。

当使用命令式风格进行映射时，属性字典直接作为`properties`参数传递给`registry.map_imperatively()`，该方法将将其传递给`Mapper.properties`参数。

### 其他映射器配置参数

当使用声明式映射风格进行映射时，额外的映射器配置参数通过`__mapper_args__`类属性进行配置。使用示例可在声明式映射器配置选项处找到。

当使用命令式风格进行映射时，关键字参数传递给`registry.map_imperatively()`方法，该方法将其传递给`Mapper`类。

可接受的所有参数范围在`Mapper`中有文档记录。## 映射类行为

使用`registry`对象进行所有映射风格时，以下行为是共同的：

### 默认构造函数

`registry`将默认构造函数，即`__init__`方法，应用于所有未明确具有自己`__init__`方法的映射类。该方法的行为是提供一个方便的关键字构造函数，将接受所有命名属性作为可选关键字参数。例如：

```py
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "user"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    fullname: Mapped[str]
```

上述`User`类型的对象将具有一个构造函数，允许像这样创建`User`对象：

```py
u1 = User(name="some name", fullname="some fullname")
```

提示

声明式数据类映射功能提供了一种通过使用 Python 数据类生成默认`__init__()`方法的替代方法，并允许高度可配置的构造函数形式。

警告

当对象在 Python 代码中构造时，仅在调用类的`__init__()`方法时才会调用`__init__()`方法，**而不是在从数据库加载或刷新对象时**。请参阅下一节在加载时保持非映射状态了解如何在加载对象时调用特殊逻辑的基础知识。

包含显式`__init__()`方法的类将保留该方法，并且不会应用默认构造函数。

要更改使用的默认构造函数，可以向`registry.constructor`参数提供用户定义的 Python 可调用对象，该对象将用作默认构造函数。

构造函数也适用于命令式映射：

```py
from sqlalchemy.orm import registry

mapper_registry = registry()

user_table = Table(
    "user",
    mapper_registry.metadata,
    Column("id", Integer, primary_key=True),
    Column("name", String(50)),
)

class User:
    pass

mapper_registry.map_imperatively(User, user_table)
```

上述类，如 命令式映射 中所述的那样被命令式映射，还将具有与 `registry` 关联的默认构造函数。

从版本 1.4 开始：经典映射现在支持通过 `registry.map_imperatively()` 方法进行映射时的标准配置级构造函数。### 在加载过程中保持非映射状态

当对象直接在 Python 代码中构造时，会调用映射类的 `__init__()` 方法：

```py
u1 = User(name="some name", fullname="some fullname")
```

但是，当使用 ORM `Session` 加载对象时，**不**会调用 `__init__()` 方法：

```py
u1 = session.scalars(select(User).where(User.name == "some name")).first()
```

这样做的原因是，从数据库加载时，用于构造对象的操作，如上例中的 `User`，更类似于**反序列化**，比如反序列化，而不是初始构造。对象的大部分重要状态不是首次组装，而是重新从数据库行加载。

因此，为了在对象内部维护不属于存储到数据库的数据的状态，使得当对象加载和构造时都存在这些状态，有两种通用方法如下所述。

1.  使用 Python 描述符，比如 `@property`，而不是状态，根据需要动态计算属性。

    对于简单的属性，这是最简单的方法，也是最不容易出错的方法。例如，如果一个对象 `Point` 有 `Point.x` 和 `Point.y`，想要一个这些属性的和的属性：

    ```py
    class Point(Base):
        __tablename__ = "point"
        id: Mapped[int] = mapped_column(primary_key=True)
        x: Mapped[int]
        y: Mapped[int]

        @property
        def x_plus_y(self):
            return self.x + self.y
    ```

    使用动态描述符的优点是值每次都会计算，这意味着它会根据底层属性（在本例中为 `x` 和 `y`）的更改来维护正确的值。

    其他形式的上述模式包括 Python 标准库[cached_property](https://docs.python.org/3/library/functools.html#functools.cached_property) 装饰器（它是缓存的，并且不会每次重新计算），以及 SQLAlchemy 的`hybrid_property` 装饰器，允许属性在 SQL 查询中使用。

1.  使用 `InstanceEvents.load()` 来在加载时建立状态，并可选地使用补充方法 `InstanceEvents.refresh()` 和 `InstanceEvents.refresh_flush()`。

    这些是在对象从数据库加载或在过期后刷新时调用的事件钩子。通常只需要`InstanceEvents.load()`，因为非映射的本地对象状态不受过期操作的影响。修改上面的`Point`示例如下所示：

    ```py
    from sqlalchemy import event

    class Point(Base):
        __tablename__ = "point"
        id: Mapped[int] = mapped_column(primary_key=True)
        x: Mapped[int]
        y: Mapped[int]

        def __init__(self, x, y, **kw):
            super().__init__(x=x, y=y, **kw)
            self.x_plus_y = x + y

    @event.listens_for(Point, "load")
    def receive_load(target, context):
        target.x_plus_y = target.x + target.y
    ```

    如果还要使用刷新事件，可以根据需要将事件钩子叠加在一个可调用对象上，如下所示：

    ```py
    @event.listens_for(Point, "load")
    @event.listens_for(Point, "refresh")
    @event.listens_for(Point, "refresh_flush")
    def receive_load(target, context, attrs=None):
        target.x_plus_y = target.x + target.y
    ```

    上面，`attrs`属性将出现在`refresh`和`refresh_flush`事件中，并指示正在刷新的属性名称列表。### 映射类、实例和映射器的运行时内省

使用`registry`映射的类也将包含一些对所有映射共通的属性：

+   `__mapper__`属性将引用与该类相关联的`Mapper`：

    ```py
    mapper = User.__mapper__
    ```

    这个`Mapper`也是使用`inspect()`函数对映射类进行检查时返回的对象：

    ```py
    from sqlalchemy import inspect

    mapper = inspect(User)
    ```

+   `__table__`属性将引用与该类映射的`Table`，或更一般地，将引用`FromClause`对象：

    ```py
    table = User.__table__
    ```

    这个`FromClause`也是在使用`Mapper.local_table`属性时返回的对象`Mapper`：

    ```py
    table = inspect(User).local_table
    ```

    对于单表继承映射，其中类是没有自己的表的子类，`Mapper.local_table`属性以及`.__table__`属性将为`None`。要检索在查询此类时实际选择的“可选择”对象，可以通过`Mapper.selectable`属性获取：

    ```py
    table = inspect(User).selectable
    ```

#### 映射器对象的检查

如前一节所示，无论使用何种方法，`Mapper`对象都可以从任何映射类中获取，使用运行时检查 API 系统。使用`inspect()`函数，可以从映射类获取`Mapper`：

```py
>>> from sqlalchemy import inspect
>>> insp = inspect(User)
```

可用的详细信息包括`Mapper.columns`：

```py
>>> insp.columns
<sqlalchemy.util._collections.OrderedProperties object at 0x102f407f8>
```

这是一个可以以列表格式或单个名称查看的命名空间：

```py
>>> list(insp.columns)
[Column('id', Integer(), table=<user>, primary_key=True, nullable=False), Column('name', String(length=50), table=<user>), Column('fullname', String(length=50), table=<user>), Column('nickname', String(length=50), table=<user>)]
>>> insp.columns.name
Column('name', String(length=50), table=<user>)
```

其他命名空间包括`Mapper.all_orm_descriptors`，其中包括所有映射属性以及混合属性，关联代理：

```py
>>> insp.all_orm_descriptors
<sqlalchemy.util._collections.ImmutableProperties object at 0x1040e2c68>
>>> insp.all_orm_descriptors.keys()
['fullname', 'nickname', 'name', 'id']
```

以及`Mapper.column_attrs`：

```py
>>> list(insp.column_attrs)
[<ColumnProperty at 0x10403fde0; id>, <ColumnProperty at 0x10403fce8; name>, <ColumnProperty at 0x1040e9050; fullname>, <ColumnProperty at 0x1040e9148; nickname>]
>>> insp.column_attrs.name
<ColumnProperty at 0x10403fce8; name>
>>> insp.column_attrs.name.expression
Column('name', String(length=50), table=<user>)
```

另请参阅

`Mapper`  #### Inspection of Mapped Instances

`inspect()`函数还提供有关映射类的实例的信息。当应用于映射类的实例时，而不是类本身时，返回的对象被称为`InstanceState`，它将提供链接到不仅是类使用的`Mapper`的详细接口，还提供有关实例内个别属性状态的信息，包括它们当前的值以及这如何与它们的数据库加载值相关联。

给定从数据库加载的`User`类的实例：

```py
>>> u1 = session.scalars(select(User)).first()
```

`inspect()`函数会返回一个`InstanceState`对象给我们：

```py
>>> insp = inspect(u1)
>>> insp
<sqlalchemy.orm.state.InstanceState object at 0x7f07e5fec2e0>
```

通过这个对象，我们可以看到诸如`Mapper`等元素：

```py
>>> insp.mapper
<Mapper at 0x7f07e614ef50; User>
```

对象所附加到的`Session`（如果有的话）：

```py
>>> insp.session
<sqlalchemy.orm.session.Session object at 0x7f07e614f160>
```

关于对象的当前 persistence state 的信息：

```py
>>> insp.persistent
True
>>> insp.pending
False
```

属性状态信息，如尚未加载或延迟加载的属性（假设`addresses`指的是映射类上到相关类的`relationship()`）：

```py
>>> insp.unloaded
{'addresses'}
```

有关属性的当前 Python 状态的信息，例如自上次刷新以来未经修改的属性：

```py
>>> insp.unmodified
{'nickname', 'name', 'fullname', 'id'}
```

以及自上次刷新以来对属性进行的修改的特定历史记录：

```py
>>> insp.attrs.nickname.value
'nickname'
>>> u1.nickname = "new nickname"
>>> insp.attrs.nickname.history
History(added=['new nickname'], unchanged=(), deleted=['nickname'])
```

另请参阅

`InstanceState`

`InstanceState.attrs`

`AttributeState`  ## ORM Mapping Styles

SQLAlchemy 提供了两种不同风格的映射配置，然后进一步提供了设置它们的子选项。映射样式的可变性存在是为了适应开发者偏好的多样性，包括用户定义类与如何映射到关系模式表和列之间的抽象程度，使用的类层次结构种类，包括是否存在自定义元类方案，以及是否同时使用了其他类内部操作方法，例如是否同时使用了 Python [dataclasses](https://docs.python.org/3/library/dataclasses.html)。

在现代 SQLAlchemy 中，这些风格之间的区别主要是表面的；当使用特定的 SQLAlchemy 配置风格来表达映射类的意图时，映射类的内部映射过程在大多数情况下是相同的，最终的结果总是一个用户定义的类，该类已经针对可选择的单元（通常由一个`Table`对象表示）配置了一个`Mapper`，并且该类本身已经被 instrumented 以包括与关系操作相关的行为，既在类的级别，也在该类的实例上。由于在所有情况下该过程基本相同，从不同风格映射的类始终是完全可互操作的。当使用诸如 mypy 等类型检查器时，可以使用协议`MappedClassProtocol`来指示已映射的类。

最初的映射 API 通常被称为“古典”风格，而更自动化的映射风格则被称为“声明式”风格。SQLAlchemy 现在将这两种映射风格称为**命令式映射**和**声明式映射**。

无论使用何种映射样式，自 SQLAlchemy 1.4 起，所有 ORM 映射都源自一个名为`registry`的单个对象，它是一组映射类的注册表。使用此注册表，一组映射配置可以作为一个组完成，并且在配置过程中，特定注册表中的类可以通过名称相互引用。

在 1.4 版本中更改：声明式和古典映射现在被称为“声明式”和“命令式”映射，并在内部统一，所有这些都源自代表相关映射的`registry` 构造。

### 声明式映射

**Declarative Mapping** 是在现代 SQLAlchemy 中构建映射的典型方式。最常见的模式是首先使用 `DeclarativeBase` 超类构造一个基类。结果基类，当被子类继承时，将对所有从它继承的子类应用声明式映射过程，相对于默认情况下新基类的本地 `registry`。下面的示例说明了使用声明基类然后在声明式表映射中使用它的方法：

```py
from sqlalchemy import Integer, String, ForeignKey
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column

# declarative base class
class Base(DeclarativeBase):
    pass

# an example mapping using the base
class User(Base):
    __tablename__ = "user"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    fullname: Mapped[str] = mapped_column(String(30))
    nickname: Mapped[Optional[str]]
```

上文中，`DeclarativeBase` 类用于生成一个新的基类（在 SQLAlchemy 的文档中通常称为 `Base`，但可以使用任何想要的名称），新的映射类可以从中继承，就像上面构造了一个新的映射类 `User` 一样。

从 2.0 版本开始更改：`DeclarativeBase` 超类取代了 `declarative_base()` 函数和 `registry.generate_base()` 方法的使用；超类方法与 [**PEP 484**](https://peps.python.org/pep-0484/) 工具集成，无需使用插件。有关迁移说明，请参阅 ORM Declarative Models。

基类指的是一个维护一组相关映射类的 `registry` 对象，以及一个保留了一组将类映射到其中的 `Table` 对象的 `MetaData` 对象。

主要的 Declarative 映射风格在以下各节中进一步详细说明：

+   使用声明基类 - 使用基类进行声明式映射。

+   使用装饰器进行声明式映射（无声明基类） - 使用装饰器进行声明式映射，而不是使用基类。

在 Declarative 映射类的范围内，还有两种声明 `Table` 元数据的方式。它们包括：

+   带`mapped_column()`的声明式表 - 在映射类内联声明表列，使用`mapped_column()`指令（或者在遗留形式中，直接使用`Column`对象）。`mapped_column()`指令也可以选择性地与类型注解结合使用，使用`Mapped`类可以直接提供有关映射列的一些细节。列指令与`__tablename__`和可选的`__table_args__`类级指令的组合将允许声明式映射过程构造一个要映射的`Table`对象。

+   声明式与命令式表（也称为混合声明式） - 不是分别指定表名和属性，而是将明确构造的`Table`对象与否则以声明方式映射的类相关联。这种映射风格是“声明式”和“命令式”映射的混合体，并适用于将类映射到反射的`Table`对象，以及将类映射到现有 Core 构造，如连接和子查询。

声明式映射的文档继续在使用声明性映射类 ### 命令式映射

**命令式**或**经典**映射是指使用`registry.map_imperatively()`方法配置映射类的配置，其中目标类不包含任何声明式类属性。

提示

命令式映射形式是 SQLAlchemy 在 2006 年的最初版本中少用的一种映射形式。它本质上是绕过声明式系统提供一种更“精简”的映射系统，不提供现代特性，如[**PEP 484**](https://peps.python.org/pep-0484/)支持。因此，大多数文档示例使用声明式形式，并建议新用户从声明式表配置开始。

在 2.0 版中更改：现在使用`registry.map_imperatively()`方法创建经典映射。`sqlalchemy.orm.mapper()`独立函数被有效删除。

在“经典”形式中，表元数据是分别使用`Table`构造创建的，然后通过`registry.map_imperatively()`方法与`User`类关联，在建立`registry`实例之后。通常，一个`registry`的单个实例共享所有彼此相关的映射类:

```py
from sqlalchemy import Table, Column, Integer, String, ForeignKey
from sqlalchemy.orm import registry

mapper_registry = registry()

user_table = Table(
    "user",
    mapper_registry.metadata,
    Column("id", Integer, primary_key=True),
    Column("name", String(50)),
    Column("fullname", String(50)),
    Column("nickname", String(12)),
)

class User:
    pass

mapper_registry.map_imperatively(User, user_table)
```

关于映射属性的信息，例如与其他类的关系，通过`properties`字典提供。下面的示例说明了第二个`Table`对象，映射到名为`Address`的类，然后通过`relationship()`链接到`User`:

```py
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
        "addresses": relationship(Address, backref="user", order_by=address.c.id)
    },
)

mapper_registry.map_imperatively(Address, address)
```

请注意，使用命令式方法映射的类与使用声明式方法映射的类**完全可互换**。两个系统最终创建相同的配置，由一个`Table`、用户定义类和一个`Mapper`对象组成。当我们谈论“`Mapper`的行为”时，这也包括在使用声明式系统时 - 它仍然被使用，只是在幕后。 ### 声明式映射

**声明式映射**是在现代 SQLAlchemy 中构建映射的典型方式。最常见的模式是首先使用`DeclarativeBase`超类构造基类。生成的基类，在其派生的所有子类中应用声明式映射过程，相对于一个默认情况下局部于新基类的`registry`。下面的示例说明了使用声明基类的情况，然后在声明表映射中使用它:

```py
from sqlalchemy import Integer, String, ForeignKey
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column

# declarative base class
class Base(DeclarativeBase):
    pass

# an example mapping using the base
class User(Base):
    __tablename__ = "user"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    fullname: Mapped[str] = mapped_column(String(30))
    nickname: Mapped[Optional[str]]
```

上面，`DeclarativeBase`类用于生成一个新的基类（在 SQLAlchemy 的文档中通常称为`Base`，但可以有任何所需的名称），从中新映射类`User`构造。

从版本 2.0 开始更改：`DeclarativeBase`超类取代了`declarative_base()`函数和`registry.generate_base()`方法的使用；超类方法集成了[**PEP 484**](https://peps.python.org/pep-0484/)工具，无需使用插件。有关迁移说明，请参阅 ORM 声明性模型。

基类指的是维护一组相关映射类的`registry`对象，以及维护一组映射到这些类的`Table`对象的`MetaData`对象。

主要的声明性映射样式在以下各节中进一步详细说明：

+   使用声明性基类 - 使用基类的声明性映射。

+   使用装饰器进行声明性映射（无声明性基类） - 使用装饰器而不是基类的声明性映射。

在声明性映射类的范围内，还有两种`Table`元数据声明的方式。这些包括：

+   使用`mapped_column()`声明的声明性表格 - 表格列在映射类中使用`mapped_column()`指令内联声明（或者在传统形式中，直接使用`Column`对象）。`mapped_column()`指令还可以选择性地与使用`Mapped`类进行类型注释，该类可以直接提供有关映射列的一些详细信息相结合。列指令与`__tablename__`以及可选的`__table_args__`类级别指令的组合将允许声明性映射过程构造一个要映射的`Table`对象。

+   声明式与命令式表格（即混合声明式） - 不是单独指定表名和属性，而是将显式构建的`Table`对象与在其他情况下以声明方式映射的类关联起来。这种映射方式是“声明式”和“命令式”映射的混合体，适用于诸如将类映射到反射的`Table`对象，以及将类映射到现有的 Core 构造，如联接和子查询的技术。

声明式映射的文档继续在用声明式映射类中。

### 命令式映射

**命令式**或**经典**映射是指使用`registry.map_imperatively()`方法配置映射类的一种方法，其中目标类不包含任何声明式类属性。

提示

命令式映射形式是 SQLAlchemy 最早期发布的较少使用的映射形式。它基本上是绕过声明式系统提供更“简化”的映射系统，并且不提供现代特性，例如[**PEP 484**](https://peps.python.org/pep-0484/)支持。因此，大多数文档示例使用声明式形式，建议新用户从声明式表格配置开始。

在 2.0 版更改：`registry.map_imperatively()` 方法现在用于创建经典映射。`sqlalchemy.orm.mapper()` 独立函数已被有效移除。

在“经典”形式中，表元数据是使用`Table`构造单独创建的，然后通过`registry.map_imperatively()`方法与`User`类关联，在建立 `registry` 实例后。通常，一个`registry`实例共享给所有彼此相关的映射类：

```py
from sqlalchemy import Table, Column, Integer, String, ForeignKey
from sqlalchemy.orm import registry

mapper_registry = registry()

user_table = Table(
    "user",
    mapper_registry.metadata,
    Column("id", Integer, primary_key=True),
    Column("name", String(50)),
    Column("fullname", String(50)),
    Column("nickname", String(12)),
)

class User:
    pass

mapper_registry.map_imperatively(User, user_table)
```

映射属性的信息，如与其他类的关系，通过`properties`字典提供。下面的示例说明了第二个`Table`对象，映射到名为`Address`的类，然后通过`relationship()`与`User`关联：

```py
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
        "addresses": relationship(Address, backref="user", order_by=address.c.id)
    },
)

mapper_registry.map_imperatively(Address, address)
```

注意，使用命令式方法映射的类与使用声明式方法映射的类**完全可互换**。这两种系统最终都创建相同的配置，包括一个由`Table`、用户定义类和一个与之关联的`Mapper`对象组成的配置。当我们谈论“`Mapper`的行为”时，这也包括使用声明式系统 - 它仍然在幕后使用。

## 映射类的基本组件

通过所有映射形式，通过传递最终成为`Mapper`对象的构造参数，可以通过多种方式配置类的映射。传递给`Mapper`的参数来自给定的映射形式，包括传递给`registry.map_imperatively()`的参数，用于命令式映射，或者使用声明式系统时，来自与被映射的表列、SQL 表达式和关系相关联的参数以及属性的参数，如 __mapper_args__。

`Mapper`类寻找的四类常规配置信息如下：

### 待映射的类

这是我们在应用程序中构造的类。通常情况下，这个类的结构没有限制。[[1]](#id4)当一个 Python 类被映射时，该类只能有**一个**`Mapper`对象。[[2]](#id5)

当使用声明式映射风格时，要映射的类要么是声明基类的子类，要么由装饰器或函数处理，如`registry.mapped()`。

当使用命令式映射风格时，类直接作为`map_imperatively.class_`参数传递。

### 表格或其他来源子句对象

在绝大多数常见情况下，这是`Table`的实例。对于更高级的用例，它还可以指代任何一种`FromClause`对象，最常见的替代对象是`Subquery`和`Join`对象。

当使用声明性映射样式进行映射时，主题表格要么是由声明性系统基于`__tablename__`属性和所呈现的`Column`对象生成的，要么是通过`__table__`属性建立的。这两种配置样式分别在具有映射列的声明性表格和具有命令式表格的声明性（又名混合声明性）中呈现。

当使用命令式样式进行映射时，主题表格作为`map_imperatively.local_table`参数按位置传递。

与映射类的“每个类一个映射器”的要求相反，作为映射主题的`Table`或其他`FromClause`对象可以与任意数量的映射关联。`Mapper`直接对用户定义的类进行修改，但不以任何方式修改给定的`Table`或其他`FromClause`。

### 属性字典

这是将与映射类关联的所有属性关联起来的字典。默认情况下，`Mapper` 从给定的`Table`生成此字典的条目，形式为每个都引用映射表的单个`Column`的`ColumnProperty`对象。属性字典还将包含所有其他需要配置的`MapperProperty`对象，最常见的是通过`relationship()`构造函数生成的实例。

当使用声明式映射样式进行映射时，属性字典是由声明式系统通过扫描要映射的类以获取适当属性而生成的。请参阅使用声明式定义映射属性部分以获取有关此过程的说明。

当使用命令式映射样式进行映射时，属性字典直接作为`properties`参数传递给`registry.map_imperatively()`，该参数将其传递给`Mapper.properties`参数。

### 其他映射器配置参数

当使用声明式映射样式进行映射时，附加的映射器配置参数通过`__mapper_args__`类属性进行配置。使用示例请参见使用声明式定义的映射器配置选项。

当使用命令式映射样式进行映射时，关键字参数传递给`registry.map_imperatively()`方法，该方法将其传递给`Mapper`类。

接受的全部参数范围在`Mapper`中有文档记录。

### 要映射的类

这是我们应用程序中构建的一个类。通常情况下，此类的结构没有任何限制。[[1]](#id4) 当映射 Python 类时，该类只能有**一个**`Mapper`对象。[[2]](#id5)

当使用声明式映射风格进行映射时，要映射的类是声明基类的子类，或者由装饰器或函数（如`registry.mapped()`）处理。

当使用命令式风格进行映射时，类直接作为`map_imperatively.class_`参数传递。

### 表或其他来自子句对象

在绝大多数常见情况下，这是一个`Table`的实例。对于更高级的用例，它也可能指的是任何类型的`FromClause`对象，最常见的替代对象是`Subquery`和`Join`对象。

当使用声明式映射风格进行映射时，主题表通过声明系统基于`__tablename__`属性和提供的`Column`对象生成，或者通过`__table__`属性建立。这两种配置样式在使用 mapped_column() 进行声明性表配置和具有命令式表的声明式（也称为混合声明式）中介绍。

当使用命令式风格进行映射时，主题表作为`map_imperatively.local_table`参数按位置传递。

与映射类“每个类一个映射器”的要求相反，映射的`Table`或其他`FromClause`对象可以与任意数量的映射相关联。`Mapper`直接将修改应用于用户定义的类，但不以任何方式修改给定的`Table`或其他`FromClause`。

### 属性字典

这是一个与映射类相关联的所有属性的字典。默认情况下，`Mapper`从给定的`Table`中派生此字典的条目，形成每个映射表的`Column`的`ColumnProperty`对象。属性字典还将包含要配置的所有其他种类的`MapperProperty`对象，最常见的是由`relationship()`构造生成的实例。

当使用声明性映射风格进行映射时，属性字典由声明性系统通过扫描要映射的类以找到合适的属性而生成。有关此过程的说明，请参见使用声明性定义映射属性部分。

当使用命令式映射风格进行映射时，属性字典直接作为`properties`参数传递给`registry.map_imperatively()`，它将把它传递给`Mapper.properties`参数。

### 其他映射器配置参数

当使用声明性映射风格进行映射时，额外的映射器配置参数通过`__mapper_args__`类属性配置。有关用法示例，请参阅使用声明性配置选项的映射器。

当使用命令式映射风格进行映射时，关键字参数传递给`registry.map_imperatively()`方法，该方法将它们传递给`Mapper`类。

接受的参数的完整范围在`Mapper`中有文档记录。

## 映射类行为

在使用`registry`对象进行所有映射样式时，以下行为是共同的：

### 默认构造函数

`registry`将默认构造函数，即`__init__`方法，应用于所有没有明确自己的`__init__`方法的映射类。此方法的行为是提供一个方便的关键字构造函数，将接受所有命名属性作为可选关键字参数。例如：

```py
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "user"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    fullname: Mapped[str]
```

上面的`User`类型对象将具有允许创建`User`对象的构造函数，如下所示：

```py
u1 = User(name="some name", fullname="some fullname")
```

提示

声明式数据类映射功能通过使用 Python 数据类提供了一种生成默认`__init__()`方法的替代方法，并且允许高度可配置的构造函数形式。

警告

类的`__init__()`方法仅在 Python 代码中构造对象时调用，**而不是在从数据库加载或刷新对象时调用**。请参阅下一节在加载过程中保持非映射状态，了解如何在加载对象时调用特殊逻辑的入门知识。

包含显式`__init__()`方法的类将保留该方法，并且不会应用默认构造函数。

要更改所使用的默认构造函数，可以向`registry.constructor`参数提供用户定义的 Python 可调用对象，该对象将用作默认构造函数。

构造函数也适用于命令式映射：

```py
from sqlalchemy.orm import registry

mapper_registry = registry()

user_table = Table(
    "user",
    mapper_registry.metadata,
    Column("id", Integer, primary_key=True),
    Column("name", String(50)),
)

class User:
    pass

mapper_registry.map_imperatively(User, user_table)
```

如上所述，通过命令式映射描述的类也将具有与`registry`相关联的默认构造函数。

新版 1.4 中：经典映射现在在通过`registry.map_imperatively()`方法进行映射时支持标准配置级别的构造函数。### 在加载过程中保持非映射状态

映射类的`__init__()`方法在 Python 代码中直接构造对象时被调用：

```py
u1 = User(name="some name", fullname="some fullname")
```

然而，当使用 ORM `Session`加载对象时，**不会**调用`__init__()`方法：

```py
u1 = session.scalars(select(User).where(User.name == "some name")).first()
```

这是因为从数据库加载时，用于构造对象的操作（在上面的示例中为`User`）更类似于**反序列化**，如取消持久性，而不是初始构造。大多数对象的重要状态不是首次组装，而是从数据库行重新加载。

因此，为了在对象中维护不是数据库中存储的数据的状态，使得当对象被加载和构造时此状态存在，下面详细介绍了两种一般方法。

1.  使用 Python 描述符（如 `@property`），而不是状态，根据需要动态计算属性。

    对于简单的属性，这是最简单且最不容易出错的方法。例如，如果一个名为 `Point` 的对象希望具有这些属性的总和： 

    ```py
    class Point(Base):
        __tablename__ = "point"
        id: Mapped[int] = mapped_column(primary_key=True)
        x: Mapped[int]
        y: Mapped[int]

        @property
        def x_plus_y(self):
            return self.x + self.y
    ```

    使用动态描述符的优势在于，值每次都会重新计算，这意味着它会随着基础属性（在本例中为 `x` 和 `y`）可能会发生变化而保持正确的值。

    上述模式的其他形式包括 Python 标准库的 [cached_property](https://docs.python.org/3/library/functools.html#functools.cached_property) 装饰器（它被缓存，而不是每次重新计算），以及 SQLAlchemy 的 `hybrid_property` 装饰器，它允许属性既可用于 SQL 查询，也可用于 Python 属性。

1.  使用 `InstanceEvents.load()` 建立加载时的状态，并且可选地使用补充方法 `InstanceEvents.refresh()` 和 `InstanceEvents.refresh_flush()`。

    这些是在从数据库加载对象或在对象过期后刷新时调用的事件钩子。通常只需要 `InstanceEvents.load()`，因为非映射的本地对象状态不受过期操作的影响。要修改上面的 `Point` 示例，如下所示：

    ```py
    from sqlalchemy import event

    class Point(Base):
        __tablename__ = "point"
        id: Mapped[int] = mapped_column(primary_key=True)
        x: Mapped[int]
        y: Mapped[int]

        def __init__(self, x, y, **kw):
            super().__init__(x=x, y=y, **kw)
            self.x_plus_y = x + y

    @event.listens_for(Point, "load")
    def receive_load(target, context):
        target.x_plus_y = target.x + target.y
    ```

    如果还使用刷新事件，事件钩子可以根据需要堆叠在一个可调用对象上，如下所示：

    ```py
    @event.listens_for(Point, "load")
    @event.listens_for(Point, "refresh")
    @event.listens_for(Point, "refresh_flush")
    def receive_load(target, context, attrs=None):
        target.x_plus_y = target.x + target.y
    ```

    在上述情况下，`attrs` 属性将出现在 `refresh` 和 `refresh_flush` 事件中，并指示正在刷新的属性名称列表。### 映射类、实例和映射器的运行时内省

使用 `registry` 映射的类还将包含一些对所有映射通用的属性：

+   `__mapper__` 属性将引用与类相关联的 `Mapper`：

    ```py
    mapper = User.__mapper__
    ```

    当对映射类使用 `inspect()` 函数时，返回的也是此 `Mapper`：

    ```py
    from sqlalchemy import inspect

    mapper = inspect(User)
    ```

+   `__table__` 属性将引用类被映射到的 `Table`，或者更通用地引用类被映射到的 `FromClause` 对象：

    ```py
    table = User.__table__
    ```

    当使用 `Mapper.local_table` 属性时，返回的也是这个 `FromClause`：

    ```py
    table = inspect(User).local_table
    ```

    对于单表继承映射，其中类是没有自己的表的子类，`Mapper.local_table` 属性以及 `.__table__` 属性将为 `None`。要检索在查询此类时实际选择的“可选择项”，可通过 `Mapper.selectable` 属性获得：

    ```py
    table = inspect(User).selectable
    ```

#### Mapper 对象的检查

如前一节所示，`Mapper` 对象可从任何映射类获得，而不管方法如何，使用 Runtime Inspection API 系统。使用 `inspect()` 函数，可以从映射类获取 `Mapper`：

```py
>>> from sqlalchemy import inspect
>>> insp = inspect(User)
```

可用的详细信息包括 `Mapper.columns`：

```py
>>> insp.columns
<sqlalchemy.util._collections.OrderedProperties object at 0x102f407f8>
```

这是一个可以以列表格式或通过单个名称查看的命名空间：

```py
>>> list(insp.columns)
[Column('id', Integer(), table=<user>, primary_key=True, nullable=False), Column('name', String(length=50), table=<user>), Column('fullname', String(length=50), table=<user>), Column('nickname', String(length=50), table=<user>)]
>>> insp.columns.name
Column('name', String(length=50), table=<user>)
```

其他命名空间包括 `Mapper.all_orm_descriptors`，其中包括所有映射属性以及混合属性、关联代理：

```py
>>> insp.all_orm_descriptors
<sqlalchemy.util._collections.ImmutableProperties object at 0x1040e2c68>
>>> insp.all_orm_descriptors.keys()
['fullname', 'nickname', 'name', 'id']
```

以及 `Mapper.column_attrs`：

```py
>>> list(insp.column_attrs)
[<ColumnProperty at 0x10403fde0; id>, <ColumnProperty at 0x10403fce8; name>, <ColumnProperty at 0x1040e9050; fullname>, <ColumnProperty at 0x1040e9148; nickname>]
>>> insp.column_attrs.name
<ColumnProperty at 0x10403fce8; name>
>>> insp.column_attrs.name.expression
Column('name', String(length=50), table=<user>)
```

另请参阅

`Mapper`  #### 映射实例的检查

`inspect()` 函数还提供关于映射类的实例的信息。当应用于映射类的实例时，而不是类本身时，返回的对象被称为 `InstanceState`，它将提供链接，不仅链接到类使用的 `Mapper`，还提供了一个详细的界面，提供了关于实例内部属性状态的信息，包括它们当前的值以及这与它们的数据库加载值有何关系。

给定从数据库加载的 `User` 类的实例：

```py
>>> u1 = session.scalars(select(User)).first()
```

`inspect()` 函数将返回给我们一个 `InstanceState` 对象：

```py
>>> insp = inspect(u1)
>>> insp
<sqlalchemy.orm.state.InstanceState object at 0x7f07e5fec2e0>
```

通过该对象，我们可以查看诸如 `Mapper` 等元素：

```py
>>> insp.mapper
<Mapper at 0x7f07e614ef50; User>
```

对象所附属的 `Session`（如果有）：

```py
>>> insp.session
<sqlalchemy.orm.session.Session object at 0x7f07e614f160>
```

对象的当前持久状态的信息：

```py
>>> insp.persistent
True
>>> insp.pending
False
```

属性状态信息，例如未加载或延迟加载的属性（假设 `addresses` 是映射类到相关类的 `relationship()`）：

```py
>>> insp.unloaded
{'addresses'}
```

关于当前 Python 中属性的状态信息，例如自上次刷新以来未修改的属性：

```py
>>> insp.unmodified
{'nickname', 'name', 'fullname', 'id'}
```

以及自上次刷新以来对属性进行修改的具体历史：

```py
>>> insp.attrs.nickname.value
'nickname'
>>> u1.nickname = "new nickname"
>>> insp.attrs.nickname.history
History(added=['new nickname'], unchanged=(), deleted=['nickname'])
```

另请参阅

`InstanceState`

`InstanceState.attrs`

`AttributeState`  ### 默认构造函数

`registry` 对所有未显式拥有自己 `__init__` 方法的映射类应用默认构造函数，即 `__init__` 方法。该方法的行为是提供一个方便的关键字构造函数，将接受所有命名属性作为可选关键字参数。例如：

```py
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "user"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    fullname: Mapped[str]
```

上面的 `User` 类的对象将具有一个允许创建 `User` 对象的构造函数：

```py
u1 = User(name="some name", fullname="some fullname")
```

提示

声明式数据类映射 功能通过使用 Python 数据类提供了一种生成默认 `__init__()` 方法的替代方式，并允许高度可配置的构造函数形式。

警告

当对象在 Python 代码中构造时才调用类的 `__init__()` 方法，而**不是在从数据库加载或刷新对象时**。请参阅下一节在加载时保持非映射状态，了解如何在加载对象时调用特殊逻辑的基本知识。

包含显式 `__init__()` 方法的类将保持该方法，不会应用默认构造函数。

若要更改使用的默认构造函数，可以提供用户定义的 Python 可调用对象给 `registry.constructor` 参数，该对象将用作默认构造函数。

构造函数也适用于命令式映射：

```py
from sqlalchemy.orm import registry

mapper_registry = registry()

user_table = Table(
    "user",
    mapper_registry.metadata,
    Column("id", Integer, primary_key=True),
    Column("name", String(50)),
)

class User:
    pass

mapper_registry.map_imperatively(User, user_table)
```

如在命令式映射中所述，上述类将还具有与`registry`相关联的默认构造函数。

新版本 1.4 中：经典映射现在支持通过`registry.map_imperatively()`方法映射时的标准配置级构造函数。

### 在加载期间保持非映射状态

当直接在 Python 代码中构造对象时，会调用映射类的`__init__()`方法：

```py
u1 = User(name="some name", fullname="some fullname")
```

然而，当使用 ORM `Session`加载对象时，**不会**调用`__init__()`方法：

```py
u1 = session.scalars(select(User).where(User.name == "some name")).first()
```

原因在于，当从数据库加载时，用于构造对象的操作，例如上面的`User`，更类似于**反序列化**，例如取消选中，而不是初始构造。对象的大部分重要状态不是首次组装的，而是重新从数据库行加载的。

因此，为了在对象加载以及构造时保持对象中不是存储到数据库的数据的状态，以下详细介绍了两种一般方法。

1.  使用 Python 描述符，如`@property`，而不是状态，根据需要动态计算属性。

    对于简单属性，这是最简单且最少错误的方法。例如，如果具有`Point.x`和`Point.y`的对象`Point`希望具有这些属性的和：

    ```py
    class Point(Base):
        __tablename__ = "point"
        id: Mapped[int] = mapped_column(primary_key=True)
        x: Mapped[int]
        y: Mapped[int]

        @property
        def x_plus_y(self):
            return self.x + self.y
    ```

    使用动态描述符的优点是值每次计算，这意味着它保持正确的值，因为底层属性（在本例中为`x`和`y`）可能会更改。

    上述模式的其他形式包括 Python 标准库[cached_property](https://docs.python.org/3/library/functools.html#functools.cached_property)装饰器（它是缓存的，不会每次重新计算），以及 SQLAlchemy 的`hybrid_property`装饰器，允许属性同时适用于 SQL 查询。

1.  使用`InstanceEvents.load()`来在加载时建立状态，可选地使用补充方法`InstanceEvents.refresh()`和`InstanceEvents.refresh_flush()`。

    这些是在对象从数据库加载时或在过期后刷新时调用的事件钩子。通常只需要 `InstanceEvents.load()`，因为非映射的本地对象状态不受到过期操作的影响。要修改上面的 `Point` 示例，看起来像这样：

    ```py
    from sqlalchemy import event

    class Point(Base):
        __tablename__ = "point"
        id: Mapped[int] = mapped_column(primary_key=True)
        x: Mapped[int]
        y: Mapped[int]

        def __init__(self, x, y, **kw):
            super().__init__(x=x, y=y, **kw)
            self.x_plus_y = x + y

    @event.listens_for(Point, "load")
    def receive_load(target, context):
        target.x_plus_y = target.x + target.y
    ```

    如果需要同时使用刷新事件，事件钩子可以叠加在一个可调用对象上，如下所示：

    ```py
    @event.listens_for(Point, "load")
    @event.listens_for(Point, "refresh")
    @event.listens_for(Point, "refresh_flush")
    def receive_load(target, context, attrs=None):
        target.x_plus_y = target.x + target.y
    ```

    在上面的示例中，`attrs` 属性将出现在 `refresh` 和 `refresh_flush` 事件中，并指示正在刷新的属性名称列表。

### 映射类、实例和映射器的运行时内省

使用 `registry` 进行映射的类还将具有一些所有映射的共同属性：

+   `__mapper__` 属性将引用与该类关联的 `Mapper`：

    ```py
    mapper = User.__mapper__
    ```

    当使用 `inspect()` 函数对映射类进行检查时，也将返回此 `Mapper`：

    ```py
    from sqlalchemy import inspect

    mapper = inspect(User)
    ```

+   `__table__` 属性将引用将类映射到的 `Table`，或更一般地，引用 `FromClause` 对象：

    ```py
    table = User.__table__
    ```

    当使用 `Mapper.local_table` 属性时，此 `FromClause` 也将返回：

    ```py
    table = inspect(User).local_table
    ```

    对于单表继承映射，其中类是没有自己的表的子类，`Mapper.local_table` 属性以及 `.__table__` 属性都将为 `None`。要检索在查询此类时实际选择的“可选项”，可以通过 `Mapper.selectable` 属性获取：

    ```py
    table = inspect(User).selectable
    ```

#### 映射器对象的检查

如前一节所示，`Mapper` 对象可从任何映射类中使用 运行时内省 API 系统获取。使用 `inspect()` 函数，可以从映射类中获取 `Mapper`：

```py
>>> from sqlalchemy import inspect
>>> insp = inspect(User)
```

可用的详细信息包括 `Mapper.columns`:

```py
>>> insp.columns
<sqlalchemy.util._collections.OrderedProperties object at 0x102f407f8>
```

这是一个可以以列表格式或通过单个名称查看的命名空间：

```py
>>> list(insp.columns)
[Column('id', Integer(), table=<user>, primary_key=True, nullable=False), Column('name', String(length=50), table=<user>), Column('fullname', String(length=50), table=<user>), Column('nickname', String(length=50), table=<user>)]
>>> insp.columns.name
Column('name', String(length=50), table=<user>)
```

其他命名空间包括 `Mapper.all_orm_descriptors`，其中包括所有映射属性以及混合体，关联代理：

```py
>>> insp.all_orm_descriptors
<sqlalchemy.util._collections.ImmutableProperties object at 0x1040e2c68>
>>> insp.all_orm_descriptors.keys()
['fullname', 'nickname', 'name', 'id']
```

以及 `Mapper.column_attrs`:

```py
>>> list(insp.column_attrs)
[<ColumnProperty at 0x10403fde0; id>, <ColumnProperty at 0x10403fce8; name>, <ColumnProperty at 0x1040e9050; fullname>, <ColumnProperty at 0x1040e9148; nickname>]
>>> insp.column_attrs.name
<ColumnProperty at 0x10403fce8; name>
>>> insp.column_attrs.name.expression
Column('name', String(length=50), table=<user>)
```

另请参阅

`Mapper`  #### 映射实例的检查

`inspect()` 函数还提供了关于映射类的实例的信息。当应用于映射类的实例而不是类本身时，返回的对象被称为 `InstanceState`，它将提供链接到不仅由该类使用的 `Mapper`，还提供了有关实例内部属性状态的详细接口的信息，包括它们的当前值以及这与它们的数据库加载值的关系。

给定从数据库加载的 `User` 类的实例：

```py
>>> u1 = session.scalars(select(User)).first()
```

`inspect()` 函数将返回一个 `InstanceState` 对象：

```py
>>> insp = inspect(u1)
>>> insp
<sqlalchemy.orm.state.InstanceState object at 0x7f07e5fec2e0>
```

使用此对象，我们可以查看诸如 `Mapper` 之类的元素：

```py
>>> insp.mapper
<Mapper at 0x7f07e614ef50; User>
```

对象所附加到的 `Session`（如果有）：

```py
>>> insp.session
<sqlalchemy.orm.session.Session object at 0x7f07e614f160>
```

关于对象当前的持久性状态的信息：

```py
>>> insp.persistent
True
>>> insp.pending
False
```

属性状态信息，例如尚未加载或延迟加载的属性（假设 `addresses` 指的是映射类上的 `relationship()` 到相关类）：

```py
>>> insp.unloaded
{'addresses'}
```

有关属性的当前 Python 内部状态的信息，例如自上次刷新以来未被修改的属性：

```py
>>> insp.unmodified
{'nickname', 'name', 'fullname', 'id'}
```

以及自上次刷新以来对属性进行的修改的特定历史记录：

```py
>>> insp.attrs.nickname.value
'nickname'
>>> u1.nickname = "new nickname"
>>> insp.attrs.nickname.history
History(added=['new nickname'], unchanged=(), deleted=['nickname'])
```

另请参阅

`InstanceState`

`InstanceState.attrs`

`AttributeState`  #### 映射对象的检查

如前一节所示，`Mapper`对象可以从任何映射类中使用，无论方法如何，都可以使用 Runtime Inspection API 系统。使用`inspect()`函数，可以从映射类获取`Mapper`：

```py
>>> from sqlalchemy import inspect
>>> insp = inspect(User)
```

可用的详细信息包括`Mapper.columns`：

```py
>>> insp.columns
<sqlalchemy.util._collections.OrderedProperties object at 0x102f407f8>
```

这是一个可以以列表格式或通过单个名称查看的命名空间：

```py
>>> list(insp.columns)
[Column('id', Integer(), table=<user>, primary_key=True, nullable=False), Column('name', String(length=50), table=<user>), Column('fullname', String(length=50), table=<user>), Column('nickname', String(length=50), table=<user>)]
>>> insp.columns.name
Column('name', String(length=50), table=<user>)
```

其他命名空间包括`Mapper.all_orm_descriptors`，其中包括所有映射属性以及混合属性，关联代理：

```py
>>> insp.all_orm_descriptors
<sqlalchemy.util._collections.ImmutableProperties object at 0x1040e2c68>
>>> insp.all_orm_descriptors.keys()
['fullname', 'nickname', 'name', 'id']
```

以及`Mapper.column_attrs`：

```py
>>> list(insp.column_attrs)
[<ColumnProperty at 0x10403fde0; id>, <ColumnProperty at 0x10403fce8; name>, <ColumnProperty at 0x1040e9050; fullname>, <ColumnProperty at 0x1040e9148; nickname>]
>>> insp.column_attrs.name
<ColumnProperty at 0x10403fce8; name>
>>> insp.column_attrs.name.expression
Column('name', String(length=50), table=<user>)
```

另请参阅

`Mapper`

#### 映射实例的检查

`inspect()`函数还提供关于映射类的实例的信息。当应用于映射类的实例而不是类本身时，返回的对象称为`InstanceState`，它将提供指向类使用的`Mapper`的链接，以及提供有关实例内部属性状态的详细接口，包括它们当前的值以及这与它们的数据库加载值的关系。

给定从数据库加载的`User`类的实例：

```py
>>> u1 = session.scalars(select(User)).first()
```

`inspect()`函数将向我们返回一个`InstanceState`对象：

```py
>>> insp = inspect(u1)
>>> insp
<sqlalchemy.orm.state.InstanceState object at 0x7f07e5fec2e0>
```

使用此对象，我们可以查看诸如`Mapper`之类的元素：

```py
>>> insp.mapper
<Mapper at 0x7f07e614ef50; User>
```

对象附加到的`Session`（如果有）：

```py
>>> insp.session
<sqlalchemy.orm.session.Session object at 0x7f07e614f160>
```

对象的当前 persistence state 的信息：

```py
>>> insp.persistent
True
>>> insp.pending
False
```

属性状态信息，如未加载或延迟加载的属性（假设`addresses`指的是映射类上与相关类的`relationship()`）:

```py
>>> insp.unloaded
{'addresses'}
```

关于属性的当前 Python 状态的信息，例如自上次刷新以来未被修改的属性：

```py
>>> insp.unmodified
{'nickname', 'name', 'fullname', 'id'}
```

以及自上次刷新以来属性修改的具体历史：

```py
>>> insp.attrs.nickname.value
'nickname'
>>> u1.nickname = "new nickname"
>>> insp.attrs.nickname.history
History(added=['new nickname'], unchanged=(), deleted=['nickname'])
```

同样参见

`InstanceState`

`InstanceState.attrs`

`AttributeState`
