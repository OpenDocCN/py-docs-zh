# 更改属性行为

> 原文：[`docs.sqlalchemy.org/en/20/orm/mapped_attributes.html`](https://docs.sqlalchemy.org/en/20/orm/mapped_attributes.html)

本节将讨论用于修改 ORM 映射属性行为的特性和技术，包括那些使用`mapped_column()`、`relationship()`等映射的属性。

## 简单的验证器

一个快速添加“验证”程序到属性的方法是使用`validates()`装饰器。属性验证器可以引发异常，停止突变属性值的过程，或者可以将给定值更改为其他值。像所有属性扩展一样，验证器仅在普通用户代码中调用；在 ORM 填充对象时，它们不会被调用：

```py
from sqlalchemy.orm import validates

class EmailAddress(Base):
    __tablename__ = "address"

    id = mapped_column(Integer, primary_key=True)
    email = mapped_column(String)

    @validates("email")
    def validate_email(self, key, address):
        if "@" not in address:
            raise ValueError("failed simple email validation")
        return address
```

当向集合添加项目时，验证器还会接收集合追加事件：

```py
from sqlalchemy.orm import validates

class User(Base):
    # ...

    addresses = relationship("Address")

    @validates("addresses")
    def validate_address(self, key, address):
        if "@" not in address.email:
            raise ValueError("failed simplified email validation")
        return address
```

验证函数默认不会为集合移除事件发出，因为典型的期望是被丢弃的值不需要验证。然而，`validates()`支持通过向装饰器指定`include_removes=True`来接收这些事件。当设置了此标志时，验证函数必须接收一个额外的布尔参数，如果为`True`，则表示操作是一个移除：

```py
from sqlalchemy.orm import validates

class User(Base):
    # ...

    addresses = relationship("Address")

    @validates("addresses", include_removes=True)
    def validate_address(self, key, address, is_remove):
        if is_remove:
            raise ValueError("not allowed to remove items from the collection")
        else:
            if "@" not in address.email:
                raise ValueError("failed simplified email validation")
            return address
```

通过反向引用链接的相互依赖验证器的情况也可以进行定制，使用`include_backrefs=False`选项；当设置为`False`时，此选项会阻止验证函数在由反向引用导致的事件发生时发出：

```py
from sqlalchemy.orm import validates

class User(Base):
    # ...

    addresses = relationship("Address", backref="user")

    @validates("addresses", include_backrefs=False)
    def validate_address(self, key, address):
        if "@" not in address:
            raise ValueError("failed simplified email validation")
        return address
```

在上面的例子中，如果我们像这样分配到`Address.user`，如`some_address.user = some_user`，那么`validate_address()`函数将*不会*被发出，即使`some_user.addresses`中发生了追加 - 该事件是由反向引用引起的。

请注意，`validates()`装饰器是建立在属性事件之上的方便函数。需要更多控制属性更改行为配置的应用程序可以利用此系统，详见`AttributeEvents`。

| 对象名称 | 描述 |
| --- | --- |
| validates(*names, [include_removes, include_backrefs]) | 将方法装饰为一个或多个命名属性的“验证器”。 |

```py
function sqlalchemy.orm.validates(*names: str, include_removes: bool = False, include_backrefs: bool = True) → Callable[[_Fn], _Fn]
```

将方法装饰为一个或多个命名属性的“验证器”。

将方法指定为验证器，该方法接收属性的名称以及要分配的值，或者在集合的情况下，要添加到集合的值。然后，函数可以引发验证异常以阻止进程继续（在这种情况下，Python 的内置`ValueError`和`AssertionError`异常是合理的选择），或者可以在继续之前修改或替换值。否则，该函数应返回给定的值。

注意，集合的验证器**不能**在验证过程中发出该集合的加载操作 - 这种用法会引发断言以避免递归溢出。这是一种不支持的可重入条件。

参数：

+   `*names` – 要验证的属性名称列表。

+   `include_removes` – 如果为 True，则也将发送“remove”事件 - 验证函数必须接受一个额外参数“is_remove”，其值为布尔值。

+   `include_backrefs` –

    默认为`True`；如果为`False`，则验证函数不会在原始操作者是通过 backref 相关的属性事件时发出。这可用于双向`validates()`使用，其中每个属性操作只应发出一个验证器。

    从版本 2.0.16 开始更改：此参数在版本 2.0.0 到 2.0.15 中无意中默认为`False`。在 2.0.16 中恢复了其正确的默认值为`True`。

另请参阅

简单验证器 - `validates()`的使用示例

## 在核心级别使用自定义数据类型

影响列值的非 ORM 方式，以适合在 Python 中的表示方式与在数据库中的表示方式之间转换数据，可以通过使用应用于映射的`Table`元数据的自定义数据类型来实现。这在一些编码/解码风格在数据进入数据库和返回时都会发生的情况下更为常见；在 Core 文档的扩充现有类型中了解更多信息。

## 使用描述符和混合体

影响属性的修改行为的更全面的方法是使用描述符。这在 Python 中通常使用`property()`函数。描述符的标准 SQLAlchemy 技术是创建一个普通的描述符，并从具有不同名称的映射属性读取/写入。下面我们使用 Python 2.6 风格的属性进行说明：

```py
class EmailAddress(Base):
    __tablename__ = "email_address"

    id = mapped_column(Integer, primary_key=True)

    # name the attribute with an underscore,
    # different from the column name
    _email = mapped_column("email", String)

    # then create an ".email" attribute
    # to get/set "._email"
    @property
    def email(self):
        return self._email

    @email.setter
    def email(self, email):
        self._email = email
```

上述方法可以工作，但我们可以添加更多内容。虽然我们的 `EmailAddress` 对象将通过 `email` 描述符将值传递到 `_email` 映射属性中，但类级别的 `EmailAddress.email` 属性没有通常的表达式语义可用于 `Select`。为了提供这些，我们可以使用 `hybrid` 扩展，如下所示：

```py
from sqlalchemy.ext.hybrid import hybrid_property

class EmailAddress(Base):
    __tablename__ = "email_address"

    id = mapped_column(Integer, primary_key=True)

    _email = mapped_column("email", String)

    @hybrid_property
    def email(self):
        return self._email

    @email.setter
    def email(self, email):
        self._email = email
```

`.email` 属性除了在有 `EmailAddress` 实例时提供 getter/setter 行为外，还在类级别使用时提供 SQL 表达式，即直接从 `EmailAddress` 类中使用时：

```py
from sqlalchemy.orm import Session
from sqlalchemy import select

session = Session()

address = session.scalars(
    select(EmailAddress).where(EmailAddress.email == "address@example.com")
).one()
SELECT  address.email  AS  address_email,  address.id  AS  address_id
FROM  address
WHERE  address.email  =  ?
('address@example.com',)
address.email = "otheraddress@example.com"
session.commit()
UPDATE  address  SET  email=?  WHERE  address.id  =  ?
('otheraddress@example.com',  1)
COMMIT 
```

`hybrid_property` 还允许我们更改属性的行为，包括在实例级别与类/表达式级别访问属性时定义不同的行为，使用 `hybrid_property.expression()` 修饰符。例如，如果我们想要自动添加主机名，我们可以定义两组字符串操作逻辑：

```py
class EmailAddress(Base):
    __tablename__ = "email_address"

    id = mapped_column(Integer, primary_key=True)

    _email = mapped_column("email", String)

    @hybrid_property
    def email(self):
  """Return the value of _email up until the last twelve
 characters."""

        return self._email[:-12]

    @email.setter
    def email(self, email):
  """Set the value of _email, tacking on the twelve character
 value @example.com."""

        self._email = email + "@example.com"

    @email.expression
    def email(cls):
  """Produce a SQL expression that represents the value
 of the _email column, minus the last twelve characters."""

        return func.substr(cls._email, 0, func.length(cls._email) - 12)
```

在上面的例子中，访问 `EmailAddress` 实例的 `email` 属性将返回 `_email` 属性的值，从值中移除或添加主机名 `@example.com`。当我们针对 `email` 属性进行查询时，会呈现一个产生相同效果的 SQL 函数：

```py
address = session.scalars(
    select(EmailAddress).where(EmailAddress.email == "address")
).one()
SELECT  address.email  AS  address_email,  address.id  AS  address_id
FROM  address
WHERE  substr(address.email,  ?,  length(address.email)  -  ?)  =  ?
(0,  12,  'address') 
```

阅读更多关于混合属性的信息请参阅 混合属性。 ## 同义词

同义词是一个映射器级别的构造，允许类上的任何属性“镜像”另一个映射的属性。

从最基本的角度来看，同义词是一种使某个属性通过额外的名称轻松可用的方式：

```py
from sqlalchemy.orm import synonym

class MyClass(Base):
    __tablename__ = "my_table"

    id = mapped_column(Integer, primary_key=True)
    job_status = mapped_column(String(50))

    status = synonym("job_status")
```

上面的 `MyClass` 类具有两个属性，`.job_status` 和 `.status`，它们将作为一个属性在表达式级别上行为一致：

```py
>>> print(MyClass.job_status == "some_status")
my_table.job_status  =  :job_status_1
>>> print(MyClass.status == "some_status")
my_table.job_status  =  :job_status_1 
```

并在实例级别：

```py
>>> m1 = MyClass(status="x")
>>> m1.status, m1.job_status
('x', 'x')

>>> m1.job_status = "y"
>>> m1.status, m1.job_status
('y', 'y')
```

`synonym()` 可用于任何类型的映射属性，包括映射列和关系，以及同义词本身，它们都是 `MapperProperty` 的子类。

除了简单的镜像外，`synonym()` 还可以被设置为引用用户定义的 描述符。我们可以用 `@property` 来提供我们的 `status` 同义词：

```py
class MyClass(Base):
    __tablename__ = "my_table"

    id = mapped_column(Integer, primary_key=True)
    status = mapped_column(String(50))

    @property
    def job_status(self):
        return "Status: " + self.status

    job_status = synonym("status", descriptor=job_status)
```

在使用 Declarative 时，可以更简洁地使用 `synonym_for()` 装饰器表达上述模式：

```py
from sqlalchemy.ext.declarative import synonym_for

class MyClass(Base):
    __tablename__ = "my_table"

    id = mapped_column(Integer, primary_key=True)
    status = mapped_column(String(50))

    @synonym_for("status")
    @property
    def job_status(self):
        return "Status: " + self.status
```

虽然 `synonym()` 对于简单的镜像很有用，但是使用描述符增强属性行为的用例更好地使用了现代用法中的 混合属性 功能，后者更加面向 Python 描述符。 从技术上讲，`synonym()` 可以做到与 `hybrid_property` 相同的所有事情，因为它还支持注入自定义 SQL 功能，但是混合属性在更复杂的情况下更容易使用。

| 对象名称 | 描述 |
| --- | --- |
| 同义词(name, *, [map_column, descriptor, comparator_factory, init, repr, default, default_factory, compare, kw_only, info, doc]) | 将属性名称标记为映射属性的同义词，即属性将反映另一个属性的值和表达行为。 |

```py
function sqlalchemy.orm.synonym(name: str, *, map_column: bool | None = None, descriptor: Any | None = None, comparator_factory: Type[PropComparator[_T]] | None = None, init: _NoArg | bool = _NoArg.NO_ARG, repr: _NoArg | bool = _NoArg.NO_ARG, default: _NoArg | _T = _NoArg.NO_ARG, default_factory: _NoArg | Callable[[], _T] = _NoArg.NO_ARG, compare: _NoArg | bool = _NoArg.NO_ARG, kw_only: _NoArg | bool = _NoArg.NO_ARG, info: _InfoType | None = None, doc: str | None = None) → Synonym[Any]
```

将属性名称标记为映射属性的同义词，即属性将反映另一个属性的值和表达行为。

例如：

```py
class MyClass(Base):
    __tablename__ = 'my_table'

    id = Column(Integer, primary_key=True)
    job_status = Column(String(50))

    status = synonym("job_status")
```

参数：

+   `name` – 现有映射属性的名称。这可以是配置在类上的字符串名称 ORM 映射属性，包括列绑定属性和关系。

+   `descriptor` – 当在实例级别访问此属性时将用作 getter（和可能的 setter）的 Python descriptor。

+   `map_column` –

    **仅适用于经典映射和映射到现有 Table 对象的情况**。 如果为 `True`，`synonym()` 构造将定位到与此同义词的属性名称通常关联的映射表上的 `Column` 对象，并生成一个新的 `ColumnProperty`，将此 `Column` 映射到作为“name”参数给定的替代名称；这样，重新定义将 `Column` 的映射放在不同名称下的常规步骤是不必要的。这通常用于当 `Column` 要替换为也使用描述符的属性时，即与 `synonym.descriptor` 参数一起使用：

    ```py
    my_table = Table(
        "my_table", metadata,
        Column('id', Integer, primary_key=True),
        Column('job_status', String(50))
    )

    class MyClass:
        @property
        def _job_status_descriptor(self):
            return "Status: %s" % self._job_status

    mapper(
        MyClass, my_table, properties={
            "job_status": synonym(
                "_job_status", map_column=True,
                descriptor=MyClass._job_status_descriptor)
        }
    )
    ```

    上述，名为 `_job_status` 的属性自动映射到 `job_status` 列：

    ```py
    >>> j1 = MyClass()
    >>> j1._job_status = "employed"
    >>> j1.job_status
    Status: employed
    ```

    在使用 Declarative 时，为了在同义词中提供一个描述符，请使用`sqlalchemy.ext.declarative.synonym_for()`辅助程序。但是，请注意，通常应优先选择混合属性功能，特别是在重新定义属性行为时。

+   `info` – 将填充到此对象的`InspectionAttr.info`属性中的可选数据字典。

+   `comparator_factory` –

    一个`PropComparator`的子类，将在 SQL 表达式级别提供自定义比较行为。

    注意

    对于提供重新定义属性的 Python 级别和 SQL 表达式级别行为的用例，请参考使用描述符和混合属性中介绍的混合属性，以获得更有效的技术。

另请参阅

同义词 - 同义词概述

`synonym_for()` - 面向 Declarative 的辅助程序

使用描述符和混合属性 - 混合属性扩展提供了一种更新的方法，比使用同义词更灵活地增强属性行为。## 操作符定制

SQLAlchemy ORM 和 Core 表达式语言使用的“操作符”是完全可定制的。例如，比较表达式`User.name == 'ed'`使用了 Python 本身内置的名为`operator.eq`的操作符 - SQLAlchemy 将与此类操作符关联的实际 SQL 构造可以进行修改。新操作也可以与列表达式关联。列表达式发生的操作符最直接在类型级别重新定义 - 请参阅 Redefining and Creating New Operators 部分进行描述。

ORM 级别的函数，如`column_property()`，`relationship()`和`composite()`还提供了在 ORM 级别重新定义操作符的功能，通过将`PropComparator`子类传递给每个函数的`comparator_factory`参数。在这个级别上定制操作符是一个罕见的用例。请参阅`PropComparator`的文档以获取概述。## 简单验证器

将“验证”程序快速添加到属性的一种方法是使用 `validates()` 装饰器。属性验证器可以引发异常，从而停止变异属性值的过程，或者可以将给定值更改为其他内容。验证器，如所有属性扩展一样，仅在正常用户代码中调用；当 ORM 正在填充对象时，不会发出它们：

```py
from sqlalchemy.orm import validates

class EmailAddress(Base):
    __tablename__ = "address"

    id = mapped_column(Integer, primary_key=True)
    email = mapped_column(String)

    @validates("email")
    def validate_email(self, key, address):
        if "@" not in address:
            raise ValueError("failed simple email validation")
        return address
```

当项目被添加到集合时，验证器也会收到集合追加事件：

```py
from sqlalchemy.orm import validates

class User(Base):
    # ...

    addresses = relationship("Address")

    @validates("addresses")
    def validate_address(self, key, address):
        if "@" not in address.email:
            raise ValueError("failed simplified email validation")
        return address
```

默认情况下，验证函数不会为集合删除事件发出，因为典型的期望是被丢弃的值不需要验证。但是，`validates()` 通过将 `include_removes=True` 指定给装饰器来支持接收这些事件。当设置了此标志时，验证函数必须接收一个额外的布尔参数，如果为 `True`，则表示该操作是一个删除操作：

```py
from sqlalchemy.orm import validates

class User(Base):
    # ...

    addresses = relationship("Address")

    @validates("addresses", include_removes=True)
    def validate_address(self, key, address, is_remove):
        if is_remove:
            raise ValueError("not allowed to remove items from the collection")
        else:
            if "@" not in address.email:
                raise ValueError("failed simplified email validation")
            return address
```

通过使用 `include_backrefs=False` 选项，还可以针对通过反向引用链接的相互依赖验证器的情况进行定制；当设置为 `False` 时，该选项将阻止验证函数在事件发生时由于反向引用而发出：

```py
from sqlalchemy.orm import validates

class User(Base):
    # ...

    addresses = relationship("Address", backref="user")

    @validates("addresses", include_backrefs=False)
    def validate_address(self, key, address):
        if "@" not in address:
            raise ValueError("failed simplified email validation")
        return address
```

在上面的例子中，如果我们像这样分配给 `Address.user`：`some_address.user = some_user`，即使向 `some_user.addresses` 追加了一个元素，也不会触发 `validate_address()` 函数 - 事件是由一个反向引用引起的。

请注意，`validates()` 装饰器是在属性事件之上构建的一个方便函数。需要更多控制属性更改行为配置的应用程序可以使用此系统，该系统在 `AttributeEvents` 中描述。

| 对象名称 | 描述 |
| --- | --- |
| 验证(*names, [include_removes, include_backrefs]) | 将方法装饰为一个或多个命名属性的“验证器”。 |

```py
function sqlalchemy.orm.validates(*names: str, include_removes: bool = False, include_backrefs: bool = True) → Callable[[_Fn], _Fn]
```

将方法装饰为一个或多个命名属性的“验证器”。

指定一个方法作为验证器，该方法接收属性的名称以及要分配的值，或者在集合的情况下，要添加到集合的值。该函数然后可以引发验证异常以阻止继续处理过程（在这种情况下，Python 的内置`ValueError`和`AssertionError`异常是合理的选择），或者可以修改或替换值然后继续。该函数否则应返回给定的值。

请注意，集合的验证器**不能**在验证例程中发出该集合的加载 - 这种用法会引发一个断言以避免递归溢出。这是一个不支持的可重入条件。

参数：

+   `*names` – 要验证的属性名称列表。

+   `include_removes` – 如果为 True，则“remove”事件也将发送 - 验证函数必须接受一个额外的参数“is_remove”，它将是一个布尔值。

+   `include_backrefs` –

    默认为`True`；如果为`False`，则验证函数在原始生成器是通过 backref 相关的属性事件时不会发出。这可用于双向 `validates()` 用法，其中每个属性操作只应发出一个验证器。

    从版本 2.0.16 开始更改：此参数意外地在 2.0.0 至 2.0.15 版本中默认为 `False`。在 2.0.16 版本中恢复了其正确的默认值为`True`。

另请参阅

简单验证器 - `validates()` 的用法示例

## 在核心级别使用自定义数据类型

通过使用应用于映射的 `Table` 元数据的自定义数据类型，可以以适合在 Python 中的表示方式与在数据库中的表示方式之间转换数据的方式来影响列的值的非 ORM 方法。这在某些编码/解码样式在数据进入数据库和返回时都发生的情况下更为常见；在核心文档中阅读更多关于此的内容，参见扩充现有类型。

## 使用描述符和混合类型

产生修改后的属性行为的更全面的方法是使用描述符。在 Python 中，通常使用 `property()` 函数来使用这些。描述符的标准 SQLAlchemy 技术是创建一个普通描述符，并从具有不同名称的映射属性读取/写入。下面我们使用 Python 2.6 风格的属性来说明这一点：

```py
class EmailAddress(Base):
    __tablename__ = "email_address"

    id = mapped_column(Integer, primary_key=True)

    # name the attribute with an underscore,
    # different from the column name
    _email = mapped_column("email", String)

    # then create an ".email" attribute
    # to get/set "._email"
    @property
    def email(self):
        return self._email

    @email.setter
    def email(self, email):
        self._email = email
```

上述方法可行，但我们还可以添加更多内容。虽然我们的`EmailAddress`对象将值通过`email`描述符传递到`_email`映射属性中，但类级别的`EmailAddress.email`属性不具有通常可用于`Select`的表达语义。为了提供这些功能，我们使用 `hybrid` 扩展，如下所示：

```py
from sqlalchemy.ext.hybrid import hybrid_property

class EmailAddress(Base):
    __tablename__ = "email_address"

    id = mapped_column(Integer, primary_key=True)

    _email = mapped_column("email", String)

    @hybrid_property
    def email(self):
        return self._email

    @email.setter
    def email(self, email):
        self._email = email
```

`.email` 属性除了在我们有`EmailAddress`实例时提供 getter/setter 行为外，在类级别使用时也提供了一个 SQL 表达式，即直接从`EmailAddress`类中：

```py
from sqlalchemy.orm import Session
from sqlalchemy import select

session = Session()

address = session.scalars(
    select(EmailAddress).where(EmailAddress.email == "address@example.com")
).one()
SELECT  address.email  AS  address_email,  address.id  AS  address_id
FROM  address
WHERE  address.email  =  ?
('address@example.com',)
address.email = "otheraddress@example.com"
session.commit()
UPDATE  address  SET  email=?  WHERE  address.id  =  ?
('otheraddress@example.com',  1)
COMMIT 
```

`hybrid_property`还允许我们更改属性的行为，包括在实例级别与类/表达式级别访问属性时定义不同的行为，使用`hybrid_property.expression()`修饰符。例如，如果我们想要自动添加主机名，我们可能会定义两组字符串操作逻辑：

```py
class EmailAddress(Base):
    __tablename__ = "email_address"

    id = mapped_column(Integer, primary_key=True)

    _email = mapped_column("email", String)

    @hybrid_property
    def email(self):
  """Return the value of _email up until the last twelve
 characters."""

        return self._email[:-12]

    @email.setter
    def email(self, email):
  """Set the value of _email, tacking on the twelve character
 value @example.com."""

        self._email = email + "@example.com"

    @email.expression
    def email(cls):
  """Produce a SQL expression that represents the value
 of the _email column, minus the last twelve characters."""

        return func.substr(cls._email, 0, func.length(cls._email) - 12)
```

以上，访问`EmailAddress`实例的`email`属性将返回`_email`属性的值，从值中删除或添加主机名`@example.com`。当我们针对`email`属性进行查询时，将呈现出一个产生相同效果的 SQL 函数：

```py
address = session.scalars(
    select(EmailAddress).where(EmailAddress.email == "address")
).one()
SELECT  address.email  AS  address_email,  address.id  AS  address_id
FROM  address
WHERE  substr(address.email,  ?,  length(address.email)  -  ?)  =  ?
(0,  12,  'address') 
```

在混合属性中阅读更多内容。

## 同义词

同义词是一个映射级别的构造，允许类上的任何属性“镜像”另一个被映射的属性。

从最基本的意义上讲，同义词是一种简单的方式，通过额外的名称使某个属性可用：

```py
from sqlalchemy.orm import synonym

class MyClass(Base):
    __tablename__ = "my_table"

    id = mapped_column(Integer, primary_key=True)
    job_status = mapped_column(String(50))

    status = synonym("job_status")
```

上述`MyClass`类有两个属性，`.job_status`和`.status`，它们将作为一个属性行为，无论在表达式级别还是在实例级别：

```py
>>> print(MyClass.job_status == "some_status")
my_table.job_status  =  :job_status_1
>>> print(MyClass.status == "some_status")
my_table.job_status  =  :job_status_1 
```

在实例级别上：

```py
>>> m1 = MyClass(status="x")
>>> m1.status, m1.job_status
('x', 'x')

>>> m1.job_status = "y"
>>> m1.status, m1.job_status
('y', 'y')
```

`synonym()`可以用于任何一种映射属性，包括映射列和关系，以及同义词本身，这些属性都是`MapperProperty`的子类。

除了简单的镜像之外，`synonym()`还可以引用用户定义的描述符。我们可以用`@property`来提供我们的`status`同义词：

```py
class MyClass(Base):
    __tablename__ = "my_table"

    id = mapped_column(Integer, primary_key=True)
    status = mapped_column(String(50))

    @property
    def job_status(self):
        return "Status: " + self.status

    job_status = synonym("status", descriptor=job_status)
```

在使用声明性时，可以使用`synonym_for()`装饰器更简洁地表达上述模式：

```py
from sqlalchemy.ext.declarative import synonym_for

class MyClass(Base):
    __tablename__ = "my_table"

    id = mapped_column(Integer, primary_key=True)
    status = mapped_column(String(50))

    @synonym_for("status")
    @property
    def job_status(self):
        return "Status: " + self.status
```

虽然`synonym()`对于简单的镜像很有用，但是使用描述符增强属性行为的用例更好地在现代使用中使用混合属性特性来处理，后者更加面向 Python 描述符。从技术上讲，一个`synonym()`可以做任何一个`hybrid_property`能做的事情，因为它也支持注入自定义 SQL 功能，但是在更复杂的情况下混合属性更容易使用。

| 对象名称 | 描述 |
| --- | --- |
| synonym(name, *, [map_column, descriptor, comparator_factory, init, repr, default, default_factory, compare, kw_only, info, doc]) | 将一个属性名表示为映射属性的同义词，即该属性将反映另一个属性的值和表达式行为。 |

```py
function sqlalchemy.orm.synonym(name: str, *, map_column: bool | None = None, descriptor: Any | None = None, comparator_factory: Type[PropComparator[_T]] | None = None, init: _NoArg | bool = _NoArg.NO_ARG, repr: _NoArg | bool = _NoArg.NO_ARG, default: _NoArg | _T = _NoArg.NO_ARG, default_factory: _NoArg | Callable[[], _T] = _NoArg.NO_ARG, compare: _NoArg | bool = _NoArg.NO_ARG, kw_only: _NoArg | bool = _NoArg.NO_ARG, info: _InfoType | None = None, doc: str | None = None) → Synonym[Any]
```

将一个属性名表示为映射属性的同义词，即该属性将反映另一个属性的值和表达式行为。

例如：

```py
class MyClass(Base):
    __tablename__ = 'my_table'

    id = Column(Integer, primary_key=True)
    job_status = Column(String(50))

    status = synonym("job_status")
```

参数：

+   `name` – 现有映射属性的名称。这可以引用在类上配置的 ORM 映射属性的字符串名称，包括列绑定属性和关系。

+   `descriptor` – 一个 Python 描述符，当访问此属性时将用作 getter（和可能的 setter）。

+   `map_column` –

    **仅适用于传统映射和对现有表对象的映射**。如果为`True`，则`synonym()`构造将定位到在此同义词的属性名称通常与该同义词的属性名称相关联的映射表上的`Column`对象，并生成一个新的`ColumnProperty`，该属性将此`Column`映射到作为同义词的“name”参数给定的替代名称；通过这种方式，重新定义`Column`的映射为不同名称的步骤是不必要的。这通常用于当`Column`要被替换为也使用描述符的属性时，也就是与`synonym.descriptor`参数结合使用时：

    ```py
    my_table = Table(
        "my_table", metadata,
        Column('id', Integer, primary_key=True),
        Column('job_status', String(50))
    )

    class MyClass:
        @property
        def _job_status_descriptor(self):
            return "Status: %s" % self._job_status

    mapper(
        MyClass, my_table, properties={
            "job_status": synonym(
                "_job_status", map_column=True,
                descriptor=MyClass._job_status_descriptor)
        }
    )
    ```

    在上面的例子中，名为`_job_status`的属性会自动映射到`job_status`列：

    ```py
    >>> j1 = MyClass()
    >>> j1._job_status = "employed"
    >>> j1.job_status
    Status: employed
    ```

    当使用声明式时，为了与同义词结合使用提供描述符，请使用`sqlalchemy.ext.declarative.synonym_for()`助手。但是，请注意，通常应优选混合属性功能，特别是在重新定义属性行为时。

+   `info` – 可选的数据字典，将填充到此对象的`InspectionAttr.info`属性中。

+   `comparator_factory` –

    `PropComparator`的子类，将在 SQL 表达式级别提供自定义比较行为。

    注意

    对于提供重新定义属性的 Python 级别和 SQL 表达式级别行为的用例，请参阅使用描述符和混合中介绍的混合属性，这是一种更有效的技术。

另请参阅

同义词 - 同义词概述

`synonym_for()` - 一种面向声明式的辅助工具

使用描述符和混合 - 混合属性扩展提供了一种更新的方法，可以更灵活地增强属性行为，比同义词更有效。

## 运算符定制

SQLAlchemy ORM 和 Core 表达式语言使用的“运算符”是完全可定制的。例如，比较表达式 `User.name == 'ed'` 使用了 Python 本身内置的名为 `operator.eq` 的运算符 - SQLAlchemy 关联的实际 SQL 构造可以被修改。新的操作也可以与列表达式关联起来。最直接重新定义列表达式的运算符的方法是在类型级别进行 - 详细信息请参阅重新定义和创建新的运算符。

ORM 级别的函数如`column_property()`、`relationship()`和`composite()`还提供了在 ORM 级别重新定义运算符的功能，方法是将`PropComparator`子类传递给每个函数的`comparator_factory`参数。在这个级别定制运算符的情况很少见。详细信息请参阅`PropComparator`的文档概述。
