# 混合属性

> 原文：[`docs.sqlalchemy.org/en/20/orm/extensions/hybrid.html`](https://docs.sqlalchemy.org/en/20/orm/extensions/hybrid.html)

定义在 ORM 映射类上具有“混合”行为的属性。

“混合”意味着属性在类级别和实例级别具有不同的行为。

`hybrid` 扩展提供了一种特殊形式的方法装饰器，并且对 SQLAlchemy 的其余部分具有最小的依赖性。 它的基本操作理论可以与任何基于描述符的表达式系统一起使用。

考虑一个映射 `Interval`，表示整数 `start` 和 `end` 值。 我们可以在映射类上定义更高级别的函数，这些函数在类级别生成 SQL 表达式，并在实例级别进行 Python 表达式评估。 下面，每个使用 `hybrid_method` 或 `hybrid_property` 装饰的函数可能会接收 `self` 作为类的实例，或者直接接收类，具体取决于上下文：

```py
from __future__ import annotations

from sqlalchemy.ext.hybrid import hybrid_method
from sqlalchemy.ext.hybrid import hybrid_property
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column

class Base(DeclarativeBase):
    pass

class Interval(Base):
    __tablename__ = 'interval'

    id: Mapped[int] = mapped_column(primary_key=True)
    start: Mapped[int]
    end: Mapped[int]

    def __init__(self, start: int, end: int):
        self.start = start
        self.end = end

    @hybrid_property
    def length(self) -> int:
        return self.end - self.start

    @hybrid_method
    def contains(self, point: int) -> bool:
        return (self.start <= point) & (point <= self.end)

    @hybrid_method
    def intersects(self, other: Interval) -> bool:
        return self.contains(other.start) | self.contains(other.end)
```

上面，`length` 属性返回 `end` 和 `start` 属性之间的差异。 对于 `Interval` 的实例，这个减法在 Python 中发生，使用正常的 Python 描述符机制：

```py
>>> i1 = Interval(5, 10)
>>> i1.length
5
```

处理 `Interval` 类本身时，`hybrid_property` 描述符将函数体评估为给定 `Interval` 类作为参数，当使用 SQLAlchemy 表达式机制评估时，将返回新的 SQL 表达式：

```py
>>> from sqlalchemy import select
>>> print(select(Interval.length))
SELECT  interval."end"  -  interval.start  AS  length
FROM  interval
>>> print(select(Interval).filter(Interval.length > 10))
SELECT  interval.id,  interval.start,  interval."end"
FROM  interval
WHERE  interval."end"  -  interval.start  >  :param_1 
```

过滤方法如 `Select.filter_by()` 也支持混合属性：

```py
>>> print(select(Interval).filter_by(length=5))
SELECT  interval.id,  interval.start,  interval."end"
FROM  interval
WHERE  interval."end"  -  interval.start  =  :param_1 
```

`Interval` 类示例还说明了两种方法，`contains()` 和 `intersects()`，使用 `hybrid_method` 装饰。 这个装饰器将相同的思想应用于方法，就像 `hybrid_property` 将其应用于属性一样。 这些方法返回布尔值，并利用 Python 的 `|` 和 `&` 位运算符产生等效的实例级和 SQL 表达式级布尔行为：

```py
>>> i1.contains(6)
True
>>> i1.contains(15)
False
>>> i1.intersects(Interval(7, 18))
True
>>> i1.intersects(Interval(25, 29))
False

>>> print(select(Interval).filter(Interval.contains(15)))
SELECT  interval.id,  interval.start,  interval."end"
FROM  interval
WHERE  interval.start  <=  :start_1  AND  interval."end"  >  :end_1
>>> ia = aliased(Interval)
>>> print(select(Interval, ia).filter(Interval.intersects(ia)))
SELECT  interval.id,  interval.start,
interval."end",  interval_1.id  AS  interval_1_id,
interval_1.start  AS  interval_1_start,  interval_1."end"  AS  interval_1_end
FROM  interval,  interval  AS  interval_1
WHERE  interval.start  <=  interval_1.start
  AND  interval."end"  >  interval_1.start
  OR  interval.start  <=  interval_1."end"
  AND  interval."end"  >  interval_1."end" 
```

## 定义与属性行为不同的表达行为

在前一节中，我们在 `Interval.contains` 和 `Interval.intersects` 方法中使用 `&` 和 `|` 按位运算符是幸运的，考虑到我们的函数操作两个布尔值以返回一个新值。在许多情况下，Python 函数的构建和 SQLAlchemy SQL 表达式有足够的差异，因此应该定义两个独立的 Python 表达式。`hybrid` 装饰器为此目的定义了一个 **修饰符** `hybrid_property.expression()`。作为示例，我们将定义区间的半径，这需要使用绝对值函数：

```py
from sqlalchemy import ColumnElement
from sqlalchemy import Float
from sqlalchemy import func
from sqlalchemy import type_coerce

class Interval(Base):
    # ...

    @hybrid_property
    def radius(self) -> float:
        return abs(self.length) / 2

    @radius.inplace.expression
    @classmethod
    def _radius_expression(cls) -> ColumnElement[float]:
        return type_coerce(func.abs(cls.length) / 2, Float)
```

在上述示例中，首先分配给名称 `Interval.radius` 的 `hybrid_property` 在后续使用 `Interval._radius_expression` 方法进行修改，使用装饰器 `@radius.inplace.expression`，将两个修饰符 `hybrid_property.inplace` 和 `hybrid_property.expression` 连接在一起。使用 `hybrid_property.inplace` 指示 `hybrid_property.expression()` 修饰符应在原地突变现有的混合对象 `Interval.radius`，而不是创建一个新对象。有关此修饰符及其基本原理的注释将在下一节 使用 inplace 创建符合 pep-484 的混合属性 中讨论。使用 `@classmethod` 是可选的，严格来说是为了给类型提示工具一个提示，即这种情况下 `cls` 应该是 `Interval` 类，而不是 `Interval` 的实例。

注意

`hybrid_property.inplace` 以及使用 `@classmethod` 进行正确类型支持的功能在 SQLAlchemy 2.0.4 中可用，之前的版本不支持。

`Interval.radius` 现在包含一个表达式元素，当在类级别访问 `Interval.radius` 时，会返回 SQL 函数 `ABS()`：

```py
>>> from sqlalchemy import select
>>> print(select(Interval).filter(Interval.radius > 5))
SELECT  interval.id,  interval.start,  interval."end"
FROM  interval
WHERE  abs(interval."end"  -  interval.start)  /  :abs_1  >  :param_1 
```  ## 使用 `inplace` 创建符合 pep-484 的混合属性

在前一节中，说明了一个`hybrid_property`装饰器，其中包含两个独立的方法级函数被装饰，都用于生成一个称为`Interval.radius`的单个对象属性。实际上，我们可以使用几种不同的修饰符来修饰`hybrid_property`，包括`hybrid_property.expression()`、`hybrid_property.setter()`和`hybrid_property.update_expression()`。

SQLAlchemy 的`hybrid_property`装饰器意味着可以以与 Python 内置的`@property`装饰器相同的方式添加这些方法，其中惯用的用法是继续重定义属性，每次都使用**相同的属性名称**，就像下面的示例中演示的那样，说明了使用`hybrid_property.setter()`和`hybrid_property.expression()`来描述`Interval.radius`的用法：

```py
# correct use, however is not accepted by pep-484 tooling

class Interval(Base):
    # ...

    @hybrid_property
    def radius(self):
        return abs(self.length) / 2

    @radius.setter
    def radius(self, value):
        self.length = value * 2

    @radius.expression
    def radius(cls):
        return type_coerce(func.abs(cls.length) / 2, Float)
```

如上所述，有三个`Interval.radius`方法，但由于每个都被`hybrid_property`装饰器和`@radius`名称本身装饰，因此最终效果是`Interval.radius`是一个具有三个不同功能的单个属性。这种使用方式取自于[Python 文档中对@property 的使用](https://docs.python.org/3/library/functions.html#property)。值得注意的是，`@property`以及`hybrid_property`的工作方式，每次都会**复制描述符**。也就是说，每次调用`@radius.expression`、`@radius.setter`等都会完全创建一个新对象。这允许在子类中重新定义属性而无需问题（请参阅本节稍后的在子类中重用混合属性的使用方式）。

然而，上述方法与 mypy 和 pyright 等类型工具不兼容。 Python 自己的`@property`装饰器之所以没有此限制，只是因为[这些工具硬编码了@property 的行为](https://github.com/python/typing/discussions/1102)，这意味着此语法不符合[**PEP 484**](https://peps.python.org/pep-0484/)的要求。

为了产生合理的语法，同时保持类型兼容性，`hybrid_property.inplace`装饰器允许使用不同的方法名重复使用相同的装饰器，同时仍然在一个名称下生成一个装饰器：

```py
# correct use which is also accepted by pep-484 tooling

class Interval(Base):
    # ...

    @hybrid_property
    def radius(self) -> float:
        return abs(self.length) / 2

    @radius.inplace.setter
    def _radius_setter(self, value: float) -> None:
        # for example only
        self.length = value * 2

    @radius.inplace.expression
    @classmethod
    def _radius_expression(cls) -> ColumnElement[float]:
        return type_coerce(func.abs(cls.length) / 2, Float)
```

使用`hybrid_property.inplace`进一步限定了应该不制作新副本的装饰器的使用，从而保持了`Interval.radius`名称，同时允许其他方法`Interval._radius_setter`和`Interval._radius_expression`命名不同。

2.0.4 版中的新功能：添加了`hybrid_property.inplace`，以允许更少冗长的构造复合`hybrid_property`对象，同时无需使用重复的方法名称。此外，允许在`hybrid_property.expression`、`hybrid_property.update_expression`和`hybrid_property.comparator`内使用`@classmethod`，以允许类型工具将`cls`识别为类而不是方法签名中的实例。

## 定义设置器

`hybrid_property.setter()`修饰符允许构造自定义的设置器方法，可以修改对象上的值：

```py
class Interval(Base):
    # ...

    @hybrid_property
    def length(self) -> int:
        return self.end - self.start

    @length.inplace.setter
    def _length_setter(self, value: int) -> None:
        self.end = self.start + value
```

现在，在设置时调用`length(self, value)`方法：

```py
>>> i1 = Interval(5, 10)
>>> i1.length
5
>>> i1.length = 12
>>> i1.end
17
```

## 允许批量 ORM 更新

一个混合可以为启用 ORM 的更新定义自定义的“UPDATE”处理程序，从而允许混合用于更新的 SET 子句中。

通常，在使用带有`update()`的混合时，SQL 表达式被用作作为 SET 目标的列。如果我们的`Interval`类有一个混合`start_point`，它链接到`Interval.start`，这可以直接替换：

```py
from sqlalchemy import update
stmt = update(Interval).values({Interval.start_point: 10})
```

但是，当使用类似`Interval.length`的复合混合类型时，此混合类型表示不止一个列。我们可以设置一个处理程序，该处理程序将适应传递给 VALUES 表达式的值，这可能会影响到这一点，使用`hybrid_property.update_expression()`装饰器。一个类似于我们的设置器的处理程序将是：

```py
from typing import List, Tuple, Any

class Interval(Base):
    # ...

    @hybrid_property
    def length(self) -> int:
        return self.end - self.start

    @length.inplace.setter
    def _length_setter(self, value: int) -> None:
        self.end = self.start + value

    @length.inplace.update_expression
    def _length_update_expression(cls, value: Any) -> List[Tuple[Any, Any]]:
        return [
            (cls.end, cls.start + value)
        ]
```

以上，如果我们在 UPDATE 表达式中使用`Interval.length`，我们将得到一个混合 SET 表达式：

```py
>>> from sqlalchemy import update
>>> print(update(Interval).values({Interval.length: 25}))
UPDATE  interval  SET  "end"=(interval.start  +  :start_1) 
```

这个 SET 表达式会被 ORM 自动处理。

另请参阅

ORM-启用的 INSERT、UPDATE 和 DELETE 语句 - 包括 ORM 启用的 UPDATE 语句的背景信息

## 处理关系

创建与基于列的数据不同的混合对象时，本质上没有区别。对于不同的表达式的需求往往更大。我们将展示的两种变体是“连接依赖”混合和“相关子查询”混合。

### 连接依赖关系混合

考虑以下将 `User` 与 `SavingsAccount` 关联的声明性映射：

```py
from __future__ import annotations

from decimal import Decimal
from typing import cast
from typing import List
from typing import Optional

from sqlalchemy import ForeignKey
from sqlalchemy import Numeric
from sqlalchemy import String
from sqlalchemy import SQLColumnExpression
from sqlalchemy.ext.hybrid import hybrid_property
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

class SavingsAccount(Base):
    __tablename__ = 'account'
    id: Mapped[int] = mapped_column(primary_key=True)
    user_id: Mapped[int] = mapped_column(ForeignKey('user.id'))
    balance: Mapped[Decimal] = mapped_column(Numeric(15, 5))

    owner: Mapped[User] = relationship(back_populates="accounts")

class User(Base):
    __tablename__ = 'user'
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))

    accounts: Mapped[List[SavingsAccount]] = relationship(
        back_populates="owner", lazy="selectin"
    )

    @hybrid_property
    def balance(self) -> Optional[Decimal]:
        if self.accounts:
            return self.accounts[0].balance
        else:
            return None

    @balance.inplace.setter
    def _balance_setter(self, value: Optional[Decimal]) -> None:
        assert value is not None

        if not self.accounts:
            account = SavingsAccount(owner=self)
        else:
            account = self.accounts[0]
        account.balance = value

    @balance.inplace.expression
    @classmethod
    def _balance_expression(cls) -> SQLColumnExpression[Optional[Decimal]]:
        return cast("SQLColumnExpression[Optional[Decimal]]", SavingsAccount.balance)
```

上面的混合属性 `balance` 与此用户的账户列表中的第一个 `SavingsAccount` 条目一起工作。在 Python 中的 getter/setter 方法可以将 `accounts` 视为可在 `self` 上使用的 Python 列表。

提示

在上面的例子中，`User.balance` 的 getter 方法访问了 `self.accounts` 集合，通常会通过配置在 `User.balance` 的 `relationship()` 上的 `selectinload()` 加载策略来加载。当在 `relationship()` 上没有另外指定时，默认的加载策略是 `lazyload()`，它会按需发出 SQL。在使用 asyncio 时，像 `lazyload()` 这样的按需加载器不受支持，因此在使用 asyncio 时，应确保 `self.accounts` 集合对这个混合访问器是可访问的。

在表达式级别，预期 `User` 类将在适当的上下文中使用，以便存在与 `SavingsAccount` 的适当连接：

```py
>>> from sqlalchemy import select
>>> print(select(User, User.balance).
...       join(User.accounts).filter(User.balance > 5000))
SELECT  "user".id  AS  user_id,  "user".name  AS  user_name,
account.balance  AS  account_balance
FROM  "user"  JOIN  account  ON  "user".id  =  account.user_id
WHERE  account.balance  >  :balance_1 
```

但需要注意的是，尽管实例级别的访问器需要担心 `self.accounts` 是否存在，但在 SQL 表达式级别，这个问题表现得不同，我们基本上会使用外连接：

```py
>>> from sqlalchemy import select
>>> from sqlalchemy import or_
>>> print (select(User, User.balance).outerjoin(User.accounts).
...         filter(or_(User.balance < 5000, User.balance == None)))
SELECT  "user".id  AS  user_id,  "user".name  AS  user_name,
account.balance  AS  account_balance
FROM  "user"  LEFT  OUTER  JOIN  account  ON  "user".id  =  account.user_id
WHERE  account.balance  <  :balance_1  OR  account.balance  IS  NULL 
```

### 相关子查询关系混合

当然，我们可以放弃依赖于包含查询中连接的使用，而选择相关子查询，它可以被打包成一个单列表达式。相关子查询更具可移植性，但在 SQL 层面通常性能较差。使用在 使用 column_property 中展示的相同技术，我们可以调整我们的 `SavingsAccount` 示例来聚合*所有*账户的余额，并使用相关子查询作为列表达式：

```py
from __future__ import annotations

from decimal import Decimal
from typing import List

from sqlalchemy import ForeignKey
from sqlalchemy import func
from sqlalchemy import Numeric
from sqlalchemy import select
from sqlalchemy import SQLColumnExpression
from sqlalchemy import String
from sqlalchemy.ext.hybrid import hybrid_property
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

class SavingsAccount(Base):
    __tablename__ = 'account'
    id: Mapped[int] = mapped_column(primary_key=True)
    user_id: Mapped[int] = mapped_column(ForeignKey('user.id'))
    balance: Mapped[Decimal] = mapped_column(Numeric(15, 5))

    owner: Mapped[User] = relationship(back_populates="accounts")

class User(Base):
    __tablename__ = 'user'
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))

    accounts: Mapped[List[SavingsAccount]] = relationship(
        back_populates="owner", lazy="selectin"
    )

    @hybrid_property
    def balance(self) -> Decimal:
        return sum((acc.balance for acc in self.accounts), start=Decimal("0"))

    @balance.inplace.expression
    @classmethod
    def _balance_expression(cls) -> SQLColumnExpression[Decimal]:
        return (
            select(func.sum(SavingsAccount.balance))
            .where(SavingsAccount.user_id == cls.id)
            .label("total_balance")
        )
```

上面的示例将给我们一个 `balance` 列，它呈现一个相关的 SELECT：

```py
>>> from sqlalchemy import select
>>> print(select(User).filter(User.balance > 400))
SELECT  "user".id,  "user".name
FROM  "user"
WHERE  (
  SELECT  sum(account.balance)  AS  sum_1  FROM  account
  WHERE  account.user_id  =  "user".id
)  >  :param_1 
```

## 构建自定义比较器

混合属性还包括一个辅助程序，允许构建自定义比较器。比较器对象允许单独定制每个 SQLAlchemy 表达式操作符的行为。在创建在 SQL 方面具有某些高度特殊行为的自定义类型时很有用。

注意

本节中引入的`hybrid_property.comparator()`装饰器**替换**了`hybrid_property.expression()`装饰器的使用。它们不能一起使用。

下面的示例类允许在名为`word_insensitive`的属性上进行不区分大小写的比较：

```py
from __future__ import annotations

from typing import Any

from sqlalchemy import ColumnElement
from sqlalchemy import func
from sqlalchemy.ext.hybrid import Comparator
from sqlalchemy.ext.hybrid import hybrid_property
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column

class Base(DeclarativeBase):
    pass

class CaseInsensitiveComparator(Comparator[str]):
    def __eq__(self, other: Any) -> ColumnElement[bool]:  # type: ignore[override]  # noqa: E501
        return func.lower(self.__clause_element__()) == func.lower(other)

class SearchWord(Base):
    __tablename__ = 'searchword'

    id: Mapped[int] = mapped_column(primary_key=True)
    word: Mapped[str]

    @hybrid_property
    def word_insensitive(self) -> str:
        return self.word.lower()

    @word_insensitive.inplace.comparator
    @classmethod
    def _word_insensitive_comparator(cls) -> CaseInsensitiveComparator:
        return CaseInsensitiveComparator(cls.word)
```

在上述情况下，针对`word_insensitive`的 SQL 表达式将对两侧应用`LOWER()` SQL 函数：

```py
>>> from sqlalchemy import select
>>> print(select(SearchWord).filter_by(word_insensitive="Trucks"))
SELECT  searchword.id,  searchword.word
FROM  searchword
WHERE  lower(searchword.word)  =  lower(:lower_1) 
```

上述的`CaseInsensitiveComparator`实现了`ColumnOperators`接口的部分内容。可以使用`Operators.operate()`对所有比较操作（即`eq`、`lt`、`gt`等）应用“强制转换”操作，如转换为小写：

```py
class CaseInsensitiveComparator(Comparator):
    def operate(self, op, other, **kwargs):
        return op(
            func.lower(self.__clause_element__()),
            func.lower(other),
            **kwargs,
        )
```  ## 在子类之间重用混合属性

可以从超类中引用混合体，以允许修改方法，如`hybrid_property.getter()`，`hybrid_property.setter()`，以便在子类中重新定义这些方法。这类似于标准 Python 的`@property`对象的工作原理：

```py
class FirstNameOnly(Base):
    # ...

    first_name: Mapped[str]

    @hybrid_property
    def name(self) -> str:
        return self.first_name

    @name.inplace.setter
    def _name_setter(self, value: str) -> None:
        self.first_name = value

class FirstNameLastName(FirstNameOnly):
    # ...

    last_name: Mapped[str]

    # 'inplace' is not used here; calling getter creates a copy
    # of FirstNameOnly.name that is local to FirstNameLastName
    @FirstNameOnly.name.getter
    def name(self) -> str:
        return self.first_name + ' ' + self.last_name

    @name.inplace.setter
    def _name_setter(self, value: str) -> None:
        self.first_name, self.last_name = value.split(' ', 1)
```

在上述情况下，`FirstNameLastName`类引用了从`FirstNameOnly.name`到子类的混合体，以重新利用其 getter 和 setter。

当仅在首次引用超类时覆盖`hybrid_property.expression()`和`hybrid_property.comparator()`作为类级别的第一个引用时，这些名称会与返回类级别的`QueryableAttribute`对象上的同名访问器发生冲突。要在直接引用父类描述符时覆盖这些方法，请添加特殊限定词`hybrid_property.overrides`，该限定词将仪表化的属性引用回混合对象：

```py
class FirstNameLastName(FirstNameOnly):
    # ...

    last_name: Mapped[str]

    @FirstNameOnly.name.overrides.expression
    @classmethod
    def name(cls):
        return func.concat(cls.first_name, ' ', cls.last_name)
```

## 混合值对象

请注意，在我们之前的例子中，如果我们将`SearchWord`实例的`word_insensitive`属性与普通的 Python 字符串进行比较，普通的 Python 字符串不会被强制转换为小写 - 我们构建的`CaseInsensitiveComparator`，由`@word_insensitive.comparator`返回，仅适用于 SQL 端。

自定义比较器的更全面形式是构建一个*混合值对象*。这种技术将目标值或表达式应用于一个值对象，然后由访问器在所有情况下返回。值对象允许控制对值的所有操作，以及如何处理比较的值，无论是在 SQL 表达式方面还是在 Python 值方面。用新的`CaseInsensitiveWord`类替换以前的`CaseInsensitiveComparator`类：

```py
class CaseInsensitiveWord(Comparator):
    "Hybrid value representing a lower case representation of a word."

    def __init__(self, word):
        if isinstance(word, basestring):
            self.word = word.lower()
        elif isinstance(word, CaseInsensitiveWord):
            self.word = word.word
        else:
            self.word = func.lower(word)

    def operate(self, op, other, **kwargs):
        if not isinstance(other, CaseInsensitiveWord):
            other = CaseInsensitiveWord(other)
        return op(self.word, other.word, **kwargs)

    def __clause_element__(self):
        return self.word

    def __str__(self):
        return self.word

    key = 'word'
    "Label to apply to Query tuple results"
```

在上面的例子中，`CaseInsensitiveWord`对象表示`self.word`，它可能是一个 SQL 函数，也可能是一个 Python 本机对象。通过重写`operate()`和`__clause_element__()`以使用`self.word`，所有比较操作将针对“转换”形式的`word`进行，无论是在 SQL 端还是在 Python 端。我们的`SearchWord`类现在可以无条件地从单个混合调用中交付`CaseInsensitiveWord`对象：

```py
class SearchWord(Base):
    __tablename__ = 'searchword'
    id: Mapped[int] = mapped_column(primary_key=True)
    word: Mapped[str]

    @hybrid_property
    def word_insensitive(self) -> CaseInsensitiveWord:
        return CaseInsensitiveWord(self.word)
```

`word_insensitive`属性现在在所有情况下都具有不区分大小写的比较行为，包括 SQL 表达式与 Python 表达式（请注意，此处 Python 值在 Python 端转换为小写）：

```py
>>> print(select(SearchWord).filter_by(word_insensitive="Trucks"))
SELECT  searchword.id  AS  searchword_id,  searchword.word  AS  searchword_word
FROM  searchword
WHERE  lower(searchword.word)  =  :lower_1 
```

SQL 表达式与 SQL 表达式：

```py
>>> from sqlalchemy.orm import aliased
>>> sw1 = aliased(SearchWord)
>>> sw2 = aliased(SearchWord)
>>> print(
...     select(sw1.word_insensitive, sw2.word_insensitive).filter(
...         sw1.word_insensitive > sw2.word_insensitive
...     )
... )
SELECT  lower(searchword_1.word)  AS  lower_1,
lower(searchword_2.word)  AS  lower_2
FROM  searchword  AS  searchword_1,  searchword  AS  searchword_2
WHERE  lower(searchword_1.word)  >  lower(searchword_2.word) 
```

仅 Python 表达式：

```py
>>> ws1 = SearchWord(word="SomeWord")
>>> ws1.word_insensitive == "sOmEwOrD"
True
>>> ws1.word_insensitive == "XOmEwOrX"
False
>>> print(ws1.word_insensitive)
someword
```

混合值模式非常适用于可能具有多种表示形式的任何类型的值，例如时间戳、时间差、测量单位、货币和加密密码。

另请参阅

[混合和值不可知类型](https://techspot.zzzeek.org/2011/10/21/hybrids-and-value-agnostic-types/) - 在 techspot.zzzeek.org 博客上

[值不可知类型，第二部分](https://techspot.zzzeek.org/2011/10/29/value-agnostic-types-part-ii/) - 在 techspot.zzzeek.org 博客上

## API 参考

| 对象名称 | 描述 |
| --- | --- |
| 比较器 | 一个帮助类，允许轻松构建用于混合使用的自定义`PropComparator`类。 |
| hybrid_method | 一个装饰器，允许定义具有实例级和类级行为的 Python 对象方法。 |
| hybrid_property | 一个装饰器，允许定义具有实例级和类级行为的 Python 描述符。 |
| 混合扩展类型 | 一个枚举类型。 |

```py
class sqlalchemy.ext.hybrid.hybrid_method
```

一个装饰器，允许定义具有实例级和类级行为的 Python 对象方法。

**成员**

__init__(), expression(), extension_type, inplace, is_attribute

**类签名**

类`sqlalchemy.ext.hybrid.hybrid_method`（`sqlalchemy.orm.base.InspectionAttrInfo`, `typing.Generic`)

```py
method __init__(func: Callable[[Concatenate[Any, _P]], _R], expr: Callable[[Concatenate[Any, _P]], SQLCoreOperations[_R]] | None = None)
```

创建一个新的`hybrid_method`。

通常使用装饰器：

```py
from sqlalchemy.ext.hybrid import hybrid_method

class SomeClass:
    @hybrid_method
    def value(self, x, y):
        return self._value + x + y

    @value.expression
    @classmethod
    def value(cls, x, y):
        return func.some_function(cls._value, x, y)
```

```py
method expression(expr: Callable[[Concatenate[Any, _P]], SQLCoreOperations[_R]]) → hybrid_method[_P, _R]
```

提供一个修改装饰器，定义一个生成 SQL 表达式的方法。

```py
attribute extension_type: InspectionAttrExtensionType = 'HYBRID_METHOD'
```

扩展类型，如果有的话。默认为`NotExtension.NOT_EXTENSION`

另请参阅

`HybridExtensionType`

`AssociationProxyExtensionType`

```py
attribute inplace
```

返回此`hybrid_method`的原地变异器。

当调用`hybrid_method.expression()`装饰器时，`hybrid_method`类已经执行“原地”变异，因此此属性返回 Self。

版本 2.0.4 中的新功能。

另请参阅

使用 inplace 创建符合 pep-484 的混合属性

```py
attribute is_attribute = True
```

如果这个对象是一个 Python 描述符，则为 True。

这可以指代许多类型之一。通常是一个处理属性事件的`QueryableAttribute`，代表一个`MapperProperty`。但也可以是一个扩展类型，如`AssociationProxy`或`hybrid_property`。`InspectionAttr.extension_type`将指代一个标识特定子类型的常量。

另请参阅

`Mapper.all_orm_descriptors`

```py
class sqlalchemy.ext.hybrid.hybrid_property
```

一个允许定义具有实例级和类级行为的 Python 描述符的装饰器。

**成员**

__init__(), comparator(), deleter(), expression(), extension_type, getter(), inplace, is_attribute, overrides, setter(), update_expression()

**类签名**

类`sqlalchemy.ext.hybrid.hybrid_property`（`sqlalchemy.orm.base.InspectionAttrInfo`, `sqlalchemy.orm.base.ORMDescriptor`)

```py
method __init__(fget: _HybridGetterType[_T], fset: _HybridSetterType[_T] | None = None, fdel: _HybridDeleterType[_T] | None = None, expr: _HybridExprCallableType[_T] | None = None, custom_comparator: Comparator[_T] | None = None, update_expr: _HybridUpdaterType[_T] | None = None)
```

创建一个新的`hybrid_property`。

通常使用修饰器：

```py
from sqlalchemy.ext.hybrid import hybrid_property

class SomeClass:
    @hybrid_property
    def value(self):
        return self._value

    @value.setter
    def value(self, value):
        self._value = value
```

```py
method comparator(comparator: _HybridComparatorCallableType[_T]) → hybrid_property[_T]
```

提供一个修饰器，定义一个自定义比较器生成方法。

被修饰方法的返回值应该是`Comparator`的一个实例。

注意

`hybrid_property.comparator()`修饰器**替代**了`hybrid_property.expression()`修饰器的使用。它们不能同时使用。

当在类级别调用混合属性时，这里给出的`Comparator`对象被包装在一个专门的`QueryableAttribute`中，这是 ORM 用来表示其他映射属性的对象。这样做的原因是为了在返回的结构中保留其他类级别属性，如文档字符串和对混合属性本身的引用，而不对传入的原始比较器对象进行任何修改。

注意

当引用拥有类（例如 `SomeClass.some_hybrid`）的混合属性时，会返回一个`QueryableAttribute`的实例，表示此混合对象的表达式或比较器对象。但是，该对象本身具有名为 `expression` 和 `comparator` 的访问器；因此，在尝试在子类上覆盖这些装饰器时，可能需要首先使用 `hybrid_property.overrides` 修饰符进行限定。有关详细信息，请参阅该修饰符。

```py
method deleter(fdel: _HybridDeleterType[_T]) → hybrid_property[_T]
```

提供一个修改装饰器，定义一个删除方法。

```py
method expression(expr: _HybridExprCallableType[_T]) → hybrid_property[_T]
```

提供一个修改装饰器，定义一个生成 SQL 表达式的方法。

当在类级别调用混合时，此处给出的 SQL 表达式将包装在一个专门的 `QueryableAttribute` 内，这是 ORM 用于表示其他映射属性的相同类型的对象。这样做的原因是为了在返回的结构中维护其他类级别属性，例如文档字符串和混合本身的引用，而不对传入的原始 SQL 表达式进行任何修改。

注意

当引用拥有类（例如 `SomeClass.some_hybrid`）的混合属性时，会返回一个`QueryableAttribute`的实例，表示表达式或比较器对象以及此混合对象。但是，该对象本身具有名为 `expression` 和 `comparator` 的访问器；因此，在尝试在子类上覆盖这些装饰器时，可能需要首先使用 `hybrid_property.overrides` 修饰符进行限定。有关详细信息，请参阅该修饰符。

请参见

定义与属性行为不同的表达行为

```py
attribute extension_type: InspectionAttrExtensionType = 'HYBRID_PROPERTY'
```

扩展类型，如果有的话。默认为 `NotExtension.NOT_EXTENSION`

请参见

`HybridExtensionType`

`AssociationProxyExtensionType`

```py
method getter(fget: _HybridGetterType[_T]) → hybrid_property[_T]
```

提供一个修改装饰器，定义一个获取器方法。

自 1.2 版开始新添加。

```py
attribute inplace
```

返回此 `hybrid_property` 的原地变异器。

这是为了允许对混合进行就地变异，允许重用特定名称的第一个混合方法以添加更多方法，而不必将这些方法命名为相同的名称，例如：

```py
class Interval(Base):
    # ...

    @hybrid_property
    def radius(self) -> float:
        return abs(self.length) / 2

    @radius.inplace.setter
    def _radius_setter(self, value: float) -> None:
        self.length = value * 2

    @radius.inplace.expression
    def _radius_expression(cls) -> ColumnElement[float]:
        return type_coerce(func.abs(cls.length) / 2, Float)
```

自 2.0.4 版开始新添加。

请参见

使用 inplace 创建符合 pep-484 的混合属性

```py
attribute is_attribute = True
```

如果此对象是 Python 的 描述符，则为 True。

这可能是多种类型之一。通常是一个`QueryableAttribute`，它代表一个`MapperProperty`上的属性事件。但也可以是诸如`AssociationProxy`或`hybrid_property`之类的扩展类型。`InspectionAttr.extension_type` 将引用一个常量，以识别特定的子类型。

另请参阅

`Mapper.all_orm_descriptors`

```py
attribute overrides
```

重写现有属性的方法的前缀。

`hybrid_property.overrides` 访问器只是返回这个混合对象，当在父类的类级别从父类调用时，它将取消引用通常在此级别返回的“instrumented attribute”，并允许修改装饰器，例如 `hybrid_property.expression()` 和 `hybrid_property.comparator()` 被使用而不与通常存在于`QueryableAttribute`上的同名属性冲突：

```py
class SuperClass:
    # ...

    @hybrid_property
    def foobar(self):
        return self._foobar

class SubClass(SuperClass):
    # ...

    @SuperClass.foobar.overrides.expression
    def foobar(cls):
        return func.subfoobar(self._foobar)
```

1.2 版的新功能。

另请参阅

在子类中重用混合属性

```py
method setter(fset: _HybridSetterType[_T]) → hybrid_property[_T]
```

提供一个定义 setter 方法的修改装饰器。

```py
method update_expression(meth: _HybridUpdaterType[_T]) → hybrid_property[_T]
```

提供一个定义产生 UPDATE 元组的修改装饰器方法。

该方法接受一个值，该值将被渲染到 UPDATE 语句的 SET 子句中。然后，该方法应将此值处理为适合最终 SET 子句的单个列表达式，并将它们作为 2 元组的序列返回。每个元组包含一个列表达式作为键和要渲染的值。

例如：

```py
class Person(Base):
    # ...

    first_name = Column(String)
    last_name = Column(String)

    @hybrid_property
    def fullname(self):
        return first_name + " " + last_name

    @fullname.update_expression
    def fullname(cls, value):
        fname, lname = value.split(" ", 1)
        return [
            (cls.first_name, fname),
            (cls.last_name, lname)
        ]
```

1.2 版的新功能。

```py
class sqlalchemy.ext.hybrid.Comparator
```

一个辅助类，允许轻松构建自定义`PropComparator`类以与混合一起使用。

**类签名**

类`sqlalchemy.ext.hybrid.Comparator` (`sqlalchemy.orm.PropComparator`)

```py
class sqlalchemy.ext.hybrid.HybridExtensionType
```

一个枚举。

**成员**

HYBRID_METHOD, HYBRID_PROPERTY

**类签名**

类`sqlalchemy.ext.hybrid.HybridExtensionType` (`sqlalchemy.orm.base.InspectionAttrExtensionType`)

```py
attribute HYBRID_METHOD = 'HYBRID_METHOD'
```

表示一个`InspectionAttr`的符号，其类型为`hybrid_method`。

被赋予`InspectionAttr.extension_type`属性。

另请参阅

`Mapper.all_orm_attributes`

```py
attribute HYBRID_PROPERTY = 'HYBRID_PROPERTY'
```

表示一个`InspectionAttr`的符号

类型为`hybrid_method`。

被赋予`InspectionAttr.extension_type`属性。

另请参阅

`Mapper.all_orm_attributes`

## 定义与属性行为不同的表达行为

在前一节中，我们在`Interval.contains`和`Interval.intersects`方法中使用`&`和`|`位运算符是幸运的，考虑到我们的函数操作两个布尔值以返回一个新值。在许多情况下，一个在 Python 函数和 SQLAlchemy SQL 表达式之间有足够差异的情况下，应该定义两个单独的 Python 表达式。`hybrid`装饰器为此目的定义了一个**修饰符**`hybrid_property.expression()`。作为示例，我们将定义间隔的半径，这需要使用绝对值函数：

```py
from sqlalchemy import ColumnElement
from sqlalchemy import Float
from sqlalchemy import func
from sqlalchemy import type_coerce

class Interval(Base):
    # ...

    @hybrid_property
    def radius(self) -> float:
        return abs(self.length) / 2

    @radius.inplace.expression
    @classmethod
    def _radius_expression(cls) -> ColumnElement[float]:
        return type_coerce(func.abs(cls.length) / 2, Float)
```

在上面的示例中，首先将 `hybrid_property` 分配给名称 `Interval.radius`，然后通过后续调用名为 `Interval._radius_expression` 的方法，使用装饰器 `@radius.inplace.expression` 对其进行修改，该装饰器链接了两个修饰符 `hybrid_property.inplace` 和 `hybrid_property.expression`。使用 `hybrid_property.inplace` 表示 `hybrid_property.expression()` 修饰符应在原地改变 `Interval.radius` 中的现有混合对象，而不创建新对象。关于此修饰符及其基本原理的说明在下一节中讨论 使用 inplace 创建符合 pep-484 的混合属性。使用 `@classmethod` 是可选的，并且严格来说是为了给出类型提示，以表明在这种情况下，`cls` 预期是 `Interval` 类，而不是 `Interval` 的实例。

注意

`hybrid_property.inplace` 以及使用 `@classmethod` 以获得正确的类型支持在 SQLAlchemy 2.0.4 中可用，并且在早期版本中不起作用。

现在，由于 `Interval.radius` 现在包含表达式元素，因此在访问类级别的 `Interval.radius` 时会返回 SQL 函数 `ABS()`：

```py
>>> from sqlalchemy import select
>>> print(select(Interval).filter(Interval.radius > 5))
SELECT  interval.id,  interval.start,  interval."end"
FROM  interval
WHERE  abs(interval."end"  -  interval.start)  /  :abs_1  >  :param_1 
```

## 使用 `inplace` 创建符合 pep-484 的混合属性

在前一节中，演示了一个 `hybrid_property` 装饰器，其中包括两个单独的方法级函数被装饰，都用于生成一个名为 `Interval.radius` 的单个对象属性。实际上，我们可以使用几种不同的修饰符来使用 `hybrid_property`，包括 `hybrid_property.expression()`、`hybrid_property.setter()` 和 `hybrid_property.update_expression()`。

SQLAlchemy 的 `hybrid_property` 装饰器打算通过与 Python 内置的 `@property` 装饰器相同的方式添加这些方法，其中惯用法是重复重新定义属性，每次使用相同的属性名称，如下面的示例所示，演示了使用 `hybrid_property.setter()` 和 `hybrid_property.expression()` 为 `Interval.radius` 描述符的用法：

```py
# correct use, however is not accepted by pep-484 tooling

class Interval(Base):
    # ...

    @hybrid_property
    def radius(self):
        return abs(self.length) / 2

    @radius.setter
    def radius(self, value):
        self.length = value * 2

    @radius.expression
    def radius(cls):
        return type_coerce(func.abs(cls.length) / 2, Float)
```

如上，有三个 `Interval.radius` 方法，但是由于每个都被 `hybrid_property` 装饰器和 `@radius` 名称本身装饰，最终的效果是 `Interval.radius` 是一个包含三个不同函数的单个属性。这种用法风格来自于 [Python 的@property 的文档用法](https://docs.python.org/3/library/functions.html#property)。需要注意的是，无论是 `@property` 还是 `hybrid_property` 的工作方式，每次都会 **复制描述符**。也就是说，每次调用 `@radius.expression`、`@radius.setter` 等都会完全创建一个新的对象。这使得属性在子类中重新定义时不会出现问题（请参阅本节稍后的 在子类之间重用混合属性 来了解如何使用）。

然而，上述方法不兼容于诸如 mypy 和 pyright 等类型工具。Python 自己的 `@property` 装饰器之所以没有这个限制，仅仅是因为 [这些工具硬编码了@property 的行为](https://github.com/python/typing/discussions/1102)，这意味着这种语法在 SQLAlchemy 下不符合 [**PEP 484**](https://peps.python.org/pep-0484/) 的规范。

为了在保持打字兼容的同时产生合理的语法，`hybrid_property.inplace` 装饰器允许同一装饰器以不同的方法名称被重复使用，同时仍然产生一个单一的装饰器在一个名称下：

```py
# correct use which is also accepted by pep-484 tooling

class Interval(Base):
    # ...

    @hybrid_property
    def radius(self) -> float:
        return abs(self.length) / 2

    @radius.inplace.setter
    def _radius_setter(self, value: float) -> None:
        # for example only
        self.length = value * 2

    @radius.inplace.expression
    @classmethod
    def _radius_expression(cls) -> ColumnElement[float]:
        return type_coerce(func.abs(cls.length) / 2, Float)
```

使用 `hybrid_property.inplace` 进一步限定了装饰器的使用，不应该创建一个新的副本，从而保持了 `Interval.radius` 名称，同时允许额外的方法 `Interval._radius_setter` 和 `Interval._radius_expression` 使用不同的名称。

版本 2.0.4 中的新功能：添加`hybrid_property.inplace`，以允许更简洁地构建复合`hybrid_property`对象，同时不必重复使用方法名称。此外，允许在`hybrid_property.expression`、`hybrid_property.update_expression`和`hybrid_property.comparator`中使用`@classmethod`，以便允许类型工具将`cls`识别为类而不是方法签名中的实例。

## 定义 Setter

`hybrid_property.setter()`修饰符允许构建自定义 setter 方法，可以修改对象上的值：

```py
class Interval(Base):
    # ...

    @hybrid_property
    def length(self) -> int:
        return self.end - self.start

    @length.inplace.setter
    def _length_setter(self, value: int) -> None:
        self.end = self.start + value
```

现在，在设置时调用`length(self, value)`方法：

```py
>>> i1 = Interval(5, 10)
>>> i1.length
5
>>> i1.length = 12
>>> i1.end
17
```

## 允许批量 ORM 更新

当使用 ORM 启用的更新时，混合类型可以为自定义的“UPDATE”处理程序定义处理程序，允许将混合类型用于更新的 SET 子句中。

通常，当使用`update()`与混合类型时，SQL 表达式将用作 SET 的目标列。如果我们的`Interval`类具有链接到`Interval.start`的混合类型`start_point`，则可以直接替换： 

```py
from sqlalchemy import update
stmt = update(Interval).values({Interval.start_point: 10})
```

然而，当使用像`Interval.length`这样的复合混合类型时，这个混合类型代表多于一个列。我们可以设置一个处理程序，该处理程序将适应传递到 VALUES 表达式中的值，这可能会影响此处理程序，使用`hybrid_property.update_expression()`装饰器。一个与我们的 setter 类似的处理程序将是：

```py
from typing import List, Tuple, Any

class Interval(Base):
    # ...

    @hybrid_property
    def length(self) -> int:
        return self.end - self.start

    @length.inplace.setter
    def _length_setter(self, value: int) -> None:
        self.end = self.start + value

    @length.inplace.update_expression
    def _length_update_expression(cls, value: Any) -> List[Tuple[Any, Any]]:
        return [
            (cls.end, cls.start + value)
        ]
```

如果我们在 UPDATE 表达式中使用`Interval.length`，我们将获得一个混合类型的 SET 表达式：

```py
>>> from sqlalchemy import update
>>> print(update(Interval).values({Interval.length: 25}))
UPDATE  interval  SET  "end"=(interval.start  +  :start_1) 
```

此 SET 表达式将由 ORM 自动处理。

另请参阅

ORM 启用的 INSERT、UPDATE 和 DELETE 语句 - 包括 ORM 启用的 UPDATE 语句的背景信息

## 与关系一起工作

创建与基于列的数据相反的与相关对象一起工作的混合类型时，没有本质区别。对于不同的表达式的需求往往更大。我们将说明的两种变体是“join-dependent”混合类型和“correlated subquery”混合类型。

### Join-Dependent Relationship Hybrid

考虑以下将`User`与`SavingsAccount`相关联的声明性映射：

```py
from __future__ import annotations

from decimal import Decimal
from typing import cast
from typing import List
from typing import Optional

from sqlalchemy import ForeignKey
from sqlalchemy import Numeric
from sqlalchemy import String
from sqlalchemy import SQLColumnExpression
from sqlalchemy.ext.hybrid import hybrid_property
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

class SavingsAccount(Base):
    __tablename__ = 'account'
    id: Mapped[int] = mapped_column(primary_key=True)
    user_id: Mapped[int] = mapped_column(ForeignKey('user.id'))
    balance: Mapped[Decimal] = mapped_column(Numeric(15, 5))

    owner: Mapped[User] = relationship(back_populates="accounts")

class User(Base):
    __tablename__ = 'user'
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))

    accounts: Mapped[List[SavingsAccount]] = relationship(
        back_populates="owner", lazy="selectin"
    )

    @hybrid_property
    def balance(self) -> Optional[Decimal]:
        if self.accounts:
            return self.accounts[0].balance
        else:
            return None

    @balance.inplace.setter
    def _balance_setter(self, value: Optional[Decimal]) -> None:
        assert value is not None

        if not self.accounts:
            account = SavingsAccount(owner=self)
        else:
            account = self.accounts[0]
        account.balance = value

    @balance.inplace.expression
    @classmethod
    def _balance_expression(cls) -> SQLColumnExpression[Optional[Decimal]]:
        return cast("SQLColumnExpression[Optional[Decimal]]", SavingsAccount.balance)
```

上述混合属性`balance`与此用户的账户列表中的第一个`SavingsAccount`条目配合使用。在 Python 中，getter/setter 方法可以将`accounts`视为`self`上可用的 Python 列表。

提示

上述示例中的`User.balance` getter 访问`self.acccounts`集合，通常会通过配置在`User.balance` `relationship()`上的`selectinload()`加载策略加载。当未在`relationship()`上另行说明时，默认加载策略是`lazyload()`，它会按需发出 SQL。在使用 asyncio 时，不支持按需加载程序，因此在使用 asyncio 时，应确保`self.accounts`集合对此混合访问器可访问。

在表达式级别，预期`User`类将在适当的上下文中使用，以便存在适当的连接到`SavingsAccount`：

```py
>>> from sqlalchemy import select
>>> print(select(User, User.balance).
...       join(User.accounts).filter(User.balance > 5000))
SELECT  "user".id  AS  user_id,  "user".name  AS  user_name,
account.balance  AS  account_balance
FROM  "user"  JOIN  account  ON  "user".id  =  account.user_id
WHERE  account.balance  >  :balance_1 
```

但请注意，尽管实例级访问器需要担心`self.accounts`是否存在，但在 SQL 表达式级别，这个问题表现得不同，基本上我们会使用外连接：

```py
>>> from sqlalchemy import select
>>> from sqlalchemy import or_
>>> print (select(User, User.balance).outerjoin(User.accounts).
...         filter(or_(User.balance < 5000, User.balance == None)))
SELECT  "user".id  AS  user_id,  "user".name  AS  user_name,
account.balance  AS  account_balance
FROM  "user"  LEFT  OUTER  JOIN  account  ON  "user".id  =  account.user_id
WHERE  account.balance  <  :balance_1  OR  account.balance  IS  NULL 
```

### 相关子查询关系混合

当然，我们可以放弃依赖于连接的查询使用，转而使用相关子查询，这可以方便地打包到单个列表达式中。相关子查询更具可移植性，但在 SQL 级别通常性能较差。使用与使用 column_property 中所示技术相同的技术，我们可以调整我们的`SavingsAccount`示例以聚合*所有*账户的余额，并使用相关子查询作为列表达式：

```py
from __future__ import annotations

from decimal import Decimal
from typing import List

from sqlalchemy import ForeignKey
from sqlalchemy import func
from sqlalchemy import Numeric
from sqlalchemy import select
from sqlalchemy import SQLColumnExpression
from sqlalchemy import String
from sqlalchemy.ext.hybrid import hybrid_property
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

class SavingsAccount(Base):
    __tablename__ = 'account'
    id: Mapped[int] = mapped_column(primary_key=True)
    user_id: Mapped[int] = mapped_column(ForeignKey('user.id'))
    balance: Mapped[Decimal] = mapped_column(Numeric(15, 5))

    owner: Mapped[User] = relationship(back_populates="accounts")

class User(Base):
    __tablename__ = 'user'
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))

    accounts: Mapped[List[SavingsAccount]] = relationship(
        back_populates="owner", lazy="selectin"
    )

    @hybrid_property
    def balance(self) -> Decimal:
        return sum((acc.balance for acc in self.accounts), start=Decimal("0"))

    @balance.inplace.expression
    @classmethod
    def _balance_expression(cls) -> SQLColumnExpression[Decimal]:
        return (
            select(func.sum(SavingsAccount.balance))
            .where(SavingsAccount.user_id == cls.id)
            .label("total_balance")
        )
```

上述配方将为我们提供呈现相关 SELECT 的`balance`列：

```py
>>> from sqlalchemy import select
>>> print(select(User).filter(User.balance > 400))
SELECT  "user".id,  "user".name
FROM  "user"
WHERE  (
  SELECT  sum(account.balance)  AS  sum_1  FROM  account
  WHERE  account.user_id  =  "user".id
)  >  :param_1 
```

### 连接依赖关系混合

考虑以下声明性映射，将`User`与`SavingsAccount`相关联：

```py
from __future__ import annotations

from decimal import Decimal
from typing import cast
from typing import List
from typing import Optional

from sqlalchemy import ForeignKey
from sqlalchemy import Numeric
from sqlalchemy import String
from sqlalchemy import SQLColumnExpression
from sqlalchemy.ext.hybrid import hybrid_property
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

class SavingsAccount(Base):
    __tablename__ = 'account'
    id: Mapped[int] = mapped_column(primary_key=True)
    user_id: Mapped[int] = mapped_column(ForeignKey('user.id'))
    balance: Mapped[Decimal] = mapped_column(Numeric(15, 5))

    owner: Mapped[User] = relationship(back_populates="accounts")

class User(Base):
    __tablename__ = 'user'
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))

    accounts: Mapped[List[SavingsAccount]] = relationship(
        back_populates="owner", lazy="selectin"
    )

    @hybrid_property
    def balance(self) -> Optional[Decimal]:
        if self.accounts:
            return self.accounts[0].balance
        else:
            return None

    @balance.inplace.setter
    def _balance_setter(self, value: Optional[Decimal]) -> None:
        assert value is not None

        if not self.accounts:
            account = SavingsAccount(owner=self)
        else:
            account = self.accounts[0]
        account.balance = value

    @balance.inplace.expression
    @classmethod
    def _balance_expression(cls) -> SQLColumnExpression[Optional[Decimal]]:
        return cast("SQLColumnExpression[Optional[Decimal]]", SavingsAccount.balance)
```

上述混合属性`balance`与此用户的账户列表中的第一个`SavingsAccount`条目配合使用。在 Python 中，getter/setter 方法可以将`accounts`视为`self`上可用的 Python 列表。

提示

上面示例中的`User.balance` getter 访问了`self.acccounts`集合，这通常会通过在`User.balance` `relationship()`上配置的`selectinload()`加载器策略进行加载。当没有在其他地方声明`relationship()`时，默认的加载器策略是`lazyload()`，它按需发出 SQL。当使用 asyncio 时，不支持按需加载器，如`lazyload()`，因此在使用 asyncio 时应注意确保`self.accounts`集合对此混合访问器是可访问的。

在表达式级别，预计`User`类将在适当的上下文中使用，以便存在适当的连接到`SavingsAccount`：

```py
>>> from sqlalchemy import select
>>> print(select(User, User.balance).
...       join(User.accounts).filter(User.balance > 5000))
SELECT  "user".id  AS  user_id,  "user".name  AS  user_name,
account.balance  AS  account_balance
FROM  "user"  JOIN  account  ON  "user".id  =  account.user_id
WHERE  account.balance  >  :balance_1 
```

但请注意，尽管实例级别的访问器需要担心`self.accounts`是否存在，但这个问题在 SQL 表达式级别上表现得不同，我们基本上会使用外连接：

```py
>>> from sqlalchemy import select
>>> from sqlalchemy import or_
>>> print (select(User, User.balance).outerjoin(User.accounts).
...         filter(or_(User.balance < 5000, User.balance == None)))
SELECT  "user".id  AS  user_id,  "user".name  AS  user_name,
account.balance  AS  account_balance
FROM  "user"  LEFT  OUTER  JOIN  account  ON  "user".id  =  account.user_id
WHERE  account.balance  <  :balance_1  OR  account.balance  IS  NULL 
```

### 关联子查询关系混合

当然，我们可以放弃依赖包含查询中的连接，而倾向于关联子查询，这可以被封装成一个单列表达式。关联子查询更具可移植性，但在 SQL 级别上通常性能较差。使用在使用 column_property 中说明的相同技术，我们可以调整我们的`SavingsAccount`示例来聚合*所有*账户的余额，并为列表达式使用关联子查询：

```py
from __future__ import annotations

from decimal import Decimal
from typing import List

from sqlalchemy import ForeignKey
from sqlalchemy import func
from sqlalchemy import Numeric
from sqlalchemy import select
from sqlalchemy import SQLColumnExpression
from sqlalchemy import String
from sqlalchemy.ext.hybrid import hybrid_property
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

class SavingsAccount(Base):
    __tablename__ = 'account'
    id: Mapped[int] = mapped_column(primary_key=True)
    user_id: Mapped[int] = mapped_column(ForeignKey('user.id'))
    balance: Mapped[Decimal] = mapped_column(Numeric(15, 5))

    owner: Mapped[User] = relationship(back_populates="accounts")

class User(Base):
    __tablename__ = 'user'
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))

    accounts: Mapped[List[SavingsAccount]] = relationship(
        back_populates="owner", lazy="selectin"
    )

    @hybrid_property
    def balance(self) -> Decimal:
        return sum((acc.balance for acc in self.accounts), start=Decimal("0"))

    @balance.inplace.expression
    @classmethod
    def _balance_expression(cls) -> SQLColumnExpression[Decimal]:
        return (
            select(func.sum(SavingsAccount.balance))
            .where(SavingsAccount.user_id == cls.id)
            .label("total_balance")
        )
```

上面的配方将为我们提供一个渲染关联 SELECT 的`balance`列：

```py
>>> from sqlalchemy import select
>>> print(select(User).filter(User.balance > 400))
SELECT  "user".id,  "user".name
FROM  "user"
WHERE  (
  SELECT  sum(account.balance)  AS  sum_1  FROM  account
  WHERE  account.user_id  =  "user".id
)  >  :param_1 
```

## 构建自定义比较器

混合属性还包括一个助手，允许构建自定义比较器。比较器对象允许用户单独定制每个 SQLAlchemy 表达式操作符的行为。当创建具有一些高度特殊的 SQL 端行为的自定义类型时，它们非常有用。

注意

此部分介绍的`hybrid_property.comparator()`装饰器**替换了**`hybrid_property.expression()`装饰器的使用。它们不能一起使用。

下面的示例类允许在名为`word_insensitive`的属性上进行不区分大小写的比较：

```py
from __future__ import annotations

from typing import Any

from sqlalchemy import ColumnElement
from sqlalchemy import func
from sqlalchemy.ext.hybrid import Comparator
from sqlalchemy.ext.hybrid import hybrid_property
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column

class Base(DeclarativeBase):
    pass

class CaseInsensitiveComparator(Comparator[str]):
    def __eq__(self, other: Any) -> ColumnElement[bool]:  # type: ignore[override]  # noqa: E501
        return func.lower(self.__clause_element__()) == func.lower(other)

class SearchWord(Base):
    __tablename__ = 'searchword'

    id: Mapped[int] = mapped_column(primary_key=True)
    word: Mapped[str]

    @hybrid_property
    def word_insensitive(self) -> str:
        return self.word.lower()

    @word_insensitive.inplace.comparator
    @classmethod
    def _word_insensitive_comparator(cls) -> CaseInsensitiveComparator:
        return CaseInsensitiveComparator(cls.word)
```

上面，针对`word_insensitive`的 SQL 表达式将对双方都应用`LOWER()` SQL 函数：

```py
>>> from sqlalchemy import select
>>> print(select(SearchWord).filter_by(word_insensitive="Trucks"))
SELECT  searchword.id,  searchword.word
FROM  searchword
WHERE  lower(searchword.word)  =  lower(:lower_1) 
```

上面的`CaseInsensitiveComparator`实现了`ColumnOperators`接口的一部分。像小写化这样的“强制转换”操作可以应用于所有比较操作（即`eq`、`lt`、`gt`等）使用`Operators.operate()`：

```py
class CaseInsensitiveComparator(Comparator):
    def operate(self, op, other, **kwargs):
        return op(
            func.lower(self.__clause_element__()),
            func.lower(other),
            **kwargs,
        )
```

## 在子类之间重用混合属性

可以从超类中引用混合，以允许修改方法，如`hybrid_property.getter()`、`hybrid_property.setter()`等，用于在子类上重新定义这些方法。这类似于标准的 Python `@property`对象的工作方式：

```py
class FirstNameOnly(Base):
    # ...

    first_name: Mapped[str]

    @hybrid_property
    def name(self) -> str:
        return self.first_name

    @name.inplace.setter
    def _name_setter(self, value: str) -> None:
        self.first_name = value

class FirstNameLastName(FirstNameOnly):
    # ...

    last_name: Mapped[str]

    # 'inplace' is not used here; calling getter creates a copy
    # of FirstNameOnly.name that is local to FirstNameLastName
    @FirstNameOnly.name.getter
    def name(self) -> str:
        return self.first_name + ' ' + self.last_name

    @name.inplace.setter
    def _name_setter(self, value: str) -> None:
        self.first_name, self.last_name = value.split(' ', 1)
```

上面，`FirstNameLastName`类引用了从`FirstNameOnly.name`到混合的混合，以重新用于子类的 getter 和 setter。

当仅覆盖`hybrid_property.expression()`和`hybrid_property.comparator()`作为对超类的第一引用时，这些名称与在类级别返回的类级别`QueryableAttribute`对象上的同名访问器发生冲突。要在直接引用父类描述符时覆盖这些方法，请添加特殊限定符`hybrid_property.overrides`，它将将被仪器化的属性反向引用回混合对象：

```py
class FirstNameLastName(FirstNameOnly):
    # ...

    last_name: Mapped[str]

    @FirstNameOnly.name.overrides.expression
    @classmethod
    def name(cls):
        return func.concat(cls.first_name, ' ', cls.last_name)
```

## 混合值对象

请注意，在我们之前的示例中，如果我们将`SearchWord`实例的`word_insensitive`属性与普通的 Python 字符串进行比较，普通的 Python 字符串不会被强制转换为小写 - 我们构建的`CaseInsensitiveComparator`，由`@word_insensitive.comparator`返回，仅适用于 SQL 端。

自定义比较器的更全面形式是构建一个*混合值对象*。这种技术将目标值或表达式应用于一个值对象，然后该值对象在所有情况下由访问器返回。值对象允许控制对值的所有操作以及如何处理比较值，无论是在 SQL 表达式端还是 Python 值端。用新的`CaseInsensitiveWord`类替换之前的`CaseInsensitiveComparator`类：

```py
class CaseInsensitiveWord(Comparator):
    "Hybrid value representing a lower case representation of a word."

    def __init__(self, word):
        if isinstance(word, basestring):
            self.word = word.lower()
        elif isinstance(word, CaseInsensitiveWord):
            self.word = word.word
        else:
            self.word = func.lower(word)

    def operate(self, op, other, **kwargs):
        if not isinstance(other, CaseInsensitiveWord):
            other = CaseInsensitiveWord(other)
        return op(self.word, other.word, **kwargs)

    def __clause_element__(self):
        return self.word

    def __str__(self):
        return self.word

    key = 'word'
    "Label to apply to Query tuple results"
```

在上文中，`CaseInsensitiveWord` 对象表示 `self.word`，它可能是一个 SQL 函数，也可能是一个 Python 本地函数。通过重写 `operate()` 和 `__clause_element__()` 方法，以 `self.word` 为基础进行操作，所有比较操作都将针对 `word` 的“转换”形式进行，无论是在 SQL 还是 Python 方面。我们的 `SearchWord` 类现在可以通过单一的混合调用无条件地提供 `CaseInsensitiveWord` 对象：

```py
class SearchWord(Base):
    __tablename__ = 'searchword'
    id: Mapped[int] = mapped_column(primary_key=True)
    word: Mapped[str]

    @hybrid_property
    def word_insensitive(self) -> CaseInsensitiveWord:
        return CaseInsensitiveWord(self.word)
```

`word_insensitive` 属性现在具有普遍的不区分大小写的比较行为，包括 SQL 表达式与 Python 表达式（请注意此处的 Python 值在 Python 侧被转换为小写）：

```py
>>> print(select(SearchWord).filter_by(word_insensitive="Trucks"))
SELECT  searchword.id  AS  searchword_id,  searchword.word  AS  searchword_word
FROM  searchword
WHERE  lower(searchword.word)  =  :lower_1 
```

SQL 表达式与 SQL 表达式：

```py
>>> from sqlalchemy.orm import aliased
>>> sw1 = aliased(SearchWord)
>>> sw2 = aliased(SearchWord)
>>> print(
...     select(sw1.word_insensitive, sw2.word_insensitive).filter(
...         sw1.word_insensitive > sw2.word_insensitive
...     )
... )
SELECT  lower(searchword_1.word)  AS  lower_1,
lower(searchword_2.word)  AS  lower_2
FROM  searchword  AS  searchword_1,  searchword  AS  searchword_2
WHERE  lower(searchword_1.word)  >  lower(searchword_2.word) 
```

Python 只有表达式：

```py
>>> ws1 = SearchWord(word="SomeWord")
>>> ws1.word_insensitive == "sOmEwOrD"
True
>>> ws1.word_insensitive == "XOmEwOrX"
False
>>> print(ws1.word_insensitive)
someword
```

混合值模式对于任何可能具有多个表示形式的值非常有用，例如时间戳、时间间隔、测量单位、货币和加密密码等。

另请参阅

[混合类型和值无关类型](https://techspot.zzzeek.org/2011/10/21/hybrids-and-value-agnostic-types/) - 在 techspot.zzzeek.org 博客上

[值无关类型，第二部分](https://techspot.zzzeek.org/2011/10/29/value-agnostic-types-part-ii/) - 在 techspot.zzzeek.org 博客上

## API 参考

| 对象名称 | 描述 |
| --- | --- |
| Comparator | 一个辅助类，允许轻松构建用于混合类型的自定义 `PropComparator` 类。 |
| hybrid_method | 允许定义具有实例级和类级行为的 Python 对象方法的装饰器。 |
| hybrid_property | 允许定义具有实例级和类级行为的 Python 描述符的装饰器。 |
| HybridExtensionType | 枚举类型。 |

```py
class sqlalchemy.ext.hybrid.hybrid_method
```

允许定义具有实例级和类级行为的 Python 对象方法的装饰器。

**成员**

__init__(), expression(), extension_type, inplace, is_attribute

**类签名**

class `sqlalchemy.ext.hybrid.hybrid_method` (`sqlalchemy.orm.base.InspectionAttrInfo`, `typing.Generic`)

```py
method __init__(func: Callable[[Concatenate[Any, _P]], _R], expr: Callable[[Concatenate[Any, _P]], SQLCoreOperations[_R]] | None = None)
```

创建一个新的 `hybrid_method`。

通常使用装饰器：

```py
from sqlalchemy.ext.hybrid import hybrid_method

class SomeClass:
    @hybrid_method
    def value(self, x, y):
        return self._value + x + y

    @value.expression
    @classmethod
    def value(cls, x, y):
        return func.some_function(cls._value, x, y)
```

```py
method expression(expr: Callable[[Concatenate[Any, _P]], SQLCoreOperations[_R]]) → hybrid_method[_P, _R]
```

提供一个修改装饰器，定义一个生成 SQL 表达式的方法。

```py
attribute extension_type: InspectionAttrExtensionType = 'HYBRID_METHOD'
```

扩展类型（如果有）。默认为 `NotExtension.NOT_EXTENSION`

另请参阅

`HybridExtensionType`

`AssociationProxyExtensionType`

```py
attribute inplace
```

返回此 `hybrid_method` 的 inplace mutator。

当调用 `hybrid_method.expression()` 装饰器时，`hybrid_method` 类已经执行“in place”变异，因此此属性返回 Self。

2.0.4 版中的新功能。

另请参阅

使用 inplace 创建符合 pep-484 标准的混合属性（#hybrid-pep484-naming）

```py
attribute is_attribute = True
```

如果此对象是 Python 描述符，则为 True。

这可能是许多类型之一。通常是一个 `QueryableAttribute`，它代表一个 `MapperProperty` 上的属性事件。但也可以是一个扩展类型，例如 `AssociationProxy` 或 `hybrid_property`。`InspectionAttr.extension_type` 将指示一个常量，用于标识特定的子类型。

另请参阅

`Mapper.all_orm_descriptors`

```py
class sqlalchemy.ext.hybrid.hybrid_property
```

一个装饰器，允许定义既有实例级别又有类级别行为的 Python 描述符。

**成员**

__init__(), comparator(), deleter(), expression(), extension_type, getter(), inplace, is_attribute, overrides, setter(), update_expression()

**类签名**

类`sqlalchemy.ext.hybrid.hybrid_property`（`sqlalchemy.orm.base.InspectionAttrInfo`, `sqlalchemy.orm.base.ORMDescriptor`)

```py
method __init__(fget: _HybridGetterType[_T], fset: _HybridSetterType[_T] | None = None, fdel: _HybridDeleterType[_T] | None = None, expr: _HybridExprCallableType[_T] | None = None, custom_comparator: Comparator[_T] | None = None, update_expr: _HybridUpdaterType[_T] | None = None)
```

创建一个新的`hybrid_property`。

通常通过装饰器来使用：

```py
from sqlalchemy.ext.hybrid import hybrid_property

class SomeClass:
    @hybrid_property
    def value(self):
        return self._value

    @value.setter
    def value(self, value):
        self._value = value
```

```py
method comparator(comparator: _HybridComparatorCallableType[_T]) → hybrid_property[_T]
```

提供一个修改装饰器，定义一个自定义比较器生成方法。

被装饰方法的返回值应该是`Comparator`的一个实例。

注意

`hybrid_property.comparator()`装饰器**替换**了`hybrid_property.expression()`装饰器的使用。它们不能同时使用。

当在类级别调用混合属性时，此处给出的`Comparator`对象被包装在一个专门的`QueryableAttribute`中，这是 ORM 用来表示其他映射属性的相同类型的对象。这样做的原因是为了在返回的结构中保留其他类级别属性，如文档字符串和对混合属性本身的引用，而不对传入的原始比较器对象进行任何修改。

注意

当从拥有类引用混合属性时（例如`SomeClass.some_hybrid`），返回一个`QueryableAttribute`的实例，表示表达式或比较器对象作为这个混合对象。然而，该对象本身有名为`expression`和`comparator`的访问器；因此，在子类中尝试覆盖这些装饰器时，可能需要首先使用`hybrid_property.overrides`修饰符进行限定。有关详细信息，请参阅该修饰符。

```py
method deleter(fdel: _HybridDeleterType[_T]) → hybrid_property[_T]
```

提供一个修改装饰器，定义一个删除方法。

```py
method expression(expr: _HybridExprCallableType[_T]) → hybrid_property[_T]
```

提供一个修改装饰器，定义一个生成 SQL 表达式的方法。

当在类级别调用混合时，此处给出的 SQL 表达式将包装在一个专门的 `QueryableAttribute` 中，该对象与 ORM 用于表示其他映射属性的对象相同。这样做的原因是为了在返回的结构中保持其他类级别属性（如文档字符串和对混合本身的引用），而不对传入的原始 SQL 表达式进行任何修改。

注意

当从拥有类引用混合属性时（例如 `SomeClass.some_hybrid`），会返回一个 `QueryableAttribute` 的实例，表示表达式或比较器对象以及此混合对象。然而，该对象本身有名为 `expression` 和 `comparator` 的访问器；因此，在尝试在子类上覆盖这些装饰器时，可能需要首先使用 `hybrid_property.overrides` 修饰符进行限定。详情请参阅该修饰符。

参见

定义与属性行为不同的表达式行为

```py
attribute extension_type: InspectionAttrExtensionType = 'HYBRID_PROPERTY'
```

扩展类型，如果有的话。默认为 `NotExtension.NOT_EXTENSION`

参见

`HybridExtensionType`

`AssociationProxyExtensionType`

```py
method getter(fget: _HybridGetterType[_T]) → hybrid_property[_T]
```

提供一个修改装饰器，定义一个 getter 方法。

自 1.2 版新功能。

```py
attribute inplace
```

返回此 `hybrid_property` 的 inplace mutator。

这是为了允许对混合进行原地变异，从而允许重用某个特定名称的第一个混合方法以添加更多方法，而无需将这些方法命名为相同的名称，例如：

```py
class Interval(Base):
    # ...

    @hybrid_property
    def radius(self) -> float:
        return abs(self.length) / 2

    @radius.inplace.setter
    def _radius_setter(self, value: float) -> None:
        self.length = value * 2

    @radius.inplace.expression
    def _radius_expression(cls) -> ColumnElement[float]:
        return type_coerce(func.abs(cls.length) / 2, Float)
```

自 2.0.4 版新功能。

参见

使用 inplace 创建符合 pep-484 的混合属性

```py
attribute is_attribute = True
```

如果此对象是 Python 描述符，则为 True。

这可以指代许多类型之一。通常是一个 `QueryableAttribute`，它代表一个 `MapperProperty` 的属性事件。但也可以是一个扩展类型，如 `AssociationProxy` 或 `hybrid_property`。`InspectionAttr.extension_type` 将引用一个常量，用于标识特定的子类型。

另请参见

`Mapper.all_orm_descriptors`

```py
attribute overrides
```

用于覆盖现有属性的方法的前缀。

`hybrid_property.overrides` 访问器只是返回这个混合对象，当在父类的类级别调用时，将取消引用通常在此级别返回的“instrumented attribute”，并允许修改装饰器，如 `hybrid_property.expression()` 和 `hybrid_property.comparator()` 被使用，而不会与通常存在于 `QueryableAttribute` 上的同名属性发生冲突：

```py
class SuperClass:
    # ...

    @hybrid_property
    def foobar(self):
        return self._foobar

class SubClass(SuperClass):
    # ...

    @SuperClass.foobar.overrides.expression
    def foobar(cls):
        return func.subfoobar(self._foobar)
```

版本 1.2 中新增。

另请参见

在子类中重用混合属性

```py
method setter(fset: _HybridSetterType[_T]) → hybrid_property[_T]
```

提供一个定义 setter 方法的修改装饰器。

```py
method update_expression(meth: _HybridUpdaterType[_T]) → hybrid_property[_T]
```

提供一个定义 UPDATE 元组生成方法的修改装饰器。

该方法接受一个值，该值将被渲染到 UPDATE 语句的 SET 子句中。然后该方法应将此值处理为适合最终 SET 子句的单独列表达式，并将它们作为 2 元组序列返回。每个元组包含一个列表达式作为键和要渲染的值。

例如：

```py
class Person(Base):
    # ...

    first_name = Column(String)
    last_name = Column(String)

    @hybrid_property
    def fullname(self):
        return first_name + " " + last_name

    @fullname.update_expression
    def fullname(cls, value):
        fname, lname = value.split(" ", 1)
        return [
            (cls.first_name, fname),
            (cls.last_name, lname)
        ]
```

版本 1.2 中新增。

```py
class sqlalchemy.ext.hybrid.Comparator
```

一个辅助类，允许轻松构建用于混合使用的自定义 `PropComparator` 类。

**类签名**

class `sqlalchemy.ext.hybrid.Comparator` (`sqlalchemy.orm.PropComparator`)

```py
class sqlalchemy.ext.hybrid.HybridExtensionType
```

一个枚举。

**成员**

HYBRID_METHOD, HYBRID_PROPERTY

**类签名**

类`sqlalchemy.ext.hybrid.HybridExtensionType`（`sqlalchemy.orm.base.InspectionAttrExtensionType`）

```py
attribute HYBRID_METHOD = 'HYBRID_METHOD'
```

表示一个`InspectionAttr`的符号，类型为`hybrid_method`

赋予`InspectionAttr.extension_type`属性。

另请参阅

`Mapper.all_orm_attributes`

```py
attribute HYBRID_PROPERTY = 'HYBRID_PROPERTY'
```

表示一个`InspectionAttr`的符号

类型为`hybrid_method`。

赋予`InspectionAttr.extension_type`属性。

另请参阅

`Mapper.all_orm_attributes`
