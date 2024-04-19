# 列加载选项

> 原文：[`docs.sqlalchemy.org/en/20/orm/queryguide/columns.html`](https://docs.sqlalchemy.org/en/20/orm/queryguide/columns.html)

关于本文档

本节介绍了有关加载列的其他选项。使用的映射包括将存储大字符串值的列，我们可能希望限制它们何时加载。

查看此页面的 ORM 设置。以下示例中的一些将重新定义 `Book` 映射器以修改某些列定义。

## 使用列推迟限制加载的列

**列推迟** 指的是在查询该类型的对象时，从 SELECT 语句中省略的 ORM 映射列。这里的一般原理是性能，在表中具有很少使用的列，并且具有潜在的大数据值，因为在每次查询时完全加载这些列可能会耗费时间和/或内存。当实体加载时，SQLAlchemy ORM 提供了各种控制列加载的方式。

本节大多数示例演示了**ORM 加载器选项**。这些是传递给 `Select.options()` 方法的小构造，该方法是 `Select` 对象的一部分，当对象编译为 SQL 字符串时，ORM 将使用它们。

### 使用 `load_only()` 减少加载的列

`load_only()`加载器选项是在加载对象时最为便捷的选项，当已知只有少量列将被访问时，可以使用该选项。该选项接受一个可变数量的类绑定属性对象，指示应该加载的列映射属性，除了主键之外的所有其他列映射属性将不包括在检索的列中。在下面的示例中，`Book` 类包含列 `.title`、`.summary` 和 `.cover_photo`。使用 `load_only()` 我们可以指示 ORM 仅预先加载 `.title` 和 `.summary` 列：

```py
>>> from sqlalchemy import select
>>> from sqlalchemy.orm import load_only
>>> stmt = select(Book).options(load_only(Book.title, Book.summary))
>>> books = session.scalars(stmt).all()
SELECT  book.id,  book.title,  book.summary
FROM  book
[...]  ()
>>> for book in books:
...     print(f"{book.title}  {book.summary}")
100 Years of Krabby Patties  some long summary
Sea Catch 22  another long summary
The Sea Grapes of Wrath  yet another summary
A Nut Like No Other  some long summary
Geodesic Domes: A Retrospective  another long summary
Rocketry for Squirrels  yet another summary
```

在上面的示例中，SELECT 语句省略了 `.cover_photo` 列，并仅包括 `.title` 和 `.summary`，以及主键列 `.id`；ORM 通常会始终获取主键列，因为这些列是必需的，以建立行的标识。

加载后，对象通常将对其余未加载属性应用延迟加载行为，这意味着当首次访问时，将在当前事务中发出一个 SQL 语句以加载值。在下面的示例中，访问 `.cover_photo` 会发出一个 SELECT 语句来加载其值：

```py
>>> img_data = books[0].cover_photo
SELECT  book.cover_photo  AS  book_cover_photo
FROM  book
WHERE  book.id  =  ?
[...]  (1,) 
```

惰性加载始终使用对象处于 持久 状态的 `Session` 进行。如果对象从任何 `Session` 中 分离，操作将失败，引发异常。

作为在访问时进行惰性加载的替代方法，延迟列还可以配置为在访问时引发信息异常，而不考虑它们的附加状态。当使用 `load_only()` 构造时，可以使用 `load_only.raiseload` 参数来指示此行为。有关背景和示例，请参阅 使用 raiseload 防止延迟列加载 部分。

提示

正如其他地方所指出的，当使用异步 I/O（asyncio） 时，惰性加载不可用。

#### 使用 `load_only()` 处理多个实体

`load_only()` 限制自己仅适用于其属性列表中引用的单个实体（目前不允许传递跨越多个实体的属性列表）。在下面的示例中，给定的 `load_only()` 选项仅适用于 `Book` 实体。选择的 `User` 实体不受影响；在生成的 SELECT 语句中，所有 `user_account` 列均存在，而 `book` 表仅存在 `book.id` 和 `book.title`：

```py
>>> stmt = select(User, Book).join_from(User, Book).options(load_only(Book.title))
>>> print(stmt)
SELECT  user_account.id,  user_account.name,  user_account.fullname,
book.id  AS  id_1,  book.title
FROM  user_account  JOIN  book  ON  user_account.id  =  book.owner_id 
```

如果我们想要将 `load_only()` 选项应用于 `User` 和 `Book`，我们将使用两个单独的选项：

```py
>>> stmt = (
...     select(User, Book)
...     .join_from(User, Book)
...     .options(load_only(User.name), load_only(Book.title))
... )
>>> print(stmt)
SELECT  user_account.id,  user_account.name,  book.id  AS  id_1,  book.title
FROM  user_account  JOIN  book  ON  user_account.id  =  book.owner_id 
```

#### 在相关对象和集合上使用 `load_only()`

当使用关系加载器来控制相关对象的加载时，任何关系加载器的 `Load.load_only()` 方法都可以用于将 `load_only()` 规则应用于子实体上的列。在下面的示例中，`selectinload()` 用于在每个 `User` 对象上加载相关的 `books` 集合。通过将 `Load.load_only()` 应用于结果选项对象，当为关系加载对象时，生成的 SELECT 将仅引用 `title` 列以及主键列：

```py
>>> from sqlalchemy.orm import selectinload
>>> stmt = select(User).options(selectinload(User.books).load_only(Book.title))
>>> for user in session.scalars(stmt):
...     print(f"{user.fullname}   {[b.title for b in user.books]}")
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
[...]  ()
SELECT  book.owner_id  AS  book_owner_id,  book.id  AS  book_id,  book.title  AS  book_title
FROM  book
WHERE  book.owner_id  IN  (?,  ?)
[...]  (1,  2)
Spongebob Squarepants   ['100 Years of Krabby Patties', 'Sea Catch 22', 'The Sea Grapes of Wrath']
Sandy Cheeks   ['A Nut Like No Other', 'Geodesic Domes: A Retrospective', 'Rocketry for Squirrels']
```

`load_only()` 也可以应用于子实体，而无需声明要为关系本身使用的加载样式。如果我们不想更改 `User.books` 的默认加载方式，但仍然要应用于 `Book` 的 load only 规则，我们将使用 `defaultload()` 选项进行链接，在这种情况下，它将保留默认关系加载样式 `"lazy"`，并将我们的自定义 `load_only()` 规则应用于为每个 `User.books` 集合发出的 SELECT 语句：

```py
>>> from sqlalchemy.orm import defaultload
>>> stmt = select(User).options(defaultload(User.books).load_only(Book.title))
>>> for user in session.scalars(stmt):
...     print(f"{user.fullname}   {[b.title for b in user.books]}")
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
[...]  ()
SELECT  book.id  AS  book_id,  book.title  AS  book_title
FROM  book
WHERE  ?  =  book.owner_id
[...]  (1,)
Spongebob Squarepants   ['100 Years of Krabby Patties', 'Sea Catch 22', 'The Sea Grapes of Wrath']
SELECT  book.id  AS  book_id,  book.title  AS  book_title
FROM  book
WHERE  ?  =  book.owner_id
[...]  (2,)
Sandy Cheeks   ['A Nut Like No Other', 'Geodesic Domes: A Retrospective', 'Rocketry for Squirrels']
```  ### 使用 `defer()` 省略特定列

`defer()` 加载器选项是 `load_only()` 的一种更细粒度的替代方案，它允许将单个特定列标记为“不加载”。在下面的示例中，`defer()` 直接应用于 `.cover_photo` 列，而所有其他列的行为保持不变：

```py
>>> from sqlalchemy.orm import defer
>>> stmt = select(Book).where(Book.owner_id == 2).options(defer(Book.cover_photo))
>>> books = session.scalars(stmt).all()
SELECT  book.id,  book.owner_id,  book.title,  book.summary
FROM  book
WHERE  book.owner_id  =  ?
[...]  (2,)
>>> for book in books:
...     print(f"{book.title}: {book.summary}")
A Nut Like No Other: some long summary
Geodesic Domes: A Retrospective: another long summary
Rocketry for Squirrels: yet another summary
```

与 `load_only()` 一样，未加载的列默认情况下会在使用 惰性加载 访问时加载自身：

```py
>>> img_data = books[0].cover_photo
SELECT  book.cover_photo  AS  book_cover_photo
FROM  book
WHERE  book.id  =  ?
[...]  (4,) 
```

可以在一条语句中使用多个 `defer()` 选项来标记多个列为延迟加载。

与 `load_only()` 一样，`defer()` 选项也包括使延迟属性在访问时引发异常而不是惰性加载的能力。这在部分 使用 raiseload 防止延迟列加载 中有所说明。 ### 使用 raiseload 防止延迟列加载

在使用 `load_only()` 或 `defer()` 加载器选项时，对于对象上标记为延迟加载的属性，默认行为是在首次访问时，在当前事务中发出 SELECT 语句以加载它们的值。通常需要防止此加载发生，并在访问属性时引发异常，指示没有预期需要为该列查询数据库。典型的场景是使用已知对操作进行操作所需的所有列加载对象，然后将它们传递到视图层。应该捕获在视图层内部发出的任何进一步的 SQL 操作，以便可以调整预先加载的操作以适应该额外的数据，而不是产生额外的惰性加载。

对于此用例，`defer()`和`load_only()`选项包括一个布尔参数`defer.raiseload`，当设置为`True`时，将导致受影响的属性在访问时引发异常。在下面的示例中，延迟加载的列`.cover_photo`将禁止属性访问：

```py
>>> book = session.scalar(
...     select(Book).options(defer(Book.cover_photo, raiseload=True)).where(Book.id == 4)
... )
SELECT  book.id,  book.owner_id,  book.title,  book.summary
FROM  book
WHERE  book.id  =  ?
[...]  (4,)
>>> book.cover_photo
Traceback (most recent call last):
...
sqlalchemy.exc.InvalidRequestError: 'Book.cover_photo' is not available due to raiseload=True
```

当使用`load_only()`指定一组非延迟加载列时，可以使用`load_only.raiseload`参数来应用`raiseload`行为到其余列，该参数将应用于所有延迟属性：

```py
>>> session.expunge_all()
>>> book = session.scalar(
...     select(Book).options(load_only(Book.title, raiseload=True)).where(Book.id == 5)
... )
SELECT  book.id,  book.title
FROM  book
WHERE  book.id  =  ?
[...]  (5,)
>>> book.summary
Traceback (most recent call last):
...
sqlalchemy.exc.InvalidRequestError: 'Book.summary' is not available due to raiseload=True
```

注意

目前尚不可能在一条语句中混合使用指向同一实体的`load_only()`和`defer()`选项，以改变某些属性的`raiseload`行为；目前，这样做将产生未定义的属性加载行为。

另请参阅

`defer.raiseload`功能是与关系可用的相同“raiseload”功能的列级别版本。对于关系的“raiseload”，请参阅本指南的关系加载技术部分中的使用 raiseload 防止不必要的延迟加载。  ## 在映射上配置列延迟加载

对于映射列，默认情况下，`defer()`的功能可作为映射列的默认行为，这对于不应在每次查询时无条件加载的列可能是合适的。要配置，请使用`mapped_column.deferred`参数的`mapped_column()`。下面的示例说明了对`Book`的映射，该示例将默认列延迟应用于`summary`和`cover_photo`列：

```py
>>> class Book(Base):
...     __tablename__ = "book"
...     id: Mapped[int] = mapped_column(primary_key=True)
...     owner_id: Mapped[int] = mapped_column(ForeignKey("user_account.id"))
...     title: Mapped[str]
...     summary: Mapped[str] = mapped_column(Text, deferred=True)
...     cover_photo: Mapped[bytes] = mapped_column(LargeBinary, deferred=True)
...
...     def __repr__(self) -> str:
...         return f"Book(id={self.id!r}, title={self.title!r})"
```

使用上述映射，针对`Book`的查询将自动不包括`summary`和`cover_photo`列：

```py
>>> book = session.scalar(select(Book).where(Book.id == 2))
SELECT  book.id,  book.owner_id,  book.title
FROM  book
WHERE  book.id  =  ?
[...]  (2,) 
```

与所有延迟加载一样，当首次访问已加载对象上的延迟属性时，默认行为是它们将延迟加载它们的值：

```py
>>> img_data = book.cover_photo
SELECT  book.cover_photo  AS  book_cover_photo
FROM  book
WHERE  book.id  =  ?
[...]  (2,) 
```

与`defer()`和`load_only()`加载器选项一样，映射器级别的延迟加载还包括一个选项，当语句中没有其他选项时，可以发生`raiseload`行为，而不是惰性加载。这允许映射其中某些列默认情况下不加载，并且在语句中不使用明确指令时也永远不会懒加载。有关如何配置和使用此行为的背景信息，请参阅配置映射器级别的`raiseload`行为一节。

### 对于命令式映射器、映射 SQL 表达式使用`deferred()`

`deferred()`函数是早期的、更通用的“延迟列”映射指令，在引入`mapped_column()`构造之前就存在于 SQLAlchemy 中。

在配置 ORM 映射器时使用`deferred()`，并接受任意 SQL 表达式或`Column`对象。因此，它适用于非声明式命令式映射，将其传递给`map_imperatively.properties`字典：

```py
from sqlalchemy import Blob
from sqlalchemy import Column
from sqlalchemy import ForeignKey
from sqlalchemy import Integer
from sqlalchemy import String
from sqlalchemy import Table
from sqlalchemy import Text
from sqlalchemy.orm import registry

mapper_registry = registry()

book_table = Table(
    "book",
    mapper_registry.metadata,
    Column("id", Integer, primary_key=True),
    Column("title", String(50)),
    Column("summary", Text),
    Column("cover_image", Blob),
)

class Book:
    pass

mapper_registry.map_imperatively(
    Book,
    book_table,
    properties={
        "summary": deferred(book_table.c.summary),
        "cover_image": deferred(book_table.c.cover_image),
    },
)
```

当映射的 SQL 表达式应该延迟加载时，`deferred()`也可以用于替代`column_property()`：

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

使用 column_property - 在 SQL 表达式作为映射属性一节中

应用负载、持久性和映射选项到命令式表列 - 在使用声明式配置表一节中

### 使用`undefer()`来“急切地”加载延迟列

对于默认配置为延迟加载的映射上的列，`undefer()`选项将导致任何通常延迟加载的列变为未延迟加载，即与映射的所有其他列一起提前加载。例如，我们可以将`undefer()`应用于在前述映射中标记为延迟加载的`Book.summary`列：

```py
>>> from sqlalchemy.orm import undefer
>>> book = session.scalar(select(Book).where(Book.id == 2).options(undefer(Book.summary)))
SELECT  book.id,  book.owner_id,  book.title,  book.summary
FROM  book
WHERE  book.id  =  ?
[...]  (2,) 
```

现在，`Book.summary`列已经被急切地加载，并且可以在不发出额外 SQL 的情况下访问：

```py
>>> print(book.summary)
another long summary
```

### 将延迟列分组加载

通常，当列使用 `mapped_column(deferred=True)` 进行映射时，当在对象上访问延迟属性时，将发出 SQL 仅加载该特定列而不加载其他列，即使映射还有其他标记为延迟的列。在延迟属性是应一次性加载的一组属性的常见情况下，而不是为每个属性单独发出 SQL，可以使用 `mapped_column.deferred_group` 参数，该参数接受一个任意字符串，该字符串将定义要取消延迟的一组常见列：

```py
>>> class Book(Base):
...     __tablename__ = "book"
...     id: Mapped[int] = mapped_column(primary_key=True)
...     owner_id: Mapped[int] = mapped_column(ForeignKey("user_account.id"))
...     title: Mapped[str]
...     summary: Mapped[str] = mapped_column(
...         Text, deferred=True, deferred_group="book_attrs"
...     )
...     cover_photo: Mapped[bytes] = mapped_column(
...         LargeBinary, deferred=True, deferred_group="book_attrs"
...     )
...
...     def __repr__(self) -> str:
...         return f"Book(id={self.id!r}, title={self.title!r})"
```

使用上述映射，访问 `summary` 或 `cover_photo` 将同时加载两个列，只需使用一个 SELECT 语句：

```py
>>> book = session.scalar(select(Book).where(Book.id == 2))
SELECT  book.id,  book.owner_id,  book.title
FROM  book
WHERE  book.id  =  ?
[...]  (2,)
>>> img_data, summary = book.cover_photo, book.summary
SELECT  book.summary  AS  book_summary,  book.cover_photo  AS  book_cover_photo
FROM  book
WHERE  book.id  =  ?
[...]  (2,) 
```

### 使用 `undefer_group()` 按组取消延迟加载

如果在前一节中引入了 `mapped_column.deferred_group` 配置了延迟列，则可以指示整个组使用 `undefer_group()` 选项进行急切加载，传递要急切加载的组的字符串名称：

```py
>>> from sqlalchemy.orm import undefer_group
>>> book = session.scalar(
...     select(Book).where(Book.id == 2).options(undefer_group("book_attrs"))
... )
SELECT  book.id,  book.owner_id,  book.title,  book.summary,  book.cover_photo
FROM  book
WHERE  book.id  =  ?
[...]  (2,) 
```

`summary` 和 `cover_photo` 都可以在不加载其他内容的情况下使用：

```py
>>> img_data, summary = book.cover_photo, book.summary
```

### 使用通配符取消延迟加载

大多数 ORM 加载器选项接受通配符表达式，由 `"*"` 表示，表示该选项应用于所有相关属性。如果映射具有一系列延迟列，则可以一次性取消所有这些列的延迟，而无需使用组名，只需指定通配符：

```py
>>> book = session.scalar(select(Book).where(Book.id == 3).options(undefer("*")))
SELECT  book.id,  book.owner_id,  book.title,  book.summary,  book.cover_photo
FROM  book
WHERE  book.id  =  ?
[...]  (3,) 
```

### 配置映射器级别的“提前加载”行为

在 使用 raiseload 防止延迟列加载 中首次引入的 “raiseload” 行为也可以作为默认的映射器级别行为应用，使用 `mapped_column.deferred_raiseload` 参数的 `mapped_column()`。当使用此参数时，受影响的列将在所有情况下在访问时引发，除非在查询时显式地使用 `undefer()` 或 `load_only()` 进行“取消延迟”：

```py
>>> class Book(Base):
...     __tablename__ = "book"
...     id: Mapped[int] = mapped_column(primary_key=True)
...     owner_id: Mapped[int] = mapped_column(ForeignKey("user_account.id"))
...     title: Mapped[str]
...     summary: Mapped[str] = mapped_column(Text, deferred=True, deferred_raiseload=True)
...     cover_photo: Mapped[bytes] = mapped_column(
...         LargeBinary, deferred=True, deferred_raiseload=True
...     )
...
...     def __repr__(self) -> str:
...         return f"Book(id={self.id!r}, title={self.title!r})"
```

使用上述映射，`.summary` 和 `.cover_photo` 列默认情况下不可加载：

```py
>>> book = session.scalar(select(Book).where(Book.id == 2))
SELECT  book.id,  book.owner_id,  book.title
FROM  book
WHERE  book.id  =  ?
[...]  (2,)
>>> book.summary
Traceback (most recent call last):
...
sqlalchemy.exc.InvalidRequestError: 'Book.summary' is not available due to raiseload=True
```

只有在查询时重写它们的行为，通常使用 `undefer()` 或 `undefer_group()`，或者更少见的 `defer()`，属性才能被加载。下面的示例将 `undefer('*')` 应用于未延迟加载所有属性，并且还利用了填充现有对象来刷新已加载对象的加载器选项：

```py
>>> book = session.scalar(
...     select(Book)
...     .where(Book.id == 2)
...     .options(undefer("*"))
...     .execution_options(populate_existing=True)
... )
SELECT  book.id,  book.owner_id,  book.title,  book.summary,  book.cover_photo
FROM  book
WHERE  book.id  =  ?
[...]  (2,)
>>> book.summary
'another long summary'
```  ## 将任意 SQL 表达式加载到对象上

如选择 ORM 实体和属性及其他地方所讨论的，可以使用 `select()` 结构在结果集中加载任意 SQL 表达式。比如，如果我们想要发出一个查询，加载 `User` 对象，但也包括每个 `User` 拥有多少书籍的计数，我们可以使用 `func.count(Book.id)` 将“计数”列添加到一个查询中，该查询包括与 `Book` 的 JOIN 以及按所有者 id 进行的 GROUP BY。这将产生 `Row` 对象，每个对象包含两个条目，一个是 `User`，一个是 `func.count(Book.id)`：

```py
>>> from sqlalchemy import func
>>> stmt = select(User, func.count(Book.id)).join_from(User, Book).group_by(Book.owner_id)
>>> for user, book_count in session.execute(stmt):
...     print(f"Username: {user.name}  Number of books: {book_count}")
SELECT  user_account.id,  user_account.name,  user_account.fullname,
count(book.id)  AS  count_1
FROM  user_account  JOIN  book  ON  user_account.id  =  book.owner_id
GROUP  BY  book.owner_id
[...]  ()
Username: spongebob  Number of books: 3
Username: sandy  Number of books: 3
```

在上面的例子中，`User` 实体和“书籍数量”SQL 表达式分别返回。然而，一个常见的用例是生成一个查询，仅产生 `User` 对象，可以通过`Session.scalars()`来迭代，其中 `func.count(Book.id)` SQL 表达式的结果被*动态地*应用到每个 `User` 实体上。最终结果类似于在类上使用 `column_property()` 将任意 SQL 表达式映射到类的情况，只是 SQL 表达式可以在查询时进行修改。对于这种用例，SQLAlchemy 提供了 `with_expression()` 加载器选项，当与映射器级别的 `query_expression()` 指令结合使用时，可以产生这种结果。

要将 `with_expression()` 应用于查询，映射类必须预先使用 `query_expression()` 指令配置了一个 ORM 映射属性；这个指令将在映射类上生成一个适合接收查询时 SQL 表达式的属性。下面我们将一个新属性 `User.book_count` 添加到 `User` 中。这个 ORM 映射属性是只读的，没有默认值；在加载的实例上访问它通常会产生 `None`：

```py
>>> from sqlalchemy.orm import query_expression
>>> class User(Base):
...     __tablename__ = "user_account"
...     id: Mapped[int] = mapped_column(primary_key=True)
...     name: Mapped[str]
...     fullname: Mapped[Optional[str]]
...     book_count: Mapped[int] = query_expression()
...
...     def __repr__(self) -> str:
...         return f"User(id={self.id!r}, name={self.name!r}, fullname={self.fullname!r})"
```

使用我们映射中配置的 `User.book_count` 属性，我们可以使用 `with_expression()` 加载器选项，将数据从 SQL 表达式应用到每个 `User` 对象中加载的自定义 SQL 表达式中：

```py
>>> from sqlalchemy.orm import with_expression
>>> stmt = (
...     select(User)
...     .join_from(User, Book)
...     .group_by(Book.owner_id)
...     .options(with_expression(User.book_count, func.count(Book.id)))
... )
>>> for user in session.scalars(stmt):
...     print(f"Username: {user.name}  Number of books: {user.book_count}")
SELECT  count(book.id)  AS  count_1,  user_account.id,  user_account.name,
user_account.fullname
FROM  user_account  JOIN  book  ON  user_account.id  =  book.owner_id
GROUP  BY  book.owner_id
[...]  ()
Username: spongebob  Number of books: 3
Username: sandy  Number of books: 3
```

在上述示例中，我们将我们的 `func.count(Book.id)` 表达式从 `select()` 构造的 columns 参数中移出，并将其放入 `with_expression()` 加载器选项中。ORM 然后将其视为一个特殊的列加载选项，动态应用于语句。

`query_expression()` 映射有以下注意事项：

+   在未使用 `with_expression()` 来填充属性的对象上，对象实例上的属性将具有值 `None`，除非在映射上将 `query_expression.default_expr` 参数设置为默认的 SQL 表达式。

+   `with_expression()` 值**不会在已加载的对象上填充**，除非使用了 Populate Existing。如下示例**不起作用**，因为 `A` 对象已经加载：

    ```py
    # load the first A
    obj = session.scalars(select(A).order_by(A.id)).first()

    # load the same A with an option; expression will **not** be applied
    # to the already-loaded object
    obj = session.scalars(select(A).options(with_expression(A.expr, some_expr))).first()
    ```

    要确保在现有对象上重新加载属性，请使用 Populate Existing 执行选项以确保重新填充所有列：

    ```py
    obj = session.scalars(
        select(A)
        .options(with_expression(A.expr, some_expr))
        .execution_options(populate_existing=True)
    ).first()
    ```

+   当对象过期时，`with_expression()` SQL 表达式**会丢失**。一旦对象过期，无论是通过 `Session.expire()` 还是通过 `Session.commit()` 的 expire_on_commit 行为，SQL 表达式及其值将不再与属性关联，并且在后续访问时将返回 `None`。

+   `with_expression()` 作为对象加载选项，仅对**查询的最外层部分**以及对完整实体的查询起作用，而不适用于任意列选择、子查询或复合语句的元素，比如 UNION。请参阅下一节 使用 with_expression() 与 UNIONs、其他子查询 查看示例。

+   映射的属性**不能**应用于查询的其他部分，比如 WHERE 子句、ORDER BY 子句，并且使用临时表达式；也就是说，以下示例不起作用：

    ```py
    # can't refer to A.expr elsewhere in the query
    stmt = (
        select(A)
        .options(with_expression(A.expr, A.x + A.y))
        .filter(A.expr > 5)
        .order_by(A.expr)
    )
    ```

    在上述 WHERE 子句和 ORDER BY 子句中，`A.expr` 表达式将解析为 NULL。要在整个查询中使用该表达式，请赋值给一个变量然后使用它：

    ```py
    # assign desired expression up front, then refer to that in
    # the query
    a_expr = A.x + A.y
    stmt = (
        select(A)
        .options(with_expression(A.expr, a_expr))
        .filter(a_expr > 5)
        .order_by(a_expr)
    )
    ```

另请参阅

`with_expression()` 选项是一种特殊选项，用于在查询时动态应用 SQL 表达式到映射类。对于在映射器上配置的普通固定 SQL 表达式，请参阅 SQL 表达式作为映射属性 部分。

### 使用 `with_expression()` 与 UNIONs、其他子查询

`with_expression()` 构造是一种 ORM 加载器选项，因此只能应用于要加载特定 ORM 实体的 SELECT 语句的最外层级。如果在 `select()` 中使用，而后将其用作子查询或作为复合语句中的元素，如 UNION，它将不起作用。

要在子查询中使用任意 SQL 表达式，应使用常规的 Core 风格添加表达式的方法。要将子查询派生的表达式组装到 ORM 实体的 `query_expression()` 属性上，应在 ORM 对象加载的顶层使用 `with_expression()`，引用子查询中的 SQL 表达式。

在下面的示例中，使用两个 `select()` 构造针对带有额外 SQL 表达式标记为 `expr` 的 ORM 实体 `A`，并使用 `union_all()` 组合。然后，在最顶层，从此 UNION 中 SELECT `A` 实体，使用在 从 UNION 和其他集合操作中选择实体 中描述的查询技术，添加一个选项，使用 `with_expression()` 提取此 SQL 表达式到新加载的 `A` 实例上：

```py
>>> from sqlalchemy import union_all
>>> s1 = (
...     select(User, func.count(Book.id).label("book_count"))
...     .join_from(User, Book)
...     .where(User.name == "spongebob")
... )
>>> s2 = (
...     select(User, func.count(Book.id).label("book_count"))
...     .join_from(User, Book)
...     .where(User.name == "sandy")
... )
>>> union_stmt = union_all(s1, s2)
>>> orm_stmt = (
...     select(User)
...     .from_statement(union_stmt)
...     .options(with_expression(User.book_count, union_stmt.selected_columns.book_count))
... )
>>> for user in session.scalars(orm_stmt):
...     print(f"Username: {user.name}  Number of books: {user.book_count}")
SELECT  user_account.id,  user_account.name,  user_account.fullname,  count(book.id)  AS  book_count
FROM  user_account  JOIN  book  ON  user_account.id  =  book.owner_id
WHERE  user_account.name  =  ?
UNION  ALL
SELECT  user_account.id,  user_account.name,  user_account.fullname,  count(book.id)  AS  book_count
FROM  user_account  JOIN  book  ON  user_account.id  =  book.owner_id
WHERE  user_account.name  =  ?
[...]  ('spongebob',  'sandy')
Username: spongebob  Number of books: 3
Username: sandy  Number of books: 3
```

## 列加载 API

| 对象名称 | 描述 |
| --- | --- |
| defer(key, *addl_attrs, [raiseload]) | 指示给定的面向列的属性应该被延迟加载，例如，直到访问时才加载。 |
| deferred(column, *additional_columns, [group, raiseload, comparator_factory, init, repr, default, default_factory, compare, kw_only, active_history, expire_on_flush, info, doc]) | 指示默认情况下不加载的基于列的映射属性。 |
| load_only(*attrs, [raiseload]) | 表示对于特定实体，仅加载给定的列属性名列表；所有其他列将被延迟加载。 |
| query_expression([default_expr], *, [repr, compare, expire_on_flush, info, doc]) | 指示从查询时 SQL 表达式填充的属性。 |
| undefer(key, *addl_attrs) | 指示给定的基于列的属性应取消延迟加载，例如，可以在实体的 SELECT 语句中指定。 |
| undefer_group(name) | 指示给定延迟组名中的列应取消延迟加载。 |
| with_expression(key, expression) | 将临时 SQL 表达式应用于“延迟表达式”属性。 |

```py
function sqlalchemy.orm.defer(key: Literal['*'] | QueryableAttribute[Any], *addl_attrs: Literal['*'] | QueryableAttribute[Any], raiseload: bool = False) → _AbstractLoad
```

指示给定的基于列的属性应延迟加载，例如，直到访问时才加载。

此函数是 `Load` 接口的一部分，并支持方法链接和独立操作。

例如：

```py
from sqlalchemy.orm import defer

session.query(MyClass).options(
 defer(MyClass.attribute_one),
 defer(MyClass.attribute_two)
)
```

要指定对相关类的属性进行延迟加载，可以逐个令牌指定路径，并指定沿链的每个链接的加载样式。要保留链接的加载样式不变，请使用 `defaultload()`：

```py
session.query(MyClass).options(
 defaultload(MyClass.someattr).defer(RelatedClass.some_column)
)
```

可以使用 `Load.options()` 一次捆绑与关系相关的多个延迟选项：

```py
select(MyClass).options(
 defaultload(MyClass.someattr).options(
 defer(RelatedClass.some_column),
 defer(RelatedClass.some_other_column),
 defer(RelatedClass.another_column)
 )
)
```

参数：

+   `key` – 要延迟加载的属性。

+   `raiseload` – 在访问延迟属性时，引发 `InvalidRequestError` 而不是懒加载值。用于防止生成不需要的 SQL。

版本 1.4 中的新功能。

另请参阅

限制哪些列随列延迟加载 - 在 ORM 查询指南 中

`load_only()`

`undefer()`

```py
function sqlalchemy.orm.deferred(column: _ORMColumnExprArgument[_T], *additional_columns: _ORMColumnExprArgument[Any], group: str | None = None, raiseload: bool = False, comparator_factory: Type[PropComparator[_T]] | None = None, init: _NoArg | bool = _NoArg.NO_ARG, repr: _NoArg | bool = _NoArg.NO_ARG, default: Any | None = _NoArg.NO_ARG, default_factory: _NoArg | Callable[[], _T] = _NoArg.NO_ARG, compare: _NoArg | bool = _NoArg.NO_ARG, kw_only: _NoArg | bool = _NoArg.NO_ARG, active_history: bool = False, expire_on_flush: bool = True, info: _InfoType | None = None, doc: str | None = None) → MappedSQLExpression[_T]
```

表示默认情况下不会加载的基于列的映射属性，除非访问。

在使用 `mapped_column()` 时，通过使用 `mapped_column.deferred` 参数提供了与 `deferred()` 构造相同的功能。

参数：

+   `*columns` – 要映射的列。通常这是一个单独的 `Column` 对象，但是为了支持在同一属性下映射多个列，也支持集合。

+   `raiseload` –

    布尔值，如果为 True，则表示如果执行加载操作，则应引发异常。

    1.4 版中的新内容。

额外的参数与 `column_property()` 相同。

另请参阅

对命令式映射器、映射的 SQL 表达式使用 deferred()

```py
function sqlalchemy.orm.query_expression(default_expr: _ORMColumnExprArgument[_T] = <sqlalchemy.sql.elements.Null object>, *, repr: Union[_NoArg, bool] = _NoArg.NO_ARG, compare: Union[_NoArg, bool] = _NoArg.NO_ARG, expire_on_flush: bool = True, info: Optional[_InfoType] = None, doc: Optional[str] = None) → MappedSQLExpression[_T]
```

指示从查询时间 SQL 表达式填充的属性。

参数：

**default_expr** – 可选的 SQL 表达式对象，如果没有后续使用 `with_expression()` 分配，则将在所有情况下使用。

1.2 版中的新内容。

另请参阅

将任意 SQL 表达式加载到对象 - 背景和用法示例

```py
function sqlalchemy.orm.load_only(*attrs: Literal['*'] | QueryableAttribute[Any], raiseload: bool = False) → _AbstractLoad
```

指示对于特定实体，只加载给定的列名列表；所有其他属性将被延迟。

此函数是 `Load` 接口的一部分，支持方法链接和独立操作。

示例 - 给定一个类 `User`，只加载 `name` 和 `fullname` 属性：

```py
session.query(User).options(load_only(User.name, User.fullname))
```

示例 - 给定一个关系 `User.addresses -> Address`，为 `User.addresses` 集合指定子查询加载，但在每个 `Address` 对象上仅加载 `email_address` 属性：

```py
session.query(User).options(
 subqueryload(User.addresses).load_only(Address.email_address)
)
```

对于具有多个实体的语句，可以使用 `Load` 构造函数来明确指定引导实体：

```py
stmt = (
 select(User, Address)
 .join(User.addresses)
 .options(
 Load(User).load_only(User.name, User.fullname),
 Load(Address).load_only(Address.email_address),
 )
)
```

与 populate_existing 执行选项一起使用时，只会刷新列出的属性。

参数：

+   `*attrs` – 要加载的属性，所有其他属性都将延迟。

+   `raiseload` –

    当访问延迟属性时，引发 `InvalidRequestError` 而不是惰性加载值。用于防止不必要的 SQL 发出。

    2.0 版中的新内容。

另请参阅

限制加载的列与列延迟 - 在 ORM 查询指南 中

参数：

+   `*attrs` – 要加载的属性，所有其他属性都将延迟。

+   `raiseload` –

    当访问延迟属性时，引发 `InvalidRequestError` 而不是惰性加载值。用于防止不必要的 SQL 发出。

    2.0 版中的新内容。

```py
function sqlalchemy.orm.undefer(key: Literal['*'] | QueryableAttribute[Any], *addl_attrs: Literal['*'] | QueryableAttribute[Any]) → _AbstractLoad
```

指示给定的基于列的属性应该取消延迟，例如，在整个实体的 SELECT 语句中指定。

通常在映射上设置未延迟的列作为`deferred()` 属性。

此函数是 `Load` 接口的一部分，支持方法链接和独立操作。

示例：

```py
# undefer two columns
session.query(MyClass).options(
 undefer(MyClass.col1), undefer(MyClass.col2)
)

# undefer all columns specific to a single class using Load + *
session.query(MyClass, MyOtherClass).options(
 Load(MyClass).undefer("*")
)

# undefer a column on a related object
select(MyClass).options(
 defaultload(MyClass.items).undefer(MyClass.text)
)
```

参数：

**key** – 要取消延迟的属性。

另请参阅

使用列推迟限制加载的列 - 在 ORM 查询指南 中

`defer()`

`undefer_group()`

```py
function sqlalchemy.orm.undefer_group(name: str) → _AbstractLoad
```

指示给定延迟组名内的列应取消延迟。

正在取消延迟的列在映射上设置为 `deferred()` 属性，并包括一个“组”名称。

例如：

```py
session.query(MyClass).options(undefer_group("large_attrs"))
```

要取消相关实体上的一组属性的延迟加载，可以使用关系加载器选项（如`defaultload()`）拼写路径：

```py
select(MyClass).options(
 defaultload("someattr").undefer_group("large_attrs")
)
```

另请参阅

使用列推迟限制加载的列 - 在 ORM 查询指南 中

`defer()`

`undefer()`

```py
function sqlalchemy.orm.with_expression(key: _AttrType, expression: _ColumnExpressionArgument[Any]) → _AbstractLoad
```

将临时 SQL 表达式应用于“延迟表达式”属性。

此选项与`query_expression()` mapper-level 构造一起使用，指示应该是临时 SQL 表达式目标的属性。

例如：

```py
stmt = select(SomeClass).options(
 with_expression(SomeClass.x_y_expr, SomeClass.x + SomeClass.y)
)
```

版本 1.2 中的新增内容。

参数：

+   `key` – 要填充的属性

+   `expr` – 要应用于属性的 SQL 表达式。

另请参阅

将任意 SQL 表达式加载到对象上 - 背景和使用示例

## 使用列推迟限制加载的列

**列推迟**是指在查询该类型的对象时，ORM 映射的列在 SELECT 语句中被省略的列。 这里的一般原因是性能，在表具有很少使用的列且具有潜在的大数据值的情况下，完全在每次查询时加载这些列可能会耗费时间和/或内存。 SQLAlchemy ORM 提供了多种控制加载列的方式。

本节中的大多数示例都是**ORM 加载器选项**的示例。 这些是小型构造，传递给 `Select.options()` 方法的 `Select` 对象，然后在对象编译为 SQL 字符串时由 ORM 消耗。

### 使用 `load_only()` 来减少加载的列

`load_only()` 加载器选项是在已知只会访问少量列的对象时使用的最快捷的选项。此选项接受一个可变数量的类绑定属性对象，指示应加载的那些列映射属性，其中除主键外的所有其他列映射属性将不会成为被获取的列的一部分。在下面的示例中，`Book` 类包含列 `.title`、`.summary` 和 `.cover_photo`。使用 `load_only()`，我们可以指示 ORM 仅预先加载 `.title` 和 `.summary` 列：

```py
>>> from sqlalchemy import select
>>> from sqlalchemy.orm import load_only
>>> stmt = select(Book).options(load_only(Book.title, Book.summary))
>>> books = session.scalars(stmt).all()
SELECT  book.id,  book.title,  book.summary
FROM  book
[...]  ()
>>> for book in books:
...     print(f"{book.title}  {book.summary}")
100 Years of Krabby Patties  some long summary
Sea Catch 22  another long summary
The Sea Grapes of Wrath  yet another summary
A Nut Like No Other  some long summary
Geodesic Domes: A Retrospective  another long summary
Rocketry for Squirrels  yet another summary
```

在上面的例子中，SELECT 语句省略了 `.cover_photo` 列，并且仅包含了 `.title` 和 `.summary` 列，以及主键列 `.id`；ORM 通常会始终获取主键列，因为这些列是必需的，用于建立行的标识。

一旦加载，对象通常会对其余未加载的属性应用惰性加载行为，这意味着当首次访问任何属性时，将在当前事务中发出 SQL 语句以加载该值。下面，访问 `.cover_photo` 会发出一个 SELECT 语句来加载其值：

```py
>>> img_data = books[0].cover_photo
SELECT  book.cover_photo  AS  book_cover_photo
FROM  book
WHERE  book.id  =  ?
[...]  (1,) 
```

惰性加载始终使用对象处于持久状态的`Session`发出。如果对象已分离于任何`Session`，操作将失败，引发异常。

作为在访问时惰性加载的替代方案，还可以配置延迟列在访问时引发一个信息性异常，而不考虑它们的附加状态。当使用 `load_only()` 构造时，可以使用 `load_only.raiseload` 参数来指示这一点。有关背景和示例，请参见使用 raiseload 防止延迟列加载部分。

提示

如其他地方所述，在使用异步 I/O (asyncio)时，不可用惰性加载。

#### 使用 `load_only()` 与多个实体

`load_only()` 限制自身仅针对其属性列表中引用的单个实体（目前不允许传递跨越多个实体的属性列表）。在下面的示例中，给定的 `load_only()` 选项仅适用于 `Book` 实体。也选择的 `User` 实体不受影响；在生成的 SELECT 语句中，`user_account` 的所有列都存在，而 `book` 表只有 `book.id` 和 `book.title`：

```py
>>> stmt = select(User, Book).join_from(User, Book).options(load_only(Book.title))
>>> print(stmt)
SELECT  user_account.id,  user_account.name,  user_account.fullname,
book.id  AS  id_1,  book.title
FROM  user_account  JOIN  book  ON  user_account.id  =  book.owner_id 
```

如果我们想要将 `load_only()` 选项应用于 `User` 和 `Book`，我们将使用两个单独的选项：

```py
>>> stmt = (
...     select(User, Book)
...     .join_from(User, Book)
...     .options(load_only(User.name), load_only(Book.title))
... )
>>> print(stmt)
SELECT  user_account.id,  user_account.name,  book.id  AS  id_1,  book.title
FROM  user_account  JOIN  book  ON  user_account.id  =  book.owner_id 
```

#### 在相关对象和集合上使用 `load_only()`

当使用关系加载器来控制相关对象的加载时，任何关系加载器的 `Load.load_only()` 方法都可以用于将 `load_only()` 规则应用于子实体的列。在下面的示例中，使用 `selectinload()` 来加载每个 `User` 对象上的相关 `books` 集合。通过将 `Load.load_only()` 应用于生成的选项对象，当加载关系的对象时，生成的 SELECT 语句将仅引用 `title` 列以及主键列：

```py
>>> from sqlalchemy.orm import selectinload
>>> stmt = select(User).options(selectinload(User.books).load_only(Book.title))
>>> for user in session.scalars(stmt):
...     print(f"{user.fullname}   {[b.title for b in user.books]}")
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
[...]  ()
SELECT  book.owner_id  AS  book_owner_id,  book.id  AS  book_id,  book.title  AS  book_title
FROM  book
WHERE  book.owner_id  IN  (?,  ?)
[...]  (1,  2)
Spongebob Squarepants   ['100 Years of Krabby Patties', 'Sea Catch 22', 'The Sea Grapes of Wrath']
Sandy Cheeks   ['A Nut Like No Other', 'Geodesic Domes: A Retrospective', 'Rocketry for Squirrels']
```

`load_only()` 也可以应用于子实体，而无需声明要用于关系本身的加载样式。如果我们不想更改 `User.books` 的默认加载样式，但仍要对 `Book` 应用仅加载规则，我们将使用 `defaultload()` 选项进行链接，在这种情况下，将保留 `"lazy"` 的默认关系加载样式，并将我们的自定义 `load_only()` 规则应用于为每个 `User.books` 集合发出的 SELECT 语句：

```py
>>> from sqlalchemy.orm import defaultload
>>> stmt = select(User).options(defaultload(User.books).load_only(Book.title))
>>> for user in session.scalars(stmt):
...     print(f"{user.fullname}   {[b.title for b in user.books]}")
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
[...]  ()
SELECT  book.id  AS  book_id,  book.title  AS  book_title
FROM  book
WHERE  ?  =  book.owner_id
[...]  (1,)
Spongebob Squarepants   ['100 Years of Krabby Patties', 'Sea Catch 22', 'The Sea Grapes of Wrath']
SELECT  book.id  AS  book_id,  book.title  AS  book_title
FROM  book
WHERE  ?  =  book.owner_id
[...]  (2,)
Sandy Cheeks   ['A Nut Like No Other', 'Geodesic Domes: A Retrospective', 'Rocketry for Squirrels']
```  ### 使用 `defer()` 省略特定列

`defer()` 加载选项是对 `load_only()` 的更精细的替代方案，允许将单个特定列标记为“不加载”。在下面的示例中，`defer()` 直接应用于 `.cover_photo` 列，保持所有其他列的行为不变：

```py
>>> from sqlalchemy.orm import defer
>>> stmt = select(Book).where(Book.owner_id == 2).options(defer(Book.cover_photo))
>>> books = session.scalars(stmt).all()
SELECT  book.id,  book.owner_id,  book.title,  book.summary
FROM  book
WHERE  book.owner_id  =  ?
[...]  (2,)
>>> for book in books:
...     print(f"{book.title}: {book.summary}")
A Nut Like No Other: some long summary
Geodesic Domes: A Retrospective: another long summary
Rocketry for Squirrels: yet another summary
```

与 `load_only()` 相同，未加载的列默认情况下将在使用惰性加载时自行加载：

```py
>>> img_data = books[0].cover_photo
SELECT  book.cover_photo  AS  book_cover_photo
FROM  book
WHERE  book.id  =  ?
[...]  (4,) 
```

可以在一个语句中使用多个 `defer()` 选项来标记多个列为延迟加载。

与 `load_only()` 相同，`defer()` 选项也包括使延迟属性在访问时引发异常而不是惰性加载的功能。这在使用 raiseload 防止延迟列加载一节中进行了说明。### 使用 raiseload 防止延迟列加载

当使用 `load_only()` 或 `defer()` 加载器选项时，对象上标记为延迟的属性具有默认行为，即在首次访问时，将在当前事务中发出 SELECT 语句以加载其值。通常需要阻止此加载操作，并在访问属性时引发异常，指示不期望为此列查询数据库的需要。典型的情况是加载具有操作所需的所有已知列的对象，然后将它们传递到视图层。视图层中发出的任何进一步的 SQL 操作都应该被捕获，以便调整前期加载操作以适应那些额外的数据，而不是额外的惰性加载。

对于这种用例，`defer()` 和 `load_only()` 选项包括一个布尔参数 `defer.raiseload`，当设置为 `True` 时，将导致受影响的属性在访问时引发异常。在下面的示例中，延迟加载的列 `.cover_photo` 将禁止属性访问：

```py
>>> book = session.scalar(
...     select(Book).options(defer(Book.cover_photo, raiseload=True)).where(Book.id == 4)
... )
SELECT  book.id,  book.owner_id,  book.title,  book.summary
FROM  book
WHERE  book.id  =  ?
[...]  (4,)
>>> book.cover_photo
Traceback (most recent call last):
...
sqlalchemy.exc.InvalidRequestError: 'Book.cover_photo' is not available due to raiseload=True
```

当使用 `load_only()` 命名一组特定的非延迟加载列时，可以使用 `load_only.raiseload` 参数将 `raiseload` 行为应用于其余列，该参数将应用于所有延迟加载属性：

```py
>>> session.expunge_all()
>>> book = session.scalar(
...     select(Book).options(load_only(Book.title, raiseload=True)).where(Book.id == 5)
... )
SELECT  book.id,  book.title
FROM  book
WHERE  book.id  =  ?
[...]  (5,)
>>> book.summary
Traceback (most recent call last):
...
sqlalchemy.exc.InvalidRequestError: 'Book.summary' is not available due to raiseload=True
```

注意

目前尚不能在一个语句中混合使用指向同一实体的 `load_only()` 和 `defer()` 选项，以改变某些属性的 `raiseload` 行为；目前，这样做会产生未定义的属性加载行为。

另请参见

`defer.raiseload` 功能是关系的同一“raiseload”功能的列级版本。有关关系的“raiseload”，请参见本指南的关系加载技术部分中的使用 `load_only()` 减少加载的列。

当已知只有少数几列将被访问时，`load_only()`加载器选项是最方便的选项。该选项接受一个变量数量的类绑定属性对象，指示应该加载的列映射属性，除了主键之外的所有其他列映射属性都不会成为获取的列的一部分。在下面的示例中，`Book` 类包含列 `.title`、`.summary` 和 `.cover_photo`。使用`load_only()`，我们可以指示 ORM 仅预先加载 `.title` 和 `.summary` 列：

```py
>>> from sqlalchemy import select
>>> from sqlalchemy.orm import load_only
>>> stmt = select(Book).options(load_only(Book.title, Book.summary))
>>> books = session.scalars(stmt).all()
SELECT  book.id,  book.title,  book.summary
FROM  book
[...]  ()
>>> for book in books:
...     print(f"{book.title}  {book.summary}")
100 Years of Krabby Patties  some long summary
Sea Catch 22  another long summary
The Sea Grapes of Wrath  yet another summary
A Nut Like No Other  some long summary
Geodesic Domes: A Retrospective  another long summary
Rocketry for Squirrels  yet another summary
```

在上面的示例中，SELECT 语句省略了 `.cover_photo` 列，仅包含了 `.title` 和 `.summary`，以及主键列 `.id`；ORM 通常会获取主键列，因为这些列是必需的，以建立行的标识。

加载后，对象通常将对其余未加载的属性应用惰性加载行为，这意味着首次访问任何属性时，将在当前事务中发出 SQL 语句以加载值。下面，访问 `.cover_photo` 会发出一个 SELECT 语句来加载它的值：

```py
>>> img_data = books[0].cover_photo
SELECT  book.cover_photo  AS  book_cover_photo
FROM  book
WHERE  book.id  =  ?
[...]  (1,) 
```

惰性加载始终使用对象所处的处于持久状态的 `Session` 发出。如果对象从任何`Session`中分离，操作将失败，引发异常。

作为访问时惰性加载的替代方案，还可以配置延迟列以在访问时引发信息性异常，而不考虑它们的附加状态。在使用`load_only()`构造时，可以使用`load_only.raiseload`参数来指示此情况。有关背景和示例，请参阅使用 raiseload 防止延迟列加载部分。

提示

正如在其他地方所指出的，使用异步 I/O（asyncio）时不可用惰性加载。

#### 使用 `load_only()` 处理多个实体

`load_only()` 限制了其属性列表中所引用的单个实体（当前不允许传递跨越多个实体的属性列表）。在下面的示例中，给定的 `load_only()` 选项仅适用于 `Book` 实体。被选中的 `User` 实体不受影响；在生成的 SELECT 语句中，`user_account` 的所有列都存在，而 `book` 表只有 `book.id` 和 `book.title`：

```py
>>> stmt = select(User, Book).join_from(User, Book).options(load_only(Book.title))
>>> print(stmt)
SELECT  user_account.id,  user_account.name,  user_account.fullname,
book.id  AS  id_1,  book.title
FROM  user_account  JOIN  book  ON  user_account.id  =  book.owner_id 
```

如果我们想要同时将 `load_only()` 选项应用于 `User` 和 `Book`，我们将使用两个单独的选项：

```py
>>> stmt = (
...     select(User, Book)
...     .join_from(User, Book)
...     .options(load_only(User.name), load_only(Book.title))
... )
>>> print(stmt)
SELECT  user_account.id,  user_account.name,  book.id  AS  id_1,  book.title
FROM  user_account  JOIN  book  ON  user_account.id  =  book.owner_id 
```

#### 对相关对象和集合使用 `load_only()`

在使用 关系加载器 控制相关对象加载时，任何关系加载器的 `Load.load_only()` 方法都可以用于将 `load_only()` 规则应用于子实体上的列。在下面的示例中，`selectinload()` 用于加载每个 `User` 对象上的相关 `books` 集合。通过将 `Load.load_only()` 应用于结果选项对象，当为关系加载对象时，生成的 SELECT 仅引用 `title` 列以及主键列：

```py
>>> from sqlalchemy.orm import selectinload
>>> stmt = select(User).options(selectinload(User.books).load_only(Book.title))
>>> for user in session.scalars(stmt):
...     print(f"{user.fullname}   {[b.title for b in user.books]}")
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
[...]  ()
SELECT  book.owner_id  AS  book_owner_id,  book.id  AS  book_id,  book.title  AS  book_title
FROM  book
WHERE  book.owner_id  IN  (?,  ?)
[...]  (1,  2)
Spongebob Squarepants   ['100 Years of Krabby Patties', 'Sea Catch 22', 'The Sea Grapes of Wrath']
Sandy Cheeks   ['A Nut Like No Other', 'Geodesic Domes: A Retrospective', 'Rocketry for Squirrels']
```

`load_only()` 还可以应用于子实体，而无需声明要在关系本身使用的加载样式。如果我们不想改变 `User.books` 的默认加载样式，但仍要对 `Book` 应用加载规则，我们将使用 `defaultload()` 选项进行关联，在这种情况下，将保留默认关系加载样式 `"lazy"`，并将我们的自定义 `load_only()` 规则应用于为每个 `User.books` 集合发出的 SELECT 语句：

```py
>>> from sqlalchemy.orm import defaultload
>>> stmt = select(User).options(defaultload(User.books).load_only(Book.title))
>>> for user in session.scalars(stmt):
...     print(f"{user.fullname}   {[b.title for b in user.books]}")
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
[...]  ()
SELECT  book.id  AS  book_id,  book.title  AS  book_title
FROM  book
WHERE  ?  =  book.owner_id
[...]  (1,)
Spongebob Squarepants   ['100 Years of Krabby Patties', 'Sea Catch 22', 'The Sea Grapes of Wrath']
SELECT  book.id  AS  book_id,  book.title  AS  book_title
FROM  book
WHERE  ?  =  book.owner_id
[...]  (2,)
Sandy Cheeks   ['A Nut Like No Other', 'Geodesic Domes: A Retrospective', 'Rocketry for Squirrels']
```

#### 使用 `load_only()` 处理多个实体

`load_only()` 限制了其属性列表中所引用的单个实体（当前不允许传递跨越多个实体的属性列表）。在下面的示例中，给定的 `load_only()` 选项仅适用于 `Book` 实体。被选中的 `User` 实体不受影响；在生成的 SELECT 语句中，`user_account` 的所有列都存在，而 `book` 表只有 `book.id` 和 `book.title`：

```py
>>> stmt = select(User, Book).join_from(User, Book).options(load_only(Book.title))
>>> print(stmt)
SELECT  user_account.id,  user_account.name,  user_account.fullname,
book.id  AS  id_1,  book.title
FROM  user_account  JOIN  book  ON  user_account.id  =  book.owner_id 
```

如果我们想要将`load_only()`选项应用于`User`和`Book`，我们将使用两个单独的选项：

```py
>>> stmt = (
...     select(User, Book)
...     .join_from(User, Book)
...     .options(load_only(User.name), load_only(Book.title))
... )
>>> print(stmt)
SELECT  user_account.id,  user_account.name,  book.id  AS  id_1,  book.title
FROM  user_account  JOIN  book  ON  user_account.id  =  book.owner_id 
```

#### 在相关对象和集合上使用 `load_only()`

当使用关系加载器来控制相关对象的加载时，可以使用任何关系加载器的`Load.load_only()`方法将`load_only()`规则应用于子实体上的列。在下面的示例中，使用`selectinload()`加载每个`User`对象上的相关`books`集合。通过将`Load.load_only()`应用于结果选项对象，当为关系加载对象时，生成的 SELECT 将仅引用`title`列以及主键列：

```py
>>> from sqlalchemy.orm import selectinload
>>> stmt = select(User).options(selectinload(User.books).load_only(Book.title))
>>> for user in session.scalars(stmt):
...     print(f"{user.fullname}   {[b.title for b in user.books]}")
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
[...]  ()
SELECT  book.owner_id  AS  book_owner_id,  book.id  AS  book_id,  book.title  AS  book_title
FROM  book
WHERE  book.owner_id  IN  (?,  ?)
[...]  (1,  2)
Spongebob Squarepants   ['100 Years of Krabby Patties', 'Sea Catch 22', 'The Sea Grapes of Wrath']
Sandy Cheeks   ['A Nut Like No Other', 'Geodesic Domes: A Retrospective', 'Rocketry for Squirrels']
```

`load_only()`也可以应用于子实体，而无需说明用于关系本身的加载样式。如果我们不想更改`User.books`的默认加载样式，但仍要将加载仅规则应用于`Book`，我们将使用`defaultload()`选项进行链接，在这种情况下，将保留默认关系加载样式`"lazy"`，并将我们的自定义`load_only()`规则应用于为每个`User.books`集合发出的 SELECT 语句：

```py
>>> from sqlalchemy.orm import defaultload
>>> stmt = select(User).options(defaultload(User.books).load_only(Book.title))
>>> for user in session.scalars(stmt):
...     print(f"{user.fullname}   {[b.title for b in user.books]}")
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
[...]  ()
SELECT  book.id  AS  book_id,  book.title  AS  book_title
FROM  book
WHERE  ?  =  book.owner_id
[...]  (1,)
Spongebob Squarepants   ['100 Years of Krabby Patties', 'Sea Catch 22', 'The Sea Grapes of Wrath']
SELECT  book.id  AS  book_id,  book.title  AS  book_title
FROM  book
WHERE  ?  =  book.owner_id
[...]  (2,)
Sandy Cheeks   ['A Nut Like No Other', 'Geodesic Domes: A Retrospective', 'Rocketry for Squirrels']
```

### 使用 `defer()` 来省略特定列

`defer()`加载器选项是对`load_only()`的更精细的替代，它允许将单个特定列标记为“不加载”。在下面的示例中，直接应用`defer()`到`.cover_photo`列，保持所有其他列的行为不变：

```py
>>> from sqlalchemy.orm import defer
>>> stmt = select(Book).where(Book.owner_id == 2).options(defer(Book.cover_photo))
>>> books = session.scalars(stmt).all()
SELECT  book.id,  book.owner_id,  book.title,  book.summary
FROM  book
WHERE  book.owner_id  =  ?
[...]  (2,)
>>> for book in books:
...     print(f"{book.title}: {book.summary}")
A Nut Like No Other: some long summary
Geodesic Domes: A Retrospective: another long summary
Rocketry for Squirrels: yet another summary
```

与`load_only()`一样，默认情况下未加载的列在使用惰性加载时会自行加载：

```py
>>> img_data = books[0].cover_photo
SELECT  book.cover_photo  AS  book_cover_photo
FROM  book
WHERE  book.id  =  ?
[...]  (4,) 
```

可以在一条语句中使用多个`defer()`选项，以将多个列标记为延迟加载。

与`load_only()`一样，`defer()`选项也包括将延迟属性在访问时引发异常而不是惰性加载的能力。这在 使用 raiseload 防止延迟列加载 部分中有所说明。

### 使用 raiseload 防止延迟加载列

当使用 `load_only()` 或 `defer()` 加载器选项时，标记为延迟加载的对象属性在首次访问时具有默认行为，即在当前事务中发出 SELECT 语句以加载其值。通常需要阻止此加载的发生，并在访问属性时引发异常，表示不期望需要查询数据库以获取此列的需求。典型场景是使用已知需要用于操作进行的所有列加载对象，然后将其传递到视图层。应捕获视图层内发出的任何进一步的 SQL 操作，以便可以调整预先加载的操作以适应该额外的数据，而不是产生额外的惰性加载。

对于此用例，`defer()` 和 `load_only()` 选项包括一个布尔参数 `defer.raiseload`，当设置为 `True` 时，将导致受影响的属性在访问时引发异常。在下面的示例中，延迟列 `.cover_photo` 将禁止属性访问：

```py
>>> book = session.scalar(
...     select(Book).options(defer(Book.cover_photo, raiseload=True)).where(Book.id == 4)
... )
SELECT  book.id,  book.owner_id,  book.title,  book.summary
FROM  book
WHERE  book.id  =  ?
[...]  (4,)
>>> book.cover_photo
Traceback (most recent call last):
...
sqlalchemy.exc.InvalidRequestError: 'Book.cover_photo' is not available due to raiseload=True
```

当使用 `load_only()` 命名一组特定的非延迟加载列时，可以使用 `load_only.raiseload` 参数将 `raiseload` 行为应用于其余列，该参数将应用于所有延迟加载的属性：

```py
>>> session.expunge_all()
>>> book = session.scalar(
...     select(Book).options(load_only(Book.title, raiseload=True)).where(Book.id == 5)
... )
SELECT  book.id,  book.title
FROM  book
WHERE  book.id  =  ?
[...]  (5,)
>>> book.summary
Traceback (most recent call last):
...
sqlalchemy.exc.InvalidRequestError: 'Book.summary' is not available due to raiseload=True
```

注意

目前还不能混合使用 `load_only()` 和 `defer()` 选项，这两个选项指向同一个实体，在一个语句中改变某些属性的 `raiseload` 行为；目前这样做会产生未定义的属性加载行为。

另请参阅

`defer.raiseload` 特性是与关系对应的相同“raiseload”特性的列级版本。有关关系的“raiseload”，请参见防止不必要的惰性加载使用 raiseload 在本指南的关系加载技术部分。

## 配置映射上的列延迟

`defer()` 的功能作为映射列的默认行为可用，适用于不应在每次查询时无条件加载的列。要配置，请使用 `mapped_column.deferred` 参数。下面的示例说明了对 `Book` 应用默认列延迟加载的映射：

```py
>>> class Book(Base):
...     __tablename__ = "book"
...     id: Mapped[int] = mapped_column(primary_key=True)
...     owner_id: Mapped[int] = mapped_column(ForeignKey("user_account.id"))
...     title: Mapped[str]
...     summary: Mapped[str] = mapped_column(Text, deferred=True)
...     cover_photo: Mapped[bytes] = mapped_column(LargeBinary, deferred=True)
...
...     def __repr__(self) -> str:
...         return f"Book(id={self.id!r}, title={self.title!r})"
```

使用上述映射，对 `Book` 的查询将自动不包括 `summary` 和 `cover_photo` 列：

```py
>>> book = session.scalar(select(Book).where(Book.id == 2))
SELECT  book.id,  book.owner_id,  book.title
FROM  book
WHERE  book.id  =  ?
[...]  (2,) 
```

与所有延迟加载属性一样，当首次访问加载的对象上的延迟加载属性时，默认行为是它们将 延迟加载 它们的值：

```py
>>> img_data = book.cover_photo
SELECT  book.cover_photo  AS  book_cover_photo
FROM  book
WHERE  book.id  =  ?
[...]  (2,) 
```

与 `defer()` 和 `load_only()` 加载器选项一样，映射器级别的延迟还包括一个选项，即当语句中没有其他选项时，可以发生 `raiseload` 行为，而不是延迟加载。这允许某些列不会默认加载，并且也永远不会在语句中使用显式指令时延迟加载。请参阅 配置映射器级别的`raiseload`行为 部分，了解如何配置和使用此行为的背景信息。

### 使用 `deferred()` 来命令式映射，映射 SQL 表达式

`deferred()` 函数是早期的、更通用的“延迟列”映射指令，在引入 SQLAlchemy 的 `mapped_column()` 构造之前就存在。

`deferred()` 在配置 ORM 映射器时使用，接受任意的 SQL 表达式或 `Column` 对象。因此，它适用于非声明性 命令式映射，将其传递给 `map_imperatively.properties` 字典：

```py
from sqlalchemy import Blob
from sqlalchemy import Column
from sqlalchemy import ForeignKey
from sqlalchemy import Integer
from sqlalchemy import String
from sqlalchemy import Table
from sqlalchemy import Text
from sqlalchemy.orm import registry

mapper_registry = registry()

book_table = Table(
    "book",
    mapper_registry.metadata,
    Column("id", Integer, primary_key=True),
    Column("title", String(50)),
    Column("summary", Text),
    Column("cover_image", Blob),
)

class Book:
    pass

mapper_registry.map_imperatively(
    Book,
    book_table,
    properties={
        "summary": deferred(book_table.c.summary),
        "cover_image": deferred(book_table.c.cover_image),
    },
)
```

当映射的 SQL 表达式应该在延迟加载时，可以使用 `deferred()` 代替 `column_property()`：

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

使用 column_property - 在 SQL 表达式作为映射属性 部分

对声明性表列应用加载、持久性和映射选项 - 在使用声明性进行表配置章节中

### 使用`undefer()`来“急切地”加载延迟列

对于默认配置为延迟的映射上的列，`undefer()`选项将导致任何通常延迟的列都会在前端加载，也就是说，与映射的所有其他列一起加载。例如，我们可以对前面映射中标记为延迟的`Book.summary`列应用`undefer()`：

```py
>>> from sqlalchemy.orm import undefer
>>> book = session.scalar(select(Book).where(Book.id == 2).options(undefer(Book.summary)))
SELECT  book.id,  book.owner_id,  book.title,  book.summary
FROM  book
WHERE  book.id  =  ?
[...]  (2,) 
```

`Book.summary`列现在已经被急切加载，并且可以在不发出额外 SQL 的情况下访问：

```py
>>> print(book.summary)
another long summary
```

### 按组加载延迟列

通常，当一个列使用`mapped_column(deferred=True)`进行映射时，当在对象上访问延迟属性时，SQL 将被发出以仅加载该特定列，而不加载其他列，即使映射还有其他被标记为延迟的列也是如此。在延迟属性是应该一次性加载一组属性的情况下，而不是针对每个属性单独发出 SQL 时，可以使用`mapped_column.deferred_group`参数，它接受一个任意字符串，用于定义要取消延迟的列的通用组：

```py
>>> class Book(Base):
...     __tablename__ = "book"
...     id: Mapped[int] = mapped_column(primary_key=True)
...     owner_id: Mapped[int] = mapped_column(ForeignKey("user_account.id"))
...     title: Mapped[str]
...     summary: Mapped[str] = mapped_column(
...         Text, deferred=True, deferred_group="book_attrs"
...     )
...     cover_photo: Mapped[bytes] = mapped_column(
...         LargeBinary, deferred=True, deferred_group="book_attrs"
...     )
...
...     def __repr__(self) -> str:
...         return f"Book(id={self.id!r}, title={self.title!r})"
```

使用上述映射，访问`summary`或`cover_photo`将同时使用一个 SELECT 语句加载两个列：

```py
>>> book = session.scalar(select(Book).where(Book.id == 2))
SELECT  book.id,  book.owner_id,  book.title
FROM  book
WHERE  book.id  =  ?
[...]  (2,)
>>> img_data, summary = book.cover_photo, book.summary
SELECT  book.summary  AS  book_summary,  book.cover_photo  AS  book_cover_photo
FROM  book
WHERE  book.id  =  ?
[...]  (2,) 
```

### 使用`undefer_group()`按组取消延迟

如果延迟列配置为使用前一节中引入的`mapped_column.deferred_group`，则可以使用`undefer_group()`选项来急切加载整个组，传递要急切加载的组的字符串名称：

```py
>>> from sqlalchemy.orm import undefer_group
>>> book = session.scalar(
...     select(Book).where(Book.id == 2).options(undefer_group("book_attrs"))
... )
SELECT  book.id,  book.owner_id,  book.title,  book.summary,  book.cover_photo
FROM  book
WHERE  book.id  =  ?
[...]  (2,) 
```

`summary`和`cover_photo`都可以在不进行额外加载的情况下使用：

```py
>>> img_data, summary = book.cover_photo, book.summary
```

### 通配符上的取消延迟

大多数 ORM 加载器选项都接受通配符表达式，用`"*"`表示，表示该选项应用于所有相关属性。如果一个映射具有一系列延迟列，那么所有这些列都可以一次性进行取消延迟，而不需要使用组名，只需指定通配符即可：

```py
>>> book = session.scalar(select(Book).where(Book.id == 3).options(undefer("*")))
SELECT  book.id,  book.owner_id,  book.title,  book.summary,  book.cover_photo
FROM  book
WHERE  book.id  =  ?
[...]  (3,) 
```

### 配置映射级别的“raiseload”行为

首次引入的“raiseload”行为可用于 使用 raiseload 防止延迟列加载，还可以作为默认的映射器级行为应用，使用 `mapped_column.deferred_raiseload` 参数传递给 `mapped_column()`。使用此参数时，受影响的列将在所有情况下访问时引发异常，除非在查询时显式“未延迟”使用 `undefer()` 或 `load_only()`：

```py
>>> class Book(Base):
...     __tablename__ = "book"
...     id: Mapped[int] = mapped_column(primary_key=True)
...     owner_id: Mapped[int] = mapped_column(ForeignKey("user_account.id"))
...     title: Mapped[str]
...     summary: Mapped[str] = mapped_column(Text, deferred=True, deferred_raiseload=True)
...     cover_photo: Mapped[bytes] = mapped_column(
...         LargeBinary, deferred=True, deferred_raiseload=True
...     )
...
...     def __repr__(self) -> str:
...         return f"Book(id={self.id!r}, title={self.title!r})"
```

使用以上映射，`.summary` 和 `.cover_photo` 列默认情况下不可加载：

```py
>>> book = session.scalar(select(Book).where(Book.id == 2))
SELECT  book.id,  book.owner_id,  book.title
FROM  book
WHERE  book.id  =  ?
[...]  (2,)
>>> book.summary
Traceback (most recent call last):
...
sqlalchemy.exc.InvalidRequestError: 'Book.summary' is not available due to raiseload=True
```

只有在查询时覆盖它们的行为，通常使用 `undefer()` 或 `undefer_group()`，或者较少使用 `defer()`，属性才能被加载。下面的示例将 `undefer('*')` 应用于取消延迟所有属性，还使用了填充现有以刷新已加载对象的加载器选项：

```py
>>> book = session.scalar(
...     select(Book)
...     .where(Book.id == 2)
...     .options(undefer("*"))
...     .execution_options(populate_existing=True)
... )
SELECT  book.id,  book.owner_id,  book.title,  book.summary,  book.cover_photo
FROM  book
WHERE  book.id  =  ?
[...]  (2,)
>>> book.summary
'another long summary'
```  ### 使用 `deferred()` 进行命令式映射，映射的 SQL 表达式

`deferred()` 函数是早期的、更通用的“延迟列”映射指令，它在引入 `mapped_column()` 构造之前就存在于 SQLAlchemy 中。

在配置 ORM 映射器时使用 `deferred()`，它接受任意的 SQL 表达式或 `Column` 对象。因此，它适用于非声明式的命令式映射，可以将其传递给 `map_imperatively.properties` 字典：

```py
from sqlalchemy import Blob
from sqlalchemy import Column
from sqlalchemy import ForeignKey
from sqlalchemy import Integer
from sqlalchemy import String
from sqlalchemy import Table
from sqlalchemy import Text
from sqlalchemy.orm import registry

mapper_registry = registry()

book_table = Table(
    "book",
    mapper_registry.metadata,
    Column("id", Integer, primary_key=True),
    Column("title", String(50)),
    Column("summary", Text),
    Column("cover_image", Blob),
)

class Book:
    pass

mapper_registry.map_imperatively(
    Book,
    book_table,
    properties={
        "summary": deferred(book_table.c.summary),
        "cover_image": deferred(book_table.c.cover_image),
    },
)
```

当映射的 SQL 表达式应该以延迟方式加载时，也可以使用 `deferred()` 替代 `column_property()`：

```py
from sqlalchemy.orm import deferred

class User(Base):
    __tablename__ = "user"

    id: Mapped[int] = mapped_column(primary_key=True)
    firstname: Mapped[str] = mapped_column()
    lastname: Mapped[str] = mapped_column()
    fullname: Mapped[str] = deferred(firstname + " " + lastname)
```

请参阅

使用 column_property - 在 SQL 表达式作为映射属性 部分中

应用 Imperative 表列的加载、持久化和映射选项 - 在 声明式表配置 部分中

### 使用`undefer()`“急切”加载延迟列

使用默认延迟列配置的映射上的列，`undefer()`选项将导致通常延迟的任何列被解除延迟，即，与映射的所有其他列一起前端加载。例如，我们可以将`undefer()`应用于前一映射中指定为延迟的`Book.summary`列：

```py
>>> from sqlalchemy.orm import undefer
>>> book = session.scalar(select(Book).where(Book.id == 2).options(undefer(Book.summary)))
SELECT  book.id,  book.owner_id,  book.title,  book.summary
FROM  book
WHERE  book.id  =  ?
[...]  (2,) 
```

`Book.summary`列现在已经被急切加载，可以在不发出额外 SQL 的情况下访问：

```py
>>> print(book.summary)
another long summary
```

### 按组加载延迟列

通常，当列被映射为`mapped_column(deferred=True)`时，当在对象上访问延迟属性时，将发出 SQL 仅加载该特定列，而不加载其他列，即使映射还有其他列也被标记为延迟。在常见情况下，延迟属性是一组应该同时加载的属性的一部分时，而不是为每个属性单独发出 SQL，可以使用`mapped_column.deferred_group`参数，该参数接受一个任意字符串，该字符串将定义一个通用列组以解除延迟：

```py
>>> class Book(Base):
...     __tablename__ = "book"
...     id: Mapped[int] = mapped_column(primary_key=True)
...     owner_id: Mapped[int] = mapped_column(ForeignKey("user_account.id"))
...     title: Mapped[str]
...     summary: Mapped[str] = mapped_column(
...         Text, deferred=True, deferred_group="book_attrs"
...     )
...     cover_photo: Mapped[bytes] = mapped_column(
...         LargeBinary, deferred=True, deferred_group="book_attrs"
...     )
...
...     def __repr__(self) -> str:
...         return f"Book(id={self.id!r}, title={self.title!r})"
```

使用上述映射，访问`summary`或`cover_photo`将一次性使用一个 SELECT 语句加载两个列：

```py
>>> book = session.scalar(select(Book).where(Book.id == 2))
SELECT  book.id,  book.owner_id,  book.title
FROM  book
WHERE  book.id  =  ?
[...]  (2,)
>>> img_data, summary = book.cover_photo, book.summary
SELECT  book.summary  AS  book_summary,  book.cover_photo  AS  book_cover_photo
FROM  book
WHERE  book.id  =  ?
[...]  (2,) 
```

### 使用`undefer_group()`按组解除延迟

如果延迟列配置为`mapped_column.deferred_group`，如前一节介绍的，可以通过指定要急切加载的组的字符串名称来指示整个组的加载：

```py
>>> from sqlalchemy.orm import undefer_group
>>> book = session.scalar(
...     select(Book).where(Book.id == 2).options(undefer_group("book_attrs"))
... )
SELECT  book.id,  book.owner_id,  book.title,  book.summary,  book.cover_photo
FROM  book
WHERE  book.id  =  ?
[...]  (2,) 
```

`summary`和`cover_photo`都可用，无需额外加载：

```py
>>> img_data, summary = book.cover_photo, book.summary
```

### 使用通配符解除延迟加载

大多数 ORM 加载器选项都接受通配符表达式，由 `"*"` 表示，表示该选项应用于所有相关属性。如果映射具有一系列延迟列，则可以通过指定通配符一次性解除所有这些列的延迟，而无需使用组名：

```py
>>> book = session.scalar(select(Book).where(Book.id == 3).options(undefer("*")))
SELECT  book.id,  book.owner_id,  book.title,  book.summary,  book.cover_photo
FROM  book
WHERE  book.id  =  ?
[...]  (3,) 
```

### 配置映射器级别的“raiseload”行为

“raiseload” 行为最初是在使用 raiseload 防止延迟列加载中介绍的，也可以作为默认的映射器级行为应用，使用 `mapped_column.deferred_raiseload` 参数的 `mapped_column()`。当使用此参数时，受影响的列将在所有情况下在访问时引发异常，除非在查询时显式地使用 `undefer()` 或 `load_only()` 进行“取消延迟”：

```py
>>> class Book(Base):
...     __tablename__ = "book"
...     id: Mapped[int] = mapped_column(primary_key=True)
...     owner_id: Mapped[int] = mapped_column(ForeignKey("user_account.id"))
...     title: Mapped[str]
...     summary: Mapped[str] = mapped_column(Text, deferred=True, deferred_raiseload=True)
...     cover_photo: Mapped[bytes] = mapped_column(
...         LargeBinary, deferred=True, deferred_raiseload=True
...     )
...
...     def __repr__(self) -> str:
...         return f"Book(id={self.id!r}, title={self.title!r})"
```

使用上述映射，`.summary` 和 `.cover_photo` 列默认情况下不可加载：

```py
>>> book = session.scalar(select(Book).where(Book.id == 2))
SELECT  book.id,  book.owner_id,  book.title
FROM  book
WHERE  book.id  =  ?
[...]  (2,)
>>> book.summary
Traceback (most recent call last):
...
sqlalchemy.exc.InvalidRequestError: 'Book.summary' is not available due to raiseload=True
```

只有通过在查询时覆盖它们的行为，通常使用 `undefer()` 或 `undefer_group()`，或者较少使用 `defer()`，属性才能被加载。下面的示例将 `undefer('*')` 应用于取消延迟加载所有属性，同时还利用填充现有对象来刷新已加载对象的加载器选项：

```py
>>> book = session.scalar(
...     select(Book)
...     .where(Book.id == 2)
...     .options(undefer("*"))
...     .execution_options(populate_existing=True)
... )
SELECT  book.id,  book.owner_id,  book.title,  book.summary,  book.cover_photo
FROM  book
WHERE  book.id  =  ?
[...]  (2,)
>>> book.summary
'another long summary'
```

## 加载任意 SQL 表达式到对象上

如在选择 ORM 实体和属性和其他地方讨论的，`select()` 构造可以用于在结果集中加载任意 SQL 表达式。例如，如果我们想要发出一个查询，加载 `User` 对象，但还包括每个 `User` 拥有多少书籍的计数，我们可以使用 `func.count(Book.id)` 来向查询中添加一个“计数”列，该查询包括与 `Book` 的 JOIN 以及按所有者 id 分组。这将产生包含两个条目的 `Row` 对象，一个是 `User`，另一个是 `func.count(Book.id)`：

```py
>>> from sqlalchemy import func
>>> stmt = select(User, func.count(Book.id)).join_from(User, Book).group_by(Book.owner_id)
>>> for user, book_count in session.execute(stmt):
...     print(f"Username: {user.name}  Number of books: {book_count}")
SELECT  user_account.id,  user_account.name,  user_account.fullname,
count(book.id)  AS  count_1
FROM  user_account  JOIN  book  ON  user_account.id  =  book.owner_id
GROUP  BY  book.owner_id
[...]  ()
Username: spongebob  Number of books: 3
Username: sandy  Number of books: 3
```

在上面的例子中，`User` 实体和 “书籍数量” SQL 表达式是分开返回的。然而，一个常见的用例是生成一个查询，该查询仅返回 `User` 对象，例如可以使用 `Session.scalars()` 进行迭代，其中 `func.count(Book.id)` SQL 表达式的结果被 *动态* 应用于每个 `User` 实体。最终结果将类似于使用 `column_property()` 将任意 SQL 表达式映射到类的情况，只不过 SQL 表达式可以在查询时修改。对于这种用例，SQLAlchemy 提供了 `with_expression()` 加载器选项，当与映射器级别的 `query_expression()` 指令结合使用时，可能会产生此结果。

要将 `with_expression()` 应用于查询，映射类必须预先使用 `query_expression()` 指令配置好一个 ORM 映射属性；此指令将在映射类上生成一个适合接收查询时 SQL 表达式的属性。下面我们向 `User` 添加一个新属性 `User.book_count`。这个 ORM 映射属性是只读的，没有默认值；在加载的实例上访问它通常会返回 `None`：

```py
>>> from sqlalchemy.orm import query_expression
>>> class User(Base):
...     __tablename__ = "user_account"
...     id: Mapped[int] = mapped_column(primary_key=True)
...     name: Mapped[str]
...     fullname: Mapped[Optional[str]]
...     book_count: Mapped[int] = query_expression()
...
...     def __repr__(self) -> str:
...         return f"User(id={self.id!r}, name={self.name!r}, fullname={self.fullname!r})"
```

在我们的映射中配置了 `User.book_count` 属性后，我们可以使用 `with_expression()` 加载器选项从 SQL 表达式中填充数据，以便在加载每个 `User` 对象时应用自定义 SQL 表达式：

```py
>>> from sqlalchemy.orm import with_expression
>>> stmt = (
...     select(User)
...     .join_from(User, Book)
...     .group_by(Book.owner_id)
...     .options(with_expression(User.book_count, func.count(Book.id)))
... )
>>> for user in session.scalars(stmt):
...     print(f"Username: {user.name}  Number of books: {user.book_count}")
SELECT  count(book.id)  AS  count_1,  user_account.id,  user_account.name,
user_account.fullname
FROM  user_account  JOIN  book  ON  user_account.id  =  book.owner_id
GROUP  BY  book.owner_id
[...]  ()
Username: spongebob  Number of books: 3
Username: sandy  Number of books: 3
```

在上面的例子中，我们将我们的 `func.count(Book.id)` 表达式从 `select()` 构造函数的 columns 参数中移出，并将其放入 `with_expression()` 加载器选项中。ORM 然后将其视为一个特殊的列加载选项，该选项动态应用于语句。

`query_expression()` 映射有以下注意事项：

+   在一个对象上，如果没有使用 `with_expression()` 来填充属性，对象实例上的属性将具有值 `None`，除非在映射上设置了 `query_expression.default_expr` 参数为默认 SQL 表达式。

+   `with_expression()` 值**不会填充已加载的对象**，除非使用 Populate Existing。如下示例**不会起作用**，因为 `A` 对象已经加载：

    ```py
    # load the first A
    obj = session.scalars(select(A).order_by(A.id)).first()

    # load the same A with an option; expression will **not** be applied
    # to the already-loaded object
    obj = session.scalars(select(A).options(with_expression(A.expr, some_expr))).first()
    ```

    要确保属性在现有对象上重新加载，请使用 Populate Existing 执行选项以确保重新填充所有列：

    ```py
    obj = session.scalars(
        select(A)
        .options(with_expression(A.expr, some_expr))
        .execution_options(populate_existing=True)
    ).first()
    ```

+   当对象过期时，`with_expression()` SQL 表达式**会丢失**。一旦对象过期，无论是通过 `Session.expire()` 还是通过 `Session.commit()` 的 expire_on_commit 行为，SQL 表达式及其值将不再与属性关联，并且在后续访问时将返回 `None`。

+   `with_expression()` 作为对象加载选项，只对查询的最外层部分生效，并且仅适用于对完整实体进行的查询，而不适用于子查询中的任意列选择或复合语句（如 UNION）的元素。请参阅下一节使用 `with_expression()` 与 UNIONs、其他子查询中的示例。

+   映射属性**无法**应用于查询的其他部分，例如 WHERE 子句、ORDER BY 子句，并利用临时表达式；也就是说，以下方式行不通：

    ```py
    # can't refer to A.expr elsewhere in the query
    stmt = (
        select(A)
        .options(with_expression(A.expr, A.x + A.y))
        .filter(A.expr > 5)
        .order_by(A.expr)
    )
    ```

    在上述的 WHERE 子句和 ORDER BY 子句中，`A.expr` 表达式将会解析为 NULL。要在整个查询中使用该表达式，请将其赋值给一个变量并使用它：

    ```py
    # assign desired expression up front, then refer to that in
    # the query
    a_expr = A.x + A.y
    stmt = (
        select(A)
        .options(with_expression(A.expr, a_expr))
        .filter(a_expr > 5)
        .order_by(a_expr)
    )
    ```

另请参见

`with_expression()` 选项是一种特殊选项，用于在查询时动态地将 SQL 表达式应用于映射类。对于在映射器上配置的普通固定 SQL 表达式，请参阅作为映射属性的 SQL 表达式部分。

### 使用 `with_expression()` 与 UNIONs、其他子查询

`with_expression()` 构造是一个 ORM 加载器选项，因此只能应用于用于加载特定 ORM 实体的 SELECT 语句的最外层级别。如果在后续用作子查询或复合语句（如 UNION）中使用，它将不起作用。

为了在子查询中使用任意的 SQL 表达式，应该使用正常的 Core 风格添加表达式的方法。要将子查询派生的表达式组装到 ORM 实体的`query_expression()`属性上，需要在 ORM 对象加载的顶层使用`with_expression()`，引用子查询中的 SQL 表达式。

在下面的示例中，针对 ORM 实体 `A` 使用了两个`select()` 构造，其中包含一个标记为 `expr` 的额外 SQL 表达式，并使用`union_all()` 进行组合。然后，在最顶层，从这个 UNION 中选择了 `A` 实体，使用了在从 UNIONs 和其他集合操作中选择实体中描述的查询技术，添加了一个使用`with_expression()`的选项，以将这个 SQL 表达式提取到新加载的 `A` 实例中：

```py
>>> from sqlalchemy import union_all
>>> s1 = (
...     select(User, func.count(Book.id).label("book_count"))
...     .join_from(User, Book)
...     .where(User.name == "spongebob")
... )
>>> s2 = (
...     select(User, func.count(Book.id).label("book_count"))
...     .join_from(User, Book)
...     .where(User.name == "sandy")
... )
>>> union_stmt = union_all(s1, s2)
>>> orm_stmt = (
...     select(User)
...     .from_statement(union_stmt)
...     .options(with_expression(User.book_count, union_stmt.selected_columns.book_count))
... )
>>> for user in session.scalars(orm_stmt):
...     print(f"Username: {user.name}  Number of books: {user.book_count}")
SELECT  user_account.id,  user_account.name,  user_account.fullname,  count(book.id)  AS  book_count
FROM  user_account  JOIN  book  ON  user_account.id  =  book.owner_id
WHERE  user_account.name  =  ?
UNION  ALL
SELECT  user_account.id,  user_account.name,  user_account.fullname,  count(book.id)  AS  book_count
FROM  user_account  JOIN  book  ON  user_account.id  =  book.owner_id
WHERE  user_account.name  =  ?
[...]  ('spongebob',  'sandy')
Username: spongebob  Number of books: 3
Username: sandy  Number of books: 3
```  ### 使用 `with_expression()` 与 UNIONs，其他子查询

`with_expression()` 构造是一个 ORM 加载器选项，因此只能应用于要加载特定 ORM 实体的 SELECT 语句的最外层级别。如果在将用作子查询或作为联合等复合语句中的元素的`select()`内部使用，则不会产生任何效果。

为了在子查询中使用任意的 SQL 表达式，应该使用正常的 Core 风格添加表达式的方法。要将子查询派生的表达式组装到 ORM 实体的`query_expression()`属性上，需要在 ORM 对象加载的顶层使用`with_expression()`，引用子查询中的 SQL 表达式。

在下面的示例中，使用两个`select()`构造针对 ORM 实体 `A`，并在`expr`中标记了一个额外的 SQL 表达式，并使用`union_all()`将它们组合起来。然后，在最顶层，使用查询技术描述的 `with_expression()`从这个 UNION 中选择 `A` 实体，以将此 SQL 表达式提取到新加载的 `A` 实例上：

```py
>>> from sqlalchemy import union_all
>>> s1 = (
...     select(User, func.count(Book.id).label("book_count"))
...     .join_from(User, Book)
...     .where(User.name == "spongebob")
... )
>>> s2 = (
...     select(User, func.count(Book.id).label("book_count"))
...     .join_from(User, Book)
...     .where(User.name == "sandy")
... )
>>> union_stmt = union_all(s1, s2)
>>> orm_stmt = (
...     select(User)
...     .from_statement(union_stmt)
...     .options(with_expression(User.book_count, union_stmt.selected_columns.book_count))
... )
>>> for user in session.scalars(orm_stmt):
...     print(f"Username: {user.name}  Number of books: {user.book_count}")
SELECT  user_account.id,  user_account.name,  user_account.fullname,  count(book.id)  AS  book_count
FROM  user_account  JOIN  book  ON  user_account.id  =  book.owner_id
WHERE  user_account.name  =  ?
UNION  ALL
SELECT  user_account.id,  user_account.name,  user_account.fullname,  count(book.id)  AS  book_count
FROM  user_account  JOIN  book  ON  user_account.id  =  book.owner_id
WHERE  user_account.name  =  ?
[...]  ('spongebob',  'sandy')
Username: spongebob  Number of books: 3
Username: sandy  Number of books: 3
```

## 列加载 API

| 对象名称 | 描述 |
| --- | --- |
| defer(key, *addl_attrs, [raiseload]) | 指示给定的面向列的属性应该延迟加载，例如在访问之前不加载。 |
| deferred(column, *additional_columns, [group, raiseload, comparator_factory, init, repr, default, default_factory, compare, kw_only, active_history, expire_on_flush, info, doc]) | 表示默认情况下不加载的基于列的映射属性。 |
| load_only(*attrs, [raiseload]) | 表示对于特定实体，只应加载给定列表的基于列的属性名称；所有其他属性都将被延迟加载。 |
| query_expression([default_expr], *, [repr, compare, expire_on_flush, info, doc]) | 表示从查询时 SQL 表达式填充的属性。 |
| undefer(key, *addl_attrs) | 指示给定的面向列的属性应该取消延迟加载，例如在整个实体的 SELECT 语句中指定。 |
| undefer_group(name) | 表示给定延迟组名内的列应该取消延迟加载。 |
| with_expression(key, expression) | 将特定的 SQL 表达式应用于“延迟表达式”属性。 |

```py
function sqlalchemy.orm.defer(key: Literal['*'] | QueryableAttribute[Any], *addl_attrs: Literal['*'] | QueryableAttribute[Any], raiseload: bool = False) → _AbstractLoad
```

指示给定的面向列的属性应该延迟加载，例如在访问之前不加载。

此函数是`Load`接口的一部分，支持方法链和独立操作。

例如：

```py
from sqlalchemy.orm import defer

session.query(MyClass).options(
 defer(MyClass.attribute_one),
 defer(MyClass.attribute_two)
)
```

要指定相关类上属性的延迟加载，可以逐个指定路径，沿着链指定每个链接的加载样式。要保持链接的加载样式不变，请使用`defaultload()`：

```py
session.query(MyClass).options(
 defaultload(MyClass.someattr).defer(RelatedClass.some_column)
)
```

可以使用`Load.options()`一次捆绑多个与关系相关的延迟选项。

```py
select(MyClass).options(
 defaultload(MyClass.someattr).options(
 defer(RelatedClass.some_column),
 defer(RelatedClass.some_other_column),
 defer(RelatedClass.another_column)
 )
)
```

参数：

+   `key` – 要延迟加载的属性。

+   `raiseload` – 当访问延迟属性时引发`InvalidRequestError`而不是惰性加载值。用于防止发出不必要的 SQL。

版本 1.4 中的新功能。

另请参阅

限制使用列延迟加载 - 在 ORM 查询指南中

`load_only()`

`undefer()`

```py
function sqlalchemy.orm.deferred(column: _ORMColumnExprArgument[_T], *additional_columns: _ORMColumnExprArgument[Any], group: str | None = None, raiseload: bool = False, comparator_factory: Type[PropComparator[_T]] | None = None, init: _NoArg | bool = _NoArg.NO_ARG, repr: _NoArg | bool = _NoArg.NO_ARG, default: Any | None = _NoArg.NO_ARG, default_factory: _NoArg | Callable[[], _T] = _NoArg.NO_ARG, compare: _NoArg | bool = _NoArg.NO_ARG, kw_only: _NoArg | bool = _NoArg.NO_ARG, active_history: bool = False, expire_on_flush: bool = True, info: _InfoType | None = None, doc: str | None = None) → MappedSQLExpression[_T]
```

指示一个基于列的映射属性，默认情况下不会加载，除非访问。

在使用`mapped_column()`时，通过使用`mapped_column.deferred`参数，提供了与`deferred()`构造相同的功能。

参数：

+   `*columns` – 要映射的列。通常是单个`Column`对象，但为了支持在同一属性下映射多个列，也支持集合。

+   `raiseload` –

    boolean，如果为 True，则表示在执行加载操作时应引发异常。

    版本 1.4 中的新功能。

其他参数与`column_property()`相同。

另请参阅

使用 deferred()为命令式映射器、映射的 SQL 表达式

```py
function sqlalchemy.orm.query_expression(default_expr: _ORMColumnExprArgument[_T] = <sqlalchemy.sql.elements.Null object>, *, repr: Union[_NoArg, bool] = _NoArg.NO_ARG, compare: Union[_NoArg, bool] = _NoArg.NO_ARG, expire_on_flush: bool = True, info: Optional[_InfoType] = None, doc: Optional[str] = None) → MappedSQLExpression[_T]
```

指示从查询时 SQL 表达式填充的属性。

参数：

**default_expr** – 可选的 SQL 表达式对象，如果未使用`with_expression()`分配，则将在所有情况下使用。

版本 1.2 中的新功能。

另请参阅

将任意 SQL 表达式加载到对象上 - 背景和使用示例

```py
function sqlalchemy.orm.load_only(*attrs: Literal['*'] | QueryableAttribute[Any], raiseload: bool = False) → _AbstractLoad
```

指示对于特定实体，只应加载给定的列名列表的基于列的属性；所有其他属性将被延迟加载。

此函数是`Load`接口的一部分，支持方法链接和独立操作。

示例 - 给定一个类`User`，仅加载`name`和`fullname`属性：

```py
session.query(User).options(load_only(User.name, User.fullname))
```

示例 - 给定一个关系`User.addresses -> Address`，为`User.addresses`集合指定子查询加载，但在每个`Address`对象上仅加载`email_address`属性：

```py
session.query(User).options(
 subqueryload(User.addresses).load_only(Address.email_address)
)
```

对于具有多个实体的语句，可以使用`Load`构造函数来特定引用主实体：

```py
stmt = (
 select(User, Address)
 .join(User.addresses)
 .options(
 Load(User).load_only(User.name, User.fullname),
 Load(Address).load_only(Address.email_address),
 )
)
```

与 populate_existing 执行选项一起使用时，只会刷新列出的属性。

参数：

+   `*attrs` – 需要加载的属性，其他所有属性都将被延迟加载。

+   `raiseload` –

    当访问延迟属性时引发 `InvalidRequestError` 而不是懒加载值。用于防止不必要的 SQL 发出。

    2.0 版本中新增。

参见

限制加载哪些列与列延迟 - 在 ORM 查询指南 中

参数：

+   `*attrs` – 需要加载的属性，其他所有属性都将被延迟加载。

+   `raiseload` –

    当访问延迟属性时引发 `InvalidRequestError` 而不是懒加载值。用于防止不必要的 SQL 发出。

    2.0 版本中新增。

```py
function sqlalchemy.orm.undefer(key: Literal['*'] | QueryableAttribute[Any], *addl_attrs: Literal['*'] | QueryableAttribute[Any]) → _AbstractLoad
```

表示给定的面向列的属性应该取消延迟加载，例如在整个实体的 SELECT 语句中指定。

通常在映射上设置的列作为 `deferred()` 属性。

此函数是 `Load` 接口的一部分，支持方法链接和独立操作。

示例：

```py
# undefer two columns
session.query(MyClass).options(
 undefer(MyClass.col1), undefer(MyClass.col2)
)

# undefer all columns specific to a single class using Load + *
session.query(MyClass, MyOtherClass).options(
 Load(MyClass).undefer("*")
)

# undefer a column on a related object
select(MyClass).options(
 defaultload(MyClass.items).undefer(MyClass.text)
)
```

参数：

**key** – 需要取消延迟加载的属性。

参见

限制加载哪些列与列延迟 - 在 ORM 查询指南 中

`defer()`

`undefer_group()`

```py
function sqlalchemy.orm.undefer_group(name: str) → _AbstractLoad
```

表示给定延迟组名称内的列应取消延迟加载。

正在取消延迟加载的列设置在映射上作为`deferred()`属性，并包括一个“组”名称。

例如：

```py
session.query(MyClass).options(undefer_group("large_attrs"))
```

要在相关实体上取消一组属性的延迟加载，可以使用关系加载器选项拼写出路径，例如 `defaultload()`:

```py
select(MyClass).options(
 defaultload("someattr").undefer_group("large_attrs")
)
```

参见

限制加载哪些列与列延迟 - 在 ORM 查询指南 中

`defer()`

`undefer()`

```py
function sqlalchemy.orm.with_expression(key: _AttrType, expression: _ColumnExpressionArgument[Any]) → _AbstractLoad
```

对“延迟表达式”属性应用临时 SQL 表达式。

此选项与指示应该成为临时 SQL 表达式目标的属性的 `query_expression()` 映射器级构造一起使用。

例如：

```py
stmt = select(SomeClass).options(
 with_expression(SomeClass.x_y_expr, SomeClass.x + SomeClass.y)
)
```

1.2 版本中新增。

参数：

+   `key` – 需要填充的属性

+   `expr` – 应用于属性的 SQL 表达式。

参见

将任意的 SQL 表达式加载到对象上 - 背景和使用示例
