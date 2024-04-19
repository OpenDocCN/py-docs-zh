# SQLAlchemy 1.2 中的新内容是什么？

> 原文：[`docs.sqlalchemy.org/en/20/changelog/migration_12.html`](https://docs.sqlalchemy.org/en/20/changelog/migration_12.html)

关于本文档

本文描述了 SQLAlchemy 1.1 版本与 SQLAlchemy 1.2 版本之间的更改。

## 简介

本指南介绍了 SQLAlchemy 版本 1.2 中的新功能，并记录了影响用户将其应用程序从 SQLAlchemy 1.1 系列迁移到 1.2 系列的更改。

请仔细查看行为更改部分，可能会出现不兼容的行为更改。

## 平台支持

### 针对 Python 2.7 及更高版本

SQLAlchemy 1.2 现在将最低 Python 版本提高到 2.7，不再支持 2.6。预计会将不支持 Python 2.6 的新语言特性合并到 1.2 系列中。对于 Python 3 的支持，SQLAlchemy 目前在版本 3.5 和 3.6 上进行了测试。

## ORM 中的新功能和改进

### “Baked” 加载现在是懒加载的默认设置

`sqlalchemy.ext.baked` 扩展是在 1.0 系列中首次引入的，允许构建所谓的 `BakedQuery` 对象，它是一个生成 `Query` 对象的对象，与表示查询结构的缓存键相结合；然后将此缓存键链接到生成的字符串 SQL 语句，以便后续使用具有相同结构的另一个 `BakedQuery` 将绕过构建 `Query` 对象的所有开销，构建内部的核心 `select()` 对象，以及 `select()` 编译为字符串，大大减少了通常与构建和发出 ORM `Query` 对象相关的函数调用开销。

当 ORM 为惰性加载一个`relationship()`构造生成“懒惰”查询时，默认情况下现在使用`BakedQuery`，例如默认的`lazy="select"`关系加载器策略。这将允许在应用程序使用惰性加载查询加载集合和相关对象的范围内显著减少函数调用。以前，此功能在 1.0 和 1.1 中通过使用全局 API 方法或使用`baked_select`策略可用，现在是此行为的唯一实现。该功能还得到了改进，以便对于在延迟加载后生效的具有附加加载器选项的对象仍然可以进行缓存。

可以通过`relationship.bake_queries`标志在每个关系基础上禁用缓存行为，这对于非常罕见的情况非常有用，比如使用不兼容缓存的自定义`Query`实现的关系。

[#3954](https://www.sqlalchemy.org/trac/ticket/3954)  ### 新的“selectin”急切加载，使用 IN 一次加载所有集合

添加了一个名为“selectin”加载的新急切加载器，这在许多方面类似于“子查询”加载，但是生成了一个更简单的 SQL 语句，该语句也可以缓存并且更有效。

给定如下查询：

```py
q = (
    session.query(User)
    .filter(User.name.like("%ed%"))
    .options(subqueryload(User.addresses))
)
```

生成的 SQL 将是针对`User`的查询，然后是`User.addresses`的子查询加载（注意还列出了参数）：

```py
SELECT  users.id  AS  users_id,  users.name  AS  users_name
FROM  users
WHERE  users.name  LIKE  ?
('%ed%',)

SELECT  addresses.id  AS  addresses_id,
  addresses.user_id  AS  addresses_user_id,
  addresses.email_address  AS  addresses_email_address,
  anon_1.users_id  AS  anon_1_users_id
FROM  (SELECT  users.id  AS  users_id
FROM  users
WHERE  users.name  LIKE  ?)  AS  anon_1
JOIN  addresses  ON  anon_1.users_id  =  addresses.user_id
ORDER  BY  anon_1.users_id
('%ed%',)
```

使用“selectin”加载，我们得到一个 SELECT 语句，该语句引用在父查询中加载的实际主键值：

```py
q = (
    session.query(User)
    .filter(User.name.like("%ed%"))
    .options(selectinload(User.addresses))
)
```

产生：

```py
SELECT  users.id  AS  users_id,  users.name  AS  users_name
FROM  users
WHERE  users.name  LIKE  ?
('%ed%',)

SELECT  users_1.id  AS  users_1_id,
  addresses.id  AS  addresses_id,
  addresses.user_id  AS  addresses_user_id,
  addresses.email_address  AS  addresses_email_address
FROM  users  AS  users_1
JOIN  addresses  ON  users_1.id  =  addresses.user_id
WHERE  users_1.id  IN  (?,  ?)
ORDER  BY  users_1.id
(1,  3)
```

上述 SELECT 语句包括以下优点：

+   它不使用子查询，只使用 INNER JOIN，这意味着在像 MySQL 这样不喜欢子查询的数据库上性能会更好。

+   其结构与原始查询无关；与新的扩展的 IN 参数系统结合，我们在大多数情况下可以使用“烘焙”查询来缓存字符串 SQL，从而显著减少每个查询的开销。

+   由于查询仅针对给定的主键标识符列表进行，"selectin" 加载可能与 `Query.yield_per()` 兼容，以便一次操作 SELECT 结果的一部分，前提是数据库驱动程序允许多个同时游标（SQLite、PostgreSQL；**不**是 MySQL 驱动程序或 SQL Server ODBC 驱动程序）。联接式急切加载和子查询急切加载都不兼容 `Query.yield_per()`。

selectin 急切加载的缺点是可能产生大量的 SQL 查询，具有大量的 IN 参数列表。IN 参数列表本身被分组为每组 500 个，因此超过 500 个主对象的结果集将有更多的额外“SELECT IN”查询。此外，对复合主键的支持取决于数据库能否使用包含 IN 的元组，例如 `(table.column_one, table_column_two) IN ((?, ?), (?, ?) (?, ?))`。目前，已知 PostgreSQL 和 MySQL 兼容此语法，SQLite 不兼容。

另请参阅

选择 IN 加载

[#3944](https://www.sqlalchemy.org/trac/ticket/3944)  ### “selectin” 多态加载，使用单独的 IN 查询加载子类

与刚刚描述的“selectin”关系加载功能类似的是“selectin”多态加载。这是一个专门针对联接式急切加载的多态加载功能，允许基本实体的加载通过简单的 SELECT 语句进行，然后额外子类的属性通过额外的 SELECT 语句进行加载：

```py
>>> from sqlalchemy.orm import selectin_polymorphic

>>> query = session.query(Employee).options(
...     selectin_polymorphic(Employee, [Manager, Engineer])
... )

>>> query.all()
SELECT
  employee.id  AS  employee_id,
  employee.name  AS  employee_name,
  employee.type  AS  employee_type
FROM  employee
()

SELECT
  engineer.id  AS  engineer_id,
  employee.id  AS  employee_id,
  employee.type  AS  employee_type,
  engineer.engineer_name  AS  engineer_engineer_name
FROM  employee  JOIN  engineer  ON  employee.id  =  engineer.id
WHERE  employee.id  IN  (?,  ?)  ORDER  BY  employee.id
(1,  2)

SELECT
  manager.id  AS  manager_id,
  employee.id  AS  employee_id,
  employee.type  AS  employee_type,
  manager.manager_name  AS  manager_manager_name
FROM  employee  JOIN  manager  ON  employee.id  =  manager.id
WHERE  employee.id  IN  (?)  ORDER  BY  employee.id
(3,) 
```

另请参阅

使用 selectin_polymorphic()

[#3948](https://www.sqlalchemy.org/trac/ticket/3948)  ### ORM 属性可以接收临时 SQL 表达式

新的 ORM 属性类型 `query_expression()` 被添加，类似于 `deferred()`，不同之处在于它的 SQL 表达式是在查询时确定的，使用了一个新选项 `with_expression()`；如果未指定，则属性默认为 `None`：

```py
from sqlalchemy.orm import query_expression
from sqlalchemy.orm import with_expression

class A(Base):
    __tablename__ = "a"
    id = Column(Integer, primary_key=True)
    x = Column(Integer)
    y = Column(Integer)

    # will be None normally...
    expr = query_expression()

# but let's give it x + y
a1 = session.query(A).options(with_expression(A.expr, A.x + A.y)).first()
print(a1.expr)
```

另请参阅

查询时 SQL 表达式作为映射属性

[#3058](https://www.sqlalchemy.org/trac/ticket/3058)  ### ORM 支持多表删除

ORM `Query.delete()` 方法支持多表条件的 DELETE，就像在支持多表条件的 DELETE 中介绍的那样。该功能的工作方式与在 0.8 中首次引入的 UPDATE 的多表条件相同，并在 Query.update()支持 UPDATE..FROM 中描述。

下面，我们对`SomeEntity`执行 DELETE 操作，并添加一个 FROM 子句（或等效的，取决于后端）对`SomeOtherEntity`进行操作：

```py
query(SomeEntity).filter(SomeEntity.id == SomeOtherEntity.id).filter(
    SomeOtherEntity.foo == "bar"
).delete()
```

另请参阅

支持多表条件的 DELETE

[#959](https://www.sqlalchemy.org/trac/ticket/959)  ### 支持混合属性、复合属性的批量更新

现在混合属性（例如`sqlalchemy.ext.hybrid`）以及复合属性（复合列类型）在使用`Query.update()`时支持在 UPDATE 语句的 SET 子句中使用。

对于混合属性，可以直接使用简单表达式，或者可以使用新的装饰器`hybrid_property.update_expression()`将一个值拆分为多个列/表达式：

```py
class Person(Base):
    # ...

    first_name = Column(String(10))
    last_name = Column(String(10))

    @hybrid.hybrid_property
    def name(self):
        return self.first_name + " " + self.last_name

    @name.expression
    def name(cls):
        return func.concat(cls.first_name, " ", cls.last_name)

    @name.update_expression
    def name(cls, value):
        f, l = value.split(" ", 1)
        return [(cls.first_name, f), (cls.last_name, l)]
```

上面，可以使用以下方式呈现 UPDATE：

```py
session.query(Person).filter(Person.id == 5).update({Person.name: "Dr. No"})
```

类似的功能也适用于复合属性，其中复合值将被拆分为其各个列以进行批量 UPDATE：

```py
session.query(Vertex).update({Edge.start: Point(3, 4)})
```

另请参阅

允许批量 ORM 更新  ### 混合属性支持在子类之间重用，重新定义@getter

`sqlalchemy.ext.hybrid.hybrid_property` 类现在支持在子类中多次调用诸如`@setter`、`@expression`等的变异器，并且现在提供了`@getter`变异器，以便特定的混合属性可以在子类或其他类中重新使用。这与标准 Python 中`@property`的行为类似：

```py
class FirstNameOnly(Base):
    # ...

    first_name = Column(String)

    @hybrid_property
    def name(self):
        return self.first_name

    @name.setter
    def name(self, value):
        self.first_name = value

class FirstNameLastName(FirstNameOnly):
    # ...

    last_name = Column(String)

    @FirstNameOnly.name.getter
    def name(self):
        return self.first_name + " " + self.last_name

    @name.setter
    def name(self, value):
        self.first_name, self.last_name = value.split(" ", maxsplit=1)

    @name.expression
    def name(cls):
        return func.concat(cls.first_name, " ", cls.last_name)
```

上面，`FirstNameOnly.name`混合属性被`FirstNameLastName`子类引用，以便将其专门用于新子类。这是通过在每次调用`@getter`、`@setter`以及所有其他变异器方法（如`@expression`）中将混合对象复制到新对象中来实现的，从而保持先前混合属性的定义不变。以前，诸如`@setter`的方法会直接修改现有的混合属性，干扰了超类的定义。

注意

请务必阅读在子类之间重用混合属性处的文档，了解如何覆盖`hybrid_property.expression()`和`hybrid_property.comparator()`的重要注意事项，因为在某些情况下可能需要一个特殊的限定符`hybrid_property.overrides`来避免与`QueryableAttribute`发生名称冲突。

注意

这种对`@hybrid_property`的更改意味着，当向`@hybrid_property`添加 setter 和其他状态时，**方法必须保留原始混合的名称**，否则具有附加状态的新混合将作为不匹配的名称存在于类中。这与标准 Python 的`@property`构造的行为相同：

```py
class FirstNameOnly(Base):
    @hybrid_property
    def name(self):
        return self.first_name

    # WRONG - will raise AttributeError: can't set attribute when
    # assigning to .name
    @name.setter
    def _set_name(self, value):
        self.first_name = value

class FirstNameOnly(Base):
    @hybrid_property
    def name(self):
        return self.first_name

    # CORRECT - note regular Python @property works the same way
    @name.setter
    def name(self, value):
        self.first_name = value
```

[#3911](https://www.sqlalchemy.org/trac/ticket/3911)

[#3912](https://www.sqlalchemy.org/trac/ticket/3912)  ### 新的 bulk_replace 事件

为了适应 A @validates method receives all values on bulk-collection set before comparison 中描述的验证用例，添加了一个新的`AttributeEvents.bulk_replace()`方法，该方法与`AttributeEvents.append()`和`AttributeEvents.remove()`事件一起调用。“bulk_replace”在“append”和“remove”之前调用，以便在与现有集合进行比较之前修改集合。之后，单个项目将附加到新的目标集合，触发对于集合中新项目的“append”事件，这与以前的行为相同。下面同时说明了“bulk_replace”和“append”，包括“append”将接收已由“bulk_replace”处理的对象（如果使用集合赋值）。一个新的符号`attributes.OP_BULK_REPLACE`可用于确定此“append”事件是否是批量替换的第二部分：

```py
from sqlalchemy.orm.attributes import OP_BULK_REPLACE

@event.listens_for(SomeObject.collection, "bulk_replace")
def process_collection(target, values, initiator):
    values[:] = [_make_value(value) for value in values]

@event.listens_for(SomeObject.collection, "append", retval=True)
def process_collection(target, value, initiator):
    # make sure bulk_replace didn't already do it
    if initiator is None or initiator.op is not OP_BULK_REPLACE:
        return _make_value(value)
    else:
        return value
```

[#3896](https://www.sqlalchemy.org/trac/ticket/3896)  ### 新的“modified”事件处理程序用于 sqlalchemy.ext.mutable

添加了新的事件处理程序`AttributeEvents.modified()`，它与对`flag_modified()`方法的调用对应，通常从`sqlalchemy.ext.mutable`扩展调用：

```py
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.ext.mutable import MutableDict
from sqlalchemy import event

Base = declarative_base()

class MyDataClass(Base):
    __tablename__ = "my_data"
    id = Column(Integer, primary_key=True)
    data = Column(MutableDict.as_mutable(JSONEncodedDict))

@event.listens_for(MyDataClass.data, "modified")
def modified_json(instance):
    print("json value modified:", instance.data)
```

上述情况下，当对`.data`字典进行原地更改时，事件处理程序将被触发。

[#3303](https://www.sqlalchemy.org/trac/ticket/3303)  ### 在 Session.refresh 中添加了“for update”参数

为`Session.refresh()`方法添加了新参数`Session.refresh.with_for_update`。当`Query.with_lockmode()`方法被弃用，而是采用`Query.with_for_update()`时，`Session.refresh()`方法从未更新以反映新选项：

```py
session.refresh(some_object, with_for_update=True)
```

`Session.refresh.with_for_update` 参数接受一个选项字典，该字典将作为与`Query.with_for_update()`发送的相同参数一样发送的参数：

```py
session.refresh(some_objects, with_for_update={"read": True})
```

新参数取代了`Session.refresh.lockmode` 参数。

[#3991](https://www.sqlalchemy.org/trac/ticket/3991)  ### 原地突变操作符适用于 MutableSet、MutableList

为`MutableSet`实现了原地突变操作符`__ior__`、`__iand__`、`__ixor__`和`__isub__`，以及`MutableList`的`__iadd__`。虽然这些方法以前可以成功更新集合，但它们不会正确地触发更改事件。这些操作符像以前一样突变集合，但额外地发出正确的更改事件，以便更改成为下一个刷新过程的一部分：

```py
model = session.query(MyModel).first()
model.json_set &= {1, 3}
```

[#3853](https://www.sqlalchemy.org/trac/ticket/3853)  ### AssociationProxy 的 any()、has()、contains() 方法与链式关联代理一起工作

`AssociationProxy.any()`、`AssociationProxy.has()`和`AssociationProxy.contains()`比较方法现在支持链接到一个属性，该属性本身也是`AssociationProxy`，递归地。下面，`A.b_values`是一个关联代理，链接到`AtoB.bvalue`，而`AtoB.bvalue`本身是一个关联代理，链接到`B`：

```py
class A(Base):
    __tablename__ = "a"
    id = Column(Integer, primary_key=True)

    b_values = association_proxy("atob", "b_value")
    c_values = association_proxy("atob", "c_value")

class B(Base):
    __tablename__ = "b"
    id = Column(Integer, primary_key=True)
    a_id = Column(ForeignKey("a.id"))
    value = Column(String)

    c = relationship("C")

class C(Base):
    __tablename__ = "c"
    id = Column(Integer, primary_key=True)
    b_id = Column(ForeignKey("b.id"))
    value = Column(String)

class AtoB(Base):
    __tablename__ = "atob"

    a_id = Column(ForeignKey("a.id"), primary_key=True)
    b_id = Column(ForeignKey("b.id"), primary_key=True)

    a = relationship("A", backref="atob")
    b = relationship("B", backref="atob")

    b_value = association_proxy("b", "value")
    c_value = association_proxy("b", "c")
```

我们可以使用`AssociationProxy.contains()`在`A.b_values`上进行查询，以跨两个代理`A.b_values`、`AtoB.b_value`进行查询：

```py
>>> s.query(A).filter(A.b_values.contains("hi")).all()
SELECT  a.id  AS  a_id
FROM  a
WHERE  EXISTS  (SELECT  1
FROM  atob
WHERE  a.id  =  atob.a_id  AND  (EXISTS  (SELECT  1
FROM  b
WHERE  b.id  =  atob.b_id  AND  b.value  =  :value_1))) 
```

类似地，我们可以使用`AssociationProxy.any()`在`A.c_values`上进行查询，以跨两个代理`A.c_values`、`AtoB.c_value`进行查询：

```py
>>> s.query(A).filter(A.c_values.any(value="x")).all()
SELECT  a.id  AS  a_id
FROM  a
WHERE  EXISTS  (SELECT  1
FROM  atob
WHERE  a.id  =  atob.a_id  AND  (EXISTS  (SELECT  1
FROM  b
WHERE  b.id  =  atob.b_id  AND  (EXISTS  (SELECT  1
FROM  c
WHERE  b.id  =  c.b_id  AND  c.value  =  :value_1))))) 
```

[#3769](https://www.sqlalchemy.org/trac/ticket/3769)  ### 身份键增强以支持分片

现在 ORM 使用的身份键结构包含一个额外的成员，以便来自不同上下文的两个相同的主键可以共存于同一身份映射中。

水平分片中的示例已更新以说明这种行为。示例展示了一个分片类`WeatherLocation`，引用一个依赖的`WeatherReport`对象，其中`WeatherReport`类映射到一个存储简单整数主键的表。来自不同数据库的两个`WeatherReport`对象可能具有相同的主键值。该示例现在说明了一个新的`identity_token`字段跟踪这种差异，以便这两个对象可以共存于同一身份映射中：

```py
tokyo = WeatherLocation("Asia", "Tokyo")
newyork = WeatherLocation("North America", "New York")

tokyo.reports.append(Report(80.0))
newyork.reports.append(Report(75))

sess = create_session()

sess.add_all([tokyo, newyork, quito])

sess.commit()

# the Report class uses a simple integer primary key.  So across two
# databases, a primary key will be repeated.  The "identity_token" tracks
# in memory that these two identical primary keys are local to different
# databases.

newyork_report = newyork.reports[0]
tokyo_report = tokyo.reports[0]

assert inspect(newyork_report).identity_key == (Report, (1,), "north_america")
assert inspect(tokyo_report).identity_key == (Report, (1,), "asia")

# the token representing the originating shard is also available directly

assert inspect(newyork_report).identity_token == "north_america"
assert inspect(tokyo_report).identity_token == "asia"
```

[#4137](https://www.sqlalchemy.org/trac/ticket/4137)

## 新功能和改进 - 核心

### 布尔数据类型现在强制使用严格的 True/False/None 值

在 1.1 版本中，描述的更改将非本地布尔整数值强制转换为零/一/None 产生了一个意外的副作用，改变了当`Boolean`遇到非整数值（如字符串）时的行为。特别是，先前会生成值`False`的字符串值`"0"`，现在会产生`True`。更糟糕的是，行为的改变只针对某些后端而不是其他后端，这意味着将字符串`"0"`值发送给`Boolean`的代码在各个后端上会不一致地中断。

这个问题的最终解决方案是**不支持字符串值与布尔值**，因此在 1.2 版本中，如果传递了非整数/True/False/None 值，将引发严格的`TypeError`。此外，只接受整数值 0 和 1。

为了适应希望对布尔值有更自由解释的应用程序，应使用`TypeDecorator`。下面演示了一个配方，允许对 1.1 版本之前的`Boolean`数据类型进行“自由”行为：

```py
from sqlalchemy import Boolean
from sqlalchemy import TypeDecorator

class LiberalBoolean(TypeDecorator):
    impl = Boolean

    def process_bind_param(self, value, dialect):
        if value is not None:
            value = bool(int(value))
        return value
```

[#4102](https://www.sqlalchemy.org/trac/ticket/4102)  ### 连接池中添加了悲观断开检测

连接池文档长期以来一直提供了一个使用`ConnectionEvents.engine_connect()`引擎事件在检出的连接上发出简单语句以测试其活动性的方法。现在，当与适当的方言一起使用时，此配方的功能已添加到连接池本身中。使用新参数`create_engine.pool_pre_ping`，每个检出的连接在返回之前都将被测试是否新鲜：

```py
engine = create_engine("mysql+pymysql://", pool_pre_ping=True)
```

虽然“预先 ping”方法会在连接池检出时增加一点延迟，但对于典型的面向事务的应用程序（包括大多数 ORM 应用程序），这种开销是很小的，并且消除了获取到一个过时连接会引发错误的问题，需要应用程序放弃或重试操作。

该功能**不**适用于在进行中的事务或 SQL 操作中断开的连接。如果应用程序必须从这些错误中恢复，它需要使用自己的操作重试逻辑来预期这些错误。

另请参阅

断开处理 - 悲观

[#3919](https://www.sqlalchemy.org/trac/ticket/3919)  ### IN / NOT IN 运算符的空集合行为现在是可配置的；默认表达式简化了

诸如`column.in_([])`这样的表达式，假定为 false，现在默认产生表达式`1 != 1`，而不是`column != column`。这将**改变查询结果**，比较 SQL 表达式或列与空集合时，产生一个布尔值 false 或 true（对于 NOT IN），而不是 NULL。在这种情况下发出的警告也被移除了。可以使用`create_engine.empty_in_strategy`参数来`create_engine()`获取旧的行为。

在 SQL 中，IN 和 NOT IN 运算符不支持与明确为空的值集合进行比较；也就是说，这种语法是非法的：

```py
mycolumn  IN  ()
```

为了解决这个问题，SQLAlchemy 和其他数据库库检测到这种情况，并渲染一个替代表达式，该表达式评估为 false，或者在 NOT IN 的情况下评估为 true，基于“col IN ()”始终为 false 的理论，因为“空集合”中没有任何内容。通常，为了生成一个跨数据库可移植且在 WHERE 子句上下文中起作用的 false/true 常量，通常使用简单的重言式，如`1 != 1`评估为 false，`1 = 1`评估为 true（简单的常量“0”或“1”通常不能作为 WHERE 子句的目标）。

SQLAlchemy 在早期也采用了这种方法，但很快有人推测 SQL 表达式`column IN ()`如果“column”为 NULL，则不会评估为 false；相反，该表达式会产生 NULL，因为“NULL”表示“未知”，在 SQL 中与 NULL 的比较通常产生 NULL。

为了模拟这个结果，SQLAlchemy 从使用`1 != 1`改为使用表达式`expr != expr`来处理空的“IN”，并使用`expr = expr`来处理空的“NOT IN”；也就是说，我们使用表达式的实际左侧而不是固定值。如果传递的表达式左侧求值为 NULL，则整体比较结果也会得到 NULL 结果，而不是 false 或 true。

不幸的是，用户最终抱怨说这种表达式对一些查询规划器的性能影响非常严重。在那时，当遇到空的 IN 表达式时，会添加警告，建议 SQLAlchemy 继续保持“正确”，并敦促用户避免通常可以安全省略的生成空 IN 谓词的代码。然而，在动态构建查询的情况下，这当然会增加负担，因为输入变量的一组值可能为空。

最近几个月，这个决定的最初假设受到了质疑。表达式“NULL IN ()”应该返回 NULL 的想法只是理论上的，无法测试，因为数据库不支持该语法。然而，事实证明，实际上可以通过模拟空集合来询问关系数据库对于“NULL IN ()”会返回什么值：

```py
SELECT  NULL  IN  (SELECT  1  WHERE  1  !=  1)
```

通过上述测试，我们看到数据库本身无法就答案达成一致。大多数人认为最“正确”的数据库 PostgreSQL 返回 False；因为即使“NULL”代表“未知”，“空集合”意味着没有任何内容，包括所有未知值。另一方面，MySQL 和 MariaDB 对上述表达式返回 NULL，采用更常见的“所有与 NULL 的比较都返回 NULL”的行为。

SQLAlchemy 的 SQL 架构比在做出此设计决定时更复杂，因此现在可以在 SQL 字符串编译时调用任一行为。以前，转换为比较表达式是在构造时完成的，也就是说，在调用`ColumnOperators.in_()`或`ColumnOperators.notin_()`运算符时。使用编译时行为，可以指示方言本身调用任一方法，即“static”`1 != 1`比较或“dynamic”`expr != expr`比较。默认已被**更改**为“static”比较，因为这与 PostgreSQL 在任何情况下的行为一致，这也是绝大多数用户喜欢的。这将**改变查询结果**，特别是将空表达式与空集进行比较的查询，特别是查询否定`where(~null_expr.in_([]))`，因为现在这将评估为 true 而不是 NULL。

现在可以使用标志`create_engine.empty_in_strategy`来控制行为，其默认设置为`"static"`，但也可以设置为`"dynamic"`或`"dynamic_warn"`，其中`"dynamic_warn"`设置等同于以前发出`expr != expr`以及性能警告的行为。然而，预计大多数用户会喜欢`"static"`默认设置。

[#3907](https://www.sqlalchemy.org/trac/ticket/3907)  ### 允许使用缓存语句的延迟扩展 IN 参数集合

添加了一种名为“expanding”的新类型`bindparam()`。这用于在语句执行时将元素列表渲染为单独的绑定参数，而不是在语句编译时。这允许将单个绑定参数名称链接到多个元素的 IN 表达式，同时还允许使用查询缓存与 IN 表达式。这一新功能允许相关功能“select in”加载和“polymorphic in”加载利用烘焙查询扩展来减少调用开销：

```py
stmt = select([table]).where(table.c.col.in_(bindparam("foo", expanding=True)))
conn.execute(stmt, {"foo": [1, 2, 3]})
```

该功能在 1.2 系列中应被视为**实验性**。

[#3953](https://www.sqlalchemy.org/trac/ticket/3953)  ### 压平比较运算符的运算符优先级

像 IN、LIKE、equals、IS、MATCH 和其他比较运算符的运算符优先级已经被压平到一个级别。当比较运算符组合在一起时，将生成更多的括号，例如：

```py
(column("q") == null()) != (column("y") == null())
```

现在将生成`(q IS NULL) != (y IS NULL)`而不是`q IS NULL != y IS NULL`。

[#3999](https://www.sqlalchemy.org/trac/ticket/3999)  ### 支持在 Table、Column 上的 SQL 注释，包括 DDL、反射

Core 接收了与表和列关联的字符串注释的支持。这些通过`Table.comment`和`Column.comment`参数指定：

```py
Table(
    "my_table",
    metadata,
    Column("q", Integer, comment="the Q value"),
    comment="my Q table",
)
```

上面的 DDL 将在表创建时适当地呈现，以将上述注释与模式中的表/列关联起来。当上述表被 autoload 或使用`Inspector.get_columns()`检查时，注释将被包含在内。表注释也可以独立使用`Inspector.get_table_comment()`方法获得。

当前后端支持包括 MySQL，PostgreSQL 和 Oracle。

[#1546](https://www.sqlalchemy.org/trac/ticket/1546)  ### 支持多表条件的 DELETE

`Delete` 构造现在支持多表条件，已在支持的后端实现，目前支持的后端有 PostgreSQL，MySQL 和 Microsoft SQL Server（对目前不工作的 Sybase 方言也添加了支持）。该功能的工作方式与 0.7 和 0.8 系列中首次引入的 UPDATE 的多表条件相同。

给定一个语句如下：

```py
stmt = (
    users.delete()
    .where(users.c.id == addresses.c.id)
    .where(addresses.c.email_address.startswith("ed%"))
)
conn.execute(stmt)
```

在 PostgreSQL 后端上，上述语句的生成 SQL 将呈现为：

```py
DELETE  FROM  users  USING  addresses
WHERE  users.id  =  addresses.id
AND  (addresses.email_address  LIKE  %(email_address_1)s  ||  '%%')
```

另请参阅

多表删除

[#959](https://www.sqlalchemy.org/trac/ticket/959)  ### 新的“autoescape”选项用于 startswith()，endswith()

“autoescape”参数被添加到`ColumnOperators.startswith()`，`ColumnOperators.endswith()`，`ColumnOperators.contains()`。当设置为`True`时，此参数将自动转义所有出现的`%`、`_`，并使用默认的转义字符，默认为斜杠`/`；转义字符本身的出现也会被转义。斜杠用于避免与诸如 PostgreSQL 的`standard_confirming_strings`（从 PostgreSQL 9.1 开始默认值已更改）和 MySQL 的`NO_BACKSLASH_ESCAPES`设置等设置发生冲突。现在可以使用现有的“escape”参数来更改自动转义字符，如果需要的话。

注意

从 1.2.0b2 的初始实现到 1.2.0，此功能已更改，现在 autoescape 被传递为布尔值，而不是用作转义字符的特定字符。

例如一个表达式：

```py
>>> column("x").startswith("total%score", autoescape=True)
```

渲染为：

```py
x  LIKE  :x_1  ||  '%'  ESCAPE  '/'
```

参数“x_1”的值为`'total/%score'`。

同样，一个带有反斜杠的表达式：

```py
>>> column("x").startswith("total/score", autoescape=True)
```

将以相同方式渲染，参数“x_1”的值为`'total//score'`。

[#2694](https://www.sqlalchemy.org/trac/ticket/2694)  ### “float”数据类型的强类型化

一系列更改允许使用`Float`数据类型更强烈地将自己与 Python 浮点值联系起来，而不是更通用的`Numeric`。这些更改主要与确保 Python 浮点值不会错误地被强制转换为`Decimal()`有关，并且在需要时被强制转换为`float`，如果应用程序正在处理普通浮点数。

+   传递给 SQL 表达式的普通 Python“float”值现在将被拉入具有类型`Float`的文字参数；以前，类型为`Numeric`，默认情况下“asdecimal=True”标志，这意味着结果类型将强制转换为`Decimal()`。特别是，这将在 SQLite 上发出令人困惑的警告：

    ```py
    float_value = connection.scalar(
        select([literal(4.56)])  # the "BindParameter" will now be
        # Float, not Numeric(asdecimal=True)
    )
    ```

+   在`Numeric`、`Float`和`Integer`之间的数学运算现在会保留结果表达式的类型，包括`asdecimal`标志以及类型是否应该是`Float`：

    ```py
    # asdecimal flag is maintained
    expr = column("a", Integer) * column("b", Numeric(asdecimal=False))
    assert expr.type.asdecimal == False

    # Float subclass of Numeric is maintained
    expr = column("a", Integer) * column("b", Float())
    assert isinstance(expr.type, Float)
    ```

+   如果 DBAPI 已知支持本机`Decimal()`模式，则`Float`数据类型将无条件地将`float()`处理器应用于结果值。一些后端不总是保证浮点数以纯浮点数而不是精确数值（如 MySQL）的形式返回。

[#4017](https://www.sqlalchemy.org/trac/ticket/4017)

[#4018](https://www.sqlalchemy.org/trac/ticket/4018)

[#4020](https://www.sqlalchemy.org/trac/ticket/4020)

### 支持 GROUPING SETS、CUBE、ROLLUP

所有的 GROUPING SETS、CUBE、ROLLUP 都可以通过`func`命名空间访问。在 CUBE 和 ROLLUP 的情况下，这些函数在之前的版本中已经可以使用，但是对于 GROUPING SETS，编译器中添加了一个占位符以便为其腾出空间。现在文档中已经命名了这三个函数：

```py
>>> from sqlalchemy import select, table, column, func, tuple_
>>> t = table("t", column("value"), column("x"), column("y"), column("z"), column("q"))
>>> stmt = select([func.sum(t.c.value)]).group_by(
...     func.grouping_sets(
...         tuple_(t.c.x, t.c.y),
...         tuple_(t.c.z, t.c.q),
...     )
... )
>>> print(stmt)
SELECT  sum(t.value)  AS  sum_1
FROM  t  GROUP  BY  GROUPING  SETS((t.x,  t.y),  (t.z,  t.q)) 
```

[#3429](https://www.sqlalchemy.org/trac/ticket/3429)

### 用于具有上下文默认生成器的多值插入的参数助手

默认生成函数，例如在上下文敏感默认函数中描述的函数，可以通过`DefaultExecutionContext.current_parameters` 属性查看与语句相关的当前参数。然而，在通过`Insert.values()` 方法指定多个 VALUES 子句的`Insert` 构造中，用户定义的函数会被多次调用，每个参数集一次，但是没有办法知道`DefaultExecutionContext.current_parameters` 中的哪些键子集适用于该列。添加了一个新函数`DefaultExecutionContext.get_current_parameters()`，其中包括一个关键字参数`DefaultExecutionContext.get_current_parameters.isolate_multiinsert_groups` 默认为`True`，它执行额外的工作，提供一个`DefaultExecutionContext.current_parameters` 的子字典，其中的名称被本地化为当前正在处理的 VALUES 子句：

```py
def mydefault(context):
    return context.get_current_parameters()["counter"] + 12

mytable = Table(
    "mytable",
    metadata_obj,
    Column("counter", Integer),
    Column("counter_plus_twelve", Integer, default=mydefault, onupdate=mydefault),
)

stmt = mytable.insert().values([{"counter": 5}, {"counter": 18}, {"counter": 20}])

conn.execute(stmt)
```

[#4075](https://www.sqlalchemy.org/trac/ticket/4075)

## 键行为更改 - ORM

### 在对象过期之前，`after_rollback()` 会话事件现在会发出

`SessionEvents.after_rollback()` 事件现在可以访问对象的属性状态，而不是在它们的状态被过期之前（例如，“快照删除”）。这使得该事件与`SessionEvents.after_commit()` 事件的行为保持一致，后者也会在“快照”被删除之前发出：

```py
sess = Session()

user = sess.query(User).filter_by(name="x").first()

@event.listens_for(sess, "after_rollback")
def after_rollback(session):
    # 'user.name' is now present, assuming it was already
    # loaded.  previously this would raise upon trying
    # to emit a lazy load.
    print("user name: %s" % user.name)

@event.listens_for(sess, "after_commit")
def after_commit(session):
    # 'user.name' is present, assuming it was already
    # loaded.  this is the existing behavior.
    print("user name: %s" % user.name)

if should_rollback:
    sess.rollback()
else:
    sess.commit()
```

请注意，`Session` 仍将禁止在此事件中发出 SQL；这意味着未加载的属性仍然无法在事件范围内加载。

[#3934](https://www.sqlalchemy.org/trac/ticket/3934)  ### 修复了与 `select_from()` 结合使用单表继承的问题

当生成 SQL 时，`Query.select_from()` 方法现在将遵循单表继承列鉴别器；以前，仅查询列列表中的表达式会被考虑进去。

假设 `Manager` 是 `Employee` 的子类。像以下这样的查询：

```py
sess.query(Manager.id)
```

将生成的 SQL 如下：

```py
SELECT  employee.id  FROM  employee  WHERE  employee.type  IN  ('manager')
```

但是，如果仅在列列表中指定了 `Manager`，而没有在 `Query.select_from()` 中指定，那么将不会添加鉴别器：

```py
sess.query(func.count(1)).select_from(Manager)
```

将生成如下：

```py
SELECT  count(1)  FROM  employee
```

通过此修复，`Query.select_from()` 现在可以正确工作，我们可以得到：

```py
SELECT  count(1)  FROM  employee  WHERE  employee.type  IN  ('manager')
```

可能已经通过手动提供 WHERE 子句来解决此问题的应用程序可能需要进行调整。

[#3891](https://www.sqlalchemy.org/trac/ticket/3891)  ### 替换集合时，先前的集合不再发生变化

当映射的集合成员发生更改时，ORM 会发出事件。在将集合分配给将替换先前集合的属性时，这样做的副作用是被替换的集合也将被改变，这是误导性和不必要的：

```py
>>> a1, a2, a3 = Address("a1"), Address("a2"), Address("a3")
>>> user.addresses = [a1, a2]

>>> previous_collection = user.addresses

# replace the collection with a new one
>>> user.addresses = [a2, a3]

>>> previous_collection
[Address('a1'), Address('a2')]
```

在上述更改之前，`previous_collection` 将已删除 “a1” 成员，对应于不再存在于新集合中的成员。

[#3913](https://www.sqlalchemy.org/trac/ticket/3913)  ### 在进行批量集合设置之前，@validates 方法接收所有值

在“批量设置”操作期间，使用 `@validates` 的方法现在将接收到集合的所有成员，然后再对现有集合进行比较。

给定映射如下：

```py
class A(Base):
    __tablename__ = "a"
    id = Column(Integer, primary_key=True)
    bs = relationship("B")

    @validates("bs")
    def convert_dict_to_b(self, key, value):
        return B(data=value["data"])

class B(Base):
    __tablename__ = "b"
    id = Column(Integer, primary_key=True)
    a_id = Column(ForeignKey("a.id"))
    data = Column(String)
```

在上述情况中，我们可以按照以下方式使用验证器，在集合附加时将传入的字典转换为 `B` 的实例：

```py
a1 = A()
a1.bs.append({"data": "b1"})
```

但是，集合赋值将失败，因为 ORM 将假定传入的对象已经是 `B` 的实例，因此在进行集合成员比较之前，它将尝试将它们与现有集合成员进行比较，然后执行实际调用验证器的集合附加操作。这将使得批量设置操作无法适应需要提前修改的非 ORM 对象，如需要提前修改的字典：

```py
a1 = A()
a1.bs = [{"data": "b1"}]
```

新逻辑使用新的 `AttributeEvents.bulk_replace()` 事件确保所有值在开始时发送到 `@validates` 函数。

作为此更改的一部分，这意味着验证器现在将在批量设置时接收**所有**集合成员，而不仅仅是新成员。假设一个简单的验证器如下：

```py
class A(Base):
    # ...

    @validates("bs")
    def validate_b(self, key, value):
        assert value.data is not None
        return value
```

在上述情况下，如果我们从一个集合开始：

```py
a1 = A()

b1, b2 = B(data="one"), B(data="two")
a1.bs = [b1, b2]
```

然后，用与第一个重叠的集合替换了该集合：

```py
b3 = B(data="three")
a1.bs = [b2, b3]
```

以前，第二个赋值将仅触发一次 `A.validate_b` 方法，对于 `b3` 对象。`b2` 对象将被视为已经存在于集合中并且不受验证。采用新行为后，`b2` 和 `b3` 都会在传递到集合之前传递给 `A.validate_b`。因此，验证方法必须采用幂等行为以适应这种情况。

另见

新的 bulk_replace 事件

[#3896](https://www.sqlalchemy.org/trac/ticket/3896)  ### 使用 flag_dirty() 将对象标记为“脏”，而不改变任何属性

如果使用 `flag_modified()` 函数标记一个实际未加载的属性为已修改，则现在会引发异常：

```py
a1 = A(data="adf")
s.add(a1)

s.flush()

# expire, similarly as though we said s.commit()
s.expire(a1, "data")

# will raise InvalidRequestError
attributes.flag_modified(a1, "data")
```

这是因为如果属性在冲刷发生时仍然未出现，则刷新过程很可能无论如何都会失败。要将对象标记为“修改”，而不具体引用任何属性，以便在自定义事件处理程序（如 `SessionEvents.before_flush()`）中考虑到刷新过程，请使用新的 `flag_dirty()` 函数：

```py
from sqlalchemy.orm import attributes

attributes.flag_dirty(a1)
```

[#3753](https://www.sqlalchemy.org/trac/ticket/3753)  ### 从 scoped_session 中删除“scope”关键字

一个非常古老且未记录的关键字参数 `scope` 已被删除：

```py
from sqlalchemy.orm import scoped_session

Session = scoped_session(sessionmaker())

session = Session(scope=None)
```

此关键字的目的是尝试允许可变“范围”，其中 `None` 表示“无范围”，因此将返回一个新的 `Session`。此关键字从未被文档化，并且现在如果遇到将会引发 `TypeError`。尽管不预期使用此关键字，但如果用户在测试期间报告与此相关的问题，则可以通过弃用来恢复。

[#3796](https://www.sqlalchemy.org/trac/ticket/3796)  ### 与 onupdate 结合使用的 post_update 的细化

使用 `relationship.post_update` 功能的关系现在将更好地与设置了 `Column.onupdate` 值的列进行交互。如果对象插入了列的显式值，则在更新期间重新声明它，以便“onupdate”规则不会覆盖它：

```py
class A(Base):
    __tablename__ = "a"
    id = Column(Integer, primary_key=True)
    favorite_b_id = Column(ForeignKey("b.id", name="favorite_b_fk"))
    bs = relationship("B", primaryjoin="A.id == B.a_id")
    favorite_b = relationship(
        "B", primaryjoin="A.favorite_b_id == B.id", post_update=True
    )
    updated = Column(Integer, onupdate=my_onupdate_function)

class B(Base):
    __tablename__ = "b"
    id = Column(Integer, primary_key=True)
    a_id = Column(ForeignKey("a.id", name="a_fk"))

a1 = A()
b1 = B()

a1.bs.append(b1)
a1.favorite_b = b1
a1.updated = 5
s.add(a1)
s.flush()
```

上面，以前的行为是在 INSERT 之后发出 UPDATE，从而触发“onupdate”并覆盖值“5”。现在的 SQL 看起来像这样：

```py
INSERT  INTO  a  (favorite_b_id,  updated)  VALUES  (?,  ?)
(None,  5)
INSERT  INTO  b  (a_id)  VALUES  (?)
(1,)
UPDATE  a  SET  favorite_b_id=?,  updated=?  WHERE  a.id  =  ?
(1,  5,  1)
```

此外，如果“updated”的值*未*设置，那么我们将会正确地在`a1.updated`上获取到新生成的值；以前，刷新或使属性过期以允许生成的值出现的逻辑不会对 post-update 触发。在这种情况下，当刷新 flush 内发生时，也会触发`InstanceEvents.refresh_flush()`事件。

[#3471](https://www.sqlalchemy.org/trac/ticket/3471)

[#3472](https://www.sqlalchemy.org/trac/ticket/3472)  ### post_update 与 ORM 版本控制集成

post_update 功能，文档中记录在指向自身的行 / 相互依赖的行，涉及到对特定与关系绑定的外键的更改而发出 UPDATE 语句，除了针对目标行通常会发出的 INSERT/UPDATE/DELETE。这个 UPDATE 语句现在参与版本控制功能，文档记录在配置版本计数器。

鉴于一个映射：

```py
class Node(Base):
    __tablename__ = "node"
    id = Column(Integer, primary_key=True)
    version_id = Column(Integer, default=0)
    parent_id = Column(ForeignKey("node.id"))
    favorite_node_id = Column(ForeignKey("node.id"))

    nodes = relationship("Node", primaryjoin=remote(parent_id) == id)
    favorite_node = relationship(
        "Node", primaryjoin=favorite_node_id == remote(id), post_update=True
    )

    __mapper_args__ = {"version_id_col": version_id}
```

更新将另一个节点关联为“favorite”的节点现在也将增加版本计数器，并匹配当前版本：

```py
node = Node()
session.add(node)
session.commit()  # node is now version #1

node = session.query(Node).get(node.id)
node.favorite_node = Node()
session.commit()  # node is now version #2
```

注意这意味着一个对象在响应其他属性变化而接收到 UPDATE，并且由于 post_update 关系变化而收到第二个 UPDATE，现在将会**为一个 flush 接收到两次版本计数更新**。然而，如果对象在当前 flush 内受到 INSERT，版本计数**将不会**额外增加一次，除非服务器端采用了版本控制方案。

现在讨论 post_update 即使对于 UPDATE 也会发出 UPDATE 的原因在为什么 post_update 除了第一个 UPDATE 之外还会发出 UPDATE？。

另请参阅

指向自身的行 / 相互依赖的行

为什么 post_update 除了第一个 UPDATE 之外还会发出 UPDATE？

[#3496](https://www.sqlalchemy.org/trac/ticket/3496)

## 关键行为更改 - 核心

### 自定义运算符的类型行为已经变得一致

可以使用`Operators.op()`函数即时制作用户定义的运算符。以前，针对这样的运算符的表达式的类型行为是不一致的，也是不可控的。

而在 1.1 中，以下表达式将产生没有返回类型的结果（假设`-%>`是数据库支持的某个特殊运算符）：

```py
>>> column("x", types.DateTime).op("-%>")(None).type
NullType()
```

其他类型将使用使用左侧类型作为返回类型的默认行为：

```py
>>> column("x", types.String(50)).op("-%>")(None).type
String(length=50)
```

这些行为大多是偶然发生的，因此行为已经与第二种形式保持一致，即默认返回类型与左侧表达式相同：

```py
>>> column("x", types.DateTime).op("-%>")(None).type
DateTime()
```

由于大多数用户定义的运算符往往是“比较”运算符，通常是由 PostgreSQL 定义的许多特殊运算符之一，`Operators.op.is_comparison` 标志已经修复，遵循其文档化行为，允许返回类型在所有情况下都是 `Boolean`，包括对于 `ARRAY` 和 `JSON`：

```py
>>> column("x", types.String(50)).op("-%>", is_comparison=True)(None).type
Boolean()
>>> column("x", types.ARRAY(types.Integer)).op("-%>", is_comparison=True)(None).type
Boolean()
>>> column("x", types.JSON()).op("-%>", is_comparison=True)(None).type
Boolean()
```

为了辅助布尔比较运算符，新增了一个新的简写方法 `Operators.bool_op()`。这个方法应该优先用于即时布尔运算符：

```py
>>> print(column("x", types.Integer).bool_op("-%>")(5))
x  -%>  :x_1 
```  ### literal_column() 中的百分号现在有条件地转义

`literal_column` 构造现在根据使用的 DBAPI 是否使用了百分号敏感的参数风格（例如‘format’或‘pyformat’）有条件地转义百分号字符。

以前，无法生成一个声明单个百分号的 `literal_column` 构造：

```py
>>> from sqlalchemy import literal_column
>>> print(literal_column("some%symbol"))
some%%symbol 
```

百分号现在不受未设置为使用‘format’或‘pyformat’参数风格的方言的影响；大多数 MySQL 方言等声明了其中一个参数风格的方言将继续适当地转义：

```py
>>> from sqlalchemy import literal_column
>>> print(literal_column("some%symbol"))
some%symbol
>>> from sqlalchemy.dialects import mysql
>>> print(literal_column("some%symbol").compile(dialect=mysql.dialect()))
some%%symbol 
```

作为这一变化的一部分，使用像 `ColumnOperators.contains()`、`ColumnOperators.startswith()` 和 `ColumnOperators.endswith()` 这样的运算符时，现在只在适当时才会发生加倍。

[#3740](https://www.sqlalchemy.org/trac/ticket/3740)  ### 列级别的 COLLATE 关键字现在引用排序规则名称

修复了在`collate()`和`ColumnOperators.collate()`函数中的一个错误，用于在语句级别提供临时列排序规则，其中区分大小写的名称不会被引用：

```py
stmt = select([mytable.c.x, mytable.c.y]).order_by(
    mytable.c.somecolumn.collate("fr_FR")
)
```

现在呈现为：

```py
SELECT  mytable.x,  mytable.y,
FROM  mytable  ORDER  BY  mytable.somecolumn  COLLATE  "fr_FR"
```

以前，区分大小写的名称“fr_FR”不会被引用。目前，手动引用“fr_FR”名称**不会**被检测到，因此手动引用标识符的应用程序应进行调整。请注意，此更改不影响在类型级别使用排序规则（例如在数据类型上指定的`String`在表级别），其中已经应用了引用。

[#3785](https://www.sqlalchemy.org/trac/ticket/3785)

## 方言改进和更改 - PostgreSQL

### 支持批处理模式 / 快速执行助手

已确定 psycopg2 的 `cursor.executemany()` 方法性能较差，特别是在 INSERT 语句中。为了缓解这一问题，psycopg2 添加了[快速执行助手](https://www.psycopg.org/docs/extras.html#fast-execution-helpers)，通过将多个 DML 语句批量发送，将语句重新组织为更少的服务器往返次数。SQLAlchemy 1.2 现在包括对这些助手的支持，以便在 `Engine` 使用 `cursor.executemany()` 对多个参数集调用语句时，可以透明地使用这些助手。该功能默认关闭，可以通过在 `create_engine()` 上使用 `use_batch_mode` 参数来启用：

```py
engine = create_engine(
    "postgresql+psycopg2://scott:tiger@host/dbname", use_batch_mode=True
)
```

目前该功能被视为��验性质，但可能在将来的版本中默认开启。

另请参阅

Psycopg2 快速执行助手

[#4109](https://www.sqlalchemy.org/trac/ticket/4109)  ### 支持 INTERVAL 中字段规范的指定，包括完整反射

PostgreSQL 的 INTERVAL 数据类型中的“fields”规范允许指定要存储的间隔的字段，包括诸如“YEAR”、“MONTH”、“YEAR TO MONTH”等值。 `INTERVAL` 数据类型现在允许指定这些值：

```py
from sqlalchemy.dialects.postgresql import INTERVAL

Table("my_table", metadata, Column("some_interval", INTERVAL(fields="DAY TO SECOND")))
```

此外，现在所有 INTERVAL 数据类型都可以独立于“fields”规范进行反射；数据类型本身中的“fields”参数也将存在：

```py
>>> inspect(engine).get_columns("my_table")
[{'comment': None,
 'name': u'some_interval', 'nullable': True,
 'default': None, 'autoincrement': False,
 'type': INTERVAL(fields=u'day to second')}]
```

[#3959](https://www.sqlalchemy.org/trac/ticket/3959)

## 方言改进和更改 - MySQL

### 支持 INSERT..ON DUPLICATE KEY UPDATE

MySQL 支持的 `INSERT` 的 `ON DUPLICATE KEY UPDATE` 子句现在可以使用 MySQL 特定版本的 `Insert` 对象来支持，通过 `sqlalchemy.dialects.mysql.dml.insert()`。这个 `Insert` 子类添加了一个新方法 `Insert.on_duplicate_key_update()`，实现了 MySQL 的语法：

```py
from sqlalchemy.dialects.mysql import insert

insert_stmt = insert(my_table).values(id="some_id", data="some data to insert")

on_conflict_stmt = insert_stmt.on_duplicate_key_update(
    data=insert_stmt.inserted.data, status="U"
)

conn.execute(on_conflict_stmt)
```

以上将呈现为：

```py
INSERT  INTO  my_table  (id,  data)
VALUES  (:id,  :data)
ON  DUPLICATE  KEY  UPDATE  data=VALUES(data),  status=:status_1
```

另请参阅

INSERT…ON DUPLICATE KEY UPDATE (Upsert)

[#4009](https://www.sqlalchemy.org/trac/ticket/4009)

## 方言改进和变更 - Oracle

### cx_Oracle 方言、类型系统的重大重构

随着 cx_Oracle DBAPI 的 6.x 系列的引入，SQLAlchemy 的 cx_Oracle 方言已经重新设计和简化，以利用 cx_Oracle 的最新改进，并放弃了在 cx_Oracle 的 5.x 系列之前更相关的模式支持。

+   支持的最低 cx_Oracle 版本现在是 5.1.3；推荐使用 5.3 或最新的 6.x 系列。

+   数据类型的处理已经重构。根据 cx_Oracle 的开发人员建议，`cursor.setinputsizes()` 方法不再用于除 LOB 类型之外的任何数据类型。因此，参数 `auto_setinputsizes` 和 `exclude_setinputsizes` 已被弃用，也不再起作用。

+   当将 `coerce_to_decimal` 标志设置为 False 以指示不应发生具有精度和标度的数值类型到 `Decimal` 的强制转换时，仅影响未经类型化的语句（例如，没有 `TypeEngine` 对象的普通字符串）。包含 `Numeric` 类型或子类型的 Core 表达式现在将遵循该类型的十进制强制转换规则。

+   “两阶段”事务支持在方言中已经在 cx_Oracle 的 6.x 系列中被删除，现在已完全移除，因为这个功能从未正确工作过，也不太可能被投入生产使用。因此，`allow_twophase` 方言标志已被弃用，也不再起作用。

+   修复了涉及带有 RETURNING 的列键的 bug。给定如下语句：

    ```py
    result = conn.execute(table.insert().values(x=5).returning(table.c.a, table.c.b))
    ```

    以前，结果中每行的键将是 `ret_0` 和 `ret_1`，这是 cx_Oracle RETURNING 实现内部的标识符。现在键将是 `a` 和 `b`，与其他方言的预期相符。

+   cx_Oracle 的 LOB 数据类型将返回值表示为 `cx_Oracle.LOB` 对象，这是一个与游标关联的代理，通过`.read()` 方法返回最终数据值。从历史上看，如果在消耗这些 LOB 对象之前读取了更多行（具体来说，读取了比 cursor.arraysize 值更多的行，这会导致读取新批次的行），这些 LOB 对象将引发错误“在后续获取后 LOB 变量不再有效”。SQLAlchemy 通过其类型系统自动调用这些 LOB 的`.read()`，以及使用特殊的 `BufferedColumnResultSet` 来解决这个问题，该结果集将确保在使用`cursor.fetchmany()` 或 `cursor.fetchall()` 这样的调用时，这些数据被缓冲。

    方言现在使用 cx_Oracle outputtypehandler 来处理这些`.read()` 调用，以便无论获取多少行，它们始终被提前调用，因此不再会发生此错误。因此，`BufferedColumnResultSet` 的使用，以及一些其他特定于此用例的 Core `ResultSet` 内部部分已被移除。由于类型对象不再需要处理二进制列结果，因此它们也变得更简化。

    此外，cx_Oracle 6.x 已删除了发生此错误的任何情况，因此不再可能发生错误。如果在使用极少（如果有的话）使用的 `auto_convert_lobs=False` 选项的情况下，与先前的 5.x 系列 cx_Oracle 结合使用，并且在 LOB 对象可以被消耗之前读取了更多行，则可能会在 SQLAlchemy 中发生此错误。升级到 cx_Oracle 6.x 将���决此问题。### Oracle Unique, Check 约束现在反映出来

UNIQUE 和 CHECK 约束现在通过`Inspector.get_unique_constraints()` 和 `Inspector.get_check_constraints()` 反映出来。被反映的`Table` 对象现在也将包括`CheckConstraint` 对象。有关此处行为怪癖的信息，请参阅约束反射，包括大多数`Table` 对象仍然不会包括任何`UniqueConstraint` 对象，因为这些通常通过`Index` 表示。

另请参见

约束反射

[#4003](https://www.sqlalchemy.org/trac/ticket/4003)  ### Oracle 外键约束名称现在是“名称标准化”

在表反射期间传递给 `ForeignKeyConstraint` 对象的外键约束名称以及在 `Inspector.get_foreign_keys()` 方法中，现在将被“名称标准化”，即，以小写形式表示以进行大小写不敏感的名称，而不是 Oracle 使用的原始大写格式：

```py
>>> insp.get_indexes("addresses")
[{'unique': False, 'column_names': [u'user_id'],
 'name': u'address_idx', 'dialect_options': {}}]

>>> insp.get_pk_constraint("addresses")
{'name': u'pk_cons', 'constrained_columns': [u'id']}

>>> insp.get_foreign_keys("addresses")
[{'referred_table': u'users', 'referred_columns': [u'id'],
 'referred_schema': None, 'name': u'user_id_fk',
 'constrained_columns': [u'user_id']}]
```

以前，外键结果看起来像：

```py
[
    {
        "referred_table": "users",
        "referred_columns": ["id"],
        "referred_schema": None,
        "name": "USER_ID_FK",
        "constrained_columns": ["user_id"],
    }
]
```

上述可能会特别与 Alembic autogenerate 创建问题。

[#3276](https://www.sqlalchemy.org/trac/ticket/3276)

## 方言改进和更改 - SQL Server

### 支持带有嵌入点的 SQL Server 模式名称

SQL Server 方言具有这样的行为，即假定具有其中一个点的模式名称是“数据库”。“所有者”标识符对，这在表和组件反射操作以及在呈现模式名称的引号时必须将这两个符号分开时会被分开。现在可以使用括号传递模式参数以手动指定此拆分发生的位置，从而允许数据库和/或所有者名称本身包含一个或多个点：

```py
Table("some_table", metadata, Column("q", String(50)), schema="[MyDataBase.dbo]")
```

上表将考虑“所有者”为 `MyDataBase.dbo`，在呈现时也将被引用，并且“数据库”为 None。要单独引用数据库名称和所有者，请使用两对括号：

```py
Table(
    "some_table",
    metadata,
    Column("q", String(50)),
    schema="[MyDataBase.SomeDB].[MyDB.owner]",
)
```

此外，当传递给 SQL Server 方言的“模式”时，现在将尊重 `quoted_name` 构造；如果引号标志为 True，则给定的符号不会在点上拆分，并且将被解释为“所有者”。

另请参阅

多部分模式名称

[#2626](https://www.sqlalchemy.org/trac/ticket/2626)

### AUTOCOMMIT 隔离级别支持

现在 PyODBC 和 pymssql 方言都支持由 `Connection.execution_options()` 设置的“AUTOCOMMIT”隔离级别，这将在 DBAPI 连接对象上建立正确的标志。

## 介绍

本指南介绍了 SQLAlchemy 版本 1.2 中的新功能，并记录了影响从 SQLAlchemy 1.1 系列迁移其应用程序的用户的更改。

请仔细查看行为变化部分，可能会对行为产生不兼容的变化。

## 平台支持

### 针对 Python 2.7 及更高版本

SQLAlchemy 1.2 现在将最低 Python 版本提升至 2.7，不再支持 2.6。预计将合并到 1.2 系列中的新语言特性在 Python 2.6 中不受支持。对于 Python 3 的支持，SQLAlchemy 目前在 3.5 和 3.6 版本上进行测试。

### 针对 Python 2.7 及更高版本

SQLAlchemy 1.2 现在将最低 Python 版本提升至 2.7，不再支持 2.6。预计将合并到 1.2 系列中的新语言特性在 Python 2.6 中不受支持。对于 Python 3 的支持，SQLAlchemy 目前在 3.5 和 3.6 版本上进行测试。

## 新功能和改进 - ORM

### “Baked” 加载现在是懒加载的默认选项

`sqlalchemy.ext.baked` 扩展首次引入于 1.0 系列，允许构建所谓的`BakedQuery`对象，该对象与表示查询结构的缓存键一起生成`Query`对象；然后将此缓存键链接到生成的字符串 SQL 语句，以便后续使用具有相同结构的另一个`BakedQuery`将绕过构建`Query`对象、构建其中的核心`select()`对象，以及将`select()`编译为字符串的所有开销，从而削减通常与构建和发出 ORM `Query`对象相关的大部分函数调用开销。

`BakedQuery` 现在在 ORM 默认情况下用于生成“延迟”查询，用于懒加载`relationship()`构造，例如默认的`lazy="select"`关系加载策略。这将显著减少应用程序在使用懒加载查询加载集合和相关对象时的函数调用。此功能以前在 1.0 和 1.1 中通过使用全局 API 方法或使用`baked_select`策略可用，现在是此行为的唯一实现。该功能还得到改进，使得对于具有懒加载后生效的其他加载器选项的对象仍然可以进行缓存。

可以使用 `relationship.bake_queries` 标志在每个关系基础上禁用缓存行为，这对于非常罕见的情况非常有用，例如使用不兼容缓存的自定义 `Query` 实现的关系。

[#3954](https://www.sqlalchemy.org/trac/ticket/3954)  ### 新的“selectin”急加载，一次性使用 IN 加载所有集合

添加了一个名为“selectin”加载的新急加载器，这在许多方面类似于“子查询”加载，但生成的 SQL 语句更简单，可缓存且更高效。

给定以下查询：

```py
q = (
    session.query(User)
    .filter(User.name.like("%ed%"))
    .options(subqueryload(User.addresses))
)
```

生成的 SQL 将是针对 `User` 的查询，然后是 `User.addresses` 的 subqueryload（请注意还列出了参数）：

```py
SELECT  users.id  AS  users_id,  users.name  AS  users_name
FROM  users
WHERE  users.name  LIKE  ?
('%ed%',)

SELECT  addresses.id  AS  addresses_id,
  addresses.user_id  AS  addresses_user_id,
  addresses.email_address  AS  addresses_email_address,
  anon_1.users_id  AS  anon_1_users_id
FROM  (SELECT  users.id  AS  users_id
FROM  users
WHERE  users.name  LIKE  ?)  AS  anon_1
JOIN  addresses  ON  anon_1.users_id  =  addresses.user_id
ORDER  BY  anon_1.users_id
('%ed%',)
```

使用“selectin”加载，我们实际上得到了一个引用父查询中加载的实际主键值的 SELECT：

```py
q = (
    session.query(User)
    .filter(User.name.like("%ed%"))
    .options(selectinload(User.addresses))
)
```

产生：

```py
SELECT  users.id  AS  users_id,  users.name  AS  users_name
FROM  users
WHERE  users.name  LIKE  ?
('%ed%',)

SELECT  users_1.id  AS  users_1_id,
  addresses.id  AS  addresses_id,
  addresses.user_id  AS  addresses_user_id,
  addresses.email_address  AS  addresses_email_address
FROM  users  AS  users_1
JOIN  addresses  ON  users_1.id  =  addresses.user_id
WHERE  users_1.id  IN  (?,  ?)
ORDER  BY  users_1.id
(1,  3)
```

上述 SELECT 语句包括以下优点：

+   它不使用子查询，只是一个 INNER JOIN，这意味着在像 MySQL 这样不喜欢子查询的数据库上性能会更好

+   其结构独立于原始查询；与新的 扩展 IN 参数系统 结合使用，我们在大多数情况下可以使用“烘焙”查询来缓存字符串 SQL，从而显著减少每个查询的开销。

+   因为查询仅获取给定主键标识符列表，“selectin”加载可能与 `Query.yield_per()` 兼容，以便一次处理 SELECT 结果的块，前提是数据库驱动程序允许多个同时游标（SQLite、PostgreSQL；**不**是 MySQL 驱动程序或 SQL Server ODBC 驱动程序）。联接急加载和子查询急加载都不兼容 `Query.yield_per()`。

选择急加载的缺点可能是潜在的大型 SQL 查询，带有大量的 IN 参数列表。 IN 参数列表本身被分组为 500 个一组，因此超过 500 个结果对象的结果集将有更多额外的“SELECT IN”查询。此外，对复合主键的支持取决于数据库是否能够使用带有 IN 的元组，例如 `(table.column_one, table_column_two) IN ((?, ?), (?, ?) (?, ?))`。目前，已知 PostgreSQL 和 MySQL 兼容此语法，而 SQLite 不兼容。

另请参见

Select IN 加载

[#3944](https://www.sqlalchemy.org/trac/ticket/3944)  ### “selectin” 多态加载，使用单独的 IN 查询加载子类

与刚刚在新“selectin” eager loading, loads all collections at once using IN 中描述的“selectin”关系加载功能类似的是“selectin”多态加载。这是一种专为连接式的急加载定制的多态加载功能，允许基本实体的加载通过简单的 SELECT 语句进行，但是额外的子类属性使用额外的 SELECT 语句加载：

```py
>>> from sqlalchemy.orm import selectin_polymorphic

>>> query = session.query(Employee).options(
...     selectin_polymorphic(Employee, [Manager, Engineer])
... )

>>> query.all()
SELECT
  employee.id  AS  employee_id,
  employee.name  AS  employee_name,
  employee.type  AS  employee_type
FROM  employee
()

SELECT
  engineer.id  AS  engineer_id,
  employee.id  AS  employee_id,
  employee.type  AS  employee_type,
  engineer.engineer_name  AS  engineer_engineer_name
FROM  employee  JOIN  engineer  ON  employee.id  =  engineer.id
WHERE  employee.id  IN  (?,  ?)  ORDER  BY  employee.id
(1,  2)

SELECT
  manager.id  AS  manager_id,
  employee.id  AS  employee_id,
  employee.type  AS  employee_type,
  manager.manager_name  AS  manager_manager_name
FROM  employee  JOIN  manager  ON  employee.id  =  manager.id
WHERE  employee.id  IN  (?)  ORDER  BY  employee.id
(3,) 
```

另请参阅

使用 selectin_polymorphic()

[#3948](https://www.sqlalchemy.org/trac/ticket/3948)  ### 可接收临时 SQL 表达式的 ORM 属性

新增了一个 ORM 属性类型`query_expression()`，与`deferred()`类似，但其 SQL 表达式在查询时确定，使用新选项`with_expression()`；如果未指定，属性默认为`None`：

```py
from sqlalchemy.orm import query_expression
from sqlalchemy.orm import with_expression

class A(Base):
    __tablename__ = "a"
    id = Column(Integer, primary_key=True)
    x = Column(Integer)
    y = Column(Integer)

    # will be None normally...
    expr = query_expression()

# but let's give it x + y
a1 = session.query(A).options(with_expression(A.expr, A.x + A.y)).first()
print(a1.expr)
```

另请参阅

查询时的 SQL 表达式作为映射属性

[#3058](https://www.sqlalchemy.org/trac/ticket/3058)  ### ORM 支持多表删除

ORM `Query.delete()` 方法支持多表条件的删除，如多表条件支持的删除中所介绍的。该功能的工作方式与更新的多表条件相同，最初在 0.8 版本中引入，并在 Query.update()支持 UPDATE..FROM 中描述。

下面，我们对`SomeEntity`发出一个 DELETE 请求，添加一个 FROM 子句（或者等效的，根据后端不同）对`SomeOtherEntity`进行操作：

```py
query(SomeEntity).filter(SomeEntity.id == SomeOtherEntity.id).filter(
    SomeOtherEntity.foo == "bar"
).delete()
```

另请参阅

多表条件支持的删除

[#959](https://www.sqlalchemy.org/trac/ticket/959)  ### 支持混合、复合的批量更新

现在，混合属性（例如`sqlalchemy.ext.hybrid`）以及复合属性（复合列类型）在使用`Query.update()`更新语句的 SET 子句中均得到支持。

对于混合属性，可以直接使用简单的表达式，或者可以使用新的装饰器`hybrid_property.update_expression()`将值拆分为多个列/表达式：

```py
class Person(Base):
    # ...

    first_name = Column(String(10))
    last_name = Column(String(10))

    @hybrid.hybrid_property
    def name(self):
        return self.first_name + " " + self.last_name

    @name.expression
    def name(cls):
        return func.concat(cls.first_name, " ", cls.last_name)

    @name.update_expression
    def name(cls, value):
        f, l = value.split(" ", 1)
        return [(cls.first_name, f), (cls.last_name, l)]
```

如上所述，可以使用以下方式渲染 UPDATE：

```py
session.query(Person).filter(Person.id == 5).update({Person.name: "Dr. No"})
```

类似的功能也适用于复合属性，其中复合值将被拆分为其各个列以进行批量更新：

```py
session.query(Vertex).update({Edge.start: Point(3, 4)})
```

另请参阅

允许批量 ORM 更新  ### 混合属性支持在子类之间重用，重新定义@getter

`sqlalchemy.ext.hybrid.hybrid_property`类现在支持在子类之间多次调用像`@setter`、`@expression`等的变异器，并且现在提供了一个`@getter`变异器，以便特定的混合属性可以在子类或其他类之间重新使用。这与标准 Python 中`@property`的行为类似：

```py
class FirstNameOnly(Base):
    # ...

    first_name = Column(String)

    @hybrid_property
    def name(self):
        return self.first_name

    @name.setter
    def name(self, value):
        self.first_name = value

class FirstNameLastName(FirstNameOnly):
    # ...

    last_name = Column(String)

    @FirstNameOnly.name.getter
    def name(self):
        return self.first_name + " " + self.last_name

    @name.setter
    def name(self, value):
        self.first_name, self.last_name = value.split(" ", maxsplit=1)

    @name.expression
    def name(cls):
        return func.concat(cls.first_name, " ", cls.last_name)
```

在上面的示例中，`FirstNameOnly.name`混合属性被`FirstNameLastName`子类引用，以便将其专门用于新子类。这是通过在每次调用`@getter`、`@setter`以及所有其他变异器方法像`@expression`中复制混合对象到新对象来实现的，从而保持先前混合属性的定义不变。以前，像`@setter`这样的方法会直接修改现有的混合属性，干扰了超类上的定义。

注意

请务必阅读在子类之间重用混合属性的文档，了解如何覆盖`hybrid_property.expression()`和`hybrid_property.comparator()`的重要注意事项，因为在某些情况下，可能需要使用特殊限定符`hybrid_property.overrides`来避免与`QueryableAttribute`发生名称冲突。

注意

`@hybrid_property`中的这种变化意味着，当向`@hybrid_property`添加 setter 和其他状态时，**方法必须保留原始混合属性的名称**，否则具有附加状态的新混合属性将以不匹配的名称存在于类中。这与标准 Python 中的`@property`构造的行为相同：

```py
class FirstNameOnly(Base):
    @hybrid_property
    def name(self):
        return self.first_name

    # WRONG - will raise AttributeError: can't set attribute when
    # assigning to .name
    @name.setter
    def _set_name(self, value):
        self.first_name = value

class FirstNameOnly(Base):
    @hybrid_property
    def name(self):
        return self.first_name

    # CORRECT - note regular Python @property works the same way
    @name.setter
    def name(self, value):
        self.first_name = value
```

[#3911](https://www.sqlalchemy.org/trac/ticket/3911)

[#3912](https://www.sqlalchemy.org/trac/ticket/3912)  ### 新的 bulk_replace 事件

为了适应 A @validates method receives all values on bulk-collection set before comparison 中描述的验证用例，添加了一个新的`AttributeEvents.bulk_replace()`方法，该方法与`AttributeEvents.append()`和`AttributeEvents.remove()`事件一起调用。在比较现有集合之前调用“bulk_replace”，以便可以修改集合。之后，单个项目将附加到新的目标集合，触发为集合中的新项目触发的“append”事件，这与以前的行为相同。下面同时说明了“bulk_replace”和“append”，包括如果使用集合赋值，“append”将接收到已由“bulk_replace”处理的对象。新符号`attributes.OP_BULK_REPLACE`可以用于确定此“append”事件是否是批量替换的第二部分：

```py
from sqlalchemy.orm.attributes import OP_BULK_REPLACE

@event.listens_for(SomeObject.collection, "bulk_replace")
def process_collection(target, values, initiator):
    values[:] = [_make_value(value) for value in values]

@event.listens_for(SomeObject.collection, "append", retval=True)
def process_collection(target, value, initiator):
    # make sure bulk_replace didn't already do it
    if initiator is None or initiator.op is not OP_BULK_REPLACE:
        return _make_value(value)
    else:
        return value
```

[#3896](https://www.sqlalchemy.org/trac/ticket/3896)  ### 为 sqlalchemy.ext.mutable 添加了新的“modified”事件处理程序

添加了一个新的事件处理程序`AttributeEvents.modified()`，该处理程序在对`flag_modified()`方法的调用时触发，通常是从`sqlalchemy.ext.mutable`扩展调用的。

```py
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.ext.mutable import MutableDict
from sqlalchemy import event

Base = declarative_base()

class MyDataClass(Base):
    __tablename__ = "my_data"
    id = Column(Integer, primary_key=True)
    data = Column(MutableDict.as_mutable(JSONEncodedDict))

@event.listens_for(MyDataClass.data, "modified")
def modified_json(instance):
    print("json value modified:", instance.data)
```

上述事件处理程序将在对`.data`字典进行就地更改时触发。

[#3303](https://www.sqlalchemy.org/trac/ticket/3303)  ### 添加了对 Session.refresh 的“for update”参数

添加了新参数`Session.refresh.with_for_update`到`Session.refresh()`方法。当`Query.with_lockmode()`方法被弃用，而倾向于`Query.with_for_update()`时，`Session.refresh()`方法从未更新以反映新选项：

```py
session.refresh(some_object, with_for_update=True)
```

`Session.refresh.with_for_update`参数现在接受一个选项字典，该字典将作为发送给`Query.with_for_update()`的相同参数：

```py
session.refresh(some_objects, with_for_update={"read": True})
```

新参数取代了`Session.refresh.lockmode`参数。

[#3991](https://www.sqlalchemy.org/trac/ticket/3991)  ### 就地变异操作符适用于 MutableSet、MutableList

对于`MutableSet`，我们实现了就地变异操作符`__ior__`、`__iand__`、`__ixor__`和`__isub__`，以及对于`MutableList`的`__iadd__`。虽然这些方法以前可以成功地更新集合，但它们不会正确地触发更改事件。这些操作符像以前一样改变集合，但额外地发出了正确的更改事件，以便更改成为下一个刷新进程的一部分：

```py
model = session.query(MyModel).first()
model.json_set &= {1, 3}
```

[#3853](https://www.sqlalchemy.org/trac/ticket/3853)  ### AssociationProxy any()、has()、contains()可以与链式关联代理一起使用

`AssociationProxy.any()`、`AssociationProxy.has()`和`AssociationProxy.contains()`比较方法现在支持链接到一个属性，该属性本身也是一个`AssociationProxy`，递归地。在下面的示例中，`A.b_values`是一个关联到`AtoB.bvalue`的关联代理，它本身是一个关联到`B`的关联代理：

```py
class A(Base):
    __tablename__ = "a"
    id = Column(Integer, primary_key=True)

    b_values = association_proxy("atob", "b_value")
    c_values = association_proxy("atob", "c_value")

class B(Base):
    __tablename__ = "b"
    id = Column(Integer, primary_key=True)
    a_id = Column(ForeignKey("a.id"))
    value = Column(String)

    c = relationship("C")

class C(Base):
    __tablename__ = "c"
    id = Column(Integer, primary_key=True)
    b_id = Column(ForeignKey("b.id"))
    value = Column(String)

class AtoB(Base):
    __tablename__ = "atob"

    a_id = Column(ForeignKey("a.id"), primary_key=True)
    b_id = Column(ForeignKey("b.id"), primary_key=True)

    a = relationship("A", backref="atob")
    b = relationship("B", backref="atob")

    b_value = association_proxy("b", "value")
    c_value = association_proxy("b", "c")
```

我们可以使用`AssociationProxy.contains()`在`A.b_values`上进行查询，以跨越两个代理`A.b_values`，`AtoB.b_value`：

```py
>>> s.query(A).filter(A.b_values.contains("hi")).all()
SELECT  a.id  AS  a_id
FROM  a
WHERE  EXISTS  (SELECT  1
FROM  atob
WHERE  a.id  =  atob.a_id  AND  (EXISTS  (SELECT  1
FROM  b
WHERE  b.id  =  atob.b_id  AND  b.value  =  :value_1))) 
```

类似地，我们可以使用`AssociationProxy.any()`在`A.c_values`上进行查询，以跨越两个代理`A.c_values`，`AtoB.c_value`：

```py
>>> s.query(A).filter(A.c_values.any(value="x")).all()
SELECT  a.id  AS  a_id
FROM  a
WHERE  EXISTS  (SELECT  1
FROM  atob
WHERE  a.id  =  atob.a_id  AND  (EXISTS  (SELECT  1
FROM  b
WHERE  b.id  =  atob.b_id  AND  (EXISTS  (SELECT  1
FROM  c
WHERE  b.id  =  c.b_id  AND  c.value  =  :value_1))))) 
```

[#3769](https://www.sqlalchemy.org/trac/ticket/3769)  ### 标识键增强以支持分片

现在，ORM 使用的标识键结构包含了一个额外的成员，以便来自不同上下文的两个相同的主键可以共存于同一个标识映射中。

水平分片 中的示例已更新以说明此行为。该示例显示了一个分片类 `WeatherLocation`，引用一个依赖的 `WeatherReport` 对象，其中 `WeatherReport` 类映射到一个存储简单整数主键的表。来自不同数据库的两个 `WeatherReport` 对象可能具有相同的主键值。该示例现在说明了一个新的 `identity_token` 字段跟踪此差异，以便这两个对象可以共存于同一标识映射中：

```py
tokyo = WeatherLocation("Asia", "Tokyo")
newyork = WeatherLocation("North America", "New York")

tokyo.reports.append(Report(80.0))
newyork.reports.append(Report(75))

sess = create_session()

sess.add_all([tokyo, newyork, quito])

sess.commit()

# the Report class uses a simple integer primary key.  So across two
# databases, a primary key will be repeated.  The "identity_token" tracks
# in memory that these two identical primary keys are local to different
# databases.

newyork_report = newyork.reports[0]
tokyo_report = tokyo.reports[0]

assert inspect(newyork_report).identity_key == (Report, (1,), "north_america")
assert inspect(tokyo_report).identity_key == (Report, (1,), "asia")

# the token representing the originating shard is also available directly

assert inspect(newyork_report).identity_token == "north_america"
assert inspect(tokyo_report).identity_token == "asia"
```

[#4137](https://www.sqlalchemy.org/trac/ticket/4137)  ### “烘焙”加载现在是延迟加载的默认设置

`sqlalchemy.ext.baked` 扩展首次在 1.0 系列中引入，允许构建所谓的 `BakedQuery` 对象，该对象生成一个与表示查询结构的缓存键相关联的 `Query` 对象；然后将此缓存键链接到生成的字符串 SQL 语句，以便后续使用具有相同结构的另一个 `BakedQuery` 将绕过构建 `Query` 对象的所有开销，构建其中的核心 `select()` 对象，以及将 `select()` 编译为字符串，从而削减通常与构建和发出 ORM `Query` 对象相关的大部分函数调用开销。

当 ORM 生成“延迟”查询以懒加载 `relationship()` 构造时，默认现在使用 `BakedQuery`，例如默认的 `lazy="select"` 关系加载器策略。这将显著减少应用程序使用延迟加载查询加载集合和相关对象时的函数调用数量。以前，此功能在 1.0 和 1.1 中通过使用全局 API 方法或使用 `baked_select` 策略可用，现在是此行为的唯一实现。该功能还得到改进，使得对于具有延迟加载后生效的其他加载器选项的对象仍然可以进行缓存。

可以使用 `relationship.bake_queries` 标志在每个关系基础上禁用缓存行为，这对于非常不寻常的情况非常有用，例如使用不兼容缓存的自定义 `Query` 实现的关系。

[#3954](https://www.sqlalchemy.org/trac/ticket/3954)

### 新的 “selectin” 急切加载，一次加载所有集合使用 IN

添加了一个名为 “selectin” 加载的新急切加载器，从许多方面来看，它类似于 “subquery” 加载，但是生成了一个更简单的可缓存的 SQL 语句，而且更有效率。

给定如下查询：

```py
q = (
    session.query(User)
    .filter(User.name.like("%ed%"))
    .options(subqueryload(User.addresses))
)
```

生成的 SQL 将是针对 `User` 的查询，然后是 `User.addresses` 的 subqueryload（注意还列出了参数）：

```py
SELECT  users.id  AS  users_id,  users.name  AS  users_name
FROM  users
WHERE  users.name  LIKE  ?
('%ed%',)

SELECT  addresses.id  AS  addresses_id,
  addresses.user_id  AS  addresses_user_id,
  addresses.email_address  AS  addresses_email_address,
  anon_1.users_id  AS  anon_1_users_id
FROM  (SELECT  users.id  AS  users_id
FROM  users
WHERE  users.name  LIKE  ?)  AS  anon_1
JOIN  addresses  ON  anon_1.users_id  =  addresses.user_id
ORDER  BY  anon_1.users_id
('%ed%',)
```

使用 “selectin” 加载，我们得到���是一个 SELECT，它引用了在父查询中加载的实际主键值：

```py
q = (
    session.query(User)
    .filter(User.name.like("%ed%"))
    .options(selectinload(User.addresses))
)
```

产生：

```py
SELECT  users.id  AS  users_id,  users.name  AS  users_name
FROM  users
WHERE  users.name  LIKE  ?
('%ed%',)

SELECT  users_1.id  AS  users_1_id,
  addresses.id  AS  addresses_id,
  addresses.user_id  AS  addresses_user_id,
  addresses.email_address  AS  addresses_email_address
FROM  users  AS  users_1
JOIN  addresses  ON  users_1.id  =  addresses.user_id
WHERE  users_1.id  IN  (?,  ?)
ORDER  BY  users_1.id
(1,  3)
```

上述 SELECT 语句包括以下优点：

+   它不使用子查询，只是一个 INNER JOIN，这意味着在像 MySQL 这样不喜欢子查询的数据库上性能会更好

+   其结构独立于原始查询；与新的 扩展 IN 参数系统 结合，我们在大多数情况下可以使用 “baked” 查询来缓存字符串 SQL，显著减少每个查询的开销

+   由于查询仅为给定的主键标识符列表获取数据，“selectin” 加载可能与 `Query.yield_per()` 兼容，以便一次操作 SELECT 结果的一部分，前提是数据库驱动程序允许多个同时游标（SQLite，PostgreSQL；**不**是 MySQL 驱动程序或 SQL Server ODBC 驱动程序）。联接式急切加载和子查询急切加载都不兼容 `Query.yield_per()`。

selectin 急切加载的缺点是潜在的大型 SQL 查询，具有大量的 IN 参数列表。 IN 参数列表本身被分组为 500 个一组，因此超过 500 个 lead 对象的结果集将有更多的附加 “SELECT IN” 查询。此外，对复合主键的支持取决于数据库是否能够使用带有 IN 的元组，例如 `(table.column_one, table_column_two) IN ((?, ?), (?, ?) (?, ?))`。目前，已知 PostgreSQL 和 MySQL 兼容此语法，SQLite 不兼容。

另请参见

选择 IN 加载

[#3944](https://www.sqlalchemy.org/trac/ticket/3944)

### “selectin” 多态加载，使用单独的 IN 查询加载子类

与刚刚在新的“selectin”急加载，使用 IN 一次加载所有集合中描述的“selectin”关系加载功能类似的是“selectin”多态加载。这是一个主要针对连接式急加载的多态加载功能，允许基本实体的加载通过简单的 SELECT 语句进行，然后额外子类的属性通过额外的 SELECT 语句进行加载：

```py
>>> from sqlalchemy.orm import selectin_polymorphic

>>> query = session.query(Employee).options(
...     selectin_polymorphic(Employee, [Manager, Engineer])
... )

>>> query.all()
SELECT
  employee.id  AS  employee_id,
  employee.name  AS  employee_name,
  employee.type  AS  employee_type
FROM  employee
()

SELECT
  engineer.id  AS  engineer_id,
  employee.id  AS  employee_id,
  employee.type  AS  employee_type,
  engineer.engineer_name  AS  engineer_engineer_name
FROM  employee  JOIN  engineer  ON  employee.id  =  engineer.id
WHERE  employee.id  IN  (?,  ?)  ORDER  BY  employee.id
(1,  2)

SELECT
  manager.id  AS  manager_id,
  employee.id  AS  employee_id,
  employee.type  AS  employee_type,
  manager.manager_name  AS  manager_manager_name
FROM  employee  JOIN  manager  ON  employee.id  =  manager.id
WHERE  employee.id  IN  (?)  ORDER  BY  employee.id
(3,) 
```

另请参阅

使用 selectin_polymorphic()

[#3948](https://www.sqlalchemy.org/trac/ticket/3948)

### 可接收临时 SQL 表达式的 ORM 属性

新的 ORM 属性类型`query_expression()`被添加，类似于`deferred()`，不同之处在于其 SQL 表达式在查询时使用新选项`with_expression()`确定；如果未指定，则属性默认为`None`：

```py
from sqlalchemy.orm import query_expression
from sqlalchemy.orm import with_expression

class A(Base):
    __tablename__ = "a"
    id = Column(Integer, primary_key=True)
    x = Column(Integer)
    y = Column(Integer)

    # will be None normally...
    expr = query_expression()

# but let's give it x + y
a1 = session.query(A).options(with_expression(A.expr, A.x + A.y)).first()
print(a1.expr)
```

另请参阅

查询时 SQL 表达式作为映射属性

[#3058](https://www.sqlalchemy.org/trac/ticket/3058)

### ORM 支持多表删除

ORM `Query.delete()` 方法支持多表条件的 DELETE，就像在支持多表条件的 DELETE 中介绍的那样。该功能与 0.8 中首次引入的 UPDATE 的多表条件相同，详细描述在 Query.update()支持 UPDATE..FROM 中。

下面，我们对`SomeEntity`执行一个 DELETE 操作，添加一个 FROM 子句（或等效的，取决于后端）对`SomeOtherEntity`：

```py
query(SomeEntity).filter(SomeEntity.id == SomeOtherEntity.id).filter(
    SomeOtherEntity.foo == "bar"
).delete()
```

另请参阅

支持多表条件的 DELETE

[#959](https://www.sqlalchemy.org/trac/ticket/959)

### 支持混合属性，复合属性的批量更新

混合属性（例如`sqlalchemy.ext.hybrid`）以及复合属性（复合列类型）现在都支持在使用`Query.update()`时用于 UPDATE 语句的 SET 子句中。

对于混合属性，可以直接使用简单表达式，或者可以使用新的装饰器`hybrid_property.update_expression()`将一个值分解为多个列/表达式：

```py
class Person(Base):
    # ...

    first_name = Column(String(10))
    last_name = Column(String(10))

    @hybrid.hybrid_property
    def name(self):
        return self.first_name + " " + self.last_name

    @name.expression
    def name(cls):
        return func.concat(cls.first_name, " ", cls.last_name)

    @name.update_expression
    def name(cls, value):
        f, l = value.split(" ", 1)
        return [(cls.first_name, f), (cls.last_name, l)]
```

上面，一个 UPDATE 可以使用以下方式呈现：

```py
session.query(Person).filter(Person.id == 5).update({Person.name: "Dr. No"})
```

类似的功能也适用于复合类型，其中复合值将被拆分为其各个列以进行批量更新：

```py
session.query(Vertex).update({Edge.start: Point(3, 4)})
```

另请参阅

允许批量 ORM 更新

### 混合属性支持在子类之间重用，重新定义 @getter

`sqlalchemy.ext.hybrid.hybrid_property` 类现在支持在子类之间多次调用修改器，如 `@setter`、`@expression` 等，并且现在提供了一个 `@getter` 修改器，以便可以在子类或其他类之间重新用特定的混合属性。这与标准 Python 中 `@property` 的行为类似：

```py
class FirstNameOnly(Base):
    # ...

    first_name = Column(String)

    @hybrid_property
    def name(self):
        return self.first_name

    @name.setter
    def name(self, value):
        self.first_name = value

class FirstNameLastName(FirstNameOnly):
    # ...

    last_name = Column(String)

    @FirstNameOnly.name.getter
    def name(self):
        return self.first_name + " " + self.last_name

    @name.setter
    def name(self, value):
        self.first_name, self.last_name = value.split(" ", maxsplit=1)

    @name.expression
    def name(cls):
        return func.concat(cls.first_name, " ", cls.last_name)
```

在上面的例子中，`FirstNameOnly.name` 混合属性被 `FirstNameLastName` 子类引用，以便将其专门重新用于新的子类。这是通过在每次调用 `@getter`、`@setter` 以及所有其他修改器方法（如 `@expression`）中将混合对象复制到一个新对象中来实现的，从而保持先前混合属性的定义不变。以前，像 `@setter` 这样的方法会就地修改现有的混合属性，干扰了超类上的定义。

注意

请务必阅读在子类之间重用混合属性的文档，了解如何覆盖`hybrid_property.expression()` 和 `hybrid_property.comparator()` 的重要注意事项，因为在某些情况下可能需要一个特殊的限定符 `hybrid_property.overrides` 来避免与 `QueryableAttribute` 的名称冲突。

注意

这种对 `@hybrid_property` 的更改意味着，当向 `@hybrid_property` 添加 setter 和其他状态时，**方法必须保留原始混合属性的名称**，否则新的带有额外状态的混合属性将以不匹配的名称存在于类中。这与标准 Python 中 `@property` 的行为相同：

```py
class FirstNameOnly(Base):
    @hybrid_property
    def name(self):
        return self.first_name

    # WRONG - will raise AttributeError: can't set attribute when
    # assigning to .name
    @name.setter
    def _set_name(self, value):
        self.first_name = value

class FirstNameOnly(Base):
    @hybrid_property
    def name(self):
        return self.first_name

    # CORRECT - note regular Python @property works the same way
    @name.setter
    def name(self, value):
        self.first_name = value
```

[#3911](https://www.sqlalchemy.org/trac/ticket/3911)

[#3912](https://www.sqlalchemy.org/trac/ticket/3912)

### 新的 bulk_replace 事件

为了适应 在批量集合设置之前比较时，@validates 方法接收所有值 中描述的验证用例，添加了一个新的 `AttributeEvents.bulk_replace()` 方法，它与 `AttributeEvents.append()` 和 `AttributeEvents.remove()` 事件一起调用。“bulk_replace” 在 “append” 和 “remove” 之前调用，以便在比较现有集合之前修改集合。之后，单个项目将被附加到新的目标集合，触发针对集合中新项目的 “append” 事件，就像以前的行为一样。下面同时说明了 “bulk_replace” 和 “append”，包括如果使用集合赋值，“append” 将接收到已由 “bulk_replace” 处理过的对象的情况。新的符号 `attributes.OP_BULK_REPLACE` 可以用于确定此 “append” 事件是否是批量替换的第二部分：

```py
from sqlalchemy.orm.attributes import OP_BULK_REPLACE

@event.listens_for(SomeObject.collection, "bulk_replace")
def process_collection(target, values, initiator):
    values[:] = [_make_value(value) for value in values]

@event.listens_for(SomeObject.collection, "append", retval=True)
def process_collection(target, value, initiator):
    # make sure bulk_replace didn't already do it
    if initiator is None or initiator.op is not OP_BULK_REPLACE:
        return _make_value(value)
    else:
        return value
```

[#3896](https://www.sqlalchemy.org/trac/ticket/3896)

### 新的 sqlalchemy.ext.mutable 的“modified”事件处理器

新的事件处理器 `AttributeEvents.modified()` 被添加了，它会在调用 `flag_modified()` 方法时触发，通常这个方法是从 `sqlalchemy.ext.mutable` 扩展中调用的：

```py
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.ext.mutable import MutableDict
from sqlalchemy import event

Base = declarative_base()

class MyDataClass(Base):
    __tablename__ = "my_data"
    id = Column(Integer, primary_key=True)
    data = Column(MutableDict.as_mutable(JSONEncodedDict))

@event.listens_for(MyDataClass.data, "modified")
def modified_json(instance):
    print("json value modified:", instance.data)
```

上面的事件处理程序将在对 `.data` 字典进行原位更改时触发。

[#3303](https://www.sqlalchemy.org/trac/ticket/3303)

### 添加了“for update”参数到 Session.refresh

向 `Session.refresh()` 方法添加了新参数 `Session.refresh.with_for_update`。当 `Query.with_lockmode()` 方法被弃用，改用 `Query.with_for_update()` 后，`Session.refresh()` 方法从未更新以反映新选项：

```py
session.refresh(some_object, with_for_update=True)
```

`Session.refresh.with_for_update` 参数接受一个选项字典，这些选项将作为传递给 `Query.with_for_update()` 的相同参数：

```py
session.refresh(some_objects, with_for_update={"read": True})
```

新参数取代了 `Session.refresh.lockmode` 参数。

[#3991](https://www.sqlalchemy.org/trac/ticket/3991)

### 可变集合 `MutableSet` 和 可变列表 `MutableList` 支持原地变异操作符

为 `MutableSet` 实现了原地变异操作符 `__ior__`、`__iand__`、`__ixor__` 和 `__isub__`，以及为 `MutableList` 实现了 `__iadd__`。虽然这些方法以前可以成功更新集合，但它们不会正确触发更改事件。这些操作符像以前一样改变集合，但另外会发出正确的更改事件，以便更改成为下一个刷新过程的一部分：

```py
model = session.query(MyModel).first()
model.json_set &= {1, 3}
```

[#3853](https://www.sqlalchemy.org/trac/ticket/3853)

### `AssociationProxy` 的 `any()`、`has()` 和 `contains()` 方法可以与链式关联代理一起使用

`AssociationProxy.any()`、`AssociationProxy.has()` 和 `AssociationProxy.contains()` 比较方法现在支持链接到一个属性，该属性本身也是一个 `AssociationProxy`，递归地。下面，`A.b_values` 是一个关联代理，链接到 `AtoB.bvalue`，而 `AtoB.bvalue` 本身是一个关联代理，链接到 `B`：

```py
class A(Base):
    __tablename__ = "a"
    id = Column(Integer, primary_key=True)

    b_values = association_proxy("atob", "b_value")
    c_values = association_proxy("atob", "c_value")

class B(Base):
    __tablename__ = "b"
    id = Column(Integer, primary_key=True)
    a_id = Column(ForeignKey("a.id"))
    value = Column(String)

    c = relationship("C")

class C(Base):
    __tablename__ = "c"
    id = Column(Integer, primary_key=True)
    b_id = Column(ForeignKey("b.id"))
    value = Column(String)

class AtoB(Base):
    __tablename__ = "atob"

    a_id = Column(ForeignKey("a.id"), primary_key=True)
    b_id = Column(ForeignKey("b.id"), primary_key=True)

    a = relationship("A", backref="atob")
    b = relationship("B", backref="atob")

    b_value = association_proxy("b", "value")
    c_value = association_proxy("b", "c")
```

我们可以使用 `AssociationProxy.contains()` 在 `A.b_values` 上进行查询，以跨越两个代理 `A.b_values`、`AtoB.b_value`：

```py
>>> s.query(A).filter(A.b_values.contains("hi")).all()
SELECT  a.id  AS  a_id
FROM  a
WHERE  EXISTS  (SELECT  1
FROM  atob
WHERE  a.id  =  atob.a_id  AND  (EXISTS  (SELECT  1
FROM  b
WHERE  b.id  =  atob.b_id  AND  b.value  =  :value_1))) 
```

类似地，我们可以使用 `AssociationProxy.any()` 在 `A.c_values` 上进行查询，以跨越两个代理 `A.c_values`、`AtoB.c_value`：

```py
>>> s.query(A).filter(A.c_values.any(value="x")).all()
SELECT  a.id  AS  a_id
FROM  a
WHERE  EXISTS  (SELECT  1
FROM  atob
WHERE  a.id  =  atob.a_id  AND  (EXISTS  (SELECT  1
FROM  b
WHERE  b.id  =  atob.b_id  AND  (EXISTS  (SELECT  1
FROM  c
WHERE  b.id  =  c.b_id  AND  c.value  =  :value_1))))) 
```

[#3769](https://www.sqlalchemy.org/trac/ticket/3769)

### 身份键增强以支持分片

ORM 现在使用的身份键结构包含一个额外成员，因此来自不同上下文的两个相同主键可以共存于同一身份映射中。

水平分片的示例已更新以说明这种行为。示例展示了一个分片类`WeatherLocation`，它引用一个依赖的`WeatherReport`对象，其中`WeatherReport`类映射到一个存储简单整数主键的表。来自不同数据库的两个`WeatherReport`对象可能具有相同的主键值。现在的示例说明了一个新的`identity_token`字段跟踪这种差异，以便这两个对象可以共存于同一个标识映射中：

```py
tokyo = WeatherLocation("Asia", "Tokyo")
newyork = WeatherLocation("North America", "New York")

tokyo.reports.append(Report(80.0))
newyork.reports.append(Report(75))

sess = create_session()

sess.add_all([tokyo, newyork, quito])

sess.commit()

# the Report class uses a simple integer primary key.  So across two
# databases, a primary key will be repeated.  The "identity_token" tracks
# in memory that these two identical primary keys are local to different
# databases.

newyork_report = newyork.reports[0]
tokyo_report = tokyo.reports[0]

assert inspect(newyork_report).identity_key == (Report, (1,), "north_america")
assert inspect(tokyo_report).identity_key == (Report, (1,), "asia")

# the token representing the originating shard is also available directly

assert inspect(newyork_report).identity_token == "north_america"
assert inspect(tokyo_report).identity_token == "asia"
```

[#4137](https://www.sqlalchemy.org/trac/ticket/4137)

## 新功能和改进 - 核心

### 布尔数据类型现在强制使用严格的 True/False/None 值

在版本 1.1 中，描述的更改将非本地布尔整数值强制为零/一/无的所有情况产生了一个意外的副作用，改变了当`Boolean`遇到非整数值（如字符串）时的行为。特别是，先前会生成值`False`的字符串值`"0"`现在会生成`True`。更糟糕的是，行为的更改只针对某些后端而不是其他后端，这意味着将字符串`"0"`值发送给`Boolean`的代码在不同后端上会不一致地中断。

这个问题的最终解决方案是**不支持将字符串值与布尔值一起使用**，因此在 1.2 中，如果传递了非整数/True/False/None 值，将引发严格的`TypeError`。此外，只接受整数值 0 和 1。

为了适应希望对布尔值有更自由解释的应用程序，应使用`TypeDecorator`。下面说明了一个配方，将允许对 1.1 之前的`Boolean`数据类型进行“自由”行为：

```py
from sqlalchemy import Boolean
from sqlalchemy import TypeDecorator

class LiberalBoolean(TypeDecorator):
    impl = Boolean

    def process_bind_param(self, value, dialect):
        if value is not None:
            value = bool(int(value))
        return value
```

[#4102](https://www.sqlalchemy.org/trac/ticket/4102)  ### 悲观的断开连接检测添加到连接池

连接池文档长期以来一直提供了一个使用`ConnectionEvents.engine_connect()`引擎事件在检出的连接上发出简单语句以测试其活动性的示例。现在，当与适当的方言一起使用时，此示例的功能已添加到连接池本身中。使用新参数`create_engine.pool_pre_ping`，每个检出的连接在返回之前将被测试是否新鲜：

```py
engine = create_engine("mysql+pymysql://", pool_pre_ping=True)
```

尽管“预检测”方法会在连接池检出时增加少量延迟，但对于典型的事务性应用（包括大多数 ORM 应用），这种开销很小，并且消除了获取到过期连接而引发错误、需要应用程序放弃或重试操作的问题。

该特性**不**适用于在进行中的事务或 SQL 操作中断开的连接。如果应用程序必须从中恢复，它需要使用自己的操作重试逻辑来预期这些错误。

另请参阅

断开连接处理 - 悲观

[#3919](https://www.sqlalchemy.org/trac/ticket/3919)  ### IN / NOT IN 运算符的空集合行为现在可配置；默认表达式简化

诸如`column.in_([])`这样的表达式，默认情况下现在会产生表达式`1 != 1`，而不是`column != column`。这将**改变查询的结果**，该查询比较了一个在与空集合进行比较时求值为 NULL 的 SQL 表达式或列，产生了布尔值 false 或 true（对于 NOT IN），而不是 NULL。在这种情况下发出的警告也被移除了。可以使用`create_engine()`的`create_engine.empty_in_strategy`参数来获得旧行为。

在 SQL 中，IN 和 NOT IN 运算符不支持显式空值集合的比较；也就是说，这种语法是非法的：

```py
mycolumn  IN  ()
```

为了解决这个问题，SQLAlchemy 和其他数据库库检测到这种情况，并渲染一个替代表达式，该表达式求值为 false，或者在 NOT IN 的情况下，求值为 true，根据“col IN ()”始终为 false 的理论，因为“空集合中没有任何东西”。通常为了生成一个跨数据库可移植且在 WHERE 子句的上下文中起作用的 false/true 常量，会使用一个简单的重言式，比如`1 != 1`会求值为 false，`1 = 1`会求值为 true（简单的常量“0”或“1”通常不能作为 WHERE 子句的目标）。

SQLAlchemy 在早期也采用了这种方法，但很快有人推测，如果“column”为 NULL，SQL 表达式`column IN ()`不会求值为 false；相反，该表达式将产生 NULL，因为“NULL”表示“未知”，而在 SQL 中与 NULL 的比较通常产生 NULL。

为了模拟这个结果，SQLAlchemy 从使用`1 != 1`改为使用表达式`expr != expr`来处理空的“IN”，并使用`expr = expr`处理空的“NOT IN”；也就是说，我们使用表达式的实际左侧而不是固定值。如果传递的表达式左侧评估为 NULL，则整体比较也会得到 NULL 结果，而不是 false 或 true。

不幸的是，用户最终抱怨这个表达式对一些查询规划器有非常严重的性能影响。在那时，当遇到空的 IN 表达式时添加了一个警告，支持 SQLAlchemy 继续保持“正确”，并敦促用户避免一般情况下生成空的 IN 谓词的代码，因为通常它们可以安全地省略。然而，在从输入变量动态构建查询的情况下，这在查询中是繁琐的，因为传入的值集可能为空。

最近几个月，对这个决定的最初假设进行了质疑。表达式“NULL IN ()”应该返回 NULL 的想法只是理论上的，无法测试，因为数据库不支持该语法。然而，事实证明，实际上可以通过模拟空集来询问关系数据库对“NULL IN ()”会返回什么值：

```py
SELECT  NULL  IN  (SELECT  1  WHERE  1  !=  1)
```

通过上述测试，我们发现数据库本身无法就答案达成一致。被大多数人认为是最“正确”的数据库 PostgreSQL 返回 False；因为即使“NULL”代表“未知”，“空集”意味着没有任何内容，包括所有未知值。另一方面，MySQL 和 MariaDB 对上述表达式返回 NULL，采用更常见的“所有与 NULL 的比较都返回 NULL”的行为。

SQLAlchemy 的 SQL 架构比最初做出这个设计决定时更复杂，因此我们现在可以在 SQL 字符串编译时调用任一行为。以前，转换为比较表达式是在构造时完成的，也就是说，在调用`ColumnOperators.in_()`或`ColumnOperators.notin_()`操作符时。通过编译时行为，方言本身可以被指示调用任一方法，即“静态”的`1 != 1`比较或“动态”的`expr != expr`比较。默认已经**更改**为“静态”比较，因为这与 PostgreSQL 的行为一致，而且这也是绝大多数用户喜欢的。这将**改变查询结果**，特别是对比较空表达式和空集的查询，特别是查询否定`where(~null_expr.in_([]))`，因为现在这将评估为 true 而不是 NULL。

现在可以使用标志 `create_engine.empty_in_strategy` 控制此行为，默认为`"static"`设置，但也可以设置为`"dynamic"`或`"dynamic_warn"`，其中`"dynamic_warn"`设置等效于以前的行为，即发出 `expr != expr` 以及性能警告。但预计大多数用户将欣赏`"static"`默认设置。

[#3907](https://www.sqlalchemy.org/trac/ticket/3907)  ### 支持延迟扩展的 IN 参数集，允许具有缓存语句的 IN 表达式

添加了一种名为“expanding”的新类型的 `bindparam()`。这是用于 `IN` 表达式的，其中元素列表在语句执行时被渲染为单独的绑定参数，而不是在语句编译时。这允许将单个绑定参数名称链接到多个元素的 IN 表达式，并且允许使用查询缓存与 IN 表达式。新功能允许相关功能的“select in”加载和“polymorphic in”加载利用烘焙查询扩展以减少调用开销：

```py
stmt = select([table]).where(table.c.col.in_(bindparam("foo", expanding=True)))
conn.execute(stmt, {"foo": [1, 2, 3]})
```

在 1.2 系列中，该功能应被视为 **实验性的**。

[#3953](https://www.sqlalchemy.org/trac/ticket/3953)  ### 对比运算符的展开操作优先级

对于诸如 IN、LIKE、等于、IS、MATCH 等比较运算符的运算符优先级已被展开为一个级别。当组合比较运算符时，将生成更多的括号化效果，例如：

```py
(column("q") == null()) != (column("y") == null())
```

现在将生成 `(q IS NULL) != (y IS NULL)` 而不是 `q IS NULL != y IS NULL`。

[#3999](https://www.sqlalchemy.org/trac/ticket/3999)  ### 支持对表、列的 SQL 注释，包括 DDL、反射

数据核心现支持与表和列相关联的字符串注释。这些通过 `Table.comment` 和 `Column.comment` 参数指定：

```py
Table(
    "my_table",
    metadata,
    Column("q", Integer, comment="the Q value"),
    comment="my Q table",
)
```

上述 DDL 将在创建表时适当渲染，以将上述注释与模式中的表/列关联起来。当上述表以自动加载或通过 `Inspector.get_columns()` 进行检查时，将包括这些注释。表注释也可以使用 `Inspector.get_table_comment()` 方法独立获取。

当前后端支持包括 MySQL、PostgreSQL 和 Oracle。

[#1546](https://www.sqlalchemy.org/trac/ticket/1546)  ### 支持 DELETE 的多表条件

`Delete` 构造现在支持多表条件，已在支持的后端实现，目前这些后端包括 PostgreSQL、MySQL 和 Microsoft SQL Server（对当前不工作的 Sybase 方言也添加了支持）。该功能的工作方式与 0.7 和 0.8 系列中首次引入的 UPDATE 的多表条件相同。

给出一个语句如下：

```py
stmt = (
    users.delete()
    .where(users.c.id == addresses.c.id)
    .where(addresses.c.email_address.startswith("ed%"))
)
conn.execute(stmt)
```

在 PostgreSQL 后端上，上述语句生成的 SQL 将呈现为：

```py
DELETE  FROM  users  USING  addresses
WHERE  users.id  =  addresses.id
AND  (addresses.email_address  LIKE  %(email_address_1)s  ||  '%%')
```

另请参阅

多表删除

[#959](https://www.sqlalchemy.org/trac/ticket/959)  ### 新的 “autoescape” 选项用于 startswith()、endswith()

“autoescape” 参数已添加到 `ColumnOperators.startswith()`、`ColumnOperators.endswith()`、`ColumnOperators.contains()`。当将此参数设置为 `True` 时，将自动使用转义字符转义所有 `%`、`_` 的出现，默认为斜杠 `/`；转义字符本身的出现也会被转义。斜杠用于避免与诸如 PostgreSQL 的 `standard_confirming_strings`、MySQL 的 `NO_BACKSLASH_ESCAPES` 等设置发生冲突。现在可以使用现有的 “escape” 参数来更改自动转义字符，如果需要的话。

注意

从 1.2.0 的初始实现 1.2.0b2 开始，此功能已更改，现在 autoescape 被传递为布尔值，而不是用作转义字符的特定字符。

诸如以下表达式：

```py
>>> column("x").startswith("total%score", autoescape=True)
```

呈现为：

```py
x  LIKE  :x_1  ||  '%'  ESCAPE  '/'
```

其中参数 “x_1” 的值为 `'total/%score'`。

同样，具有反斜杠的表达式：

```py
>>> column("x").startswith("total/score", autoescape=True)
```

将以相同方式呈现，参数 “x_1” 的值为 `'total//score'`。

[#2694](https://www.sqlalchemy.org/trac/ticket/2694)  ### “float” 数据类型增加更强的类型化

一系列更改允许使用 `Float` 数据类型更强烈地将其与 Python 浮点值联系起来，而不是更通用的 `Numeric`。这些更改主要涉及确保 Python 浮点值不会错误地被强制转换为 `Decimal()`，并且在需要时被强制转��为 `float`，在结果方面，如果应用程序正在处理普通浮点数。

+   当传递给 SQL 表达式的普通 Python “float” 值现在将被拉入具有类型 `Float` 的字面参数中；之前，该类型为 `Numeric`，带有默认的“asdecimal=True”标志，这意味着结果类型将被强制转换为 `Decimal()`。特别是，这将在 SQLite 上发出令人困惑的警告：

    ```py
    float_value = connection.scalar(
        select([literal(4.56)])  # the "BindParameter" will now be
        # Float, not Numeric(asdecimal=True)
    )
    ```

+   `Numeric`、`Float` 和 `Integer` 之间的数学运算现在将保留结果表达式的类型 `Numeric` 或 `Float`，包括 `asdecimal` 标志以及类型是否应为 `Float`：

    ```py
    # asdecimal flag is maintained
    expr = column("a", Integer) * column("b", Numeric(asdecimal=False))
    assert expr.type.asdecimal == False

    # Float subclass of Numeric is maintained
    expr = column("a", Integer) * column("b", Float())
    assert isinstance(expr.type, Float)
    ```

+   如果 DBAPI 已知支持本地 `Decimal()` 模式，则 `Float` 数据类型将无条件地将 `float()` 处理器应用于结果值。某些后端不总是保证浮点数返回为普通浮点数，而不是诸如 MySQL 等精度数字。

[#4017](https://www.sqlalchemy.org/trac/ticket/4017)

[#4018](https://www.sqlalchemy.org/trac/ticket/4018)

[#4020](https://www.sqlalchemy.org/trac/ticket/4020)

### 对 GROUPING SETS、CUBE、ROLLUP 的支持

GROUPING SETS、CUBE、ROLLUP 三者都可以通过 `func` 命名空间使用。对于 CUBE 和 ROLLUP，在之前的版本中这些函数已经可以使用，但对于 GROUPING SETS，在编译器中添加了一个占位符以允许空间。这三个函数现在在文档中命名：

```py
>>> from sqlalchemy import select, table, column, func, tuple_
>>> t = table("t", column("value"), column("x"), column("y"), column("z"), column("q"))
>>> stmt = select([func.sum(t.c.value)]).group_by(
...     func.grouping_sets(
...         tuple_(t.c.x, t.c.y),
...         tuple_(t.c.z, t.c.q),
...     )
... )
>>> print(stmt)
SELECT  sum(t.value)  AS  sum_1
FROM  t  GROUP  BY  GROUPING  SETS((t.x,  t.y),  (t.z,  t.q)) 
```

[#3429](https://www.sqlalchemy.org/trac/ticket/3429)

### 用于具有上下文默认生成器的多值插入的参数助手

默认生成函数，例如在上下文敏感的默认函数中描述的函数，可以通过`DefaultExecutionContext.current_parameters`属性查看与语句相关的当前参数。然而，在通过`Insert.values()`方法指定多个 VALUES 子句的`Insert`构造中，用户定义的函数会被多次调用，每个参数集一次，但是无法知道`DefaultExecutionContext.current_parameters`中的哪个键子集适用于该列。添加了一个新函数`DefaultExecutionContext.get_current_parameters()`，其中包括一个关键字参数`DefaultExecutionContext.get_current_parameters.isolate_multiinsert_groups`默认为`True`，执行额外的工作，提供一个`DefaultExecutionContext.current_parameters`的子字典，其中的名称局限于当前正在处理的 VALUES 子句：

```py
def mydefault(context):
    return context.get_current_parameters()["counter"] + 12

mytable = Table(
    "mytable",
    metadata_obj,
    Column("counter", Integer),
    Column("counter_plus_twelve", Integer, default=mydefault, onupdate=mydefault),
)

stmt = mytable.insert().values([{"counter": 5}, {"counter": 18}, {"counter": 20}])

conn.execute(stmt)
```

[#4075](https://www.sqlalchemy.org/trac/ticket/4075)  ### 布尔数据类型现在强制使用严格的 True/False/None 值

在版本 1.1 中，将非本地布尔整数值强制转换为零/一/None 的所有情况中描述的更改产生了一个意外的副作用，改变了当`Boolean`遇到非整数值（如字符串）时的行为。特别是，先前会生成值`False`的字符串值`"0"`，现在会生成`True`。更糟糕的是，行为的变化只针对某些后端而不是其他后端，这意味着将字符串`"0"`值发送给`Boolean`的代码在不同后端上会不一致地出现故障。

这个问题的最终解决方案是**不支持将字符串值与布尔值一起使用**，因此在 1.2 版本中，如果传递了非整数/True/False/None 值，将会引发严格的`TypeError`。此外，只有整数值 0 和 1 会被接受。

为了适应希望对布尔值有更自由解释的应用程序，应该使用`TypeDecorator`。下面演示了一个配方，可以允许在 1.1 版本之前的`Boolean`数据类型的“自由”行为：

```py
from sqlalchemy import Boolean
from sqlalchemy import TypeDecorator

class LiberalBoolean(TypeDecorator):
    impl = Boolean

    def process_bind_param(self, value, dialect):
        if value is not None:
            value = bool(int(value))
        return value
```

[#4102](https://www.sqlalchemy.org/trac/ticket/4102)

### 将悲观的断开检测添加到连接池

连接池文档长期以来一直提供了一个使用`ConnectionEvents.engine_connect()`引擎事件在检出的连接上发出简单语句以测试其活动性的配方。现在，当与适当的方言一起使用时，此配方的功能已经添加到连接池本身中。使用新参数`create_engine.pool_pre_ping`，每个检出的连接在返回之前都会被测试是否仍然有效：

```py
engine = create_engine("mysql+pymysql://", pool_pre_ping=True)
```

虽然“预检”方法会在连接池检出时增加一小部分延迟，但对于典型的面向事务的应用程序（其中包括大多数 ORM 应用程序），这种开销是很小的，并且消除了获取到一个过时连接会引发错误的问题，需要应用程序放弃或重试操作。

该功能**不**适用于在进行中的事务或 SQL 操作中断开的连接。如果应用程序必须从这些错误中恢复，它需要使用自己的操作重试逻辑来预期这些错误。

另请参阅

断开处理 - 悲观

[#3919](https://www.sqlalchemy.org/trac/ticket/3919)

### IN / NOT IN 运算符的空集合行为现在是可配置的；默认表达式简化

例如，假设`column.in_([])`这样的表达式被假定为 false，默认情况下现在会产生表达式`1 != 1`，而不是`column != column`。这将**改变查询结果**，如果比较 SQL 表达式或列与空集合时评估为 NULL，则会产生布尔值 false 或 true（对于 NOT IN），而不是 NULL。在这种情况下发出的警告也被移除了。可以使用`create_engine.empty_in_strategy`参数来`create_engine()`，以保留旧的行为。

在 SQL 中，IN 和 NOT IN 运算符不支持与明确为空的值集合进行比较；也就是说，以下语法是不合法的：

```py
mycolumn  IN  ()
```

为了解决这个问题，SQLAlchemy 和其他数据库库检测到这种情况，并生成一个替代表达式，该表达式评估为 false，或者在 NOT IN 的情况下，根据“col IN ()”始终为 false 的理论，评估为 true，因为“空集合”中没有任何内容。通常，为了生成一个跨数据库可移植且在 WHERE 子句上下文中起作用的 false/true 常量，会使用一个简单的重言式，比如`1 != 1`评估为 false，`1 = 1`评估为 true（一个简单的常量“0”或“1”通常不能作为 WHERE 子句的目标）。

SQLAlchemy 在早期也采用了这种方法，但很快就有人推测，如果 SQL 表达式`column IN ()`中的“column”为 NULL，则不会评估为 false；相反，该表达式会产生 NULL，因为“NULL”表示“未知”，而在 SQL 中与 NULL 的比较通常会产生 NULL。

为了模拟这个结果，SQLAlchemy 从使用`1 != 1`改为使用表达式`expr != expr`来表示空的“IN”，以及使用`expr = expr`来表示空的“NOT IN”；也就是说，我们不再使用固定值，而是使用表达式的实际左侧。如果传递的表达式左侧评估为 NULL，则比较整体也会得到 NULL 结果，而不是 false 或 true。

不幸的是，用户最终抱怨说这个表达式对一些查询规划器有非常严重的性能影响。在那时，当遇到空的 IN 表达式时，添加了一个警告，建议 SQLAlchemy 继续保持“正确”，并敦促用户避免生成通常可以安全省略的空 IN 谓词的代码。然而，在动态构建查询的情况下，这在输入变量为空时可能会带来负担。

近几个月来，对这个决定的原始假设受到了质疑。认为表达式“NULL IN ()”应该返回 NULL 的想法只是理论上的，无法进行测试，因为数据库不支持该语法。然而，事实证明，你确实可以询问关系数据库关于“NULL IN ()”将返回什么值，方法是模拟空集如下：

```py
SELECT  NULL  IN  (SELECT  1  WHERE  1  !=  1)
```

通过上述测试，我们发现数据库本身无法就答案达成一致。大多数人认为最“正确”的数据库 PostgreSQL 返回 False；因为即使“NULL”表示“未知”，“空集”表示什么都没有，包括所有未知的值。另一方面，MySQL 和 MariaDB 返回上述表达式的 NULL，默认为“所有与 NULL 的比较都返回 NULL”的更常见行为。

SQLAlchemy 的 SQL 架构比最初做出这个设计决定时更复杂，因此我们现在可以在 SQL 字符串编译时调用任一行为。先前，将转换为比较表达式是在构建时完成的，也就是说，在调用 `ColumnOperators.in_()` 或 `ColumnOperators.notin_()` 操作符时完成。通过编译时行为，方言本身可以被指示调用任一方法，即“static” `1 != 1` 比较或“dynamic” `expr != expr` 比较。默认值已经**更改**为“static”比较，因为这与 PostgreSQL 在任何情况下的行为一致，这也是绝大多数用户喜欢的。这将**改变**查询空表达式与空集的比较结果，特别是查询否定 `where(~null_expr.in_([]))` 的查询，因为现在这将计算为 true 而不是 NULL。

可以使用标志`create_engine.empty_in_strategy`来控制行为，该标志默认为`"static"`设置，但也可以设置为`"dynamic"`或`"dynamic_warn"`，其中`"dynamic_warn"`设置等效于以前的行为，即发出`expr != expr`以及性能警告。但预计大多数用户会喜欢`"static"`默认值。

[#3907](https://www.sqlalchemy.org/trac/ticket/3907)

### 晚扩展的 IN 参数集允许使用缓存语句的 IN 表达式

添加了一种名为“expanding”的新类型的`bindparam()`。这用于在`IN`表达式中，元素列表在语句执行时被渲染为单独的绑定参数，而不是在语句编译时。这允许将单个绑定参数名称链接到多个元素的 IN 表达式，并允许使用查询缓存与 IN 表达式一起使用。新功能允许“select in”加载和“polymorphic in”加载相关功能利用烘焙查询扩展以减少调用开销：

```py
stmt = select([table]).where(table.c.col.in_(bindparam("foo", expanding=True)))
conn.execute(stmt, {"foo": [1, 2, 3]})
```

该功能应被视为**实验性**，属于 1.2 系列。

[#3953](https://www.sqlalchemy.org/trac/ticket/3953)

### 比较运算符的操作符优先级已经被展开

对于 IN、LIKE、equals、IS、MATCH 和其他比较运算符等运算符的操作符优先级已被展开为一个级别。当比较运算符组合在一起时，将生成更多的括号，例如：

```py
(column("q") == null()) != (column("y") == null())
```

现在将生成`(q IS NULL) != (y IS NULL)`而不是`q IS NULL != y IS NULL`。

[#3999](https://www.sqlalchemy.org/trac/ticket/3999)

### 支持在表、列上添加 SQL 注释，包括 DDL、反射

核心支持与表和列相关的字符串注释。这些通过`Table.comment`和`Column.comment`参数指定：

```py
Table(
    "my_table",
    metadata,
    Column("q", Integer, comment="the Q value"),
    comment="my Q table",
)
```

上述 DDL 将在表创建时适当地呈现，以将上述注释与模式中的表/列关联起来。当上述表被自动加载或使用`Inspector.get_columns()`检查时，注释将被包含在内。表注释也可以通过`Inspector.get_table_comment()`方法独立获取。

当前后端支持包括 MySQL、PostgreSQL 和 Oracle。

[#1546](https://www.sqlalchemy.org/trac/ticket/1546)

### DELETE 的多表条件支持

`Delete`构造现在支持多表条件，已在支持的后端实现，目前这些后端是 PostgreSQL、MySQL 和 Microsoft SQL Server（支持也已添加到当前不工作的 Sybase 方言）。该功能的工作方式与 0.7 和 0.8 系列中首次引入的 UPDATE 的多表条件相同。

给定语句如下：

```py
stmt = (
    users.delete()
    .where(users.c.id == addresses.c.id)
    .where(addresses.c.email_address.startswith("ed%"))
)
conn.execute(stmt)
```

在 PostgreSQL 后端上，上述语句的结果 SQL 将呈现为：

```py
DELETE  FROM  users  USING  addresses
WHERE  users.id  =  addresses.id
AND  (addresses.email_address  LIKE  %(email_address_1)s  ||  '%%')
```

另请参见

多表删除

[#959](https://www.sqlalchemy.org/trac/ticket/959)

### 新的“autoescape”选项用于 startswith()，endswith()

“autoescape”参数被添加到`ColumnOperators.startswith()`，`ColumnOperators.endswith()`，`ColumnOperators.contains()`中。当将此参数设置为`True`时，将自动使用转义字符转义所有出现的`%`、`_`，默认为斜杠`/`；转义字符本身的出现也会被转义。斜杠用于避免与诸如 PostgreSQL 的`standard_confirming_strings`（从 PostgreSQL 9.1 开始更改默认值）和 MySQL 的`NO_BACKSLASH_ESCAPES`设置等设置发生冲突。现在可以使用现有的“escape”参数来更改自动转义字符，如果需要的话。

注意

从 1.2.0 的初始实现 1.2.0b2 开始，此功能已更改，现在 autoescape 被传递为布尔值，而不是用作转义字符的特定字符。

例如：

```py
>>> column("x").startswith("total%score", autoescape=True)
```

渲染为：

```py
x  LIKE  :x_1  ||  '%'  ESCAPE  '/'
```

其中参数“x_1”的值为`'total/%score'`。

同样，具有反斜杠的表达式：

```py
>>> column("x").startswith("total/score", autoescape=True)
```

将以相同方式渲染，参数“x_1”的值为`'total//score'`。

[#2694](https://www.sqlalchemy.org/trac/ticket/2694)

### 对“float”数据类型进行了更强的类型化

一系列更改允许使用`Float`数据类型更强烈地将其与 Python 浮点值关联起来，而不是更通用的`Numeric`。这些更改主要涉及确保 Python 浮点值不会错误地被强制转换为`Decimal()`，并且在需要时，如果应用程序正在处理普通浮点数，则会被强制转换为`float`。

+   传递给 SQL 表达式的普通 Python“float”值现在将被拉入具有类型`Float`的文字参数；以前，类型为`Numeric`，带有默认的“asdecimal=True”标志，这意味着结果类型将强制转换为`Decimal()`。特别是，这将在 SQLite 上发出令人困惑的警告：

    ```py
    float_value = connection.scalar(
        select([literal(4.56)])  # the "BindParameter" will now be
        # Float, not Numeric(asdecimal=True)
    )
    ```

+   在 `Numeric`、`Float` 和 `Integer` 之间的数学操作现在将保留结果表达式的类型，包括 `asdecimal` 标志以及类型是否应为 `Float`： 

    ```py
    # asdecimal flag is maintained
    expr = column("a", Integer) * column("b", Numeric(asdecimal=False))
    assert expr.type.asdecimal == False

    # Float subclass of Numeric is maintained
    expr = column("a", Integer) * column("b", Float())
    assert isinstance(expr.type, Float)
    ```

+   `Float` 数据类型将始终将 `float()` 处理器应用于结果值，如果 DBAPI 已知支持原生 `Decimal()` 模式。一些后端并不总是保证浮点数作为普通浮点数返回，而不是像 MySQL 这样的精度数字。

[#4017](https://www.sqlalchemy.org/trac/ticket/4017)

[#4018](https://www.sqlalchemy.org/trac/ticket/4018)

[#4020](https://www.sqlalchemy.org/trac/ticket/4020)

### 支持 GROUPING SETS、CUBE、ROLLUP

GROUPING SETS、CUBE、ROLLUP 这三个功能都可以通过 `func` 命名空间来调用。在 CUBE 和 ROLLUP 的情况下，这些函数在之前的版本中已经可以使用，但是对于 GROUPING SETS，编译器中添加了一个占位符以允许使用这个功能。现在文档中已经命名了这三个函数：

```py
>>> from sqlalchemy import select, table, column, func, tuple_
>>> t = table("t", column("value"), column("x"), column("y"), column("z"), column("q"))
>>> stmt = select([func.sum(t.c.value)]).group_by(
...     func.grouping_sets(
...         tuple_(t.c.x, t.c.y),
...         tuple_(t.c.z, t.c.q),
...     )
... )
>>> print(stmt)
SELECT  sum(t.value)  AS  sum_1
FROM  t  GROUP  BY  GROUPING  SETS((t.x,  t.y),  (t.z,  t.q)) 
```

[#3429](https://www.sqlalchemy.org/trac/ticket/3429)

### 多值插入的参数辅助器，带有上下文默认生成器

默认生成函数，例如在上下文敏感的默认函数中描述的函数，可以通过`DefaultExecutionContext.current_parameters`属性查看与语句相关的当前参数。然而，在通过`Insert.values()`方法指定多个 VALUES 子句的`Insert`构造中，用户定义的函数会被多次调用，每个参数集一次，但是无法知道`DefaultExecutionContext.current_parameters`中的哪些键子集适用于该列。添加了一个新函数`DefaultExecutionContext.get_current_parameters()`，其中包括一个关键字参数`DefaultExecutionContext.get_current_parameters.isolate_multiinsert_groups`默认为`True`，执行额外的工作，提供一个`DefaultExecutionContext.current_parameters`的子字典，其中的名称局限于当前正在处理的 VALUES 子句：

```py
def mydefault(context):
    return context.get_current_parameters()["counter"] + 12

mytable = Table(
    "mytable",
    metadata_obj,
    Column("counter", Integer),
    Column("counter_plus_twelve", Integer, default=mydefault, onupdate=mydefault),
)

stmt = mytable.insert().values([{"counter": 5}, {"counter": 18}, {"counter": 20}])

conn.execute(stmt)
```

[#4075](https://www.sqlalchemy.org/trac/ticket/4075)

## 关键行为变化 - ORM

### after_rollback() Session 事件现在在对象过期之前触发

`SessionEvents.after_rollback()` 事件现在可以访问对象在其状态被过期之前的属性状态（例如“快照移除”）。这使得该事件与`SessionEvents.after_commit()`事件的行为保持一致，后者在“快照”被移除之前也会触发：

```py
sess = Session()

user = sess.query(User).filter_by(name="x").first()

@event.listens_for(sess, "after_rollback")
def after_rollback(session):
    # 'user.name' is now present, assuming it was already
    # loaded.  previously this would raise upon trying
    # to emit a lazy load.
    print("user name: %s" % user.name)

@event.listens_for(sess, "after_commit")
def after_commit(session):
    # 'user.name' is present, assuming it was already
    # loaded.  this is the existing behavior.
    print("user name: %s" % user.name)

if should_rollback:
    sess.rollback()
else:
    sess.commit()
```

请注意，`Session` 仍将禁止在此事件中发出 SQL；这意味着未加载的属性仍无法在事件范围内加载。

[#3934](https://www.sqlalchemy.org/trac/ticket/3934)  ### 修复了与 `select_from()` 一起使用单表继承的问题

当生成 SQL 时，`Query.select_from()` 方法现在会尊重单表继承列鉴别器；之前，只有查询列列表中的表达式会被考虑。

假设 `Manager` 是 `Employee` 的子类。像下面这样的查询：

```py
sess.query(Manager.id)
```

会生成如下 SQL：

```py
SELECT  employee.id  FROM  employee  WHERE  employee.type  IN  ('manager')
```

然而，如果 `Manager` 只是通过 `Query.select_from()` 指定而不在列列表中，鉴别器将不会被添加：

```py
sess.query(func.count(1)).select_from(Manager)
```

会生成：

```py
SELECT  count(1)  FROM  employee
```

通过修复，`Query.select_from()` 现在可以正常工作，我们得到：

```py
SELECT  count(1)  FROM  employee  WHERE  employee.type  IN  ('manager')
```

可能一直通过手动提供 WHERE 子句来解决此问题的应用程序可能需要进行调整。

[#3891](https://www.sqlalchemy.org/trac/ticket/3891)  ### 在替换时不再改变先前集合

ORM 在映射集合成员更改时会发出事件。将集合分配给将替换先前集合的属性时，这样做的一个副作用是，被替换的集合也会被改变，这是误导性的和不必要的：

```py
>>> a1, a2, a3 = Address("a1"), Address("a2"), Address("a3")
>>> user.addresses = [a1, a2]

>>> previous_collection = user.addresses

# replace the collection with a new one
>>> user.addresses = [a2, a3]

>>> previous_collection
[Address('a1'), Address('a2')]
```

在更改之前，`previous_collection` 将删除“a1”成员，对应于不再在新集合中的成员。

[#3913](https://www.sqlalchemy.org/trac/ticket/3913)  ### 在批量集合设置之前，@validates 方法会接收所有值

使用 `@validates` 的方法现在在“批量设置”操作期间会接收集合的所有成员，然后再应用比较到现有集合上。

给定一个映射如下：

```py
class A(Base):
    __tablename__ = "a"
    id = Column(Integer, primary_key=True)
    bs = relationship("B")

    @validates("bs")
    def convert_dict_to_b(self, key, value):
        return B(data=value["data"])

class B(Base):
    __tablename__ = "b"
    id = Column(Integer, primary_key=True)
    a_id = Column(ForeignKey("a.id"))
    data = Column(String)
```

在上面的示例中，我们可以如下使用验证器，将传入的字典转换为 `B` 的实例并在集合附加时使用：

```py
a1 = A()
a1.bs.append({"data": "b1"})
```

然而，集合赋值会失败，因为 ORM 会假定传入的对象已经是 `B` 的实例，因此在进行集合附加之前会尝试将它们与集合的现有成员进行比较，而实际上是在执行调用验证器的集合附加之前。这将使得批量设置操作无法适应需要事先修改的非 ORM 对象，比如字典：

```py
a1 = A()
a1.bs = [{"data": "b1"}]
```

新逻辑使用新的 `AttributeEvents.bulk_replace()` 事件，以确保所有值都会被提前发送到 `@validates` 函数。

作为这一变化的一部分，这意味着验证器现在将在批量设置时接收**所有**集合成员，而不仅仅是新成员。假设一个简单的验证器如下：

```py
class A(Base):
    # ...

    @validates("bs")
    def validate_b(self, key, value):
        assert value.data is not None
        return value
```

如果我们开始的集合如下：

```py
a1 = A()

b1, b2 = B(data="one"), B(data="two")
a1.bs = [b1, b2]
```

然后，用与第一个重叠的集合替换了原集合：

```py
b3 = B(data="three")
a1.bs = [b2, b3]
```

以前，第二次赋值只会触发一次`A.validate_b`方法，对于`b3`对象。`b2`对象将被视为已经存在于集合中且不会被验证。使用新行为，`b2`和`b3`都会在传递到集合之前传递给`A.validate_b`。因此，验证方法必须具有幂等行为以适应这种情况。

另请参阅

新的批量替换事件

[#3896](https://www.sqlalchemy.org/trac/ticket/3896)  ### 使用 flag_dirty()标记对象为“脏”而不更改任何属性

如果`flag_modified()`函数用于标记未加载的属性为已修改，则会引发异常：

```py
a1 = A(data="adf")
s.add(a1)

s.flush()

# expire, similarly as though we said s.commit()
s.expire(a1, "data")

# will raise InvalidRequestError
attributes.flag_modified(a1, "data")
```

这是因为如果属性在刷新发生时仍未出现，那么刷新过程很可能会失败。要将对象标记为“已修改”而不指定任何特定属性，以便在自定义事件处理程序（如`SessionEvents.before_flush()`）中考虑到刷新过程中，使用新的`flag_dirty()`函数：

```py
from sqlalchemy.orm import attributes

attributes.flag_dirty(a1)
```

[#3753](https://www.sqlalchemy.org/trac/ticket/3753)  ### 从 scoped_session 中移除“scope”关键字

一个非常古老且未记录的关键字参数`scope`已被移除：

```py
from sqlalchemy.orm import scoped_session

Session = scoped_session(sessionmaker())

session = Session(scope=None)
```

该关键字的目的是尝试允许变量“作用域”，其中`None`表示“无作用域”，因此会返回一个新的`Session`。该关键字从未被记录在案，如果遇到将会引发`TypeError`。预计该关键字未被使用，但如果用户在测试期间报告与此相关的问题，可以通过弃用来恢复。

[#3796](https://www.sqlalchemy.org/trac/ticket/3796)  ### 与 onupdate 一起对 post_update 进行细化

使用`relationship.post_update`功能的关系现在将与设置了`Column.onupdate`值的列更好地交互。如果插入对象时为列显式指定了值，则在 UPDATE 期间会重新声明该值，以便“onupdate”规则不会覆盖它：

```py
class A(Base):
    __tablename__ = "a"
    id = Column(Integer, primary_key=True)
    favorite_b_id = Column(ForeignKey("b.id", name="favorite_b_fk"))
    bs = relationship("B", primaryjoin="A.id == B.a_id")
    favorite_b = relationship(
        "B", primaryjoin="A.favorite_b_id == B.id", post_update=True
    )
    updated = Column(Integer, onupdate=my_onupdate_function)

class B(Base):
    __tablename__ = "b"
    id = Column(Integer, primary_key=True)
    a_id = Column(ForeignKey("a.id", name="a_fk"))

a1 = A()
b1 = B()

a1.bs.append(b1)
a1.favorite_b = b1
a1.updated = 5
s.add(a1)
s.flush()
```

在上述情况下，以前的行为是 UPDATE 会在 INSERT 之后发出，从而触发“onupdate”并覆盖值“5”。现在的 SQL 如下：

```py
INSERT  INTO  a  (favorite_b_id,  updated)  VALUES  (?,  ?)
(None,  5)
INSERT  INTO  b  (a_id)  VALUES  (?)
(1,)
UPDATE  a  SET  favorite_b_id=?,  updated=?  WHERE  a.id  =  ?
(1,  5,  1)
```

此外，如果“updated”的值*未*设置，则我们将正确地在`a1.updated`上获取新生成的值；以前，刷新或过期属性以允许生成的值存在的逻辑不会为 post-update 触发。在这种情况下，当刷新中发生 flush 时，也会发出`InstanceEvents.refresh_flush()`事件。

[#3471](https://www.sqlalchemy.org/trac/ticket/3471)

[#3472](https://www.sqlalchemy.org/trac/ticket/3472)  ### post_update 与 ORM 版本控制集成

post_update 功能，文档化在指向自身的行 / 相互依赖的行，涉及对特定关系绑定外键的更改发出 UPDATE 语句，除了针对目标行通常会发出的 INSERT/UPDATE/DELETE。此 UPDATE 语句现在参与版本控制功能，文档化在配置版本计数器。

给定一个映射：

```py
class Node(Base):
    __tablename__ = "node"
    id = Column(Integer, primary_key=True)
    version_id = Column(Integer, default=0)
    parent_id = Column(ForeignKey("node.id"))
    favorite_node_id = Column(ForeignKey("node.id"))

    nodes = relationship("Node", primaryjoin=remote(parent_id) == id)
    favorite_node = relationship(
        "Node", primaryjoin=favorite_node_id == remote(id), post_update=True
    )

    __mapper_args__ = {"version_id_col": version_id}
```

更新将另一个节点关联为“favorite”的节点的 UPDATE 现在也会增加版本计数器，并匹配当前版本：

```py
node = Node()
session.add(node)
session.commit()  # node is now version #1

node = session.query(Node).get(node.id)
node.favorite_node = Node()
session.commit()  # node is now version #2
```

请注意，这意味着一个对象由于其他属性的更改而接收到一个 UPDATE，以及由于 post_update 关系更改而接收到第二个 UPDATE，现在将会为一个 flush 接收到**两个版本计数器更新**。但是，如果对象在当前 flush 中受到 INSERT 的影响，则版本计数器**不会**额外增加一次，除非存在服务器端版本控制方案。

现在讨论 post_update 为什么即使是 UPDATE 也会发出 UPDATE 的原因在为什么 post_update 除了第一个 UPDATE 还会发出 UPDATE？。

另请参阅

指向自身的行 / 相互依赖的行

为什么 post_update 除了第一个 UPDATE 还会发出 UPDATE？

[#3496](https://www.sqlalchemy.org/trac/ticket/3496)  ### 在对象过期之前，after_rollback() 会发出 Session 事件

`SessionEvents.after_rollback()` 事件现在可以在对象状态被过期之前访问属性状态（例如“快照移除”）。这使得事件与`SessionEvents.after_commit()`事件的行为一致，后者也会在“快照”被移除之前发出：

```py
sess = Session()

user = sess.query(User).filter_by(name="x").first()

@event.listens_for(sess, "after_rollback")
def after_rollback(session):
    # 'user.name' is now present, assuming it was already
    # loaded.  previously this would raise upon trying
    # to emit a lazy load.
    print("user name: %s" % user.name)

@event.listens_for(sess, "after_commit")
def after_commit(session):
    # 'user.name' is present, assuming it was already
    # loaded.  this is the existing behavior.
    print("user name: %s" % user.name)

if should_rollback:
    sess.rollback()
else:
    sess.commit()
```

请注意，`Session`仍将禁止在此事件中发出 SQL；这意味着未加载的属性仍将无法在事件范围内加载。

[#3934](https://www.sqlalchemy.org/trac/ticket/3934)

### 修复了与`select_from()`一起使用单表继承的问题

`Query.select_from()`方法现在在生成 SQL 时尊重单表继承列鉴别器；以前，只有查询列列表中的表达式会被考虑进去。

假设`Manager`是`Employee`的子类。像下面这样的查询：

```py
sess.query(Manager.id)
```

将生成的 SQL 如下：

```py
SELECT  employee.id  FROM  employee  WHERE  employee.type  IN  ('manager')
```

然而，如果`Manager`仅由`Query.select_from()`指定，而不在列列表中，那么鉴别器将不会被添加：

```py
sess.query(func.count(1)).select_from(Manager)
```

将生成：

```py
SELECT  count(1)  FROM  employee
```

通过修复，`Query.select_from()`现在可以正常工作，我们得到：

```py
SELECT  count(1)  FROM  employee  WHERE  employee.type  IN  ('manager')
```

可能一直通过手动提供 WHERE 子句来解决此问题的应用程序可能需要进行调整。

[#3891](https://www.sqlalchemy.org/trac/ticket/3891)

### 在替换���不再改变先前集合

ORM 在映射集合的成员发生变化时会发出事件。将集合分配给将替换先前集合的属性时，这样做的一个副作用是，被替换的集合也会被改变，这是误导性和不必要的：

```py
>>> a1, a2, a3 = Address("a1"), Address("a2"), Address("a3")
>>> user.addresses = [a1, a2]

>>> previous_collection = user.addresses

# replace the collection with a new one
>>> user.addresses = [a2, a3]

>>> previous_collection
[Address('a1'), Address('a2')]
```

在上面的示例中，在更改之前，`previous_collection`将删除“a1”成员，对应于不再在新集合中的成员。

[#3913](https://www.sqlalchemy.org/trac/ticket/3913)

### 在比较之前，@validates 方法会接收批量集合设置的所有值

使用`@validates`的方法现在在“批量设置”操作期间将接收集合的所有成员，然后再将比较应用于现有集合。

给定一个映射如下：

```py
class A(Base):
    __tablename__ = "a"
    id = Column(Integer, primary_key=True)
    bs = relationship("B")

    @validates("bs")
    def convert_dict_to_b(self, key, value):
        return B(data=value["data"])

class B(Base):
    __tablename__ = "b"
    id = Column(Integer, primary_key=True)
    a_id = Column(ForeignKey("a.id"))
    data = Column(String)
```

在上面的示例中，我们可以使用验证器如下，将传入的字典转换为`B`的实例进行集合附加：

```py
a1 = A()
a1.bs.append({"data": "b1"})
```

然而，集合赋值将失败，因为 ORM 会假定传入的对象已经是`B`的实例，因为它在尝试将它们与集合的现有成员进行比较之前，会执行集合附加操作，这实际上会调用验证器。这将使得批量设置操作无法容纳像需要事先修改的字典这样的非 ORM 对象：

```py
a1 = A()
a1.bs = [{"data": "b1"}]
```

新逻辑使用新的`AttributeEvents.bulk_replace()`事件，以确保所有值都被提前发送到`@validates`函数。

作为这一变化的一部分，现在验证器将在批量设置时接收**所有**集合成员，而不仅仅是新成员。假设一个简单的验证器如下：

```py
class A(Base):
    # ...

    @validates("bs")
    def validate_b(self, key, value):
        assert value.data is not None
        return value
```

如上所述，如果我们从一个集合开始：

```py
a1 = A()

b1, b2 = B(data="one"), B(data="two")
a1.bs = [b1, b2]
```

然后，用一个与第一个重叠的集合替换了该集合：

```py
b3 = B(data="three")
a1.bs = [b2, b3]
```

以前，第二次赋值只会触发一次`A.validate_b`方法，对于`b3`对象。`b2`对象将被视为已经存在于集合中并且不会被验证。通过新行为，`b2`和`b3`都会在传递到集合之前传递给`A.validate_b`。因此，验证方法必须采用幂等行为以适应这种情况。

另请参阅

新的 bulk_replace 事件

[#3896](https://www.sqlalchemy.org/trac/ticket/3896)

### 使用 flag_dirty()将对象标记为“dirty”而不改变任何属性

如果使用`flag_modified()`函数标记一个未加载的属性为已修改，现在会引发异常：

```py
a1 = A(data="adf")
s.add(a1)

s.flush()

# expire, similarly as though we said s.commit()
s.expire(a1, "data")

# will raise InvalidRequestError
attributes.flag_modified(a1, "data")
```

这是因为如果属性在刷新发生时仍未出现，则刷新过程很可能会失败。要将对象标记为“修改”，而不是特指任何属性，以便考虑到自定义事件处理程序（如`SessionEvents.before_flush()`）的刷新过程中，使用新的`flag_dirty()`函数：

```py
from sqlalchemy.orm import attributes

attributes.flag_dirty(a1)
```

[#3753](https://www.sqlalchemy.org/trac/ticket/3753)

### 从 scoped_session 中移除了“scope”关键字

一个非常古老且未记录的关键字参数`scope`已被移除：

```py
from sqlalchemy.orm import scoped_session

Session = scoped_session(sessionmaker())

session = Session(scope=None)
```

此关键字的目的是尝试允许变量“scopes”，其中`None`表示“无范围”，因此将返回一个新的`Session`。该关键字从未被记录在案，如果遇到将引发`TypeError`。预计不会使用此关键字，但如果用户在测试期间报告与此相关的问题，则可以通过弃用来恢复。

[#3796](https://www.sqlalchemy.org/trac/ticket/3796)

### 与 onupdate 一起的 post_update 的改进

使用`relationship.post_update`功能的关系现在将更好地与设置了`Column.onupdate`值的列交互。如果插入对象时为列显式指定了值，则在 UPDATE 期间将重新声明该值，以便“onupdate”规则不会覆盖它：

```py
class A(Base):
    __tablename__ = "a"
    id = Column(Integer, primary_key=True)
    favorite_b_id = Column(ForeignKey("b.id", name="favorite_b_fk"))
    bs = relationship("B", primaryjoin="A.id == B.a_id")
    favorite_b = relationship(
        "B", primaryjoin="A.favorite_b_id == B.id", post_update=True
    )
    updated = Column(Integer, onupdate=my_onupdate_function)

class B(Base):
    __tablename__ = "b"
    id = Column(Integer, primary_key=True)
    a_id = Column(ForeignKey("a.id", name="a_fk"))

a1 = A()
b1 = B()

a1.bs.append(b1)
a1.favorite_b = b1
a1.updated = 5
s.add(a1)
s.flush()
```

在上述情况下，以前的行为是在插入之后会发出更新，从而触发“onupdate”，并覆盖值“5”。现在的 SQL 如下：

```py
INSERT  INTO  a  (favorite_b_id,  updated)  VALUES  (?,  ?)
(None,  5)
INSERT  INTO  b  (a_id)  VALUES  (?)
(1,)
UPDATE  a  SET  favorite_b_id=?,  updated=?  WHERE  a.id  =  ?
(1,  5,  1)
```

此外，如果“updated”的值 *未* 设置，则我们将在 `a1.updated` 上正确地获得新生成的值；以前，刷新或过期属性的逻辑以允许生成的值存在将不会触发 post-update。在此情况下，还会在刷新期间发出 `InstanceEvents.refresh_flush()` 事件。

[#3471](https://www.sqlalchemy.org/trac/ticket/3471)

[#3472](https://www.sqlalchemy.org/trac/ticket/3472)

### post_update 与 ORM 版本控制集成

post_update 功能，记录在 指向自身的行 / 相互依赖的行，涉及响应特定关系绑定外键的更改而发出 UPDATE 语句，除了为目标行正常发出的 INSERT/UPDATE/DELETE 外。此 UPDATE 语句现在参与版本控制功能，记录在 配置版本计数器。 

给定一个映射：

```py
class Node(Base):
    __tablename__ = "node"
    id = Column(Integer, primary_key=True)
    version_id = Column(Integer, default=0)
    parent_id = Column(ForeignKey("node.id"))
    favorite_node_id = Column(ForeignKey("node.id"))

    nodes = relationship("Node", primaryjoin=remote(parent_id) == id)
    favorite_node = relationship(
        "Node", primaryjoin=favorite_node_id == remote(id), post_update=True
    )

    __mapper_args__ = {"version_id_col": version_id}
```

将另一个节点关联为“favorite”的节点进行更新将会增加版本计数器并匹配当前版本：

```py
node = Node()
session.add(node)
session.commit()  # node is now version #1

node = session.query(Node).get(node.id)
node.favorite_node = Node()
session.commit()  # node is now version #2
```

请注意，这意味着对象在响应其他属性更改而接收到 UPDATE，并在 post_update 关系更改导致第二个 UPDATE 时，现在将收到**一次刷新两次版本计数器更新**。但是，如果对象在当前刷新中接受到 INSERT，则版本计数器 **不会** 再次增加，除非存在服务器端版本控制方案。

现在讨论了 post_update 为何即使是对 UPDATE 也会发出 UPDATE 的原因 为什么 post_update 除了第一个 UPDATE 还会发出 UPDATE？。

另请参阅

指向自身的行 / 相互依赖的行

为什么 post_update 除了第一个 UPDATE 还会发出 UPDATE？

[#3496](https://www.sqlalchemy.org/trac/ticket/3496)

## 关键行为变化 - 核心

### 自定义运算符的类型行为已经统一

可以使用 `Operators.op()` 函数动态创建用户定义的运算符。以前，表达式对此类运算符的类型行为不一致，也无法控制。

而在 1.1 版本中，类似以下表达式将产生无返回类型的结果（假设 `-%>` 是数据库支持的某个特殊运算符）：

```py
>>> column("x", types.DateTime).op("-%>")(None).type
NullType()
```

其他类型将使用使用左侧类型作为返回类型的默认行为：

```py
>>> column("x", types.String(50)).op("-%>")(None).type
String(length=50)
```

这些行为大多是偶然发生的，因此行为已与第二种形式保持一致，即默认返回类型与左侧表达式相同：

```py
>>> column("x", types.DateTime).op("-%>")(None).type
DateTime()
```

由于大多数用户定义的运算符往往是“比较”运算符，通常是由 PostgreSQL 定义的许多特殊运算符之一，`Operators.op.is_comparison` 标志已修复，以遵循其允许返回类型为 `Boolean` 的文档行为，包括对 `ARRAY` 和 `JSON`：

```py
>>> column("x", types.String(50)).op("-%>", is_comparison=True)(None).type
Boolean()
>>> column("x", types.ARRAY(types.Integer)).op("-%>", is_comparison=True)(None).type
Boolean()
>>> column("x", types.JSON()).op("-%>", is_comparison=True)(None).type
Boolean()
```

为了辅助布尔比较运算符，添加了一个新的简写方法 `Operators.bool_op()`。应优先使用此方法进行即时布尔运算符：

```py
>>> print(column("x", types.Integer).bool_op("-%>")(5))
x  -%>  :x_1 
```  ### `literal_column()` 中的百分号现在有条件地转义

`literal_column` 结构现在根据使用的 DBAPI 是否使用了对百分号敏感的参数样式有条件地转义百分号字符（例如‘format’或‘pyformat’）。

以前，无法生成一个声明单个百分号的 `literal_column` 结构：

```py
>>> from sqlalchemy import literal_column
>>> print(literal_column("some%symbol"))
some%%symbol 
```

对于未设置为使用‘format’或‘pyformat’参数样式的方言，百分号现在不受影响；大多数 MySQL 方言等声明了其中一个参数样式的方言将继续适当转义：

```py
>>> from sqlalchemy import literal_column
>>> print(literal_column("some%symbol"))
some%symbol
>>> from sqlalchemy.dialects import mysql
>>> print(literal_column("some%symbol").compile(dialect=mysql.dialect()))
some%%symbol 
```

作为这一变化的一部分，使用诸如 `ColumnOperators.contains()`、`ColumnOperators.startswith()` 和 `ColumnOperators.endswith()` 等运算符时，之前出现的加倍现象也被精细化，只在适当时发生。

[#3740](https://www.sqlalchemy.org/trac/ticket/3740)  ### 列级别的 COLLATE 关键字现在引用排序规则名称

`collate()` 和 `ColumnOperators.collate()` 函数中的一个 bug，用于在语句级别提供临时列排序，已修复，其中一个大小写敏感的名称不会被引用：

```py
stmt = select([mytable.c.x, mytable.c.y]).order_by(
    mytable.c.somecolumn.collate("fr_FR")
)
```

现在显示为：

```py
SELECT  mytable.x,  mytable.y,
FROM  mytable  ORDER  BY  mytable.somecolumn  COLLATE  "fr_FR"
```

之前，大小写敏感的名称“fr_FR”不会被引用。目前，手动引用“fr_FR”名称**不**会被检测到，因此手动引用标识符的应用程序应进行调整。请注意，此更改不会影响类型级别的排序的使用（例如，在表级别上指定在 `String` 类型上的排序），其中已经应用了引用。

[#3785](https://www.sqlalchemy.org/trac/ticket/3785)  ### 自定义运算符的输入行为已经保持一致

用户定义的运算符可以使用 `Operators.op()` 函数动态创建。先前，针对此类运算符的表达式的输入行为是不一致的，也无法控制。

在 1.1 中，例如以下表达式将产生一个没有返回类型的结果（假设 `-%>` 是数据库支持的某种特殊运算符）：

```py
>>> column("x", types.DateTime).op("-%>")(None).type
NullType()
```

其他类型将使用默认行为，即使用左侧类型作为返回类型：

```py
>>> column("x", types.String(50)).op("-%>")(None).type
String(length=50)
```

这些行为大多是偶然发生的，因此行为已与第二种形式保持一致，即默认返回类型与左侧表达式相同：

```py
>>> column("x", types.DateTime).op("-%>")(None).type
DateTime()
```

由于大多数用户定义的运算符倾向于是“比较”运算符，通常是 PostgreSQL 定义的许多特殊运算符之一，因此已修复了 `Operators.op.is_comparison` 标志，使其遵循其文档化的行为，即在所有情况下允许返回类型为 `Boolean`，包括对于 `ARRAY` 和 `JSON`：

```py
>>> column("x", types.String(50)).op("-%>", is_comparison=True)(None).type
Boolean()
>>> column("x", types.ARRAY(types.Integer)).op("-%>", is_comparison=True)(None).type
Boolean()
>>> column("x", types.JSON()).op("-%>", is_comparison=True)(None).type
Boolean()
```

为了辅助布尔比较运算符，添加了一个新的简写方法 `Operators.bool_op()`。应优先使用此方法进行即时布尔运算：

```py
>>> print(column("x", types.Integer).bool_op("-%>")(5))
x  -%>  :x_1 
```

### `literal_column()` 中的百分号现在有条件地转义

`literal_column` 构造现在有条件地转义百分号字符，取决于正在使用的 DBAPI 是否使用了对百分号敏感的参数样式（例如‘format’或‘pyformat’）。

以前，无法生成一个声明了单个百分号的`literal_column` 构造：

```py
>>> from sqlalchemy import literal_column
>>> print(literal_column("some%symbol"))
some%%symbol 
```

对于未设置为使用‘format’或‘pyformat’参数样式的方言，百分号现在不受影响；大多数 MySQL 方言等声明了这些参数样式的方言将继续适当地进行转义：

```py
>>> from sqlalchemy import literal_column
>>> print(literal_column("some%symbol"))
some%symbol
>>> from sqlalchemy.dialects import mysql
>>> print(literal_column("some%symbol").compile(dialect=mysql.dialect()))
some%%symbol 
```

作为这一变化的一部分，使用`ColumnOperators.contains()`、`ColumnOperators.startswith()`和`ColumnOperators.endswith()`等操作符时，之前出现的加倍现象也被优化为仅在适当时发生。

[#3740](https://www.sqlalchemy.org/trac/ticket/3740)

### 列级别的 COLLATE 关键字现在引用了排序规则名称

修复了`collate()`和`ColumnOperators.collate()`函数中的一个错误，用于在语句级别提供临时列排序规则，其中一个区分大小写的名称将不会被引用：

```py
stmt = select([mytable.c.x, mytable.c.y]).order_by(
    mytable.c.somecolumn.collate("fr_FR")
)
```

现在呈现为：

```py
SELECT  mytable.x,  mytable.y,
FROM  mytable  ORDER  BY  mytable.somecolumn  COLLATE  "fr_FR"
```

以前，区分大小写的名称“fr_FR”将不会被引用。目前，不会检测手动引用“fr_FR”名称，因此手动引用标识符的应用程序应进行调整。请注意，此更改不影响在类型级别使用排序规则（例如在数据类型上指定的`String`在表级别），其中已经应用了引用。

[#3785](https://www.sqlalchemy.org/trac/ticket/3785)

## 方言改进和变更 - PostgreSQL

### 支持批处理模式 / 快速执行助手

psycopg2 的 `cursor.executemany()` 方法被认为性能较差，特别是在 INSERT 语句中。为了缓解这一问题，psycopg2 添加了[快速执行助手](https://www.psycopg.org/docs/extras.html#fast-execution-helpers)，通过批量发送多个 DML 语句来减少服务器往返次数。SQLAlchemy 1.2 现在包括对这些助手的支持，可以在`Engine` 使用 `cursor.executemany()` 对多个参数集合执行语句时透明地使用。该功能默认关闭，可以通过在 `create_engine()` 上使用 `use_batch_mode` 参数来启用：

```py
engine = create_engine(
    "postgresql+psycopg2://scott:tiger@host/dbname", use_batch_mode=True
)
```

目前该功能被视为实验性质，但可能在未来的版本中默认开启。

另请参阅

Psycopg2 快速执行助手

[#4109](https://www.sqlalchemy.org/trac/ticket/4109)  ### 支持 INTERVAL 中字段规范的支持，包括完整反射

PostgreSQL 的 INTERVAL 数据类型中的“fields”规范允许指定要存储的间隔的哪些字段，包括“YEAR”、“MONTH”、“YEAR TO MONTH”等值。`INTERVAL` 数据类型现在允许指定这些值：

```py
from sqlalchemy.dialects.postgresql import INTERVAL

Table("my_table", metadata, Column("some_interval", INTERVAL(fields="DAY TO SECOND")))
```

此外，所有 INTERVAL 数据类型现在都可以独立于存在的“fields”规范进行反射；数据类型本身的“fields”参数也将存在：

```py
>>> inspect(engine).get_columns("my_table")
[{'comment': None,
 'name': u'some_interval', 'nullable': True,
 'default': None, 'autoincrement': False,
 'type': INTERVAL(fields=u'day to second')}]
```

[#3959](https://www.sqlalchemy.org/trac/ticket/3959)  ### 支持批处理模式 / 快速执行助手

psycopg2 的 `cursor.executemany()` 方法被认为性能较差，特别是在 INSERT 语句中。为了缓解这一问题，psycopg2 添加了[快速执行助手](https://www.psycopg.org/docs/extras.html#fast-execution-helpers)，通过批量发送多个 DML 语句来减少服务器往返次数。SQLAlchemy 1.2 现在包括对这些助手的支持，可以在`Engine` 使用 `cursor.executemany()` 对多个参数集合执行语句时透明地使用。该功能默认关闭，可以通过在 `create_engine()` 上使用 `use_batch_mode` 参数来启用：

```py
engine = create_engine(
    "postgresql+psycopg2://scott:tiger@host/dbname", use_batch_mode=True
)
```

目前该功能被视为实验性质，但可能在未来的版本中默认开启。

另请参阅

Psycopg2 快速执行助手

[#4109](https://www.sqlalchemy.org/trac/ticket/4109)

### 支持 INTERVAL 中字段规范的支持，包括完整反射

PostgreSQL 的 INTERVAL 数据类型中的“fields”指定符允许指定要存储的间隔的哪些字段，包括“YEAR”、“MONTH”、“YEAR TO MONTH”等值。`INTERVAL` 数据类型现在允许指定这些值：

```py
from sqlalchemy.dialects.postgresql import INTERVAL

Table("my_table", metadata, Column("some_interval", INTERVAL(fields="DAY TO SECOND")))
```

此外，现在可以独立于“fields”指定符反映所有 INTERVAL 数据类型；数据类型本身的“fields”参数也将存在：

```py
>>> inspect(engine).get_columns("my_table")
[{'comment': None,
 'name': u'some_interval', 'nullable': True,
 'default': None, 'autoincrement': False,
 'type': INTERVAL(fields=u'day to second')}]
```

[#3959](https://www.sqlalchemy.org/trac/ticket/3959)

## 方言改进和变更 - MySQL

### 支持 INSERT..ON DUPLICATE KEY UPDATE

MySQL 支持的 `INSERT` 的 `ON DUPLICATE KEY UPDATE` 子句现在可以使用 MySQL 特定版本的 `Insert` 对象来支持，通过 `sqlalchemy.dialects.mysql.dml.insert()`。这个 `Insert` 子类添加了一个新方法 `Insert.on_duplicate_key_update()`，实现了 MySQL 的语法：

```py
from sqlalchemy.dialects.mysql import insert

insert_stmt = insert(my_table).values(id="some_id", data="some data to insert")

on_conflict_stmt = insert_stmt.on_duplicate_key_update(
    data=insert_stmt.inserted.data, status="U"
)

conn.execute(on_conflict_stmt)
```

以上内容将呈现为：

```py
INSERT  INTO  my_table  (id,  data)
VALUES  (:id,  :data)
ON  DUPLICATE  KEY  UPDATE  data=VALUES(data),  status=:status_1
```

另请参阅

INSERT…ON DUPLICATE KEY UPDATE (Upsert)

[#4009](https://www.sqlalchemy.org/trac/ticket/4009)  ### 支持 INSERT..ON DUPLICATE KEY UPDATE

MySQL 支持的 `INSERT` 的 `ON DUPLICATE KEY UPDATE` 子句现在可以使用 MySQL 特定版本的 `Insert` 对象来支持，通过 `sqlalchemy.dialects.mysql.dml.insert()`。这个 `Insert` 子类添加了一个新方法 `Insert.on_duplicate_key_update()`，实现了 MySQL 的语法：

```py
from sqlalchemy.dialects.mysql import insert

insert_stmt = insert(my_table).values(id="some_id", data="some data to insert")

on_conflict_stmt = insert_stmt.on_duplicate_key_update(
    data=insert_stmt.inserted.data, status="U"
)

conn.execute(on_conflict_stmt)
```

以上内容将呈现为：

```py
INSERT  INTO  my_table  (id,  data)
VALUES  (:id,  :data)
ON  DUPLICATE  KEY  UPDATE  data=VALUES(data),  status=:status_1
```

另请参阅

INSERT…ON DUPLICATE KEY UPDATE (Upsert)

[#4009](https://www.sqlalchemy.org/trac/ticket/4009)

## 方言改进和变更 - Oracle

### cx_Oracle 方言、类型系统的重大重构

随着 cx_Oracle DBAPI 的 6.x 系列的推出，SQLAlchemy 的 cx_Oracle 方言已经重新设计和简化，以利用 cx_Oracle 的最新改进，并放弃了在 cx_Oracle 的 5.x 系列之前更相关的模式的支持。

+   支持的最低 cx_Oracle 版本现在是 5.1.3；推荐使用 5.3 或最新的 6.x 系列。

+   数据类型的处理已经重构。`cursor.setinputsizes()` 方法不再用于除 LOB 类型之外的任何数据类型，根据 cx_Oracle 的开发人员的建议。因此，参数 `auto_setinputsizes` 和 `exclude_setinputsizes` 已被弃用，不再起作用。

+   当 `coerce_to_decimal` 标志设置为 False 时，表示不应发生对具有精度和标度的数值类型进行到 `Decimal` 的强制转换，仅影响未定义类型（例如，没有 `TypeEngine` 对象的普通字符串）的语句。现在，包含 `Numeric` 类型或子类型的 Core 表达式将遵循该类型的十进制强制转换规则。

+   “两阶段”事务支持在方言中已经在 cx_Oracle 的 6.x 系列中被删除，现在已完全移除，因为这个功能从未正确工作过，也不太可能被用于生产环境。因此，`allow_twophase` 方言标志已被弃用，也不再起作用。

+   修复了涉及 RETURNING 中存在的列键的错误。给定如下语句：

    ```py
    result = conn.execute(table.insert().values(x=5).returning(table.c.a, table.c.b))
    ```

    以前，结果中每行的键是 `ret_0` 和 `ret_1`，这些是 cx_Oracle RETURNING 实现内部的标识符。现在，键将是 `a` 和 `b`，这是其他方言所期望的。

+   cx_Oracle 的 LOB 数据类型将返回值表示为 `cx_Oracle.LOB` 对象，这是一个与游标关联的代理，通过 `.read()` 方法返回最终数据值。在历史上，如果在这些 LOB 对象被消耗之前读取了更多行（特别是在读取了比 cursor.arraysize 值更多的行，导致读取了一批新行），这些 LOB 对象会引发错误“在后续获取后 LOB 变量不再有效”。SQLAlchemy 通过在其类型系统内部自动调用 `.read()`，以及使用一个特殊的 `BufferedColumnResultSet` 来解决这个问题，该对象将确保在使用 `cursor.fetchmany()` 或 `cursor.fetchall()` 这样的调用时，这些数据被缓冲。

    方言现在使用 cx_Oracle outputtypehandler 来处理这些 `.read()` 调用，以便无论获取多少行，它们始终被立即调用，因此不再会发生此错误。因此，`BufferedColumnResultSet` 的使用，以及一些其他特定于此用例的 Core `ResultSet` 内部机制已被移除。类型对象也变得更简化，因为它们不再需要处理二进制列结果。

    另外，cx_Oracle 6.x 已经删除了此错误发生的条件，因此不再可能发生此错误。在 SQLAlchemy 中，如果很少（如果有的话）使用了 `auto_convert_lobs=False` 选项，并且在 LOB 对象可以被消耗之前读取了更多行，则可能会发生错误。升级到 cx_Oracle 6.x 将解决这个问题。### Oracle 唯一性、检查约束现在已反映。

UNIQUE 和 CHECK 约束现在通过 `Inspector.get_unique_constraints()` 和 `Inspector.get_check_constraints()` 反映出来。反映的 `Table` 对象现在也将包括 `CheckConstraint` 对象。有关此处行为怪癖的信息，请参阅 约束反射，包括大多数 `Table` 对象仍不会包括任何 `UniqueConstraint` 对象，因为这些通常通过 `Index` 表示。

另请参见

约束反射

[#4003](https://www.sqlalchemy.org/trac/ticket/4003)  ### Oracle 外键约束名称现在是“名称标准化”

在表反射期间传递给 `ForeignKeyConstraint` 对象的外键约束名称以及在 `Inspector.get_foreign_keys()` 方法中将会“名称标准化”，即以小写形式表示以进行大小写不敏感的命名，而不是 Oracle 使用的原始大写格式：

```py
>>> insp.get_indexes("addresses")
[{'unique': False, 'column_names': [u'user_id'],
 'name': u'address_idx', 'dialect_options': {}}]

>>> insp.get_pk_constraint("addresses")
{'name': u'pk_cons', 'constrained_columns': [u'id']}

>>> insp.get_foreign_keys("addresses")
[{'referred_table': u'users', 'referred_columns': [u'id'],
 'referred_schema': None, 'name': u'user_id_fk',
 'constrained_columns': [u'user_id']}]
```

以前，外键约束的结果看起来像：

```py
[
    {
        "referred_table": "users",
        "referred_columns": ["id"],
        "referred_schema": None,
        "name": "USER_ID_FK",
        "constrained_columns": ["user_id"],
    }
]
```

上述内容可能会在特别是与 Alembic autogenerate 一起创建问题。

[#3276](https://www.sqlalchemy.org/trac/ticket/3276)  ### cx_Oracle 方言、类型系统的重大重构

随着 cx_Oracle DBAPI 推出的 6.x 系列，SQLAlchemy 的 cx_Oracle 方言已经进行了重构和简化，以利用 cx_Oracle 的最新改进，并放弃了在 cx_Oracle 5.x 系列之前更相关的模式的支持。

+   支持的最低 cx_Oracle 版本现在是 5.1.3；建议使用 5.3 或最新的 6.x 系列。

+   数据类型的处理已经重构。`cursor.setinputsizes()` 方法现在仅用于 LOB 类型，根据 cx_Oracle 的开发人员的建议。因此，参数 `auto_setinputsizes` 和 `exclude_setinputsizes` 已被弃用，不再起作用。

+   当设置为 False 时，`coerce_to_decimal` 标志表示不应进行具有精度和标度的数字类型到 `Decimal` 的强制转换，仅影响未类型化的（例如，没有 `TypeEngine` 对象的普通字符串）语句。包含 `Numeric` 类型或子类型的 Core 表达式现在将遵循该类型的十进制强制转换规则。

+   方言中的“两阶段”事务支持已经在 cx_Oracle 的 6.x 系列中删除，因为这个功能从未正确工作过，并且不太可能已经投入生产使用。因此，`allow_twophase` 方言标志已被弃用，也不再起作用。

+   修复了涉及 RETURNING 的列键存在的错误。给定以下语句：

    ```py
    result = conn.execute(table.insert().values(x=5).returning(table.c.a, table.c.b))
    ```

    以前，结果中每行的键是 `ret_0` 和 `ret_1`，这是 cx_Oracle RETURNING 实现的内部标识符。现在，键将是 `a` 和 `b`，这是其他方言所期望的。

+   cx_Oracle 的 LOB 数据类型将返回值表示为 `cx_Oracle.LOB` 对象，它是一个与游标关联的代理，通过 `.read()` 方法返回最终数据值。历史上，如果在这些 LOB 对象被消耗之前读取了更多的行（具体来说，如果读取的行数超过了 cursor.arraysize 的值，这会导致读取一批新的行），这些 LOB 对象将引发错误“在后续获取后 LOB 变量不再有效”。SQLAlchemy 解决了这个问题，通过在其类型系统中自动调用 `.read()` 来处理这些 LOB，并使用特殊的 `BufferedColumnResultSet` 确保这些数据被缓冲，以防使用了 `cursor.fetchmany()` 或 `cursor.fetchall()` 这样的调用。

    方言现在使用 cx_Oracle 的 outputtypehandler 来处理这些 `.read()` 调用，以便无论读取多少行，它们都始终被提前调用，因此不再会发生此错误。因此，已删除了 `BufferedColumnResultSet` 的使用，以及一些特定于此用例的 Core `ResultSet` 的其他内部内容。由于不再需要处理二进制列结果，类型对象也变得简化了。

    另外，cx_Oracle 6.x 已经删除了此错误在任何情况下发生的条件，因此该错误不再可能发生。在 SQLAlchemy 中，该错误可能发生在很少（如果有的话）使用了 `auto_convert_lobs=False` 选项，并且与之前的 cx_Oracle 5.x 系列一起使用，以及在 LOB 对象可以被消耗之前读取了更多行的情况下。升级到 cx_Oracle 6.x 将解决该问题。

### Oracle 唯一性、检查约束现在已反映

唯一约束和检查约束现在通过 `Inspector.get_unique_constraints()` 和 `Inspector.get_check_constraints()` 反映出来。一个反映的 `Table` 对象现在也将包括 `CheckConstraint` 对象。请参阅 Constraint Reflection 中的注意事项，了解这里的行为怪癖，包括大多数 `Table` 对象仍将不包括任何 `UniqueConstraint` 对象，因为这些通常通过 `Index` 表示。

另请参阅

Constraint Reflection

[#4003](https://www.sqlalchemy.org/trac/ticket/4003)

### Oracle 外键约束名称现在已经“名称标准化”

在表反射期间传递给 `ForeignKeyConstraint` 对象以及在 `Inspector.get_foreign_keys()` 方法内部的外键约束的名称现在将被“名称标准化”，即，以小写形式表示以便于不区分大小写的名称，而不是 Oracle 使用的原始大写格式：

```py
>>> insp.get_indexes("addresses")
[{'unique': False, 'column_names': [u'user_id'],
 'name': u'address_idx', 'dialect_options': {}}]

>>> insp.get_pk_constraint("addresses")
{'name': u'pk_cons', 'constrained_columns': [u'id']}

>>> insp.get_foreign_keys("addresses")
[{'referred_table': u'users', 'referred_columns': [u'id'],
 'referred_schema': None, 'name': u'user_id_fk',
 'constrained_columns': [u'user_id']}]
```

以前，外键结果看起来像是：

```py
[
    {
        "referred_table": "users",
        "referred_columns": ["id"],
        "referred_schema": None,
        "name": "USER_ID_FK",
        "constrained_columns": ["user_id"],
    }
]
```

上述情况可能会特别在 Alembic autogenerate 方面造成问题。

[#3276](https://www.sqlalchemy.org/trac/ticket/3276)

## 方言改进和更改 - SQL Server

### 支持具有嵌入点的 SQL Server 架构名称

SQL Server 方言有一种行为，即假定带有点的架构名称是“数据库”。“所有者”标识符对，这在表和组件反射操作以及在呈现架构名称的引用时必须在这些单独的组件之间进行拆分，以使这两个符号分别引用。现在可以使用方括号传递架构参数以手动指定此拆分发生的位置，允许包含一个或多个点的数据库和/或所有者名称：

```py
Table("some_table", metadata, Column("q", String(50)), schema="[MyDataBase.dbo]")
```

上表将考虑“owner”为`MyDataBase.dbo`，在呈现时也将被引用，并且“database”为 None。要分别引用数据库名称和所有者，请使用两对括号：

```py
Table(
    "some_table",
    metadata,
    Column("q", String(50)),
    schema="[MyDataBase.SomeDB].[MyDB.owner]",
)
```

此外，当传递给 SQL Server 方言的“schema”时，现在会尊重`quoted_name`构造；如果引号标志为 True，则给定的符号不会在点上拆分，并且将被解释为“owner”。

另请参见

多部分模式名称

[#2626](https://www.sqlalchemy.org/trac/ticket/2626)

### 支持 AUTOCOMMIT 隔离级别

PyODBC 和 pymssql 方言现在都支持由`Connection.execution_options()`设置的“AUTOCOMMIT”隔离级别，这将在 DBAPI 连接对象上建立正确的标志。

### 支持带有嵌入点的 SQL Server 模式名称

SQL Server 方言具有这样的行为，即假定具有其中的点的模式名称是“数据库”。“所有者”标识符对，这在表和组件反射操作以及在呈现模式名称的引用时必须将这两个符号分开时发生，以便分别引用这两个符号。现在可以使用括号传递模式参数以手动指定此拆分发生的位置，允许数据库和/或所有者名称本身包含一个或多个点：

```py
Table("some_table", metadata, Column("q", String(50)), schema="[MyDataBase.dbo]")
```

上表将考虑“owner”为`MyDataBase.dbo`，在呈现时也将被引用，并且“database”为 None。要分别引用数据库名称和所有者，请使用两对括号：

```py
Table(
    "some_table",
    metadata,
    Column("q", String(50)),
    schema="[MyDataBase.SomeDB].[MyDB.owner]",
)
```

此外，当传递给 SQL Server 方言的“schema”时，现在会尊重`quoted_name`构造；如果引号标志为 True，则给定的符号不会在点上拆分，并且将被解释为“owner”。

另请参见

多部分模式名称

[#2626](https://www.sqlalchemy.org/trac/ticket/2626)

### 支持 AUTOCOMMIT 隔离级别

PyODBC 和 pymssql 方言现在都支持由`Connection.execution_options()`设置的“AUTOCOMMIT”隔离级别，这将在 DBAPI 连接对象上建立正确的标志。
