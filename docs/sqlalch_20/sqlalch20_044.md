# ORM-启用的 INSERT、UPDATE 和 DELETE 语句

> 原文：[`docs.sqlalchemy.org/en/20/orm/queryguide/dml.html`](https://docs.sqlalchemy.org/en/20/orm/queryguide/dml.html)

关于本文档

本节利用了首次在 SQLAlchemy 统一教程中展示的 ORM 映射，如声明映射类一节所示，以及映射类继承层次结构一节中展示的继承映射。

查看此页面的 ORM 设置。

除了处理 ORM 启用的`Select`对象外，`Session.execute()`方法还可以容纳 ORM 启用的`Insert`、`Update`和`Delete`对象，它们分别以各种方式用于一次性插入、更新或删除多个数据库行。此外，还有特定于方言的支持 ORM 启用的“upserts”，这是一种自动使用 UPDATE 来处理已经存在的行的 INSERT 语句。

下表总结了本文讨论的调用形式：

| ORM 用例 | 使用的 DML 构造 | 使用以下方式传递数据 | 是否支持 RETURNING？ | 是否支持多表映射？ |
| --- | --- | --- | --- | --- |
| ORM 批量插入语句 | `insert()` | 字典列表到`Session.execute.params` | 是 | 是 |
| 使用 SQL 表达式的 ORM 批量插入 | `insert()` | 使用`Insert.values()`的`Session.execute.params` | 是 | 是 |
| 使用每行 SQL 表达式进行 ORM 批量插入 | `insert()` | 字典列表`Insert.values()` | 是 | 否 |
| ORM “upsert” 语句 | `insert()` | 字典列表`Insert.values()` | 是 | 否 |
| 通过主键进行 ORM 批量更新 | `update()` | 字典列表`Session.execute.params` | 否 | 是 |
| 使用自定义 WHERE 条件的 ORM UPDATE 和 DELETE | `update()`, `delete()` | 关键字`Update.values()` | 是 | 部分，需要手动步骤 |

## ORM 批量插入语句

一个`insert()`构造可以根据 ORM 类构建，并传递给`Session.execute()`方法。发送到`Session.execute.params`参数的参数字典列表，与`Insert`对象本身分开，将为语句调用**批量插入模式**，这基本上意味着该操作将尽可能地优化多行：

```py
>>> from sqlalchemy import insert
>>> session.execute(
...     insert(User),
...     [
...         {"name": "spongebob", "fullname": "Spongebob Squarepants"},
...         {"name": "sandy", "fullname": "Sandy Cheeks"},
...         {"name": "patrick", "fullname": "Patrick Star"},
...         {"name": "squidward", "fullname": "Squidward Tentacles"},
...         {"name": "ehkrabs", "fullname": "Eugene H. Krabs"},
...     ],
... )
INSERT  INTO  user_account  (name,  fullname)  VALUES  (?,  ?)
[...]  [('spongebob',  'Spongebob Squarepants'),  ('sandy',  'Sandy Cheeks'),  ('patrick',  'Patrick Star'),
('squidward',  'Squidward Tentacles'),  ('ehkrabs',  'Eugene H. Krabs')]
<...>
```

参数字典包含键/值对，这些对应于 ORM 映射属性，与映射的`Column`或`mapped_column()`声明以及复合声明对齐，如果这两个名称恰好不同，则键应与**ORM 映射属性名称**匹配，而不是实际数据库列名称。

在 2.0 版本中更改：将 `Insert` 构造传递给 `Session.execute()` 方法现在会调用“批量插入”，这使用了与传统的 `Session.bulk_insert_mappings()` 方法相同的功能。这是与 1.x 系列相比的行为变更，在那里 `Insert` 将以 Core 为中心的方式解释，使用列名作为值键；现在接受 ORM 属性键。通过将执行选项 `{"dml_strategy": "raw"}`传递给 `Session.execute()` 的 `Session.execution_options` 参数，可以使用 Core 风格的功能。

### 使用 RETURNING 获取新对象

批量 ORM 插入功能支持选定后端的 INSERT..RETURNING，该功能可以返回一个`Result`对象，该对象可能会返回单个列以及对应于新生成记录的完全构造的 ORM 对象。INSERT..RETURNING 需要使用支持 SQL RETURNING 语法以及支持带 RETURNING 的 executemany 的后端；除了 MySQL（MariaDB 已包含在内）外，此功能适用于所有 SQLAlchemy 包含的 后端。

举个例子，我们可以运行与之前相同的语句，同时使用 `UpdateBase.returning()` 方法，将完整的 `User` 实体作为我们希望返回的内容传递进去。 `Session.scalars()` 用于允许迭代 `User` 对象：

```py
>>> users = session.scalars(
...     insert(User).returning(User),
...     [
...         {"name": "spongebob", "fullname": "Spongebob Squarepants"},
...         {"name": "sandy", "fullname": "Sandy Cheeks"},
...         {"name": "patrick", "fullname": "Patrick Star"},
...         {"name": "squidward", "fullname": "Squidward Tentacles"},
...         {"name": "ehkrabs", "fullname": "Eugene H. Krabs"},
...     ],
... )
INSERT  INTO  user_account  (name,  fullname)
VALUES  (?,  ?),  (?,  ?),  (?,  ?),  (?,  ?),  (?,  ?)
RETURNING  id,  name,  fullname,  species
[...]  ('spongebob',  'Spongebob Squarepants',  'sandy',  'Sandy Cheeks',
'patrick',  'Patrick Star',  'squidward',  'Squidward Tentacles',
'ehkrabs',  'Eugene H. Krabs')
>>> print(users.all())
[User(name='spongebob', fullname='Spongebob Squarepants'),
 User(name='sandy', fullname='Sandy Cheeks'),
 User(name='patrick', fullname='Patrick Star'),
 User(name='squidward', fullname='Squidward Tentacles'),
 User(name='ehkrabs', fullname='Eugene H. Krabs')]
```

在上面的例子中，渲染的 SQL 采用了由 SQLite 后端请求的插入多个值功能所使用的形式，在这里，单个参数字典被嵌入到一个单个的 INSERT 语句中，以便可以使用 RETURNING。

从版本 2.0 开始更改：ORM `Session` 现在在 ORM 上下文中解释来自 `Insert`、`Update` 甚至 `Delete` 构造的 RETURNING 子句，这意味着可以传递一种混合的列表达式和 ORM 映射实体到 `Insert.returning()` 方法中，然后将以 ORM 结果从构造物如 `Select` 中提供的方式传递，包括映射实体将以 ORM 映射对象的形式在结果中提供。还存在对 ORM 加载器选项（如 `load_only()` 和 `selectinload()`）的有限支持。

#### 将返回的记录与输入数据顺序相关联

在使用带 RETURNING 的批量 INSERT 时，重要的是要注意，大多数数据库后端不提供返回的 RETURNING 记录的顺序的正式保证，包括不保证它们的顺序与输入记录的顺序相对应。对于需要确保 RETURNING 记录能够与输入数据相关联的应用程序，可以指定额外的参数 `Insert.returning.sort_by_parameter_order`，这依赖于后端可能使用特殊的 INSERT 形式来维护一个标记，该标记用于适当地重新排序返回的行，或者在某些情况下，例如在下面使用 SQLite 后端的示例中，该操作将逐行插入：

```py
>>> data = [
...     {"name": "pearl", "fullname": "Pearl Krabs"},
...     {"name": "plankton", "fullname": "Plankton"},
...     {"name": "gary", "fullname": "Gary"},
... ]
>>> user_ids = session.scalars(
...     insert(User).returning(User.id, sort_by_parameter_order=True), data
... )
INSERT  INTO  user_account  (name,  fullname)  VALUES  (?,  ?)  RETURNING  id
[...  (insertmanyvalues)  1/3  (ordered;  batch  not  supported)]  ('pearl',  'Pearl Krabs')
INSERT  INTO  user_account  (name,  fullname)  VALUES  (?,  ?)  RETURNING  id
[insertmanyvalues  2/3  (ordered;  batch  not  supported)]  ('plankton',  'Plankton')
INSERT  INTO  user_account  (name,  fullname)  VALUES  (?,  ?)  RETURNING  id
[insertmanyvalues  3/3  (ordered;  batch  not  supported)]  ('gary',  'Gary')
>>> for user_id, input_record in zip(user_ids, data):
...     input_record["id"] = user_id
>>> print(data)
[{'name': 'pearl', 'fullname': 'Pearl Krabs', 'id': 6},
{'name': 'plankton', 'fullname': 'Plankton', 'id': 7},
{'name': 'gary', 'fullname': 'Gary', 'id': 8}]
```

2.0.10 新功能：添加了 `Insert.returning.sort_by_parameter_order`，该功能在 insertmanyvalues 架构中实现。

参见

将返回的行与参数集相关联 - 介绍了确保输入数据和结果行之间对应关系的方法背景，而不会显著降低性能 ### 使用异构参数字典

ORM 批量插入功能支持“异构”的参数字典列表，这基本上意味着“各个字典可以具有不同的键”。当检测到这种情况时，ORM 将根据每个键集将参数字典分组，并相应地批处理到单独的 INSERT 语句中：

```py
>>> users = session.scalars(
...     insert(User).returning(User),
...     [
...         {
...             "name": "spongebob",
...             "fullname": "Spongebob Squarepants",
...             "species": "Sea Sponge",
...         },
...         {"name": "sandy", "fullname": "Sandy Cheeks", "species": "Squirrel"},
...         {"name": "patrick", "species": "Starfish"},
...         {
...             "name": "squidward",
...             "fullname": "Squidward Tentacles",
...             "species": "Squid",
...         },
...         {"name": "ehkrabs", "fullname": "Eugene H. Krabs", "species": "Crab"},
...     ],
... )
INSERT  INTO  user_account  (name,  fullname,  species)
VALUES  (?,  ?,  ?),  (?,  ?,  ?)  RETURNING  id,  name,  fullname,  species
[...  (insertmanyvalues)  1/1  (unordered)]  ('spongebob',  'Spongebob Squarepants',  'Sea Sponge',
'sandy',  'Sandy Cheeks',  'Squirrel')
INSERT  INTO  user_account  (name,  species)
VALUES  (?,  ?)  RETURNING  id,  name,  fullname,  species
[...]  ('patrick',  'Starfish')
INSERT  INTO  user_account  (name,  fullname,  species)
VALUES  (?,  ?,  ?),  (?,  ?,  ?)  RETURNING  id,  name,  fullname,  species
[...  (insertmanyvalues)  1/1  (unordered)]  ('squidward',  'Squidward Tentacles',
'Squid',  'ehkrabs',  'Eugene H. Krabs',  'Crab') 
```

在上面的例子中，传递的五个参数字典被转换为三个 INSERT 语句，按照每个字典中特定的键集分组，同时仍保持行顺序，即`("name", "fullname", "species")`，`("name", "species")`，`("name","fullname", "species")`。### 在 ORM 批量 INSERT 语句中发送 NULL 值

批量 ORM 插入功能利用了遗留“批量”插入行为以及总体 ORM 工作单元中存在的行为，即包含 NULL 值的行使用不引用这些列的语句进行 INSERT；这样做的理由是，包含服务器端 INSERT 默认值的后端和模式可能对 NULL 值与没有值的存在敏感，并且会产生预期的服务器端值。这种默认行为会将批量插入的批次分解为更多的行数较少的批次：

```py
>>> session.execute(
...     insert(User),
...     [
...         {
...             "name": "name_a",
...             "fullname": "Employee A",
...             "species": "Squid",
...         },
...         {
...             "name": "name_b",
...             "fullname": "Employee B",
...             "species": "Squirrel",
...         },
...         {
...             "name": "name_c",
...             "fullname": "Employee C",
...             "species": None,
...         },
...         {
...             "name": "name_d",
...             "fullname": "Employee D",
...             "species": "Bluefish",
...         },
...     ],
... )
INSERT  INTO  user_account  (name,  fullname,  species)  VALUES  (?,  ?,  ?)
[...]  [('name_a',  'Employee A',  'Squid'),  ('name_b',  'Employee B',  'Squirrel')]
INSERT  INTO  user_account  (name,  fullname)  VALUES  (?,  ?)
[...]  ('name_c',  'Employee C')
INSERT  INTO  user_account  (name,  fullname,  species)  VALUES  (?,  ?,  ?)
[...]  ('name_d',  'Employee D',  'Bluefish')
... 
```

在上面，四行的批量 INSERT 被分解成三个单独的语句，第二个语句重新格式化，不再引用包含`None`值的单个参数字典的 NULL 列。当数据集中的许多行包含随机 NULL 值时，这种默认行为可能是不希望的，因为它会导致“executemany”操作被分解为更多的较小操作；特别是当依赖于 insertmanyvalues 来减少总语句数时，这可能会产生更大的性能影响。

要禁用对参数中的`None`值进行分批处理的操作，请传递执行选项`render_nulls=True`；这将导致所有参数字典被等效处理，假定每个字典中具有相同的键集：

```py
>>> session.execute(
...     insert(User).execution_options(render_nulls=True),
...     [
...         {
...             "name": "name_a",
...             "fullname": "Employee A",
...             "species": "Squid",
...         },
...         {
...             "name": "name_b",
...             "fullname": "Employee B",
...             "species": "Squirrel",
...         },
...         {
...             "name": "name_c",
...             "fullname": "Employee C",
...             "species": None,
...         },
...         {
...             "name": "name_d",
...             "fullname": "Employee D",
...             "species": "Bluefish",
...         },
...     ],
... )
INSERT  INTO  user_account  (name,  fullname,  species)  VALUES  (?,  ?,  ?)
[...]  [('name_a',  'Employee A',  'Squid'),  ('name_b',  'Employee B',  'Squirrel'),  ('name_c',  'Employee C',  None),  ('name_d',  'Employee D',  'Bluefish')]
... 
```

在上面，所有的参数字典都被发送到一个单独的 INSERT 批处理中，包括第三个参数字典中存在的`None`值。

新版本 2.0.23 中：添加了`render_nulls`执行选项，该选项反映了遗留的`Session.bulk_insert_mappings.render_nulls`参数的行为。### 用于连接表继承的批量 INSERT

ORM 批量插入建立在传统的工作单元系统使用的内部系统之上，以发出 INSERT 语句。这意味着对于映射到多个表的 ORM 实体，通常是使用联接表继承进行映射的实体，批量插入操作将为映射表示的每个表发出一个 INSERT 语句，正确地将服务器生成的主键值传递给依赖于它们的表行。此处还支持 RETURNING 功能，ORM 将为执行的每个 INSERT 语句接收`Result`对象，然后“水平拼接”它们，以便返回的行包括插入的所有列的值：

```py
>>> managers = session.scalars(
...     insert(Manager).returning(Manager),
...     [
...         {"name": "sandy", "manager_name": "Sandy Cheeks"},
...         {"name": "ehkrabs", "manager_name": "Eugene H. Krabs"},
...     ],
... )
INSERT  INTO  employee  (name,  type)  VALUES  (?,  ?)  RETURNING  id,  name,  type
[...  (insertmanyvalues)  1/2  (ordered;  batch  not  supported)]  ('sandy',  'manager')
INSERT  INTO  employee  (name,  type)  VALUES  (?,  ?)  RETURNING  id,  name,  type
[insertmanyvalues  2/2  (ordered;  batch  not  supported)]  ('ehkrabs',  'manager')
INSERT  INTO  manager  (id,  manager_name)  VALUES  (?,  ?),  (?,  ?)  RETURNING  id,  manager_name,  id  AS  id__1
[...  (insertmanyvalues)  1/1  (ordered)]  (1,  'Sandy Cheeks',  2,  'Eugene H. Krabs') 
```

提示

加入继承映射的批量插入要求 ORM 在内部使用`Insert.returning.sort_by_parameter_order`参数，以便它可以将来自基表的 RETURNING 行的主键值与用于插入“子”表的参数集相关联，这就是为什么上面示例中的 SQLite 后端会透明地降级到使用非批量语句。有关此功能的背景信息，请参阅将 RETURNING 行与参数集相关联。### 使用 SQL 表达式进行 ORM 批量插入

ORM 批量插入功能支持添加一组固定的参数，其中可能包括要应用于每个目标行的 SQL 表达式。为了实现这一点，结合使用`Insert.values()`方法，传递一个将应用于所有行的参数字典，以及在调用`Session.execute()`时包含包含单个行值的参数字典列表的常规批量调用形式。

例如，给定一个包含“timestamp”列的 ORM 映射：

```py
import datetime

class LogRecord(Base):
    __tablename__ = "log_record"
    id: Mapped[int] = mapped_column(primary_key=True)
    message: Mapped[str]
    code: Mapped[str]
    timestamp: Mapped[datetime.datetime]
```

如果我们想要插入一系列具有唯一`message`字段的`LogRecord`元素，但是我们希望对所有行应用 SQL 函数`now()`，我们可以在`Insert.values()`中传递`timestamp`，然后使用“bulk”模式传递额外的记录：

```py
>>> from sqlalchemy import func
>>> log_record_result = session.scalars(
...     insert(LogRecord).values(code="SQLA", timestamp=func.now()).returning(LogRecord),
...     [
...         {"message": "log message #1"},
...         {"message": "log message #2"},
...         {"message": "log message #3"},
...         {"message": "log message #4"},
...     ],
... )
INSERT  INTO  log_record  (message,  code,  timestamp)
VALUES  (?,  ?,  CURRENT_TIMESTAMP),  (?,  ?,  CURRENT_TIMESTAMP),
(?,  ?,  CURRENT_TIMESTAMP),  (?,  ?,  CURRENT_TIMESTAMP)
RETURNING  id,  message,  code,  timestamp
[...  (insertmanyvalues)  1/1  (unordered)]  ('log message #1',  'SQLA',  'log message #2',
'SQLA',  'log message #3',  'SQLA',  'log message #4',  'SQLA')
>>> print(log_record_result.all())
[LogRecord('log message #1', 'SQLA', datetime.datetime(...)),
 LogRecord('log message #2', 'SQLA', datetime.datetime(...)),
 LogRecord('log message #3', 'SQLA', datetime.datetime(...)),
 LogRecord('log message #4', 'SQLA', datetime.datetime(...))]
```

#### 使用每行 SQL 表达式进行 ORM 批量插入

`Insert.values()`方法本身直接接受参数字典列表。当以这种方式使用`Insert`构造时，如果没有将参数字典列表传递给`Session.execute.params`参数，则不使用批量 ORM 插入模式，而是将 INSERT 语句完全按照给定的方式呈现并且仅调用一次。这种操作模式既对于逐行传递 SQL 表达式的情况有用，也适用于使用 ORM 的“upsert”语句时，本章后面的文档中有介绍，位于 ORM “upsert” Statements。

下面是一个构造性的示例，其中嵌入了每行 SQL 表达式的 INSERT，还以这种形式演示了`Insert.returning()`：

```py
>>> from sqlalchemy import select
>>> address_result = session.scalars(
...     insert(Address)
...     .values(
...         [
...             {
...                 "user_id": select(User.id).where(User.name == "sandy"),
...                 "email_address": "sandy@company.com",
...             },
...             {
...                 "user_id": select(User.id).where(User.name == "spongebob"),
...                 "email_address": "spongebob@company.com",
...             },
...             {
...                 "user_id": select(User.id).where(User.name == "patrick"),
...                 "email_address": "patrick@company.com",
...             },
...         ]
...     )
...     .returning(Address),
... )
INSERT  INTO  address  (user_id,  email_address)  VALUES
((SELECT  user_account.id
FROM  user_account
WHERE  user_account.name  =  ?),  ?),  ((SELECT  user_account.id
FROM  user_account
WHERE  user_account.name  =  ?),  ?),  ((SELECT  user_account.id
FROM  user_account
WHERE  user_account.name  =  ?),  ?)  RETURNING  id,  user_id,  email_address
[...]  ('sandy',  'sandy@company.com',  'spongebob',  'spongebob@company.com',
'patrick',  'patrick@company.com')
>>> print(address_result.all())
[Address(email_address='sandy@company.com'),
 Address(email_address='spongebob@company.com'),
 Address(email_address='patrick@company.com')]
```

因为上面没有使用批量 ORM 插入模式，所以下面的功能不可用：

+   不支持联接表继承或其他多表映射，因为这将需要多个 INSERT 语句。

+   不支持异构参数集 - VALUES 集合中的每个元素必须具有相同的列。

+   不提供核心级别的规模优化，例如 insertmanyvalues 提供的批处理; 语句需要确保参数的总数不超过后端数据库施加的限制。

由于上述原因，通常不建议在 ORM INSERT 语句中使用`Insert.values()`与多个参数集合，除非有明确的理由，即要么使用了“upsert”，要么需要在每个参数集合中嵌入每行 SQL 表达式。

另请参阅

ORM “upsert” Statements  ### Legacy Session Bulk INSERT Methods

`Session`包括用于执行“批量”INSERT 和 UPDATE 语句的传统方法。 这些方法与 SQLAlchemy 2.0 版本的这些功能共享实现，描述在 ORM 批量 INSERT 语句和 ORM 按主键批量 UPDATE，但缺少许多功能，即不支持 RETURNING 支持以及不支持会话同步。

使用`Session.bulk_insert_mappings()` 的代码，例如可以像下面这样移植代码，从这个映射示例开始：

```py
session.bulk_insert_mappings(User, [{"name": "u1"}, {"name": "u2"}, {"name": "u3"}])
```

以上内容可使用新 API 表达为：

```py
from sqlalchemy import insert

session.execute(insert(User), [{"name": "u1"}, {"name": "u2"}, {"name": "u3"}])
```

另请参阅

传统会话批量更新方法  ### ORM “upsert” 语句

在 SQLAlchemy 中，选定的后端可能包括特定方言的`Insert` 构造，这些构造还具有执行“upserts”或将参数集中的现有行转换为近似 UPDATE 语句的能力。对于“现有行”，这可能意味着共享相同主键值的行，或者可能是指被视为唯一的行内其他索引列；这取决于正在使用的后端的能力。

SQLAlchemy 包含有包含特定方言的“upsert” API 特性的方言，它们是：

+   SQLite - 使用`Insert`，文档位于 INSERT…ON CONFLICT（Upsert）

+   PostgreSQL - 使用`Insert`，文档位于 INSERT…ON CONFLICT（Upsert）

+   MySQL/MariaDB - 使用`Insert`，文档位于 INSERT…ON DUPLICATE KEY UPDATE（Upsert）

用户应该查阅上述章节以了解正确构建这些对象的背景；特别是，“upsert” 方法通常需要参考原始语句，因此通常语句会分为两个独立的步骤构建。

第三方后端，如在 外部方言 中提到的后端，也可能具有类似的构造。

虽然 SQLAlchemy 还没有与后端无关的 upsert 构造，但上述的 `Insert` 变体仍然与 ORM 兼容，因为它们可以像在 ORM Bulk Insert with Per Row SQL Expressions 中所记录的那样使用与 `Insert` 构造本身相同的方式，即通过在 `Insert.values()` 方法中嵌入要插入的所需行。在下面的例子中，使用 SQLite 的 `insert()` 函数来生成包含 “ON CONFLICT DO UPDATE” 支持的 `Insert` 构造。然后，将语句传递给 `Session.execute()`，它会正常进行，但额外的特点是传递给 `Insert.values()` 的参数字典被解释为 ORM 映射的属性键，而不是列名：

```py
>>> from sqlalchemy.dialects.sqlite import insert as sqlite_upsert
>>> stmt = sqlite_upsert(User).values(
...     [
...         {"name": "spongebob", "fullname": "Spongebob Squarepants"},
...         {"name": "sandy", "fullname": "Sandy Cheeks"},
...         {"name": "patrick", "fullname": "Patrick Star"},
...         {"name": "squidward", "fullname": "Squidward Tentacles"},
...         {"name": "ehkrabs", "fullname": "Eugene H. Krabs"},
...     ]
... )
>>> stmt = stmt.on_conflict_do_update(
...     index_elements=[User.name], set_=dict(fullname=stmt.excluded.fullname)
... )
>>> session.execute(stmt)
INSERT  INTO  user_account  (name,  fullname)
VALUES  (?,  ?),  (?,  ?),  (?,  ?),  (?,  ?),  (?,  ?)
ON  CONFLICT  (name)  DO  UPDATE  SET  fullname  =  excluded.fullname
[...]  ('spongebob',  'Spongebob Squarepants',  'sandy',  'Sandy Cheeks',
'patrick',  'Patrick Star',  'squidward',  'Squidward Tentacles',
'ehkrabs',  'Eugene H. Krabs')
<...>
```

#### 使用 RETURNING 与 upsert 语句

从 SQLAlchemy ORM 的角度来看，upsert 语句看起来就像普通的 `Insert` 构造，其中包括 `Insert.returning()` 与 upsert 语句的使用方式与在 ORM Bulk Insert with Per Row SQL Expressions 中演示的方式相同，因此可以传递任何列表达式或相关的 ORM 实体类。接着上一节的例子继续：

```py
>>> result = session.scalars(
...     stmt.returning(User), execution_options={"populate_existing": True}
... )
INSERT  INTO  user_account  (name,  fullname)
VALUES  (?,  ?),  (?,  ?),  (?,  ?),  (?,  ?),  (?,  ?)
ON  CONFLICT  (name)  DO  UPDATE  SET  fullname  =  excluded.fullname
RETURNING  id,  name,  fullname,  species
[...]  ('spongebob',  'Spongebob Squarepants',  'sandy',  'Sandy Cheeks',
'patrick',  'Patrick Star',  'squidward',  'Squidward Tentacles',
'ehkrabs',  'Eugene H. Krabs')
>>> print(result.all())
[User(name='spongebob', fullname='Spongebob Squarepants'),
 User(name='sandy', fullname='Sandy Cheeks'),
 User(name='patrick', fullname='Patrick Star'),
 User(name='squidward', fullname='Squidward Tentacles'),
 User(name='ehkrabs', fullname='Eugene H. Krabs')]
```

上述示例使用 RETURNING 来返回由语句插入或更新的每一行的 ORM 对象。该示例还添加了 现有数据填充 执行选项的使用。此选项表示 `Session` 中已经存在的 `User` 对象应该使用新行的数据进行**刷新**。对于纯 `Insert` 语句来说，此选项并不重要，因为生成的每一行都是全新的主键标识。但是，当 `Insert` 还包括“upsert”选项时，它也可能会产生来自已经存在的行的结果，因此可能已经在 `Session` 对象的标识映射中具有主键标识。

另请参阅

现有数据填充  ## 按主键进行 ORM 批量更新

`Update` 构造可以与 `Session.execute()` 以类似的方式使用，就像 ORM 批量插入语句中描述的使用 `Insert` 语句一样，传递一个参数字典列表，每个字典表示对应单个主键值的单个行。这种用法不应与在 ORM 中更常见的使用 `Update` 语句的方式混淆，该方式使用显式的 WHERE 子句，在 ORM UPDATE and DELETE with Custom WHERE Criteria 中有文档记录。

对于“批量”版本的 UPDATE，通过 ORM 类构造一个`update()`语句，并传递给`Session.execute()`方法；生成的`Update`对象不应具有任何值，通常也不应具有 WHERE 条件，也就是说，不使用`Update.values()`方法，通常也不使用`Update.where()`方法，但在不寻常的情况下可能会使用，以添加额外的过滤条件。

将`Update`构造与包含完整主键值的参数字典列表一起传递将触发**主键批量 UPDATE 模式**，生成适当的 WHERE 条件以按主键匹配每一行，并使用 executemany 对 UPDATE 语句运行每个参数集：

```py
>>> from sqlalchemy import update
>>> session.execute(
...     update(User),
...     [
...         {"id": 1, "fullname": "Spongebob Squarepants"},
...         {"id": 3, "fullname": "Patrick Star"},
...         {"id": 5, "fullname": "Eugene H. Krabs"},
...     ],
... )
UPDATE  user_account  SET  fullname=?  WHERE  user_account.id  =  ?
[...]  [('Spongebob Squarepants',  1),  ('Patrick Star',  3),  ('Eugene H. Krabs',  5)]
<...>
```

请注意，每个参数字典必须为每个记录包含完整的主键，否则将引发错误。

与批量 INSERT 功能类似，这里也支持异构参数列表，其中参数将被分组为 UPDATE 运行的子批次。

在 2.0.11 版本中更改：可以使用`Update.where()`方法将附加的 WHERE 条件与 ORM 主键批量 UPDATE 组合使用以添加额外的条件。但是，此条件始终是附加到已经存在的包括主键值在内的 WHERE 条件之上的。

当使用“主键批量 UPDATE”功能时，不支持 RETURNING 功能；多个参数字典的列表必然使用了 DBAPI executemany，通常情况下，这种形式不支持结果行。

在 2.0 版中更改：将 `Update` 结构传递给 `Session.execute()` 方法以及参数字典列表现在调用“批量更新”，这使用的是与旧版 `Session.bulk_update_mappings()` 方法相同的功能。这与 1.x 系列中的行为更改不同，在 1.x 系列中，`Update` 仅受到显式 WHERE 条件和内联 VALUES 的支持。

### 禁用对具有多个参数集的 UPDATE 语句进行按主键的 ORM 批量更新

当：

1.  给出的 UPDATE 语句针对 ORM 实体

1.  `Session` 用于执行语句，而不是核心 `Connection`

1.  传递的参数是**字典列表**。

为了调用不使用“按主键的 ORM 批量更新”的 UPDATE 语句，直接使用 `Session.connection()` 方法对当前事务获取 `Connection` 执行语句：

```py
>>> from sqlalchemy import bindparam
>>> session.connection().execute(
...     update(User).where(User.name == bindparam("u_name")),
...     [
...         {"u_name": "spongebob", "fullname": "Spongebob Squarepants"},
...         {"u_name": "patrick", "fullname": "Patrick Star"},
...     ],
... )
UPDATE  user_account  SET  fullname=?  WHERE  user_account.name  =  ?
[...]  [('Spongebob Squarepants',  'spongebob'),  ('Patrick Star',  'patrick')]
<...>
```

另请参阅

按主键每行的 ORM 批量更新要求记录包含主键值  ### 联合表继承的按主键批量更新

当使用具有联合表继承的映射时，ORM 批量更新的行为与使用映射进行批量插入时类似；如 联合表继承的批量插入 中所述，批量更新操作将为映射中表示的每个表发出一条 UPDATE 语句，其中给定的参数包括要更新的值（不受影响的表将被跳过）。

示例:

```py
>>> session.execute(
...     update(Manager),
...     [
...         {
...             "id": 1,
...             "name": "scheeks",
...             "manager_name": "Sandy Cheeks, President",
...         },
...         {
...             "id": 2,
...             "name": "eugene",
...             "manager_name": "Eugene H. Krabs, VP Marketing",
...         },
...     ],
... )
UPDATE  employee  SET  name=?  WHERE  employee.id  =  ?
[...]  [('scheeks',  1),  ('eugene',  2)]
UPDATE  manager  SET  manager_name=?  WHERE  manager.id  =  ?
[...]  [('Sandy Cheeks, President',  1),  ('Eugene H. Krabs, VP Marketing',  2)]
<...>
```  ### 旧版会话批量更新方法

如传统会话批量 INSERT 方法所讨论的，`Session.bulk_update_mappings()`方法是批量更新的传统形式，当解释具有给定主键参数的`update()`语句时，ORM 在内部使用它；但是，当使用传统版本时，诸如会话同步支持之类的功能是不包括的。

下面的示例：

```py
session.bulk_update_mappings(
 User,
 [
 {"id": 1, "name": "scheeks", "manager_name": "Sandy Cheeks, President"},
 {"id": 2, "name": "eugene", "manager_name": "Eugene H. Krabs, VP Marketing"},
 ],
)
```

使用新 API 表示为：

```py
from sqlalchemy import update

session.execute(
 update(User),
 [
 {"id": 1, "name": "scheeks", "manager_name": "Sandy Cheeks, President"},
 {"id": 2, "name": "eugene", "manager_name": "Eugene H. Krabs, VP Marketing"},
 ],
)
```

请参见

传统会话批量 INSERT 方法  ## 使用自定义 WHERE 条件的 ORM UPDATE 和 DELETE

当使用自定义 WHERE 条件构造`Update`和`Delete`构造时（即使用`Update.where()`和`Delete.where()`方法），可以通过将它们传递给`Session.execute()`在 ORM 上下文中调用，而不使用`Session.execute.params`参数。对于`Update`，要更新的值应该使用`Update.values()`传递。

此使用模式与之前描述的功能不同 ORM 按主键批量更新，ORM 使用给定的 WHERE 子句，而不是将 WHERE 子句固定为主键。这意味着单个 UPDATE 或 DELETE 语句可以一次性影响许多行。

举例来说，下面发出一个 UPDATE，影响多行的“fullname”字段

```py
>>> from sqlalchemy import update
>>> stmt = (
...     update(User)
...     .where(User.name.in_(["squidward", "sandy"]))
...     .values(fullname="Name starts with S")
... )
>>> session.execute(stmt)
UPDATE  user_account  SET  fullname=?  WHERE  user_account.name  IN  (?,  ?)
[...]  ('Name starts with S',  'squidward',  'sandy')
<...>
```

对于 DELETE，基于条件删除行的示例：

```py
>>> from sqlalchemy import delete
>>> stmt = delete(User).where(User.name.in_(["squidward", "sandy"]))
>>> session.execute(stmt)
DELETE  FROM  user_account  WHERE  user_account.name  IN  (?,  ?)
[...]  ('squidward',  'sandy')
<...>
```

警告

请阅读以下部分 ORM 启用更新和删除的重要注意事项和注意事项，以了解 ORM 启用的 UPDATE 和 DELETE 功能与 ORM 工作单元 功能的功能不同，例如使用`Session.delete()`方法删除单个对象。

### ORM-启用的 Update 和 Delete 的重要说明和注意事项

ORM 启用的 UPDATE 和 DELETE 功能绕过 ORM 工作单元 自动化，以便能够发出一条匹配多行的 UPDATE 或 DELETE 语句，而不会复杂化。

+   操作不提供 Python 中的关系级联功能 - 假定任何需要的外键引用都已配置为 ON UPDATE CASCADE 和/或 ON DELETE CASCADE，否则如果强制执行外键引用，则数据库可能会发出完整性违规。有关一些示例，请参阅使用外键 ON DELETE cascade 与 ORM 关系的注意事项。

+   在 UPDATE 或 DELETE 之后，`Session` 中受到影响的依赖对象，特别是那些引用现在已被删除的行的 ON UPDATE CASCADE 或 ON DELETE CASCADE 的相关表的对象，可能仍然引用这些对象。此问题在 `Session` 过期时解决，通常发生在 `Session.commit()` 时或可以通过使用 `Session.expire_all()` 强制执行。

+   启用 ORM 的 UPDATE 和 DELETE 不会自动处理连接的表继承。有关如何处理连接继承映射的说明，请参阅具有自定义 WHERE 条件的连接表继承的 UPDATE/DELETE 部分。

+   为了将单表继承映射的多态标识限制为特定子类所需的 WHERE 条件**会自动包含**。这仅适用于没有自己表的子类映射器。

+   ORM 更新和删除操作支持 `with_loader_criteria()` 选项；此处的条件将被添加到正在发出的 UPDATE 或 DELETE 语句的条件中，并在“同步”过程中考虑。

+   要拦截启用 ORM 的 UPDATE 和 DELETE 操作以使用事件处理程序，请使用`SessionEvents.do_orm_execute()`事件。  ### 选择同步策略

在使用`update()`或`delete()`与启用 ORM 执行一起使用`Session.execute()`时，将存在额外的 ORM 特定功能，该功能将**同步**语句更改的状态与当前存在于`Session`的 identity map 中的对象的状态。通过“同步”，我们指的是更新的属性将使用新值刷新，或者至少过期，以便在下次访问时重新填充其新值，并且删除的对象将移至 deleted 状态。

此同步可通过“同步策略”控制，该策略作为字符串 ORM 执行选项传递，通常使用`Session.execute.execution_options` 字典：

```py
>>> from sqlalchemy import update
>>> stmt = (
...     update(User).where(User.name == "squidward").values(fullname="Squidward Tentacles")
... )
>>> session.execute(stmt, execution_options={"synchronize_session": False})
UPDATE  user_account  SET  fullname=?  WHERE  user_account.name  =  ?
[...]  ('Squidward Tentacles',  'squidward')
<...>
```

执行选项也可以与语句本身捆绑在一起，使用`Executable.execution_options()` 方法：

```py
>>> from sqlalchemy import update
>>> stmt = (
...     update(User)
...     .where(User.name == "squidward")
...     .values(fullname="Squidward Tentacles")
...     .execution_options(synchronize_session=False)
... )
>>> session.execute(stmt)
UPDATE  user_account  SET  fullname=?  WHERE  user_account.name  =  ?
[...]  ('Squidward Tentacles',  'squidward')
<...>
```

支持以下`synchronize_session`的值：

+   `'auto'` - 这是默认值。在支持 RETURNING 的后端上将使用 `'fetch'` 策略，这包括除 MySQL 外的所有 SQLAlchemy 本机驱动程序。如果不支持 RETURNING，则将改为使用 `'evaluate'` 策略。

+   `'fetch'` - 通过在执行 UPDATE 或 DELETE 之前执行 SELECT 或使用 RETURNING（如果数据库支持）来检索受影响行的主键标识，以便受操作影响的内存对象可以使用新值刷新（更新）或从`Session`中删除（删除）。即使给定的`update()`或`delete()`构造明确指定实体或列使用`UpdateBase.returning()`，也可以使用此同步策略。

    2.0 版中的更改：在使用启用 ORM 的 UPDATE 和 DELETE 以 WHERE 条件时，可以将明确的`UpdateBase.returning()` 与 `'fetch'` 同步策略结合使用。实际语句将包含`'fetch'`策略所需的列和请求的列之间的并集。

+   `'evaluate'` - 这表示在 Python 中评估 UPDATE 或 DELETE 语句中给定的 WHERE 条件，以定位`Session`中的匹配对象。这种方法不会为操作添加任何 SQL 往返，并且在没有 RETURNING 支持的情况下，可能更有效。对于具有复杂条件的 UPDATE 或 DELETE 语句，`'evaluate'` 策略可能无法在 Python 中评估表达式，并且会引发错误。如果发生这种情况，请改用该操作的 `'fetch'` 策略。

    提示

    如果 SQL 表达式使用`Operators.op()` 或 `custom_op` 功能使用自定义运算符，则可以使用 `Operators.op.python_impl` 参数指示将由`"evaluate"`同步策略使用的 Python 函数。

    2.0 版中的新功能。

    警告

    如果要在具有许多已过期对象的`Session`上运行 UPDATE 操作，则应避免使用`"evaluate"`策略，因为它将必须刷新对象以便根据给定的 WHERE 条件测试它们，这将为每个对象发出一个 SELECT。在这种情况下，特别是如果后端支持 RETURNING，则应优先选择`"fetch"`策略。

+   `False` - 不同步会话。该选项对于不支持 RETURNING 的后端可能很有用，其中无法使用`"evaluate"`策略。在这种情况下，`Session` 中对象的状态不变，不会自动对应于发出的 UPDATE 或 DELETE 语句，如果存在通常与匹配行对应的对象。### 使用 UPDATE/DELETE 和自定义 WHERE 条件的 RETURNING

`UpdateBase.returning()` 方法与启用 ORM 的 UPDATE 和 DELETE 以 WHERE 条件完全兼容。完整的 ORM 对象和/或列可以用于 RETURNING：

```py
>>> from sqlalchemy import update
>>> stmt = (
...     update(User)
...     .where(User.name == "squidward")
...     .values(fullname="Squidward Tentacles")
...     .returning(User)
... )
>>> result = session.scalars(stmt)
UPDATE  user_account  SET  fullname=?  WHERE  user_account.name  =  ?
RETURNING  id,  name,  fullname,  species
[...]  ('Squidward Tentacles',  'squidward')
>>> print(result.all())
[User(name='squidward', fullname='Squidward Tentacles')]
```

RETURNING 的支持也与 `fetch` 同步策略兼容，`fetch` 同样使用 RETURNING。ORM 将适当地组织 RETURNING 中的列，以便同步进程顺利进行，并且返回的 `Result` 将以请求的实体和 SQL 列的请求顺序包含。

2.0 版中的新功能：`UpdateBase.returning()` 可用于启用 ORM 的 UPDATE 和 DELETE，并仍保留与 `fetch` 同步策略的完全兼容性。### 使用自定义 WHERE 条件进行连接表继承的 UPDATE/DELETE

带有 WHERE 条件的 UPDATE/DELETE 功能，不像 基于主键的 ORM 大规模 UPDATE，每次调用 `Session.execute()` 时只发出单个 UPDATE 或 DELETE 语句。这意味着当针对多表映射运行 `update()` 或 `delete()` 语句时，如连接表继承映射中的子类，该语句必须符合后端当前的功能，这可能包括后端不支持引用多个表的 UPDATE 或 DELETE 语句，或者仅对此有限支持。这意味着对于诸如连接继承子类之类的映射，UPDATE/DELETE 功能的 ORM 版本只能在有限程度上使用或根本无法使用，具体取决于具体情况。

对于连接表子类发出多行 UPDATE 语句的最直接方法是仅引用子表。这意味着 `Update()` 构造应仅引用本地于子类表的属性，如下例所示：

```py
>>> stmt = (
...     update(Manager)
...     .where(Manager.id == 1)
...     .values(manager_name="Sandy Cheeks, President")
... )
>>> session.execute(stmt)
UPDATE  manager  SET  manager_name=?  WHERE  manager.id  =  ?
[...]  ('Sandy Cheeks, President',  1)
<...> 
```

使用上述形式，一个简单的方法是引用基表来定位任何 SQL 后端上的行，可以使用子查询：

```py
>>> stmt = (
...     update(Manager)
...     .where(
...         Manager.id
...         == select(Employee.id).where(Employee.name == "sandy").scalar_subquery()
...     )
...     .values(manager_name="Sandy Cheeks, President")
... )
>>> session.execute(stmt)
UPDATE  manager  SET  manager_name=?  WHERE  manager.id  =  (SELECT  employee.id
FROM  employee
WHERE  employee.name  =  ?)  RETURNING  id
[...]  ('Sandy Cheeks, President',  'sandy')
<...>
```

对于支持 UPDATE…FROM 的后端，子查询可以作为额外的普通 WHERE 条件陈述，但是两个表之间的条件必须以某种方式明确陈述：

```py
>>> stmt = (
...     update(Manager)
...     .where(Manager.id == Employee.id, Employee.name == "sandy")
...     .values(manager_name="Sandy Cheeks, President")
... )
>>> session.execute(stmt)
UPDATE  manager  SET  manager_name=?  FROM  employee
WHERE  manager.id  =  employee.id  AND  employee.name  =  ?
[...]  ('Sandy Cheeks, President',  'sandy')
<...>
```

对于 DELETE 操作，预期基表和子表中的行将同时被 DELETE。要在不使用级联外键的情况下 DELETE 多行连接继承对象，应分别发出针对每个表的 DELETE：

```py
>>> from sqlalchemy import delete
>>> session.execute(delete(Manager).where(Manager.id == 1))
DELETE  FROM  manager  WHERE  manager.id  =  ?
[...]  (1,)
<...>
>>> session.execute(delete(Employee).where(Employee.id == 1))
DELETE  FROM  employee  WHERE  employee.id  =  ?
[...]  (1,)
<...>
```

总的来说，对于更新和删除联合继承和其他多表映射的行，应**优先**使用普通的工作单元流程，除非存在使用自定义 WHERE 条件的性能原因。

### 旧版查询方法

原始的 ORM 启用的带有 WHERE 功能的 UPDATE/DELETE 最初是 `Query` 对象的一部分，位于 `Query.update()` 和 `Query.delete()` 方法中。这些方法仍然可用，并提供与 ORM UPDATE and DELETE with Custom WHERE Criteria 描述的部分相同的功能。主要区别在于旧版方法不提供显式的 RETURNING 支持。

请参阅

`Query.update()`

`Query.delete()`  ## ORM 批量插入语句

可以基于 ORM 类构建 `insert()` 构造，并将其传递给 `Session.execute()` 方法。发送到 `Session.execute.params` 参数的参数字典列表，与 `Insert` 对象本身分开，将为语句调用**批量插入模式**，这基本上意味着操作将尽可能地为许多行进行优化：

```py
>>> from sqlalchemy import insert
>>> session.execute(
...     insert(User),
...     [
...         {"name": "spongebob", "fullname": "Spongebob Squarepants"},
...         {"name": "sandy", "fullname": "Sandy Cheeks"},
...         {"name": "patrick", "fullname": "Patrick Star"},
...         {"name": "squidward", "fullname": "Squidward Tentacles"},
...         {"name": "ehkrabs", "fullname": "Eugene H. Krabs"},
...     ],
... )
INSERT  INTO  user_account  (name,  fullname)  VALUES  (?,  ?)
[...]  [('spongebob',  'Spongebob Squarepants'),  ('sandy',  'Sandy Cheeks'),  ('patrick',  'Patrick Star'),
('squidward',  'Squidward Tentacles'),  ('ehkrabs',  'Eugene H. Krabs')]
<...>
```

参数字典包含键值对，这些键值对可能对应于与映射的 `Column` 或 `mapped_column()` 声明相对应的 ORM 映射属性，以及与组合声明相对应的映射。如果这两个名称恰好不同，则键应与**ORM 映射属性名称**匹配，而**不是**实际的数据库列名称。

2.0 版本中的更改：将 `Insert` 结构传递给 `Session.execute()` 方法现在会调用“批量插入”，这利用了与传统的 `Session.bulk_insert_mappings()` 方法相同的功能。这与 1.x 系列中的行为变化相比，1.x 系列中 `Insert` 将以核心为中心的方式解释，使用列名作为值键；现在接受 ORM 属性键。通过将执行选项 `{"dml_strategy": "raw"}` 传递给 `Session.execute()` 方法的 `Session.execution_options` 参数，可以使用核心样式功能。

### 使用 RETURNING 获取新对象

批量 ORM 插入功能支持为选定的后端进行 INSERT..RETURNING，该功能可以返回一个 `Result` 对象，该对象可以返回单个列以及对应于新生成记录的完全构造的 ORM 对象。INSERT..RETURNING 需要使用支持 SQL RETURNING 语法以及支持带有 RETURNING 的 executemany 的后端；除了 MySQL（包括 MariaDB）之外，所有 SQLAlchemy 包含的 后端都支持此功能。

例如，我们可以运行与之前相同的语句，添加使用 `UpdateBase.returning()` 方法，并将完整的 `User` 实体作为我们要返回的内容。使用 `Session.scalars()` 允许迭代 `User` 对象：

```py
>>> users = session.scalars(
...     insert(User).returning(User),
...     [
...         {"name": "spongebob", "fullname": "Spongebob Squarepants"},
...         {"name": "sandy", "fullname": "Sandy Cheeks"},
...         {"name": "patrick", "fullname": "Patrick Star"},
...         {"name": "squidward", "fullname": "Squidward Tentacles"},
...         {"name": "ehkrabs", "fullname": "Eugene H. Krabs"},
...     ],
... )
INSERT  INTO  user_account  (name,  fullname)
VALUES  (?,  ?),  (?,  ?),  (?,  ?),  (?,  ?),  (?,  ?)
RETURNING  id,  name,  fullname,  species
[...]  ('spongebob',  'Spongebob Squarepants',  'sandy',  'Sandy Cheeks',
'patrick',  'Patrick Star',  'squidward',  'Squidward Tentacles',
'ehkrabs',  'Eugene H. Krabs')
>>> print(users.all())
[User(name='spongebob', fullname='Spongebob Squarepants'),
 User(name='sandy', fullname='Sandy Cheeks'),
 User(name='patrick', fullname='Patrick Star'),
 User(name='squidward', fullname='Squidward Tentacles'),
 User(name='ehkrabs', fullname='Eugene H. Krabs')]
```

在上面的示例中，渲染的 SQL 采用了由 SQLite 后端请求的 insertmanyvalues 功能使用的形式，其中个别参数字典被内联到单个 INSERT 语句中，以便使用 RETURNING。

在 2.0 版本中更改：ORM `Session` 现在在 ORM 上下文中解释来自 `Insert`、`Update` 甚至 `Delete` 构造的 RETURNING 子句，这意味着可以将一系列列表达式和 ORM 映射实体传递给 `Insert.returning()` 方法，然后以从构造物如 `Select` 传递 ORM 结果的方式传递，包括映射实体将作为 ORM 映射对象在结果中传递。还存在对于 ORM 加载器选项的有限支持，如 `load_only()` 和 `selectinload()`。

#### 将 RETURNING 记录与输入数据顺序相关联

使用带有 RETURNING 的批量 INSERT 时，重要的是要注意，大多数数据库后端不保证从 RETURNING 返回的记录的顺序，包括不能保证它们的顺序与输入记录的顺序对应。对于需要确保 RETURNING 记录与输入数据相关联的应用程序，可以指定额外的参数 `Insert.returning.sort_by_parameter_order`，根据后端的不同，可能使用特殊的 INSERT 表单来维护一个标记，该标记用于适当地重新排序返回的行，或者在某些情况下，例如在下面使用 SQLite 后端的示例中，操作将一次插入一行：

```py
>>> data = [
...     {"name": "pearl", "fullname": "Pearl Krabs"},
...     {"name": "plankton", "fullname": "Plankton"},
...     {"name": "gary", "fullname": "Gary"},
... ]
>>> user_ids = session.scalars(
...     insert(User).returning(User.id, sort_by_parameter_order=True), data
... )
INSERT  INTO  user_account  (name,  fullname)  VALUES  (?,  ?)  RETURNING  id
[...  (insertmanyvalues)  1/3  (ordered;  batch  not  supported)]  ('pearl',  'Pearl Krabs')
INSERT  INTO  user_account  (name,  fullname)  VALUES  (?,  ?)  RETURNING  id
[insertmanyvalues  2/3  (ordered;  batch  not  supported)]  ('plankton',  'Plankton')
INSERT  INTO  user_account  (name,  fullname)  VALUES  (?,  ?)  RETURNING  id
[insertmanyvalues  3/3  (ordered;  batch  not  supported)]  ('gary',  'Gary')
>>> for user_id, input_record in zip(user_ids, data):
...     input_record["id"] = user_id
>>> print(data)
[{'name': 'pearl', 'fullname': 'Pearl Krabs', 'id': 6},
{'name': 'plankton', 'fullname': 'Plankton', 'id': 7},
{'name': 'gary', 'fullname': 'Gary', 'id': 8}]
```

2.0.10 新内容：添加了 `Insert.returning.sort_by_parameter_order`，该内容在 insertmanyvalues 架构中实现。

另请参阅

将 RETURNING 行与参数集相关联 - 关于保证输入数据和结果行之间对应关系的方法的背景信息，而又不显著降低性能 ### 使用异构参数字典

ORM 批量插入功能支持“异构”参数字典列表，这基本上意味着“各个字典可以具有不同的键”。当检测到这种条件时，ORM 将参数字典分组成对应于每个键集的组，并相应地批量处理成单独的 INSERT 语句：

```py
>>> users = session.scalars(
...     insert(User).returning(User),
...     [
...         {
...             "name": "spongebob",
...             "fullname": "Spongebob Squarepants",
...             "species": "Sea Sponge",
...         },
...         {"name": "sandy", "fullname": "Sandy Cheeks", "species": "Squirrel"},
...         {"name": "patrick", "species": "Starfish"},
...         {
...             "name": "squidward",
...             "fullname": "Squidward Tentacles",
...             "species": "Squid",
...         },
...         {"name": "ehkrabs", "fullname": "Eugene H. Krabs", "species": "Crab"},
...     ],
... )
INSERT  INTO  user_account  (name,  fullname,  species)
VALUES  (?,  ?,  ?),  (?,  ?,  ?)  RETURNING  id,  name,  fullname,  species
[...  (insertmanyvalues)  1/1  (unordered)]  ('spongebob',  'Spongebob Squarepants',  'Sea Sponge',
'sandy',  'Sandy Cheeks',  'Squirrel')
INSERT  INTO  user_account  (name,  species)
VALUES  (?,  ?)  RETURNING  id,  name,  fullname,  species
[...]  ('patrick',  'Starfish')
INSERT  INTO  user_account  (name,  fullname,  species)
VALUES  (?,  ?,  ?),  (?,  ?,  ?)  RETURNING  id,  name,  fullname,  species
[...  (insertmanyvalues)  1/1  (unordered)]  ('squidward',  'Squidward Tentacles',
'Squid',  'ehkrabs',  'Eugene H. Krabs',  'Crab') 
```

在上面的示例中，传递的五个参数字典被转换为三个 INSERT 语句，按照每个字典中特定键的组合进行分组，同时保持行顺序，即 `("name", "fullname", "species")`、 `("name", "species")`、 `("name","fullname", "species")`。### 在 ORM 批量 INSERT 语句中发送 NULL 值

批量 ORM 插入功能借鉴了遗留的“批量”插入行为，以及 ORM 工作单元总体上的行为，即包含 NULL 值的行将使用不引用这些列的语句进行插入；这样做的理由是，包含服务器端插入默认值的后端和模式可能对 NULL 值的存在与不存在敏感，将产生预期的服务器端值。这种默认行为会将批量插入的批次分解成更多行数较少的批次：

```py
>>> session.execute(
...     insert(User),
...     [
...         {
...             "name": "name_a",
...             "fullname": "Employee A",
...             "species": "Squid",
...         },
...         {
...             "name": "name_b",
...             "fullname": "Employee B",
...             "species": "Squirrel",
...         },
...         {
...             "name": "name_c",
...             "fullname": "Employee C",
...             "species": None,
...         },
...         {
...             "name": "name_d",
...             "fullname": "Employee D",
...             "species": "Bluefish",
...         },
...     ],
... )
INSERT  INTO  user_account  (name,  fullname,  species)  VALUES  (?,  ?,  ?)
[...]  [('name_a',  'Employee A',  'Squid'),  ('name_b',  'Employee B',  'Squirrel')]
INSERT  INTO  user_account  (name,  fullname)  VALUES  (?,  ?)
[...]  ('name_c',  'Employee C')
INSERT  INTO  user_account  (name,  fullname,  species)  VALUES  (?,  ?,  ?)
[...]  ('name_d',  'Employee D',  'Bluefish')
... 
```

在上面的示例中，四行的批量插入被分成三个单独的语句，第二个语句重新格式化以不引用包含 `None` 值的单个参数字典的 NULL 列。当数据集中的许多行包含随机 NULL 值时，这种默认行为可能是不希望的，因为它会将“executemany”操作分解成更多的较小操作；特别是当依赖 insertmanyvalues 来减少总语句数时，这可能会产生更大的性能影响。

要禁用将参数中的 `None` 值处理为单独批次的行为，请传递执行选项 `render_nulls=True`；这将导致所有参数字典被视为等效处理，假定每个字典中具有相同的键集：

```py
>>> session.execute(
...     insert(User).execution_options(render_nulls=True),
...     [
...         {
...             "name": "name_a",
...             "fullname": "Employee A",
...             "species": "Squid",
...         },
...         {
...             "name": "name_b",
...             "fullname": "Employee B",
...             "species": "Squirrel",
...         },
...         {
...             "name": "name_c",
...             "fullname": "Employee C",
...             "species": None,
...         },
...         {
...             "name": "name_d",
...             "fullname": "Employee D",
...             "species": "Bluefish",
...         },
...     ],
... )
INSERT  INTO  user_account  (name,  fullname,  species)  VALUES  (?,  ?,  ?)
[...]  [('name_a',  'Employee A',  'Squid'),  ('name_b',  'Employee B',  'Squirrel'),  ('name_c',  'Employee C',  None),  ('name_d',  'Employee D',  'Bluefish')]
... 
```

在上面的示例中，所有参数字典都在单个 INSERT 批次中发送，包括第三个参数字典中的 `None` 值。

从版本 2.0.23 开始：添加了 `render_nulls` 执行选项，其反映了遗留的 `Session.bulk_insert_mappings.render_nulls` 参数的行为。### 关于连接表继承的批量 INSERT

ORM 批量插入建立在传统的 unit of work 系统使用的内部系统之上，以发出 INSERT 语句。这意味着对于一个被映射到多个表的 ORM 实体，通常是使用 joined table inheritance 映射的实体，批量插入操作将为每个由映射表示的表发出一个 INSERT 语句，正确地将服务器生成的主键值传递给依赖于它们的表行。RETURNING 特性在这里也受支持，ORM 将为每个执行的 INSERT 语句接收`Result`对象，然后将它们“水平拼接”在一起，以便返回的行包括插入的所有列的值：

```py
>>> managers = session.scalars(
...     insert(Manager).returning(Manager),
...     [
...         {"name": "sandy", "manager_name": "Sandy Cheeks"},
...         {"name": "ehkrabs", "manager_name": "Eugene H. Krabs"},
...     ],
... )
INSERT  INTO  employee  (name,  type)  VALUES  (?,  ?)  RETURNING  id,  name,  type
[...  (insertmanyvalues)  1/2  (ordered;  batch  not  supported)]  ('sandy',  'manager')
INSERT  INTO  employee  (name,  type)  VALUES  (?,  ?)  RETURNING  id,  name,  type
[insertmanyvalues  2/2  (ordered;  batch  not  supported)]  ('ehkrabs',  'manager')
INSERT  INTO  manager  (id,  manager_name)  VALUES  (?,  ?),  (?,  ?)  RETURNING  id,  manager_name,  id  AS  id__1
[...  (insertmanyvalues)  1/1  (ordered)]  (1,  'Sandy Cheeks',  2,  'Eugene H. Krabs') 
```

提示

插入连接继承映射的批量操作要求 ORM 内部使用 `Insert.returning.sort_by_parameter_order` 参数，以便它可以将来自 RETURNING 行的主键值从基表相关联到用于插入到“子”表中的参数集，这就是为什么上面示例中的 SQLite 后端会透明地降级到使用非批处理语句的原因。关于此功能的背景请参阅将 RETURNING 行与参数集相关联。### 带 SQL 表达式的 ORM 批量插入

ORM 批量插入功能支持添加一组固定的参数，其中可能包括要应用于每个目标行的 SQL 表达式。要实现这一点，请将`Insert.values()`方法与通常的批量调用形式结合使用，方法是在调用`Session.execute()`时包含包含单独行值的参数字典列表。

举例来说，考虑一个包含“timestamp”列的 ORM 映射：

```py
import datetime

class LogRecord(Base):
    __tablename__ = "log_record"
    id: Mapped[int] = mapped_column(primary_key=True)
    message: Mapped[str]
    code: Mapped[str]
    timestamp: Mapped[datetime.datetime]
```

如果我们想要插入一系列具有唯一 `message` 字段的 `LogRecord` 元素，但是我们希望将 SQL 函数 `now()` 应用于所有行，我们可以在 `Insert.values()` 中传递 `timestamp`，然后使用“批量”模式传递额外的记录：

```py
>>> from sqlalchemy import func
>>> log_record_result = session.scalars(
...     insert(LogRecord).values(code="SQLA", timestamp=func.now()).returning(LogRecord),
...     [
...         {"message": "log message #1"},
...         {"message": "log message #2"},
...         {"message": "log message #3"},
...         {"message": "log message #4"},
...     ],
... )
INSERT  INTO  log_record  (message,  code,  timestamp)
VALUES  (?,  ?,  CURRENT_TIMESTAMP),  (?,  ?,  CURRENT_TIMESTAMP),
(?,  ?,  CURRENT_TIMESTAMP),  (?,  ?,  CURRENT_TIMESTAMP)
RETURNING  id,  message,  code,  timestamp
[...  (insertmanyvalues)  1/1  (unordered)]  ('log message #1',  'SQLA',  'log message #2',
'SQLA',  'log message #3',  'SQLA',  'log message #4',  'SQLA')
>>> print(log_record_result.all())
[LogRecord('log message #1', 'SQLA', datetime.datetime(...)),
 LogRecord('log message #2', 'SQLA', datetime.datetime(...)),
 LogRecord('log message #3', 'SQLA', datetime.datetime(...)),
 LogRecord('log message #4', 'SQLA', datetime.datetime(...))]
```

#### 带每行 SQL 表达式的 ORM 批量插入

`Insert.values()` 方法本身直接适应参数字典列表。 当以这种方式使用 `Insert` 构造时，而不向 `Session.execute.params` 参数传递任何参数字典列表时，不会使用批量 ORM 插入模式，而是精确地按照给定的方式呈现 INSERT 语句，并且仅调用一次。 这种操作模式在每行基础上传递 SQL 表达式的情况下可能有用，并且在使用 ORM 时使用“upsert”语句时也会使用，本章后面的文档中有描述，位于 ORM “upsert” 语句。

下面是一个人为的示例，展示了嵌入每行 SQL 表达式的 INSERT，并演示了此形式中的 `Insert.returning()`：

```py
>>> from sqlalchemy import select
>>> address_result = session.scalars(
...     insert(Address)
...     .values(
...         [
...             {
...                 "user_id": select(User.id).where(User.name == "sandy"),
...                 "email_address": "sandy@company.com",
...             },
...             {
...                 "user_id": select(User.id).where(User.name == "spongebob"),
...                 "email_address": "spongebob@company.com",
...             },
...             {
...                 "user_id": select(User.id).where(User.name == "patrick"),
...                 "email_address": "patrick@company.com",
...             },
...         ]
...     )
...     .returning(Address),
... )
INSERT  INTO  address  (user_id,  email_address)  VALUES
((SELECT  user_account.id
FROM  user_account
WHERE  user_account.name  =  ?),  ?),  ((SELECT  user_account.id
FROM  user_account
WHERE  user_account.name  =  ?),  ?),  ((SELECT  user_account.id
FROM  user_account
WHERE  user_account.name  =  ?),  ?)  RETURNING  id,  user_id,  email_address
[...]  ('sandy',  'sandy@company.com',  'spongebob',  'spongebob@company.com',
'patrick',  'patrick@company.com')
>>> print(address_result.all())
[Address(email_address='sandy@company.com'),
 Address(email_address='spongebob@company.com'),
 Address(email_address='patrick@company.com')]
```

因为上面没有使用批量 ORM 插入模式，所以以下功能不可用：

+   不支持联合表继承或其他多表映射，因为这将需要多个 INSERT 语句。

+   不支持异构参数集 - 值集中的每个元素必须具有相同的列。

+   不可用于核心级别的规模优化，例如 insertmanyvalues 提供的批处理；语句需要确保参数的总数不超过由支持数据库施加的限制。

由于上述原因，通常不建议在 ORM INSERT 语句中使用多个参数集与 `Insert.values()`，除非有明确的理由，即正在使用“upsert”或需要在每个参数集中嵌入每行 SQL 表达式。

另请参见

ORM “upsert” 语句  ### 传统会话批量插入方法

`Session` 包括执行“批量”插入和更新语句的传统方法。 这些方法与 SQLAlchemy 2.0 版本的这些功能共享实现，描述在 ORM 批量插入语句 和 ORM 按主键批量更新，但缺少许多功能，即不支持 RETURNING 和会话同步支持。

使用`Session.bulk_insert_mappings()`等代码可以按照以下方式移植代码，从这个映射示例开始：

```py
session.bulk_insert_mappings(User, [{"name": "u1"}, {"name": "u2"}, {"name": "u3"}])
```

上述内容使用新 API 表达为：

```py
from sqlalchemy import insert

session.execute(insert(User), [{"name": "u1"}, {"name": "u2"}, {"name": "u3"}])
```

另请参阅

传统会话批量更新方法  ### ORM “upsert”语句

使用 SQLAlchemy 的选定后端可能包括特定于方言的`Insert`构造，这些构造还具有执行“upserts”或将参数集中的现有行转换为近似 UPDATE 语句的能力。通过“现有行”，这可能意味着共享相同主键值的行，或者可能指其他被视为唯一的行内索引列；这取决于所使用后端的功能。

SQLAlchemy 包含的包括特定于方言的“upsert”API 功能的方言有：

+   SQLite - 使用`Insert`，文档位于 INSERT…ON CONFLICT (Upsert)

+   PostgreSQL - 使用`Insert`，文档位于 INSERT…ON CONFLICT (Upsert)

+   MySQL/MariaDB - 使用`Insert`，文档位于 INSERT…ON DUPLICATE KEY UPDATE (Upsert)

用户应该查看上述部分以了解正确构建这些对象的背景；特别是，“upsert”方法通常需要参考原始语句，因此语句通常分两步构建。

第三方后端，如在外部方言中提到的可能还具有类似的构造。

虽然 SQLAlchemy 尚未拥有与后端无关的 upsert 构造，但上述`Insert`变体在 ORM 兼容方面仍然可用，因为它们可以像文档中记录的`Insert`构造本身一样使用，方法是将要插入的期望行嵌入到`Insert.values()`方法中。在下面的示例中，使用 SQLite 的`insert()`函数生成了一个包含“ON CONFLICT DO UPDATE”支持的`Insert`构造。然后将该语句传递给`Session.execute()`，它会正常进行，额外的特点是传递给`Insert.values()`的参数字典被解释为 ORM 映射的属性键，而不是列名：

```py
>>> from sqlalchemy.dialects.sqlite import insert as sqlite_upsert
>>> stmt = sqlite_upsert(User).values(
...     [
...         {"name": "spongebob", "fullname": "Spongebob Squarepants"},
...         {"name": "sandy", "fullname": "Sandy Cheeks"},
...         {"name": "patrick", "fullname": "Patrick Star"},
...         {"name": "squidward", "fullname": "Squidward Tentacles"},
...         {"name": "ehkrabs", "fullname": "Eugene H. Krabs"},
...     ]
... )
>>> stmt = stmt.on_conflict_do_update(
...     index_elements=[User.name], set_=dict(fullname=stmt.excluded.fullname)
... )
>>> session.execute(stmt)
INSERT  INTO  user_account  (name,  fullname)
VALUES  (?,  ?),  (?,  ?),  (?,  ?),  (?,  ?),  (?,  ?)
ON  CONFLICT  (name)  DO  UPDATE  SET  fullname  =  excluded.fullname
[...]  ('spongebob',  'Spongebob Squarepants',  'sandy',  'Sandy Cheeks',
'patrick',  'Patrick Star',  'squidward',  'Squidward Tentacles',
'ehkrabs',  'Eugene H. Krabs')
<...>
```

#### 使用 RETURNING 语句与 upsert 语句

从 SQLAlchemy ORM 的角度来看，upsert 语句看起来像是常规的`Insert`构造，其中包括`Insert.returning()`与在 ORM Bulk Insert with Per Row SQL Expressions 中演示的方式一样与 upsert 语句一起工作，以便传递任何列表达式或相关的 ORM 实体类。继续从前一节的示例继续进行：

```py
>>> result = session.scalars(
...     stmt.returning(User), execution_options={"populate_existing": True}
... )
INSERT  INTO  user_account  (name,  fullname)
VALUES  (?,  ?),  (?,  ?),  (?,  ?),  (?,  ?),  (?,  ?)
ON  CONFLICT  (name)  DO  UPDATE  SET  fullname  =  excluded.fullname
RETURNING  id,  name,  fullname,  species
[...]  ('spongebob',  'Spongebob Squarepants',  'sandy',  'Sandy Cheeks',
'patrick',  'Patrick Star',  'squidward',  'Squidward Tentacles',
'ehkrabs',  'Eugene H. Krabs')
>>> print(result.all())
[User(name='spongebob', fullname='Spongebob Squarepants'),
 User(name='sandy', fullname='Sandy Cheeks'),
 User(name='patrick', fullname='Patrick Star'),
 User(name='squidward', fullname='Squidward Tentacles'),
 User(name='ehkrabs', fullname='Eugene H. Krabs')]
```

以上示例使用 RETURNING 来返回由语句插入或更新的每一行的 ORM 对象。该示例还添加了对 已有对象填充 执行选项的使用。此选项表示对于已存在的 `Session` 中已经存在的行的 `User` 对象应该使用新行的数据进行 **刷新**。对于纯 `Insert` 语句来说，此选项并不重要，因为每个生成的行都是全新的主键标识。但是当 `Insert` 还包括“upsert”选项时，它也可能会产生来自已存在行的结果，因此这些行可能已经在 `Session` 对象的 标识映射 中表示了主键标识。

另见

已有对象填充 ### 使用 RETURNING 获取新对象

批量 ORM 插入功能支持选定后端的 INSERT..RETURNING，它可以返回一个 `Result` 对象，该对象可以返回单独的列以及与新生成记录相对应的完全构造的 ORM 对象。INSERT..RETURNING 需要使用支持 SQL RETURNING 语法以及支持带 RETURNING 的 executemany 的后端；除了 MySQL（MariaDB 已包含在内）之外，此功能在所有 SQLAlchemy 包含的 后端中都可用。

例如，我们可以运行与之前相同的语句，添加对 `UpdateBase.returning()` 方法的使用，并将完整的 `User` 实体作为我们想要返回的内容传递。使用 `Session.scalars()` 允许迭代 `User` 对象：

```py
>>> users = session.scalars(
...     insert(User).returning(User),
...     [
...         {"name": "spongebob", "fullname": "Spongebob Squarepants"},
...         {"name": "sandy", "fullname": "Sandy Cheeks"},
...         {"name": "patrick", "fullname": "Patrick Star"},
...         {"name": "squidward", "fullname": "Squidward Tentacles"},
...         {"name": "ehkrabs", "fullname": "Eugene H. Krabs"},
...     ],
... )
INSERT  INTO  user_account  (name,  fullname)
VALUES  (?,  ?),  (?,  ?),  (?,  ?),  (?,  ?),  (?,  ?)
RETURNING  id,  name,  fullname,  species
[...]  ('spongebob',  'Spongebob Squarepants',  'sandy',  'Sandy Cheeks',
'patrick',  'Patrick Star',  'squidward',  'Squidward Tentacles',
'ehkrabs',  'Eugene H. Krabs')
>>> print(users.all())
[User(name='spongebob', fullname='Spongebob Squarepants'),
 User(name='sandy', fullname='Sandy Cheeks'),
 User(name='patrick', fullname='Patrick Star'),
 User(name='squidward', fullname='Squidward Tentacles'),
 User(name='ehkrabs', fullname='Eugene H. Krabs')]
```

在上面的示例中，呈现的 SQL 采用了由 SQLite 后端请求的 insertmanyvalues 功能使用的形式，其中个别参数字典被内联到单个 INSERT 语句中，以便使用 RETURNING。

2.0 版本中的更改：ORM `Session` 现在会在 ORM 上下文中解释来自 `Insert`、`Update` 甚至 `Delete` 构造的 RETURNING 子句，这意味着可以传递混合的列表达式和 ORM 映射实体给 `Insert.returning()` 方法，然后将以与 `Select` 等构造中的 ORM 结果相同的方式传递，包括将映射实体作为 ORM 映射对象在结果中传递。还存在对 ORM 加载器选项（例如 `load_only()` 和 `selectinload()`）的有限支持。

#### 将 RETURNING 记录与输入数据顺序相关联

在使用带有 RETURNING 的批量 INSERT 时，重要的是要注意，大多数数据库后端没有明确保证返回的 RETURNING 记录的顺序，包括没有保证其顺序与输入记录的顺序相对应。对于需要确保 RETURNING 记录与输入数据相关联的应用程序，可以指定额外的参数 `Insert.returning.sort_by_parameter_order`，这取决于后端可能使用特殊的 INSERT 形式，以保持适当地重新排序返回的行，或者在某些情况下，例如在使用 SQLite 后端的下面示例中，该操作将逐行插入：

```py
>>> data = [
...     {"name": "pearl", "fullname": "Pearl Krabs"},
...     {"name": "plankton", "fullname": "Plankton"},
...     {"name": "gary", "fullname": "Gary"},
... ]
>>> user_ids = session.scalars(
...     insert(User).returning(User.id, sort_by_parameter_order=True), data
... )
INSERT  INTO  user_account  (name,  fullname)  VALUES  (?,  ?)  RETURNING  id
[...  (insertmanyvalues)  1/3  (ordered;  batch  not  supported)]  ('pearl',  'Pearl Krabs')
INSERT  INTO  user_account  (name,  fullname)  VALUES  (?,  ?)  RETURNING  id
[insertmanyvalues  2/3  (ordered;  batch  not  supported)]  ('plankton',  'Plankton')
INSERT  INTO  user_account  (name,  fullname)  VALUES  (?,  ?)  RETURNING  id
[insertmanyvalues  3/3  (ordered;  batch  not  supported)]  ('gary',  'Gary')
>>> for user_id, input_record in zip(user_ids, data):
...     input_record["id"] = user_id
>>> print(data)
[{'name': 'pearl', 'fullname': 'Pearl Krabs', 'id': 6},
{'name': 'plankton', 'fullname': 'Plankton', 'id': 7},
{'name': 'gary', 'fullname': 'Gary', 'id': 8}]
```

新版本 2.0.10 中新增了 `Insert.returning.sort_by_parameter_order`，该功能是在 insertmanyvalues 架构内实现的。

请参见

将 RETURNING 行与参数集相关联 - 关于采取的方法背景，以确保输入数据与结果行之间的对应关系，而不会显著降低性能  #### 将 RETURNING 记录与输入数据顺序相关联

当使用带有 RETURNING 的批量 INSERT 时，重要的是要注意，大多数数据库后端不保证返回 RETURNING 记录的顺序，包括它们的顺序与输入记录的顺序相对应这一点。对于需要确保 RETURNING 记录与输入数据相关联的应用程序，可以指定额外的参数 `Insert.returning.sort_by_parameter_order`，具体取决于后端，它可能使用特殊的 INSERT 形式来维护一个令牌，该令牌用于适当地重新排序返回的行，或者在某些情况下，例如使用 SQLite 后端的以下示例中，该操作将一次插入一行：

```py
>>> data = [
...     {"name": "pearl", "fullname": "Pearl Krabs"},
...     {"name": "plankton", "fullname": "Plankton"},
...     {"name": "gary", "fullname": "Gary"},
... ]
>>> user_ids = session.scalars(
...     insert(User).returning(User.id, sort_by_parameter_order=True), data
... )
INSERT  INTO  user_account  (name,  fullname)  VALUES  (?,  ?)  RETURNING  id
[...  (insertmanyvalues)  1/3  (ordered;  batch  not  supported)]  ('pearl',  'Pearl Krabs')
INSERT  INTO  user_account  (name,  fullname)  VALUES  (?,  ?)  RETURNING  id
[insertmanyvalues  2/3  (ordered;  batch  not  supported)]  ('plankton',  'Plankton')
INSERT  INTO  user_account  (name,  fullname)  VALUES  (?,  ?)  RETURNING  id
[insertmanyvalues  3/3  (ordered;  batch  not  supported)]  ('gary',  'Gary')
>>> for user_id, input_record in zip(user_ids, data):
...     input_record["id"] = user_id
>>> print(data)
[{'name': 'pearl', 'fullname': 'Pearl Krabs', 'id': 6},
{'name': 'plankton', 'fullname': 'Plankton', 'id': 7},
{'name': 'gary', 'fullname': 'Gary', 'id': 8}]
```

从 2.0.10 版开始：新增了 `Insert.returning.sort_by_parameter_order`，它是在 insertmanyvalues 架构中实现的。

另请参阅

将 RETURNING 行与参数集对应起来 - 关于采取的方法，以确保输入数据与结果行之间的对应关系而不会显著降低性能

### 使用异构参数字典

ORM 批量插入功能支持“异构”参数字典列表，这基本上意味着“各个字典可以具有不同的键”。当检测到这种条件时，ORM 将参数字典分组为对应于每个键集的组，并相应地将它们分批成单独的 INSERT 语句：

```py
>>> users = session.scalars(
...     insert(User).returning(User),
...     [
...         {
...             "name": "spongebob",
...             "fullname": "Spongebob Squarepants",
...             "species": "Sea Sponge",
...         },
...         {"name": "sandy", "fullname": "Sandy Cheeks", "species": "Squirrel"},
...         {"name": "patrick", "species": "Starfish"},
...         {
...             "name": "squidward",
...             "fullname": "Squidward Tentacles",
...             "species": "Squid",
...         },
...         {"name": "ehkrabs", "fullname": "Eugene H. Krabs", "species": "Crab"},
...     ],
... )
INSERT  INTO  user_account  (name,  fullname,  species)
VALUES  (?,  ?,  ?),  (?,  ?,  ?)  RETURNING  id,  name,  fullname,  species
[...  (insertmanyvalues)  1/1  (unordered)]  ('spongebob',  'Spongebob Squarepants',  'Sea Sponge',
'sandy',  'Sandy Cheeks',  'Squirrel')
INSERT  INTO  user_account  (name,  species)
VALUES  (?,  ?)  RETURNING  id,  name,  fullname,  species
[...]  ('patrick',  'Starfish')
INSERT  INTO  user_account  (name,  fullname,  species)
VALUES  (?,  ?,  ?),  (?,  ?,  ?)  RETURNING  id,  name,  fullname,  species
[...  (insertmanyvalues)  1/1  (unordered)]  ('squidward',  'Squidward Tentacles',
'Squid',  'ehkrabs',  'Eugene H. Krabs',  'Crab') 
```

在上面的示例中，传递的五个参数字典被转换为三个 INSERT 语句，根据每个字典中的特定键组合成组，同时保持行顺序，即 `("name", "fullname", "species")`，`("name", "species")`，`("name","fullname", "species")`。

### 在 ORM 批量 INSERT 语句中发送 NULL 值

批量 ORM 插入功能利用了在传统“批量”插入行为以及整体 ORM 工作单元中也存在的行为，即包含 NULL 值的行将使用不引用这些列的语句进行 INSERT；其理由是后端和包含服务器端 INSERT 默认值的模式可能对存在 NULL 值与不存在值的情况敏感，将产生预期的服务器端值。这种默认行为会导致批量插入的批次被分成更多的少行批次：

```py
>>> session.execute(
...     insert(User),
...     [
...         {
...             "name": "name_a",
...             "fullname": "Employee A",
...             "species": "Squid",
...         },
...         {
...             "name": "name_b",
...             "fullname": "Employee B",
...             "species": "Squirrel",
...         },
...         {
...             "name": "name_c",
...             "fullname": "Employee C",
...             "species": None,
...         },
...         {
...             "name": "name_d",
...             "fullname": "Employee D",
...             "species": "Bluefish",
...         },
...     ],
... )
INSERT  INTO  user_account  (name,  fullname,  species)  VALUES  (?,  ?,  ?)
[...]  [('name_a',  'Employee A',  'Squid'),  ('name_b',  'Employee B',  'Squirrel')]
INSERT  INTO  user_account  (name,  fullname)  VALUES  (?,  ?)
[...]  ('name_c',  'Employee C')
INSERT  INTO  user_account  (name,  fullname,  species)  VALUES  (?,  ?,  ?)
[...]  ('name_d',  'Employee D',  'Bluefish')
... 
```

上面，四行的批量插入被分解为三个单独的语句，第二个语句重新格式化以不引用包含`None`值的单个参数字典的 NULL 列。当数据集中的许多行包含随机 NULL 值时，此默认行为可能是不希望的，因为它会导致“executemany”操作被分解为更多的较小操作；特别是当依赖于 insertmanyvalues 来减少总体语句数时，这可能会产生更大的性能影响。

要禁用对参数中的`None`值进行单独批处理的处理，请传递执行选项`render_nulls=True`；这将导致所有参数字典被等同对待，假设每个字典中都有相同的键：

```py
>>> session.execute(
...     insert(User).execution_options(render_nulls=True),
...     [
...         {
...             "name": "name_a",
...             "fullname": "Employee A",
...             "species": "Squid",
...         },
...         {
...             "name": "name_b",
...             "fullname": "Employee B",
...             "species": "Squirrel",
...         },
...         {
...             "name": "name_c",
...             "fullname": "Employee C",
...             "species": None,
...         },
...         {
...             "name": "name_d",
...             "fullname": "Employee D",
...             "species": "Bluefish",
...         },
...     ],
... )
INSERT  INTO  user_account  (name,  fullname,  species)  VALUES  (?,  ?,  ?)
[...]  [('name_a',  'Employee A',  'Squid'),  ('name_b',  'Employee B',  'Squirrel'),  ('name_c',  'Employee C',  None),  ('name_d',  'Employee D',  'Bluefish')]
... 
```

上面，所有参数字典都在单个插入批次中发送，包括第三个参数字典中的`None`值。

从版本 2.0.23 开始：添加了`render_nulls`执行选项，它镜像了传统`Session.bulk_insert_mappings.render_nulls`参数的行为。

### 批量插入联合表继承

ORM 批量插入建立在传统工作单元系统中使用的内部系统之上，以发出 INSERT 语句。这意味着对于映射到多个表的 ORM 实体，通常是使用联合表继承映射的实体，批量插入操作将为映射的每个表发出一个 INSERT 语句，将服务器生成的主键值正确传递给依赖于它们的表行。此外，这里还支持 RETURNING 功能，ORM 将接收每个执行的 INSERT 语句的`Result`对象，然后将它们“横向拼接”起来，以便返回的行包括插入的所有列的值：

```py
>>> managers = session.scalars(
...     insert(Manager).returning(Manager),
...     [
...         {"name": "sandy", "manager_name": "Sandy Cheeks"},
...         {"name": "ehkrabs", "manager_name": "Eugene H. Krabs"},
...     ],
... )
INSERT  INTO  employee  (name,  type)  VALUES  (?,  ?)  RETURNING  id,  name,  type
[...  (insertmanyvalues)  1/2  (ordered;  batch  not  supported)]  ('sandy',  'manager')
INSERT  INTO  employee  (name,  type)  VALUES  (?,  ?)  RETURNING  id,  name,  type
[insertmanyvalues  2/2  (ordered;  batch  not  supported)]  ('ehkrabs',  'manager')
INSERT  INTO  manager  (id,  manager_name)  VALUES  (?,  ?),  (?,  ?)  RETURNING  id,  manager_name,  id  AS  id__1
[...  (insertmanyvalues)  1/1  (ordered)]  (1,  'Sandy Cheeks',  2,  'Eugene H. Krabs') 
```

提示

批量插入联合继承映射要求 ORM 在内部使用`Insert.returning.sort_by_parameter_order`参数，以便它可以将 RETURNING 表中的主键值与用于插入“子”表的参数集相关联，这就是为什么上面的 SQLite 后端在透明地降级为使用非批处理语句的原因。关于此功能的背景信息，请参阅将 RETURNING 行与参数集相关联。

### 使用 SQL 表达式进行 ORM 批量插入

ORM 批量插入功能支持添加一组固定的参数，其中可以包括要应用于每个目标行的 SQL 表达式。为此，将使用 `Insert.values()` 方法，传递一个参数字典，该字典将应用于所有行，与通常的批量调用形式结合使用，方法是在调用 `Session.execute()` 时包含包含单独行值的参数字典列表。

例如，给定包括“timestamp”列的 ORM 映射：

```py
import datetime

class LogRecord(Base):
    __tablename__ = "log_record"
    id: Mapped[int] = mapped_column(primary_key=True)
    message: Mapped[str]
    code: Mapped[str]
    timestamp: Mapped[datetime.datetime]
```

如果我们想要插入一系列具有唯一 `message` 字段的 `LogRecord` 元素，但是我们希望将 SQL 函数 `now()` 应用于所有行，我们可以在 `Insert.values()` 中传递 `timestamp`，然后使用“批量”模式传递附加记录：

```py
>>> from sqlalchemy import func
>>> log_record_result = session.scalars(
...     insert(LogRecord).values(code="SQLA", timestamp=func.now()).returning(LogRecord),
...     [
...         {"message": "log message #1"},
...         {"message": "log message #2"},
...         {"message": "log message #3"},
...         {"message": "log message #4"},
...     ],
... )
INSERT  INTO  log_record  (message,  code,  timestamp)
VALUES  (?,  ?,  CURRENT_TIMESTAMP),  (?,  ?,  CURRENT_TIMESTAMP),
(?,  ?,  CURRENT_TIMESTAMP),  (?,  ?,  CURRENT_TIMESTAMP)
RETURNING  id,  message,  code,  timestamp
[...  (insertmanyvalues)  1/1  (unordered)]  ('log message #1',  'SQLA',  'log message #2',
'SQLA',  'log message #3',  'SQLA',  'log message #4',  'SQLA')
>>> print(log_record_result.all())
[LogRecord('log message #1', 'SQLA', datetime.datetime(...)),
 LogRecord('log message #2', 'SQLA', datetime.datetime(...)),
 LogRecord('log message #3', 'SQLA', datetime.datetime(...)),
 LogRecord('log message #4', 'SQLA', datetime.datetime(...))]
```

#### 通过每行 SQL 表达式进行 ORM 批量插入

`Insert.values()` 方法本身直接支持参数字典列表。当以这种方式使用 `Insert` 构造时，如果没有将参数字典列表传递给 `Session.execute.params` 参数，将不会使用批量 ORM 插入模式，而是将 INSERT 语句按原样呈现并精确调用一次。这种操作模式可能在按行基础上传递 SQL 表达式的情况下非常有用，并且在使用 ORM 进行“upsert”语句时也会使用，后文会在本章节中的 ORM “upsert” Statements 进行详细记录。

以下是嵌入每行 SQL 表达式的 INSERT 的人为示例，同时也演示了这种形式中的 `Insert.returning()`：

```py
>>> from sqlalchemy import select
>>> address_result = session.scalars(
...     insert(Address)
...     .values(
...         [
...             {
...                 "user_id": select(User.id).where(User.name == "sandy"),
...                 "email_address": "sandy@company.com",
...             },
...             {
...                 "user_id": select(User.id).where(User.name == "spongebob"),
...                 "email_address": "spongebob@company.com",
...             },
...             {
...                 "user_id": select(User.id).where(User.name == "patrick"),
...                 "email_address": "patrick@company.com",
...             },
...         ]
...     )
...     .returning(Address),
... )
INSERT  INTO  address  (user_id,  email_address)  VALUES
((SELECT  user_account.id
FROM  user_account
WHERE  user_account.name  =  ?),  ?),  ((SELECT  user_account.id
FROM  user_account
WHERE  user_account.name  =  ?),  ?),  ((SELECT  user_account.id
FROM  user_account
WHERE  user_account.name  =  ?),  ?)  RETURNING  id,  user_id,  email_address
[...]  ('sandy',  'sandy@company.com',  'spongebob',  'spongebob@company.com',
'patrick',  'patrick@company.com')
>>> print(address_result.all())
[Address(email_address='sandy@company.com'),
 Address(email_address='spongebob@company.com'),
 Address(email_address='patrick@company.com')]
```

由于上面未使用批量 ORM 插入模式，因此以下特性不可用：

+   不支持联接表继承或其他多表映射，因为那将需要多个 INSERT 语句。

+   不支持异构参数集 - VALUES 集合中的每个元素必须具有相同的列。

+   不提供诸如 insertmanyvalues 所提供的批处理等核心级别的规模优化；语句将需要确保参数总数不超过由后端数据库施加的限制。

出于上述原因，通常不建议在 ORM INSERT 语句中使用多个参数集合`Insert.values()`，除非有明确的理由，即要么使用了“upsert”，要么需要在每个参数集合中嵌入每行 SQL 表达式。

另请参阅

ORM“upsert”语句  #### 使用每行 SQL 表达式进行 ORM 批量插入

`Insert.values()`方法本身直接支持参数字典列表。当以这种方式使用`Insert`构造时，在不将任何参数字典列表传递给`Session.execute.params`参数的情况下，将不使用批量 ORM 插入模式，而是完全按照给定的方式呈现 INSERT 语句，并且仅调用一次。这种操作模式对于在每行基础上传递 SQL 表达式的情况可能很有用，并且在使用 ORM 时使用“upsert”语句时也会使用，后文将在本章的 ORM“upsert”语句中进行说明。

下面是一个虚构的示例，演示了嵌入每行 SQL 表达式的 INSERT，并以这种形式演示了`Insert.returning()`：

```py
>>> from sqlalchemy import select
>>> address_result = session.scalars(
...     insert(Address)
...     .values(
...         [
...             {
...                 "user_id": select(User.id).where(User.name == "sandy"),
...                 "email_address": "sandy@company.com",
...             },
...             {
...                 "user_id": select(User.id).where(User.name == "spongebob"),
...                 "email_address": "spongebob@company.com",
...             },
...             {
...                 "user_id": select(User.id).where(User.name == "patrick"),
...                 "email_address": "patrick@company.com",
...             },
...         ]
...     )
...     .returning(Address),
... )
INSERT  INTO  address  (user_id,  email_address)  VALUES
((SELECT  user_account.id
FROM  user_account
WHERE  user_account.name  =  ?),  ?),  ((SELECT  user_account.id
FROM  user_account
WHERE  user_account.name  =  ?),  ?),  ((SELECT  user_account.id
FROM  user_account
WHERE  user_account.name  =  ?),  ?)  RETURNING  id,  user_id,  email_address
[...]  ('sandy',  'sandy@company.com',  'spongebob',  'spongebob@company.com',
'patrick',  'patrick@company.com')
>>> print(address_result.all())
[Address(email_address='sandy@company.com'),
 Address(email_address='spongebob@company.com'),
 Address(email_address='patrick@company.com')]
```

由于上述未使用批量 ORM 插入模式，因此不存在以下功能：

+   连接表继承或其他多表映射不受支持，因为这将需要多个 INSERT 语句。

+   不支持异构参数集合 - VALUES 集合中的每个元素必须具有相同的列。

+   不可用核心级别的缩放优化，例如 insertmanyvalues 提供的批处理；语句需要确保参数的总数不超过由支持数据库施加的限制。

出于上述原因，通常不建议在 ORM INSERT 语句中使用多个参数集合`Insert.values()`，除非有明确的理由，即要么使用了“upsert”，要么需要在每个参数集合中嵌入每行 SQL 表达式。

另请参阅

ORM “upsert”语句

### 旧版会话批量插入方法

`Session`包括用于执行“批量”INSERT 和 UPDATE 语句的传统方法。这些方法与 SQLAlchemy 2.0 版本的这些特性共享实现，描述在 ORM 批量 INSERT 语句和 ORM 主键批量 UPDATE 中，但缺少许多功能，特别是缺少对 RETURNING 的支持以及对会话同步的支持。

使用`Session.bulk_insert_mappings()`的代码示例可以像下面这样移植代码，从这个映射示例开始：

```py
session.bulk_insert_mappings(User, [{"name": "u1"}, {"name": "u2"}, {"name": "u3"}])
```

以上是使用新 API 表达的：

```py
from sqlalchemy import insert

session.execute(insert(User), [{"name": "u1"}, {"name": "u2"}, {"name": "u3"}])
```

另请参阅

旧版会话批量 UPDATE 方法

### ORM “upsert” 语句

SQLAlchemy 的部分后端可能包含特定于方言的`Insert`构造，此外还具有执行“upserts”或将参数集中的现有行转换为近似 UPDATE 语句的能力。通过“现有行”，这可能意味着具有相同主键值的行，或者可能是指其他被认为是唯一的行中的索引列；这取决于正在使用的后端的功能。

SQLAlchemy 包含的方言特定“upsert”API 功能的方言包括：

+   SQLite - 使用`Insert`，在 INSERT…ON CONFLICT (Upsert)有详细说明。

+   PostgreSQL - 使用`Insert`，在 INSERT…ON CONFLICT (Upsert)有详细说明。

+   MySQL/MariaDB - 使用`Insert`，在 INSERT…ON DUPLICATE KEY UPDATE (Upsert)有详细说明。

用户应该查看上述部分了解这些对象的正确构造背景；特别是，“upsert”方法通常需要引用原始语句，因此语句通常是分两步构建的。

第三方后端，如在外部方言中提到的那些，也可能具有类似的构造。

虽然 SQLAlchemy 还没有与后端无关的 upsert 构造，但以上的`Insert`变体仍然与 ORM 兼容，因为它们可以像文档中记录的`Insert`构造本身一样使用，即通过在`Insert.values()`方法中嵌入要插入的行。在下面的示例中，使用 SQLite `insert()`函数生成一个包含“ON CONFLICT DO UPDATE”支持的`Insert`构造。然后，将语句传递给`Session.execute()`，它会按照正常流程进行，额外的特点是传递给`Insert.values()`的参数字典被解释为 ORM 映射的属性键，而不是列名。

```py
>>> from sqlalchemy.dialects.sqlite import insert as sqlite_upsert
>>> stmt = sqlite_upsert(User).values(
...     [
...         {"name": "spongebob", "fullname": "Spongebob Squarepants"},
...         {"name": "sandy", "fullname": "Sandy Cheeks"},
...         {"name": "patrick", "fullname": "Patrick Star"},
...         {"name": "squidward", "fullname": "Squidward Tentacles"},
...         {"name": "ehkrabs", "fullname": "Eugene H. Krabs"},
...     ]
... )
>>> stmt = stmt.on_conflict_do_update(
...     index_elements=[User.name], set_=dict(fullname=stmt.excluded.fullname)
... )
>>> session.execute(stmt)
INSERT  INTO  user_account  (name,  fullname)
VALUES  (?,  ?),  (?,  ?),  (?,  ?),  (?,  ?),  (?,  ?)
ON  CONFLICT  (name)  DO  UPDATE  SET  fullname  =  excluded.fullname
[...]  ('spongebob',  'Spongebob Squarepants',  'sandy',  'Sandy Cheeks',
'patrick',  'Patrick Star',  'squidward',  'Squidward Tentacles',
'ehkrabs',  'Eugene H. Krabs')
<...>
```

#### 使用 RETURNING 与 upsert 语句

从 SQLAlchemy ORM 的角度来看，upsert 语句看起来像是常规的`Insert`构造，其中包括`Insert.returning()`与 upsert 语句的工作方式相同，就像在 ORM 批量插入与每行 SQL 表达式中演示的那样，因此可以传递任何列表达式或相关的 ORM 实体类。继续上一节中的示例：

```py
>>> result = session.scalars(
...     stmt.returning(User), execution_options={"populate_existing": True}
... )
INSERT  INTO  user_account  (name,  fullname)
VALUES  (?,  ?),  (?,  ?),  (?,  ?),  (?,  ?),  (?,  ?)
ON  CONFLICT  (name)  DO  UPDATE  SET  fullname  =  excluded.fullname
RETURNING  id,  name,  fullname,  species
[...]  ('spongebob',  'Spongebob Squarepants',  'sandy',  'Sandy Cheeks',
'patrick',  'Patrick Star',  'squidward',  'Squidward Tentacles',
'ehkrabs',  'Eugene H. Krabs')
>>> print(result.all())
[User(name='spongebob', fullname='Spongebob Squarepants'),
 User(name='sandy', fullname='Sandy Cheeks'),
 User(name='patrick', fullname='Patrick Star'),
 User(name='squidward', fullname='Squidward Tentacles'),
 User(name='ehkrabs', fullname='Eugene H. Krabs')]
```

上面的示例使用 RETURNING 语句来返回每个被插入或合并的行的 ORM 对象。该示例还添加了对 现有数据的填充 执行选项的使用。该选项指示对于已经存在于 `Session` 中的行，应该使用新行的数据**刷新**`User`对象。对于纯粹的 `Insert` 语句，此选项不重要，因为每个生成的行都是全新的主键标识。但是，当 `Insert` 还包括“upsert”选项时，它可能也会产生已经存在的行的结果，因此可能已经在 `Session` 对象的身份映射中表示了主键标识。

另请参阅

使用 RETURNING 的 upsert 语句  #### 使用 RETURNING 语句的合并插入

从 SQLAlchemy ORM 的角度来看，upsert 语句看起来像是常规的 `Insert` 构造，这包括 `Insert.returning()` 在与示例 每行 SQL 表达式的 ORM 批量插入 中展示的方式上与 upsert 语句一样工作，因此可以传递任何列表达式或相关的 ORM 实体类。继续上一节中的示例：

```py
>>> result = session.scalars(
...     stmt.returning(User), execution_options={"populate_existing": True}
... )
INSERT  INTO  user_account  (name,  fullname)
VALUES  (?,  ?),  (?,  ?),  (?,  ?),  (?,  ?),  (?,  ?)
ON  CONFLICT  (name)  DO  UPDATE  SET  fullname  =  excluded.fullname
RETURNING  id,  name,  fullname,  species
[...]  ('spongebob',  'Spongebob Squarepants',  'sandy',  'Sandy Cheeks',
'patrick',  'Patrick Star',  'squidward',  'Squidward Tentacles',
'ehkrabs',  'Eugene H. Krabs')
>>> print(result.all())
[User(name='spongebob', fullname='Spongebob Squarepants'),
 User(name='sandy', fullname='Sandy Cheeks'),
 User(name='patrick', fullname='Patrick Star'),
 User(name='squidward', fullname='Squidward Tentacles'),
 User(name='ehkrabs', fullname='Eugene H. Krabs')]
```

上面的示例使用 RETURNING 语句来返回每个被插入或合并的行的 ORM 对象。该示例还添加了对 现有数据的填充 执行选项的使用。该选项指示对于已经存在于 `Session` 中的行，应该使用新行的数据**刷新**`User`对象。对于纯粹的 `Insert` 语句，此选项不重要，因为每个生成的行都是全新的主键标识。但是，当 `Insert` 还包括“upsert”选项时，它可能也会产生已经存在的行的结果，因此可能已经在 `Session` 对象的身份映射中表示了主键标识。

另请参阅

填充现有

## 根据主键进行 ORM 批量更新

`Update` 构造可以与 `Session.execute()` 一起使用，类似于描述的 `Insert` 语句在 ORM 批量插入语句 中的使用方式，传递许多参数字典的列表，每个字典代表一个对应于单个主键值的单独行。这种用法不应与更常见的使用 `Update` 语句与 ORM 一起使用的方式混淆，使用显式的 WHERE 子句，该方式在 ORM 更新和删除自定义 WHERE 条件 中有记录。

对于 UPDATE 的“批量”版本，将根据 ORM 类制作一个 `update()` 构造，并传递给 `Session.execute()` 方法；生成的 `Update` 对象应该**没有值，通常也没有 WHERE 条件**，也就是说，不使用 `Update.values()` 方法，通常也不使用 `Update.where()`，但在需要添加额外过滤条件的不寻常情况下可能会使用。

传递包含完整主键值的参数字典列表以及 `Update` 构造将调用**根据主键进行批量更新模式**的语句，生成适当的 WHERE 条件以匹配每个主键的行，并使用 executemany 对 UPDATE 语句运行每个参数集：

```py
>>> from sqlalchemy import update
>>> session.execute(
...     update(User),
...     [
...         {"id": 1, "fullname": "Spongebob Squarepants"},
...         {"id": 3, "fullname": "Patrick Star"},
...         {"id": 5, "fullname": "Eugene H. Krabs"},
...     ],
... )
UPDATE  user_account  SET  fullname=?  WHERE  user_account.id  =  ?
[...]  [('Spongebob Squarepants',  1),  ('Patrick Star',  3),  ('Eugene H. Krabs',  5)]
<...>
```

请注意，每个参数字典**必须包含每个记录的完整主键**，否则会引发错误。

像批量插入功能一样，这里也支持异构参数列表，其中参数将被分组为更新运行的子批次。

在 2.0.11 版本中更改：可以使用`Update.where()`方法添加额外的 WHERE 条件与 ORM 按主键批量更新相结合。但是，此条件始终是额外添加的，这包括主键值。

在使用“按主键批量更新”功能时，不支持 RETURNING 功能；多个参数字典列表必须使用 DBAPI 的 executemany，通常情况下不支持结果行。

在 2.0 版本中更改：将`Update`构造传递给`Session.execute()`方法，以及参数字典列表，现在会调用“批量更新”，这与传统的`Session.bulk_update_mappings()`方法使用相同的功能。这是与 1.x 系列不同的行为更改，1.x 系列中的`Update`只支持显式的 WHERE 条件和内联 VALUES。

### 禁用多参数集 UPDATE 语句的按主键批量 ORM 更新

当满足以下条件时，自动使用 ORM 按主键批量更新功能：

1.  给定的 UPDATE 语句针对的是 ORM 实体。

1.  使用`Session`执行语句，而不是核心`Connection`

1.  传递的参数是**字典列表**。

为了在不使用“ORM 按主键批量更新”功能的情况下调用 UPDATE 语句，请直接针对`Connection`使用`Session.connection()`方法来获取当前事务的`Connection`：

```py
>>> from sqlalchemy import bindparam
>>> session.connection().execute(
...     update(User).where(User.name == bindparam("u_name")),
...     [
...         {"u_name": "spongebob", "fullname": "Spongebob Squarepants"},
...         {"u_name": "patrick", "fullname": "Patrick Star"},
...     ],
... )
UPDATE  user_account  SET  fullname=?  WHERE  user_account.name  =  ?
[...]  [('Spongebob Squarepants',  'spongebob'),  ('Patrick Star',  'patrick')]
<...>
```

另请参阅

按行 ORM 按主键批量更新需要记录包含主键值  ### 按主键批量更新联合表继承

当使用具有联合表继承的映射时，ORM 批量更新与 ORM 批量插入具有类似的行为；如在 Bulk INSERT for Joined Table Inheritance 中所述，批量 UPDATE 操作将为映射中表示的每个表发出 UPDATE 语句，其中给定的参数包括要更新的值（不受影响的表将被跳过）。

示例：

```py
>>> session.execute(
...     update(Manager),
...     [
...         {
...             "id": 1,
...             "name": "scheeks",
...             "manager_name": "Sandy Cheeks, President",
...         },
...         {
...             "id": 2,
...             "name": "eugene",
...             "manager_name": "Eugene H. Krabs, VP Marketing",
...         },
...     ],
... )
UPDATE  employee  SET  name=?  WHERE  employee.id  =  ?
[...]  [('scheeks',  1),  ('eugene',  2)]
UPDATE  manager  SET  manager_name=?  WHERE  manager.id  =  ?
[...]  [('Sandy Cheeks, President',  1),  ('Eugene H. Krabs, VP Marketing',  2)]
<...>
```  ### 旧版 Session 批量更新方法

如在旧版 Session 批量插入方法中讨论的那样，`Session.bulk_update_mappings()`方法是批量更新的旧版形式，ORM 在解释给定带有主键参数的`update()`语句时内部使用；但是，当使用旧版时，不包括诸如会话同步支持等功能。

下面的示例：

```py
session.bulk_update_mappings(
 User,
 [
 {"id": 1, "name": "scheeks", "manager_name": "Sandy Cheeks, President"},
 {"id": 2, "name": "eugene", "manager_name": "Eugene H. Krabs, VP Marketing"},
 ],
)
```

使用新 API 表示为：

```py
from sqlalchemy import update

session.execute(
 update(User),
 [
 {"id": 1, "name": "scheeks", "manager_name": "Sandy Cheeks, President"},
 {"id": 2, "name": "eugene", "manager_name": "Eugene H. Krabs, VP Marketing"},
 ],
)
```

另请参阅

旧版 Session 批量插入方法  ### 禁用 UPDATE 语句的多参数集的基于主键的批量 ORM 更新

当满足以下条件时，会自动使用基于主键的 ORM 批量更新功能，该功能对每个包含主键值的记录运行 UPDATE 语句，并包括每个主键值的 WHERE 条件：

1.  给定的 UPDATE 语句针对一个 ORM 实体

1.  使用`Session`执行该语句，而不是使用核心`Connection`

1.  传递的参数是**字典列表**。

为了调用 UPDATE 语句而不使用“基于主键的 ORM 批量更新”，直接使用`Session.connection()`方法针对当前事务获取`Connection`：

```py
>>> from sqlalchemy import bindparam
>>> session.connection().execute(
...     update(User).where(User.name == bindparam("u_name")),
...     [
...         {"u_name": "spongebob", "fullname": "Spongebob Squarepants"},
...         {"u_name": "patrick", "fullname": "Patrick Star"},
...     ],
... )
UPDATE  user_account  SET  fullname=?  WHERE  user_account.name  =  ?
[...]  [('Spongebob Squarepants',  'spongebob'),  ('Patrick Star',  'patrick')]
<...>
```

另请参阅

基于主键的逐行 ORM 批量更新要求记录包含主键值

### 基于主键的联合表继承批量更新

ORM 批量更新在使用具有联合表继承的映射时与 ORM 批量插入具有相似的行为；正如联合表继承的批量插入中所描述的，批量更新操作将为映射中表示的每个表发出一个更新语句，其中给定的参数包括要更新的值（未受影响的表将被跳过）。

示例：

```py
>>> session.execute(
...     update(Manager),
...     [
...         {
...             "id": 1,
...             "name": "scheeks",
...             "manager_name": "Sandy Cheeks, President",
...         },
...         {
...             "id": 2,
...             "name": "eugene",
...             "manager_name": "Eugene H. Krabs, VP Marketing",
...         },
...     ],
... )
UPDATE  employee  SET  name=?  WHERE  employee.id  =  ?
[...]  [('scheeks',  1),  ('eugene',  2)]
UPDATE  manager  SET  manager_name=?  WHERE  manager.id  =  ?
[...]  [('Sandy Cheeks, President',  1),  ('Eugene H. Krabs, VP Marketing',  2)]
<...>
```

### 旧版会话批量更新方法

如旧版会话批量插入方法中所讨论的，`Session.bulk_update_mappings()` 方法是批量更新的旧式形式，当给定主键参数时，ORM 在解释 `update()` 语句时内部使用它；然而，当使用旧版时，诸如会话同步支持之类的功能将不包括在内。

下面的示例：

```py
session.bulk_update_mappings(
 User,
 [
 {"id": 1, "name": "scheeks", "manager_name": "Sandy Cheeks, President"},
 {"id": 2, "name": "eugene", "manager_name": "Eugene H. Krabs, VP Marketing"},
 ],
)
```

使用新 API 表达为：

```py
from sqlalchemy import update

session.execute(
 update(User),
 [
 {"id": 1, "name": "scheeks", "manager_name": "Sandy Cheeks, President"},
 {"id": 2, "name": "eugene", "manager_name": "Eugene H. Krabs, VP Marketing"},
 ],
)
```

另请参阅

旧版会话批量插入方法

## 使用自定义 WHERE 条件的 ORM 更新和删除

当使用自定义 WHERE 条件构建 `Update` 和 `Delete` 构造时（即使用 `Update.where()` 和 `Delete.where()` 方法），可以通过将它们传递给 `Session.execute()` 在 ORM 上下文中调用它们，而不使用 `Session.execute.params` 参数。对于 `Update`，应该使用 `Update.values()` 传递要更新的值。

这种使用方式与之前描述的 ORM 按主键批量更新中的功能不同，ORM 使用给定的 WHERE 子句如所示，而不是将 WHERE 子句修复为按主键。这意味着单个 UPDATE 或 DELETE 语句可以一次性影响许多行。

举个例子，下面发出了一个更新，影响了多行的“fullname”字段

```py
>>> from sqlalchemy import update
>>> stmt = (
...     update(User)
...     .where(User.name.in_(["squidward", "sandy"]))
...     .values(fullname="Name starts with S")
... )
>>> session.execute(stmt)
UPDATE  user_account  SET  fullname=?  WHERE  user_account.name  IN  (?,  ?)
[...]  ('Name starts with S',  'squidward',  'sandy')
<...>
```

对于 DELETE，基于条件删除行的示例：

```py
>>> from sqlalchemy import delete
>>> stmt = delete(User).where(User.name.in_(["squidward", "sandy"]))
>>> session.execute(stmt)
DELETE  FROM  user_account  WHERE  user_account.name  IN  (?,  ?)
[...]  ('squidward',  'sandy')
<...>
```

警告

请阅读以下部分 Important Notes and Caveats for ORM-Enabled Update and Delete，了解关于 ORM 启用的 UPDATE 和 DELETE 功能与 ORM 工作单元 功能的功能差异的重要说明，例如使用 `Session.delete()` 方法删除单个对象。

### 关于 ORM 启用的更新和删除的重要说明和注意事项

ORM 启用的 UPDATE 和 DELETE 功能绕过 ORM 工作单元 自动化，以便能够发出一条匹配多行的单个 UPDATE 或 DELETE 语句，而不会增加复杂性。

+   操作不提供 Python 中的关系级联 - 假定对于需要它的任何外键引用已配置了 ON UPDATE CASCADE 和/或 ON DELETE CASCADE，否则如果正在执行外键引用，则数据库可能会发出完整性违规。请参阅 Using foreign key ON DELETE cascade with ORM relationships 中的说明，了解一些示例。

+   在 UPDATE 或 DELETE 之后，受到与相关表上的 ON UPDATE CASCADE 或 ON DELETE CASCADE 相关的影响的`Session`中的依赖对象，特别是引用现在已被删除的行的对象，可能仍然引用这些对象。一旦`Session`过期，通常发生在 `Session.commit()` 或可以通过使用 `Session.expire_all()` 强制进行。

+   启用 ORM 的 UPDATE 和 DELETE 操作不会自动处理连接表继承。请参阅 UPDATE/DELETE with Custom WHERE Criteria for Joined Table Inheritance 部分，了解如何处理连接继承映射。

+   为了将多态标识限制为单表继承映射的特定子类，**自动包含了** WHERE 条件。这仅适用于没有自己的表的子类映射。

+   `with_loader_criteria()` 选项 **受支持** ，可用于 ORM 更新和删除操作；此处的条件将添加到正在发出的 UPDATE 或 DELETE 语句的条件中，并在“同步”过程中考虑到。

+   为了使用事件处理程序拦截启用 ORM 的 UPDATE 和 DELETE 操作，请使用`SessionEvents.do_orm_execute()`事件。### 选择同步策略

当结合使用`update()`或`delete()`与启用 ORM 的执行时，使用`Session.execute()`，还会出现额外的 ORM 特定功能，该功能将**同步**语句更改的状态与当前存在于`Session`的身份映射中的对象的状态。我们所说的“同步”是指，更新的属性将使用新值刷新，或者至少会过期，以便它们在下一次访问时重新填充其新值，并且删除的对象将移动到已删除状态。

这种同步是可控的，作为“同步策略”，传递为字符串 ORM 执行选项，通常使用`Session.execute.execution_options`字典：

```py
>>> from sqlalchemy import update
>>> stmt = (
...     update(User).where(User.name == "squidward").values(fullname="Squidward Tentacles")
... )
>>> session.execute(stmt, execution_options={"synchronize_session": False})
UPDATE  user_account  SET  fullname=?  WHERE  user_account.name  =  ?
[...]  ('Squidward Tentacles',  'squidward')
<...>
```

执行选项也可以与语句本身捆绑在一起，使用`Executable.execution_options()`方法：

```py
>>> from sqlalchemy import update
>>> stmt = (
...     update(User)
...     .where(User.name == "squidward")
...     .values(fullname="Squidward Tentacles")
...     .execution_options(synchronize_session=False)
... )
>>> session.execute(stmt)
UPDATE  user_account  SET  fullname=?  WHERE  user_account.name  =  ?
[...]  ('Squidward Tentacles',  'squidward')
<...>
```

支持`synchronize_session`的以下值：

+   `'auto'` - 这是默认值。对于支持 RETURNING 的后端将使用`'fetch'`策略，这包括除 MySQL 之外的所有 SQLAlchemy 本机驱动程序。如果不支持 RETURNING，则将改用`'evaluate'`策略。

+   `'fetch'` - 通过在执行 UPDATE 或 DELETE 之前执行 SELECT 或通过使用数据库支持的 RETURNING 来检索受影响行的主键标识，以便可以刷新受操作影响的内存中的对象（更新）或从 `Session` 中清除（删除）。即使给定的 `update()` 或 `delete()` 构造显式指定了使用 `UpdateBase.returning()` 的实体或列，也可以使用此同步策略。

    从版本 2.0 开始更改：在使用 ORM 启用的 UPDATE 和 DELETE 与 WHERE 条件时，可以将显式的 `UpdateBase.returning()` 与 `'fetch'` 同步策略结合使用。实际语句将包含 `'fetch'` 策略所需的列与请求的列之间的并集。

+   `'evaluate'` - 这表示在 Python 中评估 UPDATE 或 DELETE 语句中给定的 WHERE 条件，以定位 `Session` 中的匹配对象。该方法不会增加任何 SQL 往返到操作中，在没有 RETURNING 支持的情况下，可能更有效。对于具有复杂条件的 UPDATE 或 DELETE 语句，`'evaluate'` 策略可能无法在 Python 中评估表达式并将引发错误。如果发生这种情况，请改用操作的 `'fetch'` 策略。

    提示

    如果 SQL 表达式使用 `Operators.op()` 或 `custom_op` 功能使用自定义运算符，则可以使用 `Operators.op.python_impl` 参数指示将由 `"evaluate"` 同步策略使用的 Python 函数。

    从版本 2.0 开始新增。

    警告

    如果要在已过期的 `Session` 上运行 UPDATE 操作，则应避免使用 `"evaluate"` 策略，因为它必然需要刷新对象以测试它们是否符合给定的 WHERE 条件，这将为每个对象发出一个 SELECT。在这种情况下，特别是如果后端支持 RETURNING，则应首选 `"fetch"` 策略。

+   `False` - 不同步会话。此选项对于不支持 RETURNING 的后端可能很有用，其中无法使用 `"evaluate"` 策略。在这种情况下，`Session` 中对象的状态不变，不会自动与生成的 UPDATE 或 DELETE 语句相对应，如果存在通常与匹配的行相对应的对象。 ### 使用 RETURNING 进行 UPDATE/DELETE 和自定义 WHERE 条件

`UpdateBase.returning()` 方法与启用了 ORM 的带有 WHERE 条件的 UPDATE 和 DELETE 完全兼容。可以指定完整的 ORM 对象和/或列用于 RETURNING：

```py
>>> from sqlalchemy import update
>>> stmt = (
...     update(User)
...     .where(User.name == "squidward")
...     .values(fullname="Squidward Tentacles")
...     .returning(User)
... )
>>> result = session.scalars(stmt)
UPDATE  user_account  SET  fullname=?  WHERE  user_account.name  =  ?
RETURNING  id,  name,  fullname,  species
[...]  ('Squidward Tentacles',  'squidward')
>>> print(result.all())
[User(name='squidward', fullname='Squidward Tentacles')]
```

对 RETURNING 的支持也与 `fetch` 同步策略兼容，该策略也使用 RETURNING。ORM 将适当地组织 RETURNING 中的列，以便同步进行，以及返回的`Result`将按请求的顺序包含请求的实体和 SQL 列。

版本 2.0 中的新功能：`UpdateBase.returning()` 可用于启用 ORM 的 UPDATE 和 DELETE，同时保留与 `fetch` 同步策略完全兼容。 ### 使用自定义 WHERE 条件进行联接表继承的 UPDATE/DELETE

与基于主键的 ORM 批量 UPDATE 不同，带有 WHERE 条件的 UPDATE/DELETE 功能在每次调用`Session.execute()`时仅生成单个 UPDATE 或 DELETE 语句。这意味着当运行对多表映射进行`update()`或`delete()`操作时，例如联接表继承映射中的子类时，语句必须符合后端当前的能力，这可能包括后端不支持涉及多个表的 UPDATE 或 DELETE 语句，或者对此仅有限支持。这意味着对于诸如联接继承子类之类的映射，带有 WHERE 条件的 ORM 版本 UPDATE/DELETE 功能只能在一定程度上或根本不能使用，具体取决于具体情况。

发出联合表子类的多行 UPDATE 语句的最直接方法是仅引用子表。这意味着`Update()` 构造应该仅引用子类表本地的属性，如下例所示：

```py
>>> stmt = (
...     update(Manager)
...     .where(Manager.id == 1)
...     .values(manager_name="Sandy Cheeks, President")
... )
>>> session.execute(stmt)
UPDATE  manager  SET  manager_name=?  WHERE  manager.id  =  ?
[...]  ('Sandy Cheeks, President',  1)
<...> 
```

使用上述形式，一个简单的引用基本表以定位将在任何 SQL 后端上工作的行的方法是使用子查询：

```py
>>> stmt = (
...     update(Manager)
...     .where(
...         Manager.id
...         == select(Employee.id).where(Employee.name == "sandy").scalar_subquery()
...     )
...     .values(manager_name="Sandy Cheeks, President")
... )
>>> session.execute(stmt)
UPDATE  manager  SET  manager_name=?  WHERE  manager.id  =  (SELECT  employee.id
FROM  employee
WHERE  employee.name  =  ?)  RETURNING  id
[...]  ('Sandy Cheeks, President',  'sandy')
<...>
```

对于支持 UPDATE…FROM 的后端，子查询可以作为额外的普通 WHERE 条件来陈述，但是两个表之间的条件必须以某种方式明确陈述：

```py
>>> stmt = (
...     update(Manager)
...     .where(Manager.id == Employee.id, Employee.name == "sandy")
...     .values(manager_name="Sandy Cheeks, President")
... )
>>> session.execute(stmt)
UPDATE  manager  SET  manager_name=?  FROM  employee
WHERE  manager.id  =  employee.id  AND  employee.name  =  ?
[...]  ('Sandy Cheeks, President',  'sandy')
<...>
```

对于 DELETE，预期基表和子表中的行将同时被 DELETE。要删除多行联合继承对象，**而不使用**级联外键，请分别对每个表发出 DELETE：

```py
>>> from sqlalchemy import delete
>>> session.execute(delete(Manager).where(Manager.id == 1))
DELETE  FROM  manager  WHERE  manager.id  =  ?
[...]  (1,)
<...>
>>> session.execute(delete(Employee).where(Employee.id == 1))
DELETE  FROM  employee  WHERE  employee.id  =  ?
[...]  (1,)
<...>
```

总的来说，应该**优先选择**常规的 unit of work 流程来更新和删除联合继承和其他多表映射的行，除非有使用自定义 WHERE 条件的性能原因。

### 旧版查询方法

启用 ORM 的 UPDATE/DELETE 与 WHERE 功能最初是作为现在已过时的`Query`对象的一部分，出现在`Query.update()` 和 `Query.delete()` 方法中。这些方法仍然可用，并且提供与 ORM UPDATE 和 DELETE 与自定义 WHERE 条件中描述的相同功能的子集。主要区别在于旧版方法不提供显式的 RETURNING 支持。

另请参阅

`Query.update()`

`Query.delete()`

### ORM-启用的更新和删除的重要注意事项和警告

启用 ORM 的 UPDATE 和 DELETE 功能绕过 ORM 的 unit of work 自动化，而是支持发出一条单独的 UPDATE 或 DELETE 语句，一次匹配多行，而不复杂。

+   这些操作不提供 Python 中关系的级联 - 假设对于需要的任何外键引用配置了 ON UPDATE CASCADE 和/或 ON DELETE CASCADE，否则如果正在强制执行外键引用，则数据库可能会发出完整性违规。请参阅使用 ORM 关系的外键 ON DELETE 级联中的说明以获取一些示例。

+   在 UPDATE 或 DELETE 之后，`Session`中的依赖对象，受到相关表上的 ON UPDATE CASCADE 或 ON DELETE CASCADE 的影响，特别是那些引用现已被删除的行的对象，可能仍然引用这些对象。一旦`Session`过期，这个问题就会得到解决，通常发生在`Session.commit()`时，或者可以通过使用`Session.expire_all()`来强制实现。

+   启用 ORM 的 UPDATE 和 DELETE 操作不会自动处理连接表继承。请参阅 UPDATE/DELETE with Custom WHERE Criteria for Joined Table Inheritance 部分，了解如何处理连接继承映射。

+   为了限制多态标识仅适用于单表继承映射中的特定子类，**WHERE 条件会自动包含**。这仅适用于没有自己的表的子类映射器。

+   `with_loader_criteria()`选项**支持**ORM 更新和删除操作；这里的条件将被添加到正在发出的 UPDATE 或 DELETE 语句的条件中，并在“同步”过程中考虑到这些条件。

+   要拦截 ORM 启用的 UPDATE 和 DELETE 操作以使用事件处理程序，请使用`SessionEvents.do_orm_execute()`事件。

### 选择同步策略

当使用`update()`或`delete()`与 ORM 启用的执行结合使用时，还存在额外的 ORM 特定功能，将会**同步**语句所更改的状态与当前存在于`Session`的标识映射中的对象状态。通过“同步”，我们指的是 UPDATE 的属性将使用新值刷新，或者至少过期，以便在下次访问时重新填充为新值，而 DELETE 的对象将移至删除状态。

此同步可通过“同步策略”来控制，该策略作为字符串 ORM 执行选项传递，通常通过使用 `Session.execute.execution_options` 字典：

```py
>>> from sqlalchemy import update
>>> stmt = (
...     update(User).where(User.name == "squidward").values(fullname="Squidward Tentacles")
... )
>>> session.execute(stmt, execution_options={"synchronize_session": False})
UPDATE  user_account  SET  fullname=?  WHERE  user_account.name  =  ?
[...]  ('Squidward Tentacles',  'squidward')
<...>
```

执行选项也可以与语句本身捆绑在一起，使用 `Executable.execution_options()` 方法：

```py
>>> from sqlalchemy import update
>>> stmt = (
...     update(User)
...     .where(User.name == "squidward")
...     .values(fullname="Squidward Tentacles")
...     .execution_options(synchronize_session=False)
... )
>>> session.execute(stmt)
UPDATE  user_account  SET  fullname=?  WHERE  user_account.name  =  ?
[...]  ('Squidward Tentacles',  'squidward')
<...>
```

支持以下值作为 `synchronize_session`：

+   `'auto'` - 这是默认设置。在支持 RETURNING 的后端上将使用 `'fetch'` 策略，这包括除 MySQL 外的所有 SQLAlchemy 本机驱动程序。如果不支持 RETURNING，则将使用 `'evaluate'` 策略。

+   `'fetch'` - 通过在执行 UPDATE 或 DELETE 之前执行 SELECT 或使用 RETURNING（如果数据库支持），检索受影响行的主键标识，以便可以使用新值刷新受操作影响的内存对象（更新），或者从 `Session` 中清除（删除）。即使给定的 `update()` 或 `delete()` 构造明确指定了使用 `UpdateBase.returning()` 的实体或列，也可以使用此同步策略。

    从版本 2.0 开始更改：当使用启用 ORM 的 UPDATE 和 DELETE 与 WHERE 条件时，可以将显式的 `UpdateBase.returning()` 与 `'fetch'` 同步策略结合使用。实际语句将包含 `'fetch'` 策略所需的列和请求的列之间的并集。

+   `'evaluate'` - 这表示在 Python 中评估 UPDATE 或 DELETE 语句中给定的 WHERE 条件，以在 `Session` 中定位匹配的对象。此方法不会为操作添加任何 SQL 往返，并且在没有 RETURNING 支持的情况下，可能更有效。对于具有复杂条件的 UPDATE 或 DELETE 语句，`'evaluate'` 策略可能无法在 Python 中评估表达式，并将引发错误。如果发生这种情况，请改为使用 `'fetch'` 策略执行操作。

    提示

    如果 SQL 表达式使用 `Operators.op()` 或 `custom_op` 特性使用自定义运算符，则可以使用 `Operators.op.python_impl` 参数指示一个 Python 函数，该函数将由 `"evaluate"` 同步策略使用。

    2.0 版本中新增。

    警告

    如果要在具有许多已过期对象的 `Session` 上运行 UPDATE 操作，则应避免使用 `"evaluate"` 策略，因为它必然需要刷新对象以便根据给定的 WHERE 条件测试它们，这将为每个对象发出一个 SELECT。在这种情况下，特别是如果后端支持 RETURNING，则应首选 `"fetch"` 策略。

+   `False` - 不同步会话。该选项对于不支持 RETURNING 的后端可能很有用，其中无法使用 `"evaluate"` 策略。在这种情况下，`Session` 中的对象状态保持不变，并且不会自动与发出的 UPDATE 或 DELETE 语句对应，如果存在通常会与匹配行对应的对象。

### 使用 RETURNING 进行 UPDATE/DELETE 和自定义 WHERE 条件

`UpdateBase.returning()` 方法与启用 ORM 的带有 WHERE 条件的 UPDATE 和 DELETE 完全兼容。可以指定完整的 ORM 对象和/或列来进行 RETURNING：

```py
>>> from sqlalchemy import update
>>> stmt = (
...     update(User)
...     .where(User.name == "squidward")
...     .values(fullname="Squidward Tentacles")
...     .returning(User)
... )
>>> result = session.scalars(stmt)
UPDATE  user_account  SET  fullname=?  WHERE  user_account.name  =  ?
RETURNING  id,  name,  fullname,  species
[...]  ('Squidward Tentacles',  'squidward')
>>> print(result.all())
[User(name='squidward', fullname='Squidward Tentacles')]
```

RETURNING 的支持也与 `fetch` 同步策略兼容，该策略也使用 RETURNING。ORM 将适当地组织 RETURNING 中的列，以使同步进行得很好，并且返回的 `Result` 将按请求的顺序包含请求的实体和 SQL 列。

2.0 版本中新增：`UpdateBase.returning()` 可用于启用 ORM 的 UPDATE 和 DELETE，同时仍保留与 `fetch` 同步策略的完全兼容性。

### 用于联接表继承的 UPDATE/DELETE 自定义 WHERE 条件

与 ORM Bulk UPDATE by Primary Key 不同，具有 WHERE 条件的 UPDATE/DELETE 功能在每次调用`Session.execute()`时仅发出单个 UPDATE 或 DELETE 语句。这意味着当针对多表映射（如联接表继承映射中的子类）运行`update()`或`delete()`语句时，语句必须符合后端的当前能力，这可能包括后端不支持引用多个表的 UPDATE 或 DELETE 语句，或者仅对此提供有限的支持。这意味着对于诸如联接继承子类之类的映射，ORM 版本的具有 WHERE 条件的 UPDATE/DELETE 功能仅能在有限程度上或根据具体情况根本无法使用。

最直接的方法是为联接表子类发出多行更新语句，只需引用子表即可。这意味着`Update()`构造应仅引用子类表本地的属性，如下例所示：

```py
>>> stmt = (
...     update(Manager)
...     .where(Manager.id == 1)
...     .values(manager_name="Sandy Cheeks, President")
... )
>>> session.execute(stmt)
UPDATE  manager  SET  manager_name=?  WHERE  manager.id  =  ?
[...]  ('Sandy Cheeks, President',  1)
<...> 
```

使用上述形式，一种简单的引用基表以定位行的方法，可以在任何 SQL 后端上工作，即使用子查询：

```py
>>> stmt = (
...     update(Manager)
...     .where(
...         Manager.id
...         == select(Employee.id).where(Employee.name == "sandy").scalar_subquery()
...     )
...     .values(manager_name="Sandy Cheeks, President")
... )
>>> session.execute(stmt)
UPDATE  manager  SET  manager_name=?  WHERE  manager.id  =  (SELECT  employee.id
FROM  employee
WHERE  employee.name  =  ?)  RETURNING  id
[...]  ('Sandy Cheeks, President',  'sandy')
<...>
```

对于支持 UPDATE…FROM 的后端，子查询可以改为额外的纯 WHERE 条件，但是两个表之间的条件必须以某种方式明确说明：

```py
>>> stmt = (
...     update(Manager)
...     .where(Manager.id == Employee.id, Employee.name == "sandy")
...     .values(manager_name="Sandy Cheeks, President")
... )
>>> session.execute(stmt)
UPDATE  manager  SET  manager_name=?  FROM  employee
WHERE  manager.id  =  employee.id  AND  employee.name  =  ?
[...]  ('Sandy Cheeks, President',  'sandy')
<...>
```

对于 DELETE 操作，预期基表和子表中的行将同时被删除。要删除多行联接继承对象而**不使用**级联外键，需分别为每个表发出 DELETE 语句：

```py
>>> from sqlalchemy import delete
>>> session.execute(delete(Manager).where(Manager.id == 1))
DELETE  FROM  manager  WHERE  manager.id  =  ?
[...]  (1,)
<...>
>>> session.execute(delete(Employee).where(Employee.id == 1))
DELETE  FROM  employee  WHERE  employee.id  =  ?
[...]  (1,)
<...>
```

总体而言，通常应**优先选择**普通的工作单元流程来更新和删除联接继承和其他多表映射的行，除非使用自定义的 WHERE 条件有性能上的理由。

### 旧式查询方法

原始`Query`对象的 ORM 启用的 UPDATE/DELETE with WHERE 功能最初是`Query.update()`和`Query.delete()`方法的一部分。这些方法仍然可用，并且提供与 ORM UPDATE and DELETE with Custom WHERE Criteria 描述的相同功能的子集。主要区别在于旧式方法不支持显式的 RETURNING 支持。

另请参见

`Query.update()`

`Query.delete()`
