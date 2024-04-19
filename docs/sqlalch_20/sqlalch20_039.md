# 使用遗留的 ‘backref’ 关系参数

> 原文：[`docs.sqlalchemy.org/en/20/orm/backref.html`](https://docs.sqlalchemy.org/en/20/orm/backref.html)

注意

应考虑使用遗留的 `relationship.backref` 关键字，并且应优先使用明确的 `relationship()` 构造与 `relationship.back_populates`。使用单独的 `relationship()` 构造提供了诸如 ORM 映射类都将其属性提前包括在类构造时等优点，而不是作为延迟步骤，并且配置更为直观，因为所有参数都是明确的。SQLAlchemy 2.0 中的新 [**PEP 484**](https://peps.python.org/pep-0484/) 特性还利用了属性在源代码中明确存在而不是使用动态属性生成。

请参见

有关双向关系的一般信息，请参阅以下部分：

与 ORM 相关对象一起工作 - 在 SQLAlchemy 统一教程 中，介绍了使用 `relationship.back_populates` 进行双向关联配置和行为的概览。

双向关系中保存更新级联的行为 - 关于双向 `relationship()` 行为在 `Session` 级联行为方面的注意事项。

`relationship.back_populates`

在`relationship()`构造函数中的`relationship.backref`关键字参数允许自动生成一个新的`relationship()`，该关系将自动添加到相关类的 ORM 映射中。然后，它将被放置到当前正在配置的`relationship()`的`relationship.back_populates`配置中，其中两个`relationship()`构造相互引用。

以以下示例开始：

```py
from sqlalchemy import Column, ForeignKey, Integer, String
from sqlalchemy.orm import DeclarativeBase, relationship

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "user"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String)

    addresses = relationship("Address", backref="user")

class Address(Base):
    __tablename__ = "address"
    id = mapped_column(Integer, primary_key=True)
    email = mapped_column(String)
    user_id = mapped_column(Integer, ForeignKey("user.id"))
```

以上配置在`User`上建立了一个名为`User.addresses`的`Address`对象集合。它还在`Address`上建立了一个`.user`属性，该属性将指向父`User`对象。使用`relationship.back_populates`等效于以下操作：

```py
from sqlalchemy import Column, ForeignKey, Integer, String
from sqlalchemy.orm import DeclarativeBase, relationship

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "user"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String)

    addresses = relationship("Address", back_populates="user")

class Address(Base):
    __tablename__ = "address"
    id = mapped_column(Integer, primary_key=True)
    email = mapped_column(String)
    user_id = mapped_column(Integer, ForeignKey("user.id"))

    user = relationship("User", back_populates="addresses")
```

`User.addresses`和`Address.user`关系的行为是以**双向**方式进行的，表示关系的一侧发生变化会影响另一侧。有关此行为的示例和讨论，请参阅 SQLAlchemy 统一教程的使用 ORM 相关对象部分。

## 反向引用默认参数

由于`relationship.backref`会生成一个全新的`relationship()`，默认情况下，生成过程将尝试在新的`relationship()`中包含对应于原始参数的相应参数。举例说明，下面是一个包含自定义连接条件的`relationship()`，该条件还包括`relationship.backref`关键字：

```py
from sqlalchemy import Column, ForeignKey, Integer, String
from sqlalchemy.orm import DeclarativeBase, relationship

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "user"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String)

    addresses = relationship(
        "Address",
        primaryjoin=(
            "and_(User.id==Address.user_id, Address.email.startswith('tony'))"
        ),
        backref="user",
    )

class Address(Base):
    __tablename__ = "address"
    id = mapped_column(Integer, primary_key=True)
    email = mapped_column(String)
    user_id = mapped_column(Integer, ForeignKey("user.id"))
```

当生成“backref”时，`relationship.primaryjoin`条件也被复制到新的`relationship()`中：

```py
>>> print(User.addresses.property.primaryjoin)
"user".id = address.user_id AND address.email LIKE :email_1 || '%%'
>>>
>>> print(Address.user.property.primaryjoin)
"user".id = address.user_id AND address.email LIKE :email_1 || '%%'
>>>
```

其他可转移的参数包括`relationship.secondary`参数，它指的是多对多关联表，以及“join”参数`relationship.primaryjoin`和`relationship.secondaryjoin`；“backref”足够智能，知道在生成相反的一侧时这两个参数也应该被“翻转”。

## 指定反向引用参数

很多其他用于“backref”的参数都不是隐含的，包括像`relationship.lazy`、`relationship.remote_side`、`relationship.cascade`和`relationship.cascade_backrefs`等参数。对于这种情况，我们使用`backref()`函数代替字符串；这将存储一组特定的参数，这些参数将在生成新的`relationship()`时传递： 

```py
# <other imports>
from sqlalchemy.orm import backref

class User(Base):
    __tablename__ = "user"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String)

    addresses = relationship(
        "Address",
        backref=backref("user", lazy="joined"),
    )
```

在上面的例子中，我们只在`Address.user`一侧放置了`lazy="joined"`指令，这表示当对`Address`进行查询时，应自动执行与`User`实体的连接，这将填充每个返回的`Address`的`.user`属性。 `backref()`函数将我们给定的参数格式化成一个由接收的`relationship()`解释的形式，作为应用于它创建的新关系的附加参数。

## 反向引用默认参数

由于`relationship.backref`生成了一个全新的`relationship()`，默认情况下生成过程将尝试在新的`relationship()`中包含与原始参数相对应的参数。例如，下面是一个包含自定义连接条件的`relationship()`示例，该连接条件还包括`relationship.backref`关键字：

```py
from sqlalchemy import Column, ForeignKey, Integer, String
from sqlalchemy.orm import DeclarativeBase, relationship

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "user"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String)

    addresses = relationship(
        "Address",
        primaryjoin=(
            "and_(User.id==Address.user_id, Address.email.startswith('tony'))"
        ),
        backref="user",
    )

class Address(Base):
    __tablename__ = "address"
    id = mapped_column(Integer, primary_key=True)
    email = mapped_column(String)
    user_id = mapped_column(Integer, ForeignKey("user.id"))
```

当生成“反向引用”时，`relationship.primaryjoin`条件也会被复制到新的`relationship()`中：

```py
>>> print(User.addresses.property.primaryjoin)
"user".id = address.user_id AND address.email LIKE :email_1 || '%%'
>>>
>>> print(Address.user.property.primaryjoin)
"user".id = address.user_id AND address.email LIKE :email_1 || '%%'
>>>
```

可传递的其他参数包括指向多对多关联表的`relationship.secondary`参数，以及“join”参数`relationship.primaryjoin`和`relationship.secondaryjoin`；“反向引用”足够智能，可以知道在生成相反方向时这两个参数也应该“反转”。

## 指定反向引用参数

“反向引用”的许多其他参数都不是隐式的，包括像`relationship.lazy`、`relationship.remote_side`、`relationship.cascade`和`relationship.cascade_backrefs`等参数。对于这种情况，我们使用`backref()`函数来代替字符串；这将存储一组特定的参数，这些参数在生成新的`relationship()`时将被传递：

```py
# <other imports>
from sqlalchemy.orm import backref

class User(Base):
    __tablename__ = "user"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String)

    addresses = relationship(
        "Address",
        backref=backref("user", lazy="joined"),
    )
```

在上面的例子中，我们只在`Address.user`一侧放置了`lazy="joined"`指令，这意味着当对`Address`进行查询时，应自动执行与`User`实体的连接，从而填充每个返回的`Address`的`.user`属性。`backref()`函数将我们给定的参数格式化成一种被接收`relationship()`解释为要应用于它创建的新关系的附加参数的形式。
