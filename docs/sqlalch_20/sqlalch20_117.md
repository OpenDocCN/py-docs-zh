# SQL 表达式

> 原文：[`docs.sqlalchemy.org/en/20/faq/sqlexpressions.html`](https://docs.sqlalchemy.org/en/20/faq/sqlexpressions.html)

+   如何将 SQL 表达式呈现为字符串，可能包含内联的绑定参数？

    +   针对特定数据库进行字符串化

    +   内联呈现绑定参数

    +   将“POSTCOMPILE”参数呈现为绑定参数

+   在字符串化 SQL 语句时为什么百分号会被双倍显示？

+   我正在使用 op() 生成自定义运算符，但我的括号没有正确显示

    +   为什么括号规则是这样的？

## 如何将 SQL 表达式呈现为字符串，可能包含内联的绑定参数？

SQLAlchemy Core 语句对象或表达式片段的“字符串化”，以及 ORM `Query` 对象，在大多数简单情况下都可以简单地使用 `str()` 内置函数来实现，如下所示，当与 `print` 函数一起使用时（请注意 Python `print` 函数如果不显式使用 `str()`，也会自动调用它）：

```py
>>> from sqlalchemy import table, column, select
>>> t = table("my_table", column("x"))
>>> statement = select(t)
>>> print(str(statement))
SELECT  my_table.x
FROM  my_table 
```

`str()` 内置函数或等效函数，可在 ORM `Query` 对象上调用，也可在诸如 `select()`、`insert()` 等语句上调用，还可在任何表达式片段上调用，例如：

```py
>>> from sqlalchemy import column
>>> print(column("x") == "some value")
x  =  :x_1 
```

### 针对特定数据库进行字符串化

当我们要将语句或片段字符串化时，如果包含具有特定于数据库的字符串格式的元素，或者包含仅在某种类型的数据库中可用的元素，则会出现复杂情况。在这些情况下，我们可能会得到一个不符合我们目标数据库正确语法的字符串化语句，或者操作可能会引发一个`UnsupportedCompilationError`异常。在这些情况下，有必要使用`ClauseElement.compile()`方法将语句字符串化，同时传递一个代表目标数据库的`Engine`或`Dialect`对象。例如，如果我们有一个 MySQL 数据库引擎，我们可以按照 MySQL 方言字符串化一个语句：

```py
from sqlalchemy import create_engine

engine = create_engine("mysql+pymysql://scott:tiger@localhost/test")
print(statement.compile(engine))
```

更直接地，不需要构建一个`Engine`对象，我们可以直接实例化一个`Dialect`对象，如下所示，我们使用一个 PostgreSQL 方言：

```py
from sqlalchemy.dialects import postgresql

print(statement.compile(dialect=postgresql.dialect()))
```

请注意，任何方言都可以使用`create_engine()`本身组装，使用一个虚拟 URL，然后访问`Engine.dialect`属性，比如如果我们想要一个 psycopg2 的方言对象：

```py
e = create_engine("postgresql+psycopg2://")
psycopg2_dialect = e.dialect
```

给定一个 ORM `Query`对象时，为了访问`ClauseElement.compile()`方法，我们只需要首先访问`Query.statement`访问器：

```py
statement = query.statement
print(statement.compile(someengine))
```

### 内联渲染绑定参数

警告

**永远**不要使用这些技术处理来自不受信任输入的字符串内容，比如来自 Web 表单或其他用户输入应用程序。SQLAlchemy 将 Python 值强制转换为直接 SQL 字符串值的功能**不安全**，并且不验证传递的数据类型。在针对关系数据库编程调用非 DDL SQL 语句时，始终使用绑定参数。

上述形式将渲染 SQL 语句，因为它被传递到 Python DBAPI，其中包括绑定参数不会内联渲染。SQLAlchemy 通常不会对绑定参数进行字符串化处理，因为这由 Python DBAPI 适当处理，更不用说绕过绑定参数可能是现代 Web 应用中被广泛利用的安全漏洞之一了。SQLAlchemy 在某些情况下有限的能力执行此字符串化，例如发出 DDL 时。为了访问此功能，可以使用传递给`compile_kwargs`的`literal_binds`标志：

```py
from sqlalchemy.sql import table, column, select

t = table("t", column("x"))

s = select(t).where(t.c.x == 5)

# **do not use** with untrusted input!!!
print(s.compile(compile_kwargs={"literal_binds": True}))

# to render for a specific dialect
print(s.compile(dialect=dialect, compile_kwargs={"literal_binds": True}))

# or if you have an Engine, pass as first argument
print(s.compile(some_engine, compile_kwargs={"literal_binds": True}))
```

此功能主要用于日志记录或调试目的，其中获得查询的原始 sql 字符串可能会很有用。

上述方法的注意事项是，它仅支持基本类型，如整数和字符串，而且如果直接使用未设置预设值的`bindparam()`，它也无法对其进行字符串化处理。下面详细介绍了无条件对所有参数进行字符串化的方法。

提示

SQLAlchemy 不支持对所有数据类型进行完全字符串化的原因有三个：

1.  当正常使用 DBAPI 时，该功能已被当前 DBAPI 支持。SQLAlchemy 项目不能被要求为所有后端的所有数据类型重复这种功能，因为这是多余的工作，还带来了重大的测试和持续支持开销。

1.  对于特定数据库的绑定参数进行字符串化建议一种实际上将这些完全字符串化的语句传递给数据库以进行执行的用法。这是不必要和不安全的，SQLAlchemy 不希望以任何方式鼓励这种用法。

1.  渲染字面值的区域是最有可能报告安全问题的地方。SQLAlchemy 尽量将安全参数字符串化的区域留给 DBAPI 驱动程序，这样每个 DBAPI 的具体细节可以得到适当和安全地处理。

由于 SQLAlchemy 故意不支持对所有数据类型的完全字符串化，因此在特定调试场景下执行此操作的技术包括以下内容。作为示例，我们将使用 PostgreSQL 的`UUID`数据类型：

```py
import uuid

from sqlalchemy import Column
from sqlalchemy import create_engine
from sqlalchemy import Integer
from sqlalchemy import select
from sqlalchemy.dialects.postgresql import UUID
from sqlalchemy.orm import declarative_base

Base = declarative_base()

class A(Base):
    __tablename__ = "a"

    id = Column(Integer, primary_key=True)
    data = Column(UUID)

stmt = select(A).where(A.data == uuid.uuid4())
```

给定上述模型和语句，将比较一列与单个 UUID 值，将此语句与内联值一起进行字符串化的选项包括：

+   一些 DBAPI，如 psycopg2，支持像[mogrify()](https://www.psycopg.org/docs/cursor.html#cursor.mogrify)这样的辅助函数，提供对它们的字面渲染功能的访问。要使用此类功能，请渲染 SQL 字符串而不使用`literal_binds`，并通过`SQLCompiler.params`访问器分别传递参数：

    ```py
    e = create_engine("postgresql+psycopg2://scott:tiger@localhost/test")

    with e.connect() as conn:
        cursor = conn.connection.cursor()
        compiled = stmt.compile(e)

        print(cursor.mogrify(str(compiled), compiled.params))
    ```

    上述代码将生成 psycopg2 的原始字节串：

    ```py
    b"SELECT a.id, a.data \nFROM a \nWHERE a.data = 'a511b0fc-76da-4c47-a4b4-716a8189b7ac'::uuid"
    ```

+   直接将`SQLCompiler.params`渲染到语句中，使用目标 DBAPI 的适当[paramstyle](https://www.python.org/dev/peps/pep-0249/#paramstyle)。例如，psycopg2 DBAPI 使用命名的`pyformat`样式。`render_postcompile`的含义将在下一节中讨论。**警告，这是不安全的，请勿使用不受信任的输入**：

    ```py
    e = create_engine("postgresql+psycopg2://")

    # will use pyformat style, i.e. %(paramname)s for param
    compiled = stmt.compile(e, compile_kwargs={"render_postcompile": True})

    print(str(compiled) % compiled.params)
    ```

    这将产生一个非工作的字符串，但仍然适合调试：

    ```py
    SELECT  a.id,  a.data
    FROM  a
    WHERE  a.data  =  9eec1209-50b4-4253-b74b-f82461ed80c1
    ```

    另一个示例使用位置参数风格，如`qmark`，我们可以使用`SQLCompiler.positiontup`集合与`SQLCompiler.params`一起编译我们上面的语句，以便按其位置顺序检索语句的参数：

    ```py
    import re

    e = create_engine("sqlite+pysqlite://")

    # will use qmark style, i.e. ? for param
    compiled = stmt.compile(e, compile_kwargs={"render_postcompile": True})

    # params in positional order
    params = (repr(compiled.params[name]) for name in compiled.positiontup)

    print(re.sub(r"\?", lambda m: next(params), str(compiled)))
    ```

    上述片段打印：

    ```py
    SELECT  a.id,  a.data
    FROM  a
    WHERE  a.data  =  UUID('1bd70375-db17-4d8c-94f1-fc2ef3aada26')
    ```

+   使用自定义 SQL 构造和编译扩展扩展，在用户定义的标志存在时以自定义方式渲染`BindParameter`对象。这个标志通过`compile_kwargs`字典发送，就像任何其他标志一样：

    ```py
    from sqlalchemy.ext.compiler import compiles
    from sqlalchemy.sql.expression import BindParameter

    @compiles(BindParameter)
    def _render_literal_bindparam(element, compiler, use_my_literal_recipe=False, **kw):
        if not use_my_literal_recipe:
            # use normal bindparam processing
            return compiler.visit_bindparam(element, **kw)

        # if use_my_literal_recipe was passed to compiler_kwargs,
        # render the value directly
        return repr(element.value)

    e = create_engine("postgresql+psycopg2://")
    print(stmt.compile(e, compile_kwargs={"use_my_literal_recipe": True}))
    ```

    上述配方将打印：

    ```py
    SELECT  a.id,  a.data
    FROM  a
    WHERE  a.data  =  UUID('47b154cd-36b2-42ae-9718-888629ab9857')
    ```

+   对于内置于模型或语句中的特定类型的字符串化，可以使用`TypeDecorator`类使用`TypeDecorator.process_literal_param()`方法来提供任何数据类型的自定义字符串化：

    ```py
    from sqlalchemy import TypeDecorator

    class UUIDStringify(TypeDecorator):
        impl = UUID

        def process_literal_param(self, value, dialect):
            return repr(value)
    ```

    上述数据类型需要在模型内部明确使用，或者在语句内部使用`type_coerce()`，例如

    ```py
    from sqlalchemy import type_coerce

    stmt = select(A).where(type_coerce(A.data, UUIDStringify) == uuid.uuid4())

    print(stmt.compile(e, compile_kwargs={"literal_binds": True}))
    ```

    再次打印相同形式：

    ```py
    SELECT  a.id,  a.data
    FROM  a
    WHERE  a.data  =  UUID('47b154cd-36b2-42ae-9718-888629ab9857')
    ```

### 将“POSTCOMPILE”参数渲染为绑定参数

SQLAlchemy 包含一个称为 `BindParameter.expanding` 的绑定参数变体，这是一个“延迟评估”的参数，当 SQL 构造编译时以中间状态呈现，然后在语句执行时进一步处理，当传递实际已知值时。默认情况下，通过 `ColumnOperators.in_()` 表达式使用“扩展”参数，以便 SQL 字符串可以安全地独立缓存，而不受传递给 `ColumnOperators.in_()` 的特定调用的实际值列表的影响：

```py
>>> stmt = select(A).where(A.id.in_([1, 2, 3]))
```

若要将 IN 子句呈现为真实的绑定参数符号，请在 `ClauseElement.compile()` 中使用 `render_postcompile=True` 标志：

```py
>>> e = create_engine("postgresql+psycopg2://")
>>> print(stmt.compile(e, compile_kwargs={"render_postcompile": True}))
SELECT  a.id,  a.data
FROM  a
WHERE  a.id  IN  (%(id_1_1)s,  %(id_1_2)s,  %(id_1_3)s) 
```

在关于呈现绑定参数的前一节中描述的 `literal_binds` 标志会自动将 `render_postcompile` 设置为 True，因此对于带有简单整数/字符串的语句，这些可以直接转换为字符串：

```py
# render_postcompile is implied by literal_binds
>>> print(stmt.compile(e, compile_kwargs={"literal_binds": True}))
SELECT  a.id,  a.data
FROM  a
WHERE  a.id  IN  (1,  2,  3) 
```

`SQLCompiler.params` 和 `SQLCompiler.positiontup` 也与 `render_postcompile` 兼容，因此在这里以相同的方式工作，例如 SQLite 的位置形式：

```py
>>> u1, u2, u3 = uuid.uuid4(), uuid.uuid4(), uuid.uuid4()
>>> stmt = select(A).where(A.data.in_([u1, u2, u3]))

>>> import re
>>> e = create_engine("sqlite+pysqlite://")
>>> compiled = stmt.compile(e, compile_kwargs={"render_postcompile": True})
>>> params = (repr(compiled.params[name]) for name in compiled.positiontup)
>>> print(re.sub(r"\?", lambda m: next(params), str(compiled)))
SELECT  a.id,  a.data
FROM  a
WHERE  a.data  IN  (UUID('aa1944d6-9a5a-45d5-b8da-0ba1ef0a4f38'),  UUID('a81920e6-15e2-4392-8a3c-d775ffa9ccd2'),  UUID('b5574cdb-ff9b-49a3-be52-dbc89f087bfa')) 
```

警告

请记住，**所有**上述代码配方都是用于字符串化字面值，在将语句发送到数据库时绕过绑定参数的情况下，仅适用于：

1.  使用仅限于**调试目的**

1.  字符串**不应传递到活动的生产数据库**

1.  仅与**本地、可信赖的输入**一起使用

上述用于字符串化字面值的配方在任何情况下都**不安全**，绝不应该用于生产数据库。## 字符串化 SQL 语句时为什么要双倍百分号？

许多 DBAPI 实现使用 `pyformat` 或 `format` [paramstyle](https://www.python.org/dev/peps/pep-0249/#paramstyle)，其语法中必然涉及百分号。大多数这样做的 DBAPI 期望在用于语句的字符串形式中，百分号用于其他目的时应该是双倍的（即转义），例如：

```py
SELECT  a,  b  FROM  some_table  WHERE  a  =  %s  AND  c  =  %s  AND  num  %%  modulus  =  0
```

当 SQL 语句由 SQLAlchemy 传递给底层的 DBAPI 时，绑定参数的替换方式与 Python 字符串插值运算符 `%` 相同，在许多情况下，DBAPI 实际上直接使用此运算符。以上，绑定参数的替换看起来像：

```py
SELECT  a,  b  FROM  some_table  WHERE  a  =  5  AND  c  =  10  AND  num  %  modulus  =  0
```

像 PostgreSQL（默认 DBAPI 是 psycopg2）和 MySQL（默认 DBAPI 是 mysqlclient）这样的数据库的默认编译器将具有百分号转义行为：

```py
>>> from sqlalchemy import table, column
>>> from sqlalchemy.dialects import postgresql
>>> t = table("my_table", column("value % one"), column("value % two"))
>>> print(t.select().compile(dialect=postgresql.dialect()))
SELECT  my_table."value %% one",  my_table."value %% two"
FROM  my_table 
```

当使用此类方言时，如果需要非 DBAPI 语句，而这些语句不包括绑定的参数符号，则可通过直接使用 Python 的 `%` 运算符来简单地替换空参数集来删除百分号：

```py
>>> strstmt = str(t.select().compile(dialect=postgresql.dialect()))
>>> print(strstmt % ())
SELECT  my_table."value % one",  my_table."value % two"
FROM  my_table 
```

另一种方法是在使用的方言上设置不同的参数样式；所有 `Dialect` 实现都接受一个参数 `paramstyle`，将导致该方言的编译器使用给定的参数样式。下面，在用于编译的方言中设置了非常常见的 `named` 参数样式，以便百分号在 SQL 的编译形式中不再具有重要意义，并且将不再被转义：

```py
>>> print(t.select().compile(dialect=postgresql.dialect(paramstyle="named")))
SELECT  my_table."value % one",  my_table."value % two"
FROM  my_table 
```  ## 我使用 op() 来生成自定义操作符，但是我的括号没有正确显示

`Operators.op()` 方法允许创建自定义数据库操作符，否则 SQLAlchemy 不会识别：

```py
>>> print(column("q").op("->")(column("p")))
q  ->  p 
```

但是，当在复合表达式的右侧使用时，它不会按我们的预期生成括号：

```py
>>> print((column("q1") + column("q2")).op("->")(column("p")))
q1  +  q2  ->  p 
```

在上面的情况下，我们可能希望 `(q1 + q2) -> p`。

对于此情况的解决方案是设置操作符的优先级，使用 `Operators.op.precedence` 参数，将其设置为一个较高的数字，其中 `100` 是最大值，而 SQLAlchemy 当前使用的任何操作符的最高数字为 `15`：

```py
>>> print((column("q1") + column("q2")).op("->", precedence=100)(column("p")))
(q1  +  q2)  ->  p 
```

我们还可以使用 `ColumnElement.self_group()` 方法通常强制将二元表达式（例如具有左/右操作数和运算符的表达式）括在括号中：

```py
>>> print((column("q1") + column("q2")).self_group().op("->")(column("p")))
(q1  +  q2)  ->  p 
```

### 为什么括号规则是这样的？

当存在过多的括号或括号处于它们不期望的不寻常位置时，很多数据库都会出现问题，因此 SQLAlchemy 不会基于分组生成括号，它使用操作符优先级，如果操作符已知是可结合的，那么生成的括号将最小化。否则，像下面这样的表达式：

```py
column("a") & column("b") & column("c") & column("d")
```

将产生：

```py
(((a  AND  b)  AND  c)  AND  d)
```

这是可以接受的，但可能会让人们感到恼火（并被报告为错误）。在其他情况下，它会导致更容易混淆数据库或至少可读性更差的事物，例如：

```py
column("q", ARRAY(Integer, dimensions=2))[5][6]
```

将产生：

```py
((q[5])[6])
```

也有一些边缘情况，我们会得到类似`"(x) = 7"`这样的东西，数据库真的不喜欢这样。所以括号化并不是简单地加括号，它使用运算符优先级和结合性来确定分组。

对于`Operators.op()`，优先级的值默认为零。

如果我们将`Operators.op.precedence`的值默认为 100，例如最高值，会怎么样？然后这个表达式会加更多括号，但其他方面都没问题，也就是说，这两个是等价的：

```py
>>> print((column("q") - column("y")).op("+", precedence=100)(column("z")))
(q  -  y)  +  z
>>> print((column("q") - column("y")).op("+")(column("z")))
q  -  y  +  z 
```

但这两个不是：

```py
>>> print(column("q") - column("y").op("+", precedence=100)(column("z")))
q  -  y  +  z
>>> print(column("q") - column("y").op("+")(column("z")))
q  -  (y  +  z) 
```

目前，尚不清楚只要我们根据运算符优先级和结合性进行括号化，是否真的有一种方法可以自动为没有给定优先级的通用运算符进行括号化，以使其在所有情况下都有效，因为有时您希望自定义运算符具有比其他运算符更低的优先级，有时您希望它更高。

可能如果上面的“二元”表达式在调用`op()`时强制使用`self_group()`方法，假设左侧的复合表达式总是可以无害地加括号。也许这种改变可以在某个时候实现，但是目前保持括号化规则更加内部一致似乎是更安全的方法。  ## 如何将 SQL 表达式呈现为字符串，可能包含内联的绑定参数？

在大多数简单情况下，将 SQLAlchemy Core 语句对象或表达式片段以及 ORM `Query` 对象“字符串化”，就像在使用`str()`内置函数时一样简单，如下所示，当与`print`函数一起使用时（请注意 Python 的`print`函数如果我们不显式使用`str()`，也会自动调用它）：

```py
>>> from sqlalchemy import table, column, select
>>> t = table("my_table", column("x"))
>>> statement = select(t)
>>> print(str(statement))
SELECT  my_table.x
FROM  my_table 
```

内置函数`str()`，或者等效函数，可以在 ORM `Query` 对象上调用，也可以在任何语句上调用，比如`select()`，`insert()`等，以及任何表达式片段，比如：

```py
>>> from sqlalchemy import column
>>> print(column("x") == "some value")
x  =  :x_1 
```

### 针对特定数据库的字符串化

当我们要字符串化的语句或片段包含具有数据库特定字符串格式的元素，或者包含仅在某种类型的数据库中可用的元素时，会出现一个复杂性。在这些情况下，我们可能会得到一个不符合我们所针对的数据库的正确语法的字符串化语句，或者该操作可能会引发一个`UnsupportedCompilationError`异常。在这些情况下，必须使用`ClauseElement.compile()`方法对语句进行字符串化，同时传递一个表示目标数据库的`Engine`或`Dialect`对象。例如，如果我们有一个 MySQL 数据库引擎，我们可以如下将语句字符串化为 MySQL 方言：

```py
from sqlalchemy import create_engine

engine = create_engine("mysql+pymysql://scott:tiger@localhost/test")
print(statement.compile(engine))
```

更直接地，不需要构建`Engine`对象，我们可以直接实例化一个`Dialect`对象，如下所示，我们使用 PostgreSQL 方言：

```py
from sqlalchemy.dialects import postgresql

print(statement.compile(dialect=postgresql.dialect()))
```

请注意，可以使用`create_engine()`本身来组装任何方言，只需使用一个虚拟 URL 并访问`Engine.dialect`属性即可，例如，如果我们想要 psycopg2 的方言对象：

```py
e = create_engine("postgresql+psycopg2://")
psycopg2_dialect = e.dialect
```

给定一个 ORM `Query` 对象，为了获取`ClauseElement.compile()`方法，我们只需要先访问`Query.statement`访问器：

```py
statement = query.statement
print(statement.compile(someengine))
```

### 将绑定参数嵌入渲染

警告

**永远**不要将这些技术与来自不受信任输入的字符串内容一起使用，例如来自 Web 表单或其他用户输入应用程序。SQLAlchemy 将 Python 值强制转换为直接 SQL 字符串值的设施**不安全**，不安全地针对不受信任的输入，并且不验证传递的数据类型。在针对关系数据库程序化地调用非 DDL SQL 语句时，始终使用绑定参数。

上述形式将呈现 SQL 语句，就像它传递给 Python DBAPI 一样，其中绑定参数不会被内联呈现。SQLAlchemy 通常不会将绑定参数字符串化，因为这由 Python DBAPI 适当处理，更不用说绕过绑定参数可能是现代 Web 应用程序中最广泛利用的安全漏洞之一。SQLAlchemy 在某些情况下有限地能够执行此字符串化，比如发出 DDL。为了访问此功能，可以使用传递给 `compile_kwargs` 的 `literal_binds` 标志：

```py
from sqlalchemy.sql import table, column, select

t = table("t", column("x"))

s = select(t).where(t.c.x == 5)

# **do not use** with untrusted input!!!
print(s.compile(compile_kwargs={"literal_binds": True}))

# to render for a specific dialect
print(s.compile(dialect=dialect, compile_kwargs={"literal_binds": True}))

# or if you have an Engine, pass as first argument
print(s.compile(some_engine, compile_kwargs={"literal_binds": True}))
```

此功能主要用于记录或调试目的，其中查询的原始 SQL 字符串可能会证明有用。

上述方法的注意事项是它仅支持基本类型，如整数和字符串，而且如果直接使用没有预设值的 `bindparam()`，它也无法将其字符串化。无条件地将所有参数字符串化的方法如下所述。

提示

SQLAlchemy 不支持所有数据类型的完全字符串化的原因有三个：

1.  当正常使用 DBAPI 时，这是已经受支持的功能。SQLAlchemy 项目无法被要求为所有后端的每种数据类型复制这种功能，因为这是多余的工作，还会带来重大的测试和持续支持开销。

1.  使用内联绑定参数进行字符串化，针对特定数据库，表明了一种实际将这些完全字符串化的语句传递到数据库执行的用法。这是不必要且不安全的，SQLAlchemy 不希望以任何方式鼓励这种用法。

1.  渲染字面值的领域是最有可能报告安全问题的领域。SQLAlchemy 尽量将安全参数字符串化的问题留给 DBAPI 驱动程序处理，其中每个 DBAPI 的具体情况可以得到适当和安全的处理。

由于 SQLAlchemy 故意不支持对字面值的完全字符串化，因此在特定调试场景中执行此操作的技术包括以下内容。作为示例，我们将使用 PostgreSQL 的 `UUID` 数据类型：

```py
import uuid

from sqlalchemy import Column
from sqlalchemy import create_engine
from sqlalchemy import Integer
from sqlalchemy import select
from sqlalchemy.dialects.postgresql import UUID
from sqlalchemy.orm import declarative_base

Base = declarative_base()

class A(Base):
    __tablename__ = "a"

    id = Column(Integer, primary_key=True)
    data = Column(UUID)

stmt = select(A).where(A.data == uuid.uuid4())
```

鉴于上述模型和语句，将比较列与单个 UUID 值，将此语句与内联值字符串化的选项包括：

+   一些 DBAPI，如 psycopg2，支持像 [mogrify()](https://www.psycopg.org/docs/cursor.html#cursor.mogrify) 这样的辅助函数，提供对它们的字面渲染功能的访问。要使用这些功能，渲染 SQL 字符串时不要使用 `literal_binds`，并通过 `SQLCompiler.params` 访问器单独传递参数：

    ```py
    e = create_engine("postgresql+psycopg2://scott:tiger@localhost/test")

    with e.connect() as conn:
        cursor = conn.connection.cursor()
        compiled = stmt.compile(e)

        print(cursor.mogrify(str(compiled), compiled.params))
    ```

    上述代码将生成 psycopg2 的原始字节串：

    ```py
    b"SELECT a.id, a.data \nFROM a \nWHERE a.data = 'a511b0fc-76da-4c47-a4b4-716a8189b7ac'::uuid"
    ```

+   直接将`SQLCompiler.params`渲染到语句中，使用目标 DBAPI 的适当[paramstyle](https://www.python.org/dev/peps/pep-0249/#paramstyle)。例如，psycopg2 DBAPI 使用命名的`pyformat`样式。`render_postcompile`的含义将在下一节中讨论。**警告这不安全，请勿使用不受信任的输入**：

    ```py
    e = create_engine("postgresql+psycopg2://")

    # will use pyformat style, i.e. %(paramname)s for param
    compiled = stmt.compile(e, compile_kwargs={"render_postcompile": True})

    print(str(compiled) % compiled.params)
    ```

    这将产生一个非工作的字符串，但适合用于调试：

    ```py
    SELECT  a.id,  a.data
    FROM  a
    WHERE  a.data  =  9eec1209-50b4-4253-b74b-f82461ed80c1
    ```

    另一个例子是使用位置参数风格，例如`qmark`，我们可以结合使用`SQLCompiler.positiontup`集合和`SQLCompiler.params`来在 SQLite 中呈现上述语句，以便按照编译后的顺序检索参数：

    ```py
    import re

    e = create_engine("sqlite+pysqlite://")

    # will use qmark style, i.e. ? for param
    compiled = stmt.compile(e, compile_kwargs={"render_postcompile": True})

    # params in positional order
    params = (repr(compiled.params[name]) for name in compiled.positiontup)

    print(re.sub(r"\?", lambda m: next(params), str(compiled)))
    ```

    上述片段打印：

    ```py
    SELECT  a.id,  a.data
    FROM  a
    WHERE  a.data  =  UUID('1bd70375-db17-4d8c-94f1-fc2ef3aada26')
    ```

+   当存在用户定义的标志时，使用自定义 SQL 构造和编译扩展扩展以自定义方式呈现`BindParameter`对象。此标志通过`compile_kwargs`字典像其他标志一样发送：

    ```py
    from sqlalchemy.ext.compiler import compiles
    from sqlalchemy.sql.expression import BindParameter

    @compiles(BindParameter)
    def _render_literal_bindparam(element, compiler, use_my_literal_recipe=False, **kw):
        if not use_my_literal_recipe:
            # use normal bindparam processing
            return compiler.visit_bindparam(element, **kw)

        # if use_my_literal_recipe was passed to compiler_kwargs,
        # render the value directly
        return repr(element.value)

    e = create_engine("postgresql+psycopg2://")
    print(stmt.compile(e, compile_kwargs={"use_my_literal_recipe": True}))
    ```

    上述配方将打印：

    ```py
    SELECT  a.id,  a.data
    FROM  a
    WHERE  a.data  =  UUID('47b154cd-36b2-42ae-9718-888629ab9857')
    ```

+   用于内置于模型或语句的特定类型字符串化的`TypeDecorator`类可使用`TypeDecorator.process_literal_param()`方法来提供任何数据类型的自定义字符串化：

    ```py
    from sqlalchemy import TypeDecorator

    class UUIDStringify(TypeDecorator):
        impl = UUID

        def process_literal_param(self, value, dialect):
            return repr(value)
    ```

    上述数据类型需要在模型内明确使用或在语句内部使用`type_coerce()`，例如

    ```py
    from sqlalchemy import type_coerce

    stmt = select(A).where(type_coerce(A.data, UUIDStringify) == uuid.uuid4())

    print(stmt.compile(e, compile_kwargs={"literal_binds": True}))
    ```

    再次打印相同形式：

    ```py
    SELECT  a.id,  a.data
    FROM  a
    WHERE  a.data  =  UUID('47b154cd-36b2-42ae-9718-888629ab9857')
    ```

### 将“POSTCOMPILE”参数呈现为绑定参数

SQLAlchemy 包括一个变体绑定参数，称为 `BindParameter.expanding`，它是一个“延迟评估”的参数，在编译 SQL 构造时呈现为中间状态，然后在语句执行时进一步处理，当实际已知值传递时。 “扩展”参数默认用于 `ColumnOperators.in_()` 表达式，以便 SQL 字符串可以安全地独立于传递给 `ColumnOperators.in_()` 的特定值列表进行缓存：

```py
>>> stmt = select(A).where(A.id.in_([1, 2, 3]))
```

要使用实际的绑定参数符号呈现 IN 子句，请在 `ClauseElement.compile()` 中使用 `render_postcompile=True` 标志：

```py
>>> e = create_engine("postgresql+psycopg2://")
>>> print(stmt.compile(e, compile_kwargs={"render_postcompile": True}))
SELECT  a.id,  a.data
FROM  a
WHERE  a.id  IN  (%(id_1_1)s,  %(id_1_2)s,  %(id_1_3)s) 
```

前一节中关于渲染绑定参数的 `literal_binds` 标志自动将 `render_postcompile` 设置为 True，因此对于具有简单整数/字符串的语句，可以直接进行字符串化：

```py
# render_postcompile is implied by literal_binds
>>> print(stmt.compile(e, compile_kwargs={"literal_binds": True}))
SELECT  a.id,  a.data
FROM  a
WHERE  a.id  IN  (1,  2,  3) 
```

`SQLCompiler.params` 和 `SQLCompiler.positiontup` 也与 `render_postcompile` 兼容，因此以前的渲染内联绑定参数的方法在这里也可以正常工作，例如 SQLite 的位置形式：

```py
>>> u1, u2, u3 = uuid.uuid4(), uuid.uuid4(), uuid.uuid4()
>>> stmt = select(A).where(A.data.in_([u1, u2, u3]))

>>> import re
>>> e = create_engine("sqlite+pysqlite://")
>>> compiled = stmt.compile(e, compile_kwargs={"render_postcompile": True})
>>> params = (repr(compiled.params[name]) for name in compiled.positiontup)
>>> print(re.sub(r"\?", lambda m: next(params), str(compiled)))
SELECT  a.id,  a.data
FROM  a
WHERE  a.data  IN  (UUID('aa1944d6-9a5a-45d5-b8da-0ba1ef0a4f38'),  UUID('a81920e6-15e2-4392-8a3c-d775ffa9ccd2'),  UUID('b5574cdb-ff9b-49a3-be52-dbc89f087bfa')) 
```

警告

请记住，**所有**上述代码示例，用于将字面值字符串化，将语句发送到数据库时绕过绑定参数的使用，**仅在以下情况下使用**：

1.  仅用于**调试目的**。

1.  字符串**不应传递给生产数据库**。

1.  仅用于**本地、可信的输入**。

上述对字面值字符串化的方法**在任何情况下都不安全，绝不应该用于生产数据库**。

### 针对特定数据库的字符串化

当我们要将要串化的语句或片段包含有特定于数据库的字符串格式的元素，或者当它包含有仅在某种类型的数据库中可用的元素时，就会出现一些复杂情况。在这些情况下，我们可能会得到一个串化的语句，该语句不符合我们所针对的数据库的正确语法，或者该操作可能会引发一个 `UnsupportedCompilationError` 异常。在这些情况下，有必要使用 `ClauseElement.compile()` 方法串化该语句，同时传递一个代表目标数据库的 `Engine` 或 `Dialect` 对象。如下，如果我们有一个 MySQL 数据库引擎，我们可以根据 MySQL 方言串化一个语句：

```py
from sqlalchemy import create_engine

engine = create_engine("mysql+pymysql://scott:tiger@localhost/test")
print(statement.compile(engine))
```

更直接地，我们可以在不构建 `Engine` 对象的情况下直接实例化一个 `Dialect` 对象，如下所示，我们使用了一个 PostgreSQL 方言：

```py
from sqlalchemy.dialects import postgresql

print(statement.compile(dialect=postgresql.dialect()))
```

注意，任何方言都可以使用 `create_engine()` 方法与一个虚拟 URL 配合组装，然后访问 `Engine.dialect` 属性，比如说如果我们想要一个 psycopg2 的方言对象：

```py
e = create_engine("postgresql+psycopg2://")
psycopg2_dialect = e.dialect
```

当给定一个 ORM `Query` 对象时，为了获取到 `ClauseElement.compile()` 方法，我们只需要先访问 `Query.statement` 属性：

```py
statement = query.statement
print(statement.compile(someengine))
```

### 将绑定参数内联渲染

警告

**永远**不要使用这些技术处理来自不受信任输入的字符串内容，比如来自网络表单或其他用户输入应用程序。SQLAlchemy 将 Python 值强制转换为直接的 SQL 字符串值的能力**不安全且不验证传递的数据类型**。在针对关系数据库进行非 DDL SQL 语句的编程调用时，始终使用绑定参数。

上述形式将渲染传递给 Python DBAPI 的 SQL 语句，其中包括绑定参数不会内联渲染。SQLAlchemy 通常不会字符串化绑定参数，因为这由 Python DBAPI 适当处理，更不用说绕过绑定参数可能是现代 Web 应用程序中被广泛利用的安全漏洞之一。SQLAlchemy 在某些情况下（如发出 DDL）有限地执行此字符串化。为了访问此功能，可以使用传递给 `compile_kwargs` 的 `literal_binds` 标志：

```py
from sqlalchemy.sql import table, column, select

t = table("t", column("x"))

s = select(t).where(t.c.x == 5)

# **do not use** with untrusted input!!!
print(s.compile(compile_kwargs={"literal_binds": True}))

# to render for a specific dialect
print(s.compile(dialect=dialect, compile_kwargs={"literal_binds": True}))

# or if you have an Engine, pass as first argument
print(s.compile(some_engine, compile_kwargs={"literal_binds": True}))
```

此功能主要用于日志记录或调试目的，其中查询的原始 SQL 字符串可能会证明有用。

上述方法的注意事项是，它仅支持基本类型，如整数和字符串，而且如果直接使用没有预设值的 `bindparam()`，它也无法将其字符串化。在下面详细描述了无条件字符串化所有参数的方法。

提示

SQLAlchemy 不支持所有数据类型的完全字符串化的原因有三：

1.  当正常使用 DBAPI 时，已经支持此功能。SQLAlchemy 项目不能被要求为所有后端的每种数据类型复制此功能，因为这是多余的工作，还会产生重大的测试和持续支持开销。

1.  对于特定数据库，将边界参数内联化字符串化建议使用实际将这些完全字符串化的语句传递给数据库执行。这是不必要且不安全的，SQLAlchemy 不希望以任何方式鼓励这种用法。

1.  字面值渲染领域是最有可能报告安全问题的领域。SQLAlchemy 尽量使安全参数字符串化领域成为 DBAPI 驱动程序的问题，其中每个 DBAPI 的具体情况都可以得到适当和安全地处理。

由于 SQLAlchemy 故意不支持对字面值的完全字符串化，因此在特定调试场景中进行这样的技术包括以下内容。例如，我们将使用 PostgreSQL 的 `UUID` 数据类型：

```py
import uuid

from sqlalchemy import Column
from sqlalchemy import create_engine
from sqlalchemy import Integer
from sqlalchemy import select
from sqlalchemy.dialects.postgresql import UUID
from sqlalchemy.orm import declarative_base

Base = declarative_base()

class A(Base):
    __tablename__ = "a"

    id = Column(Integer, primary_key=True)
    data = Column(UUID)

stmt = select(A).where(A.data == uuid.uuid4())
```

针对以上模型和语句将比较一列与单个 UUID 值的情况，使用内联值对该语句进行字符串化的选项包括：

+   一些 DBAPI（如 psycopg2）支持像 [mogrify()](https://www.psycopg.org/docs/cursor.html#cursor.mogrify) 这样的辅助函数，提供对它们的字面值渲染功能的访问。要使用这些特性，渲染 SQL 字符串时不要使用 `literal_binds`，而是通过 `SQLCompiler.params` 访问器分别传递参数：

    ```py
    e = create_engine("postgresql+psycopg2://scott:tiger@localhost/test")

    with e.connect() as conn:
        cursor = conn.connection.cursor()
        compiled = stmt.compile(e)

        print(cursor.mogrify(str(compiled), compiled.params))
    ```

    上述代码将产生 psycopg2 的原始字节字符串：

    ```py
    b"SELECT a.id, a.data \nFROM a \nWHERE a.data = 'a511b0fc-76da-4c47-a4b4-716a8189b7ac'::uuid"
    ```

+   直接将 `SQLCompiler.params` 渲染到语句中，使用目标 DBAPI 的适当 [paramstyle](https://www.python.org/dev/peps/pep-0249/#paramstyle)。例如，psycopg2 DBAPI 使用命名的 `pyformat` 样式。 `render_postcompile` 的含义将在下一节中讨论。 **警告：这不安全，请不要使用不受信任的输入**：

    ```py
    e = create_engine("postgresql+psycopg2://")

    # will use pyformat style, i.e. %(paramname)s for param
    compiled = stmt.compile(e, compile_kwargs={"render_postcompile": True})

    print(str(compiled) % compiled.params)
    ```

    这将产生一个无效的字符串，尽管它适用于调试：

    ```py
    SELECT  a.id,  a.data
    FROM  a
    WHERE  a.data  =  9eec1209-50b4-4253-b74b-f82461ed80c1
    ```

    另一个示例使用了位置参数风格，如 `qmark`，我们还可以使用 `SQLCompiler.positiontup` 集合与 `SQLCompiler.params` 结合使用，以便按编译后的语句中的位置顺序检索参数：

    ```py
    import re

    e = create_engine("sqlite+pysqlite://")

    # will use qmark style, i.e. ? for param
    compiled = stmt.compile(e, compile_kwargs={"render_postcompile": True})

    # params in positional order
    params = (repr(compiled.params[name]) for name in compiled.positiontup)

    print(re.sub(r"\?", lambda m: next(params), str(compiled)))
    ```

    上述代码段将打印：

    ```py
    SELECT  a.id,  a.data
    FROM  a
    WHERE  a.data  =  UUID('1bd70375-db17-4d8c-94f1-fc2ef3aada26')
    ```

+   当存在用户定义的标志时，使用 自定义 SQL 构造和编译扩展 扩展以自定义方式渲染 `BindParameter` 对象。此标志通过 `compile_kwargs` 字典发送，就像发送任何其他标志一样：

    ```py
    from sqlalchemy.ext.compiler import compiles
    from sqlalchemy.sql.expression import BindParameter

    @compiles(BindParameter)
    def _render_literal_bindparam(element, compiler, use_my_literal_recipe=False, **kw):
        if not use_my_literal_recipe:
            # use normal bindparam processing
            return compiler.visit_bindparam(element, **kw)

        # if use_my_literal_recipe was passed to compiler_kwargs,
        # render the value directly
        return repr(element.value)

    e = create_engine("postgresql+psycopg2://")
    print(stmt.compile(e, compile_kwargs={"use_my_literal_recipe": True}))
    ```

    上述示例将打印：

    ```py
    SELECT  a.id,  a.data
    FROM  a
    WHERE  a.data  =  UUID('47b154cd-36b2-42ae-9718-888629ab9857')
    ```

+   对于内置于模型或语句中的特定类型字符串化，可以使用 `TypeDecorator` 类来使用 `TypeDecorator.process_literal_param()` 方法提供任何数据类型的自定义字符串化：

    ```py
    from sqlalchemy import TypeDecorator

    class UUIDStringify(TypeDecorator):
        impl = UUID

        def process_literal_param(self, value, dialect):
            return repr(value)
    ```

    上述数据类型需要在模型内或在语句中本地使用 `type_coerce()` 明确使用，例如

    ```py
    from sqlalchemy import type_coerce

    stmt = select(A).where(type_coerce(A.data, UUIDStringify) == uuid.uuid4())

    print(stmt.compile(e, compile_kwargs={"literal_binds": True}))
    ```

    再次打印相同的形式：

    ```py
    SELECT  a.id,  a.data
    FROM  a
    WHERE  a.data  =  UUID('47b154cd-36b2-42ae-9718-888629ab9857')
    ```

### 将 “POSTCOMPILE” 参数呈现为绑定参数

SQLAlchemy 包含一个称为`BindParameter.expanding`的绑定参数变体，这是一个“延迟评估”的参数，当编译 SQL 结构时以中间状态呈现，然后在语句执行时进一步处理，当实际已知值被传递时。 “扩展”参数默认用于`ColumnOperators.in_()`表达式，以便 SQL 字符串可以安全地独立于传递给`ColumnOperators.in_()`的特定调用的实际值被缓存：

```py
>>> stmt = select(A).where(A.id.in_([1, 2, 3]))
```

要使用实际的绑定参数符号呈现 IN 子句，请在`ClauseElement.compile()`中使用`render_postcompile=True`标志：

```py
>>> e = create_engine("postgresql+psycopg2://")
>>> print(stmt.compile(e, compile_kwargs={"render_postcompile": True}))
SELECT  a.id,  a.data
FROM  a
WHERE  a.id  IN  (%(id_1_1)s,  %(id_1_2)s,  %(id_1_3)s) 
```

在先前有关渲染绑定参数的部分中描述的`literal_binds`标志会自动将`render_postcompile`设置为 True，因此对于具有简单 int/字符串的语句，可以直接将它们字符串化：

```py
# render_postcompile is implied by literal_binds
>>> print(stmt.compile(e, compile_kwargs={"literal_binds": True}))
SELECT  a.id,  a.data
FROM  a
WHERE  a.id  IN  (1,  2,  3) 
```

`SQLCompiler.params`和`SQLCompiler.positiontup`与`render_postcompile`兼容，因此在此处渲染内联绑定参数的先前方法也将以相同的方式工作，例如 SQLite 的位置形式：

```py
>>> u1, u2, u3 = uuid.uuid4(), uuid.uuid4(), uuid.uuid4()
>>> stmt = select(A).where(A.data.in_([u1, u2, u3]))

>>> import re
>>> e = create_engine("sqlite+pysqlite://")
>>> compiled = stmt.compile(e, compile_kwargs={"render_postcompile": True})
>>> params = (repr(compiled.params[name]) for name in compiled.positiontup)
>>> print(re.sub(r"\?", lambda m: next(params), str(compiled)))
SELECT  a.id,  a.data
FROM  a
WHERE  a.data  IN  (UUID('aa1944d6-9a5a-45d5-b8da-0ba1ef0a4f38'),  UUID('a81920e6-15e2-4392-8a3c-d775ffa9ccd2'),  UUID('b5574cdb-ff9b-49a3-be52-dbc89f087bfa')) 
```

警告

请记住，**所有**上述字符串化文字值的代码示例，当将语句发送到数据库时绕过绑定参数的使用，**只能在以下情况下使用**：

1.  仅用于**调试目的**

1.  该字符串**不应传递给实时生产数据库**

1.  仅限于**本地，可信任的输入**

上述用于将文字值字符串化的方法**在任何情况下都不安全，绝对不应该用于生产数据库**。

## 为什么在将 SQL 语句字符串化时百分号会被加倍？

许多 DBAPI 实现采用`pyformat`或`format` [paramstyle](https://www.python.org/dev/peps/pep-0249/#paramstyle)，这在其语法中必然涉及百分号。这样做的大多数 DBAPI 都希望在使用的语句的字符串形式中，用于其他目的的百分号被双倍化（即转义），例如：

```py
SELECT  a,  b  FROM  some_table  WHERE  a  =  %s  AND  c  =  %s  AND  num  %%  modulus  =  0
```

当 SQL 语句通过 SQLAlchemy 传递给底层 DBAPI 时，绑定参数的替换方式与 Python 字符串插值运算符`%`相同，在许多情况下，DBAPI 实际上直接使用这个运算符。上面，绑定参数的替换看起来像是：

```py
SELECT  a,  b  FROM  some_table  WHERE  a  =  5  AND  c  =  10  AND  num  %  modulus  =  0
```

像 PostgreSQL（默认 DBAPI 是 psycopg2）和 MySQL（默认 DBAPI 是 mysqlclient）这样的数据库的默认编译器将具有这种百分号转义行为：

```py
>>> from sqlalchemy import table, column
>>> from sqlalchemy.dialects import postgresql
>>> t = table("my_table", column("value % one"), column("value % two"))
>>> print(t.select().compile(dialect=postgresql.dialect()))
SELECT  my_table."value %% one",  my_table."value %% two"
FROM  my_table 
```

当使用这样的方言时，如果需要不包含绑定参数符号的非 DBAPI 语句，一种快速删除百分号的方法是直接使用 Python 的`%`运算符替换一个空的参数集：

```py
>>> strstmt = str(t.select().compile(dialect=postgresql.dialect()))
>>> print(strstmt % ())
SELECT  my_table."value % one",  my_table."value % two"
FROM  my_table 
```

另一种方法是在使用的方言上设置不同的参数样式；所有`Dialect`实现都接受一个`paramstyle`参数，该参数将导致该方言的编译器使用给定的参数样式。下面，非常常见的`named`参数样式在用于编译的方言中设置，以便百分号在 SQL 的编译形式中不再重要，并且不再被转义：

```py
>>> print(t.select().compile(dialect=postgresql.dialect(paramstyle="named")))
SELECT  my_table."value % one",  my_table."value % two"
FROM  my_table 
```

## 我正在使用 op()生成自定义运算符，但我的括号没出来正确

`Operators.op()`方法允许创建一个 SQLAlchemy 中未知的自定义数据库操作符：

```py
>>> print(column("q").op("->")(column("p")))
q  ->  p 
```

然而，当将其用于复合表达式的右侧时，它不会生成我们期望的括号：

```py
>>> print((column("q1") + column("q2")).op("->")(column("p")))
q1  +  q2  ->  p 
```

在上面的情况下，我们可能想要`(q1 + q2) -> p`。

对于这种情况的解决方案是设置运算符的优先级，使用`Operators.op.precedence`参数，设置为一个高数字，其中 100 是最大值，当前任何 SQLAlchemy 运算符使用的最高数字是 15：

```py
>>> print((column("q1") + column("q2")).op("->", precedence=100)(column("p")))
(q1  +  q2)  ->  p 
```

我们还可以通常通过使用`ColumnElement.self_group()`方法强制在二元表达式（例如具有左/右操作数和运算符的表达式）周围加上括号：

```py
>>> print((column("q1") + column("q2")).self_group().op("->")(column("p")))
(q1  +  q2)  ->  p 
```

### 为什么括号的规则是这样的？

当存在过多的括号或者括号处于数据库不期望的不寻常位置时，许多数据库会报错，因此 SQLAlchemy 不基于分组生成括号，它使用操作符优先级以及如果操作符已知是可结合的，则生成最小的括号。否则，表达式如下：

```py
column("a") & column("b") & column("c") & column("d")
```

将产生：

```py
(((a  AND  b)  AND  c)  AND  d)
```

这样做可能会让人们感到不爽（并被报告为错误）。在其他情况下，它会导致更容易让数据库混淆或至少降低可读性，比如：

```py
column("q", ARRAY(Integer, dimensions=2))[5][6]
```

将产生：

```py
((q[5])[6])
```

还有一些边界情况，我们会得到像`"(x) = 7"`这样的东西，数据库真的不喜欢这样。因此，括号化不是简单地添加括号，而是使用运算符优先级和结合性来确定分组。

对于`Operators.op()`，优先级的值默认为零。

如果我们将`Operators.op.precedence`的值默认为 100，即最高值，会怎样呢？然后这个表达式会多加括号，但除此之外还是可以的，也就是说，这两个表达式是等价的：

```py
>>> print((column("q") - column("y")).op("+", precedence=100)(column("z")))
(q  -  y)  +  z
>>> print((column("q") - column("y")).op("+")(column("z")))
q  -  y  +  z 
```

但是这两种情况不是：

```py
>>> print(column("q") - column("y").op("+", precedence=100)(column("z")))
q  -  y  +  z
>>> print(column("q") - column("y").op("+")(column("z")))
q  -  (y  +  z) 
```

目前来看，只要我们根据运算符的优先级和结合性进行括号化，如果真的有一种方法可以自动为没有给定优先级的通用运算符进行括号化，从而在所有情况下都能正常工作，这还不清楚，因为有时您希望自定义的运算符具有比其他运算符更低的优先级，有时您希望它更高。

如果上面的“binary”表达式强制在调用`op()`时使用`self_group()`方法，假设左侧的复合表达式总是可以无害地加上括号，那么这种可能性是存在的。也许这种改变以后可以实现，但是目前来看，保持括号规则在内部更一致似乎是更安全的方法。

### 为什么括号规则会是这样？

当括号过多或者括号出现在它们不期望的不寻常位置时，许多数据库会抛出错误，因此 SQLAlchemy 不基于分组生成括号，而是使用运算符优先级，如果运算符已知为结合性，那么会尽量生成最少的括号。否则，表达式如下：

```py
column("a") & column("b") & column("c") & column("d")
```

会产生：

```py
(((a  AND  b)  AND  c)  AND  d)
```

这是可以的，但可能会让人们感到烦恼（并报告为错误）。在其他情况下，它会导致更容易让数据库混淆，或者至少影响可读性，比如：

```py
column("q", ARRAY(Integer, dimensions=2))[5][6]
```

会产生：

```py
((q[5])[6])
```

还有一些边界情况，我们会得到像`"(x) = 7"`这样的东西，数据库真的不喜欢这样。因此，括号化不是简单地添加括号，而是使用运算符优先级和结合性来确定分组。

对于`Operators.op()`，优先级的值默认为零。

如果我们将`Operators.op.precedence`的值默认为 100，即最高值，会怎样呢？然后这个表达式会多加括号，但除此之外还是可以的，也就是说，这两个表达式是等价的：

```py
>>> print((column("q") - column("y")).op("+", precedence=100)(column("z")))
(q  -  y)  +  z
>>> print((column("q") - column("y")).op("+")(column("z")))
q  -  y  +  z 
```

但是这两种情况不是：

```py
>>> print(column("q") - column("y").op("+", precedence=100)(column("z")))
q  -  y  +  z
>>> print(column("q") - column("y").op("+")(column("z")))
q  -  (y  +  z) 
```

现在，尚不清楚只要我们基于操作符优先级和结合性进行括号化，是否真的有一种方法可以自动为没有给定优先级的通用运算符添加括号，以便在所有情况下都能正常工作，因为有时您希望自定义操作符的优先级低于其他操作符，有时您希望它更高。

也许，如果上面的“二元”表达式在调用`op()`时强制使用了`self_group()`方法，假设左侧的复合表达式总是可以无害地加括号。也许这种改变可以在某个时候实现，然而就目前而言，保持括号规则更加内部一致似乎是更安全的做法。
