# SQL 表达式作为映射属性

> 原文：[`docs.sqlalchemy.org/en/20/orm/mapped_sql_expr.html`](https://docs.sqlalchemy.org/en/20/orm/mapped_sql_expr.html)

映射类上的属性可以链接到 SQL 表达式，这些表达式可以在查询中使用。

## 使用混合

将相对简单的 SQL 表达式链接到类的最简单和最灵活的方法是使用所谓的“混合属性”，在 混合属性 部分中描述。混合提供了一个同时在 Python 级别和 SQL 表达式级别工作的表达式。例如，我们将一个类 `User`，其中包含属性 `firstname` 和 `lastname`，映射到下面一个混合，该混合将为我们提供 `fullname`，即这两者的字符串连接：

```py
from sqlalchemy.ext.hybrid import hybrid_property

class User(Base):
    __tablename__ = "user"
    id = mapped_column(Integer, primary_key=True)
    firstname = mapped_column(String(50))
    lastname = mapped_column(String(50))

    @hybrid_property
    def fullname(self):
        return self.firstname + " " + self.lastname
```

在上面，`fullname` 属性在实例和类级别都被解释，因此可以从一个实例中使用：

```py
some_user = session.scalars(select(User).limit(1)).first()
print(some_user.fullname)
```

以及可在查询中使用：

```py
some_user = session.scalars(
    select(User).where(User.fullname == "John Smith").limit(1)
).first()
```

字符串连接示例是一个简单的示例，其中 Python 表达式可以在实例和类级别上兼用。通常，必须区分 SQL 表达式和 Python 表达式，可以使用`hybrid_property.expression()`来实现。下面我们展示了在混合内部需要存在条件的情况，使用 Python 中的`if`语句和 SQL 表达式的`case()`构造：

```py
from sqlalchemy.ext.hybrid import hybrid_property
from sqlalchemy.sql import case

class User(Base):
    __tablename__ = "user"
    id = mapped_column(Integer, primary_key=True)
    firstname = mapped_column(String(50))
    lastname = mapped_column(String(50))

    @hybrid_property
    def fullname(self):
        if self.firstname is not None:
            return self.firstname + " " + self.lastname
        else:
            return self.lastname

    @fullname.expression
    def fullname(cls):
        return case(
            (cls.firstname != None, cls.firstname + " " + cls.lastname),
            else_=cls.lastname,
        )
```

## 使用 column_property

`column_property()` 函数可用于将 SQL 表达式映射到与常规映射的 `Column` 类似的方式。使用这种技术，属性在加载时与所有其他列映射的属性一起加载。这在某些情况下优于使用混合的用法，因为该值可以在对象的父行加载时一次性加载，特别是如果表达式是链接到其他表（通常作为相关子查询）以访问通常不会在已加载对象上可用的数据的情况。

使用`column_property()`来表示 SQL 表达式的缺点包括表达式必须与整个类所发出的 SELECT 语句兼容，以及在使用来自声明性混合的`column_property()`时可能会出现一些配置怪癖。

我们的“fullname”示例可以使用`column_property()`表示如下：

```py
from sqlalchemy.orm import column_property

class User(Base):
    __tablename__ = "user"
    id = mapped_column(Integer, primary_key=True)
    firstname = mapped_column(String(50))
    lastname = mapped_column(String(50))
    fullname = column_property(firstname + " " + lastname)
```

也可以使用相关子查询。 下面我们使用`select()`构造创建一个`ScalarSelect`，表示一个面向列的 SELECT 语句，将特定`User`的可用`Address`对象的计数链接在一起：

```py
from sqlalchemy.orm import column_property
from sqlalchemy import select, func
from sqlalchemy import Column, Integer, String, ForeignKey

from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    pass

class Address(Base):
    __tablename__ = "address"
    id = mapped_column(Integer, primary_key=True)
    user_id = mapped_column(Integer, ForeignKey("user.id"))

class User(Base):
    __tablename__ = "user"
    id = mapped_column(Integer, primary_key=True)
    address_count = column_property(
        select(func.count(Address.id))
        .where(Address.user_id == id)
        .correlate_except(Address)
        .scalar_subquery()
    )
```

在上述示例中，我们定义了一个类似以下的`ScalarSelect()`构造：

```py
stmt = (
    select(func.count(Address.id))
    .where(Address.user_id == id)
    .correlate_except(Address)
    .scalar_subquery()
)
```

首先，我们使用`select()`创建一个`Select`构造，然后使用`Select.scalar_subquery()`方法将其转换为标量子查询，表示我们打算在列表达式上下文中使用此`Select`语句。

在`Select`本身中，我们选择计数`Address.id`行，其中`Address.user_id`列等于`id`，在`User`类的上下文中，`id`是名为`id`的`Column`（请注意，`id`也是 Python 内置函数的名称，这不是我们想在这里使用的 - 如果我们在`User`类定义之外，我们将使用`User.id`）。

`Select.correlate_except()`方法指示此`select()`中 FROM 子句的每个元素都可以从 FROM 列表中省略（即与针对`User`的封闭 SELECT 语句相关联），除了与`Address`对应的元素。 这并不是绝对必要的，但是在`User`和`Address`表之间进行一长串连接的情况下，防止`Address`意外地从 FROM 列表中省略。

对于引用从多对多关系链接的列的`column_property()`，使用`and_()`将关联表的字段与关系中的两个表连接起来：

```py
from sqlalchemy import and_

class Author(Base):
    # ...

    book_count = column_property(
        select(func.count(books.c.id))
        .where(
            and_(
                book_authors.c.author_id == authors.c.id,
                book_authors.c.book_id == books.c.id,
            )
        )
        .scalar_subquery()
    )
```

### 将 column_property()添加到现有的声明映射类

如果导入问题阻止在类中定义`column_property()`，则可以在两者配置后将其分配给类。当使用使用声明性基类（即由`DeclarativeBase`超类或遗留函数（例如`declarative_base()`）生成的映射时，此属性分配的效果是在事后调用`Mapper.add_property()`以添加额外的属性：

```py
# only works if a declarative base class is in use
User.address_count = column_property(
    select(func.count(Address.id)).where(Address.user_id == User.id).scalar_subquery()
)
```

当使用不使用声明性基类的映射样式，如`registry.mapped()`装饰器时，可以在底层`Mapper`对象上显式调用`Mapper.add_property()`方法，该对象可以使用`inspect()`获取：

```py
from sqlalchemy.orm import registry

reg = registry()

@reg.mapped
class User:
    __tablename__ = "user"

    # ... additional mapping directives

# later ...

# works for any kind of mapping
from sqlalchemy import inspect

inspect(User).add_property(
    column_property(
        select(func.count(Address.id))
        .where(Address.user_id == User.id)
        .scalar_subquery()
    )
)
```

另请参阅

将附加列添加到现有的声明式映射类

### 在映射时从列属性组合

可以创建将多个`ColumnProperty`对象组合在一起的映射。当在核心表达式上下文中使用`ColumnProperty`时，它将被解释为 SQL 表达式，前提是它被现有的表达式对象所指向；这是通过核心检测到对象具有`__clause_element__()`方法并返回 SQL 表达式来实现的。然而，如果`ColumnProperty`作为表达式中的主对象使用，而没有其他核心 SQL 表达式对象来指向它，那么`ColumnProperty.expression`属性将返回底层 SQL 表达式，以便可以一致地用于构建 SQL 表达式。下面，`File`类包含一个属性`File.path`，它将一个字符串令牌连接到`File.filename`属性上，该属性本身是一个`ColumnProperty`：

```py
class File(Base):
    __tablename__ = "file"

    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(64))
    extension = mapped_column(String(8))
    filename = column_property(name + "." + extension)
    path = column_property("C:/" + filename.expression)
```

当`File`类通常在表达式中使用时，分配给`filename`和`path`的属性可以直接使用。仅当直接在映射定义中使用`ColumnProperty`时，才需要使用`ColumnProperty.expression`属性：

```py
stmt = select(File.path).where(File.filename == "foo.txt")
```

### 使用 Column Deferral 与`column_property()`

在 ORM 查询指南中引入的列延迟特性可在映射时应用于由`column_property()`映射的 SQL 表达式，方法是在`column_property()`的位置使用`deferred()`函数而不是`column_property()`：

```py
from sqlalchemy.orm import deferred

class User(Base):
    __tablename__ = "user"

    id: Mapped[int] = mapped_column(primary_key=True)
    firstname: Mapped[str] = mapped_column()
    lastname: Mapped[str] = mapped_column()
    fullname: Mapped[str] = deferred(firstname + " " + lastname)
```

参见

使用 deferred()用于命令式映射器，映射的 SQL 表达式

## 使用简单描述符

在需要发出比`column_property()`或`hybrid_property`提供的 SQL 查询更复杂的情况下，可以使用作为属性访问的常规 Python 函数，假设表达式仅需要在已加载的实例上可用。该函数使用 Python 自己的`@property`装饰器装饰，将其标记为只读属性。在函数内部，使用`object_session()`定位到与当前对象对应的`Session`，然后用于发出查询：

```py
from sqlalchemy.orm import object_session
from sqlalchemy import select, func

class User(Base):
    __tablename__ = "user"
    id = mapped_column(Integer, primary_key=True)
    firstname = mapped_column(String(50))
    lastname = mapped_column(String(50))

    @property
    def address_count(self):
        return object_session(self).scalar(
            select(func.count(Address.id)).where(Address.user_id == self.id)
        )
```

简单描述符方法在紧急情况下很有用，但在通常情况下比混合和列属性方法性能更低，因为它需要在每次访问时发出 SQL 查询。

## 映射属性中的查询时 SQL 表达式

除了能够在映射类上配置固定的 SQL 表达式之外，SQLAlchemy ORM 还包括一个功能，可以在查询时将对象加载为任意 SQL 表达式的结果，并将其设置为其状态的一部分。通过使用 `query_expression()` 配置 ORM 映射属性，然后在查询时使用 `with_expression()` 加载器选项来实现此行为。查看 将任意 SQL 表达式加载到对象上 中的示例映射和用法。

## 使用混合

将相对简单的 SQL 表达式链接到类的最简单和最灵活的方法是使用所谓的“混合属性”，在 混合属性 部分中描述。混合提供了一个在 Python 级别和 SQL 表达式级别都起作用的表达式。例如，下面我们映射一个类 `User`，包含属性 `firstname` 和 `lastname`，并包含一个混合，将为我们提供 `fullname`，即两者的字符串连接：

```py
from sqlalchemy.ext.hybrid import hybrid_property

class User(Base):
    __tablename__ = "user"
    id = mapped_column(Integer, primary_key=True)
    firstname = mapped_column(String(50))
    lastname = mapped_column(String(50))

    @hybrid_property
    def fullname(self):
        return self.firstname + " " + self.lastname
```

在上面的示例中，`fullname` 属性在实例和类级别都被解释，因此可以从实例中使用：

```py
some_user = session.scalars(select(User).limit(1)).first()
print(some_user.fullname)
```

以及可在查询中使用：

```py
some_user = session.scalars(
    select(User).where(User.fullname == "John Smith").limit(1)
).first()
```

字符串拼接示例是一个简单的示例，其中 Python 表达式可以在实例和类级别上都起到双重作用。通常，必须区分 SQL 表达式和 Python 表达式，可以使用 `hybrid_property.expression()` 来实现。下面我们举例说明一个需要在混合中存在条件的情况，使用 Python 中的 `if` 语句和 SQL 表达式的 `case()` 结构：

```py
from sqlalchemy.ext.hybrid import hybrid_property
from sqlalchemy.sql import case

class User(Base):
    __tablename__ = "user"
    id = mapped_column(Integer, primary_key=True)
    firstname = mapped_column(String(50))
    lastname = mapped_column(String(50))

    @hybrid_property
    def fullname(self):
        if self.firstname is not None:
            return self.firstname + " " + self.lastname
        else:
            return self.lastname

    @fullname.expression
    def fullname(cls):
        return case(
            (cls.firstname != None, cls.firstname + " " + cls.lastname),
            else_=cls.lastname,
        )
```

## 使用 column_property

`column_property()` 函数可用于以与常规映射的 `Column` 类似的方式映射 SQL 表达式。通过此技术，属性在加载时与所有其他列映射的属性一起加载。在某些情况下，这比使用混合的优势更大，因为值可以在与对象的父行同时加载的同时前置加载，特别是如果表达式是链接到其他表的（通常作为关联子查询）以访问在已加载对象上通常不可用的数据。

使用`column_property()`进行 SQL 表达式的缺点包括表达式必须与整个类的 SELECT 语句兼容，并且在使用`column_property()`时可能会出现一些配置怪癖。

我们的“fullname”示例可以使用`column_property()`表示如下：

```py
from sqlalchemy.orm import column_property

class User(Base):
    __tablename__ = "user"
    id = mapped_column(Integer, primary_key=True)
    firstname = mapped_column(String(50))
    lastname = mapped_column(String(50))
    fullname = column_property(firstname + " " + lastname)
```

也可以使用相关子查询。下面我们使用`select()`构造创建一个`ScalarSelect`，表示一个面向列的 SELECT 语句，它链接了特定`User`的可用`Address`对象的计数：

```py
from sqlalchemy.orm import column_property
from sqlalchemy import select, func
from sqlalchemy import Column, Integer, String, ForeignKey

from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    pass

class Address(Base):
    __tablename__ = "address"
    id = mapped_column(Integer, primary_key=True)
    user_id = mapped_column(Integer, ForeignKey("user.id"))

class User(Base):
    __tablename__ = "user"
    id = mapped_column(Integer, primary_key=True)
    address_count = column_property(
        select(func.count(Address.id))
        .where(Address.user_id == id)
        .correlate_except(Address)
        .scalar_subquery()
    )
```

在上面的例子中，我们定义一个如下所示的`ScalarSelect()`构造：

```py
stmt = (
    select(func.count(Address.id))
    .where(Address.user_id == id)
    .correlate_except(Address)
    .scalar_subquery()
)
```

首先，我们使用`select()`创建一个`Select`构造，然后使用`Select.scalar_subquery()`方法将其转换为标量子查询，表明我们打算在列表达式上下文中使用这个`Select`语句。

在`Select`本身中，我们选择`Address.id`行的计数，其中`Address.user_id`列等于`id`，在`User`类的上下文中，`id`是名为`id`的`Column`（请注意，`id`也是 Python 内置函数的名称，这不是我们想要在此处使用的 - 如果我们在`User`类定义之外，我们将使用`User.id`）。

`Select.correlate_except()` 方法指示此 `select()` 的 FROM 子句中的每个元素都可以从 FROM 列表中省略（即与针对 `User` 的封闭 SELECT 语句相关联），除了与 `Address` 对应的元素。这并非绝对必要，但在 `User` 和 `Address` 表之间的一长串联接中，防止了 `Address` 在 SELECT 语句嵌套中无意中被省略出 FROM 列表。

对于引用来自多对多关系的列的 `column_property()`，使用 `and_()` 来将关联表的字段连接到关系中的两个表：

```py
from sqlalchemy import and_

class Author(Base):
    # ...

    book_count = column_property(
        select(func.count(books.c.id))
        .where(
            and_(
                book_authors.c.author_id == authors.c.id,
                book_authors.c.book_id == books.c.id,
            )
        )
        .scalar_subquery()
    )
```

### 向现有的声明式映射类添加 `column_property()` 

如果导入问题阻止内联定义 `column_property()` 与类一起定义，则在两者配置后可以将其分配给类。当使用使用声明式基类（即由 `DeclarativeBase` 超类或遗留函数如 `declarative_base()` 生成的映射）时，此属性分配具有调用 `Mapper.add_property()` 的效果，以在事后添加附加属性。

```py
# only works if a declarative base class is in use
User.address_count = column_property(
    select(func.count(Address.id)).where(Address.user_id == User.id).scalar_subquery()
)
```

当使用不使用声明式基类的映射样式时，例如 `registry.mapped()` 装饰器时，可以在底层的 `Mapper` 对象上显式调用 `Mapper.add_property()` 方法，可以使用 `inspect()` 获取该对象：

```py
from sqlalchemy.orm import registry

reg = registry()

@reg.mapped
class User:
    __tablename__ = "user"

    # ... additional mapping directives

# later ...

# works for any kind of mapping
from sqlalchemy import inspect

inspect(User).add_property(
    column_property(
        select(func.count(Address.id))
        .where(Address.user_id == User.id)
        .scalar_subquery()
    )
)
```

另请参阅

向现有的声明式映射类添加额外的列

### 在映射时从列属性组成

可以创建结合多个 `ColumnProperty` 对象的映射。当在核心表达式上下文中使用时，`ColumnProperty` 将被解释为 SQL 表达式，前提是它被现有表达式对象所定位；这通过核心检测对象是否具有返回 SQL 表达式的 `__clause_element__()` 方法来完成。然而，如果在表达式中将 `ColumnProperty` 用作领导对象，而没有其他核心 SQL 表达式对象来定位它，那么 `ColumnProperty.expression` 属性将返回底层 SQL 表达式，以便可以一致地构建 SQL 表达式。下面，`File` 类包含一个属性 `File.path`，它将一个字符串标记连接到 `File.filename` 属性，后者本身就是一个 `ColumnProperty`：

```py
class File(Base):
    __tablename__ = "file"

    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(64))
    extension = mapped_column(String(8))
    filename = column_property(name + "." + extension)
    path = column_property("C:/" + filename.expression)
```

当 `File` 类在表达式中正常使用时，分配给 `filename` 和 `path` 的属性可以直接使用。只有在映射定义中直接使用 `ColumnProperty` 时才需要使用 `ColumnProperty.expression` 属性：

```py
stmt = select(File.path).where(File.filename == "foo.txt")
```

### 使用 `column_property()` 进行列延迟

ORM 查询指南中介绍的列延迟功能可在映射时应用到由 `column_property()` 映射的 SQL 表达式上，方法是使用 `deferred()` 函数代替 `column_property()`：

```py
from sqlalchemy.orm import deferred

class User(Base):
    __tablename__ = "user"

    id: Mapped[int] = mapped_column(primary_key=True)
    firstname: Mapped[str] = mapped_column()
    lastname: Mapped[str] = mapped_column()
    fullname: Mapped[str] = deferred(firstname + " " + lastname)
```

另请参阅

使用 imperative mappers、映射的 SQL 表达式进行延迟加载

### 向现有的 Declarative 映射类添加 `column_property()`

如果导入问题阻止 `column_property()` 在类中内联定义，可以在两者配置后将其分配给类。在使用使用声明基类（即由 `DeclarativeBase` 超类或遗留函数如 `declarative_base()` 生成的）的映射时，此属性分配将调用 `Mapper.add_property()` 来添加一个额外的属性：

```py
# only works if a declarative base class is in use
User.address_count = column_property(
    select(func.count(Address.id)).where(Address.user_id == User.id).scalar_subquery()
)
```

在使用不使用声明基类的映射样式，例如 `registry.mapped()` 装饰器时，可以显式调用底层的 `Mapper.add_property()` 方法，这可以通过 `inspect()` 获取底层的 `Mapper` 对象来实现：

```py
from sqlalchemy.orm import registry

reg = registry()

@reg.mapped
class User:
    __tablename__ = "user"

    # ... additional mapping directives

# later ...

# works for any kind of mapping
from sqlalchemy import inspect

inspect(User).add_property(
    column_property(
        select(func.count(Address.id))
        .where(Address.user_id == User.id)
        .scalar_subquery()
    )
)
```

详见

向现有的声明映射类添加额外列

### 在映射时从列属性组成

可以创建将多个 `ColumnProperty` 对象组合在一起的映射。当在核心表达式上下文中使用时，如果 `ColumnProperty` 被现有表达式对象所定位，则它将被解释为 SQL 表达式；这是通过核心检测到对象具有返回 SQL 表达式的 `__clause_element__()` 方法来完成的。但是，如果在表达式中使用 `ColumnProperty` 作为主要对象，而没有其他核心 SQL 表达式对象来定位它，则 `ColumnProperty.expression` 属性将返回底层的 SQL 表达式，以便可以一致地构建 SQL 表达式。在下面的示例中，`File` 类包含一个属性 `File.path`，它将一个字符串令牌连接到 `File.filename` 属性上，后者本身是一个 `ColumnProperty`：

```py
class File(Base):
    __tablename__ = "file"

    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(64))
    extension = mapped_column(String(8))
    filename = column_property(name + "." + extension)
    path = column_property("C:/" + filename.expression)
```

当`File`类在表达式中正常使用时，分配给`filename`和`path`的属性可以直接使用。仅在映射定义中直接使用`ColumnProperty`时才需要使用`ColumnProperty.expression`属性：

```py
stmt = select(File.path).where(File.filename == "foo.txt")
```

### 使用`column_property()`进行列延迟

在 ORM 查询指南中引入的列延迟功能可以在映射时应用于由`column_property()`映射的 SQL 表达式，方法是在映射定义中使用 `deferred()` 函数代替`column_property()`：

```py
from sqlalchemy.orm import deferred

class User(Base):
    __tablename__ = "user"

    id: Mapped[int] = mapped_column(primary_key=True)
    firstname: Mapped[str] = mapped_column()
    lastname: Mapped[str] = mapped_column()
    fullname: Mapped[str] = deferred(firstname + " " + lastname)
```

另请参阅

对于命令式映射器，映射 SQL 表达式使用 deferred()

## 使用简单描述符

在需要发出比`column_property()`或`hybrid_property`提供的更复杂的 SQL 查询的情况下，可以使用作为属性访问的常规 Python 函数，假设表达式仅需要在已加载的实例上可用。该函数使用 Python 自己的 `@property` 装饰器来将其标记为只读属性。在函数内部，使用`object_session()`来定位与当前对象对应的`Session`，然后用于发出查询：

```py
from sqlalchemy.orm import object_session
from sqlalchemy import select, func

class User(Base):
    __tablename__ = "user"
    id = mapped_column(Integer, primary_key=True)
    firstname = mapped_column(String(50))
    lastname = mapped_column(String(50))

    @property
    def address_count(self):
        return object_session(self).scalar(
            select(func.count(Address.id)).where(Address.user_id == self.id)
        )
```

简单描述符方法通常作为最后一手之计，但在通常情况下，它的性能不如混合和列属性方法，因为每次访问都需要发出一条 SQL 查询。

## 查询时 SQL 表达式作为映射属性

除了能够在映射类上配置固定的 SQL 表达式之外，SQLAlchemy ORM 还包括一个功能，即对象可以使用在查询时设置为其状态的任意 SQL 表达式的结果进行加载。通过使用 `query_expression()` 配置 ORM 映射属性，然后在查询时使用 `with_expression()` 加载选项来实现这种行为。有关示例映射和用法，请参阅 将任意 SQL 表达式加载到对象上。
