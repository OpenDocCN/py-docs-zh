# 声明式的 Mapper 配置

> 原文：[`docs.sqlalchemy.org/en/20/orm/declarative_config.html`](https://docs.sqlalchemy.org/en/20/orm/declarative_config.html)

映射类基本组件一节讨论了`Mapper`构造的一般配置元素，它是定义特定用户定义类如何映射到数据库表或其他 SQL 构造的结构。以下各节描述了关于声明式系统如何构建 `Mapper` 的具体细节。

## 使用声明式定义映射属性

使用声明式进行表配置 中给出的示例说明了针对表绑定列的映射，使用了 `mapped_column()` 构造。除了表绑定列之外，还有几种其他类型的 ORM 映射构造可以配置，最常见的是 `relationship()` 构造。其他类型的属性包括使用 `column_property()` 构造定义的 SQL 表达式，以及使用 `composite()` 构造进行多列映射。

虽然命令式映射使用 properties 字典来建立所有映射类属性，但在声明式映射中，这些属性都在类定义中内联指定，在声明性表映射的情况下，这些属性都与将用于生成 `Table` 对象的 `Column` 对象内联。

在示例映射的 `User` 和 `Address` 上工作时，我们可以演示一个声明性表映射，其中不仅包括 `mapped_column()` 对象，还包括关系和 SQL 表达式：

```py
from typing import List
from typing import Optional

from sqlalchemy import Column
from sqlalchemy import ForeignKey
from sqlalchemy import String
from sqlalchemy import Text
from sqlalchemy.orm import column_property
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "user"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    firstname: Mapped[str] = mapped_column(String(50))
    lastname: Mapped[str] = mapped_column(String(50))
    fullname: Mapped[str] = column_property(firstname + " " + lastname)

    addresses: Mapped[List["Address"]] = relationship(back_populates="user")

class Address(Base):
    __tablename__ = "address"

    id: Mapped[int] = mapped_column(primary_key=True)
    user_id: Mapped[int] = mapped_column(ForeignKey("user.id"))
    email_address: Mapped[str]
    address_statistics: Mapped[Optional[str]] = mapped_column(Text, deferred=True)

    user: Mapped["User"] = relationship(back_populates="addresses")
```

上述声明式表映射具有两个表，每个表都有一个相互引用的`relationship()`，以及一个简单的 SQL 表达式由`column_property()`映射，还有一个额外的`mapped_column()`，它指示加载应根据`mapped_column.deferred` 关键字的定义进行“延迟”。有关这些特定概念的更多文档可以在基本关系模式、使用 column_property 和限制哪些列使用列延迟加载中找到。

属性可以使用上述的声明式映射以“混合表”风格指定；直接属于表的`Column` 对象移到`Table` 定义中，但包括组成的 SQL 表达式在内的其他所有内容仍将与类定义内联。需要直接引用`Column` 的构造将使用`Table` 对象来引用它。使用混合表风格进行上述映射的示例如下：

```py
# mapping attributes using declarative with imperative table
# i.e. __table__

from sqlalchemy import Column, ForeignKey, Integer, String, Table, Text
from sqlalchemy.orm import column_property
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import deferred
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

class User(Base):
    __table__ = Table(
        "user",
        Base.metadata,
        Column("id", Integer, primary_key=True),
        Column("name", String),
        Column("firstname", String(50)),
        Column("lastname", String(50)),
    )

    fullname = column_property(__table__.c.firstname + " " + __table__.c.lastname)

    addresses = relationship("Address", back_populates="user")

class Address(Base):
    __table__ = Table(
        "address",
        Base.metadata,
        Column("id", Integer, primary_key=True),
        Column("user_id", ForeignKey("user.id")),
        Column("email_address", String),
        Column("address_statistics", Text),
    )

    address_statistics = deferred(__table__.c.address_statistics)

    user = relationship("User", back_populates="addresses")
```

以上需要注意的事项：

+   地址`Table` 包含一个名为 `address_statistics` 的列，然而我们将这一列重新映射到同一属性名称下，以便由`deferred()` 构造进行控制。

+   在声明式表和混合表映射中，当我们定义一个`ForeignKey` 构造时，我们总是使用**表名称**而不是映射的类名称来命名目标表。

+   当我们定义`relationship()`构造时，由于这些构造在两个映射类之间创建了一个链接，其中一个必然在另一个之前被定义，我们可以使用其字符串名称引用远程类。此功能还扩展到在`relationship()`上指定的其他参数，如“primary join”和“order by”参数。有关详细信息，请参阅延迟评估关系参数章节。## 使用声明配置选项的映射器

对于所有的映射形式，类的映射是通过成为`Mapper`对象的一部分的参数配置的。最终接收这些参数的函数是`Mapper`函数，并且这些参数是从`registry`对象上定义的其中一个前端映射函数传递给它的。

对于映射的声明形式，映射器参数是使用`__mapper_args__`声明性类变量指定的，它是一个字典，作为关键字参数传递给`Mapper`函数。一些示例：

**映射特定的主键列**

下面的示例说明了`Mapper.primary_key`参数的声明级设置，该参数将特定列作为 ORM 应考虑为类的主键的一部分，而不受架构级主键约束的影响：

```py
class GroupUsers(Base):
    __tablename__ = "group_users"

    user_id = mapped_column(String(40))
    group_id = mapped_column(String(40))

    __mapper_args__ = {"primary_key": [user_id, group_id]}
```

另请参阅

映射到显式一组主键列 - 进一步了解显式列作为主键列的 ORM 映射的背景

**版本 ID 列**

下面的示例说明了`Mapper.version_id_col`和`Mapper.version_id_generator`参数的声明级设置，它们配置了一个 ORM 维护的版本计数器，在工作单元刷新过程中更新和检查：

```py
from datetime import datetime

class Widget(Base):
    __tablename__ = "widgets"

    id = mapped_column(Integer, primary_key=True)
    timestamp = mapped_column(DateTime, nullable=False)

    __mapper_args__ = {
        "version_id_col": timestamp,
        "version_id_generator": lambda v: datetime.now(),
    }
```

另请参阅

配置版本计数器 - 关于 ORM 版本计数器功能的背景

**单表继承**

下面的示例说明了用于配置单表继承映射时使用的 `Mapper.polymorphic_on` 和 `Mapper.polymorphic_identity` 参数的声明级别设置：

```py
class Person(Base):
    __tablename__ = "person"

    person_id = mapped_column(Integer, primary_key=True)
    type = mapped_column(String, nullable=False)

    __mapper_args__ = dict(
        polymorphic_on=type,
        polymorphic_identity="person",
    )

class Employee(Person):
    __mapper_args__ = dict(
        polymorphic_identity="employee",
    )
```

另请参阅

单表继承 - ORM 单表继承映射功能的背景。

### 动态构造映射器参数

`__mapper_args__` 字典可以通过使用 `declared_attr()` 构造而不是固定字典而生成。通过此方式生成 `__mapper_args__` 对于从表配置或映射类的其他方面程序化派生映射器参数非常有用。动态 `__mapper_args__` 属性通常在使用声明性混合或抽象基类时非常有用。

例如，为了从映射中省略具有特殊 `Column.info` 值的任何列，一个混合类可以使用一个 `__mapper_args__` 方法，该方法从 `cls.__table__` 属性中扫描这些列并将其传递给 `Mapper.exclude_properties` 集合：

```py
from sqlalchemy import Column
from sqlalchemy import Integer
from sqlalchemy import select
from sqlalchemy import String
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import declared_attr

class ExcludeColsWFlag:
    @declared_attr
    def __mapper_args__(cls):
        return {
            "exclude_properties": [
                column.key
                for column in cls.__table__.c
                if column.info.get("exclude", False)
            ]
        }

class Base(DeclarativeBase):
    pass

class SomeClass(ExcludeColsWFlag, Base):
    __tablename__ = "some_table"

    id = mapped_column(Integer, primary_key=True)
    data = mapped_column(String)
    not_needed = mapped_column(String, info={"exclude": True})
```

上面，`ExcludeColsWFlag` 混合类提供了一个每个类的 `__mapper_args__` 钩子，该钩子将扫描包含传递给 `Column.info` 参数的键/值 `'exclude': True` 的 `Column` 对象，然后将其字符串“键”名称添加到 `Mapper.exclude_properties` 集合中，这将防止生成的 `Mapper` 考虑这些列进行任何 SQL 操作。  

另请参阅

使用混合组合映射层次结构

## 其他声明性映射指令

### `__declare_last__()`

`__declare_last__()` 钩子允许定义一个类级别函数，该函数将自动由 `MapperEvents.after_configured()` 事件调用，在映射假定完成并且“配置”步骤已经完成后发生：

```py
class MyClass(Base):
    @classmethod
    def __declare_last__(cls):
  """ """
        # do something with mappings
```

### `__declare_first__()`

类似于 `__declare_last__()`，但是在通过 `MapperEvents.before_configured()` 事件开始映射器配置时调用：

```py
class MyClass(Base):
    @classmethod
    def __declare_first__(cls):
  """ """
        # do something before mappings are configured
```

### `metadata`

通常用于分配新`Table`的`MetaData`集合是与正在使用的`registry`对象关联的`registry.metadata`属性。当使用像`DeclarativeBase`超类生成的声明基类时，以及像`declarative_base()`和`registry.generate_base()`这样的旧函数时，这个`MetaData`通常也作为一个名为`.metadata`的属性直接存在于基类上，因此也通过继承存在于映射类上。当存在时，声明会使用此属性来确定目标`MetaData`集合，如果不存在，则使用与直接与`registry`关联的`MetaData`。

这个属性还可以分配给单个基类和/或`registry`以影响每个映射层次结构的`MetaData`集合的使用。这对于使用声明基类或直接使用`registry.mapped()`装饰器的情况都会生效，从而允许如下所示的每个抽象基类的元数据模式示例，在下一节 __abstract__。可以使用`registry.mapped()`来说明类似的模式：

```py
reg = registry()

class BaseOne:
    metadata = MetaData()

class BaseTwo:
    metadata = MetaData()

@reg.mapped
class ClassOne:
    __tablename__ = "t1"  # will use reg.metadata

    id = mapped_column(Integer, primary_key=True)

@reg.mapped
class ClassTwo(BaseOne):
    __tablename__ = "t1"  # will use BaseOne.metadata

    id = mapped_column(Integer, primary_key=True)

@reg.mapped
class ClassThree(BaseTwo):
    __tablename__ = "t1"  # will use BaseTwo.metadata

    id = mapped_column(Integer, primary_key=True)
```

另请参阅

__abstract__  ### `__abstract__`

`__abstract__`会导致声明跳过对该类的表或映射器的生成。类可以像混合类一样添加到层次结构中（参见 Mixin and Custom Base Classes），允许子类仅从特殊类扩展：

```py
class SomeAbstractBase(Base):
    __abstract__ = True

    def some_helpful_method(self):
  """ """

    @declared_attr
    def __mapper_args__(cls):
        return {"helpful mapper arguments": True}

class MyMappedClass(SomeAbstractBase):
    pass
```

`__abstract__`的一个可能用途是为不同的基类使用不同的`MetaData`：

```py
class Base(DeclarativeBase):
    pass

class DefaultBase(Base):
    __abstract__ = True
    metadata = MetaData()

class OtherBase(Base):
    __abstract__ = True
    metadata = MetaData()
```

类从 `DefaultBase` 继承的将使用一个 `MetaData` 作为表的注册表，而从 `OtherBase` 继承的将使用另一个。然后，这些表本身可以被创建在不同的数据库中：

```py
DefaultBase.metadata.create_all(some_engine)
OtherBase.metadata.create_all(some_other_engine)
```

另请参阅

使用 polymorphic_abstract 构建更深层次的层次结构 - 这是适用于继承层次结构的另一种“抽象”映射类的替代形式。### `__table_cls__`

允许自定义用于生成 `Table` 的可调用/类。这是一个非常开放的钩子，可以允许对在此生成的 `Table` 进行特殊的自定义：

```py
class MyMixin:
    @classmethod
    def __table_cls__(cls, name, metadata_obj, *arg, **kw):
        return Table(f"my_{name}", metadata_obj, *arg, **kw)
```

上述的混合类将导致所有生成的 `Table` 对象都包含前缀 `"my_"`，后跟通常使用 `__tablename__` 属性指定的名称。

`__table_cls__` 还支持返回 `None` 的情况，这会导致将该类视为单表继承与其子类。这在某些定制方案中可能很有用，以确定是否应该基于表本身的参数来执行单表继承，例如，如果没有主键存在，则定义为单继承：

```py
class AutoTable:
    @declared_attr
    def __tablename__(cls):
        return cls.__name__

    @classmethod
    def __table_cls__(cls, *arg, **kw):
        for obj in arg[1:]:
            if (isinstance(obj, Column) and obj.primary_key) or isinstance(
                obj, PrimaryKeyConstraint
            ):
                return Table(*arg, **kw)

        return None

class Person(AutoTable, Base):
    id = mapped_column(Integer, primary_key=True)

class Employee(Person):
    employee_name = mapped_column(String)
```

上述的 `Employee` 类将被映射为单表继承，对应于 `Person`；`employee_name` 列将被添加为 `Person` 表的成员。## 使用声明式定义映射属性

在使用声明式配置表的示例中，说明了针对表绑定列的映射，使用了 `mapped_column()` 构造。除了针对表绑定列之外，还可以配置几种其他类型的 ORM 映射构造，最常见的是 `relationship()` 构造。其他类型的属性包括使用 `column_property()` 构造定义的 SQL 表达式和使用 `composite()` 构造的多列映射。

在命令式映射中，利用属性字典来建立所有映射类属性，而在声明式映射中，这些属性都与类定义一起内联指定，这在声明式表映射的情况下与将用于生成 Table 对象的 Column 对象一起内联。

使用`User`和`Address`的示例映射，我们可以说明一个包括不仅是 mapped_column()对象还包括关系和 SQL 表达式的声明式表映射：

```py
from typing import List
from typing import Optional

from sqlalchemy import Column
from sqlalchemy import ForeignKey
from sqlalchemy import String
from sqlalchemy import Text
from sqlalchemy.orm import column_property
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "user"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    firstname: Mapped[str] = mapped_column(String(50))
    lastname: Mapped[str] = mapped_column(String(50))
    fullname: Mapped[str] = column_property(firstname + " " + lastname)

    addresses: Mapped[List["Address"]] = relationship(back_populates="user")

class Address(Base):
    __tablename__ = "address"

    id: Mapped[int] = mapped_column(primary_key=True)
    user_id: Mapped[int] = mapped_column(ForeignKey("user.id"))
    email_address: Mapped[str]
    address_statistics: Mapped[Optional[str]] = mapped_column(Text, deferred=True)

    user: Mapped["User"] = relationship(back_populates="addresses")
```

上述声明式表映射具有两个表，每个表都具有相互引用的 relationship()以及由 column_property()映射的简单 SQL 表达式，以及一个额外的 mapped_column()，该列指示加载应根据 mapped_column.deferred 关键字定义为“deferred”。有关这些特定概念的更多文档可在基本关系模式、使用 column_property 和使用列推迟限制加载的列中找到。

使用声明式映射，可以像上面那样使用“混合表”样式来指定属性；直接属于表的 Column 对象移入 Table 定义，但包括组合 SQL 表达式在内的其他所有内容仍将内联到类定义中。需要引用 Column 的构造将以 Table 对象的术语引用它。为了使用混合表样式说明上述映射：

```py
# mapping attributes using declarative with imperative table
# i.e. __table__

from sqlalchemy import Column, ForeignKey, Integer, String, Table, Text
from sqlalchemy.orm import column_property
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import deferred
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

class User(Base):
    __table__ = Table(
        "user",
        Base.metadata,
        Column("id", Integer, primary_key=True),
        Column("name", String),
        Column("firstname", String(50)),
        Column("lastname", String(50)),
    )

    fullname = column_property(__table__.c.firstname + " " + __table__.c.lastname)

    addresses = relationship("Address", back_populates="user")

class Address(Base):
    __table__ = Table(
        "address",
        Base.metadata,
        Column("id", Integer, primary_key=True),
        Column("user_id", ForeignKey("user.id")),
        Column("email_address", String),
        Column("address_statistics", Text),
    )

    address_statistics = deferred(__table__.c.address_statistics)

    user = relationship("User", back_populates="addresses")
```

以上需要注意的事项：

+   地址 Table 包含一个名为`address_statistics`的列，然而我们将此列重新映射到同一属性名称下，以便受`deferred()`构造的控制。

+   在声明性表和混合表映射中，当我们定义一个 `ForeignKey` 构造时，我们总是使用**表名**来命名目标表，而不是映射类名。

+   当我们定义 `relationship()` 构造时，由于这些构造在两个映射类之间创建了链接，其中一个必然在另一个之前被定义，我们可以使用其字符串名称引用远程类。这个功能也扩展到 `relationship()` 上指定的其他参数，如“主连接”和“排序”参数。有关详细信息，请参阅 Relationship 参数的延迟评估 部分。

## 带有声明性的 Mapper 配置选项

对于所有映射形式，类的映射通过成为 `Mapper` 对象的一部分的参数进行配置。最终接收这些参数的函数是 `Mapper` 函数，并且它们从定义在 `registry` 对象上的一个前置映射函数中传递给它。

对于映射的声明形式，映射器参数是使用 `__mapper_args__` 声明性类变量指定的，该变量是一个字典，作为关键字参数传递给 `Mapper` 函数。一些示例：

**映射特定的主键列**

下面的示例说明了 `Mapper.primary_key` 参数的声明级别设置，它将特定列作为 ORM 应该考虑为类的主键的一部分，与架构级主键约束独立：

```py
class GroupUsers(Base):
    __tablename__ = "group_users"

    user_id = mapped_column(String(40))
    group_id = mapped_column(String(40))

    __mapper_args__ = {"primary_key": [user_id, group_id]}
```

另请参阅

映射到一组显式主键列 - ORM 将显式列映射为主键列的更多背景

**版本 ID 列**

下面的示例说明了 `Mapper.version_id_col` 和 `Mapper.version_id_generator` 参数的声明级别设置，它们配置了一个由 ORM 维护的版本计数器，在 工作单元 刷新过程中进行更新和检查：

```py
from datetime import datetime

class Widget(Base):
    __tablename__ = "widgets"

    id = mapped_column(Integer, primary_key=True)
    timestamp = mapped_column(DateTime, nullable=False)

    __mapper_args__ = {
        "version_id_col": timestamp,
        "version_id_generator": lambda v: datetime.now(),
    }
```

另请参阅

配置版本计数器 - ORM 版本计数器功能的背景

**单表继承**

下面的示例说明了用于配置单表继承映射时使用的 `Mapper.polymorphic_on` 和 `Mapper.polymorphic_identity` 参数的声明级别设置：

```py
class Person(Base):
    __tablename__ = "person"

    person_id = mapped_column(Integer, primary_key=True)
    type = mapped_column(String, nullable=False)

    __mapper_args__ = dict(
        polymorphic_on=type,
        polymorphic_identity="person",
    )

class Employee(Person):
    __mapper_args__ = dict(
        polymorphic_identity="employee",
    )
```

另见

单表继承 - ORM 单表继承映射功能的背景介绍。

### 动态构建映射器参数

`__mapper_args__` 字典可以通过使用 `declared_attr()` 构造而不是固定字典从类绑定描述符方法生成。这对于从表配置或映射类的其他方面编程派生映射器参数非常有用。动态的 `__mapper_args__` 属性通常在使用声明性 Mixin 或抽象基类时非常有用。

例如，要从映射中省略具有特殊 `Column.info` 值的任何列，一个 Mixin 可以使用一个 `__mapper_args__` 方法，该方法从 `cls.__table__` 属性扫描这些列并将它们传递给 `Mapper.exclude_properties` 集合：

```py
from sqlalchemy import Column
from sqlalchemy import Integer
from sqlalchemy import select
from sqlalchemy import String
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import declared_attr

class ExcludeColsWFlag:
    @declared_attr
    def __mapper_args__(cls):
        return {
            "exclude_properties": [
                column.key
                for column in cls.__table__.c
                if column.info.get("exclude", False)
            ]
        }

class Base(DeclarativeBase):
    pass

class SomeClass(ExcludeColsWFlag, Base):
    __tablename__ = "some_table"

    id = mapped_column(Integer, primary_key=True)
    data = mapped_column(String)
    not_needed = mapped_column(String, info={"exclude": True})
```

在上面的示例中，`ExcludeColsWFlag` Mixin 提供了一个每个类的 `__mapper_args__` 钩子，它将扫描包含传递给 `Column.info` 参数的键/值 `'exclude': True` 的 `Column` 对象，然后将它们的字符串“键”名称添加到 `Mapper.exclude_properties` 集合中，这将阻止生成的 `Mapper` 对这些列进行任何 SQL 操作的考虑。

另见

使用 Mixin 组合映射层次结构

### 动态构建映射器参数

`__mapper_args__` 字典可以通过使用 `declared_attr()` 构造而不是固定字典从类绑定描述符方法生成。这对于从表配置或映射类的其他方面编程派生映射器参数非常有用。动态的 `__mapper_args__` 属性通常在使用声明性 Mixin 或抽象基类时非常有用。

例如，要从映射中省略具有特殊`Column.info`值的任何列，mixin 可以使用一个`__mapper_args__`方法从`cls.__table__`属性中扫描这些列，并将它们传递给`Mapper.exclude_properties`集合：

```py
from sqlalchemy import Column
from sqlalchemy import Integer
from sqlalchemy import select
from sqlalchemy import String
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import declared_attr

class ExcludeColsWFlag:
    @declared_attr
    def __mapper_args__(cls):
        return {
            "exclude_properties": [
                column.key
                for column in cls.__table__.c
                if column.info.get("exclude", False)
            ]
        }

class Base(DeclarativeBase):
    pass

class SomeClass(ExcludeColsWFlag, Base):
    __tablename__ = "some_table"

    id = mapped_column(Integer, primary_key=True)
    data = mapped_column(String)
    not_needed = mapped_column(String, info={"exclude": True})
```

在上面的例子中，`ExcludeColsWFlag` mixin 提供了一个每个类的`__mapper_args__`钩子，该钩子将扫描包含传递给`Column.info`参数的键/值`'exclude': True`的`Column`对象，然后将其字符串“key”名称添加到`Mapper.exclude_properties`集合中，这将阻止生成的`Mapper`考虑这些列进行任何 SQL 操作。

另请参阅

使用 Mixins 构建映射的分层结构

## 其他声明性映射指令

### `__declare_last__()`

`__declare_last__()`钩子允许定义一个类级别函数，该函数会在`MapperEvents.after_configured()`事件自动调用，此事件发生在假定映射已完成且“configure”步骤已完成之后：

```py
class MyClass(Base):
    @classmethod
    def __declare_last__(cls):
  """ """
        # do something with mappings
```

### `__declare_first__()`

像`__declare_last__()`一样，但是通过`MapperEvents.before_configured()`事件在映射器配置的开始时调用：

```py
class MyClass(Base):
    @classmethod
    def __declare_first__(cls):
  """ """
        # do something before mappings are configured
```

### `metadata`

`MetaData` 集合通常用于为新的 `Table` 分配，它是与正在使用的 `registry` 对象相关联的 `registry.metadata` 属性。当使用诸如 `DeclarativeBase` 超类生成的声明性基类，以及诸如 `declarative_base()` 和 `registry.generate_base()` 等遗留函数时，通常也会存在这个 `MetaData`，它作为名为 `.metadata` 的属性直接存在于基类上，因此也通过继承存在于映射类上。当存在时，声明性使用此属性来确定目标 `MetaData` 集合，如果不存在，则使用与 `registry` 直接关联的 `MetaData`。

这个属性也可以被分配到，以便对单个基类和/或 `registry` 的每个映射层次结构基础使用影响 `MetaData` 集合。这将影响到是否使用声明性基类或直接使用 `registry.mapped()` 装饰器，从而允许模式，例如下一节中的基于抽象基类的元数据示例，__abstract__。类似的模式可以使用 `registry.mapped()` 来说明如下：

```py
reg = registry()

class BaseOne:
    metadata = MetaData()

class BaseTwo:
    metadata = MetaData()

@reg.mapped
class ClassOne:
    __tablename__ = "t1"  # will use reg.metadata

    id = mapped_column(Integer, primary_key=True)

@reg.mapped
class ClassTwo(BaseOne):
    __tablename__ = "t1"  # will use BaseOne.metadata

    id = mapped_column(Integer, primary_key=True)

@reg.mapped
class ClassThree(BaseTwo):
    __tablename__ = "t1"  # will use BaseTwo.metadata

    id = mapped_column(Integer, primary_key=True)
```

另请参阅

__abstract__  ### `__abstract__`

`__abstract__` 会导致声明性完全跳过为类生成表或映射器。可以像 mixin 一样在层次结构中添加类（参见 Mixin and Custom Base Classes），允许子类仅从特殊类扩展：

```py
class SomeAbstractBase(Base):
    __abstract__ = True

    def some_helpful_method(self):
  """ """

    @declared_attr
    def __mapper_args__(cls):
        return {"helpful mapper arguments": True}

class MyMappedClass(SomeAbstractBase):
    pass
```

`__abstract__` 的一种可能用法是为不同的基类使用不同的 `MetaData`：

```py
class Base(DeclarativeBase):
    pass

class DefaultBase(Base):
    __abstract__ = True
    metadata = MetaData()

class OtherBase(Base):
    __abstract__ = True
    metadata = MetaData()
```

在上述示例中，从 `DefaultBase` 继承的类将使用一个 `MetaData` 作为表的注册表，而从 `OtherBase` 继承的类将使用不同的注册表。然后，表本身可能会被创建在不同的数据库中：

```py
DefaultBase.metadata.create_all(some_engine)
OtherBase.metadata.create_all(some_other_engine)
```

另请参见

使用 polymorphic_abstract 构建更深层次的继承结构 - 适用于继承层次结构的另一种“抽象”映射类形式。### `__table_cls__`

允许定制用于生成 `Table` 的可调用 / 类。这是一个非常开放的钩子，可以允许对在此生成的 `Table` 进行特殊定制：

```py
class MyMixin:
    @classmethod
    def __table_cls__(cls, name, metadata_obj, *arg, **kw):
        return Table(f"my_{name}", metadata_obj, *arg, **kw)
```

上述的混合类将导致所有生成的 `Table` 对象都包含前缀 `"my_"`，后跟通常使用 `__tablename__` 属性指定的名称。

`__table_cls__` 还支持返回 `None` 的情况，这会导致将类视为单表继承 vs. 其子类。在某些定制方案中，这可能是有用的，以确定应基于表本身的参数进行单表继承，例如，如果不存在主键，则定义为单继承：

```py
class AutoTable:
    @declared_attr
    def __tablename__(cls):
        return cls.__name__

    @classmethod
    def __table_cls__(cls, *arg, **kw):
        for obj in arg[1:]:
            if (isinstance(obj, Column) and obj.primary_key) or isinstance(
                obj, PrimaryKeyConstraint
            ):
                return Table(*arg, **kw)

        return None

class Person(AutoTable, Base):
    id = mapped_column(Integer, primary_key=True)

class Employee(Person):
    employee_name = mapped_column(String)
```

上述的 `Employee` 类将被映射为针对 `Person` 的单表继承；`employee_name` 列将作为 `Person` 表的成员添加。

### `__declare_last__()`

`__declare_last__()` 钩子允许定义一个类级别的函数，该函数会在 `MapperEvents.after_configured()` 事件之后自动调用，该事件发生在映射被认为已完成并且“配置”步骤已经结束之后：

```py
class MyClass(Base):
    @classmethod
    def __declare_last__(cls):
  """ """
        # do something with mappings
```

### `__declare_first__()`

类似于 `__declare_last__()`，但是在映射器配置的开始阶段通过 `MapperEvents.before_configured()` 事件调用：

```py
class MyClass(Base):
    @classmethod
    def __declare_first__(cls):
  """ """
        # do something before mappings are configured
```

### `metadata`

通常用于分配新`Table`的`MetaData`集合是与正在使用的`registry`对象关联的`registry.metadata`属性。当使用像`DeclarativeBase`超类生成的声明性基类时，以及诸如`declarative_base()`和`registry.generate_base()`这样的旧函数时，这个`MetaData`通常也作为一个名为`.metadata`的属性存在于直接在基类上的，并且通过继承也存在于映射类上。当存在时，声明性会使用这个属性来确定目标`MetaData`集合，如果不存在，则使用直接与`registry`关联的`MetaData`。

此属性还可以被分配到`MetaData`集合上，以便对单个基类和/或`registry`在每个映射的层次结构上进行影响。这将在使用声明性基类或直接使用`registry.mapped()`装饰器时生效，从而允许像下一节中的抽象基类示例一样的模式，__abstract__。类似的模式可以使用`registry.mapped()`来说明如下：

```py
reg = registry()

class BaseOne:
    metadata = MetaData()

class BaseTwo:
    metadata = MetaData()

@reg.mapped
class ClassOne:
    __tablename__ = "t1"  # will use reg.metadata

    id = mapped_column(Integer, primary_key=True)

@reg.mapped
class ClassTwo(BaseOne):
    __tablename__ = "t1"  # will use BaseOne.metadata

    id = mapped_column(Integer, primary_key=True)

@reg.mapped
class ClassThree(BaseTwo):
    __tablename__ = "t1"  # will use BaseTwo.metadata

    id = mapped_column(Integer, primary_key=True)
```

另请参阅

__abstract__

### `__abstract__`

`__abstract__`导致声明性完全跳过为类生成表或映射。可以以与 mixin 相同的方式将类添加到层次结构中（请参阅 Mixin and Custom Base Classes），从而允许子类仅从特殊类扩展：

```py
class SomeAbstractBase(Base):
    __abstract__ = True

    def some_helpful_method(self):
  """ """

    @declared_attr
    def __mapper_args__(cls):
        return {"helpful mapper arguments": True}

class MyMappedClass(SomeAbstractBase):
    pass
```

`__abstract__`的一个可能用法是为不同的基类使用不同的`MetaData`：

```py
class Base(DeclarativeBase):
    pass

class DefaultBase(Base):
    __abstract__ = True
    metadata = MetaData()

class OtherBase(Base):
    __abstract__ = True
    metadata = MetaData()
```

在上面，继承自 `DefaultBase` 的类将使用一个 `MetaData` 作为表的注册表，而继承自 `OtherBase` 的类将使用另一个注册表。然后，这些表可以在不同的数据库中创建：

```py
DefaultBase.metadata.create_all(some_engine)
OtherBase.metadata.create_all(some_other_engine)
```

另请参阅

使用 polymorphic_abstract 构建更深层次的层次结构 - 一种适用于继承层次结构的“抽象”映射类的替代形式。

### `__table_cls__`

允许定制用于生成 `Table` 的可调用函数/类。这是一个非常开放的钩子，可以允许对在此生成的 `Table` 进行特殊的定制:

```py
class MyMixin:
    @classmethod
    def __table_cls__(cls, name, metadata_obj, *arg, **kw):
        return Table(f"my_{name}", metadata_obj, *arg, **kw)
```

上述混合类将导致所有生成的 `Table` 对象都包含前缀 `"my_"`，后跟通常使用 `__tablename__` 属性指定的名称。

`__table_cls__` 也支持返回 `None` 的情况，这将导致将类视为单表继承与其子类。在一些定制方案中，这可能是有用的，以确定基于表本身的参数是否应该进行单表继承，例如，如果没有主键存在，则定义为单继承：

```py
class AutoTable:
    @declared_attr
    def __tablename__(cls):
        return cls.__name__

    @classmethod
    def __table_cls__(cls, *arg, **kw):
        for obj in arg[1:]:
            if (isinstance(obj, Column) and obj.primary_key) or isinstance(
                obj, PrimaryKeyConstraint
            ):
                return Table(*arg, **kw)

        return None

class Person(AutoTable, Base):
    id = mapped_column(Integer, primary_key=True)

class Employee(Person):
    employee_name = mapped_column(String)
```

上述 `Employee` 类将被映射为单表继承，对 `Person` 进行映射；`employee_name` 列将作为 `Person` 表的成员添加。
