# 传统查询 API

> 原文：[`docs.sqlalchemy.org/en/20/orm/queryguide/query.html`](https://docs.sqlalchemy.org/en/20/orm/queryguide/query.html)

关于传统查询 API

本页包含了由 Python 生成的`Query`构造的文档，多年来这是与 SQLAlchemy ORM 一起使用时的唯一 SQL 接口。从版本 2.0 开始，现在采用的是全新的工作方式，其中与 Core 相同的`select()`构造对 ORM 同样有效，为构建查询提供了一致的接口。

对于在 SQLAlchemy 2.0 API 之前构建的任何应用程序，`Query` API 通常表示应用程序中绝大多数数据库访问代码，并且大部分`Query` API **不会从 SQLAlchemy 中删除**。在执行`Query`对象时，`Query`对象在幕后现在会将自己转换为 2.0 样式的`select()`对象，因此现在它只是一个非常薄的适配器 API。

要了解如何将基于`Query`的应用程序迁移到 2.0 样式，请参阅 2.0 迁移 - ORM 用法。

要了解如何以 2.0 样式编写 ORM 对象的 SQL，请从 SQLAlchemy 统一教程开始。2.0 样式查询的其他参考资料请参阅 ORM 查询指南。

## 查询对象

`Query`是根据给定的`Session`产生的，使用`Session.query()`方法：

```py
q = session.query(SomeMappedClass)
```

以下是`Query`对象的完整接口。

| 对象名称 | 描述 |
| --- | --- |
| 查询 | ORM 级别的 SQL 构造对象。 |

```py
class sqlalchemy.orm.Query
```

ORM 级别的 SQL 构造对象。

传统特性

ORM `Query`对象是 SQLAlchemy 2.0 的传统构造。请参阅传统查询 API 顶部的注释，其中包括迁移文档的链接。

`查询` 对象通常最初是使用 `Session.query()` 方法生成的，`Session` 的情况比较少是直接实例化 `Query` 并使用 `Query.with_session()` 方法与 `Session` 关联。

**成员**

__init__(), add_column(), add_columns(), add_entity(), all(), apply_labels(), as_scalar(), autoflush(), column_descriptions, correlate(), count(), cte(), delete(), distinct(), enable_assertions(), enable_eagerloads(), except_(), except_all(), execution_options(), exists(), filter(), filter_by(), first(), from_statement(), get(), get_children(), get_execution_options(), get_label_style, group_by(), having(), instances(), intersect(), intersect_all(), is_single_entity, join(), label(), lazy_loaded_from, limit(), merge_result(), offset(), one(), one_or_none(), only_return_tuples(), options(), order_by(), outerjoin(), params(), populate_existing(), prefix_with(), reset_joinpoint(), scalar(), scalar_subquery(), select_from(), selectable, set_label_style(), slice(), statement, subquery(), suffix_with(), tuples(), union(), union_all(), update(), value(), values(), where(), whereclause, with_entities(), with_for_update(), with_hint(), with_labels(), with_parent(), with_session(), with_statement_hint(), with_transformation(), yield_per()

**类签名**

类`sqlalchemy.orm.Query`（`sqlalchemy.sql.expression._SelectFromElements`，`sqlalchemy.sql.annotation.SupportsCloneAnnotations`，`sqlalchemy.sql.expression.HasPrefixes`，`sqlalchemy.sql.expression.HasSuffixes`，`sqlalchemy.sql.expression.HasHints`，`sqlalchemy.event.registry.EventTarget`，`sqlalchemy.log.Identified`，`sqlalchemy.sql.expression.Generative`，`sqlalchemy.sql.expression.Executable`，`typing.Generic`）

```py
method __init__(entities: _ColumnsClauseArgument[Any] | Sequence[_ColumnsClauseArgument[Any]], session: Session | None = None)
```

直接构造一个`Query`。

例如：

```py
q = Query([User, Address], session=some_session)
```

以上等价于：

```py
q = some_session.query(User, Address)
```

参数：

+   `entities` – 一个实体和/或 SQL 表达式的序列。

+   `session` – 与`Query`将关联的`Session`。可选；也可以通过`Query.with_session()`方法将`Query`与`Session`关联。

另请参见

`Session.query()`

`Query.with_session()`

```py
method add_column(column: _ColumnExpressionArgument[Any]) → Query[Any]
```

将列表达式添加到要返回的结果列列表中。

自版本 1.4 起已弃用：`Query.add_column()`已弃用，并将在将来的版本中删除。请使用 `Query.add_columns()`

```py
method add_columns(*column: _ColumnExpressionArgument[Any]) → Query[Any]
```

将一个或多个列表达式添加到要返回的结果列列表中。

另请参见

`Select.add_columns()` - v2 可比较的方法。

```py
method add_entity(entity: _EntityType[Any], alias: Alias | Subquery | None = None) → Query[Any]
```

将映射实体添加到要返回的结果列列表中。

另请参见

`Select.add_columns()` - v2 可比较的方法。

```py
method all() → List[_T]
```

将由此`Query`表示的结果返回为列表。

这将导致底层 SQL 语句的执行。

警告

当要求 `Query` 对象返回由完整的 ORM 映射实体组成的序列或迭代器时，将根据主键**对条目进行去重**。有关更多详情，请参阅 FAQ。

> 另请参阅
> 
> 我的查询返回的对象数量与 query.count() 告诉我的数量不一致 - 为什么？

另请参阅

`Result.all()` - v2 可比较方法。

`Result.scalars()` - v2 可比较方法。

```py
method apply_labels() → Self
```

自版本 2.0 弃用：`Query.with_labels()` 和 `Query.apply_labels()` 方法被视为 SQLAlchemy 1.x 系列的遗留构造，在 2.0 中成为遗留构造。请改用 `set_label_style(LABEL_STYLE_TABLENAME_PLUS_COL)`。 (有关 SQLAlchemy 2.0 的背景，请参阅：SQLAlchemy 2.0 - Major Migration Guide)

```py
method as_scalar() → ScalarSelect[Any]
```

返回由此 `Query` 表示的完整 SELECT 语句，转换为标量子查询。

自版本 1.4 弃用：`Query.as_scalar()` 方法已弃用，并将在将来的版本中删除。请参考 `Query.scalar_subquery()`。

```py
method autoflush(setting: bool) → Self
```

返回具有特定“autoflush”设置的查询。

自 SQLAlchemy 1.4 起，`Query.autoflush()` 方法等效于在 ORM 级别使用 `autoflush` 执行选项。有关此选项的更多背景，请参阅 Autoflush 部分。

```py
attribute column_descriptions
```

返回有关此 `Query` 将返回的列的元数据。

格式是一个字典列表：

```py
user_alias = aliased(User, name='user2')
q = sess.query(User, User.id, user_alias)

# this expression:
q.column_descriptions

# would return:
[
 {
 'name':'User',
 'type':User,
 'aliased':False,
 'expr':User,
 'entity': User
 },
 {
 'name':'id',
 'type':Integer(),
 'aliased':False,
 'expr':User.id,
 'entity': User
 },
 {
 'name':'user2',
 'type':User,
 'aliased':True,
 'expr':user_alias,
 'entity': user_alias
 }
]
```

另请参阅

此 API 也可使用 2.0 风格 查询，文档位于：

+   检查来自启用 ORM 的 SELECT 和 DML 语句的实体和列

+   `Select.column_descriptions`

```py
method correlate(*fromclauses: Literal[None, False] | FromClauseRole | Type[Any] | Inspectable[_HasClauseElement[Any]] | _HasClauseElement[Any]) → Self
```

返回一个 `Query` 构造，将给定的 FROM 子句与封闭的 `Query` 或 `select()` 关联起来。

此处的方法接受映射类、`aliased()` 构造和 `Mapper` 构造作为参数，这些参数会被解析为表达式构造，以及适当的表达式构造。

最终，相关参数将被强制转换为表达式构造，然后传递给 `Select.correlate()`。

在这种情况下，相关参数会生效，例如在使用 `Query.from_self()` 时，或者在将由`Query.subquery()`返回的子查询嵌入到另一个`select()` 构造中时。

另请参阅

`Select.correlate()` - v2 等效方法。

```py
method count() → int
```

返回此`Query`形成的 SQL 将返回的行数计数。

这将生成以下查询的 SQL 语句：

```py
SELECT count(1) AS count_1 FROM (
 SELECT <rest of query follows...>
) AS anon_1
```

上述 SQL 返回一个单行，即 count 函数的聚合值；然后`Query.count()` 方法返回该单个整数值。

警告

需要注意的是，count() 返回的值**并不等同于此 Query 通过 .all() 等方法返回的 ORM 对象数**。当 `Query` 对象被要求返回完整实体时，将根据主键**对条目进行重复消除**，这意味着如果相同的主键值在结果中出现超过一次，则只会存在一个该主键的对象。这不适用于针对单个列的查询。

另请参阅

我的查询的返回对象数与 query.count() 告诉我的不一样 - 为什么？

对于对特定列进行精细控制的计数，跳过子查询的使用或以其他方式控制 FROM 子句，或使用其他聚合函数，可以结合使用`expression.func`表达式和 `Session.query()`，例如：

```py
from sqlalchemy import func

# count User records, without
# using a subquery.
session.query(func.count(User.id))

# return count of user "id" grouped
# by "name"
session.query(func.count(User.id)).\
 group_by(User.name)

from sqlalchemy import distinct

# count distinct "name" values
session.query(func.count(distinct(User.name)))
```

另请参阅

2.0 迁移 - ORM 用法

```py
method cte(name: str | None = None, recursive: bool = False, nesting: bool = False) → CTE
```

返回由此`Query`表示的完整 SELECT 语句，表示为公共表达式（CTE）。

参数和用法与 `SelectBase.cte()` 方法相同；有关更多详细信息，请参阅该方法。

这里是 [PostgreSQL WITH RECURSIVE 示例](https://www.postgresql.org/docs/current/static/queries-with.html)。请注意，在此示例中，`included_parts` cte 和其 `incl_alias` 别名是核心可选择的，这意味着可以通过 `.c.` 属性访问列。`parts_alias` 对象是 `Part` 实体的 `aliased()` 实例，因此可以直接访问列映射属性：

```py
from sqlalchemy.orm import aliased

class Part(Base):
 __tablename__ = 'part'
 part = Column(String, primary_key=True)
 sub_part = Column(String, primary_key=True)
 quantity = Column(Integer)

included_parts = session.query(
 Part.sub_part,
 Part.part,
 Part.quantity).\
 filter(Part.part=="our part").\
 cte(name="included_parts", recursive=True)

incl_alias = aliased(included_parts, name="pr")
parts_alias = aliased(Part, name="p")
included_parts = included_parts.union_all(
 session.query(
 parts_alias.sub_part,
 parts_alias.part,
 parts_alias.quantity).\
 filter(parts_alias.part==incl_alias.c.sub_part)
 )

q = session.query(
 included_parts.c.sub_part,
 func.sum(included_parts.c.quantity).
 label('total_quantity')
 ).\
 group_by(included_parts.c.sub_part)
```

请参阅

`Select.cte()` - v2 等效方法。

```py
method delete(synchronize_session: SynchronizeSessionArgument = 'auto') → int
```

使用任意 WHERE 子句执行 DELETE。

从数据库中删除与此查询匹配的行。

例如：

```py
sess.query(User).filter(User.age == 25).\
 delete(synchronize_session=False)

sess.query(User).filter(User.age == 25).\
 delete(synchronize_session='evaluate')
```

警告

请参阅 ORM-Enabled INSERT、UPDATE 和 DELETE 语句 章节以了解重要的注意事项和警告，包括在使用映射器继承配置时批量 UPDATE 和 DELETE 的限制。

参数：

**synchronize_session** – 选择在会话中更新对象属性的策略。请参阅 ORM-Enabled INSERT、UPDATE 和 DELETE 语句 章节讨论这些策略。

返回：

数据库的“行计数”功能返回的匹配行数。

请参阅

ORM-Enabled INSERT、UPDATE 和 DELETE 语句

```py
method distinct(*expr: _ColumnExpressionArgument[Any]) → Self
```

对查询应用 `DISTINCT` 并返回新生成的 `Query`。

注意

ORM 级别的 `distinct()` 调用包含逻辑，将自动将查询的 ORDER BY 中的列添加到 SELECT 语句的列子句中，以满足数据库后端的常见需求，即在使用 DISTINCT 时，ORDER BY 列应作为 SELECT 列的一部分。然而，这些列 *不会* 添加到实际由 `Query` 获取的列列表中，因此不会影响结果。然而，在使用 `Query.statement` 访问器时，这些列会通过。

自版本 2.0 起已弃用：此逻辑已弃用，将在 SQLAlchemy 2.0 中删除。请参阅 使用 DISTINCT 与其他列，但仅选择实体 了解 2.0 中此用例的描述。

请参阅

`Select.distinct()` - v2 等效方法。

参数：

***expr** –

可选的列表达式。当存在时，PostgreSQL 方言将呈现 `DISTINCT ON (<expressions>)` 结构。

自 1.4 版本起已弃用：在其他方言中使用*expr 已弃用，并将在将来的版本中引发`CompileError`。

```py
method enable_assertions(value: bool) → Self
```

控制是否生成断言。

当设置为 False 时，返回的 Query 在某些操作之前不会断言其状态，包括调用 filter() 时未应用 LIMIT/OFFSET，调用 get() 时不存在条件，以及调用 filter()/order_by()/group_by() 等时不存在“from_statement()”。此更宽松的模式由自定义的 Query 子类使用，以指定标准或其他修改器在通常的使用模式之外。

应注意确保使用模式是可行的。例如，由 from_statement()应用的语句将覆盖由 filter()或 order_by()设置的任何条件。

```py
method enable_eagerloads(value: bool) → Self
```

控制是否呈现急切连接和子查询。

当设置为 False 时，返回的 Query 将不会渲染急切连接，无论 `joinedload()`、`subqueryload()` 选项或映射器级别的 `lazy='joined'`/`lazy='subquery'` 配置如何。

当将 Query 的语句嵌套到子查询或其他可选择项中时，或者当使用`Query.yield_per()`时主要用于。

```py
method except_(*q: Query) → Self
```

生成此 Query 对一项或多项查询的 EXCEPT。

与`Query.union()`的工作方式相同。请参阅该方法以获取用法示例。

另请参阅

`Select.except_()` - v2 等效方法。

```py
method except_all(*q: Query) → Self
```

生成此 Query 对一项或多项查询的 EXCEPT ALL。

与`Query.union()`的工作方式相同。请参阅该方法以获取用法示例。

另请参阅

`Select.except_all()` - v2 等效方法。

```py
method execution_options(**kwargs: Any) → Self
```

设置在执行期间生效的非 SQL 选项。

此处允许的选项包括所有被`Connection.execution_options()`接受的选项，以及一系列 ORM 特定选项：

`populate_existing=True` - 等效于使用`Query.populate_existing()`

`autoflush=True|False` - 等效于使用`Query.autoflush()`

`yield_per=<value>` - 等效于使用`Query.yield_per()`

注意，如果使用了`Query.yield_per()`方法或执行选项，则`stream_results`执行选项会自动启用。

版本 1.4 中的新功能：- 添加了 ORM 选项到`Query.execution_options()`

在使用 2.0 风格查询时，执行选项也可以在每次执行时指定，通过`Session.execution_options`参数。

警告

`Connection.execution_options.stream_results`参数不应在单个 ORM 语句执行的级别使用，因为`Session`不会跟踪来自不同模式转换映射的对象在单个会话中。对于单个`Session`范围内的多个模式转换映射，请参见水平分片。

另请参阅

使用服务器端游标（又名流式结果）

`Query.get_execution_options()`

`Select.execution_options()` - v2 等效方法。

```py
method exists() → Exists
```

一个方便的方法，将查询转换为形式为 EXISTS（SELECT 1 FROM … WHERE …）的 EXISTS 子查询。

例如：

```py
q = session.query(User).filter(User.name == 'fred')
session.query(q.exists())
```

生成类似于：

```py
SELECT EXISTS (
 SELECT 1 FROM users WHERE users.name = :name_1
) AS anon_1
```

EXISTS 构造通常用于 WHERE 子句中：

```py
session.query(User.id).filter(q.exists()).scalar()
```

请注意，某些数据库（如 SQL Server）不允许在 SELECT 的列子句中存在 EXISTS 表达式。要基于存在性选择简单的布尔值作为 WHERE，使用`literal()`：

```py
from sqlalchemy import literal

session.query(literal(True)).filter(q.exists()).scalar()
```

另请参阅

`Select.exists()` - v2 可比较的方法。

```py
method filter(*criterion: _ColumnExpressionArgument[bool]) → Self
```

将给定的过滤条件应用于此`Query`的副本，使用 SQL 表达式。

例如：

```py
session.query(MyClass).filter(MyClass.name == 'some name')
```

多个条件可以以逗号分隔的方式指定；效果是它们将使用`and_()`函数连接在一起：

```py
session.query(MyClass).\
 filter(MyClass.name == 'some name', MyClass.id > 5)
```

条件是适用于 select 的 WHERE 子句的任何 SQL 表达式对象。字符串表达式通过`text()`构造被强制转换为 SQL 表达式构造。

另请参阅

`Query.filter_by()` - 根据关键字表达式进行过滤。

`Select.where()` - v2 等效方法。

```py
method filter_by(**kwargs: Any) → Self
```

将给定的过滤条件应用于此`Query`的副本，使用关键字表达式。

例如：

```py
session.query(MyClass).filter_by(name = 'some name')
```

可以指定多个条件，以逗号分隔；其效果是它们将使用`and_()`函数连接在一起：

```py
session.query(MyClass).\
 filter_by(name = 'some name', id = 5)
```

关键字表达式是从查询的主要实体或最后一个曾被调用过`Query.join()`的目标实体中提取的。

另请参阅

`Query.filter()` - 根据 SQL 表达式进行过滤。

`Select.filter_by()` - v2 可比较的方法。

```py
method first() → _T | None
```

返回此`Query`的第一个结果，如果结果不包含任何行，则返回 None。

`first()`在生成的 SQL 中应用了一个限制为 1，因此仅在服务器端生成一个主要实体行（请注意，如果存在联接加载的集合，则可能由多个结果行组成）。

调用`Query.first()`会导致基础查询的执行。

另请参阅

`Query.one()`

`Query.one_or_none()`

`Result.first()` - v2 可比较的方法。

`Result.scalars()` - v2 可比较的方法。

```py
method from_statement(statement: ExecutableReturnsRows) → Self
```

执行给定的 SELECT 语句并返回结果。

此方法绕过所有内部语句编译，并且语句在不修改的情况下执行。

该语句通常是一个`text()`或`select()`结构，应返回与此`Query`所代表的实体类相对应的列集。

另请参阅

`Select.from_statement()` - v2 可比较的方法。

```py
method get(ident: _PKIdentityArgument) → Any | None
```

根据给定的主键标识符返回一个实例，如果找不到则返回`None`。

自版本 2.0 起已弃用：`Query.get()` 方法被认为是 SQLAlchemy 1.x 系列的遗留部分，并且在 2.0 中成为遗留构造。该方法现在可用作 `Session.get()`（关于 SQLAlchemy 2.0 的背景信息，请参阅：SQLAlchemy 2.0 - 主要迁移指南)

例如：

```py
my_user = session.query(User).get(5)

some_object = session.query(VersionedFoo).get((5, 10))

some_object = session.query(VersionedFoo).get(
 {"id": 5, "version_id": 10})
```

`Query.get()` 特殊之处在于它提供对所属 `Session` 的标识映射的直接访问。如果给定的主键标识符存在于本地标识映射中，则对象将直接从此集合返回，而不会发出任何 SQL，除非对象已被标记为完全过期。如果不存在，则执行 SELECT 来定位对象。

`Query.get()` 会检查对象是否存在于标识映射中并标记为过期 - 会发出一个 SELECT 来刷新对象并确保行仍然存在。如果不存在，则会引发 `ObjectDeletedError`。

`Query.get()` 仅用于返回单个映射实例，而不是多个实例或单个列构造，并且严格限于单个主键值。源 `Query` 必须以这种方式构造，即针对单个映射实体，没有额外的过滤条件。可以通过 `Query.options()` 应用加载选项，如果对象尚未在本地存在，则将使用该选项。

参数：

**ident** –

表示主键的标量、元组或字典。对于复合（例如，多列）主键，应传递元组或字典。

对于单列主键，标量调用形式通常最为便捷。如果一行的主键是值“5”，则调用如下所示：

```py
my_object = query.get(5)
```

元组形式包含主键值，通常按照它们对应于映射的 `Table` 对象的主键列的顺序，或者如果使用了 `Mapper.primary_key` 配置参数，则按照该参数的使用顺序。例如，如果一行的主键由整数数字“5, 10”表示，则调用如下所示：

```py
my_object = query.get((5, 10))
```

字典形式应该以键的形式包含对应于主键每个元素的映射属性名称。如果映射类具有 `id`、`version_id` 作为存储对象主键值的属性，则调用将如下所示：

```py
my_object = query.get({"id": 5, "version_id": 10})
```

新版本 1.3 中的 `Query.get()` 方法现在可选择性地接受属性名到值的字典，以指示主键标识符。

返回：

对象实例，或 `None`。

```py
method get_children(*, omit_attrs: Tuple[str, ...] = (), **kw: Any) → Iterable[HasTraverseInternals]
```

*继承自* `HasTraverseInternals.get_children()` *方法的* `HasTraverseInternals`

返回此 `HasTraverseInternals` 的即时子 `HasTraverseInternals` 元素。

这用于访问遍历。

**kw 可以包含改变返回集合的标志，例如为了减少更大的遍历而返回子集合中的项目，或者从不同的上下文中返回子项（例如模式级别的集合而不是从子句级别返回）。

```py
method get_execution_options() → _ImmutableExecuteOptions
```

获取在执行期间生效的非 SQL 选项。

新版本 1.3 中新增。

另请参阅

`Query.execution_options()`

`Select.get_execution_options()` - v2 可比较方法。

```py
attribute get_label_style
```

检索当前的标签样式。

新版本 1.4 中新增。

另请参阅

`Select.get_label_style()` - v2 等效方法。

```py
method group_by(_Query__first: Literal[None, False, _NoArg.NO_ARG] | _ColumnExpressionOrStrLabelArgument[Any] = _NoArg.NO_ARG, *clauses: _ColumnExpressionOrStrLabelArgument[Any]) → Self
```

将一个或多个 GROUP BY 准则应用于查询，并返回新生成的 `Query`。

所有现有的 GROUP BY 设置都可以通过传递 `None` 来抑制 - 这将抑制任何配置在映射器上的 GROUP BY。

另请参阅

这些部分描述了 GROUP BY，是以 2.0 样式 调用的，但也适用于 `Query`：

带有 GROUP BY / HAVING 的聚合函数 - 在 SQLAlchemy 统一教程 中

按标签排序或分组 - 在 SQLAlchemy 统一教程 中

`Select.group_by()` - v2 等效方法。

```py
method having(*having: _ColumnExpressionArgument[bool]) → Self
```

将 HAVING 准则应用于查询，并返回新生成的 `Query`。

`Query.having()` 与 `Query.group_by()` 结合使用。

HAVING 条件使得可以在聚合函数（如 COUNT、SUM、AVG、MAX 和 MIN）上使用过滤器，例如：

```py
q = session.query(User.id).\
 join(User.addresses).\
 group_by(User.id).\
 having(func.count(Address.id) > 2)
```

另请参阅

`Select.having()` - v2 等效方法。

```py
method instances(result_proxy: CursorResult[Any], context: QueryContext | None = None) → Any
```

针对`CursorResult`和`QueryContext`返回一个 ORM 结果。

自 2.0 版本起已弃用：`Query.instances()`方法已弃用，并将在将来的版本中移除。请改为使用 Select.from_statement()方法或与 Session.execute()结合使用 aliased()构造。

```py
method intersect(*q: Query) → Self
```

对此查询与一个或多个查询进行 INTERSECT。

与`Query.union()`的工作方式相同。参见该方法的使用示例。

另请参阅

`Select.intersect()` - v2 等效方法。

```py
method intersect_all(*q: Query) → Self
```

对此查询与一个或多个查询进行 INTERSECT ALL。

与`Query.union()`的工作方式相同。参见该方法的使用示例。

另请参阅

`Select.intersect_all()` - v2 等效方法。

```py
attribute is_single_entity
```

指示此`Query`是否返回元组或单个实体。

如果此查询对其结果列表中的每个实例返回单个实体，则返回 True，如果此查询对其结果返回实体的元组，则返回 False。

从版本 1.3.11 开始的新功能。

另请参阅

`Query.only_return_tuples()`

```py
method join(target: _JoinTargetArgument, onclause: _OnClauseArgument | None = None, *, isouter: bool = False, full: bool = False) → Self
```

创建针对此`Query`对象的标准的 SQL JOIN，并应用生成性地返回新生成的`Query`。

**简单关系连接**

考虑两个类`User`和`Address`之间的映射，其中存在一个关系`User.addresses`表示与每个`User`关联的`Address`对象的集合。`Query.join()`的最常见用法是沿着这个关系创建一个 JOIN，使用`User.addresses`属性作为指示器指示应该如何发生：

```py
q = session.query(User).join(User.addresses)
```

在上面的情况下，调用`Query.join()`沿着`User.addresses`将导致大致等同于以下 SQL 的结果： 

```py
SELECT user.id, user.name
FROM user JOIN address ON user.id = address.user_id
```

在上述示例中，我们将`User.addresses`称为传递给`Query.join()`的“on clause”，即，它指示如何构造 JOIN 的“ON”部分。

要构建连接的链，可以使用多个`Query.join()`调用。关联绑定属性一次暗示了连接的左侧和右侧：

```py
q = session.query(User).\
 join(User.orders).\
 join(Order.items).\
 join(Item.keywords)
```

注意

如上例所示，**调用 join()方法的顺序很重要**。例如，如果我们在连接链中依次指定`User`、`Item`和`Order`，则 Query 将不知道如何正确连接；在这种情况下，根据传递的参数，它可能会引发一个不知道如何连接的错误，或者可能会产生无效的 SQL，数据库会因此而引发错误。在正确的实践中，应以使 JOIN 子句在 SQL 中呈现的方式调用`Query.join()`方法，并且每个调用应表示与之前内容的清晰链接。

**连接到目标实体或可选择项**

第二种形式的`Query.join()`允许将任何映射实体或核心可选择构造作为目标。在此用法中，`Query.join()`将尝试沿着两个实体之间的自然外键关系创建一个 JOIN：

```py
q = session.query(User).join(Address)
```

在上述调用形式中，`Query.join()`会自动为我们创建“on 子句”。如果两个实体之间没有外键，或者如果目标实体与已在左侧的实体之间存在多个外键链接，从而创建连接需要更多信息，则此调用形式最终会引发错误。请注意，当指示连接到一个没有 ON 子句的目标时，不会考虑 ORM 配置的关系。

**连接到具有 ON 子句的目标**

第三种调用形式允许显式传递目标实体以及 ON 子句。一个包含 SQL 表达式作为 ON 子句的示例如下：

```py
q = session.query(User).join(Address, User.id==Address.user_id)
```

上述形式也可以使用一个关联绑定属性作为 ON 子句：

```py
q = session.query(User).join(Address, User.addresses)
```

上述语法对于希望连接到特定目标实体的别名的情况很有用。如果我们想要两次连接到`Address`，可以使用`aliased()`函数设置两个别名：

```py
a1 = aliased(Address)
a2 = aliased(Address)

q = session.query(User).\
 join(a1, User.addresses).\
 join(a2, User.addresses).\
 filter(a1.email_address=='ed@foo.com').\
 filter(a2.email_address=='ed@bar.com')
```

使用关联绑定调用形式还可以使用`PropComparator.of_type()`方法指定目标实体；与上面的查询等效的查询如下：

```py
a1 = aliased(Address)
a2 = aliased(Address)

q = session.query(User).\
 join(User.addresses.of_type(a1)).\
 join(User.addresses.of_type(a2)).\
 filter(a1.email_address == 'ed@foo.com').\
 filter(a2.email_address == 'ed@bar.com')
```

**增强内置 ON 子句**

作为为现有关系提供完整自定义 ON 条件的替代方法，可以将`PropComparator.and_()`函数应用于关系属性，以将额外条件增加到 ON 子句中；附加条件将使用 AND 与默认条件组合：

```py
q = session.query(User).join(
 User.addresses.and_(Address.email_address != 'foo@bar.com')
)
```

版本 1.4 中的新功能。

**连接到表和子查询**

加入的目标也可以是任何表或 SELECT 语句，它可能与目标实体相关或不相关。使用适当的`.subquery()`方法以将查询转换为子查询：

```py
subq = session.query(Address).\
 filter(Address.email_address == 'ed@foo.com').\
 subquery()

q = session.query(User).join(
 subq, User.id == subq.c.user_id
)
```

通过使用`aliased()`将子查询链接到实体，可以以特定关系和/或目标实体的术语连接到子查询：

```py
subq = session.query(Address).\
 filter(Address.email_address == 'ed@foo.com').\
 subquery()

address_subq = aliased(Address, subq)

q = session.query(User).join(
 User.addresses.of_type(address_subq)
)
```

**控制从何处连接**

在当前`Query`状态的左侧与我们要连接的内容不一致的情况下，可以使用`Query.select_from()`方法： 

```py
q = session.query(Address).select_from(User).\
 join(User.addresses).\
 filter(User.name == 'ed')
```

这将生成类似于以下 SQL：

```py
SELECT address.* FROM user
 JOIN address ON user.id=address.user_id
 WHERE user.name = :name_1
```

另请参阅

`Select.join()` - v2 相当的方法。

参数：

+   `*props` – 用于`Query.join()`的传入参数，现代用法中的 props 集合应视为一种或两种参数形式，即作为单个“目标”实体或 ORM 属性绑定关系，或作为目标实体加上一个“on clause”，该“on clause”可以是 SQL 表达式或 ORM 属性绑定关系。

+   `isouter=False` – 如果为 True，则使用的连接将是左外连接，就像调用了`Query.outerjoin()`方法一样。

+   `full=False` – 渲染 FULL OUTER JOIN；隐含`isouter`。

```py
method label(name: str | None) → Label[Any]
```

返回由此`Query`表示的完整 SELECT 语句，转换为具有给定名称标签的标量子查询。

另请参阅

`Select.label()` - v2 类似的方法。

```py
attribute lazy_loaded_from
```

正在将此`Query`用于惰性加载操作的`InstanceState`。

从版本 1.4 开始不推荐使用：此属性应通过`ORMExecuteState.lazy_loaded_from`属性查看，在`SessionEvents.do_orm_execute()`事件的上下文中。

另请参阅

`ORMExecuteState.lazy_loaded_from`

```py
method limit(limit: _LimitOffsetType) → Self
```

对查询应用 `LIMIT` 并返回新生成的 `Query`。

另请参阅

`Select.limit()` - v2 等效方法。

```py
method merge_result(iterator: FrozenResult[Any] | Iterable[Sequence[Any]] | Iterable[object], load: bool = True) → FrozenResult[Any] | Iterable[Any]
```

将结果合并到此 `Query` 对象的会话中。

自版本 2.0 弃用：`Query.merge_result()` 方法被视为 SQLAlchemy 1.x 系列的遗留构造，并在 2.0 中成为遗留构造。该方法已被 `merge_frozen_result()` 函数取代。 （有关 SQLAlchemy 2.0 的背景信息，请参阅：SQLAlchemy 2.0 - 主要迁移指南）

给定与此查询相同结构的 `Query` 返回的迭代器，返回一个相同的结果迭代器，所有映射实例都使用 `Session.merge()` 合并到会话中。 这是一种优化方法，将合并所有映射实例，保留结果行的结构和未映射列，比显式为每个值调用 `Session.merge()` 的方法开销小。

结果的结构是基于此 `Query` 的列列表确定的 - 如果这些列不对应，将会发生未经检查的错误。

‘load’ 参数与 `Session.merge()` 相同。

有关 `Query.merge_result()` 的用法示例，请参阅示例 Dogpile Caching 的源代码，其中 `Query.merge_result()` 用于有效地从缓存中恢复状态到目标 `Session`。

```py
method offset(offset: _LimitOffsetType) → Self
```

对查询应用 `OFFSET` ��返回新生成的 `Query`。

另请参阅

`Select.offset()` - v2 等效方法。

```py
method one() → _T
```

返回确切的一个结果或引发异常。

如果查询未选择任何行，则引发 `sqlalchemy.orm.exc.NoResultFound`。如果返回多个对象标识，或者对于仅返回标量值而不是完全映射实体的查询返回多行，则引发 `sqlalchemy.orm.exc.MultipleResultsFound`。

调用`one()`会导致执行底层查询。

另请参见

`Query.first()`

`Query.one_or_none()`

`Result.one()` - v2 可比较方法。

`Result.scalar_one()` - v2 可比较方法。

```py
method one_or_none() → _T | None
```

返回最多一个结果或引发异常。

如果查询未选择任何行，则返回`None`。 如果返回多个对象标识，或者如果对于返回标量值而不是完整标识映射的实体的查询返回多行，则引发`sqlalchemy.orm.exc.MultipleResultsFound`。

调用`Query.one_or_none()`会导致执行底层查询。

另请参见

`Query.first()`

`Query.one()`

`Result.one_or_none()` - v2 可比较方法。

`Result.scalar_one_or_none()` - v2 可比较方法。

```py
method only_return_tuples(value: bool) → Query
```

当设置为 True 时，查询结果将始终是一个`Row`对象。

这可以将通常返回单个实体作为标量的查询，在所有情况下返回一个`Row`结果。

另请参见

`Query.tuples()` - 返回元组，但在类型级别上也将结果类型化为`Tuple`。

`Query.is_single_entity()`

`Result.tuples()` - v2 可比较方法。

```py
method options(*args: ExecutableOption) → Self
```

返回一个新的`Query`对象，应用给定的映射器选项列表。

大多数提供的选项都涉及更改如何加载列和关系映射的属性。

另请参见

列加载选项

使用加载选项进行关系加载

```py
method order_by(_Query__first: Literal[None, False, _NoArg.NO_ARG] | _ColumnExpressionOrStrLabelArgument[Any] = _NoArg.NO_ARG, *clauses: _ColumnExpressionOrStrLabelArgument[Any]) → Self
```

应用一个或多个 ORDER BY 标准到查询，并返回新生成的`Query`。

例如：

```py
q = session.query(Entity).order_by(Entity.id, Entity.name)
```

多次调用此方法等效于一次将所有子句连接起来调用。所有现有的 ORDER BY 条件都可以通过单独传递`None`来取消。然后可以通过再次调用`Query.order_by()`来添加新的 ORDER BY 条件，例如：

```py
# will erase all ORDER BY and ORDER BY new_col alone
q = q.order_by(None).order_by(new_col)
```

另请参阅

这些部分描述了按 2.0 风格调用的 ORDER BY，但也适用于`Query`：

ORDER BY - 在 SQLAlchemy 统一教程中

按标签排序或分组 - 在 SQLAlchemy 统一教程中

`Select.order_by()` - v2 等效方法。

```py
method outerjoin(target: _JoinTargetArgument, onclause: _OnClauseArgument | None = None, *, full: bool = False) → Self
```

在此`Query`对象的条件上创建左外连接，并在生成式上应用，返回新生成的`Query`。

使用方法与`join()`方法相同。

另请参阅

`Select.outerjoin()` - v2 等效方法。

```py
method params(_Query__params: Dict[str, Any] | None = None, **kw: Any) → Self
```

为可能已在 filter() 中指定的绑定参数添加值。

参数可以使用**kwargs 指定，或者作为第一个位置参数使用单个字典。两者之所以都存在是因为**kwargs 很方便，但是一些参数字典包含 Unicode 键，**kwargs 就不能用。

```py
method populate_existing() → Self
```

返回一个将在加载时过期并刷新所有实例，或者从当前`Session`中重用的`Query`。

从 SQLAlchemy 1.4 开始，`Query.populate_existing()`方法等效于在 ORM 级别使用`populate_existing`执行选项。有关此选项的更多背景信息，请参见 填充现有 部分。

```py
method prefix_with(*prefixes: _TextCoercedExpressionArgument[Any], dialect: str = '*') → Self
```

*继承自* `HasPrefixes.prefix_with()` *方法的* `HasPrefixes`

在语句关键字后添加一个或多个表达式，即 SELECT、INSERT、UPDATE 或 DELETE。生成式。

这用于支持特定于后端的前缀关键字，例如 MySQL 提供的关键字。

例如：

```py
stmt = table.insert().prefix_with("LOW_PRIORITY", dialect="mysql")

# MySQL 5.7 optimizer hints
stmt = select(table).prefix_with(
 "/*+ BKA(t1) */", dialect="mysql")
```

可以通过多次调用`HasPrefixes.prefix_with()`来指定多个前缀。

参数：

+   `*prefixes` – 文本或`ClauseElement` 构造，将在插入、更新或删除关键字之后呈现。

+   `dialect` – 可选的字符串方言名称，将仅限于将此前缀呈现为该方言。

```py
method reset_joinpoint() → Self
```

返回一个新的 `Query`，其中“连接点”已被重置回查询的基本 FROM 实体。

该方法通常与 `Query.join()` 方法的 `aliased=True` 特性一起使用。请参阅 `Query.join()` 中的示例，了解其使用方法。

```py
method scalar() → Any
```

返回第一个结果的第一个元素，如果没有行存在则返回 None。如果返回多行，则引发 MultipleResultsFound。

```py
>>> session.query(Item).scalar()
<Item>
>>> session.query(Item.id).scalar()
1
>>> session.query(Item.id).filter(Item.id < 0).scalar()
None
>>> session.query(Item.id, Item.name).scalar()
1
>>> session.query(func.count(Parent.id)).scalar()
20
```

这将导致执行基础查询。

另请参阅

`Result.scalar()` - v2 可比较方法。

```py
method scalar_subquery() → ScalarSelect[Any]
```

返回由此 `Query` 表示的完整 SELECT 语句，转换为标量子查询。

类似于 `SelectBase.scalar_subquery()`。

自版本 `1.4` 起变更：`Query.scalar_subquery()` 方法取代了 `Query.as_scalar()` 方法。

另请参阅

`Select.scalar_subquery()` - v2 可比较方法。

```py
method select_from(*from_obj: FromClauseRole | Type[Any] | Inspectable[_HasClauseElement[Any]] | _HasClauseElement[Any]) → Self
```

显式设置此 `Query` 的 FROM 子句。

`Query.select_from()` 常常与 `Query.join()` 结合使用，以控制从连接的“左”侧选择的实体。

此处的实体或可选择对象有效地替换了任何对 `Query.join()` 的调用的“左边缘”，当没有其他方式建立连接点时 - 通常，默认的“连接点”是查询对象的要选择的实体列表中最左边的实体。

一个典型的例子：

```py
q = session.query(Address).select_from(User).\
 join(User.addresses).\
 filter(User.name == 'ed')
```

这将生成等效于以下 SQL：

```py
SELECT address.* FROM user
JOIN address ON user.id=address.user_id
WHERE user.name = :name_1
```

参数：

***from_obj** – 一个或多个要应用于 FROM 子句的实体集合。实体可以是映射类、`AliasedClass`对象、`Mapper`对象，以及核心`FromClause`元素，如子查询。

另请参阅

`Query.join()`

`Query.select_entity_from()`

`Select.select_from()` - v2 等效方法。

```py
attribute selectable
```

返回由此`Query`发出的`Select`对象。

用于`inspect()`兼容性，这相当于：

```py
query.enable_eagerloads(False).with_labels().statement
```

```py
method set_label_style(style: SelectLabelStyle) → Self
```

将列标签应用于 Query.statement 的返回值。

表示此查询的语句访问器应返回一个 SELECT 语句，该语句将标签应用于形式为<tablename>_<columnname>的所有列；这通常用于消除具有相同名称的多个表中的列的歧义。

当查询实际发出 SQL 以加载行时，它总是使用列标签。

注意

`Query.set_label_style()`方法*仅*应用于`Query.statement`的输出，*不*应用于`Query`本身的任何结果行调用系统，例如`Query.first()`，`Query.all()`等。要使用`Query.set_label_style()`执行查询，请使用`Session.execute()`调用`Query.statement`：

```py
result = session.execute(
 query
 .set_label_style(LABEL_STYLE_TABLENAME_PLUS_COL)
 .statement
)
```

1.4 版本中的新功能。

另请参阅

`Select.set_label_style()` - v2 等效方法。

```py
method slice(start: int, stop: int) → Self
```

计算由给定索引表示的`Query`的“切片”，并返回结果`Query`。

开始和停止索引的行为类似于 Python 内置`range()`函数的参数。此方法提供了使用`LIMIT`/`OFFSET`来获取查询的切片的替代方法。

例如，

```py
session.query(User).order_by(User.id).slice(1, 3)
```

渲染为

```py
SELECT  users.id  AS  users_id,
  users.name  AS  users_name
FROM  users  ORDER  BY  users.id
LIMIT  ?  OFFSET  ?
(2,  1)
```

另请参阅

`Query.limit()`

`Query.offset()`

`Select.slice()` - v2 等效方法。

```py
attribute statement
```

由此 Query 表示的完整 SELECT 语句。

该语句默认情况下不会对构造应用歧义标签，除非首先调用 with_labels(True)。

```py
method subquery(name: str | None = None, with_labels: bool = False, reduce_columns: bool = False) → Subquery
```

返回由此 `Query` 表示的完整 SELECT 语句，嵌入在一个 `Alias` 中。

查询中禁用了急切的 JOIN 生成。

另请参阅

`Select.subquery()` - v2 可比较方法。

参数：

+   `name` – 要分配为别名的字符串名称；这将传递给 `FromClause.alias()`。如果为 `None`，则在编译时将确定性地生成一个名称。

+   `with_labels` – 如果为 True，则首先将 `with_labels()` 应用于 `Query`，以将表限定标签应用于所有列。

+   `reduce_columns` – 如果为 True，则将调用 `Select.reduce_columns()` 来删除结果 `select()` 构造中的同名列，其中一个还通过外键或 WHERE 子句等价关系引用另一个。

```py
method suffix_with(*suffixes: _TextCoercedExpressionArgument[Any], dialect: str = '*') → Self
```

*继承自* `HasSuffixes.suffix_with()` *方法* `HasSuffixes`

将作为整个语句后的一个或多个表达式添加。

这用于支持特定于后端的后缀关键字在某些构造上。

例如：

```py
stmt = select(col1, col2).cte().suffix_with(
 "cycle empno set y_cycle to 1 default 0", dialect="oracle")
```

可以通过多次调用 `HasSuffixes.suffix_with()` 来指定多个后缀。

参数：

+   `*suffixes` – 将在目标子句后呈现的文本或 `ClauseElement` 构造。

+   `dialect` – 可选的字符串方言名称，将限制仅将此后缀呈现为该方言。

```py
method tuples() → Query
```

返回这个`Query`的元组类型形式。

此方法调用`Query.only_return_tuples()`方法，并将其值设置为`True`，这本身就确保了这个`Query`总是返回`Row`对象，即使查询是针对单个实体的。然后，它还会在类型级别返回一个“类型化”的查询，如果可能的话，该查询将将结果行类型化为具有类型的元组对象。

这种方法可以与`Result.tuples()`方法进行比较，该方法返回“self”，但从类型的角度来看，返回一个将产生带有类型的`Tuple`对象的对象。只有当这个`Query`对象已经是一个类型化的查询对象时，类型才会生效。

版本 2.0 中的新功能。

另请参阅

`Result.tuples()` - v2 等效方法。

```py
method union(*q: Query) → Self
```

对一个或多个查询执行 UNION。

例如：

```py
q1 = sess.query(SomeClass).filter(SomeClass.foo=='bar')
q2 = sess.query(SomeClass).filter(SomeClass.bar=='foo')

q3 = q1.union(q2)
```

该方法接受多个查询对象，以控制嵌套的级别。一系列`union()`调用，如下所示：

```py
x.union(y).union(z).all()
```

将在每个`union()`上进行嵌套，并生成：

```py
SELECT * FROM (SELECT * FROM (SELECT * FROM X UNION
 SELECT * FROM y) UNION SELECT * FROM Z)
```

而：

```py
x.union(y, z).all()
```

生成：

```py
SELECT * FROM (SELECT * FROM X UNION SELECT * FROM y UNION
 SELECT * FROM Z)
```

请注意，许多数据库后端不允许在 UNION、EXCEPT 等内部调用的查询上渲染 ORDER BY。要禁用所有 ORDER BY 子句，包括在映射器上配置的子句，请发出`query.order_by(None)` - 结果的`Query`对象将不会在其 SELECT 语句中渲染 ORDER BY。

另请参阅

`Select.union()` - v2 等效方法。

```py
method union_all(*q: Query) → Self
```

对一个或多个查询执行 UNION ALL。

与`Query.union()`的工作方式相同。请参阅该方法以获取用法示例。

另请参阅

`Select.union_all()` - v2 等效方法。

```py
method update(values: Dict[_DMLColumnArgument, Any], synchronize_session: SynchronizeSessionArgument = 'auto', update_args: Dict[Any, Any] | None = None) → int
```

使用任意 WHERE 子句执行 UPDATE。

更新数据库中与此查询匹配的行。

例如：

```py
sess.query(User).filter(User.age == 25).\
 update({User.age: User.age - 10}, synchronize_session=False)

sess.query(User).filter(User.age == 25).\
 update({"age": User.age - 10}, synchronize_session='evaluate')
```

警告

查看 ORM 启用的 INSERT、UPDATE 和 DELETE 语句一节，了解重要的警告和注意事项，包括在使用任意 UPDATE 和 DELETE 与映射器继承配置时的限制。

参数：

+   `values` – 一个包含属性名称的字典，或者作为键的映射属性或 SQL 表达式，以及作为值的文字值或 SQL 表达式。如果希望使用 参数排序模式，则值可以作为 2 元组的列表传递；这要求将 `update.preserve_parameter_order` 标志也传递给 `Query.update.update_args` 字典。

+   `synchronize_session` – 选择在会话中更新对象属性的策略。参见 ORM-Enabled INSERT, UPDATE, and DELETE statements 章节，讨论这些策略。

+   `update_args` – 可选字典，如果存在，则会作为对象的 `update()` 构造函数的 `**kw` 参数传递给底层。可以用于传递特定于方言的参数，如 `mysql_limit`，以及其他特殊参数，如 `update.preserve_parameter_order`。

返回：

数据库的“行计数”功能返回的匹配行数。

另请参见

ORM-Enabled INSERT, UPDATE, and DELETE statements

```py
method value(column: _ColumnExpressionArgument[Any]) → Any
```

返回与给定列表达式对应的标量结果。

自版本 1.4 起弃用：`Query.value()` 已弃用，并将在将来的版本中删除。请结合使用 `Query.with_entities()` 和 `Query.scalar()`。

```py
method values(*columns: _ColumnsClauseArgument[Any]) → Iterable[Any]
```

返回一个迭代器，生成与给定列列表对应的结果元组。

自版本 1.4 起弃用：`Query.values()` 已弃用，并将在将来的版本中删除。请使用 `Query.with_entities()`。

```py
method where(*criterion: _ColumnExpressionArgument[bool]) → Self
```

`Query.filter()` 的别名。

版本 1.4 中的新功能。

另请参见

`Select.where()` - v2 等效方法。

```py
attribute whereclause
```

返回此查询的当前 WHERE 条件的只读属性。

返回的值是一个 SQL 表达式构造，如果没有建立条件，则为 `None`。

另请参见

`Select.whereclause` - v2 等效属性。

```py
method with_entities(*entities: _ColumnsClauseArgument[Any], **_Query__kw: Any) → Query[Any]
```

返回一个用给定实体替换 SELECT 列表的新`Query`。

例如：

```py
# Users, filtered on some arbitrary criterion
# and then ordered by related email address
q = session.query(User).\
 join(User.address).\
 filter(User.name.like('%ed%')).\
 order_by(Address.email)

# given *only* User.id==5, Address.email, and 'q', what
# would the *next* User in the result be ?
subq = q.with_entities(Address.email).\
 order_by(None).\
 filter(User.id==5).\
 subquery()
q = q.join((subq, subq.c.email < Address.email)).\
 limit(1)
```

另请参阅

`Select.with_only_columns()` - v2 可比较方法。

```py
method with_for_update(*, nowait: bool = False, read: bool = False, of: _ForUpdateOfArgument | None = None, skip_locked: bool = False, key_share: bool = False) → Self
```

返回一个具有指定`FOR UPDATE`子句选项的新`Query`。

此方法的行为与`GenerativeSelect.with_for_update()`相同。当没有参数调用时，生成的 `SELECT` 语句将附加一个 `FOR UPDATE` 子句。当指定了额外的参数时，如 `FOR UPDATE NOWAIT` 或 `LOCK IN SHARE MODE`，特定于后端的选项会生效。

例如：

```py
q = sess.query(User).populate_existing().with_for_update(nowait=True, of=User)
```

在 PostgreSQL 后端上执行上述查询会呈现如下：

```py
SELECT users.id AS users_id FROM users FOR UPDATE OF users NOWAIT
```

警告

在使用`with_for_update`来进行急加载关系时，它并不受 SQLAlchemy 官方支持或推荐，并且可能无法与各种数据库后端上的某些查询一起正常工作。当成功使用`with_for_update`与涉及到`joinedload()`的查询时，SQLAlchemy 将尝试生成锁定所有涉及的表的 SQL。

注意

通常在使用`Query.with_for_update()`方法时，结合使用`Query.populate_existing()`方法是一个好主意。`Query.populate_existing()`的目的是强制将从 SELECT 中读取的所有数据都填充到返回的 ORM 对象中，即使这些对象已经存在于标识映射中。

另请参阅

`GenerativeSelect.with_for_update()` - 具有完整参数和行为描述的核心级方法。

`Query.populate_existing()` - 覆盖已加载到标识映射中的对象的属性。

```py
method with_hint(selectable: _FromClauseArgument, text: str, dialect_name: str = '*') → Self
```

*继承自* `HasHints.with_hint()` *方法的* `HasHints`

为给定的可选对象添加索引或其他执行上下文提示到这个`Select`或其他可选对象中。

提示的文本将根据正在使用的数据库后端在给定的 `Table` 或 `Alias` 中的适当位置进行渲染。方言实现通常使用 Python 字符串替换语法，其中令牌 `%(name)s` 用于呈现表或别名的名称。例如，在使用 Oracle 时，以下内容：

```py
select(mytable).\
 with_hint(mytable, "index(%(name)s ix_mytable)")
```

渲染 SQL 如下：

```py
select /*+ index(mytable ix_mytable) */ ... from mytable
```

`dialect_name` 选项将限制特定后端的特定提示的渲染。例如，同时为 Oracle 和 Sybase 添加提示：

```py
select(mytable).\
 with_hint(mytable, "index(%(name)s ix_mytable)", 'oracle').\
 with_hint(mytable, "WITH INDEX ix_mytable", 'mssql')
```

另请参阅

`Select.with_statement_hint()`

```py
method with_labels() → Self
```

自 2.0 版本起弃用：`Query.with_labels()` 和 `Query.apply_labels()` 方法在 SQLAlchemy 1.x 系列中被视为遗留，且在 2.0 版本中成为遗留构造。请改用 set_label_style(LABEL_STYLE_TABLENAME_PLUS_COL)。（关于 SQLAlchemy 2.0 的背景信息请参见：SQLAlchemy 2.0 - 主要迁移指南）

```py
method with_parent(instance: object, property: attributes.QueryableAttribute[Any] | None = None, from_entity: _ExternalEntityType[Any] | None = None) → Self
```

添加筛选条件，将给定实例与子对象或集合关联起来，使用其属性状态以及已建立的 `relationship()` 配置。

自 2.0 版本起弃用：`Query.with_parent()` 方法在 SQLAlchemy 1.x 系列中被视为遗留，且在 2.0 版本中成为遗留构造。请使用独立构造的 `with_parent()`。（关于 SQLAlchemy 2.0 的背景信息请参见：SQLAlchemy 2.0 - 主要迁移指南）

此方法使用 `with_parent()` 函数生成子句，其结果传递给 `Query.filter()`。

参数与 `with_parent()` 相同，唯一的例外是给定属性可以为 None，在这种情况下，将针对此 `Query` 对象的目标映射器执行搜索。

参数：

+   `instance` – 具有一些 `relationship()` 的实例。

+   `property` – 表示应使用实例哪个关系来协调父/子关系的类绑定属性。

+   `from_entity` – 要考虑为左侧的实体。默认为`Query`本身的“零”实体。

```py
method with_session(session: Session) → Self
```

返回一个将使用给定`Session`的`Query`。

虽然`Query`对象通常是使用`Session.query()`方法实例化的，但也可以直接构建`Query`而无需必然使用`Session`。这样的`Query`对象，或者已与不同`Session`关联的任何`Query`对象，可以使用此方法生成一个与目标会话关联的新`Query`对象：

```py
from sqlalchemy.orm import Query

query = Query([MyClass]).filter(MyClass.id == 5)

result = query.with_session(my_session).one()
```

```py
method with_statement_hint(text: str, dialect_name: str = '*') → Self
```

*继承自* `HasHints.with_statement_hint()` *方法的* `HasHints`

为此`Select`或其他可选择的对象添加语句提示。

此方法类似于`Select.with_hint()`，不过不需要单独的表，而是适用于整个语句。

这里的提示是特定于后端数据库的，并且可能包括隔离级别、文件指令、提取指令等指令。

另请参阅

`Select.with_hint()`

`Select.prefix_with()` - 通用的 SELECT 前缀，也可以适用于某些数据库特定的 HINT 语法，如 MySQL 优化器提示

```py
method with_transformation(fn: Callable[[Query], Query]) → Query
```

通过给定的函数返回一个经过转换的新`Query`对象。

例如：

```py
def filter_something(criterion):
 def transform(q):
 return q.filter(criterion)
 return transform

q = q.with_transformation(filter_something(x==5))
```

这允许为`Query`对象创建临时配方。

```py
method yield_per(count: int) → Self
```

一次只产出`count`行。

此方法的目的是在获取非常大的结果集（> 10K 行）时，将结果批处理到子集合中并部分地将其产出，以便 Python 解释器不需要声明非常大的内存区域，这既费时又导致内存使用过多。当使用合适的产出设置（例如大约 1000）时，即使使用缓冲行的 DBAPI（大多数情况下都是），从获取数十万行的性能通常也会提高一倍。

从 SQLAlchemy 1.4 开始，`Query.yield_per()`方法等同于在 ORM 级别使用`yield_per`执行选项。有关此选项的更多背景信息，请参阅使用 Yield Per 获取大型结果集部分。

另请参阅

使用 Yield Per 获取大型结果集

## ORM 特定查询构造

本节已移至附加 ORM API 构造。

## 查询对象

`Query`是根据给定的`Session`使用`Session.query()`方法生成的：

```py
q = session.query(SomeMappedClass)
```

以下是`Query`对象的完整接口。

| 对象名称 | 描述 |
| --- | --- |
| Query | ORM 级别的 SQL 构造对象。 |

```py
class sqlalchemy.orm.Query
```

ORM 级别的 SQL 构造对象。

遗留功能

ORM `Query`对象是 SQLAlchemy 2.0 的遗留构造。请参阅遗留查询 API 顶部的注释，包括迁移文档的链接。 

`Query` 对象通常最初是使用`Session.query()`方法从`Session`生成的，并在较少的情况下通过直接实例化`Query`并使用`Query.with_session()`方法与`Session`关联。

**成员**

__init__(), add_column(), add_columns(), add_entity(), all(), apply_labels(), as_scalar(), autoflush(), column_descriptions, correlate(), count(), cte(), delete(), distinct(), enable_assertions(), enable_eagerloads(), except_(), except_all(), execution_options(), exists(), filter(), filter_by(), first(), from_statement(), get(), get_children(), get_execution_options(), get_label_style, group_by(), having(), instances(), intersect(), intersect_all(), is_single_entity, join(), label(), lazy_loaded_from, limit(), merge_result(), offset(), one(), one_or_none(), only_return_tuples(), options(), order_by(), outerjoin(), params(), populate_existing(), prefix_with(), reset_joinpoint(), scalar(), scalar_subquery(), select_from(), selectable, set_label_style(), slice(), statement, subquery(), suffix_with(), tuples(), union(), union_all(), update(), value(), values(), where(), whereclause, with_entities(), with_for_update(), with_hint(), with_labels(), with_parent(), with_session(), with_statement_hint(), with_transformation(), yield_per()

**类签名**

类 `sqlalchemy.orm.Query` (`sqlalchemy.sql.expression._SelectFromElements`, `sqlalchemy.sql.annotation.SupportsCloneAnnotations`, `sqlalchemy.sql.expression.HasPrefixes`, `sqlalchemy.sql.expression.HasSuffixes`, `sqlalchemy.sql.expression.HasHints`, `sqlalchemy.event.registry.EventTarget`, `sqlalchemy.log.Identified`, `sqlalchemy.sql.expression.Generative`, `sqlalchemy.sql.expression.Executable`, `typing.Generic`)

```py
method __init__(entities: _ColumnsClauseArgument[Any] | Sequence[_ColumnsClauseArgument[Any]], session: Session | None = None)
```

直接构造一个 `Query`。

例如：

```py
q = Query([User, Address], session=some_session)
```

以上等同于：

```py
q = some_session.query(User, Address)
```

参数：

+   `entities` – 一个实体和/或 SQL 表达式序列。

+   `session` – 一个 `Session`，将与 `Query` 关联。可选；也可以通过 `Query.with_session()` 方法将 `Query` 与 `Session` 关联。

另请参阅

`Session.query()`

`Query.with_session()`

```py
method add_column(column: _ColumnExpressionArgument[Any]) → Query[Any]
```

将一个列表达式添加到要返回的结果列列表中。

自版本 1.4 起已弃用：`Query.add_column()` 已弃用，将在未来的发布中删除。请使用 `Query.add_columns()`

```py
method add_columns(*column: _ColumnExpressionArgument[Any]) → Query[Any]
```

将一个或多个列表达式添加到要返回的结果列列表中。

另请参阅

`Select.add_columns()` - v2 可比较的方法。

```py
method add_entity(entity: _EntityType[Any], alias: Alias | Subquery | None = None) → Query[Any]
```

将一个映射实体添加到要返回的结果列列表中。

另请参阅

`Select.add_columns()` - v2 可比较的方法。

```py
method all() → List[_T]
```

将此 `Query` 表示的结果作为列表返回。

这将导致底层 SQL 语句的执行。

警告

当询问 `Query` 对象返回由全 ORM 映射的实体组成的序列或迭代器时，将**根据主键对条目进行去重**。有关更多详细信息，请参阅 FAQ。

> 另请参阅
> 
> 我的查询返回的对象数与 query.count() 告诉我的不同 - 为什么？

另请参阅

`Result.all()` - v2 可比较的方法。

`Result.scalars()` - v2 可比较的方法。

```py
method apply_labels() → Self
```

自 2.0 版以来已弃用：`Query.with_labels()` 和 `Query.apply_labels()` 方法被视为 SQLAlchemy 1.x 系列的传统构造，并在 2.0 版中成为传统构造。请改用 set_label_style(LABEL_STYLE_TABLENAME_PLUS_COL)。 (有关 SQLAlchemy 2.0 的背景，请参阅：SQLAlchemy 2.0 - 主要迁移指南)

```py
method as_scalar() → ScalarSelect[Any]
```

将由此 `Query` 表示的完整 SELECT 语句转换为标量子查询。

自 1.4 版以来已弃用：`Query.as_scalar()` 方法已弃用，并将在将来的版本中删除。请参阅 `Query.scalar_subquery()`。

```py
method autoflush(setting: bool) → Self
```

返回具有特定 'autoflush' 设置的查询。

自 SQLAlchemy 1.4 起，`Query.autoflush()` 方法等同于在 ORM 级别使用 `autoflush` 执行选项。有关此选项的更多背景，请参阅自动刷新部分。

```py
attribute column_descriptions
```

返回由此 `Query` 返回的列的元数据。

格式是一个字典列表：

```py
user_alias = aliased(User, name='user2')
q = sess.query(User, User.id, user_alias)

# this expression:
q.column_descriptions

# would return:
[
 {
 'name':'User',
 'type':User,
 'aliased':False,
 'expr':User,
 'entity': User
 },
 {
 'name':'id',
 'type':Integer(),
 'aliased':False,
 'expr':User.id,
 'entity': User
 },
 {
 'name':'user2',
 'type':User,
 'aliased':True,
 'expr':user_alias,
 'entity': user_alias
 }
]
```

另请参阅

此 API 也可以使用 2.0 风格 查询，在以下文档中有所记录：

+   检查来自启用 ORM 的 SELECT 和 DML 语句的实体和列

+   `Select.column_descriptions`

```py
method correlate(*fromclauses: Literal[None, False] | FromClauseRole | Type[Any] | Inspectable[_HasClauseElement[Any]] | _HasClauseElement[Any]) → Self
```

返回一个将给定的 FROM 子句与封闭的 `Query` 或 `select()` 相关联的 `Query` 构造。

此处的方法接受映射类、`aliased()` 构造和 `Mapper` 构造作为参数，它们会被解析为表达式构造，以及适当的表达式构造。

相关参数最终在转换为表达式构造后传递给 `Select.correlate()`。

在诸如使用 `Query.from_self()` 或者当由 `Query.subquery()` 返回的子查询嵌入到另一个 `select()` 构造中时，相关参数才会生效。

另请参阅

`Select.correlate()` - v2 等效方法。

```py
method count() → int
```

返回此 `Query` 生成的 SQL 所返回的行数的计数。

这将生成此查询的 SQL 如下：

```py
SELECT count(1) AS count_1 FROM (
 SELECT <rest of query follows...>
) AS anon_1
```

上述 SQL 返回单行，这是计数函数的聚合值；然后 `Query.count()` 方法返回该单个整数值。

警告

重要的是要注意，count() 返回的值 **与此 Query 从 .all() 方法等返回的 ORM 对象数不同**。当 `Query` 对象被要求返回完整实体时，将根据主键去重，这意味着如果相同的主键值在结果中出现多次，则只会存在一个该主键的对象。这不适用于针对单个列的查询。

另请参阅

我的查询返回的对象数量与 query.count() 告诉我的不同 - 为什么？

要对特定列进行精细控制以进行计数，跳过子查询的使用或以其他方式控制 FROM 子句，或者使用其他聚合函数，请结合 `Session.query()` 中的 `expression.func` 表达式，例如：

```py
from sqlalchemy import func

# count User records, without
# using a subquery.
session.query(func.count(User.id))

# return count of user "id" grouped
# by "name"
session.query(func.count(User.id)).\
 group_by(User.name)

from sqlalchemy import distinct

# count distinct "name" values
session.query(func.count(distinct(User.name)))
```

另请参阅

2.0 迁移 - ORM 使用

```py
method cte(name: str | None = None, recursive: bool = False, nesting: bool = False) → CTE
```

返回由此 `Query` 表示的完整 SELECT 语句，表示为一个通用表达式（CTE）。

参数和用法与`SelectBase.cte()`方法相同；请参阅该方法以获取更多详细信息。

这里是[PostgreSQL WITH RECURSIVE 示例](https://www.postgresql.org/docs/current/static/queries-with.html)。请注意，在此示例中，`included_parts` cte 和其别名`incl_alias`都是 Core 可选择项，这意味着可以通过`.c.`属性访问列。`parts_alias`对象是`Part`实体的`aliased()`实例，因此可以直接访问映射到列的属性：

```py
from sqlalchemy.orm import aliased

class Part(Base):
 __tablename__ = 'part'
 part = Column(String, primary_key=True)
 sub_part = Column(String, primary_key=True)
 quantity = Column(Integer)

included_parts = session.query(
 Part.sub_part,
 Part.part,
 Part.quantity).\
 filter(Part.part=="our part").\
 cte(name="included_parts", recursive=True)

incl_alias = aliased(included_parts, name="pr")
parts_alias = aliased(Part, name="p")
included_parts = included_parts.union_all(
 session.query(
 parts_alias.sub_part,
 parts_alias.part,
 parts_alias.quantity).\
 filter(parts_alias.part==incl_alias.c.sub_part)
 )

q = session.query(
 included_parts.c.sub_part,
 func.sum(included_parts.c.quantity).
 label('total_quantity')
 ).\
 group_by(included_parts.c.sub_part)
```

另请参阅

`Select.cte()` - v2 等效方法。

```py
method delete(synchronize_session: SynchronizeSessionArgument = 'auto') → int
```

使用任意 WHERE 子句执行 DELETE。

从数据库中删除此查询匹配的行。

例如：

```py
sess.query(User).filter(User.age == 25).\
 delete(synchronize_session=False)

sess.query(User).filter(User.age == 25).\
 delete(synchronize_session='evaluate')
```

警告

请参阅 ORM-Enabled INSERT、UPDATE 和 DELETE 语句以获取重要的注意事项和警告，包括在使用 mapper 继承配置时使用批量 UPDATE 和 DELETE 时的限制。

参数：

**synchronize_session** – 选择更新会话中对象属性的策略。请参阅 ORM-Enabled INSERT、UPDATE 和 DELETE 语句部分，了解这些策略的讨论。

返回：

由数据库的“行计数”功能返回的匹配行数。

另请参阅

ORM-Enabled INSERT、UPDATE 和 DELETE 语句

```py
method distinct(*expr: _ColumnExpressionArgument[Any]) → Self
```

对查询应用`DISTINCT`并返回新结果的`Query`。

注意

ORM 级别的`distinct()`调用包括逻辑，将查询的 ORDER BY 中的列自动添加到 SELECT 语句的列子句中，以满足数据库后端的常见需求，即使用 DISTINCT 时，ORDER BY 列必须是 SELECT 列的一部分。然而，这些列*不会*添加到实际由`Query`获取的列列表中，因此不会影响结果。但是，在使用`Query.statement`访问器时，这些列会被传递。

自版本 2.0 起已弃用：此逻辑已弃用，并将在 SQLAlchemy 2.0 中删除。请参阅仅选择实体时使用 DISTINCT 添加额外列以获取 2.0 版中此用例的描述。

另请参阅

`Select.distinct()` - v2 等效方法。

参数：

***expr** –

可选的列表达式。当存在时，PostgreSQL 方言将渲染`DISTINCT ON (<expressions>)`结构。

自 1.4 版弃用：在其他方言中使用*expr 已弃用，并将在将来的版本中引发`CompileError`。

```py
method enable_assertions(value: bool) → Self
```

控制是否生成断言。

当设置为 False 时，返回的查询在执行某些操作之前不会断言其状态，包括在调用`filter()`时未应用 LIMIT/OFFSET，在调用`get()`时不存在条件，以及在调用`filter()`/`order_by()`/`group_by()`时不存在“from_statement()”。 此更宽松的模式由自定义查询子类使用，以指定通常用法模式之外的条件或其他修改器。

应注意确保使用模式是可行的。 例如，由`from_statement()`应用的语句将覆盖`filter()`或`order_by()`设置的任何条件。

```py
method enable_eagerloads(value: bool) → Self
```

控制是否渲染急切连接和子查询。

当设置为 False 时，返回的查询不会渲染急切连接，而不管`joinedload()`，`subqueryload()`选项或映射级别的`lazy='joined'`/`lazy='subquery'`配置。

主要用于将查询的语句嵌套到子查询或其他可选择项中，或者使用`Query.yield_per()`时。

```py
method except_(*q: Query) → Self
```

对一个或多个查询产生此查询的差集。

与`Query.union()`的工作方式相同。 有关用法示例，请参见该方法。

另请参阅

`Select.except_()` - v2 等效方法。

```py
method except_all(*q: Query) → Self
```

对一个或多个查询产生此查询的全集。

与`Query.union()`的工作方式相同。 有关用法示例，请参见该方法。

另请参阅

`Select.except_all()` - v2 等效方法。

```py
method execution_options(**kwargs: Any) → Self
```

设置在执行期间生效的非 SQL 选项。

此处允许的选项包括`Connection.execution_options()`接受的所有选项，以及一系列 ORM 特定选项：

`populate_existing=True` - 相当于使用`Query.populate_existing()`

`autoflush=True|False` - 相当于使用`Query.autoflush()`

`yield_per=<value>` - 相当于使用`Query.yield_per()`

注意，如果使用`Query.yield_per()`方法或执行选项，则`stream_results`执行选项会自动启用。

1.4 版本中新增：- 在 `Query.execution_options()` 中添加了 ORM 选项

使用 2.0 风格 查询时，也可以在每次执行时指定执行选项，通过 `Session.execution_options` 参数。

警告

不应在单个 ORM 语句执行的级别使用 `Connection.execution_options.stream_results` 参数，因为 `Session` 不会跟踪来自单个会话中的不同模式转换映射的对象。对于在单个 `Session` 范围内的多个模式转换映射，请参阅水平分片。

另请参阅

使用服务器端游标（即流式结果）

`Query.get_execution_options()`

`Select.execution_options()` - v2 相当的方法。

```py
method exists() → Exists
```

一个方便的方法，将查询转换为 EXISTS 子查询的形式 EXISTS (SELECT 1 FROM … WHERE …)。

例如：

```py
q = session.query(User).filter(User.name == 'fred')
session.query(q.exists())
```

生成类似于以下 SQL：

```py
SELECT EXISTS (
 SELECT 1 FROM users WHERE users.name = :name_1
) AS anon_1
```

EXISTS 构造通常用于 WHERE 子句中：

```py
session.query(User.id).filter(q.exists()).scalar()
```

请注意，一些数据库（如 SQL Server）不允许在 SELECT 的列子句中出现 EXISTS 表达式。要根据 EXISTS 在 WHERE 中作为 WHERE 子句的简单布尔值选择，请使用 `literal()`：

```py
from sqlalchemy import literal

session.query(literal(True)).filter(q.exists()).scalar()
```

另请参阅

`Select.exists()` - v2 相当的方法。

```py
method filter(*criterion: _ColumnExpressionArgument[bool]) → Self
```

将给定的过滤条件应用于此 `Query` 的副本，使用 SQL 表达式。

例如：

```py
session.query(MyClass).filter(MyClass.name == 'some name')
```

可以指定多个条件，用逗号分隔；效果是它们将使用 `and_()` 函数连接在一起：

```py
session.query(MyClass).\
 filter(MyClass.name == 'some name', MyClass.id > 5)
```

条件可以是任何适用于 select WHERE 子句的 SQL 表达式对象。字符串表达式会通过 `text()` 构造转换为 SQL 表达式结构。

另请参阅

`Query.filter_by()` - 使用关键字表达式进行过滤。

`Select.where()` - v2 相当的方法。

```py
method filter_by(**kwargs: Any) → Self
```

将给定的过滤条件应用于此`Query`的副本，使用关键字表达式。

例如：

```py
session.query(MyClass).filter_by(name = 'some name')
```

可以指定多个条件，用逗号分隔；效果是它们将使用`and_()`函数连接在一起：

```py
session.query(MyClass).\
 filter_by(name = 'some name', id = 5)
```

关键字表达式是从查询的主要实体中提取的，或者是最后一个被调用`Query.join()`的目标实体。

另请参阅

`Query.filter()` - 根据 SQL 表达式进行过滤。

`Select.filter_by()` - v2 相当的方法。

```py
method first() → _T | None
```

返回此`Query`的第一个结果，如果结果不包含任何行，则返回`None`。

first()在生成的 SQL 中应用了一个限制为一的限制，因此只在服务器端生成一个主实体行（请注意，如果存在联接加载的集合，则可能由多个结果行组成）。

调用`Query.first()`会导致基础查询的执行。

另请参阅

`Query.one()`

`Query.one_or_none()`

`Result.first()` - v2 相当的方法。

`Result.scalars()` - v2 相当的方法。

```py
method from_statement(statement: ExecutableReturnsRows) → Self
```

执行给定的 SELECT 语句并返回结果。

此方法绕过所有内部语句编译，并且语句在不经修改的情况下执行。

语句通常是`text()`或`select()`构造，并且应返回适合此`Query`所代表的实体类的列集。

另请参阅

`Select.from_statement()` - v2 相当的方法。

```py
method get(ident: _PKIdentityArgument) → Any | None
```

根据给定的主键标识符返回一个实例，如果未找到则返回`None`。

自版本 2.0 起弃用：`Query.get()` 方法被认为是 SQLAlchemy 1.x 系列的遗留功能，并在 2.0 版本中成为遗留构造。该方法现在作为 `Session.get()` 可用（有关 SQLAlchemy 2.0 的背景，请参见：SQLAlchemy 2.0 - 主要迁移指南）

例如：

```py
my_user = session.query(User).get(5)

some_object = session.query(VersionedFoo).get((5, 10))

some_object = session.query(VersionedFoo).get(
 {"id": 5, "version_id": 10})
```

`Query.get()` 在提供对所属 `Session` 的标识映射的直接访问方面是特殊的。如果给定的主键标识符存在于本地标识映射中，则对象将直接从该集合返回，而不会发出 SQL，除非对象已被标记为完全过期。如果不存在，则执行 SELECT 以定位对象。

`Query.get()` 也会检查对象是否存在于标识映射中并标记为过期 - 发出一个 SELECT 来刷新对象以及确保行仍然存在。如果不是，`ObjectDeletedError` 被引发。

`Query.get()` 仅用于返回单个映射实例，而不是多个实例或单个列构造，并且严格地基于单个主键值。源 `Query` 必须以这种方式构造，即针对单个映射实体，没有额外的过滤条件。但是，可以通过`Query.options()` 应用加载选项，并且如果对象尚未在本地存在，则将使用该选项。

参数：

**ident** –

代表主键的标量、元组或字典。对于复合（例如多列）主键，应传递元组或字典。

对于单列主键，标量调用形式通常是最方便的。如果一行的主键是值“5”，则调用如下所示：

```py
my_object = query.get(5)
```

元组形式包含主键值，通常按照它们对应于映射的`Table` 对象的主键列的顺序，或者如果使用了`Mapper.primary_key` 配置参数，则按照该参数的顺序使用。例如，如果一行的主键由整数数字“5, 10”表示，调用将如下所示：

```py
my_object = query.get((5, 10))
```

字典形式应将映射属性名称作为每个主键元素对应的键。如果映射类具有 `id`、`version_id` 作为存储对象主键值的属性，则调用如下：

```py
my_object = query.get({"id": 5, "version_id": 10})
```

版本 1.3 中的新功能：`Query.get()` 方法现在可选地接受属性名称到值的字典，以指示主键标识符。

返回：

对象实例，或 `None`。

```py
method get_children(*, omit_attrs: Tuple[str, ...] = (), **kw: Any) → Iterable[HasTraverseInternals]
```

*继承自* `HasTraverseInternals.get_children()` *方法的* `HasTraverseInternals`

返回此 `HasTraverseInternals` 的直接子级 `HasTraverseInternals` 元素。

这用于访问遍历。

**kw 可能包含改变返回的集合的标志，例如为了减少更大的遍历而返回子集，或者从不同上下文（例如模式级别集合而不是从子句级别）返回子项。

```py
method get_execution_options() → _ImmutableExecuteOptions
```

获取在执行期间生效的非 SQL 选项。

版本 1.3 中的新功能。

另请参阅

`Query.execution_options()`

`Select.get_execution_options()` - v2 可比较方法。

```py
attribute get_label_style
```

检索当前的标签样式。

版本 1.4 中的新功能。

另请参阅

`Select.get_label_style()` - v2 等效方法。

```py
method group_by(_Query__first: Literal[None, False, _NoArg.NO_ARG] | _ColumnExpressionOrStrLabelArgument[Any] = _NoArg.NO_ARG, *clauses: _ColumnExpressionOrStrLabelArgument[Any]) → Self
```

对查询应用一个或多个 GROUP BY 准则，并返回新生成的 `Query`。

所有现有的 GROUP BY 设置都可以通过传递 `None` 来抑制 - 这也会抑制映射器上配置的任何 GROUP BY。

另请参阅

这些部分描述了 GROUP BY 的 2.0 风格 调用，但同样适用于 `Query`：

带有 GROUP BY / HAVING 的聚合函数 - 在 SQLAlchemy 统一教程 中

按标签排序或分组 - 在 SQLAlchemy 统一教程 中

`Select.group_by()` - v2 等效方法。

```py
method having(*having: _ColumnExpressionArgument[bool]) → Self
```

对查询应用 HAVING 准则，并返回新生成的 `Query`。

`Query.having()` 与 `Query.group_by()` 结合使用。

HAVING 条件使得可以对像 COUNT、SUM、AVG、MAX 和 MIN 这样的聚合函数使用过滤器，例如：

```py
q = session.query(User.id).\
 join(User.addresses).\
 group_by(User.id).\
 having(func.count(Address.id) > 2)
```

另请参见

`Select.having()` - v2 等效方法。

```py
method instances(result_proxy: CursorResult[Any], context: QueryContext | None = None) → Any
```

在给定 `CursorResult` 和 `QueryContext` 的情况下返回 ORM 结果。

自版本 2.0 起已弃用：`Query.instances()` 方法已弃用，并将在以后的版本中删除。请改用 Select.from_statement() 方法或与 Session.execute() 结合使用的 aliased() 构造。

```py
method intersect(*q: Query) → Self
```

对此查询与一个或多个查询进行 INTERSECT 操作。

与 `Query.union()` 的工作方式相同。查看该方法以获取使用示例。

另请参见

`Select.intersect()` - v2 等效方法。

```py
method intersect_all(*q: Query) → Self
```

对此查询与一个或多个查询进行 INTERSECT ALL 操作。

与 `Query.union()` 的工作方式相同。查看该方法以获取使用示例。

另请参见

`Select.intersect_all()` - v2 等效方法。

```py
attribute is_single_entity
```

指示此 `Query` 返回元组还是单个实体。

如果此查询为其结果列表中的每个实例返回单个实体，则返回 True，如果此查询为每个结果返回实体元组，则返回 False。

新版本 1.3.11 中新增。

另请参见

`Query.only_return_tuples()`

```py
method join(target: _JoinTargetArgument, onclause: _OnClauseArgument | None = None, *, isouter: bool = False, full: bool = False) → Self
```

创建针对此 `Query` 对象的条件的 SQL JOIN，并以生成方式应用，返回新生成的 `Query`。

**简单关系连接**

考虑两个类 `User` 和 `Address` 之间的映射，其中关系 `User.addresses` 表示与每个 `User` 关联的 `Address` 对象的集合。`Query.join()` 的最常见用法是沿着这个关系创建 JOIN，使用 `User.addresses` 属性作为指示器来指示如何发生这种情况：

```py
q = session.query(User).join(User.addresses)
```

在上面的例子中，沿着 `User.addresses` 调用 `Query.join()` 将导致大致等同于以下 SQL：

```py
SELECT user.id, user.name
FROM user JOIN address ON user.id = address.user_id
```

在上面的例子中，我们将`User.addresses`称为传递给 `Query.join()` 的“on clause”，也就是说，它指示了“JOIN”的“ON”部分应如何构建。

要构建一系列连接，可以使用多个`Query.join()`调用。关系绑定的属性同时暗示连接的左侧和右侧：

```py
q = session.query(User).\
 join(User.orders).\
 join(Order.items).\
 join(Item.keywords)
```

注意

如上例所示，**每次调用 join()方法的顺序很重要**。例如，如果我们在连接链中指定`User`、然后是`Item`、然后是`Order`，那么 Query 不会正确知道如何连接；在这种情况下，根据传递的参数，它可能会引发一个无法连接的错误，或者它可能会生成无效的 SQL，而数据库会引发一个错误。在正确的实践中，应以使得 JOIN 子句在 SQL 中呈现的方式调用`Query.join()`方法，并且每次调用应该表示与之前内容的清晰关联。

**连接到目标实体或可选择的**

`Query.join()`的第二种形式允许任何映射实体或核心可选择的构造作为目标。在这种用法中，`Query.join()`将尝试沿着两个实体之间的自然外键关系创建一个 JOIN：

```py
q = session.query(User).join(Address)
```

在上述调用形式中，`Query.join()`被调用来自动为我们创建“on 子句”。如果两个实体之间没有外键，或者如果目标实体和左侧已存在的实体之间有多个外键链接，以至于创建连接需要更多信息，则此调用形式最终将引发错误。请注意，在指示对没有任何 ON 子句的目标的连接时，不会考虑 ORM 配置的关系。

**连接到具有 ON 子句的目标**

第三种调用形式允许显式传递目标实体以及 ON 子句。包含 SQL 表达式作为 ON 子句的示例如下所示：

```py
q = session.query(User).join(Address, User.id==Address.user_id)
```

上述形式也可以使用一个与关系绑定的属性作为 ON 子句：

```py
q = session.query(User).join(Address, User.addresses)
```

上述语法对于我们希望连接到特定目标实体的别名的情况可能很有用。如果我们想要两次连接到`Address`，可以使用`aliased()`函数设置两个别名：

```py
a1 = aliased(Address)
a2 = aliased(Address)

q = session.query(User).\
 join(a1, User.addresses).\
 join(a2, User.addresses).\
 filter(a1.email_address=='ed@foo.com').\
 filter(a2.email_address=='ed@bar.com')
```

与关系绑定的调用形式还可以使用`PropComparator.of_type()`方法指定目标实体；一个与上述相同的查询将是：

```py
a1 = aliased(Address)
a2 = aliased(Address)

q = session.query(User).\
 join(User.addresses.of_type(a1)).\
 join(User.addresses.of_type(a2)).\
 filter(a1.email_address == 'ed@foo.com').\
 filter(a2.email_address == 'ed@bar.com')
```

**扩充内置的 ON 子句**

作为为现有关系提供完整自定义 ON 条件的替代方法，可以将`PropComparator.and_()`函数应用于关系属性，以将其他条件合并到 ON 子句中；其他条件将与默认条件使用 AND 组合：

```py
q = session.query(User).join(
 User.addresses.and_(Address.email_address != 'foo@bar.com')
)
```

1.4 版本中新增。

**加入表格和子查询**

加入的目标也可以是任何表格或 SELECT 语句，它可以与目标实体相关联，也可以不相关联。使用适当的`.subquery()`方法以将查询转换为子查询：

```py
subq = session.query(Address).\
 filter(Address.email_address == 'ed@foo.com').\
 subquery()

q = session.query(User).join(
 subq, User.id == subq.c.user_id
)
```

通过使用`aliased()`将子查询链接到实体，可以在特定关系和/或目标实体方面加入子查询：

```py
subq = session.query(Address).\
 filter(Address.email_address == 'ed@foo.com').\
 subquery()

address_subq = aliased(Address, subq)

q = session.query(User).join(
 User.addresses.of_type(address_subq)
)
```

**控制加入来源**

在当前`Query`状态的左侧与我们想要加入的内容不一致的情况下，可以使用`Query.select_from()`方法：

```py
q = session.query(Address).select_from(User).\
 join(User.addresses).\
 filter(User.name == 'ed')
```

这将产生类似于以下的 SQL：

```py
SELECT address.* FROM user
 JOIN address ON user.id=address.user_id
 WHERE user.name = :name_1
```

另请参阅

`Select.join()` - v2 等效方法。

参数：

+   `*props` – `Query.join()`的传入参数，现代用法中的 props 集合应被视为一种或两种参数形式，要么是一个“目标”实体或 ORM 属性绑定的关系，要么是一个目标实体加上一个“on 子句”，该子句可以是 SQL 表达式或 ORM 属性绑定的关系。

+   `isouter=False` – 如果为 True，则使用的连接将是左外连接，就像调用`Query.outerjoin()`方法一样。

+   `full=False` – 渲染 FULL OUTER JOIN；意味着`isouter`。

```py
method label(name: str | None) → Label[Any]
```

返回由此`Query`表示的完整 SELECT 语句，转换为具有给定名称标签的标量子查询。

另请参阅

`Select.label()` - v2 可比较方法。

```py
attribute lazy_loaded_from
```

使用此`Query`进行惰性加载操作的`InstanceState`。

自 1.4 版本起弃用：此属性应通过`ORMExecuteState.lazy_loaded_from`属性查看，在`SessionEvents.do_orm_execute()`事件的上下文中。

另请参阅

`ORMExecuteState.lazy_loaded_from`

```py
method limit(limit: _LimitOffsetType) → Self
```

将`LIMIT`应用于查询并返回新结果的`Query`。

请参阅

`Select.limit()` - v2 等效方法。

```py
method merge_result(iterator: FrozenResult[Any] | Iterable[Sequence[Any]] | Iterable[object], load: bool = True) → FrozenResult[Any] | Iterable[Any]
```

将一个结果合并到这个`Query`对象的会话中。

自 2.0 版开始弃用：`Query.merge_result()`方法被认为是 SQLAlchemy 1.x 系列的遗留方法，并在 2.0 版中成为遗留构造。 该方法被`merge_frozen_result()`函数取代。（有关 SQLAlchemy 2.0 的背景，请参阅：SQLAlchemy 2.0 - 主要迁移指南）

给定与此相同结构的`Query`返回的迭代器，返回一个相同的结果迭代器，所有映射实例都使用`Session.merge()`合并到会话中。 这是一种优化方法，将合并所有映射实例，保留结果行的结构和未映射列，比直接为每个值显式调用`Session.merge()`方法的方法开销小。

结果的结构是基于这个`Query`的列列表来确定的 - 如果它们不对应，将会发生未经检查的错误。

‘load’参数与`Session.merge()`相同。

有关如何使用`Query.merge_result()`的示例，请参阅示例 Dogpile Caching 的源代码，其中`Query.merge_result()`用于有效地从缓存中恢复状态到目标`Session`中。

```py
method offset(offset: _LimitOffsetType) → Self
```

将`OFFSET`应用于查询并返回新结果的`Query`。

请参阅

`Select.offset()` - v2 等效方法。

```py
method one() → _T
```

返回一个结果或引发异常。

如果查询未选择任何行，则引发`sqlalchemy.orm.exc.NoResultFound`。 如果返回多个对象标识，或者如果返回多行用于仅返回标量值而不是完整身份映射实体的查询，则引发`sqlalchemy.orm.exc.MultipleResultsFound`。

调用`one()`会执行基础查询。

另请参阅

`Query.first()`

`Query.one_or_none()`

`Result.one()` - v2 可比较的方法。

`Result.scalar_one()` - v2 可比较的方法。

```py
method one_or_none() → _T | None
```

返回最多一个结果或引发异常。

如果查询不选择任何行，则返回`None`。如果返回了多个对象标识或者对于只返回标量值而不是完整身份映射实体的查询返回了多行，则会引发`sqlalchemy.orm.exc.MultipleResultsFound`异常。

调用`Query.one_or_none()`会执行基础查询。

另请参阅

`Query.first()`

`Query.one()`

`Result.one_or_none()` - v2 可比较的方法。

`Result.scalar_one_or_none()` - v2 可比较的方法。

```py
method only_return_tuples(value: bool) → Query
```

当设置为 True 时，查询结果将始终是一个`Row`对象。

这可以将通常返回标量的单个实体的查询更改为在所有情况下返回`Row`结果。

另请参阅

`Query.tuples()` - 返回元组，但在类型级别上也将结果类型化为`Tuple`。

`Query.is_single_entity()`

`Result.tuples()` - v2 可比较的方法。

```py
method options(*args: ExecutableOption) → Self
```

返回一个新的`Query`对象，应用给定的映射器选项列表。

大多数提供的选项涉及更改如何加载列和关系映射的属性。

另请参阅

列加载选项

使用加载器选项进行关系加载

```py
method order_by(_Query__first: Literal[None, False, _NoArg.NO_ARG] | _ColumnExpressionOrStrLabelArgument[Any] = _NoArg.NO_ARG, *clauses: _ColumnExpressionOrStrLabelArgument[Any]) → Self
```

对查询应用一个或多个 ORDER BY 条件，并返回新生成的`Query`。

例如：

```py
q = session.query(Entity).order_by(Entity.id, Entity.name)
```

多次调用此方法等同于一次调用并连接所有子句。所有现有的 ORDER BY 条件可以通过仅传递 `None` 来取消。然后可以通过再次调用 `Query.order_by()` 来添加新的 ORDER BY 条件，例如：

```py
# will erase all ORDER BY and ORDER BY new_col alone
q = q.order_by(None).order_by(new_col)
```

另见

这些部分以 2.0 风格 调用 ORDER BY 描述，但也适用于 `Query`：

ORDER BY - 在 SQLAlchemy 统一教程 中

按标签排序或分组 - 在 SQLAlchemy 统一教程 中

`Select.order_by()` - v2 等效方法。

```py
method outerjoin(target: _JoinTargetArgument, onclause: _OnClauseArgument | None = None, *, full: bool = False) → Self
```

对此 `Query` 对象的条件创建一个左外连接，并生成应用，返回新生成的 `Query`。

用法与 `join()` 方法相同。

另见

`Select.outerjoin()` - v2 等效方法。

```py
method params(_Query__params: Dict[str, Any] | None = None, **kw: Any) → Self
```

为在 filter() 中指定的绑定参数添加值。

参数可以使用 **kwargs 指定，或者作为第一个位置参数使用一个可选的字典。两者之所以同时存在是因为 **kwargs 方便，但是一些参数字典包含 unicode 键，此时无法使用 **kwargs。

```py
method populate_existing() → Self
```

返回一个 `Query`，它将在加载时使所有实例过期并刷新，或从当前 `Session` 复用。

自 SQLAlchemy 1.4 起，`Query.populate_existing()` 方法等同于在 ORM 级别使用 `populate_existing` 执行选项。有关此选项的更多背景，请参见 Populate Existing 部分。

```py
method prefix_with(*prefixes: _TextCoercedExpressionArgument[Any], dialect: str = '*') → Self
```

*继承自* `HasPrefixes.prefix_with()` *方法的* `HasPrefixes`

在语句关键字之后添加一个或多个表达式，即 SELECT、INSERT、UPDATE 或 DELETE。生成的。

这用于支持特定于后端的前缀关键字，例如 MySQL 提供的关键字。

例如：

```py
stmt = table.insert().prefix_with("LOW_PRIORITY", dialect="mysql")

# MySQL 5.7 optimizer hints
stmt = select(table).prefix_with(
 "/*+ BKA(t1) */", dialect="mysql")
```

可以通过多次调用 `HasPrefixes.prefix_with()` 来指定多个前缀。

参数：

+   `*prefixes` – 文本或`ClauseElement`构造，将在 INSERT、UPDATE 或 DELETE 关键字之后呈现。

+   `dialect` – 可选的字符串方言名称，将限制仅将此前缀呈现到该方言。

```py
method reset_joinpoint() → Self
```

返回一个新的`Query`，其中“连接点”已重置回查询的基本 FROM 实体。

此方法通常与`Query.join()`方法的`aliased=True`特性一起使用。有关其用法，请参见`Query.join()`中的示例。

```py
method scalar() → Any
```

返回第一个结果的第一个元素，如果没有行，则返回 None。如果返回多行，则引发 MultipleResultsFound。

```py
>>> session.query(Item).scalar()
<Item>
>>> session.query(Item.id).scalar()
1
>>> session.query(Item.id).filter(Item.id < 0).scalar()
None
>>> session.query(Item.id, Item.name).scalar()
1
>>> session.query(func.count(Parent.id)).scalar()
20
```

这会导致底层查询的执行。

另请参阅

`Result.scalar()` - v2 可比较的方法。

```py
method scalar_subquery() → ScalarSelect[Any]
```

返回由此`Query`表示的完整 SELECT 语句，转换为标量子查询。

类似于`SelectBase.scalar_subquery()`。

在 1.4 版本中更改：`Query.scalar_subquery()`方法替换了`Query.as_scalar()`方法。

另请参阅

`Select.scalar_subquery()` - v2 可比较的方法。

```py
method select_from(*from_obj: FromClauseRole | Type[Any] | Inspectable[_HasClauseElement[Any]] | _HasClauseElement[Any]) → Self
```

明确设置此`Query`的 FROM 子句。

`Query.select_from()`通常与`Query.join()`结合使用，以控制在连接的“左”侧选择哪个实体。

此处的实体或可选择对象有效地替换了任何对`Query.join()`的调用的“左边缘”，否则，当没有其他方式建立连接点时，通常默认的“连接点”是`Query`对象的实体列表中的最左边的实体。

典型的例子：

```py
q = session.query(Address).select_from(User).\
 join(User.addresses).\
 filter(User.name == 'ed')
```

这会产生等效于以下 SQL 的结果：

```py
SELECT address.* FROM user
JOIN address ON user.id=address.user_id
WHERE user.name = :name_1
```

参数：

***from_obj** – 用于应用于 FROM 子句的一个或多个实体的集合。实体可以是映射类，`AliasedClass` 对象，`Mapper` 对象以及核心 `FromClause` 元素，如子查询。

另请参阅

`Query.join()` - `Query.join()` 方法。

`Query.select_entity_from()`

`Select.select_from()` - v2 相关的方法。

```py
attribute selectable
```

返回由此 `Query` 发出的 `Select` 对象。

用于 `inspect()` 兼容性，这相当于：

```py
query.enable_eagerloads(False).with_labels().statement
```

```py
method set_label_style(style: SelectLabelStyle) → Self
```

将列标签应用于 Query.statement 的返回值。

表示此 Query 的语句访问器应返回一个 SELECT 语句，该语句对所有列应用标签的形式为 <tablename>_<columnname>；这通常用于消除具有相同名称的多个表的列的歧义性。

当 Query 实际发出 SQL 加载行时，它总是使用列标签。

注意

`Query.set_label_style()` 方法*仅*适用于 `Query.statement` 的输出，*不*适用于 `Query` 本身的任何结果行调用系统，例如 `Query.first()`，`Query.all()` 等。要使用 `Query.set_label_style()` 执行查询，请使用 `Session.execute()` 调用 `Query.statement`：

```py
result = session.execute(
 query
 .set_label_style(LABEL_STYLE_TABLENAME_PLUS_COL)
 .statement
)
```

版本 1.4 中的新功能。

另请参阅

`Select.set_label_style()` - v2 相关的方法。

```py
method slice(start: int, stop: int) → Self
```

计算由给定索引表示的 `Query` 的“切片”，并返回生成的 `Query`。

开始和停止索引的行为类似于 Python 内置函数 `range()` 的参数。此方法提供了一种替代使用 `LIMIT`/`OFFSET` 来获取查询切片的方法。

例如，

```py
session.query(User).order_by(User.id).slice(1, 3)
```

渲染为

```py
SELECT  users.id  AS  users_id,
  users.name  AS  users_name
FROM  users  ORDER  BY  users.id
LIMIT  ?  OFFSET  ?
(2,  1)
```

另请参阅

`Query.limit()`

`Query.offset()`

`Select.slice()` - v2 等效方法。

```py
attribute statement
```

由此查询表示的完整 SELECT 语句。

默认情况下，语句不会对构造应用歧义性标签，除非首先调用`with_labels(True)`。

```py
method subquery(name: str | None = None, with_labels: bool = False, reduce_columns: bool = False) → Subquery
```

返回由此`Query`表示的完整 SELECT 语句，嵌入在`Alias`中。

查询中的急切 JOIN 生成被禁用。

另请参阅

`Select.subquery()` - v2 可比较方法。

参数：

+   `name` – 要分配为别名的字符串名称；这将通过`FromClause.alias()`传递。如果为`None`，则在编译时将确定性地生成一个名称。

+   `with_labels` – 如果为 True，则首先将在`Query`上调用`with_labels()`，以将表限定标签应用于所有列。

+   `reduce_columns` – 如果为 True，则将在生成的`select()`构造上调用`Select.reduce_columns()`，以删除通过外键或 WHERE 子句等价关系相互引用的同名列。

```py
method suffix_with(*suffixes: _TextCoercedExpressionArgument[Any], dialect: str = '*') → Self
```

*继承自* `HasSuffixes.suffix_with()` *方法的* `HasSuffixes`

在语句后添加一个或多个表达式。

这用于支持特定于后端的后缀关键字在某些构造上。

例如：

```py
stmt = select(col1, col2).cte().suffix_with(
 "cycle empno set y_cycle to 1 default 0", dialect="oracle")
```

可以通过多次调用`HasSuffixes.suffix_with()`来指定多个后缀。

参数：

+   `*suffixes` – 文本或`ClauseElement`构造，将在目标子句后呈现。

+   `dialect` – 可选的字符串方言名称，它将限制此后缀的呈现仅限于该方言。

```py
method tuples() → Query
```

返回此 `Query` 的元组形式。

该方法调用了 `Query.only_return_tuples()` 方法，并将其值设为 `True`，这本身就确保了这个 `Query` 将始终返回 `Row` 对象，即使查询只针对单个实体也是如此。它还在类型级别返回一个“类型化”的查询，如果可能的话，将结果行类型化为带有类型的 `Tuple` 对象。

此方法可以与 `Result.tuples()` 方法进行比较，后者返回“self”，但从类型的角度来看，返回一个对象，该对象将为结果生成带有类型的 `Tuple` 对象。仅当此 `Query` 对象已经是一个类型化的查询对象时，类型才会生效。

2.0 版本中的新功能。

另请参阅

`Result.tuples()` - v2 等效方法。

```py
method union(*q: Query) → Self
```

生成该查询与一个或多个查询的 UNION。

例如：

```py
q1 = sess.query(SomeClass).filter(SomeClass.foo=='bar')
q2 = sess.query(SomeClass).filter(SomeClass.bar=='foo')

q3 = q1.union(q2)
```

该方法接受多个查询对象，以控制嵌套级别。例如一系列的 `union()` 调用：

```py
x.union(y).union(z).all()
```

将在每个 `union()` 上进行嵌套，并产生：

```py
SELECT * FROM (SELECT * FROM (SELECT * FROM X UNION
 SELECT * FROM y) UNION SELECT * FROM Z)
```

而:

```py
x.union(y, z).all()
```

产生：

```py
SELECT * FROM (SELECT * FROM X UNION SELECT * FROM y UNION
 SELECT * FROM Z)
```

请注意，许多数据库后端不允许在 UNION、EXCEPT 等中调用的查询上渲染 ORDER BY。要禁用所有 ORDER BY 子句，包括在映射器上配置的子句，请发出 `query.order_by(None)` - 结果的 `Query` 对象不会在其 SELECT 语句中渲染 ORDER BY。

另请参阅

`Select.union()` - v2 等效方法。

```py
method union_all(*q: Query) → Self
```

生成该查询与一个或多个查询的 UNION ALL。

与 `Query.union()` 的工作方式相同。查看该方法以获取用法示例。

另请参阅

`Select.union_all()` - v2 等效方法。

```py
method update(values: Dict[_DMLColumnArgument, Any], synchronize_session: SynchronizeSessionArgument = 'auto', update_args: Dict[Any, Any] | None = None) → int
```

使用任意 WHERE 子句执行 UPDATE。

更新数据库中与此查询匹配的行。

例如：

```py
sess.query(User).filter(User.age == 25).\
 update({User.age: User.age - 10}, synchronize_session=False)

sess.query(User).filter(User.age == 25).\
 update({"age": User.age - 10}, synchronize_session='evaluate')
```

警告

查看 ORM-启用的 INSERT、UPDATE 和 DELETE 语句部分以获取重要的注意事项和警告，包括在使用任意 UPDATE 和 DELETE 与映射器继承配置时的限制。

参数：

+   `values` – 一个带有属性名的字典，或者作为键的映射属性或 SQL 表达式，以及作为值的文字值或 SQL 表达式。 如果需要参数顺序模式，则可以将值作为 2 元组列表传递； 这需要还将 `update.preserve_parameter_order` 标志传递给 `Query.update.update_args` 字典。

+   `synchronize_session` – 选择更新会话中对象属性的策略。 有关这些策略的讨论，请参阅 启用 ORM 的 INSERT、UPDATE 和 DELETE 语句 部分。

+   `update_args` – 可选字典，如果存在，则作为对象的 `**kw` 传递给底层的 `update()` 构造。 可以用于传递特定于方言的参数，如 `mysql_limit`，以及其他特殊参数，如 `update.preserve_parameter_order`。

返回：

数据库的“行计数”功能返回的匹配行数。

另请参阅

启用 ORM 的 INSERT、UPDATE 和 DELETE 语句

```py
method value(column: _ColumnExpressionArgument[Any]) → Any
```

返回与给定列表对应的标量结果。

自版本 1.4 起弃用：`Query.value()`已弃用，将在将来的版本中删除。 请使用 `Query.with_entities()` 与 `Query.scalar()` 结合使用。

```py
method values(*columns: _ColumnsClauseArgument[Any]) → Iterable[Any]
```

返回一个迭代器，产生与给定列列表对应的结果元组

自版本 1.4 起弃用：`Query.values()`已弃用，将在将来的版本中删除。 请使用 `Query.with_entities()`

```py
method where(*criterion: _ColumnExpressionArgument[bool]) → Self
```

`Query.filter()`的同义词。

版本 1.4 中的新功能。

另请参阅

`Select.where()` - v2 等效方法。

```py
attribute whereclause
```

只读属性，返回此查询的当前 WHERE 条件。

此返回值是一个 SQL 表达式构造，如果没有建立条件，则为 `None`。

另请参阅

`Select.whereclause` - v2 等效属性。

```py
method with_entities(*entities: _ColumnsClauseArgument[Any], **_Query__kw: Any) → Query[Any]
```

返回一个用给定实体替换 SELECT 列的新 `Query`。

例如：

```py
# Users, filtered on some arbitrary criterion
# and then ordered by related email address
q = session.query(User).\
 join(User.address).\
 filter(User.name.like('%ed%')).\
 order_by(Address.email)

# given *only* User.id==5, Address.email, and 'q', what
# would the *next* User in the result be ?
subq = q.with_entities(Address.email).\
 order_by(None).\
 filter(User.id==5).\
 subquery()
q = q.join((subq, subq.c.email < Address.email)).\
 limit(1)
```

另请参阅

`Select.with_only_columns()` - v2 可比较的方法。

```py
method with_for_update(*, nowait: bool = False, read: bool = False, of: _ForUpdateOfArgument | None = None, skip_locked: bool = False, key_share: bool = False) → Self
```

返回一个带有指定选项的新 `Query`，用于 `FOR UPDATE` 子句。

此方法的行为与 `GenerativeSelect.with_for_update()` 相同。当不带参数调用时，生成的 `SELECT` 语句将附加一个 `FOR UPDATE` 子句。当指定其他参数时，后端特定选项，如 `FOR UPDATE NOWAIT` 或 `LOCK IN SHARE MODE` 可以生效。

例如：

```py
q = sess.query(User).populate_existing().with_for_update(nowait=True, of=User)
```

在 PostgreSQL 后端上执行上述查询将呈现如下：

```py
SELECT users.id AS users_id FROM users FOR UPDATE OF users NOWAIT
```

警告

在急加载关系的上下文中使用 `with_for_update` 不受 SQLAlchemy 官方支持或推荐，并且可能无法与各种数据库后端上的某些查询一起使用。当成功使用 `with_for_update` 与涉及 `joinedload()` 的查询时，SQLAlchemy 将尝试发出锁定所有涉及表的 SQL。

注意

通常建议在使用 `Query.with_for_update()` 方法时结合使用 `Query.populate_existing()` 方法。`Query.populate_existing()` 的目的是强制将从 SELECT 中读取的所有数据填充到返回的 ORM 对象中，即使这些对象已经在标识映射中。

另请参阅

`GenerativeSelect.with_for_update()` - 具有完整参数和行为描述的核心级方法。

`Query.populate_existing()` - 覆盖已加载到标识映射中的对象的属性。

```py
method with_hint(selectable: _FromClauseArgument, text: str, dialect_name: str = '*') → Self
```

*继承自* `HasHints.with_hint()` *方法的* `HasHints` *方法*

为给定的可选择对象添加索引或其他执行上下文提示到此 `Select` 或其他可选择对象。

提示文本在使用的数据库后端的适当位置呈现，相对于作为 `selectable` 参数传递的给定 `Table` 或 `Alias`。方言实现通常使用 Python 字符串替换语法，其中的令牌 `%(name)s` 用于呈现表或别名的名称。例如，当使用 Oracle 时，以下内容：

```py
select(mytable).\
 with_hint(mytable, "index(%(name)s ix_mytable)")
```

SQL 渲染如下：

```py
select /*+ index(mytable ix_mytable) */ ... from mytable
```

`dialect_name` 选项将限制特定后端的特定提示的呈现。例如，同时为 Oracle 和 Sybase 添加提示：

```py
select(mytable).\
 with_hint(mytable, "index(%(name)s ix_mytable)", 'oracle').\
 with_hint(mytable, "WITH INDEX ix_mytable", 'mssql')
```

也请参见

`Select.with_statement_hint()`

```py
method with_labels() → Self
```

自 2.0 版本起已废弃：`Query.with_labels()` 和 `Query.apply_labels()` 方法被认为是 SQLAlchemy 1.x 系列的遗留代码，并在 2.0 版本中成为遗留构造。请改用 `set_label_style(LABEL_STYLE_TABLENAME_PLUS_COL)`。 (关于 SQLAlchemy 2.0 的背景信息请参见：SQLAlchemy 2.0 - 主要迁移指南)

```py
method with_parent(instance: object, property: attributes.QueryableAttribute[Any] | None = None, from_entity: _ExternalEntityType[Any] | None = None) → Self
```

添加筛选条件，将给定实例与子对象或集合关联起来，同时使用其属性状态和已建立的 `relationship()` 配置。

自 2.0 版本起已废弃：`Query.with_parent()` 方法被认为是 SQLAlchemy 1.x 系列的遗留代码，并在 2.0 版本中成为遗留构造。请使用独立构造 `with_parent()`。(关于 SQLAlchemy 2.0 的背景信息请参见：SQLAlchemy 2.0 - 主要迁移指南)

该方法使用 `with_parent()` 函数生成子句，其结果传递给 `Query.filter()`。

参数与 `with_parent()` 相同，但给定的属性可以为 None，在这种情况下，将对此 `Query` 对象的目标映射执行搜索。

参数：

+   `instance` – 一个具有一些 `relationship()` 的实例。

+   `property` – 类绑定属性，指示应从实例中使用哪个关系来协调父/子关系。

+   `from_entity` – 将其视为左侧的实体。 默认为 `Query` 本身的“零”实体。

```py
method with_session(session: Session) → Self
```

返回一个将使用给定 `Session` 的 `Query`。 

虽然通常使用 `Session.query()` 方法来实例化 `Query` 对象，但也可以直接构建 `Query` 对象，而无需必须使用 `Session`。 这样的 `Query` 对象，或者已经与不同 `Session` 关联的任何 `Query`，可以使用这种方法产生与目标会话相关联的新的 `Query` 对象：

```py
from sqlalchemy.orm import Query

query = Query([MyClass]).filter(MyClass.id == 5)

result = query.with_session(my_session).one()
```

```py
method with_statement_hint(text: str, dialect_name: str = '*') → Self
```

*继承自* `HasHints.with_statement_hint()` *方法的* `HasHints`

向此 `Select` 或其他可选择对象添加语句提示。

此方法类似于 `Select.with_hint()`，但不需要单独的表，而是适用于整个语句。

此处的提示是特定于后端数据库的，并且可能包括诸如隔离级别、文件指令、提取指令等的指令。

另见

`Select.with_hint()`

`Select.prefix_with()` - 通用的 SELECT 前缀，也可以适用于某些特定于数据库的 HINT 语法，例如 MySQL 优化器提示

```py
method with_transformation(fn: Callable[[Query], Query]) → Query
```

返回一个通过给定函数转换的新的 `Query` 对象。

例如：

```py
def filter_something(criterion):
 def transform(q):
 return q.filter(criterion)
 return transform

q = q.with_transformation(filter_something(x==5))
```

这允许为 `Query` 对象创建临时配方。

```py
method yield_per(count: int) → Self
```

每次仅产生 `count` 行。

此方法的目的是在获取非常大的结果集（> 10K 行）时，将结果批处理为子集合并部分地输出它们，以便 Python 解释器不需要声明非常大的内存区域，这既费时又导致内存使用过多。 当使用适当的 yield-per 设置时（例如，大约为 1000），即使使用缓冲行的 DBAPI（大多数情况下），从获取数十万行的性能也经常会翻倍。

从 SQLAlchemy 1.4 开始，`Query.yield_per()` 方法等同于在 ORM 级别使用 `yield_per` 执行选项。有关此选项的更多背景信息，请参见 使用 Yield Per 获取大型结果集 部分。

另请参见

使用 Yield Per 获取大型结果集

## ORM 特定查询构造

本节已移至 附加 ORM API 构造。
