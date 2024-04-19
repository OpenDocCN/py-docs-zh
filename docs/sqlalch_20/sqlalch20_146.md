# SQLAlchemy 0.9 中的新功能是什么？

> 原文：[`docs.sqlalchemy.org/en/20/changelog/migration_09.html`](https://docs.sqlalchemy.org/en/20/changelog/migration_09.html)

关于本文档

本文档描述了 SQLAlchemy 版本 0.8 与版本 0.9 之间的变化，截至 2013 年 5 月，0.8 版本正在进行维护，而 0.9 版本在 2013 年 12 月 30 日首次发布。

文档最后更新日期：2015 年 6 月 10 日

## 介绍

本指南介绍了 SQLAlchemy 版本 0.9 中的新功能，还记录了影响将应用程序从 SQLAlchemy 0.8 系列迁移到 0.9 的用户的更改。

请仔细查看行为变化 - ORM 和行为变化 - 核心，以了解可能导致不兼容的变化。

## 平台支持

### 现在，Python 2.6 及以上版本的目标是，Python 3 不再需要 2to3

0.9 版本的第一个成就是移除对 Python 3 兼容性的 2to3 工具的依赖。为了更直接，现在目标最低的 Python 版本是 2.6，它具有与 Python 3 广泛的交叉兼容性。现在，所有 SQLAlchemy 模块和单元测试都可以在从 2.6 开始的任何 Python 解释器上等效地解释，包括 3.1 和 3.2 解释器。

[#2671](https://www.sqlalchemy.org/trac/ticket/2671)

### C 扩展在 Python 3 上得到支持

C 扩展已被移植以支持 Python 3，现在在 Python 2 和 Python 3 环境中均可构建。

[#2161](https://www.sqlalchemy.org/trac/ticket/2161)

## 行为变化 - ORM

### 当按属性查询时，现在会返回组合属性的对象形式

现在，将`Query`与组合属性结合使用时，会返回由该组合维护的对象类型，而不是被拆分为个别列。使用在组合列类型中设置的映射：

```py
>>> session.query(Vertex.start, Vertex.end).filter(Vertex.start == Point(3, 4)).all()
[(Point(x=3, y=4), Point(x=5, y=6))]
```

这个改变与期望个别属性扩展为个别列的代码不兼容。要获得该行为，请使用`.clauses`访问器：

```py
>>> session.query(Vertex.start.clauses, Vertex.end.clauses).filter(
...     Vertex.start == Point(3, 4)
... ).all()
[(3, 4, 5, 6)]
```

另请参阅

ORM 查询的列捆绑

[#2824](https://www.sqlalchemy.org/trac/ticket/2824)  ### `Query.select_from()`不再将子句应用于相应的实体

在最近的版本中，`Query.select_from()`方法已经被广泛应用，作为控制`Query`对象“选择自”的第一件事的手段，通常用于控制 JOIN 的渲染方式。

请考虑以下例子与通常的`User`映射对比：

```py
select_stmt = select([User]).where(User.id == 7).alias()

q = (
    session.query(User)
    .join(select_stmt, User.id == select_stmt.c.id)
    .filter(User.name == "ed")
)
```

上述语句可预见地生成类似以下的 SQL：

```py
SELECT  "user".id  AS  user_id,  "user".name  AS  user_name
FROM  "user"  JOIN  (SELECT  "user".id  AS  id,  "user".name  AS  name
FROM  "user"
WHERE  "user".id  =  :id_1)  AS  anon_1  ON  "user".id  =  anon_1.id
WHERE  "user".name  =  :name_1
```

如果我们想要颠倒 JOIN 的左右元素的顺序，文档会让我们相信可以使用`Query.select_from()`来实现：

```py
q = (
    session.query(User)
    .select_from(select_stmt)
    .join(User, User.id == select_stmt.c.id)
    .filter(User.name == "ed")
)
```

然而，在 0.8 版本及更早版本中，上述对`Query.select_from()`的使用会将`select_stmt`应用于**替换**`User`实体，因为它选择了与`User`兼容的`user`表：

```py
-- SQLAlchemy 0.8 and earlier...
SELECT  anon_1.id  AS  anon_1_id,  anon_1.name  AS  anon_1_name
FROM  (SELECT  "user".id  AS  id,  "user".name  AS  name
FROM  "user"
WHERE  "user".id  =  :id_1)  AS  anon_1  JOIN  "user"  ON  anon_1.id  =  anon_1.id
WHERE  anon_1.name  =  :name_1
```

上述语句混乱不堪，ON 子句引用了`anon_1.id = anon_1.id`，我们的 WHERE 子句也被替换为`anon_1`。

这种行为是完全有意的，但与已经变得流行的`Query.select_from()`有不同的用例。上述行为现在可以通过一个名为`Query.select_entity_from()`的新方法来实现。这是一个较少使用的行为，在现代 SQLAlchemy 中大致相当于从自定义的`aliased()`构造中选择：

```py
select_stmt = select([User]).where(User.id == 7)
user_from_stmt = aliased(User, select_stmt.alias())

q = session.query(user_from_stmt).filter(user_from_stmt.name == "ed")
```

因此，在 SQLAlchemy 0.9 中，我们从`select_stmt`选择的查询会产生我们期望的 SQL：

```py
-- SQLAlchemy 0.9
SELECT  "user".id  AS  user_id,  "user".name  AS  user_name
FROM  (SELECT  "user".id  AS  id,  "user".name  AS  name
FROM  "user"
WHERE  "user".id  =  :id_1)  AS  anon_1  JOIN  "user"  ON  "user".id  =  id
WHERE  "user".name  =  :name_1
```

`Query.select_entity_from()`方法将在 SQLAlchemy **0.8.2**中可用，因此依赖旧行为的应用程序可以首先过渡到这种方法，确保所有测试继续正常运行，然后无问题地升级到 0.9。

[#2736](https://www.sqlalchemy.org/trac/ticket/2736)  ### `viewonly=True` on `relationship()` prevents history from taking effect

在`relationship()`上的`viewonly`标志被应用于防止对目标属性的更改在刷新过程中产生任何影响。这是通过在刷新过程中排除属性来实现的。然而，直到现在，对属性的更改仍然会将父对象标记为“脏”，并触发潜在的刷新。改变是`viewonly`标志现在也阻止为目标属性设置历史记录。像反向引用和用户定义事件这样的属性事件仍然会正常工作。

改变如下所示：

```py
from sqlalchemy import Column, Integer, ForeignKey, create_engine
from sqlalchemy.orm import backref, relationship, Session
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy import inspect

Base = declarative_base()

class A(Base):
    __tablename__ = "a"
    id = Column(Integer, primary_key=True)

class B(Base):
    __tablename__ = "b"

    id = Column(Integer, primary_key=True)
    a_id = Column(Integer, ForeignKey("a.id"))
    a = relationship("A", backref=backref("bs", viewonly=True))

e = create_engine("sqlite://")
Base.metadata.create_all(e)

a = A()
b = B()

sess = Session(e)
sess.add_all([a, b])
sess.commit()

b.a = a

assert b in sess.dirty

# before 0.9.0
# assert a in sess.dirty
# assert inspect(a).attrs.bs.history.has_changes()

# after 0.9.0
assert a not in sess.dirty
assert not inspect(a).attrs.bs.history.has_changes()
```

[#2833](https://www.sqlalchemy.org/trac/ticket/2833)  ### 关联代理 SQL 表达式改进和修复

通过关联代理实现的`==`和`!=`运算符，引用标量关系上的标量值，现在会产生更完整的 SQL 表达式，旨在考虑当比较对象为`None`时“关联”行是否存在。

考虑以下映射：

```py
class A(Base):
    __tablename__ = "a"

    id = Column(Integer, primary_key=True)

    b_id = Column(Integer, ForeignKey("b.id"), primary_key=True)
    b = relationship("B")
    b_value = association_proxy("b", "value")

class B(Base):
    __tablename__ = "b"
    id = Column(Integer, primary_key=True)
    value = Column(String)
```

在 0.8 之前，像下面这样的查询：

```py
s.query(A).filter(A.b_value == None).all()
```

会产生：

```py
SELECT  a.id  AS  a_id,  a.b_id  AS  a_b_id
FROM  a
WHERE  EXISTS  (SELECT  1
FROM  b
WHERE  b.id  =  a.b_id  AND  b.value  IS  NULL)
```

在 0.9 中，现在会产生：

```py
SELECT  a.id  AS  a_id,  a.b_id  AS  a_b_id
FROM  a
WHERE  (EXISTS  (SELECT  1
FROM  b
WHERE  b.id  =  a.b_id  AND  b.value  IS  NULL))  OR  a.b_id  IS  NULL
```

不同之处在于，它不仅检查 `b.value`，还检查 `a` 是否根本没有关联到任何 `b` 行。对于使用这种类型比较的系统，一些父行没有关联行，这将与之前的版本返回不同的结果。

更为关键的是，对于 `A.b_value != None`，会发出正确的表达式。在 0.8 中，对于没有 `b` 的 `A` 行，这将返回 `True`：

```py
SELECT  a.id  AS  a_id,  a.b_id  AS  a_b_id
FROM  a
WHERE  NOT  (EXISTS  (SELECT  1
FROM  b
WHERE  b.id  =  a.b_id  AND  b.value  IS  NULL))
```

现在在 0.9 中，检查已经重新设计，以确保 A.b_id 行存在，另外 `B.value` 不为 NULL：

```py
SELECT  a.id  AS  a_id,  a.b_id  AS  a_b_id
FROM  a
WHERE  EXISTS  (SELECT  1
FROM  b
WHERE  b.id  =  a.b_id  AND  b.value  IS  NOT  NULL)
```

此外，`has()` 操作符得到增强，使得你可以只针对标量列值调用它，而无需任何条件，它将生成检查关联行是否存在的条件：

```py
s.query(A).filter(A.b_value.has()).all()
```

输出：

```py
SELECT  a.id  AS  a_id,  a.b_id  AS  a_b_id
FROM  a
WHERE  EXISTS  (SELECT  1
FROM  b
WHERE  b.id  =  a.b_id)
```

这等同于 `A.b.has()`，但允许直接针对 `b_value` 进行查询。

[#2751](https://www.sqlalchemy.org/trac/ticket/2751)  ### 关联代理缺失标量返回 None

从标量属性到标量的关联代理现在如果被代理对象不存在将返回 `None`。这与 SQLAlchemy 中缺少多对一关系返回 None 的事实一致，因此代理值也应该如此。例如：

```py
from sqlalchemy import *
from sqlalchemy.orm import *
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.ext.associationproxy import association_proxy

Base = declarative_base()

class A(Base):
    __tablename__ = "a"

    id = Column(Integer, primary_key=True)
    b = relationship("B", uselist=False)

    bname = association_proxy("b", "name")

class B(Base):
    __tablename__ = "b"

    id = Column(Integer, primary_key=True)
    a_id = Column(Integer, ForeignKey("a.id"))
    name = Column(String)

a1 = A()

# this is how m2o's always have worked
assert a1.b is None

# but prior to 0.9, this would raise AttributeError,
# now returns None just like the proxied value.
assert a1.bname is None
```

[#2810](https://www.sqlalchemy.org/trac/ticket/2810)  ### attributes.get_history() 默认情况下将从数据���查询如果值不存在

修复了关于`get_history()`的 bug，允许基于列的属性向数据库查询未加载的值，假设 `passive` 标志保持默认值 `PASSIVE_OFF`。之前，此标志不会被遵守。此外，新增了一个方法`AttributeState.load_history()`来补充`AttributeState.history`属性，它将为未加载的属性发出加载器可调用。

这是一个小改变的示例：

```py
from sqlalchemy import Column, Integer, String, create_engine, inspect
from sqlalchemy.orm import Session, attributes
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()

class A(Base):
    __tablename__ = "a"
    id = Column(Integer, primary_key=True)
    data = Column(String)

e = create_engine("sqlite://", echo=True)
Base.metadata.create_all(e)

sess = Session(e)

a1 = A(data="a1")
sess.add(a1)
sess.commit()  # a1 is now expired

# history doesn't emit loader callables
assert inspect(a1).attrs.data.history == (None, None, None)

# in 0.8, this would fail to load the unloaded state.
assert attributes.get_history(a1, "data") == (
    (),
    [
        "a1",
    ],
    (),
)

# load_history() is now equivalent to get_history() with
# passive=PASSIVE_OFF ^ INIT_OK
assert inspect(a1).attrs.data.load_history() == (
    (),
    [
        "a1",
    ],
    (),
)
```

[#2787](https://www.sqlalchemy.org/trac/ticket/2787)  ## 行为变更 - 核心

### 类型对象不再接受被忽略的关键字参数

在 0.8 系列中，大多数类型对象接受任意关键字参数，这些参数会被静默忽略：

```py
from sqlalchemy import Date, Integer

# storage_format argument here has no effect on any backend;
# it needs to be on the SQLite-specific type
d = Date(storage_format="%(day)02d.%(month)02d.%(year)04d")

# display_width argument here has no effect on any backend;
# it needs to be on the MySQL-specific type
i = Integer(display_width=5)
```

这是一个非常古老的 bug，为此在 0.8 系列中添加了一个弃用警告，但因为几乎没有人使用带有“-W”标志的 Python，所以几乎从未见过：

```py
$ python -W always::DeprecationWarning ~/dev/sqlalchemy/test.py
/Users/classic/dev/sqlalchemy/test.py:5: SADeprecationWarning: Passing arguments to
type object constructor <class 'sqlalchemy.types.Date'> is deprecated
  d = Date(storage_format="%(day)02d.%(month)02d.%(year)04d")
/Users/classic/dev/sqlalchemy/test.py:9: SADeprecationWarning: Passing arguments to
type object constructor <class 'sqlalchemy.types.Integer'> is deprecated
  i = Integer(display_width=5)
```

从 0.9 系列开始，`TypeEngine` 中的“catch all” 构造函数被移除，这些无意义的参数不再被接受。

使用方言特定参数如 `storage_format` 和 `display_width` 的正确方法是使用适当的方言特定类型：

```py
from sqlalchemy.dialects.sqlite import DATE
from sqlalchemy.dialects.mysql import INTEGER

d = DATE(storage_format="%(day)02d.%(month)02d.%(year)04d")

i = INTEGER(display_width=5)
```

那么当我们还需要方言无关的类型时呢？我们使用 `TypeEngine.with_variant()` 方法：

```py
from sqlalchemy import Date, Integer
from sqlalchemy.dialects.sqlite import DATE
from sqlalchemy.dialects.mysql import INTEGER

d = Date().with_variant(
    DATE(storage_format="%(day)02d.%(month)02d.%(year)04d"), "sqlite"
)

i = Integer().with_variant(INTEGER(display_width=5), "mysql")
```

`TypeEngine.with_variant()` 并不是新的，它是在 SQLAlchemy 0.7.2 中添加的。所以可以将在 0.8 系列上运行的代码修改为使用这种方法，并在升级到 0.9 之前进行测试。

### `None` 不再能够被用作 “部分 AND” 构造函数

`None` 不再能够被用作逐步形成 AND 条件的 “后备”。即使一些 SQLAlchemy 内部使用了这种模式，但这种模式并没有被记录在案：

```py
condition = None

for cond in conditions:
    condition = condition & cond

if condition is not None:
    stmt = stmt.where(condition)
```

在 0.9 上，当 `conditions` 不为空时，将产生 `SELECT .. WHERE <condition> AND NULL`。`None` 不再被隐式忽略，而是与在其他上下文中解释 `None` 时一致。

0.8 和 0.9 的正确代码应该是：

```py
from sqlalchemy.sql import and_

if conditions:
    stmt = stmt.where(and_(*conditions))
```

另一个变体，在 0.9 上对所有后端都有效，但在 0.8 上仅在支持布尔常量的后端上有效：

```py
from sqlalchemy.sql import true

condition = true()

for cond in conditions:
    condition = cond & condition

stmt = stmt.where(condition)
```

在 0.8 上，这将生成一个始终在 WHERE 子句中具有 `AND true` 的 SELECT 语句，这是不被不支持布尔常量的后端（MySQL，MSSQL）接受的。在 0.9 上，`true` 常量将在 `and_()` 连接中被删除。

另请参阅

布尔常量、NULL 常量、连接词的渲染已经得到改进

### `create_engine()` 的 “password” 部分不再将 `+` 号视为已编码的空格

不知何故，Python 函数 `unquote_plus()` 被应用于 URL 的 `password` 字段，这是对 [RFC 1738](https://www.ietf.org/rfc/rfc1738.txt) 中描述的编码规则的错误应用，因为它将空格转义为加号。URL 的字符串化现在只编码 “:”，“@” 或 “/”，不编码其他任何字符，并且现在应用于 `username` 和 `password` 字段（以前只应用于密码）。在解析时，编码字符会被转换，但加号和空格会原样传递：

```py
# password: "pass word + other:words"
dbtype://user:pass word + other%3Awords@host/dbname

# password: "apples/oranges"
dbtype://username:apples%2Foranges@hostspec/database

# password: "apples@oranges@@"
dbtype://username:apples%40oranges%40%40@hostspec/database

# password: '', username is "username@"
dbtype://username%40:@hostspec/database
```

[#2873](https://www.sqlalchemy.org/trac/ticket/2873)  ### COLLATE 的优先规则已经更改

以前，类似以下的表达式：

```py
print((column("x") == "somevalue").collate("en_EN"))
```

会产生这样的表达式：

```py
-- 0.8 behavior
(x  =  :x_1)  COLLATE  en_EN
```

上述情况被 MSSQL 误解，通常不是任何数据库建议的语法。现在该表达式将生成大多数数据库文档所示的语法：

```py
-- 0.9 behavior
x  =  :x_1  COLLATE  en_EN
```

如果 `ColumnOperators.collate()` 操作符被应用于右侧列，则会出现潜在的不向后兼容的更改，如下所示：

```py
print(column("x") == literal("somevalue").collate("en_EN"))
```

在 0.8 中，这将产生：

```py
x  =  :param_1  COLLATE  en_EN
```

但在 0.9 中，现在将产生更准确的，但可能不是您想要的形式：

```py
x  =  (:param_1  COLLATE  en_EN)
```

`ColumnOperators.collate()` 运算符现在在`ORDER BY`表达式中的使用更加恰当，因为给`ASC`和`DESC`运算符指定了特定的优先级，这将再次确保不生成括号：

```py
>>> # 0.8
>>> print(column("x").collate("en_EN").desc())
(x  COLLATE  en_EN)  DESC
>>> # 0.9
>>> print(column("x").collate("en_EN").desc())
x  COLLATE  en_EN  DESC 
```

[#2879](https://www.sqlalchemy.org/trac/ticket/2879)  ### PostgreSQL CREATE TYPE <x> AS ENUM 现在对值应用引号

`ENUM` 类型现在将对枚举值中的单引号符号应用转义：

```py
>>> from sqlalchemy.dialects import postgresql
>>> type = postgresql.ENUM("one", "two", "three's", name="myenum")
>>> from sqlalchemy.dialects.postgresql import base
>>> print(base.CreateEnumType(type).compile(dialect=postgresql.dialect()))
CREATE  TYPE  myenum  AS  ENUM  ('one','two','three''s') 
```

已经转义单引号符号的现有解决方法需要进行修改，否则它们现在会双重转义。

[#2878](https://www.sqlalchemy.org/trac/ticket/2878)

## 新特性

### 事件移除 API

使用`listen()`或`listens_for()`建立的事件现在可以使用新的`remove()`函数进行移除。传递给`remove()`的`target`、`identifier`和`fn`参数需要与监听时发送的完全匹配，并且事件将从其已建立的所有位置中移除：

```py
@event.listens_for(MyClass, "before_insert", propagate=True)
def my_before_insert(mapper, connection, target):
  """listen for before_insert"""
    # ...

event.remove(MyClass, "before_insert", my_before_insert)
```

在上面的示例中，设置了`propagate=True`标志。这意味着`my_before_insert()`被建立为`MyClass`以及`MyClass`的所有子类的监听器。系统跟踪到`my_before_insert()`监听函数在此调用的结果中被放置的所有位置，并在调用`remove()`时将其移除。

移除系统使用注册表将传递给`listen()`的参数与事件监听器的集合相关联，这些监听器在许多情况下是原始用户提供的函数的包装版本。此注册表大量使用弱引用，以允许所有包含的内容（如监听器目标）在其超出范围时被垃圾收集。

[#2268](https://www.sqlalchemy.org/trac/ticket/2268)  ### 新查询选项 API; `load_only()` 选项

加载器选项的系统，如`joinedload()`、`subqueryload()`、`lazyload()`、`defer()`等，都建立在一个称为`Load`的新系统之上。`Load`提供了一种“方法链式”（又称生成式）的加载器选项方法，因此不再需要使用点号或多个属性名称将长路径连接在一起，而是为每个路径提供明确的加载器样式。

虽然新方式稍微更冗长，但更容易理解，因为在应用哪些选项到哪些路径上没有歧义；它简化了选项的方法签名，并为基于列的选项提供了更大的灵活性。旧系统将一直保持功能，并且所有样式都可以混合使用。

**旧方式**

要在多元素路径中的每个链接上设置特定的加载样式，必须使用`_all()`选项：

```py
query(User).options(joinedload_all("orders.items.keywords"))
```

**新方式**

现在加载器选项是可链式的，因此相同的`joinedload(x)`方法等同地应用于每个链接，无需在`joinedload()`和`joinedload_all()`之间保持清晰：

```py
query(User).options(joinedload("orders").joinedload("items").joinedload("keywords"))
```

**旧方式**

在基于子类的路径上设置选项需要将路径中的所有链接拼写为类绑定属性，因为需要调用`PropComparator.of_type()`方法：

```py
session.query(Company).options(
    subqueryload_all(Company.employees.of_type(Engineer), Engineer.machines)
)
```

**新方式**

只有路径中实际需要`PropComparator.of_type()`的元素需要设置为类绑定属性，之后可以恢复使用基于字符串的名称：

```py
session.query(Company).options(
    subqueryload(Company.employees.of_type(Engineer)).subqueryload("machines")
)
```

**旧方式**

在长路径中设置加载器选项的最后一个链接使用的语法看起来很像应该为路径中的所有链接设置选项，导致混淆：

```py
query(User).options(subqueryload("orders.items.keywords"))
```

**新方式**

现在可以使用`defaultload()`来明确指定路径，其中现有的加载器样式不应更改。更冗长但意图更清晰：

```py
query(User).options(defaultload("orders").defaultload("items").subqueryload("keywords"))
```

仍然可以利用点��样式，特别是在跳过几个路径元素的情况下：

```py
query(User).options(defaultload("orders.items").subqueryload("keywords"))
```

**旧方式**

在路径上需要为每一列拼写完整路径的 `defer()` 选项：

```py
query(User).options(defer("orders.description"), defer("orders.isopen"))
```

**新方式**

一个到达目标路径的单个 `Load` 对象可以反复调用 `Load.defer()`：

```py
query(User).options(defaultload("orders").defer("description").defer("isopen"))
```

#### 加载类

`Load` 类可以直接用于提供“绑定”目标，特别是当存在多个父实体时：

```py
from sqlalchemy.orm import Load

query(User, Address).options(Load(Address).joinedload("entries"))
```

#### 仅加载

一个新选项 `load_only()` 实现了“除了延迟加载其他所有内容”的加载方式，仅加载给定列并推迟其余内容：

```py
from sqlalchemy.orm import load_only

query(User).options(load_only("name", "fullname"))

# specify explicit parent entity
query(User, Address).options(Load(User).load_only("name", "fullname"))

# specify path
query(User).options(joinedload(User.addresses).load_only("email_address"))
```

#### 类特定的通配符

使用 `Load`，可以使用通配符来设置给定实体上所有关系（或者列）的加载方式，而不影响其他实体：

```py
# lazyload all User relationships
query(User).options(Load(User).lazyload("*"))

# undefer all User columns
query(User).options(Load(User).undefer("*"))

# lazyload all Address relationships
query(User).options(defaultload(User.addresses).lazyload("*"))

# undefer all Address columns
query(User).options(defaultload(User.addresses).undefer("*"))
```

[#1418](https://www.sqlalchemy.org/trac/ticket/1418)  ### 新的 `text()` 功能

`text()` 构造获得了新的方法：

+   `TextClause.bindparams()` 允许灵活设置绑定参数类型和值：

    ```py
    # setup values
    stmt = text(
        "SELECT id, name FROM user WHERE name=:name AND timestamp=:timestamp"
    ).bindparams(name="ed", timestamp=datetime(2012, 11, 10, 15, 12, 35))

    # setup types and/or values
    stmt = (
        text("SELECT id, name FROM user WHERE name=:name AND timestamp=:timestamp")
        .bindparams(bindparam("name", value="ed"), bindparam("timestamp", type_=DateTime()))
        .bindparam(timestamp=datetime(2012, 11, 10, 15, 12, 35))
    )
    ```

+   `TextClause.columns()` 取代了 `text()` 的 `typemap` 选项，返回一个新的构造 `TextAsFrom`：

    ```py
    # turn a text() into an alias(), with a .c. collection:
    stmt = text("SELECT id, name FROM user").columns(id=Integer, name=String)
    stmt = stmt.alias()

    stmt = select([addresses]).select_from(
        addresses.join(stmt), addresses.c.user_id == stmt.c.id
    )

    # or into a cte():
    stmt = text("SELECT id, name FROM user").columns(id=Integer, name=String)
    stmt = stmt.cte("x")

    stmt = select([addresses]).select_from(
        addresses.join(stmt), addresses.c.user_id == stmt.c.id
    )
    ```

[#2877](https://www.sqlalchemy.org/trac/ticket/2877)  ### 从 SELECT 插入

经过几乎多年的毫无意义的拖延，这个相对较小的语法特性已经被添加，并且也被回溯到了 0.8.3，所以在技术上并不是 0.9 中的“新”特性。可以将一个 `select()` 构造或其他兼容的构造传递给新方法 `Insert.from_select()`，它将用于渲染一个 `INSERT .. SELECT` 构造：

```py
>>> from sqlalchemy.sql import table, column
>>> t1 = table("t1", column("a"), column("b"))
>>> t2 = table("t2", column("x"), column("y"))
>>> print(t1.insert().from_select(["a", "b"], t2.select().where(t2.c.y == 5)))
INSERT  INTO  t1  (a,  b)  SELECT  t2.x,  t2.y
FROM  t2
WHERE  t2.y  =  :y_1 
```

该构造足够智能，也可以适应诸如类和 `Query` 对象等 ORM 对象：

```py
s = Session()
q = s.query(User.id, User.name).filter_by(name="ed")
ins = insert(Address).from_select((Address.id, Address.email_address), q)
```

渲染：

```py
INSERT  INTO  addresses  (id,  email_address)
SELECT  users.id  AS  users_id,  users.name  AS  users_name
FROM  users  WHERE  users.name  =  :name_1
```

[#722](https://www.sqlalchemy.org/trac/ticket/722)  ### `select()`、`Query()` 上的新 FOR UPDATE 支持

试图简化 Core 和 ORM 中对 `SELECT` 语句上的 `FOR UPDATE` 子句的规范，并支持 PostgreSQL 和 Oracle 支持的 `FOR UPDATE OF` SQL。

使用核心 `GenerativeSelect.with_for_update()`，可以单独指定 `FOR SHARE` 和 `NOWAIT` 等选项，而不是链接到任意字符串代码：

```py
stmt = select([table]).with_for_update(read=True, nowait=True, of=table)
```

在 Posgtresql 上述语句可能会呈现为：

```py
SELECT  table.a,  table.b  FROM  table  FOR  SHARE  OF  table  NOWAIT
```

`Query` 对象获得了一个类似的方法 `Query.with_for_update()`，其行为方式相同。这个方法取代了现有的 `Query.with_lockmode()` 方法，该方法使用不同的系统翻译 `FOR UPDATE` 子句。目前，“lockmode” 字符串参数仍然被 `Session.refresh()` 方法接受。### 可配置原生浮点类型的浮点字符串转换精度

每当 DBAPI 返回一个要转换为 Python `Decimal()` 的 Python 浮点类型时，SQLAlchemy 都会进行转换，这必然涉及将浮点值转换为字符串的中间步骤。此字符串转换的比例以前是硬编码为 10，现在可以配置。这个设置在 `Numeric` 和 `Float` 类型以及所有 SQL 和方言特定的后代类型上都可用，使用参数 `decimal_return_scale`。如果类型支持 `.scale` 参数，比如 `Numeric` 和一些浮点类型如 `DOUBLE`，如果没有另外指定，`.scale` 的值将作为 `.decimal_return_scale` 的默认值。如果 `.scale` 和 `.decimal_return_scale` 都不存在，则默认值为 10。例如：

```py
from sqlalchemy.dialects.mysql import DOUBLE
import decimal

data = Table(
    "data",
    metadata,
    Column("double_value", mysql.DOUBLE(decimal_return_scale=12, asdecimal=True)),
)

conn.execute(
    data.insert(),
    double_value=45.768392065789,
)
result = conn.scalar(select([data.c.double_value]))

# previously, this would typically be Decimal("45.7683920658"),
# e.g. trimmed to 10 decimal places

# now we get 12, as requested, as MySQL can support this
# much precision for DOUBLE
assert result == decimal.Decimal("45.768392065789")
```

[#2867](https://www.sqlalchemy.org/trac/ticket/2867)  ### ORM 查询的列捆绑

`Bundle` 允许查询一组列，然后将它们分组为查询返回的元组下的一个名称。 `Bundle` 的初始目的是 1\. 允许将“复合”ORM 列作为列式结果集中的单个值返回，而不是将它们展开为单独的列，以及 2\. 允许在 ORM 中创建自定义结果集构造，使用临时列和返回类型，而不涉及映射类的更重量级机制。

另请参见

组合属性现在在按属性查询时以其对象形式返回

使用 Bundles 对选定属性进行分组

[#2824](https://www.sqlalchemy.org/trac/ticket/2824)

### 服务器端版本计数

ORM 的版本控制功能（现在也在配置版本计数器中有文档）现在可以利用服务器端的版本计数方案，例如由触发器或数据库系统列生成的方案，以及版本 _id_counter 函数之外的条件编程方案。 通过向 `version_id_generator` 参数提供值 `False`，ORM 将使用已设置的版本标识符，或者在发出 INSERT 或 UPDATE 时同时从每行获取版本标识符。 当使用服务器生成的版本标识符时，强烈建议仅在具有强大 RETURNING 支持的后端上使用此功能（PostgreSQL、SQL Server；Oracle 也支持 RETURNING，但 cx_oracle 驱动程序仅具有有限的支持），否则额外的 SELECT 语句将增加显着的性能开销。 在服务器端版本计数器提供的示例中说明了使用 PostgreSQL 的 `xmin` 系统列以将其与 ORM 的版本控制功能集成。

另请参见

服务器端版本计数器

[#2793](https://www.sqlalchemy.org/trac/ticket/2793)

### `include_backrefs=False` 选项用于 `@validates`

`validates()` 函数现在接受一个选项 `include_backrefs=True`，这将绕过为从 backref 发起的事件触发验证器的情况：

```py
from sqlalchemy import Column, Integer, ForeignKey
from sqlalchemy.orm import relationship, validates
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()

class A(Base):
    __tablename__ = "a"

    id = Column(Integer, primary_key=True)
    bs = relationship("B", backref="a")

    @validates("bs")
    def validate_bs(self, key, item):
        print("A.bs validator")
        return item

class B(Base):
    __tablename__ = "b"

    id = Column(Integer, primary_key=True)
    a_id = Column(Integer, ForeignKey("a.id"))

    @validates("a", include_backrefs=False)
    def validate_a(self, key, item):
        print("B.a validator")
        return item

a1 = A()
a1.bs.append(B())  # prints only "A.bs validator"
```

[#1535](https://www.sqlalchemy.org/trac/ticket/1535)

### PostgreSQL JSON 类型

PostgreSQL 方言现在具有一个 `JSON` 类型，以补充 `HSTORE` 类型。

另请参见

`JSON`

[#2581](https://www.sqlalchemy.org/trac/ticket/2581)

### Automap 扩展

在 **0.9.1** 中添加了一个名为 `sqlalchemy.ext.automap` 的新扩展。 这是一个 **实验性** 扩展，它扩展了声明性的功能以及 `DeferredReflection` 类的功能。 本质上，该扩展提供了一个基类 `AutomapBase`，根据给定的表元数据自动生成映射类和它们之间的关系。

通常使用的 `MetaData` 可能是通过反射生成的，但不要求使用反射。 最基本的用法说明了 `sqlalchemy.ext.automap` 如何根据反射模式提供映射类，包括关系：

```py
from sqlalchemy.ext.automap import automap_base
from sqlalchemy.orm import Session
from sqlalchemy import create_engine

Base = automap_base()

# engine, suppose it has two tables 'user' and 'address' set up
engine = create_engine("sqlite:///mydatabase.db")

# reflect the tables
Base.prepare(engine, reflect=True)

# mapped classes are now created with names matching that of the table
# name.
User = Base.classes.user
Address = Base.classes.address

session = Session(engine)

# rudimentary relationships are produced
session.add(Address(email_address="foo@bar.com", user=User(name="foo")))
session.commit()

# collection-based relationships are by default named "<classname>_collection"
print(u1.address_collection)
```

除此之外，`AutomapBase` 类是一个声明基类，并支持所有声明所支持的功能。 “自动映射”功能可用于现有的、明确声明的模式，以仅生成关系和缺失类。 命名方案和关系生成例程可以通过可调用函数添加。

希望 `AutomapBase` 系统提供了一个快速和现代化的解决方案，解决了非常著名的 [SQLSoup](https://sqlsoup.readthedocs.io/en/latest/) 也试图解决的问题，即从现有数据库快速生成一个简单的对象模型。 通过严格在映射器配置级别解决问题，并与现有的声明类技术完全集成，`AutomapBase` 试图提供一个与问题紧密集成的方法，以便快速生成临时映射。

另请参阅

Automap

## 行为改进

应该产生没有兼容性问题的改进，除非在极为罕见和异常的假设情况下，但最好知道这些改进，以防出现意外问题。

### 许多 JOIN 和 LEFT OUTER JOIN 表达式将不再被包装在 (SELECT * FROM ..) AS ANON_1 中

多年来，SQLAlchemy ORM 一直无法将 JOIN 嵌套在现有 JOIN 的右侧（通常是 LEFT OUTER JOIN，因为 INNER JOIN 总是可以被展平）：

```py
SELECT  a.*,  b.*,  c.*  FROM  a  LEFT  OUTER  JOIN  (b  JOIN  c  ON  b.id  =  c.id)  ON  a.id
```

这是因为 SQLite 直到版本 **3.7.16** 都无法解析上述格式的语句：

```py
SQLite version 3.7.15.2 2013-01-09 11:53:05
Enter ".help" for instructions
Enter SQL statements terminated with a ";"
sqlite> create table a(id integer);
sqlite> create table b(id integer);
sqlite> create table c(id integer);
sqlite> select a.id, b.id, c.id from a left outer join (b join c on b.id=c.id) on b.id=a.id;
Error: no such column: b.id
```

右外连接当然是解决右侧括号化的另一种方法；这将变得非常复杂和视觉上不愉快，但幸运的是 SQLite 也不支持 RIGHT OUTER JOIN :):

```py
sqlite>  select  a.id,  b.id,  c.id  from  b  join  c  on  b.id=c.id
  ...>  right  outer  join  a  on  b.id=a.id;
Error:  RIGHT  and  FULL  OUTER  JOINs  are  not  currently  supported
```

早在 2005 年，不清楚其他数据库是否有问题，但今天似乎很明显，除 SQLite 外，每个经过测试的数据库都支持它（Oracle 8，一个非常古老的数据库，根本不支持 JOIN 关键字，但 SQLAlchemy 一直对 Oracle 的语法有一个简单的重写方案）。更糟糕的是，SQLAlchemy 通常的解决方法是在像 PostgreSQL 和 MySQL 这样的平台上应用 SELECT 通常会降低性能：

```py
SELECT  a.*,  anon_1.*  FROM  a  LEFT  OUTER  JOIN  (
  SELECT  b.id  AS  b_id,  c.id  AS  c_id
  FROM  b  JOIN  c  ON  b.id  =  c.id
  )  AS  anon_1  ON  a.id=anon_1.b_id
```

类似上面形式的 JOIN 在处理连接表继承结构时很常见；每当使用 `Query.join()` 从某个父类连接到一个连接表子类，或者类似地使用 `joinedload()`，SQLAlchemy 的 ORM 总是确保不会渲染嵌套的 JOIN，以免查询无法在 SQLite 上运行。尽管 Core 一直支持更紧凑形式的 JOIN，ORM 必须避免使用它。

当在 ON 子句中存在特殊条件时，通过多对多关系生成连接时会出现另一个问题。考虑以下急加载连接：

```py
session.query(Order).outerjoin(Order.items)
```

假设从 `Order` 到 `Item` 的多对多实际上指的是一个子类，如 `Subitem`，上述情况的 SQL 如下所示：

```py
SELECT  order.id,  order.name
FROM  order  LEFT  OUTER  JOIN  order_item  ON  order.id  =  order_item.order_id
LEFT  OUTER  JOIN  item  ON  order_item.item_id  =  item.id  AND  item.type  =  'subitem'
```

上面的查询有什么问题？基本上，它将加载许多 `order` / `order_item` 行，其中 `item.type == 'subitem'` 的条件不成立。

从 SQLAlchemy 0.9 开始，采取了一种全新的方法。ORM 不再担心将 JOIN 嵌套在封闭 JOIN 的右侧，现在它会尽可能地渲染这些 JOIN，同时仍然返回正确的结果。当 SQL 语句被传递进行编译时，**方言编译器**会根据目标后端进行 **重写 JOIN**，如果该后端已知不支持右嵌套 JOIN（目前只有 SQLite - 如果其他后端有此问题，请告诉我们！）。

因此，一个常规的 `query(Parent).join(Subclass)` 现在通常会产生一个更简单的表达式：

```py
SELECT  parent.id  AS  parent_id
FROM  parent  JOIN  (
  base_table  JOIN  subclass_table
  ON  base_table.id  =  subclass_table.id)  ON  parent.id  =  base_table.parent_id
```

类似 `query(Parent).options(joinedload(Parent.subclasses))` 的连接急加载将对各个表进行别名处理，而不是包装在 `ANON_1` 中：

```py
SELECT  parent.*,  base_table_1.*,  subclass_table_1.*  FROM  parent
  LEFT  OUTER  JOIN  (
  base_table  AS  base_table_1  JOIN  subclass_table  AS  subclass_table_1
  ON  base_table_1.id  =  subclass_table_1.id)
  ON  parent.id  =  base_table_1.parent_id
```

多对多连接和急加载将右嵌套“secondary”和“right”表：

```py
SELECT  order.id,  order.name
FROM  order  LEFT  OUTER  JOIN
(order_item  JOIN  item  ON  order_item.item_id  =  item.id  AND  item.type  =  'subitem')
ON  order_item.order_id  =  order.id
```

所有这些连接，当使用`Select`语句渲染时，该语句明确指定`use_labels=True`，这对 ORM 发出的所有查询都是真实的，都是“连接重写”的候选对象，这是将所有这些右嵌套连接重写为嵌套的 SELECT 语句的过程，同时保持`Select`使用的相同标签。因此，SQLite，即使在 2013 年，也不支持这种非常常见的 SQL 语法，也要自己承担额外的复杂性，以上查询被重写为：

```py
-- sqlite only!
SELECT  parent.id  AS  parent_id
  FROM  parent  JOIN  (
  SELECT  base_table.id  AS  base_table_id,
  base_table.parent_id  AS  base_table_parent_id,
  subclass_table.id  AS  subclass_table_id
  FROM  base_table  JOIN  subclass_table  ON  base_table.id  =  subclass_table.id
  )  AS  anon_1  ON  parent.id  =  anon_1.base_table_parent_id

-- sqlite only!
SELECT  parent.id  AS  parent_id,  anon_1.subclass_table_1_id  AS  subclass_table_1_id,
  anon_1.base_table_1_id  AS  base_table_1_id,
  anon_1.base_table_1_parent_id  AS  base_table_1_parent_id
FROM  parent  LEFT  OUTER  JOIN  (
  SELECT  base_table_1.id  AS  base_table_1_id,
  base_table_1.parent_id  AS  base_table_1_parent_id,
  subclass_table_1.id  AS  subclass_table_1_id
  FROM  base_table  AS  base_table_1
  JOIN  subclass_table  AS  subclass_table_1  ON  base_table_1.id  =  subclass_table_1.id
)  AS  anon_1  ON  parent.id  =  anon_1.base_table_1_parent_id

-- sqlite only!
SELECT  "order".id  AS  order_id
FROM  "order"  LEFT  OUTER  JOIN  (
  SELECT  order_item_1.order_id  AS  order_item_1_order_id,
  order_item_1.item_id  AS  order_item_1_item_id,
  item.id  AS  item_id,  item.type  AS  item_type
FROM  order_item  AS  order_item_1
  JOIN  item  ON  item.id  =  order_item_1.item_id  AND  item.type  IN  (?)
)  AS  anon_1  ON  "order".id  =  anon_1.order_item_1_order_id
```

注意

从 SQLAlchemy 1.1 开始，此功能中存在的 SQLite 的解决方法将在检测到 SQLite 版本**3.7.16**或更高版本时自动禁用自身，因为 SQLite 已修复了对右嵌套连接的支持。

`Join.alias()`，`aliased()`和`with_polymorphic()`函数现在支持一个新参数`flat=True`，用于构建别名的连接表实体，而不嵌入到 SELECT 中。默认情况下，此标志未启用，以帮助向后兼容性 - 但现在可以将“多态”可选择地作为目标连接，而不生成任何子查询：

```py
employee_alias = with_polymorphic(Person, [Engineer, Manager], flat=True)

session.query(Company).join(Company.employees.of_type(employee_alias)).filter(
    or_(Engineer.primary_language == "python", Manager.manager_name == "dilbert")
)
```

生成（除了 SQLite）：

```py
SELECT  companies.company_id  AS  companies_company_id,  companies.name  AS  companies_name
FROM  companies  JOIN  (
  people  AS  people_1
  LEFT  OUTER  JOIN  engineers  AS  engineers_1  ON  people_1.person_id  =  engineers_1.person_id
  LEFT  OUTER  JOIN  managers  AS  managers_1  ON  people_1.person_id  =  managers_1.person_id
)  ON  companies.company_id  =  people_1.company_id
WHERE  engineers.primary_language  =  %(primary_language_1)s
  OR  managers.manager_name  =  %(manager_name_1)s
```

[#2369](https://www.sqlalchemy.org/trac/ticket/2369) [#2587](https://www.sqlalchemy.org/trac/ticket/2587)  ### 可在连接的急切加载中使用右嵌套内连接

从版本 0.9.4 开始，在连接的急切加载情况下，可以启用上述提到的右嵌套连接，其中“外部”连接与右侧的“内部”连接相关联。

通常，像下面这样的连接急切加载链：

```py
query(User).options(
    joinedload("orders", innerjoin=False).joinedload("items", innerjoin=True)
)
```

不会产生内连接；由于从 user->order 的 LEFT OUTER JOIN，连接的急切加载无法使用从 order->items 到 INNER join，而不更改返回的用户行，并且会忽略“链接”`innerjoin=True`指令。0.9.0 应该交付的是，而不是：

```py
FROM  users  LEFT  OUTER  JOIN  orders  ON  <onclause>  LEFT  OUTER  JOIN  items  ON  <onclause>
```

新的“右嵌套连接是可以的”逻辑将启动，我们将得到：

```py
FROM  users  LEFT  OUTER  JOIN  (orders  JOIN  items  ON  <onclause>)  ON  <onclause>
```

由于我们错过了这一点，为了避免进一步的退化，我们通过向`joinedload.innerjoin`指定字符串`"nested"`来添加上述功能：

```py
query(User).options(
    joinedload("orders", innerjoin=False).joinedload("items", innerjoin="nested")
)
```

此功能是 0.9.4 中的新功能。

[#2976](https://www.sqlalchemy.org/trac/ticket/2976)

### ORM 可以使用 RETURNING 高效地获取刚生成的 INSERT/UPDATE 默认值

`Mapper` 长期以来一直支持一个名为 `eager_defaults=True` 的未记录的标志。此标志的作用是，当进行 INSERT 或 UPDATE 操作时，如果知道行具有由服务器生成的默认值，则会立即跟随一个 SELECT 来“急切地”加载这些新值。通常，服务器生成的列会在对象上标记为“过期”，因此除非应用程序在刷新后立即访问这些列，否则不会产生任何开销。因此，`eager_defaults` 标志并不是很有用，因为它只会降低性能，并且只存在于支持需要默认值在刷新过程中立即可用的奇特事件方案的情况下。

在 0.9 版本中，由于版本 ID 的增强，`eager_defaults` 现在可以为这些值发出 RETURNING 子句，因此在具有强大 RETURNING 支持的后端，特别是 PostgreSQL 上，ORM 可以在 INSERT 或 UPDATE 中内联获取新生成的默认和 SQL 表达式值。当启用 `eager_defaults` 时，将自动使用 RETURNING，当目标后端和 `Table` 支持“隐式返回”时。

### 对于某些查询，子查询预加载将在最内层的 SELECT 上应用 DISTINCT

在涉及到一对多关系时，子查询预加载可能会生成重复行的数量，因此当连接目标列不包含主键时，会对最内层的 SELECT 应用 DISTINCT 关键字，例如在沿着一对多加载时。

也就是说，在从 A->B 的一对多子查询加载时：

```py
SELECT  b.id  AS  b_id,  b.name  AS  b_name,  anon_1.b_id  AS  a_b_id
FROM  (SELECT  DISTINCT  a_b_id  FROM  a)  AS  anon_1
JOIN  b  ON  b.id  =  anon_1.a_b_id
```

由于 `a.b_id` 是一个非唯一的外键，所以会应用 DISTINCT，以消除冗余的 `a.b_id`。此行为可以通过为特定的 `relationship()` 设置标志 `distinct_target_key` 来无条件地打开或关闭，将值设置为 `True` 表示无条件打开，`False` 表示无条件关闭，`None` 表示当目标 SELECT 针对不包含完整主键的列时生效。在 0.9 版本中，`None` 是默认值。

这个选项也被回溯到了 0.8 版本，其中 `distinct_target_key` 选项的默认值为 `False`。

虽然此处的功能旨在通过消除重复行来帮助性能，但 SQL 中的 DISTINCT 关键字本身可能会对性能产生负面影响。如果 SELECT 中的列没有索引，则 DISTINCT 可能会对行集执行 ORDER BY，这可能会很昂贵。通过将此功能限制在外键上，希望外键无论如何都已被索引，可以预期新的默认值是合理的。

该功能也不能消除每种可能的重复行情况；如果在连接链中的其他地方存在多对一关系，重复行可能仍然存在。

[#2836](https://www.sqlalchemy.org/trac/ticket/2836)  ### 反向引用处理程序现在可以传播多于一级的深度

属性事件传递其“发起者”的机制已经发生了变化；不再传递`AttributeImpl`，而是传递一个新的对象`Event`；该对象引用`AttributeImpl`以及一个“操作令牌”，表示操作是追加、移除还是替换操作。

属性事件系统不再查看这个“发起者”对象以阻止递归系列的属性事件。相反，防止由于相互依赖的反向引用处理程序而导致无限递归的系统已经移动到了 ORM 反向引用事件处理程序中，这些处理程序现在负责确保相互依赖事件链（例如向集合 A.bs 追加，响应中设置多对一属性 B.a）不会进入无限递归流。这里的理念是，反向引用系统，通过更详细和控制事件传播，最终可以允许超过一级深度的操作发生；典型情况是集合追加导致多对一替换操作，进而应导致项目从先前的集合中移除的情况：

```py
class Parent(Base):
    __tablename__ = "parent"

    id = Column(Integer, primary_key=True)
    children = relationship("Child", backref="parent")

class Child(Base):
    __tablename__ = "child"

    id = Column(Integer, primary_key=True)
    parent_id = Column(ForeignKey("parent.id"))

p1 = Parent()
p2 = Parent()
c1 = Child()

p1.children.append(c1)

assert c1.parent is p1  # backref event establishes c1.parent as p1

p2.children.append(c1)

assert c1.parent is p2  # backref event establishes c1.parent as p2
assert c1 not in p1.children  # second backref event removes c1 from p1.children
```

在此更改之前，`c1`对象仍然会存在于`p1.children`中，即使它同时也存在于`p2.children`中；反向引用处理程序会停止在用`p2`替换`c1.parent`而不是`p1`的操作上。在 0.9 版本中，使用更详细的`Event`对象以及让反向引用处理程序对这些对象做出更详细的决策，传播可以继续进行，从而将`c1`从`p1.children`中移除，同时保持检查以防止传播进入无限递归循环。

使用 `AttributeEvents.set()`、`AttributeEvents.append()` 或 `AttributeEvents.remove()` 事件的终端用户代码可能需要修改，以防止递归循环，因为在缺少反向引用事件处理程序的情况下，属性系统不再阻止事件链无限传播。此外，依赖于 `initiator` 值的代码将需要调整到新的 API，并且还必须准备好 `initiator` 值在一系列反向引用引发的事件中从其原始值更改，因为现在反向引用处理程序可能会为某些操作交换一个新的 `initiator` 值。

[#2789](https://www.sqlalchemy.org/trac/ticket/2789)  ### 类型系统现在处理呈现“字面绑定”值的任务

`TypeEngine` 和 `TypeDecorator` 分别添加了新的方法 `TypeEngine.literal_processor()` 和 `TypeDecorator.process_literal_param()`，它们负责呈现所谓的“内联字面参数” - 由于编译器配置的原因，这些参数通常呈现为“绑定”值，但实际上是被内联呈现到 SQL 语句中。此功能用于生成诸如 `CheckConstraint` 这样的结构的 DDL，以及当使用诸如 `op.inline_literal()` 这样的结构时，Alembic 会使用它。以前，一个简单的“isinstance”检查检查了一些基本类型，并且“绑定处理器”被无条件使用，导致字符串过早编码为 utf-8 的问题。

使用 `TypeDecorator` 编写的自定义类型应继续在“内联文字”场景中工作，因为 `TypeDecorator.process_literal_param()` 默认会回退到 `TypeDecorator.process_bind_param()`，因为这些方法通常处理的是数据操作，而不是数据如何呈现给数据库。`TypeDecorator.process_literal_param()` 可以被指定为明确产生一个表示如何将值渲染成内联 DDL 语句的字符串。

[#2838](https://www.sqlalchemy.org/trac/ticket/2838)  ### 模式标识符现在携带其自身的引号信息

此更改简化了 Core 对所谓的“引号”标志的使用，比如传递给 `Table` 和 `Column` 的 `quote` 标志。该标志现在内部化在字符串名称本身中，现在表示为 `quoted_name` 的一个实例，即一个字符串子类。 `IdentifierPreparer` 现在仅依赖于由 `quoted_name` 对象报告的引号首选项，而不是在大多数情况下检查任何显式的 `quote` 标志。这里解决的问题包括，各种区分大小写的方法（例如 `Engine.has_table()` 以及方言内部的类似方法）现在能够以显式引号的名称正确地运行，而不需要复杂化或引入与引号标志的细节不兼容的更改到这些 API（其中许多是第三方的） - 特别是，更广泛范围的标识符现在能够与所谓的“大写”后端（像 Oracle、Firebird 和 DB2 这样的后端）正确地工作，这些后端使用全部大写存储和报告不区分大小写的名称的表和列名。

内部根据需要使用 `quoted_name` 对象；但是，如果其他关键字需要固定的引号首选项，则该类是公开可用的。

[#2812](https://www.sqlalchemy.org/trac/ticket/2812)  ### 改进布尔常量、NULL 常量、连接词的渲染

新功能已添加到 `true()` 和 `false()` 常量中，特别是与 `and_()` 和 `or_()` 函数以及 WHERE/HAVING 子句与这些类型、整体布尔类型和 `null()` 常量的行为结合使用时。

从这样的表开始：

```py
from sqlalchemy import Table, Boolean, Integer, Column, MetaData

t1 = Table("t", MetaData(), Column("x", Boolean()), Column("y", Integer))
```

在不具有 `true`/`false` 常量行为的后端上，选择构造现在将布尔列渲染为二进制表达式：

```py
>>> from sqlalchemy import select, and_, false, true
>>> from sqlalchemy.dialects import mysql, postgresql

>>> print(select([t1]).where(t1.c.x).compile(dialect=mysql.dialect()))
SELECT  t.x,  t.y  FROM  t  WHERE  t.x  =  1 
```

`and_()` 和 `or_()` 构造现在将表现出准“短路”行为，即在存在 `true()` 或 `false()` 常量时截断渲染表达式：

```py
>>> print(
...     select([t1]).where(and_(t1.c.y > 5, false())).compile(dialect=postgresql.dialect())
... )
SELECT  t.x,  t.y  FROM  t  WHERE  false 
```

`true()` 可以用作构建表达式的基础：

```py
>>> expr = true()
>>> expr = expr & (t1.c.y > 5)
>>> print(select([t1]).where(expr))
SELECT  t.x,  t.y  FROM  t  WHERE  t.y  >  :y_1 
```

布尔常量 `true()` 和 `false()` 本身渲染为 `0 = 1` 和 `1 = 1`，对于没有布尔常量的后端：

```py
>>> print(select([t1]).where(and_(t1.c.y > 5, false())).compile(dialect=mysql.dialect()))
SELECT  t.x,  t.y  FROM  t  WHERE  0  =  1 
```

`None` 的解释，虽然不是特别有效的 SQL，但至少现在是一致的：

```py
>>> print(select([t1.c.x]).where(None))
SELECT  t.x  FROM  t  WHERE  NULL
>>> print(select([t1.c.x]).where(None).where(None))
SELECT  t.x  FROM  t  WHERE  NULL  AND  NULL
>>> print(select([t1.c.x]).where(and_(None, None)))
SELECT  t.x  FROM  t  WHERE  NULL  AND  NULL 
```

[#2804](https://www.sqlalchemy.org/trac/ticket/2804)  ### 标签构造现在可以��� ORDER BY 中仅呈现为其名称

在 SELECT 的列子句和 ORDER BY 子句中都使用 `Label` 的情况下，标签将仅在 ORDER BY 子句中呈现为其名称，假设底层方言报告支持此功能。

例如，像这样的示例：

```py
from sqlalchemy.sql import table, column, select, func

t = table("t", column("c1"), column("c2"))
expr = (func.foo(t.c.c1) + t.c.c2).label("expr")

stmt = select([expr]).order_by(expr)

print(stmt)
```

在 0.9 之前将渲染为：

```py
SELECT  foo(t.c1)  +  t.c2  AS  expr
FROM  t  ORDER  BY  foo(t.c1)  +  t.c2
```

现在将渲染为：

```py
SELECT  foo(t.c1)  +  t.c2  AS  expr
FROM  t  ORDER  BY  expr
```

ORDER BY 仅在标签未在 ORDER BY 中进一步嵌入到表达式中时呈现标签，除了简单的 `ASC` 或 `DESC`。

上述格式在所有经过测试的数据库上都有效，但可能与旧数据库版本（MySQL 4？Oracle 8？等）存在兼容性问题。根据用户报告，我们可以添加规则，根据数据库版本检测禁用该功能。

[#1068](https://www.sqlalchemy.org/trac/ticket/1068)  ### `RowProxy`现在具有元组排序行为

`RowProxy`对象的行为很像一个元组，但直到现在，如果使用`sorted()`对它们的列表进行排序，它们不会像元组一样排序。现在，`__eq__()`方法将两侧都作为元组进行比较，并且还添加了一个`__lt__()`方法：

```py
users.insert().execute(
    dict(user_id=1, user_name="foo"),
    dict(user_id=2, user_name="bar"),
    dict(user_id=3, user_name="def"),
)

rows = users.select().order_by(users.c.user_name).execute().fetchall()

eq_(rows, [(2, "bar"), (3, "def"), (1, "foo")])

eq_(sorted(rows), [(1, "foo"), (2, "bar"), (3, "def")])
```

[#2848](https://www.sqlalchemy.org/trac/ticket/2848)  ### 当`bindparam()`构造没有类型时，通过复制进行升级，当有类型可用时

“升级”`bindparam()`构造以采用封闭表达式类型的逻辑已经通过两种方式得到改进。首先，在分配新类型之前，`bindparam()`对象会被**复制**，以便给定的`bindparam()`不会就地突变。其次，在编译`Insert`或`Update`构造时，通过`ValuesBase.values()`方法设置的“values”在语句中也会发生相同的操作。

如果给定一个无类型的`bindparam()`:

```py
bp = bindparam("some_col")
```

如果我们像下面这样使用这个参数：

```py
expr = mytable.c.col == bp
```

`bp`的类型仍然是`NullType`，但是如果`mytable.c.col`的类型是`String`，那么二元表达式的右侧`expr.right`将采用`String`类型。以前，`bp`本身会被就地更改为具有`String`类型。

同样，这个操作也会在`Insert`或`Update`中发生：

```py
stmt = mytable.update().values(col=bp)
```

在上面的例子中，`bp`保持不变，但在执行语句时将使用`String`类型，我们可以通过检查`binds`字典来看到这一点：

```py
>>> compiled = stmt.compile()
>>> compiled.binds["some_col"].type
String
```

该功能允许自定义类型在 INSERT/UPDATE 语句中产生预期效果，而无需在每个`bindparam()`表达式中显式指定这些类型。

可能向后兼容的更改涉及两种不太可能的情况。由于绑定参数是**克隆**的，用户不应该依赖于对一旦创建的`bindparam()`构造进行就地更改。此外，使用`bindparam()`在依赖于`bindparam()`未根据分配给的列进行类型化的事实的`Insert`或`Update`语句的代码将不再以这种方式运行。

[#2850](https://www.sqlalchemy.org/trac/ticket/2850)  ### 列可以可靠地从通过外键引用的列中获取其类型

有一个长期存在的行为，即可以声明没有类型的`Column`，只要该`Column`被`ForeignKeyConstraint`引用，并且引用列的类型将被复制到此列中。问题在于这个功能从来没有很好地工作过，也没有得到维护。核心问题是`ForeignKey`对象不知道它引用的目标`Column`是什么，直到被询问，通常是第一次使用外键来构造`Join`时。因此，在那个时候，父`Column`将没有类型，或更具体地说，它将具有默认类型`NullType`。

尽管花费了很长时间，重新组织`ForeignKey`对象初始化的工作已经完成，使得这个功能最终可以正常工作。这个改变的核心是`ForeignKey.column`属性不再延迟初始化目标`Column`的位置；这个系统的问题在于拥有的`Column`会一直被固定为`NullType`类型，直到`ForeignKey`被使用。

在新版本中，`ForeignKey`通过内部附加事件与最终引用的`Column`协调，因此一旦引用的`Column`与`MetaData`关联，所有引用它的`ForeignKey`对象都会收到一条消息，告诉它们需要初始化其父列。这个系统更加复杂，但更加稳固；作为奖励，现在已经为各种`Column` / `ForeignKey`配置场景设置了测试，并且错误消息已经改进，对不少于七种不同的错误条件进行了非常具体的描述。

现在正确工作的场景包括：

1.  当目标`Column`与相同的`MetaData`关联时，`Column`上的类型会立即出现；无论哪一边先配置都可以：

    ```py
    >>> from sqlalchemy import Table, MetaData, Column, Integer, ForeignKey
    >>> metadata = MetaData()
    >>> t2 = Table("t2", metadata, Column("t1id", ForeignKey("t1.id")))
    >>> t2.c.t1id.type
    NullType()
    >>> t1 = Table("t1", metadata, Column("id", Integer, primary_key=True))
    >>> t2.c.t1id.type
    Integer()
    ```

1.  系统现在也可以使用`ForeignKeyConstraint`：

    ```py
    >>> from sqlalchemy import Table, MetaData, Column, Integer, ForeignKeyConstraint
    >>> metadata = MetaData()
    >>> t2 = Table(
    ...     "t2",
    ...     metadata,
    ...     Column("t1a"),
    ...     Column("t1b"),
    ...     ForeignKeyConstraint(["t1a", "t1b"], ["t1.a", "t1.b"]),
    ... )
    >>> t2.c.t1a.type
    NullType()
    >>> t2.c.t1b.type
    NullType()
    >>> t1 = Table(
    ...     "t1",
    ...     metadata,
    ...     Column("a", Integer, primary_key=True),
    ...     Column("b", Integer, primary_key=True),
    ... )
    >>> t2.c.t1a.type
    Integer()
    >>> t2.c.t1b.type
    Integer()
    ```

1.  它甚至适用于“多跳” - 即，一个指向另一个`Column`的`Column`的`ForeignKey`:

    ```py
    >>> from sqlalchemy import Table, MetaData, Column, Integer, ForeignKey
    >>> metadata = MetaData()
    >>> t2 = Table("t2", metadata, Column("t1id", ForeignKey("t1.id")))
    >>> t3 = Table("t3", metadata, Column("t2t1id", ForeignKey("t2.t1id")))
    >>> t2.c.t1id.type
    NullType()
    >>> t3.c.t2t1id.type
    NullType()
    >>> t1 = Table("t1", metadata, Column("id", Integer, primary_key=True))
    >>> t2.c.t1id.type
    Integer()
    >>> t3.c.t2t1id.type
    Integer()
    ```

[#1765](https://www.sqlalchemy.org/trac/ticket/1765)

## 方言更改

### Firebird `fdb` 现在是默认的 Firebird 方言。

如果创建引擎时没有指定方言，则现在使用 `fdb` 方言，即 `firebird://`。`fdb` 是一个与 `kinterbasdb` 兼容的 DBAPI，根据 Firebird 项目的说法，现在是他们官方的 Python 驱动程序。

[#2504](https://www.sqlalchemy.org/trac/ticket/2504)

### Firebird `fdb` 和 `kinterbasdb` 默认设置 `retaining=False`

`fdb` 和 `kinterbasdb` DBAPI 都支持一个标志 `retaining=True`，可以传递给其连接的 `commit()` 和 `rollback()` 方法。文档中对这个标志的理由是，DBAPI 可以重用内部事务状态进行后续事务，以提高性能。然而，更新的文档提到了 Firebird 的“垃圾回收”分析，表明这个标志可能对数据库处理清理任务的能力产生负面影响，并因此被报告为*降低*性能。

鉴于这些信息，目前不清楚这个标志实际上如何可用，而且由于它似乎只是一个性能增强功能，现在默认值为 `False`。可以通过向 `create_engine()` 调用传递标志 `retaining=True` 来控制该值。这是一个新标志，从 0.8.2 版本开始添加，因此在 0.8.2 版本的应用程序可以根据需要将其设置为 `True` 或 `False`。

另请参阅

`sqlalchemy.dialects.firebird.fdb`

`sqlalchemy.dialects.firebird.kinterbasdb`

[`pythonhosted.org/fdb/usage-guide.html#retaining-transactions`](https://pythonhosted.org/fdb/usage-guide.html#retaining-transactions) - 有关“保留”标志的信息。

[#2763](https://www.sqlalchemy.org/trac/ticket/2763)

## 介绍

本指南介绍了 SQLAlchemy 版本 0.9 中的新功能，并记录了影响用户将其应用程序从 SQLAlchemy 0.8 系列迁移到 0.9 的变化。

请仔细查看行为变化 - ORM 和行为变化 - Core，可能存在不兼容的变化。

## 平台支持

### 现在针对 Python 2.6 及更高版本，Python 3 不再需要 2to3

第一个 0.9 版本的成就是移除了对 Python 3 兼容性的 2to3 工具的依赖。为了使这更加直观，现在目标最低的 Python 发布版本是 2.6，它具有与 Python 3 广泛的交叉兼容性。所有 SQLAlchemy 模块和单元测试现在都能在任何从 2.6 开始的 Python 解释器上同样良好地解释，包括 3.1 和 3.2 解释器。

[#2671](https://www.sqlalchemy.org/trac/ticket/2671)

### C 扩展在 Python 3 上受支持

C 扩展已被移植以支持 Python 3，并且现在在 Python 2 和 Python 3 环境中都构建。

[#2161](https://www.sqlalchemy.org/trac/ticket/2161)

### 现在目标是 Python 2.6 及更高版本，Python 3 不再需要 2to3

第一个 0.9 版本的成就是移除了对 Python 3 兼容性的 2to3 工具的依赖。为了使这更加直观，现在目标最低的 Python 发布版本是 2.6，它具有与 Python 3 广泛的交叉兼容性。所有 SQLAlchemy 模块和单元测试现在都能在任何从 2.6 开始的 Python 解释器上同样良好地解释，包括 3.1 和 3.2 解释器。

[#2671](https://www.sqlalchemy.org/trac/ticket/2671)

### C 扩展在 Python 3 上受支持

C 扩展已被移植以支持 Python 3，并且现在在 Python 2 和 Python 3 环境中都构建。

[#2161](https://www.sqlalchemy.org/trac/ticket/2161)

## 行为变更 - ORM

### 当按属性查询时，现在会以它们的对象形式返回复合属性

现在，使用 `Query` 与复合属性一起，会返回该复合属性维护的对象类型，而不是拆分为各个列。使用在 复合列类型 中设置的映射：

```py
>>> session.query(Vertex.start, Vertex.end).filter(Vertex.start == Point(3, 4)).all()
[(Point(x=3, y=4), Point(x=5, y=6))]
```

此更改与期望将各个属性扩展为各个列的代码不兼容。要获得该行为，请使用 `.clauses` 访问器：

```py
>>> session.query(Vertex.start.clauses, Vertex.end.clauses).filter(
...     Vertex.start == Point(3, 4)
... ).all()
[(3, 4, 5, 6)]
```

另请参阅

ORM 查询的列捆绑

[#2824](https://www.sqlalchemy.org/trac/ticket/2824)  ### `Query.select_from()` 不再将该子句应用于对应的实体

近期版本中，`Query.select_from()` 方法已被广泛使用，作为控制 `Query` 对象“选择的第一件事”的手段，通常是为了控制 JOIN 如何渲染。

与通常的 `User` 映射相比，请考虑以下示例：

```py
select_stmt = select([User]).where(User.id == 7).alias()

q = (
    session.query(User)
    .join(select_stmt, User.id == select_stmt.c.id)
    .filter(User.name == "ed")
)
```

上述声明可预期地渲染为以下 SQL：

```py
SELECT  "user".id  AS  user_id,  "user".name  AS  user_name
FROM  "user"  JOIN  (SELECT  "user".id  AS  id,  "user".name  AS  name
FROM  "user"
WHERE  "user".id  =  :id_1)  AS  anon_1  ON  "user".id  =  anon_1.id
WHERE  "user".name  =  :name_1
```

如果我们想要颠倒 JOIN 的左右元素的顺序，文档会让我们相信可以使用`Query.select_from()`来实现：

```py
q = (
    session.query(User)
    .select_from(select_stmt)
    .join(User, User.id == select_stmt.c.id)
    .filter(User.name == "ed")
)
```

然而，在 0.8 版本及更早版本中，上述对`Query.select_from()`的使用会将`select_stmt`应用于**替换**`User`实体，因为它选择了与`User`兼容的`user`表：

```py
-- SQLAlchemy 0.8 and earlier...
SELECT  anon_1.id  AS  anon_1_id,  anon_1.name  AS  anon_1_name
FROM  (SELECT  "user".id  AS  id,  "user".name  AS  name
FROM  "user"
WHERE  "user".id  =  :id_1)  AS  anon_1  JOIN  "user"  ON  anon_1.id  =  anon_1.id
WHERE  anon_1.name  =  :name_1
```

上述语句很混乱，ON 子句引用了`anon_1.id = anon_1.id`，我们的 WHERE 子句也被`anon_1`替换了。

这种行为是完全有意的，但与已经流行的`Query.select_from()`的用例不同。上述行为现在可以通过一个名为`Query.select_entity_from()`的新方法实现。这是一个较少使用的行为，在现代的 SQLAlchemy 中大致相当于从自定义`aliased()`构造中选择：

```py
select_stmt = select([User]).where(User.id == 7)
user_from_stmt = aliased(User, select_stmt.alias())

q = session.query(user_from_stmt).filter(user_from_stmt.name == "ed")
```

因此，在 SQLAlchemy 0.9 中，我们从`select_stmt`选择的查询产生了我们期望的 SQL：

```py
-- SQLAlchemy 0.9
SELECT  "user".id  AS  user_id,  "user".name  AS  user_name
FROM  (SELECT  "user".id  AS  id,  "user".name  AS  name
FROM  "user"
WHERE  "user".id  =  :id_1)  AS  anon_1  JOIN  "user"  ON  "user".id  =  id
WHERE  "user".name  =  :name_1
```

`Query.select_entity_from()` 方法将在 SQLAlchemy **0.8.2** 中可用，因此依赖旧行为的应用程序可以首先过渡到该方法，确保所有测试继续正常运行，然后无问题地升级到 0.9。

[#2736](https://www.sqlalchemy.org/trac/ticket/2736)  ### `viewonly=True` 在 `relationship()` 上阻止历史记录生效

在`relationship()`上的`viewonly`标志被应用于防止对目标属性的更改在刷新过程中产生任何影响。这是通过在刷新过程中排除属性来实现的。然而，直到现在，对属性的更改仍然会将父对象标记为“脏”，并触发潜在的刷新。更改是`viewonly`标志现在还阻止为目标属性设置历史记录。像反向引用和用户定义事件之类的属性事件仍然正常运行。

更改如下所示：

```py
from sqlalchemy import Column, Integer, ForeignKey, create_engine
from sqlalchemy.orm import backref, relationship, Session
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy import inspect

Base = declarative_base()

class A(Base):
    __tablename__ = "a"
    id = Column(Integer, primary_key=True)

class B(Base):
    __tablename__ = "b"

    id = Column(Integer, primary_key=True)
    a_id = Column(Integer, ForeignKey("a.id"))
    a = relationship("A", backref=backref("bs", viewonly=True))

e = create_engine("sqlite://")
Base.metadata.create_all(e)

a = A()
b = B()

sess = Session(e)
sess.add_all([a, b])
sess.commit()

b.a = a

assert b in sess.dirty

# before 0.9.0
# assert a in sess.dirty
# assert inspect(a).attrs.bs.history.has_changes()

# after 0.9.0
assert a not in sess.dirty
assert not inspect(a).attrs.bs.history.has_changes()
```

[#2833](https://www.sqlalchemy.org/trac/ticket/2833)  ### 关联代理 SQL 表达式改进和修复

现在，通过关联代理实现的`==`和`!=`运算符，引用标量关系上的标量值，现在产生更完整的 SQL 表达式，旨在考虑“关联”行是否存在，当比较为`None`时。

考虑这个映射：

```py
class A(Base):
    __tablename__ = "a"

    id = Column(Integer, primary_key=True)

    b_id = Column(Integer, ForeignKey("b.id"), primary_key=True)
    b = relationship("B")
    b_value = association_proxy("b", "value")

class B(Base):
    __tablename__ = "b"
    id = Column(Integer, primary_key=True)
    value = Column(String)
```

直到 0.8 版本，像下面这样的查询：

```py
s.query(A).filter(A.b_value == None).all()
```

会产生：

```py
SELECT  a.id  AS  a_id,  a.b_id  AS  a_b_id
FROM  a
WHERE  EXISTS  (SELECT  1
FROM  b
WHERE  b.id  =  a.b_id  AND  b.value  IS  NULL)
```

在 0.9 中，现在产生：

```py
SELECT  a.id  AS  a_id,  a.b_id  AS  a_b_id
FROM  a
WHERE  (EXISTS  (SELECT  1
FROM  b
WHERE  b.id  =  a.b_id  AND  b.value  IS  NULL))  OR  a.b_id  IS  NULL
```

不同之处在于，它不仅检查`b.value`，还检查`a`是否根本没有引用任何`b`行。对于使用这种类型比较的系统，一些父行没有关联行，这将与先前版本产生不同的结果。

更为关键的是，对于`A.b_value != None`，会发出正确的表达式。在 0.8 版本中，对于没有`b`的`A`行，这将返回`True`：

```py
SELECT  a.id  AS  a_id,  a.b_id  AS  a_b_id
FROM  a
WHERE  NOT  (EXISTS  (SELECT  1
FROM  b
WHERE  b.id  =  a.b_id  AND  b.value  IS  NULL))
```

现在在 0.9 版本中，检查已经重新设计，以确保 A.b_id 行存在，另外`B.value`不为 NULL：

```py
SELECT  a.id  AS  a_id,  a.b_id  AS  a_b_id
FROM  a
WHERE  EXISTS  (SELECT  1
FROM  b
WHERE  b.id  =  a.b_id  AND  b.value  IS  NOT  NULL)
```

此外，`has()`操作符得到增强，使您可以只针对标量列值调用它，而不需要任何条件，它将生成检查关联行是否存在的条件：

```py
s.query(A).filter(A.b_value.has()).all()
```

输出：

```py
SELECT  a.id  AS  a_id,  a.b_id  AS  a_b_id
FROM  a
WHERE  EXISTS  (SELECT  1
FROM  b
WHERE  b.id  =  a.b_id)
```

这等同于`A.b.has()`，但允许直接针对`b_value`进行查询。

[#2751](https://www.sqlalchemy.org/trac/ticket/2751)  ### 关联代理缺失标量返回 None

从标量属性到标量的关联代理现在如果被代理的对象不存在将返回`None`。这与 SQLAlchemy 中缺失的一对多关联返回 None 的事实一致，因此代理值也应该如此。例如：

```py
from sqlalchemy import *
from sqlalchemy.orm import *
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.ext.associationproxy import association_proxy

Base = declarative_base()

class A(Base):
    __tablename__ = "a"

    id = Column(Integer, primary_key=True)
    b = relationship("B", uselist=False)

    bname = association_proxy("b", "name")

class B(Base):
    __tablename__ = "b"

    id = Column(Integer, primary_key=True)
    a_id = Column(Integer, ForeignKey("a.id"))
    name = Column(String)

a1 = A()

# this is how m2o's always have worked
assert a1.b is None

# but prior to 0.9, this would raise AttributeError,
# now returns None just like the proxied value.
assert a1.bname is None
```

[#2810](https://www.sqlalchemy.org/trac/ticket/2810)  ### attributes.get_history()如果值不存在，默认情况下将从数据库查询

有关`get_history()`的修复 bug 允许基于列的属性查询到数据库中未加载的值，假设`passive`标志保持默认的`PASSIVE_OFF`。以前，此标志将不被尊重。此外，添加了一个新方法`AttributeState.load_history()`来补充`AttributeState.history`属性，它将为未加载的属性发出加载器可调用。

这是一个小改变的演示如下：

```py
from sqlalchemy import Column, Integer, String, create_engine, inspect
from sqlalchemy.orm import Session, attributes
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()

class A(Base):
    __tablename__ = "a"
    id = Column(Integer, primary_key=True)
    data = Column(String)

e = create_engine("sqlite://", echo=True)
Base.metadata.create_all(e)

sess = Session(e)

a1 = A(data="a1")
sess.add(a1)
sess.commit()  # a1 is now expired

# history doesn't emit loader callables
assert inspect(a1).attrs.data.history == (None, None, None)

# in 0.8, this would fail to load the unloaded state.
assert attributes.get_history(a1, "data") == (
    (),
    [
        "a1",
    ],
    (),
)

# load_history() is now equivalent to get_history() with
# passive=PASSIVE_OFF ^ INIT_OK
assert inspect(a1).attrs.data.load_history() == (
    (),
    [
        "a1",
    ],
    (),
)
```

[#2787](https://www.sqlalchemy.org/trac/ticket/2787)  ### 当按属性基础查询时，复合属性现在以其对象形式返回

现在，与复合属性一起使用`Query`现在返回由该复合属性维护的对象类型，而不是分解为单独列。使用在复合列类型设置的映射：

```py
>>> session.query(Vertex.start, Vertex.end).filter(Vertex.start == Point(3, 4)).all()
[(Point(x=3, y=4), Point(x=5, y=6))]
```

这个改变与期望将单个属性扩展为单独列的代码不兼容。要获得该行为，请使用`.clauses`访问器：

```py
>>> session.query(Vertex.start.clauses, Vertex.end.clauses).filter(
...     Vertex.start == Point(3, 4)
... ).all()
[(3, 4, 5, 6)]
```

另请参阅

ORM 查询的列捆绑

[#2824](https://www.sqlalchemy.org/trac/ticket/2824)

### `Query.select_from()`不再将子句应用于相应的实体

近期版本中，`Query.select_from()`方法已经变得流行，作为控制`Query`对象“选择自”的一种方式，通常用于控制 JOIN 的渲染方式。

请考虑以下示例与通常的`User`映射相对比：

```py
select_stmt = select([User]).where(User.id == 7).alias()

q = (
    session.query(User)
    .join(select_stmt, User.id == select_stmt.c.id)
    .filter(User.name == "ed")
)
```

上述语句可预见地生成类似以下的 SQL：

```py
SELECT  "user".id  AS  user_id,  "user".name  AS  user_name
FROM  "user"  JOIN  (SELECT  "user".id  AS  id,  "user".name  AS  name
FROM  "user"
WHERE  "user".id  =  :id_1)  AS  anon_1  ON  "user".id  =  anon_1.id
WHERE  "user".name  =  :name_1
```

如果我们想要颠倒 JOIN 的左右元素的顺序，文档会让我们相信可以使用`Query.select_from()`来实现：

```py
q = (
    session.query(User)
    .select_from(select_stmt)
    .join(User, User.id == select_stmt.c.id)
    .filter(User.name == "ed")
)
```

然而，在 0.8 版本及之前，上述对`Query.select_from()`的使用会将`select_stmt`应用于**替换**`User`实体，因为它选择自与`User`兼容的`user`表：

```py
-- SQLAlchemy 0.8 and earlier...
SELECT  anon_1.id  AS  anon_1_id,  anon_1.name  AS  anon_1_name
FROM  (SELECT  "user".id  AS  id,  "user".name  AS  name
FROM  "user"
WHERE  "user".id  =  :id_1)  AS  anon_1  JOIN  "user"  ON  anon_1.id  =  anon_1.id
WHERE  anon_1.name  =  :name_1
```

上述语句是一团糟，ON 子句引用了`anon_1.id = anon_1.id`，我们的 WHERE 子句也被替换为`anon_1`。

这种行为是完全有意的，但与`Query.select_from()`变得流行的用例不同。上述行为现在可以通过一个名为`Query.select_entity_from()`的新方法来实现。这是一个较少使用的行为，在现代的 SQLAlchemy 中大致相当于从自定义的`aliased()`构造中选择：

```py
select_stmt = select([User]).where(User.id == 7)
user_from_stmt = aliased(User, select_stmt.alias())

q = session.query(user_from_stmt).filter(user_from_stmt.name == "ed")
```

因此，在 SQLAlchemy 0.9 中，我们的从`select_stmt`选择的查询会产生我们期望的 SQL：

```py
-- SQLAlchemy 0.9
SELECT  "user".id  AS  user_id,  "user".name  AS  user_name
FROM  (SELECT  "user".id  AS  id,  "user".name  AS  name
FROM  "user"
WHERE  "user".id  =  :id_1)  AS  anon_1  JOIN  "user"  ON  "user".id  =  id
WHERE  "user".name  =  :name_1
```

`Query.select_entity_from()`方法将在 SQLAlchemy **0.8.2**中可用，因此依赖旧行为的应用程序可以首先过渡到这种方法，确保所有测试继续正常运行，然后无问题地升级到 0.9。

[#2736](https://www.sqlalchemy.org/trac/ticket/2736)

### 在`relationship()`上使用`viewonly=True`会阻止历史记录生效

在`relationship()`上的`viewonly`标志被应用以防止对目标属性的更改在刷新过程中产生任何影响。这是通过在刷新过程中不考虑该属性来实现的。然而，直到现在，对属性的更改仍会将父对象注册为“脏”，并触发潜在的刷新。改变是，`viewonly`标志现在也阻止为目标属性设置历史记录。像反向引用和用户定义事件这样的属性事件仍然正常运行。

改变如下所示：

```py
from sqlalchemy import Column, Integer, ForeignKey, create_engine
from sqlalchemy.orm import backref, relationship, Session
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy import inspect

Base = declarative_base()

class A(Base):
    __tablename__ = "a"
    id = Column(Integer, primary_key=True)

class B(Base):
    __tablename__ = "b"

    id = Column(Integer, primary_key=True)
    a_id = Column(Integer, ForeignKey("a.id"))
    a = relationship("A", backref=backref("bs", viewonly=True))

e = create_engine("sqlite://")
Base.metadata.create_all(e)

a = A()
b = B()

sess = Session(e)
sess.add_all([a, b])
sess.commit()

b.a = a

assert b in sess.dirty

# before 0.9.0
# assert a in sess.dirty
# assert inspect(a).attrs.bs.history.has_changes()

# after 0.9.0
assert a not in sess.dirty
assert not inspect(a).attrs.bs.history.has_changes()
```

[#2833](https://www.sqlalchemy.org/trac/ticket/2833)

### 关联代理 SQL 表达式改进和修复

通过一个关联代理实现的`==`和`!=`运算符，它引用标量关系上的标量值，现在会产生一个更完整的 SQL 表达式，旨在考虑“关联”行在与`None`比较时是否存在。

考虑以下映射：

```py
class A(Base):
    __tablename__ = "a"

    id = Column(Integer, primary_key=True)

    b_id = Column(Integer, ForeignKey("b.id"), primary_key=True)
    b = relationship("B")
    b_value = association_proxy("b", "value")

class B(Base):
    __tablename__ = "b"
    id = Column(Integer, primary_key=True)
    value = Column(String)
```

直到 0.8 版本，像下面这样的查询：

```py
s.query(A).filter(A.b_value == None).all()
```

将产生：

```py
SELECT  a.id  AS  a_id,  a.b_id  AS  a_b_id
FROM  a
WHERE  EXISTS  (SELECT  1
FROM  b
WHERE  b.id  =  a.b_id  AND  b.value  IS  NULL)
```

在 0.9 中，现在产生：

```py
SELECT  a.id  AS  a_id,  a.b_id  AS  a_b_id
FROM  a
WHERE  (EXISTS  (SELECT  1
FROM  b
WHERE  b.id  =  a.b_id  AND  b.value  IS  NULL))  OR  a.b_id  IS  NULL
```

不同之处在于，它不仅检查`b.value`，还检查`a`是否根本没有指向任何`b`行。这将与先前版本产生不同的结果，对于使用这种类型比较的系统，其中一些父行没有关联行。

更为关键的是，对于`A.b_value != None`，现在会生成正确的表达式。在 0.8 中，对于没有`b`的`A`行，这将返回`True`：

```py
SELECT  a.id  AS  a_id,  a.b_id  AS  a_b_id
FROM  a
WHERE  NOT  (EXISTS  (SELECT  1
FROM  b
WHERE  b.id  =  a.b_id  AND  b.value  IS  NULL))
```

现在在 0.9 版本中，检查已经重新设计，以确保`A.b_id`行存在，另外`B.value`不为 NULL：

```py
SELECT  a.id  AS  a_id,  a.b_id  AS  a_b_id
FROM  a
WHERE  EXISTS  (SELECT  1
FROM  b
WHERE  b.id  =  a.b_id  AND  b.value  IS  NOT  NULL)
```

此外，`has()`运算符得到增强，使您可以只针对标量列值调用它，而不需要任何条件，它将生成检查关联行是否存在的条件：

```py
s.query(A).filter(A.b_value.has()).all()
```

输出：

```py
SELECT  a.id  AS  a_id,  a.b_id  AS  a_b_id
FROM  a
WHERE  EXISTS  (SELECT  1
FROM  b
WHERE  b.id  =  a.b_id)
```

这等同于`A.b.has()`，但允许直接针对`b_value`进行查询。

[#2751](https://www.sqlalchemy.org/trac/ticket/2751)

### 关联代理缺失标量返回 None

从标量属性到标量的关联代理现在如果代理对象不存在将返回`None`。这与 SQLAlchemy 中缺少多对一关系返回 None 的事实一致，所以代理值也应该如此。例如：

```py
from sqlalchemy import *
from sqlalchemy.orm import *
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.ext.associationproxy import association_proxy

Base = declarative_base()

class A(Base):
    __tablename__ = "a"

    id = Column(Integer, primary_key=True)
    b = relationship("B", uselist=False)

    bname = association_proxy("b", "name")

class B(Base):
    __tablename__ = "b"

    id = Column(Integer, primary_key=True)
    a_id = Column(Integer, ForeignKey("a.id"))
    name = Column(String)

a1 = A()

# this is how m2o's always have worked
assert a1.b is None

# but prior to 0.9, this would raise AttributeError,
# now returns None just like the proxied value.
assert a1.bname is None
```

[#2810](https://www.sqlalchemy.org/trac/ticket/2810)

### attributes.get_history()如果值不存在将默认从数据库查询

有关`get_history()`的修复允许基于列的属性查询数据库中未加载的值，假设`passive`标志保持默认值`PASSIVE_OFF`。以前，这个标志不会被遵守。此外，新增了一个新方法`AttributeState.load_history()`来补充`AttributeState.history`属性，该属性将为未加载的属性发出加载器可调用。

这是一个小改变的示例：

```py
from sqlalchemy import Column, Integer, String, create_engine, inspect
from sqlalchemy.orm import Session, attributes
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()

class A(Base):
    __tablename__ = "a"
    id = Column(Integer, primary_key=True)
    data = Column(String)

e = create_engine("sqlite://", echo=True)
Base.metadata.create_all(e)

sess = Session(e)

a1 = A(data="a1")
sess.add(a1)
sess.commit()  # a1 is now expired

# history doesn't emit loader callables
assert inspect(a1).attrs.data.history == (None, None, None)

# in 0.8, this would fail to load the unloaded state.
assert attributes.get_history(a1, "data") == (
    (),
    [
        "a1",
    ],
    (),
)

# load_history() is now equivalent to get_history() with
# passive=PASSIVE_OFF ^ INIT_OK
assert inspect(a1).attrs.data.load_history() == (
    (),
    [
        "a1",
    ],
    (),
)
```

[#2787](https://www.sqlalchemy.org/trac/ticket/2787)

## 行为变化 - 核心

### 类型对象不再接受被忽略的关键字参数

在 0.8 系列之前，大多数类型对象接受任意关键字参数，这些参数会被静默忽略：

```py
from sqlalchemy import Date, Integer

# storage_format argument here has no effect on any backend;
# it needs to be on the SQLite-specific type
d = Date(storage_format="%(day)02d.%(month)02d.%(year)04d")

# display_width argument here has no effect on any backend;
# it needs to be on the MySQL-specific type
i = Integer(display_width=5)
```

这是一个非常古老的 bug，为此在 0.8 系列中添加了一个弃用警告，但因为没有人会使用“-W”标志来运行 Python，所以几乎从未被看到：

```py
$ python -W always::DeprecationWarning ~/dev/sqlalchemy/test.py
/Users/classic/dev/sqlalchemy/test.py:5: SADeprecationWarning: Passing arguments to
type object constructor <class 'sqlalchemy.types.Date'> is deprecated
  d = Date(storage_format="%(day)02d.%(month)02d.%(year)04d")
/Users/classic/dev/sqlalchemy/test.py:9: SADeprecationWarning: Passing arguments to
type object constructor <class 'sqlalchemy.types.Integer'> is deprecated
  i = Integer(display_width=5)
```

从 0.9 系列开始，`TypeEngine`中的“catch all”构造函数被移除，这些无意义的参数不再被接受。

利用方言特定参数如`storage_format`和`display_width`的正确方式是使用适当的方言特定类型：

```py
from sqlalchemy.dialects.sqlite import DATE
from sqlalchemy.dialects.mysql import INTEGER

d = DATE(storage_format="%(day)02d.%(month)02d.%(year)04d")

i = INTEGER(display_width=5)
```

那么当我们也想要方言无关的类型时怎么办？我们使用`TypeEngine.with_variant()`方法：

```py
from sqlalchemy import Date, Integer
from sqlalchemy.dialects.sqlite import DATE
from sqlalchemy.dialects.mysql import INTEGER

d = Date().with_variant(
    DATE(storage_format="%(day)02d.%(month)02d.%(year)04d"), "sqlite"
)

i = Integer().with_variant(INTEGER(display_width=5), "mysql")
```

`TypeEngine.with_variant()`并不是新功能，它是在 SQLAlchemy 0.7.2 中添加的。因此，在 0.8 系列上运行的代码可以校正为使用这种方法，并在升级到 0.9 之前进行测试。

### `None`不再能被用作“部分 AND”构造函数

`None`不再能被用作“后备”来逐步形成 AND 条件。即使一些 SQLAlchemy 内部使用了这种模式，这种模式也没有被记录在案：

```py
condition = None

for cond in conditions:
    condition = condition & cond

if condition is not None:
    stmt = stmt.where(condition)
```

当`conditions`不为空时，上述序列在 0.9 上会产生`SELECT .. WHERE <condition> AND NULL`。`None`不再被隐式忽略，而是与在其他上下文中解释`None`时保持一致。

0.8 和 0.9 的正确代码应该是：

```py
from sqlalchemy.sql import and_

if conditions:
    stmt = stmt.where(and_(*conditions))
```

另一个在 0.9 上适用于所有后端的变体，在 0.8 上只适用于支持布尔常量的后端：

```py
from sqlalchemy.sql import true

condition = true()

for cond in conditions:
    condition = cond & condition

stmt = stmt.where(condition)
```

在 0.8 版本中，这将生成一个 SELECT 语句，其 WHERE 子句中始终包含`AND true`，这不被不支持布尔常量的后端所接受（MySQL、MSSQL）。在 0.9 版本中，`true`常量将在`and_()`连接中被删除。

另请参见

改进的布尔常量、NULL 常量、连接词的呈现方式

### `create_engine()` 的“password”部分不再将`+`号视为编码空格。

由于某种原因，Python 函数`unquote_plus()`被应用于 URL 的`password`字段，这是对[RFC 1738](https://www.ietf.org/rfc/rfc1738.txt)中描述的编码规则的错误应用，因为它将空格转义为加号。现在 URL 的字符串化仅编码“:”、“@”或“/”，不再应用于`username`和`password`字段（以前仅应用于密码）。在解析时，编码字符被转换，但加号和空格保持不变：

```py
# password: "pass word + other:words"
dbtype://user:pass word + other%3Awords@host/dbname

# password: "apples/oranges"
dbtype://username:apples%2Foranges@hostspec/database

# password: "apples@oranges@@"
dbtype://username:apples%40oranges%40%40@hostspec/database

# password: '', username is "username@"
dbtype://username%40:@hostspec/database
```

[#2873](https://www.sqlalchemy.org/trac/ticket/2873)  ### COLLATE 的优先规则已更改

以前，类似以下的表达式：

```py
print((column("x") == "somevalue").collate("en_EN"))
```

将会产生如下表达式：

```py
-- 0.8 behavior
(x  =  :x_1)  COLLATE  en_EN
```

上述内容被 MSSQL 误解，通常不是任何数据库建议的语法。该表达式现在将产生大多数数据库文档所示的语法：

```py
-- 0.9 behavior
x  =  :x_1  COLLATE  en_EN
```

如果`ColumnOperators.collate()` 操作符被应用于右侧列，则可能会出现潜在的不兼容更改：

```py
print(column("x") == literal("somevalue").collate("en_EN"))
```

在 0.8 版本中，会产生：

```py
x  =  :param_1  COLLATE  en_EN
```

然而在 0.9 版本中，将会产生更准确的，但可能不是您想要的形式：

```py
x  =  (:param_1  COLLATE  en_EN)
```

`ColumnOperators.collate()` 操作符现在在`ORDER BY`表达式中更为适当地工作，因为`ASC`和`DESC`操作符已被赋予特定的优先级，这将再次确保不会生成括号：

```py
>>> # 0.8
>>> print(column("x").collate("en_EN").desc())
(x  COLLATE  en_EN)  DESC
>>> # 0.9
>>> print(column("x").collate("en_EN").desc())
x  COLLATE  en_EN  DESC 
```

[#2879](https://www.sqlalchemy.org/trac/ticket/2879)  ### PostgreSQL CREATE TYPE <x> AS ENUM 现在对值应用引号

`ENUM` 类型现在将对枚举值中的单引号符号进行转义：

```py
>>> from sqlalchemy.dialects import postgresql
>>> type = postgresql.ENUM("one", "two", "three's", name="myenum")
>>> from sqlalchemy.dialects.postgresql import base
>>> print(base.CreateEnumType(type).compile(dialect=postgresql.dialect()))
CREATE  TYPE  myenum  AS  ENUM  ('one','two','three''s') 
```

已经转义单引号符号的现有解决方法需要进行修改，否则它们现在将会双重转义。

[#2878](https://www.sqlalchemy.org/trac/ticket/2878)

### 类型对象不再接受被忽略的关键字参数

直到 0.8 系列，大多数类型对象接受任意关键字参数，这些参数被静默忽略：

```py
from sqlalchemy import Date, Integer

# storage_format argument here has no effect on any backend;
# it needs to be on the SQLite-specific type
d = Date(storage_format="%(day)02d.%(month)02d.%(year)04d")

# display_width argument here has no effect on any backend;
# it needs to be on the MySQL-specific type
i = Integer(display_width=5)
```

这是一个非常古老的 bug，已经在 0.8 系列中添加了弃用警告，但因为几乎没有人使用“-W”标志来运行 Python，所以几乎从未被看到：

```py
$ python -W always::DeprecationWarning ~/dev/sqlalchemy/test.py
/Users/classic/dev/sqlalchemy/test.py:5: SADeprecationWarning: Passing arguments to
type object constructor <class 'sqlalchemy.types.Date'> is deprecated
  d = Date(storage_format="%(day)02d.%(month)02d.%(year)04d")
/Users/classic/dev/sqlalchemy/test.py:9: SADeprecationWarning: Passing arguments to
type object constructor <class 'sqlalchemy.types.Integer'> is deprecated
  i = Integer(display_width=5)
```

从 0.9 系列开始，`TypeEngine`中的“catch all”构造函数已被移除，这些无意义的参数不再被接受。

使用方言特定参数（如`storage_format`和`display_width`）的正确方法是使用适当的方言特定类型：

```py
from sqlalchemy.dialects.sqlite import DATE
from sqlalchemy.dialects.mysql import INTEGER

d = DATE(storage_format="%(day)02d.%(month)02d.%(year)04d")

i = INTEGER(display_width=5)
```

如果我们还想要方言不可知的类型怎么办？我们使用`TypeEngine.with_variant()`方法：

```py
from sqlalchemy import Date, Integer
from sqlalchemy.dialects.sqlite import DATE
from sqlalchemy.dialects.mysql import INTEGER

d = Date().with_variant(
    DATE(storage_format="%(day)02d.%(month)02d.%(year)04d"), "sqlite"
)

i = Integer().with_variant(INTEGER(display_width=5), "mysql")
```

`TypeEngine.with_variant()`并不是新功能，它是在 SQLAlchemy 0.7.2 中添加的。因此，在 0.8 系列上运行的代码可以根据需要使用这种方法进行更正并在升级到 0.9 之前进行测试。

### `None`不再能够被用作“部分 AND”构造函数

`None`不再能够被用作逐步形成 AND 条件的“后备”。尽管一些 SQLAlchemy 内部使用了这种模式，但这种模式并未被记录：

```py
condition = None

for cond in conditions:
    condition = condition & cond

if condition is not None:
    stmt = stmt.where(condition)
```

上述序列在`conditions`非空时，将在 0.9 上产生`SELECT .. WHERE <condition> AND NULL`。`None`不再被隐式忽略，而是与在其他上下文中解释`None`时保持一致。

对于 0.8 和 0.9 的正确代码应该是：

```py
from sqlalchemy.sql import and_

if conditions:
    stmt = stmt.where(and_(*conditions))
```

另一个变体在 0.9 上适用于所有后端，但在 0.8 上仅适用于支持布尔常量的后端：

```py
from sqlalchemy.sql import true

condition = true()

for cond in conditions:
    condition = cond & condition

stmt = stmt.where(condition)
```

在 0.8 上，这将产生一个 SELECT 语句，在 WHERE 子句中始终有`AND true`，这不被不支持布尔常量的后端（MySQL、MSSQL）接受。在 0.9 上，`true`常量将在`and_()`连接中被删除。

另请参阅

布尔常量、NULL 常量、连接的改进渲染

### `create_engine()`的“password”部分不再将`+`号视为编码空格

由于某种原因，Python 函数`unquote_plus()`被应用于 URL 的`password`字段，这是对[RFC 1738](https://www.ietf.org/rfc/rfc1738.txt)中描述的编码规则的错误应用，因为它将空格转义为加号。现在，URL 的字符串化仅编码“:”、“@”或“/”，而不再应用于`username`和`password`字段（以前仅应用于密码）。在解析时，编码字符被转换，但加号和空格保持不变：

```py
# password: "pass word + other:words"
dbtype://user:pass word + other%3Awords@host/dbname

# password: "apples/oranges"
dbtype://username:apples%2Foranges@hostspec/database

# password: "apples@oranges@@"
dbtype://username:apples%40oranges%40%40@hostspec/database

# password: '', username is "username@"
dbtype://username%40:@hostspec/database
```

[#2873](https://www.sqlalchemy.org/trac/ticket/2873)

### COLLATE 的优先规则已更改

以前，类似以下的表达式：

```py
print((column("x") == "somevalue").collate("en_EN"))
```

会产生如下表达式：

```py
-- 0.8 behavior
(x  =  :x_1)  COLLATE  en_EN
```

上述方法被 MSSQL 误解，通常不是任何数据库建议的语法。现在该表达式将产生大多数数据库文档所示的语法：

```py
-- 0.9 behavior
x  =  :x_1  COLLATE  en_EN
```

如果 `ColumnOperators.collate()` 操作符应用于右列，则可能会出现潜在的不兼容变化，如下所示：

```py
print(column("x") == literal("somevalue").collate("en_EN"))
```

在 0.8 中，这将产生：

```py
x  =  :param_1  COLLATE  en_EN
```

但在 0.9 中，现在将产生更准确的，但可能不是您想要的形式：

```py
x  =  (:param_1  COLLATE  en_EN)
```

`ColumnOperators.collate()` 操作符现在在 `ORDER BY` 表达式中更合适地工作，因为给了 `ASC` 和 `DESC` 操作符一个特定的优先级，这将再次确保不生成括号：

```py
>>> # 0.8
>>> print(column("x").collate("en_EN").desc())
(x  COLLATE  en_EN)  DESC
>>> # 0.9
>>> print(column("x").collate("en_EN").desc())
x  COLLATE  en_EN  DESC 
```

[#2879](https://www.sqlalchemy.org/trac/ticket/2879)

### PostgreSQL CREATE TYPE <x> AS ENUM 现在对值应用引用

`ENUM` 类型现在将在枚举值中应用转义到单引号符：

```py
>>> from sqlalchemy.dialects import postgresql
>>> type = postgresql.ENUM("one", "two", "three's", name="myenum")
>>> from sqlalchemy.dialects.postgresql import base
>>> print(base.CreateEnumType(type).compile(dialect=postgresql.dialect()))
CREATE  TYPE  myenum  AS  ENUM  ('one','two','three''s') 
```

已经转义单引号的现有解决方法将需要修改，否则它们将会双重转义。

[#2878](https://www.sqlalchemy.org/trac/ticket/2878)

## 新特性

### 事件移除 API

使用 `listen()` 或 `listens_for()` 建立的事件现在可以使用新的 `remove()` 函数进行移除。传递给 `remove()` 的 `target`、`identifier` 和 `fn` 参数需要与用于监听的参数完全匹配，事件将从所有已建立的位置移除：

```py
@event.listens_for(MyClass, "before_insert", propagate=True)
def my_before_insert(mapper, connection, target):
  """listen for before_insert"""
    # ...

event.remove(MyClass, "before_insert", my_before_insert)
```

在上述示例中，设置了 `propagate=True` 标志。这意味着 `my_before_insert()` 被建立为 `MyClass` 及其所有子类的监听器。系统跟踪了 `my_before_insert()` 监听器函数作为此调用的结果放置的所有位置，并作为调用 `remove()` 的结果将其移除。

移除系统使用注册表将传递给 `listen()` 的参数与事件监听器的集合相关联，这些事件监听器在许多情况下是原始用户提供的函数的包装版本。该注册表大量使用弱引用，以允许所有包含的内容（例如监听器目标）在超出范围时被垃圾回收。

[#2268](https://www.sqlalchemy.org/trac/ticket/2268)  ### 新查询选项 API；`load_only()` 选项

加载器选项系统，如`joinedload()`、`subqueryload()`、`lazyload()`、`defer()`等，都建立在一个名为`Load`的新系统之上。`Load`提供了一种“方法链式”（又名生成式）的加载器选项方法，因此，不再需要使用点号或多个属性名称将长路径连接在一起，而是为每个路径明确指定加载器样式。

虽然新方法稍微更冗长，但更容易理解，因为对哪些路径应用了哪些选项没有歧义；它简化了选项的方法签名，并为基于列的选项提供了更大的灵活性。旧系统将继续无限期保持功能，并且所有样式都可以混合使用。

**旧方法**

要在多元素路径中的每个链接上设置某种加载样式，必须使用`_all()`选项：

```py
query(User).options(joinedload_all("orders.items.keywords"))
```

**新方法**

现在加载器选项是可链接的，因此相同的`joinedload(x)`方法等同地应用于每个链接，无需在`joinedload()`和`joinedload_all()`之间保持清晰：

```py
query(User).options(joinedload("orders").joinedload("items").joinedload("keywords"))
```

**旧方法**

在基于子类的路径上设置一个选项需要将路径中的所有链接拼写为类绑定属性，因为`PropComparator.of_type()`方法需要被调用：

```py
session.query(Company).options(
    subqueryload_all(Company.employees.of_type(Engineer), Engineer.machines)
)
```

**新方法**

只有那些实际需要`PropComparator.of_type()`的路径中的元素需要被设置为类绑定属性，之后可以恢复基于字符串的名称：

```py
session.query(Company).options(
    subqueryload(Company.employees.of_type(Engineer)).subqueryload("machines")
)
```

**旧方法**

在长路径中的最后一个链接上设置加载器选项使用的语法看起来很像应该为路径中的所有链接设置选项，导致混淆：

```py
query(User).options(subqueryload("orders.items.keywords"))
```

**新方法**

现在可以使用`defaultload()`为路径中的条目拼写出路径，其中现有的加载器样式不应更改。更冗长但意图更清晰：

```py
query(User).options(defaultload("orders").defaultload("items").subqueryload("keywords"))
```

仍然可以利用点号样式，特别是在跳过几个路径元素的情况下：

```py
query(User).options(defaultload("orders.items").subqueryload("keywords"))
```

**旧方法**

在路径上使用`defer()`选项需要为每个列明确指定完整路径：

```py
query(User).options(defer("orders.description"), defer("orders.isopen"))
```

**新方式**

到达目标路径的单个`Load`对象可以反复调用`Load.defer()`：

```py
query(User).options(defaultload("orders").defer("description").defer("isopen"))
```

#### Load 类

`Load`类可以直接用于提供“绑定”目标，特别是当存在多个父实体时：

```py
from sqlalchemy.orm import Load

query(User, Address).options(Load(Address).joinedload("entries"))
```

#### 仅加载

一个新选项`load_only()`实现了“除了延迟加载一切之外”的加载方式，仅加载给定的列，并延迟其余部分：

```py
from sqlalchemy.orm import load_only

query(User).options(load_only("name", "fullname"))

# specify explicit parent entity
query(User, Address).options(Load(User).load_only("name", "fullname"))

# specify path
query(User).options(joinedload(User.addresses).load_only("email_address"))
```

#### 类特定的通配符

使用`Load`，可以使用通配符为给定实体上的所有关系（或者列）设置加载方式，而不影响其他实体：

```py
# lazyload all User relationships
query(User).options(Load(User).lazyload("*"))

# undefer all User columns
query(User).options(Load(User).undefer("*"))

# lazyload all Address relationships
query(User).options(defaultload(User.addresses).lazyload("*"))

# undefer all Address columns
query(User).options(defaultload(User.addresses).undefer("*"))
```

[#1418](https://www.sqlalchemy.org/trac/ticket/1418)  ### 新的`text()`功能

`text()`构造获得了新的方法：

+   `TextClause.bindparams()`允许灵活设置绑定参数类型和值：

    ```py
    # setup values
    stmt = text(
        "SELECT id, name FROM user WHERE name=:name AND timestamp=:timestamp"
    ).bindparams(name="ed", timestamp=datetime(2012, 11, 10, 15, 12, 35))

    # setup types and/or values
    stmt = (
        text("SELECT id, name FROM user WHERE name=:name AND timestamp=:timestamp")
        .bindparams(bindparam("name", value="ed"), bindparam("timestamp", type_=DateTime()))
        .bindparam(timestamp=datetime(2012, 11, 10, 15, 12, 35))
    )
    ```

+   `TextClause.columns()`取代了`text()`的`typemap`选项，返回一个新的构造`TextAsFrom`：

    ```py
    # turn a text() into an alias(), with a .c. collection:
    stmt = text("SELECT id, name FROM user").columns(id=Integer, name=String)
    stmt = stmt.alias()

    stmt = select([addresses]).select_from(
        addresses.join(stmt), addresses.c.user_id == stmt.c.id
    )

    # or into a cte():
    stmt = text("SELECT id, name FROM user").columns(id=Integer, name=String)
    stmt = stmt.cte("x")

    stmt = select([addresses]).select_from(
        addresses.join(stmt), addresses.c.user_id == stmt.c.id
    )
    ```

[#2877](https://www.sqlalchemy.org/trac/ticket/2877)  ### 从 SELECT 中插入

经过几乎多年的毫无意义的拖延，这个相对较小的语法特性已经被添加，并且也被回溯到 0.8.3，所以在技术上在 0.9 中并不是“新”的。一个`select()`构造或其他兼容的构造可以传递给新方法`Insert.from_select()`，在那里它将被用于渲染一个`INSERT .. SELECT`构造：

```py
>>> from sqlalchemy.sql import table, column
>>> t1 = table("t1", column("a"), column("b"))
>>> t2 = table("t2", column("x"), column("y"))
>>> print(t1.insert().from_select(["a", "b"], t2.select().where(t2.c.y == 5)))
INSERT  INTO  t1  (a,  b)  SELECT  t2.x,  t2.y
FROM  t2
WHERE  t2.y  =  :y_1 
```

这个结构足够智能，也可以适应 ORM 对象，比如类和`Query`对象：

```py
s = Session()
q = s.query(User.id, User.name).filter_by(name="ed")
ins = insert(Address).from_select((Address.id, Address.email_address), q)
```

渲染：

```py
INSERT  INTO  addresses  (id,  email_address)
SELECT  users.id  AS  users_id,  users.name  AS  users_name
FROM  users  WHERE  users.name  =  :name_1
```

[#722](https://www.sqlalchemy.org/trac/ticket/722)  ### `select()`、`Query()`上的新 FOR UPDATE 支持

尝试简化在 Core 和 ORM 中对 `SELECT` 语句中的 `FOR UPDATE` 子句的规范，并支持 PostgreSQL 和 Oracle 支持的 `FOR UPDATE OF` SQL。

使用核心`GenerativeSelect.with_for_update()`，可以单独指定`FOR SHARE`和`NOWAIT`等选项，而不是链接到任意字符串代码：

```py
stmt = select([table]).with_for_update(read=True, nowait=True, of=table)
```

在 Posgtresql 上，上述语句可能会呈现如下：

```py
SELECT  table.a,  table.b  FROM  table  FOR  SHARE  OF  table  NOWAIT
```

`Query` 对象获得了类似的方法 `Query.with_for_update()`，其行为方式相同。该方法取代了现有的 `Query.with_lockmode()` 方法，该方法使用不同的系统翻译 `FOR UPDATE` 子句。目前，“lockmode”字符串参数仍然被 `Session.refresh()` 方法接受。  ### 本机浮点字符串转换精度可配置的浮点类型

每当 DBAPI 返回要转换为 Python `Decimal()` 的 Python 浮点类型时，SQLAlchemy 所做的转换必然涉及将浮点值转换为字符串的中间步骤。此字符串转换的比例以前是硬编码为 10，现在可配置。该设置可用于 `Numeric` 以及 `Float` 类型，以及所有 SQL 和方言特定的后代类型，使用参数 `decimal_return_scale`。如果类型支持 `.scale` 参数，例如 `Numeric` 和一些浮点类型如 `DOUBLE`，则如果未另行指定，`.scale` 的值将用作 `.decimal_return_scale` 的默认值。如果 `.scale` 和 `.decimal_return_scale` 都不存在，则默认值为 10。例如：

```py
from sqlalchemy.dialects.mysql import DOUBLE
import decimal

data = Table(
    "data",
    metadata,
    Column("double_value", mysql.DOUBLE(decimal_return_scale=12, asdecimal=True)),
)

conn.execute(
    data.insert(),
    double_value=45.768392065789,
)
result = conn.scalar(select([data.c.double_value]))

# previously, this would typically be Decimal("45.7683920658"),
# e.g. trimmed to 10 decimal places

# now we get 12, as requested, as MySQL can support this
# much precision for DOUBLE
assert result == decimal.Decimal("45.768392065789")
```

[#2867](https://www.sqlalchemy.org/trac/ticket/2867)  ### ORM 查询的列捆绑

`Bundle` 允许查询一组列，然后将它们分组为查询返回的元组下的一个名称。`Bundle` 的最初目的是 1\. 允许将“复合”ORM 列作为列式结果集中的单个值返回，而不是将它们扩展为单独的列，以及 2\. 允许在 ORM 中创建自定义结果集构造，使用临时列和返回类型，而不涉及映射类的更重量级机制。

另请参阅

当按属性基础查询时，复合属性现在以其对象形式返回

使用 Bundles 分组选定属性

[#2824](https://www.sqlalchemy.org/trac/ticket/2824)

### 服务器端版本计数

ORM 的版本控制功能（现在也在配置版本计数器中记录）现在可以利用服务器端版本计数方案，例如由触发器或数据库系统列生成的方案，以及版本 _id_counter 函数本身之外的条件编程方案。通过向 `version_id_generator` 参数提供值 `False`，ORM 将使用已设置的版本标识符，或者在发出 INSERT 或 UPDATE 时同时从每行获取版本标识符。当使用服务器生成的版本标识符时，强烈建议仅在具有强大 RETURNING 支持的后端上使用此功能（PostgreSQL、SQL Server；Oracle 也支持 RETURNING，但 cx_oracle 驱动程序仅支持有限），否则额外的 SELECT 语句将增加显著的性能开销。服务器端版本计数器提供的示例说明了如何使用 PostgreSQL 的 `xmin` 系统列将其与 ORM 的版本控制功能集成。

另请参阅

服务器端版本计数器

[#2793](https://www.sqlalchemy.org/trac/ticket/2793)

### `include_backrefs=False` 选项用于 `@validates`

`validates()` 函数现在接受一个选项 `include_backrefs=True`，这将跳过为从反向引用发起的事件触发验证器的情况：

```py
from sqlalchemy import Column, Integer, ForeignKey
from sqlalchemy.orm import relationship, validates
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()

class A(Base):
    __tablename__ = "a"

    id = Column(Integer, primary_key=True)
    bs = relationship("B", backref="a")

    @validates("bs")
    def validate_bs(self, key, item):
        print("A.bs validator")
        return item

class B(Base):
    __tablename__ = "b"

    id = Column(Integer, primary_key=True)
    a_id = Column(Integer, ForeignKey("a.id"))

    @validates("a", include_backrefs=False)
    def validate_a(self, key, item):
        print("B.a validator")
        return item

a1 = A()
a1.bs.append(B())  # prints only "A.bs validator"
```

[#1535](https://www.sqlalchemy.org/trac/ticket/1535)

### PostgreSQL JSON 类型

PostgreSQL 方言现在具有一个 `JSON` 类型，以补充 `HSTORE` 类型。

另请参阅

`JSON`

[#2581](https://www.sqlalchemy.org/trac/ticket/2581)

### 自动映射扩展

新的扩展在**0.9.1**中添加，称为`sqlalchemy.ext.automap`。这是一个**实验性**扩展，它扩展了声明式的功能以及`DeferredReflection`类的功能。基本上，该扩展提供了一个基类`AutomapBase`，根据给定的表元数据自动生成映射类和它们之间的关系。

正常使用的`MetaData`可能是通过反射生成的，但不要求使用反射。最基本的用法说明了`sqlalchemy.ext.automap`如何根据反射模式提供映射类，包括关系：

```py
from sqlalchemy.ext.automap import automap_base
from sqlalchemy.orm import Session
from sqlalchemy import create_engine

Base = automap_base()

# engine, suppose it has two tables 'user' and 'address' set up
engine = create_engine("sqlite:///mydatabase.db")

# reflect the tables
Base.prepare(engine, reflect=True)

# mapped classes are now created with names matching that of the table
# name.
User = Base.classes.user
Address = Base.classes.address

session = Session(engine)

# rudimentary relationships are produced
session.add(Address(email_address="foo@bar.com", user=User(name="foo")))
session.commit()

# collection-based relationships are by default named "<classname>_collection"
print(u1.address_collection)
```

此外，`AutomapBase`类是一个声明性基类，并支持所有声明性的功能。可以使用“自动映射”功能与现有的明确定义的模式一起使用，仅生成关系和缺失类。命名方案和关系生成例程可以使用可调用函数来插入。

期望`AutomapBase`系统提供了一个快速且现代化的解决方案，解决了非常著名的[SQLSoup](https://sqlsoup.readthedocs.io/en/latest/)也尝试解决的问题，即从现有数据库动态生成快速和简陋的对象模型的问题。通过严格在映射器配置级别解决该问题，并与现有的声明式类技术完全集成，`AutomapBase`旨在为快速自动生成临时映射的问题提供一个良好集成的方法。

另请参阅

自动映射  ### 事件移除 API

使用`listen()`或`listens_for()`建立的事件现在可以使用新的`remove()`函数进行移除。发送给`remove()`的`target`、`identifier`和`fn`参数需要与用于监听的参数完全匹配，并且事件将从建立的所有位置中移除：

```py
@event.listens_for(MyClass, "before_insert", propagate=True)
def my_before_insert(mapper, connection, target):
  """listen for before_insert"""
    # ...

event.remove(MyClass, "before_insert", my_before_insert)
```

在上面的示例中，设置了`propagate=True`标志。这意味着`my_before_insert()`被建立为`MyClass`以及`MyClass`的所有子类的监听器。系统跟踪了`my_before_insert()`监听函数作为此调用的结果放置的所有位置，并在调用`remove()`后将其移除。

移除系统使用注册表将传递给`listen()`的参数与事件监听器集合关联，这些事件监听器在许多情况下是原始用户提供的函数的包装版本。该注册表大量使用弱引用，以允许所有包含的内容（如监听器目标）在超出范围时被垃圾回收。

[#2268](https://www.sqlalchemy.org/trac/ticket/2268)

### 新查询选项 API；`load_only()` 选项

加载器选项系统，如`joinedload()`、`subqueryload()`、`lazyload()`、`defer()`等，都建立在一个名为`Load`的新系统之上。`Load`提供了一种“方法链式”（又名生成式）的加载器选项方法，因此不再需要使用点号或多个属性名称连接长路径，而是为每个路径指定明确的加载器样式。

新方法虽然稍微冗长，但更容易理解，因为对应哪些路径应用了哪些选项没有歧义；它简化了选项的方法签名，并为基于列的选项提供了更大的灵活性。旧系统将永远保持功能，并且所有样式都可以混合使用。

**旧方法**

要在多元素路径中的每个链接上设置特定的加载样式，必须使用`_all()`选项：

```py
query(User).options(joinedload_all("orders.items.keywords"))
```

**新方法**

加载器选项现在可以链式调用，因此相同的`joinedload(x)`方法同样适用于每个链接，无需在`joinedload()`和`joinedload_all()`之间保持清晰：

```py
query(User).options(joinedload("orders").joinedload("items").joinedload("keywords"))
```

**旧方法**

在基于子类的路径上设置选项需要将路径中的所有链接拼写为类绑定属性，因为需要调用`PropComparator.of_type()`方法：

```py
session.query(Company).options(
    subqueryload_all(Company.employees.of_type(Engineer), Engineer.machines)
)
```

**新方法**

只有实际需要`PropComparator.of_type()`的路径元素需要��置为类绑定属性，之后可以恢复基于字符串的名称：

```py
session.query(Company).options(
    subqueryload(Company.employees.of_type(Engineer)).subqueryload("machines")
)
```

**旧方法**

在长路径中设置加载器选项的最后一个链接使用的语法看起来很像应该为路径中的所有链接设置选项，导致混淆：

```py
query(User).options(subqueryload("orders.items.keywords"))
```

**新方法**

现在可以使用`defaultload()`来拼写路径，其中现有的加载器样式应保持不变。更冗长但意图更清晰：

```py
query(User).options(defaultload("orders").defaultload("items").subqueryload("keywords"))
```

点线样式仍然可以被充分利用，特别是在跳过多个路径元素的情况下：

```py
query(User).options(defaultload("orders.items").subqueryload("keywords"))
```

**旧方法**

需要为路径上的每个列拼写出完整路径的`defer()`选项：

```py
query(User).options(defer("orders.description"), defer("orders.isopen"))
```

**新方法**

到达目标路径的单个`Load`对象可以反复调用`Load.defer()`：

```py
query(User).options(defaultload("orders").defer("description").defer("isopen"))
```

#### 加载类

`Load`类可以直接用于提供“绑定”目标，特别是当存在多个父实体时：

```py
from sqlalchemy.orm import Load

query(User, Address).options(Load(Address).joinedload("entries"))
```

#### 仅加载

新选项`load_only()`实现了“除了加载之外的一切都延迟”的加载方式，仅加载给定列并推迟其余部分：

```py
from sqlalchemy.orm import load_only

query(User).options(load_only("name", "fullname"))

# specify explicit parent entity
query(User, Address).options(Load(User).load_only("name", "fullname"))

# specify path
query(User).options(joinedload(User.addresses).load_only("email_address"))
```

#### 类特定通配符

使用`Load`，可以使用通配符为给定实体上的所有关系（或可能列）设置加载，而不影响其他实体：

```py
# lazyload all User relationships
query(User).options(Load(User).lazyload("*"))

# undefer all User columns
query(User).options(Load(User).undefer("*"))

# lazyload all Address relationships
query(User).options(defaultload(User.addresses).lazyload("*"))

# undefer all Address columns
query(User).options(defaultload(User.addresses).undefer("*"))
```

[#1418](https://www.sqlalchemy.org/trac/ticket/1418)

#### 加载类

`Load`类可以直接用于提供“绑定”目标，特别是当存在多个父实体时：

```py
from sqlalchemy.orm import Load

query(User, Address).options(Load(Address).joinedload("entries"))
```

#### 仅加载

新选项`load_only()`实现了“除了加载之外的一切都延迟加载”的加载方式，仅加载给定的列并推迟其余的列：

```py
from sqlalchemy.orm import load_only

query(User).options(load_only("name", "fullname"))

# specify explicit parent entity
query(User, Address).options(Load(User).load_only("name", "fullname"))

# specify path
query(User).options(joinedload(User.addresses).load_only("email_address"))
```

#### 类特定的通配符

使用`Load`，可以使用通配符为给定实体上的所有关系（或可能是列）设置加载，而不影响其他实体：

```py
# lazyload all User relationships
query(User).options(Load(User).lazyload("*"))

# undefer all User columns
query(User).options(Load(User).undefer("*"))

# lazyload all Address relationships
query(User).options(defaultload(User.addresses).lazyload("*"))

# undefer all Address columns
query(User).options(defaultload(User.addresses).undefer("*"))
```

[#1418](https://www.sqlalchemy.org/trac/ticket/1418)

### 新的`text()`功能

`text()`构造获得了新方法：

+   `TextClause.bindparams()`允许灵活设置绑定参数类型和值：

    ```py
    # setup values
    stmt = text(
        "SELECT id, name FROM user WHERE name=:name AND timestamp=:timestamp"
    ).bindparams(name="ed", timestamp=datetime(2012, 11, 10, 15, 12, 35))

    # setup types and/or values
    stmt = (
        text("SELECT id, name FROM user WHERE name=:name AND timestamp=:timestamp")
        .bindparams(bindparam("name", value="ed"), bindparam("timestamp", type_=DateTime()))
        .bindparam(timestamp=datetime(2012, 11, 10, 15, 12, 35))
    )
    ```

+   `TextClause.columns()`取代了`text()`的`typemap`选项，返回一个新的构造`TextAsFrom`：

    ```py
    # turn a text() into an alias(), with a .c. collection:
    stmt = text("SELECT id, name FROM user").columns(id=Integer, name=String)
    stmt = stmt.alias()

    stmt = select([addresses]).select_from(
        addresses.join(stmt), addresses.c.user_id == stmt.c.id
    )

    # or into a cte():
    stmt = text("SELECT id, name FROM user").columns(id=Integer, name=String)
    stmt = stmt.cte("x")

    stmt = select([addresses]).select_from(
        addresses.join(stmt), addresses.c.user_id == stmt.c.id
    )
    ```

[#2877](https://www.sqlalchemy.org/trac/ticket/2877)

### 从 SELECT 插入

经过几乎多年的毫无意义的拖延，这个相对较小的语法特性已经被添加，并且也被回溯到了 0.8.3 版本，所以在技术上在 0.9 版本中并不是“新”功能。可以将`select()`构造或其他兼容的构造传递给新方法`Insert.from_select()`，其中它将被用于渲染`INSERT .. SELECT`构造：

```py
>>> from sqlalchemy.sql import table, column
>>> t1 = table("t1", column("a"), column("b"))
>>> t2 = table("t2", column("x"), column("y"))
>>> print(t1.insert().from_select(["a", "b"], t2.select().where(t2.c.y == 5)))
INSERT  INTO  t1  (a,  b)  SELECT  t2.x,  t2.y
FROM  t2
WHERE  t2.y  =  :y_1 
```

该构造足够智能，也可以适应 ORM 对象，如类和`Query`对象：

```py
s = Session()
q = s.query(User.id, User.name).filter_by(name="ed")
ins = insert(Address).from_select((Address.id, Address.email_address), q)
```

渲染：

```py
INSERT  INTO  addresses  (id,  email_address)
SELECT  users.id  AS  users_id,  users.name  AS  users_name
FROM  users  WHERE  users.name  =  :name_1
```

[#722](https://www.sqlalchemy.org/trac/ticket/722)

### `select()`，`Query()`上的新的 FOR UPDATE 支持

尝试简化在 Core 和 ORM 中制作`SELECT`语句时`FOR UPDATE`子句的规范，并支持 PostgreSQL 和 Oracle 支持的`FOR UPDATE OF` SQL。

使用核心`GenerativeSelect.with_for_update()`，可以单独指定选项，如`FOR SHARE`和`NOWAIT`，而不是链接到任意字符串代码：

```py
stmt = select([table]).with_for_update(read=True, nowait=True, of=table)
```

在 Posgtresql 上述语句可能会呈现如下：

```py
SELECT  table.a,  table.b  FROM  table  FOR  SHARE  OF  table  NOWAIT
```

`Query` 对象获得了一个类似的方法 `Query.with_for_update()`，其行为方式相同。该方法取代了现有的 `Query.with_lockmode()` 方法，该方法使用不同的系统翻译 `FOR UPDATE` 子句。目前，“lockmode” 字符串参数仍然被 `Session.refresh()` 方法接受。

### 本机浮点字符串转换精度可配置

每当 DBAPI 返回一个要转换为 Python `Decimal()` 的 Python 浮点类型时，SQLAlchemy 所做的转换必然涉及一个中间步骤，将浮点值转换为字符串。用于此字符串转换的标度以前是硬编码为 10，现在是可配置的。该设置在 `Numeric` 以及 `Float` 类型上都可用，以及所有 SQL 和特定方言的后代类型，使用参数 `decimal_return_scale`。如果类型支持 `.scale` 参数，如 `Numeric` 和一些浮点类型如 `DOUBLE`，则如果未另行指定，则 `.scale` 的值将用作 `.decimal_return_scale` 的默认值。如果 `.scale` 和 `.decimal_return_scale` 都不存在，则默认值为 10。例如：

```py
from sqlalchemy.dialects.mysql import DOUBLE
import decimal

data = Table(
    "data",
    metadata,
    Column("double_value", mysql.DOUBLE(decimal_return_scale=12, asdecimal=True)),
)

conn.execute(
    data.insert(),
    double_value=45.768392065789,
)
result = conn.scalar(select([data.c.double_value]))

# previously, this would typically be Decimal("45.7683920658"),
# e.g. trimmed to 10 decimal places

# now we get 12, as requested, as MySQL can support this
# much precision for DOUBLE
assert result == decimal.Decimal("45.768392065789")
```

[#2867](https://www.sqlalchemy.org/trac/ticket/2867)

### ORM 查询的列捆绑

`Bundle` 允许查询一组列，然后将它们分组为查询返回的元组下的一个名称。`Bundle` 的最初目的是 1\. 允许将“复合”ORM 列作为列结果集中的单个值返回，而不是将它们展开为单独的列，以及 2\. 允许在 ORM 中创建自定义结果集构造，使用临时列和返回类型，而不涉及映射类的更重量级机制。

另请参见

当按属性基础查询时，复合属性现在以其对象形式返回

使用捆绑组合选定属性

[#2824](https://www.sqlalchemy.org/trac/ticket/2824)

### 服务器端版本计数

ORM 的版本控制功能（现在还在 配置版本计数器 中有文档记录）现在可以利用服务器端的版本计数方案，例如由触发器或数据库系统列生成的方案，以及版本 _id_counter 函数本身之外的条件编程方案。通过向 `version_id_generator` 参数提供值 `False`，ORM 将使用已设置的版本标识符，或者在发出 INSERT 或 UPDATE 时同时从每一行获取版本标识符。当使用服务器生成的版本标识符时，强烈建议仅在具有强大 RETURNING 支持的后端上使用此功能（PostgreSQL、SQL Server；Oracle 也支持 RETURNING，但 cx_oracle 驱动程序仅具有有限的支持），否则额外的 SELECT 语句将增加显著的性能开销。在 服务器端版本计数器 提供的示例中说明了使用 PostgreSQL `xmin` 系统列将其与 ORM 的版本控制功能集成的用法。

另请参阅

服务器端版本计数器

[#2793](https://www.sqlalchemy.org/trac/ticket/2793)

### `include_backrefs=False` 选项用于 `@validates`

`validates()` 函数现在接受一个选项 `include_backrefs=True`，将为从反向引用发起事件的情况跳过触发器：

```py
from sqlalchemy import Column, Integer, ForeignKey
from sqlalchemy.orm import relationship, validates
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()

class A(Base):
    __tablename__ = "a"

    id = Column(Integer, primary_key=True)
    bs = relationship("B", backref="a")

    @validates("bs")
    def validate_bs(self, key, item):
        print("A.bs validator")
        return item

class B(Base):
    __tablename__ = "b"

    id = Column(Integer, primary_key=True)
    a_id = Column(Integer, ForeignKey("a.id"))

    @validates("a", include_backrefs=False)
    def validate_a(self, key, item):
        print("B.a validator")
        return item

a1 = A()
a1.bs.append(B())  # prints only "A.bs validator"
```

[#1535](https://www.sqlalchemy.org/trac/ticket/1535)

### PostgreSQL JSON 类型

PostgreSQL 方言现在具有 `JSON` 类型，以补充 `HSTORE` 类型。

另请参阅

`JSON`

[#2581](https://www.sqlalchemy.org/trac/ticket/2581)

### Automap 扩展

**0.9.1** 版本新增了一个名为`sqlalchemy.ext.automap`的扩展。这是一个**实验性**扩展，它扩展了声明式以及`DeferredReflection`类的功能。本质上，该扩展提供了一个基类`AutomapBase`，根据给定的表元数据自动生成映射类和它们之间的关系。

使用的`MetaData`通常可能是通过反射生成的，但不要求使用反射。最基本的用法说明了`sqlalchemy.ext.automap`如何能够根据反射模式提供映射类，包括关系：

```py
from sqlalchemy.ext.automap import automap_base
from sqlalchemy.orm import Session
from sqlalchemy import create_engine

Base = automap_base()

# engine, suppose it has two tables 'user' and 'address' set up
engine = create_engine("sqlite:///mydatabase.db")

# reflect the tables
Base.prepare(engine, reflect=True)

# mapped classes are now created with names matching that of the table
# name.
User = Base.classes.user
Address = Base.classes.address

session = Session(engine)

# rudimentary relationships are produced
session.add(Address(email_address="foo@bar.com", user=User(name="foo")))
session.commit()

# collection-based relationships are by default named "<classname>_collection"
print(u1.address_collection)
```

此外，`AutomapBase`类是一个声明基类，并支持声明的所有功能。可以将“自动映射”功能用于现有的、明确声明的模式，以仅生成关系和缺失类。可以使用可调用函数添加命名方案和关系生成例程。

希望`AutomapBase`系统提供了一个快速且现代化的解决方案，解决了非常著名的[SQLSoup](https://sqlsoup.readthedocs.io/en/latest/)也试图解决的问题，即从现有数据库快速生成一个简单的对象模型。通过严格在映射器配置级别解决该问题，并与现有的声明类技术完全集成，`AutomapBase`旨在提供一个完全集成的方法来解决迅速自动生成临时映射的问题。

另请参阅

自动映射

## 行为改进

改进应该不会产生兼容性问题，除非在极为罕见和不寻常的假设情况下，但如果有意外问题，了解这些改进是很好的。

### 许多 JOIN 和 LEFT OUTER JOIN 表达式将不再包含在 (SELECT * FROM ..) AS ANON_1 中

多年来，SQLAlchemy ORM 一直无法在现有 JOIN 的右侧嵌套 JOIN（通常是 LEFT OUTER JOIN），因为内部 JOIN 通常可以被展平：

```py
SELECT  a.*,  b.*,  c.*  FROM  a  LEFT  OUTER  JOIN  (b  JOIN  c  ON  b.id  =  c.id)  ON  a.id
```

这是因为 SQLite 直到版本**3.7.16**都无法解析上述格式的语句：

```py
SQLite version 3.7.15.2 2013-01-09 11:53:05
Enter ".help" for instructions
Enter SQL statements terminated with a ";"
sqlite> create table a(id integer);
sqlite> create table b(id integer);
sqlite> create table c(id integer);
sqlite> select a.id, b.id, c.id from a left outer join (b join c on b.id=c.id) on b.id=a.id;
Error: no such column: b.id
```

右外连接当然是解决右侧括号化的另一种方法；这将显着复杂化并且视觉上不美观，但幸运的是 SQLite 也不支持 RIGHT OUTER JOIN :)：

```py
sqlite>  select  a.id,  b.id,  c.id  from  b  join  c  on  b.id=c.id
  ...>  right  outer  join  a  on  b.id=a.id;
Error:  RIGHT  and  FULL  OUTER  JOINs  are  not  currently  supported
```

早在 2005 年，其他数据库是否有问题尚不清楚，但今天看来，除 SQLite 外的每个测试数据库都支持它（Oracle 8 是一个非常老的数据库，根本不支持 JOIN 关键字，但 SQLAlchemy 一直为 Oracle 的语法制定了一个简单的重写方案）。更糟糕的是，SQLAlchemy 常规的解决方法在诸如 PostgreSQL 和 MySQL 等平台上应用 SELECT 通常会降低性能：

```py
SELECT  a.*,  anon_1.*  FROM  a  LEFT  OUTER  JOIN  (
  SELECT  b.id  AS  b_id,  c.id  AS  c_id
  FROM  b  JOIN  c  ON  b.id  =  c.id
  )  AS  anon_1  ON  a.id=anon_1.b_id
```

当使用联接表继承结构时，像上面的 JOIN 形式是司空见惯的；每当使用`Query.join()`从某个父类连接到联接表子类，或者类似地使用`joinedload()`，SQLAlchemy 的 ORM 总是确保不会渲染嵌套 JOIN，以免查询无法在 SQLite 上运行。即使 Core 一直支持更紧凑形式的 JOIN，ORM 也必须避免它。

当在跨多对多关系上生成连接时，如果 ON 子句中存在特殊条件，将会出现另一个问题。考虑下面这样的 eager load 连接：

```py
session.query(Order).outerjoin(Order.items)
```

假设从`Order`到`Item`的多对多关系实际上指的是一个子类`Subitem`，上述情况的 SQL 如下所示：

```py
SELECT  order.id,  order.name
FROM  order  LEFT  OUTER  JOIN  order_item  ON  order.id  =  order_item.order_id
LEFT  OUTER  JOIN  item  ON  order_item.item_id  =  item.id  AND  item.type  =  'subitem'
```

上面的查询有什么问题？基本上，它会加载许多`order` / `order_item`行，其中`item.type == 'subitem'`的条件不成立。

从 SQLAlchemy 0.9 开始，采取了全新的方法。ORM 不再担心将 JOIN 嵌套在连接 JOIN 的右侧，现在会尽可能地渲染这些 JOIN，同时仍然返回正确的结果。当 SQL 语句被传递进行编译时，**方言编译器**将会**重写 JOIN**以适应目标后端，如果该后端已知不支持右嵌套 JOIN（目前只有 SQLite - 如果其他后端也有此问题，请告诉我们！）。

因此，现在通常会生成一个更简单的表达式： 

```py
SELECT  parent.id  AS  parent_id
FROM  parent  JOIN  (
  base_table  JOIN  subclass_table
  ON  base_table.id  =  subclass_table.id)  ON  parent.id  =  base_table.parent_id
```

使用像`query(Parent).options(joinedload(Parent.subclasses))`这样的 eager loads 会给每个表起别名，而不是包装在`ANON_1`中：

```py
SELECT  parent.*,  base_table_1.*,  subclass_table_1.*  FROM  parent
  LEFT  OUTER  JOIN  (
  base_table  AS  base_table_1  JOIN  subclass_table  AS  subclass_table_1
  ON  base_table_1.id  =  subclass_table_1.id)
  ON  parent.id  =  base_table_1.parent_id
```

多对多连接和 eagerloads 将会将“secondary”和“right”表右嵌套：

```py
SELECT  order.id,  order.name
FROM  order  LEFT  OUTER  JOIN
(order_item  JOIN  item  ON  order_item.item_id  =  item.id  AND  item.type  =  'subitem')
ON  order_item.order_id  =  order.id
```

所有这些连接，当与明确指定`use_labels=True`的`Select`语句一起渲染时，这对于 ORM 发出的所有查询都是真实的，都是“连接重写”的候选对象，这是将所有这些右嵌套连接重写为嵌套的 SELECT 语句的过程，同时保持`Select`使用的相同标签。因此，SQLite，即在 2013 年仍不支持这种非常常见的 SQL 语法的数据库，自身承担了额外的复杂性，上述查询被重写为：

```py
-- sqlite only!
SELECT  parent.id  AS  parent_id
  FROM  parent  JOIN  (
  SELECT  base_table.id  AS  base_table_id,
  base_table.parent_id  AS  base_table_parent_id,
  subclass_table.id  AS  subclass_table_id
  FROM  base_table  JOIN  subclass_table  ON  base_table.id  =  subclass_table.id
  )  AS  anon_1  ON  parent.id  =  anon_1.base_table_parent_id

-- sqlite only!
SELECT  parent.id  AS  parent_id,  anon_1.subclass_table_1_id  AS  subclass_table_1_id,
  anon_1.base_table_1_id  AS  base_table_1_id,
  anon_1.base_table_1_parent_id  AS  base_table_1_parent_id
FROM  parent  LEFT  OUTER  JOIN  (
  SELECT  base_table_1.id  AS  base_table_1_id,
  base_table_1.parent_id  AS  base_table_1_parent_id,
  subclass_table_1.id  AS  subclass_table_1_id
  FROM  base_table  AS  base_table_1
  JOIN  subclass_table  AS  subclass_table_1  ON  base_table_1.id  =  subclass_table_1.id
)  AS  anon_1  ON  parent.id  =  anon_1.base_table_1_parent_id

-- sqlite only!
SELECT  "order".id  AS  order_id
FROM  "order"  LEFT  OUTER  JOIN  (
  SELECT  order_item_1.order_id  AS  order_item_1_order_id,
  order_item_1.item_id  AS  order_item_1_item_id,
  item.id  AS  item_id,  item.type  AS  item_type
FROM  order_item  AS  order_item_1
  JOIN  item  ON  item.id  =  order_item_1.item_id  AND  item.type  IN  (?)
)  AS  anon_1  ON  "order".id  =  anon_1.order_item_1_order_id
```

注意

从 SQLAlchemy 1.1 开始，此功能中存在的针对 SQLite 的解决方法将在检测到 SQLite 版本**3.7.16**或更高版本时自动禁用，因为 SQLite 已修复了对右嵌套连接的支持。

`Join.alias()`，`aliased()`和`with_polymorphic()`函数现在支持一个新参数，`flat=True`，用于构建别名的连接表实体而不嵌入到 SELECT 中。这个标志默认情况下是关闭的，以帮助向后兼容 - 但现在一个“多态”可选择可以作为目标连接而不生成任何子查询：

```py
employee_alias = with_polymorphic(Person, [Engineer, Manager], flat=True)

session.query(Company).join(Company.employees.of_type(employee_alias)).filter(
    or_(Engineer.primary_language == "python", Manager.manager_name == "dilbert")
)
```

生成（除了 SQLite 之外的所有地方）：

```py
SELECT  companies.company_id  AS  companies_company_id,  companies.name  AS  companies_name
FROM  companies  JOIN  (
  people  AS  people_1
  LEFT  OUTER  JOIN  engineers  AS  engineers_1  ON  people_1.person_id  =  engineers_1.person_id
  LEFT  OUTER  JOIN  managers  AS  managers_1  ON  people_1.person_id  =  managers_1.person_id
)  ON  companies.company_id  =  people_1.company_id
WHERE  engineers.primary_language  =  %(primary_language_1)s
  OR  managers.manager_name  =  %(manager_name_1)s
```

[#2369](https://www.sqlalchemy.org/trac/ticket/2369) [#2587](https://www.sqlalchemy.org/trac/ticket/2587)  ### 右嵌套内连接在连接的急切加载中可用

从版本 0.9.4 开始，上述提到的右嵌套连接可以在连接的急切加载中启用，在这种情况下，一个“外部”连接链接到右侧的“内部”连接。

通常，像下面这样的连接急切加载链：

```py
query(User).options(
    joinedload("orders", innerjoin=False).joinedload("items", innerjoin=True)
)
```

不会产生内连接；因为从用户->订单的 LEFT OUTER JOIN，连接的急切加载不能使用从订单->项目的 INNER join，而不更改返回的用户行，并且会忽略“链接”`innerjoin=True`指令。0.9.0 应该交付的是，而不是：

```py
FROM  users  LEFT  OUTER  JOIN  orders  ON  <onclause>  LEFT  OUTER  JOIN  items  ON  <onclause>
```

新的“右嵌套连接是可以的”逻辑会启动，我们会得到：

```py
FROM  users  LEFT  OUTER  JOIN  (orders  JOIN  items  ON  <onclause>)  ON  <onclause>
```

由于我们错过了这一点，为了避免进一步的退化，我们通过将字符串`"nested"`指定给`joinedload.innerjoin`来添加上述功能：

```py
query(User).options(
    joinedload("orders", innerjoin=False).joinedload("items", innerjoin="nested")
)
```

这个功能是在 0.9.4 中新增的。

[#2976](https://www.sqlalchemy.org/trac/ticket/2976)

### ORM 可以高效地使用 RETURNING 获取刚生成的 INSERT/UPDATE 默认值

`Mapper`长期以来支持一个名为`eager_defaults=True`的未记录标志。这个标志的效果是，当进行 INSERT 或 UPDATE 时，如果知道行具有服务器生成的默认值，那么会立即跟随一个 SELECT 以“急切地”加载这些新值。通常，服务器生成的列会在对象上标记为“过期”，因此除非应用程序在刷新后立即访问这些列，否则不会产生任何开销。因此，`eager_defaults`标志并没有太大用处，因为它只会降低性能，并且只存在于支持需要默认值在刷新过程中立即可用的奇特事件方案中。

在 0.9 版本中，由于版本 id 增强，`eager_defaults`现在可以为这些值发出一个 RETURNING 子句，因此在具有强大 RETURNING 支持的后端，特别是 PostgreSQL 中，ORM 可以在 INSERT 或 UPDATE 中内联获取新生成的默认值和 SQL 表达式值。当启用`eager_defaults`时，当目标后端和`Table`支持“隐式返回”时，会自动使用 RETURNING。

### 子查询急加载将对某些查询的最内层 SELECT 应用 DISTINCT

为了减少在涉及到多对一关系时子查询急加载可能生成的重复行数，当连接的目标是不包含主键的列时，将在最内层的 SELECT 中应用 DISTINCT 关键字，就像在加载多对一关系时一样。

也就是说，在从 A->B 进行子查询加载时：

```py
SELECT  b.id  AS  b_id,  b.name  AS  b_name,  anon_1.b_id  AS  a_b_id
FROM  (SELECT  DISTINCT  a_b_id  FROM  a)  AS  anon_1
JOIN  b  ON  b.id  =  anon_1.a_b_id
```

由于`a.b_id`是一个非唯一的外键，所以应用了 DISTINCT 以消除冗余的`a.b_id`。可以针对特定的`relationship()`使用`distinct_target_key`标志来无条件地打开或关闭此行为，将值设置为`True`表示无条件打开，`False`表示无条件关闭，`None`表示当目标 SELECT 针对不包含完整主键的列时才生效。在 0.9 版本中，`None`是默认值。

该选项也被回溯到了 0.8 版本，其中`distinct_target_key`选项的默认值为`False`。

尽管此功能旨在通过消除重复行来提高性能，但 SQL 中的`DISTINCT`关键字本身可能会对性能产生负面影响。如果 SELECT 中的列没有索引，`DISTINCT`可能会对行集执行`ORDER BY`，这可能是昂贵的。通过将该功能限制在希望在任何情况下都具有索引的外键上，预计新的默认值是合理的。

该功能也不能消除每种可能的重复行情况；如果在连接链中的其他地方存在多对一关系，则可能仍然存在重复行。

[#2836](https://www.sqlalchemy.org/trac/ticket/2836)  ### Backref 处理程序现在可以传播超过一层

属性事件沿着它们的“发起者”传递的机制已经发生了变化；不再传递`AttributeImpl`，而是传递一个新的对象`Event`；这个对象同时指向`AttributeImpl`和一个“操作令牌”，表示操作是追加、删除还是替换操作。

属性事件系统不再查看这个“initiator”对象以阻止一系列递归属性事件。相反，防止由于相互依赖的返回处理程序而导致无限递归的系统已经移动到了 ORM 返回事件处理程序中，这些处理程序现在接管了确保一系列相互依赖事件（例如向集合 A.bs 添加，响应中设置多对一属性 B.a）不会进入无限递归流的角色。这里的理念是，给予返回系统更多的细节和对事件传播的控制，最终可以允许操作深于一个级别的发生；典型情况是集合追加导致多对一替换操作，然后应该导致从以前的集合中移除该项：

```py
class Parent(Base):
    __tablename__ = "parent"

    id = Column(Integer, primary_key=True)
    children = relationship("Child", backref="parent")

class Child(Base):
    __tablename__ = "child"

    id = Column(Integer, primary_key=True)
    parent_id = Column(ForeignKey("parent.id"))

p1 = Parent()
p2 = Parent()
c1 = Child()

p1.children.append(c1)

assert c1.parent is p1  # backref event establishes c1.parent as p1

p2.children.append(c1)

assert c1.parent is p2  # backref event establishes c1.parent as p2
assert c1 not in p1.children  # second backref event removes c1 from p1.children
```

在此之前，在此更改之前，`c1`对象仍然存在于`p1.children`中，即使它同时也存在于`p2.children`中；返回处理程序将停止替换`c1.parent`为`p2`而不是`p1`。在 0.9 版本中，使用更详细的`Event`对象以及让返回处理程序对这些对象做出更详细的决策，传播可以继续到从`p1.children`中移除`c1`，同时保持对传播进入无限递归循环的检查。

使用 `AttributeEvents.set()`、`AttributeEvents.append()` 或 `AttributeEvents.remove()` 事件的最终用户代码，并且作为这些事件的结果启动进一步的属性修改操作的可能需要进行修改，以防止递归循环，因为在没有返回事件处理程序的情况下，属性系统不再阻止一系列事件无休止地传播。此外，依赖于`initiator`值的代码将需要调整到新的 API，并且必须准备好在一系列由返回引发的事件中，`initiator`的值从其原始值更改为其他值，因为返回处理程序现在可能会为某些操作替换新的`initiator`值。

[#2789](https://www.sqlalchemy.org/trac/ticket/2789)  ### 类型系统现在处理呈现“文字绑定”值的任务

一个新的方法被添加到`TypeEngine` `TypeEngine.literal_processor()`以及`TypeDecorator.process_literal_param()`，用于处理所谓的“内联文字参数” - 通常呈现为“绑定”值的参数，但由于编译器配置的原因而被内联渲染到 SQL 语句中。此功能在生成构造如`CheckConstraint`的 DDL 时使用，以及当使用像 `op.inline_literal()` 这样的构造时，由 Alembic 使用。之前，一个简单的“isinstance”检查仅检查了几种基本类型，并且“绑定处理器”无条件地被使用，导致诸如字符串过早编码为 utf-8 等问题。

使用`TypeDecorator`编写的自定义类型应继续在“内联文字”场景中工作，因为`TypeDecorator.process_literal_param()`默认情况下回退到`TypeDecorator.process_bind_param()`，因为这些方法通常处理数据操作，而不是数据如何呈现给数据库。 `TypeDecorator.process_literal_param()`可以被指定为特别生成表示值应该如何呈现为内联 DDL 语句的字符串。

[#2838](https://www.sqlalchemy.org/trac/ticket/2838)  ### 架构标识符现在携带其自身的引号信息

此更改简化了核心对所谓的“引号”标志的使用，例如传递给`Table`和`Column`的`quote`标志。该标志现在内部化在字符串名称本身中，现在表示为`quoted_name`的实例，一个字符串子类。`IdentifierPreparer`现在仅依赖于由`quoted_name`对象报告的引号偏好，而不再在大多数情况下检查任何显式的`quote`标志。此处解决的问题包括各种区分大小写的方法，如`Engine.has_table()`以及方言内的类似方法现在可以使用显式带引号的名称正常工作，而无需复杂化或引入与引号标志的细节相关的不兼容更改到这些 API（其中许多是第三方）- 特别是，更广泛范围的标识符现在可以与所谓的“大写”后端（如 Oracle、Firebird 和 DB2 等后端，这些后端使用全大写存储和报告表和列名称以用于不区分大小写的名称）正确地运行。

`quoted_name`对象根据需要在内部使用；但是，如果其他关键字需要固定引号偏好，则该类可公开使用。

[#2812](https://www.sqlalchemy.org/trac/ticket/2812)  ### 改进的布尔常量、NULL 常量、连接的渲染

新功能已添加到`true()`和`false()`常量中，特别是与`and_()`和`or_()`函数以及与这些类型、布尔类型总体以及`null()`常量一起使用的 WHERE/HAVING 子句的行为。

从这样的表格开始：

```py
from sqlalchemy import Table, Boolean, Integer, Column, MetaData

t1 = Table("t", MetaData(), Column("x", Boolean()), Column("y", Integer))
```

在不支持`true`/`false`常量行为的后端上，选择构造现在将布尔列呈现为二进制表达式：

```py
>>> from sqlalchemy import select, and_, false, true
>>> from sqlalchemy.dialects import mysql, postgresql

>>> print(select([t1]).where(t1.c.x).compile(dialect=mysql.dialect()))
SELECT  t.x,  t.y  FROM  t  WHERE  t.x  =  1 
```

`and_()`和`or_()`构造现在将表现出准“短路”行为，即当存在`true()`或`false()`常量时，截断呈现的表达式：

```py
>>> print(
...     select([t1]).where(and_(t1.c.y > 5, false())).compile(dialect=postgresql.dialect())
... )
SELECT  t.x,  t.y  FROM  t  WHERE  false 
```

`true()`可以用作构建表达式的基础：

```py
>>> expr = true()
>>> expr = expr & (t1.c.y > 5)
>>> print(select([t1]).where(expr))
SELECT  t.x,  t.y  FROM  t  WHERE  t.y  >  :y_1 
```

布尔常量`true()`和`false()`本身在没有布尔常量的后端上呈现为`0 = 1`和`1 = 1`：

```py
>>> print(select([t1]).where(and_(t1.c.y > 5, false())).compile(dialect=mysql.dialect()))
SELECT  t.x,  t.y  FROM  t  WHERE  0  =  1 
```

`None`的解释，虽然不是特别有效的 SQL，但至少现在是一致的：

```py
>>> print(select([t1.c.x]).where(None))
SELECT  t.x  FROM  t  WHERE  NULL
>>> print(select([t1.c.x]).where(None).where(None))
SELECT  t.x  FROM  t  WHERE  NULL  AND  NULL
>>> print(select([t1.c.x]).where(and_(None, None)))
SELECT  t.x  FROM  t  WHERE  NULL  AND  NULL 
```

[#2804](https://www.sqlalchemy.org/trac/ticket/2804)  ### 标签构造现在可以仅在 ORDER BY 中呈现为它们的名称

对于在 SELECT 的列子句和 ORDER BY 子句中都使用`Label`的情况，假设底层方言报告支持此功能，则标签将仅在 ORDER BY 子句中呈现为其名称。

例如一个示例如：

```py
from sqlalchemy.sql import table, column, select, func

t = table("t", column("c1"), column("c2"))
expr = (func.foo(t.c.c1) + t.c.c2).label("expr")

stmt = select([expr]).order_by(expr)

print(stmt)
```

在 0.9 之前将呈现为：

```py
SELECT  foo(t.c1)  +  t.c2  AS  expr
FROM  t  ORDER  BY  foo(t.c1)  +  t.c2
```

现在呈现为：

```py
SELECT  foo(t.c1)  +  t.c2  AS  expr
FROM  t  ORDER  BY  expr
```

仅当标签未进一步嵌入到 ORDER BY 中的表达式中时，ORDER BY 才会呈现标签，除了简单的`ASC`或`DESC`。

上述格式在所有经过测试的数据库上都有效，但可能与旧数据库版本（MySQL 4？Oracle 8？等）存在兼容性问题。根据用户报告，我们可以添加规则，根据数据库版本检测禁用该功能。

[#1068](https://www.sqlalchemy.org/trac/ticket/1068)  ### `RowProxy`现在具有元组排序行为

`RowProxy`对象的行为很像元组，但直到现在，如果使用`sorted()`对它们的列表进行排序，它们不会像元组一样排序。现在`__eq__()`方法将两侧都作为元组进行比较，还添加了一个`__lt__()`方法：

```py
users.insert().execute(
    dict(user_id=1, user_name="foo"),
    dict(user_id=2, user_name="bar"),
    dict(user_id=3, user_name="def"),
)

rows = users.select().order_by(users.c.user_name).execute().fetchall()

eq_(rows, [(2, "bar"), (3, "def"), (1, "foo")])

eq_(sorted(rows), [(1, "foo"), (2, "bar"), (3, "def")])
```

[#2848](https://www.sqlalchemy.org/trac/ticket/2848)  ### 当类型可用时，没有类型的 bindparam()构造会通过复制进行升级

将“升级”`bindparam()`构造以采用封闭表达式类型的逻辑已经以两种方式得到改进。首先，在分配新类型之前，会**复制**`bindparam()`对象，以便给定的`bindparam()`不会在原地改变。其次，在编译`Insert`或`Update`构造时，会对通过`ValuesBase.values()`方法在语句中设置的“values”进行相同的操作。

如果给定一个未指定类型的`bindparam()`：

```py
bp = bindparam("some_col")
```

如果我们像下面这样使用这个参数：

```py
expr = mytable.c.col == bp
```

对于`bp`的类型仍然是`NullType`，但是如果`mytable.c.col`的类型是`String`，那么`expr.right`，即二进制表达式的右侧，将采用`String`类型。以前，`bp`本身会被直接更改为`String`类型。

类似地，这个操作发生在`Insert`或`Update`中：

```py
stmt = mytable.update().values(col=bp)
```

在上面的例子中，`bp`保持不变，但当语句执行时将使用`String`类型，我们可以通过检查`binds`字典来看到这一点：

```py
>>> compiled = stmt.compile()
>>> compiled.binds["some_col"].type
String
```

该功能允许自定义类型在 INSERT/UPDATE 语句中发挥其预期效果，而无需在每个`bindparam()`表达式中显式指定这些类型。

可能的向后兼容更改涉及两种不太可能的情况。由于绑定参数是**克隆**的，用户不应该依赖于对创建后的`bindparam()`构造进行原地更改。此外，使用`Insert`或`Update`语句中的`bindparam()`的代码，如果依赖于`bindparam()`不根据分配给的列进行类型化，则不再以这种方式运行。

[#2850](https://www.sqlalchemy.org/trac/ticket/2850)  ### Columns can reliably get their type from a column referred to via ForeignKey

有一个长期存在的行为，它规定可以声明一个`Column`而不需要类型，只要这个`Column`被一个`ForeignKeyConstraint`所引用，并且被引用的列的类型会被复制到这个列中。问题在于，这个特性从来没有很好地工作过，并且没有得到维护。核心问题是`ForeignKey`对象在被询问之前不知道它引用的目标`Column`是哪一个，通常是第一次使用外键来构造一个`Join`时。因此，在那之前，父`Column`将没有类型，或者更具体地说，它将具有一个`NullType`的默认类型。

虽然花费了很长时间，重新组织`ForeignKey`对象的初始化工作已经完成，以便使这一功能最终能够令人满意地运行。这一变化的核心是，`ForeignKey.column`属性不再延迟初始化目标`Column`的位置；这一系统的问题在于，拥有的`Column`会一直被固定为`NullType`类型，直到`ForeignKey`被使用。

在新版本中，`ForeignKey`通过内部附加事件与最终将引用的`Column`协调，因此一旦引用的`Column`与`MetaData`关联，所有引用它的`ForeignKey`对象都将收到一条消息，告诉它们需要初始化其父列。这一系统更加复杂，但更加稳固；作为奖励，现在已经为各种`Column` / `ForeignKey`配置场景设置了测试，并且错误消息已经改进，以便非常具体地指出不少于七种不同的错误条件。

现在可以正确工作的场景包括：

1.  一旦目标`Column`与相同的`MetaData`关联，`Column`上的类型立即存在；无论哪一侧首先配置，这都能正常工作：

    ```py
    >>> from sqlalchemy import Table, MetaData, Column, Integer, ForeignKey
    >>> metadata = MetaData()
    >>> t2 = Table("t2", metadata, Column("t1id", ForeignKey("t1.id")))
    >>> t2.c.t1id.type
    NullType()
    >>> t1 = Table("t1", metadata, Column("id", Integer, primary_key=True))
    >>> t2.c.t1id.type
    Integer()
    ```

1.  系统现在也与`ForeignKeyConstraint`一起工作：

    ```py
    >>> from sqlalchemy import Table, MetaData, Column, Integer, ForeignKeyConstraint
    >>> metadata = MetaData()
    >>> t2 = Table(
    ...     "t2",
    ...     metadata,
    ...     Column("t1a"),
    ...     Column("t1b"),
    ...     ForeignKeyConstraint(["t1a", "t1b"], ["t1.a", "t1.b"]),
    ... )
    >>> t2.c.t1a.type
    NullType()
    >>> t2.c.t1b.type
    NullType()
    >>> t1 = Table(
    ...     "t1",
    ...     metadata,
    ...     Column("a", Integer, primary_key=True),
    ...     Column("b", Integer, primary_key=True),
    ... )
    >>> t2.c.t1a.type
    Integer()
    >>> t2.c.t1b.type
    Integer()
    ```

1.  它甚至适用于“多跳” - 也就是，一个`ForeignKey`指向一个`Column`再指向另一个`Column`：

    ```py
    >>> from sqlalchemy import Table, MetaData, Column, Integer, ForeignKey
    >>> metadata = MetaData()
    >>> t2 = Table("t2", metadata, Column("t1id", ForeignKey("t1.id")))
    >>> t3 = Table("t3", metadata, Column("t2t1id", ForeignKey("t2.t1id")))
    >>> t2.c.t1id.type
    NullType()
    >>> t3.c.t2t1id.type
    NullType()
    >>> t1 = Table("t1", metadata, Column("id", Integer, primary_key=True))
    >>> t2.c.t1id.type
    Integer()
    >>> t3.c.t2t1id.type
    Integer()
    ```

[#1765](https://www.sqlalchemy.org/trac/ticket/1765)  ### 许多 JOIN 和 LEFT OUTER JOIN 表达式将不再被包装在(SELECT * FROM ..) AS ANON_1 中

多年来，SQLAlchemy ORM 一直无法将 JOIN 嵌套在现有 JOIN 的右侧（通常是 LEFT OUTER JOIN，因为 INNER JOIN 始终可以被展平）：

```py
SELECT  a.*,  b.*,  c.*  FROM  a  LEFT  OUTER  JOIN  (b  JOIN  c  ON  b.id  =  c.id)  ON  a.id
```

这是因为 SQLite 直到版本**3.7.16**之前无法解析上述格式的语句：

```py
SQLite version 3.7.15.2 2013-01-09 11:53:05
Enter ".help" for instructions
Enter SQL statements terminated with a ";"
sqlite> create table a(id integer);
sqlite> create table b(id integer);
sqlite> create table c(id integer);
sqlite> select a.id, b.id, c.id from a left outer join (b join c on b.id=c.id) on b.id=a.id;
Error: no such column: b.id
```

右外连接当然是另一种解决右侧括号化的方法；这将会显著复杂化并且视觉上不愉快去实现，但幸运的是 SQLite 也不支持 RIGHT OUTER JOIN :):

```py
sqlite>  select  a.id,  b.id,  c.id  from  b  join  c  on  b.id=c.id
  ...>  right  outer  join  a  on  b.id=a.id;
Error:  RIGHT  and  FULL  OUTER  JOINs  are  not  currently  supported
```

回到 2005 年，其他数据库是否有问题这种形式并不清楚，但今天看来，除了 SQLite 之外的每个测试过的数据库现在都支持它（Oracle 8，一个非常古老的数据库，根本不支持 JOIN 关键字，但 SQLAlchemy 一直对 Oracle 的语法有一个简单的重写方案）。更糟糕的是，SQLAlchemy 通常的解决方法，即应用 SELECT，通常会降低像 PostgreSQL 和 MySQL 这样的平台的性能：

```py
SELECT  a.*,  anon_1.*  FROM  a  LEFT  OUTER  JOIN  (
  SELECT  b.id  AS  b_id,  c.id  AS  c_id
  FROM  b  JOIN  c  ON  b.id  =  c.id
  )  AS  anon_1  ON  a.id=anon_1.b_id
```

像上述形式的 JOIN 在处理联接表继承结构时很常见；每当使用`Query.join()`从某个父类连接到联接表子类，或者类似地使用`joinedload()`时，SQLAlchemy 的 ORM 总是确保不会呈现嵌套的 JOIN，以免查询无法在 SQLite 上运行。即使 Core 始终支持更紧凑形式的 JOIN，ORM 也必须避免使用它。

当在 ON 子句中存在特殊条件时，跨多对多关系生成连接时会出现另一个问题。考虑像下面这样的急切加载连接：

```py
session.query(Order).outerjoin(Order.items)
```

假设从`Order`到`Item`的多对多实际上指的是像`Subitem`这样的子类，上述情况的 SQL 将如下所示：

```py
SELECT  order.id,  order.name
FROM  order  LEFT  OUTER  JOIN  order_item  ON  order.id  =  order_item.order_id
LEFT  OUTER  JOIN  item  ON  order_item.item_id  =  item.id  AND  item.type  =  'subitem'
```

上述查询有什么问题？基本上，它将加载许多`order` / `order_item`行，其中`item.type == 'subitem'`的条件不成立。

从 SQLAlchemy 0.9 开始，采取了一种全新的方法。ORM 不再担心在封闭连接的右侧嵌套 JOIN，并且现在将尽可能经常地呈现这些，同时仍然返回正确的结果。当 SQL 语句被传递进行编译时，**方言编译器**将会**重写连接**以适应目标后端，如果该后端已知不支持右嵌套 JOIN（目前只有 SQLite - 如果其他后端有此问题，请告诉我们！）。

因此，一个常规的 `query(Parent).join(Subclass)` 现在通常会产生一个更简单的表达式：

```py
SELECT  parent.id  AS  parent_id
FROM  parent  JOIN  (
  base_table  JOIN  subclass_table
  ON  base_table.id  =  subclass_table.id)  ON  parent.id  =  base_table.parent_id
```

像 `query(Parent).options(joinedload(Parent.subclasses))` 这样的连接急切加载将为各个表创建别名，而不是包装在 `ANON_1` 中：

```py
SELECT  parent.*,  base_table_1.*,  subclass_table_1.*  FROM  parent
  LEFT  OUTER  JOIN  (
  base_table  AS  base_table_1  JOIN  subclass_table  AS  subclass_table_1
  ON  base_table_1.id  =  subclass_table_1.id)
  ON  parent.id  =  base_table_1.parent_id
```

多对多连接和急切加载将右嵌套“次要”和“右”表：

```py
SELECT  order.id,  order.name
FROM  order  LEFT  OUTER  JOIN
(order_item  JOIN  item  ON  order_item.item_id  =  item.id  AND  item.type  =  'subitem')
ON  order_item.order_id  =  order.id
```

所有这些连接，当与明确指定 `use_labels=True` 的 `Select` 语句一起呈现时，这对于 ORM 发出的所有查询都是真实的，都是“连接重写”的候选对象，这是将所有这些右嵌套连接重写为嵌套的 SELECT 语句的过程，同时保持与 `Select` 使用的相同标签。因此，SQLite，即使在 2013 年，仍然不支持这种非常常见的 SQL 语法，也要承担额外的复杂性，以上查询被重写为：

```py
-- sqlite only!
SELECT  parent.id  AS  parent_id
  FROM  parent  JOIN  (
  SELECT  base_table.id  AS  base_table_id,
  base_table.parent_id  AS  base_table_parent_id,
  subclass_table.id  AS  subclass_table_id
  FROM  base_table  JOIN  subclass_table  ON  base_table.id  =  subclass_table.id
  )  AS  anon_1  ON  parent.id  =  anon_1.base_table_parent_id

-- sqlite only!
SELECT  parent.id  AS  parent_id,  anon_1.subclass_table_1_id  AS  subclass_table_1_id,
  anon_1.base_table_1_id  AS  base_table_1_id,
  anon_1.base_table_1_parent_id  AS  base_table_1_parent_id
FROM  parent  LEFT  OUTER  JOIN  (
  SELECT  base_table_1.id  AS  base_table_1_id,
  base_table_1.parent_id  AS  base_table_1_parent_id,
  subclass_table_1.id  AS  subclass_table_1_id
  FROM  base_table  AS  base_table_1
  JOIN  subclass_table  AS  subclass_table_1  ON  base_table_1.id  =  subclass_table_1.id
)  AS  anon_1  ON  parent.id  =  anon_1.base_table_1_parent_id

-- sqlite only!
SELECT  "order".id  AS  order_id
FROM  "order"  LEFT  OUTER  JOIN  (
  SELECT  order_item_1.order_id  AS  order_item_1_order_id,
  order_item_1.item_id  AS  order_item_1_item_id,
  item.id  AS  item_id,  item.type  AS  item_type
FROM  order_item  AS  order_item_1
  JOIN  item  ON  item.id  =  order_item_1.item_id  AND  item.type  IN  (?)
)  AS  anon_1  ON  "order".id  =  anon_1.order_item_1_order_id
```

注意

从 SQLAlchemy 1.1 开始，此功能中存在的针对 SQLite 的解决方法将在检测到 SQLite 版本 **3.7.16** 或更高版本时自动禁用自身，因为 SQLite 已修复了对右嵌套连接的支持。

`Join.alias()`，`aliased()` 和 `with_polymorphic()` 函数现在支持一个新参数 `flat=True`，用于构建连接表实体的别名而不嵌入到 SELECT 中。这个标志默认情况下是关闭的，以帮助向后兼容性 - 但现在一个“多态”可选择可以作为目标连接而不生成任何子查询：

```py
employee_alias = with_polymorphic(Person, [Engineer, Manager], flat=True)

session.query(Company).join(Company.employees.of_type(employee_alias)).filter(
    or_(Engineer.primary_language == "python", Manager.manager_name == "dilbert")
)
```

生成（除了 SQLite 之外的所有地方）：

```py
SELECT  companies.company_id  AS  companies_company_id,  companies.name  AS  companies_name
FROM  companies  JOIN  (
  people  AS  people_1
  LEFT  OUTER  JOIN  engineers  AS  engineers_1  ON  people_1.person_id  =  engineers_1.person_id
  LEFT  OUTER  JOIN  managers  AS  managers_1  ON  people_1.person_id  =  managers_1.person_id
)  ON  companies.company_id  =  people_1.company_id
WHERE  engineers.primary_language  =  %(primary_language_1)s
  OR  managers.manager_name  =  %(manager_name_1)s
```

[#2369](https://www.sqlalchemy.org/trac/ticket/2369) [#2587](https://www.sqlalchemy.org/trac/ticket/2587)

### 右嵌套内连接在连接的急切加载中可用

从版本 0.9.4 开始，在连接的急切加载情况下，可以启用上述提到的右嵌套连接，其中“外部”连接链接到右侧的“内部”连接。

通常，像下面这样的连接急切加载链：

```py
query(User).options(
    joinedload("orders", innerjoin=False).joinedload("items", innerjoin=True)
)
```

不会产生内连接；由于从 user->order 的 LEFT OUTER JOIN，连接的急切加载无法使用从 order->items 的 INNER join 而不更改返回的用户行，并且会忽略“链接”的`innerjoin=True`指令。0.9.0 应该交付的是，而不是：

```py
FROM  users  LEFT  OUTER  JOIN  orders  ON  <onclause>  LEFT  OUTER  JOIN  items  ON  <onclause>
```

新的“右嵌套连接是可以的”逻辑将启动，我们将得到：

```py
FROM  users  LEFT  OUTER  JOIN  (orders  JOIN  items  ON  <onclause>)  ON  <onclause>
```

由于我们错过了这一点，为了避免进一步的退化，我们通过将字符串`"nested"`指定给`joinedload.innerjoin`来添加了上述功能：

```py
query(User).options(
    joinedload("orders", innerjoin=False).joinedload("items", innerjoin="nested")
)
```

此功能是在 0.9.4 中新增的。

[#2976](https://www.sqlalchemy.org/trac/ticket/2976)

### ORM 可以使用 RETURNING 高效地获取刚生成的 INSERT/UPDATE 默认值

`Mapper`长期以来支持一个名为`eager_defaults=True`的未记录标志。此标志的效果是，当进行 INSERT 或 UPDATE 时，并且已知该行具有服务器生成的默认值时，将立即跟随 SELECT 以“急切地”加载这些新值。通常，服务器生成的列会在对象上标记为“过期”，因此除非应用程序在刷新后立即访问这些列，否则不会产生任何开销。因此，`eager_defaults`标志实际上没有太大用处，因为它只会降低性能，并且仅用于支持需要默认值在刷新过程中立即可用的奇特事件方案。

0.9 版本由于版本 ID 增强，`eager_defaults`现在可以为这些值发出一个 RETURNING 子句，因此在具有强大 RETURNING 支持的后端，特别是 PostgreSQL 上，ORM 可以在 INSERT 或 UPDATE 中内联获取新生成的默认值和 SQL 表达式值。`eager_defaults`在启用时，当目标后端和`Table`支持“隐式返回”时，会自动使用 RETURNING。

### 子查询急切加载将对某些查询的最内部 SELECT 应用 DISTINCT

为了减少涉及多对一关系时子查询急切加载可能生成的重复行数，当连接针对不包括主键的列时，将在最内部 SELECT 中应用 DISTINCT 关键字，例如在沿着多对一加载时。

也就是说，当在 A->B 的多对一上进行子查询加载时：

```py
SELECT  b.id  AS  b_id,  b.name  AS  b_name,  anon_1.b_id  AS  a_b_id
FROM  (SELECT  DISTINCT  a_b_id  FROM  a)  AS  anon_1
JOIN  b  ON  b.id  =  anon_1.a_b_id
```

由于`a.b_id`是一个非唯一的外键，因此应用了 DISTINCT，以消除冗余的`a.b_id`。可以通过在特定的`relationship()`上设置`distinct_target_key`标志来无条件地打开或关闭此行为，将值设置为`True`表示无条件打开，`False`表示无条件关闭，`None`表示当目标 SELECT 针对不包含完整主键的列时，该特性生效。在 0.9 版本中，`None`是默认值。

该选项也被回溯到了 0.8 版本，其中`distinct_target_key`选项的默认值为`False`。

虽然这个特性旨在通过消除重复行来提高性能，但 SQL 中的`DISTINCT`关键字本身可能会对性能产生负面影响。如果 SELECT 中的列没有索引，`DISTINCT`可能会对行集执行`ORDER BY`，这可能是昂贵的。通过将该特性限制在希望在任何情况下都有索引的外键上，预计新的默认值是合理的。

该特性也不会消除每种可能的重复行情况；如果在连接链中的其他地方存在多对一关系，则可能仍然存在重复行。

[#2836](https://www.sqlalchemy.org/trac/ticket/2836)

### 反向引用处理程序现在可以传播超过一级深度

属性事件传递其“发起者”的机制已经发生了变化；不再传递`AttributeImpl`，而是传递一个新对象`Event`；这个对象同时指向`AttributeImpl`和一个“操作令牌”，表示操作是追加、移除还是替换操作。

属性事件系统不再查看这个“发起者”对象以阻止属性事件的递归系列。相反，为了防止由于相互依赖的反向引用处理程序而导致的无限递归，现在将这一系统移动到了 ORM 反向引用事件处理程序中，这些处理程序现在负责确保一系列相互依赖的事件（例如向集合 A.bs 追加，在响应中设置多对一属性 B.a）不会进入无限递归流。这里的理念是，反向引用系统，通过更详细和控制事件传播，最终可以允许超过一级深度的操作发生；典型情况是，当集合追加导致多对一替换操作时，这反过来应该导致项目从先前的集合中移除：

```py
class Parent(Base):
    __tablename__ = "parent"

    id = Column(Integer, primary_key=True)
    children = relationship("Child", backref="parent")

class Child(Base):
    __tablename__ = "child"

    id = Column(Integer, primary_key=True)
    parent_id = Column(ForeignKey("parent.id"))

p1 = Parent()
p2 = Parent()
c1 = Child()

p1.children.append(c1)

assert c1.parent is p1  # backref event establishes c1.parent as p1

p2.children.append(c1)

assert c1.parent is p2  # backref event establishes c1.parent as p2
assert c1 not in p1.children  # second backref event removes c1 from p1.children
```

在此更改之前，`c1` 对象仍然会存在于`p1.children`中，即使它同时也存在于`p2.children`中；回引处理程序会在替换`c1.parent`为`p2`而不是`p1`时停止。在 0.9 版本中，使用更详细的`Event`对象，让回引处理程序对这些对象做出更详细的决策，传播可以继续删除`p1.children`中的`c1`，同时保持检查以防止传播进入无限递归循环。

终端用户代码，a. 使用`AttributeEvents.set()`、`AttributeEvents.append()`或`AttributeEvents.remove()`事件，并且 b. 由于这些事件导致进一步的属性修改操作，可能需要修改以防止递归循环，因为在缺少回引事件处理程序的情况下，属性系统不再阻止事件链无限传播。此外，依赖于`initiator`值的代码将需要调整到新的 API，并且必须准备好`initiator`值在一系列由回引引发的事件中从其原始值更改，因为回引处理程序现在可能会为某些操作交换新的`initiator`值。

[#2789](https://www.sqlalchemy.org/trac/ticket/2789)

### 类型系统现在处理呈现“文字绑定”值的任务

为`TypeEngine`添加了一个新方法`TypeEngine.literal_processor()`以及`TypeDecorator.process_literal_param()`用于`TypeDecorator`，它们负责呈现所谓的“内联文字参数” - 通常呈现为“绑定”值的参数，但由于编译器配置的原因而被内联呈现到 SQL 语句中。此功能用于生成诸如`CheckConstraint`等结构的 DDL，以及在使用诸如`op.inline_literal()`之类的结构时，被 Alembic 使用。以前，一个简单的“isinstance”检查检查了一些基本类型，并且“绑定处理程序”无条件地被使用，导致诸如字符串过早编码为 utf-8 等问题。

使用 `TypeDecorator` 编写的自定义类型应继续在“内联文字”场景中工作，因为 `TypeDecorator.process_literal_param()` 默认情况下会退回到 `TypeDecorator.process_bind_param()`，因为这些方法通常处理数据操作，而不是数据如何呈现给数据库。`TypeDecorator.process_literal_param()` 可以指定特定地生成一个表示值应如何呈现为内联 DDL 语句的字符串。

[#2838](https://www.sqlalchemy.org/trac/ticket/2838)

### 现在模式标识符携带自己的引号信息

此更改简化了 Core 对所谓的“引号”标志的使用，例如传递给 `Table` 和 `Column` 的 `quote` 标志。该标志现在内部化在字符串名称本身中，现在表示为 `quoted_name` 的实例，一个字符串子类。`IdentifierPreparer` 现在仅依赖于由 `quoted_name` 对象报告的引号偏好，而不是在大多数情况下检查任何显式的 `quote` 标志。此处解决的问题包括，各种区分大小写的方法，如 `Engine.has_table()` 以及方言内的类似方法现在可以使用显式引号名称正常工作，而无需复杂化或引入对这些 API（其中许多是第三方的）的引号标志的变更。特别是，更广泛范围的标识符现在可以与所谓的“大写”后端（如 Oracle、Firebird 和 DB2）正确地工作，这些后端使用全大写存储和报告不区分大小写的名称的表和列名称。

`quoted_name` 对象在需要时在内部使用；然而，如果其他关键字需要固定引号偏好，该类是公开可用的。

[#2812](https://www.sqlalchemy.org/trac/ticket/2812)

### 改进的布尔常量、NULL 常量、连接的���现

新功能已添加到`true()`和`false()`常量中，特别是与`and_()`和`or_()`函数以及与这些类型、布尔类型总体以及`null()`常量一起使用时的 WHERE/HAVING 子句的行为。

从这样的表开始：

```py
from sqlalchemy import Table, Boolean, Integer, Column, MetaData

t1 = Table("t", MetaData(), Column("x", Boolean()), Column("y", Integer))
```

在不具有`true`/`false`常量行为的后端上，select 构造现在将将布尔列呈现为二进制表达式：

```py
>>> from sqlalchemy import select, and_, false, true
>>> from sqlalchemy.dialects import mysql, postgresql

>>> print(select([t1]).where(t1.c.x).compile(dialect=mysql.dialect()))
SELECT  t.x,  t.y  FROM  t  WHERE  t.x  =  1 
```

`and_()`和`or_()`构造现在将表现出准“短路”行为，即当存在`true()`或`false()`常量时，将截断呈现的表达式：

```py
>>> print(
...     select([t1]).where(and_(t1.c.y > 5, false())).compile(dialect=postgresql.dialect())
... )
SELECT  t.x,  t.y  FROM  t  WHERE  false 
```

`true()`可以用作构建表达式的基础：

```py
>>> expr = true()
>>> expr = expr & (t1.c.y > 5)
>>> print(select([t1]).where(expr))
SELECT  t.x,  t.y  FROM  t  WHERE  t.y  >  :y_1 
```

布尔常量`true()`和`false()`本身在没有布尔常量的后端上呈现为`0 = 1`和`1 = 1`：

```py
>>> print(select([t1]).where(and_(t1.c.y > 5, false())).compile(dialect=mysql.dialect()))
SELECT  t.x,  t.y  FROM  t  WHERE  0  =  1 
```

对`None`的解释，虽然不是特别有效的 SQL，但至少现在是一致的：

```py
>>> print(select([t1.c.x]).where(None))
SELECT  t.x  FROM  t  WHERE  NULL
>>> print(select([t1.c.x]).where(None).where(None))
SELECT  t.x  FROM  t  WHERE  NULL  AND  NULL
>>> print(select([t1.c.x]).where(and_(None, None)))
SELECT  t.x  FROM  t  WHERE  NULL  AND  NULL 
```

[#2804](https://www.sqlalchemy.org/trac/ticket/2804)

### Label 构造现在可以在 ORDER BY 中仅呈现为其名称

对于在 SELECT 的列子句和 ORDER BY 子句中都使用`Label`的情况，假设底层方言报告支持此功能，则标签将仅在 ORDER BY 子句中呈现为其名称。

例如，一个示例：

```py
from sqlalchemy.sql import table, column, select, func

t = table("t", column("c1"), column("c2"))
expr = (func.foo(t.c.c1) + t.c.c2).label("expr")

stmt = select([expr]).order_by(expr)

print(stmt)
```

在 0.9 之前会呈现为：

```py
SELECT  foo(t.c1)  +  t.c2  AS  expr
FROM  t  ORDER  BY  foo(t.c1)  +  t.c2
```

现在呈现为：

```py
SELECT  foo(t.c1)  +  t.c2  AS  expr
FROM  t  ORDER  BY  expr
```

仅当标签未进一步嵌入到 ORDER BY 中的表达式中时，ORDER BY 才会呈现标签，除了简单的`ASC`或`DESC`。

上述格式在所有经过测试的数据库上都有效，但可能与旧版本数据库（MySQL 4？Oracle 8？等）存在兼容性问题。根据用户报告，我们可以添加规则，根据数据库版本检测来禁用该功能。

[#1068](https://www.sqlalchemy.org/trac/ticket/1068)

### `RowProxy`现在具有元组排序行为

`RowProxy`对象的行为很像元组，但直到现在，如果使用`sorted()`对它们的列表进行排序，它们将不会作为元组进行排序。现在，`__eq__()`方法会将两边都作为元组进行比较，同时还添加了一个`__lt__()`方法：

```py
users.insert().execute(
    dict(user_id=1, user_name="foo"),
    dict(user_id=2, user_name="bar"),
    dict(user_id=3, user_name="def"),
)

rows = users.select().order_by(users.c.user_name).execute().fetchall()

eq_(rows, [(2, "bar"), (3, "def"), (1, "foo")])

eq_(sorted(rows), [(1, "foo"), (2, "bar"), (3, "def")])
```

[#2848](https://www.sqlalchemy.org/trac/ticket/2848)

### 当类型可用时，`bindparam()`构造不带类型的通过复制进行升级

对于将`bindparam()`构造升级为采用封闭表达式类型的逻辑已经有了两方面的改进。首先，在分配新类型之前，`bindparam()`对象会被**复制**，以便给定的`bindparam()`不会在原地改变。其次，当编译`Insert`或`Update`构造时，通过`ValuesBase.values()`方法在语句中设置的“values”也会发生相同的操作。

如果给定一个未类型化的`bindparam()`：

```py
bp = bindparam("some_col")
```

如果我们使用这个参数如下：

```py
expr = mytable.c.col == bp
```

对于`bp`的类型仍然是`NullType`，但是如果`mytable.c.col`是`String`类型，则`expr.right`，即二进制表达式的右侧，将采用`String`类型。以前，`bp`本身会被直接更改为具有`String`类型。

类似地，这个操作发生在`Insert`或`Update`中：

```py
stmt = mytable.update().values(col=bp)
```

在上面的例子中，`bp`保持不变，但当语句执行时将使用`String`类型，我们可以通过检查`binds`字典来看到这一点：

```py
>>> compiled = stmt.compile()
>>> compiled.binds["some_col"].type
String
```

该功能允许自定义类型在 INSERT/UPDATE 语句中产生预期效果，而无需在每个`bindparam()`表达式中显式指定这些类型。

潜在的向后兼容更改涉及两种不太可能的情况。由于绑定参数是**克隆**的，用户不应该依赖于对一旦创建的`bindparam()`构造进行就地更改。此外，使用`bindparam()`的代码在`Insert`或`Update`语句中，依赖于`bindparam()`未根据分配给的列进行类型化的事实，将不再以这种方式运行。

[#2850](https://www.sqlalchemy.org/trac/ticket/2850)

### 列可以可靠地从通过 ForeignKey 引用的列获取其类型

存在一个长期存在的行为，即可以声明不带类型的`Column`，只要该`Column`被`ForeignKeyConstraint`引用，并且引用列的类型将被复制到此列中。问题在于，这个功能从未很好地工作过，并且没有得到维护。核心问题在于，`ForeignKey`对象在被要求之前不知道它引用的目标`Column`，通常是第一次外键用于构造`Join`时。因此，在那之前，父`Column`将没有类型，或者更具体地说，它将具有默认类型`NullType`。

虽然花了很长时间，但重新组织 `ForeignKey` 对象初始化的工作已经完成，以便这个功能最终可以令人满意地工作。 这个变化的核心是 `ForeignKey.column` 属性不再延迟初始化目标 `Column` 的位置；这个系统的问题是拥有的 `Column` 会被困在 `NullType` 作为其类型，直到 `ForeignKey` 被使用。

在新版本中，`ForeignKey` 与最终将引用的 `Column` 协调使用内部附加事件，因此一旦引用的 `Column` 与 `MetaData` 关联，所有引用它的 `ForeignKey` 对象都会收到一条消息，告诉它们需要初始化其父列。 这个系统更复杂，但更可靠；作为奖励，现在已经为各种 `Column` / `ForeignKey` 配置方案设置了测试，并且错误消息已经改进为非常具体，涵盖了不少于七种不同的错误条件。

现在正确工作的场景包括：

1.  `Column` 上的类型会在目标 `Column` 与相同的 `MetaData` 关联后立即出现；无论哪一边先配置都可以：

    ```py
    >>> from sqlalchemy import Table, MetaData, Column, Integer, ForeignKey
    >>> metadata = MetaData()
    >>> t2 = Table("t2", metadata, Column("t1id", ForeignKey("t1.id")))
    >>> t2.c.t1id.type
    NullType()
    >>> t1 = Table("t1", metadata, Column("id", Integer, primary_key=True))
    >>> t2.c.t1id.type
    Integer()
    ```

1.  系统现在也可以与 `ForeignKeyConstraint` 一起工作：

    ```py
    >>> from sqlalchemy import Table, MetaData, Column, Integer, ForeignKeyConstraint
    >>> metadata = MetaData()
    >>> t2 = Table(
    ...     "t2",
    ...     metadata,
    ...     Column("t1a"),
    ...     Column("t1b"),
    ...     ForeignKeyConstraint(["t1a", "t1b"], ["t1.a", "t1.b"]),
    ... )
    >>> t2.c.t1a.type
    NullType()
    >>> t2.c.t1b.type
    NullType()
    >>> t1 = Table(
    ...     "t1",
    ...     metadata,
    ...     Column("a", Integer, primary_key=True),
    ...     Column("b", Integer, primary_key=True),
    ... )
    >>> t2.c.t1a.type
    Integer()
    >>> t2.c.t1b.type
    Integer()
    ```

1.  它甚至适用于“多次跳跃” - 即，一个`ForeignKey`指向一个`Column`，该列指向另一个`Column`:

    ```py
    >>> from sqlalchemy import Table, MetaData, Column, Integer, ForeignKey
    >>> metadata = MetaData()
    >>> t2 = Table("t2", metadata, Column("t1id", ForeignKey("t1.id")))
    >>> t3 = Table("t3", metadata, Column("t2t1id", ForeignKey("t2.t1id")))
    >>> t2.c.t1id.type
    NullType()
    >>> t3.c.t2t1id.type
    NullType()
    >>> t1 = Table("t1", metadata, Column("id", Integer, primary_key=True))
    >>> t2.c.t1id.type
    Integer()
    >>> t3.c.t2t1id.type
    Integer()
    ```

[#1765](https://www.sqlalchemy.org/trac/ticket/1765)

## 方言变更

### Firebird `fdb` 现在是默认的 Firebird 方言。

如果创建引擎时没有指定方言，即 `firebird://`，则现在使用 `fdb` 方言。`fdb` 是一个兼容 `kinterbasdb` 的 DBAPI，根据 Firebird 项目，现在是他们官方的 Python 驱动程序。

[#2504](https://www.sqlalchemy.org/trac/ticket/2504)

### Firebird `fdb` 和 `kinterbasdb` 默认设置 `retaining=False`。

`fdb` 和 `kinterbasdb` DBAPI 都支持一个标志 `retaining=True`，可以传递给其连接的 `commit()` 和 `rollback()` 方法。文档中对此标志的解释是，DBAPI 可以重新使用内部事务状态进行后续事务，以提高性能。但是，较新的文档提到了 Firebird 的“垃圾收集”的分析，表明此标志可能对数据库的处理清理任务的能力产生负面影响，并且因此报告了性能的*降低*。

鉴于此信息，目前不清楚此标志实际上如何可用，并且由于它似乎仅是一种性能增强功能，因此现在默认设置为 `False`。可以通过向`create_engine()`调用传递标志 `retaining=True` 来控制值。这是从 0.8.2 开始添加的新标志，因此在 0.8.2 上的应用程序可以根据需要将其设置为 `True` 或 `False`。

另见

`sqlalchemy.dialects.firebird.fdb`

`sqlalchemy.dialects.firebird.kinterbasdb`

[`pythonhosted.org/fdb/usage-guide.html#retaining-transactions`](https://pythonhosted.org/fdb/usage-guide.html#retaining-transactions) - 有关“保留”标志的信息。

[#2763](https://www.sqlalchemy.org/trac/ticket/2763)

### Firebird `fdb` 现在是默认的 Firebird 方言。

如果创建引擎时没有指定方言，即 `firebird://`，则现在使用 `fdb` 方言。`fdb` 是一个兼容 `kinterbasdb` 的 DBAPI，根据 Firebird 项目，现在是他们官方的 Python 驱动程序。

[#2504](https://www.sqlalchemy.org/trac/ticket/2504)

### Firebird `fdb` 和 `kinterbasdb` 默认设置 `retaining=False`。

`fdb`和`kinterbasdb`两个 DBAPI 都支持一个名为`retaining=True`的标志，可以传递给其连接的`commit()`和`rollback()`方法。文档中对这个标志的理由是，DBAPI 可以重用内部事务状态以用于后续事务，以提高性能。然而，更新的文档提到了对 Firebird 的“垃圾回收”的分析，表明这个标志可能会对数据库处理清理任务的能力产生负面影响，并因此被报告为*降低*性能。

鉴于这些信息，目前尚不清楚如何实际使用这个标志，而且由于它似乎只是一个性能增强功能，现在默认值为`False`。可以通过在`create_engine()`调用中传递标志`retaining=True`来控制该值。这是一个新标志，从 0.8.2 版本开始添加，因此在 0.8.2 上的应用程序可以根据需要将其设置为`True`或`False`。

另请参阅

`sqlalchemy.dialects.firebird.fdb`

`sqlalchemy.dialects.firebird.kinterbasdb`

[`pythonhosted.org/fdb/usage-guide.html#retaining-transactions`](https://pythonhosted.org/fdb/usage-guide.html#retaining-transactions) - 有关“retaining”标志的信息。

[#2763](https://www.sqlalchemy.org/trac/ticket/2763)
