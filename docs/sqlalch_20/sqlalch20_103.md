# 事件

> 原文：[`docs.sqlalchemy.org/en/20/core/event.html`](https://docs.sqlalchemy.org/en/20/core/event.html)

SQLAlchemy 包括一个事件 API，它发布了一系列钩子，可以进入 SQLAlchemy 核心和 ORM 的内部。

## 事件注册

订阅事件通过单个 API 点完成，即`listen()` 函数，或者可以使用`listens_for()` 装饰器。这些函数接受一个目标，一个字符串标识符，用于标识要拦截的事件，以及一个用户定义的监听函数。这两个函数的额外位置参数和关键字参数可能会被特定类型的事件支持，这些事件可能会指定给定事件函数的备用接口，或者根据给定目标提供有关次要事件目标的指令。

事件的名称和相应监听函数的参数签名是从绑定到文档中描述的标记类的绑定规范方法派生的。例如，对于`PoolEvents.connect()` 的文档表明，事件名称为`"connect"`，并且用户定义的监听函数应接收两个位置参数：

```py
from sqlalchemy.event import listen
from sqlalchemy.pool import Pool

def my_on_connect(dbapi_con, connection_record):
    print("New DBAPI connection:", dbapi_con)

listen(Pool, "connect", my_on_connect)
```

使用 `listens_for()` 装饰器进行监听如下所示：

```py
from sqlalchemy.event import listens_for
from sqlalchemy.pool import Pool

@listens_for(Pool, "connect")
def my_on_connect(dbapi_con, connection_record):
    print("New DBAPI connection:", dbapi_con)
```

## 命名参数样式

监听函数可以接受一些参数样式的变化。以`PoolEvents.connect()` 为例，此函数被记录为接收`dbapi_connection` 和 `connection_record` 参数。我们可以选择通过名称接收这些参数，通过建立一个接受 `**keyword` 参数的监听函数，或通过向 `listen()` 或 `listens_for()` 传递 `named=True` 来实现：

```py
from sqlalchemy.event import listens_for
from sqlalchemy.pool import Pool

@listens_for(Pool, "connect", named=True)
def my_on_connect(**kw):
    print("New DBAPI connection:", kw["dbapi_connection"])
```

使用命名参数传递时，函数参数规范中列出的名称将用作字典中的键。

命名样式通过名称传递所有参数，而不考虑函数签名，因此特定参数也可以以任何顺序列出，只要名称匹配即可：

```py
from sqlalchemy.event import listens_for
from sqlalchemy.pool import Pool

@listens_for(Pool, "connect", named=True)
def my_on_connect(dbapi_connection, **kw):
    print("New DBAPI connection:", dbapi_connection)
    print("Connection record:", kw["connection_record"])
```

上面，存在 `**kw` 告诉 `listens_for()` 参数应该通过名称而不是位置传递给函数。

## 目标

`listen()` 函数在目标方面非常灵活。它通常接受类、这些类的实例以及相关类或对象，可以从中派生适当的目标。例如，上述提到的 `"connect"` 事件接受 `Engine` 类和对象，以及 `Pool` 类和对象：

```py
from sqlalchemy.event import listen
from sqlalchemy.pool import Pool, QueuePool
from sqlalchemy import create_engine
from sqlalchemy.engine import Engine
import psycopg2

def connect():
    return psycopg2.connect(user="ed", host="127.0.0.1", dbname="test")

my_pool = QueuePool(connect)
my_engine = create_engine("postgresql+psycopg2://ed@localhost/test")

# associate listener with all instances of Pool
listen(Pool, "connect", my_on_connect)

# associate listener with all instances of Pool
# via the Engine class
listen(Engine, "connect", my_on_connect)

# associate listener with my_pool
listen(my_pool, "connect", my_on_connect)

# associate listener with my_engine.pool
listen(my_engine, "connect", my_on_connect)
```

## 修饰符

一些监听器允许传递修饰符给 `listen()`。这些修饰符有时会为监听器提供替代的调用签名。例如，在 ORM 事件中，一些事件监听器可以具有修改后续处理的返回值。默认情况下，没有监听器需要返回值，但通过传递 `retval=True` 可以支持此值：

```py
def validate_phone(target, value, oldvalue, initiator):
  """Strip non-numeric characters from a phone number"""

    return re.sub(r"\D", "", value)

# setup listener on UserContact.phone attribute, instructing
# it to use the return value
listen(UserContact.phone, "set", validate_phone, retval=True)
```

## 事件和多进程

SQLAlchemy 的事件钩子是用 Python 函数和对象实现的，因此事件通过 Python 函数调用进行传播。Python 的多进程遵循我们对操作系统多进程的思考方式，例如父进程派生子进程，因此我们可以使用相同的模型描述 SQLAlchemy 事件系统的行为。

在父进程中注册的事件钩子将存在于从注册这些钩子后派生的新子进程中，因为在生成子进程时，子进程会以父进程的所有现有 Python 结构的副本开始。在注册钩子之前已存在的子进程将不会接收到这些新的事件钩子，因为父进程中对 Python 结构的更改不会传播到子进程。

对于事件本身，这些都是 Python 函数调用，没有任何能力在进程之间传播。SQLAlchemy 的事件系统不实现任何进程间通信。可以实现使用 Python 进程间消息传递的事件钩子，但这需要用户自己实现。

## 事件参考

SQLAlchemy Core 和 SQLAlchemy ORM 都提供了各种各样的事件钩子：

+   **核心事件** - 这些事件在 核心事件 中描述，包括特定于连接池生命周期、SQL 语句执行、事务生命周期和模式创建和拆除的事件钩子。

+   **ORM 事件** - 这些事件在 ORM 事件 中描述，包括特定于类和属性检测、对象初始化钩子、属性更改钩子、会话状态、刷新和提交钩子、映射器初始化、对象/结果填充和每个实例持久性钩子的事件钩子。

## API 参考

| 对象名称 | 描述 |
| --- | --- |
| contains(target, identifier, fn) | 如果给定的目标/标识符/fn 设置为监听，则返回 True。 |
| listen(target, identifier, fn, *args, **kw) | 为给定目标注册监听器函数。 |
| listens_for(target, identifier, *args, **kw) | 将函数装饰为给定目标 + 标识符的监听器。 |
| remove(target, identifier, fn) | 移除事件监听器。 |

```py
function sqlalchemy.event.listen(target: Any, identifier: str, fn: Callable[[...], Any], *args: Any, **kw: Any) → None
```

为给定目标注册监听器函数。

`listen()` 函数是 SQLAlchemy 事件系统的主要接口之一，在 Events 中有文档说明。

例如：

```py
from sqlalchemy import event
from sqlalchemy.schema import UniqueConstraint

def unique_constraint_name(const, table):
    const.name = "uq_%s_%s" % (
        table.name,
        list(const.columns)[0].name
    )
event.listen(
        UniqueConstraint,
        "after_parent_attach",
        unique_constraint_name)
```

参数：

+   `insert`（*bool*） - 事件处理程序的默认行为是在发现时将装饰的用户定义函数附加到注册的事件监听器的内部列表中。如果用户使用 `insert=True` 注册函数，则 SQLAlchemy 将在发现时将函数插入（前置）到内部列表中。此功能通常不由 SQLAlchemy 维护者使用或推荐使用，但提供此功能是为了确保某些用户定义的函数可以在其他函数之前运行，例如在 修改 MySQL 中的 sql_mode 时。

+   `named`（*bool*） - 使用命名参数传递时，函数参数规范中列出的名称将用作字典中的键。参见 命名参数风格。

+   `once`（*bool*） - 私有/内部 API 使用。已弃用。该参数将确保事件函数仅在给定目标上运行一次。但这并不意味着自动取消注册监听器函数；如果不显式地移除监听器函数，即使指定了 `once=True`，也会导致内存无限增长。

+   `propagate`（*bool*） - 在使用 ORM 仪表化和映射事件时可用 `propagate` kwarg。参见 `MapperEvents` 和 `MapperEvents.before_mapper_configured()` 以获取示例。

+   `retval`（*bool*） -

    此标志仅适用于特定的事件监听器，每个监听器都包含解释何时应使用它的文档。默认情况下，没有监听器需要返回值。但是，一些监听器支持返回值的特殊行为，并在其文档中包含 `retval=True` 标志是必要的信息。

    利用 `listen.retval` 的事件监听器套件包括 `ConnectionEvents` 和 `AttributeEvents`。

注意

`listen()` 函数不能在目标事件正在运行时调用。这对线程安全性有影响，并且意味着无法从监听器函数内部添加事件本身。在可变集合中存在要运行的事件列表，在迭代过程中不能更改。

事件注册和移除不打算是“高速”操作；这是一个配置操作。对于需要在高比例上快速关联和取消关联事件的系统，请使用从单个监听器内部处理的可变结构。

另见

`listens_for()`

`remove()`

```py
function sqlalchemy.event.listens_for(target: Any, identifier: str, *args: Any, **kw: Any) → Callable[[Callable[[...], Any]], Callable[[...], Any]]
```

将函数装饰为给定目标 + 标识符的监听器。

`listens_for()` 装饰器是 SQLAlchemy 事件系统的主要接口的一部分，文档位于 Events。

此函数通常与 `listens()` 共享相同的 kwargs。

例如：

```py
from sqlalchemy import event
from sqlalchemy.schema import UniqueConstraint

@event.listens_for(UniqueConstraint, "after_parent_attach")
def unique_constraint_name(const, table):
    const.name = "uq_%s_%s" % (
        table.name,
        list(const.columns)[0].name
    )
```

通过使用 `once` 参数，也可以仅调用给定函数的事件的第一次调用：

```py
@event.listens_for(Mapper, "before_configure", once=True)
def on_config():
    do_config()
```

警告

`once` 参数并不意味着在第一次调用后自动注销监听器函数；监听器条目将保留与目标对象的关联。即使指定了 `once=True`，如果不明确删除，关联任意数量的监听器将导致内存无限增长。

另见

`listen()` - 事件监听的概述

```py
function sqlalchemy.event.remove(target: Any, identifier: str, fn: Callable[[...], Any]) → None
```

移除事件监听器。

此处的参数应与发送到 `listen()` 的参数完全匹配；通过使用相同的参数调用 `remove()` 将撤消由此调用导致的所有事件注册。

例如：

```py
# if a function was registered like this...
@event.listens_for(SomeMappedClass, "before_insert", propagate=True)
def my_listener_function(*arg):
    pass

# ... it's removed like this
event.remove(SomeMappedClass, "before_insert", my_listener_function)
```

在上面的示例中，与 `SomeMappedClass` 关联的监听器函数也被传播到 `SomeMappedClass` 的子类；`remove()` 函数将撤消所有这些操作。

注意

`remove()` 函数不能在目标事件正在运行时调用。这对线程安全性有影响，并且意味着无法从监听器函数内部删除事件本身。在可变集合中存在要运行的事件列表，在迭代过程中不能更改。

事件注册和移除不打算是“高速”操作；这是一个配置操作。对于需要在高比例上快速关联和取消关联事件的系统，请使用从单个监听器内部处理的可变结构。

另见

`listen()`

```py
function sqlalchemy.event.contains(target: Any, identifier: str, fn: Callable[[...], Any]) → bool
```

如果给定的目标/标识/函数设置为监听，则返回 True。

## 事件注册

通过单个 API 点进行事件订阅，即`listen()`函数，或者选择使用`listens_for()`装饰器。这些函数接受一个目标，一个字符串标识符，用于识别要拦截的事件，并且接受一个用户定义的监听函数。这两个函数的其他位置参数和关键字参数可能会被特定类型的事件支持，这些事件可能会指定给定事件函数的替代接口，或者根据给定目标提供关于次要事件目标的说明。

事件的名称和相应监听函数的参数签名是从绑定到在文档中描述的标记类的类绑定规范方法派生的。例如，`PoolEvents.connect()` 的文档指示事件名称是`"connect"`，并且用户定义的监听函数应该接收两个位置参数：

```py
from sqlalchemy.event import listen
from sqlalchemy.pool import Pool

def my_on_connect(dbapi_con, connection_record):
    print("New DBAPI connection:", dbapi_con)

listen(Pool, "connect", my_on_connect)
```

使用`listens_for()`装饰器进行监听如下所示：

```py
from sqlalchemy.event import listens_for
from sqlalchemy.pool import Pool

@listens_for(Pool, "connect")
def my_on_connect(dbapi_con, connection_record):
    print("New DBAPI connection:", dbapi_con)
```

## 命名参数风格

有一些论证风格的变体可以被监听函数接受。以`PoolEvents.connect()`为例，该函数被记录为接收`dbapi_connection`和`connection_record`参数。我们可以选择通过名称接收这些参数，通过建立一个接受`**keyword`参数的监听函数，或者通过将`named=True`传递给`listen()`或`listens_for()`来接收：

```py
from sqlalchemy.event import listens_for
from sqlalchemy.pool import Pool

@listens_for(Pool, "connect", named=True)
def my_on_connect(**kw):
    print("New DBAPI connection:", kw["dbapi_connection"])
```

使用命名参数传递时，函数参数规范中列出的名称将用作字典中的键。

Named 风格通过名称传递所有参数，而不管函数签名如何，因此特定参数也可以列出，以任何顺序，只要名称匹配即可：

```py
from sqlalchemy.event import listens_for
from sqlalchemy.pool import Pool

@listens_for(Pool, "connect", named=True)
def my_on_connect(dbapi_connection, **kw):
    print("New DBAPI connection:", dbapi_connection)
    print("Connection record:", kw["connection_record"])
```

上述代码中的 `**kw` 告诉`listens_for()`应该按名称而不是按位置传递参数给函数。

## 目标

`listen()` 函数在目标方面非常灵活。它通常接受类、这些类的实例以及相关类或对象，从中可以派生出适当的目标。例如，上述提到的 `"connect"` 事件也接受 `Engine` 类和对象，以及 `Pool` 类和对象：

```py
from sqlalchemy.event import listen
from sqlalchemy.pool import Pool, QueuePool
from sqlalchemy import create_engine
from sqlalchemy.engine import Engine
import psycopg2

def connect():
    return psycopg2.connect(user="ed", host="127.0.0.1", dbname="test")

my_pool = QueuePool(connect)
my_engine = create_engine("postgresql+psycopg2://ed@localhost/test")

# associate listener with all instances of Pool
listen(Pool, "connect", my_on_connect)

# associate listener with all instances of Pool
# via the Engine class
listen(Engine, "connect", my_on_connect)

# associate listener with my_pool
listen(my_pool, "connect", my_on_connect)

# associate listener with my_engine.pool
listen(my_engine, "connect", my_on_connect)
```

## 修饰符

一些监听器允许传递修饰符给 `listen()`。这些修饰符有时会为监听器提供替代的调用签名。例如，对于 ORM 事件，一些事件监听器可以有一个返回值，该返回值修改了后续的处理方式。默认情况下，没有任何监听器需要返回值，但是通过传递 `retval=True`，可以支持该值：

```py
def validate_phone(target, value, oldvalue, initiator):
  """Strip non-numeric characters from a phone number"""

    return re.sub(r"\D", "", value)

# setup listener on UserContact.phone attribute, instructing
# it to use the return value
listen(UserContact.phone, "set", validate_phone, retval=True)
```

## 事件和多处理

SQLAlchemy 的事件钩子是通过 Python 函数和对象实现的，因此事件通过 Python 函数调用传播。Python 多进程遵循与我们所思考的操作系统多进程相同的方式，例如父进程分叉出子进程，因此我们可以使用相同的模型描述 SQLAlchemy 事件系统的行为。

在父进程中注册的事件钩子将存在于从该父进程分叉出的新子进程中，因为子进程在生成时从父进程开始时具有所有现有 Python 结构的副本。在注册钩子之前存在的子进程将不会接收到这些新的事件钩子，因为在父进程中对 Python 结构所做的更改不会传播到子进程。

对于事件本身，这些是 Python 函数调用，它们没有任何传播到进程之间的能力。SQLAlchemy 的事件系统不实现任何进程间通信。可能会实现在其中使用 Python 进程间消息传递的事件钩子，但这需要用户实现。

## 事件参考

SQLAlchemy 核心和 SQLAlchemy ORM 都具有各种各样的事件钩子：

+   **核心事件** - 这些在 核心事件 中描述，包括特定于连接池生命周期、SQL 语句执行、事务生命周期和架构创建和拆除的事件钩子。

+   **ORM 事件** - 这些在 ORM 事件 中描述，包括特定于类和属性检测、对象初始化钩子、属性变更钩子、会话状态、刷新和提交钩子、映射器初始化、对象/结果填充和每个实例持久性钩子的事件钩子。

## API 参考

| 对象名称 | 描述 |
| --- | --- |
| contains(target, identifier, fn) | 如果给定的目标/标识符/fn 被设置为监听，则返回 True。 |
| listen(target, identifier, fn, *args, **kw) | 为给定的目标注册一个侦听器函数。 |
| listens_for(target, identifier, *args, **kw) | 将函数装饰为给定目标 + 标识符的侦听器。 |
| remove(target, identifier, fn) | 移除一个事件侦听器。 |

```py
function sqlalchemy.event.listen(target: Any, identifier: str, fn: Callable[[...], Any], *args: Any, **kw: Any) → None
```

为给定的目标注册一个侦听器函数。

`listen()`函数是 SQLAlchemy 事件系统的主要接口之一，在事件中有文档说明。

例如：

```py
from sqlalchemy import event
from sqlalchemy.schema import UniqueConstraint

def unique_constraint_name(const, table):
    const.name = "uq_%s_%s" % (
        table.name,
        list(const.columns)[0].name
    )
event.listen(
        UniqueConstraint,
        "after_parent_attach",
        unique_constraint_name)
```

参数：

+   `insert` (*bool*) – 事件处理程序的默认行为是在发现时将装饰的用户定义函数追加到已注册的事件侦听器的内部列表中。如果用户使用 `insert=True` 注册一个函数，SQLAlchemy 将在发现时将函数插入（预置）到内部列表中。这个功能通常不被 SQLAlchemy 维护者使用或推荐，但为了确保某些用户定义的函数可以在其他函数之前运行，比如更改 MySQL 中的 sql_mode 时，提供了此功能。

+   `named` (*bool*) – 在使用命名参数传递时，函数参数规范中列出的名称将用作字典中的键。请参阅命名参数风格。

+   `once` (*bool*) – 私有/内部 API 使用。已弃用。此参数将提供事件函数仅在给定目标上运行一次。但是，这并不意味着侦听器函数会自动取消注册；如果未显式删除关联的任意数量的侦听器，则即使指定了 `once=True`，内存也会无限增长。

+   `propagate` (*bool*) – 在使用 ORM 仪器和映射事件时，`propagate` 关键字参数是可用的。参见`MapperEvents`和`MapperEvents.before_mapper_configured()`进行示例。

+   `retval` (*bool*) –

    此标志仅适用于特定的事件侦听器，每个事件侦听器都包含说明何时应该使用它的文档。默认情况下，没有侦听器需要返回值。但是，一些侦听器确实支持返回值的特殊行为，并在其文档中包含了必须使用 `retval=True` 标志才能处理返回值的说明。

    利用`listen.retval`的事件侦听器套件包括`ConnectionEvents`和`AttributeEvents`。

注意

`listen()` 函数不能在目标事件正在运行时调用。这对线程安全性有影响，并且还意味着无法从监听器函数内部为自身添加事件。要运行的事件列表存在于一个可变集合内，在迭代期间不能更改。

事件注册和移除并不意味着是“高速”操作；它是一种配置操作。对于需要在高比例下快速关联和取消关联事件的系统，请使用可变结构，并且从单个监听器内部处理。

另请参阅

`listens_for()`

`remove()`

```py
function sqlalchemy.event.listens_for(target: Any, identifier: str, *args: Any, **kw: Any) → Callable[[Callable[[...], Any]], Callable[[...], Any]]
```

将函数装饰为给定目标 + 标识符的监听器。

`listens_for()` 装饰器是 SQLAlchemy 事件系统的主要接口之一，在事件文档中有说明。

此函数通常具有与 `listens()` 相同的 kwargs。

例如：

```py
from sqlalchemy import event
from sqlalchemy.schema import UniqueConstraint

@event.listens_for(UniqueConstraint, "after_parent_attach")
def unique_constraint_name(const, table):
    const.name = "uq_%s_%s" % (
        table.name,
        list(const.columns)[0].name
    )
```

使用 `once` 参数，给定函数还可以仅在事件的第一次调用时被调用：

```py
@event.listens_for(Mapper, "before_configure", once=True)
def on_config():
    do_config()
```

警告

`once` 参数不意味着在第一次调用后自动取消注册监听器函数；监听器条目将保持与目标对象的关联。如果不显式移除，关联任意数量的监听器将导致内存无限增长，即使指定了 `once=True`。

另请参阅

`listen()` - 事件监听的一般描述

```py
function sqlalchemy.event.remove(target: Any, identifier: str, fn: Callable[[...], Any]) → None
```

移除事件监听器。

此处的参数应与发送到 `listen()` 的参数完全匹配；通过使用相同的参数调用 `remove()` 将撤消此调用结果的所有事件注册。

例如：

```py
# if a function was registered like this...
@event.listens_for(SomeMappedClass, "before_insert", propagate=True)
def my_listener_function(*arg):
    pass

# ... it's removed like this
event.remove(SomeMappedClass, "before_insert", my_listener_function)
```

上述中，与 `SomeMappedClass` 关联的监听器函数也传播到了 `SomeMappedClass` 的子类；`remove()` 函数将撤销所有这些操作。

注意

`remove()` 函数不能在目标事件正在运行时调用。这对线程安全性有影响，并且还意味着无法从监听器函数内部移除事件本身。要运行的事件列表存在于一个可变集合内，在迭代期间不能更改。

事件注册和移除并不意味着是“高速”操作；它是一种配置操作。对于需要在高比例下快速关联和取消关联事件的系统，请使用可变结构，并且从单个监听器内部处理。

另请参阅

`listen()`

```py
function sqlalchemy.event.contains(target: Any, identifier: str, fn: Callable[[...], Any]) → bool
```

如果给定的目标/标识符/函数已设置为监听，则返回 True。
