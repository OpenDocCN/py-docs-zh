# 使用混合类组合映射层次结构

> 原文：[`docs.sqlalchemy.org/en/20/orm/declarative_mixins.html`](https://docs.sqlalchemy.org/en/20/orm/declarative_mixins.html)

在使用 Declarative 风格映射类时，常见的需求是共享常见功能，例如特定列、表或映射器选项、命名方案或其他映射属性，跨多个类。在使用声明性映射时，可以通过使用 mixin 类，以及通过扩展声明性基类本身来支持此习惯用法。

提示

除了 mixin 类之外，还可以使用[**PEP 593**](https://peps.python.org/pep-0593/) `Annotated` 类型共享许多类的常见列选项；请参阅将多种类型配置映射到 Python 类型和将整个列声明映射到 Python 类型以获取有关这些 SQLAlchemy 2.0 功能的背景信息。

以下是一些常见的混合用法示例：

```py
from sqlalchemy import ForeignKey
from sqlalchemy.orm import declared_attr
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

class CommonMixin:
  """define a series of common elements that may be applied to mapped
 classes using this class as a mixin class."""

    @declared_attr.directive
    def __tablename__(cls) -> str:
        return cls.__name__.lower()

    __table_args__ = {"mysql_engine": "InnoDB"}
    __mapper_args__ = {"eager_defaults": True}

    id: Mapped[int] = mapped_column(primary_key=True)

class HasLogRecord:
  """mark classes that have a many-to-one relationship to the
 ``LogRecord`` class."""

    log_record_id: Mapped[int] = mapped_column(ForeignKey("logrecord.id"))

    @declared_attr
    def log_record(self) -> Mapped["LogRecord"]:
        return relationship("LogRecord")

class LogRecord(CommonMixin, Base):
    log_info: Mapped[str]

class MyModel(CommonMixin, HasLogRecord, Base):
    name: Mapped[str]
```

上述示例说明了一个类`MyModel`，它包括两个混合类`CommonMixin`和`HasLogRecord`，以及一个补充类`LogRecord`，该类也包括`CommonMixin`，演示了在混合类和基类上支持的各种构造，包括：

+   使用`mapped_column()`、`Mapped`或`Column`声明的列将从混合类或基类复制到要映射的目标类；上面通过列属性`CommonMixin.id`和`HasLogRecord.log_record_id`说明了这一点。

+   可以将声明性指令（如`__table_args__`和`__mapper_args__`）分配给混合类或基类，在继承混合类或基类的任何类中，这些指令将自动生效。上述示例使用`__table_args__`和`__mapper_args__`属性说明了这一点。

+   所有的 Declarative 指令，包括 `__tablename__`、`__table__`、`__table_args__` 和 `__mapper_args__`，都可以使用用户定义的类方法来实现，这些方法使用了 `declared_attr` 装饰器进行修饰（具体地说是 `declared_attr.directive` 子成员，稍后会详细介绍）。上面的示例使用了一个 `def __tablename__(cls)` 类方法动态生成一个 `Table` 名称；当应用到 `MyModel` 类时，表名将生成为 `"mymodel"`，而当应用到 `LogRecord` 类时，表名将生成为 `"logrecord"`。

+   其他 ORM 属性，如 `relationship()`，也可以通过在目标类上生成的用户定义的类方法来生成，并且这些类方法也使用了 `declared_attr` 装饰器进行修饰。上面的例子演示了通过生成一个多对一的 `relationship()` 到一个名为 `LogRecord` 的映射对象来实现此功能。

上述功能都可以使用 `select()` 示例进行演示：

```py
>>> from sqlalchemy import select
>>> print(select(MyModel).join(MyModel.log_record))
SELECT  mymodel.name,  mymodel.id,  mymodel.log_record_id
FROM  mymodel  JOIN  logrecord  ON  logrecord.id  =  mymodel.log_record_id 
```

提示

`declared_attr` 的示例将尝试说明每个方法示例的正确的 [**PEP 484**](https://peps.python.org/pep-0484/) 注解。使用 `declared_attr` 函数的注解是**完全可选**的，并且不会被 Declarative 消耗；然而，为了通过 Mypy 的 `--strict` 类型检查，这些注解是必需的。

另外，上面所示的 `declared_attr.directive` 子成员也是可选的，它只对 [**PEP 484**](https://peps.python.org/pep-0484/) 类型工具有意义，因为它调整了创建用于重写 Declarative 指令的方法时的期望返回类型，例如 `__tablename__`、`__mapper_args__` 和 `__table_args__`。

新版本 2.0 中新增：作为 SQLAlchemy ORM 的 [**PEP 484**](https://peps.python.org/pep-0484/) 类型支持的一部分，添加了 `declared_attr.directive` 来将 `declared_attr` 区分为 `Mapped` 属性和声明性配置属性之间的区别。

混合类和基类的顺序没有固定的约定。普通的 Python 方法解析规则适用，上述示例也同样适用：

```py
class MyModel(Base, HasLogRecord, CommonMixin):
    name: Mapped[str] = mapped_column()
```

这是因为这里的 `Base` 没有定义 `CommonMixin` 或 `HasLogRecord` 定义的任何变量，即 `__tablename__`、`__table_args__`、`id` 等。如果 `Base` 定义了同名属性，则位于继承列表中的第一个类将决定在新定义的类上使用哪个属性。

提示

虽然上述示例使用了基于 `Mapped` 注解类的注释声明表形式，但混合类与非注释和遗留声明形式也完全兼容，比如直接使用 `Column` 而不是 `mapped_column()` 时。

从版本 2.0 开始更改：对于从 SQLAlchemy 1.4 系列迁移到的用户可能一直在使用 mypy 插件，不再需要使用 `declarative_mixin()` 类装饰器来标记声明性混合类，假设不再使用 mypy 插件。

## 扩充基类

除了使用纯混合类之外，本节中的大多数技术也可以直接应用于基类，用于适用于从特定基类派生的所有类的模式。下面的示例演示了上一节中的一些示例在 `Base` 类方面的情况：

```py
from sqlalchemy import ForeignKey
from sqlalchemy.orm import declared_attr
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
  """define a series of common elements that may be applied to mapped
 classes using this class as a base class."""

    @declared_attr.directive
    def __tablename__(cls) -> str:
        return cls.__name__.lower()

    __table_args__ = {"mysql_engine": "InnoDB"}
    __mapper_args__ = {"eager_defaults": True}

    id: Mapped[int] = mapped_column(primary_key=True)

class HasLogRecord:
  """mark classes that have a many-to-one relationship to the
 ``LogRecord`` class."""

    log_record_id: Mapped[int] = mapped_column(ForeignKey("logrecord.id"))

    @declared_attr
    def log_record(self) -> Mapped["LogRecord"]:
        return relationship("LogRecord")

class LogRecord(Base):
    log_info: Mapped[str]

class MyModel(HasLogRecord, Base):
    name: Mapped[str]
```

上述，`MyModel` 以及 `LogRecord`，在从 `Base` 派生时，它们的表名都将根据其类名派生，一个名为 `id` 的主键列，以及由 `Base.__table_args__` 和 `Base.__mapper_args__` 定义的上述表和映射器参数。

当使用遗留 `declarative_base()` 或 `registry.generate_base()` 时，可以像下面的非注释示例中所示使用 `declarative_base.cls` 参数来生成等效效果：

```py
# legacy declarative_base() use

from sqlalchemy import Integer, String
from sqlalchemy import ForeignKey
from sqlalchemy.orm import declared_attr
from sqlalchemy.orm import declarative_base
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship

class Base:
  """define a series of common elements that may be applied to mapped
 classes using this class as a base class."""

    @declared_attr.directive
    def __tablename__(cls):
        return cls.__name__.lower()

    __table_args__ = {"mysql_engine": "InnoDB"}
    __mapper_args__ = {"eager_defaults": True}

    id = mapped_column(Integer, primary_key=True)

Base = declarative_base(cls=Base)

class HasLogRecord:
  """mark classes that have a many-to-one relationship to the
 ``LogRecord`` class."""

    log_record_id = mapped_column(ForeignKey("logrecord.id"))

    @declared_attr
    def log_record(self):
        return relationship("LogRecord")

class LogRecord(Base):
    log_info = mapped_column(String)

class MyModel(HasLogRecord, Base):
    name = mapped_column(String)
```

## 混合列

如果使用声明式表 风格的配置（而不是命令式表 配置），则可以在混合中指定列，以便混合中声明的列随后将被复制为声明式进程生成的`Table` 的一部分。在声明式混合中可以内联声明三种构造：`mapped_column()`、`Mapped` 和`Column`。

```py
class TimestampMixin:
    created_at: Mapped[datetime] = mapped_column(default=func.now())
    updated_at: Mapped[datetime]

class MyModel(TimestampMixin, Base):
    __tablename__ = "test"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
```

在上述位置，所有包含`TimestampMixin`在其类基础中的声明性类将自动包含一个应用于所有行插入的时间戳的列`created_at`，以及一个`updated_at`列，该列不包含示例目的的默认值（如果有的话，我们将使用`Column.onupdate` 参数，该参数由`mapped_column()` 接受）。这些列结构始终是**从原始混合或基类复制**的，因此相同的混合/基类可以应用于任意数量的目标类，每个目标类都将具有自己的列结构。

所有声明式列形式都受混合支持，包括：

+   **带有注解的属性** - 无论是否存在`mapped_column()`：

    ```py
    class TimestampMixin:
        created_at: Mapped[datetime] = mapped_column(default=func.now())
        updated_at: Mapped[datetime]
    ```

+   **mapped_column** - 无论是否存在 `Mapped`：

    ```py
    class TimestampMixin:
        created_at = mapped_column(default=func.now())
        updated_at: Mapped[datetime] = mapped_column()
    ```

+   **列** - 传统的声明式形式：

    ```py
    class TimestampMixin:
        created_at = Column(DateTime, default=func.now())
        updated_at = Column(DateTime)
    ```

在上述每种形式中，声明式通过创建构造的**副本**来处理混合类上的基于列的属性，然后将其应用于目标类。

自版本 2.0 起更改：声明式 API 现在可以容纳`Column` 对象以及使用混合时的任何形式的`mapped_column()` 构造，而无需使用`declared_attr()`。已经删除了以前的限制，该限制阻止具有`ForeignKey` 元素的列直接在混合中使用。

## 混入关系

通过`relationship()`创建的关系，仅使用`declared_attr`方法提供声明性混合类，从而消除了复制关系及其可能绑定到列的内容时可能出现的任何歧义。下面是一个示例，其中结合了外键列和关系，以便两个类`Foo`和`Bar`都可以通过多对一引用到一个公共目标类：

```py
from sqlalchemy import ForeignKey
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import declared_attr
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

class RefTargetMixin:
    target_id: Mapped[int] = mapped_column(ForeignKey("target.id"))

    @declared_attr
    def target(cls) -> Mapped["Target"]:
        return relationship("Target")

class Foo(RefTargetMixin, Base):
    __tablename__ = "foo"
    id: Mapped[int] = mapped_column(primary_key=True)

class Bar(RefTargetMixin, Base):
    __tablename__ = "bar"
    id: Mapped[int] = mapped_column(primary_key=True)

class Target(Base):
    __tablename__ = "target"
    id: Mapped[int] = mapped_column(primary_key=True)
```

使用上面的映射，`Foo`和`Bar`中的每一个都包含一个访问`.target`属性的到`Target`的关系：

```py
>>> from sqlalchemy import select
>>> print(select(Foo).join(Foo.target))
SELECT  foo.id,  foo.target_id
FROM  foo  JOIN  target  ON  target.id  =  foo.target_id
>>> print(select(Bar).join(Bar.target))
SELECT  bar.id,  bar.target_id
FROM  bar  JOIN  target  ON  target.id  =  bar.target_id 
```

特殊参数，比如`relationship.primaryjoin`，也可以在混合类方法中使用，这些方法通常需要引用正在映射的类。对于需要引用本地映射列的方案，在普通情况下，这些列将作为 Declarative 的属性在映射类上提供，并作为传递给装饰类方法的`cls`参数。利用这个特性，我们可以例如重新编写`RefTargetMixin.target`方法，使用明确的 primaryjoin，它引用了`Target`和`cls`上的待定映射列：

```py
class Target(Base):
    __tablename__ = "target"
    id: Mapped[int] = mapped_column(primary_key=True)

class RefTargetMixin:
    target_id: Mapped[int] = mapped_column(ForeignKey("target.id"))

    @declared_attr
    def target(cls) -> Mapped["Target"]:
        # illustrates explicit 'primaryjoin' argument
        return relationship("Target", primaryjoin=Target.id == cls.target_id)
```  ## 混合使用`column_property()` 和其他 `MapperProperty` 类

像`relationship()`一样，其他的`MapperProperty`子类，比如`column_property()`，在混合使用时也需要生成类本地副本，因此也在被`declared_attr`装饰的函数中声明。在函数内部，使用`mapped_column()`、`Mapped`或`Column`声明的其他普通映射列将从`cls`参数中提供，以便可以用于组合新的属性，如下面的示例，将两列相加：

```py
from sqlalchemy.orm import column_property
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import declared_attr
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column

class Base(DeclarativeBase):
    pass

class SomethingMixin:
    x: Mapped[int]
    y: Mapped[int]

    @declared_attr
    def x_plus_y(cls) -> Mapped[int]:
        return column_property(cls.x + cls.y)

class Something(SomethingMixin, Base):
    __tablename__ = "something"

    id: Mapped[int] = mapped_column(primary_key=True)
```

在上面的示例中，我们可以在语句中使用`Something.x_plus_y`，产生完整的表达式：

```py
>>> from sqlalchemy import select
>>> print(select(Something.x_plus_y))
SELECT  something.x  +  something.y  AS  anon_1
FROM  something 
```

提示

`declared_attr` 装饰器使装饰的可调用对象的行为与类方法完全相同。然而，像[Pylance](https://github.com/microsoft/pylance-release)这样的类型工具可能无法识别这一点，有时会因为无法访问函数体内的 `cls` 变量而抱怨。要解决此问题，当发生时，可以直接将 `@classmethod` 装饰器与`declared_attr` 结合使用，如下所示：

```py
class SomethingMixin:
    x: Mapped[int]
    y: Mapped[int]

    @declared_attr
    @classmethod
    def x_plus_y(cls) -> Mapped[int]:
        return column_property(cls.x + cls.y)
```

新版本 2.0 中：- `declared_attr` 可以适应使用 `@classmethod` 装饰的函数来协助[**PEP 484**](https://peps.python.org/pep-0484/)集成的需要。## 使用混合和基类进行映射继承模式

在处理如映射类继承层次结构中记录的映射器继承模式时，当使用 `declared_attr` 时，可以使用一些附加功能，无论是与混合类一起使用，还是在类层次结构中增加映射和未映射的超类时。

当在映射继承层次结构中由子类解释的函数由`declared_attr`装饰在混合或基类上定义时，必须区分两种情况，即生成 Declarative 使用的特殊名称如 `__tablename__`、`__mapper_args__` 与生成普通映射属性如`mapped_column()` 和`relationship()`。定义 **Declarative 指令** 的函数会 **在层次结构中的每个子类中调用**，而生成 **映射属性** 的函数只会 **在层次结构中的第一个映射的超类中调用**。

此行为差异的基本原理是，映射属性已经可以被类继承，例如，超类映射表上的特定列不应该在子类中重复出现，而特定于特定类或其映射表的元素不可继承，例如，局部映射的表的名称。

这两种用例之间行为上的差异在以下两个部分中得到了展示。

### 使用`declared_attr()`结合继承的`Table`和`Mapper`参数

使用 mixin 的一个常见方法是创建一个 `def __tablename__(cls)` 函数，动态生成映射的 `Table` 名称。

这个方法可以用于生成继承映射层次结构中的表名称，就像下面的示例一样，该示例创建一个 mixin，根据类名给每个类生成一个简单的表名称。下面的示例说明了如何为 `Person` 映射类和 `Person` 的 `Engineer` 子类生成表名称，但不为 `Person` 的 `Manager` 子类生成表名称：

```py
from typing import Optional

from sqlalchemy import ForeignKey
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import declared_attr
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column

class Base(DeclarativeBase):
    pass

class Tablename:
    @declared_attr.directive
    def __tablename__(cls) -> Optional[str]:
        return cls.__name__.lower()

class Person(Tablename, Base):
    id: Mapped[int] = mapped_column(primary_key=True)
    discriminator: Mapped[str]
    __mapper_args__ = {"polymorphic_on": "discriminator"}

class Engineer(Person):
    id: Mapped[int] = mapped_column(ForeignKey("person.id"), primary_key=True)

    primary_language: Mapped[str]

    __mapper_args__ = {"polymorphic_identity": "engineer"}

class Manager(Person):
    @declared_attr.directive
    def __tablename__(cls) -> Optional[str]:
  """override __tablename__ so that Manager is single-inheritance to Person"""

        return None

    __mapper_args__ = {"polymorphic_identity": "manager"}
```

在上述示例中，`Person` 基类和 `Engineer` 类都是 `Tablename` mixin 类的子类，该类生成新的表名称，因此它们都将具有生成的 `__tablename__` 属性。对于 Declarative，这意味着每个类都应该有自己的 `Table` 生成，并将其映射到其中。对于 `Engineer` 子类，所应用的继承风格是联合表继承，因为它将映射到一个与基本 `person` 表连接的 `engineer` 表。从 `Person` 继承的任何其他子类也将默认应用此继承风格（在此特定示例中，每个子类都需要指定一个主键列；关于这一点，后面会详细介绍）。

相比之下，`Person` 的 `Manager` 子类**覆盖**了 `__tablename__` 类方法，将其返回值设为 `None`。这表明对于 Declarative 来说，该类**不应**生成一个 `Table`，而应仅使用 `Person` 映射到的基本 `Table`。对于 `Manager` 子类，所应用的继承风格是单表继承。

上面的示例说明了 Declarative 指令（如 `__tablename__`）必须**分别应用于每个子类**，因为每个映射类都需要说明将映射到哪个 `Table`，或者是否将自身映射到继承的超类的 `Table`。

如果我们希望**反转**上面说明的默认表方案，使得单表继承成为默认，只有在提供了 `__tablename__` 指令以覆盖它时才能定义连接表继承，我们可以在顶级 `__tablename__()` 方法中使用 Declarative 助手，本例中称为 `has_inherited_table()`。此函数将返回 `True` 如果超类已经映射到一个 `Table`。我们可以在基类中的最低级 `__tablename__()` 类方法中使用此辅助函数，以便我们**有条件地**如果表已经存在，则返回 `None` 作为表名，从而默认为继承子类的单表继承：

```py
from sqlalchemy import ForeignKey
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import declared_attr
from sqlalchemy.orm import has_inherited_table
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column

class Base(DeclarativeBase):
    pass

class Tablename:
    @declared_attr.directive
    def __tablename__(cls):
        if has_inherited_table(cls):
            return None
        return cls.__name__.lower()

class Person(Tablename, Base):
    id: Mapped[int] = mapped_column(primary_key=True)
    discriminator: Mapped[str]
    __mapper_args__ = {"polymorphic_on": "discriminator"}

class Engineer(Person):
    @declared_attr.directive
    def __tablename__(cls):
  """override __tablename__ so that Engineer is joined-inheritance to Person"""

        return cls.__name__.lower()

    id: Mapped[int] = mapped_column(ForeignKey("person.id"), primary_key=True)

    primary_language: Mapped[str]

    __mapper_args__ = {"polymorphic_identity": "engineer"}

class Manager(Person):
    __mapper_args__ = {"polymorphic_identity": "manager"}
```

### 使用 `declared_attr()` 生成特定于表的继承列

与在与 `declared_attr` 一起使用时如何处理 `__tablename__` 和其他特殊名称不同，当我们混入列和属性（例如关系、列属性等）时，该函数仅在层次结构中的**基类**调用，除非结合使用 `declared_attr` 指令和 `declared_attr.cascading` 子指令。在下面的示例中，只有 `Person` 类将收到名为 `id` 的列；对于未给出主键的 `Engineer`，映射将失败：

```py
class HasId:
    id: Mapped[int] = mapped_column(primary_key=True)

class Person(HasId, Base):
    __tablename__ = "person"

    discriminator: Mapped[str]
    __mapper_args__ = {"polymorphic_on": "discriminator"}

# this mapping will fail, as there's no primary key
class Engineer(Person):
    __tablename__ = "engineer"

    primary_language: Mapped[str]
    __mapper_args__ = {"polymorphic_identity": "engineer"}
```

在连接表继承中，通常情况下我们希望每个子类具有不同命名的列。然而在这种情况下，我们可能希望每个表上都有一个 `id` 列，并且通过外键相互引用。我们可以通过使用 `declared_attr.cascading` 修饰符作为 mixin 来实现这一点，该修饰符表示应该**对层次结构中的每个类**调用该函数，几乎（见下面的警告）与对 `__tablename__` 调用方式相同：

```py
class HasIdMixin:
    @declared_attr.cascading
    def id(cls) -> Mapped[int]:
        if has_inherited_table(cls):
            return mapped_column(ForeignKey("person.id"), primary_key=True)
        else:
            return mapped_column(Integer, primary_key=True)

class Person(HasIdMixin, Base):
    __tablename__ = "person"

    discriminator: Mapped[str]
    __mapper_args__ = {"polymorphic_on": "discriminator"}

class Engineer(Person):
    __tablename__ = "engineer"

    primary_language: Mapped[str]
    __mapper_args__ = {"polymorphic_identity": "engineer"}
```

警告

目前，`declared_attr.cascading` 功能**不允许**子类使用不同的函数或值覆盖属性。这是在如何解析 `@declared_attr` 的机制中的当前限制，并且如果检测到此条件，则会发出警告。此限制仅适用于 ORM 映射的列、关系和其他属性的`MapperProperty`风格。它**不**适用于诸如 `__tablename__`、`__mapper_args__` 等的声明性指令，这些指令在内部以与`declared_attr.cascading`不同的方式解析。

## 从多个混合类组合表/映射器参数

当使用声明性混合类指定 `__table_args__` 或 `__mapper_args__` 时，您可能希望将一些参数从多个混合类中与您希望在类本身上定义的参数结合起来。可以在这里使用 `declared_attr` 装饰器来创建用户定义的排序例程，该例程从多个集合中提取：

```py
from sqlalchemy.orm import declarative_mixin, declared_attr

class MySQLSettings:
    __table_args__ = {"mysql_engine": "InnoDB"}

class MyOtherMixin:
    __table_args__ = {"info": "foo"}

class MyModel(MySQLSettings, MyOtherMixin, Base):
    __tablename__ = "my_model"

    @declared_attr.directive
    def __table_args__(cls):
        args = dict()
        args.update(MySQLSettings.__table_args__)
        args.update(MyOtherMixin.__table_args__)
        return args

    id = mapped_column(Integer, primary_key=True)
```

## 在混合类上使用命名约定创建索引和约束

对于具有命名约束的使用，如 `Index`、`UniqueConstraint`、`CheckConstraint`，其中每个对象应该是唯一的，针对从混合类派生的特定表，需要为每个实际映射的类创建每个对象的单独实例。

作为一个简单的示例，要定义一个命名的、可能是多列的 `Index`，该索引适用于从混合类派生的所有表，可以使用 `Index` 的“内联”形式，并将其建立为 `__table_args__` 的一部分，使用 `declared_attr` 来建立 `__table_args__()` 作为一个类方法，该方法将被每个子类调用：

```py
class MyMixin:
    a = mapped_column(Integer)
    b = mapped_column(Integer)

    @declared_attr.directive
    def __table_args__(cls):
        return (Index(f"test_idx_{cls.__tablename__}", "a", "b"),)

class MyModelA(MyMixin, Base):
    __tablename__ = "table_a"
    id = mapped_column(Integer, primary_key=True)

class MyModelB(MyMixin, Base):
    __tablename__ = "table_b"
    id = mapped_column(Integer, primary_key=True)
```

上面的示例将生成两个表 `"table_a"` 和 `"table_b"`，其中包含索引 `"test_idx_table_a"` 和 `"test_idx_table_b"`

通常，在现代 SQLAlchemy 中，我们会使用一种命名约定，如在配置约束命名约定中记录的那样。虽然命名约定会在创建新的`Constraint`对象时自动进行，因为此约定是在基于特定`Constraint`的父`Table`的对象构造时间应用的，因此需要为每个继承子类创建一个不同的`Constraint`对象，并再次使用`declared_attr`与`__table_args__()`，下面通过使用抽象映射基类进行说明：

```py
from uuid import UUID

from sqlalchemy import CheckConstraint
from sqlalchemy import create_engine
from sqlalchemy import MetaData
from sqlalchemy import UniqueConstraint
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import declared_attr
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column

constraint_naming_conventions = {
    "ix": "ix_%(column_0_label)s",
    "uq": "uq_%(table_name)s_%(column_0_name)s",
    "ck": "ck_%(table_name)s_%(constraint_name)s",
    "fk": "fk_%(table_name)s_%(column_0_name)s_%(referred_table_name)s",
    "pk": "pk_%(table_name)s",
}

class Base(DeclarativeBase):
    metadata = MetaData(naming_convention=constraint_naming_conventions)

class MyAbstractBase(Base):
    __abstract__ = True

    @declared_attr.directive
    def __table_args__(cls):
        return (
            UniqueConstraint("uuid"),
            CheckConstraint("x > 0 OR y < 100", name="xy_chk"),
        )

    id: Mapped[int] = mapped_column(primary_key=True)
    uuid: Mapped[UUID]
    x: Mapped[int]
    y: Mapped[int]

class ModelAlpha(MyAbstractBase):
    __tablename__ = "alpha"

class ModelBeta(MyAbstractBase):
    __tablename__ = "beta"
```

上述映射将生成包含所有约束的表特定名称的 DDL，包括主键、CHECK 约束、唯一约束：

```py
CREATE  TABLE  alpha  (
  id  INTEGER  NOT  NULL,
  uuid  CHAR(32)  NOT  NULL,
  x  INTEGER  NOT  NULL,
  y  INTEGER  NOT  NULL,
  CONSTRAINT  pk_alpha  PRIMARY  KEY  (id),
  CONSTRAINT  uq_alpha_uuid  UNIQUE  (uuid),
  CONSTRAINT  ck_alpha_xy_chk  CHECK  (x  >  0  OR  y  <  100)
)

CREATE  TABLE  beta  (
  id  INTEGER  NOT  NULL,
  uuid  CHAR(32)  NOT  NULL,
  x  INTEGER  NOT  NULL,
  y  INTEGER  NOT  NULL,
  CONSTRAINT  pk_beta  PRIMARY  KEY  (id),
  CONSTRAINT  uq_beta_uuid  UNIQUE  (uuid),
  CONSTRAINT  ck_beta_xy_chk  CHECK  (x  >  0  OR  y  <  100)
)
```

## 增强基类

除了使用纯混合外，本节中的大多数技术也可以直接应用于基类，用于适用于从特定基类派生的所有类的模式。下面的示例说明了如何在`Base`类方面应用上一节的一些示例：

```py
from sqlalchemy import ForeignKey
from sqlalchemy.orm import declared_attr
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
  """define a series of common elements that may be applied to mapped
 classes using this class as a base class."""

    @declared_attr.directive
    def __tablename__(cls) -> str:
        return cls.__name__.lower()

    __table_args__ = {"mysql_engine": "InnoDB"}
    __mapper_args__ = {"eager_defaults": True}

    id: Mapped[int] = mapped_column(primary_key=True)

class HasLogRecord:
  """mark classes that have a many-to-one relationship to the
 ``LogRecord`` class."""

    log_record_id: Mapped[int] = mapped_column(ForeignKey("logrecord.id"))

    @declared_attr
    def log_record(self) -> Mapped["LogRecord"]:
        return relationship("LogRecord")

class LogRecord(Base):
    log_info: Mapped[str]

class MyModel(HasLogRecord, Base):
    name: Mapped[str]
```

在上述示例中，`MyModel` 和 `LogRecord`，在派生自`Base`时，它们的表名都将根据其类名派生，一个名为`id`的主键列，以及由`Base.__table_args__`和`Base.__mapper_args__`定义的上述表和映射器参数。

在使用旧版`declarative_base()`或`registry.generate_base()`时，可以使用`declarative_base.cls`参数来生成等效效果，如下所示的未注释示例：

```py
# legacy declarative_base() use

from sqlalchemy import Integer, String
from sqlalchemy import ForeignKey
from sqlalchemy.orm import declared_attr
from sqlalchemy.orm import declarative_base
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship

class Base:
  """define a series of common elements that may be applied to mapped
 classes using this class as a base class."""

    @declared_attr.directive
    def __tablename__(cls):
        return cls.__name__.lower()

    __table_args__ = {"mysql_engine": "InnoDB"}
    __mapper_args__ = {"eager_defaults": True}

    id = mapped_column(Integer, primary_key=True)

Base = declarative_base(cls=Base)

class HasLogRecord:
  """mark classes that have a many-to-one relationship to the
 ``LogRecord`` class."""

    log_record_id = mapped_column(ForeignKey("logrecord.id"))

    @declared_attr
    def log_record(self):
        return relationship("LogRecord")

class LogRecord(Base):
    log_info = mapped_column(String)

class MyModel(HasLogRecord, Base):
    name = mapped_column(String)
```

## 混合列

如果使用声明式表的配置风格（而不是命令式表配置），则可以在混合类中指示列，以便在声明式过程生成的 `Table` 的一部分。可以在声明式混合类中内联声明三种构造：`mapped_column()`、`Mapped` 和 `Column`：

```py
class TimestampMixin:
    created_at: Mapped[datetime] = mapped_column(default=func.now())
    updated_at: Mapped[datetime]

class MyModel(TimestampMixin, Base):
    __tablename__ = "test"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
```

在上述情况下，所有包括 `TimestampMixin` 在其类基类中的声明式类将自动包含一个名为 `created_at` 的列，该列对所有行插入应用时间戳，以及一个名为 `updated_at` 的列，该列不包含默认值以示例为目的（如果有的话，我们将使用 `Column.onupdate` 参数，该参数被 `mapped_column()` 接受）。这些列构造始终从源混合类或基类**复制**，因此可以将相同的混合类/基类应用于任意数量的目标类，每个目标类都将有自己的列构造。

所有声明式列形式都受到混合类的支持，包括：

+   **带注释的属性** - 无论是否存在 `mapped_column()`：

    ```py
    class TimestampMixin:
        created_at: Mapped[datetime] = mapped_column(default=func.now())
        updated_at: Mapped[datetime]
    ```

+   **mapped_column** - 无论是否存在 `Mapped`：

    ```py
    class TimestampMixin:
        created_at = mapped_column(default=func.now())
        updated_at: Mapped[datetime] = mapped_column()
    ```

+   **Column** - 传统的声明式形式：

    ```py
    class TimestampMixin:
        created_at = Column(DateTime, default=func.now())
        updated_at = Column(DateTime)
    ```

在上述每种形式中，声明式通过创建构造的**副本**来处理混合类上的基于列的属性，然后将其应用于目标类。

**版本 2.0 中的变化**：声明式 API 现在可以接受 `Column` 对象以及任何形式的 `mapped_column()` 构造，当使用混合类时无需使用 `declared_attr()`。已经移除了以前的限制，这些限制阻止直接在混合类中使用具有 `ForeignKey` 元素的列。

## 混入关系

通过`relationship()`创建的关系通过`declared_attr`方法提供的声明式混合类，排除了在复制关系及其可能与列绑定的内容时可能出现的任何歧义。下面是一个示例，将外键列和关系组合在一起，以便两个类`Foo`和`Bar`都可以配置为通过多对一引用一个共同的目标类：

```py
from sqlalchemy import ForeignKey
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import declared_attr
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

class RefTargetMixin:
    target_id: Mapped[int] = mapped_column(ForeignKey("target.id"))

    @declared_attr
    def target(cls) -> Mapped["Target"]:
        return relationship("Target")

class Foo(RefTargetMixin, Base):
    __tablename__ = "foo"
    id: Mapped[int] = mapped_column(primary_key=True)

class Bar(RefTargetMixin, Base):
    __tablename__ = "bar"
    id: Mapped[int] = mapped_column(primary_key=True)

class Target(Base):
    __tablename__ = "target"
    id: Mapped[int] = mapped_column(primary_key=True)
```

使用上述映射，每个`Foo`和`Bar`都包含一个到`Target`的关系，通过`.target`属性访问：

```py
>>> from sqlalchemy import select
>>> print(select(Foo).join(Foo.target))
SELECT  foo.id,  foo.target_id
FROM  foo  JOIN  target  ON  target.id  =  foo.target_id
>>> print(select(Bar).join(Bar.target))
SELECT  bar.id,  bar.target_id
FROM  bar  JOIN  target  ON  target.id  =  bar.target_id 
```

类似`relationship.primaryjoin`的特殊参数也可以在混入的 classmethod 中使用，这些参数通常需要引用正在映射的类。对于需要引用本地映射列的方案，在普通情况下，这些列通过 Declarative 作为映射类的属性提供，该类作为参数`cls`传递给修饰的 classmethod。使用此功能，我们可以例如使用显式的 primaryjoin 重写`RefTargetMixin.target`方法，该方法引用了`Target`和`cls`上的待定映射列：

```py
class Target(Base):
    __tablename__ = "target"
    id: Mapped[int] = mapped_column(primary_key=True)

class RefTargetMixin:
    target_id: Mapped[int] = mapped_column(ForeignKey("target.id"))

    @declared_attr
    def target(cls) -> Mapped["Target"]:
        # illustrates explicit 'primaryjoin' argument
        return relationship("Target", primaryjoin=Target.id == cls.target_id)
```

## 混合使用`column_property()`和其他`MapperProperty`类

与`relationship()`类似，其他`MapperProperty`子类，如`column_property()`在被混合使用时也需要生成类局部副本，因此在被`declared_attr`修饰的函数内声明。在该函数内，使用`mapped_column()`、`Mapped`或`Column`声明的其他普通映射列将从`cls`参数中提取，以便它们可以被用来组合新的属性，如下例所示，将两个列相加：

```py
from sqlalchemy.orm import column_property
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import declared_attr
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column

class Base(DeclarativeBase):
    pass

class SomethingMixin:
    x: Mapped[int]
    y: Mapped[int]

    @declared_attr
    def x_plus_y(cls) -> Mapped[int]:
        return column_property(cls.x + cls.y)

class Something(SomethingMixin, Base):
    __tablename__ = "something"

    id: Mapped[int] = mapped_column(primary_key=True)
```

在上面的例子中，我们可以在生成完整表达式的语句中使用`Something.x_plus_y`：

```py
>>> from sqlalchemy import select
>>> print(select(Something.x_plus_y))
SELECT  something.x  +  something.y  AS  anon_1
FROM  something 
```

提示

`declared_attr`装饰器使装饰的可调用对象表现得完全像一个类方法。然而，像[Pylance](https://github.com/microsoft/pylance-release)这样的类型工具可能无法识别这一点，这有时会导致它在函数体内部抱怨无法访问`cls`变量。当出现此问题时，可以直接将`@classmethod`装饰器与`declared_attr`结合使用来解决：

```py
class SomethingMixin:
    x: Mapped[int]
    y: Mapped[int]

    @declared_attr
    @classmethod
    def x_plus_y(cls) -> Mapped[int]:
        return column_property(cls.x + cls.y)
```

新版本 2.0 中：- `declared_attr`可以容纳使用`@classmethod`装饰的函数，以帮助需要的[**PEP 484**](https://peps.python.org/pep-0484/)集成。

## 使用混合类和基类与映射继承模式

在处理映射类继承层次结构中所记录的映射继承模式时，使用`declared_attr`时，无论是使用混合类还是在类层次结构中增加映射和非映射的超类时，都会存在一些额外的功能。

当在混合类或基类上定义由`declared_attr`装饰的函数，以便由映射继承层次结构中的子类解释时，有一个重要的区别是函数生成特殊名称（例如`__tablename__`、`__mapper_args__`）与生成普通映射属性（例如`mapped_column()`和`relationship()`）之间的区别。定义**声明性指令**的函数在层次结构中的每个子类中都会被**调用**，而生成**映射属性**的函数仅在层次结构中的第一个映射的超类中被**调用**。

此行为差异的原理是映射属性已经可以被类继承，例如，超类映射表上的特定列不应该重复到子类中，而特定于特定类或其映射表的元素不可继承，例如本地映射的表名。

这两种用例之间行为上的差异在以下两个部分中得到展示。

### 使用带有继承`Table`和`Mapper`参数的`declared_attr()`

使用混合类的常见方法是创建一个`def __tablename__(cls)`函数，动态生成映射的`Table`的名称。

这个配方可用于为继承映射器层次结构生成表名，如下例所示，该示例创建了一个混合类，根据类名为每个类提供一个简单的表名。下面的示例说明了这个配方，在这个示例中为`Person`映射类和`Person`的子类`Engineer`生成了一个表名，但没有为`Person`的子类`Manager`生成表名：

```py
from typing import Optional

from sqlalchemy import ForeignKey
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import declared_attr
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column

class Base(DeclarativeBase):
    pass

class Tablename:
    @declared_attr.directive
    def __tablename__(cls) -> Optional[str]:
        return cls.__name__.lower()

class Person(Tablename, Base):
    id: Mapped[int] = mapped_column(primary_key=True)
    discriminator: Mapped[str]
    __mapper_args__ = {"polymorphic_on": "discriminator"}

class Engineer(Person):
    id: Mapped[int] = mapped_column(ForeignKey("person.id"), primary_key=True)

    primary_language: Mapped[str]

    __mapper_args__ = {"polymorphic_identity": "engineer"}

class Manager(Person):
    @declared_attr.directive
    def __tablename__(cls) -> Optional[str]:
  """override __tablename__ so that Manager is single-inheritance to Person"""

        return None

    __mapper_args__ = {"polymorphic_identity": "manager"}
```

在上面的示例中，`Person`基类以及`Engineer`类，作为生成新表名的`Tablename`混合类的子类，都将有一个生成的`__tablename__`属性，这对于 Declarative 表示每个类应该有自己的`Table`生成并映射到它。对于`Engineer`子类，应用的继承风格是联接表继承，因为它将映射到一个与基本`person`表连接的`engineer`表。从`Person`继承的任何其他子类也将默认应用这种继承风格（在这个特定示例中，需要为每个子类指定一个主键列；在下一节中会详细介绍）。

相比之下，`Person`的子类`Manager` **覆盖** 了`__tablename__`类方法以返回`None`。这告诉 Declarative 这个类 **不应** 生成一个`Table`，而应该完全使用`Person`映射到的基本`Table`。对于`Manager`子类，应用的继承风格是单表继承。

上面的示例说明了 Declarative 指令如`__tablename__`必须**分别应用于每个子类**，因为每个映射类都需要声明将映射到哪个`Table`，或者是否将自身映射到继承的超类的`Table`。

如果我们希望**反转**上述默认表方案，使单表继承成为默认情况，并且只有在提供`__tablename__`指令以覆盖它时才能定义联接表继承，则可以在顶层`__tablename__()`方法中使用 Declarative 辅助函数，在本例中是一个称为`has_inherited_table()`的辅助函数。如果超类已经映射到`Table`，此函数将返回`True`。我们可以在最基本的`__tablename__()`类方法中使用此辅助函数，以便在表已存在时**有条件地**返回`None`作为表名，从而默认情况下通过继承子类进行单表继承：

```py
from sqlalchemy import ForeignKey
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import declared_attr
from sqlalchemy.orm import has_inherited_table
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column

class Base(DeclarativeBase):
    pass

class Tablename:
    @declared_attr.directive
    def __tablename__(cls):
        if has_inherited_table(cls):
            return None
        return cls.__name__.lower()

class Person(Tablename, Base):
    id: Mapped[int] = mapped_column(primary_key=True)
    discriminator: Mapped[str]
    __mapper_args__ = {"polymorphic_on": "discriminator"}

class Engineer(Person):
    @declared_attr.directive
    def __tablename__(cls):
  """override __tablename__ so that Engineer is joined-inheritance to Person"""

        return cls.__name__.lower()

    id: Mapped[int] = mapped_column(ForeignKey("person.id"), primary_key=True)

    primary_language: Mapped[str]

    __mapper_args__ = {"polymorphic_identity": "engineer"}

class Manager(Person):
    __mapper_args__ = {"polymorphic_identity": "manager"}
```

### 使用`declared_attr()`生成特定表继承列

与在使用`declared_attr`时处理`__tablename__`和其他特殊名称的方式相反，当我们混合列和属性（例如关系、列属性等）时，该函数仅在层次结构中的**基类**上调用，除非在与`declared_attr.cascading`子指令结合使用时使用`declared_attr`指令。在下面的示例中，只有`Person`类将接收一个名为`id`的列；对于未给出主键的`Engineer`，映射将失败：

```py
class HasId:
    id: Mapped[int] = mapped_column(primary_key=True)

class Person(HasId, Base):
    __tablename__ = "person"

    discriminator: Mapped[str]
    __mapper_args__ = {"polymorphic_on": "discriminator"}

# this mapping will fail, as there's no primary key
class Engineer(Person):
    __tablename__ = "engineer"

    primary_language: Mapped[str]
    __mapper_args__ = {"polymorphic_identity": "engineer"}
```

在联接表继承中，通常情况下我们希望每个子类上都有不同命名的列。然而，在这种情况下，我们可能希望在每个表上都有一个`id`列，并通过外键引用它们。我们可以通过使用`declared_attr.cascading`修饰符作为混合来实现这一点，该修饰符指示该函数应该以*几乎*（请参阅下面的警告）与`__tablename__`相同的方式为层次结构中的每个类调用：  

```py
class HasIdMixin:
    @declared_attr.cascading
    def id(cls) -> Mapped[int]:
        if has_inherited_table(cls):
            return mapped_column(ForeignKey("person.id"), primary_key=True)
        else:
            return mapped_column(Integer, primary_key=True)

class Person(HasIdMixin, Base):
    __tablename__ = "person"

    discriminator: Mapped[str]
    __mapper_args__ = {"polymorphic_on": "discriminator"}

class Engineer(Person):
    __tablename__ = "engineer"

    primary_language: Mapped[str]
    __mapper_args__ = {"polymorphic_identity": "engineer"}
```

警告

`declared_attr.cascading` 特性目前**不允许**子类使用不同的函数或值覆盖属性。这是在解析`@declared_attr`的机制中的当前限制，并且如果检测到此条件，则会发出警告。这个限制仅适用于 ORM 映射的列、关系和其他`MapperProperty`风格的属性。它**不适用于**声明性指令，如`__tablename__`、`__mapper_args__`等，在内部的解析方式与`declared_attr.cascading`不同。

### 使用 `declared_attr()` 与继承的 `Table` 和 `Mapper` 参数

使用 mixin 的一个常见方法是创建一个`def __tablename__(cls)`函数，该函数动态生成映射的 `Table` 名称。

此示例可用于为继承的映射器层次结构生成表名，如下例所示，它创建了一个基于类名的简单表名的 mixin。下面的示例说明了该示例，其中为`Person`映射类和`Person`的`Engineer`子类生成了一个表名，但未为`Person`的`Manager`子类生成表名：

```py
from typing import Optional

from sqlalchemy import ForeignKey
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import declared_attr
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column

class Base(DeclarativeBase):
    pass

class Tablename:
    @declared_attr.directive
    def __tablename__(cls) -> Optional[str]:
        return cls.__name__.lower()

class Person(Tablename, Base):
    id: Mapped[int] = mapped_column(primary_key=True)
    discriminator: Mapped[str]
    __mapper_args__ = {"polymorphic_on": "discriminator"}

class Engineer(Person):
    id: Mapped[int] = mapped_column(ForeignKey("person.id"), primary_key=True)

    primary_language: Mapped[str]

    __mapper_args__ = {"polymorphic_identity": "engineer"}

class Manager(Person):
    @declared_attr.directive
    def __tablename__(cls) -> Optional[str]:
  """override __tablename__ so that Manager is single-inheritance to Person"""

        return None

    __mapper_args__ = {"polymorphic_identity": "manager"}
```

在上面的示例中，`Person`基类和`Engineer`类都是`Tablename` mixin 类的子类，该类生成新的表名，因此它们都会有一个生成的`__tablename__`属性，对于声明性来说，这表示每个类都应该有自己的 `Table` 生成，并且将映射到该表。对于`Engineer`子类，应用的继承风格是联接表继承，因为它将映射到一个连接到基本`person`表的表`engineer`。从`Person`继承的任何其他子类也将默认应用此继承风格（并且在这个特定示例中，每个子类都需要指定一个主键列；更多关于这一点的内容将在下一节中介绍）。

相比之下，`Person`的`Manager`子类**覆盖**了`__tablename__`类方法以返回`None`。这表明对于 Declarative 来说，这个类应该**不**生成一个`Table`，而是完全使用`Person`被映射到的基础`Table`。对于`Manager`子类，应用的继承样式是单表继承。

上面的示例说明 Declarative 指令如`__tablename__`必须**分别应用于每个子类**，因为每个映射类都需要说明它将映射到哪个`Table`，或者它将自行映射到继承超类的`Table`。

如果我们希望**反转**上面示例中的默认表方案，使得单表继承成为默认，并且只有在提供了`__tablename__`指令来覆盖它时才能定义连接表继承，我们可以在最顶层的`__tablename__()`方法中使用 Declarative 助手，本例中称为`has_inherited_table()`。此函数将在超类已经映射到`Table`时返回`True`。我们可以在最基本的`__tablename__()`类方法中使用此助手，以便我们可以在表已经存在时**有条件地**返回`None`作为表名，从而默认指示继承子类的单表继承：

```py
from sqlalchemy import ForeignKey
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import declared_attr
from sqlalchemy.orm import has_inherited_table
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column

class Base(DeclarativeBase):
    pass

class Tablename:
    @declared_attr.directive
    def __tablename__(cls):
        if has_inherited_table(cls):
            return None
        return cls.__name__.lower()

class Person(Tablename, Base):
    id: Mapped[int] = mapped_column(primary_key=True)
    discriminator: Mapped[str]
    __mapper_args__ = {"polymorphic_on": "discriminator"}

class Engineer(Person):
    @declared_attr.directive
    def __tablename__(cls):
  """override __tablename__ so that Engineer is joined-inheritance to Person"""

        return cls.__name__.lower()

    id: Mapped[int] = mapped_column(ForeignKey("person.id"), primary_key=True)

    primary_language: Mapped[str]

    __mapper_args__ = {"polymorphic_identity": "engineer"}

class Manager(Person):
    __mapper_args__ = {"polymorphic_identity": "manager"}
```

### 使用`declared_attr()`生成特定于表的继承列

与在使用`declared_attr`时处理`__tablename__`和其他特殊名称的方式相反，当我们混合列和属性（例如关系、列属性等）时，该函数仅在层次结构中的**基类**中调用，除非与`declared_attr.cascading`子指令结合使用`declared_attr`指令。下面，只有`Person`类将收到名为`id`的列；对于未给出主键的`Engineer`，映射将失败：

```py
class HasId:
    id: Mapped[int] = mapped_column(primary_key=True)

class Person(HasId, Base):
    __tablename__ = "person"

    discriminator: Mapped[str]
    __mapper_args__ = {"polymorphic_on": "discriminator"}

# this mapping will fail, as there's no primary key
class Engineer(Person):
    __tablename__ = "engineer"

    primary_language: Mapped[str]
    __mapper_args__ = {"polymorphic_identity": "engineer"}
```

在连接表继承中通常情况下，我们希望每个子类都有具有不同名称的列。但在这种情况下，我们可能希望在每个表上都有一个`id`列，并且通过外键相互引用。我们可以通过使用`declared_attr.cascading`修饰符作为混入来实现此目的，该修饰符指示该函数应在**层次结构中的每个类**中调用，与`__tablename__`几乎（参见下面的警告）相同的方式：

```py
class HasIdMixin:
    @declared_attr.cascading
    def id(cls) -> Mapped[int]:
        if has_inherited_table(cls):
            return mapped_column(ForeignKey("person.id"), primary_key=True)
        else:
            return mapped_column(Integer, primary_key=True)

class Person(HasIdMixin, Base):
    __tablename__ = "person"

    discriminator: Mapped[str]
    __mapper_args__ = {"polymorphic_on": "discriminator"}

class Engineer(Person):
    __tablename__ = "engineer"

    primary_language: Mapped[str]
    __mapper_args__ = {"polymorphic_identity": "engineer"}
```

警告

`declared_attr.cascading` 特性目前**不**允许子类使用不同的函数或值来覆盖属性。这是在解析`@declared_attr`时当前机制的限制，并且如果检测到此条件，则会发出警告。此限制仅适用于 ORM 映射的列、关联和其他`MapperProperty`风格的属性。它**不**适用于诸如`__tablename__`、`__mapper_args__`等的声明性指令，后者在内部解析方式与`declared_attr.cascading`不同。

## 将来自多个混入的表/映射器参数组合起来

在使用声明性混入指定的`__table_args__`或`__mapper_args__`的情况下，您可能希望将几个混入的一些参数与您希望在类本身上定义的参数合并。在这里可以使用`declared_attr`装饰器来创建用户定义的排序例程，这些例程来自多个集合：

```py
from sqlalchemy.orm import declarative_mixin, declared_attr

class MySQLSettings:
    __table_args__ = {"mysql_engine": "InnoDB"}

class MyOtherMixin:
    __table_args__ = {"info": "foo"}

class MyModel(MySQLSettings, MyOtherMixin, Base):
    __tablename__ = "my_model"

    @declared_attr.directive
    def __table_args__(cls):
        args = dict()
        args.update(MySQLSettings.__table_args__)
        args.update(MyOtherMixin.__table_args__)
        return args

    id = mapped_column(Integer, primary_key=True)
```

## 使用混入创建具有命名约定的索引和约束

使用命名约束，如`Index`、`UniqueConstraint`、`CheckConstraint`，其中每个对象应该是从混入派生的特定表上唯一的，需要为每个实际映射类创建每个对象的单个实例。

作为一个简单的例子，要定义一个命名的、可能是多列的`Index`，该索引适用于从混合类派生的所有表，可以使用`Index`的“内联”形式，并将其建立为`__table_args__`的一部分，使用`declared_attr`来建立`__table_args__()`作为一个类方法，该方法将被调用用于每个子类：

```py
class MyMixin:
    a = mapped_column(Integer)
    b = mapped_column(Integer)

    @declared_attr.directive
    def __table_args__(cls):
        return (Index(f"test_idx_{cls.__tablename__}", "a", "b"),)

class MyModelA(MyMixin, Base):
    __tablename__ = "table_a"
    id = mapped_column(Integer, primary_key=True)

class MyModelB(MyMixin, Base):
    __tablename__ = "table_b"
    id = mapped_column(Integer, primary_key=True)
```

上面的例子将生成两个表`"table_a"`和`"table_b"`，带有索引`"test_idx_table_a"`和`"test_idx_table_b"`。

通常，在现代 SQLAlchemy 中，我们会使用命名约定，如配置约束命名约定中所述。虽然命名约定在创建新的`Constraint`对象时会自动进行，因为该约定是根据特定的父`Table`在对象构建时应用于对象构建时应用于对象构建时应用于对象构建时应用于对象构建时应用于对象构建时应用于对象构建时应用于对象构建时应用于对象构建时应用于对象构建时应用于对象构建时应用于对象构建时应用于对象构建时应用于对象构建时应用于对象构建时应用于对象构建时应用于对象构建时应用于对象构建时应用于对象构建时应用于对象构建时时的，需要为每个继承子类创建一个独立的`Constraint`对象，并再次使用`declared_attr`与`__table_args__()`，下面以抽象映射基类进行说明：

```py
from uuid import UUID

from sqlalchemy import CheckConstraint
from sqlalchemy import create_engine
from sqlalchemy import MetaData
from sqlalchemy import UniqueConstraint
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import declared_attr
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column

constraint_naming_conventions = {
    "ix": "ix_%(column_0_label)s",
    "uq": "uq_%(table_name)s_%(column_0_name)s",
    "ck": "ck_%(table_name)s_%(constraint_name)s",
    "fk": "fk_%(table_name)s_%(column_0_name)s_%(referred_table_name)s",
    "pk": "pk_%(table_name)s",
}

class Base(DeclarativeBase):
    metadata = MetaData(naming_convention=constraint_naming_conventions)

class MyAbstractBase(Base):
    __abstract__ = True

    @declared_attr.directive
    def __table_args__(cls):
        return (
            UniqueConstraint("uuid"),
            CheckConstraint("x > 0 OR y < 100", name="xy_chk"),
        )

    id: Mapped[int] = mapped_column(primary_key=True)
    uuid: Mapped[UUID]
    x: Mapped[int]
    y: Mapped[int]

class ModelAlpha(MyAbstractBase):
    __tablename__ = "alpha"

class ModelBeta(MyAbstractBase):
    __tablename__ = "beta"
```

上述映射将生成包括所有约束的特定于表的名称的 DDL，包括主键、CHECK 约束、唯一约束：

```py
CREATE  TABLE  alpha  (
  id  INTEGER  NOT  NULL,
  uuid  CHAR(32)  NOT  NULL,
  x  INTEGER  NOT  NULL,
  y  INTEGER  NOT  NULL,
  CONSTRAINT  pk_alpha  PRIMARY  KEY  (id),
  CONSTRAINT  uq_alpha_uuid  UNIQUE  (uuid),
  CONSTRAINT  ck_alpha_xy_chk  CHECK  (x  >  0  OR  y  <  100)
)

CREATE  TABLE  beta  (
  id  INTEGER  NOT  NULL,
  uuid  CHAR(32)  NOT  NULL,
  x  INTEGER  NOT  NULL,
  y  INTEGER  NOT  NULL,
  CONSTRAINT  pk_beta  PRIMARY  KEY  (id),
  CONSTRAINT  uq_beta_uuid  UNIQUE  (uuid),
  CONSTRAINT  ck_beta_xy_chk  CHECK  (x  >  0  OR  y  <  100)
)
```
