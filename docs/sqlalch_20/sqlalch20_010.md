# 使用 ORM 进行数据操作

> 原文：[`docs.sqlalchemy.org/en/20/tutorial/orm_data_manipulation.html`](https://docs.sqlalchemy.org/en/20/tutorial/orm_data_manipulation.html)

上一节处理数据保持了从核心角度来看 SQL 表达语言的关注，以便提供各种主要 SQL 语句结构的连续性。接下来的部分将扩展`Session`的生命周期，以及它如何与这些结构交互。

**先决条件部分** - 教程中 ORM 重点部分建立在本文档中的两个先前 ORM 中心部分的基础上：

+   使用 ORM 会话执行 - 介绍如何创建 ORM `Session`对象

+   使用 ORM 声明性表单定义表元数据 - 我们在这里设置了`User`和`Address`实体的 ORM 映射

+   选择 ORM 实体和列 - 一些关于如何为诸如`User`之类的实体运行 SELECT 语句的示例

## 使用 ORM 工作单元模式插入行

当使用 ORM 时，`Session`对象负责构造`Insert`构造并将它们作为 INSERT 语句发出到正在进行的事务中。我们指示`Session`这样做的方式是通过**添加**对象条目到它; `Session`然后确保这些新条目在需要时被发出到数据库，使用称为**flush**的过程。`Session`用于持久化对象的整体过程被称为工作单元模式。

### 类的实例代表行

而在前一个示例中，我们使用 Python 字典发出了一个 INSERT，以指示我们要添加的数据，使用 ORM 时，我们直接使用我们定义的自定义 Python 类，在使用 ORM 声明性表单定义表元数据中。在类级别，`User`和`Address`类用作定义相应数据库表应该如何查看的位置。这些类还用作可扩展的数据对象，我们用它们来创建和操作事务中的行。下面我们将创建两个`User`对象，每个对象代表一个要插入的潜在数据库行：

```py
>>> squidward = User(name="squidward", fullname="Squidward Tentacles")
>>> krabs = User(name="ehkrabs", fullname="Eugene H. Krabs")
```

我们可以使用映射列的名称作为构造函数中的关键字参数来构造这些对象。这是可能的，因为 `User` 类包含一个由 ORM 映射提供的自动生成的 `__init__()` 构造函数，以便我们可以使用构造函数中的列名作为键来创建每个对象。

类似于我们在 `Insert` 的核心示例中的做法，我们没有包含主键（即 `id` 列的条目），因为我们希望利用数据库的自动递增主键功能，这里是 SQLite，ORM 也与之集成。上述对象的 `id` 属性的值，如果我们查看它，会显示为 `None`：

```py
>>> squidward
User(id=None, name='squidward', fullname='Squidward Tentacles')
```

`None` 值由 SQLAlchemy 提供，表示属性目前没有值。在处理尚未分配值的新对象时，SQLAlchemy 映射的属性始终在 Python 中返回一个值，并且如果缺少值，则不会引发 `AttributeError`。

目前，上述两个对象被称为处于 transient 状态 - 它们与任何数据库状态都没有关联，尚未与可以为它们生成 INSERT 语句的 `Session` 对象关联。

### 将对象添加到会话

为了逐步说明添加过程，我们将创建一个不使用上下文管理器的 `Session`（因此我们必须确保稍后关闭它！）：

```py
>>> session = Session(engine)
```

然后使用 `Session.add()` 方法将对象添加到 `Session` 中。当调用此方法时，对象处于一种称为 pending 的状态，尚未插入：

```py
>>> session.add(squidward)
>>> session.add(krabs)
```

当我们有待处理的对象时，我们可以通过查看 `Session` 上的一个集合来查看这种状态，该集合称为 `Session.new`：

```py
>>> session.new
IdentitySet([User(id=None, name='squidward', fullname='Squidward Tentacles'), User(id=None, name='ehkrabs', fullname='Eugene H. Krabs')])
```

上述视图使用一个名为 `IdentitySet` 的集合，它本质上是一个 Python 集合，以所有情况下的对象标识哈希（即使用 Python 内置的 `id()` 函数，而不是 Python 的 `hash()` 函数）。

### 刷新

`Session` 使用一种称为工作单元（unit of work）的模式。这通常意味着它逐个累积更改，但实际上直到需要时才将它们传递到数据库。这使它能够根据给定的一组待处理更改，更好地决定如何在事务中发出 SQL DML。当它确实向数据库发出 SQL 以推送当前更改集时，该过程被称为**刷新**。

我们可以通过调用`Session.flush()`方法来手动说明刷新过程：

```py
>>> session.flush()
BEGIN  (implicit)
INSERT  INTO  user_account  (name,  fullname)  VALUES  (?,  ?)  RETURNING  id
[...  (insertmanyvalues)  1/2  (ordered;  batch  not  supported)]  ('squidward',  'Squidward Tentacles')
INSERT  INTO  user_account  (name,  fullname)  VALUES  (?,  ?)  RETURNING  id
[insertmanyvalues  2/2  (ordered;  batch  not  supported)]  ('ehkrabs',  'Eugene H. Krabs') 
```

上面我们观察到首先调用`Session`以发出 SQL，因此它创建了一个新的事务并为两个对象发出了适当的 INSERT 语句。事务现在**保持打开**，直到我们调用任何`Session.commit()`、`Session.rollback()`或`Session.close()`方法。

虽然`Session.flush()`可用于手动推送待处理更改到当前事务，但通常是不必要的，因为`Session`具有一种称为**自动刷新**的行为，我们稍后将说明。每当调用`Session.commit()`时，它也会刷新更改。

### 自动产生的主键属性

一旦行被插入，我们创建的两个 Python 对象处于持久（persistent）状态，它们与它们被添加或加载到的`Session`对象相关联，并具有稍后将介绍的许多其他行为。

发生的 INSERT 的另一个效果是 ORM 检索了每个新对象的新主键标识符；在内部，它通常使用我们之前介绍的相同的`CursorResult.inserted_primary_key`访问器。`squidward` 和 `krabs` 对象现在具有这些新的主键标识符，并且我们可以通过访问 `id` 属性查看它们：

```py
>>> squidward.id
4
>>> krabs.id
5
```

提示

当 ORM 在刷新对象时为什么会发出两个单独的 INSERT 语句，而不是使用 executemany？正如我们将在下一节中看到的，`Session`在刷新对象时始终需要知道新插入对象的主键。如果使用了诸如 SQLite 的自动增量（其他示例包括 PostgreSQL IDENTITY 或 SERIAL，使用序列等）之类的功能，则`CursorResult.inserted_primary_key`功能通常要求每次 INSERT 都逐行发出。如果我们提前为主键提供了值，ORM 将能够更好地优化操作。一些数据库后端，如 psycopg2，还可以一次插入多行，同时仍然能够检索主键值。

### 通过主键从身份映射获取对象

对象的主键标识对于`Session`非常重要，因为这些对象现在使用称为身份映射的功能与此标识在内存中连接在一起。身份映射是一个内存存储器，它将当前加载在内存中的所有对象与它们的主键标识链接起来。我们可以通过使用`Session.get()`方法之一来检索上述对象之一来观察到这一点，如果本地存在，则返回身份映射中的条目，否则发出一个 SELECT：

```py
>>> some_squidward = session.get(User, 4)
>>> some_squidward
User(id=4, name='squidward', fullname='Squidward Tentacles')
```

身份映射的重要一点是，在特定`Session`对象的范围内，它维护着特定 Python 对象的**唯一实例**与特定数据库标识的关系。我们可以观察到，`some_squidward`指的是之前`squidward`所指的**同一个对象**：

```py
>>> some_squidward is squidward
True
```

身份映射是一个关键特性，允许在事务中操作复杂的对象集合而不会出现同步问题。

### 提交

关于`Session`的工作方式还有很多要说的，这将在后续进一步讨论。现在我们将提交事务，以便在深入研究 ORM 行为和特性之前积累关于如何在 SELECT 行之前的知识：

```py
>>> session.commit()
COMMIT
```

上述操作将提交正在进行的事务。 我们处理过的对象仍然附加到 `Session`，这是一个状态，直到 `Session`关闭（在关闭会话中介绍）。

提示

注意的一件重要事情是，我们刚刚处理的对象上的属性已经过期，意味着，当我们下一次访问它们的任何属性时，`Session` 将启动一个新的事务并重新加载它们的状态。 这个选项有时对性能原因或者如果希望在关闭`Session`后继续使用对象（这被称为分离状态）可能会有问题，因为它们将不会有任何状态，并且将没有 `Session` 与其一起加载该状态，导致“分离实例”错误。 可以使用一个名为`Session.expire_on_commit`的参数来控制行为。 更多信息请参见关闭会话。  ## 使用工作单元模式更新 ORM 对象

在前面的一节使用 UPDATE 和 DELETE 语句中，我们介绍了代表 SQL UPDATE 语句的 `Update` 构造。 当使用 ORM 时，有两种方式使用此构造。 主要方式是，它作为`Session`使用的工作单元过程的一部分自动发出，其中对具有更改的单个对象对应的每个主键发出一个 UPDATE 语句。

假设我们将用户名为`sandy`的`User`对象加载到一个事务中（同时还展示了`Select.filter_by()`方法以及`Result.scalar_one()`方法）：

```py
>>> sandy = session.execute(select(User).filter_by(name="sandy")).scalar_one()
BEGIN  (implicit)
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
WHERE  user_account.name  =  ?
[...]  ('sandy',) 
```

如前所述，Python 对象`sandy`充当数据库中的行的**代理**，更具体地说，是**相对于当前事务的**具有主键标识`2`的数据库行：

```py
>>> sandy
User(id=2, name='sandy', fullname='Sandy Cheeks')
```

如果我们更改此对象的属性，`Session`将跟踪此更改：

```py
>>> sandy.fullname = "Sandy Squirrel"
```

对象出现在一个名为`Session.dirty`的集合中，表示对象“脏”：

```py
>>> sandy in session.dirty
True
```

当`Session`下次执行 flush 时，将会发出一个 UPDATE，以在数据库中更新此值。如前所述，在发出任何 SELECT 之前，会自动执行 flush，这种行为称为**自动 flush**。我们可以直接查询这一行的`User.fullname`列，我们将得到我们的更新值：

```py
>>> sandy_fullname = session.execute(select(User.fullname).where(User.id == 2)).scalar_one()
UPDATE  user_account  SET  fullname=?  WHERE  user_account.id  =  ?
[...]  ('Sandy Squirrel',  2)
SELECT  user_account.fullname
FROM  user_account
WHERE  user_account.id  =  ?
[...]  (2,)
>>> print(sandy_fullname)
Sandy Squirrel
```

我们可以看到上面我们请求`Session`执行了一个单独的`select()`语句。然而，发出的 SQL 显示了还发出了一个 UPDATE，这是 flush 过程推出挂起的更改。`sandy` Python 对象现在不再被认为是脏的：

```py
>>> sandy in session.dirty
False
```

然而请注意，我们**仍然处于一个事务中**，我们的更改尚未推送到数据库的永久存储中。由于桑迪的姓实际上是“Cheeks”而不是“Squirrel”，我们将在回滚事务时修复这个错误。但首先我们会做一些更多的数据更改。

亦可参见

Flush-详细说明了 flush 过程以及关于`Session.autoflush`设置的信息。##使用工作单元模式删除 ORM 对象

为了完成基本的持久性操作，可以使用`Session.delete()`方法在工作单元过程中标记一个个别的 ORM 对象以进行删除操作。让我们从数据库加载`patrick`：

```py
>>> patrick = session.get(User, 3)
SELECT  user_account.id  AS  user_account_id,  user_account.name  AS  user_account_name,
user_account.fullname  AS  user_account_fullname
FROM  user_account
WHERE  user_account.id  =  ?
[...]  (3,) 
```

如果我们标记`patrick`以进行删除，与其他操作一样，直到进行 flush 才会实际发生任何事情：

```py
>>> session.delete(patrick)
```

当前的 ORM 行为是`patrick`会一直留在`Session`中，直到 flush 进行，如前所述，如果我们发出查询，就会发生 flush：

```py
>>> session.execute(select(User).where(User.name == "patrick")).first()
SELECT  address.id  AS  address_id,  address.email_address  AS  address_email_address,
address.user_id  AS  address_user_id
FROM  address
WHERE  ?  =  address.user_id
[...]  (3,)
DELETE  FROM  user_account  WHERE  user_account.id  =  ?
[...]  (3,)
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
WHERE  user_account.name  =  ?
[...]  ('patrick',) 
```

在上面，我们要求发出的 SELECT 语句之前是一个 DELETE，这表明 `patrick` 的待删除操作已经进行了。还有一个针对 `address` 表的 `SELECT`，这是由于 ORM 在寻找与目标行可能相关的这个表中的行而引起的；这种行为是所谓的 级联 行为的一部分，并且可以通过允许数据库自动处理 `address` 中的相关行来更有效地工作；关于此的详细信息请参见 delete。

另请参见

delete - 描述了如何调整 `Session.delete()` 的行为，以便处理其他表中的相关行应该如何处理。

除此之外，现在正在被删除的 `patrick` 对象实例不再被视为在 `Session` 中持久存在，这可以通过包含性检查来显示：

```py
>>> patrick in session
False
```

然而，就像我们对 `sandy` 对象进行的更新一样，我们在这里做出的每一个改变都只在正在进行的事务中有效，如果我们不提交事务，这些改变就不会永久保存。由于此刻回滚事务更加有趣，我们将在下一节中进行。## 批量/多行 INSERT、upsert、UPDATE 和 DELETE

本节讨论的工作单元技术旨在将 dml（即 INSERT/UPDATE/DELETE 语句）与 Python 对象机制集成，通常涉及到相互关联对象的复杂图。一旦对象使用 `Session.add()` 添加到 `Session` 中，工作单元过程会自动代表我们发出 INSERT/UPDATE/DELETE，因为我们的对象属性被创建和修改。

但是，ORM `Session` 也有处理命令的能力，使其能够直接发出 INSERT、UPDATE 和 DELETE 语句，而不需要传递任何 ORM 持久化的对象，而是传递要 INSERT、UPDATE 或 upsert 的值列表，或者 WHERE 条件，以便可以调用一次匹配多行的 UPDATE 或 DELETE 语句。当需要影响大量行而无需构建和操作映射对象时，这种用法尤为重要，因为对于简单、性能密集型的任务，如大批量插入，这可能是繁琐和不必要的。

ORM 的批量/多行功能`Session`直接使用`insert()`、`update()`和`delete()`构造，并且它们的使用方式类似于与 SQLAlchemy Core 一起使用它们的方式（首次在本教程中介绍于使用 INSERT 语句和使用 UPDATE 和 DELETE 语句）。当使用这些构造与 ORM`Session`而不是普通的`Connection`时，它们的构建、执行和结果处理与 ORM 完全集成。

关于使用这些功能的背景和示例，请参见 ORM-启用的 INSERT、UPDATE 和 DELETE 语句部分，位于 ORM 查询指南中。

另请参阅

ORM-启用的 INSERT、UPDATE 和 DELETE 语句 - 在 ORM 查询指南中

## 回滚

`Session`有一个`Session.rollback()`方法，如预期般在进行中的 SQL 连接上发出 ROLLBACK。但是，它还会影响当前与`Session`关联的对象，例如我们先前示例中的 Python 对象`sandy`。虽然我们将`sandy`对象的`.fullname`更改为读取`"Sandy Squirrel"`，但我们想要回滚此更改。调用`Session.rollback()`不仅会回滚事务，还会**过期**与此`Session`当前关联的所有对象，这将使它们在下次使用时自动刷新，使用一种称为延迟加载的过程：

```py
>>> session.rollback()
ROLLBACK
```

要更仔细地查看“过期”过程，我们可以观察到 Python 对象`sandy`在其 Python`__dict__`中没有留下状态，除了一个特殊的 SQLAlchemy 内部状态对象：

```py
>>> sandy.__dict__
{'_sa_instance_state': <sqlalchemy.orm.state.InstanceState object at 0x...>}
```

这是“过期”状态；再次访问属性将自动开始一个新的事务，并使用当前数据库行刷新`sandy`：

```py
>>> sandy.fullname
BEGIN  (implicit)
SELECT  user_account.id  AS  user_account_id,  user_account.name  AS  user_account_name,
user_account.fullname  AS  user_account_fullname
FROM  user_account
WHERE  user_account.id  =  ?
[...]  (2,)
'Sandy Cheeks'
```

我们现在可以观察到完整的数据库行也被填充到`sandy`对象的`__dict__`中：

```py
>>> sandy.__dict__  
{'_sa_instance_state': <sqlalchemy.orm.state.InstanceState object at 0x...>,
 'id': 2, 'name': 'sandy', 'fullname': 'Sandy Cheeks'}
```

对于删除的对象，当我们之前注意到`patrick`不再在会话中时，该对象的身份也被恢复：

```py
>>> patrick in session
True
```

当然，数据库数据也再次出现了：

```py
>>> session.execute(select(User).where(User.name == "patrick")).scalar_one() is patrick
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
WHERE  user_account.name  =  ?
[...]  ('patrick',)
True
```

## 关闭会话

在上述部分中，我们在 Python 上下文管理器之外使用了一个`Session`对象，也就是说，我们没有使用`with`语句。这没问题，但是如果我们以这种方式操作，最好在完成后明确关闭`Session`：

```py
>>> session.close()
ROLLBACK 
```

关闭`Session`，也就是当我们在上下文管理器中使用它时发生的情况，会实现以下几个目标：

+   它释放所有连接资源到连接池中，取消（例如回滚）任何正在进行的事务。

    这意味着当我们使用一个会话执行一些只读任务然后关闭它时，我们不需要显式调用`Session.rollback()`来确保事务被回滚；连接池会处理这个问题。

+   它**清除**`Session`中的所有对象。

    这意味着我们为这个`Session`加载的所有 Python 对象，比如`sandy`、`patrick`和`squidward`，现在处于称为分离的状态。特别是，我们会注意到仍处于过期状态的对象，例如由于调用了`Session.commit()`，现在已经不可用，因为它们不包含当前行的状态，并且不再与任何数据库事务相关联，也不再可以被刷新：

    ```py
    # note that 'squidward.name' was just expired previously, so its value is unloaded
    >>> squidward.name
    Traceback (most recent call last):
      ...
    sqlalchemy.orm.exc.DetachedInstanceError: Instance <User at 0x...> is not bound to a Session; attribute refresh operation cannot proceed
    ```

    分离的对象可以使用`Session.add()`方法重新与相同或新的`Session`关联，这将重新建立它们与特定数据库行的关系：

    ```py
    >>> session.add(squidward)
    >>> squidward.name
    BEGIN  (implicit)
    SELECT  user_account.id  AS  user_account_id,  user_account.name  AS  user_account_name,  user_account.fullname  AS  user_account_fullname
    FROM  user_account
    WHERE  user_account.id  =  ?
    [...]  (4,)
    'squidward'
    ```

    提示

    尽量避免使用对象处于分离状态。当关闭 `Session` 时，也清理对所有先前附加对象的引用。对于需要分离对象的情况，通常是在 Web 应用程序中立即显示刚提交的对象的情况下，其中 `Session` 在渲染视图之前关闭，在这种情况下，将 `Session.expire_on_commit` 标志设置为 `False`。## 使用 ORM 工作单元模式插入行

在使用 ORM 时，`Session` 对象负责构造 `Insert` 构造，并在进行中的事务中发出它们作为 INSERT 语句。我们指示 `Session` 这样做的方式是通过向其中**添加**对象条目；然后，`Session` 确保这些新条目在需要时将被发出到数据库中，使用一种称为 **flush** 的过程。`Session` 用于持久化对象的整体过程称为 工作单元 模式。

### 类的实例代表行

而在上一个示例中，我们使用 Python 字典发出了一个 INSERT，以指示我们要添加的数据，使用 ORM 时，我们直接使用我们在 使用 ORM 声明性表单定义表元数据 中定义的自定义 Python 类。在类级别上，`User` 和 `Address` 类充当了定义相应数据库表应该如何的地方。这些类还作为可扩展的数据对象，我们用它来在事务中创建和操作行。下面我们将创建两个 `User` 对象，每个对象代表一个待插入的潜在数据库行：

```py
>>> squidward = User(name="squidward", fullname="Squidward Tentacles")
>>> krabs = User(name="ehkrabs", fullname="Eugene H. Krabs")
```

我们可以使用映射列的名称作为构造函数中的关键字参数来构造这些对象。这是可能的，因为 `User` 类包含了由 ORM 映射提供的自动生成的 `__init__()` 构造函数，以便我们可以使用列名作为构造函数中的键来创建每个对象。

与我们 Core 示例中的`Insert`类似，我们没有包含主键（即`id`列的条目），因为我们希望利用数据库的自动递增主键特性，此处为 SQLite，ORM 也与之集成。如果我们要查看上述对象的`id`属性的值，则显示为`None`：

```py
>>> squidward
User(id=None, name='squidward', fullname='Squidward Tentacles')
```

`None`值由 SQLAlchemy 提供，以指示属性目前尚无值。SQLAlchemy 映射的属性始终在 Python 中返回一个值，并且在处理尚未分配值的新对象时不会引发`AttributeError`。

目前，我们上述的两个对象被称为 transient 状态 - 它们与任何数据库状态都没有关联，尚未关联到可以为它们生成 INSERT 语句的`Session`对象。

### 添加对象到会话

为了逐步说明添加过程，我们将创建一个不使用上下文管理器的`Session`（因此我们必须确保稍后关闭它！）：

```py
>>> session = Session(engine)
```

然后使用`Session.add()`方法将对象添加到`Session`中。当调用此方法时，对象处于称为 pending 的状态，尚未插入：

```py
>>> session.add(squidward)
>>> session.add(krabs)
```

当我们有待处理的对象时，我们可以通过查看`Session`上的一个集合来查看此状态，该集合称为`Session.new`：

```py
>>> session.new
IdentitySet([User(id=None, name='squidward', fullname='Squidward Tentacles'), User(id=None, name='ehkrabs', fullname='Eugene H. Krabs')])
```

上述视图使用了一个名为`IdentitySet`的集合，实质上是一个 Python 集合，以所有情况下的对象标识进行哈希（即使用 Python 内置的`id()`函数，而不是 Python 的`hash()`函数）。

### 刷新

`Session`使用一种称为 unit of work 的模式。这通常意味着它逐一累积更改，但实际上直到需要才会将它们传达到数据库。这允许它根据给定的一组待处理更改做出有关在事务中应该发出 SQL DML 的更好决策。当它发出 SQL 到数据库以推出当前一组更改时，该过程称为**flush**。

我们可以通过手动调用`Session.flush()`方法来说明刷新过程：

```py
>>> session.flush()
BEGIN  (implicit)
INSERT  INTO  user_account  (name,  fullname)  VALUES  (?,  ?)  RETURNING  id
[...  (insertmanyvalues)  1/2  (ordered;  batch  not  supported)]  ('squidward',  'Squidward Tentacles')
INSERT  INTO  user_account  (name,  fullname)  VALUES  (?,  ?)  RETURNING  id
[insertmanyvalues  2/2  (ordered;  batch  not  supported)]  ('ehkrabs',  'Eugene H. Krabs') 
```

在上面的例子中，`Session`首先被调用以发出 SQL，因此它创建了一个新事务，并为两个对象发出了适当的 INSERT 语句。直到我们调用`Session.commit()`、`Session.rollback()`或`Session.close()`方法之一，该事务现在**保持打开状态**`Session`。

虽然`Session.flush()`可以用于手动推送当前事务中的待处理更改，但通常是不必要的，因为`Session`具有一个被称为**自动刷新**的行为，我们稍后会说明。每当调用`Session.commit()`时，它也会刷新更改。

### 自动生成的主键属性

一旦行被插入，我们创建的两个 Python 对象处于一种称为持久性的状态，它们与它们所添加或加载的`Session`对象相关联，并具有许多其他行为，稍后将进行介绍。

INSERT 操作的另一个效果是 ORM 检索了每个新对象的新主键标识符；内部通常使用我们之前介绍的相同的`CursorResult.inserted_primary_key`访问器。`squidward`和`krabs`对象现在具有这些新的主键标识符，并且我们可以通过访问`id`属性查看它们：

```py
>>> squidward.id
4
>>> krabs.id
5
```

提示

为什么 ORM 在可以使用 executemany 时发出两个单独的 INSERT 语句？正如我们将在下一节中看到的那样，当刷新对象时，`Session`总是需要知道新插入对象的主键。如果使用了诸如 SQLite 的自动增量（其他示例包括 PostgreSQL IDENTITY 或 SERIAL，使用序列等）之类的功能，则`CursorResult.inserted_primary_key`功能通常要求每个 INSERT 逐行发出。如果我们事先提供了主键的值，ORM 将能够更好地优化操作。一些数据库后端，如 psycopg2，也可以一次插入多行，同时仍然能够检索主键值。

### 通过主键从标识映射获取对象

对象的主键标识对于`Session`来说非常重要，因为这些对象现在使用一种称为标识映射的特性与此标识在内存中连接起来。标识映射是一个在内存中的存储器，将当前加载在内存中的所有对象链接到它们的主键标识。我们可以通过使用`Session.get()`方法检索上述对象之一来观察到这一点，如果本地存在，则会从标识映射中返回一个条目，否则会发出一个 SELECT：

```py
>>> some_squidward = session.get(User, 4)
>>> some_squidward
User(id=4, name='squidward', fullname='Squidward Tentacles')
```

关于标识映射的重要事情是，它在特定`Session`对象的范围内维护特定数据库标识的特定 Python 对象的**唯一实例**。我们可以观察到，`some_squidward`指的是先前的`squidward`的**相同对象**：

```py
>>> some_squidward is squidward
True
```

标识映射是一个关键特性，它允许在事务中处理复杂的对象集合而不会使事情失去同步。

### Committing

关于`Session`的工作还有很多要说的内容，这将在更进一步讨论。现在我们将提交事务，以便在查看更多 ORM 行为和特性之前构建对如何 SELECT 行的知识：

```py
>>> session.commit()
COMMIT
```

上述操作将提交进行中的事务。我们处理过的对象仍然附加到`Session`上，这是它们保持的状态，直到关闭`Session`（在关闭会话中介绍）。

提示

值得注意的一点是，我们刚刚使用的对象上的属性已经过期，意味着当我们下次访问它们的任何属性时，`Session`将启动一个新的事务并重新加载它们的状态。这个选项有时会因为性能原因或者如果希望在关闭`Session`后继续使用对象（即已知的分离状态），而带来问题，因为它们将没有任何状态，并且将没有任何`Session`来加载该状态，导致“分离实例”错误。这种行为可以通过一个名为`Session.expire_on_commit`的参数来控制。更多信息请参阅关闭会话。

### 类的实例代表行

在前面的示例中，我们使用 Python 字典发出了一个 INSERT，以指示我们想要添加的数据，而使用 ORM 时，我们直接使用了我们定义的自定义 Python 类，在使用 ORM 声明式表单定义表元数据回到之前。在类级别上，`User`和`Address`类用作定义相应数据库表应该是什么样子的地方。这些类还充当我们用于在事务内创建和操作行的可扩展数据对象。接下来，我们将创建两个`User`对象，每个对象都代表一个可能要 INSERT 的数据库行：

```py
>>> squidward = User(name="squidward", fullname="Squidward Tentacles")
>>> krabs = User(name="ehkrabs", fullname="Eugene H. Krabs")
```

我们能够使用映射列的名称作为构造函数中的关键字参数来构造这些对象。这是可能的，因为`User`类包含了一个由 ORM 映射提供的自动生成的`__init__()`构造函数，以便我们可以使用列名作为构造函数中的键来创建每个对象。

与我们在核心示例中的`Insert`类似，我们没有包含主键（即`id`列的条目），因为我们希望利用数据库的自动递增主键功能，本例中为 SQLite，ORM 也与之集成。如果我们查看上述对象的`id`属性的值，会显示为`None`：

```py
>>> squidward
User(id=None, name='squidward', fullname='Squidward Tentacles')
```

`None`值由 SQLAlchemy 提供，表示该属性目前没有值。SQLAlchemy 映射的属性始终在 Python 中返回一个值，并且在处理尚未分配值的新对象时，不会引发`AttributeError`。

目前，我们上面的两个对象被称为瞬态状态 - 它们与任何数据库状态都没有关联，尚未与可以为它们生成 INSERT 语句的`Session`对象关联。

### 将对象添加到会话

为了逐步说明添加过程，我们将创建一个不使用上下文管理器的`Session`（因此我们必须确保稍后关闭它！）：

```py
>>> session = Session(engine)
```

然后使用`Session.add()`方法将对象添加到`Session`中。调用此方法时，对象处于称为待定状态，尚未插入：

```py
>>> session.add(squidward)
>>> session.add(krabs)
```

当我们有待定对象时，可以通过查看`Session`上的一个集合来查看这种状态，该集合称为`Session.new`:

```py
>>> session.new
IdentitySet([User(id=None, name='squidward', fullname='Squidward Tentacles'), User(id=None, name='ehkrabs', fullname='Eugene H. Krabs')])
```

上述视图使用一个名为`IdentitySet`的集合，它本质上是一个 Python 集合，在所有情况下都使用对象标识哈希（即使用 Python 内置的`id()`函数，而不是 Python 的`hash()`函数）。

### 刷新

`Session`使用一种称为工作单元的模式。这通常意味着它逐个累积更改，但直到需要才实际将它们传达给数据库。这使其能够根据给定的一组待定更改更好地决定应该如何发出 SQL DML。当它向数据库发出 SQL 以推送当前一组更改时，该过程称为**刷新**。

我们可以通过调用`Session.flush()`方法手动说明刷新过程：

```py
>>> session.flush()
BEGIN  (implicit)
INSERT  INTO  user_account  (name,  fullname)  VALUES  (?,  ?)  RETURNING  id
[...  (insertmanyvalues)  1/2  (ordered;  batch  not  supported)]  ('squidward',  'Squidward Tentacles')
INSERT  INTO  user_account  (name,  fullname)  VALUES  (?,  ?)  RETURNING  id
[insertmanyvalues  2/2  (ordered;  batch  not  supported)]  ('ehkrabs',  'Eugene H. Krabs') 
```

首先我们观察到 `Session` 首次被调用以发出 SQL，因此它创建了一个新的事务并为两个对象发出了适当的 INSERT 语句。这个事务现在 **保持开启**，直到我们调用 `Session.commit()`、`Session.rollback()` 或 `Session.close()` 方法之一。

虽然 `Session.flush()` 可以用来手动推送待定更改到当前事务，但通常不需要，因为 `Session` 具有一种称为 **自动刷新** 的行为，稍后我们将说明。它还会在每次调用 `Session.commit()` 时刷新更改。

### 自动生成的主键属性

一旦行被插入，我们创建的两个 Python 对象处于所谓的 持久化 状态，它们与它们被添加或加载的 `Session` 对象相关联，并具有稍后将会介绍的许多其他行为。

INSERT 操作的另一个效果是 ORM 检索了每个新对象的新主键标识符；内部通常使用我们之前介绍的相同的 `CursorResult.inserted_primary_key` 访问器。`squidward` 和 `krabs` 对象现在与这些新的主键标识符相关联，我们可以通过访问 `id` 属性查看它们：

```py
>>> squidward.id
4
>>> krabs.id
5
```

提示

为什么 ORM 在可以使用 executemany 的情况下发出了两个单独的 INSERT 语句？正如我们将在下一节中看到的那样，当刷新对象时，`Session`始终需要知道新插入对象的主键。如果使用了诸如 SQLite 的自增等功能（其他示例包括 PostgreSQL 的 IDENTITY 或 SERIAL，使用序列等），则`CursorResult.inserted_primary_key`特性通常要求每个 INSERT 一次发出一行。如果我们提前为主键提供了值，ORM 将能够更好地优化操作。一些数据库后端，如 psycopg2，也可以一次插入多行，同时仍然能够检索主键值。

### 从标识映射获取主键的对象

对象的主键身份对于`Session`非常重要，因为现在使用称为标识映射的功能将对象与此标识在内存中连接起来。标识映射是一个在内存中的存储，它将当前加载在内存中的所有对象与它们的主键标识连接起来。我们可以通过使用`Session.get()`方法之一检索上述对象来观察到这一点，如果在本地存在，则会从标识映射返回一个条目，否则会发出一个 SELECT：

```py
>>> some_squidward = session.get(User, 4)
>>> some_squidward
User(id=4, name='squidward', fullname='Squidward Tentacles')
```

关于标识映射的重要事情是，它在特定的`Session`对象的范围内维护了一个特定数据库标识的特定 Python 对象的**唯一实例**。我们可以观察到，`some_squidward`引用的是之前`squidward`的**同一对象**：

```py
>>> some_squidward is squidward
True
```

标识映射是一个关键功能，允许在事务中操作复杂的对象集合而不会出现不同步的情况。

### 提交

关于`Session`如何工作还有很多要说的内容，这将在以后进一步讨论。目前，我们将提交事务，以便在检查更多 ORM 行为和特性之前积累关于如何 SELECT 行的知识：

```py
>>> session.commit()
COMMIT
```

上述操作将提交正在进行的事务。我们处理过的对象仍然附加到`Session`，这是它们保持的状态，直到`Session`关闭（在关闭会话中介绍）。

提示

需要注意的重要事项是，我们刚刚处理过的对象上的属性已经过期，意味着，当我们下次访问它们的任何属性时，`Session`将启动一个新事务并重新加载它们的状态。这个选项有时会因为性能原因或者在关闭`Session`后希望使用对象（即分离状态）而带来问题，因为它们将不再具有任何状态，并且没有`Session`来加载该状态，导致“分离实例”错误。这种行为可以通过一个名为`Session.expire_on_commit`的参数来控制。更多信息请参考关闭会话。

## 使用工作单元模式更新 ORM 对象

在前面的章节使用 UPDATE 和 DELETE 语句中，我们介绍了代表 SQL UPDATE 语句的`Update`构造。在使用 ORM 时，有两种方式可以使用这个构造。主要方式是它会自动作为`Session`使用的工作单元过程的一部分发出，其中会针对具有更改的单个对象按照每个主键的方式发出 UPDATE 语句。

假设我们将用户名为`sandy`的`User`对象加载到一个事务中（同时展示`Select.filter_by()`方法以及`Result.scalar_one()`方法）：

```py
>>> sandy = session.execute(select(User).filter_by(name="sandy")).scalar_one()
BEGIN  (implicit)
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
WHERE  user_account.name  =  ?
[...]  ('sandy',) 
```

如前所述，Python 对象`sandy`充当数据库中行的**代理**，更具体地说是当前事务中具有主键标识`2`的数据库行：

```py
>>> sandy
User(id=2, name='sandy', fullname='Sandy Cheeks')
```

如果我们更改此对象的属性，`Session`会跟踪此更改：

```py
>>> sandy.fullname = "Sandy Squirrel"
```

对象出现在称为`Session.dirty`的集合中，表示对象是“脏”的：

```py
>>> sandy in session.dirty
True
```

当`Session`再次发出刷新时，将发出一个更新，将此值在数据库中更新。如前所述，在发出任何 SELECT 之前，刷新会自动发生，使用称为**自动刷新**的行为。我们可以直接查询该行的 `User.fullname` 列，我们将得到我们的更新值：

```py
>>> sandy_fullname = session.execute(select(User.fullname).where(User.id == 2)).scalar_one()
UPDATE  user_account  SET  fullname=?  WHERE  user_account.id  =  ?
[...]  ('Sandy Squirrel',  2)
SELECT  user_account.fullname
FROM  user_account
WHERE  user_account.id  =  ?
[...]  (2,)
>>> print(sandy_fullname)
Sandy Squirrel
```

我们可以看到我们请求`Session`执行了一个单独的`select()`语句。但是，发出的 SQL 表明还发出了 UPDATE，这是刷新过程推出挂起更改。`sandy` Python 对象现在不再被视为脏：

```py
>>> sandy in session.dirty
False
```

但请注意，我们仍然**处于事务中**，我们的更改尚未推送到数据库的永久存储中。由于 Sandy 的姓实际上是“Cheeks”而不是“Squirrel”，我们稍后会在回滚事务时修复此错误。但首先我们将进行更多的数据更改。

另见

刷新-详细介绍了刷新过程以及有关`Session.autoflush`设置的信息。

## 使用工作单元模式删除 ORM 对象

为了完善基本的持久性操作，可以通过使用`Session.delete()`方法在工作单元过程中标记要删除的单个 ORM 对象。让我们从数据库中加载 `patrick`：

```py
>>> patrick = session.get(User, 3)
SELECT  user_account.id  AS  user_account_id,  user_account.name  AS  user_account_name,
user_account.fullname  AS  user_account_fullname
FROM  user_account
WHERE  user_account.id  =  ?
[...]  (3,) 
```

如果我们标记`patrick`要删除，就像其他操作一样，直到刷新进行，实际上什么也不会发生：

```py
>>> session.delete(patrick)
```

当前的 ORM 行为是，`patrick` 会留在`Session`中，直到刷新进行，正如之前提到的，如果我们发出查询：

```py
>>> session.execute(select(User).where(User.name == "patrick")).first()
SELECT  address.id  AS  address_id,  address.email_address  AS  address_email_address,
address.user_id  AS  address_user_id
FROM  address
WHERE  ?  =  address.user_id
[...]  (3,)
DELETE  FROM  user_account  WHERE  user_account.id  =  ?
[...]  (3,)
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
WHERE  user_account.name  =  ?
[...]  ('patrick',) 
```

上面，我们要求发出的 SELECT 之前是一个 DELETE，这表明了对 `patrick` 的待删除操作。还有一个针对 `address` 表的 `SELECT`，这是因为 ORM 在该表中查找可能与目标行相关的行；这种行为是作为 级联 行为的一部分，并且可以通过允许数据库自动处理 `address` 中的相关行来更高效地进行调整；delete 部分详细介绍了这一点。

另请参阅

delete - 描述了如何调整 `Session.delete()` 的行为，以便处理其他表中相关行的方式。

此外，正在删除的 `patrick` 对象实例不再被认为是 `Session` 中的持久对象，这可以通过包含检查来展示：

```py
>>> patrick in session
False
```

然而，就像我们对 `sandy` 对象进行的 UPDATE 一样，我们在这里所做的每一个更改都仅限于正在进行的事务，如果我们不提交它，这些更改就不会变得永久。由于在当前情况下回滚事务更有趣，我们将在下一节中执行该操作。

## 批量/多行 INSERT、upsert、UPDATE 和 DELETE

此部分讨论的工作单元技术旨在将 dml 或 INSERT/UPDATE/DELETE 语句与 Python 对象机制集成，通常涉及复杂的相互关联对象图。一旦对象使用 `Session.add()` 添加到 `Session` 中，工作单元过程将自动代表我们发出 INSERT/UPDATE/DELETE，因为我们的对象属性被创建和修改。

然而，ORM `Session` 还具有处理命令的能力，使其能够直接发出 INSERT、UPDATE 和 DELETE 语句，而无需传递任何 ORM 持久化的对象，而是传递要 INSERT、UPDATE 或 upsert 或 WHERE 条件的值列表，以便一次匹配多行的 UPDATE 或 DELETE 语句可以被调用。当需要影响大量行而无需构造和操作映射对象时，此使用模式尤为重要，因为对于简单、性能密集的任务（如大型批量插入），构造和操作映射对象可能会很麻烦和不必要。

ORM `Session`的批量/多行功能直接使用了 `insert()`、`update()` 和 `delete()` 构造，并且它们的使用方式类似于它们在 SQLAlchemy Core 中的使用方式（首次在本教程中介绍了使用 INSERT 语句和使用 UPDATE 和 DELETE 语句）。当使用这些构造与 ORM `Session` 而不是普通的`Connection`时，它们的构建、执行和结果处理与 ORM 完全集成。

有关使用这些功能的背景和示例，请参见 ORM 启用的 INSERT、UPDATE 和 DELETE 语句部分，在 ORM 查询指南中。

另请参见

ORM 启用的 INSERT、UPDATE 和 DELETE 语句 - 在 ORM 查询指南中

## 回滚

`Session`有一个 `Session.rollback()` 方法，如预期的那样，在进行中的 SQL 连接上发出一个 ROLLBACK。然而，它也对当前与`Session`关联的对象产生影响，例如我们之前的示例中的 Python 对象`sandy`。虽然我们已经将`sandy`对象的`.fullname`更改为`"Sandy Squirrel"`，但我们想要回滚此更改。调用`Session.rollback()`不仅会回滚事务，还会**使当前与此`Session`关联的所有对象过期**，这将导致它们在下次使用时自动刷新，这个过程称为惰性加载：

```py
>>> session.rollback()
ROLLBACK
```

要更仔细地查看“到期”过程，我们可以观察到 Python 对象`sandy`在其 Python `__dict__`中没有剩余的状态，除了一个特殊的 SQLAlchemy 内部状态对象：

```py
>>> sandy.__dict__
{'_sa_instance_state': <sqlalchemy.orm.state.InstanceState object at 0x...>}
```

这是“过期”状态；再次访问属性将自动开始一个新的事务，并使用当前数据库行刷新`sandy`：

```py
>>> sandy.fullname
BEGIN  (implicit)
SELECT  user_account.id  AS  user_account_id,  user_account.name  AS  user_account_name,
user_account.fullname  AS  user_account_fullname
FROM  user_account
WHERE  user_account.id  =  ?
[...]  (2,)
'Sandy Cheeks'
```

现在我们可以观察到`__dict__`中还填充了`sandy`对象的完整数据库行：

```py
>>> sandy.__dict__  
{'_sa_instance_state': <sqlalchemy.orm.state.InstanceState object at 0x...>,
 'id': 2, 'name': 'sandy', 'fullname': 'Sandy Cheeks'}
```

对于已删除的对象，当我们之前注意到`patrick`不再在会话中时，该对象的标识也被恢复：

```py
>>> patrick in session
True
```

当然，数据库数据也再次出现了：

```py
>>> session.execute(select(User).where(User.name == "patrick")).scalar_one() is patrick
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
WHERE  user_account.name  =  ?
[...]  ('patrick',)
True
```

## 关闭会话

在上述部分中，我们在 Python 上下文管理器之外使用了一个`Session`对象，也就是说，我们没有使用`with`语句。虽然这样做没问题，但如果我们以这种方式操作，最好在完成后明确关闭`Session`：

```py
>>> session.close()
ROLLBACK 
```

关闭`Session`，也就是我们在上下文管理器中使用它时发生的事情，可以完成以下工作：

+   它释放所有连接资源到连接池，取消（例如回滚）任何正在进行的事务。

    这意味着当我们使用会话执行一些只读任务然后关闭它时，我们不需要显式调用`Session.rollback()`来确保事务被回滚；连接池会处理这个。

+   它**从`Session`中清除**所有对象。

    这意味着我们为此`Session`加载的所有 Python 对象，如`sandy`、`patrick`和`squidward`，现在处于一种称为分离(detached)的状态。特别是，我们会注意到仍处于过期(expired)状态的对象，例如由于对`Session.commit()`的调用而导致的对象，现在已经不再可用，因为它们不包含当前行的状态，也不再与任何数据库事务相关联以进行刷新：

    ```py
    # note that 'squidward.name' was just expired previously, so its value is unloaded
    >>> squidward.name
    Traceback (most recent call last):
      ...
    sqlalchemy.orm.exc.DetachedInstanceError: Instance <User at 0x...> is not bound to a Session; attribute refresh operation cannot proceed
    ```

    分离的对象可以使用`Session.add()`方法重新关联到相同或新的`Session`中，该方法将重新建立它们与特定数据库行的关系：

    ```py
    >>> session.add(squidward)
    >>> squidward.name
    BEGIN  (implicit)
    SELECT  user_account.id  AS  user_account_id,  user_account.name  AS  user_account_name,  user_account.fullname  AS  user_account_fullname
    FROM  user_account
    WHERE  user_account.id  =  ?
    [...]  (4,)
    'squidward'
    ```

    提示

    尽量避免在可能的情况下使用对象处于分离状态。当`Session`关闭时，同时清理所有先前附加对象的引用。对于需要分离对象的情况，通常是在 Web 应用程序中即时显示刚提交的对象，而`Session`在视图呈现之前关闭的情况下，将`Session.expire_on_commit`标志设置为`False`。
