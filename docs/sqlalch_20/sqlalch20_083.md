# 自定义 SQL 构造和编译扩展

> 原文：[`docs.sqlalchemy.org/en/20/core/compiler.html`](https://docs.sqlalchemy.org/en/20/core/compiler.html)

提供了用于创建自定义 ClauseElements 和编译器的 API。

## 概要

使用涉及创建一个或多个`ClauseElement`子类和一个或多个定义其编译的可调用对象：

```py
from sqlalchemy.ext.compiler import compiles
from sqlalchemy.sql.expression import ColumnClause

class MyColumn(ColumnClause):
    inherit_cache = True

@compiles(MyColumn)
def compile_mycolumn(element, compiler, **kw):
    return "[%s]" % element.name
```

在上面，`MyColumn`扩展了`ColumnClause`，这是命名列对象的基本表达式元素。`compiles`装饰器向`MyColumn`类注册自身，以便在将对象编译为字符串时调用它：

```py
from sqlalchemy import select

s = select(MyColumn('x'), MyColumn('y'))
print(str(s))
```

产生：

```py
SELECT [x], [y]
```

## 方言特定的编译规则

编译器也可以是特定于方言的。将为使用的方言调用适当的编译器：

```py
from sqlalchemy.schema import DDLElement

class AlterColumn(DDLElement):
    inherit_cache = False

    def __init__(self, column, cmd):
        self.column = column
        self.cmd = cmd

@compiles(AlterColumn)
def visit_alter_column(element, compiler, **kw):
    return "ALTER COLUMN %s ..." % element.column.name

@compiles(AlterColumn, 'postgresql')
def visit_alter_column(element, compiler, **kw):
    return "ALTER TABLE %s ALTER COLUMN %s ..." % (element.table.name,
                                                   element.column.name)
```

当使用任何`postgresql`方言时，第二个`visit_alter_table`将被调用。

## 编译自定义表达式构造的子元素

`compiler`参数是正在使用的`Compiled`对象。可以检查此对象的任何有关进行中编译的信息，包括`compiler.dialect`、`compiler.statement`等。`SQLCompiler`和`DDLCompiler`都包括一个`process()`方法，可用于编译嵌入属性：

```py
from sqlalchemy.sql.expression import Executable, ClauseElement

class InsertFromSelect(Executable, ClauseElement):
    inherit_cache = False

    def __init__(self, table, select):
        self.table = table
        self.select = select

@compiles(InsertFromSelect)
def visit_insert_from_select(element, compiler, **kw):
    return "INSERT INTO %s (%s)" % (
        compiler.process(element.table, asfrom=True, **kw),
        compiler.process(element.select, **kw)
    )

insert = InsertFromSelect(t1, select(t1).where(t1.c.x>5))
print(insert)
```

产生：

```py
"INSERT INTO mytable (SELECT mytable.x, mytable.y, mytable.z
                      FROM mytable WHERE mytable.x > :x_1)"
```

注意

上述的`InsertFromSelect`构造仅是一个示例，这种实际功能已经可以使用`Insert.from_select()`方法。

### 在 SQL 和 DDL 编译器之间进行交叉编译

SQL 和 DDL 构造分别使用不同的基本编译器 - `SQLCompiler`和`DDLCompiler`。一个常见的需求是从 DDL 表达式中访问 SQL 表达式的编译规则。`DDLCompiler`包含一个访问器`sql_compiler`，因此我们可以生成嵌入 SQL 表达式的 CHECK 约束，如下所示：

```py
@compiles(MyConstraint)
def compile_my_constraint(constraint, ddlcompiler, **kw):
    kw['literal_binds'] = True
    return "CONSTRAINT %s CHECK (%s)" % (
        constraint.name,
        ddlcompiler.sql_compiler.process(
            constraint.expression, **kw)
    )
```

在上面，我们在`SQLCompiler.process()`中调用的过程步骤中添加了一个额外的标志，即`literal_binds`标志。这表示任何引用`BindParameter`对象或其他“literal”对象（如引用字符串或整数的对象）的 SQL 表达式应该**原地**呈现，而不是作为绑定参数引用；在发出 DDL 时，通常不支持绑定参数。

## 更改现有构造的默认编译

编译器扩展同样适用于现有的结构。当覆盖内置 SQL 结构的编译时，@compiles 装饰器会调用适当的类（确保使用类，即 `Insert` 或 `Select`，而不是创建函数，比如 `insert()` 或 `select()`）。

在新的编译函数中，要获取“原始”的编译例程，使用适当的 visit_XXX 方法 - 这是因为编译器.process() 将调用重写的例程并导致无限循环。比如，要向所有的插入语句添加“前缀”：

```py
from sqlalchemy.sql.expression import Insert

@compiles(Insert)
def prefix_inserts(insert, compiler, **kw):
    return compiler.visit_insert(insert.prefix_with("some prefix"), **kw)
```

上述编译器在编译时会在所有的 INSERT 语句前加上“某个前缀”。

## 更改类型的编译

`compiler` 也适用于类型，比如下面我们为 `String`/`VARCHAR` 实现了 MS-SQL 特定的 ‘max’ 关键字：

```py
@compiles(String, 'mssql')
@compiles(VARCHAR, 'mssql')
def compile_varchar(element, compiler, **kw):
    if element.length == 'max':
        return "VARCHAR('max')"
    else:
        return compiler.visit_VARCHAR(element, **kw)

foo = Table('foo', metadata,
    Column('data', VARCHAR('max'))
)
```

## 子类化指南

使用编译器扩展的一个重要部分是子类化 SQLAlchemy 表达式结构。为了使这更容易，表达式和模式包含一组旨在用于常见任务的“基础”。概要如下：

+   `ClauseElement` - 这是根表达式类。任何 SQL 表达式都可以从这个基类派生，并且对于长一些的构造，比如专门的 INSERT 语句，这可能是最好的选择。

+   `ColumnElement` - 所有“类似列”的元素的根。你在 SELECT 语句的“列”子句中（以及 order by 和 group by）放置的任何东西都可以从这个派生 - 该对象将自动具有 Python 的“比较”行为。

    `ColumnElement` 类希望有一个 `type` 成员，该成员是表达式的返回类型。这可以在构造函数的实例级别或类级别上建立：

    ```py
    class timestamp(ColumnElement):
        type = TIMESTAMP()
        inherit_cache = True
    ```

+   `FunctionElement` - 这是 `ColumnElement` 和“from 子句”类似对象的混合体，并表示 SQL 函数或存储过程类型的调用。由于大多数数据库支持“SELECT FROM <某个函数>”这样的语句，`FunctionElement` 添加了在 `select()` 构造的 FROM 子句中使用的能力：

    ```py
    from sqlalchemy.sql.expression import FunctionElement

    class coalesce(FunctionElement):
        name = 'coalesce'
        inherit_cache = True

    @compiles(coalesce)
    def compile(element, compiler, **kw):
        return "coalesce(%s)" % compiler.process(element.clauses, **kw)

    @compiles(coalesce, 'oracle')
    def compile(element, compiler, **kw):
        if len(element.clauses) > 2:
            raise TypeError("coalesce only supports two arguments on Oracle")
        return "nvl(%s)" % compiler.process(element.clauses, **kw)
    ```

+   `ExecutableDDLElement` - 所有 DDL 表达式的根，比如 CREATE TABLE，ALTER TABLE 等。`ExecutableDDLElement`的子类的编译由`DDLCompiler`发出，而不是由`SQLCompiler`发出。`ExecutableDDLElement`也可以与`DDLEvents.before_create()`和`DDLEvents.after_create()`等事件钩子一起用作事件钩子，允许在 CREATE TABLE 和 DROP TABLE 序列期间自动调用构造。

    另请参见

    自定义 DDL - 包含将`DDL`对象（它们本身是`ExecutableDDLElement`实例）与`DDLEvents`事件钩子相关联的示例。

+   `Executable` - 这是一个 mixin，应该与任何表示可以直接传递给`execute()`方法的“独立”SQL 语句的表达式类一起使用。它已经隐含在`DDLElement`和`FunctionElement`中。

上述大部分构造也响应 SQL 语句缓存。子类构造将希望为对象定义缓存行为，这通常意味着将标志`inherit_cache`设置为`False`或`True`的值。有关背景信息，请参见下一节为自定义构造启用缓存支持。

## 为自定义构造启用缓存支持

从版本 1.4 开始，SQLAlchemy 包括一个 SQL 编译缓存设施，它将允许等效的 SQL 构造缓存它们的字符串形式，以及用于从语句中获取结果的其他结构信息。

如在对象不会生成缓存键，性能影响中讨论的原因，该缓存系统的实现对于在缓存系统中包含自定义 SQL 构造和/或子类采取了保守的方法。这包括任何用户定义的 SQL 构造，包括此扩展的所有示例，除非它们明确声明能够这样做，否则默认情况下不会参与缓存。当在特定子类的类级别设置`HasCacheKey.inherit_cache`属性为`True`时，将指示该类的实例可以安全地进行缓存，使用直接父类的缓存键生成方案。例如，这适用于先前指示的“概要”示例：

```py
class MyColumn(ColumnClause):
    inherit_cache = True

@compiles(MyColumn)
def compile_mycolumn(element, compiler, **kw):
    return "[%s]" % element.name
```

上面，`MyColumn`类不包含任何影响其 SQL 编译的新状态；`MyColumn`实例的缓存键将利用`ColumnClause`超类的缓存键，这意味着它将考虑对象的类（`MyColumn`）、对象的字符串名称和数据类型：

```py
>>> MyColumn("some_name", String())._generate_cache_key()
CacheKey(
 key=('0', <class '__main__.MyColumn'>,
 'name', 'some_name',
 'type', (<class 'sqlalchemy.sql.sqltypes.String'>,
 ('length', None), ('collation', None))
), bindparams=[])
```

对于那些**在许多更大语句中可能被大量使用作为组件**的对象，比如`Column`子类和自定义 SQL 数据类型，尽可能**启用缓存**非常重要，否则可能会对性能产生负面影响。

一个**包含影响其 SQL 编译的状态**的对象示例是在编译自定义表达式构造的子元素中所示的一个示例；这是一个将`Table`和`Select`构造组合在一起的“INSERT FROM SELECT”构造，每个构造独立影响构造的 SQL 字符串生成。对于这个类，示例说明它根本不参与缓存：

```py
class InsertFromSelect(Executable, ClauseElement):
    inherit_cache = False

    def __init__(self, table, select):
        self.table = table
        self.select = select

@compiles(InsertFromSelect)
def visit_insert_from_select(element, compiler, **kw):
    return "INSERT INTO %s (%s)" % (
        compiler.process(element.table, asfrom=True, **kw),
        compiler.process(element.select, **kw)
    )
```

虽然上述`InsertFromSelect`也可能生成由`Table`和`Select`组件组合而成的缓存键，但目前该 API 并不完全公开。然而，对于“INSERT FROM SELECT”构造，它仅用于特定操作，缓存并不像前面的示例那样关键。

对于**在相对孤立并且通常是独立的**对象，例如自定义 DML 构造，如 “INSERT FROM SELECT”，**缓存通常不那么关键**，因为对于这种构造缺乏缓存仅对该特定操作具有局部影响。

## 更多示例

### “UTC 时间戳”函数

一个类似于 “CURRENT_TIMESTAMP” 的函数，但应用适当的转换，使时间为 UTC 时间。时间戳最好存储在关系型数据库中作为 UTC，不带时区。UTC 使您的数据库在夏令时结束时不会认为时间已经倒退，不带时区是因为时区就像字符编码 - 最好只在应用程序的端点（即在用户输入时转换为 UTC，在显示时重新应用所需的时区）应用它们。

对于 PostgreSQL 和 Microsoft SQL Server：

```py
from sqlalchemy.sql import expression
from sqlalchemy.ext.compiler import compiles
from sqlalchemy.types import DateTime

class utcnow(expression.FunctionElement):
    type = DateTime()
    inherit_cache = True

@compiles(utcnow, 'postgresql')
def pg_utcnow(element, compiler, **kw):
    return "TIMEZONE('utc', CURRENT_TIMESTAMP)"

@compiles(utcnow, 'mssql')
def ms_utcnow(element, compiler, **kw):
    return "GETUTCDATE()"
```

示例用法：

```py
from sqlalchemy import (
            Table, Column, Integer, String, DateTime, MetaData
        )
metadata = MetaData()
event = Table("event", metadata,
    Column("id", Integer, primary_key=True),
    Column("description", String(50), nullable=False),
    Column("timestamp", DateTime, server_default=utcnow())
)
```

### “GREATEST”函数

“GREATEST”函数接受任意数量的参数，并返回具有最高值的参数 - 它等同于 Python 的 `max` 函数。与仅容纳两个参数的基于 CASE 的版本相比，SQL 标准版本：

```py
from sqlalchemy.sql import expression, case
from sqlalchemy.ext.compiler import compiles
from sqlalchemy.types import Numeric

class greatest(expression.FunctionElement):
    type = Numeric()
    name = 'greatest'
    inherit_cache = True

@compiles(greatest)
def default_greatest(element, compiler, **kw):
    return compiler.visit_function(element)

@compiles(greatest, 'sqlite')
@compiles(greatest, 'mssql')
@compiles(greatest, 'oracle')
def case_greatest(element, compiler, **kw):
    arg1, arg2 = list(element.clauses)
    return compiler.process(case((arg1 > arg2, arg1), else_=arg2), **kw)
```

示例用法：

```py
Session.query(Account).\
        filter(
            greatest(
                Account.checking_balance,
                Account.savings_balance) > 10000
        )
```

### “false” 表达式

渲染“false”常量表达式，对于没有“false”常量的平台，渲染为“0”：

```py
from sqlalchemy.sql import expression
from sqlalchemy.ext.compiler import compiles

class sql_false(expression.ColumnElement):
    inherit_cache = True

@compiles(sql_false)
def default_false(element, compiler, **kw):
    return "false"

@compiles(sql_false, 'mssql')
@compiles(sql_false, 'mysql')
@compiles(sql_false, 'oracle')
def int_false(element, compiler, **kw):
    return "0"
```

示例用法：

```py
from sqlalchemy import select, union_all

exp = union_all(
    select(users.c.name, sql_false().label("enrolled")),
    select(customers.c.name, customers.c.enrolled)
)
```

| 对象名称 | 描述 |
| --- | --- |
| compiles(class_, *specs) | 为给定`ClauseElement`类型注册函数作为编译器。 |
| deregister(class_) | 删除与给定`ClauseElement`类型关联的所有自定义编译器。 |

```py
function sqlalchemy.ext.compiler.compiles(class_, *specs)
```

为给定`ClauseElement`类型注册函数作为编译器。

```py
function sqlalchemy.ext.compiler.deregister(class_)
```

删除与给定`ClauseElement`类型关联的所有自定义编译器。

## 概要

使用涉及创建一个或多个`ClauseElement`子类和一个或多个定义其编译的可调用对象：

```py
from sqlalchemy.ext.compiler import compiles
from sqlalchemy.sql.expression import ColumnClause

class MyColumn(ColumnClause):
    inherit_cache = True

@compiles(MyColumn)
def compile_mycolumn(element, compiler, **kw):
    return "[%s]" % element.name
```

上面，`MyColumn` 扩展了`ColumnClause`，命名列对象的基本表达式元素。`compiles` 装饰器将自身注册到 `MyColumn` 类，以便在对象编译为字符串时调用它：

```py
from sqlalchemy import select

s = select(MyColumn('x'), MyColumn('y'))
print(str(s))
```

产生：

```py
SELECT [x], [y]
```

## 特定于方言的编译规则

编译器也可以是特定于方言的。将为使用的方言调用适当的编译器：

```py
from sqlalchemy.schema import DDLElement

class AlterColumn(DDLElement):
    inherit_cache = False

    def __init__(self, column, cmd):
        self.column = column
        self.cmd = cmd

@compiles(AlterColumn)
def visit_alter_column(element, compiler, **kw):
    return "ALTER COLUMN %s ..." % element.column.name

@compiles(AlterColumn, 'postgresql')
def visit_alter_column(element, compiler, **kw):
    return "ALTER TABLE %s ALTER COLUMN %s ..." % (element.table.name,
                                                   element.column.name)
```

当使用任何 `postgresql` 方言时，将调用第二个 `visit_alter_table`。

## 编译自定义表达式结构的子元素

`compiler` 参数是正在使用的 `Compiled` 对象。此对象可以用于检查关于正在进行的编译的任何信息，包括 `compiler.dialect`、`compiler.statement` 等。`SQLCompiler` 和 `DDLCompiler` 都包含一个 `process()` 方法，可用于编译嵌入属性：

```py
from sqlalchemy.sql.expression import Executable, ClauseElement

class InsertFromSelect(Executable, ClauseElement):
    inherit_cache = False

    def __init__(self, table, select):
        self.table = table
        self.select = select

@compiles(InsertFromSelect)
def visit_insert_from_select(element, compiler, **kw):
    return "INSERT INTO %s (%s)" % (
        compiler.process(element.table, asfrom=True, **kw),
        compiler.process(element.select, **kw)
    )

insert = InsertFromSelect(t1, select(t1).where(t1.c.x>5))
print(insert)
```

产生：

```py
"INSERT INTO mytable (SELECT mytable.x, mytable.y, mytable.z
                      FROM mytable WHERE mytable.x > :x_1)"
```

注意

上述的 `InsertFromSelect` 构造只是一个例子，实际功能已经可以使用 `Insert.from_select()` 方法实现。

### 在 SQL 和 DDL 编译器之间进行交叉编译

SQL 和 DDL 构造使用不同的基础编译器 - `SQLCompiler` 和 `DDLCompiler` 进行编译。常见的需要是从 DDL 表达式中访问 SQL 表达式的编译规则。因此，`DDLCompiler` 包含一个访问器 `sql_compiler`，如下所示，我们生成一个嵌入了 SQL 表达式的 CHECK 约束：

```py
@compiles(MyConstraint)
def compile_my_constraint(constraint, ddlcompiler, **kw):
    kw['literal_binds'] = True
    return "CONSTRAINT %s CHECK (%s)" % (
        constraint.name,
        ddlcompiler.sql_compiler.process(
            constraint.expression, **kw)
    )
```

在上面的例子中，我们在由 `SQLCompiler.process()` 调用的处理步骤中添加了一个额外的标志，即 `literal_binds` 标志。这表示任何引用 `BindParameter` 对象或其他“文字”对象（如引用字符串或整数的对象）的 SQL 表达式应该**就地**渲染，而不是作为一个绑定参数引用；在发出 DDL 时，通常不支持绑定参数。

### 在 SQL 和 DDL 编译器之间进行交叉编译

SQL 和 DDL 构造使用不同的基础编译器 - `SQLCompiler` 和 `DDLCompiler` 进行编译。常见的需要是从 DDL 表达式中访问 SQL 表达式的编译规则。因此，`DDLCompiler` 包含一个访问器 `sql_compiler`，如下所示，我们生成一个嵌入了 SQL 表达式的 CHECK 约束：

```py
@compiles(MyConstraint)
def compile_my_constraint(constraint, ddlcompiler, **kw):
    kw['literal_binds'] = True
    return "CONSTRAINT %s CHECK (%s)" % (
        constraint.name,
        ddlcompiler.sql_compiler.process(
            constraint.expression, **kw)
    )
```

在上面的例子中，我们在由 `SQLCompiler.process()` 调用的处理步骤中添加了一个额外的标志，即 `literal_binds` 标志。这表示任何引用 `BindParameter` 对象或其他“文字”对象（如引用字符串或整数的对象）的 SQL 表达式应该**就地**渲染，而不是作为一个绑定参数引用；在发出 DDL 时，通常不支持绑定参数。

## 更改现有构造的默认编译

编译器扩展同样适用于现有构造。当重写内置 SQL 构造的编译时，@compiles 装饰器会在适当的类上调用（确保使用类，即 `Insert` 或 `Select`，而不是创建函数，如 `insert()` 或 `select()`）。

在新的编译函数中，要获取“原始”编译例程，使用适当的 visit_XXX 方法 - 这是因为编译器.process() 将调用重写例程并导致无限循环。例如，要向所有插入语句添加“前缀”：

```py
from sqlalchemy.sql.expression import Insert

@compiles(Insert)
def prefix_inserts(insert, compiler, **kw):
    return compiler.visit_insert(insert.prefix_with("some prefix"), **kw)
```

上述编译器在编译时将所有 INSERT 语句前缀为“some prefix”。

## 更改类型的编译

`compiler` 也适用于类型，比如下面我们为 `String`/`VARCHAR` 实现 MS-SQL 特定的 ‘max’ 关键字：

```py
@compiles(String, 'mssql')
@compiles(VARCHAR, 'mssql')
def compile_varchar(element, compiler, **kw):
    if element.length == 'max':
        return "VARCHAR('max')"
    else:
        return compiler.visit_VARCHAR(element, **kw)

foo = Table('foo', metadata,
    Column('data', VARCHAR('max'))
)
```

## 子类指南

使用编译器扩展的一个重要部分是子类化 SQLAlchemy 表达式构造。为了使这更容易，表达式和模式包含一组用于常见任务的“基类”。概要如下：

+   `ClauseElement` - 这是根表达式类。任何 SQL 表达式都可以从这个基类派生，对于像专门的 INSERT 语句这样的较长构造来说，这可能是最好的选择。

+   `ColumnElement` - 所有“列样”元素的根。您在 SELECT 语句的“columns”子句中（以及 order by 和 group by）中放置的任何内容都可以从这里派生 - 该对象将自动具有 Python 的“比较”行为。

    `ColumnElement` 类希望有一个 `type` 成员，该成员是表达式的返回类型。这可以在构造函数的实例级别或在类级别（如果通常是常量）中建立：

    ```py
    class timestamp(ColumnElement):
        type = TIMESTAMP()
        inherit_cache = True
    ```

+   `FunctionElement` - 这是 `ColumnElement` 和“from clause”类似对象的混合体，表示 SQL 函数或存储过程类型的调用。由于大多数数据库支持类似“SELECT FROM <some function>”的语句，`FunctionElement` 添加了在 `select()` 构造的 FROM 子句中使用的能力：

    ```py
    from sqlalchemy.sql.expression import FunctionElement

    class coalesce(FunctionElement):
        name = 'coalesce'
        inherit_cache = True

    @compiles(coalesce)
    def compile(element, compiler, **kw):
        return "coalesce(%s)" % compiler.process(element.clauses, **kw)

    @compiles(coalesce, 'oracle')
    def compile(element, compiler, **kw):
        if len(element.clauses) > 2:
            raise TypeError("coalesce only supports two arguments on Oracle")
        return "nvl(%s)" % compiler.process(element.clauses, **kw)
    ```

+   `ExecutableDDLElement` - 所有 DDL 表达式的根，比如 CREATE TABLE，ALTER TABLE 等。 `ExecutableDDLElement` 的子类的编译由 `DDLCompiler` 发出，而不是 `SQLCompiler`。 `ExecutableDDLElement` 还可以与诸如 `DDLEvents.before_create()` 和 `DDLEvents.after_create()` 等事件钩子一起用作事件钩子，允许在 CREATE TABLE 和 DROP TABLE 序列期间自动调用构造。

    另请参阅

    自定义 DDL - 包含将 `DDL` 对象（它们本身是 `ExecutableDDLElement` 实例）与 `DDLEvents` 事件钩子相关联的示例。

+   `Executable` - 这是一个混合类，应该与表示“独立”SQL 语句的任何表达式类一起使用，可以直接传递给`execute()`方法。 它已经隐式地存在于 `DDLElement` 和 `FunctionElement` 中。

上述大多数构造也会响应 SQL 语句缓存。 子类化的构造将希望为对象定义缓存行为，这通常意味着将标志 `inherit_cache` 设置为 `False` 或 `True` 的值。 有关背景信息，请参见下一节 为自定义构造启用缓存支持。

## 为自定义构造启用缓存支持

截至版本 1.4，SQLAlchemy 包括一个 SQL 编译缓存功能，它将允许等效的 SQL 构造缓存它们的字符串形式，以及用于从语句获取结果的其他结构信息。

由于讨论的原因在对象不会生成缓存键，性能影响，这个缓存系统的实现采用了一种保守的方式来包括自定义 SQL 构造和/或子类在缓存系统中。这包括任何用户定义的 SQL 构造，包括此扩展的所有示例，默认情况下将不参与缓存，除非它们明确声明能够参与缓存。当`HasCacheKey.inherit_cache`属性在特定子类的类级别上设置为`True`时，将表示此类的实例可以安全地缓存，使用其直接超类的缓存键生成方案。例如，这适用于先前指示的“概要”示例：

```py
class MyColumn(ColumnClause):
    inherit_cache = True

@compiles(MyColumn)
def compile_mycolumn(element, compiler, **kw):
    return "[%s]" % element.name
```

在上述示例中，`MyColumn` 类不包含任何影响其 SQL 编译的新状态；`MyColumn` 实例的缓存键将利用 `ColumnClause` 超类的缓存键，这意味着它将考虑对象的类（`MyColumn`）、对象的字符串名称和数据类型：

```py
>>> MyColumn("some_name", String())._generate_cache_key()
CacheKey(
 key=('0', <class '__main__.MyColumn'>,
 'name', 'some_name',
 'type', (<class 'sqlalchemy.sql.sqltypes.String'>,
 ('length', None), ('collation', None))
), bindparams=[])
```

对于可能在许多较大语句中自由使用的对象，例如 `Column` 子类和自定义 SQL 数据类型，尽可能启用缓存是很重要的，否则可能会对性能产生负面影响。

一个包含影响其 SQL 编译的状态的对象示例是在编译自定义表达式结构的子元素中所示的对象；这是一个将 `Table` 与 `Select` 构造组合在一起的“INSERT FROM SELECT”构造，它们各自独立地影响构造的 SQL 字符串生成。对于这个类，示例说明了它根本不参与缓存：

```py
class InsertFromSelect(Executable, ClauseElement):
    inherit_cache = False

    def __init__(self, table, select):
        self.table = table
        self.select = select

@compiles(InsertFromSelect)
def visit_insert_from_select(element, compiler, **kw):
    return "INSERT INTO %s (%s)" % (
        compiler.process(element.table, asfrom=True, **kw),
        compiler.process(element.select, **kw)
    )
```

虽然上述的 `InsertFromSelect` 也可能生成由 `Table` 和 `Select` 组件组成的缓存键，但目前该 API 并不完全公开。但是，对于“INSERT FROM SELECT”构造，它只用于特定操作，缓存并不像前面的示例那样关键。

对于**在相对孤立并且通常是独立的对象**，比如自定义 DML 构造，比如“INSERT FROM SELECT”，**缓存通常不太关键**，因为对于这种构造物的缺乏缓存只会对该特定操作产生局部影响。

## 更多示例

### “UTC 时间戳”函数

一个类似于“CURRENT_TIMESTAMP”的函数，但应用适当的转换，使时间处于 UTC 时间。时间戳最好存储在关系数据库中作为 UTC 时间，不带时区。UTC 时间是为了在夏令时结束时，数据库不会认为时间倒退一小时，不带时区是因为时区就像字符编码一样——最好只在应用程序的端点应用（即在用户输入时转换为 UTC 时间，在显示时重新应用所需的时区）。

对于 PostgreSQL 和 Microsoft SQL Server：

```py
from sqlalchemy.sql import expression
from sqlalchemy.ext.compiler import compiles
from sqlalchemy.types import DateTime

class utcnow(expression.FunctionElement):
    type = DateTime()
    inherit_cache = True

@compiles(utcnow, 'postgresql')
def pg_utcnow(element, compiler, **kw):
    return "TIMEZONE('utc', CURRENT_TIMESTAMP)"

@compiles(utcnow, 'mssql')
def ms_utcnow(element, compiler, **kw):
    return "GETUTCDATE()"
```

示例用法：

```py
from sqlalchemy import (
            Table, Column, Integer, String, DateTime, MetaData
        )
metadata = MetaData()
event = Table("event", metadata,
    Column("id", Integer, primary_key=True),
    Column("description", String(50), nullable=False),
    Column("timestamp", DateTime, server_default=utcnow())
)
```

### “GREATEST”函数

“GREATEST”函数被赋予任意数量的参数，并返回具有最高值的参数——它等同于 Python 的`max`函数。一个 SQL 标准版本与一个基于 CASE 的版本相对应，后者仅容纳两个参数：

```py
from sqlalchemy.sql import expression, case
from sqlalchemy.ext.compiler import compiles
from sqlalchemy.types import Numeric

class greatest(expression.FunctionElement):
    type = Numeric()
    name = 'greatest'
    inherit_cache = True

@compiles(greatest)
def default_greatest(element, compiler, **kw):
    return compiler.visit_function(element)

@compiles(greatest, 'sqlite')
@compiles(greatest, 'mssql')
@compiles(greatest, 'oracle')
def case_greatest(element, compiler, **kw):
    arg1, arg2 = list(element.clauses)
    return compiler.process(case((arg1 > arg2, arg1), else_=arg2), **kw)
```

示例用法：

```py
Session.query(Account).\
        filter(
            greatest(
                Account.checking_balance,
                Account.savings_balance) > 10000
        )
```

### “false”表达式

渲染“false”常量表达式，在没有“false”常量的平台上呈现为“0”：

```py
from sqlalchemy.sql import expression
from sqlalchemy.ext.compiler import compiles

class sql_false(expression.ColumnElement):
    inherit_cache = True

@compiles(sql_false)
def default_false(element, compiler, **kw):
    return "false"

@compiles(sql_false, 'mssql')
@compiles(sql_false, 'mysql')
@compiles(sql_false, 'oracle')
def int_false(element, compiler, **kw):
    return "0"
```

示例用法：

```py
from sqlalchemy import select, union_all

exp = union_all(
    select(users.c.name, sql_false().label("enrolled")),
    select(customers.c.name, customers.c.enrolled)
)
```

### “UTC 时间戳”函数

一个类似于“CURRENT_TIMESTAMP”的函数，但应用适当的转换，使时间处于 UTC 时间。时间戳最好存储在关系数据库中作为 UTC 时间，不带时区。UTC 时间是为了在夏令时结束时，数据库不会认为时间倒退一小时，不带时区是因为时区就像字符编码一样——最好只在应用程序的端点应用（即在用户输入时转换为 UTC 时间，在显示时重新应用所需的时区）。

对于 PostgreSQL 和 Microsoft SQL Server：

```py
from sqlalchemy.sql import expression
from sqlalchemy.ext.compiler import compiles
from sqlalchemy.types import DateTime

class utcnow(expression.FunctionElement):
    type = DateTime()
    inherit_cache = True

@compiles(utcnow, 'postgresql')
def pg_utcnow(element, compiler, **kw):
    return "TIMEZONE('utc', CURRENT_TIMESTAMP)"

@compiles(utcnow, 'mssql')
def ms_utcnow(element, compiler, **kw):
    return "GETUTCDATE()"
```

示例用法：

```py
from sqlalchemy import (
            Table, Column, Integer, String, DateTime, MetaData
        )
metadata = MetaData()
event = Table("event", metadata,
    Column("id", Integer, primary_key=True),
    Column("description", String(50), nullable=False),
    Column("timestamp", DateTime, server_default=utcnow())
)
```

### “GREATEST”函数

“GREATEST”函数被赋予任意数量的参数，并返回具有最高值的参数——它等同于 Python 的`max`函数。一个 SQL 标准版本与一个基于 CASE 的版本相对应，后者仅容纳两个参数：

```py
from sqlalchemy.sql import expression, case
from sqlalchemy.ext.compiler import compiles
from sqlalchemy.types import Numeric

class greatest(expression.FunctionElement):
    type = Numeric()
    name = 'greatest'
    inherit_cache = True

@compiles(greatest)
def default_greatest(element, compiler, **kw):
    return compiler.visit_function(element)

@compiles(greatest, 'sqlite')
@compiles(greatest, 'mssql')
@compiles(greatest, 'oracle')
def case_greatest(element, compiler, **kw):
    arg1, arg2 = list(element.clauses)
    return compiler.process(case((arg1 > arg2, arg1), else_=arg2), **kw)
```

示例用法：

```py
Session.query(Account).\
        filter(
            greatest(
                Account.checking_balance,
                Account.savings_balance) > 10000
        )
```

### “false”表达式

渲染“false”常量表达式，在没有“false”常量的平台上呈现为“0”：

```py
from sqlalchemy.sql import expression
from sqlalchemy.ext.compiler import compiles

class sql_false(expression.ColumnElement):
    inherit_cache = True

@compiles(sql_false)
def default_false(element, compiler, **kw):
    return "false"

@compiles(sql_false, 'mssql')
@compiles(sql_false, 'mysql')
@compiles(sql_false, 'oracle')
def int_false(element, compiler, **kw):
    return "0"
```

示例用法：

```py
from sqlalchemy import select, union_all

exp = union_all(
    select(users.c.name, sql_false().label("enrolled")),
    select(customers.c.name, customers.c.enrolled)
)
```
