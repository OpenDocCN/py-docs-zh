# 声明性映射样式

> 原文：[`docs.sqlalchemy.org/en/20/orm/declarative_styles.html`](https://docs.sqlalchemy.org/en/20/orm/declarative_styles.html)

如在 声明性映射 中介绍的，**声明性映射** 是现代 SQLAlchemy 中构建映射的典型方式。本节将概述可用于声明性映射器配置的形式。

## 使用声明性基类

最常见的方法是通过将 `DeclarativeBase` 超类作为子类生成“声明性基类”：

```py
from sqlalchemy.orm import DeclarativeBase

# declarative base class
class Base(DeclarativeBase):
    pass
```

也可以通过将现有的 `registry` 赋值为名为 `registry` 的类变量来创建声明性基类：

```py
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import registry

reg = registry()

# declarative base class
class Base(DeclarativeBase):
    registry = reg
```

在 2.0 版本中更改：`DeclarativeBase` 超类取代了 `declarative_base()` 函数和 `registry.generate_base()` 方法的使用；超类方法与 [**PEP 484**](https://peps.python.org/pep-0484/) 工具集成，无需使用插件。请参阅 ORM 声明模型 迁移说明。

使用声明性基类，新的映射类被声明为基类的子类：

```py
from datetime import datetime
from typing import Optional

from sqlalchemy import ForeignKey
from sqlalchemy import func
from sqlalchemy import Integer
from sqlalchemy import String
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "user"

    id = mapped_column(Integer, primary_key=True)
    name: Mapped[str]
    fullname: Mapped[Optional[str]]
    nickname: Mapped[Optional[str]] = mapped_column(String(64))
    create_date: Mapped[datetime] = mapped_column(insert_default=func.now())

    addresses: Mapped[List["Address"]] = relationship(back_populates="user")

class Address(Base):
    __tablename__ = "address"

    id = mapped_column(Integer, primary_key=True)
    user_id = mapped_column(ForeignKey("user.id"))
    email_address: Mapped[str]

    user: Mapped["User"] = relationship(back_populates="addresses")
```

上述中，`Base` 类作为新映射类的基础，如上所述，新映射类 `User` 和 `Address` 被构建。

对于每个构建的子类，类的主体随后遵循声明性映射方法，该方法在幕后定义了既是 `Table` 也是 `Mapper` 对象的一体化映射。

另请参阅

使用声明性表配置 - 描述如何指定生成的映射 `Table` 的组件，包括关于使用 `mapped_column()` 构造的说明和选项，以及它与 `Mapped` 注解类型的交互方式。

使用声明性配置的映射器 - 描述了声明性中 ORM 映射器配置的所有其他方面，包括`relationship()`配置、SQL 表达式和`Mapper`参数  ## 使用装饰器的声明性映射（无声明性基类）

作为使用“声明性基类”的替代方案，可以将声明性映射明确应用于类，方法是使用类似于“经典”映射的命令式技术，或者更简洁地使用装饰器。 `registry.mapped()`函数是一个类装饰器，可应用于任何没有层次结构的 Python 类。否则，Python 类通常以声明性样式进行配置。

下面的示例设置了与上一节中看到的相同的映射，使用`registry.mapped()`装饰器而不是使用`DeclarativeBase`超类：

```py
from datetime import datetime
from typing import List
from typing import Optional

from sqlalchemy import ForeignKey
from sqlalchemy import func
from sqlalchemy import Integer
from sqlalchemy import String
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import registry
from sqlalchemy.orm import relationship

mapper_registry = registry()

@mapper_registry.mapped
class User:
    __tablename__ = "user"

    id = mapped_column(Integer, primary_key=True)
    name: Mapped[str]
    fullname: Mapped[Optional[str]]
    nickname: Mapped[Optional[str]] = mapped_column(String(64))
    create_date: Mapped[datetime] = mapped_column(insert_default=func.now())

    addresses: Mapped[List["Address"]] = relationship(back_populates="user")

@mapper_registry.mapped
class Address:
    __tablename__ = "address"

    id = mapped_column(Integer, primary_key=True)
    user_id = mapped_column(ForeignKey("user.id"))
    email_address: Mapped[str]

    user: Mapped["User"] = relationship(back_populates="addresses")
```

当使用上述风格时，特定类的映射**仅**在直接应用装饰器到该类时进行。对于继承映射（在映射类继承层次结构中详细描述），应将装饰器应用于要映射的每个子类：

```py
from sqlalchemy.orm import registry

mapper_registry = registry()

@mapper_registry.mapped
class Person:
    __tablename__ = "person"

    person_id = mapped_column(Integer, primary_key=True)
    type = mapped_column(String, nullable=False)

    __mapper_args__ = {
        "polymorphic_on": type,
        "polymorphic_identity": "person",
    }

@mapper_registry.mapped
class Employee(Person):
    __tablename__ = "employee"

    person_id = mapped_column(ForeignKey("person.person_id"), primary_key=True)

    __mapper_args__ = {
        "polymorphic_identity": "employee",
    }
```

无论是使用声明性基类还是装饰器样式的声明性映射，都可以使用声明性表和命令式表表配置样式。

当将 SQLAlchemy 声明性映射与其他类仪器化系统（如[dataclasses](https://docs.python.org/3/library/dataclasses.html)和[attrs](https://pypi.org/project/attrs/)）结合使用时，装饰器形式的映射很有用，尽管请注意，SQLAlchemy 2.0 现在也具有与声明性基类的 dataclasses 集成。  ## 使用声明性基类

最常见的方法是通过对`DeclarativeBase`超类进行子类化来生成“声明性基类”：

```py
from sqlalchemy.orm import DeclarativeBase

# declarative base class
class Base(DeclarativeBase):
    pass
```

也可以通过将其分配为名为`registry`的类变量来使用现有的`registry`创建声明性基类：

```py
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import registry

reg = registry()

# declarative base class
class Base(DeclarativeBase):
    registry = reg
```

从版本 2.0 开始更改：`DeclarativeBase`超类取代了`declarative_base()`函数和`registry.generate_base()`方法的使用；超类方法集成了[**PEP 484**](https://peps.python.org/pep-0484/)工具，无需使用插件。有关迁移说明，请参见 ORM 声明模型。

使用声明性基类，新的映射类被声明为基类的子类：

```py
from datetime import datetime
from typing import Optional

from sqlalchemy import ForeignKey
from sqlalchemy import func
from sqlalchemy import Integer
from sqlalchemy import String
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "user"

    id = mapped_column(Integer, primary_key=True)
    name: Mapped[str]
    fullname: Mapped[Optional[str]]
    nickname: Mapped[Optional[str]] = mapped_column(String(64))
    create_date: Mapped[datetime] = mapped_column(insert_default=func.now())

    addresses: Mapped[List["Address"]] = relationship(back_populates="user")

class Address(Base):
    __tablename__ = "address"

    id = mapped_column(Integer, primary_key=True)
    user_id = mapped_column(ForeignKey("user.id"))
    email_address: Mapped[str]

    user: Mapped["User"] = relationship(back_populates="addresses")
```

在上面，`Base` 类充当要映射的新类的基类，如上所述，构造了新的映射类`User`和`Address`。

对于每个构造的子类，类的主体随后遵循声明性映射方法，该方法在幕后定义了一个`Table`以及一个`Mapper`对象，它们组成了一个完整的映射。

另请参见

使用声明性进行表配置 - 描述了如何指定要生成的映射`Table`的组件，包括有关使用`mapped_column()`构造的注释和选项以及它与`Mapped`注解类型的交互方式。

使用声明性进行映射配置 - 描述了在声明中进行的 ORM 映射器配置的所有其他方面，包括`relationship()`配置、SQL 表达式和`Mapper`参数。

## 使用装饰器进行声明性映射（无声明基类）

作为使用“声明基类”类的替代方法是显式地将声明映射应用于类，可以使用类似于“传统”映射的命令式技术，也可以更简洁地使用装饰器。 `registry.mapped()` 函数是一个类装饰器，可以应用于任何没有层次结构的 Python 类。否则，Python 类通常以声明样式配置。

下面的示例设置了与前一部分中相同的映射，使用 `registry.mapped()` 装饰器而不是使用 `DeclarativeBase` 超类：

```py
from datetime import datetime
from typing import List
from typing import Optional

from sqlalchemy import ForeignKey
from sqlalchemy import func
from sqlalchemy import Integer
from sqlalchemy import String
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import registry
from sqlalchemy.orm import relationship

mapper_registry = registry()

@mapper_registry.mapped
class User:
    __tablename__ = "user"

    id = mapped_column(Integer, primary_key=True)
    name: Mapped[str]
    fullname: Mapped[Optional[str]]
    nickname: Mapped[Optional[str]] = mapped_column(String(64))
    create_date: Mapped[datetime] = mapped_column(insert_default=func.now())

    addresses: Mapped[List["Address"]] = relationship(back_populates="user")

@mapper_registry.mapped
class Address:
    __tablename__ = "address"

    id = mapped_column(Integer, primary_key=True)
    user_id = mapped_column(ForeignKey("user.id"))
    email_address: Mapped[str]

    user: Mapped["User"] = relationship(back_populates="addresses")
```

在使用上述风格时，特定类的映射仅在直接将装饰器应用于该类时才会进行。对于继承映射（在映射类继承层次结构中详细描述），应该将装饰器应用于要映射的每个子类：

```py
from sqlalchemy.orm import registry

mapper_registry = registry()

@mapper_registry.mapped
class Person:
    __tablename__ = "person"

    person_id = mapped_column(Integer, primary_key=True)
    type = mapped_column(String, nullable=False)

    __mapper_args__ = {
        "polymorphic_on": type,
        "polymorphic_identity": "person",
    }

@mapper_registry.mapped
class Employee(Person):
    __tablename__ = "employee"

    person_id = mapped_column(ForeignKey("person.person_id"), primary_key=True)

    __mapper_args__ = {
        "polymorphic_identity": "employee",
    }
```

无论是 声明式表 还是 命令式表 的表配置风格都可以与声明式映射的声明基类或装饰器风格一起使用。

使用装饰器形式的映射在将 SQLAlchemy 声明式映射与其他类的装配系统（如[dataclasses](https://docs.python.org/3/library/dataclasses.html)和[attrs](https://pypi.org/project/attrs/)）结合时很有用，但要注意，SQLAlchemy 2.0 现在也支持在声明式基类中与 dataclasses 集成。
