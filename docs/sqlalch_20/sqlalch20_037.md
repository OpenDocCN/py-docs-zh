# 集合自定义和 API 详情

> 原文：[`docs.sqlalchemy.org/en/20/orm/collection_api.html`](https://docs.sqlalchemy.org/en/20/orm/collection_api.html)

`relationship()` 函数定义了两个类之间的链接。当链接定义了一对多或多对多的关系时，在加载和操作对象时，它被表示为 Python 集合。本节介绍了有关集合配置和技术的其他信息。

## 自定义集合访问

将一对多或多对多的关系映射为一组可通过父实例上的属性访问的值的集合。对于这些关系的两种常见集合类型是 `list` 和 `set`，在使用 `Mapped` 的 声明式 映射中，通过在 `Mapped` 容器中使用集合类型来建立，如下面的 `Parent.children` 集合中所示，其中使用了 `list`：

```py
from sqlalchemy import ForeignKey

from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

class Parent(Base):
    __tablename__ = "parent"

    parent_id: Mapped[int] = mapped_column(primary_key=True)

    # use a list
    children: Mapped[List["Child"]] = relationship()

class Child(Base):
    __tablename__ = "child"

    child_id: Mapped[int] = mapped_column(primary_key=True)
    parent_id: Mapped[int] = mapped_column(ForeignKey("parent.id"))
```

或者使用 `set`，在相同的 `Parent.children` 集合中进行说明：

```py
from typing import Set
from sqlalchemy import ForeignKey

from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

class Parent(Base):
    __tablename__ = "parent"

    parent_id: Mapped[int] = mapped_column(primary_key=True)

    # use a set
    children: Mapped[Set["Child"]] = relationship()

class Child(Base):
    __tablename__ = "child"

    child_id: Mapped[int] = mapped_column(primary_key=True)
    parent_id: Mapped[int] = mapped_column(ForeignKey("parent.id"))
```

注意

如果使用 Python 3.7 或 3.8，则集合的注释需要使用 `typing.List` 或 `typing.Set`，例如 `Mapped[List["Child"]]` 或 `Mapped[Set["Child"]]`；在这些 Python 版本中，`list` 和 `set` Python 内置的类型尚不支持通用注释，例如：

```py
from typing import List

class Parent(Base):
    __tablename__ = "parent"

    parent_id: Mapped[int] = mapped_column(primary_key=True)

    # use a List, Python 3.8 and earlier
    children: Mapped[List["Child"]] = relationship()
```

当使用没有 `Mapped` 注释的映射时，比如使用 命令式映射 或者未经类型化的 Python 代码，以及在一些特殊情况下，`relationship()` 的集合类始终可以直接使用 `relationship.collection_class` 参数进行指定：

```py
# non-annotated mapping

class Parent(Base):
    __tablename__ = "parent"

    parent_id = mapped_column(Integer, primary_key=True)

    children = relationship("Child", collection_class=set)

class Child(Base):
    __tablename__ = "child"

    child_id = mapped_column(Integer, primary_key=True)
    parent_id = mapped_column(ForeignKey("parent.id"))
```

在缺少 `relationship.collection_class` 或 `Mapped` 的情况下，默认的集合类型是 `list`。

除了内置的 `list` 和 `set`，还支持两种字典的变体，下文将进行描述 字典集合。还支持任何任意可变序列类型可以设置为目标集合，需要一些额外的配置步骤；这在 自定义集合实现 部分进行了描述。

### 字典集合

当使用字典作为集合时需要一些额外的细节。这是因为对象总是作为列表从数据库加载的，必须提供一种键生成策略才能正确地填充字典。`attribute_keyed_dict()`函数是实现简单字典集合的最常见方式。它生成一个字典类，该类将映射类的特定属性作为键。下面我们映射了一个包含以`Note.keyword`属性作为键的`Note`项目字典的`Item`类。当使用`attribute_keyed_dict()`时，可以使用`Mapped`注释，可以使用`KeyFuncDict`或仅使用普通的`dict`，如下面的示例所示。然而，在这种情况下，需要使用`relationship.collection_class`参数，以便适当地参数化`attribute_keyed_dict()`：

```py
from typing import Dict
from typing import Optional

from sqlalchemy import ForeignKey
from sqlalchemy.orm import attribute_keyed_dict
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

class Item(Base):
    __tablename__ = "item"

    id: Mapped[int] = mapped_column(primary_key=True)

    notes: Mapped[Dict[str, "Note"]] = relationship(
        collection_class=attribute_keyed_dict("keyword"),
        cascade="all, delete-orphan",
    )

class Note(Base):
    __tablename__ = "note"

    id: Mapped[int] = mapped_column(primary_key=True)
    item_id: Mapped[int] = mapped_column(ForeignKey("item.id"))
    keyword: Mapped[str]
    text: Mapped[Optional[str]]

    def __init__(self, keyword: str, text: str):
        self.keyword = keyword
        self.text = text
```

`Item.notes`然后是一个字典：

```py
>>> item = Item()
>>> item.notes["a"] = Note("a", "atext")
>>> item.notes.items()
{'a': <__main__.Note object at 0x2eaaf0>}
```

`attribute_keyed_dict()`将确保每个`Note`的`.keyword`属性与字典中的键相匹配。例如，当分配给`Item.notes`时，我们提供的字典键必须与实际`Note`对象的键相匹配：

```py
item = Item()
item.notes = {
    "a": Note("a", "atext"),
    "b": Note("b", "btext"),
}
```

`attribute_keyed_dict()`用作键的属性根本不需要被映射！使用常规的 Python `@property` 允许使用对象的几乎任何细节或组合细节作为键，就像下面我们将其建立为`Note.keyword`元组和`Note.text`字段的前十个字母一样：

```py
class Item(Base):
    __tablename__ = "item"

    id: Mapped[int] = mapped_column(primary_key=True)

    notes: Mapped[Dict[str, "Note"]] = relationship(
        collection_class=attribute_keyed_dict("note_key"),
        back_populates="item",
        cascade="all, delete-orphan",
    )

class Note(Base):
    __tablename__ = "note"

    id: Mapped[int] = mapped_column(primary_key=True)
    item_id: Mapped[int] = mapped_column(ForeignKey("item.id"))
    keyword: Mapped[str]
    text: Mapped[str]

    item: Mapped["Item"] = relationship()

    @property
    def note_key(self):
        return (self.keyword, self.text[0:10])

    def __init__(self, keyword: str, text: str):
        self.keyword = keyword
        self.text = text
```

在上面，我们添加了一个带有双向`relationship.back_populates`配置的`Note.item`关系。将`Note`分配给这个反向关系时，`Note`被添加到`Item.notes`字典中，并且键会自动为我们生成：

```py
>>> item = Item()
>>> n1 = Note("a", "atext")
>>> n1.item = item
>>> item.notes
{('a', 'atext'): <__main__.Note object at 0x2eaaf0>}
```

其他内置的字典类型包括`column_keyed_dict()`，几乎与`attribute_keyed_dict()`类似，只是直接给出`Column`对象：

```py
from sqlalchemy.orm import column_keyed_dict

class Item(Base):
    __tablename__ = "item"

    id: Mapped[int] = mapped_column(primary_key=True)

    notes: Mapped[Dict[str, "Note"]] = relationship(
        collection_class=column_keyed_dict(Note.__table__.c.keyword),
        cascade="all, delete-orphan",
    )
```

以及`mapped_collection()`，它接受任何可调用函数。请注意，通常更容易使用前面提到的`@property`与`attribute_keyed_dict()`一起使用：

```py
from sqlalchemy.orm import mapped_collection

class Item(Base):
    __tablename__ = "item"

    id: Mapped[int] = mapped_column(primary_key=True)

    notes: Mapped[Dict[str, "Note"]] = relationship(
        collection_class=mapped_collection(lambda note: note.text[0:10]),
        cascade="all, delete-orphan",
    )
```

字典映射经常与“Association Proxy”扩展组合以产生简化的字典视图。请参阅 Proxying to Dictionary Based Collections 和 Composite Association Proxies 以获取示例。

#### 处理键突变和字典集合的反向填充

当使用`attribute_keyed_dict()`时，字典的“键”来自目标对象上的属性。**对此键的更改不会被跟踪**。这意味着必须在第一次使用时分配键，并且如果键发生更改，则集合将不会突变。在依赖于反向引用来填充属性映射集合时，这可能是一个典型的问题。给定以下情况：

```py
class A(Base):
    __tablename__ = "a"

    id: Mapped[int] = mapped_column(primary_key=True)

    bs: Mapped[Dict[str, "B"]] = relationship(
        collection_class=attribute_keyed_dict("data"),
        back_populates="a",
    )

class B(Base):
    __tablename__ = "b"

    id: Mapped[int] = mapped_column(primary_key=True)
    a_id: Mapped[int] = mapped_column(ForeignKey("a.id"))
    data: Mapped[str]

    a: Mapped["A"] = relationship(back_populates="bs")
```

上面，如果我们创建一个引用特定`A()`的`B()`，那么反向填充将将`B()`添加到`A.bs`集合中，但是如果`B.data`的值尚未设置，则键将为`None`：

```py
>>> a1 = A()
>>> b1 = B(a=a1)
>>> a1.bs
{None: <test3.B object at 0x7f7b1023ef70>}
```

设置`b1.data`之后不会更新集合：

```py
>>> b1.data = "the key"
>>> a1.bs
{None: <test3.B object at 0x7f7b1023ef70>}
```

如果尝试在构造函数中设置`B()`，也可以看到这一点。参数顺序更改了结果：

```py
>>> B(a=a1, data="the key")
<test3.B object at 0x7f7b10114280>
>>> a1.bs
{None: <test3.B object at 0x7f7b10114280>}
```

vs：

```py
>>> B(data="the key", a=a1)
<test3.B object at 0x7f7b10114340>
>>> a1.bs
{'the key': <test3.B object at 0x7f7b10114340>}
```

如果正在以这种方式使用反向引用，请确保使用`__init__`方法按正确顺序填充属性。

以下事件处理程序也可以用于跟踪集合中的更改：

```py
from sqlalchemy import event
from sqlalchemy.orm import attributes

@event.listens_for(B.data, "set")
def set_item(obj, value, previous, initiator):
    if obj.a is not None:
        previous = None if previous == attributes.NO_VALUE else previous
        obj.a.bs[value] = obj
        obj.a.bs.pop(previous)
```  ## 自定义集合实现

您也可以为集合使用自己的类型。在简单情况下，继承自`list`或`set`，添加自定义行为就足够了。在其他情况下，需要特殊的装饰器来告诉 SQLAlchemy 关于集合操作的更多详细信息。

SQLAlchemy 中的集合是透明的*instrumented*。仪器化意味着对集合的常规操作将被跟踪，并且在刷新时将更改写入数据库。此外，集合操作可以触发*事件*，这些事件表明必须进行某些次要操作。次要操作的示例包括将子项保存在父项的`Session`中（即`save-update`级联），以及同步双向关系的状态（即`backref()`）。

集合包理解列表、集合和字典的基本接口，并将自动对这些内置类型及其子类应用仪表化。实现基本集合接口的对象衍生类型会通过鸭子类型检测到并进行仪表化：

```py
class ListLike:
    def __init__(self):
        self.data = []

    def append(self, item):
        self.data.append(item)

    def remove(self, item):
        self.data.remove(item)

    def extend(self, items):
        self.data.extend(items)

    def __iter__(self):
        return iter(self.data)

    def foo(self):
        return "foo"
```

`append`、`remove`和`extend`是`list`的已知成员，并且将自动进行仪表化。`__iter__`不是一个修改器方法，不会进行仪表化，`foo`也不会进行仪表化。

当然，鸭子类型（即猜测）并不是十分可靠，因此您可以通过提供`__emulates__`类属性明确地指定您要实现的接口：

```py
class SetLike:
    __emulates__ = set

    def __init__(self):
        self.data = set()

    def append(self, item):
        self.data.add(item)

    def remove(self, item):
        self.data.remove(item)

    def __iter__(self):
        return iter(self.data)
```

这个类看起来类似于 Python 的`list`（即“类似列表”），因为它有一个`append`方法，但是`__emulates__`属性将其强制视为`set`。`remove`被认为是集合接口的一部分，并将被仪表化。

但是这个类目前还不起作用：需要一点粘合剂来使其适应 SQLAlchemy 的使用。ORM 需要知道使用哪些方法来附加、删除和迭代集合的成员。当使用`list`或`set`等类型时，适当的方法是众所周知的，并且在存在时会自动使用。然而，上面的类只粗略地类似于`set`，并没有提供预期的`add`方法，因此我们必须告诉 ORM 将代替`add`方法的方法，在本例中使用装饰器`@collection.appender`来说明这一点；这将在下一节中进行说明。

### 通过装饰器对自定义集合进行注释

当您的类不完全符合其容器类型的常规接口时，或者当您希望以不同的方法完成工作时，可以使用装饰器标记单个方法供 ORM 管理集合时使用。

```py
from sqlalchemy.orm.collections import collection

class SetLike:
    __emulates__ = set

    def __init__(self):
        self.data = set()

    @collection.appender
    def append(self, item):
        self.data.add(item)

    def remove(self, item):
        self.data.remove(item)

    def __iter__(self):
        return iter(self.data)
```

这就是完成示例所需的全部内容。SQLAlchemy 将通过`append`方法添加实例。`remove`和`__iter__`是集合的默认方法，并将用于删除和迭代。默认方法也可以更改：

```py
from sqlalchemy.orm.collections import collection

class MyList(list):
    @collection.remover
    def zark(self, item):
        # do something special...
        ...

    @collection.iterator
    def hey_use_this_instead_for_iteration(self): ...
```

完全不需要“类似列表”或“类似集合”。集合类可以是任何形状，只要它们具有由 SQLAlchemy 标记的附加、删除和迭代接口。附加和删除方法将以映射的实体作为单个参数调用，迭代器方法将不带参数调用，并且必须返回一个迭代器。

### 自定义基于字典的集合

`KeyFuncDict`类可用作自定义类型的基类，也可以用作快速将`dict`集合支持添加到其他类的混合。它使用键函数来委托给`__setitem__`和`__delitem__`：

```py
from sqlalchemy.orm.collections import KeyFuncDict

class MyNodeMap(KeyFuncDict):
  """Holds 'Node' objects, keyed by the 'name' attribute."""

    def __init__(self, *args, **kw):
        super().__init__(keyfunc=lambda node: node.name)
        dict.__init__(self, *args, **kw)
```

当子类化 `KeyFuncDict` 时，如果调用相同的方法 `__setitem__()` 或 `__delitem__()`，则用户定义的版本应当被装饰 `collection.internally_instrumented()`，**如果** 它们在 `KeyFuncDict` 上调用这些方法。因为 `KeyFuncDict` 上的方法已经被内部装饰 - 在已经被内部装饰的调用中调用它们可能会导致事件被重复触发，或不恰当地，在极少数情况下导致内部状态损坏：

```py
from sqlalchemy.orm.collections import KeyFuncDict, collection

class MyKeyFuncDict(KeyFuncDict):
  """Use @internally_instrumented when your methods
 call down to already-instrumented methods.

 """

    @collection.internally_instrumented
    def __setitem__(self, key, value, _sa_initiator=None):
        # do something with key, value
        super(MyKeyFuncDict, self).__setitem__(key, value, _sa_initiator)

    @collection.internally_instrumented
    def __delitem__(self, key, _sa_initiator=None):
        # do something with key
        super(MyKeyFuncDict, self).__delitem__(key, _sa_initiator)
```

ORM 理解 `dict` 接口就像列表和集合一样，并且如果选择子类化 `dict` 或在鸭子类型类中提供类似于 dict 的集合行为，则会自动为所有“类似于字典”的方法进行仪器化。然而，您必须装饰添加器和删除器方法-因为 SQLAlchemy 没有默认使用的基本字典接口的兼容方法。迭代将通过 `values()` 进行，除非另有装饰。

### 仪器化和自定义类型

许多自定义类型和现有库类可以直接用作实体集合类型而无需进一步操作。但是，重要的是要注意，仪器化过程将修改类型，自动在方法周围添加装饰器。

这些装饰很轻量级，在关系之外是无操作的，但是在其他地方触发时会增加不必要的开销。当将库类用作集合时，将装饰限制为仅在关系中使用的“简单子类”技巧是一个好习惯。例如：

```py
class MyAwesomeList(some.great.library.AwesomeList):
    pass

# ... relationship(..., collection_class=MyAwesomeList)
```

ORM 对内置类型使用此方法，当直接使用 `list`、`set` 或 `dict` 时，会悄悄地替换为一个简单的子类。

## 集合 API

| 对象名称 | 描述 |
| --- | --- |
| attribute_keyed_dict(attr_name, *, [ignore_unpopulated_attribute]) | 基于字典的集合类型，具有基于属性的键。 |
| attribute_mapped_collection | 基于字典的集合类型，具有基于属性的键。 |
| column_keyed_dict(mapping_spec, *, [ignore_unpopulated_attribute]) | 基于列的键的字典型集合类型。 |
| column_mapped_collection | 基于列的键的字典型集合类型。 |
| keyfunc_mapping(keyfunc, *, [ignore_unpopulated_attribute]) | 基于字典的集合类型，具有任意的键。 |
| KeyFuncDict | 基于 ORM 映射字典类的基类。 |
| mapped_collection | 基于字典的集合类型，具有任意的键。 |
| MappedCollection | ORM 映射字典类的基类。 |

```py
function sqlalchemy.orm.attribute_keyed_dict(attr_name: str, *, ignore_unpopulated_attribute: bool = False) → Type[KeyFuncDict[Any, Any]]
```

基于属性键的字典类型的集合。

版本 2.0 中的更改：将`attribute_mapped_collection`重命名为`attribute_keyed_dict()`。

返回一个`KeyFuncDict`工厂，它将根据 ORM 映射实例上的特定命名属性的值生成新的字典键，以添加到字典中。

注意

目标属性的值必须在将对象添加到字典集合时被赋予其值。另外，**不会跟踪**键属性的更改，这意味着字典中的键不会自动与目标对象本身的键值同步。有关详细信息，请参阅处理键变异和反向填充字典集合。

另请参见

字典集合 - 使用背景

参数：

+   `attr_name` - 映射类上的 ORM 映射属性的字符串名称，其值将在特定实例上用作新字典条目的键。

+   `ignore_unpopulated_attribute` -

    如果为 True，并且对象上的目标属性根本未填充，则操作将被静默跳过。默认情况下，会引发错误。

    版本 2.0 中的新特性：如果确定用于字典键的属性从未填充过任何值，则默认会引发错误。可以设置`attribute_keyed_dict.ignore_unpopulated_attribute`参数，该参数将指示忽略此条件，并在静默跳过附加操作。这与 1.x 系列的行为相反，后者会错误地使用任意键值`None`填充字典中的值。

```py
function sqlalchemy.orm.column_keyed_dict(mapping_spec: Type[_KT] | Callable[[_KT], _VT], *, ignore_unpopulated_attribute: bool = False) → Type[KeyFuncDict[_KT, _KT]]
```

基于列键的字典类型的集合。

版本 2.0 中的更改：将`column_mapped_collection`重命名为`column_keyed_dict`。

返回一个`KeyFuncDict`工厂，它将根据 ORM 映射实例上的特定`Column`映射属性的值生成新的字典键，以添加到字典中。

注意

目标属性的值必须在将对象添加到字典集合时分配其值。此外，**不会跟踪**键属性的更改，这意味着字典中的键不会自动与目标对象本身的键值同步。有关详细信息，请参见处理键突变和为字典集合回填。

另请参阅

字典集合 - 使用背景

参数：

+   `mapping_spec` - 一个预期由目标映射器映射到映射类上特定属性的`Column`对象，其在特定实例上的值将用作该实例的新字典条目的键。

+   `ignore_unpopulated_attribute` - 

    如果为 True，并且对象上由给定`Column`目标属性指示的映射属性根本未填充，则操作将被静默跳过。默认情况下，会引发错误。

    2.0 版本中的新功能：如果确定用于字典键的属性从未填充任何值，则默认情况下会引发错误。可以设置`column_keyed_dict.ignore_unpopulated_attribute`参数，该参数将指示应忽略此条件，并且附加操作将被静默跳过。这与 1.x 系列的行为相反，后者会错误地使用任意键值`None`填充字典中的值。

```py
function sqlalchemy.orm.keyfunc_mapping(keyfunc: _F, *, ignore_unpopulated_attribute: bool = False) → Type[KeyFuncDict[_KT, Any]]
```

基于字典的集合类型，具有任意键。

2.0 版本中的更改：将`mapped_collection`重命名为`keyfunc_mapping()`。

返回一个从 keyfunc 生成的键函数的`KeyFuncDict`工厂，一个可调用对象，接受一个实体并返回一个键值。

注意

给定的 keyfunc 仅在将目标对象添加到集合时调用一次。不会跟踪函数返回的有效值的更改。

另请参阅

字典集合 - 使用背景

参数：

+   `keyfunc` - 一个可调用对象，将传递 ORM 映射的实例，然后生成一个用于字典中的新键。如果返回的值是`LoaderCallableStatus.NO_VALUE`，则会引发错误。

+   `ignore_unpopulated_attribute` - 

    如果为 True，并且可调用函数对特定实例返回`LoaderCallableStatus.NO_VALUE`，则操作将被静默跳过。默认情况下会引发错误。

    2.0 版本中的新功能：如果用于字典键的可调用函数返回`LoaderCallableStatus.NO_VALUE`，则默认情况下会引发错误，这在 ORM 属性上下文中表示从未填充任何值的属性。可以设置`mapped_collection.ignore_unpopulated_attribute`参数，该参数将指示应忽略此条件，并且附加操作将被静默跳过。这与 1.x 系列的行为相反，后者会错误地使用任意键值`None`填充字典中的值。

```py
sqlalchemy.orm.attribute_mapped_collection = <function attribute_keyed_dict>
```

基于属性的键的字典集合类型。

2.0 版本中的更改：将`attribute_mapped_collection`重命名为`attribute_keyed_dict()`。

返回一个`KeyFuncDict`工厂，该工厂将根据 ORM 映射实例上特定命名属性的值生成新的字典键，以添加到字典中。

注意

目标属性的值必须在将对象添加到字典集合时分配其值。此外，**不会跟踪**键属性的更改，这意味着字典中的键不会自动与目标对象本身的键值同步。有关详细信息，请参阅处理键突变和为字典集合回填。

另请参见

字典集合 - 使用背景

参数：

+   `attr_name` – 映射类上 ORM 映射属性的字符串名称，特定实例上的该值将用作该实例的新字典条目的键。

+   `ignore_unpopulated_attribute` –

    如果为 True，并且对象上的目标属性根本未填充，则操作将被静默跳过。默认情况下会引发错误。

    2.0 版新功能：默认情况下，如果确定用于字典键的属性从未被填充任何值，则将引发错误。可以设置 `attribute_keyed_dict.ignore_unpopulated_attribute` 参数，以指示应忽略此条件，并静默跳过追加操作。这与 1.x 系列的行为相反，后者将错误地使用任意键值 `None` 填充字典中的值。

```py
sqlalchemy.orm.column_mapped_collection = <function column_keyed_dict>
```

基于字典的集合类型，使用列作为键。

2.0 版更改：将 `column_mapped_collection` 重命名为 `column_keyed_dict`。

返回一个 `KeyFuncDict` 工厂，它将根据 ORM 映射实例上的特定 `Column` 映射的属性的值产生新的字典键，并将其添加到字典中。

注意

目标属性的值必须在将对象添加到字典集合时被赋值。另外，不会**跟踪**键属性的更改，这意味着字典中的键不会自动与目标对象本身的键值同步。参见处理键变化和字典集合的反填充获取更多详细信息。

另见

字典集合 - 使用背景

参数：

+   `mapping_spec` – 一个预期由目标映射器映射到映射类上特定属性的 `Column` 对象，其在特定实例上的值将用作该实例的新字典条目的键。

+   `ignore_unpopulated_attribute` –

    如果为 True，并且对象上由给定 `Column` 目标属性指示的映射属性根本未被填充，则操作将被静默跳过。默认情况下，将引发错误。

    2.0 版新功能：默认情况下，如果确定用于字典键的属性从未被填充任何值，则将引发错误。可以设置 `column_keyed_dict.ignore_unpopulated_attribute` 参数，以指示应忽略此条件，并静默跳过追加操作。这与 1.x 系列的行为相反，后者将错误地使用任意键值 `None` 填充字典中的值。

```py
sqlalchemy.orm.mapped_collection = <function keyfunc_mapping>
```

一种基于字典的集合类型，具有任意键。

从版本 2.0 开始更改：将`mapped_collection`重命名为`keyfunc_mapping()`。

返回一个`KeyFuncDict`工厂，其中包含从 keyfunc 生成的键函数，一个接受实体并返回键值的可调用对象。

注意

给定的 keyfunc 仅在将目标对象添加到集合时调用一次。不会跟踪函数返回的有效值的更改。

另请参见

字典集合 - 使用背景

参数：

+   `keyfunc` – 一个可调用对象，将传递给 ORM 映射的实例，然后生成一个新的键用于字典。如果返回的值是`LoaderCallableStatus.NO_VALUE`，则会引发错误。

+   `ignore_unpopulated_attribute` –

    如果为 True，并且可调用对象对于特定实例返回`LoaderCallableStatus.NO_VALUE`，则操作将被静默跳过。默认情况下，会引发错误。

    从版本 2.0 开始：如果用于字典键的可调用对象返回`LoaderCallableStatus.NO_VALUE`，则默认情况下会引发错误，这在 ORM 属性上下文中表示从未用任何值填充的属性。可以设置`mapped_collection.ignore_unpopulated_attribute`参数，该参数将指示应忽略此条件，并且附加操作将被静默跳过。这与 1.x 系列的行为相反，后者会错误地使用任意键值`None`填充字典中的值。

```py
class sqlalchemy.orm.KeyFuncDict
```

ORM 映射字典类的基础。

使用额外方法扩展了`dict`类型，这些方法是 SQLAlchemy ORM 集合类所需的。最直接使用`attribute_keyed_dict()`或`column_keyed_dict()`类工厂来使用`KeyFuncDict`。`KeyFuncDict`也可以作为用户定义的自定义字典类的基础。

从版本 2.0 开始更改：将`MappedCollection`重命名为`KeyFuncDict`。

另请参见

`attribute_keyed_dict()`

`column_keyed_dict()`

字典集合

自定义集合实现

**成员**

__init__(), clear(), pop(), popitem(), remove(), set(), setdefault(), update()

**类签名**

类 `sqlalchemy.orm.KeyFuncDict` (`builtins.dict`, `typing.Generic`)

```py
method __init__(keyfunc: _F, *dict_args: Any, ignore_unpopulated_attribute: bool = False) → None
```

使用 keyfunc 提供的键创建一个新的集合。

keyfunc 可以是任何接受对象并返回用作字典键的对象的可调用对象。

每次 ORM 需要按值添加成员（例如从数据库加载实例时）或移除成员时都会调用 keyfunc。通常的字典键警告适用- `keyfunc(object)` 应该在集合的生命周期内返回相同的输出。基于可变属性的键可能会导致集合中“丢失”的不可达实例。

```py
method clear() → None.  Remove all items from D.
```

```py
method pop(k[, d]) → v, remove specified key and return the corresponding value.
```

如果未找到键，则返回给定的默认值；否则，引发 KeyError。

```py
method popitem()
```

移除并返回一个（键，值）对作为 2 元组。

对中的对以 LIFO（后进先出）顺序返回。如果字典为空，则引发 KeyError。

```py
method remove(value: _KT, _sa_initiator: AttributeEventToken | Literal[None, False] = None) → None
```

通过值删除项，查询 keyfunc 以获取键。

```py
method set(value: _KT, _sa_initiator: AttributeEventToken | Literal[None, False] = None) → None
```

通过值添加项，查询 keyfunc 以获取键。

```py
method setdefault(key, default=None)
```

如果键不在字典中，则将键插入并将默认值设置为默认值。

如果键在字典中，则返回键的值，否则返回默认值。

```py
method update([E, ]**F) → None.  Update D from dict/iterable E and F.
```

如果 E 存在并且具有 .keys() 方法，则执行以下操作：for k in E: D[k] = E[k] 如果 E 存在并且缺少 .keys() 方法，则执行以下操作：for k, v in E: D[k] = v 在任一情况下，这之后都会执行：for k in F: D[k] = F[k]

```py
sqlalchemy.orm.MappedCollection = <class 'sqlalchemy.orm.mapped_collection.KeyFuncDict'>
```

ORM 映射字典类的基础。

扩展了 `dict` 类型，提供了 SQLAlchemy ORM 集合类所需的附加方法。最直接使用 `attribute_keyed_dict()` 或 `column_keyed_dict()` 类工厂创建 `KeyFuncDict`。`KeyFuncDict` 也可以作为用户定义的自定义字典类的基类。

在 2.0 版本中更改：将 `MappedCollection` 重命名为 `KeyFuncDict`。

另请参阅

`attribute_keyed_dict()`

`column_keyed_dict()`

字典集合

自定义集合实现

## 集合内部

| 对象名称 | 描述 |
| --- | --- |
| bulk_replace(values, existing_adapter, new_adapter[, initiator]) | 加载一个新的集合，根据先前的成员关系触发事件。 |
| collection | 实体集合类的装饰器。 |
| collection_adapter | attrgetter(attr, …) –> attrgetter 对象 |
| CollectionAdapter | ORM 和任意 Python 集合之间的桥梁。 |
| InstrumentedDict | 内置字典的受控版本。 |
| InstrumentedList | 内置列表的受控版本。 |
| InstrumentedSet | 内置集合的受控版本。 |
| prepare_instrumentation(factory) | 为将来用作集合类工厂的可调用对象做准备。 |

```py
function sqlalchemy.orm.collections.bulk_replace(values, existing_adapter, new_adapter, initiator=None)
```

加载一个新的集合，根据先前的成员关系触发事件。

将`values`中的实例附加到`new_adapter`。对于`existing_adapter`中不存在的任何实例，将触发事件。对于`existing_adapter`中存在但在`values`中不存在的任何实例，将触发删除事件。

参数：

+   `values` – 一个集合成员实例的可迭代对象

+   `existing_adapter` – 要替换的实例的`CollectionAdapter`

+   `new_adapter` – 一个空的`CollectionAdapter`，用于加载`values`

```py
class sqlalchemy.orm.collections.collection
```

实体集合类的装饰器。

这些装饰器分为两组：注释和拦截配方。

标注装饰器（appender, remover, iterator, converter, internally_instrumented）指示方法的目的，并且不接受任何参数。它们不带括号：

```py
@collection.appender
def append(self, append): ...
```

所有配方装饰器都需要括号，即使没有参数：

**成员**

adds(), appender(), converter(), internally_instrumented(), iterator(), remover(), removes(), removes_return(), replaces()

```py
@collection.adds('entity')
def insert(self, position, entity): ...

@collection.removes_return()
def popitem(self): ...
```

```py
method static adds(arg)
```

将方法标记为向集合添加实体。

将“添加到集合”处理添加到方法中。装饰器参数指示哪个方法参数保存着与 SQLAlchemy 相关的值。参数可以通过位置指定（即整数），也可以通过名称指定：

```py
@collection.adds(1)
def push(self, item): ...

@collection.adds('entity')
def do_stuff(self, thing, entity=None): ...
```

```py
method static appender(fn)
```

将方法标记为集合附加器。

调用 appender 方法时，将使用一个位置参数：要附加的值。如果尚未装饰该方法，将自动用 'adds(1)' 装饰该方法：

```py
@collection.appender
def add(self, append): ...

# or, equivalently
@collection.appender
@collection.adds(1)
def add(self, append): ...

# for mapping type, an 'append' may kick out a previous value
# that occupies that slot.  consider d['a'] = 'foo'- any previous
# value in d['a'] is discarded.
@collection.appender
@collection.replaces(1)
def add(self, entity):
    key = some_key_func(entity)
    previous = None
    if key in self:
        previous = self[key]
    self[key] = entity
    return previous
```

如果要附加的值不允许在集合中，您可以引发异常。需要记住的一件事是，对于由数据库查询映射的每个对象，都会调用 appender。如果数据库包含违反集合语义的行，则需要有创造性地解决问题，因为通过集合访问将无法正常工作。

如果 appender 方法在内部被仪器化，则还必须接收关键字参数 ‘_sa_initiator’ 并确保其在集合事件中的传播。

```py
method static converter(fn)
```

将方法标记为集合转换器。

自版本 1.3 起弃用：`collection.converter()` 处理程序已弃用，并将在将来的版本中删除。请参考 `listen()` 函数结合 `bulk_replace` 监听器接口。

当要完全替换集合时，将调用此可选方法，例如：

```py
myobj.acollection = [newvalue1, newvalue2]
```

转换器方法将接收到被分配的对象，并应返回适用于 `appender` 方法使用的值的可迭代对象。转换器不能分配值或改变集合，它的唯一工作是将用户提供的值适应为 ORM 使用的值的可迭代对象。

默认的转换器实现将使用鸭子类型进行转换。类似字典的集合将转换为字典值的可迭代对象，而其他类型将简单地进行迭代：

```py
@collection.converter
def convert(self, other): ...
```

如果对象的鸭子类型与此集合的类型不匹配，则会引发 TypeError。

如果要扩展可以批量分配的可能类型的范围，或者对即将被分配的值执行验证，请提供此方法的实现。

```py
method static internally_instrumented(fn)
```

将方法标记为受仪器控制的。

此标记将阻止对该方法应用任何装饰。如果您正在编排自己对 `collection_adapter()` 的调用，并且在基本 SQLAlchemy 接口方法中的一个方法中，或者防止自动 ABC 方法装饰包装您的实现，请使用此标记：

```py
# normally an 'extend' method on a list-like class would be
# automatically intercepted and re-implemented in terms of
# SQLAlchemy events and append().  your implementation will
# never be called, unless:
@collection.internally_instrumented
def extend(self, items): ...
```

```py
method static iterator(fn)
```

将方法标记为集合移除器。

调用 iterator 方法时不使用参数。预期它将返回所有集合成员的迭代器：

```py
@collection.iterator
def __iter__(self): ...
```

```py
method static remover(fn)
```

将方法标记为集合移除器。

`remover` 方法接受一个位置参数：要移除的值。如果方法尚未装饰，则会自动装饰为 `removes_return()`：

```py
@collection.remover
def zap(self, entity): ...

# or, equivalently
@collection.remover
@collection.removes_return()
def zap(self, ): ...
```

如果要移除的值在集合中不存在，则可以引发异常或返回 None 以忽略错误。

如果 remove 方法在内部被检测，则还必须接收关键字参数 ‘_sa_initiator’ 并确保其传播到集合事件。

```py
method static removes(arg)
```

将该方法标记为从集合中移除实体。

将“从集合中移除”处理添加到方法中。装饰器参数指示哪个方法参数保存了要从 SQLAlchemy 中移除的值。参数可以通过位置（即整数）或名称指定：

```py
@collection.removes(1)
def zap(self, item): ...
```

对于在调用时不知道要移除的值的方法，请使用 `collection.removes_return`。

```py
method static removes_return()
```

将该方法标记为从集合中移除实体。

将“从集合中移除”处理添加到方法中。如果有，则方法的返回值将被视为要移除的值。不会检查方法参数：

```py
@collection.removes_return()
def pop(self): ...
```

对于在调用时知道要移除的值的方法，请使用 `collection.remove`。

```py
method static replaces(arg)
```

将该方法标记为替换集合中的实体。

将“添加到集合中”和“从集合中移除”处理添加到方法中。装饰器参数指示哪个方法参数保存了要添加到 SQLAlchemy 中的值，如果有，则返回值将被视为要移除的值。

参数可以通过位置（即整数）或名称指定：

```py
@collection.replaces(2)
def __setitem__(self, index, item): ...
```

```py
sqlalchemy.orm.collections.collection_adapter = operator.attrgetter('_sa_adapter')
```

`attrgetter(attr, …)` –> `attrgetter` 对象

返回一个可调用对象，它从其操作数中获取给定属性。执行 `f = attrgetter('name')` 后，调用 `f(r)` 返回 `r.name`。执行 `g = attrgetter('name', 'date')` 后，调用 `g(r)` 返回 `(r.name, r.date)`。执行 `h = attrgetter('name.first', 'name.last')` 后，调用 `h(r)` 返回 `(r.name.first, r.name.last)`。

```py
class sqlalchemy.orm.collections.CollectionAdapter
```

在 ORM 和任意 Python 集合之间架设桥梁。

将基本级别的集合操作（追加、移除、迭代）代理到底层的 Python 集合，并为进入或离开集合的实体发出添加/移除事件。

ORM 专门使用 `CollectionAdapter` 与实体集合交互。

```py
class sqlalchemy.orm.collections.InstrumentedDict
```

内置字典的受检测版本。

**类签名**

`sqlalchemy.orm.collections.InstrumentedDict` 类（`builtins.dict`, `typing.Generic`）

```py
class sqlalchemy.orm.collections.InstrumentedList
```

内置列表的受检测版本。

**类签名**

`sqlalchemy.orm.collections.InstrumentedList` 类（`builtins.list`, `typing.Generic`）

```py
class sqlalchemy.orm.collections.InstrumentedSet
```

内置集合的受检测版本。

**类签名**

类 `sqlalchemy.orm.collections.InstrumentedSet` (`builtins.set`, `typing.Generic`)

```py
function sqlalchemy.orm.collections.prepare_instrumentation(factory: Type[Collection[Any]] | Callable[[], _AdaptedCollectionProtocol]) → Callable[[], _AdaptedCollectionProtocol]
```

准备一个可调用对象，以便将来用作集合类工厂。

给定一个集合类工厂（类型或无参数可调用对象），返回另一个工厂，当调用时将产生兼容的实例。

此函数负责将 collection_class=list 转换为 collection_class=InstrumentedList 的运行时行为。

## 自定义集合访问

映射一对多或多对多的关系会导致通过父实例上的属性访问的值集合。这两种常见的集合类型是 `list` 和 `set`，在使用 `Mapped` 的 声明 映射中，通过在 `Mapped` 容器内使用集合类型来建立，如下面 `Parent.children` 集合中所示，其中使用了 `list`：

```py
from sqlalchemy import ForeignKey

from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

class Parent(Base):
    __tablename__ = "parent"

    parent_id: Mapped[int] = mapped_column(primary_key=True)

    # use a list
    children: Mapped[List["Child"]] = relationship()

class Child(Base):
    __tablename__ = "child"

    child_id: Mapped[int] = mapped_column(primary_key=True)
    parent_id: Mapped[int] = mapped_column(ForeignKey("parent.id"))
```

或者对于 `set`，在同样的 `Parent.children` 集合中示例：

```py
from typing import Set
from sqlalchemy import ForeignKey

from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

class Parent(Base):
    __tablename__ = "parent"

    parent_id: Mapped[int] = mapped_column(primary_key=True)

    # use a set
    children: Mapped[Set["Child"]] = relationship()

class Child(Base):
    __tablename__ = "child"

    child_id: Mapped[int] = mapped_column(primary_key=True)
    parent_id: Mapped[int] = mapped_column(ForeignKey("parent.id"))
```

注意

如果使用 Python 3.7 或 3.8，集合的注释需要使用 `typing.List` 或 `typing.Set`，例如 `Mapped[List["Child"]]` 或 `Mapped[Set["Child"]]`；在这些 Python 版本中，`list` 和 `set` 内置的 Python 不支持泛型注释，例如：

```py
from typing import List

class Parent(Base):
    __tablename__ = "parent"

    parent_id: Mapped[int] = mapped_column(primary_key=True)

    # use a List, Python 3.8 and earlier
    children: Mapped[List["Child"]] = relationship()
```

在不使用 `Mapped` 注解的映射中，比如在使用 命令式映射 或未类型化的 Python 代码时，以及在一些特殊情况下，总是可以直接指定 `relationship()` 的集合类，使用 `relationship.collection_class` 参数：

```py
# non-annotated mapping

class Parent(Base):
    __tablename__ = "parent"

    parent_id = mapped_column(Integer, primary_key=True)

    children = relationship("Child", collection_class=set)

class Child(Base):
    __tablename__ = "child"

    child_id = mapped_column(Integer, primary_key=True)
    parent_id = mapped_column(ForeignKey("parent.id"))
```

在缺少 `relationship.collection_class` 或 `Mapped` 的情况下，默认的集合类型是 `list`。

除了内置的 `list` 和 `set` 外，还支持两种字典的变体，下面在 字典集合 中描述。还支持将任何任意可变序列类型设置为目标集合，只需进行一些额外的配置步骤；这在 自定义集合实现 部分有描述。

### 字典集合

使用字典作为集合时需要一些额外的细节。这是因为对象总是以列表形式从数据库加载的，必须提供一种键生成策略以正确地填充字典。`attribute_keyed_dict()` 函数是实现简单字典集合的最常见方式。它生成一个字典类，将映射类的特定属性应用为键。在下面的示例中，我们映射了一个包含以 `Note.keyword` 属性为键的 `Note` 项字典的 `Item` 类。当使用 `attribute_keyed_dict()` 时，可能会使用 `Mapped` 注释使用 `KeyFuncDict` 或普通的 `dict`，如下例所示。但是，在这种情况下，必须使用 `relationship.collection_class` 参数，以便适当地参数化 `attribute_keyed_dict()`：

```py
from typing import Dict
from typing import Optional

from sqlalchemy import ForeignKey
from sqlalchemy.orm import attribute_keyed_dict
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

class Item(Base):
    __tablename__ = "item"

    id: Mapped[int] = mapped_column(primary_key=True)

    notes: Mapped[Dict[str, "Note"]] = relationship(
        collection_class=attribute_keyed_dict("keyword"),
        cascade="all, delete-orphan",
    )

class Note(Base):
    __tablename__ = "note"

    id: Mapped[int] = mapped_column(primary_key=True)
    item_id: Mapped[int] = mapped_column(ForeignKey("item.id"))
    keyword: Mapped[str]
    text: Mapped[Optional[str]]

    def __init__(self, keyword: str, text: str):
        self.keyword = keyword
        self.text = text
```

`Item.notes` 现在是一个字典：

```py
>>> item = Item()
>>> item.notes["a"] = Note("a", "atext")
>>> item.notes.items()
{'a': <__main__.Note object at 0x2eaaf0>}
```

`attribute_keyed_dict()` 会确保每个 `Note` 的 `.keyword` 属性与字典中的键一致。例如，当分配给 `Item.notes` 时，我们提供的字典键必须与实际 `Note` 对象的键匹配：

```py
item = Item()
item.notes = {
    "a": Note("a", "atext"),
    "b": Note("b", "btext"),
}
```

`attribute_keyed_dict()` 用作键的属性根本不需要被映射！使用普通的 Python `@property` 允许几乎任何关于对象的细节或组合细节被用作键，就像下面我们将其建立为 `Note.keyword` 和 `Note.text` 字段的前十个字母的元组时那样：

```py
class Item(Base):
    __tablename__ = "item"

    id: Mapped[int] = mapped_column(primary_key=True)

    notes: Mapped[Dict[str, "Note"]] = relationship(
        collection_class=attribute_keyed_dict("note_key"),
        back_populates="item",
        cascade="all, delete-orphan",
    )

class Note(Base):
    __tablename__ = "note"

    id: Mapped[int] = mapped_column(primary_key=True)
    item_id: Mapped[int] = mapped_column(ForeignKey("item.id"))
    keyword: Mapped[str]
    text: Mapped[str]

    item: Mapped["Item"] = relationship()

    @property
    def note_key(self):
        return (self.keyword, self.text[0:10])

    def __init__(self, keyword: str, text: str):
        self.keyword = keyword
        self.text = text
```

在上面的示例中，我们添加了一个 `Note.item` 关系，具有双向的 `relationship.back_populates` 配置。将 `Note` 分配给这个反向关系时，`Note` 被添加到 `Item.notes` 字典中，键会自动为我们生成：

```py
>>> item = Item()
>>> n1 = Note("a", "atext")
>>> n1.item = item
>>> item.notes
{('a', 'atext'): <__main__.Note object at 0x2eaaf0>}
```

其他内置字典类型包括 `column_keyed_dict()`，它几乎和 `attribute_keyed_dict()` 类似，除了直接给出 `Column` 对象之外：

```py
from sqlalchemy.orm import column_keyed_dict

class Item(Base):
    __tablename__ = "item"

    id: Mapped[int] = mapped_column(primary_key=True)

    notes: Mapped[Dict[str, "Note"]] = relationship(
        collection_class=column_keyed_dict(Note.__table__.c.keyword),
        cascade="all, delete-orphan",
    )
```

以及传递任何可调用函数的 `mapped_collection()`。请注意，通常更容易使用 `attribute_keyed_dict()` 以及前面提到的 `@property`：

```py
from sqlalchemy.orm import mapped_collection

class Item(Base):
    __tablename__ = "item"

    id: Mapped[int] = mapped_column(primary_key=True)

    notes: Mapped[Dict[str, "Note"]] = relationship(
        collection_class=mapped_collection(lambda note: note.text[0:10]),
        cascade="all, delete-orphan",
    )
```

字典映射通常与“Association Proxy”扩展结合使用以生成简化的字典视图。有关示例，请参见代理到基于字典的集合 和 复合关联代理。

#### 处理键的突变和字典集合的反向填充

当使用 `attribute_keyed_dict()` 时，字典的“键”来自目标对象上的属性。**不会跟踪此键的更改**。这意味着必须在首次使用时分配键，如果键更改，则集合将不会发生变化。可能出现问题的典型示例是依赖 backrefs 填充属性映射集合。给定以下情况：

```py
class A(Base):
    __tablename__ = "a"

    id: Mapped[int] = mapped_column(primary_key=True)

    bs: Mapped[Dict[str, "B"]] = relationship(
        collection_class=attribute_keyed_dict("data"),
        back_populates="a",
    )

class B(Base):
    __tablename__ = "b"

    id: Mapped[int] = mapped_column(primary_key=True)
    a_id: Mapped[int] = mapped_column(ForeignKey("a.id"))
    data: Mapped[str]

    a: Mapped["A"] = relationship(back_populates="bs")
```

如果我们创建一个引用特定 `A()` 的 `B()`，那么 back populates 将把 `B()` 添加到 `A.bs` 集合中，但是如果 `B.data` 的值尚未设置，则键将为 `None`：

```py
>>> a1 = A()
>>> b1 = B(a=a1)
>>> a1.bs
{None: <test3.B object at 0x7f7b1023ef70>}
```

事后设置 `b1.data` 并不会更新集合：

```py
>>> b1.data = "the key"
>>> a1.bs
{None: <test3.B object at 0x7f7b1023ef70>}
```

如果尝试在构造函数中设置 `B()`，也会出现这种情况。参数的顺序改变了结果：

```py
>>> B(a=a1, data="the key")
<test3.B object at 0x7f7b10114280>
>>> a1.bs
{None: <test3.B object at 0x7f7b10114280>}
```

对比：

```py
>>> B(data="the key", a=a1)
<test3.B object at 0x7f7b10114340>
>>> a1.bs
{'the key': <test3.B object at 0x7f7b10114340>}
```

如果 backrefs 被这样使用，请确保使用 `__init__` 方法按正确顺序填充属性。

事件处理程序可以用来跟踪集合中的更改，例如以下示例：

```py
from sqlalchemy import event
from sqlalchemy.orm import attributes

@event.listens_for(B.data, "set")
def set_item(obj, value, previous, initiator):
    if obj.a is not None:
        previous = None if previous == attributes.NO_VALUE else previous
        obj.a.bs[value] = obj
        obj.a.bs.pop(previous)
```  ### 字典集合

使用字典作为集合时需要一些额外的细节。这是因为对象总是以列表形式从数据库加载，必须提供一种键生成策略来正确填充字典。`attribute_keyed_dict()` 函数是实现简单字典集合的最常见方式。它生成一个字典类，该类将应用映射类的特定属性作为键。下面我们映射了一个包含以`Note.keyword`属性为键的`Note`项目字典的`Item`类。在使用`attribute_keyed_dict()`时，可以使用`Mapped`注释使用`KeyFuncDict`或普通的`dict`，如下例所示。然而，在这种情况下，必须使用`relationship.collection_class`参数，以便适当地对`attribute_keyed_dict()`进行参数化：

```py
from typing import Dict
from typing import Optional

from sqlalchemy import ForeignKey
from sqlalchemy.orm import attribute_keyed_dict
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

class Item(Base):
    __tablename__ = "item"

    id: Mapped[int] = mapped_column(primary_key=True)

    notes: Mapped[Dict[str, "Note"]] = relationship(
        collection_class=attribute_keyed_dict("keyword"),
        cascade="all, delete-orphan",
    )

class Note(Base):
    __tablename__ = "note"

    id: Mapped[int] = mapped_column(primary_key=True)
    item_id: Mapped[int] = mapped_column(ForeignKey("item.id"))
    keyword: Mapped[str]
    text: Mapped[Optional[str]]

    def __init__(self, keyword: str, text: str):
        self.keyword = keyword
        self.text = text
```

然后，`Item.notes`是一个字典：

```py
>>> item = Item()
>>> item.notes["a"] = Note("a", "atext")
>>> item.notes.items()
{'a': <__main__.Note object at 0x2eaaf0>}
```

`attribute_keyed_dict()` 将确保每个 `Note` 的 `.keyword` 属性与字典中的键相符合。例如，当分配给 `Item.notes` 时，我们提供的字典键必须与实际 `Note` 对象的键相匹配：

```py
item = Item()
item.notes = {
    "a": Note("a", "atext"),
    "b": Note("b", "btext"),
}
```

`attribute_keyed_dict()`用作键的属性根本不需要被映射！使用普通的 Python `@property` 允许使用对象的几乎任何细节或细节组合作为键，如下所示，当我们将其建立为 `Note.keyword` 的元组和 `Note.text` 字段的前十个字母时：

```py
class Item(Base):
    __tablename__ = "item"

    id: Mapped[int] = mapped_column(primary_key=True)

    notes: Mapped[Dict[str, "Note"]] = relationship(
        collection_class=attribute_keyed_dict("note_key"),
        back_populates="item",
        cascade="all, delete-orphan",
    )

class Note(Base):
    __tablename__ = "note"

    id: Mapped[int] = mapped_column(primary_key=True)
    item_id: Mapped[int] = mapped_column(ForeignKey("item.id"))
    keyword: Mapped[str]
    text: Mapped[str]

    item: Mapped["Item"] = relationship()

    @property
    def note_key(self):
        return (self.keyword, self.text[0:10])

    def __init__(self, keyword: str, text: str):
        self.keyword = keyword
        self.text = text
```

上面我们添加了一个具有双向 `relationship.back_populates` 配置的 `Note.item` 关系。给这个反向关系赋值时，`Note` 被添加到 `Item.notes` 字典中，并且键会自动生成：

```py
>>> item = Item()
>>> n1 = Note("a", "atext")
>>> n1.item = item
>>> item.notes
{('a', 'atext'): <__main__.Note object at 0x2eaaf0>}
```

其他内置的字典类型包括 `column_keyed_dict()`，它几乎类似于 `attribute_keyed_dict()`，只是直接给出了 `Column` 对象：

```py
from sqlalchemy.orm import column_keyed_dict

class Item(Base):
    __tablename__ = "item"

    id: Mapped[int] = mapped_column(primary_key=True)

    notes: Mapped[Dict[str, "Note"]] = relationship(
        collection_class=column_keyed_dict(Note.__table__.c.keyword),
        cascade="all, delete-orphan",
    )
```

以及`mapped_collection()`，它将传递任何可调用函数。请注意，通常最好与前面提到的`@property`一起使用`attribute_keyed_dict()`更容易：

```py
from sqlalchemy.orm import mapped_collection

class Item(Base):
    __tablename__ = "item"

    id: Mapped[int] = mapped_column(primary_key=True)

    notes: Mapped[Dict[str, "Note"]] = relationship(
        collection_class=mapped_collection(lambda note: note.text[0:10]),
        cascade="all, delete-orphan",
    )
```

字典映射经常与“关联代理”扩展组合以产生流畅的字典视图。参见基于字典的集合的代理和复合关联代理以获取示例。

#### 处理键变化和字典集合的反向填充

当使用`attribute_keyed_dict()`时，字典的“键”来自目标对象上的属性。**对此键的更改不会被跟踪**。这意味着键必须在首次使用时被分配，并且如果键发生更改，则集合将不会发生变化。一个典型的例子是当依赖反向引用来填充属性映射集合时可能会出现问题。给定以下内容：

```py
class A(Base):
    __tablename__ = "a"

    id: Mapped[int] = mapped_column(primary_key=True)

    bs: Mapped[Dict[str, "B"]] = relationship(
        collection_class=attribute_keyed_dict("data"),
        back_populates="a",
    )

class B(Base):
    __tablename__ = "b"

    id: Mapped[int] = mapped_column(primary_key=True)
    a_id: Mapped[int] = mapped_column(ForeignKey("a.id"))
    data: Mapped[str]

    a: Mapped["A"] = relationship(back_populates="bs")
```

如果我们创建引用特定`A()`的`B()`，那么反向填充将把`B()`添加到`A.bs`集合中，但是如果`B.data`的值尚未设置，则键将为`None`：

```py
>>> a1 = A()
>>> b1 = B(a=a1)
>>> a1.bs
{None: <test3.B object at 0x7f7b1023ef70>}
```

在事后设置`b1.data`不会更新集合：

```py
>>> b1.data = "the key"
>>> a1.bs
{None: <test3.B object at 0x7f7b1023ef70>}
```

如果尝试在构造函数中设置 `B()`，也可以看到这一点。参数的顺序会改变结果：

```py
>>> B(a=a1, data="the key")
<test3.B object at 0x7f7b10114280>
>>> a1.bs
{None: <test3.B object at 0x7f7b10114280>}
```

对比：

```py
>>> B(data="the key", a=a1)
<test3.B object at 0x7f7b10114340>
>>> a1.bs
{'the key': <test3.B object at 0x7f7b10114340>}
```

如果以这种方式使用反向引用，请确保使用`__init__`方法以正确的顺序填充属性。

还可以使用以下事件处理程序来跟踪集合中的更改：

```py
from sqlalchemy import event
from sqlalchemy.orm import attributes

@event.listens_for(B.data, "set")
def set_item(obj, value, previous, initiator):
    if obj.a is not None:
        previous = None if previous == attributes.NO_VALUE else previous
        obj.a.bs[value] = obj
        obj.a.bs.pop(previous)
``` #### 处理键变化和字典集合的反向填充

当使用`attribute_keyed_dict()`时，字典的“键”来自目标对象上的属性。**对此键的更改不会被跟踪**。这意味着键必须在首次使用时被分配，并且如果键发生更改，则集合将不会发生变化。一个典型的例子是当依赖反向引用来填充属性映射集合时可能会出现问题。给定以下内容：

```py
class A(Base):
    __tablename__ = "a"

    id: Mapped[int] = mapped_column(primary_key=True)

    bs: Mapped[Dict[str, "B"]] = relationship(
        collection_class=attribute_keyed_dict("data"),
        back_populates="a",
    )

class B(Base):
    __tablename__ = "b"

    id: Mapped[int] = mapped_column(primary_key=True)
    a_id: Mapped[int] = mapped_column(ForeignKey("a.id"))
    data: Mapped[str]

    a: Mapped["A"] = relationship(back_populates="bs")
```

如果我们创建引用特定`A()`的`B()`，那么反向填充将把`B()`添加到`A.bs`集合中，但是如果`B.data`的值尚未设置，则键将为`None`：

```py
>>> a1 = A()
>>> b1 = B(a=a1)
>>> a1.bs
{None: <test3.B object at 0x7f7b1023ef70>}
```

在事后设置`b1.data`不会更新集合：

```py
>>> b1.data = "the key"
>>> a1.bs
{None: <test3.B object at 0x7f7b1023ef70>}
```

如果尝试在构造函数中设置 `B()`，也可以看到这一点。参数的顺序会改变结果：

```py
>>> B(a=a1, data="the key")
<test3.B object at 0x7f7b10114280>
>>> a1.bs
{None: <test3.B object at 0x7f7b10114280>}
```

对比：

```py
>>> B(data="the key", a=a1)
<test3.B object at 0x7f7b10114340>
>>> a1.bs
{'the key': <test3.B object at 0x7f7b10114340>}
```

如果以这种方式使用反向引用，请确保使用`__init__`方法以正确的顺序填充属性。

还可以使用以下事件处理程序来跟踪集合中的更改：

```py
from sqlalchemy import event
from sqlalchemy.orm import attributes

@event.listens_for(B.data, "set")
def set_item(obj, value, previous, initiator):
    if obj.a is not None:
        previous = None if previous == attributes.NO_VALUE else previous
        obj.a.bs[value] = obj
        obj.a.bs.pop(previous)
```

## 自定义集合实现

您也可以为集合使用自己的类型。在简单情况下，继承自 `list` 或 `set`，添加自定义行为即可。在其他情况下，需要特殊的装饰器来告诉 SQLAlchemy 关于集合操作的更多细节。

SQLAlchemy 中的集合是**透明的** *instrumented*。仪器化意味着对集合的常规操作将被跟踪，并在刷新时写入数据库中。此外，集合操作可以触发 *事件*，指示必须进行某些次要操作。次要操作的示例包括在父项的`Session`（即 `save-update` 级联）中保存子项，以及同步双向关系的状态（即 `backref()`）。

集合包理解列表、集合和字典的基本接口，并将自动将仪器应用于这些内置类型及其子类。通过鸭子类型检测实现基本集合接口的对象派生类型，以进行仪器化：

```py
class ListLike:
    def __init__(self):
        self.data = []

    def append(self, item):
        self.data.append(item)

    def remove(self, item):
        self.data.remove(item)

    def extend(self, items):
        self.data.extend(items)

    def __iter__(self):
        return iter(self.data)

    def foo(self):
        return "foo"
```

`append`、`remove` 和 `extend` 是 `list` 的已知成员，并将自动进行仪器化。`__iter__` 不是一个修改器方法，不会被仪器化，`foo` 也不会被仪器化。

当然，鸭子类型（即猜测）并不是百分之百可靠的，所以您可以通过提供一个 `__emulates__` 类属性来明确您正在实现的接口：

```py
class SetLike:
    __emulates__ = set

    def __init__(self):
        self.data = set()

    def append(self, item):
        self.data.add(item)

    def remove(self, item):
        self.data.remove(item)

    def __iter__(self):
        return iter(self.data)
```

此类似于 Python `list`（即 “类似列表”），因为它有一个 `append` 方法，但 `__emulates__` 属性强制将其视为 `set`。 `remove` 已知是集合接口的一部分，并将被仪器化。

但是，此类暂时无法正常工作：需要一点粘合剂来使其适应 SQLAlchemy 的使用。ORM 需要知道用于附加、删除和迭代集合成员的方法。当使用类似于 `list` 或 `set` 的类型时，当存在时，适当的方法是众所周知的并且会自动使用。然而，上面的类，虽然只大致类似于 `set`，但并未提供预期的 `add` 方法，因此我们必须告诉 ORM 替代 `add` 方法的方法，在这种情况下使用装饰器 `@collection.appender`；这在下一节中进行了说明。

### 通过装饰器注释自定义集合

当您的类不完全符合其容器类型的常规接口时，或者您希望以其他方式使用不同的方法来完成工作时，可以使用装饰器标记 ORM 需要管理集合的各个方法。

```py
from sqlalchemy.orm.collections import collection

class SetLike:
    __emulates__ = set

    def __init__(self):
        self.data = set()

    @collection.appender
    def append(self, item):
        self.data.add(item)

    def remove(self, item):
        self.data.remove(item)

    def __iter__(self):
        return iter(self.data)
```

而这就是完成示例所需的全部。SQLAlchemy 将通过`append`方法添加实例。`remove`和`__iter__`是集合的默认方法，将用于移除和迭代。默认方法也可以更改：

```py
from sqlalchemy.orm.collections import collection

class MyList(list):
    @collection.remover
    def zark(self, item):
        # do something special...
        ...

    @collection.iterator
    def hey_use_this_instead_for_iteration(self): ...
```

完全不需要“类似于列表”或“类似于集合”。集合类可以是任何形状，只要它们有被标记为 SQLAlchemy 使用的 append、remove 和 iterate 接口。append 和 remove 方法将以映射实体作为唯一参数调用，迭代器方法将以无参数调用，并且必须返回迭代器。

### 自定义基于字典的集合

`KeyFuncDict`类可以用作自定义类型的基类，也可以作为混合类快速为其他类添加`dict`集合支持。它使用一个键函数来委托`__setitem__`和`__delitem__`：

```py
from sqlalchemy.orm.collections import KeyFuncDict

class MyNodeMap(KeyFuncDict):
  """Holds 'Node' objects, keyed by the 'name' attribute."""

    def __init__(self, *args, **kw):
        super().__init__(keyfunc=lambda node: node.name)
        dict.__init__(self, *args, **kw)
```

当子类化`KeyFuncDict`时，如果用户定义了`__setitem__()`或`__delitem__()`的版本，并且它们调用了相同的方法`KeyFuncDict`上的方法，则应使用`collection.internally_instrumented()`进行装饰，**如果**它们调用了相同的方法`KeyFuncDict`上的方法。这是因为`KeyFuncDict`上的方法已经被仪器化了 - 从已经仪器化的调用中调用它们可能会导致事件被重复触发，或者不适当地触发，在极少数情况下会导致内部状态损坏：

```py
from sqlalchemy.orm.collections import KeyFuncDict, collection

class MyKeyFuncDict(KeyFuncDict):
  """Use @internally_instrumented when your methods
 call down to already-instrumented methods.

 """

    @collection.internally_instrumented
    def __setitem__(self, key, value, _sa_initiator=None):
        # do something with key, value
        super(MyKeyFuncDict, self).__setitem__(key, value, _sa_initiator)

    @collection.internally_instrumented
    def __delitem__(self, key, _sa_initiator=None):
        # do something with key
        super(MyKeyFuncDict, self).__delitem__(key, _sa_initiator)
```

ORM 与`dict`接口的理解方式与列表和集合一样，并且如果选择对`dict`进行子类化或在鸭式类型的类中提供类似于字典的集合行为，则会自动对所有“类似于字典”的方法进行仪器化。但是，必须装饰追加和删除方法 - 基本字典接口中没有兼容的方法供 SQLAlchemy 默认使用。迭代将通过`values()`进行，除非另有装饰。

### 仪器化和自定义类型

许多自定义类型和现有库类可以直接使用作为实体集合类型，无需额外操作。但是，重要的是要注意，仪器化过程将修改类型，自动在方法周围添加装饰器。

装饰很轻量级，在关系之外不起作用，但是当在其他地方触发时会增加不必要的开销。当将库类用作集合时，最好使用“微不足道的子类”技巧将装饰限制为关系中的使用。例如：

```py
class MyAwesomeList(some.great.library.AwesomeList):
    pass

# ... relationship(..., collection_class=MyAwesomeList)
```

ORM 使用这种方法进行内置，当直接使用`list`、`set`或`dict`时，会悄悄地替换为一个微不足道的子类。

### 通过装饰器注释自定义集合

可以使用装饰器标记 ORM 需要管理集合的各个方法。当您的类不完全符合其容器类型的常规接口时，或者当您希望以不同的方法完成工作时，请使用它们。

```py
from sqlalchemy.orm.collections import collection

class SetLike:
    __emulates__ = set

    def __init__(self):
        self.data = set()

    @collection.appender
    def append(self, item):
        self.data.add(item)

    def remove(self, item):
        self.data.remove(item)

    def __iter__(self):
        return iter(self.data)
```

这就是完成示例所需的全部内容。SQLAlchemy 将通过`append`方法添加实例。`remove`和`__iter__`是集合的默认方法，将用于删除和迭代。默认方法也可以更改：

```py
from sqlalchemy.orm.collections import collection

class MyList(list):
    @collection.remover
    def zark(self, item):
        # do something special...
        ...

    @collection.iterator
    def hey_use_this_instead_for_iteration(self): ...
```

完全不需要“类似于列表”或“类似于集合”。集合类可以是任何形状，只要它们具有标记为 SQLAlchemy 使用的追加、移除和迭代接口即可。追加和移除方法将以映射实体作为单个参数调用，并且迭代方法将不带参数调用，并且必须返回迭代器。

### 自定义基于字典的集合

`KeyFuncDict` 类可以作为自定义类型的基类，也可以作为混合类快速将`dict`集合支持添加到其他类中。它使用键函数来委托`__setitem__`和`__delitem__`：

```py
from sqlalchemy.orm.collections import KeyFuncDict

class MyNodeMap(KeyFuncDict):
  """Holds 'Node' objects, keyed by the 'name' attribute."""

    def __init__(self, *args, **kw):
        super().__init__(keyfunc=lambda node: node.name)
        dict.__init__(self, *args, **kw)
```

当子类化`KeyFuncDict`时，用户定义的`__setitem__()`或`__delitem__()`版本应该用`collection.internally_instrumented()`进行修饰，**如果**它们调用同样的方法。这是因为`KeyFuncDict`上的方法已经被仪器化-在已经仪器化的调用中调用它们可能会导致事件重复触发，或者在罕见情况下导致内部状态损坏：

```py
from sqlalchemy.orm.collections import KeyFuncDict, collection

class MyKeyFuncDict(KeyFuncDict):
  """Use @internally_instrumented when your methods
 call down to already-instrumented methods.

 """

    @collection.internally_instrumented
    def __setitem__(self, key, value, _sa_initiator=None):
        # do something with key, value
        super(MyKeyFuncDict, self).__setitem__(key, value, _sa_initiator)

    @collection.internally_instrumented
    def __delitem__(self, key, _sa_initiator=None):
        # do something with key
        super(MyKeyFuncDict, self).__delitem__(key, _sa_initiator)
```

ORM 理解`dict`接口就像理解列表和集合一样，并且如果您选择子类化`dict`或在鸭子类型的类中提供类似于 dict 的集合行为，则会自动仪器化所有“类似于 dict”的方法。但是，您必须修饰追加和移除方法-默认情况下，基本字典接口中没有兼容的方法供 SQLAlchemy 使用。迭代将通过`values()`进行，除非另有修饰。

### 仪器化和自定义类型

许多自定义类型和现有的库类可以直接用作实体集合类型，无需进一步操作。但是，需要注意的是，仪器化过程将修改类型，自动在方法周围添加修饰符。

装饰是轻量级的，并且在关系之外不起作用，但是当在其他地方触发时它们会增加不必要的开销。当将库类用作集合时，使用“trivial subclass”技巧将装饰限制为仅在关系中使用的情况可能是一个好习惯。例如：

```py
class MyAwesomeList(some.great.library.AwesomeList):
    pass

# ... relationship(..., collection_class=MyAwesomeList)
```

ORM 使用此方法处理内置功能，当直接使用 `list`、`set` 或 `dict` 时，会静默地替换为一个微不足道的子类。

## 集合 API

| 对象名称 | 描述 |
| --- | --- |
| attribute_keyed_dict(attr_name, *, [ignore_unpopulated_attribute]) | 基于属性的键的字典类型集合。 |
| attribute_mapped_collection | 基于属性的键的字典类型集合。 |
| column_keyed_dict(mapping_spec, *, [ignore_unpopulated_attribute]) | 基于列的键的字典类型集合。 |
| column_mapped_collection | 基于列的键的字典类型集合。 |
| keyfunc_mapping(keyfunc, *, [ignore_unpopulated_attribute]) | 具有任意键的基于字典的集合类型。 |
| KeyFuncDict | 用于 ORM 映射字典类的基础。 |
| mapped_collection | 具有任意键的基于字典的集合类型。 |
| MappedCollection | 用于 ORM 映射字典类的基础。 |

```py
function sqlalchemy.orm.attribute_keyed_dict(attr_name: str, *, ignore_unpopulated_attribute: bool = False) → Type[KeyFuncDict[Any, Any]]
```

基于属性的键的字典类型集合。

2.0 版本更改：将`attribute_mapped_collection`重命名为`attribute_keyed_dict()`。

返回一个`KeyFuncDict`工厂，它将根据要添加到字典中的 ORM 映射实例上的特定命名属性的值产生新的字典键。

注意

目标属性的值必须在对象添加到字典集合时被赋值。此外，**不会跟踪**关键属性的更改，这意味着字典中的键不会自动与目标对象本身的键值同步。有关更多详细信息，请参阅处理键变化和反向填充字典集合。

另请参阅

字典集合 - 使用背景

参数：

+   `attr_name` – 映射类上的 ORM 映射属性的字符串名称，该属性的值将作为该实例的新字典条目的键值。

+   `ignore_unpopulated_attribute` –

    如果为 True，并且对象上的目标属性根本未填充，则该操作将被静默跳过。默认情况下，将引发错误。

    新版 2.0 中：如果确定用于字典键的属性从未填充任何值，则默认情况下会引发错误。`attribute_keyed_dict.ignore_unpopulated_attribute` 参数可以设置，表示应忽略此条件，并且静默跳过追加操作。这与 1.x 系列的行为形成对比，后者会错误地用任意键值`None`填充字典中的值。

```py
function sqlalchemy.orm.column_keyed_dict(mapping_spec: Type[_KT] | Callable[[_KT], _VT], *, ignore_unpopulated_attribute: bool = False) → Type[KeyFuncDict[_KT, _KT]]
```

基于列的键控字典集合类型。

2.0 中更改：将`column_mapped_collection` 更名为 `column_keyed_dict`。

返回一个`KeyFuncDict` 工厂，它将根据 ORM 映射实例上的特定`Column`-映射属性的值生成新的字典键，以添加到字典中。

注意

目标属性的值必须在将对象添加到字典集合时分配其值。此外，不跟踪键属性的更改，这意味着字典中的键不会自动与目标对象本身的键值同步。有关详细信息，请参阅处理键变化和向字典集合回填。

也可参阅

字典集合 - 使用背景

参数：

+   `mapping_spec` – 预期由目标映射器映射到映射类上的特定属性的`Column`对象，该属性的值在特定实例上将用作该实例的新字典条目的键。

+   `ignore_unpopulated_attribute` –

    如果为 True，并且对象上由给定`Column`目标属性指示的映射属性根本未填充，则操作将被静默跳过。默认情况下，会引发错误。

    新版 2.0 中：如果确定用于字典键的属性从未填充任何值，则默认情况下会引发错误。`column_keyed_dict.ignore_unpopulated_attribute` 参数可以设置，表示应忽略此条件，并且静默跳过追加操作。这与 1.x 系列的行为形成对比，后者会错误地用任意键值`None`填充字典中的值。

```py
function sqlalchemy.orm.keyfunc_mapping(keyfunc: _F, *, ignore_unpopulated_attribute: bool = False) → Type[KeyFuncDict[_KT, Any]]
```

一个具有任意键的基于字典的集合类型。

从版本 2.0 开始更改：将 `mapped_collection` 重命名为 `keyfunc_mapping()`。

返回一个从 `keyfunc` 生成的键函数的 `KeyFuncDict` 工厂，`keyfunc` 是一个可调用对象，接受一个实体并返回一个键值。

注意

给定的 `keyfunc` 只在将目标对象添加到集合时调用一次。不跟踪对函数返回的有效值的更改。

另请参见

字典集合 - 使用背景

参数：

+   `keyfunc` – 应传递 ORM 映射实例的可调用对象，然后生成一个新的用于字典的键。如果返回的值是 `LoaderCallableStatus.NO_VALUE`，则会引发错误。

+   `ignore_unpopulated_attribute` –

    如果为 True，并且可调用对象对于特定实例返回 `LoaderCallableStatus.NO_VALUE`，则操作将被静默跳过。默认情况下，会引发错误。

    从版本 2.0 开始：如果用于字典键的可调用对象返回 `LoaderCallableStatus.NO_VALUE`，默认情况下会引发错误，这在 ORM 属性上下文中表示一个从未填充任何值的属性。可以设置 `mapped_collection.ignore_unpopulated_attribute` 参数，该参数将指示忽略此条件，并且追加操作将被静默跳过。这与 1.x 系列的行为形成对比，后者会错误地用任意的键值 `None` 填充字典中的值。

```py
sqlalchemy.orm.attribute_mapped_collection = <function attribute_keyed_dict>
```

一个基于字典的集合类型，具有基于属性的键。

从版本 2.0 开始更改：将 `attribute_mapped_collection` 重命名为 `attribute_keyed_dict()`。

返回一个根据要添加到字典中的 ORM 映射实例的特定命名属性的值生成新字典键的 `KeyFuncDict` 工厂。

注意

目标属性的值必须在将对象添加到字典集合时分配其值。此外，不跟踪键属性的更改，这意味着字典中的键不会自动与目标对象本身的键值同步。有关详细信息，请参见处理键变异和向字典集合反填充。

另请参见

字典集合 - 使用背景

参数：

+   `attr_name` – 映射类上的 ORM 映射属性的字符串名称，在特定实例上的值将用作该实例的新字典条目的键。

+   `ignore_unpopulated_attribute` –

    如果为 True，并且对象上的目标属性根本未填充，则操作将被静默跳过。默认情况下，将引发错误。

    新版本 2.0 中：如果确定用于字典键的属性从未填充任何值，则默认会引发错误。 `attribute_keyed_dict.ignore_unpopulated_attribute` 参数可以设置，以指示应忽略此条件，并且附加操作将被静默跳过。这与 1.x 系列的行为相反，后者会错误地使用任意键值为`None`填充字典中的值。

```py
sqlalchemy.orm.column_mapped_collection = <function column_keyed_dict>
```

基于字典的集合类型，以列为键。

2.0 版更改：将`column_mapped_collection`重命名为`column_keyed_dict`。

返回一个`KeyFuncDict`工厂，它将根据 ORM 映射实例上的特定`Column`属性的值生成新的字典键，以添加到字典中。

注意

目标属性的值必须在将对象添加到字典集合时分配其值。此外，不跟踪键属性的更改，这意味着字典中的键不会自动与目标对象本身的键值同步。有关详细信息，请参见处理键变异和向字典集合反填充。

另请参见

字典集合 - 使用背景

参数：

+   `mapping_spec` – 预期由目标映射器映射到映射类上的特定属性的`Column`对象，在特定实例上的值将用作该实例的新字典条目的键。

+   `ignore_unpopulated_attribute` –

    如果为 True，并且对象上的给定`Column`目标属性指示的映射属性根本未填充，则操作将被静默跳过。默认情况下，会引发错误。

    版本 2.0 中的新功能：如果确定用于字典键的属性从未填充任何值，则默认情况下会引发错误。可以设置`column_keyed_dict.ignore_unpopulated_attribute`参数，该参数将指示忽略此条件，并静默跳过追加操作。这与 1.x 系列的行为相反，后者将错误地用任意键值`None`填充字典中的值。

```py
sqlalchemy.orm.mapped_collection = <function keyfunc_mapping>
```

具有任意键的基于字典的集合类型。

自版本 2.0 起更改：将`mapped_collection`重命名为`keyfunc_mapping()`。

返回一个由 keyfunc 生成的键函数的`KeyFuncDict`工厂，keyfunc 是一个可调用的函数，它接受一个实体并返回一个键值。

注意

给定的 keyfunc 仅在将目标对象添加到集合时调用一次。不跟踪函数返回的有效值的更改。

另请参阅

字典集合 - 使用背景

参数：

+   `keyfunc` – 一个可调用的函数，将传递 ORM 映射实例，然后生成一个新的键用于字典中。如果返回的值为`LoaderCallableStatus.NO_VALUE`，则会引发错误。

+   `ignore_unpopulated_attribute` –

    如果为 True，并且可调用函数对于特定实例返回`LoaderCallableStatus.NO_VALUE`，则操作将被静默跳过。默认情况下，会引发错误。

    新版本 2.0 中，默认情况下，如果用于字典键的可调用函数返回`LoaderCallableStatus.NO_VALUE`，则会引发错误。在 ORM 属性上下文中，这表示从未填充任何值的属性。可以设置`mapped_collection.ignore_unpopulated_attribute`参数，该参数将表示应忽略此条件，并且附加操作会被静默跳过。这与 1.x 系列的行为形成对比，后者会错误地使用任意键值`None`填充字典中的值。

```py
class sqlalchemy.orm.KeyFuncDict
```

ORM 映射字典类的基类。

通过向 SQLAlchemy ORM 集合类添加所需的附加方法来扩展`dict`类型。最直接使用`attribute_keyed_dict()`或`column_keyed_dict()`类工厂来使用`KeyFuncDict`。`KeyFuncDict`也可以用作用户定义的自定义字典类的基类。

2.0 中的变更：将`MappedCollection`重命名为`KeyFuncDict`。

另请参见

`attribute_keyed_dict()`

`column_keyed_dict()`

字典集合

自定义集合实现

**成员**

__init__(), clear(), pop(), popitem(), remove(), set(), setdefault(), update()

**类签名**

class `sqlalchemy.orm.KeyFuncDict` (`builtins.dict`, `typing.Generic`)

```py
method __init__(keyfunc: _F, *dict_args: Any, ignore_unpopulated_attribute: bool = False) → None
```

使用 keyfunc 提供的键制作新集合。

keyfunc 可以是任何接受对象并返回对象以用作字典键的可调用函数。

每当 ORM 需要通过仅基于值的方式添加成员（例如从数据库加载实例时）或删除成员时，都会调用 keyfunc。通常的字典键的注意事项也适用 - `keyfunc(object)` 应该在集合的生命周期内返回相同的输出。基于可变属性的键值可能导致集合中“丢失”的不可达实例。

```py
method clear() → None.  Remove all items from D.
```

```py
method pop(k[, d]) → v, remove specified key and return the corresponding value.
```

如果找不到键，则如果给定默认值，则返回默认值；否则，引发 KeyError。

```py
method popitem()
```

移除并返回一个 (key, value) 对作为 2-元组。

以 LIFO（后进先出）顺序返回对。如果字典为空，则引发 KeyError。

```py
method remove(value: _KT, _sa_initiator: AttributeEventToken | Literal[None, False] = None) → None
```

通过值删除项目，并查询键的键函数。

```py
method set(value: _KT, _sa_initiator: AttributeEventToken | Literal[None, False] = None) → None
```

通过值添加项目，并查询键的键函数。

```py
method setdefault(key, default=None)
```

插入具有默认值的键，如果键不在字典中。

如果键在字典中，则返回键的值，否则返回默认值。

```py
method update([E, ]**F) → None.  Update D from dict/iterable E and F.
```

如果 E 存在且具有 .keys() 方法，则执行以下操作：for k in E: D[k] = E[k] 如果 E 存在且缺少 .keys() 方法，则执行以下操作：for k, v in E: D[k] = v 在任何一种情况下，都会执行以下操作：for k in F: D[k] = F[k]

```py
sqlalchemy.orm.MappedCollection = <class 'sqlalchemy.orm.mapped_collection.KeyFuncDict'>
```

ORM 映射字典类的基础。

扩展了 `dict` 类型，以包含 SQLAlchemy ORM 集合类所需的附加方法。最直接使用 `attribute_keyed_dict()` 或 `column_keyed_dict()` 类工厂来使用 `KeyFuncDict`。`KeyFuncDict` 也可以作为用户定义的自定义字典类的基础。

在 2.0 版本中更改：将 `MappedCollection` 重命名为 `KeyFuncDict`。

另请参阅

`attribute_keyed_dict()`

`column_keyed_dict()`

字典集合

自定义集合实现

## 集合内部

| Object Name | 描述 |
| --- | --- |
| bulk_replace(values, existing_adapter, new_adapter[, initiator]) | 加载一个新的集合，并根据先前的相似成员资格触发事件。 |
| collection | 实体集合类的装饰器。 |
| collection_adapter | attrgetter(attr, …) –> attrgetter 对象 |
| CollectionAdapter | 在 ORM 和任意 Python 集合之间建立桥梁。 |
| InstrumentedDict | 内置字典的受控版本。 |
| InstrumentedList | 内置列表的受控版本。 |
| InstrumentedSet | 内置集合的受控版本。 |
| prepare_instrumentation(factory) | 准备一个可调用对象，以便将来用作集合类工厂。 |

```py
function sqlalchemy.orm.collections.bulk_replace(values, existing_adapter, new_adapter, initiator=None)
```

加载一个新的集合，并根据先前的相似成员资格触发事件。

将`values`中的实例追加到`new_adapter`中。对于`existing_adapter`中不存在的任何实例，将触发事件。`existing_adapter`中存在但在`values`中不存在的任何实例将触发删除事件。

参数：

+   `values` – 一个包含集合成员实例的可迭代对象

+   `existing_adapter` – 一个要替换的实例的`CollectionAdapter`

+   `new_adapter` – 一个空的`CollectionAdapter`，用于加载`values`

```py
class sqlalchemy.orm.collections.collection
```

用于实体集合类的装饰器。

装饰器分为两组：注解和拦截配方。

注解装饰器（appender、remover、iterator、converter、internally_instrumented）指示方法的目的并且不带参数。它们不带括号写成：

```py
@collection.appender
def append(self, append): ...
```

所有的装饰器都需要括号，即使没有参数：

**成员**

adds(), appender(), converter(), internally_instrumented(), iterator(), remover(), removes(), removes_return(), replaces()

```py
@collection.adds('entity')
def insert(self, position, entity): ...

@collection.removes_return()
def popitem(self): ...
```

```py
method static adds(arg)
```

将方法标记为向集合添加实体。

将“添加到集合”处理添加到方法中。装饰器参数指示哪个方法参数保存了与 SQLAlchemy 相关的值。参数可以按位置指定（即整数）或按名称指定：

```py
@collection.adds(1)
def push(self, item): ...

@collection.adds('entity')
def do_stuff(self, thing, entity=None): ...
```

```py
method static appender(fn)
```

将方法标记为集合追加器。

如果追加器方法调用时带有一个位置参数：要追加的值。如果尚未装饰，则该方法将自动装饰为‘adds(1)’：

```py
@collection.appender
def add(self, append): ...

# or, equivalently
@collection.appender
@collection.adds(1)
def add(self, append): ...

# for mapping type, an 'append' may kick out a previous value
# that occupies that slot.  consider d['a'] = 'foo'- any previous
# value in d['a'] is discarded.
@collection.appender
@collection.replaces(1)
def add(self, entity):
    key = some_key_func(entity)
    previous = None
    if key in self:
        previous = self[key]
    self[key] = entity
    return previous
```

如果要追加的值不允许在集合中，您可以引发异常。需要记住的是，追加器将针对数据库查询映射的每个对象调用。如果数据库包含违反集合语义的行，则您需要有创意地解决问题，因为通过集合访问将无法工作。

如果追加器方法在内部被检测到，您还必须接收关键字参数‘_sa_initiator’并确保将其传播到集合事件。

```py
method static converter(fn)
```

将方法标记为集合转换器。

自版本 1.3 弃用：`collection.converter()` 处理程序已弃用，并将在未来的版本中移除。请参考 `listen()` 函数结合 `bulk_replace` 监听器接口。

当集合被完全替换时，将调用此可选方法，如下所示：

```py
myobj.acollection = [newvalue1, newvalue2]
```

转换器方法将接收到要分配的对象，并应返回适用于 `appender` 方法使用的值的可迭代对象。转换器不得分配值或更改集合，它的唯一任务是将用户提供的值适应为 ORM 使用的值的可迭代对象。

默认的转换器实现将使用鸭子类型进行转换。类似字典的集合将被转换为字典值的可迭代对象，而其他类型将简单地进行迭代：

```py
@collection.converter
def convert(self, other): ...
```

如果对象的鸭子类型与此集合的类型不匹配，则会引发 TypeError。

如果您希望扩展可以批量分配的可能类型的范围或对即将分配的值进行验证，请提供此方法的实现。

```py
method static internally_instrumented(fn)
```

将该方法标记为受检测的。

此标记将阻止对该方法应用任何装饰。如果您正在 orchestrating 在基本的 SQLAlchemy 接口方法之一中调用 `collection_adapter()`，或者要防止自动 ABC 方法装饰器包装您的实现，请使用此标记：

```py
# normally an 'extend' method on a list-like class would be
# automatically intercepted and re-implemented in terms of
# SQLAlchemy events and append().  your implementation will
# never be called, unless:
@collection.internally_instrumented
def extend(self, items): ...
```

```py
method static iterator(fn)
```

将该方法标记为集合移除器。

迭代器方法无需参数调用。它应返回所有集合成员的迭代器：

```py
@collection.iterator
def __iter__(self): ...
```

```py
method static remover(fn)
```

将该方法标记为集合移除器。

移除器方法使用一个位置参数调用：要移除的值。如果尚未被其他装饰器装饰，该方法将自动使用 `removes_return()` 进行装饰：

```py
@collection.remover
def zap(self, entity): ...

# or, equivalently
@collection.remover
@collection.removes_return()
def zap(self, ): ...
```

如果要移除的值不存在于集合中，则可以引发异常或返回 None 以忽略错误。

如果移除方法在内部进行了检测，请确保也接收关键字参数 ‘_sa_initiator’ 并确保其在集合事件中传播。

```py
method static removes(arg)
```

将该方法标记为从集合中移除实体。

为方法添加“从集合中移除”的处理。修饰器参数指示哪个方法参数包含要移除的与 SQLAlchemy 相关的值。参数可以按位置指定（即整数），也可以按名称指定：

```py
@collection.removes(1)
def zap(self, item): ...
```

对于在调用时未知要移除的值的方法，请使用 collection.removes_return。

```py
method static removes_return()
```

将该方法标记为从集合中移除实体。

为方法添加“从集合中移除”的处理。如果没有被其他装饰器装饰，该方法的返回值（如果有）将被视为要移除的值。不会检查方法参数：

```py
@collection.removes_return()
def pop(self): ...
```

对于在调用时已知要移除的值的方法，请使用 collection.remove。

```py
method static replaces(arg)
```

标记该方法用于替换集合中的实体。

为方法添加“添加到集合”和“从集合中移除”处理。装饰器参数指示哪个方法参数保存了要添加的与 SQLAlchemy 相关的值，以及返回值（如果有）将被视为要移除的值。

参数可以通过位置（即整数）或名称指定：

```py
@collection.replaces(2)
def __setitem__(self, index, item): ...
```

```py
sqlalchemy.orm.collections.collection_adapter = operator.attrgetter('_sa_adapter')
```

attrgetter(attr, …) –> attrgetter 对象

返回一个可调用对象，从其操作数中提取给定的属性。在 f = attrgetter(‘name’) 后，调用 f(r) 返回 r.name。在 g = attrgetter(‘name’, ‘date’) 后，调用 g(r) 返回 (r.name, r.date)。在 h = attrgetter(‘name.first’, ‘name.last’) 后，调用 h(r) 返回 (r.name.first, r.name.last)。

```py
class sqlalchemy.orm.collections.CollectionAdapter
```

在 ORM 和任意 Python 集合之间建立桥梁。

将基本级别的集合操作（追加、删除、迭代）代理给底层的 Python 集合，并为进入或离开集合的实体发出添加/删除事件。

ORM 专门使用`CollectionAdapter` 与实体集合进行交互。

```py
class sqlalchemy.orm.collections.InstrumentedDict
```

内置字典的受检版本。

**类签名**

类`sqlalchemy.orm.collections.InstrumentedDict` (`builtins.dict`, `typing.Generic`)

```py
class sqlalchemy.orm.collections.InstrumentedList
```

内置列表的受检版本。

**类签名**

类`sqlalchemy.orm.collections.InstrumentedList` (`builtins.list`, `typing.Generic`)

```py
class sqlalchemy.orm.collections.InstrumentedSet
```

内置集合的受检版本。

**类签名**

类`sqlalchemy.orm.collections.InstrumentedSet` (`builtins.set`, `typing.Generic`)

```py
function sqlalchemy.orm.collections.prepare_instrumentation(factory: Type[Collection[Any]] | Callable[[], _AdaptedCollectionProtocol]) → Callable[[], _AdaptedCollectionProtocol]
```

准备一个可调用对象，以便将来用作集合类工厂。

给定一个集合类工厂（类型或无参数可调用对象），返回另一个工厂，当调用时将生成兼容的实例。

该函数负责将 collection_class=list 转换为 collection_class=InstrumentedList 的运行时行为。
