# SQLAlchemy 2.0 有哪些新功能？

> 原文：[`docs.sqlalchemy.org/en/20/changelog/whatsnew_20.html`](https://docs.sqlalchemy.org/en/20/changelog/whatsnew_20.html)

读者注意事项

SQLAlchemy 2.0 的过渡文档分为 **两个** 文档 - 一个详细说明了从 1.x 到 2.x 系列的主要 API 转换，另一个详细说明了与 SQLAlchemy 1.4 相关的新功能和行为：

+   SQLAlchemy 2.0 - Major Migration Guide - 1.x 到 2.x API 转换

+   SQLAlchemy 2.0 有哪些新功能？ - 本文档，SQLAlchemy 2.0 的新功能和行为

尚未将其 1.4 应用程序更新为遵循 SQLAlchemy 2.0 引擎和 ORM 约定的读者可以导航到 SQLAlchemy 2.0 - Major Migration Guide 了解确保 SQLAlchemy 2.0 兼容性的指南，这是在版本 2.0 下拥有可工作代码的先决条件。

关于本文档

本文描述了 SQLAlchemy 版本 1.4 与版本 2.0 之间的变化，**与** 1.x 风格和 2.0 风格的主要变化无关。读者应该从 SQLAlchemy 2.0 - Major Migration Guide 文档开始，以了解 1.x 和 2.x 系列之间的主要兼容性变化的整体图片。

除了主要的 1.x->2.x 迁移路径之外，SQLAlchemy 2.0 中下一个最大的范式转变是与[**PEP 484**](https://peps.python.org/pep-0484/)类型实践和当前能力的深度集成，特别是在 ORM 中。受 Python [dataclasses](https://docs.python.org/3/library/dataclasses.html)启发的新型基于类型的 ORM 声明风格，以及与 dataclasses 本身的新集成，补充了一种不再需要存根并且在从 SQL 语句到结果集的类型感知方法链方面取得了很大进展的整体方法。

Python 类型的突出地位不仅仅在于使得诸如[mypy](https://mypy.readthedocs.io/en/stable/)之类的类型检查器可以无需插件而运行；更重要的是，它使得像[vscode](https://code.visualstudio.com/)和[pycharm](https://www.jetbrains.com/pycharm/)这样的集成开发环境能够在辅助编写 SQLAlchemy 应用程序时发挥更加积极的作用。

## Core 和 ORM 中的新类型支持 - 不再使用存根 / 扩展

与版本 1.4 中通过[sqlalchemy2-stubs](https://github.com/sqlalchemy/sqlalchemy2-stubs)包提供的临时方法相比，Core 和 ORM 的类型化方法已经完全重新设计。新方法从 SQLAlchemy 中最基本的元素开始，即`Column`，或者更准确地说是支撑所有具有类型的 SQL 表达式的`ColumnElement`。然后，这种表达级别的类型化扩展到语句构造、语句执行和结果集，并最终扩展到 ORM，其中新的 declarative 形式允许完全类型化的 ORM 模型，从语句到结果集完全集成。

提示

对于 2.0 系列，类型化支持应该被视为**beta 级别**软件。类型化细节可能会更改，但不计划进行重大的不兼容性更改。

### SQL 表达式/语句/结果集类型化

本节提供了关于 SQLAlchemy 新的 SQL 表达式类型化方法的背景和示例，该方法从基本的`ColumnElement`构造扩展到 SQL 语句和结果集，以及 ORM 映射的领域。

#### 理论基础和概述

提示

本节是一个架构讨论。跳转到 SQL Expression Typing - Examples 只需查看新类型化的外观。

在[sqlalchemy2-stubs](https://github.com/sqlalchemy/sqlalchemy2-stubs)中，SQL 表达式被标记为[泛型](https://peps.python.org/pep-0484/#generics)，然后引用一个`TypeEngine`对象，比如`Integer`、`DateTime`或`String`作为它们的泛型参数（例如`Column[Integer]`）。这本身就是与原始的 Dropbox [sqlalchemy-stubs](https://github.com/dropbox/sqlalchemy-stubs)包不同的地方，原始的包直接将`Column`及其基本构造标记为 Python 类型的泛型，比如`int`、`datetime`和`str`。人们希望由于`Integer` / `DateTime` / `String`本身与`int` / `datetime` / `str`泛型相关，会有方法来保持两个级别的信息，并能够通过中间构造`TypeEngine`从列表达式中提取 Python 类型。然而，事实并非如此，因为[**PEP 484**](https://peps.python.org/pep-0484/)实际上没有足够丰富的功能集来使这成为可行的选择，缺乏诸如[higher kinded TypeVars](https://github.com/python/typing/issues/548)之类的功能。

因此，在对[**PEP 484**](https://peps.python.org/pep-0484/)当前功能进行[深入评估](https://github.com/python/typing/discussions/999)之后，SQLAlchemy 2.0 认识到了在这个领域原始的[sqlalchemy-stubs](https://github.com/dropbox/sqlalchemy-stubs)的智慧，并回归到了直接将列表达式链接到 Python 类型的做法。这意味着，如果有 SQL 表达式到不同子类型的情况，比如`Column(VARCHAR)`与`Column(Unicode)`，这两种`String`子类型的具体细节不会被传递，因为类型只会传递`str`，但在实践中，这通常不是一个问题，直接出现 Python 类型通常更有用，因为它代表了将要直接存储和接收的 Python 数据。

具体来说，这意味着像 `Column('id', Integer)` 这样的表达式被类型为 `Column[int]`。这允许建立一个可行的 SQLAlchemy 构造 -> Python 数据类型的流水线，而无需使用类型插件。关键是，它允许完全与 ORM 的范式进行互操作，即使用引用 ORM 映射类类型的 `select()` 和 `Row` 构造（例如，包含用户映射实例的 `Row`，例如在我们的教程中使用的 `User` 和 `Address` 示例）。虽然 Python 的类型当前对于元组类型的定制支持非常有限（其中 [**PEP 646**](https://peps.python.org/pep-0646/)，第一个试图处理类似元组的对象的 pep，[在功能上受到故意限制](https://mail.python.org/archives/list/typing-sig@python.org/message/G2PNHRR32JMFD3JR7ACA2NDKWTDSEPUG/)，并且本身尚不适用于任意元组操作），但已经设计了一个相当不错的方法，允许基本的 `select()` -> `Result` -> `Row` 类型功能，包括用于 ORM 类的功能，在要将 `Row` 对象展开为单独的列条目时，会添加一个小的面向类型的访问器，允许各个 Python 值保持与它们来源于的 SQL 表达式相关联的 Python 类型（翻译：它可以工作）。

#### SQL 表达式类型 - 示例

类型行为简要介绍。评论指示在 [vscode](https://code.visualstudio.com/) 中悬停在代码上会看到什么（或者在使用 [reveal_type()](https://mypy.readthedocs.io/en/latest/common_issues.html?highlight=reveal_type#reveal-type) 助手时，大致会显示什么类型工具）：

+   分配给 SQL 表达式的简单 Python 类型

    ```py
    # (variable) str_col: ColumnClause[str]
    str_col = column("a", String)

    # (variable) int_col: ColumnClause[int]
    int_col = column("a", Integer)

    # (variable) expr1: ColumnElement[str]
    expr1 = str_col + "x"

    # (variable) expr2: ColumnElement[int]
    expr2 = int_col + 10

    # (variable) expr3: ColumnElement[bool]
    expr3 = int_col == 15
    ```

+   分配给 `select()` 构造的单个 SQL 表达式，以及任何返回行的构造，包括返回行的 DML，例如带有 `Insert` 的 `Insert.returning()`，都被打包成一个 `Tuple[]` 类型，其中保留了每个元素的 Python 类型。

    ```py
    # (variable) stmt: Select[Tuple[str, int]]
    stmt = select(str_col, int_col)

    # (variable) stmt: ReturningInsert[Tuple[str, int]]
    ins_stmt = insert(table("t")).returning(str_col, int_col)
    ```

+   从任何返回行构造的`Tuple[]`类型，在调用`.execute()`方法时，传递到`Result`和`Row`。为了将`Row`对象解包为元组，`Row.tuple()`或`Row.t`访问器基本上将`Row`强制转换为相应的`Tuple[]`（尽管在运行时仍然是相同的`Row`对象）。

    ```py
    with engine.connect() as conn:
        # (variable) stmt: Select[Tuple[str, int]]
        stmt = select(str_col, int_col)

        # (variable) result: Result[Tuple[str, int]]
        result = conn.execute(stmt)

        # (variable) row: Row[Tuple[str, int]] | None
        row = result.first()

        if row is not None:
            # for typed tuple unpacking or indexed access,
            # use row.tuple() or row.t  (this is the small typing-oriented accessor)
            strval, intval = row.t

            # (variable) strval: str
            strval

            # (variable) intval: int
            intval
    ```

+   对于单列语句的标量值，像`Connection.scalar()`、`Result.scalars()`等方法会做正确的事情。

    ```py
    # (variable) data: Sequence[str]
    data = connection.execute(select(str_col)).scalars().all()
    ```

+   上述对于返回行构造的支持与 ORM 映射类一起运作最好，因为映射类可以为其成员列出具体类型。下面的示例设置了一个类，使用了新的类型感知语法，在下一节中描述：

    ```py
    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column

    class Base(DeclarativeBase):
        pass

    class User(Base):
        __tablename__ = "user_account"

        id: Mapped[int] = mapped_column(primary_key=True)
        name: Mapped[str]
        addresses: Mapped[List["Address"]] = relationship()

    class Address(Base):
        __tablename__ = "address"

        id: Mapped[int] = mapped_column(primary_key=True)
        email_address: Mapped[str]
        user_id = mapped_column(ForeignKey("user_account.id"))
    ```

    使用上述映射，属性从语句到结果集都有类型并自我表达：

    ```py
    with Session(engine) as session:
        # (variable) stmt: Select[Tuple[int, str]]
        stmt_1 = select(User.id, User.name)

        # (variable) result_1: Result[Tuple[int, str]]
        result_1 = session.execute(stmt_1)

        # (variable) intval: int
        # (variable) strval: str
        intval, strval = result_1.one().t
    ```

    映射类本身也是类型，并且表现得相同，例如针对两个映射类进行的 SELECT：

    ```py
    with Session(engine) as session:
        # (variable) stmt: Select[Tuple[User, Address]]
        stmt_2 = select(User, Address).join_from(User, Address)

        # (variable) result_2: Result[Tuple[User, Address]]
        result_2 = session.execute(stmt_2)

        # (variable) user_obj: User
        # (variable) address_obj: Address
        user_obj, address_obj = result_2.one().t
    ```

    在选择映射类时，像`aliased`这样的构造也可以工作，保持原始映射类的列级属性以及语句预期的返回类型：

    ```py
    with Session(engine) as session:
        # this is in fact an Annotated type, but typing tools don't
        # generally display this

        # (variable) u1: Type[User]
        u1 = aliased(User)

        # (variable) stmt: Select[Tuple[User, User, str]]
        stmt = select(User, u1, User.name).filter(User.id == 5)

        # (variable) result: Result[Tuple[User, User, str]]
        result = session.execute(stmt)
    ```

+   Core Table 还没有一个良好的方法来在通过`Table.c`访问它们时维护`Column`对象的类型。

    由于`Table`设置为类的一个实例，而`Table.c`访问器通常通过名称动态访问`Column`对象，因此尚未建立针对此的已知类型方法；需要一些替代语法。

+   ORM 类、标量等都很好用。

    典型的 ORM 类选择用例，作为标量或元组，所有工作都可以，无论是 2.0 还是 1.x 风格的查询，都可以得到精确的类型，要么是自身，要么包含在适当的容器内，如 `Sequence[]`、`List[]` 或 `Iterator[]`：

    ```py
    # (variable) users1: Sequence[User]
    users1 = session.scalars(select(User)).all()

    # (variable) user: User
    user = session.query(User).one()

    # (variable) user_iter: Iterator[User]
    user_iter = iter(session.scalars(select(User)))
    ```

+   旧版 `Query` 也获得了元组类型。

    对于 `Query` 的类型支持远远超出了 [sqlalchemy-stubs](https://github.com/dropbox/sqlalchemy-stubs) 或 [sqlalchemy2-stubs](https://github.com/sqlalchemy/sqlalchemy2-stubs) 提供的范围，其中标量对象和元组类型的 `Query` 对象在大多数情况下都会保留结果级别的类型：

    ```py
    # (variable) q1: RowReturningQuery[Tuple[int, str]]
    q1 = session.query(User.id, User.name)

    # (variable) rows: List[Row[Tuple[int, str]]]
    rows = q1.all()

    # (variable) q2: Query[User]
    q2 = session.query(User)

    # (variable) users: List[User]
    users = q2.all()
    ```

#### 陷阱 - 所有存根必须被卸载

类型支持的一个关键注意事项是，**所有 SQLAlchemy 存根包都必须被卸载** 才能使类型化工作。当针对 Python 虚拟环境运行 [mypy](https://mypy.readthedocs.io/en/stable/) 时，只需卸载这些包即可。但是，目前 typeshed 中也包含 SQLAlchemy 存根包，typeshed 本身被捆绑到一些类型工具中，如 [Pylance](https://github.com/microsoft/pylance-release)，因此在某些情况下，可能需要定位这些包的文件并删除它们，以确保新的类型化能够正确工作。

一旦 SQLAlchemy 2.0 发布为最终状态，typeshed 将从其自己的存根源中删除 SQLAlchemy。

### ORM 声明性模型

SQLAlchemy 1.4 引入了第一个使用 [sqlalchemy2-stubs](https://github.com/sqlalchemy/sqlalchemy2-stubs) 和 Mypy Plugin 的 SQLAlchemy 本机 ORM 类型支持的方法。在 SQLAlchemy 2.0 中，Mypy 插件 **仍然可用，并已更新以与 SQLAlchemy 2.0 的类型系统一起工作**。但是，现在应该将其视为**已弃用**，因为应用程序现在有了采用新的不使用插件或存根的类型支持的简单路径。

#### 概览

新系统的基本方法是，当使用完全声明性模型（即不使用混合声明性或命令式配置，这些配置保持不变）时，映射列声明首先通过检查每个属性声明左侧的类型注释（如果存在）在运行时派生。左手类型注释应包含在`Mapped`泛型类型中，否则不认为属性是映射属性。然后，属性声明可以引用右手边的`mapped_column()`构造，该构造用于提供有关要生成和映射的`Column`的附加 Core 级架构信息。如果左侧存在`Mapped`注释，则此右侧声明是可选的；如果左侧没有注释，则`mapped_column()`可用作`Column`指令的精确替代，在这种情况下，它将提供更准确（但不精确）的属性类型行为，即使没有注释也是如此。

这种方法的灵感来自于 Python 的 [dataclasses](https://docs.python.org/3/library/dataclasses.html) 方法，该方法从左侧开始注释，然后允许在右侧进行可选的 `dataclasses.field()` 规范；与 dataclasses 方法的关键区别在于 SQLAlchemy 的方法严格地 **选择加入**，其中使用 `Column` 而没有任何类型注释的现有映射继续按照其原来的方式工作，而且 `mapped_column()` 构造可以直接替代 `Column` 而不需要任何显式类型注释。只有在需要精确的属性级 Python 类型时才需要使用 `Mapped` 进行显式注释。这些注释可以根据需要在每个属性上使用，对于那些具体类型有帮助的属性；使用 `mapped_column()` 的非注释属性将在实例级别被标记为 `Any`。

#### 迁移现有映射

迁移到新的 ORM 方法开始时更加冗长，但随着可用的新功能的充分使用，变得比以前更加简洁。以下步骤详细说明了一个典型的过渡，然后继续说明了一些更多的选项。

##### 第一步 - `declarative_base()` 被 `DeclarativeBase` 取代。

在 Python 类型中观察到的一个限制是似乎没有能力从函数动态生成一个类，然后这个类被理解为新类的基础。为了解决这个问题而不使用插件，通常对 `declarative_base()` 的调用可以被替换为使用 `DeclarativeBase` 类，该类产生与通常相同的 `Base` 对象，但是类型工具理解它：

```py
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    pass
```

##### 第二步 - 用 `mapped_column()` 替换 Declarative 中的 `Column`

`mapped_column()`是一个 ORM 类型感知的构造，可以直接替换为`Column`的使用。给定一个 1.x 风格的映射如下：

```py
from sqlalchemy import Column
from sqlalchemy.orm import relationship
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "user_account"

    id = Column(Integer, primary_key=True)
    name = Column(String(30), nullable=False)
    fullname = Column(String)
    addresses = relationship("Address", back_populates="user")

class Address(Base):
    __tablename__ = "address"

    id = Column(Integer, primary_key=True)
    email_address = Column(String, nullable=False)
    user_id = Column(ForeignKey("user_account.id"), nullable=False)
    user = relationship("User", back_populates="addresses")
```

我们用`mapped_column()`替换`Column`; 不需要更改任何参数：

```py
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "user_account"

    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(30), nullable=False)
    fullname = mapped_column(String)
    addresses = relationship("Address", back_populates="user")

class Address(Base):
    __tablename__ = "address"

    id = mapped_column(Integer, primary_key=True)
    email_address = mapped_column(String, nullable=False)
    user_id = mapped_column(ForeignKey("user_account.id"), nullable=False)
    user = relationship("User", back_populates="addresses")
```

上面的单独列 **尚未使用 Python 类型进行类型化**，而是被类型化为`Mapped[Any]`；这是因为我们可以声明任何列为`Optional`或者不声明，而且没有办法在我们明确地对其进行类型化时有一个“猜测”。

但是，在这一步，我们上面的映射已经为所有属性设置了适当的描述符类型，并且可以用于查询以及实例级别的操作，所有这些操作都将以**不使用插件的 mypy –strict 模式**通过。

##### 第三步 - 使用`Mapped`根据需要应用确切的 Python 类型。

这可以用于所有需要确切类型的属性；那些可以留作`Any`的属性可以跳过。为了上下文，我们还说明了在一个`relationship()`中应用确切类型时如何使用`Mapped`。这个中间步骤中的映射会更冗长，但是熟练之后，这一步可以与后续步骤结合起来更直接地更新映射：

```py
from typing import List
from typing import Optional
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "user_account"

    id: Mapped[int] = mapped_column(Integer, primary_key=True)
    name: Mapped[str] = mapped_column(String(30), nullable=False)
    fullname: Mapped[Optional[str]] = mapped_column(String)
    addresses: Mapped[List["Address"]] = relationship("Address", back_populates="user")

class Address(Base):
    __tablename__ = "address"

    id: Mapped[int] = mapped_column(Integer, primary_key=True)
    email_address: Mapped[str] = mapped_column(String, nullable=False)
    user_id: Mapped[int] = mapped_column(ForeignKey("user_account.id"), nullable=False)
    user: Mapped["User"] = relationship("User", back_populates="addresses")
```

在这一点上，我们的 ORM 映射是完全类型化的，并将生成精确类型的`select()`、`Query`和`Result`构造。我们现在可以继续简化映射声明中的冗余部分。

##### 第四步 - 删除不再需要的`mapped_column()`指令

所有的`nullable`参数都可以使用`Optional[]`隐含；在没有`Optional[]`的情况下，`nullable`默认为`False`。所有没有参数的 SQL 类型，如`Integer`和`String`，可以单独表示为 Python 注释。不带参数的`mapped_column()`指令可以完全移除。`relationship()`现在从左侧注释派生其类，还支持前向引用（正如`relationship()`已经支持了十年的字符串型前向引用一样 ;) ）：

```py
from typing import List
from typing import Optional
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "user_account"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(30))
    fullname: Mapped[Optional[str]]
    addresses: Mapped[List["Address"]] = relationship(back_populates="user")

class Address(Base):
    __tablename__ = "address"

    id: Mapped[int] = mapped_column(primary_key=True)
    email_address: Mapped[str]
    user_id: Mapped[int] = mapped_column(ForeignKey("user_account.id"))
    user: Mapped["User"] = relationship(back_populates="addresses")
```

##### 第五步 - 利用 pep-593 `Annotated` 将常见指令打包到类型中

这是一项根本性的新功能，提供了一种替代或补充方法，用于声明式混合作为提供类型定向配置的手段，并且在大多数情况下也替代了`declared_attr`装饰函数的需要。

首先，声明式映射允许将 Python 类型映射到 SQL 类型，例如将`str`映射到`String`，通过使用`registry.type_annotation_map`进行自定义。使用[**PEP 593**](https://peps.python.org/pep-0593/) `Annotated` 允许我们创建特定 Python 类型的变体，以便可以使用相同的类型，例如`str`，每个类型提供`String`的变体，如下所示，其中使用名为`str50`的`Annotated` `str` 将指示`String(50)`：

```py
from typing_extensions import Annotated
from sqlalchemy.orm import DeclarativeBase

str50 = Annotated[str, 50]

# declarative base with a type-level override, using a type that is
# expected to be used in multiple places
class Base(DeclarativeBase):
    type_annotation_map = {
        str50: String(50),
    }
```

其次，如果使用`Annotated[]`，声明式将从左侧类型中提取完整的`mapped_column()`定义，方法是将`mapped_column()`结构作为`Annotated[]`结构的任何参数传递（感谢[@adriangb01](https://twitter.com/adriangb01/status/1532841383647657988)说明这个想法）。在未来的版本中，这项功能可能会扩展到包括`relationship()`、`composite()`和其他结构，但目前仅限于`mapped_column()`。下面的示例除了我们的`str50`示例外，还添加了额外的`Annotated`类型，以说明此功能：

```py
from typing_extensions import Annotated
from typing import List
from typing import Optional
from sqlalchemy import ForeignKey
from sqlalchemy import String
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship

# declarative base from previous example
str50 = Annotated[str, 50]

class Base(DeclarativeBase):
    type_annotation_map = {
        str50: String(50),
    }

# set up mapped_column() overrides, using whole column styles that are
# expected to be used in multiple places
intpk = Annotated[int, mapped_column(primary_key=True)]
user_fk = Annotated[int, mapped_column(ForeignKey("user_account.id"))]

class User(Base):
    __tablename__ = "user_account"

    id: Mapped[intpk]
    name: Mapped[str50]
    fullname: Mapped[Optional[str]]
    addresses: Mapped[List["Address"]] = relationship(back_populates="user")

class Address(Base):
    __tablename__ = "address"

    id: Mapped[intpk]
    email_address: Mapped[str50]
    user_id: Mapped[user_fk]
    user: Mapped["User"] = relationship(back_populates="addresses")
```

上面，与`Mapped[str50]`、`Mapped[intpk]`或`Mapped[user_fk]`映射的列从`registry.type_annotation_map`以及直接使用`Annotated`构造从中提取，以便重用预先建立的类型和列配置。

##### 可选步骤 - 将映射类转换为[dataclasses](https://docs.python.org/3/library/dataclasses.html)

我们可以将映射类转换为[dataclasses](https://docs.python.org/3/library/dataclasses.html)，其中一个关键优势是我们可以构建一个严格类型化的`__init__()`方法，具有明确的位置参数、关键字参数和默认参数，更不用说我们可以免费获得`__str__()`和`__repr__()`等方法。接下来的部分作为 ORM 模型映射的数据类的本机支持进一步说明了以上模型的转换。

##### 从第 3 步开始支持类型标注

通过上述示例，从“第 3 步”开始的任何示例都将包括模型的属性已进行类型标注，并将填充到`select()`、`Query`和`Row`对象中：

```py
# (variable) stmt: Select[Tuple[int, str]]
stmt = select(User.id, User.name)

with Session(e) as sess:
    for row in sess.execute(stmt):
        # (variable) row: Row[Tuple[int, str]]
        print(row)

    # (variable) users: Sequence[User]
    users = sess.scalars(select(User)).all()

    # (variable) users_legacy: List[User]
    users_legacy = sess.query(User).all()
```

另请参阅

具有 mapped_column()的声明性表 - 更新的声明性文档，用于声明性生成和映射`Table`列。  ### 使用传统的 Mypy 类型化模型

使用 Mypy 插件的 SQLAlchemy 应用程序，其中明确注释不使用`Mapped`在其注释中的，当使用诸如`relationship()`之类的构造时，将根据新系统标记为错误。

部分迁移到 2.0 步骤六 - 为显式类型的 ORM 模型添加 __allow_unmapped__ 说明了如何为使用显式注释的遗留 ORM 模型临时禁用这些错误的触发。

另请参阅

迁移到 2.0 步骤六 - 为显式类型的 ORM 模型添加 __allow_unmapped__  ### 作为 ORM 模型映射的数据类的本机支持

在上面介绍的新 ORM 声明式特性中，引入了新的`mapped_column()`构造，以及可选使用[**PEP 593**](https://peps.python.org/pep-0593/) `Annotated`进行类型中心化映射的示例。我们可以通过将其与 Python 的[dataclasses](https://docs.python.org/3/library/dataclasses.html)集成，进一步完善这种映射。这一新特性是通过[**PEP 681**](https://peps.python.org/pep-0681/)实现的，允许类型检查器识别与`dataclass`兼容的类，或者完全是`dataclass`，但是通过其他 API 声明的类。

使用`dataclasses`特性，映射类获得了一个`__init__()`方法，支持位置参数以及可选关键字参数的自定义默认值。如前所述，`dataclasses`还会生成许多有用的方法，比如`__str__()`、`__eq__()`。`dataclasses`的序列化方法，比如[dataclasses.asdict()](https://docs.python.org/3/library/dataclasses.html#dataclasses.asdict)和[dataclasses.astuple()](https://docs.python.org/3/library/dataclasses.html#dataclasses.astuple)也可以使用，但目前不支持自引用结构，这使得它们对于具有双向关系的映射来说不太适用。

SQLAlchemy 当前的集成方法将用户定义的类转换为一个**真实的 dataclass**，以提供运行时功能；该特性利用了 SQLAlchemy 1.4 中引入的现有 dataclass 特性，在 Python Dataclasses, attrs Supported w/ Declarative, Imperative Mappings 中生成一个等效的运行时映射，具有完全集成的配置样式，比以前的方法更正确地类型化。

为了支持符合 [**PEP 681**](https://peps.python.org/pep-0681/) 的数据类，ORM 构造如 `mapped_column()` 和 `relationship()` 现在接受额外的 [**PEP 681**](https://peps.python.org/pep-0681/) 参数 `init`、`default` 和 `default_factory`，这些参数将传递到数据类创建过程中。这些参数目前必须以右侧的显式指令的形式存在，就像它们在 `dataclasses.field()` 中使用时一样；它们目前不能局部存在于左侧的 `Annotated` 构造中。为了支持方便地使用 `Annotated` 并仍然支持数据类配置，`mapped_column()` 可以将一组最小的右侧参数与位于左侧的 `Annotated` 构造内的现有 `mapped_column()` 构造合并，以便保持大部分简洁性，如下所示。

为了启用使用类继承的数据类，我们利用了 `MappedAsDataclass` mixin，可以直接在每个类上使用，也可以在 `Base` 类上使用，如下所示，在这里我们进一步修改了来自 “Step 5” 的示例映射 ORM 声明性模型。

```py
from typing_extensions import Annotated
from typing import List
from typing import Optional
from sqlalchemy import ForeignKey
from sqlalchemy import String
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import MappedAsDataclass
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship

class Base(MappedAsDataclass, DeclarativeBase):
  """subclasses will be converted to dataclasses"""

intpk = Annotated[int, mapped_column(primary_key=True)]
str30 = Annotated[str, mapped_column(String(30))]
user_fk = Annotated[int, mapped_column(ForeignKey("user_account.id"))]

class User(Base):
    __tablename__ = "user_account"

    id: Mapped[intpk] = mapped_column(init=False)
    name: Mapped[str30]
    fullname: Mapped[Optional[str]] = mapped_column(default=None)
    addresses: Mapped[List["Address"]] = relationship(
        back_populates="user", default_factory=list
    )

class Address(Base):
    __tablename__ = "address"

    id: Mapped[intpk] = mapped_column(init=False)
    email_address: Mapped[str]
    user_id: Mapped[user_fk] = mapped_column(init=False)
    user: Mapped["User"] = relationship(back_populates="addresses", default=None)
```

上述映射已经直接在每个映射类上使用 `@dataclasses.dataclass` 装饰器设置，同时在设置声明性映射时内部设置了每个 `dataclasses.field()` 指令，如所示。可以使用按位置配置的 `User` / `Address` 结构创建：

```py
>>> u1 = User("username", fullname="full name", addresses=[Address("email@address")])
>>> u1
User(id=None, name='username', fullname='full name', addresses=[Address(id=None, email_address='email@address', user_id=None, user=...)])
```

另请参阅

声明式数据类映射  ## 除了 MySQL 外，所有后端现在都实现了优化的 ORM 批量插入

在 1.4 系列中引入的戏剧性性能改进，并在 ORM Batch inserts with psycopg2 now batch statements with RETURNING in most cases 中描述，现已推广到所有支持 RETURNING 的包含后端，除了 MySQL 之外的所有后端：SQLite，MariaDB，PostgreSQL（所有驱动程序）和 Oracle； SQL Server 具有支持，但在版本 2.0.9 中暂时禁用[[1]](#id2)。虽然原始功能对于 psycopg2 驱动程序最为关键，否则在使用`cursor.executemany()`时会有严重的性能问题，但该变更对于其他 PostgreSQL 驱动程序如 asyncpg 同样关键，因为在使用 RETURNING 时，单语句 INSERT 语句仍然不可接受地慢，以及在使用 SQL Server 时，似乎无论是否使用 RETURNING，INSERT 语句的 executemany 速度都非常慢。

新功能的性能几乎在各个方面都提供了一个数量级的性能增加，当 INSERT ORM 对象时，这些对象没有预先分配的主键值，在下表中有所指示，大多数情况下特定于使用 RETURNING，而这通常不支持 executemany()。

psycopg2 的“快速执行助手”方法包括将一个带有单个参数集的 INSERT..RETURNING 语句转换为一个语句，该语句插入了许多参数集，使用多个“VALUES…”子句，以便一次容纳许多参数集。然后，参数集通常被分批为 1000 个或类似的组，以便没有单个 INSERT 语句过大，并且 INSERT 语句然后为每批参数调用，而不是为每个单独的参数集调用。主键值和服务器默认值由 RETURNING 返回，这仍然有效，因为每个语句执行都是使用`cursor.execute()`调用的，而不是`cursor.executemany()`。

这允许在一个语句中插入许多行，同时还能返回新生成的主键值以及 SQL 和服务器默认值。 SQLAlchemy 在历史上一直需要为每个参数集调用一个语句，因为它依赖于 Python DBAPI 功能，如`cursor.lastrowid`，这些功能不支持多行。

由于大多数数据库现在都提供 RETURNING（明显的例外是 MySQL，鉴于 MariaDB 支持它），新的更改将 psycopg2 的“快速执行助手”方法推广到所有支持 RETURNING 的方言，现在包括 SQlite 和 MariaDB，对于支持“executemany 加 RETURNING”的其他方法不可能的方言，包括 SQLite、MariaDB 和所有 PG 驱动程序。用于 Oracle 的 cx_Oracle 和 oracledb 驱动程序本地支持使用 executemany 返回，这也已经实现以提供等效的性能改进。由于 SQLite 和 MariaDB 现在支持 RETURNING，ORM 对`cursor.lastrowid`的使用几乎已经成为过去，只有 MySQL 仍然依赖它。

对于不使用 RETURNING 的 INSERT 语句，大多数后端使用传统的 executemany()行为，当前的例外是 psycopg2，它的 executemany()性能总体上非常慢，并且仍然通过“insertmanyvalues”方法得到改进。

### 基准测试

SQLAlchemy 在`examples/`目录中包含一个性能套件，我们可以利用`bulk_insert`套件以不同的方式使用 Core 和 ORM 来对许多行进行 INSERT 的基准测试。

对于以下测试，我们正在插入**100,000 个对象**，在所有情况下，我们实际上在内存中有 100,000 个真实的 Python ORM 对象，要么是预先创建的，要么是动态生成的。除了 SQLite 之外的所有数据库都通过本地网络连接运行，而不是 localhost；这导致“较慢”的结果非常慢。

通过此功能改进的操作包括：

+   使用`Session.add()`和`Session.add_all()`将对象添加到会话中的工作单元刷新。

+   新的 ORM 批量插入语句功能，改进了 SQLAlchemy 1.4 中首次引入的实验性版本。

+   在 Bulk Operations 中描述的`Session`“批量”操作，被上述 ORM 批量插入功能取代。

为了了解操作的规模，以下是使用`test_flush_no_pk`性能套件进行的性能测量，该套件历史上代表了 SQLAlchemy 的最坏情况 INSERT 性能任务，其中需要 INSERT 没有主键值的对象，然后必须获取新生成的主键值，以便这些对象可以用于后续的 flush 操作，比如在关系中建立关系，刷新联合继承模型等：

```py
@Profiler.profile
def test_flush_no_pk(n):
  """INSERT statements via the ORM (batched with RETURNING if available),
 fetching generated row id"""
    session = Session(bind=engine)
    for chunk in range(0, n, 1000):
        session.add_all(
            [
                Customer(
                    name="customer name %d" % i,
                    description="customer description %d" % i,
                )
                for i in range(chunk, chunk + 1000)
            ]
        )
        session.flush()
    session.commit()
```

可以从任何 SQLAlchemy 源代码树中运行此测试：

```py
python -m examples.performance.bulk_inserts --test test_flush_no_pk
```

下表总结了最新的 1.4 系列 SQLAlchemy 与 2.0 在运行相同测试时的性能测量：

| 驱动程序 | SQLA 1.4 时间（秒） | SQLA 2.0 时间（秒） |
| --- | --- | --- |
| sqlite+pysqlite2 (memory) | 6.204843 | 3.554856 |
| postgresql+asyncpg (network) | 88.292285 | 4.561492 |
| postgresql+psycopg (network) | N/A (psycopg3) | 4.861368 |
| mssql+pyodbc (network) | 158.396667 | 4.825139 |
| oracle+cx_Oracle (network) | 92.603953 | 4.809520 |
| mariadb+mysqldb (network) | 71.705197 | 4.075377 |

注意

另外两个驱动程序在性能上没有变化；psycopg2 驱动程序，在 SQLAlchemy 1.4 中已经实现了快速 executemany，以及 MySQL，继续不提供 RETURNING 支持：

| 驱动程序 | SQLA 1.4 时间（秒） | SQLA 2.0 时间（秒） |
| --- | --- | --- |
| postgresql+psycopg2 (network) | 4.704876 | 4.699883 |
| mysql+mysqldb (network) | 77.281997 | 76.132995 |

### 变更摘要

下面的项目列出了 2.0 中为使所有驱动程序达到这种状态而进行的各个更改：

+   SQLite 实现了 RETURNING - [#6195](https://www.sqlalchemy.org/trac/ticket/6195)

+   MariaDB 实现了 RETURNING - [#7011](https://www.sqlalchemy.org/trac/ticket/7011)

+   修复 Oracle 的多行 RETURNING - [#6245](https://www.sqlalchemy.org/trac/ticket/6245)

+   使 insert() executemany()支持尽可能多的方言，通常使用 VALUES() - [#6047](https://www.sqlalchemy.org/trac/ticket/6047)

+   当 RETURNING 与 executemany 一起用于不支持的后端时发出警告（当前没有 RETURNING 后端有此限制） - [#7907](https://www.sqlalchemy.org/trac/ticket/7907)

+   ORM `Mapper.eager_defaults` 参数现在默认为新设置`"auto"`，当使用的后端支持“insertmanyvalues”时，将自动为 INSERT 语句启用“急切默认值”。请参阅获取服务器生成的默认值以获取文档。

另请参阅

“插入多个值”行为适用于 INSERT 语句 - 新功能的文档和背景以及如何配置它的说明  ## 启用 ORM 的插入、更新和删除语句，带有 ORM RETURNING

SQLAlchemy 1.4 将传统的`Query`对象的特性移植到 2.0 风格的执行中，这意味着`Select`构造可以传递给`Session.execute()`以提供 ORM 结果。还添加了对`Update`和`Delete`的支持，可以传递给`Session.execute()`，以便它们可以提供`Query.update()`和`Query.delete()`的实现。

主要缺失的元素是对`Insert`构造的支持。1.4 文档通过一些关于在 ORM 上下文中使用`Select.from_statement()`来集成 RETURNING 的“插入”和“upserts”的示例来解决这个问题。2.0 现在通过将`Insert`直接支持作为`Session.bulk_insert_mappings()`方法的增强版本，以及对所有 DML 结构的完整 ORM RETURNING 支持来完全弥补这一差距。

### 带有 RETURNING 的批量插入

`Insert`可以传递给`Session.execute()`，可以带有或不带有`Insert.returning()`，当与一个单独的参数列表一起传递时，将调用与以前由`Session.bulk_insert_mappings()`实现的相同过程，同时还添加了额外的增强功能。这将通过利用新的快速插入多行功能来优化行的批处理，同时还支持异构参数集和多表映射，如联合表继承：

```py
>>> users = session.scalars(
...     insert(User).returning(User),
...     [
...         {"name": "spongebob", "fullname": "Spongebob Squarepants"},
...         {"name": "sandy", "fullname": "Sandy Cheeks"},
...         {"name": "patrick", "fullname": "Patrick Star"},
...         {"name": "squidward", "fullname": "Squidward Tentacles"},
...         {"name": "ehkrabs", "fullname": "Eugene H. Krabs"},
...     ],
... )
>>> print(users.all())
[User(name='spongebob', fullname='Spongebob Squarepants'),
 User(name='sandy', fullname='Sandy Cheeks'),
 User(name='patrick', fullname='Patrick Star'),
 User(name='squidward', fullname='Squidward Tentacles'),
 User(name='ehkrabs', fullname='Eugene H. Krabs')]
```

RETURNING 支持所有这些用例，其中 ORM 将从多个语句调用构造完整的结果集。

另请参见

ORM 批量 INSERT 语句

### 批量 UPDATE

与`Insert`类似，将`Update`构造与包含主键值的参数列表一起传递给`Session.execute()`将调用与以前由`Session.bulk_update_mappings()`方法支持的相同过程。但是，此功能不支持 RETURNING，因为它使用 SQL UPDATE 语句，该语句使用 DBAPI executemany��行调用：

```py
>>> from sqlalchemy import update
>>> session.execute(
...     update(User),
...     [
...         {"id": 1, "fullname": "Spongebob Squarepants"},
...         {"id": 3, "fullname": "Patrick Star"},
...     ],
... )
```

另请参见

按主键进行 ORM 批量 UPDATE

### INSERT / upsert … VALUES … RETURNING

当使用`Insert`与`Insert.values()`时，参数集合可能包含 SQL 表达式。此外，还支持 SQLite、PostgreSQL 和 MariaDB 等数据库的 upsert 变体。这些语句现在可以包括带有列表达式或完整 ORM 实体的`Insert.returning()`子句：

```py
>>> from sqlalchemy.dialects.sqlite import insert as sqlite_upsert
>>> stmt = sqlite_upsert(User).values(
...     [
...         {"name": "spongebob", "fullname": "Spongebob Squarepants"},
...         {"name": "sandy", "fullname": "Sandy Cheeks"},
...         {"name": "patrick", "fullname": "Patrick Star"},
...         {"name": "squidward", "fullname": "Squidward Tentacles"},
...         {"name": "ehkrabs", "fullname": "Eugene H. Krabs"},
...     ]
... )
>>> stmt = stmt.on_conflict_do_update(
...     index_elements=[User.name], set_=dict(fullname=stmt.excluded.fullname)
... )
>>> result = session.scalars(stmt.returning(User))
>>> print(result.all())
[User(name='spongebob', fullname='Spongebob Squarepants'),
User(name='sandy', fullname='Sandy Cheeks'),
User(name='patrick', fullname='Patrick Star'),
User(name='squidward', fullname='Squidward Tentacles'),
User(name='ehkrabs', fullname='Eugene H. Krabs')]
```

另请参见

ORM 批量插入带有每行 SQL 表达式

ORM “upsert”语句

### 带有 WHERE … RETURNING 的 ORM UPDATE / DELETE

SQLAlchemy 1.4 还对 RETURNING 功能提供了一些支持，可与`update()`和`delete()`构造一起使用，当与`Session.execute()`一起使用时。此支持现已升级为完全本地化，包括`fetch`同步策略是否存在 RETURNING 的明确使用：

```py
>>> from sqlalchemy import update
>>> stmt = (
...     update(User)
...     .where(User.name == "squidward")
...     .values(name="spongebob")
...     .returning(User)
... )
>>> result = session.scalars(stmt, execution_options={"synchronize_session": "fetch"})
>>> print(result.all())
```

另请参见

带有自定义 WHERE 条件的 ORM UPDATE 和 DELETE

使用 RETURNING 进行 UPDATE/DELETE 和自定义 WHERE 条件

### 改进了 ORM UPDATE / DELETE 的`synchronize_session`行为

synchronize_session 的默认策略现在是一个新值`"auto"`。此策略将尝试使用`"evaluate"`策略，然后自动回退到`"fetch"`策略。对于除了 MySQL / MariaDB 之外的所有后端，`"fetch"`使用 RETURNING 在同一语句中获取已更新/删除的主键标识符，因此通常比以前版本更有效（在 1.4 版本中，RETURNING 仅适用于 PostgreSQL、SQL Server）。

亦见

选择同步策略

### 变更摘要

新的 ORM DML 带有 RETURNING 特性的已列出的票证：

+   将 ORM 级别的`insert()`转换为在 ORM 上下文中解释`values()` - [#7864](https://www.sqlalchemy.org/trac/ticket/7864)

+   评估`dml.returning(Entity)`的可行性，以提供 ORM 表达式，自动应用`select().from_statement`等效 - [#7865](https://www.sqlalchemy.org/trac/ticket/7865)

+   给定 ORM 插入，尝试携带批量方法，有关继承 - [#8360](https://www.sqlalchemy.org/trac/ticket/8360)  ## 新的“仅写入”关系策略取代了“动态”

“懒加载”策略`lazy="dynamic"`已经过时，因为它被硬编码为使用传统的`Query`。这个加载策略既不兼容 asyncio，而且还有许多行为隐式迭代其内容，这违背了“动态”关系的原始目的，即用于非常大的集合，不应随时隐式加载到内存中。

“动态”策略现已由新策略`lazy="write_only"`取代。可以使用`relationship.lazy`参数的配置`relationship()`来实现“仅写入”，或者在使用类型注释映射时，指示`WriteOnlyMapped`注解作为映射样式：

```py
from sqlalchemy.orm import WriteOnlyMapped

class Base(DeclarativeBase):
    pass

class Account(Base):
    __tablename__ = "account"
    id: Mapped[int] = mapped_column(primary_key=True)
    identifier: Mapped[str]
    account_transactions: WriteOnlyMapped["AccountTransaction"] = relationship(
        cascade="all, delete-orphan",
        passive_deletes=True,
        order_by="AccountTransaction.timestamp",
    )

class AccountTransaction(Base):
    __tablename__ = "account_transaction"
    id: Mapped[int] = mapped_column(primary_key=True)
    account_id: Mapped[int] = mapped_column(
        ForeignKey("account.id", ondelete="cascade")
    )
    description: Mapped[str]
    amount: Mapped[Decimal]
    timestamp: Mapped[datetime] = mapped_column(default=func.now())
```

写入唯一映射集合类似于`lazy="dynamic"`，因为集合可以提前分配，并且还具有诸如`WriteOnlyCollection.add()`和`WriteOnlyCollection.remove()`等方法，以逐个项目的方式修改集合：

```py
new_account = Account(
    identifier="account_01",
    account_transactions=[
        AccountTransaction(description="initial deposit", amount=Decimal("500.00")),
        AccountTransaction(description="transfer", amount=Decimal("1000.00")),
        AccountTransaction(description="withdrawal", amount=Decimal("-29.50")),
    ],
)

new_account.account_transactions.add(
    AccountTransaction(description="transfer", amount=Decimal("2000.00"))
)
```

更大的区别在于数据库加载方面，集合无法直接从数据库加载对象；而是使用诸如 `WriteOnlyCollection.select()` 等 SQL 构造方法来生成诸如 `Select` 的 SQL 构造，然后使用 2.0 风格 以显式方式加载所需对象：

```py
account_transactions = session.scalars(
    existing_account.account_transactions.select()
    .where(AccountTransaction.amount < 0)
    .limit(10)
).all()
```

`WriteOnlyCollection` 也与新的 ORM 批量 DML 特性集成，包括支持带有 WHERE 条件的批量 INSERT 和 UPDATE/DELETE，全部支持 RETURNING。详见完整文档 Write Only Relationships。

参见

Write Only Relationships

### 为动态关系添加了新的 pep-484 / 类型注释映射支持

尽管“动态”关系在 2.0 中是遗留的，但由于这些模式预计具有很长的寿命，类型注释映射 现在也为“动态”关系添加了支持，方式与新的 `lazy="write_only"` 方法相同，使用 `DynamicMapped` 注释：

```py
from sqlalchemy.orm import DynamicMapped

class Base(DeclarativeBase):
    pass

class Account(Base):
    __tablename__ = "account"
    id: Mapped[int] = mapped_column(primary_key=True)
    identifier: Mapped[str]
    account_transactions: DynamicMapped["AccountTransaction"] = relationship(
        cascade="all, delete-orphan",
        passive_deletes=True,
        order_by="AccountTransaction.timestamp",
    )

class AccountTransaction(Base):
    __tablename__ = "account_transaction"
    id: Mapped[int] = mapped_column(primary_key=True)
    account_id: Mapped[int] = mapped_column(
        ForeignKey("account.id", ondelete="cascade")
    )
    description: Mapped[str]
    amount: Mapped[Decimal]
    timestamp: Mapped[datetime] = mapped_column(default=func.now())
```

上述映射将提供一个类型为 `AppenderQuery` 集合类型的 `Account.account_transactions` 集合，包括其元素类型，例如 `AppenderQuery[AccountTransaction]`。然后允许迭代和查询产生类型为 `AccountTransaction` 的对象。

参见

动态关系加载器

[#7123](https://www.sqlalchemy.org/trac/ticket/7123)  ## 安装现在完全启用了 pep-517

源发行版现在包括一个 `pyproject.toml` 文件，以支持完整的 [**PEP 517**](https://peps.python.org/pep-0517/) 支持。特别是，这允许使用 `pip` 进行本地源构建时自动安装 [Cython](https://cython.org/) 可选依赖项。

[#7311](https://www.sqlalchemy.org/trac/ticket/7311)  ## C 扩展现在已转换为 Cython

SQLAlchemy C 扩展已被全部采用 [Cython](https://cython.org/) 编写的全新扩展所取代。虽然在创建 C 扩展时曾于 2010 年评估过 Cython，但如今实际使用的 C 扩展的性质和重点与当时已经发生了很大变化。与此同时，Cython 显然也有了显著的发展，Python 的构建/分发工具链也使我们有可能重新审视它。

切换到 Cython 提供了明显的新优势，而没有明显的不利因素：

+   替换特定 C 扩展的 Cython 扩展已经被全部作为**更快**的扩展进行了基准测试，通常情况下稍微快一点，但有时比 SQLAlchemy 以前包含的几乎所有 C 代码都要显著快。虽然这看起来很神奇，但似乎是 Cython 实现中的一些非显而易见的优化的产物，在许多情况下，这些优化不会出现在直接的 Python 到 C 的函数移植中，特别是对于许多添加到 C 扩展中的自定义集合类型而言。

+   与原始的 C 代码相比，Cython 扩展编写、维护和调试都要容易得多，在大多数情况下与 Python 代码几乎一致。预计未来的版本中会有更多的 SQLAlchemy 元素被移植到 Cython 中，这将打开许多以前无法触及的性能改进的新门。

+   Cython 非常成熟并被广泛使用，包括成为 SQLAlchemy 支持的一些著名数据库驱动程序的基础，包括 `asyncpg`、`psycopg3` 和 `asyncmy`。

与之前的 C 扩展一样，Cython 扩展已经预先构建在 SQLAlchemy 的 wheel 发布中，这些发布会自动提供给 `pip` 从 PyPi 安装。手动构建说明也没有改变，除了对 Cython 的要求。

另请参阅

构建 Cython 扩展

[#7256](https://www.sqlalchemy.org/trac/ticket/7256)  ## 数据库反射的主要架构、性能和 API 增强

完全重新设计了 `Table` 对象及其组件 反射 的内部系统，以允许参与的方言一次性高性能地批量反射数千个表。目前，**PostgreSQL** 和 **Oracle** 方言参与了新的架构，其中 PostgreSQL 方言现在可以比以前快近三倍地反射大量的 `Table` 对象，而 Oracle 方言现在可以比以前快十倍地反射大量的 `Table` 对象。

重新架构主要适用于使用 SELECT 查询系统目录表以反映表的方言，而其余包含的方言可以从这种方法中受益的是 SQL Server 方言。相比之下，MySQL/MariaDB 和 SQLite 方言使用非关系型系统来反映数据库表，并且没有受到现有性能问题的影响。

新 API 兼容先前的系统，并且不需要更改第三方方言以保持兼容性；第三方方言也可以通过实现批量查询模式反射模式来选择新系统。

除此之外，`Inspector` 对象的 API 和行为已经改进和增强，具有更一致的跨方言行为以及新方法和新性能特性。

### 性能概述

源分发包括一个脚本 `test/perf/many_table_reflection.py`，用于测试现有的反射功能以及新功能。其中一部分测试可以在旧版本的 SQLAlchemy 上运行，我们在这里使用它来说明性能差异，以在本地网络连接上一次性反射 250 个 `Table` 对象：

| 方言 | 操作 | SQLA 1.4 时间（秒） | SQLA 2.0 时间（秒） |
| --- | --- | --- | --- |
| postgresql+psycopg2 | `metadata.reflect()`，250 张表 | 8.2 | 3.3 |
| oracle+cx_oracle | `metadata.reflect()`，250 张表 | 60.4 | 6.8 |

### `Inspector()` 的行为变更

对于 SQLAlchemy 包含的 SQLite、PostgreSQL、MySQL/MariaDB、Oracle 和 SQL Server 的方言，`Inspector.has_table()`、`Inspector.has_sequence()`、`Inspector.has_index()`、`Inspector.get_table_names()`和`Inspector.get_sequence_names()`现在在缓存方面都表现一致：在为特定`Inspector`对象第一次调用后，它们都会完全缓存其结果。在调用相同的`Inspector`对象时创建或删除表/序列的程序将在数据库状态更改后不会收到更新的状态。当要执行 DDL 更改时，应使用调用`Inspector.clear_cache()`或新的`Inspector`。以前，`Inspector.has_table()`、`Inspector.has_sequence()`方法未实现缓存，`Inspector`也不支持这些方法的缓存，而`Inspector.get_table_names()`和`Inspector.get_sequence_names()`方法则是，导致两种方法之间的结果不一致。

第三方方言的行为取决于它们是否实现了这些方法的方言级别实现的“反射缓存”装饰器。

### 新的方法和`Inspector()`的改进

+   添加了一个方法`Inspector.has_schema()`，用于返回目标数据库中是否存在模式

+   添加了一个方法`Inspector.has_index()`，用于返回表是否具有特定索引。

+   对一次只对单个表起作用的检查方法，例如`Inspector.get_columns()`，现在应该一致地引发`NoSuchTableError`如果找不到表或视图; 此更改特定于各个方言，因此对于现有的第三方方言可能不适用。

+   将“视图”和“材料化视图”的处理分开，因为在现实世界的用例中，这两个构造使用不同的 DDL 来创建和删除; 这包括现在有单独的`Inspector.get_view_names()`和`Inspector.get_materialized_view_names()`方法。

[#4379](https://www.sqlalchemy.org/trac/ticket/4379)  ## 对 psycopg 3（也称为“psycopg”）的方言支持

为[psycopg 3](https://pypi.org/project/psycopg/) DBAPI 增加了方言支持，尽管现在被称为`psycopg`，但它的包名仍然取代了之前的`psycopg2`包，后者目前仍然是 SQLAlchemy“默认”的`postgresql`方言驱动程序。 `psycopg`是一个完全重新设计和现代化的用于 PostgreSQL 的数据库适配器，支持诸如准备语句和 Python asyncio 等概念。

`psycopg`是 SQLAlchemy 支持的第一个同时提供 pep-249 同步 API 和 asyncio 驱动程序的 DBAPI。可以使用相同的`psycopg`数据库 URL 与`create_engine()`和`create_async_engine()`引擎创建函数，自动选择相应的同步或 asyncio 版本的方言。

另请参阅

psycopg  ## 对 oracledb 的方言支持

为[oracledb](https://pypi.org/project/oracledb/) DBAPI 增加了方言支持，这是流行的 cx_Oracle 驱动程序的重命名的新主要版本。

另请参阅

python-oracledb  ## 新的条件 DDL 用于约束和索引

一个新的方法 `Constraint.ddl_if()` 和 `Index.ddl_if()` 允许诸如 `CheckConstraint`、`UniqueConstraint` 和 `Index` 这样的构造在给定的 `Table` 上有条件地呈现，基于与 `DDLElement.execute_if()` 方法接受的相同类型的条件。在下面的示例中，CHECK 约束和索引只会针对 PostgreSQL 后端生成：

```py
meta = MetaData()

my_table = Table(
    "my_table",
    meta,
    Column("id", Integer, primary_key=True),
    Column("num", Integer),
    Column("data", String),
    Index("my_pg_index", "data").ddl_if(dialect="postgresql"),
    CheckConstraint("num > 5").ddl_if(dialect="postgresql"),
)

e1 = create_engine("sqlite://", echo=True)
meta.create_all(e1)  # will not generate CHECK and INDEX

e2 = create_engine("postgresql://scott:tiger@localhost/test", echo=True)
meta.create_all(e2)  # will generate CHECK and INDEX
```

另见

控制约束和索引的 DDL 生成

[#7631](https://www.sqlalchemy.org/trac/ticket/7631)  ## DATE、TIME、DATETIME 数据类型现在在所有后端上都支持字面渲染

字面渲染现在已经实现了对于日期和时间类型的后端特定编译，包括 PostgreSQL 和 Oracle：

```py
>>> import datetime

>>> from sqlalchemy import DATETIME
>>> from sqlalchemy import literal
>>> from sqlalchemy.dialects import oracle
>>> from sqlalchemy.dialects import postgresql

>>> date_literal = literal(datetime.datetime.now(), DATETIME)

>>> print(
...     date_literal.compile(
...         dialect=postgresql.dialect(), compile_kwargs={"literal_binds": True}
...     )
... )
'2022-12-17 11:02:13.575789'
>>> print(
...     date_literal.compile(
...         dialect=oracle.dialect(), compile_kwargs={"literal_binds": True}
...     )
... )
TO_TIMESTAMP('2022-12-17 11:02:13.575789',  'YYYY-MM-DD HH24:MI:SS.FF') 
```

以前，这样的字面渲染仅在未提供任何方言的情况下将语句转换为字符串时才起作用；当尝试使用特定于方言的类型进行渲染时，会引发 `NotImplementedError`，直到 SQLAlchemy 1.4.45，这变成了一个 `CompileError`（部分来源于 [#8800](https://www.sqlalchemy.org/trac/ticket/8800)）。

当使用 PostgreSQL、MySQL、MariaDB、MSSQL、Oracle 方言提供的 SQL 编译器的 `literal_binds` 时，默认渲染是修改后的 ISO-8601 渲染（即将 T 转换为空格的 ISO-8601）。对于 Oracle，ISO 格式被包装在适当的 TO_DATE() 函数调用中。对于 SQLite，渲染保持不变，因为这个方言始终为日期值包含字符串渲染。

[#5052](https://www.sqlalchemy.org/trac/ticket/5052)  ## `Result`、`AsyncResult` 的上下文管理器支持

`Result` 对象现在支持上下文管理器的使用，这将确保对象及其底层游标在块结束时关闭。这在特定于服务器端游标的情况下特别有用，在这种情况下，重要的是在操作结束时关闭打开的游标对象，即使发生了用户定义的异常：

```py
with engine.connect() as conn:
    with conn.execution_options(yield_per=100).execute(
        text("select * from table")
    ) as result:
        for row in result:
            print(f"{row}")
```

在使用 asyncio 时，`AsyncResult`和`AsyncConnection`已经改变，以提供可选的异步上下文管理器使用，如下所示：

```py
async with async_engine.connect() as conn:
    async with conn.execution_options(yield_per=100).execute(
        text("select * from table")
    ) as result:
        for row in result:
            print(f"{row}")
```

[#8710](https://www.sqlalchemy.org/trac/ticket/8710)

## 行为变化

本节涵盖了 SQLAlchemy 2.0 中的行为变化，这些变化不属于主要的 1.4->2.0 迁移路径；这里的变化不会对向后兼容性产生重大影响。

### `Session` 的新事务加入模式

“将外部事务加入到会话中”的行为已经修订和改进，允许显式控制`Session`如何适应已经建立了事务并可能已经建立了保存点的传入`Connection`。新参数`Session.join_transaction_mode`包括一系列选项值，可以以几种方式适应现有事务，最重要的是允许`Session`专门使用保存点以全事务方式操作，同时始终将外部启动的事务未提交且活跃，允许测试套件回滚所有在测试中发生的更改。

这带来的主要改进是将会话加入外部事务（例如用于测试套件）的文档化配方，它也从 SQLAlchemy 1.3 更改为 1.4，现在简化为不再需要显式使用事件处理程序或任何提及显式保存点；通过使用`join_transaction_mode="create_savepoint"`，`Session`永远不会影响传入事务的状态，而是创建一个保存点（即“嵌套事务”）作为其根事务。

以下是将会话加入外部事务（例如用于测试套件）中给出的示例的一部分；请参阅该部分以获取完整示例：

```py
class SomeTest(TestCase):
    def setUp(self):
        # connect to the database
        self.connection = engine.connect()

        # begin a non-ORM transaction
        self.trans = self.connection.begin()

        # bind an individual Session to the connection, selecting
        # "create_savepoint" join_transaction_mode
        self.session = Session(
            bind=self.connection, join_transaction_mode="create_savepoint"
        )

    def tearDown(self):
        self.session.close()

        # rollback non-ORM transaction
        self.trans.rollback()

        # return connection to the Engine
        self.connection.close()
```

选定的`Session.join_transaction_mode`的默认模式是`"conditional_savepoint"`，如果给定的`Connection`本身已经在一个保存点上，则使用`"create_savepoint"`行为。如果给定的`Connection`在一个事务中但不在一个保存点上，则`Session`会传播“回滚”调用但不会传播“提交”调用，但不会自行开始一个新的保存点。这种行为被默认选择是因为它与旧版本的 SQLAlchemy 兼容性最大，并且只有在给定的驱动程序已经使用 SAVEPOINT 时才会开始一个新的 SAVEPOINT，因为对 SAVEPOINT 的支持不仅取决于特定的后端和驱动程序，还取决于配置。

以下说明了一个在 SQLAlchemy 1.3 中工作的情况，在 SQLAlchemy 1.4 中停止工作，并在 SQLAlchemy 2.0 中恢复的情况：

```py
engine = create_engine("...")

# setup outer connection with a transaction and a SAVEPOINT
conn = engine.connect()
trans = conn.begin()
nested = conn.begin_nested()

# bind a Session to that connection and operate upon it, including
# a commit
session = Session(conn)
session.connection()
session.commit()
session.close()

# assert both SAVEPOINT and transaction remain active
assert nested.is_active
nested.rollback()
trans.rollback()
```

在上述情况下，`Session`与一个在其上启动了保存点的`Connection`连接在一起；这两个单元的状态在`Session`处理完事务后保持不变。在 SQLAlchemy 1.3 中，上述情况能够正常工作是因为`Session`会在`Connection`上开始一个“子事务”，这样外部保存点/事务可以保持不受影响，就像上面的简单情况一样。由于子事务在 1.4 中已被弃用并在 2.0 中被移除，这种行为不再可用。新的默认行为通过使用一个真正的第二个 SAVEPOINT 来改进“子事务”的行为，因此即使调用`Session.rollback()`也会阻止`Session`“跳出”到外部启动的 SAVEPOINT 或事务。

将一个已启动事务的`Connection`加入到一个`Session`中的新代码应该明确选择一个`Session.join_transaction_mode`，以便明确定义所需的行为。

[#9015](https://www.sqlalchemy.org/trac/ticket/9015)  ### `str(engine.url)` 现在默认会混淆密码

为了避免数据库密码泄漏，在`URL`上调用`str()`现在将默认启用密码混淆功能。以前，这种混淆会在`__repr__()`调用中生效，但不会在`__str__()`中生效。这种变化将影响那些试图从另一个引擎传递字符串化的 URL 调用`create_engine()`的应用程序和测试套件，例如：

```py
>>> e1 = create_engine("postgresql+psycopg2://scott:tiger@localhost/test")
>>> e2 = create_engine(str(e1.url))
```

上述引擎`e2`将不会具有正确的密码；它将具有混淆的字符串`"***"`。

上述模式的首选方法是直接传递`URL`对象，无需将其字符串化：

```py
>>> e1 = create_engine("postgresql+psycopg2://scott:tiger@localhost/test")
>>> e2 = create_engine(e1.url)
```

否则，对于具有明文密码的字符串化 URL，请使用`URL.render_as_string()`方法，将`URL.render_as_string.hide_password`参数设置为`False`：

```py
>>> e1 = create_engine("postgresql+psycopg2://scott:tiger@localhost/test")
>>> url_string = e1.url.render_as_string(hide_password=False)
>>> e2 = create_engine(url_string)
```

[#8567](https://www.sqlalchemy.org/trac/ticket/8567)  ### 对具有相同名称和键的表对象中列的替换规则更加严格

对于向`Table`对象附加`Column`对象，现在有更严格的规则，将一些先前的弃用警告转换为异常，并阻止一些先前可能导致表中出现重复列的情况，当`Table.extend_existing`设置为`True`时，无论是在编程式`Table`构建还是在反射操作期间。

+   无论如何，`Table`对象永远不应该具有两个或更多具有相同名称的`Column`对象，无论它们有什么`.key`。已经确定并修复了一个仍然可能发生这种情况的边缘情况。

+   向具有与现有 `Column` 相同名称或键的 `Table` 添加 `Column` 将始终引发 `DuplicateColumnError`（在 2.0.0b4 中是 `ArgumentError` 的新子类），除非存在其他参数；`Table.append_column.replace_existing` 用于 `Table.append_column()`，以及 `Table.extend_existing` 用于构建与现有表同名的表，无论是否使用了反射。以前，此场景会出现弃用警告。

+   现在，如果创建了一个包含 `Table` 的警告，该警告会包括 `Table.extend_existing`，其中一个没有单独的 `Column.key` 的传入 `Column` 将完全替换一个具有键的现有 `Column`，这表明操作不是用户想要的。这可能特别发生在二次反射步骤期间，例如 `metadata.reflect(extend_existing=True)`。警告建议将 `Table.autoload_replace` 参数设置为 `False` 以防止此问题。在 1.4 及更早版本中，传入的列将**额外添加**到现有列中。这是一个错误，在 2.0（截至 2.0.0b4）中是行为更改，因为当这种情况发生时，先前的键将**不再存在**于列集合中。

[#8925](https://www.sqlalchemy.org/trac/ticket/8925)  ### ORM Declarative 对列顺序的应用方式不同；使用 `sort_order` 控制行为

声明式已经改变了从混入或抽象基类派生的映射列的排序系统，以及与声明类本身上的列一起排序，将来自声明类的列放在首位，然后是混入列。以下映射：

```py
class Foo:
    col1 = mapped_column(Integer)
    col3 = mapped_column(Integer)

class Bar:
    col2 = mapped_column(Integer)
    col4 = mapped_column(Integer)

class Model(Base, Foo, Bar):
    id = mapped_column(Integer, primary_key=True)
    __tablename__ = "model"
```

在 1.4 上产生一个 CREATE TABLE 如下所示：

```py
CREATE  TABLE  model  (
  col1  INTEGER,
  col3  INTEGER,
  col2  INTEGER,
  col4  INTEGER,
  id  INTEGER  NOT  NULL,
  PRIMARY  KEY  (id)
)
```

而在 2.0 上则产生：

```py
CREATE  TABLE  model  (
  id  INTEGER  NOT  NULL,
  col1  INTEGER,
  col3  INTEGER,
  col2  INTEGER,
  col4  INTEGER,
  PRIMARY  KEY  (id)
)
```

对于上述特定情况，这可以看作是一种改进，因为 `Model` 上的主键列现在通常在一个人更喜欢的地方。然而，对于以另一种方式定义模型的应用程序来说，这并不令人舒服，因为：

```py
class Foo:
    id = mapped_column(Integer, primary_key=True)
    col1 = mapped_column(Integer)
    col3 = mapped_column(Integer)

class Model(Foo, Base):
    col2 = mapped_column(Integer)
    col4 = mapped_column(Integer)
    __tablename__ = "model"
```

现在，这将产生 CREATE TABLE 输出如下所示：

```py
CREATE  TABLE  model  (
  col2  INTEGER,
  col4  INTEGER,
  id  INTEGER  NOT  NULL,
  col1  INTEGER,
  col3  INTEGER,
  PRIMARY  KEY  (id)
)
```

为了解决这个问题，SQLAlchemy 2.0.4 引入了一个新参数 `mapped_column()`，名为 `mapped_column.sort_order`，它是一个整数值，默认为 `0`，可以设置为正值或负值，以使列在其他列之前或之后排列，如下面的示例所示：

```py
class Foo:
    id = mapped_column(Integer, primary_key=True, sort_order=-10)
    col1 = mapped_column(Integer, sort_order=-1)
    col3 = mapped_column(Integer)

class Model(Foo, Base):
    col2 = mapped_column(Integer)
    col4 = mapped_column(Integer)
    __tablename__ = "model"
```

上述模型将 “id” 放在所有其他列之前，将 “col1” 放在 “id” 之后：

```py
CREATE  TABLE  model  (
  id  INTEGER  NOT  NULL,
  col1  INTEGER,
  col2  INTEGER,
  col4  INTEGER,
  col3  INTEGER,
  PRIMARY  KEY  (id)
)
```

未来的 SQLAlchemy 版本可能会选择为 `mapped_column` 结构提供显式的排序提示，因为这种排序是 ORM 特定的。### `Sequence` 结构恢复为没有任何显式默认的 “start” 值；影响 MS SQL Server

在 SQLAlchemy 1.4 之前，如果未指定其他参数，则 `Sequence` 结构将仅发出简单的 `CREATE SEQUENCE` DDL：

```py
>>> # SQLAlchemy 1.3 (and 2.0)
>>> from sqlalchemy import Sequence
>>> from sqlalchemy.schema import CreateSequence
>>> print(CreateSequence(Sequence("my_seq")))
CREATE  SEQUENCE  my_seq 
```

然而，由于在 MS SQL Server 上增加了对 `Sequence` 的支持，其中默认的起始值设置为 `-2**63`，版本 1.4 决定将 DDL 默认设置为 1，如果未提供 `Sequence.start` 参数：

```py
>>> # SQLAlchemy 1.4 (only)
>>> from sqlalchemy import Sequence
>>> from sqlalchemy.schema import CreateSequence
>>> print(CreateSequence(Sequence("my_seq")))
CREATE  SEQUENCE  my_seq  START  WITH  1 
```

这个改变引入了其他复杂性，包括当包括 `Sequence.min_value` 参数时，这个 `1` 的默认值实际上应该默认为 `Sequence.min_value` 声明的内容，否则，一个低于起始值的 min_value 可能会被视为矛盾的。由于观察这个问题开始变得有点复杂，我们决定撤销这个改变，并恢复 `Sequence` 的原始行为，即不表达任何观点，只是发出 CREATE SEQUENCE，允许数据库自己决定 `SEQUENCE` 的各个参数如何相互作用。

因此，为了确保所有后端的起始值都为 1，**可能需要显式指定起始值为 1**，如下所示：

```py
>>> # All SQLAlchemy versions
>>> from sqlalchemy import Sequence
>>> from sqlalchemy.schema import CreateSequence
>>> print(CreateSequence(Sequence("my_seq", start=1)))
CREATE  SEQUENCE  my_seq  START  WITH  1 
```

此外，对于在现代后端包括 PostgreSQL、Oracle、SQL Server 上自动生成整数主键，应优先使用`Identity`构造，这在 1.4 和 2.0 中的行为没有变化。

[#7211](https://www.sqlalchemy.org/trac/ticket/7211)  ### “with_variant()”克隆原始 TypeEngine 而不是更改类型

`TypeEngine.with_variant()`方法用于将特定类型应用于每个数据库的替代行为，现在返回原始`TypeEngine`对象的副本，其中包含内部存储的变体信息，而不是将其包装在`Variant`类中。

而以前的`Variant`方法能够通过动态属性获取器维护原始类型的所有 Python 行为，这里的改进是当调用变体时，返回的类型仍然是原始类型的实例，这与 mypy 和 pylance 等类型检查器更加顺畅地配合。给定以下程序：

```py
import typing

from sqlalchemy import String
from sqlalchemy.dialects.mysql import VARCHAR

type_ = String(255).with_variant(VARCHAR(255, charset="utf8mb4"), "mysql", "mariadb")

if typing.TYPE_CHECKING:
    reveal_type(type_)
```

类型检查器如 pyright 现在将报告类型为：

```py
info: Type of "type_" is "String"
```

此外，如上所示，可以为单个类型传递多个方言名称，特别是对于被视为 SQLAlchemy 1.4 的`"mysql"`和`"mariadb"`方言对来说，这是有帮助的。

[#6980](https://www.sqlalchemy.org/trac/ticket/6980)  ### Python 除法运算符对所有后端执行真除法；添加地板除法

核心表达语言现在支持“真除法”（即 Python 运算符`/`）和“地板除法”（即 Python 运算符`//`），包括后端特定的行为以规范不同数据库在这方面的行为。

给定两个整数值进行“真除法”操作：

```py
expr = literal(5, Integer) / literal(10, Integer)
```

例如，在 PostgreSQL 上，SQL 除法运算符在针对整数时通常作为“地板除法”运行，这意味着上述结果将返回整数“0”。对于这样的后端，SQLAlchemy 现在呈现的 SQL 形式等效于：

```py
%(param_1)s  /  CAST(%(param_2)s  AS  NUMERIC)
```

使用`param_1=5`，`param_2=10`，以便返回表达式的类型为 NUMERIC，通常作为 Python 值`decimal.Decimal("0.5")`。

给定两个整数值进行“地板除法”操作：

```py
expr = literal(5, Integer) // literal(10, Integer)
```

例如，在 MySQL 和 Oracle 上，SQL 除法运算符在针对整数时通常作为“真除法”运行，这意味着上述结果将返回浮点值“0.5”。对于这些和类似的后端，SQLAlchemy 现在呈现的 SQL 形式等效于：

```py
FLOOR(%(param_1)s  /  %(param_2)s)
```

使用`param_1=5`，`param_2=10`，以便返回表达式的类型为 INTEGER，如 Python 值`0`。

此处的不兼容更改是，如果一个应用程序使用 PostgreSQL、SQL Server 或 SQLite，并依赖于 Python 的“truediv”运算符在所有情况下返回整数值。依赖于此行为的应用程序应该使用 Python 的“floor division”运算符 `//` 进行这些操作，或者在使用之前的 SQLAlchemy 版本时，使用 floor 函数以实现向前兼容性：

```py
expr = func.floor(literal(5, Integer) / literal(10, Integer))
```

在任何 SQLAlchemy 2.0 之前的版本中，都需要上述形式来提供与后端无关的地板除法。

[#4926](https://www.sqlalchemy.org/trac/ticket/4926)  ### 当检测到非法并发或重入访问时，会主动引发会话错误

`Session` 现在可以捕获更多与多线程或其他并发场景中的非法并发状态更改以及执行意外状态更改的事件钩子相关的错误。

当一个 `Session` 在多个线程同时使用时，已知会发生的一个错误是 `AttributeError: 'NoneType' object has no attribute 'twophase'`，这完全晦涩难懂。这个错误发生在一个线程调用 `Session.commit()` 时，内部调用 `SessionTransaction.close()` 方法来结束事务上下文，同时另一个线程正在运行查询，如 `Session.execute()`。在 `Session.execute()` 中，获取当前事务的数据库连接的内部方法首先开始断言会话是“活动的”，但在此断言通过后，同时进行的对 `Session.close()` 的并发调用会干扰这种状态，导致上述未定义的条件。

该更改对围绕 `SessionTransaction` 对象的所有改变状态的方法应用了保护措施，因此在上述情况下，`Session.commit()` 方法将会失败，因为它将试图将状态更改为在已经进行中的方法期间不允许的状态，而该方法希望获取当前连接以运行数据库查询。

使用在 [#7433](https://www.sqlalchemy.org/trac/ticket/7433) 中说明的测试脚本，先前的错误案例如下：

```py
Traceback (most recent call last):
File "/home/classic/dev/sqlalchemy/test3.py", line 30, in worker
    sess.execute(select(A)).all()
File "/home/classic/tmp/sqlalchemy/lib/sqlalchemy/orm/session.py", line 1691, in execute
    conn = self._connection_for_bind(bind)
File "/home/classic/tmp/sqlalchemy/lib/sqlalchemy/orm/session.py", line 1532, in _connection_for_bind
    return self._transaction._connection_for_bind(
File "/home/classic/tmp/sqlalchemy/lib/sqlalchemy/orm/session.py", line 754, in _connection_for_bind
    if self.session.twophase and self._parent is None:
AttributeError: 'NoneType' object has no attribute 'twophase'
```

当`_connection_for_bind()`方法无法继续运行，因为并发访问使其处于无效状态时。使用新方法，状态更改的发起者会抛出错误：

```py
File "/home/classic/dev/sqlalchemy/lib/sqlalchemy/orm/session.py", line 1785, in close
   self._close_impl(invalidate=False)
File "/home/classic/dev/sqlalchemy/lib/sqlalchemy/orm/session.py", line 1827, in _close_impl
   transaction.close(invalidate)
File "<string>", line 2, in close
File "/home/classic/dev/sqlalchemy/lib/sqlalchemy/orm/session.py", line 506, in _go
   raise sa_exc.InvalidRequestError(
sqlalchemy.exc.InvalidRequestError: Method 'close()' can't be called here;
method '_connection_for_bind()' is already in progress and this would cause
an unexpected state change to symbol('CLOSED')
```

状态转换检查故意不使用显式锁来检测并发线程活动，而是依赖于简单的属性设置/值测试操作，当意外的并发更改发生时，这些操作会自然失败。其理念在于，该方法可以检测到完全在单个线程内发生的非法状态更改，例如在事件处理程序上运行会话事务事件调用了一个不被期望的改变状态的方法，或者在 asyncio 中，如果一个特定的`Session`被多个 asyncio 任务共享，以及在使用类似 gevent 的补丁式并发方法时。

[#7433](https://www.sqlalchemy.org/trac/ticket/7433)  ### SQLite 方言在基于文件的数据库中使用 QueuePool

当使用基于文件的数据库时，SQLite 方言现在默认使用 `QueuePool`。这是与将 `check_same_thread` 参数设置为 `False` 一起设置的。已经观察到，以前默认使用 `NullPool` 的方法，在释放数据库连接后不保留连接，实际上对性能产生了可衡量的负面影响。如常，池类可通过 `create_engine.poolclass` 参数进行自定义。

另请参阅

线程/池行为

[#7490](https://www.sqlalchemy.org/trac/ticket/7490)  ### 新的 Oracle FLOAT 类型，具有二进制精度；不直接接受十进制精度

Oracle 方言现已添加了新的数据类型 `FLOAT`，以配合 `Double` 和数据库特定的 `DOUBLE`、`DOUBLE_PRECISION` 和 `REAL` 数据类型的添加。Oracle 的 `FLOAT` 接受所谓的“二进制精度”参数，根据 Oracle 文档，大致为标准“精度”值除以 0.3103：

```py
from sqlalchemy.dialects import oracle

Table("some_table", metadata, Column("value", oracle.FLOAT(126)))
```

二进制精度值 126 与使用 `DOUBLE_PRECISION` 数据类型是等效的，值 63 等效于使用 `REAL` 数据类型。其他精度值特定于 `FLOAT` 类型本身。

SQLAlchemy `Float` 数据类型也接受“精度”参数，但这是十进制精度，Oracle 不接受。与其尝试猜测转换，Oracle 方言现在将在针对 Oracle 后端使用带有精度值的 `Float` 时引发一个信息丰富的错误。为了为支持的后端指定具有显式精度值的 `Float` 数据类型，同时还支持其他后端，可以使用`TypeEngine.with_variant()` 方法，如下所示：

```py
from sqlalchemy.types import Float
from sqlalchemy.dialects import oracle

Table(
    "some_table",
    metadata,
    Column("value", Float(5).with_variant(oracle.FLOAT(16), "oracle")),
)
```  ### PostgreSQL 后端的新 RANGE / MULTIRANGE 支持和更改

RANGE / MULTIRANGE 支持已完全实现为 psycopg2、psycopg3 和 asyncpg 方言。新支持使用了一个新的 SQLAlchemy 特定的 `Range` 对象，它对不同的后端是不可知的，不需要使用后端特定的导入或扩展步骤。对于多范围支持，使用 `Range` 对象的列表。

之前使用的特定于 psycopg2 的类型的代码应修改为使用`Range`，它提供了兼容的接口。

`Range` 对象还具有与 PostgreSQL 相同的比较支持。到目前为止已实现了 `Range.contains()` 和 `Range.contained_by()` 方法，它们的工作方式与 PostgreSQL 的 `@>` 和 `<@` 相同。未来的版本可能会添加其他运算符支持。

请参阅范围和多范围类型处的文档，了解如何使用新功能的背景信息。

另请参阅

范围和多范围类型

[#7156](https://www.sqlalchemy.org/trac/ticket/7156) [#8706](https://www.sqlalchemy.org/trac/ticket/8706)  ### 在 PostgreSQL 上使用`match()`操作符使用`plainto_tsquery()`而不是`to_tsquery()`

`Operators.match()`函数现在在 PostgreSQL 后端上呈现为`col @@ plainto_tsquery(expr)`，而不是`col @@ to_tsquery()`。`plainto_tsquery()`接受纯文本，而`to_tsquery()`接受专门的查询符号，因此与其他后端的兼容性较差。

通过使用`func`生成 PostgreSQL 特定函数和`Operators.bool_op()`（`Operators.op()`的布尔类型版本）来生成任意运算符，以与之前版本相同的方式，所有 PostgreSQL 搜索函数和操作符都可用。请参阅全文搜索中的示例。

现有的 SQLAlchemy 项目如果在`Operators.match()`中使用了 PG 特定的指令，应该直接使用`func.to_tsquery()`。要以与 1.4 版本完全相同的形式呈现 SQL，请参阅使用 match()进行简单纯文本匹配的版本说明。

[#7086](https://www.sqlalchemy.org/trac/ticket/7086)

## Core 和 ORM 中的新类型支持-不再使用存根/扩展

与 1.4 版本中通过[sqlalchemy2-stubs](https://github.com/sqlalchemy/sqlalchemy2-stubs)包提供的临时方法相比，Core 和 ORM 的类型化方法已完全重写。新方法始于 SQLAlchemy 中最基本的元素，即`Column`，更准确地说是底层所有具有类型的 SQL 表达式的`ColumnElement`。然后，这个表达式级别的类型化扩展到语句构造、语句执行和结果集的领域，最终扩展到 ORM，其中新的声明性形式允许完全类型化的 ORM 模型，从语句到结果集全部集成。

提示

对于 2.0 系列，类型支持应被视为**beta 级别**软件。类型详细信息可能会发生变化，但不计划进行重大不兼容更改。

### SQL 表达式/语句/结果集类型化

本节提供了 SQLAlchemy 的新 SQL 表达类型方法的背景和示例，它从基本的`ColumnElement`构造扩展到 SQL 语句和结果集，以及 ORM 映射的领域。

#### 原因和概述

提示

本节是一个架构讨论。 跳转到 SQL 表达类型 - 示例以查看新的类型化外观。

在[sqlalchemy2-stubs](https://github.com/sqlalchemy/sqlalchemy2-stubs)中，SQL 表达式被类型化为[泛型](https://peps.python.org/pep-0484/#generics)，然后引用`TypeEngine`对象，如`Integer`，`DateTime`，或`String`作为它们的泛型参数（如`Column[Integer]`）。 这本身就是与最初的 Dropbox [sqlalchemy-stubs](https://github.com/dropbox/sqlalchemy-stubs)包不同的地方，其中`Column`及其基本构造直接以 Python 类型为泛型，例如`int`，`datetime`和`str`。 希望由于`Integer` / `DateTime` / `String`本身是对`int` / `datetime` / `str`的泛型，因此有方法可以保持两个级别的信息，并且可以通过`TypeEngine`从列表达式中提取 Python 类型作为中间构造。 但是，事实并非如此，因为[**PEP 484**](https://peps.python.org/pep-0484/)没有足够丰富的功能集使其可行，缺乏诸如[higher kinded TypeVars](https://github.com/python/typing/issues/548)之类的功能。

经过对[当前能力的深入评估](https://github.com/python/typing/discussions/999)，SQLAlchemy 2.0 实现了[**PEP 484**](https://peps.python.org/pep-0484/)的原始智慧，直接将列表达式与 Python 类型进行关联。这意味着，如果有不同子类型的 SQL 表达式，比如`Column(VARCHAR)`和`Column(Unicode)`，那么这两个`String`子类型的具体信息不会随类型一起传递，因为类型只会携带`str`，但实际上这通常不是问题，通常更有用的是 Python 类型立即出现，因为它代表了将直接存储和接收的 Python 数据。

具体来说，这意味着像`Column('id', Integer)`这样的表达式被类型化为`Column[int]`。这允许建立一个可行的 SQLAlchemy 构造 -> Python 数据类型的管道，而无需使用类型插件。至关重要的是，它允许与 ORM 使用`select()`和`Row`构造的完全互操作性，这些构造引用 ORM 映射的类类型（例如，包含用户映射实例的`Row`，如我们教程中使用的`User`和`Address`示例）。虽然 Python 类型当前对元组类型的定制支持非常有限（其中[**PEP 646**](https://peps.python.org/pep-0646/)是第一个试图处理类似元组对象的 pep，[故意在功能上受到限制](https://mail.python.org/archives/list/typing-sig@python.org/message/G2PNHRR32JMFD3JR7ACA2NDKWTDSEPUG/)，并且本身尚不适用于任意元组操作），但已经设计出了一个相当不错的方法，允许基本的`select()` -> `Result` -> `Row`类型化功能，包括 ORM 类，其中在将`Row`对象解包为单独列条目时，添加了一个小的面向类型的访问器，允许各个 Python 值保持与其来源的 SQL 表达式相关联的 Python 类型（翻译：它有效）。

#### SQL 表达式类型化 - 示例

简要介绍了类型行为。注释指示在[vscode](https://code.visualstudio.com/)中悬停在代码上会看到什么（或者使用[reveal_type()](https://mypy.readthedocs.io/en/latest/common_issues.html?highlight=reveal_type#reveal-type)辅助工具时大致会显示什么）：

+   简单的 Python 类型分配给 SQL 表达式

    ```py
    # (variable) str_col: ColumnClause[str]
    str_col = column("a", String)

    # (variable) int_col: ColumnClause[int]
    int_col = column("a", Integer)

    # (variable) expr1: ColumnElement[str]
    expr1 = str_col + "x"

    # (variable) expr2: ColumnElement[int]
    expr2 = int_col + 10

    # (variable) expr3: ColumnElement[bool]
    expr3 = int_col == 15
    ```

+   分配给`select()`构造的单个 SQL 表达式以及任何返回行的构造，包括返回行的 DML，如带有`Insert.returning()`的`Insert`，都打包成一个保留每个元素 Python 类型的`Tuple[]`类型。

    ```py
    # (variable) stmt: Select[Tuple[str, int]]
    stmt = select(str_col, int_col)

    # (variable) stmt: ReturningInsert[Tuple[str, int]]
    ins_stmt = insert(table("t")).returning(str_col, int_col)
    ```

+   任何返回行的结构中的`Tuple[]`类型，在调用`.execute()`方法时，传递到`Result`和`Row`。为了将`Row`对象解包为元组，`Row.tuple()`或`Row.t`访问器本质上将`Row`转换为相应的`Tuple[]`（但在运行时仍保持相同的`Row`对象）。

    ```py
    with engine.connect() as conn:
        # (variable) stmt: Select[Tuple[str, int]]
        stmt = select(str_col, int_col)

        # (variable) result: Result[Tuple[str, int]]
        result = conn.execute(stmt)

        # (variable) row: Row[Tuple[str, int]] | None
        row = result.first()

        if row is not None:
            # for typed tuple unpacking or indexed access,
            # use row.tuple() or row.t  (this is the small typing-oriented accessor)
            strval, intval = row.t

            # (variable) strval: str
            strval

            # (variable) intval: int
            intval
    ```

+   对于单列语句的标量值，使用`Connection.scalar()`、`Result.scalars()`等方法是正确的。

    ```py
    # (variable) data: Sequence[str]
    data = connection.execute(select(str_col)).scalars().all()
    ```

+   对于 ORM 映射类，上述对返回行构造的支持与其最配套，因为映射类可以列出其成员的特定类型。下面的示例设置了一个使用新的类型感知语法的类，将在下一节中描述：

    ```py
    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column

    class Base(DeclarativeBase):
        pass

    class User(Base):
        __tablename__ = "user_account"

        id: Mapped[int] = mapped_column(primary_key=True)
        name: Mapped[str]
        addresses: Mapped[List["Address"]] = relationship()

    class Address(Base):
        __tablename__ = "address"

        id: Mapped[int] = mapped_column(primary_key=True)
        email_address: Mapped[str]
        user_id = mapped_column(ForeignKey("user_account.id"))
    ```

    通过上述映射，属性从语句到结果集一路类型化并表达自己：

    ```py
    with Session(engine) as session:
        # (variable) stmt: Select[Tuple[int, str]]
        stmt_1 = select(User.id, User.name)

        # (variable) result_1: Result[Tuple[int, str]]
        result_1 = session.execute(stmt_1)

        # (variable) intval: int
        # (variable) strval: str
        intval, strval = result_1.one().t
    ```

    映射类本身也是类型，并且行为相同，例如针对两个映射类进行 SELECT 查询：

    ```py
    with Session(engine) as session:
        # (variable) stmt: Select[Tuple[User, Address]]
        stmt_2 = select(User, Address).join_from(User, Address)

        # (variable) result_2: Result[Tuple[User, Address]]
        result_2 = session.execute(stmt_2)

        # (variable) user_obj: User
        # (variable) address_obj: Address
        user_obj, address_obj = result_2.one().t
    ```

    当选择映射类时，像`aliased`这样的构造也能正常工作，保持原始映射类的列级属性以及语句期望的返回类型：

    ```py
    with Session(engine) as session:
        # this is in fact an Annotated type, but typing tools don't
        # generally display this

        # (variable) u1: Type[User]
        u1 = aliased(User)

        # (variable) stmt: Select[Tuple[User, User, str]]
        stmt = select(User, u1, User.name).filter(User.id == 5)

        # (variable) result: Result[Tuple[User, User, str]]
        result = session.execute(stmt)
    ```

+   核心表（Core Table）目前还没有一个合适的方法来在通过`Table.c`访问时维护`Column`对象的类型。

    由于`Table`被设置为类的实例，并且`Table.c`访问器通常通过名称动态访问`Column`对象，因此尚未为此建立类型化方法；需要一些替代语法。

+   ORM 类、标量等工作得很好。

    选择 ORM 类作为标量或元组的典型用例都有效，无论是 2.0 还是 1.x 风格的查询，都可以获得确切的类型，无论是单独还是包含在适当的容器中，如`Sequence[]`、`List[]`或`Iterator[]`：

    ```py
    # (variable) users1: Sequence[User]
    users1 = session.scalars(select(User)).all()

    # (variable) user: User
    user = session.query(User).one()

    # (variable) user_iter: Iterator[User]
    user_iter = iter(session.scalars(select(User)))
    ```

+   传统的`Query`也获得了元组类型。

    对于`Query`的类型支持远远超出了[sqlalchemy-stubs](https://github.com/dropbox/sqlalchemy-stubs)或[sqlalchemy2-stubs](https://github.com/sqlalchemy/sqlalchemy2-stubs)提供的范围，其中标量对象以及元组类型的`Query`对象将保留大多数情况下的结果级别类型：

    ```py
    # (variable) q1: RowReturningQuery[Tuple[int, str]]
    q1 = session.query(User.id, User.name)

    # (variable) rows: List[Row[Tuple[int, str]]]
    rows = q1.all()

    # (variable) q2: Query[User]
    q2 = session.query(User)

    # (variable) users: List[User]
    users = q2.all()
    ```

#### 注意 - 所有存根必须被卸载

类型支持的一个关键警告是**必须卸载所有 SQLAlchemy 存根包**才能使类型化工作。在针对 Python 虚拟环境运行[mypy](https://mypy.readthedocs.io/en/stable/)时，只需卸载这些包。但是，SQLAlchemy 存根包目前也是[typeshed](https://github.com/python/typeshed)的一部分，它本身捆绑在一些类型工具中，如[Pylance](https://github.com/microsoft/pylance-release)，因此在某些情况下可能需要定位这些包的文件并删除它们，如果它们实际上干扰了新的类型化正确工作。

一旦 SQLAlchemy 2.0 以最终状态发布，typeshed 将从其自己的存根源中删除 SQLAlchemy。

### ORM 声明模型

SQLAlchemy 1.4 引入了第一个使用[sqlalchemy2-stubs](https://github.com/sqlalchemy/sqlalchemy2-stubs)和 Mypy Plugin 组合的 SQLAlchemy 本机 ORM 类型支持。在 SQLAlchemy 2.0 中，Mypy 插件**仍然可用，并已更新以与 SQLAlchemy 2.0 的类型系统一起使用**。但是，现在应该将其视为**已弃用**，因为应用程序现在有一条直接的路径来采用新的类型支持，而不使用插件或存根。

#### 概述

新系统的基本方法是，当使用完全声明式模型（即不使用混合声明式或命令式配置，这些配置不变）时，映射列声明首先在运行时通过检查每个属性声明左侧的类型注释来推导，如果存在的话。左手类型注释应该包含在`Mapped`泛型类型中，否则该属性不被视为映射属性。然后属性声明可以引用右侧的`mapped_column()`构造，用于提供有关要生成和映射的`Column`的附加核心级模式信息。如果左侧存在`Mapped`注释，则此右侧声明是可选的；如果左侧没有注释，则`mapped_column()`可以用作`Column`指令的精确替代，其中它将提供更准确（但不精确）的属性类型行为，即使没有注释存在。

这种方法受到 Python [dataclasses](https://docs.python.org/3/library/dataclasses.html)方法的启发，它从左边开始注释，然后允许右边的可选`dataclasses.field()`规范；与 dataclasses 方法的关键区别在于 SQLAlchemy 的方法是严格的**选择加入**，其中使用`Column`的现有映射如果没有任何类型注释，将继续像以往一样工作，并且`mapped_column()`构造可以直接替换`Column`而不需要任何显式类型注释。只有在确切的属性级 Python 类型存在时，才需要使用带有`Mapped`的显式注释。这些注释可以根据需要，按属性基础使用，对于那些特定类型有帮助的属性；使用`mapped_column()`的未注释属性将在实例级别被标记为`Any`。

#### 迁移现有映射

迁移到新的 ORM 方法开始时更加冗长，但随着可用的新功能的充分利用，变得比以前更简洁。以下步骤详细说明了典型的过渡，然后继续说明了一些更多的选项。

##### 第一步 - `declarative_base()`被`DeclarativeBase`取代。

Python 类型中观察到的一个限制是似乎没有能力从函数动态生成一个类，然后被类型工具理解为新类的基础。为了解决这个问题而不使用插件，通常对`declarative_base()`的调用可以替换为使用`DeclarativeBase`类，它产生与通常相同的`Base`对象，只是类型工具理解它：

```py
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    pass
```

##### 第二步 - 用`mapped_column()`替换`Column`的声明性使用

`mapped_column()`是一个 ORM-typing 感知构造，可以直接用于`Column`的使用。鉴于一个 1.x 风格的映射如下：

```py
from sqlalchemy import Column
from sqlalchemy.orm import relationship
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "user_account"

    id = Column(Integer, primary_key=True)
    name = Column(String(30), nullable=False)
    fullname = Column(String)
    addresses = relationship("Address", back_populates="user")

class Address(Base):
    __tablename__ = "address"

    id = Column(Integer, primary_key=True)
    email_address = Column(String, nullable=False)
    user_id = Column(ForeignKey("user_account.id"), nullable=False)
    user = relationship("User", back_populates="addresses")
```

我们用`mapped_column()`替换`Column`；不需要更改任何参数：

```py
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "user_account"

    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(30), nullable=False)
    fullname = mapped_column(String)
    addresses = relationship("Address", back_populates="user")

class Address(Base):
    __tablename__ = "address"

    id = mapped_column(Integer, primary_key=True)
    email_address = mapped_column(String, nullable=False)
    user_id = mapped_column(ForeignKey("user_account.id"), nullable=False)
    user = relationship("User", back_populates="addresses")
```

上述各列目前都**尚未使用 Python 类型进行类型化**，而是被类型化为`Mapped[Any]`；这是因为我们可以将任何列声明为可选的或非可选的，并且没有办法在我们明确地对其进行类型化时不引起类型错误。

但是，在这一步中，我们上述的映射已经为所有属性设置了适当的描述符，并且可以用于查询以及实例级别的操作，所有这些操作都将在**mypy –strict mode**下通过，而无需插件。

##### 第三步 - 使用`Mapped`根据需要应用确切的 Python 类型。

这可以应用于所有需要确切类型的属性；可以跳过那些可以保留为`Any`的属性。为了上下文，我们还说明了在一个`relationship()`中使用`Mapped`的情况，我们在这个中间步骤中应用了一个确切的类型。映射在这个中间步骤中将更加冗长，但是通过熟练掌握，这一步可以与后续步骤结合起来更直接地更新映射：

```py
from typing import List
from typing import Optional
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "user_account"

    id: Mapped[int] = mapped_column(Integer, primary_key=True)
    name: Mapped[str] = mapped_column(String(30), nullable=False)
    fullname: Mapped[Optional[str]] = mapped_column(String)
    addresses: Mapped[List["Address"]] = relationship("Address", back_populates="user")

class Address(Base):
    __tablename__ = "address"

    id: Mapped[int] = mapped_column(Integer, primary_key=True)
    email_address: Mapped[str] = mapped_column(String, nullable=False)
    user_id: Mapped[int] = mapped_column(ForeignKey("user_account.id"), nullable=False)
    user: Mapped["User"] = relationship("User", back_populates="addresses")
```

此时，我们的 ORM 映射已完全类型化，并将生成精确类型的`select()`、`Query`和`Result`构造。现在我们可以继续减少映射声明中的冗余部分。

##### 第四步 - 移除不再需要的`mapped_column()`指令

所有`nullable`参数都可以使用`Optional[]`隐含；在没有`Optional[]`的情况下，`nullable`默认为`False`。所有没有参数的 SQL 类型，如`Integer`和`String`，可以仅表示为 Python 注释。不带参数的`mapped_column()`指令可以完全删除。现在，`relationship()`从左手注释中派生其类，还支持向前引用（正如`relationship()`已经支持字符串型向前引用十年一样；）：

```py
from typing import List
from typing import Optional
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "user_account"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(30))
    fullname: Mapped[Optional[str]]
    addresses: Mapped[List["Address"]] = relationship(back_populates="user")

class Address(Base):
    __tablename__ = "address"

    id: Mapped[int] = mapped_column(primary_key=True)
    email_address: Mapped[str]
    user_id: Mapped[int] = mapped_column(ForeignKey("user_account.id"))
    user: Mapped["User"] = relationship(back_populates="addresses")
```

##### 第五步 - 利用 pep-593 `Annotated` 将常见指令打包成类型

这是一项全新的功能，提供了一种替代或补充声明性混合的方法，作为提供基于类型的配置的手段，并且在大多数情况下取代了`declared_attr`装饰的函数的需求。

首先，声明式映射允许将 Python 类型映射到 SQL 类型，例如`str`映射到`String`，通过`registry.type_annotation_map`进行定制。使用[**PEP 593**](https://peps.python.org/pep-0593/)中的`Annotated`，我们可以创建特定 Python 类型的变体，以便相同的类型（例如`str`）可以提供`String`的不同变体，如下所示，使用`Annotated` `str`称为`str50`将表示`String(50)`：

```py
from typing_extensions import Annotated
from sqlalchemy.orm import DeclarativeBase

str50 = Annotated[str, 50]

# declarative base with a type-level override, using a type that is
# expected to be used in multiple places
class Base(DeclarativeBase):
    type_annotation_map = {
        str50: String(50),
    }
```

其次，如果使用`Annotated[]`，声明式将从左手类型中提取完整的`mapped_column()`定义，方法是将`mapped_column()`结构作为`Annotated[]`结构的任何参数传递（感谢[@adriangb01](https://twitter.com/adriangb01/status/1532841383647657988)对此想法的说明）。该功能可能会在未来的版本中扩展到包括`relationship()`、`composite()`和其他结构，但目前仅限于`mapped_column()`。下面的示例除了我们的`str50`示例之外，还添加了其他`Annotated`类型，以说明此功能：

```py
from typing_extensions import Annotated
from typing import List
from typing import Optional
from sqlalchemy import ForeignKey
from sqlalchemy import String
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship

# declarative base from previous example
str50 = Annotated[str, 50]

class Base(DeclarativeBase):
    type_annotation_map = {
        str50: String(50),
    }

# set up mapped_column() overrides, using whole column styles that are
# expected to be used in multiple places
intpk = Annotated[int, mapped_column(primary_key=True)]
user_fk = Annotated[int, mapped_column(ForeignKey("user_account.id"))]

class User(Base):
    __tablename__ = "user_account"

    id: Mapped[intpk]
    name: Mapped[str50]
    fullname: Mapped[Optional[str]]
    addresses: Mapped[List["Address"]] = relationship(back_populates="user")

class Address(Base):
    __tablename__ = "address"

    id: Mapped[intpk]
    email_address: Mapped[str50]
    user_id: Mapped[user_fk]
    user: Mapped["User"] = relationship(back_populates="addresses")
```

在上面，使用`Mapped[str50]`、`Mapped[intpk]`或`Mapped[user_fk]`映射的列从`registry.type_annotation_map`以及`Annotated`构造中直接获取，以便重用预先建立的类型和列配置。

##### 可选步骤 - 将映射类转换为[dataclasses](https://docs.python.org/3/library/dataclasses.html)

我们可以将映射类转换为[dataclasses](https://docs.python.org/3/library/dataclasses.html)，其中一个关键优势是我们可以构建一个严格类型化的`__init__()`方法，具有显式的位置参数、仅关键字参数和默认参数，更不用说我们还可以免费获得`__str__()`和`__repr__()`等方法。接下来的部分作为 ORM 模型映射的 Dataclasses 的本机支持进一步说明了上述模型的转换。

##### 从第 3 步开始支持类型注释

通过上述示例，从“第 3 步”开始的任何示例都将包括模型的属性是类型化的，并将传递到`select()`、`Query`和`Row`对象：

```py
# (variable) stmt: Select[Tuple[int, str]]
stmt = select(User.id, User.name)

with Session(e) as sess:
    for row in sess.execute(stmt):
        # (variable) row: Row[Tuple[int, str]]
        print(row)

    # (variable) users: Sequence[User]
    users = sess.scalars(select(User)).all()

    # (variable) users_legacy: List[User]
    users_legacy = sess.query(User).all()
```

另请参阅

使用 mapped_column() 的声明性表 - 更新了声明性文档，用于声明性生成和映射`Table`列。  ### 使用传统 Mypy 类型化模型

使用 Mypy 插件进行 SQLAlchemy 应用程序，其中明确注释不使用`Mapped`在其注释中的构造的应用程序，在新系统下会出现错误，因为这些注释在使用`relationship()`等构造时被标记为错误。

迁移至 2.0 步骤六 - 为显式类型的 ORM 模型添加 __allow_unmapped__ 部分说明了如何临时禁用这些错误，以避免针对使用显式注释的传统 ORM 模型引发错误。

另请参阅

迁移至 2.0 步骤六 - 为显式类型的 ORM 模型添加 __allow_unmapped__  ### 作为 ORM 模型映射的 Dataclasses 的本机支持

上面介绍的新 ORM 声明式特性在 ORM 声明式模型中引入了新的`mapped_column()`构造，并通过可选使用[**PEP 593**](https://peps.python.org/pep-0593/) `Annotated`展示了以类型为中心的映射。我们可以通过将这个与 Python [dataclasses](https://docs.python.org/3/library/dataclasses.html)集成，进一步推进映射。这个新特性是通过[**PEP 681**](https://peps.python.org/pep-0681/)实现的，该特性允许类型检查器识别与数据类兼容或完全是数据类但是通过替代 API 声明的类。

使用数据类特性，映射类获得一个`__init__()`方法，支持位置参数以及可选关键字参数的可定制默认值。正如前面提到的，数据类还生成许多有用的方法，如`__str__()`，`__eq__()`。数据类的序列化方法，如[dataclasses.asdict()](https://docs.python.org/3/library/dataclasses.html#dataclasses.asdict)和[dataclasses.astuple()](https://docs.python.org/3/library/dataclasses.html#dataclasses.astuple)，也可以使用，但目前不支持自引用结构，这使得它们对具有双向关系的映射不太适用。

SQLAlchemy 的当前集成方法将用户定义的类转换为**真正的数据类**，以提供运行时功能；该功能利用了 SQLAlchemy 1.4 中引入的现有数据类功能，以在完全集成的配置样式下生成等效的运行时映射，这种映射比以前的方法更正确地进行了类型标记，并且还使用了在 Python 数据类，attrs 支持的情况下，声明式，命令式映射中介绍的特性。

为了支持符合[**PEP 681**](https://peps.python.org/pep-0681/)的数据类，ORM 构造如`mapped_column()`和`relationship()`接受额外的[**PEP 681**](https://peps.python.org/pep-0681/)参数`init`、`default`和`default_factory`，这些参数被传递给数据类创建过程。这些参数目前必须存在于右侧的显式指令中，就像它们与`dataclasses.field()`一起使用时一样；它们目前不能是左侧`Annotated`构造的本地参数。为了支持方便使用`Annotated`，同时仍支持数据类配置，`mapped_column()`可以将右侧参数的最小集合与位于`Annotated`构造内的现有`mapped_column()`构造合并，以便保持大部分简洁性，正如下面将看到的那样。

为了启用使用类继承的数据类，我们利用了`MappedAsDataclass` mixin，可以直接在每个类上使用，也可以在`Base`类上使用，如下面的示例所示，我们进一步修改了“步骤 5”的示例映射来自 ORM 声明模型：

```py
from typing_extensions import Annotated
from typing import List
from typing import Optional
from sqlalchemy import ForeignKey
from sqlalchemy import String
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import MappedAsDataclass
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship

class Base(MappedAsDataclass, DeclarativeBase):
  """subclasses will be converted to dataclasses"""

intpk = Annotated[int, mapped_column(primary_key=True)]
str30 = Annotated[str, mapped_column(String(30))]
user_fk = Annotated[int, mapped_column(ForeignKey("user_account.id"))]

class User(Base):
    __tablename__ = "user_account"

    id: Mapped[intpk] = mapped_column(init=False)
    name: Mapped[str30]
    fullname: Mapped[Optional[str]] = mapped_column(default=None)
    addresses: Mapped[List["Address"]] = relationship(
        back_populates="user", default_factory=list
    )

class Address(Base):
    __tablename__ = "address"

    id: Mapped[intpk] = mapped_column(init=False)
    email_address: Mapped[str]
    user_id: Mapped[user_fk] = mapped_column(init=False)
    user: Mapped["User"] = relationship(back_populates="addresses", default=None)
```

上述映射已直接在每个映射类上使用了`@dataclasses.dataclass`装饰器，同时设置了声明性映射，内部将每个`dataclasses.field()`指令设置为所示。可以使用配置的位置参数创建`User` / `Address`结构：

```py
>>> u1 = User("username", fullname="full name", addresses=[Address("email@address")])
>>> u1
User(id=None, name='username', fullname='full name', addresses=[Address(id=None, email_address='email@address', user_id=None, user=...)])
```

另请参阅

声明性数据类映射

### SQL 表达式 / 语句 / 结果集类型

本节为 SQLAlchemy 的新 SQL 表达式类型方法提供了背景和示例，该方法从基本的`ColumnElement`构造扩展到 SQL 语句和结果集，并进入 ORM 映射的领域。

#### 原理与概述

提示

本节是一个架构讨论。直接跳转到 SQL 表达式类型化 - 示例来查看新的类型化是什么样子的。

在 [sqlalchemy2-stubs](https://github.com/sqlalchemy/sqlalchemy2-stubs) 中，SQL 表达式被类型化为[泛型](https://peps.python.org/pep-0484/#generics)，然后引用了一个 `TypeEngine` 对象，如 `Integer`、`DateTime` 或 `String` 作为它们的泛型参数（如 `Column[Integer]`）。这本身就是与原始 Dropbox [sqlalchemy-stubs](https://github.com/dropbox/sqlalchemy-stubs) 软件包所做的不同，其中 `Column` 及其基础构造直接是 Python 类型的泛型，比如 `int`、`datetime` 和 `str`。希望通过 `Integer` / `DateTime` / `String` 本身对 `int` / `datetime` / `str` 的泛型，可以维护两个信息级别，并且可以通过 `TypeEngine` 作为中间构造从列表达式中提取 Python 类型。然而，情况并非如此，因为[**PEP 484**](https://peps.python.org/pep-0484/) 没有足够丰富的功能集来实现这一点，缺乏诸如[高阶类型变量](https://github.com/python/typing/issues/548)之类的功能。

经过对当前[**PEP 484**](https://peps.python.org/pep-0484/)的能力的[深度评估](https://github.com/python/typing/discussions/999)，SQLAlchemy 2.0 在这一领域实现了 [sqlalchemy-stubs](https://github.com/dropbox/sqlalchemy-stubs) 的原始智慧，并直接将列表达式与 Python 类型进行了链接。这意味着，如果有 SQL 表达式到不同子类型的情况，比如 `Column(VARCHAR)` vs. `Column(Unicode)`，那么这两种 `String` 子类型的具体细节并不会随着类型一起传递，但在实践中，这通常不是问题，而且通常情况下，Python 类型是立即存在的，因为它代表了直接存储和接收该列的 Python 数据。

具体来说，这意味着像 `Column('id', Integer)` 这样的表达式被类型化为 `Column[int]`。 这允许建立一个可行的 SQLAlchemy 构造 -> Python 数据类型的流水线，而无需使用类型插件。 至关重要的是，它允许完全与 ORM 使用的 `select()` 和 `Row` 构造的范式进行交互，这些构造引用了 ORM 映射的类类型（例如包含用户映射实例的 `Row`，例如我们教程中使用的 `User` 和 `Address` 示例）。 虽然 Python 类型目前对于元组类型的定制支持非常有限（其中 [**PEP 646**](https://peps.python.org/pep-0646/)，第一个试图处理类似元组的对象的 pep，[在其功能上有意受到了限制](https://mail.python.org/archives/list/typing-sig@python.org/message/G2PNHRR32JMFD3JR7ACA2NDKWTDSEPUG/)，并且本身尚不适用于任意元组操作），但已经设计出了一种相当不错的方法，允许基本的 `select()` -> `Result` -> `Row` 类型功能，包括对 ORM 类的支持，在要将 `Row` 对象展开为单独的列条目时，添加了一个小的面向类型的访问器，允许各个 Python 值保持与其来源的 SQL 表达式相关联的 Python 类型（翻译：它有效）。

#### SQL 表达式类型化 - 示例

类型行为的简要介绍。 注释指示了在 [vscode](https://code.visualstudio.com/) 上悬停在代码上会看到什么（或者使用 [reveal_type()](https://mypy.readthedocs.io/en/latest/common_issues.html?highlight=reveal_type#reveal-type) 助手时，大致会显示什么类型工具）：

+   Python 中的简单类型赋给 SQL 表达式

    ```py
    # (variable) str_col: ColumnClause[str]
    str_col = column("a", String)

    # (variable) int_col: ColumnClause[int]
    int_col = column("a", Integer)

    # (variable) expr1: ColumnElement[str]
    expr1 = str_col + "x"

    # (variable) expr2: ColumnElement[int]
    expr2 = int_col + 10

    # (variable) expr3: ColumnElement[bool]
    expr3 = int_col == 15
    ```

+   分配给 `select()` 构造的单个 SQL 表达式，以及任何返回行的构造，包括返回行的 DML，例如 `Insert` 与 `Insert.returning()`，都被打包到一个 `Tuple[]` 类型中，该类型保留了每个元素的 Python 类型。

    ```py
    # (variable) stmt: Select[Tuple[str, int]]
    stmt = select(str_col, int_col)

    # (variable) stmt: ReturningInsert[Tuple[str, int]]
    ins_stmt = insert(table("t")).returning(str_col, int_col)
    ```

+   从任何返回行构造的`Tuple[]`类型，在调用`.execute()`方法时，传递到`Result`和`Row`。为了将`Row`对象解包为元组，`Row.tuple()`或`Row.t`访问器本质上将`Row`转换为相应的`Tuple[]`（尽管在运行时仍保持相同的`Row`对象）。

    ```py
    with engine.connect() as conn:
        # (variable) stmt: Select[Tuple[str, int]]
        stmt = select(str_col, int_col)

        # (variable) result: Result[Tuple[str, int]]
        result = conn.execute(stmt)

        # (variable) row: Row[Tuple[str, int]] | None
        row = result.first()

        if row is not None:
            # for typed tuple unpacking or indexed access,
            # use row.tuple() or row.t  (this is the small typing-oriented accessor)
            strval, intval = row.t

            # (variable) strval: str
            strval

            # (variable) intval: int
            intval
    ```

+   单列语句的标量值通过`Connection.scalar()`、`Result.scalars()`等方法执行正确操作。

    ```py
    # (variable) data: Sequence[str]
    data = connection.execute(select(str_col)).scalars().all()
    ```

+   上述对于返回行构造的支持与 ORM 映射类一起效果最佳，因为映射类可以列出其成员的具体类型。下面的示例设置了一个类，使用了新的类型感知语法，在下一节中描述：

    ```py
    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column

    class Base(DeclarativeBase):
        pass

    class User(Base):
        __tablename__ = "user_account"

        id: Mapped[int] = mapped_column(primary_key=True)
        name: Mapped[str]
        addresses: Mapped[List["Address"]] = relationship()

    class Address(Base):
        __tablename__ = "address"

        id: Mapped[int] = mapped_column(primary_key=True)
        email_address: Mapped[str]
        user_id = mapped_column(ForeignKey("user_account.id"))
    ```

    通过上述映射，属性被类型化，并且从语句一直表达到结果集：

    ```py
    with Session(engine) as session:
        # (variable) stmt: Select[Tuple[int, str]]
        stmt_1 = select(User.id, User.name)

        # (variable) result_1: Result[Tuple[int, str]]
        result_1 = session.execute(stmt_1)

        # (variable) intval: int
        # (variable) strval: str
        intval, strval = result_1.one().t
    ```

    映射类本身也是类型，并且表现方式相同，例如针对两个映射类的 SELECT：

    ```py
    with Session(engine) as session:
        # (variable) stmt: Select[Tuple[User, Address]]
        stmt_2 = select(User, Address).join_from(User, Address)

        # (variable) result_2: Result[Tuple[User, Address]]
        result_2 = session.execute(stmt_2)

        # (variable) user_obj: User
        # (variable) address_obj: Address
        user_obj, address_obj = result_2.one().t
    ```

    在选择映射类时，像`aliased`这样的构造也可以正常工作，保持原始映射类的列级属性以及语句期望的返回类型：

    ```py
    with Session(engine) as session:
        # this is in fact an Annotated type, but typing tools don't
        # generally display this

        # (variable) u1: Type[User]
        u1 = aliased(User)

        # (variable) stmt: Select[Tuple[User, User, str]]
        stmt = select(User, u1, User.name).filter(User.id == 5)

        # (variable) result: Result[Tuple[User, User, str]]
        result = session.execute(stmt)
    ```

+   核心表目前尚无良好的方法来维护通过`Table.c`访问器访问时`Column`对象的类型。

    由于`Table`被设置为类的实例，并且`Table.c`访问器通常通过名称动态访问`Column`对象，目前尚未为此建立类型化方法；需要一些替代语法。

+   ORM 类、标量等效果很好。

    选择 ORM 类作为标量或元组的典型用例都适用，无论是 2.0 还是 1.x 样式的查询，都能返回准确的类型，无论是独立的还是包含在适当的容器中，如 `Sequence[]`、`List[]` 或 `Iterator[]`：

    ```py
    # (variable) users1: Sequence[User]
    users1 = session.scalars(select(User)).all()

    # (variable) user: User
    user = session.query(User).one()

    # (variable) user_iter: Iterator[User]
    user_iter = iter(session.scalars(select(User)))
    ```

+   传统的 `Query` 也获得了元组类型化。

    `Query` 的类型支持远远超出了 [sqlalchemy-stubs](https://github.com/dropbox/sqlalchemy-stubs) 或 [sqlalchemy2-stubs](https://github.com/sqlalchemy/sqlalchemy2-stubs) 提供的范围，其中标量对象和元组类型的 `Query` 对象在大多数情况下将保留结果级的类型：

    ```py
    # (variable) q1: RowReturningQuery[Tuple[int, str]]
    q1 = session.query(User.id, User.name)

    # (variable) rows: List[Row[Tuple[int, str]]]
    rows = q1.all()

    # (variable) q2: Query[User]
    q2 = session.query(User)

    # (variable) users: List[User]
    users = q2.all()
    ```

#### 注意事项 - 必须卸载所有存根

关于类型支持的一个关键注意事项是 **必须卸载所有 SQLAlchemy 存根包** 才能使类型化工作。当对 Python 虚拟环境运行 [mypy](https://mypy.readthedocs.io/en/stable/) 时，只需卸载这些包。但是，目前 SQLAlchemy 存根包也是 [typeshed](https://github.com/python/typeshed) 的一部分，它本身被捆绑到一些类型工具中，例如 [Pylance](https://github.com/microsoft/pylance-release)，因此在某些情况下，可能需要定位这些包的文件并将其删除，以确保它们不会干扰新的类型化工作正常运行。

一旦 SQLAlchemy 2.0 正式发布，typeshed 将从其自己的存根源中删除 SQLAlchemy。

#### 原理与概述

提示

本节是一个架构讨论。要快速查看新的类型，请跳转到 SQL 表达式类型化 - 示例。

在 [sqlalchemy2-stubs](https://github.com/sqlalchemy/sqlalchemy2-stubs) 中，SQL 表达式被类型化为 [泛型](https://peps.python.org/pep-0484/#generics)，然后引用了 `TypeEngine` 对象，例如 `Integer`、`DateTime` 或 `String` 作为它们的泛型参数（如 `Column[Integer]`）。这本身是对原始 Dropbox [sqlalchemy-stubs](https://github.com/dropbox/sqlalchemy-stubs) 包的偏离，其中 `Column` 及其基本构造直接泛型化为 Python 类型，如 `int`、`datetime` 和 `str`。希望 `Integer` / `DateTime` / `String` 本身是对 `int` / `datetime` / `str` 泛型化的，就有可能保持两个层次的信息并能够通过 `TypeEngine` 作为中介构造从列表达式中提取 Python 类型。然而，事实并非如此，因为 [**PEP 484**](https://peps.python.org/pep-0484/) 实际上没有足够丰富的功能集使得这成为可行的，缺乏诸如 [更高种类的类型变量](https://github.com/python/typing/issues/548) 等能力。

经过对当前 [深度评估](https://github.com/python/typing/discussions/999)，SQLAlchemy 2.0 在这一领域实现了 [sqlalchemy-stubs](https://github.com/dropbox/sqlalchemy-stubs) 的原始智慧，并直接将列表达式与 Python 类型关联起来。这意味着如果有 SQL 表达式到不同子类型的情况，比如 `Column(VARCHAR)` 和 `Column(Unicode)`，那么这两个 `String` 子类型的具体细节并不会随着类型一起传递，但实际上这通常不是问题，通常更有用的是 Python 类型立即出现，因为它代表了直接存储和接收此列的 Python 数据。

具体来说，这意味着像 `Column('id', Integer)` 这样的表达式被类型化为 `Column[int]`。这允许建立一个可行的 SQLAlchemy 构造 -> Python 数据类型的管道，而无需使用类型插件。至关重要的是，它允许完全与 ORM 的范式进行互操作，即使用引用 ORM 映射类类型的 `select()` 和 `Row` 构造（例如包含用户映射实例的 `Row`，例如我们教程中使用的 `User` 和 `Address` 示例）。虽然 Python 类型当前对元组类型的自定义支持非常有限（其中 [**PEP 646**](https://peps.python.org/pep-0646/) 是第一个试图处理类似元组对象的 pep，但[其功能故意受到限制](https://mail.python.org/archives/list/typing-sig@python.org/message/G2PNHRR32JMFD3JR7ACA2NDKWTDSEPUG/)，本身尚不适用于任意元组操作），但已经设计出了一个相当不错的方法，允许基本的 `select()` -> `Result` -> `Row` 类型功能运行，包括对 ORM 类的支持，在将 `Row` 对象拆包为单独的列条目时，添加了一个小的面向类型的访问器，以便使得每个 Python 值都能保持与其来源的 SQL 表达式关联的 Python 类型（翻译：它可以正常工作）。

#### SQL 表达式类型 - 示例

对类型行为的简要介绍。注释指示了在 [vscode](https://code.visualstudio.com/) 中悬停在代码上会看到什么（或者在使用 [reveal_type()](https://mypy.readthedocs.io/en/latest/common_issues.html?highlight=reveal_type#reveal-type) 助手时大致会显示什么类型工具）：

+   将简单的 Python 类型分配给 SQL 表达式

    ```py
    # (variable) str_col: ColumnClause[str]
    str_col = column("a", String)

    # (variable) int_col: ColumnClause[int]
    int_col = column("a", Integer)

    # (variable) expr1: ColumnElement[str]
    expr1 = str_col + "x"

    # (variable) expr2: ColumnElement[int]
    expr2 = int_col + 10

    # (variable) expr3: ColumnElement[bool]
    expr3 = int_col == 15
    ```

+   将分配给 `select()` 构造的各个 SQL 表达式，以及任何返回行的构造，包括返回行的 DML，如 `Insert` 与 `Insert.returning()`，都打包成一个 `Tuple[]` 类型，其中保留了每个元素的 Python 类型。

    ```py
    # (variable) stmt: Select[Tuple[str, int]]
    stmt = select(str_col, int_col)

    # (variable) stmt: ReturningInsert[Tuple[str, int]]
    ins_stmt = insert(table("t")).returning(str_col, int_col)
    ```

+   任何行返回结构的 `Tuple[]` 类型，在调用 `.execute()` 方法时，都会传递到 `Result` 和 `Row`。为了将 `Row` 对象解包为元组，`Row.tuple()` 或 `Row.t` 访问器实质上将 `Row` 强制转换为相应的 `Tuple[]`（尽管在运行时仍然是相同的 `Row` 对象）。

    ```py
    with engine.connect() as conn:
        # (variable) stmt: Select[Tuple[str, int]]
        stmt = select(str_col, int_col)

        # (variable) result: Result[Tuple[str, int]]
        result = conn.execute(stmt)

        # (variable) row: Row[Tuple[str, int]] | None
        row = result.first()

        if row is not None:
            # for typed tuple unpacking or indexed access,
            # use row.tuple() or row.t  (this is the small typing-oriented accessor)
            strval, intval = row.t

            # (variable) strval: str
            strval

            # (variable) intval: int
            intval
    ```

+   单列语句的标量值通过诸如 `Connection.scalar()`、`Result.scalars()` 等方法执行正确操作。

    ```py
    # (variable) data: Sequence[str]
    data = connection.execute(select(str_col)).scalars().all()
    ```

+   上述对行返回结构的支持最适用于 ORM 映射类，因为映射类可以为其成员列出特定类型。下面的示例设置了一个类，使用了新的类型感知语法，在下一节中描述：

    ```py
    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column

    class Base(DeclarativeBase):
        pass

    class User(Base):
        __tablename__ = "user_account"

        id: Mapped[int] = mapped_column(primary_key=True)
        name: Mapped[str]
        addresses: Mapped[List["Address"]] = relationship()

    class Address(Base):
        __tablename__ = "address"

        id: Mapped[int] = mapped_column(primary_key=True)
        email_address: Mapped[str]
        user_id = mapped_column(ForeignKey("user_account.id"))
    ```

    通过上述映射，属性被类型化，并且从语句一直表达到结果集：

    ```py
    with Session(engine) as session:
        # (variable) stmt: Select[Tuple[int, str]]
        stmt_1 = select(User.id, User.name)

        # (variable) result_1: Result[Tuple[int, str]]
        result_1 = session.execute(stmt_1)

        # (variable) intval: int
        # (variable) strval: str
        intval, strval = result_1.one().t
    ```

    映射类本身也是类型，并且行为相同，例如对两个映射类进行 SELECT：

    ```py
    with Session(engine) as session:
        # (variable) stmt: Select[Tuple[User, Address]]
        stmt_2 = select(User, Address).join_from(User, Address)

        # (variable) result_2: Result[Tuple[User, Address]]
        result_2 = session.execute(stmt_2)

        # (variable) user_obj: User
        # (variable) address_obj: Address
        user_obj, address_obj = result_2.one().t
    ```

    当选择映射类时，像 `aliased` 这样的结构也可以正常工作，同时保持原始映射类的列级属性以及语句期望的返回类型：

    ```py
    with Session(engine) as session:
        # this is in fact an Annotated type, but typing tools don't
        # generally display this

        # (variable) u1: Type[User]
        u1 = aliased(User)

        # (variable) stmt: Select[Tuple[User, User, str]]
        stmt = select(User, u1, User.name).filter(User.id == 5)

        # (variable) result: Result[Tuple[User, User, str]]
        result = session.execute(stmt)
    ```

+   `Core Table` 目前尚无一个合适的方式来在通过 `Table.c` 访问时维护 `Column` 对象的类型。

    由于 `Table` 被设置为类的一个实例，并且 `Table.c` 访问器通常通过名称动态访问 `Column` 对象，目前尚无一种确定的类型化方法；需要一些替代语法。 

+   ORM 类、标量等都能很好地工作。

    选择 ORM 类的典型用例，作为标量或元组，都可以正常工作，无论是 2.0 还是 1.x 风格的查询，都可以得到确切的类型，无论是单独的还是包含在适当容器中，如`Sequence[]`，`List[]`或`Iterator[]`：

    ```py
    # (variable) users1: Sequence[User]
    users1 = session.scalars(select(User)).all()

    # (variable) user: User
    user = session.query(User).one()

    # (variable) user_iter: Iterator[User]
    user_iter = iter(session.scalars(select(User)))
    ```

+   传统的`Query`也获得了元组类型支持。

    对于`Query`的类型支持远远超出了[sqlalchemy-stubs](https://github.com/dropbox/sqlalchemy-stubs)或[sqlalchemy2-stubs](https://github.com/sqlalchemy/sqlalchemy2-stubs)提供的范围，其中标量对象以及元组类型的`Query`对象将保留大多数情况下的结果级别类型：

    ```py
    # (variable) q1: RowReturningQuery[Tuple[int, str]]
    q1 = session.query(User.id, User.name)

    # (variable) rows: List[Row[Tuple[int, str]]]
    rows = q1.all()

    # (variable) q2: Query[User]
    q2 = session.query(User)

    # (variable) users: List[User]
    users = q2.all()
    ```

#### 要点是 - 必须卸载所有存根

与类型支持相关的一个关键注意事项是**必须卸载所有 SQLAlchemy 存根包**才能使类型工作。当针对 Python 虚拟环境运行[mypy](https://mypy.readthedocs.io/en/stable/)时，只需卸载这些包。然而，SQLAlchemy 存根包目前也是[typeshed](https://github.com/python/typeshed)的一部分，它本身被捆绑到一些类型工具中，如[Pylance](https://github.com/microsoft/pylance-release)，因此在某些情况下可能需要定位这些包的文件并删除它们，以确保新的类型正确工作。

一旦 SQLAlchemy 2.0 发布为最终状态，typeshed 将从其自己的存根源中删除 SQLAlchemy。

### ORM 声明性模型

SQLAlchemy 1.4 引入了第一个使用[sqlalchemy2-stubs](https://github.com/sqlalchemy/sqlalchemy2-stubs)和 Mypy 插件的 SQLAlchemy 本机 ORM 类型支持。在 SQLAlchemy 2.0 中，Mypy 插件**仍然可用，并已更新以与 SQLAlchemy 2.0 的类型系统配合使用**。然而，现在应该将其视为**已弃用**，因为应用程序现在有一条直接的路径来采用新的类型支持，而不使用插件或存根。

#### 概述

新系统的基本方法是，当使用完全声明式模型（即不使用混合声明式或命令式配置，这些配置不变）时，映射的列声明首先通过检查每个属性声明左侧的类型注解（如果存在）在运行时派生。期望左手类型注释包含在`Mapped`泛型类型中，否则不认为该属性是映射属性。然后，属性声明可以引用右侧的`mapped_column()`构造，该构造用于提供关于要生成和映射的`Column`的附加核心级模式信息。如果左侧存在`Mapped`注解，则此右侧声明是可选的；如果左侧没有注解，则`mapped_column()`可以作为`Column`指令的精确替换使用，在这种情况下，它将为属性提供更准确（但不是精确）的类型行为，即使没有注解存在也是如此。

这种方法受到了 Python [dataclasses](https://docs.python.org/3/library/dataclasses.html) 的启发，它从左边开始注释，然后允许在右边进行可选的`dataclasses.field()`规范；与 dataclasses 方法的主要区别在于 SQLAlchemy 的方法是严格的**选择加入**，其中使用 `Column` 的现有映射而没有任何类型注释的映射将继续像以往一样工作，而 `mapped_column()` 构造可以直接替换为 `Column` 而不需要任何显式的类型注释。只有在需要存在确切的属性级 Python 类型时，才需要使用 `Mapped` 的显式注释。这些注释可以根据需要，按属性基础在那些有用的特定类型的属性上使用；使用 `mapped_column()` 的未注释的属性将在实例级别上被类型为`Any`。

#### 迁移现有映射

转向新的 ORM 方法开始时更加冗长，但随着可用的新功能的充分利用，它变得比以前更加简洁。以下步骤详细介绍了一个典型的转换，然后继续说明了一些更多的选项。

##### 第一步 - `declarative_base()`被`DeclarativeBase`所取代。

在 Python 类型中观察到的一个限制是似乎没有能力从一个函数动态生成一个类，然后被类型工具理解为新类的基础。为了解决这个问题而不使用插件，通常调用`declarative_base()`可以被替换为使用`DeclarativeBase`类，它产生与通常相同的`Base`对象，只是类型工具理解它：

```py
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    pass
```

##### 第二步 - 使用`mapped_column()`替换`Column`的声明性使用。

`mapped_column()`是一个 ORM 类型感知构造，可以直接替换为`Column`的使用。给定一个 1.x 风格的映射如下：

```py
from sqlalchemy import Column
from sqlalchemy.orm import relationship
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "user_account"

    id = Column(Integer, primary_key=True)
    name = Column(String(30), nullable=False)
    fullname = Column(String)
    addresses = relationship("Address", back_populates="user")

class Address(Base):
    __tablename__ = "address"

    id = Column(Integer, primary_key=True)
    email_address = Column(String, nullable=False)
    user_id = Column(ForeignKey("user_account.id"), nullable=False)
    user = relationship("User", back_populates="addresses")
```

我们用`mapped_column()`替换`Column`；不需要更改任何参数：

```py
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "user_account"

    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(30), nullable=False)
    fullname = mapped_column(String)
    addresses = relationship("Address", back_populates="user")

class Address(Base):
    __tablename__ = "address"

    id = mapped_column(Integer, primary_key=True)
    email_address = mapped_column(String, nullable=False)
    user_id = mapped_column(ForeignKey("user_account.id"), nullable=False)
    user = relationship("User", back_populates="addresses")
```

上面的各个列**尚未使用 Python 类型进行类型化**，而是被类型化为`Mapped[Any]`；这是因为我们可以声明任何列为`Optional`或非`Optional`，并且没有办法在不引起类型错误的情况下进行“猜测”。

然而，在这一步，我们上面的映射已经为所有属性设置了适当的描述符类型，并且可以用于查询以及实例级别的操作，所有这些都将**通过 mypy 的--strict 模式**而无需插件。

##### 第三步 - 根据需要使用`Mapped`应用精确的 Python 类型。

这可以用于所有需要精确类型的属性；可以跳过那些可以保留为`Any`的属性。为了上下文，我们还演示了在`relationship()`中应用精确类型时使用`Mapped`。在这个中间步骤中，映射会更加冗长，但是通过熟练掌握，这一步可以与后续步骤结合起来更直接地更新映射：

```py
from typing import List
from typing import Optional
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "user_account"

    id: Mapped[int] = mapped_column(Integer, primary_key=True)
    name: Mapped[str] = mapped_column(String(30), nullable=False)
    fullname: Mapped[Optional[str]] = mapped_column(String)
    addresses: Mapped[List["Address"]] = relationship("Address", back_populates="user")

class Address(Base):
    __tablename__ = "address"

    id: Mapped[int] = mapped_column(Integer, primary_key=True)
    email_address: Mapped[str] = mapped_column(String, nullable=False)
    user_id: Mapped[int] = mapped_column(ForeignKey("user_account.id"), nullable=False)
    user: Mapped["User"] = relationship("User", back_populates="addresses")
```

此时，我们的 ORM 映射已经完全类型化，并将生成精确类型的`select()`、`Query`和`Result`构造。现在我们可以继续简化映射声明中的冗余部分。

##### 第四步 - 删除不再需要的`mapped_column()`指令

所有 `nullable` 参数都可以使用 `Optional[]` 隐含; 在没有 `Optional[]` 的情况下，`nullable` 默认为 `False`。所有没有参数的 SQL 类型，如 `Integer` 和 `String`，可以单独用 Python 注释表示。一个没有参数的 `mapped_column()` 指令可以完全移除。`relationship()` 现在从左手注释派生其类，支持正向引用（因为 `relationship()` 已经支持基于字符串的正向引用十年了 ;) ）：

```py
from typing import List
from typing import Optional
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "user_account"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(30))
    fullname: Mapped[Optional[str]]
    addresses: Mapped[List["Address"]] = relationship(back_populates="user")

class Address(Base):
    __tablename__ = "address"

    id: Mapped[int] = mapped_column(primary_key=True)
    email_address: Mapped[str]
    user_id: Mapped[int] = mapped_column(ForeignKey("user_account.id"))
    user: Mapped["User"] = relationship(back_populates="addresses")
```

##### 第五步 - 利用 pep-593 `Annotated` 将常见指令打包成类型

这是一个全新的功能，提供了一种替代或补充的方法，作为声明性混入的一种方式，以提供面向类型的配置，并且在大多数情况下取代了 `declared_attr` 装饰函数的需要。

首先，Declarative 映射允许将 Python 类型映射到 SQL 类型，例如将 `str` 映射到 `String`，通过使用 `registry.type_annotation_map` 进行自定义。使用 [**PEP 593**](https://peps.python.org/pep-0593/) `Annotated` 允许我们创建特定 Python 类型的变体，以便可以使用相同的类型，如 `str`，每个都提供 `String` ���变体，如下所示，使用 `Annotated` `str`，称为 `str50` 将指示 `String(50)`：

```py
from typing_extensions import Annotated
from sqlalchemy.orm import DeclarativeBase

str50 = Annotated[str, 50]

# declarative base with a type-level override, using a type that is
# expected to be used in multiple places
class Base(DeclarativeBase):
    type_annotation_map = {
        str50: String(50),
    }
```

其次，如果使用 `Annotated[]`，Declarative 将从左手类型中提取完整的 `mapped_column()` 定义，通过将任何参数传递给 `Annotated[]` 构造 `mapped_column()` 构造（感谢 [@adriangb01](https://twitter.com/adriangb01/status/1532841383647657988) 展示这个想法）。这种能力可能在未来的版本中扩展到包括 `relationship()`、`composite()` 和其他构造，但目前仅限于 `mapped_column()`。下面的示例除了我们的 `str50` 示例外，还添加了额外的 `Annotated` 类型，以说明这个特性：

```py
from typing_extensions import Annotated
from typing import List
from typing import Optional
from sqlalchemy import ForeignKey
from sqlalchemy import String
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship

# declarative base from previous example
str50 = Annotated[str, 50]

class Base(DeclarativeBase):
    type_annotation_map = {
        str50: String(50),
    }

# set up mapped_column() overrides, using whole column styles that are
# expected to be used in multiple places
intpk = Annotated[int, mapped_column(primary_key=True)]
user_fk = Annotated[int, mapped_column(ForeignKey("user_account.id"))]

class User(Base):
    __tablename__ = "user_account"

    id: Mapped[intpk]
    name: Mapped[str50]
    fullname: Mapped[Optional[str]]
    addresses: Mapped[List["Address"]] = relationship(back_populates="user")

class Address(Base):
    __tablename__ = "address"

    id: Mapped[intpk]
    email_address: Mapped[str50]
    user_id: Mapped[user_fk]
    user: Mapped["User"] = relationship(back_populates="addresses")
```

上面，使用`Mapped[str50]`、`Mapped[intpk]`或`Mapped[user_fk]`映射的列直接从`registry.type_annotation_map`以及`Annotated`构造中绘制，以便重新使用预先建立的类型和列配置。

##### 可选步骤 - 将映射类转换为[dataclasses](https://docs.python.org/3/library/dataclasses.html)

我们可以将映射类转换为[dataclasses](https://docs.python.org/3/library/dataclasses.html)，其中一个关键优势是我们可以构建一个严格类型的`__init__()`方法，具有显式的位置、关键字参数和默认参数，更不用说我们还可以免费获取`__str__()`和`__repr__()`等方法。下一节作为 ORM 模型映射的 dataclasses 的本机支持进一步说明了上述模型的转换。 

##### 从第 3 步开始支持键入。

通过上面的示例，从“第 3 步”开始的任何示例都将包括模型属性的类型，并将传播到`select()`、`Query`和`Row` 对象：

```py
# (variable) stmt: Select[Tuple[int, str]]
stmt = select(User.id, User.name)

with Session(e) as sess:
    for row in sess.execute(stmt):
        # (variable) row: Row[Tuple[int, str]]
        print(row)

    # (variable) users: Sequence[User]
    users = sess.scalars(select(User)).all()

    # (variable) users_legacy: List[User]
    users_legacy = sess.query(User).all()
```

请参阅

使用 mapped_column() 的声明性表 - 更新了声明性文档以声明性生成和映射 `Table` 列。

#### 概述

新系统的基本方法是，当使用完全声明式模型（即不使用混合声明式或命令式配置，这些配置不变）时，映射列声明首先在运行时通过检查每个属性声明左侧的类型注释来推导，如果存在的话。左手类型注释应该包含在`Mapped`泛型类型中，否则该属性不被视为映射属性。然后属性声明可以引用右侧的`mapped_column()`构造，用于提供有关要生成和映射的`Column`的附加核心级模式信息。如果左侧存在`Mapped`注释，则此右侧声明是可选的；如果左侧没有注释，则`mapped_column()`可以用作`Column`指令的精确替代，其中它将提供更准确（但不精确）的属性类型行为，即使没有注释存在。

这种方法受到了 Python [dataclasses](https://docs.python.org/3/library/dataclasses.html)方法的启发，它从左边的注解开始，然后允许在右边进行可选的`dataclasses.field()`规范；与 dataclasses 方法的主要区别在于 SQLAlchemy 方法是严格的**选择性**，其中使用`Column`的现有映射不带任何类型注释仍然像以往一样工作，并且`mapped_column()`构造可以直接替换`Column`而不需要任何显式的类型注释。只有在确切的属性级 Python 类型存在时，才需要使用具有`Mapped`的显式注释。这些注释可以根据需要在每个属性的基础上使用，对于那些特定类型有帮助的属性；使用`mapped_column()`的未注释属性将在实例级别被标记为`Any`。

#### 迁移现有映射

切换到新的 ORM 方法开始时可能更加冗长，但随着可用的新功能的充分利用，它变得比以前更加简洁。以下步骤详细说明了一个典型的过渡，然后继续说明了一些更多的选项。

##### 第一步 - `declarative_base()`被`DeclarativeBase`所取代。

在 Python 类型中观察到的一个限制是似乎没有能力从一个函数中动态生成一个类，然后让类型工具将其理解为新类的基类。为了解决这个问题而不使用插件，通常对`declarative_base()`的调用可以被替换为使用`DeclarativeBase`类，它生成与通常相同的`Base`对象，只是类型工具理解它：

```py
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    pass
```

##### 第二步 - 将`Column`的声明性使用替换为`mapped_column()`

`mapped_column()`是一个 ORM 类型感知的构造，可以直接替换为 `Column` 的使用。给定一个 1.x 风格的映射，如下所示：

```py
from sqlalchemy import Column
from sqlalchemy.orm import relationship
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "user_account"

    id = Column(Integer, primary_key=True)
    name = Column(String(30), nullable=False)
    fullname = Column(String)
    addresses = relationship("Address", back_populates="user")

class Address(Base):
    __tablename__ = "address"

    id = Column(Integer, primary_key=True)
    email_address = Column(String, nullable=False)
    user_id = Column(ForeignKey("user_account.id"), nullable=False)
    user = relationship("User", back_populates="addresses")
```

我们用`mapped_column()`替换了`Column`；不需要更改任何参数：

```py
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "user_account"

    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(30), nullable=False)
    fullname = mapped_column(String)
    addresses = relationship("Address", back_populates="user")

class Address(Base):
    __tablename__ = "address"

    id = mapped_column(Integer, primary_key=True)
    email_address = mapped_column(String, nullable=False)
    user_id = mapped_column(ForeignKey("user_account.id"), nullable=False)
    user = relationship("User", back_populates="addresses")
```

上述各列目前**尚未使用 Python 类型进行类型化**，而是以`Mapped[Any]`类型进行类型化；这是因为我们可以声明任何列是否为`Optional`，而且在明确类型时不可能有“猜测”，否则会在明确类型时导致类型错误。

但是，在这一步骤中，我们上述的映射已经为所有属性设置了适当的描述符类型，并且可以用于查询以及实例级别的操作，所有这些操作都可以在不使用插件的情况下**通过 mypy –strict 模式**。

##### 第三步 - 根据需要使用`Mapped`应用精确的 Python 类型。

所有需要精确类型的属性都可以进行此操作；可以略过那些可以保持`Any`的属性。为了上下文，我们还演示了在其中应用精确类型的情况下使用`Mapped`的情况，用于`relationship()`。在这个临时步骤中的映射会更加冗长，但是熟练掌握后，这一步可以与后续步骤结合起来直接更新映射：

```py
from typing import List
from typing import Optional
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "user_account"

    id: Mapped[int] = mapped_column(Integer, primary_key=True)
    name: Mapped[str] = mapped_column(String(30), nullable=False)
    fullname: Mapped[Optional[str]] = mapped_column(String)
    addresses: Mapped[List["Address"]] = relationship("Address", back_populates="user")

class Address(Base):
    __tablename__ = "address"

    id: Mapped[int] = mapped_column(Integer, primary_key=True)
    email_address: Mapped[str] = mapped_column(String, nullable=False)
    user_id: Mapped[int] = mapped_column(ForeignKey("user_account.id"), nullable=False)
    user: Mapped["User"] = relationship("User", back_populates="addresses")
```

此时，我们的 ORM 映射已经完全类型化，并将生成精确类型的`select()`、`Query`和`Result` 构造。现在我们可以继续简化映射声明中的冗余部分。

##### 第四步 - 移除不再需要的`mapped_column()`指令。

所有`nullable`参数都可以使用`Optional[]`来隐含表示；在没有`Optional[]`的情况下，`nullable`默认为`False`。所有没有参数的 SQL 类型，如`Integer`和`String`，都可以单独用 Python 注释来表示。一个不带参数的`mapped_column()`指令可以完全删除。`relationship()`现在会从左手注释派生其类，支持正向引用（正如`relationship()`已经支持字符串型正向引用十年一样；）：

```py
from typing import List
from typing import Optional
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "user_account"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(30))
    fullname: Mapped[Optional[str]]
    addresses: Mapped[List["Address"]] = relationship(back_populates="user")

class Address(Base):
    __tablename__ = "address"

    id: Mapped[int] = mapped_column(primary_key=True)
    email_address: Mapped[str]
    user_id: Mapped[int] = mapped_column(ForeignKey("user_account.id"))
    user: Mapped["User"] = relationship(back_populates="addresses")
```

##### 第五步 - 利用 pep-593 `Annotated`将常见指令打包成类型

这是一个激进的新功能，提供了一个替代方案，或者说是补充性方法，用于提供类型定向配置，还可以在大多数情况下替代`declared_attr`修饰的函数的需求。

首先，Declarative 映射允许将 Python 类型映射到 SQL 类型，例如`str`到`String`的自定义，使用`registry.type_annotation_map`。使用[**PEP 593**](https://peps.python.org/pep-0593/) `Annotated`允许我们创建特定 Python 类型的变体，以便使用相同的类型，例如`str`，为其提供`String`的变体，如下所示，使用`Annotated` `str`称为`str50`将指示`String(50)`：

```py
from typing_extensions import Annotated
from sqlalchemy.orm import DeclarativeBase

str50 = Annotated[str, 50]

# declarative base with a type-level override, using a type that is
# expected to be used in multiple places
class Base(DeclarativeBase):
    type_annotation_map = {
        str50: String(50),
    }
```

其次，如果使用`Annotated[]`，Declarative 将从左侧类型中提取完整的`mapped_column()`定义，通过将`mapped_column()`构造传递给`Annotated[]`构造（感谢[@adriangb01](https://twitter.com/adriangb01/status/1532841383647657988)说明了这个想法）。此功能可能在将来的版本中扩展到还包括`relationship()`、`composite()`和其他构造，但目前仅限于`mapped_column()`。下面的示例除了我们的`str50`示例之外，还添加了其他额外的`Annotated`类型，以说明此功能：

```py
from typing_extensions import Annotated
from typing import List
from typing import Optional
from sqlalchemy import ForeignKey
from sqlalchemy import String
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship

# declarative base from previous example
str50 = Annotated[str, 50]

class Base(DeclarativeBase):
    type_annotation_map = {
        str50: String(50),
    }

# set up mapped_column() overrides, using whole column styles that are
# expected to be used in multiple places
intpk = Annotated[int, mapped_column(primary_key=True)]
user_fk = Annotated[int, mapped_column(ForeignKey("user_account.id"))]

class User(Base):
    __tablename__ = "user_account"

    id: Mapped[intpk]
    name: Mapped[str50]
    fullname: Mapped[Optional[str]]
    addresses: Mapped[List["Address"]] = relationship(back_populates="user")

class Address(Base):
    __tablename__ = "address"

    id: Mapped[intpk]
    email_address: Mapped[str50]
    user_id: Mapped[user_fk]
    user: Mapped["User"] = relationship(back_populates="addresses")
```

上面，使用`Mapped[str50]`、`Mapped[intpk]`或`Mapped[user_fk]`映射的列直接从`registry.type_annotation_map`和`Annotated`构造中汲取，以便重新使用预先建立的类型和列配置。

##### 可选步骤 - 将映射类转换为[dataclasses](https://docs.python.org/3/library/dataclasses.html)

我们可以将映射类转换为[dataclasses](https://docs.python.org/3/library/dataclasses.html)，其中一个关键优势是我们可以构建一个严格类型化的`__init__()`方法，具有显式位置参数、仅关键字参数和默认参数，更不用说我们免费获取了`__str__()`和`__repr__()`等方法。 下一节 Native Support for Dataclasses Mapped as ORM Models 进一步说明了上述模型的转换。

##### 从第 3 步开始支持类型注解

配上上述示例，从“步骤 3”开始的任何示例都会包括模型的属性类型，并将通过`select()`，`Query`和`Row`对象进行填充：

```py
# (variable) stmt: Select[Tuple[int, str]]
stmt = select(User.id, User.name)

with Session(e) as sess:
    for row in sess.execute(stmt):
        # (variable) row: Row[Tuple[int, str]]
        print(row)

    # (variable) users: Sequence[User]
    users = sess.scalars(select(User)).all()

    # (variable) users_legacy: List[User]
    users_legacy = sess.query(User).all()
```

请参见

使用 mapped_column()声明式表 - 更新了声明式文档以声明式生成和映射`Table`列。

##### 第一步 - `declarative_base()`已被`DeclarativeBase`取代。

在 Python 类型注解中观察到的一个限制是似乎没有能力从函数中动态生成类，然后将其理解为新类的基础的功能。 要解决此问题而不使用插件，通常调用`declarative_base()`的方法可以替换为使用`DeclarativeBase`类，该类产生与通常相同的`Base`对象，只是类型工具将其理解为：

```py
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    pass
```

##### 第二步 - 将`Column`的声明式用法替换为`mapped_column()`

`mapped_column()` 是一个 ORM 类型感知的构造，可以直接替换为 `Column` 的使用。在 1.x 风格的映射中如下所示：

```py
from sqlalchemy import Column
from sqlalchemy.orm import relationship
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "user_account"

    id = Column(Integer, primary_key=True)
    name = Column(String(30), nullable=False)
    fullname = Column(String)
    addresses = relationship("Address", back_populates="user")

class Address(Base):
    __tablename__ = "address"

    id = Column(Integer, primary_key=True)
    email_address = Column(String, nullable=False)
    user_id = Column(ForeignKey("user_account.id"), nullable=False)
    user = relationship("User", back_populates="addresses")
```

我们用 `mapped_column()` 替换了 `Column`；不需要更改任何参数：

```py
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "user_account"

    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(30), nullable=False)
    fullname = mapped_column(String)
    addresses = relationship("Address", back_populates="user")

class Address(Base):
    __tablename__ = "address"

    id = mapped_column(Integer, primary_key=True)
    email_address = mapped_column(String, nullable=False)
    user_id = mapped_column(ForeignKey("user_account.id"), nullable=False)
    user = relationship("User", back_populates="addresses")
```

上述各列目前**尚未使用 Python 类型进行类型化**，而是被类型化为 `Mapped[Any]`；这是因为我们可以声明任何列是可选的或不可选的，而且在我们显式类型化时，没有办法有一个“猜测”的方式不会在类型化时导致类型错误。

然而，在此步骤中，我们上述的映射已经为所有属性设置了适当的 描述符 类型，并且可以在查询中使用以及进行实例级别的操作，所有这些操作都将**在不使用插件的情况下通过 mypy –strict 模式**。

##### 步骤三 - 使用 `Mapped` 需要的精确 Python 类型。

对于希望精确类型化的所有属性，都可以执行此操作；对于希望保留为 `Any` 的属性可以跳过。为了提供背景信息，我们还展示了将 `Mapped` 用于 `relationship()` 的情况，我们在此应用了精确类型。在此过渡阶段内的映射将更为冗长，但是随着熟练程度的提高，可以将此步骤与后续步骤结合起来更直接地更新映射：

```py
from typing import List
from typing import Optional
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "user_account"

    id: Mapped[int] = mapped_column(Integer, primary_key=True)
    name: Mapped[str] = mapped_column(String(30), nullable=False)
    fullname: Mapped[Optional[str]] = mapped_column(String)
    addresses: Mapped[List["Address"]] = relationship("Address", back_populates="user")

class Address(Base):
    __tablename__ = "address"

    id: Mapped[int] = mapped_column(Integer, primary_key=True)
    email_address: Mapped[str] = mapped_column(String, nullable=False)
    user_id: Mapped[int] = mapped_column(ForeignKey("user_account.id"), nullable=False)
    user: Mapped["User"] = relationship("User", back_populates="addresses")
```

此时，我们的 ORM 映射已完全类型化，并将生成精确类型化的 `select()`、`Query` 和 `Result` 构造。现在我们可以开始减少映射声明中的冗余。

##### 步骤四 - 移除不再需要的 `mapped_column()` 指令。

所有`nullable`参数都可以使用`Optional[]`来隐含；在没有`Optional[]`的情况下，`nullable`默认为`False`。所有没有参数的 SQL 类型，如`Integer`和`String`，可以仅用 Python 注释表示。没有参数的`mapped_column()`指令可以完全删除。`relationship()`现在从左侧注释派生其类，还支持前向引用（就像`relationship()`已经支持基于字符串的前向引用十年一样 ;)）：

```py
from typing import List
from typing import Optional
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "user_account"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(30))
    fullname: Mapped[Optional[str]]
    addresses: Mapped[List["Address"]] = relationship(back_populates="user")

class Address(Base):
    __tablename__ = "address"

    id: Mapped[int] = mapped_column(primary_key=True)
    email_address: Mapped[str]
    user_id: Mapped[int] = mapped_column(ForeignKey("user_account.id"))
    user: Mapped["User"] = relationship(back_populates="addresses")
```

##### 第五步 - 利用 pep-593 的`Annotated`将常见指令打包成类型

这是一个全新的功能，提供了一种替代或补充方法，作为提供面向类型的配置的手段，也替代了大多数情况下对`declared_attr`装饰函数的需求。

首先，Declarative 映射允许将 Python 类型映射到 SQL 类型，例如将`str`定制为`String`，使用`registry.type_annotation_map`进行自定义。使用[**PEP 593**](https://peps.python.org/pep-0593/)的`Annotated`允许我们创建特定 Python 类型的变体，以便可以使用相同的类型，例如`str`，每个都提供`String`的变体，如下所示，使用`Annotated` `str`称为`str50`将指示`String(50)`：

```py
from typing_extensions import Annotated
from sqlalchemy.orm import DeclarativeBase

str50 = Annotated[str, 50]

# declarative base with a type-level override, using a type that is
# expected to be used in multiple places
class Base(DeclarativeBase):
    type_annotation_map = {
        str50: String(50),
    }
```

第二，如果使用`Annotated[]`，Declarative 将从左侧类型中提取完整的`mapped_column()`定义，方法是将`mapped_column()`构造作为任何参数传递给`Annotated[]`构造（感谢[@adriangb01](https://twitter.com/adriangb01/status/1532841383647657988)提出这个想法）。未来的版本可能会扩展此功能，以包括`relationship()`、`composite()`和其他构造，但目前仅限于`mapped_column()`。下面的示例除了我们的`str50`示例外，还添加了额外的`Annotated`类型，以说明此功能：

```py
from typing_extensions import Annotated
from typing import List
from typing import Optional
from sqlalchemy import ForeignKey
from sqlalchemy import String
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship

# declarative base from previous example
str50 = Annotated[str, 50]

class Base(DeclarativeBase):
    type_annotation_map = {
        str50: String(50),
    }

# set up mapped_column() overrides, using whole column styles that are
# expected to be used in multiple places
intpk = Annotated[int, mapped_column(primary_key=True)]
user_fk = Annotated[int, mapped_column(ForeignKey("user_account.id"))]

class User(Base):
    __tablename__ = "user_account"

    id: Mapped[intpk]
    name: Mapped[str50]
    fullname: Mapped[Optional[str]]
    addresses: Mapped[List["Address"]] = relationship(back_populates="user")

class Address(Base):
    __tablename__ = "address"

    id: Mapped[intpk]
    email_address: Mapped[str50]
    user_id: Mapped[user_fk]
    user: Mapped["User"] = relationship(back_populates="addresses")
```

以上，使用 `Mapped[str50]`、`Mapped[intpk]` 或 `Mapped[user_fk]` 映射的列会直接从 `registry.type_annotation_map` 和 `Annotated` 结构中获取，以便重新使用预先建立的类型化和列配置。

##### 可选步骤 - 将映射类转换为 [数据类](https://docs.python.org/3/library/dataclasses.html)

我们可以将映射类转换为 [数据类](https://docs.python.org/3/library/dataclasses.html)，其中一个关键优势是，我们可以构建一个严格类型化的 `__init__()` 方法，具有显式的位置、关键字和默认参数，更不用说我们可以免费获得 `__str__()` 和 `__repr__()` 等方法了。下一节 数据类作为 ORM 模型的本地支持 进一步说明了以上模型的转换。 

##### 从第三步开始支持类型化

通过以上示例，从“第三步”开始的任何示例都将包括模型属性是经过类型化的，并将通过到 `select()`、`Query` 和 `Row` 对象：

```py
# (variable) stmt: Select[Tuple[int, str]]
stmt = select(User.id, User.name)

with Session(e) as sess:
    for row in sess.execute(stmt):
        # (variable) row: Row[Tuple[int, str]]
        print(row)

    # (variable) users: Sequence[User]
    users = sess.scalars(select(User)).all()

    # (variable) users_legacy: List[User]
    users_legacy = sess.query(User).all()
```

另请参见

使用 mapped_column() 的声明式表 - 更新了声明式文档以声明性生成和映射 `Table` 列。

### 使用传统 Mypy 类型化模型

使用 Mypy 插件 的 SQLAlchemy 应用，在显式注释中不使用 `Mapped` 的情况下，会在新系统下产生错误，因为这样的注释在使用 `relationship()` 等结构时被标记为错误。

章节 2.0 迁移第六步 - 向显式类型化的 ORM 模型添加 __allow_unmapped__ 说明了如何临时禁用对使用显式注释的遗留 ORM 模型引发的错误。

另请参见

2.0 迁移第六步 - 向显式类型化的 ORM 模型添加 __allow_unmapped__

### 数据类作为 ORM 模型的本地支持

上面介绍的新的 ORM 声明性特性在 ORM 声明模型中引入了新的`mapped_column()`构造，并且演示了以类型为中心的映射，可选地使用[**PEP 593**](https://peps.python.org/pep-0593/) `Annotated`。我们可以通过将其与 Python 的[dataclasses](https://docs.python.org/3/library/dataclasses.html)集成，进一步推进映射。这个新特性通过[**PEP 681**](https://peps.python.org/pep-0681/)实现，允许类型检查器识别符合数据类兼容性的类，或者是完全数据类，但是通过替代 API 声明的类。

使用数据类特性，映射类获得了一个支持位置参数以及可选关键字参数的可定制默认值的`__init__()`方法。正如之前提到的，数据类还生成许多有用的方法，如`__str__()`，`__eq__()`。数据类序列化方法，如[dataclasses.asdict()](https://docs.python.org/3/library/dataclasses.html#dataclasses.asdict)和[dataclasses.astuple()](https://docs.python.org/3/library/dataclasses.html#dataclasses.astuple)也可以使用，但目前不支持自引用结构，这使得它们对于具有双向关系的映射不太可行。

SQLAlchemy 当前的集成方法将用户定义的类转换为**真实数据类**以提供运行时功能；该特性利用了 SQLAlchemy 1.4 中引入的现有数据类功能，在 Python 数据类，支持 attrs w/声明性，命令式映射中介绍了一个等效的运行时映射，具有完全集成的配置样式，这样做比以前的方法更正确地类型化。

为了支持符合[**PEP 681**](https://peps.python.org/pep-0681/)的数据类，ORM 构造如`mapped_column()`和`relationship()`接受额外的[**PEP 681**](https://peps.python.org/pep-0681/)参数`init`、`default`和`default_factory`，这些参数会传递到数据类创建过程中。这些参数目前必须存在于右侧的显式指令中，就像它们将与`dataclasses.field()`一起使用一样；目前它们不能作为左侧`Annotated`构造中的局部变量存在。为了支持方便使用`Annotated`同时仍支持数据类配置，`mapped_column()`可以将右侧的最小一组参数与左侧`Annotated`构造中的现有`mapped_column()`构造合并，以便保持大部分的简洁性，如下所示。

为了使用类继承启用数据类，我们使用`MappedAsDataclass` mixin，可以直接在每个类上使用，也可以在`Base`类上使用，如下所示，我们进一步修改了来自 ORM 声明性模型“步骤 5”的示例映射：

```py
from typing_extensions import Annotated
from typing import List
from typing import Optional
from sqlalchemy import ForeignKey
from sqlalchemy import String
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import MappedAsDataclass
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship

class Base(MappedAsDataclass, DeclarativeBase):
  """subclasses will be converted to dataclasses"""

intpk = Annotated[int, mapped_column(primary_key=True)]
str30 = Annotated[str, mapped_column(String(30))]
user_fk = Annotated[int, mapped_column(ForeignKey("user_account.id"))]

class User(Base):
    __tablename__ = "user_account"

    id: Mapped[intpk] = mapped_column(init=False)
    name: Mapped[str30]
    fullname: Mapped[Optional[str]] = mapped_column(default=None)
    addresses: Mapped[List["Address"]] = relationship(
        back_populates="user", default_factory=list
    )

class Address(Base):
    __tablename__ = "address"

    id: Mapped[intpk] = mapped_column(init=False)
    email_address: Mapped[str]
    user_id: Mapped[user_fk] = mapped_column(init=False)
    user: Mapped["User"] = relationship(back_populates="addresses", default=None)
```

上述映射在设置声明性映射的同时直接在每个映射类上使用了`@dataclasses.dataclass`装饰器，内部设置了每个`dataclasses.field()`指令，如所示。使用位置参数可以配置`User` / `Address`结构：

```py
>>> u1 = User("username", fullname="full name", addresses=[Address("email@address")])
>>> u1
User(id=None, name='username', fullname='full name', addresses=[Address(id=None, email_address='email@address', user_id=None, user=...)])
```

另请参阅

声明式数据类映射

## 优化的 ORM 批量插入现在已经针对除 MySQL 之外的所有后端实现了

1.4 系列中引入的显著性能改进，如 ORM Batch inserts with psycopg2 now batch statements with RETURNING in most cases 中所述，现已普遍适用于所有支持 RETURNING 的后端，除了 MySQL：SQLite、MariaDB、PostgreSQL（所有驱动程序）和 Oracle；SQL Server 有支持，但在版本 2.0.9 中暂时禁用 [[1]](#id2)。虽然原始功能对于 psycopg2 驱动程序至关重要，否则在使用 `cursor.executemany()` 时存在严重的性能问题，但对于其他 PostgreSQL 驱动程序，如 asyncpg，此更改也至关重要，因为在使用 RETURNING 时，单语句 INSERT 语句仍然不可接受地缓慢，以及在使用 SQL Server 时，无论是否使用 RETURNING，插入语句的 executemany 速度也似乎非常缓慢。

新功能的性能为每个驱动程序的 INSERT ORM 对象的性能提供了几乎横跨所有板块的数量级的性能提升，如下表所示，大多数情况下特定于 RETURNING 的使用，通常不支持 executemany()。

psycopg2 的“快速执行助手”方法包括将具有单个参数集的 INSERT..RETURNING 语句转换为一个插入多个参数集的单个语句，使用多个“VALUES…”子句，以便它可以一次容纳多个参数集。然后，参数集通常被批处理成一组 1000 或类似的参数集，以便没有单个 INSERT 语句过于大，并且 INSERT 语句然后针对每个参数批次调用，而不是针对每个单独的参数集。通过 RETURNING 返回主键值和服务器默认值，这仍然会在每个语句执行时使用 `cursor.execute()` 调用，而不是 `cursor.executemany()`。

这允许一次插入许多行，同时还能够返回新生成的主键值以及 SQL 和服务器默认值。历史上，SQLAlchemy 一直需要对每个参数集调用一条语句，因为它依赖于诸如 `cursor.lastrowid` 等 Python DBAPI 功能，这些功能不支持多行。

由于现在大多数数据库都提供了 RETURNING（尤其是 MySQL 是一个明显的例外，因为 MariaDB 支持它），新的更改将 psycopg2 的“快速执行助手”方法推广到支持 RETURNING 的所有方言，现在包括 SQlite 和 MariaDB，并且对于没有其他方法来执行“executemany 加 RETURNING”的方言，包括 SQLite、MariaDB 和所有 PG 驱动程序。用于 Oracle 支持 RETURNING 的 cx_Oracle 和 oracledb 驱动程序会在本机支持 executemany，这也已经实现了相应的性能改进。由于 SQLite 和 MariaDB 现在提供 RETURNING 支持，ORM 对`cursor.lastrowid`的使用几乎成为历史，只有 MySQL 仍然依赖于它。

对于不使用 RETURNING 的 INSERT 语句，对于大多数后端，都使用传统的 executemany()行为，当前的例外是 psycopg2，它的 executemany()性能总体上非常慢，并且仍然受到“insertmanyvalues”方法的改进。

### 基准测试

SQLAlchemy 在`examples/`目录中包含一个性能套件，在这里我们可以利用`bulk_insert`套件以不同的方式使用 Core 和 ORM 来对插入多行的 INSERT 进行基准测试。

对于下面的测试，我们插入了**100,000 个对象**，在所有情况下，我们实际上都有 100,000 个真实的 Python ORM 对象在内存中，无论是预先创建的还是动态生成的。除了 SQLite 之外的所有数据库都通过本地网络连接运行，而不是本地主机；这导致“较慢”的结果非常慢。

此功能改进的操作包括：

+   使用`Session.add()`和`Session.add_all()`将对象添加到会话中的工作单元刷新。

+   新的 ORM 批量插入语句功能，改进了 SQLAlchemy 1.4 中首次引入的此功能的试验版本。

+   在批量操作中描述的`Session` “批量”操作，这些操作被上述 ORM 批量插入功能所取代。

为了对操作的规模有所了解，以下是使用`test_flush_no_pk`性能套件的性能测量结果，该套件历史上代表了 SQLAlchemy 的最坏情况 INSERT 性能任务，其中需要插入没有主键值的对象，然后必须获取新生成的主键值，以便对象可以用于后续的 flush 操作，例如在关系中建立关系，刷新连接继承模型等：

```py
@Profiler.profile
def test_flush_no_pk(n):
  """INSERT statements via the ORM (batched with RETURNING if available),
 fetching generated row id"""
    session = Session(bind=engine)
    for chunk in range(0, n, 1000):
        session.add_all(
            [
                Customer(
                    name="customer name %d" % i,
                    description="customer description %d" % i,
                )
                for i in range(chunk, chunk + 1000)
            ]
        )
        session.flush()
    session.commit()
```

可以从任何 SQLAlchemy 源代码树中运行此测试，如下所示：

```py
python -m examples.performance.bulk_inserts --test test_flush_no_pk
```

下表总结了最新的 1.4 系列 SQLAlchemy 与 2.0 的性能测量结果，两者都运行相同的测试：

| 驱动程序 | SQLA 1.4 时间（秒） | SQLA 2.0 时间（秒） |
| --- | --- | --- |
| sqlite+pysqlite2 (内存) | 6.204843 | 3.554856 |
| postgresql+asyncpg (网络) | 88.292285 | 4.561492 |
| postgresql+psycopg (网络) | N/A (psycopg3) | 4.861368 |
| mssql+pyodbc (网络) | 158.396667 | 4.825139 |
| oracle+cx_Oracle (网络) | 92.603953 | 4.809520 |
| mariadb+mysqldb (网络) | 71.705197 | 4.075377 |

注意

另外两个驱动程序在性能上没有变化；psycopg2 驱动程序，其在 SQLAlchemy 1.4 中已经实现了快速 executemany，以及 MySQL，它仍然不提供 RETURNING 支持：

| 驱动程序 | SQLA 1.4 时间（秒） | SQLA 2.0 时间（秒） |
| --- | --- | --- |
| postgresql+psycopg2 (网络) | 4.704876 | 4.699883 |
| mysql+mysqldb (网络) | 77.281997 | 76.132995 |

### 变更摘要

以下项目列出了 2.0 版本中为使所有驱动程序达到这种状态所做的各项更改：

+   SQLite 实现了 RETURNING - [#6195](https://www.sqlalchemy.org/trac/ticket/6195)

+   MariaDB 实现了 RETURNING - [#7011](https://www.sqlalchemy.org/trac/ticket/7011)

+   修复 Oracle 的多行 RETURNING - [#6245](https://www.sqlalchemy.org/trac/ticket/6245)

+   使 insert() executemany() 支持尽可能多的方言的 RETURNING，通常使用 VALUES() - [#6047](https://www.sqlalchemy.org/trac/ticket/6047)

+   当对不支持的后端使用 RETURNING w/ executemany 时发出警告（当前没有 RETURNING 后端有此限制）- [#7907](https://www.sqlalchemy.org/trac/ticket/7907)

+   ORM `Mapper.eager_defaults` 参数现在默认为新设置 `"auto"`，当使用的后端支持带有“insertmanyvalues”的 RETURNING 时，将自动启用“eager defaults”用于 INSERT 语句。请参阅获取服务器生成的默认值以获取文档。

另请参阅

INSERT 语句的“Insert Many Values”行为 - 新功能的文档和背景以及如何配置它

### 基准测试

SQLAlchemy 在 `examples/` 目录中包含一个性能套件，在这里我们可以利用 `bulk_insert` 套件以不同方式使用 Core 和 ORM 来对插入多行的性能进行基准测试。

对于下面的测试，我们插入了**100,000 个对象**，在所有情况下，我们实际上在内存中有 100,000 个真实的 Python ORM 对象，要么事先创建，要么动态生成。除 SQLite 外的所有数据库都通过本地网络连接运行，而不是 localhost；这导致“较慢”的结果非常慢。

通过这个功能改进的操作包括：

+   通过`Session.add()`和`Session.add_all()`向会话添加的对象的工作单元刷新。

+   新的 ORM 批量插入语句功能，改进了首次在 SQLAlchemy 1.4 中引入的此功能的实验版本。

+   `Session`中描述的“bulk”操作，已被上述 ORM 批量插入功能取代。

为了了解操作的规模，以下是使用`test_flush_no_pk`性能套件进行的性能测量，这通常代表 SQLAlchemy 的最坏情况 INSERT 性能任务，其中需要 INSERT 没有主键值的对象，然后必须获取新生成的主键值，以便这些对象可以用于后续的 flush 操作，比如在关系中建立，刷新加入继承模型等：

```py
@Profiler.profile
def test_flush_no_pk(n):
  """INSERT statements via the ORM (batched with RETURNING if available),
 fetching generated row id"""
    session = Session(bind=engine)
    for chunk in range(0, n, 1000):
        session.add_all(
            [
                Customer(
                    name="customer name %d" % i,
                    description="customer description %d" % i,
                )
                for i in range(chunk, chunk + 1000)
            ]
        )
        session.flush()
    session.commit()
```

可以从任何 SQLAlchemy 源代码树中运行此测试：

```py
python -m examples.performance.bulk_inserts --test test_flush_no_pk
```

下表总结了最新的 SQLAlchemy 1.4 系列与 2.0 之间的性能测量结果，两者都运行相同的测试：

| 驱动程序 | SQLA 1.4 时间（秒） | SQLA 2.0 时间（秒） |
| --- | --- | --- |
| sqlite+pysqlite2 (内存) | 6.204843 | 3.554856 |
| postgresql+asyncpg (网络) | 88.292285 | 4.561492 |
| postgresql+psycopg (网络) | N/A (psycopg3) | 4.861368 |
| mssql+pyodbc (网络) | 158.396667 | 4.825139 |
| oracle+cx_Oracle (网络) | 92.603953 | 4.809520 |
| mariadb+mysqldb (网络) | 71.705197 | 4.075377 |

注意

另外两个驱动程序在性能上没有变化；对于已在 SQLAlchemy 1.4 中实现了快速 executemany 的 psycopg2 驱动程序以及继续不提供 RETURNING 支持的 MySQL：

| 驱动程序 | SQLA 1.4 时间（秒） | SQLA 2.0 时间（秒） |
| --- | --- | --- |
| postgresql+psycopg2 (网络) | 4.704876 | 4.699883 |
| mysql+mysqldb (网络) | 77.281997 | 76.132995 |

### 变更摘要

以下项目列出了 2.0 中进行的各项更改，以使所有驱动程序达到此状态：

+   为 SQLite 实现了 RETURNING - [#6195](https://www.sqlalchemy.org/trac/ticket/6195)

+   为 MariaDB 实现了 RETURNING - [#7011](https://www.sqlalchemy.org/trac/ticket/7011)

+   修复了 Oracle 的多行 RETURNING - [#6245](https://www.sqlalchemy.org/trac/ticket/6245)

+   使 insert() executemany()支持尽可能多的方言，通常使用 VALUES() - [#6047](https://www.sqlalchemy.org/trac/ticket/6047)

+   当用于不支持的后端时发出警告 RETURNING w/ executemany（当前没有 RETURNING 后端具有此限制）- [#7907](https://www.sqlalchemy.org/trac/ticket/7907)

+   ORM `Mapper.eager_defaults`参数现在默认为一个新设置 `"auto"`, 当使用的后端支持带有“insertmanyvalues”的 RETURNING 时，会自动为 INSERT 语句启用“急切默认”。请参阅获取服务器生成的默认值了解文档。

另请参阅

INSERT 语句的“Insert Many Values”行为 - 新功能的文档和背景，以及如何配置它

## 启用 ORM 的插入、更新和删除语句，带有 ORM RETURNING

SQLAlchemy 1.4 将遗留的`Query`对象的特性移植到 2.0 样式执行，这意味着`Select`构造可以传递给`Session.execute()`以传递 ORM 结果。还增加了对`Update`和`Delete`的支持，以便它们可以提供`Query.update()`和`Query.delete()`的实现。

主要缺失的元素是对`Insert`构造的支持。1.4 文档通过使用`Select.from_statement()`的一些“插入”和“更新”配方来解决这个问题，将 RETURNING 集成到 ORM 上下文中。2.0 现在通过将`Insert`的直接支持整合为增强版本的`Session.bulk_insert_mappings()`方法以及对所有 DML 结构的完整 ORM RETURNING 支持来完全填补了这一空白。

### 带有 RETURNING 的批量插入

`Insert` 可以传递给 `Session.execute()`，可以带有或不带有 `Insert.returning()`，当传递给一个单独的参数列表时，将调用与以前由 `Session.bulk_insert_mappings()` 实现的相同的过程，同时增强了附加功能。这将优化使用新的快速插入功能的行批处理，同时还添加了对异构参数集和多表映射（如联合表继承）的支持：

```py
>>> users = session.scalars(
...     insert(User).returning(User),
...     [
...         {"name": "spongebob", "fullname": "Spongebob Squarepants"},
...         {"name": "sandy", "fullname": "Sandy Cheeks"},
...         {"name": "patrick", "fullname": "Patrick Star"},
...         {"name": "squidward", "fullname": "Squidward Tentacles"},
...         {"name": "ehkrabs", "fullname": "Eugene H. Krabs"},
...     ],
... )
>>> print(users.all())
[User(name='spongebob', fullname='Spongebob Squarepants'),
 User(name='sandy', fullname='Sandy Cheeks'),
 User(name='patrick', fullname='Patrick Star'),
 User(name='squidward', fullname='Squidward Tentacles'),
 User(name='ehkrabs', fullname='Eugene H. Krabs')]
```

[RETURNING](https://example.org/returning) 支持所有这些用例，ORM 将从多个语句调用中构造完整的结果集。

另请参阅

ORM 大规模插入语句

### 大规模 UPDATE

与 `Insert` 类似，将 `Update` 构造传递给包含主键值的参数列表的 `Session.execute()` 将调用之前由 `Session.bulk_update_mappings()` 方法支持的相同过程。但是，该功能不支持 RETURNING，因为它使用一个 SQL UPDATE 语句，该语句使用 DBAPI 的 executemany 调用：

```py
>>> from sqlalchemy import update
>>> session.execute(
...     update(User),
...     [
...         {"id": 1, "fullname": "Spongebob Squarepants"},
...         {"id": 3, "fullname": "Patrick Star"},
...     ],
... )
```

另请参阅

按主键进行 ORM 大规模 UPDATE

### 插入/ upsert … VALUES … RETURNING

使用 `Insert` 时，可在 `Insert.values()` 中包含一组参数，其中可能包括 SQL 表达式。此外，还支持 SQLite、PostgreSQL 和 MariaDB 的 upsert 变体。这些语句现在可以包含 `Insert.returning()` 子句，其中包括列表达式或完整的 ORM 实体：

```py
>>> from sqlalchemy.dialects.sqlite import insert as sqlite_upsert
>>> stmt = sqlite_upsert(User).values(
...     [
...         {"name": "spongebob", "fullname": "Spongebob Squarepants"},
...         {"name": "sandy", "fullname": "Sandy Cheeks"},
...         {"name": "patrick", "fullname": "Patrick Star"},
...         {"name": "squidward", "fullname": "Squidward Tentacles"},
...         {"name": "ehkrabs", "fullname": "Eugene H. Krabs"},
...     ]
... )
>>> stmt = stmt.on_conflict_do_update(
...     index_elements=[User.name], set_=dict(fullname=stmt.excluded.fullname)
... )
>>> result = session.scalars(stmt.returning(User))
>>> print(result.all())
[User(name='spongebob', fullname='Spongebob Squarepants'),
User(name='sandy', fullname='Sandy Cheeks'),
User(name='patrick', fullname='Patrick Star'),
User(name='squidward', fullname='Squidward Tentacles'),
User(name='ehkrabs', fullname='Eugene H. Krabs')]
```

另请参阅

使用每行 SQL 表达式进行 ORM 大规模插入

ORM "upsert" 语句

### ORM UPDATE / DELETE with WHERE … RETURNING

SQLAlchemy 1.4 还对与`update()`和`delete()`构造一起使用时与`Session.execute()`一起使用 RETURNING 功能提供了一些有限的支持。现在，此支持已经升级为完全本地化，包括`fetch`同步策略也可以继续进行，无论是否明确使用 RETURNING：

```py
>>> from sqlalchemy import update
>>> stmt = (
...     update(User)
...     .where(User.name == "squidward")
...     .values(name="spongebob")
...     .returning(User)
... )
>>> result = session.scalars(stmt, execution_options={"synchronize_session": "fetch"})
>>> print(result.all())
```

另请参阅

使用自定义 WHERE 条件的 ORM UPDATE 和 DELETE

使用 RETURNING 进行 UPDATE/DELETE 和自定义 WHERE 条件

### 改进的 ORM UPDATE / DELETE 的`synchronize_session`行为

synchronize_session 的默认策略现在是一个新值`"auto"`。此策略将尝试使用`"evaluate"`策略，然后自动回退到`"fetch"`策略。除了 MySQL / MariaDB 之外的所有后端，`"fetch"`使用 RETURNING 在同一语句中获取 UPDATE/DELETE 的主键标识符，因此通常比以前的版本更有效（在 1.4 中，RETURNING 仅适用于 PostgreSQL、SQL Server）。

另请参阅

选择同步策略

### 变更摘要

新的 ORM DML 带有 RETURNING 功能的列出的票据：

+   将 ORM 级别的`insert()`转换为在 ORM 上下文中解释`values()` - [#7864](https://www.sqlalchemy.org/trac/ticket/7864)

+   评估`dml.returning(Entity)`的可行性，以提供 ORM 表达式，自动应用`select().from_statement`等效 - [#7865](https://www.sqlalchemy.org/trac/ticket/7865)

+   给定 ORM 插入，尝试沿用批量方法，关于继承 - [#8360](https://www.sqlalchemy.org/trac/ticket/8360)

### 带有 RETURNING 的批量插入

`Insert`可以传递给`Session.execute()`，可以带有或不带有`Insert.returning()`，当与单独的参数列表一起传递时，将调用与以前由`Session.bulk_insert_mappings()`实现的相同过程，同时增加了额外的增强功能。这将优化行的批处理，利用新的快速插入多行功能，同时还支持异构参数集和多表映射，如联合表继承：

```py
>>> users = session.scalars(
...     insert(User).returning(User),
...     [
...         {"name": "spongebob", "fullname": "Spongebob Squarepants"},
...         {"name": "sandy", "fullname": "Sandy Cheeks"},
...         {"name": "patrick", "fullname": "Patrick Star"},
...         {"name": "squidward", "fullname": "Squidward Tentacles"},
...         {"name": "ehkrabs", "fullname": "Eugene H. Krabs"},
...     ],
... )
>>> print(users.all())
[User(name='spongebob', fullname='Spongebob Squarepants'),
 User(name='sandy', fullname='Sandy Cheeks'),
 User(name='patrick', fullname='Patrick Star'),
 User(name='squidward', fullname='Squidward Tentacles'),
 User(name='ehkrabs', fullname='Eugene H. Krabs')]
```

对于所有这些用例，都支持 RETURNING，其中 ORM 将从多个语句调用构造完整的结果集。

另请参阅

ORM 批量插入语句

### 批量更新

与`Insert`类似，将`Update`构造与包含主键值的参数列表一起传递给`Session.execute()`将调用与之前由`Session.bulk_update_mappings()`方法支持的相同过程。但是，此功能不支持 RETURNING，因为它使用了通过 DBAPI executemany 调用的 SQL UPDATE 语句：

```py
>>> from sqlalchemy import update
>>> session.execute(
...     update(User),
...     [
...         {"id": 1, "fullname": "Spongebob Squarepants"},
...         {"id": 3, "fullname": "Patrick Star"},
...     ],
... )
```

另请参阅

按主键批量更新的 ORM UPDATE

### INSERT / upsert … VALUES … RETURNING

当使用`Insert`与`Insert.values()`时，参数集合可以包含 SQL 表达式。此外，还支持 SQLite、PostgreSQL 和 MariaDB 等数据库的 upsert 变体。这些语句现在可以包括带有列表达式或完整 ORM 实体的`Insert.returning()`子句：

```py
>>> from sqlalchemy.dialects.sqlite import insert as sqlite_upsert
>>> stmt = sqlite_upsert(User).values(
...     [
...         {"name": "spongebob", "fullname": "Spongebob Squarepants"},
...         {"name": "sandy", "fullname": "Sandy Cheeks"},
...         {"name": "patrick", "fullname": "Patrick Star"},
...         {"name": "squidward", "fullname": "Squidward Tentacles"},
...         {"name": "ehkrabs", "fullname": "Eugene H. Krabs"},
...     ]
... )
>>> stmt = stmt.on_conflict_do_update(
...     index_elements=[User.name], set_=dict(fullname=stmt.excluded.fullname)
... )
>>> result = session.scalars(stmt.returning(User))
>>> print(result.all())
[User(name='spongebob', fullname='Spongebob Squarepants'),
User(name='sandy', fullname='Sandy Cheeks'),
User(name='patrick', fullname='Patrick Star'),
User(name='squidward', fullname='Squidward Tentacles'),
User(name='ehkrabs', fullname='Eugene H. Krabs')]
```

另请参阅

使用每行 SQL 表达式的 ORM 批量插入

ORM “upsert” 语句

### 带 WHERE … RETURNING 的 ORM UPDATE / DELETE

SQLAlchemy 1.4 也对与`update()`和`delete()`构造一起使用时与`Session.execute()`一起使用 RETURNING 功能提供了一些有限支持。此支持现已升级为完全本机，包括`fetch`同步策略也可以继续进行，无论是否存在显式使用 RETURNING：

```py
>>> from sqlalchemy import update
>>> stmt = (
...     update(User)
...     .where(User.name == "squidward")
...     .values(name="spongebob")
...     .returning(User)
... )
>>> result = session.scalars(stmt, execution_options={"synchronize_session": "fetch"})
>>> print(result.all())
```

另请参阅

使用自定义 WHERE 条件的 ORM UPDATE 和 DELETE

使用 UPDATE/DELETE 和自定义 WHERE 条件的 RETURNING

### ORM UPDATE / DELETE 的改进`synchronize_session`行为

synchronize_session 的默认策略现在是一个新值`"auto"`。此策略将尝试使用`"evaluate"`策略，然后自动回退到`"fetch"`策略。对于除 MySQL / MariaDB 之外的所有后端，`"fetch"`使用 RETURNING 在同一语句中获取 UPDATE/DELETE 的主键标识符，因此通常比以前的版本更有效率（在 1.4 中，RETURNING 仅适用于 PostgreSQL，SQL Server）。

另请参阅

选择同步策略

### 变更摘要

新 ORM DML 的带有 RETURNING 特性的已列出的票证：

+   将 ORM 级别的`insert()`转换为在 ORM 上下文中解释`values()`- [#7864](https://www.sqlalchemy.org/trac/ticket/7864)

+   评估 dml.returning(Entity)提供 ORM 表达式的可行性，自动应用 select().from_statement 等效 - [#7865](https://www.sqlalchemy.org/trac/ticket/7865)

+   给定 ORM 插入，尝试沿用批量方法，即继承关系- [#8360](https://www.sqlalchemy.org/trac/ticket/8360)

## 新的“只写”关系策略取代了“动态”

`lazy="dynamic"`加载策略已经过时，因为它是硬编码的，使用了遗留的`Query`。这种加载策略既不兼容 asyncio，而且还有许多行为会隐式地迭代其内容，这些行为背离了“动态”关系最初的目的，即针对不应在任何时候隐式完全加载到内存中的非常大的集合。

“动态”策略现已由新策略`lazy="write_only"`取代。可以使用`relationship()`的`relationship.lazy`参数进行“只写”配置，或者在使用类型注释映射时，将`WriteOnlyMapped`注释指示为映射样式：

```py
from sqlalchemy.orm import WriteOnlyMapped

class Base(DeclarativeBase):
    pass

class Account(Base):
    __tablename__ = "account"
    id: Mapped[int] = mapped_column(primary_key=True)
    identifier: Mapped[str]
    account_transactions: WriteOnlyMapped["AccountTransaction"] = relationship(
        cascade="all, delete-orphan",
        passive_deletes=True,
        order_by="AccountTransaction.timestamp",
    )

class AccountTransaction(Base):
    __tablename__ = "account_transaction"
    id: Mapped[int] = mapped_column(primary_key=True)
    account_id: Mapped[int] = mapped_column(
        ForeignKey("account.id", ondelete="cascade")
    )
    description: Mapped[str]
    amount: Mapped[Decimal]
    timestamp: Mapped[datetime] = mapped_column(default=func.now())
```

写入仅映射集合类似于`lazy="dynamic"`，因为集合可以提前分配，并且还具有`WriteOnlyCollection.add()`和`WriteOnlyCollection.remove()`等方法，以逐个项目的方式修改集合：

```py
new_account = Account(
    identifier="account_01",
    account_transactions=[
        AccountTransaction(description="initial deposit", amount=Decimal("500.00")),
        AccountTransaction(description="transfer", amount=Decimal("1000.00")),
        AccountTransaction(description="withdrawal", amount=Decimal("-29.50")),
    ],
)

new_account.account_transactions.add(
    AccountTransaction(description="transfer", amount=Decimal("2000.00"))
)
```

在数据库加载方面的主要区别在于，该集合没有直接从数据库加载对象的能力；相反，使用 SQL 构造方法，如 `WriteOnlyCollection.select()` 来生成 SQL 构造，比如 `Select`，然后使用 2.0 风格 显式加载所需对象：

```py
account_transactions = session.scalars(
    existing_account.account_transactions.select()
    .where(AccountTransaction.amount < 0)
    .limit(10)
).all()
```

`WriteOnlyCollection` 也与新的 ORM bulk dml 功能集成，包括支持带有 WHERE 条件的批量 INSERT 和 UPDATE/DELETE，所有这些都包括 RETURNING 支持。请参阅完整文档 Write Only Relationships。

另请参阅

Write Only Relationships

### 动态关系的新 pep-484 / 类型注释映射支持

尽管“动态”关系在 2.0 中是遗留的，但由于这些模式预计具有较长的生命周期，类型注释映射 现在已添加到“动态”关系中，与新的 `lazy="write_only"` 方法可用的方式相同，使用 `DynamicMapped` 注释：

```py
from sqlalchemy.orm import DynamicMapped

class Base(DeclarativeBase):
    pass

class Account(Base):
    __tablename__ = "account"
    id: Mapped[int] = mapped_column(primary_key=True)
    identifier: Mapped[str]
    account_transactions: DynamicMapped["AccountTransaction"] = relationship(
        cascade="all, delete-orphan",
        passive_deletes=True,
        order_by="AccountTransaction.timestamp",
    )

class AccountTransaction(Base):
    __tablename__ = "account_transaction"
    id: Mapped[int] = mapped_column(primary_key=True)
    account_id: Mapped[int] = mapped_column(
        ForeignKey("account.id", ondelete="cascade")
    )
    description: Mapped[str]
    amount: Mapped[Decimal]
    timestamp: Mapped[datetime] = mapped_column(default=func.now())
```

上述映射将提供一个 `Account.account_transactions` 集合，其类型为 `AppenderQuery` 集合类型，包括其元素类型，例如 `AppenderQuery[AccountTransaction]`。然后允许迭代和查询产生类型为 `AccountTransaction` 的对象。

另请参阅

动态关系加载器

[#7123](https://www.sqlalchemy.org/trac/ticket/7123)

### 动态关系的新 pep-484 / 类型注释映射支持

尽管“动态”关系在 2.0 中是遗留的，但由于这些模式预计具有较长的生命周期，类型注释映射 现在已添加到“动态”关系中，与新的 `lazy="write_only"` 方法可用的方式相同，使用 `DynamicMapped` 注释：

```py
from sqlalchemy.orm import DynamicMapped

class Base(DeclarativeBase):
    pass

class Account(Base):
    __tablename__ = "account"
    id: Mapped[int] = mapped_column(primary_key=True)
    identifier: Mapped[str]
    account_transactions: DynamicMapped["AccountTransaction"] = relationship(
        cascade="all, delete-orphan",
        passive_deletes=True,
        order_by="AccountTransaction.timestamp",
    )

class AccountTransaction(Base):
    __tablename__ = "account_transaction"
    id: Mapped[int] = mapped_column(primary_key=True)
    account_id: Mapped[int] = mapped_column(
        ForeignKey("account.id", ondelete="cascade")
    )
    description: Mapped[str]
    amount: Mapped[Decimal]
    timestamp: Mapped[datetime] = mapped_column(default=func.now())
```

上述映射将提供一个类型为返回`AppenderQuery`集合类型的`Account.account_transactions`集合，包括其元素类型，例如`AppenderQuery[AccountTransaction]`。这样就允许迭代和查询产生类型为`AccountTransaction`的对象。

另请参阅

动态关系加载器

[#7123](https://www.sqlalchemy.org/trac/ticket/7123)

## 安装现在完全支持 PEP-517

源代码分发现在包含一个`pyproject.toml`文件，以允许完全支持[**PEP 517**](https://peps.python.org/pep-0517/)。特别是，这允许使用`pip`进行本地源构建时自动安装可选依赖[Cython](https://cython.org/)。

[#7311](https://www.sqlalchemy.org/trac/ticket/7311)

## C 扩展现在转移到了 Cython

SQLAlchemy 的 C 扩展已被全部用[Cython](https://cython.org/)编写的新扩展替换。虽然 Cython 在 2010 年评估过，当时创建了 C 扩展，但今天使用的 C 扩展的性质和重点与当时相比已经发生了很大变化。同时，Cython 显然已经有了很大发展，Python 构建/分发工具链也使我们重新审视它成为可能。

迁移到 Cython 提供了明显的新优势，而没有明显的缺点：

+   用 Cython 替换特定 C 扩展的 Cython 扩展都经过了基准测试，通常比 SQLAlchemy 以前包含的几乎所有 C 代码都**更快**，有时显着快。虽然这看起来很神奇，但似乎是 Cython 实现中的一些非明显优化的结果，这些优化在直接将函数从 Python 转换为 C 时不会存在，特别是对于添加到 C 扩展的许多自定义集合类型的情况。

+   与原始 C 代码相比，Cython 扩展更容易编写、维护和调试，在大多数情况下与 Python 代码是逐行等效的。预计在即将发布的版本中，SQLAlchemy 的许多元素都将被转移到 Cython 中，这将打开许多以前无法实现的性能改进的新门路。

+   Cython 非常成熟且被广泛使用，包括成为 SQLAlchemy 支持的一些显著数据库驱动程序的基础，包括`asyncpg`、`psycopg3`和`asyncmy`。

像以前的 C 扩展一样，Cython 扩展被预先构建在 SQLAlchemy 的 wheel 分发中，这些分发可以自动从 PyPi 中的`pip`获得。手动构建说明也没有变化，除了 Cython 要求。

另请参阅

构建 Cython 扩展

[#7256](https://www.sqlalchemy.org/trac/ticket/7256)

## 数据库反射的重大架构、性能和 API 增强

`Table` 对象及其组件被反射的内部系统已经被完全重新架构，以允许参与方言一次性高性能地大量反射数千个表。目前，**PostgreSQL** 和 **Oracle** 方言参与了新的架构，其中 PostgreSQL 方言现在可以将大量 `Table` 对象反射得快近三倍，而 Oracle 方言现在可以将大量 `Table` 对象反射得快十倍。

重新架构最直接适用于利用 SELECT 查询系统目录表以反射表的方言，并且剩下的包括可以受益于这种方法的方言将是 SQL Server 方言。相比之下，MySQL/MariaDB 和 SQLite 方言利用非关系系统反射数据库表，并且没有受到现有性能问题的影响。

新的 API 与之前的系统向后兼容，并且不需要对第三方方言进行任何更改以保持兼容性；第三方方言也可以通过实现批量查询来选择加入新系统以进行模式反射。

除了这一变化，`Inspector` 对象的 API 和行为已经改进和增强，具有更一致的跨方言行为以及新的方法和性能特性。

### 性能概览

源分发包括一个脚本`test/perf/many_table_reflection.py`，它对现有的反射功能和新功能进行基准测试。其中一部分测试可以在较旧版本的 SQLAlchemy 上运行，在这里我们使用它来说明性能差异，调用`metadata.reflect()`一次性反射 250 个 `Table` 对象在本地网络连接上：

| 方言 | 操作 | SQLA 1.4 时间（秒） | SQLA 2.0 时间（秒） |
| --- | --- | --- | --- |
| postgresql+psycopg2 | `metadata.reflect()`，250 个表 | 8.2 | 3.3 |
| oracle+cx_oracle | `metadata.reflect()`，250 个表 | 60.4 | 6.8 |

### `Inspector()` 的行为变化

对于包含在 SQLite、PostgreSQL、MySQL/MariaDB、Oracle 和 SQL Server 中的 SQLAlchemy 方言，`Inspector.has_table()`、`Inspector.has_sequence()`、`Inspector.has_index()`、`Inspector.get_table_names()`和`Inspector.get_sequence_names()`现在在缓存方面都表现一致：在第一次为特定的`Inspector`对象调用后，它们都完全缓存其结果。当调用相同的`Inspector`对象创建或删除表/序列时，程序将不会在数据库状态发生更改后收到更新的状态。当要执行 DDL 更改时，应使用调用`Inspector.clear_cache()`或新的`Inspector`。之前，`Inspector.has_table()`、`Inspector.has_sequence()`方法没有实现缓存，也没有`Inspector`支持这些方法的缓存，而`Inspector.get_table_names()`和`Inspector.get_sequence_names()`方法是，导致两种类型的方法之间的结果不一致。

对于第三方方言的行为取决于它们是否实现了“反射缓存”装饰器来实现这些方法的方言级实现。

### 新的方法和改进`Inspector()`的行为

+   添加了一个方法`Inspector.has_schema()`，用于返回目标数据库中是否存在模式。

+   添加了一个方法`Inspector.has_index()`，用于返回表是否具有特定索引。

+   诸如`Inspector.get_columns()`之类的检查方法现在在一次只处理一个表时，如果未找到表或视图，将一致引发`NoSuchTableError`；此更改特定于各个方言，因此对于现有的第三方方言可能不适用。

+   将“视图”和“物化视图”的处理分开，因为在实际用例中，这两个构造使用不同的 DDL 来进行 CREATE 和 DROP；现在有单独的`Inspector.get_view_names()`和`Inspector.get_materialized_view_names()`方法。

[#4379](https://www.sqlalchemy.org/trac/ticket/4379)

### 性能概述

源代码分发包括一个脚本`test/perf/many_table_reflection.py`，该脚本对现有的反射功能以及新功能进行基准测试。其一部分测试可以在较旧版本的 SQLAlchemy 上运行，我们在这里使用它来说明在本地网络连接上一次性反射 250 个`Table`对象时调用`metadata.reflect()`的性能差异：

| 方言 | 操作 | SQLA 1.4 时间（秒） | SQLA 2.0 时间（秒） |
| --- | --- | --- | --- |
| postgresql+psycopg2 | `metadata.reflect()`, 250 tables | 8.2 | 3.3 |
| oracle+cx_oracle | `metadata.reflect()`, 250 tables | 60.4 | 6.8 |

### `Inspector()`的行为变化

对于包含在 SQLite、PostgreSQL、MySQL/MariaDB、Oracle 和 SQL Server 中的 SQLAlchemy 内置方言，`Inspector.has_table()`，`Inspector.has_sequence()`，`Inspector.has_index()`，`Inspector.get_table_names()`和`Inspector.get_sequence_names()`现在在缓存方面行为一致：它们在第一次为特定`Inspector`对象调用后完全缓存其结果。在调用相同的`Inspector`对象时创建或删除表/序列的程序在数据库状态发生变化后将不会接收到更新的状态。当要执行 DDL 更改时应使用`Inspector.clear_cache()`或一个新的`Inspector`。先前，`Inspector.has_table()`，`Inspector.has_sequence()`方法未实现缓存，`Inspector`也不支持这些方法的缓存，而`Inspector.get_table_names()`和`Inspector.get_sequence_names()`方法则是，导致两种方法之间结果不一致。

第三方方言的行为取决于它们是否实现了这些方法的方言级实现的“反射缓存”装饰器。

### 新的方法和改进对于`Inspector()`而言

+   添加了一个方法`Inspector.has_schema()`，用于返回目标数据库中是否存在模式

+   添加了一个方法`Inspector.has_index()`，用于返回表是否具有特定索引。

+   诸如`Inspector.get_columns()`之类的检查方法现在在一次只处理一个表时应一致地引发`NoSuchTableError`，如果未找到表或视图，则此更改特定于各个方言，因此对于现有的第三方方言可能不适用。

+   将“视图”和“物化视图”的处理分开，因为在实际用例中，这两个构造使用不同的 DDL 来进行 CREATE 和 DROP；现在有单独的`Inspector.get_view_names()`和`Inspector.get_materialized_view_names()`方法。

[#4379](https://www.sqlalchemy.org/trac/ticket/4379)

## 为 psycopg 3（又名“psycopg”）添加方言支持

为[psycopg 3](https://pypi.org/project/psycopg/) DBAPI 添加了方言支持，尽管现在以包名`psycopg`取代了之前的`psycopg2`包，后者目前仍然是 SQLAlchemy“默认”驱动程序的`postgresql`方言。 `psycopg`是一个完全重做和现代化的用于 PostgreSQL 的数据库适配器，支持诸如准备语句和 Python asyncio 等概念。

`psycopg`是 SQLAlchemy 支持的第一个 DBAPI，它提供了 pep-249 同步 API 和一个 asyncio 驱动程序。 可以使用相同的`psycopg`数据库 URL 与`create_engine()`和`create_async_engine()`引擎创建函数，并且相应的同步或 asyncio 版本的方言将自动选择。

另请参阅

psycopg

## 为 oracledb 添加方言支持

为[oracledb](https://pypi.org/project/oracledb/) DBAPI 添加了方言支持，这是流行的 cx_Oracle 驱动程序的重命名、新主要版本。

另请参阅

python-oracledb

## 新的条件 DDL 用于约束和索引

一个新的方法`Constraint.ddl_if()`和`Index.ddl_if()`允许像`CheckConstraint`、`UniqueConstraint`和`Index`这样的构造在给定的`Table`上有条件地渲染，基于与`DDLElement.execute_if()`方法接受的相同类型的条件。在下面的示例中，CHECK 约束和索引只会针对 PostgreSQL 后端生成：

```py
meta = MetaData()

my_table = Table(
    "my_table",
    meta,
    Column("id", Integer, primary_key=True),
    Column("num", Integer),
    Column("data", String),
    Index("my_pg_index", "data").ddl_if(dialect="postgresql"),
    CheckConstraint("num > 5").ddl_if(dialect="postgresql"),
)

e1 = create_engine("sqlite://", echo=True)
meta.create_all(e1)  # will not generate CHECK and INDEX

e2 = create_engine("postgresql://scott:tiger@localhost/test", echo=True)
meta.create_all(e2)  # will generate CHECK and INDEX
```

另请参见

控制约束和索引的 DDL 生成

[#7631](https://www.sqlalchemy.org/trac/ticket/7631)

## DATE、TIME、DATETIME 数据类型现在在所有后端上支持文字渲染

现在已经为后端特定编译实现了日期和时间类型的文字渲染，包括 PostgreSQL 和 Oracle：

```py
>>> import datetime

>>> from sqlalchemy import DATETIME
>>> from sqlalchemy import literal
>>> from sqlalchemy.dialects import oracle
>>> from sqlalchemy.dialects import postgresql

>>> date_literal = literal(datetime.datetime.now(), DATETIME)

>>> print(
...     date_literal.compile(
...         dialect=postgresql.dialect(), compile_kwargs={"literal_binds": True}
...     )
... )
'2022-12-17 11:02:13.575789'
>>> print(
...     date_literal.compile(
...         dialect=oracle.dialect(), compile_kwargs={"literal_binds": True}
...     )
... )
TO_TIMESTAMP('2022-12-17 11:02:13.575789',  'YYYY-MM-DD HH24:MI:SS.FF') 
```

以前，这种文字渲染仅在没有给定方言的情况下将语句字符串化时起作用；当尝试使用特定于方言的类型进行渲染时，会引发`NotImplementedError`，直到 SQLAlchemy 1.4.45，这变为���`CompileError`（属于[#8800](https://www.sqlalchemy.org/trac/ticket/8800)的一部分）。

当使用 PostgreSQL、MySQL、MariaDB、MSSQL、Oracle 方言提供的 SQL 编译器和`literal_binds`时，默认渲染为修改后的 ISO-8601 渲染（即将 T 转换为空格的 ISO-8601），对于 Oracle，ISO 格式被包装在适当的 TO_DATE() 函数调用中。对于 SQLite，渲染保持不变，因为该方言始终包含日期值的字符串渲染。

[#5052](https://www.sqlalchemy.org/trac/ticket/5052)

## `Result`、`AsyncResult` 的上下文管理器支持

`Result` 对象现在支持上下文管理器使用，这将确保对象及其底层游标在块结束时关闭。这在特定于服务器端游标的情况下特别有用，其中重要的是在操作结束时关闭打开的游标对象，即使发生了用户定义的异常：

```py
with engine.connect() as conn:
    with conn.execution_options(yield_per=100).execute(
        text("select * from table")
    ) as result:
        for row in result:
            print(f"{row}")
```

在使用 asyncio 时，`AsyncResult` 和 `AsyncConnection` 已经修改，以提供可选的异步上下文管理器使用，例如：

```py
async with async_engine.connect() as conn:
    async with conn.execution_options(yield_per=100).execute(
        text("select * from table")
    ) as result:
        for row in result:
            print(f"{row}")
```

[#8710](https://www.sqlalchemy.org/trac/ticket/8710)

## 行为变更

本节涵盖了在 SQLAlchemy 2.0 中进行的行为更改，这些更改在主要的 1.4->2.0 迁移路径中不是主要的一部分；这里的更改不应对向后兼容性产生重大影响。

### `Session` 的新事务加入模式

“将外部事务加入会话”的行为已经进行了修订和改进，允许显式控制`Session`将如何适应已经建立了事务和可能已经建立了保存点的传入`Connection`。新参数`Session.join_transaction_mode`包括一系列选项值，可以以多种方式适应现有事务，最重要的是允许`Session`仅使用保存点以完全事务化的方式运行，同时在任何情况下都保持外部启动的事务为非提交且处于活动状态，允许测试套件回滚测试中发生的所有更改。

这一主要改进允许文档中记录的将会话加入外部事务的方法（例如用于测试套件）的步骤，也从 SQLAlchemy 1.3 到 1.4 进行了更改，现在简化为不再需要显式使用事件处理程序或提及显式保存点；通过使用`join_transaction_mode="create_savepoint"`，`Session`永远不会影响传入事务的状态，而是创建一个保存点（即“嵌套事务”）作为其根事务。

以下是在将会话加入外部事务的方法（例如用于测试套件）给出的示例的部分内容；查看该部分以获取完整示例：

```py
class SomeTest(TestCase):
    def setUp(self):
        # connect to the database
        self.connection = engine.connect()

        # begin a non-ORM transaction
        self.trans = self.connection.begin()

        # bind an individual Session to the connection, selecting
        # "create_savepoint" join_transaction_mode
        self.session = Session(
            bind=self.connection, join_transaction_mode="create_savepoint"
        )

    def tearDown(self):
        self.session.close()

        # rollback non-ORM transaction
        self.trans.rollback()

        # return connection to the Engine
        self.connection.close()
```

`Session.join_transaction_mode`的默认模式选择为`"conditional_savepoint"`，如果给定的`Connection`本身已经在保存点上，则使用`"create_savepoint"`行为。如果给定的`Connection`处于事务中但不在保存点上，则`Session`将传播“rollback”调用但不会传播“commit”调用，但不会自行开始新的保存点。此行为被默认选择，因为它与旧版 SQLAlchemy 版本的兼容性最大，并且它不会启动新的 SAVEPOINT，除非给定的驱动程序已经在使用 SAVEPOINT，因为对 SAVEPOINT 的支持不仅取决于特定的后端和驱动程序，还取决于配置。

以下是一个案例，该案例在 SQLAlchemy 1.3 中有效，在 SQLAlchemy 1.4 中停止工作，现在在 SQLAlchemy 2.0 中已经恢复：

```py
engine = create_engine("...")

# setup outer connection with a transaction and a SAVEPOINT
conn = engine.connect()
trans = conn.begin()
nested = conn.begin_nested()

# bind a Session to that connection and operate upon it, including
# a commit
session = Session(conn)
session.connection()
session.commit()
session.close()

# assert both SAVEPOINT and transaction remain active
assert nested.is_active
nested.rollback()
trans.rollback()
```

在上述情况下，`Session`加入到已经启动保存点的`Connection`中；在`Session`处理事务后，这两个单元的状态保持不变。在 SQLAlchemy 1.3 中，上述案例有效，因为`Session`会在`Connection`上开始一个“子事务”，这将允许外部保存点/事务保持不受影响，就像上面的简单情况一样。由于子事务在 1.4 中已被弃用并在 2.0 中已被移除，因此此行为不再可用。新的默认行为通过使用真正的第二个 SAVEPOINT 来改进“子事务”的行为，因此即使调用`Session.rollback()`也会阻止`Session`“突破”到外部启动的 SAVEPOINT 或事务。

将已启动事务的`Connection`加入到`Session`中的新代码应明确选择`Session.join_transaction_mode`，以便明确定义所��的行为。

[#9015](https://www.sqlalchemy.org/trac/ticket/9015)  ### `str(engine.url)` 将默认混淆密码

为了避免数据库密码泄露，对 `URL` 调用 `str()` 现在默认启用密码混淆功能。以前，这种混淆将在 `__repr__()` 调用中生效，但不会在 `__str__()` 中生效。此更改将影响试图从另一个引擎的字符串化 URL 调用 `create_engine()` 的应用程序和测试套件，例如：

```py
>>> e1 = create_engine("postgresql+psycopg2://scott:tiger@localhost/test")
>>> e2 = create_engine(str(e1.url))
```

上述引擎 `e2` 将不会有正确的密码；它将有混淆的字符串 `"***"`。

上述模式的首选方法是直接传递 `URL` 对象，无需进行字符串化：

```py
>>> e1 = create_engine("postgresql+psycopg2://scott:tiger@localhost/test")
>>> e2 = create_engine(e1.url)
```

否则，对于具有明文密码的字符串化 URL，请使用 `URL.render_as_string()` 方法，并将 `URL.render_as_string.hide_password` 参数设置为 `False`：

```py
>>> e1 = create_engine("postgresql+psycopg2://scott:tiger@localhost/test")
>>> url_string = e1.url.render_as_string(hide_password=False)
>>> e2 = create_engine(url_string)
```

[#8567](https://www.sqlalchemy.org/trac/ticket/8567)  ### 对具有相同名称、键的 Table 对象中列的替换有更严格的规则

对于将 `Column` 对象附加到 `Table` 对象，现在有更严格的规则，将一些先前的弃用警告移至异常，并阻止一些以前会导致表中出现重复列的情况，当 `Table.extend_existing` 设置为 `True` 时，无论是在编程时 `Table` 构建还是在反射操作期间。

+   无论什么情况下，`Table` 对象都不应该有两个或更多具有相同名称的 `Column` 对象，无论它们有什么 .key。识别并修复了仍然可能出现此情况的边缘案例。

+   向具有与现有 `Column` 相同名称或键的 `Table` 添加 `Column` 将始终引发 `DuplicateColumnError`（在 2.0.0b4 中是 `ArgumentError` 的新子类）除非存在其他参数；对于 `Table.append_column()`，使用 `Table.append_column.replace_existing`，以及对于使用反射或不使用反射的构建一个具有相同名称的 `Table` 的 `Table.extend_existing`。此前，该情况已经有了废弃警告。

+   创建 `Table` 时现在会发出警告，如果其中包含 `Table.extend_existing`，其中一个没有单独的 `Column.key` 的传入 `Column` 会完全替换具有键的现有 `Column`，这表明操作并非用户意图。这种情况可能特别发生在次要反射步骤期间，例如 `metadata.reflect(extend_existing=True)`。警告建议将 `Table.autoload_replace` 参数设置为 `False` 以防止这种情况发生。在之前的版本中（1.4 及更早），传入的列会**额外**添加到现有列中。这是一个错误，并且在 2.0 中（截至 2.0.0b4）是一种行为变化，因为此时先前的键将**不再存在于**列集合中。

[#8925](https://www.sqlalchemy.org/trac/ticket/8925)  ### ORM 声明式应用列顺序不同；使用 `sort_order` 控制行为

声明式已更改了从 mixin 或抽象基类产生的映射列与声明类本身上的列一起排序的系统，以便先将声明类的列放在前面，然后是 mixin 列。以下映射：

```py
class Foo:
    col1 = mapped_column(Integer)
    col3 = mapped_column(Integer)

class Bar:
    col2 = mapped_column(Integer)
    col4 = mapped_column(Integer)

class Model(Base, Foo, Bar):
    id = mapped_column(Integer, primary_key=True)
    __tablename__ = "model"
```

在 1.4 上产生一个 CREATE TABLE 如下所示：

```py
CREATE  TABLE  model  (
  col1  INTEGER,
  col3  INTEGER,
  col2  INTEGER,
  col4  INTEGER,
  id  INTEGER  NOT  NULL,
  PRIMARY  KEY  (id)
)
```

而在 2.0 上它产生：

```py
CREATE  TABLE  model  (
  id  INTEGER  NOT  NULL,
  col1  INTEGER,
  col3  INTEGER,
  col2  INTEGER,
  col4  INTEGER,
  PRIMARY  KEY  (id)
)
```

对于上述特定情况，这可以被看作是一种改进，因为 `Model` 上的主键列现在位于人们通常更喜欢的位置。然而，对于以其他方式定义模型的应用程序来说，这并不令人感到安慰，因为：

```py
class Foo:
    id = mapped_column(Integer, primary_key=True)
    col1 = mapped_column(Integer)
    col3 = mapped_column(Integer)

class Model(Foo, Base):
    col2 = mapped_column(Integer)
    col4 = mapped_column(Integer)
    __tablename__ = "model"
```

现在的输出为 CREATE TABLE 如下：

```py
CREATE  TABLE  model  (
  col2  INTEGER,
  col4  INTEGER,
  id  INTEGER  NOT  NULL,
  col1  INTEGER,
  col3  INTEGER,
  PRIMARY  KEY  (id)
)
```

要解决这个问题，SQLAlchemy 2.0.4 引入了 `mapped_column()` 上的一个新参数 `mapped_column.sort_order`，它是一个整数值，默认为 `0`，可以设置为正值或负值，以便将列放置在其他列之前或之后，如下例所示：

```py
class Foo:
    id = mapped_column(Integer, primary_key=True, sort_order=-10)
    col1 = mapped_column(Integer, sort_order=-1)
    col3 = mapped_column(Integer)

class Model(Foo, Base):
    col2 = mapped_column(Integer)
    col4 = mapped_column(Integer)
    __tablename__ = "model"
```

上述模型将“id”放在所有其他列之前，将“col1”放在“id”之后：

```py
CREATE  TABLE  model  (
  id  INTEGER  NOT  NULL,
  col1  INTEGER,
  col2  INTEGER,
  col4  INTEGER,
  col3  INTEGER,
  PRIMARY  KEY  (id)
)
```

未来的 SQLAlchemy 发布版本可能会选择为 `mapped_column` 构造提供一个显式的排序提示，因为这种排序是 ORM 特定的。### `Sequence` 构造恢复为不具有任何显式默认的“start”值；影响 MS SQL Server

在 SQLAlchemy 1.4 之前，如果未指定任何其他参数，`Sequence` 构造将只发出简单的 `CREATE SEQUENCE` DDL：

```py
>>> # SQLAlchemy 1.3 (and 2.0)
>>> from sqlalchemy import Sequence
>>> from sqlalchemy.schema import CreateSequence
>>> print(CreateSequence(Sequence("my_seq")))
CREATE  SEQUENCE  my_seq 
```

然而，由于在 MS SQL Server 上添加了 `Sequence` 的支持，其中默认的起始值不方便设置为 `-2**63`，因此版本 1.4 决定默认情况下发出 DDL 以发射起始值为 1，如果未提供 `Sequence.start`：

```py
>>> # SQLAlchemy 1.4 (only)
>>> from sqlalchemy import Sequence
>>> from sqlalchemy.schema import CreateSequence
>>> print(CreateSequence(Sequence("my_seq")))
CREATE  SEQUENCE  my_seq  START  WITH  1 
```

此更改引入了其他复杂性，包括当包括 `Sequence.min_value` 参数时，默认值 `1` 实际上应该默认为 `Sequence.min_value` 所述的内容，否则，小于 start_value 的 min_value 可能被视为矛盾。由于查看此问题开始变得有点复杂，涉及到其他各种边缘情况，我们决定撤销此更改，并恢复 `Sequence` 的原始行为，即没有任何意见，只是发出 CREATE SEQUENCE，让数据库本身决定 `SEQUENCE` 的各种参数应如何相互作用。

因此，为了确保所有后端的起始值都为 1，**可以明确指示起始值为 1**，如下所示：

```py
>>> # All SQLAlchemy versions
>>> from sqlalchemy import Sequence
>>> from sqlalchemy.schema import CreateSequence
>>> print(CreateSequence(Sequence("my_seq", start=1)))
CREATE  SEQUENCE  my_seq  START  WITH  1 
```

此外，对于现代后端包括 PostgreSQL、Oracle、SQL Server 上的整数主键的自动生成，应优先使用`Identity`构造，这在 1.4 和 2.0 中的行为没有变化。

[#7211](https://www.sqlalchemy.org/trac/ticket/7211)  ### “with_variant()”克隆原始 TypeEngine 而不是更改类型

`TypeEngine.with_variant()`方法，用于将特定数据库的备用行为应用于特定类型，现在返回原始`TypeEngine`对象的副本，其中包含内部存储的变体信息，而不是将其包装在`Variant`类中。

虽然以前的`Variant`方法能够使用动态属性获取器保持原始类型的所有 Python 行为，但这里的改进是，调用变体时，返回的类型仍然是原始类型的实例，这更顺畅地与类型检查器如 mypy 和 pylance 配合使用。给定以下程序：

```py
import typing

from sqlalchemy import String
from sqlalchemy.dialects.mysql import VARCHAR

type_ = String(255).with_variant(VARCHAR(255, charset="utf8mb4"), "mysql", "mariadb")

if typing.TYPE_CHECKING:
    reveal_type(type_)
```

类型检查器如 pyright 现在将报告类型为：

```py
info: Type of "type_" is "String"
```

此外，如上所示，可以为单个类型传递多个方言名称，特别是对于被视为分开的"mysql"和"mariadb"方言对，这在 SQLAlchemy 1.4 中是有帮助的。

[#6980](https://www.sqlalchemy.org/trac/ticket/6980)  ### Python 除法运算符对所有后端执行真除法；添加了地板除法。

核心表达式语言现在支持“真除法”（即 Python 操作符`/`）和“地板除法”（即 Python 操作符`//`），包括后端特定的行为以规范化这方面不同数据库的行为。

给定两个整数值进行“真除法”操作：

```py
expr = literal(5, Integer) / literal(10, Integer)
```

例如，在 PostgreSQL 上，SQL 除法运算符通常在对整数使用时作为“地板除法”运行，这意味着上述结果将返回整数“0”。对于这些和类似的后端，SQLAlchemy 现在使用等效于以下形式的 SQL 来呈现：

```py
%(param_1)s  /  CAST(%(param_2)s  AS  NUMERIC)
```

当`param_1=5`，`param_2=10`时，返回表达式将是`NUMERIC`类型，通常作为 Python 值`decimal.Decimal("0.5")`。

给定两个整数值进行“地板除法”操作：

```py
expr = literal(5, Integer) // literal(10, Integer)
```

例如，在 MySQL 和 Oracle 上，SQL 除法运算符通常在对整数使用时作为“真除法”运行，这意味着上述结果将返回浮点值“0.5”。对于这些和类似的后端，SQLAlchemy 现在使用等效于以下形式的 SQL 来呈现：

```py
FLOOR(%(param_1)s  /  %(param_2)s)
```

当`param_1=5`，`param_2=10`时，返回表达式将是`INTEGER`类型，就像 Python 值`0`一样。

这里的不兼容变化是，如果一个应用程序使用 PostgreSQL、SQL Server 或 SQLite，并依赖于 Python 的“truediv”运算符在所有情况下返回整数值。依赖于这种行为的应用程序应该使用 Python 的“floor division”运算符 `//` 进行这些操作，或者在使用之前的 SQLAlchemy 版本时，使用 floor 函数以确保向前兼容性。

```py
expr = func.floor(literal(5, Integer) / literal(10, Integer))
```

在任何 SQLAlchemy 版本 2.0 之前的版本中，都需要上述形式来提供与后端无关的地板除法。

[#4926](https://www.sqlalchemy.org/trac/ticket/4926)  ### 当检测到非法并发或重入访问时，Session 现在会主动引发异常

`Session`现在可以捕获更多与多线程或其他并发场景中的非法并发状态更改相关的错误，以及执行意外状态更改的事件钩子。

当一个`Session`在多个线程同时使用时，可能会发生一个错误：`AttributeError: 'NoneType' object has no attribute 'twophase'`，这个错误完全是晦涩的。当一个线程调用`Session.commit()`时，内部会调用`SessionTransaction.close()`方法来结束事务上下文，与此同时另一个线程正在运行一个查询，如`Session.execute()`。在`Session.execute()`中，获取当前事务的数据库连接的内部方法首先会断言会话是“活动的”，但在这个断言通过后，同时调用`Session.close()`会干扰这个状态，导致上述未定义的条件。

这个改变对围绕`SessionTransaction`对象的所有改变状态的方法应用了保护措施，因此在上述情况下，`Session.commit()`方法将会失败，因为它试图将状态更改为在已经进行中的方法中不允许的状态，而这个方法想要获取当前连接来运行数据库查询。

使用在[#7433](https://www.sqlalchemy.org/trac/ticket/7433)中说明的测试脚本，前面的错误案例看起来像这样：

```py
Traceback (most recent call last):
File "/home/classic/dev/sqlalchemy/test3.py", line 30, in worker
    sess.execute(select(A)).all()
File "/home/classic/tmp/sqlalchemy/lib/sqlalchemy/orm/session.py", line 1691, in execute
    conn = self._connection_for_bind(bind)
File "/home/classic/tmp/sqlalchemy/lib/sqlalchemy/orm/session.py", line 1532, in _connection_for_bind
    return self._transaction._connection_for_bind(
File "/home/classic/tmp/sqlalchemy/lib/sqlalchemy/orm/session.py", line 754, in _connection_for_bind
    if self.session.twophase and self._parent is None:
AttributeError: 'NoneType' object has no attribute 'twophase'
```

当`_connection_for_bind()`方法由于并发访问而无法继续时。使用新方法，状态更改的发起者会抛出错误：

```py
File "/home/classic/dev/sqlalchemy/lib/sqlalchemy/orm/session.py", line 1785, in close
   self._close_impl(invalidate=False)
File "/home/classic/dev/sqlalchemy/lib/sqlalchemy/orm/session.py", line 1827, in _close_impl
   transaction.close(invalidate)
File "<string>", line 2, in close
File "/home/classic/dev/sqlalchemy/lib/sqlalchemy/orm/session.py", line 506, in _go
   raise sa_exc.InvalidRequestError(
sqlalchemy.exc.InvalidRequestError: Method 'close()' can't be called here;
method '_connection_for_bind()' is already in progress and this would cause
an unexpected state change to symbol('CLOSED')
```

状态转换检查故意不使用显式锁来检测并发线程活动，而是依赖于简单的属性设置/值测试操作，当发生意外的并发更改时会自然失败。其理念是该方法可以检测在单个线程内完全发生的非法状态更改，例如在会话事务事件上运行的事件处理程序调用了不被期望的状态更改方法，或者在 asyncio 中，如果一个特定的`Session`被多个 asyncio 任务共享，以及在使用诸如 gevent 之类的补丁样式并发方法时。

[#7433](https://www.sqlalchemy.org/trac/ticket/7433)  ### SQLite 方言在基于文件的数据库中使用 QueuePool

当使用基于文件的数据库时，SQLite 方言现在默认为`QueuePool`。这是在将`check_same_thread`参数设置为`False`的同时进行的。已经观察到，以前默认为`NullPool`的方法，在释放连接后不会保留数据库连接，实际上确实对性能产生了可测量的负面影响。像往常一样，可以通过`create_engine.poolclass`参数自定义池类。

另请参阅

线程/池行为

[#7490](https://www.sqlalchemy.org/trac/ticket/7490)  ### 具有二进制精度的新 Oracle FLOAT 类型；不能直接接受十进制精度

Oracle 方言添加了新的数据类型`FLOAT`，以配合`Double`和数据库特定的`DOUBLE`、`DOUBLE_PRECISION`和`REAL`数据类型的添加。 Oracle 的`FLOAT`接受所谓的“二进制精度”参数，根据 Oracle 文档，这大致是标准“精度”值除以 0.3103：

```py
from sqlalchemy.dialects import oracle

Table("some_table", metadata, Column("value", oracle.FLOAT(126)))
```

二进制精度值 126 等同于使用 `DOUBLE_PRECISION` 数据类型，而值 63 等效于使用 `REAL` 数据类型。其他精度值特定于 `FLOAT` 类型本身。

SQLAlchemy `Float` 数据类型还接受“精度”参数，但这是十进制精度，Oracle 不接受。Oracle 方言现在将在针对 Oracle 后端使用 `Float` 与精度值时引发信息性错误，而不是尝试猜测转换。要为支持的后端指定具有显式精度值的 `Float` 数据类型，同时还支持其他后端，请使用 `TypeEngine.with_variant()` 方法如下：

```py
from sqlalchemy.types import Float
from sqlalchemy.dialects import oracle

Table(
    "some_table",
    metadata,
    Column("value", Float(5).with_variant(oracle.FLOAT(16), "oracle")),
)
```  ### PostgreSQL 后端的新 RANGE / MULTIRANGE 支持和更改

对于 psycopg2、psycopg3 和 asyncpg 方言，已完全实现了 RANGE / MULTIRANGE 支持。新的支持使用一个新的与后端无关的 SQLAlchemy 特定的 `Range` 对象，不需要使用后端特定的导入或扩展步骤。对于多范围支持，使用 `Range` 对象的列表。

使用先前的 psycopg2 特定类型的代码应修改为使用 `Range`，它提供了兼容的接口。

`Range` 对象还具有与 PostgreSQL 相同的比较支持。到目前为止已经实现了 `Range.contains()` 和 `Range.contained_by()` 方法，它们的工作方式与 PostgreSQL 的 `@>` 和 `<@` 相同。未来的版本可能会增加其他操作符支持。

请参阅 Range and Multirange Types 文档，了解使用这一新功能的背景。

另请参阅

范围和多范围类型

[#7156](https://www.sqlalchemy.org/trac/ticket/7156) [#8706](https://www.sqlalchemy.org/trac/ticket/8706)  ### 在 PostgreSQL 上，`match()` 运算符使用 `plainto_tsquery()` 而不是 `to_tsquery()`

在 PostgreSQL 后端上，`Operators.match()` 函数现在呈现 `col @@ plainto_tsquery(expr)`，而不是 `col @@ to_tsquery()`。`plainto_tsquery()` 接受纯文本，而 `to_tsquery()` 接受专用查询符号，因此与其他后端的兼容性较差。

通过使用 `func` 生成 PostgreSQL 特定函数和 `Operators.bool_op()`（`Operators.op()` 的布尔类型版本）生成任意运算符，可以使用所有 PostgreSQL 搜索函数和操作符，方式与先前版本中提供的方式相同。请参阅 全文搜索 中的示例。

在使用 `Operators.match()` 内的 PG 特定指令的现有 SQLAlchemy 项目应直接使用 `func.to_tsquery()`。要以与 1.4 中相同的形式呈现 SQL，请参阅 使用 match() 进行简单的纯文本匹配 的版本说明。

[#7086](https://www.sqlalchemy.org/trac/ticket/7086)  ### `Session` 的新事务连接模式

“将外部事务加入到会话中”的行为已经修订和改进，允许对 `Session` 如何适应已经建立了事务和可能已经建立了保存点的传入 `Connection` 进行显式控制。新的参数 `Session.join_transaction_mode` 包括一系列选项值，可以以几种方式适应现有的事务，最重要的是允许一个 `Session` 以完全事务性的方式操作，专门使用保存点，同时在所有情况下保持外部启动的事务未提交且处于活动状态，从而允许测试套件回滚在测试中发生的所有更改。

这样做的主要改进是，文档中记录的 将会话加入外部事务（例如测试套件） 的配方，也从 SQLAlchemy 1.3 更改为 1.4，现在简化为不再需要显式使用事件处理程序或任何提及显式保存点；通过使用 `join_transaction_mode="create_savepoint"`，`Session` 将永远不会影响传入事务的状态，而是创建一个保存点（即“嵌套事务”）作为其根事务。

以下是在 将会话加入外部事务（例如测试套件） 中给出的示例的一部分说明；请参阅该部分以获取完整示例：

```py
class SomeTest(TestCase):
    def setUp(self):
        # connect to the database
        self.connection = engine.connect()

        # begin a non-ORM transaction
        self.trans = self.connection.begin()

        # bind an individual Session to the connection, selecting
        # "create_savepoint" join_transaction_mode
        self.session = Session(
            bind=self.connection, join_transaction_mode="create_savepoint"
        )

    def tearDown(self):
        self.session.close()

        # rollback non-ORM transaction
        self.trans.rollback()

        # return connection to the Engine
        self.connection.close()
```

`Session.join_transaction_mode` 的默认模式选择是 `"conditional_savepoint"`，如果给定的 `Connection` 已经处于保存点上，则使用 `"create_savepoint"` 行为。如果给定的 `Connection` 处于事务中但不在保存点中，则 `Session` 将传播“回滚”调用但不传播“提交”调用，但不会自行开始新的保存点。此行为被默认选择，因为它最大程度地与旧版本的 SQLAlchemy 兼容，并且它不会启动新的 SAVEPOINT，除非给定的驱动程序已经在使用 SAVEPOINT，因为 SAVEPOINT 的支持不仅与特定的后端和驱动程序有关，还与配置有关。

以下是一个案例的示例，在 SQLAlchemy 1.3 中有效，在 SQLAlchemy 1.4 中停止工作，并在 SQLAlchemy 2.0 中恢复：

```py
engine = create_engine("...")

# setup outer connection with a transaction and a SAVEPOINT
conn = engine.connect()
trans = conn.begin()
nested = conn.begin_nested()

# bind a Session to that connection and operate upon it, including
# a commit
session = Session(conn)
session.connection()
session.commit()
session.close()

# assert both SAVEPOINT and transaction remain active
assert nested.is_active
nested.rollback()
trans.rollback()
```

在上述情况下，`Session` 加入到具有已启动保存点的 `Connection` 中后；这两个单元的状态在 `Session` 处理事务后保持不变。在 SQLAlchemy 1.3 中，上述情况有效，因为 `Session` 将在 `Connection` 上开始一个“子事务”，这将允许外部保存点 / 事务保持不受影响，对于上述简单情况而言。由于子事务在 1.4 中已弃用并在 2.0 中已删除，因此此行为不再可用。新的默认行为通过使用真正的第二个 SAVEPOINT 改进了“子事务”的行为，以便即使调用 `Session.rollback()` 也会阻止 `Session` “突破”到外部启动的 SAVEPOINT 或事务。

新代码将 `Connection` 加入到 `Session` 的事务中时，应明确选择一个 `Session.join_transaction_mode` ，以明确定义所需的行为。

[#9015](https://www.sqlalchemy.org/trac/ticket/9015)

### `str(engine.url)` 将默认混淆密码

为了避免数据库密码泄露，现在在 `URL` 上调用 `str()` 将默认启用密码混淆功能。之前，此混淆对于 `__repr__()` 调用有效，但对于 `__str__()` 调用无效。此更改将影响尝试使用来自另一个引擎的字符串化 URL 调用 `create_engine()` 的应用程序和测试套件，例如：

```py
>>> e1 = create_engine("postgresql+psycopg2://scott:tiger@localhost/test")
>>> e2 = create_engine(str(e1.url))
```

上述引擎 `e2` 将不会有正确的密码；它将有混淆的字符串 `"***"`。

上述模式的首选方法是直接传递 `URL` 对象，无需转换为字符串：

```py
>>> e1 = create_engine("postgresql+psycopg2://scott:tiger@localhost/test")
>>> e2 = create_engine(e1.url)
```

否则，对于具有明文密码的字符串化 URL，请使用 `URL.render_as_string()` 方法，并将 `URL.render_as_string.hide_password` 参数设置为 `False`：

```py
>>> e1 = create_engine("postgresql+psycopg2://scott:tiger@localhost/test")
>>> url_string = e1.url.render_as_string(hide_password=False)
>>> e2 = create_engine(url_string)
```

[#8567](https://www.sqlalchemy.org/trac/ticket/8567)

### 替换具有相同名称和键的 Table 对象中的 Columns 的更严格规则

对于将 `Column` 对象附加到 `Table` 对象，有更严格的规则，将一些先前的弃用警告转移到异常中，并防止一些先前可能导致表中出现重复列的情况，当 `Table.extend_existing` 设置为 `True` 时，对于编程方式的 `Table` 构建以及在反射操作期间。

+   无论如何，`Table` 对象都不应该具有两个或更多具有相同名称的 `Column` 对象，无论它们的 .key 如何。已经确定并修复了仍然可能发生此情况的边缘情况。

+   向具有与现有 `Column` 相同名称或键的 `Table` 添加 `Column` 将始终引发 `DuplicateColumnError`（在 2.0.0b4 中是 `ArgumentError` 的新子类），除非存在额外参数；对于 `Table.append_column()`，使用 `Table.append_column.replace_existing`，以及对于构建具有与现有 `Table` 相同名称的 `Table`（使用或不使用反射）时使用 `Table.extend_existing`。此前，针对此情况已经放置了弃用警告。

+   如果创建了一个包含`Table.extend_existing`的`Table`，并且存在一个没有单独的`Column.key`的传入`Column`，该传入的`Column`将完全替换具有键的现有`Column`，这表明操作不是用户所期望的。这在特别是在次要反射步骤期间可能发生，例如`metadata.reflect(extend_existing=True)`。警告建议将`Table.autoload_replace`参数设置为`False`以防止这种情况发生。在 1.4 及以前的版本中，传入的列会**额外**添加到现有列中。这是一个错误，在 2.0（截至 2.0.0b4）中是一种行为变更，因为在这种情况发生时，以前的键将**不再存在**于列集合中。

[#8925](https://www.sqlalchemy.org/trac/ticket/8925)

### ORM 声明式不同的列顺序应用方式；使用`sort_order`控制行为

声明式已更改了来自混合或抽象基类的映射列与声明类本身上的列一起排序的系统，以便首先放置来自声明类的列，然后是混合列。以下映射：

```py
class Foo:
    col1 = mapped_column(Integer)
    col3 = mapped_column(Integer)

class Bar:
    col2 = mapped_column(Integer)
    col4 = mapped_column(Integer)

class Model(Base, Foo, Bar):
    id = mapped_column(Integer, primary_key=True)
    __tablename__ = "model"
```

在 1.4 上产生的 CREATE TABLE 如下：

```py
CREATE  TABLE  model  (
  col1  INTEGER,
  col3  INTEGER,
  col2  INTEGER,
  col4  INTEGER,
  id  INTEGER  NOT  NULL,
  PRIMARY  KEY  (id)
)
```

而在 2.0 上它产生：

```py
CREATE  TABLE  model  (
  id  INTEGER  NOT  NULL,
  col1  INTEGER,
  col3  INTEGER,
  col2  INTEGER,
  col4  INTEGER,
  PRIMARY  KEY  (id)
)
```

对于上述特定情况，这可以视为一种改进，因为`Model`上的主键列现在位于人们通常更喜欢的位置。然而，对于以相反方式定义模型的应用程序来说，这并没有什么安慰，因为：

```py
class Foo:
    id = mapped_column(Integer, primary_key=True)
    col1 = mapped_column(Integer)
    col3 = mapped_column(Integer)

class Model(Foo, Base):
    col2 = mapped_column(Integer)
    col4 = mapped_column(Integer)
    __tablename__ = "model"
```

现在，这将产生以下 CREATE TABLE 输出：

```py
CREATE  TABLE  model  (
  col2  INTEGER,
  col4  INTEGER,
  id  INTEGER  NOT  NULL,
  col1  INTEGER,
  col3  INTEGER,
  PRIMARY  KEY  (id)
)
```

为解决此问题，SQLAlchemy 2.0.4 引入了`mapped_column()`上的一个新参数，称为`mapped_column.sort_order`，它是一个整数值，默认为`0`，可以设置为正值或负值，以便将列放置在其他列之前或之后，如下例所示：

```py
class Foo:
    id = mapped_column(Integer, primary_key=True, sort_order=-10)
    col1 = mapped_column(Integer, sort_order=-1)
    col3 = mapped_column(Integer)

class Model(Foo, Base):
    col2 = mapped_column(Integer)
    col4 = mapped_column(Integer)
    __tablename__ = "model"
```

上述模型将“id”放置在所有其他列之前，将“col1”放置在“id”之后：

```py
CREATE  TABLE  model  (
  id  INTEGER  NOT  NULL,
  col1  INTEGER,
  col2  INTEGER,
  col4  INTEGER,
  col3  INTEGER,
  PRIMARY  KEY  (id)
)
```

未来的 SQLAlchemy 版本可能选择为`mapped_column`构造提供显式排序提示，因为此排序是 ORM 特定的。

### `Sequence` 构造不再具有任何显式默认的“start”值；影响 MS SQL Server

在 SQLAlchemy 1.4 之前，`Sequence` 构造将仅在未指定其他参数时发出简单的 `CREATE SEQUENCE` DDL：

```py
>>> # SQLAlchemy 1.3 (and 2.0)
>>> from sqlalchemy import Sequence
>>> from sqlalchemy.schema import CreateSequence
>>> print(CreateSequence(Sequence("my_seq")))
CREATE  SEQUENCE  my_seq 
```

但是，由于为 MS SQL Server 添加了 `Sequence` 支持，其中默认起始值不方便地设置为 `-2**63`，版本 1.4 决定将 DDL 默认为发出起始值为 1，如果未提供 `Sequence.start`：

```py
>>> # SQLAlchemy 1.4 (only)
>>> from sqlalchemy import Sequence
>>> from sqlalchemy.schema import CreateSequence
>>> print(CreateSequence(Sequence("my_seq")))
CREATE  SEQUENCE  my_seq  START  WITH  1 
```

此更改引入了其他复杂性，包括当包含 `Sequence.min_value` 参数时，默认值为 `1` 实际上应默认为 `Sequence.min_value` 所述的内容，否则，将看到低于 start_value 的 min_value 可能被视为矛盾。由于研究此问题开始变得有点复杂，包括各种其他边缘情况，我们决定撤消此更改，并恢复 `Sequence` 的原始行为，即不表达任何观点，只是发出 CREATE SEQUENCE，允许数据库本身决定如何处理 `SEQUENCE` 的各种参数之间的交互。

因此，为了确保所有后端的起始值都为 1，**可能需要显式指定起始值为 1**，如下所示：

```py
>>> # All SQLAlchemy versions
>>> from sqlalchemy import Sequence
>>> from sqlalchemy.schema import CreateSequence
>>> print(CreateSequence(Sequence("my_seq", start=1)))
CREATE  SEQUENCE  my_seq  START  WITH  1 
```

此外，在现代后端（包括 PostgreSQL、Oracle、SQL Server）上自动生成整数主键时，应优先使用 `Identity` 构造，在 1.4 和 2.0 中也以相同方式工作，行为没有任何更改。

[#7211](https://www.sqlalchemy.org/trac/ticket/7211)

### “with_variant()” 克隆原始 TypeEngine 而不是更改类型

`TypeEngine.with_variant()` 方法用于将特定类型应用于数据库的备用行为，现在返回原始 `TypeEngine` 对象的副本，并在内部存储变体信息，而不是将其包装在 `Variant` 类中。

虽然以前的 `Variant` 方法能够使用动态属性获取器维护原始类型的所有 Python 行为，但这里的改进是，当调用变体时，返回的类型仍然是原始类型的实例，这与诸如 mypy 和 pylance 的类型检查器更加顺畅地配合。给定以下程序：

```py
import typing

from sqlalchemy import String
from sqlalchemy.dialects.mysql import VARCHAR

type_ = String(255).with_variant(VARCHAR(255, charset="utf8mb4"), "mysql", "mariadb")

if typing.TYPE_CHECKING:
    reveal_type(type_)
```

类型检查器如 pyright 现在将报告类型为：

```py
info: Type of "type_" is "String"
```

此外，如上所示，对于单个类型，可以传递多个方言名称，特别是对于被视为分开的`"mysql"`和`"mariadb"`方言的成对，这对于 SQLAlchemy 1.4 很有帮助。

[#6980](https://www.sqlalchemy.org/trac/ticket/6980)

### Python 除法运算符对所有后端执行真除法；添加地板除法

核心表达式语言现在支持“真除法”（即 `/` Python 运算符）和“地板除法”（即 `//` Python 运算符），包括标准化此方面不同数据库的后端特定行为。

给定两个整数值进行“真除法”操作：

```py
expr = literal(5, Integer) / literal(10, Integer)
```

PostgreSQL 上的 SQL 除法运算符通常在对整数进行操作时作为“地板除法”（floor division）操作，意味着以上结果将返回整数“0”。对于此类后端，SQLAlchemy 现在使用等效于以下形式的 SQL 来渲染 SQL：

```py
%(param_1)s  /  CAST(%(param_2)s  AS  NUMERIC)
```

使用 `param_1=5`，`param_2=10`，以便返回表达式将是 NUMERIC 类型，通常为 Python 值 `decimal.Decimal("0.5")`。

给定两个整数值进行“地板除法”操作：

```py
expr = literal(5, Integer) // literal(10, Integer)
```

MySQL 和 Oracle 上的 SQL 除法运算符通常在对整数进行操作时作为“真除法”（true division）操作，意味着以上结果将返回浮点值“0.5”。对于这些和类似的后端，SQLAlchemy 现在使用等效于以下形式的 SQL 渲染 SQL：

```py
FLOOR(%(param_1)s  /  %(param_2)s)
```

使用 param_1=5，param_2=10，以便返回表达式将是整数类型，就像 Python 值 `0` 一样。

此处的不兼容变更将是，如果一个应用程序使用 PostgreSQL、SQL Server 或 SQLite，并且依赖于 Python 的“truediv”运算符在所有情况下返回整数值。依赖于此行为的应用程序应该使用 Python 的“地板除法”运算符 `//` 进行这些操作，或者在使用之前的 SQLAlchemy 版本时进行前向兼容，使用 floor 函数：

```py
expr = func.floor(literal(5, Integer) / literal(10, Integer))
```

在 SQLAlchemy 版本 2.0 之前的任何版本中，将需要上述形式以提供与后端无关的地板除法。

[#4926](https://www.sqlalchemy.org/trac/ticket/4926)

### Session 主动提出当检测到非法并发或重入访问时

`Session` 现在可以更全面地捕获与多线程或其他并发场景中的非法并发状态更改相关的错误，以及执行意外状态更改的事件钩子。

已知的一个错误是当一个`Session`同时在多个线程中使用时会出现`AttributeError: 'NoneType' object has no attribute 'twophase'`，这完全是神秘的。当一个线程调用`Session.commit()`时，内部调用`SessionTransaction.close()`方法来结束事务上下文，与此同时另一个线程正在运行一个查询，如`Session.execute()`。在`Session.execute()`中，获取当前事务的数据库连接的内部方法首先开始断言会话是“活动的”，但在此断言通过后，同时进行的对`Session.close()`的调用干扰了这种状态，导致上述未定义的条件。

更改将应用到围绕`SessionTransaction`对象的所有更改状态方法的保护措施，以便在上述情况下，`Session.commit()`方法将会失败，因为它将试图将状态更改为在已经进行中的方法期间不允许的状态，而该方法希望获取当前连接以运行数据库查询。

使用在[#7433](https://www.sqlalchemy.org/trac/ticket/7433)中展示的测试脚本，先前的错误案例如下：

```py
Traceback (most recent call last):
File "/home/classic/dev/sqlalchemy/test3.py", line 30, in worker
    sess.execute(select(A)).all()
File "/home/classic/tmp/sqlalchemy/lib/sqlalchemy/orm/session.py", line 1691, in execute
    conn = self._connection_for_bind(bind)
File "/home/classic/tmp/sqlalchemy/lib/sqlalchemy/orm/session.py", line 1532, in _connection_for_bind
    return self._transaction._connection_for_bind(
File "/home/classic/tmp/sqlalchemy/lib/sqlalchemy/orm/session.py", line 754, in _connection_for_bind
    if self.session.twophase and self._parent is None:
AttributeError: 'NoneType' object has no attribute 'twophase'
```

当`_connection_for_bind()`方法无法继续运行时，因为并发访问使其处于无效状态。使用新方法，状态更改的发起者会抛出错误：

```py
File "/home/classic/dev/sqlalchemy/lib/sqlalchemy/orm/session.py", line 1785, in close
   self._close_impl(invalidate=False)
File "/home/classic/dev/sqlalchemy/lib/sqlalchemy/orm/session.py", line 1827, in _close_impl
   transaction.close(invalidate)
File "<string>", line 2, in close
File "/home/classic/dev/sqlalchemy/lib/sqlalchemy/orm/session.py", line 506, in _go
   raise sa_exc.InvalidRequestError(
sqlalchemy.exc.InvalidRequestError: Method 'close()' can't be called here;
method '_connection_for_bind()' is already in progress and this would cause
an unexpected state change to symbol('CLOSED')
```

状态转换检查故意不使用显式锁来检测并发线程活动，而是依赖于简单的属性设置/值测试操作，当发生意外的并发更改时会自然失败。其理念是该方法可以检测到完全发生在单个线程内的非法状态更改，例如运行在会话事务事件上的事件处理程序调用了一个未预期的改变状态的方法，或者在 asyncio 中，如果一个特定的`Session`被多个 asyncio 任务共享，以及在使用类似 gevent 的补丁式并发方法时。

[#7433](https://www.sqlalchemy.org/trac/ticket/7433)

### SQLite 方言使用 QueuePool 用于基于文件的数据库

当使用基于文件的数据库时，SQLite 方言现在默认为 `QueuePool`。这与将 `check_same_thread` 参数设置为 `False` 一起设置。已观察到以前默认为 `NullPool` 的方法，在释放连接后不保留数据库连接，事实上会产生可测量的负面性能影响。与以往一样，池类可通过 `create_engine.poolclass` 参数进行自定义。

另请参阅

线程/池行为

[#7490](https://www.sqlalchemy.org/trac/ticket/7490)

### 新的 Oracle FLOAT 类型，带有二进制精度；不直接接受十进制精度

Oracle 方言现已添加了新的数据类型 `FLOAT`，以配合 `Double` 和数据库特定的 `DOUBLE`、`DOUBLE_PRECISION` 和 `REAL` 数据类型的添加。Oracle 的 `FLOAT` 接受所谓的“二进制精度”参数，根据 Oracle 文档，这大致是标准“精度”值除以 0.3103：

```py
from sqlalchemy.dialects import oracle

Table("some_table", metadata, Column("value", oracle.FLOAT(126)))
```

二进制精度值 126 等同于使用 `DOUBLE_PRECISION` 数据类型，而值 63 相当于使用 `REAL` 数据类型。其他精度值是特定于 `FLOAT` 类型本身的。

SQLAlchemy `Float` 数据类型也接受“precision”参数，但这是十进制精度，Oracle 不接受。与其试图猜测转换，Oracle 方言现在将在针对 Oracle 后端使用带有精度值的 `Float` 时引发一个信息性错误。要为支持的后端指定带有显式精度值的 `Float` 数据类型，同时还支持其他后端，请使用以下方法：`TypeEngine.with_variant()`。

```py
from sqlalchemy.types import Float
from sqlalchemy.dialects import oracle

Table(
    "some_table",
    metadata,
    Column("value", Float(5).with_variant(oracle.FLOAT(16), "oracle")),
)
```

### PostgreSQL 后端的新 RANGE / MULTIRANGE 支持和更改

对于 psycopg2、psycopg3 和 asyncpg 方言，已完全实现了 RANGE / MULTIRANGE 支持。新的支持使用了一个新的 SQLAlchemy 特定的 `Range` 对象，该对象对不同的后端是不可知的，不需要使用特定于后端的导入或扩展步骤。对于多范围支持，使用 `Range` 对象的列表。

使用之前的 psycopg2 特定类型的代码应该修改为使用 `Range`，这提供了一个兼容的接口。

`Range` 对象还具有与 PostgreSQL 相同的比较支持。目前已实现的是 `Range.contains()` 和 `Range.contained_by()` 方法，其工作方式与 PostgreSQL 中的 `@>` 和 `<@` 相同。将来的版本可能会添加更多的运算符支持。

请查看范围和多范围类型的文档，了解如何使用这个新功能的背景知识。

另请参阅

范围和多范围类型

[#7156](https://www.sqlalchemy.org/trac/ticket/7156) [#8706](https://www.sqlalchemy.org/trac/ticket/8706)

### PostgreSQL 上的 `match()` 运算符使用 `plainto_tsquery()` 而不是 `to_tsquery()`

`Operators.match()` 函数现在在 PostgreSQL 后端上呈现为 `col @@ plainto_tsquery(expr)`，而不是 `col @@ to_tsquery()`。`plainto_tsquery()` 接受纯文本，而 `to_tsquery()` 接受专用的查询符号，因此与其他后端的兼容性较差。

所有 PostgreSQL 搜索函数和运算符都可以通过使用 `func` 来生成 PostgreSQL 特定的函数和 `Operators.bool_op()`（`Operators.op()` 的布尔类型版本）来生成任意运算符，方式与之前的版本中可用的方式相同。请参阅全文搜索中的示例。

现有的使用`Operators.match()`内置 PG 指令的 SQLAlchemy 项目应直接使用`func.to_tsquery()`。要以与 1.4 中相同的形式呈现 SQL，请参阅使用 match() 进行简单纯文本匹配的版本说明。

[#7086](https://www.sqlalchemy.org/trac/ticket/7086)
