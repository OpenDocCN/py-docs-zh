# 烘焙查询

> 原文：[`docs.sqlalchemy.org/en/20/orm/extensions/baked.html`](https://docs.sqlalchemy.org/en/20/orm/extensions/baked.html)

`baked`为`Query`对象提供了一种替代的创建模式，允许缓存对象的构建和字符串编译步骤。这意味着对于一个特定的`Query`构建场景，如果该场景被多次使用，那么从初始构建查询到生成 SQL 字符串所涉及的所有 Python 函数调用将只会发生**一次**，而不是每次构建和执行查询时都会发生。

这个系统的理念是极大地减少 Python 解释器在**发出 SQL 之前**发生的一切的开销。 “baked”系统的缓存**不会**以任何方式减少 SQL 调用或缓存来自数据库的**返回结果**。一个展示 SQL 调用和结果集本身缓存的技术在 Dogpile Caching 中可用。

从版本 1.4 开始弃用：SQLAlchemy 1.4 和 2.0 具有全新的直接查询缓存系统，不再需要`BakedQuery`系统。现在，对于所有 Core 和 ORM 查询，缓存现在是透明激活的，用户不需要采取任何操作，使用在 SQL Compilation Caching 中描述的系统。

深度炼金术

`sqlalchemy.ext.baked`扩展**不适合初学者**。正确使用它需要对 SQLAlchemy、数据库驱动程序以及后端数据库之间的交互有很好的高级理解。这个扩展提供了一种非常特定的优化，通常是不需要的。如上所述，它**不会缓存查询**，只会缓存 SQL 本身的字符串形式。

## 概要

使用 baked 系统的开始是生成所谓的“面包店”，它代表了一系列特定查询对象的存储：

```py
from sqlalchemy.ext import baked

bakery = baked.bakery()
```

上述的“面包店”将缓存数据存储在一个默认为 200 个元素的 LRU 缓存中，需要注意的是 ORM 查询通常会包含一个用于调用 ORM 查询的条目，以及每个数据库方言的 SQL 字符串的一个条目。

该面包店允许我们通过指定其构造方式为一系列 Python 可调用对象（通常为 lambda 函数）来构建一个`Query`对象。为了简洁使用，它重写了`+=`运算符，使得典型的查询构建看起来像下面这样：

```py
from sqlalchemy import bindparam

def search_for_user(session, username, email=None):
    baked_query = bakery(lambda session: session.query(User))
    baked_query += lambda q: q.filter(User.name == bindparam("username"))

    baked_query += lambda q: q.order_by(User.id)

    if email:
        baked_query += lambda q: q.filter(User.email == bindparam("email"))

    result = baked_query(session).params(username=username, email=email).all()

    return result
```

以下是关于上述代码的一些观察：

1.  `baked_query` 对象是 `BakedQuery` 的一个实例。该对象本质上是一个真正的 orm `Query` 对象的“构建者”，但它本身并不是*实际的* `Query` 对象。

1.  实际的 `Query` 对象根本没有构建，直到在函数的最后调用 `Result.all()` 时。

1.  添加到 `baked_query` 对象的步骤都表示为 Python 函数，通常是 lambda。传递给 `bakery()` 函数的第一个 lambda 接收一个 `Session` 作为其参数。其余的 lambda 每个接收一个 `Query` 作为其参数。

1.  在上述代码中，即使我们的应用程序可能多次调用 `search_for_user()`，即使在每次调用中我们都建立一个全新的 `BakedQuery` 对象，*所有的 lambda 只调用一次*。只要此查询在烘培中被缓存，每个 lambda 在此期间**都不会**被第二次调用。

1.  缓存是通过存储**lambda 对象本身**的引用来实现的，以便构建缓存键；也就是说，Python 解释器将这些函数分配为 Python 标识，这决定了如何在后续运行中识别查询。对于那些指定了 `email` 参数的 `search_for_user()` 调用，可调用的 `lambda q: q.filter(User.email == bindparam('email'))` 将成为被检索到的缓存键的一部分；当 `email` 为 `None` 时，这个可调用函数不会成为缓存键的一部分。

1.  由于 lambda 都只调用一次，因此在 lambda 内部**不得**引用可能跨调用改变的变量；相反，假设这些是要绑定到 SQL 字符串中的值，我们使用 `bindparam()` 构建命名参数，稍后使用 `Result.params()` 应用它们的实际值。

## 性能

烘焙查询可能看起来有些奇怪、有些笨拙、有些冗长。然而，对于在应用程序中多次调用的查询，Python 性能的节约非常显著。在 性能 中演示的示例套件 `short_selects` 说明了查询的比较，每个查询仅返回一行，如下所示的常规查询：

```py
session = Session(bind=engine)
for id_ in random.sample(ids, n):
    session.query(Customer).filter(Customer.id == id_).one()
```

相比于等效的“烘焙”查询：

```py
bakery = baked.bakery()
s = Session(bind=engine)
for id_ in random.sample(ids, n):
    q = bakery(lambda s: s.query(Customer))
    q += lambda q: q.filter(Customer.id == bindparam("id"))
    q(s).params(id=id_).one()
```

对于每个块的 10000 次调用的 Python 函数调用计数的差异为：

```py
test_baked_query : test a baked query of the full entity.
                   (10000 iterations); total fn calls 1951294

test_orm_query :   test a straight ORM query of the full entity.
                   (10000 iterations); total fn calls 7900535
```

以强大的笔记本电脑上的秒数来看，这是这样的：

```py
test_baked_query : test a baked query of the full entity.
                   (10000 iterations); total time 2.174126 sec

test_orm_query :   test a straight ORM query of the full entity.
                   (10000 iterations); total time 7.958516 sec
```

请注意，此测试非常有意地包含了仅返回一行的查询。对于返回许多行的查询，烘焙查询的性能优势将越来越小，与获取行所花费的时间成比例。必须牢记的是，**烘焙查询功能仅适用于构建查询本身，而不适用于获取结果**。使用烘焙特性绝不是对更快应用程序的担保；它只是一种潜在有用的功能，适用于那些已经被证明受到这种特定形式的开销影响的应用程序。

## 理由

上述“lambda”方法是更传统的“参数化”方法的一个超集。假设我们希望构建一个简单的系统，在该系统中我们仅构建一次`Query`，然后将其存储在字典中以供重复使用。现在就可以通过简单地构建查询并通过调用`my_cached_query = query.with_session(None)`来移除其`Session`来实现这一点：

```py
my_simple_cache = {}

def lookup(session, id_argument):
    if "my_key" not in my_simple_cache:
        query = session.query(Model).filter(Model.id == bindparam("id"))
        my_simple_cache["my_key"] = query.with_session(None)
    else:
        query = my_simple_cache["my_key"].with_session(session)

    return query.params(id=id_argument).all()
```

上述方法为我们带来了非常小的性能优势。通过重用`Query`，我们节省了`session.query(Model)`构造函数内部的 Python 工作以及调用`filter(Model.id == bindparam('id'))`，这将跳过为我们构建核心表达式以及将其发送到`Query.filter()`的过程。然而，该方法仍然每次调用`Query.all()`时重新生成完整的`Select`对象，并且每次都将此全新的`Select`发送到字符串编译步骤，对于像上面这样的简单情况，这可能约占开销的 70%。

为了减少额外的开销，我们需要一些更专门的逻辑，一些记忆构造选择对象和 SQL 构造的方法。在维基百科的 [BakedQuery](https://bitbucket.org/zzzeek/sqlalchemy/wiki/UsageRecipes/BakedQuery) 部分有一个示例，这是这个特性的前身，但在那个系统中，我们没有缓存查询的*构造*。为了去除所有开销，我们需要缓存查询的构造以及 SQL 编译。假设我们按照这种方式调整了配方，并制作了一个 `.bake()` 方法，用于预编译查询的 SQL，生成一个可以以最小开销调用的新对象。我们的例子变成了：

```py
my_simple_cache = {}

def lookup(session, id_argument):
    if "my_key" not in my_simple_cache:
        query = session.query(Model).filter(Model.id == bindparam("id"))
        my_simple_cache["my_key"] = query.with_session(None).bake()
    else:
        query = my_simple_cache["my_key"].with_session(session)

    return query.params(id=id_argument).all()
```

在上述例子中，我们已经解决了性能问题，但我们仍然需要处理这个字符串缓存键。

我们可以使用“面包店”方法来重新构建上面的内容，使其看起来不像“逐步建立 lambda”方法那样不寻常，而更像是对简单“重用查询”方法的简单改进：

```py
bakery = baked.bakery()

def lookup(session, id_argument):
    def create_model_query(session):
        return session.query(Model).filter(Model.id == bindparam("id"))

    parameterized_query = bakery.bake(create_model_query)
    return parameterized_query(session).params(id=id_argument).all()
```

在上述示例中，我们使用“烘焙”系统的方式与简单的“缓存查询”系统非常相似。但是，它使用了两行代码少，不需要制造一个“my_key”的缓存键，还包括与我们自定义的“烘焙”函数相同的功能，该函数从查询的构造函数到过滤器调用再到`Select`对象的生成，再到字符串编译步骤，都缓存了 100% 的 Python 调用工作。

从上面的内容，如果我们问自己，“如果查找需要根据查询结构做条件决策怎么办？”，这就是为什么“烘焙”是这样的方式的地方。我们可以从*任意数量*的函数构建参数化查询，而不是从一个函数（这是我们最初认为烘焙可能的工作方式）开始。考虑我们的简单例子，如果我们需要在条件基础上在查询中添加一个附加子句：

```py
my_simple_cache = {}

def lookup(session, id_argument, include_frobnizzle=False):
    if include_frobnizzle:
        cache_key = "my_key_with_frobnizzle"
    else:
        cache_key = "my_key_without_frobnizzle"

    if cache_key not in my_simple_cache:
        query = session.query(Model).filter(Model.id == bindparam("id"))
        if include_frobnizzle:
            query = query.filter(Model.frobnizzle == True)

        my_simple_cache[cache_key] = query.with_session(None).bake()
    else:
        query = my_simple_cache[cache_key].with_session(session)

    return query.params(id=id_argument).all()
```

我们的“简单”参数化系统现在必须负责生成考虑到“include_frobnizzle”标志是否已传递的缓存键，因为此标志的存在意味着生成的 SQL 将完全不同。很明显，随着查询构建复杂性的提高，缓存这些查询的任务会非常快地变得繁重。我们可以将上述示例转换为以下对“面包店”直接使用：

```py
bakery = baked.bakery()

def lookup(session, id_argument, include_frobnizzle=False):
    def create_model_query(session):
        return session.query(Model).filter(Model.id == bindparam("id"))

    parameterized_query = bakery.bake(create_model_query)

    if include_frobnizzle:

        def include_frobnizzle_in_query(query):
            return query.filter(Model.frobnizzle == True)

        parameterized_query = parameterized_query.with_criteria(
            include_frobnizzle_in_query
        )

    return parameterized_query(session).params(id=id_argument).all()
```

在上述示例中，我们再次缓存的不仅是查询对象，还有它需要执行的所有工作以生成 SQL。我们也不再需要处理确保生成准确考虑到我们所做的所有结构修改的缓存键；这现在是自动处理的，而且没有错误的机会。

此代码示例比朴素示例少了几行代码，消除了处理缓存键的需求，并且具有完整的所谓“已烘焙”功能的巨大性能优势。但仍然有点啰嗦！因此，我们将像`BakedQuery.add_criteria()`和`BakedQuery.with_criteria()`这样的方法简化为操作符，并鼓励（尽管绝对不是必需的！）使用简单的 lambda 函数，仅作为减少冗长性的手段：

```py
bakery = baked.bakery()

def lookup(session, id_argument, include_frobnizzle=False):
    parameterized_query = bakery.bake(
        lambda s: s.query(Model).filter(Model.id == bindparam("id"))
    )

    if include_frobnizzle:
        parameterized_query += lambda q: q.filter(Model.frobnizzle == True)

    return parameterized_query(session).params(id=id_argument).all()
```

在上述情况中，该方法更容易实现，并且在代码流程上更类似于非缓存查询函数的情况，因此使代码更易于移植。

上述描述本质上是对到达当前“已烘焙”方法的设计过程的总结。从“正常”方法开始，还需要解决缓存键的构建和管理、移除所有冗余的 Python 执行以及需要使用条件构建的查询等附加问题，从而导致了最终的方法。

## 特殊查询技术

本节将描述一些特定查询情况下的技术。

### 使用 IN 表达式

SQLAlchemy 中的`ColumnOperators.in_()`方法在历史上基于传递给方法的项目列表呈现一个可变的绑定参数集。对于已烘焙的查询，这不起作用，因为该列表的长度可能在不同的调用中发生变化。为了解决这个问题，`bindparam.expanding`参数支持一个延迟呈现的 IN 表达式，在烘焙查询内安全地进行缓存。实际元素列表在语句执行时呈现，而不是在语句编译时：

```py
bakery = baked.bakery()

baked_query = bakery(lambda session: session.query(User))
baked_query += lambda q: q.filter(User.name.in_(bindparam("username", expanding=True)))

result = baked_query.with_session(session).params(username=["ed", "fred"]).all()
```

请参阅下文

`bindparam.expanding`

`ColumnOperators.in_()`

### 使用子查询

当使用`Query`对象时，通常需要一个`Query`对象用于在另一个查询中生成子查询。在`Query`目前处于烘焙形式的情况下，可以使用一个临时方法来检索`Query`对象，使用`BakedQuery.to_query()`方法。此方法传递给生成烘焙查询特定步骤的 lambda 可调用参数的`Session`或`Query`：

```py
bakery = baked.bakery()

# a baked query that will end up being used as a subquery
my_subq = bakery(lambda s: s.query(User.id))
my_subq += lambda q: q.filter(User.id == Address.user_id)

# select a correlated subquery in the top columns list,
# we have the "session" argument, pass that
my_q = bakery(lambda s: s.query(Address.id, my_subq.to_query(s).as_scalar()))

# use a correlated subquery in some of the criteria, we have
# the "query" argument, pass that.
my_q += lambda q: q.filter(my_subq.to_query(q).exists())
```

版本 1.3 中的新功能。

### 使用 before_compile 事件

从 SQLAlchemy 1.3.11 开始，针对特定`Query`使用`QueryEvents.before_compile()`事件将禁止烘焙查询系统缓存查询，如果事件挂钩返回一个与传入的不同的新`Query`对象。这样，`QueryEvents.before_compile()`挂钩可以在每次使用特定`Query`时被调用，以适应每次以不同方式修改查询的挂钩。要允许`QueryEvents.before_compile()`修改`sqlalchemy.orm.Query()`对象，但仍然允许结果被缓存，可以注册传递`bake_ok=True`标志的事件：

```py
@event.listens_for(Query, "before_compile", retval=True, bake_ok=True)
def my_event(query):
    for desc in query.column_descriptions:
        if desc["type"] is User:
            entity = desc["entity"]
            query = query.filter(entity.deleted == False)
    return query
```

上述策略适用于每次都以完全相同方式修改给定`Query`的事件，不依赖于特定参数或外部状态的更改。

版本 1.3.11 中的新功能：- 在`QueryEvents.before_compile()`事件中添加了“bake_ok”标志，并且如果此标志未设置，则不允许通过“baked”扩展进行缓存以发生对返回新`Query`对象的事件处理程序。

## 禁用全局会话烘焙查询

标志`Session.enable_baked_queries`可以设置为 False，导致所有烘焙查询在针对该`Session`使用时不使用缓存：

```py
session = Session(engine, enable_baked_queries=False)
```

像所有会话标志一样，它也被工厂对象（如`sessionmaker`）和方法（如`sessionmaker.configure()`）接受。

此标志的直接理由是，一个应用程序如果出现问题，可能是由于用户定义的烘焙查询或其他烘焙查询问题导致的缓存键冲突，可以关闭该行为，以确定或排除烘焙查询作为问题原因。

版本 1.2 中的新功能。

## 惰性加载集成

从版本 1.4 开始更改：从 SQLAlchemy 1.4 开始，“烘焙查询”系统不再是关系加载系统的一部分。而是改用本地缓存系统。

## API 文档

| 对象名称 | 描述 |
| --- | --- |
| BakedQuery | 用于构建`Query`对象的构建器对象。 |
| bakery | 构建一个新的烘焙坊。 |
| Bakery | 返回一个`BakedQuery`的可调用对象。 |

```py
function sqlalchemy.ext.baked.bakery(size=200, _size_alert=None)
```

构建一个新的烘焙坊。

返回：

一个`Bakery`的实例。

```py
class sqlalchemy.ext.baked.BakedQuery
```

**成员**

add_criteria(), bakery(), for_session(), spoil(), to_query(), with_criteria()

用于构建`Query`对象的构建器对象。

```py
method add_criteria(fn, *args)
```

向这个`BakedQuery`添加一个条件函数。

这相当于使用`+=`运算符就地修改`BakedQuery`。

```py
classmethod bakery(size=200, _size_alert=None)
```

构建一个新的烘焙坊。

返回：

一个`Bakery`的实例。

```py
method for_session(session)
```

为这个`BakedQuery`返回一个`Result`对象。

这相当于将`BakedQuery`作��Python 可调用对象调用，例如`result = my_baked_query(session)`。

```py
method spoil(full=False)
```

取消在此`BakedQuery`对象上发生的任何查询缓存。

`BakedQuery`可以继续正常使用，但是附加的创建函数不会被缓存；它们将在每次调用时被调用。

这是为了支持在构建烘焙查询的特定步骤中，某些使查询无法缓存的情况，例如依赖于某些不可缓存值的变体。

参数：

**full** – 如果为 False，则仅在破坏步骤之后添加到此`BakedQuery`对象的函数将不被缓存；直到此时为止的`BakedQuery`的状态将从缓存中拉取。如果为 True，则每次完全从头构建整个`Query`对象，每次调用都会调用所有创建函数。

```py
method to_query(query_or_session)
```

返回作为子查询使用的`Query`对象。

此方法应在用于生成封闭的`BakedQuery`步骤的 lambda 可调用对象内部使用。参数通常应该是传递给 lambda 的`Query`对象：

```py
sub_bq = self.bakery(lambda s: s.query(User.name))
sub_bq += lambda q: q.filter(
    User.id == Address.user_id).correlate(Address)

main_bq = self.bakery(lambda s: s.query(Address))
main_bq += lambda q: q.filter(
    sub_bq.to_query(q).exists())
```

在子查询用于第一个针对`Session`的可调用对象时，也接受`Session`：

```py
sub_bq = self.bakery(lambda s: s.query(User.name))
sub_bq += lambda q: q.filter(
    User.id == Address.user_id).correlate(Address)

main_bq = self.bakery(
    lambda s: s.query(
    Address.id, sub_bq.to_query(q).scalar_subquery())
)
```

参数：

**query_or_session** –

一个`Query`对象或一个类`Session`对象，假设它在封闭的`BakedQuery`可调用对象的上下文中。

版本 1.3 中的新功能。

```py
method with_criteria(fn, *args)
```

向从此克隆的`BakedQuery`添加一个条件函数。

这相当于使用`+`运算符产生一个具有修改的新`BakedQuery`。

```py
class sqlalchemy.ext.baked.Bakery
```

返回一个返回`BakedQuery`的可调用对象。

此对象由类方法`BakedQuery.bakery()`返回。它存在为了便于检查“缓存”。

版本 1.2 中的新功能。

```py
class sqlalchemy.ext.baked.Result
```

对一个`Session`发起一个`BakedQuery`的调用。

`Result`对象是实际创建或从缓存中检索的针对目标`Session`的`Query`对象，并且然后为结果调用。

```py
method all()
```

返回所有行。

等效于`Query.all()`。

```py
method count()
```

返回‘count’。

等效于`Query.count()`。

请注意，这使用子查询确保准确计算，而不管原始语句的结构如何。

```py
method first()
```

返回第一行。

等效于`Query.first()`。

```py
method get(ident)
```

根据标识检索对象。

等效于`Query.get()`。

```py
method one()
```

返回确切的一个结果或引发异常。

等效于`Query.one()`。

```py
method one_or_none()
```

返回一个或零个结果，或者对于多行引发异常。

等效于`Query.one_or_none()`。

```py
method params(*args, **kw)
```

指定要替换为字符串 SQL 语句的参数。

```py
method scalar()
```

返回第一个结果的第一个元素，如果没有行，则返回 None。如果返回多行，则引发 MultipleResultsFound 异常。

等效于`Query.scalar()`。

```py
method with_post_criteria(fn)
```

添加一个将在缓存后应用的条件函数。

这将添加一个函数，该函数将针对从缓存中检索的`Query`对象运行。目前仅包括`Query.params()`和`Query.execution_options()`方法。

警告

`Result.with_post_criteria()`函数应用于`Query`对象**之后**查询的 SQL 语句对象已从缓存中检索。只应使用`Query.params()`和`Query.execution_options()`方法。

在版本 1.2 中新增。

## 概要

使用烘焙系统的方法是首先生成所谓的“烘焙坊”，该坊代表一系列特定的查询对象的存储：

```py
from sqlalchemy.ext import baked

bakery = baked.bakery()
```

上述“面包店”将在默认为 200 个元素的 LRU 缓存中存储缓存数据，需要注意的是，ORM 查询通常会包含一个为调用的 ORM 查询条目，以及每个数据库方言的 SQL 字符串的条目。

面包店允许我们通过指定其构造方式为一系列 Python 可调用对象来构建一个`Query`对象，这些对象通常是 lambda 表达式。为了简洁使用，它重写了`+=`运算符，使得典型的查询构建看起来像下面这样：

```py
from sqlalchemy import bindparam

def search_for_user(session, username, email=None):
    baked_query = bakery(lambda session: session.query(User))
    baked_query += lambda q: q.filter(User.name == bindparam("username"))

    baked_query += lambda q: q.order_by(User.id)

    if email:
        baked_query += lambda q: q.filter(User.email == bindparam("email"))

    result = baked_query(session).params(username=username, email=email).all()

    return result
```

以下是关于上述代码的一些观察：

1.  `baked_query`对象是`BakedQuery`的一个实例。这个对象本质上是一个真正的 orm `Query`对象的“构造器”，但它本身并不是*真正的* `Query`对象。

1.  实际的`Query`对象根本没有构建，直到函数的最后一刻调用`Result.all()`时。

1.  添加到`baked_query`对象的步骤都表示为 Python 函数，通常是 lambda 函数。给`bakery()`函数的第一个 lambda 函数以`Session`作为其参数。其余的 lambda 函数每个都以`Query`作为其参数。

1.  在上述代码中，即使我们的应用程序可能多次调用`search_for_user()`，即使在每次调用中我们都会构建一个全新的`BakedQuery`对象，*所有的 lambda 函数只会被调用一次*。只要此查询被缓存在面包店中，每个 lambda 函数**永远**不会再次被调用。

1.  缓存是通过存储**lambda 对象本身**的引用来实现的，以形成一个缓存键；也就是说，Python 解释器将这些函数分配给 Python 标识符，这决定了如何在后续运行中识别查询。对于那些指定了`email`参数的`search_for_user()`调用，可调用对象`lambda q: q.filter(User.email == bindparam('email'))`将成为检索到的缓存键的一部分；当`email`为`None`时，此可调用对象不会成为缓存键的一部分。

1.  因为 lambda 函数只被调用一次，所以至关重要的是在 lambda 函数**内部**不引用可能在调用之间更改的变量；相反，假设这些是要绑定到 SQL 字符串中的值，我们使用 `bindparam()` 来构造命名参数，在稍后使用 `Result.params()` 应用其实际值。

## 性能

烘焙查询可能看起来有点奇怪，有点笨拙，有点啰嗦。然而，在应用程序中调用多次的查询中，Python 性能的节约非常显著。在 Performance 中演示的示例套件 `short_selects` 对比了每个仅返回一行的查询，例如以下常规查询：

```py
session = Session(bind=engine)
for id_ in random.sample(ids, n):
    session.query(Customer).filter(Customer.id == id_).one()
```

与等效的“烘焙”查询相比：

```py
bakery = baked.bakery()
s = Session(bind=engine)
for id_ in random.sample(ids, n):
    q = bakery(lambda s: s.query(Customer))
    q += lambda q: q.filter(Customer.id == bindparam("id"))
    q(s).params(id=id_).one()
```

对于对每个块进行 10000 次调用的 Python 函数调用次数的差异为：

```py
test_baked_query : test a baked query of the full entity.
                   (10000 iterations); total fn calls 1951294

test_orm_query :   test a straight ORM query of the full entity.
                   (10000 iterations); total fn calls 7900535
```

在一台性能强大的笔记本电脑上，这在秒数上表现如下：

```py
test_baked_query : test a baked query of the full entity.
                   (10000 iterations); total time 2.174126 sec

test_orm_query :   test a straight ORM query of the full entity.
                   (10000 iterations); total time 7.958516 sec
```

请注意，这个测试非常有意地包含了只返回一行的查询。对于返回许多行的查询，烘焙查询的性能优势将逐渐减少，与获取行所花费的时间成比例。必须牢记的是，**烘焙查询功能仅适用于构建查询本身，而不适用于获取结果**。使用烘焙功能绝不是使应用程序更快的保证；它只是一个可能有用的功能，适用于那些已经被测量为受到这种特定形式的开销影响的应用程序。

## 理念

上面的“lambda”方法是更传统的“参数化”方法的超集。假设我们希望构建一个简单的系统，在这个系统中我们只需构建一个`Query`，然后将其存储在字典中以便重复使用。现在，我们可以通过构建查询，然后通过调用 `my_cached_query = query.with_session(None)` 来移除其`Session`来实现这一点：

```py
my_simple_cache = {}

def lookup(session, id_argument):
    if "my_key" not in my_simple_cache:
        query = session.query(Model).filter(Model.id == bindparam("id"))
        my_simple_cache["my_key"] = query.with_session(None)
    else:
        query = my_simple_cache["my_key"].with_session(session)

    return query.params(id=id_argument).all()
```

上述方法只能带来非常微小的性能提升。通过重用`Query`，我们可以节省在`session.query(Model)`构造函数内部的 Python 工作以及调用`filter(Model.id == bindparam('id'))`时所需的工作，这将为我们跳过 Core 表达式的构建以及将其发送到`Query.filter()`。然而，该方法仍然在每次调用`Query.all()`时重新生成完整的`Select`对象，并且每次还会将这个全新的`Select`对象发送到字符串编译步骤中，对于像上面这样的简单情况，这可能占据了大约 70% 的开销。

为了减少额外的开销，我们需要一些更专门的逻辑，一种记忆构建选择对象和构建 SQL 的方法。在维基中的[BakedQuery](https://bitbucket.org/zzzeek/sqlalchemy/wiki/UsageRecipes/BakedQuery)部分有一个例子，这是该功能的前身，但在那个系统中，我们没有缓存查询的*构建*。为了消除所有开销，我们需要缓存查询的构建以及 SQL 编译。假设我们按照这种方式调整了配方，并制作了一个`.bake()`方法，用于预先编译查询的 SQL，生成一个可以以最小开销调用的新对象。我们的例子变成了：

```py
my_simple_cache = {}

def lookup(session, id_argument):
    if "my_key" not in my_simple_cache:
        query = session.query(Model).filter(Model.id == bindparam("id"))
        my_simple_cache["my_key"] = query.with_session(None).bake()
    else:
        query = my_simple_cache["my_key"].with_session(session)

    return query.params(id=id_argument).all()
```

上面，我们已经解决了性能问题，但我们仍然需要处理这个字符串缓存键。

我们可以使用“面包房”方法来重新构建上述方法，使其看起来不像“构建 lambda”方法那样不寻常，而更像是对简单“重用查询”的简单改进：

```py
bakery = baked.bakery()

def lookup(session, id_argument):
    def create_model_query(session):
        return session.query(Model).filter(Model.id == bindparam("id"))

    parameterized_query = bakery.bake(create_model_query)
    return parameterized_query(session).params(id=id_argument).all()
```

上面，我们使用“baked”系统的方式与简单的“缓存查询”系统非常相似。但是，它使用了两行更少的代码，不需要制造一个“my_key”的缓存键，而且还包含了与我们自定义的“bake”函数相同的功能，该函数缓存了查询构造函数，筛选调用，生成`Select`对象以及字符串编译步骤的 100% Python 调用工作。

从上面的内容中，如果我们问自己，“如果查找需要根据查询结构做条件决策，会怎样？”，这就是为什么“烘焙”是这样的方式的原因。我们可以从*任意数量*的函数构建参数化查询，而不是从一个函数构建（这是我们最初认为烘焙可能起作用的方式）。考虑我们的简单示例，如果我们需要在查询中有一个额外的条件子句：

```py
my_simple_cache = {}

def lookup(session, id_argument, include_frobnizzle=False):
    if include_frobnizzle:
        cache_key = "my_key_with_frobnizzle"
    else:
        cache_key = "my_key_without_frobnizzle"

    if cache_key not in my_simple_cache:
        query = session.query(Model).filter(Model.id == bindparam("id"))
        if include_frobnizzle:
            query = query.filter(Model.frobnizzle == True)

        my_simple_cache[cache_key] = query.with_session(None).bake()
    else:
        query = my_simple_cache[cache_key].with_session(session)

    return query.params(id=id_argument).all()
```

我们的“简单”参数化系统现在必须负责生成缓存键，考虑到是否传递了“include_frobnizzle”标志，因为该标志的存在意味着生成的 SQL 将完全不同。很明显，随着查询构建的复杂性增加，缓存这些查询的任务会很快变得繁重。我们可以将上面的示例转换为直接使用“bakery”如下：

```py
bakery = baked.bakery()

def lookup(session, id_argument, include_frobnizzle=False):
    def create_model_query(session):
        return session.query(Model).filter(Model.id == bindparam("id"))

    parameterized_query = bakery.bake(create_model_query)

    if include_frobnizzle:

        def include_frobnizzle_in_query(query):
            return query.filter(Model.frobnizzle == True)

        parameterized_query = parameterized_query.with_criteria(
            include_frobnizzle_in_query
        )

    return parameterized_query(session).params(id=id_argument).all()
```

在上面的情况下，我们不仅缓存查询对象，还缓存生成 SQL 所需的所有工作。我们也不再需要处理确保生成准确考虑到我们所做的所有结构修改的缓存键；这现在是自动处理的，而且没有错误的机会。

这段代码示例比简单示例少了几行，消除了处理缓存键的需要，并具有完整的所谓“烘焙”功能的巨大性能优势。但仍然有点冗长！因此，我们将像`BakedQuery.add_criteria()`和`BakedQuery.with_criteria()`这样的方法缩短为运算符，并鼓励（尽管当然不是必须！）使用简单的 lambda 表达式，只是为了减少冗长：

```py
bakery = baked.bakery()

def lookup(session, id_argument, include_frobnizzle=False):
    parameterized_query = bakery.bake(
        lambda s: s.query(Model).filter(Model.id == bindparam("id"))
    )

    if include_frobnizzle:
        parameterized_query += lambda q: q.filter(Model.frobnizzle == True)

    return parameterized_query(session).params(id=id_argument).all()
```

在上面的情况下，这种方法更容易实现，并且在代码流程上更类似于非缓存查询函数的代码，因此使得代码更容易移植。

上述描述基本上是到达当前“烘焙”方法的设计过程的总结。从“正常”方法开始，缓存键构建和管理的额外问题，消除所有多余的 Python 执行以及需要处理条件构建的查询都需要解决，最终导致了最终方法。

## 特殊查询技术

这一部分将描述一些特定查询情况下的技术。

### 使用 IN 表达式

在 SQLAlchemy 中，`ColumnOperators.in_()` 方法在历史上基于传递给方法的项目列表渲染一组变量绑定参数。这对于烘焙查询不起作用，因为该列表的长度可能在不同调用时发生变化。为了解决这个问题，`bindparam.expanding` 参数支持一个延迟渲染的 IN 表达式，可以安全地缓存在烘焙查询内部。实际元素列表在语句执行时渲染，而不是在语句编译时：

```py
bakery = baked.bakery()

baked_query = bakery(lambda session: session.query(User))
baked_query += lambda q: q.filter(User.name.in_(bindparam("username", expanding=True)))

result = baked_query.with_session(session).params(username=["ed", "fred"]).all()
```

另请参阅

`bindparam.expanding`

`ColumnOperators.in_()`

### 使用子查询

在使用`Query`对象时，通常需要一个`Query`对象用于在另一个内部生成子查询。在`Query`当前处于烘焙形式的情况下，可以使用一个中间方法来检索`Query`对象，使用`BakedQuery.to_query()`方法。该方法传递给生成烘焙查询特定步骤的 lambda 可调用参数的`Session`或`Query`：

```py
bakery = baked.bakery()

# a baked query that will end up being used as a subquery
my_subq = bakery(lambda s: s.query(User.id))
my_subq += lambda q: q.filter(User.id == Address.user_id)

# select a correlated subquery in the top columns list,
# we have the "session" argument, pass that
my_q = bakery(lambda s: s.query(Address.id, my_subq.to_query(s).as_scalar()))

# use a correlated subquery in some of the criteria, we have
# the "query" argument, pass that.
my_q += lambda q: q.filter(my_subq.to_query(q).exists())
```

版本 1.3 中新增。

### 使用 before_compile 事件

自 SQLAlchemy 1.3.11 起，针对特定的 `Query` 使用 `QueryEvents.before_compile()` 事件将禁止烘焙查询系统缓存查询，如果事件挂钩返回一个与传入的不同的新 `Query` 对象。这样，每次使用特定的 `Query` 都可以调用 `QueryEvents.before_compile()` 钩子，以适应每次更改查询的钩子。要允许 `QueryEvents.before_compile()` 修改 `sqlalchemy.orm.Query()` 对象，但仍然允许结果被缓存，可以注册事件并传递 `bake_ok=True` 标志：

```py
@event.listens_for(Query, "before_compile", retval=True, bake_ok=True)
def my_event(query):
    for desc in query.column_descriptions:
        if desc["type"] is User:
            entity = desc["entity"]
            query = query.filter(entity.deleted == False)
    return query
```

上述策略适用于每次以完全相同方式修改给定的 `Query` 的事件，不依赖于特定参数或更改的外部状态。

新版本 1.3.11 中添加了 `bake_ok` 标志到 `QueryEvents.before_compile()` 事件，并且如果此标志未设置，则不允许为返回新 `Query` 对象的事件处理程序通过“烘焙”扩展进行缓存。  ### 使用 IN 表达式

SQLAlchemy 中的 `ColumnOperators.in_()` 方法在历史上根据传递给方法的项目列表呈现一组变量绑定参数。对于烘焙查询，这不起作用，因为该列表的长度可以在不同的调用中改变。为解决此问题，`bindparam.expanding` 参数支持在烘焙查询中安全缓存的延迟呈现 IN 表达式。实际元素列表在语句执行时呈现，而不是在语句编译时：

```py
bakery = baked.bakery()

baked_query = bakery(lambda session: session.query(User))
baked_query += lambda q: q.filter(User.name.in_(bindparam("username", expanding=True)))

result = baked_query.with_session(session).params(username=["ed", "fred"]).all()
```

另请参阅

`bindparam.expanding`

`ColumnOperators.in_()`

### 使用子查询

当使用`Query`对象时，通常需要一个`Query`对象用于在另一个查询中生成子查询。在当前`Query`处于烘焙形式时，可能需要使用一个临时方法来检索`Query`对象，该方法使用`BakedQuery.to_query()`方法。此方法传递给用于生成烘焙查询特定步骤的 lambda 可调用的`Session`或`Query`参数：

```py
bakery = baked.bakery()

# a baked query that will end up being used as a subquery
my_subq = bakery(lambda s: s.query(User.id))
my_subq += lambda q: q.filter(User.id == Address.user_id)

# select a correlated subquery in the top columns list,
# we have the "session" argument, pass that
my_q = bakery(lambda s: s.query(Address.id, my_subq.to_query(s).as_scalar()))

# use a correlated subquery in some of the criteria, we have
# the "query" argument, pass that.
my_q += lambda q: q.filter(my_subq.to_query(q).exists())
```

新版本 1.3。

### 使用 before_compile 事件

从 SQLAlchemy 1.3.11 开始，针对特定`Query`使用`QueryEvents.before_compile()`事件将阻止烘焙查询系统缓存查询，如果事件钩子返回一个与传入的不同的新`Query`对象。这是为了每次使用时都可以调用特定`QueryEvents.before_compile()`钩子，以适应每次都以不同方式修改查询的钩子。要允许`QueryEvents.before_compile()`修改`sqlalchemy.orm.Query()`对象，但仍然允许结果被缓存，可以注册事件并传递`bake_ok=True`标志：

```py
@event.listens_for(Query, "before_compile", retval=True, bake_ok=True)
def my_event(query):
    for desc in query.column_descriptions:
        if desc["type"] is User:
            entity = desc["entity"]
            query = query.filter(entity.deleted == False)
    return query
```

以上策略适用于每次都会以完全相同方式修改给定的`Query`的事件，不依赖于特定参数或会发生变化的外部状态。

新版本 1.3.11 中新增：- 在`QueryEvents.before_compile()`事件中添加了“bake_ok”标志，并且如果未设置此标志，则禁止通过“baked”扩展对返回新的`Query`对象的事件处理程序进行缓存。

## 禁用全局 Baked 查询

标志`Session.enable_baked_queries`可以设置为 False，导致所有烘焙查询在用于该`Session`时不使用缓存：

```py
session = Session(engine, enable_baked_queries=False)
```

与所有会话标志一样，它也被工厂对象如`sessionmaker`和方法如`sessionmaker.configure()`所接受。

此标志的直接理由是，应用程序可能由于用户定义的烘焙查询或其他烘焙查询问题而看到问题，可以将行为关闭，以识别或排除烘焙查询作为问题的原因。

版本 1.2 中的新功能。

## 惰性加载集成

从版本 1.4 起更改：自 SQLAlchemy 1.4 起，“烘焙查询”系统不再是关系加载系统的一部分。取而代之的是使用本地缓存系统。

## API 文档

| 对象名称 | 描述 |
| --- | --- |
| BakedQuery | 用于`Query`对象的构建器对象。 |
| bakery | 构建一个新的面包店。 |
| Bakery | 返回一个`BakedQuery`的可调用对象。 |

```py
function sqlalchemy.ext.baked.bakery(size=200, _size_alert=None)
```

构建一个新的面包店。

返回：

一个`Bakery`的实例

```py
class sqlalchemy.ext.baked.BakedQuery
```

**成员**

add_criteria(), bakery(), for_session(), spoil(), to_query(), with_criteria()

用于`Query`对象的构建器对象。

```py
method add_criteria(fn, *args)
```

为此`BakedQuery`添加一个条件函数。

这相当于使用 `+=` 运算符就地修改`BakedQuery`。

```py
classmethod bakery(size=200, _size_alert=None)
```

构建一个新的面包店。

返回：

一个`Bakery`的实例

```py
method for_session(session)
```

为此`BakedQuery`返回一个`Result`对象。

这相当于将`BakedQuery`作为 Python 可调用对象调用，例如 `result = my_baked_query(session)`。

```py
method spoil(full=False)
```

取消此`BakedQuery`对象上将发生的任何查询缓存。

`BakedQuery`仍然可以正常使用，但是额外的创建函数不会被缓存；它们将在每次调用时被调用。

这是为了支持构建烘焙查询的特定步骤使查询无法缓存的情况，例如依赖于某些不可缓存值的变体。

参数：

**full** – 如果为 False，则仅在 spoil 步骤之后添加到此`BakedQuery`对象的函数将不被缓存；直到此点为止的`BakedQuery`状态将从缓存中提取。如果为 True，则每次都会从头开始构建整个`Query`对象，每次调用都会调用所有创建函数。

```py
method to_query(query_or_session)
```

返回用作子查询的`Query`对象。

此方法应在用于生成封闭`BakedQuery`步骤的 lambda 可调用内使用。参数通常应为传递给 lambda 的`Query`对象：

```py
sub_bq = self.bakery(lambda s: s.query(User.name))
sub_bq += lambda q: q.filter(
    User.id == Address.user_id).correlate(Address)

main_bq = self.bakery(lambda s: s.query(Address))
main_bq += lambda q: q.filter(
    sub_bq.to_query(q).exists())
```

在第一个可调用中使用子查询针对`Session`的情况下，也接受`Session`：

```py
sub_bq = self.bakery(lambda s: s.query(User.name))
sub_bq += lambda q: q.filter(
    User.id == Address.user_id).correlate(Address)

main_bq = self.bakery(
    lambda s: s.query(
    Address.id, sub_bq.to_query(q).scalar_subquery())
)
```

参数：

**query_or_session** –

一个`Query`对象或一个类`Session`对象，假定在封闭`BakedQuery`可调用的上下文中。

版本 1.3 中新增。

```py
method with_criteria(fn, *args)
```

向从此克隆的`BakedQuery`添加一个条件函数。

这相当于使用`+`运算符生成具有修改的新`BakedQuery`。

```py
class sqlalchemy.ext.baked.Bakery
```

返回一个返回`BakedQuery`的可调用对象。

此对象由类方法`BakedQuery.bakery()`返回。它作为一个对象存在，以便可以轻松检查“缓存”。

版本 1.2 中新增。

```py
class sqlalchemy.ext.baked.Result
```

针对`Session`调用`BakedQuery`。

`Result` 对象是实际创建或从缓存中检索到的`Query`对象，针对目标`Session`进行调用以获取结果。

```py
method all()
```

返回所有行。

等同于`Query.all()`。

```py
method count()
```

返回‘count’。

等同于`Query.count()`。

请注意，这使用子查询来确保准确计数，而不考虑原始语句的结构。

```py
method first()
```

返回第一行。

等同于`Query.first()`。

```py
method get(ident)
```

根据标识检索对象。

等同于`Query.get()`。

```py
method one()
```

返回确切的一个结果或引发异常。

等同于`Query.one()`。

```py
method one_or_none()
```

返回一个或零个结果，或者对于多行会引发异常。

等同于`Query.one_or_none()`。

```py
method params(*args, **kw)
```

指定要替换到字符串 SQL 语句中的参数。

```py
method scalar()
```

返回第一个结果的第一个元素，如果没有行则返回 None。如果返回多行，则引发 MultipleResultsFound 异常。

等同于`Query.scalar()`。

```py
method with_post_criteria(fn)
```

添加一个将在缓存后应用的条件函数。

这添加了一个将在从缓存中检索到的`Query`对象上运行的函数。目前仅包括`Query.params()`和`Query.execution_options()`方法。

警告

`Result.with_post_criteria()` 函数应用于查询的`Query`对象**之后**查询的 SQL 语句对象已从缓存中检索。只应使用`Query.params()`和`Query.execution_options()`方法。

新版本 1.2 中新增。
