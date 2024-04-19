# 状态管理

> 原文：[`docs.sqlalchemy.org/en/20/orm/session_state_management.html`](https://docs.sqlalchemy.org/en/20/orm/session_state_management.html)

## 对象状态简介

知道对象在会话中可能具有的状态是有帮助的：

+   **Transient** - 一个不在会话中的实例，也没有保存到数据库；即它没有数据库标识。这样的对象与 ORM 的唯一关系是其类与一个`Mapper`相关联。

+   **Pending** - 当您`Session.add()`一个瞬态实例时，它变为待定状态。它实际上还没有被刷新到数据库中，但在下一次刷新时会被刷新到数据库中。

+   **Persistent** - 存在于会话中并在数据库中具有记录的实例。您可以通过刷新使待定实例变为持久实例，或者通过查询数据库获取现有实例（或将其他会话中的持久实例移动到您的本地会话中）来获取持久实例。

+   **Deleted** - 在刷新中已被删除的实例，但事务尚未完成。处于此状态的对象基本上处于“待定”状态的相反状态；当会话的事务提交时，对象将移动到分离状态。或者，当会话的事务回滚时，删除的对象将*返回*到持久状态。

+   **Detached** - 一个对应于数据库中的记录，但目前不在任何会话中的实例。分离的对象将包含一个数据库标识标记，但是由于它没有与会话关联，因此无法确定此数据库标识是否实际存在于目标数据库中。分离的对象通常可以安全使用，但它们无法加载未加载的属性或先前标记为“过期”的属性。

深入了解所有可能的状态转换，请参阅对象生命周期事件部分，其中描述了每个转换以及如何以编程方式跟踪每个转换。

### 获取对象的当前状态

您可以随时使用`inspect()`函数在任何映射对象上查看实际状态；此函数将返回管理对象的内部 ORM 状态的相应`InstanceState`对象。`InstanceState`提供了其他访问器，包括指示对象持久状态的布尔属性，包括：

+   `InstanceState.transient`

+   `InstanceState.pending`

+   `InstanceState.persistent`

+   `InstanceState.deleted`

+   `InstanceState.detached`

例如：

```py
>>> from sqlalchemy import inspect
>>> insp = inspect(my_object)
>>> insp.persistent
True
```

另请参阅

映射实例的检查 - `InstanceState` 的更多示例  ## 会话属性

`Session` 本身在某种程度上就像一个集合。可以使用迭代器接口访问所有已存在的项目：

```py
for obj in session:
    print(obj)
```

并且可以使用常规的“包含”语义来测试存在性：

```py
if obj in session:
    print("Object is present")
```

会话还跟踪所有新创建的（即待处理的）对象，自上次加载或保存以来发生了更改的所有对象（即“脏对象”），以及标记为已删除的所有对象：

```py
# pending objects recently added to the Session
session.new

# persistent objects which currently have changes detected
# (this collection is now created on the fly each time the property is called)
session.dirty

# persistent objects that have been marked as deleted via session.delete(obj)
session.deleted

# dictionary of all persistent objects, keyed on their
# identity key
session.identity_map
```

（文档：`Session.new`, `Session.dirty`, `Session.deleted`, `Session.identity_map`).  ## 会话引用行为

会话内的对象是*弱引用*的。这意味着当它们在外部应用程序中取消引用时，它们也从`Session` 中消失，并且受 Python 解释器的垃圾收集影响。这种情况的例外包括待处理的对象、标记为已删除的对象或具有待处理更改的持久对象。在完全刷新后，这些集合都为空，并且所有对象再次成为弱引用。

为了使`Session`中的对象保持强引用，通常只需要一个简单的方法。外部管理强引用行为的示例包括将对象加载到以其主键为键的本地字典中，或者在它们需要保持引用的时间段内加载到列表或集合中。如果需要，这些集合可以与 `Session` 关联，方法是将它们放入 `Session.info` 字典中。

也可以采用基于事件的方法。一个简单的方法可以为所有对象在持久状态下保持“强引用”行为，具体如下：

```py
from sqlalchemy import event

def strong_reference_session(session):
    @event.listens_for(session, "pending_to_persistent")
    @event.listens_for(session, "deleted_to_persistent")
    @event.listens_for(session, "detached_to_persistent")
    @event.listens_for(session, "loaded_as_persistent")
    def strong_ref_object(sess, instance):
        if "refs" not in sess.info:
            sess.info["refs"] = refs = set()
        else:
            refs = sess.info["refs"]

        refs.add(instance)

    @event.listens_for(session, "persistent_to_detached")
    @event.listens_for(session, "persistent_to_deleted")
    @event.listens_for(session, "persistent_to_transient")
    def deref_object(sess, instance):
        sess.info["refs"].discard(instance)
```

在上面的示例中，我们拦截了 `SessionEvents.pending_to_persistent()`、`SessionEvents.detached_to_persistent()`、`SessionEvents.deleted_to_persistent()` 和 `SessionEvents.loaded_as_persistent()` 事件钩子，以便拦截对象在进入持久状态时的行为，并在对象离开持久状态时拦截 `SessionEvents.persistent_to_detached()` 和 `SessionEvents.persistent_to_deleted()` 事件钩子。

上述函数可用于任何 `Session`，以在每个`Session`基础上提供强引用行为：

```py
from sqlalchemy.orm import Session

my_session = Session()
strong_reference_session(my_session)
```

对于任何 `sessionmaker`，也可能被调用：

```py
from sqlalchemy.orm import sessionmaker

maker = sessionmaker()
strong_reference_session(maker)
```  ## 合并

`Session.merge()` 将状态从外部对象传输到会话中的新实例或已存在的实例。它还将传入的数据与数据库状态进行对比，生成一个历史流，该流将被应用于下一次刷新，或者可以被设置为生成简单的状态“传输”，而不生成变更历史或访问数据库。使用方法如下：

```py
merged_object = session.merge(existing_object)
```

给定一个实例时，它遵循以下步骤：

+   它检查实例的主键。如果存在，则尝试在本地标识映射中定位该实例。如果 `load=True` 标志保持默认设置，则还会检查数据库是否存在此主键，如果在本地找不到，则检查数据库是否存在此主键。

+   如果给定实例没有主键，或者给定的主键找不到实例，则创建一个新实例。

+   然后将给定实例的状态复制到定位的/新创建的实例上。对于源实例上存在的属性值，该值将转移到目标实例上。对于源实例上不存在的属性值，目标实例上的相应属性将从内存中过期，这会丢弃目标实例的该属性的任何本地存在值，但不会对该属性的数据库持久化值进行直接修改。

    如果 `load=True` 标志保持默认设置，此复制过程会发出事件，并且将为源对象上的每个属性加载目标对象的未加载集合，以便可以根据数据库中存在的内容来协调传入状态。如果传递 `load` 为 `False`，则传入的数据将直接“标记”，而不产生任何历史记录。

+   操作会根据 `merge` 级联（请参阅级联）传播到相关对象和集合。

+   返回新实例。

使用`Session.merge()`，给定的“源”实例不会被修改，也不会与目标`Session`关联，并且仍然可以与任意数量的其他`Session`对象合并。`Session.merge()`对于获取任何类型的对象结构的状态而无需考虑其来源或当前会话关联，并将其状态复制到新会话中非常有用。以下是一些示例：

+   从文件读取对象结构并希望将其保存到数据库的应用程序可能会解析文件，构建结构，然后使用`Session.merge()`将其保存到数据库中，确保文件中的数据用于构造结构的每个元素的主键。稍后，当文件发生更改时，可以重新运行相同的过程，生成稍微不同的对象结构，然后可以再次进行`merge`，并且`Session`将自动更新数据库以反映这些更改，通过主键从数据库加载每个对象，然后使用新状态更新其状态。

+   一个应用程序将对象存储在一个内存缓存中，由许多`Session`对象同时共享。每次从缓存中检索对象时，都会使用`Session.merge()`创建它的本地副本，以便在每个请求它的`Session`中。缓存的对象保持分离状态；只有它的状态被移动到本地于各个`Session`对象的副本中。

    在缓存用例中，通常使用`load=False`标志来消除对象状态与数据库之间的开销。还有一个“批量”版本的`Session.merge()`称为`Query.merge_result()`，它被设计用于与缓存扩展的`Query`对象一起使用 - 请参阅 Dogpile Caching 部分。

+   一个应用程序想要将一系列对象的状态转移到由工作线程或其他并发系统维护的`Session`中。`Session.merge()`将每个要放入新`Session`中的对象复制一份。操作结束时，父线程/进程保留了其开始的对象，而线程/工作程序可以继续使用这些对象的本地副本。

    在“线程/进程之间传输”用例中，应用程序可能希望同时使用`load=False`标志，以避免开销和冗余的 SQL 查询，因为数据正在传输。

### 合并提示

`Session.merge()`是一个非常有用的方法，适用于许多目的。然而，它处理的是瞬态/分离对象和持久化对象之间复杂的边界，以及状态的自动转移。这里可能出现的各种各样的场景通常需要对对象状态更加谨慎的处理。合并的常见问题通常涉及到传递给`Session.merge()`的对象的一些意外状态。

让我们以用户和地址对象的典型示例为例：

```py
class User(Base):
    __tablename__ = "user"

    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50), nullable=False)
    addresses = relationship("Address", backref="user")

class Address(Base):
    __tablename__ = "address"

    id = mapped_column(Integer, primary_key=True)
    email_address = mapped_column(String(50), nullable=False)
    user_id = mapped_column(Integer, ForeignKey("user.id"), nullable=False)
```

假设一个具有一个地址的`User`对象，已经持久化：

```py
>>> u1 = User(name="ed", addresses=[Address(email_address="ed@ed.com")])
>>> session.add(u1)
>>> session.commit()
```

现在我们创建了一个在会话之外的对象`a1`，我们希望将其合并到现有的`Address`上：

```py
>>> existing_a1 = u1.addresses[0]
>>> a1 = Address(id=existing_a1.id)
```

如果我们这样说会有一个意外的情况：

```py
>>> a1.user = u1
>>> a1 = session.merge(a1)
>>> session.commit()
sqlalchemy.orm.exc.FlushError: New instance <Address at 0x1298f50>
with identity key (<class '__main__.Address'>, (1,)) conflicts with
persistent instance <Address at 0x12a25d0>
```

为什么会这样？我们没有正确处理级联。将`a1.user`分配给持久对象级联到`User.addresses`的反向引用，并使我们的`a1`对象挂起，就好像我们已经添加了它一样。现在我们的会话中有*两个*`Address`对象：

```py
>>> a1 = Address()
>>> a1.user = u1
>>> a1 in session
True
>>> existing_a1 in session
True
>>> a1 is existing_a1
False
```

上面，我们的`a1`已经在会话中挂起。随后的`Session.merge()`操作实际上什么都不做。级联可以通过`relationship.cascade`选项在`relationship()`上配置，尽管在这种情况下，这意味着从`User.addresses`关系中删除了`save-update`级联 - 而且通常，那种行为非常方便。这里的解决方案通常是不将`a1.user`分配给已经存在于目标会话中的对象。

`relationship()`的`cascade_backrefs=False`选项也将阻止通过`a1.user = u1`分配将`Address`添加到会话中。

更多关于级联操作的细节请参阅级联。

另一个意外状态的例子：

```py
>>> a1 = Address(id=existing_a1.id, user_id=u1.id)
>>> a1.user = None
>>> a1 = session.merge(a1)
>>> session.commit()
sqlalchemy.exc.IntegrityError: (IntegrityError) address.user_id
may not be NULL
```

上面，将`user`的分配优先于`user_id`的外键分配，最终导致`user_id`应用了`None`，导致失败。

大多数`Session.merge()`问题可以通过首先检查 - 对象是否过早地在会话中？

```py
>>> a1 = Address(id=existing_a1, user_id=user.id)
>>> assert a1 not in session
>>> a1 = session.merge(a1)
```

或者对象上有我们不想要的状态吗？检查`__dict__`是一个快速检查的方法：

```py
>>> a1 = Address(id=existing_a1, user_id=user.id)
>>> a1.user
>>> a1.__dict__
{'_sa_instance_state': <sqlalchemy.orm.state.InstanceState object at 0x1298d10>,
 'user_id': 1,
 'id': 1,
 'user': None}
>>> # we don't want user=None merged, remove it
>>> del a1.user
>>> a1 = session.merge(a1)
>>> # success
>>> session.commit()
```

## 清除

Expunge 将对象从会话中删除，将持久实例发送到脱机状态，将待处理实例发送到瞬态状态：

```py
session.expunge(obj1)
```

要删除所有项目，请调用`Session.expunge_all()`（此方法以前称为`clear()`）。

## 刷新 / 过期

过期意味着数据库持久化数据存储在一系列对象属性中被清除，这样当下次访问这些属性时，将发出一个 SQL 查询，该查询将从数据库中刷新数据。

当我们谈论数据的过期时，我们通常是指处于持久状态的对象。例如，如果我们像这样加载一个对象：

```py
user = session.scalars(select(User).filter_by(name="user1").limit(1)).first()
```

上述的`User`对象是持久的，并且具有一系列存在的属性；如果我们查看它的`__dict__`，我们会看到已加载的状态：

```py
>>> user.__dict__
{
 'id': 1, 'name': u'user1',
 '_sa_instance_state': <...>,
}
```

其中`id`和`name`是数据库中的列。`_sa_instance_state`是 SQLAlchemy 内部使用的非数据库持久化值（它引用了实例的`InstanceState`。虽然与本节直接相关，但如果我们想要获取它，我们应该使用`inspect()`函数来访问它）。

此时，我们`User`对象中的状态与加载的数据库行的状态相匹配。但是在使用诸如`Session.expire()`之类的方法使对象过期后，我们会看到状态被删除：

```py
>>> session.expire(user)
>>> user.__dict__
{'_sa_instance_state': <...>}
```

我们看到，虽然内部的“状态”仍然存在，但与`id`和`name`列对应的值已经消失。如果我们要访问其中一列并观察 SQL，我们会看到这样的情况：

```py
>>> print(user.name)
SELECT  user.id  AS  user_id,  user.name  AS  user_name
FROM  user
WHERE  user.id  =  ?
(1,)
user1
```

上面，在访问已过期的属性`user.name`时，ORM 启动了一个惰性加载以从数据库中检索最新状态，通过向这个用户引用的用户行发出一个 SELECT。之后，`__dict__`再次被填充：

```py
>>> user.__dict__
{
 'id': 1, 'name': u'user1',
 '_sa_instance_state': <...>,
}
```

注意

虽然我们正在查看`__dict__`的内容，以便了解 SQLAlchemy 对对象属性的处理，但我们**不应直接修改**`__dict__`的内容，至少不应修改 SQLAlchemy ORM 正在维护的属性（SQLA 领域之外的其他属性没问题）。这是因为 SQLAlchemy 使用描述符来跟踪我们对对象所做的更改，当我们直接修改`__dict__`时，ORM 将无法跟踪到我们做出的更改。

`Session.expire()`和`Session.refresh()`的另一个关键行为是，对象上的所有未刷新的更改都将被丢弃。也就是说，如果我们要修改`User`上的属性：

```py
>>> user.name = "user2"
```

但是当我们在调用`Session.expire()`之前没有调用`Session.flush()`时，我们挂起的值`'user2'`将被丢弃：

```py
>>> session.expire(user)
>>> user.name
'user1'
```

`Session.expire()`方法可用于将实例的所有 ORM 映射属性标记为“过期”：

```py
# expire all ORM-mapped attributes on obj1
session.expire(obj1)
```

它还可以传递一个字符串属性名称列表，指定要标记为过期的特定属性：

```py
# expire only attributes obj1.attr1, obj1.attr2
session.expire(obj1, ["attr1", "attr2"])
```

`Session.expire_all()`方法允许我们一次性对`Session`中包含的所有对象调用`Session.expire()`：

```py
session.expire_all()
```

`Session.refresh()`方法具有类似的接口，但是不是使过期，而是立即发出对象行的 SELECT：

```py
# reload all attributes on obj1
session.refresh(obj1)
```

`Session.refresh()`还接受一个字符串属性名称的列表，但与`Session.expire()`不同，它期望至少一个名称是列映射属性的名称：

```py
# reload obj1.attr1, obj1.attr2
session.refresh(obj1, ["attr1", "attr2"])
```

提示

通常更灵活的刷新替代方法是使用 ORM 的填充现有内容功能，适用于使用`select()`进行 2.0 风格查询以及在 1.x 风格查询中的`Query.populate_existing()`方法。使用此执行选项，语句结果集中返回的所有 ORM 对象都将使用来自数据库的数据进行刷新：

```py
stmt = (
    select(User)
    .execution_options(populate_existing=True)
    .where((User.name.in_(["a", "b", "c"])))
)
for user in session.execute(stmt).scalars():
    print(user)  # will be refreshed for those columns that came back from the query
```

查看填充现有内容以获取更多详细信息。

### 实际加载内容

当标记为`Session.expire()`或使用`Session.refresh()`加载的对象时，所发出的 SELECT 语句因多种因素而异，包括：

+   仅从**列映射属性**加载过期属性。虽然可以将任何类型的属性标记为过期，包括`relationship()` - 映射属性，但访问过期的`relationship()`属性将仅为该属性发出加载，使用标准的基于关系的惰性加载。即使过期，基于列的属性也不会作为此操作的一部分加载，而是在访问任何基于列的属性时加载。

+   通过 `relationship()` 映射的属性不会在访问过期的基于列的属性时加载。

+   关于关系，`Session.refresh()` 在属性不是列映射的情况下比 `Session.expire()` 更为严格。调用 `Session.refresh()` 并传递一个只包括关系映射属性的名称列表将会引发错误。无论如何，非急切加载的 `relationship()` 属性都不会包含在任何刷新操作中。

+   通过 `relationship.lazy` 参数配置为“急切加载”的 `relationship()` 属性将在 `Session.refresh()` 的情况下加载，如果未指定任何属性名称，或者如果它们的名称包含在要刷新的属性列表中。

+   配置为 `deferred()` 的属性通常不会在过期属性加载期间或刷新期间加载。当直接访问未加载的属性或者作为延迟属性组的一部分访问该组中的未加载属性时，配置为 `deferred()` 的未加载属性将自行加载。

+   对于在访问时加载的过期属性，连接继承表映射将发出一个通常只包含那些存在未加载属性的表的 SELECT。在这里的操作足够复杂，以仅加载父表或子表，例如，如果最初过期的列的子集仅包含其中一个表或另一个表。

+   当在连接继承表映射上使用 `Session.refresh()` 时，所发出的 SELECT 与在目标对象的类上使用 `Session.query()` 时的类似。这通常是映射的一部分设置的所有表。

### 何时过期或刷新

`Session` 在会话引用的事务结束时自动使用过期功能。这意味着，每当调用 `Session.commit()` 或 `Session.rollback()`，会话中的所有对象都会过期，使用与 `Session.expire_all()` 方法相当的功能。其原因在于事务的结束是一个标志性的点，在此点上不再有可用于了解数据库当前状态的上下文，因为任意数量的其他事务可能正在影响它。只有当新事务开始时，我们才能再次访问数据库的当前状态，在此时可能已经发生了任意数量的更改。

当希望强制对象重新从数据库加载其数据时，应使用 `Session.expire()` 和 `Session.refresh()` 方法，当已知数据的当前状态可能过时时。这样做的原因可能包括：

+   一些 SQL 在 ORM 对象处理范围之外的事务中被发出，比如如果使用 `Session.execute()` 方法发出了 `Table.update()` 构造；

+   如果应用程序试图获取已知在并发事务中已修改的数据，并且已知正在生效的隔离规则允许该数据可见。

第二条警告很重要，即“也已知在生效的隔离规则下，这些数据可见。”这意味着不能假设在另一个数据库连接上发生的更新在本地已经可见；在许多情况下，它是不可见的。这就是为什么如果想要在正在进行的事务之间使用 `Session.expire()` 或 `Session.refresh()` 查看数据，就必须了解正在生效的隔离行为。

另见

`Session.expire()`

`Session.expire_all()` 

`Session.refresh()`

填充现有 - 允许任何 ORM 查询在 SELECT 语句的结果中刷新对象，就像它们通常加载一样，刷新标识映射中所有匹配的对象。

隔离 - 隔离的词汇解释，其中包括指向维基百科的链接。

[SQLAlchemy 会话深入解析](https://techspot.zzzeek.org/2012/11/14/pycon-canada-the-sqlalchemy-session-in-depth/) - 一个关于对象生命周期的深入讨论的视频 + 幻灯片，包括数据过期的角色。## 快速对象状态介绍

了解实例在会话中可能具有的状态是有帮助的：

+   **瞬时** - 一个不在会话中并且没有保存到数据库的实例；即它没有数据库标识。这样的对象与 ORM 的唯一关系是其类与一个 `Mapper` 相关联。

+   **待定** - 当你 `Session.add()` 一个瞬时实例时，它变为待定状态。它实际上还没有被刷新到数据库，但在下一次刷新时会被刷新到数据库。

+   **持久** - 存在于会话中并且在数据库中有记录的实例。您可以通过刷新使待定实例变为持久实例，或通过查询数据库获取现有实例（或将其他会话中的持久实例移动到您的本地会话）来获得持久实例。

+   **已删除** - 在刷新中已删除的实例，但事务尚未完成。处于这种状态的对象基本上与“待定”状态相反；当会话的事务提交时，对象将移至分离状态。另外，当会话的事务回滚时，已删除的对象将*回到*持久状态。

+   **分离** - 一个实例，它对应于或以前对应于数据库中的记录，但当前不在任何会话中。分离的对象将包含一个数据库标识标记，但由于它没有关联到会话，因此不知道此数据库标识实际上是否存在于目标数据库中。分离的对象通常是安全的使用，除了它们无法加载未加载的属性或以前标记为“过期”的属性。

深入研究所有可能的状态转换，请参阅 对象生命周期事件 部分，该部分描述了每个转换以及如何以编程方式跟踪每个转换。

### 获取对象的当前状态

任何映射对象的实际状态都可以随时使用映射实例上的 `inspect()` 函数查看；此函数将返回管理对象的内部 ORM 状态的相应 `InstanceState` 对象。`InstanceState` 提供了其他访问器，其中包括指示对象持久性状态的布尔属性，包括：

+   `InstanceState.transient`

+   `InstanceState.pending`

+   `InstanceState.persistent`

+   `InstanceState.deleted`

+   `InstanceState.detached`

例如：

```py
>>> from sqlalchemy import inspect
>>> insp = inspect(my_object)
>>> insp.persistent
True
```

另请参阅

映射实例的检查 - 更多有关 `InstanceState` 的示例

### 获取对象的当前状态

任何映射对象的实际状态都可以随时使用映射实例上的 `inspect()` 函数查看；此函数将返回管理对象的内部 ORM 状态的相应 `InstanceState` 对象。`InstanceState` 提供了其他访问器，其中包括指示对象持久性状态的布尔属性，包括：

+   `InstanceState.transient`

+   `InstanceState.pending`

+   `InstanceState.persistent`

+   `InstanceState.deleted`

+   `InstanceState.detached`

例如：

```py
>>> from sqlalchemy import inspect
>>> insp = inspect(my_object)
>>> insp.persistent
True
```

另请参阅

映射实例的检查 - 更多有关 `InstanceState` 的示例

## 会话属性

`Session` 本身的行为有点像一个类似集合的集合。可以使用迭代器接口访问所有存在的项目：

```py
for obj in session:
    print(obj)
```

并且可以使用常规的“包含”语义进行测试：

```py
if obj in session:
    print("Object is present")
```

会话还会跟踪所有新创建的（即待处理的）对象，所有自上次加载或保存以来发生更改的对象（即“脏对象”），以及所有被标记为已删除的对象：

```py
# pending objects recently added to the Session
session.new

# persistent objects which currently have changes detected
# (this collection is now created on the fly each time the property is called)
session.dirty

# persistent objects that have been marked as deleted via session.delete(obj)
session.deleted

# dictionary of all persistent objects, keyed on their
# identity key
session.identity_map
```

（文档：`Session.new`，`Session.dirty`，`Session.deleted`，`Session.identity_map`）。

## 会话引用行为

会话中的对象是*弱引用*的。这意味着当它们在外部应用程序中取消引用时，它们也会从`Session`中失去作用域，并且由 Python 解释器进行垃圾回收。这种情况的例外包括待处理对象、标记为已删除的对象或具有待处理更改的持久对象。在完全刷新后，这些集合都为空，并且所有对象再次是弱引用的。

使`Session`中的对象保持强引用通常只需要简单的方法。外部管理的强引用行为示例包括将对象加载到以其主键为键的本地字典中，或者在它们需要保持引用的时间段内加载到列表或集合中。如果需要，这些集合可以与`Session`关联，方法是将它们放入`Session.info`字典中。

还可以使用基于事件的方法。以下是一个提供了所有对象在持久化状态下保持“强引用”行为的简单方案：

```py
from sqlalchemy import event

def strong_reference_session(session):
    @event.listens_for(session, "pending_to_persistent")
    @event.listens_for(session, "deleted_to_persistent")
    @event.listens_for(session, "detached_to_persistent")
    @event.listens_for(session, "loaded_as_persistent")
    def strong_ref_object(sess, instance):
        if "refs" not in sess.info:
            sess.info["refs"] = refs = set()
        else:
            refs = sess.info["refs"]

        refs.add(instance)

    @event.listens_for(session, "persistent_to_detached")
    @event.listens_for(session, "persistent_to_deleted")
    @event.listens_for(session, "persistent_to_transient")
    def deref_object(sess, instance):
        sess.info["refs"].discard(instance)
```

上述，我们拦截了`SessionEvents.pending_to_persistent()`，`SessionEvents.detached_to_persistent()`，`SessionEvents.deleted_to_persistent()`和`SessionEvents.loaded_as_persistent()`事件钩子，以拦截对象进入 persistent 状态转换时的情况，以及`SessionEvents.persistent_to_detached()`和`SessionEvents.persistent_to_deleted()`钩子以拦截对象离开持久状态时的情况。

上述函数可针对任何`Session`进行调用，以在每个`Session`上提供强引用行为：

```py
from sqlalchemy.orm import Session

my_session = Session()
strong_reference_session(my_session)
```

也可以针对任何`sessionmaker`进行调用：

```py
from sqlalchemy.orm import sessionmaker

maker = sessionmaker()
strong_reference_session(maker)
```

## 合并

`Session.merge()`将外部对象的状态转移到会话中的新实例或已存在的实例中。 它还将传入的数据与数据库状态进行对比，生成将应用于下一个刷新的历史流，或者可以产生简单的状态“转移”而不产生更改历史或访问数据库。 使用方法如下：

```py
merged_object = session.merge(existing_object)
```

当给定一个实例时，它按以下步骤执行：

+   它检查实例的主键。 如果存在，它会尝试在本地标识映射中定位该实例。 如果将`load=True`标志保留为其默认值，则还会检查数据库以获取该主键（如果未在本地找到）。

+   如果给定实例没有主键，或者无法找到给定主键的实例，则会创建一个新实例。

+   然后，给定实例的状态将被复制到定位/新创建的实例上。对于源实例上存在的属性值，该值将转移到目标实例。对于源实例上不存在的属性值，目标实例上的相应属性将从内存中过期，这将丢弃目标实例的该属性的任何局部存在值，但不会直接修改该属性的数据库持久化值。

    如果`load=True`标志保持其默认状态，则此复制过程会触发事件，并且将为源对象上的每个属性加载目标对象的未加载集合，以便对入站状态进行与数据库中存在的内容进行对比。如果传递了`load`为`False`，则传入的数据将直接“盖章”，而不产生任何历史记录。

+   操作会级联到相关对象和集合，如`merge`级联所示（见级联）。

+   返回新实例。

使用`Session.merge()`，给定的“源”实例不会被修改，也不会与目标`Session`关联，并且仍然可用于与任意数量的其他`Session`对象合并。`Session.merge()`对于将任何类型的对象结构的状态复制到新会话中而不考虑其来源或当前会话关联很有用。以下是一些示例：

+   从文件读取对象结构并希望将其保存到数据库的应用程序可能会解析文件，构建结构，然后使用`Session.merge()`将其保存到数据库，确保使用文件中的数据来制定结构的每个元素的主键。稍后，当文件发生更改时，可以重新运行相同的过程，生成稍微不同的对象结构，然后可以再次进行合并，并且`Session`将自动更新数据库以反映这些更改，通过主键从数据库加载每个对象，然后使用给定的新状态更新其状态。

+   一个应用程序将对象存储在一个内存缓存中，被许多`Session`对象同时共享。每次从缓存中检索对象时，都会使用`Session.merge()`在请求该对象的每个`Session`中创建一个本地副本。缓存的对象保持分离状态；只有其状态被移动到各自局部的`Session`对象的副本中。

    在缓存用例中，通常使用`load=False`标志来消除对象状态与数据库之间的协调开销。还有一个与缓存扩展`Query`对象一起工作的“批量”版本的`Session.merge()`，称为`Query.merge_result()`，请参见 Dogpile Caching 部分。

+   一个应用程序希望将一系列对象的状态转移到由工作线程或其他并发系统维护的`Session`中。`Session.merge()`会为要放入这个新`Session`的每个对象创建一个副本。操作结束时，父线程/进程保留其开始的对象，而线程/工作线程可以继续使用这些对象的本地副本。

    在“线程/进程之间传输”用例中，应用程序可能还想使用`load=False`标志，以避免在数据传输时产生额外开销和冗余的 SQL 查询。

### 合并提示

`Session.merge()`是一个非常有用的方法，适用于许多目的。然而，它处理的是瞬态/分离对象和持久对象之间复杂的边界，以及状态的自动传输。这里可能出现的各种场景通常需要更加谨慎地处理对象的状态。合并常见问题通常涉及传递给`Session.merge()`的对象的一些意外状态。

让我们使用用户和地址对象的典型示例：

```py
class User(Base):
    __tablename__ = "user"

    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50), nullable=False)
    addresses = relationship("Address", backref="user")

class Address(Base):
    __tablename__ = "address"

    id = mapped_column(Integer, primary_key=True)
    email_address = mapped_column(String(50), nullable=False)
    user_id = mapped_column(Integer, ForeignKey("user.id"), nullable=False)
```

假设一个具有一个`Address`的`User`对象，已经是持久的：

```py
>>> u1 = User(name="ed", addresses=[Address(email_address="ed@ed.com")])
>>> session.add(u1)
>>> session.commit()
```

现在我们创建`a1`，一个在会话之外的对象，我们希望将其合并到现有的`Address`上：

```py
>>> existing_a1 = u1.addresses[0]
>>> a1 = Address(id=existing_a1.id)
```

如果我们这样说，将会出现一个意外情况：

```py
>>> a1.user = u1
>>> a1 = session.merge(a1)
>>> session.commit()
sqlalchemy.orm.exc.FlushError: New instance <Address at 0x1298f50>
with identity key (<class '__main__.Address'>, (1,)) conflicts with
persistent instance <Address at 0x12a25d0>
```

为什么会这样？我们没有仔细处理级联。将`a1.user`分配给持久对象，级联到了`User.addresses`的反向引用，并使我们的`a1`对象挂起，就像我们已经将其添加一样。现在我们在会话中有*两个*`Address`对象：

```py
>>> a1 = Address()
>>> a1.user = u1
>>> a1 in session
True
>>> existing_a1 in session
True
>>> a1 is existing_a1
False
```

在上面的例子中，我们的`a1`已经在会话中处于挂起状态。随后的`Session.merge()`操作实际上什么也没做。级联可以通过`relationship()`上的`relationship.cascade`选项进行配置，尽管在这种情况下，这意味着从`User.addresses`关系中删除`save-update`级联 - 通常，这种行为非常方便。在这里的解决方案通常是不将`a1.user`分配给目标会话中已经持久化的对象。

`relationship()`的`cascade_backrefs=False`选项还将通过`a1.user = u1`赋值阻止将`Address`添加到会话中。

有关级联操作的进一步细节，请参见 Cascades。

另一个意外状态的例子：

```py
>>> a1 = Address(id=existing_a1.id, user_id=u1.id)
>>> a1.user = None
>>> a1 = session.merge(a1)
>>> session.commit()
sqlalchemy.exc.IntegrityError: (IntegrityError) address.user_id
may not be NULL
```

在上面的例子中，`user`的赋值优先于`user_id`的外键赋值，其最终结果是将`None`应用于`user_id`，导致失败。

大多数`Session.merge()`问题可以通过首先检查对象是否过早出现在会话中来检查。

```py
>>> a1 = Address(id=existing_a1, user_id=user.id)
>>> assert a1 not in session
>>> a1 = session.merge(a1)
```

或者对象上有我们不想要的状态吗？检查`__dict__`是快速检查的一种方式：

```py
>>> a1 = Address(id=existing_a1, user_id=user.id)
>>> a1.user
>>> a1.__dict__
{'_sa_instance_state': <sqlalchemy.orm.state.InstanceState object at 0x1298d10>,
 'user_id': 1,
 'id': 1,
 'user': None}
>>> # we don't want user=None merged, remove it
>>> del a1.user
>>> a1 = session.merge(a1)
>>> # success
>>> session.commit()
```

### 合并提示

`Session.merge()`对于许多目的非常有用。但是，它处理的是瞬态/游离对象与持久对象之间复杂的边界，以及状态的自动转移。这里可能出现的各种各样的场景通常需要更加谨慎的对象状态处理。与合并相关的常见问题通常涉及到传递给`Session.merge()`的对象的一些意外状态。

让我们使用用户和地址对象的典型示例：

```py
class User(Base):
    __tablename__ = "user"

    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50), nullable=False)
    addresses = relationship("Address", backref="user")

class Address(Base):
    __tablename__ = "address"

    id = mapped_column(Integer, primary_key=True)
    email_address = mapped_column(String(50), nullable=False)
    user_id = mapped_column(Integer, ForeignKey("user.id"), nullable=False)
```

假设已经持久化了一个具有一个`Address`的`User`对象：

```py
>>> u1 = User(name="ed", addresses=[Address(email_address="ed@ed.com")])
>>> session.add(u1)
>>> session.commit()
```

现在我们创建了一个在会话之外的对象`a1`，我们希望将其合并到现有的`Address`上：

```py
>>> existing_a1 = u1.addresses[0]
>>> a1 = Address(id=existing_a1.id)
```

如果我们这样说，就会出现意外的情况：

```py
>>> a1.user = u1
>>> a1 = session.merge(a1)
>>> session.commit()
sqlalchemy.orm.exc.FlushError: New instance <Address at 0x1298f50>
with identity key (<class '__main__.Address'>, (1,)) conflicts with
persistent instance <Address at 0x12a25d0>
```

为什么会这样？我们没有小心处理级联。将 `a1.user` 分配给持久对象级联到了 `User.addresses` 的反向引用，并使我们的 `a1` 对象处于待定状态，就好像我们已经将其添加了一样。现在我们会话中有 *两个* `Address` 对象。

```py
>>> a1 = Address()
>>> a1.user = u1
>>> a1 in session
True
>>> existing_a1 in session
True
>>> a1 is existing_a1
False
```

在上面，我们的 `a1` 已经在会话中处于待定状态。随后的 `Session.merge()` 操作实质上什么也不做。级联可以通过 `relationship()` 上的 `relationship.cascade` 选项进行配置，尽管在这种情况下，它意味着从 `User.addresses` 关系中删除了 `save-update` 级联 - 通常，这种行为非常方便。在这种情况下，解决方案通常是不将 `a1.user` 分配给目标会话中已经持久的对象。

`relationship()` 的 `cascade_backrefs=False` 选项还会防止通过 `a1.user = u1` 赋值将 `Address` 添加到会话中。

关于级联操作的进一步细节请参见 级联。

另一个意外状态的例子：

```py
>>> a1 = Address(id=existing_a1.id, user_id=u1.id)
>>> a1.user = None
>>> a1 = session.merge(a1)
>>> session.commit()
sqlalchemy.exc.IntegrityError: (IntegrityError) address.user_id
may not be NULL
```

在上面，`user` 的赋值优先于 `user_id` 的外键赋值，结果是 `None` 被应用于 `user_id`，导致失败。

大多数 `Session.merge()` 问题可以通过首先检查对象是否过早出现在会话中来检查。

```py
>>> a1 = Address(id=existing_a1, user_id=user.id)
>>> assert a1 not in session
>>> a1 = session.merge(a1)
```

或者对象上是否有我们不希望的状态？检查 `__dict__` 是一种快速检查的方法：

```py
>>> a1 = Address(id=existing_a1, user_id=user.id)
>>> a1.user
>>> a1.__dict__
{'_sa_instance_state': <sqlalchemy.orm.state.InstanceState object at 0x1298d10>,
 'user_id': 1,
 'id': 1,
 'user': None}
>>> # we don't want user=None merged, remove it
>>> del a1.user
>>> a1 = session.merge(a1)
>>> # success
>>> session.commit()
```

## Expunging

Expunge 从会话中移除一个对象，将持久实例发送到分离状态，将待定实例发送到瞬态状态：

```py
session.expunge(obj1)
```

要删除所有项目，请调用 `Session.expunge_all()`（此方法以前称为 `clear()`）。

## 刷新 / 到期

到期 意味着数据库持久化的数据被擦除，当下次访问这些属性时，会发出一个 SQL 查询，该查询将从数据库刷新该数据。

当我们谈论数据的到期时，通常是指处于 持久 状态的对象。例如，如果我们加载一个对象如下：

```py
user = session.scalars(select(User).filter_by(name="user1").limit(1)).first()
```

上述的 `User` 对象是持久的，并且具有一系列属性；如果我们查看它的 `__dict__`，我们会看到加载的状态：

```py
>>> user.__dict__
{
 'id': 1, 'name': u'user1',
 '_sa_instance_state': <...>,
}
```

其中 `id` 和 `name` 指的是数据库中的那些列。 `_sa_instance_state` 是 SQLAlchemy 内部使用的非数据库持久化值（它指的是实例的`InstanceState`。虽然与本节无直接关系，但如果我们想访问它，应该使用`inspect()`函数来访问它）。

此时，我们的 `User` 对象中的状态与加载的数据库行的状态相匹配。但是在使用诸如`Session.expire()`这样的方法使对象过期后，我们发现状态已被删除：

```py
>>> session.expire(user)
>>> user.__dict__
{'_sa_instance_state': <...>}
```

我们发现，虽然内部的“状态”仍然存在，但对应于 `id` 和 `name` 列的值已经消失了。如果我们尝试访问其中一个列，并且正在观察 SQL，我们会看到这样的情况：

```py
>>> print(user.name)
SELECT  user.id  AS  user_id,  user.name  AS  user_name
FROM  user
WHERE  user.id  =  ?
(1,)
user1
```

上面，访问过期属性 `user.name` 后，ORM 启动了一个延迟加载以从数据库中检索最新的状态，通过发出一个 SELECT 来检索与此用户相关联的用户行。然后，`__dict__` 再次被填充：

```py
>>> user.__dict__
{
 'id': 1, 'name': u'user1',
 '_sa_instance_state': <...>,
}
```

注意

当我们窥视 `__dict__` 以了解 SQLAlchemy 处理对象属性的一些情况时，我们**不应直接修改** `__dict__` 的内容，至少在 SQLAlchemy ORM 维护的属性方面不应该这样做（SQLA 领域之外的其他属性没问题）。这是因为 SQLAlchemy 使用描述符来跟踪我们对对象所做的更改，当我们直接修改 `__dict__` 时，ORM 将无法跟踪到我们做了什么更改。

`Session.expire()`和`Session.refresh()`的另一个关键行为是，对象上的所有未刷新的更改都将被丢弃。也就是说，如果我们要修改我们的 `User` 上的属性：

```py
>>> user.name = "user2"
```

但是当我们在首先调用`Session.flush()`之前调用`Session.expire()`时，我们挂起的值 `'user2'` 就会被丢弃：

```py
>>> session.expire(user)
>>> user.name
'user1'
```

`Session.expire()`方法可用于标记实例的所有 ORM 映射属性为“过期”状态：

```py
# expire all ORM-mapped attributes on obj1
session.expire(obj1)
```

它还可以传递一个字符串属性名称的列表，指示特定属性将被标记为过期：

```py
# expire only attributes obj1.attr1, obj1.attr2
session.expire(obj1, ["attr1", "attr2"])
```

`Session.expire_all()` 方法允许我们一次性对 `Session` 中包含的所有对象调用 `Session.expire()`：

```py
session.expire_all()
```

`Session.refresh()` 方法具有类似的接口，但不是过期，而是立即发出对象行的 SELECT：

```py
# reload all attributes on obj1
session.refresh(obj1)
```

`Session.refresh()` 也接受一个字符串属性名称列表，但与 `Session.expire()` 不同，它希望至少有一个名称是列映射属性的名称：

```py
# reload obj1.attr1, obj1.attr2
session.refresh(obj1, ["attr1", "attr2"])
```

提示

通常更灵活的刷新方法是使用 ORM 的 Populate Existing 功能，该功能适用于具有 `select()` 的 2.0 样式 查询以及 `Query.populate_existing()` 方法的 1.x 样式 查询。使用这种执行选项，语句结果集中返回的所有 ORM 对象都将使用数据库中的数据进行刷新：

```py
stmt = (
    select(User)
    .execution_options(populate_existing=True)
    .where((User.name.in_(["a", "b", "c"])))
)
for user in session.execute(stmt).scalars():
    print(user)  # will be refreshed for those columns that came back from the query
```

有关详细信息，请参见 Populate Existing。

### 实际加载的内容

当标记为 `Session.expire()` 或使用 `Session.refresh()` 加载的对象时，发出的 SELECT 语句基于几个因素而变化：

+   过期属性的加载仅从**列映射属性**触发。虽然可以将任何类型的属性标记为过期，包括 `relationship()` - 映射属性，但访问过期的 `relationship()` 属性将仅为该属性发出加载，使用标准的关联导向延迟加载。即使已过期，列导向属性也不会作为此操作的一部分加载，而是在访问任何列导向属性时加载。

+   响应于访问过期的基于列的属性时，`relationship()`-映射属性不会加载。

+   关于关系，与 `Session.expire()` 相比，`Session.refresh()` 对于未映射到列的属性更加严格。调用 `Session.refresh()` 并传递仅包括关系映射属性的名称列表实际上会引发错误。无论如何，非“eager loading” `relationship()` 属性都不会包含在任何刷新操作中。

+   `relationship()` 属性通过 `relationship.lazy` 参数配置为“eager loading”时，如果未指定属性名称，或者属性名称包含在要刷新的属性列表中，则在 `Session.refresh()` 的情况下加载。

+   配置为 `deferred()` 的属性通常不会在过期属性加载或刷新期间加载。当直接访问未加载的属性时，或者如果未加载的属性是一组“deferred”属性的一部分，其中访问了该组中的未加载属性，则`deferred()` 属性将自行加载。

+   对于在访问时加载的过期属性，连接继承表映射将发出一个 SELECT，该 SELECT 通常仅包括那些存在未加载属性的表。在这里的操作足够复杂，可以仅加载父表或子表，例如，如果最初过期的列的子集仅涵盖这两个表中的一个。

+   当在连接继承表映射上使用 `Session.refresh()` 时，发出的 SELECT 将类似于在目标对象的类上使用 `Session.query()` 时的情况。这通常是作为映射的一部分设置的所有表。

### 何时过期或刷新

`Session` 在会话结束时自动使用到期特性。意思是，每当调用 `Session.commit()` 或 `Session.rollback()` 时，`Session` 中的所有对象都会过期，使用与 `Session.expire_all()` 方法相当的特性。其理由是事务的结束是一个界定点，在此之后没有更多的上下文可用于了解数据库的当前状态，因为可能有任意数量的其他事务正在影响它。只有在新事务开始时，我们才能再次访问数据库的当前状态，在这一点上可能发生了任意数量的更改。

当希望强制对象从数据库重新加载其数据时，使用 `Session.expire()` 和 `Session.refresh()` 方法，在这些情况下，已知当前数据可能过时。 这样做的原因可能包括：

+   在 ORM 对象处理范围之外的事务内发出了一些 SQL，例如，如果使用 `Session.execute()` 方法发出了 `Table.update()` 构造；

+   如果应用程序试图获取在并发事务中已知已被修改的数据，并且已知正在生效的隔离规则允许将此数据可见。

第二条要点是“已知正在生效的隔离规则允许将此数据可见。” 这意味着不能假设在另一个数据库连接上发生的 UPDATE 在本地这里就可见；在许多情况下，不会是这样。 这就是为什么如果希望在正在进行的事务之间使用 `Session.expire()` 或 `Session.refresh()` 来查看数据，对生效的隔离行为有所了解是必要的原因。

另请参阅

`Session.expire()`

`Session.expire_all()`

`Session.refresh()`

填充现有对象 - 允许任何 ORM 查询像通常加载对象一样刷新对象，针对与 SELECT 语句的结果相匹配的所有匹配对象在标识映射中进行刷新。

隔离 - 隔离的词汇解释，其中包括指向维基百科的链接。

[SQLAlchemy 会话深入解析](https://techspot.zzzeek.org/2012/11/14/pycon-canada-the-sqlalchemy-session-in-depth/) - 一个视频+幻灯片，深入讨论对象生命周期，包括数据过期的作用。

### 实际加载情况

当一个对象被标记为`Session.expire()`或通过`Session.refresh()`加载时，所发出的 SELECT 语句会基于几个因素而变化，包括：

+   过期属性的加载仅仅是从**基于列的属性**触发的。虽然任何类型的属性都可以被标记为过期，包括`relationship()` - 映射属性，但访问过期的`relationship()`属性将仅对该属性进行加载，使用标准的基于关系的惰性加载。基于列的属性，即使过期，也不会作为此操作的一部分加载，而是在访问任何基于列的属性时加载。

+   `relationship()`- 映射属性在访问到过期的基于列的属性时不会加载。

+   关于关系，`Session.refresh()`比`Session.expire()`更严格，因为它不会加载那些非列映射的属性。调用`Session.refresh()`并传递一个只包括关系映射属性的名称列表将会引发错误。在任何情况下，非急加载的`relationship()`属性都不会包含在任何刷新操作中。

+   通过`relationship.lazy`参数配置为“急加载”的`relationship()`属性在`Session.refresh()`的情况下会加载，如果未指定属性名称，或者如果它们的名称包含在要刷新的属性列表中。

+   配置为`deferred()`的属性通常不会在过期属性加载或刷新期间加载。一个未加载的属性如果是`deferred()`，那么当直接访问时或者作为“组”中的未加载属性之一在组中的其他未加载属性被访问时会加载。

+   对于在访问时加载的已过期属性，一个联合继承表映射将会发出一个 SELECT 语句，通常只包括那些存在未加载属性的表。这里的操作足够复杂，可以只加载父表或子表，例如，如果原始已过期的列子集仅涵盖其中一个表。

+   当在联合继承表映射上使用`Session.refresh()`时，所发出的 SELECT 将类似于在目标对象的类上使用`Session.query()`时的情况。这通常是设置为映射的一部分的所有表。

### 何时过期或刷新

当`Session`引用的事务结束时，会自动使用到期特性。这意味着，无论何时调用`Session.commit()`或`Session.rollback()`，`Session`中的所有对象都将过期，使用与`Session.expire_all()`方法相当的特性。其理由在于，事务结束是一个标记点，此时不再有上下文可用以知晓数据库的当前状态，因为任何数量的其他事务可能正在影响它。只有当新事务启动时，我们才能再次访问数据库的当前状态，在这时可能已经发生了任意数量的更改。

当希望强制对象从数据库重新加载数据时，可以使用`Session.expire()`和`Session.refresh()`方法，这种情况下已知当前数据状态可能过时。 这样做的原因可能包括：

+   在 ORM 对象处理范围之外的事务中发出了一些 SQL，比如使用`Session.execute()`方法发出了`Table.update()`构造;

+   如果应用程序试图获取已知在并发事务中已被修改的数据，并且已知生效的隔离规则允许查看这些数据。

第二个要点是“已知生效的隔离规则允许查看这些数据。” 这意味着不能假设在另一个数据库连接上发生的 UPDATE 在本地这里就能看到；在许多情况下，是不会看到的。 这就是为什么如果想要在进行中的事务之间查看数据，就需要了解生效的隔离行为，从而使用`Session.expire()`或`Session.refresh()`。

另请参阅

`Session.expire()`

`Session.expire_all()`

`Session.refresh()`

填充现有对象 - 允许任何 ORM 查询刷新对象，就像它们通常加载一样，根据 SELECT 语句的结果刷新标识映射中的所有匹配对象。

隔离 - 隔离的词汇解释，包括指向维基百科的链接。

[SQLAlchemy 会话深入解析](https://techspot.zzzeek.org/2012/11/14/pycon-canada-the-sqlalchemy-session-in-depth/) - 一个视频+幻灯片，深入讨论对象生命周期，包括数据过期的作用。
