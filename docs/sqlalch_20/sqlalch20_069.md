# 突变跟踪

> 原文：[`docs.sqlalchemy.org/en/20/orm/extensions/mutable.html`](https://docs.sqlalchemy.org/en/20/orm/extensions/mutable.html)

提供对标量值的就地更改的跟踪支持，这些更改传播到拥有父对象上的 ORM 更改事件中。

## 在标量列值上建立可变性

“可变”结构的典型示例是 Python 字典。按照 SQL 数据类型对象 中介绍的示例，我们从一个自定义类型开始，该类型将 Python 字典编组为 JSON 字符串，然后再进行持久化：

```py
from sqlalchemy.types import TypeDecorator, VARCHAR
import json

class JSONEncodedDict(TypeDecorator):
    "Represents an immutable structure as a json-encoded string."

    impl = VARCHAR

    def process_bind_param(self, value, dialect):
        if value is not None:
            value = json.dumps(value)
        return value

    def process_result_value(self, value, dialect):
        if value is not None:
            value = json.loads(value)
        return value
```

仅出于示例目的使用 `json`。`sqlalchemy.ext.mutable` 扩展可与任何目标 Python 类型可能是可变的类型一起使用，包括 `PickleType`、`ARRAY` 等。

使用 `sqlalchemy.ext.mutable` 扩展时，值本身会跟踪所有引用它的父对象。下面，我们展示了 `MutableDict` 字典对象的简单版本，它将 `Mutable` mixin 应用于普通 Python 字典：

```py
from sqlalchemy.ext.mutable import Mutable

class MutableDict(Mutable, dict):
    @classmethod
    def coerce(cls, key, value):
        "Convert plain dictionaries to MutableDict."

        if not isinstance(value, MutableDict):
            if isinstance(value, dict):
                return MutableDict(value)

            # this call will raise ValueError
            return Mutable.coerce(key, value)
        else:
            return value

    def __setitem__(self, key, value):
        "Detect dictionary set events and emit change events."

        dict.__setitem__(self, key, value)
        self.changed()

    def __delitem__(self, key):
        "Detect dictionary del events and emit change events."

        dict.__delitem__(self, key)
        self.changed()
```

上述字典类采用了子类化 Python 内置的 `dict` 的方法，以生成一个 dict 子类，该子类通过 `__setitem__` 将所有突变事件路由到。这种方法有其变体，例如子类化 `UserDict.UserDict` 或 `collections.MutableMapping`；对于此示例而言，重要的部分是当数据结构发生就地更改时，将调用 `Mutable.changed()` 方法。

我们还重新定义了 `Mutable.coerce()` 方法，该方法将用于将不是 `MutableDict` 实例的任何值转换为适当的类型，例如 `json` 模块返回的普通字典。定义此方法是可选的；我们也可以创建我们的 `JSONEncodedDict`，使其始终返回 `MutableDict` 的实例，并且还确保所有调用代码都显式使用 `MutableDict`。当未覆盖 `Mutable.coerce()` 时，应用于父对象的任何不是可变类型实例的值都将引发 `ValueError`。

我们的新 `MutableDict` 类型提供了一个类方法 `Mutable.as_mutable()`，我们可以在列元数据中使用它来关联类型。该方法获取给定的类型对象或类，并关联一个监听器，该监听器将检测到该类型的所有未来映射，并对映射的属性应用事件监听仪器。例如，使用经典的表元数据：

```py
from sqlalchemy import Table, Column, Integer

my_data = Table('my_data', metadata,
    Column('id', Integer, primary_key=True),
    Column('data', MutableDict.as_mutable(JSONEncodedDict))
)
```

在上面，`Mutable.as_mutable()` 返回一个 `JSONEncodedDict` 实例（如果类型对象尚不是实例），该实例将拦截针对该类型映射的任何属性。下面我们建立一个简单的映射与 `my_data` 表：

```py
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column

class Base(DeclarativeBase):
    pass

class MyDataClass(Base):
    __tablename__ = 'my_data'
    id: Mapped[int] = mapped_column(primary_key=True)
    data: Mapped[dict[str, str]] = mapped_column(MutableDict.as_mutable(JSONEncodedDict))
```

`MyDataClass.data` 成员现在将收到对其值的原地更改的通知。

对 `MyDataClass.data` 成员的任何原地更改都会在父对象上标记属性为“脏”：

```py
>>> from sqlalchemy.orm import Session

>>> sess = Session(some_engine)
>>> m1 = MyDataClass(data={'value1':'foo'})
>>> sess.add(m1)
>>> sess.commit()

>>> m1.data['value1'] = 'bar'
>>> assert m1 in sess.dirty
True
```

`MutableDict` 可以通过一步关联所有未来的 `JSONEncodedDict` 实例，使用 `Mutable.associate_with()`。这类似于 `Mutable.as_mutable()`，但它将无条件地拦截所有映射中所有 `MutableDict` 的出现，而无需单独声明它：

```py
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column

MutableDict.associate_with(JSONEncodedDict)

class Base(DeclarativeBase):
    pass

class MyDataClass(Base):
    __tablename__ = 'my_data'
    id: Mapped[int] = mapped_column(primary_key=True)
    data: Mapped[dict[str, str]] = mapped_column(JSONEncodedDict)
```

### 支持 Pickling

`sqlalchemy.ext.mutable` 扩展的关键在于在值对象上放置了一个 `weakref.WeakKeyDictionary`，它存储了父映射对象到与该值相关联的属性名称的映射。 `WeakKeyDictionary` 对象不可 pickle，因为它们包含 weakrefs 和函数回调。在我们的情况下，这是件好事，因为如果这个字典是可 pickle 的，那么它可能会导致我们的值对象的 pickle 大小过大，因为它们在不涉及父对象上下文的情况下被单独 pickle。开发人员在这里的责任只是提供一个 `__getstate__` 方法，该方法将 `MutableBase._parents()` 集合从 pickle 流中排除：

```py
class MyMutableType(Mutable):
    def __getstate__(self):
        d = self.__dict__.copy()
        d.pop('_parents', None)
        return d
```

对于我们的字典示例，我们需要返回字典本身的内容（并在 `__setstate__` 上也进行恢复）：

```py
class MutableDict(Mutable, dict):
    # ....

    def __getstate__(self):
        return dict(self)

    def __setstate__(self, state):
        self.update(state)
```

如果我们的可变值对象作为它附加到的一个或多个父对象一起被 pickle，那么 `Mutable` mixin 将在每个值对象上重新建立 `Mutable._parents` 集合，因为拥有父对象本身被 unpickle。

### 接收事件

`AttributeEvents.modified()` 事件处理程序可用于在可变标量发出更改事件时接收事件。 当从可变扩展内调用 `flag_modified()` 函数时，将调用此事件处理程序：

```py
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy import event

class Base(DeclarativeBase):
    pass

class MyDataClass(Base):
    __tablename__ = 'my_data'
    id: Mapped[int] = mapped_column(primary_key=True)
    data: Mapped[dict[str, str]] = mapped_column(MutableDict.as_mutable(JSONEncodedDict))

@event.listens_for(MyDataClass.data, "modified")
def modified_json(instance, initiator):
    print("json value modified:", instance.data)
```  ## 在复合上建立可变性

复合是一种特殊的 ORM 功能，允许将单个标量属性分配给一个对象值，该对象值表示从底层映射表的一个或多个列中“组合”而成的信息。 通常示例是几何“点”，并在 复合列类型 中介绍。

与 `Mutable` 一样，用户定义的复合类将 `MutableComposite` 作为一个混合类，通过 `MutableComposite.changed()` 方法检测并传递更改事件给其父对象。 在复合类的情况下，检测通常通过特殊的 Python 方法 `__setattr__()` 进行。 在下面的示例中，我们扩展了 复合列类型 中介绍的 `Point` 类，以包括 `MutableComposite` 在其基类中，并通过 `__setattr__` 将属性设置事件路由到 `MutableComposite.changed()` 方法：

```py
import dataclasses
from sqlalchemy.ext.mutable import MutableComposite

@dataclasses.dataclass
class Point(MutableComposite):
    x: int
    y: int

    def __setattr__(self, key, value):
        "Intercept set events"

        # set the attribute
        object.__setattr__(self, key, value)

        # alert all parents to the change
        self.changed()
```

`MutableComposite` 类利用类映射事件自动为任何使用指定我们的 `Point` 类型的 `composite()` 的地方建立监听器。 下面，当 `Point` 映射到 `Vertex` 类时，将建立监听器，这些监听器将将来自 `Point` 对象的更改事件路由到每个 `Vertex.start` 和 `Vertex.end` 属性：

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

对 `Vertex.start` 或 `Vertex.end` 成员的任何原地更改都将在父对象上标记该属性为“脏”：

```py
>>> from sqlalchemy.orm import Session
>>> sess = Session(engine)
>>> v1 = Vertex(start=Point(3, 4), end=Point(12, 15))
>>> sess.add(v1)
sql>>> sess.flush()
BEGIN  (implicit)
INSERT  INTO  vertices  (x1,  y1,  x2,  y2)  VALUES  (?,  ?,  ?,  ?)
[...]  (3,  4,  12,  15)
>>> v1.end.x = 8
>>> assert v1 in sess.dirty
True
sql>>> sess.commit()
UPDATE  vertices  SET  x2=?  WHERE  vertices.id  =  ?
[...]  (8,  1)
COMMIT 
```

### 强制转换可变组合

`MutableBase.coerce()` 方法也支持复合类型。对于 `MutableComposite`，`MutableBase.coerce()` 方法仅在属性设置操作时调用，而不在加载操作中调用。覆盖 `MutableBase.coerce()` 方法基本上等同于为使用自定义复合类型的所有属性使用 `validates()` 验证程序：

```py
@dataclasses.dataclass
class Point(MutableComposite):
    # other Point methods
    # ...

    def coerce(cls, key, value):
        if isinstance(value, tuple):
            value = Point(*value)
        elif not isinstance(value, Point):
            raise ValueError("tuple or Point expected")
        return value
```

### 支持 Pickling

与 `Mutable` 类似，`MutableComposite` 辅助类使用 `weakref.WeakKeyDictionary`，可通过 `MutableBase._parents()` 属性获得，该属性不可 picklable。如果我们需要 pickle `Point` 的实例或其所属的类 `Vertex`，我们至少需要定义一个不包含 `_parents` 字典的 `__getstate__`。下面我们定义了 `Point` 类的最小形式的 `__getstate__` 和 `__setstate__`： 

```py
@dataclasses.dataclass
class Point(MutableComposite):
    # ...

    def __getstate__(self):
        return self.x, self.y

    def __setstate__(self, state):
        self.x, self.y = state
```

与 `Mutable` 一样，`MutableComposite` 增强了父对象的对象关系状态的 pickling 过程，以便 `MutableBase._parents()` 集合被恢复到所有 `Point` 对象中。

## API 参考

| 对象名称 | 描述 |
| --- | --- |
| Mutable | 混合类，定义对父对象的变更事件的透明传播。 |
| MutableBase | `Mutable` 和 `MutableComposite` 的通用基类。 |
| MutableComposite | 混合类，定义对 SQLAlchemy “composite” 对象的变更事件的透明传播，传播到其拥有的父对象或父对象。 |
| MutableDict | 实现了 `Mutable` 的字典类型。 |
| MutableList | 实现了 `Mutable` 的列表类型。 |
| MutableSet | 实现`Mutable`的集合类型。 |

```py
class sqlalchemy.ext.mutable.MutableBase
```

**成员**

_parents，coerce()

公共基类，用于`Mutable`和`MutableComposite`。

```py
attribute _parents
```

父对象的`InstanceState`->父对象上的属性名称的字典。

该属性是所谓的“记忆化”属性。首次访问时，它会使用一个新的`weakref.WeakKeyDictionary`进行初始化，并在后续访问时返回相同的对象。

在 1.4 版本中更改：现在使用`InstanceState`作为弱字典中的键，而不是实例本身。

```py
classmethod coerce(key: str, value: Any) → Any | None
```

给定一个值，将其强制转换为目标类型。

可以被自定义子类重写，将传入数据强制转换为特定类型。

默认情况下，引发`ValueError`。

根据父类是`Mutable`类型还是`MutableComposite`类型，在不同情况下调用此方法。对于前者，它在属性设置操作和 ORM 加载操作期间都会被调用。对于后者，它仅在属性设置操作期间被调用；`composite()`构造的机制在加载操作期间处理强制转换。

参数：

+   `key` – 正在设置的 ORM 映射属性的字符串名称。

+   `value` – 输入值。

返回：

如果无法完成强制转换，则该方法应返回强制转换后的值，或引发`ValueError`。

```py
class sqlalchemy.ext.mutable.Mutable
```

定义透明传播更改事件到父对象的混入。

查看在标量列值上建立可变性中的示例以获取用法信息。

**成员**

_get_listen_keys()，_listen_on_attribute()，_parents，as_mutable()，associate_with()，associate_with_attribute()，changed()，coerce()

**类签名**

类`sqlalchemy.ext.mutable.Mutable`（`sqlalchemy.ext.mutable.MutableBase`)

```py
classmethod _get_listen_keys(attribute: QueryableAttribute[Any]) → Set[str]
```

*继承自* `MutableBase` *的* `sqlalchemy.ext.mutable.MutableBase._get_listen_keys` *方法*

给定一个描述符属性，返回一个指示此属性状态变化的属性键的`set()`。

这通常只是`set([attribute.key])`，但可以被覆盖以提供额外的键。例如，`MutableComposite`会用与组成复合值的列相关联的属性键来增加这个集合。

在拦截`InstanceEvents.refresh()`和`InstanceEvents.refresh_flush()`事件时，将查询此集合，这些事件传递了已刷新的属性名称列表；该列表与此集合进行比较，以确定是否需要采取行动。

```py
classmethod _listen_on_attribute(attribute: QueryableAttribute[Any], coerce: bool, parent_cls: _ExternalEntityType[Any]) → None
```

*继承自* `MutableBase` *的* `sqlalchemy.ext.mutable.MutableBase._listen_on_attribute` *方法*

将此类型建立为给定映射描述符的变异监听器。

```py
attribute _parents
```

*继承自* `MutableBase` *的* `sqlalchemy.ext.mutable.MutableBase._parents` *属性*

父对象的`InstanceState`->父对象上的属性名的字典。

此属性是所谓的“记忆化”属性。它在第一次访问时使用一个新的`weakref.WeakKeyDictionary`进行初始化，并在后续访问时返回相同的对象。

自版本 1.4 更改：`InstanceState`现在作为弱字典中的键，而不是实例本身。

```py
classmethod as_mutable(sqltype: TypeEngine) → TypeEngine
```

将 SQL 类型与此可变 Python 类型关联起来。

这将建立侦听器，以检测针对给定类型的 ORM 映射，并向这些映射添加变异事件跟踪器。

类型无条件地作为实例返回，因此可以内联使用`as_mutable()`：

```py
Table('mytable', metadata,
    Column('id', Integer, primary_key=True),
    Column('data', MyMutableType.as_mutable(PickleType))
)
```

请注意，返回的类型始终是一个实例，即使给定一个类，也只有明确声明了该类型实例的列才会接收到额外的仪器设备。

要将特定的可变类型与所有特定类型的所有出现相关联，请使用特定`Mutable`子类的`Mutable.associate_with()`类方法来建立全局关联。

警告

此方法建立的侦听器对所有映射器都是*全局*的，并且*不*会被垃圾回收。只能对应用程序中永久的类型使用`as_mutable()`，不要与临时类型一起使用，否则这将导致内存使用量无限增长。

```py
classmethod associate_with(sqltype: type) → None
```

将此包装器与未来的给定类型的映射列相关联。

这是一个方便的方法，会自动调用`associate_with_attribute`。

警告

此方法建立的侦听器对所有映射器都是*全局*的，并且*不*会被垃圾回收。只能对应用程序中永久的类型使用`associate_with()`，不要与临时类型一起使用，否则这将导致内存使用量无限增长。

```py
classmethod associate_with_attribute(attribute: InstrumentedAttribute[_O]) → None
```

将此类型建立为给定映射描述符的变异侦听器。

```py
method changed() → None
```

子类应该在发生变更事件时调用此方法。

```py
classmethod coerce(key: str, value: Any) → Any | None
```

*继承自* `MutableBase.coerce()` *方法的* `MutableBase`

给定一个值，将其强制转换为目标类型。

可以由自定义子类重写以将传入数据强制转换为特定类型。

默认情况下，引发`ValueError`。

根据父类是`Mutable`类型还是`MutableComposite`类型，在不同的情况下调用此方法。对于前者，在属性设置操作和 ORM 加载操作期间都会调用它。对于后者，在属性设置操作期间才会调用它；`composite()`构造的机制处理加载操作期间的强制转换。

参数:

+   `key` – 正在设置的 ORM 映射属性的字符串名称。

+   `value` – 输入值。

返回：

如果无法完成转换，则该方法应返回转换后的值，或引发`ValueError`。

```py
class sqlalchemy.ext.mutable.MutableComposite
```

混入，定义了将 SQLAlchemy“组合”对象上的变更事件透明传播到其拥有的父对象的机制。

查看在组合上建立可变性中的示例以获取用法信息。

**成员**

changed()

**类签名**

类 `sqlalchemy.ext.mutable.MutableComposite`（`sqlalchemy.ext.mutable.MutableBase`）

```py
method changed() → None
```

子类应在更改事件发生时调用此方法。

```py
class sqlalchemy.ext.mutable.MutableDict
```

一种实现了 `Mutable` 的字典类型。

`MutableDict` 对象实现了一个字典，当更改字典的内容时会向底层映射发送更改事件，包括添加或删除值时。

请注意，`MutableDict` **不会**将可变跟踪应用于字典内部的*值本身*。因此，它不足以解决跟踪对*递归*字典结构进行深层更改的用例，例如 JSON 结构。要支持此用例，请构建 `MutableDict` 的子类，该子类提供适当的强制转换，以便将放置在字典中的值也“可变”，并将事件发送到其父结构。

另请参阅

`MutableList`

`MutableSet`

**成员**

clear(), coerce(), pop(), popitem(), setdefault(), update()

**类签名**

类 `sqlalchemy.ext.mutable.MutableDict`（`sqlalchemy.ext.mutable.Mutable`，`builtins.dict`，`typing.Generic`）

```py
method clear() → None.  Remove all items from D.
```

```py
classmethod coerce(key: str, value: Any) → MutableDict[_KT, _VT] | None
```

将普通字典转换为此类的实例。

```py
method pop(k[, d]) → v, remove specified key and return the corresponding value.
```

如果找不到键，则在给定默认值的情况下返回；否则，引发 KeyError。

```py
method popitem() → Tuple[_KT, _VT]
```

移除并返回一个（键，值）对作为 2 元组。

以 LIFO（后进先出）顺序返回键值对。如果字典为空，则引发 KeyError。

```py
method setdefault(*arg)
```

如果字典中没有键，则将键插入并将其值设置为默认值。

如果字典中存在键，则返回键的值，否则返回默认值。

```py
method update([E, ]**F) → None.  Update D from dict/iterable E and F.
```

如果 E 存在且具有 `.keys()` 方法，则执行以下操作：for k in E: D[k] = E[k] 如果 E 存在但缺少 `.keys()` 方法，则执行以下操作：for k, v in E: D[k] = v 在任一情况下，接下来执行以下操作：for k in F: D[k] = F[k]

```py
class sqlalchemy.ext.mutable.MutableList
```

一种实现了 `Mutable` 的列表类型。

`MutableList`对象实现了一个列表，在修改列表内容时会向底层映射发出更改事件，包括添加或删除值时。

注意`MutableList`不会对列表内部的*值本身*应用可变跟踪。因此，它不能解决跟踪*递归*可变结构（例如 JSON 结构）的深层更改的用例。要支持此用例，构建`MutableList`的子类，提供适当的强制转换以使放置在字典中的值也是“可变的”，并将事件传播到其父结构。

另请参见

`MutableDict`

`MutableSet`

**成员**

append(), clear(), coerce(), extend(), insert(), is_iterable(), is_scalar(), pop(), remove(), reverse(), sort()

**类签名**

类`sqlalchemy.ext.mutable.MutableList`（`sqlalchemy.ext.mutable.Mutable`，`builtins.list`，`typing.Generic`）

```py
method append(x: _T) → None
```

将对象追加到列表末尾。

```py
method clear() → None
```

从列表中删除所有项。

```py
classmethod coerce(key: str, value: MutableList[_T] | _T) → MutableList[_T] | None
```

将普通列表转换为此类的实例。

```py
method extend(x: Iterable[_T]) → None
```

通过从可迭代对象中追加元素来扩展列表。

```py
method insert(i: SupportsIndex, x: _T) → None
```

在索引之前插入对象。

```py
method is_iterable(value: _T | Iterable[_T]) → TypeGuard[Iterable[_T]]
```

```py
method is_scalar(value: _T | Iterable[_T]) → TypeGuard[_T]
```

```py
method pop(*arg: SupportsIndex) → _T
```

移除并返回索引处的项（默认为最后一个）。

如果列表为空或索引超出范围，则引发 IndexError。

```py
method remove(i: _T) → None
```

移除第一次出现的值。

如果值不存在，则引发 ValueError。

```py
method reverse() → None
```

*原地*反转。

```py
method sort(**kw: Any) → None
```

对列表进行升序排序并返回 None。

排序是原地进行的（即修改列表本身）并且是稳定的（即保持两个相等元素的顺序）。

如果给定了键函数，则将其一次应用于每个列表项并根据其函数值升序或降序排序。

反转标志可以设置为按降序排序。

```py
class sqlalchemy.ext.mutable.MutableSet
```

实现了`Mutable`的集合类型。

`MutableSet` 对象实现了一个集合，当集合的内容发生更改时，将向底层映射发出更改事件，包括添加或删除值时。

请注意，`MutableSet` **不会**对集合中*值本身*应用可变跟踪。因此，它不是跟踪*递归*可变结构的深层更改的足够解决方案。为了支持这种用例，请构建一个`MutableSet`的子类，该子类提供适当的强制转换，使放置在字典中的值也是“可变的”，并向其父结构发出事件。

另请参阅

`MutableDict`

`MutableList`

**成员**

add(), clear(), coerce(), difference_update(), discard(), intersection_update(), pop(), remove(), symmetric_difference_update(), update()

**类签名**

类 `sqlalchemy.ext.mutable.MutableSet`（`sqlalchemy.ext.mutable.Mutable`， `builtins.set`， `typing.Generic`）

```py
method add(elem: _T) → None
```

向集合添加一个元素。

如果元素已经存在，则不起作用。

```py
method clear() → None
```

从此集合中移除所有元素。

```py
classmethod coerce(index: str, value: Any) → MutableSet[_T] | None
```

将普通集合转换为此类的实例。

```py
method difference_update(*arg: Iterable[Any]) → None
```

从此集合中删除另一个集合的所有元素。

```py
method discard(elem: _T) → None
```

如果元素是成员，则从集合中删除一个元素。

如果元素不是成员，则不执行任何操作。

```py
method intersection_update(*arg: Iterable[Any]) → None
```

使用自身与另一个集合的交集更新集合。

```py
method pop(*arg: Any) → _T
```

移除并返回一个任意的集合元素。如果集合为空，则引发 KeyError。

```py
method remove(elem: _T) → None
```

从集合中删除一个元素；它必须是成员。

如果元素不是成员，则引发 KeyError。

```py
method symmetric_difference_update(*arg: Iterable[_T]) → None
```

使用自身与另一个集合的对称差更新集合。

```py
method update(*arg: Iterable[_T]) → None
```

使用自身与其他集合的并集更新集合。

## 在标量列值上建立可变性

“可变”结构的典型示例是 Python 字典。在 SQL 数据类型对象中介绍的示例中，我们从自定义类型开始，该类型在持久化之前将 Python 字典编组为 JSON 字符串：

```py
from sqlalchemy.types import TypeDecorator, VARCHAR
import json

class JSONEncodedDict(TypeDecorator):
    "Represents an immutable structure as a json-encoded string."

    impl = VARCHAR

    def process_bind_param(self, value, dialect):
        if value is not None:
            value = json.dumps(value)
        return value

    def process_result_value(self, value, dialect):
        if value is not None:
            value = json.loads(value)
        return value
```

使用`json`仅用于示例目的。`sqlalchemy.ext.mutable` 扩展可以与任何目标 Python 类型可能是可变的类型一起使用，包括`PickleType`、`ARRAY`等。

当使用`sqlalchemy.ext.mutable` 扩展时，值本身跟踪所有引用它的父对象。下面，我们展示了一个简单版本的`MutableDict`字典对象，它将`Mutable` mixin 应用于普通的 Python 字典：

```py
from sqlalchemy.ext.mutable import Mutable

class MutableDict(Mutable, dict):
    @classmethod
    def coerce(cls, key, value):
        "Convert plain dictionaries to MutableDict."

        if not isinstance(value, MutableDict):
            if isinstance(value, dict):
                return MutableDict(value)

            # this call will raise ValueError
            return Mutable.coerce(key, value)
        else:
            return value

    def __setitem__(self, key, value):
        "Detect dictionary set events and emit change events."

        dict.__setitem__(self, key, value)
        self.changed()

    def __delitem__(self, key):
        "Detect dictionary del events and emit change events."

        dict.__delitem__(self, key)
        self.changed()
```

上述字典类采用了子类化 Python 内置的`dict`的方法，以产生一个 dict 子类，通过`__setitem__`路由所有的变异事件。这种方法还有变体，比如子类化`UserDict.UserDict`或`collections.MutableMapping`；对于这个示例很重要的部分是，每当对数据结构进行就地更改时，都会调用`Mutable.changed()`方法。

我们还重新定义了`Mutable.coerce()` 方法，用于将不是`MutableDict`实例的任何值转换为适当的类型，比如`json`模块返回的普通字典。定义这个方法是可选的；我们也可以创建我们的`JSONEncodedDict`，使其始终返回`MutableDict`的实例，并确保所有调用代码都明确使用`MutableDict`。当未覆盖`Mutable.coerce()`时，应用于父对象的任何不是可变类型实例的值将引发`ValueError`。

我们的新`MutableDict`类型提供了一个类方法`Mutable.as_mutable()`，我们可以在列元数据中使用它与类型关联。这个方法获取给定的类型对象或类，并关联一个监听器，将检测到所有将来映射到该类型的映射，应用事件监听仪器到映射的属性。例如，使用经典表元数据：

```py
from sqlalchemy import Table, Column, Integer

my_data = Table('my_data', metadata,
    Column('id', Integer, primary_key=True),
    Column('data', MutableDict.as_mutable(JSONEncodedDict))
)
```

上面，`Mutable.as_mutable()` 返回一个`JSONEncodedDict`的实例（如果类型对象尚未是一个实例），它将拦截任何映射到该类型的属性。下面我们建立一个简单的映射到`my_data`表：

```py
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column

class Base(DeclarativeBase):
    pass

class MyDataClass(Base):
    __tablename__ = 'my_data'
    id: Mapped[int] = mapped_column(primary_key=True)
    data: Mapped[dict[str, str]] = mapped_column(MutableDict.as_mutable(JSONEncodedDict))
```

`MyDataClass.data` 成员现在将被通知其值的原地更改。

对 `MyDataClass.data` 成员的任何原地更改都将标记父对象的属性为“脏”：

```py
>>> from sqlalchemy.orm import Session

>>> sess = Session(some_engine)
>>> m1 = MyDataClass(data={'value1':'foo'})
>>> sess.add(m1)
>>> sess.commit()

>>> m1.data['value1'] = 'bar'
>>> assert m1 in sess.dirty
True
```

`MutableDict` 可以通过一个步骤与所有未来的 `JSONEncodedDict` 实例关联，使用 `Mutable.associate_with()`。这类似于 `Mutable.as_mutable()`，但它将无条件拦截所有映射中 `MutableDict` 的所有出现，而无需单独声明它：

```py
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column

MutableDict.associate_with(JSONEncodedDict)

class Base(DeclarativeBase):
    pass

class MyDataClass(Base):
    __tablename__ = 'my_data'
    id: Mapped[int] = mapped_column(primary_key=True)
    data: Mapped[dict[str, str]] = mapped_column(JSONEncodedDict)
```

### 支持 Pickling

`sqlalchemy.ext.mutable` 扩展的关键在于在值对象上放置一个 `weakref.WeakKeyDictionary`，它存储了父映射对象的映射，键为它们与该值相关联的属性名。 `WeakKeyDictionary` 对象不可 pickle，因为它们包含弱引用和函数回调。在我们的情况下，这是一件好事，因为如果这个字典是可 pickle 的，它可能会导致独立于父对象上下文的值对象的 pickle 大小过大。在这里，开发者的责任仅仅是提供一个 `__getstate__` 方法，从 pickle 流中排除 `MutableBase._parents()` 集合：

```py
class MyMutableType(Mutable):
    def __getstate__(self):
        d = self.__dict__.copy()
        d.pop('_parents', None)
        return d
```

对于我们的字典示例，我们需要返回字典本身的内容（并在 __setstate__ 中还原它们）：

```py
class MutableDict(Mutable, dict):
    # ....

    def __getstate__(self):
        return dict(self)

    def __setstate__(self, state):
        self.update(state)
```

如果我们的可变值对象被 pickle，而它附加到一个或多个也是 pickle 的父对象上，`Mutable` mixin 将在每个值对象上重新建立 `Mutable._parents` 集合，因为拥有父对象本身被 unpickle。

### 接收事件

当一个可变标量发出变更事件时，`AttributeEvents.modified()` 事件处理程序可以用于接收事件。当在可变扩展内部调用 `flag_modified()` 函数时，将调用此事件处理程序：

```py
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy import event

class Base(DeclarativeBase):
    pass

class MyDataClass(Base):
    __tablename__ = 'my_data'
    id: Mapped[int] = mapped_column(primary_key=True)
    data: Mapped[dict[str, str]] = mapped_column(MutableDict.as_mutable(JSONEncodedDict))

@event.listens_for(MyDataClass.data, "modified")
def modified_json(instance, initiator):
    print("json value modified:", instance.data)
```

### 支持 Pickling

`sqlalchemy.ext.mutable` 扩展的关键在于在值对象上放置一个 `weakref.WeakKeyDictionary`，该字典存储父映射对象的映射，以属性名称为键，这些父映射对象与该值相关联。由于 `WeakKeyDictionary` 对象包含弱引用和函数回调，因此它们不可 picklable。在我们的情况下，这是一件好事，因为如果这个字典是可 pickle 的，那么它可能会导致我们的值对象的 pickle 大小过大，这些值对象是在不涉及父对象的情况下 pickle 的。开发者在这里的责任只是提供一个 `__getstate__` 方法，该方法从 pickle 流中排除了 `MutableBase._parents()` 集合：

```py
class MyMutableType(Mutable):
    def __getstate__(self):
        d = self.__dict__.copy()
        d.pop('_parents', None)
        return d
```

对于我们的字典示例，我们需要返回字典本身的内容（并在 __setstate__ 中还原它们）：

```py
class MutableDict(Mutable, dict):
    # ....

    def __getstate__(self):
        return dict(self)

    def __setstate__(self, state):
        self.update(state)
```

在我们可变的值对象作为 pickle 对象时，如果它附加在一个或多个也是 pickle 的父对象上，`Mutable` mixin 将在每个值对象上重新建立 `Mutable._parents` 集合，因为拥有父对象的本身会被 unpickle。

### 接收事件

`AttributeEvents.modified()` 事件处理程序可用于在可变标量发出更改事件时接收事件。当在可变扩展中调用 `flag_modified()` 函数时，将调用此事件处理程序：

```py
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy import event

class Base(DeclarativeBase):
    pass

class MyDataClass(Base):
    __tablename__ = 'my_data'
    id: Mapped[int] = mapped_column(primary_key=True)
    data: Mapped[dict[str, str]] = mapped_column(MutableDict.as_mutable(JSONEncodedDict))

@event.listens_for(MyDataClass.data, "modified")
def modified_json(instance, initiator):
    print("json value modified:", instance.data)
```

## 确立组合物的可变性

组合物是 ORM 的一种特殊功能，它允许将单个标量属性分配给一个对象值，该对象值表示从底层映射表中的一个或多个列中“组合”出的信息。通常的例子是几何“点”，并在 Composite Column Types 中介绍。

与`Mutable`类似，用户定义的复合类作为一个混合类继承`MutableComposite`，通过`MutableComposite.changed()`方法检测并传递更改事件给其父类。对于复合类，通常是通过使用特殊的 Python 方法`__setattr__()`来进行检测。在下面的示例中，我们扩展了复合列类型中介绍的`Point`类，将`MutableComposite`包含在其基类中，并通过`__setattr__`将属性设置事件路由到`MutableComposite.changed()`方法：

```py
import dataclasses
from sqlalchemy.ext.mutable import MutableComposite

@dataclasses.dataclass
class Point(MutableComposite):
    x: int
    y: int

    def __setattr__(self, key, value):
        "Intercept set events"

        # set the attribute
        object.__setattr__(self, key, value)

        # alert all parents to the change
        self.changed()
```

`MutableComposite`类利用类映射事件自动为任何指定我们的`Point`类型的`composite()`的使用建立监听器。下面，当`Point`映射到`Vertex`类时，将建立监听器，这些监听器将把`Point`对象的更改事件路由到`Vertex.start`和`Vertex.end`属性中的每一个：

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

对`Vertex.start`或`Vertex.end`成员的任何就地更改都会在父对象上标记属性为“脏”：

```py
>>> from sqlalchemy.orm import Session
>>> sess = Session(engine)
>>> v1 = Vertex(start=Point(3, 4), end=Point(12, 15))
>>> sess.add(v1)
sql>>> sess.flush()
BEGIN  (implicit)
INSERT  INTO  vertices  (x1,  y1,  x2,  y2)  VALUES  (?,  ?,  ?,  ?)
[...]  (3,  4,  12,  15)
>>> v1.end.x = 8
>>> assert v1 in sess.dirty
True
sql>>> sess.commit()
UPDATE  vertices  SET  x2=?  WHERE  vertices.id  =  ?
[...]  (8,  1)
COMMIT 
```

### 强制可变复合类型

在复合类型上也支持`MutableBase.coerce()`方法。对于`MutableComposite`，`MutableBase.coerce()`方法仅在属性设置操作中调用，而不是在加载操作中调用。覆盖`MutableBase.coerce()`方法基本上等同于为使用自定义复合类型的所有属性使用`validates()`验证程序：

```py
@dataclasses.dataclass
class Point(MutableComposite):
    # other Point methods
    # ...

    def coerce(cls, key, value):
        if isinstance(value, tuple):
            value = Point(*value)
        elif not isinstance(value, Point):
            raise ValueError("tuple or Point expected")
        return value
```

### 支持 Pickling

与`Mutable`类似，`MutableComposite`辅助类使用了通过`MutableBase._parents()`属性获得的`weakref.WeakKeyDictionary`，该字典不可 pickle。如果我们需要 pickle `Point` 或其拥有类 `Vertex` 的实例，至少需要定义一个不包含 `_parents` 字典的`__getstate__`。下面我们定义了`Point`类的最小形式的`__getstate__`和`__setstate__`：

```py
@dataclasses.dataclass
class Point(MutableComposite):
    # ...

    def __getstate__(self):
        return self.x, self.y

    def __setstate__(self, state):
        self.x, self.y = state
```

与`Mutable`类似，`MutableComposite`增强了父对象的对象关系状态的 pickling 过程，以便将`MutableBase._parents()`集合还原为所有`Point`对象。

### 强制转换可变复合类型

`MutableBase.coerce()`方法也支持复合类型。在`MutableComposite`的情况下，`MutableBase.coerce()`方法仅在属性设置操作而非加载操作时调用。覆盖`MutableBase.coerce()`方法基本上等同于对使用自定义复合类型的所有属性使用`validates()`验证程序：

```py
@dataclasses.dataclass
class Point(MutableComposite):
    # other Point methods
    # ...

    def coerce(cls, key, value):
        if isinstance(value, tuple):
            value = Point(*value)
        elif not isinstance(value, Point):
            raise ValueError("tuple or Point expected")
        return value
```

### 支持 Pickling

与`Mutable`类似，`MutableComposite`辅助类使用了通过`MutableBase._parents()`属性获得的`weakref.WeakKeyDictionary`，该字典不可 pickle。如果我们需要 pickle `Point` 或其拥有类 `Vertex` 的实例，至少需要定义一个不包含 `_parents` 字典的`__getstate__`。下面我们定义了`Point`类的最小形式的`__getstate__`和`__setstate__`：

```py
@dataclasses.dataclass
class Point(MutableComposite):
    # ...

    def __getstate__(self):
        return self.x, self.y

    def __setstate__(self, state):
        self.x, self.y = state
```

与`Mutable`一样，`MutableComposite`增强了父对象的对象关系状态的 pickling 过程，以便将`MutableBase._parents()`集合还原为所有`Point`对象。

## API 参考

| 对象名称 | 描述 |
| --- | --- |
| Mutable | 定义更改事件透明传播到父对象的混合类。 |
| MutableBase | `Mutable`和`MutableComposite`的通用基类。 |
| MutableComposite | 定义 SQLAlchemy “composite” 对象上的更改事件透明传播到其拥有的父对象或父对象的混合类。 |
| MutableDict | 一个实现`Mutable`的字典类型。 |
| MutableList | 一个实现`Mutable`的列表类型。 |
| MutableSet | 一个实现`Mutable`的集合类型。 |

```py
class sqlalchemy.ext.mutable.MutableBase
```

**成员**

_parents, coerce()

`Mutable`和`MutableComposite`的通用基类。

```py
attribute _parents
```

父对象的`InstanceState`字典->父对象上的属性名称。

此属性是所谓的“记忆化”属性。第一次访问时，它会用一个新的`weakref.WeakKeyDictionary`初始化自己，并在后续访问时返回相同的对象。

在 1.4 版中更改：`InstanceState`现在被用作弱字典中的键，而不是实例本身。

```py
classmethod coerce(key: str, value: Any) → Any | None
```

给定一个值，将其强制转换为目标类型。

可以被自定义子类覆盖，将传入数据强制转换为特定类型。

默认情况下，引发`ValueError`。

根据父类是`Mutable`类型还是`MutableComposite`类型，在不同的情况下调用此方法。对于前者，它在属性集操作和 ORM 加载操作期间都会被调用。对于后者，它仅在属性集操作期间被调用；`composite()`构造的机制在加载操作期间处理强制转换。

参数：

+   `key` – 正在设置的 ORM 映射属性的字符串名称。

+   `value` – 传入的值。

返回：

如果无法完成强制转换，该方法应返回强制转换后的值，或引发`ValueError`。

```py
class sqlalchemy.ext.mutable.Mutable
```

定义将更改事件透明传播到父对象的混合类。

查看在标量列值上建立可变性中的示例以获取用法信息。

**成员**

_get_listen_keys(), _listen_on_attribute(), _parents, as_mutable(), associate_with(), associate_with_attribute(), changed(), coerce()

**类签名**

类`sqlalchemy.ext.mutable.Mutable`（`sqlalchemy.ext.mutable.MutableBase`）

```py
classmethod _get_listen_keys(attribute: QueryableAttribute[Any]) → Set[str]
```

*继承自* `MutableBase` *的* `sqlalchemy.ext.mutable.MutableBase._get_listen_keys` *方法*

给定一个描述符属性，返回指示此属性状态变化的属性键的`set()`。

通常只是`set([attribute.key])`，但可以被覆盖以提供额外的键。例如，`MutableComposite` 会用包含组合值的列相关联的属性键来增加这个集合。

在拦截`InstanceEvents.refresh()`和`InstanceEvents.refresh_flush()`事件时，会查询此集合，这些事件会传递一个已刷新的属性名称列表；该列表将与此集合进行比较，以确定是否需要采取行动。

```py
classmethod _listen_on_attribute(attribute: QueryableAttribute[Any], coerce: bool, parent_cls: _ExternalEntityType[Any]) → None
```

*继承自* `MutableBase` 的 `sqlalchemy.ext.mutable.MutableBase._listen_on_attribute` *方法*

将此类型作为给定映射描述符的变异监听器。

```py
attribute _parents
```

*继承自* `MutableBase` 的 `sqlalchemy.ext.mutable.MutableBase._parents` *属性*

父对象的`InstanceState`->父对象上的属性名称的字典。

此属性是所谓的“记忆化”属性。它在首次访问时使用一个新的`weakref.WeakKeyDictionary`进行初始化，并在后续访问时返回相同的对象。

在 1.4 版本中更改：现在使用`InstanceState`作为弱字典中的键，而不是实例本身。

```py
classmethod as_mutable(sqltype: TypeEngine) → TypeEngine
```

将 SQL 类型与此可变 Python 类型关联。

这将建立监听器，用于检测针对给定类型的 ORM 映射，向这些映射添加变异事件跟踪器。

该类型无条件地作为一个实例返回，以便可以内联使用`as_mutable()`：

```py
Table('mytable', metadata,
    Column('id', Integer, primary_key=True),
    Column('data', MyMutableType.as_mutable(PickleType))
)
```

请注意，返回的类型始终是一个实例，即使给定一个类，也只有明确声明了该类型实例的列才会接收到额外的仪器化。

要将特定的可变类型与特定类型的所有出现关联起来，请使用`Mutable.associate_with()`类方法的特定`Mutable`子类来建立全局关联。

警告

此方法建立的监听器是*全局*的，适用于所有映射器，并且*不*会被垃圾回收。只能对应用程序中永久的类型使用`as_mutable()`，而不是临时类型，否则会导致内存使用量无限增长。

```py
classmethod associate_with(sqltype: type) → None
```

将此包装器与将来的给定类型的映射列关联起来。

这是一个方便的方法，会自动调用`associate_with_attribute`。

警告

该方法建立的监听器是*全局*的，适用于所有映射器，并且*不*会被垃圾回收。只能对应用程序中永久的类型使用`associate_with()`，而不是临时类型，否则会导致内存使用量无限增长。

```py
classmethod associate_with_attribute(attribute: InstrumentedAttribute[_O]) → None
```

将此类型作为给定映射描述符的变异监听器。

```py
method changed() → None
```

子类在发生更改事件时应调用此方法。

```py
classmethod coerce(key: str, value: Any) → Any | None
```

*继承自* `MutableBase.coerce()` *方法* 的 `MutableBase`

给定一个值，将其强制转换为目标类型。

可以被自定义子类重写以将传入的数据强制转换为特定类型。

默认情况下，引发 `ValueError`。

此方法在不同的情况下被调用，具体取决于父类是 `Mutable` 类型还是 `MutableComposite` 类型。在前者的情况下，它将在属性设置操作以及 ORM 加载操作期间被调用。对于后者，它仅在属性设置操作期间被调用；`composite()` 构造的机制在加载操作期间处理强制转换。

参数：

+   `key` – 被设置的 ORM 映射属性的字符串名称。

+   `value` – 传入的值。

返回：

如果无法完成强制转换，则该方法应返回强制转换后的值，或引发 `ValueError`。

```py
class sqlalchemy.ext.mutable.MutableComposite
```

定义了对 SQLAlchemy “组合”对象的更改事件的透明传播的混合类到其拥有的父对象。

请参阅 在组合上建立可变性 中的示例以获取用法信息。

**成员**

changed()

**类签名**

class `sqlalchemy.ext.mutable.MutableComposite` (`sqlalchemy.ext.mutable.MutableBase`)

```py
method changed() → None
```

子类应在更改事件发生时调用此方法。

```py
class sqlalchemy.ext.mutable.MutableDict
```

实现了 `Mutable` 的字典类型。

`MutableDict` 对象实现了一个字典，在字典内容发生更改时将向基础映射发出更改事件，包括添加或移除值时。

请注意，`MutableDict` **不会** 对字典内部的*值本身*应用可变跟踪。因此，它不足以解决跟踪*递归*字典结构（例如 JSON 结构）的深层更改的用例。要支持此用例，请构建一个 `MutableDict` 的子类，以提供适当的强制转换，以便放置在字典中的值也是“可变的”，并将事件传播到其父结构。

另请参见

`MutableList`

`MutableSet`

**成员**

clear(), coerce(), pop(), popitem(), setdefault(), update()

**类签名**

类`sqlalchemy.ext.mutable.MutableDict` (`sqlalchemy.ext.mutable.Mutable`, `builtins.dict`, `typing.Generic`)

```py
method clear() → None.  Remove all items from D.
```

```py
classmethod coerce(key: str, value: Any) → MutableDict[_KT, _VT] | None
```

将普通字典转换为此类的实例。

```py
method pop(k[, d]) → v, remove specified key and return the corresponding value.
```

如果找不到键，则返回默认值（如果给定）；否则，引发 KeyError。

```py
method popitem() → Tuple[_KT, _VT]
```

移除并返回一个(key, value)对作为 2 元组。

键值对以 LIFO（后进先出）顺序返回。如果字典为空，则引发 KeyError。

```py
method setdefault(*arg)
```

如果键不在字典中，则将键插入并设置默认值。

如果键在字典中，则返回键的值，否则返回默认值。

```py
method update([E, ]**F) → None.  Update D from dict/iterable E and F.
```

如果 E 存在并且具有.keys()方法，则执行： for k in E: D[k] = E[k] 如果 E 存在但缺少.keys()方法，则执行： for k, v in E: D[k] = v 在任一情况下，接下来执行： for k in F: D[k] = F[k]

```py
class sqlalchemy.ext.mutable.MutableList
```

一个实现了`Mutable`的列表类型。

`MutableList` 对象实现了一个列表，当列表的内容被更改时，包括添加或删除值时，将向底层映射发送更改事件。

请注意，`MutableList` 不会对列表内部的*值本身*应用可变跟踪。因此，它不是跟踪对*递归*可变结构进行深层更改的使用案例的充分解决方案，例如 JSON 结构。为支持此使用案例，请构建`MutableList`的子类，该子类提供适当的强制转换以使放置在字典中的值也是“可变的”，并将事件发送到其父结构。

另请参阅

`MutableDict`

`MutableSet`

**成员**

append(), clear(), coerce(), extend(), insert(), is_iterable(), is_scalar(), pop(), remove(), reverse(), sort()

**类签名**

类`sqlalchemy.ext.mutable.MutableList`（`sqlalchemy.ext.mutable.Mutable`，`builtins.list`，`typing.Generic`）

```py
method append(x: _T) → None
```

将对象追加到列表末尾。

```py
method clear() → None
```

从列表中删除所有项。

```py
classmethod coerce(key: str, value: MutableList[_T] | _T) → MutableList[_T] | None
```

将普通列表转换为此类的实例。

```py
method extend(x: Iterable[_T]) → None
```

通过将来自可迭代对象的元素附加到列表来扩展列表。

```py
method insert(i: SupportsIndex, x: _T) → None
```

在索引之前插入对象。

```py
method is_iterable(value: _T | Iterable[_T]) → TypeGuard[Iterable[_T]]
```

```py
method is_scalar(value: _T | Iterable[_T]) → TypeGuard[_T]
```

```py
method pop(*arg: SupportsIndex) → _T
```

删除并返回索引处的项（默认为最后一个）。

如果列表为空或索引超出范围，则引发 IndexError。

```py
method remove(i: _T) → None
```

删除值的第一个出现。

如果值不存在，则引发 ValueError。

```py
method reverse() → None
```

就地反转。

```py
method sort(**kw: Any) → None
```

将列表按升序排序并返回 None。

排序是原地进行的（即列表本身被修改）并且稳定的（即保持两个相等元素的顺序不变）。

如果给定了键函数，则将其应用于每个列表项一次，并根据其函数值按升序或降序对它们进行排序。

反转标志可以设置为按降序排序。

```py
class sqlalchemy.ext.mutable.MutableSet
```

实现了`Mutable`的集合类型。

`MutableSet` 对象实现了一个集合，当集合的内容发生变化时，包括添加或移除值时，会向底层映射发送更改事件。

注意，`MutableSet` **不会**对集合内部的*值本身*应用可变跟踪。因此，它不是跟踪对*递归*可变结构进行深层更改的足够解决方案。为了支持这种用例，构建一个`MutableSet`的子类，提供适当的强制转换，以便放置在字典中的值也是“可变的”，并向它们的父结构发出事件。

另请参阅

`MutableDict`

`MutableList`

**成员**

add(), clear(), coerce(), difference_update(), discard(), intersection_update(), pop(), remove(), symmetric_difference_update(), update()

**类签名**

类`sqlalchemy.ext.mutable.MutableSet`（`sqlalchemy.ext.mutable.Mutable`, `builtins.set`, `typing.Generic`)

```py
method add(elem: _T) → None
```

向集合添加一个元素。

如果元素已经存在，则不产生任何效果。

```py
method clear() → None
```

从此集合中移除所有元素。

```py
classmethod coerce(index: str, value: Any) → MutableSet[_T] | None
```

将普通集合转换为此类的实例。

```py
method difference_update(*arg: Iterable[Any]) → None
```

从此集合中移除另一个集合的所有元素。

```py
method discard(elem: _T) → None
```

如果元素是集合的成员，则从集合中移除一个元素。

如果元素不是成员，则不执行任何操作。

```py
method intersection_update(*arg: Iterable[Any]) → None
```

使用自身和另一个集合的交集更新集合。

```py
method pop(*arg: Any) → _T
```

移除并返回任意集合元素。如果集合为空，则引发 KeyError。

```py
method remove(elem: _T) → None
```

从集合中移除一个元素；它必须是成员。

如果元素不是成员，则引发 KeyError。

```py
method symmetric_difference_update(*arg: Iterable[_T]) → None
```

使用自身和另一个集合的对称差集更新集合。

```py
method update(*arg: Iterable[_T]) → None
```

使用自身和其他集合的并集更新集合。
