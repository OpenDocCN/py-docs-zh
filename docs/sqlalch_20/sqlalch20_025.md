# 复合列类型

> 原文：[`docs.sqlalchemy.org/en/20/orm/composites.html`](https://docs.sqlalchemy.org/en/20/orm/composites.html)

列集合可以关联到一个单一用户定义的数据类型，现代使用中通常是一个 Python [dataclass](https://docs.python.org/3/library/dataclasses.html)。ORM 提供了一个属性，使用您提供的类来表示列的组。

一个简单的例子表示一对 `Integer` 列作为一个 `Point` 对象，带有属性 `.x` 和 `.y`。使用 dataclass，这些属性使用相应的 `int` Python 类型定义：

```py
import dataclasses

@dataclasses.dataclass
class Point:
    x: int
    y: int
```

也接受非 dataclass 形式，但需要实现额外的方法。有关使用非 dataclass 类的示例，请参见 Using Legacy Non-Dataclasses 部分。

2.0 版中新增：`composite()` 构造完全支持 Python dataclasses，包括从复合类派生映射列数据类型的能力。

我们将创建一个映射到表 `vertices` 的映射，表示两个点为 `x1/y1` 和 `x2/y2`。使用 `composite()` 构造将 `Point` 类与映射列关联起来。

下面的示例说明了与完全 Annotated Declarative Table 配置一起使用的最现代形式的 `composite()`，`mapped_column()` 构造表示每个列直接传递给 `composite()`，指示要生成的列的零个或多个方面，在这种情况下是名称；`composite()` 构造直接从数据类中推导列类型（在本例中为 `int`，对应于 `Integer`）：

```py
from sqlalchemy.orm import DeclarativeBase, Mapped
from sqlalchemy.orm import composite, mapped_column

class Base(DeclarativeBase):
    pass

class Vertex(Base):
    __tablename__ = "vertices"

    id: Mapped[int] = mapped_column(primary_key=True)

    start: Mapped[Point] = composite(mapped_column("x1"), mapped_column("y1"))
    end: Mapped[Point] = composite(mapped_column("x2"), mapped_column("y2"))

    def __repr__(self):
        return f"Vertex(start={self.start}, end={self.end})"
```

提示

在上面的示例中，表示复合的列（`x1`、`y1` 等）也可以在类上访问，但类型检查器不能正确理解。如果访问单列很重要，可以明确声明它们，如 Map columns directly, pass attribute names to composite 中所示。

上述映射将对应于 CREATE TABLE 语句：

```py
>>> from sqlalchemy.schema import CreateTable
>>> print(CreateTable(Vertex.__table__))
CREATE  TABLE  vertices  (
  id  INTEGER  NOT  NULL,
  x1  INTEGER  NOT  NULL,
  y1  INTEGER  NOT  NULL,
  x2  INTEGER  NOT  NULL,
  y2  INTEGER  NOT  NULL,
  PRIMARY  KEY  (id)
) 
```

## 使用映射复合列类型

使用顶部部分示例中说明的映射，我们可以使用 `Vertex` 类，在其中 `.start` 和 `.end` 属性将透明地引用 `Point` 类引用的列，以及使用 `Vertex` 类的实例，在其中 `.start` 和 `.end` 属性将引用 `Point` 类的实例。 `x1`、`y1`、`x2` 和 `y2` 列被透明处理：

+   **持久化 Point 对象**

    我们可以创建一个 `Vertex` 对象，将 `Point` 对象分配为成员，并且它们将如预期一样持久化：

    ```py
    >>> v = Vertex(start=Point(3, 4), end=Point(5, 6))
    >>> session.add(v)
    >>> session.commit()
    BEGIN  (implicit)
    INSERT  INTO  vertices  (x1,  y1,  x2,  y2)  VALUES  (?,  ?,  ?,  ?)
    [generated  in  ...]  (3,  4,  5,  6)
    COMMIT 
    ```

+   **选择 Point 对象作为列**

    `composite()` 将允许 `Vertex.start` 和 `Vertex.end` 属性在使用 ORM `Session`（包括传统的 `Query` 对象）选择 `Point` 对象时尽可能地行为像单个 SQL 表达式：

    ```py
    >>> stmt = select(Vertex.start, Vertex.end)
    >>> session.execute(stmt).all()
    SELECT  vertices.x1,  vertices.y1,  vertices.x2,  vertices.y2
    FROM  vertices
    [...]  ()
    [(Point(x=3, y=4), Point(x=5, y=6))]
    ```

+   **在 SQL 表达式中比较 Point 对象**

    `Vertex.start` 和 `Vertex.end` 属性可以在 WHERE 条件和类似情况下使用，使用临时的 `Point` 对象进行比较：

    ```py
    >>> stmt = select(Vertex).where(Vertex.start == Point(3, 4)).where(Vertex.end < Point(7, 8))
    >>> session.scalars(stmt).all()
    SELECT  vertices.id,  vertices.x1,  vertices.y1,  vertices.x2,  vertices.y2
    FROM  vertices
    WHERE  vertices.x1  =  ?  AND  vertices.y1  =  ?  AND  vertices.x2  <  ?  AND  vertices.y2  <  ?
    [...]  (3,  4,  7,  8)
    [Vertex(Point(x=3, y=4), Point(x=5, y=6))]
    ```

    从 2.0 版开始：`composite()` 构造现在支持“排序”比较，例如 `<`、`>=` 等，除了已经存在的支持 `==`、`!=`。

    提示

    上面使用“小于”运算符 (`<`) 的“排序”比较以及使用 `==` 的“相等”比较，当用于生成 SQL 表达式时，是由 `Comparator` 类实现的，并不使用复合类本身的比较方法，例如 `__lt__()` 或 `__eq__()` 方法。 由此可见，上述 SQL 操作也不需要实现数据类 `order=True` 参数。重新定义复合操作部分包含如何自定义比较操作的背景信息。

+   **更新顶点实例上的 Point 对象**

    默认情况下，必须用新对象替换 `Point` 对象才能检测到更改：

    ```py
    >>> v1 = session.scalars(select(Vertex)).one()
    SELECT  vertices.id,  vertices.x1,  vertices.y1,  vertices.x2,  vertices.y2
    FROM  vertices
    [...]  ()
    >>> v1.end = Point(x=10, y=14)
    >>> session.commit()
    UPDATE  vertices  SET  x2=?,  y2=?  WHERE  vertices.id  =  ?
    [...]  (10,  14,  1)
    COMMIT 
    ```

    为了允许在复合对象上进行原地更改，必须使用 Mutation Tracking 扩展。请参阅在复合对象上建立可变性部分以获取示例。

## 复合对象的其他映射形式

`composite()` 构造可以使用 `mapped_column()` 构造、`Column` 或现有映射列的字符串名称来传递相关列。以下示例说明了与上述主要部分相同的等效映射。

### 直接映射列，然后传递给复合对象

在这里，我们将现有的 `mapped_column()` 实例传递给 `composite()` 构造函数，就像下面的非注释示例中一样，我们还将 `Point` 类作为第一个参数传递给 `composite()`：

```py
from sqlalchemy import Integer
from sqlalchemy.orm import mapped_column, composite

class Vertex(Base):
    __tablename__ = "vertices"

    id = mapped_column(Integer, primary_key=True)
    x1 = mapped_column(Integer)
    y1 = mapped_column(Integer)
    x2 = mapped_column(Integer)
    y2 = mapped_column(Integer)

    start = composite(Point, x1, y1)
    end = composite(Point, x2, y2)
```

### 直接映射列，将属性名称传递给组合类型

我们可以使用更多的注释形式编写上面相同的示例，其中我们有选项将属性名称传递给 `composite()`，而不是完整的列构造：

```py
from sqlalchemy.orm import mapped_column, composite, Mapped

class Vertex(Base):
    __tablename__ = "vertices"

    id: Mapped[int] = mapped_column(primary_key=True)
    x1: Mapped[int]
    y1: Mapped[int]
    x2: Mapped[int]
    y2: Mapped[int]

    start: Mapped[Point] = composite("x1", "y1")
    end: Mapped[Point] = composite("x2", "y2")
```

### 命令式映射和命令式表

在使用命令式表或完全命令式映射时，我们可以直接访问 `Column` 对象。这些也可以传递给 `composite()`，就像下面的命令式示例中一样：

```py
mapper_registry.map_imperatively(
    Vertex,
    vertices_table,
    properties={
        "start": composite(Point, vertices_table.c.x1, vertices_table.c.y1),
        "end": composite(Point, vertices_table.c.x2, vertices_table.c.y2),
    },
)
```  ## 使用遗留的非数据类

如果不使用数据类，则自定义数据类型类的要求是，它具有一个构造函数，该构造函数接受与其列格式相对应的位置参数，并且还提供一个方法 `__composite_values__()`，该方法返回对象的状态作为列表或元组，按照其基于列的属性顺序。它还应该提供足够的 `__eq__()` 和 `__ne__()` 方法，用于测试两个实例的相等性。

为了说明主要部分中的等效 `Point` 类不使用数据类：

```py
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y

    def __composite_values__(self):
        return self.x, self.y

    def __repr__(self):
        return f"Point(x={self.x!r}, y={self.y!r})"

    def __eq__(self, other):
        return isinstance(other, Point) and other.x == self.x and other.y == self.y

    def __ne__(self, other):
        return not self.__eq__(other)
```

使用 `composite()` 时，需要先声明要与 `Point` 类关联的列，并使用 其他复合类型的映射形式 中的一种形式进行显式类型声明。

## 跟踪组合类型的就地变更

对现有的组合类型值进行的就地更改不会自动跟踪。相反，组合类需要显式向其父对象提供事件。通过使用 `MutableComposite` 混合类，大部分工作已自动化，该类使用事件将每个用户定义的组合对象与所有父关联关联起来。请参阅在组合类型上建立可变性中的示例。

## 重新定义组合类型的比较操作

默认情况下，“equals”比较操作产生所有相应列相等的 AND。这可以通过`composite()`的`comparator_factory`参数进行更改，其中我们指定一个自定义的`Comparator`类来定义现有或新的操作。下面我们举例说明“大于”运算符，实现与基本“大于”相同的表达式：

```py
import dataclasses

from sqlalchemy.orm import composite
from sqlalchemy.orm import CompositeProperty
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.sql import and_

@dataclasses.dataclass
class Point:
    x: int
    y: int

class PointComparator(CompositeProperty.Comparator):
    def __gt__(self, other):
  """redefine the 'greater than' operation"""

        return and_(
            *[
                a > b
                for a, b in zip(
                    self.__clause_element__().clauses,
                    dataclasses.astuple(other),
                )
            ]
        )

class Base(DeclarativeBase):
    pass

class Vertex(Base):
    __tablename__ = "vertices"

    id: Mapped[int] = mapped_column(primary_key=True)

    start: Mapped[Point] = composite(
        mapped_column("x1"), mapped_column("y1"), comparator_factory=PointComparator
    )
    end: Mapped[Point] = composite(
        mapped_column("x2"), mapped_column("y2"), comparator_factory=PointComparator
    )
```

由于`Point`是一个数据类，我们可以使用`dataclasses.astuple()`来获得`Point`实例的元组形式。

然后，自定义比较器返回适当的 SQL 表达式：

```py
>>> print(Vertex.start > Point(5, 6))
vertices.x1  >  :x1_1  AND  vertices.y1  >  :y1_1 
```

## 嵌套复合体

复合对象可以被定义为在简单的嵌套方案中工作，通过在复合类内重新定义所需的行为，然后将复合类映射到通常的各列的全长。这需要定义额外的方法来在“嵌套”和“平面”形式之间移动。

下面我们重新组织`Vertex`类，使其本身成为一个复合对象，引用`Point`对象。`Vertex`和`Point`可以是数据类，但是我们将在`Vertex`中添加一个自定义的构造方法，该方法可以用于根据四个列值创建新的`Vertex`对象，我们将其任意命名为`_generate()`并定义为一个类方法，这样我们就可以通过向`Vertex._generate()`方法传递值来创建新的`Vertex`对象。

我们还将实现`__composite_values__()`方法，这是一个固定名称，被`composite()`构造函数所识别（在使用传统非数据类中介绍过），它指示了一种接收对象作为列值的标准方式，这种情况下将取代通常的数据类方法论。

有了我们自定义的`_generate()`构造函数和`__composite_values__()`序列化方法，我们现在可以在列的平面元组和包含`Point`实例的`Vertex`对象之间进行转换。`Vertex._generate`方法作为`composite()`构造函数的第一个参数传递，用于源`Vertex`实例的创建，并且`__composite_values__()`方法将隐式地被`composite()`使用。

为了例子的目的，`Vertex`复合体随后被映射到一个名为`HasVertex`的类中，该类是包含四个源列的`Table`最终所在的地方：

```py
from __future__ import annotations

import dataclasses
from typing import Any
from typing import Tuple

from sqlalchemy.orm import composite
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column

@dataclasses.dataclass
class Point:
    x: int
    y: int

@dataclasses.dataclass
class Vertex:
    start: Point
    end: Point

    @classmethod
    def _generate(cls, x1: int, y1: int, x2: int, y2: int) -> Vertex:
  """generate a Vertex from a row"""
        return Vertex(Point(x1, y1), Point(x2, y2))

    def __composite_values__(self) -> Tuple[Any, ...]:
  """generate a row from a Vertex"""
        return dataclasses.astuple(self.start) + dataclasses.astuple(self.end)

class Base(DeclarativeBase):
    pass

class HasVertex(Base):
    __tablename__ = "has_vertex"
    id: Mapped[int] = mapped_column(primary_key=True)
    x1: Mapped[int]
    y1: Mapped[int]
    x2: Mapped[int]
    y2: Mapped[int]

    vertex: Mapped[Vertex] = composite(Vertex._generate, "x1", "y1", "x2", "y2")
```

然后，上述映射可以根据`HasVertex`、`Vertex`和`Point`来使用：

```py
hv = HasVertex(vertex=Vertex(Point(1, 2), Point(3, 4)))

session.add(hv)
session.commit()

stmt = select(HasVertex).where(HasVertex.vertex == Vertex(Point(1, 2), Point(3, 4)))

hv = session.scalars(stmt).first()
print(hv.vertex.start)
print(hv.vertex.end)
```

## 复合 API

| 对象名称 | 描述 |
| --- | --- |
| composite([_class_or_attr], *attrs, [group, deferred, raiseload, comparator_factory, active_history, init, repr, default, default_factory, compare, kw_only, info, doc], **__kw) | 返回一个基于复合列的属性，供 Mapper 使用。 |

```py
function sqlalchemy.orm.composite(_class_or_attr: None | Type[_CC] | Callable[..., _CC] | _CompositeAttrType[Any] = None, *attrs: _CompositeAttrType[Any], group: str | None = None, deferred: bool = False, raiseload: bool = False, comparator_factory: Type[Composite.Comparator[_T]] | None = None, active_history: bool = False, init: _NoArg | bool = _NoArg.NO_ARG, repr: _NoArg | bool = _NoArg.NO_ARG, default: Any | None = _NoArg.NO_ARG, default_factory: _NoArg | Callable[[], _T] = _NoArg.NO_ARG, compare: _NoArg | bool = _NoArg.NO_ARG, kw_only: _NoArg | bool = _NoArg.NO_ARG, info: _InfoType | None = None, doc: str | None = None, **__kw: Any) → Composite[Any]
```

返回一个基于复合列的属性，供 Mapper 使用。

参见映射文档部分 复合列类型 以获取完整的使用示例。

由 `composite()` 返回的 `MapperProperty` 是 `Composite`。

参数:

+   `class_` – “复合类型”类，或任何类方法或可调用对象，根据顺序给出列值来产生复合对象的新实例。

+   `*attrs` –

    要映射的元素列表，可能包括:

    +   `Column` 对象

    +   `mapped_column()` 构造

    +   映射类上的其他属性的字符串名称，这些属性可以是任何其他 SQL 或对象映射属性。例如，这可以允许一个复合列引用一个一对多关系。

+   `active_history=False` – 当为 `True` 时，表示替换时标量属性的“上一个”值应加载，如果尚未加载。请参见 `column_property()` 上相同的标志。

+   `group` – 当标记为延迟加载时，此属性的分组名称。

+   `deferred` – 如果为 True，则列属性是“延迟加载”的，意味着它不会立即加载，而是在首次在实例上访问属性时加载。另请参见 `deferred()`。

+   `comparator_factory` – 一个扩展了 `Comparator` 的类，为比较操作提供自定义的 SQL 子句生成。

+   `doc` – 可选字符串，将作为类绑定描述符的文档应用。

+   `info` – 可选的数据字典，将填充到此对象的 `MapperProperty.info` 属性中。

+   `init` – 特定于 声明性数据类映射，指定映射属性是否应该是由数据类流程生成的 `__init__()` 方法的一部分。

+   `repr` – 特定于 声明性数据类映射，指定映射属性是否应该是由数据类流程生成的 `__repr__()` 方法的一部分。

+   `default_factory` – 特定于 声明性 Dataclass 映射，指定将作为 `__init__()` 方法的一部分执行的默认值生成函数。

+   `compare` –

    特定于 声明性 Dataclass 映射，指示在生成映射类的 `__eq__()` 和 `__ne__()` 方法时是否应包括此字段的比较操作。

    新版本 2.0.0b4 中。

+   `kw_only` – 特定于 声明性 Dataclass 映射，指示是否在生成 `__init__()` 时将此字段标记为关键字参数。

## 使用映射复合列类型

如上一节所示的映射，我们可以使用 `Vertex` 类，其中 `.start` 和 `.end` 属性将透明地引用 `Point` 类引用的列，以及使用 `Vertex` 类的实例，其中 `.start` 和 `.end` 属性将引用 `Point` 类的实例。`x1`、`y1`、`x2` 和 `y2` 列将被透明处理：

+   **持久化 Point 对象**

    我们可以创建一个 `Vertex` 对象，将 `Point` 对象分配为成员，并且它们将按预期持久化：

    ```py
    >>> v = Vertex(start=Point(3, 4), end=Point(5, 6))
    >>> session.add(v)
    >>> session.commit()
    BEGIN  (implicit)
    INSERT  INTO  vertices  (x1,  y1,  x2,  y2)  VALUES  (?,  ?,  ?,  ?)
    [generated  in  ...]  (3,  4,  5,  6)
    COMMIT 
    ```

+   **选择 Point 对象作为列**

    `composite()` 将允许 `Vertex.start` 和 `Vertex.end` 属性在使用 ORM `Session`（包括传统的 `Query` 对象）选择 `Point` 对象时尽可能地像单个 SQL 表达式一样行为：

    ```py
    >>> stmt = select(Vertex.start, Vertex.end)
    >>> session.execute(stmt).all()
    SELECT  vertices.x1,  vertices.y1,  vertices.x2,  vertices.y2
    FROM  vertices
    [...]  ()
    [(Point(x=3, y=4), Point(x=5, y=6))]
    ```

+   **比较 SQL 表达式中的 Point 对象**

    `Vertex.start` 和 `Vertex.end` 属性可用于 WHERE 条件和类似条件，使用临时 `Point` 对象进行比较：

    ```py
    >>> stmt = select(Vertex).where(Vertex.start == Point(3, 4)).where(Vertex.end < Point(7, 8))
    >>> session.scalars(stmt).all()
    SELECT  vertices.id,  vertices.x1,  vertices.y1,  vertices.x2,  vertices.y2
    FROM  vertices
    WHERE  vertices.x1  =  ?  AND  vertices.y1  =  ?  AND  vertices.x2  <  ?  AND  vertices.y2  <  ?
    [...]  (3,  4,  7,  8)
    [Vertex(Point(x=3, y=4), Point(x=5, y=6))]
    ```

    新版本 2.0 中：`composite()` 构造现在支持“排序”比较，如 `<`、`>=`，以及已经存在的 `==`、`!=` 的支持。

    提示

    使用“小于”运算符（`<`）的“排序”比较以及使用 `==` 的“相等性”比较，用于生成 SQL 表达式时，由 `Comparator` 类实现，并不使用复合类本身的比较方法，例如 `__lt__()` 或 `__eq__()` 方法。由此可见，上面的 `Point` 数据类也无需实现 dataclasses 的 `order=True` 参数，上述 SQL 操作就可以正常工作。复合操作重新定义比较操作 包含了如何定制比较操作的背景信息。

+   **在 Vertex 实例上更新 Point 对象**

    默认情况下，必须通过一个新对象来替换 `Point` 对象才能检测到更改：

    ```py
    >>> v1 = session.scalars(select(Vertex)).one()
    SELECT  vertices.id,  vertices.x1,  vertices.y1,  vertices.x2,  vertices.y2
    FROM  vertices
    [...]  ()
    >>> v1.end = Point(x=10, y=14)
    >>> session.commit()
    UPDATE  vertices  SET  x2=?,  y2=?  WHERE  vertices.id  =  ?
    [...]  (10,  14,  1)
    COMMIT 
    ```

    为了允许对复合对象进行原地更改，必须使用 Mutation Tracking 扩展。参见在复合对象上建立可变性部分中的示例。

## 复合对象的其他映射形式

`composite()` 构造可以使用 `mapped_column()` 构造、`Column` 或现有映射列的字符串名称来传递相关列。以下示例说明了与上述主要部分相同的等效映射。

### 直接映射列，然后传递给复合对象

在这里，我们将现有的 `mapped_column()` 实例传递给 `composite()` 构造，就像下面的非注释示例中我们还将 `Point` 类作为 `composite()` 的第一个参数传递一样：

```py
from sqlalchemy import Integer
from sqlalchemy.orm import mapped_column, composite

class Vertex(Base):
    __tablename__ = "vertices"

    id = mapped_column(Integer, primary_key=True)
    x1 = mapped_column(Integer)
    y1 = mapped_column(Integer)
    x2 = mapped_column(Integer)
    y2 = mapped_column(Integer)

    start = composite(Point, x1, y1)
    end = composite(Point, x2, y2)
```

### 直接映射列，将属性名称传递给复合对象

我们可以使用更多带有注释形式编写上面相同的示例，其中我们可以选择将属性名称传递给 `composite()` 而不是完整的列构造：

```py
from sqlalchemy.orm import mapped_column, composite, Mapped

class Vertex(Base):
    __tablename__ = "vertices"

    id: Mapped[int] = mapped_column(primary_key=True)
    x1: Mapped[int]
    y1: Mapped[int]
    x2: Mapped[int]
    y2: Mapped[int]

    start: Mapped[Point] = composite("x1", "y1")
    end: Mapped[Point] = composite("x2", "y2")
```

### 命令式映射和命令式表

在使用命令式表或完全命令式映射时，我们直接可以访问 `Column` 对象。这些也可以像以下命令式示例中那样传递给 `composite()`：

```py
mapper_registry.map_imperatively(
    Vertex,
    vertices_table,
    properties={
        "start": composite(Point, vertices_table.c.x1, vertices_table.c.y1),
        "end": composite(Point, vertices_table.c.x2, vertices_table.c.y2),
    },
)
```

### 直接映射列，然后传递给复合对象

在这里，我们将现有的 `mapped_column()` 实例传递给 `composite()` 构造，就像下面的非注释示例中我们还将 `Point` 类作为 `composite()` 的第一个参数传递一样：

```py
from sqlalchemy import Integer
from sqlalchemy.orm import mapped_column, composite

class Vertex(Base):
    __tablename__ = "vertices"

    id = mapped_column(Integer, primary_key=True)
    x1 = mapped_column(Integer)
    y1 = mapped_column(Integer)
    x2 = mapped_column(Integer)
    y2 = mapped_column(Integer)

    start = composite(Point, x1, y1)
    end = composite(Point, x2, y2)
```

### 直接映射列，将属性名称传递给复合对象

我们可以使用更多带有注释形式编写上面相同的示例，其中我们可以选择将属性名称传递给 `composite()` 而不是完整的列构造：

```py
from sqlalchemy.orm import mapped_column, composite, Mapped

class Vertex(Base):
    __tablename__ = "vertices"

    id: Mapped[int] = mapped_column(primary_key=True)
    x1: Mapped[int]
    y1: Mapped[int]
    x2: Mapped[int]
    y2: Mapped[int]

    start: Mapped[Point] = composite("x1", "y1")
    end: Mapped[Point] = composite("x2", "y2")
```

### 命令式映射和命令式表

当使用命令式表或完全命令式映射时，我们可以直接访问`Column`对象。这些也可以传递给`composite()`，如下所示的命令式示例：

```py
mapper_registry.map_imperatively(
    Vertex,
    vertices_table,
    properties={
        "start": composite(Point, vertices_table.c.x1, vertices_table.c.y1),
        "end": composite(Point, vertices_table.c.x2, vertices_table.c.y2),
    },
)
```

## 使用传统非数据类

如果不使用数据类，则自定义数据类型类的要求是它具有一个构造函数，该构造函数接受与其列格式对应的位置参数，并且还提供一个`__composite_values__()`方法，该方法按照其基于列的属性的顺序返回对象的状态列表或元组。它还应该提供足够的`__eq__()`和`__ne__()`方法来测试两个实例的相等性。

为了说明主要部分中的等效`Point`类不使用数据类的情况：

```py
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y

    def __composite_values__(self):
        return self.x, self.y

    def __repr__(self):
        return f"Point(x={self.x!r}, y={self.y!r})"

    def __eq__(self, other):
        return isinstance(other, Point) and other.x == self.x and other.y == self.y

    def __ne__(self, other):
        return not self.__eq__(other)
```

使用`composite()`进行如下操作，必须使用显式类型声明与`Point`类关联的列，使用其他复合对象映射形式中的一种形式。

## 跟踪复合对象的原位变化

对现有复合值的原位更改不会自动跟踪。相反，复合类需要显式为其父对象提供事件。通过使用`MutableComposite` mixin，这项任务主要通过使用事件将每个用户定义的复合对象与所有父关联关联起来来自动完成。请参阅为复合对象建立可变性中的示例。

## 重新定义复合对象的比较操作

默认情况下，“equals”比较操作会产生所有对应列等于彼此的 AND。可以使用`composite()`的`comparator_factory`参数进行更改，其中我们指定一个自定义的`Comparator`类来定义现有或新的操作。下面我们说明“greater than”运算符，实现与基本“greater than”相同的表达式：

```py
import dataclasses

from sqlalchemy.orm import composite
from sqlalchemy.orm import CompositeProperty
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.sql import and_

@dataclasses.dataclass
class Point:
    x: int
    y: int

class PointComparator(CompositeProperty.Comparator):
    def __gt__(self, other):
  """redefine the 'greater than' operation"""

        return and_(
            *[
                a > b
                for a, b in zip(
                    self.__clause_element__().clauses,
                    dataclasses.astuple(other),
                )
            ]
        )

class Base(DeclarativeBase):
    pass

class Vertex(Base):
    __tablename__ = "vertices"

    id: Mapped[int] = mapped_column(primary_key=True)

    start: Mapped[Point] = composite(
        mapped_column("x1"), mapped_column("y1"), comparator_factory=PointComparator
    )
    end: Mapped[Point] = composite(
        mapped_column("x2"), mapped_column("y2"), comparator_factory=PointComparator
    )
```

由于`Point`是一个数据类，我们可以利用`dataclasses.astuple()`来获得`Point`实例的元组形式。

然后，自定义比较器返回适当的 SQL 表达式：

```py
>>> print(Vertex.start > Point(5, 6))
vertices.x1  >  :x1_1  AND  vertices.y1  >  :y1_1 
```

## 嵌套复合对象

可以定义复合对象以在简单的嵌套方案中工作，方法是在复合类中重新定义所需的行为，然后将复合类映射到通常的单个列的完整长度。这要求定义额外的方法来在“嵌套”和“扁平”形式之间移动。

接下来，我们重新组织`Vertex`类本身成为一个引用`Point`对象的复合对象。 `Vertex`和`Point`可以是数据类，但是我们将向`Vertex`添加一个自定义构造方法，该方法可用于根据四个列值创建新的`Vertex`对象，我们将任意命名为`_generate()`并定义为类方法，以便我们可以通过将值传递给`Vertex._generate()`方法来创建新的`Vertex`对象。

我们还将实现`__composite_values__()`方法，这是由`composite()`构造（在使用传统非数据类中介绍）中识别的固定名称，表示以列值的扁平元组形式接收对象的标准方式，在这种情况下将取代通常的数据类导向方法。

通过我们的自定义`_generate()`构造函数和`__composite_values__()`序列化方法，我们现在可以在扁平列元组和包含`Point`实例的`Vertex`对象之间进行转换。 `Vertex._generate`方法作为`composite()`构造的第一个参数传递，用作新`Vertex`实例的来源，并且`__composite_values__()`方法将隐式地被`composite()`使用。

为了示例的目的，`Vertex`复合然后映射到一个名为`HasVertex`的类，其中包含最终包含四个源列的`Table`：

```py
from __future__ import annotations

import dataclasses
from typing import Any
from typing import Tuple

from sqlalchemy.orm import composite
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column

@dataclasses.dataclass
class Point:
    x: int
    y: int

@dataclasses.dataclass
class Vertex:
    start: Point
    end: Point

    @classmethod
    def _generate(cls, x1: int, y1: int, x2: int, y2: int) -> Vertex:
  """generate a Vertex from a row"""
        return Vertex(Point(x1, y1), Point(x2, y2))

    def __composite_values__(self) -> Tuple[Any, ...]:
  """generate a row from a Vertex"""
        return dataclasses.astuple(self.start) + dataclasses.astuple(self.end)

class Base(DeclarativeBase):
    pass

class HasVertex(Base):
    __tablename__ = "has_vertex"
    id: Mapped[int] = mapped_column(primary_key=True)
    x1: Mapped[int]
    y1: Mapped[int]
    x2: Mapped[int]
    y2: Mapped[int]

    vertex: Mapped[Vertex] = composite(Vertex._generate, "x1", "y1", "x2", "y2")
```

上述映射可以根据`HasVertex`、`Vertex`和`Point`来使用：

```py
hv = HasVertex(vertex=Vertex(Point(1, 2), Point(3, 4)))

session.add(hv)
session.commit()

stmt = select(HasVertex).where(HasVertex.vertex == Vertex(Point(1, 2), Point(3, 4)))

hv = session.scalars(stmt).first()
print(hv.vertex.start)
print(hv.vertex.end)
```

## 复合 API

| 对象名称 | 描述 |
| --- | --- |
| composite([_class_or_attr], *attrs, [group, deferred, raiseload, comparator_factory, active_history, init, repr, default, default_factory, compare, kw_only, info, doc], **__kw) | 返回用于与 Mapper 一起使用的基于复合列的属性。 |

```py
function sqlalchemy.orm.composite(_class_or_attr: None | Type[_CC] | Callable[..., _CC] | _CompositeAttrType[Any] = None, *attrs: _CompositeAttrType[Any], group: str | None = None, deferred: bool = False, raiseload: bool = False, comparator_factory: Type[Composite.Comparator[_T]] | None = None, active_history: bool = False, init: _NoArg | bool = _NoArg.NO_ARG, repr: _NoArg | bool = _NoArg.NO_ARG, default: Any | None = _NoArg.NO_ARG, default_factory: _NoArg | Callable[[], _T] = _NoArg.NO_ARG, compare: _NoArg | bool = _NoArg.NO_ARG, kw_only: _NoArg | bool = _NoArg.NO_ARG, info: _InfoType | None = None, doc: str | None = None, **__kw: Any) → Composite[Any]
```

返回用于与 Mapper 一起使用的基于复合列的属性。

查看映射文档部分复合列类型以获取完整的使用示例。

`composite()`返回的`MapperProperty`是`Composite`。

参数：

+   `class_` – “复合类型”类，或任何类方法或可调用对象，将根据顺序的列值生成复合对象的新实例。

+   `*attrs` –

    要映射的元素列表，可能包括：

    +   `Column`对象

    +   `mapped_column()`构造

    +   映射类上的其他属性的字符串名称，这些属性可以是任何其他 SQL 或对象映射的属性。例如，这可以允许一个复合属性引用到一个多对一的关系。

+   `active_history=False` – 当为`True`时，指示在替换时应加载标量属性的“先前”值，如果尚未加载。请参阅`column_property()`上的相同标志。

+   `group` – 标记为延迟加载时，此属性的组名。

+   `deferred` – 当为 True 时，列属性为“延迟加载”，意味着它不会立即加载，而是在首次访问实例上的属性时加载。另请参阅`deferred()`。

+   `comparator_factory` – 一个扩展`Comparator`的类，提供自定义的 SQL 子句生成以进行比较操作。

+   `doc` – 可选字符串，将应用为类绑定描述符的文档。

+   `info` – 可选的数据字典，将填充到此对象的`MapperProperty.info`属性中。

+   `init` – 特定于声明式数据类映射，指定映射属性是否应作为数据类处理生成的`__init__()`方法的一部分。

+   `repr` – 特定于声明式数据类映射，指定映射属性是否应作为数据类处理生成的`__repr__()`方法的一部分。

+   `default_factory` – 特定于声明式数据类映射，指定将作为数据类处理生成的`__init__()`方法的一部分而发生的默认值生成函数。

+   `compare` –

    特定于声明式数据类映射，指示在为映射类生成`__eq__()`和`__ne__()`方法时，此字段是否应包含在比较操作中。

    新功能在版本 2.0.0b4 中引入。

+   `kw_only` – 特定于声明式数据类映射，指示在生成`__init__()`时此字段是否应标记为关键字参数。
