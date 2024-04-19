# 使用事件跟踪查询、对象和会话更改

> 原文：[`docs.sqlalchemy.org/en/20/orm/session_events.html`](https://docs.sqlalchemy.org/en/20/orm/session_events.html)

SQLAlchemy 特色是一个广泛的事件监听系统，贯穿于核心和 ORM 中。在 ORM 中，有各种各样的事件监听器钩子，这些钩子在 ORM 事件的 API 级别有文档记录。这些事件的集合多年来已经增长，包括许多非常有用的新事件，以及一些曾经不那么相关的旧事件。本节将尝试介绍主要的事件钩子以及它们何时可能被使用。

## 执行事件

从版本 1.4 新增：现在`Session`提供了一个单一全面的钩子，用于拦截 ORM 代表进行的所有 SELECT 语句，以及大量的 UPDATE 和 DELETE 语句。这个钩子取代了以前的`QueryEvents.before_compile()`事件以及`QueryEvents.before_compile_update()`和`QueryEvents.before_compile_delete()`。

`Session`提供了一个全面的系统，通过该系统，通过`Session.execute()`方法调用的所有查询，包括由`Query`发出的所有 SELECT 语句以及代表列和关系加载器发出的所有 SELECT 语句，都可以被拦截和修改。该系统使用`SessionEvents.do_orm_execute()`事件钩子以及`ORMExecuteState`对象来表示事件状态。

### 基本查询拦截

`SessionEvents.do_orm_execute()` 首先对查询的任何拦截都是有用的，包括由 `Query` 以 1.x 风格 发出的查询，以及当 ORM 启用的 2.0 风格 下传递给 `Session.execute()` 的 `select()`、`update()` 或 `delete()` 构造。`ORMExecuteState` 构造提供了访问器，以允许修改语句、参数和选项：

```py
Session = sessionmaker(engine)

@event.listens_for(Session, "do_orm_execute")
def _do_orm_execute(orm_execute_state):
    if orm_execute_state.is_select:
        # add populate_existing for all SELECT statements

        orm_execute_state.update_execution_options(populate_existing=True)

        # check if the SELECT is against a certain entity and add an
        # ORDER BY if so
        col_descriptions = orm_execute_state.statement.column_descriptions

        if col_descriptions[0]["entity"] is MyEntity:
            orm_execute_state.statement = statement.order_by(MyEntity.name)
```

上述示例说明了对 SELECT 语句进行的一些简单修改。在这个层次上，`SessionEvents.do_orm_execute()` 事件钩子旨在取代之前使用的 `QueryEvents.before_compile()` 事件，该事件对各种加载器的各种类型并不一致地触发；此外，`QueryEvents.before_compile()` 仅适用于与 `Query` 的 1.x 风格 一起使用，而不适用于 2.0 风格 下使用 `Session.execute()`。

### 添加全局的 WHERE / ON 条件

最常请求的查询扩展功能之一是向所有查询中的所有实体添加 WHERE 条件的能力。这可以通过使用 `with_loader_criteria()` 查询选项来实现，该选项可以单独使用，也可以在 `SessionEvents.do_orm_execute()` 事件中使用：

```py
from sqlalchemy.orm import with_loader_criteria

Session = sessionmaker(engine)

@event.listens_for(Session, "do_orm_execute")
def _do_orm_execute(orm_execute_state):
    if (
        orm_execute_state.is_select
        and not orm_execute_state.is_column_load
        and not orm_execute_state.is_relationship_load
    ):
        orm_execute_state.statement = orm_execute_state.statement.options(
            with_loader_criteria(MyEntity.public == True)
        )
```

在上述内容中，为所有 SELECT 语句添加了一个选项，该选项将限制针对`MyEntity`的所有查询以在`public == True`上进行过滤。该条件将应用于立即查询范围内该类的**所有**加载。`with_loader_criteria()`选项默认情况下还会自动传播到关系加载程序，这将应用于后续的关系加载，包括惰性加载，selectinloads 等。

对于一系列具有某些共同列结构的类，如果使用声明性混合来组合类，那么混合类本身可以与`with_loader_criteria()`选项结合使用，通过使用 Python lambda 来使用。 Python lambda 将针对与条件匹配的特定实体在查询编译时调用。假设一系列基于名为`HasTimestamp`的混合物的类：

```py
import datetime

class HasTimestamp:
    timestamp = mapped_column(DateTime, default=datetime.datetime.now)

class SomeEntity(HasTimestamp, Base):
    __tablename__ = "some_entity"
    id = mapped_column(Integer, primary_key=True)

class SomeOtherEntity(HasTimestamp, Base):
    __tablename__ = "some_entity"
    id = mapped_column(Integer, primary_key=True)
```

上述类`SomeEntity`和`SomeOtherEntity`将分别具有一个`timestamp`列，其默认值为当前日期和时间。可以使用事件拦截所有扩展自`HasTimestamp`并过滤其`timestamp`列的对象，使其日期不晚于一个月前：

```py
@event.listens_for(Session, "do_orm_execute")
def _do_orm_execute(orm_execute_state):
    if (
        orm_execute_state.is_select
        and not orm_execute_state.is_column_load
        and not orm_execute_state.is_relationship_load
    ):
        one_month_ago = datetime.datetime.today() - datetime.timedelta(months=1)

        orm_execute_state.statement = orm_execute_state.statement.options(
            with_loader_criteria(
                HasTimestamp,
                lambda cls: cls.timestamp >= one_month_ago,
                include_aliases=True,
            )
        )
```

警告

在调用`with_loader_criteria()`时使用 lambda 仅被调用**一次每个唯一类**。在此 lambda 内部不应调用自定义函数。请参阅使用 Lambda 将重要的速度增益添加到语句生成以获取“lambda SQL”功能的概述，该功能仅用于高级用途。

另请参阅

ORM 查询事件 - 包括上述`with_loader_criteria()`配方的工作示例。 ### 重新执行语句

深度炼金术

语句重新执行功能涉及稍微复杂的递归序列，并旨在解决将 SQL 语句的执行重新路由到各种非 SQL 上下文的相当困难的问题。下面链接的“狗窝缓存”和“水平分片”的双例应该用作指导，以确定何时适合使用此相当高级的功能。

`ORMExecuteState`能够控制给定语句的执行；这包括能力要么根本不调用语句，而是返回一个从缓存中检索到的预先构建的结果集，要么调用相同的语句多次，每次使用不同的状态，例如对多个数据库连接调用它，然后在内存中合并结果。这两种高级模式都在 SQLAlchemy 的示例套件中有详细说明。

在`SessionEvents.do_orm_execute()`事件钩子内部时，可以使用`ORMExecuteState.invoke_statement()`方法来使用新的嵌套调用`Session.execute()`来调用语句，这将抢占当前正在进行的执行的后续处理，而是返回内部执行返回的`Result`。因此，在此过程中已调用的`SessionEvents.do_orm_execute()`钩子的事件处理程序也将在此嵌套调用中被跳过。

`ORMExecuteState.invoke_statement()`方法返回一个`Result`对象；然后该对象具有冻结为可缓存格式和解冻为新的`Result`对象的能力，以及将其数据与其他`Result`对象的数据合并的能力。

例如，使用`SessionEvents.do_orm_execute()`来实现缓存：

```py
from sqlalchemy.orm import loading

cache = {}

@event.listens_for(Session, "do_orm_execute")
def _do_orm_execute(orm_execute_state):
    if "my_cache_key" in orm_execute_state.execution_options:
        cache_key = orm_execute_state.execution_options["my_cache_key"]

        if cache_key in cache:
            frozen_result = cache[cache_key]
        else:
            frozen_result = orm_execute_state.invoke_statement().freeze()
            cache[cache_key] = frozen_result

        return loading.merge_frozen_result(
            orm_execute_state.session,
            orm_execute_state.statement,
            frozen_result,
            load=False,
        )
```

当上述钩子生效时，使用缓存的示例如下：

```py
stmt = (
    select(User).where(User.name == "sandy").execution_options(my_cache_key="key_sandy")
)

result = session.execute(stmt)
```

上面的例子中，自定义的执行选项被传递给`Select.execution_options()`，以建立一个“缓存键”，然后该键将被`SessionEvents.do_orm_execute()`钩子拦截。然后，这个缓存键将与可能存在于缓存中的`FrozenResult`对象进行匹配，并且如果存在，则重新使用该对象。该示例利用了`Result.freeze()`方法来“冻结”一个包含 ORM 结果的`Result`对象，以便它可以被存储在缓存中并多次使用。为了从“冻结”结果中返回一个活动结果，使用`merge_frozen_result()`函数将结果对象中的“冻结”数据合并到当前会话中。

上面的例子在 Dogpile 缓存中作为一个完整的例子实现。

`ORMExecuteState.invoke_statement()`方法也可以被多次调用，传递不同的信息给`ORMExecuteState.invoke_statement.bind_arguments`参数，以便`Session`每次都使用不同的`Engine`对象。这将每次返回一个不同的`Result`对象；这些结果可以使用`Result.merge()`方法合并在一起。这是水平分片扩展所采用的技术；请查看源代码以熟悉它。

另请参阅

Dogpile 缓存

水平分片  ## 持久化事件

可能是最广泛使用的一系列事件是“持久化”事件，它们对应于刷新过程。刷新是所有关于待处理对象更改的决定都会被做出，并以 INSERT、UPDATE 和 DELETE 语句的形式发送到数据库的地方。

### `before_flush()`

`SessionEvents.before_flush()` 钩子是当应用程序希望在提交刷新时确保额外的持久性更改被执行时最常用的事件。 使用 `SessionEvents.before_flush()` 来验证对象的状态并在持久化之前组合其他对象和引用。在此事件中，**可以安全地操纵会话的状态**，即可以附加新对象，删除对象，并且可以自由更改对象上的单个属性，这些更改将在事件钩子完成时被纳入刷新过程中。

典型的 `SessionEvents.before_flush()` 钩子将被指示扫描集合 `Session.new`、`Session.dirty` 和 `Session.deleted`，以查找将要发生更改的对象。

有关 `SessionEvents.before_flush()` 的示例，请参见具有历史表的版本控制和使用时间行进行版本控制等示例。

### `after_flush()`

`SessionEvents.after_flush()` 钩子在刷新过程的 SQL 被生成之后，但在被刷新的对象状态被更改之前调用。也就是说，您仍然可以检查 `Session.new`、`Session.dirty` 和 `Session.deleted` 集合，以查看刚刷新的内容，并且还可以使用像 `AttributeState` 这样的历史跟踪功能来查看刚刚持久化的更改。在 `SessionEvents.after_flush()` 事件中，可以根据观察到的更改向数据库发送额外的 SQL。

### `after_flush_postexec()`

`SessionEvents.after_flush_postexec()` 在`SessionEvents.after_flush()` 之后不久调用，但是在对象状态已经被修改以考虑刚刚发生的刷新之后调用。`Session.new`、`Session.dirty` 和 `Session.deleted` 集合通常在此时完全为空。使用 `SessionEvents.after_flush_postexec()` 检查最终对象的标识映射，并可能发出附加的 SQL。在这个钩子中，有能力对对象进行新的更改，这意味着 `Session` 再次进入“脏”状态；`Session` 的机制会导致如果在此钩子中检测到新的更改，那么再次刷新如果在 `Session.commit()` 的上下文中调用了刷新；否则，待定更改将作为下一个正常刷新的一部分进行捆绑。当钩子在 `Session.commit()` 中检测到新的更改时，一个计数器确保在每次调用时，如果 `SessionEvents.after_flush_postexec()` 钩子持续添加新状态以刷新，则此方面的无限循环在 100 次迭代后停止。

### 映射器级刷新事件

除了刷新级别的钩子外，还有一套更精细的钩子，这些钩子更加细致，因为它们是基于每个对象调用的，并且根据刷新过程中的 INSERT、UPDATE 或 DELETE 进行分组。这些是映射器持久性钩子，它们也非常受欢迎，但是需要更加谨慎地对待这些事件，因为它们在已经进行的刷新过程的上下文中进行；在这里进行许多操作是不安全的。

这些事件是：

+   `MapperEvents.before_insert()` 

+   `MapperEvents.after_insert()` 

+   `MapperEvents.before_update()`

+   `MapperEvents.after_update()`

+   `MapperEvents.before_delete()`

+   `MapperEvents.after_delete()`

注意

需要注意的是，这些事件仅适用于会话刷新操作，而不适用于在 ORM-启用的 INSERT、UPDATE 和 DELETE 语句中描述的 ORM 级别的 INSERT/UPDATE/DELETE 功能。要拦截 ORM 级别的 DML，请使用`SessionEvents.do_orm_execute()`事件。

每个事件都会传递`Mapper`、映射对象本身以及用于发出 INSERT、UPDATE 或 DELETE 语句的`Connection`。这些事件的吸引力显而易见，因为如果应用程序想要将某些活动绑定到特定类型的对象在 INSERT 时被持久化的时间，钩子就非常具体；不像`SessionEvents.before_flush()`事件，不需要搜索诸如`Session.new`之类的集合以找到目标。然而，当调用这些事件时，表示完整列表的刷新计划，即将发出的每个单独的 INSERT、UPDATE、DELETE 语句已经**已经决定**，在这个阶段不允许进行任何更改。因此，甚至对于给定对象的其他属性也只能进行**局部**更改。对对象或其他对象的任何其他更改将影响`Session`的状态，这将导致其无法正常运行。

在这些映射器级持久性事件中不支持的操作包括：

+   `Session.add()`

+   `Session.delete()`

+   映射集合追加、添加、移除、删除、丢弃等操作。

+   映射关系属性设置/删除事件，即`someobject.related = someotherobject`

传递`Connection`的原因是鼓励**在这里进行简单的 SQL 操作**，直接在`Connection`上进行，例如在日志表中递增计数器或插入额外行。

也有许多每个对象操作根本不需要在刷新事件中处理。最常见的替代方法是在对象的`__init__()`方法中简单地建立额外的状态，例如创建要与新对象关联的其他对象。使用 Simple Validators 中描述的验证器是另一种方法；这些函数可以拦截属性的更改，并在响应属性更改时在目标对象上建立额外的状态更改。使用这两种方法，对象在到达刷新步骤之前就处于正确的状态。## 对象生命周期事件

事件的另一个用例是跟踪对象的生命周期。这指的是首次介绍于 Quickie Intro to Object States 的状态。

所有上述状态都可以完全通过事件进行跟踪。每个事件代表着一个独立的状态转换，意味着起始状态和目标状态都是被跟踪的一部分。除了初始的瞬态事件之外，所有事件都是以`Session`对象或类的形式出现的，这意味着它们可以与特定的`Session`对象关联：

```py
from sqlalchemy import event
from sqlalchemy.orm import Session

session = Session()

@event.listens_for(session, "transient_to_pending")
def object_is_pending(session, obj):
    print("new pending: %s" % obj)
```

或者使用`Session`类本身，以及特定的`sessionmaker`，这可能是最有用的形式：

```py
from sqlalchemy import event
from sqlalchemy.orm import sessionmaker

maker = sessionmaker()

@event.listens_for(maker, "transient_to_pending")
def object_is_pending(session, obj):
    print("new pending: %s" % obj)
```

监听器当然可以堆叠在一个函数上，这很可能是常见的情况。例如，要跟踪所有进入持久状态的对象：

```py
@event.listens_for(maker, "pending_to_persistent")
@event.listens_for(maker, "deleted_to_persistent")
@event.listens_for(maker, "detached_to_persistent")
@event.listens_for(maker, "loaded_as_persistent")
def detect_all_persistent(session, instance):
    print("object is now persistent: %s" % instance)
```

### 瞬态

所有映射对象在首次构建时都是瞬态的。在这种状态下，对象独立存在，不与任何`Session`关联。对于这种初始状态，没有特定的“转换”事件，因为没有`Session`，但是如果想要拦截任何瞬态对象被创建时，`InstanceEvents.init()`方法可能是最好的事件。此事件应用于特定类或超类。例如，要拦截特定声明基类的所有新对象：

```py
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy import event

class Base(DeclarativeBase):
    pass

@event.listens_for(Base, "init", propagate=True)
def intercept_init(instance, args, kwargs):
    print("new transient: %s" % instance)
```

### 瞬态到待定

当瞬态对象首次通过`Session.add()`或`Session.add_all()`方法与`Session`关联时，瞬态对象变为待定。对象也可能作为引用对象的“级联”结果成为`Session`的一部分，该引用对象是显式添加的。使用`SessionEvents.transient_to_pending()`事件检测瞬态到待定的转换过程：

```py
@event.listens_for(sessionmaker, "transient_to_pending")
def intercept_transient_to_pending(session, object_):
    print("transient to pending: %s" % object_)
```

### 待定到持久化

当一个刷新操作进行并且为实例执行 INSERT 语句时，待定对象变为持久化。该对象现在具有标识键。使用`SessionEvents.pending_to_persistent()`事件跟踪待定到持久化的过程：

```py
@event.listens_for(sessionmaker, "pending_to_persistent")
def intercept_pending_to_persistent(session, object_):
    print("pending to persistent: %s" % object_)
```

### 待定到瞬态

如果在待定对象被刷新之前调用`Session.rollback()`方法，或者在刷新对象之前调用`Session.expunge()`方法，则待定对象可以回退到瞬态状态。使用`SessionEvents.pending_to_transient()`事件跟踪待定到瞬态的过程：

```py
@event.listens_for(sessionmaker, "pending_to_transient")
def intercept_pending_to_transient(session, object_):
    print("transient to pending: %s" % object_)
```

### 作为持久化加载

当对象从数据库加载时，它们可以直接进入 `Session` 中的 persistent 状态。跟踪此状态转换等同于跟踪对象加载的方式，并且等同于使用 `InstanceEvents.load()` 实例级事件。但是，`SessionEvents.loaded_as_persistent()` 事件作为一个会话中心的钩子提供，用于拦截通过这种特定方式进入持久化状态的对象：

```py
@event.listens_for(sessionmaker, "loaded_as_persistent")
def intercept_loaded_as_persistent(session, object_):
    print("object loaded into persistent state: %s" % object_)
```

### 持久化到瞬时

如果针对对象首次作为待处理对象添加的事务调用了 `Session.rollback()` 方法，持久化对象可以恢复到瞬时状态。在 ROLLBACK 的情况下，将该对象持久化的 INSERT 语句回滚，并将对象从 `Session` 中驱逐，使其再次成为瞬时状态。使用 `SessionEvents.persistent_to_transient()` 事件钩子跟踪从持久化恢复为瞬时的对象：

```py
@event.listens_for(sessionmaker, "persistent_to_transient")
def intercept_persistent_to_transient(session, object_):
    print("persistent to transient: %s" % object_)
```

### 持久化到删除

当在 flush 过程中从数据库中删除了标记为删除的对象时，持久化对象进入 deleted 状态。请注意，这**与**调用 `Session.delete()` 方法删除目标对象时**并不相同**。`Session.delete()` 方法只是将对象标记为删除；直到 flush 进行之后才会发出实际的 DELETE 语句。在 flush 进行之后，目标对象的“deleted”状态才存在。

在“deleted”状态中，对象与 `Session` 仅有轻微关联。它既不在标识映射中，也不在指示其曾待删除的 `Session.deleted` 集合中。

从“deleted”状态，当事务提交时，对象可以进入分离状态，或者如果事务被回滚，则可以重新进入持久化状态。

使用 `SessionEvents.persistent_to_deleted()` 跟踪持久化到删除的转换：

```py
@event.listens_for(sessionmaker, "persistent_to_deleted")
def intercept_persistent_to_deleted(session, object_):
    print("object was DELETEd, is now in deleted state: %s" % object_)
```

### 已删除到已分离

当会话的事务提交时，已删除对象将变为分离。在调用 `Session.commit()` 方法后，数据库事务已完成，`Session` 现在完全丢弃了已删除对象并删除了所有与其相关的关联。使用 `SessionEvents.deleted_to_detached()` 跟踪已删除到分离的转换：

```py
@event.listens_for(sessionmaker, "deleted_to_detached")
def intercept_deleted_to_detached(session, object_):
    print("deleted to detached: %s" % object_)
```

注意

当对象处于已删除状态时，`InstanceState.deleted` 属性，可使用 `inspect(object).deleted` 访问，将返回 True。但是，当对象分离时，`InstanceState.deleted` 将再次返回 False。要检测对象是否已删除，无论它是否已分离，请使用 `InstanceState.was_deleted` 访问器。

### 持久到分离

当对象与 `Session` 解除关联时，通过 `Session.expunge()`、`Session.expunge_all()` 或 `Session.close()` 方法，持久对象将变为分离状态。

注意

如果一个对象的拥有者 `Session` 被应用程序解除引用并由于垃圾回收而被丢弃，该对象也可能会**隐式分离**。在这种情况下，**不会发出任何事件**。

使用 `SessionEvents.persistent_to_detached()` 事件跟踪对象从持久状态转为分离状态：

```py
@event.listens_for(sessionmaker, "persistent_to_detached")
def intercept_persistent_to_detached(session, object_):
    print("object became detached: %s" % object_)
```

### 分离到持久

当分离对象使用 `Session.add()` 或等效方法重新关联到会话时，它将变为持久对象。跟踪对象从分离状态返回持久状态时使用 `SessionEvents.detached_to_persistent()` 事件：

```py
@event.listens_for(sessionmaker, "detached_to_persistent")
def intercept_detached_to_persistent(session, object_):
    print("object became persistent again: %s" % object_)
```

### 已删除到持久

当删除对象在其所在的事务被回滚时，可以将其恢复为持久状态，使用`Session.rollback()`方法回滚。使用`SessionEvents.deleted_to_persistent()`事件跟踪将删除的对象移回持久状态：

```py
@event.listens_for(sessionmaker, "deleted_to_persistent")
def intercept_deleted_to_persistent(session, object_):
    print("deleted to persistent: %s" % object_)
```  ## 事务事件

事务事件允许应用在`Session`级别上发生事务边界时收到通知，以及当`Session`更改`Connection`对象上的事务状态时。

+   `SessionEvents.after_transaction_create()`，`SessionEvents.after_transaction_end()` - 这些事件跟踪`Session`的逻辑事务范围，不特定于单个数据库连接。这些事件旨在帮助集成事务跟踪系统，如`zope.sqlalchemy`。当应用程序需要将某些外部范围与`Session`的事务范围对齐时，请使用这些事件。这些挂钩反映了`Session`的“嵌套”事务行为，因为它们跟踪逻辑“子事务”以及“嵌套”（例如，SAVEPOINT）事务。

+   `SessionEvents.before_commit()`、`SessionEvents.after_commit()`、`SessionEvents.after_begin()`、`SessionEvents.after_rollback()`、`SessionEvents.after_soft_rollback()` - 这些事件允许从数据库连接的角度跟踪事务事件。特别是`SessionEvents.after_begin()`是一个每个连接的事件；一个维护多个连接的`Session`将为每个连接在当前事务中使用时单独发出此事件。然后回滚和提交事件指的是 DBAPI 连接自身直接接收回滚或提交指令的时候。

## 属性更改事件

属性更改事件允许拦截对象上特定属性被修改的时机。这些事件包括`AttributeEvents.set()`、`AttributeEvents.append()`和`AttributeEvents.remove()`。这些事件非常有用，特别是对于每个对象的验证操作；然而，使用“验证器”钩子通常更加方便，它在幕后使用这些钩子；请参阅 Simple Validators 以了解背景信息。属性事件也是反向引用机制的基础。一个说明属性事件使用的示例在 Attribute Instrumentation 中。

## 执行事件

新版本 1.4 中新增：`Session`现在具有一个全面的钩子，旨在拦截所有代表 ORM 执行的 SELECT 语句以及批量 UPDATE 和 DELETE 语句。这个钩子取代了之前的`QueryEvents.before_compile()`事件以及`QueryEvents.before_compile_update()`和`QueryEvents.before_compile_delete()`。

`Session`具有一个全面的系统，通过该系统可以拦截和修改通过`Session.execute()`方法调用的所有查询，其中包括由`Query`发出的所有 SELECT 语句以及所有代表列和关系加载程序发出的 SELECT 语句。该系统利用了`SessionEvents.do_orm_execute()`事件钩子以及`ORMExecuteState`对象来表示事件状态。

### 基本查询拦截

`SessionEvents.do_orm_execute()`首先对查询的任何拦截都是有用的，这包括由`Query`发出的 1.x 风格以及当 ORM 启用的 2.0 风格的`select()`，`update()`或`delete()`构造被传递给`Session.execute()`时。`ORMExecuteState`构造提供了访问器，允许修改语句、参数和选项：

```py
Session = sessionmaker(engine)

@event.listens_for(Session, "do_orm_execute")
def _do_orm_execute(orm_execute_state):
    if orm_execute_state.is_select:
        # add populate_existing for all SELECT statements

        orm_execute_state.update_execution_options(populate_existing=True)

        # check if the SELECT is against a certain entity and add an
        # ORDER BY if so
        col_descriptions = orm_execute_state.statement.column_descriptions

        if col_descriptions[0]["entity"] is MyEntity:
            orm_execute_state.statement = statement.order_by(MyEntity.name)
```

上面的示例说明了对 SELECT 语句的一些简单修改。在这个级别上，`SessionEvents.do_orm_execute()` 事件钩子旨在替换以前对各种加载器不一致触发的 `QueryEvents.before_compile()` 事件的使用；此外，`QueryEvents.before_compile()` 仅适用于 1.x 样式 与 `Query` 一起使用，并不适用于 2.0 样式 与 `Session.execute()` 一起使用。

### 添加全局 WHERE / ON 条件

最常请求的查询扩展功能之一是能够向所有查询中的所有实体添加 WHERE 条件。通过使用 `with_loader_criteria()` 查询选项，可以实现此目的，该选项可以单独使用，或者最好在 `SessionEvents.do_orm_execute()` 事件中使用：

```py
from sqlalchemy.orm import with_loader_criteria

Session = sessionmaker(engine)

@event.listens_for(Session, "do_orm_execute")
def _do_orm_execute(orm_execute_state):
    if (
        orm_execute_state.is_select
        and not orm_execute_state.is_column_load
        and not orm_execute_state.is_relationship_load
    ):
        orm_execute_state.statement = orm_execute_state.statement.options(
            with_loader_criteria(MyEntity.public == True)
        )
```

上面，所有 SELECT 语句都添加了一个选项，将限制针对 `MyEntity` 的所有查询，以在 `public == True` 上进行过滤。这些条件将应用于立即查询范围内该类的**所有**加载。`with_loader_criteria()` 选项默认情况下也会自动传播到关系加载器，这将应用于后续的关系加载，包括延迟加载、selectinloads 等。

对于一系列具有某些共同列结构的类，如果这些类使用 declarative mixin 进行组合，那么 mixin 类本身可以与 `with_loader_criteria()` 选项结合使用，方法是使用 Python lambda。Python lambda 将在查询编译时针对符合条件的特定实体被调用。假设有一系列基于名为 `HasTimestamp` 的 mixin 的类：

```py
import datetime

class HasTimestamp:
    timestamp = mapped_column(DateTime, default=datetime.datetime.now)

class SomeEntity(HasTimestamp, Base):
    __tablename__ = "some_entity"
    id = mapped_column(Integer, primary_key=True)

class SomeOtherEntity(HasTimestamp, Base):
    __tablename__ = "some_entity"
    id = mapped_column(Integer, primary_key=True)
```

上述类 `SomeEntity` 和 `SomeOtherEntity` 将分别具有一个默认为当前日期和时间的列 `timestamp`。可以使用事件拦截从 `HasTimestamp` 扩展的所有对象，并在一个月前之内的日期上过滤它们的 `timestamp` 列：

```py
@event.listens_for(Session, "do_orm_execute")
def _do_orm_execute(orm_execute_state):
    if (
        orm_execute_state.is_select
        and not orm_execute_state.is_column_load
        and not orm_execute_state.is_relationship_load
    ):
        one_month_ago = datetime.datetime.today() - datetime.timedelta(months=1)

        orm_execute_state.statement = orm_execute_state.statement.options(
            with_loader_criteria(
                HasTimestamp,
                lambda cls: cls.timestamp >= one_month_ago,
                include_aliases=True,
            )
        )
```

警告

在调用`with_loader_criteria()`时使用 lambda 只会**每个唯一类调用一次**。在此 lambda 内部不应调用自定义函数。有关“lambda SQL”功能的概述，请参阅使用 Lambda 为语句生成带来显著速度提升，这仅适用于高级用法。

另请参阅

ORM 查询事件 - 包括上述`with_loader_criteria()`示例的工作示例。### 重新执行语句

深度炼金术

重新执行功能涉及稍微复杂的递归序列，并旨在解决能够将 SQL 语句的执行重新路由到各种非 SQL 上下文的相当困难的问题。下面链接的“狗窝缓存”和“水平分片”这两个示例应该作为指导，指出何时适合使用这个相当高级的功能。

`ORMExecuteState`能够控制给定语句的执行；这包括不执行语句的能力，允许从缓存中检索到的预构建结果集返回，以及多次以不同状态调用相同语句的能力，例如针对多个数据库连接调用它，然后在内存中合并结果。这两种高级模式在 SQLAlchemy 的示例套件中有详细展示。

当在`SessionEvents.do_orm_execute()`事件钩子内部时，可以使用`ORMExecuteState.invoke_statement()`方法来使用新的嵌套调用`Session.execute()`来调用语句，这将预先中断当前正在进行的执行的后续处理，而是返回内部执行返回的`Result`。在此过程中为`SessionEvents.do_orm_execute()`钩子调用的事件处理程序也将在此嵌套调用中被跳过。

`ORMExecuteState.invoke_statement()`方法返回一个`Result`对象；该对象具有将其“冻结”为可缓存格式并“解冻”为新的`Result`对象的能力，以及将其数据与其他`Result`对象的数据合并的能力。

例如，使用`SessionEvents.do_orm_execute()`来实现缓存：

```py
from sqlalchemy.orm import loading

cache = {}

@event.listens_for(Session, "do_orm_execute")
def _do_orm_execute(orm_execute_state):
    if "my_cache_key" in orm_execute_state.execution_options:
        cache_key = orm_execute_state.execution_options["my_cache_key"]

        if cache_key in cache:
            frozen_result = cache[cache_key]
        else:
            frozen_result = orm_execute_state.invoke_statement().freeze()
            cache[cache_key] = frozen_result

        return loading.merge_frozen_result(
            orm_execute_state.session,
            orm_execute_state.statement,
            frozen_result,
            load=False,
        )
```

有了上述钩子，使用缓存的示例如下：

```py
stmt = (
    select(User).where(User.name == "sandy").execution_options(my_cache_key="key_sandy")
)

result = session.execute(stmt)
```

以上，在`Select.execution_options()`中传递了一个自定义执行选项，以建立一个“缓存键”，然后会被`SessionEvents.do_orm_execute()`钩子拦截。这个缓存键然后会与可能存在于缓存中的`FrozenResult`对象匹配，如果存在，则会重新使用该对象。该示例利用了`Result.freeze()`方法来“冻结”一个`Result`对象，其中将包含 ORM 结果，以便将其存储在缓存中并多次使用。为了从“冻结”结果中返回一个实时结果，使用`merge_frozen_result()`函数将结果对象中的“冻结”数据合并到当前会话中。

上述示例在 Dogpile Caching 中作为一个完整示例实现。

`ORMExecuteState.invoke_statement()` 方法也可以被多次调用，传递不同的信息给 `ORMExecuteState.invoke_statement.bind_arguments` 参数，以便 `Session` 每次都使用不同的 `Engine` 对象。每次都会返回一个不同的 `Result` 对象；这些结果可以使用 `Result.merge()` 方法合并在一起。这是 水平分片 扩展所使用的技术；请查看源代码以熟悉。

另请参阅

Dogpile Caching

水平分片

### 基本查询拦截

`SessionEvents.do_orm_execute()` 首先适用于任何类型的查询拦截，包括由 `Query` 以 1.x 样式 发出的查询，以及当启用 ORM 的 2.0 样式 `select()`、`update()` 或 `delete()` 构造传递给 `Session.execute()` 时。 `ORMExecuteState` 构造提供了访问器，允许对语句、参数和选项进行修改：

```py
Session = sessionmaker(engine)

@event.listens_for(Session, "do_orm_execute")
def _do_orm_execute(orm_execute_state):
    if orm_execute_state.is_select:
        # add populate_existing for all SELECT statements

        orm_execute_state.update_execution_options(populate_existing=True)

        # check if the SELECT is against a certain entity and add an
        # ORDER BY if so
        col_descriptions = orm_execute_state.statement.column_descriptions

        if col_descriptions[0]["entity"] is MyEntity:
            orm_execute_state.statement = statement.order_by(MyEntity.name)
```

上述示例说明了对 SELECT 语句的一些简单修改。在此级别上，`SessionEvents.do_orm_execute()`事件挂钩旨在替换先前使用的`QueryEvents.before_compile()`事件，该事件对各种加载程序未始终触发；另外，`QueryEvents.before_compile()`仅适用于 1.x 样式与`Query`一起使用，而不适用于 2.0 样式使用`Session.execute()`。  

### 添加全局 WHERE / ON 条件

最常请求的查询扩展功能之一是能够向所有查询中的实体添加 WHERE 条件的能力。这可以通过使用`with_loader_criteria()`查询选项来实现，该选项可以单独使用，或者最好在`SessionEvents.do_orm_execute()`事件中使用：

```py
from sqlalchemy.orm import with_loader_criteria

Session = sessionmaker(engine)

@event.listens_for(Session, "do_orm_execute")
def _do_orm_execute(orm_execute_state):
    if (
        orm_execute_state.is_select
        and not orm_execute_state.is_column_load
        and not orm_execute_state.is_relationship_load
    ):
        orm_execute_state.statement = orm_execute_state.statement.options(
            with_loader_criteria(MyEntity.public == True)
        )
```

在上述示例中，一个选项被添加到所有 SELECT 语句中，该选项将限制所有针对`MyEntity`的查询以在`public == True`上进行过滤。这些条件将应用于立即查询范围内该类的**所有**加载。`with_loader_criteria()`选项默认情况下也会自动传播到关系加载程序，这将应用于后续的关系加载，包括惰性加载、selectinloads 等。

对于一系列具有一些共同列结构的类，如果使用声明性混合组合类，那么混合类本身可以与`with_loader_criteria()`选项一起使用，方法是使用 Python lambda。Python lambda 将在与条件匹配的特定实体的查询编译时间调用。假设有一系列基于名为`HasTimestamp`的混合类的类：

```py
import datetime

class HasTimestamp:
    timestamp = mapped_column(DateTime, default=datetime.datetime.now)

class SomeEntity(HasTimestamp, Base):
    __tablename__ = "some_entity"
    id = mapped_column(Integer, primary_key=True)

class SomeOtherEntity(HasTimestamp, Base):
    __tablename__ = "some_entity"
    id = mapped_column(Integer, primary_key=True)
```

上述类`SomeEntity`和`SomeOtherEntity`将各自具有一个列`timestamp`，默认值为当前日期和时间。可以使用事件拦截所有从`HasTimestamp`扩展的对象，并将它们的`timestamp`列过滤为一个月前的日期：

```py
@event.listens_for(Session, "do_orm_execute")
def _do_orm_execute(orm_execute_state):
    if (
        orm_execute_state.is_select
        and not orm_execute_state.is_column_load
        and not orm_execute_state.is_relationship_load
    ):
        one_month_ago = datetime.datetime.today() - datetime.timedelta(months=1)

        orm_execute_state.statement = orm_execute_state.statement.options(
            with_loader_criteria(
                HasTimestamp,
                lambda cls: cls.timestamp >= one_month_ago,
                include_aliases=True,
            )
        )
```

警告

在调用 `with_loader_criteria()` 时，使用 lambda 内部的调用**每个唯一类仅调用一次**。自定义函数不应在此 lambda 内部调用。请参阅 使用 Lambda 来为语句生成带来显著的速度提升 以获得“lambda SQL”功能的概述，该功能仅供高级使用。

另请参阅

ORM 查询事件 - 包括上述 `with_loader_criteria()` 配方的工作示例。

### 重新执行语句

深度炼金术

语句重新执行功能涉及稍微复杂的递归序列，并且旨在解决将 SQL 语句的执行重新路由到各种非 SQL 上下文的相当困难的问题。下面链接的“狗窝缓存”和“水平分片”两个示例应该用作指导，以确定何时使用此相当高级的功能。

`ORMExecuteState` 能够控制给定语句的执行；这包括不执行该语句的能力，允许从缓存中检索到的预构造结果集被返回，以及多次以不同状态调用相同语句的能力，例如对多个数据库连接执行它，然后在内存中合并结果。这两种高级模式都在下面详细介绍了 SQLAlchemy 的示例套件中进行了演示。

在 `SessionEvents.do_orm_execute()` 事件钩子内部时，可以使用 `ORMExecuteState.invoke_statement()` 方法来使用新的嵌套调用 `Session.execute()` 来调用语句，这将预先阻止当前正在进行的执行的后续处理，并返回内部执行的 `Result`。在此过程中对 `SessionEvents.do_orm_execute()` 钩子所调用的事件处理程序也将在此嵌套调用中被跳过。

`ORMExecuteState.invoke_statement()`方法返回一个`Result`对象；此对象具有将其“冻结”为可缓存格式并“解冻”为新的`Result`对象的能力，以及将其数据与其他`Result`对象的数据合并的能力。

例如，使用`SessionEvents.do_orm_execute()`实现缓存：

```py
from sqlalchemy.orm import loading

cache = {}

@event.listens_for(Session, "do_orm_execute")
def _do_orm_execute(orm_execute_state):
    if "my_cache_key" in orm_execute_state.execution_options:
        cache_key = orm_execute_state.execution_options["my_cache_key"]

        if cache_key in cache:
            frozen_result = cache[cache_key]
        else:
            frozen_result = orm_execute_state.invoke_statement().freeze()
            cache[cache_key] = frozen_result

        return loading.merge_frozen_result(
            orm_execute_state.session,
            orm_execute_state.statement,
            frozen_result,
            load=False,
        )
```

有了上述钩子，使用缓存的示例如下所示：

```py
stmt = (
    select(User).where(User.name == "sandy").execution_options(my_cache_key="key_sandy")
)

result = session.execute(stmt)
```

上面，在`Select.execution_options()`中传递了自定义执行选项，以建立一个“缓存键”，然后会被`SessionEvents.do_orm_execute()`钩子拦截。然后将该缓存键与可能存在于缓存中的`FrozenResult`对象进行匹配，如果存在，则重新使用该对象。该方案利用了`Result.freeze()`方法来“冻结”一个`Result`对象，其中包含上面的 ORM 结果，以便将其存储在缓存中并多次使用。为了从“冻结”的结果中返回实时结果，使用了`merge_frozen_result()`函数将结果对象中的“冻结”数据合并到当前会话中。

上述示例已作为一个完整示例在 Dogpile Caching 中实现。

`ORMExecuteState.invoke_statement()` 方法也可能被多次调用，传递不同的信息给 `ORMExecuteState.invoke_statement.bind_arguments` 参数，以便 `Session` 每次使用不同的 `Engine` 对象。这将每次返回一个不同的 `Result` 对象；这些结果可以使用 `Result.merge()` 方法合并在一起。这是水平分片扩展所采用的技术；请参阅源代码以熟悉。

另请参阅

Dogpile 缓存

水平分片

## 持久性事件

最广泛使用的系列事件可能是“持久性”事件，它们对应于刷新过程。 刷新是关于对对象的待定更改的所有决定都是在这里做出的，然后以 INSERT、UPDATE 和 DELETE 语句的形式发出到数据库。

### `before_flush()`

`SessionEvents.before_flush()` 钩子是应用程序希望在提交时确保对数据库进行额外持久性更改时最常用的事件。使用 `SessionEvents.before_flush()` 来在对象上操作以验证它们的状态，以及在它们持久化之前组合额外的对象和引用。在此事件中，**操纵会话状态是安全的**，也就是说，新对象可以附加到它，对象可以被删除，并且可以自由更改对象上的单个属性，并且这些更改将在事件挂钩完成时被纳入到刷新过程中。

典型的`SessionEvents.before_flush()`挂钩将负责扫描集合`Session.new`、`Session.dirty`和`Session.deleted`，以查找将要发生变化的对象。

对于`SessionEvents.before_flush()`的示例，请参阅使用历史表进行版本控制和使用时间行进行版本控制等示例。

### `after_flush()`

在 SQL 被发出进行刷新过程后，但是在被刷新的对象的状态被改变之前，会调用`SessionEvents.after_flush()`挂钩。也就是说，你仍然可以检查`Session.new`、`Session.dirty`和`Session.deleted`这些集合，看看刚刚刷新了什么，你也可以使用像`AttributeState`提供的历史跟踪功能来查看刚刚持久化了什么更改。在`SessionEvents.after_flush()`事件中，可以根据观察到的变化向数据库发出其他 SQL。

### `after_flush_postexec()`

`SessionEvents.after_flush_postexec()` 在 `SessionEvents.after_flush()` 之后不久被调用，但在考虑了刚刚发生的刷新后对象状态被修改的情况下调用。`Session.new`、`Session.dirty` 和 `Session.deleted` 集合通常在这里是完全空的。使用 `SessionEvents.after_flush_postexec()` 来检查已完成对象的标识映射并可能发出额外的 SQL。在这个钩子中，有能力对对象进行新的更改，这意味着 `Session` 将再次进入“dirty”状态；如果在此钩子中检测到新的更改，则会导致 `Session` 的机制再次刷新**一次**，如果在 `Session.commit()` 的上下文中调用刷新并检测到新的更改，则否则，挂起的更改将作为下一个正常刷新的一部分捆绑在一起。当钩子在 `Session.commit()` 中检测到新的更改时，计数器确保在每次调用时都添加新的状态时不会无限循环，以防止无休止的循环在这方面在经过 100 次迭代后停止。

### Mapper 级别的刷新事件

除了刷新级别的钩子外，还有一组更细粒度的钩子，它们是基于每个对象的并且根据刷新过程中的 INSERT、UPDATE 或 DELETE 而分开的。这些是映射器持久性钩子，它们也非常受欢迎，但是需要更加谨慎地对待这些事件，因为它们在已经进行中的刷新过程的上下文中进行；在这里进行许多操作是不安全的。

事件包括：

+   `MapperEvents.before_insert()`

+   `MapperEvents.after_insert()`

+   `MapperEvents.before_update()`

+   `MapperEvents.after_update()`

+   `MapperEvents.before_delete()`

+   `MapperEvents.after_delete()`

注意

重要的是要注意，这些事件**仅适用于**会话刷新操作，而**不适用于**在 ORM-Enabled INSERT、UPDATE 和 DELETE 语句中描述的 ORM 级别的 INSERT/UPDATE/DELETE 功能。要拦截 ORM 级别的 DML，请使用`SessionEvents.do_orm_execute()`事件。

每个事件都会传递`Mapper`、映射对象本身以及正在用于发出 INSERT、UPDATE 或 DELETE 语句的`Connection`。这些事件的吸引力是显而易见的，因为如果一个应用程序想要将某些活动与特定类型的对象被插入时联系起来，那么钩子就非常具体；不像`SessionEvents.before_flush()`事件那样，不需要搜索像`Session.new`这样的集合以找到目标。然而，在调用这些事件时，表示将要发出的每个单独的 INSERT、UPDATE、DELETE 语句的 flush 计划已经*已经决定*，并且在此阶段不能做出任何更改。因此，对给定对象的唯一可能的更改是对对象行**本地**的属性进行。对对象或其他对象的任何其他更改将影响`Session`的状态，这将导致其无法正常工作。

不支持这些映射器级别持久性事件中的操作包括：

+   `Session.add()`

+   `Session.delete()`

+   映射集合的附加、添加、删除、丢弃等操作。

+   映射关系属性设置/删除事件，即`someobject.related = someotherobject`

传递`Connection`的原因是鼓励**在此处执行简单的 SQL 操作**，直接在`Connection`上，例如增加计数器或在日志表中插入额外行。  

也有许多不需要在刷新事件中处理的每个对象操作。最常见的替代方法是简单地在对象的`__init__()`方法中与对象一起建立附加状态，例如创建与新对象关联的其他对象。使用如简单验证器中所述的验证器是另一种方法；这些函数可以拦截对属性的更改，并在响应属性更改时在目标对象上建立额外的状态更改。使用这两种方法，对象在到达刷新步骤之前处于正确的状态。  

### `before_flush()`  

`SessionEvents.before_flush()` 钩子是应用程序希望确保在提交刷新时进行额外持久化更改时最常用的事件。使用`SessionEvents.before_flush()` 来操作对象以验证其状态，并在持久化之前组成额外的对象和引用。在此事件中，**操作会话的状态是安全的**，即，新对象可以附加到其中，对象可以被删除，并且可以自由更改对象上的单个属性，并且这些更改将在事件钩子完成时被引入到刷新过程中。  

典型的`SessionEvents.before_flush()` 钩子将负责扫描集合`Session.new`、`Session.dirty` 和`Session.deleted`，以查找将发生事情的对象。  

有关`SessionEvents.before_flush()`的示例，请参见带有历史表的版本控制和使用时间行进行版本控制等示例。  

### `after_flush()`  

`SessionEvents.after_flush()` 钩子在 flush 过程中的 SQL 已经被发出，但是在被刷新的对象状态被改变之前被调用。也就是说，您仍然可以检查 `Session.new`、`Session.dirty` 和 `Session.deleted` 集合，看看刚刚被刷新了什么，您还可以使用像 `AttributeState` 提供的历史跟踪功能来查看刚刚持久化的更改。在 `SessionEvents.after_flush()` 事件中，根据观察到的更改，可以向数据库发出其他 SQL。

### `after_flush_postexec()`

`SessionEvents.after_flush_postexec()` 在 `SessionEvents.after_flush()` 之后不久调用，但是在对象的状态已经修改以反映刚刚发生的刷新之后调用。这里 `Session.new`、`Session.dirty` 和 `Session.deleted` 集合通常完全为空。使用 `SessionEvents.after_flush_postexec()` 来检查最终化对象的标识映射，并可能发出其他 SQL。在这个钩子中，有能力对对象进行新的更改，这意味着 `Session` 将再次进入“dirty”状态；如果在 `Session.commit()` 的上下文中调用此钩子时检测到新的更改，那么`Session` 的机制会导致它再次刷新；否则，待定的更改将作为下一个正常刷新的一部分捆绑在一起。当钩子在 `Session.commit()` 中检测到新的更改时，计数器会在此方面的无限循环在 100 次迭代后停止，在这种情况下，如果 `SessionEvents.after_flush_postexec()` 钩子每次调用时都持续添加新的要刷新的状态，就会停止。

### 映射器级别的刷新事件

除了刷新级别的钩子外，还有一套更精细的钩子，因为它们是基于每个对象调用的，并根据刷新过程中的 INSERT、UPDATE 或 DELETE 进行分组。这些是映射器持久性钩子，它们也非常受欢迎，但是这些事件需要更谨慎地处理，因为它们在已经进行的刷新过程的上下文中进行；许多操作在这里进行不安全。

事件包括：

+   `MapperEvents.before_insert()`

+   `MapperEvents.after_insert()`

+   `MapperEvents.before_update()`

+   `MapperEvents.after_update()`

+   `MapperEvents.before_delete()`

+   `MapperEvents.after_delete()`

注意

重要的是要注意，这些事件**仅**适用于会话刷新操作，**而不是**描述的 ORM 级别的 INSERT/UPDATE/DELETE 功能 ORM-Enabled INSERT, UPDATE, and DELETE statements。要拦截 ORM 级别的 DML，请使用`SessionEvents.do_orm_execute()`事件。

每个事件都会传递`Mapper`、映射对象本身和用于发出 INSERT、UPDATE 或 DELETE 语句的`Connection`。这些事件的吸引力是显而易见的，因为如果应用程序想要将某些活动与在 INSERT 时持久化特定类型的对象绑定起来，这个钩子就非常具体；与`SessionEvents.before_flush()`事件不同，不需要搜索像`Session.new`这样的集合来找到目标。然而，在调用这些事件时，表示要发出的每个单独的 INSERT、UPDATE、DELETE 语句的刷新计划已经**已经确定**，在此阶段无法进行任何更改。因此，甚至可能对给定对象进行的唯一更改是对对象行的**本地**属性。对于对象或其他对象的任何其他更改都将影响到`Session`的状态，这将导致其无法正常工作。

在这些映射器级别的持久性事件中不支持的操作包括：

+   `Session.add()`

+   `Session.delete()`

+   映射的集合追加、添加、移除、删除、丢弃等操作。

+   映射的关系属性设置/删除事件，即`someobject.related = someotherobject`

传递`Connection`的原因是鼓励**在这里进行简单的 SQL 操作**，直接在`Connection`上进行，比如增加计数器或在日志表中插入额外行。

还有许多不需要在刷新事件中处理的每个对象操作。最常见的替代方法是在对象的`__init__()`方法中简单地建立额外状态，比如创建要与新对象关联的其他对象。另一种方法是使用简单验证器中描述的验证器；这些函数可以拦截属性的更改，并在响应属性更改时在目标对象上建立额外的状态更改。使用这两种方法，对象在进入刷新步骤之前就处于正确状态。

## 对象生命周期事件

事件的另一个用例是跟踪对象的生命周期。这指的是首次介绍的状态，即快速介绍对象状态。

所有上述状态都可以通过事件完全跟踪。每个事件代表一个不同的状态转换，意味着起始状态和目标状态都是被跟踪的一部分。除了初始瞬态事件之外，所有事件都是以`Session`对象或类的形式，意味着它们可以与特定的`Session`对象关联：

```py
from sqlalchemy import event
from sqlalchemy.orm import Session

session = Session()

@event.listens_for(session, "transient_to_pending")
def object_is_pending(session, obj):
    print("new pending: %s" % obj)
```

或者与`Session`类本身一起，以及与特定的`sessionmaker`一起，这可能是最有用的形式：

```py
from sqlalchemy import event
from sqlalchemy.orm import sessionmaker

maker = sessionmaker()

@event.listens_for(maker, "transient_to_pending")
def object_is_pending(session, obj):
    print("new pending: %s" % obj)
```

当然，监听器可以堆叠在一个函数之上，这很常见。例如，要跟踪进入持久状态的所有对象：

```py
@event.listens_for(maker, "pending_to_persistent")
@event.listens_for(maker, "deleted_to_persistent")
@event.listens_for(maker, "detached_to_persistent")
@event.listens_for(maker, "loaded_as_persistent")
def detect_all_persistent(session, instance):
    print("object is now persistent: %s" % instance)
```

### 瞬态

当所有映射对象首次构建时，它们都是瞬态的。在此状态下，对象单独存在，并且不与任何`Session`相关联。对于这种初始状态，没有特定的“转换”事件，因为没有`Session`，但是如果想要拦截任何瞬态对象被创建时，`InstanceEvents.init()` 方法可能是最好的事件。此事件适用于特定的类或超类。例如，要拦截特定声明基类的所有新对象：

```py
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy import event

class Base(DeclarativeBase):
    pass

@event.listens_for(Base, "init", propagate=True)
def intercept_init(instance, args, kwargs):
    print("new transient: %s" % instance)
```

### 瞬态到挂起

当瞬态对象首次通过`Session.add()`或`Session.add_all()`方法与`Session`关联时，瞬态对象变为挂起。对象也可能作为显式添加的引用对象的“级联”的结果之一而成为`Session`的一部分。可以使用`SessionEvents.transient_to_pending()`事件检测瞬态到挂起的转换：

```py
@event.listens_for(sessionmaker, "transient_to_pending")
def intercept_transient_to_pending(session, object_):
    print("transient to pending: %s" % object_)
```

### 挂起到持久

当刷新继续并且针对实例进行 INSERT 语句时，挂起对象变为持久。对象现在具有标识键。使用`SessionEvents.pending_to_persistent()`事件跟踪挂起到持久的转换：

```py
@event.listens_for(sessionmaker, "pending_to_persistent")
def intercept_pending_to_persistent(session, object_):
    print("pending to persistent: %s" % object_)
```

### 挂起到瞬态

如果在挂起对象刷新之前调用`Session.rollback()`方法，或者在刷新对象之前调用`Session.expunge()`方法，则挂起对象可以恢复为瞬态。使用`SessionEvents.pending_to_transient()`事件跟踪挂起到瞬态的转换：

```py
@event.listens_for(sessionmaker, "pending_to_transient")
def intercept_pending_to_transient(session, object_):
    print("transient to pending: %s" % object_)
```

### 加载为持久对象

当从数据库加载对象时，对象可以直接进入 `Session` 中的 persistent 状态。跟踪这种状态转换与跟踪对象的加载是同义的，并且等同于使用 `InstanceEvents.load()` 实例级事件。然而，`SessionEvents.loaded_as_persistent()` 事件作为一个针对会话的钩子，用于拦截对象通过这种特定途径进入持久状态：

```py
@event.listens_for(sessionmaker, "loaded_as_persistent")
def intercept_loaded_as_persistent(session, object_):
    print("object loaded into persistent state: %s" % object_)
```

### 持久到瞬态

当对象处于持久状态时，如果调用了 `Session.rollback()` 方法，该对象会恢复到瞬态状态，前提是它最初是作为待定对象添加的事务。在回滚的情况下，使该对象持久的 INSERT 语句将被回滚，并且该对象将从 `Session` 中驱逐出去，再次变为瞬态。使用 `SessionEvents.persistent_to_transient()` 事件钩子来跟踪从持久状态恢复到瞬态状态的对象：

```py
@event.listens_for(sessionmaker, "persistent_to_transient")
def intercept_persistent_to_transient(session, object_):
    print("persistent to transient: %s" % object_)
```

### 持久到已删除

当标记为删除的对象在刷新过程中从数据库中删除时，持久对象进入了 deleted 状态。请注意，这与调用 `Session.delete()` 方法删除目标对象时**不同**。`Session.delete()` 方法只是**标记**对象为删除；直到刷新进行时，实际的 DELETE 语句才会被发出。在刷新之后，目标对象将处于“deleted”状态。

在“deleted”状态下，对象与 `Session` 仅有轻微关联。它不在标识映射中，也不在指示它何时待删除的 `Session.deleted` 集合中。

从“deleted”状态，对象可以在事务提交时进入分离状态，或者如果事务回滚，则返回持久状态。

使用 `SessionEvents.persistent_to_deleted()` 跟踪从持久到已删除的转换：

```py
@event.listens_for(sessionmaker, "persistent_to_deleted")
def intercept_persistent_to_deleted(session, object_):
    print("object was DELETEd, is now in deleted state: %s" % object_)
```

### 删除到分离

当会话的事务提交后，删除的对象会变成分离。在调用`Session.commit()`方法后，数据库事务已经完成，`Session`现在完全丢弃了删除的对象并删除了所有与之相关的关联。使用`SessionEvents.deleted_to_detached()`跟踪删除到分离的转换：

```py
@event.listens_for(sessionmaker, "deleted_to_detached")
def intercept_deleted_to_detached(session, object_):
    print("deleted to detached: %s" % object_)
```

注意

当对象处于删除状态时，可通过`inspect(object).deleted`访问`InstanceState.deleted`属性，返回 True。然而，当对象被分离时，`InstanceState.deleted`将再次返回 False。要检测对象是否已被删除，无论它是否分离，都可以使用`InstanceState.was_deleted`访问器。

### 持久到分离

当对象使用`Session.expunge()`、`Session.expunge_all()`或`Session.close()`方法与`Session`解除关联时，持久对象将变为分离。

注意

如果应用程序解除引用并由于垃圾回收而丢弃了拥有的`Session`，则对象也可能**隐式分离**。在这种情况下，**不会触发任何事件**。

使用`SessionEvents.persistent_to_detached()`事件跟踪对象从持久到分离的移动：

```py
@event.listens_for(sessionmaker, "persistent_to_detached")
def intercept_persistent_to_detached(session, object_):
    print("object became detached: %s" % object_)
```

### 分离到持久

当分离的对象使用`Session.add()`或等效方法重新关联到会话时，它将变成持久对象。使用`SessionEvents.detached_to_persistent()`事件跟踪从分离返回到持久的对象：

```py
@event.listens_for(sessionmaker, "detached_to_persistent")
def intercept_detached_to_persistent(session, object_):
    print("object became persistent again: %s" % object_)
```

### 删除到持久

当用于 DELETE 的事务使用`Session.rollback()`方法回滚时，已删除对象可以恢复为持久状态。使用`SessionEvents.deleted_to_persistent()`事件跟踪已删除对象移回持久状态：

```py
@event.listens_for(sessionmaker, "deleted_to_persistent")
def intercept_deleted_to_persistent(session, object_):
    print("deleted to persistent: %s" % object_)
```

### 暂时

所有映射对象在首次构造时都作为暂时状态开始。在这种状态下，对象单独存在，并且不与任何`Session`相关联。对于这种初始状态，没有特定的“转换”事件，因为没有`Session`，但是如果想要拦截任何暂时对象被创建的情况，`InstanceEvents.init()`方法可能是最好的事件。此事件适用于特定类或超类。例如，要拦截特定声明基类的所有新对象：

```py
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy import event

class Base(DeclarativeBase):
    pass

@event.listens_for(Base, "init", propagate=True)
def intercept_init(instance, args, kwargs):
    print("new transient: %s" % instance)
```

### 暂时到待定

当暂时对象首次通过`Session.add()`或`Session.add_all()`方法与`Session`相关联时，暂时对象变为待定状态。对象也可能作为引用对象的“级联”结果成为`Session`的一部分，该引用对象已明确添加。可以使用`SessionEvents.transient_to_pending()`事件检测从暂时到待定的转换：

```py
@event.listens_for(sessionmaker, "transient_to_pending")
def intercept_transient_to_pending(session, object_):
    print("transient to pending: %s" % object_)
```

### 待定到持久

待定对象在执行冲洗并为该实例执行插入语句时变为持久状态。对象现在具有标识键。使用`SessionEvents.pending_to_persistent()`事件跟踪待定对象到持久状态的转换：

```py
@event.listens_for(sessionmaker, "pending_to_persistent")
def intercept_pending_to_persistent(session, object_):
    print("pending to persistent: %s" % object_)
```

### 待定到暂时

如果在待定对象被刷新之前调用了`Session.rollback()`方法，或者在对象被刷新之前调用了`Session.expunge()`方法，则待定对象可以恢复到瞬态状态。使用`SessionEvents.pending_to_transient()`事件跟踪从待定到瞬态的对象：

```py
@event.listens_for(sessionmaker, "pending_to_transient")
def intercept_pending_to_transient(session, object_):
    print("transient to pending: %s" % object_)
```

### 加载为持久化

对象可以直接以持久化状态出现在`Session`中，当它们从数据库中加载时。跟踪这种状态转换与跟踪对象的加载是同义的，并且与使用`InstanceEvents.load()`实例级事件是同义的。但是，提供了`SessionEvents.loaded_as_persistent()`事件作为一个会话中心的钩子，用于拦截通过这种特定途径进入持久化状态的对象：

```py
@event.listens_for(sessionmaker, "loaded_as_persistent")
def intercept_loaded_as_persistent(session, object_):
    print("object loaded into persistent state: %s" % object_)
```

### 持久化到瞬态

如果在对象首次被添加为待定的事务中调用了`Session.rollback()`方法，则持久化对象可以恢复到瞬态状态。在回滚的情况下，使该对象持久化的 INSERT 语句被回滚，对象被从`Session`中驱逐，再次成为瞬态。使用`SessionEvents.persistent_to_transient()`事件钩子跟踪从持久化到瞬态的对象：

```py
@event.listens_for(sessionmaker, "persistent_to_transient")
def intercept_persistent_to_transient(session, object_):
    print("persistent to transient: %s" % object_)
```

### 持久化到已删除

在刷新过程中，如果标记为删除的对象从数据库中被删除，则持久化对象进入已删除状态。请注意，这**与**调用`Session.delete()`方法删除目标对象时不同。`Session.delete()`方法仅**标记**对象以进行删除；直到刷新进行之后，实际的 DELETE 语句才会被发出。在刷新之后，目标对象的“已删除”状态才存在。

在“删除”状态中，对象与`Session`仅具有较弱的关联。它不在标识映射中，也不在指向它曾经等待删除的`Session.deleted`集合中。

从“删除”状态，对象可以通过提交事务进入分离状态，或者如果事务被回滚，则返回持久状态。

使用`SessionEvents.persistent_to_deleted()`来追踪持久到删除的转变：

```py
@event.listens_for(sessionmaker, "persistent_to_deleted")
def intercept_persistent_to_deleted(session, object_):
    print("object was DELETEd, is now in deleted state: %s" % object_)
```

### 删除到分离

当会话的事务提交后，被删除的对象会变成分离状态。在调用`Session.commit()`方法后，数据库事务已经最终化，`Session`现在完全丢弃了被删除的对象并移除了所有与其相关的关联。使用`SessionEvents.deleted_to_detached()`来追踪从删除到分离的转变：

```py
@event.listens_for(sessionmaker, "deleted_to_detached")
def intercept_deleted_to_detached(session, object_):
    print("deleted to detached: %s" % object_)
```

注意

当对象处于删除状态时，可以通过`inspect(object).deleted`访问的`InstanceState.deleted`属性返回 True。但是当对象分离时，`InstanceState.deleted`将再次返回 False。要检测对象是否已被删除，无论它是否分离，请使用`InstanceState.was_deleted`访问器。

### 持久到分离

当对象与`Session`取消关联时，通过`Session.expunge()`、`Session.expunge_all()`或`Session.close()`方法，持久对象会变成分离状态。

注意

如果由于垃圾回收应用程序取消引用并丢弃了拥有对象的`Session`，则对象也可能**隐式分离**。在这种情况下，**不会发出任何事件**。

使用`SessionEvents.persistent_to_detached()`事件跟踪对象从持久化到分离的移动：

```py
@event.listens_for(sessionmaker, "persistent_to_detached")
def intercept_persistent_to_detached(session, object_):
    print("object became detached: %s" % object_)
```

### 分离到持久化

当将分离的对象重新关联到会话时，该分离对象变为持久化状态，可使用`Session.add()`或等效方法。跟踪对象从分离状态回到持久化状态时，请使用`SessionEvents.detached_to_persistent()`事件：

```py
@event.listens_for(sessionmaker, "detached_to_persistent")
def intercept_detached_to_persistent(session, object_):
    print("object became persistent again: %s" % object_)
```

### 删除到持久化

当以 DELETE 方式删除对象的事务被使用`Session.rollback()`方法回滚时，删除对象可以恢复到持久化状态。跟踪已删除对象从删除状态返回持久化状态，请使用`SessionEvents.deleted_to_persistent()`事件：

```py
@event.listens_for(sessionmaker, "deleted_to_persistent")
def intercept_deleted_to_persistent(session, object_):
    print("deleted to persistent: %s" % object_)
```

## 事务事件

事务事件允许应用程序在`Session`级别发生事务边界以及`Session`在`Connection`对象上更改事务状态时得到通知。

+   `SessionEvents.after_transaction_create()`, `SessionEvents.after_transaction_end()` - 这些事件跟踪`Session`的逻辑事务范围，与个别数据库连接无关。这些事件旨在帮助集成诸如`zope.sqlalchemy`之类的事务跟踪系统。当应用程序需要将某些外部范围与`Session`的事务范围对齐时，请使用这些事件。这些钩子反映了`Session`的“嵌套”事务行为，因为它们不仅跟踪逻辑“子事务”，还跟踪“嵌套”（例如 SAVEPOINT）事务。

+   `SessionEvents.before_commit()`, `SessionEvents.after_commit()`, `SessionEvents.after_begin()`, `SessionEvents.after_rollback()`, `SessionEvents.after_soft_rollback()` - 这些事件允许从数据库连接的角度跟踪事务事件。特别是`SessionEvents.after_begin()` 是一个每个连接的事件；一个维护多个连接的 `Session` 将在当前事务中使用这些连接时为每个连接单独发出此事件。然后回滚和提交事件是指 DBAPI 连接本身何时直接收到回滚或提交指令。

## 属性更改事件

属性更改事件允许拦截对象上特定属性被修改的情况。这些事件包括`AttributeEvents.set()`, `AttributeEvents.append()`, 和 `AttributeEvents.remove()`。这些事件非常有用，特别是用于每个对象的验证操作；然而，通常更方便使用“验证器”钩子，该钩子在幕后使用这些钩子；请参阅简单验证器 了解背景信息。属性事件也在反向引用的机制后面。一个示例说明了属性事件的使用在属性检测。
