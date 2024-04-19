# 上下文/线程本地会话

> 原文：[`docs.sqlalchemy.org/en/20/orm/contextual.html`](https://docs.sqlalchemy.org/en/20/orm/contextual.html)

回顾一下何时构建会话，何时提交，何时关闭？一节中，介绍了“会话范围”的概念，强调了在 Web 应用程序中链接`Session`的范围与 Web 请求的范围之间的实践。大多数现代 Web 框架都包括集成工具，以便自动管理`Session`的范围，并且应该使用这些工具，只要它们可用。

SQLAlchemy 包括其自己的辅助对象，它有助于建立用户定义的`Session`范围。它也被第三方集成系统用于帮助构建它们的集成方案。

该对象是`scoped_session`对象，它表示一组`Session`对象的**注册表**。如果您对注册表模式不熟悉，可以在[企业架构模式](https://martinfowler.com/eaaCatalog/registry.html)中找到一个很好的介绍。

警告

`scoped_session`注册表默认使用 Python 的`threading.local()`来跟踪`Session`实例。**这不一定与所有应用服务器兼容**，特别是那些使用绿色线程或其他替代形式的并发控制的服务器，这可能导致在中高并发情况下使用时出现竞争条件（例如，随机发生的故障）。请阅读下面的线程局部范围和在 Web 应用程序中使用线程局部范围以更充分地理解使用`threading.local()`来跟踪`Session`对象的影响，并在使用不基于传统线程的应用服务器时考虑更明确的范围。

注意

`scoped_session`对象是许多 SQLAlchemy 应用程序中非常流行和有用的对象。然而，重要的是要注意，它只提供了解决`Session`管理问题的**一个方法**。如果你对 SQLAlchemy 还不熟悉，特别是如果“线程本地变量”这个术语对你来说很陌生，我们建议你如果可能的话，首先熟悉一下诸如[Flask-SQLAlchemy](https://pypi.org/project/Flask-SQLAlchemy/)或[zope.sqlalchemy](https://pypi.org/project/zope.sqlalchemy/)之类的现成集成系统。

通过调用它并传递一个可以创建新`Session`对象的**工厂**来构造`scoped_session`。工厂只是在调用时生成一个新对象的东西，在`Session`的情况下，最常见的工厂是在本节前面介绍的`sessionmaker`。下面我们举例说明这种用法：

```py
>>> from sqlalchemy.orm import scoped_session
>>> from sqlalchemy.orm import sessionmaker

>>> session_factory = sessionmaker(bind=some_engine)
>>> Session = scoped_session(session_factory)
```

我们创建的`scoped_session`对象现在将在我们“调用”注册表时调用`sessionmaker`：

```py
>>> some_session = Session()
```

在上面，`some_session`是`Session`的一个实例，我们现在可以用它来与数据库交互。这个相同的`Session`也存在于我们创建的`scoped_session`注册表中。如果我们第二次调用注册表，我们会得到**相同的**`Session`：

```py
>>> some_other_session = Session()
>>> some_session is some_other_session
True
```

这种模式允许应用程序的不同部分调用全局的`scoped_session`，这样所有这些区域就可以在不需要显式传递的情况下共享同一个会话。我们在注册表中建立的`Session`将保持不变，直到我们显式告诉注册表将其销毁，方法是调用`scoped_session.remove()`：

```py
>>> Session.remove()
```

`scoped_session.remove()` 方法首先调用当前 `Session` 上的 `Session.close()`，其效果是首先释放任何由 `Session` 拥有的连接/事务资源，然后丢弃 `Session` 本身。这里的“释放”意味着连接被返回到其连接池，并且任何事务状态都被回滚，最终使用底层 DBAPI 连接的 `rollback()` 方法。

此时，`scoped_session` 对象是“空的”，在再次调用时将创建一个**新的** `Session`。如下所示，这不是我们之前所拥有的相同 `Session`：

```py
>>> new_session = Session()
>>> new_session is some_session
False
```

上述一系列步骤简要说明了“注册表”模式的概念。有了这个基本概念，我们可以讨论这种模式如何进行的一些细节。

## 隐式方法访问

`scoped_session` 的工作很简单；为所有请求它的人保留一个 `Session`。为了更透明地访问这个 `Session`，`scoped_session` 还包括**代理行为**，这意味着注册表本身可以直接像 `Session` 一样对待；当在此对象上调用方法时，它们会**代理**到注册表维护的底层 `Session`：

```py
Session = scoped_session(some_factory)

# equivalent to:
#
# session = Session()
# print(session.scalars(select(MyClass)).all())
#
print(Session.scalars(select(MyClass)).all())
```

上述代码实现了通过调用注册表获取当前 `Session` 然后使用该 `Session` 的相同任务。

## 线程本地作用域

熟悉多线程编程的用户会注意到，将任何东西表示为全局变量通常是一个坏主意，因为这意味着全局对象将被许多线程同时访问。`Session` 对象完全设计成以**非并发**方式使用，从多线程的角度来看，这意味着“一次只能在一个线程中”。因此，我们上面对 `scoped_session` 的使用示例，其中相同的 `Session` 对象在多次调用中保持不变，暗示着需要某种处理方式，以使多个线程中的多次调用实际上不会获取到同一个会话的句柄。我们称这个概念为**线程本地存储**，意思是，使用一个特殊的对象，它将为每个应用程序线程维护一个独立的对象。Python 通过 [threading.local()](https://docs.python.org/library/threading.html#threading.local) 构造提供了这个功能。`scoped_session` 对象默认使用此对象作为存储，以便为所有调用 `scoped_session` 注册表的人维护一个单一的 `Session`，但仅在单个线程范围内。在不同线程中调用注册表的调用者会获取一个仅限于该其他线程的 `Session` 实例。

使用这种技术，`scoped_session` 提供了一种快速且相对简单（如果熟悉线程本地存储的话）的方式，在应用程序中提供一个单一的全局对象，可以安全地从多个线程调用。

`scoped_session.remove()` 方法始终会删除与该线程关联的当前 `Session`（如果有的话）。然而，`threading.local()` 对象的一个优点是，如果应用程序线程本身结束，那么该线程的“存储”也会被垃圾回收。因此，在一个产生并销毁线程的应用程序中使用线程局部范围实际上是“安全”的，而不需要调用 `scoped_session.remove()`。然而，事务本身的范围，即通过 `Session.commit()` 或 `Session.rollback()` 结束它们，通常仍然是必须在适当的时候明确安排的东西，除非应用程序实际上将线程的寿命与事务的寿命绑定在一起。## 使用线程局部范围与 Web 应用程序

如在何时构建会话，何时提交它，何时关闭它？一节中所讨论的，一个 web 应用程序是围绕着**网络请求**的概念构建的，并且将这样的应用程序与 `Session` 集成通常意味着将 `Session` 与该请求相关联。事实证明，大多数 Python web 框架，特别是异步框架 Twisted 和 Tornado 之类的著名例外，以简单的方式使用线程，使得一个特定的网络请求在一个单独的*工作线程*的范围内接收、处理和完成。当请求结束时，工作线程被释放到一个工作线程池中，在那里它可以处理另一个请求。

这种简单的网络请求和线程的对应关系意味着，将 `Session` 关联到一个线程意味着它也与在该线程内运行的网络请求关联，反之亦然，前提是 `Session` 只在网络请求开始后创建并在网络请求结束前被销毁。因此，将 `scoped_session` 用作将 `Session` 与 web 应用程序集成的快速方法是一种常见做法。下面的顺序图说明了这个流程：

```py
Web Server          Web Framework        SQLAlchemy ORM Code
--------------      --------------       ------------------------------
startup        ->   Web framework        # Session registry is established
                    initializes          Session = scoped_session(sessionmaker())

incoming
web request    ->   web request     ->   # The registry is *optionally*
                    starts               # called upon explicitly to create
                                         # a Session local to the thread and/or request
                                         Session()

                                         # the Session registry can otherwise
                                         # be used at any time, creating the
                                         # request-local Session() if not present,
                                         # or returning the existing one
                                         Session.execute(select(MyClass)) # ...

                                         Session.add(some_object) # ...

                                         # if data was modified, commit the
                                         # transaction
                                         Session.commit()

                    web request ends  -> # the registry is instructed to
                                         # remove the Session
                                         Session.remove()

                    sends output      <-
outgoing web    <-
response
```

使用上述流程，将 `Session` 与 Web 应用程序集成的过程只有两个要求：

1.  在 Web 应用程序首次启动时创建一个单一的 `scoped_session` 注册表，确保此对象可被应用程序的其余部分访问。

1.  确保在 Web 请求结束时调用 `scoped_session.remove()`，通常是通过与 Web 框架的事件系统集成来建立“请求结束时”事件。

如前所述，上述模式只是整合 `Session` 到 Web 框架的**一种潜在方式**，特别是假定**Web 框架将 Web 请求与应用线程关联**。然而，**强烈建议使用 Web 框架本身提供的集成工具，如果有的话**，而不是 `scoped_session`。

特别是，虽然使用线程本地存储很方便，但最好将 `Session` **直接与请求关联**，而不是与当前线程关联。下一节关于自定义范围详细介绍了一种更高级的配置，可以将 `scoped_session` 的使用与直接基于请求的范围，或任何类型的范围结合起来。

## 使用自定义创建的范围

`scoped_session` 对象的默认行为“线程本地”范围只是如何“范围” `Session` 的许多选项之一。可以根据任何现有的“我们正在处理的当前事物”的系统来定义自定义范围。

假设 Web 框架定义了一个库函数 `get_current_request()`。使用此框架构建的应用程序可以随时调用此函数，结果将是表示正在处理的当前请求的某种 `Request` 对象。如果 `Request` 对象是可哈希的，那么此函数可以很容易地与 `scoped_session` 集成，以将 `Session` 与请求关联起来。下面我们结合 Web 框架提供的假设事件标记器 `on_request_end`，说明了这一点，该标记器允许在请求结束时调用代码：

```py
from my_web_framework import get_current_request, on_request_end
from sqlalchemy.orm import scoped_session, sessionmaker

Session = scoped_session(sessionmaker(bind=some_engine), scopefunc=get_current_request)

@on_request_end
def remove_session(req):
    Session.remove()
```

在上述情况中，我们以通常的方式实例化`scoped_session`，唯一的区别是我们将请求返回函数作为“scopefunc”传递。这指示`scoped_session`在每次调用注册表返回当前`Session`时使用此函数生成字典键。在这种情况下，我们特别需要确保实现可靠的“删除”系统，因为否则此字典不会自行管理。

## 上下文会话 API

| 对象名称 | 描述 |
| --- | --- |
| QueryPropertyDescriptor | 描述应用于类级别`scoped_session.query_property()`属性的类型。 |
| scoped_session | 提供`Session`对象的作用域管理。 |
| ScopedRegistry | 可以根据“作用域”函数存储单个类的一个或多个实例的注册表。 |
| ThreadLocalRegistry | 使用`threading.local()`变量进行存储的`ScopedRegistry`。 |

```py
class sqlalchemy.orm.scoped_session
```

提供`Session`对象的作用域管理。

参见上下文/线程本地会话教程。

注意

在使用异步 I/O (asyncio)时，应该使用与`scoped_session`类异步兼容的`async_scoped_session`类。

**成员**

__call__(), __init__(), add(), add_all(), autoflush, begin(), begin_nested(), bind, bulk_insert_mappings(), bulk_save_objects(), bulk_update_mappings(), close(), close_all(), commit(), configure(), connection(), delete(), deleted, dirty, execute(), expire(), expire_all(), expunge(), expunge_all(), flush(), get(), get_bind(), get_one(), identity_key(), identity_map, info, is_active, is_modified(), merge(), new, no_autoflush, object_session(), query(), query_property(), refresh(), remove(), reset(), rollback(), scalar(), scalars(), session_factory

**类签名**

类`sqlalchemy.orm.scoped_session` (`typing.Generic`)

```py
method __call__(**kw: Any) → _S
```

返回当前`Session`，如果不存在，则使用`scoped_session.session_factory`创建它。

参数:

****kw** – 关键字参数将传递给`scoped_session.session_factory`可调用对象，如果不存在现有的`Session`。如果存在`Session`并且已传递关键字参数，则会引发`InvalidRequestError`。

```py
method __init__(session_factory: sessionmaker[_S], scopefunc: Callable[[], Any] | None = None)
```

构建一个新的`scoped_session`。

参数:

+   `session_factory` – 一个用于创建新的`Session`实例的工厂。通常情况下，但不一定，这是一个`sessionmaker`的实例。

+   `scopefunc` – 可选函数，定义当前范围。如果未传递，`scoped_session`对象假定“线程本地”范围，并将使用 Python 的`threading.local()`来维护当前`Session`。如果传递了函数，该函数应返回一个可散列的令牌；此令牌将用作字典中的键，以便存储和检索当前`Session`。

```py
method add(instance: object, _warn: bool = True) → None
```

将一个对象放入此`Session`。

代表`scoped_session`类的`Session`类的代理。

当通过`Session.add()`方法传递的对象处于瞬态状态时，它们将移动到挂起状态，直到下一次刷新，然后它们将移动到持久状态。

当通过`Session.add()`方法传递的对象处于分离状态时，它们将直接移动到持久状态。

如果`Session`使用的事务被回滚，则在它们被传递给`Session.add()`时处于瞬态的对象将被移回瞬态状态，并且将不再存在于此`Session`中。

另请参阅

`Session.add_all()`

添加新项目或现有项目 - 在使用会话基础知识中

```py
method add_all(instances: Iterable[object]) → None
```

将给定的实例集添加到此`Session`中。

代表`scoped_session`类的`Session`类的代理。

有关`Session.add()`的一般行为描述，请参阅文档。

另请参阅

`Session.add()`

添加新项目或现有项目 - 在使用会话基础知识中

```py
attribute autoflush
```

代表`scoped_session`类的`Session.autoflush`属性的代理。

```py
method begin(nested: bool = False) → SessionTransaction
```

如果尚未开始事务，则在此`Session`上开始事务或嵌套事务。

代表`scoped_session`类的`Session`类的代理。

`Session`对象具有**自动开始**行为，因此通常不需要显式调用`Session.begin()`方法。但是，它可以用于控制事务状态开始的范围。

当用于开始最外层事务时，如果此`Session`已在事务内部，则会引发错误。

参数：

**嵌套** - 如果为 True，则开始 SAVEPOINT 事务，并等效于调用`Session.begin_nested()`。有关 SAVEPOINT 事务的文档，请参阅使用 SAVEPOINT。

返回：

`SessionTransaction`对象。请注意，`SessionTransaction`充当 Python 上下文管理器，允许在“with”块中使用`Session.begin()`。请参阅显式开始获取示例。

另请参阅

自动开始

管理事务

`Session.begin_nested()`

```py
method begin_nested() → SessionTransaction
```

在此 Session 上开始一个“嵌套”事务，例如 SAVEPOINT。

代理 `scoped_session` 类的 `Session` 类。

目标数据库及其关联的驱动程序必须支持 SQL SAVEPOINT，该方法才能正确运行。

有关 SAVEPOINT 事务的文档，请参阅使用 SAVEPOINT。

返回：

`SessionTransaction` 对象。请注意，`SessionTransaction` 作为上下文管理器，允许在“with”块中使用 `Session.begin_nested()`。有关用法示例，请参阅使用 SAVEPOINT。

另请参阅

使用 SAVEPOINT

Serializable isolation / Savepoints / Transactional DDL - 为了使 SAVEPOINT 正确工作，SQLite 驱动程序需要特殊的解决方法。对于 asyncio 用例，请参阅 Serializable isolation / Savepoints / Transactional DDL（asyncio 版本） 部分。

```py
attribute bind
```

代理 `scoped_session` 类的 `Session.bind` 属性。

```py
method bulk_insert_mappings(mapper: Mapper[Any], mappings: Iterable[Dict[str, Any]], return_defaults: bool = False, render_nulls: bool = False) → None
```

对给定的映射字典列表执行批量插入。

代理 `scoped_session` 类的 `Session` 类。

传统功能

此方法是 SQLAlchemy 2.0 系列的传统功能。对于现代批量插入和更新，请参阅 ORM 批量 INSERT 语句 和 ORM 根据主键批量更新。2.0 API 与此方法共享实现细节，并添加了新功能。

参数：

+   `mapper` - 一个映射类，或者实际的 `Mapper` 对象，表示映射列表中表示的单个对象类型。

+   `mappings` – 一个字典序列，每个字典包含要插入的映射行的状态，以映射类上的属性名称表示。如果映射引用多个表，例如联合继承映射，每个字典必须包含要填充到所有表中的所有键。

+   `return_defaults` –

    当设置为 True 时，将更改 INSERT 过程以确保获取新生成的主键值。通常设置此参数的原因是启用联合表继承映射的批量插入。

    注意

    对于不支持 RETURNING 的后端，`Session.bulk_insert_mappings.return_defaults` 参数可以显著降低性能，因为无法批量处理 INSERT 语句。请参阅 “插入多个值”行为的 INSERT 语句 了解哪些后端会受到影响的背景信息。

+   `render_nulls` –

    当设置为 True 时，`None` 的值将导致 NULL 值包含在 INSERT 语句中，而不是将列从 INSERT 中省略。这允许要 INSERT 的所有行具有相同的列集，从而允许将所有行批量发送到 DBAPI。通常，包含与上一行不同的 NULL 值组合的每个列集必须省略 INSERT 语句中的一系列不同列，这意味着必须将其作为单独的语句发出。通过传递此标志，可以确保将所有行的完整集合批量处理到一个批次中；但是，成本是将被省略的列调用的服务器端默认值将被跳过，因此必须确保这些值不是必需的。

    警告

    当设置此标志时，**不会为那些以 NULL 插入的列调用服务器端默认 SQL 值**；NULL 值将被明确发送。必须小心确保整个操作不需要调用服务器端默认函数。

请参阅

ORM 启用的 INSERT、UPDATE 和 DELETE 语句

`Session.bulk_save_objects()` 

`Session.bulk_update_mappings()`

```py
method bulk_save_objects(objects: Iterable[object], return_defaults: bool = False, update_changed_only: bool = True, preserve_order: bool = True) → None
```

对给定对象列表执行批量保存。

代理`Session`类，代表`scoped_session`类。

旧特性

此方法作为 SQLAlchemy 2.0 系列的传统功能。有关现代批量 INSERT 和 UPDATE，请参见 ORM 批量 INSERT 语句和 ORM 批量按主键 UPDATE 部分。

对于一般的 ORM 映射对象的 INSERT 和 UPDATE，请优先使用标准的工作单元数据管理模式，在 SQLAlchemy 统一教程的 ORM 数据操作中引入。SQLAlchemy 2.0 现在使用现代方言的“Insert Many Values”行为用于 INSERT 语句，解决了以前批量 INSERT 速度慢的问题。

参数：

+   `objects` –

    一个映射对象实例的序列。映射对象按原样持久化，并且在之后**不**与`Session`相关联。

    对于每个对象，该对象是作为 INSERT 还是 UPDATE 发送取决于传统操作中`Session`使用的相同规则；如果对象具有`InstanceState.key`属性设置，则假定对象为“分离”，并将导致 UPDATE。否则，使用 INSERT。

    在 UPDATE 的情况下，语句根据已更改的属性分组，因此将成为每个 SET 子句的主题。如果`update_changed_only`为 False，则将应用每个对象中存在的所有属性到 UPDATE 语句中，这可能有助于将语句分组到更大的 executemany()中，并且还将减少检查属性历史记录的开销。

+   `return_defaults` – 当为 True 时，将缺少生成默认值的值的行插入“一次”，以便主键值可用。特别是，这将允许联合继承和其他多表映射正确插入，而无需提前提供主键值；但是，`Session.bulk_save_objects.return_defaults` `大大降低了`该方法的性能收益。强烈建议请使用标准的`Session.add_all()`方法。

+   `update_changed_only` – 当为 True 时，基于每个状态中已记录更改的属性渲染 UPDATE 语句。当为 False 时，除主键属性外，将所有存在的属性渲染到 SET 子句中。

+   `preserve_order` - 当为 True 时，插入和更新的顺序与给定对象的顺序完全匹配。当为 False 时，常见类型的对象被分组为插入和更新，以便提供更多的批处理机会。

另请参阅

ORM-Enabled INSERT、UPDATE 和 DELETE 语句

`Session.bulk_insert_mappings()`

`Session.bulk_update_mappings()`

```py
method bulk_update_mappings(mapper: Mapper[Any], mappings: Iterable[Dict[str, Any]]) → None
```

对给定的映射字典列表执行批量更新。

代理了`scoped_session`类的`Session`类。

传统功能

作为 SQLAlchemy 2.0 系列的一个传统功能。有关现代批量 INSERT 和 UPDATE，请参阅 ORM 批量 INSERT 语句和 ORM 通过主键进行批量 UPDATE 部分。2.0 API 与此方法共享实现细节，并添加了新功能。

参数：

+   `mapper` - 一个映射类，或者表示映射列表中所表示的单一对象的实际`Mapper`对象。

+   `mappings` - 一系列字典，每个字典包含要更新的映射行的状态，以映射类上的属性名称为准。如果映射涉及多个表，比如联接继承映射，则每个字典可能包含对所有表对应的键。所有这些已存在且不是主键的键都将应用于 UPDATE 语句的 SET 子句；所需的主键值将应用于 WHERE 子句。

另请参阅

ORM-Enabled INSERT、UPDATE 和 DELETE 语句

`Session.bulk_insert_mappings()`

`Session.bulk_save_objects()`

```py
method close() → None
```

关闭此`Session`所使用的事务资源和 ORM 对象。

代理了`scoped_session`类的`Session`类。

这会清除与此`Session`关联的所有 ORM 对象，结束任何正在进行的事务，并释放此`Session`自身从关联的`Engine`对象中签出的任何`Connection`对象。然后，该操作将使`Session`处于可以再次使用的状态。

提示

在默认运行模式下，`Session.close()`方法**不会阻止再次使用会话**。`Session`本身实际上没有明确的“关闭”状态；它只是表示`Session`将释放所有数据库连接和 ORM 对象。

将参数`Session.close_resets_only`设置为`False`将使`close`最终，意味着会禁止对会话的任何进一步操作。

从版本 1.4 开始更改：`Session.close()`方法不会立即创建新的`SessionTransaction`对象；只有在再次为数据库操作使用`Session`时才会创建新的`SessionTransaction`。

另请参见

关闭 - 关于`Session.close()`和`Session.reset()`语义的详细信息。

`Session.reset()` - 一个类似的方法，行为类似于`close()`，参数`Session.close_resets_only`设置为`True`。

```py
classmethod close_all() → None
```

关闭*所有*内存中的会话。

代理`scoped_session`类的`Session`类。

自 1.3 版本起已弃用：`Session.close_all()` 方法已弃用，将在将来的版本中删除。请参考`close_all_sessions()`。

```py
method commit() → None
```

刷新待处理更改并提交当前事务。

代理了`scoped_session`类的`Session`类。

当 COMMIT 操作完成时，所有对象都被完全过期，擦除其内部内容，在下次访问对象时会自动重新加载。在此期间，这些对象处于过期状态，如果从`Session`中分离，它们将无法运行。此外，在使用基于 asyncio 的 API 时不支持此重新加载操作。`Session.expire_on_commit`参数可用于禁用此行为。

当`Session`没有正在进行的事务时，表示自上次调用`Session.commit()`以来没有对此`Session`进行任何操作，则该方法将开始并提交一个仅限内部的“逻辑”事务，通常不会影响数据库，除非检测到待处理的刷新更改，但仍将调用事件处理程序和对象过期规则。

最外层数据库事务会无条件提交，自动释放任何正在进行的 SAVEPOINT。

请参阅。

提交。

管理事务。

在使用 AsyncSession 时避免隐式 IO。

```py
method configure(**kwargs: Any) → None
```

重新配置由此`scoped_session`使用的`sessionmaker`。

查看`sessionmaker.configure()`。

```py
method connection(bind_arguments: _BindArguments | None = None, execution_options: CoreExecuteOptionsParameter | None = None) → Connection
```

返回一个对应于此`Session`对象的事务状态的`Connection`对象。

代理了`scoped_session`类的`Session`类。

返回当前事务对应的`Connection`，如果没有进行中的事务，则开始一个新事务并返回`Connection`（注意，直到发出第一个 SQL 语句之前，才会与 DBAPI 建立事务状态）。

多绑定或未绑定的`Session`对象中的歧义可以通过任何可选的关键字参数来解决。最终，这使得使用`get_bind()`方法来解析。

参数：

+   `bind_arguments` – 绑定参数字典。可能包括“mapper”、“bind”、“clause”、“其他传递给`Session.get_bind()`的自定义参数。

+   `execution_options` –

    将传递给`Connection.execution_options()`的执行选项字典，**仅在首次获取连接时**。如果连接已经存在于`Session`中，则会发出警告并忽略参数。

    另请参阅

    设置事务隔离级别 / DBAPI AUTOCOMMIT

```py
method delete(instance: object) → None
```

将实例标记为已删除。

代表`scoped_session`类为`Session`类代理。

当传递的对象被假定为持久的或分离的时，调用该方法后，对象将保持在持久状态，直到下一次刷新进行。在此期间，该对象还将成为`Session.deleted`集合的成员。

下一次刷新进行时，对象将转移到删除状态，表示在当前事务中为其行发出了`DELETE`语句。当事务成功提交时，已删除的对象将转移到分离状态，并且不再存在于此`Session`中。

另请参阅

删除 - 在使用会话的基础知识

```py
attribute deleted
```

所有在此`Session`中标记为“已删除”的实例集合

代表`scoped_session`类为`Session`类进行了代理。

```py
attribute dirty
```

被视为脏的所有持久实例的集合。

代表`scoped_session`类为`Session`类进行了代理。

例如：

```py
some_mapped_object in session.dirty
```

实例在被修改但未被删除时被视为脏。

请注意，这个“脏”计算是“乐观”的；大多数属性设置或集合修改操作都会将实例标记为“脏”，并将其放入这个集合中，即使属性的值没有净变化。在刷新时，将每个属性的值与其先前保存的值进行比较，如果没有净变化，则不会发生 SQL 操作（这是一项更昂贵的操作，因此只在刷新时执行）。

要检查实例是否对其属性有可操作的净变化，请使用`Session.is_modified()`方法。

```py
method execute(statement: Executable, params: _CoreAnyExecuteParams | None = None, *, execution_options: OrmExecuteOptionsParameter = {}, bind_arguments: _BindArguments | None = None, _parent_execute_state: Any | None = None, _add_event: Any | None = None) → Result[Any]
```

执行 SQL 表达式构造。

代表`scoped_session`类为`Session`类进行了代理。

返回表示语句执行结果的`Result`对象。

例如：

```py
from sqlalchemy import select
result = session.execute(
    select(User).where(User.id == 5)
)
```

`Session.execute()`的 API 合同类似于`Connection.execute()`，2.0 风格版本的`Connection`。

从版本 1.4 开始变更：当使用 2.0 风格 ORM 使用时，`Session.execute()`方法现在是 ORM 语句执行的主要点。

参数：

+   `statement` – 可执行的语句（即`Executable`表达式，如`select()`）。

+   `params` – 可选字典或字典列表，其中包含绑定的参数值。如果是单个字典，则执行单行操作；如果是字典列表，则将调用“executemany”。每个字典中的键必须对应于语句中存在的参数名称。

+   `execution_options` –

    可选的执行选项字典，将与语句执行相关联。此字典可以提供`Connection.execution_options()`接受的选项子集，并且还可以提供只在 ORM 上下文中理解的其他选项。

    另请参阅

    ORM 执行选项 - ORM 特定的执行选项

+   `bind_arguments` – 用于确定绑定的其他参数的字典。可能包括“mapper”，“bind”或其他自定义参数。此字典的内容传递给`Session.get_bind()`方法。

返回：

一个`Result`对象。

```py
method expire(instance: object, attribute_names: Iterable[str] | None = None) → None
```

将实例的属性过期。

代表`scoped_session`类的`Session`类的代理。

将实例的属性标记为过时。下次访问过期属性时，将向`Session`对象的当前事务上下文发出查询，以加载给定实例的所有过期属性。请注意，高度隔离的事务将返回与先前在同一事务中读取的相同值，而不管该事务之外的数据库状态的变化。

要同时使`Session`中的所有对象过期，请使用`Session.expire_all()`。

`Session`对象的默认行为是在调用`Session.rollback()`或`Session.commit()`方法时将所有状态过期，以便为新的事务加载新状态。因此，仅在当前事务中发出了非 ORM SQL 语句的特定情况下调用`Session.expire()`才有意义。

参数：

+   `instance` – 要刷新的实例。

+   `attribute_names` – 可选的字符串属性名称列表，指示要过期的属性子集。

另请参阅

刷新 / 过期 - 入门材料

`Session.expire()`

`Session.refresh()`

`Query.populate_existing()`

```py
method expire_all() → None
```

使此会话中的所有持久实例过期。

代表 `scoped_session` 类，为 `Session` 类提供代理。

当下次访问持久实例的任何属性时，将使用 `Session` 对象的当前事务上下文发出查询，以加载给定实例的所有过期属性。请注意，高度隔离的事务将返回与之前在该事务中读取的相同值，而不考虑该事务之外的数据库状态的更改。

要使单个对象和这些对象上的单个属性过期，请使用 `Session.expire()`。

`Session` 对象的默认行为是在调用 `Session.rollback()` 或 `Session.commit()` 方法时使所有状态过期，以便为新事务加载新状态。因此，通常不需要调用 `Session.expire_all()`，假设事务是隔离的。

另请参阅

刷新 / 过期 - 入门材料

`Session.expire()`

`Session.refresh()`

`Query.populate_existing()`

```py
method expunge(instance: object) → None
```

从此 `Session` 中移除实例。

代表 `scoped_session` 类，为 `Session` 类提供代理。

这将释放对实例的所有内部引用。将根据 *expunge* 级联规则应用级联。

```py
method expunge_all() → None
```

从此 `Session` 中移除所有对象实例。

代表 `scoped_session` 类，为 `Session` 类提供代理。

这相当于在此 `Session` 中对所有对象调用 `expunge(obj)`。

```py
method flush(objects: Sequence[Any] | None = None) → None
```

将所有对象更改刷新到数据库。

代表`Session`类的`scoped_session`类。

将所有待处理的对象创建、删除和修改写入数据库，作为 INSERTs、DELETEs、UPDATEs 等。操作会自动按照会话的工作单元依赖解析器进行排序。

数据库操作将在当前事务上下文中发出，并且不会影响事务的状态，除非发生错误，在这种情况下，整个事务都将回滚。您可以在事务中随意刷新()以将更改从 Python 移动到数据库的事务缓冲区。

参数：

**objects** –

可选；限制刷新操作仅对给定集合中存在的元素进行操作。

此功能适用于极为狭窄的一组使用案例，其中可能需要在完全刷新()发生之前对特定对象进行操作。不适用于常规用途。

```py
method get(entity: _EntityBindKey[_O], ident: _PKIdentityArgument, *, options: Sequence[ORMOption] | None = None, populate_existing: bool = False, with_for_update: ForUpdateParameter = None, identity_token: Any | None = None, execution_options: OrmExecuteOptionsParameter = {}, bind_arguments: _BindArguments | None = None) → _O | None
```

返回基于给定主键标识符的实例，如果找不到则返回`None`。

代表`Session`类的`scoped_session`类。

例如：

```py
my_user = session.get(User, 5)

some_object = session.get(VersionedFoo, (5, 10))

some_object = session.get(
    VersionedFoo,
    {"id": 5, "version_id": 10}
)
```

从版本 1.4 开始：添加了`Session.get()`，它已从现在的遗留`Query.get()`方法中移动。

`Session.get()`是特殊的，它直接提供对`Session`的标识映射的访问。如果给定的主键标识符存在于本地标识映射中，则直接从此集合返回对象，并且不会发出 SQL，除非对象已被标记为完全过期。如果不存在，则执行 SELECT 以定位对象。

`Session.get()`还将执行检查，看对象是否存在于标识映射中并标记为过期 - 还会发出 SELECT 以刷新对象以及确保行仍然存在。如果不是，则引发`ObjectDeletedError`。

参数：

+   `entity` – 指示要加载的实体类型的映射类或`Mapper`。

+   `ident` –

    代表主键的标量、元组或字典。对于复合（例如，多列）主键，应传递元组或字典。

    对于单列主键，标量调用形式通常是最方便的。如果一行的主键是值“5”，调用看起来像：

    ```py
    my_object = session.get(SomeClass, 5)
    ```

    元组形式包含主键值，通常按照其对应于映射的`Table`对象的主键列的顺序排列，或者如果使用了`Mapper.primary_key`配置参数，则按照该参数使用的顺序排列。例如，如果一行的主键由整数数字“5, 10”表示，调用将如下所示：

    ```py
    my_object = session.get(SomeClass, (5, 10))
    ```

    字典形式应包含键，对应于每个主键元素的映射属性名称。如果映射类具有存储对象主键值的属性 `id`、`version_id`，则调用将如下所示：

    ```py
    my_object = session.get(SomeClass, {"id": 5, "version_id": 10})
    ```

+   `options` – 可选的加载器选项序列，如果发出查询，则将应用于该查询。

+   `populate_existing` – 导致该方法无条件发出 SQL 查询并使用新加载的数据刷新对象，无论对象是否已存在。

+   `with_for_update` – 可选布尔值 `True`，表示应该使用 FOR UPDATE，或者可以是一个包含标志的字典，表示用于 SELECT 的一组更具体的 FOR UPDATE 标志；标志应该与`Query.with_for_update()`方法的参数匹配。取代 `Session.refresh.lockmode` 参数。

+   `execution_options` –

    可选的执行选项字典，如果发出查询，则与查询执行相关联。此字典可以提供`Connection.execution_options()`接受的选项子集，并且还可以提供只在 ORM 上下文中理解的其他选项。

    从版本 1.4.29 开始新增。

    另请参阅

    ORM 执行选项 - ORM 特定的执行选项

+   `bind_arguments` –

    用于确定绑定的附加参数字典。可能包括“mapper”、“bind”或其他自定义参数。此字典的内容传递给 `Session.get_bind()`方法。

返回：

对象实例，或 `None`。

```py
method get_bind(mapper: _EntityBindKey[_O] | None = None, *, clause: ClauseElement | None = None, bind: _SessionBind | None = None, _sa_skip_events: bool | None = None, _sa_skip_for_implicit_returning: bool = False, **kw: Any) → Engine | Connection
```

返回此`Session`绑定的“bind”。

代理为`Session`类，代表`scoped_session`类。

“bind”通常是 `Engine` 的实例，除非 `Session` 已经被明确地直接绑定到 `Connection` 的情况除外。

对于多重绑定或未绑定的 `Session`，使用 `mapper` 或 `clause` 参数来确定要返回的适当绑定。

注意，当通过 ORM 操作调用 `Session.get_bind()`，比如 `Session.query()`，以及 `Session.flush()` 中的每个单独的 INSERT/UPDATE/DELETE 操作时，“mapper”参数通常会出现在调用中。

解析顺序为：

1.  如果给定了 mapper 并且 `Session.binds` 存在，则根据首先使用的 mapper，然后使用的 mapped class，然后使用 mapped class 的 `__mro__` 中存在的任何基类来定位绑定，从更具体的超类到更一般的超类。

1.  如果给定了 clause 并且存在 `Session.binds`，则根据 `Session.binds` 中存在的给定 clause 中的 `Table` 对象来定位绑定。

1.  如果存在 `Session.binds`，则返回该绑定。

1.  如果给定了 clause，则尝试返回与 clause 最终关联的元数据的绑定。

1.  如果给定了 mapper，则尝试返回与 mapper 映射到的 `Table` 或其他可选择的元数据最终关联的绑定。

1.  如果找不到绑定，则会引发 `UnboundExecutionError`。

注意，`Session.get_bind()` 方法可以在 `Session` 的用户定义的子类上被覆盖，以提供任何类型的绑定解析方案。请参阅自定义垂直分区中的示例。

参数：

+   `mapper` – 可选的映射类或相应的`Mapper`实例。绑定可以首先通过查看与此`Session`相关联的“绑定”映射来派生自`Mapper`，其次是通过查看与`Mapper`映射到的`Table`相关联的`MetaData`来派生绑定。

+   `clause` – 一个`ClauseElement`（即`select()`、`text()`等）。如果未提供`mapper`参数或无法生成绑定，则将搜索给定的表达式构造以获取绑定元素，通常是与绑定的`MetaData`相关联的`Table`。

另请参阅

分区策略（例如每个会话的多个数据库后端）

`Session.binds`

`Session.bind_mapper()`

`Session.bind_table()`

```py
method get_one(entity: _EntityBindKey[_O], ident: _PKIdentityArgument, *, options: Sequence[ORMOption] | None = None, populate_existing: bool = False, with_for_update: ForUpdateParameter = None, identity_token: Any | None = None, execution_options: OrmExecuteOptionsParameter = {}, bind_arguments: _BindArguments | None = None) → _O
```

基于给定的主键标识符返回一个实例，如果找不到则引发异常。

代表`scoped_session`类的`Session`类的代理。

如果查询未选择任何行，则引发`sqlalchemy.orm.exc.NoResultFound`异常。

查看有关参数的详细文档，请参阅方法`Session.get()`。

2.0.22 版中的新功能。

返回：

对象实例。

另请参阅

`Session.get()` - 相应的方法，用于

如果找不到提供的主键的行，则返回`None`。

```py
classmethod identity_key(class_: Type[Any] | None = None, ident: Any | Tuple[Any, ...] = None, *, instance: Any | None = None, row: Row[Any] | RowMapping | None = None, identity_token: Any | None = None) → _IdentityKeyType[Any]
```

返回一个标识键。

代表`scoped_session`类的`Session`类的代理。

这是`identity_key()`的别名。

```py
attribute identity_map
```

代表`scoped_session`类的`Session.identity_map`属性的代理。

```py
attribute info
```

用户可修改的字典。

代表`scoped_session`类的`Session`类的代理。

此字典的初始值可以使用`Session`构造函数或`sessionmaker`构造函数或工厂方法中的`info`参数进行填充。此处的字典始终局限于此`Session`并且可以独立于所有其他`Session`对象进行修改。

```py
attribute is_active
```

如果此`Session`不处于“部分回滚”状态，则返回 True。

代表`scoped_session`类的`Session`类的代理。

从版本 1.4 开始更改：`Session`不再立即开始新事务，因此在首次实例化`Session`时，此属性将为 False。

“部分回滚”状态通常表示`Session`的刷新过程失败，并且必须发出`Session.rollback()`方法以完全回滚事务。

如果此`Session`根本不处于事务中，则在首次使用时`Session`将自动开始，因此在这种情况下`Session.is_active`将返回 True。

否则，如果此`Session`在事务中，并且该事务尚未在内部回滚，则`Session.is_active`也将返回 True。

另请参阅

“由于刷新期间先前的异常，此会话的事务已回滚。”（或类似）

`Session.in_transaction()`

```py
method is_modified(instance: object, include_collections: bool = True) → bool
```

如果给定实例具有本地修改的属性，则返回`True`。

代理`scoped_session`类为`Session`类。

此方法检索实例上每个受检的属性的历史记录，并将当前值与其先前提交的值进行比较（如果有）。

这实际上是对在`Session.dirty`集合中检查给定实例的更昂贵且准确的版本；会执行每个属性净“脏”状态的完整测试。

例如：

```py
return session.is_modified(someobject)
```

此方法有一些注意事项适用：

+   在`Session.dirty`集合中存在的实例在使用此方法进行测试时可能报告`False`。这是因为对象可能已经通过属性变异接收到更改事件，从而将其放置在`Session.dirty`中，但最终状态与从数据库加载的状态相同，在此处没有净变化。

+   当应用新值时，如果标量属性未加载或已过期，则可能未记录先前设置的值 - 在这些情况下，即使最终对其数据库值没有净变化，也假定属性已更改。大多数情况下，SQLAlchemy 在设置事件发生时不需要“旧”值，因此如果旧值不存在，则会跳过 SQL 调用的开销，这基于以下假设：标量值通常需要更新，在那些几种情况中不需要，平均而言比发出防御性 SELECT 要便宜。

    只有在属性容器的`active_history`标志设置为`True`时，才会无条件地在设置时获取“旧”值。通常为主键属性和不是简单多对一的标量对象引用设置此标志。要为任意映射列设置此标志，请使用带有`column_property()`的`active_history`参数。

参数：

+   `instance` – 要测试的映射实例的待处理更改。

+   `include_collections` – 表示是否应该包含多值集合在操作中。将其设置为`False`是一种仅检测基于本地列的属性（即标量列或多对一外键）的方法，这些属性在刷新时会导致此实例的更新。

```py
method merge(instance: _O, *, load: bool = True, options: Sequence[ORMOption] | None = None) → _O
```

将给定实例的状态复制到此`Session`中的相应实例。

代理为`scoped_session`类代表`Session`类。

`Session.merge()` 检查源实例的主键属性，并尝试将其与会话中具有相同主键的实例进行协调。如果在本地找不到，它将尝试根据主键从数据库加载对象，如果找不到，则创建一个新实例。然后将源实例上的每个属性的状态复制到目标实例。然后方法返回生成的目标实例；如果原始源实例尚未关联，则保持不变且未关联`Session`。

此操作如果关联映射使用`cascade="merge"`，将级联到关联的实例。

有关合并的详细讨论，请参阅合并。

参数：

+   `instance` – 要合并的实例。

+   `load` –

    布尔值，当为 False 时，`merge()` 切换到“高性能”模式，导致它放弃发出历史事件以及所有数据库访问。此标志用于诸如从二级缓存传输对象图到`Session`，或将刚加载的对象传输到工作线程或进程拥有的`Session`中而无需重新查询数据库的情况。

    `load=False` 的用例添加了一个警告，即给定对象必须处于“干净”状态，即没有要刷新的挂起更改 - 即使传入对象与任何`Session`都分离。这样，当合并操作填充本地属性并级联到相关对象和集合时，值可以“盖章”到目标对象上，而不会生成任何历史或属性事件，并且不需要将传入数据与可能未加载的任何现有相关对象或集合进行协调。`load=False` 的结果对象始终以“干净”方式生成，因此只有给定对象也应该“干净”，否则这表明方法的误用。 

+   `options` –

    可选的加载器选项序列，将在合并操作从数据库加载现有对象的版本时应用于`Session.get()`方法。

    1.4.24 版本中新增。

另请参阅

`make_transient_to_detached()` - 提供了将单个对象“合并”到`Session`中的替代方法

```py
attribute new
```

在此`Session`中标记为“新”的所有实例的集合。

代表`scoped_session`类的`Session`类的代理。

```py
attribute no_autoflush
```

返回一个禁用自动刷新的上下文管理器。

代表`scoped_session`类的`Session`类的代理。

例如：

```py
with session.no_autoflush:

    some_object = SomeClass()
    session.add(some_object)
    # won't autoflush
    some_object.related_thing = session.query(SomeRelated).first()
```

在`with:`块内进行的操作不会受到查询访问时发生的刷新的影响。这在初始化涉及现有数据库查询的一系列对象时非常有用，其中未完成的对象不应立即被刷新。

```py
classmethod object_session(instance: object) → Session | None
```

返回一个对象所属的`Session`。

代表`scoped_session`类的`Session`类的代理。

这是`object_session()`的别名。

```py
method query(*entities: _ColumnsClauseArgument[Any], **kwargs: Any) → Query[Any]
```

返回与此`Session`对应的新`Query`对象。

代表`scoped_session`类的`Session`类的代理。

注意`Query`对象在 SQLAlchemy 2.0 中已被废弃；现在使用`select()`构造 ORM 查询。

另请参阅

SQLAlchemy 统一教程

ORM 查询指南

旧版查询 API - 旧版 API 文档

```py
method query_property(query_cls: Type[Query[_T]] | None = None) → QueryPropertyDescriptor
```

返回一个类属性，当调用时会针对该类和当前`Session`产生一个旧版的`Query`对象。

旧版特性

`scoped_session.query_property()` 访问器专门针对传统的 `Query` 对象，不被视为 2.0-style ORM 使用的一部分。

例如：

```py
from sqlalchemy.orm import QueryPropertyDescriptor
from sqlalchemy.orm import scoped_session
from sqlalchemy.orm import sessionmaker

Session = scoped_session(sessionmaker())

class MyClass:
    query: QueryPropertyDescriptor = Session.query_property()

# after mappers are defined
result = MyClass.query.filter(MyClass.name=='foo').all()
```

默认情况下，通过会话配置的查询类产生会话的实例。要覆盖并使用自定义实现，请提供一个 `query_cls` 可调用对象。将使用类的映射器作为位置参数和会话关键字参数调用可调用对象。

在类上放置的查询属性的数量没有限制。

```py
method refresh(instance: object, attribute_names: Iterable[str] | None = None, with_for_update: ForUpdateParameter = None) → None
```

对给定实例的过期和刷新属性。

代理为 `scoped_session` 类代表 `Session` 类。

选定的属性将首先过期，就像使用 `Session.expire()` 时一样；然后将向数据库发出 SELECT 语句，以使用当前事务中可用的当前值刷新面向列的属性。

如果对象已经急加载了，那么 `relationship()` 导向的属性也将立即加载，并使用它们最初加载的急加载策略。

新版本 1.4 中：- `Session.refresh()` 方法现在也可以刷新急加载的属性。

如果惰性加载的关系不在 `Session.refresh.attribute_names` 中命名，则它们将保持为“惰性加载”属性，并且不会隐式刷新。

2.0.4 版本中的更改：`Session.refresh()` 方法现在将刷新那些在 `Session.refresh.attribute_names` 集合中显式命名的惰性加载的 `relationship()` 导向的属性。

提示

虽然 `Session.refresh()` 方法能够刷新列和关系导向属性，但其主要焦点是在单个实例上刷新本地列导向属性。对于更开放式的“刷新”功能，包括在具有显式控制关系加载器策略的同时刷新多个对象的属性的能力，请改用 populate existing 功能。

注意，高度隔离的事务将返回在同一事务中先前读取的相同值，而不管该事务外部数据库状态的变化如何。刷新属性通常只在事务开始时有意义，此时数据库行尚未被访问。

参数：

+   `attribute_names` – 可选。一个字符串属性名称的可迭代集合，指示要刷新的属性的子集。

+   `with_for_update` – 可选布尔值 `True`，表示应该使用 FOR UPDATE，或者可以是一个包含标志的字典，指示要在 SELECT 中使用一组更具体的 FOR UPDATE 标志；标志应该与 `Query.with_for_update()` 的参数匹配。覆盖 `Session.refresh.lockmode` 参数。

另请参阅

Refreshing / Expiring - 入门材料

`Session.expire()`

`Session.expire_all()`

Populate Existing - 允许任何 ORM 查询按照正常加载的方式刷新对象。

```py
method remove() → None
```

如果存在，释放当前 `Session`。

这首先会在当前 `Session` 上调用 `Session.close()` 方法，释放仍在持有的任何现有的事务/连接资源；具体地，会回滚事务。然后丢弃 `Session`。在同一作用域内的下一次使用时，`scoped_session` 将生成一个新的 `Session` 对象。

```py
method reset() → None
```

结束此 `Session` 使用的事务资源和 ORM 对象，将会将会将会将会将会将会将会将会将会将会将会将会将会将会将会将会将会将会将会将会将会将会将会将会将会将会将会将会将会将会将会将会将会将会将会将会将会将会将会将会将会将会将会将会将会将会将会将会将会将会将会将会...

代理`Session`类，代表`scoped_session`类。

此方法提供了与`Session.close()`方法历史上提供的相同的“仅重置”行为，其中`Session`的状态被重置，就像对象是全新的，准备好再次使用一样。然后，此方法可能对将`Session.close_resets_only`设置为`False`的`Session`对象有用，以便“仅重置”行为仍然可用。

新版本 2.0.22 中的内容。

另请参阅

关闭 - 关于`Session.close()`和`Session.reset()`语义的详细信息。

`Session.close()` - 当参数`Session.close_resets_only`设置为`False`时，类似的方法还会阻止对会话的重新使用。

```py
method rollback() → None
```

回滚当前进行中的事务。

代理`Session`类，代表`scoped_session`类。

如果没有事务正在进行，则此方法将被忽略。

该方法总是回滚最顶层的数据库事务，丢弃可能正在进行的任何嵌套事务。

另请参阅

回滚

管理事务

```py
method scalar(statement: Executable, params: _CoreSingleExecuteParams | None = None, *, execution_options: OrmExecuteOptionsParameter = {}, bind_arguments: _BindArguments | None = None, **kw: Any) → Any
```

执行语句并返回标量结果。

代理`Session`类，代表`scoped_session`类。

使用方法和参数与`Session.execute()`相同；返回结果是一个标量 Python 值。

```py
method scalars(statement: Executable, params: _CoreAnyExecuteParams | None = None, *, execution_options: OrmExecuteOptionsParameter = {}, bind_arguments: _BindArguments | None = None, **kw: Any) → ScalarResult[Any]
```

执行语句并将结果作为标量返回。

代理`Session`类，代表`scoped_session`类。

使用和参数与 `Session.execute()` 相同；返回结果是一个过滤对象 `ScalarResult`，该对象将返回单个元素而不是 `Row` 对象。

返回：

一个 `ScalarResult` 对象

新特性在版本 1.4.24 中添加：增加了 `Session.scalars()`

新特性在版本 1.4.26 中添加：增加了`scoped_session.scalars()`

另见

选择 ORM 实体 - 将`Session.execute()`的行为与`Session.scalars()`进行对比

```py
attribute session_factory: sessionmaker[_S]
```

提供给 `__init__` 的 session_factory 存储在这个属性中，以后可以访问。当需要一个新的非范围 `Session` 时，这可能会很有用。

```py
class sqlalchemy.util.ScopedRegistry
```

一个可以基于“作用域”函数存储一个或多个单个类实例的注册表。

该对象实现了 `__call__` 作为“getter”，因此通过调用 `myregistry()` 返回当前范围的包含对象。

参数：

+   `createfunc` – 一个可调用的函数，返回要放置在注册表中的新对象

+   `scopefunc` – 一个可调用的函数，它将返回一个用于存储/检索对象的键。

**成员**

__init__(), clear(), has(), set()

**类签名**

类 `sqlalchemy.util.ScopedRegistry` (`typing.Generic`)

```py
method __init__(createfunc: Callable[[], _T], scopefunc: Callable[[], Any])
```

构建一个新的 `ScopedRegistry`。

参数：

+   `createfunc` – 一个创建函数，如果当前范围中不存在，则会生成一个新值。

+   `scopefunc` – 返回表示当前范围的可哈希令牌的函数（例如，当前线程标识符）。

```py
method clear() → None
```

清除当前范围，如果有的话。

```py
method has() → bool
```

如果对象存在于当前范围中，则返回 True。

```py
method set(obj: _T) → None
```

设置当前范围的值。

```py
class sqlalchemy.util.ThreadLocalRegistry
```

使用 `threading.local()` 变量进行存储的 `ScopedRegistry`。

**类签名**

类`sqlalchemy.util.ThreadLocalRegistry`（`sqlalchemy.util.ScopedRegistry`）

```py
class sqlalchemy.orm.QueryPropertyDescriptor
```

描述应用于类级别的`scoped_session.query_property()`属性的类型。

新版本 2.0.5 中新增。

**类签名**

类`sqlalchemy.orm.QueryPropertyDescriptor`（`typing_extensions.Protocol`）

## 隐式方法访问

`scoped_session`的工作很简单；为所有请求它的人保存一个`Session`。为了更透明地访问这个`Session`，`scoped_session`还包括**代理行为**，意味着可以直接将注册表本身视为`Session`；当在这个对象上调用方法时，它们会**被代理**到注册表维护的基础`Session`上：

```py
Session = scoped_session(some_factory)

# equivalent to:
#
# session = Session()
# print(session.scalars(select(MyClass)).all())
#
print(Session.scalars(select(MyClass)).all())
```

上述代码完成了与通过调用注册表获取当前`Session`相同的任务，然后使用该`Session`。

## 线程本地作用域

对于熟悉多线程编程的用户来说，将任何东西表示为全局变量通常都是一个坏主意，因为这意味着全局对象将被许多线程同时访问。`Session`对象完全设计成以**非并发**方式使用，从多线程的角度来看，这意味着“一次只能在一个线程中”。因此，我们上面的`scoped_session`使用示例，其中同一个`Session`对象在多个调用之间保持不变，表明需要有一些进程存在，以确保许多线程中的多个调用实际上不会获得相同的会话句柄。我们将此概念称为**线程本地存储**，这意味着使用一个特殊对象，该对象将维护每个应用程序线程的独立对象。Python 通过[threading.local()](https://docs.python.org/library/threading.html#threading.local)构造提供了这一功能。`scoped_session`对象默认使用此对象作为存储，以便在调用`scoped_session`注册表的所有调用者中维护单个`Session`，但仅在单个线程的范围内。在不同线程中调用注册表的调用者将获得一个针对该其他线程本地的`Session`实例。

使用这种技术，`scoped_session`提供了一种快速而相对简单（如果熟悉线程本地存储的话）的方式，在应用程序中提供一个可以安全地从多个线程调用的单一全局对象。

与往常一样，`scoped_session.remove()`方法会删除当前与线程关联的`Session`（如果有的话）。然而，`threading.local()`对象的一个优点是，如果应用程序线程本身结束，那么该线程的“存储”也会被垃圾回收。因此，在应用程序生成和销毁线程的情况下，使用线程本地作用域实际上是“安全”的，而无需调用`scoped_session.remove()`。然而，事务本身的范围，即通过`Session.commit()`或`Session.rollback()`结束它们，通常仍然是必须在适当时候显式安排的，除非应用程序实际上将线程的寿命与事务的寿命绑定在一起。

## 在 Web 应用程序中使用线程本地作用域

如在何时构建会话、何时提交以及何时关闭会话？一节中所讨论的，Web 应用程序的架构围绕着**web 请求**的概念展开，而将这样的应用程序与`Session`集成通常意味着`Session`将与该请求相关联。事实证明，大多数 Python Web 框架（Twisted 和 Tornado 等异步框架是显著的例外）都以简单的方式使用线程，这样一个特定的 web 请求就在一个*工作线程*的范围内接收、处理和完成。当请求结束时，工作线程被释放到一个工作线程池中，在那里它可以处理另一个请求。

Web 请求与线程的这种简单对应关系意味着将`Session`与线程关联也意味着它也与在该线程中运行的 web 请求相关联，反之亦然，前提是`Session`仅在 Web 请求开始后创建，并在 Web 请求结束前销毁。因此，将`scoped_session`作为将`Session`与 Web 应用程序集成的一种快速方法是一种常见做法。下面的时序图说明了这个流程：

```py
Web Server          Web Framework        SQLAlchemy ORM Code
--------------      --------------       ------------------------------
startup        ->   Web framework        # Session registry is established
                    initializes          Session = scoped_session(sessionmaker())

incoming
web request    ->   web request     ->   # The registry is *optionally*
                    starts               # called upon explicitly to create
                                         # a Session local to the thread and/or request
                                         Session()

                                         # the Session registry can otherwise
                                         # be used at any time, creating the
                                         # request-local Session() if not present,
                                         # or returning the existing one
                                         Session.execute(select(MyClass)) # ...

                                         Session.add(some_object) # ...

                                         # if data was modified, commit the
                                         # transaction
                                         Session.commit()

                    web request ends  -> # the registry is instructed to
                                         # remove the Session
                                         Session.remove()

                    sends output      <-
outgoing web    <-
response
```

使用上述流程，将 `Session` 与 Web 应用程序集成的过程具有确切的两个要求：

1.  当 Web 应用程序首次启动时创建单个 `scoped_session` 注册表，确保此对象可被应用程序的其余部分访问。

1.  确保在 Web 请求结束时调用 `scoped_session.remove()`，通常通过与 Web 框架的事件系统集成以建立“请求结束时”事件来实现。

如前所述，上述模式仅是将 `Session` 与 Web 框架集成的**一种潜在方式**，特别是假设**Web 框架将 Web 请求与应用程序线程关联**。但是，**强烈建议**如果有的话，使用 Web 框架本身提供的集成工具，而不是 `scoped_session`。

特别地，虽然使用线程本地可能很方便，但最好将 `Session` 与请求**直接关联**，而不是与当前线程关联。下一节关于自定义作用域详细介绍了一种更高级的配置，可以将 `scoped_session` 的使用与直接基于请求的作用域或任何类型的作用域相结合。

## 使用自定义创建的作用域

`scoped_session` 对象的“线程本地”作用域是“对 `Session` 进行作用域”多种选项之一。可以基于任何现有的获取“我们正在处理的当前事物”的系统定义自定义作用域。

假设一个 Web 框架定义了一个名为 `get_current_request()` 的库函数。使用该框架构建的应用程序可以随时调用此函数，其结果将是表示正在处理的当前请求的某种 `Request` 对象。如果 `Request` 对象是可散列的，那么此函数可以很容易地与 `scoped_session` 集成以将 `Session` 与请求关联起来。下面我们结合 Web 框架提供的假设事件标记 `on_request_end` 来说明这一点，该事件标记允许在请求结束时调用代码：

```py
from my_web_framework import get_current_request, on_request_end
from sqlalchemy.orm import scoped_session, sessionmaker

Session = scoped_session(sessionmaker(bind=some_engine), scopefunc=get_current_request)

@on_request_end
def remove_session(req):
    Session.remove()
```

在上面的例子中，我们以通常的方式实例化 `scoped_session`，唯一不同的是我们将我们的请求返回函数作为“scopefunc”传递。这指示 `scoped_session` 使用此函数生成字典键，每当注册表被调用以返回当前 `Session` 时。在这种情况下，确保实现可靠的“移除”系统非常重要，因为否则这个字典不会自我管理。

## 上下文会话 API

| 对象名称 | 描述 |
| --- | --- |
| QueryPropertyDescriptor | 描述应用于类级 `scoped_session.query_property()` 属性的类型。 |
| scoped_session | 提供对 `Session` 对象的作用域管理。 |
| ScopedRegistry | 可以根据“范围”函数存储单个类的一个或多个实例的注册表。 |
| ThreadLocalRegistry | 使用 `threading.local()` 变量进行存储的 `ScopedRegistry`。 |

```py
class sqlalchemy.orm.scoped_session
```

提供对 `Session` 对象的作用域管理。

请查看上下文/线程本地会话以获取教程。

注意

当使用异步 I/O (asyncio)时，应该使用与 `scoped_session` 替代的异步兼容 `async_scoped_session` 类。

**成员**

__call__(), __init__(), add(), add_all(), autoflush, begin(), begin_nested(), bind, bulk_insert_mappings(), bulk_save_objects(), bulk_update_mappings(), close(), close_all(), commit(), configure(), connection(), delete(), deleted, dirty, execute(), expire(), expire_all(), expunge(), expunge_all(), flush(), get(), get_bind(), get_one(), identity_key(), identity_map, info, is_active, is_modified(), merge(), new, no_autoflush, object_session(), query(), query_property(), refresh(), remove(), reset(), rollback(), scalar(), scalars(), session_factory

**类签名**

类`sqlalchemy.orm.scoped_session` (`typing.Generic`)

```py
method __call__(**kw: Any) → _S
```

如果不存在，使用`scoped_session.session_factory`创建当前`Session`。

参数：

****kw** – 如果不存在现有的 `Session`，则将关键字参数传递给 `scoped_session.session_factory` 可调用对象。如果存在 `Session` 并且传递了关键字参数，则会引发 `InvalidRequestError`。

```py
method __init__(session_factory: sessionmaker[_S], scopefunc: Callable[[], Any] | None = None)
```

构建一个新的 `scoped_session`。

参数：

+   `session_factory` – 用于创建新的 `Session` 实例的工厂。通常情况下，但并非一定如此，它是 `sessionmaker` 的实例。

+   `scopefunc` – 定义当前范围的可选函数。如果未传递，则 `scoped_session` 对象假定“线程本地”范围，并将使用 Python 的 `threading.local()` 来维护当前 `Session`。如果传递，则该函数应返回可哈希的标记；此标记将用作字典中的键，以便存储和检索当前 `Session`。

```py
method add(instance: object, _warn: bool = True) → None
```

将对象放入此 `Session` 中。

代表 `scoped_session` 类的 `Session` 类的代理。

当传递给 `Session.add()` 方法时处于 transient 状态的对象将移至 pending 状态，直到下一次刷新，此时它们将移至 persistent 状态。

当传递给 `Session.add()` 方法时处于 detached 状态的对象将直接移至 persistent 状态。

如果由 `Session` 使用的事务被回滚，则在传递给 `Session.add()` 时处于暂时状态的对象将被移回到 transient 状态，并且将不再存在于此 `Session` 中。

另请参见

`Session.add_all()`

添加新项目或现有项目 - 在使用会话基础知识中

```py
method add_all(instances: Iterable[object]) → None
```

将给定的实例集合添加到此`Session`。

代理`Session`类，代表`scoped_session`类。

有关一般行为描述，请参阅`Session.add()`的文档。

另请参阅

`Session.add()`

添加新项目或现有项目 - 在使用会话基础知识中

```py
attribute autoflush
```

代表`scoped_session`类的`Session.autoflush`属性的代理。

```py
method begin(nested: bool = False) → SessionTransaction
```

在此`Session`上开始事务或嵌套事务，如果尚未开始。

代理`Session`类，代表`scoped_session`类。

`Session`对象具有**自动开始**行为，因此通常不需要显式调用`Session.begin()`方法。但是，可以使用它来控制事务状态开始的范围。

用于开始最外层事务时，如果此`Session`已在事务内部，则会引发错误。

参数：

**nested** – 如果为 True，则开始一个 SAVEPOINT 事务，并等同于调用`Session.begin_nested()`。有关 SAVEPOINT 事务的文档，请参阅使用 SAVEPOINT。

返回：

`SessionTransaction`对象。请注意，`SessionTransaction`充当 Python 上下文管理器，允许在“with”块中使用`Session.begin()`。有关示例，请参阅显式开始。

另请参阅

自动开始

管理事务

`Session.begin_nested()`

```py
method begin_nested() → SessionTransaction
```

在此 Session 上开始一个“嵌套”事务，例如 SAVEPOINT。

代理`scoped_session`类的`Session`类。

目标数据库及其关联的驱动程序必须支持 SQL SAVEPOINT 才能使该方法正常运行。

有关 SAVEPOINT 事务的文档，请参阅使用 SAVEPOINT。

返回：

`SessionTransaction`对象。请注意，`SessionTransaction`充当上下文管理器，允许在“with”块中使用`Session.begin_nested()`。请参阅使用 SAVEPOINT 以获取用法示例。

另请参阅

使用 SAVEPOINT

可序列化隔离/保存点/事务 DDL - 在 SQLite 驱动程序中，需要特殊的解决方法才能使 SAVEPOINT 正常工作。对于 asyncio 用例，请参阅可序列化隔离/保存点/事务 DDL（asyncio 版本）部分。

```py
attribute bind
```

代理`scoped_session`类的`Session.bind`属性。

```py
method bulk_insert_mappings(mapper: Mapper[Any], mappings: Iterable[Dict[str, Any]], return_defaults: bool = False, render_nulls: bool = False) → None
```

对给定的映射字典列表执行批量插入。

代理`scoped_session`类的`Session`类。

旧特性

此方法是 SQLAlchemy 2.0 系列的旧特性。对于现代批量 INSERT 和 UPDATE，请参阅 ORM 批量 INSERT 语句和 ORM 按主键批量 UPDATE 部分。2.0 API 与此方法共享实现细节，并添加了新功能。

参数：

+   `mapper` – 一个映射类，或者实际的`Mapper`对象，代表映射列表中表示的对象类型。

+   `mappings` – 一系列字典，每个字典包含要插入的映射行的状态，以映射类上的属性名称表示。如果映射涉及多个表，例如联接继承映射，则每个字典必须包含要填充到所有表中的所有键。

+   `return_defaults` –

    当为 True 时，插入过程将被改变，以确保新生成的主键值将被获取。通常，此参数的理由是为了使联接表继承映射能够被批量插入。

    注意

    对于不支持 RETURNING 的后端，`Session.bulk_insert_mappings.return_defaults` 参数可能会显著降低性能，因为 INSERT 语句无法再批量处理。请参阅“插入多个值”行为的 INSERT 语句以了解受影响的后端的背景信息。

+   `render_nulls` –

    当为 True 时，`None` 的值将导致 NULL 值被包含在 INSERT 语句中，而不是从 INSERT 中省略该列。这允许所有要插入的行具有相同的列集，从而允许将完整的行集批量到 DBAPI。通常，每个包含与上一行不同组合的 NULL 值的列集必须从呈现的 INSERT 语句中省略一个不同的系列列，这意味着它必须作为一个单独的语句发出。通过传递此标志，保证了整个行集将被批处理到一个批次中；但代价是，由省略列调用的服务器端默认值将被跳过，因此必须小心确保这些值不是必需的。

    警告

    当设置此标志时，**不会调用服务器端默认的 SQL 值**用于那些以 NULL 插入的列；NULL 值将显式发送。必须小心确保整个操作不需要调用任何服务器端默认函数。

另请参阅

ORM-启用的 INSERT、UPDATE 和 DELETE 语句

`Session.bulk_save_objects()`

`Session.bulk_update_mappings()`

```py
method bulk_save_objects(objects: Iterable[object], return_defaults: bool = False, update_changed_only: bool = True, preserve_order: bool = True) → None
```

对给定对象列表执行批量保存。

代理 `Session` 类，代表 `scoped_session` 类。

遗留特性

该方法是 SQLAlchemy 2.0 系列的传统功能。对于现代批量插入和更新，请参阅 ORM 批量插入语句和 ORM 按主键批量更新部分。

对于一般的 ORM 映射对象的 INSERT 和 UPDATE，请优先使用标准的 unit of work 数据管理模式，介绍在 SQLAlchemy 统一教程的 ORM 数据操作部分。SQLAlchemy 2.0 现在使用现代方言的“插入多个值”的行为用于 INSERT 语句，解决了以前的批量插入缓慢的问题。

参数：

+   `objects` –

    一系列映射对象实例。映射对象按原样保留，并且之后**不**与`Session`关联。

    对于每个对象，对象是作为 INSERT 还是 UPDATE 发送取决于`Session`在传统操作中使用的相同规则；如果对象具有`InstanceState.key`属性设置，则假定对象是“分离的”，将导致 UPDATE。否则，将使用 INSERT。

    在 UPDATE 的情况下，语句根据已更改的属性分组，并且因此将成为每个 SET 子句的主题。如果 `update_changed_only` 设置为 False，则每个对象中存在的所有属性都将应用于 UPDATE 语句，这可能有助于将语句组合成更大的 executemany()，还将减少检查属性历史的开销。

+   `return_defaults` – 当设置为 True 时，缺少生成默认值的值的行，即整数主键默认值和序列，将`逐个插入`，以便主键值可用。特别是这将允许加入继承和其他多表映射正确插入，无需提前提供主键值；然而，`Session.bulk_save_objects.return_defaults` `极大地降低了`该方法的整体性能。强烈建议请使用标准的`Session.add_all()`方法。

+   `update_changed_only` – 当设置为 True 时，UPDATE 语句基于每个状态中已记录更改的属性。当设置为 False 时，所有存在的属性（主键属性除外）都将进入 SET 子句。

+   `preserve_order` – 当为 True 时，插入和更新的顺序与对象给出的顺序完全匹配。当为 False 时，常见类型的对象被分组为插入和更新，以便更多批处理机会。

另请参阅

启用 ORM 的 INSERT、UPDATE 和 DELETE 语句

`Session.bulk_insert_mappings()`

`Session.bulk_update_mappings()`

```py
method bulk_update_mappings(mapper: Mapper[Any], mappings: Iterable[Dict[str, Any]]) → None
```

执行给定映射字典列表的批量更新。

代表 `scoped_session` 类的 `Session` 类的代理。

遗留特性

此方法是 SQLAlchemy 2.0 系列的遗留特性。对于现代批量插入和更新，请参阅 ORM 批量插入语句 和 ORM 通过主键批量更新 部分。2.0 API 共享此方法的实现细节，并添加了新功能。

参数：

+   `mapper` – 一个映射类，或者实际的 `Mapper` 对象，代表映射列表中表示的单个对象类型。

+   `mappings` – 一个字典序列，每个字典包含要更新的映射行的状态，以映射类上的属性名称表示。如果映射涉及多个表，例如连接继承映射，则每个字典可能包含对应于所有表的键。所有那些出现且不是主键的键都应用于 UPDATE 语句的 SET 子句；必需的主键值应用于 WHERE 子句。

另请参阅

启用 ORM 的 INSERT、UPDATE 和 DELETE 语句

`Session.bulk_insert_mappings()`

`Session.bulk_save_objects()`

```py
method close() → None
```

关闭此 `Session` 使用的事务资源和 ORM 对象。

代表 `scoped_session` 类的 `Session` 类的代理。

这会清除与此`Session`关联的所有 ORM 对象，结束进行中的任何事务，并释放此`Session`本身从关联的`Engine`对象中签出的任何`Connection`对象。然后，操作会使`Session`处于可以再次使用的状态。

提示

在默认运行模式下，`Session.close()`方法**不会阻止使用该 Session**。 `Session`本身实际上并没有明确的“关闭”状态； 它只是表示`Session`将释放所有数据库连接和 ORM 对象。

将参数`Session.close_resets_only`设置为`False`将使`close`最终化，这意味着对会话的任何进一步操作都将被禁止。

从版本 1.4 开始：`Session.close()`方法不会立即创建新的`SessionTransaction`对象； 只有在再次为数据库操作使用`Session`时，才会创建新的`SessionTransaction`。

请参阅

关闭 - 关于`Session.close()`和`Session.reset()`语义的详细信息。

`Session.reset()` - 一种行为类似于带有参数`Session.close_resets_only`设置为`True`的`close()`的方法。

```py
classmethod close_all() → None
```

关闭*所有*内存中的会话。

代表`scoped_session`类为`Session`类进行了代理。

自版本 1.3 起不推荐使用：`Session.close_all()`方法已弃用，并将在将来的版本中删除。请参阅`close_all_sessions()`。

```py
method commit() → None
```

刷新待处理更改并提交当前事务。

代表`scoped_session`类的`Session`类的代理。

当 COMMIT 操作完成时，所有对象都将被完全过期，擦除其内部内容，在下次访问对象时将自动重新加载。在此期间，这些对象处于过期状态，如果它们从`Session`中分离出来，则将无法正常工作。此外，使用基于 asyncio 的 API 时不支持此重新加载操作。`Session.expire_on_commit`参数可用于禁用此行为。

当`Session`没有正在进行的事务时，表示自上次调用`Session.commit()`以来在此`Session`上没有调用操作，该方法将开始并提交一个仅内部使用的“逻辑”事务，通常不会影响数据库，除非检测到待冲洗的更改，但仍将调用事件处理程序和对象过期规则。

最外层的数据库事务无条件提交，自动释放任何正在生效的 SAVEPOINT。

另请参见

提交

事务管理

在使用 AsyncSession 时防止隐式 IO

```py
method configure(**kwargs: Any) → None
```

重新配置由此`scoped_session`使用的`sessionmaker`。

参见 `sessionmaker.configure()`。

```py
method connection(bind_arguments: _BindArguments | None = None, execution_options: CoreExecuteOptionsParameter | None = None) → Connection
```

返回与此`Session`对象的事务状态对应的`Connection`对象。

代表`scoped_session`类的`Session`类的代理。

返回当前事务对应的`Connection`，或者如果没有进行事务，则开始新事务并返回`Connection`（请注意，在发出第一条 SQL 语句之前，与 DBAPI 之间不会建立事务状态）。

多绑定或未绑定的`Session`对象中的歧义可以通过任何可选关键字参数来解决。最终，通过使用`get_bind()`方法进行解决。

参数：

+   `bind_arguments` – 绑定参数字典。可能包括“mapper”，“bind”，“clause”，其他传递给`Session.get_bind()`的自定义参数。

+   `execution_options` –

    执行选项字典，将在首次获得连接时传递给`Connection.execution_options()`，**仅在首次获得连接时**。如果连接已经存在于`Session`中，则发出警告并忽略参数。

    另请参阅

    设置事务隔离级别/DBAPI AUTOCOMMIT

```py
method delete(instance: object) → None
```

将实例标记为已删除。

代表`scoped_session`类的`Session`类的代理。

当传递时，假定对象为持久或分离状态；在调用该方法之后，对象将保持持久状态，直到下一个刷新发生。在此期间，对象还将是`Session.deleted`集合的成员。

下一次刷新发生时，对象将移动到删除状态，表示在当前事务中为其行发出了`DELETE`语句。当事务成功提交时，删除的对象将移动到分离状态，并且不再存在于此`Session`中。

另请参阅

删除 - 在使用会话的基础知识

```py
attribute deleted
```

在此`Session`中标记为“已删除”的所有实例的集合

代理了`scoped_session`类，代表`Session`类。

```py
attribute dirty
```

被视为脏的所有持久实例的集合。

代理了`scoped_session`类，代表`Session`类。

例如：

```py
some_mapped_object in session.dirty
```

当实例被修改但未被删除时被视为脏。

请注意，此“脏”计算是“乐观”的；大多数属性设置或集合修改操作都会将实例标记为“脏”，并将其放入此集合中，即使属性的值没有净变化。在刷新时，将每个属性的值与其先前保存的值进行比较，如果没有净变化，则不会发生 SQL 操作（这是一个更昂贵的操作，因此只在刷新时执行）。

要检查实例是否对其属性具有可操作的净变化，请使用`Session.is_modified()`方法。

```py
method execute(statement: Executable, params: _CoreAnyExecuteParams | None = None, *, execution_options: OrmExecuteOptionsParameter = {}, bind_arguments: _BindArguments | None = None, _parent_execute_state: Any | None = None, _add_event: Any | None = None) → Result[Any]
```

执行 SQL 表达式构造。

代理了`scoped_session`类，代表`Session`类。

返回一个表示语句执行结果的`Result`对象。

例如：

```py
from sqlalchemy import select
result = session.execute(
    select(User).where(User.id == 5)
)
```

`Session.execute()`的 API 合同与`Connection.execute()`类似，是`Connection`的 2.0 风格版本。

从版本 1.4 开始：当使用 2.0 风格 ORM 使用时，`Session.execute()`方法现在是 ORM 语句执行的主要点。

参数：

+   `statement` – 可执行的语句（即`Executable`表达式，如`select()`）。

+   `params` – 可选的字典，或包含绑定参数值的字典列表。如果是单个字典，则执行单行；如果是字典列表，则会触发“executemany”。每个字典中的键必须与语句中存在的参数名相对应。

+   `execution_options` –

    可选的执行选项字典，将与语句执行关联。该字典可以提供 `Connection.execution_options()` 接受的选项子集，并且还可以提供仅在 ORM 上下文中理解的其他选项。

    另请参阅

    ORM 执行选项 - ORM 特定的执行选项

+   `bind_arguments` – 用于确定绑定的其他参数字典。可能包括“mapper”、“bind”或其他自定义参数。此字典的内容将传递给 `Session.get_bind()` 方法。

返回：

一个 `Result` 对象。

```py
method expire(instance: object, attribute_names: Iterable[str] | None = None) → None
```

使实例上的属性过期。

代表 `scoped_session` 类，代理 `Session` 类。

将实例的属性标记为过期。当下一次访问已过期的属性时，将向 `Session` 对象的当前事务上下文发出查询，以加载给定实例的所有过期属性。请注意，高度隔离的事务将返回与该事务中先前读取的相同值，而不管该事务之外的数据库状态的变化。

同时使`Session`中的所有对象过期，可使用 `Session.expire_all()` 方法。

当调用 `Session.rollback()` 或 `Session.commit()` 方法时，`Session` 对象的默认行为是使所有状态过期，以便为新事务加载新状态。因此，仅在当前事务中发出了非 ORM SQL 语句的情况下调用 `Session.expire()` 才有意义。

参数：

+   `instance` – 要刷新的实例。

+   `attribute_names` – 可选的字符串属性名称列表，指示要过期的属性子集。

另请参阅

刷新 / 过期 - 入门材料

`Session.expire()`

`Session.refresh()`

`Query.populate_existing()`

```py
method expire_all() → None
```

使本 `Session` 中的所有持久化实例过期。

代理了 `scoped_session` 类的代表 `Session` 类。

下次访问持久化实例上的任何属性时，将使用 `Session` 对象的当前事务上下文发出查询，以加载给定实例的所有过期属性。请注意，高度隔离的事务将返回与之前在同一事务中读取的相同值，而不管该事务之外的数据库状态是否发生变化。

要使单个对象以及这些对象上的单个属性过期，请使用 `Session.expire()`。

当调用 `Session.rollback()` 或 `Session.commit()` 方法时，`Session` 对象的默认行为是使所有状态过期，以便为新事务加载新状态。因此，通常情况下不需要调用 `Session.expire_all()`，假设事务是隔离的。

另请参阅

刷新 / 过期 - 入门材料

`Session.expire()`

`Session.refresh()`

`Query.populate_existing()`

```py
method expunge(instance: object) → None
```

从此 `Session` 中删除实例。

代理了 `scoped_session` 类的代表 `Session` 类。

这将释放对实例的所有内部引用。将根据 *expunge* 级联规则应用级联。

```py
method expunge_all() → None
```

从此 `Session` 中删除所有对象实例。

代理了 `scoped_session` 类的代表 `Session` 类。

这相当于在此 `Session` 中的所有对象上调用 `expunge(obj)`。

```py
method flush(objects: Sequence[Any] | None = None) → None
```

将所有对象更改刷新到数据库。

代表 `scoped_session` 类，为 `Session` 类代理。

将所有待处理的对象创建、删除和修改写入数据库，作为 INSERT、DELETE、UPDATE 等操作。操作会自动按照 Session 的工作单元依赖解决器进行排序。

数据库操作将在当前事务上下文中发出，并且不会影响事务的状态，除非发生错误，此时整个事务将回滚。您可以在事务中随意刷新（flush()）以将更改从 Python 移动到数据库的事务缓冲区。

参数：

**对象** -

可选；将刷新操作限制为仅对给定集合中存在的元素进行操作。

此功能仅适用于极少数情况，特定对象可能需要在完全执行 flush() 之前进行操作。不适用于一般用途。

```py
method get(entity: _EntityBindKey[_O], ident: _PKIdentityArgument, *, options: Sequence[ORMOption] | None = None, populate_existing: bool = False, with_for_update: ForUpdateParameter = None, identity_token: Any | None = None, execution_options: OrmExecuteOptionsParameter = {}, bind_arguments: _BindArguments | None = None) → _O | None
```

返回基于给定主键标识符的实例，如果找不到，则返回 `None`。

代表 `scoped_session` 类，为 `Session` 类代理。

例如：

```py
my_user = session.get(User, 5)

some_object = session.get(VersionedFoo, (5, 10))

some_object = session.get(
    VersionedFoo,
    {"id": 5, "version_id": 10}
)
```

版本 1.4 中新增：新增 `Session.get()`，它从现在遗留的 `Query.get()` 方法中移动而来。

`Session.get()` 特殊之处在于它直接提供对 `Session` 的标识映射的访问。如果给定的主键标识符存在于本地标识映射中，则直接从此集合返回对象，而不发出 SQL，除非对象已标记为完全过期。如果不存在，则执行 SELECT 以定位对象。

`Session.get()` 方法也会检查对象是否存在于标识映射中并标记为过期 - 还会发出 SELECT 来刷新对象以及确保行仍然存在。如果不存在，则会引发 `ObjectDeletedError`。

参数：

+   `entity` - 表示要加载的实体类型的映射类或 `Mapper`。

+   `ident` -

    表示主键的标量、元组或字典。对于复合（例如多列）主键，应传递元组或字典。

    对于单列主键，标量调用形式通常是最方便的。如果行的主键值为“5”，则调用如下所示：

    ```py
    my_object = session.get(SomeClass, 5)
    ```

    元组形式包含主键值，通常按照它们对应于映射的`Table`对象的主键列的顺序排列，或者如果使用了`Mapper.primary_key`配置参数，则按照该参数使用的顺序排列。例如，如果行的主键由整数数字“5, 10”表示，则调用将如下所示：

    ```py
    my_object = session.get(SomeClass, (5, 10))
    ```

    字典形式应包含键，这些键对应于主键的每个元素的映射属性名称。如果映射类具有`id`、`version_id`作为存储对象主键值的属性，则调用如下所示：

    ```py
    my_object = session.get(SomeClass, {"id": 5, "version_id": 10})
    ```

+   `options` – 可选的加载器选项序列，如果发出了查询，则将应用于该查询。

+   `populate_existing` – 导致该方法无条件发出 SQL 查询，并使用新加载的数据刷新对象，而不管对象是否已存在。

+   `with_for_update` – 可选的布尔值`True`，表示应使用 FOR UPDATE，或者可以是一个包含标志的字典，用于指示用于 SELECT 的更具体的 FOR UPDATE 标志集；标志应与`Query.with_for_update()`的参数匹配。取代`Session.refresh.lockmode`参数。

+   `execution_options` –

    可选的执行选项字典，如果发出了查询，则将与查询执行关联起来。此字典可以提供`Connection.execution_options()`接受的选项子集，并且还可以提供仅在 ORM 上下文中理解的其他选项。

    1.4.29 版本中的新功能。

    另请参阅

    ORM 执行选项 - ORM 特定的执行选项

+   `bind_arguments` –

    用于确定绑定的其他参数的字典。可能包括“mapper”、“bind”或其他自定义参数。此字典的内容传递给 `Session.get_bind()` 方法。

返回：

对象实例，或`None`。

```py
method get_bind(mapper: _EntityBindKey[_O] | None = None, *, clause: ClauseElement | None = None, bind: _SessionBind | None = None, _sa_skip_events: bool | None = None, _sa_skip_for_implicit_returning: bool = False, **kw: Any) → Engine | Connection
```

返回此`Session`绑定的“bind”。

代理给`scoped_session`类的`Session`类。

“bind”通常是`Engine`的实例，除非`Session`已经明确地直接绑定到`Connection`的情况除外。

对于多重绑定或未绑定的`Session`，使用`mapper`或`clause`参数来确定要返回的适当绑定。

请注意，通常在通过 ORM 操作调用`Session.get_bind()`时，会出现“mapper”参数，例如`Session.query()`中的每个单独的 INSERT/UPDATE/DELETE 操作，`Session.flush()`调用等。

解析顺序为：

1.  如果提供了映射器并且`Session.binds`存在，则首先基于正在使用的映射器，然后基于正在使用的映射类，最后基于映射类的`__mro__`中存在的任何基类来定位绑定，从更具体的超类到更一般的超类。

1.  如果提供了 clause 并且存在`Session.binds`，则基于`Session.binds`中存在的给定 clause 中的`Table`对象来定位绑定。

1.  如果存在`Session.binds`，则返回该绑定。

1.  如果提供了 clause，则尝试返回与最终与 clause 相关联的`MetaData`相关联的绑定。

1.  如果提供了映射器，则尝试返回与映射器映射到的`Table`或其他可选择对象最终相关联的`MetaData`相关联的绑定。

1.  无法找到绑定时，引发`UnboundExecutionError`。

请注意，`Session.get_bind()`方法可以在`Session`的用户定义子类上被重写，以提供任何类型的绑定解析方案。请参阅 Custom Vertical Partitioning 中的示例。

参数：

+   `mapper` – 可选的映射类或对应的`Mapper`实例。绑定可以首先通过查看与此`Session`关联的“binds”映射，其次通过查看与此`Mapper`映射到的`Table`相关联的`MetaData`来派生绑定。

+   `clause` – 一个`ClauseElement`（即`select()`, `text()`等）。如果不存在`mapper`参数或无法生成绑定，则将搜索给定的表达式构造，通常是与绑定的`MetaData`关联的`Table`。

另请参见

分区策略（例如每个会话多个数据库后端）

`Session.binds`

`Session.bind_mapper()`

`Session.bind_table()`

```py
method get_one(entity: _EntityBindKey[_O], ident: _PKIdentityArgument, *, options: Sequence[ORMOption] | None = None, populate_existing: bool = False, with_for_update: ForUpdateParameter = None, identity_token: Any | None = None, execution_options: OrmExecuteOptionsParameter = {}, bind_arguments: _BindArguments | None = None) → _O
```

根据给定的主键标识符返回确切的一个实例，如果未找到则引发异常。

代理为`scoped_session`类代表`Session`类。

如果查询未选择任何行，则引发`sqlalchemy.orm.exc.NoResultFound`异常。

要详细了解参数的文档，请参阅方法`Session.get()`。

版本 2.0.22 中的新功能。

返回：

对象实例。

另请参见

`Session.get()` - 相同的方法，但代替

如果找不到提供的主键的行，则返回`None`。

```py
classmethod identity_key(class_: Type[Any] | None = None, ident: Any | Tuple[Any, ...] = None, *, instance: Any | None = None, row: Row[Any] | RowMapping | None = None, identity_token: Any | None = None) → _IdentityKeyType[Any]
```

返回一个身份键。

代理为`scoped_session`类代表`Session`类。

这是`identity_key()`的别名。

```py
attribute identity_map
```

代理`scoped_session`类的`Session.identity_map`属性。

```py
attribute info
```

用户可修改的字典。

代理`scoped_session`类的`Session`类。

此字典的初始值可以使用`Session`构造函数或`sessionmaker`构造函数或工厂方法的`info`参数进行填充。此处的字典始终局限于此`Session`并且可以独立于所有其他`Session`对象进行修改。

```py
attribute is_active
```

如果此`Session`不处于“部分回滚”状态，则返回 True。

代理`scoped_session`类的`Session`类。

在 1.4 版本中更改：`Session`不再立即开始新的事务，因此当首次实例化`Session`时，此属性将为 False。

“部分回滚”状态通常表示`Session`的刷新过程失败，并且必须发出`Session.rollback()`方法以完全回滚事务。

如果此`Session`根本不在事务中，则第一次使用时会自动开始，因此在这种情况下`Session.is_active`将返回 True。

否则，如果此`Session`在事务中，并且该事务尚未在内部回滚，则`Session.is_active`也将返回 True。

另请参阅

“由于在刷新期间发生的先前异常，此会话的事务已回滚。”（或类似）

`Session.in_transaction()`

```py
method is_modified(instance: object, include_collections: bool = True) → bool
```

如果给定的实例具有本地修改的属性，则返回`True`。

代表`scoped_session`类的`Session`类的代理。

此方法检索实例上每个受控属性的历史记录，并将当前值与其以前提交的值（如果有）进行比较。

实际上，这是一个更昂贵且更准确的版本，用于检查给定实例是否在`Session.dirty`集合中；对于每个属性的净“脏”状态进行了全面测试。

例如：

```py
return session.is_modified(someobject)
```

此方法有一些注意事项：

+   当使用此方法测试时，`Session.dirty`集合中存在的实例可能报告`False`。这是因为该对象可能已通过属性突变接收到更改事件，从而将其放置在`Session.dirty`中，但最终状态与从数据库加载的状态相同，在此处没有净更改。

+   当新值被应用时，标量属性可能未记录先前设置的值，如果属性在新值接收时未加载或过期，则在这些情况下，即使最终没有对其数据库值进行净更改，也假定该属性发生了更改。大多数情况下，当发生设置事件时，SQLAlchemy 不需要“旧”值，因此，如果旧值不存在，则跳过 SQL 调用的开销，基于假设更新标量值通常是必要的，而在那些很少的情况下它不是，平均而言比发出防御性 SELECT 要便宜。

    当属性容器的`active_history`标志设置为`True`时，将无条件获取“旧”值，仅在发生设置时。通常为主键属性和非简单多对一的标量对象引用设置此标志。要为任意映射列设置此标志，请使用`column_property()`中的`active_history`参数。

参数：

+   `instance` - 要测试是否存在未决更改的映射实例。

+   `include_collections` - 表示是否应在操作中包含多值集合。将其设置为`False`是一种检测仅基于本地列的属性（即标量列或多对一外键）的方法，这些属性在刷新此实例时将导致 UPDATE。

```py
method merge(instance: _O, *, load: bool = True, options: Sequence[ORMOption] | None = None) → _O
```

将给定实例的状态复制到此`Session`中的相应实例。

代理`Session`类，代表`scoped_session`类。

`Session.merge()`检查源实例的主键属性，并尝试将其与会话中具有相同主键的实例进行协调。如果在本地找不到，它会尝试根据主键从数据库加载对象，如果找不到任何对象，则创建一个新实例。然后将源实例上的每个属性的状态复制到目标实例。然后，该方法将返回结果目标实例；原始源实例保持不变，并且如果尚未与`Session`关联，则保持不相关。

如果关联映射了`cascade="merge"`，此操作会级联到关联的实例。

有关合并的详细讨论，请参阅合并。

参数：

+   `instance` – 要合并的实例。

+   `load` –

    当为 False 时，`merge()`切换到“高性能”模式，导致它放弃发出历史事件以及所有数据库访问。此标志用于将对象图转移到来自第二级缓存的`Session`中，或者将刚加载的对象转移到由工作线程或进程拥有的`Session`中，而无需重新查询数据库。

    `load=False`用例添加了这样的警告，即给定对象必须处于“干净”状态，即没有待刷新的更改 - 即使传入对象与任何`Session`都没有关联。这是为了当合并操作填充本地属性并级联到相关对象和集合时，值可以按原样“盖印”到目标对象上，而不生成任何历史或属性事件，并且无需将传入数据与可能未加载的任何现有相关对象或集合进行协调。来自`load=False`的结果对象始终作为“干净”生成，因此只有给定的对象也应该“干净”，否则这表明方法的错误使用。

+   `options` –

    在合并操作加载对象的现有版本时，将应用到 `Session.get()` 方法的可选加载器选项序列。

    新版本 1.4.24 中新增。

另请参阅

`make_transient_to_detached()` - 提供了将单个对象“合并”到 `Session` 中的替代方法

```py
attribute new
```

在此 `Session` 中标记为“新”的所有实例的集合。

代表 `scoped_session` 类而为 `Session` 类代理。

```py
attribute no_autoflush
```

返回一个上下文管理器，用于禁用自动提交。

代表 `scoped_session` 类而为 `Session` 类代理。

例如：

```py
with session.no_autoflush:

    some_object = SomeClass()
    session.add(some_object)
    # won't autoflush
    some_object.related_thing = session.query(SomeRelated).first()
```

在 `with:` 块中进行的操作将不会受到在查询访问时发生的刷新的影响。当初始化涉及现有数据库查询的一系列对象时，尚未完成的对象不应立即被刷新时，这将很有用。

```py
classmethod object_session(instance: object) → Session | None
```

返回对象所属的 `Session`。

代表 `scoped_session` 类而为 `Session` 类代理。

这是 `object_session()` 的别名。

```py
method query(*entities: _ColumnsClauseArgument[Any], **kwargs: Any) → Query[Any]
```

返回一个新的与此 `Session` 对应的 `Query` 对象。

代表 `scoped_session` 类而为 `Session` 类代理。

请注意，`Query` 对象在 SQLAlchemy 2.0 中已被标记为遗留；现在使用 `select()` 构造 ORM 查询。

另请参阅

SQLAlchemy 统一教程

ORM 查询指南

遗留查询 API - 遗留 API 文档

```py
method query_property(query_cls: Type[Query[_T]] | None = None) → QueryPropertyDescriptor
```

返回一个类属性，当调用时，该属性会针对类和当前 `Session` 生成一个遗留 `Query` 对象。

遗留特性

`scoped_session.query_property()` 访问器是特定于传统的 `Query` 对象的，不被视为 2.0 风格 ORM 使用的一部分。

例如：

```py
from sqlalchemy.orm import QueryPropertyDescriptor
from sqlalchemy.orm import scoped_session
from sqlalchemy.orm import sessionmaker

Session = scoped_session(sessionmaker())

class MyClass:
    query: QueryPropertyDescriptor = Session.query_property()

# after mappers are defined
result = MyClass.query.filter(MyClass.name=='foo').all()
```

默认情况下，生成会话配置的查询类的实例。要覆盖并使用自定义实现，请提供一个 `query_cls` 可调用对象。将以类的映射器作为位置参数和一个会话关键字参数调用该可调用对象。

类上放置的查询属性数量没有限制。

```py
method refresh(instance: object, attribute_names: Iterable[str] | None = None, with_for_update: ForUpdateParameter = None) → None
```

在给定实例上过期并刷新属性。

代理`scoped_session` 类的 `Session` 类。

选定的属性将首先过期，就像使用 `Session.expire()` 时一样；然后将向数据库发出 SELECT 语句，以使用当前事务中可用的当前值刷新列导向属性。

`relationship()` 导向属性如果已经在对象上急切加载，将立即被加载，使用与最初加载时相同的急切加载策略。

自 1.4 版开始：- `Session.refresh()` 方法也可以刷新急切加载的属性。

如果通常使用`select`（或“延迟”）加载器策略加载的 `relationship()` 导向属性也将加载，**如果它们在 attribute_names 集合中明确命名**，则使用 `immediate` 加载器策略发出 SELECT 语句加载该属性。如果惰性加载的关系未在 `Session.refresh.attribute_names` 中命名，则它们将保持为“惰性加载”属性，不会被隐式刷新。

从版本 2.0.4 开始更改：`Session.refresh()` 方法现在将刷新 `relationship()` 导向属性的惰性加载属性，对于那些在 `Session.refresh.attribute_names` 集合中明确命名的属性。 

提示

虽然 `Session.refresh()` 方法能够刷新列和关系导向属性，但其主要重点是刷新单个实例上的本地列导向属性。对于更开放式的“刷新”功能，包括能够同时刷新多个对象的属性，并对关系加载器策略有明确控制的功能，请使用填充现有对象功能。

请注意，高度隔离的事务将返回与在同一事务中先前读取的相同值，而不考虑该事务之外数据库状态的更改。通常只在事务开始时刷新属性才有意义，在那时数据库行尚未被访问。

参数：

+   `attribute_names` – 可选。一个包含字符串属性名称的可迭代集合，指示要刷新的属性子集。

+   `with_for_update` – 可选布尔值 `True` 表示应该使用 FOR UPDATE，或者可以是一个包含标志的字典，指示应该使用更具体的 FOR UPDATE 标志集合进行 SELECT；标志应该与 `Query.with_for_update()` 的参数匹配。取代 `Session.refresh.lockmode` 参数。

另请参阅

刷新 / 过期 - 入门材料

`Session.expire()`

`Session.expire_all()`

填充现有对象 - 允许任何 ORM 查询刷新对象，就像它们通常加载的方式一样。

```py
method remove() → None
```

丢弃当前`Session`（如果存在）。

这将首先在当前`Session`上调用`Session.close()`方法，释放仍在保持的任何现有事务/连接资源；具体来说，事务将被回滚。然后丢弃`Session`。在同一范围内的下一次使用时，`scoped_session`将生成一个新的`Session`对象。

```py
method reset() → None
```

关闭此`Session`使用的事务资源和 ORM 对象，将会重置会话到其初始状态。

代理`scoped_session`类代表`Session`类。

此方法提供了与`Session.close()`方法在历史上提供的相同的“仅重置”行为，其中`Session`的状态被重置，就像对象是全新的，准备好再次使用。对于将`Session.close_resets_only`设置为`False`的`Session`对象，此方法可能会很有用，以便仍然可以使用“仅重置”行为。

版本 2.0.22 中的新功能。

另请参见

关闭 - 关于`Session.close()`和`Session.reset()`语义的详细信息。

`Session.close()` - 当参数`Session.close_resets_only`设置为`False`时，类似的方法还会阻止重复使用 Session。

```py
method rollback() → None
```

回滚当前进行中的事务。

代理`scoped_session`类代表`Session`类。

如果没有进行中的事务，则此方���是一个透传。

该方法始终回滚顶层数据库事务，丢弃可能正在进行的任何嵌套事务。

另请参见

回滚

管理事务

```py
method scalar(statement: Executable, params: _CoreSingleExecuteParams | None = None, *, execution_options: OrmExecuteOptionsParameter = {}, bind_arguments: _BindArguments | None = None, **kw: Any) → Any
```

执行语句并返回标量结果。

代理`scoped_session`类代表`Session`类。

使用和参数与`Session.execute()`相同；返回结果是一个标量 Python 值。

```py
method scalars(statement: Executable, params: _CoreAnyExecuteParams | None = None, *, execution_options: OrmExecuteOptionsParameter = {}, bind_arguments: _BindArguments | None = None, **kw: Any) → ScalarResult[Any]
```

执行语句并将结果作为标量返回。

代理`scoped_session`类代表`Session`类。

使用方法和参数与`Session.execute()`相同；返回结果是一个`ScalarResult`过滤对象，它将返回单个元素而不是`Row`对象。

返回：

一个`ScalarResult`对象

新版本 1.4.24 中新增：`Session.scalars()`

新版本 1.4.26 中新增：`scoped_session.scalars()`

另请参见

选择 ORM 实体 - 将`Session.execute()`的行为与`Session.scalars()`进行对比

```py
attribute session_factory: sessionmaker[_S]
```

提供给 __init__ 的 session_factory 存储在此属性中，可以在以后访问。当需要新的非范围化的`Session`时，这可能会有用。

```py
class sqlalchemy.util.ScopedRegistry
```

一个可以根据“scope”函数存储单个类的一个或多个实例的注册表。

此对象实现了`__call__`作为“getter”，因此通过调用`myregistry()`将返回当前范围内的包含对象。

参数：

+   `createfunc` – 返回要放置在注册表中的新对象的可调用函数

+   `scopefunc` – 一个可调用函数，将返回一个键以存储/检索对象。

**成员**

__init__(), clear(), has(), set()

**类签名**

类`sqlalchemy.util.ScopedRegistry`（`typing.Generic`）

```py
method __init__(createfunc: Callable[[], _T], scopefunc: Callable[[], Any])
```

构造一个新的`ScopedRegistry`。

参数：

+   `createfunc` – 如果当前范围中不存在，则生成新值的创建函数。

+   `scopefunc` – 一个返回表示当前范围的可哈希令牌的函数（例如，当前线程标识符）。

```py
method clear() → None
```

清除当前范围（如果有）。

```py
method has() → bool
```

如果当前范围中存在对象，则返回 True。

```py
method set(obj: _T) → None
```

设置当前范围的值。

```py
class sqlalchemy.util.ThreadLocalRegistry
```

使用`threading.local()`变量进行存储的`ScopedRegistry`。

**类签名**

类`sqlalchemy.util.ThreadLocalRegistry`（`sqlalchemy.util.ScopedRegistry`）

```py
class sqlalchemy.orm.QueryPropertyDescriptor
```

描述应用于类级别`scoped_session.query_property()`属性的类型。

自 2.0.5 版本新增。

**类签名**

类`sqlalchemy.orm.QueryPropertyDescriptor`（`typing_extensions.Protocol`）
