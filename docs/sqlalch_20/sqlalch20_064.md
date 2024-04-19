# 关联代理

> 原文：[`docs.sqlalchemy.org/en/20/orm/extensions/associationproxy.html`](https://docs.sqlalchemy.org/en/20/orm/extensions/associationproxy.html)

`associationproxy` 用于在关系之间创建目标属性的读/写视图。它实质上隐藏了两个端点之间的“中间”属性的使用，并且可以用于从相关对象的集合或标量关系中挑选字段，或者减少使用关联对象模式时的冗长性。创造性地应用，关联代理允许构建复杂的集合和字典视图，几乎可以对任何几何形状进行持久化，使用标准、透明配置的关系模式保存到数据库中。

## 简化标量集合

考虑两个类`User`和`Keyword`之间的多对多映射。每个`User`可以拥有任意数量的`Keyword`对象，反之亦然（多对多模式在 Many To Many 中有描述）。下面的示例以相同的方式说明了这种模式，唯一的区别是在`User`类中添加了一个额外的属性`User.keywords`：

```py
from __future__ import annotations

from typing import Final
from typing import List

from sqlalchemy import Column
from sqlalchemy import ForeignKey
from sqlalchemy import Integer
from sqlalchemy import String
from sqlalchemy import Table
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship
from sqlalchemy.ext.associationproxy import association_proxy
from sqlalchemy.ext.associationproxy import AssociationProxy

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "user"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(64))
    kw: Mapped[List[Keyword]] = relationship(secondary=lambda: user_keyword_table)

    def __init__(self, name: str):
        self.name = name

    # proxy the 'keyword' attribute from the 'kw' relationship
    keywords: AssociationProxy[List[str]] = association_proxy("kw", "keyword")

class Keyword(Base):
    __tablename__ = "keyword"
    id: Mapped[int] = mapped_column(primary_key=True)
    keyword: Mapped[str] = mapped_column(String(64))

    def __init__(self, keyword: str):
        self.keyword = keyword

user_keyword_table: Final[Table] = Table(
    "user_keyword",
    Base.metadata,
    Column("user_id", Integer, ForeignKey("user.id"), primary_key=True),
    Column("keyword_id", Integer, ForeignKey("keyword.id"), primary_key=True),
)
```

在上面的示例中，`association_proxy()`应用于`User`类，以生成`kw`关系的“视图”，该视图公开与每个`Keyword`对象关联的`.keyword`的字符串值。当向集合添加字符串时，它还会透明地创建新的`Keyword`对象：

```py
>>> user = User("jek")
>>> user.keywords.append("cheese-inspector")
>>> user.keywords.append("snack-ninja")
>>> print(user.keywords)
['cheese-inspector', 'snack-ninja']
```

要理解这一点的机制，首先回顾一下在不使用`.keywords`关联代理的情况下，`User`和`Keyword`的行为。通常，阅读和操作与`User`关联的“keyword”字符串集合需要从每个集合元素遍历到`.keyword`属性，这可能很麻烦。下面的示例说明了在不使用关联代理的情况下应用相同系列操作的情况：

```py
>>> # identical operations without using the association proxy
>>> user = User("jek")
>>> user.kw.append(Keyword("cheese-inspector"))
>>> user.kw.append(Keyword("snack-ninja"))
>>> print([keyword.keyword for keyword in user.kw])
['cheese-inspector', 'snack-ninja']
```

由`association_proxy()`函数生成的`AssociationProxy`对象是[Python 描述符](https://docs.python.org/howto/descriptor.html)的一个实例，并且不被`Mapper`以任何方式视为“映射”。因此，无论是使用声明式还是命令式映射，它始终内联显示在映射类的类定义中。

代理通过对操作响应中的底层映射属性或集合进行操作，通过代理进行的更改立即反映在映射属性中，反之亦然。底层属性仍然完全可访问。

当第一次访问时，关联代理对目标集合执行内省操作，以便其行为正确对应。细节，例如本地代理的属性是否是集合（通常情况下是这样）或标量引用，以及集合是否像集合、列表或字典一样行事，都被考虑在内，这样代理应该像底层集合或属性一样行事。

### 创造新价值

当一个列表 `append()` 事件（或集合 `add()`，字典 `__setitem__()`，或标量赋值事件）被关联代理拦截时，它使用它的构造函数实例化一个新的“中间”对象实例，将给定值作为单个参数传递。在我们上面的示例中，一个像这样的操作：

```py
user.keywords.append("cheese-inspector")
```

被关联代理翻译成以下操作：

```py
user.kw.append(Keyword("cheese-inspector"))
```

此示例有效的原因在于我们已经设计了 `Keyword` 的构造函数以接受一个单一的位置参数 `keyword`。对于那些不可行的单参数构造函数的情况，关联代理的创建行为可以使用 `association_proxy.creator` 参数进行自定义，该参数引用了一个可调用对象（即 Python 函数），该对象将根据单一参数产生一个新的对象实例。下面我们使用一个典型的 lambda 函数来说明这一点：

```py
class User(Base):
    ...

    # use Keyword(keyword=kw) on append() events
    keywords: AssociationProxy[List[str]] = association_proxy(
        "kw", "keyword", creator=lambda kw: Keyword(keyword=kw)
    )
```

当一个基于列表或集合的集合，或者是一个标量属性的情况下，`creator` 函数接受一个参数。在基于字典的集合的情况下，它接受两个参数，“key” 和 “value”。下面是一个示例，代理到基于字典的集合。

## 简化关联对象

“关联对象”模式是一种扩展形式的多对多关系，并在关联对象中描述。关联代理对于在常规使用过程中使“关联对象”保持不受干扰是很有用的。

假设我们上面的 `user_keyword` 表有额外的列，我们希望显式地映射这些列，但在大多数情况下，我们不需要直接访问这些属性。下面，我们展示了一个新的映射，介绍了 `UserKeywordAssociation` 类，它映射到上面展示的 `user_keyword` 表。这个类添加了一个额外的列 `special_key`，这个值我们偶尔想要访问，但不是在通常情况下。我们在 `User` 类上创建了一个名为 `keywords` 的关联代理，它将弥合从 `User` 的 `user_keyword_associations` 集合到每个 `UserKeywordAssociation` 上的 `.keyword` 属性的差距：

```py
from __future__ import annotations

from typing import List
from typing import Optional

from sqlalchemy import ForeignKey
from sqlalchemy import String
from sqlalchemy.ext.associationproxy import association_proxy
from sqlalchemy.ext.associationproxy import AssociationProxy
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "user"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(64))

    user_keyword_associations: Mapped[List[UserKeywordAssociation]] = relationship(
        back_populates="user",
        cascade="all, delete-orphan",
    )

    # association proxy of "user_keyword_associations" collection
    # to "keyword" attribute
    keywords: AssociationProxy[List[Keyword]] = association_proxy(
        "user_keyword_associations",
        "keyword",
        creator=lambda keyword_obj: UserKeywordAssociation(keyword=keyword_obj),
    )

    def __init__(self, name: str):
        self.name = name

class UserKeywordAssociation(Base):
    __tablename__ = "user_keyword"
    user_id: Mapped[int] = mapped_column(ForeignKey("user.id"), primary_key=True)
    keyword_id: Mapped[int] = mapped_column(ForeignKey("keyword.id"), primary_key=True)
    special_key: Mapped[Optional[str]] = mapped_column(String(50))

    user: Mapped[User] = relationship(back_populates="user_keyword_associations")

    keyword: Mapped[Keyword] = relationship()

class Keyword(Base):
    __tablename__ = "keyword"
    id: Mapped[int] = mapped_column(primary_key=True)
    keyword: Mapped[str] = mapped_column("keyword", String(64))

    def __init__(self, keyword: str):
        self.keyword = keyword

    def __repr__(self) -> str:
        return f"Keyword({self.keyword!r})"
```

通过上述配置，我们可以操作每个 `User` 对象的 `.keywords` 集合，其中每个对象都公开了从底层 `UserKeywordAssociation` 元素获取的 `Keyword` 对象集合：

```py
>>> user = User("log")
>>> for kw in (Keyword("new_from_blammo"), Keyword("its_big")):
...     user.keywords.append(kw)
>>> print(user.keywords)
[Keyword('new_from_blammo'), Keyword('its_big')]
```

此示例与先前在 Simplifying Scalar Collections 中说明的示例形成对比，后者的关联代理公开了一个字符串集合，而不是一个组合对象集合。在这种情况下，每个`.keywords.append()`操作等效于：

```py
>>> user.user_keyword_associations.append(
...     UserKeywordAssociation(keyword=Keyword("its_heavy"))
... )
```

`UserKeywordAssociation`对象有两个属性，两者都在关联代理的`append()`操作的范围内填充；`.keyword`指的是`Keyword`对象，`.user`指的是`User`对象。`.keyword`属性首先填充，因为关联代理响应于`.append()`操作生成新的`UserKeywordAssociation`对象，将给定的`Keyword`实例分配给`.keyword`属性。然后，随着`UserKeywordAssociation`对象附加到`User.user_keyword_associations`集合，针对`User.user_keyword_associations`的`back_populates`配置的`UserKeywordAssociation.user`属性在给定的`UserKeywordAssociation`实例上进行初始化，以指向接收附加操作的父`User`。上面的`special_key`参数保持其默认值为`None`。

对于那些我们确实希望`special_key`具有值的情况，我们明确创建`UserKeywordAssociation`对象。在下面，我们分配了所有三个属性，其中在构造过程中分配`.user`的效果是将新的`UserKeywordAssociation`附加到`User.user_keyword_associations`集合（通过关系）：

```py
>>> UserKeywordAssociation(
...     keyword=Keyword("its_wood"), user=user, special_key="my special key"
... )
```

关联代理将以这些操作代表的`Keyword`对象集合返回给我们：

```py
>>> print(user.keywords)
[Keyword('new_from_blammo'), Keyword('its_big'), Keyword('its_heavy'), Keyword('its_wood')]
```

## 代理到基于字典的集合

关联代理也可以代理到基于字典的集合。SQLAlchemy 映射通常使用`attribute_keyed_dict()`集合类型来创建字典集合，以及 Custom Dictionary-Based Collections 中描述的扩展技术。

当检测到基于字典的集合的使用时，关联代理会调整其行为。当新值添加到字典时，关联代理通过将两个参数传递给创建函数来实例化中间对象，而不是一个，即键和值。与往常一样，此创建函数默认为中间类的构造函数，并且可以使用`creator`参数进行定制。

下面，我们修改了我们的`UserKeywordAssociation`示例，以便`User.user_keyword_associations`集合现在将使用字典映射，其中`UserKeywordAssociation.special_key`参数将用作字典的键。我们还对`User.keywords`代理应用了一个`creator`参数，以便在将新元素添加到字典时适当地分配这些值：

```py
from __future__ import annotations
from typing import Dict

from sqlalchemy import ForeignKey
from sqlalchemy import String
from sqlalchemy.ext.associationproxy import association_proxy
from sqlalchemy.ext.associationproxy import AssociationProxy
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship
from sqlalchemy.orm.collections import attribute_keyed_dict

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "user"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(64))

    # user/user_keyword_associations relationship, mapping
    # user_keyword_associations with a dictionary against "special_key" as key.
    user_keyword_associations: Mapped[Dict[str, UserKeywordAssociation]] = relationship(
        back_populates="user",
        collection_class=attribute_keyed_dict("special_key"),
        cascade="all, delete-orphan",
    )
    # proxy to 'user_keyword_associations', instantiating
    # UserKeywordAssociation assigning the new key to 'special_key',
    # values to 'keyword'.
    keywords: AssociationProxy[Dict[str, Keyword]] = association_proxy(
        "user_keyword_associations",
        "keyword",
        creator=lambda k, v: UserKeywordAssociation(special_key=k, keyword=v),
    )

    def __init__(self, name: str):
        self.name = name

class UserKeywordAssociation(Base):
    __tablename__ = "user_keyword"
    user_id: Mapped[int] = mapped_column(ForeignKey("user.id"), primary_key=True)
    keyword_id: Mapped[int] = mapped_column(ForeignKey("keyword.id"), primary_key=True)
    special_key: Mapped[str]

    user: Mapped[User] = relationship(
        back_populates="user_keyword_associations",
    )
    keyword: Mapped[Keyword] = relationship()

class Keyword(Base):
    __tablename__ = "keyword"
    id: Mapped[int] = mapped_column(primary_key=True)
    keyword: Mapped[str] = mapped_column(String(64))

    def __init__(self, keyword: str):
        self.keyword = keyword

    def __repr__(self) -> str:
        return f"Keyword({self.keyword!r})"
```

我们将`.keywords`集合表示为一个字典，将`UserKeywordAssociation.special_key`值映射到`Keyword`对象：

```py
>>> user = User("log")

>>> user.keywords["sk1"] = Keyword("kw1")
>>> user.keywords["sk2"] = Keyword("kw2")

>>> print(user.keywords)
{'sk1': Keyword('kw1'), 'sk2': Keyword('kw2')}
```  ## 复合关联代理

鉴于我们之前关于从关系到标量属性的代理、跨关联对象的代理以及代理字典的示例，我们可以将所有三种技术结合起来，为`User`提供一个严格处理`special_key`字符串值映射到`keyword`字符串的`keywords`字典。`UserKeywordAssociation`和`Keyword`类完全被隐藏。这是通过在`User`上建立一个关联到`UserKeywordAssociation`上存在的关联代理来实现的：

```py
from __future__ import annotations

from sqlalchemy import ForeignKey
from sqlalchemy import String
from sqlalchemy.ext.associationproxy import association_proxy
from sqlalchemy.ext.associationproxy import AssociationProxy
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship
from sqlalchemy.orm.collections import attribute_keyed_dict

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "user"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(64))

    user_keyword_associations: Mapped[Dict[str, UserKeywordAssociation]] = relationship(
        back_populates="user",
        collection_class=attribute_keyed_dict("special_key"),
        cascade="all, delete-orphan",
    )
    # the same 'user_keyword_associations'->'keyword' proxy as in
    # the basic dictionary example.
    keywords: AssociationProxy[Dict[str, str]] = association_proxy(
        "user_keyword_associations",
        "keyword",
        creator=lambda k, v: UserKeywordAssociation(special_key=k, keyword=v),
    )

    def __init__(self, name: str):
        self.name = name

class UserKeywordAssociation(Base):
    __tablename__ = "user_keyword"
    user_id: Mapped[int] = mapped_column(ForeignKey("user.id"), primary_key=True)
    keyword_id: Mapped[int] = mapped_column(ForeignKey("keyword.id"), primary_key=True)
    special_key: Mapped[str] = mapped_column(String(64))
    user: Mapped[User] = relationship(
        back_populates="user_keyword_associations",
    )

    # the relationship to Keyword is now called
    # 'kw'
    kw: Mapped[Keyword] = relationship()

    # 'keyword' is changed to be a proxy to the
    # 'keyword' attribute of 'Keyword'
    keyword: AssociationProxy[Dict[str, str]] = association_proxy("kw", "keyword")

class Keyword(Base):
    __tablename__ = "keyword"
    id: Mapped[int] = mapped_column(primary_key=True)
    keyword: Mapped[str] = mapped_column(String(64))

    def __init__(self, keyword: str):
        self.keyword = keyword
```

`User.keywords` 现在是一个字符串到字符串的字典，`UserKeywordAssociation`和`Keyword`对象会透明地为我们创建和删除，使用关联代理。在下面的示例中，我们展示了赋值运算符的使用，也由关联代理适当处理，一次将字典值应用到集合中：

```py
>>> user = User("log")
>>> user.keywords = {"sk1": "kw1", "sk2": "kw2"}
>>> print(user.keywords)
{'sk1': 'kw1', 'sk2': 'kw2'}

>>> user.keywords["sk3"] = "kw3"
>>> del user.keywords["sk2"]
>>> print(user.keywords)
{'sk1': 'kw1', 'sk3': 'kw3'}

>>> # illustrate un-proxied usage
... print(user.user_keyword_associations["sk3"].kw)
<__main__.Keyword object at 0x12ceb90>
```

我们上面示例的一个注意事项是，由于为每个字典设置操作创建了`Keyword`对象，所以示例未能保持`Keyword`对象在其字符串名称上的唯一性，这是像这种标记场景的典型要求。对于这种用例，建议使用[UniqueObject](https://www.sqlalchemy.org/trac/wiki/UsageRecipes/UniqueObject)这个配方，或类似的创建策略，它将应用“先查找，然后创建”策略到`Keyword`类的构造函数，以便如果给定名称已经存在，则返回已经存在的`Keyword`。

## 使用关联代理进行查询

`AssociationProxy` 具有简单的 SQL 构建能力，类似于其他 ORM 映射属性在类级别上的工作方式，并且基本上基于 SQL `EXISTS` 关键字提供了基本的过滤支持。

注意

关联代理扩展的主要目的是允许改进已加载的映射对象实例的持久性和对象访问模式。类绑定查询功能的用途有限，并且不会取代在构建具有 JOINs、急加载选项等 SQL 查询时需要引用底层属性的需求。

对于本节，假设一个类既有一个关联到列的关联代理，又有一个关联到相关对象的关联代理，就像下面的映射示例一样：

```py
from __future__ import annotations
from sqlalchemy import Column, ForeignKey, Integer, String
from sqlalchemy.ext.associationproxy import association_proxy, AssociationProxy
from sqlalchemy.orm import DeclarativeBase, relationship
from sqlalchemy.orm.collections import attribute_keyed_dict
from sqlalchemy.orm.collections import Mapped

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "user"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(64))

    user_keyword_associations: Mapped[UserKeywordAssociation] = relationship(
        cascade="all, delete-orphan",
    )

    # object-targeted association proxy
    keywords: AssociationProxy[List[Keyword]] = association_proxy(
        "user_keyword_associations",
        "keyword",
    )

    # column-targeted association proxy
    special_keys: AssociationProxy[List[str]] = association_proxy(
        "user_keyword_associations", "special_key"
    )

class UserKeywordAssociation(Base):
    __tablename__ = "user_keyword"
    user_id: Mapped[int] = mapped_column(ForeignKey("user.id"), primary_key=True)
    keyword_id: Mapped[int] = mapped_column(ForeignKey("keyword.id"), primary_key=True)
    special_key: Mapped[str] = mapped_column(String(64))
    keyword: Mapped[Keyword] = relationship()

class Keyword(Base):
    __tablename__ = "keyword"
    id: Mapped[int] = mapped_column(primary_key=True)
    keyword: Mapped[str] = mapped_column(String(64))
```

生成的 SQL 采用了针对 EXISTS SQL 操作符的相关子查询的形式，以便在不需要对封闭查询进行额外修改的情况下在 WHERE 子句中使用。如果关联代理的直接目标是一个**映射的列表达式**，则可以使用标准列操作符，这些操作符将嵌入到子查询中。例如，一个直接的等式操作符：

```py
>>> print(session.scalars(select(User).where(User.special_keys == "jek")))
SELECT  "user".id  AS  user_id,  "user".name  AS  user_name
FROM  "user"
WHERE  EXISTS  (SELECT  1
FROM  user_keyword
WHERE  "user".id  =  user_keyword.user_id  AND  user_keyword.special_key  =  :special_key_1) 
```

一个 LIKE 操作符：

```py
>>> print(session.scalars(select(User).where(User.special_keys.like("%jek"))))
SELECT  "user".id  AS  user_id,  "user".name  AS  user_name
FROM  "user"
WHERE  EXISTS  (SELECT  1
FROM  user_keyword
WHERE  "user".id  =  user_keyword.user_id  AND  user_keyword.special_key  LIKE  :special_key_1) 
```

对于关联代理，其直接目标是**相关对象或集合，或相关对象上的另一个关联代理或属性**的情况，可以使用关系导向操作符，如 `PropComparator.has()` 和 `PropComparator.any()`。`User.keywords` 属性实际上是两个链接在一起的关联代理，因此在使用此代理生成 SQL 短语时，我们得到两个级别的 EXISTS 子查询：

```py
>>> print(session.scalars(select(User).where(User.keywords.any(Keyword.keyword == "jek"))))
SELECT  "user".id  AS  user_id,  "user".name  AS  user_name
FROM  "user"
WHERE  EXISTS  (SELECT  1
FROM  user_keyword
WHERE  "user".id  =  user_keyword.user_id  AND  (EXISTS  (SELECT  1
FROM  keyword
WHERE  keyword.id  =  user_keyword.keyword_id  AND  keyword.keyword  =  :keyword_1))) 
```

这不是 SQL 的最有效形式，因此，虽然关联代理可以方便快速生成 WHERE 条件，但应该检查 SQL 结果，并将其“展开”为明确的 JOIN 条件以获得最佳效果，特别是在将关联代理链接在一起时。

在 1.3 版更改：根据目标类型，关联代理功能提供了不同的查询模式。请参阅 关联代理现在为面向列的目标提供标准列操作符。

## 级联标量删除

1.3 版中的新功能。

给定一个映射如下：

```py
from __future__ import annotations
from sqlalchemy import Column, ForeignKey, Integer, String
from sqlalchemy.ext.associationproxy import association_proxy, AssociationProxy
from sqlalchemy.orm import DeclarativeBase, relationship
from sqlalchemy.orm.collections import attribute_keyed_dict
from sqlalchemy.orm.collections import Mapped

class Base(DeclarativeBase):
    pass

class A(Base):
    __tablename__ = "test_a"
    id: Mapped[int] = mapped_column(primary_key=True)
    ab: Mapped[AB] = relationship(uselist=False)
    b: AssociationProxy[B] = association_proxy(
        "ab", "b", creator=lambda b: AB(b=b), cascade_scalar_deletes=True
    )

class B(Base):
    __tablename__ = "test_b"
    id: Mapped[int] = mapped_column(primary_key=True)

class AB(Base):
    __tablename__ = "test_ab"
    a_id: Mapped[int] = mapped_column(ForeignKey(A.id), primary_key=True)
    b_id: Mapped[int] = mapped_column(ForeignKey(B.id), primary_key=True)

    b: Mapped[B] = relationship()
```

对 `A.b` 的赋值将生成一个 `AB` 对象：

```py
a.b = B()
```

`A.b` 关联是标量的，并包括使用参数 `AssociationProxy.cascade_scalar_deletes`。当启用此参数时，将 `A.b` 设置为 `None` 将同时移除 `A.ab`：

```py
a.b = None
assert a.ab is None
```

当未设置 `AssociationProxy.cascade_scalar_deletes` 时，上述关联对象 `a.ab` 将保持不变。

请注意，这不是针对基于集合的关联代理的行为；在这种情况下，当删除代理集合的成员时，中间关联对象总是被移除。是否删除行取决于关系级联设置。

另请参阅

级联

## 标量关系

下面的示例说明了在一对多关系的多方上使用关联代理，访问标量对象的属性：

```py
from __future__ import annotations

from typing import List

from sqlalchemy import ForeignKey
from sqlalchemy import String
from sqlalchemy.ext.associationproxy import association_proxy
from sqlalchemy.ext.associationproxy import AssociationProxy
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

class Recipe(Base):
    __tablename__ = "recipe"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(64))

    steps: Mapped[List[Step]] = relationship(back_populates="recipe")
    step_descriptions: AssociationProxy[List[str]] = association_proxy(
        "steps", "description"
    )

class Step(Base):
    __tablename__ = "step"
    id: Mapped[int] = mapped_column(primary_key=True)
    description: Mapped[str]
    recipe_id: Mapped[int] = mapped_column(ForeignKey("recipe.id"))
    recipe: Mapped[Recipe] = relationship(back_populates="steps")

    recipe_name: AssociationProxy[str] = association_proxy("recipe", "name")

    def __init__(self, description: str) -> None:
        self.description = description

my_snack = Recipe(
    name="afternoon snack",
    step_descriptions=[
        "slice bread",
        "spread peanut butted",
        "eat sandwich",
    ],
)
```

可以使用以下方式打印 `my_snack` 的步骤摘要：

```py
>>> for i, step in enumerate(my_snack.steps, 1):
...     print(f"Step {i} of {step.recipe_name!r}: {step.description}")
Step 1 of 'afternoon snack': slice bread
Step 2 of 'afternoon snack': spread peanut butted
Step 3 of 'afternoon snack': eat sandwich
```

## API 文档

| 对象名称 | 描述 |
| --- | --- |
| association_proxy(target_collection, attr, *, [creator, getset_factory, proxy_factory, proxy_bulk_set, info, cascade_scalar_deletes, create_on_none_assignment, init, repr, default, default_factory, compare, kw_only]) | 返回一个实现对目标属性的视图的 Python 属性，该属性引用目标对象的成员上的属性。 |
| AssociationProxy | 一个描述符，提供对象属性的读写视图。 |
| AssociationProxyExtensionType | 一个枚举。 |
| AssociationProxyInstance | 一个每个类的对象，提供类和对象特定的结果。 |
| ColumnAssociationProxyInstance | 一个`AssociationProxyInstance`，它具有数据库列作为目标。 |
| ObjectAssociationProxyInstance | 一个`AssociationProxyInstance`，它具有一个对象作为目标。 |

```py
function sqlalchemy.ext.associationproxy.association_proxy(target_collection: str, attr: str, *, creator: _CreatorProtocol | None = None, getset_factory: _GetSetFactoryProtocol | None = None, proxy_factory: _ProxyFactoryProtocol | None = None, proxy_bulk_set: _ProxyBulkSetProtocol | None = None, info: _InfoType | None = None, cascade_scalar_deletes: bool = False, create_on_none_assignment: bool = False, init: _NoArg | bool = _NoArg.NO_ARG, repr: _NoArg | bool = _NoArg.NO_ARG, default: Any | None = _NoArg.NO_ARG, default_factory: _NoArg | Callable[[], _T] = _NoArg.NO_ARG, compare: _NoArg | bool = _NoArg.NO_ARG, kw_only: _NoArg | bool = _NoArg.NO_ARG) → AssociationProxy[Any]
```

返回一个实现对目标属性的视图的 Python 属性，该属性引用目标的成员上的属性。

返回的值是一个`AssociationProxy`的实例。

实现一个 Python 属性，表示关系作为一组更简单的值，或标量值。代理属性将模仿目标的集合类型（list、dict 或 set），或者在一对一关系的情况下，一个简单的标量值。

参数：

+   `target_collection` – 立即目标的属性名称。该属性通常由`relationship()`映射，链接到一个目标集合，但也可以是多对一或非标量关系。

+   `attr` – 目标实例或实例上可用的关联实例上的属性。

+   `creator` –

    可选。

    定义添加新项到代理集合时的自定义行为。

    默认情况下，向集合添加新项将触发构造目标对象的实例，将给定的项作为位置参数传递给目标构造函数。对于这些情况不足的情况，`association_proxy.creator`可以提供一个可调用的函数，该函数将以适当的方式构造对象，给定传递的项。

    对于列表和集合导向的集合，将一个参数传递给可调用对象。对于字典导向的集合，将传递两个参数，对应键和值。

    `association_proxy.creator`可调用对象也用于标量（即多对一，一对一）关系。如果目标关系属性的当前值为`None`，则使用可调用对象构造一个新对象。如果对象值已存在，则给定的属性值将填充到该对象上。

    另请参阅

    创建新值

+   `cascade_scalar_deletes` –

    当为 True 时，表示将代理值设置为`None`，或通过`del`删除时，也应删除源对象。仅适用于标量属性。通常，删除代理目标不会删除代理源，因为该对象可能还有其他状态需要保留。

    新版本 1.3 中引入。

    另请参阅

    级联标量删除 - 完整的使用示例

+   `create_on_none_assignment` –

    当为 True 时，表示将代理值设置为`None`时应**创建**源对象（如果不存在），使用创建者。仅适用于标量属性。这与`assocation_proxy.cascade_scalar_deletes`互斥��

    新版本 2.0.18 中引入。

+   `init` –

    特定于声明性数据类映射，指定映射属性是否应作为由数据类过程生成的`__init__()`方法的一部分。

    新版本 2.0.0b4 中引入。

+   `repr` –

    特定于声明性数据类映射，指定由此`AssociationProxy`建立的属性是否应作为由数据类过程生成的`__repr__()`方法的一部分。

    新版本 2.0.0b4 中引入。

+   `default_factory` –

    特定于声明性数据类映射，指定一个默认值生成函数，该函数将作为由数据类过程生成的`__init__()`方法的一部分进行。

    新版本 2.0.0b4 中引入。

+   `compare` –

    特定于声明性数据类映射，指示在为映射类生成`__eq__()`和`__ne__()`方法时，是否应包含此字段进行比较操作。

    新版本 2.0.0b4 中引入。

+   `kw_only` –

    特定于声明性数据类映射，指示在为映射类生成`__init__()`方法时，是否应将此字段标记为仅关键字参数。

    新版本 2.0.0b4 中引入。

+   `info` – 可选，如果存在将分配给`AssociationProxy.info`。

以下附加参数涉及在`AssociationProxy`对象中注入自定义行为，仅供高级使用：

参数：

+   `getset_factory` –

    可选。代理属性访问由根据此代理的 attr 参数获取和设置值的例程自动处理。

    如果您想要自定义此行为，可以提供一个 getset_factory 可调用对象，用于生成 getter 和 setter 函数的元组。工厂函数接受两个参数，即基础集合的抽象类型和此代理实例。

+   `proxy_factory` – 可选。要模拟的集合类型是通过嗅探目标集合确定的。如果您的集合类型无法通过鸭子类型确定，或者您想使用不同的集合实现，可以提供一个工厂函数来生成这些集合。仅适用于非标量关系。

+   `proxy_bulk_set` – 可选，与 proxy_factory 一起使用。

```py
class sqlalchemy.ext.associationproxy.AssociationProxy
```

一个呈现对象属性的读/写视图的描述符。

**成员**

__init__(), cascade_scalar_deletes, create_on_none_assignment, creator, extension_type, for_class(), getset_factory, info, is_aliased_class, is_attribute, is_bundle, is_clause_element, is_instance, is_mapper, is_property, is_selectable, key, proxy_bulk_set, proxy_factory, target_collection, value_attr

**类签名**

类 `sqlalchemy.ext.associationproxy.AssociationProxy` (`sqlalchemy.orm.base.InspectionAttrInfo`, `sqlalchemy.orm.base.ORMDescriptor`, `sqlalchemy.orm._DCAttributeOptions`, `sqlalchemy.ext.associationproxy._AssociationProxyProtocol`)

```py
method __init__(target_collection: str, attr: str, *, creator: _CreatorProtocol | None = None, getset_factory: _GetSetFactoryProtocol | None = None, proxy_factory: _ProxyFactoryProtocol | None = None, proxy_bulk_set: _ProxyBulkSetProtocol | None = None, info: _InfoType | None = None, cascade_scalar_deletes: bool = False, create_on_none_assignment: bool = False, attribute_options: _AttributeOptions | None = None)
```

构造一个新的 `AssociationProxy`。

`AssociationProxy` 对象通常使用 `association_proxy()` 构造函数来构建。查看 `association_proxy()` 的描述以获取所有参数的描述。

```py
attribute cascade_scalar_deletes: bool
```

```py
attribute create_on_none_assignment: bool
```

```py
attribute creator: _CreatorProtocol | None
```

```py
attribute extension_type: InspectionAttrExtensionType = 'ASSOCIATION_PROXY'
```

扩展类型，如果有的话。默认为`NotExtension.NOT_EXTENSION`

另请参阅

`HybridExtensionType`

`AssociationProxyExtensionType`

```py
method for_class(class_: Type[Any], obj: object | None = None) → AssociationProxyInstance[_T]
```

返回特定于特定映射类的内部状态。

例如，给定一个类`User`:

```py
class User(Base):
    # ...

    keywords = association_proxy('kws', 'keyword')
```

如果我们从`Mapper.all_orm_descriptors`中访问此`AssociationProxy`，并且我们希望查看由`User`映射的此代理的目标类：

```py
inspect(User).all_orm_descriptors["keywords"].for_class(User).target_class
```

返回一个特定于`User`类的`AssociationProxyInstance`实例。`AssociationProxy`对象保持不对其父类进行操作。

参数:

+   `class_` – 我们正在返回其状态的类。

+   `obj` – 可选，如果属性引用多态目标，则需要类的实例，例如，我们必须查看实际目标对象的类型以获取完整路径。

从版本 1.3 开始新增: - `AssociationProxy`不再存储特定于特定父类的任何状态; 状态现在存储在每个类的`AssociationProxyInstance`对象中。

```py
attribute getset_factory: _GetSetFactoryProtocol | None
```

```py
attribute info
```

*继承自* `InspectionAttrInfo` *的* `InspectionAttrInfo.info` *属性*

与对象相关联的信息字典，允许将用户定义的数据与此`InspectionAttr`相关联。

字典在首次访问时生成。或者，它可以作为`column_property()`、`relationship()`或`composite()`函数的构造函数参数指定。

另请参阅

`QueryableAttribute.info`

`SchemaItem.info`

```py
attribute is_aliased_class = False
```

*继承自* `InspectionAttr` *的* `InspectionAttr.is_aliased_class` *属性*

如果此对象是 `AliasedClass` 的实例，则为真。

```py
attribute is_attribute = True
```

如果此对象是 Python 描述符，则为真。

此处可能指的是许多类型之一。通常是一个 `QueryableAttribute`，它代表 `MapperProperty` 处理属性事件。但也可以是扩展类型，例如 `AssociationProxy` 或 `hybrid_property`。`InspectionAttr.extension_type` 将引用一个常量，用于标识特定的子类型。

另请参阅

`Mapper.all_orm_descriptors`。

```py
attribute is_bundle = False
```

*继承自* `InspectionAttr` *的* `InspectionAttr.is_bundle` *属性*。

如果此对象是 `Bundle` 的实例，则为真。

```py
attribute is_clause_element = False
```

*继承自* `InspectionAttr` *的* `InspectionAttr.is_clause_element` *属性*。

如果此对象是 `ClauseElement` 的实例，则为真。

```py
attribute is_instance = False
```

*继承自* `InspectionAttr` *的* `InspectionAttr.is_instance` *属性*。

如果此对象是 `InstanceState` 的实例，则为真。

```py
attribute is_mapper = False
```

*继承自* `InspectionAttr` *的* `InspectionAttr.is_mapper` *属性*。

如果此对象是 `Mapper` 的实例，则为真。

```py
attribute is_property = False
```

*继承自* `InspectionAttr` *的* `InspectionAttr.is_property` *属性*。

如果此对象是 `MapperProperty` 的实例，则为真。

```py
attribute is_selectable = False
```

*继承自* `InspectionAttr` *的* `InspectionAttr.is_selectable` *属性*。

如果此对象是`Selectable`的实例，则返回 True。

```py
attribute key: str
```

```py
attribute proxy_bulk_set: _ProxyBulkSetProtocol | None
```

```py
attribute proxy_factory: _ProxyFactoryProtocol | None
```

```py
attribute target_collection: str
```

```py
attribute value_attr: str
```

```py
class sqlalchemy.ext.associationproxy.AssociationProxyInstance
```

一个每类对象，用于提供类和对象特定的结果。

当以特定类或类实例的术语调用`AssociationProxy`时，即当它被用作常规 Python 描述符时，就会使用此功能。

当作为普通的 Python 描述符引用`AssociationProxy`时，`AssociationProxyInstance`是实际提供信息的对象。在正常情况下，其存在是透明的：

```py
>>> User.keywords.scalar
False
```

当直接访问`AssociationProxy`对象时，为了获取到`AssociationProxyInstance`的显式句柄，请使用`AssociationProxy.for_class()`方法：

```py
proxy_state = inspect(User).all_orm_descriptors["keywords"].for_class(User)

# view if proxy object is scalar or not
>>> proxy_state.scalar
False
```

在 1.3 版中新增。

**成员**

__eq__(), __le__(), __lt__(), __ne__(), all_(), any(), any_(), asc(), attr, between(), bitwise_and(), bitwise_lshift(), bitwise_not(), bitwise_or(), bitwise_rshift(), bitwise_xor(), bool_op(), collate(), collection_class, concat(), contains(), delete(), desc(), distinct(), endswith(), for_proxy(), get(), has(), icontains(), iendswith(), ilike(), in_(), info, is_(), is_distinct_from(), is_not(), is_not_distinct_from(), isnot(), isnot_distinct_from(), istartswith(), like(), local_attr, match(), not_ilike(), not_in(), not_like(), notilike(), notin_(), notlike(), nulls_first(), nulls_last(), nullsfirst(), nullslast(), op(), operate(), parent, regexp_match(), regexp_replace(), remote_attr, reverse_operate(), scalar, set(), startswith(), target_class, timetuple

**类签名**

类`sqlalchemy.ext.associationproxy.AssociationProxyInstance` (`sqlalchemy.orm.base.SQLORMOperations`)

```py
method __eq__(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators` *的* `sqlalchemy.sql.expression.ColumnOperators.__eq__` *方法。

实现`==`运算符。

在列上下文中，生成子句`a = b`。如果目标是`None`，则生成`a IS NULL`。

```py
method __le__(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators` *的* `sqlalchemy.sql.expression.ColumnOperators.__le__` *方法。

实现`<=`运算符。

在列上下文中，生成子句`a <= b`。

```py
method __lt__(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators` *的* `sqlalchemy.sql.expression.ColumnOperators.__lt__` *方法。

实现`<`运算符。

在列上下文中，生成子句`a < b`。

```py
method __ne__(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators` *的* `sqlalchemy.sql.expression.ColumnOperators.__ne__` *方法。

实现`!=`运算符。

在列上下文中，生成子句`a != b`。如果目标是`None`，则生成`a IS NOT NULL`。

```py
method all_() → ColumnOperators
```

*继承自* `ColumnOperators` *的* `ColumnOperators.all_()` *方法。

对父对象生成一个`all_()`子句。

请查看`all_()`的文档以获取示例。

注意

请确保不要混淆较新的`ColumnOperators.all_()`方法与此方法的**传统**版本，即特定于`ARRAY`的`Comparator.all()`方法，其使用不同的调用风格。

```py
method any(criterion: _ColumnExpressionArgument[bool] | None = None, **kwargs: Any) → ColumnElement[bool]
```

使用 EXISTS 生成一个代理的‘any’表达式。

此表达式将使用基础代理属性的`Comparator.any()`和/或`Comparator.has()`运算符进行组合。

```py
method any_() → ColumnOperators
```

*继承自* `ColumnOperators.any_()` *方法的* `ColumnOperators`

生成针对父对象的`any_()`子句。

请参阅`any_()`文档中的示例。

注意

请务必不要混淆较新的`ColumnOperators.any_()`方法与**旧版**该方法，即特定于`ARRAY`的`Comparator.any()`方法，后者使用了不同的调用风格。

```py
method asc() → ColumnOperators
```

*继承自* `ColumnOperators.asc()` *方法的* `ColumnOperators`

生成针对父对象的`asc()`子句。

```py
attribute attr
```

返回一个元组`(local_attr, remote_attr)`。

该属性最初旨在促进使用`Query.join()`方法一次加入两个关系，但这使用了一个已弃用的调用风格。

要使用`select.join()`或`Query.join()`与关联代理，请当前方法是分别使用`AssociationProxyInstance.local_attr`和`AssociationProxyInstance.remote_attr`属性：

```py
stmt = (
    select(Parent).
    join(Parent.proxied.local_attr).
    join(Parent.proxied.remote_attr)
)
```

未来版本可能会为关联代理属性提供更简洁的连接模式。

参见

`AssociationProxyInstance.local_attr`

`AssociationProxyInstance.remote_attr`

```py
method between(cleft: Any, cright: Any, symmetric: bool = False) → ColumnOperators
```

*继承自* `ColumnOperators.between()` *方法的* `ColumnOperators`

对父对象执行`between()`子句，给定下限和上限范围。

```py
method bitwise_and(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_and()` *方法的* `ColumnOperators`

进行按位与操作，通常通过`&`运算符实现。

版本 2.0.2 中新增。

另请参阅

按位运算符

```py
method bitwise_lshift(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_lshift()` *方法的* `ColumnOperators`

进行按位左移操作，通常通过`<<`运算符实现。

版本 2.0.2 中新增。

另请参阅

按位运算符

```py
method bitwise_not() → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_not()` *方法的* `ColumnOperators`

进行按位非操作，通常通过`~`运算符实现。

版本 2.0.2 中新增。

另请参阅

按位运算符

```py
method bitwise_or(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_or()` *方法的* `ColumnOperators`

进行按位或操作，通常通过`|`运算符实现。

版本 2.0.2 中新增。

另请参阅

按位运算符

```py
method bitwise_rshift(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_rshift()` *方法的* `ColumnOperators`

进行按位右移操作，通常通过`>>`运算符实现。

版本 2.0.2 中新增。

另请参阅

按位运算符

```py
method bitwise_xor(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_xor()` *方法的* `ColumnOperators`

执行按位异或操作，通常通过`^`运算符，或在 PostgreSQL 中使用`#`。

版本 2.0.2 中的新功能。

另请参阅

按位运算符

```py
method bool_op(opstring: str, precedence: int = 0, python_impl: Callable[[...], Any] | None = None) → Callable[[Any], Operators]
```

*继承自* `Operators.bool_op()` *方法的* `Operators`

返回自定义布尔运算符。

此方法是调用`Operators.op()`并传递带有 True 的`Operators.op.is_comparison`标志的简写。 使用`Operators.bool_op()`的一个关键优势是，在使用列构造时，返回表达式的“布尔”性质将出现在[**PEP 484**](https://peps.python.org/pep-0484/)的目的中。

另请参阅

`Operators.op()`

```py
method collate(collation: str) → ColumnOperators
```

*继承自* `ColumnOperators.collate()` *方法的* `ColumnOperators`

生成针对父对象的`collate()`子句，给定排序字符串。

另请参阅

`collate()`

```py
attribute collection_class: Type[Any] | None
```

```py
method concat(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.concat()` *方法的* `ColumnOperators`

实现‘concat’运算符。

在列上下文中，生成子句`a || b`，或在 MySQL 上使用`concat()`运算符。

```py
method contains(other: Any, **kw: Any) → ColumnOperators
```

*从* `ColumnOperators.contains()` *方法继承而来* *的* `ColumnOperators`

实现“包含”操作符。

生成一个对字符串值中间进行匹配的 LIKE 表达式：

```py
column LIKE '%' || <other> || '%'
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.contains("foobar"))
```

由于该操作符使用`LIKE`，因此存在于<other>表达式中的通配符字符`"%"`和`"_"`也将像通配符一样工作。对于字面字符串值，可以将`ColumnOperators.contains.autoescape`标志设置为`True`，以将这些字符的出现进行转义，使它们匹配为自己而不是通配符字符。或者，`ColumnOperators.contains.escape`参数将确定给定字符作为转义字符，这在目标表达式不是字面字符串时很有用。

参数：

+   `other` – 要进行比较的表达式。这通常是一个普通的字符串值，但也可以是任意的 SQL 表达式。除非将`ColumnOperators.contains.autoescape`标志设置为 True，否则不会转义 LIKE 通配符字符`%`和`_`。

+   `autoescape` –

    布尔值；当为 True 时，在 LIKE 表达式中建立一个转义字符，然后将其应用于比较值内的所有`"%"`、`"_"`和转义字符本身的出现，假定比较值为字面字符串而不是 SQL 表达式。

    例如表达式：

    ```py
    somecolumn.contains("foo%bar", autoescape=True)
    ```

    将呈现为：

    ```py
    somecolumn LIKE '%' || :param || '%' ESCAPE '/'
    ```

    其中`:param`的值为`"foo/%bar"`。

+   `escape` –

    一个字符，当给定时，将使用`ESCAPE`关键字将其作为转义字符。然后可以将此字符放在`%`和`_`的前面，以使它们像自己一样工作，而不是通配符字符。

    例如表达式：

    ```py
    somecolumn.contains("foo/%bar", escape="^")
    ```

    将呈现为：

    ```py
    somecolumn LIKE '%' || :param || '%' ESCAPE '^'
    ```

    参数也可以与`ColumnOperators.contains.autoescape`结合使用：

    ```py
    somecolumn.contains("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上面的示例中，给定的字面参数将在传递到数据库之前转换为`"foo^%bar^^bat"`。

参见

`ColumnOperators.startswith()`

`ColumnOperators.endswith()`

`ColumnOperators.like()`

```py
method delete(obj: Any) → None
```

```py
method desc() → ColumnOperators
```

*从* `ColumnOperators.desc()` *方法继承* *自* `ColumnOperators`

对父对象产生一个`desc()`子句。

```py
method distinct() → ColumnOperators
```

*从* `ColumnOperators.distinct()` *方法继承* *自* `ColumnOperators`

对父对象产生一个`distinct()`子句。

```py
method endswith(other: Any, escape: str | None = None, autoescape: bool = False) → ColumnOperators
```

*从* `ColumnOperators.endswith()` *方法继承* *自* `ColumnOperators`

实现‘endswith’操作符。

生成一个对字符串值末尾进行匹配的 LIKE 表达式：

```py
column LIKE '%' || <other>
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.endswith("foobar"))
```

由于操作符使用了`LIKE`，存在于<other>表达式中的通配符字符`"%"`和`"_"`也将像通配符一样起作用。对于文字字符串值，`ColumnOperators.endswith.autoescape`标志可以设置为`True`，以对字符串值中这些字符的出现进行转义，使它们作为自身而不是通配符字符进行匹配。或者，`ColumnOperators.endswith.escape`参数将建立一个给定的字符作为转义字符，当目标表达式不是文字字符串时可能会有用。

参数：

+   `other` – 要比较的表达式。这通常是一个普通的字符串值，但也可以是任意的 SQL 表达式。除非将`ColumnOperators.endswith.autoescape`标志设置为 True，否则不会默认转义 LIKE 通配符`%`和`_`。

+   `autoescape` –

    布尔值；当为 True 时，在 LIKE 表达式中建立一个转义字符，然后将其应用于比较值中所有的`"%"`、`"_"`以及转义字符本身的出现，假定比较值是一个文字字符串而不是 SQL 表达式。

    一个表达式如下：

    ```py
    somecolumn.endswith("foo%bar", autoescape=True)
    ```

    将被渲染为：

    ```py
    somecolumn LIKE '%' || :param ESCAPE '/'
    ```

    其中`:param`的值为`"foo/%bar"`。

+   `escape` –

    一个字符，当给定时将会使用`ESCAPE`关键字来确定该字符作为转义字符。然后可以将此字符放在`%`和`_`的前面，以允许它们作为自身而不是通配符字符。

    一个表达式如下：

    ```py
    somecolumn.endswith("foo/%bar", escape="^")
    ```

    将被渲染为：

    ```py
    somecolumn LIKE '%' || :param ESCAPE '^'
    ```

    此参数还可以与`ColumnOperators.endswith.autoescape`结合使用：

    ```py
    somecolumn.endswith("foo%bar^bat", escape="^", autoescape=True)
    ```

    给定的文字参数在传递到数据库之前将被转换为`"foo^%bar^^bat"`。

另请参阅

`ColumnOperators.startswith()`

`ColumnOperators.contains()`

`ColumnOperators.like()`

```py
classmethod for_proxy(parent: AssociationProxy[_T], owning_class: Type[Any], parent_instance: Any) → AssociationProxyInstance[_T]
```

```py
method get(obj: Any) → _T | None | AssociationProxyInstance[_T]
```

```py
method has(criterion: _ColumnExpressionArgument[bool] | None = None, **kwargs: Any) → ColumnElement[bool]
```

使用 EXISTS 生成一个代理的‘has’表达式。

此表达式将使用底层代理属性的`Comparator.any()`和/或`Comparator.has()`运算符的组合产品。

```py
method icontains(other: Any, **kw: Any) → ColumnOperators
```

*继承自* `ColumnOperators.icontains()` *方法的* `ColumnOperators`

实现`icontains`运算符，例如`ColumnOperators.contains()`的不区分大小写版本。

生成一个对字符串值中间进行大小写不敏感匹配的 LIKE 表达式：

```py
lower(column) LIKE '%' || lower(<other>) || '%'
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.icontains("foobar"))
```

由于操作符使用了 `LIKE`，存在于<other>表达式中的通配符字符 `"%"` 和 `"_"` 也会像通配符一样行为。对于字面字符串值，可以将`ColumnOperators.icontains.autoescape`标志设置为 `True`，以将这些字符在字符串值中的出现转义，使它们与自身匹配而不是通配符字符。或者，`ColumnOperators.icontains.escape`参数将建立一个给定字符作为转义字符，当目标表达式不是字面字符串时，这可能会有用。

参数：

+   `other` – 待比较的表达式。通常这是一个简单的字符串值，但也可以是任意的 SQL 表达式。除非设置了`ColumnOperators.icontains.autoescape`标志为 True，否则 LIKE 通配符字符 `%` 和 `_` 不会被转义。

+   `autoescape` –

    布尔值；当为 True 时，在 LIKE 表达式中建立一个转义字符，然后将其应用于比较值中的所有出现的 `"%"`、`"_"` 和转义字符本身，假定比较值是一个字面字符串而不是 SQL 表达式。

    一个表达式，比如：

    ```py
    somecolumn.icontains("foo%bar", autoescape=True)
    ```

    将呈现为：

    ```py
    lower(somecolumn) LIKE '%' || lower(:param) || '%' ESCAPE '/'
    ```

    具有值 `:param` 的 `"foo/%bar"`。

+   `escape` –

    一个字符，当给定时，将使用 `ESCAPE` 关键字将该字符建立为转义字符。然后，该字符可以放在 `%` 和 `_` 的前面，以允许它们以自身形式而不是通配符字符的形式进行操作。

    一个表达式，比如：

    ```py
    somecolumn.icontains("foo/%bar", escape="^")
    ```

    将呈现为：

    ```py
    lower(somecolumn) LIKE '%' || lower(:param) || '%' ESCAPE '^'
    ```

    参数也可以与`ColumnOperators.contains.autoescape`结合使用：

    ```py
    somecolumn.icontains("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上述情况下，给定的字面参数将在传递到数据库之前转换为 `"foo^%bar^^bat"`。

另请参阅

`ColumnOperators.contains()`

```py
method iendswith(other: Any, escape: str | None = None, autoescape: bool = False) → ColumnOperators
```

*继承自* `ColumnOperators.iendswith()` *方法的* `ColumnOperators`

实现`iendswith`操作符，例如，`ColumnOperators.endswith()`的不区分大小写版本。

生成一个 LIKE 表达式，用于对字符串值的结尾进行不区分大小写匹配：

```py
lower(column) LIKE '%' || lower(<other>)
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.iendswith("foobar"))
```

由于该操作符使用`LIKE`，在<other>表达式中存在的通配符字符`"%"`和`"_"`也将像通配符一样起作用。对于文字字符串值，可以将`ColumnOperators.iendswith.autoescape`标志设置为`True`，以对字符串值中这些字符的出现进行转义，使它们匹配为它们自身而不是通配符字符。或者，`ColumnOperators.iendswith.escape`参数将建立一个给定字符作为转义字符，当目标表达式不是文字字符串时可能会有用。

参数：

+   `other` – 要进行比较的表达式。这通常是一个简单的字符串值，但也可以是任意的 SQL 表达式。默认情况下，LIKE 通配符字符`%`和`_`不会被转义，除非将`ColumnOperators.iendswith.autoescape`标志设置为 True。

+   `autoescape` –

    布尔值；当为 True 时，在 LIKE 表达式中建立一个转义字符，然后将其应用于比较值中所有的`"%"`、`"_"`和转义字符本身的出现，假定比较值是一个文字字符串而不是 SQL 表达式。

    一个表达式，例如：

    ```py
    somecolumn.iendswith("foo%bar", autoescape=True)
    ```

    将呈现为：

    ```py
    lower(somecolumn) LIKE '%' || lower(:param) ESCAPE '/'
    ```

    其中`:param`的值为`"foo/%bar"`。

+   `escape` –

    一个字符，当给定时，将呈现为`ESCAPE`关键字，以将该字符建立为转义字符。然后可以将此字符放在`%`和`_`的前面，以允许它们作为自身而不是通配符字符。

    一个表达式，例如：

    ```py
    somecolumn.iendswith("foo/%bar", escape="^")
    ```

    将呈现为：

    ```py
    lower(somecolumn) LIKE '%' || lower(:param) ESCAPE '^'
    ```

    该参数也可以与`ColumnOperators.iendswith.autoescape`结合使用：

    ```py
    somecolumn.endswith("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上述情况下，给定的文字参数将在传递到数据库之前转换为`"foo^%bar^^bat"`。

另请参阅

`ColumnOperators.endswith()`

```py
method ilike(other: Any, escape: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.ilike()` *方法的* `ColumnOperators`

实现`ilike`运算符，例如，不区分大小写的 LIKE。

在列上下文中，产生一个形式为：

```py
lower(a) LIKE lower(other)
```

或者在支持 ILIKE 运算符的后端上：

```py
a ILIKE other
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.ilike("%foobar%"))
```

参数：

+   `other` – 要比较的表达式

+   `escape` –

    可选的转义字符，呈现`ESCAPE`关键字，例如：

    ```py
    somecolumn.ilike("foo/%bar", escape="/")
    ```

另请参阅

`ColumnOperators.like()`

```py
method in_(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.in_()` *方法的* `ColumnOperators`

实现`in`运算符。

在列上下文中，产生子句`column IN <other>`。

给定的参数`other`可以是：

+   一个字面值列表，例如：

    ```py
    stmt.where(column.in_([1, 2, 3]))
    ```

    在这种调用形式中，项目列表被转换为与给定列表长度相同的一组绑定参数：

    ```py
    WHERE COL IN (?, ?, ?)
    ```

+   如果比较是针对包含多个表达式的`tuple_()`的元组，则可以提供一个元组列表：

    ```py
    from sqlalchemy import tuple_
    stmt.where(tuple_(col1, col2).in_([(1, 10), (2, 20), (3, 30)]))
    ```

+   一个空列表，例如：

    ```py
    stmt.where(column.in_([]))
    ```

    在这种调用形式中，表达式呈现出一个“空集”表达式。这些表达式是针对各个后端进行定制的，通常试图将一个空的 SELECT 语句作为子查询。例如，在 SQLite 上，表达式是：

    ```py
    WHERE col IN (SELECT 1 FROM (SELECT 1) WHERE 1!=1)
    ```

    从版本 1.4 开始：所有情况下空的 IN 表达式现在使用执行时生成的 SELECT 子查询。

+   如果包含`bindparam()`，则可以使用绑定参数，例如包含`bindparam.expanding`标志：

    ```py
    stmt.where(column.in_(bindparam('value', expanding=True)))
    ```

    在这种调用形式中，表达式呈现出一个特殊的非 SQL 占位符表达式，看起来像：

    ```py
    WHERE COL IN ([EXPANDING_value])
    ```

    此占位符表达式在语句执行时被拦截，转换为前面所示的可变数量的绑定参数形式。如果语句执行为：

    ```py
    connection.execute(stmt, {"value": [1, 2, 3]})
    ```

    数据库将为每个值传递一个绑定参数：

    ```py
    WHERE COL IN (?, ?, ?)
    ```

    版本 1.2 中新增：添加了“expanding”绑定参数

    如果传递一个空列表，则呈现一个特殊的“空列表”表达式，这是特定于正在使用的数据库的。在 SQLite 上，这将是：

    ```py
    WHERE COL IN (SELECT 1 FROM (SELECT 1) WHERE 1!=1)
    ```

    版本 1.3 中新增��现在“expanding”绑定参数支持空列表

+   一个`select()`构造，通常是一个相关的标量选择：

    ```py
    stmt.where(
        column.in_(
            select(othertable.c.y).
            where(table.c.x == othertable.c.x)
        )
    )
    ```

    在这种调用形式中，`ColumnOperators.in_()`呈现如下：

    ```py
    WHERE COL IN (SELECT othertable.y
    FROM othertable WHERE othertable.x = table.x)
    ```

参数：

**other** – 一个文字列表，一个`select()`构造，或者一个包含设置为 True 的`bindparam()`构造，其中包括`bindparam.expanding`标志。

```py
attribute info
```

```py
method is_(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.is_()` *方法的* `ColumnOperators`

实现`IS`运算符。

通常，当与`None`的值进行比较时，`IS`会自动生成，解析为`NULL`。但是，在某些平台上，如果要与布尔值进行比较，则可能希望显式使用`IS`。

另请参阅

`ColumnOperators.is_not()`

```py
method is_distinct_from(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.is_distinct_from()` *方法的* `ColumnOperators`

实现`IS DISTINCT FROM`运算符。

在大多数平台上呈现“a IS DISTINCT FROM b”；在某些平台上，如 SQLite，可能呈现“a IS NOT b”。

```py
method is_not(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.is_not()` *方法的* `ColumnOperators`

实现`IS NOT`运算符。

通常，当与`None`的值进行比较时，`IS NOT`会自动生成，解析为`NULL`。但是，在某些平台上，如果要与布尔值进行比较，则可能希望显式使用`IS NOT`。

在 1.4 版本中更改：`is_not()`运算符从先前版本的`isnot()`重命名。以前的名称仍可用于向后兼容。

另请参阅

`ColumnOperators.is_()`

```py
method is_not_distinct_from(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.is_not_distinct_from()` *方法的* `ColumnOperators`

实现 `IS NOT DISTINCT FROM` 操作符。

在大多数平台上呈现“a IS NOT DISTINCT FROM b”；在某些平台上，如 SQLite，可能呈现“a IS b”。

从版本 1.4 开始更改：`is_not_distinct_from()` 操作符从先前版本的 `isnot_distinct_from()` 重命名。以确保向后兼容性，先前的名称仍然可用。

```py
method isnot(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.isnot()` *方法的* `ColumnOperators`

实现 `IS NOT` 操作符。

通常，当与 `None` 的值进行比较时，会自动生成 `IS NOT`，这将解析为 `NULL`。但是，如果在某些平台上与布尔值进行比较时，可能希望显式使用 `IS NOT`。

从版本 1.4 开始更改：`is_not()` 操作符从先前版本的 `isnot()` 重命名。��确保向后兼容性，先前的名称仍然可用。

另请参见

`ColumnOperators.is_()`

```py
method isnot_distinct_from(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.isnot_distinct_from()` *方法的* `ColumnOperators`

实现 `IS NOT DISTINCT FROM` 操作符。

在大多数平台上呈现“a IS NOT DISTINCT FROM b”；在某些平台上，如 SQLite，可能呈现“a IS b”。

从版本 1.4 开始更改：`is_not()` 操作符从先前版本的 `isnot()` 重命名。以确保向后兼容性，先前的名称仍然可用。

```py
method istartswith(other: Any, escape: str | None = None, autoescape: bool = False) → ColumnOperators
```

*继承自* `ColumnOperators.istartswith()` *方法的* `ColumnOperators`

实现 `istartswith` 操作符，例如，`ColumnOperators.startswith()` 的不区分大小写版本。

生成一个 LIKE 表达式，用于对字符串值的开头进行不区分大小写匹配：

```py
lower(column) LIKE lower(<other>) || '%'
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.istartswith("foobar"))
```

由于该运算符使用`LIKE`，在<other>表达式中存在的通配符字符`"%"`和`"_"`也将像通配符一样起作用。对于文字字符串值，可以将`ColumnOperators.istartswith.autoescape`标志设置为`True`，以对字符串值中这些字符的出现应用转义，使它们匹配为自身而不是通配符字符。或者，`ColumnOperators.istartswith.escape`参数将建立一个给定字符作为转义字符，当目标表达式不是文字字符串时可能会有用。

参数：

+   `other` – 待比较的表达式。通常是一个简单的字符串值，但也可以是任意的 SQL 表达式。除非设置`ColumnOperators.istartswith.autoescape`标志为 True，否则不会默认转义 LIKE 通配符`%`和`_`。

+   `autoescape` –

    布尔值；当为 True 时，在 LIKE 表达式中建立一个转义字符，然后将其应用于比较值中的所有`"%"`、`"_"`和转义字符本身的出现，假定比较值是一个文字字符串而不是一个 SQL 表达式。

    一个表达式，例如：

    ```py
    somecolumn.istartswith("foo%bar", autoescape=True)
    ```

    将呈现为：

    ```py
    lower(somecolumn) LIKE lower(:param) || '%' ESCAPE '/'
    ```

    使用值`:param`为`"foo/%bar"`。

+   `escape` –

    一个字符，当给定时，将使用`ESCAPE`关键字来将该字符作为转义字符。然后可以将该字符放在`%`和`_`之前，以允许它们作为自身而不是通配符字符。

    例如：

    ```py
    somecolumn.istartswith("foo/%bar", escape="^")
    ```

    将呈现为：

    ```py
    lower(somecolumn) LIKE lower(:param) || '%' ESCAPE '^'
    ```

    该参数也可以与`ColumnOperators.istartswith.autoescape`结合使用：

    ```py
    somecolumn.istartswith("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上述情况下，给定的文字参数在传递到数据库之前将被转换为`"foo^%bar^^bat"`。

另请参阅

`ColumnOperators.startswith()`

```py
method like(other: Any, escape: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.like()` *方法的* `ColumnOperators`

实现`like`运算符。

在列上下文中，生成表达式：

```py
a LIKE other
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.like("%foobar%"))
```

参数：

+   `other` – 待比较的表达式

+   `escape` –

    可选的转义字符，渲染`ESCAPE`关键字，例如：

    ```py
    somecolumn.like("foo/%bar", escape="/")
    ```

另请参阅

`ColumnOperators.ilike()`

```py
attribute local_attr
```

此`AssociationProxyInstance`引用的‘local’类属性。

另请参阅

`AssociationProxyInstance.attr`

`AssociationProxyInstance.remote_attr`

```py
method match(other: Any, **kwargs: Any) → ColumnOperators
```

*继承自* `ColumnOperators.match()` *方法的* `ColumnOperators`

实现特定于数据库的‘match’操作符。

`ColumnOperators.match()`尝试解析为后端提供的类似 MATCH 的函数或操作符。例如：

+   PostgreSQL - 渲染`x @@ plainto_tsquery(y)`

    > 从版本 2.0 开始更改：现在在 PostgreSQL 中使用`plainto_tsquery()`代替`to_tsquery()`；为了与其他形式兼容，请参阅全文搜索。

+   MySQL - 渲染`MATCH (x) AGAINST (y IN BOOLEAN MODE)`

    另请参阅

    `match` - 具有附加功能的 MySQL 特定构造。

+   Oracle - 渲染`CONTAINS(x, y)`

+   其他后端可能提供��殊实现。

+   没有任何特殊实现的后端将将操作符发出为“MATCH”。例如，这与 SQLite 兼容。

```py
method not_ilike(other: Any, escape: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.not_ilike()` *方法的* `ColumnOperators`

实现`NOT ILIKE`操作符。

这等同于使用`ColumnOperators.ilike()`进行否定，即`~x.ilike(y)`。

从版本 1.4 开始更改：`not_ilike()`操作符从先前版本的`notilike()`重命名。以前的名称仍可用于向后兼容。

另请参阅

`ColumnOperators.ilike()`

```py
method not_in(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.not_in()` *方法的* `ColumnOperators`

实现 `NOT IN` 运算符。

这相当于使用 `ColumnOperators.in_()` 的否定，即 `~x.in_(y)`。

在 `other` 是空序列的情况下，编译器会生成一个“空 not in” 表达式。 默认情况下，这会转换为表达式 “1 = 1”，以在所有情况下产生 true。 可以使用 `create_engine.empty_in_strategy` 来更改此行为。

在版本 1.4 中更改：`not_in()` 运算符从先前版本的 `notin_()` 重命名。 以前的名称仍可用于向后兼容。

在版本 1.2 中更改：`ColumnOperators.in_()` 和 `ColumnOperators.not_in()` 运算符现在默认情况下会为空的 IN 序列生成一个“静态” 表达式。

另请参阅

`ColumnOperators.in_()`

```py
method not_like(other: Any, escape: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.not_like()` *方法的* `ColumnOperators`

实现 `NOT LIKE` 运算符。

这相当于使用 `ColumnOperators.like()` 的否定，即 `~x.like(y)`。

在���本 1.4 中更改：`not_like()` 运算符从先前版本的 `notlike()` 重命名。 以前的名称仍可用于向后兼容。

另请参阅

`ColumnOperators.like()`

```py
method notilike(other: Any, escape: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.notilike()` *方法的* `ColumnOperators`

实现 `NOT ILIKE` 运算符。

这相当于使用`ColumnOperators.ilike()`时进行否定，即`~x.ilike(y)`。

从 1.4 版本开始更改：`not_ilike()`操作符从先前版本的`notilike()`重命名。以前的名称仍可用于向后兼容。

另请参阅

`ColumnOperators.ilike()`

```py
method notin_(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.notin_()` *的方法* `ColumnOperators`

实现`NOT IN`操作符。

这相当于使用`ColumnOperators.in_()`时进行否定，即`~x.in_(y)`。

如果`other`是一个空序列，则编译器会生成一个“空 not in”表达式。默认情况下，这将产生“1 = 1”表达式，以在所有情况下产生 true。`create_engine.empty_in_strategy`可用于更改此行为。

从 1.4 版本开始更改：`not_in()`操作符从先前版本的`notin_()`重命名。以前的名称仍可用于向后兼容。

从 1.2 版本开始更改：`ColumnOperators.in_()`和`ColumnOperators.not_in()`操作符现在默认情况下为一个空 IN 序列生成“静态”表达式。

另请参阅

`ColumnOperators.in_()`

```py
method notlike(other: Any, escape: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.notlike()` *的方法* `ColumnOperators`

实现`NOT LIKE`操作符。

这相当于使用`ColumnOperators.like()`时进行否定，即`~x.like(y)`。

从 1.4 版本开始更改：`not_like()`操作符从先前版本的`notlike()`重命名。以前的名称仍可用于向后兼容。

另请参阅

`ColumnOperators.like()`

```py
method nulls_first() → ColumnOperators
```

*继承自* `ColumnOperators.nulls_first()` *方法* `ColumnOperators` *中*

生成针对父对象的 `nulls_first()` 子句。

从版本 1.4 开始：`nulls_first()` 操作符从之前的版本 `nullsfirst()` 重新命名。以前的名称仍然可用于向后兼容。

```py
method nulls_last() → ColumnOperators
```

*继承自* `ColumnOperators.nulls_last()` *方法* `ColumnOperators`

生成针对父对象的 `nulls_last()` 子句。

从版本 1.4 开始：`nulls_last()` 操作符从之前的版本 `nullslast()` 重新命名。以前的名称仍然可用于向后兼容。

```py
method nullsfirst() → ColumnOperators
```

*继承自* `ColumnOperators.nullsfirst()` *方法* `ColumnOperators`

生成针对父对象的 `nulls_first()` 子句。

从版本 1.4 开始：`nulls_first()` 操作符从之前的版本 `nullsfirst()` 重新命名。以前的名称仍然可用于向后兼容。

```py
method nullslast() → ColumnOperators
```

*继承自* `ColumnOperators.nullslast()` *方法* `ColumnOperators`

生成针对父对象的 `nulls_last()` 子句。

从版本 1.4 开始：`nulls_last()` 操作符从之前的版本 `nullslast()` 重新命名。以前的名称仍然可用于向后兼容。

```py
method op(opstring: str, precedence: int = 0, is_comparison: bool = False, return_type: Type[TypeEngine[Any]] | TypeEngine[Any] | None = None, python_impl: Callable[..., Any] | None = None) → Callable[[Any], Operators]
```

*继承自* `Operators.op()` *方法的* `Operators`

生成一个通用的运算符函数。

例如：

```py
somecolumn.op("*")(5)
```

产生：

```py
somecolumn * 5
```

这个函数也可以用于使按位运算符明确。例如：

```py
somecolumn.op('&')(0xff)
```

是`somecolumn`中值的按位与。

参数：

+   `opstring` - 一个字符串，将作为此元素与传递给生成函数的表达式之间的中缀运算符输出。

+   `precedence` -

    数据库在 SQL 表达式中预期应用于运算符的优先级。这个整数值作为 SQL 编译器的提示，用于确定何时应该在特定操作周围添加显式括号。较低的数字将导致表达式在应用于具有较高优先级的其他运算符时被括在括号中。默认值为`0`，低于所有运算符，除了逗号（`,`）和`AS`运算符。值为 100 将高于或等于所有运算符，-100 将低于或等于所有运算符。

    另请参阅

    我正在使用 op() 生成自定义运算符，但我的括号没有正确显示 - SQLAlchemy SQL 编译器如何渲染括号的详细描述

+   `is_comparison` -

    传统；如果为 True，则该运算符将被视为“比较”运算符，即评估为布尔真/假值的运算符，如`==`，`>`等。提供此标志是为了使 ORM 关系能够在自定义连接条件中确定该运算符是比较运算符。

    使用`is_comparison`参数已被使用`Operators.bool_op()`方法取代；这个更简洁的运算符会自动设置这个参数，但也提供了正确的[**PEP 484**](https://peps.python.org/pep-0484/)类型支持，因为返回的对象将表达“布尔”数据类型，即`BinaryExpression[bool]`。

+   `return_type` - 一个`TypeEngine`类或对象，将强制此运算符生成的表达式的返回类型为该类型。默认情况下，指定`Operators.op.is_comparison`的运算符将解析为`Boolean`，而那些不指定的将与左操作数的类型相同。

+   `python_impl` -

    可选的 Python 函数，可以以与在数据库服务器上运行时此运算符的工作方式相同的方式评估两个 Python 值。适用于在 Python 中进行 SQL 表达式评估函数，例如用于 ORM 混合属性的函数，以及在多行更新或删除后用于匹配会话中的对象的 ORM “评估器”。

    例如：

    ```py
    >>> expr = column('x').op('+', python_impl=lambda a, b: a + b)('y')
    ```

    上述表达式的运算符也适用于非 SQL 左侧和右侧对象：

    ```py
    >>> expr.operator(5, 10)
    15
    ```

    2.0 版新功能。

另请参阅

`Operators.bool_op()`

重新定义和创建新运算符

在连接条件中使用自定义运算符

```py
method operate(op: OperatorType, *other: Any, **kwargs: Any) → Operators
```

*继承自* `Operators.operate()` *方法的* `Operators`

对参数进行操作。

这是操作的最低级别，默认情况下引发 `NotImplementedError`。

在子类上重新定义此操作可以使常见行为应用于所有操作。例如，覆盖 `ColumnOperators` 以将 `func.lower()` 应用于左侧和右侧：

```py
class MyComparator(ColumnOperators):
    def operate(self, op, other, **kwargs):
        return op(func.lower(self), func.lower(other), **kwargs)
```

参数：

+   `op` – 运算符可调用。

+   `*other` – 操作的“其他”一侧。对于大多数操作，将是单个标量。

+   `**kwargs` – 修饰符。这些可能由特殊运算符传递，例如 `ColumnOperators.contains()`。

```py
attribute parent: _AssociationProxyProtocol[_T]
```

```py
method regexp_match(pattern: Any, flags: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.regexp_match()` *方法的* `ColumnOperators`

实现特定于数据库的“regexp 匹配”运算符。

例如：

```py
stmt = select(table.c.some_column).where(
    table.c.some_column.regexp_match('^(b|c)')
)
```

`ColumnOperators.regexp_match()` 尝试解析为后端提供的类似 REGEXP 的函数或运算符，但特定的正则表达式语法和可用标志**不是后端不可知的**。

示例包括：

+   PostgreSQL - 渲染 `x ~ y` 或 `x !~ y`（在否定时）。

+   Oracle - 渲染 `REGEXP_LIKE(x, y)`

+   SQLite - 使用 SQLite 的 `REGEXP` 占位符运算符，并调用 Python 的 `re.match()` 内置函数。

+   其他后端可能提供特殊实现。

+   没有任何特殊实现的后端将将运算符发出为 “REGEXP” 或 “NOT REGEXP”。例如，这与 SQLite 和 MySQL 兼容。

目前正实现正则表达式支持的数据库有 Oracle、PostgreSQL、MySQL 和 MariaDB。SQLite 可用部分支持。第三方方言中的支持可能有所不同。

参数：

+   `pattern` – 正则表达式模式字符串或列子句。

+   `flags` – 要应用的任何正则表达式字符串标志，仅作为普通 Python 字符串传递。这些标志是特定于后端的。某些后端，如 PostgreSQL 和 MariaDB，可能会将标志作为模式的一部分指定。在 PostgreSQL 中使用忽略大小写标志‘i’ 时，将使用忽略大小写的正则表达式匹配运算符 `~*` 或 `!~*`。

版本 1.4 中的新功能。

从版本 1.4.48 更改为 2.0.18：请注意，由于实现错误，“flags”参数先前接受 SQL 表达式对象，例如列表达式，而不仅仅是普通的 Python 字符串。这种实现与缓存不兼容，并已被移除；“flags”参数应仅传递字符串，因为这些标志将作为 SQL 表达式中的文字内联值呈现。

另请参阅

`ColumnOperators.regexp_replace()`

```py
method regexp_replace(pattern: Any, replacement: Any, flags: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.regexp_replace()` *方法的* `ColumnOperators`

实现了特定于数据库的‘regexp replace’运算符。

例如：

```py
stmt = select(
    table.c.some_column.regexp_replace(
        'b(..)',
        'XY',
        flags='g'
    )
)
```

`ColumnOperators.regexp_replace()` 试图解析为后端提供的类似 REGEXP_REPLACE 的函数，通常会生成函数 `REGEXP_REPLACE()`。然而，特定的正则表达式语法和可用的标志**并非后端通用**。

目前正实现正则表达式替换支持的数据库有 Oracle、PostgreSQL、MySQL 8 或更高版本以及 MariaDB。第三方方言中的支持可能有所不同。

参数：

+   `pattern` – 正则表达式模式字符串或列子句。

+   `pattern` – 替换字符串或列子句。

+   `flags` – 要应用的任何正则表达式字符串标志，仅作为普通 Python 字符串传递。这些标志是特定于后端的。某些后端，如 PostgreSQL 和 MariaDB，可能会将标志作为模式的一部分指定。

版本 1.4 中的新功能。

在版本 1.4.48 更改为：2.0.18 请注意，由于实现错误，先前的“flags”参数接受了 SQL 表达式对象，例如列表达式，而不仅仅是普通的 Python 字符串。此实现与缓存一起使用时无法正常工作，并已删除；应该仅传递字符串作为“flags”参数，因为这些标志将作为 SQL 表达式中的文字内联值呈现。

另请参见

`ColumnOperators.regexp_match()`

```py
attribute remote_attr
```

被此`AssociationProxyInstance`引用的“远程”类属性。

另请参见

`AssociationProxyInstance.attr`

`AssociationProxyInstance.local_attr`

```py
method reverse_operate(op: OperatorType, other: Any, **kwargs: Any) → Operators
```

*继承自* `Operators.reverse_operate()` *方法的* `Operators`

对参数进行反向操作。

用法与`operate()`相同。

```py
attribute scalar
```

如果此`AssociationProxyInstance`代理本地端的标量关系，则返回`True`。

```py
method set(obj: Any, values: _T) → None
```

```py
method startswith(other: Any, escape: str | None = None, autoescape: bool = False) → ColumnOperators
```

*继承自* `ColumnOperators.startswith()` *方法的* `ColumnOperators`

实现`startswith`运算符。

生成一个 LIKE 表达式，用于测试字符串值开头的匹配：

```py
column LIKE <other> || '%'
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.startswith("foobar"))
```

由于该操作符使用 `LIKE`，因此存在于 `<other>` 表达式中的通配符字符 `"%"` 和 `"_"` 也会像通配符一样起作用。对于字面字符串值，可以将 `ColumnOperators.startswith.autoescape` 标志设置为 `True`，以对字符串值内出现的这些字符进行转义，使它们匹配为它们自身而不是通配符字符。或者，`ColumnOperators.startswith.escape` 参数将建立一个给定字符作为转义字符，当目标表达式不是字面字符串时，这可能会有用。

参数：

+   `other` – 要比较的表达式。通常是一个普通的字符串值，但也可以是任意的 SQL 表达式。`LIKE` 通配符字符 `%` 和 `_` 默认情况下不会被转义，除非设置了 `ColumnOperators.startswith.autoescape` 标志为 True。

+   `autoescape` –

    布尔值；当为 True 时，在 `LIKE` 表达式中建立一个转义字符，然后将其应用于比较值内所有出现的 `"%"`、`"_"` 和转义字符本身，假定比较值是一个字面字符串而不是 SQL 表达式。

    诸如：

    ```py
    somecolumn.startswith("foo%bar", autoescape=True)
    ```

    将呈现为：

    ```py
    somecolumn LIKE :param || '%' ESCAPE '/'
    ```

    其中值为 `:param`，为 `"foo/%bar"`。

+   `escape` –

    一个字符，当给定时，将使用 `ESCAPE` 关键字来建立该字符作为转义字符。然后可以将该字符放在 `%` 和 `_` 的出现之前，以允许它们作为自身而不是通配符字符。

    诸如：

    ```py
    somecolumn.startswith("foo/%bar", escape="^")
    ```

    将呈现为：

    ```py
    somecolumn LIKE :param || '%' ESCAPE '^'
    ```

    参数也可以与 `ColumnOperators.startswith.autoescape` 结合使用：

    ```py
    somecolumn.startswith("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上述情况下，给定的字面参数将在传递到数据库之前转换为 `"foo^%bar^^bat"`。

另请参阅

`ColumnOperators.endswith()`

`ColumnOperators.contains()`

`ColumnOperators.like()`

```py
attribute target_class: Type[Any]
```

由`AssociationProxyInstance`处理的中间类。

拦截的追加/设置/赋值事件将导致生成此类的新实例。

```py
attribute timetuple: Literal[None] = None
```

*继承自* `ColumnOperators.timetuple` *属性的* `ColumnOperators`

Hack，允许在左侧比较日期时间对象。

```py
class sqlalchemy.ext.associationproxy.ObjectAssociationProxyInstance
```

一个`AssociationProxyInstance`，其目标是一个对象。

**成员**

__le__(), __lt__(), all_(), any(), any_(), asc(), attr, between(), bitwise_and(), bitwise_lshift(), bitwise_not(), bitwise_or(), bitwise_rshift(), bitwise_xor(), bool_op(), collate(), concat(), contains(), desc(), distinct(), endswith(), has(), icontains(), iendswith(), ilike(), in_(), is_(), is_distinct_from(), is_not(), is_not_distinct_from(), isnot(), isnot_distinct_from(), istartswith(), like(), local_attr, match(), not_ilike(), not_in(), not_like(), notilike(), notin_(), notlike(), nulls_first(), nulls_last(), nullsfirst(), nullslast(), op(), operate(), regexp_match(), regexp_replace(), remote_attr, reverse_operate(), scalar, startswith(), target_class, timetuple

**类签名**

类`sqlalchemy.ext.associationproxy.ObjectAssociationProxyInstance`（`sqlalchemy.ext.associationproxy.AssociationProxyInstance`）

```py
method __le__(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators` *的* `sqlalchemy.sql.expression.ColumnOperators.__le__` *方法*

实现`<=`运算符。

在列上下文中，生成子句`a <= b`。

```py
method __lt__(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators` *的* `sqlalchemy.sql.expression.ColumnOperators.__lt__` *方法*

实现`<`运算符。

在列上下文中，生成子句`a < b`。

```py
method all_() → ColumnOperators
```

*继承自* `ColumnOperators` *的* `ColumnOperators.all_()` *方法*

生成针对父对象的`all_()`子句。

有关`all_()`的文档，请参阅示例。

注意

请务必不要将更新的`ColumnOperators.all_()`方法与此方法的**旧版**混淆，此方法的旧版是针对`ARRAY`特定的`Comparator.all()`方法，其使用不同的调用风格。

```py
method any(criterion: _ColumnExpressionArgument[bool] | None = None, **kwargs: Any) → ColumnElement[bool]
```

*继承自* `AssociationProxyInstance` *的* `AssociationProxyInstance.any()` *方法*

使用 EXISTS 生成代理的“any”表达式。

此表达式将是使用底层代理属性的`Comparator.any()`和/或`Comparator.has()`运算符的组合产品。

```py
method any_() → ColumnOperators
```

*继承自* `ColumnOperators` *的* `ColumnOperators.any_()` *方法*

对父对象生成一个`any_()`子句。

请参阅`any_()`的文档以获取示例。

注意

请确保不要混淆较新的`ColumnOperators.any_()`方法与这个方法的**旧版**，即专用于`ARRAY`的`Comparator.any()`方法，它使用不同的调用风格。

```py
method asc() → ColumnOperators
```

*继承自* `ColumnOperators.asc()` *方法的* `ColumnOperators`

对父对象生成一个`asc()`子句。

```py
attribute attr
```

*继承自* `AssociationProxyInstance.attr` *属性的* `AssociationProxyInstance`

返回一个元组`(local_attr, remote_attr)`。

该属性最初旨在促进使用`Query.join()`方法一次性跨越两个关系的连接，但这使用了一个已弃用的调用风格。

要使用`select.join()`或`Query.join()`与关联代理一起使用，当前的方法是分别利用`AssociationProxyInstance.local_attr`和`AssociationProxyInstance.remote_attr`属性：

```py
stmt = (
    select(Parent).
    join(Parent.proxied.local_attr).
    join(Parent.proxied.remote_attr)
)
```

未来的版本可能会为关联代理属性提供更简洁的连接模式。

另请参阅

`AssociationProxyInstance.local_attr`

`AssociationProxyInstance.remote_attr`

```py
method between(cleft: Any, cright: Any, symmetric: bool = False) → ColumnOperators
```

*继承自* `ColumnOperators.between()` *方法的* `ColumnOperators`

对父对象生成 `between()` 子句，给定下限和上限范围。

```py
method bitwise_and(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_and()` *方法的* `ColumnOperators`

通过 `&` 运算符生成位与操作。

版本 2.0.2 中的新功能。

另请参阅

位运算符

```py
method bitwise_lshift(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_lshift()` *方法的* `ColumnOperators`

通过 `<<` 运算符生成位左移操作。

版本 2.0.2 中的新功能。

另请参阅

位运算符

```py
method bitwise_not() → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_not()` *方法的* `ColumnOperators`

通过 `~` 运算符生成位非操作。

版本 2.0.2 中的新功能。

另请参阅

位运算符

```py
method bitwise_or(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_or()` *方法的* `ColumnOperators`

通过 `|` 运算符生成位或操作。

版本 2.0.2 中的新功能。

另请参阅

位运算符

```py
method bitwise_rshift(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_rshift()` *方法的* `ColumnOperators`

通过 `>>` 运算符生成位右移操作。

版本 2.0.2 中的新功能。

另请参阅

位运算符

```py
method bitwise_xor(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_xor()` *方法的* `ColumnOperators`

生成一个按位异或操作，通常通过 `^` 运算符实现，或在 PostgreSQL 上使用 `#`。

版本 2.0.2 中的新功能。

另请参阅

位运算符

```py
method bool_op(opstring: str, precedence: int = 0, python_impl: Callable[[...], Any] | None = None) → Callable[[Any], Operators]
```

*继承自* `Operators.bool_op()` *方法的* `Operators`

返回一个自定义布尔运算符。

这个方法是调用 `Operators.op()` 并传递 `Operators.op.is_comparison` 标志为 True 的简写。 使用 `Operators.bool_op()` 的一个关键优势是，在使用列构造时，返回表达式的“布尔”性质将出现在 [**PEP 484**](https://peps.python.org/pep-0484/) 目的中。

另请参阅

`Operators.op()`

```py
method collate(collation: str) → ColumnOperators
```

*继承自* `ColumnOperators.collate()` *方法的* `ColumnOperators`

生成一个针对父对象的 `collate()` 子句，给定排序规则字符串。

另请参阅

`collate()`

```py
method concat(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.concat()` *方法的* `ColumnOperators`

实现“concat”运算符。

在列上下文中，生成子句 `a || b`，或在 MySQL 上使用 `concat()` 运算符。

```py
method contains(other: Any, **kw: Any) → ColumnElement[bool]
```

使用 EXISTS 生成一个代理的“包含”表达式。

这个表达式将使用底层代理属性的`Comparator.any()`、`Comparator.has()`和/或`Comparator.contains()`操作符组成的产品。

```py
method desc() → ColumnOperators
```

*继承自* `ColumnOperators.desc()` *方法的* `ColumnOperators`

对父对象生成一个`desc()`子句。

```py
method distinct() → ColumnOperators
```

*继承自* `ColumnOperators.distinct()` *方法的* `ColumnOperators`

对父对象生成一个`distinct()`子句。

```py
method endswith(other: Any, escape: str | None = None, autoescape: bool = False) → ColumnOperators
```

*继承自* `ColumnOperators.endswith()` *方法的* `ColumnOperators`

实现‘endswith’操作符。

生成一个 LIKE 表达式，用于测试字符串值的结尾匹配：

```py
column LIKE '%' || <other>
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.endswith("foobar"))
```

由于该操作符使用 `LIKE`，存在于<other>表达式中的通配符字符 `"%"` 和 `"_"` 也将像通配符一样起作用。对于字面字符串值，可以将`ColumnOperators.endswith.autoescape`标志设置为`True`，以对字符串值中这些字符的出现进行转义，使它们匹配为它们自身而不是通配符字符。或者，`ColumnOperators.endswith.escape`参数将建立一个给定字符作为转义字符，当目标表达式不是字面字符串时，这可能会有用。

参数：

+   `other` – 要比较的表达式。通常是一个普通的字符串值，但也可以是任意的 SQL 表达式。除非设置`ColumnOperators.endswith.autoescape`标志为 True，否则 LIKE 通配符字符 `%` 和 `_` 默认不会被转义。

+   `autoescape` –

    布尔值；当为 True 时，在 LIKE 表达式中建立一个转义字符，然后将其应用于比较值中的所有`"%"`、`"_"`和转义字符本身的出现，假定比较值为文字字符串而不是 SQL 表达式。

    例如：

    ```py
    somecolumn.endswith("foo%bar", autoescape=True)
    ```

    将呈现为：

    ```py
    somecolumn LIKE '%' || :param ESCAPE '/'
    ```

    使用值`:param`为`"foo/%bar"`。

+   `escape` –

    一个字符，当给定时，将使用`ESCAPE`关键字来建立该字符作为转义字符。然后可以将此字符放在`%`和`_`之前，以允许它们作为自身而不是通配符字符。

    例如：

    ```py
    somecolumn.endswith("foo/%bar", escape="^")
    ```

    将呈现为：

    ```py
    somecolumn LIKE '%' || :param ESCAPE '^'
    ```

    参数也可以与`ColumnOperators.endswith.autoescape`结合使用：

    ```py
    somecolumn.endswith("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上述情况下，给定的文字参数将在传递到数据库之前转换为`"foo^%bar^^bat"`。

另请参阅

`ColumnOperators.startswith()`

`ColumnOperators.contains()`

`ColumnOperators.like()`

```py
method has(criterion: _ColumnExpressionArgument[bool] | None = None, **kwargs: Any) → ColumnElement[bool]
```

*继承自* `AssociationProxyInstance.has()` *方法的* `AssociationProxyInstance`

生成一个使用 EXISTS 的代理‘has’表达式。

此表达式将使用基础代理属性的`Comparator.any()`和/或`Comparator.has()`运算符的组合产品。

```py
method icontains(other: Any, **kw: Any) → ColumnOperators
```

*继承自* `ColumnOperators.icontains()` *方法的* `ColumnOperators`

实现`icontains`运算符，例如，`ColumnOperators.contains()`的不区分大小写版本。

生成一个 LIKE 表达式，用于对字符串值中间的不区分大小写匹配进行测试：

```py
lower(column) LIKE '%' || lower(<other>) || '%'
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.icontains("foobar"))
```

由于该操作符使用`LIKE`，因此在<other>表达式中存在的`"%"`和`"_"`通配符字符也将像通配符一样起作用。对于文字字符串值，可以将`ColumnOperators.icontains.autoescape`标志设置为`True`，以对字符串值中这些字符的出现进行转义，使它们作为自身而不是通配符字符进行匹配。或者，`ColumnOperators.icontains.escape`参数将确定一个给定字符作为转义字符，当目标表达式不是文字字符串时可能会有用。

参数：

+   `other` – 要进行比较的表达式。通常是一个普通字符串值，但也可以是任意 SQL 表达式。默认情况下，`LIKE`通配符字符`%`和`_`不会被转义，除非设置了`ColumnOperators.icontains.autoescape`标志为 True。

+   `autoescape` –

    布尔值；当为 True 时，在`LIKE`表达式中建立一个转义字符，然后将其应用于比较值中所有的`"%"`、`"_"`和转义字符本身的出现次数，假定比较值是一个文字字符串而不是 SQL 表达式。

    例如表达式：

    ```py
    somecolumn.icontains("foo%bar", autoescape=True)
    ```

    将呈现为：

    ```py
    lower(somecolumn) LIKE '%' || lower(:param) || '%' ESCAPE '/'
    ```

    其中`:param`的值为`"foo/%bar"`。

+   `escape` –

    一个字符，当给定时，将使用`ESCAPE`关键字来确定该字符作为转义字符。然后可以将该字符放在`%`和`_`之前，以使它们作为自身而不是通配符字符。

    例如表达式：

    ```py
    somecolumn.icontains("foo/%bar", escape="^")
    ```

    将呈现为：

    ```py
    lower(somecolumn) LIKE '%' || lower(:param) || '%' ESCAPE '^'
    ```

    该参数也可以与`ColumnOperators.contains.autoescape`结合使用：

    ```py
    somecolumn.icontains("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上述情况下，给定的文字参数将在传递到数据库之前转换为`"foo^%bar^^bat"`。

另请参见

`ColumnOperators.contains()`

```py
method iendswith(other: Any, escape: str | None = None, autoescape: bool = False) → ColumnOperators
```

*继承自* `ColumnOperators.iendswith()` *方法的* `ColumnOperators`

实现`iendswith`运算符，例如，对`ColumnOperators.endswith()`的不区分大小写版本。

生成一个 LIKE 表达式，用于对字符串值的不区分大小写匹配进行测试：

```py
lower(column) LIKE '%' || lower(<other>)
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.iendswith("foobar"))
```

由于运算符使用了`LIKE`，因此存在于<other>表达式中的通配符字符`"%"`和`"_"`也将像通配符一样工作。对于字面字符串值，可以将`ColumnOperators.iendswith.autoescape`标志设置为`True`，以对字符串值中这些字符的出现应用转义，以使它们与自身匹配而不是通配符字符。或者，`ColumnOperators.iendswith.escape`参数将建立一个给定的字符作为转义字符，这在目标表达式不是字面字符串时可能有用。

参数：

+   `other` – 要比较的表达式。这通常是一个简单的字符串值，但也可以是任意的 SQL 表达式。LIKE 通配符字符`%`和`_`默认情况下不会转义，除非将`ColumnOperators.iendswith.autoescape`标志设置为 True。

+   `autoescape` –

    布尔值；当为 True 时，在 LIKE 表达式中建立一个转义字符，然后将其应用于比较值中的所有出现的`"%"`、`"_"`和转义字符本身，假定比较值是一个字面字符串而不是一个 SQL 表达式。

    一个表达式，例如：

    ```py
    somecolumn.iendswith("foo%bar", autoescape=True)
    ```

    将呈现为：

    ```py
    lower(somecolumn) LIKE '%' || lower(:param) ESCAPE '/'
    ```

    以`param`的值为`"foo/%bar"`。

+   `escape` –

    一个字符，当给定时，将使用`ESCAPE`关键字来建立该字符作为转义字符。然后，可以将此字符放在`%`和`_`之前，以允许它们作为自身而不是通配符字符。

    一个表达式，例如：

    ```py
    somecolumn.iendswith("foo/%bar", escape="^")
    ```

    将呈现为：

    ```py
    lower(somecolumn) LIKE '%' || lower(:param) ESCAPE '^'
    ```

    参数也可以与`ColumnOperators.iendswith.autoescape`结合使用：

    ```py
    somecolumn.endswith("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上述情况下，给定的字面参数将在传递到数据库之前被转换为`"foo^%bar^^bat"`。

另见

`ColumnOperators.endswith()`

```py
method ilike(other: Any, escape: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.ilike()` *方法的* `ColumnOperators`

实现`ilike`运算符，例如不区分大小写的 LIKE。

在列上下文中，产生形式为：

```py
lower(a) LIKE lower(other)
```

或者在支持 ILIKE 运算符的后端：

```py
a ILIKE other
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.ilike("%foobar%"))
```

参数：

+   `other` – 要比较的表达式

+   `escape` –

    可选的转义字符，呈现`ESCAPE`关键字，例如：

    ```py
    somecolumn.ilike("foo/%bar", escape="/")
    ```

另请参阅

`ColumnOperators.like()`

```py
method in_(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.in_()` *方法的* `ColumnOperators`

实现`in`运算符。

在列上下文中，生成子句`column IN <other>`。

给定的参数`other`可以是：

+   一个字面值列表，例如：

    ```py
    stmt.where(column.in_([1, 2, 3]))
    ```

    在这种调用形式中，项目列表被转换为与给定列表相同长度的一组绑定参数：

    ```py
    WHERE COL IN (?, ?, ?)
    ```

+   如果比较是针对包含多个表达式的`tuple_()`，则可以提供元组列表：

    ```py
    from sqlalchemy import tuple_
    stmt.where(tuple_(col1, col2).in_([(1, 10), (2, 20), (3, 30)]))
    ```

+   一个空列表，例如：

    ```py
    stmt.where(column.in_([]))
    ```

    在这种调用形式中，表达式呈现一个“空集”表达式。这些表达式针对各个后端进行了定制，并且通常试图将空的 SELECT 语句作为子查询。例如在 SQLite 上，表达式是：

    ```py
    WHERE col IN (SELECT 1 FROM (SELECT 1) WHERE 1!=1)
    ```

    从版本 1.4 开始更改：在所有情况下，空的 IN 表达式现在都使用运行时生成的 SELECT 子查询。

+   可以使用绑定参数，例如`bindparam()`，如果它包含`bindparam.expanding`标志：

    ```py
    stmt.where(column.in_(bindparam('value', expanding=True)))
    ```

    在这种调用形式中，表达式呈现一个特殊的非 SQL 占位符表达式，看起来像：

    ```py
    WHERE COL IN ([EXPANDING_value])
    ```

    此占位符表达式在语句执行时被拦截，以转换为前面所示的可变数量的绑定参数形式。如果执行语句如下：

    ```py
    connection.execute(stmt, {"value": [1, 2, 3]})
    ```

    数据库将传递一个绑定参数给每个值：

    ```py
    WHERE COL IN (?, ?, ?)
    ```

    从版本 1.2 开始：添加了“扩展”绑定参数

    如果传递了一个空列表，则会呈现一个特殊的“空列表”表达式，该表达式特定于正在使用的数据库。在 SQLite 上，这将是：

    ```py
    WHERE COL IN (SELECT 1 FROM (SELECT 1) WHERE 1!=1)
    ```

    从版本 1.3 开始：“扩展”绑定参数现在支持空列表

+   一个`select()`构造，通常是一个相关的标量选择：

    ```py
    stmt.where(
        column.in_(
            select(othertable.c.y).
            where(table.c.x == othertable.c.x)
        )
    )
    ```

    在这种调用形式中，`ColumnOperators.in_()` 渲染如下：

    ```py
    WHERE COL IN (SELECT othertable.y
    FROM othertable WHERE othertable.x = table.x)
    ```

参数：

**other** – 一组字面常量，一个`select()`构造，或者一个包含 `bindparam.expanding` 标志设置为 True 的 `bindparam()` 构造。

```py
method is_(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.is_()` *方法的* `ColumnOperators`

实现`IS`运算符。

通常，当与`None`值进行比较时，`IS`会自动生成，该值解析为`NULL`。然而，在某些平台上，如果要与布尔值进行比较，则可能希望显式使用`IS`。

另请参阅

`ColumnOperators.is_not()`

```py
method is_distinct_from(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.is_distinct_from()` *方法的* `ColumnOperators`

实现`IS DISTINCT FROM`运算符。

大多数平台上会渲染“a IS DISTINCT FROM b”；在某些平台上，例如 SQLite，可能会渲染“a IS NOT b”。

```py
method is_not(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.is_not()` *方法的* `ColumnOperators`

实现`IS NOT`运算符。

通常，当与`None`值进行比较时，`IS NOT`会自动生成，该值解析为`NULL`。然而，在某些平台上，如果要与布尔值进行比较，则可能希望显式使用`IS NOT`。

从版本 1.4 开始更改：`is_not()`运算符从先前版本的`isnot()`中重命名。以前的名称仍可用于向后兼容。

另请参阅

`ColumnOperators.is_()`

```py
method is_not_distinct_from(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.is_not_distinct_from()` *方法的* `ColumnOperators`

实现`IS NOT DISTINCT FROM`运算符。

在大多数平台上呈现“a IS NOT DISTINCT FROM b”; 在某些平台上，如 SQLite 可能会呈现“a IS b”。

1.4 版更改：`is_not_distinct_from()` 运算符从之前的版本`isnot_distinct_from()` 重命名。以前的名称仍可用于向后兼容性。

```py
method isnot(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.isnot()` *方法的* `ColumnOperators`

实现 `IS NOT` 运算符。

通常，与`None`的值进行比较时会自动生成`IS NOT`，它解析为`NULL`。但是，如果在某些平台上与布尔值进行比较时，明确使用`IS NOT`可能是可取的。

1.4 版更改：`is_not()` 运算符从之前的版本`isnot()` 重命名。以前的名称仍可用于向后兼容性。

另请参阅

`ColumnOperators.is_()`

```py
method isnot_distinct_from(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.isnot_distinct_from()` *方法的* `ColumnOperators`

实现 `IS NOT DISTINCT FROM` 运算符。

在大多数平台上呈现“a IS NOT DISTINCT FROM b”; 在某些平台上，如 SQLite 可能会呈现“a IS b”。

1.4 版更改：`is_not_distinct_from()` 运算符从之前的版本`isnot_distinct_from()` 重命名。以前的名称仍可用于向后兼容性。

```py
method istartswith(other: Any, escape: str | None = None, autoescape: bool = False) → ColumnOperators
```

*继承自* `ColumnOperators.istartswith()` *方法的* `ColumnOperators`

实现 `istartswith` 运算符，例如`ColumnOperators.startswith()` 的不区分大小写版本。

生成一个 LIKE 表达式，用于对字符串值的起始部分进行不区分大小写的匹配测试：

```py
lower(column) LIKE lower(<other>) || '%'
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.istartswith("foobar"))
```

由于操作符使用 `LIKE`，存在于 <other> 表达式中的通配符 `"%"` 和 `"_"` 也会像通配符一样起作用。对于字面字符串值，可以将 `ColumnOperators.istartswith.autoescape` 标志设置为 `True`，以将这些字符的出现转义为字符串值内部的这些字符，使它们匹配为它们自身而不是通配符。或者，`ColumnOperators.istartswith.escape` 参数将建立一个给定字符作为转义字符，当目标表达式不是字面字符串时可能会有用。

参数：

+   `other` – 待比较的表达式。通常是一个普通的字符串值，但也可以是任意的 SQL 表达式。LIKE 通配符 `%` 和 `_` 默认情况下不会被转义，除非 `ColumnOperators.istartswith.autoescape` 标志被设置为 True。

+   `autoescape` –

    布尔值；当为 True 时，在 LIKE 表达式中建立一个转义字符，然后将其应用于比较值中的所有 `"%"`、`"_"` 和转义字符本身的出现，假设比较值是一个字面字符串而不是 SQL 表达式。

    诸如：

    ```py
    somecolumn.istartswith("foo%bar", autoescape=True)
    ```

    将呈现为：

    ```py
    lower(somecolumn) LIKE lower(:param) || '%' ESCAPE '/'
    ```

    其中，参数的值为 `:param`，为 `"foo/%bar"`。

+   `escape` –

    一个字符，当给定时，将以 `ESCAPE` 关键字渲染，以将该字符作为转义字符。然后可以将该字符放在 `%` 和 `_` 的前面，以允许它们作为自身而不是通配符字符。

    诸如：

    ```py
    somecolumn.istartswith("foo/%bar", escape="^")
    ```

    将呈现为：

    ```py
    lower(somecolumn) LIKE lower(:param) || '%' ESCAPE '^'
    ```

    此参数也可以与`ColumnOperators.istartswith.autoescape`组合使用：

    ```py
    somecolumn.istartswith("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上述情况下，给定的字面参数将在传递到数据库之前转换为 `"foo^%bar^^bat"`。

另见

`ColumnOperators.startswith()`

```py
method like(other: Any, escape: str | None = None) → ColumnOperators
```

*从* `ColumnOperators.like()` *方法继承*

实现 `like` 操作符。

在列上下文中，生成表达式：

```py
a LIKE other
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.like("%foobar%"))
```

参数：

+   `other` – 待比较的表达式

+   `escape` –

    可选的转义字符，渲染 `ESCAPE` 关键字，例如：

    ```py
    somecolumn.like("foo/%bar", escape="/")
    ```

另请参阅

`ColumnOperators.ilike()`

```py
attribute local_attr
```

*继承自* `AssociationProxyInstance.local_attr` *属性的* `AssociationProxyInstance`

此 `AssociationProxyInstance` 引用的 ‘local’ 类属性。

另请参阅

`AssociationProxyInstance.attr`

`AssociationProxyInstance.remote_attr`

```py
method match(other: Any, **kwargs: Any) → ColumnOperators
```

*继承自* `ColumnOperators.match()` *方法的* `ColumnOperators`

实现了特定于数据库的 ‘match’ 运算符。

`ColumnOperators.match()` 尝试解析为后端提供的类似 MATCH 的函数或运算符。例如：

+   PostgreSQL - 渲染 `x @@ plainto_tsquery(y)`

    > 从版本 2.0 开始更改：现在对于 PostgreSQL 使用 `plainto_tsquery()` 而不是 `to_tsquery()`；为了与其他形式兼容，请参阅全文搜索。

+   MySQL - 渲染 `MATCH (x) AGAINST (y IN BOOLEAN MODE)`

    另请参阅

    `match` - 具有附加功能的 MySQL 特定构造。

+   Oracle - 渲染 `CONTAINS(x, y)`

+   其他后端可能提供特殊实现。

+   没有任何特殊实现的后端将将运算符输出为 “MATCH”。例如，这与 SQLite 兼容。

```py
method not_ilike(other: Any, escape: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.not_ilike()` *方法的* `ColumnOperators`

实现 `NOT ILIKE` 运算符。

这相当于使用否定与`ColumnOperators.ilike()`，即`~x.ilike(y)`。

从版本 1.4 开始更改：`not_ilike()`操作符从先前版本的`notilike()`重新命名。先前的名称仍然可用于向后兼容。

另请参阅

`ColumnOperators.ilike()`

```py
method not_in(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.not_in()` *方法的* `ColumnOperators`

实现`NOT IN`操作符。

这相当于使用否定与`ColumnOperators.in_()`，即`~x.in_(y)`。

在`other`为空序列的情况下，编译器会产生一个“空的不包含”表达式。默认情况下，这相当于表达式“1 = 1”，以在所有情况下产生真值。可以使用`create_engine.empty_in_strategy`来更改此行为。

从版本 1.4 开始更改：`not_in()`操作符从先前版本的`notin_()`重新命名。先前的名称仍然可用于向后兼容。

从版本 1.2 开始更改：`ColumnOperators.in_()` 和 `ColumnOperators.not_in()` 操作符现在默认情况下会为空 IN 序列生成“静态”表达式。

另请参阅

`ColumnOperators.in_()`

```py
method not_like(other: Any, escape: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.not_like()` *方法的* `ColumnOperators`

实现`NOT LIKE`操作符。

这相当于使用否定与`ColumnOperators.like()`，即`~x.like(y)`。

从版本 1.4 开始更改：`not_like()`操作符从先前版本的`notlike()`重新命名。先前的名称仍然可用于向后兼容。

另请参阅

`ColumnOperators.like()`

```py
method notilike(other: Any, escape: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.notilike()` *方法的* `ColumnOperators`

实现`NOT ILIKE`运算符。

这相当于使用对`ColumnOperators.ilike()`进行否定，即 `~x.ilike(y)`。

从版本 1.4 起更改：`not_ilike()`运算符从之前的版本`notilike()`重命名。以确保向后兼容性，以前的名称仍然可用。

另请参阅

`ColumnOperators.ilike()`

```py
method notin_(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.notin_()` *方法的* `ColumnOperators`

实现`NOT IN`运算符。

这相当于使用对`ColumnOperators.in_()`进行否定，即 `~x.in_(y)`。

在`other`为空序列的情况下，编译器会生成一个“空不包含”表达式。这默认为表达式“1 = 1”，以在所有情况下产生 true。`create_engine.empty_in_strategy` 可用于更改此行为。

从版本 1.4 起更改：`not_in()`运算符从之前的版本`notin_()`重命名。以确保向后兼容性，以前的名称仍然可用。

从版本 1.2 起更改：`ColumnOperators.in_()` 和 `ColumnOperators.not_in()` 运算符现在默认情况下为一个“静态”表达式，用于空 IN 序列。

另请参阅

`ColumnOperators.in_()`

```py
method notlike(other: Any, escape: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.notlike()` *方法* `ColumnOperators`

实现 `NOT LIKE` 操作符。

这相当于使用 `ColumnOperators.like()` 的否定，即 `~x.like(y)`。

从版本 1.4 开始：`not_like()` 操作符从以前的版本中的 `notlike()` 改名为 `not_like()`。以前的名称仍然可用于向后兼容。

另请参见

`ColumnOperators.like()`

```py
method nulls_first() → ColumnOperators
```

*继承自* `ColumnOperators.nulls_first()` *方法* `ColumnOperators`

生成针对父对象的 `nulls_first()` 子句。

从版本 1.4 开始：`nulls_first()` 操作符从以前的版本中的 `nullsfirst()` 改名为 `nulls_first()`。以前的名称仍然可用于向后兼容。

```py
method nulls_last() → ColumnOperators
```

*继承自* `ColumnOperators.nulls_last()` *方法* `ColumnOperators`

生成针对父对象的 `nulls_last()` 子句。

从版本 1.4 开始：`nulls_last()` 操作符从以前的版本中的 `nullslast()` 改名为 `nulls_last()`。以前的名称仍然可用于向后兼容。

```py
method nullsfirst() → ColumnOperators
```

*继承自* `ColumnOperators.nullsfirst()` *方法* `ColumnOperators`

生成针对父对象的 `nulls_first()` 子句。

从版本 1.4 开始：`nulls_first()` 操作符从以前的版本中的 `nullsfirst()` 改名为 `nulls_first()`。以前的名称仍然可用于向后兼容。

```py
method nullslast() → ColumnOperators
```

*继承自* `ColumnOperators.nullslast()` *方法的* `ColumnOperators`

生成一个针对父对象的`nulls_last()`子句。

从版本 1.4 开始更改：`nulls_last()`运算符从先前版本的`nullslast()`重命名。以前的名称仍可用于向后兼容。

```py
method op(opstring: str, precedence: int = 0, is_comparison: bool = False, return_type: Type[TypeEngine[Any]] | TypeEngine[Any] | None = None, python_impl: Callable[..., Any] | None = None) → Callable[[Any], Operators]
```

*继承自* `Operators.op()` *方法的* `Operators`

生成一个通用的运算符函数。

例如：

```py
somecolumn.op("*")(5)
```

产生：

```py
somecolumn * 5
```

这个函数也可以用来明确地表示位运算符。例如：

```py
somecolumn.op('&')(0xff)
```

是`somecolumn`中值的按位与。

参数：

+   `opstring` – 一个字符串，将作为中缀运算符输出在此元素和传递给生成函数的表达式之间。

+   `precedence` –

    数据库在 SQL 表达式中期望应用于运算符的优先级。这个整数值作为一个提示，告诉 SQL 编译器何时应该在特定操作周围渲染显式的括号。较低的数字将导致表达式在应用于具有更高优先级的另一个运算符时被括号括起来。默认值为`0`，低于所有运算符，除了逗号（`,`）和`AS`运算符。值为 100 将高于或等于所有运算符，-100 将低于或等于所有运算符。

    另请参见

    我正在使用 op()生成自定义运算符，但我的括号没有正确显示 - SQLAlchemy SQL 编译器如何渲染括号的详细描述

+   `is_comparison` –

    legacy; 如果为 True，则该运算符将被视为“比较”运算符，即评估为布尔值的运算符，如`==`，`>`等。提供此标志是为了 ORM 关系可以在自定义连接条件中建立该运算符是比较运算符。

    使用`is_comparison`参数已被使用`Operators.bool_op()`方法取代；这个更简洁的运算符会自动设置这个参数，但也提供正确的[**PEP 484**](https://peps.python.org/pep-0484/)类型支持，因为返回的对象将表示“布尔”数据类型，即`BinaryExpression[bool]`。

+   `return_type` – `TypeEngine`类或对象，它将强制此运算符生成的表达式的返回类型为该类型。默认情况下，指定`Operators.op.is_comparison`的运算符将解析为`Boolean`，而未指定的运算符将与左操作数具有相同的类型。

+   `python_impl` –

    可选的 Python 函数，可以在与在数据库服务器上运行此运算符时相同的方式评估两个 Python 值。用于在 Python 中的 SQL 表达式评估函数，例如用于 ORM 混合属性的函数，以及用于在多行更新或删除后匹配会话中的对象的 ORM “评估器”。

    例如：

    ```py
    >>> expr = column('x').op('+', python_impl=lambda a, b: a + b)('y')
    ```

    上述表达式的运算符也适用于非 SQL 左右对象：

    ```py
    >>> expr.operator(5, 10)
    15
    ```

    自版本 2.0 新增。

另请参阅

`Operators.bool_op()`

重新定义和创建新运算符

在连接条件中使用自定义运算符

```py
method operate(op: OperatorType, *other: Any, **kwargs: Any) → Operators
```

*从* `Operators.operate()` *方法继承自* `Operators`

对参数进行操作。

这是操作的最低级别，默认情况下引发`NotImplementedError`。

在子类中覆盖此方法可以使通用行为应用于所有操作。例如，覆盖`ColumnOperators`以将`func.lower()`应用于左右两侧：

```py
class MyComparator(ColumnOperators):
    def operate(self, op, other, **kwargs):
        return op(func.lower(self), func.lower(other), **kwargs)
```

参数：

+   `op` – 运算符可调用对象。

+   `*other` – 操作的“other”一侧。对于大多数操作，它将是一个单一标量。

+   `**kwargs` – 修饰符。这些可以由特殊运算符（如`ColumnOperators.contains()`）传递。

```py
method regexp_match(pattern: Any, flags: str | None = None) → ColumnOperators
```

*从* `ColumnOperators.regexp_match()` *方法继承自* `ColumnOperators`

实现一个特定于数据库的“正则表达式匹配”运算符。

例如：

```py
stmt = select(table.c.some_column).where(
    table.c.some_column.regexp_match('^(b|c)')
)
```

`ColumnOperators.regexp_match()` 尝试解析为后端提供的类似 REGEXP 的函数或运算符，但特定的正则表达式语法和可用标志 **不是后端无关的**。

例如：

+   PostgreSQL - 当否定时渲染 `x ~ y` 或 `x !~ y`。

+   Oracle - 渲染 `REGEXP_LIKE(x, y)`

+   SQLite - 使用 SQLite 的 `REGEXP` 占位符运算符，并调用 Python 的 `re.match()` 内置函数。

+   其他后端可能提供特殊实现。

+   没有任何特殊实现的后端将发出运算符 “REGEXP” 或 “NOT REGEXP”。例如，这与 SQLite 和 MySQL 兼容。

正则表达式支持当前已为 Oracle、PostgreSQL、MySQL 和 MariaDB 实现。对于 SQLite，部分支持可用。第三方方言之间的支持可能会有所不同。

参数：

+   `pattern` – 正则表达式模式字符串或列子句。

+   `flags` – 要应用的任何正则表达式字符串标志，仅作为纯 Python 字符串传递。这些标志是后端特定的。某些后端，例如 PostgreSQL 和 MariaDB，可能会将标志作为模式的一部分来指定。在 PostgreSQL 中使用忽略大小写标志 ‘i’ 时，将使用忽略大小写正则表达式匹配运算符 `~*` 或 `!~*`。

新功能，版本 1.4。

从版本 1.4.48 更改为：2.0.18 请注意，由于实现错误，先前“flags”参数接受 SQL 表达式对象，例如列表达式，而不仅仅是纯 Python 字符串。这个实现与缓存一起使用时无法正常工作，并且已被删除；仅应传递字符串作为“flags”参数，因为这些标志在 SQL 表达式中被呈现为文字行内值。

另请参见

`ColumnOperators.regexp_replace()`

```py
method regexp_replace(pattern: Any, replacement: Any, flags: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.regexp_replace()` *方法于* `ColumnOperators`

实现特定于数据库的 “regexp 替换” 运算符。

例如：

```py
stmt = select(
    table.c.some_column.regexp_replace(
        'b(..)',
        'XY',
        flags='g'
    )
)
```

`ColumnOperators.regexp_replace()` 尝试解析为后端提供的类似 REGEXP_REPLACE 的函数，通常会发出函数 `REGEXP_REPLACE()`。但是，特定的正则表达式语法和可用标志 **不是后端无关的**。

正则表达式替换支持当前已为 Oracle、PostgreSQL、MySQL 8 或更高版本和 MariaDB 实现。第三方方言之间的支持可能会有所不同。

参数：

+   `pattern` – 正则表达式模式字符串或列子句。

+   `pattern` – 替换字符串或列子句。

+   `flags` – 要应用的任何正则表达式字符串标志，仅作为普通 Python 字符串传递。这些标志是特定于后端的。某些后端，如 PostgreSQL 和 MariaDB，可能会将标志作为模式的一部分来指定。

版本 1.4 中的新功能。

在版本 1.4.48 更改为：2.0.18 请注意，由于实现错误，“flags”参数先前接受了 SQL 表达式对象，例如列表达式，除了普通的 Python 字符串。此实现与缓存不正确，已被移除；应仅传递字符串作为“flags”参数，因为这些标志将作为 SQL 表达式中的文字内联值呈现。

另请参阅

`ColumnOperators.regexp_match()`

```py
attribute remote_attr
```

*继承自* `AssociationProxyInstance.remote_attr` *属性的* `AssociationProxyInstance`

由`AssociationProxyInstance`引用的“remote”类属性。

另请参阅

`AssociationProxyInstance.attr`

`AssociationProxyInstance.local_attr`

```py
method reverse_operate(op: OperatorType, other: Any, **kwargs: Any) → Operators
```

*继承自* `Operators.reverse_operate()` *方法的* `Operators`

对参数进行反向操作。

使用与`operate()`相同。

```py
attribute scalar
```

*继承自* `AssociationProxyInstance.scalar` *属性��* `AssociationProxyInstance`

如果此`AssociationProxyInstance`代理本地端的标量关系，则返回`True`。

```py
method startswith(other: Any, escape: str | None = None, autoescape: bool = False) → ColumnOperators
```

*继承自* `ColumnOperators.startswith()` *方法的* `ColumnOperators`

实现`startswith`操作符。

生成一个 LIKE 表达式，用于测试字符串值的起始匹配：

```py
column LIKE <other> || '%'
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.startswith("foobar"))
```

由于该操作符使用`LIKE`，因此存在于<other>表达式中的通配符字符`"%"`和`"_"`也将像通配符一样起作用。对于字面字符串值，可以将 `ColumnOperators.startswith.autoescape` 标志设置为`True`，以对字符串值内部这些字符的出现应用转义，使它们匹配为自身而不是通配符字符。或者，`ColumnOperators.startswith.escape` 参数将建立给定字符作为逃逸字符，当目标表达式不是字面字符串时，这可能会有用。

参数：

+   `other` – 要比较的表达式。通常这是一个普通字符串值，但也可以是任意的 SQL 表达式。LIKE 通配符字符`%`和`_`默认情况下不会被转义，除非设置了`ColumnOperators.startswith.autoescape` 标志为 True。

+   `autoescape` –

    boolean；当为 True 时，在 LIKE 表达式中建立一个转义字符，然后将其应用于比较值中的所有出现的`"%"`、`"_"`和转义字符本身，假定比较值是一个字面字符串而不是 SQL 表达式。

    例如：

    ```py
    somecolumn.startswith("foo%bar", autoescape=True)
    ```

    将渲染为：

    ```py
    somecolumn LIKE :param || '%' ESCAPE '/'
    ```

    具有`param`的值为`"foo/%bar"`。

+   `escape` –

    当给定的字符与`ESCAPE`关键字一起使用时，将渲染为逃逸字符。然后，可以将该字符放置在`%`和`_`的前面，以允许它们作为自身而不是通配符字符。

    例如：

    ```py
    somecolumn.startswith("foo/%bar", escape="^")
    ```

    将渲染为：

    ```py
    somecolumn LIKE :param || '%' ESCAPE '^'
    ```

    参数还可以与 `ColumnOperators.startswith.autoescape` 结合使用：

    ```py
    somecolumn.startswith("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上面的例子中，给定的字面参数将在传递到数据库之前转换为`"foo^%bar^^bat"`。

另请参阅

`ColumnOperators.endswith()`

`ColumnOperators.contains()`

`ColumnOperators.like()`

```py
attribute target_class: Type[Any]
```

此 `AssociationProxyInstance` 处理的中介类。

拦截的追加/设置/赋值事件将导致生成此类的新实例。

```py
attribute timetuple: Literal[None] = None
```

*继承自* `ColumnOperators.timetuple` *属性的* `ColumnOperators`

黑客，允许在左手边比较日期时间对象。

```py
class sqlalchemy.ext.associationproxy.ColumnAssociationProxyInstance
```

一个将数据库列作为目标的 `AssociationProxyInstance`。

**成员**

__le__(), __lt__(), __ne__(), all_(), any(), any_(), asc(), attr, between(), bitwise_and(), bitwise_lshift(), bitwise_not(), bitwise_or(), bitwise_rshift(), bitwise_xor(), bool_op(), collate(), concat(), contains(), desc(), distinct(), endswith(), has(), icontains(), iendswith(), ilike(), in_(), is_(), is_distinct_from(), is_not(), is_not_distinct_from(), isnot(), isnot_distinct_from(), istartswith(), like(), local_attr, match(), not_ilike(), not_in(), not_like(), notilike(), notin_(), notlike(), nulls_first(), nulls_last(), nullsfirst(), nullslast(), op(), operate(), regexp_match(), regexp_replace(), remote_attr, reverse_operate(), scalar, startswith(), target_class, timetuple

**类签名**

类`sqlalchemy.ext.associationproxy.ColumnAssociationProxyInstance`（`sqlalchemy.ext.associationproxy.AssociationProxyInstance`）。

```py
method __le__(other: Any) → ColumnOperators
```

*继承自* `sqlalchemy.sql.expression.ColumnOperators.__le__` *方法的* `ColumnOperators`。

实现 `<=` 运算符。

在列的上下文中，产生子句 `a <= b`。

```py
method __lt__(other: Any) → ColumnOperators
```

*继承自* `sqlalchemy.sql.expression.ColumnOperators.__lt__` *方法的* `ColumnOperators`。

实现 `<` 运算符。

在列的上下文中，产生子句 `a < b`。

```py
method __ne__(other: Any) → ColumnOperators
```

*继承自* `sqlalchemy.sql.expression.ColumnOperators.__ne__` *方法的* `ColumnOperators`。

实现 `!=` 运算符。

在列的上下文中，产生子句 `a != b`。如果目标是 `None`，则产生 `a IS NOT NULL`。

```py
method all_() → ColumnOperators
```

*继承自* `ColumnOperators.all_()` *方法的* `ColumnOperators`。

对父对象生成一个`all_()` 子句。

参见`all_()` 的文档以获取示例。

注意

一定要不要混淆新版本的 `ColumnOperators.all_()` 方法与 **遗留** 版本的该方法，即 `Comparator.all()` 方法，该方法特定于 `ARRAY`，使用不同的调用方式。

```py
method any(criterion: _ColumnExpressionArgument[bool] | None = None, **kwargs: Any) → ColumnElement[bool]
```

*继承自* `AssociationProxyInstance.any()` *方法的* `AssociationProxyInstance`。

使用 EXISTS 生成一个代理的‘any’表达式。

此表达式将使用底层代理属性的 `Comparator.any()` 和/或 `Comparator.has()` 运算符进行组合。

```py
method any_() → ColumnOperators
```

*从* `ColumnOperators` *的* `ColumnOperators.any_()` *方法继承*

生成一个针对父对象的 `any_()` 子句。

请参考 `any_()` 的文档以获取示例。

注意

请确保不要混淆较新的 `ColumnOperators.any_()` 方法与此方法的**传统**版本，即 `ARRAY` 特定的 `Comparator.any()` 方法，后者使用不同的调用风格。

```py
method asc() → ColumnOperators
```

*从* `ColumnOperators` *的* `ColumnOperators.asc()` *方法继承*

产生一个针对父对象的 `asc()` 子句。

```py
attribute attr
```

*从* `AssociationProxyInstance` *的* `AssociationProxyInstance.attr` *属性继承*

返回一个 `(local_attr, remote_attr)` 元组。

该属性最初旨在简化使用 `Query.join()` 方法一次跨两个关系进行联接，但这样做使用了一个已弃用的调用风格。

要使用 `select.join()` 或 `Query.join()` 与关联代理，当前的方法是分别使用 `AssociationProxyInstance.local_attr` 和 `AssociationProxyInstance.remote_attr` 属性：

```py
stmt = (
    select(Parent).
    join(Parent.proxied.local_attr).
    join(Parent.proxied.remote_attr)
)
```

未来的版本可能会提供更简洁的关联代理属性连接模式。

另请参阅

`AssociationProxyInstance.local_attr`

`AssociationProxyInstance.remote_attr`

```py
method between(cleft: Any, cright: Any, symmetric: bool = False) → ColumnOperators
```

*继承自* `ColumnOperators.between()` *方法的* `ColumnOperators`

对父对象生成`between()`子句，给定下限和上限范围。

```py
method bitwise_and(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_and()` *方法的* `ColumnOperators`

通过`&`操作符进行位与操作。

2.0.2 版本中新增。

另请参阅

位运算符

```py
method bitwise_lshift(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_lshift()` *方法的* `ColumnOperators`

通过`<<`操作符进行位左移操作。

2.0.2 版本中新增。

另请参阅

位运算符

```py
method bitwise_not() → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_not()` *方法的* `ColumnOperators`

通过`~`操作符进行位取反操作。

2.0.2 版本中新增。

另请参阅

位运算符

```py
method bitwise_or(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_or()` *方法的* `ColumnOperators`

通过`|`操作符进行位或操作。

2.0.2 版本中新增。

另请参阅

位运算符

```py
method bitwise_rshift(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_rshift()` *方法的* `ColumnOperators`

生成一个按位右移操作，通常通过 `>>` 运算符实现。

新版本 2.0.2 中新增。

另请参阅

按位运算符

```py
method bitwise_xor(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_xor()` *方法的* `ColumnOperators`

生成一个按位异或操作，通常通过 `^` 运算符实现，或者对于 PostgreSQL 使用 `#`。

新版本 2.0.2 中新增。

另请参阅

按位运算符

```py
method bool_op(opstring: str, precedence: int = 0, python_impl: Callable[[...], Any] | None = None) → Callable[[Any], Operators]
```

*继承自* `Operators.bool_op()` *方法的* `Operators`

返回一个自定义布尔运算符。

这个方法是调用 `Operators.op()` 并传递 `Operators.op.is_comparison` 标志为 True 的简写。 使用 `Operators.bool_op()` 的一个关键优势是，在使用列构造时，返回表达式的“布尔”性质将出现在 [**PEP 484**](https://peps.python.org/pep-0484/) 目的中。

另请参阅

`Operators.op()`

```py
method collate(collation: str) → ColumnOperators
```

*继承自* `ColumnOperators.collate()` *方法的* `ColumnOperators`

生成一个针对父对象的 `collate()` 子句，给定排序规则字符串。

另请参阅

`collate()`

```py
method concat(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.concat()` *方法* of `ColumnOperators`

实现 'concat' 运算符。

在列上下文中，生成子句 `a || b`，或者在 MySQL 上使用 `concat()` 运算符。

```py
method contains(other: Any, **kw: Any) → ColumnOperators
```

*继承自* `ColumnOperators.contains()` *方法* of `ColumnOperators`

实现 'contains' 运算符。

生成一个 LIKE 表达式，用于测试字符串值中间的匹配：

```py
column LIKE '%' || <other> || '%'
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.contains("foobar"))
```

由于运算符使用了 `LIKE`，所以在 <other> 表达式中存在的通配符字符 `"%"` 和 `"_"` 也将像通配符一样起作用。 对于字面字符串值，可以将 `ColumnOperators.contains.autoescape` 标志设置为 `True`，以对字符串值中这些字符的出现应用转义，使它们与自身匹配而不是通配符字符。 或者，`ColumnOperators.contains.escape` 参数将建立一个给定字符作为转义字符，当目标表达式不是字面字符串时，这可能会有所帮助。

参数：

+   `other` – 要比较的表达式。 这通常是一个普通字符串值，但也可以是任意的 SQL 表达式。 除非设置了 `ColumnOperators.contains.autoescape` 标志为 True，否则 LIKE 通配符字符 `%` 和 `_` 不会被转义。

+   `autoescape` –

    布尔值；当为 True 时，在 LIKE 表达式中建立转义字符，然后将其应用于比较值中所有出现的 `"%"`、`"_"` 和转义字符本身，比较值假定为字面字符串而不是 SQL 表达式。

    诸如：

    ```py
    somecolumn.contains("foo%bar", autoescape=True)
    ```

    将呈现为：

    ```py
    somecolumn LIKE '%' || :param || '%' ESCAPE '/'
    ```

    值为 `:param` 的情况下，为 `"foo/%bar"`。

+   `escape` –

    给定字符，当指定时，将使用 `ESCAPE` 关键字来将该字符建立为转义字符。 然后，可以将此字符放置在 `%` 和 `_` 的出现之前，以允许它们作为自身而不是通配符字符。

    诸如：

    ```py
    somecolumn.contains("foo/%bar", escape="^")
    ```

    将呈现为：

    ```py
    somecolumn LIKE '%' || :param || '%' ESCAPE '^'
    ```

    参数也可以与`ColumnOperators.contains.autoescape`结合使用：

    ```py
    somecolumn.contains("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上述情况中，给定的文字参数将在传递到数据库之前转换为`"foo^%bar^^bat"`。

参见

`ColumnOperators.startswith()`

`ColumnOperators.endswith()`

`ColumnOperators.like()`

```py
method desc() → ColumnOperators
```

*继承自* `ColumnOperators.desc()` *方法的* `ColumnOperators`

对父对象生成一个`desc()`子句。

```py
method distinct() → ColumnOperators
```

*继承自* `ColumnOperators.distinct()` *方法的* `ColumnOperators`

对父对象生成一个`distinct()`子句。

```py
method endswith(other: Any, escape: str | None = None, autoescape: bool = False) → ColumnOperators
```

*继承自* `ColumnOperators.endswith()` *方法的* `ColumnOperators`

实现‘endswith’运算符。

产生一个 LIKE 表达式，测试字符串值的结尾是否匹配：

```py
column LIKE '%' || <other>
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.endswith("foobar"))
```

由于该操作符使用`LIKE`，在<other>表达式中存在的通配符字符`"%"`和`"_"`也将像通配符一样起作用。对于字面字符串值，可以将`ColumnOperators.endswith.autoescape`标志设置为`True`，以对字符串值中这些字符的出现应用转义，使它们匹配为自身而不是通配符字符。或者，`ColumnOperators.endswith.escape`参数将建立一个给定字符作为转义字符，当目标表达式不是字面字符串时，这可能会有用。

参数：

+   `other` – 要进行比较的表达式。通常是一个普通的字符串值，但也可以是任意的 SQL 表达式。除非将`ColumnOperators.endswith.autoescape`标志设置为 True，否则不会转义 LIKE 通配符`%`和`_`。

+   `autoescape` –

    布尔值；当为 True 时，在 LIKE 表达式中建立一个转义字符，然后将其应用于比较值中所有的`"%"`、`"_"`和转义字符本身的出现，假定比较值是一个字面字符串而不是 SQL 表达式。

    例如：

    ```py
    somecolumn.endswith("foo%bar", autoescape=True)
    ```

    将呈现为：

    ```py
    somecolumn LIKE '%' || :param ESCAPE '/'
    ```

    值为`:param`，为`"foo/%bar"`。

+   `escape` –

    一个字符，当给定时，将使用`ESCAPE`关键字将该字符作为转义字符。然后可以将此字符放在`%`和`_`之前，以允许它们作为自身而不是通配符字符。

    例如：

    ```py
    somecolumn.endswith("foo/%bar", escape="^")
    ```

    将呈现为：

    ```py
    somecolumn LIKE '%' || :param ESCAPE '^'
    ```

    该参数也可以与`ColumnOperators.endswith.autoescape`结合使用：

    ```py
    somecolumn.endswith("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上述情况下，给定的字面参数将在传递到数据库之前转换为`"foo^%bar^^bat"`。

另请参阅

`ColumnOperators.startswith()`

`ColumnOperators.contains()`

`ColumnOperators.like()`

```py
method has(criterion: _ColumnExpressionArgument[bool] | None = None, **kwargs: Any) → ColumnElement[bool]
```

*继承自* `AssociationProxyInstance.has()` *方法的* `AssociationProxyInstance`

使用 EXISTS 生成一个代理的‘has’表达式。

此表达式将使用基础代理属性的`Comparator.any()`和/或`Comparator.has()`运算符组成的产品。

```py
method icontains(other: Any, **kw: Any) → ColumnOperators
```

*继承自* `ColumnOperators.icontains()` *方法的* `ColumnOperators`

实现`icontains`运算符，例如`ColumnOperators.contains()`的不区分大小写版本。

生成一个 LIKE 表达式，用于对字符串值中间的不区分大小写匹配进行测试：

```py
lower(column) LIKE '%' || lower(<other>) || '%'
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.icontains("foobar"))
```

由于该运算符使用`LIKE`，因此存在于<other>表达式中的通配符`"%"`和`"_"`也将像通配符一样起作用。对于字面字符串值，可以将`ColumnOperators.icontains.autoescape`标志设置为`True`，以对字符串值中这些字符的出现进行转义，使它们作为自身而不是通配符字符进行匹配。或者，`ColumnOperators.icontains.escape`参数将建立一个给定字符作为转义字符，当目标表达式不是字面字符串时，这可能会有用。

参数：

+   `other` – 要进行比较的表达式。通常是一个普通字符串值，但也可以是任意 SQL 表达式。除非将`ColumnOperators.icontains.autoescape`标志设置为 True，否则不会转义 LIKE 通配符`%`和`_`。

+   `autoescape` –

    布尔值；当为 True 时，在 LIKE 表达式中建立一个转义字符，然后将其应用于比较值中所有出现的`"%"`、`"_"`和转义字符本身，假定比较值是一个字面字符串而不是 SQL 表达式。

    例如这样的表达式：

    ```py
    somecolumn.icontains("foo%bar", autoescape=True)
    ```

    将呈现为：

    ```py
    lower(somecolumn) LIKE '%' || lower(:param) || '%' ESCAPE '/'
    ```

    使用值`:param`为`"foo/%bar"`。

+   `escape` –

    一个字符，当给定时，将与`ESCAPE`关键字一起呈现，将该字符作为转义字符。然后可以将该字符放置在`%`和`_`的前面，以使它们像自己一样工作，而不是通配符字符。

    例如以下表达式：

    ```py
    somecolumn.icontains("foo/%bar", escape="^")
    ```

    将呈现为：

    ```py
    lower(somecolumn) LIKE '%' || lower(:param) || '%' ESCAPE '^'
    ```

    此参数也可以与`ColumnOperators.contains.autoescape`组合使用：

    ```py
    somecolumn.icontains("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上述示例中，给定的文字参数在传递到数据库之前将被转换为`"foo^%bar^^bat"`。

另请参阅

`ColumnOperators.contains()`

```py
method iendswith(other: Any, escape: str | None = None, autoescape: bool = False) → ColumnOperators
```

*继承自* `ColumnOperators.iendswith()` *方法的* `ColumnOperators`

实现`iendswith`操作符，例如，`ColumnOperators.endswith()`的不区分大小写版本。

生成一个 LIKE 表达式，用于对字符串值的末尾进行不区分大小写的匹配：

```py
lower(column) LIKE '%' || lower(<other>)
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.iendswith("foobar"))
```

由于操作符使用`LIKE`，因此存在于<other>表达式中的通配符字符`"%"`和`"_"`也将像通配符一样工作。对于字面字符串值，可以将`ColumnOperators.iendswith.autoescape`标志设置为`True`，以对字符串值内的这些字符进行转义，使它们匹配为自身而不是通配符字符。或者，`ColumnOperators.iendswith.escape`参数将确定给定字符作为转义字符，当目标表达式不是字面字符串时，这可能会有用。

参数：

+   `other` – 要进行比较的表达式。这通常是一个普通的字符串值，但也可以是任意的 SQL 表达式。LIKE 通配符字符`%`和`_`默认情况下不会被转义，除非设置了`ColumnOperators.iendswith.autoescape`标志为 True。

+   `autoescape` –

    布尔值；当为 True 时，在 LIKE 表达式中建立转义字符，然后将其应用于比较值中的所有出现的`"%"`、`"_"`和转义字符本身，假定比较值是一个字面字符串而不是一个 SQL 表达式。

    诸如：

    ```py
    somecolumn.iendswith("foo%bar", autoescape=True)
    ```

    将呈现为：

    ```py
    lower(somecolumn) LIKE '%' || lower(:param) ESCAPE '/'
    ```

    具有值`：param`为`"foo/%bar"`。

+   `escape` –

    一个字符，当给定时，将呈现为带有`ESCAPE`关键字以将该字符作为转义字符。然后可以将此字符放在`%`和`_`的前面，以允许它们充当它们自己而不是通配符字符。

    诸如：

    ```py
    somecolumn.iendswith("foo/%bar", escape="^")
    ```

    将呈现为：

    ```py
    lower(somecolumn) LIKE '%' || lower(:param) ESCAPE '^'
    ```

    该参数还可以与`ColumnOperators.iendswith.autoescape`组合：

    ```py
    somecolumn.endswith("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上述情况下，给定的字面参数将在传递到数据库之前转换为`"foo^%bar^^bat"`。

另请参阅

`ColumnOperators.endswith()`

```py
method ilike(other: Any, escape: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.ilike()` *的方法* `ColumnOperators`

实现`ilike`运算符，例如，不区分大小写的 LIKE。

在列上下文中，产生一个形式为：

```py
lower(a) LIKE lower(other)
```

或在支持 ILIKE 运算符的后端上：

```py
a ILIKE other
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.ilike("%foobar%"))
```

参数：

+   `other` – 要比较的表达式

+   `escape` –

    可选的转义字符，呈现`ESCAPE`关键字，例如：

    ```py
    somecolumn.ilike("foo/%bar", escape="/")
    ```

另请参阅

`ColumnOperators.like()`

```py
method in_(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.in_()` *的方法* `ColumnOperators`

实现`in`运算符。

在列上下文中，生成子句`column IN <other>`。

给定参数`other`可能是：

+   字面值列表，例如：

    ```py
    stmt.where(column.in_([1, 2, 3]))
    ```

    在这种调用形式中，项目列表被转换为与给定列表长度相同的一组绑定参数：

    ```py
    WHERE COL IN (?, ?, ?)
    ```

+   如果比较是针对包含多个表达式的`tuple_()`，则可以提供元组列表：

    ```py
    from sqlalchemy import tuple_
    stmt.where(tuple_(col1, col2).in_([(1, 10), (2, 20), (3, 30)]))
    ```

+   空列表，例如：

    ```py
    stmt.where(column.in_([]))
    ```

    在这种调用形式中，该表达式呈现一个“空集”表达式。这些表达式针对各个后端进行了定制，并且通常试图将空的 SELECT 语句作为子查询。例如，在 SQLite 上，该表达式是：

    ```py
    WHERE col IN (SELECT 1 FROM (SELECT 1) WHERE 1!=1)
    ```

    自版本 1.4 更改：在所有情况下，空的 IN 表达式现在使用执行时生成的 SELECT 子查询。

+   可以使用绑定参数，例如`bindparam()`，如果包含`bindparam.expanding`标志：

    ```py
    stmt.where(column.in_(bindparam('value', expanding=True)))
    ```

    在这种调用形式中，该表达式呈现一个特殊的非 SQL 占位符表达式，看起来像：

    ```py
    WHERE COL IN ([EXPANDING_value])
    ```

    此占位符表达式在语句执行时被拦截，以转换为前面所示的可变数量的绑定参数形式。如果执行语句如下：

    ```py
    connection.execute(stmt, {"value": [1, 2, 3]})
    ```

    数据库将为每个值传递一个绑定参数：

    ```py
    WHERE COL IN (?, ?, ?)
    ```

    自版本 1.2 新增：添加了“扩展”绑定参数

    如果传递一个空列表，则会呈现一个特殊的“空列表”表达式，该表达式特定于正在使用的数据库。在 SQLite 上，这将是：

    ```py
    WHERE COL IN (SELECT 1 FROM (SELECT 1) WHERE 1!=1)
    ```

    自版本 1.3 新增：现在支持空列表的“扩展”绑定参数

+   一个`select()`构造，通常是一个相关的标量选择：

    ```py
    stmt.where(
        column.in_(
            select(othertable.c.y).
            where(table.c.x == othertable.c.x)
        )
    )
    ```

    在这种调用形式中，`ColumnOperators.in_()`呈现如下：

    ```py
    WHERE COL IN (SELECT othertable.y
    FROM othertable WHERE othertable.x = table.x)
    ```

参数：

**other** – 一个字面量列表，一个`select()`构造，或者一个包含设置为 True 的`bindparam()`构造，其中包括`bindparam.expanding`标志。

```py
method is_(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.is_()` *方法的* `ColumnOperators`

实现`IS`运算符。

通常，当与`None`的值进行比较时，会自动生成`IS`，这会解析为`NULL`。然而，在某些平台上，如果与布尔值进行比较，则可能希望显式使用`IS`。

另请参阅

`ColumnOperators.is_not()`

```py
method is_distinct_from(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.is_distinct_from()` *方法的* `ColumnOperators`

实现 `IS DISTINCT FROM` 操作符。

在大多数平台上渲染为 “a IS DISTINCT FROM b”；在某些平台上（如 SQLite）可能渲染为 “a IS NOT b”。

```py
method is_not(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.is_not()` *方法的* `ColumnOperators`

实现 `IS NOT` 操作符。

通常，与 `None` 值比较时会自动生成 `IS NOT`，它解析为 `NULL`。然而，在某些平台上，如果与布尔值比较，显式使用 `IS NOT` 可能更可取。

从版本 1.4 起更改：`is_not()` 操作符从先前版本的 `isnot()` 重命名。先前的名称保留以保持向后兼容性。

另请参阅

`ColumnOperators.is_()`

```py
method is_not_distinct_from(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.is_not_distinct_from()` *方法的* `ColumnOperators`

实现 `IS NOT DISTINCT FROM` 操作符。

在大多数平台上渲染为 “a IS NOT DISTINCT FROM b”；在某些平台上（如 SQLite）可能渲染为 “a IS b”。

从版本 1.4 起更改：`is_not_distinct_from()` 操作符从先前版本的 `isnot_distinct_from()` 重命名。先前的名称保留以保持向后兼容性。

```py
method isnot(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.isnot()` *方法的* `ColumnOperators`

实现 `IS NOT` 操作符。

通常，与 `None` 值比较时会自动生成 `IS NOT`，它解析为 `NULL`。然而，在某些平台上，如果与布尔值比较，显式使用 `IS NOT` 可能更可取。

从版本 1.4 起更改：`is_not()` 操作符从先前版本的 `isnot()` 重命名。先前的名称保留以保持向后兼容性。

另请参阅

`ColumnOperators.is_()`

```py
method isnot_distinct_from(other: Any) → ColumnOperators
```

继承自`ColumnOperators.isnot_distinct_from()`*方法的* `ColumnOperators`

实现`IS NOT DISTINCT FROM`运算符。

在大多数平台上呈现“a IS NOT DISTINCT FROM b”；在某些平台上，如 SQLite，可能呈现“a IS b”。

在 1.4 版中更改：`is_not_distinct_from()`运算符从先前版本的`isnot_distinct_from()`重新命名。先前的名称仍然可用以实现向后兼容性。

```py
method istartswith(other: Any, escape: str | None = None, autoescape: bool = False) → ColumnOperators
```

继承自`ColumnOperators.istartswith()`*方法的* `ColumnOperators`

实现`istartswith`运算符，例如`ColumnOperators.startswith()`的不区分大小写版本。

生成一个 LIKE 表达式，用于对字符串值的开头进行不区分大小写的匹配：

```py
lower(column) LIKE lower(<other>) || '%'
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.istartswith("foobar"))
```

由于该运算符使用了`LIKE`，因此在<other>表达式内部存在的通配符`"%"`和`"_"`也会像通配符一样起作用。对于文本字符串值，可以将`ColumnOperators.istartswith.autoescape`标志设置为`True`，以对字符串值内出现的这些字符进行转义，以便它们匹配为其自身而不是通配符。另外，`ColumnOperators.istartswith.escape`参数将建立一个给定字符作为转义字符，当目标表达式不是文本字符串时可能会有用。

参数：

+   `other` – 要比较的表达式。通常这是一个普通的字符串值，但也可以是一个任意的 SQL 表达式。除非设置了`ColumnOperators.istartswith.autoescape`标志为 True，否则`%`和`_`这两个 LIKE 通配符默认不会被转义。

+   `autoescape` –

    布尔值；当为 True 时，在 LIKE 表达式内建立转义字符，然后将其应用于比较值中的所有出现的`"%"`、`"_"`以及转义字符本身，假定比较值是一个文本字符串而不是一个 SQL 表达式。

    一个表达式，比如：

    ```py
    somecolumn.istartswith("foo%bar", autoescape=True)
    ```

    渲染为：

    ```py
    lower(somecolumn) LIKE lower(:param) || '%' ESCAPE '/'
    ```

    参数值为 `:param` 时为 `"foo/%bar"`。

+   `escape` –

    一个字符，当给定时将与 `ESCAPE` 关键字一起渲染，将该字符作为转义字符。然后可以将此字符放在 `%` 和 `_` 的前面，以使它们作为自身而不是通配符字符。

    一个表达式，比如：

    ```py
    somecolumn.istartswith("foo/%bar", escape="^")
    ```

    渲染为：

    ```py
    lower(somecolumn) LIKE lower(:param) || '%' ESCAPE '^'
    ```

    此参数也可以与 `ColumnOperators.istartswith.autoescape` 结合使用：

    ```py
    somecolumn.istartswith("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上面的例子中，给定的文字参数在传递到数据库之前会被转换为`"foo^%bar^^bat"`。

另见

`ColumnOperators.startswith()`

```py
method like(other: Any, escape: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.like()` *方法的* `ColumnOperators`

实现 `like` 运算符。

在列上下文中，产生如下表达式：

```py
a LIKE other
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.like("%foobar%"))
```

参数：

+   `other` – 待比较的表达式

+   `escape` –

    可选的转义字符，渲染为 `ESCAPE` 关键字，例如：

    ```py
    somecolumn.like("foo/%bar", escape="/")
    ```

另见

`ColumnOperators.ilike()`

```py
attribute local_attr
```

*继承自* `AssociationProxyInstance.local_attr` *属性的* `AssociationProxyInstance`

由此 `AssociationProxyInstance` 引用的 ‘local’ 类属性。

另见

`AssociationProxyInstance.attr`

`AssociationProxyInstance.remote_attr`

```py
method match(other: Any, **kwargs: Any) → ColumnOperators
```

*继承自* `ColumnOperators.match()` *方法的* `ColumnOperators`

实现了数据库特定的 ‘match’ 运算符。

`ColumnOperators.match()` 尝试解析为后端提供的 MATCH-like 函数或运算符。例如：

+   PostgreSQL - 渲染 `x @@ plainto_tsquery(y)`

    > 从版本 2.0 开始：对于 PostgreSQL，现在使用 `plainto_tsquery()` 而不是 `to_tsquery()`；有关与其他形式的兼容性，请参阅 全文搜索。

+   MySQL - 渲染 `MATCH (x) AGAINST (y IN BOOLEAN MODE)`

    另请参阅

    `match` - 具有附加功能的特定于 MySQL 的构造。

+   Oracle - 渲染 `CONTAINS(x, y)`

+   其他后端可能提供特殊的实现。

+   没有特殊实现的后端将将运算符发出为“MATCH”。例如，这与 SQLite 兼容。

```py
method not_ilike(other: Any, escape: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.not_ilike()` *方法* `ColumnOperators`

实现 `NOT ILIKE` 运算符。

这相当于使用 `ColumnOperators.ilike()` 的否定，即 `~x.ilike(y)`。

从版本 1.4 开始：`not_ilike()` 运算符从先前版本的 `notilike()` 重命名。先前的名称仍可用于向后兼容。

另请参阅

`ColumnOperators.ilike()`

```py
method not_in(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.not_in()` *方法* `ColumnOperators`

实现 `NOT IN` 运算符。

这相当于使用 `ColumnOperators.in_()` 的否定，即 `~x.in_(y)`。

如果 `other` 是一个空序列，编译器将生成一个“空不在”表达式。默认情况下，这会产生一个“1 = 1”的表达式，在所有情况下都返回 true。可以使用 `create_engine.empty_in_strategy` 来更改此行为。

从版本 1.4 开始：`not_in()` 运算符从先前版本的 `notin_()` 重命名。先前的名称仍可用于向后兼容。

从版本 1.2 开始更改：`ColumnOperators.in_()` 和 `ColumnOperators.not_in()` 运算符现在默认情况下为一个空的 IN 序列生成“静态”表达式。

另请参阅

`ColumnOperators.in_()`

```py
method not_like(other: Any, escape: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.not_like()` *方法的* `ColumnOperators`

实现`NOT LIKE`运算符。

这等同于对`ColumnOperators.like()`使用否定，即`~x.like(y)`。

从版本 1.4 开始更改：`not_like()`运算符从先前版本的`notlike()`重命名。 以确保向后兼容性，先前的名称仍然可用。

另请参阅

`ColumnOperators.like()`

```py
method notilike(other: Any, escape: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.notilike()` *方法的* `ColumnOperators`

实现`NOT ILIKE`运算符。

这等同于对`ColumnOperators.ilike()`使用否定，即`~x.ilike(y)`。

从版本 1.4 开始更改：`not_ilike()`运算符从先前版本的`notilike()`重命名。 以确保向后兼容性，先前的名称仍然可用。

另请参阅

`ColumnOperators.ilike()`

```py
method notin_(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.notin_()` *方法的* `ColumnOperators`

实现`NOT IN`运算符。

这等同于对`ColumnOperators.in_()`使用否定，即`~x.in_(y)`。

如果 `other` 是一个空序列，则编译器会生成一个“空的不在”表达式。 默认情况下，这会默认为表达式“1 = 1”，以在所有情况下产生 true。 可以使用 `create_engine.empty_in_strategy` 来更改此行为。

自版本 1.4 起：`not_in()` 操作符在之前的版本中从 `notin_()` 重命名。 以前的名称仍然可用于向后兼容。

自版本 1.2 起：`ColumnOperators.in_()` 和 `ColumnOperators.not_in()` 操作符现在默认生成一个空的 IN 序列的“静态”表达式。

另请参阅

`ColumnOperators.in_()`

```py
method notlike(other: Any, escape: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.notlike()` *方法的* `ColumnOperators`

实现 `NOT LIKE` 操作符。

这相当于使用 `ColumnOperators.like()` 的否定，即 `~x.like(y)`。

自版本 1.4 起：`not_like()` 操作符在之前的版本中从 `notlike()` 重命名。 以前的名称仍然可用于向后兼容。

另请参阅

`ColumnOperators.like()`

```py
method nulls_first() → ColumnOperators
```

*继承自* `ColumnOperators.nulls_first()` *方法的* `ColumnOperators`

产生一个针对父对象的 `nulls_first()` 子句。

自版本 1.4 起：`nulls_first()` 操作符在之前的版本中从 `nullsfirst()` 重命名。 以前的名称仍然可用于向后兼容。

```py
method nulls_last() → ColumnOperators
```

*继承自* `ColumnOperators.nulls_last()` *方法的* `ColumnOperators`

产生一个针对父对象的 `nulls_last()` 子句。

在版本 1.4 中更改：`nulls_last()` 运算符从之前的版本中的 `nullslast()` 重新命名。以前的名称仍然可用于向后兼容。

```py
method nullsfirst() → ColumnOperators
```

*从* `ColumnOperators` *的* `ColumnOperators.nullsfirst()` *方法继承*

产生一个针对父对象的 `nulls_first()` 子句。

在版本 1.4 中更改：`nulls_first()` 运算符从之前的版本中的 `nullsfirst()` 重新命名。以前的名称仍然可用于向后兼容。

```py
method nullslast() → ColumnOperators
```

*从* `ColumnOperators` *的* `ColumnOperators.nullslast()` *方法继承*

产生一个针对父对象的 `nulls_last()` 子句。

在版本 1.4 中更改：`nulls_last()` 运算符从之前的版本中的 `nullslast()` 重新命名。以前的名称仍然可用于向后兼容。

```py
method op(opstring: str, precedence: int = 0, is_comparison: bool = False, return_type: Type[TypeEngine[Any]] | TypeEngine[Any] | None = None, python_impl: Callable[..., Any] | None = None) → Callable[[Any], Operators]
```

*从* `Operators` *的* `Operators.op()` *方法继承*

生成一个通用的运算符函数。

例如：

```py
somecolumn.op("*")(5)
```

产生： 

```py
somecolumn * 5
```

这个函数也可以用来使按位运算符明确。例如：

```py
somecolumn.op('&')(0xff)
```

是值在 `somecolumn` 中的按位与。

参数：

+   `opstring` – 一个字符串，它将作为中缀运算符输出，在这个元素和传递给生成函数的表达式之间。

+   `precedence` –

    数据库预期应用于 SQL 表达式中的运算符的优先级。这个整数值充当 SQL 编译器的提示，以便知道何时应该在特定操作周围呈现显式的括号。较低的数字会导致表达式在应用于具有更高优先级的另一个运算符时被括起来。默认值 `0` 低于所有运算符，除了逗号 (`,`) 和 `AS` 运算符。值为 100 将高于或等于所有运算符，而 -100 将低于或等于所有运算符。

    另请参阅

    我正在使用 op()生成自定义操作符，但我的括号没有正确显示 - SQLAlchemy SQL 编译器如何渲染括号的详细描述

+   `is_comparison` –

    legacy；如果为 True，则将该操作符视为“比较”操作符，即评估为布尔真/假值的操作符，如`==`，`>`等。提供此标志是为了使 ORM 关系能够在自定义连接条件中使用操作符时建立该操作符为比较操作符。

    使用`is_comparison`参数已被`Operators.bool_op()`方法取代；这个更简洁的操作符会自动设置此参数，同时提供正确的[**PEP 484**](https://peps.python.org/pep-0484/)类型支持，因为返回的对象将表示“布尔”数据类型，即`BinaryExpression[bool]`。

+   `return_type` – 一个`TypeEngine`类或对象，将强制此操作符生成的表达式的返回类型为该类型。默认情况下，指定`Operators.op.is_comparison`的操作符将解析为`Boolean`，而未指定的操作符将与左操作数的类型相同。

+   `python_impl` –

    一个可选的 Python 函数，可以在数据库服务器上运行时以与此操作符相同的方式评估两个 Python 值。对于在 Python 中进行 SQL 表达式评估函数非常有用，例如用于 ORM 混合属性的函数，以及用于在多行更新或删除后匹配会话中的对象的 ORM“评估器”。

    例如：

    ```py
    >>> expr = column('x').op('+', python_impl=lambda a, b: a + b)('y')
    ```

    上述表达式的操作符也适用于非 SQL 左右对象：

    ```py
    >>> expr.operator(5, 10)
    15
    ```

    2.0 版中的新功能。

参见

`Operators.bool_op()`

重新定义和创建新操作符

在连接条件中使用自定义操作符

```py
method operate(op: OperatorType, *other: Any, **kwargs: Any) → ColumnElement[Any]
```

对参数进行操作。

这是操作的最低级别，默认情况下会引发`NotImplementedError`。

在子类上覆盖这个方法可以使常见行为应用于所有操作。例如，覆盖`ColumnOperators`以将`func.lower()`应用于左右两侧：

```py
class MyComparator(ColumnOperators):
    def operate(self, op, other, **kwargs):
        return op(func.lower(self), func.lower(other), **kwargs)
```

参数：

+   `op` – 操作符可调用。

+   `*other` – 操作的“其他”一侧。对于大多数操作，将是单个标量。

+   `**kwargs` – 修饰符。这些可以由特殊操作符传递，例如`ColumnOperators.contains()`。

```py
method regexp_match(pattern: Any, flags: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.regexp_match()` *方法的* `ColumnOperators`

实现了一个特定于数据库的‘regexp match’操作符。

例如：

```py
stmt = select(table.c.some_column).where(
    table.c.some_column.regexp_match('^(b|c)')
)
```

`ColumnOperators.regexp_match()` 尝试解析为后端提供的类似 REGEXP 的函数或操作符，但具体的正则表达式语法和可用标志**不是后端通用的**。

例如：

+   PostgreSQL - 在否定时呈现`x ~ y`或`x !~ y`。

+   Oracle - 呈现`REGEXP_LIKE(x, y)`

+   SQLite - 使用了 SQLite 的`REGEXP`占位符操作符，并调用了 Python 的`re.match()`内置函数。

+   其他后端可能提供特殊实现。

+   没有任何特殊实现的后端将发出操作符“REGEXP”或“NOT REGEXP”。例如，这与 SQLite 和 MySQL 兼容。

目前正则表达式支持已实现在 Oracle、PostgreSQL、MySQL 和 MariaDB 中。对于 SQLite，部分支持可用。第三方方言中的支持可能有所不同。

参数：

+   `pattern` – 正则表达式模式字符串或列子句。

+   `flags` – 要应用的任何正则表达式字符串标志，仅作为普通 Python 字符串传递。这些标志是特定于后端的。一些后端，如 PostgreSQL 和 MariaDB，可能会将标志作为模式的一部分指定。在 PostgreSQL 中使用忽略大小写标志‘i’时，将使用忽略大小写的正则表达式匹配操作符`~*`或`!~*`。

版本 1.4 中新增。

在版本 1.4.48 更改为：2.0.18 请注意，由于实现错误，先前的“flags”参数接受了 SQL 表达式对象，例如列表达式，而不仅仅是普通的 Python 字符串。这种实现与缓存不兼容，并已被移除；“flags”参数应该只传递字符串，因为这些标志将作为 SQL 表达式中的文字内联值呈现。

另请参阅

`ColumnOperators.regexp_replace()`

```py
method regexp_replace(pattern: Any, replacement: Any, flags: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.regexp_replace()` *方法的* `ColumnOperators`

实现了一个特定于数据库的‘regexp replace’操作符。

例如：

```py
stmt = select(
    table.c.some_column.regexp_replace(
        'b(..)',
        'XY',
        flags='g'
    )
)
```

`ColumnOperators.regexp_replace()` 尝试解析为后端提供的类似 REGEXP_REPLACE 的函数，通常会生成函数 `REGEXP_REPLACE()`。然而，具体的正则表达式语法和可用标志**与后端无关**。

目前针对 Oracle、PostgreSQL、MySQL 8 或更高版本以及 MariaDB 实现了正则表达式替换支持。第三方方言中的支持可能有所不同。

参数:

+   `pattern` – 正则表达式模式字符串或列子句。

+   `pattern` – 替换字符串或列子句。

+   `flags` – 要应用的任何正则表达式字符串标志，仅作为纯 Python 字符串传递。这些标志是后端特定的。某些后端，如 PostgreSQL 和 MariaDB，也可以将标志作为模式的一部分指定。

1.4 版中新增。

从版本 1.4.48 更改,: 2.0.18 请注意，由于实现错误，以前“flags”参数接受 SQL 表达式对象，如列表达式，而不仅仅是纯 Python 字符串。此实现与缓存不正确，已删除；仅应传递字符串作为“flags”参数，因为这些标志在 SQL 表达式中被呈现为文字内联值。

另请参阅

`ColumnOperators.regexp_match()`

```py
attribute remote_attr
```

*继承自* `AssociationProxyInstance.remote_attr` *属性的* `AssociationProxyInstance`

此 `AssociationProxyInstance` 引用的“remote”类属性。

另请参阅

`AssociationProxyInstance.attr`

`AssociationProxyInstance.local_attr`

```py
method reverse_operate(op: OperatorType, other: Any, **kwargs: Any) → Operators
```

*继承自* `Operators.reverse_operate()` *方法的* `Operators`

对参数进行反向操作。

使用方式与 `operate()` *相同*。

```py
attribute scalar
```

*继承自* `AssociationProxyInstance.scalar` *属性的* `AssociationProxyInstance`

如果此 `AssociationProxyInstance` 代理一个本地方的标量关系，则返回 `True`。

```py
method startswith(other: Any, escape: str | None = None, autoescape: bool = False) → ColumnOperators
```

*继承自* `ColumnOperators.startswith()` *方法的* `ColumnOperators`

实现 `startswith` 运算符。

产生一个 LIKE 表达式，用于测试字符串值的起始匹配：

```py
column LIKE <other> || '%'
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.startswith("foobar"))
```

由于该运算符使用 `LIKE`，所以存在于 <other> 表达式内部的通配符字符 `"%"` 和 `"_"` 也将像通配符一样工作。对于文字字符串值，可以将 `ColumnOperators.startswith.autoescape` 标志设置为 `True`，以将这些字符的出现转义为它们自身，而不是作为通配符字符进行匹配。或者，`ColumnOperators.startswith.escape` 参数将建立一个给定的字符作为转义字符，当目标表达式不是文字字符串时可能会有用。

参数：

+   `other` – 待比较的表达式。通常是一个普通字符串值，但也可以是任意的 SQL 表达式。LIKE 通配符字符 `%` 和 `_` 默认情况下不会被转义，除非 `ColumnOperators.startswith.autoescape` 标志设置为 True。

+   `autoescape` –

    布尔值；当为 True 时，在 LIKE 表达式中建立转义字符，然后将其应用于比较值中所有出现的 `"%"`、`"_"` 和转义字符本身，假设比较值是一个文字字符串而不是一个 SQL 表达式。

    一个表达式，例如：

    ```py
    somecolumn.startswith("foo%bar", autoescape=True)
    ```

    将呈现为：

    ```py
    somecolumn LIKE :param || '%' ESCAPE '/'
    ```

    以 `:param` 的值为 `"foo/%bar"`。

+   `escape` –

    一个给定的字符，当给出时将使用 `ESCAPE` 关键字来建立该字符作为转义字符。然后可以将此字符放置在 `%` 和 `_` 的前面，以允许它们像自己一样起作用，而不是通配符字符。

    一个表达式，例如：

    ```py
    somecolumn.startswith("foo/%bar", escape="^")
    ```

    将呈现为：

    ```py
    somecolumn LIKE :param || '%' ESCAPE '^'
    ```

    参数也可以与`ColumnOperators.startswith.autoescape`结合使用：

    ```py
    somecolumn.startswith("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上述情况下，给定的文字参数将在传递到数据库之前转换为`"foo^%bar^^bat"`。

另请参阅

`ColumnOperators.endswith()`

`ColumnOperators.contains()`

`ColumnOperators.like()`

```py
attribute target_class: Type[Any]
```

由此`AssociationProxyInstance`处理的中间类。

拦截的追加/设置/赋值事件将导致生成此类的新实例。

```py
attribute timetuple: Literal[None] = None
```

*继承自* `ColumnOperators` *的* `ColumnOperators.timetuple` *属性*

Hack，允许在左侧比较日期时间对象。

```py
class sqlalchemy.ext.associationproxy.AssociationProxyExtensionType
```

一个枚举。

**成员**

ASSOCIATION_PROXY

**类签名**

类`sqlalchemy.ext.associationproxy.AssociationProxyExtensionType`（`sqlalchemy.orm.base.InspectionAttrExtensionType`）

```py
attribute ASSOCIATION_PROXY = 'ASSOCIATION_PROXY'
```

表示一个`InspectionAttr`的符号，其类型为`AssociationProxy`。

赋予`InspectionAttr.extension_type`属性。

## 简化标量集合

考虑两个类`User`和`Keyword`之间的多对多映射。每个`User`可以拥有任意数量的`Keyword`对象，反之亦然（多对多模式在多对多中有描述）。下面的示例以相同的��式说明了这种模式，只是在`User`类中添加了一个名为`User.keywords`的额外属性：

```py
from __future__ import annotations

from typing import Final
from typing import List

from sqlalchemy import Column
from sqlalchemy import ForeignKey
from sqlalchemy import Integer
from sqlalchemy import String
from sqlalchemy import Table
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship
from sqlalchemy.ext.associationproxy import association_proxy
from sqlalchemy.ext.associationproxy import AssociationProxy

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "user"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(64))
    kw: Mapped[List[Keyword]] = relationship(secondary=lambda: user_keyword_table)

    def __init__(self, name: str):
        self.name = name

    # proxy the 'keyword' attribute from the 'kw' relationship
    keywords: AssociationProxy[List[str]] = association_proxy("kw", "keyword")

class Keyword(Base):
    __tablename__ = "keyword"
    id: Mapped[int] = mapped_column(primary_key=True)
    keyword: Mapped[str] = mapped_column(String(64))

    def __init__(self, keyword: str):
        self.keyword = keyword

user_keyword_table: Final[Table] = Table(
    "user_keyword",
    Base.metadata,
    Column("user_id", Integer, ForeignKey("user.id"), primary_key=True),
    Column("keyword_id", Integer, ForeignKey("keyword.id"), primary_key=True),
)
```

在上面的示例中，`association_proxy()`应用于`User`类，以生成`kw`关系的“视图”，该视图公开与每个`Keyword`对象关联的`.keyword`的字符串值。当向集合添加字符串时，它还会透明地创建新的`Keyword`对象：

```py
>>> user = User("jek")
>>> user.keywords.append("cheese-inspector")
>>> user.keywords.append("snack-ninja")
>>> print(user.keywords)
['cheese-inspector', 'snack-ninja']
```

要理解这一机制，首先回顾一下在不使用`.keywords`关联代理的情况下，`User`和`Keyword`的行为。通常，读取和操作与`User`相关联的“关键词”字符串集合需要从每个集合元素遍历到`.keyword`属性，这可能很麻烦。下面的示例说明了在不使用关联代理的情况下应用的相同一系列操作：

```py
>>> # identical operations without using the association proxy
>>> user = User("jek")
>>> user.kw.append(Keyword("cheese-inspector"))
>>> user.kw.append(Keyword("snack-ninja"))
>>> print([keyword.keyword for keyword in user.kw])
['cheese-inspector', 'snack-ninja']
```

`association_proxy()`函数产生的`AssociationProxy`对象是[Python 描述符](https://docs.python.org/howto/descriptor.html)的一个实例，并且不以任何方式被`Mapper`“映射”。因此，无论是使用 Declarative 还是 Imperative 映射，它都始终在映射类的类定义中内联指示。

该代理通过响应操作来操作底层的映射属性或集合，并且通过代理进行的更改立即反映在映射属性中，反之亦然。底层属性仍然可以完全访问。

当首次访问时，关联代理会对目标集合执行内省操作，以便其行为正确对应。诸如本地代理的属性是否为集合（通常情况下）或标量引用，以及集合是否像集合、列表或字典一样操作等细节都会考虑在内，以便代理的行为应该与底层集合或属性的行为一样。

### 创造新价值

当关联代理拦截到列表`append()`事件（或集合`add()`，字典`__setitem__()`或标量赋值事件）时，它会使用其构造函数实例化“中介”对象的新实例，将给定值作为单个参数传递。在我们上面的示例中，像下面这样的操作：

```py
user.keywords.append("cheese-inspector")
```

被关联代理翻译成操作：

```py
user.kw.append(Keyword("cheese-inspector"))
```

这个示例在这里起作用，因为我们设计了`Keyword`的构造函数以接受一个单一的位置参数，`keyword`。 对于那些单参数构造函数不可行的情况，可以使用`association_proxy.creator`参数自定义关联代理的创建行为，该参数引用一个可调用对象（即 Python 函数），该对象将根据单个参数生成一个新的对象实例。 下面我们使用通常的 lambda 来说明这一点：

```py
class User(Base):
    ...

    # use Keyword(keyword=kw) on append() events
    keywords: AssociationProxy[List[str]] = association_proxy(
        "kw", "keyword", creator=lambda kw: Keyword(keyword=kw)
    )
```

在列表或集合类型的集合或标量属性的情况下，`creator`函数接受一个参数。 在基于字典的集合的情况下，它接受两个参数，“key”和“value”。 下面的示例在代理到基于字典的集合中给出。

当关联代理拦截到列表`append()`事件（或集合`add()`，字典`__setitem__()`或标量赋值事件）时，它会使用其构造函数实例化一个新的“中间”对象的实例，将给定的值作为单个参数传递。 在上面的示例中，像这样的操作：

```py
user.keywords.append("cheese-inspector")
```

由关联代理转换为操作：

```py
user.kw.append(Keyword("cheese-inspector"))
```

这个示例在这里起作用，因为我们设计了`Keyword`的构造函数以接受一个单一的位置参数，`keyword`。 对于那些单参数构造函数不可行的情况，可以使用`association_proxy.creator`参数自定义关联代理的创建行为，该参数引用一个可调用对象（即 Python 函数），该对象将根据单个参数生成一个新的对象实例。 下面我们使用通常的 lambda 来说明这一点：

```py
class User(Base):
    ...

    # use Keyword(keyword=kw) on append() events
    keywords: AssociationProxy[List[str]] = association_proxy(
        "kw", "keyword", creator=lambda kw: Keyword(keyword=kw)
    )
```

在列表或集合类型的集合或标量属性的情况下，`creator`函数接受一个参数。 在基于字典的集合的情况下，它接受两个参数，“key”和“value”。 下面的示例在代理到基于字典的集合中给出。

## 简化关联对象

“关联对象”模式是多对多关系的扩展形式，并在关联对象中进行了描述。 在常规使用过程中，关联代理对于保持“关联对象”不被干扰非常有用。

假设我们上面的`user_keyword`表有额外的列，我们希望显式映射这些列，但在大多数情况下我们不需要直接访问这些属性。下面，我们展示一个新的映射，引入了`UserKeywordAssociation`类，该类映射到前面展示的`user_keyword`表。这个类添加了一个额外的列`special_key`，我们偶尔需要访问这个值，但通常不需要。我们在`User`类上创建了一个名为`keywords`的关联代理，它将连接`User`的`user_keyword_associations`集合与每个`UserKeywordAssociation`上存在的`.keyword`属性之间的差距：

```py
from __future__ import annotations

from typing import List
from typing import Optional

from sqlalchemy import ForeignKey
from sqlalchemy import String
from sqlalchemy.ext.associationproxy import association_proxy
from sqlalchemy.ext.associationproxy import AssociationProxy
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "user"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(64))

    user_keyword_associations: Mapped[List[UserKeywordAssociation]] = relationship(
        back_populates="user",
        cascade="all, delete-orphan",
    )

    # association proxy of "user_keyword_associations" collection
    # to "keyword" attribute
    keywords: AssociationProxy[List[Keyword]] = association_proxy(
        "user_keyword_associations",
        "keyword",
        creator=lambda keyword_obj: UserKeywordAssociation(keyword=keyword_obj),
    )

    def __init__(self, name: str):
        self.name = name

class UserKeywordAssociation(Base):
    __tablename__ = "user_keyword"
    user_id: Mapped[int] = mapped_column(ForeignKey("user.id"), primary_key=True)
    keyword_id: Mapped[int] = mapped_column(ForeignKey("keyword.id"), primary_key=True)
    special_key: Mapped[Optional[str]] = mapped_column(String(50))

    user: Mapped[User] = relationship(back_populates="user_keyword_associations")

    keyword: Mapped[Keyword] = relationship()

class Keyword(Base):
    __tablename__ = "keyword"
    id: Mapped[int] = mapped_column(primary_key=True)
    keyword: Mapped[str] = mapped_column("keyword", String(64))

    def __init__(self, keyword: str):
        self.keyword = keyword

    def __repr__(self) -> str:
        return f"Keyword({self.keyword!r})"
```

使用上述配置，我们可以操作每个`User`对象的`.keywords`集合，每个对象都公开了从底层`UserKeywordAssociation`元素获取的`Keyword`对象集合：

```py
>>> user = User("log")
>>> for kw in (Keyword("new_from_blammo"), Keyword("its_big")):
...     user.keywords.append(kw)
>>> print(user.keywords)
[Keyword('new_from_blammo'), Keyword('its_big')]
```

这个例子与之前在 Simplifying Scalar Collections 中展示的例子形成对比，在那个例子中，关联代理公开了一个字符串集合，而不是一个组合对象集合。在这种情况下，每个`.keywords.append()`操作等同于：

```py
>>> user.user_keyword_associations.append(
...     UserKeywordAssociation(keyword=Keyword("its_heavy"))
... )
```

`UserKeywordAssociation`对象有两个属性，这两个属性都在关联代理的`append()`操作范围内填充；`.keyword`指的是`Keyword`对象，`.user`指的是`User`对象。首先填充`.keyword`属性，因为关联代理响应`.append()`操作生成一个新的`UserKeywordAssociation`对象，将给定的`Keyword`实例分配给`.keyword`属性。然后，由于`UserKeywordAssociation`对象被追加到`User.user_keyword_associations`集合中，为`User.user_keyword_associations`配置为`back_populates`的`UserKeywordAssociation.user`属性在给定的`UserKeywordAssociation`实例上初始化，以指向接收追加操作的父`User`。上面的`special_key`参数保持其默认值为`None`。

对于那些我们确实希望`special_key`有一个值的情况，我们显式创建`UserKeywordAssociation`对象。下面我们分配了所有三个属性，其中在构造过程中分配`.user`的效果是将新的`UserKeywordAssociation`追加到`User.user_keyword_associations`集合（通过关系）：

```py
>>> UserKeywordAssociation(
...     keyword=Keyword("its_wood"), user=user, special_key="my special key"
... )
```

关联代理通过以下所有操作返回给我们一个由`Keyword`对象表示的集合：

```py
>>> print(user.keywords)
[Keyword('new_from_blammo'), Keyword('its_big'), Keyword('its_heavy'), Keyword('its_wood')]
```

## 代理到基于字典的集合

关联代理也可以代理基于字典的集合。SQLAlchemy 映射通常使用`attribute_keyed_dict()`集合类型来创建字典集合，以及自定义基于字典的集合中描述的扩展技术。

当检测到使用基于字典的集合时，关联代理会调整其行为。当新值添加到字典中时，关联代理通过将两个参数传递给创建函数而不是一个参数来实例化中间对象，即键和值。与往常一样，这个创建函数默认为中间类的构造函数，并且可以使用`creator`参数进行定制。

下面，我们修改了我们的`UserKeywordAssociation`示例，使得`User.user_keyword_associations`集合现在将使用字典映射，其中`UserKeywordAssociation.special_key`参数将用作字典的键。我们还将`User.keywords`代理应用了`creator`参数，以便在向字典添加新元素时适当地分配这些值：

```py
from __future__ import annotations
from typing import Dict

from sqlalchemy import ForeignKey
from sqlalchemy import String
from sqlalchemy.ext.associationproxy import association_proxy
from sqlalchemy.ext.associationproxy import AssociationProxy
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship
from sqlalchemy.orm.collections import attribute_keyed_dict

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "user"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(64))

    # user/user_keyword_associations relationship, mapping
    # user_keyword_associations with a dictionary against "special_key" as key.
    user_keyword_associations: Mapped[Dict[str, UserKeywordAssociation]] = relationship(
        back_populates="user",
        collection_class=attribute_keyed_dict("special_key"),
        cascade="all, delete-orphan",
    )
    # proxy to 'user_keyword_associations', instantiating
    # UserKeywordAssociation assigning the new key to 'special_key',
    # values to 'keyword'.
    keywords: AssociationProxy[Dict[str, Keyword]] = association_proxy(
        "user_keyword_associations",
        "keyword",
        creator=lambda k, v: UserKeywordAssociation(special_key=k, keyword=v),
    )

    def __init__(self, name: str):
        self.name = name

class UserKeywordAssociation(Base):
    __tablename__ = "user_keyword"
    user_id: Mapped[int] = mapped_column(ForeignKey("user.id"), primary_key=True)
    keyword_id: Mapped[int] = mapped_column(ForeignKey("keyword.id"), primary_key=True)
    special_key: Mapped[str]

    user: Mapped[User] = relationship(
        back_populates="user_keyword_associations",
    )
    keyword: Mapped[Keyword] = relationship()

class Keyword(Base):
    __tablename__ = "keyword"
    id: Mapped[int] = mapped_column(primary_key=True)
    keyword: Mapped[str] = mapped_column(String(64))

    def __init__(self, keyword: str):
        self.keyword = keyword

    def __repr__(self) -> str:
        return f"Keyword({self.keyword!r})"
```

我们将`.keywords`集合说明为一个字典，将`UserKeywordAssociation.special_key`值映射到`Keyword`对象：

```py
>>> user = User("log")

>>> user.keywords["sk1"] = Keyword("kw1")
>>> user.keywords["sk2"] = Keyword("kw2")

>>> print(user.keywords)
{'sk1': Keyword('kw1'), 'sk2': Keyword('kw2')}
```

## 组合关联代理

考虑到我们之前的从关系到标量属性的代理示例，跨关联对象进行代理，以及代理字典的示例，我们可以将所有三种技术结合起来，为`User`提供一个严格处理`special_key`字符串值映射到字符串`keyword`的`keywords`字典。`UserKeywordAssociation`和`Keyword`类都完全隐藏了。这是通过在`User`上构建一个关联代理来实现的，该代理指向`UserKeywordAssociation`上存在的关联代理：

```py
from __future__ import annotations

from sqlalchemy import ForeignKey
from sqlalchemy import String
from sqlalchemy.ext.associationproxy import association_proxy
from sqlalchemy.ext.associationproxy import AssociationProxy
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship
from sqlalchemy.orm.collections import attribute_keyed_dict

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "user"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(64))

    user_keyword_associations: Mapped[Dict[str, UserKeywordAssociation]] = relationship(
        back_populates="user",
        collection_class=attribute_keyed_dict("special_key"),
        cascade="all, delete-orphan",
    )
    # the same 'user_keyword_associations'->'keyword' proxy as in
    # the basic dictionary example.
    keywords: AssociationProxy[Dict[str, str]] = association_proxy(
        "user_keyword_associations",
        "keyword",
        creator=lambda k, v: UserKeywordAssociation(special_key=k, keyword=v),
    )

    def __init__(self, name: str):
        self.name = name

class UserKeywordAssociation(Base):
    __tablename__ = "user_keyword"
    user_id: Mapped[int] = mapped_column(ForeignKey("user.id"), primary_key=True)
    keyword_id: Mapped[int] = mapped_column(ForeignKey("keyword.id"), primary_key=True)
    special_key: Mapped[str] = mapped_column(String(64))
    user: Mapped[User] = relationship(
        back_populates="user_keyword_associations",
    )

    # the relationship to Keyword is now called
    # 'kw'
    kw: Mapped[Keyword] = relationship()

    # 'keyword' is changed to be a proxy to the
    # 'keyword' attribute of 'Keyword'
    keyword: AssociationProxy[Dict[str, str]] = association_proxy("kw", "keyword")

class Keyword(Base):
    __tablename__ = "keyword"
    id: Mapped[int] = mapped_column(primary_key=True)
    keyword: Mapped[str] = mapped_column(String(64))

    def __init__(self, keyword: str):
        self.keyword = keyword
```

`User.keywords`现在是一个字符串到字符串的字典，其中`UserKeywordAssociation`和`Keyword`对象被透明地创建和删除，使用关联代理。在下面的示例中，我们说明了使用赋值运算符的用法，这也由关联代理适当处理，一次将字典值应用到集合中：

```py
>>> user = User("log")
>>> user.keywords = {"sk1": "kw1", "sk2": "kw2"}
>>> print(user.keywords)
{'sk1': 'kw1', 'sk2': 'kw2'}

>>> user.keywords["sk3"] = "kw3"
>>> del user.keywords["sk2"]
>>> print(user.keywords)
{'sk1': 'kw1', 'sk3': 'kw3'}

>>> # illustrate un-proxied usage
... print(user.user_keyword_associations["sk3"].kw)
<__main__.Keyword object at 0x12ceb90>
```

上面示例的一个注意事项是，因为对每个字典设置操作都会创建`Keyword`对象，所以示例无法保持`Keyword`对象在其字符串名称上的唯一性，这是像这样的标记场景的典型要求。对于这种用例，推荐使用[UniqueObject](https://www.sqlalchemy.org/trac/wiki/UsageRecipes/UniqueObject)这样的配方，或者类似的创建策略，它将对`Keyword`类的构造函数应用“先查找，然后创建”的策略，以便如果给定名称已经存在，则返回已存在的`Keyword`。

## 使用关联代理进行查询

`AssociationProxy` 具有简单的 SQL 构建能力，其工作方式类似于其他 ORM 映射的属性，并提供基于 SQL `EXISTS` 关键字的基本过滤支持。

注意

关联代理扩展的主要目的是允许改进对已加载的映射对象实例的持久性和对象访问模式。类绑定查询功能的用途有限，并不会取代在构建具有 JOIN、急加载选项等 SQL 查询时引用底层属性的需要。

对于这一部分，请假设一个既有关联代理指向列，又有关联代理指向相关对象的类，就像下面的示例映射一样：

```py
from __future__ import annotations
from sqlalchemy import Column, ForeignKey, Integer, String
from sqlalchemy.ext.associationproxy import association_proxy, AssociationProxy
from sqlalchemy.orm import DeclarativeBase, relationship
from sqlalchemy.orm.collections import attribute_keyed_dict
from sqlalchemy.orm.collections import Mapped

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "user"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(64))

    user_keyword_associations: Mapped[UserKeywordAssociation] = relationship(
        cascade="all, delete-orphan",
    )

    # object-targeted association proxy
    keywords: AssociationProxy[List[Keyword]] = association_proxy(
        "user_keyword_associations",
        "keyword",
    )

    # column-targeted association proxy
    special_keys: AssociationProxy[List[str]] = association_proxy(
        "user_keyword_associations", "special_key"
    )

class UserKeywordAssociation(Base):
    __tablename__ = "user_keyword"
    user_id: Mapped[int] = mapped_column(ForeignKey("user.id"), primary_key=True)
    keyword_id: Mapped[int] = mapped_column(ForeignKey("keyword.id"), primary_key=True)
    special_key: Mapped[str] = mapped_column(String(64))
    keyword: Mapped[Keyword] = relationship()

class Keyword(Base):
    __tablename__ = "keyword"
    id: Mapped[int] = mapped_column(primary_key=True)
    keyword: Mapped[str] = mapped_column(String(64))
```

生成的 SQL 采用针对 EXISTS SQL 操作符的相关子查询形式，以便可以在不需要对封闭查询进行其他修改的情况下在 WHERE 子句中使用。如果关联代理的直接目标是**映射的列表达式**，则可以使用标准列操作符，这些操作符将嵌入在子查询中。例如，一个直接的等式操作符：

```py
>>> print(session.scalars(select(User).where(User.special_keys == "jek")))
SELECT  "user".id  AS  user_id,  "user".name  AS  user_name
FROM  "user"
WHERE  EXISTS  (SELECT  1
FROM  user_keyword
WHERE  "user".id  =  user_keyword.user_id  AND  user_keyword.special_key  =  :special_key_1) 
```

一个 LIKE 操作符：

```py
>>> print(session.scalars(select(User).where(User.special_keys.like("%jek"))))
SELECT  "user".id  AS  user_id,  "user".name  AS  user_name
FROM  "user"
WHERE  EXISTS  (SELECT  1
FROM  user_keyword
WHERE  "user".id  =  user_keyword.user_id  AND  user_keyword.special_key  LIKE  :special_key_1) 
```

对于关联代理，其直接目标是**相关对象或集合，或相关对象上的另一个关联代理或属性**的情况，可以使用与关系相关的操作符，例如`PropComparator.has()`和`PropComparator.any()`。`User.keywords`属性实际上是两个关联代理链接在一起，因此在使用该代理生成 SQL 短语时，我们得到两个级别的 EXISTS 子查询：

```py
>>> print(session.scalars(select(User).where(User.keywords.any(Keyword.keyword == "jek"))))
SELECT  "user".id  AS  user_id,  "user".name  AS  user_name
FROM  "user"
WHERE  EXISTS  (SELECT  1
FROM  user_keyword
WHERE  "user".id  =  user_keyword.user_id  AND  (EXISTS  (SELECT  1
FROM  keyword
WHERE  keyword.id  =  user_keyword.keyword_id  AND  keyword.keyword  =  :keyword_1))) 
```

这不是最有效的 SQL 形式，因此虽然关联代理可以方便快速生成 WHERE 条件，但应该检查 SQL 结果并将其“展开”为显式 JOIN 条件以获得最佳使用，特别是当将关联代理链接在一起时。

版本 1.3 中的更改：根据目标类型，关联代理现在提供不同的查询模式。请参阅 AssociationProxy 现在为面向列的目标提供标准列操作符。

## 级联标量删除

版本 1.3 中的新增功能。

给定一个映射如下：

```py
from __future__ import annotations
from sqlalchemy import Column, ForeignKey, Integer, String
from sqlalchemy.ext.associationproxy import association_proxy, AssociationProxy
from sqlalchemy.orm import DeclarativeBase, relationship
from sqlalchemy.orm.collections import attribute_keyed_dict
from sqlalchemy.orm.collections import Mapped

class Base(DeclarativeBase):
    pass

class A(Base):
    __tablename__ = "test_a"
    id: Mapped[int] = mapped_column(primary_key=True)
    ab: Mapped[AB] = relationship(uselist=False)
    b: AssociationProxy[B] = association_proxy(
        "ab", "b", creator=lambda b: AB(b=b), cascade_scalar_deletes=True
    )

class B(Base):
    __tablename__ = "test_b"
    id: Mapped[int] = mapped_column(primary_key=True)

class AB(Base):
    __tablename__ = "test_ab"
    a_id: Mapped[int] = mapped_column(ForeignKey(A.id), primary_key=True)
    b_id: Mapped[int] = mapped_column(ForeignKey(B.id), primary_key=True)

    b: Mapped[B] = relationship()
```

对 `A.b` 的赋值将生成一个 `AB` 对象：

```py
a.b = B()
```

`A.b` 关联是标量的，并包括使用参数 `AssociationProxy.cascade_scalar_deletes`。当启用此参数时，将`A.b`设置为 `None` 将同时删除 `A.ab`：

```py
a.b = None
assert a.ab is None
```

当 `AssociationProxy.cascade_scalar_deletes` 未设置时，上述关联对象 `a.ab` 会保持不变。

请注意，这不是基于集合的关联代理的行为；在这种情况下，当代理集合的成员被移除时，中介关联对象始终会被移除。行是否被删除取决于关联级联设置。

另请参阅

级联

## 标量关系

下面的示例说明了在一对多关系的多方上使用关联代理，以访问标量对象的属性：

```py
from __future__ import annotations

from typing import List

from sqlalchemy import ForeignKey
from sqlalchemy import String
from sqlalchemy.ext.associationproxy import association_proxy
from sqlalchemy.ext.associationproxy import AssociationProxy
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

class Recipe(Base):
    __tablename__ = "recipe"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(64))

    steps: Mapped[List[Step]] = relationship(back_populates="recipe")
    step_descriptions: AssociationProxy[List[str]] = association_proxy(
        "steps", "description"
    )

class Step(Base):
    __tablename__ = "step"
    id: Mapped[int] = mapped_column(primary_key=True)
    description: Mapped[str]
    recipe_id: Mapped[int] = mapped_column(ForeignKey("recipe.id"))
    recipe: Mapped[Recipe] = relationship(back_populates="steps")

    recipe_name: AssociationProxy[str] = association_proxy("recipe", "name")

    def __init__(self, description: str) -> None:
        self.description = description

my_snack = Recipe(
    name="afternoon snack",
    step_descriptions=[
        "slice bread",
        "spread peanut butted",
        "eat sandwich",
    ],
)
```

使用以下命令可打印 `my_snack` 的摘要步骤：

```py
>>> for i, step in enumerate(my_snack.steps, 1):
...     print(f"Step {i} of {step.recipe_name!r}: {step.description}")
Step 1 of 'afternoon snack': slice bread
Step 2 of 'afternoon snack': spread peanut butted
Step 3 of 'afternoon snack': eat sandwich
```

## API 文档

| 对象名称 | 描述 |
| --- | --- |
| association_proxy(target_collection, attr, *, [creator, getset_factory, proxy_factory, proxy_bulk_set, info, cascade_scalar_deletes, create_on_none_assignment, init, repr, default, default_factory, compare, kw_only]) | 返回一个实现视图的 Python 属性，该视图引用目标的成员上的属性。 |
| AssociationProxy | 一个描述符，提供对象属性的读/写视图。 |
| AssociationProxyExtensionType | 一个枚举类型。 |
| AssociationProxyInstance | 一个每个类的对象，用于提供类和对象特定的结果。 |
| ColumnAssociationProxyInstance | 一个 `AssociationProxyInstance`，其目标为数据库列。 |
| ObjectAssociationProxyInstance | 一个 `AssociationProxyInstance`，其目标为对象。 |

```py
function sqlalchemy.ext.associationproxy.association_proxy(target_collection: str, attr: str, *, creator: _CreatorProtocol | None = None, getset_factory: _GetSetFactoryProtocol | None = None, proxy_factory: _ProxyFactoryProtocol | None = None, proxy_bulk_set: _ProxyBulkSetProtocol | None = None, info: _InfoType | None = None, cascade_scalar_deletes: bool = False, create_on_none_assignment: bool = False, init: _NoArg | bool = _NoArg.NO_ARG, repr: _NoArg | bool = _NoArg.NO_ARG, default: Any | None = _NoArg.NO_ARG, default_factory: _NoArg | Callable[[], _T] = _NoArg.NO_ARG, compare: _NoArg | bool = _NoArg.NO_ARG, kw_only: _NoArg | bool = _NoArg.NO_ARG) → AssociationProxy[Any]
```

返回一个实现视图的 Python 属性，该视图引用目标的成员上的属性。

返回的值是 `AssociationProxy` 的一个实例。

实现一个代表关系的 Python 属性，作为一组更简单值的集合，或一个标量值。代理属性将模仿目标的集合类型（list、dict 或 set），或者在一对一关系的情况下，一个简单的标量值。

参数：

+   `target_collection` – 目标属性的名称。该属性通常由`relationship()`映射，以链接到目标集合，但也可以是一对多或非标量关系。

+   `attr` – 关联实例上可用于目标对象实例的属性。

+   `creator` –

    可选。

    定义了向代理集合添加新项时的自定义行为。

    默认情况下，向集合添加新项将触发构造目标对象的实例，将给定项作为位置参数传递给目标构造函数。对于这种情况不足的情况，`association_proxy.creator`可以提供一个可调用对象，该对象将以适当的方式构造对象，给定传递的项。

    对于列表和集合导向的集合，将一个参数传递给可调用对象。对于字典导向的集合，将传递两个参数，对应于键和值。

    `association_proxy.creator`可调用对象也会为标量（即一对多，一对一）关系调用。如果目标关系属性的当前值为`None`，则使用可调用对象构造一个新对象。如果对象值已经存在，则给定的属性值将填充到该对象上。

    另请参见

    创建新值

+   `cascade_scalar_deletes` –

    当为 True 时，表示将代理值设置为`None`，或通过`del`删除时，也应该删除源对象。仅适用于标量属性。通常，删除代理目标不会删除代理源，因为该对象可能还有其他状态需要保留。

    新版本 1.3 中新增。

    另请参见

    级联标量删除 - 完整的使用示例

+   `create_on_none_assignment` –

    当为 True 时，表示将代理值设置为`None`应该**创建**源对象（如果不存在），使用创建者。仅适用于标量属性。这与`assocation_proxy.cascade_scalar_deletes`是互斥的。

    新版本 2.0.18 中新增。

+   `init` –

    特定于声明性数据类映射，指定映射属性是否应该是由数据类过程生成的`__init__()`方法的一部分。

    新版本 2.0.0b4 中新增。

+   `repr` –

    特定于 声明性数据类映射，指定由此 `AssociationProxy` 建立的属性是否应作为数据类过程生成的 `__repr__()` 方法的一部分。

    新功能，版本为 2.0.0b4。

+   `default_factory` –

    特定于 声明性数据类映射，指定作为数据类过程的一部分进行的默认值生成函数。

    新功能，版本为 2.0.0b4。

+   `compare` –

    特定于 声明性数据类映射，指示在为映射类生成 `__eq__()` 和 `__ne__()` 方法时，是否应将此字段包括在比较操作中。

    新功能，版本为 2.0.0b4。

+   `kw_only` –

    特定于 声明性数据类映射，指示在为数据类过程生成的 `__init__()` 方法中，是否应将此字段标记为仅关键字。

    新功能，版本为 2.0.0b4。

+   `info` – 可选项，如果存在，将分配给 `AssociationProxy.info`。

下列附加参数涉及在 `AssociationProxy` 对象中注入自定义行为，仅供高级使用：

参数：

+   `getset_factory` –

    可选项。代理属性访问由基于该代理的 attr 参数的 get 和 set 值的例程自动处理。

    如果您想自定义此行为，可以提供一个 getset_factory 可调用对象，该对象会生成一个 getter 和 setter 函数的元组。工厂使用两个参数调用，即底层集合的抽象类型和此代理实例。

+   `proxy_factory` – 可选项。要模拟的集合类型由嗅探目标集合确定。如果无法通过鸭子类型确定您的集合类型，或者您想使用不同的集合实现，可以提供一个工厂函数来生成这些集合。仅适用于非标量关系。

+   `proxy_bulk_set` – 可选项，与 proxy_factory 一起使用。

```py
class sqlalchemy.ext.associationproxy.AssociationProxy
```

描述符，用于呈现对象属性的读/写视图。

**成员**

__init__(), cascade_scalar_deletes, create_on_none_assignment, creator, extension_type, for_class(), getset_factory, info, is_aliased_class, is_attribute, is_bundle, is_clause_element, is_instance, is_mapper, is_property, is_selectable, key, proxy_bulk_set, proxy_factory, target_collection, value_attr

**类签名**

类`sqlalchemy.ext.associationproxy.AssociationProxy` (`sqlalchemy.orm.base.InspectionAttrInfo`, `sqlalchemy.orm.base.ORMDescriptor`, `sqlalchemy.orm._DCAttributeOptions`, `sqlalchemy.ext.associationproxy._AssociationProxyProtocol`)

```py
method __init__(target_collection: str, attr: str, *, creator: _CreatorProtocol | None = None, getset_factory: _GetSetFactoryProtocol | None = None, proxy_factory: _ProxyFactoryProtocol | None = None, proxy_bulk_set: _ProxyBulkSetProtocol | None = None, info: _InfoType | None = None, cascade_scalar_deletes: bool = False, create_on_none_assignment: bool = False, attribute_options: _AttributeOptions | None = None)
```

构造一个新的`AssociationProxy`。

`AssociationProxy`对象通常使用`association_proxy()`构造函数构造。有关所有参数的描述，请参阅`association_proxy()`的描述。

```py
attribute cascade_scalar_deletes: bool
```

```py
attribute create_on_none_assignment: bool
```

```py
attribute creator: _CreatorProtocol | None
```

```py
attribute extension_type: InspectionAttrExtensionType = 'ASSOCIATION_PROXY'
```

扩展类型，如果有的话。默认为`NotExtension.NOT_EXTENSION`

另请参阅

`HybridExtensionType`

`AssociationProxyExtensionType`

```py
method for_class(class_: Type[Any], obj: object | None = None) → AssociationProxyInstance[_T]
```

返回特定映射类的内部状态。

例如，给定一个类 `User`：

```py
class User(Base):
    # ...

    keywords = association_proxy('kws', 'keyword')
```

如果我们从 `Mapper.all_orm_descriptors` 访问此 `AssociationProxy`，并且我们想要查看由 `User` 映射的此代理的目标类：

```py
inspect(User).all_orm_descriptors["keywords"].for_class(User).target_class
```

这将返回一个特定于 `User` 类的 `AssociationProxyInstance` 实例。 `AssociationProxy` 对象保持对其父类的不可知性。

参数：

+   `class_` – 我们要返回状态的类。

+   `obj` – 可选，如果属性引用多态目标，则需要该类的实例，例如，在我们必须查看实际目标对象的类型以获取完整路径的情况下。

版本 1.3 中的新功能：- `AssociationProxy` 不再存储任何特定于特定父类的状态；状态现在存储在每个类的 `AssociationProxyInstance` 对象中。

```py
attribute getset_factory: _GetSetFactoryProtocol | None
```

```py
attribute info
```

*从* `InspectionAttrInfo` *的* `InspectionAttrInfo.info` *属性继承*

与该 `InspectionAttr` 关联的信息字典，允许将用户定义的数据与此对象关联。

字典在首次访问时生成。或者，它可以作为 `column_property()`、`relationship()` 或 `composite()` 函数的构造函数参数指定。

另请参阅

`QueryableAttribute.info`

`SchemaItem.info`

```py
attribute is_aliased_class = False
```

*从* `InspectionAttr` *的* `InspectionAttr.is_aliased_class` *属性继承*

如果此对象是 `AliasedClass` 的实例，则为 True。

```py
attribute is_attribute = True
```

如果此对象是 Python 的 descriptor 则为 True。

这可以指代多种类型。通常是一个 `QueryableAttribute`，它代表一个 `MapperProperty` 处理属性事件。但也可以是一个扩展类型，如 `AssociationProxy` 或 `hybrid_property`。`InspectionAttr.extension_type` 将指代一个特定子类型的常量。

另请参阅

`Mapper.all_orm_descriptors`

```py
attribute is_bundle = False
```

*继承自* `InspectionAttr` *的* `InspectionAttr.is_bundle` *属性。

如果此对象是 `Bundle` 的实例，则为 True。

```py
attribute is_clause_element = False
```

*继承自* `InspectionAttr` *的* `InspectionAttr.is_clause_element` *属性。

如果此对象是 `ClauseElement` 的实例，则为 True。

```py
attribute is_instance = False
```

*继承自* `InspectionAttr` *的* `InspectionAttr.is_instance` *属性。

如果此对象是 `InstanceState` 的实例，则为 True。

```py
attribute is_mapper = False
```

*继承自* `InspectionAttr` *的* `InspectionAttr.is_mapper` *属性。

如果此对象是 `Mapper` 的实例，则为 True。

```py
attribute is_property = False
```

*继承自* `InspectionAttr` *的* `InspectionAttr.is_property` *属性。

如果此对象是 `MapperProperty` 的实例，则为 True。

```py
attribute is_selectable = False
```

*继承自* `InspectionAttr` *的* `InspectionAttr.is_selectable` *属性。

如果此对象是`Selectable`的实例，则返回 True。

```py
attribute key: str
```

```py
attribute proxy_bulk_set: _ProxyBulkSetProtocol | None
```

```py
attribute proxy_factory: _ProxyFactoryProtocol | None
```

```py
attribute target_collection: str
```

```py
attribute value_attr: str
```

```py
class sqlalchemy.ext.associationproxy.AssociationProxyInstance
```

一个为类和对象提供特定结果的对象。

当`AssociationProxy`在特定类或类实例的术语中被调用时，即当它被用作常规的 Python 描述符时，会使用这个对象。

当将`AssociationProxy`作为普通的 Python 描述符引用时，`AssociationProxyInstance`是实际提供信息的对象。在正常情况下，它的存在是透明的：

```py
>>> User.keywords.scalar
False
```

在特殊情况下，直接访问`AssociationProxy`对象时，为了获得对`AssociationProxyInstance`的明确处理，使用`AssociationProxy.for_class()`方法：

```py
proxy_state = inspect(User).all_orm_descriptors["keywords"].for_class(User)

# view if proxy object is scalar or not
>>> proxy_state.scalar
False
```

版本 1.3 中的新功能。

**成员**

__eq__(), __le__(), __lt__(), __ne__(), all_(), any(), any_(), asc(), attr, between(), bitwise_and(), bitwise_lshift(), bitwise_not(), bitwise_or(), bitwise_rshift(), bitwise_xor(), bool_op(), collate(), collection_class, concat(), contains(), delete(), desc(), distinct(), endswith(), for_proxy(), get(), has(), icontains(), iendswith(), ilike(), in_(), info, is_(), is_distinct_from(), is_not(), is_not_distinct_from(), isnot(), isnot_distinct_from(), istartswith(), like(), local_attr, match(), not_ilike(), not_in(), not_like(), notilike(), notin_(), notlike(), nulls_first(), nulls_last(), nullsfirst(), nullslast(), op(), operate(), parent, regexp_match(), regexp_replace(), remote_attr, reverse_operate(), scalar, set(), startswith(), target_class, timetuple

**类签名**

类`sqlalchemy.ext.associationproxy.AssociationProxyInstance` (`sqlalchemy.orm.base.SQLORMOperations`)

```py
method __eq__(other: Any) → ColumnOperators
```

*继承自* `sqlalchemy.sql.expression.ColumnOperators.__eq__` *方法的* `ColumnOperators` *方法*

实现`==`运算符。

在列上下文中，生成子句`a = b`。如果目标是`None`，则生成`a IS NULL`。

```py
method __le__(other: Any) → ColumnOperators
```

*继承自* `sqlalchemy.sql.expression.ColumnOperators.__le__` *方法的* `ColumnOperators` *方法*

实现`<=`运算符。

在列上下文中，生成子句`a <= b`。

```py
method __lt__(other: Any) → ColumnOperators
```

*继承自* `sqlalchemy.sql.expression.ColumnOperators.__lt__` *方法的* `ColumnOperators` *方法*

实现`<`运算符。

在列上下文中，生成子句`a < b`。

```py
method __ne__(other: Any) → ColumnOperators
```

*继承自* `sqlalchemy.sql.expression.ColumnOperators.__ne__` *方法的* `ColumnOperators` *方法*

实现`!=`运算符。

在列上下文中，生成子句`a != b`。如果目标是`None`，则生成`a IS NOT NULL`。

```py
method all_() → ColumnOperators
```

*继承自* `ColumnOperators.all_()` *方法的* `ColumnOperators` *方法*

对父对象生成一个`all_()`子句。

请参阅`all_()`的文档以获取示例。

注意

一定不要混淆新的`ColumnOperators.all_()`方法与此方法的**旧版**，特定于`ARRAY`的`Comparator.all()`方法，它使用不同的调用风格。

```py
method any(criterion: _ColumnExpressionArgument[bool] | None = None, **kwargs: Any) → ColumnElement[bool]
```

使用 EXISTS 生成一个代理的“any”表达式。

此表达式将使用底层代理属性的`Comparator.any()`和/或`Comparator.has()`运算符进行组合乘积。

```py
method any_() → ColumnOperators
```

*继承自* `ColumnOperators.any_()` *方法的* `ColumnOperators`

生成一个针对父对象的 `any_()` 子句。

有关示例，请参阅 `any_()` 的文档。

注

请务必不要混淆较新的 `ColumnOperators.any_()` 方法与这个方法的**旧版**，即 `ARRAY` 的特定于 `Comparator.any()` 方法，后者使用了不同的调用风格。

```py
method asc() → ColumnOperators
```

*继承自* `ColumnOperators.asc()` *方法的* `ColumnOperators`

生成一个针对父对象的 `asc()` 子句。

```py
attribute attr
```

返回一个元组 `(local_attr, remote_attr)`。

此属性最初旨在促进使用 `Query.join()` 方法一次加入两个关系，但这使用了已弃用的调用风格。

要使用 `select.join()` 或 `Query.join()` 来处理关联代理，当前的方法是分别使用 `AssociationProxyInstance.local_attr` 和 `AssociationProxyInstance.remote_attr` 属性：

```py
stmt = (
    select(Parent).
    join(Parent.proxied.local_attr).
    join(Parent.proxied.remote_attr)
)
```

未来的版本可能会提供更简洁的关联代理属性加入模式。

另请参阅

`AssociationProxyInstance.local_attr`

`AssociationProxyInstance.remote_attr`

```py
method between(cleft: Any, cright: Any, symmetric: bool = False) → ColumnOperators
```

*继承自* `ColumnOperators.between()` *方法，来自于* `ColumnOperators`

对父对象生成 `between()` 子句，给定下限和上限范围。

```py
method bitwise_and(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_and()` *方法，来自于* `ColumnOperators`

执行位与操作，通常使用 `&` 运算符。

版本 2.0.2 中的新功能。

另请参阅

位运算符

```py
method bitwise_lshift(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_lshift()` *方法，来自于* `ColumnOperators`

执行位左移操作，通常使用 `<<` 运算符。

版本 2.0.2 中的新功能。

另请参阅

位运算符

```py
method bitwise_not() → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_not()` *方法，来自于* `ColumnOperators`

执行位非操作，通常使用 `~` 运算符。

版本 2.0.2 中的新功能。

另请参阅

位运算符

```py
method bitwise_or(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_or()` *方法，来自于* `ColumnOperators`

执行位或操作，通常使用 `|` 运算符。

版本 2.0.2 中的新功能。

另请参阅

位运算符

```py
method bitwise_rshift(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_rshift()` *方法，来自于* `ColumnOperators`

执行位右移操作，通常使用 `>>` 运算符。

版本 2.0.2 中的新功能。

另请参阅

位运算符

```py
method bitwise_xor(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_xor()` *方法的* `ColumnOperators`

执行一个位异或操作，通常通过 `^` 运算符，或 PostgreSQL 上的 `#` 运算符。

在版本 2.0.2 中新增。

另请参阅

位运算符

```py
method bool_op(opstring: str, precedence: int = 0, python_impl: Callable[[...], Any] | None = None) → Callable[[Any], Operators]
```

*继承自* `Operators.bool_op()` *方法的* `Operators`

返回一个自定义布尔运算符。

这个方法是调用 `Operators.op()` 并传递 `Operators.op.is_comparison` 标志为 True 的简写。 使用 `Operators.bool_op()` 的一个关键优势是，在使用列构造时，返回表达式的“布尔”性质将出现在 [**PEP 484**](https://peps.python.org/pep-0484/) 目的上。

另请参阅

`Operators.op()`

```py
method collate(collation: str) → ColumnOperators
```

*继承自* `ColumnOperators.collate()` *方法的* `ColumnOperators`

对父对象产生一个 `collate()` 子句，给定排序规则字符串。

另请参阅

`collate()`

```py
attribute collection_class: Type[Any] | None
```

```py
method concat(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.concat()` *方法的* `ColumnOperators`

实现‘concat’运算符。

在列上下文中，生成子句 `a || b`，或在 MySQL 上使用 `concat()` 运算符。

```py
method contains(other: Any, **kw: Any) → ColumnOperators
```

*继承自* `ColumnOperators.contains()` *方法，属于* `ColumnOperators`

实现‘包含’运算符。

生成一个 LIKE 表达式，用于测试字符串值的中间匹配：

```py
column LIKE '%' || <other> || '%'
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.contains("foobar"))
```

由于运算符使用 `LIKE`，因此存在于 <other> 表达式中的通配符字符 `"%"` 和 `"_"` 也将像通配符一样起作用。对于字面字符串值，可以将 `ColumnOperators.contains.autoescape` 标志设置为 `True`，以对字符串值中这些字符的出现应用转义，以使它们作为自身而不是通配符字符匹配。另外，`ColumnOperators.contains.escape` 参数将确立一个给定字符作为转义字符，这在目标表达式不是字面字符串时很有用。

参数：

+   `other` – 要比较的表达式。通常这是一个纯字符串值，但也可以是任意的 SQL 表达式。LIKE 通配符字符 `%` 和 `_` 默认情况下不会转义，除非 `ColumnOperators.contains.autoescape` 标志设置为 True。

+   `autoescape` –

    布尔值；当为 True 时，在 LIKE 表达式中建立一个转义字符，然后将其应用于比较值内的所有 `"%"`、`"_"` 和转义字符本身的出现，假定比较值是一个字面字符串而不是 SQL 表达式。

    例如：

    ```py
    somecolumn.contains("foo%bar", autoescape=True)
    ```

    渲染结果为：

    ```py
    somecolumn LIKE '%' || :param || '%' ESCAPE '/'
    ```

    使用值 `:param` 作为 `"foo/%bar"`。

+   `escape` –

    当给定一个字符时，该字符将使用 `ESCAPE` 关键字渲染，以将该字符作为转义字符。然后，该字符可以放在 `%` 和 `_` 的前面，以允许它们充当它们自己而不是通配符字符。

    例如：

    ```py
    somecolumn.contains("foo/%bar", escape="^")
    ```

    渲染结果为：

    ```py
    somecolumn LIKE '%' || :param || '%' ESCAPE '^'
    ```

    参数也可以与 `ColumnOperators.contains.autoescape` 结合使用：

    ```py
    somecolumn.contains("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上面的情况下，给定的字面参数将在传递到数据库之前转换为 `"foo^%bar^^bat"`。

亦参见

`ColumnOperators.startswith()`

`ColumnOperators.endswith()`

`ColumnOperators.like()`

```py
method delete(obj: Any) → None
```

```py
method desc() → ColumnOperators
```

*继承自* `ColumnOperators` *方法的* `ColumnOperators.desc()`

对父对象生成一个 `desc()` 子句。

```py
method distinct() → ColumnOperators
```

*继承自* `ColumnOperators` *方法的* `ColumnOperators.distinct()`

对父对象生成一个 `distinct()` 子句。

```py
method endswith(other: Any, escape: str | None = None, autoescape: bool = False) → ColumnOperators
```

*继承自* `ColumnOperators` *方法的* `ColumnOperators.endswith()`

实现 'endswith' 运算符。

产生一个 LIKE 表达式，用于测试字符串值的结尾匹配：

```py
column LIKE '%' || <other>
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.endswith("foobar"))
```

由于该运算符使用了 `LIKE`，所以在 <other> 表达式中存在的通配符字符 `"%"` 和 `"_"` 也会像通配符一样起作用。对于字面字符串值，可以将 `ColumnOperators.endswith.autoescape` 标志设置为 `True`，以将这些字符在字符串值内的出现进行转义，使它们作为自身而不是通配符字符进行匹配。或者，`ColumnOperators.endswith.escape` 参数将建立一个给定的字符作为转义字符，当目标表达式不是字面字符串时，这可能会有所帮助。

参数：

+   `other` – 要进行比较的表达式。通常是一个普通字符串值，但也可以是任意的 SQL 表达式。默认情况下，LIKE 通配符字符`%`和`_`不会被转义，除非设置了`ColumnOperators.endswith.autoescape`标志为 True。

+   `autoescape` –

    布尔值；当为 True 时，在 LIKE 表达式中建立一个转义字符，然后将其应用于比较值中的所有`"%"`、`"_"`和转义字符本身的出现，假定比较值是一个文字字符串而不是 SQL 表达式。

    一个表达式，例如：

    ```py
    somecolumn.endswith("foo%bar", autoescape=True)
    ```

    将呈现为：

    ```py
    somecolumn LIKE '%' || :param ESCAPE '/'
    ```

    使用值为`:param`的`"foo/%bar"`。

+   `escape` –

    一个字符，当给定时，将使用`ESCAPE`关键字来将该字符作为转义字符。然后可以将此字符放在`%`和`_`之前，以允许它们作为自身而不是通配符字符。

    一个表达式，例如：

    ```py
    somecolumn.endswith("foo/%bar", escape="^")
    ```

    将呈现为：

    ```py
    somecolumn LIKE '%' || :param ESCAPE '^'
    ```

    参数也可以与`ColumnOperators.endswith.autoescape`结合使用：

    ```py
    somecolumn.endswith("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上述情况下，给定的文字参数将在传递到数据库之前转换为`"foo^%bar^^bat"`。

另请参阅

`ColumnOperators.startswith()`

`ColumnOperators.contains()`

`ColumnOperators.like()`

```py
classmethod for_proxy(parent: AssociationProxy[_T], owning_class: Type[Any], parent_instance: Any) → AssociationProxyInstance[_T]
```

```py
method get(obj: Any) → _T | None | AssociationProxyInstance[_T]
```

```py
method has(criterion: _ColumnExpressionArgument[bool] | None = None, **kwargs: Any) → ColumnElement[bool]
```

生成一个使用 EXISTS 的代理‘has’表达式。

此表达式将使用基础代理属性的`Comparator.any()`和/或`Comparator.has()`运算符进行组合。

```py
method icontains(other: Any, **kw: Any) → ColumnOperators
```

*继承自* `ColumnOperators.icontains()` *方法的* `ColumnOperators`

实现`icontains`运算符，例如`ColumnOperators.contains()`的不区分大小写版本。

生成一个 LIKE 表达式，用于对字符串值中间的不区分大小写匹配进行测试：

```py
lower(column) LIKE '%' || lower(<other>) || '%'
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.icontains("foobar"))
```

由于操作符使用了 `LIKE`，因此在 <other> 表达式中出现的通配符字符 `"%"` 和 `"_"` 也将像通配符一样起作用。对于文本字符串值，可以将 `ColumnOperators.icontains.autoescape` 标志设置为 `True`，以对字符串值中这些字符的出现进行转义，以使它们与自身匹配而不是作为通配符字符。另外，`ColumnOperators.icontains.escape` 参数将建立一个给定的字符作为转义字符，当目标表达式不是文本字符串时，这可能会有所帮助。

参数：

+   `other` – 要比较的表达式。通常是一个普通的字符串值，但也可以是任意的 SQL 表达式。除非设置了 `ColumnOperators.icontains.autoescape` 标志为 True，否则 LIKE 通配符 `%` 和 `_` 默认不会被转义。

+   `autoescape` –

    布尔值；当为 True 时，在 LIKE 表达式中建立一个转义字符，然后将其应用于比较值中所有的 `"%"`、`"_"` 和转义字符本身的出现，假定比较值是一个文本字符串而不是一个 SQL 表达式。

    一个表达式如下：

    ```py
    somecolumn.icontains("foo%bar", autoescape=True)
    ```

    将呈现为：

    ```py
    lower(somecolumn) LIKE '%' || lower(:param) || '%' ESCAPE '/'
    ```

    其中 `:param` 的值为 `"foo/%bar"`。

+   `escape` –

    一个字符，当给定时，将呈现为 `ESCAPE` 关键字以将该字符设为转义字符。然后可以将此字符放置在 `%` 和 `_` 的前面，以允许它们作为自身而不是通配符字符。

    一个表达式如下：

    ```py
    somecolumn.icontains("foo/%bar", escape="^")
    ```

    将呈现为：

    ```py
    lower(somecolumn) LIKE '%' || lower(:param) || '%' ESCAPE '^'
    ```

    该参数也可以与 `ColumnOperators.contains.autoescape` 结合使用：

    ```py
    somecolumn.icontains("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上述情况下，给定的文本参数将在传递到数据库之前转换为 `"foo^%bar^^bat"`。

另请参见

`ColumnOperators.contains()`

```py
method iendswith(other: Any, escape: str | None = None, autoescape: bool = False) → ColumnOperators
```

*继承自* `ColumnOperators.iendswith()` *方法的* `ColumnOperators`

实现`iendswith`操作符，例如`ColumnOperators.endswith()`的不区分大小写版本。

产生一个 LIKE 表达式，用于对字符串值的末尾进行不区分大小写匹配：

```py
lower(column) LIKE '%' || lower(<other>)
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.iendswith("foobar"))
```

由于该操作符使用`LIKE`，在<other>表达式中存在的通配符字符`"%"`和`"_"`也将像通配符一样行为。对于字面字符串值，可以将`ColumnOperators.iendswith.autoescape`标志设置为`True`，以对字符串值内部这些字符的出现进行转义，使它们作为自己而不是通配符字符匹配。或者，`ColumnOperators.iendswith.escape`参数将建立一个给定字符作为转义字符，这在目标表达式不是字面字符串时可能有用。

参数：

+   `other` - 要进行比较的表达式。通常这是一个简单的字符串值，但也可以是任意 SQL 表达式。除非将`ColumnOperators.iendswith.autoescape`标志设置为 True，否则 LIKE 通配符字符`%`和`_`不会被转义。

+   `autoescape` -

    布尔值；当为 True 时，在 LIKE 表达式中建立一个转义字符，然后将其应用于比较值中的所有`"%"`、`"_"`和转义字符自身的出现，该比较值假定为字面字符串而不是 SQL 表达式。

    诸如以下表达式：

    ```py
    somecolumn.iendswith("foo%bar", autoescape=True)
    ```

    将渲染为：

    ```py
    lower(somecolumn) LIKE '%' || lower(:param) ESCAPE '/'
    ```

    其值为`:param`，为`"foo/%bar"`。

+   `escape` -

    给定字符，当使用`ESCAPE`关键字时，将该字符设定为转义字符。这个字符可以放在`%`和`_`之前，以允许它们作为自己而不是通配符字符。

    诸如以下表达式：

    ```py
    somecolumn.iendswith("foo/%bar", escape="^")
    ```

    将渲染为：

    ```py
    lower(somecolumn) LIKE '%' || lower(:param) ESCAPE '^'
    ```

    该参数还可以与`ColumnOperators.iendswith.autoescape`结合使用：

    ```py
    somecolumn.endswith("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上述情况下，给定的字面参数将在传递到数据库之前转换为`"foo^%bar^^bat"`。

另请参见

`ColumnOperators.endswith()`

```py
method ilike(other: Any, escape: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.ilike()` *方法的* `ColumnOperators`

实现`ilike`运算符，例如，不区分大小写的 LIKE。

在列上下文中，生成一个形式为的表达式：

```py
lower(a) LIKE lower(other)
```

或者在支持 ILIKE 运算符的后端上：

```py
a ILIKE other
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.ilike("%foobar%"))
```

参数：

+   `other` – 要比较的表达式

+   `escape` –

    可选的转义字符，呈现`ESCAPE`关键字，例如：

    ```py
    somecolumn.ilike("foo/%bar", escape="/")
    ```

另请参阅

`ColumnOperators.like()`

```py
method in_(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.in_()` *方法的* `ColumnOperators`

实现`in`运算符。

在列上下文中，生成子句`column IN <other>`。

给定的参数`other`可能是：

+   一个字面值列表，例如：

    ```py
    stmt.where(column.in_([1, 2, 3]))
    ```

    在这种调用形式中，项目列表被转换为与给定列表长度相同的一组绑定参数：

    ```py
    WHERE COL IN (?, ?, ?)
    ```

+   如果比较对象是包含多个表达式的`tuple_()`元组列表可以提供：

    ```py
    from sqlalchemy import tuple_
    stmt.where(tuple_(col1, col2).in_([(1, 10), (2, 20), (3, 30)]))
    ```

+   一个空列表，例如：

    ```py
    stmt.where(column.in_([]))
    ```

    在这种调用形式中，表达式呈现一个“空集”表达式。这些表达式针对各个后端进行了定制，并且通常试图获得一个空的 SELECT 语句作为子查询。例如，在 SQLite 上，表达式是：

    ```py
    WHERE col IN (SELECT 1 FROM (SELECT 1) WHERE 1!=1)
    ```

    版本 1.4 中的更改：现在在所有情况下，空的 IN 表达式都使用执行时生成的 SELECT 子查询。

+   可以使用一个绑定参数，例如`bindparam()`，如果它包含`bindparam.expanding`标志：

    ```py
    stmt.where(column.in_(bindparam('value', expanding=True)))
    ```

    在这种调用形式中，表达式呈现一个特殊的非 SQL 占位符表达式，看起来像：

    ```py
    WHERE COL IN ([EXPANDING_value])
    ```

    此占位符表达式在语句执行时被拦截，转换为前面所示的可变数量的绑定参数形式。如果语句执行为：

    ```py
    connection.execute(stmt, {"value": [1, 2, 3]})
    ```

    数据库将为每个值传递一个绑定参数：

    ```py
    WHERE COL IN (?, ?, ?)
    ```

    版本 1.2 中的新功能：添加了“expanding”绑定参数

    如果传递了一个空列表，则会呈现一个特殊的“空列表”表达式，这是特定于正在使用的数据库的。在 SQLite 上，这将是：

    ```py
    WHERE COL IN (SELECT 1 FROM (SELECT 1) WHERE 1!=1)
    ```

    版本 1.3 中的新功能：现在支持空列表的“expanding”绑定参数

+   一个`select()`构造，通常是一个相关的标量选择：

    ```py
    stmt.where(
        column.in_(
            select(othertable.c.y).
            where(table.c.x == othertable.c.x)
        )
    )
    ```

    在这种调用形式中，`ColumnOperators.in_()`呈现如下：

    ```py
    WHERE COL IN (SELECT othertable.y
    FROM othertable WHERE othertable.x = table.x)
    ```

参数：

**other** – 一个文字列表，一个`select()`构造，或者一个包含设置为 True 的`bindparam.expanding`标志的`bindparam()`构造。

```py
attribute info
```

```py
method is_(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.is_()` *方法的* `ColumnOperators`

实现`IS`运算符。

通常，当与`None`的值进行比较时，`IS`会自动生成，解析为`NULL`。然而，在某些平台上，如果与布尔值进行比较，则可能希望显式使用`IS`。

另请参阅

`ColumnOperators.is_not()`

```py
method is_distinct_from(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.is_distinct_from()` *方法的* `ColumnOperators`

实现`IS DISTINCT FROM`运算符。

在大多数平台上呈现“a IS DISTINCT FROM b”；在某些平台上如 SQLite 可能呈现“a IS NOT b”。

```py
method is_not(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.is_not()` *方法的* `ColumnOperators`

实现`IS NOT`运算符。

通常，当与`None`的值进行比较时，`IS NOT`会自动生成，解析为`NULL`。然而，在某些平台上，如果与布尔值进行比较，则可能希望显式使用`IS NOT`。

从版本 1.4 开始更改：`is_not()`运算符从先前版本的`isnot()`重命名。以确保向后兼容性，先前的名称仍然可用。

另请参阅

`ColumnOperators.is_()`

```py
method is_not_distinct_from(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.is_not_distinct_from()` *的* `ColumnOperators` *方法*

实现`IS NOT DISTINCT FROM`操作符。

大多数平台上会渲染“a IS NOT DISTINCT FROM b”；但在一些平台上如 SQLite 可能会渲染“a IS b”。

在版本 1.4 中更改：`is_not_distinct_from()`操作符从之前的版本中的`isnot_distinct_from()`重命名为`is_not_distinct_from()`。以前的名称仍然可用于向后兼容。

```py
method isnot(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.isnot()` *的* `ColumnOperators` *方法*

实现`IS NOT`操作符。

通常，当与`None`的值进行比较时，会自动生成`IS NOT`，它将解析为`NULL`。然而，在某些平台上，如果与布尔值进行比较，则可能希望明确使用`IS NOT`。

在版本 1.4 中更改：`is_not()`操作符从之前的版本中的`isnot()`重命名为`is_not()`。以前的名称仍然可用于向后兼容。

另请参阅

`ColumnOperators.is_()`

```py
method isnot_distinct_from(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.isnot_distinct_from()` *的* `ColumnOperators` *方法*

实现`IS NOT DISTINCT FROM`操作符。

大多数平台上会渲染“a IS NOT DISTINCT FROM b”；但在一些平台上如 SQLite 可能会渲染“a IS b”。

在版本 1.4 中更改：`is_not_distinct_from()`操作符从之前的版本中的`isnot_distinct_from()`重命名为`is_not_distinct_from()`。以前的名称仍然可用于向后兼容。

```py
method istartswith(other: Any, escape: str | None = None, autoescape: bool = False) → ColumnOperators
```

*继承自* `ColumnOperators.istartswith()` *的* `ColumnOperators` *方法*

实现`istartswith`操作符，例如`ColumnOperators.startswith()`的大小写不敏感版本。

生成一个 LIKE 表达式，用于对字符串值的开头进行大小写不敏感匹配测试：

```py
lower(column) LIKE lower(<other>) || '%'
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.istartswith("foobar"))
```

由于操作符使用 `LIKE`，所以存在于 <other> 表达式中的通配符字符 `"%"` 和 `"_"` 也将像通配符一样起作用。对于文字字符串值，可以将 `ColumnOperators.istartswith.autoescape` 标志设置为 `True`，以对字符串值内这些字符的出现应用转义，使它们与自身匹配而不是通配符字符。或者，`ColumnOperators.istartswith.escape` 参数将建立一个给定的字符作为转义字符，当目标表达式不是文字字符串时可能有用。

参数：

+   `other` – 要比较的表达式。通常是一个普通的字符串值，但也可以是任意的 SQL 表达式。LIKE 通配符 `%` 和 `_` 默认情况下不会转义，除非设置了 `ColumnOperators.istartswith.autoescape` 标志为 True。

+   `autoescape` –

    布尔值；当为 True 时，在 LIKE 表达式内建立一个转义字符，然后将其应用于比较值内所有的 `"%"`、`"_"` 和转义字符本身的出现，假定比较值为文字字符串而不是 SQL 表达式。

    一个表达式，例如：

    ```py
    somecolumn.istartswith("foo%bar", autoescape=True)
    ```

    将渲染为：

    ```py
    lower(somecolumn) LIKE lower(:param) || '%' ESCAPE '/'
    ```

    其中 `:param` 的值为 `"foo/%bar"`。

+   `escape` –

    一个字符，当给定时，将以 `ESCAPE` 关键字呈现，以将该字符建立为转义字符。然后，可以将此字符放在 `%` 和 `_` 的前面，以允许它们作为自身而不是通配符字符。

    一个表达式，例如：

    ```py
    somecolumn.istartswith("foo/%bar", escape="^")
    ```

    渲染为：

    ```py
    lower(somecolumn) LIKE lower(:param) || '%' ESCAPE '^'
    ```

    此参数也可以与 `ColumnOperators.istartswith.autoescape` 结合使用：

    ```py
    somecolumn.istartswith("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上述情况下，给定的文字参数将在传递到数据库之前被转换为 `"foo^%bar^^bat"`。

另请参阅

`ColumnOperators.startswith()`

```py
method like(other: Any, escape: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.like()` *方法的* `ColumnOperators`

实现 `like` 操作符。

在列上下文中，产生表达式：

```py
a LIKE other
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.like("%foobar%"))
```

参数：

+   `other` – 要比较的表达式

+   `escape` –

    可选的转义字符，呈现 `ESCAPE` 关键字，例如：

    ```py
    somecolumn.like("foo/%bar", escape="/")
    ```

另请参阅

`ColumnOperators.ilike()`

```py
attribute local_attr
```

此 `AssociationProxyInstance` 引用的 ‘local’ 类属性。

另请参阅

`AssociationProxyInstance.attr`

`AssociationProxyInstance.remote_attr`

```py
method match(other: Any, **kwargs: Any) → ColumnOperators
```

*继承自* `ColumnOperators.match()` *方法的* `ColumnOperators`

实现了特定于数据库的 ‘match’ 操作符。

`ColumnOperators.match()` 尝试解析为后端提供的 MATCH 类似函数或操作符。例如：

+   PostgreSQL - 呈现 `x @@ plainto_tsquery(y)`

    > 从版本 2.0 开始更改：现在为 PostgreSQL 使用 `plainto_tsquery()` 而不是 `to_tsquery()`；为了与其他形式兼容，请参阅 全文搜索。 

+   MySQL - 呈现 `MATCH (x) AGAINST (y IN BOOLEAN MODE)`

    另请参阅

    `match` - 具有附加功能的特定于 MySQL 的构造。

+   Oracle - 呈现 `CONTAINS(x, y)`

+   其他后端可能提供特殊实现。

+   没有任何特殊实现的后端将操作符输出为“MATCH”。例如，这与 SQLite 兼容。

```py
method not_ilike(other: Any, escape: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.not_ilike()` *方法的* `ColumnOperators`

实现 `NOT ILIKE` 操作符。

这相当于在 `ColumnOperators.ilike()` 中使用否定，即 `~x.ilike(y)`。

从版本 1.4 开始更改：`not_ilike()` 操作符从先前版本的 `notilike()` 重命名。之前的名称仍然可用于向后兼容。

另请参阅

`ColumnOperators.ilike()`

```py
method not_in(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.not_in()` *方法的* `ColumnOperators`

实现`NOT IN`运算符。

这相当于在`ColumnOperators.in_()`中使用否定，即`~x.in_(y)`。

在`other`是空序列的情况下，编译器会生成一个“空 not in”表达式。默认情况下，这会产生一个“1 = 1”的表达式，以在所有情况下产生 true。可以使用`create_engine.empty_in_strategy`来更改此行为。

从版本 1.4 开始更改：`not_in()`运算符从先前版本的`notin_()`重命名。以确保向后兼容性，先前的名称仍可用。

从版本 1.2 开始更改：`ColumnOperators.in_()`和`ColumnOperators.not_in()`运算符现在默认情况下为一个空 IN 序列生成一个“静态”表达式。

另请参阅

`ColumnOperators.in_()`

```py
method not_like(other: Any, escape: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.not_like()` *方法的* `ColumnOperators`

实现`NOT LIKE`运算符。

这相当于在`ColumnOperators.like()`中使用否定，即`~x.like(y)`。

从版本 1.4 开始更改：`not_like()`运算符从先前版本的`notlike()`重命名。以确保向后兼容性，先前的名称仍可用。

另请参阅

`ColumnOperators.like()`

```py
method notilike(other: Any, escape: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.notilike()` *方法的* `ColumnOperators`

实现`NOT ILIKE`运算符。

这等同于使用 `ColumnOperators.ilike()` 进行否定，即 `~x.ilike(y)`。

从版本 1.4 开始更改：`not_ilike()` 操作符从先前版本的 `notilike()` 重命名。以确保向后兼容性，先前的名称仍然可用。

另请参阅

`ColumnOperators.ilike()`

```py
method notin_(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.notin_()` *方法的* `ColumnOperators`

实现 `NOT IN` 操作符。

这等同于使用 `ColumnOperators.in_()` 进行否定，即 `~x.in_(y)`。

如果 `other` 是一个空序列，则编译器会生成一个“空的不在”表达式。默认情况下，这等同于表达式“1 = 1”，在所有情况下都返回 true。可以使用 `create_engine.empty_in_strategy` 来更改此行为。

从版本 1.4 开始更改：`not_in()` 操作符从先前版本的 `notin_()` 重命名。以确保向后兼容性，先前的名称仍然可用。

从版本 1.2 开始更改：`ColumnOperators.in_()` 和 `ColumnOperators.not_in()` 操作符现在默认情况下为一个空的 IN 序列生成一个“静态”表达式。

另请参阅

`ColumnOperators.in_()`

```py
method notlike(other: Any, escape: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.notlike()` *方法的* `ColumnOperators`

实现 `NOT LIKE` 操作符。

这等同于使用 `ColumnOperators.like()` 进行否定，即 `~x.like(y)`。

从版本 1.4 开始更改：`not_like()` 操作符从先前版本的 `notlike()` 重命名。以确保向后兼容性，先前的名称仍然可用。

另请参阅

`ColumnOperators.like()`

```py
method nulls_first() → ColumnOperators
```

*继承自* `ColumnOperators.nulls_first()` *方法的* `ColumnOperators`

对父对象生成一个`nulls_first()`子句。

1.4 版本更改：`nulls_first()`运算符在先前版本中从`nullsfirst()`重命名。以确保向后兼容性，先前的名称仍然可用。

```py
method nulls_last() → ColumnOperators
```

*继承自* `ColumnOperators.nulls_last()` *方法的* `ColumnOperators`

对父对象生成一个`nulls_last()`子句。

1.4 版本更改：`nulls_last()`运算符在先前版本中从`nullslast()`重命��。以确保向后兼容性，先前的名称仍然可用。

```py
method nullsfirst() → ColumnOperators
```

*继承自* `ColumnOperators.nullsfirst()` *方法的* `ColumnOperators`

对父对象生成一个`nulls_first()`子句。

1.4 版本更改：`nulls_first()`运算符在先前版本中从`nullsfirst()`重命名。以确保向后兼容性，先前的名称仍然可用。

```py
method nullslast() → ColumnOperators
```

*继承自* `ColumnOperators.nullslast()` *方法的* `ColumnOperators`

对父对象生成一个`nulls_last()`子句。

1.4 版本更改：`nulls_last()`运算符在先前版本中从`nullslast()`重命名。以确保向后兼容性，先前的名称仍然可用。

```py
method op(opstring: str, precedence: int = 0, is_comparison: bool = False, return_type: Type[TypeEngine[Any]] | TypeEngine[Any] | None = None, python_impl: Callable[..., Any] | None = None) → Callable[[Any], Operators]
```

*继承自* `Operators.op()` *方法的* `Operators`

生成一个通用的运算符函数。

例如：

```py
somecolumn.op("*")(5)
```

产生：

```py
somecolumn * 5
```

这个函数也可以用来使位运算符明确。例如：

```py
somecolumn.op('&')(0xff)
```

是`somecolumn`中值的按位与。

参数:

+   `opstring` – 一个字符串，将作为中缀运算符输出在此元素和传递给生成函数的表达式之间。

+   `precedence` –

    预期数据库在 SQL 表达式中应用于运算符的优先级。这个整数值作为 SQL 编译器的提示，用于确定何时应该在特定操作周围渲染显式括号。较低的数字将导致在应用于具有更高优先级的另一个运算符时对表达式进行括号化。默认值为`0`，低于所有运算符，除了逗号（`,`）和`AS`运算符。值为 100 将高于或等于所有运算符，-100 将低于或等于所有运算符。

    另请参见

    我正在使用`op()`生成自定义运算符，但我的括号没有正确显示 - SQLAlchemy SQL 编译器如何渲染括号的详细描述

+   `is_comparison` –

    传统; 如果为 True，则将运算符视为“比较”运算符，即评估为布尔真/假值的运算符，如`==`，`>`等。提供此标志是为了 ORM 关系可以在自定义连接条件中使用运算符时建立该运算符是比较运算符。

    使用`is_comparison`参数已被使用`Operators.bool_op()`方法取代；这个更简洁的运算符会自动设置此参数，但也提供了正确的[**PEP 484**](https://peps.python.org/pep-0484/)类型支持，因为返回的对象将表示“布尔”数据类型，即`BinaryExpression[bool]`。

+   `return_type` – 一个`TypeEngine`类或对象，将强制此运算符生成的表达式的返回类型为该类型。默认情况下，指定`Operators.op.is_comparison`的运算符将解析为`Boolean`，而未指定的运算符将与左操作数的类型相同。

+   `python_impl` –

    可选的 Python 函数，可以像数据库服务器上运行此操作符时的工作方式一样评估两个 Python 值。适用于在 Python 中的 SQL 表达式评估函数，例如用于 ORM 混合属性的，以及在多行更新或删除后用于匹配会话中的对象的 ORM“评估器”。

    例如：

    ```py
    >>> expr = column('x').op('+', python_impl=lambda a, b: a + b)('y')
    ```

    上述表达式的操作符也将适用于非 SQL 左侧和右侧对象：

    ```py
    >>> expr.operator(5, 10)
    15
    ```

    从 2.0 版开始。

另请参见

`Operators.bool_op()`

重新定义和创建新的操作符

在联接条件中使用自定义操作符

```py
method operate(op: OperatorType, *other: Any, **kwargs: Any) → Operators
```

*继承自* `Operators.operate()` *方法的* `Operators`

对一个参数进行操作。

这是操作的最低级别，默认会引发 `NotImplementedError`。

在子类上覆盖此操作可以允许将常见行为应用于所有操作。例如，覆盖 `ColumnOperators` 以将 `func.lower()` 应用于左右两侧：

```py
class MyComparator(ColumnOperators):
    def operate(self, op, other, **kwargs):
        return op(func.lower(self), func.lower(other), **kwargs)
```

参数：

+   `op` – 操作符可调用。

+   `*other` – 操作的‘其他’一侧。 对于大多数操作，将是单个标量。

+   `**kwargs` – 修饰符。这些可以通过特殊操作符传递，如 `ColumnOperators.contains()`。

```py
attribute parent: _AssociationProxyProtocol[_T]
```

```py
method regexp_match(pattern: Any, flags: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.regexp_match()` *方法的* `ColumnOperators`

实现了一个特定于数据库的‘正则匹配’操作符。

例如：

```py
stmt = select(table.c.some_column).where(
    table.c.some_column.regexp_match('^(b|c)')
)
```

`ColumnOperators.regexp_match()` 尝试解析为由后端提供的类似 REGEXP 的函数或操作符，但是具体的正则表达式语法和可用标志**与后端无关**。

示例包括：

+   PostgreSQL - 在否定时渲染 `x ~ y` 或 `x !~ y`。

+   Oracle - 渲染 `REGEXP_LIKE(x, y)`

+   SQLite - 使用 SQLite 的 `REGEXP` 占位符操作符并调用 Python 的 `re.match()` 内建函数。

+   其他后端可能提供特殊实现。

+   没有任何特殊实现的后端将将操作符发出为“REGEXP”或“NOT REGEXP”。 例如，这与 SQLite 和 MySQL 兼容。

目前 Oracle、PostgreSQL、MySQL 和 MariaDB 均已实现了正则表达式支持。SQLite 部分支持。第三方方言中的支持可能有所不同。

参数：

+   `pattern` – 正则表达式模式字符串或列子句。

+   `flags` – 要应用的任何正则表达式字符串标志，仅作为普通 Python 字符串传递。这些标志是特定于后端的。某些后端，如 PostgreSQL 和 MariaDB，可能会将标志作为模式的一部分来指定。在 PostgreSQL 中使用忽略大小写标志‘i’时，将使用忽略大小写的正则表达式匹配运算符`~*`或`!~*`。

版本 1.4 中的新功能。

在版本 1.4.48 更改：2.0.18 请注意，由于实现错误，“flags”参数先前接受了 SQL 表达式对象，例如列表达式，除了普通的 Python 字符串。此实现与缓存不兼容，并已删除；应仅传递字符串作为“flags”参数，因为这些标志将作为 SQL 表达式中的文字内联值呈现。

另请参见

`ColumnOperators.regexp_replace()`

```py
method regexp_replace(pattern: Any, replacement: Any, flags: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.regexp_replace()` *方法的* `ColumnOperators`

实现了特定于数据库的“regexp replace”运算符。

例如：

```py
stmt = select(
    table.c.some_column.regexp_replace(
        'b(..)',
        'XY',
        flags='g'
    )
)
```

`ColumnOperators.regexp_replace()` 试图解析为后端提供的类似 REGEXP_REPLACE 的函数，通常会生成函数`REGEXP_REPLACE()`。然而，特定的正则表达式语法和可用的标志**不是后端通用的**。

目前 Oracle、PostgreSQL、MySQL 8 或更高版本和 MariaDB 均已实现了正则表达式替换支持。第三方方言中的支持可能有所不同。

参数：

+   `pattern` – 正则表达式模式字符串或列子句。

+   `pattern` – 替换字符串或列子句。

+   `flags` – 要应用的任何正则表达式字符串标志，仅作为普通 Python 字符串传递。这些标志是特定于后端的。某些后端，如 PostgreSQL 和 MariaDB，可能会将标志作为模式的一部分来指定。

版本 1.4 中的新功能。

从版本 1.4.48 更改为：2.0.18 请注意，由于实现错误，之前的“flags”参数接受了 SQL 表达式对象，例如列表达式，而不仅仅是普通的 Python 字符串。该实现与缓存一起不正确地工作，并已删除；仅应传递字符串作为“flags”参数，因为这些标志被渲染为 SQL 表达式中的文字内联值。

另请参阅

`ColumnOperators.regexp_match()`

```py
attribute remote_attr
```

此`AssociationProxyInstance`引用的“remote”类属性。

另请参阅

`AssociationProxyInstance.attr`

`AssociationProxyInstance.local_attr`

```py
method reverse_operate(op: OperatorType, other: Any, **kwargs: Any) → Operators
```

*继承自* `Operators.reverse_operate()` *方法的* `Operators`

在参数上执行反向操作。

用法与`operate()`相同。

```py
attribute scalar
```

如果此`AssociationProxyInstance`代理了本地一侧的标量关系，则返回`True`。

```py
method set(obj: Any, values: _T) → None
```

```py
method startswith(other: Any, escape: str | None = None, autoescape: bool = False) → ColumnOperators
```

*继承自* `ColumnOperators.startswith()` *方法的* `ColumnOperators`

实现`startswith`运算符。

生成一个 LIKE 表达式，用于测试字符串值的开头匹配：

```py
column LIKE <other> || '%'
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.startswith("foobar"))
```

由于该操作员使用 `LIKE`，所以在 <other> 表达式内存在的通配符字符 `"%"` 和 `"_"` 也将像通配符一样运行。对于文字字符串值，可以将 `ColumnOperators.startswith.autoescape` 标志设置为 `True`，以对字符串值中这些字符的出现进行转义，使它们作为自身而不是通配符字符匹配。或者，`ColumnOperators.startswith.escape` 参数将建立一个给定的字符作为转义字符，这在目标表达式不是文字字符串时可能会有用。

参数：

+   `other` – 要比较的表达式。这通常是一个简单的字符串值，但也可以是任意的 SQL 表达式。默认情况下，LIKE 通配符字符 `%` 和 `_` 不被转义，除非 `ColumnOperators.startswith.autoescape` 标志被设置为 True。

+   `autoescape` –

    布尔值；当为 True 时，在 LIKE 表达式中建立一个转义字符，然后将其应用于比较值中的所有出现的 `"%"`、`"_"` 和转义字符本身，假定比较值是一个文字字符串而不是一个 SQL 表达式。

    一个表达式，例如：

    ```py
    somecolumn.startswith("foo%bar", autoescape=True)
    ```

    将渲染为：

    ```py
    somecolumn LIKE :param || '%' ESCAPE '/'
    ```

    使用值为 `:param` 的 `"foo/%bar"`。

+   `escape` –

    一个字符，当给定时，将使用 `ESCAPE` 关键字将该字符设定为转义字符。然后可以将此字符放置在 `%` 和 `_` 出现之前，以使它们可以作为自己而不是通配符字符。

    一个表达式，例如：

    ```py
    somecolumn.startswith("foo/%bar", escape="^")
    ```

    将渲染为：

    ```py
    somecolumn LIKE :param || '%' ESCAPE '^'
    ```

    参数也可以与 `ColumnOperators.startswith.autoescape` 结合使用：

    ```py
    somecolumn.startswith("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上述情况下，给定的文字参数在传递到数据库之前将被转换为 `"foo^%bar^^bat"`。

另请参阅

`ColumnOperators.endswith()`

`ColumnOperators.contains()`

`ColumnOperators.like()`

```py
attribute target_class: Type[Any]
```

此`AssociationProxyInstance`处理的中间类。

拦截的追加/设置/赋值事件将导致生成此类的新实例。

```py
attribute timetuple: Literal[None] = None
```

*继承自* `ColumnOperators.timetuple` *属性的* `ColumnOperators`

Hack，允许在左侧比较日期时间对象。

```py
class sqlalchemy.ext.associationproxy.ObjectAssociationProxyInstance
```

一个`AssociationProxyInstance`，其目标是一个对象。

**成员**

__le__(), __lt__(), all_(), any(), any_(), asc(), attr, between(), bitwise_and(), bitwise_lshift(), bitwise_not(), bitwise_or(), bitwise_rshift(), bitwise_xor(), bool_op(), collate(), concat(), contains(), desc(), distinct(), endswith(), has(), icontains(), iendswith(), ilike(), in_(), is_(), is_distinct_from(), is_not(), is_not_distinct_from(), isnot(), isnot_distinct_from(), istartswith(), like(), local_attr, match(), not_ilike(), not_in(), not_like(), notilike(), notin_(), notlike(), nulls_first(), nulls_last(), nullsfirst(), nullslast(), op(), operate(), regexp_match(), regexp_replace(), remote_attr, reverse_operate(), scalar, startswith(), target_class, timetuple

**类签名**

类`sqlalchemy.ext.associationproxy.ObjectAssociationProxyInstance`（`sqlalchemy.ext.associationproxy.AssociationProxyInstance`）

```py
method __le__(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators` *的* `sqlalchemy.sql.expression.ColumnOperators.__le__` *方法*

实现 `<=` 操作符。

在列上下文中，生成子句 `a <= b`。

```py
method __lt__(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators` *的* `sqlalchemy.sql.expression.ColumnOperators.__lt__` *方法*

实现 `<` 操作符。

在列上下文中，生成子句 `a < b`。

```py
method all_() → ColumnOperators
```

*继承自* `ColumnOperators` *的* `ColumnOperators.all_()` *方法*

对父对象生成一个 `all_()` 子句。

请查看`all_()`的文档以获取示例。

注意

不要混淆较新的`ColumnOperators.all_()`方法与**旧版本**该方法，即特定于`ARRAY`的`Comparator.all()`方法，后者使用不同的调用风格。

```py
method any(criterion: _ColumnExpressionArgument[bool] | None = None, **kwargs: Any) → ColumnElement[bool]
```

*继承自* `AssociationProxyInstance` *的* `AssociationProxyInstance.any()` *方法*

使用 EXISTS 生成代理的‘any’表达式。

此表达式将使用底层代理属性的`Comparator.any()`和/或`Comparator.has()`操作符进行组合。

```py
method any_() → ColumnOperators
```

*继承自* `ColumnOperators` *的* `ColumnOperators.any_()` *方法*

对父对象生成一个`any_()`子句。

有关示例，请参阅`any_()`的文档。

注意

请务必不要混淆较新的`ColumnOperators.any_()`方法与此方法的**旧版**，即`Comparator.any()`方法，后者特定于`ARRAY`，其使用了不同的调用样式。

```py
method asc() → ColumnOperators
```

*继承自* `ColumnOperators.asc()` *方法的* `ColumnOperators`

对父对象生成一个`asc()`子句。

```py
attribute attr
```

*继承自* `AssociationProxyInstance.attr` *属性的* `AssociationProxyInstance`

返回一个元组`(local_attr, remote_attr)`。

此属性最初旨在方便使用`Query.join()`方法同时跨两个关系进行连接，但这使用了已弃用的调用样式。

要使用`select.join()`或`Query.join()`与关联代理，当前方法是分别利用`AssociationProxyInstance.local_attr`和`AssociationProxyInstance.remote_attr`属性：

```py
stmt = (
    select(Parent).
    join(Parent.proxied.local_attr).
    join(Parent.proxied.remote_attr)
)
```

未来的一个版本可能会提供更简洁的关联代理属性连接模式。

另请参阅

`AssociationProxyInstance.local_attr`

`AssociationProxyInstance.remote_attr`

```py
method between(cleft: Any, cright: Any, symmetric: bool = False) → ColumnOperators
```

*继承自* `ColumnOperators.between()` *方法的* `ColumnOperators`

对父对象执行`between()` 子句，给定下限和上限范围。

```py
method bitwise_and(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_and()` *方法的* `ColumnOperators`

执行位与操作，通常使用`&`运算符。

自版本 2.0.2 新增。

另请参阅

位运算符

```py
method bitwise_lshift(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_lshift()` *方法的* `ColumnOperators`

执行位左移操作，通常使用`<<`运算符。

自版本 2.0.2 新增。

另请参阅

位运算符

```py
method bitwise_not() → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_not()` *方法的* `ColumnOperators`

执行位非操作，通常使用`~`运算符。

自版本 2.0.2 新增。

另请参阅

位运算符

```py
method bitwise_or(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_or()` *方法的* `ColumnOperators`

执行位或操作，通常使用`|`运算符。

自版本 2.0.2 新增。

另请参阅

位运算符

```py
method bitwise_rshift(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_rshift()` *方法的* `ColumnOperators`

执行位右移操作，通常使用`>>`运算符。

自版本 2.0.2 新增。

另请参阅

位运算符

```py
method bitwise_xor(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_xor()` *方法的* `ColumnOperators`

生成按位异或运算，通常通过`^`运算符，或者对于 PostgreSQL 使用`#`。

版本 2.0.2 中的新功能。

另请参阅

位运算符

```py
method bool_op(opstring: str, precedence: int = 0, python_impl: Callable[[...], Any] | None = None) → Callable[[Any], Operators]
```

*继承自* `Operators.bool_op()` *方法的* `Operators`

返回自定义布尔运算符。

此方法是调用 `Operators.op()` 并传递 `Operators.op.is_comparison` 标志为 True 的快捷方式。使用 `Operators.bool_op()` 的一个关键优势是，当使用列构造时，返回表达式的“布尔”性质将存在于[**PEP 484**](https://peps.python.org/pep-0484/)目的中。

另请参阅

`Operators.op()`

```py
method collate(collation: str) → ColumnOperators
```

*继承自* `ColumnOperators.collate()` *方法的* `ColumnOperators`

根据给定的排序字符串生成针对父对象的 `collate()` 子句。

另请参阅

`collate()`

```py
method concat(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.concat()` *方法的* `ColumnOperators`

实现‘concat’运算符。

在列上下文中，生成子句`a || b`，或在 MySQL 上使用`concat()`运算符。

```py
method contains(other: Any, **kw: Any) → ColumnElement[bool]
```

使用 EXISTS 生成代理的“包含”表达式。

此表达式将使用基础代理属性的`Comparator.any()`、`Comparator.has()`和/或`Comparator.contains()`操作符进行组合。

```py
method desc() → ColumnOperators
```

*继承自* `ColumnOperators.desc()` *方法的* `ColumnOperators`

生成针对父对象的`desc()`子句。

```py
method distinct() → ColumnOperators
```

*继承自* `ColumnOperators.distinct()` *方法的* `ColumnOperators`

生成针对父对象的`distinct()`子句。

```py
method endswith(other: Any, escape: str | None = None, autoescape: bool = False) → ColumnOperators
```

*继承自* `ColumnOperators.endswith()` *方法的* `ColumnOperators`

实现‘endswith’操作符。

生成一个 LIKE 表达式，测试字符串值的结尾是否匹配：

```py
column LIKE '%' || <other>
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.endswith("foobar"))
```

由于操作符使用`LIKE`，存在于<other>表达式中的通配符字符`"%"`和`"_"`也将像通配符一样起作用。对于字面字符串值，`ColumnOperators.endswith.autoescape`标志可以设置为`True`，以对字符串值中这些字符的出现进行转义，使它们匹配为它们自身而不是通配符字符。或者，`ColumnOperators.endswith.escape`参数将建立一个给定字符作为转义字符，当目标表达式不是字面字符串时，这可能会有用。

参数：

+   `other` – 要比较的表达式。这通常是一个简单的字符串值，但也可以是任意的 SQL 表达式。除非将`ColumnOperators.endswith.autoescape`标志设置为 True，否则 LIKE 通配符字符`%`和`_`不会被转义。

+   `autoescape` –

    布尔值；当为 True 时，在 LIKE 表达式内建立转义字符，然后将其应用于比较值中所有的`"%"`、`"_"`和转义字符本身的出现次数，假定比较值是一个字面字符串而不是一个 SQL 表达式。

    一个表达式，例如：

    ```py
    somecolumn.endswith("foo%bar", autoescape=True)
    ```

    将呈现为：

    ```py
    somecolumn LIKE '%' || :param ESCAPE '/'
    ```

    值为`:param`，值为`"foo/%bar"`。

+   `escape` –

    一个字符，当给定时，将使用`ESCAPE`关键字来建立该字符作为转义字符。然后可以将此字符放置在`%`和`_`的前面，以允许它们作为自身而不是通配符字符。

    一个表达式，例如：

    ```py
    somecolumn.endswith("foo/%bar", escape="^")
    ```

    将呈现为：

    ```py
    somecolumn LIKE '%' || :param ESCAPE '^'
    ```

    该参数也可以与 `ColumnOperators.endswith.autoescape` 结合使用：

    ```py
    somecolumn.endswith("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上述情况下，给定的字面参数将在传递到数据库之前转换为`"foo^%bar^^bat"`。

另请参阅

`ColumnOperators.startswith()`

`ColumnOperators.contains()`

`ColumnOperators.like()`

```py
method has(criterion: _ColumnExpressionArgument[bool] | None = None, **kwargs: Any) → ColumnElement[bool]
```

*继承自* `AssociationProxyInstance.has()` *方法的* `AssociationProxyInstance`

使用 EXISTS 生成一个代理的“has”表达式。

此表达式将使用底层代理属性的`Comparator.any()`和/或`Comparator.has()`操作符的组合产品。

```py
method icontains(other: Any, **kw: Any) → ColumnOperators
```

*继承自* `ColumnOperators.icontains()` *方法的* `ColumnOperators`

实现`icontains`运算符，例如 `ColumnOperators.contains()` 的不区分大小写版本。

生成一个 LIKE 表达式，用于对字符串值中间进行不区分大小写的匹配测试：

```py
lower(column) LIKE '%' || lower(<other>) || '%'
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.icontains("foobar"))
```

由于运算符使用`LIKE`，存在于<other>表达式内部的通配符字符`"%"`和`"_"`也将像通配符一样行事。对于字面字符串值，可以将`ColumnOperators.icontains.autoescape`标志设置为`True`，以对字符串值内这些字符的出现进行转义，使其与自身而不是通配符字符匹配。或者，`ColumnOperators.icontains.escape`参数将建立一个给定字符作为转义字符，当目标表达式不是字面字符串时，这可能会有用。

参数：

+   `other` – 要比较的表达式。通常这是一个简单的字符串值，但也可以是任意的 SQL 表达式。LIKE 通配符`%`和`_`默认情况下不会转义，除非设置了`ColumnOperators.icontains.autoescape`标志为 True。

+   `autoescape` –

    boolean；当为 True 时，在 LIKE 表达式中建立一个转义字符，然后将其应用于比较值内所有的`"%"`、`"_"`和转义字符本身，假定比较值是一个字面字符串而不是一个 SQL 表达式。

    例如表达式：

    ```py
    somecolumn.icontains("foo%bar", autoescape=True)
    ```

    将渲染为：

    ```py
    lower(somecolumn) LIKE '%' || lower(:param) || '%' ESCAPE '/'
    ```

    其中`param`的值为`"foo/%bar"`。

+   `escape` –

    一个字符，当给定时，将使用`ESCAPE`关键字将其渲染为转义字符。然后，可以将此字符放置在`%`和`_`之前，以允许它们作为自身而不是通配符字符。

    例如表达式：

    ```py
    somecolumn.icontains("foo/%bar", escape="^")
    ```

    将渲染为：

    ```py
    lower(somecolumn) LIKE '%' || lower(:param) || '%' ESCAPE '^'
    ```

    该参数也可以与`ColumnOperators.contains.autoescape`结合使用：

    ```py
    somecolumn.icontains("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上述情况下，给定的字面参数将在传递到数据库之前转换为`"foo^%bar^^bat"`。

另请参阅

`ColumnOperators.contains()`

```py
method iendswith(other: Any, escape: str | None = None, autoescape: bool = False) → ColumnOperators
```

*继承自* `ColumnOperators.iendswith()` *方法的* `ColumnOperators`

实现 `iendswith` 操作符，例如 `ColumnOperators.endswith()` 的不区分大小写版本。

生成一个 `LIKE` 表达式，对字符串值的末尾进行不区分大小写匹配：

```py
lower(column) LIKE '%' || lower(<other>)
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.iendswith("foobar"))
```

由于操作符使用了 `LIKE`，存在于 `<other>` 表达式内部的通配符 `"%"` 和 `"_"` 也将像通配符一样运行。对于文字字符串值，可以将 `ColumnOperators.iendswith.autoescape` 标志设置为 `True`，以对字符串值中这些字符的出现进行转义，使它们匹配为自身而不是通配符。另外，`ColumnOperators.iendswith.escape` 参数将建立一个给定的字符作为转义字符，当目标表达式不是文字字符串时可能会有用。

参数：

+   `other` – 需要进行比较的表达式。通常是一个普通的字符串值，但也可以是任意的 SQL 表达式。`LIKE` 通配符 `%` 和 `_` 默认情况下不会被转义，除非 `ColumnOperators.iendswith.autoescape` 标志被设置为 `True`。

+   `autoescape` –

    布尔值；当为 `True` 时，在 `LIKE` 表达式中建立一个转义字符，然后将其应用于比较值内的所有 `"%"`、`"_"` 和转义字符本身的出现，假定比较值为文字字符串而不是 SQL 表达式。

    如下表达式：

    ```py
    somecolumn.iendswith("foo%bar", autoescape=True)
    ```

    渲染为：

    ```py
    lower(somecolumn) LIKE '%' || lower(:param) ESCAPE '/'
    ```

    将值为 `:param` 的值设为 `"foo/%bar"`。

+   `escape` –

    给定一个字符，当给定时将使用 `ESCAPE` 关键字将其设置为转义字符。然后可以将此字符放在 `%` 和 `_` 的出现之前，以允许它们作为自身而不是通配符字符。

    如下表达式：

    ```py
    somecolumn.iendswith("foo/%bar", escape="^")
    ```

    渲染为：

    ```py
    lower(somecolumn) LIKE '%' || lower(:param) ESCAPE '^'
    ```

    参数也可以与 `ColumnOperators.iendswith.autoescape` 结合使用：

    ```py
    somecolumn.endswith("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上述情况下，给定的文字参数将在传递到数据库之前被转换为 `"foo^%bar^^bat"`。

另请参阅

`ColumnOperators.endswith()`

```py
method ilike(other: Any, escape: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.ilike()` *的* `ColumnOperators` *方法*

实现`ilike`运算符，例如，不区分大小写的 LIKE。

在列上下文中，产生形式之一的表达式：

```py
lower(a) LIKE lower(other)
```

或在支持 ILIKE 运算符的后端上：

```py
a ILIKE other
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.ilike("%foobar%"))
```

参数：

+   `other` - 要比较的表达式

+   `escape` -

    可选的转义字符，呈现`ESCAPE`关键字，例如：

    ```py
    somecolumn.ilike("foo/%bar", escape="/")
    ```

另请参阅

`ColumnOperators.like()`

```py
method in_(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.in_()` *的* `ColumnOperators` *方法*

实现`in`运算符。

在列上下文中，产生子句 `column IN <other>`。

给定参数 `other` 可能是：

+   字面值列表，例如：

    ```py
    stmt.where(column.in_([1, 2, 3]))
    ```

    在此调用形式中，项目列表被转换为与给定列表长度相同的一组绑定参数：

    ```py
    WHERE COL IN (?, ?, ?)
    ```

+   如果比较是针对包含多个表达式的 `tuple_()`，可以提供元组的列表：

    ```py
    from sqlalchemy import tuple_
    stmt.where(tuple_(col1, col2).in_([(1, 10), (2, 20), (3, 30)]))
    ```

+   一个空列表，例如：

    ```py
    stmt.where(column.in_([]))
    ```

    在此调用形式中，表达式呈现一个“空集”表达式。这些表达式针对各个后端进行了定制，并且通常试图获得一个空的 SELECT 语句作为子查询。例如，在 SQLite 上，表达式为：

    ```py
    WHERE col IN (SELECT 1 FROM (SELECT 1) WHERE 1!=1)
    ```

    1.4 版本中的更改：在所有情况下，空的 IN 表达式现在都使用执行时生成的 SELECT 子查询。

+   如果包含 `bindparam.expanding` 标志，可以使用绑定参数，例如 `bindparam()`：

    ```py
    stmt.where(column.in_(bindparam('value', expanding=True)))
    ```

    在此调用形式中，表达式呈现一个特殊的非 SQL 占位符表达式，如下所示：

    ```py
    WHERE COL IN ([EXPANDING_value])
    ```

    此占位符表达式在语句执行时拦截，以转换为前面所示的可变数量的绑定参数形式。如果语句执行为：

    ```py
    connection.execute(stmt, {"value": [1, 2, 3]})
    ```

    数据库将为每个值传递一个绑定参数：

    ```py
    WHERE COL IN (?, ?, ?)
    ```

    1.2 版本中的新功能：添加了“扩展”绑定参数

    如果传递了一个空列表，将呈现一个特殊的“空列表”表达式，该表达式特定于正在使用的数据库。在 SQLite 上，这将是：

    ```py
    WHERE COL IN (SELECT 1 FROM (SELECT 1) WHERE 1!=1)
    ```

    1.3 版本中的新功能：现在“扩展”绑定参数支持空列表

+   `select()` 构造，通常是相关的标量选择：

    ```py
    stmt.where(
        column.in_(
            select(othertable.c.y).
            where(table.c.x == othertable.c.x)
        )
    )
    ```

    在这种调用形式中，`ColumnOperators.in_()` 如下所示：

    ```py
    WHERE COL IN (SELECT othertable.y
    FROM othertable WHERE othertable.x = table.x)
    ```

参数：

**other** – 一组文字，一个`select()` 构造，或者一个包括``bindparam()`构造，该构造包含设置为`True`的`bindparam.expanding`标志。

```py
method is_(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.is_()` *方法的* `ColumnOperators`

实现`IS`运算符。

通常，当与`None`的值进行比较时，会自动生成`IS`，该值会解析为`NULL`。然而，在某些平台上，如果要与布尔值进行比较，则可能需要显式使用`IS`。

另请参见

`ColumnOperators.is_not()`

```py
method is_distinct_from(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.is_distinct_from()` *方法的* `ColumnOperators`

实现`IS DISTINCT FROM`运算符。

在大多数平台上呈现“a IS DISTINCT FROM b”；在某些平台上，例如 SQLite，可能呈现“a IS NOT b”。

```py
method is_not(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.is_not()` *方法的* `ColumnOperators`

实现`IS NOT`运算符。

通常，当与`None`的值进行比较时，会自动生成`IS NOT`，该值会解析为`NULL`。然而，在某些平台上，如果要与布尔值进行比较，则可能需要显式使用`IS NOT`。

从版本 1.4 开始更改：`is_not()`运算符从先前的版本中的`isnot()`重命名。以前的名称保留以确保向后兼容性。

另请参见

`ColumnOperators.is_()`

```py
method is_not_distinct_from(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.is_not_distinct_from()` *方法的* `ColumnOperators`

实现 `IS NOT DISTINCT FROM` 运算符。

在大多数平台上渲染为“a IS NOT DISTINCT FROM b”；在某些平台上（如 SQLite），可能渲染为“a IS b”。

在版本 1.4 中进行了更改：`is_not_distinct_from()` 运算符从先前版本的 `isnot_distinct_from()` 更名。先前的名称仍然可用于向后兼容。

```py
method isnot(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.isnot()` *方法的* `ColumnOperators`

实现 `IS NOT` 运算符。

通常，当与`None`的值比较时，将自动生成`IS NOT`，其解析为`NULL`。然而，在某些平台上，如果与布尔值进行比较，则可能需要显式使用`IS NOT`。

在版本 1.4 中进行了更改：`is_not()` 运算符从先前版本的 `isnot()` 更名。先前的名称仍然可用于向后兼容。

另请参见

`ColumnOperators.is_()`

```py
method isnot_distinct_from(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.isnot_distinct_from()` *方法的* `ColumnOperators`

实现 `IS NOT DISTINCT FROM` 运算符。

在大多数平台上渲染为“a IS NOT DISTINCT FROM b”；在某些平台上（如 SQLite），可能渲染为“a IS b”。

在版本 1.4 中进行了更改：`is_not_distinct_from()` 运算符从先前版本的 `isnot_distinct_from()` 更名。先前的名称仍然可用于向后兼容。

```py
method istartswith(other: Any, escape: str | None = None, autoescape: bool = False) → ColumnOperators
```

*继承自* `ColumnOperators.istartswith()` *方法的* `ColumnOperators`

实现 `istartswith` 运算符，例如 `ColumnOperators.startswith()` 的不区分大小写版本。

生成一个 LIKE 表达式，用于对字符串值的起始部分进行不区分大小写的匹配：

```py
lower(column) LIKE lower(<other>) || '%'
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.istartswith("foobar"))
```

由于该运算符使用 `LIKE`，因此存在于<other>表达式中的通配符字符 `"%"` 和 `"_"` 也将像通配符一样起作用。对于字面字符串值，可以将`ColumnOperators.istartswith.autoescape`标志设置为`True`，以对字符串值内这些字符的出现进行转义，使它们匹配为自身而不是通配符字符。或者，`ColumnOperators.istartswith.escape`参数将建立一个给定字符作为转义字符，当目标表达式不是字面字符串时可能会有用。

参数：

+   `other` – 要比较的表达式。通常是一个普通的字符串值，但也可以是任意的 SQL 表达式。除非将`ColumnOperators.istartswith.autoescape`标志设置为 True，否则不会默认转义 LIKE 通配符字符 `%` 和 `_`。

+   `autoescape` –

    布尔值；当为 True 时，在 LIKE 表达式中建立一个转义字符，然后将其应用于比较值中所有出现的`"%"`、`"_"`和转义字符本身，假定比较值是一个字面字符串而不是 SQL 表达式。

    例如：

    ```py
    somecolumn.istartswith("foo%bar", autoescape=True)
    ```

    将呈现为：

    ```py
    lower(somecolumn) LIKE lower(:param) || '%' ESCAPE '/'
    ```

    其值为 `:param` 的 `"foo/%bar"`。

+   `escape` –

    一个给定的字符，当给定时将使用 `ESCAPE` 关键字来建立该字符作为转义字符。然后可以将此字符放在 `%` 和 `_` 的前面，以允许它们作为自身而不是通配符字符。

    例如：

    ```py
    somecolumn.istartswith("foo/%bar", escape="^")
    ```

    将呈现为：

    ```py
    lower(somecolumn) LIKE lower(:param) || '%' ESCAPE '^'
    ```

    参数也可以与`ColumnOperators.istartswith.autoescape`结合使用：

    ```py
    somecolumn.istartswith("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上述情况下，给定的字面参数将在传递到数据库之前转换为`"foo^%bar^^bat"`。

另请参见

`ColumnOperators.startswith()`

```py
method like(other: Any, escape: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.like()` *方法的* `ColumnOperators`

实现 `like` 运算符。

在列上下文中，生成表达式：

```py
a LIKE other
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.like("%foobar%"))
```

参数：

+   `other` – 要比较的表达式

+   `escape` –

    可选的转义字符，渲染`ESCAPE`关键字，例如：

    ```py
    somecolumn.like("foo/%bar", escape="/")
    ```

另请参阅

`ColumnOperators.ilike()`

```py
attribute local_attr
```

*继承自* `AssociationProxyInstance.local_attr` *属性的* `AssociationProxyInstance`

此处引用的‘local’类属性由`AssociationProxyInstance`引用。

另请参阅

`AssociationProxyInstance.attr`

`AssociationProxyInstance.remote_attr`

```py
method match(other: Any, **kwargs: Any) → ColumnOperators
```

*继承自* `ColumnOperators.match()` *方法的* `ColumnOperators`

实现特定于数据库的‘match’运算符。

`ColumnOperators.match()` 尝试解析为后端提供的类似 MATCH 的函数或运算符。例如：

+   PostgreSQL - 渲染`x @@ plainto_tsquery(y)`

    > 在 2.0 版本中更改：现在针对 PostgreSQL 使用`plainto_tsquery()`而不是`to_tsquery()`；为了与其他形式兼容，请参阅全文搜索。

+   MySQL - 渲染`MATCH (x) AGAINST (y IN BOOLEAN MODE)`

    另请参阅

    `match` - 具有额外功能的 MySQL 特定构造。

+   Oracle - 渲染`CONTAINS(x, y)`

+   其他后端可能提供特殊实现。

+   没有任何特殊实现的后端将将运算符输出为“MATCH”。例如，这与 SQLite 兼容。

```py
method not_ilike(other: Any, escape: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.not_ilike()` *方法的* `ColumnOperators`

实现`NOT ILIKE`运算符。

这等同于使用对 `ColumnOperators.ilike()` 的否定，即 `~x.ilike(y)`。

在 1.4 版本中更改：`not_ilike()`运算符从之前的版本中的`notilike()`重命名。以前的名称仍可用于向后兼容。

参见

`ColumnOperators.ilike()`

```py
method not_in(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.not_in()` *方法的* `ColumnOperators`

实现`NOT IN`运算符。

这等同于使用对 `ColumnOperators.in_()` 的否定，即 `~x.in_(y)`。

在`other`是空序列的情况下，编译器产生一个“空的不在”表达式。这默认为表达式“1 = 1”，以在所有情况下产生真值。 `create_engine.empty_in_strategy` 可用于更改此行为。

在 1.4 版本中更改：`not_in()`运算符从之前的版本中的`notin_()`重命名。以前的名称仍可用于向后兼容。

在 1.2 版本中更改：`ColumnOperators.in_()` 和 `ColumnOperators.not_in()` 运算符现在默认为一个空的 IN 序列生成一个“静态”表达式。

参见

`ColumnOperators.in_()`

```py
method not_like(other: Any, escape: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.not_like()` *方法的* `ColumnOperators`

实现`NOT LIKE`运算符。

这等同于使用对 `ColumnOperators.like()` 的否定，即 `~x.like(y)`。

在 1.4 版本中更改：`not_like()`运算符从之前的版本中的`notlike()`重命名。以前的名称仍可用于向后兼容。

参见

`ColumnOperators.like()`。

```py
method notilike(other: Any, escape: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.notilike()` *的方法* `ColumnOperators`。

实现`NOT ILIKE`操作符。

这相当于在`ColumnOperators.ilike()`中使用否定，即`~x.ilike(y)`。

从版本 1.4 开始更改：`not_ilike()`操作符从之前的版本中的`notilike()`重命名。以前的名称仍然可用以保持向后兼容性。

另请参阅。

`ColumnOperators.ilike()`。

```py
method notin_(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.notin_()` *的方法* `ColumnOperators`。

实现`NOT IN`操作符。

这相当于在`ColumnOperators.in_()`中使用否定，即`~x.in_(y)`。

如果`other`是一个空序列，编译器将产生一个“空不在”表达式。默认情况下，这会变成表达式“1 = 1”，以在所有情况下产生 true。`create_engine.empty_in_strategy`可以用来改变这种行为。

从版本 1.4 开始更改：`not_in()`操作符从之前的版本中的`notin_()`重命名。以前的名称仍然可用以保持向后兼容性。

从版本 1.2 开始更改：`ColumnOperators.in_()`和`ColumnOperators.not_in()`操作符现在默认情况下为一个空的 IN 序列产生一个“静态”表达式。

另请参阅。

`ColumnOperators.in_()`。

```py
method notlike(other: Any, escape: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.notlike()` *方法的* `ColumnOperators`

实现`NOT LIKE`运算符。

这等同于使用`ColumnOperators.like()`的否定，即`~x.like(y)`。

从版本 1.4 开始更改：`not_like()`运算符从先前版本的`notlike()`重命名。以确保向后兼容性，先前的名称仍然可用。

另请参阅

`ColumnOperators.like()`

```py
method nulls_first() → ColumnOperators
```

*继承自* `ColumnOperators.nulls_first()` *方法的* `ColumnOperators`

生成针对父对象的`nulls_first()`子句。

从版本 1.4 开始更改：`nulls_first()`运算符从先前版本的`nullsfirst()`重命名。以确保向后兼容性，先前的名称仍然可用。

```py
method nulls_last() → ColumnOperators
```

*继承自* `ColumnOperators.nulls_last()` *方法的* `ColumnOperators`

生成针对父对象的`nulls_last()`子句。

从版本 1.4 开始更改：`nulls_last()`运算符从先前版本的`nullslast()`重命名。以确保向后兼容性，先前的名称仍然可用。

```py
method nullsfirst() → ColumnOperators
```

*继承自* `ColumnOperators.nullsfirst()` *方法的* `ColumnOperators`

生成针对父对象的`nulls_first()`子句。

从版本 1.4 开始更改：`nulls_first()`运算符从先前版本的`nullsfirst()`重命名。以确保向后兼容性，先前的名称仍然可用。

```py
method nullslast() → ColumnOperators
```

*继承自* `ColumnOperators.nullslast()` *方法的* `ColumnOperators`

生成一个针对父对象的`nulls_last()` 子句。

1.4 版更改：在以前的版本中，`nulls_last()`操作符从`nullslast()`重命名。以前的名称仍可用于向后兼容。

```py
method op(opstring: str, precedence: int = 0, is_comparison: bool = False, return_type: Type[TypeEngine[Any]] | TypeEngine[Any] | None = None, python_impl: Callable[..., Any] | None = None) → Callable[[Any], Operators]
```

*继承自* `Operators.op()` *方法的* `Operators`

生成一个通用的运算符函数。

例如：

```py
somecolumn.op("*")(5)
```

产生：

```py
somecolumn * 5
```

此函数还可用于明确表示按位运算符。例如：

```py
somecolumn.op('&')(0xff)
```

是`somecolumn`中的值的按位与。

参数：

+   `opstring` – 将作为此元素与传递给生成函数的表达式之间的中缀运算符输出的字符串。

+   `precedence` –

    数据库预计将在 SQL 表达式中应用到运算符的优先级。这个整数值作为 SQL 编译器的提示，用于知道何时应该在特定操作周围渲染显式括号。较低的数字将导致在应用到具有较高优先级的另一个运算符时表达式被括起来。默认值为`0`，低于所有运算符，除了逗号（`,`）和`AS`运算符。值为 100 将高于或等于所有运算符，而-100 将低于或等于所有运算符。

    另请参阅

    我正在使用 op()生成自定义运算符，但我的括号显示不正确 - SQLAlchemy SQL 编译器如何渲染括号的详细描述

+   `is_comparison` –

    遗留；如果为 True，则运算符将被视为“比较”运算符，即评估为布尔值的运算符，如`==`，`>`等。提供此标志是为了 ORM 关系能够在自定义连接条件中使用运算符时建立该运算符是比较运算符的关系。

    使用`is_comparison`参数已被使用`Operators.bool_op()` 方法取代；这更简洁的操作符会自动设置此参数，但也提供了正确的[**PEP 484**](https://peps.python.org/pep-0484/) 类型支持，因为返回的对象将表示“布尔”数据类型，即 `BinaryExpression[bool]`。

+   `return_type` – `TypeEngine` 类或对象，将强制此操作符产生的表达式的返回类型为该类型。默认情况下，指定了 `Operators.op.is_comparison` 的操作符将解析为 `Boolean`，而那些没有的操作符将与左操作数相同类型。

+   `python_impl` –

    一个可选的 Python 函数，可以以与数据库服务器上运行此操作符时相同的方式评估两个 Python 值。对于在 Python 中的 SQL 表达式评估函数（例如用于 ORM 混合属性的函数）以及用于在多行更新或删除后匹配会话中的对象的 ORM “评估器”非常有用。

    例如：

    ```py
    >>> expr = column('x').op('+', python_impl=lambda a, b: a + b)('y')
    ```

    上述表达式的操作符也适用于非 SQL 左右对象：

    ```py
    >>> expr.operator(5, 10)
    15
    ```

    新版本 2.0 中。

另请参阅

`Operators.bool_op()`

重新定义和创建新操作符

在连接条件中使用自定义操作符

```py
method operate(op: OperatorType, *other: Any, **kwargs: Any) → Operators
```

*继承自* `Operators.operate()` *方法的* `Operators`

对参数进行操作。

这是操作的最低级别，默认情况下引发 `NotImplementedError`。

在子类上重写此方法可以使通用行为应用于所有操作。例如，重写 `ColumnOperators` 以对左侧和右侧应用 `func.lower()`：

```py
class MyComparator(ColumnOperators):
    def operate(self, op, other, **kwargs):
        return op(func.lower(self), func.lower(other), **kwargs)
```

参数：

+   `op` – 操作符可调用对象。

+   `*other` – 操作的‘另一’方。对于大多数操作来说，将是一个单一的标量。

+   `**kwargs` – 修饰符。这些可以通过特殊操作符传递，比如 `ColumnOperators.contains()`。

```py
method regexp_match(pattern: Any, flags: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.regexp_match()` *方法的* `ColumnOperators`

实现了数据库特定的‘正则匹配’操作符。

例如：

```py
stmt = select(table.c.some_column).where(
    table.c.some_column.regexp_match('^(b|c)')
)
```

`ColumnOperators.regexp_match()` 尝试解析为后端提供的类似 REGEXP 的函数或操作符，但是特定的正则表达式语法和可用标志**不是与后端无关的**。

示例包括：

+   PostgreSQL - 在否定时呈现 `x ~ y` 或 `x !~ y`。

+   Oracle - 呈现 `REGEXP_LIKE(x, y)`

+   SQLite - 使用 SQLite 的 `REGEXP` 占位符操作符，并调用 Python 的 `re.match()` 内建函数。

+   其他后端可能提供特殊的实现。

+   没有任何特殊实现的后端将发出操作符为 “REGEXP” 或 “NOT REGEXP”。例如，这与 SQLite 和 MySQL 兼容。

目前正则表达式支持已经在 Oracle、PostgreSQL、MySQL 和 MariaDB 中实现。SQLite 可用部分支持。第三方方言中的支持可能有所不同。

参数：

+   `pattern` – 正则表达式模式字符串或列子句。

+   `flags` – 任何要应用的正则表达式字符串标志，仅作为普通 Python 字符串传递。这些标志是特定于后端的。某些后端，如 PostgreSQL 和 MariaDB，可能会将标志作为模式的一部分来指定。在 PostgreSQL 中使用忽略大小写标志 ‘i’ 时，将使用忽略大小写的正则表达式匹配操作符 `~*` 或 `!~*`。

从版本 1.4 开始。

从版本 1.4.48 改变为：2.0.18 请注意，由于实现错误，以前的“flags”参数除了纯 Python 字符串外，还接受 SQL 表达式对象（例如列表达式）。该实现在缓存方面不能正常工作，已被移除；应该只传递字符串作为“flags”参数，因为这些标志会被渲染为 SQL 表达式中的文字内联值。

另请参阅

`ColumnOperators.regexp_replace()`

```py
method regexp_replace(pattern: Any, replacement: Any, flags: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.regexp_replace()` *方法的* `ColumnOperators`

实现了数据库特定的“正则表达式替换”操作符。

例如：

```py
stmt = select(
    table.c.some_column.regexp_replace(
        'b(..)',
        'XY',
        flags='g'
    )
)
```

`ColumnOperators.regexp_replace()` 尝试解析为后端提供的类似 REGEXP_REPLACE 的函数，通常会发出函数 `REGEXP_REPLACE()`。但是，特定的正则表达式语法和可用标志**不是与后端无关的**。

目前为止，正则表达式替换支持已经在 Oracle、PostgreSQL、MySQL 8 或更高版本和 MariaDB 中实现。第三方方言中的支持可能有所不同。

参数：

+   `pattern` – 正则表达式模式字符串或列子句。

+   `pattern` – 替换字符串或列子句。

+   `flags` – 要应用的任何正则表达式字符串标志，仅作为普通 Python 字符串传递。这些标志是后端特定的。某些后端，如 PostgreSQL 和 MariaDB，可能会将标志作为模式的一部分来指定。

版本 1.4 中的新功能。

从版本 1.4.48 开始更改，: 2.0.18 请注意，由于实现错误，“flags”参数先前接受了 SQL 表达式对象，如列表达式，除了普通的 Python 字符串。此实现与缓存不正确，已删除；仅应传递字符串以用于“flags”参数，因为这些标志将作为 SQL 表达式中的文字内联值呈现。

另请参阅

`ColumnOperators.regexp_match()`。

```py
attribute remote_attr
```

*继承自* `AssociationProxyInstance.remote_attr` *属性的* `AssociationProxyInstance`。

此 `AssociationProxyInstance` 引用的“remote”类属性。

另请参阅

`AssociationProxyInstance.attr`。

`AssociationProxyInstance.local_attr`。

```py
method reverse_operate(op: OperatorType, other: Any, **kwargs: Any) → Operators
```

*继承自* `Operators.reverse_operate()` *方法的* `Operators`。

对参数进行反向操作。

用法与 `operate()` 相同。

```py
attribute scalar
```

*继承自* `AssociationProxyInstance.scalar` *属性的* `AssociationProxyInstance`。

如果此 `AssociationProxyInstance` 代理本地端的标量关系，则返回`True`。

```py
method startswith(other: Any, escape: str | None = None, autoescape: bool = False) → ColumnOperators
```

*继承自* `ColumnOperators.startswith()` *方法的* `ColumnOperators`

实现 `startswith` 运算符。

生成一个 LIKE 表达式，用于测试字符串值的开头匹配：

```py
column LIKE <other> || '%'
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.startswith("foobar"))
```

由于该运算符使用 `LIKE`，因此存在于 <other> 表达式中的通配符字符 `"%"` 和 `"_"` 也将像通配符一样起作用。对于字面字符串值，可以将 `ColumnOperators.startswith.autoescape` 标志设置为 `True`，以对字符串值中这些字符的出现进行转义，使它们与自身匹配而不是作为通配符字符。或者，`ColumnOperators.startswith.escape` 参数将建立一个给定的字符作为转义字符，当目标表达式不是字面字符串时，这可能会有用。

参数：

+   `other` – 要比较的表达式。通常是一个普通字符串值，但也可以是任意的 SQL 表达式。除非将 `ColumnOperators.startswith.autoescape` 标志设置为 True，否则 LIKE 通配符 `%` 和 `_` 默认不会被转义。

+   `autoescape` –

    布尔值；当为 True 时，在 LIKE 表达式中建立一个转义字符，然后将其应用于比较值中所有出现的 `"%"`、`"_"` 和转义字符本身，假定比较值是一个字面字符串而不是 SQL 表达式。

    诸如以下表达式：

    ```py
    somecolumn.startswith("foo%bar", autoescape=True)
    ```

    将呈现为：

    ```py
    somecolumn LIKE :param || '%' ESCAPE '/'
    ```

    其值为 `:param`，为 `"foo/%bar"`。

+   `escape` –

    一个字符，当给定时，将使用 `ESCAPE` 关键字来建立该字符作为转义字符。然后可以将该字符放在 `%` 和 `_` 的前面，以使它们作为自身而不是通配符字符。

    诸如以下表达式：

    ```py
    somecolumn.startswith("foo/%bar", escape="^")
    ```

    将呈现为：

    ```py
    somecolumn LIKE :param || '%' ESCAPE '^'
    ```

    该参数也可以与 `ColumnOperators.startswith.autoescape` 结合使用：

    ```py
    somecolumn.startswith("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上述情况下，给定的字面参数将在传递到数据库之前转换为 `"foo^%bar^^bat"`。

另请参阅

`ColumnOperators.endswith()`

`ColumnOperators.contains()`

`ColumnOperators.like()`

```py
attribute target_class: Type[Any]
```

此 `AssociationProxyInstance` 处理的中间类。

拦截的追加/设置/赋值事件将导致生成此类的新实例。

```py
attribute timetuple: Literal[None] = None
```

*继承自* `ColumnOperators` *的* `ColumnOperators.timetuple` *属性。

允许将 datetime 对象放在左侧进行比较的 Hack。

```py
class sqlalchemy.ext.associationproxy.ColumnAssociationProxyInstance
```

一个以数据库列为目标的 `AssociationProxyInstance`。

**成员**

__le__(), __lt__(), __ne__(), all_(), any(), any_(), asc(), attr, between(), bitwise_and(), bitwise_lshift(), bitwise_not(), bitwise_or(), bitwise_rshift(), bitwise_xor(), bool_op(), collate(), concat(), contains(), desc(), distinct(), endswith(), has(), icontains(), iendswith(), ilike(), in_(), is_(), is_distinct_from(), is_not(), is_not_distinct_from(), isnot(), isnot_distinct_from(), istartswith(), like(), local_attr, match(), not_ilike(), not_in(), not_like(), notilike(), notin_(), notlike(), nulls_first(), nulls_last(), nullsfirst(), nullslast(), op(), operate(), regexp_match(), regexp_replace(), remote_attr, reverse_operate(), scalar, startswith(), target_class, timetuple

**类签名**

类 `sqlalchemy.ext.associationproxy.ColumnAssociationProxyInstance` (`sqlalchemy.ext.associationproxy.AssociationProxyInstance`)

```py
method __le__(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators` *的* `sqlalchemy.sql.expression.ColumnOperators.__le__` *方法。

实现 `<=` 运算符。

在列上下文中，生成子句 `a <= b`。

```py
method __lt__(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators` *的* `sqlalchemy.sql.expression.ColumnOperators.__lt__` *方法。

实现 `<` 运算符。

在列上下文中，生成子句 `a < b`。

```py
method __ne__(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators` *的* `sqlalchemy.sql.expression.ColumnOperators.__ne__` *方法。

实现 `!=` 运算符。

在列上下文中，生成子句 `a != b`。如果目标是 `None`，则生成 `a IS NOT NULL`。

```py
method all_() → ColumnOperators
```

*继承自* `ColumnOperators` *的* `ColumnOperators.all_()` *方法。

对父对象生成一个`all_()`子句。

查看 `all_()` 的文档以获取示例。

注意

请确保不要混淆较新的 `ColumnOperators` *的* `ColumnOperators.all_()` *方法与**传统**版本的此方法，即 `Comparator.all()` 方法，该方法专门针对 `ARRAY`，使用不同的调用风格。

```py
method any(criterion: _ColumnExpressionArgument[bool] | None = None, **kwargs: Any) → ColumnElement[bool]
```

*继承自* `AssociationProxyInstance` *的* `AssociationProxyInstance.any()` *方法。

使用 EXISTS 生成一个代理的 'any' 表达式。

此表达式将使用基础代理属性的`Comparator.any()`和/或`Comparator.has()`运算符组合成一个产品。

```py
method any_() → ColumnOperators
```

*继承自* `ColumnOperators.any_()` *方法的* `ColumnOperators`

生成针对父对象的`any_()`子句。

有关示例，请参阅`any_()`的文档。

注意

请确保不要混淆较新的`ColumnOperators.any_()`方法与此方法的**传统**版本，即专用于`ARRAY`的`Comparator.any()`方法，其使用不同的调用风格。

```py
method asc() → ColumnOperators
```

*继承自* `ColumnOperators.asc()` *方法的* `ColumnOperators`

生成针对父对象的`asc()`子句。

```py
attribute attr
```

*继承自* `AssociationProxyInstance.attr` *属性的* `AssociationProxyInstance`

返回一个元组`(local_attr, remote_attr)`。

此属性最初旨在简化使用`Query.join()`方法一次性跨越两个关系，但这使用了一个已弃用的调用风格。

要使用`select.join()`或`Query.join()`与关联代理，当前方法是分别使用`AssociationProxyInstance.local_attr`和`AssociationProxyInstance.remote_attr`属性：

```py
stmt = (
    select(Parent).
    join(Parent.proxied.local_attr).
    join(Parent.proxied.remote_attr)
)
```

未来的版本可能会为关联代理属性提供更简洁的连接模式。

请参见

`AssociationProxyInstance.local_attr`

`AssociationProxyInstance.remote_attr`

```py
method between(cleft: Any, cright: Any, symmetric: bool = False) → ColumnOperators
```

*继承自* `ColumnOperators.between()` *方法的* `ColumnOperators`

生成针对父对象的 `between()` 子句，给定下限和上限范围。

```py
method bitwise_and(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_and()` *方法的* `ColumnOperators`

生成按位 AND 操作，通常通过 `&` 运算符。

2.0.2 版本中的新功能。

请参见

按位运算符

```py
method bitwise_lshift(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_lshift()` *方法的* `ColumnOperators`

生成按位 LSHIFT 操作，通常通过 `<<` 运算符。

2.0.2 版本中的新功能。

请参见

按位运算符

```py
method bitwise_not() → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_not()` *方法的* `ColumnOperators`

生成按位 NOT 操作，通常通过 `~` 运算符。

2.0.2 版本中的新功能。

请参见

按位运算符

```py
method bitwise_or(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_or()` *方法的* `ColumnOperators`

生成按位 OR 操作，通常通过 `|` 运算符。

2.0.2 版本中的新功能。

请参见

按位运算符

```py
method bitwise_rshift(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_rshift()` *方法的* `ColumnOperators`

生成位右移操作，通常通过`>>`运算符。

版本 2.0.2 中的新功能。

另请参阅

位运算符

```py
method bitwise_xor(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_xor()` *方法的* `ColumnOperators`

生成位异或操作，通常通过`^`运算符，或者对于 PostgreSQL 使用`#`。

版本 2.0.2 中的新功能。

另请参阅

位运算符

```py
method bool_op(opstring: str, precedence: int = 0, python_impl: Callable[[...], Any] | None = None) → Callable[[Any], Operators]
```

*继承自* `Operators.bool_op()` *方法的* `Operators`

返回自定义布尔运算符。

此方法是调用`Operators.op()`并传递`Operators.op.is_comparison`标志为 True 的简写。 使用`Operators.bool_op()`的一个关键优势是，在使用列构造时，返回表达式的“布尔”性质将出现在[**PEP 484**](https://peps.python.org/pep-0484/)中。

另请参阅

`Operators.op()`

```py
method collate(collation: str) → ColumnOperators
```

*继承自* `ColumnOperators.collate()` *方法的* `ColumnOperators`

生成针对父对象的`collate()`子句，给定排序字符串。

另请参阅

`collate()`

```py
method concat(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.concat()` *方法的* `ColumnOperators`

实现‘连接’操作符。

在列上下文中，产生 `a || b` 子句，或在 MySQL 上使用 `concat()` 操作符。

```py
method contains(other: Any, **kw: Any) → ColumnOperators
```

*继承自* `ColumnOperators.contains()` *方法的* `ColumnOperators`

实现‘包含’操作符。

产生一个用于测试字符串值中间匹配的 LIKE 表达式：

```py
column LIKE '%' || <other> || '%'
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.contains("foobar"))
```

由于操作符使用了 `LIKE`，所以在 <other> 表达式内出现的通配符字符 `"%"` 和 `"_"` 也会像通配符一样运行。对于字面字符串值，可以设置 `ColumnOperators.contains.autoescape` 标志为 `True` 来对字符串值内出现的这些字符进行转义，使它们与自身匹配而不是作为通配符字符。或者，`ColumnOperators.contains.escape` 参数将建立一个给定的字符作为转义字符，这在目标表达式不是字面字符串时可能会有用。

参数：

+   `other` – 要进行比较的表达式。这通常是一个普通的字符串值，但也可以是任意的 SQL 表达式。除非设置了 `ColumnOperators.contains.autoescape` 标志为 True，否则 LIKE 通配符字符 `%` 和 `_` 不会被默认转义。

+   `autoescape` –

    布尔值；当为 True 时，在 LIKE 表达式中建立一个转义字符，然后将其应用于比较值中所有出现的`"%"`、`"_"`和转义字符本身，假定比较值为字面字符串而不是 SQL 表达式。

    例如这样的表达式：

    ```py
    somecolumn.contains("foo%bar", autoescape=True)
    ```

    将渲染为：

    ```py
    somecolumn LIKE '%' || :param || '%' ESCAPE '/'
    ```

    其中`：param`的值为`"foo/%bar"`。

+   `escape` –

    当给定时，将使用 `ESCAPE` 关键字来确定该字符作为转义字符。然后，该字符可以放在 `%` 和 `_` 的前面以允许它们作为自身而不是通配符字符。

    例如这样的表达式：

    ```py
    somecolumn.contains("foo/%bar", escape="^")
    ```

    将渲染为：

    ```py
    somecolumn LIKE '%' || :param || '%' ESCAPE '^'
    ```

    参数还可以与 `ColumnOperators.contains.autoescape` 结合使用：

    ```py
    somecolumn.contains("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上述情况下，给定的字面参数在传递到数据库之前将被转换为`"foo^%bar^^bat"`。

另请参阅

`ColumnOperators.startswith()`

`ColumnOperators.endswith()`

`ColumnOperators.like()`

```py
method desc() → ColumnOperators
```

*继承自* `ColumnOperators.desc()` *方法的* `ColumnOperators`

生成一个针对父对象的 `desc()` 子句。

```py
method distinct() → ColumnOperators
```

*继承自* `ColumnOperators.distinct()` *方法的* `ColumnOperators`

生成一个针对父对象的 `distinct()` 子句。

```py
method endswith(other: Any, escape: str | None = None, autoescape: bool = False) → ColumnOperators
```

*继承自* `ColumnOperators.endswith()` *方法的* `ColumnOperators`

实现‘endswith’运算符。

生成一个 LIKE 表达式，用于测试字符串值的结尾是否匹配：

```py
column LIKE '%' || <other>
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.endswith("foobar"))
```

由于该操作符使用`LIKE`，在<other>表达式中存在的通配符字符`"%"`和`"_"`也将像通配符一样起作用。对于字面字符串值，`ColumnOperators.endswith.autoescape`标志可以设置为`True`，以对字符串值中这些字符的出现应用转义，使它们匹配为自身而不是通配符字符。或者，`ColumnOperators.endswith.escape`参数将建立一个给定字符作为转义字符，当目标表达式不是字面字符串时可能会有用。

参数：

+   `other` – 要比较的表达式。这通常是一个简单的字符串值，但也可以是任意的 SQL 表达式。默认情况下，LIKE 通配符字符`%`和`_`不会被转义，除非设置了`ColumnOperators.endswith.autoescape`标志为 True。

+   `autoescape` –

    布尔值；当为 True 时，在 LIKE 表达式中建立一个转义字符，然后将其应用于比较值中所有的`"%"`、`"_"`和转义字符本身的出现，假定比较值是一个字面字符串而不是 SQL 表达式。

    一个表达式如下：

    ```py
    somecolumn.endswith("foo%bar", autoescape=True)
    ```

    将呈现为：

    ```py
    somecolumn LIKE '%' || :param ESCAPE '/'
    ```

    使用值`:param`为`"foo/%bar"`。

+   `escape` –

    一个字符，当给定时，将使用`ESCAPE`关键字将该字符建立为转义字符。然后可以将该字符放在`%`和`_`的前面，以允许它们作为自身而不是通配符字符。

    一个表达式如下：

    ```py
    somecolumn.endswith("foo/%bar", escape="^")
    ```

    将呈现为：

    ```py
    somecolumn LIKE '%' || :param ESCAPE '^'
    ```

    该参数也可以与`ColumnOperators.endswith.autoescape`结合使用：

    ```py
    somecolumn.endswith("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上述情况下，给定的字面参数将在传递到数据库之前转换为`"foo^%bar^^bat"`。

另请参见

`ColumnOperators.startswith()`

`ColumnOperators.contains()`

`ColumnOperators.like()`

```py
method has(criterion: _ColumnExpressionArgument[bool] | None = None, **kwargs: Any) → ColumnElement[bool]
```

*继承自* `AssociationProxyInstance.has()` *方法的* `AssociationProxyInstance`

生成一个使用 EXISTS 的代理‘has’表达式。

这个表达式将是使用基础代理属性的`Comparator.any()`和/或`Comparator.has()`运算符组成的复合产品。

```py
method icontains(other: Any, **kw: Any) → ColumnOperators
```

*继承自* `ColumnOperators.icontains()` *方法的* `ColumnOperators`

实现 `icontains` 运算符，例如 `ColumnOperators.contains()` 的不区分大小写版本。

生成一个 LIKE 表达式，测试一个字符串值中间的不区分大小写匹配：

```py
lower(column) LIKE '%' || lower(<other>) || '%'
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.icontains("foobar"))
```

由于该运算符使用了`LIKE`，因此存在于<other>表达式内部的通配符字符`"%"`和`"_"`也会像通配符一样起作用。对于字面字符串值，可以将 `ColumnOperators.icontains.autoescape` 标志设置为`True`，以对字符串值内这些字符的出现进行转义，使它们与自身匹配而不是作为通配符字符。或者，`ColumnOperators.icontains.escape` 参数将建立一个给定字符作为转义字符，当目标表达式不是字面字符串时可能会有用。

参数：

+   `other` – 要比较的表达式。这通常是一个普通字符串值，但也可以是一个任意的 SQL 表达式。默认情况下，不会转义 LIKE 通配符字符`%`和`_`，除非设置了 `ColumnOperators.icontains.autoescape` 标志为 True。

+   `autoescape` –

    布尔值；当为 True 时，在 LIKE 表达式中建立一个转义字符，然后将其应用于比较值中所有出现的`"%"`、`"_"`和转义字符本身，假设比较值是一个字面字符串而不是一个 SQL 表达式。

    例如：

    ```py
    somecolumn.icontains("foo%bar", autoescape=True)
    ```

    渲染为：

    ```py
    lower(somecolumn) LIKE '%' || lower(:param) || '%' ESCAPE '/'
    ```

    其值为 `:param`，如 `"foo/%bar"`。

+   `escape` –

    当给定一个字符时，将使用 `ESCAPE` 关键字将其设置为转义字符。然后可以在 `%` 和 `_` 的前面放置该字符，以允许它们被视为其自身而不是通配符字符。

    例如：

    ```py
    somecolumn.icontains("foo/%bar", escape="^")
    ```

    将呈现为：

    ```py
    lower(somecolumn) LIKE '%' || lower(:param) || '%' ESCAPE '^'
    ```

    该参数还可以与 `ColumnOperators.contains.autoescape` 结合使用：

    ```py
    somecolumn.icontains("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上述情况下，给定的字面参数将在传递到数据库之前被转换为 `"foo^%bar^^bat"`。

另请参阅：

`ColumnOperators.contains()`

```py
method iendswith(other: Any, escape: str | None = None, autoescape: bool = False) → ColumnOperators
```

*继承自* `ColumnOperators.iendswith()` *方法的* `ColumnOperators`

实现 `iendswith` 操作符，例如对 `ColumnOperators.endswith()` 的不区分大小写版本。

生成一个 LIKE 表达式，用于对字符串值的结尾进行不区分大小写匹配：

```py
lower(column) LIKE '%' || lower(<other>)
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.iendswith("foobar"))
```

由于该操作符使用 `LIKE`，因此存在于 <other> 表达式内部的通配符字符 `"%"` 和 `"_"` 也将像通配符一样起作用。对于字面字符串值，可以将 `ColumnOperators.iendswith.autoescape` 标志设置为 `True`，以对字符串值内部这些字符的出现进行转义，使它们匹配为其自身而不是通配符字符。或者，`ColumnOperators.iendswith.escape` 参数将建立一个给定的字符作为转义字符，这在目标表达式不是字面字符串时可能会有用。

参数：

+   `other` – 表示要比较的表达式。通常是一个普通字符串值，但也可以是任意的 SQL 表达式。LIKE 通配符字符 `%` 和 `_` 默认不被转义，除非 `ColumnOperators.iendswith.autoescape` 标志被设置为 True。

+   `autoescape` –

    布尔值；当为 True 时，在 LIKE 表达式中建立转义字符，然后将其应用于比较值内的所有 `"%"`、`"_"` 和转义字符本身的出现，假定比较值为字面字符串而不是 SQL 表达式。

    诸如此类的表达式：

    ```py
    somecolumn.iendswith("foo%bar", autoescape=True)
    ```

    将渲染为：

    ```py
    lower(somecolumn) LIKE '%' || lower(:param) ESCAPE '/'
    ```

    其值为 `:param`，为 `"foo/%bar"`。

+   `escape` –

    给定一个字符，它将与 `ESCAPE` 关键字一起渲染，以将该字符确定为转义字符。然后，可以将该字符放在 `%` 和 `_` 的前面，以使它们充当自己而不是通配符字符。

    诸如此类的表达式：

    ```py
    somecolumn.iendswith("foo/%bar", escape="^")
    ```

    将渲染为：

    ```py
    lower(somecolumn) LIKE '%' || lower(:param) ESCAPE '^'
    ```

    参数也可以与 `ColumnOperators.iendswith.autoescape` 结合使用：

    ```py
    somecolumn.endswith("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上述情况下，给定的字面参数将在传递到数据库之前转换为 `"foo^%bar^^bat"`。

另请参见

`ColumnOperators.endswith()`

```py
method ilike(other: Any, escape: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.ilike()` *方法的* `ColumnOperators`

实现 `ilike` 运算符，例如，不区分大小写的 LIKE。

在列上下文中，生成以下形式的表达式：

```py
lower(a) LIKE lower(other)
```

或者在支持 ILIKE 运算符的后端上：

```py
a ILIKE other
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.ilike("%foobar%"))
```

参数：

+   `other` – 要比较的表达式

+   `escape` –

    可选转义字符，渲染 `ESCAPE` 关键字，例如：

    ```py
    somecolumn.ilike("foo/%bar", escape="/")
    ```

另请参见

`ColumnOperators.like()`

```py
method in_(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.in_()` *方法的* `ColumnOperators`

实现 `in` 运算符。

在列上下文中，生成子句 `column IN <other>`。

给定参数 `other` 可能是：

+   一个字面值列表，例如：

    ```py
    stmt.where(column.in_([1, 2, 3]))
    ```

    在此调用形式中，将项目列表转换为与给定列表相同长度的绑定参数集：

    ```py
    WHERE COL IN (?, ?, ?)
    ```

+   如果比较针对包含多个表达式的 `tuple_()`，则可以提供元组列表：

    ```py
    from sqlalchemy import tuple_
    stmt.where(tuple_(col1, col2).in_([(1, 10), (2, 20), (3, 30)]))
    ```

+   空列表，例如：

    ```py
    stmt.where(column.in_([]))
    ```

    在这种调用形式中，表达式渲染一个“空集”表达式。这些表达式是针对各个后端定制的，通常试图获得一个空的 SELECT 语句作为子查询。例如，在 SQLite 上，表达式是：

    ```py
    WHERE col IN (SELECT 1 FROM (SELECT 1) WHERE 1!=1)
    ```

    从版本 1.4 开始：所有情况下，空的 IN 表达式现在都使用执行时生成的 SELECT 子查询。

+   如果包含 `bindparam.expanding` 标志，则可以使用绑定参数，例如`bindparam()`：

    ```py
    stmt.where(column.in_(bindparam('value', expanding=True)))
    ```

    在这种调用形式中，表达式渲染一个特殊的非 SQL 占位符表达式，看起来像这样：

    ```py
    WHERE COL IN ([EXPANDING_value])
    ```

    此占位符表达式在语句执行时拦截，以转换为之前所示的变量数量的绑定参数形式。如果执行了语句：

    ```py
    connection.execute(stmt, {"value": [1, 2, 3]})
    ```

    对于每个值，数据库将传递一个绑定参数：

    ```py
    WHERE COL IN (?, ?, ?)
    ```

    从版本 1.2 开始：添加了“扩展”绑定参数

    如果传递了一个空列表，则渲染一个特殊的“空列表”表达式，该表达式特定于所使用的数据库。在 SQLite 上，这将是：

    ```py
    WHERE COL IN (SELECT 1 FROM (SELECT 1) WHERE 1!=1)
    ```

    从版本 1.3 开始：现在支持空列表的“扩展”绑定参数

+   一个`select()` 构造，通常是一个相关的标量选择：

    ```py
    stmt.where(
        column.in_(
            select(othertable.c.y).
            where(table.c.x == othertable.c.x)
        )
    )
    ```

    在这种调用形式中，`ColumnOperators.in_()` 渲染如下：

    ```py
    WHERE COL IN (SELECT othertable.y
    FROM othertable WHERE othertable.x = table.x)
    ```

参数：

**other** – 一个字面常量列表，一个`select()` 构造，或一个包含`bindparam()` 构造的列表，该构造设置了 `bindparam.expanding` 标志为 True。

```py
method is_(other: Any) → ColumnOperators
```

*从* `ColumnOperators.is_()` *方法继承*

实现 `IS` 运算符。

通常，当与`None`的值进行比较时，会自动生成 `IS`，它解析为`NULL`。但是，在某些平台上，如果要与布尔值进行比较，则可能希望明确使用 `IS`。

另请参见

`ColumnOperators.is_not()`

```py
method is_distinct_from(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.is_distinct_from()` *方法的* `ColumnOperators`

实现 `IS DISTINCT FROM` 操作符。

在大多数平台上呈现“a IS DISTINCT FROM b”；在某些平台（如 SQLite）上可能呈现“a IS NOT b”。

```py
method is_not(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.is_not()` *方法的* `ColumnOperators`

实现 `IS NOT` 操作符。

通常情况下，与 `None` 的值比较时会自动生成 `IS NOT`，该值解析为 `NULL`。但是，如果在某些平台上比较布尔值，则可能希望明确使用 `IS NOT`。

在 1.4 版本中更改：`is_not()` 操作符从以前的版本中的 `isnot()` 重命名为 `is_not()`。以前的名称仍可供向后兼容使用。

另请参阅

`ColumnOperators.is_()`

```py
method is_not_distinct_from(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.is_not_distinct_from()` *方法的* `ColumnOperators`

实现 `IS NOT DISTINCT FROM` 操作符。

在大多数平台上呈现“a IS NOT DISTINCT FROM b”；在某些平台（如 SQLite）上可能呈现“a IS b”。

在 1.4 版本中更改：`is_not_distinct_from()` 操作符从以前的版本中的 `isnot_distinct_from()` 重命名为 `is_not_distinct_from()`。以前的名称仍可供向后兼容使用。

```py
method isnot(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.isnot()` *方法的* `ColumnOperators`

实现 `IS NOT` 操作符。

通常情况下，与 `None` 的值比较时会自动生成 `IS NOT`，该值解析为 `NULL`。但是，如果在某些平台上比较布尔值，则可能希望明确使用 `IS NOT`。

在 1.4 版本中更改：`is_not()` 操作符从以前的版本中的 `isnot()` 重命名为 `is_not()`。以前的名称仍可供向后兼容使用。

另请参阅

`ColumnOperators.is_()`

```py
method isnot_distinct_from(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.isnot_distinct_from()` *方法的* `ColumnOperators`

实现 `IS NOT DISTINCT FROM` 运算符。

在大多数平台上呈现“a IS NOT DISTINCT FROM b”；在一些平台上如 SQLite 可能呈现“a IS b”。

在 1.4 版本中更改：`is_not_distinct_from()` 运算符从之前版本的 `isnot_distinct_from()` 重命名。以确保向后兼容性，之前的名称仍然可用。

```py
method istartswith(other: Any, escape: str | None = None, autoescape: bool = False) → ColumnOperators
```

*继承自* `ColumnOperators.istartswith()` *方法的* `ColumnOperators`

实现 `istartswith` 运算符，例如 `ColumnOperators.startswith()` 的不区分大小写版本。

生成一个对字符串值开头的不区分大小写匹配的 LIKE 表达式：

```py
lower(column) LIKE lower(<other>) || '%'
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.istartswith("foobar"))
```

由于该运算符使用 `LIKE`，因此存在于 <other> 表达式中的通配符字符 `"%"` 和 `"_"` 也将像通配符一样起作用。对于字面字符串值，可以将 `ColumnOperators.istartswith.autoescape` 标志设置为 `True`，以对字符串值中这些字符的出现进行转义，使其作为自身而不是通配符字符进行匹配。或者，`ColumnOperators.istartswith.escape` 参数将建立一个给定字符作为转义字符，当目标表达式不是字面字符串时可能会有用。

参数：

+   `other` – 要比较的表达式。通常是一个普通字符串值，但也可以是任意 SQL 表达式。除非将 `ColumnOperators.istartswith.autoescape` 标志设置为 True，否则 LIKE 通配符字符 `%` 和 `_` 不会被转义。

+   `autoescape` –

    布尔值；当为 True 时，在 LIKE 表达式中建立一个转义字符，然后将其应用于比较值中所有出现的 `"%"`、`"_"` 和转义字符本身，假定比较值是一个字面字符串而不是 SQL 表达式。

    例如表达式：

    ```py
    somecolumn.istartswith("foo%bar", autoescape=True)
    ```

    将呈现为：

    ```py
    lower(somecolumn) LIKE lower(:param) || '%' ESCAPE '/'
    ```

    其中`param`的值为`"foo/%bar"`。

+   `escape` –

    一个字符，当给定时，将呈现为带有`ESCAPE`关键字的转义字符。然后，可以将此字符放在`%`和`_`之前，以使它们作为自身而不是通配符字符。

    例如表达式：

    ```py
    somecolumn.istartswith("foo/%bar", escape="^")
    ```

    将呈现为：

    ```py
    lower(somecolumn) LIKE lower(:param) || '%' ESCAPE '^'
    ```

    参数也可以与`ColumnOperators.istartswith.autoescape`结合使用：

    ```py
    somecolumn.istartswith("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上述情况下，给定的文字参数将在传递到数据库之前转换为`"foo^%bar^^bat"`。

另请参阅

`ColumnOperators.startswith()`

```py
method like(other: Any, escape: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.like()` *方法的* `ColumnOperators`

实现`like`运算符。

在列上下文中，生成表达式：

```py
a LIKE other
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.like("%foobar%"))
```

参数：

+   `other` – 要比较的表达式

+   `escape` –

    可选的转义字符，呈现`ESCAPE`关键字，例如：

    ```py
    somecolumn.like("foo/%bar", escape="/")
    ```

另请参阅

`ColumnOperators.ilike()`

```py
attribute local_attr
```

*继承自* `AssociationProxyInstance.local_attr` *属性的* `AssociationProxyInstance`

此`AssociationProxyInstance`引用的“local”类属性。

另请参阅

`AssociationProxyInstance.attr`

`AssociationProxyInstance.remote_attr`

```py
method match(other: Any, **kwargs: Any) → ColumnOperators
```

*继承自* `ColumnOperators.match()` *方法的* `ColumnOperators`

实现了特定于数据库的“match”运算符。

`ColumnOperators.match()` 尝试解析为后端提供的 MATCH 类似的函数或运算符。示例包括：

+   PostgreSQL - 渲染`x @@ plainto_tsquery(y)`

    > 从 2.0 版更改：现在对于 PostgreSQL，使用`plainto_tsquery()`而不是`to_tsquery()`；为了与其他形式兼容，请参阅全文搜索。

+   MySQL - 渲染 `MATCH (x) AGAINST (y IN BOOLEAN MODE)`

    另请参见

    `match` - MySQL 特定的构造，具有额外的功能。

+   Oracle - 渲染 `CONTAINS(x, y)`

+   其他后端可能提供特殊的实现。

+   没有任何特殊实现的后端将以“MATCH”运算符发出。例如，这与 SQLite 兼容。

```py
method not_ilike(other: Any, escape: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.not_ilike()` *方法的* `ColumnOperators`

实现`NOT ILIKE`运算符。

这等同于使用`ColumnOperators.ilike()`进行否定，即`~x.ilike(y)`。

从 1.4 版更改：`not_ilike()`运算符在先前版本中从`notilike()`重命名。以确保向后兼容性，之前的名称仍可用。

另请参见

`ColumnOperators.ilike()`

```py
method not_in(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.not_in()` *方法的* `ColumnOperators`

实现`NOT IN`运算符。

这相当于使用`ColumnOperators.in_()`进行否定，即`~x.in_(y)`。

在`other`为空序列的情况下，编译器会生成一个“空的 not in”表达式。默认情况下，这会产生表达式“1 = 1”，以在所有情况下生成 true。可以使用`create_engine.empty_in_strategy`来修改此行为。

从 1.4 版更改：`not_in()`运算符在先前版本中从`notin_()`重命名。以确保向后兼容性，之前的名称仍可用。

从版本 1.2 开始更改：`ColumnOperators.in_()`和`ColumnOperators.not_in()`运算符现在默认生成空 IN 序列的“静态”表达式。

另请参阅

`ColumnOperators.in_()`

```py
method not_like(other: Any, escape: str | None = None) → ColumnOperators
```

*从* `ColumnOperators.not_like()` *方法继承的* `ColumnOperators`

实现`NOT LIKE`运算符。

这相当于使用`ColumnOperators.like()`的否定，即`~x.like(y)`。

从版本 1.4 开始更改：`not_like()`运算符从之前的发布中的`notlike()`重新命名。以前的名称保留用于向后兼容。

另请参阅

`ColumnOperators.like()`

```py
method notilike(other: Any, escape: str | None = None) → ColumnOperators
```

*从* `ColumnOperators.notilike()` *方法继承的* `ColumnOperators`

实现`NOT ILIKE`运算符。

这相当于使用`ColumnOperators.ilike()`的否定，即`~x.ilike(y)`。

从版本 1.4 开始更改：`not_ilike()`运算符从之前的发布中的`notilike()`重新命名。以前的名称保留用于向后兼容。

另请参阅

`ColumnOperators.ilike()`

```py
method notin_(other: Any) → ColumnOperators
```

*从* `ColumnOperators.notin_()` *方法继承的* `ColumnOperators`

实现`NOT IN`运算符。

这相当于使用`ColumnOperators.in_()`的否定，即`~x.in_(y)`。

如果 `other` 是一个空序列，编译器将生成一个“空不在”表达式。这默认为表达式 “1 = 1”，以在所有情况下生成 true。 `create_engine.empty_in_strategy` 可用于更改此行为。

从版本 1.4 起更改：`not_in()` 操作符从以前的版本中的 `notin_()` 重命名。以前的名称仍可用于向后兼容性。

从版本 1.2 起更改：`ColumnOperators.in_()` 和 `ColumnOperators.not_in()` 操作符现在默认情况下为一个空的 IN 序列生成一个 “静态” 表达式。

另请参阅

`ColumnOperators.in_()`

```py
method notlike(other: Any, escape: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.notlike()` *方法的* `ColumnOperators`

实现 `NOT LIKE` 操作符。

这等同于使用 `ColumnOperators.like()` 的否定，即 `~x.like(y)`。

从版本 1.4 起更改：`not_like()` 操作符从以前的版本中的 `notlike()` 重命名。以前的名称仍可用于向后兼容性。

另请参阅

`ColumnOperators.like()`

```py
method nulls_first() → ColumnOperators
```

*继承自* `ColumnOperators.nulls_first()` *方法的* `ColumnOperators`

对父对象生成一个 `nulls_first()` 子句。

从版本 1.4 起更改：`nulls_first()` 操作符从以前的版本中的 `nullsfirst()` 重命名。以前的名称仍可用于向后兼容性。

```py
method nulls_last() → ColumnOperators
```

*继承自* `ColumnOperators.nulls_last()` *方法的* `ColumnOperators`

对父对象生成一个`nulls_last()`子句。

从版本 1.4 开始更改：`nulls_last()` 操作符从之前的发布中的 `nullslast()` 改名。 以前的名称仍然可用于向后兼容。

```py
method nullsfirst() → ColumnOperators
```

*继承自* `ColumnOperators.nullsfirst()` *方法的* `ColumnOperators`

对父对象生成一个`nulls_first()`子句。

从版本 1.4 开始更改：`nulls_first()` 操作符从之前的发布中的 `nullsfirst()` 改名。 以前的名称仍然可用于向后兼容。

```py
method nullslast() → ColumnOperators
```

*继承自* `ColumnOperators.nullslast()` *方法的* `ColumnOperators`

对父对象生成一个`nulls_last()`子句。

从版本 1.4 开始更改：`nulls_last()` 操作符从之前的发布中的 `nullslast()` 改名。 以前的名称仍然可用于向后兼容。

```py
method op(opstring: str, precedence: int = 0, is_comparison: bool = False, return_type: Type[TypeEngine[Any]] | TypeEngine[Any] | None = None, python_impl: Callable[..., Any] | None = None) → Callable[[Any], Operators]
```

*继承自* `Operators.op()` *方法的* `Operators`

生成一个通用操作函数。

例如：

```py
somecolumn.op("*")(5)
```

产生：

```py
somecolumn * 5
```

这个函数也可以用来使位运算符明确。 例如：

```py
somecolumn.op('&')(0xff)
```

是 `somecolumn` 中的值的按位 AND。

参数：

+   `opstring` – 将作为这个元素与传递给生成函数的表达式之间的中缀运算符输出的字符串。

+   `precedence` –

    数据库预计在 SQL 表达式中应用到运算符的优先级。 这个整数值作为 SQL 编译器的提示，用于了解何时应在特定操作周围呈现显式括号。 较低的数字将导致将表达式与具有较高优先级的另一个运算符应用时加括号。 默认值为 `0`，低于所有运算符，除了逗号（`,`）和 `AS` 运算符。 值为 `100` 将高于或等于所有运算符，而 `-100` 将低于或等于所有运算符。

    另请参阅

    我正在使用 op() 生成自定义操作符，但我的括号没有正确输出 - SQLAlchemy SQL 编译器如何渲染括号的详细说明

+   `is_comparison` –

    旧式的；如果为 True，则将该操作符视为“比较”操作符，即评估为布尔真/假值，如 `==`、`>` 等。提供此标志是为了使 ORM 关系能够在自定义连接条件中使用操作符时确定操作符是比较操作符。

    使用 `is_comparison` 参数已被使用 `Operators.bool_op()` 方法取代；这个更简洁的操作符会自动设置该参数，但同时还提供了正确的[**PEP 484**](https://peps.python.org/pep-0484/)类型支持，因为返回的对象将表示“布尔”数据类型，即 `BinaryExpression[bool]`。

+   `return_type` – 一个`TypeEngine` 类或对象，它将强制此操作符生成的表达式的返回类型为该类型。缺省情况下，指定 `Operators.op.is_comparison` 的操作符将解析为 `Boolean`，而不指定的操作符将与左操作数具有相同的类型。

+   `python_impl` –

    可选的 Python 函数，可以在数据库服务器上运行时以相同的方式评估两个 Python 值的操作符。对于在 Python 中进行 SQL 表达式评估函数（例如用于 ORM 混合属性的函数）以及在多行更新或删除后用于匹配会话中的对象的 ORM “评估器”非常有用。

    例如：

    ```py
    >>> expr = column('x').op('+', python_impl=lambda a, b: a + b)('y')
    ```

    上述表达式的操作符也将适用于非 SQL 的左右对象：

    ```py
    >>> expr.operator(5, 10)
    15
    ```

    2.0 版中的新功能。

另请参阅

`Operators.bool_op()`

重新定义和创建新的运算符

在连接条件中使用自定义运算符

```py
method operate(op: OperatorType, *other: Any, **kwargs: Any) → ColumnElement[Any]
```

对参数进行操作。

这是操作的最低级别，缺省情况下会引发 `NotImplementedError`。

在子类上重写此功能可以使常见行为应用于所有操作。例如，重写 `ColumnOperators` 来应用 `func.lower()` 到左右两侧：

```py
class MyComparator(ColumnOperators):
    def operate(self, op, other, **kwargs):
        return op(func.lower(self), func.lower(other), **kwargs)
```

参数：

+   `op` – 操作符可调用。

+   `*other` – 操作的‘另一’方。对于大多数操作，它将是一个标量值。

+   `**kwargs` – 修饰符。这些可以由特殊运算符如 `ColumnOperators.contains()` 传递。

```py
method regexp_match(pattern: Any, flags: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.regexp_match()` *方法的* `ColumnOperators`

实现了特定于数据库的‘regexp match’运算符。

例如：

```py
stmt = select(table.c.some_column).where(
    table.c.some_column.regexp_match('^(b|c)')
)
```

`ColumnOperators.regexp_match()` 尝试解析为后端提供的类似 REGEXP 的函数或运算符，但特定的正则表达式语法和可用的标志 **不是后端无关的**。

示例包括：

+   PostgreSQL - 在否定时渲染为 `x ~ y` 或 `x !~ y`。

+   Oracle - 渲染为 `REGEXP_LIKE(x, y)`

+   SQLite - 使用 SQLite 的 `REGEXP` 占位符运算符，并调用 Python 的 `re.match()` 内置函数。

+   其他后端可能提供特殊实现。

+   没有任何特殊实现的后端将将运算符输出为 “REGEXP” 或 “NOT REGEXP”。例如，这与 SQLite 和 MySQL 兼容。

正则表达式支持目前已在 Oracle、PostgreSQL、MySQL 和 MariaDB 中实现。对于 SQLite，部分支持可用。第三方方言中的支持可能有所不同。

参数：

+   `pattern` – 正则表达式模式字符串或列子句。

+   `flags` – 要应用的任何正则表达式字符串标志，仅作为普通 Python 字符串传递。这些标志是特定于后端的。某些后端，如 PostgreSQL 和 MariaDB，可能会将标志作为模式的一部分来指定。在 PostgreSQL 中使用忽略大小写标志 ‘i’ 时，将使用忽略大小写的正则表达式匹配运算符 `~*` 或 `!~*`。

版本 1.4 中的新功能。

在版本 1.4.48 中更改为：2.0.18 请注意，由于实现错误，“flags” 参数先前接受�� SQL 表达式对象，例如列表达式，除了普通的 Python 字符串。这种实现与缓存不兼容，并已移除；“flags” 参数应仅传递字符串，因为这些标志将作为 SQL 表达式中的文字内联值呈现。

另请参见

`ColumnOperators.regexp_replace()`

```py
method regexp_replace(pattern: Any, replacement: Any, flags: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.regexp_replace()` *方法的* `ColumnOperators`

实现了特定于数据库的‘regexp replace’运算符。

例如：

```py
stmt = select(
    table.c.some_column.regexp_replace(
        'b(..)',
        'XY',
        flags='g'
    )
)
```

`ColumnOperators.regexp_replace()` 试图解析为后端提供的类似 REGEXP_REPLACE 的函数，通常会发出函数 `REGEXP_REPLACE()`。然而，特定的正则表达式语法和可用的标志 **不是后端通用的**。

目前正则表达式替换支持的后端包括 Oracle、PostgreSQL、MySQL 8 或更高版本以及 MariaDB。第三方方言中的支持可能有所不同。

参数：

+   `pattern` – 正则表达式模式字符串或列子句。

+   `pattern` – 替换字符串或列子句。

+   `flags` – 要应用的任何正则表达式字符串标志，仅作为普通 Python 字符串传递。这些标志是特定于后端的。某些后端，如 PostgreSQL 和 MariaDB，可能会将标志作为模式的一部分指定。

版本 1.4 中的新功能。

从版本 1.4.48 更改为：2.0.18 请注意，由于实现错误，以前的“flags”参数接受了 SQL 表达式对象，例如列表达式，而不仅仅是普通的 Python 字符串。这种实现与缓存不兼容，并已删除；应该仅传递字符串作为“flags”参数，因为这些标志将作为 SQL 表达式中的文字内联值呈现。

另请参阅

`ColumnOperators.regexp_match()`

```py
attribute remote_attr
```

*继承自* `AssociationProxyInstance.remote_attr` *属性的* `AssociationProxyInstance`

被此 `AssociationProxyInstance` 引用的“remote”类属性。

另请参阅

`AssociationProxyInstance.attr`

`AssociationProxyInstance.local_attr`

```py
method reverse_operate(op: OperatorType, other: Any, **kwargs: Any) → Operators
```

*继承自* `Operators.reverse_operate()` *方法的* `Operators`

对参数执行反向操作。

用法与 `operate()` 相同。

```py
attribute scalar
```

*继承自* `AssociationProxyInstance.scalar` *属性的* `AssociationProxyInstance`

如果此 `AssociationProxyInstance` 代理了本地端的标量关系，则返回`True`。

```py
method startswith(other: Any, escape: str | None = None, autoescape: bool = False) → ColumnOperators
```

*继承自* `ColumnOperators.startswith()` *方法的* `ColumnOperators`

实现`startswith`操作符。

生成一个 LIKE 表达式，用于测试字符串值的开头匹配：

```py
column LIKE <other> || '%'
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.startswith("foobar"))
```

由于操作符使用`LIKE`，在<other>表达式中存在的通配符字符`"%"`和`"_"`也将像通配符一样起作用。对于字面字符串值，可以将`ColumnOperators.startswith.autoescape`标志设置为`True`，以将这些字符在字符串值中的出现转义，使它们作为自身而不是通配符字符进行匹配。或者，`ColumnOperators.startswith.escape`参数将建立给定字符作为转义字符，这在目标表达式不是字面字符串时可能会有用。

参数：

+   `other` – 要进行比较的表达式。通常这是一个纯字符串值，但也可以是任意的 SQL 表达式。LIKE 通配符字符`%`和`_`默认情况下不会被转义，除非设置了`ColumnOperators.startswith.autoescape`标志为 True。

+   `autoescape` –

    boolean；当为 True 时，在 LIKE 表达式中建立转义字符，然后将其应用于比较值中的所有`"%"`、`"_"`和转义字符本身的出现，假定比较值是一个字面字符串而不是 SQL 表达式。

    诸如：

    ```py
    somecolumn.startswith("foo%bar", autoescape=True)
    ```

    将呈现为：

    ```py
    somecolumn LIKE :param || '%' ESCAPE '/'
    ```

    其值为`:param`，为`"foo/%bar"`。

+   `escape` –

    一个字符，当给定时将使用`ESCAPE`关键字来将该字符设定为转义字符。然后可以将此字符放置在`%`和`_`的前面，以允许它们作为自身而不是通配符字符。

    诸如：

    ```py
    somecolumn.startswith("foo/%bar", escape="^")
    ```

    将呈现为：

    ```py
    somecolumn LIKE :param || '%' ESCAPE '^'
    ```

    该参数也可以与`ColumnOperators.startswith.autoescape`结合使用：

    ```py
    somecolumn.startswith("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上述情况下，给定的文字参数将在传递到数据库之前转换为`"foo^%bar^^bat"`。

另请参见

`ColumnOperators.endswith()`

`ColumnOperators.contains()`

`ColumnOperators.like()`

```py
attribute target_class: Type[Any]
```

由此`AssociationProxyInstance`处理的中间类。

拦截附加/设置/赋值事件将导致生成此类的新实例。

```py
attribute timetuple: Literal[None] = None
```

*继承自* `ColumnOperators.timetuple` *属性的* `ColumnOperators` 类

Hack，允许在左侧比较日期时间对象。

```py
class sqlalchemy.ext.associationproxy.AssociationProxyExtensionType
```

一个枚举。

**成员**

ASSOCIATION_PROXY

**类签名**

类`sqlalchemy.ext.associationproxy.AssociationProxyExtensionType` (`sqlalchemy.orm.base.InspectionAttrExtensionType`)

```py
attribute ASSOCIATION_PROXY = 'ASSOCIATION_PROXY'
```

表示一个`InspectionAttr`的符号，其类型为`AssociationProxy`。

被分配给`InspectionAttr.extension_type`属性。
