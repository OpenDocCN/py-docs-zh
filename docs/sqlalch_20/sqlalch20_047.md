# 用于查询的 ORM API 功能

> 原文：[`docs.sqlalchemy.org/en/20/orm/queryguide/api.html`](https://docs.sqlalchemy.org/en/20/orm/queryguide/api.html)

## ORM 加载选项

加载选项是一种对象，当传递给`Select.options()`方法的`Select`对象或类似的 SQL 构造时，会影响列和关系属性的加载。大多数加载选项都来自于`Load`层次结构。有关使用加载选项的完整概述，请参阅下面的链接部分。

另请参阅

+   列加载选项 - 详细说明了影响列和 SQL 表达式映射属性加载方式的映射器和加载选项

+   关系加载技术 - 详细说明了影响`relationship()`映射属性加载方式的关系和加载选项

## ORM 执行选项

ORM 级别的执行选项是可以通过`Session.execute.execution_options`参数关联到语句执行的关键字选项，该参数是由`Session`方法（例如`Session.execute()`和`Session.scalars()`）接受的字典参数，或者直接使用`Executable.execution_options()`方法将其与要调用的语句直接关联起来，该方法接受它们作为任意关键字参数。

ORM 级别的选项与 Core 级别的执行选项有所不同，Core 级别的执行选项记录在`Connection.execution_options()`。重要的是要注意，下面讨论的 ORM 选项与 Core 级别的方法`Connection.execution_options()`或`Engine.execution_options()`**不**兼容；即使`Engine`或`Connection`与正在使用的`Session`关联，这些选项也会在此级别被忽略。

在这个部分中，将展示`Executable.execution_options()`方法的样式示例。

### 刷新现有对象

`populate_existing`执行选项确保对于加载的所有行，`Session`中对应的实例将完全被刷新 - 擦除对象中的任何现有数据（包括未决的更改），并用从结果加载的数据替换。

示例用法如下：

```py
>>> stmt = select(User).execution_options(populate_existing=True)
>>> result = session.execute(stmt)
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
... 
```

通常，ORM 对象只加载一次，如果它们在后续结果行中与主键匹配，则不会将该行应用于对象。这既是为了保留对象上未提交的更改，也是为了避免刷新已经存在的数据的开销和复杂性。`Session`假设一个高度隔离的事务的默认工作模型，并且在事务中预期的数据发生变化程度超出正在进行的本地更改时，这些用例将使用显式步骤来处理，例如此方法。

使用`populate_existing`，可以刷新与查询匹配的任何一组对象，并且还可以控制关系加载器选项。例如，要同时刷新一个实例和其相关对象集：

```py
stmt = (
    select(User)
    .where(User.name.in_(names))
    .execution_options(populate_existing=True)
    .options(selectinload(User.addresses))
)
# will refresh all matching User objects as well as the related
# Address objects
users = session.execute(stmt).scalars().all()
```

`populate_existing`的另一个用例是支持各种属性加载功能，这些功能可以根据每个查询的情况改变如何加载属性。适用于此选项的选项包括：

+   `with_expression()`选项

+   `PropComparator.and_()` 方法可以修改加载器策略加载的内容

+   `contains_eager()` 选项

+   `with_loader_criteria()` 选项

+   `load_only()` 选项来选择要刷新的属性

`populate_existing` 执行选项相当于 1.x 风格 ORM 查询中的 `Query.populate_existing()` 方法。

请参见

我使用 Session 重新加载数据，但它没有看到我在其他地方提交的更改 - 在 常见问题解答

刷新 / 过期 - 在 ORM `Session` 文档中 ### 自动刷新

当作为 `False` 传递时，此选项将导致 `Session` 不调用 “自动刷新” 步骤。这相当于使用 `Session.no_autoflush` 上下文管理器禁用自动刷新：

```py
>>> stmt = select(User).execution_options(autoflush=False)
>>> session.execute(stmt)
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
... 
```

此选项也适用于启用 ORM 的 `Update` 和 `Delete` 查询。

`autoflush` 执行选项相当于 1.x 风格 ORM 查询中的 `Query.autoflush()` 方法。

请参见

刷新  ### 使用 Yield Per 获取大型结果集

`yield_per` 执行选项是一个整数值，它将导致 `Result` 一次只缓冲有限数量的行和/或 ORM 对象，然后才将数据提供给客户端。

通常，ORM 会立即获取**所有**行，为每个行构造 ORM 对象，并将这些对象组装到单个缓冲区中，然后将此缓冲区作为要返回的行的来源传递给`Result`对象。此行为的基本原理是允许对诸如联接急加载、结果唯一化以及依赖于标识映射为每个对象在结果集中被提取时保持一致状态的结果处理逻辑等功能的正确行为。

`yield_per`选项的目的是更改此行为，以使 ORM 结果集针对迭代非常大的结果集（例如> 10K 行）进行了优化，其中用户已确定上述模式不适用。当使用`yield_per`时，ORM 将将 ORM 结果批处理到子集合中，并在`Result`对象被迭代时逐个从每个子集合中产生行，以便 Python 解释器无需声明非常大的内存区域，这既费时又导致内存使用过多。该选项影响数据库游标的使用方式以及 ORM 构造行和对象以传递给`Result`的方式。

提示

由上可知，`Result`必须以可迭代的方式被消耗，即使用迭代，如`for row in result`或使用部分行方法，如`Result.fetchmany()`或`Result.partitions()`。调用`Result.all()`将失去使用`yield_per`的目的。

使用`yield_per`相当于同时使用`Connection.execution_options.stream_results`执行选项，该选项选择在支持的情况下由后端使用服务器端游标，并且返回的`Result`对象上的`Result.yield_per()`方法，该方法建立了要提取的行的固定大小以及一次构造的 ORM 对象的相应限制。

提示

现在`yield_per`也作为 Core 执行选项可用，详细描述在使用服务器端游标（又名流式结果）。本节详细介绍了将`yield_per`作为 ORM `Session`的执行选项的用法。该选项在两种情况下的行为尽可能相似。

当与 ORM 一起使用时，`yield_per`必须通过给定语句上的`Executable.execution_options()`方法或通过将其传递给`Session.execute()`的`Session.execute.execution_options`参数来建立，或者通过其他类似的`Session`方法，例如`Session.scalars()`。下面是获取 ORM 对象的典型用法：

```py
>>> stmt = select(User).execution_options(yield_per=10)
>>> for user_obj in session.scalars(stmt):
...     print(user_obj)
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
[...]  ()
User(id=1, name='spongebob', fullname='Spongebob Squarepants')
User(id=2, name='sandy', fullname='Sandy Cheeks')
...
>>> # ... rows continue ...
```

上述代码等同于下面的示例，该示例使用了`Connection.execution_options.stream_results`和`Connection.execution_options.max_row_buffer` Core 级别执行选项，以及`Result.yield_per()`方法的使用`Result`：

```py
# equivalent code
>>> stmt = select(User).execution_options(stream_results=True, max_row_buffer=10)
>>> for user_obj in session.scalars(stmt).yield_per(10):
...     print(user_obj)
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
[...]  ()
User(id=1, name='spongebob', fullname='Spongebob Squarepants')
User(id=2, name='sandy', fullname='Sandy Cheeks')
...
>>> # ... rows continue ...
```

`yield_per`也常与`Result.partitions()`方法结合使用，该方法将对分组分区中的行进行迭代。每个分区的大小默认为传递给`yield_per`的整数值，如下例所示：

```py
>>> stmt = select(User).execution_options(yield_per=10)
>>> for partition in session.scalars(stmt).partitions():
...     for user_obj in partition:
...         print(user_obj)
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
[...]  ()
User(id=1, name='spongebob', fullname='Spongebob Squarepants')
User(id=2, name='sandy', fullname='Sandy Cheeks')
...
>>> # ... rows continue ...
```

当使用集合时，`yield_per`执行选项**与**“子查询”急加载加载或“连接”急加载**不兼容**。如果数据库驱动程序支持多个独立游标，则它可能与“选择内”急加载兼容。

此外，`yield_per`执行选项与`Result.unique()`方法不兼容；由于此方法依赖于为所有行存储完整的标识集，因此它必然会破坏使用`yield_per`的目的，即处理任意数量的行。

自 1.4.6 版本更改：当从`Result`对象中获取 ORM 行时，该对象使用`Result.unique()`过滤器，并同时使用`yield_per`执行选项时，会引发异常。

当在 1.x 风格 ORM 使用遗留`Query`对象时，`Query.yield_per()`方法的结果与`yield_per`执行选项相同。

另请参阅

使用服务器端游标（又称流式结果） ### 身份令牌

深层魔法

此选项是一个高级功能，主要用于与水平分片扩展一起使用。对于从不同“分片”或分区加载具有相同主键的对象的典型情况，请首先考虑每个分片使用单独的`Session`对象。

“身份令牌”是可以与新加载对象的标识键相关联的任意值。此元素首先存在于支持按行“分片”的扩展中，其中对象可以从特定数据库表的任意数量的副本加载，尽管这些副本具有重叠的主键值。 “身份令牌”的主要消费者是水平分片扩展，它提供了一个在特定数据库表的多个“分片”之间持久化对象的通用框架。

`identity_token`执行选项可以在每个查询的基础上直接影响此令牌。直接使用它，可以为`Session`填充具有相同主键和源表但具有不同“标识”的对象的多个实例。

其中一个示例是使用翻译模式名称功能，该功能可以影响查询范围内的模式选择，从具有相同名称的表中填充`Session`对象。给定一个映射如下：

```py
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column

class Base(DeclarativeBase):
    pass

class MyTable(Base):
    __tablename__ = "my_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
```

上述类的默认“模式”名称是 `None`，这意味着 SQL 语句中不会写入模式限定符。但是，如果我们使用 `Connection.execution_options.schema_translate_map`，将 `None` 映射到替代模式，我们可以将 `MyTable` 的实例放入两个不同的模式中：

```py
engine = create_engine(
    "postgresql+psycopg://scott:tiger@localhost/test",
)

with Session(
    engine.execution_options(schema_translate_map={None: "test_schema"})
) as sess:
    sess.add(MyTable(name="this is schema one"))
    sess.commit()

with Session(
    engine.execution_options(schema_translate_map={None: "test_schema_2"})
) as sess:
    sess.add(MyTable(name="this is schema two"))
    sess.commit()
```

上述两个块分别创建了一个与不同模式映射关联的 `Session` 对象，并且 `MyTable` 的一个实例被持久化到了 `test_schema.my_table` 和 `test_schema_2.my_table` 中。

上述 `Session` 对象是独立的。如果我们想要在一个事务中持久化这两个对象，我们需要使用 水平分片 扩展来实现这一点。

但是，我们可以通过一个会话来说明对这些对象的查询，如下所示：

```py
with Session(engine) as sess:
    obj1 = sess.scalar(
        select(MyTable)
        .where(MyTable.id == 1)
        .execution_options(
            schema_translate_map={None: "test_schema"},
            identity_token="test_schema",
        )
    )
    obj2 = sess.scalar(
        select(MyTable)
        .where(MyTable.id == 1)
        .execution_options(
            schema_translate_map={None: "test_schema_2"},
            identity_token="test_schema_2",
        )
    )
```

`obj1` 和 `obj2` 都彼此不同。但是，它们都引用了 `MyTable` 类的主键 id 1，但是是不同的。这就是 `identity_token` 发挥作用的方式，我们可以在检查每个对象时看到，其中我们查看 `InstanceState.key` 来查看两个不同的标识令牌：

```py
>>> from sqlalchemy import inspect
>>> inspect(obj1).key
(<class '__main__.MyTable'>, (1,), 'test_schema')
>>> inspect(obj2).key
(<class '__main__.MyTable'>, (1,), 'test_schema_2')
```

当使用 水平分片 扩展时，上述逻辑会自动发生。

版本 2.0.0rc1 中的新功能：- 添加了 `identity_token` ORM 级别的执行选项。

另请参阅

水平分片 - 在 ORM 示例 部分。请参阅脚本 `separate_schema_translates.py`，了解如何使用完整的分片 API 进行上述用例的演示。

#### 检查来自启用 ORM 的 SELECT 和 DML 语句的实体和列

`select()` 构造，以及 `insert()`、`update()` 和 `delete()` 构造（对于后者的 DML 构造，在 SQLAlchemy 1.4.33 中），都支持检查这些语句所针对的实体，以及将在结果集中返回的列和数据类型。

对于 `Select` 对象，此信息可从 `Select.column_descriptions` 属性获取。此属性的操作方式与传统的 `Query.column_descriptions` 属性相同。返回的格式是一个字典列表：

```py
>>> from pprint import pprint
>>> user_alias = aliased(User, name="user2")
>>> stmt = select(User, User.id, user_alias)
>>> pprint(stmt.column_descriptions)
[{'aliased': False,
 'entity': <class 'User'>,
 'expr': <class 'User'>,
 'name': 'User',
 'type': <class 'User'>},
 {'aliased': False,
 'entity': <class 'User'>,
 'expr': <....InstrumentedAttribute object at ...>,
 'name': 'id',
 'type': Integer()},
 {'aliased': True,
 'entity': <AliasedClass ...; User>,
 'expr': <AliasedClass ...; User>,
 'name': 'user2',
 'type': <class 'User'>}]
```

当与非 ORM 对象一起使用 `Select.column_descriptions` 时，诸如普通 `Table` 或 `Column` 对象，条目将在所有情况下包含有关返回的各个列的基本信息：

```py
>>> stmt = select(user_table, address_table.c.id)
>>> pprint(stmt.column_descriptions)
[{'expr': Column('id', Integer(), table=<user_account>, primary_key=True, nullable=False),
 'name': 'id',
 'type': Integer()},
 {'expr': Column('name', String(), table=<user_account>, nullable=False),
 'name': 'name',
 'type': String()},
 {'expr': Column('fullname', String(), table=<user_account>),
 'name': 'fullname',
 'type': String()},
 {'expr': Column('id', Integer(), table=<address>, primary_key=True, nullable=False),
 'name': 'id_1',
 'type': Integer()}]
```

在版本 1.4.33 中进行了更改：当对非 ORM 启用的 `Select` 使用 `Select.column_descriptions` 属性时，现在会返回一个值。以前会引发 `NotImplementedError`。

对于 `insert()`、`update()` 和 `delete()` 构造，有两个单独的属性。一个是 `UpdateBase.entity_description`，它返回有关 DML 构造将影响的主要 ORM 实体和数据库表的信息：

```py
>>> from sqlalchemy import update
>>> stmt = update(User).values(name="somename").returning(User.id)
>>> pprint(stmt.entity_description)
{'entity': <class 'User'>,
 'expr': <class 'User'>,
 'name': 'User',
 'table': Table('user_account', ...),
 'type': <class 'User'>}
```

提示

`UpdateBase.entity_description` 包括一个条目 `"table"`，实际上是语句中将要插入、更新或删除的**表**，这并不总是与该类可能被映射到的 SQL “selectable” 相同。例如，在连接表继承方案中，`"table"` 将引用给定实体的本地表。

另一个是`UpdateBase.returning_column_descriptions`，它以与`Select.column_descriptions`大致相似的方式提供有关 RETURNING 集合中列的信息：

```py
>>> pprint(stmt.returning_column_descriptions)
[{'aliased': False,
 'entity': <class 'User'>,
 'expr': <sqlalchemy.orm.attributes.InstrumentedAttribute ...>,
 'name': 'id',
 'type': Integer()}]
```

版本 1.4.33 中的新功能：增加了`UpdateBase.entity_description`和`UpdateBase.returning_column_descriptions`属性。 #### 其他 ORM API 结构

| 对象名称 | 描述 |
| --- | --- |
| aliased(element[, alias, name, flat, ...]) | 生成给定元素的别名，通常是一个`AliasedClass`实例。 |
| AliasedClass | 代表用于 Query 的映射类的“别名”形式。 |
| AliasedInsp | 为`AliasedClass`对象提供检查接口。 |
| Bundle | 一组在一个命名空间下由`Query`返回的 SQL 表达式。 |
| join(left, right[, onclause, isouter, ...]) | 生成左右子句之间的内部连接。 |
| outerjoin(left, right[, onclause, full]) | 生成左外连接 left 和 right 子句之间的左外连接。 |
| with_loader_criteria(entity_or_base, where_criteria[, loader_only, include_aliases, ...]) | 为特定实体的所有出现加载添加额外的 WHERE 条件。 |
| with_parent(instance, prop[, from_entity]) | 创建将此查询的主实体与给定相关实例相关联的过滤条件，使用已建立的`relationship()`配置。 |

```py
function sqlalchemy.orm.aliased(element: _EntityType[_O] | FromClause, alias: FromClause | None = None, name: str | None = None, flat: bool = False, adapt_on_names: bool = False) → AliasedClass[_O] | FromClause | AliasedType[_O]
```

生成给定元素的别名，通常是一个`AliasedClass`实例。

例如：

```py
my_alias = aliased(MyClass)

stmt = select(MyClass, my_alias).filter(MyClass.id > my_alias.id)
result = session.execute(stmt)
```

`aliased()` 函数用于将映射类的临时映射创建为新的可选项。默认情况下，从通常的映射可选项（通常是一个`Table` ）使用`FromClause.alias()` 方法生成可选项。但是，`aliased()` 也可以用于将类链接到新的`select()` 语句。此外，`with_polymorphic()` 函数是`aliased()` 的一个变体，旨在指定所谓的“多态可选项”，它对应于一次连接继承多个子类的联合。

为了方便起见，`aliased()` 函数还接受普通的`FromClause` 构造，例如`Table` 或 `select()` 构造。在这些情况下，对象调用`FromClause.alias()` 方法，并返回新的`Alias` 对象。在这种情况下，返回的`Alias` 未在 ORM 中映射。

另请参阅

ORM 实体别名 - 在 SQLAlchemy 统一教程 中

选择 ORM 别名 - 在 ORM 查询指南 中

参数:

+   `element` – 要别名化的元素。通常是一个映射类，但为了方便起见，也可以是一个`FromClause` 元素。

+   `alias` – 可选的可选择单元，用于将元素映射到。通常用于将对象链接到子查询，并且应该是一个别名选择构造，就像从 `Query.subquery()` 方法或 `Select.subquery()` 或 `Select.alias()` 方法的 `select()` 构造产生的一样。

+   `name` – 用于别名的可选字符串名称，如果未由 `alias` 参数指定。名称，除其他外，形成了将通过 `Query` 对象返回的元组访问的属性名称。创建 `Join` 对象的别名时不支持。

+   `flat` – 布尔值，将传递给 `FromClause.alias()` 调用，以便 `Join` 对象的别名将别名加入联接内部的各个表，而不是创建子查询。这通常由所有现代数据库支持，关于右嵌套联接通常生成更有效的查询。

+   `adapt_on_names` –

    如果为 True，则在将 ORM 实体的映射列与给定可选择的映射时将使用更宽松的 “匹配” - 如果给定的可选择没有与实体上的列对应的列，则将执行基于名称的匹配。此用例是将实体与某些派生的可选择相关联，例如使用聚合函数的可选择：

    ```py
    class UnitPrice(Base):
     __tablename__ = 'unit_price'
     ...
     unit_id = Column(Integer)
     price = Column(Numeric)

    aggregated_unit_price = Session.query(
     func.sum(UnitPrice.price).label('price')
     ).group_by(UnitPrice.unit_id).subquery()

    aggregated_unit_price = aliased(UnitPrice,
     alias=aggregated_unit_price, adapt_on_names=True)
    ```

    在上面，对 `aggregated_unit_price` 上的函数引用 `.price` 将返回 `func.sum(UnitPrice.price).label('price')` 列，因为它与名称 “price” 匹配。通常情况下，“price” 函数不会与实际的 `UnitPrice.price` 列有任何 “列对应”，因为它不是原始的代理。

```py
class sqlalchemy.orm.util.AliasedClass
```

表示用于 Query 的映射类的“别名”形式。

ORM 等效于 `alias()` 构造，此对象使用 `__getattr__` 方案模拟映射类，并维护对真实 `Alias` 对象的引用。

`AliasedClass`的一个主要目的是在 ORM 生成的 SQL 语句中作为一个替代，以便在多个上下文中使用现有映射实体。一个简单的例子：

```py
# find all pairs of users with the same name
user_alias = aliased(User)
session.query(User, user_alias).\
 join((user_alias, User.id > user_alias.id)).\
 filter(User.name == user_alias.name)
```

`AliasedClass`还能够将现有映射类映射到一个全新的可选择项，只要这个可选择项与现有映射可选择项兼容，并且还可以在映射中配置为`relationship()`的目标。查看下面的链接以获取示例。

通常使用`aliased()`函数构造`AliasedClass`对象。当使用`with_polymorphic()`函数时，还会产生附加配置。

结果对象是`AliasedClass`的一个实例。该对象实现了一个属性方案，产生与原始映射类相同的属性和方法接口，允许`AliasedClass`与在原始类上有效的任何属性技术兼容，包括混合属性（参见混合属性）。

可以使用`inspect()`检查`AliasedClass`的基础`Mapper`、别名可选择和其他信息：

```py
from sqlalchemy import inspect
my_alias = aliased(MyClass)
insp = inspect(my_alias)
```

结果检查对象是`AliasedInsp`的一个实例。

另请参见

`aliased()`

`with_polymorphic()`

与别名类的关系

使用窗口函数限制行关系

**类签名**

类`sqlalchemy.orm.AliasedClass` (`sqlalchemy.inspection.Inspectable`, `sqlalchemy.orm.ORMColumnsClauseRole`)

```py
class sqlalchemy.orm.util.AliasedInsp
```

为`AliasedClass`对象提供检查接口。

给定一个 `AliasedClass`，使用 `inspect()` 函数将返回 `AliasedInsp` 对象：

```py
from sqlalchemy import inspect
from sqlalchemy.orm import aliased

my_alias = aliased(MyMappedClass)
insp = inspect(my_alias)
```

`AliasedInsp` 的属性包括:

+   `entity` - 所代表的 `AliasedClass`。

+   `mapper` - 映射底层类的 `Mapper`。

+   `selectable` - 最终表示别名的 `Alias` 构造或 `Select` 构造。

+   `name` - 别名的名称。 在从 `Query` 中返回结果元组时，也用作属性名称。

+   `with_polymorphic_mappers` - 表示在 `AliasedClass` 的 select 构造中表达的所有那些映射器的集合。

+   `polymorphic_on` - 一个替代列或 SQL 表达式，将用作多态加载的“辨别器”。

另请参阅

运行时检查 API

**类签名**

类 `sqlalchemy.orm.AliasedInsp` (`sqlalchemy.orm.ORMEntityColumnsClauseRole`, `sqlalchemy.orm.ORMFromClauseRole`, `sqlalchemy.sql.cache_key.HasCacheKey`, `sqlalchemy.orm.base.InspectionAttr`, `sqlalchemy.util.langhelpers.MemoizedSlots`, `sqlalchemy.inspection.Inspectable`, `typing.Generic`)

```py
class sqlalchemy.orm.Bundle
```

一组由 `Query` 在一个命名空间下返回的 SQL 表达式。

`Bundle` 本质上允许列导向的 `Query` 对象返回的基于元组的结果嵌套。 它还可以通过简单的子类化进行扩展，其中主要的重写功能是如何返回表达式集，允许后处理以及自定义返回类型，而不涉及 ORM 标识映射的类。

另请参阅

属性捆绑

**成员**

__init__(), c, columns, create_row_processor(), is_aliased_class, is_bundle, is_clause_element, is_mapper, label(), single_entity

**类签名**

类`sqlalchemy.orm.Bundle` (`sqlalchemy.orm.ORMColumnsClauseRole`, `sqlalchemy.sql.annotation.SupportsCloneAnnotations`, `sqlalchemy.sql.cache_key.MemoizedHasCacheKey`, `sqlalchemy.inspection.Inspectable`, `sqlalchemy.orm.base.InspectionAttr`)

```py
method __init__(name: str, *exprs: _ColumnExpressionArgument[Any], **kw: Any)
```

构建一个新的`Bundle`。

例如：

```py
bn = Bundle("mybundle", MyClass.x, MyClass.y)

for row in session.query(bn).filter(
 bn.c.x == 5).filter(bn.c.y == 4):
 print(row.mybundle.x, row.mybundle.y)
```

参数：

+   `name` – bundle 的名称。

+   `*exprs` – 组成 bundle 的列或 SQL 表达式。

+   `single_entity=False` – 如果为 True，则此`Bundle`的行可以像映射实体一样在任何封闭元组之外作为“单个实体”返回。

```py
attribute c: ReadOnlyColumnCollection[str, KeyedColumnElement[Any]]
```

`Bundle.columns`的别名。

```py
attribute columns: ReadOnlyColumnCollection[str, KeyedColumnElement[Any]]
```

被此`Bundle`引用的 SQL 表达式的命名空间。

> 例如：
> 
> ```py
> bn = Bundle("mybundle", MyClass.x, MyClass.y)
> 
> q = sess.query(bn).filter(bn.c.x == 5)
> ```
> 
> 还支持 bundle 的嵌套：
> 
> ```py
> b1 = Bundle("b1",
>  Bundle('b2', MyClass.a, MyClass.b),
>  Bundle('b3', MyClass.x, MyClass.y)
>  )
> 
> q = sess.query(b1).filter(
>  b1.c.b2.c.a == 5).filter(b1.c.b3.c.y == 9)
> ```

另请参阅

`Bundle.c`

```py
method create_row_processor(query: Select[Any], procs: Sequence[Callable[[Row[Any]], Any]], labels: Sequence[str]) → Callable[[Row[Any]], Any]
```

为这个`Bundle`生成“行处理”函数。

可以被子类覆盖以在获取结果时提供自定义行为。该方法在查询执行时传递语句对象和一组“行处理”函数；这些处理函数在给定结果行时将返回单个属性值，然后可以将其调整为任何返回数据结构。

下面的示例说明了用直接的 Python 字典替换通常的`Row`返回结构：

```py
from sqlalchemy.orm import Bundle

class DictBundle(Bundle):
 def create_row_processor(self, query, procs, labels):
 'Override create_row_processor to return values as
 dictionaries'

 def proc(row):
 return dict(
 zip(labels, (proc(row) for proc in procs))
 )
 return proc
```

上述`Bundle`的结果将返回字典值：

```py
bn = DictBundle('mybundle', MyClass.data1, MyClass.data2)
for row in session.execute(select(bn)).where(bn.c.data1 == 'd1'):
 print(row.mybundle['data1'], row.mybundle['data2'])
```

```py
attribute is_aliased_class = False
```

如果此对象是`AliasedClass`的实例，则为 True。

```py
attribute is_bundle = True
```

如果此对象是`Bundle`的实例，则为 True。

```py
attribute is_clause_element = False
```

如果此对象是`ClauseElement`的实例，则为 True。

```py
attribute is_mapper = False
```

如果此对象是`Mapper`的实例，则为 True。

```py
method label(name)
```

提供一个传递新标签的此`Bundle`的副本。

```py
attribute single_entity = False
```

如果为 True，则单个 Bundle 的查询将作为单个实体返回，而不是作为键控元组中的元素。

```py
function sqlalchemy.orm.with_loader_criteria(entity_or_base: _EntityType[Any], where_criteria: _ColumnExpressionArgument[bool] | Callable[[Any], _ColumnExpressionArgument[bool]], loader_only: bool = False, include_aliases: bool = False, propagate_to_loaders: bool = True, track_closure_variables: bool = True) → LoaderCriteriaOption
```

为特定实体的所有出现添加额外的 WHERE 条件加载。

从版本 1.4 开始新添加的功能。

`with_loader_criteria()`选项旨在向查询中的特定类型的实体**全局**添加限制条件，这意味着它将应用于实体在 SELECT 查询中的出现以及在任何子查询、联接条件和关系加载中，包括急切和延迟加载器，而无需在查询的任何特定部分指定它。渲染逻辑使用与单表继承相同的系统来确保某个鉴别器应用于表。

例如，使用 2.0 风格的查询，我们可以限制`User.addresses`集合的加载方式，而不管使用的加载类型如何：

```py
from sqlalchemy.orm import with_loader_criteria

stmt = select(User).options(
 selectinload(User.addresses),
 with_loader_criteria(Address, Address.email_address != 'foo'))
)
```

上述示例中，“selectinload”对于`User.addresses`将将给定的过滤条件应用于 WHERE 子句。

另一个示例，其中过滤将应用于联接的 ON 子句，在本示例中使用 1.x 风格的查询：

```py
q = session.query(User).outerjoin(User.addresses).options(
 with_loader_criteria(Address, Address.email_address != 'foo'))
)
```

`with_loader_criteria()`的主要目的是在`SessionEvents.do_orm_execute()`事件处理程序中使用它，以确保特定实体的所有出现以某种方式进行过滤，例如过滤访问控制角色。它还可用于应用于关系加载的条件。在下面的示例中，我们可以将一组特定规则应用于由特定`Session`发出的所有查询：

```py
session = Session(bind=engine)

@event.listens_for("do_orm_execute", session)
def _add_filtering_criteria(execute_state):

 if (
 execute_state.is_select
 and not execute_state.is_column_load
 and not execute_state.is_relationship_load
 ):
 execute_state.statement = execute_state.statement.options(
 with_loader_criteria(
 SecurityRole,
 lambda cls: cls.role.in_(['some_role']),
 include_aliases=True
 )
 )
```

在上面的示例中，`SessionEvents.do_orm_execute()`事件将拦截使用`Session`发出的所有查询。对于那些是 SELECT 语句且不是属性或关系加载的查询，会向查询添加自定义的`with_loader_criteria()`选项。`with_loader_criteria()`选项将在给定的语句中使用，并将自动传播到所有从此查询下降的关系加载。

给定的 criteria 参数是一个接受`cls`参数的`lambda`。给定的类将扩展为包括所有映射的子类，而且本身不必是一个映射的类。

提示

当与`contains_eager()`加载选项一起使用`with_loader_criteria()`选项时，重要的是要注意`with_loader_criteria()`仅影响确定渲染的 SQL 的查询部分，涉及 WHERE 和 FROM 子句。`contains_eager()`选项不会影响 SELECT 语句的渲染，除了列子句外，因此与`with_loader_criteria()`选项没有任何交互。然而，“工作”的方式是，`contains_eager()`应该与某种方式已经从其他实体进行选择的查询一起使用，其中`with_loader_criteria()`可以应用其附加条件。

在下面的例子中，假设有一个映射关系如`A -> A.bs -> B`，给定的`with_loader_criteria()`选项将影响 JOIN 的渲染方式：

```py
stmt = select(A).join(A.bs).options(
 contains_eager(A.bs),
 with_loader_criteria(B, B.flag == 1)
)
```

在上面的例子中，给定的`with_loader_criteria()`选项将影响由`.join(A.bs)`指定的 JOIN 的 ON 子句，因此会按预期应用。`contains_eager()`选项的效果是从`B`中添加列到列子句中：

```py
SELECT
 b.id, b.a_id, b.data, b.flag,
 a.id AS id_1,
 a.data AS data_1
FROM a JOIN b ON a.id = b.a_id AND b.flag = :flag_1
```

在上述语句中使用`contains_eager()`选项对`with_loader_criteria()`选项的行为没有影响。如果省略`contains_eager()`选项，则 SQL 与 FROM 和 WHERE 子句的行为相同，其中`with_loader_criteria()`继续将其条件添加到 JOIN 的 ON 子句中。`contains_eager()`的添加仅影响列子句，其中会添加针对`b`的额外列，然后 ORM 会使用它们生成`B`实例。

警告

在对`with_loader_criteria()`的调用中使用 lambda 表达式仅在**每个唯一类**中调用一次。自定义函数不应在此 lambda 内部调用。有关“lambda SQL”功能的概述，请参阅使用 Lambda 将显著提速到语句生成，该功能仅供高级用户使用。

参数：

+   `entity_or_base` - 映射类，或者一组特定映射类的超类，适用于规则的对象。

+   `where_criteria` -

    应用限制条件的核心 SQL 表达式。这也可以是一个接受目标类作为参数的“lambda:”或 Python 函数，当给定类是一个具有许多不同映射子类的基类时。

    注意

    为了支持 pickle，使用模块级 Python 函数生成 SQL 表达式，而不是 lambda 或固定的 SQL 表达式，后者往往不能 pickle 化。

+   `include_aliases` - 如果为 True，则将规则应用于`aliased()`构造。

+   `propagate_to_loaders` -

    默认为 True，适用于关系加载器，如延迟加载器。这表示选项对象本身包括 SQL 表达式随每个加载的实例一起传递。将其设置为`False`可防止将对象分配给单个实例。

    另请参见

    ORM 查询事件 - 包含使用`with_loader_criteria()`的示例。

    添加全局 WHERE / ON 条件 - 结合`with_loader_criteria()`和`SessionEvents.do_orm_execute()`事件的基本示例。

+   `track_closure_variables` -

    当为 False 时，lambda 表达式内部的闭包变量将不会作为任何缓存键的一部分使用。这允许在 lambda 表达式内部使用更复杂的表达式，但要求 lambda 确保每次给定特定类时都返回相同的 SQL。

    新功能于版本 1.4.0b2 中添加。

```py
function sqlalchemy.orm.join(left: _FromClauseArgument, right: _FromClauseArgument, onclause: _OnClauseArgument | None = None, isouter: bool = False, full: bool = False) → _ORMJoin
```

生成左右子句之间的内部连接。

`join()` 是对核心连接接口的扩展，由 `join()` 提供，其中左右可选择的对象不仅可以是核心可选择的对象，如 `Table`，还可以是映射类或 `AliasedClass` 实例。 “on” 子句可以是 SQL 表达式，也可以是引用已配置的 `relationship()` 的 ORM 映射属性。

在现代用法中，通常不常需要 `join()`，因为其功能已封装在 `Select.join()` 和 `Query.join()` 方法中。 这些方法除了 `join()` 本身之外，还具有大量的自动化功能。 使用 ORM 启用的 SELECT 语句显式使用 `join()` 涉及使用 `Select.select_from()` 方法，如下所示：

```py
from sqlalchemy.orm import join
stmt = select(User).\
 select_from(join(User, Address, User.addresses)).\
 filter(Address.email_address=='foo@bar.com')
```

在现代的 SQLAlchemy 中，上述连接可以更简洁地编写为：

```py
stmt = select(User).\
 join(User.addresses).\
 filter(Address.email_address=='foo@bar.com')
```

警告

直接使用 `join()` 可能无法正确地处理现代 ORM 选项，例如 `with_loader_criteria()`。 强烈建议在创建 ORM 连接时使用像 `Select.join()` 和 `Select.join_from()` 这样的方法提供的惯用连接模式。

另请参阅

连接 - 在 ORM 查询指南中了解惯用的 ORM 连接模式的背景

```py
function sqlalchemy.orm.outerjoin(left: _FromClauseArgument, right: _FromClauseArgument, onclause: _OnClauseArgument | None = None, full: bool = False) → _ORMJoin
```

在左侧和右侧子句之间生成左外连接。

这是 `join()` 函数的“外连接”版本，具有相同的行为，除了生成 OUTER JOIN 外，还生成了其他用法详细信息，请参阅该函数的文档。

```py
function sqlalchemy.orm.with_parent(instance: object, prop: attributes.QueryableAttribute[Any], from_entity: _EntityType[Any] | None = None) → ColumnElement[bool]
```

创建将此查询的主要实体与给定的相关实例相关联的过滤标准，使用已建立的 `relationship()` 配置。

例如：

```py
stmt = select(Address).where(with_parent(some_user, User.addresses))
```

渲染的 SQL 与在给定父项上从该属性触发延迟加载时呈现的 SQL 相同，这意味着在 Python 中从父对象获取适当的状态而无需在呈现的语句中渲染对父表的连接。

给定的属性还可以使用 `PropComparator.of_type()` 来指示条件的左侧：

```py
a1 = aliased(Address)
a2 = aliased(Address)
stmt = select(a1, a2).where(
 with_parent(u1, User.addresses.of_type(a2))
)
```

上述用法相当于使用 `from_entity()` 参数：

```py
a1 = aliased(Address)
a2 = aliased(Address)
stmt = select(a1, a2).where(
 with_parent(u1, User.addresses, from_entity=a2)
)
```

参数：

+   `instance` – 具有某些 `relationship()` 的实例。

+   `property` – 类绑定的属性，指示应使用哪个实例的关系来协调父/子关系。

+   `from_entity` –

    要考虑为左侧的实体。默认为 `Query` 本身的“零”实体。

    1.2 版中的新内容。

## ORM Loader 选项

Loader 选项是对象，当传递给 `Select.options()` 方法时，影响了 `Select` 对象或类似的 SQL 结构的列和关系属性的加载。大多数 loader 选项都来自 `Load` 层次结构。有关使用 loader 选项的完整概述，请参阅下面的链接部分。

另请参阅

+   列加载选项 - 详细介绍了影响如何加载列和 SQL 表达式映射属性的映射和加载选项

+   关系加载技术 - 详细介绍了影响如何加载 `relationship()` 映射属性的关系和加载选项

## ORM 执行选项

ORM 级别的执行选项是关键字选项，可以通过`Session.execute.execution_options`参数与语句执行关联，该参数是由`Session`方法（如`Session.execute()`和`Session.scalars()`）接受的字典参数，或者直接通过`Executable.execution_options()`方法将它们与要调用的语句直接关联，该方法将它们作为任意关键字参数接受。

ORM 级别的选项与`Connection.execution_options()`中记录的核心级别执行选项不同。需要注意的是，下面讨论的 ORM 选项与核心级别方法`Connection.execution_options()`或`Engine.execution_options()` **不**兼容；即使`Engine`或`Connection`与正在使用的`Session`相关联，这些选项在此级别将被忽略。

在本节中，将演示`Executable.execution_options()`方法的样式示例。

### 刷新现有数据

`populate_existing`执行选项确保对于加载的所有行，`Session`中对应的实例将被完全刷新 - 擦除对象中的任何现有数据（包括待定更改）并用从结果加载的数据替换。

示例用法如下：

```py
>>> stmt = select(User).execution_options(populate_existing=True)
>>> result = session.execute(stmt)
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
... 
```

通常，ORM 对象只加载一次，如果它们与后续结果行中的主键匹配，则不会将该行应用于对象。这既是为了保留对象上未决的未刷新更改，也是为了避免刷新已经存在的数据的开销和复杂性。 `Session` 假定高度隔离的事务的默认工作模型，并且在预期事务内部的数据发生更改的程度上，会使用明确的步骤来处理那些使用情况，而不是正在进行的本地更改。

使用 `populate_existing`，可以刷新与查询匹配的任何一组对象，并且还可以控制关系加载器选项。例如，刷新一个实例同时也刷新相关的一组对象：

```py
stmt = (
    select(User)
    .where(User.name.in_(names))
    .execution_options(populate_existing=True)
    .options(selectinload(User.addresses))
)
# will refresh all matching User objects as well as the related
# Address objects
users = session.execute(stmt).scalars().all()
```

`populate_existing` 的另一个用例是支持各种属性加载功能，这些功能可以根据每个查询的情况改变属性的加载方式。适用于此选项的选项包括：

+   `with_expression()` 选项

+   `PropComparator.and_()` 方法可以修改加载器策略加载的内容

+   `contains_eager()` 选项

+   `with_loader_criteria()` 选项

+   `load_only()` 选项以选择要刷新的属性

`populate_existing` 执行选项等效于 1.x 风格 ORM 查询中的 `Query.populate_existing()` 方法。

另请参阅

我正在使用我的 Session 重新加载数据，但它没有看到我在其他地方提交的更改 - 在常见问题解答中

刷新/过期 - 在 ORM `Session` 文档中 ### 自动刷新

当传递此选项为 `False` 时，将导致 `Session` 不调用“自动刷新”步骤。这相当于使用 `Session.no_autoflush` 上下文管理器来禁用自动刷新：

```py
>>> stmt = select(User).execution_options(autoflush=False)
>>> session.execute(stmt)
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
... 
```

此选项还可用于启用 ORM 的 `Update` 和 `Delete` 查询。

`autoflush` 执行选项相当于 1.x 风格 ORM 查询中的 `Query.autoflush()` 方法。

另请参阅

刷新  ### 使用 Yield Per 获取大型结果集

`yield_per` 执行选项是一个整数值，它将导致 `Result` 一次仅缓冲有限数量的行和/或 ORM 对象，然后将数据提供给客户端。

通常，ORM 会立即获取**所有**行，为每个构造 ORM 对象，并将这些对象组装到一个单一的缓冲区中，然后将该缓冲区作为行的来源传递给 `Result` 对象以返回。此行为的理由是允许正确处理诸如联接急加载、结果唯一化以及依赖于标识映射在每个对象在被提取时保持一致状态的结果处理逻辑等功能的情况。

`yield_per` 选项的目的是更改此行为，使得 ORM 结果集针对迭代大型结果集（例如 > 10K 行）进行了优化，用户已确定上述模式不适用的情况。当使用 `yield_per` 时，ORM 将会将 ORM 结果批量成子集合，并在迭代 `Result` 对象时单独从每个子集合中产生行，这样 Python 解释器就不需要声明非常大的内存区域，这既耗时又导致内存使用过多。该选项既影响数据库游标的使用方式，也影响 ORM 构造要传递给 `Result` 的行和对象的方式。

提示

由上可见，`Result` 必须以可迭代的方式消耗，即使用诸如 `for row in result` 的迭代或使用部分行方法，如 `Result.fetchmany()` 或 `Result.partitions()`。调用 `Result.all()` 将失去使用 `yield_per` 的目的。

使用`yield_per`等同于同时利用`Connection.execution_options.stream_results`执行选项，该选项在支持的情况下选择使用后端的服务器端游标，并且在返回的`Result`对象上使用`Result.yield_per()`方法，该方法建立了要获取的行的固定大小以及一次构建多少个 ORM 对象的相应限制。

提示

`yield_per`现在也作为 Core 执行选项可用，详细描述在使用服务器端游标（即流式结果）。本节详细介绍了将`yield_per`作为 ORM `Session`的执行选项的用法。该选项在两种情境下尽可能相似地行为。

在与 ORM 一起使用时，`yield_per`必须通过给定语句上的`Executable.execution_options()`方法或通过将其传递给`Session.execute()`或其他类似`Session`方法的`Session.execute.execution_options`参数来建立。获取 ORM 对象的典型用法如下所示：

```py
>>> stmt = select(User).execution_options(yield_per=10)
>>> for user_obj in session.scalars(stmt):
...     print(user_obj)
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
[...]  ()
User(id=1, name='spongebob', fullname='Spongebob Squarepants')
User(id=2, name='sandy', fullname='Sandy Cheeks')
...
>>> # ... rows continue ...
```

上述代码等同于下面的示例，该示例使用`Connection.execution_options.stream_results`和`Connection.execution_options.max_row_buffer` Core 级别的执行选项，与`Result`的`Result.yield_per()`方法一起使用：

```py
# equivalent code
>>> stmt = select(User).execution_options(stream_results=True, max_row_buffer=10)
>>> for user_obj in session.scalars(stmt).yield_per(10):
...     print(user_obj)
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
[...]  ()
User(id=1, name='spongebob', fullname='Spongebob Squarepants')
User(id=2, name='sandy', fullname='Sandy Cheeks')
...
>>> # ... rows continue ...
```

`yield_per`通常与`Result.partitions()`方法结合使用，该方法将迭代分组分区中的行。每个分区的大小默认为传递给`yield_per`的整数值，如下例所示：

```py
>>> stmt = select(User).execution_options(yield_per=10)
>>> for partition in session.scalars(stmt).partitions():
...     for user_obj in partition:
...         print(user_obj)
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
[...]  ()
User(id=1, name='spongebob', fullname='Spongebob Squarepants')
User(id=2, name='sandy', fullname='Sandy Cheeks')
...
>>> # ... rows continue ...
```

当使用集合时，`yield_per`执行选项与“子查询”急加载或“连接”急加载不兼容。对于“select in”急加载，只要数据库驱动程序支持多个独立游标，它就可能与之兼容。

另外，`yield_per`执行选项与`Result.unique()`方法不兼容；由于此方法依赖于存储所有行的完整标识集，它必然会破坏使用`yield_per`的目的，即处理任意数量的行。

自 1.4.6 版本更改：当从使用`Result`对象获取 ORM 行时，如果同时使用了`Result.unique()`过滤器和`yield_per`执行选项，则会引发异常。

在使用旧版 `Query` 对象和 1.x 样式 ORM 时，`Query.yield_per()` 方法的结果与`yield_per`执行选项的结果相同。

另请参阅

使用服务器端游标（又名流式结果） ### 标识令牌

深度炼金术

此选项是一个高级特性，主要用于与水平分片扩展一起使用。对于从不同“分片”或分区加载具有相同主键的对象的典型情况，请首先考虑每个分片使用单独的`Session`对象。

“标识令牌”是一个任意值，可以与新加载对象的标识键相关联。这个元素首先存在于支持每行“分片”的扩展中，其中对象可以从特定数据库表的任意数量的副本中加载，尽管这些副本具有重叠的主键值。 “标识令牌”的主要消费者是 Horizontal Sharding 扩展，它提供了一个��用框架，用于在特定数据库表的多个“分片”之间持久化对象。

`identity_token`执行选项可以在每个查询基础上直接影响此令牌的使用。直接使用它，可以将一个对象的多个实例填充到`Session`中，这些实例具有相同的主键和源表，但具有不同的“标识”。

一个这样的例子是使用 Schema 名称翻译功能，该功能可以影响查询范围内的模式选择，从而将来自不同模式的同名表中的对象填充到`Session`中。给定一个映射如下：

```py
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column

class Base(DeclarativeBase):
    pass

class MyTable(Base):
    __tablename__ = "my_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
```

上述类的默认“模式”名称为`None`，意味着不会将模式限定写入 SQL 语句中。然而，如果我们利用`Connection.execution_options.schema_translate_map`，将`None`映射到替代模式，我们可以将`MyTable`的实例放入两个不同的模式中：

```py
engine = create_engine(
    "postgresql+psycopg://scott:tiger@localhost/test",
)

with Session(
    engine.execution_options(schema_translate_map={None: "test_schema"})
) as sess:
    sess.add(MyTable(name="this is schema one"))
    sess.commit()

with Session(
    engine.execution_options(schema_translate_map={None: "test_schema_2"})
) as sess:
    sess.add(MyTable(name="this is schema two"))
    sess.commit()
```

上述两个块每次创建一个与不同模式转换映射相关联的`Session`对象，并将`MyTable`的实例持久化到`test_schema.my_table`和`test_schema_2.my_table`中。

上述`Session`对象是独立的。如果我们想要在一个事务中持久化这两个对象，我们需要使用 Horizontal Sharding 扩展来实现。

然而，我们可以在一个会话中演示查询这些对象的方法如下：

```py
with Session(engine) as sess:
    obj1 = sess.scalar(
        select(MyTable)
        .where(MyTable.id == 1)
        .execution_options(
            schema_translate_map={None: "test_schema"},
            identity_token="test_schema",
        )
    )
    obj2 = sess.scalar(
        select(MyTable)
        .where(MyTable.id == 1)
        .execution_options(
            schema_translate_map={None: "test_schema_2"},
            identity_token="test_schema_2",
        )
    )
```

`obj1`和`obj2`彼此不同。然而，它们都指向`MyTable`类的主键 id 1，但是它们是不同的。这就是`identity_token`发挥作用的地方，我们可以在每个对象的检查中看到它，在那里我们查看`InstanceState.key`以查看这两个不同的标识令牌：

```py
>>> from sqlalchemy import inspect
>>> inspect(obj1).key
(<class '__main__.MyTable'>, (1,), 'test_schema')
>>> inspect(obj2).key
(<class '__main__.MyTable'>, (1,), 'test_schema_2')
```

当使用 Horizontal Sharding 扩展时，上述逻辑会自动发生。

2.0.0rc1 版本中的新功能：- 添加了`identity_token`ORM 级别执行选项。

另请参阅

水平分片 - 在 ORM 示例 部分。请参阅脚本 `separate_schema_translates.py`，了解使用完整分片 API 进行上述用例演示。

#### 检查启用 ORM 的 SELECT 和 DML 语句中的实体和列

`select()` 结构以及 `insert()`、`update()` 和 `delete()` 结构（自 SQLAlchemy 1.4.33 起，对于后三个 DML 结构）都支持检查这些语句所针对的实体，以及结果集中将返回的列和数据类型。

对于 `Select` 对象，此信息可以从 `Select.column_descriptions` 属性获取。该属性的操作方式与传统的 `Query.column_descriptions` 属性相同。返回的格式是一个字典列表：

```py
>>> from pprint import pprint
>>> user_alias = aliased(User, name="user2")
>>> stmt = select(User, User.id, user_alias)
>>> pprint(stmt.column_descriptions)
[{'aliased': False,
 'entity': <class 'User'>,
 'expr': <class 'User'>,
 'name': 'User',
 'type': <class 'User'>},
 {'aliased': False,
 'entity': <class 'User'>,
 'expr': <....InstrumentedAttribute object at ...>,
 'name': 'id',
 'type': Integer()},
 {'aliased': True,
 'entity': <AliasedClass ...; User>,
 'expr': <AliasedClass ...; User>,
 'name': 'user2',
 'type': <class 'User'>}]
```

当与非 ORM 对象（如普通的 `Table` 或 `Column` 对象）一起使用 `Select.column_descriptions` 时，所有情况下都会包含有关返回的各个列的基本信息：

```py
>>> stmt = select(user_table, address_table.c.id)
>>> pprint(stmt.column_descriptions)
[{'expr': Column('id', Integer(), table=<user_account>, primary_key=True, nullable=False),
 'name': 'id',
 'type': Integer()},
 {'expr': Column('name', String(), table=<user_account>, nullable=False),
 'name': 'name',
 'type': String()},
 {'expr': Column('fullname', String(), table=<user_account>),
 'name': 'fullname',
 'type': String()},
 {'expr': Column('id', Integer(), table=<address>, primary_key=True, nullable=False),
 'name': 'id_1',
 'type': Integer()}]
```

自 1.4.33 版更改：当对未启用 ORM 的 `Select` 使用 `Select.column_descriptions` 属性时，现在会返回一个值。之前会引发 `NotImplementedError`。

对于`insert()`、`update()`和`delete()`构造，存在两个单独的属性。一个是`UpdateBase.entity_description`，它返回关于 DML 构造将影响的主 ORM 实体和数据库表的信息：

```py
>>> from sqlalchemy import update
>>> stmt = update(User).values(name="somename").returning(User.id)
>>> pprint(stmt.entity_description)
{'entity': <class 'User'>,
 'expr': <class 'User'>,
 'name': 'User',
 'table': Table('user_account', ...),
 'type': <class 'User'>}
```

提示

`UpdateBase.entity_description`包括一个条目`"table"`，实际上是由语句插入、更新或删除的**表**，这与类可能映射到的 SQL“selectable”不一定相同。例如，在连接表继承场景中，`"table"`将引用给定实体的本地表。

另一个是`UpdateBase.returning_column_descriptions`，它以与`Select.column_descriptions`大致相似的方式提供了与 RETURNING 集合中存在的列有关的信息：

```py
>>> pprint(stmt.returning_column_descriptions)
[{'aliased': False,
 'entity': <class 'User'>,
 'expr': <sqlalchemy.orm.attributes.InstrumentedAttribute ...>,
 'name': 'id',
 'type': Integer()}]
```

新版本 1.4.33 中增加了`UpdateBase.entity_description`和`UpdateBase.returning_column_descriptions`属性。  #### 额外的 ORM API 构造

| 对象名称 | 描述 |
| --- | --- |
| aliased(element[, alias, name, flat, ...]) | 生成给定元素的别名，通常是`AliasedClass`实例。 |
| AliasedClass | 代表用于 Query 的映射类的“别名”形式。 |
| AliasedInsp | 为`AliasedClass`对象提供检查接口。 |
| Bundle | 将由`Query`返回的 SQL 表达式分组到一个命名空间下。 |
| join(left, right[, onclause, isouter, ...]) | 生成左右子句之间的内连接。 |
| outerjoin(left, right[, onclause, full]) | 在左右子句之间生成左外连接。 |
| with_loader_criteria(entity_or_base, where_criteria[, loader_only, include_aliases, ...]) | 为特定实体的所有出现加载添加额外的 WHERE 条件。 |
| with_parent(instance, prop[, from_entity]) | 创建过滤条件，将此查询的主实体与给定的相关实例关联起来，使用已建立的`relationship()`配置。 |

```py
function sqlalchemy.orm.aliased(element: _EntityType[_O] | FromClause, alias: FromClause | None = None, name: str | None = None, flat: bool = False, adapt_on_names: bool = False) → AliasedClass[_O] | FromClause | AliasedType[_O]
```

创建给定元素的别名，通常是`AliasedClass`实例。

例如：

```py
my_alias = aliased(MyClass)

stmt = select(MyClass, my_alias).filter(MyClass.id > my_alias.id)
result = session.execute(stmt)
```

`aliased()`函数用于创建一个映射类到新的可选项的临时映射。默认情况下，通过通常的映射可选项（通常是`Table`）使用`FromClause.alias()`方法生成可选项。但是，`aliased()`也可以用于将类链接到新的`select()`语句。此外，`with_polymorphic()`函数是`aliased()`的变体，旨在指定所谓的“多态可选项”，该可选项对应于一次性联接继承子类的联合。

为了方便起见，`aliased()`函数也接受普通的`FromClause`构造，比如`Table`或`select()`构造。在这些情况下，对象会调用`FromClause.alias()`方法，并返回新的`Alias`对象。在这种情况下，返回的`Alias`不会被 ORM 映射。

另请参阅

ORM 实体别名 - 在 SQLAlchemy 统一教程 中

选择 ORM 别名 - 在 ORM 查询指南 中

参数：

+   `element` – 要别名的元素。通常是一个映射类，但为了方便，也可以是一个`FromClause`元素。

+   `alias` – 可选的可选择单元，用于将元素映射到。通常用于将对象链接到子查询，并且应该是一个别名选择结构，就像从`Query.subquery()`方法或`Select.subquery()`或`Select.alias()`方法生成的那样`select()`结构。

+   `name` – 如果未由`alias`参数指定，则使用的可选字符串名称。名称，除其他外，形成了通过`Query`对象返回的元组访问的属性名称。在创建`Join`对象的别名时不支持。

+   `flat` – 布尔值，将传递到`FromClause.alias()`调用，以便`Join`对象的别名将别名加入到连接内的单个表，而不是创建子查询。这通常由所有现代数据库支持，关于右嵌套连接，通常生成更有效的查询。

+   `adapt_on_names` –

    如果为 True，则在将 ORM 实体的映射列映射到给定可选择的列时将使用更自由的“匹配” - 如果给定的可选择否则没有与实体上的列对应的列，则将执行基于名称的匹配。这种用例是当将实体与某个派生的可选择相关联时，例如使用聚合函数的可选择：

    ```py
    class UnitPrice(Base):
     __tablename__ = 'unit_price'
     ...
     unit_id = Column(Integer)
     price = Column(Numeric)

    aggregated_unit_price = Session.query(
     func.sum(UnitPrice.price).label('price')
     ).group_by(UnitPrice.unit_id).subquery()

    aggregated_unit_price = aliased(UnitPrice,
     alias=aggregated_unit_price, adapt_on_names=True)
    ```

    在上面，对`aggregated_unit_price`上的函数引用`.price`将返回`func.sum(UnitPrice.price).label('price')`列，因为它根据名称“price”进行匹配。通常，“price”函数不会与实际的`UnitPrice.price`列有任何“列对应”，因为它不是原始列的代理。

```py
class sqlalchemy.orm.util.AliasedClass
```

表示与查询一起使用的映射类的“别名”形式。

ORM 中 `alias()` 构造的等效对象，该对象使用 `__getattr__` 方案模仿映射类，并维护对真实 `Alias` 对象的引用。

`AliasedClass` 的一个主要目的是在 ORM 生成的 SQL 语句中作为一个替代，以便一个现有的映射实体可以在多个上下文中使用。一个简单的例子：

```py
# find all pairs of users with the same name
user_alias = aliased(User)
session.query(User, user_alias).\
 join((user_alias, User.id > user_alias.id)).\
 filter(User.name == user_alias.name)
```

`AliasedClass` 还能够将现有的映射类映射到一个全新的可选择项，前提是该可选择项与现有的映射可选择项兼容，并且还可以在映射中配置为 `relationship()` 的目标。请参见下面的链接以获取示例。

`AliasedClass` 对象通常使用 `aliased()` 函数构造。在使用 `with_polymorphic()` 函数时，还会进行额外配置。

结果对象是 `AliasedClass` 的一个实例。该对象实现了一个属性方案，产生与原始映射类相同的属性和方法接口，允许 `AliasedClass` 与在原始类上工作的任何属性技术兼容，包括混合属性（参见 混合属性）。

`AliasedClass` 可以通过 `inspect()` 进行检查，以获取其底层的 `Mapper`、别名可选择项和其他信息：

```py
from sqlalchemy import inspect
my_alias = aliased(MyClass)
insp = inspect(my_alias)
```

结果检查对象是 `AliasedInsp` 的一个实例。

另请参阅

`aliased()`

`with_polymorphic()`

与别名类的关系

使用窗口函数限制行关系

**类签名**

类 `sqlalchemy.orm.AliasedClass` (`sqlalchemy.inspection.Inspectable`, `sqlalchemy.orm.ORMColumnsClauseRole`)

```py
class sqlalchemy.orm.util.AliasedInsp
```

为`AliasedClass`对象提供检查接口。

给定一个`AliasedClass`，使用`inspect()`函数将返回一个`AliasedInsp`对象：

```py
from sqlalchemy import inspect
from sqlalchemy.orm import aliased

my_alias = aliased(MyMappedClass)
insp = inspect(my_alias)
```

`AliasedInsp`上的属性包括：

+   `entity` - 表示的`AliasedClass`。

+   `mapper` - 映射底层类的`Mapper`。

+   `selectable` - 最终表示别名`Table`或`Select`构造的`Alias`构造。

+   `name` - 别名的名称。 当从`Query`中的结果元组返回时，也用作属性名称。

+   `with_polymorphic_mappers` - 指示在 `AliasedClass` 的 select 构造中表达的所有这些映射器的集合。

+   `polymorphic_on` - 一个备用的列或 SQL 表达式，将用作多态加载的“辨别器”。

另请参阅

运行时检查 API

**类签名**

类`sqlalchemy.orm.AliasedInsp` (`sqlalchemy.orm.ORMEntityColumnsClauseRole`, `sqlalchemy.orm.ORMFromClauseRole`, `sqlalchemy.sql.cache_key.HasCacheKey`, `sqlalchemy.orm.base.InspectionAttr`, `sqlalchemy.util.langhelpers.MemoizedSlots`, `sqlalchemy.inspection.Inspectable`, `typing.Generic`)

```py
class sqlalchemy.orm.Bundle
```

由`Query`在一个命名空间下返回的 SQL 表达式的分组。

`Bundle`基本上允许对列导向的`Query`对象返回的基于元组的结果进行嵌套。 它还可以通过简单的子类化进行扩展，其中要覆盖的主要能力是如何返回表达式集，允许后处理以及自定义返回类型，而不涉及 ORM 身份映射的类。

另请参阅

使用 Bundles 分组选择的属性

**成员**

__init__(), c, columns, create_row_processor(), is_aliased_class, is_bundle, is_clause_element, is_mapper, label(), single_entity

**类签名**

类`sqlalchemy.orm.Bundle` (`sqlalchemy.orm.ORMColumnsClauseRole`, `sqlalchemy.sql.annotation.SupportsCloneAnnotations`, `sqlalchemy.sql.cache_key.MemoizedHasCacheKey`, `sqlalchemy.inspection.Inspectable`, `sqlalchemy.orm.base.InspectionAttr`)。

```py
method __init__(name: str, *exprs: _ColumnExpressionArgument[Any], **kw: Any)
```

构造一个新的`Bundle`。

例如：

```py
bn = Bundle("mybundle", MyClass.x, MyClass.y)

for row in session.query(bn).filter(
 bn.c.x == 5).filter(bn.c.y == 4):
 print(row.mybundle.x, row.mybundle.y)
```

参数：

+   `name` – bundle 的名称。

+   `*exprs` – 组成 bundle 的列或 SQL 表达式。

+   `single_entity=False` – 如果为 True，则此`Bundle`的行可以作为“单个实体”返回，而不是在与映射实体相同的元组中。

```py
attribute c: ReadOnlyColumnCollection[str, KeyedColumnElement[Any]]
```

`Bundle.columns`的别名。

```py
attribute columns: ReadOnlyColumnCollection[str, KeyedColumnElement[Any]]
```

被此`Bundle`引用的 SQL 表达式的命名空间。

> 例如：
> 
> ```py
> bn = Bundle("mybundle", MyClass.x, MyClass.y)
> 
> q = sess.query(bn).filter(bn.c.x == 5)
> ```
> 
> 也支持 bundle 的嵌套：
> 
> ```py
> b1 = Bundle("b1",
>  Bundle('b2', MyClass.a, MyClass.b),
>  Bundle('b3', MyClass.x, MyClass.y)
>  )
> 
> q = sess.query(b1).filter(
>  b1.c.b2.c.a == 5).filter(b1.c.b3.c.y == 9)
> ```

另请参阅

`Bundle.c`

```py
method create_row_processor(query: Select[Any], procs: Sequence[Callable[[Row[Any]], Any]], labels: Sequence[str]) → Callable[[Row[Any]], Any]
```

为此`Bundle`生成“行处理”函数。

可以被子类覆盖以在获取结果时提供自定义行为。该方法在查询执行时传递给语句对象和一组“行处理”函数；这些处理函数在给定结果行时将返回单个属性值，然后可以将其调整为任何返回数据结构。

下面的示例说明了将通常的`Row`返回结构替换为直接的 Python 字典：

```py
from sqlalchemy.orm import Bundle

class DictBundle(Bundle):
 def create_row_processor(self, query, procs, labels):
 'Override create_row_processor to return values as
 dictionaries'

 def proc(row):
 return dict(
 zip(labels, (proc(row) for proc in procs))
 )
 return proc
```

上述`Bundle`的结果将返回字典值：

```py
bn = DictBundle('mybundle', MyClass.data1, MyClass.data2)
for row in session.execute(select(bn)).where(bn.c.data1 == 'd1'):
 print(row.mybundle['data1'], row.mybundle['data2'])
```

```py
attribute is_aliased_class = False
```

如果此对象是`AliasedClass`的实例，则为 True。

```py
attribute is_bundle = True
```

如果此对象是`Bundle`的实例，则为 True。

```py
attribute is_clause_element = False
```

如果此对象是`ClauseElement`的实例，则为 True。

```py
attribute is_mapper = False
```

如果此对象是`Mapper`的实例，则为 True。

```py
method label(name)
```

提供一个传递了新标签的 `Bundle` 的副本。

```py
attribute single_entity = False
```

如果为 True，则对于单个 Bundle 的查询将返回为单个实体，而不是在一个键元组中的元素。

```py
function sqlalchemy.orm.with_loader_criteria(entity_or_base: _EntityType[Any], where_criteria: _ColumnExpressionArgument[bool] | Callable[[Any], _ColumnExpressionArgument[bool]], loader_only: bool = False, include_aliases: bool = False, propagate_to_loaders: bool = True, track_closure_variables: bool = True) → LoaderCriteriaOption
```

为所有特定实体的加载添加额外的 WHERE 条件。

新版本 1.4 中的新功能。

`with_loader_criteria()` 选项旨在向查询中的特定类型的实体添加限制条件，**全局**地，这意味着它将应用于实体在 SELECT 查询中的出现方式以及任何子查询、连接条件和关系加载中，包括急切加载和延迟加载器，而无需在查询的任何特定部分中指定它。呈现逻辑使用与单表继承相同的系统来确保某个鉴别器应用于表。

例如，使用 2.0 样式 查询，我们可以限制 `User.addresses` 集合的加载方式，而不管所使用的加载类型是什么：

```py
from sqlalchemy.orm import with_loader_criteria

stmt = select(User).options(
 selectinload(User.addresses),
 with_loader_criteria(Address, Address.email_address != 'foo'))
)
```

上面，“selectinload” 对于 `User.addresses` 将把给定的过滤条件应用于 WHERE 子句。

另一个示例，其中过滤将应用于连接的 ON 子句，在此示例中使用 1.x 样式 查询：

```py
q = session.query(User).outerjoin(User.addresses).options(
 with_loader_criteria(Address, Address.email_address != 'foo'))
)
```

`with_loader_criteria()` 的主要目的是在 `SessionEvents.do_orm_execute()` 事件处理程序中使用它，以确保对特定实体的所有出现方式都以某种方式进行过滤，例如针对访问控制角色的过滤。它还可以用于将条件应用于关系加载。在下面的示例中，我们可以将一定的规则应用于特定 `Session` 发出的所有查询：

```py
session = Session(bind=engine)

@event.listens_for("do_orm_execute", session)
def _add_filtering_criteria(execute_state):

 if (
 execute_state.is_select
 and not execute_state.is_column_load
 and not execute_state.is_relationship_load
 ):
 execute_state.statement = execute_state.statement.options(
 with_loader_criteria(
 SecurityRole,
 lambda cls: cls.role.in_(['some_role']),
 include_aliases=True
 )
 )
```

在上面的示例中，`SessionEvents.do_orm_execute()` 事件将拦截使用 `Session` 发出的所有查询。对于那些是 SELECT 语句且不是属性或关系加载的查询，会向查询中添加自定义 `with_loader_criteria()` 选项。`with_loader_criteria()` 选项将在给定的语句中使用，并将自动传播到所有从该查询继承的关系加载。

给定的 criteria 参数是一个接受 `cls` 参数的 `lambda`。给定的类将扩展为包括所有映射的子类，它本身不需要是一个映射的类。

提示

在与`contains_eager()`加载选项一起使用`with_loader_criteria()`选项时，重要的是要注意`with_loader_criteria()`仅影响决定渲染的 SQL 的部分查询，即 WHERE 和 FROM 子句。`contains_eager()`选项不影响 SELECT 语句在列子句之外的渲染，因此与`with_loader_criteria()`选项没有任何交互。然而，事情的“运作方式”是，`contains_eager()`旨在与某种方式已经从附加实体中进行选择的查询一起使用，其中`with_loader_criteria()`可以应用其附加条件。

在下面的示例中，假设有一个映射关系为`A -> A.bs -> B`，给定的`with_loader_criteria()`选项将影响 JOIN 的呈现方式：

```py
stmt = select(A).join(A.bs).options(
 contains_eager(A.bs),
 with_loader_criteria(B, B.flag == 1)
)
```

给定的`with_loader_criteria()`选项将影响由`.join(A.bs)`指定的 JOIN 的 ON 子句，因此被如预期般应用。`contains_eager()`选项的作用是将 B 的列添加到列子句中：

```py
SELECT
 b.id, b.a_id, b.data, b.flag,
 a.id AS id_1,
 a.data AS data_1
FROM a JOIN b ON a.id = b.a_id AND b.flag = :flag_1
```

在上述语句中使用`contains_eager()`选项对`with_loader_criteria()`选项的行为没有影响。如果省略了`contains_eager()`选项，那么 SQL 在 FROM 和 WHERE 子句方面的情况将与`with_loader_criteria()`继续将其条件添加到 JOIN 的 ON 子句中一样。`contains_eager()`的添加仅影响列子句，即添加了针对 b 的附加列，然后 ORM 将其消耗以产生 B 实例。

警告

在调用`with_loader_criteria()`时使用的 lambda 仅会被**每个唯一类**调用一次。自定义函数不应在此 lambda 内部调用。有关“lambda SQL”功能的概述，请参阅使用 Lambda 将语句生成速度提升到显著水平，该功能仅适用于高级用途。

参数：

+   `entity_or_base` - 一个映射类，或者是一组特定映射类的超类，适用于该规则。

+   `where_criteria` - 

    一个应用限制条件的核心 SQL 表达式。当给定类是具有许多不同映射子类的基类时，这也可以是“lambda:”或 Python 函数，它接受目标类作为参数。

    注意

    为了支持 pickle，应使用模块级别的 Python 函数来生成 SQL 表达式，而不是 lambda 或固定的 SQL 表达式，后者倾向于不可 pickle。

+   `include_aliases` - 如果为 True，则也将规则应用于`aliased()`构造。

+   `propagate_to_loaders` - 

    默认为 True，适用于诸如延迟加载器之类的关系加载器。这表示选项对象本身包括 SQL 表达式与每个加载的实例一起传递。将其设置为`False`可防止将对象分配给单个实例。

    另请参阅

    ORM 查询事件 - 包括使用`with_loader_criteria()`的示例。

    添加全局 WHERE / ON 条件 - 将`with_loader_criteria()`与`SessionEvents.do_orm_execute()`事件相结合的基本示例。

+   `track_closure_variables` -

    当 False 时，lambda 表达式内部的闭包变量将不会用作任何缓存键的一部分。这允许在 lambda 表达式内部使用更复杂的表达式，但需要 lambda 确保每次给定特定类时都返回相同的 SQL。

    从版本 1.4.0b2 开始新增。

```py
function sqlalchemy.orm.join(left: _FromClauseArgument, right: _FromClauseArgument, onclause: _OnClauseArgument | None = None, isouter: bool = False, full: bool = False) → _ORMJoin
```

生成左右子句之间的内部连接。

`join()` 是对由 `join()` 提供的核心连接接口的扩展，其中左右可选择的对象不仅可以是核心可选择对象（如 `Table`），还可以是映射类或 `AliasedClass` 实例。"on" 子句可以是 SQL 表达式，也可以是引用已配置的 `relationship()` 的 ORM 映射属性。

`join()` 在现代用法中不常用，因为其功能已封装在 `Select.join()` 和 `Query.join()` 方法中。这些方法的自动化程度远远超出了 `join()` 本身。在启用 ORM 的 SELECT 语句中明确使用 `join()`，需要使用 `Select.select_from()` 方法，示例如下：

```py
from sqlalchemy.orm import join
stmt = select(User).\
 select_from(join(User, Address, User.addresses)).\
 filter(Address.email_address=='foo@bar.com')
```

在现代 SQLAlchemy 中，以上连接可以更简洁地写为：

```py
stmt = select(User).\
 join(User.addresses).\
 filter(Address.email_address=='foo@bar.com')
```

警告

直接使用 `join()` 可能无法与现代 ORM 选项（如 `with_loader_criteria()`）正常工作。强烈建议在创建 ORM 连接时使用诸如 `Select.join()` 和 `Select.join_from()` 等方法提供的惯用连接模式。

另请参阅

连接 - 在 ORM 查询指南中了解惯用 ORM 连接模式的背景知识

```py
function sqlalchemy.orm.outerjoin(left: _FromClauseArgument, right: _FromClauseArgument, onclause: _OnClauseArgument | None = None, full: bool = False) → _ORMJoin
```

在左右子句之间产生一个左外连接。

这是 `join()` 函数的“外连接”版本，功能与其相同，只是生成了一个 OUTER JOIN。请参阅该函数的文档以获取其他用法细节。

```py
function sqlalchemy.orm.with_parent(instance: object, prop: attributes.QueryableAttribute[Any], from_entity: _EntityType[Any] | None = None) → ColumnElement[bool]
```

创建将此查询的主实体与给定的相关实例相关联的过滤条件，使用已建立的 `relationship()` 配置。

例如：

```py
stmt = select(Address).where(with_parent(some_user, User.addresses))
```

渲染的 SQL 与在给定父对象上的属性上触发惰性加载器时渲染的 SQL 相同，这意味着适当的状态从 Python 中的父对象中获取，而不需要在渲染的语句中渲染到父表的连接。

给定的属性也可以使用 `PropComparator.of_type()` 来指示条件的左侧：

```py
a1 = aliased(Address)
a2 = aliased(Address)
stmt = select(a1, a2).where(
 with_parent(u1, User.addresses.of_type(a2))
)
```

上述用法等同于使用 `from_entity()` 参数：

```py
a1 = aliased(Address)
a2 = aliased(Address)
stmt = select(a1, a2).where(
 with_parent(u1, User.addresses, from_entity=a2)
)
```

参数：

+   `instance` - 具有一些 `relationship()` 的实例。

+   `property` - 类绑定属性，指示应从实例使用哪个关系来协调父/子关系。

+   `from_entity` -

    要考虑为左侧的实体。这默认为 `Query` 本身的“零”实体。

    版本 1.2 中的新功能。### Populate Existing

`populate_existing` 执行选项确保，对于加载的所有行，`Session` 中对应的实例将被完全刷新 - 擦除对象中的任何现有数据（包括未决更改），并用从结果加载的数据替换。

示例用法如下：

```py
>>> stmt = select(User).execution_options(populate_existing=True)
>>> result = session.execute(stmt)
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
... 
```

通常，ORM 对象只加载一次，如果它们与后续结果行中的主键匹配，那么该行不会应用于对象。这既是为了保留对象上未决的未刷新更改，也是为了避免刷新已经存在的数据的开销和复杂性。`Session` 假定高度隔离的事务的默认工作模型，并且在事务中预计数据会在本地更改之外发生变化的程度上，这些用例将使用显式步骤来处理，例如这种方法。

使用 `populate_existing`，任何与查询匹配的对象集合都可以刷新，并且还允许控制关系加载器选项。例如，刷新一个实例同时刷新一组相关对象：

```py
stmt = (
    select(User)
    .where(User.name.in_(names))
    .execution_options(populate_existing=True)
    .options(selectinload(User.addresses))
)
# will refresh all matching User objects as well as the related
# Address objects
users = session.execute(stmt).scalars().all()
```

`populate_existing` 的另一个用例是支持各种属性加载功能，可以根据每个查询的情况更改如何加载属性。适用于此选项的选项包括：

+   `with_expression()` 选项

+   `PropComparator.and_()` 方法可以修改加载策略加载的内容

+   `contains_eager()` 选项

+   `with_loader_criteria()`选项

+   `load_only()`选项用于选择要刷新的属性

`populate_existing`执行选项等同于 1.x 风格 ORM 查询中的`Query.populate_existing()`方法。

另请参阅

我正在使用我的 Session 重新加载数据，但它没有看到我在其他地方提交的更改 - 在常见问题解答中

刷新/过期 - 在 ORM `Session` 文档中

### 自动刷新

当传递为`False`时，此选项将导致`Session`不调用“autoflush”步骤。这相当于使用`Session.no_autoflush`上下文管理器来禁用自动刷新：

```py
>>> stmt = select(User).execution_options(autoflush=False)
>>> session.execute(stmt)
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
... 
```

此选项也适用于启用 ORM 的`Update`和`Delete`查询。

`autoflush`执行选项等同于 1.x 风格 ORM 查询中的`Query.autoflush()`方法。

另请参阅

刷新

### 使用 Yield Per 获取大型结果集

`yield_per`执行选项是一个整数值，它将导致`Result`一次仅缓冲有限数量的行和/或 ORM 对象，然后再将数据提供给客户端。

通常，ORM 会立即获取**所有**行，为每一行构建 ORM 对象，并将这些对象组装到一个单一缓冲区中，然后将此缓冲区传递给`Result`对象作为要返回的行的来源。这种行为的理由是为了允许诸如连接的急切加载、结果的唯一化以及依赖于标识映射在获取时为结果集中的每个对象保持一致状态的结果处理逻辑等功能的正确行为。

`yield_per` 选项的目的是改变这种行为，使得 ORM 结果集对于迭代非常大的结果集（例如 > 10K 行）进行了优化，其中用户已确定上述模式不适用。当使用 `yield_per` 时，ORM 将把 ORM 结果分批到子集合中，并在迭代 `Result` 对象时，从每个子集合中分别产生行，这样 Python 解释器就不需要声明非常大的内存区域，这既耗时又导致内存使用过多。该选项影响数据库游标的使用方式，以及 ORM 构造行和对象以传递给 `Result` 的方式。

提示

由上可见，`Result` 必须以可迭代的方式被消耗，即使用迭代（如 `for row in result`）或使用部分行方法（如 `Result.fetchmany()` 或 `Result.partitions()`）。调用 `Result.all()` 将使使用 `yield_per` 的目的失败。

使用 `yield_per` 相当于同时利用了 `Connection.execution_options.stream_results` 执行选项，如果支持的话，选择使用后端使用服务器端游标，并且在返回的 `Result` 对象上使用 `Result.yield_per()` 方法，它建立了要提取的行的固定大小，以及一次构造多少个 ORM 对象的相应限制。

提示

`yield_per` 现在也作为一个 Core 执行选项可用，详细描述请参阅 使用服务器端游标（又名流式结果）。本节详细介绍了将 `yield_per` 作为 ORM `Session` 的执行选项使用的方法。该选项在两种情境下尽可能地行为相似。

当与 ORM 一起使用时，`yield_per` 必须通过给定语句上的 `Executable.execution_options()` 方法或通过将其传递给 `Session.execute()` 或其他类似的 `Session` 方法的 `Session.execute.execution_options` 参数来建立。如下是用于获取 ORM 对象的典型用法：

```py
>>> stmt = select(User).execution_options(yield_per=10)
>>> for user_obj in session.scalars(stmt):
...     print(user_obj)
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
[...]  ()
User(id=1, name='spongebob', fullname='Spongebob Squarepants')
User(id=2, name='sandy', fullname='Sandy Cheeks')
...
>>> # ... rows continue ...
```

上述代码等同于下面的示例，该示例使用了 `Connection.execution_options.stream_results` 和 `Connection.execution_options.max_row_buffer` 核心级别的执行选项，结合 `Result.yield_per()` 方法：

```py
# equivalent code
>>> stmt = select(User).execution_options(stream_results=True, max_row_buffer=10)
>>> for user_obj in session.scalars(stmt).yield_per(10):
...     print(user_obj)
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
[...]  ()
User(id=1, name='spongebob', fullname='Spongebob Squarepants')
User(id=2, name='sandy', fullname='Sandy Cheeks')
...
>>> # ... rows continue ...
```

`yield_per` 也常与 `Result.partitions()` 方法结合使用，该方法将在分组分区中迭代行。每个分区的大小默认为传递给 `yield_per` 的整数值，如下例所示：

```py
>>> stmt = select(User).execution_options(yield_per=10)
>>> for partition in session.scalars(stmt).partitions():
...     for user_obj in partition:
...         print(user_obj)
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
[...]  ()
User(id=1, name='spongebob', fullname='Spongebob Squarepants')
User(id=2, name='sandy', fullname='Sandy Cheeks')
...
>>> # ... rows continue ...
```

`yield_per` 执行选项**不兼容**于使用集合时的 “子查询”急切加载 或 “连接”急切加载。如果数据库驱动程序支持多个独立的游标，则它可能与 “select in”急切加载 兼容。

此外，`yield_per` 执行选项不兼容于 `Result.unique()` 方法；因为该方法依赖于为所有行存储完整的标识集，这必然会破坏使用 `yield_per` 的目的，即处理任意大量的行。

在版本 1.4.6 中更改：当使用`Result`对象获取 ORM 行时，如果同时使用`Result.unique()`过滤器以及`yield_per`执行选项，则会引发异常。

当使用传统的`Query`对象进行 1.x style ORM 使用时，`Query.yield_per()`方法将与`yield_per`执行选项具有相同的结果。

另请参阅

使用服务器端游标（又名流式结果）

### 身份令牌

深度炼金术

此选项是一个高级使用功能，主要用于与 Horizontal Sharding 扩展一起使用。对于从不同“shards”或分区加载具有相同主键的对象的典型情况，请首先考虑每个“shard”使用单独的`Session`对象。

“身份令牌”是一个任意值，可以与新加载对象的 identity key 相关联。此元素首先存在以支持执行按行“sharding”的扩展，其中对象可以从特定数据库表的任何数量的副本中加载，尽管它们具有重叠的主键值。 “身份令牌”的主要消费者是 Horizontal Sharding 扩展，它提供了一种在特定数据库表的多个“shards”之间持久化对象的通用框架。

`identity_token`执行选项可以根据每个查询直接影响此令牌。直接使用它，可以填充一个`Session`的多个对象实例，这些对象具有相同的主键和来源表，但具有不同的“身份”。

其中一个示例是使用 Schema Names 的翻译功能来填充一个`Session`，该功能可以影响查询范围内架构的选择，对象来自不同模式中的同名表。给定映射如下：

```py
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column

class Base(DeclarativeBase):
    pass

class MyTable(Base):
    __tablename__ = "my_table"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
```

上述类的默认“模式”名称为 `None`，意味着不会在 SQL 语句中写入模式限定符。但是，如果我们使用`Connection.execution_options.schema_translate_map`，将 `None` 映射到另一个模式，我们可以将 `MyTable` 的实例放入两个不同的模式中：

```py
engine = create_engine(
    "postgresql+psycopg://scott:tiger@localhost/test",
)

with Session(
    engine.execution_options(schema_translate_map={None: "test_schema"})
) as sess:
    sess.add(MyTable(name="this is schema one"))
    sess.commit()

with Session(
    engine.execution_options(schema_translate_map={None: "test_schema_2"})
) as sess:
    sess.add(MyTable(name="this is schema two"))
    sess.commit()
```

上述两个代码块创建一个`Session`对象，每次链接到不同的模式转换映射，并且 `MyTable` 的实例被持久化到 `test_schema.my_table` 和 `test_schema_2.my_table`。

上述的 `Session` 对象是独立的。如果我们想要在一个事务中持久化这两个对象，我们需要使用 水平分片 扩展来执行此操作。

然而，我们可以在一个会话中演示查询这些对象的方法如下：

```py
with Session(engine) as sess:
    obj1 = sess.scalar(
        select(MyTable)
        .where(MyTable.id == 1)
        .execution_options(
            schema_translate_map={None: "test_schema"},
            identity_token="test_schema",
        )
    )
    obj2 = sess.scalar(
        select(MyTable)
        .where(MyTable.id == 1)
        .execution_options(
            schema_translate_map={None: "test_schema_2"},
            identity_token="test_schema_2",
        )
    )
```

`obj1` 和 `obj2` 都彼此不同。但是，它们都指向 `MyTable` 类的主键 id 1，但是不同。这就是 `identity_token` 起作用的方式，我们可以在每个对象的检查中看到，其中我们查看 `InstanceState.key` 来查看两个不同的身份令牌：

```py
>>> from sqlalchemy import inspect
>>> inspect(obj1).key
(<class '__main__.MyTable'>, (1,), 'test_schema')
>>> inspect(obj2).key
(<class '__main__.MyTable'>, (1,), 'test_schema_2')
```

上述逻辑在使用 水平分片 扩展时会自动进行。

从版本 2.0.0rc1 开始新增： - 添加了 `identity_token` ORM 层执行选项。

另请参阅

水平分片 - 在 ORM 示例 部分。查看脚本 `separate_schema_translates.py`，演示了使用完整分片 API 的上述用例。

#### 从启用 ORM 的 SELECT 和 DML 语句中检查实体和列

`select()` 构造，以及 `insert()`、`update()` 和 `delete()` 构造（对于后两个 DML 构造，在 SQLAlchemy 1.4.33 中），都支持检查创建这些语句的实体，以及在结果集中返回的列和数据类型的能力。

对于`Select`对象，此信息可从`Select.column_descriptions`属性中获取。此属性的操作方式与传统的`Query.column_descriptions`属性相同。返回的格式是一个字典列表：

```py
>>> from pprint import pprint
>>> user_alias = aliased(User, name="user2")
>>> stmt = select(User, User.id, user_alias)
>>> pprint(stmt.column_descriptions)
[{'aliased': False,
 'entity': <class 'User'>,
 'expr': <class 'User'>,
 'name': 'User',
 'type': <class 'User'>},
 {'aliased': False,
 'entity': <class 'User'>,
 'expr': <....InstrumentedAttribute object at ...>,
 'name': 'id',
 'type': Integer()},
 {'aliased': True,
 'entity': <AliasedClass ...; User>,
 'expr': <AliasedClass ...; User>,
 'name': 'user2',
 'type': <class 'User'>}]
```

当`Select.column_descriptions`与非 ORM 对象一起使用，比如普通的`Table`或`Column`对象时，所有情况下返回的条目将包含关于各个列的基本信息：

```py
>>> stmt = select(user_table, address_table.c.id)
>>> pprint(stmt.column_descriptions)
[{'expr': Column('id', Integer(), table=<user_account>, primary_key=True, nullable=False),
 'name': 'id',
 'type': Integer()},
 {'expr': Column('name', String(), table=<user_account>, nullable=False),
 'name': 'name',
 'type': String()},
 {'expr': Column('fullname', String(), table=<user_account>),
 'name': 'fullname',
 'type': String()},
 {'expr': Column('id', Integer(), table=<address>, primary_key=True, nullable=False),
 'name': 'id_1',
 'type': Integer()}]
```

1.4.33 版本中的更改：当针对未启用 ORM 的`Select`使用时，`Select.column_descriptions`属性现在会返回一个值。以前，这会引发`NotImplementedError`。

对于`insert()`、`update()`和`delete()`构造，有两个单独的属性。一个是`UpdateBase.entity_description`，它返回有关主要 ORM 实体和数据库表的信息，该信息会受到 DML 构造的影响：

```py
>>> from sqlalchemy import update
>>> stmt = update(User).values(name="somename").returning(User.id)
>>> pprint(stmt.entity_description)
{'entity': <class 'User'>,
 'expr': <class 'User'>,
 'name': 'User',
 'table': Table('user_account', ...),
 'type': <class 'User'>}
```

提示

`UpdateBase.entity_description`包括一个条目``"table"``，它实际上是该语句要插入、更新或删除的**表**，这**并不总是**与类可能被映射到的 SQL“selectable”相同。例如，在联接表继承场景中，``"table"``将引用给定实体的本地表。

另一个是 `UpdateBase.returning_column_descriptions`，它以与 `Select.column_descriptions` 类似的方式提供有关 RETURNING 集合中存在的列的信息：

```py
>>> pprint(stmt.returning_column_descriptions)
[{'aliased': False,
 'entity': <class 'User'>,
 'expr': <sqlalchemy.orm.attributes.InstrumentedAttribute ...>,
 'name': 'id',
 'type': Integer()}]
```

新版本 1.4.33 中新增：添加了 `UpdateBase.entity_description` 和 `UpdateBase.returning_column_descriptions` 属性。  #### 附加的 ORM API 构造

| 对象名称 | 描述 |
| --- | --- |
| aliased(element[, alias, name, flat, ...]) | 生成给定元素的别名，通常是一个 `AliasedClass` 实例。 |
| AliasedClass | 代表一个与查询一起使用的映射类的“别名”形式。 |
| AliasedInsp | 为 `AliasedClass` 对象提供检查接口。 |
| Bundle | 一个由查询返回的 SQL 表达式的分组，位于一个命名空间下。 |
| join(left, right[, onclause, isouter, ...]) | 生成左右子句之间的内连接。 |
| outerjoin(left, right[, onclause, full]) | 生成左外连接 left 和 right 之间的连接。 |
| with_loader_criteria(entity_or_base, where_criteria[, loader_only, include_aliases, ...]) | 为特定实体的所有出现加载时添加额外的 WHERE 条件。 |
| with_parent(instance, prop[, from_entity]) | 创建过滤条件，将此查询的主要实体与给定的相关实例相关联，使用已建立的 `relationship()` 配置。 |

```py
function sqlalchemy.orm.aliased(element: _EntityType[_O] | FromClause, alias: FromClause | None = None, name: str | None = None, flat: bool = False, adapt_on_names: bool = False) → AliasedClass[_O] | FromClause | AliasedType[_O]
```

生成给定元素的别名，通常是一个 `AliasedClass` 实例。

例如：

```py
my_alias = aliased(MyClass)

stmt = select(MyClass, my_alias).filter(MyClass.id > my_alias.id)
result = session.execute(stmt)
```

`aliased()` 函数用于创建映射类到新可选择项的临时映射。默认情况下，从通常映射的可选择项（通常是一个 `Table` ）使用 `FromClause.alias()` 方法生成可选择项。然而，`aliased()` 还可以用于将类链接到新的 `select()` 语句。此外，`with_polymorphic()` 函数是 `aliased()` 的变体，旨在指定所谓的“多态可选择项”，它对应于一次性联接继承子类的联合。

为了方便起见，`aliased()` 函数还接受纯粹的`FromClause` 构造，比如`Table` 或 `select()` 构造。在这些情况下，调用对象的 `FromClause.alias()` 方法，并返回新的 `Alias` 对象。在这种情况下，返回的 `Alias` 对象不是 ORM 映射的。

参见

ORM 实体别名 - 在 SQLAlchemy 统一教程

选择 ORM 别名 - 在 ORM 查询指南

参数：

+   `element` – 要别名的元素。通常是一个映射类，但为了方便，也可以是一个`FromClause` 元素。

+   `alias` - 可选的可选择单元，将元素映射到该单元。这通常用于将对象链接到子查询，并且应该是从 `Query.subquery()` 方法或 `Select.subquery()` 或 `Select.alias()` 方法以及 `select()` 构造的结果中生成的别名选择构造。

+   `name` - 如果未由 `alias` 参数指定，则用于别名的可选字符串名称。名称，除其他外，形成了通过 `Query` 对象返回的元组可访问的属性名。不支持创建 `Join` 对象的别名时使用。

+   `flat` - 布尔值，将传递给 `FromClause.alias()` 调用，以便 `Join` 对象的别名将别名内部的各个表，而不是创建子查询。这通常由所有现代数据库支持，关于右嵌套连接，通常会生成更有效的查询。

+   `adapt_on_names` -

    如果为 True，则在将 ORM 实体的映射列映射到给定可选择的列时，将使用更自由的“匹配” - 如果给定可选择的没有与实体上的列对应的列，则将执行基于名称的匹配。这种情况的用例是将实体与一些派生的可选择相关联，例如使用聚合函数的可选择：

    ```py
    class UnitPrice(Base):
     __tablename__ = 'unit_price'
     ...
     unit_id = Column(Integer)
     price = Column(Numeric)

    aggregated_unit_price = Session.query(
     func.sum(UnitPrice.price).label('price')
     ).group_by(UnitPrice.unit_id).subquery()

    aggregated_unit_price = aliased(UnitPrice,
     alias=aggregated_unit_price, adapt_on_names=True)
    ```

    对于 `aggregated_unit_price` 上的函数，引用 `.price` 的将返回 `func.sum(UnitPrice.price).label('price')` 列，因为它与名称“price”匹配。通常，“price”函数不会与实际的 `UnitPrice.price` 列具有任何“列对应关系”，因为它不是原始列的代理。

```py
class sqlalchemy.orm.util.AliasedClass
```

表示映射类的“别名”形式，用于与查询一起使用。

ORM 等效于 `alias()` 构造的对象，该对象使用 `__getattr__` 方案模拟映射类，并维护对实际 `Alias` 对象的引用。

`AliasedClass` 的一个主要目的是在 ORM 生成的 SQL 语句中作为一个替代，以便一个现有的映射实体可以在多个上下文中使用。一个简单的例子：

```py
# find all pairs of users with the same name
user_alias = aliased(User)
session.query(User, user_alias).\
 join((user_alias, User.id > user_alias.id)).\
 filter(User.name == user_alias.name)
```

`AliasedClass` 还能够将一个现有的映射类映射到一个全新的可选项，前提是该可选项与现有的映射可选项兼容，并且还可以在映射中配置为 `relationship()` 的目标。请参考下面的链接获取示例。

`AliasedClass` 对象通常使用 `aliased()` 函数构建。在使用 `with_polymorphic()` 函数时，还可以进行附加配置。

结果对象是 `AliasedClass` 的一个实例。该对象实现了一个属性方案，生成与原始映射类相同的属性和方法接口，使得 `AliasedClass` 可与在原始类上有效的任何属性技术兼容，包括混合属性（参见 混合属性）。

`AliasedClass` 可以通过 `inspect()` 进行检查，以获取其底层的 `Mapper`、别名可选项等信息：

```py
from sqlalchemy import inspect
my_alias = aliased(MyClass)
insp = inspect(my_alias)
```

检查结果对象是 `AliasedInsp` 的一个实例。

另请参见

`aliased()`

`with_polymorphic()`

与别名类的关系

使用窗口函数限制行关系

**类签名**

类 `sqlalchemy.orm.AliasedClass` (`sqlalchemy.inspection.Inspectable`, `sqlalchemy.orm.ORMColumnsClauseRole`)

```py
class sqlalchemy.orm.util.AliasedInsp
```

为 `AliasedClass` 对象提供检查接口。

给定`AliasedClass`，使用`inspect()`函数返回`AliasedInsp`对象：

```py
from sqlalchemy import inspect
from sqlalchemy.orm import aliased

my_alias = aliased(MyMappedClass)
insp = inspect(my_alias)
```

`AliasedInsp`的属性包括：

+   `entity` - 表示的`AliasedClass`。

+   `mapper` - 映射底层类的`Mapper`。

+   `selectable` - 最终表示别名`Table`或`Select`构造的`Alias`构造。

+   `name` - 别名的名称。当从`Query`返回结果元组时，也用作属性名称。

+   `with_polymorphic_mappers` - 指示选择构造中表示所有这些映射的`Mapper`对象集合，用于`AliasedClass`。

+   `polymorphic_on` - 用作多态加载的“鉴别器”的备用列或 SQL 表达式。

另请参阅

运行时检查 API

**类签名**

类 `sqlalchemy.orm.AliasedInsp` (`sqlalchemy.orm.ORMEntityColumnsClauseRole`, `sqlalchemy.orm.ORMFromClauseRole`, `sqlalchemy.sql.cache_key.HasCacheKey`, `sqlalchemy.orm.base.InspectionAttr`, `sqlalchemy.util.langhelpers.MemoizedSlots`, `sqlalchemy.inspection.Inspectable`, `typing.Generic`)

```py
class sqlalchemy.orm.Bundle
```

由一个命名空间下的`Query`返回的 SQL 表达式分组。

`Bundle`基本上允许通过简单的子类化来嵌套由基于列的`Query`对象返回的基于元组的结果。它还可以通过简单的子类化来扩展，其中要重写的主要功能是如何返回表达式集，允许进行后处理以及自定义返回类型，而无需涉及 ORM 身份映射的类。

另请参阅

使用 Bundle 对选定的属性进行分组

**成员**

__init__(), c, columns, create_row_processor(), is_aliased_class, is_bundle, is_clause_element, is_mapper, label(), single_entity

**类签名**

class `sqlalchemy.orm.Bundle` (`sqlalchemy.orm.ORMColumnsClauseRole`, `sqlalchemy.sql.annotation.SupportsCloneAnnotations`, `sqlalchemy.sql.cache_key.MemoizedHasCacheKey`, `sqlalchemy.inspection.Inspectable`, `sqlalchemy.orm.base.InspectionAttr`)

```py
method __init__(name: str, *exprs: _ColumnExpressionArgument[Any], **kw: Any)
```

构建一个新的 `Bundle`。

例如：

```py
bn = Bundle("mybundle", MyClass.x, MyClass.y)

for row in session.query(bn).filter(
 bn.c.x == 5).filter(bn.c.y == 4):
 print(row.mybundle.x, row.mybundle.y)
```

参数：

+   `name` – bundle 的名称。

+   `*exprs` – 组成 bundle 的列或 SQL 表达式。

+   `single_entity=False` – 如果为 True，则此 `Bundle` 的行可以作为“单个实体”返回，方式与映射实体相同，不在任何封闭元组之外。

```py
attribute c: ReadOnlyColumnCollection[str, KeyedColumnElement[Any]]
```

`Bundle.columns` 的别名。

```py
attribute columns: ReadOnlyColumnCollection[str, KeyedColumnElement[Any]]
```

此 `Bundle` 所引用的 SQL 表达式的命名空间。

> 例如：
> 
> ```py
> bn = Bundle("mybundle", MyClass.x, MyClass.y)
> 
> q = sess.query(bn).filter(bn.c.x == 5)
> ```
> 
> 还支持 bundle 的嵌套：
> 
> ```py
> b1 = Bundle("b1",
>  Bundle('b2', MyClass.a, MyClass.b),
>  Bundle('b3', MyClass.x, MyClass.y)
>  )
> 
> q = sess.query(b1).filter(
>  b1.c.b2.c.a == 5).filter(b1.c.b3.c.y == 9)
> ```

另请参阅

`Bundle.c`

```py
method create_row_processor(query: Select[Any], procs: Sequence[Callable[[Row[Any]], Any]], labels: Sequence[str]) → Callable[[Row[Any]], Any]
```

生成此 `Bundle` 的“行处理”函数。

可以被子类重写以在获取结果时提供自定义行为。该方法在查询执行时传递了语句对象和一组“行处理器”函数；这些处理器函数在给定结果行时将返回单个属性值，然后可以将其适应为任何类型的返回数据结构。

下面的示例说明了将通常的 `Row` 返回结构替换为直接的 Python 字典：

```py
from sqlalchemy.orm import Bundle

class DictBundle(Bundle):
 def create_row_processor(self, query, procs, labels):
 'Override create_row_processor to return values as
 dictionaries'

 def proc(row):
 return dict(
 zip(labels, (proc(row) for proc in procs))
 )
 return proc
```

上述 `Bundle` 的结果将返回字典值：

```py
bn = DictBundle('mybundle', MyClass.data1, MyClass.data2)
for row in session.execute(select(bn)).where(bn.c.data1 == 'd1'):
 print(row.mybundle['data1'], row.mybundle['data2'])
```

```py
attribute is_aliased_class = False
```

如果此对象是 `AliasedClass` 的实例，则为 True。

```py
attribute is_bundle = True
```

如果此对象是 `Bundle` 的实例，则为 True。

```py
attribute is_clause_element = False
```

如果此对象是 `ClauseElement` 的实例，则为 True。

```py
attribute is_mapper = False
```

如果此对象是 `Mapper` 的实例，则为 True。

```py
method label(name)
```

提供此 `Bundle` 的副本并传递一个新标签。

```py
attribute single_entity = False
```

如果为 True，则单个 Bundle 的查询将作为单个实体返回，而不是作为键元组中的元素。

```py
function sqlalchemy.orm.with_loader_criteria(entity_or_base: _EntityType[Any], where_criteria: _ColumnExpressionArgument[bool] | Callable[[Any], _ColumnExpressionArgument[bool]], loader_only: bool = False, include_aliases: bool = False, propagate_to_loaders: bool = True, track_closure_variables: bool = True) → LoaderCriteriaOption
```

为特定实体的所有出现添加额外的 WHERE 条件。

在 1.4 版本中新增。

`with_loader_criteria()`选项旨在向查询中的特定实体添加限制条件，**全局**地应用于实体在 SELECT 查询中的出现以及任何子查询、连接条件和关系加载中，包括急切加载和延迟加载器，而无需在查询的任何特定部分指定它。渲染逻辑使用与单表继承相同的系统，以确保某个鉴别器应用于表。

例如，使用 2.0 风格的查询，我们可以限制`User.addresses`集合的加载方式，无论使用何种加载方式：

```py
from sqlalchemy.orm import with_loader_criteria

stmt = select(User).options(
 selectinload(User.addresses),
 with_loader_criteria(Address, Address.email_address != 'foo'))
)
```

上述对`User.addresses`的“selectinload”将把给定的过滤条件应用于 WHERE 子句。

另一个例子，过滤将应用于连接的 ON 子句，在这个例子中使用 1.x 风格的查询：

```py
q = session.query(User).outerjoin(User.addresses).options(
 with_loader_criteria(Address, Address.email_address != 'foo'))
)
```

`with_loader_criteria()`的主要目的是在`SessionEvents.do_orm_execute()`事件处理程序中使用它，以确保以某种方式过滤特定实体的所有出现，例如过滤访问控制角色。它还可以用于应用关系加载的条件。在下面的例子中，我们可以对特定`Session`发出的所有查询应用一定的规则：

```py
session = Session(bind=engine)

@event.listens_for("do_orm_execute", session)
def _add_filtering_criteria(execute_state):

 if (
 execute_state.is_select
 and not execute_state.is_column_load
 and not execute_state.is_relationship_load
 ):
 execute_state.statement = execute_state.statement.options(
 with_loader_criteria(
 SecurityRole,
 lambda cls: cls.role.in_(['some_role']),
 include_aliases=True
 )
 )
```

在上面的例子中，`SessionEvents.do_orm_execute()`事件将拦截使用`Session`发出的所有查询。对于那些是 SELECT 语句且不是属性或关系加载的查询，将为查询添加自定义的`with_loader_criteria()`选项。`with_loader_criteria()`选项将用于给定语句，并将自动传播到所有从此查询派生的关系加载。

给定的 criteria 参数是一个接受`cls`参数的`lambda`。给定的类将扩展以包括所有映射的子类，本身不必是一个映射的类。

提示

当与`with_loader_criteria()`选项一起使用时，需要注意`with_loader_criteria()`仅影响查询中确定渲染的 SQL 的部分，即 WHERE 和 FROM 子句。`contains_eager()`选项不会影响 SELECT 语句的渲染，除了列子句外的其他部分，因此与`with_loader_criteria()`选项没有任何交互。然而，“工作”的方式是`contains_eager()`旨在与已经以某种方式从其他实体进行选择的查询一起使用，而`with_loader_criteria()`可以应用其额外的条件。

在下面的示例中，假设一个映射关系为`A -> A.bs -> B`，给定的`with_loader_criteria()`选项将影响 JOIN 的渲染方式：

```py
stmt = select(A).join(A.bs).options(
 contains_eager(A.bs),
 with_loader_criteria(B, B.flag == 1)
)
```

在上面的例子中，给定的`with_loader_criteria()`选项将影响由`.join(A.bs)`指定的 JOIN 的 ON 子句，因此会按预期应用。`contains_eager()`选项会导致`B`的列被添加到列子句中：

```py
SELECT
 b.id, b.a_id, b.data, b.flag,
 a.id AS id_1,
 a.data AS data_1
FROM a JOIN b ON a.id = b.a_id AND b.flag = :flag_1
```

在上述语句中使用`contains_eager()`选项对`with_loader_criteria()`选项的行为没有影响。如果省略`contains_eager()`选项，则 SQL 将与 FROM 和 WHERE 子句相关，而`with_loader_criteria()`将继续将其条件添加到 JOIN 的 ON 子句中。添加`contains_eager()`仅会影响列子句，即会添加对`b`的额外列，然后 ORM 会使用这些列来生成`B`实例。

警告

在调用 `with_loader_criteria()` 内部的 lambda 中，每个唯一类只调用一次 **lambda**。自定义函数不应在此 lambda 内部调用。有关“lambda SQL”功能的概述，请参阅使用 Lambda 为语句生成添加显著的速度增益，该功能仅供高级使用。

参数：

+   `entity_or_base` – 一个映射类，或者是一组特定映射类的超类，规则将适用于这些类。

+   `where_criteria` –

    一个核心 SQL 表达式，应用限制条件。当给定类是具有许多不同映射子类的基类时，这也可以是“lambda:”或 Python 函数，接受目标类作为参数。

    注意

    为了支持 pickling，使用模块级 Python 函数来生成 SQL 表达式，而不是 lambda 或固定的 SQL 表达式，后者往往不可 picklable。

+   `include_aliases` – 如果为 True，则也将规则应用于 `aliased()` 构造。

+   `propagate_to_loaders` –

    默认为 True，适用于关系加载器，例如延迟加载器。这表示选项对象本身，包括 SQL 表达式，将与每个加载的实例一起传递。设置为 `False` 以防止对象分配给各个实例。

    另请参阅

    ORM 查询事件 - 包括使用 `with_loader_criteria()` 的示例。

    添加全局 WHERE / ON 条件 - 如何将 `with_loader_criteria()` 与 `SessionEvents.do_orm_execute()` 事件结合的基本示例。

+   `track_closure_variables` –

    当 False 时，lambda 表达式内部的闭包变量不会用作任何缓存键的一部分。这允许在 lambda 表达式内部使用更复杂的表达式，但需要 lambda 确保每次给定特定类时返回相同的 SQL。

    新版本 1.4.0b2 中新增。

```py
function sqlalchemy.orm.join(left: _FromClauseArgument, right: _FromClauseArgument, onclause: _OnClauseArgument | None = None, isouter: bool = False, full: bool = False) → _ORMJoin
```

生成左右子句之间的内部连接。

`join()`是对`join()`提供的核心连接接口的扩展，其中左右可选择的对象不仅可以是核心可选择的对象，如`Table`，还可以是映射类或`AliasedClass`实例。"on"子句可以是 SQL 表达式，也可以是引用已配置的`relationship()`的 ORM 映射属性。

`join()` 在现代用法中通常不需要，因为其功能已经封装在`Select.join()`和`Query.join()`方法中。这些方法比单独使用`join()`具有更多的自动化功能。在启用 ORM 的 SELECT 语句中显式使用`join()`，需要使用`Select.select_from()`方法，如下所示：

```py
from sqlalchemy.orm import join
stmt = select(User).\
 select_from(join(User, Address, User.addresses)).\
 filter(Address.email_address=='foo@bar.com')
```

在现代的 SQLAlchemy 中，上述连接可以更简洁地编写为：

```py
stmt = select(User).\
 join(User.addresses).\
 filter(Address.email_address=='foo@bar.com')
```

警告

直接使用`join()`可能无法与现代 ORM 选项（如`with_loader_criteria()`）正常工作。强烈建议在创建 ORM 连接时使用诸如`Select.join()`和`Select.join_from()`等方法提供的惯用连接模式。

另请参阅

连接 - 了解 ORM 连接模式的背景知识

```py
function sqlalchemy.orm.outerjoin(left: _FromClauseArgument, right: _FromClauseArgument, onclause: _OnClauseArgument | None = None, full: bool = False) → _ORMJoin
```

生成左外连接(left outer join)左边和右边子句之间的关联。

这是`join()`函数的“外连接”版本，具有相同的行为，只是生成了 OUTER JOIN。请参阅该函数的文档以获取其他用法细节。

```py
function sqlalchemy.orm.with_parent(instance: object, prop: attributes.QueryableAttribute[Any], from_entity: _EntityType[Any] | None = None) → ColumnElement[bool]
```

创建过滤条件，将此查询的主实体与给定的相关实例相关联，使用已建立的`relationship()`配置。

例如：

```py
stmt = select(Address).where(with_parent(some_user, User.addresses))
```

渲染的 SQL 与在给定父对象上的惰性加载程序触发时所渲染的 SQL 相同，这意味着在 Python 中从父对象中取得适当的状态而无需将父表的联接渲染到渲染的语句中。

给定的属性也可以使用 `PropComparator.of_type()` 来指示条件的左侧：

```py
a1 = aliased(Address)
a2 = aliased(Address)
stmt = select(a1, a2).where(
 with_parent(u1, User.addresses.of_type(a2))
)
```

上述用法等同于使用 `from_entity()` 参数：

```py
a1 = aliased(Address)
a2 = aliased(Address)
stmt = select(a1, a2).where(
 with_parent(u1, User.addresses, from_entity=a2)
)
```

参数：

+   `instance` – 一个具有一些 `relationship()` 的实例。

+   `property` – 类绑定属性，表示应该使用实例中的哪种关系来协调父/子关系。

+   `from_entity` –

    要考虑为左侧的实体。默认为 `Query` 本身的“零”实体。

    新版 1.2 中新增。#### 从 ORM 启用的 SELECT 和 DML 语句中检查实体和列

`select()` 构造，以及 `insert()`、`update()` 和 `delete()` 构造（对于后两个 DML 构造，在 SQLAlchemy 1.4.33 中），都支持检查这些语句所针对的实体，以及将在结果集中返回的列和数据类型。

对于 `Select` 对象，此信息可从 `Select.column_descriptions` 属性获得。此属性的操作方式与传统的 `Query.column_descriptions` 属性相同。返回的格式是一个字典列表：

```py
>>> from pprint import pprint
>>> user_alias = aliased(User, name="user2")
>>> stmt = select(User, User.id, user_alias)
>>> pprint(stmt.column_descriptions)
[{'aliased': False,
 'entity': <class 'User'>,
 'expr': <class 'User'>,
 'name': 'User',
 'type': <class 'User'>},
 {'aliased': False,
 'entity': <class 'User'>,
 'expr': <....InstrumentedAttribute object at ...>,
 'name': 'id',
 'type': Integer()},
 {'aliased': True,
 'entity': <AliasedClass ...; User>,
 'expr': <AliasedClass ...; User>,
 'name': 'user2',
 'type': <class 'User'>}]
```

当 `Select.column_descriptions` 与非 ORM 对象一起使用，如普通的 `Table` 或 `Column` 对象时，条目将在所有情况下包含有关返回的各个列的基本信息：

```py
>>> stmt = select(user_table, address_table.c.id)
>>> pprint(stmt.column_descriptions)
[{'expr': Column('id', Integer(), table=<user_account>, primary_key=True, nullable=False),
 'name': 'id',
 'type': Integer()},
 {'expr': Column('name', String(), table=<user_account>, nullable=False),
 'name': 'name',
 'type': String()},
 {'expr': Column('fullname', String(), table=<user_account>),
 'name': 'fullname',
 'type': String()},
 {'expr': Column('id', Integer(), table=<address>, primary_key=True, nullable=False),
 'name': 'id_1',
 'type': Integer()}]
```

版本 1.4.33 中的更改：当用于未启用 ORM 的 `Select` 时，`Select.column_descriptions` 属性现在返回一个值。先前，这会引发 `NotImplementedError`。

对于`insert()`，`update()` 和 `delete()` 构造，存在两个单独的属性。一个是`UpdateBase.entity_description`，它返回有关 DML 构造将影响的主要 ORM 实体和数据库表的信息：

```py
>>> from sqlalchemy import update
>>> stmt = update(User).values(name="somename").returning(User.id)
>>> pprint(stmt.entity_description)
{'entity': <class 'User'>,
 'expr': <class 'User'>,
 'name': 'User',
 'table': Table('user_account', ...),
 'type': <class 'User'>}
```

Tip

`UpdateBase.entity_description` 包括一个条目 `"table"`，实际上是语句要插入、更新或删除的**表**，这通常**不**与类可能被映射到的 SQL "selectable" 相同。例如，在联合表继承场景中，`"table"` 将引用给定实体的本地表。

另一个是 `UpdateBase.returning_column_descriptions`，它以一种与`Select.column_descriptions`大致相似的方式提供了有关 RETURNING 集合中存在的列的信息：

```py
>>> pprint(stmt.returning_column_descriptions)
[{'aliased': False,
 'entity': <class 'User'>,
 'expr': <sqlalchemy.orm.attributes.InstrumentedAttribute ...>,
 'name': 'id',
 'type': Integer()}]
```

版本 1.4.33 中的新内容：增加了 `UpdateBase.entity_description` 和 `UpdateBase.returning_column_descriptions` 属性。

#### 其他 ORM API 构造

| 对象名称 | 描述 |
| --- | --- |
| aliased(element[, alias, name, flat, ...]) | 生成给定元素的别名，通常是 `AliasedClass` 实例。 |
| AliasedClass | 代表映射类的“别名”形式，可用于查询。 |
| AliasedInsp | 为`AliasedClass`对象提供检查接口。 |
| Bundle | 由一个命名空间下的一个`Query`返回的 SQL 表达式组合。 |
| join(left, right[, onclause, isouter, ...]) | 在左右子句之间产生内连接。 |
| outerjoin(left, right[, onclause, full]) | 在左右子句之间生成左外连接。 |
| with_loader_criteria(entity_or_base, where_criteria[, loader_only, include_aliases, ...]) | 为特定实体的所有出现添加额外的 WHERE 条件以加载。 |
| with_parent(instance, prop[, from_entity]) | 创建过滤条件，将此查询的主要实体与给定的相关实例关联起来，使用已建立的`relationship()`配置。 |

```py
function sqlalchemy.orm.aliased(element: _EntityType[_O] | FromClause, alias: FromClause | None = None, name: str | None = None, flat: bool = False, adapt_on_names: bool = False) → AliasedClass[_O] | FromClause | AliasedType[_O]
```

生成给定元素的别名，通常是一个`AliasedClass`实例。

例如：

```py
my_alias = aliased(MyClass)

stmt = select(MyClass, my_alias).filter(MyClass.id > my_alias.id)
result = session.execute(stmt)
```

`aliased()`函数用于创建将映射类映射到新可选择项的临时映射。 默认情况下，可选择项是使用`FromClause.alias()`方法从通常映射的可选择项（通常是`Table`）生成的。但是，`aliased()`也可以用于将类链接到新的`select()`语句。 此外，`with_polymorphic()`函数是`aliased()`的变体，旨在指定所谓的“多态可选择项”，该可选择项对应于一次性联接继承子类的联合。

为方便起见，`aliased()` 函数还接受普通的`FromClause`构造，例如`Table`或`select()`构造。在这些情况下，该对象上调用`FromClause.alias()`方法，并返回新的`Alias`对象。在这种情况下，返回的`Alias`不是 ORM 映射的。

另请参阅

ORM 实体别名 - 在 SQLAlchemy 统一教程中

选择 ORM 别名 - 在 ORM 查询指南中

参数：

+   `element` - 要别名化的元素。通常是一个映射的类，但出于方便起见，也可以是一个`FromClause`元素。

+   `alias` - 可选的可选择单元，将元素映射到该单元。这通常用于将对象链接到子查询，并且应该是一个别名选择构造，就像从`Query.subquery()`方法或`Select.subquery()`或`Select.alias()`方法从`select()`构造中产生的那样。

+   `name` - 可选的字符串名称，用于别名，如果未由`alias`参数指定。该名称，除其他外，形成了由`Query`对象返回的元组访问的属性名称。在创建`Join`对象的别名时不受支持。

+   `flat` – 布尔值，将传递给`FromClause.alias()`调用，以便将`Join`对象的别名别名为加入其中的各个表，而不是创建子查询。这通常由所有现代数据库支持，关于右嵌套连接通常会产生更有效的查询。

+   `adapt_on_names` –

    如果为 True，则在将 ORM 实体的映射列与给定可选择的列进行映射时将使用更宽松的“匹配” - 如果给定的可选择没有与实体上的列对应的列，则将执行基于名称的匹配。这种用例是当将实体与一些派生可选择关联时，例如使用聚合函数的可选择：

    ```py
    class UnitPrice(Base):
     __tablename__ = 'unit_price'
     ...
     unit_id = Column(Integer)
     price = Column(Numeric)

    aggregated_unit_price = Session.query(
     func.sum(UnitPrice.price).label('price')
     ).group_by(UnitPrice.unit_id).subquery()

    aggregated_unit_price = aliased(UnitPrice,
     alias=aggregated_unit_price, adapt_on_names=True)
    ```

    上面，对`aggregated_unit_price`上的函数引用`.price`将返回`func.sum(UnitPrice.price).label('price')`列，因为它与名称“price”匹配。通常情况下，“price”函数不会与实际的`UnitPrice.price`列有任何“列对应”，因为它不是原始列的代理。

```py
class sqlalchemy.orm.util.AliasedClass
```

表示用于与查询一起使用的映射类的“别名”形式。

ORM 中的`alias()`构造的等价物，此对象使用`__getattr__`方案模仿映射类，并维护对真实`Alias`对象的引用。

`AliasedClass`的一个主要目的是在 ORM 生成的 SQL 语句中作为一个替代品，使得现有的映射实体可以在多个上下文中使用。一个简单的例子：

```py
# find all pairs of users with the same name
user_alias = aliased(User)
session.query(User, user_alias).\
 join((user_alias, User.id > user_alias.id)).\
 filter(User.name == user_alias.name)
```

`AliasedClass`还能够将现有的映射类映射到一个全新的可选择项，只要此可选择项与现有的映射可选择项兼容，并且它还可以被配置为`relationship()`的目标。请参阅下面的链接获取示例。

`AliasedClass`对象通常使用`aliased()`函数构造。当使用`with_polymorphic()`函数时，还会使用附加配置生成该对象。

结果对象是 `AliasedClass` 的一个实例。此对象实现了与原始映射类相同的属性和方法接口，允许 `AliasedClass` 兼容任何在原始类上工作的属性技术，包括混合属性（参见混合属性）。

可以使用 `inspect()` 检查 `AliasedClass` 的底层 `Mapper`、别名可选项和其他信息：

```py
from sqlalchemy import inspect
my_alias = aliased(MyClass)
insp = inspect(my_alias)
```

结果检查对象是 `AliasedInsp` 的一个实例。

另请参阅

`aliased()`

`with_polymorphic()`

与别名类的关系

带窗口函数的行限制关系

**类签名**

类 `sqlalchemy.orm.AliasedClass`（`sqlalchemy.inspection.Inspectable`，`sqlalchemy.orm.ORMColumnsClauseRole`）

```py
class sqlalchemy.orm.util.AliasedInsp
```

为 `AliasedClass` 对象提供检查接口。

使用 `inspect()` 函数给定 `AliasedClass` 返回 `AliasedInsp` 对象：

```py
from sqlalchemy import inspect
from sqlalchemy.orm import aliased

my_alias = aliased(MyMappedClass)
insp = inspect(my_alias)
```

`AliasedInsp` 上的属性包括：

+   `entity` - 代表的 `AliasedClass`。

+   `mapper` - `Mapper` 映射了底层类。

+   `selectable` - 最终表示别名的 `Alias` 构造或 `Select` 构造。

+   `name` - 别名的名称。当从 `Query` 中的结果元组中返回时，也用作属性名称。

+   `with_polymorphic_mappers` - 包含表示选择结构中所有那些表示的 `Mapper` 对象的集合，用于 `AliasedClass`。

+   `polymorphic_on` - 作为多态加载的“鉴别器”使用的替代列或 SQL 表达式。

另请参见

运行时检测 API

**类签名**

类 `sqlalchemy.orm.AliasedInsp` (`sqlalchemy.orm.ORMEntityColumnsClauseRole`, `sqlalchemy.orm.ORMFromClauseRole`, `sqlalchemy.sql.cache_key.HasCacheKey`, `sqlalchemy.orm.base.InspectionAttr`, `sqlalchemy.util.langhelpers.MemoizedSlots`, `sqlalchemy.inspection.Inspectable`, `typing.Generic`)

```py
class sqlalchemy.orm.Bundle
```

一组由 `Query` 返回的 SQL 表达式，在一个命名空间下。

`Bundle` 实质上允许嵌套列导向 `Query` 对象返回的基于元组的结果。它还可以通过简单的子类扩展，其中主要的重写功能是如何返回表达式集，允许后处理以及自定义返回类型，而不涉及 ORM 身份映射类。

另请参见

使用 Bundles 分组选择的属性

**成员**

__init__(), c, columns, create_row_processor(), is_aliased_class, is_bundle, is_clause_element, is_mapper, label(), single_entity

**类签名**

类 `sqlalchemy.orm.Bundle` (`sqlalchemy.orm.ORMColumnsClauseRole`, `sqlalchemy.sql.annotation.SupportsCloneAnnotations`, `sqlalchemy.sql.cache_key.MemoizedHasCacheKey`, `sqlalchemy.inspection.Inspectable`, `sqlalchemy.orm.base.InspectionAttr`)

```py
method __init__(name: str, *exprs: _ColumnExpressionArgument[Any], **kw: Any)
```

构造一个新的 `Bundle`。

例如：

```py
bn = Bundle("mybundle", MyClass.x, MyClass.y)

for row in session.query(bn).filter(
 bn.c.x == 5).filter(bn.c.y == 4):
 print(row.mybundle.x, row.mybundle.y)
```

参数:

+   `name` – bundle 的名称。

+   `*exprs` – 组成 bundle 的列或 SQL 表达式。

+   `single_entity=False` – 如果为 True，则此 `Bundle` 的行可以像映射实体一样在任何封闭元组之外返回。

```py
attribute c: ReadOnlyColumnCollection[str, KeyedColumnElement[Any]]
```

`Bundle.columns` 的别名。

```py
attribute columns: ReadOnlyColumnCollection[str, KeyedColumnElement[Any]]
```

此 `Bundle` 引用的 SQL 表达式的命名空间。

> 例如：
> 
> ```py
> bn = Bundle("mybundle", MyClass.x, MyClass.y)
> 
> q = sess.query(bn).filter(bn.c.x == 5)
> ```
> 
> 支持嵌套捆绑：
> 
> ```py
> b1 = Bundle("b1",
>  Bundle('b2', MyClass.a, MyClass.b),
>  Bundle('b3', MyClass.x, MyClass.y)
>  )
> 
> q = sess.query(b1).filter(
>  b1.c.b2.c.a == 5).filter(b1.c.b3.c.y == 9)
> ```

请参见

`Bundle.c`

```py
method create_row_processor(query: Select[Any], procs: Sequence[Callable[[Row[Any]], Any]], labels: Sequence[str]) → Callable[[Row[Any]], Any]
```

生成此 `Bundle` 的“行处理”函数。

可以被子类覆盖以在获取结果时提供自定义行为。 方法在查询执行时传递语句对象和一组“行处理”函数；给定结果行时，这些处理函数将返回单个属性值，然后可以将其调整为任何类型的返回数据结构。

下面的示例说明了用直接的 Python 字典替换通常的 `Row` 返回结构：

```py
from sqlalchemy.orm import Bundle

class DictBundle(Bundle):
 def create_row_processor(self, query, procs, labels):
 'Override create_row_processor to return values as
 dictionaries'

 def proc(row):
 return dict(
 zip(labels, (proc(row) for proc in procs))
 )
 return proc
```

上述 `Bundle` 的结果将返回字典值：

```py
bn = DictBundle('mybundle', MyClass.data1, MyClass.data2)
for row in session.execute(select(bn)).where(bn.c.data1 == 'd1'):
 print(row.mybundle['data1'], row.mybundle['data2'])
```

```py
attribute is_aliased_class = False
```

如果此对象是 `AliasedClass` 的实例，则为 True。

```py
attribute is_bundle = True
```

如果此对象是 `Bundle` 的实例，则为 True。

```py
attribute is_clause_element = False
```

如果此对象是 `ClauseElement` 的实例，则为 True。

```py
attribute is_mapper = False
```

如果此对象是 `Mapper` 的实例，则为 True。

```py
method label(name)
```

提供此 `Bundle` 的副本并传递一个新的标签。

```py
attribute single_entity = False
```

如果为 True，则查询单个 Bundle 将返回单个实体，而不是键入元组中的元素。

```py
function sqlalchemy.orm.with_loader_criteria(entity_or_base: _EntityType[Any], where_criteria: _ColumnExpressionArgument[bool] | Callable[[Any], _ColumnExpressionArgument[bool]], loader_only: bool = False, include_aliases: bool = False, propagate_to_loaders: bool = True, track_closure_variables: bool = True) → LoaderCriteriaOption
```

为特定实体的所有出现添加额外的 WHERE 条件到加载中。

版本 1.4 中的新功能。

`with_loader_criteria()` 选项旨在向查询中的特定类型的实体添加限制条件，**全局**，这意味着它将应用于实体在 SELECT 查询中出现的方式以及在任何子查询、连接条件和关系加载中，包括急切加载和惰性加载，而无需在查询的任何特定部分指定它。 渲染逻辑使用与单表继承相同的系统来确保某个特定的鉴别器应用于表。

例如，使用 2.0 样式 查询，我们可以限制 `User.addresses` 集合的加载方式，而不管使用的加载类型：

```py
from sqlalchemy.orm import with_loader_criteria

stmt = select(User).options(
 selectinload(User.addresses),
 with_loader_criteria(Address, Address.email_address != 'foo'))
)
```

上述中，“selectinload” 用于 `User.addresses` 将应用给定的过滤条件到 WHERE 子句。

另一个示例，其中过滤将应用于连接的 ON 子句，在此示例中使用 1.x 样式 查询：

```py
q = session.query(User).outerjoin(User.addresses).options(
 with_loader_criteria(Address, Address.email_address != 'foo'))
)
```

`with_loader_criteria()`的主要目的是在`SessionEvents.do_orm_execute()`事件处理程序中使用它，以确保特定实体的所有出现都以某种方式被过滤，例如，为访问控制角色过滤。它还可以用于应用条件于关系加载。在下面的示例中，我们可以将一组特定规则应用于特定`Session`发出的所有查询：

```py
session = Session(bind=engine)

@event.listens_for("do_orm_execute", session)
def _add_filtering_criteria(execute_state):

 if (
 execute_state.is_select
 and not execute_state.is_column_load
 and not execute_state.is_relationship_load
 ):
 execute_state.statement = execute_state.statement.options(
 with_loader_criteria(
 SecurityRole,
 lambda cls: cls.role.in_(['some_role']),
 include_aliases=True
 )
 )
```

在上面的示例中，`SessionEvents.do_orm_execute()`事件将拦截使用`Session`发出的所有查询。对于那些是 SELECT 语句且不是属性或关系加载的查询，将为查询添加一个自定义的`with_loader_criteria()`选项。`with_loader_criteria()`选项将在给定语句中使用，并且还将自动传播到所有从此查询继承的关系加载中。

给定的 criteria 参数是一个接受 cls 参数的`lambda`。给定的类将扩展以包括所有映射的子类，本身不需要是映射的类。

提示

当在与`contains_eager()`加载器选项一起使用`with_loader_criteria()`选项时，重要的是要注意，`with_loader_criteria()`仅影响决定以何种方式呈现 SQL 的查询的部分，这涉及 WHERE 和 FROM 子句。 `contains_eager()`选项不影响 SELECT 语句在列之外的呈现，因此与`with_loader_criteria()`选项没有任何交互。但是，事情的“工作”方式是`contains_eager()`应该与某种已经选择额外实体的查询一起使用，而`with_loader_criteria()`可以应用其额外的条件。

在下面的示例中，假设映射关系为 `A -> A.bs -> B`，给定的 `with_loader_criteria()` 选项将影响 JOIN 的呈现方式：

```py
stmt = select(A).join(A.bs).options(
 contains_eager(A.bs),
 with_loader_criteria(B, B.flag == 1)
)
```

在上面的示例中，给定的 `with_loader_criteria()` 选项将影响由 `.join(A.bs)` 指定的 JOIN 的 ON 子句，因此按预期应用。`contains_eager()` 选项的效果是将 `B` 的列添加到列子句中：

```py
SELECT
 b.id, b.a_id, b.data, b.flag,
 a.id AS id_1,
 a.data AS data_1
FROM a JOIN b ON a.id = b.a_id AND b.flag = :flag_1
```

在上述语句中使用 `contains_eager()` 选项对 `with_loader_criteria()` 选项的行为没有影响。如果省略了 `contains_eager()` 选项，则 SQL 将与 FROM 和 WHERE 子句相同，其中 `with_loader_criteria()` 继续将其条件添加到 JOIN 的 ON 子句中。添加 `contains_eager()` 只影响列子句，即添加了针对 `B` 的其他列，然后 ORM 消耗这些列以生成 `B` 实例。

警告

在对 `with_loader_criteria()` 的调用内部使用 lambda 只会 **对每个唯一类调用一次**。自定义函数不应在此 lambda 内调用。有关“lambda SQL”功能的概述，请参阅使用 Lambdas 为语句生成带来显著速度提升，这仅供高级使用。

参数：

+   `entity_or_base` – 映射类，或者是一组特定映射类的超类，将应用规则到其中。

+   `where_criteria` –

    核心 SQL 表达式，应用限制条件。当给定类是一个具有许多不同映射子类的基类时，这也可以是一个“lambda:”或 Python 函数，接受目标类作为参数。

    注意

    为了支持 pickle，使用模块级 Python 函数生成 SQL 表达式，而不是 lambda 或固定的 SQL 表达式，后者往往不可 picklable。

+   `include_aliases` – 如果为 True，则也将规则应用于 `aliased()` 构造。

+   `propagate_to_loaders` –

    默认为 True，适用于关系加载器，如惰性加载器。这表示选项对象本身，包括 SQL 表达式，将随每个加载的实例一起传递。将其设置为 `False` 可防止将对象分配给各个实例。

    另请参阅

    ORM 查询事件 - 包括使用 `with_loader_criteria()` 的示例。

    添加全局 WHERE / ON 条件 - 如何将 `with_loader_criteria()` 与 `SessionEvents.do_orm_execute()` 事件结合的基本示例。

+   `track_closure_variables` -

    当为 False 时，lambda 表达式中的闭包变量将不会作为任何缓存键的一部分。这允许在 lambda 表达式中使用更复杂的表达式，但要求 lambda 确保每次给定特定类时返回相同的 SQL。

    新版本 1.4.0b2 中新增。

```py
function sqlalchemy.orm.join(left: _FromClauseArgument, right: _FromClauseArgument, onclause: _OnClauseArgument | None = None, isouter: bool = False, full: bool = False) → _ORMJoin
```

在左右子句之间产生内连接。

`join()` 是对 `join()` 提供的核心连接接口的扩展，其中左右可选择的对象不仅可以是核心可选择对象，如 `Table`，还可以是映射类或 `AliasedClass` 实例。"on" 子句可以是 SQL 表达式，也可以是引用已配置的 `relationship()` 的 ORM 映射属性。

在现代用法中，通常不常用 `join()`，因为其功能已封装在 `Select.join()` 和 `Query.join()` 方法中。这两种方法在自动化方面远远超出了 `join()` 本身。在启用 ORM 的 SELECT 语句中显式使用 `join()` 涉及使用 `Select.select_from()` 方法，如下所示：

```py
from sqlalchemy.orm import join
stmt = select(User).\
 select_from(join(User, Address, User.addresses)).\
 filter(Address.email_address=='foo@bar.com')
```

在现代 SQLAlchemy 中，上述连接可以更简洁地写为：

```py
stmt = select(User).\
 join(User.addresses).\
 filter(Address.email_address=='foo@bar.com')
```

警告

直接使用`join()`可能无法与现代 ORM 选项（如`with_loader_criteria()`）正常工作。强烈建议在创建 ORM 连接时使用由`Select.join()`和`Select.join_from()`等方法提供的成语连接模式。

另请参阅

连接 - 在 ORM 查询指南中了解成语连接模式的背景

```py
function sqlalchemy.orm.outerjoin(left: _FromClauseArgument, right: _FromClauseArgument, onclause: _OnClauseArgument | None = None, full: bool = False) → _ORMJoin
```

在左侧和右侧子句之间产生左外连接。

这是`join()`函数的“外连接”版本，具有相同的行为，只是生成了一个外连接。有关其他用法细节，请参阅该函数的文档。

```py
function sqlalchemy.orm.with_parent(instance: object, prop: attributes.QueryableAttribute[Any], from_entity: _EntityType[Any] | None = None) → ColumnElement[bool]
```

创建过滤条件，将此查询的主实体与给定的相关实例关联起来，使用已建立的`relationship()`配置。

例如：

```py
stmt = select(Address).where(with_parent(some_user, User.addresses))
```

渲染的 SQL 与在给定父对象上的惰性加载器触发时渲染的 SQL 相同，这意味着在 Python 中从父对象中获取适当的状态而无需在渲染语句中渲染到父表的连接。

给定属性还可以利用`PropComparator.of_type()`指示条件的左侧：

```py
a1 = aliased(Address)
a2 = aliased(Address)
stmt = select(a1, a2).where(
 with_parent(u1, User.addresses.of_type(a2))
)
```

上述用法等同于使用`from_entity()`参数：

```py
a1 = aliased(Address)
a2 = aliased(Address)
stmt = select(a1, a2).where(
 with_parent(u1, User.addresses, from_entity=a2)
)
```

参数：

+   `instance` – 具有某些`relationship()`的实例。

+   `property` – 类绑定属性，指示应使用实例中的哪个关系来协调父/子关系。

+   `from_entity` –

    要考虑为左侧的实体。默认为`Query`本身的“零”实体。

    版本 1.2 中的新功能。
