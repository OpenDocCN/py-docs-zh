# 第三方集成问题

> 原文：[`docs.sqlalchemy.org/en/20/faq/thirdparty.html`](https://docs.sqlalchemy.org/en/20/faq/thirdparty.html)

+   我遇到了与“`numpy.int64`”、“`numpy.bool_`”等相关的错误。

+   预期为 WHERE/HAVING 角色的 SQL 表达式，实际得到了 True

## 我遇到了与“`numpy.int64`”、“`numpy.bool_`”等相关的错误。

[numpy](https://numpy.org)包具有其自己的数字数据类型，它们是从 Python 的数字类型扩展而来的，但是其中包含一些行为，在某些情况下使它们无法与 SQLAlchemy 的一些行为以及使用的底层 DBAPI 驱动程序的一些行为协调一致。

可能出现的两个错误是在诸如 psycopg2 这样的后端上出现`ProgrammingError: can't adapt type 'numpy.int64'`，以及在最近版本的 SQLAlchemy 中可能会出现`ArgumentError: SQL expression for WHERE/HAVING role expected, got True`；在更早的版本中可能会是`ArgumentError: SQL expression object expected, got object of type <class 'numpy.bool_'> instead`。

在第一种情况中，问题是由于 psycopg2 没有为`int64`数据类型提供适当的查找条目，因此它不能直接被查询接受。这可以通过以下代码进行说明：

```py
import numpy

class A(Base):
    __tablename__ = "a"

    id = Column(Integer, primary_key=True)
    data = Column(Integer)

# .. later
session.add(A(data=numpy.int64(10)))
session.commit()
```

在后一种情况中，问题是由于`numpy.int64`数据类型重写了`__eq__()`方法并强制返回表达式的返回类型为`numpy.True`或`numpy.False`，这破坏了 SQLAlchemy 的表达式语言行为，后者期望从 Python 的等式比较中返回`ColumnElement`表达式：

```py
>>> import numpy
>>> from sqlalchemy import column, Integer
>>> print(column("x", Integer) == numpy.int64(10))  # works
x  =  :x_1
>>> print(numpy.int64(10) == column("x", Integer))  # breaks
False
```

这些错误都可以通过相同的方法解决，即需要将特殊的 numpy 数据类型替换为常规的 Python 值。例如，对于诸如`numpy.int32`和`numpy.int64`之类的类型，应用 Python 的`int()`函数，对于`numpy.float32`应用 Python 的`float()`函数：

```py
data = numpy.int64(10)

session.add(A(data=int(data)))

result = session.execute(select(A.data).where(int(data) == A.data))

session.commit()
```

## 预期为 WHERE/HAVING 角色的 SQL 表达式，实际得到了 True。

参见我遇到了与“numpy.int64”、“numpy.bool_”等相关的错误。

## 我遇到了与“`numpy.int64`”、“`numpy.bool_`”等相关的错误。

[numpy](https://numpy.org)包具有其自己的数字数据类型，它们是从 Python 的数字类型扩展而来的，但是其中包含一些行为，在某些情况下使它们无法与 SQLAlchemy 的一些行为以及使用的底层 DBAPI 驱动程序的一些行为协调一致。

可能出现的两个错误是在诸如 psycopg2 这样的后端上出现`ProgrammingError: can't adapt type 'numpy.int64'`，以及在最近版本的 SQLAlchemy 中可能会出现`ArgumentError: SQL expression for WHERE/HAVING role expected, got True`；在更早的版本中可能会是`ArgumentError: SQL expression object expected, got object of type <class 'numpy.bool_'> instead`。

在第一种情况下，问题是因为 psycopg2 没有适当的查找条目来处理 `int64` 数据类型，因此它不会直接被查询接受。这可以从以下基于代码的示例中说明：

```py
import numpy

class A(Base):
    __tablename__ = "a"

    id = Column(Integer, primary_key=True)
    data = Column(Integer)

# .. later
session.add(A(data=numpy.int64(10)))
session.commit()
```

在后一种情况下，问题是由于 `numpy.int64` 数据类型覆盖了 `__eq__()` 方法，并强制表达式的返回类型为 `numpy.True` 或 `numpy.False`，这违反了 SQLAlchemy 表达式语言的行为，后者期望从 Python 相等比较中返回`ColumnElement` 表达式：

```py
>>> import numpy
>>> from sqlalchemy import column, Integer
>>> print(column("x", Integer) == numpy.int64(10))  # works
x  =  :x_1
>>> print(numpy.int64(10) == column("x", Integer))  # breaks
False
```

这些错误都可以用同样的方法解决，即需要将特殊的 numpy 数据类型替换为常规的 Python 值。例如，对像 `numpy.int32` 和 `numpy.int64` 这样的类型应用 Python 的 `int()` 函数，以及对 `numpy.float32` 应用 Python 的 `float()` 函数：

```py
data = numpy.int64(10)

session.add(A(data=int(data)))

result = session.execute(select(A.data).where(int(data) == A.data))

session.commit()
```

## 期望 WHERE/HAVING 角色的 SQL 表达式，得到了 True

请参见 I’m getting errors related to “numpy.int64”, “numpy.bool_”, 等。
