# 级联

> 原文：[`docs.sqlalchemy.org/en/20/orm/cascades.html`](https://docs.sqlalchemy.org/en/20/orm/cascades.html)

映射器支持在`relationship()`构造上配置可配置级联行为的概念。这涉及到相对于特定`Session`上执行的操作应如何传播到由该关系引用的项目（例如“子”对象），并且受到`relationship.cascade`选项的影响。

级联的默认行为仅限于所谓的 save-update 和 merge 设置的级联。级联的典型“替代”设置是添加 delete 和 delete-orphan 选项；这些设置适用于只有在附加到其父对象时才存在的相关对象，并且在其他情况下将被删除。

使用`relationship()`上的`relationship.cascade`选项配置级联行为：

```py
class Order(Base):
    __tablename__ = "order"

    items = relationship("Item", cascade="all, delete-orphan")
    customer = relationship("User", cascade="save-update")
```

要在反向引用上设置级联，可以使用相同的标志与`backref()`函数一起使用，该函数最终将其参数反馈到`relationship()`中：

```py
class Item(Base):
    __tablename__ = "item"

    order = relationship(
        "Order", backref=backref("items", cascade="all, delete-orphan")
    )
```

`relationship.cascade`的默认值为`save-update, merge`。此参数的典型替代设置为`all`或更常见的是`all, delete-orphan`。`all`符号是`save-update, merge, refresh-expire, expunge, delete`的同义词，与`delete-orphan`结合使用表示子对象应在所有情况下跟随其父对象，并且一旦不再与该父对象关联就应该被删除。

警告

`all`级联选项意味着 refresh-expire 级联设置，当使用异步 I/O（asyncio）扩展时可能不可取，因为它将比在显式 I/O 上下文中通常适当地更积极地使相关对象过期。有关更多背景信息，请参阅在使用 AsyncSession 时防止隐式 I/O 中的注释。

可以为`relationship.cascade`参数指定的可用值列表在以下各小节中进行描述。

## save-update

`save-update`级联指示当通过`Session.add()`将对象放入`Session`中时，通过这个`relationship()`与之关联的所有对象也应该添加到同一个`Session`中。假设我们有一个对象`user1`，其中包含两个相关对象`address1`、`address2`：

```py
>>> user1 = User()
>>> address1, address2 = Address(), Address()
>>> user1.addresses = [address1, address2]
```

如果我们将`user1`添加到`Session`中，它也会隐式添加`address1`、`address2`：

```py
>>> sess = Session()
>>> sess.add(user1)
>>> address1 in sess
True
```

`save-update`级联也会影响已经存在于`Session`中的对象的属性操作。如果我们将第三个对象`address3`添加到`user1.addresses`集合中，它将成为该`Session`的状态的一部分：

```py
>>> address3 = Address()
>>> user1.addresses.append(address3)
>>> address3 in sess
True
```

当从集合中移除一个项目或将对象从标量属性中解除关联时，`save-update`级联可能会表现出令人惊讶的行为。在某些情况下，被孤立的对象仍然可能被拉入原父级的`Session`中；这是为了使刷新过程可以适当地处理相关对象。这种情况通常只会在一个对象从一个`Session`中移除并添加到另一个对象时出现：

```py
>>> user1 = sess1.scalars(select(User).filter_by(id=1)).first()
>>> address1 = user1.addresses[0]
>>> sess1.close()  # user1, address1 no longer associated with sess1
>>> user1.addresses.remove(address1)  # address1 no longer associated with user1
>>> sess2 = Session()
>>> sess2.add(user1)  # ... but it still gets added to the new session,
>>> address1 in sess2  # because it's still "pending" for flush
True
```

`save-update`级联默认启用，并且通常被视为理所当然；它通过允许单个调用`Session.add()`一次性在该`Session`中注册整个对象结构来简化代码。虽然它可以被禁用，但通常没有必要这样做。

### 双向关系中 save-update 级联的行为

在双向关系的上下文中，即使用`relationship.back_populates`或`relationship.backref`参数创建相互引用的两个独立的`relationship()`对象时，`save-update`级联是**单向的**。

当一个未关联`Session`的对象被赋给与关联`Session`相关的父对象的属性或集合时，它将自动添加到同一`Session`中。然而，反向操作不会产生此效果；当一个未关联`Session`的对象被赋给与关联`Session`相关的子对象时，不会自动将该父对象添加到`Session`中。这种行为的总体主题称为“级联反向引用”，并代表了从 SQLAlchemy 2.0 开始标准化的行为变更。

以示例说明，假设给定了一个`Order`对象的映射，它与一系列`Item`对象通过关系`Order.items`和`Item.order`双向关联：

```py
mapper_registry.map_imperatively(
    Order,
    order_table,
    properties={"items": relationship(Item, back_populates="order")},
)

mapper_registry.map_imperatively(
    Item,
    item_table,
    properties={"order": relationship(Order, back_populates="items")},
)
```

如果一个`Order`已经与一个`Session`相关联，并且然后创建一个`Item`对象并将其附加到该`Order`的`Order.items`集合中，`Item`将自动级联到相同的`Session`中：

```py
>>> o1 = Order()
>>> session.add(o1)
>>> o1 in session
True

>>> i1 = Item()
>>> o1.items.append(i1)
>>> o1 is i1.order
True
>>> i1 in session
True
```

在上述情况下，`Order.items`和`Item.order`的双向性意味着附加到`Order.items`也会赋值给`Item.order`。同时，`save-update`级联允许将`Item`对象添加到与父`Order`已关联的相同`Session`中。

然而，如果上述操作以**反向**方向执行，即赋值`Item.order`而不是直接附加到`Order.item`，则级联操作不会自动进行，即使对象赋值`Order.items`和`Item.order`与上一个示例中的状态相同：

```py
>>> o1 = Order()
>>> session.add(o1)
>>> o1 in session
True

>>> i1 = Item()
>>> i1.order = o1
>>> i1 in order.items
True
>>> i1 in session
False
```

在上述情况下，`Item`对象创建并设置完所有期望的状态后，应明确将其添加到`Session`中：

```py
>>> session.add(i1)
```

在较旧版本的 SQLAlchemy 中，保存-更新级联在所有情况下都会双向发生。然后，使用一个称为`cascade_backrefs`的选项使其成为可选项。最后，在 SQLAlchemy 1.4 中，旧行为被弃用，并且在 SQLAlchemy 2.0 中删除了`cascade_backrefs`选项。其理由是用户通常不会觉得将对象的属性分配给对象上的属性是直观的，如上面所示的`i1.order = o1`的分配，会改变对象`i1`的持久状态，使其现在在`Session`中处于挂起状态，并且在那些给定对象仍在构建并且尚未准备好被刷新的情况下，自动刷新会过早地刷新对象并导致错误。选择在单向和双向行为之间选择的选项也被删除，因为此选项创建了两种略有不同的工作方式，增加了 ORM 的整体学习曲线以及文档和用户支持负担。

另请参阅

在 2.0 中弃用以删除的 cascade_backrefs 行为 - 关于“级联反向引用”行为变更的背景 ## 删除

`delete`级联表示当“父”对象标记为删除时，其相关的“子”对象也应标记为删除。例如，如果我们有一个关系`User.addresses`配置了`delete`级联：

```py
class User(Base):
    # ...

    addresses = relationship("Address", cascade="all, delete")
```

如果使用上述映射，我们有一个`User`对象和两个相关的`Address`对象：

```py
>>> user1 = sess1.scalars(select(User).filter_by(id=1)).first()
>>> address1, address2 = user1.addresses
```

如果我们标记`user1`进行删除，在刷新操作进行后，`address1`和`address2`也将被删除：

```py
>>> sess.delete(user1)
>>> sess.commit()
DELETE  FROM  address  WHERE  address.id  =  ?
((1,),  (2,))
DELETE  FROM  user  WHERE  user.id  =  ?
(1,)
COMMIT 
```

或者，如果我们的`User.addresses`关系*没有*`delete`级联，SQLAlchemy 的默认行为是通过将它们的外键引用设置为`NULL`来解除`user1`与`address1`和`address2`的关联。使用以下映射：

```py
class User(Base):
    # ...

    addresses = relationship("Address")
```

在删除父`User`对象时，`address`中的行不会被删除，而是被解除关联：

```py
>>> sess.delete(user1)
>>> sess.commit()
UPDATE  address  SET  user_id=?  WHERE  address.id  =  ?
(None,  1)
UPDATE  address  SET  user_id=?  WHERE  address.id  =  ?
(None,  2)
DELETE  FROM  user  WHERE  user.id  =  ?
(1,)
COMMIT 
```

在一对多关系中，`delete`级联通常与`delete-orphan`级联结合使用，如果“子”对象与父对象解除关联，则会为相关行发出 DELETE。`delete`和`delete-orphan`级联的组合涵盖了 SQLAlchemy 必须在将外键列设置为 NULL 与完全删除行之间做出决定的两种情况。

该功能默认完全独立于数据库配置的`FOREIGN KEY`约束，这些约束本身可能配置`CASCADE`行为。为了更有效地与此配置集成，应使用描述在使用 ORM 关系中的外键 ON DELETE 级联的附加指令。

警告

请注意，ORM 的“delete”和“delete-orphan”行为**仅**适用于使用`Session.delete()`方法在 unit of work 过程中标记单个 ORM 实例以进行删除。它**不**适用于“批量”删除，这将使用`delete()`构造发出，如在 ORM UPDATE and DELETE with Custom WHERE Criteria 中所示。有关更多背景信息，请参阅 ORM 启用的更新和删除的重要说明和警告。

另请参阅

使用 ORM 关系的外键 ON DELETE 级联

使用删除级联处理多对多关系

delete-orphan

### 使用删除级联处理多对多关系

`cascade="all, delete"`选项在多对多关系中同样有效，该关系使用`relationship.secondary`指示一个关联表。当删除父对象，因此与其相关对象解除关联时，工作单元过程通常会从关联表中删除行，但保留相关对象。与`cascade="all, delete"`结合使用时，将为子行本身执行额外的`DELETE`语句。

以下示例将 Many To Many 的示例调整为说明关联的**一侧**设置为`cascade="all, delete"`：

```py
association_table = Table(
    "association",
    Base.metadata,
    Column("left_id", Integer, ForeignKey("left.id")),
    Column("right_id", Integer, ForeignKey("right.id")),
)

class Parent(Base):
    __tablename__ = "left"
    id = mapped_column(Integer, primary_key=True)
    children = relationship(
        "Child",
        secondary=association_table,
        back_populates="parents",
        cascade="all, delete",
    )

class Child(Base):
    __tablename__ = "right"
    id = mapped_column(Integer, primary_key=True)
    parents = relationship(
        "Parent",
        secondary=association_table,
        back_populates="children",
    )
```

在上述情况下，当使用`Session.delete()`标记`Parent`对象进行删除时，刷新过程通常会从`association`表中删除相关行，但根据级联规则，它还将删除所有相关的`Child`行。

警告

如果上述`cascade="all, delete"`设置在**两个**关系上都配置了，则级联操作将继续通过所有`Parent`和`Child`对象进行级联，加载遇到的每个`children`和`parents`集合并删除所有连接的内容。通常不希望“delete”级联双向配置。

另请参阅

从多对多表中删除行

使用外键 ON DELETE 处理多对多关系  ### 使用 ORM 关系的外键 ON DELETE 级联处理

SQLAlchemy 的“delete”级联行为与数据库 `FOREIGN KEY` 约束的 `ON DELETE` 特性重叠。SQLAlchemy 允许使用 `ForeignKey` 和 `ForeignKeyConstraint` 构造配置这些模式级 DDL 行为；如何在 `Table` 元数据与这些对象的使用一起配置，在 ON UPDATE and ON DELETE 中有描述。

为了将 `ON DELETE` 外键级联与 `relationship()` 结合使用，首先必须注意 `relationship.cascade` 设置仍然必须配置为匹配所需的`delete`或“set null”行为（使用 `delete` 级联或将其省略），以便无论是 ORM 还是数据库级约束将处理实际修改数据库中的数据的任务，ORM 仍将能够适当跟踪可能受到影响的本地存在对象的状态。

在 `relationship()` 上有一个额外的选项，指示 ORM 应该尝试自行运行与相关行的 DELETE/UPDATE 操作的程度，而不是依赖于期望数据库端的 FOREIGN KEY 约束级联处理任务；这是 `relationship.passive_deletes` 参数，它接受 `False`（默认值）、`True` 和 `"all"` 选项。

最典型的例子是当删除父行时要删除子行，并且在相关的 `FOREIGN KEY` 约束上配置了 `ON DELETE CASCADE`：

```py
class Parent(Base):
    __tablename__ = "parent"
    id = mapped_column(Integer, primary_key=True)
    children = relationship(
        "Child",
        back_populates="parent",
        cascade="all, delete",
        passive_deletes=True,
    )

class Child(Base):
    __tablename__ = "child"
    id = mapped_column(Integer, primary_key=True)
    parent_id = mapped_column(Integer, ForeignKey("parent.id", ondelete="CASCADE"))
    parent = relationship("Parent", back_populates="children")
```

当删除父行时，上述配置的行为如下：

1.  应用程序调用 `session.delete(my_parent)`，其中 `my_parent` 是 `Parent` 的一个实例。

1.  当 `Session` 下次将更改刷新到数据库时，`my_parent.children` 集合中的所有 **当前加载的** 项目都将由 ORM 删除，这意味着为每条记录发出一个 `DELETE` 语句。

1.  如果`my_parent.children`集合是**未加载**的，则不会发出`DELETE`语句。如果在此`relationship()`上未设置`relationship.passive_deletes`标志，则将为未加载的`Child`对象发出`SELECT`语句。

1.  然后会为`my_parent`行本身发出`DELETE`语句。

1.  数据库级别的`ON DELETE CASCADE`设置确保将删除所有引用受影响的`parent`行的`child`中的行。

1.  由`my_parent`引用的`Parent`实例，以及所有与该对象相关联且已**加载**（即执行了步骤 2）的`Child`实例，将从`Session`中解除关联。

注意

要使用“ON DELETE CASCADE”，底层数据库引擎必须支持`FOREIGN KEY`约束，并且它们必须被强制执行：

+   在使用 MySQL 时，必须选择适当的存储引擎。有关详细信息，请参见 CREATE TABLE arguments including Storage Engines。

+   在使用 SQLite 时，必须显式启用外键支持。有关详细信息，请参见 Foreign Key Support。### 使用外键 ON DELETE 处理多对多关系

正如在使用级联删除处理多对多关系中描述的那样，“删除”级联也适用于多对多关系。要利用`ON DELETE CASCADE`外键与多对多结合使用，需要在关联表上配置`FOREIGN KEY`指令。这些指令可以处理自动从关联表中删除，但不能自动删除相关对象本身。

在这种情况下，`relationship.passive_deletes`指令可以在删除操作期间节省一些额外的`SELECT`语句，但仍然有一些集合，ORM 将继续加载它们，以定位受影响的子对象并正确处理它们。

注意

对此的假设优化可以包括一次针对关联表的所有父关联行的单个`DELETE`语句，然后使用`RETURNING`来定位受影响的相关子行，但是这目前不是 ORM 工作单元实现的一部分。

在这个配置中，我们在关联表的两个外键约束上都配置了`ON DELETE CASCADE`。我们在父->子关系的一侧配置了`cascade="all, delete"`，然后我们可以在双向关系的**另一侧**上配置`passive_deletes=True`，如下所示：

```py
association_table = Table(
    "association",
    Base.metadata,
    Column("left_id", Integer, ForeignKey("left.id", ondelete="CASCADE")),
    Column("right_id", Integer, ForeignKey("right.id", ondelete="CASCADE")),
)

class Parent(Base):
    __tablename__ = "left"
    id = mapped_column(Integer, primary_key=True)
    children = relationship(
        "Child",
        secondary=association_table,
        back_populates="parents",
        cascade="all, delete",
    )

class Child(Base):
    __tablename__ = "right"
    id = mapped_column(Integer, primary_key=True)
    parents = relationship(
        "Parent",
        secondary=association_table,
        back_populates="children",
        passive_deletes=True,
    )
```

使用上述配置，删除`Parent`对象的操作如下：

1.  使用`Session.delete()`标记要删除的`Parent`对象。

1.  当发生刷新时，如果未加载`Parent.children`集合，则 ORM 将首先发出 SELECT 语句，以加载与`Parent.children`对应的`Child`对象。

1.  然后，将发出针对与该父行对应的`association`中的行的`DELETE`语句。

1.  对于每个受此立即删除影响的`Child`对象，由于配置了`passive_deletes=True`，工作单元将不需要尝试为每个`Child.parents`集合发出 SELECT 语句，因为假定将删除`association`中的相应行。

1.  对于从`Parent.children`加载的每个`Child`对象，都会发出`DELETE`语句。  ## delete-orphan

`delete-orphan`级联会为`delete`级联添加行为，这样当子对象与父对象取消关联时，子对象将被标记为删除，而不仅仅是在父对象被标记为删除时。当处理一个与父对象“拥有”关系的相关对象时，这是一种常见的特性，该关系具有 NOT NULL 外键，因此从父集合中删除项目会导致其被删除。

`delete-orphan`级联意味着每个子对象一次只能有一个父对象，并且在绝大多数情况下，它只配置在一对多关系上。对于设置在多对一或多对多关系上的非常罕见的情况，可以通过配置`relationship.single_parent`参数来强制“多”端一次只允许一个对象，该参数建立了 Python 端验证，确保对象一次只与一个父对象关联，但这大大限制了“多”关系的功能，通常不是所需的。

另请参阅

对于关系<relationship>，delete-orphan 级联通常仅配置在一对多关系的“一”端，而不是多对一或多对多关系的“多”端。 - 关于涉及 delete-orphan 级联的常见错误场景的背景。  ## merge

`merge`级联表示应该从`Session.merge()`操作从调用`Session.merge()`的主体父对象向下传播到引用的对象。这个级联也是默认开启的。  ## refresh-expire

`refresh-expire` 是一个不常见的选项，表示`Session.refresh()`操作应该从父对象传播到引用的对象。当使用`Session.refresh()`时，引用的对象仅被过期，而不会实际刷新。## expunge

`expunge` 级联表示当使用`Session.expunge()`从`Session`中移除父对象时，操作应该向下传播到引用的对象。## 删除说明 - 删除从集合和标量关系引用的对象

通常情况下，ORM 在刷新过程中不会修改集合或标量关系的内容。这意味着，如果你的类有一个指向对象集合的`relationship()`，或者一个指向单个对象的引用，比如一对多关系，那么当刷新过程发生时，这个属性的内容不会被修改。相反，预期的是`Session`最终会过期，要么通过`Session.commit()`的提交时过期行为，要么通过显式使用`Session.expire()`。在那时，与该`Session`关联的任何引用对象或集合将被清除，并且在下次访问时将重新加载自己。

关于此行为常见的混淆涉及 `Session.delete()` 方法的使用。当调用 `Session.delete()` 删除一个对象并且刷新了 `Session` 时，该行将从数据库中删除。通过外键引用目标行的行，假设它们使用两个映射对象类型之间的 `relationship()` 跟踪，还将看到它们的外键属性被更新为 null，或者如果设置了级联删除，则相关行也将被删除。然而，即使与被删除对象相关的行可能也被修改，**在刷新本身的范围内，涉及操作的关系绑定集合或对象引用上不会发生任何更改**。这意味着如果对象是相关集合的成员，则在 Python 端它仍然存在，直到该集合过期。同样，如果对象通过多对一或一对一从另一个对象引用，那个引用也将保留在该对象上，直到该对象也过期。

在下面的例子中，我们可以看到，即使将一个 `Address` 对象标记为删除，在刷新后，它仍然存在于与父 `User` 关联的集合中：

```py
>>> address = user.addresses[1]
>>> session.delete(address)
>>> session.flush()
>>> address in user.addresses
True
```

当上述会话提交时，所有属性都将过期。下一次访问 `user.addresses` 将重新加载集合，显示所需的状态：

```py
>>> session.commit()
>>> address in user.addresses
False
```

拦截 `Session.delete()` 并自动调用其过期的方法有一种方法；请参阅 [ExpireRelationshipOnFKChange](https://www.sqlalchemy.org/trac/wiki/UsageRecipes/ExpireRelationshipOnFKChange) 查看详情。然而，通常的做法是在集合内删除项目时直接放弃使用 `Session.delete()`，而是使用级联行为自动调用删除操作，因为将对象从父集合中删除的结果。`delete-orphan` 级联可以实现这一点，如下例所示：

```py
class User(Base):
    __tablename__ = "user"

    # ...

    addresses = relationship("Address", cascade="all, delete-orphan")

# ...

del user.addresses[1]
session.flush()
```

在上面的情况中，从 `User.addresses` 集合中移除 `Address` 对象后，`delete-orphan` 级联的效果与将其传递给 `Session.delete()` 相同，标记了 `Address` 对象以删除。

`delete-orphan` 级联也可以应用于多对一或一对一关系，这样当一个对象从其父对象中取消关联时，它也会自动标记为删除。在多对一或一对一关系上使用`delete-orphan`级联需要额外的标志`relationship.single_parent`，该标志会触发一个断言，即此相关对象不应同时与任何其他父对象共享：

```py
class User(Base):
    # ...

    preference = relationship(
        "Preference", cascade="all, delete-orphan", single_parent=True
    )
```

如果上面的情况下，一个假设的`Preference`对象从一个`User`中移除，它将在 flush 时被删除：

```py
some_user.preference = None
session.flush()  # will delete the Preference object
```

另请参阅

有关级联的详细信息，请参阅 Cascades。## save-update

`save-update` 级联表示当通过`Session.add()`将对象放入`Session`时，通过此`relationship()`与之关联的所有对象也应该被添加到同一个`Session`中。假设我们有一个对象`user1`，它有两个相关对象`address1`、`address2`：

```py
>>> user1 = User()
>>> address1, address2 = Address(), Address()
>>> user1.addresses = [address1, address2]
```

如果我们将`user1`添加到`Session`中，它也会隐式添加`address1`、`address2`：

```py
>>> sess = Session()
>>> sess.add(user1)
>>> address1 in sess
True
```

`save-update` 级联也会影响已经存在于`Session`中的对象的属性操作。如果我们向`user1.addresses`集合添加第三个对象`address3`，它将成为该`Session`的状态的一部分：

```py
>>> address3 = Address()
>>> user1.addresses.append(address3)
>>> address3 in sess
True
```

当从集合中删除项目或将对象与标量属性取消关联时，`save-update` 级联可能会表现出令人惊讶的行为。在某些情况下，被孤立的对象仍然可能被拉入原父对象的`Session`；这是为了 flush 进程能够适当处理该相关对象。这种情况通常只会在对象从一个`Session`中移除并添加到另一个`Session`时出现：

```py
>>> user1 = sess1.scalars(select(User).filter_by(id=1)).first()
>>> address1 = user1.addresses[0]
>>> sess1.close()  # user1, address1 no longer associated with sess1
>>> user1.addresses.remove(address1)  # address1 no longer associated with user1
>>> sess2 = Session()
>>> sess2.add(user1)  # ... but it still gets added to the new session,
>>> address1 in sess2  # because it's still "pending" for flush
True
```

`save-update` 级联默认启用，并且通常被视为理所当然；它通过允许对那个`Session`一次注册整个对象结构的单个调用来简化代码。虽然它可以被禁用，但通常没有必要这样做。

### 具有双向关系的 save-update 级联的行为

`save-update`级联在双向关系的情况下**单向**发生，即当使用`relationship.back_populates`或`relationship.backref`参数创建两个相互引用的`relationship()`对象时。

当将一个未与`Session`关联的对象分配给与`Session`关联的父对象的属性或集合时，该对象将自动添加到同一个`Session`中。然而，反向操作不会产生这种效果；当分配一个未与`Session`关联的对象时，分配给一个与`Session`关联的子对象，不会自动将父对象添加到`Session`中。这种行为的总体主题被称为“级联反向引用”，代表了作为 SQLAlchemy 2.0 的标准化行为的变化。

为了说明，假设有一系列通过关系`Order.items`和`Item.order`与`Item`对象双向关联的`Order`对象的映射：

```py
mapper_registry.map_imperatively(
    Order,
    order_table,
    properties={"items": relationship(Item, back_populates="order")},
)

mapper_registry.map_imperatively(
    Item,
    item_table,
    properties={"order": relationship(Order, back_populates="items")},
)
```

如果`Order`已经与一个`Session`关联，并且然后创建一个`Item`对象并附加到该`Order`的`Order.items`集合中，`Item`将自动级联到相同的`Session`中：

```py
>>> o1 = Order()
>>> session.add(o1)
>>> o1 in session
True

>>> i1 = Item()
>>> o1.items.append(i1)
>>> o1 is i1.order
True
>>> i1 in session
True
```

在上面的例子中，`Order.items`和`Item.order`的双向性意味着附加到`Order.items`也会赋值给`Item.order`。同时，`save-update`级联允许将`Item`对象添加到与父`Order`已关联的同一个`Session`中。

然而，如果上述操作在**反向**方向进行，即将`Item.order`赋值而不是直接附加到`Order.item`，则级联操作不会自动进行到`Session`中，即使对象赋值`Order.items`和`Item.order`的状态与前面的示例相同：

```py
>>> o1 = Order()
>>> session.add(o1)
>>> o1 in session
True

>>> i1 = Item()
>>> i1.order = o1
>>> i1 in order.items
True
>>> i1 in session
False
```

在上述情况下，在创建`Item`对象并设置所有所需状态之后，应明确将其添加到`Session`中：

```py
>>> session.add(i1)
```

在旧版本的 SQLAlchemy 中，保存-更新级 learning method 会在所有情况下双向发生。然后，通过一个名为`cascade_backrefs`的选项将其变为可选。最后，在 SQLAlchemy 1.4 中，旧行为被弃用，并且在 SQLAlchemy 2.0 中删除了`cascade_backrefs`选项。其理由是，用户通常不会觉得在对象的属性上赋值（如上面所示的`i1.order = o1`的赋值）会改变该对象`i1`的持久化状态，使其现在处于`Session`中处于挂起状态，并且经常会出现自动刷新会过早刷新对象并导致错误的情况，在这些情况下，给定对象仍在构建中且尚未处于准备好刷新的状态状态。选择单向和双向行为之间的选项也被删除，因为此选项创建了两种略有不同的工作方式，增加了 ORM 的整体学习曲线以及文档和用户支持负担。

另请参阅

在 2.0 中弃用了 cascade_backrefs 行为 - 关于“级联 backrefs”行为变更的背景 ### 双向关系中保存-更新级联的行为

`save-update`级联在双向关系的上下文中**单向**发生，即在使用`relationship.back_populates`或`relationship.backref`参数创建相互引用的两个单独的`relationship()`对象时。

一个未与`Session`相关联的对象，当分配给与`Session`相关联的父对象的属性或集合时，将自动添加到相同的`Session`中。但是，相反的操作不会产生这种效果；一个未与`Session`相关联的对象，其中一个与`Session`相关联的子对象被分配，将不会自动将该父对象添加到`Session`中。此行为的整体主题称为“级联反向引用”，并代表了作为 SQLAlchemy 2.0 的标准化行为的变化。

为了说明，假设有一系列通过关系`Order.items`和`Item.order`与`Item`对象双向关联的`Order`对象的映射：

```py
mapper_registry.map_imperatively(
    Order,
    order_table,
    properties={"items": relationship(Item, back_populates="order")},
)

mapper_registry.map_imperatively(
    Item,
    item_table,
    properties={"order": relationship(Order, back_populates="items")},
)
```

如果`Order`已与`Session`相关联，并且然后创建`Item`对象并将其附加到该`Order`的`Order.items`集合中，那么`Item`将自动级联到相同的`Session`中：

```py
>>> o1 = Order()
>>> session.add(o1)
>>> o1 in session
True

>>> i1 = Item()
>>> o1.items.append(i1)
>>> o1 is i1.order
True
>>> i1 in session
True
```

上述案例中，`Order.items`和`Item.order`的双向性意味着附加到`Order.items`也会赋值给`Item.order`。同时，`save-update`级联允许将`Item`对象添加到与父`Order`已关联的相同`Session`中。

但是，如果上述操作是以**相反**方向执行的，即将`Item.order`赋值而不是直接附加到`Order.item`，则即使对象分配`Order.items`和`Item.order`的状态与前面的示例相同，也不会自动进入到`Session`的级联操作中：

```py
>>> o1 = Order()
>>> session.add(o1)
>>> o1 in session
True

>>> i1 = Item()
>>> i1.order = o1
>>> i1 in order.items
True
>>> i1 in session
False
```

在上述情况下，在创建`Item`对象并设置所有所需状态之后，应明确将其添加到`Session`中：

```py
>>> session.add(i1)
```

在较旧版本的 SQLAlchemy 中，保存-更新级联在所有情况下都会双向发生。然后，使用称为`cascade_backrefs`的选项将其变为可选。最后，在 SQLAlchemy 1.4 中，旧行为被弃用，并且在 SQLAlchemy 2.0 中删除了`cascade_backrefs`选项。其理由是用户通常不会觉得将对象的属性分配给对象上的属性是直观的，如上面所示的`i1.order = o1`的赋值会改变对象`i1`的持久化状态，使其现在处于`Session`中处于挂起状态，并且在那些给定对象仍在构建并且尚未准备好被刷新的情况下，会经常出现自动刷新会过早刷新对象并导致错误的情况。选择单向和双向行为之间的选项也被删除，因为此选项创建了两种略有不同的工作方式，增加了 ORM 的整体学习曲线以及文档和用户支持负担。

另请参阅

2.0 中将删除的 cascade_backrefs 行为已弃用 - 关于“级联反向引用”行为变更的背景信息

## 删除

`删除`级联表示当“父”对象标记为删除时，其相关的“子”对象也应标记为删除。例如，如果我们有一个配置了`删除`级联的关系`User.addresses`：

```py
class User(Base):
    # ...

    addresses = relationship("Address", cascade="all, delete")
```

如果使用上述映射，我们有一个`User`对象和两个相关的`Address`对象：

```py
>>> user1 = sess1.scalars(select(User).filter_by(id=1)).first()
>>> address1, address2 = user1.addresses
```

如果我们标记`user1`进行删除，在刷新操作进行后，`address1`和`address2`也将被删除：

```py
>>> sess.delete(user1)
>>> sess.commit()
DELETE  FROM  address  WHERE  address.id  =  ?
((1,),  (2,))
DELETE  FROM  user  WHERE  user.id  =  ?
(1,)
COMMIT 
```

或者，如果我们的`User.addresses`关系*没有*`删除`级联，SQLAlchemy 的默认行为是通过将它们的外键引用设置为`NULL`来解除`user1`与`address1`和`address2`的关联。使用以下映射：

```py
class User(Base):
    # ...

    addresses = relationship("Address")
```

在删除父`User`对象时，`address`中的行不会被删除，而是被解除关联：

```py
>>> sess.delete(user1)
>>> sess.commit()
UPDATE  address  SET  user_id=?  WHERE  address.id  =  ?
(None,  1)
UPDATE  address  SET  user_id=?  WHERE  address.id  =  ?
(None,  2)
DELETE  FROM  user  WHERE  user.id  =  ?
(1,)
COMMIT 
```

删除 在一对多关系上的级联通常与删除孤儿级联结合使用，如果“子”对象与父对象解除关联，则会发出与相关行相关的 DELETE 操作。`删除`和`删除孤儿`级联的组合涵盖了 SQLAlchemy 需要在将外键列设置为 NULL 与完全删除行之间做出决定的情况。

默认情况下，该功能完全独立于数据库配置的可能配置`CASCADE`行为的`FOREIGN KEY`约束。为了更有效地与此配置集成，应使用在使用 ORM 关系的外键 ON DELETE 级联中描述的附加指令。

警告

请注意，ORM 的“删除”和“删除孤立对象”行为仅适用于使用`Session.delete()`方法在工作单元过程中标记个别 ORM 实例进行删除。它不适用于“批量”删除，这将使用`delete()`构造来发出，如 ORM UPDATE and DELETE with Custom WHERE Criteria 中所示。有关更多背景信息，请参见 ORM-启用的更新和删除的重要说明和注意事项。

另见

使用 ORM 关系的外键 ON DELETE 级联

使用多对多关系的级联删除

delete-orphan

### 使用多对多关系的级联删除

`cascade="all, delete"`选项与多对多关系同样适用，这种关系使用`relationship.secondary`来指示一个关联表。当删除父对象时，因此取消与其相关的对象的关联时，工作单元过程通常会从关联表中删除行，但会保留相关的对象。当与`cascade="all, delete"`组合时，额外的`DELETE`语句将对子行本身进行操作。

下面的示例调整了多对多的例子，以说明关联的**一**端上的`cascade="all, delete"`设置：

```py
association_table = Table(
    "association",
    Base.metadata,
    Column("left_id", Integer, ForeignKey("left.id")),
    Column("right_id", Integer, ForeignKey("right.id")),
)

class Parent(Base):
    __tablename__ = "left"
    id = mapped_column(Integer, primary_key=True)
    children = relationship(
        "Child",
        secondary=association_table,
        back_populates="parents",
        cascade="all, delete",
    )

class Child(Base):
    __tablename__ = "right"
    id = mapped_column(Integer, primary_key=True)
    parents = relationship(
        "Parent",
        secondary=association_table,
        back_populates="children",
    )
```

当使用`Session.delete()`标记要删除的`Parent`对象时，上述情况下，刷新过程通常会从`association`表中删除关联行，但根据级联规则，它还将删除所有相关的`Child`行。

警告

如果上述`cascade="all, delete"`设置在**两个**关系上都配置了，则级联操作将继续通过所有`Parent`和`Child`对象，加载遇到的每个`children`和`parents`集合，并删除所有连接的内容。通常不希望将“删除”级联配置为双向。

另见

从多对多表中删除行

使用外键 ON DELETE 与多对多关系  ### 使用 ORM 关系的外键 ON DELETE 级联

SQLAlchemy 的“delete”级联行为与数据库`FOREIGN KEY`约束的`ON DELETE`特性重叠。SQLAlchemy 允许使用`ForeignKey`和`ForeignKeyConstraint`构造配置这些模式级 DDL 行为；与`Table`元数据一起使用这些对象的用法在 ON UPDATE and ON DELETE 中有描述。

为了在与`relationship()`一起使用`ON DELETE`外键级联时，首先需要注意的是`relationship.cascade`设置仍然必须配置为与所需的`delete`或“set null”行为匹配（使用`delete`级联或将其省略），以便 ORM 或数据库级约束将处理实际修改数据库中数据的任务时，ORM 仍然能够适当跟踪可能受影响的本地存在的对象的状态。

然后，在`relationship()`上有一个附加选项，指示 ORM 应该尝试自己运行 DELETE/UPDATE 操作相关行的程度，还是应该依赖于期望数据库端 FOREIGN KEY 约束级联处理任务；这是`relationship.passive_deletes`参数，它接受`False`（默认值），`True`和`"all"`选项。

最典型的示例是，在删除父行时删除子行，并且在相关的`FOREIGN KEY`约束上配置了`ON DELETE CASCADE`：

```py
class Parent(Base):
    __tablename__ = "parent"
    id = mapped_column(Integer, primary_key=True)
    children = relationship(
        "Child",
        back_populates="parent",
        cascade="all, delete",
        passive_deletes=True,
    )

class Child(Base):
    __tablename__ = "child"
    id = mapped_column(Integer, primary_key=True)
    parent_id = mapped_column(Integer, ForeignKey("parent.id", ondelete="CASCADE"))
    parent = relationship("Parent", back_populates="children")
```

当删除父行时，上述配置的行为如下：

1.  应用程序调用`session.delete(my_parent)`，其中`my_parent`是`Parent`类的一个实例。

1.  当`Session`下次将更改刷新到数据库时，`my_parent.children`集合中的**当前加载的**所有项目都将被 ORM 删除，这意味着为每个记录发出一个`DELETE`语句。

1.  如果 `my_parent.children` 集合**未加载**，则不会发出任何 `DELETE` 语句。如果在这个 `relationship()` 上**未**设置 `relationship.passive_deletes` 标志，则会发出一个用于未加载的 `Child` 对象的 `SELECT` 语句。

1.  然后为 `my_parent` 行本身发出一个 `DELETE` 语句。

1.  数据库级别的 `ON DELETE CASCADE` 设置确保了所有引用受影响的 `parent` 行的 `child` 行也被删除。

1.  由 `my_parent` 引用的 `Parent` 实例，以及与此对象相关且已经**加载**（即发生了步骤 2）的所有 `Child` 实例，都会从 `Session` 中解除关联。

注意

要使用“ON DELETE CASCADE”，底层数据库引擎必须支持`FOREIGN KEY`约束，并且它们必须是强制执行的：

+   当使用 MySQL 时，必须选择适当的存储引擎。详情请参阅包括存储引擎的 CREATE TABLE 参数。

+   当使用 SQLite 时，必须显式启用外键支持。详情请参阅外键支持。### 使用外键 ON DELETE 处理多对多关系

如 使用级联删除处理多对多关系 中所述，“delete”级联也适用于多对多关系。要使用 `ON DELETE CASCADE` 外键与多对多一起使用，必须在关联表上配置 `FOREIGN KEY` 指令。这些指令可以处理自动从关联表中删除，但不能自动删除相关对象本身。

在这种情况下，`relationship.passive_deletes` 指令可以在删除操作期间为我们节省一些额外的 `SELECT` 语句，但仍然有一些集合 ORM 将继续加载，以便定位受影响的子对象并正确处理它们。

注意

对此的假设优化可能包括一条针对关联表的所有父关联行的单个 `DELETE` 语句，然后使用 `RETURNING` 定位受影响的相关子行，但是这目前不是 ORM 工作单元实现的一部分。

在此配置中，我们在关联表的两个外键约束上都配置了 `ON DELETE CASCADE`。我们在关系的父->子方向上配置了 `cascade="all, delete"`，然后我们可以在双向关系的**另一侧**上配置 `passive_deletes=True`，如下所示：

```py
association_table = Table(
    "association",
    Base.metadata,
    Column("left_id", Integer, ForeignKey("left.id", ondelete="CASCADE")),
    Column("right_id", Integer, ForeignKey("right.id", ondelete="CASCADE")),
)

class Parent(Base):
    __tablename__ = "left"
    id = mapped_column(Integer, primary_key=True)
    children = relationship(
        "Child",
        secondary=association_table,
        back_populates="parents",
        cascade="all, delete",
    )

class Child(Base):
    __tablename__ = "right"
    id = mapped_column(Integer, primary_key=True)
    parents = relationship(
        "Parent",
        secondary=association_table,
        back_populates="children",
        passive_deletes=True,
    )
```

使用上述配置，删除 `Parent` 对象的过程如下：

1.  使用 `Session.delete()` 标记要删除的 `Parent` 对象。

1.  当刷新发生时，如果未加载 `Parent.children` 集合，则 ORM 将首先发出 SELECT 语句以加载与 `Parent.children` 对应的 `Child` 对象。

1.  然后会为对应于该父行的 `association` 中的行发出 `DELETE` 语句。

1.  对于由此立即删除受影响的每个 `Child` 对象，因为配置了 `passive_deletes=True`，工作单元不需要尝试为每个 `Child.parents` 集合发出 SELECT 语句，因为假设将删除 `association` 中对应的行。

1.  然后对从 `Parent.children` 加载的每个 `Child` 对象发出 `DELETE` 语句。 ### 使用删除级联处理多对多关系

`cascade="all, delete"` 选项与多对多关系同样有效，即使用 `relationship.secondary` 指示关联表的关系。当删除父对象并因此取消关联其相关对象时，工作单元进程通常会删除关联表中的行，但保留相关对象。当与 `cascade="all, delete"` 结合使用时，将为子行本身执行额外的 `DELETE` 语句。

以下示例将多对多的示例调整为示例，以说明在关联的**一侧**上设置 `cascade="all, delete"`。

```py
association_table = Table(
    "association",
    Base.metadata,
    Column("left_id", Integer, ForeignKey("left.id")),
    Column("right_id", Integer, ForeignKey("right.id")),
)

class Parent(Base):
    __tablename__ = "left"
    id = mapped_column(Integer, primary_key=True)
    children = relationship(
        "Child",
        secondary=association_table,
        back_populates="parents",
        cascade="all, delete",
    )

class Child(Base):
    __tablename__ = "right"
    id = mapped_column(Integer, primary_key=True)
    parents = relationship(
        "Parent",
        secondary=association_table,
        back_populates="children",
    )
```

上面，当使用 `Session.delete()` 标记要删除的 `Parent` 对象时，刷新过程将按照惯例从 `association` 表中删除相关行，但根据级联规则，它还将删除所有相关的 `Child` 行。

警告

如果上述 `cascade="all, delete"` 设置被配置在**两个**关系上，那么级联操作将继续通过所有 `Parent` 和 `Child` 对象进行级联，加载遇到的每个 `children` 和 `parents` 集合，并删除所有连接的内容。通常不希望双向配置“delete”级联。

另请参阅

从多对多表中删除行

使用外键 ON DELETE 处理多对多关系

### 使用 ORM 关系中的外键 ON DELETE 级联

SQLAlchemy 的“delete”级联的行为与数据库`FOREIGN KEY`约束的`ON DELETE`特性重叠。SQLAlchemy 允许使用 `ForeignKey` 和 `ForeignKeyConstraint` 构造配置这些模式级别的 DDL 行为；在与 `Table` 元数据结合使用这些对象的用法在 ON UPDATE and ON DELETE 中有描述。

要在 `relationship()` 中使用`ON DELETE`外键级联，首先要注意的是 `relationship.cascade` 设置必须仍然配置为匹配所需的“删除”或“设置为 null”行为（使用`delete`级联或将其省略），以便 ORM 或数据库级别的约束将处理实际修改数据库中的数据的任务时，ORM 仍将能够适当地跟踪可能受到影响的本地存在的对象的状态。

然后在`relationship()` 上有一个额外的选项，指示 ORM 应该尝试在相关行上自行运行 DELETE/UPDATE 操作的程度，而不是依靠期望数据库端 FOREIGN KEY 约束级联处理该任务；这是 `relationship.passive_deletes` 参数，它接受选项 `False`（默认值）、`True` 和 `"all"`。

最典型的例子是，在删除父行时要删除子行，并且在相关的`FOREIGN KEY`约束上配置了`ON DELETE CASCADE`：

```py
class Parent(Base):
    __tablename__ = "parent"
    id = mapped_column(Integer, primary_key=True)
    children = relationship(
        "Child",
        back_populates="parent",
        cascade="all, delete",
        passive_deletes=True,
    )

class Child(Base):
    __tablename__ = "child"
    id = mapped_column(Integer, primary_key=True)
    parent_id = mapped_column(Integer, ForeignKey("parent.id", ondelete="CASCADE"))
    parent = relationship("Parent", back_populates="children")
```

当删除父行时，上述配置的行为如下：

1.  应用程序调用`session.delete(my_parent)`，其中`my_parent`是`Parent`的实例。

1.  当 `Session` 下次将更改刷新到数据库时，`my_parent.children` 集合中的所有**当前加载的**项目都将被 ORM 删除，这意味着为每个记录发出了一个`DELETE`语句。

1.  如果`my_parent.children`集合**未加载**，则不会发出`DELETE`语句。 如果在此`relationship()`上未设置`relationship.passive_deletes`标志，那么将会发出一个针对未加载的`Child`对象的`SELECT`语句。

1.  针对`my_parent`行本身发出了一个`DELETE`语句。

1.  数据库级别的`ON DELETE CASCADE`设置确保了所有引用受影响的`parent`行的`child`中的行也被删除。

1.  由`my_parent`引用的`Parent`实例以及所有与此对象相关联且已**加载**的`Child`实例（即发生了步骤 2）都将从`Session`中解除关联。

注意

要使用“ON DELETE CASCADE”，底层数据库引擎必须支持`FOREIGN KEY`约束，并且它们必须是强制性的：

+   使用 MySQL 时，必须选择适当的存储引擎。 有关详细信息，请参阅 CREATE TABLE arguments including Storage Engines。

+   使用 SQLite 时，必须显式启用外键支持。 有关详细信息，请参阅 Foreign Key Support。

### 在多对多关系中使用外键 ON DELETE

如使用 delete cascade 与多对多关系所述，“delete”级联也适用于多对多关系。 要利用`ON DELETE CASCADE`外键与多对多关系，必须在关联表上配置`FOREIGN KEY`指令。 这些指令可以处理自动从关联表中删除，但无法适应相关对象本身的自动删除。

在这种情况下，`relationship.passive_deletes`指令可以在删除操作期间为我们节省一些额外的`SELECT`语句，但仍然有一些集合是 ORM 将继续加载的，以定位受影响的子对象并正确处理它们。

注意

对此的假设优化可以包括一次针对关联表的所有父关联行的单个`DELETE`语句，然后使用`RETURNING`来定位受影响的相关子行，但这目前不是 ORM 工作单元实现的一部分。

在此配置中，我们在关联表的两个外键约束上都配置了`ON DELETE CASCADE`。我们在关系的父->子方向上配置了`cascade="all, delete"`，然后我们可以在双向关系的**另一侧**上配置`passive_deletes=True`，如下所示：

```py
association_table = Table(
    "association",
    Base.metadata,
    Column("left_id", Integer, ForeignKey("left.id", ondelete="CASCADE")),
    Column("right_id", Integer, ForeignKey("right.id", ondelete="CASCADE")),
)

class Parent(Base):
    __tablename__ = "left"
    id = mapped_column(Integer, primary_key=True)
    children = relationship(
        "Child",
        secondary=association_table,
        back_populates="parents",
        cascade="all, delete",
    )

class Child(Base):
    __tablename__ = "right"
    id = mapped_column(Integer, primary_key=True)
    parents = relationship(
        "Parent",
        secondary=association_table,
        back_populates="children",
        passive_deletes=True,
    )
```

使用上述配置，删除 `Parent` 对象的过程如下：

1.  使用 `Session.delete()` 标记要删除的 `Parent` 对象。

1.  当发生刷新时，如果未加载 `Parent.children` 集合，则 ORM 首先会发出 SELECT 语句，以加载与 `Parent.children` 对应的 `Child` 对象。

1.  然后会为与该父行对应的 `association` 中的行发出 `DELETE` 语句。

1.  对于受此即时删除影响的每个 `Child` 对象，因为配置了 `passive_deletes=True`，工作单元不需要尝试为每个 `Child.parents` 集合发出 SELECT 语句，因为假设将删除 `association` 中的相应行。

1.  然后会为从 `Parent.children` 中加载的每个 `Child` 对象发出 `DELETE` 语句。

## 删除孤立

`delete-orphan` 级联为 `delete` 级联增加了行为，使得当子对象与父对象取消关联时，子对象将被标记为删除，而不仅仅是当父对象被标记为删除时。当处理由其父对象“拥有”的相关对象时，这是一个常见功能，具有非空的外键，以便从父集合中移除项目会导致其删除。

`delete-orphan` 级联意味着每个子对象一次只能有一个父对象，并且在绝大多数情况下仅配置在一对多关系上。在很少见的情况下，在多对一或多对多关系上设置它，“多”方可以通过配置 `relationship.single_parent` 参数，强制允许一次只有一个对象与父对象关联，从而在 Python 端建立验证，确保对象一次只与一个父对象关联，但这严重限制了“多”关系的功能，通常不是所期望的。

另请参阅

对于关系<relationship>，`delete-orphan` 级联通常仅配置在一对多关系的“一”方，并且不配置在多对一或多对多关系的“多”方。 - 关于涉及 `delete-orphan` 级联的常见错误场景的背景信息。

## 合并

`merge` 级联表示 `Session.merge()` 操作应从 `Session.merge()` 调用的主体父对象传播到引用对象。此级联默认也是打开的。

## 刷新-过期

`refresh-expire`是一个不常见的选项，表示`Session.expire()`操作应该从父对象传播到引用的对象。当使用`Session.refresh()`时，引用的对象只是过期了，而不是实际刷新了。

## 清除

`清除`级联指的是当父对象从`Session`中使用`Session.expunge()`移除时，该操作应传播到引用的对象。

## 删除注意事项 - 删除集合和标量关系中引用的对象

通常，ORM 在刷新过程中永远不会修改集合或标量关系的内容。这意味着，如果你的类有一个指向对象集合的`relationship()`，或者一个指向单个对象的引用，比如多对一，当刷新过程发生时，这个属性的内容不会被修改。相反，预计`Session`最终会过期，通过`Session.commit()`的提交时过期行为或通过`Session.expire()`的显式使用。在那时，与该`Session`相关联的任何引用对象或集合都将被清除，并在下次访问时重新加载自身。

关于这种行为产生的常见困惑涉及到`Session.delete()`方法的使用。当在一个对象上调用`Session.delete()`并且`Session`被刷新时，该行将从数据库中删除。通过外键引用目标行的行，假设它们是使用两个映射对象类型之间的`relationship()`进行跟踪的，也会看到它们的外键属性被更新为 null，或者如果设置了删除级联，相关行也将被删除。然而，即使与已删除对象相关的行可能也被修改，**在刷新范围内操作的对象上的关系绑定集合或对象引用不会发生任何更改**。这意味着如果对象是相关集合的成员，它将仍然存在于 Python 端，直到该集合过期为止。同样，如果对象通过另一个对象的多对一或一对一引用，则该引用也将保留在该对象上，直到该对象也过期为止。

下面，我们说明了在将`Address`对象标记为删除后，即使在刷新后，它仍然存在于与父`User`关联的集合中：

```py
>>> address = user.addresses[1]
>>> session.delete(address)
>>> session.flush()
>>> address in user.addresses
True
```

当以上会话提交时，所有属性都会过期。对`user.addresses`的下一次访问将重新加载集合，显示所需的状态：

```py
>>> session.commit()
>>> address in user.addresses
False
```

有一个拦截`Session.delete()`并自动调用此过期的配方；参见[ExpireRelationshipOnFKChange](https://www.sqlalchemy.org/trac/wiki/UsageRecipes/ExpireRelationshipOnFKChange)。然而，删除集合中的项目的通常做法是直接放弃使用`Session.delete()`，而是使用级联行为自动调用删除作为从父集合中删除对象的结果。`delete-orphan`级联实现了这一点，如下例所示：

```py
class User(Base):
    __tablename__ = "user"

    # ...

    addresses = relationship("Address", cascade="all, delete-orphan")

# ...

del user.addresses[1]
session.flush()
```

在上面的例子中，当从`User.addresses`集合中移除`Address`对象时，`delete-orphan`级联的效果与将其传递给`Session.delete()`相同，都将`Address`对象标记为删除。

`delete-orphan`级联也可以应用于多对一或一对一关系，这样当一个对象与其父对象解除关联时，它也会被自动标记为删除。在多对一或一对一关系上使用`delete-orphan`级联需要一个额外的标志`relationship.single_parent`，它调用一个断言，指出这个相关对象不会同时与任何其他父对象共享：

```py
class User(Base):
    # ...

    preference = relationship(
        "Preference", cascade="all, delete-orphan", single_parent=True
    )
```

上面，如果一个假设的`Preference`对象从一个`User`中移除，它将在刷新时被删除：

```py
some_user.preference = None
session.flush()  # will delete the Preference object
```

另请参阅

级联以获取级联的详细信息。
