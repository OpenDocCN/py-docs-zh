# 插入，更新，删除

> 原文：[`docs.sqlalchemy.org/en/20/core/dml.html`](https://docs.sqlalchemy.org/en/20/core/dml.html)

INSERT、UPDATE 和 DELETE 语句是基于从 `UpdateBase` 开始的层次结构构建的。`Insert` 和 `Update` 构造基于中介 `ValuesBase` 构建。 

## DML 基础构造函数

最顶层的“INSERT”，“UPDATE”，“DELETE”构造函数。

| 对象名称 | 描述 |
| --- | --- |
| delete(table) | 构造 `Delete` 对象。 |
| insert(table) | 构造 `Insert` 对象。 |
| update(table) | 构造一个 `Update` 对象。 |

```py
function sqlalchemy.sql.expression.delete(table: _DMLTableArgument) → Delete
```

构造 `Delete` 对象。

例如：

```py
from sqlalchemy import delete

stmt = (
    delete(user_table).
    where(user_table.c.id == 5)
)
```

相似的功能也可以通过 `TableClause.delete()` 方法在 `Table` 上使用。

参数：

**table** – 要从中删除行的表。

参见

使用 UPDATE 和 DELETE 语句 - 在 SQLAlchemy 统一教程 中

```py
function sqlalchemy.sql.expression.insert(table: _DMLTableArgument) → Insert
```

构造一个 `Insert` 对象。

例如：

```py
from sqlalchemy import insert

stmt = (
    insert(user_table).
    values(name='username', fullname='Full Username')
)
```

相似的功能也可以通过 `TableClause.insert()` 方法在 `Table` 上使用。

参见

使用 INSERT 语句 - 在 SQLAlchemy 统一教程 中

参数：

+   `table` – `TableClause` 插入的主题。

+   `values` – 要插入的值的集合；参见`Insert.values()`以获取这里允许的格式描述。可以完全省略；`Insert`构造也会根据传递给`Connection.execute()`的参数在执行时动态渲染 VALUES 子句。

+   `inline` – 如果为 True，则不会尝试检索 SQL 生成的默认值以在语句中提供；特别是，这允许 SQL 表达式在语句中“内联”渲染，而无需事先预先执行它们；对于支持“返回”的后端，这将关闭语句的“隐式返回”功能。

如果同时存在`insert.values`和编译时绑定参数，则编译时绑定参数将在每个键的基础上覆盖`insert.values`中指定的信息。

`Insert.values`中的键可以是`Column`对象或它们的字符串标识符。每个键可以引用以下之一：

+   一个字面数据值（即字符串，数字等）;

+   一个 Column 对象;

+   一个 SELECT 语句。

如果指定了引用此`INSERT`语句表的`SELECT`语句，则该语句将与`INSERT`语句相关联。

另请参阅

使用 INSERT 语句 - 在 SQLAlchemy 统一教程中

```py
function sqlalchemy.sql.expression.update(table: _DMLTableArgument) → Update
```

构造一个`Update`对象。

例如：

```py
from sqlalchemy import update

stmt = (
    update(user_table).
    where(user_table.c.id == 5).
    values(name='user #5')
)
```

通过`TableClause.update()`方法在`Table`上也可以实现类似功能。

参数：

**table** – 代表要更新的数据库表的`Table`对象。

另请参阅

使用 UPDATE 和 DELETE 语句 - 在 SQLAlchemy 统一教程中

## DML 类文档构造函数

DML 基础构造函数的类构造函数文档。

| 对象名称 | 描述 |
| --- | --- |
| Delete | 代表一个 DELETE 构造。 |
| 插入 | 代表一个插入操作。 |
| 更新 | 代表一个更新操作。 |
| UpdateBase | 形成 `INSERT`、`UPDATE` 和 `DELETE` 语句的基础。 |
| ValuesBase | 为 `ValuesBase.values()` 提供对 INSERT 和 UPDATE 构造的支持。 |

```py
class sqlalchemy.sql.expression.Delete
```

代表一个删除操作。

使用 `delete()` 函数创建 `Delete` 对象。

**成员**

where(), returning()

**类签名**

类 `sqlalchemy.sql.expression.Delete` (`sqlalchemy.sql.expression.DMLWhereBase`, `sqlalchemy.sql.expression.UpdateBase`)

```py
method where(*whereclause: _ColumnExpressionArgument[bool]) → Self
```

*继承自* `DMLWhereBase.where()` *方法的* `DMLWhereBase`

返回一个新构造，其中给定的表达式已添加到其 WHERE 子句中，如果有的话，通过 AND 连接到现有子句。

`Update.where()` 和 `Delete.where()` 都支持多表形式，包括特定于数据库的 `UPDATE...FROM` 和 `DELETE..USING`。对于不支持多表的后端，使用多表的跨后端方法是利用相关子查询。查看下面链接的教程部分以获取示例。

另请参见

相关更新

UPDATE..FROM

多表删除

```py
method returning(*cols: _ColumnsClauseArgument[Any], sort_by_parameter_order: bool = False, **_UpdateBase__kw: Any) → UpdateBase
```

*继承自* `UpdateBase.returning()` *方法的* `UpdateBase`

为该语句添加一个 RETURNING 或等效子句。

例如：

```py
>>> stmt = (
...     table.update()
...     .where(table.c.data == "value")
...     .values(status="X")
...     .returning(table.c.server_flag, table.c.updated_timestamp)
... )
>>> print(stmt)
UPDATE  some_table  SET  status=:status
WHERE  some_table.data  =  :data_1
RETURNING  some_table.server_flag,  some_table.updated_timestamp 
```

该方法可以多次调用，以将新条目添加到要返回的表达式列表中。

版本 1.4.0b2 中的新功能：该方法可以多次调用，以将新条目添加到要返回的表达式列表中。

给定的列表达式集合应来源于 INSERT、UPDATE 或 DELETE 的目标表。虽然 `Column` 对象是典型的，但元素也可以是表达式：

```py
>>> stmt = table.insert().returning(
...     (table.c.first_name + " " + table.c.last_name).label("fullname")
... )
>>> print(stmt)
INSERT  INTO  some_table  (first_name,  last_name)
VALUES  (:first_name,  :last_name)
RETURNING  some_table.first_name  ||  :first_name_1  ||  some_table.last_name  AS  fullname 
```

在编译时，RETURNING 子句或数据库等效项将在语句中呈现。对于 INSERT 和 UPDATE，值是新插入/更新的值。对于 DELETE，值是删除的行的值。

在执行时，要返回的列的值通过结果集可用，并可使用`CursorResult.fetchone()`等进行迭代。对于不原生支持返回值的 DBAPI（即 cx_oracle），SQLAlchemy 将在结果级别上近似此行为，以便提供合理的行为中立性。

请注意，并非所有数据库/DBAPI 支持 RETURNING。对于没有支持的后端，在编译和/或执行时会引发异常。对于支持它的后端，跨后端的功能差异很大，包括对 executemany() 和其他返回多行的语句的限制。请阅读正在使用的数据库的文档注释，以确定 RETURNING 的可用性。

参数：

+   `*cols` – 一系列列、SQL 表达式或整个表实体要返回。

+   `sort_by_parameter_order` –

    对于正在针对多个参数集执行的批量 INSERT，请组织 RETURNING 的结果，使返回的行与传入的参数集的顺序相对应。这仅适用于支持方言的 executemany 执行，并且通常利用 insertmanyvalues 功能。

    从版本 2.0.10 开始。

    另请参阅

    将 RETURNING 行与参数集相关联 - 关于批量 INSERT 的 RETURNING 行排序的背景（核心级别讨论）

    将 RETURNING 记录与输入数据顺序相关联 - 与 ORM 批量 INSERT 语句 的使用示例（ORM 级别讨论）

另请参阅

`UpdateBase.return_defaults()` - 针对单行 INSERT 或 UPDATE，旨在有效地获取服务器端默认值和触发器的替代方法。

INSERT…RETURNING - 在 SQLAlchemy 统一教程中

```py
class sqlalchemy.sql.expression.Insert
```

表示一个 INSERT 结构。

`Insert` 对象是使用 `insert()` 函数创建的。

**成员**

values(), returning(), from_select(), inline(), select

**类签名**

类 `sqlalchemy.sql.expression.Insert` (`sqlalchemy.sql.expression.ValuesBase`)

```py
method values(*args: _DMLColumnKeyMapping[Any] | Sequence[Any], **kwargs: Any) → Self
```

*继承自* `ValuesBase.values()` *方法的* `ValuesBase`

为 INSERT 语句指定固定的 VALUES 子句，或为 UPDATE 指定 SET 子句。

请注意，`Insert` 和 `Update` 构造支持根据传递给 `Connection.execute()` 的参数，在执行时格式化 VALUES 和/或 SET 子句。然而，`ValuesBase.values()` 方法可以用于将特定一组参数“固定”到语句中。

多次调用 `ValuesBase.values()` 将产生一个新的构造，每个构造的参数列表都会被修改以包含新传入的参数。在典型情况下，使用单个参数字典，新传入的键将替换前一个构造中的相同键。在基于列表的“多值”构造中，每个新的值列表都会被扩展到现有的值列表上。

参数：

+   `**kwargs` –

    键值对表示映射到要渲染到 VALUES 或 SET 子句中的值的 `Column` 的字符串键：

    ```py
    users.insert().values(name="some name")

    users.update().where(users.c.id==5).values(name="some name")
    ```

+   `*args` –

    作为传递键/值参数的替代方案，可以将字典、元组或字典或元组的列表作为单个位置参数传递，以形成语句的 VALUES 或 SET 子句。接受的形式因为是 `Insert` 还是 `Update` 构造而��所不同。

    对于 `Insert` 或 `Update` 构造，可以传递一个单独的字典，其工作方式与 kwargs 形式相同：

    ```py
    users.insert().values({"name": "some name"})

    users.update().values({"name": "some new name"})
    ```

    对于任何形式，但更常见于 `Insert` 构造，也可以接受包含表中每一列的条目的元组：

    ```py
    users.insert().values((5, "some name"))
    ```

    `Insert` 构造还支持传递字典或完整表元组的列表，在服务器上会呈现较少见的 SQL 语法“多个值” - 这种语法在后端如 SQLite、PostgreSQL、MySQL 等支持，但不一定适用于其他后端：

    ```py
    users.insert().values([
                        {"name": "some name"},
                        {"name": "some other name"},
                        {"name": "yet another name"},
                    ])
    ```

    上述形式将呈现类似于多个 VALUES 语句：

    ```py
    INSERT INTO users (name) VALUES
                    (:name_1),
                    (:name_2),
                    (:name_3)
    ```

    必须注意，**传递多个值并不等同于使用传统的 executemany()形式**。上述语法是一种**特殊**语法，通常不使用。要针对多行发出 INSERT 语句，正常的方法是将多个值列表传递给 `Connection.execute()` 方法，该方法受到所有数据库后端的支持，并且通常对大量参数更有效率。

    > > 另请参见
    > > 
    > > 发送多个参数 - 介绍了用于 INSERT 和其他语句的多参数集调用的传统 Core 方法。
    > > 
    > UPDATE 构造还支持以特定顺序渲染 SET 参数。有关此功能，请参阅 `Update.ordered_values()` 方法。
    > 
    > > 另请参见
    > > 
    > > `Update.ordered_values()`

```py
method returning(*cols: _ColumnsClauseArgument[Any], sort_by_parameter_order: bool = False, **_UpdateBase__kw: Any) → UpdateBase
```

*继承自* `UpdateBase.returning()` *方法的* `UpdateBase`

为此语句添加一个 RETURNING 或等效子句。

例如：

```py
>>> stmt = (
...     table.update()
...     .where(table.c.data == "value")
...     .values(status="X")
...     .returning(table.c.server_flag, table.c.updated_timestamp)
... )
>>> print(stmt)
UPDATE  some_table  SET  status=:status
WHERE  some_table.data  =  :data_1
RETURNING  some_table.server_flag,  some_table.updated_timestamp 
```

可以多次调用该方法以向返回的表达式列表中添加新条目。

新版本 1.4.0b2 中：可以多次调用该方法以向返回的表达式列表中添加新条目。

给定的列表达式集合应派生自 INSERT、UPDATE 或 DELETE 的目标表。虽然 `Column` 对象是典型的，但元素也可以是表达式：

```py
>>> stmt = table.insert().returning(
...     (table.c.first_name + " " + table.c.last_name).label("fullname")
... )
>>> print(stmt)
INSERT  INTO  some_table  (first_name,  last_name)
VALUES  (:first_name,  :last_name)
RETURNING  some_table.first_name  ||  :first_name_1  ||  some_table.last_name  AS  fullname 
```

在编译时，将在语句中呈现 RETURNING 子句或数据库等效物。对于 INSERT 和 UPDATE，值是新插入/更新的值。对于 DELETE，值是被删除的行的值。

在执行时，要返回的列的值通过结果集可用，并且可以使用 `CursorResult.fetchone()` 等进行迭代。对于不原生支持返回值的 DBAPI（即 cx_oracle），SQLAlchemy 将在结果级别近似此行为，以便提供合理数量的行为中立性。

请注意，并非所有数据库/DBAPI 都支持 RETURNING。对于那些没有支持的后端，在编译和/或执行时会引发异常。对于支持它的后端，跨后端的功能差异很大，包括对 executemany() 和其他返回多行语句的限制。请阅读正在使用的数据库的文档注释，以确定 RETURNING 的可用性。

参数：

+   `*cols` – 要返回的一系列列、SQL 表达式或整个表实体。

+   `sort_by_parameter_order` –

    对于针对多个参数集执行的批量插入，组织 RETURNING 的结果，使返回的行与传入的参数集的顺序对应。这仅适用于支持方言的 executemany 执行，并且通常利用了 insertmanyvalues 特性。

    新版本 2.0.10 中新增。

    另请参阅

    将 RETURNING 行与参数集相关联 - 关于批量插入的 RETURNING 行排序的背景（核心级别讨论）

    将 RETURNING 记录与输入数据顺序相关联 - 与 ORM 批量插入语句 的使用示例（ORM 级别讨论）

另请参阅

`UpdateBase.return_defaults()` - 一种针对单行插入或更新的服务器端默认值和触发器的高效获取的替代方法。

INSERT…RETURNING - 在 SQLAlchemy 统一教程 中

```py
method from_select(names: Sequence[_DMLColumnArgument], select: Selectable, include_defaults: bool = True) → Self
```

返回一个新的 `Insert` 构造，该构造表示一个 `INSERT...FROM SELECT` 语句。

例如：

```py
sel = select(table1.c.a, table1.c.b).where(table1.c.c > 5)
ins = table2.insert().from_select(['a', 'b'], sel)
```

参数：

+   `names` – 字符串列名序列或表示目标列的 `Column` 对象。

+   `select` – 一个 `select()` 构造，`FromClause` 或其他可解析为 `FromClause` 的构造，比如 ORM `Query` 对象等。此 FROM 子句返回的列的顺序应与作为 `names` 参数发送的列的顺序相对应；虽然在传递给数据库之前不会检查这一点，但如果这些列列表不对应，数据库通常会引发异常。

+   `include_defaults` –

    如果为 True，则将在 INSERT 和 SELECT 语句中呈现在 `Column` 对象上指定的非服务器默认值和 SQL 表达式（如 Column INSERT/UPDATE Defaults 中记录的）中未在名称列表中另行指定的值，以便这些值也包含在要插入的数据中。

    注意

    使用 Python 可调用函数的 Python 端默认值仅在整个语句中被调用**一次**，而不是每行**一次**。

```py
method inline() → Self
```

使此 `Insert` 构造“内联”。

当设置时，不会尝试检索要在语句中提供的 SQL 生成的默认值；特别是，这允许 SQL 表达式在语句中“内联”呈现，而无需事先执行它们；对于支持“returning”的后端，这将关闭语句的“隐式返回”功能。

自版本 1.4 起：`Insert.inline` 参数现已被 `Insert.inline()` 方法取代。

```py
attribute select: Select[Any] | None = None
```

用于 INSERT .. FROM SELECT 的 SELECT 语���

```py
class sqlalchemy.sql.expression.Update
```

表示一个 Update 构造。

使用 `update()` 函数创建 `Update` 对象。

**成员**

returning(), where(), values(), inline(), ordered_values()

**类签名**

类 `sqlalchemy.sql.expression.Update` (`sqlalchemy.sql.expression.DMLWhereBase`, `sqlalchemy.sql.expression.ValuesBase`)

```py
method returning(*cols: _ColumnsClauseArgument[Any], sort_by_parameter_order: bool = False, **_UpdateBase__kw: Any) → UpdateBase
```

*继承自* `UpdateBase.returning()` *方法的* `UpdateBase`

向此语句添加 RETURNING 或等效子句。

例如：

```py
>>> stmt = (
...     table.update()
...     .where(table.c.data == "value")
...     .values(status="X")
...     .returning(table.c.server_flag, table.c.updated_timestamp)
... )
>>> print(stmt)
UPDATE  some_table  SET  status=:status
WHERE  some_table.data  =  :data_1
RETURNING  some_table.server_flag,  some_table.updated_timestamp 
```

这种方法可以被多次调用，以向要返回的表达式列表中添加新条目。

新版本 1.4.0b2 中添加：这种方法可以被多次调用，以向要返回的表达式列表中添加新条目。

给定的列表达式集合应源自 INSERT、UPDATE 或 DELETE 的目标表。虽然 `Column` 对象是典型的，但元素也可以是表达式：

```py
>>> stmt = table.insert().returning(
...     (table.c.first_name + " " + table.c.last_name).label("fullname")
... )
>>> print(stmt)
INSERT  INTO  some_table  (first_name,  last_name)
VALUES  (:first_name,  :last_name)
RETURNING  some_table.first_name  ||  :first_name_1  ||  some_table.last_name  AS  fullname 
```

在编译时，RETURNING 子句或数据库等效项将在语句中呈现。对于 INSERT 和 UPDATE，值是新插入/更新的值。对于 DELETE，值是已删除行的值。

在执行时，要返回的列的值通过结果集提供，并且可以使用 `CursorResult.fetchone()` 等进行迭代。对于原生不支持返回值的 DBAPI（即 cx_oracle 等），SQLAlchemy 将在结果级别近似此行为，以便提供合理数量的行为中性。

请注意，并非所有数据库/DBAPI 都支持 RETURNING。对于那些不支持的后端，编译和/或执行时会引发异常。对于支持的后端，跨后端的功能差异很大，包括对 executemany() 和其他返回多行的语句的限制。请阅读所使用数据库的文档注释，以确定 RETURNING 的可用性。

参数：

+   `*cols` – 要返回的一系列列、SQL 表达式或整个表实体。

+   `sort_by_parameter_order` –

    对于针对多个参数集执行的批量 INSERT，组织 RETURNING 的结果，使返回的行对应于传递的参数集的顺序。这仅适用于支持的方言的 executemany 执行，并通常利用 insertmanyvalues 功能。

    新版本 2.0.10 中添加。

    另请参阅

    将 RETURNING 行与参数集相关联 - 关于批量插入 RETURNING 行排序的背景（核心级讨论）

    将 RETURNING 记录与输入数据顺序相关联 - 与 ORM 批量插入语句 一起使用的示例（ORM 级讨论）

另请参阅

`UpdateBase.return_defaults()` - 针对单行 INSERT 或 UPDATE 的高效提取服务器端默认值和触发器的替代方法。

INSERT…RETURNING - 在 SQLAlchemy 统一教程中

```py
method where(*whereclause: _ColumnExpressionArgument[bool]) → Self
```

*继承自* `DMLWhereBase` *的* `DMLWhereBase.where()` *方法*

返回一个新的结构，其中包含添加到其 WHERE 子句的给定表达式，并通过 AND 连接到现有子句（如果有）。

`Update.where()` 和 `Delete.where()` 都支持多表形式，包括特定于数据库的 `UPDATE...FROM` 和 `DELETE..USING`。 对于不支持多表的后端，使用多表的后端不可知方法是利用相关子查询。 有关示例，请参见下面的链接教程部分。

另请参阅

相关更新

UPDATE..FROM

多表删除

```py
method values(*args: _DMLColumnKeyMapping[Any] | Sequence[Any], **kwargs: Any) → Self
```

*继承自* `ValuesBase` *的* `ValuesBase.values()` *方法*

指定 INSERT 语句的固定 VALUES 子句，或者 UPDATE 的 SET 子句。

请注意，`Insert` 和 `Update` 结构支持基于传递给 `Connection.execute()` 的参数对 VALUES 和/或 SET 子句进行每次执行时间格式化。 但是，`ValuesBase.values()` 方法可用于将特定的一组参数“固定”到语句中。

对 `ValuesBase.values()` 的多次调用将产生一个新的结构，每个结构的参数列表都被修改以包含发送的新参数。 在单个参数字典的典型情况下，新传递的键将替换上一个结构中的相同键。 在基于列表的“多个值”结构的情况下，每个新值列表都被扩展到现有值列表上。

参数：

+   `**kwargs` –

    代表要渲染到 VALUES 或 SET 子句中的值的字符串键的键值对：

    ```py
    users.insert().values(name="some name")

    users.update().where(users.c.id==5).values(name="some name")
    ```

+   `*args` –

    作为传递键/值参数的替代方案，可以将字典、元组或字典或元组的列表作为单个位置参数传递，以形成语句的 VALUES 或 SET 子句。所接受的形式因为这是否是一个`Insert`或`Update` 构造而异。

    对于`Insert` 或 `Update` 构造，也可以传递单个字典，其工作方式与 kwargs 形式相同：

    ```py
    users.insert().values({"name": "some name"})

    users.update().values({"name": "some new name"})
    ```

    也适用于任何形式，但更常见的是对于`Insert` 构造，也接受包含表中每列的条目的元组：

    ```py
    users.insert().values((5, "some name"))
    ```

    `Insert` 构造还支持传递字典或完整表元组的列表，在服务器上，这将呈现较不常见的 SQL 语法“多个值” - 此语法支持后端，例如 SQLite、PostgreSQL、MySQL，但不一定支持其他后端：

    ```py
    users.insert().values([
                        {"name": "some name"},
                        {"name": "some other name"},
                        {"name": "yet another name"},
                    ])
    ```

    上述形式将呈现类似于多个 VALUES 语句：

    ```py
    INSERT INTO users (name) VALUES
                    (:name_1),
                    (:name_2),
                    (:name_3)
    ```

    必须注意**传递多个值并不等同于使用传统的 executemany() 形式**。上述语法是一个**特殊**语法，通常不使用。要针对多行发出 INSERT 语句，正常方法是将多个值列表传递给`Connection.execute()` 方法，此方法受到所有数据库后端的支持，并且对于非常多的参数通常更有效率。

    > > 另请参阅
    > > 
    > > 发送多个参数 - 介绍传统 Core 方法的多参数集调用方式，用于 INSERT 和其他语句。
    > > 
    > UPDATE 构造还支持按特定顺序渲染 SET 参数。有关此功能，请参阅`Update.ordered_values()` 方法。
    > 
    > > 另请参阅
    > > 
    > > `Update.ordered_values()`

```py
method inline() → Self
```

将此`Update` 构造“内联”。

当设置时，通过`default`关键字在`Column`对象上存在的 SQL 默认值将被‘内联’编译到语句中，而不是预先执行。这意味着它们的值不会在从`CursorResult.last_updated_params()`返回的字典中可用。

在 1.4 版本中更改：`update.inline`参数现在被`Update.inline()`方法取代。

```py
method ordered_values(*args: Tuple[_DMLColumnArgument, Any]) → Self
```

使用显式参数顺序指定此 UPDATE 语句的 VALUES 子句，该顺序将在生成的 UPDATE 语句的 SET 子句中保持不变。

例如：

```py
stmt = table.update().ordered_values(
    ("name", "ed"), ("ident", "foo")
)
```

另请参阅

参数顺序更新 - `Update.ordered_values()` 方法的完整示例。

在 1.4 版本中更改：`Update.ordered_values()` 方法取代了`update.preserve_parameter_order`参数，该参数将在 SQLAlchemy 2.0 中被移除。

```py
class sqlalchemy.sql.expression.UpdateBase
```

为`INSERT`、`UPDATE`和`DELETE`语句提供基础。

**成员**

entity\\_description, exported\\_columns, params(), return\\_defaults(), returning(), returning\\_column\\_descriptions, with\\_dialect\\_options(), with\\_hint()

**类签名**

类`sqlalchemy.sql.expression.UpdateBase` (`sqlalchemy.sql.roles.DMLRole`, `sqlalchemy.sql.expression.HasCTE`, `sqlalchemy.sql.expression.HasCompileState`, `sqlalchemy.sql.base.DialectKWArgs`, `sqlalchemy.sql.expression.HasPrefixes`, `sqlalchemy.sql.expression.Generative`, `sqlalchemy.sql.expression.ExecutableReturnsRows`, `sqlalchemy.sql.expression.ClauseElement`)

```py
attribute entity_description
```

返回针对此 DML 构造操作的表和/或实体的启用插件描述。

当使用 ORM 时，此属性通常很有用，因为它返回了一个扩展的结构，其中包含有关映射实体的信息。有关更多背景信息，请参阅 从 ORM 启用的 SELECT 和 DML 语句中检查实体和列。

对于核心语句，此访问器返回的结构派生自 `UpdateBase.table` 属性，并引用正在插入、更新或删除的`Table`：

```py
>>> stmt = insert(user_table)
>>> stmt.entity_description
{
 "name": "user_table",
 "table": Table("user_table", ...)
}
```

新功能，版本 1.4.33。

另请参阅

`UpdateBase.returning_column_descriptions`

`Select.column_descriptions` - `select()` 构造的实体信息

从 ORM 启用的 SELECT 和 DML 语句中检查实体和列 - ORM 背景

```py
attribute exported_columns
```

返回该语句的 RETURNING 列作为列集合。

新功能，版本 1.4。

```py
method params(*arg: Any, **kw: Any) → NoReturn
```

设置语句的参数。

此方法在基类上引发 `NotImplementedError`，并由`ValuesBase`覆盖以提供 UPDATE 和 INSERT 的 SET/VALUES 子句。

```py
method return_defaults(*cols: _DMLColumnArgument, supplemental_cols: Iterable[_DMLColumnArgument] | None = None, sort_by_parameter_order: bool = False) → Self
```

利用 RETURNING 子句以获取服务器端表达式和默认值，仅支持后端。

深度炼金术

`UpdateBase.return_defaults()`方法被 ORM 用于其内部工作中，用于获取新生成的主键和服务器默认值，特别是为了提供`Mapper.eager_defaults` ORM 特性的底层实现，以及允许在批量 ORM 插入中支持 RETURNING。其行为相当特殊，实际上不适合一般使用。最终用户应坚持使用`UpdateBase.returning()`来为他们的 INSERT、UPDATE 和 DELETE 语句添加 RETURNING 子句。

通常，执行单行 INSERT 语句时，会自动填充`CursorResult.inserted_primary_key`属性，该属性存储了刚刚插入的行的主键，以`Row`对象的形式，列名作为命名元组键（并且`Row._mapping`视图也完全填充）。使用的方言选择用于填充这些数据的策略；如果是使用服务器端默认值和/或 SQL 表达式生成的，则通常使用特定于方言的方法（如`cursor.lastrowid`或`RETURNING`）来获取新的主键值。

然而，在执行语句之前通过调用`UpdateBase.return_defaults()`修改语句时，只有支持 RETURNING 并且将`Table.implicit_returning`参数维持在其默认值`True`的后端以及维护`Table`对象的其他行为才会发生。在这些情况下，当从语句的执行返回`CursorResult`时，不仅`CursorResult.inserted_primary_key`将像往常一样被填充，`CursorResult.returned_defaults`属性还将被填充为一个名为`Row`的命名元组，代表该单行的完整服务器生成值范围，包括任何指定`Column.server_default`或使用 SQL 表达式的`Column.default`的列的值。

当使用 insertmanyvalues 调用多行的 INSERT 语句时，`UpdateBase.return_defaults()`修饰符将会影响`CursorResult.inserted_primary_key_rows`和`CursorResult.returned_defaults_rows`属性被完全填充为代表每行新插入的主键值以及新插入的服务器生成值的`Row`对象列表。`CursorResult.inserted_primary_key`和`CursorResult.returned_defaults`属性也将继续被填充为这两个集合的第一行。

如果后端不支持 RETURNING 或者正在使用的 `Table` 已经禁用了 `Table.implicit_returning`，那么就不会添加 RETURNING 子句，也不会获取任何额外的数据，但是 INSERT、UPDATE 或 DELETE 语句会正常执行。

例如：

```py
stmt = table.insert().values(data='newdata').return_defaults()

result = connection.execute(stmt)

server_created_at = result.returned_defaults['created_at']
```

当用于 UPDATE 语句时，`UpdateBase.return_defaults()` 会查找包含 `Column.onupdate` 或 `Column.server_onupdate` 参数的列，当构造默认情况下将包含在 RETURNING 子句中的列时（如果未明确指定列）。当用于 DELETE 语句时，默认情况下不会包含任何列在 RETURNING 中，而是必须明确指定，因为在 DELETE 语句执行时通常不会更改值的列。

从版本 2.0 开始：`UpdateBase.return_defaults()` 也支持 DELETE 语句，并且已从 `ValuesBase` 移动到 `UpdateBase`。

`UpdateBase.return_defaults()` 方法与 `UpdateBase.returning()` 方法互斥，在同一条语句上同时使用两者会在 SQL 编译过程中引发错误。因此，INSERT、UPDATE 或 DELETE 语句的 RETURNING 子句一次只能由其中一个方法控制。

`UpdateBase.return_defaults()` 方法与 `UpdateBase.returning()` 在以下方面不同：

1.  `UpdateBase.return_defaults()`方法导致`CursorResult.returned_defaults`集合被填充为 RETURNING 结果的第一行。当使用`UpdateBase.returning()`时，此属性不会被填充。

1.  `UpdateBase.return_defaults()`与用于获取自动生成的主键值并将其填充到`CursorResult.inserted_primary_key`属性的现有逻辑兼容。相比之下，使用`UpdateBase.returning()`将导致`CursorResult.inserted_primary_key`属性保持未填充状态。

1.  `UpdateBase.return_defaults()`可以针对任何后端调用。不支持 RETURNING 的后端将跳过该功能的使用，而不是引发异常，*除非*传递了`supplemental_cols`。对于不支持 RETURNING 或目标`Table`设置`Table.implicit_returning`为`False`的后端，`CursorResult.returned_defaults`的返回值将为`None`。

1.  使用`executemany()`调用的 INSERT 语句在后端数据库驱动程序支持 insertmanyvalues 功能的情况下得到支持，这个功能现在大多数包含在 SQLAlchemy 中的后端都支持。当使用`executemany`时，`CursorResult.returned_defaults_rows`和`CursorResult.inserted_primary_key_rows`访问器将返回插入的默认值和主键。

    1.4 版本中新增：添加了`CursorResult.returned_defaults_rows`和`CursorResult.inserted_primary_key_rows`访问器。在 2.0 版本中，用于获取和填充这些属性的底层实现被泛化以支持大多数后端，而在 1.4 版本中，它们仅由`psycopg2`驱动程序支持。

参数：

+   `cols` – 可选的列键名列表或`Column`，作为过滤器用于将要获取的列。

+   `supplemental_cols` –

    可选的 RETURNING 表达式列表，与`UpdateBase.returning()`方法传递的形式相同。当存在时，额外的列将包含在 RETURNING 子句中，并且在返回时，`CursorResult`对象将被“倒带”，以便像`CursorResult.all()`这样的方法将以大部分方式返回新行，就好像语句直接使用了`UpdateBase.returning()`。但是，与直接使用`UpdateBase.returning()`时不同，列的顺序是未定义的，因此只能使用名称或`Row._mapping`键来定位它们；它们不能可靠地以位置为目标。

    2.0 版本中新增。

+   `sort_by_parameter_order` –

    对于正在针对多个参数集执行的批量插入，组织返回的 RETURNING 结果，使返回的行与传递的参数集的顺序相对应。这仅适用于支持方言的 executemany 执行，并且通常利用 insertmanyvalues 功能。

    2.0.10 版本中新增。

    另请参阅

    将返回的行与参数集相关联 - 关于批量插入的返回行排序的背景知识

另请参阅

`UpdateBase.returning()`

`CursorResult.returned_defaults`

`CursorResult.returned_defaults_rows`

`CursorResult.inserted_primary_key`

`CursorResult.inserted_primary_key_rows`

```py
method returning(*cols: _ColumnsClauseArgument[Any], sort_by_parameter_order: bool = False, **_UpdateBase__kw: Any) → UpdateBase
```

向该语句添加一个 RETURNING 或等效子句。

例如：

```py
>>> stmt = (
...     table.update()
...     .where(table.c.data == "value")
...     .values(status="X")
...     .returning(table.c.server_flag, table.c.updated_timestamp)
... )
>>> print(stmt)
UPDATE  some_table  SET  status=:status
WHERE  some_table.data  =  :data_1
RETURNING  some_table.server_flag,  some_table.updated_timestamp 
```

该方法可以多次调用以向要返回的表达式列表添加新条目。

从版本 1.4.0b2 开始新添加：该方法可以多次调用以向要返回的表达式列表添加新条目。

给定的列表达式集合应该来源于作为 INSERT、UPDATE 或 DELETE 目标的表。虽然 `Column` 对象很典型，但元素也可以是表达式：

```py
>>> stmt = table.insert().returning(
...     (table.c.first_name + " " + table.c.last_name).label("fullname")
... )
>>> print(stmt)
INSERT  INTO  some_table  (first_name,  last_name)
VALUES  (:first_name,  :last_name)
RETURNING  some_table.first_name  ||  :first_name_1  ||  some_table.last_name  AS  fullname 
```

在编译时，RETURNING 子句或数据库等效项将在语句内呈现。对于 INSERT 和 UPDATE，这些值是新插入/更新的值。对于 DELETE，这些值是被删除的行的值。

在执行时，要返回的列的值通过结果集可用，并且可以使用 `CursorResult.fetchone()` 等进行迭代。对于不本地支持返回值的 DBAPI（即 cx_oracle），SQLAlchemy 将在结果级别上近似此行为，以便提供合理数量的行为中立性。

请注意，并非所有数据库/DBAPI 都支持 RETURNING。对于没有支持的后端，在编译和/或执行时会引发异常。对于支持的后端，跨后端的功能差异很大，包括对 executemany() 和其他返回多行的语句的限制。请阅读正在使用的数据库的文档说明，以确定 RETURNING 的可用性。

参数：

+   `*cols` – 一系列列、SQL 表达式或整个表实体，要返回。

+   `sort_by_parameter_order` –

    对于正在执行针对多个参数集的批量 INSERT，组织 RETURNING 的结果，以便返回的行与传入的参数集的顺序对应。这仅适用于对支持的方言执行的 executemany 操作，通常利用 insertmanyvalues 功能。

    从版本 2.0.10 开始新添加。

    另请参阅

    将 RETURNING 行相关联到参数集 - 对批量 INSERT 的 RETURNING 行排序的背景信息（核心级别讨论）

    将 RETURNING 记录与输入数据顺序相关联 - 与 ORM 大批量 INSERT 语句 使用示例（ORM 级别讨论）

另请参阅

`UpdateBase.return_defaults()` - 针对单行 INSERT 或 UPDATE，针对高效获取服务器端默认值和触发器的替代方法。

INSERT…RETURNING - 在 SQLAlchemy 统一教程 中

```py
attribute returning_column_descriptions
```

返回此 DML 构造体返回的列的 插件启用 描述，换句话说，作为 `UpdateBase.returning()` 的一部分建立的表达式。

当使用 ORM 时，此属性通常很有用，因为返回的扩展结构包含有关映射实体的信息。该部分 从 ORM 启用的 SELECT 和 DML 语句中检查实体和列 包含更多背景信息。

对于核心语句，此访问器返回的结构源自与 `UpdateBase.exported_columns` 访问器返回的相同对象：

```py
>>> stmt = insert(user_table).returning(user_table.c.id, user_table.c.name)
>>> stmt.entity_description
[
 {
 "name": "id",
 "type": Integer,
 "expr": Column("id", Integer(), table=<user>, ...)
 },
 {
 "name": "name",
 "type": String(),
 "expr": Column("name", String(), table=<user>, ...)
 },
]
```

版本 1.4.33 中的新功能。

另请参阅

`UpdateBase.entity_description`

`Select.column_descriptions` - 用于 `select()` 构造的实体信息

从 ORM 启用的 SELECT 和 DML 语句中检查实体和列 - ORM 背景

```py
method with_dialect_options(**opt: Any) → Self
```

将方言选项添加到此 INSERT/UPDATE/DELETE 对象中。

例如：

```py
upd = table.update().dialect_options(mysql_limit=10)
```

```py
method with_hint(text: str, selectable: _DMLTableArgument | None = None, dialect_name: str = '*') → Self
```

将单个表的表提示添加到此 INSERT/UPDATE/DELETE 语句中。

注意

`UpdateBase.with_hint()` 目前仅适用于 Microsoft SQL Server。对于 MySQL INSERT/UPDATE/DELETE 提示，请使用 `UpdateBase.prefix_with()`。

提示文本在使用的数据库后端的适当位置呈现，与此语句的主题 `Table` 相对应，或者可选地，相对于传递为 `selectable` 参数的给定 `Table`。

`dialect_name`选项将限制特定后端的特定提示的呈现。例如，要添加一个仅在 SQL Server 中生效的提示：

```py
mytable.insert().with_hint("WITH (PAGLOCK)", dialect_name="mssql")
```

参数：

+   `text` – 提示的文本。

+   `selectable` – 可选的`Table`，指定在 UPDATE 或 DELETE 中作为提示主题的 FROM 子句的元素 - 仅适用于某些后端。

+   `dialect_name` – 默认为`*`，如果指定为特定方言的名称，将仅在使用该方言时应用这些提示。

```py
class sqlalchemy.sql.expression.ValuesBase
```

为 INSERT 和 UPDATE 构造提供对`ValuesBase.values()`的支持。

**成员**

select, values()

**类签名**

类`sqlalchemy.sql.expression.ValuesBase` (`sqlalchemy.sql.expression.UpdateBase`)

```py
attribute select: Select[Any] | None = None
```

用于 INSERT .. FROM SELECT 的 SELECT 语句

```py
method values(*args: _DMLColumnKeyMapping[Any] | Sequence[Any], **kwargs: Any) → Self
```

为 INSERT 语句指定一个固定的 VALUES 子句，或者为 UPDATE 指定 SET 子句。

请注意，`Insert`和`Update`构造支持基于传递给`Connection.execute()`的参数对 VALUES 和/或 SET 子句进行每次执行时的格式化。但是，`ValuesBase.values()`方法可用于将特定一组参数“固定”到语句中。

对`ValuesBase.values()`的多次调用将产生一个新的构造，每个构造的参数列表都会修改以包含发送的新参数。在单个参数字典的典型情况下，新传递的键将替换先前构造中的相同键。在基于列表的“多个值”构造的情况下，每个新值列表都会扩展到现有值列表上。

参数：

+   `**kwargs` –

    表示要映射到 VALUES 或 SET 子句中的值的`Column`的字符串键值对：

    ```py
    users.insert().values(name="some name")

    users.update().where(users.c.id==5).values(name="some name")
    ```

+   `*args` –

    作为传递键/值参数的替代方案，可以将字典、元组或字典列表或元组作为单个位置参数传递，以形成语句的 VALUES 或 SET 子句。接受的形式因是`Insert`还是`Update`构造而异。

    对于`Insert`或`Update`构造，可以传递一个单独的字典，其工作方式与 kwargs 形式相同：

    ```py
    users.insert().values({"name": "some name"})

    users.update().values({"name": "some new name"})
    ```

    对于任何形式，但更典型的是`Insert`构造，也可以接受包含表中每一列条目的元组：

    ```py
    users.insert().values((5, "some name"))
    ```

    `Insert`构造还支持传递一个字典列表或完整表元组，服务器上将呈现较不常见的 SQL 语法“多个值” - 此语法在后端（如 SQLite、PostgreSQL、MySQL）上受支持，但不一定适用于其他后端：

    ```py
    users.insert().values([
                        {"name": "some name"},
                        {"name": "some other name"},
                        {"name": "yet another name"},
                    ])
    ```

    上述形式将呈现类似于多个 VALUES 语句：

    ```py
    INSERT INTO users (name) VALUES
                    (:name_1),
                    (:name_2),
                    (:name_3)
    ```

    需要注意的是**传递多个值并不等同于使用传统的 executemany()形式**。上述语法是一种**特殊**的语法，通常不常用。要对多行发出 INSERT 语句，正常方法是将多个值列表传递给`Connection.execute()`方法，这种方法受到所有数据库后端的支持，并且对于非常大量的参数通常更有效率。

    > > 另请参阅
    > > 
    > > 发送多个参数 - 介绍了用于 INSERT 和其他语句的传统 Core 方法的多参数集调用。
    > > 
    > UPDATE 构造还支持按特定顺序呈现 SET 参数。有关此功能，请参考`Update.ordered_values()`方法。
    > 
    > > 另请参阅
    > > 
    > > `Update.ordered_values()`

## DML 基础构造

顶层“INSERT”、“UPDATE”、“DELETE”构造函数。

| 对象名称 | 描述 |
| --- | --- |
| delete(table) | 构造`Delete`对象。 |
| insert(table) | 构造一个`Insert`对象。 |
| 更新(table) | 构造一个`Update`对象。 |

```py
function sqlalchemy.sql.expression.delete(table: _DMLTableArgument) → Delete
```

构造`Delete`对象。

例如：

```py
from sqlalchemy import delete

stmt = (
    delete(user_table).
    where(user_table.c.id == 5)
)
```

类似功能也可通过`TableClause.delete()`方法在`Table`上获得。

参数：

**table** – 要从中删除行的表。

另请参阅

使用 UPDATE 和 DELETE 语句 - 在 SQLAlchemy 统一教程中

```py
function sqlalchemy.sql.expression.insert(table: _DMLTableArgument) → Insert
```

构造一个`Insert`对象。

例如：

```py
from sqlalchemy import insert

stmt = (
    insert(user_table).
    values(name='username', fullname='Full Username')
)
```

类似功能也可通过`TableClause.insert()`方法在`Table`上获得。

另请参阅

使用 INSERT 语句 - 在 SQLAlchemy 统一教程中

参数：

+   `table` – `TableClause`，即插入主题。

+   `values` – 要插入的值集合；请参阅`Insert.values()`以获取此处允许的格式描述。可以完全省略；`Insert`构造还将根据传递给`Connection.execute()`的参数，在执行时动态渲染 VALUES 子句。

+   `inline` – 如果为 True，则不会尝试检索生成的 SQL 默认值，以便在语句中提供；特别地，这允许 SQL 表达式在语句中“内联”渲染，而无需事先执行它们；对于支持“返回”的后端，这会关闭语句的“隐式返回”功能。

如果同时存在`insert.values`和编译时绑定参数，则编译时绑定参数将覆盖在`insert.values`中指定的信息，按键分别覆盖。

`Insert.values` 中的键可以是 `Column` 对象或它们的字符串标识符。 每个键可能引用以下之一：

+   一个字面数据值（即字符串、数字等）；

+   一个 Column 对象；

+   一个 SELECT 语句。

如果指定了一个 `SELECT` 语句，该语句引用了此 `INSERT` 语句的表，那么该语句将与 `INSERT` 语句相关联。

另请参阅

使用 INSERT 语句 - 在 SQLAlchemy 统一教程 中

```py
function sqlalchemy.sql.expression.update(table: _DMLTableArgument) → Update
```

构造一个 `Update` 对象。

例如：

```py
from sqlalchemy import update

stmt = (
    update(user_table).
    where(user_table.c.id == 5).
    values(name='user #5')
)
```

类似功能也可通过 `TableClause.update()` 方法在 `Table` 上获得。

参数：

**table** – 代表要更新的数据库表的 `Table` 对象。

另请参阅

使用 UPDATE 和 DELETE 语句 - 在 SQLAlchemy 统一教程 中

## DML 类文档构造函数

DML Foundational Constructors 中列出的构造函数的类文档。

| 对象名称 | 描述 |
| --- | --- |
| Delete | 代表一个 DELETE 结构。 |
| Insert | 代表一个 INSERT 结构。 |
| Update | 代表一个 Update 结构。 |
| UpdateBase | 形成 `INSERT`、`UPDATE` 和 `DELETE` 语句的基础。 |
| ValuesBase | 为 INSERT 和 UPDATE 结构提供 `ValuesBase.values()` 的支持。 |

```py
class sqlalchemy.sql.expression.Delete
```

代表一个 DELETE 结构。

`Delete` 对象是使用 `delete()` 函数创建的。

**成员**

where(), returning()

**类签名**

类`sqlalchemy.sql.expression.Delete`（`sqlalchemy.sql.expression.DMLWhereBase`，`sqlalchemy.sql.expression.UpdateBase`）

```py
method where(*whereclause: _ColumnExpressionArgument[bool]) → Self
```

*继承自* `DMLWhereBase` *的* `DMLWhereBase.where()` *方法*

返回一个新的构造，其中给定的表达式被添加到其 WHERE 子句中，并通过 AND 连接到现有子句（如果有）。

`Update.where()` 和 `Delete.where()` 都支持多表形式，包括特定于数据库的`UPDATE...FROM`以及`DELETE..USING`。对于不支持多表的后端，使用多表的跨后端方法是利用相关子查询。请参阅下面链接的教程部分以获取示例。

另请参见

相关更新

UPDATE..FROM

多表删除

```py
method returning(*cols: _ColumnsClauseArgument[Any], sort_by_parameter_order: bool = False, **_UpdateBase__kw: Any) → UpdateBase
```

*继承自* `UpdateBase.returning()` *方法的* `UpdateBase`

为该语句添加一个 RETURNING 或等效子句。

例如：

```py
>>> stmt = (
...     table.update()
...     .where(table.c.data == "value")
...     .values(status="X")
...     .returning(table.c.server_flag, table.c.updated_timestamp)
... )
>>> print(stmt)
UPDATE  some_table  SET  status=:status
WHERE  some_table.data  =  :data_1
RETURNING  some_table.server_flag,  some_table.updated_timestamp 
```

可以多次调用该方法以向要返回的表达式列表中添加新条目。

从版本 1.4.0b2 开始：可以多次调用该方法以向要返回的表达式列表中添加新条目。

给定的列表达式集合应源自 INSERT、UPDATE 或 DELETE 的目标表。虽然`Column`对象是典型的，但元素也可以是表达式：

```py
>>> stmt = table.insert().returning(
...     (table.c.first_name + " " + table.c.last_name).label("fullname")
... )
>>> print(stmt)
INSERT  INTO  some_table  (first_name,  last_name)
VALUES  (:first_name,  :last_name)
RETURNING  some_table.first_name  ||  :first_name_1  ||  some_table.last_name  AS  fullname 
```

在编译时，将在语句中呈现 RETURNING 子句或数据库等效。对于 INSERT 和 UPDATE，值是新插入/更新的值。对于 DELETE，值是被删除的行的值。

在执行时，要返回的列的值通过结果集提供，并可以使用`CursorResult.fetchone()`等进行迭代。对于不原生支持返回值的 DBAPI（即 cx_oracle），SQLAlchemy 将在结果级别近似此行为，以便提供合理数量的行为中立性。

请注意，并非所有数据库/DBAPI 都支持 RETURNING。对于那些不支持的后端，在编译和/或执行时会引发异常。对于支持它的后端，跨后端的功能差异很大，包括对 executemany()和返回多行的其他语句的限制。请阅读所使用数据库的文档注释，以确定 RETURNING 的可用性。

参数：

+   `*cols` – 要返回的一系列列、SQL 表达式或整个表实体。

+   `sort_by_parameter_order` –

    对于针对多个参数集执行的批量插入，组织 RETURNING 的结果，使返回的行与传入的参数集的顺序对应。这仅适用于支持方言的 executemany 执行，并通常利用 insertmanyvalues 功能。

    版本 2.0.10 中的新功能。

    另请参阅

    将 RETURNING 行与参数集相关联 - 关于批量插入 RETURNING 行排序的背景（核心级别讨论）

    将 RETURNING 记录与输入数据顺序相关联 - 与 ORM 批量 INSERT 语句 的使用示例（ORM 级别讨论）

另请参阅

`UpdateBase.return_defaults()` - 一种针对单行 INSERT 或 UPDATE 的有效获取服务器端默认值和触发器的替代方法。

INSERT…RETURNING - 在 SQLAlchemy 统一教程 中

```py
class sqlalchemy.sql.expression.Insert
```

表示一个 INSERT 构造。

`Insert` 对象是使用 `insert()` 函数创建的。

**成员**

values(), returning(), from_select(), inline(), select

**类签名**

类 `sqlalchemy.sql.expression.Insert`（`sqlalchemy.sql.expression.ValuesBase`）

```py
method values(*args: _DMLColumnKeyMapping[Any] | Sequence[Any], **kwargs: Any) → Self
```

*继承自* `ValuesBase.values()` *方法的* `ValuesBase`

为 INSERT 语句指定固定的 VALUES 子句，或为 UPDATE 指定 SET 子句。

请注意，`Insert`和`Update`构造支持基于传递给`Connection.execute()`的参数对 VALUES 和/或 SET 子句进行执行时格式化。但是，`ValuesBase.values()`方法可用于将特定一组参数“固定”到语句中。

多次调用`ValuesBase.values()`将生成一个新构造，每个构造的参数列表都会修改以包含发送的新参数。在典型情况下，单个参数字典中的新传递键将替换先前构造中的相同键。在基于列表的“多个值”构造的情况下，每个新值列表都会扩展到现有值列表上。

参数：

+   `**kwargs` –

    键值对表示`Column`的字符串键映射到要呈现到 VALUES 或 SET 子句中的值：

    ```py
    users.insert().values(name="some name")

    users.update().where(users.c.id==5).values(name="some name")
    ```

+   `*args` –

    作为传递键/值参数的替代方案，可以将字典、元组或字典或元组的列表作为单个位置参数传递，以形成语句的 VALUES 或 SET 子句。接受的形式因此是一个`Insert`还是一个`Update`构造而异。

    对于`Insert`或`Update`构造，可以传递一个字典，其工作方式与 kwargs 形式相同：

    ```py
    users.insert().values({"name": "some name"})

    users.update().values({"name": "some new name"})
    ```

    对于任何形式，但更典型地用于`Insert`构造，也接受包含表中每列条目的元组：

    ```py
    users.insert().values((5, "some name"))
    ```

    `Insert`构造还支持传递字典或完整表元组的列表，这在服务器上将呈现较少见的 SQL 语法“多个值” - 此语法在后端（如 SQLite、PostgreSQL、MySQL）上受支持，但不一定在其他后端上受支持：

    ```py
    users.insert().values([
                        {"name": "some name"},
                        {"name": "some other name"},
                        {"name": "yet another name"},
                    ])
    ```

    上述形式将呈现类似于多个 VALUES 语句：

    ```py
    INSERT INTO users (name) VALUES
                    (:name_1),
                    (:name_2),
                    (:name_3)
    ```

    必须注意**传递多个值并不等同于使用传统的 executemany()形式**。上述语法是一种**特殊**的语法，通常不常用。要针对多行发出 INSERT 语句，正常方法是将多个值列表传递给`Connection.execute()`方法，该方法受到所有数据库后端支持，并且对于非常大量的参数通常更有效率。

    > > 另请参阅
    > > 
    > > 发送多个参数 - 介绍传统核心方法的多参数集调用，用于 INSERT 和其他语句。
    > > 
    > UPDATE 结构还支持按特定顺序呈现 SET 参数。有关此功能，请参考`Update.ordered_values()`方法。
    > 
    > > 另请参阅
    > > 
    > > `Update.ordered_values()`

```py
method returning(*cols: _ColumnsClauseArgument[Any], sort_by_parameter_order: bool = False, **_UpdateBase__kw: Any) → UpdateBase
```

*继承自* `UpdateBase.returning()` *方法的* `UpdateBase`

在此语句中添加一个 RETURNING 或等效子句。

例如：

```py
>>> stmt = (
...     table.update()
...     .where(table.c.data == "value")
...     .values(status="X")
...     .returning(table.c.server_flag, table.c.updated_timestamp)
... )
>>> print(stmt)
UPDATE  some_table  SET  status=:status
WHERE  some_table.data  =  :data_1
RETURNING  some_table.server_flag,  some_table.updated_timestamp 
```

可以多次调用该方法以向要返回的表达式列表添加新条目。

从版本 1.4.0b2 中新增：可以多次调用该方法以向要返回的表达式列表添加新条目。

给定的列表达式集合应源自 INSERT、UPDATE 或 DELETE 的目标表。虽然`Column`对象是典型的，但元素也可以是表达式：

```py
>>> stmt = table.insert().returning(
...     (table.c.first_name + " " + table.c.last_name).label("fullname")
... )
>>> print(stmt)
INSERT  INTO  some_table  (first_name,  last_name)
VALUES  (:first_name,  :last_name)
RETURNING  some_table.first_name  ||  :first_name_1  ||  some_table.last_name  AS  fullname 
```

在编译时，将在语句中呈现一个 RETURNING 子句，或者数据库等效项。对于 INSERT 和 UPDATE，这些值是新插入/更新的值。对于 DELETE，这些值是被删除的行的值。

在执行时，要返回的列的值通过结果集提供，并可以使用`CursorResult.fetchone()`等进行迭代。对于不本地支持返回值的 DBAPI（即 cx_oracle），SQLAlchemy 将在结果级别近似此行为，以便提供合理数量的行为中立性。

请注意，并非所有数据库/DBAPI 都支持 RETURNING。对于那些不支持的后端，在编译和/或执行时会引发异常。对于那些支持它的后端，跨后端的功能差异很大，包括对 executemany()和其他返回多行的语句的限制。请阅读正在使用的数据库的文档说明，以确定 RETURNING 的可用性。

参数：

+   `*cols` – 一系列列、SQL 表达式或整个表实体要返回。

+   `sort_by_parameter_order` –

    对于正在针对多个参数集执行的批量 INSERT，组织 RETURNING 的结果，使返回的行与传入的参数集的顺序对应。这仅适用于支持方言的 executemany 执行，并通常利用 insertmanyvalues 功能。

    2.0.10 版中的新功能。

    另请参阅

    将 RETURNING 行与参数集相关联 - 关于批量 INSERT 的 RETURNING 行排序的背景（核心级别讨论）

    将 RETURNING 记录与输入数据顺序相关联 - 与 ORM 批量 INSERT 语句一起使用的示例（ORM 级别讨论）

另请参阅

`UpdateBase.return_defaults()` - 一种针对高效获取服务器端默认值和触发器的单行 INSERT 或 UPDATE 的替代方法。

INSERT…RETURNING - 在 SQLAlchemy 统一教程中

```py
method from_select(names: Sequence[_DMLColumnArgument], select: Selectable, include_defaults: bool = True) → Self
```

返回一个新的`Insert`构造，表示一个`INSERT...FROM SELECT`语句。

例如：

```py
sel = select(table1.c.a, table1.c.b).where(table1.c.c > 5)
ins = table2.insert().from_select(['a', 'b'], sel)
```

参数：

+   `names` – 一系列字符串列名或`Column`对象，表示目标列。

+   `select` – 一个`select()`构造，`FromClause`或其他解析为`FromClause`的构造，例如 ORM `Query`对象等。从此 FROM 子句返回的列的顺序应与作为`names`参数发送的列的顺序相对应；虽然在传递给数据库之前不会检查这一点，但如果这些列列表不对应，数据库通常会引发异常。

+   `include_defaults` –

    如果为 True，则将渲染到 INSERT 和 SELECT 语句中的非服务器默认值和在 `Column` 对象上指定的 SQL 表达式（如 Column INSERT/UPDATE Defaults 中所记录）未在名称列表中另行指定，以便这些值也包含在要插入的数据中。

    注意

    使用 Python 可调用函数的 Python 端默认值将仅在整个语句中被调用 **一次**，而不是每行一次。

```py
method inline() → Self
```

将此 `Insert` 构造“内联”。

当设置时，将不会尝试检索在语句中提供的 SQL 生成的默认值；特别是，这允许 SQL 表达式在语句中“内联”渲染，无需事先对它们进行预执行；对于支持“returning”的后端，这将关闭语句的“隐式返回”功能。

在版本 1.4 中进行了更改：`Insert.inline` 参数现已被 `Insert.inline()` 方法取代。

```py
attribute select: Select[Any] | None = None
```

INSERT .. FROM SELECT 的 SELECT 语句

```py
class sqlalchemy.sql.expression.Update
```

表示一个 Update 构造。

使用 `update()` 函数创建 `Update` 对象。

**成员**

returning(), where(), values(), inline(), ordered_values()

**类签名**

类 `sqlalchemy.sql.expression.Update`（`sqlalchemy.sql.expression.DMLWhereBase`，`sqlalchemy.sql.expression.ValuesBase`)

```py
method returning(*cols: _ColumnsClauseArgument[Any], sort_by_parameter_order: bool = False, **_UpdateBase__kw: Any) → UpdateBase
```

*继承自* `UpdateBase.returning()` *方法的* `UpdateBase`

向该语句添加一个 RETURNING 或等效子句。

例如：

```py
>>> stmt = (
...     table.update()
...     .where(table.c.data == "value")
...     .values(status="X")
...     .returning(table.c.server_flag, table.c.updated_timestamp)
... )
>>> print(stmt)
UPDATE  some_table  SET  status=:status
WHERE  some_table.data  =  :data_1
RETURNING  some_table.server_flag,  some_table.updated_timestamp 
```

此方法可以多次调用以向要返回的表达式列表添加新条目。

新版本 1.4.0b2 中：此方法可以多次调用以向要返回的表达式列表添加新条目。

给定的列表达式集合应源自是 INSERT、UPDATE 或 DELETE 目标的表。虽然 `Column` 对象是典型的，但元素也可以是表达式：

```py
>>> stmt = table.insert().returning(
...     (table.c.first_name + " " + table.c.last_name).label("fullname")
... )
>>> print(stmt)
INSERT  INTO  some_table  (first_name,  last_name)
VALUES  (:first_name,  :last_name)
RETURNING  some_table.first_name  ||  :first_name_1  ||  some_table.last_name  AS  fullname 
```

编译时，RETURNING 子句或数据库等效项将包含在语句中。对于 INSERT 和 UPDATE，这些值是新插入/更新的值。对于 DELETE，这些值是被删除行的值。

在执行时，要返回的列的值通过结果集提供，并可以使用 `CursorResult.fetchone()` 和类似方法进行迭代。对于不原生支持返回值的 DBAPI（即 cx_oracle 等），SQLAlchemy 将在结果级别近似此行为，以提供合理数量的行为中立性。

请注意，并非所有数据库/DBAPI 都支持 RETURNING。对于那些没有支持的后端，在编译和/或执行时会引发异常。对于支持的后端，跨后端的功能差异很大，包括对 executemany() 和其他返回多行的语句的限制。请阅读使用中的数据库的文档注释，以确定 RETURNING 的可用性。

参数：

+   `*cols` – 一系列要返回的列、SQL 表达式或整个表实体。

+   `sort_by_parameter_order` –

    对于正在执行多个参数集的批量 INSERT，组织 RETURNING 的结果，使返回的行与传入的参数集的顺序对应。这仅适用于支持的方言的 executemany 执行，并且通常利用 insertmanyvalues 功能。

    在 2.0.10 版本中新增。

    另请参阅

    将 RETURNING 行与参数集相关联 - 关于批量插入 RETURNING 行排序的背景（核心层讨论）

    将 RETURNING 记录与输入数据顺序相关联 - 在 ORM 批量 INSERT 语句 中的使用示例（ORM 层讨论）

另请参阅

`UpdateBase.return_defaults()` - 针对高效获取单行 INSERT 或 UPDATE 的服务器端默认值和触发器的替代方法。

INSERT…RETURNING - 在 SQLAlchemy 统一教程 中

```py
method where(*whereclause: _ColumnExpressionArgument[bool]) → Self
```

*从* `DMLWhereBase.where()` *方法的* `DMLWhereBase` *继承*

返回一个新的构造，其中包含要添加到其 WHERE 子句中的给定表达式，如果有的话，通过 AND 连接到现有子句。

`Update.where()`和`Delete.where()`都支持多表形式，包括数据库特定的`UPDATE...FROM`以及`DELETE..USING`。 对于不支持多表的后端，使用多表的后端不可知方法是利用相关子查询。 有关示例，请参阅下面链接的教程部分。

亦见

相关更新

UPDATE..FROM

多表删除

```py
method values(*args: _DMLColumnKeyMapping[Any] | Sequence[Any], **kwargs: Any) → Self
```

*继承自* `ValuesBase.values()` *方法的* `ValuesBase`

为 INSERT 语句指定一个固定的 VALUES 子句，或者为 UPDATE 指定 SET 子句。

请注意，`Insert`和`Update`构造支持基于传递给`Connection.execute()`的参数对 VALUES 和/或 SET 子句进行每次执行时间格式化。 但是，`ValuesBase.values()`方法可用于将特定参数集固定到语句中。

多次调用`ValuesBase.values()`将产生一个新的构造，每个构造都将参数列表修改为包含新发送的参数。 在单个参数字典的典型情况下，新传递的键将替换先前构造中的相同键。 在基于列表的“多值”构造的情况下，每个新值列表都被扩展到现有值列表上。

参数：

+   `**kwargs` –

    表示将映射到要渲染到 VALUES 或 SET 子句中的值的`Column`的字符串键值对：

    ```py
    users.insert().values(name="some name")

    users.update().where(users.c.id==5).values(name="some name")
    ```

+   `*args` –

    作为传递键/值参数的替代方案，可以将字典、元组或字典或元组的列表作为单个位置参数传递，以形成语句的 VALUES 或 SET 子句。被接受的形式因为是 `Insert` 还是 `Update` 构造而异。

    对于 `Insert` 或 `Update` 构造，可以传递单个字典，其效果与 kwargs 形式相同：

    ```py
    users.insert().values({"name": "some name"})

    users.update().values({"name": "some new name"})
    ```

    对于任何形式，但更常见的是 `Insert` 构造，也可以接受包含表中每一列条目的元组：

    ```py
    users.insert().values((5, "some name"))
    ```

    `Insert` 构造还支持传递字典或完整表元组的列表，这将在服务器上呈现较少使用的 SQL 语法 "多个值" - 此语法受到后端（如 SQLite、PostgreSQL、MySQL）的支持，但不一定适用于其他后端：

    ```py
    users.insert().values([
                        {"name": "some name"},
                        {"name": "some other name"},
                        {"name": "yet another name"},
                    ])
    ```

    上述形式将呈现类似于多个 VALUES 语句：

    ```py
    INSERT INTO users (name) VALUES
                    (:name_1),
                    (:name_2),
                    (:name_3)
    ```

    必须注意 **传递多个值并不等同于使用传统的 executemany() 形式**。上述语法是一种 **特殊** 的语法，通常不使用。要针对多行发出 INSERT 语句，正常的方法是将多个值列表传递给 `Connection.execute()` 方法，这受到所有数据库后端的支持，并且对于非常大量的参数通常更有效率。

    > > 另请参阅
    > > 
    > > 发送多个参数 - 介绍了传统的 Core 方法，用于 INSERT 和其他语句的多参数集调用。
    > > 
    > UPDATE 构造还支持按特定顺序渲染 SET 参数。有关此功能，请参阅 `Update.ordered_values()` 方法。
    > 
    > > 另请参阅
    > > 
    > > `Update.ordered_values()`

```py
method inline() → Self
```

使此 `Update` 构造 "内联"。

当设置时，通过`default`关键字在`Column`对象上存在的 SQL 默认值将被编译为语句中的‘inline’并且不会预先执行。这意味着它们的值不会出现在`CursorResult.last_updated_params()`返回的字典中。

在 1.4 版本中更改：`update.inline`参数现已由`Update.inline()`方法取代。

```py
method ordered_values(*args: Tuple[_DMLColumnArgument, Any]) → Self
```

使用显式参数排序指定此 UPDATE 语句的 VALUES 子句，在结果 UPDATE 语句的 SET 子句中将保持该顺序。

例如：

```py
stmt = table.update().ordered_values(
    ("name", "ed"), ("ident", "foo")
)
```

另请参阅

参数有序更新 - `Update.ordered_values()`方法的完整示例。

在 1.4 版本中更改：`Update.ordered_values()`方法取代了`update.preserve_parameter_order`参数，该参数将在 SQLAlchemy 2.0 中移除。

```py
class sqlalchemy.sql.expression.UpdateBase
```

为`INSERT`、`UPDATE`和`DELETE`语句提供基础。

**成员**

entity_description, exported_columns, params(), return_defaults(), returning(), returning_column_descriptions, with_dialect_options(), with_hint()

**类签名**

类`sqlalchemy.sql.expression.UpdateBase` (`sqlalchemy.sql.roles.DMLRole`, `sqlalchemy.sql.expression.HasCTE`, `sqlalchemy.sql.expression.HasCompileState`, `sqlalchemy.sql.base.DialectKWArgs`, `sqlalchemy.sql.expression.HasPrefixes`, `sqlalchemy.sql.expression.Generative`, `sqlalchemy.sql.expression.ExecutableReturnsRows`, `sqlalchemy.sql.expression.ClauseElement`)

```py
attribute entity_description
```

返回此 DML 结构操作的表和/或实体的插件启用描述。

当使用 ORM 时，此属性通常很有用，因为它返回一个扩展结构，其中包含有关映射实体的信息。从 ORM 启用的 SELECT 和 DML 语句中检查实体和列部分提供了更多背景信息。

对于 Core 语句，此访问器返回的结构源自`UpdateBase.table`属性，并引用正在插入、更新或删除的`Table`：

```py
>>> stmt = insert(user_table)
>>> stmt.entity_description
{
 "name": "user_table",
 "table": Table("user_table", ...)
}
```

版本 1.4.33 中的新功能。

请参阅

`UpdateBase.returning_column_descriptions`

`Select.column_descriptions` - `select()`构造的实体信息

从 ORM 启用的 SELECT 和 DML 语句中检查实体和列 - ORM 背景

```py
attribute exported_columns
```

将 RETURNING 列作为此语句的列集合返回。

版本 1.4 中的新功能。

```py
method params(*arg: Any, **kw: Any) → NoReturn
```

设置语句的参数。

此方法在基类上引发`NotImplementedError`，并由`ValuesBase`覆盖，以提供 UPDATE 和 INSERT 的 SET/VALUES 子句。

```py
method return_defaults(*cols: _DMLColumnArgument, supplemental_cols: Iterable[_DMLColumnArgument] | None = None, sort_by_parameter_order: bool = False) → Self
```

仅支持后端，使用 RETURNING 子句以获取服务器端表达式和默认值。

深度炼金术

`UpdateBase.return_defaults()` 方法被 ORM 用于其内部工作中，以获取新生成的主键和服务器默认值，特别是为了提供`Mapper.eager_defaults` ORM 特性的基础实现，以及允许批量 ORM 插入时的 RETURNING 支持。它的行为相当特殊，实际上并不打算用于一般用途。终端用户应该坚持使用`UpdateBase.returning()` 来添加 RETURNING 子句到他们的 INSERT、UPDATE 和 DELETE 语句中。

通常情况下，单行 INSERT 语句在执行时会自动填充`CursorResult.inserted_primary_key` 属性，该属性以`Row` 对象的形式存储刚刚插入的行的主键，其中列名作为命名元组键（并且`Row._mapping` 视图也被完全填充）。正在使用的方言选择用于填充这些数据的策略；如果它是使用服务器端默认值和/或 SQL 表达式生成的，则通常会使用方言特定的方法，如`cursor.lastrowid`或`RETURNING` 来获取新的主键值。

然而，当在执行语句之前通过调用`UpdateBase.return_defaults()`对语句进行修改时，**仅**对支持 RETURNING 的后端以及将`Table.implicit_returning`参数保持其默认值`True`的`Table`对象发生附加行为。在这些情况下，当从语句的执行返回`CursorResult`时，不仅`CursorResult.inserted_primary_key`像往常一样被填充，`CursorResult.returned_defaults`属性还将被填充为一个命名为`Row`的元组，代表该单行的所有服务器生成的值的完整范围，包括任何指定`Column.server_default`或使用 SQL 表达式的`Column.default`的列的值。

当使用 insertmanyvalues 来调用具有多行的 INSERT 语句时，`UpdateBase.return_defaults()`修饰符将导致`CursorResult.inserted_primary_key_rows`和`CursorResult.returned_defaults_rows`属性完全填充，其中包含代表新插入的主键值以及每行插入的新生成的服务器值的`Row`对象的列表。`CursorResult.inserted_primary_key`和`CursorResult.returned_defaults`属性也将继续填充这两个集合的第一行。

如果后端不支持 RETURNING，或者正在使用的 `Table` 禁用了 `Table.implicit_returning`，则不会添加 RETURNING 子句，也不会获取额外数据，但 INSERT、UPDATE 或 DELETE 语句会正常执行。

例如：

```py
stmt = table.insert().values(data='newdata').return_defaults()

result = connection.execute(stmt)

server_created_at = result.returned_defaults['created_at']
```

当针对 UPDATE 语句使用 `UpdateBase.return_defaults()` 时，会查找包含 `Column.onupdate` 或 `Column.server_onupdate` 参数的列，用于构建默认情况下将包含在 RETURNING 子句中的列（如果未显式指定列）。当针对 DELETE 语句使用时，默认情况下不包含任何列在 RETURNING 中，而必须显式指定，因为在 DELETE 语句进行时通常不会更改值的列。

新功能在版本 2.0 中：`UpdateBase.return_defaults()` 现在也支持 DELETE 语句，并且已经从 `ValuesBase` 移动到 `UpdateBase`。

`UpdateBase.return_defaults()` 方法与 `UpdateBase.returning()` 方法是互斥的，如果同时在一个语句上使用了两者，将在 SQL 编译过程中引发错误。因此，INSERT、UPDATE 或 DELETE 语句的 RETURNING 子句只由其中一个方法控制。

`UpdateBase.return_defaults()` 方法与 `UpdateBase.returning()` 的不同之处在于：

1.  `UpdateBase.return_defaults()`方法导致`CursorResult.returned_defaults`集合被填充为 RETURNING 结果的第一行。当使用`UpdateBase.returning()`时，此属性不会被填充。

1.  `UpdateBase.return_defaults()`与用于获取自动生成的主键值并将其填充到`CursorResult.inserted_primary_key`属性的现有逻辑兼容。相比之下，使用`UpdateBase.returning()`将导致`CursorResult.inserted_primary_key`属性保持未填充。

1.  `UpdateBase.return_defaults()`可以针对任何后端调用。不支持 RETURNING 的后端将跳过该功能的使用，而不是引发异常，*除非*传递了`supplemental_cols`。对于不支持 RETURNING 或目标`Table`设置`Table.implicit_returning`为`False`的后端，`CursorResult.returned_defaults`的返回值将为`None`。

1.  使用`executemany()`调用的 INSERT 语句在后端数据库驱动程序支持 insertmanyvalues 功能的情况下得到支持，这个功能现在大多数包含在 SQLAlchemy 中的后端都支持。当使用`executemany`时，`CursorResult.returned_defaults_rows`和`CursorResult.inserted_primary_key_rows`访问器将返回插入的默认值和主键。

    1.4 版中的新功能：添加了 `CursorResult.returned_defaults_rows` 和 `CursorResult.inserted_primary_key_rows` 访问器。在 2.0 版中，为这些属性提取和填充数据的底层实现被泛化以受到大多数后端的支持，而在 1.4 版中，它们仅受到 `psycopg2` 驱动程序的支持。

参数：

+   `cols` – 可选的列键名或 `Column` 列表，充当将被提取的列的过滤器。

+   `supplemental_cols` –

    可选的 RETURNING 表达式列表，与传递给 `UpdateBase.returning()` 方法的形式相同。当存在时，额外的列将包含在 RETURNING 子句中，并且在返回时 `CursorResult` 对象将被“倒带”，因此像 `CursorResult.all()` 这样的方法将返回新的行，几乎就像语句直接使用了 `UpdateBase.returning()` 一样。但是，与直接使用 `UpdateBase.returning()` 不同，**列的顺序是未定义的**，因此只能使用名称或 `Row._mapping` 键进行定位；它们不能可靠地按位置进行定位。

    2.0 版中的新功能。

+   `sort_by_parameter_order` –

    对于针对多个参数集执行的批量插入，组织 RETURNING 的结果，使返回的行与传入的参数集的顺序对应。这仅适用于支持方言的 executemany 执行，并通常利用 insertmanyvalues 特性。

    2.0.10 版中的新功能。

    另请参阅

    将 RETURNING 行与参数集相关联 - 关于对批量插入的 RETURNING 行进行排序的背景信息

另请参阅

`UpdateBase.returning()`

`CursorResult.returned_defaults`

`CursorResult.returned_defaults_rows`

`CursorResult.inserted_primary_key`

`CursorResult.inserted_primary_key_rows`

```py
method returning(*cols: _ColumnsClauseArgument[Any], sort_by_parameter_order: bool = False, **_UpdateBase__kw: Any) → UpdateBase
```

为该语句添加 RETURNING 或等效子句。

例如：

```py
>>> stmt = (
...     table.update()
...     .where(table.c.data == "value")
...     .values(status="X")
...     .returning(table.c.server_flag, table.c.updated_timestamp)
... )
>>> print(stmt)
UPDATE  some_table  SET  status=:status
WHERE  some_table.data  =  :data_1
RETURNING  some_table.server_flag,  some_table.updated_timestamp 
```

该方法可以多次调用以将新条目添加到要返回的表达式列表中。

新版本 1.4.0b2 中添加：该方法可以多次调用以将新条目添加到要返回的表达式列表中。

给定的列表达式集应源自于 INSERT、UPDATE 或 DELETE 的目标表。虽然`Column`对象是典型的，但元素也可以是表达式：

```py
>>> stmt = table.insert().returning(
...     (table.c.first_name + " " + table.c.last_name).label("fullname")
... )
>>> print(stmt)
INSERT  INTO  some_table  (first_name,  last_name)
VALUES  (:first_name,  :last_name)
RETURNING  some_table.first_name  ||  :first_name_1  ||  some_table.last_name  AS  fullname 
```

在编译时，RETURNING 子句或数据库等效子句将在语句内呈现。对于 INSERT 和 UPDATE，值是新插入/更新的值。对于 DELETE，值是已删除行的值。

在执行时，要返回的列的值通过结果集可用，并且可以使用`CursorResult.fetchone()`等进行迭代。对于不本地支持返回值的 DBAPI（即 cx_oracle），SQLAlchemy 将在结果级别近似此行为，以便提供合理数量的行为中立性。

注意，并非所有的数据库/DBAPI 都支持 RETURNING。对于不支持的后端，在编译和/或执行时会引发异常。对于支持它的后端，跨后端的功能差异很大，包括对 executemany()和其他返回多行的语句的限制。请阅读所使用数据库的文档注释，以确定 RETURNING 的可用性。

参数：

+   `*cols` – 一系列列、SQL 表达式或整个表实体将被返回。

+   `sort_by_parameter_order` –

    对于针对多个参数集执行的批量 INSERT，请组织 RETURNING 的结果，使返回的行与传递的参数集的顺序相对应。这仅适用于支持方言的 executemany 执行，并通常利用 insertmanyvalues 功能。

    新版本中添加的 2.0.10。

    另请参阅

    将 RETURNING 行与参数集相关联 - 关于对批量 INSERT 排序 RETURNING 行的背景（核心级讨论）

    将 RETURNING 记录与输入数据顺序相关联 - 与 ORM Bulk INSERT Statements 的使用示例（ORM 级讨论）

另请参阅

`UpdateBase.return_defaults()` - 一种针对单行 INSERTs 或 UPDATEs 的有效提取服务器端默认值和触发器的替代方法。

INSERT…RETURNING - 在 SQLAlchemy 统一教程 中

```py
attribute returning_column_descriptions
```

返回此 DML 构造与之相对应的列的 插件启用 描述，换句话说，作为 `UpdateBase.returning()` 的一部分建立的表达式。

当使用 ORM 时，此属性通常很有用，因为它返回了一个包含有关映射实体信息的扩展结构。该部分 从启用 ORM 的 SELECT 和 DML 语句检查实体和列 包含了更多的背景知识。

对于 Core 语句，此访问器返回的结构源自与 `UpdateBase.exported_columns` 访问器返回的相同对象：

```py
>>> stmt = insert(user_table).returning(user_table.c.id, user_table.c.name)
>>> stmt.entity_description
[
 {
 "name": "id",
 "type": Integer,
 "expr": Column("id", Integer(), table=<user>, ...)
 },
 {
 "name": "name",
 "type": String(),
 "expr": Column("name", String(), table=<user>, ...)
 },
]
```

从版本 1.4.33 开始新添加。

另请参阅

`UpdateBase.entity_description`

`Select.column_descriptions` - 一个 `select()` 构造的实体信息

从启用 ORM 的 SELECT 和 DML 语句检查实体和列 - ORM 背景

```py
method with_dialect_options(**opt: Any) → Self
```

为这个 INSERT/UPDATE/DELETE 对象添加方言选项。

例如：

```py
upd = table.update().dialect_options(mysql_limit=10)
```

```py
method with_hint(text: str, selectable: _DMLTableArgument | None = None, dialect_name: str = '*') → Self
```

为这个 INSERT/UPDATE/DELETE 语句添加一个单独的表提示。

注意

`UpdateBase.with_hint()` 目前仅适用于 Microsoft SQL Server。对于 MySQL INSERT/UPDATE/DELETE 提示，请使用 `UpdateBase.prefix_with()`。

提示文本根据正在使用的数据库后端在适当的位置呈现，相对于这个语句的主题 `Table` ，或者可选地传递给 `Table` 的 `selectable` 参数的 `Table` 。

`dialect_name`选项将限制特定提示的渲染到特定后端。例如，要添加仅在 SQL Server 上生效的提示：

```py
mytable.insert().with_hint("WITH (PAGLOCK)", dialect_name="mssql")
```

参数：

+   `text` – 提示的文本。

+   `selectable` – 可选的`Table`，指定 UPDATE 或 DELETE 中 FROM 子句的一个元素作为提示的主题 - 仅适用于某些后端。

+   `dialect_name` – 默认为`*`，如果指定为特定方言的名称，则仅在使用该方言时应用这些提示。

```py
class sqlalchemy.sql.expression.ValuesBase
```

为`ValuesBase.values()`提供了对 INSERT 和 UPDATE 构造的支持。

**成员**

select, values()

**类签名**

类`sqlalchemy.sql.expression.ValuesBase`（`sqlalchemy.sql.expression.UpdateBase`）

```py
attribute select: Select[Any] | None = None
```

INSERT .. FROM SELECT 的 SELECT 语句

```py
method values(*args: _DMLColumnKeyMapping[Any] | Sequence[Any], **kwargs: Any) → Self
```

为 INSERT 语句指定一个固定的 VALUES 子句，或为 UPDATE 指定 SET 子句。

请注意，`Insert`和`Update`构造支持基于传递给`Connection.execute()`的参数对 VALUES 和/或 SET 子句进行执行时格式化。但是，`ValuesBase.values()`方法可用于将特定一组参数固定到语句中。

对`ValuesBase.values()`的多次调用将生成一个新的构造，每个构造的参数列表都修改为包括新发送的参数。在单个参数字典的典型情况下，新传递的键将替换上一个构造中的相同键。在基于列表的“多个值”构造的情况下，每个新值列表都会附加到现有的值列表上。

参数：

+   `**kwargs` –

    表示`Column`的字符串键的键值对映射到要呈现到 VALUES 或 SET 子句中的值：

    ```py
    users.insert().values(name="some name")

    users.update().where(users.c.id==5).values(name="some name")
    ```

+   `*args` –

    作为传递键/值参数的替代方案，可以传递一个字典、元组或字典或元组的列表作为单个位置参数，以形成语句的 VALUES 或 SET 子句。接受的形式因此是一个 `Insert` 或 `Update` 结构而异。

    对于 `Insert` 或 `Update` 结构，可以传递单个字典，其工作方式与关键字参数形式相同：

    ```py
    users.insert().values({"name": "some name"})

    users.update().values({"name": "some new name"})
    ```

    同样适用于任何形式，但更典型的是对于 `Insert` 结构，还可以接受一个包含表中每一列的条目的元组：

    ```py
    users.insert().values((5, "some name"))
    ```

    `Insert` 结构还支持传递字典或完整表元组的列表，这将在服务器上呈现较少见的 SQL 语法“多个值” - 此语法在后端，如 SQLite、PostgreSQL、MySQL 等中受支持，但不一定适用于其他后端：

    ```py
    users.insert().values([
                        {"name": "some name"},
                        {"name": "some other name"},
                        {"name": "yet another name"},
                    ])
    ```

    上述形式将呈现类似于多个 VALUES 语句的内容：

    ```py
    INSERT INTO users (name) VALUES
                    (:name_1),
                    (:name_2),
                    (:name_3)
    ```

    需要注意的是，**传递多个值并不等同于使用传统的 executemany() 形式**。上述语法是一种**特殊**的语法，通常不常用。要针对多行发出 INSERT 语句，正常方法是将多个值列表传递给 `Connection.execute()` 方法，该方法受到所有数据库后端的支持，并且通常对于非常大量的参数更有效率。

    > > 请参阅
    > > 
    > > 发送多个参数 - 介绍了用于 INSERT 和其他语句的传统 Core 方法的多个参数集调用。
    > > 
    > UPDATE 结构还支持以特定顺序呈现 SET 参数。有关此功能，请参阅 `Update.ordered_values()` 方法。
    > 
    > > 请参阅
    > > 
    > > `Update.ordered_values()`
