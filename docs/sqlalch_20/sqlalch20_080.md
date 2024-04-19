# SELECT 及相关构造

> 原文：[`docs.sqlalchemy.org/en/20/core/selectable.html`](https://docs.sqlalchemy.org/en/20/core/selectable.html)

术语“可选择的”指代任何代表数据库行的对象。在 SQLAlchemy 中，这些对象都是 `Selectable` 的子类，其中最突出的是 `Select`，它表示一个 SQL SELECT 语句。`Selectable` 的一个子集是 `FromClause`，它表示可以在 `Select` 语句的 FROM 子句中的对象。`FromClause` 的一个区别性特征是 `FromClause.c` 属性，它是 FROM 子句中包含的所有列的命名空间（这些元素本身是 `ColumnElement` 的子类）。

## 可选择的基本构造

最高级的“FROM 子句”和“SELECT”构造器。

| 对象名称 | 描述 |
| --- | --- |
| except_(*selects) | 返回多个可选项的 `EXCEPT`。 |
| except_all(*selects) | 返回多个可选项的 `EXCEPT ALL`。 |
| exists([__argument]) | 构造一个新的 `Exists` 构造。 |
| intersect(*selects) | 返回多个可选项的 `INTERSECT`。 |
| intersect_all(*selects) | 返回多个可选项的 `INTERSECT ALL`。 |
| select(*entities, **__kw) | 构造一个新的 `Select`。 |
| table(name, *columns, **kw) | 生成一个新的 `TableClause`。 |
| union(*selects) | 返回多个可选项的 `UNION`。 |
| union_all(*selects) | 返回多个可选项的 `UNION ALL`。 |
| values(*columns, [name, literal_binds]) | 构造一个 `Values` 构造。 |

```py
function sqlalchemy.sql.expression.except_(*selects: _SelectStatementForCompoundArgument) → CompoundSelect
```

返回多个可选项的 `EXCEPT`。

返回的对象是一个`CompoundSelect`的实例。

参数：

***selects** – 一个`Select`实例的列表。

```py
function sqlalchemy.sql.expression.except_all(*selects: _SelectStatementForCompoundArgument) → CompoundSelect
```

返回多个可选择的`EXCEPT ALL`。

返回的对象是一个`CompoundSelect`的实例。

参数：

***selects** – 一个`Select`实例的列表。

```py
function sqlalchemy.sql.expression.exists(__argument: _ColumnsClauseArgument[Any] | SelectBase | ScalarSelect[Any] | None = None) → Exists
```

构建一个新的`Exists`构造。

`exists()`可以单独调用以生成一个`Exists`构造，该构造将接受简单的 WHERE 条件：

```py
exists_criteria = exists().where(table1.c.col1 == table2.c.col2)
```

然而，为了在构建 SELECT 时具有更大的灵活性，可以将现有的`Select`构造转换为`Exists`，最方便的方法是利用`SelectBase.exists()`方法：

```py
exists_criteria = (
    select(table2.c.col2).
    where(table1.c.col1 == table2.c.col2).
    exists()
)
```

EXISTS 条件然后在封闭的 SELECT 中使用：

```py
stmt = select(table1.c.col1).where(exists_criteria)
```

上述语句将如下形式：

```py
SELECT col1 FROM table1 WHERE EXISTS
(SELECT table2.col2 FROM table2 WHERE table2.col2 = table1.col1)
```

另请参阅

EXISTS 子查询 - 在 2.0 风格教程中。

`SelectBase.exists()` - 将`SELECT`转换为`EXISTS`子句的方法。

```py
function sqlalchemy.sql.expression.intersect(*selects: _SelectStatementForCompoundArgument) → CompoundSelect
```

返回多个可选择的`INTERSECT`。

返回的对象是一个`CompoundSelect`的实例。

参数：

***selects** – 一个`Select`实例的列表。

```py
function sqlalchemy.sql.expression.intersect_all(*selects: _SelectStatementForCompoundArgument) → CompoundSelect
```

返回多个可选择的`INTERSECT ALL`。

返回的对象是一个`CompoundSelect`的实例。

参数：

***selects** – 一个`Select`实例的列表。

```py
function sqlalchemy.sql.expression.select(*entities: _ColumnsClauseArgument[Any], **__kw: Any) → Select[Any]
```

构建一个新的`Select`。

新版本 1.4 中：- `select()`函数现在可以按位置接受列参数。顶层的`select()`函数将根据传入的参数自动使用 1.x 或 2.x 风格的 API；使用来自`sqlalchemy.future`模块的`select()`将强制使用仅使用 2.x 风格的构造函数。

类似的功能也可通过任何`FromClause`上的`FromClause.select()`方法获得。

另请参阅

使用 SELECT 语句 - 在 SQLAlchemy 统一教程中

参数：

***entities** –

要从中选择的实体。对于核心用法，这通常是一系列将形成结果语句的列子句的`ColumnElement`和/或`FromClause`对象。对于那些是`FromClause`的实例的对象（通常是`Table`或`Alias`对象），`FromClause.c` 集合被提取出来形成`ColumnElement`对象的集合。

此参数还将接受`TextClause`构造，以及 ORM 映射的类。

```py
function sqlalchemy.sql.expression.table(name: str, *columns: ColumnClause[Any], **kw: Any) → TableClause
```

创建一个新的`TableClause`。

返回的对象是`TableClause`的一个实例，它表示架构级别的`Table`对象的“句法”部分。它可用于构建轻量级的表构造。

参数：

+   `name` – 表的名称。

+   `columns` – 一个`column()` 构造的集合。

+   `schema` –

    此表的架构名称。

    新版本 1.3.18 中：`table()`现在可以接受一个`schema`参数。

```py
function sqlalchemy.sql.expression.union(*selects: _SelectStatementForCompoundArgument) → CompoundSelect
```

返回多个可选择的`UNION`。

返回的对象是`CompoundSelect`的一个实例。

所有`FromClause`子类上都有一个类似的`union()`方法。

参数：

+   `*selects` – 一个`Select`实例列表。

+   `**kwargs` – 可用的关键字参数与`select()`的相同。

```py
function sqlalchemy.sql.expression.union_all(*selects: _SelectStatementForCompoundArgument) → CompoundSelect
```

返回多个可选择的`UNION ALL`。

返回的对象是`CompoundSelect`的一个实例。

所有`FromClause`子类上都有一个类似的`union_all()`方法。

参数：

***selects** – 一个`Select`实例列表。

```py
function sqlalchemy.sql.expression.values(*columns: ColumnClause[Any], name: str | None = None, literal_binds: bool = False) → Values
```

构建一个`Values`构造。

列表达式和`Values`的实际数据在两个独立的步骤中给出。构造函数通常接收列表达式，通常作为`column()`构造，并且数据通过`Values.data()`方法传递为一个列表，可以多次调用以添加更多数据，例如：

```py
from sqlalchemy import column
from sqlalchemy import values

value_expr = values(
    column('id', Integer),
    column('name', String),
    name="my_values"
).data(
    [(1, 'name1'), (2, 'name2'), (3, 'name3')]
)
```

参数：

+   `*columns` – 列表达式，通常使用`column()`对象组成。

+   `name` – 此 VALUES 构造的名称。如果省略，VALUES 构造将在 SQL 表达式中无名。不同的后端可能对此有不同的要求。

+   `literal_binds` – 默认为 False。是否在 SQL 输出中内联呈现数据值，而不是使用绑定参数。## 可选择修饰符构造函数

此处列出的函数通常作为`FromClause`和`Selectable`元素的方法更常见，例如，`alias()`函数通常通过`FromClause.alias()`方法调用。

| 对象名称 | 描述 |
| --- | --- |
| alias(selectable[, name, flat]) | 返回给定`FromClause`的命名别名。 |
| cte(selectable[, name, recursive]) | 返回一个新的 `CTE`，或者公共表达式实例。 |
| join(left, right[, onclause, isouter, ...]) | 生成一个给定两个`FromClause`表达式的`Join`对象。 |
| lateral(selectable[, name]) | 返回一个`Lateral`对象。 |
| outerjoin(left, right[, onclause, full]) | 返回一个 `OUTER JOIN` 子句元素。 |
| tablesample(selectable, sampling[, name, seed]) | 返回一个`TableSample`对象。 |

```py
function sqlalchemy.sql.expression.alias(selectable: FromClause, name: str | None = None, flat: bool = False) → NamedFromClause
```

返回给定`FromClause`的命名别名。

对于`Table`和`Join`对象，返回类型为`Alias`对象。其他类型的`NamedFromClause`对象可能会针对其他类型的`FromClause`对象返回。

命名别名表示任何在 SQL 中分配了替代名称的`FromClause`，通常在生成时使用 `AS` 子句，例如 `SELECT * FROM table AS aliasname`。

等效功能可通过`FromClause.alias()`方法在所有`FromClause`对象上使用。

参数：

+   `selectable` – 任何`FromClause`子类，例如表格，选择语句等。

+   `name` – 要分配为别名的字符串名称。如果为`None`，则将在编译时确定性地生成一个名称。确定性意味着该名称保证与同一语句中使用的其他构造唯一，并且对于同一语句对象的每次连续编译也将是相同的名称。

+   `flat` – 如果给定的可选对象是 `Join` 的实例，则将传递给 `Join.alias()` - 有关详细信息，请参阅 `Join.alias()`。

```py
function sqlalchemy.sql.expression.cte(selectable: HasCTE, name: str | None = None, recursive: bool = False) → CTE
```

返回一个新的 `CTE` 或公共表达式实例。

请参阅 `HasCTE.cte()` 了解 CTE 用法的详细信息。

```py
function sqlalchemy.sql.expression.join(left: _FromClauseArgument, right: _FromClauseArgument, onclause: _OnClauseArgument | None = None, isouter: bool = False, full: bool = False) → Join
```

给定两个 `FromClause` 表达式，生成一个 `Join` 对象。

例如：

```py
j = join(user_table, address_table,
         user_table.c.id == address_table.c.user_id)
stmt = select(user_table).select_from(j)
```

将会生成类似的 SQL：

```py
SELECT user.id, user.name FROM user
JOIN address ON user.id = address.user_id
```

类似的功能可在任何 `FromClause` 对象（例如 `Table`）上使用 `FromClause.join()` 方法。

参数：

+   `left` – 连接的左侧。

+   `right` – 连接的右侧；这是任何 `FromClause` 对象，例如 `Table` 对象，也可以是 ORM 映射的类等可选对象。

+   `onclause` – 表示连接的 ON 子句的 SQL 表达式。如果保留为 `None`，`FromClause.join()` 将尝试根据外键关系连接两个表。

+   `isouter` – 如果为 True，则渲染一个 LEFT OUTER JOIN，而不是 JOIN。

+   `full` – 如果为 True，则渲染一个 FULL OUTER JOIN，而不是 JOIN。

另请参见

`FromClause.join()` - 方法形式，基于给定的左侧。

`Join` - 生成的对象类型。

```py
function sqlalchemy.sql.expression.lateral(selectable: SelectBase | _FromClauseArgument, name: str | None = None) → LateralFromClause
```

返回一个 `Lateral` 对象。

`Lateral` 是表示具有 LATERAL 关键字的子查询的 `Alias` 子类。

LATERAL 子查询的特殊行为是，它出现在封闭 SELECT 的 FROM 子句中，但可以与该 SELECT 的其他 FROM 子句相关联。这是一种特殊情况的子查询，仅受一小部分后端支持，目前支持较新版本的 PostgreSQL。

另请参见

LATERAL 关联 - 使用概述。

```py
function sqlalchemy.sql.expression.outerjoin(left: _FromClauseArgument, right: _FromClauseArgument, onclause: _OnClauseArgument | None = None, full: bool = False) → Join
```

返回一个 `OUTER JOIN` 子句元素。

返回的对象是 `Join` 的实例。

类似的功能也可通过任何 `FromClause` 上的 `FromClause.outerjoin()` 方法获得。

参数：

+   `left` – 连接的左侧。

+   `right` – 连接的右侧。

+   `onclause` – 可选的 `ON` 子句条件，否则会从左侧和右侧之间建立的外键关系中派生。

要将连接链在一起，请使用结果 `Join` 对象上的 `FromClause.join()` 或 `FromClause.outerjoin()` 方法。

```py
function sqlalchemy.sql.expression.tablesample(selectable: _FromClauseArgument, sampling: float | Function[Any], name: str | None = None, seed: roles.ExpressionElementRole[Any] | None = None) → TableSample
```

返回一个 `TableSample` 对象。

`TableSample` 是一个 `Alias` 子类，表示应用了 TABLESAMPLE 子句的表。 `tablesample()` 也可从 `FromClause` 类中通过 `FromClause.tablesample()` 方法获得。

TABLESAMPLE 子句允许从表中随机选择近似百分比的行。它支持多种采样方法，最常见的是 BERNOULLI 和 SYSTEM。

例如：

```py
from sqlalchemy import func

selectable = people.tablesample(
            func.bernoulli(1),
            name='alias',
            seed=func.random())
stmt = select(selectable.c.people_id)
```

假设 `people` 有一个列 `people_id`，上述语句将呈现为：

```py
SELECT alias.people_id FROM
people AS alias TABLESAMPLE bernoulli(:bernoulli_1)
REPEATABLE (random())
```

参数：

+   `sampling` – 一个介于 0 和 100 之间的 `float` 百分比或 `Function`。

+   `name` – 可选别名名称

+   `seed` – 任意实值 SQL 表达式。当指定时，还会呈现 REPEATABLE 子句。

## 可选择的类文档

这里的类是使用 可选择的基础构造函数 和 可选择的修饰符构造函数 列出的构造函数生成的。

| 对象名称 | 描述 |
| --- | --- |
| 别名 | 表示表或可选择的别名（AS）。 |
| AliasedReturnsRows | 对表、子查询和其他可选择的别名的基类。 |
| CompoundSelect | 形成 `UNION`, `UNION ALL`, 和其他基于 SELECT 的集合操作的基础。 |
| CTE | 表示公共表达式。 |
| Executable | 将`ClauseElement`标记为支持执行。 |
| Exists | 表示一个`EXISTS`子句。 |
| FromClause | 表示可在`SELECT`语句的`FROM`子句中使用的元素。 |
| GenerativeSelect | SELECT 语句的基类，可以添加额外的元素。 |
| HasCTE | 声明一个类包含 CTE 支持的 Mixin。 |
| HasPrefixes |  |
| HasSuffixes |  |
| Join | 表示两个`FromClause`元素之间的`JOIN`构造。 |
| Lateral | 表示一个 LATERAL 子查询。 |
| ReturnsRows | Core 构造的基类，具有某种可以表示行的列的概念。 |
| ScalarSelect | 表示一个标量子查询。 |
| ScalarValues | 表示可用作语句中 COLUMN 元素的标量`VALUES`构造。 |
| Select | 表示一个`SELECT`语句。 |
| Selectable | 将类标记为可选择。 |
| SelectBase | SELECT 语句的基类。 |
| Subquery | 表示一个 SELECT 的子查询。 |
| TableClause | 表示最小的“表”构造。 |
| TableSample | 表示一个 TABLESAMPLE 子句。 |
| TableValuedAlias | 对“表值”SQL 函数的别名。 |
| TextualSelect | 在`TextClause`构造内部包装一个`SelectBase`接口。 |
| Values | 表示可用作语句中 FROM 元素的`VALUES`构造。 |

```py
class sqlalchemy.sql.expression.Alias
```

表示一个表或可选择的别名（AS）。

表示别名，通常使用`AS`关键字（或在某些数据库上不使用关键字，如 Oracle）应用于 SQL 语句中的任何表或子选择。

此对象是从`alias()`模块级函数以及所有`FromClause`子类上可用的`FromClause.alias()`方法构造的。

另请参阅

`FromClause.alias()`

**成员**

inherit_cache

**类签名**

类`sqlalchemy.sql.expression.Alias`（`sqlalchemy.sql.roles.DMLTableRole`，`sqlalchemy.sql.expression.FromClauseAlias`）

```py
attribute inherit_cache: bool | None = True
```

指示此`HasCacheKey`实例是否应使用其直接超类使用的缓存键生成方案。

此属性默认为`None`，表示构造尚未考虑是否应该参与缓存；这在功能上等同于将值设置为`False`，除了还会发出警告。

如果与此类本地属性而不是其超类相关的属性不会更改对象对应的 SQL，则可以将此标志设置为`True`。

另请参阅

启用自定义构造的缓存支持 - 设置第三方或用户定义的 SQL 构造的`HasCacheKey.inherit_cache`属性的通用指南。

```py
class sqlalchemy.sql.expression.AliasedReturnsRows
```

别名对表、子查询和其他可选择项的基类。

**成员**

description, is_derived_from(), original

**类签名**

类`sqlalchemy.sql.expression.AliasedReturnsRows`（`sqlalchemy.sql.expression.NoInit`，`sqlalchemy.sql.expression.NamedFromClause`）

```py
attribute description
```

```py
method is_derived_from(fromclause: FromClause | None) → bool
```

如果此`FromClause`是从给定的`FromClause`“派生”的话，则返回`True`。

一个例子是表的别名是从该表派生的。

```py
attribute original
```

适用于引用 Alias.original 的方言的遗留。

```py
class sqlalchemy.sql.expression.CompoundSelect
```

形成`UNION`，`UNION ALL`和其他基于 SELECT 的集合操作的基础。

另请参阅

`union()`

`union_all()`

`intersect()`

`intersect_all()`

`except()`

`except_all()`

**成员**

add_cte(), alias(), as_scalar(), c, corresponding_column(), cte(), execution_options(), exists(), exported_columns, fetch(), get_execution_options(), get_label_style(), group_by(), is_derived_from(), label(), lateral(), limit(), offset(), options(), order_by(), replace_selectable(), scalar_subquery(), select(), selected_columns, self_group(), set_label_style(), slice(), subquery(), with_for_update()

**类签名**

类`sqlalchemy.sql.expression.CompoundSelect` (`sqlalchemy.sql.expression.HasCompileState`, `sqlalchemy.sql.expression.GenerativeSelect`, `sqlalchemy.sql.expression.ExecutableReturnsRows`)

```py
method add_cte(*ctes: CTE, nest_here: bool = False) → Self
```

*继承自* `HasCTE.add_cte()` *方法的* `HasCTE`

向此语句添加一个或多个`CTE`构造。

此方法将给定的`CTE`构造与父语句关联，以便它们将无条件地在最终语句的 WITH 子句中呈现，即使在语句或任何子选择中没有其他地方引用它们。

当设置为 True 时，可选的`HasCTE.add_cte.nest_here`参数将使每个给定的`CTE`在与此语句直接一起呈现的 WITH 子句中呈现，而不是将其移动到最终呈现语句的顶部，即使此语句作为较大语句中的子查询呈现。

此方法有两个一般用途。一个是嵌入一些用途的 CTE 语句，而不被明确引用，例如将 DML 语句（如 INSERT 或 UPDATE）作为 CTE 内联到可能间接从其结果中获取的主语句中的用例。另一个是提供对应该保持直接呈现的特定一系列 CTE 构造的放置的控制，这些构造可能嵌套在较大语句中。

例如：

```py
from sqlalchemy import table, column, select
t = table('t', column('c1'), column('c2'))

ins = t.insert().values({"c1": "x", "c2": "y"}).cte()

stmt = select(t).add_cte(ins)
```

将呈现为：

```py
WITH anon_1 AS
(INSERT INTO t (c1, c2) VALUES (:param_1, :param_2))
SELECT t.c1, t.c2
FROM t
```

在上面的示例中，“anon_1” CTE 在 SELECT 语句中未被引用，但仍完成运行 INSERT 语句的任务。

类似地，在与 DML 相关的上下文中，使用 PostgreSQL `Insert`构造生成“upsert”：

```py
from sqlalchemy import table, column
from sqlalchemy.dialects.postgresql import insert

t = table("t", column("c1"), column("c2"))

delete_statement_cte = (
    t.delete().where(t.c.c1 < 1).cte("deletions")
)

insert_stmt = insert(t).values({"c1": 1, "c2": 2})
update_statement = insert_stmt.on_conflict_do_update(
    index_elements=[t.c.c1],
    set_={
        "c1": insert_stmt.excluded.c1,
        "c2": insert_stmt.excluded.c2,
    },
).add_cte(delete_statement_cte)

print(update_statement)
```

上述语句呈现为：

```py
WITH deletions AS
(DELETE FROM t WHERE t.c1 < %(c1_1)s)
INSERT INTO t (c1, c2) VALUES (%(c1)s, %(c2)s)
ON CONFLICT (c1) DO UPDATE SET c1 = excluded.c1, c2 = excluded.c2
```

从版本 1.4.21 开始新增。

参数：

+   `*ctes` –

    零个或多个`CTE`构造。

    从版本 2.0 开始更改：接受多个 CTE 实例

+   `nest_here` –

    如果为 True，则给定的 CTE 或 CTEs 将被呈现为当它们被添加到此`HasCTE`时指定`HasCTE.cte.nesting`标志为`True`。假设给定的 CTEs 在外部封闭语句中也没有被引用，当给出此标志时，给定的 CTEs 应在此语句级别呈现。

    从版本 2.0 开始新增。

    另请参阅

    `HasCTE.cte.nesting`

```py
method alias(name: str | None = None, flat: bool = False) → Subquery
```

*继承自* `SelectBase.alias()` *方法的* `SelectBase`

返回针对此 `SelectBase` 的命名子查询。

对于 `SelectBase`（而不是 `FromClause`），这将返回一个行为大部分与用于 `FromClause` 的 `Alias` 对象相同的 `Subquery` 对象。

自版本 1.4 起更改：`SelectBase.alias()` 方法现在是 `SelectBase.subquery()` 方法的同义词。

```py
method as_scalar() → ScalarSelect[Any]
```

*继承自* `SelectBase.as_scalar()` *方法的* `SelectBase`

自版本 1.4 起弃用：`SelectBase.as_scalar()` 方法已被弃用，并将在将来的版本中移除。请参考 `SelectBase.scalar_subquery()`。

```py
attribute c
```

*继承自* `SelectBase.c` *属性的* `SelectBase`

自版本 1.4 起弃用：`SelectBase.c` 和 `SelectBase.columns` 属性已被弃用，并将在将来的版本中移除；这些属性隐式创建一个子查询，应该明确指定。请先调用 `SelectBase.subquery()` 来创建一个子查询，然后再包含此属性。要访问此 SELECT 对象所选择的列，请使用 `SelectBase.selected_columns` 属性。

```py
method corresponding_column(column: KeyedColumnElement[Any], require_embedded: bool = False) → KeyedColumnElement[Any] | None
```

*继承自* `Selectable.corresponding_column()` *方法的* `Selectable`

给定一个`ColumnElement`，从这个`Selectable`的`Selectable.exported_columns`集合中返回对应于原始`ColumnElement`的导出`ColumnElement`对象，通过一个共同的祖先列。

参数：

+   `column` – 要匹配的目标`ColumnElement`。

+   `require_embedded` – 只返回给定`ColumnElement`的相应列，如果给定的`ColumnElement`实际上存在于此`Selectable`的子元素之内。通常，如果列仅与此`Selectable`的导出列之一共享一个共同的祖先，则列将匹配。

另请参阅

`Selectable.exported_columns` - 用于操作的`ColumnCollection`。

`ColumnCollection.corresponding_column()` - 实现方法。

```py
method cte(name: str | None = None, recursive: bool = False, nesting: bool = False) → CTE
```

*继承自* `HasCTE` *的* `HasCTE.cte()` *方法。

返回一个新的`CTE`或通用表达式实例。

公共表达式是 SQL 标准，其中 SELECT 语句可以利用与主要语句一起指定的辅助语句，使用一个称为“WITH”的子句。还可以采用有关 UNION 的特殊语义，以允许“递归”查询，其中 SELECT 语句可以利用先前已选择的行集。

CTEs 也可以应用于某些数据库上的 DML 构造 UPDATE、INSERT 和 DELETE，既作为与 RETURNING 结合使用时 CTE 行的源，也作为 CTE 行的使用者。

SQLAlchemy 检测到 `CTE` 对象，这些对象与 `Alias` 对象类似，被视为要传递到语句的 FROM 子句以及语句顶部的 WITH 子句的特殊元素。

对于特殊前缀，如 PostgreSQL 的 “MATERIALIZED” 和 “NOT MATERIALIZED”，可以使用 `CTE.prefix_with()` 方法来建立这些。

1.3.13 版本更改：增加了对前缀的支持。特别是 - MATERIALIZED 和 NOT MATERIALIZED。

参数：

+   `name` – 给公共表达式的名称。类似于 `FromClause.alias()`，名称可以保留为 `None`，在这种情况下，将在查询编译时使用匿名符号。

+   `recursive` – 如果为 `True`，将呈现 `WITH RECURSIVE`。递归公共表达式旨在与 UNION ALL 结合使用，以从已选择的行中派生行。

+   `nesting` –

    如果为 `True`，将在引用它的语句中本地呈现 CTE。对于更复杂的情况，还可以使用 `HasCTE.add_cte()` 方法，使用 `HasCTE.add_cte.nest_here` 参数更精确地控制特定 CTE 的精确放置。

    1.4.24 版本新增。

    另请参阅

    `HasCTE.add_cte()`

以下示例包括两个来自 PostgreSQL 文档的示例 [`www.postgresql.org/docs/current/static/queries-with.html`](https://www.postgresql.org/docs/current/static/queries-with.html)，以及其他示例。

示例 1，非递归：

```py
from sqlalchemy import (Table, Column, String, Integer,
                        MetaData, select, func)

metadata = MetaData()

orders = Table('orders', metadata,
    Column('region', String),
    Column('amount', Integer),
    Column('product', String),
    Column('quantity', Integer)
)

regional_sales = select(
                    orders.c.region,
                    func.sum(orders.c.amount).label('total_sales')
                ).group_by(orders.c.region).cte("regional_sales")

top_regions = select(regional_sales.c.region).\
        where(
            regional_sales.c.total_sales >
            select(
                func.sum(regional_sales.c.total_sales) / 10
            )
        ).cte("top_regions")

statement = select(
            orders.c.region,
            orders.c.product,
            func.sum(orders.c.quantity).label("product_units"),
            func.sum(orders.c.amount).label("product_sales")
    ).where(orders.c.region.in_(
        select(top_regions.c.region)
    )).group_by(orders.c.region, orders.c.product)

result = conn.execute(statement).fetchall()
```

示例 2，WITH RECURSIVE：

```py
from sqlalchemy import (Table, Column, String, Integer,
                        MetaData, select, func)

metadata = MetaData()

parts = Table('parts', metadata,
    Column('part', String),
    Column('sub_part', String),
    Column('quantity', Integer),
)

included_parts = select(\
    parts.c.sub_part, parts.c.part, parts.c.quantity\
    ).\
    where(parts.c.part=='our part').\
    cte(recursive=True)

incl_alias = included_parts.alias()
parts_alias = parts.alias()
included_parts = included_parts.union_all(
    select(
        parts_alias.c.sub_part,
        parts_alias.c.part,
        parts_alias.c.quantity
    ).\
    where(parts_alias.c.part==incl_alias.c.sub_part)
)

statement = select(
            included_parts.c.sub_part,
            func.sum(included_parts.c.quantity).
              label('total_quantity')
        ).\
        group_by(included_parts.c.sub_part)

result = conn.execute(statement).fetchall()
```

示例 3，使用 UPDATE 和 INSERT 进行 upsert 的 CTE：

```py
from datetime import date
from sqlalchemy import (MetaData, Table, Column, Integer,
                        Date, select, literal, and_, exists)

metadata = MetaData()

visitors = Table('visitors', metadata,
    Column('product_id', Integer, primary_key=True),
    Column('date', Date, primary_key=True),
    Column('count', Integer),
)

# add 5 visitors for the product_id == 1
product_id = 1
day = date.today()
count = 5

update_cte = (
    visitors.update()
    .where(and_(visitors.c.product_id == product_id,
                visitors.c.date == day))
    .values(count=visitors.c.count + count)
    .returning(literal(1))
    .cte('update_cte')
)

upsert = visitors.insert().from_select(
    [visitors.c.product_id, visitors.c.date, visitors.c.count],
    select(literal(product_id), literal(day), literal(count))
        .where(~exists(update_cte.select()))
)

connection.execute(upsert)
```

示例 4，嵌套 CTE（SQLAlchemy 1.4.24 及以上版本）：

```py
value_a = select(
    literal("root").label("n")
).cte("value_a")

# A nested CTE with the same name as the root one
value_a_nested = select(
    literal("nesting").label("n")
).cte("value_a", nesting=True)

# Nesting CTEs takes ascendency locally
# over the CTEs at a higher level
value_b = select(value_a_nested.c.n).cte("value_b")

value_ab = select(value_a.c.n.label("a"), value_b.c.n.label("b"))
```

上述查询将第二个 CTE 嵌套在第一个内部，如下所示带有内联参数：

```py
WITH
    value_a AS
        (SELECT 'root' AS n),
    value_b AS
        (WITH value_a AS
            (SELECT 'nesting' AS n)
        SELECT value_a.n AS n FROM value_a)
SELECT value_a.n AS a, value_b.n AS b
FROM value_a, value_b
```

可以使用 `HasCTE.add_cte()` 方法来设置相同的 CTE（SQLAlchemy 2.0 及以上版本）：

```py
value_a = select(
    literal("root").label("n")
).cte("value_a")

# A nested CTE with the same name as the root one
value_a_nested = select(
    literal("nesting").label("n")
).cte("value_a")

# Nesting CTEs takes ascendency locally
# over the CTEs at a higher level
value_b = (
    select(value_a_nested.c.n).
    add_cte(value_a_nested, nest_here=True).
    cte("value_b")
)

value_ab = select(value_a.c.n.label("a"), value_b.c.n.label("b"))
```

示例 5，非线性 CTE（SQLAlchemy 1.4.28 及以上版本）：

```py
edge = Table(
    "edge",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("left", Integer),
    Column("right", Integer),
)

root_node = select(literal(1).label("node")).cte(
    "nodes", recursive=True
)

left_edge = select(edge.c.left).join(
    root_node, edge.c.right == root_node.c.node
)
right_edge = select(edge.c.right).join(
    root_node, edge.c.left == root_node.c.node
)

subgraph_cte = root_node.union(left_edge, right_edge)

subgraph = select(subgraph_cte)
```

上述查询将在递归 CTE 中呈现 2 个 UNION：

```py
WITH RECURSIVE nodes(node) AS (
        SELECT 1 AS node
    UNION
        SELECT edge."left" AS "left"
        FROM edge JOIN nodes ON edge."right" = nodes.node
    UNION
        SELECT edge."right" AS "right"
        FROM edge JOIN nodes ON edge."left" = nodes.node
)
SELECT nodes.node FROM nodes
```

另请参阅

`Query.cte()` - `HasCTE.cte()` 的 ORM 版本。

```py
method execution_options(**kw: Any) → Self
```

*继承自* `Executable.execution_options()` *方法的* `Executable`

为语句设置在执行期间生效的非 SQL 选项。

可以在许多范围内设置执行选项，包括每个语句、每个连接或每次执行，使用诸如`Connection.execution_options()`之类的方法和接受选项字典的参数，如`Connection.execute.execution_options`和`Session.execute.execution_options`。

执行选项的主要特征，与其他类型的选项（如 ORM 加载器选项）相反，是**执行选项永远不会影响查询的编译 SQL，只会影响 SQL 语句本身的调用方式或结果的获取方式**。也就是说，执行选项不是 SQL 编译所考虑的内容，也不被视为语句的缓存状态的一部分。

`Executable.execution_options()` 方法是生成的，就像应用于`Engine`和`Query`对象的方法一样，这意味着当调用该方法时，会返回对象的副本，将给定的参数应用于该新副本，但原始对象保持不变：

```py
statement = select(table.c.x, table.c.y)
new_statement = statement.execution_options(my_option=True)
```

这种行为的一个例外是`Connection`对象，其中`Connection.execution_options()`方法明确地**不是**生成的。

可传递给 `Executable.execution_options()` 和其他相关方法和参数字典的选项类型包括由 SQLAlchemy Core 或 ORM 明确消耗的参数，以及未由 SQLAlchemy 定义的任意关键字参数，这意味着可以使用这些方法和/或参数字典来处理与自定义代码交互的用户定义参数，该自定义代码可以使用诸如 `Executable.get_execution_options()` 和 `Connection.get_execution_options()` 的方法访问这些参数，或者在选择的事件钩子中使用专用的 `execution_options` 事件参数，如 `ConnectionEvents.before_execute.execution_options` 或 `ORMExecuteState.execution_options`。例如：

```py
from sqlalchemy import event

@event.listens_for(some_engine, "before_execute")
def _process_opt(conn, statement, multiparams, params, execution_options):
    "run a SQL function before invoking a statement"

    if execution_options.get("do_special_thing", False):
        conn.exec_driver_sql("run_special_function()")
```

在 SQLAlchemy 明确识别的选项范围内，大多数适用于特定类对象而不适用于其他类对象。最常见的执行选项包括：

+   `Connection.execution_options.isolation_level` - 通过 `Engine` 为连接或一类连接设置隔离级别。此选项仅由 `Connection` 或 `Engine` 接受。

+   `Connection.execution_options.stream_results` - 指示应使用服务器端游标获取结果；此选项由 `Connection` 接受，在 `Connection.execute()` 的 `execution_options` 参数上，以及在 SQL 语句对象上的 `Executable.execution_options()` 上，以及像 `Session.execute()` 这样的 ORM 构造函数。

+   `Connection.execution_options.compiled_cache` - 指示一个将用作 `Connection` 或 `Engine` 的 SQL 编译缓存 的字典，以及像 `Session.execute()` 这样的 ORM 方法。可以传递 `None` 来禁用语句的缓存。此选项不被 `Executable.execution_options()` 接受，因为在语句对象中携带编译缓存是不可取的。

+   `Connection.execution_options.schema_translate_map` - 模式翻译映射 功能使用的模式名称的映射，由 `Connection`、`Engine`、`Executable` 接受，以及像 `Session.execute()` 这样的 ORM 构造函数。

另请参阅

`Connection.execution_options()`

`Connection.execute.execution_options`

`Session.execute.execution_options`

ORM 执行选项 - 所有 ORM 特定执行选项的文档

```py
method exists() → Exists
```

*继承自* `SelectBase.exists()` *方法的* `SelectBase`

返回此可选择性的 `Exists` 表示，可用作列表达式。

返回的对象是 `Exists` 的一个实例。

另请参阅

`exists()`

EXISTS 子查询 - 在 2.0 风格 教程中。

版本 1.4 中的新功能。

```py
attribute exported_columns
```

*继承自* `SelectBase.exported_columns` *属性的* `SelectBase`

一个 `ColumnCollection` 代表此 `Selectable` 的“导出”列，不包括 `TextClause` 结构。

一个 `SelectBase` 对象的“导出”列与 `SelectBase.selected_columns` 集合是同义词。

版本 1.4 中的新功能。

另请参阅

`Select.exported_columns`

`Selectable.exported_columns`

`FromClause.exported_columns`

```py
method fetch(count: _LimitOffsetType, with_ties: bool = False, percent: bool = False) → Self
```

*继承自* `GenerativeSelect.fetch()` *方法的* `GenerativeSelect`

返回一个应用了给定 FETCH FIRST 标准的新可选择性。

这是一个数字值，通常在结果选择中呈现为`FETCH {FIRST | NEXT} [ count ] {ROW | ROWS} {ONLY | WITH TIES}`表达式。此功能目前已在 Oracle、PostgreSQL、MSSQL 上实现。

使用 `GenerativeSelect.offset()` 来指定偏移量。

注意

`GenerativeSelect.fetch()` 方法将替换应用了 `GenerativeSelect.limit()` 的任何子句。

自版本 1.4 起新增。

参数：

+   `count` - 一个整数 COUNT 参数，或者提供整数结果的 SQL 表达式。当`percent=True`时，这将表示要返回的行数的百分比，而不是绝对值。传递`None`来重置它。

+   `with_ties` - 当为`True`时，使用 WITH TIES 选项来返回结果集中与`ORDER BY`子句中最后一位并列的任何其他行。在这种情况下，`ORDER BY`可能是强制性的。默认为`False`。

+   `percent` - 当为`True`时，`count`表示要返回的所选行总数的百分比。默认为`False`。

另请参阅

`GenerativeSelect.limit()`

`GenerativeSelect.offset()`

```py
method get_execution_options() → _ExecuteOptions
```

*继承自* `Executable.get_execution_options()` *方法的* `Executable`

获取在执行期间生效的非 SQL 选项。

自版本 1.3 起新增。

另请参阅

`Executable.execution_options()`

```py
method get_label_style() → SelectLabelStyle
```

*继承自* `GenerativeSelect.get_label_style()` *方法的* `GenerativeSelect`

检索当前的标签样式。

自版本 1.4 起新增。

```py
method group_by(_GenerativeSelect__first: Literal[None, _NoArg.NO_ARG] | _ColumnExpressionOrStrLabelArgument[Any] = _NoArg.NO_ARG, *clauses: _ColumnExpressionOrStrLabelArgument[Any]) → Self
```

*继承自* `GenerativeSelect.group_by()` *方法的* `GenerativeSelect`

返回一个应用了给定的 GROUP BY 标准列表的新可选择对象。

通过传递`None`可以抑制所有现有的 GROUP BY 设置。

例如：

```py
stmt = select(table.c.name, func.max(table.c.stat)).\
group_by(table.c.name)
```

参数：

***子句** – 一系列将用于生成 GROUP BY 子句的 `ColumnElement` 构造。

另请参阅

带有 GROUP BY / HAVING 的聚合函数 - 在 SQLAlchemy 统一教程 中

按标签排序或分组 - 在 SQLAlchemy 统一教程 中

```py
method is_derived_from(fromclause: FromClause | None) → bool
```

如果此 `ReturnsRows` 是从给定的 `FromClause` ‘派生’的，则返回 `True`。

一个示例是一个表的别名是从该表派生的。

```py
method label(name: str | None) → Label[Any]
```

*继承自* `SelectBase.label()` *方法的* `SelectBase`

返回此可选择对象的‘标量’表示，嵌入为带有标签的子查询。

另请参阅

`SelectBase.scalar_subquery()`。

```py
method lateral(name: str | None = None) → LateralFromClause
```

*继承自* `SelectBase.lateral()` *方法的* `SelectBase`

返回此 `Selectable` 的 LATERAL 别名。

返回值是顶层 `lateral()` 函数提供的 `Lateral` 构造。

另请参阅

LATERAL 关联 - 用法概述。

```py
method limit(limit: _LimitOffsetType) → Self
```

*继承自* `GenerativeSelect.limit()` *方法的* `GenerativeSelect`

返回应用了给定 LIMIT 条件的新可选择对象。

这是一个通常呈现为 `LIMIT` 表达式的数值值，在生成的选择中。不支持 `LIMIT` 的后端将尝试提供类似的功能。

注意

`GenerativeSelect.limit()` 方法将替换应用的任何子句，使用 `GenerativeSelect.fetch()`。

参数：

**limit** – 整数 LIMIT 参数，或提供整数结果的 SQL 表达式。传递`None`来重置它。

另请参见

`GenerativeSelect.fetch()`

`GenerativeSelect.offset()`

```py
method offset(offset: _LimitOffsetType) → Self
```

*继承自* `GenerativeSelect.offset()` *的* `GenerativeSelect` *方法*

返回一个应用了给定 OFFSET 条件的新可选择项。

这是一个通常渲染为结果选择中的`OFFSET`表达式的数值。不支持`OFFSET`的后端将尝试提供类似的功能。

参数：

**offset** – 整数 OFFSET 参数，或提供整数结果的 SQL 表达式。传递`None`来重置它。

另请参见

`GenerativeSelect.limit()`

`GenerativeSelect.fetch()`

```py
method options(*options: ExecutableOption) → Self
```

*继承自* `Executable.options()` *的* `Executable` *方法*

对该语句应用选项。

从一般意义上讲，选项是任何可以由语句的 SQL 编译器解释的 Python 对象。这些选项可以被特定方言或特定类型的编译器消耗。

最常见的选项类型是应用“急加载”和其他加载行为到 ORM 查询的 ORM 级别选项。然而，选项理论上可以用于许多其他目的。

关于特定类型语句的特定类型选项的背景，请参阅这些选项对象的文档。

自版本 1.4 起更改：- 将`Executable.options()` 添加到核心语句对象，以实现统一的核心/ORM 查询功能目标。

另请参见

列加载选项 - 指的是用于 ORM 查询的使用特定选项的选项

带有加载器选项的关系加载 - 指的是用于 ORM 查询的使用特定选项的选项

```py
method order_by(_GenerativeSelect__first: Literal[None, _NoArg.NO_ARG] | _ColumnExpressionOrStrLabelArgument[Any] = _NoArg.NO_ARG, *clauses: _ColumnExpressionOrStrLabelArgument[Any]) → Self
```

*继承自* `GenerativeSelect.order_by()` *的* `GenerativeSelect` *方法*

返回一个应用了给定 ORDER BY 条件列表的新可选择性。

例如：

```py
stmt = select(table).order_by(table.c.id, table.c.name)
```

多次调用此方法相当于一次性将所有子句连接起来调用一次。 通过单独传递`None`可以取消所有现有的 ORDER BY 条件。 然后可以通过再次调用`Query.order_by()` 来添加新的 ORDER BY 条件，例如：

```py
# will erase all ORDER BY and ORDER BY new_col alone
stmt = stmt.order_by(None).order_by(new_col)
```

参数：

***子句** – 一系列将用于生成 ORDER BY 子句的`ColumnElement`构造。

另请参阅

ORDER BY - 在 SQLAlchemy 统一教程中

按标签排序或分组 - 在 SQLAlchemy 统一教程中

```py
method replace_selectable(old: FromClause, alias: Alias) → Self
```

*继承自* `Selectable.replace_selectable()` *方法的* `Selectable`

用给定的`Alias`对象替换所有`FromClause` ‘old’的所有出现，返回此`FromClause`的副本。

自版本 1.4 起弃用：`Selectable.replace_selectable()`方法已弃用，并将在将来的版本中删除。 通过 sqlalchemy.sql.visitors 模块可以获得类似的功能。

```py
method scalar_subquery() → ScalarSelect[Any]
```

*继承自* `SelectBase.scalar_subquery()` *方���的* `SelectBase`

返回此可选择性的‘标量’表示，可用作列表达式。

返回的对象是`ScalarSelect`的一个实例。

通常，在其 columns 子句中只有一个列的 select 语句有资格用作标量表达式。 然后可以在封闭 SELECT 的 WHERE 子句或 columns 子句中使用标量子查询。

请注意，标量子查询与使用`SelectBase.subquery()`方法生成的 FROM 级子查询不同。

另请参阅

标量和相关子查询 - 在 2.0 教程中

```py
method select(*arg: Any, **kw: Any) → Select
```

*继承自* `SelectBase` *方法的* `SelectBase.select()`

自版本 1.4 起弃用：`SelectBase.select()`方法已弃用，并将在将来的版本中删除；此方法隐式创建一个应明确的子查询。请先调用`SelectBase.subquery()`以创建子查询，然后可以选择它。

```py
attribute selected_columns
```

一个`ColumnCollection`代表此 SELECT 语句或类似结构在其结果集中返回的列，不包括`TextClause`构造。

对于`CompoundSelect`，`CompoundSelect.selected_columns`属性返回一系列语句中第一个 SELECT 语句的选定列。

另请参阅

`Select.selected_columns`

版本 1.4 中新增。

```py
method self_group(against: OperatorType | None = None) → GroupedElement
```

对这个`ClauseElement`应用一个“分组”。

此方法被子类重写以返回一个“分组”构造，即括号。特别是它被“二进制”表达式使用时，当它们被放置到更大的表达式中时，提供了一个围绕自身的分组，以及当`select()`构造被放置到另一个`select()`的 FROM 子句中时。（请注意，子查询通常应使用`Select.alias()`方法创建，因为许多平台要求嵌套的 SELECT 语句需要命名）。

随着表达式的组合，`self_group()`的应用是自动的 - 最终用户代码不应直接使用此方法。请注意，SQLAlchemy 的子句构造考虑了操作符优先级 - 因此在表达式中可能不需要括号，例如`x OR (y AND z)` - AND 优先于 OR。

`ClauseElement`的基本`self_group()`方法只返回 self。

```py
method set_label_style(style: SelectLabelStyle) → CompoundSelect
```

返回具有指定标签样式的新可选择项。

有三种“标签样式”可用，`SelectLabelStyle.LABEL_STYLE_DISAMBIGUATE_ONLY`、`SelectLabelStyle.LABEL_STYLE_TABLENAME_PLUS_COL`和`SelectLabelStyle.LABEL_STYLE_NONE`。默认样式是`SelectLabelStyle.LABEL_STYLE_TABLENAME_PLUS_COL`。

在现代 SQLAlchemy 中，通常不需要更改标签样式，因为通过使用`ColumnElement.label()`方法更有效地使用逐表达式标签。在过去的版本中，`LABEL_STYLE_TABLENAME_PLUS_COL`用于消除来自不同表、别名或子查询的同名列的歧义；较新的`LABEL_STYLE_DISAMBIGUATE_ONLY`现在仅对与现有名称冲突的名称应用标签，因此此标签的影响最小。

消除歧义的理由主要是为了在创建子查询时从给定的`FromClause.c`集合中使所有列表达式可用。

从版本 1.4 开始：- `GenerativeSelect.set_label_style()`方法替换了以前的`.apply_labels()`、`.with_labels()`和`use_labels=True`方法和/或参数的组合。

另请参阅

`LABEL_STYLE_DISAMBIGUATE_ONLY`

`LABEL_STYLE_TABLENAME_PLUS_COL`

`LABEL_STYLE_NONE`

`LABEL_STYLE_DEFAULT`

```py
method slice(start: int, stop: int) → Self
```

*继承自* `GenerativeSelect.slice()` *方法的* `GenerativeSelect`

根据切片对此语句应用 LIMIT / OFFSET。

开始和停止索引的行为类似于 Python 内置`range()`函数的参数。此方法提供了一种替代方法，用于使用`LIMIT`/`OFFSET`获取查询的切片。

例如，

```py
stmt = select(User).order_by(User).id.slice(1, 3)
```

渲染为

```py
SELECT  users.id  AS  users_id,
  users.name  AS  users_name
FROM  users  ORDER  BY  users.id
LIMIT  ?  OFFSET  ?
(2,  1)
```

注意

`GenerativeSelect.slice()` 方法将替换应用的任何子句，该子句应用了 `GenerativeSelect.fetch()`。

版本 1.4 中的新功能：增加了从 ORM 泛化的 `GenerativeSelect.slice()` 方法。

另请参阅

`GenerativeSelect.limit()`

`GenerativeSelect.offset()`

`GenerativeSelect.fetch()`

```py
method subquery(name: str | None = None) → Subquery
```

*从* `SelectBase` *的* `SelectBase.subquery()` *方法继承*

返回此 `SelectBase` 的子查询。

从 SQL 角度看，子查询是一种括号括起来的、命名的构造，可以放置在另一个 SELECT 语句的 FROM 子句中。

假设有一个如下所示的 SELECT 语句：

```py
stmt = select(table.c.id, table.c.name)
```

上述语句可能如下所示：

```py
SELECT table.id, table.name FROM table
```

单独呈现子查询形式时，呈现方式相同，但是当嵌入到另一个 SELECT 语句的 FROM 子句中时，它变成了一个命名子元素：

```py
subq = stmt.subquery()
new_stmt = select(subq)
```

上述内容呈现为：

```py
SELECT anon_1.id, anon_1.name
FROM (SELECT table.id, table.name FROM table) AS anon_1
```

从历史上看，`SelectBase.subquery()` 等同于在 FROM 对象上调用 `FromClause.alias()` 方法；但是，由于 `SelectBase` 对象不是直接的 FROM 对象，因此 `SelectBase.subquery()` 方法提供了更清晰的语义。

版本 1.4 中的新功能。

```py
method with_for_update(*, nowait: bool = False, read: bool = False, of: _ForUpdateOfArgument | None = None, skip_locked: bool = False, key_share: bool = False) → Self
```

*从* `GenerativeSelect` *的* `GenerativeSelect.with_for_update()` *方法继承*

为此 `GenerativeSelect` 指定一个 `FOR UPDATE` 子句。

例如：

```py
stmt = select(table).with_for_update(nowait=True)
```

在像 PostgreSQL 或 Oracle 这样的数据库上，上述语句将呈现为如下语句：

```py
SELECT table.a, table.b FROM table FOR UPDATE NOWAIT
```

在其他后端，`nowait` 选项将被忽略，而会产生以下输出：

```py
SELECT table.a, table.b FROM table FOR UPDATE
```

当不带参数调用时，语句将带有后缀`FOR UPDATE`。然后可以提供额外的参数，允许使用常见的特定于数据库的变体。

参数：

+   `nowait` – 布尔值；在 Oracle 和 PostgreSQL 方言上会渲染`FOR UPDATE NOWAIT`。

+   `read` – 布尔值；在 MySQL 上会渲染`LOCK IN SHARE MODE`，在 PostgreSQL 上会渲染`FOR SHARE`。在 PostgreSQL 上，与`nowait`组合时，会渲染`FOR SHARE NOWAIT`。

+   `of` – SQL 表达式或 SQL 表达式元素列表，（通常是`Column`对象或兼容表达式，对于某些后端也可以是表达式）将渲染为`FOR UPDATE OF`子句；受 PostgreSQL、Oracle、某些 MySQL 版本和可能其他后端支持。根据后端的不同，可能会渲染为表或列。

+   `skip_locked` – 布尔值，在 Oracle 和 PostgreSQL 方言上会渲染`FOR UPDATE SKIP LOCKED`，如果还指定了`read=True`，则会渲染`FOR SHARE SKIP LOCKED`。

+   `key_share` – 布尔值，会渲染`FOR NO KEY UPDATE`，或者如果与`read=True`组合，会在 PostgreSQL 方言上渲染`FOR KEY SHARE`。

```py
class sqlalchemy.sql.expression.CTE
```

表示一个公共表达式。

`CTE`对象是通过任何 SELECT 语句的`SelectBase.cte()`方法获得的。较少见的语法还允许在 DML 构造上使用`HasCTE.cte()`方法，例如`Insert`、`Update`和`Delete`。查看`HasCTE.cte()`方法以获取有关 CTE 的用法详细信息。

参见

子查询和 CTE - 在 2.0 教程中

`HasCTE.cte()` - 调用样式示例

**成员**

alias(), union(), union_all()

**类签名**

类`sqlalchemy.sql.expression.CTE` (`sqlalchemy.sql.roles.DMLTableRole`, `sqlalchemy.sql.roles.IsCTERole`, `sqlalchemy.sql.expression.Generative`, `sqlalchemy.sql.expression.HasPrefixes`, `sqlalchemy.sql.expression.HasSuffixes`, `sqlalchemy.sql.expression.AliasedReturnsRows`)

```py
method alias(name: str | None = None, flat: bool = False) → CTE
```

返回此`CTE`的`Alias`。

此方法是`FromClause.alias()`方法的 CTE 特定专业化。

另请参阅

使用别名

`alias()`

```py
method union(*other: _SelectStatementForCompoundArgument) → CTE
```

返回一个新的`CTE`，其中包含原始 CTE 与提供的可选参数作为位置参数的 SQL `UNION`。

参数：

***其他** –

一个或多个用于创建 UNION 的元素。

从版本 1.4.28 更改：现在接受多个元素。

另请参阅

`HasCTE.cte()` - 调用样式示例

```py
method union_all(*other: _SelectStatementForCompoundArgument) → CTE
```

返回一个新的`CTE`，其中包含原始 CTE 与提供的可选参数作为位置参数的 SQL `UNION ALL`。

参数：

***其他** –

一个或多个用于创建 UNION 的元素。

从版本 1.4.28 更改：现在接受多个元素。

另请参阅

`HasCTE.cte()` - 调用样式示例

```py
class sqlalchemy.sql.expression.Executable
```

将一个`ClauseElement`标记为支持执行。

`Executable`是所有“语句”类型对象的超类，包括`select()`、`delete()`、`update()`、`insert()`、`text()`。

**成员**

execution_options(), get_execution_options(), options()

**类签名**

类`sqlalchemy.sql.expression.Executable` (`sqlalchemy.sql.roles.StatementRole`)

```py
method execution_options(**kw: Any) → Self
```

为语句设置在执行期间生效的非 SQL 选项。

可以在多个范围设置执行选项，包括每个语句、每个连接或每次执行，使用诸如`Connection.execution_options()`和接受选项字典的参数的方法，如`Connection.execute.execution_options`和`Session.execute.execution_options`。

执行选项的主要特征与其他类型的选项（如 ORM 加载器选项）不同，**执行选项永远不会影响查询的编译 SQL，只会影响 SQL 语句本身的调用方式或结果的获取方式**。也就是说，执行选项不是 SQL 编译所考虑的内容，也不被视为语句的缓存状态的一部分。

`Executable.execution_options()`方法是生成的，就像应用于`Engine`和`Query`对象的方法一样，这意味着当调用该方法时，会返回对象的副本，将给定的参数应用于该新副本，但原始对象保持不变：

```py
statement = select(table.c.x, table.c.y)
new_statement = statement.execution_options(my_option=True)
```

一个例外是`Connection`对象，其中`Connection.execution_options()`方法明确地**不**是生成的。

可以传递给`Executable.execution_options()`和其他相关方法和参数字典的选项类型包括被 SQLAlchemy Core 或 ORM 明确消耗的参数，以及 SQLAlchemy 未定义的任意关键字参数，这意味着这些方法和/或参数字典可用于与自定义代码交互的用户定义参数，可以使用诸如`Executable.get_execution_options()`和`Connection.get_execution_options()`等方法访问参数，或者在选定的事件钩子中使用专用的`execution_options`事件参数，例如`ConnectionEvents.before_execute.execution_options`或`ORMExecuteState.execution_options`，例如：

```py
from sqlalchemy import event

@event.listens_for(some_engine, "before_execute")
def _process_opt(conn, statement, multiparams, params, execution_options):
    "run a SQL function before invoking a statement"

    if execution_options.get("do_special_thing", False):
        conn.exec_driver_sql("run_special_function()")
```

在 SQLAlchemy 明确识别的选项范围内，大多数适用于特定类别的对象而不是其他对象。最常见的执行选项包括：

+   `Connection.execution_options.isolation_level` - 通过`Engine`为连接或一类连接设置隔离级别。此选项仅被`Connection`或`Engine`接受。

+   `Connection.execution_options.stream_results` - 表示应使用服务器端游标获取结果；此选项被`Connection`，`Connection.execute()`上的`Connection.execute.execution_options`参数以及 SQL 语句对象上的`Executable.execution_options()`以及 ORM 构造如`Session.execute()`所接受。

+   `Connection.execution_options.compiled_cache` - 表示将作为`Connection`或`Engine`的 SQL 编译缓存的字典，以及`Session.execute()`等 ORM 方法的缓存。 可以传递为`None`以禁用语句的缓存。 此选项不被`Executable.execution_options()`接受，因为在语句对象中携带编译缓存是不明智的。

+   `Connection.execution_options.schema_translate_map` - 用于模式转换映射功能的模式名称映射，被`Connection`，`Engine`，`Executable`接受，以及 ORM 构造如`Session.execute()`。

参见

`Connection.execution_options()`

`Connection.execute.execution_options`

`Session.execute.execution_options`

ORM 执行选项 - 所有 ORM 特定执行选项的文档

```py
method get_execution_options() → _ExecuteOptions
```

获取在执行期间生效的非 SQL 选项。

版本 1.3 中的新功能。

另请参阅

`Executable.execution_options()`

```py
method options(*options: ExecutableOption) → Self
```

将选项应用于此语句。

一般来说，选项是任何可以被 SQL 编译器解释为语句的 Python 对象。这些选项可以被特定的方言或特定类型的编译器消耗。

最常见的选项类型是应用“急加载”和其他加载行为到 ORM 查询的 ORM 级选项。然而，选项理论上可以用于许多其他目的。

有关特定类型语句的特定类型选项的背景，请参阅这些选项对象的文档。

版本 1.4 中的更改：- 向核心语句对象添加了 `Executable.options()`，以实现统一的核心 / ORM 查询功能。

另请参阅

列加载选项 - 指特定于 ORM 查询使用的选项

使用加载器选项加载关系 - 指特定于 ORM 查询使用的选项

```py
class sqlalchemy.sql.expression.Exists
```

表示一个 `EXISTS` 子句。

请参阅 `exists()` 以获取用法描述。

通过调用 `SelectBase.exists()` 可以从 `select()` 实例构建 `EXISTS` 子句。

**成员**

correlate(), correlate_except(), inherit_cache, select(), select_from(), where()

**类签名**

类 `sqlalchemy.sql.expression.Exists` (`sqlalchemy.sql.expression.UnaryExpression`)

```py
method correlate(*fromclauses: Literal[None, False] | _FromClauseArgument) → Self
```

将相关性应用于由此 `Exists` 指示的子查询。

另请参阅

`ScalarSelect.correlate()`

```py
method correlate_except(*fromclauses: Literal[None, False] | _FromClauseArgument) → Self
```

将关联应用于由此 `Exists` 指示的子查询。

参见

`ScalarSelect.correlate_except()`

```py
attribute inherit_cache: bool | None = True
```

指示此 `HasCacheKey` 实例是否应该使用其直接超类使用的缓存键生成方案。

该属性默认为 `None`，表示结构尚未考虑是否适合参与缓存；这在功能上等同于将值设置为 `False`，只是还会发出警告。

如果与对象对应的 SQL 不会基于仅属于此类而不属于其超类的属性更改，则可以将此标志设置为 `True`。

参见

为自定义结构启用缓存支持 - 为第三方或用户定义的 SQL 结构设置 `HasCacheKey.inherit_cache` 属性的通用指南。

```py
method select() → Select
```

返回此 `Exists` 的 SELECT。

例如：

```py
stmt = exists(some_table.c.id).where(some_table.c.id == 5).select()
```

这将生成一个类似于的语句：

```py
SELECT EXISTS (SELECT id FROM some_table WHERE some_table = :param) AS anon_1
```

参见

`select()` - 允许任意列列表的通用方法。

```py
method select_from(*froms: _FromClauseArgument) → Self
```

返回一个新的 `Exists` 构造，将给定表达式应用于包含的 select 语句的 `Select.select_from()` 方法。

注意

通常最好首先构建一个 `Select` 语句，包括所需的 WHERE 子句，然后一次使用 `SelectBase.exists()` 方法生成一个 `Exists` 对象。

```py
method where(*clause: _ColumnExpressionArgument[bool]) → Self
```

返回一个新的带有给定表达式添加到其 WHERE 子句中的 `exists()` 构造，如果有的话，通过 AND 连接到现有子句。

注意

通常最好先构建一个包含所需 WHERE 子句的 `Select` 语句，然后立即使用 `SelectBase.exists()` 方法生成一个 `Exists` 对象。

```py
class sqlalchemy.sql.expression.FromClause
```

表示可以在 `SELECT` 语句的 `FROM` 子句中使用的元素。

最常见的 `FromClause` 形式是 `Table` 和 `select()` 构造。所有 `FromClause` 对象的共同特征包括：

+   一个 `c` 集合，提供对一组 `ColumnElement` 对象的按名称访问。

+   一个 `primary_key` 属性，其中包含所有指示 `primary_key` 标志的 `ColumnElement` 对象的集合。

+   生成各种“from”子句的方法，包括 `FromClause.alias()`、`FromClause.join()`、`FromClause.select()`。

**成员**

alias(), c, columns, description, entity_namespace, exported_columns, foreign_keys, is_derived_from(), join(), outerjoin(), primary_key, schema, select(), tablesample()

**类签名**

类`sqlalchemy.sql.expression.FromClause`（`sqlalchemy.sql.roles.AnonymizedFromClauseRole`, `sqlalchemy.sql.expression.Selectable`）

```py
method alias(name: str | None = None, flat: bool = False) → NamedFromClause
```

返回此`FromClause`的别名。

例如：

```py
a2 = some_table.alias('a2')
```

上述代码创建了一个`Alias`对象，可以在任何 SELECT 语句中作为 FROM 子句使用。

另请参阅

使用别名

`alias()`

```py
attribute c
```

`FromClause.columns`的同义词

返回：

一个`ColumnCollection`。

```py
attribute columns
```

此`FromClause`维护的`ColumnElement`对象的基于名称的集合。

`columns`或`c`集合是使用与表绑定或其他可选择绑定的列构造 SQL 表达式的入口：

```py
select(mytable).where(mytable.c.somecolumn == 5)
```

返回：

一个`ColumnCollection`对象。

```py
attribute description
```

此`FromClause`的简要描述。

主要用于错误消息格式化。

```py
attribute entity_namespace
```

返回一个用于在 SQL 表达式中基于名称访问的命名空间。

这是用于解析“filter_by()”类型表达式的命名空间，例如：

```py
stmt.filter_by(address='some address')
```

默认为`.c`集合，但在内部可以使用“entity_namespace”注释进行覆盖以提供替代结果。

```py
attribute exported_columns
```

一个`ColumnCollection`，表示此`Selectable`的“导出”列。

`FromClause`对象的“导出”列与`FromClause.columns`集合是同义词。

新版本中新增。

另请参阅

`Selectable.exported_columns`

`SelectBase.exported_columns`

```py
attribute foreign_keys
```

返回此 `FromClause` 引用的 `ForeignKey` 标记对象的集合。

每个 `ForeignKey` 都是一个属于 `Table`-范围的 `ForeignKeyConstraint` 的成员。

另请参见

`Table.foreign_key_constraints`

```py
method is_derived_from(fromclause: FromClause | None) → bool
```

如果此 `FromClause` 是从给定的 `FromClause` 衍生出来的，则返回 `True`。

一个示例是一个表的别名是从该表派生的。

```py
method join(right: _FromClauseArgument, onclause: _ColumnExpressionArgument[bool] | None = None, isouter: bool = False, full: bool = False) → Join
```

从此 `FromClause` 返回一个 `Join` 到另一个 `FromClause`。

例如：

```py
from sqlalchemy import join

j = user_table.join(address_table,
                user_table.c.id == address_table.c.user_id)
stmt = select(user_table).select_from(j)
```

将会发出类似以下的 SQL：

```py
SELECT user.id, user.name FROM user
JOIN address ON user.id = address.user_id
```

参数：

+   `right` – 连接的右侧；这是任何 `FromClause` 对象，如 `Table` 对象，也可以是一个可选择兼容对象，如 ORM 映射的类。

+   `onclause` – 表示连接的 ON 子句的 SQL 表达式。如果保留为 `None`，则 `FromClause.join()` 将尝试根据外键关系连接两个表。

+   `isouter` – 如果为 True，则渲染一个左外连接，而不是连接。

+   `full` – 如果为 True，则渲染一个完整的外连接，而不是左外连接。意味着 `FromClause.join.isouter`。

另请参见

`join()` - 独立函数

`Join` - 生成的对象类型

```py
method outerjoin(right: _FromClauseArgument, onclause: _ColumnExpressionArgument[bool] | None = None, full: bool = False) → Join
```

从此 `FromClause` 返回一个带有 "isouter" 标志设置为 True 的 `Join` 到另一个 `FromClause`。

例如：

```py
from sqlalchemy import outerjoin

j = user_table.outerjoin(address_table,
                user_table.c.id == address_table.c.user_id)
```

以上等同于：

```py
j = user_table.join(
    address_table,
    user_table.c.id == address_table.c.user_id,
    isouter=True)
```

参数：

+   `right` – 连接的右侧；这是任何 `FromClause` 对象，例如 `Table` 对象，也可以是 ORM 映射的类等可选兼容对象。

+   `onclause` – 代表连接的 ON 子句的 SQL 表达式。如果留空，`FromClause.join()` 将尝试根据外键关系连接两个表。

+   `full` – 如果为 True，则渲染 FULL OUTER JOIN，而不是 LEFT OUTER JOIN。

另见

`FromClause.join()`

`Join`

```py
attribute primary_key
```

返回这个 `_selectable.FromClause` 的主键所组成的可迭代列 `Column` 对象集合。

对于 `Table` 对象，这个集合由 `PrimaryKeyConstraint` 表示，它本身是一个可迭代的 `Column` 对象集合。

```py
attribute schema: str | None = None
```

为这个 `FromClause` 定义 ‘schema’ 属性。

对于大多数对象而言，这通常为 `None`，除了 `Table` 的情况，其中它被视为 `Table.schema` 参数的值。

```py
method select() → Select
```

返回这个 `FromClause` 的 SELECT。

例如：

```py
stmt = some_table.select().where(some_table.c.id == 5)
```

另见

`select()` - 允许任意列列表的通用方法。

```py
method tablesample(sampling: float | Function[Any], name: str | None = None, seed: roles.ExpressionElementRole[Any] | None = None) → TableSample
```

返回这个 `FromClause` 的 TABLESAMPLE 别名。

返回值也是顶层 `tablesample()` 函数提供的 `TableSample` 构造。

另见

`tablesample()` - 使用指南和参数

```py
class sqlalchemy.sql.expression.GenerativeSelect
```

可以添加额外元素的 SELECT 语句的基类。

这用作`Select`和`CompoundSelect`的基础，其中可以添加诸如 ORDER BY、GROUP BY 之类的元素，并且可以控制列的呈现。与`TextualSelect`相比，虽然它是`SelectBase`的子类，并且也是一个 SELECT 构造，但它代表的是一个固定的文本字符串，无法在这个级别上更改，只能包装为子查询。

**成员**

fetch(), get_label_style(), group_by(), limit(), offset(), order_by(), set_label_style(), slice(), with_for_update()

**类签名**

class `sqlalchemy.sql.expression.GenerativeSelect` (`sqlalchemy.sql.expression.SelectBase`, `sqlalchemy.sql.expression.Generative`)

```py
method fetch(count: _LimitOffsetType, with_ties: bool = False, percent: bool = False) → Self
```

使用给定的 FETCH FIRST 标准返回一个新的可选择的。

这是一个数值，通常在结果选择中呈现为`FETCH {FIRST | NEXT} [ count ] {ROW | ROWS} {ONLY | WITH TIES}`表达式。此功能目前已为 Oracle、PostgreSQL、MSSQL 实现。

使用`GenerativeSelect.offset()`来指定偏移量。

注意

`GenerativeSelect.fetch()`方法将替换任何应用于`GenerativeSelect.limit()`的子句。

1.4 版中的新功能。

参数：

+   `count` – 一个整数 COUNT 参数，或者提供整数结果的 SQL 表达式。当`percent=True`时，这将表示要返回的行的百分比，而不是绝对值。传递`None`来重置它。

+   `with_ties` – 当为`True`时，使用 WITH TIES 选项来返回任何根据`ORDER BY`子句在结果集中处于最后位置的附加行。在这种情况下，`ORDER BY`可能是强制性的。默认为`False`

+   `percent` – 当`True`时，`count`表示要返回的所选行总数的百分比。默认为`False`。

另请参阅

`GenerativeSelect.limit()`

`GenerativeSelect.offset()`

```py
method get_label_style() → SelectLabelStyle
```

检索当前的标签样式。

1.4 版本中的新内容。

```py
method group_by(_GenerativeSelect__first: Literal[None, _NoArg.NO_ARG] | _ColumnExpressionOrStrLabelArgument[Any] = _NoArg.NO_ARG, *clauses: _ColumnExpressionOrStrLabelArgument[Any]) → Self
```

返回具有给定的 GROUP BY 条件列表的新可选择对象。

通过传递`None`可以抑制所有现有的 GROUP BY 设置。

例如：

```py
stmt = select(table.c.name, func.max(table.c.stat)).\
group_by(table.c.name)
```

参数：

***clauses** – 一系列将用于生成 GROUP BY 子句的`ColumnElement`构造。

另请参阅

带有 GROUP BY / HAVING 的聚合函数 - 在 SQLAlchemy 统一教程中

按标签排序或分组 - 在 SQLAlchemy 统一教程中

```py
method limit(limit: _LimitOffsetType) → Self
```

返回具有给定 LIMIT 条件的新可选择对象。

这是一个数值，通常在结果选择中呈现为`LIMIT`表达式。不支持`LIMIT`的后端将尝试提供类似的功能。

注意

`GenerativeSelect.limit()`方法将替换应用的任何子句与`GenerativeSelect.fetch()`。

参数：

**limit** – 一个整数 LIMIT 参数，或者提供整数结果的 SQL 表达式。传递`None`来重置它。

另请参阅

`GenerativeSelect.fetch()`

`GenerativeSelect.offset()`

```py
method offset(offset: _LimitOffsetType) → Self
```

返回具有给定 OFFSET 条件的新可选择对象。

这是一个数值，通常在结果选择中呈现为`OFFSET`表达式。不支持`OFFSET`的后端将尝试提供类似的功能。

参数：

**offset** – 一个整数 OFFSET 参数，或者提供整数结果的 SQL 表达式。传递`None`来重置它。

另请参阅

`GenerativeSelect.limit()`

`GenerativeSelect.fetch()`

```py
method order_by(_GenerativeSelect__first: Literal[None, _NoArg.NO_ARG] | _ColumnExpressionOrStrLabelArgument[Any] = _NoArg.NO_ARG, *clauses: _ColumnExpressionOrStrLabelArgument[Any]) → Self
```

返回具有给定的 ORDER BY 条件列表的新可选择对象。

例如：

```py
stmt = select(table).order_by(table.c.id, table.c.name)
```

多次调用此方法等效于将所有子句连接一次。所有现有的 ORDER BY 条件都可以通过单独传递 `None` 来取消。然后，可以通过再次调用 `Query.order_by()` 来添加新的 ORDER BY 条件，例如：

```py
# will erase all ORDER BY and ORDER BY new_col alone
stmt = stmt.order_by(None).order_by(new_col)
```

参数:

***clauses** – 一系列用于生成 ORDER BY 子句的 `ColumnElement` 构造。

请参见

ORDER BY - 在 SQLAlchemy 统一教程中

按标签排序或分组 - 在 SQLAlchemy 统一教程中

```py
method set_label_style(style: SelectLabelStyle) → Self
```

返回具有指定标签样式的新可选择项。

有三种“标签样式”可用，`SelectLabelStyle.LABEL_STYLE_DISAMBIGUATE_ONLY`、`SelectLabelStyle.LABEL_STYLE_TABLENAME_PLUS_COL` 和 `SelectLabelStyle.LABEL_STYLE_NONE`。默认样式是 `SelectLabelStyle.LABEL_STYLE_TABLENAME_PLUS_COL`。

在现代 SQLAlchemy 中，通常不需要更改标签样式，因为通过使用 `ColumnElement.label()` 方法更有效地使用每个表达式的标签。在以前的版本中，`LABEL_STYLE_TABLENAME_PLUS_COL` 用于消除来自不同表、别名或子查询的相同命名列的歧义; 新的 `LABEL_STYLE_DISAMBIGUATE_ONLY` 现在仅将标签应用于与现有名称冲突的名称，以使此标签的影响最小化。

消除歧义的原因主要是在创建子查询时，所有列表达式都可以从给定的 `FromClause.c` 集合中使用。

从版本 1.4 开始: - `GenerativeSelect.set_label_style()` 方法取代了先前的 `.apply_labels()`、`.with_labels()` 和 `use_labels=True` 方法和/或参数的组合。

请参见

`LABEL_STYLE_DISAMBIGUATE_ONLY`

`LABEL_STYLE_TABLENAME_PLUS_COL`

`LABEL_STYLE_NONE`

`LABEL_STYLE_DEFAULT`

```py
method slice(start: int, stop: int) → Self
```

根据一个切片对该语句应用 LIMIT / OFFSET。

起始和停止索引的行为类似于 Python 内置的 `range()` 函数的参数。此方法提供了一个替代方案，用于使用 `LIMIT`/`OFFSET` 获取查询的一个切片。

例如，

```py
stmt = select(User).order_by(User).id.slice(1, 3)
```

渲染为

```py
SELECT  users.id  AS  users_id,
  users.name  AS  users_name
FROM  users  ORDER  BY  users.id
LIMIT  ?  OFFSET  ?
(2,  1)
```

注意

`GenerativeSelect.slice()` 方法将替换任何使用 `GenerativeSelect.fetch()` 应用的子句。

自 1.4 版本新增：从 ORM 推广出的 `GenerativeSelect.slice()` 方法。

另请参见

`GenerativeSelect.limit()`

`GenerativeSelect.offset()`

`GenerativeSelect.fetch()`

```py
method with_for_update(*, nowait: bool = False, read: bool = False, of: _ForUpdateOfArgument | None = None, skip_locked: bool = False, key_share: bool = False) → Self
```

为此 `GenerativeSelect` 指定一个 `FOR UPDATE` 子句。

例如：

```py
stmt = select(table).with_for_update(nowait=True)
```

在像 PostgreSQL 或 Oracle 这样的数据库上，上述内容将渲染为类似于以下语句：

```py
SELECT table.a, table.b FROM table FOR UPDATE NOWAIT
```

在其他后端，`nowait` 选项将被忽略，并且会产生：

```py
SELECT table.a, table.b FROM table FOR UPDATE
```

在不带参数调用时，该语句将以后缀 `FOR UPDATE` 渲染。然后可以提供其他参数，以允许常见的数据库特定变体。

参数：

+   `nowait` – 布尔值；将在 Oracle 和 PostgreSQL 方言上渲染为 `FOR UPDATE NOWAIT`。

+   `read` – 布尔值；将在 MySQL 上渲染为 `LOCK IN SHARE MODE`，在 PostgreSQL 上渲染为 `FOR SHARE`。在 PostgreSQL 上，当与 `nowait` 结合使用时，将渲染为 `FOR SHARE NOWAIT`。

+   `of` – SQL 表达式或 SQL 表达式元素列表（通常是 `Column` 对象或兼容的表达式，对于某些后端，也可以是表达式）；它将被渲染为一个 `FOR UPDATE OF` 子句；受 PostgreSQL、Oracle、一些 MySQL 版本和可能其他一些后端的支持。根据后端的不同，可能会渲染为表或列。

+   `skip_locked` – 布尔值，将在 Oracle 和 PostgreSQL 方言上渲染为 `FOR UPDATE SKIP LOCKED`，如果还指定了 `read=True`，则会渲染为 `FOR SHARE SKIP LOCKED`。

+   `key_share` – 布尔值，将在 PostgreSQL 方言上渲染为 `FOR NO KEY UPDATE`，或者如果与 `read=True` 结合使用，则会渲染为 `FOR KEY SHARE`。

```py
class sqlalchemy.sql.expression.HasCTE
```

声明一个包含 CTE 支持的类的 Mixin。

**成员**

add_cte(), cte()

**类签名**

类`sqlalchemy.sql.expression.HasCTE`（`sqlalchemy.sql.roles.HasCTERole`，`sqlalchemy.sql.expression.SelectsRows`）

```py
method add_cte(*ctes: CTE, nest_here: bool = False) → Self
```

向该语句添加一个或多个`CTE` 构造。

此方法将使给定的`CTE` 构造与父语句相关联，以便它们将分别无条件地在最终语句的 WITH 子句中渲染，即使在语句或任何子选择中未在其他地方引用。 

当可选参数`HasCTE.add_cte.nest_here` 设置为 True 时，每个给定的`CTE` 将直接在此语句中渲染为 WITH 子句，而不会像将该语句渲染为更大语句中的子查询时被移动到最终渲染语句的顶部。

此方法有两个一般用途。一个是嵌入一些没有明确引用的 CTE 语句，例如将 DML 语句（例如 INSERT 或 UPDATE）作为 CTE 内联到可能间接从其结果中提取的主语句中的用例。另一个是提供对应该保持直接渲染为特定语句的特定 CTE 构造的精确放置的控制，该语句可能嵌套在较大的语句中。

例如：

```py
from sqlalchemy import table, column, select
t = table('t', column('c1'), column('c2'))

ins = t.insert().values({"c1": "x", "c2": "y"}).cte()

stmt = select(t).add_cte(ins)
```

将渲染为：

```py
WITH anon_1 AS
(INSERT INTO t (c1, c2) VALUES (:param_1, :param_2))
SELECT t.c1, t.c2
FROM t
```

上述“anon_1”CTE 在 SELECT 语句中没有被引用，但仍然完成了运行 INSERT 语句的任务。

在与 DML 相关的上下文中，使用 PostgreSQL `Insert` 构造生成“upsert”：

```py
from sqlalchemy import table, column
from sqlalchemy.dialects.postgresql import insert

t = table("t", column("c1"), column("c2"))

delete_statement_cte = (
    t.delete().where(t.c.c1 < 1).cte("deletions")
)

insert_stmt = insert(t).values({"c1": 1, "c2": 2})
update_statement = insert_stmt.on_conflict_do_update(
    index_elements=[t.c.c1],
    set_={
        "c1": insert_stmt.excluded.c1,
        "c2": insert_stmt.excluded.c2,
    },
).add_cte(delete_statement_cte)

print(update_statement)
```

上述语句渲染为：

```py
WITH deletions AS
(DELETE FROM t WHERE t.c1 < %(c1_1)s)
INSERT INTO t (c1, c2) VALUES (%(c1)s, %(c2)s)
ON CONFLICT (c1) DO UPDATE SET c1 = excluded.c1, c2 = excluded.c2
```

从版本 1.4.21 开始新增。

参数：

+   `*ctes` –

    零个或多个`CTE` 构造。

    从版本 2.0 开始更改：接受多个 CTE 实例

+   `nest_here` –

    如果为 True，则给定的 CTE 或 CTE 将被渲染为如果它们在添加到此`HasCTE`时指定了`HasCTE.cte.nesting`标志为 True。假设给定的 CTE 也没有在外部包含语句中被引用，则给定的 CTE 在给定此标志时应在此语句级别渲染。

    从版本 2.0 开始新增。

    另请参阅

    `HasCTE.cte.nesting`

```py
method cte(name: str | None = None, recursive: bool = False, nesting: bool = False) → CTE
```

返回一个新的`CTE`，或者通用表达式实例。

公共表达式是 SQL 标准的一部分，其中 SELECT 语句可以在主语句之后指定的辅助语句中进行绘制，使用名为“WITH”的子句。还可以使用关于 UNION 的特殊语义来允许“递归”查询，其中 SELECT 语句可以绘制出先前已选择的行集。

在一些数据库中，CTE 也可以应用于 DML 构造 UPDATE、INSERT 和 DELETE，既可以作为 CTE 行的来源（与 RETURNING 结合使用），也可以作为 CTE 行的消费者。

SQLAlchemy 检测到`CTE`对象，这些对象与`Alias`对象类似，被视为要传递到语句的 FROM 子句以及语句顶部的 WITH 子句中的特殊元素。

对于诸如 PostgreSQL“MATERIALIZED”和“NOT MATERIALIZED”之类的特殊前缀，可以使用`CTE.prefix_with()`方法来建立这些前缀。

版本 1.3.13 中的更改：增加了对前缀的支持。特别是- MATERIALIZED 和 NOT MATERIALIZED。

参数：

+   `name` – 给公共表达式的名称。与`FromClause.alias()`类似，名称可以保持为`None`，在这种情况下，将在查询编译时使用匿名符号。

+   `recursive` – 如果为`True`，则呈现`WITH RECURSIVE`。递归公共表达式旨在与 UNION ALL 一起使用，以从已经选择的行中派生行。

+   `nesting` –

    如果为`True`，则将 CTE 本地呈现到引用它的语句中。对于更复杂的情况，还可以使用`HasCTE.add_cte()`方法，使用`HasCTE.add_cte.nest_here`参数来更精细地控制特定 CTE 的确切放置位置。

    1.4.24 版本中的新增内容。

    另请参阅

    `HasCTE.add_cte()`

以下示例包括两个来自 PostgreSQL 文档的示例，网址为[`www.postgresql.org/docs/current/static/queries-with.html`](https://www.postgresql.org/docs/current/static/queries-with.html)，以及额外的示例。

示例 1，非递归：

```py
from sqlalchemy import (Table, Column, String, Integer,
                        MetaData, select, func)

metadata = MetaData()

orders = Table('orders', metadata,
    Column('region', String),
    Column('amount', Integer),
    Column('product', String),
    Column('quantity', Integer)
)

regional_sales = select(
                    orders.c.region,
                    func.sum(orders.c.amount).label('total_sales')
                ).group_by(orders.c.region).cte("regional_sales")

top_regions = select(regional_sales.c.region).\
        where(
            regional_sales.c.total_sales >
            select(
                func.sum(regional_sales.c.total_sales) / 10
            )
        ).cte("top_regions")

statement = select(
            orders.c.region,
            orders.c.product,
            func.sum(orders.c.quantity).label("product_units"),
            func.sum(orders.c.amount).label("product_sales")
    ).where(orders.c.region.in_(
        select(top_regions.c.region)
    )).group_by(orders.c.region, orders.c.product)

result = conn.execute(statement).fetchall()
```

示例 2，WITH RECURSIVE：

```py
from sqlalchemy import (Table, Column, String, Integer,
                        MetaData, select, func)

metadata = MetaData()

parts = Table('parts', metadata,
    Column('part', String),
    Column('sub_part', String),
    Column('quantity', Integer),
)

included_parts = select(\
    parts.c.sub_part, parts.c.part, parts.c.quantity\
    ).\
    where(parts.c.part=='our part').\
    cte(recursive=True)

incl_alias = included_parts.alias()
parts_alias = parts.alias()
included_parts = included_parts.union_all(
    select(
        parts_alias.c.sub_part,
        parts_alias.c.part,
        parts_alias.c.quantity
    ).\
    where(parts_alias.c.part==incl_alias.c.sub_part)
)

statement = select(
            included_parts.c.sub_part,
            func.sum(included_parts.c.quantity).
              label('total_quantity')
        ).\
        group_by(included_parts.c.sub_part)

result = conn.execute(statement).fetchall()
```

示例 3，使用 CTE 的 UPDATE 和 INSERT 的 upsert：

```py
from datetime import date
from sqlalchemy import (MetaData, Table, Column, Integer,
                        Date, select, literal, and_, exists)

metadata = MetaData()

visitors = Table('visitors', metadata,
    Column('product_id', Integer, primary_key=True),
    Column('date', Date, primary_key=True),
    Column('count', Integer),
)

# add 5 visitors for the product_id == 1
product_id = 1
day = date.today()
count = 5

update_cte = (
    visitors.update()
    .where(and_(visitors.c.product_id == product_id,
                visitors.c.date == day))
    .values(count=visitors.c.count + count)
    .returning(literal(1))
    .cte('update_cte')
)

upsert = visitors.insert().from_select(
    [visitors.c.product_id, visitors.c.date, visitors.c.count],
    select(literal(product_id), literal(day), literal(count))
        .where(~exists(update_cte.select()))
)

connection.execute(upsert)
```

示例 4，嵌套 CTE（SQLAlchemy 1.4.24 及以上版本）：

```py
value_a = select(
    literal("root").label("n")
).cte("value_a")

# A nested CTE with the same name as the root one
value_a_nested = select(
    literal("nesting").label("n")
).cte("value_a", nesting=True)

# Nesting CTEs takes ascendency locally
# over the CTEs at a higher level
value_b = select(value_a_nested.c.n).cte("value_b")

value_ab = select(value_a.c.n.label("a"), value_b.c.n.label("b"))
```

上述查询将第二个 CTE 嵌套在第一个内部，并显示为以下内联参数：

```py
WITH
    value_a AS
        (SELECT 'root' AS n),
    value_b AS
        (WITH value_a AS
            (SELECT 'nesting' AS n)
        SELECT value_a.n AS n FROM value_a)
SELECT value_a.n AS a, value_b.n AS b
FROM value_a, value_b
```

可以使用`HasCTE.add_cte()`方法设置相同的 CTE，如下所示（SQLAlchemy 2.0 及以上版本）：

```py
value_a = select(
    literal("root").label("n")
).cte("value_a")

# A nested CTE with the same name as the root one
value_a_nested = select(
    literal("nesting").label("n")
).cte("value_a")

# Nesting CTEs takes ascendency locally
# over the CTEs at a higher level
value_b = (
    select(value_a_nested.c.n).
    add_cte(value_a_nested, nest_here=True).
    cte("value_b")
)

value_ab = select(value_a.c.n.label("a"), value_b.c.n.label("b"))
```

示例 5，非线性 CTE（SQLAlchemy 1.4.28 及以上版本）：

```py
edge = Table(
    "edge",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("left", Integer),
    Column("right", Integer),
)

root_node = select(literal(1).label("node")).cte(
    "nodes", recursive=True
)

left_edge = select(edge.c.left).join(
    root_node, edge.c.right == root_node.c.node
)
right_edge = select(edge.c.right).join(
    root_node, edge.c.left == root_node.c.node
)

subgraph_cte = root_node.union(left_edge, right_edge)

subgraph = select(subgraph_cte)
```

上述查询将在递归 CTE 内呈现 2 个 UNION：

```py
WITH RECURSIVE nodes(node) AS (
        SELECT 1 AS node
    UNION
        SELECT edge."left" AS "left"
        FROM edge JOIN nodes ON edge."right" = nodes.node
    UNION
        SELECT edge."right" AS "right"
        FROM edge JOIN nodes ON edge."left" = nodes.node
)
SELECT nodes.node FROM nodes
```

另见

`Query.cte()` - `HasCTE.cte()` 的 ORM 版本。

```py
class sqlalchemy.sql.expression.HasPrefixes
```

**成员**

prefix_with()

```py
method prefix_with(*prefixes: _TextCoercedExpressionArgument[Any], dialect: str = '*') → Self
```

在语句关键字后添加一个或多个表达式，即 SELECT、INSERT、UPDATE 或 DELETE。生成式。

用于支持像 MySQL 提供的特定于后端的前缀关键字。

例如：

```py
stmt = table.insert().prefix_with("LOW_PRIORITY", dialect="mysql")

# MySQL 5.7 optimizer hints
stmt = select(table).prefix_with(
    "/*+ BKA(t1) */", dialect="mysql")
```

可以通过多次调用 `HasPrefixes.prefix_with()` 来指定多个前缀。

参数：

+   `*prefixes` – 将在 INSERT、UPDATE 或 DELETE 关键字之后呈现的文本或 `ClauseElement` 构造。

+   `dialect` – 可选的字符串方言名称，将此前缀的呈现限制为仅限于该方言。

```py
class sqlalchemy.sql.expression.HasSuffixes
```

**成员**

suffix_with()

```py
method suffix_with(*suffixes: _TextCoercedExpressionArgument[Any], dialect: str = '*') → Self
```

作为整体的语句后添加一个或多个表达式。

用于支持某些构造的特定于后端的后缀关键字。

例如：

```py
stmt = select(col1, col2).cte().suffix_with(
    "cycle empno set y_cycle to 1 default 0", dialect="oracle")
```

可以通过多次调用 `HasSuffixes.suffix_with()` 来指定多个后缀。

参数：

+   `*suffixes` – 将在目标子句之后呈现的文本或 `ClauseElement` 构造。

+   `dialect` – 可选的字符串方言名称，将此后缀的呈现限制为仅限于该方言。

```py
class sqlalchemy.sql.expression.Join
```

表示两个 `FromClause` 元素之间的 `JOIN` 构造。

`Join` 的公共构造函数是模块级别的 `join()` 函数，以及任何 `FromClause` 的 `FromClause.join()` 方法（例如 `Table`）。

另见

`join()`

`FromClause.join()`

**成员**

__init__(), description, is_derived_from(), select(), self_group()

**类签名**

`sqlalchemy.sql.expression.Join` 类（`sqlalchemy.sql.roles.DMLTableRole`, `sqlalchemy.sql.expression.FromClause`）

```py
method __init__(left: _FromClauseArgument, right: _FromClauseArgument, onclause: _OnClauseArgument | None = None, isouter: bool = False, full: bool = False)
```

构造一个新的 `Join`。

常见的入口点是 `join()` 函数或任何 `FromClause` 对象的 `FromClause.join()` 方法。

```py
attribute description
```

```py
method is_derived_from(fromclause: FromClause | None) → bool
```

如果此 `FromClause` 是从给定的 `FromClause`‘派生’，则返回 `True`。

一个示例是表的别名是从该表派生的。

```py
method select() → Select
```

从此 `Join` 创建一个 `Select`。

例如：

```py
stmt = table_a.join(table_b, table_a.c.id == table_b.c.a_id)

stmt = stmt.select()
```

以上将产生类似于以下的 SQL 字符串：

```py
SELECT table_a.id, table_a.col, table_b.id, table_b.a_id
FROM table_a JOIN table_b ON table_a.id = table_b.a_id
```

```py
method self_group(against: OperatorType | None = None) → FromGrouping
```

对此 `ClauseElement` 应用‘分组’。

子类会重写此方法以返回一个“分组”构造，即括号。特别是它被“二元”表达式使用，当它们被放置到更大的表达式中时，会提供一个围绕自己的分组，以及当被放置到另一个 `select()` 的 FROM 子句中时，由 `select()` 构造使用。（注意，子查询通常应使用 `Select.alias()` 方法创建，因为许多平台要求嵌套的 SELECT 语句必须具有名称）。

当表达式组合在一起时，会自动应用 `self_group()` - 最终用户代码不应直接使用此方法。请注意，SQLAlchemy 的子句构造会考虑运算符优先级 - 因此在像 `x OR (y AND z)` 这样的表达式中可能不需要括号 - AND 优先于 OR。

`self_group()` 方法属于 `ClauseElement` 的基类，仅返回自身。

```py
class sqlalchemy.sql.expression.Lateral
```

表示一个 LATERAL 子查询。

该对象可以通过 `lateral()` 模块级函数以及所有 `FromClause` 子类的 `FromClause.lateral()` 方法构建。

虽然 LATERAL 是 SQL 标准的一部分，但目前只有较新版本的 PostgreSQL 支持该关键字。

另请参阅

LATERAL 相关性 - 用法概述。

**成员**

inherit_cache

**类签名**

类 `sqlalchemy.sql.expression.Lateral` (`sqlalchemy.sql.expression.FromClauseAlias`, `sqlalchemy.sql.expression.LateralFromClause`)

```py
attribute inherit_cache: bool | None = True
```

指示此 `HasCacheKey` 实例是否应使用其直接超类使用的缓存键生成方案。

此属性默认为 `None`，表示构造尚未考虑其是否适合参与缓存；这在功能上相当于将值设置为 `False`，但也会发出警告。

如果与对象相对应的 SQL 不会因为该类局部属性而改变，而是改变其超类，则可以在特定类上将此标志设置为 `True`。

另请参阅

为自定义构造启用缓存支持 - 为第三方或用户定义的 SQL 构造设置 `HasCacheKey.inherit_cache` 属性的一般指南。

```py
class sqlalchemy.sql.expression.ReturnsRows
```

最基本的核心构造类，具有可表示行的列概念。

虽然 SELECT 语句和 TABLE 是我们在此类别中首先考虑的主要内容，但是像 INSERT、UPDATE 和 DELETE 这样的 DML 也可以指定 RETURNING，这意味着它们可以在 CTEs 和其他形式中使用，而且 PostgreSQL 也有返回行的函数。

自 1.4 版本新增。

**成员**

exported_columns, is_derived_from()

**类签名**

类 `sqlalchemy.sql.expression.ReturnsRows` (`sqlalchemy.sql.roles.ReturnsRowsRole`, `sqlalchemy.sql.expression.DQLDMLClauseElement`)

```py
attribute exported_columns
```

一个 `ColumnCollection`，代表这个 `ReturnsRows` 的“导出”列。

“导出”列代表此 SQL 构造呈现的 `ColumnElement` 表达式集合。主要有几种类型，包括 FROM 子句的“FROM 子句列”，例如表、连接或子查询，以及“SELECTed 列”，即 SELECT 语句的“列子句”中的列，以及 DML 语句中的 RETURNING 列。

自版本 1.4 新增。

另请参阅

`FromClause.exported_columns`

`SelectBase.exported_columns`

```py
method is_derived_from(fromclause: FromClause | None) → bool
```

如果此 `ReturnsRows` 是从给定的 `FromClause` “派生”出来的，则返回 `True`。

例如，表的别名是从该表派生出来的。

```py
class sqlalchemy.sql.expression.ScalarSelect
```

表示一个标量子查询。

通过调用 `SelectBase.scalar_subquery()` 方法创建 `ScalarSelect`。然后，该对象作为 `ColumnElement` 层次结构中的 SQL 列表达式参与其他 SQL 表达式。

另请参阅

`SelectBase.scalar_subquery()`

标量和相关子查询 - 在 2.0 教程中

**成员**

correlate(), correlate_except(), inherit_cache, self_group(), where()

**类签名**

类 `sqlalchemy.sql.expression.ScalarSelect` (`sqlalchemy.sql.roles.InElementRole`, `sqlalchemy.sql.expression.Generative`, `sqlalchemy.sql.expression.GroupedElement`, `sqlalchemy.sql.expression.ColumnElement`)

```py
method correlate(*fromclauses: Literal[None, False] | _FromClauseArgument) → Self
```

返回一个新的`ScalarSelect`，它将相关联给定的 FROM 子句到封闭`Select`的 FROM 子句。

该方法是从底层`Select`的`Select.correlate()`方法中镜像出来的。该方法应用了:meth:_sql.Select.correlate`方法，然后返回一个新的`ScalarSelect`针对该语句。

版本 1.4 中的新内容：以前，`ScalarSelect.correlate()`方法仅从`Select`中可用。

参数：

***fromclauses** - 一个或多个`FromClause`构造的列表，或其他兼容的构造（即 ORM 映射的类），以成为相关集合的一部分。

还请参阅

`ScalarSelect.correlate_except()`

标量和相关子查询 - 在 2.0 教程

```py
method correlate_except(*fromclauses: Literal[None, False] | _FromClauseArgument) → Self
```

返回一个新的`ScalarSelect`，它将从自动相关过程中省略给定的 FROM 子句。

该方法是从底层`Select`的`Select.correlate_except()`方法中镜像出来的。该方法应用了:meth:_sql.Select.correlate_except`方法，然后返回一个新的`ScalarSelect`针对该语句。

版本 1.4 中的新内容：以前，`ScalarSelect.correlate_except()`方法仅从`Select`中可用。

参数：

***fromclauses** - 一个或多个`FromClause`构造的列表，或其他兼容的构造（即 ORM 映射的类），以成为相关例外集合的一部分。

另请参见

`ScalarSelect.correlate()`

标量和相关子查询 - 在 2.0 教程中

```py
attribute inherit_cache: bool | None = True
```

指示此 `HasCacheKey` 实例是否应使用其直接超类使用的缓存键生成方案。

该属性默认为 `None`，表示构造尚未考虑是否适合参与缓存；这在功能上等同于将值设置为 `False`，只是还会发出警告。

如果与对象对应的 SQL 不基于仅属于此类而不是其超类的属性更改，则可以在特定类上将此标志设置为 `True`。

另请参阅

为自定义构造启用缓存支持 - 设置第三方或用户定义的 SQL 构造的 `HasCacheKey.inherit_cache` 属性的一般指南。

```py
method self_group(against: OperatorType | None = None) → ColumnElement[Any]
```

对这个 `ClauseElement` 应用一个“分组”。

此方法被子类重写以返回一个“分组”构造，即括号。特别是当“二元”表达式被放置到更大的表达式中时，它们会提供一个围绕自身的分组，以及当 `select()` 构造被放置到另一个 `select()` 的 FROM 子句中时。 （请注意，子查询通常应使用 `Select.alias()` 方法创建，因为许多平台要求嵌套的 SELECT 语句必须被命名）。

当表达式组合在一起时，`self_group()` 的应用是自动的 - 最终用户代码不应直接使用此方法。请注意，SQLAlchemy 的子句构造会考虑运算符优先级 - 因此在诸如 `x OR (y AND z)` 这样的表达式中可能不需要括号 - AND 优先于 OR。

`ClauseElement` 的基本 `self_group()` 方法只返回自身。

```py
method where(crit: _ColumnExpressionArgument[bool]) → Self
```

对此 `ScalarSelect` 引用的 SELECT 语句应用 WHERE 子句。

```py
class sqlalchemy.sql.expression.Select
```

表示一个 `SELECT` 语句。

`Select` 对象通常使用 `select()` 函数构造。详情请参阅该函数。

另请参阅

`select()`

使用 SELECT 语句 - 在 2.0 教程中

**成员**

__init__(), add_columns(), add_cte(), alias(), as_scalar(), c, column(), column_descriptions, columns_clause_froms, correlate(), correlate_except(), corresponding_column(), cte(), distinct(), except_(), except_all(), execution_options(), exists(), exported_columns, fetch(), filter(), filter_by(), from_statement(), froms, get_children(), get_execution_options(), get_final_froms(), get_label_style(), group_by(), having(), inherit_cache, inner_columns, intersect(), intersect_all(), is_derived_from(), join(), join_from(), label(), lateral(), limit(), offset(), options(), order_by(), outerjoin(), outerjoin_from(), prefix_with(), reduce_columns(), replace_selectable(), scalar_subquery(), select(), select_from(), selected_columns, self_group(), set_label_style(), slice(), subquery(), suffix_with(), union(), union_all(), where(), whereclause, with_for_update(), with_hint(), with_only_columns(), with_statement_hint()

**类签名**

类`sqlalchemy.sql.expression.Select`（`sqlalchemy.sql.expression.HasPrefixes`，`sqlalchemy.sql.expression.HasSuffixes`，`sqlalchemy.sql.expression.HasHints`，`sqlalchemy.sql.expression.HasCompileState`，`sqlalchemy.sql.expression._SelectFromElements`，`sqlalchemy.sql.expression.GenerativeSelect`，`sqlalchemy.sql.expression.TypedReturnsRows`）

```py
method __init__(*entities: _ColumnsClauseArgument[Any])
```

构造一个新的`Select`。

`Select`的公共构造函数是`select()`函数。

```py
method add_columns(*entities: _ColumnsClauseArgument[Any]) → Select[Any]
```

返回一个新的`select()`构造，其中包含给定实体附加到其列子句中。

例如：

```py
my_select = my_select.add_columns(table.c.new_column)
```

列子句中的原始表达式保持不变。要用新表达式替换原始表达式，请参见方法`Select.with_only_columns()`。

参数：

***entities** – 要添加到列子句中的列、表或其他实体表达式

另请参见

`Select.with_only_columns()` - 替换现有表达式而不是追加。

同时选择多个 ORM 实体 - 以 ORM 为中心的示例

```py
method add_cte(*ctes: CTE, nest_here: bool = False) → Self
```

*继承自* `HasCTE.add_cte()` *方法的* `HasCTE`

向此语句添加一个或多个`CTE`构造。

此方法将将给定的`CTE`构造与父语句关联，以便它们将分别无条件地呈现在最终语句的 WITH 子句中，即使在语句或任何子选择中未引用它们。

当可选的`HasCTE.add_cte.nest_here`参数设置为 True 时，每个给定的`CTE`将直接与此语句一起渲染为 WITH 子句，而不是被移动到最终渲染的语句顶部，即使此语句作为较大语句中的子查询进行渲染也是如此。

此方法有两个一般用途。一个是嵌入服务于某种目的而不被显式引用的 CTE 语句，比如将 DML 语句（如 INSERT 或 UPDATE）嵌入为一个 CTE 与可能间接引用其结果的主要语句内联的用例。另一个是提供对应于应保持直接渲染为可能嵌套在较大语句中的特定语句的 CTE 结构系列的确切放置位置的控制。

例如：

```py
from sqlalchemy import table, column, select
t = table('t', column('c1'), column('c2'))

ins = t.insert().values({"c1": "x", "c2": "y"}).cte()

stmt = select(t).add_cte(ins)
```

渲染后：

```py
WITH anon_1 AS
(INSERT INTO t (c1, c2) VALUES (:param_1, :param_2))
SELECT t.c1, t.c2
FROM t
```

在上面的示例中，“anon_1” CTE 虽然在 SELECT 语句中没有被引用，但仍完成了运行 INSERT 语句的任务。

类似地，在与 DML 相关的上下文中，使用 PostgreSQL `Insert` 构造来生成“upsert”：

```py
from sqlalchemy import table, column
from sqlalchemy.dialects.postgresql import insert

t = table("t", column("c1"), column("c2"))

delete_statement_cte = (
    t.delete().where(t.c.c1 < 1).cte("deletions")
)

insert_stmt = insert(t).values({"c1": 1, "c2": 2})
update_statement = insert_stmt.on_conflict_do_update(
    index_elements=[t.c.c1],
    set_={
        "c1": insert_stmt.excluded.c1,
        "c2": insert_stmt.excluded.c2,
    },
).add_cte(delete_statement_cte)

print(update_statement)
```

上述语句渲染为：

```py
WITH deletions AS
(DELETE FROM t WHERE t.c1 < %(c1_1)s)
INSERT INTO t (c1, c2) VALUES (%(c1)s, %(c2)s)
ON CONFLICT (c1) DO UPDATE SET c1 = excluded.c1, c2 = excluded.c2
```

新版本 1.4.21 中新增。

参数：

+   `*ctes` –

    零个或多个`CTE` 构造。

    从版本 2.0 开始更改：接受多个 CTE 实例

+   `nest_here` –

    如果为 True，则给定的 CTE 或 CTEs 将被渲染，就好像在将它们添加到此`HasCTE`时指定了`HasCTE.cte.nesting`标志为 True。假设给定的 CTEs 在外部包含语句中也没有被引用，则在给出此标志时，当这些 CTEs 被渲染时，应在此语句级别上渲染。

    新版本 2.0 中新增。

    另请参阅

    `HasCTE.cte.nesting`

```py
method alias(name: str | None = None, flat: bool = False) → Subquery
```

*从* `SelectBase` *的* `SelectBase.alias()` *方法继承*

返回针对此`SelectBase`的命名子查询。

对于一个 `SelectBase`（而不是一个 `FromClause`），这将返回一个行为大部分与与 `FromClause` 一起使用的 `Alias` 对象相同的 `Subquery` 对象。

从版本 1.4 开始更改：`SelectBase.alias()` 方法现在是 `SelectBase.subquery()` 方法的同义词。

```py
method as_scalar() → ScalarSelect[Any]
```

*继承自* `SelectBase.as_scalar()` *方法的* `SelectBase`

从版本 1.4 开始弃用：`SelectBase.as_scalar()` 方法已被弃用，并将在将来的版本中删除。请参考 `SelectBase.scalar_subquery()`。

```py
attribute c
```

*继承自* `SelectBase.c` *属性的* `SelectBase`

从版本 1.4 开始弃用：`SelectBase.c` 和 `SelectBase.columns` 属性已被弃用，并将在将来的版本中删除；这些属性隐式地创建一个应该是明确的子查询。请首先调用 `SelectBase.subquery()` 来创建一个子查询，然后该子查询包含此属性。要访问此 SELECT 对象从中选择的列，请使用 `SelectBase.selected_columns` 属性。

```py
method column(column: _ColumnsClauseArgument[Any]) → Select[Any]
```

返回一个新的带有给定列表达式添加到其列子句的 `select()` 构造。

从版本 1.4 开始弃用：`Select.column()` 方法已被弃用，并将在将来的版本中删除。请使用 `Select.add_columns()`

例如：

```py
my_select = my_select.column(table.c.new_column)
```

请参阅 `Select.with_only_columns()` 的文档，了解如何添加 / 替换 `Select` 对象的列的指南。

```py
attribute column_descriptions
```

返回一个 启用插件 的‘列描述’结构，引用此语句所选的列。

这个属性通常在使用 ORM 时很有用，因为会返回一个包含有关映射实体信息的扩展结构。部分 检查来自启用了 ORM 的 SELECT 和 DML 语句的实体和列 包含更多背景信息。

对于仅核心语句，此访问器返回的结构派生自由 `Select.selected_columns` 访问器返回的相同对象，格式化为包含键 `name`、`type` 和 `expr` 的字典列表，这些键指示要选择的列表达式：

```py
>>> stmt = select(user_table)
>>> stmt.column_descriptions
[
 {
 'name': 'id',
 'type': Integer(),
 'expr': Column('id', Integer(), ...)},
 {
 'name': 'name',
 'type': String(length=30),
 'expr': Column('name', String(length=30), ...)}
]
```

1.4.33 版中的更改：`Select.column_descriptions` 属性返回一个仅针对核心实体的结构，而不仅仅是 ORM 实体。

另请参阅

`UpdateBase.entity_description` - `insert()`、`update()` 或 `delete()` 的实体信息

从启用了 ORM 的 SELECT 和 DML 语句中检查实体和列 - ORM 背景

```py
attribute columns_clause_froms
```

返回由此 SELECT 语句的列子句暗示的 `FromClause` 对象集。

版本 1.4.23 中的新内容。

另请参阅

`Select.froms` - 考虑了完整语句的“最终” FROM 列表

`Select.with_only_columns()` - 使用此集合设置新的 FROM 列表

```py
method correlate(*fromclauses: Literal[None, False] | _FromClauseArgument) → Self
```

返回一个新的 `Select`，将给定的 FROM 子句与封闭 `Select` 的 FROM 子句相关联。

调用此方法将关闭 `Select` 对象的默认行为“自动关联”。通常情况下，通过其 WHERE 子句、ORDER BY、HAVING 或 columns 子句 封闭此对象的 `Select` 中出现的 FROM 元素将从此 `Select` 对象的 FROM 子句 中省略。使用 `Select.correlate()` 方法设置显式关联集合提供了一个固定的 FROM 对象列表，这些对象可能参与此过程。

当使用 `Select.correlate()` 应用特定的 FROM 子句进行关联时，无论此 `Select` 对象相对于封闭的引用相对于相同的 FROM 对象的外层 `Select` 嵌套多深，FROM 元素都会成为关联的候选对象。这与“自动关联”的行为形成对比，后者仅与一个直接封闭的 `Select` 相关联。多级关联确保封闭和封闭的 `Select` 之间的链接始终通过至少一个 WHERE/ORDER BY/HAVING/columns 子句，以便进行关联。

如果传入 `None`，`Select` 对象将不会关联其任何 FROM 条目，所有条目都将无条件地在本地 FROM 子句中渲染。

参数：

***fromclauses** – 一个或多个 `FromClause` 或其他与 FROM 兼容的构造，如 ORM 映射实体，以成为关联集合的一部分；或者传递单个值 `None` 来删除所有现有的关联。

另请参阅

`Select.correlate_except()`

标量和关联子查询

```py
method correlate_except(*fromclauses: Literal[None, False] | _FromClauseArgument) → Self
```

返回一个新的 `Select`，它将从自动关联过程中省略给定的 FROM 子句。

调用 `Select.correlate_except()` 会关闭给定 FROM 元素的`Select`对象的默认“自动关联”行为。在此指定的元素将无条件出现在 FROM 列表中，而所有其他 FROM 元素仍然受到正常的自动关联行为的影响。

如果传递了`None`，或者没有传递参数，则`Select`对象将关联其所有 FROM 条目。

参数：

***fromclauses** – 一个或多个`FromClause`构造的列表，或其他兼容的构造（即 ORM 映射的类），成为关联例外集合的一部分。

另请参见

`Select.correlate()`

标量和相关子查询

```py
method corresponding_column(column: KeyedColumnElement[Any], require_embedded: bool = False) → KeyedColumnElement[Any] | None
```

*继承自* `Selectable.corresponding_column()` *方法的* `Selectable`

给定一个`ColumnElement`，返回此`Selectable`的`Selectable.exported_columns`集合中对应于该原始`ColumnElement`的导出`ColumnElement`对象，通过共同祖先列进行对应。

参数：

+   `column` – 要匹配的目标`ColumnElement`。

+   `require_embedded` – 仅当给定的`ColumnElement`实际存在于此`Selectable`的子元素中时，才返回相应列，通常情况下，如果该列仅与此`Selectable`的导出列之一共享共同祖先，则列会匹配。

另请参见

`Selectable.exported_columns` - 用于操作的`ColumnCollection`。

`ColumnCollection.corresponding_column()` - 实现方法。

```py
method cte(name: str | None = None, recursive: bool = False, nesting: bool = False) → CTE
```

*从* `HasCTE.cte()` *方法继承而来* `HasCTE`

返回一个新的`CTE`或公共表达式实例。

公共表达式是一种 SQL 标准，其中 SELECT 语句可以在主语句的基础上绘制出与主语句一起指定的辅助语句，使用一个叫做“WITH”的子句。还可以使用特殊的语义关于 UNION 来允许“递归”查询，其中一个 SELECT 语句可以绘制出先前已选择的行集。

CTE 也可以应用于某些数据库上的 DML 构造 UPDATE、INSERT 和 DELETE，既作为与 RETURNING 结合使用时 CTE 行的来源，也作为 CTE 行的消费者。

SQLAlchemy 检测到`CTE`对象，这些对象与`Alias`对象类似，作为要传递到语句的 FROM 子句以及语句顶部的 WITH 子句的特殊元素。

对于诸如 PostgreSQL 的“MATERIALIZED”和“NOT MATERIALIZED”等特殊前缀，可以使用`CTE.prefix_with()`方法来建立这些。

在版本 1.3.13 中更改：添加了对前缀的支持。特别是- MATERIALIZED 和 NOT MATERIALIZED。

参数：

+   `name` – 给公共表达式的名称。与`FromClause.alias()`类似，如果将名称留空，则在查询编译时将使用匿名符号。

+   `recursive` – 如果设置为`True`，将渲染`WITH RECURSIVE`。递归公共表达式旨在与 UNION ALL 结合使用，以便从已选择的行中派生行。

+   `nesting` –

    如果设置为`True`，将在引用它的语句中本地渲染 CTE。对于更复杂的场景，也可以使用`HasCTE.add_cte()`方法，使用`HasCTE.add_cte.nest_here`参数更精细地控制特定 CTE 的确切放置位置。

    新功能在版本 1.4.24 中添加。

    另请参阅

    `HasCTE.add_cte()`

以下示例包括两个来自 PostgreSQL 文档的示例[`www.postgresql.org/docs/current/static/queries-with.html`](https://www.postgresql.org/docs/current/static/queries-with.html)，以及其他示例。

示例 1，非递归：

```py
from sqlalchemy import (Table, Column, String, Integer,
                        MetaData, select, func)

metadata = MetaData()

orders = Table('orders', metadata,
    Column('region', String),
    Column('amount', Integer),
    Column('product', String),
    Column('quantity', Integer)
)

regional_sales = select(
                    orders.c.region,
                    func.sum(orders.c.amount).label('total_sales')
                ).group_by(orders.c.region).cte("regional_sales")

top_regions = select(regional_sales.c.region).\
        where(
            regional_sales.c.total_sales >
            select(
                func.sum(regional_sales.c.total_sales) / 10
            )
        ).cte("top_regions")

statement = select(
            orders.c.region,
            orders.c.product,
            func.sum(orders.c.quantity).label("product_units"),
            func.sum(orders.c.amount).label("product_sales")
    ).where(orders.c.region.in_(
        select(top_regions.c.region)
    )).group_by(orders.c.region, orders.c.product)

result = conn.execute(statement).fetchall()
```

示例 2，WITH RECURSIVE：

```py
from sqlalchemy import (Table, Column, String, Integer,
                        MetaData, select, func)

metadata = MetaData()

parts = Table('parts', metadata,
    Column('part', String),
    Column('sub_part', String),
    Column('quantity', Integer),
)

included_parts = select(\
    parts.c.sub_part, parts.c.part, parts.c.quantity\
    ).\
    where(parts.c.part=='our part').\
    cte(recursive=True)

incl_alias = included_parts.alias()
parts_alias = parts.alias()
included_parts = included_parts.union_all(
    select(
        parts_alias.c.sub_part,
        parts_alias.c.part,
        parts_alias.c.quantity
    ).\
    where(parts_alias.c.part==incl_alias.c.sub_part)
)

statement = select(
            included_parts.c.sub_part,
            func.sum(included_parts.c.quantity).
              label('total_quantity')
        ).\
        group_by(included_parts.c.sub_part)

result = conn.execute(statement).fetchall()
```

示例 3，使用 UPDATE 和 INSERT 进行 upsert 与 CTE：

```py
from datetime import date
from sqlalchemy import (MetaData, Table, Column, Integer,
                        Date, select, literal, and_, exists)

metadata = MetaData()

visitors = Table('visitors', metadata,
    Column('product_id', Integer, primary_key=True),
    Column('date', Date, primary_key=True),
    Column('count', Integer),
)

# add 5 visitors for the product_id == 1
product_id = 1
day = date.today()
count = 5

update_cte = (
    visitors.update()
    .where(and_(visitors.c.product_id == product_id,
                visitors.c.date == day))
    .values(count=visitors.c.count + count)
    .returning(literal(1))
    .cte('update_cte')
)

upsert = visitors.insert().from_select(
    [visitors.c.product_id, visitors.c.date, visitors.c.count],
    select(literal(product_id), literal(day), literal(count))
        .where(~exists(update_cte.select()))
)

connection.execute(upsert)
```

示例 4，嵌套 CTE（SQLAlchemy 1.4.24 及以上）：

```py
value_a = select(
    literal("root").label("n")
).cte("value_a")

# A nested CTE with the same name as the root one
value_a_nested = select(
    literal("nesting").label("n")
).cte("value_a", nesting=True)

# Nesting CTEs takes ascendency locally
# over the CTEs at a higher level
value_b = select(value_a_nested.c.n).cte("value_b")

value_ab = select(value_a.c.n.label("a"), value_b.c.n.label("b"))
```

上述查询将呈现第二个 CTE 嵌套在第一个内部，如下所示：

```py
WITH
    value_a AS
        (SELECT 'root' AS n),
    value_b AS
        (WITH value_a AS
            (SELECT 'nesting' AS n)
        SELECT value_a.n AS n FROM value_a)
SELECT value_a.n AS a, value_b.n AS b
FROM value_a, value_b
```

可以使用`HasCTE.add_cte()`方法设置相同的 CTE，如下所示（SQLAlchemy 2.0 及以上）：

```py
value_a = select(
    literal("root").label("n")
).cte("value_a")

# A nested CTE with the same name as the root one
value_a_nested = select(
    literal("nesting").label("n")
).cte("value_a")

# Nesting CTEs takes ascendency locally
# over the CTEs at a higher level
value_b = (
    select(value_a_nested.c.n).
    add_cte(value_a_nested, nest_here=True).
    cte("value_b")
)

value_ab = select(value_a.c.n.label("a"), value_b.c.n.label("b"))
```

示例 5，非线性 CTE（SQLAlchemy 1.4.28 及以上）：

```py
edge = Table(
    "edge",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("left", Integer),
    Column("right", Integer),
)

root_node = select(literal(1).label("node")).cte(
    "nodes", recursive=True
)

left_edge = select(edge.c.left).join(
    root_node, edge.c.right == root_node.c.node
)
right_edge = select(edge.c.right).join(
    root_node, edge.c.left == root_node.c.node
)

subgraph_cte = root_node.union(left_edge, right_edge)

subgraph = select(subgraph_cte)
```

上述查询将在递归 CTE 内呈现 2 个 UNION：

```py
WITH RECURSIVE nodes(node) AS (
        SELECT 1 AS node
    UNION
        SELECT edge."left" AS "left"
        FROM edge JOIN nodes ON edge."right" = nodes.node
    UNION
        SELECT edge."right" AS "right"
        FROM edge JOIN nodes ON edge."left" = nodes.node
)
SELECT nodes.node FROM nodes
```

另请参见

`Query.cte()` - `HasCTE.cte()`的 ORM 版本。

```py
method distinct(*expr: _ColumnExpressionArgument[Any]) → Self
```

返回一个新的`select()`构造，该构造将对整个 SELECT 语句应用 DISTINCT。

例如：

```py
from sqlalchemy import select
stmt = select(users_table.c.id, users_table.c.name).distinct()
```

上述将产生类似于的语句：

```py
SELECT DISTINCT user.id, user.name FROM user
```

该方法还接受一个`*expr`参数，该参数生成 PostgreSQL 特定的`DISTINCT ON`表达式。在不支持此语法的���他后端上使用此参数将引发错误。

参数：

***expr** –

可选的列表达式。当存在时，PostgreSQL 方言将呈现`DISTINCT ON (<expressions>)`构造。在其他后端上将引发弃用警告和/或`CompileError`。

从版本 1.4 开始弃用：在其他方言中使用*expr 已弃用，并将在将来的版本中引发`CompileError`。

```py
method except_(*other: _SelectStatementForCompoundArgument) → CompoundSelect
```

返回此`select()`构造与提供的可选参数中的给定可选择的 SQL`EXCEPT`。

参数：

***other** –

一个或多个用于创建 UNION 的元素。

从版本 1.4.28 开始更改：现在接受多个元素。

```py
method except_all(*other: _SelectStatementForCompoundArgument) → CompoundSelect
```

返回此`select()`构造与提供的可选参数中的给定可选择的 SQL`EXCEPT ALL`。

参数：

***other** –

一个或多个用于创建 UNION 的元素。

从版本 1.4.28 开始更改：现在接受多个元素。

```py
method execution_options(**kw: Any) → Self
```

*继承自* `Executable.execution_options()` *方法的* `Executable`

为在执行期间生效的语句设置非 SQL 选项。

可以在许多范围内设置执行选项，包括每个语句、每个连接或每次执行，使用诸如`Connection.execution_options()`和接受选项字典的参数的方法，例如`Connection.execute.execution_options`和`Session.execute.execution_options`。

与其他类型的选项（如 ORM 加载程序选项）相比，执行选项的主要特征是**执行选项从不影响查询的编译 SQL，只影响 SQL 语句本身如何调用或结果如何获取**。也就是说，执行选项不是 SQL 编译所能容纳的内容，它们也不被认为是语句的缓存状态的一部分。

`Executable.execution_options()` 方法是生成的，就像应用于`Engine`和`Query`对象的方法一样，这意味着当调用该方法时，将返回对象的副本，该副本应用给定的参数，但原始对象保持不变：

```py
statement = select(table.c.x, table.c.y)
new_statement = statement.execution_options(my_option=True)
```

这种行为的一个例外是`Connection`对象，在这种情况下，`Connection.execution_options()` 方法明确**不**是生成的。

可以传递给`Executable.execution_options()`和其他相关方法和参数字典的选项类型包括被 SQLAlchemy Core 或 ORM 明确消耗的参数，以及 SQLAlchemy 未定义的任意关键字参数，这意味着这些方法和/或参数字典可用于与自定义代码交互的用户定义参数，可以使用诸如`Executable.get_execution_options()`和`Connection.get_execution_options()`等方法访问参数，或者在选定的事件钩子中使用专用的`execution_options`事件参数，例如`ConnectionEvents.before_execute.execution_options`或`ORMExecuteState.execution_options`，例如：

```py
from sqlalchemy import event

@event.listens_for(some_engine, "before_execute")
def _process_opt(conn, statement, multiparams, params, execution_options):
    "run a SQL function before invoking a statement"

    if execution_options.get("do_special_thing", False):
        conn.exec_driver_sql("run_special_function()")
```

在 SQLAlchemy 明确识别的选项范围内，大多数适用于特定类别的对象而不是其他对象。最常见的执行选项包括：

+   `Connection.execution_options.isolation_level` - 通过`Engine`为连接或一类连接设置隔离级别。此选项仅被`Connection`或`Engine`接受。

+   `Connection.execution_options.stream_results` - 表示结果应该使用服务器端游标获取；此选项可被`Connection`、`Connection.execute.execution_options`参数上的`Connection.execute()`以及`Executable.execution_options()`在 SQL 语句对象上接受，同样也被 ORM 构造如`Session.execute()`接受。

+   `Connection.execution_options.compiled_cache` - 表示将作为`Connection`或`Engine`的 SQL 编译缓存的字典，同样也适用于 ORM 方法如`Session.execute()`。可以传递`None`来禁用语句的缓存。此选项不被`Executable.execution_options()`接受，因为在语句对象中携带编译缓存是不明智的。

+   `Connection.execution_options.schema_translate_map` - 由模式翻译映射功能使用的模式名称映射，被`Connection`、`Engine`、`Executable`接受，同样也被 ORM 构造如`Session.execute()`接受。

另请参阅

`Connection.execution_options()`

`Connection.execute.execution_options`

`Session.execute.execution_options`

ORM 执行选项 - 所有 ORM 特定执行选项的文档

```py
method exists() → Exists
```

*继承自* `SelectBase.exists()` *方法的* `SelectBase`

返回此可选择项的`Exists`表示，可用作列表达式。

返回的对象是`Exists`的一个实例。

另请参阅

`exists()`

EXISTS 子查询 - 在 2.0 风格教程中。

版本 1.4 中新增。

```py
attribute exported_columns
```

*继承自* `SelectBase.exported_columns` *属性的* `SelectBase`

代表此`Selectable`的“导出”列的`ColumnCollection`，不包括`TextClause`构造。

`SelectBase`对象的“导出”列与`SelectBase.selected_columns`集合是同义词。

版本 1.4 中新增。

另请参阅

`Select.exported_columns`

`Selectable.exported_columns`

`FromClause.exported_columns`

```py
method fetch(count: _LimitOffsetType, with_ties: bool = False, percent: bool = False) → Self
```

*继承自* `GenerativeSelect.fetch()` *方法的* `GenerativeSelect`

返回一个应用了给定 FETCH FIRST 标准的新可选择项。

此为数值，通常在生成的选择中以 `FETCH {FIRST | NEXT} [ count ] {ROW | ROWS} {ONLY | WITH TIES}` 表达式的形式呈现。此功能目前已实现在 Oracle、PostgreSQL 和 MSSQL 中。

使用 `GenerativeSelect.offset()` 指定偏移量。

注意

`GenerativeSelect.fetch()` 方法将替换任何使用 `GenerativeSelect.limit()` 应用的子句。

新版本 1.4 中新增。

参数：

+   `count` – 整数 COUNT 参数，或提供整数结果的 SQL 表达式。当 `percent=True` 时，这将表示要返回的行的百分比，而不是绝对值。传递 `None` 以重置它。

+   `with_ties` – 当为 `True` 时，使用 WITH TIES 选项以返回与 `ORDER BY` 子句中的最后一个位置并列的任何附加行。在这种情况下，`ORDER BY` 可能是强制性的。默认为 `False`

+   `percent` – 当为 `True` 时，`count` 表示要返回的所选行的百分比。默认为 `False`

另请参阅

`GenerativeSelect.limit()`

`GenerativeSelect.offset()`

```py
method filter(*criteria: _ColumnExpressionArgument[bool]) → Self
```

`Select.where()` 方法的同义词。

```py
method filter_by(**kwargs: Any) → Self
```

将给定的过滤条件作为 WHERE 子句应用于此选择。

```py
method from_statement(statement: ReturnsRowsRole) → ExecutableReturnsRows
```

将此 `Select` 选择的列应用于另一个语句。

此操作是特定于插件的，如果此 `Select` 不选择来自启用插件的实体，则会引发不支持的异常。

语句通常是 `text()` 或 `select()` 构造，并应返回适用于由此 `Select` 表示的实体的列集。

另请参阅

从文本语句获取 ORM 结果 - ORM 查询指南中的使用示例

```py
attribute froms
```

返回显示的 `FromClause` 元素列表。

自版本 1.4.23 起已弃用：`Select.froms` 属性已移至 `Select.get_final_froms()` 方法。

```py
method get_children(**kw: Any) → Iterable[ClauseElement]
```

返回此 `HasTraverseInternals` 的直接子元素。

这用于访问遍历。

**kw 可能包含改变返回集合的标志，例如返回一部分项目以减少更大的遍历，或者返回不同上下文的子项目（例如模式级别的集合而不是从句级别的集合）。

```py
method get_execution_options() → _ExecuteOptions
```

*继承自* `Executable` *的* `Executable.get_execution_options()` *方法*

获取在执行期间生效的非 SQL 选项。

自版本 1.3 起新增。

另请参见

`Executable.execution_options()`

```py
method get_final_froms() → Sequence[FromClause]
```

计算最终显示的 `FromClause` 元素列表。

此方法将通过完整的计算来确定结果 SELECT 语句中将显示哪些 FROM 元素，包括用 JOIN 对象遮蔽单个表以及用于 ORM 使用情况的完整计算，包括急加载子句。

对于 ORM 使用，此访问器返回**编译后**的 FROM 对象列表；此集合将包括诸如急加载表和连接之类的元素。对象将**不**被启用 ORM，并且不能作为 `Select.select_froms()` 集合的替代；此外，对于启用 ORM 语句，该方法的性能不佳，因为它将导致完整的 ORM 构造过程。

要检索由最初传递给 `Select` 的“列”集合隐含的 FROM 列表，请使用 `Select.columns_clause_froms` 访问器。

要从替代列集中选择而保持 FROM 列表，请使用 `Select.with_only_columns()` 方法，并传递 `Select.with_only_columns.maintain_column_froms` 参数。

1.4.23 版本中的新功能：- `Select.get_final_froms()` 方法取代了之前的 `Select.froms` 访问器，该访问器已被弃用。

另请参见

`Select.columns_clause_froms`

```py
method get_label_style() → SelectLabelStyle
```

*继承自* `GenerativeSelect.get_label_style()` *方法的* `GenerativeSelect`

检索当前标签样式。

1.4 版本中的新功能。

```py
method group_by(_GenerativeSelect__first: Literal[None, _NoArg.NO_ARG] | _ColumnExpressionOrStrLabelArgument[Any] = _NoArg.NO_ARG, *clauses: _ColumnExpressionOrStrLabelArgument[Any]) → Self
```

*继承自* `GenerativeSelect.group_by()` *方法的* `GenerativeSelect`

返回一个具有给定的 GROUP BY 准则列表的新的可选择项。

通过传递`None`可以取消所有现有的 GROUP BY 设置。

例如：

```py
stmt = select(table.c.name, func.max(table.c.stat)).\
group_by(table.c.name)
```

参数：

***clauses** – 一系列`ColumnElement`构造，将用于生成 GROUP BY 子句。

另请参见

GROUP BY / HAVING 中的聚合函数 - 在 SQLAlchemy 统一教程中

按标签排序或分组 - 在 SQLAlchemy 统一教程中

```py
method having(*having: _ColumnExpressionArgument[bool]) → Self
```

返回一个新的 `select()` 构造，其中包含给定表达式添加到其 HAVING 子句中，并通过 AND 连接到现有子句（如果有）。

```py
attribute inherit_cache: bool | None = None
```

*继承自* `HasCacheKey.inherit_cache` *属性的* `HasCacheKey` *实例*

表示此`HasCacheKey`实例是否应使用其直接超类使用的缓存密钥生成方案。

该属性默认为`None`，表示一个构造尚未考虑是否适合参与缓存；这在功能上等同于将值设置为`False`，除了还会发出警告。

如果与此类本地属性以及不是其超类的属性无关的属性对应的 SQL 不会更改，则可以在特定类上将此标志设置为`True`。

另请参见

为自定义构造启用缓存支持 - 为第三方或用户定义的 SQL 构造设置`HasCacheKey.inherit_cache`属性的一般指南。

```py
attribute inner_columns
```

所有将被渲染到生成的 SELECT 语句的列子句中的`ColumnElement`表达式的迭代器。

从 1.4 版本开始，此方法已被弃用，并由`Select.exported_columns`集合取代。

```py
method intersect(*other: _SelectStatementForCompoundArgument) → CompoundSelect
```

返回此 select()构造与作为位置参数提供的给定 selectables 的 SQL `INTERSECT`。

参数：

+   `*other` –

    一个或多个要创建联合的元素。

    自版本 1.4.28 起更改：现在接受多个元素。

+   `**kwargs` – 关键字参数将转发到新创建的`CompoundSelect`对象的构造函数。

```py
method intersect_all(*other: _SelectStatementForCompoundArgument) → CompoundSelect
```

返回此 select()构造与作为位置参数提供的给定 selectables 的 SQL `INTERSECT ALL`。

参数：

+   `*other` –

    一个或多个要创建联合的元素。

    自版本 1.4.28 起更改：现在接受多个元素。

+   `**kwargs` – 关键字参数将转发到新创建的`CompoundSelect`对象的构造函数。

```py
method is_derived_from(fromclause: FromClause | None) → bool
```

如果此`ReturnsRows`是从给定的`FromClause`‘派生’，则返回`True`。

一个示例是，表的别名是从该表派生的。

```py
method join(target: _JoinTargetArgument, onclause: _OnClauseArgument | None = None, *, isouter: bool = False, full: bool = False) → Self
```

对此`Select`对象的条件进行 SQL JOIN，并应用生成，返回新生成的`Select`。

例如：

```py
stmt = select(user_table).join(address_table, user_table.c.id == address_table.c.user_id)
```

上述语句生成类似于以下的 SQL：

```py
SELECT user.id, user.name FROM user JOIN address ON user.id = address.user_id
```

从版本 1.4 开始改变：`Select.join()` 现在在现有 SELECT 的 FROM 子句中创建一个 `FromClause` 源和给定目标 `FromClause` 之间的 `Join` 对象，然后将此 `Join` 添加到新生成的 SELECT 语句的 FROM 子句中。这完全重写自 1.3 中的行为，1.3 中的行为会创建一个整个 `Select` 的子查询，然后将该子查询连接到目标。

这是一个**向后不兼容的更改**，因为先前的行为大多是无用的，它会产生一个未命名的子查询，大多数数据库都会拒绝这种情况。新的行为是基于 ORM 中非常成功的 `Query.join()` 方法建模的，以支持使用具有 `Session` 的 `Select` 对象可用的 `Query` 的功能。

有关此更改的注释，请参阅 select().join() 和 outerjoin() 向当前查询添加 JOIN 条件，而不是创建子查询。

参数：

+   `target` – 连接的目标表

+   `onclause` – 连接的 ON 子句。如果省略，则根据两个表之间的 `ForeignKey` 关系自动生成 ON 子句，如果可以明确确定，则引发错误。

+   `isouter` – 如果为 True，则生成 LEFT OUTER 连接。与 `Select.outerjoin()` 相同。

+   `full` – 如果为 True，则生成 FULL OUTER 连接。

另请参见

明确的 FROM 子句和 JOINs - 在 SQLAlchemy 统一教程 中

连接 - 在 ORM 查询指南 中

`Select.join_from()`

`Select.outerjoin()`

```py
method join_from(from_: _FromClauseArgument, target: _JoinTargetArgument, onclause: _OnClauseArgument | None = None, *, isouter: bool = False, full: bool = False) → Self
```

对此`Select`对象的条件进行 SQL JOIN 并进行生成，返回新生成的`Select`。

例如：

```py
stmt = select(user_table, address_table).join_from(
    user_table, address_table, user_table.c.id == address_table.c.user_id
)
```

上述语句生成类似于以下的 SQL：

```py
SELECT user.id, user.name, address.id, address.email, address.user_id
FROM user JOIN address ON user.id = address.user_id
```

1.4 版中的新功能。

参数：

+   `from_` – 连接的左侧，在 FROM 子句中呈现，并且大致相当于使用`Select.select_from()`方法。

+   `target` – 向其连接的目标表

+   `onclause` – 连接的 ON 子句。

+   `isouter` – 如果为 True，则生成 LEFT OUTER 连接。与`Select.outerjoin()`相同。

+   `full` – 如果为 True，则生成 FULL OUTER 连接。

另请参阅

显式 FROM 子句和 JOINs - 在 SQLAlchemy 统一教程中

连接 - 在 ORM 查询指南中

`Select.join()`

```py
method label(name: str | None) → Label[Any]
```

*继承自* `SelectBase.label()` *方法的* `SelectBase`

返回此可选择的‘标量’表示，嵌入为具有标签的子查询。

另请参阅

`SelectBase.scalar_subquery()`。

```py
method lateral(name: str | None = None) → LateralFromClause
```

*继承自* `SelectBase.lateral()` *方法的* `SelectBase`

返回此`Selectable`的 LATERAL 别名。

返回值也是顶级`lateral()`函数提供的`Lateral`构造。

另请参阅

横向相关性 - 用法概述。

```py
method limit(limit: _LimitOffsetType) → Self
```

*继承自* `GenerativeSelect.limit()` *方法的* `GenerativeSelect`

返回应用给定 LIMIT 条件的新可选择对象。

这是通常呈现为`LIMIT`表达式的数值值，不能支持`LIMIT`的后端将尝试提供类似的功能。

注意

`GenerativeSelect.limit()` 方法将替换使用 `GenerativeSelect.fetch()` 应用的任何子句。

参数：

**limit** – 一个整数 `LIMIT` 参数，或者提供整数结果的 SQL 表达式。传递 `None` 来重置它。

请参阅

`GenerativeSelect.fetch()`

`GenerativeSelect.offset()`

```py
method offset(offset: _LimitOffsetType) → Self
```

*继承自* `GenerativeSelect.offset()` *方法的* `GenerativeSelect`

返回一个应用了给定 `OFFSET` 条件的新可选择项。

这是一个数值，通常在结果选择中呈现为 `OFFSET` 表达式。不支持 `OFFSET` 的后端将尝试提供类似的功能。

参数：

**offset** – 一个整数 `OFFSET` 参数，或者提供整数结果的 SQL 表达式。传递 `None` 来重置它。

请参阅

`GenerativeSelect.limit()`

`GenerativeSelect.fetch()`

```py
method options(*options: ExecutableOption) → Self
```

*继承自* `Executable.options()` *方法的* `Executable`

将选项应用于此语句。

从一般意义上讲，选项是任何可以由 SQL 编译器解释为该语句的 Python 对象。这些选项可以被特定的方言或特定类型的编译器所使用。

最常见的选项类型是应用“急加载”和其他加载行为到 ORM 查询的 ORM 级选项。然而，选项理论上可以用于许多其他目的。

关于特定类型语句的特定类型选项的背景，请参阅这些选项对象的文档。

版本 1.4 中的变更：- 在核心语句对象中添加了`Executable.options()`，以实现统一的核心/ORM 查询功能的目标。

请参阅

列加载选项 - 指的是与 ORM 查询的使用相关的选项

使用加载器选项加载关系 - 指的是与 ORM 查询的使用相关的选项

```py
method order_by(_GenerativeSelect__first: Literal[None, _NoArg.NO_ARG] | _ColumnExpressionOrStrLabelArgument[Any] = _NoArg.NO_ARG, *clauses: _ColumnExpressionOrStrLabelArgument[Any]) → Self
```

*继承自* `GenerativeSelect.order_by()` *方法的* `GenerativeSelect`

返回一个应用给定的 ORDER BY 标准的新可选择对象。

例如：

```py
stmt = select(table).order_by(table.c.id, table.c.name)
```

多次调用此方法等效于一次调用，其中所有子句都连接在一起。通过单独传递 `None` 可以取消所有现有的 ORDER BY 标准。然后可以通过再次调用 `Query.order_by()` 来添加新的 ORDER BY 标准，例如：

```py
# will erase all ORDER BY and ORDER BY new_col alone
stmt = stmt.order_by(None).order_by(new_col)
```

参数：

***clauses** – 一系列将用于生成 ORDER BY 子句的 `ColumnElement` 构造。

另请参见

ORDER BY - 在 SQLAlchemy 统一教程中

按标签排序或分组 - 在 SQLAlchemy 统一教程中

```py
method outerjoin(target: _JoinTargetArgument, onclause: _OnClauseArgument | None = None, *, full: bool = False) → Self
```

创建一个左外连接。

参数与 `Select.join()` 相同。

从版本 1.4 开始：`Select.outerjoin()` 现在在现有 SELECT 的 FROM 子句中创建一个 `Join` 对象，以及一个给定的目标 `FromClause`，然后将这个 `Join` 添加到新生成的 SELECT 语句的 FROM 子句中。这与 1.3 版本中的行为完全不同，1.3 版本会创建整个 `Select` 的子查询，然后将该子查询连接到目标。

这是一个**向后不兼容的更改**，因为以前的行为大多是无用的，产生的无名称子查询在任何情况下都被大多数数据库拒绝。新的行为是模仿 ORM 中非常成功的 `Query.join()` 方法的行为，以支持通过使用带有 `Session` 的 `Select` 对象来使 `Query` 的功能可用。

查看此更改的注释：select().join() 和 outerjoin() 将 JOIN 条件添加到当前查询，而不是创建子查询。

另请参阅

显式的 FROM 子句和 JOINs - 在 SQLAlchemy 统一教程 中

连接 - 在 ORM 查询指南 中

`Select.join()`

```py
method outerjoin_from(from_: _FromClauseArgument, target: _JoinTargetArgument, onclause: _OnClauseArgument | None = None, *, full: bool = False) → Self
```

对此 `Select` 对象的条件创建一个 SQL LEFT OUTER JOIN 并进行生成，返回新生成的 `Select`。

用法与 `Select.join_from()` 相同。

```py
method prefix_with(*prefixes: _TextCoercedExpressionArgument[Any], dialect: str = '*') → Self
```

*来自* `HasPrefixes.prefix_with()` *方法的继承* `HasPrefixes`

在语句关键字（如 SELECT、INSERT、UPDATE 或 DELETE）之后添加一个或多个表达式。生成式。

这用于支持后端特定的前缀关键字，例如 MySQL 提供的那些。

例如：

```py
stmt = table.insert().prefix_with("LOW_PRIORITY", dialect="mysql")

# MySQL 5.7 optimizer hints
stmt = select(table).prefix_with(
    "/*+ BKA(t1) */", dialect="mysql")
```

可以通过多次调用 `HasPrefixes.prefix_with()` 来指定多个前缀。

参数：

+   `*prefixes` – 文本或者`ClauseElement` 构造，将在 INSERT、UPDATE 或 DELETE 关键字之后呈现。

+   `dialect` – 可选的字符串方言名称，将限制该前缀的渲染仅适用于该方言。

```py
method reduce_columns(only_synonyms: bool = True) → Select
```

从列子句中删除冗余命名但值等效的列，并返回一个新的 `select()` 构造。

这里的“冗余”指的是一个列引用另一个列，要么基于外键，要么通过语句的 WHERE 子句中的简单等式比较。此方法的主要目的是自动构造一个具有所有唯一命名列的选择语句，而无需像`Select.set_label_style()`那样使用表格限定标签。

当基于外键省略列时，保留的是所引用的列。当基于 WHERE 等价性省略列时，保留的是列子句中的第一列。

参数：

**only_synonyms** – 当为 True 时，限制删除与等效项同名的列。否则，删除所有与另一个等效项相同的列。

```py
method replace_selectable(old: FromClause, alias: Alias) → Self
```

*继承自* `Selectable.replace_selectable()` *方法的* `Selectable`

用给定的`Alias`对象替换所有`FromClause` ‘old’的出现，返回此`FromClause`的副本。

自 1.4 版本起已弃用：`Selectable.replace_selectable()`方法已弃用，并将在将来的版本中删除。类似的功能可通过 sqlalchemy.sql.visitors 模块获得。

```py
method scalar_subquery() → ScalarSelect[Any]
```

*继承自* `SelectBase.scalar_subquery()` *方法的* `SelectBase`

返回此可选择对象的“标量”表示，可用作列表达式。

返回的对象是`ScalarSelect`的实例。

通常，只有在其列子句中只有一个列的选择语句才有资格用作标量表达式。然后可以在封闭 SELECT 的 WHERE 子句或列子句中使用标量子查询。

请注意，标量子查询与使用`SelectBase.subquery()`方法生成的 FROM 级子查询有所不同。

参见

标量和相关子查询 - 在 2.0 教程中

```py
method select(*arg: Any, **kw: Any) → Select
```

*继承自* `SelectBase.select()` *方法的* `SelectBase`

自版本 1.4 弃用：`SelectBase.select()` 方法已弃用，并将在以后的版本中删除；该方法隐式创建了一个应显式的子查询。请首先调用 `SelectBase.subquery()` 以创建一个子查询，然后可以选择它。

```py
method select_from(*froms: _FromClauseArgument) → Self
```

返回一个将给定的 FROM 表达式合并到其 FROM 对象列表中的新 `select()` 结构。

例如：

```py
table1 = table('t1', column('a'))
table2 = table('t2', column('b'))
s = select(table1.c.a).\
    select_from(
        table1.join(table2, table1.c.a==table2.c.b)
    )
```

“from” 列表是根据每个元素的标识符组成的唯一集合，因此添加一个已经存在的 `Table` 或其他可选项将不会产生任何影响。传递一个指向已经存在的 `Table` 或其他可选项的 `Join` 将隐藏该可选项的存在，而是将其呈现为 JOIN 子句中的单独元素。

虽然 `Select.select_from()` 的典型目的是用 JOIN 替换默认的派生 FROM 子句，但也可以调用它以单独的表元素，如果需要的话，多次调用，在无法从列子句完全派生 FROM 子句的情况下：

```py
select(func.count('*')).select_from(table1)
```

```py
attribute selected_columns
```

一个表示此 SELECT 语句或类似结构返回的结果集中的列的 `ColumnCollection` ，不包括 `TextClause` 结构。

此集合与 `FromClause.columns` 集合不同，因为此集合中的列不能直接嵌套在另一个 SELECT 语句内；必须首先应用一个子查询，该子查询提供了 SQL 所需的必要括号化。

对于`select()`构造，这里的集合正是在“SELECT”语句内部呈现的内容，`ColumnElement`对象直接按照给定的方式呈现，例如：

```py
col1 = column('q', Integer)
col2 = column('p', Integer)
stmt = select(col1, col2)
```

在上面，`stmt.selected_columns`将是一个包含`col1`和`col2`对象的集合。对于针对`Table`或其他`FromClause`的语句，集合将使用`FromClause.c`中的`ColumnElement`对象。

`Select.selected_columns`集合的一个用例是允许在添加额外条件时引用现有列，例如：

```py
def filter_on_id(my_select, id):
    return my_select.where(my_select.selected_columns['id'] == id)

stmt = select(MyModel)

# adds "WHERE id=:param" to the statement
stmt = filter_on_id(stmt, 42)
```

注意

`Select.selected_columns`集合不包括在列子句中使用`text()`构造建立的表达式；这些将被静默地从集合中省略。要在`Select`构造内部使用纯文本列表达式，使用`literal_column()`构造。

版本 1.4 中的新功能。

```py
method self_group(against: OperatorType | None = None) → SelectStatementGrouping | Self
```

对这个`ClauseElement`应用‘分组’。

此方法被子类重写以返回一个“分组”构造，即括号。特别是它被“二元”表达式使用，当放置到更大的表达式中时提供一个围绕自身的分组，以及当放置到另一个`select()`构造的 FROM 子句中时使用。（请注意，子查询通常应使用`Select.alias()`方法创建，因为许多平台要求嵌套的 SELECT 语句必须命名）。

随着表达式的组合，`self_group()` 的应用是自动的 - 最终用户代码不应直接使用此方法。请注意，SQLAlchemy 的子句构造会考虑运算符优先级 - 因此，例如，在表达式`x OR (y AND z)`中可能不需要括号 - AND 优先于 OR。

`ClauseElement` 的基础 `self_group()` 方法只是返回自身。

```py
method set_label_style(style: SelectLabelStyle) → Self
```

*继承自* `GenerativeSelect.set_label_style()` *方法的* `GenerativeSelect`

返回一个具有指定标签样式的新可选择对象。

有三种“标签样式”可用，`SelectLabelStyle.LABEL_STYLE_DISAMBIGUATE_ONLY`、`SelectLabelStyle.LABEL_STYLE_TABLENAME_PLUS_COL` 和 `SelectLabelStyle.LABEL_STYLE_NONE`。默认样式是 `SelectLabelStyle.LABEL_STYLE_TABLENAME_PLUS_COL`。

在现代的 SQLAlchemy 中，通常不需要更改标签样式，因为通过使用 `ColumnElement.label()` 方法更有效地使用每个表达式的标签。在过去的版本中，`LABEL_STYLE_TABLENAME_PLUS_COL` 用于区分来自不同表、别名或子查询的同名列；新的 `LABEL_STYLE_DISAMBIGUATE_ONLY` 仅将标签应用于与现有名称冲突的名称，以使此标记的影响最小化。

区分的理由主要是为了在创建子查询时，所有列表达式都可以从给定的 `FromClause.c` 集合中使用。

新版本 1.4 中：`GenerativeSelect.set_label_style()`方法取代了以前的`.apply_labels()`、`.with_labels()`和`use_labels=True`方法和/或参数的组合。

另请参阅

`LABEL_STYLE_DISAMBIGUATE_ONLY`

`LABEL_STYLE_TABLENAME_PLUS_COL`

`LABEL_STYLE_NONE`

`LABEL_STYLE_DEFAULT`

```py
method slice(start: int, stop: int) → Self
```

*继承自* `GenerativeSelect.slice()` *方法的* `GenerativeSelect`

根据切片将 LIMIT / OFFSET 应用于此语句。

开始和停止索引的行为类似于 Python 内置的`range()`函数的参数。该方法提供了一种使用`LIMIT`/`OFFSET`来获取查询片段的替代方法。

例如，

```py
stmt = select(User).order_by(User).id.slice(1, 3)
```

呈现为

```py
SELECT  users.id  AS  users_id,
  users.name  AS  users_name
FROM  users  ORDER  BY  users.id
LIMIT  ?  OFFSET  ?
(2,  1)
```

注意

`GenerativeSelect.slice()`方法将替换任何应用于`GenerativeSelect.fetch()`的子句。

新版本 1.4 中：从 ORM 推广出`GenerativeSelect.slice()`方法。

另请参阅

`GenerativeSelect.limit()`

`GenerativeSelect.offset()`

`GenerativeSelect.fetch()`

```py
method subquery(name: str | None = None) → Subquery
```

*继承自* `SelectBase.subquery()` *方法的* `SelectBase`

返回此 `SelectBase` 的子查询。

从 SQL 的角度来看，子查询是一种带有括号的命名构造，可以放置在另一个 SELECT 语句的 FROM 子句中。

给定如下 SELECT 语句：

```py
stmt = select(table.c.id, table.c.name)
```

上述语句看起来可能像这样：

```py
SELECT table.id, table.name FROM table
```

子查询本身以相同方式呈现，但是当嵌入到另一个 SELECT 语句的 FROM 子句中时，它就变成了一个命名子元素：

```py
subq = stmt.subquery()
new_stmt = select(subq)
```

上述呈现为：

```py
SELECT anon_1.id, anon_1.name
FROM (SELECT table.id, table.name FROM table) AS anon_1
```

历史上，`SelectBase.subquery()` 等同于在 FROM 对象上调用 `FromClause.alias()` 方法；然而，由于 `SelectBase` 对象不是直接的 FROM 对象，所以 `SelectBase.subquery()` 方法提供了更清晰的语义。

新版本中新增。

```py
method suffix_with(*suffixes: _TextCoercedExpressionArgument[Any], dialect: str = '*') → Self
```

*继承自* `HasSuffixes.suffix_with()` *方法的* `HasSuffixes`

在语句之后添加一个或多个表达式。

这用于支持在某些结构上的特定于后端的后缀关键字。

例如：

```py
stmt = select(col1, col2).cte().suffix_with(
    "cycle empno set y_cycle to 1 default 0", dialect="oracle")
```

可以通过多次调用`HasSuffixes.suffix_with()`来指定多个后缀。

参数：

+   `*suffixes` – 将在目标子句之后呈现的文本或`ClauseElement`构造。

+   `dialect` – 可选的字符串方言名称，将此后缀的渲染限制为仅限于该方言。

```py
method union(*other: _SelectStatementForCompoundArgument) → CompoundSelect
```

返回此 select() 构造相对于作为位置参数提供的给定可选择对象的 SQL `UNION`。

参数：

+   `*other` –

    用于创建 UNION 的一个或多个元素。

    从版本 1.4.28 开始更改：现在接受多个元素。

+   `**kwargs` – 关键字参数将转发到新创建的`CompoundSelect`对象的构造函数。

```py
method union_all(*other: _SelectStatementForCompoundArgument) → CompoundSelect
```

返回此 select() 构造相对于作为位置参数提供的给定可选择对象的 SQL `UNION ALL`。

参数：

+   `*other` –

    用于创建 UNION 的一个或多个元素。

    从版本 1.4.28 开始更改：现在接受多个元素。

+   `**kwargs` – 关键字参数将转发到新创建的`CompoundSelect`对象的构造函数。

```py
method where(*whereclause: _ColumnExpressionArgument[bool]) → Self
```

返回一个新的 `select()` 构造，其中给定的表达式添加到其 WHERE 子句中，并通过 AND 连接到现有子句（如果有）。

```py
attribute whereclause
```

返回此 `Select` 语句的完成 WHERE 子句。

将当前的 WHERE 条件集合装配成一个单个的 `BooleanClauseList` 构造。

新版本中新增。

```py
method with_for_update(*, nowait: bool = False, read: bool = False, of: _ForUpdateOfArgument | None = None, skip_locked: bool = False, key_share: bool = False) → Self
```

*继承自* `GenerativeSelect.with_for_update()` *方法的* `GenerativeSelect`

为此 `GenerativeSelect` 指定一个 `FOR UPDATE` 子句。

例如：

```py
stmt = select(table).with_for_update(nowait=True)
```

在像 PostgreSQL 或 Oracle 这样的数据库上，上述内容会呈现为如下语句：

```py
SELECT table.a, table.b FROM table FOR UPDATE NOWAIT
```

在其他后端上，`nowait` 选项被忽略，而会产生：

```py
SELECT table.a, table.b FROM table FOR UPDATE
```

当不带参数调用时，语句将以后缀 `FOR UPDATE` 呈现。然后可以提供其他参数，以允许常见的特定于数据库的变体。

参数：

+   `nowait` – 布尔值；将在 Oracle 和 PostgreSQL 方言上呈现 `FOR UPDATE NOWAIT`。

+   `read` – 布尔值；将在 MySQL 上呈现 `LOCK IN SHARE MODE`，在 PostgreSQL 上呈现 `FOR SHARE`。在 PostgreSQL 上，当与 `nowait` 结合使用时，将呈现 `FOR SHARE NOWAIT`。

+   `of` – SQL 表达式或 SQL 表达式元素的列表（通常是 `Column` 对象或兼容的表达式，对于某些后端也可能是表达式），它们将呈现为 `FOR UPDATE OF` 子句；由 PostgreSQL、Oracle、某些 MySQL 版本和可能其他后端支持。可能根据后端呈现为表或列。

+   `skip_locked` – 布尔值，将在 Oracle 和 PostgreSQL 方言上呈现 `FOR UPDATE SKIP LOCKED`，或者如果也指定了 `read=True`，则在 PostgreSQL 和 Oracle 方言上呈现 `FOR SHARE SKIP LOCKED`。

+   `key_share` – 布尔值，将在 PostgreSQL 方言上呈现 `FOR NO KEY UPDATE`，或者如果与 `read=True` 结合使用，则在 PostgreSQL 方言上呈现 `FOR KEY SHARE`。

```py
method with_hint(selectable: _FromClauseArgument, text: str, dialect_name: str = '*') → Self
```

*继承自* `HasHints.with_hint()` *方法的* `HasHints`

为给定的可选择项添加索引或其他执行上下文提示到此 `Select` 或其他可选择项对象。

提示文本会根据所使用的数据库后端，在相应的位置呈现，相对于传递给 `selectable` 参数的 `Table` 或 `Alias`。方言实现通常使用 Python 字符串替换语法，使用标记 `%(name)s` 来呈现表或别名的名称。例如，在使用 Oracle 时，以下内容：

```py
select(mytable).\
    with_hint(mytable, "index(%(name)s ix_mytable)")
```

会呈现 SQL 如下：

```py
select /*+ index(mytable ix_mytable) */ ... from mytable
```

`dialect_name` 选项将限制特定提示的呈现到特定的后端。例如，同时为 Oracle 和 Sybase 添加提示：

```py
select(mytable).\
    with_hint(mytable, "index(%(name)s ix_mytable)", 'oracle').\
    with_hint(mytable, "WITH INDEX ix_mytable", 'mssql')
```

另请参阅

`Select.with_statement_hint()`

```py
method with_only_columns(*entities: _ColumnsClauseArgument[Any], maintain_column_froms: bool = False, **_Select__kw: Any) → Select[Any]
```

返回一个新的 `select()` 构造，其列子句替换为给定的实体。

默认情况下，这个方法与原始的 `select()` 被调用时给���的实体完全等效。例如，一个语句：

```py
s = select(table1.c.a, table1.c.b)
s = s.with_only_columns(table1.c.b)
```

应该完全等效于:

```py
s = select(table1.c.b)
```

在这种操作模式下，如果没有明确说明，`Select.with_only_columns()` 也会动态修改语句的 FROM 子句。为了保留当前列子句隐含的现有 FROM 集合，包括那些暗示的列子句，添加 `Select.with_only_columns.maintain_column_froms` 参数：

```py
s = select(table1.c.a, table2.c.b)
s = s.with_only_columns(table1.c.a, maintain_column_froms=True)
```

上述参数将在列集合中的有效 FROM 转移到 `Select.select_from()` 方法中，就好像调用了以下内容：

```py
s = select(table1.c.a, table2.c.b)
s = s.select_from(table1, table2).with_only_columns(table1.c.a)
```

`Select.with_only_columns.maintain_column_froms` 参数利用了 `Select.columns_clause_froms` 集合，并执行等效于以下操作：

```py
s = select(table1.c.a, table2.c.b)
s = s.select_from(*s.columns_clause_froms).with_only_columns(table1.c.a)
```

参数：

+   `*entities` – 要使用的列表达式。

+   `maintain_column_froms` –

    一个布尔参数，将确保从当前列子句暗示的 FROM 列表首先传递给 `Select.select_from()` 方法。

    从版本 1.4.23 开始。

```py
method with_statement_hint(text: str, dialect_name: str = '*') → Self
```

*继承自* `HasHints.with_statement_hint()` *方法的* `HasHints`

为这个 `Select` 或其他可选择对象添加一个语句提示。

这个方法类似于 `Select.with_hint()`，但不需要单独的表，而是应用于整个语句。

这里的提示是特定于后端数据库的，可能包括隔离级别、文件指令、提取指令等。

另请参阅

`Select.with_hint()`

`Select.prefix_with()` - 通用的 SELECT 前缀，也可以适用于一些特定于数据库的 HINT 语法，如 MySQL 优化提示

```py
class sqlalchemy.sql.expression.Selectable
```

将一个类标记为可选择的。

**成员**

corresponding_column(), exported_columns, inherit_cache, is_derived_from(), lateral(), replace_selectable()

**类签名**

class `sqlalchemy.sql.expression.Selectable` (`sqlalchemy.sql.expression.ReturnsRows`)

```py
method corresponding_column(column: KeyedColumnElement[Any], require_embedded: bool = False) → KeyedColumnElement[Any] | None
```

给定一个`ColumnElement`，从此`Selectable.exported_columns`的集合中返回与该原始`ColumnElement`通过共同祖先列相对应的导出`ColumnElement`对象。

参数：

+   `column` - 要匹配的目标`ColumnElement`。

+   `require_embedded` - 仅返回给定的`ColumnElement`对应的列，如果给定的`ColumnElement`实际上存在于此`Selectable`的子元素中。通常，如果列仅与此`Selectable`的导出列之一共享共同的祖先，则列将匹配。

另请参阅

`Selectable.exported_columns` - 用于操作的`ColumnCollection`。

`ColumnCollection.corresponding_column()` - 实现方法。

```py
attribute exported_columns
```

*继承自* `ReturnsRows.exported_columns` *属性*，*的* `ReturnsRows`

一个 `ColumnCollection` 代表了这个 `ReturnsRows` 的“导出”列。

“导出”列代表了这个 SQL 结构所呈现的 `ColumnElement` 表达式的集合。有主要的变体是 FROM 子句的“FROM 子句列”，例如表、连接或子查询，被“SELECT”列选中，这些列是 SELECT 语句的“columns clause”中的列，并且在 DML 语句中的 RETURNING 列。

1.4 版中的新内容。

另请参阅

`FromClause.exported_columns`

`SelectBase.exported_columns`

```py
attribute inherit_cache: bool | None = None
```

*继承自* `HasCacheKey` *的* `HasCacheKey.inherit_cache` *属性*。

指示此 `HasCacheKey` 实例是否应该使用其直接超类使用的缓存键生成方案。

该属性默认为 `None`，表示构造尚未考虑它是否适合参与缓存；这在功能上等同于将值设置为 `False`，只是还会发出警告。

如果对象对应的 SQL 不会根据本类而不是其超类的属性而改变，那么可以将此标志设置为 `True`。

另请参阅

为自定义结构启用缓存支持 - 为第三方或用户定义的 SQL 结构设置 `HasCacheKey.inherit_cache` 属性的一般指南。

```py
method is_derived_from(fromclause: FromClause | None) → bool
```

*继承自* `ReturnsRows.is_derived_from()` *方法*，*的* `ReturnsRows`

如果此`ReturnsRows`是从给定的`FromClause`“派生”出来，则返回`True`。

一个示例是一个表的别名是从该表派生的。

```py
method lateral(name: str | None = None) → LateralFromClause
```

返回此`Selectable`的 LATERAL 别名。

返回值是顶层`lateral()`函数提供的`Lateral`构造。

另请参阅

LATERAL 相关性 - 用法概述。

```py
method replace_selectable(old: FromClause, alias: Alias) → Self
```

用给定的`Alias`对象替换所有`FromClause`‘old’的所有出现，返回此`FromClause`的副本。

自版本 1.4 起弃用：`Selectable.replace_selectable()`方法已弃用，并将在将来的版本中删除。类似功能可通过 sqlalchemy.sql.visitors 模块获得。

```py
class sqlalchemy.sql.expression.SelectBase
```

SELECT 语句的基类。

这包括`Select`、`CompoundSelect`和`TextualSelect`。

**成员**

add_cte(), alias(), as_scalar(), c, corresponding_column(), cte(), exists(), exported_columns, get_label_style(), inherit_cache, is_derived_from(), label(), lateral(), replace_selectable(), scalar_subquery(), select(), selected_columns, set_label_style(), subquery()

**类签名**

类 `sqlalchemy.sql.expression.SelectBase` (`sqlalchemy.sql.roles.SelectStatementRole`, `sqlalchemy.sql.roles.DMLSelectRole`, `sqlalchemy.sql.roles.CompoundElementRole`, `sqlalchemy.sql.roles.InElementRole`, `sqlalchemy.sql.expression.HasCTE`, `sqlalchemy.sql.annotation.SupportsCloneAnnotations`, `sqlalchemy.sql.expression.Selectable`)

```py
method add_cte(*ctes: CTE, nest_here: bool = False) → Self
```

*继承自* `HasCTE.add_cte()` *方法的* `HasCTE`

向此语句添加一个或多个 `CTE` 构造。

此方法将给定的 `CTE` 构造与父语句关联，以便它们将在最终语句的 WITH 子句中无条件地呈现，即使在语句或任何子选择中没有其他地方引用它们。

可选的 `HasCTE.add_cte.nest_here` 参数设置为 True 时，每个给定的 `CTE` 将以一个 WITH 子句的形式直接与此语句一起呈现，而不是被移动到最终呈现语句的顶部，即使此语句作为一个子查询在较大的语句中被呈现。

此方法有两个通用用途。一个是嵌入一些没有被显式引用的 CTE 语句，比如将 DML 语句（比如 INSERT 或 UPDATE）作为 CTE 内联到一个主要语句中，这个主要语句可能间接地引用其结果。另一个是提供对一系列 CTE 构造的确切放置位置的控制，这些构造应该保持直接在一个特定语句中呈现，而这个语句可能嵌套在一个较大的语句中。

例如：

```py
from sqlalchemy import table, column, select
t = table('t', column('c1'), column('c2'))

ins = t.insert().values({"c1": "x", "c2": "y"}).cte()

stmt = select(t).add_cte(ins)
```

将呈现为：

```py
WITH anon_1 AS
(INSERT INTO t (c1, c2) VALUES (:param_1, :param_2))
SELECT t.c1, t.c2
FROM t
```

上面，“anon_1” CTE 在 SELECT 语句中没有被引用，但仍然完成了运行 INSERT 语句的任务。

类似地，在与 DML 相关的上下文中，使用 PostgreSQL 的 `Insert` 构造来生成“upsert”：

```py
from sqlalchemy import table, column
from sqlalchemy.dialects.postgresql import insert

t = table("t", column("c1"), column("c2"))

delete_statement_cte = (
    t.delete().where(t.c.c1 < 1).cte("deletions")
)

insert_stmt = insert(t).values({"c1": 1, "c2": 2})
update_statement = insert_stmt.on_conflict_do_update(
    index_elements=[t.c.c1],
    set_={
        "c1": insert_stmt.excluded.c1,
        "c2": insert_stmt.excluded.c2,
    },
).add_cte(delete_statement_cte)

print(update_statement)
```

上面的语句呈现为：

```py
WITH deletions AS
(DELETE FROM t WHERE t.c1 < %(c1_1)s)
INSERT INTO t (c1, c2) VALUES (%(c1)s, %(c2)s)
ON CONFLICT (c1) DO UPDATE SET c1 = excluded.c1, c2 = excluded.c2
```

1.4.21 版本中新增。

参数：

+   `*ctes` -

    零个或多个 `CTE` 构造。

    从 2.0 版本开始更改：接受多个 CTE 实例

+   `nest_here` -

    如果为 True，则给定的 CTE 或 CTEs 将被呈现为如果它们在被添加到此 `HasCTE` 时指定了 `HasCTE.cte.nesting` 标志为 `True`。假设给定的 CTEs 在外部封闭语句中也没有被引用，那么当给出此标志时，给定的 CTEs 应该在此语句的级别呈现。

    2.0 版本中新增。

    参见

    `HasCTE.cte.nesting`

```py
method alias(name: str | None = None, flat: bool = False) → Subquery
```

返回针对此 `SelectBase` 的命名子查询。

对于 `SelectBase`（与 `FromClause` 相对），这将返回一个 `Subquery` 对象，其行为与与 `FromClause` 一起使用的 `Alias` 对象基本相同。

在 1.4 版本中更改：`SelectBase.alias()` 方法现在是 `SelectBase.subquery()` 方法的同义词。

```py
method as_scalar() → ScalarSelect[Any]
```

自 1.4 版本起已弃用：`SelectBase.as_scalar()` 方法已弃用，并将在以后的版本中移除。请参考 `SelectBase.scalar_subquery()`。

```py
attribute c
```

自 1.4 版本起已弃用：`SelectBase.c` 和 `SelectBase.columns` 属性已弃用，并将在以后的版本中移除；这些属性隐式创建一个应该明确的子查询。请首先调用 `SelectBase.subquery()` 来创建一个子查询，然后再使用此属性。要访问此 SELECT 对象从中选择的列，请使用 `SelectBase.selected_columns` 属性。

```py
method corresponding_column(column: KeyedColumnElement[Any], require_embedded: bool = False) → KeyedColumnElement[Any] | None
```

*继承自* `Selectable.corresponding_column()` *方法的* `Selectable` *对象*

给定一个 `ColumnElement`，返回此 `Selectable` 的导出 `ColumnElement` 对象，该对象通过共同的祖先列与原始 `ColumnElement` 对应。

参数：

+   `column` – 要匹配的目标 `ColumnElement`。

+   `require_embedded` – 仅返回给定 `ColumnElement` 的相应列，如果给定的 `ColumnElement` 实际上存在于此 `Selectable` 的子元素中。通常情况下，如果该列仅仅与此 `Selectable` 的导出列之一共享共同的祖先，则列将匹配。

另请参阅

`Selectable.exported_columns` - 用于操作的 `ColumnCollection`。

`ColumnCollection.corresponding_column()` - 实现方法。

```py
method cte(name: str | None = None, recursive: bool = False, nesting: bool = False) → CTE
```

*继承自* `HasCTE.cte()` *方法的* `HasCTE`

返回一个新的 `CTE`，或通用表达式实例。

通用表达式（Common table expressions）是 SQL 标准的一部分，SELECT 语句可以在主语句的基础上引用指定的辅助语句，使用一个叫做“WITH”的子句。特殊的关于 UNION 的语义也可以被采用，以允许“递归”查询，其中一个 SELECT 语句可以引用之前已经被选定的行的集合。

CTEs 也可以应用于 DML 构造 UPDATE、INSERT 和 DELETE 在一些数据库上，既作为与 RETURNING 结合使用时的 CTE 行的来源，也作为 CTE 行的消费者。

SQLAlchemy 检测到 `CTE` 对象，它们被视为类似于 `Alias` 对象，作为特殊元素交付给语句的 FROM 子句以及语句顶部的 WITH 子句。

对于像 PostgreSQL 的“MATERIALIZED”和“NOT MATERIALIZED”这样的特殊前缀，可以使用 `CTE.prefix_with()` 方法来建立这些前缀。

版本 1.3.13 中的更改：增加了对前缀的支持。具体来说 - MATERIALIZED 和 NOT MATERIALIZED。

参数：

+   `name` – 给通用表达式的名称。像 `FromClause.alias()` 一样，名称可以保持为 `None`，在这种情况下，将在查询编译时使用一个匿名符号。

+   `recursive` – 如果`True`，将呈现`WITH RECURSIVE`。递归公共表达式旨在与 UNION ALL 结合使用，以从已选择的行中派生行。

+   `nesting` –

    如果`True`，将在引用它的语句中本地呈现 CTE。对于更复杂的情况，也可以使用`HasCTE.add_cte()`方法，使用`HasCTE.add_cte.nest_here`参数更精确地控制特定 CTE 的确切放置。

    新版本 1.4.24。

    另请参阅

    `HasCTE.add_cte()`

以下示例包括两个来自 PostgreSQL 文档的示例，网址为 [`www.postgresql.org/docs/current/static/queries-with.html`](https://www.postgresql.org/docs/current/static/queries-with.html)，以及其他示例。

示例 1，非递归：

```py
from sqlalchemy import (Table, Column, String, Integer,
                        MetaData, select, func)

metadata = MetaData()

orders = Table('orders', metadata,
    Column('region', String),
    Column('amount', Integer),
    Column('product', String),
    Column('quantity', Integer)
)

regional_sales = select(
                    orders.c.region,
                    func.sum(orders.c.amount).label('total_sales')
                ).group_by(orders.c.region).cte("regional_sales")

top_regions = select(regional_sales.c.region).\
        where(
            regional_sales.c.total_sales >
            select(
                func.sum(regional_sales.c.total_sales) / 10
            )
        ).cte("top_regions")

statement = select(
            orders.c.region,
            orders.c.product,
            func.sum(orders.c.quantity).label("product_units"),
            func.sum(orders.c.amount).label("product_sales")
    ).where(orders.c.region.in_(
        select(top_regions.c.region)
    )).group_by(orders.c.region, orders.c.product)

result = conn.execute(statement).fetchall()
```

示例 2，WITH RECURSIVE：

```py
from sqlalchemy import (Table, Column, String, Integer,
                        MetaData, select, func)

metadata = MetaData()

parts = Table('parts', metadata,
    Column('part', String),
    Column('sub_part', String),
    Column('quantity', Integer),
)

included_parts = select(\
    parts.c.sub_part, parts.c.part, parts.c.quantity\
    ).\
    where(parts.c.part=='our part').\
    cte(recursive=True)

incl_alias = included_parts.alias()
parts_alias = parts.alias()
included_parts = included_parts.union_all(
    select(
        parts_alias.c.sub_part,
        parts_alias.c.part,
        parts_alias.c.quantity
    ).\
    where(parts_alias.c.part==incl_alias.c.sub_part)
)

statement = select(
            included_parts.c.sub_part,
            func.sum(included_parts.c.quantity).
              label('total_quantity')
        ).\
        group_by(included_parts.c.sub_part)

result = conn.execute(statement).fetchall()
```

示例 3，使用 UPDATE 和 INSERT 进行 upsert 与 CTEs：

```py
from datetime import date
from sqlalchemy import (MetaData, Table, Column, Integer,
                        Date, select, literal, and_, exists)

metadata = MetaData()

visitors = Table('visitors', metadata,
    Column('product_id', Integer, primary_key=True),
    Column('date', Date, primary_key=True),
    Column('count', Integer),
)

# add 5 visitors for the product_id == 1
product_id = 1
day = date.today()
count = 5

update_cte = (
    visitors.update()
    .where(and_(visitors.c.product_id == product_id,
                visitors.c.date == day))
    .values(count=visitors.c.count + count)
    .returning(literal(1))
    .cte('update_cte')
)

upsert = visitors.insert().from_select(
    [visitors.c.product_id, visitors.c.date, visitors.c.count],
    select(literal(product_id), literal(day), literal(count))
        .where(~exists(update_cte.select()))
)

connection.execute(upsert)
```

示例 4���嵌套 CTE（SQLAlchemy 1.4.24 及以上版本）：

```py
value_a = select(
    literal("root").label("n")
).cte("value_a")

# A nested CTE with the same name as the root one
value_a_nested = select(
    literal("nesting").label("n")
).cte("value_a", nesting=True)

# Nesting CTEs takes ascendency locally
# over the CTEs at a higher level
value_b = select(value_a_nested.c.n).cte("value_b")

value_ab = select(value_a.c.n.label("a"), value_b.c.n.label("b"))
```

上述查询将呈现第二个 CTE 嵌套在第一个内部，如下所示：

```py
WITH
    value_a AS
        (SELECT 'root' AS n),
    value_b AS
        (WITH value_a AS
            (SELECT 'nesting' AS n)
        SELECT value_a.n AS n FROM value_a)
SELECT value_a.n AS a, value_b.n AS b
FROM value_a, value_b
```

可以使用以下方式设置相同的 CTE，使用`HasCTE.add_cte()`方法（SQLAlchemy 2.0 及以上版本）：

```py
value_a = select(
    literal("root").label("n")
).cte("value_a")

# A nested CTE with the same name as the root one
value_a_nested = select(
    literal("nesting").label("n")
).cte("value_a")

# Nesting CTEs takes ascendency locally
# over the CTEs at a higher level
value_b = (
    select(value_a_nested.c.n).
    add_cte(value_a_nested, nest_here=True).
    cte("value_b")
)

value_ab = select(value_a.c.n.label("a"), value_b.c.n.label("b"))
```

示例 5，非线性 CTE（SQLAlchemy 1.4.28 及以上版本）：

```py
edge = Table(
    "edge",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("left", Integer),
    Column("right", Integer),
)

root_node = select(literal(1).label("node")).cte(
    "nodes", recursive=True
)

left_edge = select(edge.c.left).join(
    root_node, edge.c.right == root_node.c.node
)
right_edge = select(edge.c.right).join(
    root_node, edge.c.left == root_node.c.node
)

subgraph_cte = root_node.union(left_edge, right_edge)

subgraph = select(subgraph_cte)
```

上述查询将在递归 CTE 内呈现 2 个 UNION：

```py
WITH RECURSIVE nodes(node) AS (
        SELECT 1 AS node
    UNION
        SELECT edge."left" AS "left"
        FROM edge JOIN nodes ON edge."right" = nodes.node
    UNION
        SELECT edge."right" AS "right"
        FROM edge JOIN nodes ON edge."left" = nodes.node
)
SELECT nodes.node FROM nodes
```

另请参阅

`Query.cte()` - `HasCTE.cte()` 的 ORM 版本。

```py
method exists() → Exists
```

返回此可选择的`Exists`表示，可用作列表达式。

返回的对象是`Exists`的实例。

另请参阅

`exists()`

EXISTS 子查询 - 在 2.0 风格教程中。

新版本 1.4。

```py
attribute exported_columns
```

一个`ColumnCollection`，表示此`Selectable`的“导出”列，不包括`TextClause`构造。

`SelectBase` 对象的“导出”列与 `SelectBase.selected_columns` 集合是同义词。

版本 1.4 中的新功能。

另请参阅

`Select.exported_columns`

`Selectable.exported_columns`

`FromClause.exported_columns`

```py
method get_label_style() → SelectLabelStyle
```

检索当前标签样式。

由子类实现。

```py
attribute inherit_cache: bool | None = None
```

*继承自* `HasCacheKey` *的* `HasCacheKey.inherit_cache` *属性。

指示此 `HasCacheKey` 实例是否应使用其直接超类使用的缓存键生成方案。

该属性默认为`None`，表示构造尚未考虑其是否适合参与缓存；这在功能上等同于将值设置为`False`，只是还会发出警告。

如果与对象对应的 SQL 不会根据本类而不是其超类的属性而改变，则可以在特定类上将此标志设置为`True`。

另请参阅

为自定义构造启用缓存支持 - 设置第三方或用户定义的 SQL 构造的 `HasCacheKey.inherit_cache` 属性的一般指南。

```py
method is_derived_from(fromclause: FromClause | None) → bool
```

*继承自* `ReturnsRows` *的* `ReturnsRows.is_derived_from()` *方法。

如果此 `ReturnsRows` 是从给定的 `FromClause` ‘派生’，则返回`True`。

一个示例是，表的别名是从该表派生的。

```py
method label(name: str | None) → Label[Any]
```

返回此可选择项的“标量”表示，嵌入为带有标签的子查询。

另请参阅

`SelectBase.scalar_subquery()`。

```py
method lateral(name: str | None = None) → LateralFromClause
```

返回此 `Selectable` 的 LATERAL 别名。

返回值也是顶级`lateral()`函数提供的`Lateral`构造。

请参见

横向相关 - 用法概述。

```py
method replace_selectable(old: FromClause, alias: Alias) → Self
```

*继承自* `Selectable.replace_selectable()` *方法的* `Selectable`

将所有`FromClause` ‘old’的出现替换为给定的`Alias`对象，返回此`FromClause`的副本。

自版本 1.4 起不推荐使用：`Selectable.replace_selectable()` 方法已弃用，并将在将来的版本中删除。类似的功能可通过 sqlalchemy.sql.visitors 模块使用。

```py
method scalar_subquery() → ScalarSelect[Any]
```

返回可用作列表达式的此可选择性的“标量”表示。

返回的对象是`ScalarSelect`的一个实例。

通常，在其列子句中只有一个列的 select 语句有资格用作标量表达式。然后可以在封闭 SELECT 的 WHERE 子句或列子句中使用标量子查询。

请注意，标量子查询与使用`SelectBase.subquery()` 方法生成的 FROM 级子查询不同。

请参见

标量和相关子查询 - 在 2.0 教程中

```py
method select(*arg: Any, **kw: Any) → Select
```

自版本 1.4 起不推荐使用：`SelectBase.select()` 方法已弃用，并将在将来的版本中删除；此方法隐式创建应明确的子查询。请先调用 `SelectBase.subquery()` 以创建子查询，然后可以选择该子查询。

```py
attribute selected_columns
```

表示此 SELECT 语句或类似构造返回的结果集中的列的 `ColumnCollection`。

该集合与 `FromClause.columns` 集合不同之处在于，该集合中的列不能直接嵌套在另一个 SELECT 语句中；必须首先应用一个子查询，该子查询提供了 SQL 所需的必要括号。

注意

`SelectBase.selected_columns` 集合不包括使用 `text()` 构造在列子句中建立的表达式；这些表达式会被默默地从集合中省略掉。要在 `Select` 构造内使用纯文本列表达式，请使用 `literal_column()` 构造。

另请参阅

`Select.selected_columns`

版本 1.4 中新增。

```py
method set_label_style(style: SelectLabelStyle) → Self
```

使用指定的标签样式返回新的可选择对象。

由子类实现。

```py
method subquery(name: str | None = None) → Subquery
```

返回此 `SelectBase` 的子查询。

从 SQL 的角度来看，子查询是一种带有名称的括号括起来的构造，在另一个 SELECT 语句的 FROM 子句中可以放置。

给定如下 SELECT 语句：

```py
stmt = select(table.c.id, table.c.name)
```

上述语句可能看起来像这样：

```py
SELECT table.id, table.name FROM table
```

单独的子查询形式呈现方式相同，但是当嵌入到另一个 SELECT 语句的 FROM 子句中时，它就成为一个命名的子元素：

```py
subq = stmt.subquery()
new_stmt = select(subq)
```

上述呈现为：

```py
SELECT anon_1.id, anon_1.name
FROM (SELECT table.id, table.name FROM table) AS anon_1
```

从历史上看，`SelectBase.subquery()` 相当于在 FROM 对象上调用 `FromClause.alias()` 方法；然而，由于 `SelectBase` 对象不是直接的 FROM 对象，所以 `SelectBase.subquery()` 方法提供了更清晰的语义。

版本 1.4 中新增。

```py
class sqlalchemy.sql.expression.Subquery
```

表示一个 SELECT 的子查询。

通过在任何包含 `Select`, `CompoundSelect`, 和 `TextualSelect` 的 `SelectBase` 子类上调用 `SelectBase.subquery()` 方法或为方便起见调用 `SelectBase.alias()` 方法来创建 `Subquery`。在 FROM 子句中表示 SELECT 语句的主体部分，后面跟着通常的“AS <somename>”，定义了所有“别名”对象。

`Subquery` 对象与 `Alias` 对象非常相似，并且可以以等效的方式使用。`Alias` 和 `Subquery` 之间的区别在于 `Alias` 始终包含一个 `FromClause` 对象，而 `Subquery` 始终包含一个 `SelectBase` 对象。

新版本 1.4 中：添加了 `Subquery` 类，该类现在用于提供 SELECT 语句的别名版本。

**成员**

as_scalar(), inherit_cache

**类签名**

类 `sqlalchemy.sql.expression.Subquery` (`sqlalchemy.sql.expression.AliasedReturnsRows`)

```py
method as_scalar() → ScalarSelect[Any]
```

自版本 1.4 起已弃用：`Subquery.as_scalar()`方法，在版本 1.4 之前曾是`Alias.as_scalar()`，已弃用并将在将来的版本中删除；请在构造子查询对象之前使用`Select.scalar_subquery()`方法的`select()`构造，或者在 ORM 中使用`Query.scalar_subquery()`方法。

```py
attribute inherit_cache: bool | None = True
```

指示此`HasCacheKey`实例是否应使用其直接超类使用的缓存键生成方案。

该属性默认为`None`，表示构造尚未考虑是否适合参与缓存；这在功能上等同于将值设置为`False`，只是还会发出警告。

如果与对象对应的 SQL 不基于此类的本地属性而是其超类，则可以在特定类上将此标志设置为`True`。

另请参见

为自定义构造启用缓存支持 - 设置第三方或用户定义的 SQL 构造的`HasCacheKey.inherit_cache`属性的一般指南。

```py
class sqlalchemy.sql.expression.TableClause
```

表示一个最小的“表”构造。

这是一个轻量级的表对象，只有一个名称、一组列（通常由`column()`函数生成），以及一个模式：

```py
from sqlalchemy import table, column

user = table("user",
        column("id"),
        column("name"),
        column("description"),
)
```

`TableClause`构造用作更常用的`Table`对象的基础，提供通常的`FromClause`服务，包括`.c.`集合和语句生成方法。

它**不**提供`Table`的所有附加模式级服务，包括约束、对其他表的引用，或者对`MetaData`级别服务的支持。它本身很有用，作为一种临时构造，用于在没有更完整的`Table`的情况下生成快速的 SQL 语句。

**成员**

alias(), c, columns, compare(), compile(), corresponding_column(), delete(), description, entity_namespace, exported_columns, foreign_keys, get_children(), implicit_returning, inherit_cache, insert(), is_derived_from(), join(), lateral(), outerjoin(), params(), primary_key, replace_selectable(), schema, select(), self_group(), table_valued(), tablesample(), unique_params(), update()

**类签名**

class `sqlalchemy.sql.expression.TableClause` (`sqlalchemy.sql.roles.DMLTableRole`, `sqlalchemy.sql.expression.Immutable`, `sqlalchemy.sql.expression.NamedFromClause`)

```py
method alias(name: str | None = None, flat: bool = False) → NamedFromClause
```

*继承自* `FromClause.alias()` *方法的* `FromClause`

返回此`FromClause`的别名。

例如：

```py
a2 = some_table.alias('a2')
```

上述代码创建了一个`Alias`对象，可以在任何 SELECT 语句中用作 FROM 子句。

请参阅

使用别名

`alias()`

```py
attribute c
```

*继承自* `FromClause.c` *属性的* `FromClause`

`FromClause.columns`的同义词

返回：

一个`ColumnCollection`

```py
attribute columns
```

*继承自* `FromClause.columns` *属性的* `FromClause`

由此`FromClause`维护的基于名称的`ColumnElement`对象的命名集合。

`columns`或`c`集合是使用绑定到表或其他可选择列构建 SQL 表达式的入口：

```py
select(mytable).where(mytable.c.somecolumn == 5)
```

返回：

一个`ColumnCollection`对象。

```py
method compare(other: ClauseElement, **kw: Any) → bool
```

*继承自* `ClauseElement.compare()` *方法的* `ClauseElement`

将此`ClauseElement`与给定的`ClauseElement`进行比较。

子类应该重写默认行为，即直接的身份比较。

**kw 是子类`compare()`方法消耗的参数，可以用来修改比较的标准（参见`ColumnElement`）。

```py
method compile(bind: _HasDialect | None = None, dialect: Dialect | None = None, **kw: Any) → Compiled
```

*继承自* `CompilerElement.compile()` *方法的* `CompilerElement`

编译此 SQL 表达式。

返回值是一个`Compiled`对象。对返回值调用`str()`或`unicode()`将产生结果的字符串表示。`Compiled`对象还可以使用`params`访问器返回绑定参数名称和值的字典。

参数：

+   `bind` – 一个`Connection`或`Engine`，可以提供一个`Dialect`以生成一个`Compiled`对象。如果`bind`和`dialect`参数都被省略，则使用默认的 SQL 编译器。

+   `column_keys` – 用于 INSERT 和 UPDATE 语句，一个列名列表，应该出现在编译语句的 VALUES 子句中。如果为`None`，则渲染来自目标表对象的所有列。

+   `dialect` – 一个`Dialect`实例，可以生成一个`Compiled`对象。此参数优先于`bind`参数。

+   `compile_kwargs` –

    可选的附加参数字典，将传递给所有“visit”方法中的编译器。这允许将任何自定义标志传递给自定义编译结构，例如。它也用于通过传递`literal_binds`标志的情况：

    ```py
    from sqlalchemy.sql import table, column, select

    t = table('t', column('x'))

    s = select(t).where(t.c.x == 5)

    print(s.compile(compile_kwargs={"literal_binds": True}))
    ```

另请参阅

如何将 SQL 表达式渲染为字符串，可能包含内联的绑定参数？

```py
method corresponding_column(column: KeyedColumnElement[Any], require_embedded: bool = False) → KeyedColumnElement[Any] | None
```

*从* `Selectable.corresponding_column()` *方法继承* `Selectable`

给定一个`ColumnElement`，从此`Selectable`的`Selectable.exported_columns`集合中返回导出的`ColumnElement`对象，该对象对应于通过公共祖先列对应于该原始`ColumnElement`。

参数：

+   `column` – 要匹配的目标`ColumnElement`。

+   `require_embedded` – 仅返回给定`ColumnElement`的相应列，如果给定的`ColumnElement`实际上存在于此`Selectable`的子元素中。通常，如果列仅与此`Selectable`的导出列之一共享共同的祖先，则列将匹配。

另请参阅

`Selectable.exported_columns` - 用于操作的`ColumnCollection`。

`ColumnCollection.corresponding_column()` - 实现方法。

```py
method delete() → Delete
```

生成针对此`TableClause`的`delete()`构造。

例如：

```py
table.delete().where(table.c.id==7)
```

查看`delete()`以获取参数和用法信息。

```py
attribute description
```

```py
attribute entity_namespace
```

*继承自* `FromClause.entity_namespace` *属性的* `FromClause`

返回用于在 SQL 表达式中进行基于名称访问的命名空间。

这是用于解析“filter_by()”类型表达式的命名空间，例如：

```py
stmt.filter_by(address='some address')
```

默认为`.c`集合，但在内部可以使用“entity_namespace”注释进行覆盖以提供替代结果。

```py
attribute exported_columns
```

*继承自* `FromClause.exported_columns` *属性的* `FromClause`

代表此`Selectable`的“导出”列的`ColumnCollection`。

`FromClause`对象的“导出”列与`FromClause.columns`集合是同义词。

版本 1.4 中的新功能。

另请参阅

`Selectable.exported_columns`

`SelectBase.exported_columns`

```py
attribute foreign_keys
```

*继承自* `FromClause.foreign_keys` *属性的* `FromClause`

返回此 FromClause 引用的`ForeignKey`标记对象的集合。

每个`ForeignKey`都是`Table`范围内的一个`ForeignKeyConstraint`的成员。

另请参阅

`Table.foreign_key_constraints`

```py
method get_children(*, omit_attrs: Tuple[str, ...] = (), **kw: Any) → Iterable[HasTraverseInternals]
```

*继承自* `HasTraverseInternals.get_children()` *方法的* `HasTraverseInternals`

返回此`HasTraverseInternals`的直接子`HasTraverseInternals`元素。

用于访问遍历。

**kw 可能包含改变返回集合的标志，例如返回子项的子集以减少更大的遍历，或者从不同的上下文（例如模式级别的集合而不是子句级别）返回子项。

```py
attribute implicit_returning = False
```

`TableClause`不支持具有主键或列级默认值，因此隐式返回不适用。

```py
attribute inherit_cache: bool | None = None
```

*继承自* `HasCacheKey.inherit_cache` *属性的* `HasCacheKey`

指示此`HasCacheKey`实例是否应使用其直接超类使用的缓存键生成方案。

该属性默认为`None`，表示构造尚未考虑其是否适合参与缓存；这在功能上等效于将值设置为`False`，除了还会发出警告。

如果对象对应的 SQL 不会根据本类本地属性而变化，并且不是其超类，则可以在特定类上设置此标志为`True`。

另请参阅

为自定义构造启用缓存支持 - 设置第三方或用户定义的 SQL 构造的 `HasCacheKey.inherit_cache` 属性的一般指南。

```py
method insert() → Insert
```

生成一个针对这个`TableClause`的 `Insert` 构造。

例如：

```py
table.insert().values(name='foo')
```

有关参数和用法信息，请参阅 `insert()`。

```py
method is_derived_from(fromclause: FromClause | None) → bool
```

*继承自* `FromClause.is_derived_from()` *方法于* `FromClause`

如果这个`FromClause`是从给定的`FromClause`‘派生’，则返回`True`。

一个示例是表的别名是从该表派生的。

```py
method join(right: _FromClauseArgument, onclause: _ColumnExpressionArgument[bool] | None = None, isouter: bool = False, full: bool = False) → Join
```

*继承自* `FromClause.join()` *方法于* `FromClause`

从这个`FromClause`返回一个`Join`到另一个`FromClause`。

例如：

```py
from sqlalchemy import join

j = user_table.join(address_table,
                user_table.c.id == address_table.c.user_id)
stmt = select(user_table).select_from(j)
```

会发出类似以下的 SQL 语句：

```py
SELECT user.id, user.name FROM user
JOIN address ON user.id = address.user_id
```

参数：

+   `right` – 连接的右侧；这是任何 `FromClause` 对象，比如一个 `Table` 对象，也可以是一个可选择的兼容对象，比如一个 ORM 映射类。

+   `onclause` – 表示连接的 ON 子句的 SQL 表达式。如果保留为 `None`，`FromClause.join()` 将尝试基于外键关系连接两个表。

+   `isouter` – 如果为 True，则渲染一个 LEFT OUTER JOIN，而不是 JOIN。

+   `full` – 如果为 True，则渲染一个 FULL OUTER JOIN，而不是 LEFT OUTER JOIN。暗示 `FromClause.join.isouter`。

另见

`join()` - 独立的函数

`Join` - 生成对象的类型

```py
method lateral(name: str | None = None) → LateralFromClause
```

*继承自* `Selectable` 的 `Selectable.lateral()` *方法*

返回此 `Selectable` 的 LATERAL 别名。

返回值是顶层 `lateral()` 函数提供的 `Lateral` 构造。

另请参见

LATERAL 关联 - 用法概述。

```py
method outerjoin(right: _FromClauseArgument, onclause: _ColumnExpressionArgument[bool] | None = None, full: bool = False) → Join
```

*继承自* `FromClause` 的 `FromClause.outerjoin()` *方法*

返回一个从此 `FromClause` 到另一个 `FromClause` 的 `Join`，并将“isouter”标志设置为 True。

例如：

```py
from sqlalchemy import outerjoin

j = user_table.outerjoin(address_table,
                user_table.c.id == address_table.c.user_id)
```

以上等同于：

```py
j = user_table.join(
    address_table,
    user_table.c.id == address_table.c.user_id,
    isouter=True)
```

参数：

+   `right` – 连接的右侧；这是任何 `FromClause` 对象，比如一个 `Table` 对象，也可以是一个可选择兼容对象，比如一个 ORM 映射类。

+   `onclause` – 表示连接的 ON 子句的 SQL 表达式。如果保留为 `None`，`FromClause.join()` 将尝试基于外键关系连接这两个表。

+   `full` – 如果为 True，则渲染 FULL OUTER JOIN，而不是 LEFT OUTER JOIN。

另请参见

`FromClause.join()`

`Join`

```py
method params(*optionaldict, **kwargs)
```

*继承自* `Immutable` 的 `Immutable.params()` *方法*

返回一个替换了 `bindparam()` 元素的副本。

返回此 ClauseElement 的副本，其中的 `bindparam()` 元素替换为从给定字典中取出的值：

```py
>>> clause = column('x') + bindparam('foo')
>>> print(clause.compile().params)
{'foo':None}
>>> print(clause.params({'foo':7}).compile().params)
{'foo':7}
```

```py
attribute primary_key
```

*继承自* `FromClause` 的 `FromClause.primary_key` *属性*

返回由此 `_selectable.FromClause` 的主键组成的 `Column` 对象的可迭代集合。

对于 `Table` 对象，此集合由 `PrimaryKeyConstraint` 表示，它本身是一个 `Column` 对象的可迭代集合。

```py
method replace_selectable(old: FromClause, alias: Alias) → Self
```

*继承自* `Selectable.replace_selectable()` *方法的* `Selectable`

用给定的 `Alias` 对象替换所有 `FromClause` 中的‘old’，返回此 `FromClause` 的副本。

自版本 1.4 起已弃用：`Selectable.replace_selectable()` 方法已弃用，并将在将来的版本中移除。类似功能可通过 sqlalchemy.sql.visitors 模块实现。

```py
attribute schema: str | None = None
```

*继承自* `FromClause.schema` *属性的* `FromClause`

为此 `FromClause` 定义‘schema’属性。

对于大多数对象来说，这通常是`None`，除了 `Table`，在这种情况下，它被视为 `Table.schema` 参数的值。

```py
method select() → Select
```

*继承自* `FromClause.select()` *方法的* `FromClause`

返回此 `FromClause` 的 SELECT。

例如：

```py
stmt = some_table.select().where(some_table.c.id == 5)
```

另请参阅

`select()` - 允许任意列列表的通用方法。

```py
method self_group(against: OperatorType | None = None) → ClauseElement
```

*继承自* `ClauseElement.self_group()` *方法的* `ClauseElement`

对此`ClauseElement`应用一个‘分组’。

此方法被子类重写以返回一个“分组”构造，即括号。特别是当“二进制”表达式被放置到更大的表达式中时，它们用于在自身周围提供分组，以及当`select()`构造被放置到另一个`select()`的 FROM 子句中时。(请注意，子查询通常应使用`Select.alias()`方法创建，因为许多平台要求嵌套的 SELECT 语句必须命名)。

随着表达式的组合，自动应用`self_group()` - 最终用户代码不应直接使用此方法。请注意，SQLAlchemy 的子句构造考虑了操作符优先级 - 因此，例如，在表达式`x OR (y AND z)`中可能不需要括号 - AND 优先于 OR。

`ClauseElement`的基本`self_group()`方法只返回自身。

```py
method table_valued() → TableValuedColumn[Any]
```

*继承自* `NamedFromClause`的`NamedFromClause.table_valued()` *方法。

为此`FromClause`返回一个`TableValuedColumn`对象。

`TableValuedColumn`是一个代表表中完整行的`ColumnElement`。对此构造的支持取决于后端，并且由后端（如 PostgreSQL、Oracle 和 SQL Server）以各种形式支持。

例如:

```py
>>> from sqlalchemy import select, column, func, table
>>> a = table("a", column("id"), column("x"), column("y"))
>>> stmt = select(func.row_to_json(a.table_valued()))
>>> print(stmt)
SELECT  row_to_json(a)  AS  row_to_json_1
FROM  a 
```

版本 1.4.0b2 中的新功能。

另请参见

使用 SQL 函数 - 在 SQLAlchemy 统一教程中

```py
method tablesample(sampling: float | Function[Any], name: str | None = None, seed: roles.ExpressionElementRole[Any] | None = None) → TableSample
```

*继承自* `FromClause.tablesample()` *方法的* `FromClause`

为此`FromClause`返回一个 TABLESAMPLE 别名。

返回值也是顶级`tablesample()`函数提供的`TableSample`构造。

另请参见

`tablesample()` - 用法指南和参数

```py
method unique_params(*optionaldict, **kwargs)
```

*继承自* `Immutable` *的* `Immutable.unique_params()` *方法*

返回一个副本，其中的`bindparam()`元素被替换。

与`ClauseElement.params()`具有相同的功能，只是将 unique=True 添加到受影响的绑定参数中，以便可以使用多个语句。

```py
method update() → Update
```

根据此`TableClause`生成一个`update()`构造。

例如：

```py
table.update().where(table.c.id==7).values(name='foo')
```

有关参数和用法信息，请参阅`update()`。

```py
class sqlalchemy.sql.expression.TableSample
```

代表一个 TABLESAMPLE 子句。

此对象是从`tablesample()`模块级函数以及所有`FromClause`子类上可用的`FromClause.tablesample()`方法构建的。

另请参阅

`tablesample()`

**类签名**

类`sqlalchemy.sql.expression.TableSample` (`sqlalchemy.sql.expression.FromClauseAlias`)

```py
class sqlalchemy.sql.expression.TableValuedAlias
```

对“表值”SQL 函数的别名。

此结构提供了一个 SQL 函数，该函数返回用于 SELECT 语句的 FROM 子句中的列。可以使用`FunctionElement.table_valued()`方法生成对象，例如：

```py
>>> from sqlalchemy import select, func
>>> fn = func.json_array_elements_text('["one", "two", "three"]').table_valued("value")
>>> print(select(fn.c.value))
SELECT  anon_1.value
FROM  json_array_elements_text(:json_array_elements_text_1)  AS  anon_1 
```

新功能在版本 1.4.0b2 中引入。

另请参阅

表值函数 - 在 SQLAlchemy 统一教程中

**成员**

alias(), column, lateral(), render_derived()

**类签名**

类`sqlalchemy.sql.expression.TableValuedAlias` (`sqlalchemy.sql.expression.LateralFromClause`, `sqlalchemy.sql.expression.Alias`)

```py
method alias(name: str | None = None, flat: bool = False) → TableValuedAlias
```

返回此`TableValuedAlias`的新别名。

这将创建一个独特的 FROM 对象，当在 SQL 语句中使用时，它将与原始对象区分开。

```py
attribute column
```

返回表示此`TableValuedAlias`的列表达式。

此访问器用于实现`FunctionElement.column_valued()`方法。有关详细信息，请参阅该方法。

例如：

```py
>>> print(select(func.some_func().table_valued("value").column))
SELECT  anon_1  FROM  some_func()  AS  anon_1 
```

另请参阅

`FunctionElement.column_valued()`

```py
method lateral(name: str | None = None) → LateralFromClause
```

返回一个带有 LATERAL 标志设置的新`TableValuedAlias`，以便在渲染时呈现为 LATERAL。

另请参阅

`lateral()`

```py
method render_derived(name: str | None = None, with_types: bool = False) → TableValuedAlias
```

将“render derived”应用于此`TableValuedAlias`。

这将导致在别名名称后按“AS”顺序列出各个列名，例如：

```py
>>> print(
...     select(
...         func.unnest(array(["one", "two", "three"])).
 table_valued("x", with_ordinality="o").render_derived()
...     )
... )
SELECT  anon_1.x,  anon_1.o
FROM  unnest(ARRAY[%(param_1)s,  %(param_2)s,  %(param_3)s])  WITH  ORDINALITY  AS  anon_1(x,  o) 
```

`with_types`关键字将在别名表达式中内联呈现列类型（此语法目前适用于 PostgreSQL 数据库）:

```py
>>> print(
...     select(
...         func.json_to_recordset(
...             '[{"a":1,"b":"foo"},{"a":"2","c":"bar"}]'
...         )
...         .table_valued(column("a", Integer), column("b", String))
...         .render_derived(with_types=True)
...     )
... )
SELECT  anon_1.a,  anon_1.b  FROM  json_to_recordset(:json_to_recordset_1)
AS  anon_1(a  INTEGER,  b  VARCHAR) 
```

参数：

+   `name` – 将应用于生成的别名的可选字符串名称。如果保持为 None，则将使用唯一的匿名化名称。

+   `with_types` – 如果为 True，则派生列将包括每个列的数据类型规范。这是当前已知对于某些 SQL 函数在 PostgreSQL 中所需的一种特殊语法。

```py
class sqlalchemy.sql.expression.TextualSelect
```

在`SelectBase`接口中包装一个`TextClause`构造。

这允许`TextClause`对象获得一个`.c`集合和其他类似 FROM 的功能，例如`FromClause.alias()`，`SelectBase.cte()`等。

`TextualSelect`构造是通过`TextClause.columns()`方法产生的 - 有关详细信息，请参阅该方法。

从版本 1.4 开始更改：`TextualSelect` 类从`TextAsFrom`重命名为更正确地适应其作为 SELECT 导向对象而不是 FROM 子句的角色。

另请参阅

`text()`

`TextClause.columns()` - 主要创建接口。

**成员**

add_cte(), alias(), as_scalar(), c, compare(), compile(), corresponding_column(), cte(), execution_options(), exists(), exported_columns, get_children(), get_execution_options(), get_label_style(), inherit_cache, is_derived_from(), label(), lateral(), options(), params(), replace_selectable(), scalar_subquery(), select(), selected_columns, self_group(), set_label_style(), subquery(), unique_params()

**类签名**

类`sqlalchemy.sql.expression.TextualSelect` (`sqlalchemy.sql.expression.SelectBase`, `sqlalchemy.sql.expression.ExecutableReturnsRows`, `sqlalchemy.sql.expression.Generative`)

```py
method add_cte(*ctes: CTE, nest_here: bool = False) → Self
```

*继承自* `HasCTE.add_cte()` *方法的* `HasCTE`

向此语句添加一个或多个 `CTE` 构造。

此方法将使给定的 `CTE` 构造与父语句关联起来，以便它们将分别无条件地在最终语句的 WITH 子句中呈现，即使在语句或任何子选择中没有其他地方引用它们。

可选参数`HasCTE.add_cte.nest_here`设置为 True 时，会导致每个给定的`CTE`将直接在此语句中呈现为一个 WITH 子句，而不是被移动到最终呈现的语句顶部，即使此语句作为较大语句内的子查询呈现。

此方法有两个通用用途。一个是嵌入一些没有被显式引用的用途的 CTE 语句，比如将 DML 语句（比如 INSERT 或 UPDATE）作为 CTE 内联到可能间接地引用其结果的主语句中的用例。另一个是提供对应特定一系列 CTE 构造的精确放置的控制，这些 CTE 构造应直接呈现为可能嵌套在较大语句中的特定语句的一部分。

例如：

```py
from sqlalchemy import table, column, select
t = table('t', column('c1'), column('c2'))

ins = t.insert().values({"c1": "x", "c2": "y"}).cte()

stmt = select(t).add_cte(ins)
```

将呈现为：

```py
WITH anon_1 AS
(INSERT INTO t (c1, c2) VALUES (:param_1, :param_2))
SELECT t.c1, t.c2
FROM t
```

上述中，“anon_1” CTE 在 SELECT 语句中没有被引用，但仍完成了运行 INSERT 语句的任务。

同样，在与 DML 相关的上下文中，使用 PostgreSQL `Insert` 构造来生成“upsert”：

```py
from sqlalchemy import table, column
from sqlalchemy.dialects.postgresql import insert

t = table("t", column("c1"), column("c2"))

delete_statement_cte = (
    t.delete().where(t.c.c1 < 1).cte("deletions")
)

insert_stmt = insert(t).values({"c1": 1, "c2": 2})
update_statement = insert_stmt.on_conflict_do_update(
    index_elements=[t.c.c1],
    set_={
        "c1": insert_stmt.excluded.c1,
        "c2": insert_stmt.excluded.c2,
    },
).add_cte(delete_statement_cte)

print(update_statement)
```

以上语句的呈现为：

```py
WITH deletions AS
(DELETE FROM t WHERE t.c1 < %(c1_1)s)
INSERT INTO t (c1, c2) VALUES (%(c1)s, %(c2)s)
ON CONFLICT (c1) DO UPDATE SET c1 = excluded.c1, c2 = excluded.c2
```

版本 1.4.21 中新增。

参数：

+   `*ctes` –

    零个或多个 `CTE` 构造。

    在版本 2.0 中更改：接受多个 CTE 实例

+   `nest_here` –

    如果为 True，则给定的 CTE 或 CTE 将被呈现为若它们在添加到此 `HasCTE` 时指定了 `HasCTE.cte.nesting` 标志为 `True`。假设给定的 CTE 在外部封闭语句中也没有被引用，则当给出此标志时，给定的 CTE 应在此语句级别呈现。

    版本 2.0 中新增。

    另请参见

    `HasCTE.cte.nesting`

```py
method alias(name: str | None = None, flat: bool = False) → Subquery
```

*继承自* `SelectBase.alias()` *方法的* `SelectBase`

返回针对此 `SelectBase` 的命名子查询。

对于 `SelectBase`（与 `FromClause` 相对），这将返回一个大部分与与 `FromClause` 使用的 `Alias` 对象相同的 `Subquery` 对象。

自版本 1.4 起更改：`SelectBase.alias()` 方法现在是 `SelectBase.subquery()` 方法的同义词。

```py
method as_scalar() → ScalarSelect[Any]
```

*继承自* `SelectBase.as_scalar()` *方法的* `SelectBase`

自版本 1.4 起弃用：`SelectBase.as_scalar()` 方法已弃用，并将在将来的版本中移除。请参考 `SelectBase.scalar_subquery()`。

```py
attribute c
```

*继承自* `SelectBase.c` *属性的* `SelectBase`

自版本 1.4 起弃用：`SelectBase.c` 和 `SelectBase.columns` 属性已弃用，并将在将来的版本中移除；这些属性隐式创建了一个应该是明确的子查询。请首先调用 `SelectBase.subquery()` 以创建子查询，然后该子查询包含此属性。要访问此 SELECT 对象从中选择的列，请使用 `SelectBase.selected_columns` 属性。

```py
method compare(other: ClauseElement, **kw: Any) → bool
```

*继承自* `ClauseElement.compare()` *方法的* `ClauseElement`

将这个 `ClauseElement` 与给定的 `ClauseElement` 进行比较。

子类应该重写默认行为，即直接的标识比较。

**kw 是由子类 `compare()` 方法使用的参数，可以用于修改比较条件（请参阅 `ColumnElement`）。

```py
method compile(bind: _HasDialect | None = None, dialect: Dialect | None = None, **kw: Any) → Compiled
```

*继承自* `CompilerElement.compile()` *方法的* `CompilerElement`

编译这个 SQL 表达式。

返回值是一个 `Compiled` 对象。在返回值上调用 `str()` 或 `unicode()` 将产生结果的字符串表示。`Compiled` 对象还可以使用 `params` 访问器返回绑定参数名称和值的字典。

参数：

+   `bind` – 可以提供 `Dialect` 以生成 `Compiled` 对象的 `Connection` 或 `Engine`。如果 `bind` 和 `dialect` 参数都被省略，将使用默认的 SQL 编译器。

+   `column_keys` – 用于 INSERT 和 UPDATE 语句，应在编译语句的 VALUES 子句中存在的列名列表。如果为 `None`，则从目标表对象中呈现所有列。

+   `dialect` – 可以生成 `Compiled` 对象的 `Dialect` 实例。此参数优先于 `bind` 参数。

+   `compile_kwargs` –

    可选的额外参数字典，将在所有“visit”方法中传递给编译器。这允许将任何自定义标志传递给自定义编译结构，例如。它还用于通过以下方式传递`literal_binds`标志的情况：

    ```py
    from sqlalchemy.sql import table, column, select

    t = table('t', column('x'))

    s = select(t).where(t.c.x == 5)

    print(s.compile(compile_kwargs={"literal_binds": True}))
    ```

另请参阅

我如何将 SQL 表达式呈现为字符串，可能包含内联的绑定参数？

```py
method corresponding_column(column: KeyedColumnElement[Any], require_embedded: bool = False) → KeyedColumnElement[Any] | None
```

*继承自* `Selectable.corresponding_column()` *方法，属于* `Selectable`

给定一个 `ColumnElement`，从此 `Selectable.exported_columns` 集合中返回与该原始 `ColumnElement` 相对应的导出 `ColumnElement` 对象，通过一个共同的祖先列。

参数：

+   `column` – 要匹配的目标 `ColumnElement`。

+   `require_embedded` – 仅在给定 `ColumnElement` 的情况下返回对应列，如果给定的 `ColumnElement` 实际上存在于此 `Selectable` 的子元素中。通常，如果该列仅与此 `Selectable` 的导出列之一共享一个公共祖先，则该列将匹配。

另请参阅

`Selectable.exported_columns` - 用于操作的 `ColumnCollection`。

`ColumnCollection.corresponding_column()` - 实现方法。

```py
method cte(name: str | None = None, recursive: bool = False, nesting: bool = False) → CTE
```

*继承自* `HasCTE.cte()` *方法，属于* `HasCTE`

返回一个新的 `CTE` 或公共表达式实例。

公共表达式是 SQL 标准，其中 SELECT 语句可以借助与主语句一起指定的次要语句，在使用名为 “WITH” 的子句时绘制。还可以使用关于 UNION 的特殊语义来允许 “递归” 查询，其中 SELECT 语句可以借助以前已选择的行集。

CTEs 也可应用于某些数据库上的 DML 构造 UPDATE、INSERT 和 DELETE，当与 RETURNING 结合时，作为 CTE 行的来源，以及作为 CTE 行的使用者。

SQLAlchemy 检测到 `CTE` 对象，它们被视为类似于 `Alias` 对象的特殊元素，应交付给语句的 FROM 子句以及语句顶部的 WITH 子句。

对于诸如 PostgreSQL 的 “MATERIALIZED” 和 “NOT MATERIALIZED” 等特殊前缀，可以使用 `CTE.prefix_with()` 方法来建立这些前缀。

自版本 1.3.13 中更改：增加了对前缀的支持。特别是 - MATERIALIZED 和 NOT MATERIALIZED。

参数：

+   `name` - 给常用表达式命名。像 `FromClause.alias()` 一样，名称可以留空，此时在查询编译时将使用匿名符号。

+   `recursive` - 如果为 `True`，将呈现 `WITH RECURSIVE`。递归公共表达式旨在与 UNION ALL 结合使用，以从已选择的行派生行。

+   `nesting` - 

    如果为 `True`，将在引用它的语句中本地呈现 CTE。对于更复杂的情况，也可以使用 `HasCTE.add_cte()` 方法，并使用 `HasCTE.add_cte.nest_here` 参数更精确地控制特定 CTE 的确切放置位置。

    版本 1.4.24 中的新功能。

    另请参阅

    `HasCTE.add_cte()`

以下示例包括两个来自 PostgreSQL 文档的示例 [`www.postgresql.org/docs/current/static/queries-with.html`](https://www.postgresql.org/docs/current/static/queries-with.html)，以及其他示例。

示例 1，非递归：

```py
from sqlalchemy import (Table, Column, String, Integer,
                        MetaData, select, func)

metadata = MetaData()

orders = Table('orders', metadata,
    Column('region', String),
    Column('amount', Integer),
    Column('product', String),
    Column('quantity', Integer)
)

regional_sales = select(
                    orders.c.region,
                    func.sum(orders.c.amount).label('total_sales')
                ).group_by(orders.c.region).cte("regional_sales")

top_regions = select(regional_sales.c.region).\
        where(
            regional_sales.c.total_sales >
            select(
                func.sum(regional_sales.c.total_sales) / 10
            )
        ).cte("top_regions")

statement = select(
            orders.c.region,
            orders.c.product,
            func.sum(orders.c.quantity).label("product_units"),
            func.sum(orders.c.amount).label("product_sales")
    ).where(orders.c.region.in_(
        select(top_regions.c.region)
    )).group_by(orders.c.region, orders.c.product)

result = conn.execute(statement).fetchall()
```

示例 2，WITH RECURSIVE：

```py
from sqlalchemy import (Table, Column, String, Integer,
                        MetaData, select, func)

metadata = MetaData()

parts = Table('parts', metadata,
    Column('part', String),
    Column('sub_part', String),
    Column('quantity', Integer),
)

included_parts = select(\
    parts.c.sub_part, parts.c.part, parts.c.quantity\
    ).\
    where(parts.c.part=='our part').\
    cte(recursive=True)

incl_alias = included_parts.alias()
parts_alias = parts.alias()
included_parts = included_parts.union_all(
    select(
        parts_alias.c.sub_part,
        parts_alias.c.part,
        parts_alias.c.quantity
    ).\
    where(parts_alias.c.part==incl_alias.c.sub_part)
)

statement = select(
            included_parts.c.sub_part,
            func.sum(included_parts.c.quantity).
              label('total_quantity')
        ).\
        group_by(included_parts.c.sub_part)

result = conn.execute(statement).fetchall()
```

示例 3，使用 UPDATE 和 INSERT 进行 upsert，带有 CTEs：

```py
from datetime import date
from sqlalchemy import (MetaData, Table, Column, Integer,
                        Date, select, literal, and_, exists)

metadata = MetaData()

visitors = Table('visitors', metadata,
    Column('product_id', Integer, primary_key=True),
    Column('date', Date, primary_key=True),
    Column('count', Integer),
)

# add 5 visitors for the product_id == 1
product_id = 1
day = date.today()
count = 5

update_cte = (
    visitors.update()
    .where(and_(visitors.c.product_id == product_id,
                visitors.c.date == day))
    .values(count=visitors.c.count + count)
    .returning(literal(1))
    .cte('update_cte')
)

upsert = visitors.insert().from_select(
    [visitors.c.product_id, visitors.c.date, visitors.c.count],
    select(literal(product_id), literal(day), literal(count))
        .where(~exists(update_cte.select()))
)

connection.execute(upsert)
```

示例 4，嵌套 CTE（SQLAlchemy 1.4.24 及更高版本）：

```py
value_a = select(
    literal("root").label("n")
).cte("value_a")

# A nested CTE with the same name as the root one
value_a_nested = select(
    literal("nesting").label("n")
).cte("value_a", nesting=True)

# Nesting CTEs takes ascendency locally
# over the CTEs at a higher level
value_b = select(value_a_nested.c.n).cte("value_b")

value_ab = select(value_a.c.n.label("a"), value_b.c.n.label("b"))
```

上述查询将呈现第二个 CTE 嵌套在第一个中，如下所示，带有内联参数：

```py
WITH
    value_a AS
        (SELECT 'root' AS n),
    value_b AS
        (WITH value_a AS
            (SELECT 'nesting' AS n)
        SELECT value_a.n AS n FROM value_a)
SELECT value_a.n AS a, value_b.n AS b
FROM value_a, value_b
```

可以使用 `HasCTE.add_cte()` 方法来设置相同的 CTE，如下所示（SQLAlchemy 2.0 及更高版本）：

```py
value_a = select(
    literal("root").label("n")
).cte("value_a")

# A nested CTE with the same name as the root one
value_a_nested = select(
    literal("nesting").label("n")
).cte("value_a")

# Nesting CTEs takes ascendency locally
# over the CTEs at a higher level
value_b = (
    select(value_a_nested.c.n).
    add_cte(value_a_nested, nest_here=True).
    cte("value_b")
)

value_ab = select(value_a.c.n.label("a"), value_b.c.n.label("b"))
```

示例 5，非线性 CTE（SQLAlchemy 1.4.28 及更高版本）：

```py
edge = Table(
    "edge",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("left", Integer),
    Column("right", Integer),
)

root_node = select(literal(1).label("node")).cte(
    "nodes", recursive=True
)

left_edge = select(edge.c.left).join(
    root_node, edge.c.right == root_node.c.node
)
right_edge = select(edge.c.right).join(
    root_node, edge.c.left == root_node.c.node
)

subgraph_cte = root_node.union(left_edge, right_edge)

subgraph = select(subgraph_cte)
```

上述查询将在递归 CTE 内渲染 2 个 UNIONs。

```py
WITH RECURSIVE nodes(node) AS (
        SELECT 1 AS node
    UNION
        SELECT edge."left" AS "left"
        FROM edge JOIN nodes ON edge."right" = nodes.node
    UNION
        SELECT edge."right" AS "right"
        FROM edge JOIN nodes ON edge."left" = nodes.node
)
SELECT nodes.node FROM nodes
```

另请参阅

`Query.cte()` - ORM 版本的 `HasCTE.cte()`。

```py
method execution_options(**kw: Any) → Self
```

*继承自* `Executable.execution_options()` *方法的* `Executable`

为语句设置在执行期间生效的非 SQL 选项。

执行选项可以在许多范围内设置，包括每个语句，每个连接或每次执行，使用诸如 `Connection.execution_options()` 这样的方法和接受选项字典的参数，例如 `Connection.execute.execution_options` 和 `Session.execute.execution_options`。

执行选项的主要特点与其他类型的选项（如 ORM 加载器选项）不同，**执行选项从不影响查询的编译 SQL，只影响 SQL 语句本身如何调用或结果如何获取**。也就是说，执行选项不是 SQL 编译所涵盖的部分，也不被视为语句的缓存状态的一部分。

`Executable.execution_options()` 方法是生成性的，就像应用于 `Engine` 和 `Query` 对象的方法一样，这意味着当调用方法时，会返回对象的副本，该副本应用给定的参数到新的副本中，但原始对象保持不变：

```py
statement = select(table.c.x, table.c.y)
new_statement = statement.execution_options(my_option=True)
```

这种行为的一个例外是 `Connection` 对象，在该对象中 `Connection.execution_options()` 方法显式地 **不** 是生成性的。

可传递给`Executable.execution_options()`及其他相关方法和参数字典的选项类型包括被 SQLAlchemy Core 或 ORM 明确消耗的参数，以及未由 SQLAlchemy 定义的任意关键字参数，这意味着这些方法和/或参数字典可用于与自定义代码交互的用户定义参数，可通过诸如`Executable.get_execution_options()`和`Connection.get_execution_options()`等方法访问参数，或者在选择的事件钩子中使用专用的`execution_options`事件参数，例如`ConnectionEvents.before_execute.execution_options`或`ORMExecuteState.execution_options`，例如：

```py
from sqlalchemy import event

@event.listens_for(some_engine, "before_execute")
def _process_opt(conn, statement, multiparams, params, execution_options):
    "run a SQL function before invoking a statement"

    if execution_options.get("do_special_thing", False):
        conn.exec_driver_sql("run_special_function()")
```

在 SQLAlchemy 明确识别的选项范围内，大多数适用于特定类别的对象而不是其他对象。最常见的执行选项包括：

+   `Connection.execution_options.isolation_level` - 通过`Engine`为连接或一类连接设置隔离级别。此选项仅由`Connection`或`Engine`接受。

+   `Connection.execution_options.stream_results` - 表示应使用服务器端游标获取结果；此选项被 `Connection`, `Connection.execute.execution_options` 参数上的 `Connection.execute()`，以及 SQL 语句对象上的 `Executable.execution_options()` 以及 ORM 构造如 `Session.execute()` 所接受。

+   `Connection.execution_options.compiled_cache` - 表示将作为`Connection`或 `Engine` 的 SQL 编译缓存的字典，以及像 `Session.execute()` 这样的 ORM 方法。可以将其传递为 `None` 以禁用语句的缓存。由于在语句对象中携带编译缓存是不明智的，因此此选项不被 `Executable.execution_options()` 接受。

+   `Connection.execution_options.schema_translate_map` - 一个由模式转换映射功能使用的模式名称映射，由 `Connection`, `Engine`, `Executable` 以及像 `Session.execute()` 这样的 ORM 结构所接受。

另请参阅

`Connection.execution_options()`

`Connection.execute.execution_options`

`Session.execute.execution_options`

ORM Execution Options - documentation on all ORM-specific execution options

```py
method exists() → Exists
```

*inherited from the* `SelectBase.exists()` *method of* `SelectBase`

Return an `Exists` representation of this selectable, which can be used as a column expression.

The returned object is an instance of `Exists`.

See also

`exists()`

EXISTS subqueries - in the 2.0 style tutorial.

New in version 1.4.

```py
attribute exported_columns
```

*inherited from the* `SelectBase.exported_columns` *attribute of* `SelectBase`

A `ColumnCollection` that represents the “exported” columns of this `Selectable`, not including `TextClause` constructs.

The “exported” columns for a `SelectBase` object are synonymous with the `SelectBase.selected_columns` collection.

New in version 1.4.

See also

`Select.exported_columns`

`Selectable.exported_columns`

`FromClause.exported_columns`

```py
method get_children(*, omit_attrs: Tuple[str, ...] = (), **kw: Any) → Iterable[HasTraverseInternals]
```

*inherited from the* `HasTraverseInternals.get_children()` *method of* `HasTraverseInternals`

Return immediate child `HasTraverseInternals` elements of this `HasTraverseInternals`.

This is used for visit traversal.

**kw may contain flags that change the collection that is returned, for example to return a subset of items in order to cut down on larger traversals, or to return child items from a different context (such as schema-level collections instead of clause-level).

```py
method get_execution_options() → _ExecuteOptions
```

*继承自* `Executable.get_execution_options()` *方法的* `Executable`

获取执行期间生效的非 SQL 选项。

版本 1.3 中的新功能。

另请参阅

`Executable.execution_options()`

```py
method get_label_style() → SelectLabelStyle
```

*继承自* `SelectBase.get_label_style()` *方法的* `SelectBase`

检索当前的标签样式。

由子类实现。

```py
attribute inherit_cache: bool | None = None
```

*继承自* `HasCacheKey.inherit_cache`

表示此`HasCacheKey` 实例是否应使用其直接超类使用的缓存键生成方案。

该属性默认为`None`，表示构造尚未考虑是否适合参与缓存；这在功能上等同于将值设置为`False`，但还会发出警告。

如果对应于对象的 SQL 不基于此类本地属性而变化，而是基于其超类，则可以将此标志设置为`True`。

另请参阅

启用自定义结构的缓存支持 - 为第三方或用户定义的 SQL 构造设置`HasCacheKey.inherit_cache` 属性的通用指南。

```py
method is_derived_from(fromclause: FromClause | None) → bool
```

*继承自* `ReturnsRows.is_derived_from()` *方法的* `ReturnsRows`

如果此`ReturnsRows` 是从给定的`FromClause`“派生”的，则返回`True`。

一个示例是，表的别名是从该表派生的。

```py
method label(name: str | None) → Label[Any]
```

*继承自* `SelectBase.label()` *方法的* `SelectBase`

返回此可选择性的“标量”表示，嵌入为具有标签的子查询。

另请参阅

`SelectBase.scalar_subquery()`。

```py
method lateral(name: str | None = None) → LateralFromClause
```

*继承自* `SelectBase.lateral()` *方法的* `SelectBase`

返回此 `Selectable` 的 LATERAL 别名。

返回值是顶层 `lateral()` 函数提供的 `Lateral` 构造。

另请参阅

LATERAL 关联 - 用法概述。

```py
method options(*options: ExecutableOption) → Self
```

*继承自* `Executable.options()` *方法的* `Executable`

将选项应用于此语句。

从一般意义上讲，选项是任何可以被 SQL 编译器解释为语句的 Python 对象。这些选项可以被特定方言或特定类型的编译器消耗。

最常见的选项类型是应用于 ORM 查询的“急切加载”和其他加载行为的 ORM 级选项。然而，选项理论上可以用于许多其他目的。

关于特定类型语句的特定类型选项的背景，请参考这些选项对象的文档。

从版本 1.4 开始更改：- 向核心语句对象添加了 `Executable.options()`，以实现统一的核心 / ORM 查询功能。

另请参阅

列加载选项 - 指的是 ORM 查询的特定选项

带有加载器选项的关系加载 - 指的是 ORM 查询的特定选项

```py
method params(_ClauseElement__optionaldict: Mapping[str, Any] | None = None, **kwargs: Any) → Self
```

*继承自* `ClauseElement.params()` *方法的* `ClauseElement`

返回一个副本，其中的 `bindparam()` 元素被替换。

返回此 ClauseElement 的副本，其中的 `bindparam()` 元素被从给定字典中取出的值替换：

```py
>>> clause = column('x') + bindparam('foo')
>>> print(clause.compile().params)
{'foo':None}
>>> print(clause.params({'foo':7}).compile().params)
{'foo':7}
```

```py
method replace_selectable(old: FromClause, alias: Alias) → Self
```

*继承自* `Selectable.replace_selectable()` *方法的* `Selectable`

将所有 `FromClause` 中的 'old' 替换为给定的 `Alias` 对象，并返回此 `FromClause` 的副本。

自版本 1.4 弃用：`Selectable.replace_selectable()` 方法已弃用，并将在将来的发布版本中删除。类似功能可通过 sqlalchemy.sql.visitors 模块获得。

```py
method scalar_subquery() → ScalarSelect[Any]
```

*继承自* `SelectBase.scalar_subquery()` *方法的* `SelectBase`

返回此可选对象的‘标量’表示，可用作列表达式。

返回的对象是 `ScalarSelect` 的实例。

通常，仅在其列子句中有一个列的选择语句有资格用作标量表达式。然后可以在封闭的 SELECT 的 WHERE 子句或列子句中使用标量子查询。

请注意，标量子查询与可使用 `SelectBase.subquery()` 方法生成的 FROM 级子查询不同。

另请参阅

标量子查询和相关子查询 - 在 2.0 教程中

```py
method select(*arg: Any, **kw: Any) → Select
```

*继承自* `SelectBase.select()` *方法的* `SelectBase`

自版本 1.4 弃用：`SelectBase.select()` 方法已弃用，并将在将来的发布版本中删除；此方法隐式创建应显式创建的子查询。请先调用 `SelectBase.subquery()` 以创建子查询，然后可以选择该子查询。

```py
attribute selected_columns
```

代表这个 SELECT 语句或类似结构在其结果集中返回的列的`ColumnCollection`，不包括`TextClause`构造。

此集合与`FromClause`的`FromClause.columns`集合不同，因为该集合中的列不能直接嵌套在另一个 SELECT 语句内；必须先应用一个子查询，该子查询提供了 SQL 所需的必要括号。

对于`TextualSelect`结构，该集合包含通过构造函数传递的`ColumnElement`对象，通常通过`TextClause.columns()`方法传递。

自版本 1.4 新增。

```py
method self_group(against: OperatorType | None = None) → ClauseElement
```

*从* `ClauseElement.self_group()` *方法继承而来的* `ClauseElement`。

对这个`ClauseElement`应用“分组”。

此方法被子类覆盖以返回一个“分组”构造，即括号。特别是它被“二进制”表达式使用，当它们放置到更大的表达式中时，提供了一个围绕自身的分组，以及由`select()`构造在放置到另一个`select()`的 FROM 子句中时使用（请注意，子查询通常应该使用`Select.alias()`方法创建，因为许多平台要求嵌套的 SELECT 语句必须被命名）。

当表达式组合在一起时，`self_group()`的应用是自动的 - 最终用户代码通常不需要直接使用此方法。请注意，SQLAlchemy 的子句构造考虑了运算符优先级 - 所以括号可能不是必需的，例如，在表达式`x OR (y AND z)`中 - AND 优先于 OR。

`ClauseElement`的基本`self_group()`方法只返回自身。

```py
method set_label_style(style: SelectLabelStyle) → TextualSelect
```

返回具有指定标签样式的新可选择项。

由子类实现。

```py
method subquery(name: str | None = None) → Subquery
```

*继承自* `SelectBase.subquery()` *方法的* `SelectBase`

返回此`SelectBase`的子查询。

从 SQL 的角度来看，子查询是一个带有括号的命名构造，可以放置在另一个 SELECT 语句的 FROM 子句中。

给定一个 SELECT 语句，例如：

```py
stmt = select(table.c.id, table.c.name)
```

上述语句可能如下所示：

```py
SELECT table.id, table.name FROM table
```

单独使用子查询形式时，渲染方式相同，但是当嵌入到另一个 SELECT 语句的 FROM 子句中时，它变成了一个命名的子元素：

```py
subq = stmt.subquery()
new_stmt = select(subq)
```

上述内容的渲染如下：

```py
SELECT anon_1.id, anon_1.name
FROM (SELECT table.id, table.name FROM table) AS anon_1
```

从历史上看，`SelectBase.subquery()`等同于在 FROM 对象上调用`FromClause.alias()`方法；然而，由于`SelectBase`对象不是直接的 FROM 对象，因此`SelectBase.subquery()`方法提供了更清晰的语义。

版本 1.4 中的新功能。

```py
method unique_params(_ClauseElement__optionaldict: Dict[str, Any] | None = None, **kwargs: Any) → Self
```

*继承自* `ClauseElement.unique_params()` *方法的* `ClauseElement`

返回一个副本，其中`bindparam()`元素被替换。

与`ClauseElement.params()`具有相同功能，只是将 unique=True 添加到受影响的绑定参数中，以便可以使用多个语句。

```py
class sqlalchemy.sql.expression.Values
```

表示可以作为语句中的 FROM 元素使用的`VALUES`构造。

`Values`对象是从`values()`函数创建的。

版本 1.4 中的新功能。

**成员**

alias(), data(), lateral(), scalar_values()

**类签名**

class `sqlalchemy.sql.expression.Values` (`sqlalchemy.sql.roles.InElementRole`, `sqlalchemy.sql.expression.Generative`, `sqlalchemy.sql.expression.LateralFromClause`)

```py
method alias(name: str | None = None, flat: bool = False) → Self
```

返回一个具有给定名称的新 `Values` 构造的副本。

此方法是 `FromClause.alias()` 方法的 `VALUES` 特定专业化。

另请参阅

使用别名

`alias()`

```py
method data(values: Sequence[Tuple[Any, ...]]) → Self
```

返回一个新的 `Values` 构造，将给定的数据添加到数据列表中。

例如：

```py
my_values = my_values.data([(1, 'value 1'), (2, 'value2')])
```

参数：

**values** – 一个元组序列（即列表），映射到在 `Values` 构造函数中给出的列表达式。

```py
method lateral(name: str | None = None) → LateralFromClause
```

返回一个新的具有 LATERAL 标志设置的 `Values`，以便它呈现为 LATERAL。

另请参阅

`lateral()`

```py
method scalar_values() → ScalarValues
```

返回一个标量 `VALUES` 构造，可用作语句中的 COLUMN 元素。

新版本 2.0.0b4 中的新增内容。

```py
class sqlalchemy.sql.expression.ScalarValues
```

表示一个标量 `VALUES` 构造，可用作语句中的 COLUMN 元素。

`ScalarValues` 对象是从 `Values.scalar_values()` 方法创建的。当 `Values` 在 `IN` 或 `NOT IN` 条件中使用时，它也会自动生成。 

新版本 2.0.0b4 中的新增内容。

**类签名**

class `sqlalchemy.sql.expression.ScalarValues` (`sqlalchemy.sql.roles.InElementRole`, `sqlalchemy.sql.expression.GroupedElement`, `sqlalchemy.sql.expression.ColumnElement`)

## 标签样式常量

与 `GenerativeSelect.set_label_style()` 方法一起使用的常量。

| 对象名称 | 描述 |
| --- | --- |
| SelectLabelStyle | 可传递给 `Select.set_label_style()` 方法的标签样式常量。 |

```py
class sqlalchemy.sql.expression.SelectLabelStyle
```

可传递给 `Select.set_label_style()` 方法的标签样式常量。

**成员**

LABEL_STYLE_DEFAULT，LABEL_STYLE_DISAMBIGUATE_ONLY，LABEL_STYLE_NONE，LABEL_STYLE_TABLENAME_PLUS_COL

**类签名**

`sqlalchemy.sql.expression.SelectLabelStyle` 类 (`enum.Enum`)

```py
attribute LABEL_STYLE_DEFAULT = 2
```

默认标签样式，指的是 `LABEL_STYLE_DISAMBIGUATE_ONLY`。

自 1.4 版本新增。

```py
attribute LABEL_STYLE_DISAMBIGUATE_ONLY = 2
```

标签样式指示，当生成 SELECT 语句的列子句时，具有与现有名称冲突的名称的列应使用半匿名标签进行标记。

下面，大多数列名保持不变，除了第二次出现的 `columna` 名称，它使用标签 `columna_1` 进行标记，以区分它与 `tablea.columna` 的区别：

```py
>>> from sqlalchemy import table, column, select, true, LABEL_STYLE_DISAMBIGUATE_ONLY
>>> table1 = table("table1", column("columna"), column("columnb"))
>>> table2 = table("table2", column("columna"), column("columnc"))
>>> print(select(table1, table2).join(table2, true()).set_label_style(LABEL_STYLE_DISAMBIGUATE_ONLY))
SELECT  table1.columna,  table1.columnb,  table2.columna  AS  columna_1,  table2.columnc
FROM  table1  JOIN  table2  ON  true 
```

与 `GenerativeSelect.set_label_style()` 方法一起使用时，`LABEL_STYLE_DISAMBIGUATE_ONLY` 是所有 SELECT 语句的默认标签样式，适用于 1.x 风格 ORM 查询之外。

自 1.4 版本新增。

```py
attribute LABEL_STYLE_NONE = 0
```

标签样式指示不应将任何自动标签应用于 SELECT 语句的列子句。

下面，名为 `columna` 的列都保持原样，这意味着名称 `columna` 只能指代结果集中的第一次出现，以及如果该语句用作子查询时：

```py
>>> from sqlalchemy import table, column, select, true, LABEL_STYLE_NONE
>>> table1 = table("table1", column("columna"), column("columnb"))
>>> table2 = table("table2", column("columna"), column("columnc"))
>>> print(select(table1, table2).join(table2, true()).set_label_style(LABEL_STYLE_NONE))
SELECT  table1.columna,  table1.columnb,  table2.columna,  table2.columnc
FROM  table1  JOIN  table2  ON  true 
```

与 `Select.set_label_style()` 方法一起使用。

自 1.4 版本新增。

```py
attribute LABEL_STYLE_TABLENAME_PLUS_COL = 1
```

标签样式指示当生成 SELECT 语句的列子句时，所有列都应标记为 `<tablename>_<columnname>`，以区分来自不同表、别名或子查询的同名列。

下面，所有列名都被赋予标签，以便将两个同名列 `columna` 区分为 `table1_columna` 和 `table2_columna`：

```py
>>> from sqlalchemy import table, column, select, true, LABEL_STYLE_TABLENAME_PLUS_COL
>>> table1 = table("table1", column("columna"), column("columnb"))
>>> table2 = table("table2", column("columna"), column("columnc"))
>>> print(select(table1, table2).join(table2, true()).set_label_style(LABEL_STYLE_TABLENAME_PLUS_COL))
SELECT  table1.columna  AS  table1_columna,  table1.columnb  AS  table1_columnb,  table2.columna  AS  table2_columna,  table2.columnc  AS  table2_columnc
FROM  table1  JOIN  table2  ON  true 
```

与 `GenerativeSelect.set_label_style()` 方法一起使用。相当于传统方法 `Select.apply_labels()`；`LABEL_STYLE_TABLENAME_PLUS_COL` 是 SQLAlchemy 的传统自动标签样式。`LABEL_STYLE_DISAMBIGUATE_ONLY` 提供了一种更少侵入性的方法来消除同名列表达式的歧义。

自 1.4 版本新增。

另见

`Select.set_label_style()`

`Select.get_label_style()`

## 可选的基础构造器

顶层“FROM 子句”和“SELECT”构造器。

| 对象名称 | 描述 |
| --- | --- |
| 除去(*selects) | 返回多个可选择的`EXCEPT`。 |
| 除去全部(*selects) | 返回多个可选择的`EXCEPT ALL`。 |
| 存在([__argument]) | 构造一个新的`Exists`构造。 |
| 交集(*selects) | 返回多个可选择的`INTERSECT`。 |
| 全交集(*selects) | 返回多个可选择的`INTERSECT ALL`。 |
| 选择(*entities, **__kw) | 构造一个新的`Select`。 |
| 表(name, *columns, **kw) | 生成一个新的`TableClause`。 |
| 联合(*selects) | 返回多个可选择的`UNION`。 |
| 全联合(*selects) | 返回多个可选择的`UNION ALL`。 |
| 数值(*columns, [name, literal_binds]) | 构造一个`Values`构造。 |

```py
function sqlalchemy.sql.expression.except_(*selects: _SelectStatementForCompoundArgument) → CompoundSelect
```

返回多个可选择的`EXCEPT`。

返回的对象是一个`CompoundSelect`的实例。

参数：

***selects** – 一个`Select`实例的列表。

```py
function sqlalchemy.sql.expression.except_all(*selects: _SelectStatementForCompoundArgument) → CompoundSelect
```

返回多个可选择的`EXCEPT ALL`。

返回的对象是一个`CompoundSelect`的实例。

参数：

***selects** – 一个`Select`实例的列表。

```py
function sqlalchemy.sql.expression.exists(__argument: _ColumnsClauseArgument[Any] | SelectBase | ScalarSelect[Any] | None = None) → Exists
```

构造一个新的`Exists`构造。

`exists()`可以单独调用以生成一个`Exists`构造，它将接受简单的 WHERE 条件：

```py
exists_criteria = exists().where(table1.c.col1 == table2.c.col2)
```

但是，为了在构建 SELECT 语句时更灵活，可以将现有的`Select`构造转换为`Exists`，最方便的方法是利用`SelectBase.exists()`方法：

```py
exists_criteria = (
    select(table2.c.col2).
    where(table1.c.col1 == table2.c.col2).
    exists()
)
```

EXISTS 条件然后用于封闭 SELECT 中：

```py
stmt = select(table1.c.col1).where(exists_criteria)
```

上述语句将具有以下形式：

```py
SELECT col1 FROM table1 WHERE EXISTS
(SELECT table2.col2 FROM table2 WHERE table2.col2 = table1.col1)
```

另请参见

EXISTS 子查询 - 在 2.0 样式教程中。

`SelectBase.exists()` - 将`SELECT`转换为`EXISTS`子句的方法。

```py
function sqlalchemy.sql.expression.intersect(*selects: _SelectStatementForCompoundArgument) → CompoundSelect
```

返回多个可选项的`INTERSECT`。

返回的对象是`CompoundSelect`的实例。

参数：

***selects** – 一个`Select`实例列表。

```py
function sqlalchemy.sql.expression.intersect_all(*selects: _SelectStatementForCompoundArgument) → CompoundSelect
```

返回多个可选项的`INTERSECT ALL`。

返回的对象是`CompoundSelect`的实例。

参数：

***selects** – 一个`Select`实例列表。

```py
function sqlalchemy.sql.expression.select(*entities: _ColumnsClauseArgument[Any], **__kw: Any) → Select[Any]
```

构建一个新的`Select`。

1.4 版新功能： - `select()`函数现在接受位置参数列。顶级`select()`函数将根据传入的参数自动使用 1.x 或 2.x 样式的 API；使用`sqlalchemy.future`模块中的`select()`将强制使用只使用 2.x 样式构造函数。

任何`FromClause`上的`FromClause.select()`方法也提供了类似的功能。

另请参见

使用 SELECT 语句 - 在 SQLAlchemy 统一教程中。

参数：

***entities** –

要从中选择的实体。对于 Core 使用，这通常是一系列 `ColumnElement` 和/或 `FromClause` 对象，它们将形成生成语句的列子句。对于那些是 `FromClause` 实例的对象（通常是 `Table` 或 `Alias` 对象），`FromClause.c` 集合被提取以形成 `ColumnElement` 对象的集合。

此参数还将接受给定的 `TextClause` 构造，以及 ORM 映射的类。

```py
function sqlalchemy.sql.expression.table(name: str, *columns: ColumnClause[Any], **kw: Any) → TableClause
```

生成一个新的 `TableClause`。

返回的对象是 `TableClause` 的一个实例，它表示模式级别的 “语法” 部分 `Table` 对象。它可以用于构建轻量级表构造。

参数：

+   `name` – 表的名称。

+   `columns` – 一组 `column()` 构造。

+   `schema` –

    此表的模式名称。

    在 1.3.18 版中新增：`table()` 现在可以接受 `schema` 参数。

```py
function sqlalchemy.sql.expression.union(*selects: _SelectStatementForCompoundArgument) → CompoundSelect
```

返回多个可选择的 `UNION`。

返回的对象是 `CompoundSelect` 的一个实例。

所有 `FromClause` 子类上都有类似的 `union()` 方法可用。

参数：

+   `*selects` – `Select` 实例的列表。

+   `**kwargs` – 可用的关键字参数与 `select()` 的相同。

```py
function sqlalchemy.sql.expression.union_all(*selects: _SelectStatementForCompoundArgument) → CompoundSelect
```

返回多个可选择的 `UNION ALL`。

返回的对象是 `CompoundSelect` 的一个实例。

所有`FromClause`子类上都有一个类似的`union_all()`方法可用。

参数：

***selects** – 一个`Select`实例的列表。

```py
function sqlalchemy.sql.expression.values(*columns: ColumnClause[Any], name: str | None = None, literal_binds: bool = False) → Values
```

构造一个`Values`构造。

列表达式和`Values`的实际数据是在两个独立的步骤中给出的。构造函数通常接收列表达式，如`column()`构造，然后数据通过`Values.data()`方法作为列表传递，可以多次调用以添加更多数据，例如：

```py
from sqlalchemy import column
from sqlalchemy import values

value_expr = values(
    column('id', Integer),
    column('name', String),
    name="my_values"
).data(
    [(1, 'name1'), (2, 'name2'), (3, 'name3')]
)
```

参数：

+   `*columns` – 列表达式，通常使用`column()`对象组合而成。

+   `name` – 此 VALUES 构造的名称。如果省略，VALUES 构造将在 SQL 表达式中无名。不同的后端可能具有不同的要求。

+   `literal_binds` – 默认为 False。是否在 SQL 输出中将数据值直接呈现为内联形式，而不是使用绑定参数。

## 可选修饰符构造函数

此处列出的函数更常见地作为`FromClause`和`Selectable`元素的方法，例如，`alias()`函数通常通过`FromClause.alias()`方法调用。

| 对象名称 | 描述 |
| --- | --- |
| alias(selectable[, name, flat]) | 返回给定`FromClause`的命名别名。 |
| cte(selectable[, name, recursive]) | 返回一个新的`CTE`，或公共表达式实例。 |
| join(left, right[, onclause, isouter, ...]) | 根据两个`FromClause`表达式生成一个`Join`对象。 |
| lateral(selectable[, name]) | 返回一个`Lateral`对象。 |
| outerjoin(left, right[, onclause, full]) | 返回一个`OUTER JOIN`子句元素。 |
| tablesample(selectable, sampling[, name, seed]) | 返回一个`TableSample`对象。 |

```py
function sqlalchemy.sql.expression.alias(selectable: FromClause, name: str | None = None, flat: bool = False) → NamedFromClause
```

返回给定`FromClause`的命名别名。

对于`Table`和`Join`对象，返回类型是`Alias`对象。其他类型的`NamedFromClause`对象可能会返回其他类型的`FromClause`对象。

命名别名代表具有 SQL 中分配的替代名称的任何`FromClause`，通常在生成时使用`AS`子句，例如`SELECT * FROM table AS aliasname`。

可通过所有`FromClause`对象上可用的`FromClause.alias()`方法获得等效功能。

参数：

+   `selectable` – 任何`FromClause`子类，例如表、选择语句等。

+   `name` – 要分配为别名的字符串名称。如果为`None`，则会在编译时确定性地生成名称。确定性意味着该名称确保与同一语句中使用的其他结构相对于其他结构的唯一性，并且对同一语句对象的每次后续编译也将是相同的名称。

+   `flat` – 如果给定的可选择对象是`Join`的实例，则将传递给 - 有关详细信息，请参阅`Join.alias()`。

```py
function sqlalchemy.sql.expression.cte(selectable: HasCTE, name: str | None = None, recursive: bool = False) → CTE
```

返回一个新的`CTE`或通用表达式实例。

有关 CTE 用法的详细信息，请参阅`HasCTE.cte()`。

```py
function sqlalchemy.sql.expression.join(left: _FromClauseArgument, right: _FromClauseArgument, onclause: _OnClauseArgument | None = None, isouter: bool = False, full: bool = False) → Join
```

给定两个`FromClause`表达式生成一个`Join`对象。

例如：

```py
j = join(user_table, address_table,
         user_table.c.id == address_table.c.user_id)
stmt = select(user_table).select_from(j)
```

将生成类似以下内容的 SQL：

```py
SELECT user.id, user.name FROM user
JOIN address ON user.id = address.user_id
```

任何`FromClause`对象（例如`Table`）都可以使用`FromClause.join()`方法提供类似的功能。

参数：

+   `left` – 连接的左侧。

+   `right` – 连接的右侧；这是任何`FromClause`对象，例如`Table`对象，也可以是一个可选择兼容对象，例如 ORM 映射类。

+   `onclause` – 表示连接的 ON 子句的 SQL 表达式。如果保持为`None`，`FromClause.join()`将尝试基于外键关系连接两个表。

+   `isouter` – 如果为 True，则渲染一个 LEFT OUTER JOIN，而不是 JOIN。

+   `full` – 如果为 True，则渲染一个 FULL OUTER JOIN，而不是 JOIN。

另请参阅

`FromClause.join()` - 基于给定左侧的方法形式。

`Join` - 生成的对象类型。

```py
function sqlalchemy.sql.expression.lateral(selectable: SelectBase | _FromClauseArgument, name: str | None = None) → LateralFromClause
```

返回一个`Lateral`对象。

`Lateral`是一个带有 LATERAL 关键字的子查询的`Alias`子类。

LATERAL 子查询的特殊行为是，它出现在封闭 SELECT 的 FROM 子句中，但可能与该 SELECT 的其他 FROM 子句相关联。这是仅受少数后端支持的子查询的特殊情况，目前更多是最近的 PostgreSQL 版本。

另请参阅

LATERAL 相关性 - 用法概述。

```py
function sqlalchemy.sql.expression.outerjoin(left: _FromClauseArgument, right: _FromClauseArgument, onclause: _OnClauseArgument | None = None, full: bool = False) → Join
```

返回一个`OUTER JOIN`子句元素。

返回的对象是`Join`的一个实例。

任何`FromClause`对象上也可以通过`FromClause.outerjoin()`方法提供类似的功能。

参数：

+   `left` – 连接的左侧。

+   `right` – 连接的右侧。

+   `onclause` – 可选的`ON`子句条件，如果没有指定，则从左侧和右侧之间建立的外键关系中派生。

要将连接链接在一起，请使用所得到的 `Join` 对象上的 `FromClause.join()` 或 `FromClause.outerjoin()` 方法。

```py
function sqlalchemy.sql.expression.tablesample(selectable: _FromClauseArgument, sampling: float | Function[Any], name: str | None = None, seed: roles.ExpressionElementRole[Any] | None = None) → TableSample
```

返回一个 `TableSample` 对象。

`TableSample` 是一个`Alias` 的子类，表示应用了 TABLESAMPLE 子句的表。`tablesample()` 还可以通过 `FromClause.tablesample()` 方法从 `FromClause` 类中获得。

TABLESAMPLE 子句允许从表中随机选择近似百分比的行。它支持多种抽样方法，最常见的是 BERNOULLI 和 SYSTEM。

例如：

```py
from sqlalchemy import func

selectable = people.tablesample(
            func.bernoulli(1),
            name='alias',
            seed=func.random())
stmt = select(selectable.c.people_id)
```

假设 `people` 有一个 `people_id` 列，上述语句将呈现为：

```py
SELECT alias.people_id FROM
people AS alias TABLESAMPLE bernoulli(:bernoulli_1)
REPEATABLE (random())
```

参数：

+   `sampling` – 一个介于 0 和 100 之间的`float`百分比，或 `Function`。

+   `name` – 可选的别名

+   `seed` – 任何实值 SQL 表达式。当指定时，也会呈现 REPEATABLE 子句。

## 可选择类文档

这里的类是使用 Selectable Foundational Constructors 和 Selectable Modifier Constructors 中列出的构造函数生成的。

| Object Name | Description |
| --- | --- |
| Alias | 表示表或可选择的别名（AS）。 |
| AliasedReturnsRows | 对表、子查询和其他可选择的别名的基类。 |
| CompoundSelect | 构成 `UNION`、`UNION ALL` 和其他基于 SELECT 的集合操作的基础。 |
| CTE | 表示一个公共表达式。 |
| Executable | 将一个 `ClauseElement` 标记为支持执行。 |
| Exists | 表示一个 `EXISTS` 子句。 |
| FromClause | 表示可以在 `SELECT` 语句的 `FROM` 子句中使用的元素。 |
| GenerativeSelect | SELECT 语句的基类，允许添加额外的元素。 |
| HasCTE | 声明一个类包含 CTE 支持的混合类。 |
| HasPrefixes |  |
| HasSuffixes |  |
| Join | 表示两个`FromClause`元素之间的 `JOIN` 构造。 |
| Lateral | 表示 LATERAL 子查询。 |
| ReturnsRows | 核心结构的基类，具有某种可以表示行的列的概念。 |
| ScalarSelect | 表示一个标量子查询。 |
| ScalarValues | 表示可以作为语句中的 COLUMN 元素使用的标量 `VALUES` 结构。 |
| Select | 表示一个 `SELECT` 语句。 |
| Selectable | 将一个类标记为可选择的。 |
| SelectBase | SELECT 语句的基类。 |
| Subquery | 表示 SELECT 的子查询。 |
| TableClause | 表示最小的“表”构造。 |
| TableSample | 表示 TABLESAMPLE 子句。 |
| TableValuedAlias | 针对“表值”SQL 函数的别名。 |
| TextualSelect | 在 `SelectBase` 接口中包装一个 `TextClause` 结构。 |
| Values | 表示可以在语句中作为 FROM 元素使用的 `VALUES` 结构。 |

```py
class sqlalchemy.sql.expression.Alias
```

代表一个表或可选择的别名（AS）。

表示别名，通常应用于 SQL 语句中的任何表或子查询，使用 `AS` 关键字（或在某些数据库中不使用关键字，例如 Oracle）。

此对象可以通过`alias()`模块级函数以及所有`FromClause`子类上可用的`FromClause.alias()`方法构建。

另请参阅

`FromClause.alias()`

**成员**

inherit_cache

**类签名**

类`sqlalchemy.sql.expression.Alias`（`sqlalchemy.sql.roles.DMLTableRole`，`sqlalchemy.sql.expression.FromClauseAlias`）

```py
attribute inherit_cache: bool | None = True
```

指示此`HasCacheKey`实例是否应使用其直接超类使用的缓存密钥生成方案。

该属性默认为`None`，表示一个构造尚未考虑是否适合参与缓存；这在功能上等同于将值设置为`False`，只是还会发出警告。

如果与此类本地属性无关并且不是其超类，则可以将此标志设置为`True`。

参见

为自定义构造启用缓存支持 - 有关设置`HasCacheKey.inherit_cache`属性以用于第三方或用户定义的 SQL 构造的一般指南。

```py
class sqlalchemy.sql.expression.AliasedReturnsRows
```

表别名的基类，针对表、子查询和其他可选择项。

**成员**

描述，is_derived_from()，原始

**类签名**

类`sqlalchemy.sql.expression.AliasedReturnsRows`（`sqlalchemy.sql.expression.NoInit`，`sqlalchemy.sql.expression.NamedFromClause`）

```py
attribute description
```

```py
method is_derived_from(fromclause: FromClause | None) → bool
```

如果此`FromClause`从给定的`FromClause`“派生”，则返回`True`。

一个示例是来自该表的一个别名。

```py
attribute original
```

适用于引用 Alias.original 的方言的旧版本。

```py
class sqlalchemy.sql.expression.CompoundSelect
```

构成`UNION`，`UNION ALL`和其他基于 SELECT 的集合操作的基础。

参见

`union()`

`union_all()`

`intersect()`

`intersect_all()`

`except()`

`except_all()`

**成员**

add_cte(), alias(), as_scalar(), c, corresponding_column(), cte(), execution_options(), exists(), exported_columns, fetch(), get_execution_options(), get_label_style(), group_by(), is_derived_from(), label(), lateral(), limit(), offset(), options(), order_by(), replace_selectable(), scalar_subquery(), select(), selected_columns, self_group(), set_label_style(), slice(), subquery(), with_for_update()

**类签名**

类`sqlalchemy.sql.expression.CompoundSelect`（`sqlalchemy.sql.expression.HasCompileState`、`sqlalchemy.sql.expression.GenerativeSelect`、`sqlalchemy.sql.expression.ExecutableReturnsRows`)

```py
method add_cte(*ctes: CTE, nest_here: bool = False) → Self
```

*继承自* `HasCTE.add_cte()` *方法的* `HasCTE`

向该语句添加一个或多个`CTE`构造。

此方法将将给定的`CTE`构造与父语句关联起来，以便它们将无条件地在最终语句的 WITH 子句中呈现，即使在语句或任何子选择中没有引用它们也是如此。

当将可选的`HasCTE.add_cte.nest_here`参数设置为 True 时，每个给定的`CTE`都将以与此语句一起直接呈现的 WITH 子句中呈现，而不是移动到最终呈现语句的顶部，即使此语句在较大语句内作为子查询呈现时也是如此。

此方法有两个常见用途。一个是嵌入 CTE 语句，这些语句具有某种用途，而不需要明确引用，例如，将 DML 语句（如 INSERT 或 UPDATE）作为 CTE 内联到可能间接从其结果中获得结果的主要语句中。另一个用途是提供对应于应该直接呈现为特定语句的一系列特定 CTE 构造的精确放置的控制，该语句可能嵌套在较大语句中。

例如：

```py
from sqlalchemy import table, column, select
t = table('t', column('c1'), column('c2'))

ins = t.insert().values({"c1": "x", "c2": "y"}).cte()

stmt = select(t).add_cte(ins)
```

将呈现为：

```py
WITH anon_1 AS
(INSERT INTO t (c1, c2) VALUES (:param_1, :param_2))
SELECT t.c1, t.c2
FROM t
```

在上述示例中，“anon_1” CTE 没有在 SELECT 语句中被引用，但仍然完成了运行 INSERT 语句的任务。

类似地，在与 DML 相关的上下文中，使用 PostgreSQL`Insert`构造来生成“upsert”：

```py
from sqlalchemy import table, column
from sqlalchemy.dialects.postgresql import insert

t = table("t", column("c1"), column("c2"))

delete_statement_cte = (
    t.delete().where(t.c.c1 < 1).cte("deletions")
)

insert_stmt = insert(t).values({"c1": 1, "c2": 2})
update_statement = insert_stmt.on_conflict_do_update(
    index_elements=[t.c.c1],
    set_={
        "c1": insert_stmt.excluded.c1,
        "c2": insert_stmt.excluded.c2,
    },
).add_cte(delete_statement_cte)

print(update_statement)
```

上述语句呈现为：

```py
WITH deletions AS
(DELETE FROM t WHERE t.c1 < %(c1_1)s)
INSERT INTO t (c1, c2) VALUES (%(c1)s, %(c2)s)
ON CONFLICT (c1) DO UPDATE SET c1 = excluded.c1, c2 = excluded.c2
```

自版本 1.4.21 起新添加。

参数：

+   `*ctes*` –

    零个或多个`CTE`构造。

    自版本 2.0 起更改：接受多个 CTE 实例

+   `nest_here` –

    如果为 True，则给定的 CTE 或 CTE 将呈现为如果当它们被添加到此`HasCTE`时指定了`HasCTE.cte.nesting`标志为`True`。假设给定的 CTE 在外部封闭语句中也没有被引用，那么在给出此标志时，给定的 CTE 应在此语句的级别呈现。

    自版本 2.0 起新添加。

    另请参阅

    `HasCTE.cte.nesting`

```py
method alias(name: str | None = None, flat: bool = False) → Subquery
```

*继承自* `SelectBase` *的* `SelectBase.alias()` *方法*

返回针对此`SelectBase`的命名子查询。

对于 `SelectBase`（与 `FromClause` 相对），此方法返回一个 `Subquery` 对象，其行为与用于 `FromClause` 的 `Alias` 对象大致相同。

自版本 1.4 起更改：`SelectBase.alias()` 方法现在是 `SelectBase.subquery()` 方法的同义词。

```py
method as_scalar() → ScalarSelect[Any]
```

*继承自* `SelectBase.as_scalar()` *方法的* `SelectBase`

自版本 1.4 起弃用：`SelectBase.as_scalar()` 方法已被弃用，并将在未来版本中移除。请参阅 `SelectBase.scalar_subquery()`。

```py
attribute c
```

*继承自* `SelectBase.c` *属性的* `SelectBase`

自版本 1.4 起弃用：`SelectBase.c` 和 `SelectBase.columns` 属性已被弃用，并将在未来版本中移除；这些属性隐式创建了一个应该明确的子查询。请首先调用 `SelectBase.subquery()` 来创建一个子查询，然后该子查询包含此属性。要访问此 SELECT 对象所选列，请使用 `SelectBase.selected_columns` 属性。

```py
method corresponding_column(column: KeyedColumnElement[Any], require_embedded: bool = False) → KeyedColumnElement[Any] | None
```

*继承自* `Selectable.corresponding_column()` *方法的* `Selectable`

给定一个`ColumnElement`，返回此`Selectable`的`Selectable.exported_columns`集合中对应于该原始`ColumnElement`的导出`ColumnElement`对象，通过公共祖先列。

参数：

+   `column` – 要匹配的目标`ColumnElement`。

+   `require_embedded` – 仅当给定的`ColumnElement`实际上存在于此`Selectable`的子元素中时，返回给定`ColumnElement`的相应列。通常，如果列仅与此`Selectable`的导出列之一共享公共祖先，则列将匹配。

另请参见

`Selectable.exported_columns` - 用于操作的`ColumnCollection`。

`ColumnCollection.corresponding_column()` - 实现方法。

```py
method cte(name: str | None = None, recursive: bool = False, nesting: bool = False) → CTE
```

*继承自* `HasCTE.cte()` *方法的* `HasCTE`

返回一个新的`CTE`，即公共表达式实例。

公共表达式是一种 SQL 标准，其中 SELECT 语句可以与主语句一起使用指定的次要语句，使用一个称为“WITH”的子句。还可以使用有关 UNION 的特殊语义，以允许“递归”查询，其中 SELECT 语句可以利用先前已选择的行集。

CTE 也可以应用于某些数据库上的 DML 构造 UPDATE、INSERT 和 DELETE，既作为与 RETURNING 一起组合时 CTE 行的来源，也作为 CTE 行的消费者。

SQLAlchemy 检测到 `CTE` 对象，这些对象与 `Alias` 对象类似，被视为要传递到语句的 FROM 子句以及语句顶部的 WITH 子句的特殊元素。

对于特殊前缀，如 PostgreSQL 的 “MATERIALIZED” 和 “NOT MATERIALIZED”，可以使用 `CTE.prefix_with()` 方法来建立这些前缀。

从版本 1.3.13 开始更改：增加了对前缀的支持。特别是 - MATERIALIZED 和 NOT MATERIALIZED。

参数：

+   `name` – 给公共表达式指定的名称。与 `FromClause.alias()` 类似，名称可以留空，此时将在查询编译时使用匿名符号。

+   `recursive` – 如果为 `True`，将渲染 `WITH RECURSIVE`。递归公共表达式旨在与 UNION ALL 结合使用，以从已选定的行中派生行。

+   `nesting` –

    如果为 `True`，将在引用它的语句中本地渲染 CTE。对于更复杂的场景，还可以使用 `HasCTE.add_cte()` 方法，使用 `HasCTE.add_cte.nest_here` 参数更精确地控制特定 CTE 的精确放置位置。

    从版本 1.4.24 开始新增。

    另见

    `HasCTE.add_cte()`

以下示例包括两个来自 PostgreSQL 文档 [`www.postgresql.org/docs/current/static/queries-with.html`](https://www.postgresql.org/docs/current/static/queries-with.html) 的示例，以及其他示例。

例 1，非递归：

```py
from sqlalchemy import (Table, Column, String, Integer,
                        MetaData, select, func)

metadata = MetaData()

orders = Table('orders', metadata,
    Column('region', String),
    Column('amount', Integer),
    Column('product', String),
    Column('quantity', Integer)
)

regional_sales = select(
                    orders.c.region,
                    func.sum(orders.c.amount).label('total_sales')
                ).group_by(orders.c.region).cte("regional_sales")

top_regions = select(regional_sales.c.region).\
        where(
            regional_sales.c.total_sales >
            select(
                func.sum(regional_sales.c.total_sales) / 10
            )
        ).cte("top_regions")

statement = select(
            orders.c.region,
            orders.c.product,
            func.sum(orders.c.quantity).label("product_units"),
            func.sum(orders.c.amount).label("product_sales")
    ).where(orders.c.region.in_(
        select(top_regions.c.region)
    )).group_by(orders.c.region, orders.c.product)

result = conn.execute(statement).fetchall()
```

例 2，使用 WITH RECURSIVE：

```py
from sqlalchemy import (Table, Column, String, Integer,
                        MetaData, select, func)

metadata = MetaData()

parts = Table('parts', metadata,
    Column('part', String),
    Column('sub_part', String),
    Column('quantity', Integer),
)

included_parts = select(\
    parts.c.sub_part, parts.c.part, parts.c.quantity\
    ).\
    where(parts.c.part=='our part').\
    cte(recursive=True)

incl_alias = included_parts.alias()
parts_alias = parts.alias()
included_parts = included_parts.union_all(
    select(
        parts_alias.c.sub_part,
        parts_alias.c.part,
        parts_alias.c.quantity
    ).\
    where(parts_alias.c.part==incl_alias.c.sub_part)
)

statement = select(
            included_parts.c.sub_part,
            func.sum(included_parts.c.quantity).
              label('total_quantity')
        ).\
        group_by(included_parts.c.sub_part)

result = conn.execute(statement).fetchall()
```

例 3，使用 UPDATE 和 INSERT 与 CTE 进行 upsert：

```py
from datetime import date
from sqlalchemy import (MetaData, Table, Column, Integer,
                        Date, select, literal, and_, exists)

metadata = MetaData()

visitors = Table('visitors', metadata,
    Column('product_id', Integer, primary_key=True),
    Column('date', Date, primary_key=True),
    Column('count', Integer),
)

# add 5 visitors for the product_id == 1
product_id = 1
day = date.today()
count = 5

update_cte = (
    visitors.update()
    .where(and_(visitors.c.product_id == product_id,
                visitors.c.date == day))
    .values(count=visitors.c.count + count)
    .returning(literal(1))
    .cte('update_cte')
)

upsert = visitors.insert().from_select(
    [visitors.c.product_id, visitors.c.date, visitors.c.count],
    select(literal(product_id), literal(day), literal(count))
        .where(~exists(update_cte.select()))
)

connection.execute(upsert)
```

例 4，嵌套 CTE（SQLAlchemy 1.4.24 及更高版本）：

```py
value_a = select(
    literal("root").label("n")
).cte("value_a")

# A nested CTE with the same name as the root one
value_a_nested = select(
    literal("nesting").label("n")
).cte("value_a", nesting=True)

# Nesting CTEs takes ascendency locally
# over the CTEs at a higher level
value_b = select(value_a_nested.c.n).cte("value_b")

value_ab = select(value_a.c.n.label("a"), value_b.c.n.label("b"))
```

上述查询将第二个 CTE 嵌套在第一个内部，并显示为以下内联参数：

```py
WITH
    value_a AS
        (SELECT 'root' AS n),
    value_b AS
        (WITH value_a AS
            (SELECT 'nesting' AS n)
        SELECT value_a.n AS n FROM value_a)
SELECT value_a.n AS a, value_b.n AS b
FROM value_a, value_b
```

可以使用以下方法设置相同的 CTE：`HasCTE.add_cte()`（SQLAlchemy 2.0 及更高版本）：

```py
value_a = select(
    literal("root").label("n")
).cte("value_a")

# A nested CTE with the same name as the root one
value_a_nested = select(
    literal("nesting").label("n")
).cte("value_a")

# Nesting CTEs takes ascendency locally
# over the CTEs at a higher level
value_b = (
    select(value_a_nested.c.n).
    add_cte(value_a_nested, nest_here=True).
    cte("value_b")
)

value_ab = select(value_a.c.n.label("a"), value_b.c.n.label("b"))
```

例 5，非线性 CTE（SQLAlchemy 1.4.28 及更高版本）：

```py
edge = Table(
    "edge",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("left", Integer),
    Column("right", Integer),
)

root_node = select(literal(1).label("node")).cte(
    "nodes", recursive=True
)

left_edge = select(edge.c.left).join(
    root_node, edge.c.right == root_node.c.node
)
right_edge = select(edge.c.right).join(
    root_node, edge.c.left == root_node.c.node
)

subgraph_cte = root_node.union(left_edge, right_edge)

subgraph = select(subgraph_cte)
```

上述查询将在递归 CTE 中渲染 2 个 UNION：

```py
WITH RECURSIVE nodes(node) AS (
        SELECT 1 AS node
    UNION
        SELECT edge."left" AS "left"
        FROM edge JOIN nodes ON edge."right" = nodes.node
    UNION
        SELECT edge."right" AS "right"
        FROM edge JOIN nodes ON edge."left" = nodes.node
)
SELECT nodes.node FROM nodes
```

另见

`Query.cte()` - ORM 版本的 `HasCTE.cte()`。

```py
method execution_options(**kw: Any) → Self
```

*继承自* `Executable.execution_options()` *方法的* `Executable`

设置在执行期间生效的语句的非 SQL 选项。

执行选项可以在许多范围内设置，包括每个语句、每个连接或每次执行，使用诸如 `Connection.execution_options()` 和接受选项字典的参数的方法，如 `Connection.execute.execution_options` 和 `Session.execute.execution_options`.

执行选项的主要特征与其他类型的选项（如 ORM 加载器选项）相反，**执行选项从不影响查询的编译 SQL，只影响 SQL 语句本身的调用方式或结果的提取方式**。也就是说，执行选项不是 SQL 编译所容纳的部分，也不被视为语句的缓存状态的一部分。

`Executable.execution_options()` 方法是生成的，就像应用于 `Engine` 和 `Query` 对象的方法一样，这意味着当调用方法时，将返回对象的副本，该副本将给定的参数应用于该新副本，但原始对象保持不变：

```py
statement = select(table.c.x, table.c.y)
new_statement = statement.execution_options(my_option=True)
```

这种行为的一个例外是 `Connection` 对象，在这里 `Connection.execution_options()` 方法明确地 **不是** 生成的。

`Executable.execution_options()` 和其他相关方法以及参数字典可以接受的选项类型包括由 SQLAlchemy Core 或 ORM 明确使用的参数，以及未被 SQLAlchemy 定义的任意关键字参数。这意味着这些方法和/或参数字典可以用于与自定义代码交互的用户定义参数，可以使用诸如 `Executable.get_execution_options()` 和 `Connection.get_execution_options()` 这样的方法来访问参数，或者在选定的事件钩子中使用专用的 `execution_options` 事件参数，例如 `ConnectionEvents.before_execute.execution_options` 或 `ORMExecuteState.execution_options`，例如：

```py
from sqlalchemy import event

@event.listens_for(some_engine, "before_execute")
def _process_opt(conn, statement, multiparams, params, execution_options):
    "run a SQL function before invoking a statement"

    if execution_options.get("do_special_thing", False):
        conn.exec_driver_sql("run_special_function()")
```

在 SQLAlchemy 明确识别的选项范围内，大多数适用于特定类别的对象而不是其他对象。最常见的执行选项包括：

+   `Connection.execution_options.isolation_level` - 通过 `Engine` 为连接或连接类设置隔离级别。此选项仅由 `Connection` 或 `Engine` 接受。

+   `Connection.execution_options.stream_results` - 指示结果应使用服务器端游标获取；这个选项被`Connection`接受，由`Connection.execute()`上的`Connection.execute.execution_options`参数接受，并且由 SQL 语句对象的`Executable.execution_options()`以及 ORM 构造如`Session.execute()`额外接受。

+   `Connection.execution_options.compiled_cache` - 指示一个字典，将用作`Connection`或`Engine`的 SQL 编译缓存的服务，以及`Session.execute()`等 ORM 方法的编译缓存。可以传递`None`来禁用语句的缓存。这个选项不被`Executable.execution_options()`接受，因为在语句对象中携带编译缓存是不明智的。

+   `Connection.execution_options.schema_translate_map` - 一个由模式翻译映射功能使用的模式名称映射，被`Connection`、`Engine`、`Executable`接受，以及 ORM 构造如`Session.execute()`。

另请参见

`Connection.execution_options()`

`Connection.execute.execution_options`

`Session.execute.execution_options`

ORM 执行选项 - 所有 ORM 特定执行选项的文档

```py
method exists() → Exists
```

*继承自* `SelectBase.exists()` *方法的* `SelectBase`

返回此可选择性的`Exists`表示，可用作列表达式。

返回对象是`Exists`的实例。

另请参阅

`exists()`

EXISTS 子查询 - 在 2.0 风格教程中。

新版本 1.4 中推出。

```py
attribute exported_columns
```

*继承自* `SelectBase.exported_columns` *属性的* `SelectBase`

一个`ColumnCollection`，代表此`Selectable`的“导出”列，不包括`TextClause`构造。

`SelectBase`对象的“导出”列与`SelectBase.selected_columns`集合是同义词。

新版本 1.4 中推出。

另请参阅

`Select.exported_columns`

`Selectable.exported_columns`

`FromClause.exported_columns`

```py
method fetch(count: _LimitOffsetType, with_ties: bool = False, percent: bool = False) → Self
```

*继承自* `GenerativeSelect.fetch()` *方法的* `GenerativeSelect`

返回一个应用了给定 FETCH FIRST 条件的新可选择项。

这是一个数值，通常在结果选择中呈现为 `FETCH {FIRST | NEXT} [ count ] {ROW | ROWS} {ONLY | WITH TIES}` 表达式。此功能目前已为 Oracle、PostgreSQL、MSSQL 实现。

使用`GenerativeSelect.offset()`来指定偏移量。

注意

`GenerativeSelect.fetch()` 方法将替换任何应用了 `GenerativeSelect.limit()` 的子句。

版本 1.4 中新增。

参数：

+   `count` – 一个整数 COUNT 参数，或者提供整数结果的 SQL 表达式。当 `percent=True` 时，这将代表要返回的行的百分比，而不是绝对值。传递 `None` 来重置它。

+   `with_ties` – 当为`True`时，使用 WITH TIES 选项来返回任何与结果集中的最后一位并列的额外行，根据 `ORDER BY` 子句确定。在这种情况下，`ORDER BY` 可能是强制性的。默认为`False`

+   `percent` – 当为`True`时，`count`表示要返回的所选行总数的百分比。默认为`False`

另请参阅

`GenerativeSelect.limit()`

`GenerativeSelect.offset()`

```py
method get_execution_options() → _ExecuteOptions
```

*继承自* `Executable.get_execution_options()` *方法的参数* `Executable`

获取在执行期间生效的非 SQL 选项。

版本 1.3 中新增。

另请参阅

`Executable.execution_options()`

```py
method get_label_style() → SelectLabelStyle
```

*继承自* `GenerativeSelect.get_label_style()` *方法的参数* `GenerativeSelect`

检索当前标签样式。

版本 1.4 中新增。

```py
method group_by(_GenerativeSelect__first: Literal[None, _NoArg.NO_ARG] | _ColumnExpressionOrStrLabelArgument[Any] = _NoArg.NO_ARG, *clauses: _ColumnExpressionOrStrLabelArgument[Any]) → Self
```

*继承自* `GenerativeSelect.group_by()` *方法的参数* `GenerativeSelect`

返回一个应用了给定的 GROUP BY 条件列表的新可选择项。

所有现有的 GROUP BY 设置都可以通过传递 `None` 来抑制。

例如：

```py
stmt = select(table.c.name, func.max(table.c.stat)).\
group_by(table.c.name)
```

参数：

***子句** – 一系列将用于生成 GROUP BY 子句的`ColumnElement`构造。

另请参阅

带有 GROUP BY / HAVING 的聚合函数 - 在 SQLAlchemy 统一教程中

按标签排序或分组 - 在 SQLAlchemy 统一教程中

```py
method is_derived_from(fromclause: FromClause | None) → bool
```

如果此`ReturnsRows`是从给定的`FromClause`‘派生’，则返回`True`。

一个示例是，表的别名是从该表派生的。

```py
method label(name: str | None) → Label[Any]
```

*继承自* `SelectBase.label()` *方法的* `SelectBase`

返回此可选择项的“标量”表示，嵌入为带有标签的子查询。

另请参阅

`SelectBase.scalar_subquery()`.

```py
method lateral(name: str | None = None) → LateralFromClause
```

*继承自* `SelectBase.lateral()` *方法的* `SelectBase`

返回此`Selectable`的 LATERAL 别名。

返回值是由顶层`lateral()`函数提供的`Lateral`构造。

另请参阅

LATERAL 相关性 - 用法概述。

```py
method limit(limit: _LimitOffsetType) → Self
```

*继承自* `GenerativeSelect.limit()` *方法的* `GenerativeSelect`

返回一个应用给定 LIMIT 条件的新可选择项。

这是一个通常呈现为`LIMIT`表达式的数值。不支持`LIMIT`的后端将尝试提供类似的功能。

注意

`GenerativeSelect.limit()` 方法将替换应用的任何子句与`GenerativeSelect.fetch()`。

参数：

**limit** – 一个整数 LIMIT 参数，或提供整数结果的 SQL 表达式。传递`None`以重置它。

另请参阅

`GenerativeSelect.fetch()`

`GenerativeSelect.offset()`

```py
method offset(offset: _LimitOffsetType) → Self
```

*继承自* `GenerativeSelect.offset()` *方法的* `GenerativeSelect`

返回一个应用了给定 OFFSET 条件的新可选择项。

这是一个数值，通常在生成的选择中呈现为`OFFSET`表达式。不支持`OFFSET`的后端将尝试提供类似的功能。

参数：

**offset** – 一个整数 OFFSET 参数，或提供整数结果的 SQL 表达式。传递`None`以重置它。

另请参阅

`GenerativeSelect.limit()`

`GenerativeSelect.fetch()`

```py
method options(*options: ExecutableOption) → Self
```

*继承自* `Executable.options()` *方法的* `Executable`

将选项应用于此语句。

从一般意义上讲，选项是任何 SQL 编译器可以解释的 Python 对象。这些选项可以被特定的方言或特定类型的编译器消耗。

最常见的选项类型是应用于 ORM 查询的“急加载”和其他加载行为的 ORM 级选项。然而，选项理论上可以用于许多其他目的。

有关特定类型语句的特定类型选项的背景，请参阅这些选项对象的文档。

自版本 1.4 更改：- 向核心语句对象添加`Executable.options()`，以实现统一的核心/ORM 查询功能的目标。

另请参阅

列加载选项 - 指特定于 ORM 查询使用的选项

使用加载器选项加载关系 - 指特定于 ORM 查询使用的选项

```py
method order_by(_GenerativeSelect__first: Literal[None, _NoArg.NO_ARG] | _ColumnExpressionOrStrLabelArgument[Any] = _NoArg.NO_ARG, *clauses: _ColumnExpressionOrStrLabelArgument[Any]) → Self
```

*继承自* `GenerativeSelect.order_by()` *方法的* `GenerativeSelect`

返回一个应用了给定 ORDER BY 条件列表的新可选择项。

例如：

```py
stmt = select(table).order_by(table.c.id, table.c.name)
```

多次调用此方法等同于一次调用，其中所有子句都被连接起来。 通过单独传递`None`可以取消所有现有的 ORDER BY 条件。 然后可以通过再次调用`Query.order_by()`来添加新的 ORDER BY 条件，例如：

```py
# will erase all ORDER BY and ORDER BY new_col alone
stmt = stmt.order_by(None).order_by(new_col)
```

参数：

***clauses** – 一系列将用于生成 ORDER BY 子句的`ColumnElement`构造。

另请参阅

ORDER BY - 在 SQLAlchemy 统一教程中

按标签排序或分组 - 在 SQLAlchemy 统一教程中

```py
method replace_selectable(old: FromClause, alias: Alias) → Self
```

*继承自* `Selectable.replace_selectable()` *方法的* `Selectable`

用给定的`Alias`对象替换所有`FromClause` ‘old’的出现，返回此`FromClause`的副本。

自 1.4 版本起弃用：`Selectable.replace_selectable()`方法已弃用，并将在将来的版本中删除。 类似功能可通过 sqlalchemy.sql.visitors 模块获得。

```py
method scalar_subquery() → ScalarSelect[Any]
```

*继承自* `SelectBase.scalar_subquery()` *方法的* `SelectBase`

返回此可选择项的‘标量’表示，可用作列表达式。

返回的对象是`ScalarSelect`的实例。

通常，仅在其列子句中具有一个列的选择语句有资格用作标量表达式。 然后可以在封闭 SELECT 的 WHERE 子句或列子句中使用标量子查询。

请注意，标量子查询与使用`SelectBase.subquery()`方法生成的 FROM 级子查询有所不同。

另请参阅

标量和相关子查询 - 在 2.0 教程中

```py
method select(*arg: Any, **kw: Any) → Select
```

*继承自* `SelectBase.select()` *方法的* `SelectBase`

自版本 1.4 开始不推荐使用：`SelectBase.select()` 方法已被弃用，并将在未来的版本中删除；此方法隐式创建了一个应明确表示的子查询。请先调用 `SelectBase.subquery()` 来创建子查询，然后再进行选择。

```py
attribute selected_columns
```

表示此 SELECT 语句或类似构造返回其结果集中的列的 `ColumnCollection`，不包括 `TextClause` 构造。

对于 `CompoundSelect`，`CompoundSelect.selected_columns` 属性返回包含在集合操作中的语句系列中的第一个 SELECT 语句中选择的列。

参见

`Select.selected_columns`

版本 1.4 中的新内容。

```py
method self_group(against: OperatorType | None = None) → GroupedElement
```

对此 `ClauseElement` 应用“分组”。

子类覆盖此方法以返回“分组”构造，即括号。特别是在将“二元”表达式放入较大表达式时，它被“二元”表达式用于提供围绕自己的分组，以及当 `select()` 构造放入另一个 `select()` 的 FROM 子句时。 (请注意，通常应使用 `Select.alias()` 方法创建子查询，因为许多平台要求嵌套的 SELECT 语句具有名称)。

随着表达式的组合，`self_group()` 的应用是自动的 - 最终用户代码不应直接使用此方法。请注意，SQLAlchemy 的子句构造会考虑运算符优先级 - 因此在像 `x OR (y AND z)` 这样的表达式中可能不需要括号 - AND 优先于 OR。

`ClauseElement` 的基本 `self_group()` 方法只返回自身。

```py
method set_label_style(style: SelectLabelStyle) → CompoundSelect
```

返回具有指定标签样式的新可选择项。

有三种“标签样式”可用，`SelectLabelStyle.LABEL_STYLE_DISAMBIGUATE_ONLY`，`SelectLabelStyle.LABEL_STYLE_TABLENAME_PLUS_COL`，以及`SelectLabelStyle.LABEL_STYLE_NONE`。默认样式是`SelectLabelStyle.LABEL_STYLE_TABLENAME_PLUS_COL`。

在现代的 SQLAlchemy 中，通常不需要更改标签样式，因为通过使用 `ColumnElement.label()` 方法更有效地使用每个表达式的标签。在以前的版本中，`LABEL_STYLE_TABLENAME_PLUS_COL` 用于消除来自不同表、别名或子查询的同名列；更新后的 `LABEL_STYLE_DISAMBIGUATE_ONLY` 仅对与现有名称冲突的名称应用标签，因此标签的影响最小。

消除歧义的原因主要是，当创建子查询时，从给定的 `FromClause.c` 集合中可以使用所有列表达式。

从版本 1.4 新增：- `GenerativeSelect.set_label_style()` 方法替换了以前的 `.apply_labels()`、`.with_labels()` 和 `use_labels=True` 方法和/或参数的组合。

另请参阅

`LABEL_STYLE_DISAMBIGUATE_ONLY`

`LABEL_STYLE_TABLENAME_PLUS_COL`

`LABEL_STYLE_NONE`

`LABEL_STYLE_DEFAULT`

```py
method slice(start: int, stop: int) → Self
```

*继承自* `GenerativeSelect.slice()` *方法的* `GenerativeSelect`

根据片段将 LIMIT / OFFSET 应用于此语句。

起始和停止索引的行为类似于 Python 内置 `range()` 函数的参数。此方法提供了一种替代方法，用于使用 `LIMIT`/`OFFSET` 获取查询的片段。

例如，

```py
stmt = select(User).order_by(User).id.slice(1, 3)
```

渲染为

```py
SELECT  users.id  AS  users_id,
  users.name  AS  users_name
FROM  users  ORDER  BY  users.id
LIMIT  ?  OFFSET  ?
(2,  1)
```

注意

`GenerativeSelect.slice()` 方法将替换任何应用了 `GenerativeSelect.fetch()` 的子句。

新版本 1.4 中新增：从 ORM 泛化的 `GenerativeSelect.slice()` 方法。

另请参阅

`GenerativeSelect.limit()`

`GenerativeSelect.offset()`

`GenerativeSelect.fetch()`

```py
method subquery(name: str | None = None) → Subquery
```

*继承自* `SelectBase.subquery()` *方法的* `SelectBase`

返回此 `SelectBase` 的子查询。

从 SQL 视角来看，子查询是一个带有括号的命名构造，可以放置在另一个 SELECT 语句的 FROM 子句中。

给定一个类似的 SELECT 语句：

```py
stmt = select(table.c.id, table.c.name)
```

上述语句可能看起来像：

```py
SELECT table.id, table.name FROM table
```

单独的子查询形式呈现相同的方式，但是当嵌入到另一个 SELECT 语句的 FROM 子句中时，它变成了一个命名子元素：

```py
subq = stmt.subquery()
new_stmt = select(subq)
```

上述呈现为：

```py
SELECT anon_1.id, anon_1.name
FROM (SELECT table.id, table.name FROM table) AS anon_1
```

历史上，`SelectBase.subquery()` 等同于在 FROM 对象上调用 `FromClause.alias()` 方法；然而，由于 `SelectBase` 对象不是直接的 FROM 对象，因此 `SelectBase.subquery()` 方法提供了更清晰的语义。

新版本 1.4 中新增。

```py
method with_for_update(*, nowait: bool = False, read: bool = False, of: _ForUpdateOfArgument | None = None, skip_locked: bool = False, key_share: bool = False) → Self
```

*继承自* `GenerativeSelect.with_for_update()` *方法的* `GenerativeSelect`

为这个 `GenerativeSelect` 指定一个 `FOR UPDATE` 子句。

例如：

```py
stmt = select(table).with_for_update(nowait=True)
```

在像 PostgreSQL 或 Oracle 这样的数据库中，上述将呈现为类似于：

```py
SELECT table.a, table.b FROM table FOR UPDATE NOWAIT
```

在其他后端，`nowait` 选项会被忽略，而会产生：

```py
SELECT table.a, table.b FROM table FOR UPDATE
```

当没有参数调用时，语句将以后缀`FOR UPDATE`渲染。然后可以提供额外的参数，允许常见的特定于数据库的变体。

参数：

+   `nowait` – 布尔值；在 Oracle 和 PostgreSQL 方言上会渲染为`FOR UPDATE NOWAIT`。

+   `read` – 布尔值；在 MySQL 上会渲染为`LOCK IN SHARE MODE`，在 PostgreSQL 上会渲染为`FOR SHARE`。在 PostgreSQL 上，与`nowait`结合使用时，会渲染为`FOR SHARE NOWAIT`。

+   `of` – SQL 表达式或 SQL 表达式元素列表（通常是`Column`对象或兼容表达式，对于某些后端也可以是表达式）将渲染为`FOR UPDATE OF`子句；由 PostgreSQL、Oracle、某些 MySQL 版本和可能其他后端支持。根据后端的不同，可能会渲染为表或列。

+   `skip_locked` – 布尔值，会在 Oracle 和 PostgreSQL 方言上渲染为`FOR UPDATE SKIP LOCKED`，如果还指定了`read=True`，则会渲染为`FOR SHARE SKIP LOCKED`。

+   `key_share` – 布尔值，会在 PostgreSQL 方言上渲染为`FOR NO KEY UPDATE`，或者如果与`read=True`结合使用，则会渲染为`FOR KEY SHARE`。

```py
class sqlalchemy.sql.expression.CTE
```

代表一个公共表达式。

`CTE`对象是使用任何 SELECT 语句的`SelectBase.cte()`方法获得的。较少见的语法还允许在 DML 构造上存在的`HasCTE.cte()`方法的使用，例如`Insert`、`Update`和`Delete`。有关 CTE 的用法详细信息，请参阅`HasCTE.cte()`方法。

另请参阅

子查询和 CTE - 在 2.0 教程中

`HasCTE.cte()` - 调用样式示例

**成员**

alias(), union(), union_all()

**类签名**

类 `sqlalchemy.sql.expression.CTE` (`sqlalchemy.sql.roles.DMLTableRole`, `sqlalchemy.sql.roles.IsCTERole`, `sqlalchemy.sql.expression.Generative`, `sqlalchemy.sql.expression.HasPrefixes`, `sqlalchemy.sql.expression.HasSuffixes`, `sqlalchemy.sql.expression.AliasedReturnsRows`)

```py
method alias(name: str | None = None, flat: bool = False) → CTE
```

返回此 `CTE` 的一个 `Alias` 。

此方法是 `FromClause.alias()` 方法的 CTE 特定专业化。

另请参阅

使用别名

`alias()`

```py
method union(*other: _SelectStatementForCompoundArgument) → CTE
```

返回一个新的 `CTE` ，其中包含原始 CTE 与作为位置参数提供的给定可选项的 SQL `UNION`。

参数:

***其他** –

一个或多个用于创建 UNION 的元素。

在 1.4.28 版更改：现在接受多个元素。

另请参阅

`HasCTE.cte()` - 调用样式示例

```py
method union_all(*other: _SelectStatementForCompoundArgument) → CTE
```

返回一个新的 `CTE` ，其中包含原始 CTE 与作为位置参数提供的给定可选项的 SQL `UNION ALL`。

参数:

***其他** –

一个或多个用于创建 UNION 的元素。

在 1.4.28 版更改：现在接受多个元素。

另请参阅

`HasCTE.cte()` - 调用样式示例

```py
class sqlalchemy.sql.expression.Executable
```

将一个 `ClauseElement` 标记为支持执行。

`Executable` 是所有“语句”类型对象的超类，包括 `select()`, `delete()`, `update()`, `insert()`, `text()`。

**成员**

execution_options(), get_execution_options(), options()

**类签名**

类`sqlalchemy.sql.expression.Executable` (`sqlalchemy.sql.roles.StatementRole`)

```py
method execution_options(**kw: Any) → Self
```

设置在执行过程中生效的非 SQL 选项。

可以使用诸如`Connection.execution_options()`和接受选项字典的参数，如`Connection.execute.execution_options`和`Session.execute.execution_options`等方法，在许多范围内设置执行选项，包括每个语句，每个连接或每次执行。

与其他类型的选项（如 ORM 加载器选项）相比，执行选项的主要特征在于**执行选项从不影响查询的编译 SQL，而只影响 SQL 语句本身如何被调用或结果如何获取**。也就是说，执行选项不是 SQL 编译的一部分，也不被视为语句的缓存状态的一部分。

`Executable.execution_options()` 方法是生成器的，就像该方法应用于`Engine`和`Query`对象时一样，这意味着当调用该方法时，将返回对象的副本，该副本应用给定参数到新副本，但保留原始副本不变：

```py
statement = select(table.c.x, table.c.y)
new_statement = statement.execution_options(my_option=True)
```

这种行为的一个例外是`Connection`对象，其中`Connection.execution_options()`方法明确地**不**是生成器。

可以传递给`Executable.execution_options()`和其他相关方法和参数字典的选项种类包括被 SQLAlchemy Core 或 ORM 明确消耗的参数，以及未被 SQLAlchemy 定义的任意关键字参数，这意味着这些方法和/或参数字典可用于与自定义代码交互的用户定义参数，可以使用诸如 `Executable.get_execution_options()` 和 `Connection.get_execution_options()` 这样的方法访问参数，或者在选定的事件钩子中使用专用的 `execution_options` 事件参数，例如 `ConnectionEvents.before_execute.execution_options` 或 `ORMExecuteState.execution_options`，例如：

```py
from sqlalchemy import event

@event.listens_for(some_engine, "before_execute")
def _process_opt(conn, statement, multiparams, params, execution_options):
    "run a SQL function before invoking a statement"

    if execution_options.get("do_special_thing", False):
        conn.exec_driver_sql("run_special_function()")
```

在 SQLAlchemy 明确识别的选项范围内，大多数适用于特定类的对象而不是其他对象。最常见的执行选项包括：

+   `Connection.execution_options.isolation_level` - 通过 `Engine` 为连接或连接类设置隔离级别。此选项仅由 `Connection` 或 `Engine` 接受。

+   `Connection.execution_options.stream_results` - 指示应该使用服务器端游标获取结果；这个选项被`Connection.execute()`上的`Connection`，`Connection.execute()`参数以及 SQL 语句对象上的`Executable.execution_options()`接受，以及 ORM 构造上的`Session.execute()`。

+   `Connection.execution_options.compiled_cache` - 指示一个字典，将作为`Connection`或`Engine`的 SQL 编译缓存的服务，以及`Session.execute()`等 ORM 方法的缓存。可以传递为`None`来禁用语句的缓存。这个选项不被`Executable.execution_options()`接受，因为在语句对象中携带编译缓存是不明智的。

+   `Connection.execution_options.schema_translate_map` - 模式翻译映射功能使用的模式名称的映射，被`Connection`，`Engine`，`Executable`以及 ORM 构造如`Session.execute()`所接受。

另请参见

`Connection.execution_options()`

`Connection.execute.execution_options`

`Session.execute.execution_options`

ORM 执行选项 - 所有 ORM 特定执行选项的文档

```py
method get_execution_options() → _ExecuteOptions
```

获取在执行期间生效的非 SQL 选项。

1.3 版本中新增。

另请参见

`Executable.execution_options()`

```py
method options(*options: ExecutableOption) → Self
```

将选项应用于此语句。

从一般意义上讲，选项是可以被 SQL 编译器解释为语句的任何 Python 对象。这些选项可以被特定的方言或特定类型的编译器消耗。

最常见的选项类型是应用“急加载”和其他加载行为到 ORM 查询的 ORM 级别选���。然而，选项理论上可以用于许多其他目的。

有关特定类型语句的特定类型选项的背景，请参阅这些选项对象的文档。

从 1.4 版本开始更改：- 向核心语句对象添加`Executable.options()`，以实现统一的核心/ORM 查询功能目标。

另请参见

列加载选项 - 指特定于 ORM 查询使用的选项

使用加载选项加载关系 - 指特定于 ORM 查询使用的选项

```py
class sqlalchemy.sql.expression.Exists
```

表示一个`EXISTS`子句。

有关用法的描述，请参见`exists()`。

通过调用`SelectBase.exists()`，也可以从`select()`实例构建`EXISTS`子句。

**成员**

correlate(), correlate_except(), inherit_cache, select(), select_from(), where()

**类签名**

类`sqlalchemy.sql.expression.Exists`（`sqlalchemy.sql.expression.UnaryExpression`）

```py
method correlate(*fromclauses: Literal[None, False] | _FromClauseArgument) → Self
```

将相关性应用于此`Exists`指定的子查询。

另请参见

`ScalarSelect.correlate()`

```py
method correlate_except(*fromclauses: Literal[None, False] | _FromClauseArgument) → Self
```

将相关性应用于此`Exists` 指示的子查询。

另请参阅

`ScalarSelect.correlate_except()`

```py
attribute inherit_cache: bool | None = True
```

指示此`HasCacheKey` 实例是否应使用其直接超类使用的缓存密钥生成方案。

该属性默认为 `None`，表示构造尚未考虑其是否适合参与缓存；这在功能上等同于将值设置为 `False`，除了还会发出警告。

如果对象对应的 SQL 不基于仅限于此类而不是其超类的属性更改，则可以在特定类上将此标志设置为 `True`。

另请参阅

为自定义结构启用缓存支持 - 为第三方或用户定义的 SQL 结构设置`HasCacheKey.inherit_cache` 属性的通用指南。

```py
method select() → Select
```

返回此`Exists` 的 SELECT。

例如：

```py
stmt = exists(some_table.c.id).where(some_table.c.id == 5).select()
```

这将产生一个类似于的语句：

```py
SELECT EXISTS (SELECT id FROM some_table WHERE some_table = :param) AS anon_1
```

另请参阅

`select()` - 允许任意列列表的通用方法。

```py
method select_from(*froms: _FromClauseArgument) → Self
```

返回一个新的`Exists` 构造，将给定的表达式应用于所包含的选择语句的`Select.select_from()` 方法。

注意

通常最好首先构建一个包括所需 WHERE 子句的`Select` 语句，然后使用`SelectBase.exists()` 方法立即生成一个`Exists` 对象。

```py
method where(*clause: _ColumnExpressionArgument[bool]) → Self
```

返回一个新的`exists()` 构造，将给定的表达式添加到其 WHERE 子句中，并通过 AND 连接到现有子句中（如果有）。

注意

通常最好先构建一个包括所需的 WHERE 子句的`Select`语句，然后使用`SelectBase.exists()`方法一次性生成一个`Exists`对象。

```py
class sqlalchemy.sql.expression.FromClause
```

代表一个可在`SELECT`语句的`FROM`子句中使用的元素。

`FromClause`的最常见形式是`Table`和`select()`构造。所有`FromClause`对象共有的关键特征包括：

+   一个`c`集合，提供对`ColumnElement`对象集合的按名称访问。

+   一个`primary_key`属性，它是指示`primary_key`标志的所有`ColumnElement`对象的集合。

+   生成“from”子句的各种派生方法，包括`FromClause.alias()`，`FromClause.join()`，`FromClause.select()`。

**成员**

alias()，c，columns，description，entity_namespace，exported_columns，foreign_keys，is_derived_from()，join()，outerjoin()，primary_key，schema，select()，tablesample()

**类签名**

类`sqlalchemy.sql.expression.FromClause`（`sqlalchemy.sql.roles.AnonymizedFromClauseRole`，`sqlalchemy.sql.expression.Selectable`）。

```py
method alias(name: str | None = None, flat: bool = False) → NamedFromClause
```

返回此`FromClause`的别名。

例如：

```py
a2 = some_table.alias('a2')
```

上述代码创建了一个`Alias`对象，可用作任何 SELECT 语句中的 FROM 子句。

另请参阅

使用别名

`alias()`

```py
attribute c
```

`FromClause.columns`的同义词。

返回：

一个`ColumnCollection`。

```py
attribute columns
```

由此`FromClause`维护的`ColumnElement`对象的基于名称的集合。

`columns`或`c`集合是使用绑定表或其他可选择列构建 SQL 表达式的入口：

```py
select(mytable).where(mytable.c.somecolumn == 5)
```

返回：

一个`ColumnCollection`对象。

```py
attribute description
```

这是`FromClause`的简要描述。

主要用于错误消息格式化。

```py
attribute entity_namespace
```

返回用于在 SQL 表达式中基于名称访问的命名空间。

这是用于解析“filter_by()”类型表达式的命名空间，例如：

```py
stmt.filter_by(address='some address')
```

默认为`.c`集合，但在内部可以使用“entity_namespace”注释进行覆盖，以提供替代结果。

```py
attribute exported_columns
```

代表此`Selectable`的“导出”列的`ColumnCollection`。

`FromClause`对象的“导出”列与`FromClause.columns`集合是同义词。

版本 1.4 中的新功能。

另请参阅

`Selectable.exported_columns`

`SelectBase.exported_columns`

```py
attribute foreign_keys
```

返回此 FromClause 引用的`ForeignKey`标记对象的集合。

每个`ForeignKey`都是一个`Table`范围内的`ForeignKeyConstraint`的成员。

另请参阅

`Table.foreign_key_constraints`

```py
method is_derived_from(fromclause: FromClause | None) → bool
```

如果此`FromClause`是从给定的`FromClause`‘派生’，则返回`True`。

一个示例是一个表的别名是从该表派生的。

```py
method join(right: _FromClauseArgument, onclause: _ColumnExpressionArgument[bool] | None = None, isouter: bool = False, full: bool = False) → Join
```

从这个`FromClause`返回到另一个带有“isouter”标志设置为 True 的`Join`。

例如：

```py
from sqlalchemy import join

j = user_table.join(address_table,
                user_table.c.id == address_table.c.user_id)
stmt = select(user_table).select_from(j)
```

将会生成类似以下的 SQL：

```py
SELECT user.id, user.name FROM user
JOIN address ON user.id = address.user_id
```

参数：

+   `right` – 连接的右侧；这是任何`FromClause`对象，如一个`Table`对象，也可以是一个可选择兼容的对象，如 ORM 映射类。

+   `onclause` – 代表连接的 ON 子句的 SQL 表达式。如果保持为`None`，`FromClause.join()`将尝试基于外键关系连接这两个表。

+   `isouter` – 如果为 True，则渲染 LEFT OUTER JOIN，而不是 JOIN。

+   `full` – 如果为 True，则渲染 FULL OUTER JOIN，而不是 LEFT OUTER JOIN。意味着`FromClause.join.isouter`。

另请参阅

`join()` - 独立函数

`Join` - 生成的对象类型

```py
method outerjoin(right: _FromClauseArgument, onclause: _ColumnExpressionArgument[bool] | None = None, full: bool = False) → Join
```

从这个`FromClause`返回到另一个带有“isouter”标志设置为 True 的`Join`。

例如：

```py
from sqlalchemy import outerjoin

j = user_table.outerjoin(address_table,
                user_table.c.id == address_table.c.user_id)
```

以上等同于：

```py
j = user_table.join(
    address_table,
    user_table.c.id == address_table.c.user_id,
    isouter=True)
```

参数：

+   `right` – 连接的右侧；这是任何`FromClause`对象，如`Table`对象，也可以是 ORM 映射类等可选择兼容对象。

+   `onclause` – 表示连接的 ON 子句的 SQL 表达式。如果保持为`None`，`FromClause.join()`将尝试基于外键关系连接这两个表。

+   `full` – 如果为 True，则渲染 FULL OUTER JOIN，而不是 LEFT OUTER JOIN。

另请参阅

`FromClause.join()`

`Join`

```py
attribute primary_key
```

返回由此`_selectable.FromClause`的主键组成的`Column`对象的可迭代集合。

对于`Table`对象，这个集合由`PrimaryKeyConstraint`表示，它本身是一个`Column`对象的可迭代集合。

```py
attribute schema: str | None = None
```

为此`FromClause`定义‘schema’属性。

对于大多数对象来说，这通常是`None`，除了`Table`对象，其中它被视为`Table.schema`参数的值。

```py
method select() → Select
```

返回此`FromClause`的 SELECT。

例如：

```py
stmt = some_table.select().where(some_table.c.id == 5)
```

另请参阅

`select()` - 允许任意列列表的通用方法。

```py
method tablesample(sampling: float | Function[Any], name: str | None = None, seed: roles.ExpressionElementRole[Any] | None = None) → TableSample
```

返回此`FromClause`的 TABLESAMPLE 别名。

返回值是顶层`tablesample()`函数提供的`TableSample`构造。

另请参阅

`tablesample()` - 用法指南和参数

```py
class sqlalchemy.sql.expression.GenerativeSelect
```

SELECT 语句的基类，可以添加额外的元素。

这用作 `Select` 和 `CompoundSelect` 的基础，其中可以添加诸如 ORDER BY、GROUP BY 等元素，并且可以控制列的呈现。与 `TextualSelect` 相比，它虽然是 `SelectBase` 的子类，也是一个 SELECT 构造，但代表一个固定的文本字符串，在这个级别无法更改，只能作为子查询包装。

**成员**

fetch(), get_label_style(), group_by(), limit(), offset(), order_by(), set_label_style(), slice(), with_for_update()

**类签名**

类 `sqlalchemy.sql.expression.GenerativeSelect`（`sqlalchemy.sql.expression.SelectBase`，`sqlalchemy.sql.expression.Generative`）

```py
method fetch(count: _LimitOffsetType, with_ties: bool = False, percent: bool = False) → Self
```

返回一个应用了给定 FETCH FIRST 准则的新可选择项。

这是一个数值，通常在结果选择中呈现为 `FETCH {FIRST | NEXT} [ count ] {ROW | ROWS} {ONLY | WITH TIES}` 表达式。此功能目前已为 Oracle、PostgreSQL、MSSQL 实现。

使用 `GenerativeSelect.offset()` 来指定偏移量。

注意

`GenerativeSelect.fetch()` 方法将替换任何应用的子句，使用 `GenerativeSelect.limit()`。

版本 1.4 中新增。

参数：

+   `count` – 一个整数 COUNT 参数，或者提供整数结果的 SQL 表达式。当 `percent=True` 时，这将表示要返回的行数的百分比，而不是绝对值。传递 `None` 来重置它。

+   `with_ties` – 当为 `True` 时，使用 WITH TIES 选项返回任何与结果集中最后位置并列的额外行，根据 `ORDER BY` 子句。在这种情况下，`ORDER BY` 可能是强制性的。默认为 `False`

+   `percent` – 当为 `True` 时，`count` 表示要返回的选定行总数的百分比。默认为 `False`。

另请参阅

`GenerativeSelect.limit()`

`GenerativeSelect.offset()`

```py
method get_label_style() → SelectLabelStyle
```

检索当前标签样式。

自 1.4 版本新功能。

```py
method group_by(_GenerativeSelect__first: Literal[None, _NoArg.NO_ARG] | _ColumnExpressionOrStrLabelArgument[Any] = _NoArg.NO_ARG, *clauses: _ColumnExpressionOrStrLabelArgument[Any]) → Self
```

返回一个应用了给定 GROUP BY 条件列表的新可选择对象。

通过传递 `None`，可以取消所有现有的 GROUP BY 设置。

例如：

```py
stmt = select(table.c.name, func.max(table.c.stat)).\
group_by(table.c.name)
```

参数：

***clauses** – 一系列将用于生成 GROUP BY 子句的 `ColumnElement` 构造。

另请参阅

使用 GROUP BY / HAVING 的聚合函数 - 在 SQLAlchemy 统一教程 中

按标签排序或分组 - 在 SQLAlchemy 统一教程 中

```py
method limit(limit: _LimitOffsetType) → Self
```

返回一个应用了给定 LIMIT 条件的新可选择对象。

这是一个数值，通常在结果选择中呈现为 `LIMIT` 表达式。不支持 `LIMIT` 的后端将尝试提供类似的功能。

注意

`GenerativeSelect.limit()` 方法将替换任何应用了 `GenerativeSelect.fetch()` 的子句。

参数：

**limit** – 一个整数的 LIMIT 参数，或者提供整数结果的 SQL 表达式。传递 `None` 来重置它。

另请参阅

`GenerativeSelect.fetch()`

`GenerativeSelect.offset()`

```py
method offset(offset: _LimitOffsetType) → Self
```

返回一个应用了给定 OFFSET 条件的新可选择对象。

这是一个数值，通常在结果选择中呈现为 `OFFSET` 表达式。不支持 `OFFSET` 的后端将尝试提供类似的功能。

参数：

**offset** – 一个整数的 OFFSET 参数，或者提供整数结果的 SQL 表达式。传递 `None` 来重置它。

另请参阅

`GenerativeSelect.limit()`

`GenerativeSelect.fetch()`

```py
method order_by(_GenerativeSelect__first: Literal[None, _NoArg.NO_ARG] | _ColumnExpressionOrStrLabelArgument[Any] = _NoArg.NO_ARG, *clauses: _ColumnExpressionOrStrLabelArgument[Any]) → Self
```

返回一个应用了给定 ORDER BY 条件列表的新可选择对象。

例如：

```py
stmt = select(table).order_by(table.c.id, table.c.name)
```

多次调用此方法等效于将所有子句连接一次。通过仅传递 `None` 可取消所有现有的 ORDER BY 标准。然后，可以通过再次调用 `Query.order_by()` 来添加新的 ORDER BY 标准，例如：

```py
# will erase all ORDER BY and ORDER BY new_col alone
stmt = stmt.order_by(None).order_by(new_col)
```

参数：

***clauses** – 一系列用于生成 ORDER BY 子句的`ColumnElement`构造。

另请参阅

ORDER BY - 在 SQLAlchemy 统一教程中

按标签排序或分组 - 在 SQLAlchemy 统一教程中

```py
method set_label_style(style: SelectLabelStyle) → Self
```

返回一个具有指定标签样式的新可选择项。

有三种可用的“标签样式”，分别是`SelectLabelStyle.LABEL_STYLE_DISAMBIGUATE_ONLY`、`SelectLabelStyle.LABEL_STYLE_TABLENAME_PLUS_COL`和`SelectLabelStyle.LABEL_STYLE_NONE`。默认样式是`SelectLabelStyle.LABEL_STYLE_TABLENAME_PLUS_COL`。

在现代 SQLAlchemy 中，通常不需要更改标签样式，因为通过使用 `ColumnElement.label()` 方法更有效地使用按表达式标记。在过去的版本中，`LABEL_STYLE_TABLENAME_PLUS_COL` 用于消除来自不同表、别名或子查询的同名列；较新的 `LABEL_STYLE_DISAMBIGUATE_ONLY` 现在仅对与现有名称冲突的名称应用标签，因此此标记的影响是最小的。

正确性歧义的原因主要是当创建子查询时，所有列表达式都可以从给定的`FromClause.c`集合中使用。

新于 1.4 版本： - `GenerativeSelect.set_label_style()` 方法替换了先前的 `.apply_labels()`、`.with_labels()` 和 `use_labels=True` 方法和/或参数的组合。

另请参阅

`LABEL_STYLE_DISAMBIGUATE_ONLY`

`LABEL_STYLE_TABLENAME_PLUS_COL`

`LABEL_STYLE_NONE`

`LABEL_STYLE_DEFAULT`

```py
method slice(start: int, stop: int) → Self
```

根据片段对该语句应用 LIMIT / OFFSET。

开始和停止索引的行为类似于 Python 内置的 `range()` 函数的参数。该方法提供了使用 `LIMIT`/`OFFSET` 获取查询片段的替代方法。

例如，

```py
stmt = select(User).order_by(User).id.slice(1, 3)
```

渲染为

```py
SELECT  users.id  AS  users_id,
  users.name  AS  users_name
FROM  users  ORDER  BY  users.id
LIMIT  ?  OFFSET  ?
(2,  1)
```

注意

`GenerativeSelect.slice()` 方法将替换应用于 `GenerativeSelect.fetch()` 的任何子句。

1.4 版中的新功能：从 ORM 泛化添加了 `GenerativeSelect.slice()` 方法。

另请参阅

`GenerativeSelect.limit()`

`GenerativeSelect.offset()`

`GenerativeSelect.fetch()`

```py
method with_for_update(*, nowait: bool = False, read: bool = False, of: _ForUpdateOfArgument | None = None, skip_locked: bool = False, key_share: bool = False) → Self
```

为此 `GenerativeSelect` 指定一个 `FOR UPDATE` 子句。

例如：

```py
stmt = select(table).with_for_update(nowait=True)
```

在像 PostgreSQL 或 Oracle 这样的数据库上，以上内容将呈现为如下语句：

```py
SELECT table.a, table.b FROM table FOR UPDATE NOWAIT
```

在其他后端上，`nowait` 选项将被忽略，而会产生如下结果：

```py
SELECT table.a, table.b FROM table FOR UPDATE
```

调用时不带参数，语句将以后缀 `FOR UPDATE` 呈现。然后可以提供附加参数，允许使用常见的数据库特定变体。

参数：

+   `nowait` – 布尔值；在 Oracle 和 PostgreSQL 方言上将呈现为 `FOR UPDATE NOWAIT`。

+   `read` – 布尔值；在 MySQL 上将呈现为 `LOCK IN SHARE MODE`，在 PostgreSQL 上将呈现为 `FOR SHARE`。在 PostgreSQL 上，与 `nowait` 结合时，将呈现为 `FOR SHARE NOWAIT`。

+   `of` – SQL 表达式或 SQL 表达式元素列表（通常为 `Column` 对象或兼容表达式，对于某些后端也可以是表达式），它将渲染为 `FOR UPDATE OF` 子句；受 PostgreSQL、Oracle、某些 MySQL 版本支持，可能还支持其他后端。可能根据后端渲染为表或列。

+   `skip_locked` – 布尔值，将在 Oracle 和 PostgreSQL 方言上呈现为 `FOR UPDATE SKIP LOCKED`，或者如果也指定了 `read=True`，则呈现为 `FOR SHARE SKIP LOCKED`。

+   `key_share` – 布尔值，将在 PostgreSQL 方言上呈现为 `FOR NO KEY UPDATE`，或者如果与 `read=True` 结合，将在 PostgreSQL 方言上呈现为 `FOR KEY SHARE`。

```py
class sqlalchemy.sql.expression.HasCTE
```

声明要包含 CTE 支持的类的 Mixin。

**成员**

add_cte(), cte()

**类签名**

类`sqlalchemy.sql.expression.HasCTE` (`sqlalchemy.sql.roles.HasCTERole`, `sqlalchemy.sql.expression.SelectsRows`)

```py
method add_cte(*ctes: CTE, nest_here: bool = False) → Self
```

将一个或多个`CTE`构造添加到该语句中。

此方法将把给定的`CTE` 构造与父语句关联起来，以便它们将在最终语句的 WITH 子句中无条件地渲染，即使在语句或任何子选择中未引用。

当设置为 True 时，可选的 `HasCTE.add_cte.nest_here` 参数将使每个给定的 `CTE` 直接与此语句一起渲染在一个 WITH 子句中，而不是被移动到最终渲染语句的顶部，即使此语句在更大语句中作为子查询渲染。

此方法有两个一般用途。一个是嵌入服务于某些目的而不被显式引用的 CTE 语句，比如将 DML 语句（比如 INSERT 或 UPDATE）作为 CTE 内联到可能间接从其结果中汲取的主语句中。另一个是提供对应特定语句的一系列 CTE 构造的确切放置的控制，这些构造应该保持直接渲染为嵌套在较大语句中的特定语句。

例如：

```py
from sqlalchemy import table, column, select
t = table('t', column('c1'), column('c2'))

ins = t.insert().values({"c1": "x", "c2": "y"}).cte()

stmt = select(t).add_cte(ins)
```

渲染如下：

```py
WITH anon_1 AS
(INSERT INTO t (c1, c2) VALUES (:param_1, :param_2))
SELECT t.c1, t.c2
FROM t
```

上面的“anon_1” CTE 在 SELECT 语句中没有被引用，但仍然完成了运行 INSERT 语句的任务。

类似地，在 DML 相关的上下文中，使用 PostgreSQL 的`Insert`构造生成“upsert”：

```py
from sqlalchemy import table, column
from sqlalchemy.dialects.postgresql import insert

t = table("t", column("c1"), column("c2"))

delete_statement_cte = (
    t.delete().where(t.c.c1 < 1).cte("deletions")
)

insert_stmt = insert(t).values({"c1": 1, "c2": 2})
update_statement = insert_stmt.on_conflict_do_update(
    index_elements=[t.c.c1],
    set_={
        "c1": insert_stmt.excluded.c1,
        "c2": insert_stmt.excluded.c2,
    },
).add_cte(delete_statement_cte)

print(update_statement)
```

上述语句渲染为：

```py
WITH deletions AS
(DELETE FROM t WHERE t.c1 < %(c1_1)s)
INSERT INTO t (c1, c2) VALUES (%(c1)s, %(c2)s)
ON CONFLICT (c1) DO UPDATE SET c1 = excluded.c1, c2 = excluded.c2
```

从版本 1.4.21 开始新增。

参数：

+   `*ctes` -

    零个或多个`CTE`构造。

    从版本 2.0 开始更改：接受多个 CTE 实例

+   `nest_here` -

    如果为 True，则给定的 CTE 或 CTE 将被渲染，就好像它们在添加到此 `HasCTE` 时指定了 `HasCTE.cte.nesting` 标志为 True 一样。假设给定的 CTE 在外层语句中也没有被引用，当给定此标志时，应该在此语句级别呈现这些 CTE。

    从版本 2.0 开始。

    另请参阅

    `HasCTE.cte.nesting`

```py
method cte(name: str | None = None, recursive: bool = False, nesting: bool = False) → CTE
```

返回一个新的`CTE`，或公共表达式实例。

通用表达式是 SQL 标准，其中 SELECT 语句可以利用与主语句一起指定的辅助语句，使用名为“WITH”的子句。关于 UNION 的特殊语义也可以用于允许“递归”查询，其中 SELECT 语句可以利用先前选择的行集。

CTE 也可以应用于某些数据库上的 DML 构造 UPDATE、INSERT 和 DELETE，既作为与 RETURNING 结合时 CTE 行的来源，也作为 CTE 行的��费者。

SQLAlchemy 检测到`CTE`对象，这些对象与`Alias`对象类似地处理，作为要传递给语句的 FROM 子句以及语句顶部的 WITH 子句的特殊元素。

对于特殊前缀，如 PostgreSQL 的“MATERIALIZED”和“NOT MATERIALIZED”，可以使用`CTE.prefix_with()`方法来建立这些。

在版本 1.3.13 中更改：添加了对前缀的支持。特别是 - MATERIALIZED 和 NOT MATERIALIZED。

参数：

+   `name` – 给通用表达式的名称。与`FromClause.alias()`类似，名称可以保留为`None`，在这种情况下，将在查询编译时使用匿名符号。

+   `recursive` – 如果为`True`，将呈现`WITH RECURSIVE`。递归通用表达式旨在与 UNION ALL 结合使用，以从已选择的行中派生行。

+   `nesting` –

    如果为`True`，将在引用它的语句中本地呈现 CTE。对于更复杂的情况，还可以使用`HasCTE.add_cte()`方法，使用`HasCTE.add_cte.nest_here`参数更精细地控制特定 CTE 的确切放置。

    新版本 1.4.24。

    另请参见

    `HasCTE.add_cte()`

以下示例包括两个来自 PostgreSQL 文档的示例，网址为[`www.postgresql.org/docs/current/static/queries-with.html`](https://www.postgresql.org/docs/current/static/queries-with.html)，以及其他示例。

示例 1，非递归：

```py
from sqlalchemy import (Table, Column, String, Integer,
                        MetaData, select, func)

metadata = MetaData()

orders = Table('orders', metadata,
    Column('region', String),
    Column('amount', Integer),
    Column('product', String),
    Column('quantity', Integer)
)

regional_sales = select(
                    orders.c.region,
                    func.sum(orders.c.amount).label('total_sales')
                ).group_by(orders.c.region).cte("regional_sales")

top_regions = select(regional_sales.c.region).\
        where(
            regional_sales.c.total_sales >
            select(
                func.sum(regional_sales.c.total_sales) / 10
            )
        ).cte("top_regions")

statement = select(
            orders.c.region,
            orders.c.product,
            func.sum(orders.c.quantity).label("product_units"),
            func.sum(orders.c.amount).label("product_sales")
    ).where(orders.c.region.in_(
        select(top_regions.c.region)
    )).group_by(orders.c.region, orders.c.product)

result = conn.execute(statement).fetchall()
```

示例 2，WITH RECURSIVE：

```py
from sqlalchemy import (Table, Column, String, Integer,
                        MetaData, select, func)

metadata = MetaData()

parts = Table('parts', metadata,
    Column('part', String),
    Column('sub_part', String),
    Column('quantity', Integer),
)

included_parts = select(\
    parts.c.sub_part, parts.c.part, parts.c.quantity\
    ).\
    where(parts.c.part=='our part').\
    cte(recursive=True)

incl_alias = included_parts.alias()
parts_alias = parts.alias()
included_parts = included_parts.union_all(
    select(
        parts_alias.c.sub_part,
        parts_alias.c.part,
        parts_alias.c.quantity
    ).\
    where(parts_alias.c.part==incl_alias.c.sub_part)
)

statement = select(
            included_parts.c.sub_part,
            func.sum(included_parts.c.quantity).
              label('total_quantity')
        ).\
        group_by(included_parts.c.sub_part)

result = conn.execute(statement).fetchall()
```

示例 3，使用 CTE 进行更新和插入：

```py
from datetime import date
from sqlalchemy import (MetaData, Table, Column, Integer,
                        Date, select, literal, and_, exists)

metadata = MetaData()

visitors = Table('visitors', metadata,
    Column('product_id', Integer, primary_key=True),
    Column('date', Date, primary_key=True),
    Column('count', Integer),
)

# add 5 visitors for the product_id == 1
product_id = 1
day = date.today()
count = 5

update_cte = (
    visitors.update()
    .where(and_(visitors.c.product_id == product_id,
                visitors.c.date == day))
    .values(count=visitors.c.count + count)
    .returning(literal(1))
    .cte('update_cte')
)

upsert = visitors.insert().from_select(
    [visitors.c.product_id, visitors.c.date, visitors.c.count],
    select(literal(product_id), literal(day), literal(count))
        .where(~exists(update_cte.select()))
)

connection.execute(upsert)
```

示例 4，嵌套 CTE（SQLAlchemy 1.4.24 及以上版本）：

```py
value_a = select(
    literal("root").label("n")
).cte("value_a")

# A nested CTE with the same name as the root one
value_a_nested = select(
    literal("nesting").label("n")
).cte("value_a", nesting=True)

# Nesting CTEs takes ascendency locally
# over the CTEs at a higher level
value_b = select(value_a_nested.c.n).cte("value_b")

value_ab = select(value_a.c.n.label("a"), value_b.c.n.label("b"))
```

上述查询将第二个 CTE 嵌套在第一个内部，如下所示，带有内联参数：

```py
WITH
    value_a AS
        (SELECT 'root' AS n),
    value_b AS
        (WITH value_a AS
            (SELECT 'nesting' AS n)
        SELECT value_a.n AS n FROM value_a)
SELECT value_a.n AS a, value_b.n AS b
FROM value_a, value_b
```

相同的 CTE 可以使用`HasCTE.add_cte()`方法设置如下（SQLAlchemy 2.0 及以上版本）：

```py
value_a = select(
    literal("root").label("n")
).cte("value_a")

# A nested CTE with the same name as the root one
value_a_nested = select(
    literal("nesting").label("n")
).cte("value_a")

# Nesting CTEs takes ascendency locally
# over the CTEs at a higher level
value_b = (
    select(value_a_nested.c.n).
    add_cte(value_a_nested, nest_here=True).
    cte("value_b")
)

value_ab = select(value_a.c.n.label("a"), value_b.c.n.label("b"))
```

示例 5，非线性 CTE（SQLAlchemy 1.4.28 及以上版本）：

```py
edge = Table(
    "edge",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("left", Integer),
    Column("right", Integer),
)

root_node = select(literal(1).label("node")).cte(
    "nodes", recursive=True
)

left_edge = select(edge.c.left).join(
    root_node, edge.c.right == root_node.c.node
)
right_edge = select(edge.c.right).join(
    root_node, edge.c.left == root_node.c.node
)

subgraph_cte = root_node.union(left_edge, right_edge)

subgraph = select(subgraph_cte)
```

上述查询将在递归 CTE 中呈现 2 个 UNIONs：

```py
WITH RECURSIVE nodes(node) AS (
        SELECT 1 AS node
    UNION
        SELECT edge."left" AS "left"
        FROM edge JOIN nodes ON edge."right" = nodes.node
    UNION
        SELECT edge."right" AS "right"
        FROM edge JOIN nodes ON edge."left" = nodes.node
)
SELECT nodes.node FROM nodes
```

另请参见

`Query.cte()` - `HasCTE.cte()`的 ORM 版本。

```py
class sqlalchemy.sql.expression.HasPrefixes
```

**成员**

prefix_with()

```py
method prefix_with(*prefixes: _TextCoercedExpressionArgument[Any], dialect: str = '*') → Self
```

在语句关键字后面添加一个或多个表达式，即 SELECT、INSERT、UPDATE 或 DELETE。生成式。

用于支持特定于后端的前缀关键字，例如 MySQL 提供的关键字。

例如：

```py
stmt = table.insert().prefix_with("LOW_PRIORITY", dialect="mysql")

# MySQL 5.7 optimizer hints
stmt = select(table).prefix_with(
    "/*+ BKA(t1) */", dialect="mysql")
```

可以通过多次调用`HasPrefixes.prefix_with()`来指定多个前缀。

参数：

+   `*prefixes` – 文本或`ClauseElement`构造，将在 INSERT、UPDATE 或 DELETE 关键字之后呈现。

+   `dialect` – 可选的字符串方言名称，将仅限制此前缀的渲染为仅该方言。

```py
class sqlalchemy.sql.expression.HasSuffixes
```

**成员**

suffix_with()

```py
method suffix_with(*suffixes: _TextCoercedExpressionArgument[Any], dialect: str = '*') → Self
```

在整个语句后面添加一个或多个表达式。

用于支持某些构造的特定于后端的后缀关键字。

例如：

```py
stmt = select(col1, col2).cte().suffix_with(
    "cycle empno set y_cycle to 1 default 0", dialect="oracle")
```

可以通过多次调用`HasSuffixes.suffix_with()`来指定多个后缀。

参数：

+   `*suffixes` – 文本或`ClauseElement`构造，将在目标子句之后呈现。

+   `dialect` – 可选字符串方言名称，将仅限制此后缀的渲染为仅该方言。

```py
class sqlalchemy.sql.expression.Join
```

表示两个`FromClause`元素之间的`JOIN`构造。

`Join`的公共构造函数是模块级别的`join()`函数，以及任何`FromClause`的`FromClause.join()`方法（例如`Table`）。

另请参见

`join()`

`FromClause.join()`

**成员**

__init__(), description, is_derived_from(), select(), self_group()

**类签名**

类 `sqlalchemy.sql.expression.Join` (`sqlalchemy.sql.roles.DMLTableRole`, `sqlalchemy.sql.expression.FromClause`)

```py
method __init__(left: _FromClauseArgument, right: _FromClauseArgument, onclause: _OnClauseArgument | None = None, isouter: bool = False, full: bool = False)
```

构造一个新的 `Join`。

此处的通常入口点是 `join()` 函数或任何 `FromClause` 对象的 `FromClause.join()` 方法。

```py
attribute description
```

```py
method is_derived_from(fromclause: FromClause | None) → bool
```

如果此 `FromClause` 是从给定的 `FromClause` 派生的，则返回 `True`。

一个示例是一个表的别名是从该表派生的。

```py
method select() → Select
```

从此 `Join` 创建一个 `Select`。

例如：

```py
stmt = table_a.join(table_b, table_a.c.id == table_b.c.a_id)

stmt = stmt.select()
```

以上将产生类似于以下的 SQL 字符串：

```py
SELECT table_a.id, table_a.col, table_b.id, table_b.a_id
FROM table_a JOIN table_b ON table_a.id = table_b.a_id
```

```py
method self_group(against: OperatorType | None = None) → FromGrouping
```

对此 `ClauseElement` 应用“分组”。

子类将重写此方法以返回“分组”构造，即括号。特别是，当“二元”表达式被放置到较大表达式中时，它们会提供自己周围的分组，以及当 `select()` 构造被放置到另一个 `select()` 的 FROM 子句中时。 （请注意，子查询通常应使用 `Select.alias()` 方法创建，因为许多平台要求嵌套的 SELECT 语句具有名称）。

当表达式被组合在一起时，`self_group()` 的应用是自动的 - 最终用户代码不应直接使用此方法。请注意，SQLAlchemy 的子句构造会考虑运算符优先级 - 因此括号可能不是必需的，例如，在表达式 `x OR (y AND z)` 中可能不需要括号 - AND 优先于 OR。

`ClauseElement` 的基本 `self_group()` 方法只返回 self。

```py
class sqlalchemy.sql.expression.Lateral
```

表示 LATERAL 子查询。

此对象可以通过 `lateral()` 模块级函数以及所有 `FromClause` 子类上可用的 `FromClause.lateral()` 方法构造。

虽然 LATERAL 是 SQL 标准的一部分，但目前只有较新版本的 PostgreSQL 提供对该关键字的支持。

另请参阅

LATERAL correlation - 用法概述。

**成员**

inherit_cache

**类签名**

class `sqlalchemy.sql.expression.Lateral` (`sqlalchemy.sql.expression.FromClauseAlias`, `sqlalchemy.sql.expression.LateralFromClause`)

```py
attribute inherit_cache: bool | None = True
```

指示此 `HasCacheKey` 实例是否应使用其直接超类使用的缓存键生成方案。

该属性默认为`None`，表示构造尚未考虑它是否应该参与缓存；这在功能上等同于将值设置为`False`，只是还会发出警告。

如果与对象对应的 SQL 不会根据本类而不是其超类的本地属性而改变，则可以将此标志设置为`True`。

另请参阅

为自定义结构启用缓存支持 - 设置第三方或用户定义的 SQL 结构的 `HasCacheKey.inherit_cache` 属性的一般指南。

```py
class sqlalchemy.sql.expression.ReturnsRows
```

具有可表示行的列概念的 Core 构造的基本类。

虽然 SELECT 语句和 TABLE 是我们在此类别中考虑的主要内容，但 DML（如 INSERT、UPDATE 和 DELETE）也可以指定 RETURNING，这意味着它们可以在 CTE 和其他形式中使用，并且 PostgreSQL 还具有返回行的函数。

新功能：1.4 版本中新增。

**成员**

exported_columns, is_derived_from()

**类签名**

class `sqlalchemy.sql.expression.ReturnsRows` (`sqlalchemy.sql.roles.ReturnsRowsRole`, `sqlalchemy.sql.expression.DQLDMLClauseElement`)

```py
attribute exported_columns
```

一个`ColumnCollection`代表了这个`ReturnsRows`的“导出”列。

“导出”列代表了这个 SQL 构造渲染的`ColumnElement`表达式的集合。有几种主要类型，即 FROM 子句的“FROM 子句列”，比如表、连接或子查询，被“SELECTed”的列，即 SELECT 语句的“列子句”中的列，以及 DML 语句中的 RETURNING 列。

1.4 版本中的新功能。

另请参见

`FromClause.exported_columns`

`SelectBase.exported_columns`

```py
method is_derived_from(fromclause: FromClause | None) → bool
```

如果这个`ReturnsRows`是从给定的`FromClause`“派生”出来的，则返回`True`。

一个示例是，表的别名是从该表派生出来的。

```py
class sqlalchemy.sql.expression.ScalarSelect
```

代表一个标量子查询。

通过调用`SelectBase.scalar_subquery()`方法创建一个`ScalarSelect`。然后，该对象作为 SQL 列表达式参与其他 SQL 表达式，在`ColumnElement`层次结构中。

另请参见

`SelectBase.scalar_subquery()`

标量和相关子查询 - 在 2.0 教程中

**成员**

correlate(), correlate_except(), inherit_cache, self_group(), where()

**类签名**

类`sqlalchemy.sql.expression.ScalarSelect` (`sqlalchemy.sql.roles.InElementRole`, `sqlalchemy.sql.expression.Generative`, `sqlalchemy.sql.expression.GroupedElement`, `sqlalchemy.sql.expression.ColumnElement`)

```py
method correlate(*fromclauses: Literal[None, False] | _FromClauseArgument) → Self
```

返回一个新的 `ScalarSelect`，它将相关联的 FROM 子句与封闭的 `Select` 相关联。

此方法是从底层 `Select.correlate()` 方法镜像过来的。该方法应用 :meth:_sql.Select.correlate` 方法，然后返回一个针对该语句的新 `ScalarSelect`。

新版本 1.4 中新增：以前，`ScalarSelect.correlate()` 方法仅从 `Select` 中可用。

参数：

***fromclauses** – 一个或多个 `FromClause` 构造的列表，或其他兼容的构造（即 ORM 映射的类），用于成为相关集合的一部分。

另请参见

`ScalarSelect.correlate_except()`

标量和相关子查询 - 在 2.0 教程中

```py
method correlate_except(*fromclauses: Literal[None, False] | _FromClauseArgument) → Self
```

返回一个新的 `ScalarSelect`，它将从自动相关过程中省略给定的 FROM 子句。

此方法是从底层 `Select.correlate_except()` 方法镜像过来的。该方法应用 :meth:_sql.Select.correlate_except` 方法，然后返回一个针对该语句的新 `ScalarSelect`。

新版本 1.4 中新增：以前，`ScalarSelect.correlate_except()` 方法仅从 `Select` 中可用。

参数：

***fromclauses** – 一个或多个 `FromClause` 构造的列表，或其他兼容的构造（即 ORM 映射的类），用于成为相关例外集合的一部分。

另请参见

`ScalarSelect.correlate()`

标量和相关子查询 - 在 2.0 教程中

```py
attribute inherit_cache: bool | None = True
```

指示此`HasCacheKey`实例是否应使用其直接超类使用的缓存密钥生成方案。

该属性默认为`None`，表示构造尚未考虑是否适合参与缓存; 这在功能上等同于将值设置为`False`，只是还会发出警告。

如果与此类本地而不是其超类相关的属性的 SQL 不会根据对象更改，则可以在特定类上将此标志设置为`True`。

另请参阅

为自定义结构启用缓存支持 - 设置第三方或用户定义的 SQL 构造的`HasCacheKey.inherit_cache`属性的一般指南。

```py
method self_group(against: OperatorType | None = None) → ColumnElement[Any]
```

对这个`ClauseElement`应用“分组”。

子类覆盖此方法以返回“分组”结构，即括号。 特别是它被“二进制”表达式使用时，用于在放置到较大表达式中时围绕自身提供分组，以及当`select()`构造放置到另一个`select()`的 FROM 子句中时。 （请注意，子查询通常应使用`Select.alias()`方法创建，因为许多平台要求嵌套的 SELECT 语句具有名称）。

随着表达式的组合，`self_group()`的应用是自动的 - 最终用户代码不应直接使用此方法。 请注意，SQLAlchemy 的子句构造会考虑运算符优先级 - 因此可能不需要括号，例如，在表达式`x OR (y AND z)`中 - AND 优先于 OR。

`ClauseElement`的基本`self_group()`方法只返回自身。

```py
method where(crit: _ColumnExpressionArgument[bool]) → Self
```

对此`ScalarSelect`引用的 SELECT 语句应用 WHERE 子句。

```py
class sqlalchemy.sql.expression.Select
```

表示一个`SELECT`语句。

`Select` 对象通常使用 `select()` 函数构建。请参阅该函数以获取详细信息。

另请参阅

`select()`

使用 SELECT 语句 - 在 2.0 教程中

**成员**

__init__(), add_columns(), add_cte(), alias(), as_scalar(), c, column(), column_descriptions, columns_clause_froms, correlate(), correlate_except(), corresponding_column(), cte(), distinct(), except_(), except_all(), execution_options(), exists(), exported_columns, fetch(), filter(), filter_by(), from_statement(), froms, get_children(), get_execution_options(), get_final_froms(), get_label_style(), group_by(), having(), inherit_cache, inner_columns, intersect(), intersect_all(), is_derived_from(), join(), join_from(), label(), lateral(), limit(), offset(), options(), order_by(), outerjoin(), outerjoin_from(), prefix_with(), reduce_columns(), replace_selectable(), scalar_subquery(), select(), select_from(), selected_columns, self_group(), set_label_style(), slice(), subquery(), suffix_with(), union(), union_all(), where(), whereclause, with_for_update(), with_hint(), with_only_columns(), with_statement_hint()

**类签名**

类 `sqlalchemy.sql.expression.Select` (`sqlalchemy.sql.expression.HasPrefixes`, `sqlalchemy.sql.expression.HasSuffixes`, `sqlalchemy.sql.expression.HasHints`, `sqlalchemy.sql.expression.HasCompileState`, `sqlalchemy.sql.expression._SelectFromElements`, `sqlalchemy.sql.expression.GenerativeSelect`, `sqlalchemy.sql.expression.TypedReturnsRows`)

```py
method __init__(*entities: _ColumnsClauseArgument[Any])
```

构造一个新的 `Select`。

`Select` 的公共构造函数是 `select()` 函数。

```py
method add_columns(*entities: _ColumnsClauseArgument[Any]) → Select[Any]
```

返回一个新的 `select()` 构造，其中给定的实体附加到其列子句中。

例如：

```py
my_select = my_select.add_columns(table.c.new_column)
```

列子句中的原始表达式保持不变。要用新表达式替换原始表达式，请参阅方法 `Select.with_only_columns()`。

参数：

***entities** – 要添加到列子句的列、表或其他实体表达式

另请参见

`Select.with_only_columns()` - 替换现有表达式而不是附加。

同时选择多个 ORM 实体 - 以 ORM 为中心的示例

```py
method add_cte(*ctes: CTE, nest_here: bool = False) → Self
```

*继承自* `HasCTE.add_cte()` *方法的* `HasCTE`

将一个或多个 `CTE` 构造添加到此语句中。

此方法将 `CTE` 构造与父语句关联，以便它们将分别无条件地呈现在最终语句的 WITH 子句中，即使在语句或任何子查询中没有其他地方引用它们。

当可选的 `HasCTE.add_cte.nest_here` 参数设置为 True 时，每个给定的 `CTE` 将直接在此语句中渲染为 WITH 子句，而不是被移动到最终渲染的语句的顶部，即使此语句被渲染为较大语句内的子查询也是如此。

此方法有两个一般用途。一个是嵌入某些用途的 CTE 语句，而不需要显式引用，例如将 DML 语句（如 INSERT 或 UPDATE）作为 CTE 内联到可能间接从其结果中获取的主语句中的用例。另一个是提供对一系列特定 CTE 构造的精确放置控制，这些构造应保持直接渲染为可能嵌套在较大语句中的特定语句。

例如：

```py
from sqlalchemy import table, column, select
t = table('t', column('c1'), column('c2'))

ins = t.insert().values({"c1": "x", "c2": "y"}).cte()

stmt = select(t).add_cte(ins)
```

将呈现为：

```py
WITH anon_1 AS
(INSERT INTO t (c1, c2) VALUES (:param_1, :param_2))
SELECT t.c1, t.c2
FROM t
```

上述中，“anon_1” CTE 在 SELECT 语句中没有被引用，但仍完成了运行 INSERT 语句的任务。

类似地，在与 DML 相关的上下文中，使用 PostgreSQL `Insert` 构造来生成“upsert”：

```py
from sqlalchemy import table, column
from sqlalchemy.dialects.postgresql import insert

t = table("t", column("c1"), column("c2"))

delete_statement_cte = (
    t.delete().where(t.c.c1 < 1).cte("deletions")
)

insert_stmt = insert(t).values({"c1": 1, "c2": 2})
update_statement = insert_stmt.on_conflict_do_update(
    index_elements=[t.c.c1],
    set_={
        "c1": insert_stmt.excluded.c1,
        "c2": insert_stmt.excluded.c2,
    },
).add_cte(delete_statement_cte)

print(update_statement)
```

上述语句的渲染结果为：

```py
WITH deletions AS
(DELETE FROM t WHERE t.c1 < %(c1_1)s)
INSERT INTO t (c1, c2) VALUES (%(c1)s, %(c2)s)
ON CONFLICT (c1) DO UPDATE SET c1 = excluded.c1, c2 = excluded.c2
```

新版本 1.4.21 中新增。

参数：

+   `*ctes` –

    零个或多个`CTE` 构造。

    2.0 版本中的更改：接受多个 CTE 实例

+   `nest_here` –

    如果为 True，则给定的 CTE 或 CTE 将渲染为当它们添加到此 `HasCTE` 时指定了 `HasCTE.cte.nesting` 标志为 True。假设给定的 CTE 在外部封闭语句中也没有被引用，则在给定此标志时，这些给定的 CTE 应该在此语句的级别上渲染。

    新版本 2.0 中新增。

    另请参阅

    `HasCTE.cte.nesting`

```py
method alias(name: str | None = None, flat: bool = False) → Subquery
```

*从* `SelectBase.alias()` *方法继承的* `SelectBase`

返回针对此 `SelectBase` 的命名子查询。

对于 `SelectBase`（与 `FromClause` 相对），这返回一个 `Subquery` 对象，其行为大部分与 `FromClause` 中使用的 `Alias` 对象相同。

自 1.4 版更改：`SelectBase.alias()` 方法现在是 `SelectBase.subquery()` 方法的同义词。

```py
method as_scalar() → ScalarSelect[Any]
```

*继承自* `SelectBase.as_scalar()` *方法的* `SelectBase`

自 1.4 版弃用：`SelectBase.as_scalar()` 方法已弃用，将在将来的版本中移除。请参考 `SelectBase.scalar_subquery()`。

```py
attribute c
```

*继承自* `SelectBase.c` *属性的* `SelectBase`

自 1.4 版弃用：`SelectBase.c` 和 `SelectBase.columns` 属性已弃用，将在将来的版本中移除；这些属性隐式创建一个应该显式的子查询。请首先调用 `SelectBase.subquery()` 来创建一个子查询，然后包含此属性。要访问此 SELECT 对象从中选择的列，请使用 `SelectBase.selected_columns` 属性。

```py
method column(column: _ColumnsClauseArgument[Any]) → Select[Any]
```

返回一个带有给定列表达式添加到其列子句的新 `select()` 结构。

自 1.4 版弃用：`Select.column()` 方法已弃用，将在将来的版本中移除。请使用 `Select.add_columns()`

例如：

```py
my_select = my_select.column(table.c.new_column)
```

有关如何添加/替换`Select`对象的列的指南，请参阅`Select.with_only_columns()`的文档。

```py
attribute column_descriptions
```

返回一个插件启用的“列描述”结构，指的是此语句所选的列。

当使用 ORM 时，此属性通常很有用，因为它返回一个包含有关映射实体信息的扩展结构。部分检查来自 ORM 启用的 SELECT 和 DML 语句的实体和列包含更多背景信息。

对于仅限于 Core 的语句，此访问器返回的结构源自`Select.selected_columns`访问器返回的相同对象，格式为包含键`name`、`type`和`expr`的字典列表，这些键指示要选择的列表达式：

```py
>>> stmt = select(user_table)
>>> stmt.column_descriptions
[
 {
 'name': 'id',
 'type': Integer(),
 'expr': Column('id', Integer(), ...)},
 {
 'name': 'name',
 'type': String(length=30),
 'expr': Column('name', String(length=30), ...)}
]
```

1.4.33 版本更改：`Select.column_descriptions`属性返回一个仅限于 Core 的实体集结构，而不仅仅是 ORM 的实体。

另请参阅

`UpdateBase.entity_description` - `insert()`、`update()`或`delete()`的实体信息

检查来自 ORM 启用的 SELECT 和 DML 语句的实体和列 - ORM 背景

```py
attribute columns_clause_froms
```

返回由此 SELECT 语句的列子句暗示的`FromClause`对象集。

版本 1.4.23 中的新功能。

另请参阅

`Select.froms` - 考虑完整语句的“最终”FROM 列表

`Select.with_only_columns()` - 使用此集合设置新的 FROM 列表

```py
method correlate(*fromclauses: Literal[None, False] | _FromClauseArgument) → Self
```

返回一个新的`Select`，它将相关联给定的 FROM 子句与封闭的`Select`。

调用此方法会关闭 `Select` 对象的“自动关联”默认行为。通常，通过其 WHERE 子句、ORDER BY、HAVING 或 columns 子句 对此 `Select` 对象进行了包围的 `Select` 中出现的 FROM 元素将从此 `Select` 对象的 FROM 子句 中省略。使用 `Select.correlate()` 方法设置一个显式的关联集合，提供了一个可能参与此过程的固定 FROM 对象列表。

当使用 `Select.correlate()` 来应用特定的 FROM 子句进行关联时，无论这个 `Select` 对象相对于引用相同 FROM 对象的包围 `Select` 有多深嵌套，FROM 元素都成为关联的候选对象。这与“自动关联”的行为形成对比，后者仅关联到一个直接包围的 `Select`。多级关联确保封闭和包围的 `Select` 之间的链接始终通过至少一个 WHERE/ORDER BY/HAVING/columns 子句以便进行关联。

如果传递了`None`，`Select` 对象将不对其任何 FROM 条目进行关联，所有条目都将无条件地在本地 FROM 子句中呈现。

参数：

***fromclauses** – 一个或多个 `FromClause` 或其他 FROM 兼容的构造，比如 ORM 映射的实体，成为关联集合的一部分；或者传递单个值 `None` 以删除所有现有的关联。

另请参阅

`Select.correlate_except()`

标量和相关子查询

```py
method correlate_except(*fromclauses: Literal[None, False] | _FromClauseArgument) → Self
```

返回一个新的 `Select`，它将从自动关联过程中省略给定的 FROM 子句。

调用 `Select.correlate_except()` 会关闭给定 FROM 元素的 `Select` 对象的默认行为“自动关联”。在此指定的元素将无条件地出现在 FROM 列表中，而所有其他 FROM 元素仍然受到正常的自动关联行为的影响。

如果传入 `None`，或者没有传入参数，`Select` 对象将关联其所有的 FROM 条目。

参数：

***fromclauses** – 一个或多个 `FromClause` 构造的列表，或其他兼容的构造（即 ORM 映射的类），将成为关联例外集合的一部分。

参见

`Select.correlate()`

标量和相关子查询

```py
method corresponding_column(column: KeyedColumnElement[Any], require_embedded: bool = False) → KeyedColumnElement[Any] | None
```

*继承自* `Selectable.corresponding_column()` *方法的* `Selectable`

给定一个 `ColumnElement`，从此 `Selectable` 的导出 `ColumnElement` 集合中返回与原始 `ColumnElement` 对应的对象，通过一个共同的祖先列。

参数：

+   `column` – 目标 `ColumnElement` 要匹配的列。

+   `require_embedded` – 仅在给定的 `ColumnElement` 实际存在于此 `Selectable` 的子元素中时，才返回对应的列。通常，如果列仅与此 `Selectable` 的导出列之一共享一个共同的祖先，列就会匹配。

参见

`Selectable.exported_columns` - 用于操作的 `ColumnCollection`。

`ColumnCollection.corresponding_column()` - 实现方法。

```py
method cte(name: str | None = None, recursive: bool = False, nesting: bool = False) → CTE
```

*继承自* `HasCTE` *的* `HasCTE.cte()` *方法

返回一个新的 `CTE`，或 Common Table Expression 实例。

公共表达式是 SQL 标准，SELECT 语句可以使用一个称为 “WITH” 的子句与主要语句一起指定的辅助语句。特殊的 UNION 语义也可以被使用，以允许 “递归” 查询，其中 SELECT 语句可以从先前选择的行集中进行选择。

CTE 也可以应用于某些数据库上的 DML 构造 UPDATE、INSERT 和 DELETE，既可以作为与 RETURNING 结合使用时 CTE 行的来源，也可以作为 CTE 行的消费者。

SQLAlchemy 检测到 `CTE` 对象，这些对象与 `Alias` 对象类似，被视为要传递到语句的 FROM 子句以及语句顶部 WITH 子句的特殊元素。

对于诸如 PostgreSQL 的 “MATERIALIZED” 和 “NOT MATERIALIZED” 等特殊前缀，可以使用 `CTE.prefix_with()` 方法来建立这些前缀。

在版本 1.3.13 中更改：增加了对前缀的支持。特别是 - MATERIALIZED 和 NOT MATERIALIZED。

参数：

+   `name` – 给公共表达式的名称。与 `FromClause.alias()` 类似，如果名称为 `None`，则在查询编译时将使用匿名符号。

+   `recursive` – 如果为 `True`，将呈现 `WITH RECURSIVE`。递归公共表达式旨在与 UNION ALL 结合使用，以从已经选择的行中导出行。

+   `nesting` –

    如果为 `True`，将在引用它的语句中本地呈现 CTE。对于更复杂的场景，可以使用 `HasCTE.add_cte()` 方法，并使用 `HasCTE.add_cte.nest_here` 参数来更精确地控制特定 CTE 的确切位置。

    在版本 1.4.24 中新增。

    另请参阅

    `HasCTE.add_cte()`

以下示例包括两个来自 PostgreSQL 文档的示例[`www.postgresql.org/docs/current/static/queries-with.html`](https://www.postgresql.org/docs/current/static/queries-with.html)，以及其他示例。

示例 1，非递归：

```py
from sqlalchemy import (Table, Column, String, Integer,
                        MetaData, select, func)

metadata = MetaData()

orders = Table('orders', metadata,
    Column('region', String),
    Column('amount', Integer),
    Column('product', String),
    Column('quantity', Integer)
)

regional_sales = select(
                    orders.c.region,
                    func.sum(orders.c.amount).label('total_sales')
                ).group_by(orders.c.region).cte("regional_sales")

top_regions = select(regional_sales.c.region).\
        where(
            regional_sales.c.total_sales >
            select(
                func.sum(regional_sales.c.total_sales) / 10
            )
        ).cte("top_regions")

statement = select(
            orders.c.region,
            orders.c.product,
            func.sum(orders.c.quantity).label("product_units"),
            func.sum(orders.c.amount).label("product_sales")
    ).where(orders.c.region.in_(
        select(top_regions.c.region)
    )).group_by(orders.c.region, orders.c.product)

result = conn.execute(statement).fetchall()
```

示例 2，WITH RECURSIVE：

```py
from sqlalchemy import (Table, Column, String, Integer,
                        MetaData, select, func)

metadata = MetaData()

parts = Table('parts', metadata,
    Column('part', String),
    Column('sub_part', String),
    Column('quantity', Integer),
)

included_parts = select(\
    parts.c.sub_part, parts.c.part, parts.c.quantity\
    ).\
    where(parts.c.part=='our part').\
    cte(recursive=True)

incl_alias = included_parts.alias()
parts_alias = parts.alias()
included_parts = included_parts.union_all(
    select(
        parts_alias.c.sub_part,
        parts_alias.c.part,
        parts_alias.c.quantity
    ).\
    where(parts_alias.c.part==incl_alias.c.sub_part)
)

statement = select(
            included_parts.c.sub_part,
            func.sum(included_parts.c.quantity).
              label('total_quantity')
        ).\
        group_by(included_parts.c.sub_part)

result = conn.execute(statement).fetchall()
```

示例 3，使用 CTE 进行更新和插入：

```py
from datetime import date
from sqlalchemy import (MetaData, Table, Column, Integer,
                        Date, select, literal, and_, exists)

metadata = MetaData()

visitors = Table('visitors', metadata,
    Column('product_id', Integer, primary_key=True),
    Column('date', Date, primary_key=True),
    Column('count', Integer),
)

# add 5 visitors for the product_id == 1
product_id = 1
day = date.today()
count = 5

update_cte = (
    visitors.update()
    .where(and_(visitors.c.product_id == product_id,
                visitors.c.date == day))
    .values(count=visitors.c.count + count)
    .returning(literal(1))
    .cte('update_cte')
)

upsert = visitors.insert().from_select(
    [visitors.c.product_id, visitors.c.date, visitors.c.count],
    select(literal(product_id), literal(day), literal(count))
        .where(~exists(update_cte.select()))
)

connection.execute(upsert)
```

示例 4，嵌套 CTE（SQLAlchemy 1.4.24 及以上版本）：

```py
value_a = select(
    literal("root").label("n")
).cte("value_a")

# A nested CTE with the same name as the root one
value_a_nested = select(
    literal("nesting").label("n")
).cte("value_a", nesting=True)

# Nesting CTEs takes ascendency locally
# over the CTEs at a higher level
value_b = select(value_a_nested.c.n).cte("value_b")

value_ab = select(value_a.c.n.label("a"), value_b.c.n.label("b"))
```

上述查询将第二个 CTE 嵌套在第一个内部，并显示为内联参数如下：

```py
WITH
    value_a AS
        (SELECT 'root' AS n),
    value_b AS
        (WITH value_a AS
            (SELECT 'nesting' AS n)
        SELECT value_a.n AS n FROM value_a)
SELECT value_a.n AS a, value_b.n AS b
FROM value_a, value_b
```

可以使用 `HasCTE.add_cte()` 方法设置相同的 CTE，如下所示（SQLAlchemy 2.0 及以上版本）：

```py
value_a = select(
    literal("root").label("n")
).cte("value_a")

# A nested CTE with the same name as the root one
value_a_nested = select(
    literal("nesting").label("n")
).cte("value_a")

# Nesting CTEs takes ascendency locally
# over the CTEs at a higher level
value_b = (
    select(value_a_nested.c.n).
    add_cte(value_a_nested, nest_here=True).
    cte("value_b")
)

value_ab = select(value_a.c.n.label("a"), value_b.c.n.label("b"))
```

示例 5，非线性 CTE（SQLAlchemy 1.4.28 及以上版本）：

```py
edge = Table(
    "edge",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("left", Integer),
    Column("right", Integer),
)

root_node = select(literal(1).label("node")).cte(
    "nodes", recursive=True
)

left_edge = select(edge.c.left).join(
    root_node, edge.c.right == root_node.c.node
)
right_edge = select(edge.c.right).join(
    root_node, edge.c.left == root_node.c.node
)

subgraph_cte = root_node.union(left_edge, right_edge)

subgraph = select(subgraph_cte)
```

上述查询将在递归 CTE 中呈现 2 个 UNION：

```py
WITH RECURSIVE nodes(node) AS (
        SELECT 1 AS node
    UNION
        SELECT edge."left" AS "left"
        FROM edge JOIN nodes ON edge."right" = nodes.node
    UNION
        SELECT edge."right" AS "right"
        FROM edge JOIN nodes ON edge."left" = nodes.node
)
SELECT nodes.node FROM nodes
```

另请参阅

`Query.cte()` - `HasCTE.cte()` 的 ORM 版本。

```py
method distinct(*expr: _ColumnExpressionArgument[Any]) → Self
```

返回一个新的 `select()` 结构，该结构将整体应用 DISTINCT 到 SELECT 语句。

例如：

```py
from sqlalchemy import select
stmt = select(users_table.c.id, users_table.c.name).distinct()
```

上述将生成类似于的语句：

```py
SELECT DISTINCT user.id, user.name FROM user
```

该方法还接受一个 `*expr` 参数，该参数生成特定于 PostgreSQL 方言的 `DISTINCT ON` 表达式。在不支持此语法的其他后端上使用此参数将引发错误。

参数：

***expr** –

可选的列表达式。当存在时，PostgreSQL 方言将呈现 `DISTINCT ON (<expressions>)` 结构。在其他后端上会引发弃用警告和/或 `CompileError`。

自 1.4 版弃用：在其他方言中使用 *expr 已弃用，并将在将来的版本中引发 `CompileError`。

```py
method except_(*other: _SelectStatementForCompoundArgument) → CompoundSelect
```

返回此 select() 结构相对于以位置参数提供的给定可选择对象的 SQL `EXCEPT`。

参数：

***other** –

一个或多个用于创建 UNION 的元素。

自 1.4.28 版更改：现在接受多个元素。

```py
method except_all(*other: _SelectStatementForCompoundArgument) → CompoundSelect
```

返回此 select() 结构相对于以位置参数提供的给定可选择对象的 SQL `EXCEPT ALL`。

参数：

***other** –

一个或多个用于创建 UNION 的元素。

自 1.4.28 版更改：现在接受多个元素。

```py
method execution_options(**kw: Any) → Self
```

*继承自* `Executable.execution_options()` *方法的* `Executable`

设置在执行期间生效的语句的非 SQL 选项。

执行选项可以在许多范围内设置，包括每个语句、每个连接或每个执行，使用诸如`Connection.execution_options()`和接受选项字典的参数的方法，例如`Connection.execute.execution_options`和`Session.execute.execution_options`。

与 ORM 加载器选项等其他类型的选项不同，执行选项的主要特征在于**执行选项从不影响查询的编译 SQL，只影响 SQL 语句本身如何被调用或结果如何被提取**。也就是说，执行选项不是 SQL 编译所容纳的部分，也不被视为语句的缓存状态的一部分。

`Executable.execution_options()`方法是生成式的，就像适用于`Engine`和`Query`对象的方法一样，这意味着当调用该方法时，将返回对象的副本，该副本将给定参数应用于该新副本，但不更改原始对象：

```py
statement = select(table.c.x, table.c.y)
new_statement = statement.execution_options(my_option=True)
```

对此行为的一个例外是`Connection`对象，在这里`Connection.execution_options()`方法明确地**不**是生成式的。

可以传递给`Executable.execution_options()`及其他相关方法和参数字典的选项类型包括 SQLAlchemy Core 或 ORM 明确消耗的参数，以及 SQLAlchemy 未定义的任意关键字参数，这意味着这些方法和/或参数字典可用于与自定义代码交互的用户定义参数，可以使用诸如`Executable.get_execution_options()`和`Connection.get_execution_options()`等方法访问这些参数，或者在选择的事件钩子中使用专用的`execution_options`事件参数，例如`ConnectionEvents.before_execute.execution_options`或`ORMExecuteState.execution_options`，例如：

```py
from sqlalchemy import event

@event.listens_for(some_engine, "before_execute")
def _process_opt(conn, statement, multiparams, params, execution_options):
    "run a SQL function before invoking a statement"

    if execution_options.get("do_special_thing", False):
        conn.exec_driver_sql("run_special_function()")
```

在 SQLAlchemy 明确识别的选项范围内，大多数适用于特定类别的对象，而不适用于其他对象。最常见的执行选项包括：

+   `Connection.execution_options.isolation_level` - 通过`Engine`设置连接或一类连接的隔离级别。此选项仅由`Connection`或`Engine`接受。

+   `Connection.execution_options.stream_results` - 表示应使用服务器端游标获取结果；此选项被 `Connection`、`Connection.execute.execution_options` 上的参数 `Connection.execute()` 以及 SQL 语句对象上的 `Executable.execution_options()` 接受，以及 ORM 构造如 `Session.execute()`。

+   `Connection.execution_options.compiled_cache` - 表示将用作 `Connection` 或 `Engine` 的 SQL 编译缓存，以及像 `Session.execute()` 这样的 ORM 方法的字典。 可以将其传递为 `None` 以禁用语句的缓存。 由于在语句对象中携带编译缓存是不可取的，因此此选项不被 `Executable.execution_options()` 接受。

+   `Connection.execution_options.schema_translate_map` - 一个由 模式转换映射 功能使用的模式名称映射，被 `Connection`、`Engine`、`Executable`接受，以及 ORM 构造如 `Session.execute()`。

亦参见

`Connection.execution_options()` - `Connection.execution_options()`方法。

`Connection.execute.execution_options` - `Connection.execute.execution_options` 方法。

`Session.execute.execution_options`

ORM 执行选项 - 关于所有 ORM 特定执行选项的文档

```py
method exists() → Exists
```

*继承自* `SelectBase` *的* `SelectBase.exists()` *方法*

返回此可选择项的 `Exists` 表示，可用作列表达式。

返回对象是 `Exists` 的一个实例。

另请参阅

`exists()`

EXISTS 子查询 - 在 2.0 风格 教程中。

新版本 1.4 中引入。

```py
attribute exported_columns
```

*继承自* `SelectBase` *的* `SelectBase.exported_columns` *属性*

一个表示此 `Selectable` 的“导出”列的 `ColumnCollection`，不包括 `TextClause` 构造。

`SelectBase` 对象的“导出”列与 `SelectBase.selected_columns` 集合是同义词。

新版本 1.4 中引入。

另请参阅

`Select.exported_columns`

`Selectable.exported_columns`

`FromClause.exported_columns`

```py
method fetch(count: _LimitOffsetType, with_ties: bool = False, percent: bool = False) → Self
```

*继承自* `GenerativeSelect` *的* `GenerativeSelect.fetch()` *方法*

返回一个应用了给定 `FETCH FIRST` 准则的新可选择项。

这是一个数值，通常在生成的选择中呈现为 `FETCH {FIRST | NEXT} [ count ] {ROW | ROWS} {ONLY | WITH TIES}` 表达式。此功能目前已在 Oracle、PostgreSQL、MSSQL 中实现。

使用 `GenerativeSelect.offset()` 来指定偏移量。

注意

`GenerativeSelect.fetch()` 方法将用 `GenerativeSelect.limit()` 应用的任何子句替换。

1.4 版本中的新功能。

参数：

+   `count` – 一个整数 COUNT 参数，或者提供整数结果的 SQL 表达式。当 `percent=True` 时，这将表示要返回的行数的百分比，而不是绝对值。传递 `None` 来重置它。

+   `with_ties` – 当为 `True` 时，使用 WITH TIES 选项来返回与结果集中最后一位并列的任何额外行，根据 `ORDER BY` 子句确定。在这种情况下，`ORDER BY` 可能是强制性的。默认为 `False`。

+   `percent` – 当为 `True` 时，`count` 表示要返回的所选行的总数的百分比。默认为 `False`。

另请参见

`GenerativeSelect.limit()`

`GenerativeSelect.offset()`

```py
method filter(*criteria: _ColumnExpressionArgument[bool]) → Self
```

`Select.where()` 方法的同义词。

```py
method filter_by(**kwargs: Any) → Self
```

将给定的过滤条件作为 WHERE 子句应用到此选择语句中。

```py
method from_statement(statement: ReturnsRowsRole) → ExecutableReturnsRows
```

将此 `Select` 所选的列应用到另一个语句中。

此操作是特定于插件的，如果此 `Select` 不从启用插件的实体中进行选择，则会引发不受支持的异常。

该语句通常是一个 `text()` 或 `select()` 结构，并且应返回适合于此 `Select` 表示的实体的列集。

另请参见

从文本语句获取 ORM 结果 - 在 ORM 查询指南中的用法示例

```py
attribute froms
```

返回显示的 `FromClause` 元素列表。

自版本 1.4.23 起已弃用：`Select.froms` 属性已移至 `Select.get_final_froms()` 方法。

```py
method get_children(**kw: Any) → Iterable[ClauseElement]
```

返回此 `HasTraverseInternals` 的直接子元素。

用于访问遍历。

**kw 可包含更改返回集合的标志，例如返回子集以减少更大的遍历，或者从不同上下文返回子项（例如模式级集合而不是子句级集合）。

```py
method get_execution_options() → _ExecuteOptions
```

*继承自* `Executable.get_execution_options()` *方法的* `Executable`

获取在执行期间生效的非 SQL 选项。

版本 1.3 中的新功能。

另请参阅

`Executable.execution_options()`

```py
method get_final_froms() → Sequence[FromClause]
```

计算最终显示的 `FromClause` 元素列表。

此方法将运行完整的计算，以确定在生成的 SELECT 语句中将显示哪些 FROM 元素，包括使用 JOIN 对象遮蔽单个表以及用于 ORM 使用情况的完整计算，包括急加载子句。

对于 ORM 使用，此访问器返回**编译后**的 FROM 对象列表；此集合将包括诸如急加载表和连接等元素。这些对象将**不**启用 ORM，并且不能作为`Select.select_froms()` 集合的替代品；此外，该方法对于启用 ORM 的语句性能不佳，因为它将导致完整的 ORM 构建过程。

要检索最初传递给 `Select` 的“columns”集合隐含的 FROM 列表，请使用 `Select.columns_clause_froms` 访问器。

若要从备选列集中选择，同时保持 FROM 列表，请使用 `Select.with_only_columns()` 方法，并传递 `Select.with_only_columns.maintain_column_froms` 参数。

从版本 1.4.23 开始：- `Select.get_final_froms()` 方法取代了之前的 `Select.froms` 访问器，该访问器已弃用。

参见

`Select.columns_clause_froms`

```py
method get_label_style() → SelectLabelStyle
```

*继承自* `GenerativeSelect` *的* `GenerativeSelect.get_label_style()` *方法。

检索当前标签样式。

从版本 1.4 开始新增。

```py
method group_by(_GenerativeSelect__first: Literal[None, _NoArg.NO_ARG] | _ColumnExpressionOrStrLabelArgument[Any] = _NoArg.NO_ARG, *clauses: _ColumnExpressionOrStrLabelArgument[Any]) → Self
```

*继承自* `GenerativeSelect` *的* `GenerativeSelect.group_by()` *方法。

返回一个应用了给定的 GROUP BY 准则列表的新可选择的。

可以通过传递 `None` 来抑制所有现有的 GROUP BY 设置。

例如：

```py
stmt = select(table.c.name, func.max(table.c.stat)).\
group_by(table.c.name)
```

参数：

***clauses** – 一系列将用于生成 GROUP BY 子句的 `ColumnElement` 结构。

参见

带有 GROUP BY / HAVING 的聚合函数 - 在 SQLAlchemy 统一教程中

按标签排序或分组 - 在 SQLAlchemy 统一教程中

```py
method having(*having: _ColumnExpressionArgument[bool]) → Self
```

返回一个带有给定表达式添加到其 HAVING 子句的新 `select()` 结构，如果存在的话，通过 AND 连接到现有子句。

```py
attribute inherit_cache: bool | None = None
```

*继承自* `HasCacheKey` *的* `HasCacheKey.inherit_cache` *属性。

指示此 `HasCacheKey` 实例是否应使用其直接超类使用的缓存密钥生成方案。

该属性默认为 `None`，表示构造尚未考虑是否应参与缓存；这在功能上等同于将值设置为 `False`，除了还会发出警告。

如果对应于对象的 SQL 不基于仅属于该类而不是其超类的属性更改，则可以将此标志设置为`True`。

参见

为自定义构造启用缓存支持 - 设置第三方或用户定义的 SQL 构造的`HasCacheKey.inherit_cache`属性的一般指南。

```py
attribute inner_columns
```

将呈现为生成的 SELECT 语句的列子句的所有`ColumnElement`表达式的迭代器。

该方法在 1.4 版本中已过时，并被`Select.exported_columns`集合取代。

```py
method intersect(*other: _SelectStatementForCompoundArgument) → CompoundSelect
```

返回此 select()构造与作为位置参数提供的给定 selectables 的 SQL `INTERSECT`。

参数：

+   `*other` –

    一个或多个用于创建联合的元素。

    从版本 1.4.28 开始更改：现在接受多个元素。

+   `**kwargs` – 关键字参数被转发到新创建的`CompoundSelect`对象的构造函数。

```py
method intersect_all(*other: _SelectStatementForCompoundArgument) → CompoundSelect
```

返回此 select()构造与作为位置参数提供的给定 selectables 的 SQL `INTERSECT ALL`。

参数：

+   `*other` –

    一个或多个用于创建联合的元素。

    从版本 1.4.28 开始更改：现在接受多个元素。

+   `**kwargs` – 关键字参数被转发到新创建的`CompoundSelect`对象的构造函数。

```py
method is_derived_from(fromclause: FromClause | None) → bool
```

如果此`ReturnsRows`‘派生’自给定的`FromClause`，则返回`True`。

例如，Table 的 Alias 源自该 Table。

```py
method join(target: _JoinTargetArgument, onclause: _OnClauseArgument | None = None, *, isouter: bool = False, full: bool = False) → Self
```

对此`Select`对象的条件进行 SQL JOIN，并适用生成，返回新生成的`Select`。

例如：

```py
stmt = select(user_table).join(address_table, user_table.c.id == address_table.c.user_id)
```

上述语句生成类似于以下的 SQL：

```py
SELECT user.id, user.name FROM user JOIN address ON user.id = address.user_id
```

1.4 版更改：`Select.join()`现在在现有 SELECT 的 FROM 子句中创建一个`FromClause`源和给定目标`FromClause`之间创建一个`Join`对象，然后将此`Join`添加到新生成的 SELECT 语句的 FROM 子句中。这与 1.3 中的行为完全不同，1.3 中的行为是创建整个`Select`的子查询，然后将该子查询连接到目标。

这是一个**不向后兼容的更改**，因为以前的行为大多是无用的，在任何情况下都会产生一个未命名的子查询，大多数数据库都会拒绝。新行为的模型是根据 ORM 中非常成功的`Query.join()`方法的行为建模的，以支持通过使用具有`Session`的`Select`对象来使用`Query`的功能。

有关此更改的说明，请参见 select().join() 和 outerjoin() 将 JOIN 条件添加到当前查询，而不是创建子查询。

参数：

+   `target` – 连接的目标表

+   `onclause` – 连接的 ON 子句。如果省略，则根据两个表之间的`ForeignKey`关系自动生成 ON 子句，如果可以明确确定，则会生成 ON 子句，否则会引发错误。

+   `isouter` – 如果为 True，则生成 LEFT OUTER 连接。与`Select.outerjoin()`相同。

+   `full` – 如果为 True，则生成 FULL OUTER 连接。

另请参阅

显式的 FROM 子句和 JOINs - 在 SQLAlchemy 统一教程中

Joins - 在 ORM 查询指南中

`Select.join_from()`

`Select.outerjoin()`

```py
method join_from(from_: _FromClauseArgument, target: _JoinTargetArgument, onclause: _OnClauseArgument | None = None, *, isouter: bool = False, full: bool = False) → Self
```

针对此 `Select` 对象的条件创建 SQL JOIN，并应用生成，返回新生成的 `Select`。

例如：

```py
stmt = select(user_table, address_table).join_from(
    user_table, address_table, user_table.c.id == address_table.c.user_id
)
```

上述语句生成类似于 SQL 的内容：

```py
SELECT user.id, user.name, address.id, address.email, address.user_id
FROM user JOIN address ON user.id = address.user_id
```

新版本 1.4 中新增。

参数：

+   `from_` – 连接的左侧，将在 FROM 子句中呈现，并且大致相当于使用 `Select.select_from()` 方法。

+   `target` – 要加入的目标表

+   `onclause` – 连接的 ON 子句。

+   `isouter` – 如果为 True，则生成 LEFT OUTER 连接。与 `Select.outerjoin()` 相同。

+   `full` – 如果为 True，则生成 FULL OUTER 连接。

另请参阅

显式 FROM 子句和 JOINs - 在 SQLAlchemy 统一教程 中

连接 - 在 ORM 查询指南 中

`Select.join()`

```py
method label(name: str | None) → Label[Any]
```

*继承自* `SelectBase.label()` *方法的* `SelectBase`

返回此可选择项的“标量”表示，嵌入为带有标签的子查询。

另请参阅

`SelectBase.scalar_subquery()`。

```py
method lateral(name: str | None = None) → LateralFromClause
```

*继承自* `SelectBase.lateral()` *方法的* `SelectBase`

返回此 `Selectable` 的 LATERAL 别名。

返回值是由顶层 `lateral()` 函数提供的 `Lateral` 结构。

另请参阅

LATERAL correlation - 用法概述。

```py
method limit(limit: _LimitOffsetType) → Self
```

*继承自* `GenerativeSelect.limit()` *方法的* `GenerativeSelect`

返回一个应用给定 LIMIT 条件的新可选择项。

这是一个通常呈现为 `LIMIT` 表达式的数值。不支持 `LIMIT` 的后端将尝试提供类似的功能。

注意

`GenerativeSelect.limit()` 方法将替换应用的任何子句与 `GenerativeSelect.fetch()`。

参数：

**limit** – 整数 LIMIT 参数，或提供整数结果的 SQL 表达式。传递 `None` 以重置它。

另请参阅

`GenerativeSelect.fetch()`

`GenerativeSelect.offset()`

```py
method offset(offset: _LimitOffsetType) → Self
```

*继承自* `GenerativeSelect.offset()` *方法的* `GenerativeSelect`

返回应用了给定 OFFSET 准则的新可选择对象。

这是一个通常呈现为`OFFSET`表达式的数字值在结果选择中。不支持 `OFFSET` 的后端将尝试提供类似的功能。

参数：

**offset** – 整数 OFFSET 参数，或提供整数结果的 SQL 表达式。传递 `None` 以重置它。

另请参阅

`GenerativeSelect.limit()`

`GenerativeSelect.fetch()`

```py
method options(*options: ExecutableOption) → Self
```

*继承自* `Executable.options()` *方法的* `Executable`

对此语句应用选项。

从一般意义上说，选项是任何可以由 SQL 编译器解释为语句的 Python 对象。这些选项可以被特定的方言或特定类型的编译器消耗。

最常见的选项类型是应用“预加载”和其他加载行为到 ORM 查询的 ORM 级别选项。然而，选项理论上可以用于许多其他目的。

有关特定类型语句的特定类型选项的背景，请参阅这些选项对象的文档。

从版本 1.4 开始更改：- 向核心语句对象添加了`Executable.options()`，以实现统一的核心/ORM 查询功能的目标。

另请参阅

列加载选项 - 指特定于 ORM 查询使用的选项

带有加载器选项的关系加载 - 指的是特定于 ORM 查询使用的选项

```py
method order_by(_GenerativeSelect__first: Literal[None, _NoArg.NO_ARG] | _ColumnExpressionOrStrLabelArgument[Any] = _NoArg.NO_ARG, *clauses: _ColumnExpressionOrStrLabelArgument[Any]) → Self
```

*继承自* `GenerativeSelect.order_by()` *方法的* `GenerativeSelect`

返回一个应用了给定的 ORDER BY 条件列表的新的可选查询。

例如：

```py
stmt = select(table).order_by(table.c.id, table.c.name)
```

多次调用此方法等同于一次将所有子句连接起来调用一次。通过单独传递 `None` 可以取消所有现有的 ORDER BY 条件。然后可以通过再次调用 `Query.order_by()` 来添加新的 ORDER BY 条件，例如：

```py
# will erase all ORDER BY and ORDER BY new_col alone
stmt = stmt.order_by(None).order_by(new_col)
```

参数：

***clauses** – 一系列将用于生成 ORDER BY 子句的 `ColumnElement` 构造。

另请参阅

ORDER BY - 在 SQLAlchemy 统一教程中

按标签排序或分组 - 在 SQLAlchemy 统一教程中

```py
method outerjoin(target: _JoinTargetArgument, onclause: _OnClauseArgument | None = None, *, full: bool = False) → Self
```

创建一个左外连接。

参数与 `Select.join()` 相同。

从版本 1.4 开始变更：`Select.outerjoin()` 现在在现有 SELECT 的 FROM 子句中创建一个 `Join` 对象，以及给定的目标 `FromClause`，然后将这个 `Join` 添加到新生成的 SELECT 语句的 FROM 子句中。这完全重写自 1.3 中的行为，1.3 中的行为会创建整个 `Select` 的子查询，然后将该子查询与目标连接。

这是一个**向后不兼容的更改**，因为先前的行为大多无用，生成的未命名子查询在大多数情况下被大多数数据库拒绝。新行为是在 ORM 中非常成功的`Query.join()`方法之后建模的，以支持通过使用带有`Session`的`Select`对象来使用`Query`的功能。

请参阅此更改的说明：select().join() and outerjoin() add JOIN criteria to the current query, rather than creating a subquery。

另请参阅

明确的 FROM 子句和 JOIN - 在 SQLAlchemy 统一教程中

连接 - 在 ORM 查询指南中

`Select.join()`

```py
method outerjoin_from(from_: _FromClauseArgument, target: _JoinTargetArgument, onclause: _OnClauseArgument | None = None, *, full: bool = False) → Self
```

对这个`Select` 对象的标准创建一个 SQL LEFT OUTER JOIN，并应用生成，返回新生成的`Select`。

用法与`Select.join_from()`相同。

```py
method prefix_with(*prefixes: _TextCoercedExpressionArgument[Any], dialect: str = '*') → Self
```

*继承自* `HasPrefixes.prefix_with()` *方法的* `HasPrefixes`

在语句关键字后添加一个或多个表达式，即 SELECT、INSERT、UPDATE 或 DELETE。生成的。

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

+   `*prefixes` – 文本或`ClauseElement` 结构，它将在 INSERT、UPDATE 或 DELETE 关键字后呈现。

+   `dialect` – 可选的字符串方言名称，将限制将此前缀呈现为仅该方言。

```py
method reduce_columns(only_synonyms: bool = True) → Select
```

返回一个新的`select()` 结构，从列子句中移除了冗余命名且值等价的列。

这里的“冗余”意味着两列，其中一列基于外键或通过语句的 WHERE 子句中的简单等式比较引用另一列。此方法的主要目的是自动构造一个具有所有唯一命名列的选择语句，而无需像`Select.set_label_style()`那样使用表限定标签。

当根据外键省略列时，被引用的列是被保留的列。当根据 WHERE 等价性省略列时，列子句中的第一列是被保留的列。

参数：

**only_synonyms** – 当为 True 时，限制列的移除只针对与等效列具有相同名称的列。否则，所有与其他列等效的列都将被移除。

```py
method replace_selectable(old: FromClause, alias: Alias) → Self
```

*继承自* `Selectable` *的* `Selectable.replace_selectable()` *方法*

用给定的`Alias`对象替换此`FromClause`中所有`FromClause` ‘old’的所有出现，并返回此`FromClause`的副本。

从版本 1.4 开始已弃用：`Selectable.replace_selectable()`方法已弃用，并将在将来的版本中删除。类似的功能可以通过 sqlalchemy.sql.visitors 模块获得。

```py
method scalar_subquery() → ScalarSelect[Any]
```

*继承自* `SelectBase` *的* `SelectBase.scalar_subquery()` *方法*

返回此可选择的‘标量’表示形式，可用作列表达式。

返回的对象是`ScalarSelect`的实例。

通常，列子句中只有一个列的选择语句可以被用作标量表达式。然后标量子查询可以在封闭 SELECT 的 WHERE 子句或列子句中使用。

请注意，标量子查询与使用`SelectBase.subquery()`方法生成的 FROM 级子查询有所不同。

另请参阅

标量和相关子查询 - 在 2.0 教程中

```py
method select(*arg: Any, **kw: Any) → Select
```

*从* `SelectBase.select()` *方法继承自* `SelectBase`

从版本 1.4 开始弃用：`SelectBase.select()`方法已弃用，并将在将来的版本中删除；此方法隐式创建一个子查询，应明确。请先调用`SelectBase.subquery()`以创建子查询，然后可以选择它。

```py
method select_from(*froms: _FromClauseArgument) → Self
```

返回一个新的`select()`构造，其中包含给定的 FROM 表达式合并到其 FROM 对象列表中。

例如：

```py
table1 = table('t1', column('a'))
table2 = table('t2', column('b'))
s = select(table1.c.a).\
    select_from(
        table1.join(table2, table1.c.a==table2.c.b)
    )
```

“from”列表是每个元素标识的唯一集合，因此添加一个已经存在的`Table`或其他可选择的内容将不会产生影响。传递一个指向已经存在的`Table`或其他可选择的内容的`Join`将使该可选择的内容作为单独的元素隐藏在渲染的 FROM 列表中，而不是将其渲染到 JOIN 子句中。

虽然`Select.select_from()`的典型目的是用联接替换默认的派生 FROM 子句，但也可以根据需要多次调用，传入单个表元素，如果 FROM 子句无法完全从列子句中派生：

```py
select(func.count('*')).select_from(table1)
```

```py
attribute selected_columns
```

一个`ColumnCollection`代表此 SELECT 语句或类似结构返回的结果集中的列，不包括`TextClause`构造。

该集合与`FromClause.columns`集合不同，因为此集合中的列不能直接嵌套在另一个 SELECT 语句中；必须首先应用子查询，该子查询提供了 SQL 所需的必要括号化。

对于一个 `select()` 构造，这里的集合正是在“SELECT”语句中渲染的内容，而 `ColumnElement` 对象会直接按照它们给出的方式出现，例如：

```py
col1 = column('q', Integer)
col2 = column('p', Integer)
stmt = select(col1, col2)
```

在上面，`stmt.selected_columns` 将是一个直接包含 `col1` 和 `col2` 对象的集合。对于针对 `Table` 或其他 `FromClause` 的语句，该集合将使用在 from 元素的 `FromClause.c` 集合中的 `ColumnElement` 对象。

`Select.selected_columns` 集合的一个用例是允许在添加额外条件时引用现有列，例如：

```py
def filter_on_id(my_select, id):
    return my_select.where(my_select.selected_columns['id'] == id)

stmt = select(MyModel)

# adds "WHERE id=:param" to the statement
stmt = filter_on_id(stmt, 42)
```

注意

`Select.selected_columns` 集合不包括在列子句中使用 `text()` 构造建立的表达式；这些表达式会被静默地从集合中省略。要在 `Select` 构造内部使用纯文本列表达式，请使用 `literal_column()` 构造。

1.4 版本中的新功能。

```py
method self_group(against: OperatorType | None = None) → SelectStatementGrouping | Self
```

对这个 `ClauseElement` 应用一个“分组”。

这个方法会被子类重写以返回一个“分组”构造，即括号。特别是它被“二元”表达式使用，当这些表达式被放置到较大表达式中时，提供一个围绕自己的分组，以及当 `select()` 构造被放置到另一个 `select()` 的 FROM 子句中时。（注意，子查询通常应该使用 `Select.alias()` 方法创建，因为许多平台要求嵌套的 SELECT 语句必须有名称）。

随着表达式的组合，`self_group()` 方法的应用是自动的 - 最终用户代码不应该直接使用这个方法。请注意，SQLAlchemy 的子句构造会考虑运算符的优先级 - 因此在像 `x OR (y AND z)` 这样的表达式中可能不需要括号 - AND 优先于 OR。

`ClauseElement` 的基本 `self_group()` 方法只返回自身。

```py
method set_label_style(style: SelectLabelStyle) → Self
```

*继承自* `GenerativeSelect.set_label_style()` *方法的* `GenerativeSelect`

返回一个具有指定标签样式的新可选择对象。

有三种“标签样式”可用，`SelectLabelStyle.LABEL_STYLE_DISAMBIGUATE_ONLY`、`SelectLabelStyle.LABEL_STYLE_TABLENAME_PLUS_COL` 和 `SelectLabelStyle.LABEL_STYLE_NONE`。默认样式是 `SelectLabelStyle.LABEL_STYLE_TABLENAME_PLUS_COL`。

在现代的 SQLAlchemy 中，通常不需要更改标签样式，因为通过使用 `ColumnElement.label()` 方法更有效地使用每个表达式的标签。在过去的版本中，`LABEL_STYLE_TABLENAME_PLUS_COL` 用于消除来自不同表、别名或子查询的同名列的歧义；新的 `LABEL_STYLE_DISAMBIGUATE_ONLY` 现在仅对与现有名称冲突的名称应用标签，以使此标签的影响最小化。

消除歧义的理由主要是为了在创建子查询时，从给定的 `FromClause.c` 集合中可用所有列表达式。

从版本 1.4 开始：- `GenerativeSelect.set_label_style()` 方法替换了以前的 `.apply_labels()`、`.with_labels()` 和 `use_labels=True` 方法和/或参数的组合。

另请参见

`LABEL_STYLE_DISAMBIGUATE_ONLY`

`LABEL_STYLE_TABLENAME_PLUS_COL`

`LABEL_STYLE_NONE`

`LABEL_STYLE_DEFAULT`

```py
method slice(start: int, stop: int) → Self
```

*从* `GenerativeSelect.slice()` *方法继承自* `GenerativeSelect`

根据片段对该语句应用 LIMIT / OFFSET。

开始和停止索引的行为类似于 Python 内置 `range()` 函数的参数。该方法提供了一种替代使用 `LIMIT`/`OFFSET` 获取查询片段的方法。

例如，

```py
stmt = select(User).order_by(User).id.slice(1, 3)
```

渲染为

```py
SELECT  users.id  AS  users_id,
  users.name  AS  users_name
FROM  users  ORDER  BY  users.id
LIMIT  ?  OFFSET  ?
(2,  1)
```

注意

`GenerativeSelect.slice()` 方法将替换任何应用了 `GenerativeSelect.fetch()` 的子句。

从版本 1.4 开始：添加了从 ORM 泛化的 `GenerativeSelect.slice()` 方法。

另请参见

`GenerativeSelect.limit()`

`GenerativeSelect.offset()`

`GenerativeSelect.fetch()`

```py
method subquery(name: str | None = None) → Subquery
```

*从* `SelectBase.subquery()` *方法继承自* `SelectBase`

返回此 `SelectBase` 的子查询。

从 SQL 角度来看，子查询是一种可以放置在另一个 SELECT 语句的 FROM 子句中的带括号的命名构造。

鉴于这样一个 SELECT 语句：

```py
stmt = select(table.c.id, table.c.name)
```

上述语句可能如下所示：

```py
SELECT table.id, table.name FROM table
```

子查询本身的形式呈现相同，但当嵌入到另一个 SELECT 语句的 FROM 子句中时，它变成了一个命名的子元素：

```py
subq = stmt.subquery()
new_stmt = select(subq)
```

上述呈现为：

```py
SELECT anon_1.id, anon_1.name
FROM (SELECT table.id, table.name FROM table) AS anon_1
```

历史上，`SelectBase.subquery()` 相当于在 FROM 对象上调用 `FromClause.alias()` 方法；但是，由于 `SelectBase` 对象不是直接的 FROM 对象，因此 `SelectBase.subquery()` 方法提供了更清晰的语义。

新功能于版本 1.4 中引入。

```py
method suffix_with(*suffixes: _TextCoercedExpressionArgument[Any], dialect: str = '*') → Self
```

*继承自* `HasSuffixes.suffix_with()` *方法的* `HasSuffixes`

在语句整体后添加一个或多个表达式。

用于支持某些结构上的特定后缀关键字的后端。

例如：

```py
stmt = select(col1, col2).cte().suffix_with(
    "cycle empno set y_cycle to 1 default 0", dialect="oracle")
```

可以通过多次调用 `HasSuffixes.suffix_with()` 来指定多个后缀。

参数：

+   `*suffixes` – 将在目标子句后面呈现的文本或 `ClauseElement` 构造。

+   `dialect` – 可选的字符串方言名称，将限制此后缀仅在该方言中呈现。

```py
method union(*other: _SelectStatementForCompoundArgument) → CompoundSelect
```

返回针对提供的位置参数的选择器构造的 SQL `UNION`。

参数：

+   `*other` –

    一个或多个用于创建 UNION 的元素。

    在版本 1.4.28 中更改：现在接受多个元素。

+   `**kwargs` – 关键字参数将转发给新创建的 `CompoundSelect` 对象的构造函数。

```py
method union_all(*other: _SelectStatementForCompoundArgument) → CompoundSelect
```

返回针对提供的位置参数的选择器构造的 SQL `UNION ALL`。

参数：

+   `*other` –

    一个或多个用于创建 UNION 的元素。

    在版本 1.4.28 中更改：现在接受多个元素。

+   `**kwargs` – 关键字参数将转发给新创建的 `CompoundSelect` 对象的构造函数。

```py
method where(*whereclause: _ColumnExpressionArgument[bool]) → Self
```

返回一个新的 `select()` 构造，并将给定表达式添加到其 WHERE 子句中，如果有的话，通过 AND 连接到现有子句。

```py
attribute whereclause
```

返回此 `Select` 语句的完整 WHERE 子句。

这将当前的 WHERE 条件集合装配成一个名为 `BooleanClauseList` 的结构。

新功能于版本 1.4 中引入。

```py
method with_for_update(*, nowait: bool = False, read: bool = False, of: _ForUpdateOfArgument | None = None, skip_locked: bool = False, key_share: bool = False) → Self
```

*继承自* `GenerativeSelect.with_for_update()` *方法的* `GenerativeSelect`

为这个 `GenerativeSelect` 指定一个 `FOR UPDATE` 子句。

例如：

```py
stmt = select(table).with_for_update(nowait=True)
```

在像 PostgreSQL 或 Oracle 这样的数据库上，上述内容会渲染为类似的语句：

```py
SELECT table.a, table.b FROM table FOR UPDATE NOWAIT
```

在其他后端上，`nowait` 选项会被忽略，而会产生：

```py
SELECT table.a, table.b FROM table FOR UPDATE
```

当不带参数调用时，语句将以后缀 `FOR UPDATE` 渲染。然后可以提供其他参数，允许常见的特定于数据库的变体。

参数：

+   `nowait` – 布尔值；在 Oracle 和 PostgreSQL 方言上会渲染 `FOR UPDATE NOWAIT`。

+   `read` – 布尔值；在 MySQL 上会渲染 `LOCK IN SHARE MODE`，在 PostgreSQL 上会渲染 `FOR SHARE`。在 PostgreSQL 上，与 `nowait` 结合时，会渲染 `FOR SHARE NOWAIT`。

+   `of` – SQL 表达式或 SQL 表达式元素列表，（通常是 `Column` 对象或兼容表达式，对于某些后端也可以是表达式）将渲染为 `FOR UPDATE OF` 子句；受 PostgreSQL、Oracle、某些 MySQL 版本以及可能其他后端支持。可能根据后端渲染为表或列。

+   `skip_locked` – 布尔值，在 Oracle 和 PostgreSQL 方言上会渲染 `FOR UPDATE SKIP LOCKED`，如果同时指定 `read=True`，则会渲染 `FOR SHARE SKIP LOCKED`。

+   `key_share` – 布尔值，会在 PostgreSQL 方言上渲染 `FOR NO KEY UPDATE`，或者如果与 `read=True` 结合，会在 PostgreSQL 方言上渲染 `FOR KEY SHARE`。

```py
method with_hint(selectable: _FromClauseArgument, text: str, dialect_name: str = '*') → Self
```

*继承自* `HasHints.with_hint()` *方法的* `HasHints`

为给定的可选择对象添加索引或其他执行上下文提示到这个 `Select` 或其他可选择对象。

提示文本会根据正在使用的数据库后端在相应位置渲染，相对于传递为 `selectable` 参数的给定 `Table` 或 `Alias`。方言实现通常使用 Python 字符串替换语法，使用令牌 `%(name)s` 渲染表或别名的名称。例如，在使用 Oracle 时，���下内容：

```py
select(mytable).\
    with_hint(mytable, "index(%(name)s ix_mytable)")
```

将渲染的 SQL 如下：

```py
select /*+ index(mytable ix_mytable) */ ... from mytable
```

`dialect_name` 选项将限制特定提示的渲染到特定后端。例如，同时为 Oracle 和 Sybase 添加提示：

```py
select(mytable).\
    with_hint(mytable, "index(%(name)s ix_mytable)", 'oracle').\
    with_hint(mytable, "WITH INDEX ix_mytable", 'mssql')
```

另请参阅

`Select.with_statement_hint()`

```py
method with_only_columns(*entities: _ColumnsClauseArgument[Any], maintain_column_froms: bool = False, **_Select__kw: Any) → Select[Any]
```

返回一个新的 `select()` 构造，其列子句替换为给定的实体。

默认情况下，此方法与原始 `select()` 被调用时给定实体完全等效。例如，一个语句：

```py
s = select(table1.c.a, table1.c.b)
s = s.with_only_columns(table1.c.b)
```

应该与以下内容完全等效：

```py
s = select(table1.c.b)
```

在这种操作模式下，如果未明确指定，`Select.with_only_columns()` 还将动态修改语句的 FROM 子句。要保持现有的 FROM 集合，包括当前列子句暗示的那些，请添加 `Select.with_only_columns.maintain_column_froms` 参数：

```py
s = select(table1.c.a, table2.c.b)
s = s.with_only_columns(table1.c.a, maintain_column_froms=True)
```

上述参数执行了将列集合中的有效 FROM 转移到 `Select.select_from()` 方法的操作，就好像调用了以下内容：

```py
s = select(table1.c.a, table2.c.b)
s = s.select_from(table1, table2).with_only_columns(table1.c.a)
```

`Select.with_only_columns.maintain_column_froms` 参数利用了 `Select.columns_clause_froms` 集合，并执行了与下面等效的操作：

```py
s = select(table1.c.a, table2.c.b)
s = s.select_from(*s.columns_clause_froms).with_only_columns(table1.c.a)
```

参数：

+   `*entities` – 要使用的列表达式。

+   `maintain_column_froms` –

    布尔参数，将确保从当前列子句暗示的 FROM 列表将首先转移到 `Select.select_from()` 方法中。

    自版本 1.4.23 新增。

```py
method with_statement_hint(text: str, dialect_name: str = '*') → Self
```

*从* `HasHints.with_statement_hint()` *方法继承* 的。

给这个 `Select` 或其他可选择的对象添加一个语句提示。

此方法与 `Select.with_hint()` 类似，但不需要单独的表，而是应用于整个语句。

此处的提示特定于后端数据库，可能包括隔离级别、文件指令、提取指令等。

另请参见

`Select.with_hint()`

`Select.prefix_with()` - 通用的 SELECT 前缀，也可以适用于一些特定于数据库的 HINT 语法，例如 MySQL 优化器提示

```py
class sqlalchemy.sql.expression.Selectable
```

将一个类标记为可选择的。

**成员**

corresponding_column(), exported_columns, inherit_cache, is_derived_from(), lateral(), replace_selectable()

**类签名**

类`sqlalchemy.sql.expression.Selectable`（`sqlalchemy.sql.expression.ReturnsRows`）

```py
method corresponding_column(column: KeyedColumnElement[Any], require_embedded: bool = False) → KeyedColumnElement[Any] | None
```

给定一个`ColumnElement`，从此`Selectable.exported_columns`集合中返回原始`ColumnElement`的导出`ColumnElement`对象，该对象通过共同的祖先列与原始`ColumnElement`对应。

参数：

+   `column` – 要匹配的目标`ColumnElement`。

+   `require_embedded` – 只返回给定`ColumnElement`对应的列，如果给定的`ColumnElement`实际上存在于此`Selectable`的子元素中。通常，如果列仅与此`Selectable`的导出列之一共享共同的祖先，则列将匹配。

另请参见

`Selectable.exported_columns` - 用于该操作的`ColumnCollection`。

`ColumnCollection.corresponding_column()` - 实现方法。

```py
attribute exported_columns
```

*继承自* `ReturnsRows` *的* `ReturnsRows.exported_columns` *属性*

表示此 `ReturnsRows` 的“导出”列的 `ColumnCollection`。

“导出”的列代表由此 SQL 构造渲染的 `ColumnElement` 表达式集合。有几种主要类型，包括 FROM 子句的“FROM 子句列”，如表、连接或子查询，“SELECT”中的列，即 SELECT 语句的“列子句”，以及 DML 语句中的 RETURNING 列。

版本 1.4 中的新功能。

另请参阅

`FromClause.exported_columns`

`SelectBase.exported_columns`

```py
attribute inherit_cache: bool | None = None
```

*继承自* `HasCacheKey` *的* `HasCacheKey.inherit_cache` *属性*

指示此`HasCacheKey`实例是否应使用其直接超类使用的缓存键生成方案。

属性默认为`None`，表示构造尚未考虑其是否适合参与缓存；这在功能上等同于将值设置为`False`，只是还会发出警告。

如果与此类本地属性而不是其超类无关的属性的 SQL 不会更改，则可以在特定类上将此标志设置为`True`。

另请参阅

启用自定义构造的缓存支持 - 设置第三方或用户定义的 SQL 构造的 `HasCacheKey.inherit_cache` 属性的通用指南。

```py
method is_derived_from(fromclause: FromClause | None) → bool
```

*继承自* `ReturnsRows` *的* `ReturnsRows.is_derived_from()` *方法*

如果此`ReturnsRows`是从给定的`FromClause`‘派生’，则返回`True`。

一个示例是 Table 的别名派生自该 Table。

```py
method lateral(name: str | None = None) → LateralFromClause
```

返回此`Selectable`的横向别名。

返回值是顶层`lateral()`函数提供的`Lateral`构造。

请参阅

横向相关 - 使用概述。

```py
method replace_selectable(old: FromClause, alias: Alias) → Self
```

用给定的`Alias`对象替换所有`FromClause`‘旧’的出现，返回此`FromClause`的副本。

自版本 1.4 起弃用：`Selectable.replace_selectable()`方法已弃用，并将在将来的版本中删除。通过 sqlalchemy.sql.visitors 模块可获得类似的功能。

```py
class sqlalchemy.sql.expression.SelectBase
```

SELECT 语句的基类。

这包括`Select`、`CompoundSelect`和`TextualSelect`。

**成员**

add_cte(), alias(), as_scalar(), c, corresponding_column(), cte(), exists(), exported_columns, get_label_style(), inherit_cache, is_derived_from(), label(), lateral(), replace_selectable(), scalar_subquery(), select(), selected_columns, set_label_style(), subquery()

**类签名**

类 `sqlalchemy.sql.expression.SelectBase` (`sqlalchemy.sql.roles.SelectStatementRole`, `sqlalchemy.sql.roles.DMLSelectRole`, `sqlalchemy.sql.roles.CompoundElementRole`, `sqlalchemy.sql.roles.InElementRole`, `sqlalchemy.sql.expression.HasCTE`, `sqlalchemy.sql.annotation.SupportsCloneAnnotations`, `sqlalchemy.sql.expression.Selectable`)

```py
method add_cte(*ctes: CTE, nest_here: bool = False) → Self
```

*继承自* `HasCTE.add_cte()` *方法的* `HasCTE`

向此语句添加一个或多个 `CTE` 构造。

此方法将将给定的 `CTE` 构造与父语句关联，以便它们将无条件地在最终语句的 WITH 子句中呈现，即使在语句或任何子选择中未引用它们。

当设置为 True 时，可选的`HasCTE.add_cte.nest_here` 参数将导致每个给定的`CTE` 将与此语句一起直接呈现在 WITH 子句中，而不是被移动到最终呈现的语句的顶部，即使此语句被呈现为在较大语句中的子查询内也是如此。

此方法有两个通用用途。一个是嵌入一些具有某种目的但不被明确引用的 CTE 语句，例如嵌入一个 DML 语句（例如 INSERT 或 UPDATE）作为 CTE 内联到主语句中，该主语句可能间接地引用其结果。另一个是控制特定一系列 CTE 构造的确切放置位置，这些构造应直接以一个特定语句的形式呈现，该语句可能嵌套在较大的语句中。

例如：

```py
from sqlalchemy import table, column, select
t = table('t', column('c1'), column('c2'))

ins = t.insert().values({"c1": "x", "c2": "y"}).cte()

stmt = select(t).add_cte(ins)
```

会呈现为：

```py
WITH anon_1 AS
(INSERT INTO t (c1, c2) VALUES (:param_1, :param_2))
SELECT t.c1, t.c2
FROM t
```

在上面的例子中，“anon_1” CTE 未在 SELECT 语句中引用，但仍然完成了运行 INSERT 语句的任务。

类似地，在与 DML 相关的上下文中，使用 PostgreSQL `Insert` 构造生成“upsert”：

```py
from sqlalchemy import table, column
from sqlalchemy.dialects.postgresql import insert

t = table("t", column("c1"), column("c2"))

delete_statement_cte = (
    t.delete().where(t.c.c1 < 1).cte("deletions")
)

insert_stmt = insert(t).values({"c1": 1, "c2": 2})
update_statement = insert_stmt.on_conflict_do_update(
    index_elements=[t.c.c1],
    set_={
        "c1": insert_stmt.excluded.c1,
        "c2": insert_stmt.excluded.c2,
    },
).add_cte(delete_statement_cte)

print(update_statement)
```

上述语句呈现为：

```py
WITH deletions AS
(DELETE FROM t WHERE t.c1 < %(c1_1)s)
INSERT INTO t (c1, c2) VALUES (%(c1)s, %(c2)s)
ON CONFLICT (c1) DO UPDATE SET c1 = excluded.c1, c2 = excluded.c2
```

新版本中新增 1.4.21。

参数：

+   `*ctes` –

    零个或多个`CTE` 构造。

    2.0 版本中的更改：接受多个 CTE 实例

+   `nest_here` –

    如果为 True，则给定的 CTE 或 CTE 将呈现为，好像它们在添加到此`HasCTE` 时指定了`HasCTE.cte.nesting` 标志为 True。假设给定的 CTE 在外部包含语句中也没有被引用，那么当给出此标志时，给定的 CTE 应该在此语句级别呈现。

    新版本中新增 2.0。

    另请参阅

    `HasCTE.cte.nesting`

```py
method alias(name: str | None = None, flat: bool = False) → Subquery
```

返回针对此 `SelectBase` 的命名子查询。

对于`SelectBase` （而不是`FromClause`），这将返回一个`Subquery` 对象，其行为基本上与与`FromClause` 一起使用的`Alias` 对象相同。

于版本 1.4 中更改：`SelectBase.alias()` 方法现在是 `SelectBase.subquery()` 方法的同义词。

```py
method as_scalar() → ScalarSelect[Any]
```

自版本 1.4 起弃用：`SelectBase.as_scalar()` 方法已弃用，并将在未来版本中移除。请参考 `SelectBase.scalar_subquery()`。

```py
attribute c
```

自版本 1.4 起弃用：`SelectBase.c` 和 `SelectBase.columns` 属性已弃用，并将在未来版本中移除；这些属性隐式创建了一个子查询，应显式指定。请先调用 `SelectBase.subquery()` 来创建一个子查询，然后该属性将包含在其中。要访问此 SELECT 对象从中选择的列，请使用 `SelectBase.selected_columns` 属性。

```py
method corresponding_column(column: KeyedColumnElement[Any], require_embedded: bool = False) → KeyedColumnElement[Any] | None
```

*继承自* `Selectable.corresponding_column()` *方法的* `Selectable`

给定一个 `ColumnElement`，返回此 `Selectable` 的 `Selectable.exported_columns` 集合中导出的 `ColumnElement` 对象，该对象与原始的 `ColumnElement` 通过共同的祖先列相对应。

参数：

+   `column` – 目标 `ColumnElement`，要匹配。

+   `require_embedded` – 仅返回给定`ColumnElement`的相应列，如果给定的`ColumnElement`实际上存在于此`Selectable`的子元素中。通常，如果列仅与此`Selectable`的导出列之一共享共同的祖先，则该列将匹配。

另请参阅

`Selectable.exported_columns` - 用于操作的`ColumnCollection`。

`ColumnCollection.corresponding_column()` - 实现方法。

```py
method cte(name: str | None = None, recursive: bool = False, nesting: bool = False) → CTE
```

*继承自* `HasCTE.cte()` *方法的* `HasCTE`

返回一个新的`CTE`，或者 Common Table Expression 实例。

公共表达式是 SQL 标准，其中 SELECT 语句可以使用名为“WITH”的子句指定的辅助语句来绘制主要语句，特殊的关于 UNION 的语义也可以用来允许“递归”查询，其中 SELECT 语句可以绘制之前已经选择的行集。

CTEs 也可以应用于一些数据库上的 DML 构造 UPDATE、INSERT 和 DELETE，既可以作为 CTE 行的来源与 RETURNING 组合使用，也可以作为 CTE 行的消费者。

SQLAlchemy 检测到`CTE`对象，它们被视为与`Alias`对象类似的特殊元素，将它们作为语句的 FROM 子句以及语句顶部的 WITH 子句传递。

对于特殊的前缀，如 PostgreSQL 的“MATERIALIZED”和“NOT MATERIALIZED”，可以使用`CTE.prefix_with()`方法来建立这些前缀。

从 1.3.13 版本开始更改：添加了对前缀的支持。特别是 - MATERIALIZED 和 NOT MATERIALIZED。

参数：

+   `name` – 给通用表达式的名称。像`FromClause.alias()`一样，如果将名称保留为`None`，则在查询编译时将使用匿名符号。

+   `recursive` – 如果为 `True`，将渲染 `WITH RECURSIVE`。递归公共表达式打算与 UNION ALL 结合使用，以从已选择的行中派生行。

+   `nesting` –

    如果为 `True`，将在引用它的语句中本地渲染 CTE。对于更复杂的场景，还可以使用 `HasCTE.add_cte()` 方法，使用 `HasCTE.add_cte.nest_here` 参数更精确地控制特定 CTE 的精确放置位置。

    在 1.4.24 版本中新增。

    另请参见

    `HasCTE.add_cte()`

以下示例包括两个来自 PostgreSQL 文档的示例，网址为 [`www.postgresql.org/docs/current/static/queries-with.html`](https://www.postgresql.org/docs/current/static/queries-with.html)，以及其他示例。

示例 1，非递归：

```py
from sqlalchemy import (Table, Column, String, Integer,
                        MetaData, select, func)

metadata = MetaData()

orders = Table('orders', metadata,
    Column('region', String),
    Column('amount', Integer),
    Column('product', String),
    Column('quantity', Integer)
)

regional_sales = select(
                    orders.c.region,
                    func.sum(orders.c.amount).label('total_sales')
                ).group_by(orders.c.region).cte("regional_sales")

top_regions = select(regional_sales.c.region).\
        where(
            regional_sales.c.total_sales >
            select(
                func.sum(regional_sales.c.total_sales) / 10
            )
        ).cte("top_regions")

statement = select(
            orders.c.region,
            orders.c.product,
            func.sum(orders.c.quantity).label("product_units"),
            func.sum(orders.c.amount).label("product_sales")
    ).where(orders.c.region.in_(
        select(top_regions.c.region)
    )).group_by(orders.c.region, orders.c.product)

result = conn.execute(statement).fetchall()
```

示例 2，使用 WITH RECURSIVE：

```py
from sqlalchemy import (Table, Column, String, Integer,
                        MetaData, select, func)

metadata = MetaData()

parts = Table('parts', metadata,
    Column('part', String),
    Column('sub_part', String),
    Column('quantity', Integer),
)

included_parts = select(\
    parts.c.sub_part, parts.c.part, parts.c.quantity\
    ).\
    where(parts.c.part=='our part').\
    cte(recursive=True)

incl_alias = included_parts.alias()
parts_alias = parts.alias()
included_parts = included_parts.union_all(
    select(
        parts_alias.c.sub_part,
        parts_alias.c.part,
        parts_alias.c.quantity
    ).\
    where(parts_alias.c.part==incl_alias.c.sub_part)
)

statement = select(
            included_parts.c.sub_part,
            func.sum(included_parts.c.quantity).
              label('total_quantity')
        ).\
        group_by(included_parts.c.sub_part)

result = conn.execute(statement).fetchall()
```

示例 3，使用 CTE 进行更新和插入：

```py
from datetime import date
from sqlalchemy import (MetaData, Table, Column, Integer,
                        Date, select, literal, and_, exists)

metadata = MetaData()

visitors = Table('visitors', metadata,
    Column('product_id', Integer, primary_key=True),
    Column('date', Date, primary_key=True),
    Column('count', Integer),
)

# add 5 visitors for the product_id == 1
product_id = 1
day = date.today()
count = 5

update_cte = (
    visitors.update()
    .where(and_(visitors.c.product_id == product_id,
                visitors.c.date == day))
    .values(count=visitors.c.count + count)
    .returning(literal(1))
    .cte('update_cte')
)

upsert = visitors.insert().from_select(
    [visitors.c.product_id, visitors.c.date, visitors.c.count],
    select(literal(product_id), literal(day), literal(count))
        .where(~exists(update_cte.select()))
)

connection.execute(upsert)
```

示例 4，CTE 嵌套（SQLAlchemy 1.4.24 及以上版本）：

```py
value_a = select(
    literal("root").label("n")
).cte("value_a")

# A nested CTE with the same name as the root one
value_a_nested = select(
    literal("nesting").label("n")
).cte("value_a", nesting=True)

# Nesting CTEs takes ascendency locally
# over the CTEs at a higher level
value_b = select(value_a_nested.c.n).cte("value_b")

value_ab = select(value_a.c.n.label("a"), value_b.c.n.label("b"))
```

上述查询将第二个 CTE 嵌套在第一个内部，如下所示：

```py
WITH
    value_a AS
        (SELECT 'root' AS n),
    value_b AS
        (WITH value_a AS
            (SELECT 'nesting' AS n)
        SELECT value_a.n AS n FROM value_a)
SELECT value_a.n AS a, value_b.n AS b
FROM value_a, value_b
```

使用 `HasCTE.add_cte()` 方法可以设置相同的 CTE（通用表达式）（SQLAlchemy 2.0 及以上版本）：

```py
value_a = select(
    literal("root").label("n")
).cte("value_a")

# A nested CTE with the same name as the root one
value_a_nested = select(
    literal("nesting").label("n")
).cte("value_a")

# Nesting CTEs takes ascendency locally
# over the CTEs at a higher level
value_b = (
    select(value_a_nested.c.n).
    add_cte(value_a_nested, nest_here=True).
    cte("value_b")
)

value_ab = select(value_a.c.n.label("a"), value_b.c.n.label("b"))
```

示例 5，非线性 CTE（SQLAlchemy 1.4.28 及以上版本）：

```py
edge = Table(
    "edge",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("left", Integer),
    Column("right", Integer),
)

root_node = select(literal(1).label("node")).cte(
    "nodes", recursive=True
)

left_edge = select(edge.c.left).join(
    root_node, edge.c.right == root_node.c.node
)
right_edge = select(edge.c.right).join(
    root_node, edge.c.left == root_node.c.node
)

subgraph_cte = root_node.union(left_edge, right_edge)

subgraph = select(subgraph_cte)
```

上述查询将在递归 CTE 内部呈现 2 个 UNION：

```py
WITH RECURSIVE nodes(node) AS (
        SELECT 1 AS node
    UNION
        SELECT edge."left" AS "left"
        FROM edge JOIN nodes ON edge."right" = nodes.node
    UNION
        SELECT edge."right" AS "right"
        FROM edge JOIN nodes ON edge."left" = nodes.node
)
SELECT nodes.node FROM nodes
```

另请参见

`Query.cte()` - `HasCTE.cte()` 的 ORM 版本。

```py
method exists() → Exists
```

返回此可选的 `Exists` 表示形式，可用作列表达式。

返回的对象是 `Exists` 的实例。

另请参见

`exists()`

EXISTS 子查询 - 在 2.0 风格 的教程中。

在 1.4 版本中新增。

```py
attribute exported_columns
```

表示此 `Selectable` 的“导出”列的 `ColumnCollection`，不包括 `TextClause` 结构。

`SelectBase`对象的“导出”列与`SelectBase.selected_columns`集合是同义词。

自版本 1.4 开始。

请参见

`Select.exported_columns`

`Selectable.exported_columns`

`FromClause.exported_columns`

```py
method get_label_style() → SelectLabelStyle
```

检索当前标签样式。

由子类实现。

```py
attribute inherit_cache: bool | None = None
```

*从* `HasCacheKey` *的`HasCacheKey.inherit_cache`属性继承*

指示此`HasCacheKey`实例是否应该使用其直接超类使用的缓存键生成方案。

该属性默认为`None`，表示构造尚未考虑其是否适合参与缓存；这在功能上等同于将值设置为`False`，只是还会发出警告。

如果与此类本地属性有关，而不是其超类，对象对应的 SQL 不会改变，则可以将此标志设置为`True`。

请参见

为自定义构造启用缓存支持 - 设置第三方或用户定义的 SQL 构造的`HasCacheKey.inherit_cache`属性的通用指南。

```py
method is_derived_from(fromclause: FromClause | None) → bool
```

*从* `ReturnsRows.is_derived_from()` *方法继承* `ReturnsRows`

如果此`ReturnsRows`是从给定的`FromClause`‘派生’，则返回`True`。

一个示例是 Table 的 Alias 从该 Table 派生而来。

```py
method label(name: str | None) → Label[Any]
```

返回此可选项的“标量”表示，嵌入为带有标签的子查询。

请参见

`SelectBase.scalar_subquery()`。

```py
method lateral(name: str | None = None) → LateralFromClause
```

返回此`Selectable`的 LATERAL 别名。

返回值也是由顶级`lateral()`函数提供的`Lateral`构造。

另请参见

LATERAL 相关 - 使用概述。

```py
method replace_selectable(old: FromClause, alias: Alias) → Self
```

*继承自* `Selectable` *的* `Selectable.replace_selectable()` *方法*

使用给定的`Alias`对象替换所有`FromClause` ‘old’的所有出现，返回此`FromClause`的副本。

自版本 1.4 起已弃用：`Selectable.replace_selectable()`方法已弃用，并将在将来的版本中删除。类似功能可通过 sqlalchemy.sql.visitors 模块获得。

```py
method scalar_subquery() → ScalarSelect[Any]
```

返回此可选择项的‘标量’表示，可用作列表达式。

返回的对象是`ScalarSelect`的实例。

通常，仅在其列子句中有一个列的选择语句有资格用作标量表达式。然后可以在包含 SELECT 的 WHERE 子句或列子句中使用标量子查询。

注意标量子查询与可以使用`SelectBase.subquery()`方法生成的 FROM 级子查询不同。

另请参见

标量和相关子查询 - 在 2.0 教程中

```py
method select(*arg: Any, **kw: Any) → Select
```

自版本 1.4 起已弃用：`SelectBase.select()`方法已弃用，并将在将来的版本中删除；此方法隐式创建一个应该显式的子查询。请先调用`SelectBase.subquery()`以创建一个子查询，然后可以选择该子查询。

```py
attribute selected_columns
```

一个表示此 SELECT 语句或类似结构在其结果集中返回的列的`ColumnCollection`。

此集合与 `FromClause.columns` 集合不同，因为此集合中的列不能直接嵌套在另一个 SELECT 语句中；必须先应用子查询，以提供 SQL 所需的必要括号化。

注意

`SelectBase.selected_columns` 集合不包括在列子句中使用 `text()` 构造建立的表达式；这些表达式在集合中被忽略。要在 `Select` 构造内部使用纯文本列表达式，请使用 `literal_column()` 构造。

另请参见

`Select.selected_columns`

新版本 1.4 中的新增内容。

```py
method set_label_style(style: SelectLabelStyle) → Self
```

返回具有指定标签样式的新可选项。

由子类实现。

```py
method subquery(name: str | None = None) → Subquery
```

返回此 `SelectBase` 的子查询。

从 SQL 角度来看，子查询是一个带括号的命名结构，可以放置在另一个 SELECT 语句的 FROM 子句中。

给定一个 SELECT 语句，例如：

```py
stmt = select(table.c.id, table.c.name)
```

上述语句可能如下所示：

```py
SELECT table.id, table.name FROM table
```

单独的子查询形式渲染方式相同，但是当嵌入到另一个 SELECT 语句的 FROM 子句中时，它变成了一个命名的子元素：

```py
subq = stmt.subquery()
new_stmt = select(subq)
```

以上渲染为：

```py
SELECT anon_1.id, anon_1.name
FROM (SELECT table.id, table.name FROM table) AS anon_1
```

从历史角度看，`SelectBase.subquery()` 等同于在 FROM 对象上调用 `FromClause.alias()` 方法；然而，由于 `SelectBase` 对象并非直接的 FROM 对象，因此 `SelectBase.subquery()` 方法提供了更清晰的语义。

新版本 1.4 中的新增内容。

```py
class sqlalchemy.sql.expression.Subquery
```

表示一个 SELECT 的子查询。

`Subquery` 是通过调用任何 `SelectBase` 子类的 `SelectBase.subquery()` 方法或便利的 `SelectBase.alias()` 方法创建的，这些子类包括 `Select`、`CompoundSelect` 和 `TextualSelect`。在 FROM 子句中渲染时，它表示 SELECT 语句的主体在括号内，后跟通常定义所有“别名”对象的常规 “AS <somename>”。

`Subquery` 对象与 `Alias` 对象非常相似，并且可以以等效的方式使用。`Alias` 和 `Subquery` 之间的区别在于，`Alias` 总是包含一个 `FromClause` 对象，而 `Subquery` 总是包含一个 `SelectBase` 对象。

新版本 1.4 中增加了 `Subquery` 类，其目的是提供 SELECT 语句的别名版本。

**成员**

as_scalar(), inherit_cache

**类签名**

类 `sqlalchemy.sql.expression.Subquery` (`sqlalchemy.sql.expression.AliasedReturnsRows`)

```py
method as_scalar() → ScalarSelect[Any]
```

自版本 1.4 弃用：`Subquery.as_scalar()` 方法（在版本 1.4 之前为 `Alias.as_scalar()`）已弃用，并将在将来的版本中删除；请在构造子查询对象之前或使用 ORM 时使用 `Select.scalar_subquery()` 方法的 `select()` 构造，或者使用 `Query.scalar_subquery()` 方法。

```py
attribute inherit_cache: bool | None = True
```

指示此 `HasCacheKey` 实例是否应使用其直接超类使用的缓存键生成方案。

此属性默认为`None`，表示构造尚未考虑其是否适合参与缓存；这在功能上等同于将值设置为`False`，只是还会发出警告。

如果与对象对应的 SQL 不基于这个类本地的属性而改变，并且不是它的超类，则可以在特定类上将此标志设置为`True`。

参见

为自定义构造启用缓存支持 - 设置第三方或用户定义的 SQL 构造的 `HasCacheKey.inherit_cache` 属性的一般指南。

```py
class sqlalchemy.sql.expression.TableClause
```

表示一个最小的“表”结构。

这是一个轻量级的表对象，只有一个名称、一个由 `column()` 函数生成的列集合，以及一个模式：

```py
from sqlalchemy import table, column

user = table("user",
        column("id"),
        column("name"),
        column("description"),
)
```

`TableClause` 构造用作更常用的 `Table` 对象的基础，提供通常的 `FromClause` 服务，包括 `.c.` 集合和语句生成方法。

它**不**提供 `Table` 的所有附加架构级服务，包括约束、对其他表的引用或对 `MetaData` 级别服务的支持。当手头没有更完整的 `Table` 时，它本身作为一种临时构造是有用的，用于生成快速的 SQL 语句。

**成员**

alias(), c, columns, compare(), compile(), corresponding_column(), delete(), description, entity_namespace, exported_columns, foreign_keys, get_children(), implicit_returning, inherit_cache, insert(), is_derived_from(), join(), lateral(), outerjoin(), params(), primary_key, replace_selectable(), schema, select(), self_group(), table_valued(), tablesample(), unique_params(), update()

**类签名**

类`sqlalchemy.sql.expression.TableClause` (`sqlalchemy.sql.roles.DMLTableRole`, `sqlalchemy.sql.expression.Immutable`, `sqlalchemy.sql.expression.NamedFromClause`)

```py
method alias(name: str | None = None, flat: bool = False) → NamedFromClause
```

*继承自* `FromClause.alias()` *方法的* `FromClause`

返回此`FromClause`的别名。

例如：

```py
a2 = some_table.alias('a2')
```

上述代码创建了一个`Alias`对象，可用作任何 SELECT 语句中的 FROM 子句。

另请参阅

使用别名

`alias()`

```py
attribute c
```

*继承自* `FromClause.c` *属性的* `FromClause`

`FromClause.columns`的同义词。

返回值：

一个`ColumnCollection`

```py
attribute columns
```

*继承自* `FromClause.columns` *属性的* `FromClause`

由此`FromClause`维护的`ColumnElement`对象的基于名称的集合。

`columns` 或 `c` 集合是使用绑定到表或其他可选择列的列构建 SQL 表达式的入口：

```py
select(mytable).where(mytable.c.somecolumn == 5)
```

返回值：

一个`ColumnCollection`对象。

```py
method compare(other: ClauseElement, **kw: Any) → bool
```

*继承自* `ClauseElement.compare()` *方法的* `ClauseElement`

将此 `ClauseElement` 与给定的 `ClauseElement` 进行比较。

子类应该覆盖默认行为，即直接的标识比较。

**kw 是子类 `compare()` 方法消耗的参数，可以用于修改比较的标准（参见 `ColumnElement`）。

```py
method compile(bind: _HasDialect | None = None, dialect: Dialect | None = None, **kw: Any) → Compiled
```

*继承自* `CompilerElement.compile()` *方法的* `CompilerElement`

编译此 SQL 表达式。

返回值是一个`Compiled`对象。调用返回值上的`str()`或`unicode()`将产生结果的字符串表示。`Compiled` 对象也可以使用 `params` 访问器返回绑定参数名称和值的字典。

参数：

+   `bind` – 一个`Connection`或`Engine`，它可以提供一个`Dialect`以生成一个`Compiled`对象。如果`bind`和`dialect`参数都被省略，将使用默认的 SQL 编译器。

+   `column_keys` – 用于 INSERT 和 UPDATE 语句，一个列名列表，应该在编译语句的 VALUES 子句中存在。如果为`None`，则从目标表对象中渲染所有列。

+   `dialect` – 一个可以生成`Compiled`对象的`Dialect`实例。该参数优先于`bind`参数。

+   `compile_kwargs` –

    可选的额外参数字典，将通过所有“visit”方法传递给编译器。这允许传递任何自定义标志到自定义编译构造中，例如。它也用于通过传递`literal_binds`标志的情况：

    ```py
    from sqlalchemy.sql import table, column, select

    t = table('t', column('x'))

    s = select(t).where(t.c.x == 5)

    print(s.compile(compile_kwargs={"literal_binds": True}))
    ```

另请参阅

如何将 SQL 表达式渲染为字符串，可能包含内联的绑定参数？

```py
method corresponding_column(column: KeyedColumnElement[Any], require_embedded: bool = False) → KeyedColumnElement[Any] | None
```

*继承自* `Selectable.corresponding_column()` *方法* 的 `Selectable`

给定一个`ColumnElement`，从这个`Selectable.exported_columns`集合中返回与原`ColumnElement`通过一个共同祖先列对应的导出`ColumnElement`对象。

参数：

+   `column` – 目标要匹配的`ColumnElement`。

+   `require_embedded` - 仅在给定 `ColumnElement` 实际存在于此 `Selectable` 的子元素中时返回相应列。通常，如果列仅与此 `Selectable` 的导出列之一共享公共祖先，则该列将匹配。

另请参阅

`Selectable.exported_columns` - 用于操作的 `ColumnCollection`。

`ColumnCollection.corresponding_column()` - 实现方法。

```py
method delete() → Delete
```

生成一个针对此 `TableClause` 的 `delete()` 构造。

例如：

```py
table.delete().where(table.c.id==7)
```

查看 `delete()` 获取参数和用法信息。

```py
attribute description
```

```py
attribute entity_namespace
```

*继承自* `FromClause.entity_namespace` *属性的* `FromClause`

返回用于 SQL 表达式中基于名称访问的命名空间。

这是用于解析“filter_by()”类型表达式的命名空间，例如：

```py
stmt.filter_by(address='some address')
```

它默认使用 `.c` 集合，但在内部可以使用“entity_namespace”注解进行覆盖，以提供替代结果。

```py
attribute exported_columns
```

*继承自* `FromClause.exported_columns` *属性的* `FromClause`

表示此 `Selectable` 的“导出”列的 `ColumnCollection`。

`FromClause` 对象的“导出”列与 `FromClause.columns` 集合是同义词。

版本 1.4 中的新功能。

另请参阅

`Selectable.exported_columns`

`SelectBase.exported_columns`

```py
attribute foreign_keys
```

*继承自* `FromClause.foreign_keys` *属性的* `FromClause`

返回此 FromClause 引用的 `ForeignKey` 标记对象的集合。

每个 `ForeignKey` 都是 `Table`-wide 的 `ForeignKeyConstraint` 的成员。

另请参阅

`Table.foreign_key_constraints`

```py
method get_children(*, omit_attrs: Tuple[str, ...] = (), **kw: Any) → Iterable[HasTraverseInternals]
```

*继承自* `HasTraverseInternals.get_children()` *方法的* `HasTraverseInternals`

返回此 `HasTraverseInternals` 的直接子元素。

用于访问遍历。

**kw 可能包含更改返回的集合的标志，例如返回子集以减少较大的遍历，或者从不同上下文返回子项目（例如模式级别的集合而不是子句级别的集合）。

```py
attribute implicit_returning = False
```

`TableClause` 不支持具有主键或列级默认值，因此隐式返回不适用。

```py
attribute inherit_cache: bool | None = None
```

*继承自* `HasCacheKey` *的* `HasCacheKey.inherit_cache` *属性*

指示此 `HasCacheKey` 实例是否应使用其直接超类使用的缓存键生成方案。

该属性的默认值为 `None`，表示构造尚未考虑该构造是否适合参与缓存；这在功能上等同于将值设置为 `False`，只是还会发出警告。

如果对象对应的 SQL 不基于此类的本地属性而改变，并且不是其超类的属性，则可以将此标志设置为 `True`。

另请参阅

为自定义结构启用缓存支持 - 为第三方或用户定义的 SQL 结构设置 `HasCacheKey.inherit_cache` 属性的通用指南。

```py
method insert() → Insert
```

生成针对此 `TableClause` 的 `Insert` 构造。

例如：

```py
table.insert().values(name='foo')
```

有关参数和用法信息，请参阅 `insert()`。

```py
method is_derived_from(fromclause: FromClause | None) → bool
```

*从* `FromClause.is_derived_from()` *方法继承* 的 `FromClause`。

如果此 `FromClause` 是从给定的 `FromClause`‘派生’，则返回 `True`。

例如，来自该表的表别名。

```py
method join(right: _FromClauseArgument, onclause: _ColumnExpressionArgument[bool] | None = None, isouter: bool = False, full: bool = False) → Join
```

*从* `FromClause.join()` *方法继承* 的示例 `FromClause`。

从此 `FromClause` 返回到另一个 `FromClause` 的 `Join`。

例如：

```py
from sqlalchemy import join

j = user_table.join(address_table,
                user_table.c.id == address_table.c.user_id)
stmt = select(user_table).select_from(j)
```

将会发出类似以下的 SQL：

```py
SELECT user.id, user.name FROM user
JOIN address ON user.id = address.user_id
```

参数：

+   `right` – 连接的右侧；这是任何 `FromClause` 对象，如一个 `Table` 对象，并且也可以是可选的兼容对象，如 ORM 映射类。

+   `onclause` – 表示连接的 ON 子句的 SQL 表达式。如果保持为 `None`，`FromClause.join()` 将尝试根据外键关系连接这两个表。

+   `isouter` – 如果为 True，则渲染 LEFT OUTER JOIN，而不是 JOIN。

+   `full` – 如果为 True，则渲染 FULL OUTER JOIN，而不是 LEFT OUTER JOIN。暗示 `FromClause.join.isouter`.

另请参阅

`join()` - 独立函数

`Join` - 生成的对象类型

```py
method lateral(name: str | None = None) → LateralFromClause
```

*继承自* `Selectable` *的* `Selectable.lateral()` *方法*

返回此`Selectable`的一个 LATERAL 别名。

返回值是`Lateral`构造，也由顶级`lateral()`函数提供。

参见

LATERAL correlation - 用法概述。

```py
method outerjoin(right: _FromClauseArgument, onclause: _ColumnExpressionArgument[bool] | None = None, full: bool = False) → Join
```

*继承自* `FromClause` *的* `FromClause.outerjoin()` *方法*

从此`FromClause`到另一个`FromClause`返回一个带有“isouter”标志设置为 True 的`Join`。

例如：

```py
from sqlalchemy import outerjoin

j = user_table.outerjoin(address_table,
                user_table.c.id == address_table.c.user_id)
```

以上等价于：

```py
j = user_table.join(
    address_table,
    user_table.c.id == address_table.c.user_id,
    isouter=True)
```

参数：

+   `right` – 连接的右侧；这是任何`FromClause`对象，如`Table`对象，并且也可以是一个可选择兼容对象，如 ORM 映射类。

+   `onclause` – 表示连接的 ON 子句的 SQL 表达式。如果保持为`None`，`FromClause.join()` 将尝试根据外键关系连接两个表。

+   `full` – 如果为 True，则渲染一个 FULL OUTER JOIN，而不是 LEFT OUTER JOIN。

参见

`FromClause.join()`

`Join`

```py
method params(*optionaldict, **kwargs)
```

*继承自* `Immutable` *的* `Immutable.params()` *方法*

返回一个副本，其中`bindparam()`元素已替换。

返回此 ClauseElement 的副本，其中`bindparam()`元素已替换为从给定字典中获取的值：

```py
>>> clause = column('x') + bindparam('foo')
>>> print(clause.compile().params)
{'foo':None}
>>> print(clause.params({'foo':7}).compile().params)
{'foo':7}
```

```py
attribute primary_key
```

*继承自* `FromClause` *的* `FromClause.primary_key` *属性*

返回此 `_selectable.FromClause` 的主键组成部分的可迭代列`Column` 对象的集合。

对于`Table`对象，此集合由 `PrimaryKeyConstraint` 表示，它本身是一个可迭代的 `Column` 对象的集合。

```py
method replace_selectable(old: FromClause, alias: Alias) → Self
```

*继承自* `Selectable.replace_selectable()` *方法的* `Selectable`

用给定的`Alias`对象替换所有`FromClause` ‘old’的出现，返回此`FromClause`的副本。

自 1.4 版本起弃用：`Selectable.replace_selectable()` 方法已弃用，并将在未来的版本中移除。类似功能可通过 sqlalchemy.sql.visitors 模块获得。

```py
attribute schema: str | None = None
```

*继承自* `FromClause.schema` *属性的* `FromClause`

为此`FromClause`定义‘schema’属性。

对于大多数对象而言，这通常是`None`，但对于`Table`对象，则被视为`Table.schema` 参数的值。

```py
method select() → Select
```

*继承自* `FromClause.select()` *方法的* `FromClause`

返回此`FromClause`的 SELECT。

例如：

```py
stmt = some_table.select().where(some_table.c.id == 5)
```

另请参阅

`select()` - 允许任意列列表的通用方法。

```py
method self_group(against: OperatorType | None = None) → ClauseElement
```

*继承自* `ClauseElement.self_group()` *方法的* `ClauseElement`

对此 `ClauseElement` 应用一个“分组”。

此方法被子类重写以返回一个“分组”构造，即括号。特别是它被“二元”表达式使用，当它们被放置到更大的表达式中时，提供一个围绕自身的分组，以及当 `select()` 构造被放置到另一个 `select()` 的 FROM 子句中时。 （请注意，子查询通常应使用 `Select.alias()` 方法创建，因为许多平台要求嵌套的 SELECT 语句必须命名）。

当表达式组合在一起时，`self_group()` 的应用是自动的 - 最终用户代码不应直接使用此方法。请注意，SQLAlchemy 的子句构造考虑了运算符优先级 - 因此在表达式中可能不需要括号，例如 `x OR (y AND z)` - AND 优先于 OR。

`ClauseElement` 的基本 `self_group()` 方法只返回自身。

```py
method table_valued() → TableValuedColumn[Any]
```

*继承自* `NamedFromClause.table_valued()` *方法的* `NamedFromClause`

为此 `FromClause` 返回一个 `TableValuedColumn` 对象。

`TableValuedColumn` 是一个代表表中完整行的 `ColumnElement`。对此构造的支持取决于后端，各种形式的支持由后端如 PostgreSQL、Oracle 和 SQL Server 提供。

例如：

```py
>>> from sqlalchemy import select, column, func, table
>>> a = table("a", column("id"), column("x"), column("y"))
>>> stmt = select(func.row_to_json(a.table_valued()))
>>> print(stmt)
SELECT  row_to_json(a)  AS  row_to_json_1
FROM  a 
```

在版本 1.4.0b2 中新增。

另请参阅

使用 SQL 函数 - 在 SQLAlchemy 统一教程中

```py
method tablesample(sampling: float | Function[Any], name: str | None = None, seed: roles.ExpressionElementRole[Any] | None = None) → TableSample
```

*继承自* `FromClause.tablesample()` *方法的* `FromClause`

返回此 `FromClause` 的 TABLESAMPLE 别名。

返回值也是由顶层 `tablesample()` 函数提供的 `TableSample` 构造。

另请参阅

`tablesample()` - 用法指南和参数

```py
method unique_params(*optionaldict, **kwargs)
```

*继承自* `Immutable` *的* `Immutable.unique_params()` *方法*

返回一个副本，其中 `bindparam()` 元素被替换。

与 `ClauseElement.params()` 具有相同功能，但将 unique=True 添加到受影响的绑定参数中，以便可以使用多个语句。

```py
method update() → Update
```

生成针对此 `TableClause` 的 `update()` 构造。

例如：

```py
table.update().where(table.c.id==7).values(name='foo')
```

参见 `update()` 获取参数和使用信息。

```py
class sqlalchemy.sql.expression.TableSample
```

表示一个 TABLESAMPLE 子句。

此对象是从 `tablesample()` 模块级函数以及所有 `FromClause` 子类上可用的 `FromClause.tablesample()` 方法构造的。

另请参阅

`tablesample()`

**类签名**

类 `sqlalchemy.sql.expression.TableSample` (`sqlalchemy.sql.expression.FromClauseAlias`)

```py
class sqlalchemy.sql.expression.TableValuedAlias
```

针对“表值”SQL 函数的别名。

此结构提供了一个 SQL 函数，该函数返回用于 SELECT 语句的 FROM 子句中使用的列。该对象使用 `FunctionElement.table_valued()` 方法生成，例如：

```py
>>> from sqlalchemy import select, func
>>> fn = func.json_array_elements_text('["one", "two", "three"]').table_valued("value")
>>> print(select(fn.c.value))
SELECT  anon_1.value
FROM  json_array_elements_text(:json_array_elements_text_1)  AS  anon_1 
```

新版本 1.4.0b2 中新增。

另请参阅

表值函数 - 在 SQLAlchemy 统一教程 中

**成员**

alias(), column, lateral(), render_derived()

**类签名**

类 `sqlalchemy.sql.expression.TableValuedAlias` (`sqlalchemy.sql.expression.LateralFromClause`, `sqlalchemy.sql.expression.Alias`)

```py
method alias(name: str | None = None, flat: bool = False) → TableValuedAlias
```

返回这个`TableValuedAlias`的新别���。

这将创建一个独立的 FROM 对象，在 SQL 语句中使用时将与原始对象区分开。

```py
attribute column
```

返回表示这个`TableValuedAlias`的列表达式。

此访问器用于实现`FunctionElement.column_valued()`方法。详细信息请参阅该方法。

例如：

```py
>>> print(select(func.some_func().table_valued("value").column))
SELECT  anon_1  FROM  some_func()  AS  anon_1 
```

另请参阅

`FunctionElement.column_valued()`

```py
method lateral(name: str | None = None) → LateralFromClause
```

返回一个带有 lateral 标志设置的新的`TableValuedAlias`，以便它呈现为 LATERAL。

另请参阅

`lateral()`

```py
method render_derived(name: str | None = None, with_types: bool = False) → TableValuedAlias
```

对这个`TableValuedAlias`应用“渲染派生”。

这会导致在别名名称后列出各个列名的“AS”序列，例如：

```py
>>> print(
...     select(
...         func.unnest(array(["one", "two", "three"])).
 table_valued("x", with_ordinality="o").render_derived()
...     )
... )
SELECT  anon_1.x,  anon_1.o
FROM  unnest(ARRAY[%(param_1)s,  %(param_2)s,  %(param_3)s])  WITH  ORDINALITY  AS  anon_1(x,  o) 
```

`with_types`关键字将在别名表达式中内联呈现列类型（此语法目前适用于 PostgreSQL 数据库）：

```py
>>> print(
...     select(
...         func.json_to_recordset(
...             '[{"a":1,"b":"foo"},{"a":"2","c":"bar"}]'
...         )
...         .table_valued(column("a", Integer), column("b", String))
...         .render_derived(with_types=True)
...     )
... )
SELECT  anon_1.a,  anon_1.b  FROM  json_to_recordset(:json_to_recordset_1)
AS  anon_1(a  INTEGER,  b  VARCHAR) 
```

参数：

+   `name` – 可选的字符串名称，将应用于生成的别名。如果保留为 None，则将使用唯一的匿名化名称。

+   `with_types` – 如果为 True，则派生列将包括每个列的数据类型规范。这是一种特殊的语法，目前已知对于某些 SQL 函数在 PostgreSQL 中是必需的。

```py
class sqlalchemy.sql.expression.TextualSelect
```

将`TextClause`构造包装在`SelectBase`接口中。

这允许`TextClause`对象获得`.c`集合和其他类似 FROM 的功能，例如`FromClause.alias()`，`SelectBase.cte()`等。

`TextualSelect`构造是通过`TextClause.columns()`方法生成的 - 详细信息请参阅该方法。

在 1.4 版本中更改：`TextualSelect` 类从 `TextAsFrom` 重命名为更正确地适应其作为 SELECT 对象而不是 FROM 子句的角色。

另请参阅

`text()`

`TextClause.columns()` - 主要的创建接口。

**成员**

add_cte(), alias(), as_scalar(), c, compare(), compile(), corresponding_column(), cte(), execution_options(), exists(), exported_columns, get_children(), get_execution_options(), get_label_style(), inherit_cache, is_derived_from(), label(), lateral(), options(), params(), replace_selectable(), scalar_subquery(), select(), selected_columns, self_group(), set_label_style(), subquery(), unique_params()

**类签名**

类 `sqlalchemy.sql.expression.TextualSelect`（`sqlalchemy.sql.expression.SelectBase`，`sqlalchemy.sql.expression.ExecutableReturnsRows`，`sqlalchemy.sql.expression.Generative`）

```py
method add_cte(*ctes: CTE, nest_here: bool = False) → Self
```

*继承自* `HasCTE.add_cte()` *方法的* `HasCTE`

向此语句添加一个或多个`CTE`结构。

这种方法将给定的`CTE`结构与父语句关联起来，使它们每个都无条件地在最终语句的 WITH 子句中呈现，即使在语句或任何子选择中没有其他地方引用它们。

当可选的`HasCTE.add_cte.nest_here`参数设置为 True 时，每个给定的`CTE`将呈现为与此语句直接一起呈现的 WITH 子句，而不是移动到最终呈现的语句的顶部，即使此语句作为更大语句中的子查询呈现。

此方法有两种通用用途。一个是嵌入一些用途的 CTE 语句，而不需要显式引用，例如，将 DML 语句（例如 INSERT 或 UPDATE）作为 CTE 嵌入到主要语句中，该主要语句可以间接地从其结果中提取。另一个是提供对应于可能嵌套在较大语句中的特定语句的确切放置的控制，这些 CTE 结构应该保持直接以某个语句的形式呈现。

例如：

```py
from sqlalchemy import table, column, select
t = table('t', column('c1'), column('c2'))

ins = t.insert().values({"c1": "x", "c2": "y"}).cte()

stmt = select(t).add_cte(ins)
```

将呈现为：

```py
WITH anon_1 AS
(INSERT INTO t (c1, c2) VALUES (:param_1, :param_2))
SELECT t.c1, t.c2
FROM t
```

在上面的例子中，“anon_1”CTE 未在 SELECT 语句中引用，但仍然完成了运行 INSERT 语句的任务。

类似地，在与 DML 相关的上下文中，使用 PostgreSQL 的`Insert`结构生成“upsert”：

```py
from sqlalchemy import table, column
from sqlalchemy.dialects.postgresql import insert

t = table("t", column("c1"), column("c2"))

delete_statement_cte = (
    t.delete().where(t.c.c1 < 1).cte("deletions")
)

insert_stmt = insert(t).values({"c1": 1, "c2": 2})
update_statement = insert_stmt.on_conflict_do_update(
    index_elements=[t.c.c1],
    set_={
        "c1": insert_stmt.excluded.c1,
        "c2": insert_stmt.excluded.c2,
    },
).add_cte(delete_statement_cte)

print(update_statement)
```

上述语句呈现为：

```py
WITH deletions AS
(DELETE FROM t WHERE t.c1 < %(c1_1)s)
INSERT INTO t (c1, c2) VALUES (%(c1)s, %(c2)s)
ON CONFLICT (c1) DO UPDATE SET c1 = excluded.c1, c2 = excluded.c2
```

新版本中新增 1.4.21。

参数：

+   `*ctes` –

    零个或多个`CTE`结构。

    2.0 版中的更改：接受多个 CTE 实例

+   `nest_here` –

    如果为 True，则给定的 CTE 或 CTE 将呈现为当它们添加到此`HasCTE`时指定`HasCTE.cte.nesting`标志为`True`。假设给定的 CTE 在外层语句中也没有引用，当给出此标志时，给定的 CTE 应在此语句的级别呈现。

    新版本中新增 2.0。

    另请参阅

    `HasCTE.cte.nesting`

```py
method alias(name: str | None = None, flat: bool = False) → Subquery
```

*继承自* `SelectBase.alias()` *方法的* `SelectBase` *对象*

返回针对此 `SelectBase` 的命名子查询。

对于`SelectBase`（而不是`FromClause`），这将返回一个行为大部分与用于`FromClause`的`Alias`对象相同的`Subquery`对象。

自 1.4 版更改：`SelectBase.alias()` 方法现在是 `SelectBase.subquery()` 方法的同义词。

```py
method as_scalar() → ScalarSelect[Any]
```

*继承自* `SelectBase.as_scalar()` *方法的* `SelectBase` *对象*

自 1.4 版弃用：`SelectBase.as_scalar()` 方法已弃用，并将在将来的版本中删除。请参阅 `SelectBase.scalar_subquery()`。

```py
attribute c
```

*继承自* `SelectBase.c` *属性的* `SelectBase` *对象*

自 1.4 版弃用：`SelectBase.c` 和 `SelectBase.columns` 属性已弃用，并将在将来的版本中删除；这些属性隐式地创建了一个应明确的子查询。请先调用 `SelectBase.subquery()` 以创建一个子查询，然后该子查询包含此属性。要访问此 SELECT 对象从中选择的列，请使用 `SelectBase.selected_columns` 属性。

```py
method compare(other: ClauseElement, **kw: Any) → bool
```

*继承自* `ClauseElement.compare()` *方法的* `ClauseElement`

将此`ClauseElement`与给定的`ClauseElement`进行比较。

子类应该覆盖默认行为，即直接标识比较。

**kw 是子类`compare()`方法消耗的参数，可以用来修改比较的标准（参见`ColumnElement`）。

```py
method compile(bind: _HasDialect | None = None, dialect: Dialect | None = None, **kw: Any) → Compiled
```

*继承自* `CompilerElement.compile()` *方法的* `CompilerElement`

编译此 SQL 表达式。

返回值是一个`Compiled`对象。对返回值调用`str()`或`unicode()`将产生结果的字符串表示。`Compiled`对象还可以使用`params`访问器返回绑定参数名称和值的字典。

参数:

+   `bind` – 一个`Connection`或`Engine`，可以提供一个`Dialect`以生成一个`Compiled`对象。如果`bind`和`dialect`参数都被省略，将使用默认的 SQL 编译器。

+   `column_keys` – 用于 INSERT 和 UPDATE 语句，一个列名列表，应该出现在编译语句的 VALUES 子句中。如果为`None`，则渲染目标表对象的所有列。

+   `dialect` – 一个`Dialect`实例，可以生成一个`Compiled`对象。此参数优先于`bind`参数。

+   `compile_kwargs` –

    可选的字典，包含将传递给编译器在所有“visit”方法中的额外参数。这允许将任何自定义标志传递给自定义编译构造，例如。它还用于通过传递`literal_binds`标志的情况：

    ```py
    from sqlalchemy.sql import table, column, select

    t = table('t', column('x'))

    s = select(t).where(t.c.x == 5)

    print(s.compile(compile_kwargs={"literal_binds": True}))
    ```

另请参阅

如何将 SQL 表达式呈现为字符串，可能包含内联的绑定参数？

```py
method corresponding_column(column: KeyedColumnElement[Any], require_embedded: bool = False) → KeyedColumnElement[Any] | None
```

*继承自* `Selectable.corresponding_column()` *方法的* `Selectable`

给定一个`ColumnElement`，从此`Selectable`的`Selectable.exported_columns`集合中返回与该原始`ColumnElement`通过共同祖先列对应的导出`ColumnElement`对象。

参数：

+   `column` – 要匹配的目标`ColumnElement`。

+   `require_embedded` – 仅返回给定`ColumnElement`的相应列，如果给定的`ColumnElement`实际上存在于此`Selectable`的子元素中。通常，如果列仅与此`Selectable`的导出列之一共享共同的祖先，则列将匹配。

另请参阅

`Selectable.exported_columns` - 用于操作的`ColumnCollection`。

`ColumnCollection.corresponding_column()` - 实现方法。

```py
method cte(name: str | None = None, recursive: bool = False, nesting: bool = False) → CTE
```

*继承自* `HasCTE.cte()` *方法的* `HasCTE`

返回一个新的`CTE`，或者通用表达式实例。

公共表达式是 SQL 标准，其中 SELECT 语句可以在主语句的同时绘制出指定的辅助语句，使用一个称为“WITH”的子句。还可以使用有关 UNION 的特殊语义，允许“递归”查询，其中 SELECT 语句可以利用先前已选择的行集。

CTE 也可以应用于某些数据库上的 DML 构造 UPDATE、INSERT 和 DELETE，既作为与 RETURNING 结合使用时 CTE 行的来源，也作为 CTE 行的消费者。

SQLAlchemy 检测到`CTE`对象，这些对象与`Alias`对象类似，被视为要传递到语句的 FROM 子句以及语句顶部的 WITH 子句的特殊元素。

对于诸如 PostgreSQL 的“MATERIALIZED”和“NOT MATERIALIZED”等特殊前缀，可以使用`CTE.prefix_with()`方法来建立这些。

1.3.13 版中的更改：增加了对前缀的支持。特别是- MATERIALIZED 和 NOT MATERIALIZED。

参数：

+   `name` – 给公共表达式起的名称。与`FromClause.alias()`一样，如果将名称留空，则在查询编译时将使用匿名符号。

+   `recursive` – 如果设置为`True`，将会渲染`WITH RECURSIVE`。递归公共表达式旨在与 UNION ALL 结合使用，以从已选择的行派生行。

+   `nesting` –

    如果设置为`True`，将在引用它的语句中将 CTE 本地渲染。对于更复杂的场景，还可以使用`HasCTE.add_cte()`方法，使用`HasCTE.add_cte.nest_here`参数更精确地控制特定 CTE 的确切放置。

    新版 1.4.24 中新增。

    另请参阅

    `HasCTE.add_cte()`

以下示例包括两个来自 PostgreSQL 文档的示例，网址为[`www.postgresql.org/docs/current/static/queries-with.html`](https://www.postgresql.org/docs/current/static/queries-with.html)，以及其他示例。

示例 1，非递归：

```py
from sqlalchemy import (Table, Column, String, Integer,
                        MetaData, select, func)

metadata = MetaData()

orders = Table('orders', metadata,
    Column('region', String),
    Column('amount', Integer),
    Column('product', String),
    Column('quantity', Integer)
)

regional_sales = select(
                    orders.c.region,
                    func.sum(orders.c.amount).label('total_sales')
                ).group_by(orders.c.region).cte("regional_sales")

top_regions = select(regional_sales.c.region).\
        where(
            regional_sales.c.total_sales >
            select(
                func.sum(regional_sales.c.total_sales) / 10
            )
        ).cte("top_regions")

statement = select(
            orders.c.region,
            orders.c.product,
            func.sum(orders.c.quantity).label("product_units"),
            func.sum(orders.c.amount).label("product_sales")
    ).where(orders.c.region.in_(
        select(top_regions.c.region)
    )).group_by(orders.c.region, orders.c.product)

result = conn.execute(statement).fetchall()
```

示例 2，WITH RECURSIVE：

```py
from sqlalchemy import (Table, Column, String, Integer,
                        MetaData, select, func)

metadata = MetaData()

parts = Table('parts', metadata,
    Column('part', String),
    Column('sub_part', String),
    Column('quantity', Integer),
)

included_parts = select(\
    parts.c.sub_part, parts.c.part, parts.c.quantity\
    ).\
    where(parts.c.part=='our part').\
    cte(recursive=True)

incl_alias = included_parts.alias()
parts_alias = parts.alias()
included_parts = included_parts.union_all(
    select(
        parts_alias.c.sub_part,
        parts_alias.c.part,
        parts_alias.c.quantity
    ).\
    where(parts_alias.c.part==incl_alias.c.sub_part)
)

statement = select(
            included_parts.c.sub_part,
            func.sum(included_parts.c.quantity).
              label('total_quantity')
        ).\
        group_by(included_parts.c.sub_part)

result = conn.execute(statement).fetchall()
```

示例 3，使用 UPDATE 和 INSERT 进行的 upsert 操作与 CTE 一起：

```py
from datetime import date
from sqlalchemy import (MetaData, Table, Column, Integer,
                        Date, select, literal, and_, exists)

metadata = MetaData()

visitors = Table('visitors', metadata,
    Column('product_id', Integer, primary_key=True),
    Column('date', Date, primary_key=True),
    Column('count', Integer),
)

# add 5 visitors for the product_id == 1
product_id = 1
day = date.today()
count = 5

update_cte = (
    visitors.update()
    .where(and_(visitors.c.product_id == product_id,
                visitors.c.date == day))
    .values(count=visitors.c.count + count)
    .returning(literal(1))
    .cte('update_cte')
)

upsert = visitors.insert().from_select(
    [visitors.c.product_id, visitors.c.date, visitors.c.count],
    select(literal(product_id), literal(day), literal(count))
        .where(~exists(update_cte.select()))
)

connection.execute(upsert)
```

示例 4，嵌套 CTE（SQLAlchemy 1.4.24 及以上版本）：

```py
value_a = select(
    literal("root").label("n")
).cte("value_a")

# A nested CTE with the same name as the root one
value_a_nested = select(
    literal("nesting").label("n")
).cte("value_a", nesting=True)

# Nesting CTEs takes ascendency locally
# over the CTEs at a higher level
value_b = select(value_a_nested.c.n).cte("value_b")

value_ab = select(value_a.c.n.label("a"), value_b.c.n.label("b"))
```

上述查询将在第一个 CTE 内嵌套第二个 CTE，如下所示，内联参数如下：

```py
WITH
    value_a AS
        (SELECT 'root' AS n),
    value_b AS
        (WITH value_a AS
            (SELECT 'nesting' AS n)
        SELECT value_a.n AS n FROM value_a)
SELECT value_a.n AS a, value_b.n AS b
FROM value_a, value_b
```

可以使用`HasCTE.add_cte()`方法设置相同的 CTE，如下所示（SQLAlchemy 2.0 及以上版本）：

```py
value_a = select(
    literal("root").label("n")
).cte("value_a")

# A nested CTE with the same name as the root one
value_a_nested = select(
    literal("nesting").label("n")
).cte("value_a")

# Nesting CTEs takes ascendency locally
# over the CTEs at a higher level
value_b = (
    select(value_a_nested.c.n).
    add_cte(value_a_nested, nest_here=True).
    cte("value_b")
)

value_ab = select(value_a.c.n.label("a"), value_b.c.n.label("b"))
```

示例 5，非线性 CTE（SQLAlchemy 1.4.28 及以上版本）：

```py
edge = Table(
    "edge",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("left", Integer),
    Column("right", Integer),
)

root_node = select(literal(1).label("node")).cte(
    "nodes", recursive=True
)

left_edge = select(edge.c.left).join(
    root_node, edge.c.right == root_node.c.node
)
right_edge = select(edge.c.right).join(
    root_node, edge.c.left == root_node.c.node
)

subgraph_cte = root_node.union(left_edge, right_edge)

subgraph = select(subgraph_cte)
```

上述查询将在递归 CTE 内呈现 2 个 UNIONs：

```py
WITH RECURSIVE nodes(node) AS (
        SELECT 1 AS node
    UNION
        SELECT edge."left" AS "left"
        FROM edge JOIN nodes ON edge."right" = nodes.node
    UNION
        SELECT edge."right" AS "right"
        FROM edge JOIN nodes ON edge."left" = nodes.node
)
SELECT nodes.node FROM nodes
```

另见

`Query.cte()` - `HasCTE.cte()` 的 ORM 版本。

```py
method execution_options(**kw: Any) → Self
```

*继承自* `Executable.execution_options()` *方法的* `Executable`

为在执行期间生效的语句设置非 SQL 选项。

可以在许多范围内设置执行选项，包括每个语句、每个连接或每个执行，使用诸如 `Connection.execution_options()` 这样的方法以及接受选项字典的参数，例如 `Connection.execute.execution_options` 和 `Session.execute.execution_options`。

与其他类型的选项（例如 ORM 加载程序选项）相比，执行选项的主要特征是**执行选项从不影响查询的编译 SQL，只影响 SQL 语句本身如何被调用或结果如何获取**。也就是说，执行选项不是 SQL 编译所容纳的部分，也不被视为语句的缓存状态的一部分。

`Executable.execution_options()` 方法是生成的，与应用于 `Engine` 和 `Query` 对象的方法相同，这意味着当调用该方法时，将返回对象的副本，该副本应用给定的参数，但原始对象保持不变：

```py
statement = select(table.c.x, table.c.y)
new_statement = statement.execution_options(my_option=True)
```

对此行为的一个例外是 `Connection` 对象，在该对象上，`Connection.execution_options()` 方法明确地**不**是生成的。

可传递给`Executable.execution_options()`和其他相关方法和参数字典的选项类型包括被 SQLAlchemy Core 或 ORM 明确消耗的参数，以及 SQLAlchemy 未定义的任意关键字参数，这意味着这些方法和/或参数字典可用于与自定义代码交互的用户定义参数，可以使用诸如`Executable.get_execution_options()`和`Connection.get_execution_options()`等方法访问这些参数，或者在选定的事件钩子中使用专用的`execution_options`事件参数，例如`ConnectionEvents.before_execute.execution_options`或`ORMExecuteState.execution_options`，例如：

```py
from sqlalchemy import event

@event.listens_for(some_engine, "before_execute")
def _process_opt(conn, statement, multiparams, params, execution_options):
    "run a SQL function before invoking a statement"

    if execution_options.get("do_special_thing", False):
        conn.exec_driver_sql("run_special_function()")
```

在 SQLAlchemy 明确识别的选项范围内，大多数适用于特定类的对象而不是其他对象。最常见的执行选项包括：

+   `Connection.execution_options.isolation_level` - 设置连接或一类连接的隔离级别，通过`Engine`接受此选项。此选项仅被`Connection`或`Engine`接受。

+   `Connection.execution_options.stream_results` - 指示应使用服务器端游标获取结果；此选项由`Connection`接受，由`Connection.execute.execution_options`参数传递给`Connection.execute()`，以及由 SQL 语句对象的`Executable.execution_options()`以及 ORM 构造函数如`Session.execute()`附加。

+   `Connection.execution_options.compiled_cache` - 指示将用作`Connection`或`Engine`的 SQL 编译缓存的字典，以及 ORM 方法如`Session.execute()`。可以将其传递为`None`以禁用语句的缓存。此选项不被`Executable.execution_options()`接受，因为将编译缓存随语句对象一起传递是不明智的。

+   `Connection.execution_options.schema_translate_map` - 用于模式翻译映射功能的模式名称映射，由`Connection`，`Engine`，`Executable`接受，以及 ORM 构造函数如`Session.execute()`。

另见

`Connection.execution_options()`

`Connection.execute.execution_options`

`Session.execute.execution_options`

ORM 执行选项 - 关于所有 ORM 特定执行选项的文档

```py
method exists() → Exists
```

*继承自* `SelectBase.exists()` *方法，属于* `SelectBase`

返回此可选择的 `Exists` 表示，可用作列表达式。

返回的对象是 `Exists` 的一个实例。

另请参阅

`exists()`

EXISTS 子查询 - 在 2.0 样式 教程中。

自版本 1.4 起新增。

```py
attribute exported_columns
```

*继承自* `SelectBase.exported_columns` *属性，属于* `SelectBase`

一个 `ColumnCollection` 代表此 `Selectable` 的“导出”列，不包括 `TextClause` 构造。

`SelectBase` 对象的“导出”列与 `SelectBase.selected_columns` 集合是同义词。

自版本 1.4 起新增。

另请参阅

`Select.exported_columns`

`Selectable.exported_columns`

`FromClause.exported_columns`

```py
method get_children(*, omit_attrs: Tuple[str, ...] = (), **kw: Any) → Iterable[HasTraverseInternals]
```

*继承自* `HasTraverseInternals.get_children()` *方法，属于* `HasTraverseInternals`

返回此 `HasTraverseInternals` 的直接子`HasTraverseInternals`元素。

用于访问遍历。

**kw 可能包含更改返回集合的标志，例如返回子集以减少较大的遍历，或者从不同上下文返回子项（例如模式级别的集合而不是子句级别的集合）。

```py
method get_execution_options() → _ExecuteOptions
```

*继承自* `Executable.get_execution_options()` *方法的* `Executable`

获取在执行期间生效的非 SQL 选项。

版本 1.3 中新增。

另请参阅

`Executable.execution_options()`

```py
method get_label_style() → SelectLabelStyle
```

*继承自* `SelectBase.get_label_style()` *方法的* `SelectBase`

检索当前的标签样式。

子类实现。

```py
attribute inherit_cache: bool | None = None
```

*继承自* `HasCacheKey.inherit_cache` *属性的* `HasCacheKey`

指示此 `HasCacheKey` 实例是否应使用其直接超类使用的缓存键生成方案。

此属性默认为 `None`，表示构造尚未考虑其是否适合参与缓存；这在功能上等同于将值设置为 `False`，只是还会发出警告。

如果与此类本地属性相关且不是其超类的属性，则可以在特定类上将此标志设置为 `True`，并且对象对应的 SQL 不会根据这些属性而变化。

另请参阅

为自定义构造启用缓存支持 - 设置第三方或用户定义的 SQL 构造的 `HasCacheKey.inherit_cache` 属性的一般指南。

```py
method is_derived_from(fromclause: FromClause | None) → bool
```

*继承自* `ReturnsRows.is_derived_from()` *方法的* `ReturnsRows`

如果此 `ReturnsRows` *衍生自* 给定的 `FromClause`，则返回 `True`。

例如，表的别名源自该表。

```py
method label(name: str | None) → Label[Any]
```

*继承自* `SelectBase.label()` *方法的* `SelectBase`

返回此可选择的‘标量’表示形式，嵌入为带有标签的子查询。

另请参阅

`SelectBase.scalar_subquery()`。

```py
method lateral(name: str | None = None) → LateralFromClause
```

*继承自* `SelectBase.lateral()` *方法的* `SelectBase`

返回此`Selectable`的 LATERAL 别名。

返回值是`Lateral`结构，由顶层`lateral()`函数提供。

也请参见

LATERAL correlation - 用法概述。

```py
method options(*options: ExecutableOption) → Self
```

*继承自* `Executable.options()` *方法的* `Executable`

对此语句应用选项。

一般来说，选项是可以被 SQL 编译器解释为语句的任何类型的 Python 对象。这些选项可以被特定方言或特定类型的编译器消耗。

最常见的选项类型是应用于 ORM 查询的 ORM 级别选项，它们可以将“急加载”和其他加载行为应用于 ORM 查询。但是，理论上选项可以用于许多其他目的。

关于特定类型语句的特定类型选项的背景，请参阅这些选项对象的文档。

在版本 1.4 中更改： - 向 Core 语句对象添加了`Executable.options()`，以实现统一的 Core / ORM 查询功能。

也请参见

列加载选项 - 指的是用于 ORM 查询的特定选项

使用加载器选项进行关系加载 - 指的是用于 ORM 查询的特定选项

```py
method params(_ClauseElement__optionaldict: Mapping[str, Any] | None = None, **kwargs: Any) → Self
```

*继承自* `ClauseElement.params()` *方法的* `ClauseElement`

返回一个副本，其中`bindparam()`元素已替换。

返回此 ClauseElement 的副本，其中的`bindparam()`元素已替换为从给定字典中取出的值：

```py
>>> clause = column('x') + bindparam('foo')
>>> print(clause.compile().params)
{'foo':None}
>>> print(clause.params({'foo':7}).compile().params)
{'foo':7}
```

```py
method replace_selectable(old: FromClause, alias: Alias) → Self
```

*继承自* `Selectable.replace_selectable()` *方法的* `Selectable` *类方法*

将 `FromClause` *中所有出现的* ‘old’ *替换为给定的* `Alias` *对象，并返回此* `FromClause` *的副本。*

自版本 1.4 起已弃用：`Selectable.replace_selectable()` *方法已弃用，并将在将来的版本中删除。类似功能可通过 sqlalchemy.sql.visitors 模块获得。*

```py
method scalar_subquery() → ScalarSelect[Any]
```

*继承自* `SelectBase.scalar_subquery()` *方法的* `SelectBase` *类方法*

返回此可选对象的 ‘scalar’ 表示形式，可用作列表达式。

返回的对象是 `ScalarSelect` *的一个实例。*

通常，列子句中只有一个列的 select 语句可以用作标量表达式。然后可以在封闭的 SELECT 的 WHERE 子句或列子句中使用标量子查询。

请注意，标量子查询与使用 `SelectBase.subquery()` *方法生成的 FROM 级子查询不同。*

另请参阅

标量和相关子查询 - 在 2.0 教程中

```py
method select(*arg: Any, **kw: Any) → Select
```

*继承自* `SelectBase.select()` *方法的* `SelectBase` *类方法*

自版本 1.4 起已弃用：`SelectBase.select()` *方法已弃用，并将在将来的版本中删除；此方法隐式创建应为明确的子查询。请首先调用 `SelectBase.subquery()` *以创建子查询，然后可以选择该子查询。*

```py
attribute selected_columns
```

一个代表该 SELECT 语句或类似结构在其结果集中返回的列的 `ColumnCollection`，不包括 `TextClause` 结构。

此集合与 `FromClause.columns` 的集合不同，后者不能直接嵌套在另一个 SELECT 语句中；必须先应用子查询，这样就提供了 SQL 所需的必要括号。

对于 `TextualSelect` 构造，该集合包含通过构造函数传递的 `ColumnElement` 对象，通常是通过 `TextClause.columns()` 方法传递的。

自 1.4 版本新增。

```py
method self_group(against: OperatorType | None = None) → ClauseElement
```

*继承自* `ClauseElement.self_group()` *方法的* `ClauseElement`

对这个 `ClauseElement` 应用“分组”。

子类覆盖此方法以返回一个“分组”结构，即括号。特别是，当“二元”表达式放置到更大的表达式中时，它们用于在自身周围提供分组，以及当 `select()` 结构放置到另一个 `select()` 的 FROM 子句中时。（请注意，通常应使用 `Select.alias()` 方法创建子查询，因为许多平台要求嵌套的 SELECT 语句必须具有名称）。

随着表达式的组合，`self_group()` 的应用是自动的 - 最终用户代码不应该直接使用此方法。请注意，SQLAlchemy 的子句构造会考虑操作符优先级 - 因此在表达式 `x OR (y AND z)` 中可能不需要括号 - AND 优先于 OR。

`ClauseElement` 的基本 `self_group()` 方法只返回自身。

```py
method set_label_style(style: SelectLabelStyle) → TextualSelect
```

返回一个使用指定标签样式的新可选项。

由子类实现。

```py
method subquery(name: str | None = None) → Subquery
```

*继承自* `SelectBase.subquery()` *方法的* `SelectBase`

返回这个`SelectBase`的子查询。

从 SQL 角度来看，子查询是一个带有括号的命名结构，可以放置在另一个 SELECT 语句的 FROM 子句中。

给定一个 SELECT 语句，比如：

```py
stmt = select(table.c.id, table.c.name)
```

上述语句可能如下所示：

```py
SELECT table.id, table.name FROM table
```

子查询本身的形式是相同的，但是当嵌入到另一个 SELECT 语句的 FROM 子句中时，它变成了一个命名的子元素：

```py
subq = stmt.subquery()
new_stmt = select(subq)
```

上述内容呈现为：

```py
SELECT anon_1.id, anon_1.name
FROM (SELECT table.id, table.name FROM table) AS anon_1
```

从历史上看，`SelectBase.subquery()` 相当于在 FROM 对象上调用 `FromClause.alias()` 方法；但是，由于 `SelectBase` 对象不是直接的 FROM 对象，所以 `SelectBase.subquery()` 方法提供了更清晰的语义。

版本 1.4 中新增。

```py
method unique_params(_ClauseElement__optionaldict: Dict[str, Any] | None = None, **kwargs: Any) → Self
```

*继承自* `ClauseElement.unique_params()` *方法的* `ClauseElement`

返回一个副本，其中的 `bindparam()` 元素被替换。

与 `ClauseElement.params()` 功能相同，只是对受影响的绑定参数添加了 unique=True，以便可以使用多个语句。

```py
class sqlalchemy.sql.expression.Values
```

表示可以在语句中作为 FROM 元素使用的 `VALUES` 结构。

`Values` 对象是从 `values()` 函数创建的。

版本 1.4 中新增。

**成员**

alias(), data(), lateral(), scalar_values()

**类签名**

类 `sqlalchemy.sql.expression.Values` (`sqlalchemy.sql.roles.InElementRole`, `sqlalchemy.sql.expression.Generative`, `sqlalchemy.sql.expression.LateralFromClause`)。

```py
method alias(name: str | None = None, flat: bool = False) → Self
```

返回一个新的 `Values` 构造，其名称与给定名称相同。

该方法是 `FromClause.alias()` 方法的 VALUES 特定的专业化。

另请参阅

使用别名

`alias()`

```py
method data(values: Sequence[Tuple[Any, ...]]) → Self
```

返回一个新的 `Values` 构造，将给定数据添加到数据列表中。

例如：

```py
my_values = my_values.data([(1, 'value 1'), (2, 'value2')])
```

参数：

**values** – 一个序列（即列表），其中的元组与 `Values` 构造中给出的列表达式相对应。

```py
method lateral(name: str | None = None) → LateralFromClause
```

返回一个新的 `Values`，并将侧面标志设置为 LATERAL，以便其渲染为 LATERAL。

另请参阅

`lateral()`

```py
method scalar_values() → ScalarValues
```

返回一个标量 `VALUES` 构造，可用作语句中的 COLUMN 元素。

新特性版本为 2.0.0b4。

```py
class sqlalchemy.sql.expression.ScalarValues
```

表示可用作语句中的 COLUMN 元素的标量 `VALUES` 构造。

`ScalarValues` 对象是通过 `Values.scalar_values()` 方法创建的。当在 `IN` 或 `NOT IN` 条件中使用 `Values` 时，它也会自动生成。

新特性版本为 2.0.0b4。

**类签名**

类 `sqlalchemy.sql.expression.ScalarValues` (`sqlalchemy.sql.roles.InElementRole`, `sqlalchemy.sql.expression.GroupedElement`, `sqlalchemy.sql.expression.ColumnElement`)。

## 标签样式常量

与 `GenerativeSelect.set_label_style()` 方法一起使用的常量。

| 对象名称 | 描述 |
| --- | --- |
| SelectLabelStyle | 可以传递给 `Select.set_label_style()` 的标签样式常量。 |

```py
class sqlalchemy.sql.expression.SelectLabelStyle
```

可传递给`Select.set_label_style()`的标签样式常量。

**成员**

LABEL_STYLE_DEFAULT，LABEL_STYLE_DISAMBIGUATE_ONLY，LABEL_STYLE_NONE，LABEL_STYLE_TABLENAME_PLUS_COL

**类签名**

类`sqlalchemy.sql.expression.SelectLabelStyle` (`enum.Enum`)

```py
attribute LABEL_STYLE_DEFAULT = 2
```

默认标签样式，指的是`LABEL_STYLE_DISAMBIGUATE_ONLY`。

1.4 版本中的新功能。

```py
attribute LABEL_STYLE_DISAMBIGUATE_ONLY = 2
```

表示当列名与现有名称冲突时，应在生成 SELECT 语句的 columns 子句时使用半匿名标签对列进行标记的标签样式。

下面，大多数列名不受影响，除了名称为`columna`的第二个出现，它使用标签`columna_1`来区分它和`tablea.columna`的列名：

```py
>>> from sqlalchemy import table, column, select, true, LABEL_STYLE_DISAMBIGUATE_ONLY
>>> table1 = table("table1", column("columna"), column("columnb"))
>>> table2 = table("table2", column("columna"), column("columnc"))
>>> print(select(table1, table2).join(table2, true()).set_label_style(LABEL_STYLE_DISAMBIGUATE_ONLY))
SELECT  table1.columna,  table1.columnb,  table2.columna  AS  columna_1,  table2.columnc
FROM  table1  JOIN  table2  ON  true 
```

与`GenerativeSelect.set_label_style()`方法一起使用，`LABEL_STYLE_DISAMBIGUATE_ONLY`是除了 1.x 风格 ORM 查询之外的所有 SELECT 语句的默认标签样式。

1.4 版本中的新功能。

```py
attribute LABEL_STYLE_NONE = 0
```

表示不应将自动标签应用于 SELECT 语句的 columns 子句的标签样式。

下面，列名为`columna`的列都按原样呈现，这意味着名称`columna`只能引用结果集中的第一个出现的这个名称，以及如果语句被用作子查询时：

```py
>>> from sqlalchemy import table, column, select, true, LABEL_STYLE_NONE
>>> table1 = table("table1", column("columna"), column("columnb"))
>>> table2 = table("table2", column("columna"), column("columnc"))
>>> print(select(table1, table2).join(table2, true()).set_label_style(LABEL_STYLE_NONE))
SELECT  table1.columna,  table1.columnb,  table2.columna,  table2.columnc
FROM  table1  JOIN  table2  ON  true 
```

与`Select.set_label_style()`方法一起使用。

1.4 版本中的新功能。

```py
attribute LABEL_STYLE_TABLENAME_PLUS_COL = 1
```

表示在生成 SELECT 语句的 columns 子句时，所有列都应标记为`<tablename>_<columnname>`，以消除从不同表、别名或子查询引用的同名列的歧义。

下面，所有列名都被赋予标签，以便两个同名列`columna`被区分为`table1_columna`和`table2_columna`：

```py
>>> from sqlalchemy import table, column, select, true, LABEL_STYLE_TABLENAME_PLUS_COL
>>> table1 = table("table1", column("columna"), column("columnb"))
>>> table2 = table("table2", column("columna"), column("columnc"))
>>> print(select(table1, table2).join(table2, true()).set_label_style(LABEL_STYLE_TABLENAME_PLUS_COL))
SELECT  table1.columna  AS  table1_columna,  table1.columnb  AS  table1_columnb,  table2.columna  AS  table2_columna,  table2.columnc  AS  table2_columnc
FROM  table1  JOIN  table2  ON  true 
```

与`GenerativeSelect.set_label_style()`方法一起使用。等效于传统方法`Select.apply_labels()`；`LABEL_STYLE_TABLENAME_PLUS_COL`是 SQLAlchemy 的传统自动标签样式。`LABEL_STYLE_DISAMBIGUATE_ONLY`提供了一种较少侵入性的方法来消除同名列表达式的歧义。

1.4 版本中的新功能。

另请参阅

`Select.set_label_style()`

`Select.get_label_style()`  
