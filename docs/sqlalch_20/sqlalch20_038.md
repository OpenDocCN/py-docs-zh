# 特殊的关系持久性模式

> 原文：[`docs.sqlalchemy.org/en/20/orm/relationship_persistence.html`](https://docs.sqlalchemy.org/en/20/orm/relationship_persistence.html)

## 指向自身的行 / 相互依赖的行

这是一种非常特殊的情况，其中 relationship()必须执行一个 INSERT 和一个第二个 UPDATE，以正确填充一行（反之亦然，为了删除而执行一个 UPDATE 和 DELETE，而不违反外键约束）。这两种用例是：

+   一个表包含对自身的外键，而且单个行将具有指向其自身主键的外键值。

+   两个表都包含对另一个表的外键引用，每个表中的一行引用另一个表中的另一行。

例如：

```py
 user
---------------------------------
user_id    name   related_user_id
   1       'ed'          1
```

或：

```py
 widget                                                  entry
-------------------------------------------             ---------------------------------
widget_id     name        favorite_entry_id             entry_id      name      widget_id
   1       'somewidget'          5                         5       'someentry'     1
```

在第一种情况下，一行指向自身。从技术上讲，使用诸如 PostgreSQL 或 Oracle 之类的序列的数据库可以使用先前生成的值一次性插入行，但是依赖于自增样式主键标识符的数据库不能。`relationship()`始终假定在刷新期间以“父/子”模型进行行填充，因此除非直接填充主键/外键列，否则`relationship()`需要使用两个语句。

在第二种情况下，“widget”行必须在引用的“entry”行之前插入，但是那个“widget”行的“favorite_entry_id”列在生成“entry”行之前无法设置。在这种情况下，通常无法仅使用两个 INSERT 语句插入“widget”和“entry”行；必须执行 UPDATE 以保持外键约束满足。例外情况是如果外键配置为“延迟至提交”（一些数据库支持的功能），并且标识符是手动填充的（再次基本上绕过`relationship()`）。

要启用补充 UPDATE 语句的使用，我们使用`relationship.post_update`选项的`relationship()`。这指定了在两个行都被 INSERTED 之后应使用 UPDATE 语句创建两行之间的关联；它还导致在发出 DELETE 之前通过 UPDATE 将行解除关联。该标志应该放置在*一个*关系上，最好是一对多的一侧。以下我们举例说明了一个完整的例子，包括两个`ForeignKey`构造：

```py
from sqlalchemy import Integer, ForeignKey
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

class Entry(Base):
    __tablename__ = "entry"
    entry_id = mapped_column(Integer, primary_key=True)
    widget_id = mapped_column(Integer, ForeignKey("widget.widget_id"))
    name = mapped_column(String(50))

class Widget(Base):
    __tablename__ = "widget"

    widget_id = mapped_column(Integer, primary_key=True)
    favorite_entry_id = mapped_column(
        Integer, ForeignKey("entry.entry_id", name="fk_favorite_entry")
    )
    name = mapped_column(String(50))

    entries = relationship(Entry, primaryjoin=widget_id == Entry.widget_id)
    favorite_entry = relationship(
        Entry, primaryjoin=favorite_entry_id == Entry.entry_id, post_update=True
    )
```

当针对上述配置的结构被刷新时，“widget”行将会插入，但不包括“favorite_entry_id”值，然后所有的“entry”行将被插入，引用父“widget”行，然后一个 UPDATE 语句将填充“widget”表的“favorite_entry_id”列（目前每次一行）：

```py
>>> w1 = Widget(name="somewidget")
>>> e1 = Entry(name="someentry")
>>> w1.favorite_entry = e1
>>> w1.entries = [e1]
>>> session.add_all([w1, e1])
>>> session.commit()
BEGIN  (implicit)
INSERT  INTO  widget  (favorite_entry_id,  name)  VALUES  (?,  ?)
(None,  'somewidget')
INSERT  INTO  entry  (widget_id,  name)  VALUES  (?,  ?)
(1,  'someentry')
UPDATE  widget  SET  favorite_entry_id=?  WHERE  widget.widget_id  =  ?
(1,  1)
COMMIT 
```

我们可以指定的另一个配置是在`Widget`上提供一个更全面的外键约束，以确保`favorite_entry_id`引用的是也引用此`Widget`的`Entry`。我们可以使用复合外键，如下所示：

```py
from sqlalchemy import (
    Integer,
    ForeignKey,
    String,
    UniqueConstraint,
    ForeignKeyConstraint,
)
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

class Entry(Base):
    __tablename__ = "entry"
    entry_id = mapped_column(Integer, primary_key=True)
    widget_id = mapped_column(Integer, ForeignKey("widget.widget_id"))
    name = mapped_column(String(50))
    __table_args__ = (UniqueConstraint("entry_id", "widget_id"),)

class Widget(Base):
    __tablename__ = "widget"

    widget_id = mapped_column(Integer, autoincrement="ignore_fk", primary_key=True)
    favorite_entry_id = mapped_column(Integer)

    name = mapped_column(String(50))

    __table_args__ = (
        ForeignKeyConstraint(
            ["widget_id", "favorite_entry_id"],
            ["entry.widget_id", "entry.entry_id"],
            name="fk_favorite_entry",
        ),
    )

    entries = relationship(
        Entry, primaryjoin=widget_id == Entry.widget_id, foreign_keys=Entry.widget_id
    )
    favorite_entry = relationship(
        Entry,
        primaryjoin=favorite_entry_id == Entry.entry_id,
        foreign_keys=favorite_entry_id,
        post_update=True,
    )
```

上面的映射具有一个复合`ForeignKeyConstraint`，连接`widget_id`和`favorite_entry_id`列。为了确保`Widget.widget_id`保持为“自动增量”列，我们在`Column`上的`autoincrement`参数上指定值`"ignore_fk"`，并且在每个`relationship()`上，我们必须限制那些被视为外键的列，以用于连接和交叉填充。##可变主键/更新级联

当实体的主键更改时，引用主键的相关项也必须更新。对于强制实施引用完整性的数据库，最佳策略是使用数据库的`ON UPDATE CASCADE`功能，以便将主键更改传播到引用的外键 - 在事务完成之前，值不能不同步，除非约束标记为“可延迟”。

强烈建议一个希望使用可变值的自然主键的应用程序使用数据库的`ON UPDATE CASCADE`功能。一个说明此功能的示例映射是：

```py
class User(Base):
    __tablename__ = "user"
    __table_args__ = {"mysql_engine": "InnoDB"}

    username = mapped_column(String(50), primary_key=True)
    fullname = mapped_column(String(100))

    addresses = relationship("Address")

class Address(Base):
    __tablename__ = "address"
    __table_args__ = {"mysql_engine": "InnoDB"}

    email = mapped_column(String(50), primary_key=True)
    username = mapped_column(
        String(50), ForeignKey("user.username", onupdate="cascade")
    )
```

在上面的示例中，我们在`ForeignKey`对象上说明了`onupdate="cascade"`，并且我们还说明了`mysql_engine='InnoDB'`设置，在 MySQL 后端上，确保使用支持引用完整性的`InnoDB`引擎。在使用 SQLite 时，应启用引用完整性，使用 Foreign Key Support 中描述的配置。

也请参阅

使用 ORM 关系的外键 ON DELETE 级联 - 支持使用关系的 ON DELETE CASCADE

`mapper.passive_updates` - 类似`Mapper`上的功能

### 模拟有限的 ON UPDATE CASCADE，没有外键支持

在使用不支持引用完整性的数据库，并且使用具有可变值的自然主键时，SQLAlchemy 提供了一个功能，允许将主键值传播到已引用的外键到**有限**程度，通过针对立即引用主键列的外键列发出 UPDATE 语句，其值已更改。没有引用完整性功能的主要平台是当使用 `MyISAM` 存储引擎时的 MySQL，以及当没有使用 `PRAGMA foreign_keys=ON` 时的 SQLite。Oracle 数据库也不支持 `ON UPDATE CASCADE`，但因为它仍然强制执行引用完整性，需要将约束标记为可延迟，以便 SQLAlchemy 可以发出 UPDATE 语句。

通过将 `relationship.passive_updates` 标志设置为 `False`，最好是在一对多或多对多的 `relationship()` 上。当“更新”不再是“被动”的时候，这表明 SQLAlchemy 将为父对象引用的集合中的对象单独发出 UPDATE 语句，这些对象的主键值会发生变化。这还意味着如果集合尚未在本地存在，集合将被完全加载到内存中。

我们之前使用 `passive_updates=False` 的映射如下：

```py
class User(Base):
    __tablename__ = "user"

    username = mapped_column(String(50), primary_key=True)
    fullname = mapped_column(String(100))

    # passive_updates=False *only* needed if the database
    # does not implement ON UPDATE CASCADE
    addresses = relationship("Address", passive_updates=False)

class Address(Base):
    __tablename__ = "address"

    email = mapped_column(String(50), primary_key=True)
    username = mapped_column(String(50), ForeignKey("user.username"))
```

`passive_updates=False` 的关键限制包括：

+   它的性能远远不及直接数据库的 ON UPDATE CASCADE，因为它需要使用 SELECT 完全预加载受影响的集合，并且还必须发出针对这些值的 UPDATE 语句，它将尝试以“批量”的方式运行，但仍然在 DBAPI 级别上按行运行。

+   该功能无法“级联”超过一个级别。也就是说，如果映射 X 有一个外键引用映射 Y 的主键，但是然后映射 Y 的主键本身是映射 Z 的外键，`passive_updates=False` 无法将主键值从 `Z` 级联到 `X`。

+   仅在关系的多对一方配置 `passive_updates=False` 将不会产生完全的效果，因为工作单元仅在当前身份映射中搜索可能引用具有变异主键的对象，而不是在整个数据库中搜索。

由于现在除了 Oracle 外，几乎所有数据库都支持 `ON UPDATE CASCADE`，因此强烈建议在使用自然且可变的主键值时使用传统的 `ON UPDATE CASCADE` 支持。## 指向自身的行 / 相互依赖的行

这是一个非常特殊的情况，其中关系（`relationship()`）必须执行 INSERT 和第二个 UPDATE，以便正确填充一行（反之亦然，执行 UPDATE 和 DELETE 以删除而不违反外键约束）。这两种用例是：

+   一张表包含一个指向自身的外键，而且一行将具有指向自己主键的外键值。

+   两个表分别包含一个外键引用另一个表，每个表中的一行引用另一个表。

例如：

```py
 user
---------------------------------
user_id    name   related_user_id
   1       'ed'          1
```

或者：

```py
 widget                                                  entry
-------------------------------------------             ---------------------------------
widget_id     name        favorite_entry_id             entry_id      name      widget_id
   1       'somewidget'          5                         5       'someentry'     1
```

在第一种情况下，一行指向自身。从技术上讲，使用序列（如 PostgreSQL 或 Oracle）的数据库可以使用先前生成的值一次性插入行，但依赖自动增量样式主键标识符的数据库则不能。`relationship()`始终假定在刷新期间使用“父/子”模型来填充行，因此除非直接填充主键/外键列，否则 `relationship()` 需要使用两个语句。

在第二种情况下，“widget”行必须在任何引用的“entry”行之前插入，但然后该“widget”行的“favorite_entry_id”列在生成“entry”行之前不能设置。在这种情况下，通常不可能只使用两个 INSERT 语句插入“widget”和“entry”行；必须执行 UPDATE 以保持外键约束得到满足。异常情况是，如果外键配置为“延迟到提交”（某些数据库支持的功能），并且标识符是手动填充的（再次基本上绕过`relationship()`）。

为了启用补充的 UPDATE 语句的使用，我们使用`relationship()`的`relationship.post_update`选项。这指定在两行都被插入后使用 UPDATE 语句创建两行之间的连接；它还导致在发出 DELETE 之前，通过 UPDATE 将行彼此解除关联。这个标志应该放在其中*一个*关系上，最好是多对一的关系。下面我们举个完整的例子，包括两个`ForeignKey`构造：

```py
from sqlalchemy import Integer, ForeignKey
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

class Entry(Base):
    __tablename__ = "entry"
    entry_id = mapped_column(Integer, primary_key=True)
    widget_id = mapped_column(Integer, ForeignKey("widget.widget_id"))
    name = mapped_column(String(50))

class Widget(Base):
    __tablename__ = "widget"

    widget_id = mapped_column(Integer, primary_key=True)
    favorite_entry_id = mapped_column(
        Integer, ForeignKey("entry.entry_id", name="fk_favorite_entry")
    )
    name = mapped_column(String(50))

    entries = relationship(Entry, primaryjoin=widget_id == Entry.widget_id)
    favorite_entry = relationship(
        Entry, primaryjoin=favorite_entry_id == Entry.entry_id, post_update=True
    )
```

当针对上述配置刷新结构时，将插入“widget”行，但不包括“favorite_entry_id”值，然后将插入所有“entry”行，引用父“widget”行，然后将“widget”表的“favorite_entry_id”列的 UPDATE 语句（目前一次一行）填充：

```py
>>> w1 = Widget(name="somewidget")
>>> e1 = Entry(name="someentry")
>>> w1.favorite_entry = e1
>>> w1.entries = [e1]
>>> session.add_all([w1, e1])
>>> session.commit()
BEGIN  (implicit)
INSERT  INTO  widget  (favorite_entry_id,  name)  VALUES  (?,  ?)
(None,  'somewidget')
INSERT  INTO  entry  (widget_id,  name)  VALUES  (?,  ?)
(1,  'someentry')
UPDATE  widget  SET  favorite_entry_id=?  WHERE  widget.widget_id  =  ?
(1,  1)
COMMIT 
```

我们可以指定的另一个配置是在 `Widget` 上提供更全面的外键约束，以确保 `favorite_entry_id` 指向也指向此 `Widget` 的 `Entry`。我们可以使用复合外键，如下所示：

```py
from sqlalchemy import (
    Integer,
    ForeignKey,
    String,
    UniqueConstraint,
    ForeignKeyConstraint,
)
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

class Entry(Base):
    __tablename__ = "entry"
    entry_id = mapped_column(Integer, primary_key=True)
    widget_id = mapped_column(Integer, ForeignKey("widget.widget_id"))
    name = mapped_column(String(50))
    __table_args__ = (UniqueConstraint("entry_id", "widget_id"),)

class Widget(Base):
    __tablename__ = "widget"

    widget_id = mapped_column(Integer, autoincrement="ignore_fk", primary_key=True)
    favorite_entry_id = mapped_column(Integer)

    name = mapped_column(String(50))

    __table_args__ = (
        ForeignKeyConstraint(
            ["widget_id", "favorite_entry_id"],
            ["entry.widget_id", "entry.entry_id"],
            name="fk_favorite_entry",
        ),
    )

    entries = relationship(
        Entry, primaryjoin=widget_id == Entry.widget_id, foreign_keys=Entry.widget_id
    )
    favorite_entry = relationship(
        Entry,
        primaryjoin=favorite_entry_id == Entry.entry_id,
        foreign_keys=favorite_entry_id,
        post_update=True,
    )
```

上述映射展示了一个由`ForeignKeyConstraint`组成的复合键，连接着 `widget_id` 和 `favorite_entry_id` 列。为了确保 `Widget.widget_id` 仍然是一个“自增”的列，我们在`Column`上指定了 `Column.autoincrement` 的值为 `"ignore_fk"`，并且在每个`relationship()`上，我们必须限制那些被视为外键的列以进行连接和交叉填充。

## 可变主键 / 更新级联

当实体的主键发生变化时，引用该主键的相关项也必须进行更新。对于强制执行引用完整性的数据库，最佳策略是使用数据库的 ON UPDATE CASCADE 功能，以便将主键更改传播到引用的外键 - 除非约束被标记为“可延迟”，即不执行直到事务完成，否则值不能在任何时刻不同步。

**强烈建议**希望使用可变值的自然主键的应用程序使用数据库的 `ON UPDATE CASCADE` 功能。一个示例映射如下：

```py
class User(Base):
    __tablename__ = "user"
    __table_args__ = {"mysql_engine": "InnoDB"}

    username = mapped_column(String(50), primary_key=True)
    fullname = mapped_column(String(100))

    addresses = relationship("Address")

class Address(Base):
    __tablename__ = "address"
    __table_args__ = {"mysql_engine": "InnoDB"}

    email = mapped_column(String(50), primary_key=True)
    username = mapped_column(
        String(50), ForeignKey("user.username", onupdate="cascade")
    )
```

在上文中，我们在 `ForeignKey` 对象上说明了 `onupdate="cascade"`，并且我们还说明了 `mysql_engine='InnoDB'` 设置，该设置在 MySQL 后端上确保使用支持引用完整性的 `InnoDB` 引擎。在使用 SQLite 时，应启用引用完整性，使用 外键支持 中描述的配置。

请参阅

使用 ORM 关系的外键 ON DELETE 级联 - 支持使用关系的 ON DELETE CASCADE

`mapper.passive_updates` - `Mapper` 上的类似功能

### 模拟有限的无外键支持的 ON UPDATE CASCADE

当使用不支持引用完整性的数据库，并且存在具有可变值的自然主键时，SQLAlchemy 提供了一项功能，以允许在**有限**范围内传播主键值到已引用的外键，方法是针对立即引用其值已更改的主键列发出 UPDATE 语句来更新外键列。不支持引用完整性功能的主要平台是在使用`MyISAM`存储引擎时的 MySQL，以及在未使用`PRAGMA foreign_keys=ON`指示的情况下的 SQLite。Oracle 数据库也不支持`ON UPDATE CASCADE`，但因为它仍然强制执行引用完整性，所以需要将约束标记为可延迟，以便 SQLAlchemy 可以发出 UPDATE 语句。

通过将`relationship.passive_updates`标志设置为`False`来启用此功能，最好是在一对多或多对多的`relationship()`上。当“更新”不再“被动”时，这表示 SQLAlchemy 将为引用具有更改的主键值的父对象的集合中的对象单独发出 UPDATE 语句。这也意味着如果集合尚未在本地存在，那么集合将完全加载到内存中。

我们以前使用`passive_updates=False`的映射如下：

```py
class User(Base):
    __tablename__ = "user"

    username = mapped_column(String(50), primary_key=True)
    fullname = mapped_column(String(100))

    # passive_updates=False *only* needed if the database
    # does not implement ON UPDATE CASCADE
    addresses = relationship("Address", passive_updates=False)

class Address(Base):
    __tablename__ = "address"

    email = mapped_column(String(50), primary_key=True)
    username = mapped_column(String(50), ForeignKey("user.username"))
```

`passive_updates=False`的关键限制包括：

+   它的性能比直接的数据库 ON UPDATE CASCADE 要差得多，因为它需要使用 SELECT 完全预加载受影响的集合，并且还必须针对这些值发出 UPDATE 语句，尽管它将尝试以“批处理”的方式运行，但仍然是在 DBAPI 级别上逐行运行。

+   此功能不能“级联”超过一级。也就是说，如果映射 X 具有一个外键，它引用映射 Y 的主键，但然后映射 Y 的主键本身是对映射 Z 的外键，则`passive_updates=False`不能将主键值从`Z`级联更改到`X`。

+   仅在关系的多对一一侧上配置`passive_updates=False`将不会产生完全效果，因为工作单元仅通过当前身份映射搜索可能引用具有变异主键的对象，而不是在整个数据库中搜索。

由于几乎所有的数据库现在都支持`ON UPDATE CASCADE`，因此强烈建议在使用自然且可变的主键值时使用传统的`ON UPDATE CASCADE`支持。

### 模拟无外键支持的有限 ON UPDATE CASCADE

在使用不支持引用完整性的数据库且存在可变值的自然主键的情况下，SQLAlchemy 提供了一种功能，允许在已经引用了外键的情况下将主键值传播到一个**有限**程度，通过针对立即引用已更改主键列值的主键列的 UPDATE 语句进行发射。主要没有引用完整性功能的平台是在使用 `MyISAM` 存储引擎时的 MySQL，以及在不使用 `PRAGMA foreign_keys=ON` pragma 的情况下的 SQLite。Oracle 数据库也不支持 `ON UPDATE CASCADE`，但由于它仍然强制引用完整性，需要将约束标记为可延迟，以便 SQLAlchemy 可以发出 UPDATE 语句。

通过将 `relationship.passive_updates` 标志设置为 `False` 来启用此功能，最好在一对多或多对多的 `relationship()` 上设置。当“更新”不再是“被动”的时候，这表明 SQLAlchemy 将针对父对象引用的集合中的对象单独发出 UPDATE 语句，而这些对象具有正在更改的主键值。这也意味着，如果集合尚未在本地存在，集合将被完全加载到内存中。

我们之前使用 `passive_updates=False` 的映射如下所示：

```py
class User(Base):
    __tablename__ = "user"

    username = mapped_column(String(50), primary_key=True)
    fullname = mapped_column(String(100))

    # passive_updates=False *only* needed if the database
    # does not implement ON UPDATE CASCADE
    addresses = relationship("Address", passive_updates=False)

class Address(Base):
    __tablename__ = "address"

    email = mapped_column(String(50), primary_key=True)
    username = mapped_column(String(50), ForeignKey("user.username"))
```

`passive_updates=False` 的关键限制包括：

+   它的性能远远不如直接的数据库 ON UPDATE CASCADE，因为它需要使用 SELECT 完全预加载受影响的集合，并且还必须发出针对这些值的 UPDATE 语句，尽管它会尝试以“批量”的方式运行，但仍然在 DBAPI 级别逐行运行。

+   该功能无法“级联”超过一级。也就是说，如果映射 X 有一个外键引用到映射 Y 的主键，但映射 Y 的主键本身是映射 Z 的外键，`passive_updates=False` 无法将来自 `Z` 到 `X` 的主键值更改级联。

+   仅在关系的多对一侧配置 `passive_updates=False` 不会产生完全效果，因为工作单元仅在当前标识映射中搜索可能引用具有突变主键的对象，而不是在整个数据库中搜索。

由于除 Oracle 外的几乎所有数据库现在都支持 `ON UPDATE CASCADE`，因此强烈建议在使用自然和可变主键值的情况下使用传统的 `ON UPDATE CASCADE` 支持。
