# 处理大型集合

> 原文链接：[`docs.sqlalchemy.org/en/20/orm/large_collections.html`](https://docs.sqlalchemy.org/en/20/orm/large_collections.html)

`relationship()`的默认行为是根据配置的 加载策略 完全将集合内容加载到内存中，该加载策略控制何时以及如何从数据库加载这些内容。 相关集合可能不仅在访问时加载到内存中，或者急切地加载，而且在集合本身发生变化时以及在由工作单元系统删除所有者对象时也需要进行填充。

当相关集合可能非常大时，无论在任何情况下将这样的集合加载到内存中都可能不可行，因为这样的操作可能会过度消耗时间、网络和内存资源。

本节包括旨在允许`relationship()`与大型集合一起使用并保持足够性能的 API 特性。

## 仅写关系

**仅写**加载器策略是配置`relationship()`的主要方法，该方法将保持可写性，但不会加载其内容到内存中。 下面是使用现代类型注释的声明式形式的仅写 ORM 配置的示例：

```py
>>> from decimal import Decimal
>>> from datetime import datetime

>>> from sqlalchemy import ForeignKey
>>> from sqlalchemy import func
>>> from sqlalchemy.orm import DeclarativeBase
>>> from sqlalchemy.orm import Mapped
>>> from sqlalchemy.orm import mapped_column
>>> from sqlalchemy.orm import relationship
>>> from sqlalchemy.orm import Session
>>> from sqlalchemy.orm import WriteOnlyMapped

>>> class Base(DeclarativeBase):
...     pass

>>> class Account(Base):
...     __tablename__ = "account"
...     id: Mapped[int] = mapped_column(primary_key=True)
...     identifier: Mapped[str]
...
...     account_transactions: WriteOnlyMapped["AccountTransaction"] = relationship(
...         cascade="all, delete-orphan",
...         passive_deletes=True,
...         order_by="AccountTransaction.timestamp",
...     )
...
...     def __repr__(self):
...         return f"Account(identifier={self.identifier!r})"

>>> class AccountTransaction(Base):
...     __tablename__ = "account_transaction"
...     id: Mapped[int] = mapped_column(primary_key=True)
...     account_id: Mapped[int] = mapped_column(
...         ForeignKey("account.id", ondelete="cascade")
...     )
...     description: Mapped[str]
...     amount: Mapped[Decimal]
...     timestamp: Mapped[datetime] = mapped_column(default=func.now())
...
...     def __repr__(self):
...         return (
...             f"AccountTransaction(amount={self.amount:.2f}, "
...             f"timestamp={self.timestamp.isoformat()!r})"
...         )
...
...     __mapper_args__ = {"eager_defaults": True}
```

上述示例中，`account_transactions` 关系不是使用普通的`Mapped`注释配置的，而是使用`WriteOnlyMapped`类型注释配置的，在运行时会将 `lazy="write_only"` 的 加载策略 分配给目标 `relationship()`。 `WriteOnlyMapped` 注释是 `Mapped` 注释的替代形式，指示对象实例上使用 `WriteOnlyCollection` 集合类型。

上述`relationship()`配置还包括几个元素，这些元素是特定于删除 `Account` 对象时要采取的操作以及从 `account_transactions` 集合中移除 `AccountTransaction` 对象时要采取的操作。 这些元素包括：

+   `passive_deletes=True` - 允许工作单元在删除`Account`时无需加载集合；参见使用 ORM 关系进行外键级联删除。

+   在`ForeignKey`约束上配置`ondelete="cascade"`。这也在使用 ORM 关系进行外键级联删除中详细说明。

+   `cascade="all, delete-orphan"` - 指示工作单元在从集合中删除时删除`AccountTransaction`对象。请参见 delete-orphan 中的 Cascades 文档。

2.0 版本新增：“仅写入”关系加载器。

### 创建和持久化新的仅写入集合

写入-仅集合仅允许对瞬态或挂起对象直接分配集合。根据我们上面的映射，这表示我们可以创建一个新的`Account`对象，其中包含一系列要添加到`Session`中的`AccountTransaction`对象。任何 Python 可迭代对象都可以用作要开始的对象的来源，下面我们使用 Python `list`：

```py
>>> new_account = Account(
...     identifier="account_01",
...     account_transactions=[
...         AccountTransaction(description="initial deposit", amount=Decimal("500.00")),
...         AccountTransaction(description="transfer", amount=Decimal("1000.00")),
...         AccountTransaction(description="withdrawal", amount=Decimal("-29.50")),
...     ],
... )

>>> with Session(engine) as session:
...     session.add(new_account)
...     session.commit()
BEGIN  (implicit)
INSERT  INTO  account  (identifier)  VALUES  (?)
[...]  ('account_01',)
INSERT  INTO  account_transaction  (account_id,  description,  amount,  timestamp)
VALUES  (?,  ?,  ?,  CURRENT_TIMESTAMP)  RETURNING  id,  timestamp
[...  (insertmanyvalues)  1/3  (ordered;  batch  not  supported)]  (1,  'initial deposit',  500.0)
INSERT  INTO  account_transaction  (account_id,  description,  amount,  timestamp)
VALUES  (?,  ?,  ?,  CURRENT_TIMESTAMP)  RETURNING  id,  timestamp
[insertmanyvalues  2/3  (ordered;  batch  not  supported)]  (1,  'transfer',  1000.0)
INSERT  INTO  account_transaction  (account_id,  description,  amount,  timestamp)
VALUES  (?,  ?,  ?,  CURRENT_TIMESTAMP)  RETURNING  id,  timestamp
[insertmanyvalues  3/3  (ordered;  batch  not  supported)]  (1,  'withdrawal',  -29.5)
COMMIT 
```

一旦对象被持久化到数据库（即处于持久化或分离状态），该集合就具有扩展新项目的能力，以及删除单个项目的能力。但是，该集合可能**不再重新分配一个完整的替换集合**，因为这样的操作需要将先前的集合完全加载到内存中，以便将旧条目与新条目进行协调：

```py
>>> new_account.account_transactions = [
...     AccountTransaction(description="some transaction", amount=Decimal("10.00"))
... ]
Traceback (most recent call last):
...
sqlalchemy.exc.InvalidRequestError: Collection "Account.account_transactions" does not
support implicit iteration; collection replacement operations can't be used
```

### 向现有集合添加新项目

对于持久对象的写入-仅集合，使用工作单元过程对集合进行修改只能通过使用`WriteOnlyCollection.add()`、`WriteOnlyCollection.add_all()`和`WriteOnlyCollection.remove()`方法进行：

```py
>>> from sqlalchemy import select
>>> session = Session(engine, expire_on_commit=False)
>>> existing_account = session.scalar(select(Account).filter_by(identifier="account_01"))
BEGIN  (implicit)
SELECT  account.id,  account.identifier
FROM  account
WHERE  account.identifier  =  ?
[...]  ('account_01',)
>>> existing_account.account_transactions.add_all(
...     [
...         AccountTransaction(description="paycheck", amount=Decimal("2000.00")),
...         AccountTransaction(description="rent", amount=Decimal("-800.00")),
...     ]
... )
>>> session.commit()
INSERT  INTO  account_transaction  (account_id,  description,  amount,  timestamp)
VALUES  (?,  ?,  ?,  CURRENT_TIMESTAMP)  RETURNING  id,  timestamp
[...  (insertmanyvalues)  1/2  (ordered;  batch  not  supported)]  (1,  'paycheck',  2000.0)
INSERT  INTO  account_transaction  (account_id,  description,  amount,  timestamp)
VALUES  (?,  ?,  ?,  CURRENT_TIMESTAMP)  RETURNING  id,  timestamp
[insertmanyvalues  2/2  (ordered;  batch  not  supported)]  (1,  'rent',  -800.0)
COMMIT 
```

上述添加的项目将在`Session`中的挂起队列中保留，直到下一次刷新，在此刻它们将被插入到数据库中，假设添加的对象之前是瞬态的。

### 查询项目

`WriteOnlyCollection` 在任何时候都不会存储对集合当前内容的引用，也不具有直接发出 SELECT 到数据库以加载它们的行为；其覆盖的假设是集合可能包含数千或数百万行，并且不应作为任何其他操作的副作用而完全加载到内存中。

相反，`WriteOnlyCollection` 包括诸如`WriteOnlyCollection.select()`之类的生成 SQL 的助手，该方法将生成一个预先配置了当前父行的正确 WHERE / FROM 条件的`Select`构造，然后可以进一步修改以选择所需的任何行范围，以及使用像服务器端游标之类的特性来调用以便以内存高效的方式迭代完整集合的进程。

下面是生成的语句的示例。请注意，它还包括在示例映射中由`relationship.order_by`参数指示的 ORDER BY 条件；如果未配置该参数，则将省略此条件：

```py
>>> print(existing_account.account_transactions.select())
SELECT  account_transaction.id,  account_transaction.account_id,  account_transaction.description,
account_transaction.amount,  account_transaction.timestamp
FROM  account_transaction
WHERE  :param_1  =  account_transaction.account_id  ORDER  BY  account_transaction.timestamp 
```

我们可以使用这个`Select`构造与`Session`一起来查询`AccountTransaction`对象，最容易的是使用`Session.scalars()`方法，该方法将返回直接生成 ORM 对象的`Result`。通常，但不是必须的，`Select`可能会进一步修改以限制返回的记录；在下面的示例中，还添加了额外的 WHERE 条件，以仅加载“debit”账户交易，以及“LIMIT 10”以仅检索前十行：

```py
>>> account_transactions = session.scalars(
...     existing_account.account_transactions.select()
...     .where(AccountTransaction.amount < 0)
...     .limit(10)
... ).all()
BEGIN  (implicit)
SELECT  account_transaction.id,  account_transaction.account_id,  account_transaction.description,
account_transaction.amount,  account_transaction.timestamp
FROM  account_transaction
WHERE  ?  =  account_transaction.account_id  AND  account_transaction.amount  <  ?
ORDER  BY  account_transaction.timestamp  LIMIT  ?  OFFSET  ?
[...]  (1,  0,  10,  0)
>>> print(account_transactions)
[AccountTransaction(amount=-29.50, timestamp='...'), AccountTransaction(amount=-800.00, timestamp='...')]
```

### 删除项目

在当前`Session`中加载的个体项可能会被标记为要从集合中删除，使用`WriteOnlyCollection.remove()`方法。当操作继续时，刷新过程将隐式地将对象视为已经是集合的一部分。下面的示例说明了如何删除单个`AccountTransaction`项，根据级联设置，将导致删除该行：

```py
>>> existing_transaction = account_transactions[0]
>>> existing_account.account_transactions.remove(existing_transaction)
>>> session.commit()
DELETE  FROM  account_transaction  WHERE  account_transaction.id  =  ?
[...]  (3,)
COMMIT 
```

与任何 ORM 映射的集合一样，对象的删除可以按照解除与集合的关联并将对象保留在数据库中的方式进行，也可以根据`relationship()`的 delete-orphan 配置发出其行的 DELETE。

在不删除的情况下删除集合涉及将外键列设置为 NULL 以进行一对多关系，或者删除相应的关联行以进行多对多关系。

### 新项目的批量插入

`WriteOnlyCollection`可以生成 DML 构造，例如`Insert`对象，可在 ORM 上下文中使用以产生批量插入行为。请参阅 ORM 批量 INSERT 语句部分，了解 ORM 批量插入的概述。

#### 一对多集合

仅针对**常规的一对多集合**，`WriteOnlyCollection.insert()`方法将生成一个预先建立了与父对象相对应的 VALUES 条件的`Insert`构造。由于这个 VALUES 条件完全针对相关表，因此该语句可用于插入新的行，这些新行同时将成为相关集合中的新记录：

```py
>>> session.execute(
...     existing_account.account_transactions.insert(),
...     [
...         {"description": "transaction 1", "amount": Decimal("47.50")},
...         {"description": "transaction 2", "amount": Decimal("-501.25")},
...         {"description": "transaction 3", "amount": Decimal("1800.00")},
...         {"description": "transaction 4", "amount": Decimal("-300.00")},
...     ],
... )
BEGIN  (implicit)
INSERT  INTO  account_transaction  (account_id,  description,  amount,  timestamp)  VALUES  (?,  ?,  ?,  CURRENT_TIMESTAMP)
[...]  [(1,  'transaction 1',  47.5),  (1,  'transaction 2',  -501.25),  (1,  'transaction 3',  1800.0),  (1,  'transaction 4',  -300.0)]
<...>
>>> session.commit()
COMMIT
```

另请参阅

ORM 批量 INSERT 语句 - 在 ORM 查询指南中

一对多 - 在基本关系模式中

#### 多对多集合

对于一个**多对多集合**，两个类之间的关系涉及一个使用`relationship.secondary`参数配置的第三个表的情况，通过`WriteOnlyCollection.add_all()`方法，可以先分别批量插入新记录，然后检索它们，并将这些记录传递给`WriteOnlyCollection.add_all()`方法，单位操作过程将继续将它们作为集合的一部分进行持久化。

假设一个类`BankAudit`使用一个多对多表引用了许多`AccountTransaction`记录：

```py
>>> from sqlalchemy import Table, Column
>>> audit_to_transaction = Table(
...     "audit_transaction",
...     Base.metadata,
...     Column("audit_id", ForeignKey("audit.id", ondelete="CASCADE"), primary_key=True),
...     Column(
...         "transaction_id",
...         ForeignKey("account_transaction.id", ondelete="CASCADE"),
...         primary_key=True,
...     ),
... )
>>> class BankAudit(Base):
...     __tablename__ = "audit"
...     id: Mapped[int] = mapped_column(primary_key=True)
...     account_transactions: WriteOnlyMapped["AccountTransaction"] = relationship(
...         secondary=audit_to_transaction, passive_deletes=True
...     )
```

为了说明这两个操作，我们使用批量插入添加更多的`AccountTransaction`对象，通过在批量插入语句中添加`returning(AccountTransaction)`来使用 RETURNING 检索它们（请注意，我们也可以同样轻松地使用现有的`AccountTransaction`对象）：

```py
>>> new_transactions = session.scalars(
...     existing_account.account_transactions.insert().returning(AccountTransaction),
...     [
...         {"description": "odd trans 1", "amount": Decimal("50000.00")},
...         {"description": "odd trans 2", "amount": Decimal("25000.00")},
...         {"description": "odd trans 3", "amount": Decimal("45.00")},
...     ],
... ).all()
BEGIN  (implicit)
INSERT  INTO  account_transaction  (account_id,  description,  amount,  timestamp)  VALUES
(?,  ?,  ?,  CURRENT_TIMESTAMP),  (?,  ?,  ?,  CURRENT_TIMESTAMP),  (?,  ?,  ?,  CURRENT_TIMESTAMP)
RETURNING  id,  account_id,  description,  amount,  timestamp
[...]  (1,  'odd trans 1',  50000.0,  1,  'odd trans 2',  25000.0,  1,  'odd trans 3',  45.0) 
```

准备好一个`AccountTransaction`对象列表后，可以使用`WriteOnlyCollection.add_all()`方法一次性将许多行与一个新的`BankAudit`对象关联起来：

```py
>>> bank_audit = BankAudit()
>>> session.add(bank_audit)
>>> bank_audit.account_transactions.add_all(new_transactions)
>>> session.commit()
INSERT  INTO  audit  DEFAULT  VALUES
[...]  ()
INSERT  INTO  audit_transaction  (audit_id,  transaction_id)  VALUES  (?,  ?)
[...]  [(1,  10),  (1,  11),  (1,  12)]
COMMIT 
```

另请参见

ORM 批量插入语句 - 在 ORM 查询指南中

多对多 - 在基本关系模式中

### 项目的批量更新和删除

类似于`WriteOnlyCollection`可以预先建立 WHERE 条件生成`Select`构造的方式，它也可以生成具有相同 WHERE 条件的`Update`和`Delete`构造，以允许针对大集合中的元素进行基于条件的 UPDATE 和 DELETE 语句。

#### 一对多集合

就像插入（INSERT）一样，这个特性在**一对多集合**中最直接。

在下面的示例中，使用`WriteOnlyCollection.update()`方法生成一个 UPDATE 语句，针对集合中的元素，定位“amount”等于`-800`的行，并将`200`的数量添加到它们中：

```py
>>> session.execute(
...     existing_account.account_transactions.update()
...     .values(amount=AccountTransaction.amount + 200)
...     .where(AccountTransaction.amount == -800),
... )
BEGIN  (implicit)
UPDATE  account_transaction  SET  amount=(account_transaction.amount  +  ?)
WHERE  ?  =  account_transaction.account_id  AND  account_transaction.amount  =  ?
[...]  (200,  1,  -800)
<...>
```

类似地，`WriteOnlyCollection.delete()`将生成一个 DELETE 语句，以相同的方式调用：

```py
>>> session.execute(
...     existing_account.account_transactions.delete().where(
...         AccountTransaction.amount.between(0, 30)
...     ),
... )
DELETE  FROM  account_transaction  WHERE  ?  =  account_transaction.account_id
AND  account_transaction.amount  BETWEEN  ?  AND  ?  RETURNING  id
[...]  (1,  0,  30)
<...> 
```

#### **多对多集合**

提示

这里的技术涉及到稍微高级的多表更新表达式。

对于**多对多集合**的批量更新和删除，为了使 UPDATE 或 DELETE 语句与父对象的主键相关联，关联表必须明确地成为 UPDATE/DELETE 语句的一部分，这要求后端包括对非标准 SQL 语法的支持，或者在构造 UPDATE 或 DELETE 语句时需要额外的显式步骤。

对于支持多表版本的 UPDATE 的后端，`WriteOnlyCollection.update()`方法应该可以在多对多集合上工作，就像下面的示例中对`AccountTransaction`对象进行的 UPDATE 一样，涉及多对多的`BankAudit.account_transactions`集合：

```py
>>> session.execute(
...     bank_audit.account_transactions.update().values(
...         description=AccountTransaction.description + " (audited)"
...     )
... )
UPDATE  account_transaction  SET  description=(account_transaction.description  ||  ?)
FROM  audit_transaction  WHERE  ?  =  audit_transaction.audit_id
AND  account_transaction.id  =  audit_transaction.transaction_id  RETURNING  id
[...]  (' (audited)',  1)
<...>
```

上述语句自动使用“UPDATE..FROM”语法，由 SQLite 和其他后端支持，在 WHERE 子句中命名附加的`audit_transaction`表。

要更新或删除多对多集合，其中不支持多表语法的情况下，多对多条件可以移动到 SELECT 中，例如可以与 IN 组合以匹配行。`WriteOnlyCollection`在这里仍然对我们有所帮助，因为我们使用`WriteOnlyCollection.select()`方法为我们生成此 SELECT，利用`Select.with_only_columns()`方法生成标量子查询：

```py
>>> from sqlalchemy import update
>>> subq = bank_audit.account_transactions.select().with_only_columns(AccountTransaction.id)
>>> session.execute(
...     update(AccountTransaction)
...     .values(description=AccountTransaction.description + " (audited)")
...     .where(AccountTransaction.id.in_(subq))
... )
UPDATE  account_transaction  SET  description=(account_transaction.description  ||  ?)
WHERE  account_transaction.id  IN  (SELECT  account_transaction.id
FROM  audit_transaction
WHERE  ?  =  audit_transaction.audit_id  AND  account_transaction.id  =  audit_transaction.transaction_id)
RETURNING  id
[...]  (' (audited)',  1)
<...> 
```

### 只写集合 - API 文档

| 对象名称 | 描述 |
| --- | --- |
| WriteOnlyCollection | 只写集合可以将更改同步到属性事件系统中。 |
| WriteOnlyMapped | 代表“只写”关系的 ORM 映射属性类型。 |

```py
class sqlalchemy.orm.WriteOnlyCollection
```

只写集合可以将更改同步到属性事件系统中。

使用`WriteOnlyCollection`在映射中使用`"write_only"`延迟加载策略与`relationship()`一起。有关此配置的背景，请参阅只写关系。

2.0 版本中的新功能。

另请参阅

只写关系

**成员**

add(), add_all(), delete(), insert(), remove(), select(), update()

**类签名**

类`sqlalchemy.orm.WriteOnlyCollection` (`sqlalchemy.orm.writeonly.AbstractCollectionWriter`)

```py
method add(item: _T) → None
```

将一个项添加到此`WriteOnlyCollection`中。

给定项将在下一个刷新时以父实例的集合的形式持久化到数据库中。

```py
method add_all(iterator: Iterable[_T]) → None
```

将一个可迭代的项添加到此`WriteOnlyCollection`中。

给定的项将在下一个刷新时以父实例的集合的形式持久化到数据库中。

```py
method delete() → Delete
```

生成一个`Delete`，该语句将以此实例本地的`WriteOnlyCollection`的形式引用行。

```py
method insert() → Insert
```

对于一对多的集合，生成一个`Insert`，该语句将以此实例本地的`WriteOnlyCollection`的形式插入新的行。

此构造仅支持不包括`relationship.secondary`参数的`Relationship`。对于指向多对多表的关系，请使用普通的批量插入技术来生成新对象，然后使用`AbstractCollectionWriter.add_all()`将它们与集合关联起来。

```py
method remove(item: _T) → None
```

从此`WriteOnlyCollection`中移除一个项。

下一个刷新时，给定项将从父实例的集合中移除。

```py
method select() → Select[Tuple[_T]]
```

生成一个`Select`构造，表示此实例本地的`WriteOnlyCollection`中的行。

```py
method update() → Update
```

生成一个`Update`，该语句将以此实例本地的`WriteOnlyCollection`的形式引用行。

```py
class sqlalchemy.orm.WriteOnlyMapped
```

表示“只写”关系的 ORM 映射属性类型。

`WriteOnlyMapped` 类型注释可以在带注释的声明性表映射中使用，以指示对于特定的`relationship()`应使用`lazy="write_only"`加载策略。

例如：

```py
class User(Base):
 __tablename__ = "user"
 id: Mapped[int] = mapped_column(primary_key=True)
 addresses: WriteOnlyMapped[Address] = relationship(
 cascade="all,delete-orphan"
 )
```

有关背景，请参阅仅写关系部分。

2.0 版中的新功能。

请参阅还有

仅写关系 - 完整背景

`DynamicMapped` - 包含遗留的`Query`支持

**类签名**

类`sqlalchemy.orm.WriteOnlyMapped` (`sqlalchemy.orm.base._MappedAnnotationBase`)  ## 动态关系加载器

遗留特性

“动态”延迟加载策略是现在在仅写关系部分中描述的“write_only”策略的遗留形式。

“动态”策略从相关集合中生成一个遗留的`Query`对象。然而，“动态”关系的一个主要缺点是，有几种情况下集合会完全迭代，其中一些是不明显的，只能通过细心的编程和逐案的测试来预防。因此，对于真正大型集合管理，应优先考虑`WriteOnlyCollection`。

动态加载器也与异步 I/O（asyncio）扩展不兼容。可以在一些限制下使用，如 Asyncio 动态指南中所示，但再次建议优先考虑与 asyncio 完全兼容的`WriteOnlyCollection`。

动态关系策略允许配置一个 `relationship()`，当在实例上访问时，将返回一个旧版的 `Query` 对象，而不是集合。然后可以进一步修改返回的 `Query` 对象，以便基于过滤条件迭代数据库集合。返回的 `Query` 对象是 `AppenderQuery` 的实例，它结合了 `Query` 的加载和迭代行为，以及 rudimentary 集合变异方法，如 `AppenderQuery.append()` 和 `AppenderQuery.remove()`。

可以使用带有类型注释的 Declarative 形式配置“动态”加载器策略，使用 `DynamicMapped` 注解类：

```py
from sqlalchemy.orm import DynamicMapped

class User(Base):
    __tablename__ = "user"

    id: Mapped[int] = mapped_column(primary_key=True)
    posts: DynamicMapped[Post] = relationship()
```

在上述情况下，单个 `User` 对象上的 `User.posts` 集合将返回 `AppenderQuery` 对象，它是 `Query` 的子类，还支持基本的集合变异操作：

```py
jack = session.get(User, id)

# filter Jack's blog posts
posts = jack.posts.filter(Post.headline == "this is a post")

# apply array slices
posts = jack.posts[5:20]
```

动态关系支持有限的写入操作，通过 `AppenderQuery.append()` 和 `AppenderQuery.remove()` 方法：

```py
oldpost = jack.posts.filter(Post.headline == "old post").one()
jack.posts.remove(oldpost)

jack.posts.append(Post("new post"))
```

由于动态关系的读取端总是查询数据库，对基础集合的更改直到数据刷新后才可见。然而，只要所使用的 `Session` 启用了“自动刷新”，这将在每次集合即将发出查询时自动发生。

### 动态关系加载器 - API

| 对象名称 | 描述 |
| --- | --- |
| AppenderQuery | 支持基本集合存储操作的动态查询。 |
| DynamicMapped | 代表“动态”关系的 ORM 映射属性类型。 |

```py
class sqlalchemy.orm.AppenderQuery
```

支持基本集合存储操作的动态查询。

`AppenderQuery` 上的方法包括 `Query` 的所有方法，以及用于集合持久化的附加方法。

**成员**

add(), add_all(), append(), count(), extend(), remove()

**类签名**

类 `sqlalchemy.orm.AppenderQuery` (`sqlalchemy.orm.dynamic.AppenderMixin`, `sqlalchemy.orm.Query`)

```py
method add(item: _T) → None
```

*继承自* `AppenderMixin.add()` *方法的* `AppenderMixin`

将项目添加到此 `AppenderQuery`。

给定的项目将在下一次 flush 时以父实例集合的形式持久化到数据库中。

此方法旨在帮助与 `WriteOnlyCollection` 集合类实现向前兼容。

版本 2.0 中的新功能。

```py
method add_all(iterator: Iterable[_T]) → None
```

*继承自* `AppenderMixin.add_all()` *方法的* `AppenderMixin`

将可迭代项目添加到此 `AppenderQuery`。

给定的项目将在下一次 flush 时以父实例集合的形式持久化到数据库中。

此方法旨在帮助与 `WriteOnlyCollection` 集合类实现向前兼容。

版本 2.0 中的新功能。

```py
method append(item: _T) → None
```

*继承自* `AppenderMixin.append()` *方法的* `AppenderMixin`

将项目追加到此 `AppenderQuery`。

给定的项目将在下一次 flush 时以父实例集合的形式持久化到数据库中。

```py
method count() → int
```

*继承自* `AppenderMixin.count()` *方法的* `AppenderMixin`

返回此 `Query` 形成的 SQL 返回的行数。

这将生成此查询的 SQL 如下：

```py
SELECT count(1) AS count_1 FROM (
    SELECT <rest of query follows...>
) AS anon_1
```

上述 SQL 返回一行，即计数函数的聚合值；然后 `Query.count()` 方法返回该单个整数值。

警告

需要注意的是，count() 返回的值**与此 Query 从诸如 .all() 方法返回的 ORM 对象数量不同**。当 `Query` 对象被要求返回完整实体时，将根据主键对条目进行去重，这意味着如果相同的主键值在结果中出现多次，则仅存在一个该主键的对象。这不适用于针对个别列的查询。

另请参阅

我的查询返回的对象数量与 query.count() 告诉我的不同 - 为什么？

若要对特定列进行精细化计数控制，跳过子查询的使用或以其他方式控制 FROM 子句，或使用其他聚合函数，请将 `expression.func` 表达式与 `Session.query()` 结合使用，例如：

```py
from sqlalchemy import func

# count User records, without
# using a subquery.
session.query(func.count(User.id))

# return count of user "id" grouped
# by "name"
session.query(func.count(User.id)).\
        group_by(User.name)

from sqlalchemy import distinct

# count distinct "name" values
session.query(func.count(distinct(User.name)))
```

另请参阅

2.0 迁移 - ORM 使用

```py
method extend(iterator: Iterable[_T]) → None
```

*继承自* `AppenderMixin.extend()` *方法的* `AppenderMixin`

将项目的可迭代对象添加到此`AppenderQuery`中。

给定的项目将在下次 flush 时以父实例的集合的形式持久化到数据库中。

```py
method remove(item: _T) → None
```

*继承自* `AppenderMixin.remove()` *方法的* `AppenderMixin`

从此 `AppenderQuery` 中移除一个项目。

下次 flush 时，给定的项目将从父实例的集合中移除。

```py
class sqlalchemy.orm.DynamicMapped
```

表示“动态”关系的 ORM 映射属性类型。

`DynamicMapped` 类型注释可以在 注释的声明性表 映射中使用，以指示应该为特定的 `relationship()` 使用 `lazy="dynamic"` 加载策略。

传统功能

“dynamic” 懒加载策略是现在称为“write_only”策略的传统形式，详情请参见 写入关系 部分。

例如：

```py
class User(Base):
    __tablename__ = "user"
    id: Mapped[int] = mapped_column(primary_key=True)
    addresses: DynamicMapped[Address] = relationship(
        cascade="all,delete-orphan"
    )
```

请参阅 动态关系加载器 部分以了解背景知识。

2.0 版中新增。

另请参阅

动态关系加载器 - 完整背景

`WriteOnlyMapped` - 完全 2.0 版本的风格

**类签名**

类 `sqlalchemy.orm.DynamicMapped` (`sqlalchemy.orm.base._MappedAnnotationBase`)  ## 设置 RaiseLoad

当属性通常会发出懒加载时，“raise”-loaded 关系将引发一个`InvalidRequestError`：

```py
class MyClass(Base):
    __tablename__ = "some_table"

    # ...

    children: Mapped[List[MyRelatedClass]] = relationship(lazy="raise")
```

在上面的示例中，如果 `children` 集合之前未填充，则对该集合进行属性访问将引发异常。这包括读取访问，但对于集合还将影响写入访问，因为集合不能在未加载的情况下进行突变。这样做的原因是确保应用程序在某个特定上下文中不会发出任何意外的惰性加载。与必须通过 SQL 日志来确定所有必要属性是否已急切加载相比，"raise" 策略将在访问时立即引发未加载的属性。raise 策略也可基于查询选项使用 `raiseload()` 加载器选项。

另请参阅

使用 raiseload 防止不必要的惰性加载

## 使用被动删除

SQLAlchemy 中集合管理的一个重要方面是，当删除引用集合的对象时，SQLAlchemy 需要考虑到位于此集合内部的对象。这些对象将需要与父对象解除关联，对于一对多集合，这意味着外键列将被设置为 NULL，或者根据 级联 设置，可能希望为这些行发出 DELETE。

工作单元 过程只考虑逐行处理对象，这意味着 DELETE 操作意味着集合内的所有行必须在刷新过程中完全加载到内存中。对于大型集合来说，这是不可行的，因此我们转而依靠数据库自身的能力，使用外键 ON DELETE 规则自动更新或删除行，指示工作单元无需实际加载这些行即可处理它们。可以通过配置 `relationship.passive_deletes` 在 `relationship()` 构造上来指示工作单元以此方式工作；正在使用的外键约束也必须正确配置。

有关完整的“被动删除”配置的进一步细节，请参阅章节 使用 ORM 关系与外键 ON DELETE 级联。

## 只写关系

**只写** 加载器策略是配置 `relationship()` 的主要手段，它将保持可写，但不会加载其内容到内存中。现代类型注释的 Declarative 形式中的只写 ORM 配置示例如下：

```py
>>> from decimal import Decimal
>>> from datetime import datetime

>>> from sqlalchemy import ForeignKey
>>> from sqlalchemy import func
>>> from sqlalchemy.orm import DeclarativeBase
>>> from sqlalchemy.orm import Mapped
>>> from sqlalchemy.orm import mapped_column
>>> from sqlalchemy.orm import relationship
>>> from sqlalchemy.orm import Session
>>> from sqlalchemy.orm import WriteOnlyMapped

>>> class Base(DeclarativeBase):
...     pass

>>> class Account(Base):
...     __tablename__ = "account"
...     id: Mapped[int] = mapped_column(primary_key=True)
...     identifier: Mapped[str]
...
...     account_transactions: WriteOnlyMapped["AccountTransaction"] = relationship(
...         cascade="all, delete-orphan",
...         passive_deletes=True,
...         order_by="AccountTransaction.timestamp",
...     )
...
...     def __repr__(self):
...         return f"Account(identifier={self.identifier!r})"

>>> class AccountTransaction(Base):
...     __tablename__ = "account_transaction"
...     id: Mapped[int] = mapped_column(primary_key=True)
...     account_id: Mapped[int] = mapped_column(
...         ForeignKey("account.id", ondelete="cascade")
...     )
...     description: Mapped[str]
...     amount: Mapped[Decimal]
...     timestamp: Mapped[datetime] = mapped_column(default=func.now())
...
...     def __repr__(self):
...         return (
...             f"AccountTransaction(amount={self.amount:.2f}, "
...             f"timestamp={self.timestamp.isoformat()!r})"
...         )
...
...     __mapper_args__ = {"eager_defaults": True}
```

上述的 `account_transactions` 关系不是使用普通的 `Mapped` 注解配置的，而是使用 `WriteOnlyMapped` 类型注解，在运行时将 `lazy="write_only"` 的加载策略分配给目标 `relationship()`。`WriteOnlyMapped` 注解是 `Mapped` 注解的替代形式，表示在对象实例上使用 `WriteOnlyCollection` 集合类型。

上述的 `relationship()` 配置还包括几个元素，用于指定在删除 `Account` 对象以及从 `account_transactions` 集合中移除 `AccountTransaction` 对象时要执行的操作。这些元素是：

+   `passive_deletes=True` - 允许工作单元在删除 `Account` 时不必加载集合；请参阅使用 ORM 关系进行级联删除。

+   `ondelete="cascade"` 配置在 `ForeignKey` 约束上。这也在 使用 ORM 关系进行级联删除 中详细说明。

+   `cascade="all, delete-orphan"` - 指示工作单元在从集合中删除 `AccountTransaction` 对象时删除它们。请参阅 delete-orphan 中的 Cascades 文档。

2.0 版本中的新功能：添加了“只写”关系加载器。

### 创建和持久化新的只写集合

只写集合允许**仅**对瞬态或待处理对象进行集合的直接赋值。通过我们上述的映射，这表示我们可以创建一个新的 `Account` 对象，并将一系列 `AccountTransaction` 对象添加到 `Session` 中。任何 Python 可迭代对象都可以作为对象的来源，下面我们使用了 Python 的 `list`：

```py
>>> new_account = Account(
...     identifier="account_01",
...     account_transactions=[
...         AccountTransaction(description="initial deposit", amount=Decimal("500.00")),
...         AccountTransaction(description="transfer", amount=Decimal("1000.00")),
...         AccountTransaction(description="withdrawal", amount=Decimal("-29.50")),
...     ],
... )

>>> with Session(engine) as session:
...     session.add(new_account)
...     session.commit()
BEGIN  (implicit)
INSERT  INTO  account  (identifier)  VALUES  (?)
[...]  ('account_01',)
INSERT  INTO  account_transaction  (account_id,  description,  amount,  timestamp)
VALUES  (?,  ?,  ?,  CURRENT_TIMESTAMP)  RETURNING  id,  timestamp
[...  (insertmanyvalues)  1/3  (ordered;  batch  not  supported)]  (1,  'initial deposit',  500.0)
INSERT  INTO  account_transaction  (account_id,  description,  amount,  timestamp)
VALUES  (?,  ?,  ?,  CURRENT_TIMESTAMP)  RETURNING  id,  timestamp
[insertmanyvalues  2/3  (ordered;  batch  not  supported)]  (1,  'transfer',  1000.0)
INSERT  INTO  account_transaction  (account_id,  description,  amount,  timestamp)
VALUES  (?,  ?,  ?,  CURRENT_TIMESTAMP)  RETURNING  id,  timestamp
[insertmanyvalues  3/3  (ordered;  batch  not  supported)]  (1,  'withdrawal',  -29.5)
COMMIT 
```

一旦对象已经持久化到数据库（即处于持久化或分离状态），集合就具有了扩展新项目以及删除单个项目的能力。但是，集合可能**不能再重新分配为完整的替换集合**，因为这样的操作需要将先前的集合完全加载到内存中，以便将旧条目与新条目进行协调：

```py
>>> new_account.account_transactions = [
...     AccountTransaction(description="some transaction", amount=Decimal("10.00"))
... ]
Traceback (most recent call last):
...
sqlalchemy.exc.InvalidRequestError: Collection "Account.account_transactions" does not
support implicit iteration; collection replacement operations can't be used
```

### 向现有集合添加新项目

对于持久化对象的只写集合，使用工作单元过程修改集合只能通过使用`WriteOnlyCollection.add()`、`WriteOnlyCollection.add_all()` 和 `WriteOnlyCollection.remove()` 方法进行：

```py
>>> from sqlalchemy import select
>>> session = Session(engine, expire_on_commit=False)
>>> existing_account = session.scalar(select(Account).filter_by(identifier="account_01"))
BEGIN  (implicit)
SELECT  account.id,  account.identifier
FROM  account
WHERE  account.identifier  =  ?
[...]  ('account_01',)
>>> existing_account.account_transactions.add_all(
...     [
...         AccountTransaction(description="paycheck", amount=Decimal("2000.00")),
...         AccountTransaction(description="rent", amount=Decimal("-800.00")),
...     ]
... )
>>> session.commit()
INSERT  INTO  account_transaction  (account_id,  description,  amount,  timestamp)
VALUES  (?,  ?,  ?,  CURRENT_TIMESTAMP)  RETURNING  id,  timestamp
[...  (insertmanyvalues)  1/2  (ordered;  batch  not  supported)]  (1,  'paycheck',  2000.0)
INSERT  INTO  account_transaction  (account_id,  description,  amount,  timestamp)
VALUES  (?,  ?,  ?,  CURRENT_TIMESTAMP)  RETURNING  id,  timestamp
[insertmanyvalues  2/2  (ordered;  batch  not  supported)]  (1,  'rent',  -800.0)
COMMIT 
```

上面添加的项目在 `Session` 内部保持在挂起队列中，直到下一次刷新，在此时它们被插入到数据库中，假设添加的对象之前是瞬态的。

### 查询项目

`WriteOnlyCollection` 在任何时候都不会存储对集合当前内容的引用，也不会具有直接发出 SELECT 到数据库以加载它们的行为；覆盖的假设是集合可能包含许多千万个行，并且绝不应作为任何其他操作的副作用完全加载到内存中。

相反，`WriteOnlyCollection` 包括 SQL 生成助手，如 `WriteOnlyCollection.select()`，它将生成一个预先配置了当前父行的正确 WHERE / FROM 条件的 `Select` 构造，然后可以进一步修改以选择所需的任何行范围，以及使用诸如服务器端游标等功能调用，以便以内存有效的方式迭代通过整个集合。

生成的语句如下所示。请注意，示例映射中包括 ORDER BY 标准，由 `relationship()` 的 `relationship.order_by` 参数指示；如果未配置该参数，则会省略此标准：

```py
>>> print(existing_account.account_transactions.select())
SELECT  account_transaction.id,  account_transaction.account_id,  account_transaction.description,
account_transaction.amount,  account_transaction.timestamp
FROM  account_transaction
WHERE  :param_1  =  account_transaction.account_id  ORDER  BY  account_transaction.timestamp 
```

我们可以使用这个 `Select` 结构以及 `Session` 来查询 `AccountTransaction` 对象，最简单的方法是使用 `Session.scalars()` 方法，该方法将直接返回一个 `Result`，其中包含 ORM 对象。通常情况下，虽然不是必需的，但`Select` 可能会进一步修改以限制返回的记录；在下面的示例中，添加了额外的 WHERE 条件，仅加载“借方”账户交易，并且使用“LIMIT 10”只检索前十行：

```py
>>> account_transactions = session.scalars(
...     existing_account.account_transactions.select()
...     .where(AccountTransaction.amount < 0)
...     .limit(10)
... ).all()
BEGIN  (implicit)
SELECT  account_transaction.id,  account_transaction.account_id,  account_transaction.description,
account_transaction.amount,  account_transaction.timestamp
FROM  account_transaction
WHERE  ?  =  account_transaction.account_id  AND  account_transaction.amount  <  ?
ORDER  BY  account_transaction.timestamp  LIMIT  ?  OFFSET  ?
[...]  (1,  0,  10,  0)
>>> print(account_transactions)
[AccountTransaction(amount=-29.50, timestamp='...'), AccountTransaction(amount=-800.00, timestamp='...')]
```

### 移除项目

在当前 `Session` 中加载到持久状态的个别项目可以使用`WriteOnlyCollection.remove()` 方法标记为从集合中移除。在操作继续时，刷新过程将隐式考虑对象已经是集合的一部分。下面的示例说明了删除单个 `AccountTransaction` 项目，根据 cascade 设置，会导致删除该行：

```py
>>> existing_transaction = account_transactions[0]
>>> existing_account.account_transactions.remove(existing_transaction)
>>> session.commit()
DELETE  FROM  account_transaction  WHERE  account_transaction.id  =  ?
[...]  (3,)
COMMIT 
```

与任何 ORM 映射的集合一样，对象移除可以根据`relationship()` 的 delete-orphan 配置，要么取消与集合的关联，同时保留对象在数据库中的存在，要么根据配置发出对其行的 DELETE 请求。

不删除的集合移除涉及将外键列设置为 NULL（对于一对多关系）或删除相应的关联行（对于多对多关系）。

### 新项目的批量插入

`WriteOnlyCollection`可以生成诸如`Insert`对象之类的 DML 构造，这些构造可以在 ORM 上下文中用于生成批量插入行为。有关 ORM 批量插入的概述，请参阅 ORM 批量插入语句部分。

#### 一对多集合

对于**普通的一对多集合**，`WriteOnlyCollection.insert()`方法将产生一个预先建立了与父对象对应的 VALUES 条件的`Insert`构造。由于这个 VALUES 条件完全针对相关表，所以该语句可以用于插入新行，这些新行将同时成为相关集合中的新记录：

```py
>>> session.execute(
...     existing_account.account_transactions.insert(),
...     [
...         {"description": "transaction 1", "amount": Decimal("47.50")},
...         {"description": "transaction 2", "amount": Decimal("-501.25")},
...         {"description": "transaction 3", "amount": Decimal("1800.00")},
...         {"description": "transaction 4", "amount": Decimal("-300.00")},
...     ],
... )
BEGIN  (implicit)
INSERT  INTO  account_transaction  (account_id,  description,  amount,  timestamp)  VALUES  (?,  ?,  ?,  CURRENT_TIMESTAMP)
[...]  [(1,  'transaction 1',  47.5),  (1,  'transaction 2',  -501.25),  (1,  'transaction 3',  1800.0),  (1,  'transaction 4',  -300.0)]
<...>
>>> session.commit()
COMMIT
```

另请参阅

ORM 批量插入语句 - 在 ORM 查询指南中

一对多 - 在基本关系模式中

#### 多对多集合

对于**多对多集合**，两个类之间的关系涉及第三个表，该表使用`relationship.secondary`参数配置`relationship`。要使用`WriteOnlyCollection`批量插入此类型的集合中的行，可以先单独批量插入新记录，然后使用 RETURNING 检索，然后将这些记录传递给`WriteOnlyCollection.add_all()`方法，工作单元过程将继续将其作为集合的一部分持久化。

假设一个类`BankAudit`通过多对多表引用了许多`AccountTransaction`记录：

```py
>>> from sqlalchemy import Table, Column
>>> audit_to_transaction = Table(
...     "audit_transaction",
...     Base.metadata,
...     Column("audit_id", ForeignKey("audit.id", ondelete="CASCADE"), primary_key=True),
...     Column(
...         "transaction_id",
...         ForeignKey("account_transaction.id", ondelete="CASCADE"),
...         primary_key=True,
...     ),
... )
>>> class BankAudit(Base):
...     __tablename__ = "audit"
...     id: Mapped[int] = mapped_column(primary_key=True)
...     account_transactions: WriteOnlyMapped["AccountTransaction"] = relationship(
...         secondary=audit_to_transaction, passive_deletes=True
...     )
```

为了说明这两个操作，我们使用批量插入添加了更多的`AccountTransaction`对象，我们通过在批量 INSERT 语句中添加`returning(AccountTransaction)`来检索它们 (注意我们也可以使用现有的`AccountTransaction`对象)：

```py
>>> new_transactions = session.scalars(
...     existing_account.account_transactions.insert().returning(AccountTransaction),
...     [
...         {"description": "odd trans 1", "amount": Decimal("50000.00")},
...         {"description": "odd trans 2", "amount": Decimal("25000.00")},
...         {"description": "odd trans 3", "amount": Decimal("45.00")},
...     ],
... ).all()
BEGIN  (implicit)
INSERT  INTO  account_transaction  (account_id,  description,  amount,  timestamp)  VALUES
(?,  ?,  ?,  CURRENT_TIMESTAMP),  (?,  ?,  ?,  CURRENT_TIMESTAMP),  (?,  ?,  ?,  CURRENT_TIMESTAMP)
RETURNING  id,  account_id,  description,  amount,  timestamp
[...]  (1,  'odd trans 1',  50000.0,  1,  'odd trans 2',  25000.0,  1,  'odd trans 3',  45.0) 
```

有了一系列准备好的`AccountTransaction`对象，可以使用`WriteOnlyCollection.add_all()`方法一次性将许多行与新的`BankAudit`对象关联起来：

```py
>>> bank_audit = BankAudit()
>>> session.add(bank_audit)
>>> bank_audit.account_transactions.add_all(new_transactions)
>>> session.commit()
INSERT  INTO  audit  DEFAULT  VALUES
[...]  ()
INSERT  INTO  audit_transaction  (audit_id,  transaction_id)  VALUES  (?,  ?)
[...]  [(1,  10),  (1,  11),  (1,  12)]
COMMIT 
```

另请参阅

ORM 批量插入语句 - 在 ORM 查询指南中

多对多 - 位于基本关系模式

### Items 的批量 UPDATE 和 DELETE

与`WriteOnlyCollection`类似，它可以生成带有预先建立的 WHERE 条件的`Select`结构，也可以生成带有相同 WHERE 条件的`Update`和`Delete`结构，以允许针对大型集合中的元素进行基于条件的 UPDATE 和 DELETE 语句。

#### 一对多集合

与 INSERT 一样，这个特性对于**一对多集合**来说最为直接。

在下面的示例中，`WriteOnlyCollection.update()`方法用于生成一条 UPDATE 语句，针对集合中的元素发出，定位“amount”等于`-800`的行，并将`200`的数量添加到它们：

```py
>>> session.execute(
...     existing_account.account_transactions.update()
...     .values(amount=AccountTransaction.amount + 200)
...     .where(AccountTransaction.amount == -800),
... )
BEGIN  (implicit)
UPDATE  account_transaction  SET  amount=(account_transaction.amount  +  ?)
WHERE  ?  =  account_transaction.account_id  AND  account_transaction.amount  =  ?
[...]  (200,  1,  -800)
<...>
```

类似地，`WriteOnlyCollection.delete()`将生成一个 DELETE 语句，以相同的方式调用：

```py
>>> session.execute(
...     existing_account.account_transactions.delete().where(
...         AccountTransaction.amount.between(0, 30)
...     ),
... )
DELETE  FROM  account_transaction  WHERE  ?  =  account_transaction.account_id
AND  account_transaction.amount  BETWEEN  ?  AND  ?  RETURNING  id
[...]  (1,  0,  30)
<...> 
```

#### 多对多集合

提示

这里涉及到稍微高级的多表 UPDATE 表达式技巧。

对于**多对多集合**的批量 UPDATE 和 DELETE，为了使 UPDATE 或 DELETE 语句与父对象的主键相关联，必须显式地将关联表包含在 UPDATE/DELETE 语句中，这要求后端要么包括对非标准 SQL 语法的支持，要么在构建 UPDATE 或 DELETE 语句时需要额外的显式步骤。

对于支持多表版本 UPDATE 的后端，`WriteOnlyCollection.update()`方法应该可以直接用于多对多集合，就像下面的示例中针对多对多`BankAudit.account_transactions`集合中的`AccountTransaction`对象发出 UPDATE 一样：

```py
>>> session.execute(
...     bank_audit.account_transactions.update().values(
...         description=AccountTransaction.description + " (audited)"
...     )
... )
UPDATE  account_transaction  SET  description=(account_transaction.description  ||  ?)
FROM  audit_transaction  WHERE  ?  =  audit_transaction.audit_id
AND  account_transaction.id  =  audit_transaction.transaction_id  RETURNING  id
[...]  (' (audited)',  1)
<...>
```

上述语句自动使用了“UPDATE..FROM”语法，该语法由 SQLite 和其他数据库支持，以在 WHERE 子句中命名额外的`audit_transaction`表。

要更新或删除多对多集合，其中多表语法不可用，多对多条件可以移动到 SELECT 语句中，例如可以与 IN 组合以匹配行。 在这里，`WriteOnlyCollection`仍然对我们有帮助，因为我们使用`WriteOnlyCollection.select()`方法为我们生成此 SELECT，利用`Select.with_only_columns()`方法生成标量子查询：

```py
>>> from sqlalchemy import update
>>> subq = bank_audit.account_transactions.select().with_only_columns(AccountTransaction.id)
>>> session.execute(
...     update(AccountTransaction)
...     .values(description=AccountTransaction.description + " (audited)")
...     .where(AccountTransaction.id.in_(subq))
... )
UPDATE  account_transaction  SET  description=(account_transaction.description  ||  ?)
WHERE  account_transaction.id  IN  (SELECT  account_transaction.id
FROM  audit_transaction
WHERE  ?  =  audit_transaction.audit_id  AND  account_transaction.id  =  audit_transaction.transaction_id)
RETURNING  id
[...]  (' (audited)',  1)
<...> 
```

### 写入仅集合 - API 文档

| 对象名称 | 描述 |
| --- | --- |
| WriteOnlyCollection | 写入仅集合，可将更改同步到属性事件系统中。 |
| WriteOnlyMapped | 代表“仅写”关系的 ORM 映射属性类型。 |

```py
class sqlalchemy.orm.WriteOnlyCollection
```

写入仅集合，可将更改同步到属性事件系统中。

`WriteOnlyCollection`在映射中使用`"write_only"`延迟加载策略与`relationship()`一起使用。有关此配置的背景，请参阅仅写关系。

版本 2.0 中的新功能。

请参阅

仅写关系

**成员**

add(), add_all(), delete(), insert(), remove(), select(), update()

**类签名**

class `sqlalchemy.orm.WriteOnlyCollection` (`sqlalchemy.orm.writeonly.AbstractCollectionWriter`)

```py
method add(item: _T) → None
```

向此`WriteOnlyCollection`添加一个项目。

下一次刷新时，给定的项目将以父实例的集合的形式持久化到数据库中。

```py
method add_all(iterator: Iterable[_T]) → None
```

向此`WriteOnlyCollection`添加一组项目。

下一次刷新时，给定的项目将以父实例的集合的形式持久化到数据库中。

```py
method delete() → Delete
```

生成一个 `Delete`，将引用以这个实例本地的`WriteOnlyCollection`。

```py
method insert() → Insert
```

对于一对多集合，产生一个 `Insert`，该插入将以此实例本地 `WriteOnlyCollection` 为条件插入新的行。

此构造仅支持不包含 `relationship.secondary` 参数的 `Relationship`。对于引用到多对多表的关系，请使用普通的批量插入技术来产生新对象，然后使用 `AbstractCollectionWriter.add_all()` 将其与集合关联起来。

```py
method remove(item: _T) → None
```

从此 `WriteOnlyCollection` 中移除一个项目。

给定的项目将在下次刷新时从父实例的集合中移除。

```py
method select() → Select[Tuple[_T]]
```

产生一个 `Select` 构造，表示此实例本地 `WriteOnlyCollection` 中的行。

```py
method update() → Update
```

产生一个 `Update`，该更新将参考以此实例本地为条件的行的 `WriteOnlyCollection`。

```py
class sqlalchemy.orm.WriteOnlyMapped
```

表示“只写”关系的 ORM 映射属性类型。

`WriteOnlyMapped` 类型注释可以在 带注释的声明性表 映射中使用，以指示特定的 `relationship()` 应使用 `lazy="write_only"` 加载策略。

例如：

```py
class User(Base):
 __tablename__ = "user"
 id: Mapped[int] = mapped_column(primary_key=True)
 addresses: WriteOnlyMapped[Address] = relationship(
 cascade="all,delete-orphan"
 )
```

参见章节 只写关系 了解背景。

新版本 2.0 中新增。

另请参阅

只写关系 - 完整的背景

`DynamicMapped` - 包含旧的 `Query` 支持

**类签名**

类 `sqlalchemy.orm.WriteOnlyMapped` (`sqlalchemy.orm.base._MappedAnnotationBase`)

### 创建和持久化新的只写集合

仅写集合允许直接将集合整体分配为**仅**用于瞬态或待处理对象。根据我们上面的映射，这表示我们可以创建一个新的 `Account` 对象，其中包含要添加到`Session`中的一系列 `AccountTransaction` 对象。任何 Python 可迭代对象都可以用作要开始的对象的源，下面我们使用 Python `list`：

```py
>>> new_account = Account(
...     identifier="account_01",
...     account_transactions=[
...         AccountTransaction(description="initial deposit", amount=Decimal("500.00")),
...         AccountTransaction(description="transfer", amount=Decimal("1000.00")),
...         AccountTransaction(description="withdrawal", amount=Decimal("-29.50")),
...     ],
... )

>>> with Session(engine) as session:
...     session.add(new_account)
...     session.commit()
BEGIN  (implicit)
INSERT  INTO  account  (identifier)  VALUES  (?)
[...]  ('account_01',)
INSERT  INTO  account_transaction  (account_id,  description,  amount,  timestamp)
VALUES  (?,  ?,  ?,  CURRENT_TIMESTAMP)  RETURNING  id,  timestamp
[...  (insertmanyvalues)  1/3  (ordered;  batch  not  supported)]  (1,  'initial deposit',  500.0)
INSERT  INTO  account_transaction  (account_id,  description,  amount,  timestamp)
VALUES  (?,  ?,  ?,  CURRENT_TIMESTAMP)  RETURNING  id,  timestamp
[insertmanyvalues  2/3  (ordered;  batch  not  supported)]  (1,  'transfer',  1000.0)
INSERT  INTO  account_transaction  (account_id,  description,  amount,  timestamp)
VALUES  (?,  ?,  ?,  CURRENT_TIMESTAMP)  RETURNING  id,  timestamp
[insertmanyvalues  3/3  (ordered;  batch  not  supported)]  (1,  'withdrawal',  -29.5)
COMMIT 
```

一旦对象被数据库持久化（即处于持久化或分离状态），集合就有能力扩展新项目以及个别项目的能力被移除。但是，集合可能**不再被重新分配为完整替换集合**，因为这样的操作要求以前的集合完全加载到内存中，以便将旧条目与新条目进行对比：

```py
>>> new_account.account_transactions = [
...     AccountTransaction(description="some transaction", amount=Decimal("10.00"))
... ]
Traceback (most recent call last):
...
sqlalchemy.exc.InvalidRequestError: Collection "Account.account_transactions" does not
support implicit iteration; collection replacement operations can't be used
```

### 向现有集合添加新项目

对于持久对象的仅写集合，使用 unit of work 过程对集合进行修改只能使用`WriteOnlyCollection.add()`、`WriteOnlyCollection.add_all()`和`WriteOnlyCollection.remove()`方法：

```py
>>> from sqlalchemy import select
>>> session = Session(engine, expire_on_commit=False)
>>> existing_account = session.scalar(select(Account).filter_by(identifier="account_01"))
BEGIN  (implicit)
SELECT  account.id,  account.identifier
FROM  account
WHERE  account.identifier  =  ?
[...]  ('account_01',)
>>> existing_account.account_transactions.add_all(
...     [
...         AccountTransaction(description="paycheck", amount=Decimal("2000.00")),
...         AccountTransaction(description="rent", amount=Decimal("-800.00")),
...     ]
... )
>>> session.commit()
INSERT  INTO  account_transaction  (account_id,  description,  amount,  timestamp)
VALUES  (?,  ?,  ?,  CURRENT_TIMESTAMP)  RETURNING  id,  timestamp
[...  (insertmanyvalues)  1/2  (ordered;  batch  not  supported)]  (1,  'paycheck',  2000.0)
INSERT  INTO  account_transaction  (account_id,  description,  amount,  timestamp)
VALUES  (?,  ?,  ?,  CURRENT_TIMESTAMP)  RETURNING  id,  timestamp
[insertmanyvalues  2/2  (ordered;  batch  not  supported)]  (1,  'rent',  -800.0)
COMMIT 
```

上述添加的项目在`Session`中被保留在待处理队列中，直到下一个 flush，此时它们被插入到数据库中，假设添加的对象之前是瞬态的。

### 查询项目

`WriteOnlyCollection`不会在任何时候存储对集合当前内容的引用，也不会有任何直接发出 SELECT 到数据库以加载它们的行为；其覆盖的假设是集合可能包含许多千万个或数百万个行，并且不应作为任何其他操作的副作用完全加载到内存中。

相反，`WriteOnlyCollection` 包括生成 SQL 的辅助工具，如 `WriteOnlyCollection.select()`，它将生成一个预先配置了正确的 WHERE / FROM 条件的 `Select` 构造，然后可以进一步修改以选择所需的任何行范围，还可以使用 服务器端游标 进行调用，以便以内存有效的方式遍历整个集合。

下面说明了生成的语句。请注意，示例映射中的 `relationship.order_by` 参数指示了 ORDER BY 条件；如果未配置该参数，则该条件将被省略：

```py
>>> print(existing_account.account_transactions.select())
SELECT  account_transaction.id,  account_transaction.account_id,  account_transaction.description,
account_transaction.amount,  account_transaction.timestamp
FROM  account_transaction
WHERE  :param_1  =  account_transaction.account_id  ORDER  BY  account_transaction.timestamp 
```

我们可以使用此 `Select` 构造以及 `Session` 来查询 `AccountTransaction` 对象，最简单的方法是使用 `Session.scalars()` 方法，该方法将返回一个直接产生 ORM 对象的 `Result`。通常情况下，但不是必需的，会进一步修改 `Select` 以限制返回的记录；在下面的示例中，添加了额外的 WHERE 条件以仅加载 “借方” 账户交易，并添加了 “LIMIT 10” 以仅检索前十行：

```py
>>> account_transactions = session.scalars(
...     existing_account.account_transactions.select()
...     .where(AccountTransaction.amount < 0)
...     .limit(10)
... ).all()
BEGIN  (implicit)
SELECT  account_transaction.id,  account_transaction.account_id,  account_transaction.description,
account_transaction.amount,  account_transaction.timestamp
FROM  account_transaction
WHERE  ?  =  account_transaction.account_id  AND  account_transaction.amount  <  ?
ORDER  BY  account_transaction.timestamp  LIMIT  ?  OFFSET  ?
[...]  (1,  0,  10,  0)
>>> print(account_transactions)
[AccountTransaction(amount=-29.50, timestamp='...'), AccountTransaction(amount=-800.00, timestamp='...')]
```

### 删除项目

在当前 `Session` 中针对持久状态的单个加载项目可以使用 `WriteOnlyCollection.remove()` 方法标记为从集合中删除。当操作进行时，刷新过程将隐式考虑对象已经是集合的一部分。下面的示例说明了如何删除单个 `AccountTransaction` 项目，根据 级联 设置，这将导致删除该行：

```py
>>> existing_transaction = account_transactions[0]
>>> existing_account.account_transactions.remove(existing_transaction)
>>> session.commit()
DELETE  FROM  account_transaction  WHERE  account_transaction.id  =  ?
[...]  (3,)
COMMIT 
```

与任何 ORM 映射的集合一样，对象的移除可以选择将对象与集合解除关联，同时保留对象在数据库中，或者可以基于 `relationship()` 的 delete-orphan 配置发出其行的 DELETE。

在不删除的情况下移除集合涉及将外键列设置为 NULL（对于 一对多 关系）或删除相应的关联行（对于 多对多 关系）。

### 批量插入新项目

`WriteOnlyCollection` 可以生成诸如 `Insert` 对象之类的 DML 构造，这些构造可以在 ORM 上下文中用于产生批量插入行为。参见 ORM 批量插入语句 章节了解 ORM 批量插入的概述。

#### 一对多集合

仅适用于**常规的一对多集合**，`WriteOnlyCollection.insert()` 方法将产生一个预先设定了与父对象相对应的 VALUES 条件的 `Insert` 构造。由于这个 VALUES 条件完全针对相关表，该语句可用于插入新行，这些新行将同时成为相关集合中的新记录：

```py
>>> session.execute(
...     existing_account.account_transactions.insert(),
...     [
...         {"description": "transaction 1", "amount": Decimal("47.50")},
...         {"description": "transaction 2", "amount": Decimal("-501.25")},
...         {"description": "transaction 3", "amount": Decimal("1800.00")},
...         {"description": "transaction 4", "amount": Decimal("-300.00")},
...     ],
... )
BEGIN  (implicit)
INSERT  INTO  account_transaction  (account_id,  description,  amount,  timestamp)  VALUES  (?,  ?,  ?,  CURRENT_TIMESTAMP)
[...]  [(1,  'transaction 1',  47.5),  (1,  'transaction 2',  -501.25),  (1,  'transaction 3',  1800.0),  (1,  'transaction 4',  -300.0)]
<...>
>>> session.commit()
COMMIT
```

另请参阅

ORM 批量插入语句 - 在 ORM 查询指南 中

一对多 - 在 基本关系模式 中

#### 多对多集合

对于**多对多集合**，两个类之间的关系涉及使用 `relationship.secondary` 参数配置的第三个表的情况。要使用 `WriteOnlyCollection` 批量插入此类型的集合中的行，新记录可能首先单独进行批量插入，然后使用 RETURNING 检索，然后将这些记录传递给 `WriteOnlyCollection.add_all()` 方法，在这个过程中，工作单元将会将它们作为集合的一部分持久化。

假设一个类`BankAudit`使用多对多表引用了许多`AccountTransaction`记录：

```py
>>> from sqlalchemy import Table, Column
>>> audit_to_transaction = Table(
...     "audit_transaction",
...     Base.metadata,
...     Column("audit_id", ForeignKey("audit.id", ondelete="CASCADE"), primary_key=True),
...     Column(
...         "transaction_id",
...         ForeignKey("account_transaction.id", ondelete="CASCADE"),
...         primary_key=True,
...     ),
... )
>>> class BankAudit(Base):
...     __tablename__ = "audit"
...     id: Mapped[int] = mapped_column(primary_key=True)
...     account_transactions: WriteOnlyMapped["AccountTransaction"] = relationship(
...         secondary=audit_to_transaction, passive_deletes=True
...     )
```

为了说明这两个操作，我们使用批量插入添加更多`AccountTransaction`对象，我们通过将`returning(AccountTransaction)`添加到批量 INSERT 语句中使用 RETURNING 检索（注意我们也可以使用现有的`AccountTransaction`对象）：

```py
>>> new_transactions = session.scalars(
...     existing_account.account_transactions.insert().returning(AccountTransaction),
...     [
...         {"description": "odd trans 1", "amount": Decimal("50000.00")},
...         {"description": "odd trans 2", "amount": Decimal("25000.00")},
...         {"description": "odd trans 3", "amount": Decimal("45.00")},
...     ],
... ).all()
BEGIN  (implicit)
INSERT  INTO  account_transaction  (account_id,  description,  amount,  timestamp)  VALUES
(?,  ?,  ?,  CURRENT_TIMESTAMP),  (?,  ?,  ?,  CURRENT_TIMESTAMP),  (?,  ?,  ?,  CURRENT_TIMESTAMP)
RETURNING  id,  account_id,  description,  amount,  timestamp
[...]  (1,  'odd trans 1',  50000.0,  1,  'odd trans 2',  25000.0,  1,  'odd trans 3',  45.0) 
```

有一组准备好的`AccountTransaction`对象列表，使用`WriteOnlyCollection.add_all()`方法可以一次性将许多行与新的`BankAudit`对象关联起来：

```py
>>> bank_audit = BankAudit()
>>> session.add(bank_audit)
>>> bank_audit.account_transactions.add_all(new_transactions)
>>> session.commit()
INSERT  INTO  audit  DEFAULT  VALUES
[...]  ()
INSERT  INTO  audit_transaction  (audit_id,  transaction_id)  VALUES  (?,  ?)
[...]  [(1,  10),  (1,  11),  (1,  12)]
COMMIT 
```

另请参阅

ORM 批量 INSERT 语句 - 在 ORM 查询指南中

多对多 - 在基本关系模式中

#### 一对多集合

对于**普通的一对多集合**，`WriteOnlyCollection.insert()`方法将生成一个与父对象对应的 VALUES 条件预设的`Insert`构造。由于这个 VALUES 条件完全针对相关表，该语句可用于插入新行，同时这些新行也将成为相关集合中的新记录：

```py
>>> session.execute(
...     existing_account.account_transactions.insert(),
...     [
...         {"description": "transaction 1", "amount": Decimal("47.50")},
...         {"description": "transaction 2", "amount": Decimal("-501.25")},
...         {"description": "transaction 3", "amount": Decimal("1800.00")},
...         {"description": "transaction 4", "amount": Decimal("-300.00")},
...     ],
... )
BEGIN  (implicit)
INSERT  INTO  account_transaction  (account_id,  description,  amount,  timestamp)  VALUES  (?,  ?,  ?,  CURRENT_TIMESTAMP)
[...]  [(1,  'transaction 1',  47.5),  (1,  'transaction 2',  -501.25),  (1,  'transaction 3',  1800.0),  (1,  'transaction 4',  -300.0)]
<...>
>>> session.commit()
COMMIT
```

另请参阅

ORM 批量 INSERT 语句 - 在 ORM 查询指南中

一对多 - 在基本关系模式中

#### 多对多集合

对于**多对多集合**，两个类之间的关系涉及使用`relationship`的`relationship.secondary`参数配置的第三个表。要使用`WriteOnlyCollection`批量插入此类型的集合中的行，新记录可能首先被单独批量插入，然后使用 RETURNING 检索，并将这些记录传递给`WriteOnlyCollection.add_all()`方法，其中工作单元过程将继续将它们持久化为集合的一部分。

假设一个类`BankAudit`使用多对多表引用了许多`AccountTransaction`记录：

```py
>>> from sqlalchemy import Table, Column
>>> audit_to_transaction = Table(
...     "audit_transaction",
...     Base.metadata,
...     Column("audit_id", ForeignKey("audit.id", ondelete="CASCADE"), primary_key=True),
...     Column(
...         "transaction_id",
...         ForeignKey("account_transaction.id", ondelete="CASCADE"),
...         primary_key=True,
...     ),
... )
>>> class BankAudit(Base):
...     __tablename__ = "audit"
...     id: Mapped[int] = mapped_column(primary_key=True)
...     account_transactions: WriteOnlyMapped["AccountTransaction"] = relationship(
...         secondary=audit_to_transaction, passive_deletes=True
...     )
```

为了说明这两个操作，我们使用批量插入添加更多的`AccountTransaction`对象，我们通过在批量插入语句中添加`returning(AccountTransaction)`来检索这些对象（请注意，我们也可以轻松地使用现有的`AccountTransaction`对象）：

```py
>>> new_transactions = session.scalars(
...     existing_account.account_transactions.insert().returning(AccountTransaction),
...     [
...         {"description": "odd trans 1", "amount": Decimal("50000.00")},
...         {"description": "odd trans 2", "amount": Decimal("25000.00")},
...         {"description": "odd trans 3", "amount": Decimal("45.00")},
...     ],
... ).all()
BEGIN  (implicit)
INSERT  INTO  account_transaction  (account_id,  description,  amount,  timestamp)  VALUES
(?,  ?,  ?,  CURRENT_TIMESTAMP),  (?,  ?,  ?,  CURRENT_TIMESTAMP),  (?,  ?,  ?,  CURRENT_TIMESTAMP)
RETURNING  id,  account_id,  description,  amount,  timestamp
[...]  (1,  'odd trans 1',  50000.0,  1,  'odd trans 2',  25000.0,  1,  'odd trans 3',  45.0) 
```

准备好一个`AccountTransaction`对象列表后，使用`WriteOnlyCollection.add_all()`方法一次将许多行与新的`BankAudit`对象关联起来：

```py
>>> bank_audit = BankAudit()
>>> session.add(bank_audit)
>>> bank_audit.account_transactions.add_all(new_transactions)
>>> session.commit()
INSERT  INTO  audit  DEFAULT  VALUES
[...]  ()
INSERT  INTO  audit_transaction  (audit_id,  transaction_id)  VALUES  (?,  ?)
[...]  [(1,  10),  (1,  11),  (1,  12)]
COMMIT 
```

请参见

ORM 批量插入语句 - 在 ORM 查询指南

多对多 - 在基本关系模式

### 批量更新和删除项目

类似于`WriteOnlyCollection`可以生成带有预先建立 WHERE 条件的`Select`构造，它也可以生成具有相同 WHERE 条件的`Update`和`Delete`构造，以允许针对大型集合中的元素的基于条件的 UPDATE 和 DELETE 语句。

#### 一对多集合

和插入一样，这个特性在**一对多集合**中最为直接。

在下面的例子中，使用`WriteOnlyCollection.update()`方法生成一个 UPDATE 语句，该语句针对集合中的元素发出，定位“amount”等于`-800`的行，并向它们添加`200`的金额：

```py
>>> session.execute(
...     existing_account.account_transactions.update()
...     .values(amount=AccountTransaction.amount + 200)
...     .where(AccountTransaction.amount == -800),
... )
BEGIN  (implicit)
UPDATE  account_transaction  SET  amount=(account_transaction.amount  +  ?)
WHERE  ?  =  account_transaction.account_id  AND  account_transaction.amount  =  ?
[...]  (200,  1,  -800)
<...>
```

类似地，`WriteOnlyCollection.delete()`将生成一个 DELETE 语句，以相同的方式调用：

```py
>>> session.execute(
...     existing_account.account_transactions.delete().where(
...         AccountTransaction.amount.between(0, 30)
...     ),
... )
DELETE  FROM  account_transaction  WHERE  ?  =  account_transaction.account_id
AND  account_transaction.amount  BETWEEN  ?  AND  ?  RETURNING  id
[...]  (1,  0,  30)
<...> 
```

#### 多对多集合

提示

这里涉及到多表 UPDATE 表达式，这略微更加复杂。

对于批量更新和删除**多对多集合**，为了使 UPDATE 或 DELETE 语句与父对象的主键相关联，关联表必须明确地包含在 UPDATE/DELETE 语句中，这要求后端包含对非标准 SQL 语法的支持，或者在构建 UPDATE 或 DELETE 语句时进行额外的明确步骤。

对于支持 UPDATE 的多表版本的后端，`WriteOnlyCollection.update()` 方法应该在多对多集合中工作而无需额外步骤，就像下面的例子中，在`BankAudit.account_transactions`集合的多对多对象`AccountTransaction`对象上发出 UPDATE 一样：

```py
>>> session.execute(
...     bank_audit.account_transactions.update().values(
...         description=AccountTransaction.description + " (audited)"
...     )
... )
UPDATE  account_transaction  SET  description=(account_transaction.description  ||  ?)
FROM  audit_transaction  WHERE  ?  =  audit_transaction.audit_id
AND  account_transaction.id  =  audit_transaction.transaction_id  RETURNING  id
[...]  (' (audited)',  1)
<...>
```

上面的语句自动使用了“UPDATE..FROM”语法，由 SQLite 和其他后端支持，在 WHERE 子句中命名附加的`audit_transaction`表。

要更新或删除多对多集合，其中多表语法不可用，多对多条件可能会移到 SELECT 中，例如可以与 IN 组合以匹配行。`WriteOnlyCollection` 仍然在这里帮助我们，因为我们使用`WriteOnlyCollection.select()` 方法为我们生成这个 SELECT，利用`Select.with_only_columns()` 方法产生一个标量子查询：

```py
>>> from sqlalchemy import update
>>> subq = bank_audit.account_transactions.select().with_only_columns(AccountTransaction.id)
>>> session.execute(
...     update(AccountTransaction)
...     .values(description=AccountTransaction.description + " (audited)")
...     .where(AccountTransaction.id.in_(subq))
... )
UPDATE  account_transaction  SET  description=(account_transaction.description  ||  ?)
WHERE  account_transaction.id  IN  (SELECT  account_transaction.id
FROM  audit_transaction
WHERE  ?  =  audit_transaction.audit_id  AND  account_transaction.id  =  audit_transaction.transaction_id)
RETURNING  id
[...]  (' (audited)',  1)
<...> 
```

#### 一对多集合

就像在 INSERT 中一样，这个特性在**一对多集合**中最直接。

在下面的例子中，`WriteOnlyCollection.update()` 方法用于生成一个 UPDATE 语句，针对集合中的元素发出，定位“amount”等于`-800`的行，并将`200`的数量添加到它们中：

```py
>>> session.execute(
...     existing_account.account_transactions.update()
...     .values(amount=AccountTransaction.amount + 200)
...     .where(AccountTransaction.amount == -800),
... )
BEGIN  (implicit)
UPDATE  account_transaction  SET  amount=(account_transaction.amount  +  ?)
WHERE  ?  =  account_transaction.account_id  AND  account_transaction.amount  =  ?
[...]  (200,  1,  -800)
<...>
```

类似地，`WriteOnlyCollection.delete()` 将产生一个 DELETE 语句，以相同的方式调用：

```py
>>> session.execute(
...     existing_account.account_transactions.delete().where(
...         AccountTransaction.amount.between(0, 30)
...     ),
... )
DELETE  FROM  account_transaction  WHERE  ?  =  account_transaction.account_id
AND  account_transaction.amount  BETWEEN  ?  AND  ?  RETURNING  id
[...]  (1,  0,  30)
<...> 
```

#### 多对多集合

小贴士

这里的技术涉及多表 UPDATE 表达式，稍微更高级一些。

对于**多对多集合**的批量 UPDATE 和 DELETE，为了使 UPDATE 或 DELETE 语句与父对象的主键相关联，关联表必须明确地成为 UPDATE/DELETE 语句的一部分，这要求后端包含支持非标准 SQL 语法的支持，或者在构造 UPDATE 或 DELETE 语句时需要额外的显式步骤。

对于支持 UPDATE 的多表版本的后端，`WriteOnlyCollection.update()` 方法应该在多对多集合中工作而无需额外步骤，就像下面的例子中，在`BankAudit.account_transactions`集合的多对多对象`AccountTransaction`对象上发出 UPDATE 一样：

```py
>>> session.execute(
...     bank_audit.account_transactions.update().values(
...         description=AccountTransaction.description + " (audited)"
...     )
... )
UPDATE  account_transaction  SET  description=(account_transaction.description  ||  ?)
FROM  audit_transaction  WHERE  ?  =  audit_transaction.audit_id
AND  account_transaction.id  =  audit_transaction.transaction_id  RETURNING  id
[...]  (' (audited)',  1)
<...>
```

上述语句自动使用“UPDATE..FROM”语法，在 SQLite 和其他支持的数据库中，在 WHERE 子句中命名附加的`audit_transaction`表。

要更新或删除多对多集合，其中多表语法不可用，多对多条件可以移动到 SELECT 中，例如可以与 IN 结合使用来匹配行。在这里，`WriteOnlyCollection` 仍然对我们有帮助，因为我们使用`WriteOnlyCollection.select()`方法为我们生成此 SELECT，利用`Select.with_only_columns()`方法生成标量子查询：

```py
>>> from sqlalchemy import update
>>> subq = bank_audit.account_transactions.select().with_only_columns(AccountTransaction.id)
>>> session.execute(
...     update(AccountTransaction)
...     .values(description=AccountTransaction.description + " (audited)")
...     .where(AccountTransaction.id.in_(subq))
... )
UPDATE  account_transaction  SET  description=(account_transaction.description  ||  ?)
WHERE  account_transaction.id  IN  (SELECT  account_transaction.id
FROM  audit_transaction
WHERE  ?  =  audit_transaction.audit_id  AND  account_transaction.id  =  audit_transaction.transaction_id)
RETURNING  id
[...]  (' (audited)',  1)
<...> 
```

### 只写集合 - API 文档

| 对象名称 | 描述 |
| --- | --- |
| WriteOnlyCollection | 可以将更改同步到属性事件系统的只写集合。 |
| WriteOnlyMapped | 表示“只写”关系的 ORM 映射属性类型。 |

```py
class sqlalchemy.orm.WriteOnlyCollection
```

只写集合，可以将更改同步到属性事件系统。

使用`relationship()`的`"write_only"`延迟加载策略在映射中使用`WriteOnlyCollection`。有关此配置的背景，请参阅只写关系。

新版本 2.0 中新增。

参见

只写关系

**成员**

add(), add_all(), delete(), insert(), remove(), select(), update()

**类签名**

类`sqlalchemy.orm.WriteOnlyCollection` (`sqlalchemy.orm.writeonly.AbstractCollectionWriter`)

```py
method add(item: _T) → None
```

向此`WriteOnlyCollection`添加项目。

下一个刷新时，给定的项目将以父实例集合的形式持久化到数据库中。

```py
method add_all(iterator: Iterable[_T]) → None
```

向此`WriteOnlyCollection`添加项目的可迭代项。

给定的项目将以父实例集合的形式在下一个刷新时持久化到数据库中。

```py
method delete() → Delete
```

生成一个`Delete`，该删除将以此实例本地的`WriteOnlyCollection`来引用行。

```py
method insert() → Insert
```

对于一对多集合，生成一个`Insert`，该插入将以此实例本地的`WriteOnlyCollection`来插入新行。

该构造仅支持**不**包括`relationship.secondary`参数的`Relationship`。对于引用多对多表的关系，请使用普通的批量插入技术来生成新对象，然后使用`AbstractCollectionWriter.add_all()`将它们与集合关联起来。

```py
method remove(item: _T) → None
```

从此`WriteOnlyCollection`中移除一个项目。

下一个刷新时，给定的项目将从父实例的集合中移除。

```py
method select() → Select[Tuple[_T]]
```

生成一个表示此实例本地`WriteOnlyCollection`内行的`Select`构造。

```py
method update() → Update
```

生成一个`Update`，该更新将以此实例本地的`WriteOnlyCollection`来引用行。

```py
class sqlalchemy.orm.WriteOnlyMapped
```

表示“仅写”关系的 ORM 映射属性类型。

`WriteOnlyMapped`类型注解可用于注释式声明表映射中，以指示特定的`relationship()`应使用`lazy="write_only"`加载策略。

例如：

```py
class User(Base):
 __tablename__ = "user"
 id: Mapped[int] = mapped_column(primary_key=True)
 addresses: WriteOnlyMapped[Address] = relationship(
 cascade="all,delete-orphan"
 )
```

请参阅仅写关系部分了解背景信息。

从版本 2.0 开始新增。

另请参阅

仅写关系 - 完整背景

`DynamicMapped` - 包含传统的`Query`支持

**类签名**

类`sqlalchemy.orm.WriteOnlyMapped` (`sqlalchemy.orm.base._MappedAnnotationBase`)

## 动态关系加载器

传统功能

“动态”惰性加载策略是现在“write_only”策略的传统形式，详细信息请参见仅写关系一节。

“动态”策略从相关集合生成传统的`Query`对象。然而，“动态”关系的一个主要缺点是，有几种情况下集合会完全迭代，其中一些情况并不明显，只有通过仔细的编程和逐个测试才能预防，因此对于真正大型的集合管理，应优先选择`WriteOnlyCollection`。

动态加载器也与异步 I/O（asyncio）扩展不兼容。它可以在一定程度上使用，如 Asyncio 动态指南中所示的，但是应优先选择与 asyncio 完全兼容的`WriteOnlyCollection`，因为有一些限制。

动态关系策略允许配置一个`relationship()`，当在实例上访问时，将返回一个传统的`Query`对象，而不是集合。然后可以进一步修改`Query`以便基于过滤条件迭代数据库集合。返回的`Query`对象是`AppenderQuery`的一个实例，它结合了`Query`的加载和迭代行为以及基本的集合变异方法，如`AppenderQuery.append()`和`AppenderQuery.remove()`。

可以使用类型注释的声明形式配置“动态”加载策略，使用`DynamicMapped`注解类：

```py
from sqlalchemy.orm import DynamicMapped

class User(Base):
    __tablename__ = "user"

    id: Mapped[int] = mapped_column(primary_key=True)
    posts: DynamicMapped[Post] = relationship()
```

上面，个体`User`对象上的`User.posts`集合将返回`AppenderQuery`对象，它是`Query`的子类，也支持基本的集合变异操作：

```py
jack = session.get(User, id)

# filter Jack's blog posts
posts = jack.posts.filter(Post.headline == "this is a post")

# apply array slices
posts = jack.posts[5:20]
```

动态关系支持有限的写操作，通过 `AppenderQuery.append()` 和 `AppenderQuery.remove()` 方法：

```py
oldpost = jack.posts.filter(Post.headline == "old post").one()
jack.posts.remove(oldpost)

jack.posts.append(Post("new post"))
```

由于动态关系的读取端总是查询数据库，对底层集合的更改在数据刷新之前将不可见。但是，只要使用的 `Session` 上启用了“自动刷新”，这将在每次集合准备发出查询时自动发生。

### 动态关系加载器 - API

| 对象名称 | 描述 |
| --- | --- |
| AppenderQuery | 支持基本集合存储操作的动态查询。 |
| DynamicMapped | 代表“动态”关系的 ORM 映射属性类型。 |

```py
class sqlalchemy.orm.AppenderQuery
```

支持基本集合存储操作的动态查询。

`AppenderQuery` 上的方法包括 `Query` 的所有方法，以及用于集合持久化的其他方法。

**成员**

add(), add_all(), append(), count(), extend(), remove()

**类签名**

类 `sqlalchemy.orm.AppenderQuery` (`sqlalchemy.orm.dynamic.AppenderMixin`, `sqlalchemy.orm.Query`)

```py
method add(item: _T) → None
```

*继承自* `AppenderMixin.add()` *方法的* `AppenderMixin`

向此 `AppenderQuery` 添加一个项目。

给定的项目将在下一次提交时以父实例集合的形式持久化到数据库中。

提供此方法是为了帮助实现与 `WriteOnlyCollection` 集合类的向前兼容。

版本 2.0 中的新功能。

```py
method add_all(iterator: Iterable[_T]) → None
```

*继承自* `AppenderMixin.add_all()` *方法的* `AppenderMixin`

向此 `AppenderQuery` 添加一个项目的可迭代对象。

给定的项目将在下一次提交时以父实例集合的形式持久化到数据库中。

提供此方法是为了帮助实现与 `WriteOnlyCollection` 集合类的向前兼容。

版本 2.0 中的新功能。

```py
method append(item: _T) → None
```

*继承自* `AppenderMixin.append()` *方法的* `AppenderMixin`

将一个项目追加到此 `AppenderQuery` 中。

给定的项目将在下一个 flush 时以父实例集合的形式持久化到数据库中。

```py
method count() → int
```

*继承自* `AppenderMixin.count()` *方法的* `AppenderMixin`

返回此 `Query` 生成的 SQL 所返回的行数。

这将生成此查询的 SQL 如下：

```py
SELECT count(1) AS count_1 FROM (
    SELECT <rest of query follows...>
) AS anon_1
```

上述 SQL 返回单行，该行是 count 函数的聚合值；然后 `Query.count()` 方法返回该单个整数值。

警告

需要注意的是，count() 返回的值与此查询从 `.all()` 方法等返回的 ORM 对象数量**不同**。当 `Query` 对象被要求返回完整实体时，将基于主键**去重**，这意味着如果相同的主键值会在结果中出现多次，那么只会有一个该主键的对象存在。这不适用于针对单个列的查询。

另请参阅

我的查询结果与 query.count() 告诉我的对象数量不同 - 为什么？

要对特定列进行精细化控制以进行计数，跳过子查询的使用或以其他方式控制 FROM 子句，或者使用 `Session.query()` 与 `expression.func` 表达式结合使用，例如：

```py
from sqlalchemy import func

# count User records, without
# using a subquery.
session.query(func.count(User.id))

# return count of user "id" grouped
# by "name"
session.query(func.count(User.id)).\
        group_by(User.name)

from sqlalchemy import distinct

# count distinct "name" values
session.query(func.count(distinct(User.name)))
```

另请参阅

2.0 迁移 - ORM 用法

```py
method extend(iterator: Iterable[_T]) → None
```

*继承自* `AppenderMixin.extend()` *方法的* `AppenderMixin`

将一个项目可迭代的添加到此 `AppenderQuery` 中。

给定的项目将在下一个 flush 时以父实例集合的形式持久化到数据库中。

```py
method remove(item: _T) → None
```

*继承自* `AppenderMixin.remove()` *方法的* `AppenderMixin`

从此 `AppenderQuery` 中删除一个项目。

给定的项目将在下一个 flush 时从父实例的集合中移除。

```py
class sqlalchemy.orm.DynamicMapped
```

代表“动态”关系的 ORM 映射属性类型。

`DynamicMapped` 类型注释可用于 Annotated Declarative Table 映射中，指示对特定 `relationship()` 使用 `lazy="dynamic"` 加载策略。

传统特性

“动态”延迟加载策略是现在在 Write Only Relationships 部分中描述的 “write_only” 策略的传统形式。

例如：

```py
class User(Base):
    __tablename__ = "user"
    id: Mapped[int] = mapped_column(primary_key=True)
    addresses: DynamicMapped[Address] = relationship(
        cascade="all,delete-orphan"
    )
```

参见 Dynamic Relationship Loaders 部分的背景信息。

2.0 版本中新增。

另请参见

Dynamic Relationship Loaders - 完整背景信息

`WriteOnlyMapped` - 完全符合 2.0 版本风格

**类签名**

类 `sqlalchemy.orm.DynamicMapped` (`sqlalchemy.orm.base._MappedAnnotationBase`)

### 动态关系加载器 - API

| 对象名称 | 描述 |
| --- | --- |
| AppenderQuery | 支持基本集合存储操作的动态查询。 |
| DynamicMapped | 代表 “动态” 关系的 ORM 映射属性类型。 |

```py
class sqlalchemy.orm.AppenderQuery
```

支持基本集合存储操作的动态查询。

`AppenderQuery` 上的方法包括 `Query` 的所有方法，以及用于集合持久性的额外方法。

**成员**

add(), add_all(), append(), count(), extend(), remove()

**类签名**

类 `sqlalchemy.orm.AppenderQuery` (`sqlalchemy.orm.dynamic.AppenderMixin`, `sqlalchemy.orm.Query`) 的签名

```py
method add(item: _T) → None
```

*继承自* `AppenderMixin.add()` *方法的* `AppenderMixin`

将项添加到此 `AppenderQuery` 中。

给定的项将以父实例集合的形式在下一个 flush 中持久化到数据库中。

提供此方法是为了帮助与 `WriteOnlyCollection` 集合类保持向前兼容。

2.0 版本中新增。

```py
method add_all(iterator: Iterable[_T]) → None
```

*继承自* `AppenderMixin.add_all()` *方法的* `AppenderMixin`

将项的可迭代对象添加到此 `AppenderQuery` 中。

下一个 flush 时，给定的项将以父实例集合的形式持久化到数据库中。

提供此方法是为了帮助与 `WriteOnlyCollection` 集合类保持向前兼容。

2.0 版本中新增。

```py
method append(item: _T) → None
```

*继承自* `AppenderMixin.append()` *方法的* `AppenderMixin`

将项目附加到此 `AppenderQuery`。

给定的项目将在下一次刷新时以父实例集合的形式持久化到数据库中。

```py
method count() → int
```

*继承自* `AppenderMixin.count()` *方法的* `AppenderMixin`

返回此 `Query` 形成的 SQL 将返回的行数计数。

这将为此查询生成以下 SQL：

```py
SELECT count(1) AS count_1 FROM (
    SELECT <rest of query follows...>
) AS anon_1
```

上述 SQL 返回单行，该行是计数函数的聚合值；然后 `Query.count()` 方法返回该单个整数值。

警告

重要的是要注意，count() 返回的值 **不同于此查询从 .all() 方法等返回的 ORM 对象数**。当 `Query` 对象被要求返回完整实体时，将 **基于主键去重** 条目，这意味着如果相同的主键值会出现在结果中超过一次，则该主键的对象只会出现一次。这不适用于针对单个列的查询。

另请参阅

我的查询的对象数与 query.count() 告诉我的不一样 - 为什么？

若要对特定列进行精细控制以计数，跳过子查询的使用或以其他方式控制 FROM 子句，或者使用 `expression.func` 表达式结合 `Session.query()` 使用，即：

```py
from sqlalchemy import func

# count User records, without
# using a subquery.
session.query(func.count(User.id))

# return count of user "id" grouped
# by "name"
session.query(func.count(User.id)).\
        group_by(User.name)

from sqlalchemy import distinct

# count distinct "name" values
session.query(func.count(distinct(User.name)))
```

另请参阅

2.0 迁移 - ORM 用法

```py
method extend(iterator: Iterable[_T]) → None
```

*继承自* `AppenderMixin.extend()` *方法的* `AppenderMixin`

将项目的可迭代项添加到此 `AppenderQuery` 中。

给定的项目将在下一次刷新时以父实例集合的形式持久化到数据库中。

```py
method remove(item: _T) → None
```

*继承自* `AppenderMixin.remove()` *方法的* `AppenderMixin`

从此 `AppenderQuery` 中删除项目。

给定的项目将在下一次刷新时从父实例的集合中移除。

```py
class sqlalchemy.orm.DynamicMapped
```

代表“动态”关系的 ORM 映射属性类型。

`DynamicMapped` 类型注释可在注释的声明性表映射中使用，以指示应该为特定 `relationship()` 使用 `lazy="dynamic"` 加载器策略。

传统特性

“dynamic”延迟加载策略是当前称为“write_only”策略的旧形式，在 只写关系部分中描述。

例如：

```py
class User(Base):
    __tablename__ = "user"
    id: Mapped[int] = mapped_column(primary_key=True)
    addresses: DynamicMapped[Address] = relationship(
        cascade="all,delete-orphan"
    )
```

查看动态关系加载器部分以了解背景。

版本 2.0 中的新功能。

另请参阅

动态关系加载器 - 完整背景

`WriteOnlyMapped` - 完全符合 2.0 风格的版本

**类签名**

类 `sqlalchemy.orm.DynamicMapped` (`sqlalchemy.orm.base._MappedAnnotationBase`)

## 设置 RaiseLoad

“raise”加载的关系将在属性通常会发出延迟加载时引发 `InvalidRequestError`：

```py
class MyClass(Base):
    __tablename__ = "some_table"

    # ...

    children: Mapped[List[MyRelatedClass]] = relationship(lazy="raise")
```

在上面，对`children`集合的属性访问将在之前未填充时引发异常。这包括读访问，但对于集合，也会影响写访问，因为集合在未加载之前无法进行变异。这样做的原因是确保应用程序在某一上下文中不会发出任何意外的延迟加载。与其必须阅读 SQL 日志以确定所有必要的属性是否已经被急加载，不如使用“raise”策略，如果访问了未加载的属性，将立即引发未加载的属性。也可以在查询选项基础上使用 `raiseload()` 加载器选项。

另请参阅

使用 raiseload 防止不需要的延迟加载

## 使用被动删除

SQLAlchemy 中集合管理的一个重要方面是，当引用集合的对象被删除时，SQLAlchemy 需要考虑到位于该集合内的对象。这些对象将需要从父对象中取消关联，对于一对多集合，这意味着外键列将被设置为 NULL，或者根据 级联 设置，可能希望对这些行发出 DELETE。

工作单元过程仅仅考虑逐行对象，这意味着 DELETE 操作意味着集合中的所有行必须在刷新过程中完全加载到内存中。对于大型集合来说，这是不可行的，因此我们转而依赖数据库自身的能力来使用外键 ON DELETE 规则自动更新或删除行，指示工作单元放弃实际需要加载这些行以处理它们。可以通过在`relationship()`构造上配置`relationship.passive_deletes`来指示工作单元以这种方式工作；使用的外键约束也必须正确配置。

有关完整“被动删除”配置的更多详细信息，请参阅使用 ORM 关系的外键 ON DELETE 级联部分。
