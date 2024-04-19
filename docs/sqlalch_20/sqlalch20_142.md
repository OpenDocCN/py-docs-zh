# SQLAlchemy 1.3 有什么新功能？

> 原文：[`docs.sqlalchemy.org/en/20/changelog/migration_13.html`](https://docs.sqlalchemy.org/en/20/changelog/migration_13.html)

关于本文档

本文描述了 SQLAlchemy 版本 1.2 和 SQLAlchemy 版本 1.3 之间的更改。

## 介绍

本指南介绍了 SQLAlchemy 版本 1.3 中的新功能，还记录了影响用户将其应用程序从 SQLAlchemy 1.2 系列迁移到 1.3 的更改。

请仔细查看行为变化部分，可能会有不兼容的行为变化。

## 通用

### 对所有弃用元素发出弃用警告；新增弃用项

发行版 1.3 确保所有被弃用的行为和 API，包括那些长期被列为“遗留”的，都会发出`DeprecationWarning`警告。这包括在使用参数时，比如`Session.weak_identity_map`和类似`MapperExtension`的情况。虽然所有弃用情况都已在文档中记录，但通常它们没有使用正确的重构文本指令，或者包含它们被弃用的版本。特定 API 功能是否实际发出弃用警告并不一致。一般的态度是，大多数或所有这些弃用功能都被视为长期遗留功能，没有计划删除它们。

这一变化包括，所有记录的弃用现在在文档中使用了正确的重构文本指令，并附有版本号，明确说明该功能或用例将在将来的版本中被移除（例如，不再有永久遗留用例），并且使用任何此类功能或用例将明确发出`DeprecationWarning`，在 Python 3 中以及使用现代测试工具如 Pytest 时，现在在标准错误流中更加明确。目标是，这些长期被弃用的功能，甚至可以追溯到版本 0.7 或 0.6，应该开始被完全移除，而不是将它们保留为“遗留”功能。此外，从版本 1.3 开始，还添加了一些重大的新弃用项。由于 SQLAlchemy 已经被成千上万的开发人员实际使用了 14 年，可以指出一个混合得很好的用例流，以及修剪掉与这种单一工作方式相悖的功能和模式。

更大的背景是，SQLAlchemy 致力于适应即将到来的仅支持 Python 3 的世界，以及类型注释的世界，为此，SQLAlchemy 有**暂定**计划进行一项重大重构，希望大大减少 API 的认知负担，并对 Core 和 ORM 之间的许多实现和使用差异进行重大调整。由于这两个系统在 SQLAlchemy 首次发布后发生了巨大变化，特别是 ORM 仍然保留着许多“外挂”行为，使得 Core 和 ORM 之间的隔离墙过高。通过提前将 API 集中在每个支持的用例的单一模式上，将来迁移到显著改变的 API 的工作变得更简单。

有关 1.3 版本中添加的最重要的弃用功能，请参见下面的链接部分。

另请参阅

“threadlocal”引擎策略已弃用

convert_unicode 参数已弃用

AliasedClass 与非主映射器的关系取代了 

[#4393](https://www.sqlalchemy.org/trac/ticket/4393)

## 新功能和改进 - ORM

### AliasedClass 与非主映射器的关系取代了

“非主映射器”是以 Imperative Mapping 风格创建的`Mapper`，它充当已经映射的类的附加映射器，针对不同类型的可选择项。非主映射器起源于 SQLAlchemy 的 0.1、0.2 系列，当时预期`Mapper`对象将是主要的查询构造接口，而`Query`对象尚不存在。

随着`Query`的出现，以及后来的`AliasedClass`构造，大多数非主映射器的用例都消失了。这是一件好事，因为 SQLAlchemy 在 0.5 系列左右也完全摆脱了“经典”映射，转而采用了声明式系统。

当意识到一些非常难以定义的`relationship()`配置可能成为可能时，保留了一个非主映射器的用例，当一个具有替代可选择项的非主映射器被作为映射目标时，而不是尝试构建一个涵盖特定对象间关系所有复杂性的`relationship.primaryjoin`。

随着这种用例变得更加流行，它的局限性变得明显，包括非主映射器难以配置到可选择添加新列的可选项上，映射器不继承原始映射的关系，显式配置在非主映射器上的关系与加载器选项不兼容，非主映射器也没有提供可用于查询的基于列的属性的完全功能命名空间（在旧的 0.1 - 0.4 版本中，人们会直接使用`Table`对象与 ORM 一起使用）。

缺失的部分是允许`relationship()`直接引用`AliasedClass`。`AliasedClass`已经做了我们希望非主映射器做的一切；它允许从替代可选择项加载现有映射类，继承现有映射器的所有属性和关系，与加载器选项非常配合，提供一个类似类的对象，可以像类本身一样混入查询中。通过这种改变，以前针对非主映射器的配方在配置关系连接方式中被更改为别名类。

在关系到别名类时，原始的非主映射器看起来像：

```py
j = join(B, D, D.b_id == B.id).join(C, C.id == D.c_id)

B_viacd = mapper(
    B,
    j,
    non_primary=True,
    primary_key=[j.c.b_id],
    properties={
        "id": j.c.b_id,  # so that 'id' looks the same as before
        "c_id": j.c.c_id,  # needed for disambiguation
        "d_c_id": j.c.d_c_id,  # needed for disambiguation
        "b_id": [j.c.b_id, j.c.d_b_id],
        "d_id": j.c.d_id,
    },
)

A.b = relationship(B_viacd, primaryjoin=A.b_id == B_viacd.c.b_id)
```

这些属性是必要的，以便重新映射额外的列，使其不与映射到`B`的现有列发生冲突，同时也需要定义一个新的主键。

使用新方法，所有这些冗长的内容都消失了，并且在建立关系时直接引用了额外的列：

```py
j = join(B, D, D.b_id == B.id).join(C, C.id == D.c_id)

B_viacd = aliased(B, j, flat=True)

A.b = relationship(B_viacd, primaryjoin=A.b_id == j.c.b_id)
```

非主映射器现在已被弃用，最终目标是使经典映射作为一项功能完全消失。声明式 API 将成为映射的唯一手段，这希望能够实现内部改进和简化，以及更清晰的文档故事。

[#4423](https://www.sqlalchemy.org/trac/ticket/4423)  ### selectin 加载不再对简单的一对多使用 JOIN。

在 1.2 版本中添加的“selectin”加载功能引入了一种极其高效的新方法来急切加载集合，在许多情况下比“subquery”急切加载要快得多，因为它不依赖于重新声明原始 SELECT 查询，而是使用一个简单的 IN 子句。然而，“selectin”加载仍然依赖于在父表和相关表之间渲染 JOIN，因为它需要父表主键值在行中以匹配行。在 1.3 中，添加了一种新的优化，将在简单的一对多加载的最常见情况下省略此 JOIN，其中相关行已经包含了父行的主键值，表达在其外键列中。这再次提供了显著的性能改进，因为 ORM 现在可以在一个查询中加载大量集合，而根本不使用 JOIN 或子查询。

给定一个映射：

```py
class A(Base):
    __tablename__ = "a"

    id = Column(Integer, primary_key=True)
    bs = relationship("B", lazy="selectin")

class B(Base):
    __tablename__ = "b"
    id = Column(Integer, primary_key=True)
    a_id = Column(ForeignKey("a.id"))
```

在“selectin”加载的 1.2 版本中，A 到 B 的加载如下：

```py
SELECT  a.id  AS  a_id  FROM  a
SELECT  a_1.id  AS  a_1_id,  b.id  AS  b_id,  b.a_id  AS  b_a_id
FROM  a  AS  a_1  JOIN  b  ON  a_1.id  =  b.a_id
WHERE  a_1.id  IN  (?,  ?,  ?,  ?,  ?,  ?,  ?,  ?,  ?,  ?)  ORDER  BY  a_1.id
(1,  2,  3,  4,  5,  6,  7,  8,  9,  10)
```

使用新行为，加载如下：

```py
SELECT  a.id  AS  a_id  FROM  a
SELECT  b.a_id  AS  b_a_id,  b.id  AS  b_id  FROM  b
WHERE  b.a_id  IN  (?,  ?,  ?,  ?,  ?,  ?,  ?,  ?,  ?,  ?)  ORDER  BY  b.a_id
(1,  2,  3,  4,  5,  6,  7,  8,  9,  10)
```

该行为被释放为自动的，使用类似于延迟加载使用的启发式方法，以确定是否可以直接从标识映射中获取相关实体。然而，与大多数查询功能一样，由于涉及多态加载的高级场景，该功能的实现变得更加复杂。如果遇到问题，用户应该报告错误，但更改还包括一个标志`relationship.omit_join`，可以在`relationship()`上设置为`False`以禁用优化。

[#4340](https://www.sqlalchemy.org/trac/ticket/4340)  ### 改进多对一查询表达式的行为

当构建一个将多对一关系与对象值进行比较的查询时，例如：

```py
u1 = session.query(User).get(5)

query = session.query(Address).filter(Address.user == u1)
```

上述表达式`Address.user == u1`，最终编译为基于`User`对象的主键列的 SQL 表达式，如`"address.user_id = 5"`，使用延迟可调用以在绑定表达式中尽可能晚地检索值`5`。这是为了适应`Address.user == u1`表达式可能针对尚未刷新的`User`对象的用例，该对象依赖于服务器生成的主键值，以及该表达式始终返回正确结果的情况，即使自创建表达式以来`u1`的主键值已更改。

然而，这种行为的一个副作用是，如果在评估表达式时`u1`最终过期，它将导致额外的 SELECT 语句，并且如果`u1`也从`Session`中分离，它将引发错误：

```py
u1 = session.query(User).get(5)

query = session.query(Address).filter(Address.user == u1)

session.expire(u1)
session.expunge(u1)

query.all()  # <-- would raise DetachedInstanceError
```

对象的过期/清除可以在`Session`提交时隐式发生，并且`u1`实例超出范围时，因为`Address.user == u1`表达式并不强烈引用对象本身，只引用其`InstanceState`。

修复的方法是允许`Address.user == u1`表达式根据尝试在表达式编译时正常检索或加载值来评估值`5`，就像现在一样，但如果对象已分离并已过期，则从`InstanceState`上的新机制中检索，该机制将在属性过期时在该状态上记忆该属性的最后已知值。当表达式功能需要时，此机制仅为特定属性/ `InstanceState`启用，以节省性能/内存开销。

最初，尝试了诸如立即评估表达式并尝试稍后加载值的各种简化方法，但困难的边缘情况是正在更改的列属性值（通常是自然主键）。为了确保像`Address.user == u1`这样的表达式始终返回`u1`当前状态的正确答案，它将返回持久对象的当前数据库持久化值，如果需要，通过 SELECT 查询取消过期，并且对于分离对象，它将返回最近已知的值，而不管对象何时使用`InstanceState`中的新功能过期跟踪列属性的最后已知值。

当值无法评估时，现代属性 API 功能用于指示特定的错误消息，这两种情况是当列属性从未设置时，以及当对象在进行第一次评估时已过期并且现在已分离。在所有情况下，不再引发`DetachedInstanceError`。

[#4359](https://www.sqlalchemy.org/trac/ticket/4359)  ### 多对一替换不会对“raiseload”或“old”对象引发异常

在许多对一关系上进行延迟加载以加载“旧”值的情况下，如果关系未指定`relationship.active_history`标志，则不会为分离对象引发断言：

```py
a1 = session.query(Address).filter_by(id=5).one()

session.expunge(a1)

a1.user = some_user
```

在上面的情况下，当在分离的`a1`对象上替换`.user`属性时，将引发`DetachedInstanceError`，因为属性试图从标识映射中检索`.user`的先前值。变化在于，操作现在继续进行而不加载旧值。

相同的更改也适用于`lazy="raise"`加载策略：

```py
class Address(Base):
    # ...

    user = relationship("User", ..., lazy="raise")
```

以前，`a1.user`的关联会触发“raiseload”异常，因为属性试图检索先前的值。在加载“旧”值的情况下，现在跳过此断言。

[#4353](https://www.sqlalchemy.org/trac/ticket/4353)  ### 为 ORM 属性实现了“del”

Python 的`del`操作实际上对于映射属性（标量列或对象引用）并不可用。已添加支持，使其能够正确工作，其中`del`操作大致相当于将属性设置为`None`值：

```py
some_object = session.query(SomeObject).get(5)

del some_object.some_attribute  # from a SQL perspective, works like "= None"
```

[#4354](https://www.sqlalchemy.org/trac/ticket/4354)  ### 在 InstanceState 中添加了 info 字典

将`.info`字典添加到`InstanceState`类中，该对象是通过调用映射对象上的`inspect()`而来。这允许自定义方案添加有关对象的额外信息，这些信息将随着对象在内存中的整个生命周期而传递：

```py
from sqlalchemy import inspect

u1 = User(id=7, name="ed")

inspect(u1).info["user_info"] = "7|ed"
```

[#4257](https://www.sqlalchemy.org/trac/ticket/4257)  ### 水平分片扩展支持批量更新和删除方法

`ShardedQuery`扩展对象支持`Query.update()`和`Query.delete()`批量更新/删除方法。在调用它们时，将咨询`query_chooser`可调用对象，以便根据给定的条件在多个分片上运行更新/删除操作。

[#4196](https://www.sqlalchemy.org/trac/ticket/4196)

### Association Proxy 改进

尽管没有特定原因，但是在本周期内，Association Proxy 扩展进行了许多改进。

#### Association proxy 新增了 cascade_scalar_deletes 标志

给定一个映射如下：

```py
class A(Base):
    __tablename__ = "test_a"
    id = Column(Integer, primary_key=True)
    ab = relationship("AB", backref="a", uselist=False)
    b = association_proxy(
        "ab", "b", creator=lambda b: AB(b=b), cascade_scalar_deletes=True
    )

class B(Base):
    __tablename__ = "test_b"
    id = Column(Integer, primary_key=True)
    ab = relationship("AB", backref="b", cascade="all, delete-orphan")

class AB(Base):
    __tablename__ = "test_ab"
    a_id = Column(Integer, ForeignKey(A.id), primary_key=True)
    b_id = Column(Integer, ForeignKey(B.id), primary_key=True)
```

对`A.b`的赋值将生成一个`AB`对象：

```py
a.b = B()
```

`A.b`关联是标量的，并包括一个新标志`AssociationProxy.cascade_scalar_deletes`。设置时，��`A.b`设置为`None`也将删除`A.ab`。默认行为仍然是保留`a.ab`不变：

```py
a.b = None
assert a.ab is None
```

起初，这种逻辑看起来应该只查看现有关系的“级联”属性，这似乎很直观，但仅凭这一点就不清楚代理对象是否应该被移除，因此行为被作为一个明确的选项提供。

此外，`del` 现在对标量的操作方式与设置为 `None` 相似：

```py
del a.b
assert a.ab is None
```

[#4308](https://www.sqlalchemy.org/trac/ticket/4308)  #### AssociationProxy 在每个类上存储特定于类的状态

`AssociationProxy` 对象基于其关联的父映射类做出许多决策。虽然 `AssociationProxy` 最初只是一个相对简单的‘getter’，但很快就明显地需要做出关于其引用的属性类型的决策——比如标量或集合、映射对象或简单值等。为了实现这一点，它需要检查映射属性或其他引用描述符或属性，如从其父类引用的那样。然而，在 Python 描述符机制中，描述符只有在在其“父”类的上下文中被访问时才会了解其“父”类，比如调用 `MyClass.some_descriptor`，这会调用 `__get__()` 方法，该方法传递类。因此，`AssociationProxy` 对象将存储特定于该类的状态，但只有在调用此方法后才会调用；在未首先将 `AssociationProxy` 作为描述符访问的情况下尝试检查此状态将引发错误。此外，它会假设 `__get__()` 首次看到的类将是唯一需要了解的父类。尽管如果特定类具有继承子类，关联代理实际上是代表不止一个父类工作，即使没有明确重用。即使在存在这种缺陷的情况下，关联代理仍然可以通过其当前行为取得相当大的进展，但在某些情况下仍存在缺陷，以及确定最佳“所有者”类的复杂问题。

现在这些问题已经得到解决，当调用 `__get__()` 时，`AssociationProxy` 不再修改自己的内部状态；相反，每个类都生成了一个名为 `AssociationProxyInstance` 的新对象，它处理了特定于特定映射父类的所有状态（当父类未映射时，不会生成 `AssociationProxyInstance`）。关于关联代理的单一“拥有类”的概念，尽管在 1.1 中有所改进，但基本上已被一种方法所取代，即 AP 现在可以平等地处理任意数量的“拥有”类。

为了适应那些想要检查这种状态的应用程序，而不一定要调用 `__get__()` 的 `AssociationProxy`，添加了一个新方法 `AssociationProxy.for_class()`，提供了直接访问特定于类的 `AssociationProxyInstance`，如下所示：

```py
class User(Base):
    # ...

    keywords = association_proxy("kws", "keyword")

proxy_state = inspect(User).all_orm_descriptors["keywords"].for_class(User)
```

一旦我们有了 `AssociationProxyInstance` 对象，如上例中存储在 `proxy_state` 变量中，我们可以查看特定于 `User.keywords` 代理的属性，例如 `target_class`：

```py
>>> proxy_state.target_class
Keyword
```

[#3423](https://www.sqlalchemy.org/trac/ticket/3423)  #### `AssociationProxy` 现在为基于列的目标提供了标准的列操作符

给定一个 `AssociationProxy`，其中目标是数据库列，并且**不是**对象引用或另一个关联代理：

```py
class User(Base):
    # ...

    elements = relationship("Element")

    # column-based association proxy
    values = association_proxy("elements", "value")

class Element(Base):
    # ...

    value = Column(String)
```

`User.values` 关联代理指的是 `Element.value` 列。现在已经提供了标准的列操作，比如 `like`：

```py
>>> print(s.query(User).filter(User.values.like("%foo%")))
SELECT  "user".id  AS  user_id
FROM  "user"
WHERE  EXISTS  (SELECT  1
FROM  element
WHERE  "user".id  =  element.user_id  AND  element.value  LIKE  :value_1) 
```

`equals`:

```py
>>> print(s.query(User).filter(User.values == "foo"))
SELECT  "user".id  AS  user_id
FROM  "user"
WHERE  EXISTS  (SELECT  1
FROM  element
WHERE  "user".id  =  element.user_id  AND  element.value  =  :value_1) 
```

当与 `None` 比较时，`IS NULL` 表达式被增强，以测试相关行根本不存在；这与以前的行为相同：

```py
>>> print(s.query(User).filter(User.values == None))
SELECT  "user".id  AS  user_id
FROM  "user"
WHERE  (EXISTS  (SELECT  1
FROM  element
WHERE  "user".id  =  element.user_id  AND  element.value  IS  NULL))  OR  NOT  (EXISTS  (SELECT  1
FROM  element
WHERE  "user".id  =  element.user_id)) 
```

注意 `ColumnOperators.contains()` 操作符实际上是一个字符串比较操作符；**这是行为上的变化**，以前，关联代理仅将 `.contains` 用作列表包含操作符。使用列导向的比较，它现在的行为类似于“like”：

```py
>>> print(s.query(User).filter(User.values.contains("foo")))
SELECT  "user".id  AS  user_id
FROM  "user"
WHERE  EXISTS  (SELECT  1
FROM  element
WHERE  "user".id  =  element.user_id  AND  (element.value  LIKE  '%'  ||  :value_1  ||  '%')) 
```

为了测试 `User.values` 集合是否包含值 `"foo"`，应使用等号操作符（例如 `User.values == 'foo'`）；这在以前的版本中也适用。

当使用基于对象的关联代理与集合时，行为与以前相同，即测试集合成员资格，例如，给定一个映射：

```py
class User(Base):
    __tablename__ = "user"

    id = Column(Integer, primary_key=True)
    user_elements = relationship("UserElement")

    # object-based association proxy
    elements = association_proxy("user_elements", "element")

class UserElement(Base):
    __tablename__ = "user_element"

    id = Column(Integer, primary_key=True)
    user_id = Column(ForeignKey("user.id"))
    element_id = Column(ForeignKey("element.id"))
    element = relationship("Element")

class Element(Base):
    __tablename__ = "element"

    id = Column(Integer, primary_key=True)
    value = Column(String)
```

`.contains()` 方法生成与以前相同的表达式，测试 `User.elements` 列表中是否存在 `Element` 对象：

```py
>>> print(s.query(User).filter(User.elements.contains(Element(id=1))))
SELECT "user".id AS user_id
FROM "user"
WHERE EXISTS (SELECT 1
FROM user_element
WHERE "user".id = user_element.user_id AND :param_1 = user_element.element_id)
```

总的来说，这个改变是基于 AssociationProxy stores class-specific state on a per-class basis 的结构性改变而启用的；因为代理现在在生成表达式时衍生了额外的状态，所以有一个对象目标版本和一个列目标版本的`AssociationProxyInstance` 类。

[#4351](https://www.sqlalchemy.org/trac/ticket/4351)

#### 关联代理现在强引用父对象

关联代理集合仅维护对父对象的弱引用的长期行为被还原；代理现在将在代理集合本身也在内存中的情况下维护对父对象的强引用，从而消除“stale association proxy”错误。此更改正在实验性地进行，以查看是否会引起任何副作用。

例如，给定一个具有关联代理的映射：

```py
class A(Base):
    __tablename__ = "a"

    id = Column(Integer, primary_key=True)
    bs = relationship("B")
    b_data = association_proxy("bs", "data")

class B(Base):
    __tablename__ = "b"
    id = Column(Integer, primary_key=True)
    a_id = Column(ForeignKey("a.id"))
    data = Column(String)

a1 = A(bs=[B(data="b1"), B(data="b2")])

b_data = a1.b_data
```

以前，如果 `a1` 超出范围被删除：

```py
del a1
```

在 `a1` 在范围内被删除后尝试迭代 `b_data` 集合会引发错误 `"stale association proxy, parent object has gone out of scope"`。这是因为关联代理需要访问实际的 `a1.bs` 集合以生成视图，在这次更改之前，它仅维护对 `a1` 的弱引用。特别是，用户在执行内联操作时经常会遇到这个错误，例如：

```py
collection = session.query(A).filter_by(id=1).first().b_data
```

以上，因为在 `b_data` 集合实际使用之前，`A` 对象将被垃圾回收。

更改是 `b_data` 集合现在维护对 `a1` 对象的强引用，以使其保持存在：

```py
assert b_data == ["b1", "b2"]
```

此更改引入了一个副作用，即如果应用程序像上面那样传递集合，**父对象在集合被丢弃之前不会被垃圾回收**。一如既往，如果`a1`在特定的`Session`中是持久的，它将保持在该会话的状态中，直到被垃圾回收。

请注意，如果此更改导致问题，可能会对其进行修订。

[#4268](https://www.sqlalchemy.org/trac/ticket/4268)

#### 为集合和关联代理实现了批量替换

将集合或字典分配给关联代理集合现在应该能正常工作了，而以前会为现有键重新创建关联代理成员，导致由于相同对象的删除+插入而导致潜在刷新失败的问题，现在应该只在适当的情况下创建新的关联对象：

```py
class A(Base):
    __tablename__ = "test_a"

    id = Column(Integer, primary_key=True)
    b_rel = relationship(
        "B",
        collection_class=set,
        cascade="all, delete-orphan",
    )
    b = association_proxy("b_rel", "value", creator=lambda x: B(value=x))

class B(Base):
    __tablename__ = "test_b"
    __table_args__ = (UniqueConstraint("a_id", "value"),)

    id = Column(Integer, primary_key=True)
    a_id = Column(Integer, ForeignKey("test_a.id"), nullable=False)
    value = Column(String)

# ...

s = Session(e)
a = A(b={"x", "y", "z"})
s.add(a)
s.commit()

# re-assign where one B should be deleted, one B added, two
# B's maintained
a.b = {"x", "z", "q"}

# only 'q' was added, so only one new B object.  previously
# all three would have been re-created leading to flush conflicts
# against the deleted ones.
assert len(s.new) == 1
```

[#2642](https://www.sqlalchemy.org/trac/ticket/2642)  ### 多对一反向引用在删除操作期间检查集合重复项

当作为 Python 序列存在的 ORM 映射集合，通常是 Python `list`（作为`relationship()`的默认值），包含重复项，并且对象从其中一个位置移除但未从其他位置移除时，多对一反向引用会将其属性设置为`None`，即使一对多侧仍然表示对象存在。即使一对多集合在关系模型中不能有重复项，但使用序列集合的 ORM 映射的`relationship()`在内存中可以有重复项，限制是此重复状态既不能持久化也不能从数据库中检索。特别是，在列表中临时存在重复项是 Python“交换”操作的固有特性。给定标准的一对多/多对一设置：

```py
class A(Base):
    __tablename__ = "a"

    id = Column(Integer, primary_key=True)
    bs = relationship("B", backref="a")

class B(Base):
    __tablename__ = "b"
    id = Column(Integer, primary_key=True)
    a_id = Column(ForeignKey("a.id"))
```

如果我们有一个具有两个`B`成员的`A`对象，并执行交换：

```py
a1 = A(bs=[B(), B()])

a1.bs[0], a1.bs[1] = a1.bs[1], a1.bs[0]
```

在上述操作期间，拦截标准 Python `__setitem__` `__delitem__`方法提供了第二个`B()`对象在集合中出现两次的临时状态。当`B()`对象从一个位置移除时，`B.a`反向引用将将引用设置为`None`，导致在刷新期间删除`A`和`B`对象之间的链接。相同的问题也可以使用普通重复项来演示：

```py
>>> a1 = A()
>>> b1 = B()
>>> a1.bs.append(b1)
>>> a1.bs.append(b1)  # append the same b1 object twice
>>> del a1.bs[1]
>>> a1.bs  # collection is unaffected so far...
[<__main__.B object at 0x7f047af5fb70>]
>>> b1.a  # however b1.a is None
>>>
>>> session.add(a1)
>>> session.commit()  # so upon flush + expire....
>>> a1.bs  # the value is gone
[]
```

此修复确保在触发反向引用之前（在集合被改变之前），会检查集合中是否恰好有一个或零个目标项的实例，然后在取消多对一侧时使用线性搜索，目前使用`list.search`和`list.__contains__`。

最初认为需要在集合内部使用基于事件的引用计数方案，以便在整个集合的生命周期中跟踪所有重复的实例，这将对所有集合操作产生性能/内存/复杂性影响，包括非常频繁的加载和追加操作。相反采取的方法将额外的开销限制在集合移除和批量替换这些不太常见的操作上，并且线性扫描的观察开销是可以忽略的；在工作单元内以及在集合进行批量替换时，已经在关系绑定集合中使用了线性扫描。

[#1103](https://www.sqlalchemy.org/trac/ticket/1103)

## 关键行为变化 - ORM

### Query.join()更明确地处理决定“左”侧的模棱两可情况

从历史上看，给定如下查询：

```py
u_alias = aliased(User)
session.query(User, u_alias).join(Address)
```

鉴于标准教程映射，查询将产生一个 FROM 子句如下：

```py
SELECT  ...
FROM  users  AS  users_1,  users  JOIN  addresses  ON  users.id  =  addresses.user_id
```

也就是说，JOIN 将隐式地与第一个匹配的实体进行连接。新的行为是，异常请求解决这种模棱两可的情况：

```py
sqlalchemy.exc.InvalidRequestError: Can't determine which FROM clause to
join from, there are multiple FROMS which can join to this entity.
Try adding an explicit ON clause to help resolve the ambiguity.
```

解决方案是提供一个 ON 子句，可以是一个表达式：

```py
# join to User
session.query(User, u_alias).join(Address, Address.user_id == User.id)

# join to u_alias
session.query(User, u_alias).join(Address, Address.user_id == u_alias.id)
```

或者使用关系属性，如果可用的话：

```py
# join to User
session.query(User, u_alias).join(Address, User.addresses)

# join to u_alias
session.query(User, u_alias).join(Address, u_alias.addresses)
```

更改包括现在 JOIN 可以正确地链接到不是列表中第一个元素的 FROM 子句，如果 JOIN 本身不是模棱两可的话：

```py
session.query(func.current_timestamp(), User).join(Address)
```

在此增强之前，上述查询将引发：

```py
sqlalchemy.exc.InvalidRequestError: Don't know how to join from
CURRENT_TIMESTAMP; please use select_from() to establish the
left entity/selectable of this join
```

现在查询正常工作：

```py
SELECT  CURRENT_TIMESTAMP  AS  current_timestamp_1,  users.id  AS  users_id,
users.name  AS  users_name,  users.fullname  AS  users_fullname,
users.password  AS  users_password
FROM  users  JOIN  addresses  ON  users.id  =  addresses.user_id
```

总体而言，这种变化直接符合 Python 的“显式优于隐式”的哲学。

[#4365](https://www.sqlalchemy.org/trac/ticket/4365)  ### FOR UPDATE 子句在联合加载子查询中以及外部呈现

此更改特别适用于使用`joinedload()`加载策略与行限制查询结合使用时，例如使用`Query.first()`或`Query.limit()`，以及使用`Query.with_for_update()`方法。

给定一个查询如下：

```py
session.query(A).options(joinedload(A.b)).limit(5)
```

当联合加载与 LIMIT 结合时，`Query`对象呈现以下形式的 SELECT：

```py
SELECT  subq.a_id,  subq.a_data,  b_alias.id,  b_alias.data  FROM  (
  SELECT  a.id  AS  a_id,  a.data  AS  a_data  FROM  a  LIMIT  5
)  AS  subq  LEFT  OUTER  JOIN  b  ON  subq.a_id=b.a_id
```

这样做是为了使主实体的行限制不影响相关项目的联合加载。当上述查询与“SELECT..FOR UPDATE”结合时，行为是这样的：

```py
SELECT  subq.a_id,  subq.a_data,  b_alias.id,  b_alias.data  FROM  (
  SELECT  a.id  AS  a_id,  a.data  AS  a_data  FROM  a  LIMIT  5
)  AS  subq  LEFT  OUTER  JOIN  b  ON  subq.a_id=b.a_id  FOR  UPDATE
```

然而，由于 MySQL [`bugs.mysql.com/bug.php?id=90693`](https://bugs.mysql.com/bug.php?id=90693) 不锁定子查询中的行，不像 PostgreSQL 和其他数据库。因此，上述查询现在呈现为：

```py
SELECT  subq.a_id,  subq.a_data,  b_alias.id,  b_alias.data  FROM  (
  SELECT  a.id  AS  a_id,  a.data  AS  a_data  FROM  a  LIMIT  5  FOR  UPDATE
)  AS  subq  LEFT  OUTER  JOIN  b  ON  subq.a_id=b.a_id  FOR  UPDATE
```

在 Oracle 方言中，内部的“FOR UPDATE”不会呈现，因为 Oracle 不支持此语法，方言会跳过针对子查询的任何“FOR UPDATE”；在任何情况下都不是必要的，因为 Oracle 像 PostgreSQL 一样正确锁定返回行的所有元素。

当使用 `Query.with_for_update.of` 修饰符时，通常在 PostgreSQL 上，外部的“FOR UPDATE”被省略，OF 现在在内部呈现；以前，OF 目标不会被正确转换以适应子查询。所以给定：

```py
session.query(A).options(joinedload(A.b)).with_for_update(of=A).limit(5)
```

查询现在会呈现为：

```py
SELECT  subq.a_id,  subq.a_data,  b_alias.id,  b_alias.data  FROM  (
  SELECT  a.id  AS  a_id,  a.data  AS  a_data  FROM  a  LIMIT  5  FOR  UPDATE  OF  a
)  AS  subq  LEFT  OUTER  JOIN  b  ON  subq.a_id=b.a_id
```

以上形式在 PostgreSQL 上应该有所帮助，此外，由于 PostgreSQL 不允许在 LEFT OUTER JOIN 目标之后呈现 FOR UPDATE 子句。

总的来说，FOR UPDATE 仍然高度特定于正在使用的目标数据库，并且不能轻易地推广到更复杂的查询。

[#4246](https://www.sqlalchemy.org/trac/ticket/4246)  ### passive_deletes=’all’ 将使 FK 在从集合中移除的对象中保持不变

`relationship.passive_deletes` 选项接受值 `"all"`，表示当对象被刷新时，不应修改任何外键属性，即使关系的集合/引用已被移除。以前，在以下情况下不会发生这种情况：

```py
class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True)
    addresses = relationship("Address", passive_deletes="all")

class Address(Base):
    __tablename__ = "addresses"
    id = Column(Integer, primary_key=True)
    email = Column(String)

    user_id = Column(Integer, ForeignKey("users.id"))
    user = relationship("User")

u1 = session.query(User).first()
address = u1.addresses[0]
u1.addresses.remove(address)
session.commit()

# would fail and be set to None
assert address.user_id == u1.id
```

修复现在包括 `address.user_id` 保持不变，根据 `passive_deletes="all"`。这种情况对于构建自定义“版本表”方案等非常有用，其中行被归档而不是删除。

[#3844](https://www.sqlalchemy.org/trac/ticket/3844)  ## 新功能和改进 - 核心

### 新的多列命名约定标记，长名称截断

为了适应一个`MetaData`命名约定需要在多列约束之间消除歧义，并希望在生成的约束名中使用所有列的情况，添加了一系列新的命名约定标记，包括`column_0N_name`、`column_0_N_name`、`column_0N_key`、`column_0_N_key`、`referred_column_0N_name`、`referred_column_0_N_name`等，它们将约束中所有列的列名（或键或标签）连接在一起，要么没有分隔符，要么用下划线分隔符连接。下面我们定义一个约定，将`UniqueConstraint`约束命名为所有列名称的组合：

```py
metadata_obj = MetaData(
    naming_convention={"uq": "uq_%(table_name)s_%(column_0_N_name)s"}
)

table = Table(
    "info",
    metadata_obj,
    Column("a", Integer),
    Column("b", Integer),
    Column("c", Integer),
    UniqueConstraint("a", "b", "c"),
)
```

上述表的 CREATE TABLE 将呈现为：

```py
CREATE  TABLE  info  (
  a  INTEGER,
  b  INTEGER,
  c  INTEGER,
  CONSTRAINT  uq_info_a_b_c  UNIQUE  (a,  b,  c)
)
```

此外，现在将长名称截断逻辑应用于由命名约定生成的名称，特别是为了适应可能产生非常长名称的多列标签。这种逻辑与在 SELECT 语句中截断长标签名称所使用的逻辑相同，它会用一个确定性生成的 4 字符哈希替换超过目标数据库标识符长度限制的多余字符。例如，在 PostgreSQL 中，标识符不能超过 63 个字符，一个长约束名通常会从下面的表定义中生成：

```py
long_names = Table(
    "long_names",
    metadata_obj,
    Column("information_channel_code", Integer, key="a"),
    Column("billing_convention_name", Integer, key="b"),
    Column("product_identifier", Integer, key="c"),
    UniqueConstraint("a", "b", "c"),
)
```

截断逻辑将确保不会为唯一约束生成过长的名称：

```py
CREATE  TABLE  long_names  (
  information_channel_code  INTEGER,
  billing_convention_name  INTEGER,
  product_identifier  INTEGER,
  CONSTRAINT  uq_long_names_information_channel_code_billing_conventi_a79e
  UNIQUE  (information_channel_code,  billing_convention_name,  product_identifier)
)
```

上述后缀`a79e`基于长名称的 md5 哈希，并且每次都会生成相同的值，以产生给定模式的一致名称。

请注意，当约束名称对于给定方言明确过大时，截断逻辑还会引发`IdentifierError`。这已经是很长时间以来`Index`对象的行为，但现在也适用于其他类型的约束：

```py
from sqlalchemy import Column
from sqlalchemy import Integer
from sqlalchemy import MetaData
from sqlalchemy import Table
from sqlalchemy import UniqueConstraint
from sqlalchemy.dialects import postgresql
from sqlalchemy.schema import AddConstraint

m = MetaData()
t = Table("t", m, Column("x", Integer))
uq = UniqueConstraint(
    t.c.x,
    name="this_is_too_long_of_a_name_for_any_database_backend_even_postgresql",
)

print(AddConstraint(uq).compile(dialect=postgresql.dialect()))
```

将输出：

```py
sqlalchemy.exc.IdentifierError: Identifier
'this_is_too_long_of_a_name_for_any_database_backend_even_postgresql'
exceeds maximum length of 63 characters
```

异常抛出阻止了由数据库后端截断的非确定性约束名称的生成，这些名称后来与数据库迁移不兼容。

要将 SQLAlchemy 端的截断规则应用于上述标识符，请使用`conv()`构造：

```py
uq = UniqueConstraint(
    t.c.x,
    name=conv("this_is_too_long_of_a_name_for_any_database_backend_even_postgresql"),
)
```

这将再次输出确定性截断的 SQL，如下所示：

```py
ALTER  TABLE  t  ADD  CONSTRAINT  this_is_too_long_of_a_name_for_any_database_backend_eve_ac05  UNIQUE  (x)
```

目前还没有选项使名称通过以允许数据库端截断。这在`Index`名称上已经有一段时间了，也没有引起问题。

此更改还修复了另外两个问题。 其中一个是`column_0_key`令牌尽管已记录文档，但却不可用，另一个是`referred_column_0_name`令牌如果这两个值不同，则会无意中渲染`.key`而不是列的`.name`。

另请参阅

配置约束命名约定

`MetaData.naming_convention`

[#3989](https://www.sqlalchemy.org/trac/ticket/3989)  ### SQL 函数的二进制比较解释

此增强功能是在核心级别实现的，但主要适用于 ORM。

现在，可以将比较两个元素的 SQL 函数用作“比较”对象，适用于 ORM `relationship()`的使用，首先像往常一样使用 `func` 工厂创建函数，然后当函数完成时调用 `FunctionElement.as_comparison()` 修改器以生成具有“左”和“右”侧的 `BinaryExpression`：

```py
class Venue(Base):
    __tablename__ = "venue"
    id = Column(Integer, primary_key=True)
    name = Column(String)

    descendants = relationship(
        "Venue",
        primaryjoin=func.instr(remote(foreign(name)), name + "/").as_comparison(1, 2)
        == 1,
        viewonly=True,
        order_by=name,
    )
```

上述`relationship.primaryjoin`中的“descendants”关系将基于传递给`instr()`的第一个和第二个参数生成“左”和“右”表达式。 这允许 ORM 懒加载等功能生成类似以下的 SQL：

```py
SELECT  venue.id  AS  venue_id,  venue.name  AS  venue_name
FROM  venue
WHERE  instr(venue.name,  (?  ||  ?))  =  ?  ORDER  BY  venue.name
('parent1',  '/',  1)
```

和 joinedload，例如：

```py
v1 = (
    s.query(Venue)
    .filter_by(name="parent1")
    .options(joinedload(Venue.descendants))
    .one()
)
```

使其工作如下：

```py
SELECT  venue.id  AS  venue_id,  venue.name  AS  venue_name,
  venue_1.id  AS  venue_1_id,  venue_1.name  AS  venue_1_name
FROM  venue  LEFT  OUTER  JOIN  venue  AS  venue_1
  ON  instr(venue_1.name,  (venue.name  ||  ?))  =  ?
WHERE  venue.name  =  ?  ORDER  BY  venue_1.name
('/',  1,  'parent1')
```

该功能预计将有助于处理诸如在关系连接条件中使用几何函数，或者任何在 SQL 连接的 ON 子句中以 SQL 函数的形式表达的情况等情况。

[#3831](https://www.sqlalchemy.org/trac/ticket/3831)  ### 扩展 IN 功能现在支持空列表

在版本 1.2 中引入的“expanding IN”功能现在支持传递给`ColumnOperators.in_()`运算符的空列表。 对于空列表的实现将生成一个针对目标后端具体的“空集合”表达式，例如对于 PostgreSQL，“SELECT CAST(NULL AS INTEGER) WHERE 1!=1”，对于 MySQL，“SELECT 1 FROM (SELECT 1) as _empty_set WHERE 1!=1”：

```py
>>> from sqlalchemy import create_engine
>>> from sqlalchemy import select, literal_column, bindparam
>>> e = create_engine("postgresql://scott:tiger@localhost/test", echo=True)
>>> with e.connect() as conn:
...     conn.execute(
...         select([literal_column("1")]).where(
...             literal_column("1").in_(bindparam("q", expanding=True))
...         ),
...         q=[],
...     )
{exexsql}SELECT 1 WHERE 1 IN (SELECT CAST(NULL AS INTEGER) WHERE 1!=1)
```

该功能还适用于基于元组的 IN 语句，其中“空 IN”表达式将被扩展以支持元组中给定的元素，例如在 PostgreSQL 上：

```py
>>> from sqlalchemy import create_engine
>>> from sqlalchemy import select, literal_column, tuple_, bindparam
>>> e = create_engine("postgresql://scott:tiger@localhost/test", echo=True)
>>> with e.connect() as conn:
...     conn.execute(
...         select([literal_column("1")]).where(
...             tuple_(50, "somestring").in_(bindparam("q", expanding=True))
...         ),
...         q=[],
...     )
{exexsql}SELECT 1 WHERE (%(param_1)s, %(param_2)s)
IN (SELECT CAST(NULL AS INTEGER), CAST(NULL AS VARCHAR) WHERE 1!=1)
```

[#4271](https://www.sqlalchemy.org/trac/ticket/4271)  ### TypeEngine 方法 bind_expression, column_expression 适用于 Variant、特定类型

当这些方法存在于特定数据类型的“impl”上时，`TypeEngine.bind_expression()` 和 `TypeEngine.column_expression()` 方法现在可以工作，允许方言以及 `TypeDecorator` 和 `Variant` 使用这些方法。

以下示例说明了一个将 SQL 时间转换函数应用于 `LargeBinary` 的 `TypeDecorator`。为了使此类型在 `Variant` 的上下文中工作，编译器需要深入到变体表达式的“impl”中以定位这些方法：

```py
from sqlalchemy import TypeDecorator, LargeBinary, func

class CompressedLargeBinary(TypeDecorator):
    impl = LargeBinary

    def bind_expression(self, bindvalue):
        return func.compress(bindvalue, type_=self)

    def column_expression(self, col):
        return func.uncompress(col, type_=self)

MyLargeBinary = LargeBinary().with_variant(CompressedLargeBinary(), "sqlite")
```

上述表达式仅在 SQLite 上使用时会呈现为 SQL 中的函数：

```py
from sqlalchemy import select, column
from sqlalchemy.dialects import sqlite

print(select([column("x", CompressedLargeBinary)]).compile(dialect=sqlite.dialect()))
```

将呈现为：

```py
SELECT  uncompress(x)  AS  x
```

此更改还包括方言可以在方言级别的实现类型上实现`TypeEngine.bind_expression()`和`TypeEngine.column_expression()`，在那里它们现在将被使用；特别是这将用于 MySQL 的新“二进制前缀”要求以及用于将 MySQL 的十进制绑定值转换为强制转换的情况。

[#3981](https://www.sqlalchemy.org/trac/ticket/3981)  ### 新的后进先出策略适用于 QueuePool

通常由`create_engine()`使用的连接池被称为`QueuePool`。 这个池使用一个类似于 Python 内置的`Queue`类的对象来存储等待使用的数据库连接。 `Queue`具有先进先出的行为，旨在提供对池中持久存在的数据库连接的循环使用。 然而，这种方法的一个潜在缺点是，当池的利用率较低时，池中每个连接的串行重复使用意味着试图减少未使用连接的服务器端超时策略被阻止关闭这些连接。 为了适应这种用例，添加了一个新标志`create_engine.pool_use_lifo`，它将`Queue`的`.get()`方法反转，从队列的开头而不是末尾获取连接，从本质上将“队列”变成“栈”（考虑到这太啰嗦，因此没有添加一个名为`StackPool`的全新池）。

另请参阅

使用 FIFO vs. LIFO

## 核心关键变化

### 完全移除将字符串 SQL 片段强制转换为 text()

首次添加于版本 1.0 的警告，描述在将完整 SQL 片段强制转换为 text() 时发出的警告，现已转换为异常。 对于像`Query.filter()`和`Select.order_by()`这样的方法传递的字符串片段自动转换为`text()`构造的自动转换引发了持续的担忧，尽管这已发出警告。 在`Select.order_by()`、`Query.order_by()`、`Select.group_by()`和`Query.group_by()`的情况下，字符串标签或列名仍然会解析为相应的表达式构造，但如果解析失败，则会引发`CompileError`，从而防止直接呈现原始 SQL 文本。

[#4481](https://www.sqlalchemy.org/trac/ticket/4481)  ### “threadlocal”引擎策略已弃用

“线程本地引擎策略”是在 SQLAlchemy 0.2 左右添加的，作为解决在 SQLAlchemy 0.1 中操作的标准方式存在问题的解决方案，可以总结为“一切都是线程本地”。回顾来看，似乎相当荒谬，SQLAlchemy 的首次发布在各方面都是“alpha”版本，已经担心太多用户已经定居在现有 API 上，无法简单地更改它。

SQLAlchemy 的原始使用模型如下：

```py
engine.begin()

table.insert().execute(parameters)
result = table.select().execute()

table.update().execute(parameters)

engine.commit()
```

经过几个月的实际使用，很明显，假装“连接”或“事务”是一个隐藏的实现细节是一个坏主意，特别是当有人需要同时处理多个数据库连接时。因此，我们今天看到的使用范式被引入，减去了上下文管理器，因为它们在 Python 中尚不存在：

```py
conn = engine.connect()
try:
    trans = conn.begin()

    conn.execute(table.insert(), parameters)
    result = conn.execute(table.select())

    conn.execute(table.update(), parameters)

    trans.commit()
except:
    trans.rollback()
    raise
finally:
    conn.close()
```

上述范式是人们所需要的，但由于仍然有点啰嗦（因为没有上下文管理器），旧的工作方式也被保留了下来，并成为了线程本地引擎策略。

今天，与 Core 一起工作要简洁得多，甚至比原始模式更简洁，这要归功于上下文管理器：

```py
with engine.begin() as conn:
    conn.execute(table.insert(), parameters)
    result = conn.execute(table.select())

    conn.execute(table.update(), parameters)
```

此时，任何仍然依赖“threadlocal”风格的代码都将通过此弃用被鼓励进行现代化改造 - 该功能应在下一个主要系列的 SQLAlchemy 中完全移除，例如 1.4 版。连接池参数`Pool.use_threadlocal`也已弃用，因为在大多数情况下实际上没有任何效果，`Engine.contextual_connect()`方法也已弃用，该方法通常与`Engine.connect()`方法是同义的，除非使用了线程本地引擎。

[#4393](https://www.sqlalchemy.org/trac/ticket/4393)  ### convert_unicode 参数已弃用

参数`String.convert_unicode`和`create_engine.convert_unicode`已被弃用。这些参数的目的是指示 SQLAlchemy 确保在 Python 2 中传递给数据库之前将传入的 Python Unicode 对象编码为字节字符串，并期望从数据库接收的字节字符串转换回 Python Unicode 对象。在 Python 3 之前的时代，这是一个巨大的挑战，因为几乎所有的 Python DBAPI 默认情况下都没有启用 Unicode 支持，并且大多数都存在与它们提供的 Unicode 扩展相关的主要问题。最终，SQLAlchemy 添加了 C 扩展，其中一个主要目的是加快结果集中的 Unicode 解码过程。

一旦引入了 Python 3，DBAPI 开始更全面地支持 Unicode，并且更重要的是，默认情况下支持 Unicode。然而，特定 DBAPI 在何种条件下返回 Unicode 数据以及接受 Python Unicode 值作为参数的条件仍然非常复杂。这标志着“convert_unicode”标志开始过时，因为它们不再足以确保编码/解码仅在需要时发生，而不是在不需要时发生。相反，“convert_unicode”开始被方言自动检测。这可以在引擎第一次连接时发出的“SELECT ‘test plain returns’”和“SELECT ‘test_unicode_returns’”SQL 中看到；方言正在测试当前 DBAPI 及其当前设置和后端数据库连接是否默认返回 Unicode。

最终结果是，终端用户在任何情况下都不再需要使用“convert_unicode”标志，如果需要，SQLAlchemy 项目需要知道这些情况及原因。目前，在所有主要数据库上，数百个 Unicode 往返测试通过，而不使用此标志，因此可以相当有信心地说它们不再需要，除非在争议的非使用情况下，例如访问来自传统数据库的错误编码数据，这种情况最好使用自定义类型。

[#4393](https://www.sqlalchemy.org/trac/ticket/4393)

## 方言改进和更改 - PostgreSQL

### 为 PostgreSQL 分区表添加基本反射支持

SQLAlchemy 可以使用版本 1.2.6 中添加的`postgresql_partition_by`标志，在 PostgreSQL 的 CREATE TABLE 语句中呈现“PARTITION BY”序列。然而，`'p'`类型直到现在都不是反射查询的一部分。

给定一个类似于以下的模式：

```py
dv = Table(
    "data_values",
    metadata_obj,
    Column("modulus", Integer, nullable=False),
    Column("data", String(30)),
    postgresql_partition_by="range(modulus)",
)

sa.event.listen(
    dv,
    "after_create",
    sa.DDL(
        "CREATE TABLE data_values_4_10 PARTITION OF data_values "
        "FOR VALUES FROM (4) TO (10)"
    ),
)
```

两个表名 `'data_values'` 和 `'data_values_4_10'` 将从 `Inspector.get_table_names()` 中返回，此外，列也将从 `Inspector.get_columns('data_values')` 以及 `Inspector.get_columns('data_values_4_10')` 中返回。这也适用于对这些表使用 `Table(..., autoload=True)`。

[#4237](https://www.sqlalchemy.org/trac/ticket/4237)

## 方言改进和变更 - MySQL

### 协议级别的 ping 现在用于预 ping

包括 mysqlclient、python-mysql、PyMySQL 和 mysql-connector-python 在内的 MySQL 方言现在使用 `connection.ping()` 方法进行池预 ping 功能，详情请参阅 Disconnect Handling - Pessimistic。这比以前在连接上发出 “SELECT 1” 的方法更轻量级。### 控制 ON DUPLICATE KEY UPDATE 中参数的排序

可以通过传递一个 2 元组列表来显式地为 `ON DUPLICATE KEY UPDATE` 子句中的 UPDATE 参数进行排序：

```py
from sqlalchemy.dialects.mysql import insert

insert_stmt = insert(my_table).values(id="some_existing_id", data="inserted value")

on_duplicate_key_stmt = insert_stmt.on_duplicate_key_update(
    [
        ("data", "some data"),
        ("updated_at", func.current_timestamp()),
    ],
)
```

另请参阅

INSERT…ON DUPLICATE KEY UPDATE（Upsert）

## 方言改进和变更 - SQLite

### 添加对 SQLite JSON 的支持

添加了新的数据类型 `JSON` ，它代表了 `JSON` 基本数据类型的 SQLite 的 json 成员访问函数。实现使用 SQLite 的 `JSON_EXTRACT` 和 `JSON_QUOTE` 函数提供基本的 JSON 支持。

请注意，数据库中呈现的数据类型本身的名称为“JSON”。这将创建一个具有“numeric”亲和性的 SQLite 数据类型，通常情况下不应该成为问题，除非是由单个整数值组成的 JSON 值的情况。尽管如此，根据 SQLite 自己文档中的示例，JSON 的名称仍然被用于其熟悉性。参见 [`www.sqlite.org/json1.html`](https://www.sqlite.org/json1.html)

[#3850](https://www.sqlalchemy.org/trac/ticket/3850)  ### 添加对约束中 SQLite ON CONFLICT 的支持

SQLite 支持非标准的 ON CONFLICT 子句，可为独立约束以及一些列内约束（如 NOT NULL）指定。通过向 `UniqueConstraint` 等对象添加 `sqlite_on_conflict` 关键字，已对这些子句进行了支持，以及几种 `Column` -特定变体：

```py
some_table = Table(
    "some_table",
    metadata_obj,
    Column("id", Integer, primary_key=True, sqlite_on_conflict_primary_key="FAIL"),
    Column("data", Integer),
    UniqueConstraint("id", "data", sqlite_on_conflict="IGNORE"),
)
```

上表将在 CREATE TABLE 语句中呈现为：

```py
CREATE  TABLE  some_table  (
  id  INTEGER  NOT  NULL,
  data  INTEGER,
  PRIMARY  KEY  (id)  ON  CONFLICT  FAIL,
  UNIQUE  (id,  data)  ON  CONFLICT  IGNORE
)
```

另请参阅

对约束的 ON CONFLICT 支持

[#4360](https://www.sqlalchemy.org/trac/ticket/4360)

## 方言改进和变化 - Oracle

### 国家字符数据类型弱化以支持通用 Unicode，可通过选项重新启用

现在，默认情况下，`Unicode`和`UnicodeText`数据类型现在对应于 Oracle 上的`VARCHAR2`和`CLOB`数据类型，而不是`NVARCHAR2`和`NCLOB`（也称为“国家”字符集类型）。这将在`CREATE TABLE`语句中的呈现行为中看到，以及当使用`Unicode`或`UnicodeText`的绑定参数时，不会传递类型对象给`setinputsizes()`；cx_Oracle 会原生处理字符串值。这种变化基于 cx_Oracle 维护者的建议，即 Oracle 中的“国家”数据类型在很大程度上已经过时且性能不佳。它们还会在某些情况下干扰，比如应用于`trunc()`等函数的格式说明符时。

当数据库不使用符合 Unicode 标准的字符集时，可能需要使用`NVARCHAR2`和相关类型的情况。在这种情况下，可以通过向`create_engine()`传递标志`use_nchar_for_unicode`来重新启用旧行为。

始终明确使用`NVARCHAR2`和`NCLOB`数据类型将继续使用`NVARCHAR2`和`NCLOB`，包括在 DDL 中以及在处理绑定参数时使用 cx_Oracle 的`setinputsizes()`。

在读取方面，在 Python 2 下已添加了对 CHAR/VARCHAR/CLOB 结果行的自动 Unicode 转换，以匹配 Python 3 下 cx_Oracle 的行为。为了减轻 cx_Oracle 方言在 Python 2 下先前具有的性能问题，SQLAlchemy 在 Python 2 下使用非常高效（当构建 C 扩展时）的本机 Unicode 处理程序。可以通过将`coerce_to_unicode`标志设置为 False 来禁用自动 Unicode 强制转换。此标志现在默认为 True，并适用于所有在结果集中返回的不明确为`Unicode`或 Oracle 的 NVARCHAR2/NCHAR/NCLOB 数据类型的字符串数据。

[#4242](https://www.sqlalchemy.org/trac/ticket/4242)  ### cx_Oracle 连接参数现代化，已弃用的参数已移除

�� cx_oracle 方言接受的参数以及 URL 字符串进行了一系列现代化处理：

+   弃用的参数`auto_setinputsizes`、`allow_twophase`、`exclude_setinputsizes`已被移除。

+   `threaded`参数的值，在 SQLAlchemy 方言中一直默认为 True，现在不再默认生成。SQLAlchemy `Connection` 对象本身不被视为线程安全，因此不需要传递此标志。

+   将`threaded`传递给`create_engine()`本身已被弃用。要将`threaded`的值设置为`True`，请将其传递给`create_engine.connect_args`字典或使用查询字符串，例如`oracle+cx_oracle://...?threaded=true`。

+   所有传递到 URL 查询字符串的参数，除非另有特殊处理，现在都传递给 cx_Oracle.connect()函数。其中一些也被强制转换为 cx_Oracle 常量或布尔值，包括`mode`、`purity`、`events`和`threaded`。

+   与以往一样，所有 cx_Oracle `.connect()` 参数都通过`create_engine.connect_args` 字典接受，文档对此描述不准确。

[#4369](https://www.sqlalchemy.org/trac/ticket/4369)

## 方言改进和变化 - SQL Server

### 支持 pyodbc fast_executemany

Pyodbc 最近添加的“fast_executemany”模式，在使用 Microsoft ODBC 驱动程序时可用，现在是 pyodbc / mssql 方言的选项。通过`create_engine()`传递：

```py
engine = create_engine(
    "mssql+pyodbc://scott:tiger@mssql2017:1433/test?driver=ODBC+Driver+13+for+SQL+Server",
    fast_executemany=True,
)
```

另请参阅

快速 Executemany 模式

[#4158](https://www.sqlalchemy.org/trac/ticket/4158)  ### 新参数影响 IDENTITY 的起始和增量，使用 Sequence 已被弃用

SQL Server 自 SQL Server 2012 起现在支持具有真实 `CREATE SEQUENCE` 语法的序列。在 [#4235](https://www.sqlalchemy.org/trac/ticket/4235) 中，SQLAlchemy 将添加对这些序列的支持，使用 `Sequence`，方式与其他任何方言一样。然而，目前的情况是 `Sequence` 已经在 SQL Server 上重新用途，以影响主键列的 `IDENTITY` 规范的 “start” 和 “increment” 参数。为了向普通序列也可用过渡，使用 `Sequence` 将在 1.3 系列中的整个过渡期间发出弃用警告。为了影响 “start” 和 “increment”，请在 `Column` 上使用新的 `mssql_identity_start` 和 `mssql_identity_increment` 参数：

```py
test = Table(
    "test",
    metadata_obj,
    Column(
        "id",
        Integer,
        primary_key=True,
        mssql_identity_start=100,
        mssql_identity_increment=10,
    ),
    Column("name", String(20)),
)
```

要在非主键列上发出 `IDENTITY`，这是一个很少使用但有效的 SQL Server 情况，请使用 `Column.autoincrement` 标志，在目标列上将其设置为 `True`，在任何整数主键列上设置为 `False`：

```py
test = Table(
    "test",
    metadata_obj,
    Column("id", Integer, primary_key=True, autoincrement=False),
    Column("number", Integer, autoincrement=True),
)
```

另请参阅

自动增量行为 / IDENTITY 列

[#4362](https://www.sqlalchemy.org/trac/ticket/4362)

[#4235](https://www.sqlalchemy.org/trac/ticket/4235)  ## 更改的 StatementError 格式（换行符和 %s）

对于 `StatementError` 的字符串表示引入了两个更改。字符串表示的“详细信息”和“SQL”部分现在由换行符分隔，并且保留了原始 SQL 语句中存在的换行符。目标是在保持原始错误消息单行记录的同时提高可读性。

这意味着以前看起来像这样的错误消息：

```py
sqlalchemy.exc.StatementError: (sqlalchemy.exc.InvalidRequestError) A value is
required for bind parameter 'id' [SQL: 'select * from reviews\nwhere id = ?']
(Background on this error at: https://sqlalche.me/e/cd3x)
```

现在将如下所示：

```py
sqlalchemy.exc.StatementError: (sqlalchemy.exc.InvalidRequestError) A value is required for bind parameter 'id'
[SQL: select * from reviews
where id = ?]
(Background on this error at: https://sqlalche.me/e/cd3x)
```

此更改的主要影响是消费者不再能假定完整的异常消息在单行上，但是来自 DBAPI 驱动程序或 SQLAlchemy 内部生成的原始 “错误” 部分仍将位于第一行。

[#4500](https://www.sqlalchemy.org/trac/ticket/4500)

## 介绍

本指南介绍了 SQLAlchemy 版本 1.3 中的新功能，并记录了对将应用程序从 SQLAlchemy 1.2 系列迁移到 1.3 系列的用户产生影响的更改。

请仔细查看行为更改部分，了解可能的向后不兼容的行为更改。

## 一般

### 为所有弃用元素发出弃用警告；添加新的弃用

发行版 1.3 确保所有被弃用的行为和 API，包括所有长期被列为“遗留”的行为和 API，都会发出 `DeprecationWarning` 警告。这包括在使用诸如 `Session.weak_identity_map` 这样的参数和 `MapperExtension` 这样的类时。虽然所有弃用已在文档中注明，但通常它们没有使用正确的重新构造文本指令，或者包含它们被弃用的版本。一个特定的 API 功能是否实际发出弃用警告并不一致。一般的态度是，大多数或所有这些弃用功能都被视为长期遗留功能，没有计划删除它们。

此次变更包括所有文档化的弃用现在都在文档中使用了正确的重新构造文本指令，并附有版本号，明确指出该功能或用例将在将来的版本中删除（例如，不再有永久使用的旧用例），以及使用任何此类功能或用例都将明确发出 `DeprecationWarning` 警告，在 Python 3 中以及使用像 Pytest 这样的现代测试工具时，现在在标准错误流中更加明确。目标是，这些长时间弃用的功能，回溯到版本 0.7 或 0.6，应该开始被完全删除，而不是保留它们作为“遗留”功能。此外，从版本 1.3 开始，一些重大的新弃用将被添加。由于 SQLAlchemy 在数千开发人员的实际使用中已有 14 年，因此可以指出一种混合得很好的用例流，并且修剪掉与这种单一工作方式相违背的功能和模式。

更大的背景是 SQLAlchemy 试图适应即将到来的仅支持 Python 3 的世界，以及类型注解的世界，为此，**初步**计划对 SQLAlchemy 进行重大改版，希望大大减少 API 的认知负担，以及对 Core 和 ORM 之间众多实现和使用差异的重大调整。由于这两个系统在 SQLAlchemy 首次发布后发生了巨大变化，特别是 ORM 仍保留了许多“后期添加的”行为，这使得 Core 和 ORM 之间的隔离墙过高。通过提前将 API 集中在每个支持的用例的单一模式上，将来对显着改变的 API 进行迁移的工作变得更加简单。

对于在 1.3 中添加的最重大的弃用，请参见下面的链接部分。

另请参阅

“threadlocal” 引擎策略已弃用

convert_unicode 参数已弃用

关于 AliasedClass 的关系取代了非主映射器的需要

[#4393](https://www.sqlalchemy.org/trac/ticket/4393)  ### 所有弃用元素都会发出弃用警告；新增弃用

发布 1.3 确保所有被弃用的行为和 API，包括那些多年来一直被列为“遗留”的行为，都会发出`DeprecationWarning`警告。这包括当使用参数如`Session.weak_identity_map`和类似`MapperExtension`时。虽然所有弃用都已在文档中记录，但通常它们没有使用适当的重构文本指令，或者包含它们被弃用的版本。特定 API 功能是否实际发出弃用警告并不一致。一般的态度是，大多数或所有这些弃用功能都被视为长期遗留功能，没有计划删除它们。

此更改包括，所有记录的弃用现在在文档中使用适当的重构文本指令，并附有版本号，明确说明该功能或用例将在将来的版本中被移除（例如，不再有永久的遗留用例），并且使用任何此类功能或用例肯定会发出`DeprecationWarning`，在 Python 3 中以及使用现代测试工具如 Pytest 时，现在在标准错误流中更加明确。目标是，这些长期弃用的功能，可以追溯到版本 0.7 或 0.6，应该开始被完全移除，而不是将它们保留为“遗留”功能。此外，一些重大的新弃用功能正在版本 1.3 中添加。由于 SQLAlchemy 在数千开发人员的实际使用中已经有了 14 年的历史，可以指出一个混合在一起的使用案例流，以及修剪掉与这种单一工作方式相悖的功能和模式。

SQLAlchemy 的更大背景是，它试图适应即将到来的 Python 3-only 世界，以及一个类型注释的世界，为了实现这个目标，有**暂定**计划对 SQLAlchemy 进行重大改造，希望能大大减少 API 的认知负荷，并对 Core 和 ORM 之间的实现和使用之间的许多差异进行重大调整。由于这两个系统在 SQLAlchemy 首次发布后发生了巨大变化，特别是 ORM 仍然保留了许多“外挂”行为，使得 Core 和 ORM 之间的隔离墙过高。通过提前将 API 集中在每个支持的用例的单一模式上，将来迁移到显著改变的 API 的工作变得更简单。

对于 1.3 中添加的最重要的弃用功能，请参见下面的链接部分。

另请参阅

“threadlocal” engine strategy deprecated

convert_unicode parameters deprecated

与别名类的关系替代了非主要映射器的需求

[#4393](https://www.sqlalchemy.org/trac/ticket/4393)

## 新功能和改进 - ORM

### 与别名类的关系替代了非主要映射器的需求

“非主要映射器”是以命令式映射风格创建的`Mapper`，它充当已经映射的类的额外映射器，针对不同类型的可选择对象。非主要映射器起源于 SQLAlchemy 的 0.1、0.2 系列，那时预计`Mapper`对象将是主要的查询构建接口，之后才有`Query`对象的存在。

随着`Query`的出现，以及后来的`AliasedClass`构造，大多数非主要映射器的用例都消失了。这是一件好事，因为 SQLAlchemy 在 0.5 系列左右也完全放弃了“经典”映射，转而采用了声明式系统。

当意识到一些非常难以定义的`relationship()`配置可能会成为可能时，仍然保留了非主要映射器的一个用例。当使用非主要映射器作为映射目标时，可以使用替代可选择项，而不是尝试构建一个`relationship.primaryjoin`，该关系涵盖了特定对象间关系的所有复杂性。

随着这个用例变得越来越流行，它的局限性也变得明显，包括非主要映射器难以配置以适应添加新列的可选择项，映射器不继承原始映射的关系，明确配置在非主要映射器上的关系与加载器选项不兼容，非主要映射器还不能提供可在查询中使用的基于列的属性的完全功能命名空间（在旧的 0.1 - 0.4 时代，人们会直接使用`Table`对象与 ORM）。

缺失的部分是允许 `relationship()` 直接引用 `AliasedClass`。`AliasedClass` 已经做了非主映射器所要做的一切；它允许从备用可选择中加载现有映射的类，它继承现有映射器的所有属性和关系，它与加载器选项非常配合，还提供了一个可以像类本身一样混入查询的类似对象。通过这个改变，原来针对非主映射器的配置关系连接的配方被更改为别名类。

在关系到别名类处，原始的非主映射器如下所示：

```py
j = join(B, D, D.b_id == B.id).join(C, C.id == D.c_id)

B_viacd = mapper(
    B,
    j,
    non_primary=True,
    primary_key=[j.c.b_id],
    properties={
        "id": j.c.b_id,  # so that 'id' looks the same as before
        "c_id": j.c.c_id,  # needed for disambiguation
        "d_c_id": j.c.d_c_id,  # needed for disambiguation
        "b_id": [j.c.b_id, j.c.d_b_id],
        "d_id": j.c.d_id,
    },
)

A.b = relationship(B_viacd, primaryjoin=A.b_id == B_viacd.c.b_id)
```

这些属性是必需的，以便重新映射额外的列，使其不与已映射到`B`的现有列发生冲突，同时还需要定义一个新的主键。

使用新方法，所有这些冗长都会消失，并且在建立关系时可以直接引用额外的列：

```py
j = join(B, D, D.b_id == B.id).join(C, C.id == D.c_id)

B_viacd = aliased(B, j, flat=True)

A.b = relationship(B_viacd, primaryjoin=A.b_id == j.c.b_id)
```

非主映射器现已不推荐使用，最终目标是经典映射作为一种特性完全消失。声明性 API 将成为唯一的映射方式，这有望带来内部改进和简化，以及更清晰的文档说明。

[#4423](https://www.sqlalchemy.org/trac/ticket/4423)  ### selectin 加载不再使用 JOIN 来进行简单的一对多加载

1.2 版中添加的“selectin”加载功能引入了一种极其高效的新方法来急切加载集合，在许多情况下比“subquery”急切加载要快得多，因为它不依赖于重复原始 SELECT 查询，而是使用一个简单的 IN 子句。然而，“selectin”加载仍然依赖于在父表和相关表之间渲染 JOIN，因为它需要在行中使用父主键值来匹配行。在 1.3 中，添加了一个新的优化，将在简单的一对多加载的最常见情况下省略这个 JOIN，其中相关的行已经包含了其外键列中表达的父行的主键。这再次提供了显着的性能改进，因为 ORM 现在可以一次性加载大量集合，而完全不使用 JOIN 或子查询。

给定一个映射：

```py
class A(Base):
    __tablename__ = "a"

    id = Column(Integer, primary_key=True)
    bs = relationship("B", lazy="selectin")

class B(Base):
    __tablename__ = "b"
    id = Column(Integer, primary_key=True)
    a_id = Column(ForeignKey("a.id"))
```

在 1.2 版本的“selectin”加载中，从 A 到 B 的加载如下所示：

```py
SELECT  a.id  AS  a_id  FROM  a
SELECT  a_1.id  AS  a_1_id,  b.id  AS  b_id,  b.a_id  AS  b_a_id
FROM  a  AS  a_1  JOIN  b  ON  a_1.id  =  b.a_id
WHERE  a_1.id  IN  (?,  ?,  ?,  ?,  ?,  ?,  ?,  ?,  ?,  ?)  ORDER  BY  a_1.id
(1,  2,  3,  4,  5,  6,  7,  8,  9,  10)
```

使用新行为，加载如下：

```py
SELECT  a.id  AS  a_id  FROM  a
SELECT  b.a_id  AS  b_a_id,  b.id  AS  b_id  FROM  b
WHERE  b.a_id  IN  (?,  ?,  ?,  ?,  ?,  ?,  ?,  ?,  ?,  ?)  ORDER  BY  b.a_id
(1,  2,  3,  4,  5,  6,  7,  8,  9,  10)
```

该行为被自动释放，使用类似于延迟加载的启发式方法，以确定相关实体是否可以直接从标识映射中获取。然而，与大多数查询功能一样，由于关于多态加载的高级场景，该功能的实现变得更加复杂。如果遇到问题，用户应该报告错误，但是该更改还包括一个标志`relationship.omit_join`，可以在`relationship()`上设置为`False`以禁用优化。

[#4340](https://www.sqlalchemy.org/trac/ticket/4340)  ### 改进多对一查询表达式的行为

当构建一个查询，将一个多对一的关系与一个对象值进行比较时，比如：

```py
u1 = session.query(User).get(5)

query = session.query(Address).filter(Address.user == u1)
```

上述表达式`Address.user == u1`最终编译成一个 SQL 表达式，通常基于`User`对象的主键列，如`"address.user_id = 5"`，使用延迟可调用来在绑定表达式中尽可能晚地检索值`5`。这是为了适应两种情况：`Address.user == u1`表达式可能针对尚未刷新的`User`对象，依赖于服务器生成的主键值，以及即使自表达式创建以来`u1`的主键值已更改，表达式始终返回正确结果的情况。

然而，这种行为的一个副作用是，如果`u1`在表达式被评估时已经过期，就会导致额外的 SELECT 语句，而且如果`u1`也已经从`Session`中分离，就会引发错误：

```py
u1 = session.query(User).get(5)

query = session.query(Address).filter(Address.user == u1)

session.expire(u1)
session.expunge(u1)

query.all()  # <-- would raise DetachedInstanceError
```

当`Session`提交并且`u1`实例超出范围时，对象的过期/清除可能会隐式发生，因为`Address.user == u1`表达式并不强烈引用对象本身，只引用其`InstanceState`。

修复方法是允许 `Address.user == u1` 表达式根据尝试在表达式编译时正常检索或加载值的结果来评估值 `5`，就像现在一样，但如果对象已分离并已过期，则从一个新的机制中检索它 `InstanceState`，该机制将在属性过期时对该状态上的特定属性的最后已知值进行备忘录。此机制仅在表达式功能需要时为特定属性/ `InstanceState` 启用，以节省性能/内存开销。

最初，尝试了诸如立即评估表达式并尝试稍后加载值的各种安排等更简单的方法，但困难的边缘情况是正在更改的列属性（通常是自然主键）的值。为了确保诸如 `Address.user == u1` 的表达式始终返回 `u1` 当前状态的正确答案，它将返回持久对象的当前数据库持久值，如果需要通过 SELECT 查询取消过期，并且对于已分离的对象，它将返回最近已知的值，而不管对象何时使用 `InstanceState` 来跟踪列属性的最后已知值，无论何时属性将被过期。

当无法评估值时，现代属性 API 功能用于指示特定的错误消息，这两种情况是当列属性从未设置时，以及当第一次进行评估时对象已过期时。在所有情况下，不再引发 `DetachedInstanceError`。

[#4359](https://www.sqlalchemy.org/trac/ticket/4359)  ### 多对一替换不会对“raiseload”或“old”对象引发异常

考虑到延迟加载将继续在多对一关系上进行，以加载“old”值，如果关系未指定 `relationship.active_history` 标志，则不会对分离的对象引发断言：

```py
a1 = session.query(Address).filter_by(id=5).one()

session.expunge(a1)

a1.user = some_user
```

上面，在分离的 `a1` 对象上替换 `.user` 属性时，如果属性试图从标识映射中检索 `.user` 的先前值，则会引发 `DetachedInstanceError`。变化在于该操作现在在不加载旧值的情况下继续进行。

同样的改变也适用于`lazy="raise"`加载器策略：

```py
class Address(Base):
    # ...

    user = relationship("User", ..., lazy="raise")
```

以前，对`a1.user`的关联将引发“raiseload”异常，因为属性试图检索先前的值。在加载“旧”值的情况下，现在会跳过此断言。

[#4353](https://www.sqlalchemy.org/trac/ticket/4353)  ### 为 ORM 属性实现了“del”

Python `del`操作实际上对于映射属性（标量列或对象引用）并不可用。已添加支持，使其可以正常工作，其中`del`操作大致等同于将属性设置为`None`值：

```py
some_object = session.query(SomeObject).get(5)

del some_object.some_attribute  # from a SQL perspective, works like "= None"
```

[#4354](https://www.sqlalchemy.org/trac/ticket/4354)  ### info 字典添加到 InstanceState

将`.info`字典添加到`InstanceState`类，这个对象是调用一个映射对象上的`inspect()`而来。这允许自定义配方添加有关对象的其他信息，这些信息将随着对象在内存中的完整生命周期一起传递：

```py
from sqlalchemy import inspect

u1 = User(id=7, name="ed")

inspect(u1).info["user_info"] = "7|ed"
```

[#4257](https://www.sqlalchemy.org/trac/ticket/4257)  ### 水平分片扩展支持批量更新和删除方法

`ShardedQuery`扩展对象支持`Query.update()`和`Query.delete()`批量更新/删除方法。在调用它们时，将咨询`query_chooser`可调用对象，以便根据给定的条件跨多个分片运行更新/删除。

[#4196](https://www.sqlalchemy.org/trac/ticket/4196)

### 关联代理改进

尽管没有任何特定的原因，但本周期的 Association Proxy 扩展有许多改进。

#### 关联代理有新的 cascade_scalar_deletes 标志

给定一个映射为：

```py
class A(Base):
    __tablename__ = "test_a"
    id = Column(Integer, primary_key=True)
    ab = relationship("AB", backref="a", uselist=False)
    b = association_proxy(
        "ab", "b", creator=lambda b: AB(b=b), cascade_scalar_deletes=True
    )

class B(Base):
    __tablename__ = "test_b"
    id = Column(Integer, primary_key=True)
    ab = relationship("AB", backref="b", cascade="all, delete-orphan")

class AB(Base):
    __tablename__ = "test_ab"
    a_id = Column(Integer, ForeignKey(A.id), primary_key=True)
    b_id = Column(Integer, ForeignKey(B.id), primary_key=True)
```

对`A.b`的赋值将生成一个`AB`对象：

```py
a.b = B()
```

`A.b`关联是标量的，并包含一个新标志`AssociationProxy.cascade_scalar_deletes`。设置时，将`A.b`设置为`None`将同时删除`A.ab`。默认行为仍然是保留`a.ab`不变：

```py
a.b = None
assert a.ab is None
```

虽然乍一看这个逻辑应该只看现有关系的“级联”属性，但仅从那个属性本身来看不清楚应该删除代理对象，因此行为被作为一个显式选项提供。

此外，`del`现在对标量的工作方式类似于设置为`None`：

```py
del a.b
assert a.ab is None
```

[#4308](https://www.sqlalchemy.org/trac/ticket/4308)  #### 关联代理在每个类上都存储特定于类的状态

`AssociationProxy` 对象根据其关联的父映射类做出许多决策。虽然 `AssociationProxy` 在历史上起初是一个相对简单的‘getter’，但很快就明显需要它也需要对其引用的属性类型做出决策——比如标量或集合、映射对象或简单值等等。为了实现这一点，它需要检查映射属性或其他引用的描述符或属性，如从其父类引用的那样。然而，在 Python 的描述符机制中，描述符仅在其在该类的上下文中被访问时才了解其“父”类，例如调用 `MyClass.some_descriptor`，这会调用 `__get__()` 方法并传入类。`AssociationProxy` 对象因此会存储特定于该类的状态，但只有在调用此方法后才会这样做；尝试在首先将 `AssociationProxy` 作为描述符之前预先检查此状态将会引发错误。此外，它会假设由 `__get__()` 首先看到的第一个类是它需要了解的唯一父类。尽管事实上，如果特定类具有继承的子类，那么关联代理实际上是代表不止一个父类工作的，尽管没有明确地重新使用它。尽管即使存在这样的缺点，关联代理仍然会通过其当前行为取得相当大的进展，但在某些情况下仍存在缺陷以及确定最佳“所有者”类的复杂问题。

这些问题现在已经在`AssociationProxy`中得到解决，当调用`__get__()`时不再修改其自身的内部状态；相反，每个已知类别都会生成一个名为`AssociationProxyInstance`的新对象，该对象处理特定映射父类的所有状态（当父类未映射时，不会生成`AssociationProxyInstance`）。 单个“拥有类”的概念用于关联代理，尽管在 1.1 中有所改进，但基本上已被替换为现在的方法，其中 AP 现在可以平等地处理任意数量的“拥有”类。

为了适应希望检查此状态的应用程序而不必调用`__get__()`的情况，添加了一个新方法`AssociationProxy.for_class()`，它提供了直接访问特定类别`AssociationProxyInstance`的功能，如下所示：

```py
class User(Base):
    # ...

    keywords = association_proxy("kws", "keyword")

proxy_state = inspect(User).all_orm_descriptors["keywords"].for_class(User)
```

一旦我们有了`AssociationProxyInstance`对象，在上面的示例中存储在`proxy_state`变量中，我们可以查看特定于`User.keywords`代理的属性，例如`target_class`：

```py
>>> proxy_state.target_class
Keyword
```

[#3423](https://www.sqlalchemy.org/trac/ticket/3423)  #### AssociationProxy 现在为基于列的目标提供了标准列操作符

鉴于`AssociationProxy`的目标是数据库列，并且**不是**对象引用或另一个关联代理的情况：

```py
class User(Base):
    # ...

    elements = relationship("Element")

    # column-based association proxy
    values = association_proxy("elements", "value")

class Element(Base):
    # ...

    value = Column(String)
```

`User.values`关联代理指的是`Element.value`列。现在可用标准列操作，例如`like`：

```py
>>> print(s.query(User).filter(User.values.like("%foo%")))
SELECT  "user".id  AS  user_id
FROM  "user"
WHERE  EXISTS  (SELECT  1
FROM  element
WHERE  "user".id  =  element.user_id  AND  element.value  LIKE  :value_1) 
```

`equals`：

```py
>>> print(s.query(User).filter(User.values == "foo"))
SELECT  "user".id  AS  user_id
FROM  "user"
WHERE  EXISTS  (SELECT  1
FROM  element
WHERE  "user".id  =  element.user_id  AND  element.value  =  :value_1) 
```

在与`None`比较时，`IS NULL`表达式增加了一个测试，即相关行根本不存在；这与以前的行为相同：

```py
>>> print(s.query(User).filter(User.values == None))
SELECT  "user".id  AS  user_id
FROM  "user"
WHERE  (EXISTS  (SELECT  1
FROM  element
WHERE  "user".id  =  element.user_id  AND  element.value  IS  NULL))  OR  NOT  (EXISTS  (SELECT  1
FROM  element
WHERE  "user".id  =  element.user_id)) 
```

注意`ColumnOperators.contains()`操作符实际上是一个字符串比较操作符；**这是一个行为上的改变**，之前，关联代理只将`.contains`作为列表包含操作符使用。现在，通过列比较，它现在的行为类似于“like”：

```py
>>> print(s.query(User).filter(User.values.contains("foo")))
SELECT  "user".id  AS  user_id
FROM  "user"
WHERE  EXISTS  (SELECT  1
FROM  element
WHERE  "user".id  =  element.user_id  AND  (element.value  LIKE  '%'  ||  :value_1  ||  '%')) 
```

为了测试`User.values`集合是否包含值`"foo"`，应该使用等于操作符（例如`User.values == 'foo'`）；这在之前的版本中也是有效的。

当使用基于对象的关联代理与集合时，行为与以前相同，即测试集合成员资格，例如给定一个映射：

```py
class User(Base):
    __tablename__ = "user"

    id = Column(Integer, primary_key=True)
    user_elements = relationship("UserElement")

    # object-based association proxy
    elements = association_proxy("user_elements", "element")

class UserElement(Base):
    __tablename__ = "user_element"

    id = Column(Integer, primary_key=True)
    user_id = Column(ForeignKey("user.id"))
    element_id = Column(ForeignKey("element.id"))
    element = relationship("Element")

class Element(Base):
    __tablename__ = "element"

    id = Column(Integer, primary_key=True)
    value = Column(String)
```

`.contains()`方法产生与以前相同的表达式，测试`User.elements`列表中是否存在`Element`对象：

```py
>>> print(s.query(User).filter(User.elements.contains(Element(id=1))))
SELECT "user".id AS user_id
FROM "user"
WHERE EXISTS (SELECT 1
FROM user_element
WHERE "user".id = user_element.user_id AND :param_1 = user_element.element_id)
```

总的来说，这个改变是基于 AssociationProxy stores class-specific state on a per-class basis 的架构变化而启用的；因为代理现在在生成表达式时会衍生出额外的状态，所以`AssociationProxyInstance`类现在有对象目标和列目标版本。

[#4351](https://www.sqlalchemy.org/trac/ticket/4351)

#### 关联代理现在强引用父对象

长期以来，关联代理集合仅保持对父对象的弱引用的行为被恢复；代理现在将保持对父对象的强引用，只要代理集合本身也在内存中，消除了“过时的关联代理”错误。这个改变是基于实验性基础进行的，以查看是否会出现任何导致副作用的用例。

举例来说，给定一个带有关联代理的映射：

```py
class A(Base):
    __tablename__ = "a"

    id = Column(Integer, primary_key=True)
    bs = relationship("B")
    b_data = association_proxy("bs", "data")

class B(Base):
    __tablename__ = "b"
    id = Column(Integer, primary_key=True)
    a_id = Column(ForeignKey("a.id"))
    data = Column(String)

a1 = A(bs=[B(data="b1"), B(data="b2")])

b_data = a1.b_data
```

之前，如果`a1`超出范围被删除：

```py
del a1
```

在`a1`超出范围被删除后尝试迭代`b_data`集合会引发错误“过时的关联代理，父对象已经超出范围”。这是因为关联代理需要访问实际的`a1.bs`集合以产生视图，在这个改变之前，它只保持对`a1`的弱引用。特别是，用户在执行内联操作时经常会遇到这个错误，比如：

```py
collection = session.query(A).filter_by(id=1).first().b_data
```

上面的例子中，因为`A`对象在实际使用`b_data`集合之前已被垃圾回收。

改变是`b_data`集合现在保持对`a1`对象的强引用，以便它保持存在：

```py
assert b_data == ["b1", "b2"]
```

此更改引入的副作用是，如果一个应用程序像上面一样传递集合，**父对象在集合被丢弃之前不会被垃圾回收**。正如以往一样，如果 `a1` 在特定的 `Session` 内是持久化的，它将保持为该会话的状态直到被垃圾回收。

请注意，如果这种变化导致问题，可能会对此进行修订。

[#4268](https://www.sqlalchemy.org/trac/ticket/4268)

#### 为集合，具有 AssociationProxy 的字典实现批量替换

现在，将集合代理分配给集合代理集合的集合应该正常工作，而以前会为现有键重新创建集合代理成员，导致潜在的刷新失败问题，因为删除+插入相同对象，现在应该只在适当的情况下创建新的关联对象：

```py
class A(Base):
    __tablename__ = "test_a"

    id = Column(Integer, primary_key=True)
    b_rel = relationship(
        "B",
        collection_class=set,
        cascade="all, delete-orphan",
    )
    b = association_proxy("b_rel", "value", creator=lambda x: B(value=x))

class B(Base):
    __tablename__ = "test_b"
    __table_args__ = (UniqueConstraint("a_id", "value"),)

    id = Column(Integer, primary_key=True)
    a_id = Column(Integer, ForeignKey("test_a.id"), nullable=False)
    value = Column(String)

# ...

s = Session(e)
a = A(b={"x", "y", "z"})
s.add(a)
s.commit()

# re-assign where one B should be deleted, one B added, two
# B's maintained
a.b = {"x", "z", "q"}

# only 'q' was added, so only one new B object.  previously
# all three would have been re-created leading to flush conflicts
# against the deleted ones.
assert len(s.new) == 1
```

[#2642](https://www.sqlalchemy.org/trac/ticket/2642)  ### 多对一反向引用在删除操作期间检查集合重复项

当一个 ORM 映射的集合存在作为 Python 序列时，通常是 Python `list`，作为 `relationship()` 的默认值，包含重复项，并且对象从其中一个位置被移除但其他位置没有移除时，一个多对一的反向引用会将其属性设置为 `None`，即使一对多的一侧仍然表示对象存在。即使一对多的集合在关系模型中不能有重复项，在内存中使用序列集合的 ORM 映射的 `relationship()` 可以包含其中的重复项，但限制是这种重复状态既不能持久化也不能从数据库中检索。特别是，在列表中临时存在重复项是 Python “swap” 操作的固有特性。考虑到标准的一对多/多对一设置：

```py
class A(Base):
    __tablename__ = "a"

    id = Column(Integer, primary_key=True)
    bs = relationship("B", backref="a")

class B(Base):
    __tablename__ = "b"
    id = Column(Integer, primary_key=True)
    a_id = Column(ForeignKey("a.id"))
```

如果我们有一个带有两个 `B` 成员的 `A` 对象，并执行交换：

```py
a1 = A(bs=[B(), B()])

a1.bs[0], a1.bs[1] = a1.bs[1], a1.bs[0]
```

在上述操作期间，拦截标准的 Python `__setitem__` `__delitem__` 方法提供了一个临时状态，其中第二个 `B()` 对象在集合中出现两次。当从一个位置移除 `B()` 对象时，`B.a` 反向引用将引用设置为 `None`，导致刷新期间删除 `A` 和 `B` 对象之间的链接。同样的问题也可以使用普通重复项演示：

```py
>>> a1 = A()
>>> b1 = B()
>>> a1.bs.append(b1)
>>> a1.bs.append(b1)  # append the same b1 object twice
>>> del a1.bs[1]
>>> a1.bs  # collection is unaffected so far...
[<__main__.B object at 0x7f047af5fb70>]
>>> b1.a  # however b1.a is None
>>>
>>> session.add(a1)
>>> session.commit()  # so upon flush + expire....
>>> a1.bs  # the value is gone
[]
```

修复确保在触发反向引用之前，即在修改集合之前，对集合进行检查，确保目标项的实例数量为零或一，然后取消多对一的一侧，使用线性搜索，目前使用 `list.search` 和 `list.__contains__`。

最初认为在集合内部需要使用基于事件的引用计数方案，以便在整个集合的生命周期中跟踪所有重复实例，这将对所有集合操作产生性能/内存/复杂性影响，包括加载和追加这些非常频繁的操作。相反采取的方法将额外的开销限制在较少常见的集合移除和批量替换操作上，线性扫描的观察开销是可以忽略的；在工作单元中已经使用了与关系绑定集合的线性扫描，以及在集合进行批量替换时。

[#1103](https://www.sqlalchemy.org/trac/ticket/1103)  ### 与 AliasedClass 的关系取代了非主映射器的需求

“非主映射器”是以 Imperative Mapping 风格创建的`Mapper`，它充当已经映射类的另一种可选择的附加映射器。非主映射器起源于 SQLAlchemy 的 0.1、0.2 系列，当时预期`Mapper`对象将是主要的查询构造接口，而`Query`对象还不存在。

随着`Query`的出现，以及后来的`AliasedClass`构造，大多数非主映射器的用例都消失了。这是一件好事，因为 SQLAlchemy 在 0.5 系列左右也完全摆脱了“经典”映射，转而采用了声明式系统。

当意识到一些非常难以定义的`relationship()`配置可能成为可能时，仍然存在一个非主映射器的用例，当一个具有替代可选择项的非主映射器被作为映射目标时，而不是尝试构建一个包含特定对象间关系所有复杂性的`relationship.primaryjoin`。

随着这种使用情况越来越普遍，它的局限性变得明显，包括非主映射器难以配置到可选的添加新列的地方，映射器不继承原始映射的关系，非主映射器上明确配置的关系在加载器选项中表现不佳，非主映射器也不提供可以在查询中使用的基于列的属性的完整功能命名空间（在旧的 0.1 至 0.4 版本中，人们可以直接使用`Table`对象与 ORM）。  

缺失的部分是允许`relationship()`直接引用`AliasedClass`。 `AliasedClass`已经实现了我们希望非主映射器实现的所有功能；它允许从替代选择加载现有映射类，它继承了现有映射器的所有属性和关系，它与加载器选项非常配合，它提供了一个可以像类本身一样混入查询的类似类对象。通过这种改变，以前针对非主映射器的配方在配置关联加入处被改为别名类。  

在关联到别名类，原始的非主映射器如下所示：  

```py
j = join(B, D, D.b_id == B.id).join(C, C.id == D.c_id)

B_viacd = mapper(
    B,
    j,
    non_primary=True,
    primary_key=[j.c.b_id],
    properties={
        "id": j.c.b_id,  # so that 'id' looks the same as before
        "c_id": j.c.c_id,  # needed for disambiguation
        "d_c_id": j.c.d_c_id,  # needed for disambiguation
        "b_id": [j.c.b_id, j.c.d_b_id],
        "d_id": j.c.d_id,
    },
)

A.b = relationship(B_viacd, primaryjoin=A.b_id == B_viacd.c.b_id)
```

这些属性是必需的，以便重新映射附加列，以便它们不与映射到`B`的现有列发生冲突，同时还需要定义一个新的主键。  

采用新方法，所有这些冗长都消失了，并且在建立关系时直接引用附加列：  

```py
j = join(B, D, D.b_id == B.id).join(C, C.id == D.c_id)

B_viacd = aliased(B, j, flat=True)

A.b = relationship(B_viacd, primaryjoin=A.b_id == j.c.b_id)
```

非主映射器现在已经被弃用，最终目标是将经典映射作为一种功能完全取消。声明性 API 将成为映射的唯一手段，这希望能够实现内部改进和简化，以及更清晰的文档编写。  

[#4423](https://www.sqlalchemy.org/trac/ticket/4423)  

### selectin 加载不再对简单的一对多使用 JOIN  

在 1.2 中添加的“selectin”加载功能引入了一种极其高效的新方法来急切加载集合，在许多情况下比“subquery”急切加载要快得多，因为它不依赖于重新声明原始 SELECT 查询，而是使用一个简单的 IN 子句。然而，“selectin”加载仍然依赖于在父表和相关表之间渲染 JOIN，因为它需要行中的父主键值以匹配行。在 1.3 中，添加了一种新的优化，将在简单的一对多加载的最常见情况下省略此 JOIN，其中相关行已经包含了父行的主键值，表达为其外键列。这再次提供了显著的性能改进，因为 ORM 现在可以一次性加载大量集合，而无需使用 JOIN 或子查询。

给定一个映射：

```py
class A(Base):
    __tablename__ = "a"

    id = Column(Integer, primary_key=True)
    bs = relationship("B", lazy="selectin")

class B(Base):
    __tablename__ = "b"
    id = Column(Integer, primary_key=True)
    a_id = Column(ForeignKey("a.id"))
```

在“selectin”加载的 1.2 版本中，从 A 到 B 的加载看起来像：

```py
SELECT  a.id  AS  a_id  FROM  a
SELECT  a_1.id  AS  a_1_id,  b.id  AS  b_id,  b.a_id  AS  b_a_id
FROM  a  AS  a_1  JOIN  b  ON  a_1.id  =  b.a_id
WHERE  a_1.id  IN  (?,  ?,  ?,  ?,  ?,  ?,  ?,  ?,  ?,  ?)  ORDER  BY  a_1.id
(1,  2,  3,  4,  5,  6,  7,  8,  9,  10)
```

使用新行为后，负载看起来像：

```py
SELECT  a.id  AS  a_id  FROM  a
SELECT  b.a_id  AS  b_a_id,  b.id  AS  b_id  FROM  b
WHERE  b.a_id  IN  (?,  ?,  ?,  ?,  ?,  ?,  ?,  ?,  ?,  ?)  ORDER  BY  b.a_id
(1,  2,  3,  4,  5,  6,  7,  8,  9,  10)
```

该行为被释放为自动的，使用了类似于延迟加载的启发式方法，以确定是否可以直接从标识映射中获取相关实体。然而，与大多数查询功能一样，由于涉及多态加载的高级场景，该功能的实现变得更加复杂。如果遇到问题，用户应该报告错误，但是该更改还包括一个标志`relationship.omit_join`，可以在`relationship()`上设置为`False`，以禁用优化。

[#4340](https://www.sqlalchemy.org/trac/ticket/4340)

### 改进多对一查询表达式的行为

当构建一个将多对一关系与对象值进行比较的查询时，例如：

```py
u1 = session.query(User).get(5)

query = session.query(Address).filter(Address.user == u1)
```

上述表达式`Address.user == u1`，最终编译成一个基于`User`对象的主键列的 SQL 表达式，如`"address.user_id = 5"`，使用延迟可调用以在绑定表达式中尽可能晚地检索值`5`。这是为了适应这样一个用例，即`Address.user == u1`表达式可能针对尚未刷新的`User`对象，该对象依赖于服务器生成的主键值，以及该表达式始终返回正确的结果，即使自创建表达式以来`u1`的主键值已更改。

然而，这种行为的一个副作用是，如果在评估表达式时`u1`最终过期，将导致额外的 SELECT 语句，并且在`u1`也从`Session`中分离的情况下，将引发错误：

```py
u1 = session.query(User).get(5)

query = session.query(Address).filter(Address.user == u1)

session.expire(u1)
session.expunge(u1)

query.all()  # <-- would raise DetachedInstanceError
```

当 `Session` 提交并且 `u1` 实例超出范围时，对象的过期 / 删除可能会隐式发生，因为 `Address.user == u1` 表达式不会强烈引用对象本身，而只会引用其`InstanceState`。

修复方法是允许 `Address.user == u1` 表达式根据在表达式编译时尝试正常检索或加载值的基础上评估值 `5`，就像现在一样，但如果对象是分离的并且已过期，则从 `InstanceState`上的新机制中检索，该机制将在该状态上的当属性过期时为该特定属性的最后已知值进行存储。仅当表达式功能需要时，此机制才会为特定属性 / `InstanceState`启用以节省性能 / 内存开销。

最初，尝试了诸如立即评估表达式并在以后尝试加载值时采取各种安排的简单方法，但困难的边缘案例是正在更改的列属性的值（通常是自然主键）的值。为了确保像 `Address.user == u1` 这样的表达式始终返回 `u1` 的当前状态的正确答案，如果需要，它将返回持久对象的当前数据库持久值，通过 SELECT 查询取消到期，并且对于分离的对象，它将返回最近已知的值，无论对象何时被使用新特性将其过期在`InstanceState`中跟踪列属性的最后已知值时。

当值无法评估时，现代属性 API 功能用于指示特定的错误消息，两种情况是当列属性从未设置过时，以及当对象在首次评估时已经过期且现在分离时。在所有情况下，`DetachedInstanceError`不再被引发。

[#4359](https://www.sqlalchemy.org/trac/ticket/4359)

### 多对一替换不会对“raiseload”或“old”对象进行提升

考虑到延迟加载将在多对一关系上进行以加载“old”值的情况，如果关系未指定`relationship.active_history`标志，则不会为分离的对象引发断言：

```py
a1 = session.query(Address).filter_by(id=5).one()

session.expunge(a1)

a1.user = some_user
```

上面，当在分离的 `a1` 对象上替换 `.user` 属性时，会引发 `DetachedInstanceError`，因为属性试图从标识映射中检索 `.user` 的先前值。更改是现在操作会在不加载旧值的情况下继续进行。

对 `lazy="raise"` 加载器策略也进行了相同的更改：

```py
class Address(Base):
    # ...

    user = relationship("User", ..., lazy="raise")
```

以前，`a1.user` 的关联会引发 “raiseload” 异常，因为属性试图检索先前的值。现在在加载 “旧” 值的情况下跳过了此断言。

[#4353](https://www.sqlalchemy.org/trac/ticket/4353)

### 为 ORM 属性实现了 “del”

Python `del` 操作实际上不能用于映射属性，无论是标量列还是对象引用。已经添加了对此的支持，使其能够正常工作，其中 `del` 操作大致相当于将属性设置为 `None` 值：

```py
some_object = session.query(SomeObject).get(5)

del some_object.some_attribute  # from a SQL perspective, works like "= None"
```

[#4354](https://www.sqlalchemy.org/trac/ticket/4354)

### 添加到 InstanceState 的 info 字典

将 `.info` 字典添加到 `InstanceState` 类，该类是通过在映射对象上调用 `inspect()` 获得的。这允许自定义方案为对象添加关于对象的其他信息，该信息将随对象在内存中的完整生命周期一起传递：

```py
from sqlalchemy import inspect

u1 = User(id=7, name="ed")

inspect(u1).info["user_info"] = "7|ed"
```

[#4257](https://www.sqlalchemy.org/trac/ticket/4257)

### 水平分片扩展支持批量更新和删除方法

`ShardedQuery` 扩展对象支持 `Query.update()` 和 `Query.delete()` 批量更新/删除方法。在调用它们时会咨询 `query_chooser` 可调用对象，以便根据给定的条件在多个分片上运行更新/删除操作。

[#4196](https://www.sqlalchemy.org/trac/ticket/4196)

### 协会代理改进

虽然没有特定的原因，但本次周期内协会代理扩展进行了许多改进。

#### 协会代理有新的 cascade_scalar_deletes 标志

给定一个映射如下：

```py
class A(Base):
    __tablename__ = "test_a"
    id = Column(Integer, primary_key=True)
    ab = relationship("AB", backref="a", uselist=False)
    b = association_proxy(
        "ab", "b", creator=lambda b: AB(b=b), cascade_scalar_deletes=True
    )

class B(Base):
    __tablename__ = "test_b"
    id = Column(Integer, primary_key=True)
    ab = relationship("AB", backref="b", cascade="all, delete-orphan")

class AB(Base):
    __tablename__ = "test_ab"
    a_id = Column(Integer, ForeignKey(A.id), primary_key=True)
    b_id = Column(Integer, ForeignKey(B.id), primary_key=True)
```

对 `A.b` 的赋值将生成一个 `AB` 对象：

```py
a.b = B()
```

`A.b` 关联是标量的，并包括一个新标志 `AssociationProxy.cascade_scalar_deletes`。设置了该标志后，将 `A.b` 设置为 `None` 将同时移除 `A.ab`。默认行为仍然是保留 `a.ab` 不变：

```py
a.b = None
assert a.ab is None
```

虽然最初看起来这个逻辑应该只需查看现有关系的“cascade”属性，但仅凭这一点并不清楚代理对象是否应该被移除，因此行为被作为显式选项提供。

另外，`del` 现在对标量的工作方式类似于设置为 `None`：

```py
del a.b
assert a.ab is None
```

[#4308](https://www.sqlalchemy.org/trac/ticket/4308)  #### AssociationProxy 在每个类的基础上存储类特定的状态

`AssociationProxy` 对象根据它关联的父映射类做出许多决策。虽然 `AssociationProxy` 在历史上始作为相对简单的‘getter’，但很早就显而易见它还需要做出关于它所引用的属性类型的决策——例如标量或集合、映射对象或简单值等。为了实现这一点，它需要检查映射属性或其他引用描述符或属性，从其父类中引用。然而，在 Python 描述符机制中，描述符仅在上下文中被访问时才了解其“父”类，例如调用 `MyClass.some_descriptor`，这将调用 `__get__()` 方法，该方法传入类。因此，`AssociationProxy` 对象将存储特定于该类的状态，但只有在首次调用此方法时才会; 在首次访问 `AssociationProxy` 作为描述符之前尝试检查此状态会引发错误。此外，它将假定`__get__()`看到的第一个类是它需要知道的唯一父类。尽管如果一个特定类有继承的子类，协会代理实际上是代表超过一个父类工作的，即使它没有明确地被重新使用。即使有这个缺陷，协会代理仍然可以通过其当前行为取得很大进展，但在某些情况下仍存在缺陷，以及确定最佳“所有者”类的复杂问题。

这些问题现在得到解决，因为当调用 `__get__()` 时，`AssociationProxy` 不再修改自己的内部状态；相反，每个类都生成一个名为 `AssociationProxyInstance` 的新对象，处理特定于特定映射父类的所有状态（当父类未映射时，不会生成 `AssociationProxyInstance`）。关联代理的单一“拥有类”概念，尽管在 1.1 中得到改进，但实质上已被一种方法取代，即 AP 现在可以平等对待任意数量的“拥有”类。

为了适应希望检查此状态的应用程序，而不一定调用 `__get__()` 的 `AssociationProxy`，添加了一个新方法 `AssociationProxy.for_class()`，提供对特定类的 `AssociationProxyInstance` 的直接访问，示例如下：

```py
class User(Base):
    # ...

    keywords = association_proxy("kws", "keyword")

proxy_state = inspect(User).all_orm_descriptors["keywords"].for_class(User)
```

一旦我们有了 `AssociationProxyInstance` 对象，在上面的示例中存储在 `proxy_state` 变量中，我们可以查看特定于 `User.keywords` 代理的��性，比如 `target_class`：

```py
>>> proxy_state.target_class
Keyword
```

[#3423](https://www.sqlalchemy.org/trac/ticket/3423)  #### AssociationProxy 现在为基于列的目标提供标准列运算符

给定一个 `AssociationProxy`，其中目标是数据库列，而**不是**对象引用或另一个关联代理：

```py
class User(Base):
    # ...

    elements = relationship("Element")

    # column-based association proxy
    values = association_proxy("elements", "value")

class Element(Base):
    # ...

    value = Column(String)
```

`User.values` 关联代理指向 `Element.value` 列。现在可以进行标准列操作，比如 `like`：

```py
>>> print(s.query(User).filter(User.values.like("%foo%")))
SELECT  "user".id  AS  user_id
FROM  "user"
WHERE  EXISTS  (SELECT  1
FROM  element
WHERE  "user".id  =  element.user_id  AND  element.value  LIKE  :value_1) 
```

`equals`：

```py
>>> print(s.query(User).filter(User.values == "foo"))
SELECT  "user".id  AS  user_id
FROM  "user"
WHERE  EXISTS  (SELECT  1
FROM  element
WHERE  "user".id  =  element.user_id  AND  element.value  =  :value_1) 
```

当与 `None` 进行比较时，`IS NULL` 表达式会增加一个测试，即相关行根本不存在；这与以前的行为相同：

```py
>>> print(s.query(User).filter(User.values == None))
SELECT  "user".id  AS  user_id
FROM  "user"
WHERE  (EXISTS  (SELECT  1
FROM  element
WHERE  "user".id  =  element.user_id  AND  element.value  IS  NULL))  OR  NOT  (EXISTS  (SELECT  1
FROM  element
WHERE  "user".id  =  element.user_id)) 
```

请注意，`ColumnOperators.contains()`操作实际上是一个字符串比较操作符；**这是行为上的变化**，以前，关联代理仅使用`.contains`作为列表包含操作符。通过基于列的比较，它现在的行为类似于“like”：

```py
>>> print(s.query(User).filter(User.values.contains("foo")))
SELECT  "user".id  AS  user_id
FROM  "user"
WHERE  EXISTS  (SELECT  1
FROM  element
WHERE  "user".id  =  element.user_id  AND  (element.value  LIKE  '%'  ||  :value_1  ||  '%')) 
```

为了测试`User.values`集合是否包含值`"foo"`，应该使用等于操作符（例如`User.values == 'foo'`）；这在以前的版本中也适用。

当使用基于对象的关联代理与集合时，行为与以前相同，即测试集合成员资格，例如给定一个映射：

```py
class User(Base):
    __tablename__ = "user"

    id = Column(Integer, primary_key=True)
    user_elements = relationship("UserElement")

    # object-based association proxy
    elements = association_proxy("user_elements", "element")

class UserElement(Base):
    __tablename__ = "user_element"

    id = Column(Integer, primary_key=True)
    user_id = Column(ForeignKey("user.id"))
    element_id = Column(ForeignKey("element.id"))
    element = relationship("Element")

class Element(Base):
    __tablename__ = "element"

    id = Column(Integer, primary_key=True)
    value = Column(String)
```

`.contains()`方法产生与以前相同的表达式，测试`User.elements`列表中是否存在`Element`对象：

```py
>>> print(s.query(User).filter(User.elements.contains(Element(id=1))))
SELECT "user".id AS user_id
FROM "user"
WHERE EXISTS (SELECT 1
FROM user_element
WHERE "user".id = user_element.user_id AND :param_1 = user_element.element_id)
```

总的来说，这种变化是基于 AssociationProxy stores class-specific state on a per-class basis 的架构变化实现的；因为代理现在在生成表达式时会产生额外的状态，所以`AssociationProxyInstance`类现在有对象目标和列目标版本。

[#4351](https://www.sqlalchemy.org/trac/ticket/4351)

#### 关联代理现在强引用父对象

关联代理集合长期以来只维护对父对象的弱引用的行为被还原；代理现在将在代理集合本身也在内存中的情况下维护对父对象的强引用，消除了“过时的关联代理”错误。这种变化是基于实验性的基础进行的，以查看是否会出现任何导致副作用的用例。

举例来说，给定一个带有关联代理的映射：

```py
class A(Base):
    __tablename__ = "a"

    id = Column(Integer, primary_key=True)
    bs = relationship("B")
    b_data = association_proxy("bs", "data")

class B(Base):
    __tablename__ = "b"
    id = Column(Integer, primary_key=True)
    a_id = Column(ForeignKey("a.id"))
    data = Column(String)

a1 = A(bs=[B(data="b1"), B(data="b2")])

b_data = a1.b_data
```

以前，如果`a1`在作用域外被删除：

```py
del a1
```

在`a1`从作用域中删除后尝试迭代`b_data`集合会引发错误“过时的关联代理，父对象已经超出作用域”。这是因为关联代理需要访问实际的`a1.bs`集合以产生视图，在此更改之前，它只维护对`a1`的弱引用。特别是，用户在执行内联操作时经常会遇到此错误，例如：

```py
collection = session.query(A).filter_by(id=1).first().b_data
```

由于`A`对象在`b_data`集合实际使用之前可能被垃圾回收。

变化在于`b_data`集合现在维护对`a1`对象的强引用，使其保持存在：

```py
assert b_data == ["b1", "b2"]
```

这个改变带来了一个副作用，即如果一个应用程序像上面那样传递集合，**父对象在集合被丢弃之前不会被垃圾回收**。一如既往，如果`a1`在特定的`Session`中是持久的，它将一直保留在该会话的状态中，直到被垃圾回收。

请注意，如果这个改变导致问题，可能会进行修订。

[#4268](https://www.sqlalchemy.org/trac/ticket/4268)

#### 为集合、字典实现了批量替换与 AssociationProxy

将集合或字典分配给关联代理集合现在应该能够正确工作，而以前会为现有键重新创建关联代理成员，导致由于相同对象的删除+插入而导致潜在的刷新失败问题，现在应该只在适当的情况下创建新的关联对象：

```py
class A(Base):
    __tablename__ = "test_a"

    id = Column(Integer, primary_key=True)
    b_rel = relationship(
        "B",
        collection_class=set,
        cascade="all, delete-orphan",
    )
    b = association_proxy("b_rel", "value", creator=lambda x: B(value=x))

class B(Base):
    __tablename__ = "test_b"
    __table_args__ = (UniqueConstraint("a_id", "value"),)

    id = Column(Integer, primary_key=True)
    a_id = Column(Integer, ForeignKey("test_a.id"), nullable=False)
    value = Column(String)

# ...

s = Session(e)
a = A(b={"x", "y", "z"})
s.add(a)
s.commit()

# re-assign where one B should be deleted, one B added, two
# B's maintained
a.b = {"x", "z", "q"}

# only 'q' was added, so only one new B object.  previously
# all three would have been re-created leading to flush conflicts
# against the deleted ones.
assert len(s.new) == 1
```

[#2642](https://www.sqlalchemy.org/trac/ticket/2642)  #### 关联代理新增了新的 cascade_scalar_deletes 标志

给定一个映射如下：

```py
class A(Base):
    __tablename__ = "test_a"
    id = Column(Integer, primary_key=True)
    ab = relationship("AB", backref="a", uselist=False)
    b = association_proxy(
        "ab", "b", creator=lambda b: AB(b=b), cascade_scalar_deletes=True
    )

class B(Base):
    __tablename__ = "test_b"
    id = Column(Integer, primary_key=True)
    ab = relationship("AB", backref="b", cascade="all, delete-orphan")

class AB(Base):
    __tablename__ = "test_ab"
    a_id = Column(Integer, ForeignKey(A.id), primary_key=True)
    b_id = Column(Integer, ForeignKey(B.id), primary_key=True)
```

对`A.b`的赋值将生成一个`AB`对象：

```py
a.b = B()
```

`A.b`关联是标量的，并包括一个新标志`AssociationProxy.cascade_scalar_deletes`。当设置时，将`A.b`设置为`None`将同时移除`A.ab`。默认行为仍然是保留`a.ab`在原地：

```py
a.b = None
assert a.ab is None
```

起初似乎很直观的是，这个逻辑应该只查看现有关系的“级联”属性，但仅仅从那个属性本身并不清楚代理对象是否应该被移除，因此行为被作为一个明确的选项提供。

另外，`del`现在对标量的操作方式与设置为`None`类似：

```py
del a.b
assert a.ab is None
```

[#4308](https://www.sqlalchemy.org/trac/ticket/4308)

#### 关联代理在每个类上存储特定于类的状态

`AssociationProxy` 对象基于其关联的父映射类做出许多决策。虽然 `AssociationProxy` 在历史上最初是一个相对简单的“getter”，但很快就明显地需要做出关于其引用的属性类型的决策——比如标量或集合、映射对象或简单值等。为了实现这一点，它需要检查映射属性或其他引用描述符或属性，这些都是从其父类引用的。然而，在 Python 描述符机制中，描述符只有在在其“父”类的上下文中被访问时才会了解其“父”类，比如调用 `MyClass.some_descriptor`，这会调用 `__get__()` 方法并传递类。因此，`AssociationProxy` 对象将存储特定于该类的状态，但只有在调用此方法后才会这样；在未首先将 `AssociationProxy` 作为描述符访问的情况下尝试检查此状态将引发错误。此外，它会假定 `__get__()` 看到的第一个类就是它需要了解的唯一父类。尽管如果一个特定类有继承的子类，关联代理实际上是代表不止一个父类工作，即使没有明确重用。尽管即使有这个缺点，关联代理仍然可以通过其当前行为取得相当大的进展，但在某些情况下仍存在缺陷，以及确定最佳“所有者”类的复杂问题。

现在这些问题已经解决了，因为在调用 `__get__()` 时，`AssociationProxy` 不再修改自己的内部状态；相反，针对每个类生成一个新对象，称为 `AssociationProxyInstance`，该对象处理与特定映射的父类相关的所有状态（当父类未映射时，不会生成 `AssociationProxyInstance`）。一个关联代理的单一“拥有类”的概念，尽管在 1.1 中得到了改进，但基本上已被一种方法取代，即 AP 现在可以平等对待任意数量的“拥有”类。

为了适应那些想要检查此状态的应用，而不一定调用 `__get__()` 的应用程序，添加了一个新方法 `AssociationProxy.for_class()`，它提供了对特定类的 `AssociationProxyInstance` 的直接访问，如下所示：

```py
class User(Base):
    # ...

    keywords = association_proxy("kws", "keyword")

proxy_state = inspect(User).all_orm_descriptors["keywords"].for_class(User)
```

一旦我们拥有 `AssociationProxyInstance` 对象，在上面的示例中存储在 `proxy_state` 变量中，我们可以查看特定于 `User.keywords` 代理的属性，例如 `target_class`：

```py
>>> proxy_state.target_class
Keyword
```

[#3423](https://www.sqlalchemy.org/trac/ticket/3423)

#### 关联代理现在为基于列的目标提供标准列操作符

给定一个 `AssociationProxy`，其中目标是数据库列，并且**不是**对象引用或另一个关联代理：

```py
class User(Base):
    # ...

    elements = relationship("Element")

    # column-based association proxy
    values = association_proxy("elements", "value")

class Element(Base):
    # ...

    value = Column(String)
```

`User.values` 关联代理指的是 `Element.value` 列。现在可使用标准列操作，例如 `like`：

```py
>>> print(s.query(User).filter(User.values.like("%foo%")))
SELECT  "user".id  AS  user_id
FROM  "user"
WHERE  EXISTS  (SELECT  1
FROM  element
WHERE  "user".id  =  element.user_id  AND  element.value  LIKE  :value_1) 
```

`equals`:

```py
>>> print(s.query(User).filter(User.values == "foo"))
SELECT  "user".id  AS  user_id
FROM  "user"
WHERE  EXISTS  (SELECT  1
FROM  element
WHERE  "user".id  =  element.user_id  AND  element.value  =  :value_1) 
```

当与 `None` 比较时，`IS NULL` 表达式会增加一个测试，即相关行根本不存在；这与以前的行为相同：

```py
>>> print(s.query(User).filter(User.values == None))
SELECT  "user".id  AS  user_id
FROM  "user"
WHERE  (EXISTS  (SELECT  1
FROM  element
WHERE  "user".id  =  element.user_id  AND  element.value  IS  NULL))  OR  NOT  (EXISTS  (SELECT  1
FROM  element
WHERE  "user".id  =  element.user_id)) 
```

请注意，`ColumnOperators.contains()` 操作符实际上是一个字符串比较操作符；**这是一个行为上的改变**，之前，关联代理仅将 `.contains` 用作列表包含操作符。通过基于列的比较，它现在的行为类似于“like”：

```py
>>> print(s.query(User).filter(User.values.contains("foo")))
SELECT  "user".id  AS  user_id
FROM  "user"
WHERE  EXISTS  (SELECT  1
FROM  element
WHERE  "user".id  =  element.user_id  AND  (element.value  LIKE  '%'  ||  :value_1  ||  '%')) 
```

为了测试 `User.values` 集合是否包含值 `"foo"`，应该使用等于操作符（例如 `User.values == 'foo'`）；这在之前的版本中也适用。

当使用基于对象的关联代理与集合时，行为与以前相同，即测试集合成员资格，例如给定一个映射：

```py
class User(Base):
    __tablename__ = "user"

    id = Column(Integer, primary_key=True)
    user_elements = relationship("UserElement")

    # object-based association proxy
    elements = association_proxy("user_elements", "element")

class UserElement(Base):
    __tablename__ = "user_element"

    id = Column(Integer, primary_key=True)
    user_id = Column(ForeignKey("user.id"))
    element_id = Column(ForeignKey("element.id"))
    element = relationship("Element")

class Element(Base):
    __tablename__ = "element"

    id = Column(Integer, primary_key=True)
    value = Column(String)
```

`.contains()` 方法产生与之前相同的表达式，测试 `User.elements` 列表中是否存在 `Element` 对象：

```py
>>> print(s.query(User).filter(User.elements.contains(Element(id=1))))
SELECT "user".id AS user_id
FROM "user"
WHERE EXISTS (SELECT 1
FROM user_element
WHERE "user".id = user_element.user_id AND :param_1 = user_element.element_id)
```

总的来说，这个改变是基于关联代理在每个类的基础上存储特定于类的状态的架构变化而启用的；由于代理现在在生成表达式时会产生额外的状态，`AssociationProxyInstance` 类现在有对象目标和列目标版本。

[#4351](https://www.sqlalchemy.org/trac/ticket/4351)

#### 关联代理现在强引用父对象

关联代理集合长期维持对父对象的弱引用的行为被撤销；代理现在将在代理集合本身也在内存中的情况下维持对父对象的强引用，消除了“过时的关联代理”错误。这个改变是基于试验性基础进行的，以查看是否会出现任何导致副作用的用例。

举例来说，给定一个带有关联代理的映射：

```py
class A(Base):
    __tablename__ = "a"

    id = Column(Integer, primary_key=True)
    bs = relationship("B")
    b_data = association_proxy("bs", "data")

class B(Base):
    __tablename__ = "b"
    id = Column(Integer, primary_key=True)
    a_id = Column(ForeignKey("a.id"))
    data = Column(String)

a1 = A(bs=[B(data="b1"), B(data="b2")])

b_data = a1.b_data
```

以前，如果 `a1` 超出范围被删除：

```py
del a1
```

在 `a1` 从范围中删除后尝试迭代 `b_data` 集合会引发错误 `"过时的关联代理，父对象已超出范围"`。这是因为关联代理需要访问实际的 `a1.bs` 集合以生成视图，在此改变之前，它只维持对 `a1` 的弱引用。特别是，用户在执行内联操作时经常会遇到这个错误：

```py
collection = session.query(A).filter_by(id=1).first().b_data
```

上面的情况是因为 `A` 对象在 `b_data` 集合实际使用之前会被垃圾回收。

变化在于 `b_data` 集合现在维持对 `a1` 对象的强引用，使其保持存在：

```py
assert b_data == ["b1", "b2"]
```

这种改变引入了一个副作用，即如果应用程序像上面那样传递集合，**父对象在集合被丢弃之前不会被垃圾回收**。一如既往，如果`a1`在特定的`Session`内是持久的，它将一直保留在该会话的状态中，直到被垃圾回收。

注意，如果这种改变导致问题，可能会进行修订。

[#4268](https://www.sqlalchemy.org/trac/ticket/4268)

#### 使用 AssociationProxy 为集合实现批量替换的功能

现在，将集合分配给关联代理集合应该可以正常工作，而以前会为现有键重新创建关联代理成员，导致由于删除+插入相同对象而导致潜在刷新失败的问题，现在应该只在适当的情况下创建新的关联对象：

```py
class A(Base):
    __tablename__ = "test_a"

    id = Column(Integer, primary_key=True)
    b_rel = relationship(
        "B",
        collection_class=set,
        cascade="all, delete-orphan",
    )
    b = association_proxy("b_rel", "value", creator=lambda x: B(value=x))

class B(Base):
    __tablename__ = "test_b"
    __table_args__ = (UniqueConstraint("a_id", "value"),)

    id = Column(Integer, primary_key=True)
    a_id = Column(Integer, ForeignKey("test_a.id"), nullable=False)
    value = Column(String)

# ...

s = Session(e)
a = A(b={"x", "y", "z"})
s.add(a)
s.commit()

# re-assign where one B should be deleted, one B added, two
# B's maintained
a.b = {"x", "z", "q"}

# only 'q' was added, so only one new B object.  previously
# all three would have been re-created leading to flush conflicts
# against the deleted ones.
assert len(s.new) == 1
```

[#2642](https://www.sqlalchemy.org/trac/ticket/2642)

### 多对一反向引用在移除操作期间检查集合中的重复项

当 ORM 映射的集合作为 Python 序列存在时，通常是 Python `list`，这是`relationship()`的默认值，包含重复项，并且对象从一个位置被移除但未从其他位置移除时，一个多对一的反向引用会将其属性设置为`None`，即使一对多方仍然表示对象存在。尽管一对多集合在关系模型中不能有重复项，但使用序列集合的 ORM 映射的`relationship()`在内存中可以有重复项，但这些重复状态既不能持久化也不能从数据库中检索。特别是，在列表中临时存在重复项是 Python“交换”操作的固有特性。考虑一个标准的一对多/多对一设置：

```py
class A(Base):
    __tablename__ = "a"

    id = Column(Integer, primary_key=True)
    bs = relationship("B", backref="a")

class B(Base):
    __tablename__ = "b"
    id = Column(Integer, primary_key=True)
    a_id = Column(ForeignKey("a.id"))
```

如果我们有一个具有两个`B`成员的`A`对象，并执行交换：

```py
a1 = A(bs=[B(), B()])

a1.bs[0], a1.bs[1] = a1.bs[1], a1.bs[0]
```

在上述操作期间，拦截标准 Python `__setitem__` `__delitem__`方法会提供一个中间状态，其中第二个`B()`对象在集合中出现两次。当`B()`对象从一个位置移除时，`B.a`反向引用会将引用设置为`None`，导致在刷新期间删除`A`和`B`对象之间的链接。相同的问题也可以使用普通重复项来演示：

```py
>>> a1 = A()
>>> b1 = B()
>>> a1.bs.append(b1)
>>> a1.bs.append(b1)  # append the same b1 object twice
>>> del a1.bs[1]
>>> a1.bs  # collection is unaffected so far...
[<__main__.B object at 0x7f047af5fb70>]
>>> b1.a  # however b1.a is None
>>>
>>> session.add(a1)
>>> session.commit()  # so upon flush + expire....
>>> a1.bs  # the value is gone
[]
```

修复确保在反向引用触发之前，在集合被改变之前，检查集合中是否恰好有一个或零个目标项的实例，然后在取消多对一方面时使用线性搜索，目前使用`list.search`和`list.__contains__`。

最初人们认为在集合内部需要使用基于事件的引用计数方案，以便在整个集合的生命周期内跟踪所有重复的实例，这将对所有集合操作产生性能/内存/复杂性影响，包括非常频繁的加载和追加操作。取而代之的方法是将额外的开销限制在较不常见的集合移除和批量替换操作上，并且观察到的线性扫描开销可以忽略不计；关系绑定集合的线性扫描已经在工作单元中使用，以及在集合被批量替换时已经被使用。

[#1103](https://www.sqlalchemy.org/trac/ticket/1103)

## 关键行为变化 - ORM

### Query.join() 更明确地处理决定“左”侧的歧义

从历史上看，给定以下查询：

```py
u_alias = aliased(User)
session.query(User, u_alias).join(Address)
```

鉴于标准教程映射，查询将生成一个 FROM 子句，如下所示：

```py
SELECT  ...
FROM  users  AS  users_1,  users  JOIN  addresses  ON  users.id  =  addresses.user_id
```

也就是说，JOIN 会隐式地针对第一个匹配的实体。新的行为是要求解决这种模糊性的异常：

```py
sqlalchemy.exc.InvalidRequestError: Can't determine which FROM clause to
join from, there are multiple FROMS which can join to this entity.
Try adding an explicit ON clause to help resolve the ambiguity.
```

解决方案是提供一个 ON 子句，可以是一个表达式：

```py
# join to User
session.query(User, u_alias).join(Address, Address.user_id == User.id)

# join to u_alias
session.query(User, u_alias).join(Address, Address.user_id == u_alias.id)
```

或者使用关系属性，如果可用的话：

```py
# join to User
session.query(User, u_alias).join(Address, User.addresses)

# join to u_alias
session.query(User, u_alias).join(Address, u_alias.addresses)
```

这个变化包括，如果连接是非模糊的，那么现在连接可以正确地链接到不是列表中第一个元素的 FROM 子句：

```py
session.query(func.current_timestamp(), User).join(Address)
```

在这项增强之前，上述查询将引发：

```py
sqlalchemy.exc.InvalidRequestError: Don't know how to join from
CURRENT_TIMESTAMP; please use select_from() to establish the
left entity/selectable of this join
```

现在查询可以正常工作了：

```py
SELECT  CURRENT_TIMESTAMP  AS  current_timestamp_1,  users.id  AS  users_id,
users.name  AS  users_name,  users.fullname  AS  users_fullname,
users.password  AS  users_password
FROM  users  JOIN  addresses  ON  users.id  =  addresses.user_id
```

总的来说，这种变化直接符合 Python 的“显式优于隐式”的哲学。

[#4365](https://www.sqlalchemy.org/trac/ticket/4365)  ### FOR UPDATE 子句在联合贪婪加载子查询中以及外部呈现

此更改特别适用于使用`joinedload()`加载策略与行限制查询相结合，例如使用`Query.first()`或`Query.limit()`，以及使用`Query.with_for_update()`方法。

给定一个查询：

```py
session.query(A).options(joinedload(A.b)).limit(5)
```

当联合贪婪加载与 LIMIT 结合时，`Query`对象呈现如下形式的 SELECT：

```py
SELECT  subq.a_id,  subq.a_data,  b_alias.id,  b_alias.data  FROM  (
  SELECT  a.id  AS  a_id,  a.data  AS  a_data  FROM  a  LIMIT  5
)  AS  subq  LEFT  OUTER  JOIN  b  ON  subq.a_id=b.a_id
```

这样，对主要实体的行限制就会发生，而不会影响到关联项的贪婪加载。当上述查询与“SELECT..FOR UPDATE”结合时，行为是这样的：

```py
SELECT  subq.a_id,  subq.a_data,  b_alias.id,  b_alias.data  FROM  (
  SELECT  a.id  AS  a_id,  a.data  AS  a_data  FROM  a  LIMIT  5
)  AS  subq  LEFT  OUTER  JOIN  b  ON  subq.a_id=b.a_id  FOR  UPDATE
```

然而，由于[`bugs.mysql.com/bug.php?id=90693`](https://bugs.mysql.com/bug.php?id=90693)，MySQL 不会锁定子查询中的行，不像 PostgreSQL 和其他数据库。因此，上述查询现在呈现为：

```py
SELECT  subq.a_id,  subq.a_data,  b_alias.id,  b_alias.data  FROM  (
  SELECT  a.id  AS  a_id,  a.data  AS  a_data  FROM  a  LIMIT  5  FOR  UPDATE
)  AS  subq  LEFT  OUTER  JOIN  b  ON  subq.a_id=b.a_id  FOR  UPDATE
```

在 Oracle 方言中，内部的“FOR UPDATE”不会呈现，因为 Oracle 不支持这种语法，方言会跳过针对子查询的任何“FOR UPDATE”；在任何情况下都不是必要的，因为 Oracle 像 PostgreSQL 一样正确锁定返回行的所有元素。

当在`Query.with_for_update.of`修饰符上使用时，通常在 PostgreSQL 上，外部的“FOR UPDATE”被省略，OF 现在在内部呈现；以前，OF 目标不会被转换以正确适应子查询。因此，考虑到：

```py
session.query(A).options(joinedload(A.b)).with_for_update(of=A).limit(5)
```

现在查询将呈现为：

```py
SELECT  subq.a_id,  subq.a_data,  b_alias.id,  b_alias.data  FROM  (
  SELECT  a.id  AS  a_id,  a.data  AS  a_data  FROM  a  LIMIT  5  FOR  UPDATE  OF  a
)  AS  subq  LEFT  OUTER  JOIN  b  ON  subq.a_id=b.a_id
```

上述形式应该对 PostgreSQL 有所帮助，因为 PostgreSQL 不允许在 LEFT OUTER JOIN 目标之后呈现 FOR UPDATE 子句。

总的来说，对于正在使用的目标数据库，FOR UPDATE 仍然非常具体，不容易推广到更复杂的查询。

[#4246](https://www.sqlalchemy.org/trac/ticket/4246)  ### passive_deletes=’all’将使 FK 对于从集合中移除的对象保持不变

`relationship.passive_deletes`选项接受值`"all"`，表示在刷新对象时不应修改任何外键属性，即使关系的集合/引用已被移除。以前，在以下情况下，这种情况并不适用于一对多或一对一关系：

```py
class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True)
    addresses = relationship("Address", passive_deletes="all")

class Address(Base):
    __tablename__ = "addresses"
    id = Column(Integer, primary_key=True)
    email = Column(String)

    user_id = Column(Integer, ForeignKey("users.id"))
    user = relationship("User")

u1 = session.query(User).first()
address = u1.addresses[0]
u1.addresses.remove(address)
session.commit()

# would fail and be set to None
assert address.user_id == u1.id
```

修复现在包括`address.user_id`根据`passive_deletes="all"`保持不变。这种情况对于构建自定义“版本表”方案等非常有用，其中行被归档而不是被删除。

[#3844](https://www.sqlalchemy.org/trac/ticket/3844)  ### Query.join()更明确地处理决定“左”侧的模棱两可性

历史上，给定如下查询：

```py
u_alias = aliased(User)
session.query(User, u_alias).join(Address)
```

鉴于标准教程映射，查询将生成一个 FROM 子句：

```py
SELECT  ...
FROM  users  AS  users_1,  users  JOIN  addresses  ON  users.id  =  addresses.user_id
```

这意味着 JOIN 将隐式地针对第一个匹配的实体。新行为是，异常请求解决这种模棱两可的情况：

```py
sqlalchemy.exc.InvalidRequestError: Can't determine which FROM clause to
join from, there are multiple FROMS which can join to this entity.
Try adding an explicit ON clause to help resolve the ambiguity.
```

解决方案是提供一个 ON 子句，可以是一个表达式：

```py
# join to User
session.query(User, u_alias).join(Address, Address.user_id == User.id)

# join to u_alias
session.query(User, u_alias).join(Address, Address.user_id == u_alias.id)
```

或者使用关系属性，如果可用的话：

```py
# join to User
session.query(User, u_alias).join(Address, User.addresses)

# join to u_alias
session.query(User, u_alias).join(Address, u_alias.addresses)
```

更改包括现在 JOIN 可以正确地链接到不是列表中第一个元素的 FROM 子句，如果 JOIN 是非模棱两可的话：

```py
session.query(func.current_timestamp(), User).join(Address)
```

在此增强之前，上述查询将引发：

```py
sqlalchemy.exc.InvalidRequestError: Don't know how to join from
CURRENT_TIMESTAMP; please use select_from() to establish the
left entity/selectable of this join
```

现在查询正常工作：

```py
SELECT  CURRENT_TIMESTAMP  AS  current_timestamp_1,  users.id  AS  users_id,
users.name  AS  users_name,  users.fullname  AS  users_fullname,
users.password  AS  users_password
FROM  users  JOIN  addresses  ON  users.id  =  addresses.user_id
```

总的来说，这种变化直接符合 Python 的“显式优于隐式”的哲学。

[#4365](https://www.sqlalchemy.org/trac/ticket/4365)

### FOR UPDATE 子句在连接贪婪加载子查询内部以及外部都被渲染

此更改特别适用于使用`joinedload()`加载策略与行限制查询结合使用时，例如使用`Query.first()`或`Query.limit()`，以及使用`Query.with_for_update()`方法。

给定一个查询如下：

```py
session.query(A).options(joinedload(A.b)).limit(5)
```

当连接贪婪加载与 LIMIT 结合时，`Query`对象会渲染以下形式的 SELECT：

```py
SELECT  subq.a_id,  subq.a_data,  b_alias.id,  b_alias.data  FROM  (
  SELECT  a.id  AS  a_id,  a.data  AS  a_data  FROM  a  LIMIT  5
)  AS  subq  LEFT  OUTER  JOIN  b  ON  subq.a_id=b.a_id
```

这样做是为了使主实体的行限制生效，而不影响相关项目的连接贪婪加载。当上述查询与“SELECT..FOR UPDATE”结合时，行为如下：

```py
SELECT  subq.a_id,  subq.a_data,  b_alias.id,  b_alias.data  FROM  (
  SELECT  a.id  AS  a_id,  a.data  AS  a_data  FROM  a  LIMIT  5
)  AS  subq  LEFT  OUTER  JOIN  b  ON  subq.a_id=b.a_id  FOR  UPDATE
```

然而，由于[`bugs.mysql.com/bug.php?id=90693`](https://bugs.mysql.com/bug.php?id=90693)，MySQL 不会锁定子查询内的行，不像 PostgreSQL 和其他数据库。因此，上述查询现在会渲染为：

```py
SELECT  subq.a_id,  subq.a_data,  b_alias.id,  b_alias.data  FROM  (
  SELECT  a.id  AS  a_id,  a.data  AS  a_data  FROM  a  LIMIT  5  FOR  UPDATE
)  AS  subq  LEFT  OUTER  JOIN  b  ON  subq.a_id=b.a_id  FOR  UPDATE
```

在 Oracle 方言中，内部的“FOR UPDATE”不会被渲染，因为 Oracle 不支持这种语法，方言会跳过任何针对子查询的“FOR UPDATE”；在任何情况下都不是必要的，因为 Oracle 像 PostgreSQL 一样正确地锁定了返回行的所有元素。

当在`Query.with_for_update.of`修饰符上使用时，通常在 PostgreSQL 上，外部的“FOR UPDATE”会被省略，OF 现在会在内部被渲染；以前，OF 目标不会被转换以适应子查询。所以给定：

```py
session.query(A).options(joinedload(A.b)).with_for_update(of=A).limit(5)
```

查询现在会渲染为：

```py
SELECT  subq.a_id,  subq.a_data,  b_alias.id,  b_alias.data  FROM  (
  SELECT  a.id  AS  a_id,  a.data  AS  a_data  FROM  a  LIMIT  5  FOR  UPDATE  OF  a
)  AS  subq  LEFT  OUTER  JOIN  b  ON  subq.a_id=b.a_id
```

以上形式在 PostgreSQL 上也应该有所帮助，因为 PostgreSQL 不允许在 LEFT OUTER JOIN 目标之后渲染 FOR UPDATE 子句。

总的来说，FOR UPDATE 对于正在使用的目标数据库非常具体，不能轻易地推广到更复杂的查询。

[#4246](https://www.sqlalchemy.org/trac/ticket/4246)

### passive_deletes='all'将使从集合中移除的对象的 FK 保持不变

`relationship.passive_deletes`选项接受值`"all"`，表示在刷新对象时不应修改任何外键属性，即使关系的集合/引用已被移除。以前，在以下情况下，这不会发生在一对多或一对一关系中：

```py
class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True)
    addresses = relationship("Address", passive_deletes="all")

class Address(Base):
    __tablename__ = "addresses"
    id = Column(Integer, primary_key=True)
    email = Column(String)

    user_id = Column(Integer, ForeignKey("users.id"))
    user = relationship("User")

u1 = session.query(User).first()
address = u1.addresses[0]
u1.addresses.remove(address)
session.commit()

# would fail and be set to None
assert address.user_id == u1.id
```

修复现在包括`address.user_id`按照`passive_deletes="all"`不变。这种情况对于构建自定义“版本表”方案等非常有用，其中行被归档而不是删除。

[#3844](https://www.sqlalchemy.org/trac/ticket/3844)

## 新功能和改进 - 核心

### 新的多列命名约定标记，长名称截断

为了适应`MetaData`命名约定需要区分多列约束并希望在生成的约束名称中使用所有列的情况，添加了一系列新的命名约定标记，包括`column_0N_name`、`column_0_N_name`、`column_0N_key`、`column_0_N_key`、`referred_column_0N_name`、`referred_column_0_N_name`等，它们将所有列的列名（或键或标签）连接在一起，没有分隔符或使用下划线分隔符。下面我们定义一个约定，将`UniqueConstraint`约束命名为连接所有列名称的名称：

```py
metadata_obj = MetaData(
    naming_convention={"uq": "uq_%(table_name)s_%(column_0_N_name)s"}
)

table = Table(
    "info",
    metadata_obj,
    Column("a", Integer),
    Column("b", Integer),
    Column("c", Integer),
    UniqueConstraint("a", "b", "c"),
)
```

上述表的 CREATE TABLE 将呈现为：

```py
CREATE  TABLE  info  (
  a  INTEGER,
  b  INTEGER,
  c  INTEGER,
  CONSTRAINT  uq_info_a_b_c  UNIQUE  (a,  b,  c)
)
```

此外，现在将长名称截断逻辑应用于命名约定生成的名称，特别是为了适应可能产生非常长名称的多列标签。这种逻辑与用于截断 SELECT 语句中的长标签名称的逻辑相同，用一个确定性生成的 4 字符哈希替换超过目标数据库标识符长度限制的多余字符。例如，在 PostgreSQL 上，标识符不能超过 63 个字符，长约束名称通常从下面的表定义中生成：

```py
long_names = Table(
    "long_names",
    metadata_obj,
    Column("information_channel_code", Integer, key="a"),
    Column("billing_convention_name", Integer, key="b"),
    Column("product_identifier", Integer, key="c"),
    UniqueConstraint("a", "b", "c"),
)
```

截断逻辑将确保不会为 UNIQUE 约束生成过长的名称：

```py
CREATE  TABLE  long_names  (
  information_channel_code  INTEGER,
  billing_convention_name  INTEGER,
  product_identifier  INTEGER,
  CONSTRAINT  uq_long_names_information_channel_code_billing_conventi_a79e
  UNIQUE  (information_channel_code,  billing_convention_name,  product_identifier)
)
```

上述后缀`a79e`基于长名称的 md5 哈希，并且每次生成相同的值，以产生给定模式的一致名称。

注意，当约束名在给定方言中显式过大时，截断逻辑也会引发`IdentifierError`。这已经是`Index`对象的行为很长时间了，但现在也适用于其他类型的约束：

```py
from sqlalchemy import Column
from sqlalchemy import Integer
from sqlalchemy import MetaData
from sqlalchemy import Table
from sqlalchemy import UniqueConstraint
from sqlalchemy.dialects import postgresql
from sqlalchemy.schema import AddConstraint

m = MetaData()
t = Table("t", m, Column("x", Integer))
uq = UniqueConstraint(
    t.c.x,
    name="this_is_too_long_of_a_name_for_any_database_backend_even_postgresql",
)

print(AddConstraint(uq).compile(dialect=postgresql.dialect()))
```

将输出：

```py
sqlalchemy.exc.IdentifierError: Identifier
'this_is_too_long_of_a_name_for_any_database_backend_even_postgresql'
exceeds maximum length of 63 characters
```

异常引发阻止了由数据库后端截断的非确定性约束名称的生成，这些名称后来与数据库迁移不兼容。

要将 SQLAlchemy 端的截断规则应用于上述标识符，请使用`conv()`构造：

```py
uq = UniqueConstraint(
    t.c.x,
    name=conv("this_is_too_long_of_a_name_for_any_database_backend_even_postgresql"),
)
```

这将再次输出确定性截断的 SQL，如下所示：

```py
ALTER  TABLE  t  ADD  CONSTRAINT  this_is_too_long_of_a_name_for_any_database_backend_eve_ac05  UNIQUE  (x)
```

目前还没有选项可以使名称传递以允许数据库端截断。这对于`Index`名称已经有一段时间了，而且并没有提出问题。

此更改还修复了另外两个问题。一个是`column_0_key`标记虽然被记录，但却无法使用，另一个是如果这两个值不同，`referred_column_0_name`标记会错误地呈现`.key`而不是`.name`列。

另请参阅

配置约束命名约定

`MetaData.naming_convention`

[#3989](https://www.sqlalchemy.org/trac/ticket/3989)  ### 用于 SQL 函数的二进制比较解释

这个增强功能是在核心层实现的，但主要适用于 ORM。

现在可以将比较两个元素的 SQL 函数用作“比较”对象，适用于 ORM `relationship()`的用法，首先像往常一样使用`func`工厂创建函数，然后在函数完成时调用`FunctionElement.as_comparison()`修饰符来生成一个具有“左”和“右”两侧的`BinaryExpression`：

```py
class Venue(Base):
    __tablename__ = "venue"
    id = Column(Integer, primary_key=True)
    name = Column(String)

    descendants = relationship(
        "Venue",
        primaryjoin=func.instr(remote(foreign(name)), name + "/").as_comparison(1, 2)
        == 1,
        viewonly=True,
        order_by=name,
    )
```

上面，“后代”关系的`relationship.primaryjoin`将基于传递给`instr()`的第一个和第二个参数产生一个“左”和一个“右”表达式。这使得 ORM 懒加载等功能能够产生 SQL，例如：

```py
SELECT  venue.id  AS  venue_id,  venue.name  AS  venue_name
FROM  venue
WHERE  instr(venue.name,  (?  ||  ?))  =  ?  ORDER  BY  venue.name
('parent1',  '/',  1)
```

以及一个 joinedload，例如：

```py
v1 = (
    s.query(Venue)
    .filter_by(name="parent1")
    .options(joinedload(Venue.descendants))
    .one()
)
```

工作原理如下：

```py
SELECT  venue.id  AS  venue_id,  venue.name  AS  venue_name,
  venue_1.id  AS  venue_1_id,  venue_1.name  AS  venue_1_name
FROM  venue  LEFT  OUTER  JOIN  venue  AS  venue_1
  ON  instr(venue_1.name,  (venue.name  ||  ?))  =  ?
WHERE  venue.name  =  ?  ORDER  BY  venue_1.name
('/',  1,  'parent1')
```

该功能预期将有助于处理诸如在关系连接条件中使用几何函数或任何通过 SQL 函数来表达 SQL 连接的 ON 子句等情况。

[#3831](https://www.sqlalchemy.org/trac/ticket/3831)  ### 扩展 IN 特性现在支持空列表

版本 1.2 中引入的“扩展 IN”功能现在支持传递给`ColumnOperators.in_()`运算符的空列表。对于空列表的实现将产生一个特定于目标后端的“空集”表达式，例如对于 PostgreSQL，“SELECT CAST(NULL AS INTEGER) WHERE 1!=1”，对于 MySQL，“SELECT 1 FROM (SELECT 1) as _empty_set WHERE 1!=1”：

```py
>>> from sqlalchemy import create_engine
>>> from sqlalchemy import select, literal_column, bindparam
>>> e = create_engine("postgresql://scott:tiger@localhost/test", echo=True)
>>> with e.connect() as conn:
...     conn.execute(
...         select([literal_column("1")]).where(
...             literal_column("1").in_(bindparam("q", expanding=True))
...         ),
...         q=[],
...     )
{exexsql}SELECT 1 WHERE 1 IN (SELECT CAST(NULL AS INTEGER) WHERE 1!=1)
```

该功能还适用于基于元组的 IN 语句，其中“空 IN”表达式将被扩展以支持元组中给定的元素，例如在 PostgreSQL 上：

```py
>>> from sqlalchemy import create_engine
>>> from sqlalchemy import select, literal_column, tuple_, bindparam
>>> e = create_engine("postgresql://scott:tiger@localhost/test", echo=True)
>>> with e.connect() as conn:
...     conn.execute(
...         select([literal_column("1")]).where(
...             tuple_(50, "somestring").in_(bindparam("q", expanding=True))
...         ),
...         q=[],
...     )
{exexsql}SELECT 1 WHERE (%(param_1)s, %(param_2)s)
IN (SELECT CAST(NULL AS INTEGER), CAST(NULL AS VARCHAR) WHERE 1!=1)
```

[#4271](https://www.sqlalchemy.org/trac/ticket/4271)  ### TypeEngine 方法 bind_expression、column_expression 与 Variant、特定类型一起工作

当`TypeEngine.bind_expression()`和`TypeEngine.column_expression()`方法存在于特定数据类型的“impl”上时，这些方法现在可以被方言使用，也可以用于`TypeDecorator`和`Variant`的用例。

以下示例说明了一个`TypeDecorator`，它将 SQL 时间转换函数应用于`LargeBinary`。为了使此类型在`Variant`的上下文中工作，编译器需要深入“impl”变体表达式以定位这些方法：

```py
from sqlalchemy import TypeDecorator, LargeBinary, func

class CompressedLargeBinary(TypeDecorator):
    impl = LargeBinary

    def bind_expression(self, bindvalue):
        return func.compress(bindvalue, type_=self)

    def column_expression(self, col):
        return func.uncompress(col, type_=self)

MyLargeBinary = LargeBinary().with_variant(CompressedLargeBinary(), "sqlite")
```

上述表达式仅在 SQLite 上使用时会呈现为 SQL 中的函数：

```py
from sqlalchemy import select, column
from sqlalchemy.dialects import sqlite

print(select([column("x", CompressedLargeBinary)]).compile(dialect=sqlite.dialect()))
```

将呈现：

```py
SELECT  uncompress(x)  AS  x
```

此更改还包括方言可以在方言级别的实现类型上实现`TypeEngine.bind_expression()`和`TypeEngine.column_expression()`，它们现在将被使用；特别是这将用于 MySQL 的新“二进制前缀”要求以及用于将 MySQL 的十进制绑定值转换为浮点数。

[#3981](https://www.sqlalchemy.org/trac/ticket/3981)  ### QueuePool 的新后进先出策略

通常由 `create_engine()` 使用的连接池称为 `QueuePool`。此连接池使用类似于 Python 内置的 `Queue` 类的对象来存储等待使用的数据库连接。`Queue` 具有先进先出的行为，旨在提供对持续在池中的数据库连接的循环使用。然而，这样做的一个潜在缺点是，当池的利用率低时，池中的每个连接的串行重复使用意味着试图减少未使用连接的服务器端超时策略被阻止关闭这些连接。为了适应这种情况，添加了一个新的标志 `create_engine.pool_use_lifo` ，它将 `Queue` 的 `.get()` 方法反转，以从队列的开头而不是末尾获取连接，从而实质上将“队列”变为“栈”（考虑到这样做会增加一个称为 `StackPool` 的全新连接池，但这太啰嗦了）。

另见

使用 FIFO vs. LIFO  ### 新的多列命名约定标记，长名称截断

为了适应需要通过 `MetaData` 命名约定消除多列约束的歧义，并希望在生成的约束名称中使用所有列的情况，添加了一系列新的命名约定标记，包括 `column_0N_name`、`column_0_N_name`、`column_0N_key`、`column_0_N_key`、`referred_column_0N_name`、`referred_column_0_N_name` 等，这些标记将约束中的所有列的列名（或键或标签）连接在一起，可以是没有分隔符或带有下划线分隔符。下面我们定义一个约定，将会以将所有列的名称连接在一起的方式命名 `UniqueConstraint` 约束：

```py
metadata_obj = MetaData(
    naming_convention={"uq": "uq_%(table_name)s_%(column_0_N_name)s"}
)

table = Table(
    "info",
    metadata_obj,
    Column("a", Integer),
    Column("b", Integer),
    Column("c", Integer),
    UniqueConstraint("a", "b", "c"),
)
```

上述表的 CREATE TABLE 将呈现为：

```py
CREATE  TABLE  info  (
  a  INTEGER,
  b  INTEGER,
  c  INTEGER,
  CONSTRAINT  uq_info_a_b_c  UNIQUE  (a,  b,  c)
)
```

此外，现在对通过命名约定生成的名称应用长名称截断逻辑，特别是为了适应可能产生非常长名称的多列标签。这个逻辑与在 SELECT 语句中截断长标签名称所使用的逻辑相同，它用一个确定性生成的 4 字符哈希替换了超过目标数据库标识符长度限制的多余字符。例如，在 PostgreSQL 中，标识符不能超过 63 个字符，长约束名通常是从下面的表定义生成的：

```py
long_names = Table(
    "long_names",
    metadata_obj,
    Column("information_channel_code", Integer, key="a"),
    Column("billing_convention_name", Integer, key="b"),
    Column("product_identifier", Integer, key="c"),
    UniqueConstraint("a", "b", "c"),
)
```

截断逻辑将确保不会为 UNIQUE 约束生成过长的名称：

```py
CREATE  TABLE  long_names  (
  information_channel_code  INTEGER,
  billing_convention_name  INTEGER,
  product_identifier  INTEGER,
  CONSTRAINT  uq_long_names_information_channel_code_billing_conventi_a79e
  UNIQUE  (information_channel_code,  billing_convention_name,  product_identifier)
)
```

上述后缀`a79e`基于长名称的 md5 哈希，并且每次生成相同的值，以为给定的模式生成一致的名称。

请注意，当约束名称显式过大时，截断逻辑还会引发`IdentifierError`。这已经是`Index`对象的行为很长时间了，但现在也适用于其他类型的约束：

```py
from sqlalchemy import Column
from sqlalchemy import Integer
from sqlalchemy import MetaData
from sqlalchemy import Table
from sqlalchemy import UniqueConstraint
from sqlalchemy.dialects import postgresql
from sqlalchemy.schema import AddConstraint

m = MetaData()
t = Table("t", m, Column("x", Integer))
uq = UniqueConstraint(
    t.c.x,
    name="this_is_too_long_of_a_name_for_any_database_backend_even_postgresql",
)

print(AddConstraint(uq).compile(dialect=postgresql.dialect()))
```

输出将是：

```py
sqlalchemy.exc.IdentifierError: Identifier
'this_is_too_long_of_a_name_for_any_database_backend_even_postgresql'
exceeds maximum length of 63 characters
```

异常抛出可防止由数据库后端截断的不确定性约束名称的生成，这些名称随后与数据库迁移不兼容。

要将 SQLAlchemy 端的截断规则应用于上述标识符，请使用`conv()`构造：

```py
uq = UniqueConstraint(
    t.c.x,
    name=conv("this_is_too_long_of_a_name_for_any_database_backend_even_postgresql"),
)
```

这将再次输出确定性截断的 SQL，如下所示：

```py
ALTER  TABLE  t  ADD  CONSTRAINT  this_is_too_long_of_a_name_for_any_database_backend_eve_ac05  UNIQUE  (x)
```

目前尚无选项可使名称通过以允许数据库端截断。这在一段时间内已经适用于`Index`名称，并且并未引起问题。

这一变更还修复了其他两个问题。其中一个是`column_0_key`令牌虽然已记录，但却不可用，另一个是如果这两个值不同，`referred_column_0_name`令牌会意外地呈现`.key`而不是`.name`。

另请参阅

配置约束命名约定

`MetaData.naming_convention`

[#3989](https://www.sqlalchemy.org/trac/ticket/3989)

### SQL 函数的二进制比较解释

此增强功能是在核心级别实现的，但主要适用于 ORM。

现在可以将比较两个元素的 SQL 函数用作适用于 ORM `relationship()`中的“比较”对象，首先使用`func`工厂通常创建该函数，然后当函数完成时调用`FunctionElement.as_comparison()`修饰符，以生成具有“左”和“右”两侧的`BinaryExpression`：

```py
class Venue(Base):
    __tablename__ = "venue"
    id = Column(Integer, primary_key=True)
    name = Column(String)

    descendants = relationship(
        "Venue",
        primaryjoin=func.instr(remote(foreign(name)), name + "/").as_comparison(1, 2)
        == 1,
        viewonly=True,
        order_by=name,
    )
```

上面，“descendants”关系的`relationship.primaryjoin`将基于传递给`instr()`的第一个和第二个参数产生一个“left”和一个“right”表达式。这允许 ORM lazyload 等功能生成类似以下的 SQL：

```py
SELECT  venue.id  AS  venue_id,  venue.name  AS  venue_name
FROM  venue
WHERE  instr(venue.name,  (?  ||  ?))  =  ?  ORDER  BY  venue.name
('parent1',  '/',  1)
```

以及一个 joinedload，例如：

```py
v1 = (
    s.query(Venue)
    .filter_by(name="parent1")
    .options(joinedload(Venue.descendants))
    .one()
)
```

作为：

```py
SELECT  venue.id  AS  venue_id,  venue.name  AS  venue_name,
  venue_1.id  AS  venue_1_id,  venue_1.name  AS  venue_1_name
FROM  venue  LEFT  OUTER  JOIN  venue  AS  venue_1
  ON  instr(venue_1.name,  (venue.name  ||  ?))  =  ?
WHERE  venue.name  =  ?  ORDER  BY  venue_1.name
('/',  1,  'parent1')
```

这个功能预计将有助于处理诸如在关系连接条件中使用几何函数或任何 ON 子句以 SQL 函数形式表达的情况。

[#3831](https://www.sqlalchemy.org/trac/ticket/3831)

### 扩展 IN 功能现在支持空列表

在版本 1.2 中引入的“扩展 IN”功能现在支持传递给`ColumnOperators.in_()`运算符的空列表。对于空列表的实现将产生一个特定于目标后端的“空集合”表达式，例如对于 PostgreSQL，“SELECT CAST(NULL AS INTEGER) WHERE 1!=1”，对于 MySQL，“SELECT 1 FROM (SELECT 1) as _empty_set WHERE 1!=1”：

```py
>>> from sqlalchemy import create_engine
>>> from sqlalchemy import select, literal_column, bindparam
>>> e = create_engine("postgresql://scott:tiger@localhost/test", echo=True)
>>> with e.connect() as conn:
...     conn.execute(
...         select([literal_column("1")]).where(
...             literal_column("1").in_(bindparam("q", expanding=True))
...         ),
...         q=[],
...     )
{exexsql}SELECT 1 WHERE 1 IN (SELECT CAST(NULL AS INTEGER) WHERE 1!=1)
```

该功能还适用于基于元组的 IN 语句，其中“空 IN”表达式将被扩展以支持元组中给定的元素，例如在 PostgreSQL 上：

```py
>>> from sqlalchemy import create_engine
>>> from sqlalchemy import select, literal_column, tuple_, bindparam
>>> e = create_engine("postgresql://scott:tiger@localhost/test", echo=True)
>>> with e.connect() as conn:
...     conn.execute(
...         select([literal_column("1")]).where(
...             tuple_(50, "somestring").in_(bindparam("q", expanding=True))
...         ),
...         q=[],
...     )
{exexsql}SELECT 1 WHERE (%(param_1)s, %(param_2)s)
IN (SELECT CAST(NULL AS INTEGER), CAST(NULL AS VARCHAR) WHERE 1!=1)
```

[#4271](https://www.sqlalchemy.org/trac/ticket/4271)

### TypeEngine 方法 bind_expression、column_expression 与 Variant、特定类型一起工作

`TypeEngine.bind_expression()`和`TypeEngine.column_expression()`方法现在在特定数据类型的“impl”上存在时也能工作，允许这些方法被方言以及`TypeDecorator`和`Variant`用例使用。

以下示例说明了一个`TypeDecorator`，它将 SQL 时间转换函数应用于`LargeBinary`。为了使这种类型在`Variant`的上下文中工作，编译器需要深入“impl”变体表达式以定位这些方法：

```py
from sqlalchemy import TypeDecorator, LargeBinary, func

class CompressedLargeBinary(TypeDecorator):
    impl = LargeBinary

    def bind_expression(self, bindvalue):
        return func.compress(bindvalue, type_=self)

    def column_expression(self, col):
        return func.uncompress(col, type_=self)

MyLargeBinary = LargeBinary().with_variant(CompressedLargeBinary(), "sqlite")
```

上述表达式仅在 SQLite 上使用时会在 SQL 中呈现一个函数：

```py
from sqlalchemy import select, column
from sqlalchemy.dialects import sqlite

print(select([column("x", CompressedLargeBinary)]).compile(dialect=sqlite.dialect()))
```

将呈现：

```py
SELECT  uncompress(x)  AS  x
```

此更改还包括方言可以在方言级别的实现类型上实现`TypeEngine.bind_expression()`和`TypeEngine.column_expression()`，在那里它们现在将被使用；特别是这将用于 MySQL 的新“二进制前缀”要求以及用于将 MySQL 的十进制绑定值转换的情况。

[#3981](https://www.sqlalchemy.org/trac/ticket/3981)

### QueuePool 的新后进先出策略

通常由`create_engine()`使用的连接池被称为`QueuePool`。此池使用一个类似于 Python 内置的`Queue`类的对象来存储等待使用的数据库连接。`Queue`具有先进先出的行为，旨在提供对持久在池中的数据库连接的循环使用。然而，这种方法的一个潜在缺点是，当池的利用率较低时，池中每个连接的串行重复使用意味着试图减少未使用连接的服务器端超时策略被阻止关闭这些连接。为了适应这种用例，添加了一个新标志`create_engine.pool_use_lifo`，它将`Queue`的`.get()`方法反转，从队列的开头而不是末尾获取连接，从本质上将“队列”变成“栈”（考虑到添加一个名为`StackPool`的全新池，但这太啰嗦了）。

另请参阅

使用 FIFO vs. LIFO

## 核心关键变化

### 完全删除将字符串 SQL 片段强制转换为 text() 

首次在版本 1.0 中添加的警告，描述在将完整 SQL 片段强制转换为 text()时发出的警告，现在已转换为异常。对于像`Query.filter()`和`Select.order_by()`等方法传递的字符串片段自动转换为`text()`构造的持续关注，尽管这已发出警告。在`Select.order_by()`、`Query.order_by()`、`Select.group_by()`和`Query.group_by()`的情况下，字符串标签或列名仍然解析为相应的表达式构造，但如果解析失败，则会引发`CompileError`，从而防止直接呈现原始 SQL 文本。

[#4481](https://www.sqlalchemy.org/trac/ticket/4481)  ### “线程本地”引擎策略已弃用

“线程本地引擎策略”是在 SQLAlchemy 0.2 左右添加的，作为解决 SQLAlchemy 0.1 中操作的标准方式的问题的解决方案，可以总结为“线程本地一切”，发现存在不足。回顾起来，似乎相当荒谬，SQLAlchemy 的首次发布在各个方面都是“alpha”，却担心太多用户已经定居在现有 API 上，无法简单地更改它。

SQLAlchemy 的原始用法模型如下：

```py
engine.begin()

table.insert().execute(parameters)
result = table.select().execute()

table.update().execute(parameters)

engine.commit()
```

在几个月的实际使用后，很明显，假装“连接”或“事务”是一个隐藏的实现细节是一个坏主意，特别是当有人需要同时处理多个数据库连接时。因此，我们今天看到的使用范式被引入，减去了上下文管理器，因为它们在 Python 中尚不存在：

```py
conn = engine.connect()
try:
    trans = conn.begin()

    conn.execute(table.insert(), parameters)
    result = conn.execute(table.select())

    conn.execute(table.update(), parameters)

    trans.commit()
except:
    trans.rollback()
    raise
finally:
    conn.close()
```

上述范式是人们所需要的，但由于它仍然有点冗长（因为没有上下文管理器），因此保留了旧的工作方式，并成为线程本地引擎策略。

今天，使用 Core 更加简洁，甚至比原始模式更加简洁，这要归功于上下文管理器：

```py
with engine.begin() as conn:
    conn.execute(table.insert(), parameters)
    result = conn.execute(table.select())

    conn.execute(table.update(), parameters)
```

此时，仍依赖“threadlocal”风格的任何剩余代码将通过此弃用来鼓励现代化 - 该功能应在下一个主要系列的 SQLAlchemy 中完全移除，例如 1.4 版。连接池参数`Pool.use_threadlocal`也已弃用，因为在大多数情况下实际上没有任何效果，`Engine.contextual_connect()`方法也是如此，该方法通常与`Engine.connect()`方法是同义词，除非使用了 threadlocal 引擎。

[#4393](https://www.sqlalchemy.org/trac/ticket/4393)  ### convert_unicode 参数已弃用

参数`String.convert_unicode`和`create_engine.convert_unicode`已弃用。这些参数的目的是指示 SQLAlchemy 确保在 Python 2 下传递给数据库之前对传入的 Python Unicode 对象进行编码为字节字符串，并期望从数据库返回的字节字符串转换回 Python Unicode 对象。在 Python 3 之前的时代，要做到这一点是一件非常艰巨的事情，因为几乎所有 Python DBAPI 默认情况下都没有启用 Unicode 支持，并且大多数都存在与它们提供的 Unicode 扩展相关的主要问题。最终，SQLAlchemy 添加了 C 扩展，这些扩展的主要目的之一是加快结果集中的 Unicode 解码过程。

一旦引入了 Python 3，DBAPI 开始更全面地支持 Unicode，并且更重要的是，默认情况下支持 Unicode。然而，特定 DBAPI 在何种条件下会或不会从结果返回 Unicode 数据，以及接受 Python Unicode 值作为参数的条件仍然非常复杂。这标志着“convert_unicode”标志开始过时，因为它们不再足以确保仅在需要时进行编码/解码，而不是在不需要时进行。相反，“convert_unicode”开始被 dialects 自动检测。这一部分可以在引擎第一次连接时发出的“SELECT ‘test plain returns’”和“SELECT ‘test_unicode_returns’”SQL 中看到；方言正在测试当前 DBAPI 及其当前设置和后端数据库连接是否默认返回 Unicode。

结果是，不再需要在任何情况下使用“convert_unicode”标志，如果需要，SQLAlchemy 项目需要知道这些情况及其原因。目前，在所有主要数据库上，使用该标志的 Unicode 往返测试通过了数百次，因此相当有把握地认为它们不再需要，除非是在争议性的非使用情况，例如访问来自传统数据库的错误编码数据，最好使用自定义类型。

[#4393](https://www.sqlalchemy.org/trac/ticket/4393)  ### 完全移除将字符串 SQL 片段强制转换为 text()

在 1.0 版本中首次添加的警告，描述在将完整 SQL 片段强制转换为 text() 时发出的警告，现已转换为异常。对于将字符串片段传递给诸如`Query.filter()` 和 `Select.order_by()`等方法的自动转换成`text()` 构造的情况仍然存在持续的担忧，尽管已发出警告。对于`Select.order_by()`、`Query.order_by()`、`Select.group_by()` 和 `Query.group_by()`，字符串标签或列名仍然解析为相应的表达式构造，但如果解析失败，则引发`CompileError`，从而防止原始 SQL 文本直接呈现。

[#4481](https://www.sqlalchemy.org/trac/ticket/4481)

### “threadlocal” 引擎策略已弃用

“threadlocal 引擎策略” 是在 SQLAlchemy 0.2 左右添加的，作为 SQLAlchemy 0.1 中标准操作方式的解决方案，这种方式可以总结为“threadlocal everything”，但后来发现存在不足之处。回顾起来，很荒谬的是，即使 SQLAlchemy 的第一个版本在各个方面都是“alpha”版本，仍然担心已有太多用户已经习惯了现有的 API 以至于不能轻易更改。

SQLAlchemy 的原始使用模型如下：

```py
engine.begin()

table.insert().execute(parameters)
result = table.select().execute()

table.update().execute(parameters)

engine.commit()
```

经过几个月的实际使用，很明显，假装“连接”或“事务”是一个隐藏的实现细节是一个坏主意，特别是当有人需要同时处理多个数据库连接时。因此，我们今天看到的使用范式被引入，减去了上下文管理器，因为它们在 Python 中尚不存在：

```py
conn = engine.connect()
try:
    trans = conn.begin()

    conn.execute(table.insert(), parameters)
    result = conn.execute(table.select())

    conn.execute(table.update(), parameters)

    trans.commit()
except:
    trans.rollback()
    raise
finally:
    conn.close()
```

上述范式是人们所需要的，但由于仍然有点啰嗦（因为没有上下文管理器），旧的工作方式也被保留下来，并成为线程本地引擎策略。

今天，使用 Core 要简洁得多，甚至比原始模式更简洁，这要归功于上下文管理器：

```py
with engine.begin() as conn:
    conn.execute(table.insert(), parameters)
    result = conn.execute(table.select())

    conn.execute(table.update(), parameters)
```

此时，任何仍然依赖“threadlocal”风格的代码都将通过此弃用被鼓励进行现代化 - 该功能应在下一个主要的 SQLAlchemy 系列（例如 1.4）中完全移除。连接池参数`Pool.use_threadlocal`也被弃用，因为在大多数情况下实际上没有任何效果，`Engine.contextual_connect()`方法也是如此，该方法通常与`Engine.connect()`方法是同义的，除非使用线程本地引擎。

[#4393](https://www.sqlalchemy.org/trac/ticket/4393)

### convert_unicode 参数已弃用

参数`String.convert_unicode`和`create_engine.convert_unicode`已被弃用。这些参数的目的是指示 SQLAlchemy 在将 Python 2 中的传入 Unicode 对象传递到数据库之前确保对其进行字节串编码，并期望从数据库接收字节串并将其转换回 Python Unicode 对象。在 Python 3 之前的时代，要做到这一点是一个巨大的挑战，因为几乎所有的 Python DBAPI 默认情况下都没有启用 Unicode 支持，并且大多数都存在与其提供的 Unicode 扩展相关的主要问题。最终，SQLAlchemy 添加了 C 扩展，其中这些扩展的主要目的之一是加速结果集中的 Unicode 解码过程。

一旦引入了 Python 3，DBAPI 开始更全面地支持 Unicode，更重要的是，默认情况下支持 Unicode。然而，特定 DBAPI 是否返回 Unicode 数据以及接受 Python Unicode 值作为参数的条件仍然非常复杂。这标志着“convert_unicode”标志开始过时，因为它们不再足以确保仅在需要时进行编码/解码，而不是在不需要时进行。相反，“convert_unicode”开始由方言自动检测。可以从引擎第一次连接时发出的 SQL “SELECT ‘test plain returns’” 和 “SELECT ‘test_unicode_returns’” 中看到其中一部分；方言正在测试当前 DBAPI 与其当前设置和后端数据库连接是否默认返回 Unicode。

最终结果是，在任何情况下，用户对“convert_unicode”标志的使用都不再需要，并且如果需要，SQLAlchemy 项目需要知道这些情况以及原因。当前，在所有主要数据库上，通过了数百个 Unicode 往返测试，而不使用此标志，因此相当有信心不再需要它们，除非在可争议的非使用情况下，例如访问来自遗留数据库的错误编码数据，此时最好使用自定义类型。

[#4393](https://www.sqlalchemy.org/trac/ticket/4393)

## 方言改进和更改 - PostgreSQL

### 为 PostgreSQL 分区表添加了基本的反射支持

SQLAlchemy 可以在 PostgreSQL CREATE TABLE 语句中使用 `postgresql_partition_by` 标志渲染“PARTITION BY”序列，该标志在版本 1.2.6 中添加。然而，`'p'` 类型直到现在都不是反射查询的一部分。

给定这样一个模式：

```py
dv = Table(
    "data_values",
    metadata_obj,
    Column("modulus", Integer, nullable=False),
    Column("data", String(30)),
    postgresql_partition_by="range(modulus)",
)

sa.event.listen(
    dv,
    "after_create",
    sa.DDL(
        "CREATE TABLE data_values_4_10 PARTITION OF data_values "
        "FOR VALUES FROM (4) TO (10)"
    ),
)
```

两个表名 `'data_values'` 和 `'data_values_4_10'` 将通过 `Inspector.get_table_names()` 返回，并且列也将通过 `Inspector.get_columns('data_values')` 和 `Inspector.get_columns('data_values_4_10')` 返回。这也适用于对这些表使用 `Table(..., autoload=True)`。

[#4237](https://www.sqlalchemy.org/trac/ticket/4237)  ### 为 PostgreSQL 分区表添加了基本的反射支持

SQLAlchemy 可以在 PostgreSQL CREATE TABLE 语句中使用 `postgresql_partition_by` 标志渲染“PARTITION BY”序列，该标志在版本 1.2.6 中添加。然而，`'p'` 类型直到现在都不是反射查询的一部分。

给定这样一个模式：

```py
dv = Table(
    "data_values",
    metadata_obj,
    Column("modulus", Integer, nullable=False),
    Column("data", String(30)),
    postgresql_partition_by="range(modulus)",
)

sa.event.listen(
    dv,
    "after_create",
    sa.DDL(
        "CREATE TABLE data_values_4_10 PARTITION OF data_values "
        "FOR VALUES FROM (4) TO (10)"
    ),
)
```

两个表名`'data_values'`和`'data_values_4_10'`将从`Inspector.get_table_names()`返回，并且还将从`Inspector.get_columns('data_values')`以及`Inspector.get_columns('data_values_4_10')`返回列。这也适用于使用这些表的`Table(..., autoload=True)`。

[#4237](https://www.sqlalchemy.org/trac/ticket/4237)

## 方言改进和变化 - MySQL

### 协议级别的 ping 现在用于预先 ping

包括 mysqlclient、python-mysql、PyMySQL 和 mysql-connector-python 在内的 MySQL 方言现在使用`connection.ping()`方法进行池预 ping 功能，详细信息请参见断开处理 - 悲观。这比以前在连接上发出“SELECT 1”的方法要轻量得多。  ### 控制 ON DUPLICATE KEY UPDATE 中参数顺序

`ON DUPLICATE KEY UPDATE`子句中 UPDATE 参数的顺序现在可以通过传递一个 2 元组列表来明确排序：

```py
from sqlalchemy.dialects.mysql import insert

insert_stmt = insert(my_table).values(id="some_existing_id", data="inserted value")

on_duplicate_key_stmt = insert_stmt.on_duplicate_key_update(
    [
        ("data", "some data"),
        ("updated_at", func.current_timestamp()),
    ],
)
```

参见

INSERT…ON DUPLICATE KEY UPDATE (Upsert)  ### 协议级别的 ping 现在用于预先 ping

包括 mysqlclient、python-mysql、PyMySQL 和 mysql-connector-python 在内的 MySQL 方言现在使用`connection.ping()`方法进行池预 ping 功能，详细信息请参见断开处理 - 悲观。这比以前在连接上发出“SELECT 1”的方法要轻量得多。

### 控制 ON DUPLICATE KEY UPDATE 中参数顺序

`ON DUPLICATE KEY UPDATE`子句中 UPDATE 参数的顺序现在可以通过传递一个 2 元组列表来明确排序：

```py
from sqlalchemy.dialects.mysql import insert

insert_stmt = insert(my_table).values(id="some_existing_id", data="inserted value")

on_duplicate_key_stmt = insert_stmt.on_duplicate_key_update(
    [
        ("data", "some data"),
        ("updated_at", func.current_timestamp()),
    ],
)
```

参见

INSERT…ON DUPLICATE KEY UPDATE (Upsert)

## 方言改进和变化 - SQLite

### 对 SQLite JSON 的支持已添加

添加了一个新的数据类型`JSON`，它代表了`JSON`基础数据类型的 SQLite 的 json 成员访问函数。实现使用 SQLite 的`JSON_EXTRACT`和`JSON_QUOTE`函数来提供基本的 JSON 支持。

请注意，数据库中呈现的数据类型本身的名称是“JSON”。这将创建一个带有“numeric”亲和力的 SQLite 数据类型，通常情况下不应该成为问题，除非是由单个整数值组成的 JSON 值的情况。尽管如此，根据 SQLite 自己文档中的示例[`www.sqlite.org/json1.html`](https://www.sqlite.org/json1.html)，名称 JSON 正在被用于其熟悉性。

[#3850](https://www.sqlalchemy.org/trac/ticket/3850)  ### 增加对 SQLite 约束中 ON CONFLICT 的支持

SQLite 支持一个非标准的 ON CONFLICT 子句，可以为独立约束以及一些列内约束（如 NOT NULL）指定。通过向诸如`UniqueConstraint`之类的对象添加`sqlite_on_conflict`关键字以及几个`Column` -特定的变体：

```py
some_table = Table(
    "some_table",
    metadata_obj,
    Column("id", Integer, primary_key=True, sqlite_on_conflict_primary_key="FAIL"),
    Column("data", Integer),
    UniqueConstraint("id", "data", sqlite_on_conflict="IGNORE"),
)
```

上述表在 CREATE TABLE 语句中呈现为：

```py
CREATE  TABLE  some_table  (
  id  INTEGER  NOT  NULL,
  data  INTEGER,
  PRIMARY  KEY  (id)  ON  CONFLICT  FAIL,
  UNIQUE  (id,  data)  ON  CONFLICT  IGNORE
)
```

另请参阅

ON CONFLICT 支持约束

[#4360](https://www.sqlalchemy.org/trac/ticket/4360)  ### 增加对 SQLite JSON 的支持

添加了一个新的数据类型`JSON`，该数据类型代表`JSON`基本数据类型的 SQLite 的 json 成员访问函数。该实现使用 SQLite 的`JSON_EXTRACT`和`JSON_QUOTE`函数来提供基本的 JSON 支持。

注意，数据库中呈现的数据类型本身的名称是“JSON”。这将创建一个带有“数字”亲和力的 SQLite 数据类型，这通常不应该是问题，除非 JSON 值仅包含单个整数值。尽管如此，根据 SQLite 自己文档中的示例，[`www.sqlite.org/json1.html`](https://www.sqlite.org/json1.html)中使用了 JSON 这个名字以保持熟悉性。

[#3850](https://www.sqlalchemy.org/trac/ticket/3850)

### 增加对 SQLite 约束中 ON CONFLICT 的支持

SQLite 支持一个非标准的 ON CONFLICT 子句，可以为独立约束以及一些列内约束（如 NOT NULL）指定。通过向诸如`UniqueConstraint`之类的对象添加`sqlite_on_conflict`关键字以及几个`Column` -特定的变体，已为这些子句添加了支持：

```py
some_table = Table(
    "some_table",
    metadata_obj,
    Column("id", Integer, primary_key=True, sqlite_on_conflict_primary_key="FAIL"),
    Column("data", Integer),
    UniqueConstraint("id", "data", sqlite_on_conflict="IGNORE"),
)
```

上述表在 CREATE TABLE 语句中呈现为：

```py
CREATE  TABLE  some_table  (
  id  INTEGER  NOT  NULL,
  data  INTEGER,
  PRIMARY  KEY  (id)  ON  CONFLICT  FAIL,
  UNIQUE  (id,  data)  ON  CONFLICT  IGNORE
)
```

另请参阅

ON CONFLICT 支持约束

[#4360](https://www.sqlalchemy.org/trac/ticket/4360)

## 方言改进和更改 - Oracle

### 对于通用 unicode，国家字符数据类型被减弱，可通过选项重新启用

现在，默认情况下，`Unicode` 和 `UnicodeText` 数据类型现在对应于 Oracle 上的 `VARCHAR2` 和 `CLOB` 数据类型，而不是 `NVARCHAR2` 和 `NCLOB`（也称为“国家”字符集类型）。这将在诸如它们在`CREATE TABLE`语句中的呈现方式等行为中看到，以及当使用`Unicode`或`UnicodeText`绑定参数时，不会传递任何类型对象给`setinputsizes()`；cx_Oracle 会原生处理字符串值。这种变化基于 cx_Oracle 的维护者的建议，即 Oracle 中的“国家”数据类型在很大程度上已经过时且性能不佳。它们还会在某些情况下干扰，比如应用于像`trunc()`这样的函数的格式说明符时。

当数据库不使用符合 Unicode 标准的字符集时，可能需要使用`NVARCHAR2`和相关类型的情况。在这种情况下，可以将标志`use_nchar_for_unicode`传递给`create_engine()`以重新启用旧行为。

如往常一样，明确使用`NVARCHAR2`和`NCLOB`数据类型将继续使用`NVARCHAR2`和`NCLOB`，包��在 DDL 中以及处理带有 cx_Oracle 的`setinputsizes()`的绑定参数时。

在读取方面，在 Python 2 下已经添加了对 CHAR/VARCHAR/CLOB 结果行的自动 Unicode 转换，以匹配 Python 3 下 cx_Oracle 的行为。为了减轻 cx_Oracle 方言在 Python 2 下先前具有的性能损失，SQLAlchemy 在 Python 2 下使用非常高效（当构建 C 扩展时）的本地 Unicode 处理程序。可以通过将`coerce_to_unicode`标志设置为 False 来禁用自动 Unicode 强制转换。此标志现在默认为 True，并适用于所有在结果集中返回的字符串数据，这些数据不明确位于`Unicode`或 Oracle 的 NVARCHAR2/NCHAR/NCLOB 数据类型下。

[#4242](https://www.sqlalchemy.org/trac/ticket/4242)  ### cx_Oracle 连接参数现代化，废弃的参数已移除

对于 cx_oracle 方言接受的参数以及 URL 字符串进行了一系列现代化处理：

+   废弃的参数`auto_setinputsizes`、`allow_twophase`、`exclude_setinputsizes`已被移除。

+   `threaded` 参数的值，对于 SQLAlchemy 方言一直默认为 True，现在不再默认生成。SQLAlchemy 的 `Connection` 对象本身不被视为线程安全，因此不需要传递此标志。

+   将 `threaded` 传递给 `create_engine()` 本身已被弃用。要将 `threaded` 的值设置为 `True`，请将其传递给 `create_engine.connect_args` 字典或使用查询字符串，例如 `oracle+cx_oracle://...?threaded=true`。

+   现在，传递到 URL 查询字符串的所有参数，如果不被特别消耗，都会传递给 cx_Oracle.connect() 函数。其中一些也会被强制转换为 cx_Oracle 常量或布尔值，包括 `mode`、`purity`、`events` 和 `threaded`。

+   与之前一样，所有 cx_Oracle `.connect()` 参数都通过 `create_engine.connect_args` 字典接受，文档在这方面是不准确的。

[#4369](https://www.sqlalchemy.org/trac/ticket/4369)  ### 国家字符数据类型被弱化以支持通用 Unicode，可通过选项重新启用。

默认情况下，`Unicode` 和 `UnicodeText` 数据类型现在对应于 Oracle 上的 `VARCHAR2` 和 `CLOB` 数据类型，而不是 `NVARCHAR2` 和 `NCLOB`（也称为“国家”字符集类型）。这将在诸如它们在 `CREATE TABLE` 语句中的呈现方式等行为中看到，以及当使用 `Unicode` 或 `UnicodeText` 绑定参数时，不会传递任何类型对象给 `setinputsizes()`；cx_Oracle 会原生处理字符串值。这一变化基于 cx_Oracle 的维护者的建议，即 Oracle 中的“国家”数据类型在很大程度上已经过时且性能不佳。它们还会在某些情况下干扰，比如应用于 `trunc()` 等函数的格式说明符时。

可能需要使用 `NVARCHAR2` 和相关类型的情况是数据库未使用符合 Unicode 标准的字符集。在这种情况下，可以通过将标志 `use_nchar_for_unicode` 传递给 `create_engine()` 来重新启用旧行为。

始终如此，在 DDL 中明确使用`NVARCHAR2`和`NCLOB`数据类型将继续使用`NVARCHAR2`和`NCLOB`，包括在处理绑定参数时使用 cx_Oracle 的`setinputsizes()`。

在读取方面，在 Python 2 下已添加了 CHAR/VARCHAR/CLOB 结果行的自动 Unicode 转换，以匹配 Python 3 下 cx_Oracle 的行为。为了减轻以前在 Python 2 下 cx_Oracle 方言在这种行为下的性能损失，SQLAlchemy 在 Python 2 下使用了非常高效（当构建了 C 扩展时）的本地 Unicode 处理程序。自动 Unicode 强制转换可以通过将`coerce_to_unicode`标志设置为 False 来禁用。该标志现在默认为 True，并适用于结果集中返回的所有不明确为`Unicode`或 Oracle 的 NVARCHAR2/NCHAR/NCLOB 数据类型的字符串数据。

[#4242](https://www.sqlalchemy.org/trac/ticket/4242)

### cx_Oracle 连接参数现代化，弃用的参数已移除

对 cx_oracle 方言接受的参数以及 URL 字符串进行了一系列现代化改进：

+   弃用的参数`auto_setinputsizes`、`allow_twophase`、`exclude_setinputsizes`已被移除。

+   `threaded`参数的值，对于 SQLAlchemy 方言始终默认为 True，现在不再默认生成。SQLAlchemy 的`Connection`对象本身不被认为是线程安全的，因此不需要传递此标志。

+   将`threaded`传递给`create_engine()`本身已被弃用。要将`threaded`的值设置为`True`，请将其传递给`create_engine.connect_args`字典或使用查询字符串，例如`oracle+cx_oracle://...?threaded=true`。

+   现在，URL 查询字符串中传递的所有参数，如果不被特殊消耗，都会传递给 cx_Oracle.connect()函数。其中一些参数也会被强制转换为 cx_Oracle 常量或布尔值，包括`mode`、`purity`、`events`和`threaded`。

+   与之前一样，所有 cx_Oracle 的`.connect()`参数都可以通过`create_engine.connect_args`字典接受，文档在这方面描述不准确。

[#4369](https://www.sqlalchemy.org/trac/ticket/4369)

## 方言改进和变化 - SQL Server

### 支持 pyodbc fast_executemany

Pyodbc 最近添加的“fast_executemany”模式，在使用 Microsoft ODBC 驱动程序时可用，现在是 pyodbc / mssql 方言的选项。通过`create_engine()`传递：

```py
engine = create_engine(
    "mssql+pyodbc://scott:tiger@mssql2017:1433/test?driver=ODBC+Driver+13+for+SQL+Server",
    fast_executemany=True,
)
```

另请参阅

快速执行多模式

[#4158](https://www.sqlalchemy.org/trac/ticket/4158)  ### 新参数影响 IDENTITY 的起始和增量，使用 Sequence 已被弃用

从 SQL Server 2012 开始，SQL Server 现在支持具有真实`CREATE SEQUENCE`语法的序列。在[#4235](https://www.sqlalchemy.org/trac/ticket/4235)中，SQLAlchemy 将添加对这些的支持，使用`Sequence`方式与任何其他方言相同。然而，当前情况是，`Sequence`已经在 SQL Server 上重新用途，以影响主键列上`IDENTITY`规范的“start”和“increment”参数。为了使过渡向正常序列也可用，使用`Sequence`将在整个 1.3 系列中发出弃用警告。为了影响“start”和“increment”，在`Column`上使用新的`mssql_identity_start`和`mssql_identity_increment`参数：

```py
test = Table(
    "test",
    metadata_obj,
    Column(
        "id",
        Integer,
        primary_key=True,
        mssql_identity_start=100,
        mssql_identity_increment=10,
    ),
    Column("name", String(20)),
)
```

为了在非主键列上发出`IDENTITY`，这是一个很少使用但有效的 SQL Server 用例，使用`Column.autoincrement`标志，将其设置为`True`在目标列上，`False`在任何整数主键列上：

```py
test = Table(
    "test",
    metadata_obj,
    Column("id", Integer, primary_key=True, autoincrement=False),
    Column("number", Integer, autoincrement=True),
)
```

另请参阅

自增行为 / IDENTITY 列

[#4362](https://www.sqlalchemy.org/trac/ticket/4362)

[#4235](https://www.sqlalchemy.org/trac/ticket/4235)  ### 支持 pyodbc fast_executemany

Pyodbc 最近添加的“fast_executemany”模式，在使用 Microsoft ODBC 驱动程序时可用，现在是 pyodbc / mssql 方言的选项。通过`create_engine()`传递：

```py
engine = create_engine(
    "mssql+pyodbc://scott:tiger@mssql2017:1433/test?driver=ODBC+Driver+13+for+SQL+Server",
    fast_executemany=True,
)
```

另请参阅

快速执行多模式

[#4158](https://www.sqlalchemy.org/trac/ticket/4158)

### 新参数影响 IDENTITY 的起始和增量，使用 Sequence 已被弃用

从 SQL Server 2012 开始，SQL Server 现在支持具有真实`CREATE SEQUENCE`语法的序列。在[#4235](https://www.sqlalchemy.org/trac/ticket/4235)中，SQLAlchemy 将使用`Sequence`来支持这些，方式与任何其他方言相同。然而，当前情况是，`Sequence`已经在 SQL Server 上重新用途，以影响主键列上`IDENTITY`规范的“start”和“increment”参数。为了使过渡向正常序列也可用，使用`Sequence`将在整个 1.3 系列中发出弃用警告。为了影响“start”和“increment”，请在`Column`上使用新的`mssql_identity_start`和`mssql_identity_increment`参数：

```py
test = Table(
    "test",
    metadata_obj,
    Column(
        "id",
        Integer,
        primary_key=True,
        mssql_identity_start=100,
        mssql_identity_increment=10,
    ),
    Column("name", String(20)),
)
```

为了在非主键列上发出`IDENTITY`，这是一个很少使用但有效的 SQL Server 用例，可以使用`Column.autoincrement`标志，在目标列上将其设置为`True`，在任何整数主键列上将其设置为`False`：

```py
test = Table(
    "test",
    metadata_obj,
    Column("id", Integer, primary_key=True, autoincrement=False),
    Column("number", Integer, autoincrement=True),
)
```

另请参阅

自动增量行为 / IDENTITY 列

[#4362](https://www.sqlalchemy.org/trac/ticket/4362)

[#4235](https://www.sqlalchemy.org/trac/ticket/4235)

## 更改了 StatementError 的格式（换行和%s）

对`StatementError`的字符串表示引入了两个更改。字符串表示的“detail”和“SQL”部分现在由换行符分隔，并保留了原始 SQL 语句中存在的换行符。目标是提高可读性，同时仍然保持原始错误消息在一行上以便于日志记录。

这意味着以前看起来像这样的错误消息：

```py
sqlalchemy.exc.StatementError: (sqlalchemy.exc.InvalidRequestError) A value is
required for bind parameter 'id' [SQL: 'select * from reviews\nwhere id = ?']
(Background on this error at: https://sqlalche.me/e/cd3x)
```

现在看起来像这样：

```py
sqlalchemy.exc.StatementError: (sqlalchemy.exc.InvalidRequestError) A value is required for bind parameter 'id'
[SQL: select * from reviews
where id = ?]
(Background on this error at: https://sqlalche.me/e/cd3x)
```

此更改的主要影响是消费者不能再假设完整的异常消息在单行上，但是从 DBAPI 驱动程序或 SQLAlchemy 内部生成的原始“error”部分仍将在第一行上。

[#4500](https://www.sqlalchemy.org/trac/ticket/4500)
