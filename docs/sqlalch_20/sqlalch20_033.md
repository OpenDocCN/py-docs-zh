# 基本关系模式

> 原文：[`docs.sqlalchemy.org/en/20/orm/basic_relationships.html`](https://docs.sqlalchemy.org/en/20/orm/basic_relationships.html)

本节通过基本关系模式的快速概述，使用基于`Mapped`注释类型的声明性样式映射来进行说明。

每个以下章节的设置如下：

```py
from __future__ import annotations
from typing import List

from sqlalchemy import ForeignKey
from sqlalchemy import Integer
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass
```

## 声明式与命令式形式的对比

随着 SQLAlchemy 的发展，不同的 ORM 配置样式已经出现。在本节和其他使用带有注释的声明性映射的示例中，相应的非注释形式应该使用所需的类或字符串类名作为传递给`relationship()`的第一个参数。下面的示例说明了本文档中使用的形式，这是一个完全使用 [**PEP 484**](https://peps.python.org/pep-0484/) 注释的声明性示例，其中 `relationship()` 构造还从 `Mapped` 注释中派生出目标类和集合类型，这是 SQLAlchemy 声明式映射的最现代形式：

```py
class Parent(Base):
    __tablename__ = "parent_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    children: Mapped[List["Child"]] = relationship(back_populates="parent")

class Child(Base):
    __tablename__ = "child_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    parent_id: Mapped[int] = mapped_column(ForeignKey("parent_table.id"))
    parent: Mapped["Parent"] = relationship(back_populates="children")
```

相比之下，使用不带注释的声明式映射是更加“经典”的映射形式，其中`relationship()`要求直接传递所有参数，就像下面的示例中所示：

```py
class Parent(Base):
    __tablename__ = "parent_table"

    id = mapped_column(Integer, primary_key=True)
    children = relationship("Child", back_populates="parent")

class Child(Base):
    __tablename__ = "child_table"

    id = mapped_column(Integer, primary_key=True)
    parent_id = mapped_column(ForeignKey("parent_table.id"))
    parent = relationship("Parent", back_populates="children")
```

最后，使用命令式映射，这是 SQLAlchemy 在声明式之前的原始映射形式（尽管仍然是一小部分用户偏爱的形式），以上配置看起来如下：

```py
registry.map_imperatively(
    Parent,
    parent_table,
    properties={"children": relationship("Child", back_populates="parent")},
)

registry.map_imperatively(
    Child,
    child_table,
    properties={"parent": relationship("Parent", back_populates="children")},
)
```

此外，非注释映射的默认集合样式是`list`。要在没有注释的情况下使用`set`或其他集合，请使用`relationship.collection_class`参数进行指定：

```py
class Parent(Base):
    __tablename__ = "parent_table"

    id = mapped_column(Integer, primary_key=True)
    children = relationship("Child", collection_class=set, ...)
```

关于`relationship()`的集合配置的详细信息，请参阅自定义集合访问。

根据需要将带有注释和不带注释 / 命令式样式之间的其他差异进行说明。

## 一对多

一对多关系在子表上放置一个引用父表的外键。然后在父表上指定`relationship()`，表示引用子项的集合：

```py
class Parent(Base):
    __tablename__ = "parent_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    children: Mapped[List["Child"]] = relationship()

class Child(Base):
    __tablename__ = "child_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    parent_id: Mapped[int] = mapped_column(ForeignKey("parent_table.id"))
```

要在一对多关系中建立双向关系，其中“反向”方是多对一，请指定一个额外的`relationship()`并使用`relationship.back_populates`参数将两者连接起来，使用每个`relationship()`的属性名称作为另一个`relationship.back_populates`上的值：

```py
class Parent(Base):
    __tablename__ = "parent_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    children: Mapped[List["Child"]] = relationship(back_populates="parent")

class Child(Base):
    __tablename__ = "child_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    parent_id: Mapped[int] = mapped_column(ForeignKey("parent_table.id"))
    parent: Mapped["Parent"] = relationship(back_populates="children")
```

`Child`将获得一个具有多对一语义的`parent`属性。

### 使用集合、列表或其他集合类型进行一对多关系

使用带注释的声明性映射，`relationship()`所使用的集合类型是从传递给`Mapped`容器类型的集合类型派生出来的。前一节中的示例可以编写为使用`set`而不是`list`作为`Parent.children`集合，使用`Mapped[Set["Child"]]`：

```py
class Parent(Base):
    __tablename__ = "parent_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    children: Mapped[Set["Child"]] = relationship(back_populates="parent")
```

在使用非带注释形式的映射时，可以通过`relationship.collection_class`参数传递要用作集合的 Python 类。

另请参阅

自定义集合访问 - 包含了对集合配置的进一步细节，包括一些将`relationship()`映射到字典的技巧。

### 配置一对多的删除行为

往往情况下，当它们所属的`Parent`被删除时，所有的`Child`对象都应该被删除。为了配置这种行为，使用在 delete 中描述的`delete`级联选项。另一个选项是，当`Child`对象与其父对象解除关联时，可以将`Child`对象本身删除。该行为在 delete-orphan 中描述。

另请参阅

delete

使用 ORM 关联的外键 ON DELETE cascade

delete-orphan  ## 多对一

多对一（Many to one）在父表中放置一个外键，指向子表。`relationship()`在父表上声明，在此将创建一个新的标量持有属性：

```py
class Parent(Base):
    __tablename__ = "parent_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    child_id: Mapped[int] = mapped_column(ForeignKey("child_table.id"))
    child: Mapped["Child"] = relationship()

class Child(Base):
    __tablename__ = "child_table"

    id: Mapped[int] = mapped_column(primary_key=True)
```

上面的例子展示了假定非空行为的多对一关系；下一节，可空的多对一（Nullable Many-to-One），说明了可空版本。

双向行为通过添加第二个 `relationship()` 并在两个方向上应用 `relationship.back_populates` 参数来实现，在另一个 `relationship()` 的属性名称作为 `relationship.back_populates` 的值：

```py
class Parent(Base):
    __tablename__ = "parent_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    child_id: Mapped[int] = mapped_column(ForeignKey("child_table.id"))
    child: Mapped["Child"] = relationship(back_populates="parents")

class Child(Base):
    __tablename__ = "child_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    parents: Mapped[List["Parent"]] = relationship(back_populates="child")
```

### 可空的多对一（Nullable Many-to-One）

在上述例子中，`Parent.child` 关系未被类型化为允许 `None`；这源于 `Parent.child_id` 列本身不可为空，因为它使用 `Mapped[int]` 类型。如果我们希望 `Parent.child` 是**可空**的多对一关系，我们可以将 `Parent.child_id` 和 `Parent.child` 都设置为 `Optional[]`，在这种情况下，配置将如下所示：

```py
from typing import Optional

class Parent(Base):
    __tablename__ = "parent_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    child_id: Mapped[Optional[int]] = mapped_column(ForeignKey("child_table.id"))
    child: Mapped[Optional["Child"]] = relationship(back_populates="parents")

class Child(Base):
    __tablename__ = "child_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    parents: Mapped[List["Parent"]] = relationship(back_populates="child")
```

上面，DDL 中将创建 `Parent.child_id` 列以允许 `NULL` 值。当使用 `mapped_column()` 与显式类型声明时，指定 `child_id: Mapped[Optional[int]]` 等效于在 `Column` 上设置 `Column.nullable` 为 `True`，而 `child_id: Mapped[int]` 等效于将其设置为 `False`。有关此行为的背景，请参见 mapped_column() 从 Mapped 注释中派生数据类型和可为空性。

提示

如果使用 Python 3.10 或更高版本，[**PEP 604**](https://peps.python.org/pep-0604/) 语法更方便，可以使用 `| None` 指示可选类型，与[**PEP 563**](https://peps.python.org/pep-0563/)延迟注释评估结合使用，这样就不需要使用带字符串引号的类型，如下所示：

```py
from __future__ import annotations

class Parent(Base):
    __tablename__ = "parent_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    child_id: Mapped[int | None] = mapped_column(ForeignKey("child_table.id"))
    child: Mapped[Child | None] = relationship(back_populates="parents")

class Child(Base):
    __tablename__ = "child_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    parents: Mapped[List[Parent]] = relationship(back_populates="child")
```  ## 一对一（One To One）

一对一（One To One）在外键视角上本质上是一对多（One To Many）关系，但表示任何时候只会有一行引用特定父行。

当使用带有`Mapped`注释的映射时，通过在关系的两端都应用非集合类型的`Mapped`注释来实现“一对一”约定，这将使 ORM 意识到不应在任一侧使用集合，就像下面的示例一样：

```py
class Parent(Base):
    __tablename__ = "parent_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    child: Mapped["Child"] = relationship(back_populates="parent")

class Child(Base):
    __tablename__ = "child_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    parent_id: Mapped[int] = mapped_column(ForeignKey("parent_table.id"))
    parent: Mapped["Parent"] = relationship(back_populates="child")
```

在上述情况中，当我们加载一个`Parent`对象时，`Parent.child`属性将引用单个`Child`对象而不是集合。如果我们用一个新的`Child`对象替换`Parent.child`的值，ORM 的工作单元过程将用新的对象替换以前的对象，将以前的`child.parent_id`列默认设置为 NULL，除非设置了特定的级联行为。

提示

正如之前提到的，ORM 将“一对一”模式视为一种约定，其中它假设当它加载`Parent.child`属性时，将只返回一行。如果返回多行，ORM 将发出警告。

但是，上述关系的`Child.parent`一侧仍然保持为“多对一”关系。单独使用它，它将无法检测到分配超过一个`Child`的情况，除非设置了`relationship.single_parent`参数，这可能很有用：

```py
class Child(Base):
    __tablename__ = "child_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    parent_id: Mapped[int] = mapped_column(ForeignKey("parent_table.id"))
    parent: Mapped["Parent"] = relationship(back_populates="child", single_parent=True)
```

在设置此参数之外，“一对多”侧（在这里按照惯例是一对一）也无法可靠地检测到一个`Parent`关联多个`Child`的情况，例如，多个`Child`对象处于挂起状态且不在数据库中持久存在的情况。

是否使用`relationship.single_parent`，建议数据库模式包含一个唯一约束，以指示`Child.parent_id`列应该是唯一的，以确保在数据库级别上，只有一个`Child`行可以同时引用特定的`Parent`行（有关`__table_args__`元组语法的背景，请参阅声明性表配置）：

```py
from sqlalchemy import UniqueConstraint

class Child(Base):
    __tablename__ = "child_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    parent_id: Mapped[int] = mapped_column(ForeignKey("parent_table.id"))
    parent: Mapped["Parent"] = relationship(back_populates="child")

    __table_args__ = (UniqueConstraint("parent_id"),)
```

新版本 2.0 中：`relationship()`构造可以从给定的`Mapped`注释中派生出`relationship.uselist`参数的有效值。

### 将非注释配置的 uselist 参数设置为 False

当使用没有 `Mapped` 注解的 `relationship()` 时，可以通过在通常是“多”的一侧将 `relationship.uselist` 参数设置为 `False` 来启用一对一模式，如下所示的非注解式声明配置：

```py
class Parent(Base):
    __tablename__ = "parent_table"

    id = mapped_column(Integer, primary_key=True)
    child = relationship("Child", uselist=False, back_populates="parent")

class Child(Base):
    __tablename__ = "child_table"

    id = mapped_column(Integer, primary_key=True)
    parent_id = mapped_column(ForeignKey("parent_table.id"))
    parent = relationship("Parent", back_populates="child")
```  ## 多对多

Many to Many 在两个类之间添加了一个关联表。关联表几乎总是作为一个核心 `Table` 对象或其他核心可选择的对象，比如一个 `Join` 对象来给出，并且通过 `relationship()` 函数的 `relationship.secondary` 参数来指定。通常，`Table` 使用与声明基类关联的 `MetaData` 对象，这样 `ForeignKey` 指令就可以定位远程表以进行关联：

```py
from __future__ import annotations

from sqlalchemy import Column
from sqlalchemy import Table
from sqlalchemy import ForeignKey
from sqlalchemy import Integer
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

# note for a Core table, we use the sqlalchemy.Column construct,
# not sqlalchemy.orm.mapped_column
association_table = Table(
    "association_table",
    Base.metadata,
    Column("left_id", ForeignKey("left_table.id")),
    Column("right_id", ForeignKey("right_table.id")),
)

class Parent(Base):
    __tablename__ = "left_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    children: Mapped[List[Child]] = relationship(secondary=association_table)

class Child(Base):
    __tablename__ = "right_table"

    id: Mapped[int] = mapped_column(primary_key=True)
```

提示

上面的“关联表”中已经建立了指向关系两侧实体表的外键约束。`association.left_id` 和 `association.right_id` 的每个数据类型通常从引用表中推断出，并且可以省略。虽然 SQLAlchemy 并不强制要求，但也**建议**将引用两个实体表的列建立在**唯一约束**或更常见的**主键约束**中；这样可以确保无论应用程序端出现什么问题，都不会在表中持久化重复的行：

```py
association_table = Table(
    "association_table",
    Base.metadata,
    Column("left_id", ForeignKey("left_table.id"), primary_key=True),
    Column("right_id", ForeignKey("right_table.id"), primary_key=True),
)
```

### 设置双向 Many-to-many

对于双向关系，关系的两侧都包含一个集合。使用 `relationship.back_populates` 进行指定，并且对于每个 `relationship()` 指定共同的关联表：

```py
from __future__ import annotations

from sqlalchemy import Column
from sqlalchemy import Table
from sqlalchemy import ForeignKey
from sqlalchemy import Integer
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

association_table = Table(
    "association_table",
    Base.metadata,
    Column("left_id", ForeignKey("left_table.id"), primary_key=True),
    Column("right_id", ForeignKey("right_table.id"), primary_key=True),
)

class Parent(Base):
    __tablename__ = "left_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    children: Mapped[List[Child]] = relationship(
        secondary=association_table, back_populates="parents"
    )

class Child(Base):
    __tablename__ = "right_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    parents: Mapped[List[Parent]] = relationship(
        secondary=association_table, back_populates="children"
    )
```

### 使用延迟评估形式来处理“次要”参数

`relationship.secondary`参数还接受两种不同的“延迟评估”形式，包括字符串表名以及 lambda 可调用。请参阅使用“secondary”参数的延迟评估形式进行多对多关系部分了解背景和示例。

### 使用集合（Sets）、列表（Lists）或其他集合类型进行多对多关系

为多对多关系配置集合的方式与一对多完全相同，如在使用集合（Sets）、列表（Lists）或其他集合类型进行一对多关系中描述的那样。对于使用`Mapped`进行注释的映射，可以通过`Mapped`泛型类内部使用的集合类型来指示集合，例如`set`：

```py
class Parent(Base):
    __tablename__ = "left_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    children: Mapped[Set["Child"]] = relationship(secondary=association_table)
```

当使用非注释形式包括命令式映射时，如一对多的情况，可以通过`relationship.collection_class`参数传递要用作集合的 Python 类。

另请参阅

自定义集合访问 - 包含有关集合配置的进一步详细信息，包括一些将`relationship()`映射到字典的技术。

### 从多对多表中删除行

`relationship.secondary`参数的一个独特行为是，此处指定的`Table`会自动受到 INSERT 和 DELETE 语句的影响，因为对象被添加或从集合中删除。**无需手动从此表中删除**。从集合中删除记录的操作将在刷新时将行删除：

```py
# row will be deleted from the "secondary" table
# automatically
myparent.children.remove(somechild)
```

经常出现的一个问题是，当直接将子对象传递给`Session.delete()`时，如何删除“secondary”表中的行：

```py
session.delete(somechild)
```

这里有几种可能性：

+   如果从`Parent`到`Child`有一个`relationship()`，但没有反向关系将特定的`Child`链接到每个`Parent`，SQLAlchemy 将不会意识到在删除此特定的`Child`对象时，需要维护将其链接到`Parent`的“secondary”表。不会删除“secondary”表。

+   如果存在将特定的`Child`链接到每个`Parent`的关系，假设它称为`Child.parents`，SQLAlchemy 默认将加载`Child.parents`集合以定位所有`Parent`对象，并从建立此链接的“secondary”表中删除每行。请注意，此关系不需要是双向的；SQLAlchemy 严格查看与要删除的`Child`对象关联的每个`relationship()`。

+   这里的一个性能更高的选项是使用数据库使用的外键 ON DELETE CASCADE 指令。假设数据库支持此功能，数据库本身可以被设置为在删除“child”中的引用行时自动删除“secondary”表中的行。在这种情况下，SQLAlchemy 可以通过在`relationship()`上使用`relationship.passive_deletes`指令来指示放弃主动加载`Child.parents`集合；有关此操作的更多详细信息，请参阅使用外键 ON DELETE cascade 处理 ORM 关系。

再次注意，这些行为仅与与`relationship()`一起使用的`relationship.secondary`选项相关。如果处理的是显式映射的关联表，并且不存在于相关`relationship()`的`relationship.secondary`选项中，那么可以使用级联规则来自动删除实体以响应相关实体的删除 - 有关此功能的信息，请参阅级联。

另请参阅

使用级联删除处理多对多关系

使用外键 ON DELETE 处理多对多关系  ## 关联对象

关联对象模式是一种与多对多模式相异的变体：当一个关联表包含除了与父表和子表（或左表和右表）是外键关系的列之外的其他列时，最理想的情况是将这些列映射到它们自己的 ORM 映射类中。这个映射类被映射到了 `Table` ，否则会在使用多对多模式时被标记为 `relationship.secondary`。

在关联对象模式中，不使用 `relationship.secondary` 参数；相反，一个类直接映射到关联表。然后，两个独立的 `relationship()` 构造将首先父侧通过一对多连接到映射的关联类，然后通过多对一将映射的关联类连接到子侧，以形成从父对象到关联对象到子对象的单向关联对象关系。对于双向关系，使用四个 `relationship()` 构造将映射的关联类链接到父对象和子对象，以在两个方向上建立联系。

下面的示例说明了一个新的类 `Association` ，它映射到了名为 `association` 的 `Table` ，这个表现在包含了一个名为 `extra_data` 的额外列，这是一个字符串值，与 `Parent` 和 `Child` 之间的每个关联一起存储。通过将表映射到一个显式类，从 `Parent` 到 `Child` 的原始访问明确地使用了 `Association`：

```py
from typing import Optional

from sqlalchemy import ForeignKey
from sqlalchemy import Integer
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

class Association(Base):
    __tablename__ = "association_table"
    left_id: Mapped[int] = mapped_column(ForeignKey("left_table.id"), primary_key=True)
    right_id: Mapped[int] = mapped_column(
        ForeignKey("right_table.id"), primary_key=True
    )
    extra_data: Mapped[Optional[str]]
    child: Mapped["Child"] = relationship()

class Parent(Base):
    __tablename__ = "left_table"
    id: Mapped[int] = mapped_column(primary_key=True)
    children: Mapped[List["Association"]] = relationship()

class Child(Base):
    __tablename__ = "right_table"
    id: Mapped[int] = mapped_column(primary_key=True)
```

为了说明双向版本，我们增加了两个 `relationship()` 构造，通过 `relationship.back_populates` 与现有的构造相连：

```py
from typing import Optional

from sqlalchemy import ForeignKey
from sqlalchemy import Integer
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

class Association(Base):
    __tablename__ = "association_table"
    left_id: Mapped[int] = mapped_column(ForeignKey("left_table.id"), primary_key=True)
    right_id: Mapped[int] = mapped_column(
        ForeignKey("right_table.id"), primary_key=True
    )
    extra_data: Mapped[Optional[str]]
    child: Mapped["Child"] = relationship(back_populates="parents")
    parent: Mapped["Parent"] = relationship(back_populates="children")

class Parent(Base):
    __tablename__ = "left_table"
    id: Mapped[int] = mapped_column(primary_key=True)
    children: Mapped[List["Association"]] = relationship(back_populates="parent")

class Child(Base):
    __tablename__ = "right_table"
    id: Mapped[int] = mapped_column(primary_key=True)
    parents: Mapped[List["Association"]] = relationship(back_populates="child")
```

在其直接形式中使用关联模式需要在将子对象附加到父对象之前将其与关联实例关联起来；同样，从父对象到子对象的访问通过关联对象进行：

```py
# create parent, append a child via association
p = Parent()
a = Association(extra_data="some data")
a.child = Child()
p.children.append(a)

# iterate through child objects via association, including association
# attributes
for assoc in p.children:
    print(assoc.extra_data)
    print(assoc.child)
```

为了增强关联对象模式，使直接访问 `Association` 对象是可选的，SQLAlchemy 提供了 Association Proxy 扩展。这个扩展允许配置属性，这些属性将通过单一访问访问两个“跳”，一个“跳”到关联对象，第二个“跳”到目标属性。

另见

关联代理 - 允许父级和子级之间进行直接“多对多”样式访问，用于三类关联对象映射。

警告

避免直接混合使用关联对象模式和多对多模式，因为这会产生数据可能以不一致的方式读取和写入的情况，而无需特殊步骤；关联代理通常用于提供更简洁的访问。有关此组合引入的注意事项的更详细背景，请参阅下一节结合关联对象与多对多访问模式。

### 结合关联对象与多对多访问模式

如前一节所述，关联对象模式不会自动与同时针对相同表/列使用多对多模式的情况集成。由此可见，读操作可能返回冲突的数据，写操作也可能尝试刷新冲突的更改，导致完整性错误或意外的插入或删除。

为了说明，下面的示例配置了`Parent`和`Child`之间的双向多对多关系，通过`Parent.children`和`Child.parents`。同时，还配置了一个关联对象关系，即`Parent.child_associations -> Association.child`和`Child.parent_associations -> Association.parent`之间的关系：

```py
from typing import Optional

from sqlalchemy import ForeignKey
from sqlalchemy import Integer
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

class Association(Base):
    __tablename__ = "association_table"

    left_id: Mapped[int] = mapped_column(ForeignKey("left_table.id"), primary_key=True)
    right_id: Mapped[int] = mapped_column(
        ForeignKey("right_table.id"), primary_key=True
    )
    extra_data: Mapped[Optional[str]]

    # association between Assocation -> Child
    child: Mapped["Child"] = relationship(back_populates="parent_associations")

    # association between Assocation -> Parent
    parent: Mapped["Parent"] = relationship(back_populates="child_associations")

class Parent(Base):
    __tablename__ = "left_table"

    id: Mapped[int] = mapped_column(primary_key=True)

    # many-to-many relationship to Child, bypassing the `Association` class
    children: Mapped[List["Child"]] = relationship(
        secondary="association_table", back_populates="parents"
    )

    # association between Parent -> Association -> Child
    child_associations: Mapped[List["Association"]] = relationship(
        back_populates="parent"
    )

class Child(Base):
    __tablename__ = "right_table"

    id: Mapped[int] = mapped_column(primary_key=True)

    # many-to-many relationship to Parent, bypassing the `Association` class
    parents: Mapped[List["Parent"]] = relationship(
        secondary="association_table", back_populates="children"
    )

    # association between Child -> Association -> Parent
    parent_associations: Mapped[List["Association"]] = relationship(
        back_populates="child"
    )
```

当使用此 ORM 模型进行更改时，在 Python 中对`Parent.children`进行的更改将不会与对`Parent.child_associations`或`Child.parent_associations`进行的更改协调；虽然所有这些关系本身都将继续正常运行，但在一个关系上的更改不会显示在另一个关系中，直到`Session`过期，通常在`Session.commit()`之后会自动发生。

另外，如果进行了冲突的更改，例如同时添加了一个新的`Association`对象，同时将相同的相关`Child`附加到`Parent.children`，则在工作单元刷新过程进行时将引发完整性错误，如下例所示：

```py
p1 = Parent()
c1 = Child()
p1.children.append(c1)

# redundant, will cause a duplicate INSERT on Association
p1.child_associations.append(Association(child=c1))
```

直接将`Child`附加到`Parent.children`中也意味着在`association`表中创建行，而不指示`association.extra_data`列的任何值，该值将接收`NULL`作为其值。

如果你知道你在做什么，像上面这样使用映射是可以的；在很少使用“关联对象”模式的情况下使用多对多关系可能是有充分理由的，因为通过单个多对多关系加载关系更容易，这也可以优化“次要”表在 SQL 语句中的使用效果，与两个对显式关联类的分开关系的使用相比略有优势。至少将 `relationship.viewonly` 参数应用于“次要”关系是一个好主意，以避免发生冲突的更改，同时防止将 `NULL` 写入额外的关联列，如下所示：

```py
class Parent(Base):
    __tablename__ = "left_table"

    id: Mapped[int] = mapped_column(primary_key=True)

    # many-to-many relationship to Child, bypassing the `Association` class
    children: Mapped[List["Child"]] = relationship(
        secondary="association_table", back_populates="parents", viewonly=True
    )

    # association between Parent -> Association -> Child
    child_associations: Mapped[List["Association"]] = relationship(
        back_populates="parent"
    )

class Child(Base):
    __tablename__ = "right_table"

    id: Mapped[int] = mapped_column(primary_key=True)

    # many-to-many relationship to Parent, bypassing the `Association` class
    parents: Mapped[List["Parent"]] = relationship(
        secondary="association_table", back_populates="children", viewonly=True
    )

    # association between Child -> Association -> Parent
    parent_associations: Mapped[List["Association"]] = relationship(
        back_populates="child"
    )
```

上述映射不会将任何更改写入到数据库的 `Parent.children` 或 `Child.parents` 中，从而防止冲突写入。然而，如果在同一个事务或 `Session` 中对视图集合进行读取的同时对 `Parent.children` 或 `Child.parents` 进行读取，则对 `Parent.children` 或 `Child.parents` 的读取不一定会匹配从 `Parent.child_associations` 或 `Child.parent_associations` 中读取的数据。如果很少使用关联对象关系，并且对访问多对多集合的代码进行了精心组织以避免过时的读取（在极端情况下，直接使用 `Session.expire()` 来使集合在当前事务内刷新），则该模式可能是可行的。

上述模式的一个流行替代方案是，直接的多对多 `Parent.children` 和 `Child.parents` 关系被替换为一个扩展，该扩展将通过 `Association` 类透明代理，同时从 ORM 的角度保持一切一致。这个扩展被称为关联代理。

另请参阅

关联代理 - 允许父对象和子对象之间直接“多对多”样式的访问，用于三类关联对象映射。## 关系参数的延迟评估

在前面的部分中，大多数示例都说明了各种`relationship()` 构造是如何使用字符串名称而不是类本身来引用它们的目标类的，比如当使用`Mapped`时，会生成一个仅在运行时存在的字符串引用：

```py
class Parent(Base):
    # ...

    children: Mapped[List["Child"]] = relationship(back_populates="parent")

class Child(Base):
    # ...

    parent: Mapped["Parent"] = relationship(back_populates="children")
```

类似地，当使用非注释形式，如非注释的声明式或命令式映射时，`relationship()`构造也直接支持字符串名称：

```py
registry.map_imperatively(
    Parent,
    parent_table,
    properties={"children": relationship("Child", back_populates="parent")},
)

registry.map_imperatively(
    Child,
    child_table,
    properties={"parent": relationship("Parent", back_populates="children")},
)
```

这些字符串名称在映射解析阶段解析为类，这是一个内部过程，通常在所有映射都被定义后发生，并且通常由映射本身的第一次使用触发。`registry`对象是存储这些名称并将其解析为它们所引用的映射类的容器。

除了`relationship()`的主类参数之外，还可以指定依赖于尚未定义类中存在的列的其他参数，这些参数可以是 Python 函数，或更常见的是字符串。对于这些参数中的大多数（除了主参数之外），字符串输入 **使用 Python 内置的 eval()函数求值为 Python 表达式**，因为它们旨在接收完整的 SQL 表达式。

警告

由于 Python 的`eval()`函数用于解释传递给`relationship()`映射配置构造函数的后期评估的字符串参数，因此这些参数 **不应该** 被重新用于接收不受信任的用户输入；`eval()`对不受信任的用户输入 **不安全**。

此评估中可用的完整命名空间包括为此声明基类映射的所有类，以及`sqlalchemy`包的内容，包括表达式函数如`desc()`和`sqlalchemy.sql.functions.func`：

```py
class Parent(Base):
    # ...

    children: Mapped[List["Child"]] = relationship(
        order_by="desc(Child.email_address)",
        primaryjoin="Parent.id == Child.parent_id",
    )
```

对于包含多个模块都包含相同名称类的情况，字符串类名也可以在任何这些字符串表达式中指定为模块限定路径：

```py
class Parent(Base):
    # ...

    children: Mapped[List["myapp.mymodel.Child"]] = relationship(
        order_by="desc(myapp.mymodel.Child.email_address)",
        primaryjoin="myapp.mymodel.Parent.id == myapp.mymodel.Child.parent_id",
    )
```

在类似上面的示例中，传递给`Mapped`的字符串也可以通过直接将类位置字符串传递给`relationship.argument`来消除特定类参数的歧义。下面演示了对`Child`进行仅类型导入，并与将在`registry`中搜索正确名称的目标类的运行时说明符相结合的示例：

```py
import typing

if typing.TYPE_CHECKING:
    from myapp.mymodel import Child

class Parent(Base):
    # ...

    children: Mapped[List["Child"]] = relationship(
        "myapp.mymodel.Child",
        order_by="desc(myapp.mymodel.Child.email_address)",
        primaryjoin="myapp.mymodel.Parent.id == myapp.mymodel.Child.parent_id",
    )
```

合格的路径可以是任何消除名称歧义的部分路径。例如，为了消除`myapp.model1.Child`和`myapp.model2.Child`之间的歧义，我们可以指定`model1.Child`或`model2.Child`：

```py
class Parent(Base):
    # ...

    children: Mapped[List["Child"]] = relationship(
        "model1.Child",
        order_by="desc(mymodel1.Child.email_address)",
        primaryjoin="Parent.id == model1.Child.parent_id",
    )
```

`relationship()` 构造还接受 Python 函数或 lambda 作为这些参数的输入。Python 函数式方法可能如下所示：

```py
import typing

from sqlalchemy import desc

if typing.TYPE_CHECKING:
    from myapplication import Child

def _resolve_child_model():
    from myapplication import Child

    return Child

class Parent(Base):
    # ...

    children: Mapped[List["Child"]] = relationship(
        _resolve_child_model,
        order_by=lambda: desc(_resolve_child_model().email_address),
        primaryjoin=lambda: Parent.id == _resolve_child_model().parent_id,
    )
```

接受 Python 函数/lambda 或将传递给 `eval()` 的字符串的完整参数列表为：

+   `relationship.order_by`

+   `relationship.primaryjoin`

+   `relationship.secondaryjoin`

+   `relationship.secondary`

+   `relationship.remote_side`

+   `relationship.foreign_keys`

+   `relationship._user_defined_foreign_keys`

警告

如前所述，上述参数`relationship()`将**作为 Python 代码表达式使用 eval() 进行评估。不要向这些参数传递不受信任的输入。**

### 在声明后向映射类添加关系

还应注意，与向现有的 Declarative 映射类添加附加列中描述的类似，任何`MapperProperty` 构造都可以随时添加到声明基本映射中（请注意，在此上下文中不支持注释形式）。如果我们希望在 `Address` 类可用后实现此 `relationship()`，我们也可以稍后应用它：

```py
# first, module A, where Child has not been created yet,
# we create a Parent class which knows nothing about Child

class Parent(Base): ...

# ... later, in Module B, which is imported after module A:

class Child(Base): ...

from module_a import Parent

# assign the User.addresses relationship as a class variable.  The
# declarative base class will intercept this and map the relationship.
Parent.children = relationship(Child, primaryjoin=Child.parent_id == Parent.id)
```

对于 ORM 映射列一样，`Mapped` 注释类型没有参与此操作的能力；因此，相关类必须直接在 `relationship()` 构造中指定，可以作为类本身、类的字符串名称或返回目标类引用的可调用函数。

注意

对于 ORM 映射列一样，只有当“声明基类”类被使用时，即用户定义的 `DeclarativeBase` 的子类或由 `declarative_base()` 或 `registry.generate_base()` 返回的动态生成的类时，将映射属性分配给已经映射的类才会正常工作。这个“基”类包含一个实现特殊 `__setattr__()` 方法的 Python 元类，该方法拦截这些操作。

如果类使用装饰器如`registry.mapped()`或者使用命令式函数如`registry.map_imperatively()`进行映射，那么将类映射属性运行时分配给映射类 **不会** 起作用。### 对于多对多关系使用后期评估的形式

多对多关系使用 `relationship.secondary` 参数，通常指示一个参考到通常非映射的 `Table` 对象或其他 Core 可选择对象。通常使用 lambda 可调用对象进行延迟评估。

对于给定的示例在多对多关系中，如果我们假设`association_table` `Table`对象会在模块中的后面定义，我们可以使用 lambda 来编写`relationship()`如下：

```py
class Parent(Base):
    __tablename__ = "left_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    children: Mapped[List["Child"]] = relationship(
        "Child", secondary=lambda: association_table
    )
```

对于也是**有效的 Python 标识符**的表名的快捷方式，`relationship.secondary` 参数也可以作为字符串传递，其中解析工作通过将字符串作为 Python 表达式进行评估，简单标识符名称与当前 `registry` 引用的相同名称 `Table` 对象链接到相同的 `MetaData` 集合中。

在下面的示例中，表达式 `"association_table"` 被视为一个名为`"association_table"`的变量，该变量相对于 `MetaData` 集合中的表名进行解析：

```py
class Parent(Base):
    __tablename__ = "left_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    children: Mapped[List["Child"]] = relationship(secondary="association_table")
```

注意

当作为字符串传递时，传递给`relationship.secondary`的名称**必须是有效的 Python 标识符**，以字母开头，并且只包含字母数字字符或下划线。其他字符如短划线等将被解释为 Python 运算符，不会解析为给定的名称。请考虑使用 lambda 表达式而不是字符串以提高清晰度。

警告

当作为字符串传递时，`relationship.secondary`参数将使用 Python 的`eval()`函数进行解释，即使它通常是表的名称。**不要传递不可信的输入给这个字符串**。

## 声明式 vs. 命令式形式

随着 SQLAlchemy 的发展，不同的 ORM 配置风格已经出现。对于本节及其他使用带注解的声明式映射的示例，相应的非带注解形式应使用所需的类或字符串类名作为传递给`relationship()`的第一个参数。下面的示例说明了本文档中使用的形式，这是一个完全声明式的示例，使用[**PEP 484**](https://peps.python.org/pep-0484/)注解，其中`relationship()`构造还从`Mapped`注解中推断目标类和集合类型，这是 SQLAlchemy 声明式映射的最现代形式：

```py
class Parent(Base):
    __tablename__ = "parent_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    children: Mapped[List["Child"]] = relationship(back_populates="parent")

class Child(Base):
    __tablename__ = "child_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    parent_id: Mapped[int] = mapped_column(ForeignKey("parent_table.id"))
    parent: Mapped["Parent"] = relationship(back_populates="children")
```

相比之下，使用不带注解的声明式映射更像是更“经典”的映射形式，其中`relationship()`需要直接传递所有参数，如下例所示：

```py
class Parent(Base):
    __tablename__ = "parent_table"

    id = mapped_column(Integer, primary_key=True)
    children = relationship("Child", back_populates="parent")

class Child(Base):
    __tablename__ = "child_table"

    id = mapped_column(Integer, primary_key=True)
    parent_id = mapped_column(ForeignKey("parent_table.id"))
    parent = relationship("Parent", back_populates="children")
```

最后，使用命令式映射，这是在声明式之前 SQLAlchemy 的原始映射形式（尽管仍然被少数用户偏爱），以上配置如下所示：

```py
registry.map_imperatively(
    Parent,
    parent_table,
    properties={"children": relationship("Child", back_populates="parent")},
)

registry.map_imperatively(
    Child,
    child_table,
    properties={"parent": relationship("Parent", back_populates="children")},
)
```

此外，对于非带注解的映射，默认的集合样式是`list`。要使用`set`或其他集合而不带注解，可以使用`relationship.collection_class`参数进行指示：

```py
class Parent(Base):
    __tablename__ = "parent_table"

    id = mapped_column(Integer, primary_key=True)
    children = relationship("Child", collection_class=set, ...)
```

有关`relationship()`的集合配置详细信息，请参阅自定义集合访问。

根据需要，将注意到注释和非注释/命令式样式之间的其他差异。

## 一对多

一对多关系在子表上放置一个外键，引用父表。然后在父表上指定`relationship()`，作为对由子表表示的项目集合的引用： 

```py
class Parent(Base):
    __tablename__ = "parent_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    children: Mapped[List["Child"]] = relationship()

class Child(Base):
    __tablename__ = "child_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    parent_id: Mapped[int] = mapped_column(ForeignKey("parent_table.id"))
```

在一对多关系中建立双向关系时，其中“反向”端是多对一，需要指定一个额外的`relationship()`并使用`relationship.back_populates`参数将两者连接起来，其中使用每个`relationship()`的属性名称作为另一个`relationship.back_populates`的值：

```py
class Parent(Base):
    __tablename__ = "parent_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    children: Mapped[List["Child"]] = relationship(back_populates="parent")

class Child(Base):
    __tablename__ = "child_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    parent_id: Mapped[int] = mapped_column(ForeignKey("parent_table.id"))
    parent: Mapped["Parent"] = relationship(back_populates="children")
```

`Child`将获得一个具有多对一语义的`parent`属性。

### 使用集合、列表或其他集合类型进行一对多关系

使用带注释的声明性映射时，用于`relationship()`的集合类型是从传递给`Mapped`容器类型的集合类型派生的。上一节的示例可以编写为使用`set`而不是`list`来表示`Parent.children`集合，使用`Mapped[Set["Child"]]`：

```py
class Parent(Base):
    __tablename__ = "parent_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    children: Mapped[Set["Child"]] = relationship(back_populates="parent")
```

当使用非注释形式，包括命令式映射时，可以使用`relationship.collection_class`参数传递要用作集合的 Python 类。

另请参见

自定义集合访问 - 包含了更多关于集合配置的细节，包括一些将`relationship()`映射到字典的技术。

### 配置一对多关系的删除行为

通常情况下，当其所属的`Parent`被删除时，所有`Child`对象都应该被删除。为了配置这种行为，使用 delete 中描述的`delete`级联选项。另一个选项是，当`Child`对象与其父对象解除关联时，可以删除`Child`对象本身。这种行为在 delete-orphan 中描述。

另请参见

delete

使用 ORM 关系的外键 ON DELETE 级联

delete-orphan

### 使用集合、列表或其他集合类型进行一对多关系

使用带注释的声明性映射，从传递给`Mapped`容器类型的集合类型派生出用于`relationship()`的集合类型。可以写一个例子，以使用`set`而不是`list`来表示`Parent.children`集合，使用`Mapped[Set["Child"]]`：

```py
class Parent(Base):
    __tablename__ = "parent_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    children: Mapped[Set["Child"]] = relationship(back_populates="parent")
```

在使用非注释形式，包括命令式映射时，可以使用`relationship.collection_class`参数传递要用作集合的 Python 类。

另请参阅

自定义集合访问 - 包含有关集合配置的更多详细信息，包括一些将`relationship()`映射到字典的技术。

### 配置一对多的删除行为

通常情况下，当其所属的`Parent`被删除时，所有的`Child`对象都应该被删除。要配置此行为，可以使用在删除中描述的`delete`级联选项。另一个选项是当`Child`对象与其父对象解除关联时，它本身也可以被删除。这种行为在删除孤儿中描述。

另请参阅

删除

使用 ORM 关系的外键 ON DELETE 级联

删除孤儿

## 多对一

多对一在父表中放置了一个引用子表的外键。在父级上声明`relationship()`，将创建一个新的标量持有属性：

```py
class Parent(Base):
    __tablename__ = "parent_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    child_id: Mapped[int] = mapped_column(ForeignKey("child_table.id"))
    child: Mapped["Child"] = relationship()

class Child(Base):
    __tablename__ = "child_table"

    id: Mapped[int] = mapped_column(primary_key=True)
```

上面的示例显示了一个假定为非空的多对一关系；下一节，可空多对一，介绍了一个可空版本。

通过在两个方向添加第二个`relationship()`并在两个方向上应用`relationship.back_populates`参数，使用每个`relationship()`的属性名称作为另一个`relationship.back_populates`的值，实现了双向行为：

```py
class Parent(Base):
    __tablename__ = "parent_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    child_id: Mapped[int] = mapped_column(ForeignKey("child_table.id"))
    child: Mapped["Child"] = relationship(back_populates="parents")

class Child(Base):
    __tablename__ = "child_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    parents: Mapped[List["Parent"]] = relationship(back_populates="child")
```

### 可空多对一

在上述示例中，`Parent.child` 的关系未被类型化为允许 `None`；这是因为 `Parent.child_id` 列本身不可为空，因为它被类型化为 `Mapped[int]`。如果我们想要 `Parent.child` 成为一个**可为空的**多对一关系，我们可以将 `Parent.child_id` 和 `Parent.child` 都设置为 `Optional[]`，在这种情况下，配置将如下所示：

```py
from typing import Optional

class Parent(Base):
    __tablename__ = "parent_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    child_id: Mapped[Optional[int]] = mapped_column(ForeignKey("child_table.id"))
    child: Mapped[Optional["Child"]] = relationship(back_populates="parents")

class Child(Base):
    __tablename__ = "child_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    parents: Mapped[List["Parent"]] = relationship(back_populates="child")
```

在上述代码中，用于 `Parent.child_id` 的列将在 DDL 中被创建以允许 `NULL` 值。在使用 `mapped_column()` 进行显式类型声明时，对 `child_id: Mapped[Optional[int]]` 的规定等效于将 `Column.nullable` 设置为 `True` 在 `Column` 上，而 `child_id: Mapped[int]` 等效于将其设置为 `False`。有关此行为的背景信息，请参阅 mapped_column() 从 Mapped 注释中推导出数据类型和可为空性。

提示

如果使用 Python 3.10 或更高版本，[**PEP 604**](https://peps.python.org/pep-0604/) 语法更方便，可使用 `| None` 来指示可选类型，结合 [**PEP 563**](https://peps.python.org/pep-0563/) 推迟注释评估，以便不需要字符串引号类型，这将如下所示：

```py
from __future__ import annotations

class Parent(Base):
    __tablename__ = "parent_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    child_id: Mapped[int | None] = mapped_column(ForeignKey("child_table.id"))
    child: Mapped[Child | None] = relationship(back_populates="parents")

class Child(Base):
    __tablename__ = "child_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    parents: Mapped[List[Parent]] = relationship(back_populates="child")
``` ### 可为空的多对一关系

在上述示例中，`Parent.child` 的关系未被类型化为允许 `None`；这是因为 `Parent.child_id` 列本身不可为空，因为它被类型化为 `Mapped[int]`。如果我们想要 `Parent.child` 成为一个**可为空的**多对一关系，我们可以将 `Parent.child_id` 和 `Parent.child` 都设置为 `Optional[]`，在这种情况下，配置将如下所示：

```py
from typing import Optional

class Parent(Base):
    __tablename__ = "parent_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    child_id: Mapped[Optional[int]] = mapped_column(ForeignKey("child_table.id"))
    child: Mapped[Optional["Child"]] = relationship(back_populates="parents")

class Child(Base):
    __tablename__ = "child_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    parents: Mapped[List["Parent"]] = relationship(back_populates="child")
```

在上述代码中，用于 `Parent.child_id` 的列将在 DDL 中被创建以允许 `NULL` 值。在使用 `mapped_column()` 进行显式类型声明时，对 `child_id: Mapped[Optional[int]]` 的规定等效于将 `Column.nullable` 设置为 `True` 在 `Column` 上，而 `child_id: Mapped[int]` 等效于将其设置为 `False`。有关此行为的背景信息，请参阅 mapped_column() 从 Mapped 注释中推导出数据类型和可为空性。

提示

如果使用 Python 3.10 或更高版本，[**PEP 604**](https://peps.python.org/pep-0604/) 语法更方便地使用`| None`来指示可选类型，与[**PEP 563**](https://peps.python.org/pep-0563/)推迟的注释评估结合使用，以便不需要字符串引号的类型，会是这样的：

```py
from __future__ import annotations

class Parent(Base):
    __tablename__ = "parent_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    child_id: Mapped[int | None] = mapped_column(ForeignKey("child_table.id"))
    child: Mapped[Child | None] = relationship(back_populates="parents")

class Child(Base):
    __tablename__ = "child_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    parents: Mapped[List[Parent]] = relationship(back_populates="child")
```

## 一对一

One To One 本质上是一个从外键角度来看的一对多关系，但表示在任何时候只会有一行指向特定父行的行。

当使用带注释的映射和`Mapped`时，“一对一”约定通过在关系的两侧应用非集合类型到`Mapped`注释来实现，这将暗示 ORM 不应在任一侧使用集合，如下面的示例所示：

```py
class Parent(Base):
    __tablename__ = "parent_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    child: Mapped["Child"] = relationship(back_populates="parent")

class Child(Base):
    __tablename__ = "child_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    parent_id: Mapped[int] = mapped_column(ForeignKey("parent_table.id"))
    parent: Mapped["Parent"] = relationship(back_populates="child")
```

上面的例子中，当我们加载一个`Parent`对象时，`Parent.child`属性将引用一个单个的`Child`对象而不是一个集合。如果我们用一个新的`Child`对象替换`Parent.child`的值，ORM 的工作单元过程将用新的对象替换之前的对象，将之前的`child.parent_id`列默认设置为 NULL，除非设置了特定的级联行为。

提示

正如前面提到的，ORM 将“一对一”模式视为一种约定，其中它假设当它加载`Parent.child`属性时，它只会得到一行返回。如果返回多行，ORM 将发出警告。

然而，上述关系中的`Child.parent`方面仍然是一个“多对一”的关系。独自使用时，它不会检测到多个`Child`的赋值，除非设置了`relationship.single_parent`参数，这可能是有用的：

```py
class Child(Base):
    __tablename__ = "child_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    parent_id: Mapped[int] = mapped_column(ForeignKey("parent_table.id"))
    parent: Mapped["Parent"] = relationship(back_populates="child", single_parent=True)
```

在设置此参数之外，“一对多”方面（这里按照惯例是一对一）也不会可靠地检测到多个`Child`关联到单个`Parent`的情况，比如多个`Child`对象是挂起的并且不是数据库持久的情况。

是否使用了`relationship.single_parent`，建议数据库模式包括一个唯一约束来指示`Child.parent_id`列应该是唯一的，以确保在数据库级别只有一行`Child`可能指向特定的`Parent`行（有关`__table_args__`元组语法的背景，请参阅声明式表配置）：

```py
from sqlalchemy import UniqueConstraint

class Child(Base):
    __tablename__ = "child_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    parent_id: Mapped[int] = mapped_column(ForeignKey("parent_table.id"))
    parent: Mapped["Parent"] = relationship(back_populates="child")

    __table_args__ = (UniqueConstraint("parent_id"),)
```

新版本 2.0 中：`relationship()` 构造可以从给定的`Mapped`注解推导出`relationship.uselist`参数的有效值。

### 对于非注解配置设置 uselist=False

当在没有使用`Mapped`注解的情况下使用`relationship()`时，可以通过将通常是“多”端的`relationship.uselist`参数设置为`False`来启用一对一模式，如下面的非注解式声明配置所示：

```py
class Parent(Base):
    __tablename__ = "parent_table"

    id = mapped_column(Integer, primary_key=True)
    child = relationship("Child", uselist=False, back_populates="parent")

class Child(Base):
    __tablename__ = "child_table"

    id = mapped_column(Integer, primary_key=True)
    parent_id = mapped_column(ForeignKey("parent_table.id"))
    parent = relationship("Parent", back_populates="child")
```

### 对于非注解配置设置 uselist=False

当在没有使用`Mapped`注解的情况下使用`relationship()`时，可以通过将通常是“多”端的`relationship.uselist`参数设置为`False`来启用一对一模式，如下面的非注解式声明配置所示：

```py
class Parent(Base):
    __tablename__ = "parent_table"

    id = mapped_column(Integer, primary_key=True)
    child = relationship("Child", uselist=False, back_populates="parent")

class Child(Base):
    __tablename__ = "child_table"

    id = mapped_column(Integer, primary_key=True)
    parent_id = mapped_column(ForeignKey("parent_table.id"))
    parent = relationship("Parent", back_populates="child")
```

## 多对多

多对多在两个类之间添加了一个关联表。这个关联表几乎总是以一个核心`Table`对象或其他核心可选项（如`Join`对象）的形式给出，并通过`relationship()`函数的`relationship.secondary`参数指示。通常，`Table`使用与声明基类关联的`MetaData`对象，以便`ForeignKey`指令可以定位要链接的远程表：

```py
from __future__ import annotations

from sqlalchemy import Column
from sqlalchemy import Table
from sqlalchemy import ForeignKey
from sqlalchemy import Integer
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

# note for a Core table, we use the sqlalchemy.Column construct,
# not sqlalchemy.orm.mapped_column
association_table = Table(
    "association_table",
    Base.metadata,
    Column("left_id", ForeignKey("left_table.id")),
    Column("right_id", ForeignKey("right_table.id")),
)

class Parent(Base):
    __tablename__ = "left_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    children: Mapped[List[Child]] = relationship(secondary=association_table)

class Child(Base):
    __tablename__ = "right_table"

    id: Mapped[int] = mapped_column(primary_key=True)
```

提示

上面的“关联表”已建立了引用关系的外键约束，这些约束指向关系两侧的两个实体表。`association.left_id`和`association.right_id`的数据类型通常是从引用表的数据类型推断出来的，可以省略。虽然 SQLAlchemy 没有要求，但**建议**将指向两个实体表的列建立在**唯一约束**或更常见的**主键约束**中；这样可以确保无论应用程序端是否存在问题，表中都不会持续存在重复行：

```py
association_table = Table(
    "association_table",
    Base.metadata,
    Column("left_id", ForeignKey("left_table.id"), primary_key=True),
    Column("right_id", ForeignKey("right_table.id"), primary_key=True),
)
```

### 设置双向多对多

对于双向关系，关系的两侧都包含一个集合。使用`relationship.back_populates`进行指定，并且对于每个`relationship()`指定共同的关联表：

```py
from __future__ import annotations

from sqlalchemy import Column
from sqlalchemy import Table
from sqlalchemy import ForeignKey
from sqlalchemy import Integer
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

association_table = Table(
    "association_table",
    Base.metadata,
    Column("left_id", ForeignKey("left_table.id"), primary_key=True),
    Column("right_id", ForeignKey("right_table.id"), primary_key=True),
)

class Parent(Base):
    __tablename__ = "left_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    children: Mapped[List[Child]] = relationship(
        secondary=association_table, back_populates="parents"
    )

class Child(Base):
    __tablename__ = "right_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    parents: Mapped[List[Parent]] = relationship(
        secondary=association_table, back_populates="children"
    )
```

### 使用“secondary”参数的后期评估形式

`relationship()`的`relationship.secondary`参数还接受两种不同的“后期评估”形式，包括字符串表名称以及 lambda 可调用。有关背景和示例，请参见使用“secondary”参数的后期评估形式进行多对多关系部分。

### 使用集合、列表或其他集合类型进行多对多

配置多对多关系的集合与一对多的配置相同，如在使用集合、列表或其他集合类型进行一对多关系中所述。对于使用`Mapped`进行注释的映射，集合可以由`Mapped`泛型类内部使用的集合类型指示，例如`set`：

```py
class Parent(Base):
    __tablename__ = "left_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    children: Mapped[Set["Child"]] = relationship(secondary=association_table)
```

当使用命令式映射（即一对多情况）的非注释形式时，可以通过`relationship.collection_class`参数传递用作集合的 Python 类。

另请参阅

自定义集合访问 - 包含有关集合配置的进一步详细信息，包括一些将`relationship()`映射到字典的技术。

### 从多对多表中删除行

对于`relationship()`的`relationship.secondary`参数是唯一的行为，这里指定的`Table`将自动受到 INSERT 和 DELETE 语句的影响，当对象被添加或从集合中删除时。**没有必要手动从此表中删除**。从集合中删除记录的行为将导致刷新时删除该行的效果：

```py
# row will be deleted from the "secondary" table
# automatically
myparent.children.remove(somechild)
```

当子对象直接传递给`Session.delete()`时，“次要”表中的行如何删除经常会引起一个问题：

```py
session.delete(somechild)
```

这里有几种可能性：

+   如果从`Parent`到`Child`有一个`relationship()`，但是没有将特定的`Child`链接到每个`Parent`的反向关系，SQLAlchemy 不会意识到删除此特定`Child`对象时需要维护链接到`Parent`的“次要”表。不会删除“次要”表的删除。

+   如果存在将特定的`Child`链接到每个`Parent`的关系，假设它被称为`Child.parents`，SQLAlchemy 默认会加载`Child.parents`集合以定位所有`Parent`对象，并从建立此链接的“次要”表中删除每行。请注意，此关系不需要是双向的；SQLAlchemy 严格地查看与被删除的`Child`对象相关联的每个`relationship()`。

+   在这里的一个性能较高的选项是使用数据库中使用的外键的 ON DELETE CASCADE 指令。假设数据库支持这个特性，数据库本身可以被设置为在“子”中的引用行被删除时自动删除“次要”表中的行。在这种情况下，SQLAlchemy 可以被指示放弃主动加载`Child.parents`集合，使用`relationship()`上的`relationship.passive_deletes`指令；参见使用 ORM 关系的外键 ON DELETE 级联以获取更多关于此的详细信息。

再次注意，这些行为*仅*与与`relationship()`一起使用的`relationship.secondary`选项相关。如果处理的是显式映射的关联表，并且这些表*不*出现在相关`relationship()`的`relationship.secondary`选项中，则可以改用级联规则来自动删除实体，以响应相关实体的删除 - 有关此功能的信息，请参阅级联。

另请参阅

在多对多关系中使用级联删除

在多对多关系中使用外键 ON DELETE

### 设置双向多对多

对于双向关系，关系的两端都包含一个集合。使用`relationship.back_populates`来指定，并且对于每个`relationship()`都要指定共同的关联表：

```py
from __future__ import annotations

from sqlalchemy import Column
from sqlalchemy import Table
from sqlalchemy import ForeignKey
from sqlalchemy import Integer
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

association_table = Table(
    "association_table",
    Base.metadata,
    Column("left_id", ForeignKey("left_table.id"), primary_key=True),
    Column("right_id", ForeignKey("right_table.id"), primary_key=True),
)

class Parent(Base):
    __tablename__ = "left_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    children: Mapped[List[Child]] = relationship(
        secondary=association_table, back_populates="parents"
    )

class Child(Base):
    __tablename__ = "right_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    parents: Mapped[List[Parent]] = relationship(
        secondary=association_table, back_populates="children"
    )
```

### 使用延迟评估形式的“secondary”参数

`relationship()`的`relationship.secondary`参数还接受两种不同的“延迟评估”形式，包括字符串表名以及 lambda 可调用。有关背景和示例，请参阅使用“secondary”参数的延迟评估形式进行多对多关系部分。

### 使用集合、列表或其他集合类型进行多对多关系

对于多对多关系的集合配置与一对多完全相同，如使用集合、列表或其他集合类型进行一对多关系中所述。对于使用`Mapped`进行注释的映射，可以通过`Mapped`泛型类中使用的集合类型来指示集合，例如`set`：

```py
class Parent(Base):
    __tablename__ = "left_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    children: Mapped[Set["Child"]] = relationship(secondary=association_table)
```

当使用非注释形式，包括命令式映射时，就像一对多一样，可以使用`relationship.collection_class`参数传递要用作集合的 Python 类。

另请参阅

自定义集合访问 - 包含有关集合配置的进一步详细信息，包括一些将`relationship()`映射到字典的技术。

### 从多对多表中删除行

`relationship()`参数中唯一的行为是，指定的`Table`在对象被添加或从集合中删除时会自动受到 INSERT 和 DELETE 语句的影响。**无需手动从此表中删除**。从集合中删除记录的行为将导致在 flush 时删除该行：

```py
# row will be deleted from the "secondary" table
# automatically
myparent.children.remove(somechild)
```

经常出现的一个问题是当直接将子对象传递给`Session.delete()`时如何删除“secondary”表中的行：

```py
session.delete(somechild)
```

这里有几种可能性：

+   如果从`Parent`到`Child`有一个`relationship()`，但没有一个反向关系将特定的`Child`与每个`Parent`关联起来，SQLAlchemy 将不会意识到当删除这个特定的`Child`对象时，它需要维护将其与`Parent`链接起来的“secondary”表。不会删除“secondary”表。

+   如果有一个将特定的`Child`与每个`Parent`关联起来的关系，假设它被称为`Child.parents`，SQLAlchemy 默认会加载`Child.parents`集合以定位所有`Parent`对象，并从建立此链接的“secondary”表中删除每一行。请注意，此关系不需要是双向的；SQLAlchemy 严格查看与正在删除的`Child`对象相关联的每一个`relationship()`。

+   这里的一个性能更高的选项是与数据库一起使用 ON DELETE CASCADE 指令。假设数据库支持这个功能，数据库本身可以被设置为在“子”中的引用行被删除时自动删除“辅助”表中的行。在这种情况下，SQLAlchemy 可以被指示不要主动加载 `Child.parents` 集合，使用 `relationship.passive_deletes` 指令在 `relationship()` 上；有关此更多详细信息，请参阅 使用外键 ON DELETE cascade 处理 ORM 关系。

再次注意，这些行为*仅*与 `relationship()` 的 `relationship.secondary` 选项相关。如果处理显式映射的关联表，而不是存在于相关 `relationship()` 的 `relationship.secondary` 选项中的关联表，那么级联规则可以被用来在相关实体被删除时自动删除实体 - 有关此功能的信息，请参阅 级联。

另请参阅

使用多对多关系的级联删除

使用外键 ON DELETE 处理多对多关系

## 协会对象

协会对象模式是多对多关系的一种变体：当一个关联表包含除了那些与父表和子表（或左表和右表）的外键不同的额外列时，通常最理想的是将这些列映射到自己的 ORM 映射类。这个映射类被映射到了`Table`，在使用多对多模式时，它本来会被指定为 `relationship.secondary`。

在关联对象模式中，不使用`relationship.secondary`参数；相反，将类直接映射到关联表。然后，两个独立的`relationship()`构造首先通过一对多将父侧链接到映射的关联类，然后通过多对一将映射的关联类链接到子侧，以形成从父对象到关联对象到子对象的单向关联对象关系。对于双向关系，使用四个`relationship()`构造将映射的关联类与父对象和子对象在两个方向上进行链接。

下面的示例说明了一个新的类`Association`，它映射到名为`association`的`Table`；此表现在包括一个额外的列称为`extra_data`，它是一个字符串值，与`Parent`和`Child`之间的每个关联一起存储。通过将表映射到显式类，从`Parent`到`Child`的基本访问明确使用了`Association`：

```py
from typing import Optional

from sqlalchemy import ForeignKey
from sqlalchemy import Integer
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

class Association(Base):
    __tablename__ = "association_table"
    left_id: Mapped[int] = mapped_column(ForeignKey("left_table.id"), primary_key=True)
    right_id: Mapped[int] = mapped_column(
        ForeignKey("right_table.id"), primary_key=True
    )
    extra_data: Mapped[Optional[str]]
    child: Mapped["Child"] = relationship()

class Parent(Base):
    __tablename__ = "left_table"
    id: Mapped[int] = mapped_column(primary_key=True)
    children: Mapped[List["Association"]] = relationship()

class Child(Base):
    __tablename__ = "right_table"
    id: Mapped[int] = mapped_column(primary_key=True)
```

为了说明双向版本，我们添加了两个更多的`relationship()`构造，使用`relationship.back_populates`连接到现有的构造：

```py
from typing import Optional

from sqlalchemy import ForeignKey
from sqlalchemy import Integer
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

class Association(Base):
    __tablename__ = "association_table"
    left_id: Mapped[int] = mapped_column(ForeignKey("left_table.id"), primary_key=True)
    right_id: Mapped[int] = mapped_column(
        ForeignKey("right_table.id"), primary_key=True
    )
    extra_data: Mapped[Optional[str]]
    child: Mapped["Child"] = relationship(back_populates="parents")
    parent: Mapped["Parent"] = relationship(back_populates="children")

class Parent(Base):
    __tablename__ = "left_table"
    id: Mapped[int] = mapped_column(primary_key=True)
    children: Mapped[List["Association"]] = relationship(back_populates="parent")

class Child(Base):
    __tablename__ = "right_table"
    id: Mapped[int] = mapped_column(primary_key=True)
    parents: Mapped[List["Association"]] = relationship(back_populates="child")
```

使用关联对象模式的直接形式需要在将子对象附加到父对象之前将其与关联实例关联；同样，从父对象到子对象的访问需要通过关联对象进行：

```py
# create parent, append a child via association
p = Parent()
a = Association(extra_data="some data")
a.child = Child()
p.children.append(a)

# iterate through child objects via association, including association
# attributes
for assoc in p.children:
    print(assoc.extra_data)
    print(assoc.child)
```

为了增强关联对象模式，使得对`Association`对象的直接访问是可选的，SQLAlchemy 提供了关联代理扩展。该扩展允许配置属性，这些属性将通过单个访问实现两次“跳跃”，一次是到关联对象，另一次是到目标属性。

另请参阅

关联代理 - 允许在三类关联对象映射中在父对象和子对象之间直接进行“多对多”样式的访问。

警告

避免直接混合使用关联对象模式和多对多模式，因为这会导致数据可能以不一致的方式读取和写入，除非采取特殊步骤；关联代理通常用于提供更简洁的访问。有关此组合引入的注意事项的更详细背景，请参阅下一节将关联对象与多对多访问模式组合使用。

### 将关联对象与多对多访问模式结合使用

如前一节所述，关联对象模式不会自动与相同表/列的多对多模式集成。由此可知，读操作可能返回冲突数据，写操作也可能尝试刷新冲突更改，导致完整性错误或意外插入或删除。

为了说明，下面的示例配置了`Parent`和`Child`之间的双向多对多关系，通过`Parent.children`和`Child.parents`。同时，还配置了一个关联对象关系，即`Parent.child_associations -> Association.child`和`Child.parent_associations -> Association.parent`之间的关系：

```py
from typing import Optional

from sqlalchemy import ForeignKey
from sqlalchemy import Integer
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

class Association(Base):
    __tablename__ = "association_table"

    left_id: Mapped[int] = mapped_column(ForeignKey("left_table.id"), primary_key=True)
    right_id: Mapped[int] = mapped_column(
        ForeignKey("right_table.id"), primary_key=True
    )
    extra_data: Mapped[Optional[str]]

    # association between Assocation -> Child
    child: Mapped["Child"] = relationship(back_populates="parent_associations")

    # association between Assocation -> Parent
    parent: Mapped["Parent"] = relationship(back_populates="child_associations")

class Parent(Base):
    __tablename__ = "left_table"

    id: Mapped[int] = mapped_column(primary_key=True)

    # many-to-many relationship to Child, bypassing the `Association` class
    children: Mapped[List["Child"]] = relationship(
        secondary="association_table", back_populates="parents"
    )

    # association between Parent -> Association -> Child
    child_associations: Mapped[List["Association"]] = relationship(
        back_populates="parent"
    )

class Child(Base):
    __tablename__ = "right_table"

    id: Mapped[int] = mapped_column(primary_key=True)

    # many-to-many relationship to Parent, bypassing the `Association` class
    parents: Mapped[List["Parent"]] = relationship(
        secondary="association_table", back_populates="children"
    )

    # association between Child -> Association -> Parent
    parent_associations: Mapped[List["Association"]] = relationship(
        back_populates="child"
    )
```

使用此 ORM 模型进行更改时，对`Parent.children`进行的更改不会与在 Python 中对`Parent.child_associations`或`Child.parent_associations`进行的更改协调；虽然所有这些关系将继续正常运作，但一个上的更改不会显示在另一个上，直到`Session`过期，通常在`Session.commit()`之后会自动发生。

此外，如果发生冲突更改，例如同时添加新的`Association`对象并将相同相关的`Child`附加到`Parent.children`，则在工作单元刷新过程中会引发完整性错误，如下例所示：

```py
p1 = Parent()
c1 = Child()
p1.children.append(c1)

# redundant, will cause a duplicate INSERT on Association
p1.child_associations.append(Association(child=c1))
```

直接将`Parent.children`附加`Child`也意味着在`association`表中创建行，而不指定`association.extra_data`列的任何值，该列将接收`NULL`作为其值。

如果知道自己在做什么，使用上述映射是可以的；在很少使用“关联对象”模式的情况下使用多对多关系可能有充分的理由，因为在单个多对多关系中加载关系更容易，这也可以稍微优化“secondary”表在 SQL 语句中的使用方式，与两个分开的关系到显式关联类的使用方式相比。至少最好将`relationship.viewonly`参数应用于“secondary”关系，以避免发生冲突更改的问题，并防止将`NULL`写入附加的关联列，如下所示：

```py
class Parent(Base):
    __tablename__ = "left_table"

    id: Mapped[int] = mapped_column(primary_key=True)

    # many-to-many relationship to Child, bypassing the `Association` class
    children: Mapped[List["Child"]] = relationship(
        secondary="association_table", back_populates="parents", viewonly=True
    )

    # association between Parent -> Association -> Child
    child_associations: Mapped[List["Association"]] = relationship(
        back_populates="parent"
    )

class Child(Base):
    __tablename__ = "right_table"

    id: Mapped[int] = mapped_column(primary_key=True)

    # many-to-many relationship to Parent, bypassing the `Association` class
    parents: Mapped[List["Parent"]] = relationship(
        secondary="association_table", back_populates="children", viewonly=True
    )

    # association between Child -> Association -> Parent
    parent_associations: Mapped[List["Association"]] = relationship(
        back_populates="child"
    )
```

上述映射不会将任何更改写入到数据库的`Parent.children`或`Child.parents`，从而防止冲突的写入。然而，如果在相同事务或`Session`中对这些集合进行更改，那么对`Parent.children`或`Child.parents`的读取将不一定匹配从`Parent.child_associations`或`Child.parent_associations`读取的数据。如果关联对象关系的使用不频繁，并且针对访问多对多集合的代码进行了精心组织以避免过时的读取（在极端情况下，直接使用`Session.expire()`来使集合在当前事务中刷新），那么这种模式可能是可行的。

上述模式的一种流行替代方案是，直接的多对多`Parent.children`和`Child.parents`关系被一个扩展所取代，该扩展将通过`Association`类透明地代理，同时从 ORM 的角度保持一切一致。这个扩展被称为关联代理。

另请参阅

关联代理 - 允许在三类关联对象映射之间直接进行“多对多”样式的父子访问。### 将关联对象与多对多访问模式结合使用

如前所述，在上一节中，关联对象模式不会自动与同时针对相同表/列使用的多对多模式集成。由此可见，读取操作可能会返回冲突的数据，并且写入操作也可能尝试刷新冲突的更改，导致完整性错误或意外的插入或删除。

为了说明，下面的示例配置了`Parent`和`Child`之间的双向多对多关系，通过`Parent.children`和`Child.parents`。同时，还配置了一个关联对象关系，`Parent.child_associations -> Association.child`和`Child.parent_associations -> Association.parent`：

```py
from typing import Optional

from sqlalchemy import ForeignKey
from sqlalchemy import Integer
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

class Association(Base):
    __tablename__ = "association_table"

    left_id: Mapped[int] = mapped_column(ForeignKey("left_table.id"), primary_key=True)
    right_id: Mapped[int] = mapped_column(
        ForeignKey("right_table.id"), primary_key=True
    )
    extra_data: Mapped[Optional[str]]

    # association between Assocation -> Child
    child: Mapped["Child"] = relationship(back_populates="parent_associations")

    # association between Assocation -> Parent
    parent: Mapped["Parent"] = relationship(back_populates="child_associations")

class Parent(Base):
    __tablename__ = "left_table"

    id: Mapped[int] = mapped_column(primary_key=True)

    # many-to-many relationship to Child, bypassing the `Association` class
    children: Mapped[List["Child"]] = relationship(
        secondary="association_table", back_populates="parents"
    )

    # association between Parent -> Association -> Child
    child_associations: Mapped[List["Association"]] = relationship(
        back_populates="parent"
    )

class Child(Base):
    __tablename__ = "right_table"

    id: Mapped[int] = mapped_column(primary_key=True)

    # many-to-many relationship to Parent, bypassing the `Association` class
    parents: Mapped[List["Parent"]] = relationship(
        secondary="association_table", back_populates="children"
    )

    # association between Child -> Association -> Parent
    parent_associations: Mapped[List["Association"]] = relationship(
        back_populates="child"
    )
```

当使用此 ORM 模型进行更改时，在 Python 中对`Parent.children`进行的更改不会与对`Parent.child_associations`或`Child.parent_associations`进行的更改协调；虽然所有这些关系都将继续正常运行，但在`Session`过期之前，一个的更改不会显示在另一个上，`Session.commit()`通常会在自动发生后使之过期。

另外，如果发生冲突的更改，例如同时添加一个新的`Association`对象，同时将相同的相关`Child`附加到`Parent.children`，则在工作单元刷新过程进行时，会引发完整性错误，如下例所示：

```py
p1 = Parent()
c1 = Child()
p1.children.append(c1)

# redundant, will cause a duplicate INSERT on Association
p1.child_associations.append(Association(child=c1))
```

直接将`Child`附加到`Parent.children`也意味着在`association`表中创建行，而不指定`association.extra_data`列的任何值，该列的值将为`NULL`。

如果你知道自己在做什么，像上面的映射那样使用映射是可以的；在很少使用“关联对象”模式的情况下使用多对多关系可能是有充分理由的，这是因为沿着单一的多对多关系加载关系是更容易的，这也可以略微优化“辅助”表在 SQL 语句中的使用方式，与如何使用两个到显式关联类的分离关系相比。至少应该将`relationship.viewonly`参数应用于“辅助”关系，以避免出现冲突更改的问题，并防止将`NULL`写入附加的关联列，如下所示：

```py
class Parent(Base):
    __tablename__ = "left_table"

    id: Mapped[int] = mapped_column(primary_key=True)

    # many-to-many relationship to Child, bypassing the `Association` class
    children: Mapped[List["Child"]] = relationship(
        secondary="association_table", back_populates="parents", viewonly=True
    )

    # association between Parent -> Association -> Child
    child_associations: Mapped[List["Association"]] = relationship(
        back_populates="parent"
    )

class Child(Base):
    __tablename__ = "right_table"

    id: Mapped[int] = mapped_column(primary_key=True)

    # many-to-many relationship to Parent, bypassing the `Association` class
    parents: Mapped[List["Parent"]] = relationship(
        secondary="association_table", back_populates="children", viewonly=True
    )

    # association between Child -> Association -> Parent
    parent_associations: Mapped[List["Association"]] = relationship(
        back_populates="child"
    )
```

上面的映射不会将对`Parent.children`或`Child.parents`的任何更改写入数据库，从而防止冲突写入。但是，如果在相同的事务或`Session`中对这些集合进行更改的地方读取`Parent.children`或`Child.parents`将不一定与从`Parent.child_associations`或`Child.parent_associations`中读取的数据匹配。如果对关联对象关系的使用不频繁，并且针对访问多对多集合的代码进行了精心组织以避免过时读取（在极端情况下，直接使用`Session.expire()`来导致集合在当前事务中被刷新），那么这种模式可能是可行的。

一个流行的替代模式是，直接的多对多`Parent.children`和`Child.parents`关系被一个扩展所取代，该扩展将通过`Association`类透明地代理，同时从 ORM 的角度保持一切一致。这个扩展被称为关联代理。

另请参阅

关联代理 - 允许在三类关联对象映射中直接实现“多对多”样式的父子访问。

## 关系参数的延迟评估

大多数前面部分的示例展示了映射，其中各种`relationship()`构造使用字符串名称而不是类本身引用其目标类，例如在使用`Mapped`时，会生成一个仅作为字符串存在的前向引用：

```py
class Parent(Base):
    # ...

    children: Mapped[List["Child"]] = relationship(back_populates="parent")

class Child(Base):
    # ...

    parent: Mapped["Parent"] = relationship(back_populates="children")
```

同样，在使用非注释形式，如非注释性的声明式或命令式映射时，`relationship()`构造也直接支持字符串名称：

```py
registry.map_imperatively(
    Parent,
    parent_table,
    properties={"children": relationship("Child", back_populates="parent")},
)

registry.map_imperatively(
    Child,
    child_table,
    properties={"parent": relationship("Parent", back_populates="children")},
)
```

这些字符串名称在映射器解析阶段被解析为类，这是一个内部过程，通常在定义所有映射之后发生，并且通常由映射本身的第一次使用触发。`registry`对象是这些名称存储和解析为它们引用的映射类的容器。

除了`relationship()`的主要类参数之外，还可以指定依赖于尚未定义类上存在的列的其他参数，这些参数可以是 Python 函数，或更常见的是字符串。对于这些参数中的大多数，除了主要参数之外，字符串输入都会**使用 Python 内置的 eval()函数评估为 Python 表达式**，因为它们旨在接收完整的 SQL 表达式。

警告

由于 Python 的`eval()`函数用于解释传递给`relationship()`映射配置构造的延迟评估的字符串参数，这些参数不应该被重新用于接收不受信任的用户输入；`eval()`对不受信任的用户输入**不安全**。

在这个评估中可用的完整命名空间包括为这个声明基类映射的所有类，以及`sqlalchemy`包的内容，包括表达式函数如`desc()`和`sqlalchemy.sql.functions.func`：

```py
class Parent(Base):
    # ...

    children: Mapped[List["Child"]] = relationship(
        order_by="desc(Child.email_address)",
        primaryjoin="Parent.id == Child.parent_id",
    )
```

对于一个模块包含多个同名类的情况，字符串类名也可以在这些字符串表达式中作为模块限定路径指定：

```py
class Parent(Base):
    # ...

    children: Mapped[List["myapp.mymodel.Child"]] = relationship(
        order_by="desc(myapp.mymodel.Child.email_address)",
        primaryjoin="myapp.mymodel.Parent.id == myapp.mymodel.Child.parent_id",
    )
```

在上述示例中，传递给`Mapped`的字符串也可以通过直接将类位置字符串传递给`relationship.argument`来消除特定类参数。下面说明了仅类型导入`Child`的示例，结合了将运行时说明符与将在`registry`中搜索正确名称的目标类相结合：

```py
import typing

if typing.TYPE_CHECKING:
    from myapp.mymodel import Child

class Parent(Base):
    # ...

    children: Mapped[List["Child"]] = relationship(
        "myapp.mymodel.Child",
        order_by="desc(myapp.mymodel.Child.email_address)",
        primaryjoin="myapp.mymodel.Parent.id == myapp.mymodel.Child.parent_id",
    )
```

合格路径可以是任何消除名称之间歧义的部分路径。例如，要消除`myapp.model1.Child`和`myapp.model2.Child`之间的歧义，我们可以指定`model1.Child`或`model2.Child`：

```py
class Parent(Base):
    # ...

    children: Mapped[List["Child"]] = relationship(
        "model1.Child",
        order_by="desc(mymodel1.Child.email_address)",
        primaryjoin="Parent.id == model1.Child.parent_id",
    )
```

`relationship()`构造还接受 Python 函数或 lambda 作为这些参数的输入。Python 函数式方法可能如下所示：

```py
import typing

from sqlalchemy import desc

if typing.TYPE_CHECKING:
    from myapplication import Child

def _resolve_child_model():
    from myapplication import Child

    return Child

class Parent(Base):
    # ...

    children: Mapped[List["Child"]] = relationship(
        _resolve_child_model,
        order_by=lambda: desc(_resolve_child_model().email_address),
        primaryjoin=lambda: Parent.id == _resolve_child_model().parent_id,
    )
```

完整的参数列表接受 Python 函数/lambda 或将传递给`eval()`的字符串的参数包括：

+   `relationship.order_by`

+   `relationship.primaryjoin`

+   `relationship.secondaryjoin`

+   `relationship.secondary`

+   `relationship.remote_side`

+   `relationship.foreign_keys`

+   `relationship._user_defined_foreign_keys`

警告

如前所述，`relationship()`中的上述参数会**作为 Python 代码表达式使用 eval()进行评估。不要将不受信任的输入传递给这些参数。**

### 在声明后将关系添加到映射类

还应注意，与向现有的声明映射类添加附加列中描述的类似方式，任何`MapperProperty`构造都可以随时添加到声明基础映射中（注意在此上下文中不支持注释形式）。如果我们想要在`Address`类可用之后实现这个`relationship()`，我们也可以随后应用它：

```py
# first, module A, where Child has not been created yet,
# we create a Parent class which knows nothing about Child

class Parent(Base): ...

# ... later, in Module B, which is imported after module A:

class Child(Base): ...

from module_a import Parent

# assign the User.addresses relationship as a class variable.  The
# declarative base class will intercept this and map the relationship.
Parent.children = relationship(Child, primaryjoin=Child.parent_id == Parent.id)
```

与 ORM 映射列一样，`Mapped`注解类型无法参与此操作；因此，相关类必须直接在`relationship()`构造中指定，可以是类本身、类的字符串名称或返回目标类引用的可调用函数。

注意

与 ORM 映射列一样，对已映射类的映射属性的赋值仅在使用“声明基类”类时才能正确执行，这意味着用户定义的`DeclarativeBase`子类或`declarative_base()`返回的动态生成类或`registry.generate_base()`。这个“基”类包括一个 Python 元类，实现了一个特殊的`__setattr__()`方法来拦截这些操作。

如果类使用像`registry.mapped()`这样的装饰器或像`registry.map_imperatively()`这样的命令式函数进行映射，则无法在运行时将类映射属性分配给映射类。  ### 使用多对多关系的“secondary”参数的延迟评估形式

多对多关系使用`relationship.secondary`参数，通常表示对通常不映射的`Table`对象或其他 Core 可选择对象的引用。通常使用 lambda 可调用进行延迟评估。

对于 Many To Many 中给出的示例，如果我们假设`association_table` `Table`对象将在模块中稍后定义，则我们可以使用 lambda 编写`relationship()`：

```py
class Parent(Base):
    __tablename__ = "left_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    children: Mapped[List["Child"]] = relationship(
        "Child", secondary=lambda: association_table
    )
```

作为也是**有效 Python 标识符**的表名的快捷方式，`relationship.secondary`参数也可以作为字符串传递，其中解析通过将字符串作为 Python 表达式进行评估来完成，简单标识符名称链接到当前`registry`引用的相同命名的`Table`对象。

在下面的示例中，表达式`"association_table"`将作为名为`"association_table"`的变量进行评估，该变量将根据`MetaData`集合中的表名进行解析：

```py
class Parent(Base):
    __tablename__ = "left_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    children: Mapped[List["Child"]] = relationship(secondary="association_table")
```

注意

当作为字符串传递时，传递给`relationship.secondary`的名称**必须是有效的 Python 标识符**，以字母开头，只包含字母数字字符或下划线。其他字符，如破折号等，将被解释为 Python 运算符，而不会解析为给定的名称。请考虑使用 lambda 表达式而不是字符串以提高清晰度。

警告

当作为字符串传递时，`relationship.secondary`参数使用 Python 的`eval()`函数进行解释，即使它通常是一个表的名称。**不要将不受信任的输入传递给此字符串**。###在声明后向映射类添加关系

还应注意，在类似于 Appending additional columns to an existing Declarative mapped class 描述的方式中，任何`MapperProperty`构造都可以随时添加到声明基本映射中（注意，此上下文中不支持注释形式）。如果我们希望在`Address`类可用后实现此`relationship()`，我们也可以随后应用它：

```py
# first, module A, where Child has not been created yet,
# we create a Parent class which knows nothing about Child

class Parent(Base): ...

# ... later, in Module B, which is imported after module A:

class Child(Base): ...

from module_a import Parent

# assign the User.addresses relationship as a class variable.  The
# declarative base class will intercept this and map the relationship.
Parent.children = relationship(Child, primaryjoin=Child.parent_id == Parent.id)
```

与 ORM 映射列一样，`Mapped`注解类型没有参与此操作的能力；因此，相关类必须直接在`relationship()`构造中指定，可以是类本身、类的字符串名称，或者返回目标类引用的可调用函数。

注意

与 ORM 映射列一样，将映射属性分配给已经映射的类只有在使用“声明式基类”时才能正确运行，这意味着必须使用用户定义的`DeclarativeBase`子类或者`declarative_base()`返回的动态生成的类或者`registry.generate_base()`返回的动态生成的类。这个“基类”包含一个实现了特殊`__setattr__()`方法的 Python 元类，它拦截这些操作。

如果使用类似于`registry.mapped()`这样的装饰器或像`registry.map_imperatively()`这样的命令式函数来映射类，则无法在运行时将映射属性分配给映射类。

### 使用“secondary”参数的延迟评估形式来处理多对多关系

多对多关系使用`relationship.secondary`参数，通常表示对通常非映射的`Table`对象或其他核心可选择对象的引用。典型的延迟评估使用 lambda 可调用。

对于 Many To Many 中给出的例子，如果我们假设`association_table` `Table`对象将在模块中的某个后续点被定义，那么我们可以使用 lambda 来编写`relationship()`，如下所示：

```py
class Parent(Base):
    __tablename__ = "left_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    children: Mapped[List["Child"]] = relationship(
        "Child", secondary=lambda: association_table
    )
```

作为表名的快捷方式，也可以将`relationship.secondary`参数传递为字符串，其中解析工作通过将字符串作为 Python 表达式进行评估，简单标识符名称链接到与当前`registry`引用的相同`MetaData`集合中存在的同名`Table`对象。

在下面的示例中，表达式`"association_table"`被解析为一个名为`"association_table"`的变量，该变量根据`MetaData`集合中的表名解析：

```py
class Parent(Base):
    __tablename__ = "left_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    children: Mapped[List["Child"]] = relationship(secondary="association_table")
```

注意

当作为字符串传递时，传递给`relationship.secondary`的名称**必须是有效的 Python 标识符**，以字母开头，仅包含字母数字字符或下划线。其他字符，如破折号等，将被解释为 Python 操作符，而不会解析为给定的名称。请考虑使用 lambda 表达式而不是字符串，以提高清晰度。

警告

当作为字符串传递时，`relationship.secondary`参数将使用 Python 的`eval()`函数进行解释，即使它通常是一个表的名称。**不要将不受信任的输入传递给该字符串**。
