# 自定义类型

> 原文：[`docs.sqlalchemy.org/en/20/core/custom_types.html`](https://docs.sqlalchemy.org/en/20/core/custom_types.html)

存在各种方法来重新定义现有类型的行为以及提供新类型。

## 覆盖类型编译

经常需要强制类型的“字符串”版本，即在 CREATE TABLE 语句或其他 SQL 函数（如 CAST）中呈现的版本进行更改。例如，应用程序可能希望强制在除一个平台外的所有平台上呈现`BINARY`，在该平台上希望呈现`BLOB`。对于大多数用例，首选使用现有的通用类型，例如`LargeBinary`。但为了更准确地控制类型，可以将每个方言的编译指令与任何类型关联起来：

```py
from sqlalchemy.ext.compiler import compiles
from sqlalchemy.types import BINARY

@compiles(BINARY, "sqlite")
def compile_binary_sqlite(type_, compiler, **kw):
    return "BLOB"
```

上述代码允许使用`BINARY`，它将针对除 SQLite 外的所有后端生成字符串`BINARY`，在 SQLite 的情况下，它将生成`BLOB`。

请参阅更改类型编译部分，这是自定义 SQL 构造和编译扩展的一个子部分，其中包含额外的示例。

## 增强现有类型

`TypeDecorator`允许创建自定义类型，为现有类型对象添加绑定参数和结果处理行为。当需要对数据进行额外的 Python 内部编组以及/或从数据库中进行时使用。

注意

`TypeDecorator`的绑定和结果处理是**额外的**，除了由托管类型已执行的处理外，SQLAlchemy 还会根据每个 DBAPI 定制来执行特定于该 DBAPI 的处理。虽然可以通过直接子类化来替换给定类型的处理，但在实践中从不需要，并且 SQLAlchemy 不再支持这作为公共用例。

| 对象名称 | 描述 |
| --- | --- |
| TypeDecorator | 允许创建类型，为现有类型添加额外功能。 |

```py
class sqlalchemy.types.TypeDecorator
```

允许创建类型，为现有类型添加额外功能。

此方法优于直接子类化 SQLAlchemy 内置类型，因为它确保保留底层类型的所有必需功能。

典型用法：

```py
import sqlalchemy.types as types

class MyType(types.TypeDecorator):
  '''Prefixes Unicode values with "PREFIX:" on the way in and
 strips it off on the way out.
 '''

    impl = types.Unicode

    cache_ok = True

    def process_bind_param(self, value, dialect):
        return "PREFIX:" + value

    def process_result_value(self, value, dialect):
        return value[7:]

    def copy(self, **kw):
        return MyType(self.impl.length)
```

类级别的`impl`属性是必需的，并且可以引用任何`TypeEngine`类。或者，可以使用`load_dialect_impl()`方法根据给定的方言提供不同的类型类；在这种情况下，`impl`变量可以引用`TypeEngine`作为占位符。

`TypeDecorator.cache_ok`类级别标志指示此自定义`TypeDecorator`是否可以安全地用作缓存键的一部分。此标志默认为`None`，当 SQL 编译器尝试为使用此类型的语句生成缓存键时，将最初生成警告。如果`TypeDecorator`不能保证每次都产生相同的绑定/结果行为和 SQL 生成，则应将此标志设置为`False`；否则，如果该类每次都产生相同的行为，则可以设置为`True`。有关此工作原理的更多说明，请参见`TypeDecorator.cache_ok`。

接收不类似于最终使用的类型的 Python 类型的类型可能希望定义`TypeDecorator.coerce_compared_value()`方法。这用于在表达式中将 Python 对象强制转换为绑定参数时给表达式系统一个提示。考虑这个表达式：

```py
mytable.c.somecol + datetime.date(2009, 5, 15)
```

在上面，如果“somecol”是一个`Integer`变体，我们做日期算术操作是有意义的，其中上面通常被数据库解释为将一些天数加到给定日期上。表达式系统通过不试图将“date()”值强制转换为面向整数的绑定参数来做正确的事情。

但是，在`TypeDecorator`的情况下，我们通常会将一个传入的 Python 类型更改为新的东西 - 默认情况下，`TypeDecorator`会将非类型化的一侧“强制”成与自身相同的类型。例如下面，我们定义了一个将日期值存储为整数的“epoch”类型：

```py
class MyEpochType(types.TypeDecorator):
    impl = types.Integer

    cache_ok = True

    epoch = datetime.date(1970, 1, 1)

    def process_bind_param(self, value, dialect):
        return (value - self.epoch).days

    def process_result_value(self, value, dialect):
        return self.epoch + timedelta(days=value)
```

使用上述类型的`somecol + date`表达式将会强制右侧的“date”也被视为`MyEpochType`。

通过`TypeDecorator.coerce_compared_value()`方法可以覆盖此行为，该方法返回一个应用于表达式值的类型。在下面的示例中，我们设置了一个整数值将被视为`Integer`，而任何其他值都被假定为日期并将被视为`MyEpochType`：

```py
def coerce_compared_value(self, op, value):
    if isinstance(value, int):
        return Integer()
    else:
        return self
```

警告

注意，**coerce_compared_value 的行为不会默认从基本类型那里继承**。如果 `TypeDecorator` 是增强某种类型需要特殊逻辑的装饰器，这个方法 **必须** 被重写。一个关键的例子是当装饰 `JSON` 和 `JSONB` 类型时；应该使用 `TypeEngine.coerce_compared_value()` 的默认规则来处理像索引操作这样的操作符：

```py
from sqlalchemy import JSON
from sqlalchemy import TypeDecorator

class MyJsonType(TypeDecorator):
    impl = JSON

    cache_ok = True

    def coerce_compared_value(self, op, value):
        return self.impl.coerce_compared_value(op, value)
```

没有上述步骤，索引操作，比如`mycol['foo']`会导致索引值`'foo'`被 JSON 编码。

类似地，当使用 `ARRAY` 数据类型时，索引操作的类型强制转换（例如 `mycol[5]`）也由 `TypeDecorator.coerce_compared_value()` 处理，再次简单的重写就足够了，除非对特定操作符需要特殊规则：

```py
from sqlalchemy import ARRAY
from sqlalchemy import TypeDecorator

class MyArrayType(TypeDecorator):
    impl = ARRAY

    cache_ok = True

    def coerce_compared_value(self, op, value):
        return self.impl.coerce_compared_value(op, value)
```

**成员**

cache_ok, operate(), reverse_operate(), __init__(), bind_expression(), bind_processor(), coerce_compared_value(), coerce_to_is_types, column_expression(), comparator_factory, compare_values(), copy(), get_dbapi_type(), literal_processor(), load_dialect_impl(), process_bind_param(), process_literal_param(), process_result_value(), result_processor(), sort_key_function, type_engine()

**类签名**

类 `sqlalchemy.types.TypeDecorator` (`sqlalchemy.sql.expression.SchemaEventTarget`, `sqlalchemy.types.ExternalType`, `sqlalchemy.types.TypeEngine`)

```py
attribute cache_ok: bool | None = None
```

*继承自* `ExternalType.cache_ok` *属性* 的 `ExternalType`

使用此 `ExternalType` 表示的 if 语句是否“可以缓存”。

默认值 `None` 会发出警告，然后不允许缓存包含此类型的语句。将其设置为 `False` 可以禁用使用此类型的语句的缓存，而不发出警告。当设置为 `True` 时，对象的类和其状态的选定元素将用作缓存键的一部分。例如，使用 `TypeDecorator`：

```py
class MyType(TypeDecorator):
    impl = String

    cache_ok = True

    def __init__(self, choices):
        self.choices = tuple(choices)
        self.internal_only = True
```

上述类型的缓存键将等同于：

```py
>>> MyType(["a", "b", "c"])._static_cache_key
(<class '__main__.MyType'>, ('choices', ('a', 'b', 'c')))
```

缓存方案将从类型中提取与 `__init__()` 方法中参数名称相对应的属性。在上面的例子中，“choices” 属性成为缓存键的一部分，但“internal_only” 不会，因为没有名为 “internal_only” 的参数。

可缓存元素的要求是它们是可哈希的，并且还要求对于给定缓存值，它们每次都指示使用此类型的表达式的相同 SQL 渲染。

为了适应引用不可哈希结构（如字典、集合和列表）的数据类型，可以通过将可哈希结构分配给其名称与参数名称对应的属性来使这些对象“可缓存”。例如，一个接受查找值字典的数据类型可以将其公布为一系列已排序的元组。给定一个先前不可缓存的类型如下：

```py
class LookupType(UserDefinedType):
  '''a custom type that accepts a dictionary as a parameter.

 this is the non-cacheable version, as "self.lookup" is not
 hashable.

 '''

    def __init__(self, lookup):
        self.lookup = lookup

    def get_col_spec(self, **kw):
        return "VARCHAR(255)"

    def bind_processor(self, dialect):
        # ...  works with "self.lookup" ...
```

其中“lookup”是一个字典。该类型将无法生成缓存键：

```py
>>> type_ = LookupType({"a": 10, "b": 20})
>>> type_._static_cache_key
<stdin>:1: SAWarning: UserDefinedType LookupType({'a': 10, 'b': 20}) will not
produce a cache key because the ``cache_ok`` flag is not set to True.
Set this flag to True if this type object's state is safe to use
in a cache key, or False to disable this warning.
symbol('no_cache')
```

如果我们**确实**设置了这样的缓存键，它将无法使用。我们将得到一个包含字典的元组结构，该字典本身无法作为“缓存字典”中的键使用，例如 SQLAlchemy 的语句缓存，因为 Python 字典不可哈希：

```py
>>> # set cache_ok = True
>>> type_.cache_ok = True

>>> # this is the cache key it would generate
>>> key = type_._static_cache_key
>>> key
(<class '__main__.LookupType'>, ('lookup', {'a': 10, 'b': 20}))

>>> # however this key is not hashable, will fail when used with
>>> # SQLAlchemy statement cache
>>> some_cache = {key: "some sql value"}
Traceback (most recent call last): File "<stdin>", line 1,
in <module> TypeError: unhashable type: 'dict'
```

通过将排序后的元组元组分配给“.lookup”属性，可以使该类型可缓存：

```py
class LookupType(UserDefinedType):
  '''a custom type that accepts a dictionary as a parameter.

 The dictionary is stored both as itself in a private variable,
 and published in a public variable as a sorted tuple of tuples,
 which is hashable and will also return the same value for any
 two equivalent dictionaries.  Note it assumes the keys and
 values of the dictionary are themselves hashable.

 '''

    cache_ok = True

    def __init__(self, lookup):
        self._lookup = lookup

        # assume keys/values of "lookup" are hashable; otherwise
        # they would also need to be converted in some way here
        self.lookup = tuple(
            (key, lookup[key]) for key in sorted(lookup)
        )

    def get_col_spec(self, **kw):
        return "VARCHAR(255)"

    def bind_processor(self, dialect):
        # ...  works with "self._lookup" ...
```

在上面，`LookupType({"a": 10, "b": 20})` 的缓存键将是：

```py
>>> LookupType({"a": 10, "b": 20})._static_cache_key
(<class '__main__.LookupType'>, ('lookup', (('a', 10), ('b', 20))))
```

新功能，在版本 1.4.14 中：- 为 `TypeDecorator` 类添加了 `cache_ok` 标志，以允许对缓存进行一些可配置性。

新版本 1.4.28 中增加了`ExternalType` mixin，它将`cache_ok`标志推广到`TypeDecorator`和`UserDefinedType`类。

另请参阅

SQL 编译缓存

```py
class Comparator
```

一个特定于`TypeDecorator`的`Comparator`。

用户定义的`TypeDecorator`类通常不需要修改此内容。

**类签名**

类`sqlalchemy.types.TypeDecorator.Comparator`（`sqlalchemy.types.Comparator`）

```py
method operate(op: OperatorType, *other: Any, **kwargs: Any) → ColumnElement[_CT]
```

对参数进行操作。

这是最低级的操作，默认情况下引发`NotImplementedError`。

在子类中覆盖此内容可以允许将通用行为应用于所有操作。例如，覆盖`ColumnOperators`以将`func.lower()`应用于左右两侧：

```py
class MyComparator(ColumnOperators):
    def operate(self, op, other, **kwargs):
        return op(func.lower(self), func.lower(other), **kwargs)
```

参数：

+   `op` – 运算符可调用。

+   `*other` – 操作的‘其他’一侧。对于大多数操作，将是单个标量。

+   `**kwargs` – 修饰符。这些可以通过特殊的运算符传递，例如`ColumnOperators.contains()`。

```py
method reverse_operate(op: OperatorType, other: Any, **kwargs: Any) → ColumnElement[_CT]
```

对参数进行反向操作。

使用方式与`operate()`相同。

```py
method __init__(*args: Any, **kwargs: Any)
```

构造一个`TypeDecorator`。

发送到这里的参数将传递给分配给`impl`类级属性的类的构造函数，假设`impl`是可调用的，并且将生成的对象分配给`self.impl`实例属性（从而覆盖同名的类属性）。

如果类级`impl`不是可调用的（不寻常的情况），它将被分配给相同的实例属性，忽略传递给构造函数的参数。

子类可以覆盖此内容以完全自定义`self.impl`的生成。

```py
method bind_expression(bindparam: BindParameter[_T]) → ColumnElement[_T] | None
```

给定一个绑定值（即一个`BindParameter`实例），返回一个 SQL 表达式，该表达式通常将给定参数包装起来。

注意

此方法在语句的**SQL 编译**阶段调用，当渲染 SQL 字符串时。它**不一定**针对特定值调用，并且不应与`TypeDecorator.process_bind_param()`方法混淆，后者是处理语句执行时传递给特定参数的实际值的更典型方法。

`TypeDecorator`的子类可以重写此方法，以提供类型的自定义绑定表达式行为。此实现将**替换**基础实现类型的实现。

```py
method bind_processor(dialect: Dialect) → _BindProcessorType[_T] | None
```

为给定的`Dialect`提供一个绑定值处理函数。

这是通过`TypeEngine.bind_processor()`方法通常发生的绑定值转换的方法，它履行了`TypeEngine`合同。

注意

`TypeDecorator` 的用户定义的子类**不应该**实现这个方法，而应该实现`TypeDecorator.process_bind_param()`，以便保持实现类型提供的“内部”处理。

参数：

**dialect** – 正在使用的方言实例。

```py
method coerce_compared_value(op: OperatorType | None, value: Any) → Any
```

在表达式中建议为“强制转换”的 Python 值提供一种类型。

默认情况下，返回 self。当使用此类型的对象在表达式左侧或右侧与尚未分配 SQLAlchemy 类型的普通 Python 对象相比时，表达式系统将调用此方法：

```py
expr = table.c.somecolumn + 35
```

在上述情况下，如果`somecolumn`使用此类型，则将使用值`operator.add`和`35`调用此方法。返回值是为这个特定操作应该使用的 SQLAlchemy 类型。

```py
attribute coerce_to_is_types: Sequence[Type[Any]] = (<class 'NoneType'>,)
```

指定那些应该在表达式级别强制转换为“IS <constant>”的 Python 类型，当使用`==`进行比较时（对于`!=`结合`IS NOT`也是如此）。

对于大多数 SQLAlchemy 类型，这包括`NoneType`，以及`bool`。

`TypeDecorator` 修改此列表，只包括`NoneType`，因为处理布尔类型的 typedecorator 实现是常见的。

自定义`TypeDecorator`类可以重写此属性以返回一个空元组，在这种情况下，不会将任何值强制转换为常量。

```py
method column_expression(column: ColumnElement[_T]) → ColumnElement[_T] | None
```

给定一个 SELECT 列表达式，返回一个包装的 SQL 表达式。

注意

这个方法在语句的**SQL 编译**阶段调用，当渲染 SQL 字符串时。它**不会**针对特定值进行调用，并且不应将其与`TypeDecorator.process_result_value()`方法混淆，后者是处理语句执行后返回的实际值的更典型的方法。

`TypeDecorator`的子类可以重写此方法，以为类型提供自定义列表达式行为。此实现将**替换**底层实现类型的实现。

有关方法用途的完整描述，请参阅`TypeEngine.column_expression()`的描述。

```py
attribute comparator_factory: _ComparatorFactory[Any]
```

一个`Comparator`类，将应用于由拥有的`ColumnElement`对象执行的操作。

当执行列和 SQL 表达式操作时，核心表达式系统会查找`comparator_factory`属性。当与此属性相关联的是一个`Comparator`类时，它允许自定义重新定义所有现有运算符，以及定义新的运算符。现有运算符包括通过 Python 运算符重载提供的运算符，如`ColumnOperators.__add__()`和`ColumnOperators.__eq__()`，以及作为`ColumnOperators`的标准属性提供的运算符，如`ColumnOperators.like()`和`ColumnOperators.in_()`。

通过简单地对现有类型进行子类化或者使用`TypeDecorator`，可以允许对这个钩子进行基本的使用。有关示例，请参阅文档中的 Redefining and Creating New Operators 部分。

```py
method compare_values(x: Any, y: Any) → bool
```

给定两个值，比较它们是否相等。

默认情况下，这将调用底层“impl”的`TypeEngine.compare_values()`，这通常使用 Python 相等运算符`==`。

此函数由 ORM 用于将原始加载的值与拦截的“更改”值进行比较，以确定是否发生了净变化。

```py
method copy(**kw: Any) → Self
```

生产这个`TypeDecorator`实例的副本。

这是一个浅拷贝，并提供了部分`TypeEngine`合约的实现。通常不需要重写，除非用户定义的`TypeDecorator`具有应该深拷贝的本地状态。

```py
method get_dbapi_type(dbapi: module) → Any | None
```

返回由此`TypeDecorator`表示的 DBAPI 类型对象。

默认情况下，这将调用底层“impl”的`TypeEngine.get_dbapi_type()`。

```py
method literal_processor(dialect: Dialect) → _LiteralProcessorType[_T] | None
```

为给定的`Dialect`提供一个字面处理函数。

这是履行通过`TypeEngine.literal_processor()`方法正常发生的字面值转换的`TypeEngine`合约的方法。

注意

用户定义的`TypeDecorator`子类**不应该**实现此方法，而应该实现`TypeDecorator.process_literal_param()`，以便维护实现类型提供的“内部”处理。

```py
method load_dialect_impl(dialect: Dialect) → TypeEngine[Any]
```

返回与方言对应的`TypeEngine`对象。

这是一个最终用户的覆盖钩子，可用于根据给定的方言提供不同的类型。它被`TypeDecorator`的实现在帮助确定对于给定的`TypeDecorator`应最终返回什么类型时使用。

默认情况下返回`self.impl`。

```py
method process_bind_param(value: _T | None, dialect: Dialect) → Any
```

接收要转换的绑定参数值。

自定义的`TypeDecorator`子类应该重写此方法，以提供传入数据值的自定义行为。此方法在**语句执行时间**被调用，并传递要与语句中的绑定参数关联的字面 Python 数据值。

操作可以是任何所需的自定义行为，例如转换或序列化数据。这也可以用作验证逻辑的钩子。

参数：

+   `value` – 要操作的数据，应为子类中此方法预期的任何类型。可以是`None`。

+   `dialect` – 使用的`Dialect`。

另请参阅

增强现有类型

`TypeDecorator.process_result_value()`

```py
method process_literal_param(value: _T | None, dialect: Dialect) → str
```

接收要在语句中内联呈现的文字参数值。

注意

这个方法在**SQL 编译**阶段的语句执行时被调用，用于渲染 SQL 字符串。与其他 SQL 编译方法不同，它接收一个特定的 Python 值作为字符串进行渲染。但是不要将其与`TypeDecorator.process_bind_param()`方法混淆，后者是在语句执行时处理传递给特定参数的实际值的更典型的方法。

`TypeDecorator`的自定义子类应重写此方法，以提供对特殊情况下作为文字呈现的传入数据值的自定义行为。

返回的字符串将被渲染到输出字符串中。

```py
method process_result_value(value: Any | None, dialect: Dialect) → _T | None
```

接收要转换的结果行列值。

`TypeDecorator`的自定义子类应重写此方法，以提供从数据库结果行中接收到的数据值的自定义行为。此方法在**结果提取时**被调用，并传递从数据库结果行中提取的字面 Python 数据值。

操作可以是任何希望执行自定义行为的内容，例如转换或反序列化数据。

参数：

+   `value` – 要操作的数据，其类型由该子类中的此方法期望的类型决定。可以是`None`。

+   `dialect` – 使用的`Dialect`。

另请参阅

增强现有类型

`TypeDecorator.process_bind_param()`

```py
method result_processor(dialect: Dialect, coltype: Any) → _ResultProcessorType[_T] | None
```

为给定的`Dialect`提供结果值处理函数。

这是满足`TypeEngine`约定的方法，用于绑定值转换，通常通过`TypeEngine.result_processor()`方法进行。

注意

用户定义的 `TypeDecorator` 的子类**不应**实现这个方法，而应该实现 `TypeDecorator.process_result_value()`，以便保持实现类型提供的“内部”处理。

参数：

+   `dialect` – 正在使用的方言实例。

+   `coltype` – 一个 SQLAlchemy 数据类型。

```py
attribute sort_key_function: Callable[[Any], Any] | None
```

一个可以作为 sorted 的键传递的排序函数。

`None` 的默认值表示此类型存储的值是自排序的。

版本 1.3.8 中的新功能。

```py
method type_engine(dialect: Dialect) → TypeEngine[Any]
```

为这个 `TypeDecorator` 返回一个特定方言的 `TypeEngine` 实例。

在大多数情况下，这将返回一个由 `self.impl` 表示的 `TypeEngine` 类型的方言适配形式。使用 `dialect_impl()`。通过覆盖 `load_dialect_impl()` 可在此处自定义行为。

## TypeDecorator 配方

以下是一些关键的 `TypeDecorator` 配方。

### 将编码字符串强制转换为 Unicode

关于 `Unicode` 类型的一个常见困惑是，它仅用于处理 Python 端的 `unicode` 对象，这意味着作为绑定参数传递给它的值必须是 `u'some string'` 的形式，如果使用的是 Python 2 而不是 3。它执行的编码/解码函数仅适应所使用的 DBAPI 需要的内容，并且主要是一个私有实现细节。

可以通过使用需要时强制转换的 `TypeDecorator` 来实现安全接收 Python 字节串的用例：

```py
from sqlalchemy.types import TypeDecorator, Unicode

class CoerceUTF8(TypeDecorator):
  """Safely coerce Python bytestrings to Unicode
 before passing off to the database."""

    impl = Unicode

    def process_bind_param(self, value, dialect):
        if isinstance(value, str):
            value = value.decode("utf-8")
        return value
```

### 四舍五入数值

一些数据库连接器（如 SQL Server 的连接器）如果传递带有太多小数位的 Decimal 会出错。以下是一个将其四舍五入的配方：

```py
from sqlalchemy.types import TypeDecorator, Numeric
from decimal import Decimal

class SafeNumeric(TypeDecorator):
  """Adds quantization to Numeric."""

    impl = Numeric

    def __init__(self, *arg, **kw):
        TypeDecorator.__init__(self, *arg, **kw)
        self.quantize_int = -self.impl.scale
        self.quantize = Decimal(10) ** self.quantize_int

    def process_bind_param(self, value, dialect):
        if isinstance(value, Decimal) and value.as_tuple()[2] < self.quantize_int:
            value = value.quantize(self.quantize)
        return value
```

### 将时区感知时间戳存储为时区无关的 UTC 时间

数据库中的时间戳应始终以不考虑时区的方式存储。对于大多数数据库，这意味着首先将时间戳设置为 UTC 时区，然后将其存储为无时区（即，没有与之关联的任何时区；假定 UTC 为“隐式”时区）。或者，通常更喜欢使用数据库特定类型，如 PostgreSQL 的“带时区的时间戳”，因为它们具有更丰富的功能；但是，以纯 UTC 存储将在所有数据库和驱动程序上运行。当智能时区的数据库类型不可用或不受欢迎时，可以使用 `TypeDecorator` 创建一种将时区感知时间戳转换为时区不敏感时间戳的数据类型。下面，使用 Python 内置的 `datetime.timezone.utc` 时区来归一化和反归一化：

```py
import datetime

class TZDateTime(TypeDecorator):
    impl = DateTime
    cache_ok = True

    def process_bind_param(self, value, dialect):
        if value is not None:
            if not value.tzinfo or value.tzinfo.utcoffset(value) is None:
                raise TypeError("tzinfo is required")
            value = value.astimezone(datetime.timezone.utc).replace(tzinfo=None)
        return value

    def process_result_value(self, value, dialect):
        if value is not None:
            value = value.replace(tzinfo=datetime.timezone.utc)
        return value
```

### 与后端无关的 GUID 类型

注意

自 2.0 版本起，应优先使用内置的 `Uuid` 类型，其行为类似。此示例仅作为接收和返回 Python 对象的类型装饰器的示例。

接收并返回 Python uuid() 对象。在使用 PostgreSQL 时使用 PG UUID 类型，在使用 MSSQL 时使用 UNIQUEIDENTIFIER，在其他后端上使用 CHAR(32)，以字符串格式存储它们。`GUIDHyphens` 版本使用带连字符的值而不仅仅是十六进制字符串，使用 CHAR(36) 类型存储：

```py
from operator import attrgetter
from sqlalchemy.types import TypeDecorator, CHAR
from sqlalchemy.dialects.mssql import UNIQUEIDENTIFIER
from sqlalchemy.dialects.postgresql import UUID
import uuid

class GUID(TypeDecorator):
  """Platform-independent GUID type.

 Uses PostgreSQL's UUID type or MSSQL's UNIQUEIDENTIFIER,
 otherwise uses CHAR(32), storing as stringified hex values.

 """

    impl = CHAR
    cache_ok = True

    _default_type = CHAR(32)
    _uuid_as_str = attrgetter("hex")

    def load_dialect_impl(self, dialect):
        if dialect.name == "postgresql":
            return dialect.type_descriptor(UUID())
        elif dialect.name == "mssql":
            return dialect.type_descriptor(UNIQUEIDENTIFIER())
        else:
            return dialect.type_descriptor(self._default_type)

    def process_bind_param(self, value, dialect):
        if value is None or dialect.name in ("postgresql", "mssql"):
            return value
        else:
            if not isinstance(value, uuid.UUID):
                value = uuid.UUID(value)
            return self._uuid_as_str(value)

    def process_result_value(self, value, dialect):
        if value is None:
            return value
        else:
            if not isinstance(value, uuid.UUID):
                value = uuid.UUID(value)
            return value

class GUIDHyphens(GUID):
  """Platform-independent GUID type.

 Uses PostgreSQL's UUID type or MSSQL's UNIQUEIDENTIFIER,
 otherwise uses CHAR(36), storing as stringified uuid values.

 """

    _default_type = CHAR(36)
    _uuid_as_str = str
```

#### 将 Python `uuid.UUID` 链接到 ORM 映射的自定义类型

在使用 注释式声明表 映射声明 ORM 映射时，可以通过将其添加到 类型注解映射 中，将上述自定义 `GUID` 类型与 Python `uuid.UUID` 数据类型相关联，该类型通常定义在 `DeclarativeBase` 类上：

```py
import uuid
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column

class Base(DeclarativeBase):
    type_annotation_map = {
        uuid.UUID: GUID,
    }
```

通过上述配置，继承自 `Base` 的 ORM 映射类可以在注解中引用 Python `uuid.UUID`，这将自动使用 `GUID`：

```py
class MyModel(Base):
    __tablename__ = "my_table"

    id: Mapped[uuid.UUID] = mapped_column(primary_key=True)
```

另请参见

自定义类型映射

### 编组 JSON 字符串

此类型使用 `simplejson` 将 Python 数据结构编组为 JSON。可修改为使用 Python 内置的 json 编码器：

```py
from sqlalchemy.types import TypeDecorator, VARCHAR
import json

class JSONEncodedDict(TypeDecorator):
  """Represents an immutable structure as a json-encoded string.

 Usage:

 JSONEncodedDict(255)

 """

    impl = VARCHAR

    cache_ok = True

    def process_bind_param(self, value, dialect):
        if value is not None:
            value = json.dumps(value)

        return value

    def process_result_value(self, value, dialect):
        if value is not None:
            value = json.loads(value)
        return value
```

#### 添加可变性

默认情况下，ORM 不会检测上述类型的“可变性”——这意味着，对值的原地更改不会被检测到，也不会被刷新。如果没有进一步的步骤，您将需要在每个父对象上使用新对象替换现有值以检测更改：

```py
obj.json_value["key"] = "value"  # will *not* be detected by the ORM

obj.json_value = {"key": "value"}  # *will* be detected by the ORM
```

上述限制可能是可以接受的，因为许多应用程序可能不需要在创建后对值进行任何变异。对于那些确实具有此要求的应用程序，最好使用`sqlalchemy.ext.mutable`扩展来支持可变性。对于以字典为导向的 JSON 结构，我们可以这样应用：

```py
json_type = MutableDict.as_mutable(JSONEncodedDict)

class MyClass(Base):
    #  ...

    json_data = Column(json_type)
```

另请参阅

变异跟踪

#### 处理比较操作

`TypeDecorator`的默认行为是将任何表达式的“右侧”强制转换为相同类型。对于像 JSON 这样的类型，这意味着任何使用的操作符必须在 JSON 方面有意义。对于某些情况，用户可能希望在某些情况下使类型像 JSON 一样行为，在其他情况下像纯文本一样行为。一个例子是如果想要处理 JSON 类型的 LIKE 操作符。LIKE 对 JSON 结构没有意义，但对底层文本表示有意义。要使用`JSONEncodedDict`这样的类型来实现这一点，我们需要在尝试使用此操作符之前使用`cast()`或`type_coerce()`将列强制转换为文本形式：

```py
from sqlalchemy import type_coerce, String

stmt = select(my_table).where(type_coerce(my_table.c.json_data, String).like("%foo%"))
```

`TypeDecorator`提供了一个内置系统，用于基于操作符构建这些类型转换。如果我们想要经常使用 LIKE 操作符，并将我们的 JSON 对象解释为字符串，我们可以通过覆盖`TypeDecorator.coerce_compared_value()`方法将其构建到类型中：

```py
from sqlalchemy.sql import operators
from sqlalchemy import String

class JSONEncodedDict(TypeDecorator):
    impl = VARCHAR

    cache_ok = True

    def coerce_compared_value(self, op, value):
        if op in (operators.like_op, operators.not_like_op):
            return String()
        else:
            return self

    def process_bind_param(self, value, dialect):
        if value is not None:
            value = json.dumps(value)

        return value

    def process_result_value(self, value, dialect):
        if value is not None:
            value = json.loads(value)
        return value
```

以上只是处理像“LIKE”这样的操作符的一种方法。其他应用程序可能希望对于 JSON 对象没有意义的操作符（如“LIKE”）引发`NotImplementedError`，而不是自动强制转换为文本。

## 应用 SQL 级别的绑定/结果处理

如在扩展现有类型一节中所见，SQLAlchemy 允许在参数发送到语句时以及从数据库加载结果行时调用 Python 函数，以对发送到或从数据库的值应用转换。还可以定义 SQL 级别的转换。其理念在于，当只有关系数据库包含一系列必要的函数来在应用程序和持久性格式之间强制转换传入和传出数据时。示例包括使用数据库定义的加密/解密函数，以及处理地理数据的存储过程。

任何 `TypeEngine`、`UserDefinedType` 或 `TypeDecorator` 的子类都可以包含 `TypeEngine.bind_expression()` 和/或 `TypeEngine.column_expression()` 的实现，当定义为返回非 `None` 值时，应返回一个 `ColumnElement` 表达式，以注入到 SQL 语句中，要么围绕绑定参数，要么是列表达式。例如，要构建一个 `Geometry` 类型，它将对所有传出值应用 PostGIS 函数 `ST_GeomFromText`，并对所有传入数据应用函数 `ST_AsText`，我们可以创建我们自己的 `UserDefinedType` 的子类，并与 `func` 一起提供这些方法：

```py
from sqlalchemy import func
from sqlalchemy.types import UserDefinedType

class Geometry(UserDefinedType):
    def get_col_spec(self):
        return "GEOMETRY"

    def bind_expression(self, bindvalue):
        return func.ST_GeomFromText(bindvalue, type_=self)

    def column_expression(self, col):
        return func.ST_AsText(col, type_=self)
```

我们可以将 `Geometry` 类型应用到 `Table` 元数据中，并在 `select()` 构造中使用它：

```py
geometry = Table(
    "geometry",
    metadata,
    Column("geom_id", Integer, primary_key=True),
    Column("geom_data", Geometry),
)

print(
    select(geometry).where(
        geometry.c.geom_data == "LINESTRING(189412 252431,189631 259122)"
    )
)
```

结果的 SQL 将两个函数嵌入到适当的位置。`ST_AsText` 应用于列子句，以便返回值在进入结果集之前通过该函数运行，而 `ST_GeomFromText` 则在绑定参数上运行，以便转换传入的值：

```py
SELECT  geometry.geom_id,  ST_AsText(geometry.geom_data)  AS  geom_data_1
FROM  geometry
WHERE  geometry.geom_data  =  ST_GeomFromText(:geom_data_2)
```

`TypeEngine.column_expression()` 方法与编译器的机制交互，以确保 SQL 表达式不会干扰包装表达式的标记。例如，如果我们对表达式的 `label()` 渲染一个 `select()`，字符串标签将移动到包装表达式的外部：

```py
print(select(geometry.c.geom_data.label("my_data")))
```

输出：

```py
SELECT  ST_AsText(geometry.geom_data)  AS  my_data
FROM  geometry
```

另一个例子是我们装饰 `BYTEA` 以提供一个 `PGPString`，它将使用 PostgreSQL 的 `pgcrypto` 扩展来透明地加密/解密值：

```py
from sqlalchemy import (
    create_engine,
    String,
    select,
    func,
    MetaData,
    Table,
    Column,
    type_coerce,
    TypeDecorator,
)

from sqlalchemy.dialects.postgresql import BYTEA

class PGPString(TypeDecorator):
    impl = BYTEA

    cache_ok = True

    def __init__(self, passphrase):
        super(PGPString, self).__init__()

        self.passphrase = passphrase

    def bind_expression(self, bindvalue):
        # convert the bind's type from PGPString to
        # String, so that it's passed to psycopg2 as is without
        # a dbapi.Binary wrapper
        bindvalue = type_coerce(bindvalue, String)
        return func.pgp_sym_encrypt(bindvalue, self.passphrase)

    def column_expression(self, col):
        return func.pgp_sym_decrypt(col, self.passphrase)

metadata_obj = MetaData()
message = Table(
    "message",
    metadata_obj,
    Column("username", String(50)),
    Column("message", PGPString("this is my passphrase")),
)

engine = create_engine("postgresql+psycopg2://scott:tiger@localhost/test", echo=True)
with engine.begin() as conn:
    metadata_obj.create_all(conn)

    conn.execute(
        message.insert(),
        {"username": "some user", "message": "this is my message"},
    )

    print(
        conn.scalar(select(message.c.message).where(message.c.username == "some user"))
    )
```

`pgp_sym_encrypt` 和 `pgp_sym_decrypt` 函数应用于 INSERT 和 SELECT 语句：

```py
INSERT  INTO  message  (username,  message)
  VALUES  (%(username)s,  pgp_sym_encrypt(%(message)s,  %(pgp_sym_encrypt_1)s))
  -- {'username': 'some user', 'message': 'this is my message',
  --  'pgp_sym_encrypt_1': 'this is my passphrase'}

SELECT  pgp_sym_decrypt(message.message,  %(pgp_sym_decrypt_1)s)  AS  message_1
  FROM  message
  WHERE  message.username  =  %(username_1)s
  -- {'pgp_sym_decrypt_1': 'this is my passphrase', 'username_1': 'some user'}
```  ## 重新定义和创建新操作符

SQLAlchemy Core 定义了一组固定的表达式运算符，可用于所有列表达式。其中一些操作的效果是重载 Python 的内置运算符；此类运算符的示例包括`ColumnOperators.__eq__()`（`table.c.somecolumn == 'foo'`）、`ColumnOperators.__invert__()`（`~table.c.flag`）和`ColumnOperators.__add__()`（`table.c.x + table.c.y`）。其他运算符作为列表达式的显式方法公开，例如`ColumnOperators.in_()`（`table.c.value.in_(['x', 'y'])`）和`ColumnOperators.like()`（`table.c.value.like('%ed%')`）。

当需要使用尚未直接支持的 SQL 运算符时，最快的方法是在任何 SQL 表达式对象上使用`Operators.op()`方法；此方法接受表示要渲染的 SQL 运算符的字符串，并返回一个 Python 可调用对象，该对象接受任意的右侧表达式：

```py
>>> from sqlalchemy import column
>>> expr = column("x").op(">>")(column("y"))
>>> print(expr)
x  >>  y 
```

当使用自定义 SQL 类型时，还有一种方法可以实现自定义运算符，这些运算符与使用该列类型的任何列表达式自动关联，而无需每次使用运算符时直接调用`Operators.op()`。

为了实现这一点，SQL 表达式构造会查询与构造关联的`TypeEngine`对象，以确定内置运算符的行为，以及查找可能已调用的新方法。 `TypeEngine` 定义了一个由`Comparator`类实现的“比较”对象，为 SQL 运算符提供基本行为，许多特定类型提供了该类的自己的子实现。用户定义的`Comparator`实现可以直接构建到特定类型的简单子类中，以覆盖或定义新操作。下面，我们创建一个`Integer`子类，该子类重写了`ColumnOperators.__add__()`运算符，而该运算符又使用`Operators.op()`来生成自定义的 SQL 本身：

```py
from sqlalchemy import Integer

class MyInt(Integer):
    class comparator_factory(Integer.Comparator):
        def __add__(self, other):
            return self.op("goofy")(other)
```

上述配置创建了一个名为`MyInt`的新类，该类将`TypeEngine.comparator_factory`属性设置为指向一个新的类，该类是`Integer`类型相关联的`Comparator`类的子类。

用法：

```py
>>> sometable = Table("sometable", metadata, Column("data", MyInt))
>>> print(sometable.c.data + 5)
sometable.data  goofy  :data_1 
```

`ColumnOperators.__add__()`的实现由拥有的 SQL 表达式查询，通过将其自身作为`expr`属性来实例化`Comparator`。当实现需要直接引用源 `ColumnElement` 对象时，可以使用该属性：

```py
from sqlalchemy import Integer

class MyInt(Integer):
    class comparator_factory(Integer.Comparator):
        def __add__(self, other):
            return func.special_addition(self.expr, other)
```

添加到 `Comparator` 的新方法通过动态查找方案暴露在拥有的 SQL 表达式对象上，该方案将 `Comparator` 添加到拥有的 `ColumnElement` 表达式构造上的方法。例如，要为整数添加 `log()` 函数：

```py
from sqlalchemy import Integer, func

class MyInt(Integer):
    class comparator_factory(Integer.Comparator):
        def log(self, other):
            return func.log(self.expr, other)
```

使用上述类型：

```py
>>> print(sometable.c.data.log(5))
log(:log_1,  :log_2) 
```

当使用返回布尔结果的比较操作的 `Operators.op()` 时，应将 `Operators.op.is_comparison` 标志设置为 `True`：

```py
class MyInt(Integer):
    class comparator_factory(Integer.Comparator):
        def is_frobnozzled(self, other):
            return self.op("--is_frobnozzled->", is_comparison=True)(other)
```

一元操作也是可能的。例如，要添加 PostgreSQL 阶乘运算符的实现，我们将 `UnaryExpression` 构造与 `custom_op` 结合起来产生阶乘表达式：

```py
from sqlalchemy import Integer
from sqlalchemy.sql.expression import UnaryExpression
from sqlalchemy.sql import operators

class MyInteger(Integer):
    class comparator_factory(Integer.Comparator):
        def factorial(self):
            return UnaryExpression(
                self.expr, modifier=operators.custom_op("!"), type_=MyInteger
            )
```

使用上述类型：

```py
>>> from sqlalchemy.sql import column
>>> print(column("x", MyInteger).factorial())
x  ! 
```

另请参阅

`Operators.op()`

`TypeEngine.comparator_factory`

## 创建新类型

`UserDefinedType` 类被提供为定义全新数据库类型的简单基类。使用此类来表示 SQLAlchemy 不知道的原生数据库类型。如果只需要 Python 转换行为，请改用 `TypeDecorator`。

| 对象名称 | 描述 |
| --- | --- |
| UserDefinedType | 用户自定义类型的基础。 |

```py
class sqlalchemy.types.UserDefinedType
```

用户自定义类型的基础。

这应该是新类型的基础。注意，对于大多数情况，`TypeDecorator` 可能更合适：

```py
import sqlalchemy.types as types

class MyType(types.UserDefinedType):
    cache_ok = True

    def __init__(self, precision = 8):
        self.precision = precision

    def get_col_spec(self, **kw):
        return "MYTYPE(%s)" % self.precision

    def bind_processor(self, dialect):
        def process(value):
            return value
        return process

    def result_processor(self, dialect, coltype):
        def process(value):
            return value
        return process
```

一旦类型创建完成，就可以立即使用：

```py
table = Table('foo', metadata_obj,
    Column('id', Integer, primary_key=True),
    Column('data', MyType(16))
    )
```

`get_col_spec()` 方法在大多数情况下将接收一个关键字参数 `type_expression`，该参数指的是正在编译的类型的拥有表达式，例如 `Column` 或 `cast()` 构造。只有在方法接受关键字参数（例如 `**kw`）时才会发送此关键字；对于支持此函数的旧形式，使用内省来检查是否存在此关键字。

`UserDefinedType.cache_ok` 类级标志指示此自定义 `UserDefinedType` 是否安全用作缓存键的一部分。此标志默认为 `None`，当 SQL 编译器尝试为使用此类型的语古句生成缓存键时，将首先生成警告。如果不能保证 `UserDefinedType` 每次产生相同的绑定/结果行为和 SQL 生成行为，则应将此标志设置为 `False`；否则，如果类每次产生相同的行为，则可以将其设置为 `True`。有关此工作原理的更多说明，请参见 `UserDefinedType.cache_ok`。

新版本 1.4.28 中的更新：将 `ExternalType.cache_ok` 标志泛化，以便它对 `TypeDecorator` 和 'dUserDefinedType`](#sqlalchemy.types.UserDefinedType "sqlalchemy.types.UserDefinedType") 都可用。

**成员**

cache_ok, coerce_compared_value(), ensure_kwarg

**类签名**

类 `sqlalchemy.types.UserDefinedType`（`sqlalchemy.types.ExternalType`、`sqlalchemy.types.TypeEngineMixin`、`sqlalchemy.types.TypeEngine`、`sqlalchemy.util.langhelpers.EnsureKWArg`)

```py
attribute cache_ok: bool | None = None
```

*继承自* `ExternalType.cache_ok` *属性的* `ExternalType`

指示使用此 `ExternalType` 的语句是否“可缓存”。

默认值 `None` 将发出警告，然后不允许缓存包含此类型的语句。将其设置为 `False` 可完全禁用对包含此类型的语句进行缓存而不发出警告。当设置为 `True` 时，对象的类和其状态中的选定元素将作为缓存键的一部分使用。例如，使用 `TypeDecorator`：

```py
class MyType(TypeDecorator):
    impl = String

    cache_ok = True

    def __init__(self, choices):
        self.choices = tuple(choices)
        self.internal_only = True
```

上述类型的缓存键将等同于：

```py
>>> MyType(["a", "b", "c"])._static_cache_key
(<class '__main__.MyType'>, ('choices', ('a', 'b', 'c')))
```

缓存方案将从与 `__init__()` 方法中的参数名称相对应的类型中提取属性。在上面的例子中，“choices”属性成为缓存键的一部分，但“internal_only”则不是，因为没有名为“internal_only”的参数。

可缓存元素的要求是它们可哈希，并且它们指示对于给定缓存值的情况下，每次使用此类型的表达式渲染的 SQL 相同。

为了适应指向不可哈希结构（如字典、集合和列表）的数据类型，这些对象可以通过将可哈希结构分配给与参数名称相对应的属性来使其“可缓存”。例如，接受查找值字典的数据类型可能会将其公开为排序后的元组系列。假设以前不可缓存的类型为：

```py
class LookupType(UserDefinedType):
  '''a custom type that accepts a dictionary as a parameter.

 this is the non-cacheable version, as "self.lookup" is not
 hashable.

 '''

    def __init__(self, lookup):
        self.lookup = lookup

    def get_col_spec(self, **kw):
        return "VARCHAR(255)"

    def bind_processor(self, dialect):
        # ...  works with "self.lookup" ...
```

其中“lookup”是一个字典。该类型将无法生成缓存键：

```py
>>> type_ = LookupType({"a": 10, "b": 20})
>>> type_._static_cache_key
<stdin>:1: SAWarning: UserDefinedType LookupType({'a': 10, 'b': 20}) will not
produce a cache key because the ``cache_ok`` flag is not set to True.
Set this flag to True if this type object's state is safe to use
in a cache key, or False to disable this warning.
symbol('no_cache')
```

如果我们**设置**了这样的缓存键，它将无法使用。我们将得到一个包含字典的元组结构，该字典本身不能作为“缓存字典”中的键，比如 SQLAlchemy 的语句缓存，因为 Python 字典不可哈希：

```py
>>> # set cache_ok = True
>>> type_.cache_ok = True

>>> # this is the cache key it would generate
>>> key = type_._static_cache_key
>>> key
(<class '__main__.LookupType'>, ('lookup', {'a': 10, 'b': 20}))

>>> # however this key is not hashable, will fail when used with
>>> # SQLAlchemy statement cache
>>> some_cache = {key: "some sql value"}
Traceback (most recent call last): File "<stdin>", line 1,
in <module> TypeError: unhashable type: 'dict'
```

可通过将排序后的元组系列分配给“.lookup”属性来使类型可缓存：

```py
class LookupType(UserDefinedType):
  '''a custom type that accepts a dictionary as a parameter.

 The dictionary is stored both as itself in a private variable,
 and published in a public variable as a sorted tuple of tuples,
 which is hashable and will also return the same value for any
 two equivalent dictionaries.  Note it assumes the keys and
 values of the dictionary are themselves hashable.

 '''

    cache_ok = True

    def __init__(self, lookup):
        self._lookup = lookup

        # assume keys/values of "lookup" are hashable; otherwise
        # they would also need to be converted in some way here
        self.lookup = tuple(
            (key, lookup[key]) for key in sorted(lookup)
        )

    def get_col_spec(self, **kw):
        return "VARCHAR(255)"

    def bind_processor(self, dialect):
        # ...  works with "self._lookup" ...
```

在上面的例子中，`LookupType({"a": 10, "b": 20})` 的缓存键将是：

```py
>>> LookupType({"a": 10, "b": 20})._static_cache_key
(<class '__main__.LookupType'>, ('lookup', (('a', 10), ('b', 20))))
```

新版本 1.4.14 中新增：- 添加了 `cache_ok` 标志，以允许对 `TypeDecorator` 类进行某些缓存的可配置性。

新版本 1.4.28 中新增：- 添加了`ExternalType` mixin，该 mixin 将 `cache_ok` 标志泛化到了 `TypeDecorator` 和 `UserDefinedType` 类。

另请参见

SQL 编译缓存

```py
method coerce_compared_value(op: OperatorType | None, value: Any) → TypeEngine[Any]
```

建议为表达式中的“强制转换”Python 值提供一个类型。

`UserDefinedType`的默认行为与`TypeDecorator`的默认行为相同；默认情况下，它返回`self`，假设比较的值应强制转换为与此相同类型。有关更多详细信息，请参阅`TypeDecorator.coerce_compared_value()`。

```py
attribute ensure_kwarg: str = 'get_col_spec'
```

表示应接受`**kw`参数的方法名称的正则表达式。

类将扫描与名称模板匹配的方法，并在必要时装饰它们，以确保接受`**kw`参数。

## 使用自定义类型和反射

需要注意的是，被修改以具有附加 Python 行为的数据库类型，包括基于`TypeDecorator`的类型以及其他用户定义的数据类型子类，不在数据库模式中具有任何表示。当使用数据库中所述的反射功能时 Reflecting Database Objects，SQLAlchemy 使用一个固定映射，将数据库服务器报告的数据类型信息链接到 SQLAlchemy 数据类型对象。例如，如果我们在 PostgreSQL 模式中查看特定数据库列的定义，我们可能会收到字符串`"VARCHAR"`。SQLAlchemy 的 PostgreSQL 方言具有一个硬编码映射，将字符串名称`"VARCHAR"`链接到 SQLAlchemy `VARCHAR`类，这就是为什么当我们发出类似`Table('my_table', m, autoload_with=engine)`的语句时，其中的`Column`对象将在其内部具有`VARCHAR`的实例的原因。

这意味着如果一个`Table`对象使用的类型对象与数据库本地类型名称不直接对应，如果我们在其他地方使用反射为此数据库表创建一个新的`Table`对象，并针对一个新的`MetaData`集合，那么它将不会有此数据类型。例如：

```py
>>> from sqlalchemy import (
...     Table,
...     Column,
...     MetaData,
...     create_engine,
...     PickleType,
...     Integer,
... )
>>> metadata = MetaData()
>>> my_table = Table(
...     "my_table", metadata, Column("id", Integer), Column("data", PickleType)
... )
>>> engine = create_engine("sqlite://", echo="debug")
>>> my_table.create(engine)
INFO  sqlalchemy.engine.base.Engine
CREATE  TABLE  my_table  (
  id  INTEGER,
  data  BLOB
) 
```

在上面，我们使用了`PickleType`，这是一个在`LargeBinary`数据类型上工作的`TypeDecorator`，在 SQLite 中对应于数据库类型`BLOB`。在 CREATE TABLE 中，我们看到使用了`BLOB`数据类型。SQLite 数据库对我们使用的`PickleType`一无所知。

如果我们查看`my_table.c.data.type`的数据类型，因为这是我们直接创建的一个 Python 对象，它是`PickleType`：

```py
>>> my_table.c.data.type
PickleType()
```

然而，如果我们使用反射创建另一个`Table`实例，我们创建的 SQLite 数据库中不会表示使用了`PickleType`；相反，我们会得到`BLOB`：

```py
>>> metadata_two = MetaData()
>>> my_reflected_table = Table("my_table", metadata_two, autoload_with=engine)
INFO  sqlalchemy.engine.base.Engine  PRAGMA  main.table_info("my_table")
INFO  sqlalchemy.engine.base.Engine  ()
DEBUG  sqlalchemy.engine.base.Engine  Col  ('cid',  'name',  'type',  'notnull',  'dflt_value',  'pk')
DEBUG  sqlalchemy.engine.base.Engine  Row  (0,  'id',  'INTEGER',  0,  None,  0)
DEBUG  sqlalchemy.engine.base.Engine  Row  (1,  'data',  'BLOB',  0,  None,  0)

>>>  my_reflected_table.c.data.type
BLOB() 
```

通常，当应用程序定义具有自定义类型的显式`Table`元数据时，没有必要使用表反射，因为必要的`Table`元数据已经存在。但是，对于需要同时使用显式`Table`元数据（其中包括自定义的 Python 级别数据类型）以及设置其`Column`对象作为从数据库反映出的`Table`对象的应用程序或它们的组合的情况，仍然需要展示自定义数据类型的附加 Python 行为，必须采取额外的步骤来允许这样做。

最简单的方法是根据重写反射列中描述的重写特定列。在这种技术中，我们简单地使用反射与那些我们想要使用自定义或装饰的数据类型的列的显式`Column`对象结合使用：

```py
>>> metadata_three = MetaData()
>>> my_reflected_table = Table(
...     "my_table",
...     metadata_three,
...     Column("data", PickleType),
...     autoload_with=engine,
... )
```

上面的`my_reflected_table`对象被反射，并将从 SQLite 数据库加载“id”列的定义。但对于“data”列，我们已经使用了一个显式的`Column`定义来覆盖反射对象，其中包括我们想要的 Python 数据类型，`PickleType`。反射过程将保持此`Column`对象不变：

```py
>>> my_reflected_table.c.data.type
PickleType()
```

一个更加详细的将数据库原生类型对象转换为自定义数据类型的方法是使用`DDLEvents.column_reflect()`事件处理程序。例如，如果我们知道我们想要所有的`BLOB`数据类型实际上是`PickleType`，我们可以设置一个全局规则：

```py
from sqlalchemy import BLOB
from sqlalchemy import event
from sqlalchemy import PickleType
from sqlalchemy import Table

@event.listens_for(Table, "column_reflect")
def _setup_pickletype(inspector, table, column_info):
    if isinstance(column_info["type"], BLOB):
        column_info["type"] = PickleType()
```

当上述代码在任何表反射发生之前被调用*之前*（还要注意它应该**仅在应用程序中调用一次**，因为它是一个全局规则）时，当反射包含有`BLOB`数据类型的任何`Table`时，结果数据类型将存储在`Column`对象中，作为`PickleType`。

在实践中，上述基于事件的方法可能会有额外的规则，以便只影响那些数据类型重要的列，比如表名和可能列名的查找表，或者其他启发式方法，以准确确定应该用 Python 数据类型来建立哪些列。

## 重写类型编译

经常需要强制使用类型的“字符串”版本，即在 CREATE TABLE 语句或其他 SQL 函数（如 CAST）中呈现的版本进行更改。例如，应用程序可能希望强制对所有平台渲染`BINARY`，除了一个平台外，该平台希望渲染`BLOB`。对于大多数用例，使用现有的通用类型，例如`LargeBinary`，更为合适。但是为了更准确地控制类型，可以将每个方言的编译指令与任何类型相关联：

```py
from sqlalchemy.ext.compiler import compiles
from sqlalchemy.types import BINARY

@compiles(BINARY, "sqlite")
def compile_binary_sqlite(type_, compiler, **kw):
    return "BLOB"
```

上面的代码允许使用`BINARY`，在除 SQLite 以外的所有后端中，它将产生字符串`BINARY`，而在 SQLite 中，它将产生`BLOB`。

请参阅 更改类型的编译 部分，自定义 SQL 构造和编译扩展 的一个子节，以获取其他示例。

## 增强现有类型

`TypeDecorator` 允许创建自定义类型，将绑定参数和结果处理行为添加到现有类型对象中。当需要额外的在 Python 中对数据进行数据库内/外编组时使用。

注意

`TypeDecorator` 的绑定和结果处理**除了**已由托管类型执行的处理外，还由 SQLAlchemy 基于每个 DBAPI 进行自定义，以执行特定于该 DBAPI 的处理。虽然可能通过直接子类化替换给定类型的此处理，但在实践中从不需要，并且 SQLAlchemy 不再支持此作为公共用例。

| 对象名称 | 描述 |
| --- | --- |
| TypeDecorator | 允许创建将额外功能添加到现有类型的类型。 |

```py
class sqlalchemy.types.TypeDecorator
```

允许创建将额外功能添加到现有类型的类型。

与直接子类化 SQLAlchemy 的内置类型相比，此方法更受欢迎，因为它确保了底层类型的所有必需功能都得到保留。

典型用法：

```py
import sqlalchemy.types as types

class MyType(types.TypeDecorator):
  '''Prefixes Unicode values with "PREFIX:" on the way in and
 strips it off on the way out.
 '''

    impl = types.Unicode

    cache_ok = True

    def process_bind_param(self, value, dialect):
        return "PREFIX:" + value

    def process_result_value(self, value, dialect):
        return value[7:]

    def copy(self, **kw):
        return MyType(self.impl.length)
```

类级别的 `impl` 属性是必需的，并且可以引用任何 `TypeEngine` 类。或者，可以使用 `load_dialect_impl()` 方法来基于给定的方言提供不同的类型类；在这种情况下，`impl` 变量可以引用 `TypeEngine` 作为占位符。

`TypeDecorator.cache_ok` 类级别的标志指示此自定义 `TypeDecorator` 是否安全用于作为缓存键的一部分。此标志默认为 `None`，在 SQL 编译器尝试为使用此类型的语句生成缓存键时，将最初生成警告。如果不能保证每次生成的绑定/结果行为和 SQL 生成都相同，则应将此标志设置为 `False`；否则，如果该类每次都产生相同的行为，则可以将其设置为 `True`。有关此功能的更多说明，请参阅 `TypeDecorator.cache_ok`。

接收 Python 类型而不是与最终使用的类型相似的 Python 类型的类型可能希望定义`TypeDecorator.coerce_compared_value()`方法。这用于在表达式中将 Python 对象强制转换为绑定参数时为表达式系统提供提示。考虑以下表达式：

```py
mytable.c.somecol + datetime.date(2009, 5, 15)
```

如果“somecol”是一个`Integer`变体，我们进行日期算术操作是有道理的，其中上面通常被数据库解释为将一定数量的天数添加到给定日期。表达式系统通过不尝试将“date()`”值强制转换为整数导向绑定参数来实现正确的操作。

然而，在`TypeDecorator`的情况下，我们通常将传入的 Python 类型更改为新的类型 - `TypeDecorator`默认会“强制”非类型化的一侧成为与其自身相同的类型。例如，我们定义了一个将日期值存储为整数的“epoch”类型如下：

```py
class MyEpochType(types.TypeDecorator):
    impl = types.Integer

    cache_ok = True

    epoch = datetime.date(1970, 1, 1)

    def process_bind_param(self, value, dialect):
        return (value - self.epoch).days

    def process_result_value(self, value, dialect):
        return self.epoch + timedelta(days=value)
```

我们的表达式`somecol + date`将使用上述类型将右侧的“date”强制转换为`MyEpochType`。

此行为可以通过`TypeDecorator.coerce_compared_value()`方法进行覆盖，该方法返回应该用于表达式值的类型。以下我们设置它，以使整数值将被视为`Integer`，并且任何其他值都被假定为日期并将被视为`MyEpochType`：

```py
def coerce_compared_value(self, op, value):
    if isinstance(value, int):
        return Integer()
    else:
        return self
```

警告

请注意，**coerce_compared_value 的行为默认不会从基本类型那里继承**。如果`TypeDecorator`是用于增强某种类型，以便针对某些类型的运算符执行特殊逻辑，那么必须重写此方法。一个关键示例是装饰`JSON`和`JSONB`类型; 应使用`TypeEngine.coerce_compared_value()`的默认规则来处理诸如索引操作之类的运算符：

```py
from sqlalchemy import JSON
from sqlalchemy import TypeDecorator

class MyJsonType(TypeDecorator):
    impl = JSON

    cache_ok = True

    def coerce_compared_value(self, op, value):
        return self.impl.coerce_compared_value(op, value)
```

如果没有上述步骤，诸如`mycol['foo']`的索引操作将导致索引值`'foo'`被 JSON 编码。

类似地，在使用`ARRAY`数据类型时，索引操作（例如`mycol[5]`）的类型强制转换也由`TypeDecorator.coerce_compared_value()`处理，除非需要特定操作符的特殊规则，否则简单的重写就足够了：

```py
from sqlalchemy import ARRAY
from sqlalchemy import TypeDecorator

class MyArrayType(TypeDecorator):
    impl = ARRAY

    cache_ok = True

    def coerce_compared_value(self, op, value):
        return self.impl.coerce_compared_value(op, value)
```

**成员**

cache_ok, operate(), reverse_operate(), __init__(), bind_expression(), bind_processor(), coerce_compared_value(), coerce_to_is_types, column_expression(), comparator_factory, compare_values(), copy(), get_dbapi_type(), literal_processor(), load_dialect_impl(), process_bind_param(), process_literal_param(), process_result_value(), result_processor(), sort_key_function, type_engine()

**类签名**

类`sqlalchemy.types.TypeDecorator` (`sqlalchemy.sql.expression.SchemaEventTarget`, `sqlalchemy.types.ExternalType`, `sqlalchemy.types.TypeEngine`)

```py
attribute cache_ok: bool | None = None
```

*继承自* `ExternalType.cache_ok` *属性的* `ExternalType`

表明使用此`ExternalType`的语句是否“可以缓存”。

默认值`None`将发出警告，然后不允许缓存包含此类型的语句。将其设置为`False`以禁用包含此类型的语句的缓存，而无需警告。当设置为`True`时，对象的类和其状态中选择的元素将用作缓存键的一部分。例如，使用`TypeDecorator`:

```py
class MyType(TypeDecorator):
    impl = String

    cache_ok = True

    def __init__(self, choices):
        self.choices = tuple(choices)
        self.internal_only = True
```

上述类型的缓存键将等同于：

```py
>>> MyType(["a", "b", "c"])._static_cache_key
(<class '__main__.MyType'>, ('choices', ('a', 'b', 'c')))
```

缓存方案将从类型中提取与`__init__()`方法中参数名称对应的属性。以上，“choices”属性成为缓存键的一部分，但“internal_only”不会，因为没有名为“internal_only”的参数。

可缓存元素的要求是它们是可哈希的，并且还表明对于给定缓存值，每次使用此类型的表达式呈现相同的 SQL。

为了适应引用不可哈希结构（如字典、集合和列表）的数据类型，这些对象可以通过将可哈希结构分配给与参数名称对应的属性来“可缓存”。例如，一个接受查找值字典的数据类型可以将其公开为一系列排序后的元组。给定先前不可缓存的类型：

```py
class LookupType(UserDefinedType):
  '''a custom type that accepts a dictionary as a parameter.

 this is the non-cacheable version, as "self.lookup" is not
 hashable.

 '''

    def __init__(self, lookup):
        self.lookup = lookup

    def get_col_spec(self, **kw):
        return "VARCHAR(255)"

    def bind_processor(self, dialect):
        # ...  works with "self.lookup" ...
```

其中“lookup”是一个字典。该类型将无法生成缓存键：

```py
>>> type_ = LookupType({"a": 10, "b": 20})
>>> type_._static_cache_key
<stdin>:1: SAWarning: UserDefinedType LookupType({'a': 10, 'b': 20}) will not
produce a cache key because the ``cache_ok`` flag is not set to True.
Set this flag to True if this type object's state is safe to use
in a cache key, or False to disable this warning.
symbol('no_cache')
```

如果我们**确实**设置了这样一个缓存键，它将无法使用。我们将得到一个包含其中一个字典的元组结构，这个字典本身不能作为“缓存字典”中的键，比如 SQLAlchemy 的语句缓存，因为 Python 字典不可哈希：

```py
>>> # set cache_ok = True
>>> type_.cache_ok = True

>>> # this is the cache key it would generate
>>> key = type_._static_cache_key
>>> key
(<class '__main__.LookupType'>, ('lookup', {'a': 10, 'b': 20}))

>>> # however this key is not hashable, will fail when used with
>>> # SQLAlchemy statement cache
>>> some_cache = {key: "some sql value"}
Traceback (most recent call last): File "<stdin>", line 1,
in <module> TypeError: unhashable type: 'dict'
```

可通过将排序后的元组分配给“.lookup”属性来使类型可缓存：

```py
class LookupType(UserDefinedType):
  '''a custom type that accepts a dictionary as a parameter.

 The dictionary is stored both as itself in a private variable,
 and published in a public variable as a sorted tuple of tuples,
 which is hashable and will also return the same value for any
 two equivalent dictionaries.  Note it assumes the keys and
 values of the dictionary are themselves hashable.

 '''

    cache_ok = True

    def __init__(self, lookup):
        self._lookup = lookup

        # assume keys/values of "lookup" are hashable; otherwise
        # they would also need to be converted in some way here
        self.lookup = tuple(
            (key, lookup[key]) for key in sorted(lookup)
        )

    def get_col_spec(self, **kw):
        return "VARCHAR(255)"

    def bind_processor(self, dialect):
        # ...  works with "self._lookup" ...
```

在上面的例子中，`LookupType({"a": 10, "b": 20})`的缓存键将是：

```py
>>> LookupType({"a": 10, "b": 20})._static_cache_key
(<class '__main__.LookupType'>, ('lookup', (('a', 10), ('b', 20))))
```

1.4.14 版本中新增：- 添加了`cache_ok`标志，以允许对`TypeDecorator`类的缓存进行一些可配置性。

1.4.28 版本中新增：- 添加了`ExternalType` mixin，将`cache_ok`标志泛化到`TypeDecorator`和`UserDefinedType`类中。

另请参阅

SQL 编译缓存

```py
class Comparator
```

一个专门针对`TypeDecorator`的`Comparator`。

通常不需要修改用户定义的`TypeDecorator`类。

**类签名**

类`sqlalchemy.types.TypeDecorator.Comparator` (`sqlalchemy.types.Comparator`)

```py
method operate(op: OperatorType, *other: Any, **kwargs: Any) → ColumnElement[_CT]
```

对参数进行操作。

这是操作的最低级别，默认情况下引发`NotImplementedError`。

在子类中覆盖这一点可以让所有操作应用通用行为。例如，覆盖`ColumnOperators`以将`func.lower()`应用于左侧和右侧：

```py
class MyComparator(ColumnOperators):
    def operate(self, op, other, **kwargs):
        return op(func.lower(self), func.lower(other), **kwargs)
```

参数：

+   `op` – 运算符可调用。

+   `*other` – 操作的‘其他’一侧。对于大多数操作，将是一个单一标量。

+   `**kwargs` – 修改器。这些可能会被特殊运算符（例如 `ColumnOperators.contains()`）传递。

```py
method reverse_operate(op: OperatorType, other: Any, **kwargs: Any) → ColumnElement[_CT]
```

对参数执行反向操作。

使用方式与 `operate()` 相同。

```py
method __init__(*args: Any, **kwargs: Any)
```

构造一个 `TypeDecorator`。

发送到此处的参数将传递给分配给 `impl` 类级别属性的类的构造函数，假设 `impl` 是可调用的，并且生成的对象将被分配给 `self.impl` 实例属性（因此覆盖了同名的类属性）。

如果类级别的 `impl` 不是可调用的（不寻常的情况），它将被分配给相同的实例属性‘原样’，忽略传递给构造函数的那些参数。

子类可以重写此方法以完全自定义 `self.impl` 的生成。

```py
method bind_expression(bindparam: BindParameter[_T]) → ColumnElement[_T] | None
```

给定一个绑定值（即一个 `BindParameter` 实例），返回一个 SQL 表达式，通常会包装给定的参数。

注意

在语句的 **SQL 编译** 阶段调用此方法，当渲染 SQL 字符串时。它**不一定**针对特定值调用，并且不应与 `TypeDecorator.process_bind_param()` 方法混淆，后者是处理语句执行时传递给特定参数的实际值的更典型方法。

`TypeDecorator` 的子类可以重写此方法以为类型提供自定义绑定表达式行为。此实现将**替换**底层实现类型的实现。

```py
method bind_processor(dialect: Dialect) → _BindProcessorType[_T] | None
```

为给定 `Dialect` 提供一个绑定值处理函数。

这是执行绑定值转换的方法，通常通过 `TypeEngine.bind_processor()` 方法在 **SQL 编译** 阶段的语句中发生。

注意

`TypeDecorator` 的用户定义子类**不应该**实现这个方法，而应该实现 `TypeDecorator.process_bind_param()` 方法，以保持实现类型提供的“内部”处理。

参数：

**dialect** – 正在使用的 Dialect 实例。

```py
method coerce_compared_value(op: OperatorType | None, value: Any) → Any
```

为表达式中的‘强制’Python 值建议一种类型。

默认情况下，返回 self。 当此类型的对象位于尚未分配 SQLAlchemy 类型的纯 Python 对象的表达式的左侧或右侧时，表达式系统会调用此方法：

```py
expr = table.c.somecolumn + 35
```

在上述情况中，如果`somecolumn`使用了这种类型，则将使用值`operator.add`和`35`调用此方法。 返回值是应该用于此特定操作的`35`的任何 SQLAlchemy 类型。

```py
attribute coerce_to_is_types: Sequence[Type[Any]] = (<class 'NoneType'>,)
```

指定那些应该在表达式级别上被强制转换为“IS <constant>”的 Python 类型，当使用`==`进行比较时（与`!=`结合使用时同样适用于`IS NOT`）。

对于大多数 SQLAlchemy 类型，这包括`NoneType`，以及`bool`。

`TypeDecorator`修改此列表以仅包括`NoneType`，因为处理布尔类型的类型装饰器实现很常见。

自定义`TypeDecorator`类可以重写此属性以返回一个空元组，在这种情况下，不会将任何值强制转换为常量。

```py
method column_expression(column: ColumnElement[_T]) → ColumnElement[_T] | None
```

给定 SELECT 列表达式，返回包装的 SQL 表达式。

注意

此方法在语句的**SQL 编译**阶段调用，用于呈现 SQL 字符串。 它**不会**针对特定值调用，并且不应与`TypeDecorator.process_result_value()`方法混淆，后者是处理在语句执行后返回的结果行中实际值的更典型方法。

`TypeDecorator`的子类可以重写此方法以为该类型提供自定义列表达式行为。 该实现将**替换**底层实现类型的实现。

有关方法使用的完整描述，请参阅`TypeEngine.column_expression()`的描述。

```py
attribute comparator_factory: _ComparatorFactory[Any]
```

一个`Comparator`类，将应用于由拥有的操作`ColumnElement`对象执行。

当在列和 SQL 表达式操作时，核心表达式系统会查询`comparator_factory`属性。当与此属性关联的`Comparator`类时，它允许自定义重新定义所有现有运算符，以及定义新的运算符。现有运算符包括通过 Python 运算符重载提供的`ColumnOperators.__add__()`和`ColumnOperators.__eq__()`，以及由`ColumnOperators`的标准属性提供的运算符，如`ColumnOperators.like()`和`ColumnOperators.in_()`。

通过简单地对现有类型进行子类化或者使用`TypeDecorator`，可以允许对此钩子进行基本用法。请参阅文档部分 Redefining and Creating New Operators 以获取示例。

```py
method compare_values(x: Any, y: Any) → bool
```

给定两个值，比较它们是否相等。

默认情况下，这会调用底层“impl”的`TypeEngine.compare_values()`，该方法通常使用 Python 的等于运算符`==`。

此函数由 ORM 用于比较原始加载的值与拦截的“更改”值，以确定是否发生了净变化。

```py
method copy(**kw: Any) → Self
```

生成此`TypeDecorator`实例的副本。

这是一个浅拷贝，并提供了`TypeEngine`合同的部分。除非用户定义的`TypeDecorator`具有应该被深度复制的本地状态，否则通常不需要重写它。

```py
method get_dbapi_type(dbapi: module) → Any | None
```

返回由此`TypeDecorator`表示的 DBAPI 类型对象。

默认情况下，这会调用底层“impl”的`TypeEngine.get_dbapi_type()`。

```py
method literal_processor(dialect: Dialect) → _LiteralProcessorType[_T] | None
```

为给定的`Dialect`提供一个字面值处理函数。

这是实现通常通过`TypeEngine.literal_processor()`方法进行的字面值转换的`TypeEngine`合同的方法。

注意

用户定义的`TypeDecorator`子类**不应该**实现这个方法，而应该实现`TypeDecorator.process_literal_param()`，以便保持实现类型提供的“内部”处理。

```py
method load_dialect_impl(dialect: Dialect) → TypeEngine[Any]
```

返回对应于方言的`TypeEngine`对象。

这是一个最终用户覆盖的钩子，可用于根据给定的方言提供不同的类型。它由`TypeDecorator`的`type_engine()`实现使用，以帮助确定对于给定的`TypeDecorator`最终应该返回什么类型。

默认情况下返回`self.impl`。

```py
method process_bind_param(value: _T | None, dialect: Dialect) → Any
```

接收要转换的绑定参数值。

自定义的`TypeDecorator`子类应该重写此方法，以提供传入数据值的自定义行为。此方法在**语句执行时间**调用，并传递要与语句中的绑定参数相关联的字面 Python 数据值。

操作可以是任何想要执行自定义行为的内容，比如转换或序列化数据。这也可以用作验证逻辑的钩子。

参数：

+   `value` – 要操作的数据，由子类中此方法期望的任何类型。可以是`None`。

+   `dialect` – 正在使用的`Dialect`。

另请参阅

扩展现有类型

`TypeDecorator.process_result_value()`

```py
method process_literal_param(value: _T | None, dialect: Dialect) → str
```

接收要在语句中内联呈现的字面参数值。

注意

在语句的**SQL 编译**阶段调用此方法，当呈现 SQL 字符串时。与其他 SQL 编译方法不同，它会传递一个具体的 Python 值，以字符串形式呈现。但是，它不应与`TypeDecorator.process_bind_param()`方法混淆，后者是处理在语句执行时传递给特定参数的实际值的更典型方法。

自定义的`TypeDecorator`子类应该重写这个方法，以提供特定情况下的自定义行为，用于处理作为字面值呈现的传入数据值。

返回的字符串将呈现到输出字符串中。

```py
method process_result_value(value: Any | None, dialect: Dialect) → _T | None
```

接收要转换的结果行列值。

自定义的`TypeDecorator`子类应重写此方法，以提供从数据库返回的结果行中接收的数据值的自定义行为。此方法在**结果获取时间**调用，并传递从数据库结果行中提取的字面 Python 数据值。

操作可以是任何所需的自定义行为，例如转换或反序列化数据。

参数：

+   `value` – 要操作的数据，子类中此方法所期望的任何类型的数据。可以为`None`。

+   `dialect` – 正在使用的`Dialect`。

另请参阅

增强现有类型

`TypeDecorator.process_bind_param()`

```py
method result_processor(dialect: Dialect, coltype: Any) → _ResultProcessorType[_T] | None
```

为给定的`Dialect`提供结果值处理函数。

这是满足绑定值转换的`TypeEngine`契约的方法，通常通过`TypeEngine.result_processor()`方法进行。

注意

用户定义的`TypeDecorator`子类**不应**实现此方法，而应该实现`TypeDecorator.process_result_value()`，以便维护实现类型提供的“内部”处理。

参数：

+   `dialect` – 正在使用的方言实例。

+   `coltype` – 一个 SQLAlchemy 数据类型。

```py
attribute sort_key_function: Callable[[Any], Any] | None
```

一个可作为排序关键字传递给`sorted`的排序函数。

`None`的默认值表示此类型存储的值是自排序的。

版本 1.3.8 中新增。

```py
method type_engine(dialect: Dialect) → TypeEngine[Any]
```

为此`TypeDecorator`返回一个特定于方言的`TypeEngine`实例。

在大多数情况下，此方法返回由`self.impl`表示的`TypeEngine`类型的方言适配形式。使用`dialect_impl()`。可以通过重写`load_dialect_impl()`来在此处自定义行为。

## TypeDecorator 示例

以下是一些关键的`TypeDecorator`示例。

### 将编码字符串强制转换为 Unicode

`Unicode` 类型常见的一个令人困惑的地方是，它仅在 Python 一侧处理 Python `unicode` 对象，这意味着作为绑定参数传递给它的值必须是 `u'some string'` 的形式，如果使用的是 Python 2 而不是 3。它执行的编码/解码函数仅适合于所使用的 DBAPI 所需，并且主要是一个私有的实现细节。

通过使用 `TypeDecorator` ，可以实现安全接收 Python 字节串的类型的用例，即包含非 ASCII 字符并且在 Python 2 中不是 `u''` 对象的字符串，必要时进行强制转换：

```py
from sqlalchemy.types import TypeDecorator, Unicode

class CoerceUTF8(TypeDecorator):
  """Safely coerce Python bytestrings to Unicode
 before passing off to the database."""

    impl = Unicode

    def process_bind_param(self, value, dialect):
        if isinstance(value, str):
            value = value.decode("utf-8")
        return value
```

### 数字四舍五入

一些数据库连接器（如 SQL Server 的连接器）如果传递的 Decimal 有太多的小数位数，会出现问题。以下是将其四舍五入的方法：

```py
from sqlalchemy.types import TypeDecorator, Numeric
from decimal import Decimal

class SafeNumeric(TypeDecorator):
  """Adds quantization to Numeric."""

    impl = Numeric

    def __init__(self, *arg, **kw):
        TypeDecorator.__init__(self, *arg, **kw)
        self.quantize_int = -self.impl.scale
        self.quantize = Decimal(10) ** self.quantize_int

    def process_bind_param(self, value, dialect):
        if isinstance(value, Decimal) and value.as_tuple()[2] < self.quantize_int:
            value = value.quantize(self.quantize)
        return value
```

### 将时区感知时间戳存储为时区无关的 UTC

数据库中的时间戳应始终以与时区无关的方式存储。对于大多数数据库来说，这意味着确保时间戳首先处于 UTC 时区，然后将其存储为时区无关的（即，没有与之关联的任何时区；UTC 被假定为“隐式”时区）。另外，通常首选数据库特定类型，如 PostgreSQL 的 “TIMESTAMP WITH TIMEZONE” ，因为其功能更丰富；但是，存储为纯 UTC 将在所有数据库和驱动程序上工作。当时区智能数据库类型不是一个选择或不被首选时，可以使用 `TypeDecorator` 创建一个将时区感知时间戳转换为时区无关的数据类型，并反之亦然。下面，Python 的内置 `datetime.timezone.utc` 时区用于标准化和非标准化：

```py
import datetime

class TZDateTime(TypeDecorator):
    impl = DateTime
    cache_ok = True

    def process_bind_param(self, value, dialect):
        if value is not None:
            if not value.tzinfo or value.tzinfo.utcoffset(value) is None:
                raise TypeError("tzinfo is required")
            value = value.astimezone(datetime.timezone.utc).replace(tzinfo=None)
        return value

    def process_result_value(self, value, dialect):
        if value is not None:
            value = value.replace(tzinfo=datetime.timezone.utc)
        return value
```

### 与后端无关的 GUID 类型

注意

自版本 2.0 起，内置的 `Uuid` 类型行为类似应该优先考虑。这个示例只是一个接收并返回 Python 对象的类型装饰器的示例。

接收和返回 Python 的 uuid() 对象。在使用 PostgreSQL 时使用 PG UUID 类型，在使用 MSSQL 时使用 UNIQUEIDENTIFIER，在其他后端使用 CHAR(32)，以字符串格式存储。`GUIDHyphens` 版本使用带连字符的方式存储值，而不仅仅是十六进制字符串，使用 CHAR(36) 类型：

```py
from operator import attrgetter
from sqlalchemy.types import TypeDecorator, CHAR
from sqlalchemy.dialects.mssql import UNIQUEIDENTIFIER
from sqlalchemy.dialects.postgresql import UUID
import uuid

class GUID(TypeDecorator):
  """Platform-independent GUID type.

 Uses PostgreSQL's UUID type or MSSQL's UNIQUEIDENTIFIER,
 otherwise uses CHAR(32), storing as stringified hex values.

 """

    impl = CHAR
    cache_ok = True

    _default_type = CHAR(32)
    _uuid_as_str = attrgetter("hex")

    def load_dialect_impl(self, dialect):
        if dialect.name == "postgresql":
            return dialect.type_descriptor(UUID())
        elif dialect.name == "mssql":
            return dialect.type_descriptor(UNIQUEIDENTIFIER())
        else:
            return dialect.type_descriptor(self._default_type)

    def process_bind_param(self, value, dialect):
        if value is None or dialect.name in ("postgresql", "mssql"):
            return value
        else:
            if not isinstance(value, uuid.UUID):
                value = uuid.UUID(value)
            return self._uuid_as_str(value)

    def process_result_value(self, value, dialect):
        if value is None:
            return value
        else:
            if not isinstance(value, uuid.UUID):
                value = uuid.UUID(value)
            return value

class GUIDHyphens(GUID):
  """Platform-independent GUID type.

 Uses PostgreSQL's UUID type or MSSQL's UNIQUEIDENTIFIER,
 otherwise uses CHAR(36), storing as stringified uuid values.

 """

    _default_type = CHAR(36)
    _uuid_as_str = str
```

#### 将 Python `uuid.UUID` 链接到 ORM 映射的自定义类型

当使用 注释的声明性表 进行 ORM 映射时，可以通过将其添加到 类型注释映射 中将上面定义的自定义 `GUID` 类型与 Python `uuid.UUID` 数据类型关联起来，该映射通常在 `DeclarativeBase` 类上定义：

```py
import uuid
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column

class Base(DeclarativeBase):
    type_annotation_map = {
        uuid.UUID: GUID,
    }
```

使用上述配置，从 `Base` 扩展的 ORM 映射类可以在注释中引用 Python `uuid.UUID`，这将自动使用 `GUID`：

```py
class MyModel(Base):
    __tablename__ = "my_table"

    id: Mapped[uuid.UUID] = mapped_column(primary_key=True)
```

另请参阅

自定义类型映射

### 编组 JSON 字符串

此类型使用 `simplejson` 将 Python 数据结构编组到 JSON 中。可以修改为使用 Python 的内置 json 编码器：

```py
from sqlalchemy.types import TypeDecorator, VARCHAR
import json

class JSONEncodedDict(TypeDecorator):
  """Represents an immutable structure as a json-encoded string.

 Usage:

 JSONEncodedDict(255)

 """

    impl = VARCHAR

    cache_ok = True

    def process_bind_param(self, value, dialect):
        if value is not None:
            value = json.dumps(value)

        return value

    def process_result_value(self, value, dialect):
        if value is not None:
            value = json.loads(value)
        return value
```

#### 添加可变性

默认情况下，ORM 不会检测上述类型的“可变性” - 这意味着对值的原地更改不会被检测到也不会被刷新。在没有进一步步骤的情况下，您需要在每个父对象上用新值替换现有值才能检测到更改：

```py
obj.json_value["key"] = "value"  # will *not* be detected by the ORM

obj.json_value = {"key": "value"}  # *will* be detected by the ORM
```

上述限制可能是可以接受的，因为许多应用程序可能不需要在创建后对值进行修改。对于那些确实具有此要求的应用程序，最好使用 `sqlalchemy.ext.mutable` 扩展来支持可变性。对于以字典为导向的 JSON 结构，我们可以这样应用：

```py
json_type = MutableDict.as_mutable(JSONEncodedDict)

class MyClass(Base):
    #  ...

    json_data = Column(json_type)
```

另请参阅

变异跟踪

#### 处理比较操作

`TypeDecorator` 的默认行为是将任何表达式的“右侧”强制转换为相同的类型。对于像 JSON 这样的类型，这意味着任何使用的运算符都必须符合 JSON 的意义。对于一些情况，用户可能希望该类型在某些情况下像 JSON 一样行事，在其他情况下像纯文本一样行事。一个例子是，如果一个人希望处理 JSON 类型的 LIKE 运算符。LIKE 对 JSON 结构毫无意义，但对底层文本表示是有意义的。要使用类似于 `JSONEncodedDict` 这样的类型来处理这个问题，我们需要在尝试使用此运算符之前将列强制转换为文本形式，使用 `cast()` 或 `type_coerce()`：

```py
from sqlalchemy import type_coerce, String

stmt = select(my_table).where(type_coerce(my_table.c.json_data, String).like("%foo%"))
```

`TypeDecorator` 提供了一个内置系统，用于基于运算符工作的类型转换，例如这些。如果我们想要频繁地使用 LIKE 运算符，并将我们的 JSON 对象解释为字符串，我们可以通过重写`TypeDecorator.coerce_compared_value()` 方法将其构建到类型中：

```py
from sqlalchemy.sql import operators
from sqlalchemy import String

class JSONEncodedDict(TypeDecorator):
    impl = VARCHAR

    cache_ok = True

    def coerce_compared_value(self, op, value):
        if op in (operators.like_op, operators.not_like_op):
            return String()
        else:
            return self

    def process_bind_param(self, value, dialect):
        if value is not None:
            value = json.dumps(value)

        return value

    def process_result_value(self, value, dialect):
        if value is not None:
            value = json.loads(value)
        return value
```

上述只是处理像“LIKE”这样的运算符的一种方法。其他应用程序可能希望对于没有与 JSON 对象具有意义的运算符（如“LIKE”）引发`NotImplementedError`，而不是自动强制转换为文本。

### 将编码字符串强制转换为 Unicode

关于 `Unicode` 类型的一个常见困惑是，它只打算在 Python 侧处理 Python `unicode` 对象，这意味着作为绑定参数传递给它的值必须是`u'some string'` 的形式，如果使用的是 Python 2 而不是 3。它执行的编码/解码函数仅适应所使用的 DBAPI 需要，主要是私有实现细节。

可以使用`TypeDecorator` 实现根据需要进行转换的类型的使用案例，该类型可以安全地接收 Python 字节串，即包含非 ASCII 字符并且不是 Python 2 中的`u''`对象的字符串：

```py
from sqlalchemy.types import TypeDecorator, Unicode

class CoerceUTF8(TypeDecorator):
  """Safely coerce Python bytestrings to Unicode
 before passing off to the database."""

    impl = Unicode

    def process_bind_param(self, value, dialect):
        if isinstance(value, str):
            value = value.decode("utf-8")
        return value
```

### 四舍五入数值

如果传递的 Decimal 具有太多小数位数，则某些数据库连接器（例如 SQL Server 的连接器）会中断。下面是一个将其舍入的方法：

```py
from sqlalchemy.types import TypeDecorator, Numeric
from decimal import Decimal

class SafeNumeric(TypeDecorator):
  """Adds quantization to Numeric."""

    impl = Numeric

    def __init__(self, *arg, **kw):
        TypeDecorator.__init__(self, *arg, **kw)
        self.quantize_int = -self.impl.scale
        self.quantize = Decimal(10) ** self.quantize_int

    def process_bind_param(self, value, dialect):
        if isinstance(value, Decimal) and value.as_tuple()[2] < self.quantize_int:
            value = value.quantize(self.quantize)
        return value
```

### 将时区感知时间戳存储为时区无关的 UTC

数据库中的时间戳应始终以时区不可知的方式存储。对于大多数数据库来说，这意味着确保时间戳首先在 UTC 时区中，然后将其存储为时区无关的（即，没有与之关联的任何时区；假定 UTC 是“隐式”时区）。或者，通常首选像 PostgreSQL 的“带时区的时间戳”这样的数据库特定类型，因为其更丰富的功能；然而，将其存储为纯 UTC 将适用于所有数据库和驱动程序。当时区智能型数据库类型不可用或不被偏爱时，`TypeDecorator` 可用于创建将时区感知时间戳转换为时区无关时间戳并再次转换的数据类型。在下面的示例中，Python 的内置`datetime.timezone.utc` 时区用于规范化和反规范化：

```py
import datetime

class TZDateTime(TypeDecorator):
    impl = DateTime
    cache_ok = True

    def process_bind_param(self, value, dialect):
        if value is not None:
            if not value.tzinfo or value.tzinfo.utcoffset(value) is None:
                raise TypeError("tzinfo is required")
            value = value.astimezone(datetime.timezone.utc).replace(tzinfo=None)
        return value

    def process_result_value(self, value, dialect):
        if value is not None:
            value = value.replace(tzinfo=datetime.timezone.utc)
        return value
```

### 跨后端 GUID 类型

注意

自 2.0 版本起，内置的`Uuid` 类型的行为类似的类型应优先考虑。此示例仅作为一个接收并返回 Python 对象的类型装饰器的示例。

接收和返回 Python `uuid()` 对象。在使用 PostgreSQL 时使用 PG UUID 类型，在使用 MSSQL 时使用 UNIQUEIDENTIFIER，在其他后端上使用 CHAR(32)，将其存储为字符串格式。`GUIDHyphens` 版本使用带连字符的值而不仅仅是十六进制字符串，使用 CHAR(36) 类型存储：

```py
from operator import attrgetter
from sqlalchemy.types import TypeDecorator, CHAR
from sqlalchemy.dialects.mssql import UNIQUEIDENTIFIER
from sqlalchemy.dialects.postgresql import UUID
import uuid

class GUID(TypeDecorator):
  """Platform-independent GUID type.

 Uses PostgreSQL's UUID type or MSSQL's UNIQUEIDENTIFIER,
 otherwise uses CHAR(32), storing as stringified hex values.

 """

    impl = CHAR
    cache_ok = True

    _default_type = CHAR(32)
    _uuid_as_str = attrgetter("hex")

    def load_dialect_impl(self, dialect):
        if dialect.name == "postgresql":
            return dialect.type_descriptor(UUID())
        elif dialect.name == "mssql":
            return dialect.type_descriptor(UNIQUEIDENTIFIER())
        else:
            return dialect.type_descriptor(self._default_type)

    def process_bind_param(self, value, dialect):
        if value is None or dialect.name in ("postgresql", "mssql"):
            return value
        else:
            if not isinstance(value, uuid.UUID):
                value = uuid.UUID(value)
            return self._uuid_as_str(value)

    def process_result_value(self, value, dialect):
        if value is None:
            return value
        else:
            if not isinstance(value, uuid.UUID):
                value = uuid.UUID(value)
            return value

class GUIDHyphens(GUID):
  """Platform-independent GUID type.

 Uses PostgreSQL's UUID type or MSSQL's UNIQUEIDENTIFIER,
 otherwise uses CHAR(36), storing as stringified uuid values.

 """

    _default_type = CHAR(36)
    _uuid_as_str = str
```

#### 将 Python `uuid.UUID` 链接到 ORM 映射的自定义类型

在使用 注释声明的声明性表 映射来声明 ORM 映射时，可以通过将其添加到 类型注释映射 中，将上面定义的自定义`GUID`类型与 Python `uuid.UUID` 数据类型关联起来，该类型通常定义在 `DeclarativeBase` 类上：

```py
import uuid
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column

class Base(DeclarativeBase):
    type_annotation_map = {
        uuid.UUID: GUID,
    }
```

有了上述配置，从`Base`继承的 ORM 映射类可以在注释中引用 Python `uuid.UUID`，这将自动使用`GUID`：

```py
class MyModel(Base):
    __tablename__ = "my_table"

    id: Mapped[uuid.UUID] = mapped_column(primary_key=True)
```

另请参阅

自定义类型映射

#### 将 Python `uuid.UUID` 链接到 ORM 映射的自定义类型

在使用 注释声明的声明性表 映射来声明 ORM 映射时，可以通过将其添加到 类型注释映射 中，将上面定义的自定义`GUID`类型与 Python `uuid.UUID` 数据类型关联起来，该类型通常定义在 `DeclarativeBase` 类上：

```py
import uuid
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column

class Base(DeclarativeBase):
    type_annotation_map = {
        uuid.UUID: GUID,
    }
```

有了上述配置，从`Base`继承的 ORM 映射类可以在注释中引用 Python `uuid.UUID`，这将自动使用`GUID`：

```py
class MyModel(Base):
    __tablename__ = "my_table"

    id: Mapped[uuid.UUID] = mapped_column(primary_key=True)
```

另请参阅

自定义类型映射

### 编组 JSON 字符串

此类型使用`simplejson`将 Python 数据结构编组为/从 JSON。可以修改为使用 Python 的内置 json 编码器：

```py
from sqlalchemy.types import TypeDecorator, VARCHAR
import json

class JSONEncodedDict(TypeDecorator):
  """Represents an immutable structure as a json-encoded string.

 Usage:

 JSONEncodedDict(255)

 """

    impl = VARCHAR

    cache_ok = True

    def process_bind_param(self, value, dialect):
        if value is not None:
            value = json.dumps(value)

        return value

    def process_result_value(self, value, dialect):
        if value is not None:
            value = json.loads(value)
        return value
```

#### 添加可变性

ORM 默认情况下不会检测上述类型的“可变性”——这意味着对值的原地更改不会被检测到也不会被刷新。如果没有进一步的步骤，您将需要在每个父对象上用新对象替换现有值以检测更改：

```py
obj.json_value["key"] = "value"  # will *not* be detected by the ORM

obj.json_value = {"key": "value"}  # *will* be detected by the ORM
```

以上限制可能是可以接受的，因为许多应用程序可能不需要在创建后对值进行变异。对于那些确实具有此要求的应用程序，最好使用`sqlalchemy.ext.mutable`扩展来支持可变性。对于以字典为导向的 JSON 结构，我们可以这样应用：

```py
json_type = MutableDict.as_mutable(JSONEncodedDict)

class MyClass(Base):
    #  ...

    json_data = Column(json_type)
```

另请参阅

变异追踪

#### 处理比较操作

`TypeDecorator`的默认行为是将任何表达式的“右手边”强制转换为相同的类型。对于 JSON 之类的类型，这意味着任何使用的运算符都必须在 JSON 的术语中有意义。对于某些情况，用户可能希望类型在某些情况下像 JSON 一样行为，在其他情况下像纯文本一样行为。一个例子是如果想要处理 JSON 类型的 LIKE 运算符。LIKE 对 JSON 结构没有意义，但对底层文本表示有意义。要通过`JSONEncodedDict`类型实现这一点，我们需要在尝试使用此运算符之前使用`cast()`或`type_coerce()`将列强制转换为文本形式：

```py
from sqlalchemy import type_coerce, String

stmt = select(my_table).where(type_coerce(my_table.c.json_data, String).like("%foo%"))
```

`TypeDecorator`提供了一种基于运算符的类型翻译的内置系统。如果我们希望经常使用 LIKE 运算符，将我们的 JSON 对象解释为字符串，我们可以通过重写`TypeDecorator.coerce_compared_value()`方法将其构建到类型中：

```py
from sqlalchemy.sql import operators
from sqlalchemy import String

class JSONEncodedDict(TypeDecorator):
    impl = VARCHAR

    cache_ok = True

    def coerce_compared_value(self, op, value):
        if op in (operators.like_op, operators.not_like_op):
            return String()
        else:
            return self

    def process_bind_param(self, value, dialect):
        if value is not None:
            value = json.dumps(value)

        return value

    def process_result_value(self, value, dialect):
        if value is not None:
            value = json.loads(value)
        return value
```

上面只是处理诸如“LIKE”之类运算符的一种方法。其他应用程序可能希望对于 JSON 对象没有意义的运算符（如“LIKE”）引发`NotImplementedError`，而不是自动强制转换为文本。

#### 添加可变性

ORM 默认不会检测上述类型的“可变性”——这意味着对值的原地更改不会被检测到，也不会被刷新。如果没有进一步的步骤，您将需要在每个父对象上用新值替换现有值以检测更改：

```py
obj.json_value["key"] = "value"  # will *not* be detected by the ORM

obj.json_value = {"key": "value"}  # *will* be detected by the ORM
```

以上限制可能是可以接受的，因为许多应用程序可能不要求一旦创建就对值进行突变。对于那些具有此要求的应用程序，最好使用`sqlalchemy.ext.mutable`扩展来支持可变性。对于面向字典的 JSON 结构，我们可以这样应用：

```py
json_type = MutableDict.as_mutable(JSONEncodedDict)

class MyClass(Base):
    #  ...

    json_data = Column(json_type)
```

参见

突变跟踪

#### 处理比较操作

`TypeDecorator`的默认行为是将任何表达式的“右侧”强制转换为相同类型。对于像 JSON 这样的类型，这意味着任何使用的运算符都必须从 JSON 的角度来看是有意义的。对于某些情况，用户可能希望该类型在某些情况下表现得像 JSON，在其他情况下表现为纯文本。一个例子是，如果一个人希望处理 JSON 类型的 LIKE 运算符。对于 JSON 结构来说，LIKE 没有意义，但对于基础文本表示来说是有意义的。要想在像`JSONEncodedDict`这样的类型中实现这一点，我们需要使用`cast()`或`type_coerce()`将列强制转换为文本形式，然后再尝试使用此运算符：

```py
from sqlalchemy import type_coerce, String

stmt = select(my_table).where(type_coerce(my_table.c.json_data, String).like("%foo%"))
```

`TypeDecorator`提供了一个基于运算符的系统，用于处理此类类型转换。如果我们想要频繁地使用 LIKE 运算符，并将我们的 JSON 对象解释为字符串，我们可以通过重写`TypeDecorator.coerce_compared_value()`方法将其构建到类型中。

```py
from sqlalchemy.sql import operators
from sqlalchemy import String

class JSONEncodedDict(TypeDecorator):
    impl = VARCHAR

    cache_ok = True

    def coerce_compared_value(self, op, value):
        if op in (operators.like_op, operators.not_like_op):
            return String()
        else:
            return self

    def process_bind_param(self, value, dialect):
        if value is not None:
            value = json.dumps(value)

        return value

    def process_result_value(self, value, dialect):
        if value is not None:
            value = json.loads(value)
        return value
```

上面只是处理“LIKE”运算符的一种方法。其他应用程序可能希望对于 JSON 对象没有意义的运算符（如“LIKE”）抛出`NotImplementedError`，而不是自动转换为文本。

## 应用 SQL 级别的绑定/结果处理

如在扩展现有类型部分所示，SQLAlchemy 允许在向语句发送参数以及从数据库加载结果行时调用 Python 函数，以对值进行转换，使其在发送到数据库时或从数据库加载时进行转换。还可以定义 SQL 级别的转换。这里的理念是，当只有关系数据库包含特定系列的函数时，这些函数对于在应用程序和持久性格式之间转换传入和传出数据是必要的。示例包括使用数据库定义的加密/解密函数，以及处理地理数据的存储过程。

任何`TypeEngine`、`UserDefinedType`或`TypeDecorator`子类都可以包含`TypeEngine.bind_expression()`和/或`TypeEngine.column_expression()`的实现，当定义为返回非`None`值时，应返回一个要注入到 SQL 语句中的`ColumnElement`表达式，无论是围绕绑定参数还是列表达式。例如，为了构建一个`Geometry`类型，该类型将对所有传出值应用 PostGIS 函数`ST_GeomFromText`，对所有传入数据应用函数`ST_AsText`，我们可以创建自己的`UserDefinedType`子类，该子类提供这些方法与`func`一起使用：

```py
from sqlalchemy import func
from sqlalchemy.types import UserDefinedType

class Geometry(UserDefinedType):
    def get_col_spec(self):
        return "GEOMETRY"

    def bind_expression(self, bindvalue):
        return func.ST_GeomFromText(bindvalue, type_=self)

    def column_expression(self, col):
        return func.ST_AsText(col, type_=self)
```

我们可以将`Geometry`类型应用到`Table`元数据中，并在`select()`构造中使用它：

```py
geometry = Table(
    "geometry",
    metadata,
    Column("geom_id", Integer, primary_key=True),
    Column("geom_data", Geometry),
)

print(
    select(geometry).where(
        geometry.c.geom_data == "LINESTRING(189412 252431,189631 259122)"
    )
)
```

结果的 SQL 嵌入了两个函数。`ST_AsText`应用于列子句，以便返回值在传递到结果集之前通过函数运行，而`ST_GeomFromText`应用于绑定参数，以便传入值被转换：

```py
SELECT  geometry.geom_id,  ST_AsText(geometry.geom_data)  AS  geom_data_1
FROM  geometry
WHERE  geometry.geom_data  =  ST_GeomFromText(:geom_data_2)
```

`TypeEngine.column_expression()`方法与编译器的机制交互，使得 SQL 表达式不会干扰包装表达式的标记。例如，如果我们针对我们表达式的`label()`进行了`select()`，字符串标签会移动到包装表达式的外部：

```py
print(select(geometry.c.geom_data.label("my_data")))
```

输出：

```py
SELECT  ST_AsText(geometry.geom_data)  AS  my_data
FROM  geometry
```

另一个例子是我们装饰`BYTEA`以提供`PGPString`，这将利用 PostgreSQL 的`pgcrypto`扩展来透明地加密/解密值：

```py
from sqlalchemy import (
    create_engine,
    String,
    select,
    func,
    MetaData,
    Table,
    Column,
    type_coerce,
    TypeDecorator,
)

from sqlalchemy.dialects.postgresql import BYTEA

class PGPString(TypeDecorator):
    impl = BYTEA

    cache_ok = True

    def __init__(self, passphrase):
        super(PGPString, self).__init__()

        self.passphrase = passphrase

    def bind_expression(self, bindvalue):
        # convert the bind's type from PGPString to
        # String, so that it's passed to psycopg2 as is without
        # a dbapi.Binary wrapper
        bindvalue = type_coerce(bindvalue, String)
        return func.pgp_sym_encrypt(bindvalue, self.passphrase)

    def column_expression(self, col):
        return func.pgp_sym_decrypt(col, self.passphrase)

metadata_obj = MetaData()
message = Table(
    "message",
    metadata_obj,
    Column("username", String(50)),
    Column("message", PGPString("this is my passphrase")),
)

engine = create_engine("postgresql+psycopg2://scott:tiger@localhost/test", echo=True)
with engine.begin() as conn:
    metadata_obj.create_all(conn)

    conn.execute(
        message.insert(),
        {"username": "some user", "message": "this is my message"},
    )

    print(
        conn.scalar(select(message.c.message).where(message.c.username == "some user"))
    )
```

`pgp_sym_encrypt`和`pgp_sym_decrypt`函数应用于 INSERT 和 SELECT 语句：

```py
INSERT  INTO  message  (username,  message)
  VALUES  (%(username)s,  pgp_sym_encrypt(%(message)s,  %(pgp_sym_encrypt_1)s))
  -- {'username': 'some user', 'message': 'this is my message',
  --  'pgp_sym_encrypt_1': 'this is my passphrase'}

SELECT  pgp_sym_decrypt(message.message,  %(pgp_sym_decrypt_1)s)  AS  message_1
  FROM  message
  WHERE  message.username  =  %(username_1)s
  -- {'pgp_sym_decrypt_1': 'this is my passphrase', 'username_1': 'some user'}
```

## 重新定义和创建新操作符

SQLAlchemy 核心定义了一组固定的表达式操作符，可用于所有列表达式。其中一些操作具有重载 Python 内置操作符的效果；此类操作符的示例包括`ColumnOperators.__eq__()`（`table.c.somecolumn == 'foo'`）、`ColumnOperators.__invert__()`（`~table.c.flag`）和`ColumnOperators.__add__()`（`table.c.x + table.c.y`）。其他操作符作为列表达式上的显式方法公开，例如`ColumnOperators.in_()`（`table.c.value.in_(['x', 'y'])`）和`ColumnOperators.like()`（`table.c.value.like('%ed%')`）。

当需要使用 SQL 操作符而已直接支持的情况下，最方便的方法是在任何 SQL 表达式对象上使用`Operators.op()`方法；该方法接受一个表示要呈现的 SQL 操作符的字符串，并返回一个接受任意右侧表达式的 Python 可调用对象：

```py
>>> from sqlalchemy import column
>>> expr = column("x").op(">>")(column("y"))
>>> print(expr)
x  >>  y 
```

当使用自定义 SQL 类型时，还有一种实现自定义操作符的方法，就像上面提到的那样，这些操作符在使用该列类型的任何列表达式上自动存在，而无需在每次使用操作符时直接调用`Operators.op()`。

为了实现这一点，SQL 表达式构造会参考与构造关联的`TypeEngine`对象，以确定内置运算符的行为，并寻找可能已被调用的新方法。`TypeEngine`定义了一个由`Comparator`类实现的“比较”对象，为 SQL 运算符提供基本行为，许多具体类型提供了它们自己的此类的子实现。用户定义的`Comparator`实现可以直接构建到特定类型的简单子类中，以覆盖或定义新操作。下面，我们创建一个`Integer`子类，它重写了`ColumnOperators.__add__()`运算符，而该运算符又使用`Operators.op()`来生成自定义的 SQL 代码：

```py
from sqlalchemy import Integer

class MyInt(Integer):
    class comparator_factory(Integer.Comparator):
        def __add__(self, other):
            return self.op("goofy")(other)
```

上述配置创建了一个新的类`MyInt`，它将`TypeEngine.comparator_factory`属性设置为引用一个新的类，该类是与`Integer`类型关联的`Comparator`类的子类。

使用方法：

```py
>>> sometable = Table("sometable", metadata, Column("data", MyInt))
>>> print(sometable.c.data + 5)
sometable.data  goofy  :data_1 
```

对于`ColumnOperators.__add__()`的实现是由拥有的 SQL 表达式进行参考的，通过使用自身作为`expr`属性来实例化`Comparator`。当实现需要直接引用原始`ColumnElement`对象时，可以使用此属性：

```py
from sqlalchemy import Integer

class MyInt(Integer):
    class comparator_factory(Integer.Comparator):
        def __add__(self, other):
            return func.special_addition(self.expr, other)
```

对于 `Comparator` 添加的新方法，通过动态查找方案暴露在拥有 SQL 表达式对象上，这样可以将添加到 `Comparator` 的方法暴露到拥有的 `ColumnElement` 表达式构造上。例如，要为整数添加一个 `log()` 函数：

```py
from sqlalchemy import Integer, func

class MyInt(Integer):
    class comparator_factory(Integer.Comparator):
        def log(self, other):
            return func.log(self.expr, other)
```

使用上述类型：

```py
>>> print(sometable.c.data.log(5))
log(:log_1,  :log_2) 
```

在使用 `Operators.op()` 进行返回布尔结果的比较操作时，应将 `Operators.op.is_comparison` 标志设置为 `True`：

```py
class MyInt(Integer):
    class comparator_factory(Integer.Comparator):
        def is_frobnozzled(self, other):
            return self.op("--is_frobnozzled->", is_comparison=True)(other)
```

一元操作也是可能的。例如，要添加 PostgreSQL 阶乘运算符的实现，我们结合 `UnaryExpression` 构造以及 `custom_op` 来生成阶乘表达式：

```py
from sqlalchemy import Integer
from sqlalchemy.sql.expression import UnaryExpression
from sqlalchemy.sql import operators

class MyInteger(Integer):
    class comparator_factory(Integer.Comparator):
        def factorial(self):
            return UnaryExpression(
                self.expr, modifier=operators.custom_op("!"), type_=MyInteger
            )
```

使用上述类型：

```py
>>> from sqlalchemy.sql import column
>>> print(column("x", MyInteger).factorial())
x  ! 
```

另请参阅

`Operators.op()`

`TypeEngine.comparator_factory`

## 创建新类型

`UserDefinedType` 类被提供作为定义全新数据库类型的简单基类。使用它来表示 SQLAlchemy 不知道的本地数据库类型。如果只需要 Python 转换行为，请改用 `TypeDecorator`。

| 对象名称 | 描述 |
| --- | --- |
| UserDefinedType | 用户定义类型的基础。 |

```py
class sqlalchemy.types.UserDefinedType
```

用户定义类型的基础。

这应该是新类型的基础。请注意，对于大多数情况，`TypeDecorator` 可能更合适：

```py
import sqlalchemy.types as types

class MyType(types.UserDefinedType):
    cache_ok = True

    def __init__(self, precision = 8):
        self.precision = precision

    def get_col_spec(self, **kw):
        return "MYTYPE(%s)" % self.precision

    def bind_processor(self, dialect):
        def process(value):
            return value
        return process

    def result_processor(self, dialect, coltype):
        def process(value):
            return value
        return process
```

一旦类型被创建，它立即可用：

```py
table = Table('foo', metadata_obj,
    Column('id', Integer, primary_key=True),
    Column('data', MyType(16))
    )
```

`get_col_spec()` 方法在大多数情况下将接收一个关键字参数 `type_expression`，该参数指的是类型的拥有表达式在编译时，例如 `Column` 或 `cast()` 构造。仅当方法接受关键字参数（例如 `**kw`）时才发送此关键字；用于检查此函数的传统形式的内省。

`UserDefinedType.cache_ok` 类级标志指示此自定义 `UserDefinedType` 是否安全用作缓存键的一部分。此标志默认为 `None`，当 SQL 编译器尝试为使用此类型的语句生成缓存键时，最初会生成警告。如果不能保证 `UserDefinedType` 每次产生相同的绑定/结果行为和 SQL 生成，应将此标志设置为 `False`；否则，如果类每次产生相同的行为，则可以设置为 `True`。有关此功能的详细说明，请参阅 `UserDefinedType.cache_ok`。

新版本 1.4.28 中：将 `ExternalType.cache_ok` 标志泛化，以便它同时适用于 `TypeDecorator` 和 `UserDefinedType`。

**成员**

cache_ok，coerce_compared_value()，ensure_kwarg

**类签名**

类 `sqlalchemy.types.UserDefinedType`（`sqlalchemy.types.ExternalType`，`sqlalchemy.types.TypeEngineMixin`，`sqlalchemy.types.TypeEngine`，`sqlalchemy.util.langhelpers.EnsureKWArg`）

```py
attribute cache_ok: bool | None = None
```

*继承自* `ExternalType.cache_ok` *属性的* `ExternalType`

指示使用此 `ExternalType` 的语句是否“可缓存”。

默认值`None`将发出警告，然后不允许缓存包含此类型的语句。将其设置为`False`以完全禁用使用此类型的语句的缓存，而无需警告。当设置为`True`时，对象的类和其状态的选定元素将用作缓存键的一部分。例如，使用`TypeDecorator`：

```py
class MyType(TypeDecorator):
    impl = String

    cache_ok = True

    def __init__(self, choices):
        self.choices = tuple(choices)
        self.internal_only = True
```

上述类型的缓存键将等同于：

```py
>>> MyType(["a", "b", "c"])._static_cache_key
(<class '__main__.MyType'>, ('choices', ('a', 'b', 'c')))
```

缓存方案将从与`__init__()`方法中的参数名称相对应的类型中提取属性。在上面的例子中，“choices”属性成为缓存键的一部分，但“internal_only”不是，因为没有名为“internal_only”的参数。

可缓存元素的要求是它们是可哈希的，并且还要表明对于给定缓存值，每次使用此类型的表达式渲染的 SQL 都相同。

为了适应引用不可哈希结构的数据类型，如字典、集合和列表的数据类型，可以通过将可哈希结构分配给名称与参数名称对应的属性来使这些对象“可缓存”。例如，接受查找值字典的数据类型可以将其发布为排序的元组系列。给定一个以前不可缓存的类型如下：

```py
class LookupType(UserDefinedType):
  '''a custom type that accepts a dictionary as a parameter.

 this is the non-cacheable version, as "self.lookup" is not
 hashable.

 '''

    def __init__(self, lookup):
        self.lookup = lookup

    def get_col_spec(self, **kw):
        return "VARCHAR(255)"

    def bind_processor(self, dialect):
        # ...  works with "self.lookup" ...
```

“查找”是一个字典。该类型将无法生成缓存键：

```py
>>> type_ = LookupType({"a": 10, "b": 20})
>>> type_._static_cache_key
<stdin>:1: SAWarning: UserDefinedType LookupType({'a': 10, 'b': 20}) will not
produce a cache key because the ``cache_ok`` flag is not set to True.
Set this flag to True if this type object's state is safe to use
in a cache key, or False to disable this warning.
symbol('no_cache')
```

如果我们**确实**设置了这样的缓存键，它将无法使用。我们将得到一个包含字典的元组结构，这个字典本身不能作为“缓存字典”中的键使用，例如 SQLAlchemy 的语句缓存，因为 Python 字典不是可哈希的：

```py
>>> # set cache_ok = True
>>> type_.cache_ok = True

>>> # this is the cache key it would generate
>>> key = type_._static_cache_key
>>> key
(<class '__main__.LookupType'>, ('lookup', {'a': 10, 'b': 20}))

>>> # however this key is not hashable, will fail when used with
>>> # SQLAlchemy statement cache
>>> some_cache = {key: "some sql value"}
Traceback (most recent call last): File "<stdin>", line 1,
in <module> TypeError: unhashable type: 'dict'
```

可通过将排序的元组分配给“.lookup”属性来使上述类型可缓存：

```py
class LookupType(UserDefinedType):
  '''a custom type that accepts a dictionary as a parameter.

 The dictionary is stored both as itself in a private variable,
 and published in a public variable as a sorted tuple of tuples,
 which is hashable and will also return the same value for any
 two equivalent dictionaries.  Note it assumes the keys and
 values of the dictionary are themselves hashable.

 '''

    cache_ok = True

    def __init__(self, lookup):
        self._lookup = lookup

        # assume keys/values of "lookup" are hashable; otherwise
        # they would also need to be converted in some way here
        self.lookup = tuple(
            (key, lookup[key]) for key in sorted(lookup)
        )

    def get_col_spec(self, **kw):
        return "VARCHAR(255)"

    def bind_processor(self, dialect):
        # ...  works with "self._lookup" ...
```

在上面的情况下，`LookupType({"a": 10, "b": 20})`的缓存键将为：

```py
>>> LookupType({"a": 10, "b": 20})._static_cache_key
(<class '__main__.LookupType'>, ('lookup', (('a', 10), ('b', 20))))
```

新版本 1.4.14 中新增了：- 添加了`cache_ok`标志，允许对`TypeDecorator`类的缓存进行一些可配置性。

新版本 1.4.28 中新增了：- 添加了`ExternalType`混合类型，它将`cache_ok`标志推广到了`TypeDecorator`和`UserDefinedType`类。

另请参阅

SQL 编译缓存

```py
method coerce_compared_value(op: OperatorType | None, value: Any) → TypeEngine[Any]
```

为表达式中的“强制转换”Python 值建议一种类型。

`UserDefinedType` 的默认行为与 `TypeDecorator` 的默认行为相同；默认情况下，它返回 `self`，假设比较的值应该被强制转换为与此相同的类型。有关更多详细信息，请参见 `TypeDecorator.coerce_compared_value()`。

```py
attribute ensure_kwarg: str = 'get_col_spec'
```

一个用于指示方法名称的正则表达式，该方法应接受`**kw`参数。

类将扫描匹配名称模板的方法，并在必要时装饰它们，以确保接受`**kw`参数。

## 使用自定义类型和反射

需要注意的是，被修改以具有额外的 Python 行为的数据库类型，包括基于`TypeDecorator`的类型以及其他用户定义的数据类型子类，在数据库模式中没有任何表示。当使用数据库中描述的反射功能时，SQLAlchemy 使用一个固定的映射，将数据库服务器报告的数据类型信息链接到一个 SQLAlchemy 数据类型对象上。例如，如果我们在 PostgreSQL 模式中查看特定数据库列的定义，可能会收到字符串`"VARCHAR"`。SQLAlchemy 的 PostgreSQL 方言有一个硬编码的映射，将字符串名称`"VARCHAR"`链接到 SQLAlchemy `VARCHAR` 类，这就是当我们发出像`Table('my_table', m, autoload_with=engine)`这样的语句时，其中的 `Column` 对象内会有一个 `VARCHAR` 的实例存在的原因。

这意味着如果一个 `Table` 对象使用的类型对象不直接对应于数据库本机类型名称，如果我们在其他地方使用反射为此数据库表创建新的 `Table` 对象，则它将没有此数据类型。例如：

```py
>>> from sqlalchemy import (
...     Table,
...     Column,
...     MetaData,
...     create_engine,
...     PickleType,
...     Integer,
... )
>>> metadata = MetaData()
>>> my_table = Table(
...     "my_table", metadata, Column("id", Integer), Column("data", PickleType)
... )
>>> engine = create_engine("sqlite://", echo="debug")
>>> my_table.create(engine)
INFO  sqlalchemy.engine.base.Engine
CREATE  TABLE  my_table  (
  id  INTEGER,
  data  BLOB
) 
```

在上面，我们使用了`PickleType`，它是一个作用于`LargeBinary`数据类型之上的`TypeDecorator`，在 SQLite 中对应着数据库类型`BLOB`。在 CREATE TABLE 中，我们可以看到使用了`BLOB`数据类型。SQLite 数据库对我们使用的`PickleType`一无所知。

如果我们看一下`my_table.c.data.type`的数据类型，因为这是我们直接创建的 Python 对象，它是`PickleType`：

```py
>>> my_table.c.data.type
PickleType()
```

然而，如果我们使用反射创建另一个`Table`实例，我们创建的 SQLite 数据库中不会反映出`PickleType`的使用；相反，我们得到的是`BLOB`:

```py
>>> metadata_two = MetaData()
>>> my_reflected_table = Table("my_table", metadata_two, autoload_with=engine)
INFO  sqlalchemy.engine.base.Engine  PRAGMA  main.table_info("my_table")
INFO  sqlalchemy.engine.base.Engine  ()
DEBUG  sqlalchemy.engine.base.Engine  Col  ('cid',  'name',  'type',  'notnull',  'dflt_value',  'pk')
DEBUG  sqlalchemy.engine.base.Engine  Row  (0,  'id',  'INTEGER',  0,  None,  0)
DEBUG  sqlalchemy.engine.base.Engine  Row  (1,  'data',  'BLOB',  0,  None,  0)

>>>  my_reflected_table.c.data.type
BLOB() 
```

通常，当应用程序使用自定义类型定义明确的`Table`元数据时，不需要使用表反射，因为必要的`Table`元数据已经存在。然而，对于一个应用程序或一组应用程序需要同时使用包含自定义 Python 级数据类型的明确`Table`元数据以及设置其`Column`对象作为从数据库反映的`Table`对象的情况，仍然需要展示自定义数据类型的附加 Python 行为，必须采取额外的步骤来允许这种情况。

最直接的方法是按照覆盖反射列中描述的覆盖特定列。在这种技术中，我们只需将反射与那些我们想要使用自定义或装饰数据类型的列的显式`Column`对象结合使用：

```py
>>> metadata_three = MetaData()
>>> my_reflected_table = Table(
...     "my_table",
...     metadata_three,
...     Column("data", PickleType),
...     autoload_with=engine,
... )
```

上面的`my_reflected_table`对象被反映出来，并将从 SQLite 数据库加载“id”列的定义。但对于“data”列，我们用一个显式的`Column`定义来覆盖了反射对象，其中包括我们想要的 Python 数据类型，`PickleType`。反射过程将保留此`Column`对象不变：

```py
>>> my_reflected_table.c.data.type
PickleType()
```

从数据库本地类型对象转换为自定义数据类型的更详细的方法是使用`DDLEvents.column_reflect()`事件处理程序。例如，如果我们知道我们想要的所有`BLOB`数据类型实际上都是`PickleType`，我们可以设置一个跨越整个的规则：

```py
from sqlalchemy import BLOB
from sqlalchemy import event
from sqlalchemy import PickleType
from sqlalchemy import Table

@event.listens_for(Table, "column_reflect")
def _setup_pickletype(inspector, table, column_info):
    if isinstance(column_info["type"], BLOB):
        column_info["type"] = PickleType()
```

当上述代码在任何表反射发生之前*调用*（还要注意它应该在应用程序中**仅调用一次**，因为它是一个全局规则）时，对于包含具有`BLOB`数据类型列的任何`Table`，结果数据类型将存储在`Column`对象中作为`PickleType`。

实际上，上述基于事件的方法可能会有额外的规则，以便仅影响那些数据类型很重要的列，例如表名和可能列名的查找表，或者其他启发式方法，以准确确定应该用 Python 数据类型建立哪些列。
