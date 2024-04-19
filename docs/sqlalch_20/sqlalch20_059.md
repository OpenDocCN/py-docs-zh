# ORM 事件

> 原文：[`docs.sqlalchemy.org/en/20/orm/events.html`](https://docs.sqlalchemy.org/en/20/orm/events.html)

ORM 包括各种可供订阅的钩子。

要了解最常用的 ORM 事件的介绍，请参阅使用事件跟踪查询、对象和会话更改部分。一般讨论事件系统，请参阅事件。关于连接和低级语句执行等非 ORM 事件的描述，请参阅核心事件。

## 会话事件

最基本的事件钩子可在 ORM `Session`对象级别使用。这里拦截的内容包括：

+   **持久化操作** - 将更改发送到数据库的 ORM 刷新过程可以使用在刷新的不同部分触发的事件进行扩展，以增强或修改发送到数据库的数据，或者在持久化发生时允许其他事情发生。在持久化事件中了解更多信息。

+   **对象生命周期事件** - 当对象被添加、持久化、从会话中删除时触发的钩子。在对象生命周期事件中了解更多信息。

+   **执行事件** - 2.0 风格执行模型的一部分，拦截所有针对 ORM 实体的 SELECT 语句，以及在刷新过程之外的批量 UPDATE 和 DELETE 语句，使用`Session.execute()`方法通过`SessionEvents.do_orm_execute()`方法。在执行事件中了解更多关于此事件的信息。

请务必阅读使用事件跟踪查询、对象和会话更改章节，以了解这些事件的背景。

| 对象名称 | 描述 |
| --- | --- |
| SessionEvents | 定义特定于`Session`生命周期的事件。 |

```py
class sqlalchemy.orm.SessionEvents
```

定义特定于`Session`生命周期的事件。

例如：

```py
from sqlalchemy import event
from sqlalchemy.orm import sessionmaker

def my_before_commit(session):
    print("before commit!")

Session = sessionmaker()

event.listen(Session, "before_commit", my_before_commit)
```

`listen()`函数将接受`Session`对象，以及`sessionmaker()`和`scoped_session()`的返回结果。

此外，它接受`Session`类，该类将全局应用监听器到所有`Session`实例。

参数：

+   `raw=False` –

    当为 True 时，传递给适用于单个对象的事件侦听器函数的“target”参数将是实例的`InstanceState`管理对象，而不是映射的实例本身。

    版本 1.3.14 中的新功能。

+   `restore_load_context=False` –

    适用于`SessionEvents.loaded_as_persistent()`事件。在事件钩子完成时恢复对象的加载器上下文，以便正在进行的急切加载操作继续正确地针对对象。如果在此事件中将对象移动到新的加载器上下文而未设置此标志，则会发出警告。

    版本 1.3.14 中的新功能。

**成员**

after_attach(), after_begin(), after_bulk_delete(), after_bulk_update(), after_commit(), after_flush(), after_flush_postexec(), after_rollback(), after_soft_rollback(), after_transaction_create(), after_transaction_end(), before_attach(), before_commit(), before_flush(), deleted_to_detached(), deleted_to_persistent(), detached_to_persistent(), dispatch, do_orm_execute(), loaded_as_persistent(), pending_to_persistent(), pending_to_transient(), persistent_to_deleted(), persistent_to_detached(), persistent_to_transient(), transient_to_pending()

**类签名**

类`sqlalchemy.orm.SessionEvents`（`sqlalchemy.event.Events`）

```py
method after_attach(session: Session, instance: _O) → None
```

在实例附加到会话后执行。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeSessionClassOrObject, 'after_attach')
def receive_after_attach(session, instance):
    "listen for the 'after_attach' event"

    # ... (event handling logic) ...
```

这在添加、删除或合并后调用。

注意

从 0.8 开始，此事件在项目完全与会话关联之后触发，这与以前的版本不同。对于需要对象尚未成为会话状态的事件处理程序（例如，在目标对象尚未完成时可能自动刷新的处理程序），请考虑新的`before_attach()`事件。

另请参阅

`SessionEvents.before_attach()`

对象生命周期事件

```py
method after_begin(session: Session, transaction: SessionTransaction, connection: Connection) → None
```

在连接上开始事务后执行。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeSessionClassOrObject, 'after_begin')
def receive_after_begin(session, transaction, connection):
    "listen for the 'after_begin' event"

    # ... (event handling logic) ...
```

注意

此事件在`Session`修改其自身内部状态的过程中调用。要在此挂钩内调用 SQL 操作，请使用事件提供的`Connection`；请勿直接使用`Session`运行 SQL 操作。

参数：

+   `session` – 目标`Session`。

+   `transaction` – `SessionTransaction`。

+   `connection` – 将用于 SQL 语句的`Connection`对象。

另请参阅

`SessionEvents.before_commit()`

`SessionEvents.after_commit()`

`SessionEvents.after_transaction_create()`

`SessionEvents.after_transaction_end()`

```py
method after_bulk_delete(delete_context: _O) → None
```

在调用传统的`Query.delete()`方法之后的事件。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeSessionClassOrObject, 'after_bulk_delete')
def receive_after_bulk_delete(delete_context):
    "listen for the 'after_bulk_delete' event"

    # ... (event handling logic) ...

# DEPRECATED calling style (pre-0.9, will be removed in a future release)
@event.listens_for(SomeSessionClassOrObject, 'after_bulk_delete')
def receive_after_bulk_delete(session, query, query_context, result):
    "listen for the 'after_bulk_delete' event"

    # ... (event handling logic) ...
```

从版本 0.9 开始更改：`SessionEvents.after_bulk_delete()`事件现在接受参数`SessionEvents.after_bulk_delete.delete_context`。将来的版本中将删除接受上述“已弃用”参数签名的监听器函数的支持。

遗留特性

`SessionEvents.after_bulk_delete()`方法是 SQLAlchemy 2.0 的传统事件钩子。该事件**不参与**使用`delete()`在 ORM UPDATE and DELETE with Custom WHERE Criteria 中记录的 2.0 风格调用。对于 2.0 风格的使用，`SessionEvents.do_orm_execute()`钩子将拦截这些调用。

参数：

**delete_context** -

一个包含有关更新的“删除上下文”对象，包括这些属性：

> +   `session` - 涉及的`Session`。
> +   
> +   `query` - 调用此更新操作的`Query`对象。
> +   
> +   `result` 作为批量 DELETE 操作的结果返回的`CursorResult`。

从版本 1.4 开始更改：update_context 不再与`QueryContext`对象相关联。

另请参阅

`QueryEvents.before_compile_delete()`

`SessionEvents.after_bulk_update()`

```py
method after_bulk_update(update_context: _O) → None
```

在调用传统`Query.update()`方法之后的事件。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeSessionClassOrObject, 'after_bulk_update')
def receive_after_bulk_update(update_context):
    "listen for the 'after_bulk_update' event"

    # ... (event handling logic) ...

# DEPRECATED calling style (pre-0.9, will be removed in a future release)
@event.listens_for(SomeSessionClassOrObject, 'after_bulk_update')
def receive_after_bulk_update(session, query, query_context, result):
    "listen for the 'after_bulk_update' event"

    # ... (event handling logic) ...
```

从版本 0.9 开始更改：`SessionEvents.after_bulk_update()`事件现在接受参数`SessionEvents.after_bulk_update.update_context`。将来的版本中将删除接受上述“已弃用”参数签名的监听器函数的支持。

遗留特性

`SessionEvents.after_bulk_update()`方法是 SQLAlchemy 2.0 版本之后的传统事件钩子。该事件**不参与**使用`update()`进行的 2.0 风格调用，该调用在 ORM UPDATE and DELETE with Custom WHERE Criteria 中有记录。对于 2.0 风格的使用，`SessionEvents.do_orm_execute()`钩子将拦截这些调用。

参数：

**update_context** –

包含有关更新的“更新上下文”对象，包括这些属性：

> +   `session` - 涉及的`Session`
> +   
> +   `query` -调用此更新操作的`Query`对象。
> +   
> +   `values` 传递给`Query.update()`的“值”字典。
> +   
> +   `result` 作为批量更新操作的结果返回的`CursorResult`。

从 1.4 版本开始更改：update_context 不再与`QueryContext`对象关联。

另请参阅

`QueryEvents.before_compile_update()`

`SessionEvents.after_bulk_delete()`

```py
method after_commit(session: Session) → None
```

在提交后执行。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeSessionClassOrObject, 'after_commit')
def receive_after_commit(session):
    "listen for the 'after_commit' event"

    # ... (event handling logic) ...
```

注意

`SessionEvents.after_commit()`钩子不是每次刷新一次，也就是说，`Session`可以在事务范围内多次向数据库发出 SQL。要拦截这些事件，请使用`SessionEvents.before_flush()`、`SessionEvents.after_flush()`或`SessionEvents.after_flush_postexec()`事件。

注意

当调用`SessionEvents.after_commit()`事件时，`Session`处于非活动事务状态，因此无法发出 SQL。若要发出与每个事务对应的 SQL，请使用`SessionEvents.before_commit()`事件。

参数：

**session** – 目标`Session`。

另请参阅

`SessionEvents.before_commit()`

`SessionEvents.after_begin()`

`SessionEvents.after_transaction_create()`

`SessionEvents.after_transaction_end()`

```py
method after_flush(session: Session, flush_context: UOWTransaction) → None
```

在刷新完成后执行，但在调用提交之前执行。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeSessionClassOrObject, 'after_flush')
def receive_after_flush(session, flush_context):
    "listen for the 'after_flush' event"

    # ... (event handling logic) ...
```

请注意，会话的状态仍处于预刷新状态，即‘new’、‘dirty’和‘deleted’列表仍然显示预刷新状态，以及实例属性的历史设置。

警告

此事件在`Session`发出 SQL 以修改数据库后，但在修改其内部状态以反映这些更改之前运行，包括将新插入的对象放入标识映射中。在此事件中发出的 ORM 操作（如加载相关项目）可能会产生新的标识映射条目，这些条目将立即被替换，有时会导致令人困惑的结果。从版本 1.3.9 开始，SQLAlchemy 将为此条件发出警告。

参数：

+   `session` – 目标`Session`。

+   `flush_context` – 处理刷新详细信息的内部`UOWTransaction`对象。

另请参阅

`SessionEvents.before_flush()`

`SessionEvents.after_flush_postexec()`

持久性事件

```py
method after_flush_postexec(session: Session, flush_context: UOWTransaction) → None
```

在刷新完成后执行，并在后执行状态发生后执行。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeSessionClassOrObject, 'after_flush_postexec')
def receive_after_flush_postexec(session, flush_context):
    "listen for the 'after_flush_postexec' event"

    # ... (event handling logic) ...
```

这将是‘new’、‘dirty’和‘deleted’列表处于最终状态的时刻。实际是否提交()取决于刷新是否启动了自己的事务或参与了较大事务。

参数：

+   `session` – 目标 `Session`。

+   `flush_context` – 处理刷新细节的内部`UOWTransaction` 对象。

另请参阅

`SessionEvents.before_flush()`

`SessionEvents.after_flush()`

持久性事件

```py
method after_rollback(session: Session) → None
```

在真正的 DBAPI 回滚发生后执行。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeSessionClassOrObject, 'after_rollback')
def receive_after_rollback(session):
    "listen for the 'after_rollback' event"

    # ... (event handling logic) ...
```

请注意，此事件仅在实际数据库回滚发生时触发 - 当底层 DBAPI 事务已经回滚时，并不会在每次调用 `Session.rollback()` 方法时触发。在许多情况下，在此事件期间，`Session` 不会处于“活动”状态，因为当前事务无效。要获取在最外层回滚已经执行后处于活动状态的 `Session`，请使用 `SessionEvents.after_soft_rollback()` 事件，并检查 `Session.is_active` 标志。

参数：

**session** – 目标 `Session`。

```py
method after_soft_rollback(session: Session, previous_transaction: SessionTransaction) → None
```

在发生任何回滚之后执行，包括“软”回滚，这些回滚实际上不会在 DBAPI 级别发出。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeSessionClassOrObject, 'after_soft_rollback')
def receive_after_soft_rollback(session, previous_transaction):
    "listen for the 'after_soft_rollback' event"

    # ... (event handling logic) ...
```

这对应于嵌套和外部回滚，即调用 DBAPI 的 rollback() 方法的最内层回滚，以及仅弹出事务堆栈的封闭回滚调用。

给定的 `Session` 可以在外部回滚后通过首先检查 `Session.is_active` 标志来调用 SQL 和 `Session.query()` 操作。

```py
@event.listens_for(Session, "after_soft_rollback")
def do_something(session, previous_transaction):
    if session.is_active:
        session.execute(text("select * from some_table"))
```

参数：

+   `session` – 目标`Session`。

+   `previous_transaction` – 刚关闭的`SessionTransaction`事务标记对象。给定`Session`的当前`SessionTransaction`可通过`Session.transaction`属性获得。

```py
method after_transaction_create(session: Session, transaction: SessionTransaction) → None
```

在创建新的`SessionTransaction`时执行。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeSessionClassOrObject, 'after_transaction_create')
def receive_after_transaction_create(session, transaction):
    "listen for the 'after_transaction_create' event"

    # ... (event handling logic) ...
```

此事件与`SessionEvents.after_begin()`不同，因为它针对每个`SessionTransaction`整体而发生，而不是在单个数据库连接上开始事务时发生。它还针对嵌套事务和子事务进行调用，并且始终与相应的`SessionEvents.after_transaction_end()`事件匹配（假设`Session`的正常操作）。

参数：

+   `session` – 目标`Session`。

+   `transaction` –

    目标`SessionTransaction`。

    要检测是否为最外层的`SessionTransaction`，而不是“子事务”或 SAVEPOINT，请测试`SessionTransaction.parent`属性是否为`None`：

    ```py
    @event.listens_for(session, "after_transaction_create")
    def after_transaction_create(session, transaction):
        if transaction.parent is None:
            # work with top-level transaction
    ```

    要检测`SessionTransaction`是否为 SAVEPOINT，请使用`SessionTransaction.nested`属性：

    ```py
    @event.listens_for(session, "after_transaction_create")
    def after_transaction_create(session, transaction):
        if transaction.nested:
            # work with SAVEPOINT transaction
    ```

另请参阅

`SessionTransaction`

`SessionEvents.after_transaction_end()`

```py
method after_transaction_end(session: Session, transaction: SessionTransaction) → None
```

当一个`SessionTransaction`的跨度结束时执行。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeSessionClassOrObject, 'after_transaction_end')
def receive_after_transaction_end(session, transaction):
    "listen for the 'after_transaction_end' event"

    # ... (event handling logic) ...
```

此事件与`SessionEvents.after_commit()`不同，因为它对所有正在使用的`SessionTransaction`对象进行对应，包括嵌套事务和子事务，并且始终与相应的`SessionEvents.after_transaction_create()`事件匹配。

参数：

+   `session` – 目标`Session`。

+   `transaction` –

    目标`SessionTransaction`。

    要检测这是否是最外层的`SessionTransaction`，而不是“子事务”或 SAVEPOINT，请测试`SessionTransaction.parent`属性是否为`None`：

    ```py
    @event.listens_for(session, "after_transaction_create")
    def after_transaction_end(session, transaction):
        if transaction.parent is None:
            # work with top-level transaction
    ```

    要检测`SessionTransaction`是否为 SAVEPOINT，请使用`SessionTransaction.nested`属性：

    ```py
    @event.listens_for(session, "after_transaction_create")
    def after_transaction_end(session, transaction):
        if transaction.nested:
            # work with SAVEPOINT transaction
    ```

另请参见

`SessionTransaction`

`SessionEvents.after_transaction_create()`

```py
method before_attach(session: Session, instance: _O) → None
```

在将实例附加到会话之前执行。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeSessionClassOrObject, 'before_attach')
def receive_before_attach(session, instance):
    "listen for the 'before_attach' event"

    # ... (event handling logic) ...
```

这在添加、删除或合并导致对象成为会话的一部分之前被调用。

另请参见

`SessionEvents.after_attach()`

对象生命周期事件

```py
method before_commit(session: Session) → None
```

在提交之前执行。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeSessionClassOrObject, 'before_commit')
def receive_before_commit(session):
    "listen for the 'before_commit' event"

    # ... (event handling logic) ...
```

注意

`SessionEvents.before_commit()`挂钩*不*是每次刷新一次，也就是说，在事务范围内，`Session`可以多次向数据库发出 SQL。要拦截这些事件，请使用`SessionEvents.before_flush()`、`SessionEvents.after_flush()`或`SessionEvents.after_flush_postexec()`事件。

参数：

**session** – 目标`Session`。

另请参阅

`SessionEvents.after_commit()`

`SessionEvents.after_begin()`

`SessionEvents.after_transaction_create()`

`SessionEvents.after_transaction_end()`

```py
method before_flush(session: Session, flush_context: UOWTransaction, instances: Sequence[_O] | None) → None
```

在刷新过程开始之前执行。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeSessionClassOrObject, 'before_flush')
def receive_before_flush(session, flush_context, instances):
    "listen for the 'before_flush' event"

    # ... (event handling logic) ...
```

参数：

+   `session` – 目标`Session`。

+   `flush_context` – 处理刷新细节的内部`UOWTransaction`对象。

+   `instances` – 通常为`None`，这是可以传递给`Session.flush()`方法的对象集合（请注意，此用法已被弃用）。

另请参阅

`SessionEvents.after_flush()`

`SessionEvents.after_flush_postexec()`

持久性事件

```py
method deleted_to_detached(session: Session, instance: _O) → None
```

拦截特定对象的“删除到分离”转换。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeSessionClassOrObject, 'deleted_to_detached')
def receive_deleted_to_detached(session, instance):
    "listen for the 'deleted_to_detached' event"

    # ... (event handling logic) ...
```

当一个被删除的对象从会话中被驱逐时，触发此事件。典型情况是当包含被删除对象的`Session`的事务被提交时；对象从被删除状态转移到分离状态。

当在调用`Session.expunge_all()`或`Session.close()`事件时，以及如果对象通过`Session.expunge()`从其删除状态单独清除时，也会为在刷新时被删除的对象调用。

另请参阅

对象生命周期事件

```py
method deleted_to_persistent(session: Session, instance: _O) → None
```

拦截特定对象的“删除到持久化”转换。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeSessionClassOrObject, 'deleted_to_persistent')
def receive_deleted_to_persistent(session, instance):
    "listen for the 'deleted_to_persistent' event"

    # ... (event handling logic) ...
```

此转换仅在成功刷新后被删除的对象由于调用`Session.rollback()`而��恢复时发生。在任何其他情况下不会调用此事件。

另请参阅

对象生命周期事件

```py
method detached_to_persistent(session: Session, instance: _O) → None
```

拦截特定对象的“分离到持久”转换。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeSessionClassOrObject, 'detached_to_persistent')
def receive_detached_to_persistent(session, instance):
    "listen for the 'detached_to_persistent' event"

    # ... (event handling logic) ...
```

此事件是 `SessionEvents.after_attach()` 事件的一种特化，仅在此特定转换时调用。通常在 `Session.add()` 调用期间以及在对象之前与 `Session` 不相关联的情况下调用 `Session.delete()` 调用（请注意，将对象标记为“已删除”在刷新进行之前仍保持“持久”状态）。

注意

如果对象在调用 `Session.delete()` 的过程中变为持久对象，则在调用此事件时对象尚未标记为已删除。要检测已删除的对象，请在刷新进行后检查发送到 `SessionEvents.persistent_to_detached()` 事件的 `deleted` 标志，或者如果需要在刷新之前拦截已删除的对象，则在 `SessionEvents.before_flush()` 事件中检查 `Session.deleted` 集合。

参数：

+   `session` – 目标 `Session`

+   `instance` – 正在操作的 ORM 映射实例。

另请参阅

对象生命周期事件

```py
attribute dispatch: _Dispatch[_ET] = <sqlalchemy.event.base.SessionEventsDispatch object>
```

参考 _Dispatch 类。

双向与 _Dispatch._events

```py
method do_orm_execute(orm_execute_state: ORMExecuteState) → None
```

拦截代表 ORM `Session` 对象执行的语句执行。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeSessionClassOrObject, 'do_orm_execute')
def receive_do_orm_execute(orm_execute_state):
    "listen for the 'do_orm_execute' event"

    # ... (event handling logic) ...
```

此事件适用于从`Session.execute()`方法以及相关方法如`Session.scalars()`和`Session.scalar()`调用的所有顶级 SQL 语句。从 SQLAlchemy 1.4 开始，通过`Session.execute()`方法以及相关方法`Session.scalars()`、`Session.scalar()`等运行的所有 ORM 查询都将参与此事件。此事件钩子**不**适用于在 ORM 刷新过程内部发出的查询，即在刷新中描述的过程。

注意

`SessionEvents.do_orm_execute()` 事件钩子仅在**ORM 语句执行**时触发，意味着仅对通过`Session.execute()`等方法在`Session`对象上调用的语句触发。它**不**适用于仅由 SQLAlchemy Core 调用的语句，即通过`Connection.execute()`直接调用或由`Engine`对象发起而不涉及任何`Session`的语句。要拦截**所有**SQL 执行，无论是否使用 Core 或 ORM API，请参阅`ConnectionEvents`中的事件钩子，例如`ConnectionEvents.before_execute()`和`ConnectionEvents.before_cursor_execute()`。

此事件钩子也**不**适用于在 ORM 刷新过程内部发出的查询，即在刷新中描述的过程；要拦截刷新过程中的步骤，请参阅 Persistence Events 以及 Mapper-level Flush Events 中描述的事件钩子。

此事件是一个`do_`事件，意味着它具有替换`Session.execute()`方法通常执行的操作的能力。这包括分片和结果缓存方案，这些方案可能希望在多个数据库连接上调用相同的语句，返回从每个连接合并的结果，或者根本不调用该语句，而是从缓存返回数据。

该钩子旨在替换在 SQLAlchemy 1.4 之前可以被子类化的`Query._execute_and_instances`方法的使用。

参数：

**orm_execute_state** – 一个`ORMExecuteState`的实例，其中包含有关当前执行的所有信息，以及用于推导其他常见所需信息的辅助函数。有关详细信息，请参阅该对象。

另请参阅

执行事件 - 如何使用`SessionEvents.do_orm_execute()`的顶级文档

`ORMExecuteState` - 传递给`SessionEvents.do_orm_execute()`事件的对象，其中包含有关要调用的语句的所有信息。它还提供了一个接口来扩展当前语句、选项和参数，以及一个选项，允许在任何时候以编程方式调用该语句。

ORM 查询事件 - 包括使用`SessionEvents.do_orm_execute()`的示例

Dogpile 缓存 - 一个示例，演示如何将 Dogpile 缓存与 ORM `Session`集成，利用`SessionEvents.do_orm_execute()`事件钩子。

水平分片 - 水平分片示例/扩展依赖于`SessionEvents.do_orm_execute()`事件钩子，在多个后端上调用 SQL 语句并返回合并结果。

1.4 版中的新功能。

```py
method loaded_as_persistent(session: Session, instance: _O) → None
```

拦截特定对象的“加载为持久性”转换。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeSessionClassOrObject, 'loaded_as_persistent')
def receive_loaded_as_persistent(session, instance):
    "listen for the 'loaded_as_persistent' event"

    # ... (event handling logic) ...
```

此事件在 ORM 加载过程中调用，与`InstanceEvents.load()`事件非常类似。但是，在这里，事件可链接到`Session`类或实例，而不是到映射器或类层次结构，并且与其他会话生命周期事件平滑集成。在调用此事件时，保证对象存在于会话的标识映射中。

注意

此事件在加载程序过程中调用，此时可能尚未完成渴望的加载器，并且对象的状态可能不完整。此外，在对象上调用行级刷新操作将使对象进入新的加载器上下文，从而干扰现有的加载上下文。请参阅`InstanceEvents.load()`中有关使用`SessionEvents.restore_load_context`参数的说明，该参数的工作方式与`InstanceEvents.restore_load_context`相同，以解决此场景。

参数：

+   `session` – 目标`Session`

+   `instance` – 正在操作的 ORM 映射实例。

另请参阅

对象生命周期事件

```py
method pending_to_persistent(session: Session, instance: _O) → None
```

拦截特定对象的“待定到持久”的转换。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeSessionClassOrObject, 'pending_to_persistent')
def receive_pending_to_persistent(session, instance):
    "listen for the 'pending_to_persistent' event"

    # ... (event handling logic) ...
```

此事件在刷新过程中调用，类似于扫描`Session.new`集合在`SessionEvents.after_flush()`事件中。但是，在这种情况下，当调用事件时，对象已经被移动到持久状态。

参数：

+   `session` – 目标`Session`

+   `instance` – 正在操作的 ORM 映射实例。

另请参阅

对象生命周期事件

```py
method pending_to_transient(session: Session, instance: _O) → None
```

拦截特定对象的“待定到瞬态”的转换。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeSessionClassOrObject, 'pending_to_transient')
def receive_pending_to_transient(session, instance):
    "listen for the 'pending_to_transient' event"

    # ... (event handling logic) ...
```

当尚未刷新的待定对象从会话中移除时，会发生这种较少见的转换；这可能发生在`Session.rollback()`方法回滚事务时，或者当使用`Session.expunge()`方法时。

参数：

+   `session` – 目标`Session`

+   `instance` – 正在操作的 ORM 映射实例。

另请参见

对象生命周期事件

```py
method persistent_to_deleted(session: Session, instance: _O) → None
```

拦截特定对象的“持久到已删除”转换。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeSessionClassOrObject, 'persistent_to_deleted')
def receive_persistent_to_deleted(session, instance):
    "listen for the 'persistent_to_deleted' event"

    # ... (event handling logic) ...
```

当在 flush 过程中从数据库中删除持久化对象的标识时，会触发此事件，但是对象仍然与`Session`相关联，直到事务完成。

如果事务回滚，则对象再次转移到持久状态，并调用 `SessionEvents.deleted_to_persistent()` 事件。如果事务提交，则对象变为分离状态，这将触发 `SessionEvents.deleted_to_detached()` 事件。

请注意，虽然 `Session.delete()` 方法是将对象标记为已删除的主要公共接口，但由于级联规则的存在，许多对象会因级联规则而被删除，这些规则直到 flush 时才确定。因此，在 flush 进行之前，没有办法捕获每个将要删除的对象。因此，在 flush 结束时会调用 `SessionEvents.persistent_to_deleted()` 事件。

另请参见

对象生命周期事件

```py
method persistent_to_detached(session: Session, instance: _O) → None
```

拦截特定对象的“持久到分离”转换。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeSessionClassOrObject, 'persistent_to_detached')
def receive_persistent_to_detached(session, instance):
    "listen for the 'persistent_to_detached' event"

    # ... (event handling logic) ...
```

当持久化对象从会话中驱逐时，将会触发此事件。导致此情况发生的条件有很多，包括：

+   使用 `Session.expunge()` 或 `Session.close()` 等方法

+   在对象属于该会话事务的 INSERT 语句的一部分时，调用 `Session.rollback()` 方法

参数：

+   `session` – 目标 `Session`

+   `instance` – 正在操作的 ORM 映射实例。

+   `deleted` – 布尔值。如果为 True，则表示此对象因标记为已删除并被 flush 而转移到分离状态。

另请参见

对象生命周期事件

```py
method persistent_to_transient(session: Session, instance: _O) → None
```

拦截特定对象的“持久到瞬时”转换。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeSessionClassOrObject, 'persistent_to_transient')
def receive_persistent_to_transient(session, instance):
    "listen for the 'persistent_to_transient' event"

    # ... (event handling logic) ...
```

当已刷新的挂起对象从会话中被逐出时，发生了这种不太常见的转换；这可能发生在 `Session.rollback()` 方法回滚事务时。

参数：

+   `session` – 目标 `Session`

+   `instance` – 正在操作的 ORM 映射实例。

另请参阅

对象生命周期事件

```py
method transient_to_pending(session: Session, instance: _O) → None
```

捕获特定对象的“瞬态到挂起”转换。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeSessionClassOrObject, 'transient_to_pending')
def receive_transient_to_pending(session, instance):
    "listen for the 'transient_to_pending' event"

    # ... (event handling logic) ...
```

此事件是 `SessionEvents.after_attach()` 事件的一种特化，仅在此特定转换时调用。它通常在 `Session.add()` 调用期间被调用。

参数：

+   `session` – 目标 `Session`

+   `instance` – 正在操作的 ORM 映射实例。

另请参阅

对象生命周期事件

## 映射器事件

映射器事件挂钩涵盖与单个或多个 `Mapper` 对象相关的事情，这些对象是将用户定义的类映射到 `Table` 对象的中心配置对象。在 `Mapper` 级别发生的类型包括：

+   **每个对象的持久性操作** - 最常见的映射器钩子是工作单元钩子，例如 `MapperEvents.before_insert()`、`MapperEvents.after_update()` 等。这些事件与更粗粒度的会话级事件（例如 `SessionEvents.before_flush()`）形成对比，因为它们在刷新过程中以每个对象为基础发生；虽然对象上的更细粒度的活动更为直接，但 `Session` 功能的可用性有限。

+   **Mapper 配置事件** - 另一个主要的映射器钩子类别是在类被映射时发生的事件，当映射器被完成时以及当映射器集合被配置为相互引用时。这些事件包括 `MapperEvents.instrument_class()`、`MapperEvents.before_mapper_configured()` 和 `MapperEvents.mapper_configured()` 在单个 `Mapper` 级别，以及 `MapperEvents.before_configured()` 和 `MapperEvents.after_configured()` 在一组 `Mapper` 对象的级别。

| 对象名称 | 描述 |
| --- | --- |
| MapperEvents | 定义特定于映射的事件。 |

```py
class sqlalchemy.orm.MapperEvents
```

定义特定于映射的事件。

例如（e.g.）：

```py
from sqlalchemy import event

def my_before_insert_listener(mapper, connection, target):
    # execute a stored procedure upon INSERT,
    # apply the value to the row to be inserted
    target.calculated_value = connection.execute(
        text("select my_special_function(%d)" % target.special_number)
    ).scalar()

# associate the listener function with SomeClass,
# to execute during the "before_insert" hook
event.listen(
    SomeClass, 'before_insert', my_before_insert_listener)
```

可用的目标包括：

+   映射的类（mapped classes）

+   已映射或待映射类的未映射超类（使用 `propagate=True` 标志）

+   `Mapper` 对象

+   `Mapper` 类本身指示监听所有映射器。

映射器事件提供了对映射器的关键部分的钩子，包括与对象工具、对象加载和对象持久性相关的部分。特别是，持久性方法 `MapperEvents.before_insert()` 和 `MapperEvents.before_update()` 是增强正在持久化状态的流行位置 - 但是，这些方法在几个重要限制下运行。鼓励用户评估 `SessionEvents.before_flush()` 和 `SessionEvents.after_flush()` 方法，作为更灵活和用户友好的挂钩，在刷新期间应用额外的数据库状态。

在使用 `MapperEvents` 时，`listen()` 函数提供了几个修饰符。

参数（Parameters）：

+   `propagate=False` – 当为 True 时，事件监听器应应用于所有继承映射器和/或继承类的映射器，以及任何作为此监听器目标的映射器。

+   `raw=False` – 当为 True 时，传递给适用事件监听器函数的“target”参数将是实例的`InstanceState`管理对象，而不是映射实例本身。

+   `retval=False` –

    当为 True 时，用户定义的事件函数必须具有返回值，其目的是控制后续事件传播，或以其他方式通过映射器改变正在进行的操作。可能的返回值包括：

    +   `sqlalchemy.orm.interfaces.EXT_CONTINUE` - 正常继续事件处理。

    +   `sqlalchemy.orm.interfaces.EXT_STOP` - 取消链中所有后续事件处理程序。

    +   其他值 - 特定监听器指定的返回值。

**成员**

after_configured(), after_delete(), after_insert(), after_mapper_constructed(), after_update(), before_configured(), before_delete(), before_insert(), before_mapper_configured(), before_update(), dispatch, instrument_class(), mapper_configured()

**类签名**

类`sqlalchemy.orm.MapperEvents` (`sqlalchemy.event.Events`)

```py
method after_configured() → None
```

被称为一系列映射器已配置完成后。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeClass, 'after_configured')
def receive_after_configured():
    "listen for the 'after_configured' event"

    # ... (event handling logic) ...
```

每次调用`MapperEvents.after_configured()`事件时，都会在`configure_mappers()`函数完成其工作后调用该事件。通常情况下，当映射首次被使用时，以及每当新的映射器可用并检测到新的映射器使用时，会自动调用`configure_mappers()`。

与`MapperEvents.mapper_configured()`事件相比，该事件在配置操作进行时基于每个映射器调用；与该事件不同，当调用此事件时，所有交叉配置（例如反向引用）也将为任何待定的映射器提供。还与`MapperEvents.before_configured()`进行对比，该事件在系列映射器配置之前被调用。

此事件**仅**适用于`Mapper`类，而不适用于单个映射或映射类。它仅对所有映射作为一个整体调用：

```py
from sqlalchemy.orm import Mapper

@event.listens_for(Mapper, "after_configured")
def go():
    # ...
```

理论上，此事件每个应用程序调用一次，但实际上每当新映射器受到`configure_mappers()`调用的影响时都会调用。如果在已经使用现有映射之后构建新映射，则可能会再次调用此事件。为确保特定事件仅被调用一次且不再调用，可以应用`once=True`参数（自 0.9.4 版起新增）：

```py
from sqlalchemy.orm import mapper

@event.listens_for(mapper, "after_configured", once=True)
def go():
    # ...
```

另请参见

`MapperEvents.before_mapper_configured()`

`MapperEvents.mapper_configured()`

`MapperEvents.before_configured()`

```py
method after_delete(mapper: Mapper[_O], connection: Connection, target: _O) → None
```

在发出与该实例对应的 DELETE 语句后接收一个对象实例。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeClass, 'after_delete')
def receive_after_delete(mapper, connection, target):
    "listen for the 'after_delete' event"

    # ... (event handling logic) ...
```

注意

此事件**仅**适用于会话刷新操作，**不**适用于 ORM-启用的 INSERT、UPDATE 和 DELETE 语句中描述的 ORM DML 操作。要拦截 ORM DML 事件，请使用`SessionEvents.do_orm_execute()`。

此事件用于在给定连接上发出额外的 SQL 语句，以及执行与删除事件相关的应用程序特定的簿记。

该事件通常在一批相同类的对象的 DELETE 语句在先前步骤中一次性发出后被调用。

警告

映射级刷新事件仅允许对仅限于操作的行本地属性进行**非常有限的操作**，同时允许在给定的`Connection`上发出任何 SQL。**请完整阅读**映射级刷新事件中关于使用这些方法的指南；通常，应优先使用`SessionEvents.before_flush()`方法进行一般性的刷新更改。

参数：

+   `mapper` – 此事件的目标`Mapper`。

+   `connection` – 用于为此实例发出 DELETE 语句的`Connection`。这为特定于此实例的目标数据库上的当前事务提供了一个句柄。

+   `target` – 被删除的映射实例。如果事件配置为`raw=True`，则这将是与实例关联的`InstanceState`状态管理对象。

返回：

此事件不支持返回值。

另请参阅

持久性事件

```py
method after_insert(mapper: Mapper[_O], connection: Connection, target: _O) → None
```

在发出对应实例的 INSERT 语句后，会收到一个对象实例。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeClass, 'after_insert')
def receive_after_insert(mapper, connection, target):
    "listen for the 'after_insert' event"

    # ... (event handling logic) ...
```

注意

此事件**仅**适用于会话刷新操作，**不**适用于描述在 ORM-启用的 INSERT、UPDATE 和 DELETE 语句中的 ORM DML 操作。要拦截 ORM DML 事件，请使用`SessionEvents.do_orm_execute()`。

此事件用于修改实例发生 INSERT 后的仅 Python 状态，并在给定连接上发出附加的 SQL 语句。

在先前步骤中一次性发出它们的 INSERT 语句后，通常会为同一类的一批对象调用此事件。在极为罕见的情况下，如果这不是理想的情况，可以使用`batch=False`配置`Mapper`对象，这将导致实例批次被拆分为单个（性能较差）事件->持久化->事件步骤。

警告

仅允许在映射器级别刷新事件上执行**非常有限的操作**，仅限于对正在操作的行本地属性的操作，并允许在给定的`Connection`上发出任何 SQL。**请完全阅读**映射器级刷新事件中的注意事项，以获取有关使用这些方法的指南；通常，应优先使用`SessionEvents.before_flush()`方法进行一般刷新更改。

参数：

+   `mapper` – 此事件的目标`Mapper`。

+   `connection` – 用于为此实例发出 INSERT 语句的`Connection`。这提供了一个句柄到目标数据库上的当前事务，该事务特定于此实例。

+   `target` – 正在持久化的映射实例。如果事件配置为`raw=True`，则这将是与实例关联的`InstanceState`状态管理对象。

返回：

此事件不支持返回值。

另请参阅

持久性事件

```py
method after_mapper_constructed(mapper: Mapper[_O], class_: Type[_O]) → None
```

当`Mapper`完全构建完成时，接收一个类和映射器。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeClass, 'after_mapper_constructed')
def receive_after_mapper_constructed(mapper, class_):
    "listen for the 'after_mapper_constructed' event"

    # ... (event handling logic) ...
```

此事件在`Mapper`的初始构造函数完成后调用。这发生在`MapperEvents.instrument_class()`事件之后，以及在`Mapper`对其参数进行初始遍历以生成其`MapperProperty`对象集合之后，这些对象可通过`Mapper.get_property()`方法和`Mapper.iterate_properties`属性访问。

此事件与 `MapperEvents.before_mapper_configured()` 事件不同之处在于，它在 `Mapper` 的构造函数内部调用，而不是在 `registry.configure()` 进程内调用。目前，此事件是唯一适用于希望在构造此 `Mapper` 时创建其他映射类的处理程序的事件，这些映射类将在下次运行 `registry.configure()` 时成为同一配置步骤的一部分。

版本 2.0.2 中的新功能。

另请参阅

对象版本控制 - 一个示例，说明使用 `MapperEvents.before_mapper_configured()` 事件创建新的映射器以记录对象的更改审计历史。

```py
method after_update(mapper: Mapper[_O], connection: Connection, target: _O) → None
```

在发出对应于该实例的 UPDATE 语句后接收对象实例。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeClass, 'after_update')
def receive_after_update(mapper, connection, target):
    "listen for the 'after_update' event"

    # ... (event handling logic) ...
```

注意

此事件**仅**适用于 会话刷新操作，并且**不**适用于在 ORM-Enabled INSERT、UPDATE 和 DELETE 语句 中描述的 ORM DML 操作。要拦截 ORM DML 事件，请使用 `SessionEvents.do_orm_execute()`。

此事件用于在发生 UPDATE 后修改仅在 Python 中的实例状态，以及在给定连接上发出额外的 SQL 语句。

此方法对所有标记为“脏”的实例进行调用，*甚至对其列基属性没有净变化* 的实例，并且没有进行 UPDATE 语句。当任何列基属性的“设置属性”操作被调用或任何集合被修改时，对象被标记为脏。如果在更新时，没有列基属性有任何净变化，将不会发出 UPDATE 语句。这意味着发送到 `MapperEvents.after_update()` 的实例*不*保证已发出 UPDATE 语句。

要检测对象上基于列的属性是否有净变化，并因此导致 UPDATE 语句，请使用 `object_session(instance).is_modified(instance, include_collections=False)`。

该事件通常在一系列相同类的对象的 UPDATE 语句一次性发出之后被调用。在极少数情况下，如果这不是期望的情况，那么可以将 `Mapper` 配置为 `batch=False`，这将导致实例批次被拆分为单个（性能较差）的事件->持久化->事件步骤。

警告

映射器级别的刷新事件仅允许对仅限于正在操作的行的属性执行**非常有限的操作**，以及允许在给定的 `Connection` 上发出任何 SQL。**请完全阅读** 映射器级刷新事件 备注中关于使用这些方法的指导原则；通常，应优先考虑使用 `SessionEvents.before_flush()` 方法进行通用刷新更改。

参数：

+   `mapper` – 此事件的目标 `Mapper`。

+   `connection` – 用于为此实例发出 UPDATE 语句的 `Connection`。这提供了一个句柄到当前事务的目标数据库，特定于这个实例。

+   `target` – 正在持久化的映射实例。如果事件配置为 `raw=True`，那么这将是与实例相关联的 `InstanceState` 状态管理对象。

返回值：

此事件不支持返回值。

另请参阅

持久化事件

```py
method before_configured() → None
```

在一系列映射器被配置之前调用。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeClass, 'before_configured')
def receive_before_configured():
    "listen for the 'before_configured' event"

    # ... (event handling logic) ...
```

`MapperEvents.before_configured()` 事件在每次调用 `configure_mappers()` 函数时被调用，在该函数执行任何工作之前。

此事件**仅**可应用于 `Mapper` 类，而不是单个映射或映射类。它仅对所有映射作为整体调用：

```py
from sqlalchemy.orm import Mapper

@event.listens_for(Mapper, "before_configured")
def go():
    ...
```

将此事件与 `MapperEvents.after_configured()` 进行对比，后者在一系列映射器配置完成后调用，以及 `MapperEvents.before_mapper_configured()` 和 `MapperEvents.mapper_configured()`，这两者都在每个映射器基础上调用。

理论上，此事件每个应用程序调用一次，但实际上，任何时候新的映射器要受 `configure_mappers()` 调用影响时都会调用此事件。如果在使用现有映射器之后构造新映射，则可能会再次调用此事件。要确保特定事件仅被调用一次且不再调用，可以应用 `once=True` 参数（0.9.4 中新增）：

```py
from sqlalchemy.orm import mapper

@event.listens_for(mapper, "before_configured", once=True)
def go():
    ...
```

另请参阅

`MapperEvents.before_mapper_configured()`

`MapperEvents.mapper_configured()`

`MapperEvents.after_configured()`

```py
method before_delete(mapper: Mapper[_O], connection: Connection, target: _O) → None
```

在发出与该实例对应的 DELETE 语句之前接收一个对象实例。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeClass, 'before_delete')
def receive_before_delete(mapper, connection, target):
    "listen for the 'before_delete' event"

    # ... (event handling logic) ...
```

注意

此事件**仅**适用于会话刷新操作，**不适用于**描述在 ORM 启用的 INSERT、UPDATE 和 DELETE 语句中的 ORM DML 操作。要拦截 ORM DML 事件，请使用 `SessionEvents.do_orm_execute()`。

此事件用于在给定连接上发出附加的 SQL 语句，以及执行与删除事件相关的应用程序特定簿记。

在后续步骤中一次性发出一批相同类别对象的 DELETE 语句之前，通常会为这个事件调用一次。

警告

映射器级刷新事件仅允许对仅针对正在操作的行的属性进行**非常有限的操作**，以及允许在给定的 `Connection` 上发出任何 SQL。**请务必完整阅读** 映射器级刷新事件 中的注释，以获取有关使用这些方法的指导；通常情况下，应首选 `SessionEvents.before_flush()` 方法进行一般性的刷新更改。

参数：

+   `mapper` – 此事件的目标`Mapper`。

+   `connection` – 用于为此实例发出 DELETE 语句的`Connection`。这提供了一个句柄到当前事务的目标数据库，特定于此实例。

+   `target` – 正在删除的映射实例。如果事件配置为`raw=True`，则这将是与实例相关联的`InstanceState`状态管理对象。

返回：

此事件不支持返回值。

请参阅

持久性事件

```py
method before_insert(mapper: Mapper[_O], connection: Connection, target: _O) → None
```

在发出与该实例对应的 INSERT 语句之前接收对象实例。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeClass, 'before_insert')
def receive_before_insert(mapper, connection, target):
    "listen for the 'before_insert' event"

    # ... (event handling logic) ...
```

注意

此事件**仅适用于**会话刷新操作，并**不适用于**描述在 ORM 启用的 INSERT、UPDATE 和 DELETE 语句中的 ORM DML 操作。要拦截 ORM DML 事件，请使用`SessionEvents.do_orm_execute()`。

此事件用于修改实例之前的本地、非对象相关属性，并在给定的连接上发出附加 SQL 语句。

在稍后的步骤中，通常会为同一类对象的一批对象调用此事件，然后在一次性发出它们的 INSERT 语句之前。在极少数情况下，如果这不是所需的，可以使用`Mapper`对象配置`batch=False`，这将导致实例的批处理被分解为单个（性能较差的）事件->持久性->事件步骤。

警告

Mapper 级别的刷新事件仅允许**非常有限的操作**，仅限于操作中的行本地属性，并允许在给定的`Connection`上发出任何 SQL。**请完全阅读**Mapper 级别的刷新事件中的注意事项，以获取使用这些方法的指南；通常，应优先考虑`SessionEvents.before_flush()`方法进行刷新时的一般更改。

参数：

+   `mapper` – 此事件的目标`Mapper`。

+   `connection` – 用于为此实例发出 INSERT 语句的`Connection`。这提供了一个句柄进入目标数据库上的当前事务，特定于此实例。

+   `target` – 正在持久化的映射实例。如果事件配置为`raw=True`，则这将是与实例关联的`InstanceState`状态管理对象。

返回：

此事件不支持返回值。

另请参见

持久性事件

```py
method before_mapper_configured(mapper: Mapper[_O], class_: Type[_O]) → None
```

在特定映射器配置之前调用。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeClass, 'before_mapper_configured')
def receive_before_mapper_configured(mapper, class_):
    "listen for the 'before_mapper_configured' event"

    # ... (event handling logic) ...
```

此事件旨在允许在配置步骤中跳过特定的映射器，通过返回`interfaces.EXT_SKIP`符号，该符号指示给`configure_mappers()`调用，表明应跳过当前配置运行中的此特定映射器（或使用`propagate=True`时的映射器层次结构）。当跳过一个或多个映射器时，"new mappers"标志将保持设置，这意味着在使用映射器时将继续调用`configure_mappers()`函数，以继续尝试配置所有可用的映射器。

与其他配置级别事件相比，`MapperEvents.before_configured()`、`MapperEvents.after_configured()`和`MapperEvents.mapper_configured()`，:meth;`.MapperEvents.before_mapper_configured`事件在注册时提供有意义的返回值，当使用`retval=True`参数注册时。

版本 1.3 中的新内容。

例如：

```py
from sqlalchemy.orm import EXT_SKIP

Base = declarative_base()

DontConfigureBase = declarative_base()

@event.listens_for(
    DontConfigureBase,
    "before_mapper_configured", retval=True, propagate=True)
def dont_configure(mapper, cls):
    return EXT_SKIP
```

另请参见

`MapperEvents.before_configured()`

`MapperEvents.after_configured()`

`MapperEvents.mapper_configured()`

```py
method before_update(mapper: Mapper[_O], connection: Connection, target: _O) → None
```

在发出与该实例对应的 UPDATE 语句之前接收对象实例。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeClass, 'before_update')
def receive_before_update(mapper, connection, target):
    "listen for the 'before_update' event"

    # ... (event handling logic) ...
```

注意

此事件**仅**适用于会话刷新操作，**不**适用于在 ORM-启用的 INSERT、UPDATE 和 DELETE 语句中描述的 ORM DML 操作。要拦截 ORM DML 事件，请使用`SessionEvents.do_orm_execute()`。

此事件用于在更新发生之前修改实例上的本地、与对象无关的属性，以及在给定连接上发出附加的 SQL 语句。

此方法将为所有标记为“脏”的实例调用，*即使它们的基于列的属性没有净变化*。当对其基于列的属性之一调用“设置属性”操作或修改其任何集合时，对象将被标记为脏。如果在更新时，没有基于列的属性有任何净变化，将不会发出 UPDATE 语句。这意味着将发送到`MapperEvents.before_update()`的实例*不*保证会发出 UPDATE 语句，尽管您可以通过修改属性以存在值的净变化来影响结果。

要检测对象上的基于列的属性是否有净变化，并因此生成 UPDATE 语句，请使用 `object_session(instance).is_modified(instance, include_collections=False)`。

在稍后的步骤中，通常会一次调用相同类的一批对象的此事件，然后发出它们的 UPDATE 语句。在极其罕见的情况下，如果这不是理想的情况，可以将`Mapper`配置为 `batch=False`，这将导致将实例批处理为单个（并且性能更差）事件->持久化->事件步骤。

警告

映射器级刷新事件仅允许对仅限于正在操作的行的属性进行**非常有限的操作**，同时允许在给定的`Connection`上发出任何 SQL。 **请务必阅读**映射器级刷新事件中的注意事项，以获取有关使用这些方法的指导；一般来说，应优先考虑`SessionEvents.before_flush()`方法进行通用刷新更改。

参数：

+   `mapper` – 此事件的目标`Mapper`。

+   `connection` – 用于为此实例发出 UPDATE 语句的`Connection`。这为当前事务提供了一个句柄，该事务针对与此实例特定相关的目标数据库。

+   `target` – 正在持久化的映射实例。如果事件配置为`raw=True`，则这将是与实例关联的`InstanceState`状态管理对象。

返回：

此事件不支持返回值。

另请参阅：

持久化事件

```py
attribute dispatch: _Dispatch[_ET] = <sqlalchemy.event.base.MapperEventsDispatch object>
```

回溯到 _Dispatch 类。

双向关联到 _Dispatch._events

```py
method instrument_class(mapper: Mapper[_O], class_: Type[_O]) → None
```

当首次构建映射器时，尚未应用到映射类的仪器。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeClass, 'instrument_class')
def receive_instrument_class(mapper, class_):
    "listen for the 'instrument_class' event"

    # ... (event handling logic) ...
```

此事件是映射器构造的最早阶段。大多数映射器属性尚未初始化。要在初始映射器构造中接收事件，其中提供了基本状态，例如`Mapper.attrs`集合，可能更好地选择`MapperEvents.after_mapper_constructed()`事件。

此侦听器可以应用于整个`Mapper`类，也可以应用于任何用作将要映射的类的基类（使用`propagate=True`标志）：

```py
Base = declarative_base()

@event.listens_for(Base, "instrument_class", propagate=True)
def on_new_class(mapper, cls_):
    " ... "
```

参数：

+   `mapper` – 此事件的目标是`Mapper`。

+   `class_` – 映射类。

另请参阅：

`MapperEvents.after_mapper_constructed()`

```py
method mapper_configured(mapper: Mapper[_O], class_: Type[_O]) → None
```

在`configure_mappers()`调用范围内，特定映射器完成其自身配置时调用。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeClass, 'mapper_configured')
def receive_mapper_configured(mapper, class_):
    "listen for the 'mapper_configured' event"

    # ... (event handling logic) ...
```

当`MapperEvents.mapper_configured()`事件在`configure_mappers()`函数通过当前未配置的映射器列表时遇到每个映射器时，将被调用。`configure_mappers()`通常在映射首次使用时自动调用，以及每当新映射器可用并检测到新的映射器使用时。

当事件被调用时，映射器应该处于其最终状态，但**不包括**可能从其他映射器调用的反向引用；它们可能仍在配置操作中待处理。而通过 `relationship.back_populates` 参数配置的双向关系*将*完全可用，因为这种关系样式不依赖于其他可能尚未配置的映射器来知道它们的存在。

对于一个确保**所有**映射器都准备就绪的事件，包括仅在其他映射上定义的反向引用，使用 `MapperEvents.after_configured()` 事件；该事件仅在所有已知映射完全配置后才调用。

`MapperEvents.mapper_configured()` 事件，与 `MapperEvents.before_configured()` 或 `MapperEvents.after_configured()` 不同，对于每个映射器/类分别调用，且映射器本身被传递给事件。它也只对特定的映射器调用一次。因此，该事件对于一次仅在特定映射器基础上受益的配置步骤非常有用，不要求“backref”配置必须已准备好。

参数：

+   `mapper` – `Mapper` 是此事件的目标。

+   `class_` – 映射类。

另请参阅

`MapperEvents.before_configured()`

`MapperEvents.after_configured()`

`MapperEvents.before_mapper_configured()`

## 实例事件

实例事件专注于 ORM 映射实例的构造，包括当它们被实例化为瞬态对象时，当它们从数据库加载并成为持久化对象时，以及当数据库刷新或对象上的过期操作发生时。

| 对象名称 | 描述 |
| --- | --- |
| InstanceEvents | 定义了与对象生命周期特定的事件。 |

```py
class sqlalchemy.orm.InstanceEvents
```

定义了与对象生命周期特定的事件。

例如：

```py
from sqlalchemy import event

def my_load_listener(target, context):
    print("on load!")

event.listen(SomeClass, 'load', my_load_listener)
```

可用的目标包括：

+   映射类

+   映射或将要映射的类的未映射超类（使用 `propagate=True` 标志）

+   `Mapper`对象

+   `Mapper`类本身指示监听所有映射器。

实例事件与映射器事件密切相关，但更具体于实例及其仪器，而不是其持久性系统。

在使用`InstanceEvents`时，`listen()`函数提供了几个修饰符。

参数：

+   `propagate=False` – 当为 True 时，事件监听器应该应用于所有继承类��以及作为此监听器目标的类。

+   `raw=False` – 当为 True 时，传递给适用事件监听器函数的“target”参数将是实例的`InstanceState`管理对象，而不是映射实例本身。

+   `restore_load_context=False` –

    适用于`InstanceEvents.load()`和`InstanceEvents.refresh()`事件。在事件挂钩完成时恢复对象的加载器上下文，以便持续的急切加载操作继续适当地针对对象。如果未设置此标志，并且在这些事件之一中将对象移动到新的加载器上下文，则会发出警告。

    自版本 1.3.14 起新增。

**成员**

dispatch, expire(), first_init(), init(), init_failure(), load(), pickle(), refresh(), refresh_flush(), unpickle()

**类签名**

类`sqlalchemy.orm.InstanceEvents` (`sqlalchemy.event.Events`)

```py
attribute dispatch: _Dispatch[_ET] = <sqlalchemy.event.base.InstanceEventsDispatch object>
```

回溯到 _Dispatch 类。

双向反对 _Dispatch._events

```py
method expire(target: _O, attrs: Iterable[str] | None) → None
```

在某些属性或其子集已过期后接收对象实例。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeClass, 'expire')
def receive_expire(target, attrs):
    "listen for the 'expire' event"

    # ... (event handling logic) ...
```

‘keys’是属性名称列表。如果为 None，则整个状态已过期。

参数：

+   `target` – 映射实例。如果事件配置为`raw=True`，则这将是与实例关联的`InstanceState`状态管理对象。

+   `attrs` – 已过期的属性名称序列，如果所有属性都已过期，则为 None。

```py
method first_init(manager: ClassManager[_O], cls: Type[_O]) → None
```

在调用特定映射的第一个实例时调用。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeClass, 'first_init')
def receive_first_init(manager, cls):
    "listen for the 'first_init' event"

    # ... (event handling logic) ...
```

当为该特定类第一次调用`__init__`方法时，调用此事件。事件在`__init__`实际进行之前以及在调用`InstanceEvents.init()`事件之前被调用。

```py
method init(target: _O, args: Any, kwargs: Any) → None
```

在构造函数被调用时接收一个实例。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeClass, 'init')
def receive_init(target, args, kwargs):
    "listen for the 'init' event"

    # ... (event handling logic) ...
```

这个方法只在对象的用户空间构造期间调用，与对象的构造函数一起，例如它的`__init__`方法。当对象从数据库加载时不会调用它；请参见`InstanceEvents.load()`事件以拦截数据库加载。

事件在实际调用对象的`__init__`构造函数之前被调用。`kwargs`字典可以被就地修改，以影响传递给`__init__`的内容。

参数：

+   `target` – 映射的实例。如果事件配置为`raw=True`，那么这将是与实例关联的`InstanceState`状态管理对象。

+   `args` – 传递给`__init__`方法的位置参数。这被传递为一个元组，目前是不可变的。

+   `kwargs` – 传递给`__init__`方法的关键字参数。这个结构*可以*被就地修改。

另请参见

`InstanceEvents.init_failure()`

`InstanceEvents.load()`

```py
method init_failure(target: _O, args: Any, kwargs: Any) → None
```

在实例构造函数被调用并引发异常时接收一个实例。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeClass, 'init_failure')
def receive_init_failure(target, args, kwargs):
    "listen for the 'init_failure' event"

    # ... (event handling logic) ...
```

这个方法只在对象的用户空间构造期间调用，与对象的构造函数一起，例如它的`__init__`方法。当对象从数据库加载时不会调用它。

事件在`__init__`方法引发异常后被调用。事件被调用后，原始异常被重新引发，以便对象的构造仍然引发异常。引发的实际异常和堆栈跟踪应该存在于`sys.exc_info()`中。

参数：

+   `target` – 映射的实例。如果事件配置为`raw=True`，那么这将是与实例关联的`InstanceState`状态管理对象。

+   `args` – 传递给`__init__`方法的位置参数。

+   `kwargs` – 传递给`__init__`方法的关键字参数。

另请参见

`InstanceEvents.init()`

`InstanceEvents.load()`

```py
method load(target: _O, context: QueryContext) → None
```

在通过`__new__`创建对象实例并进行初始属性填充后接收对象实例。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeClass, 'load')
def receive_load(target, context):
    "listen for the 'load' event"

    # ... (event handling logic) ...
```

这通常发生在基于传入结果行创建实例时，并且仅在该实例的生命周期中调用一次。

警告

在结果行加载期间，当处理此实例接收到的第一行时，将调用此事件。当使用集合导向属性进行急加载时，尚未发生要加载/处理的附加行，以便加载后续集合项。这既导致集合不会完全加载，也导致如果在此事件处理程序中发生操作，该操作会为对象发出另一个数据库加载操作，则对象的“加载上下文”可能会发生变化并干扰现有的急加载器仍在进行中。

可能导致事件处理程序内“加载上下文”更改的示例包括但不限于：

+   访问未包含在行中的延迟属性将触发“取消延迟”操作并刷新对象

+   访问联合继承子类上不属于行的属性将触发刷新操作。

从 SQLAlchemy 1.3.14 开始，当发生此情况时会发出警告。可以在事件上使用`InstanceEvents.restore_load_context`选项来防止此警告；这将确保在调用事件后保持对象的现有加载上下文：

```py
@event.listens_for(
    SomeClass, "load", restore_load_context=True)
def on_load(instance, context):
    instance.some_unloaded_attribute
```

从版本 1.3.14 开始更改：添加了`InstanceEvents.restore_load_context`和`SessionEvents.restore_load_context`标志，适用于“加载”事件，确保在事件钩子完成时恢复对象的加载上下文；如果对象的加载上下文在未设置此标志的情况下发生更改，则会发出警告。

`InstanceEvents.load()` 事件也可以以类方法装饰器的形式使用，称为`reconstructor()`。

参数：

+   `target` – 映射实例。如果事件配置为`raw=True`，则这将是与实例关联的`InstanceState`状态管理对象。

+   `context` – 与当前进行中的`Query`对应的`QueryContext`。如果加载不对应于`Query`，例如在`Session.merge()`期间，此参数可能为`None`���

另请参阅

在加载过程中保持非映射状态

`InstanceEvents.init()`

`InstanceEvents.refresh()`

`SessionEvents.loaded_as_persistent()`

```py
method pickle(target: _O, state_dict: _InstanceDict) → None
```

在关联状态被 pickle 时接收一个对象实例。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeClass, 'pickle')
def receive_pickle(target, state_dict):
    "listen for the 'pickle' event"

    # ... (event handling logic) ...
```

参数：

+   `target` – 映射实例。如果事件配置为`raw=True`，则这将是与实例关联的`InstanceState`状态管理对象。

+   `state_dict` – 由`__getstate__`返回的包含要 pickle 的状态的字典。

```py
method refresh(target: _O, context: QueryContext, attrs: Iterable[str] | None) → None
```

在从查询中刷新一个或多个属性后接收一个对象实例。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeClass, 'refresh')
def receive_refresh(target, context, attrs):
    "listen for the 'refresh' event"

    # ... (event handling logic) ...
```

与`InstanceEvents.load()`方法形成对比，该方法在对象首次从查询中加载时被调用。

注意

在加载程序完成之前，此事件在加载程序进程中被调用，对象的状态可能不完整。此外，在对象上调用行级刷新操作将使对象进入新的加载程序上下文，干扰现有的加载上下文。有关如何使用`InstanceEvents.restore_load_context`参数解决此情况的背景，请参阅有关`InstanceEvents.load()`的注释。

参数：

+   `target` – 映射实例。如果事件配置为`raw=True`，则这将是与实例关联的`InstanceState`状态管理对象。

+   `context` – 与当前进行中的`Query`对应的`QueryContext`。

+   `attrs` – 已填充的属性名称序列，如果所有列映射的非延迟属性都已填充，则为 None。

另请参阅

在加载时保持未映射状态

`InstanceEvents.load()`

```py
method refresh_flush(target: _O, flush_context: UOWTransaction, attrs: Iterable[str] | None) → None
```

在对象状态持久化期间，接收一个包含一个或多个包含列级默认值或更新处理程序的属性已被刷新的对象实例。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeClass, 'refresh_flush')
def receive_refresh_flush(target, flush_context, attrs):
    "listen for the 'refresh_flush' event"

    # ... (event handling logic) ...
```

这个事件与 `InstanceEvents.refresh()` 相同，只是在工作单元刷新过程中调用，并且仅包括具有列级默认值或更新处理程序的非主键列，包括 Python 可调用对象以及可能通过 RETURNING 子句获取的服务器端默认值和触发器。

注意

当 `InstanceEvents.refresh_flush()` 事件触发时，对于一个被插入的对象以及一个被更新的对象，该事件主要针对更新过程；这主要是一个内部工件，指出插入动作也可以触发此事件，注意**插入行的主键列在此事件中被明确省略**。为了拦截对象的新插入状态，`SessionEvents.pending_to_persistent()` 和 `MapperEvents.after_insert()` 是更好的选择。

参数：

+   `target` – 映射实例。如果事件配置为 `raw=True`，则此处将是与实例关联的 `InstanceState` 状态管理对象。

+   `flush_context` – 处理刷新细节的内部 `UOWTransaction` 对象。

+   `attrs` – 已填充的属性名称序列。

另请参阅

在加载时保持未映射状态

获取服务器生成的默认值

列插入/更新默认值

```py
method unpickle(target: _O, state_dict: _InstanceDict) → None
```

在相关联的状态被反序列化后，接收一个对象实例。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeClass, 'unpickle')
def receive_unpickle(target, state_dict):
    "listen for the 'unpickle' event"

    # ... (event handling logic) ...
```

参数：

+   `target` – 映射实例。如果事件配置为 `raw=True`，则此处将是与实例关联的 `InstanceState` 状态管理对象。

+   `state_dict` – 发送到 `__setstate__` 的字典，包含被序列化的状态字典。

## 属性事件

属性事件在 ORM 映射对象的各个属性发生时触发。这些事件为诸如自定义验证函数和反向引用处理程序等功能奠定了基础。

另请参阅

更改属性行为

| 对象名称 | 描述 |
| --- | --- |
| 属性事件 | 定义对象属性的事件。 |

```py
class sqlalchemy.orm.AttributeEvents
```

定义对象属性的事件。

这些通常在目标类的类绑定描述符上定义。

例如，要注册一个将接收`AttributeEvents.append()`事件的监听器：

```py
from sqlalchemy import event

@event.listens_for(MyClass.collection, 'append', propagate=True)
def my_append_listener(target, value, initiator):
    print("received append event for target: %s" % target)
```

当`AttributeEvents.retval`标志传递给`listen()`或`listens_for()`时，监听器可以选择返回可能修改的值，如下所示，使用`AttributeEvents.set()`事件进行说明：

```py
def validate_phone(target, value, oldvalue, initiator):
    "Strip non-numeric characters from a phone number"

    return re.sub(r'\D', '', value)

# setup listener on UserContact.phone attribute, instructing
# it to use the return value
listen(UserContact.phone, 'set', validate_phone, retval=True)
```

像上面的验证函数也可以引发异常，如`ValueError`以停止操作。

当应用监听器到具有映射子类的映射类时，`AttributeEvents.propagate`标志也很重要，例如在使用映射器继承模式时：

```py
@event.listens_for(MySuperClass.attr, 'set', propagate=True)
def receive_set(target, value, initiator):
    print("value set: %s" % target)
```

下面是可用于`listen()`和`listens_for()`函数的修饰符的完整列表。

参数：

+   `active_history=False` – 当为 True 时，表示“set”事件希望无条件地接收被替换的“旧”值，即使这需要触发数据库加载。请注意，`active_history`也可以通过`column_property()`和`relationship()`直接设置。

+   `propagate=False` – 当为 True 时，监听器函数将不仅为给定的类属性建立，还将为该类的所有当前子类以及该类的所有未来子类上具有相同名称的属性建立，使用一个额外的监听器来监听仪器事件。

+   `raw=False` – 当为 True 时，事件的“target”参数将是`InstanceState`管理对象，而不是映射实例本身。

+   `retval=False` – 当为 True 时，用户定义的事件监听必须从函数中返回“value”参数。这使得监听函数有机会更改最终用于“set”或“append”事件的值。

**成员**

append(), append_wo_mutation(), bulk_replace(), dispatch, dispose_collection(), init_collection(), init_scalar(), modified(), remove(), set()

**类签名**

类 `sqlalchemy.orm.AttributeEvents` (`sqlalchemy.event.Events`)

```py
method append(target: _O, value: _T, initiator: Event, *, key: EventConstants = EventConstants.NO_KEY) → _T | None
```

接收集合附加事件。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeClass.some_attribute, 'append')
def receive_append(target, value, initiator):
    "listen for the 'append' event"

    # ... (event handling logic) ...
```

在每个元素被附加到集合时，都会调用附加事件。这适用于单个项的附加以及“批量替换”操作。

参数:

+   `target` – 接收事件的对象实例。如果监听器以 `raw=True` 注册，这将是 `InstanceState` 对象。

+   `value` – 被附加的值。如果此监听器以 `retval=True` 注册，则监听函数必须返回此值，或者替换它的新值。

+   `initiator` – 表示事件启动的 `Event` 实例。可能会被后向引用处理程序修改以控制链接的事件传播，以及被检查以获取有关事件源的信息。

+   `key` –

    当使用 `AttributeEvents.include_key` 参数设置为 True 来建立事件时，这将是操作中使用的键，例如 `collection[some_key_or_index] = value`。如果未使用 `AttributeEvents.include_key` 设置事件，将根本不传递此参数给事件；这是为了允许与不包括 `key` 参数的现有事件处理程序向后兼容。

    版本 2.0 中的新功能。

返回:

如果事件以 `retval=True` 注册，则应返回给定值或新的有效值。

另请参阅

`AttributeEvents` - 关于监听器选项的背景信息，例如传播到子类。

`AttributeEvents.bulk_replace()`

```py
method append_wo_mutation(target: _O, value: _T, initiator: Event, *, key: EventConstants = EventConstants.NO_KEY) → None
```

接收集合附加事件，其中集合实际上未被改变。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeClass.some_attribute, 'append_wo_mutation')
def receive_append_wo_mutation(target, value, initiator):
    "listen for the 'append_wo_mutation' event"

    # ... (event handling logic) ...
```

此事件与 `AttributeEvents.append()` 不同，因为它是为去重集合（如集合和字典）触发的，当对象已存在于目标集合中时。该事件没有返回值，并且给定对象的标识不能更改。

当集合已通过后向引用事件发生变异时，该事件用于将对象级联到`Session`中。

参数：

+   `target` – 接收事件的对象实例。如果侦听器以 `raw=True` 注册，这将是 `InstanceState` 对象。

+   `value` – 如果对象在集合中尚不存在，则将追加的值。

+   `initiator` – 一个表示事件启动的`Event`实例。可以通过后向引用处理程序修改其原始值，以控制链接的事件传播，也可以检查有关事件源的信息。

+   `key` –

    当使用 `AttributeEvents.include_key` 参数设置为 True 建立事件时，这将是操作中使用的键，例如 `collection[some_key_or_index] = value`。如果未使用 `AttributeEvents.include_key` 设置事件，根本不会将参数传递给事件；这是为了与不包含 `key` 参数的现有事件处理程序向后兼容。

    在版本 2.0 中新增。

返回：

没有为此事件定义返回值。

在版本 1.4.15 中新增。

```py
method bulk_replace(target: _O, values: Iterable[_T], initiator: Event, *, keys: Iterable[EventConstants] | None = None) → None
```

接收一个集合“批量替换”事件。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeClass.some_attribute, 'bulk_replace')
def receive_bulk_replace(target, values, initiator):
    "listen for the 'bulk_replace' event"

    # ... (event handling logic) ...
```

此事件在作为 ORM 对象处理之前的批量集合设置操作的传入值序列上被调用，可以在值被视为 ORM 对象之前就地修改。这是一个“早期挂钩”，在批量替换例程尝试协调哪些对象已经存在于集合中，哪些对象正在被净替换操作移除之前运行。

通常情况下，此方法与`AttributeEvents.append()`事件的使用结合。当同时使用这两个事件时，请注意，批量替换操作将为所有新项目调用`AttributeEvents.append()`事件，即使在为整个集合调用`AttributeEvents.bulk_replace()`之后，也会调用`AttributeEvents.append()`事件。为了确定`AttributeEvents.append()`事件是否属于批量替换，请使用符号`attributes.OP_BULK_REPLACE`来测试传入的 initiator：

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

版本 1.2 中的新内容。

参数：

+   `target` – 接收事件的对象实例。如果监听器注册为`raw=True`，这将是`InstanceState`对象。

+   `value` – 正在设置的值的序列（例如列表）。处理程序可以直接修改此列表。

+   `initiator` – 代表事件启动的`Event`实例。

+   `keys` –

    当使用`AttributeEvents.include_key`参数设置为 True 来建立事件时，这将是操作中使用的键序列，通常仅用于字典更新。如果未使用`AttributeEvents.include_key`来设置事件，则根本不会将参数传递给事件；这是为了允许与不包括`key`参数的现有事件处理程序保持向后兼容性。

    版本 2.0 中的新内容。

另请参阅

`AttributeEvents` - 关于监听器选项的背景，例如传播到子类。

```py
attribute dispatch: _Dispatch[_ET] = <sqlalchemy.event.base.AttributeEventsDispatch object>
```

参考回 _Dispatch 类。

双向反对 _Dispatch._events

```py
method dispose_collection(target: _O, collection: Collection[Any], collection_adapter: CollectionAdapter) → None
```

收到一个“collection dispose”事件。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeClass.some_attribute, 'dispose_collection')
def receive_dispose_collection(target, collection, collection_adapter):
    "listen for the 'dispose_collection' event"

    # ... (event handling logic) ...
```

当集合被替换时，此事件将触发基于集合的属性，即：

```py
u1.addresses.append(a1)

u1.addresses = [a2, a3]  # <- old collection is disposed
```

旧集合将包含其先前的内容。

版本 1.2 中的更改：传递给`AttributeEvents.dispose_collection()`的集合现在将在处理之前保持其内容；以前，集合将为空。

另请参阅

`AttributeEvents` - 关于监听器选项的背景，例如传播到子类。

```py
method init_collection(target: _O, collection: Type[Collection[Any]], collection_adapter: CollectionAdapter) → None
```

收到一个“collection init”事件。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeClass.some_attribute, 'init_collection')
def receive_init_collection(target, collection, collection_adapter):
    "listen for the 'init_collection' event"

    # ... (event handling logic) ...
```

当为空属性首次生成初始“空集合”时，以及当集合被替换为新集合时（例如通过设置事件），将触发此事件。

例如，假设`User.addresses`是基于关系的集合，事件在此处触发：

```py
u1 = User()
u1.addresses.append(a1)  #  <- new collection
```

在替换操作期间也会发生：

```py
u1.addresses = [a2, a3]  #  <- new collection
```

参数：

+   `target` – 接收事件的对象实例。如果监听器以`raw=True`注册，这将是`InstanceState`对象。

+   `collection` – 新集合。这将始终从`relationship.collection_class`指定的内容生成，并且始终为空。

+   `collection_adapter` – 将调解对集合的内部访问的`CollectionAdapter`。

另请参阅

`AttributeEvents` - 关于监听器选项的背景，例如传播到子类。

`AttributeEvents.init_scalar()` - 此事件的“标量”版本。

```py
method init_scalar(target: _O, value: _T, dict_: Dict[Any, Any]) → None
```

接收一个标量“init”事件。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeClass.some_attribute, 'init_scalar')
def receive_init_scalar(target, value, dict_):
    "listen for the 'init_scalar' event"

    # ... (event handling logic) ...
```

当访问未初始化的、未持久化的标量属性时，例如读取时，将调用此事件：

```py
x = my_object.some_attribute
```

当未初始化属性发生此事件时，ORM 的默认行为是返回值`None`；请注意，这与 Python 通常引发`AttributeError`的行为不同。此处的事件可用于自定义实际返回的值，假设事件监听器将镜像配置在 Core `Column`对象上的默认生成器。

由于`Column`上的默认生成器也可能产生像时间戳这样的变化值，因此`AttributeEvents.init_scalar()`事件处理程序也可用于**设置**新返回的值，以便 Core 级别的默认生成函数实际上只触发一次，但在访问非持久化对象上的属性时。通常，当访问未初始化属性时，不会对对象的状态进行任何更改（在较旧的 SQLAlchemy 版本中实际上会更改对象的状态）。

如果列上的默认生成器返回特定常量，则可以使用处理程序如下：

```py
SOME_CONSTANT = 3.1415926

class MyClass(Base):
    # ...

    some_attribute = Column(Numeric, default=SOME_CONSTANT)

@event.listens_for(
    MyClass.some_attribute, "init_scalar",
    retval=True, propagate=True)
def _init_some_attribute(target, dict_, value):
    dict_['some_attribute'] = SOME_CONSTANT
    return SOME_CONSTANT
```

在上面的示例中，我们将属性`MyClass.some_attribute`初始化为`SOME_CONSTANT`的值。上述代码包括以下特性：

+   通过在给定的`dict_`中设置值`SOME_CONSTANT`，我们表明这个值将被持久化到数据库中。这取代了在`Column`的默认生成器中使用`SOME_CONSTANT`的方法。在属性仪器化中给出的`active_column_defaults.py`示例演示了对于变化的默认值使用相同方法的情况，例如时间戳生成器。在这个特定的例子中，这样做并不是严格必要的，因为无论如何`SOME_CONSTANT`都会成为 INSERT 语句的一部分。

+   通过建立`retval=True`标志，我们从函数返回的值将被属性获取器返回。如果没有这个标志，事件被认为是被动观察者，我们函数的返回值将被忽略。

+   如果映射类包括继承的子类，则`propagate=True`标志是重要的，这些子类也将使用此事件监听器。如果没有这个标志，继承的子类将不使用我们的事件处理程序。

在上面的例子中，当我们将我们的值应用到给定的`dict_`时，属性设置事件`AttributeEvents.set()`以及由`validates`提供的相关验证功能都**不会**被调用。要使这些事件响应我们新生成的值，将该值应用到给定对象作为正常的属性设置操作：

```py
SOME_CONSTANT = 3.1415926

@event.listens_for(
    MyClass.some_attribute, "init_scalar",
    retval=True, propagate=True)
def _init_some_attribute(target, dict_, value):
    # will also fire off attribute set events
    target.some_attribute = SOME_CONSTANT
    return SOME_CONSTANT
```

当设置了多个监听器时，值的生成会从一个监听器“链式”传递到下一个监听器，通过将前一个指定`retval=True`的监听器返回的值作为下一个监听器的`value`参数传递。

参数：

+   `target` – 接收事件的对象实例。如果监听器注册为`raw=True`，这将是`InstanceState`对象。

+   `value` – 在此事件监听器被调用之前要返回的值。这个值最初为`None`，但是如果存在多个监听器，则将是上一个事件处理程序函数的返回值。

+   `dict_` – 此映射对象的属性字典。这通常是对象的`__dict__`，但在所有情况下都表示属性系统用于访问此属性的实际值的目标。将值放入该字典中的效果是该值将在工作单元生成的 INSERT 语句中使用。

另请参阅

`AttributeEvents.init_collection()` - 此事件的集合版本

`AttributeEvents` - 关于监听器选项的背景信息，如传播到子类。

属性仪表化 - 参见 `active_column_defaults.py` 示例。

```py
method modified(target: _O, initiator: Event) → None
```

接收一个‘修改’事件。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeClass.some_attribute, 'modified')
def receive_modified(target, initiator):
    "listen for the 'modified' event"

    # ... (event handling logic) ...
```

当使用 `flag_modified()` 函数在未设置任何特定值的情况下触发修改事件时，将触发此事件。

1.2 版中的新内容。

参数：

+   `target` – 接收事件的对象实例。如果监听器使用 `raw=True` 注册，这将是 `InstanceState` 对象。

+   `initiator` – 一个代表事件启动的 `Event` 实例。

另请参阅

`AttributeEvents` - 关于监听器选项的背景，例如传播到子类。

```py
method remove(target: _O, value: _T, initiator: Event, *, key: EventConstants = EventConstants.NO_KEY) → None
```

接收一个集合移除事件。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeClass.some_attribute, 'remove')
def receive_remove(target, value, initiator):
    "listen for the 'remove' event"

    # ... (event handling logic) ...
```

参数：

+   `target` – 接收事件的对象实例。如果监听器使用 `raw=True` 注册，这将是 `InstanceState` 对象。

+   `value` – 被移除的值。

+   `initiator` – 一个代表事件启动的 `Event` 实例。可以通过 backref 处理程序修改其原始值，以控制链式事件传播。

+   `key` –

    当使用 `AttributeEvents.include_key` 参数设置为 True 建立事件时，这将是操作中使用的键，例如 `del collection[some_key_or_index]`。如果未使用 `AttributeEvents.include_key` 设置事件，参数根本不会传递给事件；这是为了允许现有事件处理程序与不包含 `key` 参数的事件处理程序向后兼容。

    2.0 版中的新内容。

返回：

未为此事件定义返回值。

另请参阅

`AttributeEvents` - 关于监听器选项的背景，例如传播到子类。

```py
method set(target: _O, value: _T, oldvalue: _T, initiator: Event) → None
```

接收一个标量集合事件。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeClass.some_attribute, 'set')
def receive_set(target, value, oldvalue, initiator):
    "listen for the 'set' event"

    # ... (event handling logic) ...
```

参数：

+   `target` – 接收事件的对象实例。如果监听器使用 `raw=True` 注册，这将是 `InstanceState` 对象。

+   `value` – 被设置的值。如果此监听器使用 `retval=True` 注册，监听器函数必须返回此值，或替换它的新值。

+   `oldvalue` – 被替换的先前值。这也可以是符号 `NEVER_SET` 或 `NO_VALUE`。如果监听器使用 `active_history=True` 注册，当现有值当前未加载或过期时，将从数据库加载属性的先前值。

+   `initiator` – 代表事件启动的`Event`实例。可能会被 backref 处理程序从其原始值修改，以控制链式事件传播。

返回：

如果事件是以`retval=True`注册的，则应返回给定值或新的有效值。

另请参见

`AttributeEvents` - 关于侦听器选项的背景，例如传播到子类。

## 查询事件

| 对象名称 | 描述 |
| --- | --- |
| QueryEvents | 代表在构建`Query`对象时的事件。 |

```py
class sqlalchemy.orm.QueryEvents
```

代表在构建`Query`对象时的事件。

传统特性

`QueryEvents`事件方法在 SQLAlchemy 2.0 中已过时，仅适用于直接使用`Query`对象。它们不适用于 2.0 风格语句。要拦截和修改 2.0 风格 ORM 使用的事件，请使用`SessionEvents.do_orm_execute()`钩子。

`QueryEvents`钩子现在已被`SessionEvents.do_orm_execute()`事件钩子取代。

**成员**

before_compile(), before_compile_delete(), before_compile_update(), dispatch

**类签名**

类`sqlalchemy.orm.QueryEvents` (`sqlalchemy.event.Events`)

```py
method before_compile(query: Query) → None
```

在核心`Select`对象之前将`Query`对象接收到。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeQuery, 'before_compile')
def receive_before_compile(query):
    "listen for the 'before_compile' event"

    # ... (event handling logic) ...
```

自版本 1.4 起弃用：`QueryEvents.before_compile()` 事件被更强大的`SessionEvents.do_orm_execute()` 钩子所取代。在版本 1.4 中，`QueryEvents.before_compile()` 事件**不再用于**ORM 级别的属性加载，例如延迟加载或过期属性以及关系加载器的加载。请参阅 ORM 查询事件中的新示例，展示了拦截和修改 ORM 查询的新方法，最常见的目的是添加任意的过滤条件。

此事件旨在允许对查询进行更改：

```py
@event.listens_for(Query, "before_compile", retval=True)
def no_deleted(query):
    for desc in query.column_descriptions:
        if desc['type'] is User:
            entity = desc['entity']
            query = query.filter(entity.deleted == False)
    return query
```

通常应该使用`retval=True`参数监听事件，以便修改后的查询可以返回。

默认情况下，`QueryEvents.before_compile()` 事件将禁止“烘焙”查询缓存查询，如果事件钩子返回一个新的`Query` 对象。这影响了烘焙查询扩展的直接使用以及它在关系的惰性加载器和急切加载器中的操作。为了重新建立被缓存的查询，请应用添加`bake_ok`标志的事件：

```py
@event.listens_for(
    Query, "before_compile", retval=True, bake_ok=True)
def my_event(query):
    for desc in query.column_descriptions:
        if desc['type'] is User:
            entity = desc['entity']
            query = query.filter(entity.deleted == False)
    return query
```

当`bake_ok`设置为 True 时，事件钩子只会被调用一次，并且不会为正在被缓存的特定查询的后续调用而调用。

自版本 1.3.11 起新增：- 在`QueryEvents.before_compile()` 事件中添加了“bake_ok”标志，并且如果未设置此标志，则不允许通过“烘焙”扩展进行缓存的事件处理程序返回一个新的`Query` 对象。

另请参阅

`QueryEvents.before_compile_update()`

`QueryEvents.before_compile_delete()`

使用 before_compile 事件

```py
method before_compile_delete(query: Query, delete_context: BulkDelete) → None
```

允许对`Query` 对象进行修改，`Query.delete()` 内部。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeQuery, 'before_compile_delete')
def receive_before_compile_delete(query, delete_context):
    "listen for the 'before_compile_delete' event"

    # ... (event handling logic) ...
```

自版本 1.4 弃用：`QueryEvents.before_compile_delete()`事件已被功能更强大的`SessionEvents.do_orm_execute()`钩取代。

类似于`QueryEvents.before_compile()`事件，此事件应配置为`retval=True`，并返回修改后的`Query`对象，如下所示

```py
@event.listens_for(Query, "before_compile_delete", retval=True)
def no_deleted(query, delete_context):
    for desc in query.column_descriptions:
        if desc['type'] is User:
            entity = desc['entity']
            query = query.filter(entity.deleted == False)
    return query
```

参数：

+   `query` – 一个`Query`实例；这也是给定“删除上下文”对象的`.query`属性。

+   `delete_context` – 一个“删除上下文”对象，与`QueryEvents.after_bulk_delete.delete_context`中描述的对象类型相同。

新版本 1.2.17 中新增。

另请参阅

`QueryEvents.before_compile()`

`QueryEvents.before_compile_update()`

```py
method before_compile_update(query: Query, update_context: BulkUpdate) → None
```

允许在`Query.update()`内修改`Query`对象。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeQuery, 'before_compile_update')
def receive_before_compile_update(query, update_context):
    "listen for the 'before_compile_update' event"

    # ... (event handling logic) ...
```

自版本 1.4 弃用：`QueryEvents.before_compile_update()`事件已被功能更强大的`SessionEvents.do_orm_execute()`钩取代。

类似于`QueryEvents.before_compile()`事件，如果要用该事件来修改`Query`对象，则应配置为`retval=True`，并返回修改后的`Query`对象，如下所示

```py
@event.listens_for(Query, "before_compile_update", retval=True)
def no_deleted(query, update_context):
    for desc in query.column_descriptions:
        if desc['type'] is User:
            entity = desc['entity']
            query = query.filter(entity.deleted == False)

            update_context.values['timestamp'] = datetime.utcnow()
    return query
```

“更新上下文”对象的`.values`字典也可以像上面示例的那样就地修改。

参数：

+   `query` – 一个`Query`实例；这也是给定“更新上下文”对象的`.query`属性。

+   `update_context` – 一个“更新上下文”对象，它与`QueryEvents.after_bulk_update.update_context`中描述的对象相同。对象具有在 UPDATE 上下文中的`.values`属性，该属性是传递给`Query.update()`的参数字典。可以修改此字典以更改生成的 UPDATE 语句的 VALUES 子句。

1.2.17 版中的新内容。

另请参阅

`QueryEvents.before_compile()`

`QueryEvents.before_compile_delete()`

```py
attribute dispatch: _Dispatch[_ET] = <sqlalchemy.event.base.QueryEventsDispatch object>
```

参考回到 _Dispatch 类。

双向针对 _Dispatch._events

## 仪器化事件

定义了 SQLAlchemy 的类仪器化系统。

这个模块通常对用户应用程序不直接可见，但定义了 ORM 交互的大部分内容。

instrumentation.py 处理了终端用户类的注册以进行状态跟踪。它与分别建立了每个实例和每个类属性仪器化的 state.py 和 attributes.py 密切交互。

类的仪器化系统可以使用`sqlalchemy.ext.instrumentation`模块进行每个类或全局基础上的定制化，该模块提供了构建和指定替代仪器化形式的方法。

| 对象名称 | 描述 |
| --- | --- |
| InstrumentationEvents | 与类仪器化事件相关的事件。 |

```py
class sqlalchemy.orm.InstrumentationEvents
```

与类仪器化事件相关的事件。

这里的监听器支持对任何新风格类进行建立，即任何‘type’的子类对象。然后将为针对该类的事件触发事件。如果传递了“propagate=True”标志给 event.listen()，则该事件也将为该类的子类触发。

Python 的 `type` 内建函数也被接受为目标，当使用时，将对所有类发出事件。

请注意，此处的“propagate”标志默认为 `True`，与其他类级别事件不同，后者的默认值为 `False`。这意味着当在超类上建立侦听器时，新的子类也将成为这些事件的主题。

**成员**

attribute_instrument(), class_instrument(), class_uninstrument(), dispatch

**类签名**

类`sqlalchemy.orm.InstrumentationEvents` (`sqlalchemy.event.Events`)

```py
method attribute_instrument(cls: ClassManager[_O], key: _KT, inst: _O) → None
```

当属性被仪器化时调用。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeBaseClass, 'attribute_instrument')
def receive_attribute_instrument(cls, key, inst):
    "listen for the 'attribute_instrument' event"

    # ... (event handling logic) ...
```

```py
method class_instrument(cls: ClassManager[_O]) → None
```

在给定类被仪器化之后调用。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeBaseClass, 'class_instrument')
def receive_class_instrument(cls):
    "listen for the 'class_instrument' event"

    # ... (event handling logic) ...
```

要获取`ClassManager`，请使用`manager_of_class()`。

```py
method class_uninstrument(cls: ClassManager[_O]) → None
```

在给定类被取消仪器化之前调用。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeBaseClass, 'class_uninstrument')
def receive_class_uninstrument(cls):
    "listen for the 'class_uninstrument' event"

    # ... (event handling logic) ...
```

要获取`ClassManager`，请使用`manager_of_class()`。

```py
attribute dispatch: _Dispatch[_ET] = <sqlalchemy.event.base.InstrumentationEventsDispatch object>
```

参考 _Dispatch 类。

双向对 _Dispatch._events

## 会话事件

最基本的事件钩子可在 ORM `Session`对象级别使用。在此拦截的内容包括：

+   **持久化操作** - 将更改发送到数据库的 ORM 刷新过程可以使用在刷新的不同部分触发的事件进行扩展，以增强或修改发送到数据库的数据，或者在持久化发生时允许其他事情发生。在持久化事件中了解更多信息。

+   **对象生命周期事件** - 当对象从会话中添加、持久化、删除时触发的钩子。在对象生命周期事件中了解更多信息。

+   **执行事件** - 作为 2.0 风格执行模型的一部分，针对 ORM 实体的所有 SELECT 语句以及刷新过程之外的批量 UPDATE 和 DELETE 语句都会被拦截，使用`Session.execute()`方法，并通过`SessionEvents.do_orm_execute()`方法。在执行事件中了解更多信息。

请务必阅读使用事件跟踪查询、对象和会话更改章节，以了解这些事件的背景。

| 对象名称 | 描述 |
| --- | --- |
| SessionEvents | 定义特定于`Session`生命周期的事件。 |

```py
class sqlalchemy.orm.SessionEvents
```

定义特定于`Session`生命周期的事件。

例如：

```py
from sqlalchemy import event
from sqlalchemy.orm import sessionmaker

def my_before_commit(session):
    print("before commit!")

Session = sessionmaker()

event.listen(Session, "before_commit", my_before_commit)
```

`listen()`函数将接受`Session`对象，以及`sessionmaker()`和`scoped_session()`的返回结果。

此外，它接受`Session`类，将全局应用监听器到所有`Session`实例。

参数：

+   `raw=False` –

    当为 True 时，传递给适用于单个对象的事件监听器函数的“target”参数将是实例的`InstanceState`管理对象，而不是映射实例本身。

    新版本 1.3.14 中新增。

+   `restore_load_context=False` –

    适用于`SessionEvents.loaded_as_persistent()`事件。在事件钩子完成时恢复对象的加载器上下文，以便持续的急切加载操作继续适当地针对对象。如果在此事件中将对象移动到新的加载器上下文而未设置此标志，则会发出警告。

    新版本 1.3.14 中新增。

**成员**

after_attach(), after_begin(), after_bulk_delete(), after_bulk_update(), after_commit(), after_flush(), after_flush_postexec(), after_rollback(), after_soft_rollback(), after_transaction_create(), after_transaction_end(), before_attach(), before_commit(), before_flush(), deleted_to_detached(), deleted_to_persistent(), detached_to_persistent(), dispatch, do_orm_execute(), loaded_as_persistent(), pending_to_persistent(), pending_to_transient(), persistent_to_deleted(), persistent_to_detached(), persistent_to_transient(), transient_to_pending()

**类签名**

类`sqlalchemy.orm.SessionEvents`（`sqlalchemy.event.Events`）

```py
method after_attach(session: Session, instance: _O) → None
```

在实例被附加到会话之后执行。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeSessionClassOrObject, 'after_attach')
def receive_after_attach(session, instance):
    "listen for the 'after_attach' event"

    # ... (event handling logic) ...
```

在添加、删除或合并后调用。

注意

自 0.8 版开始，此事件在项目完全与会话相关联之后触发，这与之前的版本不同。对于需要对象尚未成为会话状态的事件处理程序（例如，当目标对象尚未完全完成时可能自动刷新的处理程序），请考虑使用新的`before_attach()`事件。

另请参见

`SessionEvents.before_attach()`

对象生命周期事件

```py
method after_begin(session: Session, transaction: SessionTransaction, connection: Connection) → None
```

在连接上启动事务后执行。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeSessionClassOrObject, 'after_begin')
def receive_after_begin(session, transaction, connection):
    "listen for the 'after_begin' event"

    # ... (event handling logic) ...
```

注意

此事件在`Session`修改其自身内部状态的过程中调用。在此挂钩内调用 SQL 操作，请使用事件提供的`Connection`；不要直接使用`Session`运行 SQL 操作。

参数：

+   `session` – 目标`Session`。

+   `transaction` – `SessionTransaction`。

+   `connection` – 将用于 SQL 语句的`Connection`对象。

另请参见

`SessionEvents.before_commit()`

`SessionEvents.after_commit()`

`SessionEvents.after_transaction_create()`

`SessionEvents.after_transaction_end()`

```py
method after_bulk_delete(delete_context: _O) → None
```

当传统的`Query.delete()`方法被调用后的事件。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeSessionClassOrObject, 'after_bulk_delete')
def receive_after_bulk_delete(delete_context):
    "listen for the 'after_bulk_delete' event"

    # ... (event handling logic) ...

# DEPRECATED calling style (pre-0.9, will be removed in a future release)
@event.listens_for(SomeSessionClassOrObject, 'after_bulk_delete')
def receive_after_bulk_delete(session, query, query_context, result):
    "listen for the 'after_bulk_delete' event"

    # ... (event handling logic) ...
```

从版本 0.9 开始更改：`SessionEvents.after_bulk_delete()`事件现在接受参数`SessionEvents.after_bulk_delete.delete_context`。将来版本中将删除接受上述“已弃用”的先前参数签名的侦听器函数的支持。

旧特性

从 SQLAlchemy 2.0 开始，`SessionEvents.after_bulk_delete()`方法是一个旧的事件钩子。该事件**不参与**使用`delete()`在 ORM UPDATE and DELETE with Custom WHERE Criteria 中记录的 2.0 风格调用。对于 2.0 风格的使用，`SessionEvents.do_orm_execute()`钩子将拦截这些调用。

参数：

**delete_context** -

一个包含有关更新的“删除上下文”对象，包括以下属性：

> +   `session` - 参与的`Session`。
> +   
> +   `query` - 调用此更新操作的`Query`对象。
> +   
> +   `result` - 作为批量删除操作的结果返回的`CursorResult`。

从版本 1.4 开始：update_context 不再与`QueryContext`对象相关联。

另请参阅

`QueryEvents.before_compile_delete()`

`SessionEvents.after_bulk_update()`

```py
method after_bulk_update(update_context: _O) → None
```

用于在调用旧的`Query.update()`方法之后触发事件。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeSessionClassOrObject, 'after_bulk_update')
def receive_after_bulk_update(update_context):
    "listen for the 'after_bulk_update' event"

    # ... (event handling logic) ...

# DEPRECATED calling style (pre-0.9, will be removed in a future release)
@event.listens_for(SomeSessionClassOrObject, 'after_bulk_update')
def receive_after_bulk_update(session, query, query_context, result):
    "listen for the 'after_bulk_update' event"

    # ... (event handling logic) ...
```

从版本 0.9 开始更改：`SessionEvents.after_bulk_update()`事件现在接受参数`SessionEvents.after_bulk_update.update_context`。将来版本中将删除接受上述“已弃用”的先前参数签名的侦听器函数的支持。

旧特性

`SessionEvents.after_bulk_update()` 方法是 SQLAlchemy 2.0 中的一个旧式事件钩子。此事件**不参与**使用 `update()` 进行 2.0 风格调用的文档化操作。要使用 2.0 风格，`SessionEvents.do_orm_execute()` 钩子将拦截这些调用。

参数：

**update_context** -

包含关于更新的“更新上下文”对象，包括这些属性：

> +   `session` - 涉及的`Session`。
> +   
> +   `query` - 调用此更新操作的`Query` 对象。
> +   
> +   `values` - 传递给`Query.update()` 的`values`字典。
> +   
> +   `result` - 作为批量更新操作的结果返回的`CursorResult`。

从版本 1.4 起更改：update_context 不再与`QueryContext`对象关联。

另请参阅

`QueryEvents.before_compile_update()`

`SessionEvents.after_bulk_delete()`

```py
method after_commit(session: Session) → None
```

在提交之后执行。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeSessionClassOrObject, 'after_commit')
def receive_after_commit(session):
    "listen for the 'after_commit' event"

    # ... (event handling logic) ...
```

注

`SessionEvents.after_commit()`钩子不是每次刷新都执行的，也就是说，在事务的范围内，`Session` 可以多次向数据库发出 SQL。要拦截这些事件，可以使用 `SessionEvents.before_flush()`、`SessionEvents.after_flush()` 或 `SessionEvents.after_flush_postexec()` 事件。

注

当 `SessionEvents.after_commit()` 事件被调用时，`Session` 处于非活动事务状态，因此无法发出 SQL。要发出与每个事务对应的 SQL，请使用 `SessionEvents.before_commit()` 事件。

参数：

**session** – 目标 `Session`。

请参见

`SessionEvents.before_commit()`

`SessionEvents.after_begin()`

`SessionEvents.after_transaction_create()`

`SessionEvents.after_transaction_end()`

```py
method after_flush(session: Session, flush_context: UOWTransaction) → None
```

在 flush 完成后执行，但在提交之前调用。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeSessionClassOrObject, 'after_flush')
def receive_after_flush(session, flush_context):
    "listen for the 'after_flush' event"

    # ... (event handling logic) ...
```

注意，会话的状态仍然处于 pre-flush 状态，即'new'、'dirty'和'deleted'列表仍然显示 pre-flush 状态以及实例属性上的历史设置。

警告

此事件在 `Session` 发出 SQL 修改数据库之后运行，但在它修改内部状态以反映这些更改之前运行，包括将新插入的对象放入标识映射中。在此事件内发出的 ORM 操作（如加载相关项目）可能会产生新的标识映射条目，这些条目将立即被替换，有时会导致混淆的结果。从版本 1.3.9 起，SQLAlchemy 会对此条件发出警告。

参数：

+   `session` – 目标 `Session`。

+   `flush_context` – 处理 flush 细节的内部 `UOWTransaction` 对象。

请参见

`SessionEvents.before_flush()`

`SessionEvents.after_flush_postexec()`

持久性事件

```py
method after_flush_postexec(session: Session, flush_context: UOWTransaction) → None
```

在 flush 完成后执行，并在 post-exec 状态发生后执行。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeSessionClassOrObject, 'after_flush_postexec')
def receive_after_flush_postexec(session, flush_context):
    "listen for the 'after_flush_postexec' event"

    # ... (event handling logic) ...
```

这将是'new'、'dirty'和'deleted'列表处于最终状态的时候。实际的 commit() 可能已经发生，也可能没有发生，这取决于 flush 是否启动了自己的事务或者参与了更大的事务。

参数：

+   `session` – 目标`Session`。

+   `flush_context` – 处理刷新细节的内部`UOWTransaction`对象。

另请参见

`SessionEvents.before_flush()`

`SessionEvents.after_flush()`

持久性事件

```py
method after_rollback(session: Session) → None
```

在实际发生 DBAPI 回滚后执行。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeSessionClassOrObject, 'after_rollback')
def receive_after_rollback(session):
    "listen for the 'after_rollback' event"

    # ... (event handling logic) ...
```

请注意，此事件仅在实际对数据库执行回滚时触发 - 如果底层 DBAPI 事务已经被回滚，则不会每次调用`Session.rollback()`方法时都触发。在许多情况下，`Session`在此事件期间将不处于“活动”状态，因为当前事务无效。要在最外层回滚进行后获取一个活动的`Session`，请使用`SessionEvents.after_soft_rollback()`事件，并检查`Session.is_active`标志。

参数：

**session** – 目标`Session`。

```py
method after_soft_rollback(session: Session, previous_transaction: SessionTransaction) → None
```

在发生任何回滚后执行，包括“软”回滚，这种回滚在 DBAPI 级别实际上不会发出。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeSessionClassOrObject, 'after_soft_rollback')
def receive_after_soft_rollback(session, previous_transaction):
    "listen for the 'after_soft_rollback' event"

    # ... (event handling logic) ...
```

这对应于嵌套和外部回滚，即调用 DBAPI 的 rollback()方法的最内部回滚，以及仅从事务堆栈中弹出自身的封闭回滚调用。

给定的`Session`可以用于在外部回滚后通过首先检查`Session.is_active`标志来调用 SQL 和`Session.query()`操作：

```py
@event.listens_for(Session, "after_soft_rollback")
def do_something(session, previous_transaction):
    if session.is_active:
        session.execute(text("select * from some_table"))
```

参数：

+   `session` – 目标`Session`。

+   `previous_transaction` – 刚刚关闭的`SessionTransaction`事务标记对象。给定`Session`的当前`SessionTransaction`可以通过`Session.transaction`属性获得。

```py
method after_transaction_create(session: Session, transaction: SessionTransaction) → None
```

当创建新的`SessionTransaction`时执行。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeSessionClassOrObject, 'after_transaction_create')
def receive_after_transaction_create(session, transaction):
    "listen for the 'after_transaction_create' event"

    # ... (event handling logic) ...
```

此事件与`SessionEvents.after_begin()`不同，因为它针对每个`SessionTransaction`总体发生，而不是在个别数据库连接上开始事务时发生。它还用于嵌套事务和子事务，并始终与相应的`SessionEvents.after_transaction_end()`事件匹配（假设`Session`正常运行）。

参数：

+   `session` – 目标`Session`。

+   `transaction` –

    目标`SessionTransaction`。

    要检测此是否为最外层`SessionTransaction`，而不是“子事务”或 SAVEPOINT，请测试`SessionTransaction.parent`属性是否为`None`：

    ```py
    @event.listens_for(session, "after_transaction_create")
    def after_transaction_create(session, transaction):
        if transaction.parent is None:
            # work with top-level transaction
    ```

    要检测`SessionTransaction`是否为 SAVEPOINT，请使用`SessionTransaction.nested`属性：

    ```py
    @event.listens_for(session, "after_transaction_create")
    def after_transaction_create(session, transaction):
        if transaction.nested:
            # work with SAVEPOINT transaction
    ```

另请参阅

`SessionTransaction`

`SessionEvents.after_transaction_end()`

```py
method after_transaction_end(session: Session, transaction: SessionTransaction) → None
```

当`SessionTransaction`的跨度结束时执行。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeSessionClassOrObject, 'after_transaction_end')
def receive_after_transaction_end(session, transaction):
    "listen for the 'after_transaction_end' event"

    # ... (event handling logic) ...
```

此事件与`SessionEvents.after_commit()`不同，它对应于所有正在使用的`SessionTransaction`对象，包括嵌套事务和子事务，并且始终与相应的`SessionEvents.after_transaction_create()`事件匹配。

参数：

+   `session` – 目标`Session`。

+   `transaction` –

    目标`SessionTransaction`。

    要检测是否为最外层的`SessionTransaction`，而不是“子事务”或 SAVEPOINT，请测试`SessionTransaction.parent`属性是否为`None`：

    ```py
    @event.listens_for(session, "after_transaction_create")
    def after_transaction_end(session, transaction):
        if transaction.parent is None:
            # work with top-level transaction
    ```

    要检测`SessionTransaction`是否为 SAVEPOINT，请使用`SessionTransaction.nested`属性：

    ```py
    @event.listens_for(session, "after_transaction_create")
    def after_transaction_end(session, transaction):
        if transaction.nested:
            # work with SAVEPOINT transaction
    ```

另请参阅

`SessionTransaction`

`SessionEvents.after_transaction_create()`

```py
method before_attach(session: Session, instance: _O) → None
```

在实例附加到会话之前执行。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeSessionClassOrObject, 'before_attach')
def receive_before_attach(session, instance):
    "listen for the 'before_attach' event"

    # ... (event handling logic) ...
```

在添加、删除或合并导致对象成为会话的一部分之前调用此方法。

另请参阅

`SessionEvents.after_attach()`

对象生命周期事件

```py
method before_commit(session: Session) → None
```

在调用提交之前执行。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeSessionClassOrObject, 'before_commit')
def receive_before_commit(session):
    "listen for the 'before_commit' event"

    # ... (event handling logic) ...
```

注意

`SessionEvents.before_commit()`挂钩*不*是每次刷新的，也就是说，在事务范围内，`Session`可以多次向数据库发出 SQL。要拦截这些事件，请使用`SessionEvents.before_flush()`、`SessionEvents.after_flush()`或`SessionEvents.after_flush_postexec()`事件。

参数：

**session** – 目标`Session`。

另请参阅

`SessionEvents.after_commit()`

`SessionEvents.after_begin()`

`SessionEvents.after_transaction_create()`

`SessionEvents.after_transaction_end()`

```py
method before_flush(session: Session, flush_context: UOWTransaction, instances: Sequence[_O] | None) → None
```

在刷新过程开始之前执��。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeSessionClassOrObject, 'before_flush')
def receive_before_flush(session, flush_context, instances):
    "listen for the 'before_flush' event"

    # ... (event handling logic) ...
```

参数：

+   `session` – 目标`Session`。

+   `flush_context` – 处理刷新细节的内部`UOWTransaction`对象。

+   `instances` – 通常为`None`，这是可以传递给`Session.flush()`方法的对象集合（请注意，此用法已被弃用）。

另请参阅

`SessionEvents.after_flush()`

`SessionEvents.after_flush_postexec()`

持久性事件

```py
method deleted_to_detached(session: Session, instance: _O) → None
```

拦截特定对象的“删除到分离”转换。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeSessionClassOrObject, 'deleted_to_detached')
def receive_deleted_to_detached(session, instance):
    "listen for the 'deleted_to_detached' event"

    # ... (event handling logic) ...
```

当从会话中删除的对象被驱逐时，将调用此事件。典型情况是当删除对象的会话的事务被提交时发生；对象从删除状态移动到分离状态。

还会为在调用`Session.expunge_all()`或`Session.close()`事件时被删除的对象调用，以及如果对象通过`Session.expunge()`从其删除状态单独驱逐。

另请参阅

对象生命周期事件

```py
method deleted_to_persistent(session: Session, instance: _O) → None
```

拦截特定对象的“删除到持久”转换。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeSessionClassOrObject, 'deleted_to_persistent')
def receive_deleted_to_persistent(session, instance):
    "listen for the 'deleted_to_persistent' event"

    # ... (event handling logic) ...
```

仅当在刷新中成功删除的对象由于调用`Session.rollback()`而被恢复时，才会发生此转换。在任何其他情况下不会调用该事件。

另请参阅

对象生命周期事件

```py
method detached_to_persistent(session: Session, instance: _O) → None
```

拦截特定对象的“分离到持久化”转换。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeSessionClassOrObject, 'detached_to_persistent')
def receive_detached_to_persistent(session, instance):
    "listen for the 'detached_to_persistent' event"

    # ... (event handling logic) ...
```

此事件是 `SessionEvents.after_attach()` 事件的一个特化，仅针对此特定转换调用。它通常在 `Session.add()` 调用期间调用，以及在对象之前未与 `Session` 关联的情况下，在 `Session.delete()` 调用期间调用（请注意，标记为“已删除”的对象在刷新之前仍处于“持久化”状态）。

注意

如果对象在调用 `Session.delete()` 时变为持久化对象，则在调用此事件时对象尚未标记为已删除。要检测已删除的对象，请在刷新后检查发送到 `SessionEvents.persistent_to_detached()` 事件的 `deleted` 标志，或者在刷新之前需要拦截已删除对象时，在 `SessionEvents.before_flush()` 事件中检查 `Session.deleted` 集合。

参数：

+   `session` – 目标 `Session`

+   `instance` – 正在操作的 ORM 映射实例。

参见

对象生命周期事件

```py
attribute dispatch: _Dispatch[_ET] = <sqlalchemy.event.base.SessionEventsDispatch object>
```

参考回到 _Dispatch 类。

双向对抗 _Dispatch._events

```py
method do_orm_execute(orm_execute_state: ORMExecuteState) → None
```

拦截代表 ORM `Session` 对象执行的语句。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeSessionClassOrObject, 'do_orm_execute')
def receive_do_orm_execute(orm_execute_state):
    "listen for the 'do_orm_execute' event"

    # ... (event handling logic) ...
```

此事件被调用用于从`Session.execute()`方法调用的所有顶级 SQL 语句，以及相关方法，如`Session.scalars()`和`Session.scalar()`。从 SQLAlchemy 1.4 开始，所有通过`Session.execute()`方法运行的 ORM 查询以及相关方法`Session.scalars()`、`Session.scalar()`等都将参与此事件。此事件挂钩**不适用于**在 ORM 刷新过程内部发出的查询，即在刷新中描述的过程。

注意

`SessionEvents.do_orm_execute()`事件挂钩仅**针对 ORM 语句执行**触发，即通过`Session.execute()`和类似方法在`Session`对象上调用的语句。它**不**会触发仅由 SQLAlchemy Core 调用的语句，即仅通过`Connection.execute()`直接调用的语句或从不涉及任何`Session`的`Engine`对象发出的语句。要拦截**所有**SQL 执行，无论是否使用 Core 或 ORM API，请参见`ConnectionEvents`中的事件挂钩，如`ConnectionEvents.before_execute()`和`ConnectionEvents.before_cursor_execute()`。

此事件挂钩**不适用于**在 ORM 刷新过程内部发出的查询，即在刷新中描述的过程；要拦截刷新过程中的步骤，请参见持久性事件以及映射器级刷新事件中描述的事件挂钩。

此事件是一个`do_`事件，意味着它具有替换`Session.execute()`方法通常执行的操作的能力。其预期用途包括分片和结果缓存方案，这些方案可能希望在多个数据库连接上调用相同的语句，返回从每个连接合并的结果，或者根本不调用语句，而是从缓存返回数据。

该钩子旨在取代在 SQLAlchemy 1.4 之前可以被子类化的`Query._execute_and_instances`方法。

参数：

**orm_execute_state** - 一个`ORMExecuteState`的实例，其中包含有关当前执行的所有信息，以及用于推导其他常用信息的辅助函数。有关详细信息，请参阅该对象。

另请参阅

执行事件 - 关于如何使用`SessionEvents.do_orm_execute()`的顶级文档

`ORMExecuteState` - 传递给`SessionEvents.do_orm_execute()`事件的对象，其中包含有关要调用的语句的所有信息。它还提供了一个接口来扩展当前语句、选项和参数，以及一个选项，允许在任何时候以编程方式调用语句。

ORM 查询事件 - 包括使用`SessionEvents.do_orm_execute()`的示例

Dogpile 缓存 - 一个示例，演示如何将 Dogpile 缓存与 ORM `Session`集成，利用`SessionEvents.do_orm_execute()`事件钩子。

水平分片 - 水平分片示例/扩展依赖于`SessionEvents.do_orm_execute()`事件钩子，在多个后端上调用 SQL 语句并返回合并结果。

1.4 版本中的新功能。

```py
method loaded_as_persistent(session: Session, instance: _O) → None
```

拦截特定对象的“加载为持久性”转换。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeSessionClassOrObject, 'loaded_as_persistent')
def receive_loaded_as_persistent(session, instance):
    "listen for the 'loaded_as_persistent' event"

    # ... (event handling logic) ...
```

此事件在 ORM 加载过程中被调用，与`InstanceEvents.load()`事件非常相似。然而，这里的事件可以链接到`Session`类或实例，而不是映射器或类层次结构，并且与其他会话生命周期事件平滑集成。在调用此事件时，对象保证存在于会话的标识映射中。

注意

此事件在加载器过程中被调用，可能在急加载器完成之前，对象的状态可能不完整。此外，在对象上调用行级刷新操作将使对象进入新的加载器上下文，干扰现有的加载上下文。有关如何使用`SessionEvents.restore_load_context`参数的背景，请参阅有关使用`InstanceEvents.restore_load_context`的说明，以解决此场景。

参数：

+   `session` – 目标`Session`

+   `instance` – 正在操作的 ORM 映射实例。

另请参阅

对象生命周期事件

```py
method pending_to_persistent(session: Session, instance: _O) → None
```

拦截特定对象的“挂起到持久”转换。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeSessionClassOrObject, 'pending_to_persistent')
def receive_pending_to_persistent(session, instance):
    "listen for the 'pending_to_persistent' event"

    # ... (event handling logic) ...
```

此事件在刷新过程中被调用，类似于在`SessionEvents.after_flush()`事件中扫描`Session.new`集合。然而，在这种情况下，当调用事件时，对象已经被移动到持久状态。 

参数：

+   `session` – 目标`Session`

+   `instance` – 正在操作的 ORM 映射实例。

另请参阅

对象生命周期事件

```py
method pending_to_transient(session: Session, instance: _O) → None
```

拦截特定对象的“挂起到瞬态”转换。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeSessionClassOrObject, 'pending_to_transient')
def receive_pending_to_transient(session, instance):
    "listen for the 'pending_to_transient' event"

    # ... (event handling logic) ...
```

当未刷新的挂起对象从会话中驱逐时，会发生这种较少见的转换；这可能发生在`Session.rollback()`方法回滚事务时，或者在使用`Session.expunge()`方法时。

参数：

+   `session` – 目标`Session`

+   `instance` – 正在操作的 ORM 映射实例。

另请参阅

对象生命周期事件

```py
method persistent_to_deleted(session: Session, instance: _O) → None
```

拦截特定对象的“持久到已删除”转换。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeSessionClassOrObject, 'persistent_to_deleted')
def receive_persistent_to_deleted(session, instance):
    "listen for the 'persistent_to_deleted' event"

    # ... (event handling logic) ...
```

当持久对象的标识在刷新中从数据库中删除时，将调用此事件，但是对象仍然与`Session`关联，直到事务完成。

如果事务被回滚，则对象再次移动到持久状态，并调用`SessionEvents.deleted_to_persistent()`事件。如果事务被提交，则对象变为分离状态，这将触发`SessionEvents.deleted_to_detached()`事件。

请注意，虽然`Session.delete()`方法是标记对象为已删除的主要公共接口，但许多对象由于级联规则而被删除，这些规则直到刷新时才确定。因此，在刷新进行之前，没有办法捕获每个将被删除的对象。因此，在刷新结束时调用`SessionEvents.persistent_to_deleted()`事件。 

另请参阅

对象生命周期事件

```py
method persistent_to_detached(session: Session, instance: _O) → None
```

拦截特定对象的“持久到分离”转换。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeSessionClassOrObject, 'persistent_to_detached')
def receive_persistent_to_detached(session, instance):
    "listen for the 'persistent_to_detached' event"

    # ... (event handling logic) ...
```

当持久对象从会话中驱逐时，将调用此事件。导致此事件发生的许多条件，包括：

+   使用`Session.expunge()`或`Session.close()`等方法

+   当对象是该会话事务的 INSERT 语句的一部分时，调用`Session.rollback()`方法

参数：

+   `session` – 目标`Session`

+   `instance` – 正在操作的 ORM 映射实例。

+   `deleted` – 布尔值。如果为 True，则表示此对象因被标记为已删除并刷新而移动到分离状态。

另请参阅

对象生命周期事件

```py
method persistent_to_transient(session: Session, instance: _O) → None
```

拦截特定对象的“持久到瞬时”转换。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeSessionClassOrObject, 'persistent_to_transient')
def receive_persistent_to_transient(session, instance):
    "listen for the 'persistent_to_transient' event"

    # ... (event handling logic) ...
```

这种较不常见的转换发生在已刷新的挂起对象从会话中被驱逐时；当`Session.rollback()`方法回滚事务时，这种情况可能发生。

参数：

+   `session` – 目标`Session`

+   `instance` – 正在操作的 ORM 映射实例。

另请参阅

对象生命周期事件

```py
method transient_to_pending(session: Session, instance: _O) → None
```

拦截特定对象的“瞬态到挂起”转换。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeSessionClassOrObject, 'transient_to_pending')
def receive_transient_to_pending(session, instance):
    "listen for the 'transient_to_pending' event"

    # ... (event handling logic) ...
```

这个事件是`SessionEvents.after_attach()`事件的一个特例，仅在这个特定的转换中调用。通常在`Session.add()`调用期间调用。

参数：

+   `session` – 目标`Session`

+   `instance` – 正在操作的 ORM 映射实例。

另请参阅

对象生命周期事件

## 映射器事件

映射器事件钩子涵盖了与单个或多个`Mapper`对象相关的事情，这些对象是将用户定义的类映射到`Table`对象的中心配置对象。在`Mapper`级别发生的事情包括：

+   **每个对象的持久化操作** - 最常见的映射器钩子是工作单元钩子，如`MapperEvents.before_insert()`、`MapperEvents.after_update()`等。这些事件与更粗粒度的会话级事件形成对比，如`SessionEvents.before_flush()`，因为它们在每个对象的刷新过程中发生；虽然对象上的更细粒度活动更直接，但`Session`功能的可用性有限。

+   **映射器配置事件** - 另一类重要的映射器钩子是在类被映射时、映射器被最终化时以及当映射器集合被配置为相互引用时发生的事件。这些事件包括 `MapperEvents.instrument_class()`、`MapperEvents.before_mapper_configured()` 和 `MapperEvents.mapper_configured()` 在单个 `Mapper` 级别，以及 `MapperEvents.before_configured()` 和 `MapperEvents.after_configured()` 在集合的 `Mapper` 对象级别。

| 对象名称 | 描述 |
| --- | --- |
| MapperEvents | 定义特定于映射的事件。 |

```py
class sqlalchemy.orm.MapperEvents
```

定义特定于映射的事件。

例如：

```py
from sqlalchemy import event

def my_before_insert_listener(mapper, connection, target):
    # execute a stored procedure upon INSERT,
    # apply the value to the row to be inserted
    target.calculated_value = connection.execute(
        text("select my_special_function(%d)" % target.special_number)
    ).scalar()

# associate the listener function with SomeClass,
# to execute during the "before_insert" hook
event.listen(
    SomeClass, 'before_insert', my_before_insert_listener)
```

可用的目标包括：

+   映射的类

+   映射或待映射类的未映射超类（使用 `propagate=True` 标志）

+   `Mapper` 对象

+   `Mapper` 类本身表示监听所有映射器。

Mapper 事件提供对映射器关键部分的钩子，包括与对象工具化、对象加载和对象持久化相关的部分。特别是，持久化方法 `MapperEvents.before_insert()` 和 `MapperEvents.before_update()` 是增强正在持久化的状态的流行位置 - 但是，这些方法在几个重要限制下运作。鼓励用户评估 `SessionEvents.before_flush()` 和 `SessionEvents.after_flush()` 方法，作为在刷新期间应用额外数据库状态的更灵活和用户友好的钩子。

当使用 `MapperEvents` 时，`listen()` 函数提供了几个修饰符。

参数：

+   `propagate=False` – 当为 True 时，事件监听器应用于所有继承映射器和/或继承类的映射器，以及任何作为此监听器目标的映射器。

+   `raw=False` – 当为 True 时，传递给适用的事件监听器函数的“target”参数将是实例的`InstanceState`管理对象，而不是映射的实例本身。

+   `retval=False` –

    当为 True 时，用户定义的事件函数必须有一个返回值，其目的是要么控制后续事件的传播，要么通过映射器以其他方式修改正在进行的操作。可能的返回值包括：

    +   `sqlalchemy.orm.interfaces.EXT_CONTINUE` - 继续正常事件处理。

    +   `sqlalchemy.orm.interfaces.EXT_STOP` - 取消链中所有后续事件处理程序。

    +   其他值 - 特定监听器指定的返回值。

**成员**

after_configured(), after_delete(), after_insert(), after_mapper_constructed(), after_update(), before_configured(), before_delete(), before_insert(), before_mapper_configured(), before_update(), dispatch, instrument_class(), mapper_configured()

**类签名**

类 `sqlalchemy.orm.MapperEvents` (`sqlalchemy.event.Events`)

```py
method after_configured() → None
```

在一系列映射器被配置后调用。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeClass, 'after_configured')
def receive_after_configured():
    "listen for the 'after_configured' event"

    # ... (event handling logic) ...
```

每次调用 `configure_mappers()` 函数完成其工作后，都会调用 `MapperEvents.after_configured()` 事件。通常在首次使用映射时自动调用 `configure_mappers()`，以及每当新映射器可用并检测到新的映射器使用时。

将此事件与`MapperEvents.mapper_configured()`事件进行对比，该事件在配置操作进行时基于每个映射器调用；与该事件不同，当调用此事件时，所有交叉配置（例如反向引用）也将对任何待定映射器可用。还与`MapperEvents.before_configured()`进行对比，该事件在系列映射器配置之前调用。

此事件**只能**应用于`Mapper`类，而不能应用于单个映射或映射类。它仅对所有映射作为一个整体调用：

```py
from sqlalchemy.orm import Mapper

@event.listens_for(Mapper, "after_configured")
def go():
    # ...
```

理论上，这个事件在每个应用程序中只调用一次，但实际上在任何新映射器受到`configure_mappers()`调用时都会被调用。如果在已经使用现有映射后构造了新映射，则可能会再次调用此事件。要确保特定事件仅被调用一次且不再调用，可以应用`once=True`参数（0.9.4 中新增）：

```py
from sqlalchemy.orm import mapper

@event.listens_for(mapper, "after_configured", once=True)
def go():
    # ...
```

另请参阅

`MapperEvents.before_mapper_configured()`

`MapperEvents.mapper_configured()`

`MapperEvents.before_configured()`

```py
method after_delete(mapper: Mapper[_O], connection: Connection, target: _O) → None
```

在发出对应于该实例的 DELETE 语句后接收对象实例。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeClass, 'after_delete')
def receive_after_delete(mapper, connection, target):
    "listen for the 'after_delete' event"

    # ... (event handling logic) ...
```

注意

此事件**仅**适用于会话刷新操作，并且**不**适用于在 ORM 启用的 INSERT、UPDATE 和 DELETE 语句中描述的 ORM DML 操作。要拦截 ORM DML 事件，请使用`SessionEvents.do_orm_execute()`。

此事件用于在给定连接上发出额外的 SQL 语句，以及执行与删除事件相关的应用程序特定的簿记。

该事件通常在之前的步骤中一次发出多个相同类的对象的 DELETE 语句后调用。

警告

Mapper 级别的刷新事件仅允许对仅限于操作的行的本地属性进行**非常有限的操作**，同时允许在给定的`Connection`上发出任何 SQL。**请完全阅读**Mapper 级别的刷新事件中关于使用这些方法的指南；通常，应优先使用`SessionEvents.before_flush()`方法进行一般的刷新更改。

参数：

+   `mapper` – 此事件的目标`Mapper`。

+   `connection` – 用于发出此实例的 DELETE 语句的`Connection`。这为当前事务提供了一个处理该实例特定于目标数据库的句柄。

+   `target` – 正在删除的映射实例。如果事件配置为`raw=True`，则这将是与实例关联的`InstanceState`状态管理对象。

返回：

此事件不支持返回值。

另请参阅

持久性事件

```py
method after_insert(mapper: Mapper[_O], connection: Connection, target: _O) → None
```

在发出与该实例对应的 INSERT 语句后接收一个对象实例。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeClass, 'after_insert')
def receive_after_insert(mapper, connection, target):
    "listen for the 'after_insert' event"

    # ... (event handling logic) ...
```

注意

此事件**仅**适用于会话刷新操作，**不**适用于描述在 ORM-启用的 INSERT、UPDATE 和 DELETE 语句中的 ORM DML 操作。要拦截 ORM DML 事件，请使用`SessionEvents.do_orm_execute()`。

此事件用于修改实例发生 INSERT 后的仅在 Python 中的状态，以及在给定连接上发出附加的 SQL 语句。

该事件通常在一批相同类的对象的 INSERT 语句一次性发出后被调用。在极为罕见的情况下，如果这不是理想的情况，`Mapper`对象可以配置为`batch=False`，这将导致实例批次被拆分为单个（性能较差）事件->持久化->事件步骤。

警告

仅允许在操作的行上的局部属性上执行**非常有限的操作**，以及在给定的`Connection`上允许发出任何 SQL。 **请务必充分阅读**有关使用这些方法的指南的 Mapper 级别刷新事件的说明；一般情况下，应优先考虑`SessionEvents.before_flush()`方法进行一般性刷新更改。

参数：

+   `mapper` – 这个事件目标的`Mapper`。

+   `connection` – 用于为此实例发出 INSERT 语句的`Connection`。这提供了一个句柄到目标数据库上当前事务的处理，该事务特定于此实例。

+   `target` – 正在持久化的映射实例。如果事件配置为 `raw=True`，则这将是与实例关联的`InstanceState`状态管理对象。

返回值：

不支持此事件的返回值。

另请参阅

持久化事件

```py
method after_mapper_constructed(mapper: Mapper[_O], class_: Type[_O]) → None
```

当`Mapper`完全构建完成时，接收一个类和映射器。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeClass, 'after_mapper_constructed')
def receive_after_mapper_constructed(mapper, class_):
    "listen for the 'after_mapper_constructed' event"

    # ... (event handling logic) ...
```

此事件在初始构造函数完成后调用`Mapper`。这发生在`MapperEvents.instrument_class()`事件之后，也发生在`Mapper`对其参数进行初始遍历以生成其`MapperProperty`对象集合之后，该集合可通过`Mapper.get_property()`方法和`Mapper.iterate_properties`属性访问。

该事件与`MapperEvents.before_mapper_configured()`事件的不同之处在于它在`Mapper`的构造函数内调用，而不是在`registry.configure()`过程中调用。目前，这是唯一一个适用于希望在构造此`Mapper`时创建其他映射类的处理程序的事件，这些映射类将在下次运行`registry.configure()`时成为同一配置步骤的一部分。

新版本 2.0.2 中新增。

另请参阅

对象版本控制 - 一个示例，演示了使用`MapperEvents.before_mapper_configured()`事件创建新的映射器以记录对象的变更审计历史。

```py
method after_update(mapper: Mapper[_O], connection: Connection, target: _O) → None
```

在发出与该实例相对应的 UPDATE 语句之后接收对象实例。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeClass, 'after_update')
def receive_after_update(mapper, connection, target):
    "listen for the 'after_update' event"

    # ... (event handling logic) ...
```

注意

此事件**仅**适用于会话刷新操作，并且**不**适用于在 ORM 启用的 INSERT、UPDATE 和 DELETE 语句中描述的 ORM DML 操作。要拦截 ORM DML 事件，请使用`SessionEvents.do_orm_execute()`。

此事件用于修改更新后的实例上的仅在 Python 中的状态，以及在给定连接上发出附加 SQL 语句。

对于所有标记为“脏”的实例都会调用此方法，*即使它们的基于列的属性没有任何净变化*，并且没有进行 UPDATE 语句。当对象的任何基于列的属性被调用“设置属性”操作或其任何集合被修改时，对象被标记为脏。如果在更新时，没有基于列的属性有任何净变化，则不会发出 UPDATE 语句。这意味着被发送到`MapperEvents.after_update()`的实例*不能*保证已发出 UPDATE 语句。

要检测对象上基于列的属性是否有净变化，从而导致 UPDATE 语句，请使用`object_session(instance).is_modified(instance, include_collections=False)`。

在前一步骤一次性发出它们的 UPDATE 语句之后，往往为同一类对象的一批对象调用事件。在极其罕见的情况下，如果这不是可取的，`Mapper`可以配置为`batch=False`，这将导致实例批次被分解为单个（性能较差）事件->持久化->事件步骤。

警告

Mapper 级别的刷新事件仅允许对仅与正在操作的行本地属性进行**非常有限的操作**，并允许在给定的`Connection`上发出任何 SQL。**请完整阅读**Mapper 级刷新事件的注释以获取有关使用这些方法的指南；一般而言，应首选`SessionEvents.before_flush()`方法进行一般的刷新更改。

参数：

+   `mapper` – 这个事件目标的`Mapper`。

+   `connection` – 用于为此实例发出 UPDATE 语句的`Connection`。这提供了一个句柄到当前事务的目标数据库，该事务特定于此实例。

+   `target` – 被持久化的映射实例。如果事件配置为`raw=True`，则这将是与实例关联的`InstanceState`状态管理对象。

返回：

此事件不支持返回值。

另请参阅

持久性事件

```py
method before_configured() → None
```

在一系列映射器配置之前调用。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeClass, 'before_configured')
def receive_before_configured():
    "listen for the 'before_configured' event"

    # ... (event handling logic) ...
```

每次调用`configure_mappers()`函数时，都会调用`MapperEvents.before_configured()`事件，在函数尚未执行任何工作之前。 `configure_mappers()`通常在首次使用映射时自动调用，以及每次新的映射器可用并检测到新的映射器使用时调用。

此事件**仅**适用于`Mapper`类，而不适用于单个映射或映射类。它仅为所有映射作为一个整体调用：

```py
from sqlalchemy.orm import Mapper

@event.listens_for(Mapper, "before_configured")
def go():
    ...
```

将此事件与`MapperEvents.after_configured()`进行对比，后者在一系列映射器已配置之后调用，以及`MapperEvents.before_mapper_configured()`和`MapperEvents.mapper_configured()`，它们在每个映射器基础上调用。

理论上，此事件在应用程序中每次调用一次，但实际上，任何时候新的映射器都会受到`configure_mappers()`调用的影响。如果在已使用现有映射器之后构造新映射，则可能会再次调用此事件。为确保仅调用特定事件一次且不再调用，可以应用`once=True`参数（0.9.4 中的新功能）：

```py
from sqlalchemy.orm import mapper

@event.listens_for(mapper, "before_configured", once=True)
def go():
    ...
```

另请参阅

`MapperEvents.before_mapper_configured()`

`MapperEvents.mapper_configured()`

`MapperEvents.after_configured()`

```py
method before_delete(mapper: Mapper[_O], connection: Connection, target: _O) → None
```

在发出与该实例对应的 DELETE 语句之前接收一个对象实例。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeClass, 'before_delete')
def receive_before_delete(mapper, connection, target):
    "listen for the 'before_delete' event"

    # ... (event handling logic) ...
```

注意

此事件**仅适用于**会话刷新操作，**不适用于**在 ORM-Enabled INSERT, UPDATE, 和 DELETE statements 中描述的 ORM DML 操作。要拦截 ORM DML 事件，请使用`SessionEvents.do_orm_execute()`。

此事件用于在给定连接上发出额外的 SQL 语句，以及执行与删除事件相关的应用程序特定簿记。

该事件通常在后续步骤中一次性发出同一类对象的批量 DELETE 语句之前为其进行调用。

警告

仅允许在仅对操作的行本地属性上进行**非常有限的操作**，以及允许在给定的`Connection`上发出任何 SQL 语句。**请务必充分阅读**Mapper-level Flush Events 中的注意事项，以获取使用这些方法的指南；通常，应优先使用`SessionEvents.before_flush()`方法进行常规的刷新更改。

参数：

+   `mapper` – 此事件的目标`Mapper`。

+   `connection` – 用于为此实例发出 DELETE 语句的`Connection`。这提供了一个句柄进入与此实例特定目标数据库上的当前事务。

+   `target` – 正在删除的映射实例。如果事件配置为`raw=True`，则这将是与实例关联的`InstanceState`状态管理对象。

返回：

此事件不支持返回值。

另请参阅

持久化事件

```py
method before_insert(mapper: Mapper[_O], connection: Connection, target: _O) → None
```

在发出与该实例对应的 INSERT 语句之前接收对象实例。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeClass, 'before_insert')
def receive_before_insert(mapper, connection, target):
    "listen for the 'before_insert' event"

    # ... (event handling logic) ...
```

注意

此事件**仅**适用于会话刷新操作，**不**适用于 ORM 启用的 INSERT、UPDATE 和 DELETE 语句中描述的 ORM DML 操作。要拦截 ORM DML 事件，请使用`SessionEvents.do_orm_execute()`。

此事件用于在发生 INSERT 之前修改实例上的本地、非对象相关属性，以及在给定连接上发出附加的 SQL 语句。

在稍后的步骤中一次性发出它们的 INSERT 语句之前，通常为同一类对象的一批对象调用此事件。在极为罕见的情况下，如果这不是理想的情况，可以使用`batch=False`配置`Mapper`对象，这将导致实例批次被拆分为单个（性能较差）事件->持久化->事件步骤。

警告

Mapper 级别的刷新事件仅允许**非常有限的操作**，仅限于对正在操作的行本地属性的操作，以及允许在给定的`Connection`上发出任何 SQL。**请完全阅读**Mapper 级别刷新事件中的注意事项，以获取有关使用这些方法的指南；通常，应优先使用`SessionEvents.before_flush()`方法进行一般的刷新更改。

参数：

+   `mapper` – 此事件的目标`Mapper`。

+   `connection` – 用于为此实例发出 INSERT 语句的 `Connection`。这提供了一个在目标数据库上当前事务中使用的句柄，该事务特定于此实例。

+   `target` – 正在持久化的映射实例。如果事件配置为 `raw=True`，则这将改为与实例关联的 `InstanceState` 状态管理对象。

返回：

此事件不支持返回值。

另请参阅

持久性事件

```py
method before_mapper_configured(mapper: Mapper[_O], class_: Type[_O]) → None
```

在特定映射器配置之前调用。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeClass, 'before_mapper_configured')
def receive_before_mapper_configured(mapper, class_):
    "listen for the 'before_mapper_configured' event"

    # ... (event handling logic) ...
```

此事件旨在允许在配置步骤中跳过特定映射器，方法是返回 `interfaces.EXT_SKIP` 符号，该符号表示向 `configure_mappers()` 调用指示此特定映射器（或如果使用 `propagate=True` 则是映射器层次结构）应在当前配置运行中被跳过。当跳过一个或多个映射器时，将保持“新映射器”标志设置，这意味着当使用映射器时，将继续调用 `configure_mappers()` 函数，以继续尝试配置所有可用的映射器。

与其他配置级事件 `MapperEvents.before_configured()`、`MapperEvents.after_configured()` 和 `MapperEvents.mapper_configured()` 相比，当使用 `retval=True` 参数注册时，:meth;`.MapperEvents.before_mapper_configured` 事件在注册时提供了有意义的返回值。

版本 1.3 中的新功能。

例如：

```py
from sqlalchemy.orm import EXT_SKIP

Base = declarative_base()

DontConfigureBase = declarative_base()

@event.listens_for(
    DontConfigureBase,
    "before_mapper_configured", retval=True, propagate=True)
def dont_configure(mapper, cls):
    return EXT_SKIP
```

另请参阅

`MapperEvents.before_configured()`

`MapperEvents.after_configured()`

`MapperEvents.mapper_configured()`

```py
method before_update(mapper: Mapper[_O], connection: Connection, target: _O) → None
```

在发出相应于该实例的 UPDATE 语句之前接收对象实例。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeClass, 'before_update')
def receive_before_update(mapper, connection, target):
    "listen for the 'before_update' event"

    # ... (event handling logic) ...
```

注

此事件**仅**适用于会话刷新操作，**不**适用于 ORM 启用的 INSERT、UPDATE 和 DELETE 语句中描述的 ORM DML 操作。要拦截 ORM DML 事件，请使用`SessionEvents.do_orm_execute()`。

此事件用于在 UPDATE 发生之前修改实例上的本地、非对象相关属性，以及在给定连接上发出额外的 SQL 语句。

此方法适用于所有被标记为“脏”的实例，*即使它们的基于列的属性没有净变化*。当对象的任何基于列的属性被调用“设置属性”操作或其集合被修改时，对象被标记为脏。如果在更新时，没有基于列的属性有任何净变化，那么不会发出 UPDATE 语句。这意味着将实例发送到`MapperEvents.before_update()`并*不*保证会发出 UPDATE 语句，尽管您可以通过修改属性以使值存在净变化来影响结果。

要检测对象的基于列的属性是否有净变化，并因此生成 UPDATE 语句，请使用`object_session(instance).is_modified(instance, include_collections=False)`。

在稍后的步骤中，通常在一批相同类的对象之前调用此事件，然后一次发出它们的 UPDATE 语句。在极为罕见的情况下，如果这不可取，可以配置`Mapper`为`batch=False`，这将导致实例批次被拆分为单个（性能较差）事件->持久化->事件步骤。

警告

Mapper 级别的刷新事件仅允许**非常有限的操作**，仅限于操作的行本地属性，以及允许在给定的`Connection`上发出任何 SQL。请**完全阅读**Mapper 级别刷新事件中的注意事项，以获取有关使用这些方法的指导；通常，应优先使用`SessionEvents.before_flush()`方法进行一般的刷新更改。

参数:

+   `mapper` – 此事件目标的`Mapper`。

+   `connection` – 用于为此实例发出 UPDATE 语句的 `Connection` 。这提供了一个句柄进入当前数据库的事务，特定于此实例。

+   `target` – 正在持久化的映射实例。如果事件配置为 `raw=True`，则这将是与实例关联的 `InstanceState` 状态管理对象。

返回值：

此事件不支持返回值。

另请参见

持久性事件

```py
attribute dispatch: _Dispatch[_ET] = <sqlalchemy.event.base.MapperEventsDispatch object>
```

参考回到 _Dispatch 类。

双向反对 _Dispatch._events

```py
method instrument_class(mapper: Mapper[_O], class_: Type[_O]) → None
```

当映射器首次构造时接收类，然后应用到映射类之前的仪器。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeClass, 'instrument_class')
def receive_instrument_class(mapper, class_):
    "listen for the 'instrument_class' event"

    # ... (event handling logic) ...
```

此事件是映射器构造的最早阶段。大多数映射器的属性尚未初始化。要在初始映射器构造中接收事件，在其中可以使用基本状态的情况下，例如 `Mapper.attrs` 集合，可能更好地选择 `MapperEvents.after_mapper_constructed()` 事件。

此侦听器可以应用于整个 `Mapper` 类，也可以应用于任何未映射的类，该类用作将要映射的类的基类（使用 `propagate=True` 标志）：

```py
Base = declarative_base()

@event.listens_for(Base, "instrument_class", propagate=True)
def on_new_class(mapper, cls_):
    " ... "
```

参数：

+   `mapper` – 此事件目标的 `Mapper` 。

+   `class_` – 映射的类。

另请参见

`MapperEvents.after_mapper_constructed()`

```py
method mapper_configured(mapper: Mapper[_O], class_: Type[_O]) → None
```

当特定映射器在 `configure_mappers()` 调用范围内完成其自身配置时调用。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeClass, 'mapper_configured')
def receive_mapper_configured(mapper, class_):
    "listen for the 'mapper_configured' event"

    # ... (event handling logic) ...
```

当 `configure_mappers()` 函数通过当前尚未配置的映射器列表进行时，对遇到的每个映射器调用 `MapperEvents.mapper_configured()` 事件。通常在首次使用映射时自动调用 `configure_mappers()`，以及每次有新映射器可用并检测到新映射器使用时。

当调用事件时，映射器应处于最终状态，但**不包括**可能从其他映射器调用的反向引用；它们可能仍在配置操作中挂起。通过`relationship.back_populates`参数配置的双向关系将完全可用，因为这种关系方式不依赖于其他可能尚未配置的映射器来知道它们的存在。

对于一个保证**所有**映射都准备就绪，包括仅在其他映射上定义的反向引用的事件，请使用`MapperEvents.after_configured()`事件；此事件仅在所有已知映射完全配置后才调用。

与`MapperEvents.before_configured()`或`MapperEvents.after_configured()`不同，`MapperEvents.mapper_configured()`事件为每个映射/类单独调用，并将映射器传递给事件本身。对于特定映射器，该事件仅调用一次。因此，该事件对于在特定映射器基础上仅调用一次的配置步骤非常有用，这些步骤不要求“反向引用”配置必须已准备就绪。

参数：

+   `mapper` – 这个事件的目标是`Mapper`。

+   `class_` – 映射的类。

另请参阅

`MapperEvents.before_configured()`

`MapperEvents.after_configured()`

`MapperEvents.before_mapper_configured()`

## 实例事件

实例事件专注于 ORM 映射实例的构建，包括当它们作为瞬态对象实例化时，当它们从数据库加载并成为持久对象时，以及当数据库刷新或过期操作发生在对象上时。

| 对象名称 | 描述 |
| --- | --- |
| InstanceEvents | 定义特定于对象生命周期的事件。 |

```py
class sqlalchemy.orm.InstanceEvents
```

定义特定于对象生命周期的事件。

例如：

```py
from sqlalchemy import event

def my_load_listener(target, context):
    print("on load!")

event.listen(SomeClass, 'load', my_load_listener)
```

可用的目标包括：

+   已映射的类

+   已映射或将要映射的类的未映射超类（使用`propagate=True`标志）

+   `Mapper`对象

+   `Mapper`类本身指示侦听所有映射器。

实例事件与映射器事件密切相关，但更具体于实例及其仪器化，而不是其持久化系统。

使用`InstanceEvents`时，`listen()`函数提供了几个修饰符。

参数：

+   `propagate=False` – 当为 True 时，事件监听器应该应用于所有继承类，以及作为此监听器目标的类。

+   `raw=False` – 当为 True 时，适用的事件监听器函数传递给“target”参数将是实例的`InstanceState`管理对象，而不是映射实例本身。

+   `restore_load_context=False` –

    适用于`InstanceEvents.load()`和`InstanceEvents.refresh()`事件。在事件钩子完成后恢复对象的加载器上下文，以便持续的急加载操作继续适当地针对对象。如果在这些事件之一内部将对象移动到新的加载器上下文中并且未设置此标志，则会发出警告。

    版本 1.3.14 中的新内容。

**成员**

dispatch, expire(), first_init(), init(), init_failure(), load(), pickle(), refresh(), refresh_flush(), unpickle()

**类签名**

类`sqlalchemy.orm.InstanceEvents` (`sqlalchemy.event.Events`)

```py
attribute dispatch: _Dispatch[_ET] = <sqlalchemy.event.base.InstanceEventsDispatch object>
```

回溯到 _Dispatch 类。

双向对 _Dispatch._events

```py
method expire(target: _O, attrs: Iterable[str] | None) → None
```

在其属性或某些子集被过期后接收对象实例。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeClass, 'expire')
def receive_expire(target, attrs):
    "listen for the 'expire' event"

    # ... (event handling logic) ...
```

‘keys’是属性名称列表。如果为 None，则整个状态已过期。

参数：

+   `target` – 映射实例。如果事件配置为`raw=True`，则此处将替代与实例关联的`InstanceState`状态管理对象。

+   `attrs` – 被过期的属性名称序列，如果所有属性均已过期则为 None。

```py
method first_init(manager: ClassManager[_O], cls: Type[_O]) → None
```

当特定映射的第一个实例被调用时调用。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeClass, 'first_init')
def receive_first_init(manager, cls):
    "listen for the 'first_init' event"

    # ... (event handling logic) ...
```

当为该特定类第一次调用`__init__`方法时调用此事件。该事件在`__init__`实际执行之前以及在调用`InstanceEvents.init()`事件之前调用。

```py
method init(target: _O, args: Any, kwargs: Any) → None
```

当其构造函数被调用时接收一个实例。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeClass, 'init')
def receive_init(target, args, kwargs):
    "listen for the 'init' event"

    # ... (event handling logic) ...
```

此方法仅在对象的用户空间构造期间调用，与对象的构造函数（例如其`__init__`方法）一起。当对象从数据库加载时不会调用它；请参阅`InstanceEvents.load()`事件以拦截数据库加载。

在实际调用对象的`__init__`构造函数之前调用该事件。`kwargs`字典可以就地修改，以影响传递给`__init__`的内容。

参数：

+   `target` – 映射的实例。如果事件配置为`raw=True`，则这将是与该实例关联的`InstanceState`状态管理对象。

+   `args` – 传递给`__init__`方法的位置参数。这将作为元组传递，目前不可变。

+   `kwargs` – 传递给`__init__`方法的关键字参数。此结构*可以*就地更改。

另请参阅

`InstanceEvents.init_failure()`

`InstanceEvents.load()`

```py
method init_failure(target: _O, args: Any, kwargs: Any) → None
```

当其构造函数被调用并引发异常时接收一个实例。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeClass, 'init_failure')
def receive_init_failure(target, args, kwargs):
    "listen for the 'init_failure' event"

    # ... (event handling logic) ...
```

此方法仅在对象的用户空间构造期间调用，与对象的构造函数（例如其`__init__`方法）一起。当对象从数据库加载时不会调用它。

在捕获到`__init__`方法引发的异常后调用该事件。调用事件后，原始异常将重新引发，以便对象的构造仍然引发异常。应在`sys.exc_info()`中提供实际异常和堆栈跟踪引发的异常。

参数：

+   `target` – 映射的实例。如果事件配置为`raw=True`，则这将是与该实例关联的`InstanceState`状态管理对象。

+   `args` – 传递给`__init__`方法的位置参数。

+   `kwargs` – 传递给`__init__`方法的关键字参数。

另请参阅

`InstanceEvents.init()`

`InstanceEvents.load()`

```py
method load(target: _O, context: QueryContext) → None
```

在通过 `__new__` 创建对象实例并进行初始属性填充之后接收对象实例。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeClass, 'load')
def receive_load(target, context):
    "listen for the 'load' event"

    # ... (event handling logic) ...
```

这通常发生在基于传入结果行创建实例时，并且仅针对该实例的生命周期调用一次。

警告

在结果行加载期间，当处理此实例的第一行接收到时会调用此事件。当使用带有集合定向属性的急切加载时，用于加载后续集合项的其他行尚未发生 / 处理。这既会导致集合不会完全加载，也会导致如果在此事件处理程序内发生了导致对象发出另一个数据库加载操作的操作，则对象的“加载上下文”可能会发生变化，并干扰正在进行的现有急切加载程序。

导致事件处理程序内的“加载上下文”发生变化的原因示例包括但不限于：

+   访问未包含在行中的延迟属性将触发“取消延迟”操作并刷新对象。

+   访问未包含在行中的联接继承子类的属性将触发刷新操作。

从 SQLAlchemy 1.3.14 开始，当发生这种情况时会发出警告。`InstanceEvents.restore_load_context` 选项可用于事件上以防止此警告；这将确保在调用事件后保持对象的现有加载上下文：

```py
@event.listens_for(
    SomeClass, "load", restore_load_context=True)
def on_load(instance, context):
    instance.some_unloaded_attribute
```

1.3.14 版更改：增加了 `InstanceEvents.restore_load_context` 和 `SessionEvents.restore_load_context` 标志，适用于“加载”事件，将确保在事件挂钩完成时恢复对象的加载上下文；如果没有设置此标志，则会发出警告，指示对象的加载上下文发生了变化。

`InstanceEvents.load()` 事件也以类方法装饰器格式可用，称为 `reconstructor()`。

参数：

+   `target` – 映射的实例。如果事件配置了 `raw=True`，则会变成与实例关联的`InstanceState`状态管理对象。

+   `context` – 与当前进行中的`Query`对应的`QueryContext`。如果加载不对应于`Query`，例如在`Session.merge()`期间，此参数可能为`None`。

另请参阅

维护跨加载的非映射状态

`InstanceEvents.init()`

`InstanceEvents.refresh()`

`SessionEvents.loaded_as_persistent()`

```py
method pickle(target: _O, state_dict: _InstanceDict) → None
```

在其关联状态被 pickled 时接收对象实例。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeClass, 'pickle')
def receive_pickle(target, state_dict):
    "listen for the 'pickle' event"

    # ... (event handling logic) ...
```

参数：

+   `target` – 映射实例。如果事件配置为`raw=True`，则这将是与实例关联的`InstanceState`状态管理对象。

+   `state_dict` – 由`__getstate__`返回的字典，包含要被 pickled 的状态。

```py
method refresh(target: _O, context: QueryContext, attrs: Iterable[str] | None) → None
```

在从查询中刷新一个或多个属性后接收对象实例。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeClass, 'refresh')
def receive_refresh(target, context, attrs):
    "listen for the 'refresh' event"

    # ... (event handling logic) ...
```

与`InstanceEvents.load()`方法形成对比，当对象首次从查询中加载时会调用该方法。

注意

在加载器进程中在急切加载器可能已完成之前调用此事件，并且对象的状态可能不完整。此外，在对象上调用行级刷新操作将使对象进入新的加载器上下文，干扰现有的加载上下文。有关如何利用`InstanceEvents.restore_load_context`参数解决此场景的背景，请参阅有关`InstanceEvents.load()`的注释。

参数：

+   `target` – 映射实例。如果事件配置为`raw=True`，则这将是与实例关联的`InstanceState`状态管理对象。

+   `context` – 与当前进行中的`Query`对应的`QueryContext`。

+   `attrs` – 已填充的属性名称序列，如果所有列映射的非延迟属性都已填充，则为 None。

另请参阅

在加载过程中保持非映射状态

`InstanceEvents.load()`

```py
method refresh_flush(target: _O, flush_context: UOWTransaction, attrs: Iterable[str] | None) → None
```

在对象的状态持久化过程中，当一个或多个包含列级默认值或 onupdate 处理程序的属性被刷新后，会收到一个对象实例。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeClass, 'refresh_flush')
def receive_refresh_flush(target, flush_context, attrs):
    "listen for the 'refresh_flush' event"

    # ... (event handling logic) ...
```

此事件与`InstanceEvents.refresh()`相同，只是在工作单元刷新过程中调用，并且仅包括具有列级默认值或 onupdate 处理程序的非主键列，包括 Python 可调用对象以及通过 RETURNING 子句获取的服务器端默认值和触发器。

注意

虽然`InstanceEvents.refresh_flush()`事件是为 INSERT 和 UPDATE 的对象触发的，但该事件主要针对 UPDATE 过程；这主要是一个内部工件，INSERT 操作也可以触发此事件，并注意**INSERT 行的主键列明确地在此事件中被省略**。为了拦截对象的新 INSERT 状态，`SessionEvents.pending_to_persistent()`和`MapperEvents.after_insert()`是更好的选择。

参数：

+   `target` – 映射实例。如果事件配置为`raw=True`，那么这将是与实例关联的`InstanceState`状态管理对象。

+   `flush_context` – 处理刷新细节的内部`UOWTransaction`对象。

+   `attrs` – 被填充的属性名称序列。

另请参阅

在加载过程中保持非映射状态

获取服务器生成的默认值

列的 INSERT/UPDATE 默认值

```py
method unpickle(target: _O, state_dict: _InstanceDict) → None
```

在关联状态被反 pickle 后，收到一个对象实例。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeClass, 'unpickle')
def receive_unpickle(target, state_dict):
    "listen for the 'unpickle' event"

    # ... (event handling logic) ...
```

参数：

+   `target` – 映射实例。如果事件配置为`raw=True`，那么这将是与实例关联的`InstanceState`状态管理对象。

+   `state_dict` – 发送给`__setstate__`的字典，包含被 pickle 的状态字典。

## 属性事件

属性事件在 ORM 映射对象的各个属性发生事情时触发。这些事件构成了诸如自定义验证函数和反向引用处理程序等功能的基础。

另请参阅

更改属性行为

| 对象名称 | 描述 |
| --- | --- |
| AttributeEvents | 为对象属性定义事件。 |

```py
class sqlalchemy.orm.AttributeEvents
```

为对象属性定义事件。

这些通常在目标类的类绑定描述符上定义。

例如，要注册一个将接收`AttributeEvents.append()`事件的监听器：

```py
from sqlalchemy import event

@event.listens_for(MyClass.collection, 'append', propagate=True)
def my_append_listener(target, value, initiator):
    print("received append event for target: %s" % target)
```

当`AttributeEvents.retval`标志传递给`listen()`或`listens_for()`时，监听器可以选择返回可能修改的值的版本，如下所示，使用`AttributeEvents.set()`事件进行说明：

```py
def validate_phone(target, value, oldvalue, initiator):
    "Strip non-numeric characters from a phone number"

    return re.sub(r'\D', '', value)

# setup listener on UserContact.phone attribute, instructing
# it to use the return value
listen(UserContact.phone, 'set', validate_phone, retval=True)
```

类似上述的验证函数也可以引发异常，如`ValueError`以停止操作。

当将监听器应用于具有映射子类的映射类时，`AttributeEvents.propagate`标志也很重要，例如在使用映射器继承模式时：

```py
@event.listens_for(MySuperClass.attr, 'set', propagate=True)
def receive_set(target, value, initiator):
    print("value set: %s" % target)
```

`listen()` 和 `listens_for()` 函数可用的所有修饰符如下。

参数：

+   `active_history=False` – 当为 True 时，表示“set”事件希望无条件接收被替换的“旧”值，即使这需要触发数据库加载。请注意，`active_history`也可以通过`column_property()`和`relationship()`直接设置。

+   `propagate=False` – 当为 True 时，监听器函数将不仅为给定的类属性建立，还将为该类的所有当前子类以及该类的所有未来子类上具有相同名称的属性建立一个额外的监听器，该监听器监听仪器事件。

+   `raw=False` – 当为 True 时，事件的“target”参数将是`InstanceState`管理对象，而不是映射实例本身。

+   `retval=False` – 当为 True 时，用户定义的事件监听必须从函数返回“value”参数。这使得监听函数有机会改变最终用于“set”或“append”事件的值。

**成员**

append(), append_wo_mutation(), bulk_replace(), dispatch, dispose_collection(), init_collection(), init_scalar(), modified(), remove(), set()

**类签名**

类`sqlalchemy.orm.AttributeEvents` (`sqlalchemy.event.Events`)

```py
method append(target: _O, value: _T, initiator: Event, *, key: EventConstants = EventConstants.NO_KEY) → _T | None
```

接收集合追加事件。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeClass.some_attribute, 'append')
def receive_append(target, value, initiator):
    "listen for the 'append' event"

    # ... (event handling logic) ...
```

每当元素被追加到集合中时，都会调用追加事件。这适用于单个元素追加以及“批量替换”操作。

参数：

+   `target` – 接收事件的对象实例。如果监听器注册为`raw=True`，则这将是`InstanceState`对象。

+   `value` – 被追加的值。如果此监听器注册为`retval=True`，则监听函数必须返回此值，或替换它的新值。

+   `initiator` – 一个代表事件启动的`Event`实例。可能会被 backref 处理程序修改其原始值，以控制链式事件传播，同时也可以被检查以获取有关事件源的信息。

+   `key` –

    当使用`AttributeEvents.include_key`参数设置为 True 来建立事件时，这将是操作中使用的键，例如`collection[some_key_or_index] = value`。如果未使用`AttributeEvents.include_key`设置事件，则根本不会传递该参数；这是为了允许与不包括`key`参数的现���事件处理程序向后兼容。

    2.0 版中的新内容。

返回值：

如果事件注册时使用`retval=True`，应返回给定值或新的有效值。

另请参阅

`AttributeEvents` - 关于监听器选项的背景，如传播到子类。

`AttributeEvents.bulk_replace()`

```py
method append_wo_mutation(target: _O, value: _T, initiator: Event, *, key: EventConstants = EventConstants.NO_KEY) → None
```

接收一个集合追加事件，其中集合实际上未发生变化。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeClass.some_attribute, 'append_wo_mutation')
def receive_append_wo_mutation(target, value, initiator):
    "listen for the 'append_wo_mutation' event"

    # ... (event handling logic) ...
```

此事件与`AttributeEvents.append()`不同，因为它是为了去重集合（如集合和字典）而触发的，当对象已经存在于目标集合中时。该事件没有返回值，给定对象的标识不能更改。

当集合已通过反向引用事件发生变异时，此事件用于级联对象到`Session`。

参数：

+   `target` – 接收事件的对象实例。如果侦听器注册为`raw=True`，这将是`InstanceState`对象。

+   `value` – 如果对象尚未存在于集合中，则将要附加的值。

+   `initiator` – 代表事件启动的`Event`实例。可以通过反向引用处理程序修改其原始值，以控制链式事件传播，并且可以检查有关事件源的信息。

+   `key` –

    当使用`AttributeEvents.include_key`参数设置为 True 来建立事件时，这将是操作中使用的键，例如 `collection[some_key_or_index] = value`。如果没有使用`AttributeEvents.include_key`来设置事件，则根本不会将参数传递给事件；这是为了允许与不包括`key`参数的现有事件处理程序保持向后兼容。

    版本 2.0 中的新内容。

返回值：

没有为此事件定义返回值。

版本 1.4.15 中的新内容。

```py
method bulk_replace(target: _O, values: Iterable[_T], initiator: Event, *, keys: Iterable[EventConstants] | None = None) → None
```

接收一个集合‘批量替换’事件。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeClass.some_attribute, 'bulk_replace')
def receive_bulk_replace(target, values, initiator):
    "listen for the 'bulk_replace' event"

    # ... (event handling logic) ...
```

当值作为批量集合设置操作的一部分传入时，将调用此事件，可以在值被视为 ORM 对象之前就地修改。这是一个“早期挂钩”，在批量替换例程尝试协调哪些对象已经存在于集合中，哪些对象被净替换操作移除之前运行。

通常情况下，这个方法会与`AttributeEvents.append()`事件一起使用。当同时使用这两个事件时，请注意，批量替换操作将为所有新项目调用`AttributeEvents.append()`事件，即使在为整个集合调用`AttributeEvents.bulk_replace()`之后。为了确定`AttributeEvents.append()`事件是否是批量替换的一部分，请使用符号`attributes.OP_BULK_REPLACE`来测试传入的 initiator：

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

版本 1.2 中的新功能。

参数：

+   `target` – 接收事件的对象实例。如果监听器以`raw=True`注册，这将是`InstanceState`对象。

+   `value` – 被设置的值的序列（例如列表）。处理程序可以直接修改此列表。

+   `initiator` – 代表事件启动的`Event`实例。

+   `keys` –

    当使用`AttributeEvents.include_key`参数设置为 True 来建立事件时，这将是操作中使用的键的序列，通常仅用于字典更新。如果未使用`AttributeEvents.include_key`设置事件，参数根本不会传递给事件；这是为了允许与不包括`key`参数的现有事件处理程序保持向后兼容。

    版本 2.0 中的新功能。

另请参见

`AttributeEvents` - 关于监听器选项的背景，例如传播到子类。

```py
attribute dispatch: _Dispatch[_ET] = <sqlalchemy.event.base.AttributeEventsDispatch object>
```

参考回到 _Dispatch 类。

双向对 _Dispatch._events

```py
method dispose_collection(target: _O, collection: Collection[Any], collection_adapter: CollectionAdapter) → None
```

接收“集合处理”事件。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeClass.some_attribute, 'dispose_collection')
def receive_dispose_collection(target, collection, collection_adapter):
    "listen for the 'dispose_collection' event"

    # ... (event handling logic) ...
```

当集合被替换时，此事件将为基于集合的属性触发，即：

```py
u1.addresses.append(a1)

u1.addresses = [a2, a3]  # <- old collection is disposed
```

旧集合接收到的将包含其先前的内容。

版本 1.2 中的更改：传递给`AttributeEvents.dispose_collection()`的集合现在在处理之前会保持其内容；以前，集合将为空。

另请参见

`AttributeEvents` - 关于监听器选项的背景，例如传播到子类。

```py
method init_collection(target: _O, collection: Type[Collection[Any]], collection_adapter: CollectionAdapter) → None
```

接收“集合初始化”事件。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeClass.some_attribute, 'init_collection')
def receive_init_collection(target, collection, collection_adapter):
    "listen for the 'init_collection' event"

    # ... (event handling logic) ...
```

当为空属性首次生成初始的“空集合”时以及当集合被新集合替换时，例如通过 set 事件，将触发此事件。

例如，给定`User.addresses`是基于关系的集合，此处触发事件：

```py
u1 = User()
u1.addresses.append(a1)  #  <- new collection
```

并且在替换操作期间也是如此：

```py
u1.addresses = [a2, a3]  #  <- new collection
```

参数：

+   `target` – 接收事件的对象实例。如果监听器注册为`raw=True`，则这将是`InstanceState`对象。

+   `collection` – 新的集合。这将始终从`relationship.collection_class`中指定的内容生成，并且始终为空。

+   `collection_adapter` – 将调解对集合的内部访问的`CollectionAdapter`。

另请参阅

`AttributeEvents` - 关于监听器选项的背景，例如传播到子类。

`AttributeEvents.init_scalar()` - 此事件的“标量”版本。

```py
method init_scalar(target: _O, value: _T, dict_: Dict[Any, Any]) → None
```

接收一个标量“init”事件。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeClass.some_attribute, 'init_scalar')
def receive_init_scalar(target, value, dict_):
    "listen for the 'init_scalar' event"

    # ... (event handling logic) ...
```

当访问未初始化的、未持久化的标量属性时，会调用此事件，例如读取：

```py
x = my_object.some_attribute
```

当此事件发生在未初始化的属性上时，ORM 的默认行为是返回值`None`；请注意，这与 Python 的通常行为不同，Python 通常会引发`AttributeError`。此处的事件可用于自定义实际返回的值，假设事件侦听器将镜像配置在 Core `Column` 对象上的默认生成器。

由于`Column`上的默认生成器可能也会产生一个变化的值，例如时间戳，所以`AttributeEvents.init_scalar()`事件处理程序也可以用于**设置**新返回的值，以便 Core 级别的默认生成函数仅在访问非持久化对象上的属性时触发一次，但是在这一刻。通常，当访问未初始化的属性时，不会对对象的状态进行任何更改（较旧的 SQLAlchemy 版本实际上会更改对象的状态）。

如果列上的默认生成器返回特定常量，则可能会使用处理程序如下：

```py
SOME_CONSTANT = 3.1415926

class MyClass(Base):
    # ...

    some_attribute = Column(Numeric, default=SOME_CONSTANT)

@event.listens_for(
    MyClass.some_attribute, "init_scalar",
    retval=True, propagate=True)
def _init_some_attribute(target, dict_, value):
    dict_['some_attribute'] = SOME_CONSTANT
    return SOME_CONSTANT
```

在上面的示例中，我们将属性`MyClass.some_attribute`初始化为`SOME_CONSTANT`的值。以上代码包括以下功能：

+   通过在给定的`dict_`中设置值`SOME_CONSTANT`，我们指示该值将被持久化到数据库中。这将取代在`Column`的默认生成器中使用`SOME_CONSTANT`。在属性仪器化中给出的`active_column_defaults.py`示例说明了使用相同方法进行更改默认值的方法，例如时间戳生成器。在这个特定的例子中，这样做并不是严格必要的，因为`SOME_CONSTANT`无论如何都会成为 INSERT 语句的一部分。

+   通过建立`retval=True`标志，我们从函数返回的值将由属性获取器返回。没有此标志，事件被视为被动观察者，我们函数的返回值将被忽略。

+   如果映射类包括继承的子类，则`propagate=True`标志是重要的，这些子类也会使用此事件侦听器。没有此标志，继承的子类将不使用我们的事件处理程序。

在上述示例中，当我们将值应用于给定的`dict_`时，不会调用属性设置事件`AttributeEvents.set()`以及由`validates`提供的相关验证功能。要使这些事件响应我们新生成的值而调用，请将该值应用于给定对象作为普通属性设置操作：

```py
SOME_CONSTANT = 3.1415926

@event.listens_for(
    MyClass.some_attribute, "init_scalar",
    retval=True, propagate=True)
def _init_some_attribute(target, dict_, value):
    # will also fire off attribute set events
    target.some_attribute = SOME_CONSTANT
    return SOME_CONSTANT
```

当设置了多个侦听器时，值的生成会从一个侦听器“链式”传递到下一个侦听器，通过将由前一个指定了`retval=True`的侦听器返回的值作为下一个侦听器的`value`参数传递。

参数：

+   `target` – 接收事件的对象实例。如果侦听器使用`raw=True`注册，这将是`InstanceState`对象。

+   `value` – 在调用此事件侦听器之前要返回的值。此值最初为值`None`，但如果存在多个侦听器，则将是前一个事件处理程序函数的返回值。

+   `dict_` – 此映射对象的属性字典。这通常是对象的`__dict__`，但在所有情况下都表示属性系统用于访问此属性的实际值的目标。将值放入此字典中的效果是该值将在工作单元生成的 INSERT 语句中使用。

另请参见

`AttributeEvents.init_collection()` - 此事件的集合版本

`AttributeEvents` - 关于侦听器选项的背景，例如传播到子类。

属性仪表化 - 查看 `active_column_defaults.py` 示例。

```py
method modified(target: _O, initiator: Event) → None
```

接收一个“修改”事件。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeClass.some_attribute, 'modified')
def receive_modified(target, initiator):
    "listen for the 'modified' event"

    # ... (event handling logic) ...
```

当使用 `flag_modified()` 函数触发属性上的修改事件时，会触发此事件，而不设置任何特定值。

新版本 1.2 中的内容。

参数：

+   `target` – 接收事件的对象实例。如果侦听器注册为 `raw=True`，则这将是 `InstanceState` 对象。

+   `initiator` – 表示事件启动的 `Event` 的实例。

另请参阅

`AttributeEvents` - 有关侦听器选项的背景，如传播到子类。

```py
method remove(target: _O, value: _T, initiator: Event, *, key: EventConstants = EventConstants.NO_KEY) → None
```

接收一个集合移除事件。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeClass.some_attribute, 'remove')
def receive_remove(target, value, initiator):
    "listen for the 'remove' event"

    # ... (event handling logic) ...
```

参数：

+   `target` – 接收事件的对象实例。如果侦听器注册为 `raw=True`，则这将是 `InstanceState` 对象。

+   `value` – 被移除的值。

+   `initiator` – 表示事件启动的 `Event` 的实例。可能会被 backref 处理程序修改其原始值，以控制链式事件传播。

+   `key` –

    当使用 `AttributeEvents.include_key` 参数设置为 True 来建立事件时，这将是操作中使用的键，例如 `del collection[some_key_or_index]`。如果未使用 `AttributeEvents.include_key` 来设置事件，则根本不会传递该参数给事件；这是为了与不包括 `key` 参数的现有事件处理程序的向后兼容性。

    新版本 2.0 中的内容。

返回：

此事件未定义返回值。

另请参阅

`AttributeEvents` - 有关侦听器选项的背景，如传播到子类。

```py
method set(target: _O, value: _T, oldvalue: _T, initiator: Event) → None
```

接收一个标量设置事件。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeClass.some_attribute, 'set')
def receive_set(target, value, oldvalue, initiator):
    "listen for the 'set' event"

    # ... (event handling logic) ...
```

参数：

+   `target` – 接收事件的对象实例。如果侦听器注册为 `raw=True`，则这将是 `InstanceState` 对象。

+   `value` – 被设置的值。如果此侦听器注册为 `retval=True`，则侦听器函数必须返回此值，或者替换它的新值。

+   `oldvalue` – 被替换的先前值。这也可以是符号 `NEVER_SET` 或 `NO_VALUE`。如果侦听器注册为 `active_history=True`，则如果现有值当前未加载或过期，则将从数据库加载属性的先前值。

+   `initiator` – 代表事件启动的`Event`实例。可能会被 backref 处理程序修改其原始值，以控制链式事件传播。

返回值：

如果事件是以`retval=True`注册的，则应返回给定值或新的有效值。

另请参阅

`AttributeEvents` - 关于侦听器选项的背景，如传播到子类。

## 查询事件

| 对象名称 | 描述 |
| --- | --- |
| QueryEvents | 在构建`Query`对象时表示事件。 |

```py
class sqlalchemy.orm.QueryEvents
```

在构建`Query`对象时表示事件。

传统特性

`QueryEvents`事件方法在 SQLAlchemy 2.0 中已过时，仅适用于直接使用`Query`对象。它们不适用于 2.0 风格语句。要拦截和修改 2.0 风格 ORM 使用的事件，请使用`SessionEvents.do_orm_execute()`钩子。

`QueryEvents`钩子现在已被`SessionEvents.do_orm_execute()`事件钩取代。

**成员**

before_compile(), before_compile_delete(), before_compile_update(), dispatch

**类签名**

类`sqlalchemy.orm.QueryEvents` (`sqlalchemy.event.Events`)

```py
method before_compile(query: Query) → None
```

在将其组成为核心`Select`对象之前接收`Query`对象。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeQuery, 'before_compile')
def receive_before_compile(query):
    "listen for the 'before_compile' event"

    # ... (event handling logic) ...
```

自 1.4 版本起已弃用：`QueryEvents.before_compile()`事件被更强大的`SessionEvents.do_orm_execute()`钩子取代。在 1.4 版本中，`QueryEvents.before_compile()`事件**不再**用于 ORM 级别的属性加载，例如延迟加载或过期属性以及关联加载器的加载。请参阅 ORM 查询事件中的新示例，该示例说明了拦截和修改 ORM 查询以添加任意筛选条件的常见目的的新方法。

此事件旨在允许对给定的查询进行更改：

```py
@event.listens_for(Query, "before_compile", retval=True)
def no_deleted(query):
    for desc in query.column_descriptions:
        if desc['type'] is User:
            entity = desc['entity']
            query = query.filter(entity.deleted == False)
    return query
```

通常应该使用`retval=True`参数监听事件，以便修改后的查询可以返回。

默认情况下，`QueryEvents.before_compile()`事件将禁止“baked”查询缓存查询，如果事件钩子返回一个新的`Query`对象。这影响到对烘焙查询扩展的直接使用以及它在延迟加载器和关系的贪婪加载器中的操作。为了重新建立缓存查询，请应用添加了`bake_ok`标志的事件：

```py
@event.listens_for(
    Query, "before_compile", retval=True, bake_ok=True)
def my_event(query):
    for desc in query.column_descriptions:
        if desc['type'] is User:
            entity = desc['entity']
            query = query.filter(entity.deleted == False)
    return query
```

当`bake_ok`设置为 True 时，事件钩子将仅被调用一次，并且不会为正在被缓存的特定查询的后续调用调用。

从版本 1.3.11 开始：- 在`QueryEvents.before_compile()`事件中添加了“bake_ok”标志，并且如果未设置此标志，则不允许通过“baked”扩展缓存返回新的`Query`对象的事件处理程序发生。

另请参阅

`QueryEvents.before_compile_update()`

`QueryEvents.before_compile_delete()`

使用 before_compile 事件

```py
method before_compile_delete(query: Query, delete_context: BulkDelete) → None
```

允许在`Query.delete()`中对`Query`对象进行修改。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeQuery, 'before_compile_delete')
def receive_before_compile_delete(query, delete_context):
    "listen for the 'before_compile_delete' event"

    # ... (event handling logic) ...
```

自 1.4 版弃用：`QueryEvents.before_compile_delete()`事件已被更加强大的`SessionEvents.do_orm_execute()`钩子所取代。

与`QueryEvents.before_compile()`事件类似，此事件应配置为`retval=True`，并返回修改后的`Query`对象，如下所示：

```py
@event.listens_for(Query, "before_compile_delete", retval=True)
def no_deleted(query, delete_context):
    for desc in query.column_descriptions:
        if desc['type'] is User:
            entity = desc['entity']
            query = query.filter(entity.deleted == False)
    return query
```

参数：

+   `query` – 一个`Query`实例；这也是给定“删除上下文”对象的`.query`属性。

+   `delete_context` – 一个“删除上下文”对象，其类型与`QueryEvents.after_bulk_delete.delete_context`中描述的对象相同。

自版本 1.2.17 起新增。

另请参阅

`QueryEvents.before_compile()`

`QueryEvents.before_compile_update()`

```py
method before_compile_update(query: Query, update_context: BulkUpdate) → None
```

允许在`Query.update()`中修改`Query`对象。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeQuery, 'before_compile_update')
def receive_before_compile_update(query, update_context):
    "listen for the 'before_compile_update' event"

    # ... (event handling logic) ...
```

自 1.4 版弃用：`QueryEvents.before_compile_update()`事件已被更加强大的`SessionEvents.do_orm_execute()`钩子所取代。

与`QueryEvents.before_compile()`事件类似，如果要使用该事件来修改`Query`对象，则应将其配置为`retval=True`，并返回修改后的`Query`对象，如下所示：

```py
@event.listens_for(Query, "before_compile_update", retval=True)
def no_deleted(query, update_context):
    for desc in query.column_descriptions:
        if desc['type'] is User:
            entity = desc['entity']
            query = query.filter(entity.deleted == False)

            update_context.values['timestamp'] = datetime.utcnow()
    return query
```

“更新上下文”对象的`.values`字典也可以像上面示例中那样就地修改。

参数：

+   `query` – 一个`Query`实例；这也是给定“更新上下文”对象的`.query`属性。

+   `update_context` – 一个“更新上下文”对象，与 `QueryEvents.after_bulk_update.update_context` 中描述的对象类型相同。对象具有在 UPDATE 上下文中的一个 `.values` 属性，该属性是传递给 `Query.update()` 的参数字典。此字典可以修改以更改结果 UPDATE 语句的 VALUES 子句。

版本 1.2.17 中的新功能。

另请参阅

`QueryEvents.before_compile()`

`QueryEvents.before_compile_delete()`

```py
attribute dispatch: _Dispatch[_ET] = <sqlalchemy.event.base.QueryEventsDispatch object>
```

回溯到 _Dispatch 类。

对 _Dispatch._events 进行双向处理

## 仪器化事件

定义了 SQLAlchemy 的类仪器化系统。

此模块通常不直接对用户应用程序可见，但定义了 ORM 交互的大部分内容。

instrumentation.py 处理最终用户类的注册以进行状态跟踪。它与 state.py 和 attributes.py 密切交互，分别为每个实例和每个类属性仪器化。

类仪器化系统可以使用 `sqlalchemy.ext.instrumentation` 模块在每个类或全局基础上进行自定义，该模块提供了构建和指定替代仪器化形式的方法。

| 对象名称 | 描述 |
| --- | --- |
| InstrumentationEvents | 与类仪器化事件相关的事件。 |

```py
class sqlalchemy.orm.InstrumentationEvents
```

与类仪器化事件相关的事件。

这里的监听器支持针对任何新样式类（即任何类型的子类为 'type'）建立，然后对该类的事件将被触发。如果将“propagate=True”标志传递给 event.listen()，则该事件也将对该类的子类触发。

Python 的 `type` 内建函数也被接受为目标，当使用时会导致为所有类发出事件。

注意这里的“propagate”标志默认为`True`，与其他类级事件不同，默认为`False`。这意味着当在超类上建立监听器时，新的子类也将成为这些事件的对象。

**成员**

attribute_instrument(), class_instrument(), class_uninstrument(), dispatch

**类签名**

类 `sqlalchemy.orm.InstrumentationEvents` (`sqlalchemy.event.Events`)

```py
method attribute_instrument(cls: ClassManager[_O], key: _KT, inst: _O) → None
```

当属性被仪器化时调用。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeBaseClass, 'attribute_instrument')
def receive_attribute_instrument(cls, key, inst):
    "listen for the 'attribute_instrument' event"

    # ... (event handling logic) ...
```

```py
method class_instrument(cls: ClassManager[_O]) → None
```

在给定类被检测之后调用。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeBaseClass, 'class_instrument')
def receive_class_instrument(cls):
    "listen for the 'class_instrument' event"

    # ... (event handling logic) ...
```

要获取`ClassManager`，请使用`manager_of_class()`。

```py
method class_uninstrument(cls: ClassManager[_O]) → None
```

在给定类被取消检测之前调用。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeBaseClass, 'class_uninstrument')
def receive_class_uninstrument(cls):
    "listen for the 'class_uninstrument' event"

    # ... (event handling logic) ...
```

要获取`ClassManager`，请使用`manager_of_class()`。

```py
attribute dispatch: _Dispatch[_ET] = <sqlalchemy.event.base.InstrumentationEventsDispatch object>
```

参考回 _Dispatch 类。

双向反对 _Dispatch._events
