# 排序列表

> 原文：[`docs.sqlalchemy.org/en/20/orm/extensions/orderinglist.html`](https://docs.sqlalchemy.org/en/20/orm/extensions/orderinglist.html)

一个管理包含元素的索引/位置信息的自定义列表。

作者：

Jason Kirtland

`orderinglist`是一个用于可变有序关系的辅助程序。它将拦截对由`relationship()`管理的集合执行的列表操作，并自动将列表位置的更改同步到目标标量属性。

示例：一个`slide`表，其中每行引用相关`bullet`表中的零个或多个条目。幻灯片中的子弹根据`bullet`表中`position`列的值按顺序显示。当内存中重新排序条目时，`position`属性的值应更新以反映新的排序顺序：

```py
Base = declarative_base()

class Slide(Base):
    __tablename__ = 'slide'

    id = Column(Integer, primary_key=True)
    name = Column(String)

    bullets = relationship("Bullet", order_by="Bullet.position")

class Bullet(Base):
    __tablename__ = 'bullet'
    id = Column(Integer, primary_key=True)
    slide_id = Column(Integer, ForeignKey('slide.id'))
    position = Column(Integer)
    text = Column(String)
```

标准关系映射将在每个`Slide`上产生一个类似列表的属性，其中包含所有相关的`Bullet`对象，但无法自动处理顺序变化。将`Bullet`附加到`Slide.bullets`时，`Bullet.position`属性将保持未设置状态，直到手动分配。当`Bullet`插入列表中间时，后续的`Bullet`对象也需要重新编号。

`OrderingList`对象自动化此任务，管理集合中所有`Bullet`对象的`position`属性。它是使用`ordering_list()`工厂构建的：

```py
from sqlalchemy.ext.orderinglist import ordering_list

Base = declarative_base()

class Slide(Base):
    __tablename__ = 'slide'

    id = Column(Integer, primary_key=True)
    name = Column(String)

    bullets = relationship("Bullet", order_by="Bullet.position",
                            collection_class=ordering_list('position'))

class Bullet(Base):
    __tablename__ = 'bullet'
    id = Column(Integer, primary_key=True)
    slide_id = Column(Integer, ForeignKey('slide.id'))
    position = Column(Integer)
    text = Column(String)
```

使用上述映射，`Bullet.position`属性被管理：

```py
s = Slide()
s.bullets.append(Bullet())
s.bullets.append(Bullet())
s.bullets[1].position
>>> 1
s.bullets.insert(1, Bullet())
s.bullets[2].position
>>> 2
```

`OrderingList`构造仅适用于对集合的**更改**，而不是从数据库的初始加载，并要求在加载时对列表进行排序。因此，请确保在针对目标排序属性的`relationship()`上指定`order_by`，以便在首次加载时排序正确。

警告

当主键列或唯一列是排序的目标时，`OrderingList`在功能上提供的功能有限。不支持或存在问题的操作包括：

> +   两个条目必须交换值。在主键或唯一约束的情况下，这不受直接支持，因为这意味着至少需要先暂时删除一行，或者在交换发生时将其更改为第三个中性值。
> +   
> +   必须删除一个条目以为新条目腾出位置。SQLAlchemy 的工作单元在单次刷新中执行所有 INSERT 操作，然后再执行 DELETE 操作。在主键的情况下，它将交换相同主键的 INSERT/DELETE 以减轻此限制的影响，但对于唯一列不会发生这种情况。未来的功能将允许“DELETE before INSERT”行为成为可能，从而减轻此限制，但此功能将需要在映射器级别对要以这种方式处理的列集进行显式配置。

`ordering_list()`接受相关对象的排序属性名称作为参数。默认情况下，对象在`ordering_list()`中的位置与排序属性同步：索引 0 将获得位置 0，索引 1 位置 1，依此类推。要从 1 或其他整数开始编号，请提供`count_from=1`。

## API 参考

| 对象名称 | 描述 |
| --- | --- |
| count_from_0(index, collection) | 编号函数：从 0 开始的连续整数。 |
| count_from_1(index, collection) | 编号函数：从 1 开始的连续整数。 |
| count_from_n_factory(start) | 编号函数：从任意起始位置开始的连续整数。 |
| ordering_list(attr[, count_from, ordering_func, reorder_on_append]) | 为映射器定义准备一个`OrderingList`工厂。 |
| OrderingList | 一个自定义列表，用于管理其子项的位置信息。 |

```py
function sqlalchemy.ext.orderinglist.ordering_list(attr: str, count_from: int | None = None, ordering_func: Callable[[int, Sequence[_T]], int] | None = None, reorder_on_append: bool = False) → Callable[[], OrderingList]
```

准备一个`OrderingList`工厂，用于在映射器定义中使用。

返回一个适用于作为 Mapper 关系的`collection_class`选项参数的对象。例如：

```py
from sqlalchemy.ext.orderinglist import ordering_list

class Slide(Base):
    __tablename__ = 'slide'

    id = Column(Integer, primary_key=True)
    name = Column(String)

    bullets = relationship("Bullet", order_by="Bullet.position",
                            collection_class=ordering_list('position'))
```

参数：

+   `attr` – 用于存储和检索排序信息的映射属性的名称

+   `count_from` – 设置从`count_from`开始的基于整数的排序。例如，`ordering_list('pos', count_from=1)`将在 SQL 中创建一个基于 1 的列表，将值存储在‘pos’列中。如果提供了`ordering_func`，则会被忽略。

额外的参数传递给`OrderingList`构造函数。

```py
function sqlalchemy.ext.orderinglist.count_from_0(index, collection)
```

编号函数：从 0 开始的连续整数。

```py
function sqlalchemy.ext.orderinglist.count_from_1(index, collection)
```

编号函数：从 1 开始的连续整数。

```py
function sqlalchemy.ext.orderinglist.count_from_n_factory(start)
```

编号函数：从任意起始位置开始的连续整数。

```py
class sqlalchemy.ext.orderinglist.OrderingList
```

一个自定义列表，用于管理其子项的位置信息。

`OrderingList` 对象通常使用 `ordering_list()` 工厂函数设置，与 `relationship()` 函数结合使用。 

**成员**

__init__(), append(), insert(), pop(), remove(), reorder()

**类签名**

类 `sqlalchemy.ext.orderinglist.OrderingList` (`builtins.list`, `typing.Generic`)

```py
method __init__(ordering_attr: str | None = None, ordering_func: Callable[[int, Sequence[_T]], int] | None = None, reorder_on_append: bool = False)
```

一个自定义列表，用于管理其子项的位置信息。

`OrderingList` 是一个 `collection_class` 列表实现，它将 Python 列表中的位置与映射对象上的位置属性同步。

此实现依赖于列表以正确的顺序开始，因此一定要 **确保** 在关系上放置一个 `order_by`。

参数：

+   `ordering_attr` – 存储对象在关系中顺序的属性名称。

+   `ordering_func` –

    可选。将 Python 列表中的位置映射到存储在 `ordering_attr` 中的值的函数。返回的值通常（但不必！）是整数。

    `ordering_func` 被调用时具有两个位置参数：列表中元素的索引和列表本身。

    如果省略，将使用 Python 列表索引作为属性值。此模块提供了两个基本的预定义编号函数：`count_from_0` 和 `count_from_1`。有关诸如步进编号、字母编号和斐波那契编号等更奇特的示例，请参阅单元测试。

+   `reorder_on_append` –

    默认为 False。当追加一个具有现有（非 None）排序值的对象时，该值将保持不变，除非 `reorder_on_append` 为真。这是一种优化，可以避免各种危险的意外数据库写入。

    SQLAlchemy 将在对象加载时通过 append() 将实例添加到列表中。如果由于某种原因数据库的结果集跳过了排序步骤（例如，行 '1' 缺失，但你得到了 '2'、'3' 和 '4'），那么 reorder_on_append=True 将立即重新编号为 '1'、'2'、'3'。如果有多个会话进行更改，其中任何一个会话恰巧加载了这个集合，即使是临时加载，所有会话都会尝试在它们的提交中“清理”编号，可能会导致除一个之外的所有会话都以并发修改错误失败。

    建议保留默认值为 False，如果你在之前已经排序过的实例上进行`append()`操作，或者在手动执行 SQL 操作后进行一些清理工作时，只需调用`reorder()`即可。

```py
method append(entity)
```

将对象追加到列表的末尾。

```py
method insert(index, entity)
```

在索引之前插入对象。

```py
method pop(index=-1)
```

移除并返回索引处的项目（默认为最后一个）。

如果列表为空或索引超出范围，则引发 IndexError。

```py
method remove(entity)
```

移除第一次出现的值。

如果值不存在则引发 ValueError。

```py
method reorder() → None
```

同步整个集合的排序。

扫描列表并确保每个对象设置了准确的排序信息。

## API 参考

| 对象名称 | 描述 |
| --- | --- |
| count_from_0(index, collection) | 编号函数：从 0 开始的连续整数。 |
| count_from_1(index, collection) | 编号函数：从 1 开始的连续整数。 |
| count_from_n_factory(start) | 编号函数：从任意起始值开始的连续整数。 |
| ordering_list(attr[, count_from, ordering_func, reorder_on_append]) | 为映射器定义准备一个`OrderingList`工厂。 |
| OrderingList | 一个自定义列表，管理其子项的位置信息。 |

```py
function sqlalchemy.ext.orderinglist.ordering_list(attr: str, count_from: int | None = None, ordering_func: Callable[[int, Sequence[_T]], int] | None = None, reorder_on_append: bool = False) → Callable[[], OrderingList]
```

为映射器定义准备一个`OrderingList`工厂。

返回一个适合用作 Mapper 关系的`collection_class`选项参数的对象。例如：

```py
from sqlalchemy.ext.orderinglist import ordering_list

class Slide(Base):
    __tablename__ = 'slide'

    id = Column(Integer, primary_key=True)
    name = Column(String)

    bullets = relationship("Bullet", order_by="Bullet.position",
                            collection_class=ordering_list('position'))
```

参数：

+   `attr` – 用于存储和检索排序信息的映射属性的名称

+   `count_from` – 设置基于整数的排序，从`count_from`开始。例如，`ordering_list('pos', count_from=1)`将在 SQL 中创建一个以 1 为基础的列表，在‘pos’列中存储值。如果提供了`ordering_func`，则忽略。

额外的参数将传递给`OrderingList`构造函数。

```py
function sqlalchemy.ext.orderinglist.count_from_0(index, collection)
```

编号函数：从 0 开始的连续整数。

```py
function sqlalchemy.ext.orderinglist.count_from_1(index, collection)
```

编号函数：从 1 开始的连续整数。

```py
function sqlalchemy.ext.orderinglist.count_from_n_factory(start)
```

编号函数：从任意起始值开始的连续整数。

```py
class sqlalchemy.ext.orderinglist.OrderingList
```

一个自定义列表，管理其子项的位置信息。

`OrderingList`对象通常使用与`relationship()`函数配合使用的`ordering_list()`工厂函数设置。

**成员**

__init__(), append(), insert(), pop(), remove(), reorder()

**类签名**

类 `sqlalchemy.ext.orderinglist.OrderingList` (`builtins.list`, `typing.Generic`)

```py
method __init__(ordering_attr: str | None = None, ordering_func: Callable[[int, Sequence[_T]], int] | None = None, reorder_on_append: bool = False)
```

一个自定义列表，用于管理其子项的位置信息。

`OrderingList` 是一个 `collection_class` 列表实现，将 Python 列表中的位置与映射对象上的位置属性同步。

此实现依赖于列表以正确顺序开始，因此请务必在关系上放置 `order_by`。

参数:

+   `ordering_attr` – 存储对象在关系中顺序的属性名称。

+   `ordering_func` –

    可选。将 Python 列表��的位置映射到存储在 `ordering_attr` 中的值的函数。通常返回的值是整数（但不一定是！）。

    `ordering_func` 被调用时带有两个位置参数：列表中元素的索引和列表本身。

    如果省略，则使用 Python 列表索引作为属性值。本模块提供了两个基本的预构建编号函数：`count_from_0` 和 `count_from_1`。有关更奇特的示例，如步进编号、字母和斐波那契编号，请参见单元测试。

+   `reorder_on_append` –

    默认为 False。在附加具有现有（非 None）排序值的对象时，该值将保持不变，除非 `reorder_on_append` 为 true。这是一种优化，可避免各种危险的意外数据库写入。

    当您的对象加载时，SQLAlchemy 将通过 append() 将实例添加到列表中。如果由于某种原因数据库的结果集跳过了排序步骤（例如，行‘1’丢失，但您得到‘2’、‘3’和‘4’），reorder_on_append=True 将立即重新编号项目为‘1’、‘2’、‘3’。如果有多个会话进行更改，其中任何一个碰巧加载此集合，即使是临时加载，所有会话都会尝试在其提交中“清理”编号，可能导致除一个之外的所有会话都因并发修改错误而失败。

    建议保持默认值为 False，并在对先前有序实例进行 `append()` 操作或在手动 sql 操作后进行一些清理时，只调用 `reorder()`。

```py
method append(entity)
```

将对象追加到列表末尾。

```py
method insert(index, entity)
```

在索引之前插入对象。

```py
method pop(index=-1)
```

移除并返回索引处的项目（默认为最后一个）。

如果列表为空或索引超出范围，则引发 IndexError。

```py
method remove(entity)
```

移除第一次出现的值。

如果值不存在，则引发 ValueError。

```py
method reorder() → None
```

同步整个集合的排序。

扫描列表并确保每个对象具有准确的排序信息设置。
