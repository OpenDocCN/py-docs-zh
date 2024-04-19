# 配置版本计数器

> 原文：[`docs.sqlalchemy.org/en/20/orm/versioning.html`](https://docs.sqlalchemy.org/en/20/orm/versioning.html)

`Mapper`支持管理版本 ID 列，它是单个表列，每当对映射表进行`UPDATE`时，该列会递增或以其他方式更新其值。每次 ORM 发出`UPDATE`或`DELETE`对行进行操作时，都会检查该值，以确保内存中持有的值与数据库值匹配。

警告

因为版本控制功能依赖于对象的**内存**记录的比较，所以该功能仅适用于`Session.flush()`过程，在此过程中 ORM 将单个内存中的行刷新到数据库。当执行多行 UPDATE 或 DELETE 时，该功能不会生效，使用`Query.update()`或`Query.delete()`方法，因为这些方法仅发出 UPDATE 或 DELETE 语句，但否则无法直接访问受影响行的内容。

此功能的目的是检测两个并发事务在大致相同的时间修改同一行，或者在可能重用上一个事务的数据而不进行刷新的系统中提供对“过时”行的防护（例如，如果使用`Session`的`expire_on_commit=False`设置，可能会重用上一个事务的数据）。

## 简单版本计数

跟踪版本的最直接方法是向映射表添加一个整数列，然后在映射器选项中将其设为`version_id_col`：

```py
class User(Base):
    __tablename__ = "user"

    id = mapped_column(Integer, primary_key=True)
    version_id = mapped_column(Integer, nullable=False)
    name = mapped_column(String(50), nullable=False)

    __mapper_args__ = {"version_id_col": version_id}
```

注意

**强烈建议**将`version_id`列设为 NOT NULL。版本控制功能**不支持**版本列中的 NULL 值。

在上面的例子中，`User`映射使用`version_id`列跟踪整数版本。当首次刷新`User`类型的对象时，`version_id`列的值将为“1”。然后，稍后对表的 UPDATE 将始终以类似以下的方式发出：

```py
UPDATE  user  SET  version_id=:version_id,  name=:name
WHERE  user.id  =  :user_id  AND  user.version_id  =  :user_version_id
-- {"name": "new name", "version_id": 2, "user_id": 1, "user_version_id": 1}
```

上述 UPDATE 语句正在更新不仅与`user.id = 1`匹配的行，而且还要求`user.version_id = 1`，其中“1”是我们已知在此对象上使用的最后版本标识符。如果在其他地方的事务独立修改了该行，则此版本 id 将不再匹配，并且 UPDATE 语句将报告没有匹配的行；这是 SQLAlchemy 测试的条件，确保我们的 UPDATE（或 DELETE）语句匹配了恰好一行。如果没有匹配的行，这表明我们的数据版本已过时，并且会引发`StaleDataError`异常。

## 自定义版本计数器/类型

可以使用其他类型或计数器来进行版本控制。常见类型包括日期和 GUID。当使用备用类型或计数器方案时，SQLAlchemy 提供了使用`version_id_generator`参数的钩子，该参数接受一个版本生成可调用对象。此可调用对象会传递当前已知版本的值，并且预期返回后续版本。

例如，如果我们想使用随机生成的 GUID 跟踪`User`类的版本控制，我们可以这样做（请注意，某些后端支持原生 GUID 类型，但我们在这里使用简单的字符串进行说明）：

```py
import uuid

class User(Base):
    __tablename__ = "user"

    id = mapped_column(Integer, primary_key=True)
    version_uuid = mapped_column(String(32), nullable=False)
    name = mapped_column(String(50), nullable=False)

    __mapper_args__ = {
        "version_id_col": version_uuid,
        "version_id_generator": lambda version: uuid.uuid4().hex,
    }
```

每次`User`对象需要进行 INSERT 或 UPDATE 操作时，持久化引擎将调用`uuid.uuid4()`。在这种情况下，我们的版本生成函数可以忽略`version`的传入值，因为`uuid4()`函数生成的标识符不需要任何先决条件值。如果我们使用的是顺序版本控制方案，例如数字或特殊字符系统，我们可以利用给定的`version`来帮助确定后续值。

另请参阅

跨后端通用 GUID 类型  ## 服务器端版本计数器

`version_id_generator`也可以配置为依赖于数据库生成的值。在这种情况下，数据库需要某种方式在行进行 INSERT 时生成新的标识符，以及在 UPDATE 时生成。对于 UPDATE 情况，通常需要一个更新触发器，除非所涉及的数据库支持其他一些本地版本标识符。特别是 PostgreSQL 数据库支持一个称为[xmin](https://www.postgresql.org/docs/current/static/ddl-system-columns.html)的系统列，它提供了 UPDATE 版本控制。我们可以如下使用 PostgreSQL 的`xmin`列为我们的`User`类版本控制：

```py
from sqlalchemy import FetchedValue

class User(Base):
    __tablename__ = "user"

    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50), nullable=False)
    xmin = mapped_column("xmin", String, system=True, server_default=FetchedValue())

    __mapper_args__ = {"version_id_col": xmin, "version_id_generator": False}
```

使用上述映射，ORM 将依赖于`xmin`列自动提供版本 id 计数器的新值。

当 ORM 发出 INSERT 或 UPDATE 时，通常不会主动获取数据库生成的值，而是将这些列保留为“过期”，在下次访问它们时获取，除非设置了`eager_defaults` `Mapper`标志。但是，当使用服务器端版本列时，ORM 需要主动获取新生成的值。这样做是为了在任何并发事务可能再次更新之前设置版本计数器*之前*。最好同时在 INSERT 或 UPDATE 语句中使用 RETURNING 进行获取，否则，如果之后发出 SELECT 语句，则仍然存在潜在的竞争条件，版本计数器可能在获取之前更改。

当目标数据库支持 RETURNING 时，我们的`User`类的 INSERT 语句如下所示：

```py
INSERT  INTO  "user"  (name)  VALUES  (%(name)s)  RETURNING  "user".id,  "user".xmin
-- {'name': 'ed'}
```

在上述情况下，ORM 可以在一条语句中获取任何新生成的主键值以及服务器生成的版本标识符。当后端不支持 RETURNING 时，必须对**每个**INSERT 和 UPDATE 发出额外的 SELECT，这非常低效，还会引入可能丢失版本计数器的可能性：

```py
INSERT  INTO  "user"  (name)  VALUES  (%(name)s)
-- {'name': 'ed'}

SELECT  "user".version_id  AS  user_version_id  FROM  "user"  where
"user".id  =  :param_1
-- {"param_1": 1}
```

*强烈建议*仅在绝对必要时且仅在支持 RETURNING 的后端上使用服务器端版本计数器，目前支持的后端有 PostgreSQL、Oracle、MariaDB 10.5、SQLite 3.35 和 SQL Server。

## 编程或条件版本计数器

当`version_id_generator`设置为 False 时，我们还可以以与分配任何其他映射属性相同的方式，在对象上编程（和有条件地）设置版本标识符。例如，如果我们使用了 UUID 示例，但将`version_id_generator`设置为`False`，我们可以随意设置版本标识符：

```py
import uuid

class User(Base):
    __tablename__ = "user"

    id = mapped_column(Integer, primary_key=True)
    version_uuid = mapped_column(String(32), nullable=False)
    name = mapped_column(String(50), nullable=False)

    __mapper_args__ = {"version_id_col": version_uuid, "version_id_generator": False}

u1 = User(name="u1", version_uuid=uuid.uuid4())

session.add(u1)

session.commit()

u1.name = "u2"
u1.version_uuid = uuid.uuid4()

session.commit()
```

我们也可以在不增加版本计数器的情况下更新我们的`User`对象；计数器的值将保持不变，并且 UPDATE 语句仍将针对先前的值进行检查。对于仅某些类别的 UPDATE 对并发问题敏感的方案，这可能很有用：

```py
# will leave version_uuid unchanged
u1.name = "u3"
session.commit()
```

## 简单版本计数

跟踪版本的最直接方法是向映射表添加一个整数列，然后将其设置为映射选项中的`version_id_col`：

```py
class User(Base):
    __tablename__ = "user"

    id = mapped_column(Integer, primary_key=True)
    version_id = mapped_column(Integer, nullable=False)
    name = mapped_column(String(50), nullable=False)

    __mapper_args__ = {"version_id_col": version_id}
```

注意

**强烈建议**将`version_id`列设置为 NOT NULL。版本控制功能**不支持**版本控制列中的 NULL 值。

上面，`User`映射使用列`version_id`跟踪整数版本。当首次刷新`User`类型的对象时，`version_id`列的值将为“1”。然后，稍后对表的 UPDATE 将始终以类似以下方式发出：

```py
UPDATE  user  SET  version_id=:version_id,  name=:name
WHERE  user.id  =  :user_id  AND  user.version_id  =  :user_version_id
-- {"name": "new name", "version_id": 2, "user_id": 1, "user_version_id": 1}
```

上述 UPDATE 语句正在更新不仅与 `user.id = 1` 匹配的行，而且还要求 `user.version_id = 1`，其中“1”是我们已知的此对象上一次使用的最后版本标识符。如果其他地方的事务独立修改了行，则此版本 ID 将不再匹配，UPDATE 语句将报告没有匹配的行；这是 SQLAlchemy 测试的条件，确保我们的 UPDATE（或 DELETE）语句仅匹配了一行。如果没有匹配的行，则表示我们的数据版本已过期，并且会引发 `StaleDataError`。

## 自定义版本计数器 / 类型

其他类型的值或计数器可以用于版本控制。常见的类型包括日期和 GUID。当使用替代类型或计数器方案时，SQLAlchemy 提供了一个钩子来使用 `version_id_generator` 参数，该参数接受版本生成可调用对象。此可调用对象将传递当前已知版本的值，并且预计返回后续版本。

例如，如果我们想要使用随机生成的 GUID 跟踪我们的 `User` 类的版本控制，我们可以这样做（请注意，一些后端支持原生的 GUID 类型，但我们在这里使用简单的字符串进行演示）：

```py
import uuid

class User(Base):
    __tablename__ = "user"

    id = mapped_column(Integer, primary_key=True)
    version_uuid = mapped_column(String(32), nullable=False)
    name = mapped_column(String(50), nullable=False)

    __mapper_args__ = {
        "version_id_col": version_uuid,
        "version_id_generator": lambda version: uuid.uuid4().hex,
    }
```

持久性引擎每次将 `User` 对象受到 INSERT 或 UPDATE 影响时都会调用 `uuid.uuid4()`。在这种情况下，我们的版本生成函数可以忽略 `version` 的传入值，因为 `uuid4()` 函数生成的标识符没有任何先决条件值。如果我们使用的是顺序版本控制方案，例如数字或特殊字符系统，则可以利用给定的 `version` 来帮助确定后续值。

另请参阅

不特定后端的 GUID 类型

## 服务器端版本计数器

`version_id_generator` 也可以配置为依赖于数据库生成的值。在这种情况下，数据库需要在将行受到 INSERT 时以及 UPDATE 时生成新标识符的某种手段。对于 UPDATE 情况，通常需要一个更新触发器，除非所涉及的数据库支持其他本地版本标识符。特别是，PostgreSQL 数据库支持一个名为 [xmin](https://www.postgresql.org/docs/current/static/ddl-system-columns.html) 的系统列，提供 UPDATE 版本控制。我们可以利用 PostgreSQL 的 `xmin` 列来为我们的 `User` 类进行版本控制，如下所示：

```py
from sqlalchemy import FetchedValue

class User(Base):
    __tablename__ = "user"

    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50), nullable=False)
    xmin = mapped_column("xmin", String, system=True, server_default=FetchedValue())

    __mapper_args__ = {"version_id_col": xmin, "version_id_generator": False}
```

使用上述映射，ORM 将依赖于 `xmin` 列来自动提供版本 ID 计数器的新值。

当 ORM 发出 INSERT 或 UPDATE 时，通常不会主动获取数据库生成的值的值，而是将这些列保留为“过期”，并在下次访问它们时获取，除非设置了 `eager_defaults` `Mapper` 标志。然而，当使用服务器端版本列时，ORM 需要主动获取新生成的值。这是为了在任何并发事务可能再次更新它之前设置版本计数器。最好在 INSERT 或 UPDATE 语句中同时进行这个获取，使用 RETURNING，否则，如果之后发出一个 SELECT 语句，那么版本计数器在它被获取之前可能会发生竞争条件。

当目标数据库支持 RETURNING 时，我们的 `User` 类的 INSERT 语句将如下所示：

```py
INSERT  INTO  "user"  (name)  VALUES  (%(name)s)  RETURNING  "user".id,  "user".xmin
-- {'name': 'ed'}
```

在上述情况下，ORM 可以在一个语句中获取任何新生成的主键值以及服务器生成的版本标识符。当后端不支持 RETURNING 时，必须为**每个** INSERT 和 UPDATE 发出额外的 SELECT，这非常低效，还会引入遗漏版本计数器的可能性：

```py
INSERT  INTO  "user"  (name)  VALUES  (%(name)s)
-- {'name': 'ed'}

SELECT  "user".version_id  AS  user_version_id  FROM  "user"  where
"user".id  =  :param_1
-- {"param_1": 1}
```

仅在绝对必要时，并且仅在支持返回的后端上，强烈建议仅使用服务器端版本计数器，目前支持的后端有 PostgreSQL、Oracle、MariaDB 10.5、SQLite 3.35 和 SQL Server。

## 编程或有条件的版本计数器

当 `version_id_generator` 设置为 False 时，我们也可以以编程方式（并有条件地）像分配任何其他映射属性一样，在对象上设置版本标识符。例如，如果我们使用了 UUID 示例，但将 `version_id_generator` 设置为 `False`，我们可以根据自己的需要设置版本标识符：

```py
import uuid

class User(Base):
    __tablename__ = "user"

    id = mapped_column(Integer, primary_key=True)
    version_uuid = mapped_column(String(32), nullable=False)
    name = mapped_column(String(50), nullable=False)

    __mapper_args__ = {"version_id_col": version_uuid, "version_id_generator": False}

u1 = User(name="u1", version_uuid=uuid.uuid4())

session.add(u1)

session.commit()

u1.name = "u2"
u1.version_uuid = uuid.uuid4()

session.commit()
```

我们还可以在不递增版本计数器的情况下更新我们的 `User` 对象；计数器的值将保持不变，并且 UPDATE 语句仍将根据先前的值进行检查。这在仅特定类的 UPDATE 对并发问题敏感的方案中可能很有用：

```py
# will leave version_uuid unchanged
u1.name = "u3"
session.commit()
```
