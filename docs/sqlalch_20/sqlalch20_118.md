# ORM 配置

> 原文：[`docs.sqlalchemy.org/en/20/faq/ormconfiguration.html`](https://docs.sqlalchemy.org/en/20/faq/ormconfiguration.html)

+   如何映射没有主键的表？

+   如何配置一个与 Python 保留字或类似的列？

+   如何在给定映射类的情况下获取所有列、关系、映射属性等的列表？

+   我收到关于“在属性 Y 下隐式组合列 X”的警告或错误

+   我正在使用声明式并使用 `and_()` 或 `or_()` 设置 primaryjoin/secondaryjoin，但我收到了关于外键的错误消息。

+   为什么推荐在 `LIMIT` 中使用 `ORDER BY`（特别是在 `subqueryload()` 中）？

## 如何映射没有主键的表？

为了映射到特定表，SQLAlchemy ORM 需要至少有一个列被标记为主键列；当然，多列，即复合主键，也是完全可行的。这些列不需要实际被数据库知道为主键列，尽管最好是这样。只需要这些列 *行为* 象主键一样，例如，作为行的唯一且非空的标识符。

大多数 ORM 都要求对象有某种形式的主键定义，因为内存中的对象必须对应于数据库表中的唯一可识别行；至少，这允许对象可以被定位用于仅影响该对象行而不影响其他行的 UPDATE 和 DELETE 语句。然而，主键的重要性远不止于此。在 SQLAlchemy 中，所有 ORM 映射的对象始终使用称为 身份映射 的模式与它们的特定数据库行唯一链接在一起，这是 SQLAlchemy 使用的工作单元系统的核心模式，也是最常见的（和不那么常见的） ORM 使用模式的关键。

注意

需要注意的是，我们只讨论 SQLAlchemy ORM；一个基于 Core 构建并且只处理 `Table` 对象、`select()` 构造等的应用程序 **不需要** 在任何方式上存在或关联表上有任何主键（尽管再次强调，在 SQL 中，所有表应该真的有某种主键，以免您实际上需要更新或删除特定行）。

在几乎所有情况下，表确实有所谓的 候选键，它是一列或一系列列，可以唯一标识一行。如果一张表真的没有这个，而且有实际完全重复的行，那么该表就不符合 [第一范式](https://en.wikipedia.org/wiki/First_normal_form)，也不能被映射。否则，组成最佳候选键的任何列都可以直接应用到映射器上：

```py
class SomeClass(Base):
    __table__ = some_table_with_no_pk
    __mapper_args__ = {
        "primary_key": [some_table_with_no_pk.c.uid, some_table_with_no_pk.c.bar]
    }
```

当使用完全声明的表元数据时，最好在这些列上使用 `primary_key=True` 标志：

```py
class SomeClass(Base):
    __tablename__ = "some_table_with_no_pk"

    uid = Column(Integer, primary_key=True)
    bar = Column(String, primary_key=True)
```

关系数据库中的所有表都应该有主键。即使是多对多关联表 - 主键将是两个关联列的组合：

```py
CREATE  TABLE  my_association  (
  user_id  INTEGER  REFERENCES  user(id),
  account_id  INTEGER  REFERENCES  account(id),
  PRIMARY  KEY  (user_id,  account_id)
)
```

## 如何配置一个 Python 保留字或类似的 Column？

基于列的属性可以在映射中被赋予任何所需的名称。请参阅明确命名声明式映射的列。

## 如何在给定一个映射类的情况下获取所有列、关系、映射属性等列表？

所有这些信息都可以从 `Mapper` 对象中获得。

要获取特定映射类的 `Mapper`，请对其调用 `inspect()` 函数：

```py
from sqlalchemy import inspect

mapper = inspect(MyClass)
```

从那里，关于类的所有信息都可以通过属性访问，例如：

+   `Mapper.attrs` - 所有映射属性的命名空间。属性本身是 `MapperProperty` 的实例，如果适用的话，它们包含可以导致映射的 SQL 表达式或列的其他属性。

+   `Mapper.column_attrs` - 限于列和 SQL 表达式属性的映射属性命名空间。您可能想直接使用 `Mapper.columns` 来获取 `Column` 对象。

+   `Mapper.relationships` - 所有`RelationshipProperty`属性的命名空间。

+   `Mapper.all_orm_descriptors` - 所有映射属性的命名空间，以及使用`hybrid_property`、`AssociationProxy`等系统定义的用户定义属性。

+   `Mapper.columns` - `Column`对象和与映射相关的其他命名 SQL 表达式的命名空间。

+   `Mapper.mapped_table` - 该映射器所映射到的`Table`或其他可选择的对象。

+   `Mapper.local_table` - 此映射器“本地”的`Table`；这在将映射器映射到组合可选择项的情况下与`Mapper.mapped_table`不同。

## 我收到关于“隐式将列 X 组合到属性 Y 下”的警告或错误

此条件指的是当映射包含两列，这两列由于名称而被映射到同一属性名下，但没有表明这是有意的。映射的类需要为每个要存储独立值的属性明确指定名称；当两列具有相同的名称并且没有消歧时，它们就属于同一属性，其效果是将一列的值**复制**到另一列，根据哪一列首先分配给属性。

这种行为通常是可取的，在继承映射中通过外键关系将两列链接在一起时是允许的，而不会发出警告。当出现警告或异常时，可以通过将列分配给名称不同的属性来解决问题，或者如果希望将它们组合在一起，则使用`column_property()`使其明确。

给出如下示例：

```py
from sqlalchemy import Integer, Column, ForeignKey
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()

class A(Base):
    __tablename__ = "a"

    id = Column(Integer, primary_key=True)

class B(A):
    __tablename__ = "b"

    id = Column(Integer, primary_key=True)
    a_id = Column(Integer, ForeignKey("a.id"))
```

自 SQLAlchemy 版本 0.9.5 起，将检测到上述条件，并将警告说`A`和`B`的`id`列正在组合为同名属性`id`，这是一个严重的问题，因为这意味着`B`对象的主键将始终与其`A`的主键相同。

解决此问题的映射如下：

```py
class A(Base):
    __tablename__ = "a"

    id = Column(Integer, primary_key=True)

class B(A):
    __tablename__ = "b"

    b_id = Column("id", Integer, primary_key=True)
    a_id = Column(Integer, ForeignKey("a.id"))
```

假设我们确实希望`A.id`和`B.id`互为镜像，尽管`B.a_id`是`A.id`相关的地方。我们可以使用`column_property()`将它们组合在一起：

```py
class A(Base):
    __tablename__ = "a"

    id = Column(Integer, primary_key=True)

class B(A):
    __tablename__ = "b"

    # probably not what you want, but this is a demonstration
    id = column_property(Column(Integer, primary_key=True), A.id)
    a_id = Column(Integer, ForeignKey("a.id"))
```

## 我正在使用声明式语法，并使用`and_()`或`or_()`设置`primaryjoin/secondaryjoin`，但是我收到了关于外键的错误消息。

你这样做了吗？：

```py
class MyClass(Base):
    # ....

    foo = relationship(
        "Dest", primaryjoin=and_("MyClass.id==Dest.foo_id", "MyClass.foo==Dest.bar")
    )
```

那是两个字符串表达式的`and_()`，而 SQLAlchemy 不能对其应用任何映射。声明式允许将`relationship()`参数指定为字符串，这些字符串将使用`eval()`转换为表达式对象。但这不会发生在`and_()`表达式内部 - 这是声明式仅对作为字符串传递给`primaryjoin`或其他参数的*整体*应用的特殊操作：

```py
class MyClass(Base):
    # ....

    foo = relationship(
        "Dest", primaryjoin="and_(MyClass.id==Dest.foo_id, MyClass.foo==Dest.bar)"
    )
```

或者，如果您需要的对象已经可用，请跳过字符串：

```py
class MyClass(Base):
    # ....

    foo = relationship(
        Dest, primaryjoin=and_(MyClass.id == Dest.foo_id, MyClass.foo == Dest.bar)
    )
```

相同的想法也适用于所有其他参数，比如`foreign_keys`：

```py
# wrong !
foo = relationship(Dest, foreign_keys=["Dest.foo_id", "Dest.bar_id"])

# correct !
foo = relationship(Dest, foreign_keys="[Dest.foo_id, Dest.bar_id]")

# also correct !
foo = relationship(Dest, foreign_keys=[Dest.foo_id, Dest.bar_id])

# if you're using columns from the class that you're inside of, just use the column objects !
class MyClass(Base):
    foo_id = Column(...)
    bar_id = Column(...)
    # ...

    foo = relationship(Dest, foreign_keys=[foo_id, bar_id])
```

## 为什么推荐使用`ORDER BY`与`LIMIT`（特别是与`subqueryload()`一起）？

当 SELECT 语句返回行时未使用 ORDER BY 时，关系数据库可以以任意顺序返回匹配的行。虽然这种排序很常见，对应于表中行的自然顺序，但并不是所有数据库和所有查询都是如此。这样做的结果是，任何使用`LIMIT`或`OFFSET`限制行，或者仅选择结果的第一行，而放弃其余部分的查询，在返回结果行时不是确定性的，假设有多个行匹配查询的条件。

尽管我们可能不会注意到这一点，因为对于通常以其自然顺序返回行的数据库上的简单查询，它更多地成为问题，如果我们还使用`subqueryload()`来加载相关集合，并且我们可能无法按预期加载集合。

SQLAlchemy 通过发出单独的查询来实现`subqueryload()`，其结果与第一个查询的结果匹配。我们会看到像这样发出两个查询：

```py
>>> session.scalars(select(User).options(subqueryload(User.addresses))).all()
-- the "main" query
SELECT  users.id  AS  users_id
FROM  users
-- the "load" query issued by subqueryload
SELECT  addresses.id  AS  addresses_id,
  addresses.user_id  AS  addresses_user_id,
  anon_1.users_id  AS  anon_1_users_id
FROM  (SELECT  users.id  AS  users_id  FROM  users)  AS  anon_1
JOIN  addresses  ON  anon_1.users_id  =  addresses.user_id
ORDER  BY  anon_1.users_id 
```

第二个查询将第一个查询嵌入为行的来源。当内部查询使用`OFFSET`和/或`LIMIT`而没有排序时，这两个查询可能不会看到相同的结果：

```py
>>> user = session.scalars(
...     select(User).options(subqueryload(User.addresses)).limit(1)
... ).first()
-- the "main" query
SELECT  users.id  AS  users_id
FROM  users
  LIMIT  1
-- the "load" query issued by subqueryload
SELECT  addresses.id  AS  addresses_id,
  addresses.user_id  AS  addresses_user_id,
  anon_1.users_id  AS  anon_1_users_id
FROM  (SELECT  users.id  AS  users_id  FROM  users  LIMIT  1)  AS  anon_1
JOIN  addresses  ON  anon_1.users_id  =  addresses.user_id
ORDER  BY  anon_1.users_id 
```

根据数据库的具体情况，我们可能会得到如下两个查询的结果：

```py
-- query #1
+--------+
|users_id|
+--------+
|       1|
+--------+

-- query #2
+------------+-----------------+---------------+
|addresses_id|addresses_user_id|anon_1_users_id|
+------------+-----------------+---------------+
|           3|                2|              2|
+------------+-----------------+---------------+
|           4|                2|              2|
+------------+-----------------+---------------+
```

在上面的例子中，我们为`user.id`为 2 的用户接收到了两行`addresses`，而对于 id 为 1 的用户却没有。我们浪费了两行，并且未能实际加载集合。这是一个隐匿的错误，因为不查看 SQL 和结果，ORM 将不会显示任何问题；如果我们访问已有的`User`的`addresses`，它会对集合进行惰性加载，我们将看不到任何实际错误发生。

解决此问题的方法是始终指定确定性排序顺序，以便主查询始终返回相同的行集。这通常意味着您应该在表的唯一列上进行 `Select.order_by()` 排序。主键是一个不错的选择：

```py
session.scalars(
    select(User).options(subqueryload(User.addresses)).order_by(User.id).limit(1)
).first()
```

注意，`joinedload()` 这种预加载策略不会遇到相同的问题，因为只会发出一个查询，所以加载查询不会与主查询不同。同样，`selectinload()` 这种预加载策略也不会有此问题，因为它将其集合加载直接链接到刚刚加载的主键值。

另请参阅

子查询预加载  ## 我如何映射一个没有主键的表？

SQLAlchemy ORM 为了映射到特定表，需要至少有一个列被指定为主键列；多列，即复合主键，当然也是完全可行的。这些列**不**需要实际上被数据库知道为主键列，尽管它们是主键列是个好主意。只需要这些列表现出主键的行为即可，例如作为行的唯一标识符和不可为空的标识符。

大多数 ORM 要求对象定义某种主键，因为内存中的对象必须对应于数据库表中的唯一可识别行；至少，这允许对象可以成为 UPDATE 和 DELETE 语句的目标，这些语句将仅影响该对象的行，而不会影响其他行。但是，主键的重要性远不止于此。在 SQLAlchemy 中，所有 ORM 映射的对象始终通过称为标识映射的模式与其特定数据库行唯一链接到一个 `Session` 中，该模式是 SQLAlchemy 使用的工作单元系统的核心，并且也是最常见（以及不那么常见）的 ORM 使用模式的关键。

注意

需要注意的是，我们只谈论 SQLAlchemy ORM；一个建立在 Core 之上、仅处理`Table`对象、`select()`构造等的应用程序，**不需要**在任何方式上要求主键存在于或与表相关联（尽管在 SQL 中，所有表实际上都应该具有某种主键，否则你可能需要实际更新或删除特定行）。

几乎在所有情况下，表都具有所谓的 候选键，这是一列或一系列列，唯一标识一行。如果表确实没有这个，且具有实际完全重复的行，则该表不符合[第一范式](https://en.wikipedia.org/wiki/First_normal_form)，无法进行映射。否则，组成最佳候选键的任何列都可以直接应用于映射器：

```py
class SomeClass(Base):
    __table__ = some_table_with_no_pk
    __mapper_args__ = {
        "primary_key": [some_table_with_no_pk.c.uid, some_table_with_no_pk.c.bar]
    }
```

当使用完全声明的表元数据时，最好在这些列上使用`primary_key=True`标志：

```py
class SomeClass(Base):
    __tablename__ = "some_table_with_no_pk"

    uid = Column(Integer, primary_key=True)
    bar = Column(String, primary_key=True)
```

关系数据库中的所有表都应该有主键。即使是多对多的关联表 - 主键也将是两个关联列的组合：

```py
CREATE  TABLE  my_association  (
  user_id  INTEGER  REFERENCES  user(id),
  account_id  INTEGER  REFERENCES  account(id),
  PRIMARY  KEY  (user_id,  account_id)
)
```

## 如何配置一个是 Python 保留字或类似的列？

在映射中，基于列的属性可以赋予任何所需的名称。参见显式命名声明式映射的列。

## 如何获取给定映射类的所有列、关系、映射属性等列表？

所有这些信息都可以从`Mapper`对象中获取。

要获取特定映射类的`Mapper`，请在其上调用`inspect()`函数：

```py
from sqlalchemy import inspect

mapper = inspect(MyClass)
```

从那里，可以通过诸如以下属性之类的属性访问有关类的所有信息：

+   `Mapper.attrs` - 所有映射属性的命名空间。这些属性本身是`MapperProperty`的实例，其中包含了可导致映射的 SQL 表达式或列的其他属性（如果适用）。

+   `Mapper.column_attrs` - 仅限于列和 SQL 表达式属性的映射属性命名空间。你可能想使用`Mapper.columns`直接获取 `Column`对象。

+   `Mapper.relationships` - 所有 `RelationshipProperty` 属性的命名空间。

+   `Mapper.all_orm_descriptors` - 所有映射属性的命名空间，以及使用诸如 `hybrid_property`、`AssociationProxy` 等系统定义的用户定义属性等。

+   `Mapper.columns` - 与映射相关联的 `Column` 对象和其他命名 SQL 表达式的命名空间。

+   `Mapper.mapped_table` - 此映射器映射到的 `Table` 或其他可选择的对象。

+   `Mapper.local_table` - 此映射器“本地”的 `Table`；在映射器使用继承映射到组合选择时，这与 `Mapper.mapped_table` 不同。

## 我收到了一个关于“隐式组合列 X 在属性 Y 下”的警告或错误

这种情况指的是映射包含两个列，这两个列由于它们的名称而被映射到同一属性名称下，但没有迹象表明这是有意的。映射类需要为每个要存储独立值的属性指定明确的名称；当两个列具有相同的名称并且没有消歧义时，它们就会落入同一个属性下，效果是从一个列中的值被**复制**到另一个列中，取决于哪个列首先分配给属性。

这种行为通常是可取的，在继承映射内部通过外键关系链接两个列时，无需警告即可允许。当出现警告或异常时，可以通过将列分配给不同命名的属性来解决问题，或者如果希望将它们组合在一起，则可以使用`column_property()`来明确表示这一点。

给出如下示例：

```py
from sqlalchemy import Integer, Column, ForeignKey
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()

class A(Base):
    __tablename__ = "a"

    id = Column(Integer, primary_key=True)

class B(A):
    __tablename__ = "b"

    id = Column(Integer, primary_key=True)
    a_id = Column(Integer, ForeignKey("a.id"))
```

截至 SQLAlchemy 版本 0.9.5，检测到上述条件，并将警告`A`和`B`的`id`列正在合并到同名属性`id`下，上面是一个严重问题，因为这意味着`B`对象的主键将始终反映其`A`的主键。

解决此问题的映射如下：

```py
class A(Base):
    __tablename__ = "a"

    id = Column(Integer, primary_key=True)

class B(A):
    __tablename__ = "b"

    b_id = Column("id", Integer, primary_key=True)
    a_id = Column(Integer, ForeignKey("a.id"))
```

假设我们确实希望`A.id`和`B.id`彼此镜像，尽管`B.a_id`是`A.id`相关的地方。我们可以使用`column_property()`将它们合并在一起：

```py
class A(Base):
    __tablename__ = "a"

    id = Column(Integer, primary_key=True)

class B(A):
    __tablename__ = "b"

    # probably not what you want, but this is a demonstration
    id = column_property(Column(Integer, primary_key=True), A.id)
    a_id = Column(Integer, ForeignKey("a.id"))
```

## 我正在使用声明式并使用`and_()`或`or_()`设置 primaryjoin/secondaryjoin，并且收到有关外键的错误消息。

您是这样做的吗？:

```py
class MyClass(Base):
    # ....

    foo = relationship(
        "Dest", primaryjoin=and_("MyClass.id==Dest.foo_id", "MyClass.foo==Dest.bar")
    )
```

这是两个字符串表达式的`and_()`，SQLAlchemy 无法对其应用任何映射。声明式允许将`relationship()`参数指定为字符串，并使用`eval()`将其转换为表达式对象。但这不会发生在`and_()`表达式内部 - 它是声明式仅适用于作为字符串传递给 primaryjoin 或其他参数的整体的特殊操作：

```py
class MyClass(Base):
    # ....

    foo = relationship(
        "Dest", primaryjoin="and_(MyClass.id==Dest.foo_id, MyClass.foo==Dest.bar)"
    )
```

或者如果您需要的对象已经可用，请跳过字符串：

```py
class MyClass(Base):
    # ....

    foo = relationship(
        Dest, primaryjoin=and_(MyClass.id == Dest.foo_id, MyClass.foo == Dest.bar)
    )
```

相同的想法适用于所有其他参数，例如`foreign_keys`：

```py
# wrong !
foo = relationship(Dest, foreign_keys=["Dest.foo_id", "Dest.bar_id"])

# correct !
foo = relationship(Dest, foreign_keys="[Dest.foo_id, Dest.bar_id]")

# also correct !
foo = relationship(Dest, foreign_keys=[Dest.foo_id, Dest.bar_id])

# if you're using columns from the class that you're inside of, just use the column objects !
class MyClass(Base):
    foo_id = Column(...)
    bar_id = Column(...)
    # ...

    foo = relationship(Dest, foreign_keys=[foo_id, bar_id])
```

## 为什么推荐在`LIMIT`中使用`ORDER BY`（特别是在`subqueryload()`中）？

当没有为返回行的 SELECT 语句使用 ORDER BY 时，关系数据库可以以任意的顺序返回匹配的行。虽然这种排序往往对应于表内行的自然顺序，但并非所有数据库和所有查询都是如此。这样做的结果是，任何使用`LIMIT`或`OFFSET`限制行数的查询，或者仅选择结果的第一行，丢弃其余行的查询，在返回哪个结果行时不是确定性的，假设查询的条件有多个匹配行。

尽管我们可能在通常按照它们的自然顺序返回行的数据库上的简单查询中没有注意到这一点，但如果我们还使用`subqueryload()`来加载相关集合，这就更成为一个问题，我们可能不会按预期加载集合。

SQLAlchemy 通过发出单独的查询来实现`subqueryload()`，其结果与第一个查询的结果匹配。我们看到像这样发出的两个查询：

```py
>>> session.scalars(select(User).options(subqueryload(User.addresses))).all()
-- the "main" query
SELECT  users.id  AS  users_id
FROM  users
-- the "load" query issued by subqueryload
SELECT  addresses.id  AS  addresses_id,
  addresses.user_id  AS  addresses_user_id,
  anon_1.users_id  AS  anon_1_users_id
FROM  (SELECT  users.id  AS  users_id  FROM  users)  AS  anon_1
JOIN  addresses  ON  anon_1.users_id  =  addresses.user_id
ORDER  BY  anon_1.users_id 
```

第二个查询将第一个查询嵌入为行的源。当内部查询使用`OFFSET`和/或`LIMIT`而没有排序时，这两个查询可能不会看到相同的结果：

```py
>>> user = session.scalars(
...     select(User).options(subqueryload(User.addresses)).limit(1)
... ).first()
-- the "main" query
SELECT  users.id  AS  users_id
FROM  users
  LIMIT  1
-- the "load" query issued by subqueryload
SELECT  addresses.id  AS  addresses_id,
  addresses.user_id  AS  addresses_user_id,
  anon_1.users_id  AS  anon_1_users_id
FROM  (SELECT  users.id  AS  users_id  FROM  users  LIMIT  1)  AS  anon_1
JOIN  addresses  ON  anon_1.users_id  =  addresses.user_id
ORDER  BY  anon_1.users_id 
```

根据数据库的具体情况，我们可能会得到以下两个查询的结果：

```py
-- query #1
+--------+
|users_id|
+--------+
|       1|
+--------+

-- query #2
+------------+-----------------+---------------+
|addresses_id|addresses_user_id|anon_1_users_id|
+------------+-----------------+---------------+
|           3|                2|              2|
+------------+-----------------+---------------+
|           4|                2|              2|
+------------+-----------------+---------------+
```

如上所述，我们对于`user.id`为 2 的两个`addresses`行，却没有对于 1 的。我们浪费了两行，并且未能实际加载集合。这是一个潜在的错误，因为如果不查看 SQL 和结果，ORM 将不会显示任何问题；如果我们访问我们拥有的`User`的`addresses`，它将对集合进行惰性加载，并且我们将看不到任何实际出错的情况。

解决这个问题的方法是始终指定确定性的排序顺序，以便主查询始终返回相同的行集合。这通常意味着你应该在表上的一个唯一列上使用`Select.order_by()`。主键是一个不错的选择：

```py
session.scalars(
    select(User).options(subqueryload(User.addresses)).order_by(User.id).limit(1)
).first()
```

请注意，`joinedload()` 急加载策略不会遭受相同的问题，因为只发出一次查询，因此加载查询不能与主查询不同。类似地，`selectinload()` 急加载策略也不会有此问题，因为它将其集合加载直接链接到刚刚加载的主键值。

另请参阅

子查询的急加载
