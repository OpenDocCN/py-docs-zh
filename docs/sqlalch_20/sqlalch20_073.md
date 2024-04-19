# Indexable

> 原文：[`docs.sqlalchemy.org/en/20/orm/extensions/indexable.html`](https://docs.sqlalchemy.org/en/20/orm/extensions/indexable.html)

在 ORM 映射类上定义具有“index”属性的列的`Indexable`类型。

“index”表示属性与具有预定义索引以访问它的`Indexable`列的元素相关联。`Indexable`类型包括`ARRAY`、`JSON`和`HSTORE`等类型。

`indexable`扩展为任何`Indexable`类型的列的元素提供了类似于`Column`的接口。在简单情况下，它可以被视为一个`Column` - 映射属性。

## 概要

假设`Person`是一个具有主键和 JSON 数据字段的模型。虽然该字段可以包含任意数量的元素，但我们希望单独引用名为`name`的元素作为行为类似独立列的专用属性：

```py
from sqlalchemy import Column, JSON, Integer
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.ext.indexable import index_property

Base = declarative_base()

class Person(Base):
    __tablename__ = 'person'

    id = Column(Integer, primary_key=True)
    data = Column(JSON)

    name = index_property('data', 'name')
```

上面，`name`属性现在的行为类似于映射列。我们可以组合一个新的`Person`并设置`name`的值：

```py
>>> person = Person(name='Alchemist')
```

现在值是可访问的：

```py
>>> person.name
'Alchemist'
```

在幕后，JSON 字段被初始化为一个新的空字典，并设置了字段：

```py
>>> person.data
{"name": "Alchemist'}
```

该字段是可变的：

```py
>>> person.name = 'Renamed'
>>> person.name
'Renamed'
>>> person.data
{'name': 'Renamed'}
```

当使用`index_property`时，我们对可索引结构所做的更改也会自动跟踪为历史记录；我们不再需要使用`MutableDict`来跟踪此更改以进行工作单元。

删除也可以正常工作：

```py
>>> del person.name
>>> person.data
{}
```

上面，删除`person.name`会删除字典中的值，但不会删除字典本身。

缺少键将产生`AttributeError`：

```py
>>> person = Person()
>>> person.name
...
AttributeError: 'name'
```

除非设置默认值：

```py
>>> class Person(Base):
>>>     __tablename__ = 'person'
>>>
>>>     id = Column(Integer, primary_key=True)
>>>     data = Column(JSON)
>>>
>>>     name = index_property('data', 'name', default=None)  # See default

>>> person = Person()
>>> print(person.name)
None
```

这些属性也可以在类级别访问。下面，我们演示了`Person.name`用于生成带有索引的 SQL 条件：

```py
>>> from sqlalchemy.orm import Session
>>> session = Session()
>>> query = session.query(Person).filter(Person.name == 'Alchemist')
```

上述查询等效于：

```py
>>> query = session.query(Person).filter(Person.data['name'] == 'Alchemist')
```

可以链接多个`index_property`对象以生成多层索引：

```py
from sqlalchemy import Column, JSON, Integer
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.ext.indexable import index_property

Base = declarative_base()

class Person(Base):
    __tablename__ = 'person'

    id = Column(Integer, primary_key=True)
    data = Column(JSON)

    birthday = index_property('data', 'birthday')
    year = index_property('birthday', 'year')
    month = index_property('birthday', 'month')
    day = index_property('birthday', 'day')
```

上面，一个查询如下：

```py
q = session.query(Person).filter(Person.year == '1980')
```

在 PostgreSQL 后端上，上述查询将呈现为：

```py
SELECT person.id, person.data
FROM person
WHERE person.data -> %(data_1)s -> %(param_1)s = %(param_2)s
```

## 默认值

`index_property`在索引的数据结构不存在时包含特殊行为，并且调用了一个设置操作：

+   对于给定整数索引值的`index_property`，默认的数据结构将是一个 Python 列表，其中包含至少与索引值一样多的`None`值；然后将该值设置到列表中的相应位置。这意味着对于索引值为零的情况，在设置给定值之前，列表将初始化为`[None]`，对于索引值为五的情况，在设置第五个元素之前，列表将初始化为`[None, None, None, None, None]`。请注意，现有的列表**不会**直接扩展以接收一个值。

+   对于给定任何其他类型的索引值（例如通常是字符串）的`index_property`，将使用 Python 字典作为默认数据结构。

+   可以使用`index_property.datatype`参数将默认数据结构设置为任何 Python 可调用对象，覆盖以前的规则。

## 子类化

`index_property`可以被子类化，特别是针对常见的提供值或 SQL 表达式强制转换的用例。以下是在使用 PostgreSQL JSON 类型时的常见用法，其中我们还希望包括自动转换加`astext()`：

```py
class pg_json_property(index_property):
    def __init__(self, attr_name, index, cast_type):
        super(pg_json_property, self).__init__(attr_name, index)
        self.cast_type = cast_type

    def expr(self, model):
        expr = super(pg_json_property, self).expr(model)
        return expr.astext.cast(self.cast_type)
```

上述子类可以与 PostgreSQL 特定版本的`JSON`一起使用：

```py
from sqlalchemy import Column, Integer
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.dialects.postgresql import JSON

Base = declarative_base()

class Person(Base):
    __tablename__ = 'person'

    id = Column(Integer, primary_key=True)
    data = Column(JSON)

    age = pg_json_property('data', 'age', Integer)
```

实例级别的`age`属性与以前的工作方式相同；但是在渲染 SQL 时，将使用 PostgreSQL 的`->>`运算符进行索引访问，而不是通常的索引运算符`->`：

```py
>>> query = session.query(Person).filter(Person.age < 20)
```

上述查询将呈现为：

```py
SELECT person.id, person.data
FROM person
WHERE CAST(person.data ->> %(data_1)s AS INTEGER) < %(param_1)s
```

## API 参考

| 对象名称 | 描述 |
| --- | --- |
| index_property | 属性生成器。生成的属性描述了一个与`Indexable`列相对应的对象属性。 |

```py
class sqlalchemy.ext.indexable.index_property
```

属性生成器。生成的属性描述了一个与`Indexable`列相对应的对象属性。

另请参阅

`sqlalchemy.ext.indexable`

**成员**

__init__()

**类签名**

类 `sqlalchemy.ext.indexable.index_property`（`sqlalchemy.ext.hybrid.hybrid_property`）

```py
method __init__(attr_name, index, default=<object object>, datatype=None, mutable=True, onebased=True)
```

创建一个新的 `index_property`。

参数：

+   `attr_name` – Indexable 类型列的属性名，或者返回可索引结构的其他属性。

+   `index` – 用于获取和设置此值的索引。这应该是整数的 Python 端索引值。

+   `default` – 当给定索引处没有值时，将返回的值。

+   `datatype` – 当字段为空时使用的默认数据类型。默认情况下，这是从使用的索引类型派生的；对于整数索引，是 Python 列表，对于任何其他类型的索引，是 Python 字典。对于列表，列表将初始化为长度至少为 `index` 的 None 值列表。

+   `mutable` – 如果为 False，则不允许对属性进行写入和删除。

+   `onebased` – 假设此值的 SQL 表示是基于一的；也就是说，SQL 中的第一个索引是 1，而不是零。

## 概要

假设 `Person` 是一个带有主键和 JSON 数据字段的模型。虽然此字段可以包含任意数量的元素，但我们希望单独引用称为 `name` 的元素，作为一个独立的属性，其行为类似于独立的列：

```py
from sqlalchemy import Column, JSON, Integer
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.ext.indexable import index_property

Base = declarative_base()

class Person(Base):
    __tablename__ = 'person'

    id = Column(Integer, primary_key=True)
    data = Column(JSON)

    name = index_property('data', 'name')
```

上面，`name` 属性现在的行为类似于映射列。我们可以组合一个新的 `Person` 并设置 `name` 的值：

```py
>>> person = Person(name='Alchemist')
```

现在可以访问该值了：

```py
>>> person.name
'Alchemist'
```

在幕后，JSON 字段被初始化为一个新的空字典，并设置了字段：

```py
>>> person.data
{"name": "Alchemist'}
```

该字段是可变的：

```py
>>> person.name = 'Renamed'
>>> person.name
'Renamed'
>>> person.data
{'name': 'Renamed'}
```

当使用 `index_property` 时，对可索引结构所做的更改也会自动跟踪为历史记录；我们不再需要使用 `MutableDict` 来跟踪此更改的工作单元。

删除操作也正常工作：

```py
>>> del person.name
>>> person.data
{}
```

上面，对 `person.name` 的删除会删除字典中的值，但不会删除字典本身。

缺少的键会产生 `AttributeError`：

```py
>>> person = Person()
>>> person.name
...
AttributeError: 'name'
```

除非设置了默认值：

```py
>>> class Person(Base):
>>>     __tablename__ = 'person'
>>>
>>>     id = Column(Integer, primary_key=True)
>>>     data = Column(JSON)
>>>
>>>     name = index_property('data', 'name', default=None)  # See default

>>> person = Person()
>>> print(person.name)
None
```

这些属性也可以在类级别访问。下面，我们说明了 `Person.name` 用于生成带索引的 SQL 条件：

```py
>>> from sqlalchemy.orm import Session
>>> session = Session()
>>> query = session.query(Person).filter(Person.name == 'Alchemist')
```

上面的查询等效于：

```py
>>> query = session.query(Person).filter(Person.data['name'] == 'Alchemist')
```

可以链式连接多个 `index_property` 对象以产生多级索引：

```py
from sqlalchemy import Column, JSON, Integer
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.ext.indexable import index_property

Base = declarative_base()

class Person(Base):
    __tablename__ = 'person'

    id = Column(Integer, primary_key=True)
    data = Column(JSON)

    birthday = index_property('data', 'birthday')
    year = index_property('birthday', 'year')
    month = index_property('birthday', 'month')
    day = index_property('birthday', 'day')
```

上面，诸如以下查询：

```py
q = session.query(Person).filter(Person.year == '1980')
```

在 PostgreSQL 后端上，上述查询将呈现为：

```py
SELECT person.id, person.data
FROM person
WHERE person.data -> %(data_1)s -> %(param_1)s = %(param_2)s
```

## 默认值

`index_property` 包含了当索引数据结构不存在时的特殊行为，以及调用了设置操作时：

+   对于给定整数索引值的 `index_property`，默认数据结构将是包含 `None` 值的 Python 列表，至少与索引值一样长；然后将值设置在列表中的位置。这意味着对于索引值为零的索引值，列表将在设置给定值之前初始化为 `[None]`，而对于索引值为五的索引值，列表将在将第五个元素设置为给定值之前初始化为 `[None, None, None, None, None]`。请注意，现有列表 **不会** 在原地扩展以接收值。

+   对于给定任何其他类型的索引值（例如通常是字符串）的 `index_property`，将使用 Python 字典作为默认数据结构。

+   可以使用 `index_property.datatype` 参数将默认数据结构设置为任何 Python 可调用对象，从而覆盖之前的规则。

## 子类化

`index_property` 可以进行子类化，特别是用于提供在访问时进行值或 SQL 表达式强制转换的常见用例。下面是一个常见的配方，用于与 PostgreSQL JSON 类型一起使用，其中我们还希望包括自动转换以及 `astext()`：

```py
class pg_json_property(index_property):
    def __init__(self, attr_name, index, cast_type):
        super(pg_json_property, self).__init__(attr_name, index)
        self.cast_type = cast_type

    def expr(self, model):
        expr = super(pg_json_property, self).expr(model)
        return expr.astext.cast(self.cast_type)
```

上述子类可与 PostgreSQL 特定版本的 `JSON` 一起使用：

```py
from sqlalchemy import Column, Integer
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.dialects.postgresql import JSON

Base = declarative_base()

class Person(Base):
    __tablename__ = 'person'

    id = Column(Integer, primary_key=True)
    data = Column(JSON)

    age = pg_json_property('data', 'age', Integer)
```

在实例级别的 `age` 属性仍然可以正常工作；但是，在渲染 SQL 时，将使用 PostgreSQL 的 `->>` 运算符进行索引访问，而不是通常的索引运算符 `->`：

```py
>>> query = session.query(Person).filter(Person.age < 20)
```

上述查询将呈现为：

```py
SELECT person.id, person.data
FROM person
WHERE CAST(person.data ->> %(data_1)s AS INTEGER) < %(param_1)s
```

## API 参考

| 对象名称 | 描述 |
| --- | --- |
| index_property | 一个属性生成器。生成的属性描述了一个与 `Indexable` 列对应的对象属性。 |

```py
class sqlalchemy.ext.indexable.index_property
```

一个属性生成器。生成的属性描述了一个与 `Indexable` 列对应的对象属性。

另请参阅

`sqlalchemy.ext.indexable`

**成员**

__init__()

**类签名**

类 `sqlalchemy.ext.indexable.index_property`（`sqlalchemy.ext.hybrid.hybrid_property`）

```py
method __init__(attr_name, index, default=<object object>, datatype=None, mutable=True, onebased=True)
```

创建一个新的 `index_property`。

参数：

+   `attr_name` – 一个可索引类型列的属性名称，或者返回可索引结构的其他属性。

+   `index` – 用于获取和设置此值的索引。这应该是整数的 Python 端索引值。

+   `default` – 在给定索引处没有值时返回的值。

+   `datatype` – 当字段为空时使用的默认数据类型。默认情况下，这是从使用的索引类型派生的；对于整数索引，是一个 Python 列表，对于任何其他类型的索引，是一个 Python 字典。对于列表，列表将被初始化为至少`index`元素长的 None 值列表。

+   `mutable` – 如果为 False，则将禁止对属性的写入和删除。

+   `onebased` – 假设此值的 SQL 表示是基于一的；也就是说，在 SQL 中，第一个索引是 1，而不是零。
