# Automap

> 原文：[`docs.sqlalchemy.org/en/20/orm/extensions/automap.html`](https://docs.sqlalchemy.org/en/20/orm/extensions/automap.html)

定义一个扩展到`sqlalchemy.ext.declarative`系统的系统，自动生成从数据库模式到映射类和关系，通常而不一定是一个反射的数据库模式。

希望`AutomapBase`系统提供了一个快速和现代化的解决方案，解决了非常著名的[SQLSoup](https://sqlsoup.readthedocs.io/en/latest/)也试图解决的问题，即从现有数据库动态生成快速和基本的对象模型。通过严格在映射器配置级别解决该问题，并与现有的声明类技术完全集成，`AutomapBase`试图提供一个与问题紧密集成的方法，以迅速自动生成临时映射。

提示

Automap 扩展针对“零声明”方法，其中可以从数据库模式动态生成包括类和预命名关系在内的完整 ORM 模型。对于仍希望使用显式类声明以及与表反射结合使用的显式关系定义的应用程序，描述在使用 DeferredReflection 中的`DeferredReflection`类是更好的选择。

## 基本用法

最简单的用法是将现有数据库反映到一个新模型中。我们创建一个新的`AutomapBase`类，方式类似于我们创建声明性基类，使用`automap_base()`。然后，我们调用`AutomapBase.prepare()`在生成的基类上，要求它反映模式并生成映射：

```py
from sqlalchemy.ext.automap import automap_base
from sqlalchemy.orm import Session
from sqlalchemy import create_engine

Base = automap_base()

# engine, suppose it has two tables 'user' and 'address' set up
engine = create_engine("sqlite:///mydatabase.db")

# reflect the tables
Base.prepare(autoload_with=engine)

# mapped classes are now created with names by default
# matching that of the table name.
User = Base.classes.user
Address = Base.classes.address

session = Session(engine)

# rudimentary relationships are produced
session.add(Address(email_address="foo@bar.com", user=User(name="foo")))
session.commit()

# collection-based relationships are by default named
# "<classname>_collection"
u1 = session.query(User).first()
print(u1.address_collection)
```

在上面，调用`AutomapBase.prepare()`并传递`AutomapBase.prepare.reflect`参数，表示将在此声明基类的`MetaData`集合上调用`MetaData.reflect()`方法; 然后，`MetaData`中的每个** viable **`Table`都将自动生成一个新的映射类。将连接各个表的`ForeignKeyConstraint`对象将用于在类之间生成新的双向`relationship()`对象。类和关系遵循一个默认命名方案，我们可以自定义。在这一点上，我们基本的映射包含了相关的`User`和`Address`类，可以以传统方式使用。

注意

通过** viable **，我们指的是表必须指定主键才能进行映射。此外，如果检测到表是两个其他表之间的纯关联表，则不会直接映射该表，而是将其配置为两个引用表的映射之间的多对多表。

## 从现有元数据生成映射

我们可以将预先声明的`MetaData`对象传递给`automap_base()`。该对象可以以任何方式构造，包括以编程方式、从序列化文件或从使用`MetaData.reflect()`反映的自身构造。下面我们演示了反射和显式表声明的组合：

```py
from sqlalchemy import create_engine, MetaData, Table, Column, ForeignKey
from sqlalchemy.ext.automap import automap_base

engine = create_engine("sqlite:///mydatabase.db")

# produce our own MetaData object
metadata = MetaData()

# we can reflect it ourselves from a database, using options
# such as 'only' to limit what tables we look at...
metadata.reflect(engine, only=["user", "address"])

# ... or just define our own Table objects with it (or combine both)
Table(
    "user_order",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("user_id", ForeignKey("user.id")),
)

# we can then produce a set of mappings from this MetaData.
Base = automap_base(metadata=metadata)

# calling prepare() just sets up mapped classes and relationships.
Base.prepare()

# mapped classes are ready
User = Base.classes.user
Address = Base.classes.address
Order = Base.classes.user_order
```

## 从多个模式生成映射

当使用反射时，`AutomapBase.prepare()` 方法一次最多只能从一个模式中反射表，使用 `AutomapBase.prepare.schema` 参数来指示要从中反射的模式的名称。为了从多个模式中填充 `AutomapBase` 中的表，可以多次调用 `AutomapBase.prepare()`，每次将不同的名称传递给 `AutomapBase.prepare.schema` 参数。`AutomapBase.prepare()` 方法会保持一个内部列表，其中包含已经映射过的 `Table` 对象，并且只会为自上次运行 `AutomapBase.prepare()` 以来新增的那些 `Table` 对象添加新的映射：

```py
e = create_engine("postgresql://scott:tiger@localhost/test")

Base.metadata.create_all(e)

Base = automap_base()

Base.prepare(e)
Base.prepare(e, schema="test_schema")
Base.prepare(e, schema="test_schema_2")
```

新版本 2.0 中新增了 `AutomapBase.prepare()` 方法，可以任意调用；每次运行时只会映射新增的表。在 1.4 版本及之前的版本中，多次调用会导致错误，因为它会尝试重新映射已经映射过的类。之前的解决方法是直接调用 `MetaData.reflect()`，该方法仍然可用。

### 跨多个模式自动映射同名表

对于多个模式可能有同名表的常见情况，因此可能生成同名类，可以通过使用 `AutomapBase.prepare.classname_for_table` 钩子在每个模式基础上应用不同的类名来解决冲突，或者通过使用 `AutomapBase.prepare.modulename_for_table` 钩子来解决同名类的歧义，该钩子允许通过更改它们的有效 `__module__` 属性来区分同名类。在下面的示例中，此钩子用于为所有类创建一个 `__module__` 属性，其形式为 `mymodule.<schemaname>`，如果没有模式，则使用模式名称 `default`：

```py
e = create_engine("postgresql://scott:tiger@localhost/test")

Base.metadata.create_all(e)

def module_name_for_table(cls, tablename, table):
    if table.schema is not None:
        return f"mymodule.{table.schema}"
    else:
        return f"mymodule.default"

Base = automap_base()

Base.prepare(e, modulename_for_table=module_name_for_table)
Base.prepare(e, schema="test_schema", modulename_for_table=module_name_for_table)
Base.prepare(e, schema="test_schema_2", modulename_for_table=module_name_for_table)
```

同名类被组织成可在 `AutomapBase.by_module` 中使用的分层集合。使用特定包/模块的点分隔名称向下遍历到所需的类名。

注意

当使用 `AutomapBase.prepare.modulename_for_table` 钩子来返回一个不是 `None` 的新 `__module__` 时，类**不会**被放入 `AutomapBase.classes` 集合中；只有那些没有给定显式模块名的类才会放在这里，因为该集合不能单独表示同名类。

在上面的示例中，如果数据库中包含了三个默认模式、`test_schema` 模式和 `test_schema_2` 模式中都命名为 `accounts` 的表，将会有三个单独的类可用，分别是：

```py
Base.by_module.mymodule.default.accounts
Base.by_module.mymodule.test_schema.accounts
Base.by_module.mymodule.test_schema_2.accounts
```

对于所有 `AutomapBase` 类生成的默认模块命名空间是 `sqlalchemy.ext.automap`。如果没有使用 `AutomapBase.prepare.modulename_for_table` 钩子，`AutomapBase.by_module` 的内容将完全在 `sqlalchemy.ext.automap` 命名空间内（例如 `MyBase.by_module.sqlalchemy.ext.automap.<classname>`），其中将包含与 `AutomapBase.classes` 中看到的相同系列的类。因此，只有在存在显式 `__module__` 约定时才通常需要使用 `AutomapBase.by_module`。

## 显式指定类

提示

如果在应用程序中期望显式类占据主要地位，请考虑改用 `DeferredReflection`。

`automap` 扩展允许类被明确定义，类似于`DeferredReflection`类的方式。从`AutomapBase`继承的类表现得像常规的声明类，但在构建后不会立即映射，而是在调用`AutomapBase.prepare()`时映射。`AutomapBase.prepare()`方法将利用我们根据使用的表名建立的类。如果我们的模式包含表`user`和`address`，我们可以定义要使用的一个或两个类：

```py
from sqlalchemy.ext.automap import automap_base
from sqlalchemy import create_engine

# automap base
Base = automap_base()

# pre-declare User for the 'user' table
class User(Base):
    __tablename__ = "user"

    # override schema elements like Columns
    user_name = Column("name", String)

    # override relationships too, if desired.
    # we must use the same name that automap would use for the
    # relationship, and also must refer to the class name that automap will
    # generate for "address"
    address_collection = relationship("address", collection_class=set)

# reflect
engine = create_engine("sqlite:///mydatabase.db")
Base.prepare(autoload_with=engine)

# we still have Address generated from the tablename "address",
# but User is the same as Base.classes.User now

Address = Base.classes.address

u1 = session.query(User).first()
print(u1.address_collection)

# the backref is still there:
a1 = session.query(Address).first()
print(a1.user)
```

在上面，更复杂的细节之一是，我们展示了覆盖`relationship()`对象的过程，这是 automap 会创建的。为了做到这一点，我们需要确保名称与 automap 通常生成的名称匹配，即关系名称将是`User.address_collection`，而从 automap 的角度来看，所指的类的名称被称为`address`，尽管我们在使用这个类时将其称为`Address`。

## 覆盖命名方案

`automap` 负责根据模式生成映射类和关系名称，这意味着它在确定这些名称时有决策点。这三个决策点是使用函数提供的，这些函数可以传递给`AutomapBase.prepare()`方法，并被称为`classname_for_table()`、`name_for_scalar_relationship()`和`name_for_collection_relationship()`。以下示例中提供了任意或所有这些函数，我们使用“驼峰命名法”为类名和使用 [Inflect](https://pypi.org/project/inflect) 包的“复数形式”为集合名：

```py
import re
import inflect

def camelize_classname(base, tablename, table):
    "Produce a 'camelized' class name, e.g."
    "'words_and_underscores' -> 'WordsAndUnderscores'"

    return str(
        tablename[0].upper()
        + re.sub(
            r"_([a-z])",
            lambda m: m.group(1).upper(),
            tablename[1:],
        )
    )

_pluralizer = inflect.engine()

def pluralize_collection(base, local_cls, referred_cls, constraint):
    "Produce an 'uncamelized', 'pluralized' class name, e.g."
    "'SomeTerm' -> 'some_terms'"

    referred_name = referred_cls.__name__
    uncamelized = re.sub(
        r"[A-Z]",
        lambda m: "_%s" % m.group(0).lower(),
        referred_name,
    )[1:]
    pluralized = _pluralizer.plural(uncamelized)
    return pluralized

from sqlalchemy.ext.automap import automap_base

Base = automap_base()

engine = create_engine("sqlite:///mydatabase.db")

Base.prepare(
    autoload_with=engine,
    classname_for_table=camelize_classname,
    name_for_collection_relationship=pluralize_collection,
)
```

根据上述映射，我们现在将拥有`User`和`Address`两个类，其中从`User`到`Address`的集合被称为`User.addresses`：

```py
User, Address = Base.classes.User, Base.classes.Address

u1 = User(addresses=[Address(email="foo@bar.com")])
```

## 关系检测

自动映射所实现的绝大部分是基于外键生成 `relationship()` 结构。其工作原理如下：

1.  检查已知映射到特定类的给定 `Table` 是否存在`ForeignKeyConstraint` 对象。

1.  对于每个 `ForeignKeyConstraint`，将匹配到的远程`Table`对象与其应映射到的类相匹配，如果有的话，否则将跳过。

1.  由于我们正在检查的 `ForeignKeyConstraint` 对应于来自直接映射类的引用，因此关系将被设置为指向引用类的多对一关系；在引用类上将创建相应的一个对多反向引用，引用此类。

1.  如果属于`ForeignKeyConstraint` 的任何列不可为空（例如 `nullable=False`），则将在要传递给关系或反向引用的关键字参数中添加一个 `relationship.cascade` 关键字参数，其值为 `all, delete-orphan`。如果`ForeignKeyConstraint` 报告对于一组非空列设置了 `ForeignKeyConstraint.ondelete` 为 `CASCADE`，或者对于可为空列设置了 `SET NULL`，则在关系关键字参数集合中将选项`relationship.passive_deletes`标志设置为 `True`。请注意，并非所有后端都支持对 ON DELETE 的反射。

1.  关系的名称是使用`AutomapBase.prepare.name_for_scalar_relationship`和`AutomapBase.prepare.name_for_collection_relationship`可调用函数确定的。重要的是要注意，默认关系命名是从**实际类名**派生的。如果您通过声明给出了特定类的显式名称，或者指定了备用类命名方案，那么关系名称将从该名称派生。

1.  对于这些名称，类被检查是否存在匹配的已映射属性。如果在一侧检测到一个，但在另一侧没有，则`AutomapBase`尝试在缺失的一侧创建一个关系，然后使用`relationship.back_populates`参数将新关系指向另一侧。

1.  在通常情况下，如果任一侧都没有关系，则`AutomapBase.prepare()`会在“多对一”一侧生成一个`relationship()`，并使用`relationship.backref`参数将其与另一侧匹配。

1.  `relationship()`的生成以及可选地`backref()`的生成由`AutomapBase.prepare.generate_relationship`函数处理，该函数可以由最终用户提供，以增强传递给`relationship()`或`backref()`的参数或者使用这些函数的自定义实现。

### 自定义关系参数

`AutomapBase.prepare.generate_relationship` 钩子可用于向关系添加参数。对于大多数情况，我们可以利用现有的 `generate_relationship()` 函数，在使用我们自己的参数扩充给定的关键字字典后，返回对象。

下面是如何将 `relationship.cascade` 和 `relationship.passive_deletes` 选项传递给所有一对多关系的示例：

```py
from sqlalchemy.ext.automap import generate_relationship
from sqlalchemy.orm import interfaces

def _gen_relationship(
    base, direction, return_fn, attrname, local_cls, referred_cls, **kw
):
    if direction is interfaces.ONETOMANY:
        kw["cascade"] = "all, delete-orphan"
        kw["passive_deletes"] = True
    # make use of the built-in function to actually return
    # the result.
    return generate_relationship(
        base, direction, return_fn, attrname, local_cls, referred_cls, **kw
    )

from sqlalchemy.ext.automap import automap_base
from sqlalchemy import create_engine

# automap base
Base = automap_base()

engine = create_engine("sqlite:///mydatabase.db")
Base.prepare(autoload_with=engine, generate_relationship=_gen_relationship)
```

### 多对多关系

`automap` 将生成多对多关系，例如包含 `secondary` 参数的关系。生成这些关系的过程如下：

1.  在任何映射类被分配给它之前，给定的 `Table` 将被检查是否包含 `ForeignKeyConstraint` 对象。

1.  如果表包含两个且仅两个 `ForeignKeyConstraint` 对象，并且此表中的所有列都是这两个 `ForeignKeyConstraint` 对象的成员，则假定该表是“secondary”表，并且**不会直接映射**。

1.  `Table` 所指向的两个（或一个，用于自引用）外部表将与它们将要映射到的类进行匹配，如果有的话。

1.  如果双方的映射类位于同一位置，则在两个类之间创建一个双向的多对多 `relationship()` / `backref()` 对。

1.  多对多的覆盖逻辑与一对多/多对一的相同；在调用 `generate_relationship()` 函数生成结构后，现有属性将被保留。

### 具有继承关系的关系

`automap` 不会在处于继承关系的两个类之间生成任何关系。也就是说，对于以下给定的两个类：

```py
class Employee(Base):
    __tablename__ = "employee"
    id = Column(Integer, primary_key=True)
    type = Column(String(50))
    __mapper_args__ = {
        "polymorphic_identity": "employee",
        "polymorphic_on": type,
    }

class Engineer(Employee):
    __tablename__ = "engineer"
    id = Column(Integer, ForeignKey("employee.id"), primary_key=True)
    __mapper_args__ = {
        "polymorphic_identity": "engineer",
    }
```

`Engineer` 到 `Employee` 的外键不是用于建立关系，而是用于在两个类之间建立连接的继承关系。

请注意，这意味着自动映射将不会为从子类到父类的外键生成 *任何* 关系。如果一个映射还具有从子类到父类的实际关系，那么这些关系需要是显式的。在下面的例子中，由于 `Engineer` 到 `Employee` 有两个单独的外键，我们需要设置我们想要的关系以及 `inherit_condition`，因为这些都不是 SQLAlchemy 可以猜测的：

```py
class Employee(Base):
    __tablename__ = "employee"
    id = Column(Integer, primary_key=True)
    type = Column(String(50))

    __mapper_args__ = {
        "polymorphic_identity": "employee",
        "polymorphic_on": type,
    }

class Engineer(Employee):
    __tablename__ = "engineer"
    id = Column(Integer, ForeignKey("employee.id"), primary_key=True)
    favorite_employee_id = Column(Integer, ForeignKey("employee.id"))

    favorite_employee = relationship(Employee, foreign_keys=favorite_employee_id)

    __mapper_args__ = {
        "polymorphic_identity": "engineer",
        "inherit_condition": id == Employee.id,
    }
```

### 处理简单的命名冲突

在映射过程中如果出现命名冲突的情况，根据需要覆盖 `classname_for_table()`、`name_for_scalar_relationship()` 和 `name_for_collection_relationship()` 中的任何一个。例如，如果自动映射尝试将一个多对一关系命名为一个现有列相同的名称，可以有条件地选择替代约定。给定一个模式：

```py
CREATE  TABLE  table_a  (
  id  INTEGER  PRIMARY  KEY
);

CREATE  TABLE  table_b  (
  id  INTEGER  PRIMARY  KEY,
  table_a  INTEGER,
  FOREIGN  KEY(table_a)  REFERENCES  table_a(id)
);
```

上述模式将首先将 `table_a` 表自动映射为名为 `table_a` 的类；然后将在 `table_b` 的类上自动映射一个与此相关类相同名称的关系，例如 `table_a`。这个关系名称与映射列 `table_b.table_a` 冲突，并且将在映射时发出错误。

我们可以通过以下方式使用下划线解决这个冲突：

```py
def name_for_scalar_relationship(base, local_cls, referred_cls, constraint):
    name = referred_cls.__name__.lower()
    local_table = local_cls.__table__
    if name in local_table.columns:
        newname = name + "_"
        warnings.warn("Already detected name %s present.  using %s" % (name, newname))
        return newname
    return name

Base.prepare(
    autoload_with=engine,
    name_for_scalar_relationship=name_for_scalar_relationship,
)
```

或者，我们可以在列的一侧更改名称。可以使用在 Naming Declarative Mapped Columns Explicitly 中描述的技术修改映射的列，通过将列显式地分配给一个新名称：

```py
Base = automap_base()

class TableB(Base):
    __tablename__ = "table_b"
    _table_a = Column("table_a", ForeignKey("table_a.id"))

Base.prepare(autoload_with=engine)
```

## 使用明确声明的自动映射

正如前面所述，自动映射不依赖于反射，并且可以利用`MetaData` 集合内的任何 `Table` 对象集合。由此可见，自动映射也可以在完全定义了表元数据的完整模型中生成丢失的关系：

```py
from sqlalchemy.ext.automap import automap_base
from sqlalchemy import Column, Integer, String, ForeignKey

Base = automap_base()

class User(Base):
    __tablename__ = "user"

    id = Column(Integer, primary_key=True)
    name = Column(String)

class Address(Base):
    __tablename__ = "address"

    id = Column(Integer, primary_key=True)
    email = Column(String)
    user_id = Column(ForeignKey("user.id"))

# produce relationships
Base.prepare()

# mapping is complete, with "address_collection" and
# "user" relationships
a1 = Address(email="u1")
a2 = Address(email="u2")
u1 = User(address_collection=[a1, a2])
assert a1.user is u1
```

在上面的例子中，对于大部分完成的 `User` 和 `Address` 映射，我们在 `Address.user_id` 上定义的 `ForeignKey` 允许在映射的类上生成一个双向关系对 `Address.user` 和 `User.address_collection`。 

注意，当子类化`AutomapBase`时，需要调用`AutomapBase.prepare()`方法；如果不调用，我们声明的类处于未映射状态。

## 拦截列定义

`MetaData` 和 `Table` 对象支持一个事件钩子`DDLEvents.column_reflect()`，可用于拦截关于数据库列反射的信息，在构建`Column`对象之前。例如，如果我们想要使用类似`"attr_<columnname>"`的命名约定来映射列，可以应用该事件：

```py
@event.listens_for(Base.metadata, "column_reflect")
def column_reflect(inspector, table, column_info):
    # set column.key = "attr_<lower_case_name>"
    column_info["key"] = "attr_%s" % column_info["name"].lower()

# run reflection
Base.prepare(autoload_with=engine)
```

从版本 1.4.0b2 开始：`DDLEvents.column_reflect()`事件可以应用于`MetaData`对象。

另请参阅

`DDLEvents.column_reflect()`

自动从反射表中命名列 - 在 ORM 映射文档中

## API 参考

| 对象名称 | 描述 |
| --- | --- |
| automap_base([declarative_base], **kw) | 生成一个声明式自动映射基类。 |
| AutomapBase | 用于“自动映射”模式的基类。 |
| classname_for_table(base, tablename, table) | 返回应用于给定表名的类名。 |
| generate_relationship(base, direction, return_fn, attrname, ..., **kw) | 代表两个映射类生成一个`relationship()`或`backref()`。 |
| name_for_collection_relationship(base, local_cls, referred_cls, constraint) | 返回应用于从一个类到另一个类的集合引用的属性名称。 |
| name_for_scalar_relationship(base, local_cls, referred_cls, constraint) | 返回应用于标量对象引用的一个类到另一个类的属性名称。 |

```py
function sqlalchemy.ext.automap.automap_base(declarative_base: Type[Any] | None = None, **kw: Any) → Any
```

生成一个声明式自动映射基类。

此函数生成一个新的基类，它是 `AutomapBase` 类的产品，以及由 `declarative_base()` 生成的一个声明基类。

除了 `declarative_base` 外，所有参数都是直接传递给 `declarative_base()` 函数的关键字参数。

参数：

+   `declarative_base` – 由 `declarative_base()` 生成的现有类。当传递了这个参数时，函数不再调用 `declarative_base()` 本身，所有其他关键字参数都会被忽略。

+   `**kw` – 关键字参数被传递给 `declarative_base()`。

```py
class sqlalchemy.ext.automap.AutomapBase
```

用于“自动映射”模式的基类。

`AutomapBase` 类可以与由 `declarative_base()` 函数生成的“声明基类”类相比较。实际上，`AutomapBase` 类总是与实际的声明基类一起使用作为一个 mixin。

一个新的可子类化的 `AutomapBase` 通常使用 `automap_base()` 函数实例化。

**成员**

by_module, classes, metadata, prepare()

另请参阅

自动映射

```py
attribute by_module: ClassVar[ByModuleProperties]
```

一个包含点分隔的模块名称层次结构链接到类的 `Properties` 实例。

这个集合是一个替代 `AutomapBase.classes` 集合的选择，当使用 `AutomapBase.prepare.modulename_for_table` 参数时，这个参数将为生成的类应用不同的 `__module__` 属性。

自动映射生成的类的默认 `__module__` 是 `sqlalchemy.ext.automap`；使用 `AutomapBase.by_module` 访问这个命名空间会像这样：

```py
User = Base.by_module.sqlalchemy.ext.automap.User
```

如果一个类的 `__module__` 是 `mymodule.account`，访问这个命名空间会像这样：

```py
MyClass = Base.by_module.mymodule.account.MyClass
```

新特性在版本 2.0 中添加。

另请参阅

从多个模式生成映射

```py
attribute classes: ClassVar[Properties[Type[Any]]]
```

包含类的 `Properties` 实例。

这个对象的行为类似于表上的 `.c` 集合。类以它们被赋予的名称呈现，例如：

```py
Base = automap_base()
Base.prepare(autoload_with=some_engine)

User, Address = Base.classes.User, Base.classes.Address
```

对于类名与 `Properties` 方法名重叠的情况，比如 `items()`，也支持获取项的形式：

```py
Item = Base.classes["items"]
```

```py
attribute metadata: ClassVar[MetaData]
```

指的是将用于新 `Table` 对象的 `MetaData` 集合。

另请参见

访问表和元数据

```py
classmethod prepare(autoload_with: Engine | None = None, engine: Any | None = None, reflect: bool = False, schema: str | None = None, classname_for_table: PythonNameForTableType | None = None, modulename_for_table: PythonNameForTableType | None = None, collection_class: Any | None = None, name_for_scalar_relationship: NameForScalarRelationshipType | None = None, name_for_collection_relationship: NameForCollectionRelationshipType | None = None, generate_relationship: GenerateRelationshipType | None = None, reflection_options: Dict[_KT, _VT] | immutabledict[_KT, _VT] = {}) → None
```

从 `MetaData` 中提取映射类和关系，并执行映射。

有关完整文档和示例，请参阅 基本用法。

参数：

+   `autoload_with` – 用于执行模式反射的 `Engine` 或 `Connection`；当指定时，`MetaData.reflect()` 方法将在此方法的范围内调用。

+   `engine` –

    旧版；如果 `AutomapBase.reflect` 为 True，则用于指示反映表的 `Engine` 或 `Connection`。

    自 1.4 版开始弃用：`AutomapBase.prepare.engine` 参数已弃用，并将在未来版本中移除。请使用 `AutomapBase.prepare.autoload_with` 参数。

+   `reflect` –

    旧版；如果 `MetaData.reflect()` 应被调用，则使用 `AutomapBase.autoload_with`。

    自 1.4 版开始弃用：`AutomapBase.prepare.reflect` 参数已弃用，并将在未来版本中移除。当传递 `AutomapBase.prepare.autoload_with` 时，将启用反射。

+   `classname_for_table` – 可调用函数，用于根据表名生成新类名。默认为 `classname_for_table()`。

+   `modulename_for_table` –

    `__module__` 的有效值将由可调用函数产生，用于为内部生成的类生成模块名，以允许在单个自动映射基类中具有相同名称的多个类，这些类可能位于不同的“模块”中。

    默认为 `None`，表示 `__module__` 不会被显式设置；Python 运行时将使用值 `sqlalchemy.ext.automap` 用于这些类。

    当为生成的类分配 `__module__` 时，可以使用 `AutomapBase.by_module` 集合基于点分隔的模块名称进行访问。使用此钩子分配了显式 `__module_` 的类**不**会被放置到 `AutomapBase.classes` 集合中，只会放置到 `AutomapBase.by_module` 中。

    版本 2.0 中的新内容。

    另请参阅

    从多个模式生成映射

+   `name_for_scalar_relationship` – 用于生成标量关系的关系名称的可调用函数。默认为 `name_for_scalar_relationship()`。

+   `name_for_collection_relationship` – 用于为面向集合的关系生成关系名称的可调用函数。默认为 `name_for_collection_relationship()`。

+   `generate_relationship` – 实际生成 `relationship()` 和 `backref()` 构造的可调用函数。默认为 `generate_relationship()`。

+   `collection_class` – 当创建表示集合的新 `relationship()` 对象时将使用的 Python 集合类。默认为 `list`。

+   `schema` –

    在使用 `AutomapBase.prepare.autoload_with` 参数反射表时要反射的模式名称。名称传递给 `MetaData.reflect.schema` 参数的 `MetaData.reflect()`。当省略时，数据库连接使用的默认模式将被使用。

    注意

    `AutomapBase.prepare.schema` 参数支持一次反射单个模式。为了包含来自多个模式的表，请多次调用 `AutomapBase.prepare()`。

    对于多模式自动映射的概述，包括使用额外命名约定解决表名冲突，请参见 从多个模式生成映射 部分。

    版本 2.0 中的新功能：`AutomapBase.prepare()` 支持直接调用任意次数，跟踪已经处理过的表，以避免第二次处理它们。

+   `reflection_options` –

    当存在时，此选项字典将传递给 `MetaData.reflect()`，以提供一般的反射特定选项，如 `only` 和/或特定于方言的选项，如 `oracle_resolve_synonyms`。

    版本 1.4 中的新功能。

```py
function sqlalchemy.ext.automap.classname_for_table(base: Type[Any], tablename: str, table: Table) → str
```

返回给定表名时应该使用的类名。

默认实现是：

```py
return str(tablename)
```

可以使用 `AutomapBase.prepare.classname_for_table` 参数指定备用实现。

参数：

+   `base` – 执行准备工作的 `AutomapBase` 类。

+   `tablename` – `Table` 的字符串名称。

+   `table` – `Table` 对象本身。

返回：

一个字符串类名。

注意

在 Python 2 中，用于类名的字符串**必须**是非 Unicode 对象，例如 `str()` 对象。`Table` 的 `.name` 属性通常是 Python 的 unicode 子类，因此应该在考虑任何非 ASCII 字符后，对此名称应用 `str()` 函数。

```py
function sqlalchemy.ext.automap.name_for_scalar_relationship(base: Type[Any], local_cls: Type[Any], referred_cls: Type[Any], constraint: ForeignKeyConstraint) → str
```

返回应该用于从一个类到另一个类引用的属性名称，用于标量对象引用。

默认实现是：

```py
return referred_cls.__name__.lower()
```

可以使用 `AutomapBase.prepare.name_for_scalar_relationship` 参数指定备用实现。

参数：

+   `base` – 执行准备工作的 `AutomapBase` 类。

+   `local_cls` – 要映射到本地端的类。

+   `referred_cls` – 要映射到引用方的类。

+   `constraint` – 正在检查以产生此关系的`ForeignKeyConstraint`。

```py
function sqlalchemy.ext.automap.name_for_collection_relationship(base: Type[Any], local_cls: Type[Any], referred_cls: Type[Any], constraint: ForeignKeyConstraint) → str
```

返回应该用于从一个类引用到另一个类的属性名称，用于集合引用。

默认实现如下：

```py
return referred_cls.__name__.lower() + "_collection"
```

可以使用`AutomapBase.prepare.name_for_collection_relationship`参数指定备用实现。

参数：

+   `base` – 进行准备工作的`AutomapBase`类。

+   `local_cls` – 在本地端映射的类。

+   `referred_cls` – 在引用方的类。

+   `constraint` – 正在检查以产生此关系的`ForeignKeyConstraint`。

```py
function sqlalchemy.ext.automap.generate_relationship(base: Type[Any], direction: RelationshipDirection, return_fn: Callable[..., Relationship[Any]] | Callable[..., ORMBackrefArgument], attrname: str, local_cls: Type[Any], referred_cls: Type[Any], **kw: Any) → Relationship[Any] | ORMBackrefArgument
```

代表两个映射类生成`relationship()`或`backref()`。

可以使用`AutomapBase.prepare.generate_relationship`参数指定备用实现。

此函数的默认实现如下：

```py
if return_fn is backref:
    return return_fn(attrname, **kw)
elif return_fn is relationship:
    return return_fn(referred_cls, **kw)
else:
    raise TypeError("Unknown relationship function: %s" % return_fn)
```

参数：

+   `base` – 进行准备工作的`AutomapBase`类。

+   `direction` – 表示关系的“方向”; 这将是`ONETOMANY`、`MANYTOONE`、`MANYTOMANY`之一。

+   `return_fn` – 默认用于创建关系的函数。这将是`relationship()`或`backref()`中的一个。`backref()`函数的结果将用于在第二步产生一个新的`relationship()`，因此如果正在使用自定义关系函数，则用户定义的实现正确区分这两个函数非常关键。

+   `attrname` – 正在分配此关系的属性名称。如果`generate_relationship.return_fn`的值是`backref()`函数，则此名称是分配给反向引用的名称。

+   `local_cls` – 此关系或反向引用将在本地存在的“本地”类。

+   `referred_cls` – 关系或反向引用所指向的“被引用”类。

+   `**kw` – 所有额外的关键字参数都将传递给函数。

返回：

一个由 `generate_relationship.return_fn` 参数指定的 `relationship()` 或 `backref()` 结构。

## 基本用法

最简单的用法是将现有数据库反映到新模型中。我们以与创建声明性基类相似的方式创建一个新的 `AutomapBase` 类，使用 `automap_base()`。然后，我们调用 `AutomapBase.prepare()` 在生成的基类上，要求它反映架构并生成映射：

```py
from sqlalchemy.ext.automap import automap_base
from sqlalchemy.orm import Session
from sqlalchemy import create_engine

Base = automap_base()

# engine, suppose it has two tables 'user' and 'address' set up
engine = create_engine("sqlite:///mydatabase.db")

# reflect the tables
Base.prepare(autoload_with=engine)

# mapped classes are now created with names by default
# matching that of the table name.
User = Base.classes.user
Address = Base.classes.address

session = Session(engine)

# rudimentary relationships are produced
session.add(Address(email_address="foo@bar.com", user=User(name="foo")))
session.commit()

# collection-based relationships are by default named
# "<classname>_collection"
u1 = session.query(User).first()
print(u1.address_collection)
```

上面，在传递 `AutomapBase.prepare.reflect` 参数时调用 `AutomapBase.prepare()` 表示将在此声明基类的 `MetaData` 集合上调用 `MetaData.reflect()` 方法；然后，每个 **viable** `Table` 在 `MetaData` 内将自动生成一个新的映射类。将连接各个表的 `ForeignKeyConstraint` 对象用于在类之间生成新的双向 `relationship()` 对象。类和关系遵循默认命名方案，我们可以自定义。在此时，我们的基本映射由相关的 `User` 和 `Address` 类组成，可以像传统方式一样使用。

注意

这里的 **viable** 意味着要将表映射，必须指定主键。此外，如果检测到表是两个其他表之间的纯关联表，则不会直接映射，而是将其配置为两个引用表的映射之间的多对多表。

## 从现有的元数据生成映射

我们可以将预先声明的`MetaData`对象传递给`automap_base()`。这个对象可以以任何方式构建，包括以编程方式、从序列化文件中或者通过`MetaData.reflect()`自身进行反射。下面我们展示了反射和显式表声明的结合使用：

```py
from sqlalchemy import create_engine, MetaData, Table, Column, ForeignKey
from sqlalchemy.ext.automap import automap_base

engine = create_engine("sqlite:///mydatabase.db")

# produce our own MetaData object
metadata = MetaData()

# we can reflect it ourselves from a database, using options
# such as 'only' to limit what tables we look at...
metadata.reflect(engine, only=["user", "address"])

# ... or just define our own Table objects with it (or combine both)
Table(
    "user_order",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("user_id", ForeignKey("user.id")),
)

# we can then produce a set of mappings from this MetaData.
Base = automap_base(metadata=metadata)

# calling prepare() just sets up mapped classes and relationships.
Base.prepare()

# mapped classes are ready
User = Base.classes.user
Address = Base.classes.address
Order = Base.classes.user_order
```

## 从多个模式生成映射

当使用反射时，`AutomapBase.prepare()`方法最多一次只能从一个模式中反射表，使用`AutomapBase.prepare.schema`参数来指示要反射的模式的名称。为了将`AutomapBase`填充到来自多个模式的表中，可以多次调用`AutomapBase.prepare()`，每次传递不同的名称给`AutomapBase.prepare.schema`参数。`AutomapBase.prepare()`方法会保留一个已经映射过的`Table`对象的内部列表，并且只会为那些自上次运行`AutomapBase.prepare()`以来新的`Table`对象添加新的映射：

```py
e = create_engine("postgresql://scott:tiger@localhost/test")

Base.metadata.create_all(e)

Base = automap_base()

Base.prepare(e)
Base.prepare(e, schema="test_schema")
Base.prepare(e, schema="test_schema_2")
```

2.0 版本新增功能：`AutomapBase.prepare()`方法可以被任意次数调用；每次运行只会映射新添加的表。在 1.4 版本及更早版本中，多次调用会导致错误，因为它会尝试重新映射已经映射的类。直接调用`MetaData.reflect()`的先前解决方法仍然可用。

### 在多个模式中自动映射同名表

对于常见情况，即多个模式可能具有相同命名的表，因此可能生成相同命名的类，可以通过使用`AutomapBase.prepare.classname_for_table`挂钩来在每个模式基础上应用不同的类名来解决冲突，或者使用`AutomapBase.prepare.modulename_for_table`挂钩，通过更改它们的有效`__module__`属性来消除同名类的歧义。在下面的示例中，此挂钩用于为所有类创建一个`__module__`属性，其形式为`mymodule.<schemaname>`，其中如果没有模式，则使用模式名为`default`：

```py
e = create_engine("postgresql://scott:tiger@localhost/test")

Base.metadata.create_all(e)

def module_name_for_table(cls, tablename, table):
    if table.schema is not None:
        return f"mymodule.{table.schema}"
    else:
        return f"mymodule.default"

Base = automap_base()

Base.prepare(e, modulename_for_table=module_name_for_table)
Base.prepare(e, schema="test_schema", modulename_for_table=module_name_for_table)
Base.prepare(e, schema="test_schema_2", modulename_for_table=module_name_for_table)
```

相同命名的类被组织成一个层次化的集合，可在`AutomapBase.by_module`中使用。该集合使用特定包/模块的点分隔名称进行遍历，直到所需的类名。

注意

当使用`AutomapBase.prepare.modulename_for_table`挂钩返回一个不是`None`的新`__module__`时，类**不会**放置到`AutomapBase.classes`集合中；只有没有给定显式模块名称的类才会放在这里，因为该集合无法表示同名类。

在上面的示例中，如果数据库中包含默认模式，`test_schema`模式和`test_schema_2`模式中的一个名为`accounts`的表，那么将会有三个不同的类可用：

```py
Base.by_module.mymodule.default.accounts
Base.by_module.mymodule.test_schema.accounts
Base.by_module.mymodule.test_schema_2.accounts
```

对于所有`AutomapBase`类生成的默认模块命名空间是`sqlalchemy.ext.automap`。如果没有使用`AutomapBase.prepare.modulename_for_table`挂钩，则`AutomapBase.by_module`的内容将完全在`sqlalchemy.ext.automap`命名空间内（例如`MyBase.by_module.sqlalchemy.ext.automap.<classname>`），其中包含与`AutomapBase.classes`中看到的相同的一系列类。因此，通常只有在存在显式`__module__`约定时才需要使用`AutomapBase.by_module`。

### 在跨多个模式自动映射同名表时

对于常见情况，即多个模式可能具有相同命名的表，因此会生成相同命名的类，可以通过使用`AutomapBase.prepare.classname_for_table`钩子来根据每个模式应用不同的类名来解决冲突，或者通过使用`AutomapBase.prepare.modulename_for_table`钩子来解决相同命名类的歧义问题，该钩子允许通过更改它们的有效`__module__`属性来区分相同命名的类。在下面的示例中，该钩子用于创建一个形式为`mymodule.<schemaname>`的`__module__`属性，其中如果不存在模式，则使用模式名称`default`：

```py
e = create_engine("postgresql://scott:tiger@localhost/test")

Base.metadata.create_all(e)

def module_name_for_table(cls, tablename, table):
    if table.schema is not None:
        return f"mymodule.{table.schema}"
    else:
        return f"mymodule.default"

Base = automap_base()

Base.prepare(e, modulename_for_table=module_name_for_table)
Base.prepare(e, schema="test_schema", modulename_for_table=module_name_for_table)
Base.prepare(e, schema="test_schema_2", modulename_for_table=module_name_for_table)
```

相同命名的类被组织成一个层次结构集合，可在`AutomapBase.by_module`中使用。该集合通过特定包/模块的点分隔名称向下遍历到所需的类名。

注意

当使用`AutomapBase.prepare.modulename_for_table`钩子返回一个不是`None`的新`__module__`时，该类**不会**被放置到`AutomapBase.classes`集合中；只有那些没有给定显式模块名的类会被放置在此处，因为集合不能单独表示同名类。

在上述示例中，如果数据库中包含了三个默认模式、`test_schema`模式和`test_schema_2`模式中都命名为`accounts`的表，则会分别获得三个单独的类：

```py
Base.by_module.mymodule.default.accounts
Base.by_module.mymodule.test_schema.accounts
Base.by_module.mymodule.test_schema_2.accounts
```

为所有`AutomapBase`类生成的默认模块命名空间是`sqlalchemy.ext.automap`。 如果未使用`AutomapBase.prepare.modulename_for_table`挂钩，则`AutomapBase.by_module`的内容将完全在`sqlalchemy.ext.automap`命名空间内（例如，`MyBase.by_module.sqlalchemy.ext.automap.<classname>`），其中包含与`AutomapBase.classes`中看到的相同系列的类。 因此，仅当存在显式的`__module__`约定时才通常需要使用`AutomapBase.by_module`。

## 明确指定类

提示

如果明确的类在应用程序中占主导地位，请考虑改用`DeferredReflection`。

`automap`扩展允许以与`DeferredReflection`类相似的方式明确定义类。 从`AutomapBase`继承的类表现得像常规的声明性类一样，但在构造后不会立即映射，而是在调用`AutomapBase.prepare()`时映射。 `AutomapBase.prepare()`方法将利用我们基于所使用的表名建立的类。 如果我们的模式包含表`user`和`address`，我们可以定义要使用的一个或两个类：

```py
from sqlalchemy.ext.automap import automap_base
from sqlalchemy import create_engine

# automap base
Base = automap_base()

# pre-declare User for the 'user' table
class User(Base):
    __tablename__ = "user"

    # override schema elements like Columns
    user_name = Column("name", String)

    # override relationships too, if desired.
    # we must use the same name that automap would use for the
    # relationship, and also must refer to the class name that automap will
    # generate for "address"
    address_collection = relationship("address", collection_class=set)

# reflect
engine = create_engine("sqlite:///mydatabase.db")
Base.prepare(autoload_with=engine)

# we still have Address generated from the tablename "address",
# but User is the same as Base.classes.User now

Address = Base.classes.address

u1 = session.query(User).first()
print(u1.address_collection)

# the backref is still there:
a1 = session.query(Address).first()
print(a1.user)
```

上面，更复杂的细节之一是，我们说明了如何覆盖`relationship()`对象之一，该对象 automap 将会创建。 为此，我们需要确保名称与 automap 通常生成的名称相匹配，即关系名称将为`User.address_collection`，并且从 automap 的角度来看，所引用的类的名称称为`address`，即使我们在对此类的使用中将其称为`Address`。

## 覆盖命名方案

`automap` 被要求根据模式生成映射类和关系名称，这意味着它在确定这些名称的方式上有决策点。这三个决策点通过可以传递给`AutomapBase.prepare()`方法的函数来提供，分别称为`classname_for_table()`、`name_for_scalar_relationship()`和`name_for_collection_relationship()`。以下示例中提供了任意或全部这些函数，我们使用了“驼峰命名法”作为类名，并使用了 [Inflect](https://pypi.org/project/inflect) 包来对集合名称进行“复数化”：

```py
import re
import inflect

def camelize_classname(base, tablename, table):
    "Produce a 'camelized' class name, e.g."
    "'words_and_underscores' -> 'WordsAndUnderscores'"

    return str(
        tablename[0].upper()
        + re.sub(
            r"_([a-z])",
            lambda m: m.group(1).upper(),
            tablename[1:],
        )
    )

_pluralizer = inflect.engine()

def pluralize_collection(base, local_cls, referred_cls, constraint):
    "Produce an 'uncamelized', 'pluralized' class name, e.g."
    "'SomeTerm' -> 'some_terms'"

    referred_name = referred_cls.__name__
    uncamelized = re.sub(
        r"[A-Z]",
        lambda m: "_%s" % m.group(0).lower(),
        referred_name,
    )[1:]
    pluralized = _pluralizer.plural(uncamelized)
    return pluralized

from sqlalchemy.ext.automap import automap_base

Base = automap_base()

engine = create_engine("sqlite:///mydatabase.db")

Base.prepare(
    autoload_with=engine,
    classname_for_table=camelize_classname,
    name_for_collection_relationship=pluralize_collection,
)
```

从上面的映射中，我们现在会有 `User` 和 `Address` 两个类，其中从 `User` 到 `Address` 的集合被称为 `User.addresses`：

```py
User, Address = Base.classes.User, Base.classes.Address

u1 = User(addresses=[Address(email="foo@bar.com")])
```

## 关系检测

automap 的绝大部分工作是根据外键生成`relationship()`结构。它对于多对一和一对多关系的工作机制如下：

1.  已知映射到特定类的给定`Table`，会被检查其是否存在`ForeignKeyConstraint`对象。

1.  对于每一个`ForeignKeyConstraint`，远程的`Table`对象被匹配到其要映射的类，如果有的话，否则将被跳过。

1.  由于我们正在检查的`ForeignKeyConstraint`对应于从直接映射类的引用，该关系将被设置为指向被引用类的多对一关系；在被引用的类上将创建一个相应的一对多反向引用，指向该类。

1.  如果`ForeignKeyConstraint`的任何一列不可为空（例如，`nullable=False`），将会将`all, delete-orphan`的`relationship.cascade`关键字参数添加到要传递给关联或反向引用的关键字参数中。如果`ForeignKeyConstraint`报告对于一组非空列设置了`CASCADE`或对于可为空列设置了`SET NULL`的`ForeignKeyConstraint.ondelete`，则将在关系关键字参数集合中将选项`relationship.passive_deletes`标志设置为`True`。请注意，并非所有后端都支持删除操作的反射。

1.  关联的名称是使用`AutomapBase.prepare.name_for_scalar_relationship`和`AutomapBase.prepare.name_for_collection_relationship`可调用函数确定的。重要的是要注意，默认的关联命名从**实际类名**派生名称。如果您通过声明为特定类指定了显式名称，或指定了替代类命名方案，则关系名称将从该名称派生。

1.  检查类以查找与这些名称匹配的现有映射属性。如果在一侧检测到一个属性，但在另一侧没有，则`AutomapBase`尝试在缺失的一侧创建一个关联，然后使用`relationship.back_populates`参数指向新关联到另一侧。

1.  在通常情况下，如果任何一侧都没有关联，则`AutomapBase.prepare()`会在“多对一”一侧产生一个`relationship()`，并使用`relationship.backref`参数将其与另一侧匹配。

1.  `relationship()`的生成以及可选的`backref()`的生成被交由`AutomapBase.prepare.generate_relationship`函数处理，该函数可以由最终用户提供，以增强传递给`relationship()`或`backref()`的参数，或者利用这些函数的自定义实现。

### 自定义关系参数

`AutomapBase.prepare.generate_relationship`钩子可用于向关系添加参数。对于大多数情况，我们可以利用现有的`generate_relationship()`函数，在用自己的参数扩充给定关键字字典后返回对象。

下面是如何向所有一对多关系发送`relationship.cascade` 和`relationship.passive_deletes`选项的示例：

```py
from sqlalchemy.ext.automap import generate_relationship
from sqlalchemy.orm import interfaces

def _gen_relationship(
    base, direction, return_fn, attrname, local_cls, referred_cls, **kw
):
    if direction is interfaces.ONETOMANY:
        kw["cascade"] = "all, delete-orphan"
        kw["passive_deletes"] = True
    # make use of the built-in function to actually return
    # the result.
    return generate_relationship(
        base, direction, return_fn, attrname, local_cls, referred_cls, **kw
    )

from sqlalchemy.ext.automap import automap_base
from sqlalchemy import create_engine

# automap base
Base = automap_base()

engine = create_engine("sqlite:///mydatabase.db")
Base.prepare(autoload_with=engine, generate_relationship=_gen_relationship)
```

### 多对多关系

`automap`将生成多对多关系，例如包含`secondary`参数的关系。生成这些关系的过程如下：

1.  给定的`Table`在分配任何映射类之前将被检查其`ForeignKeyConstraint`对象。

1.  如果表包含两个且仅两个`ForeignKeyConstraint`对象，并且此表中的所有列都是这两个`ForeignKeyConstraint`对象的成员，则假定该表是“次要”表，并且**不会直接映射**。

1.  `Table`引用的两个（对于自引用的情况则为一个）外部表会与它们将要映射到的类匹配，如果有的话。

1.  如果两边的映射类被定位，那么在两个类之间将创建一个多对多的双向 `relationship()` / `backref()` 对。

1.  对于多对多的覆盖逻辑与一对多/多对一的逻辑相同；调用`generate_relationship()` 函数来生成结构，已存在的属性将被保留。

### 具有继承关系的关系

`automap` 不会在处于继承关系的两个类之间生成任何关系。 也就是说，给定以下两个类：

```py
class Employee(Base):
    __tablename__ = "employee"
    id = Column(Integer, primary_key=True)
    type = Column(String(50))
    __mapper_args__ = {
        "polymorphic_identity": "employee",
        "polymorphic_on": type,
    }

class Engineer(Employee):
    __tablename__ = "engineer"
    id = Column(Integer, ForeignKey("employee.id"), primary_key=True)
    __mapper_args__ = {
        "polymorphic_identity": "engineer",
    }
```

从`Engineer`到`Employee`的外键不是用于关系，而是用于在两个类之间建立联合继承。

请注意，这意味着 automap 将不会为从子类到超类的外键生成 *任何* 关系。 如果映射还具有从子类到超类的实际关系，那么这些关系需要显式说明。 如下，由于从`Engineer`到`Employee`有两个单独的外键，我们需要设置我们想要的关系以及`inherit_condition`，因为这些不是 SQLAlchemy 可以猜测的事情：

```py
class Employee(Base):
    __tablename__ = "employee"
    id = Column(Integer, primary_key=True)
    type = Column(String(50))

    __mapper_args__ = {
        "polymorphic_identity": "employee",
        "polymorphic_on": type,
    }

class Engineer(Employee):
    __tablename__ = "engineer"
    id = Column(Integer, ForeignKey("employee.id"), primary_key=True)
    favorite_employee_id = Column(Integer, ForeignKey("employee.id"))

    favorite_employee = relationship(Employee, foreign_keys=favorite_employee_id)

    __mapper_args__ = {
        "polymorphic_identity": "engineer",
        "inherit_condition": id == Employee.id,
    }
```

### 处理简单的命名冲突

在映射过程中出现命名冲突的情况下，根据需要覆盖 `classname_for_table()`、`name_for_scalar_relationship()` 和 `name_for_collection_relationship()` 中的任何一个。 例如，如果 automap 正试图将多对一关系命名为现有列相同的名称，可以条件地选择替代约定。 给定一个模式：

```py
CREATE  TABLE  table_a  (
  id  INTEGER  PRIMARY  KEY
);

CREATE  TABLE  table_b  (
  id  INTEGER  PRIMARY  KEY,
  table_a  INTEGER,
  FOREIGN  KEY(table_a)  REFERENCES  table_a(id)
);
```

上述模式首先将`table_a`表自动映射为一个名为`table_a`的类；然后将关系自动映射到`table_b`的类上，该关系的名称与此相关类的名称相同，例如`table_a`。 此关系名称与映射列`table_b.table_a`冲突，并且在映射时会发出错误。

我们可以通过以下方式使用下划线来解决此冲突：

```py
def name_for_scalar_relationship(base, local_cls, referred_cls, constraint):
    name = referred_cls.__name__.lower()
    local_table = local_cls.__table__
    if name in local_table.columns:
        newname = name + "_"
        warnings.warn("Already detected name %s present.  using %s" % (name, newname))
        return newname
    return name

Base.prepare(
    autoload_with=engine,
    name_for_scalar_relationship=name_for_scalar_relationship,
)
```

或者，我们可以在列方面更改名称。 可以使用在 Naming Declarative Mapped Columns Explicitly 中描述的技术来修改映射的列，通过将列显式地分配给新名称：

```py
Base = automap_base()

class TableB(Base):
    __tablename__ = "table_b"
    _table_a = Column("table_a", ForeignKey("table_a.id"))

Base.prepare(autoload_with=engine)
```

### 自定义关系参数

`AutomapBase.prepare.generate_relationship` 钩子可用于向关系添加参数。对于大多数情况，我们可以利用现有的 `generate_relationship()` 函数，在使用我们自己的参数扩充给定的关键字字典后返回对象。

下面是如何将 `relationship.cascade` 和 `relationship.passive_deletes` 选项传递给所有一对多关系的示例：

```py
from sqlalchemy.ext.automap import generate_relationship
from sqlalchemy.orm import interfaces

def _gen_relationship(
    base, direction, return_fn, attrname, local_cls, referred_cls, **kw
):
    if direction is interfaces.ONETOMANY:
        kw["cascade"] = "all, delete-orphan"
        kw["passive_deletes"] = True
    # make use of the built-in function to actually return
    # the result.
    return generate_relationship(
        base, direction, return_fn, attrname, local_cls, referred_cls, **kw
    )

from sqlalchemy.ext.automap import automap_base
from sqlalchemy import create_engine

# automap base
Base = automap_base()

engine = create_engine("sqlite:///mydatabase.db")
Base.prepare(autoload_with=engine, generate_relationship=_gen_relationship)
```

### 多对多关系

`automap` 将生成多对多关系，例如那些包含 `secondary` 参数的关系。生成这些关系的过程如下：

1.  在为其分配任何映射类之前，将检查给定的 `Table` 是否包含 `ForeignKeyConstraint` 对象。

1.  如果表包含两个并且仅有两个 `ForeignKeyConstraint` 对象，并且此表中的所有列都是这两个 `ForeignKeyConstraint` 对象的成员，则假定该表是一个“次要”表，并且**不会直接映射**。

1.  `Table` 所引用的两个（或一个，用于自引用）外部表将与它们将被映射到的类匹配，如果有的话。

1.  如果两侧的映射类位于同一处，则在两个类之间创建一个双向的多对多 `relationship()` / `backref()` 对。

1.  对于多对多的覆盖逻辑与一对多/多对一的逻辑相同；调用 `generate_relationship()` 函数来生成结构，并将保留现有属性。

### 继承关系

`automap` 将不会在处于继承关系的两个类之间生成任何关系。也就是说，对于以下两个给定的类：

```py
class Employee(Base):
    __tablename__ = "employee"
    id = Column(Integer, primary_key=True)
    type = Column(String(50))
    __mapper_args__ = {
        "polymorphic_identity": "employee",
        "polymorphic_on": type,
    }

class Engineer(Employee):
    __tablename__ = "engineer"
    id = Column(Integer, ForeignKey("employee.id"), primary_key=True)
    __mapper_args__ = {
        "polymorphic_identity": "engineer",
    }
```

从 `Engineer` 到 `Employee` 的外键不是用于关系，而是用于在两个类之间建立联合继承。

请注意，这意味着 automap 不会为从子类到超类的外键生成*任何*关系。如果映射实际上还有从子类到超类的关系，那么这些关系需要是显式的。在下面的例子中，由于从 `Engineer` 到 `Employee` 有两个单独的外键，我们需要设置我们想要的关系以及 `inherit_condition`，因为这些是 SQLAlchemy 无法猜测的事情：

```py
class Employee(Base):
    __tablename__ = "employee"
    id = Column(Integer, primary_key=True)
    type = Column(String(50))

    __mapper_args__ = {
        "polymorphic_identity": "employee",
        "polymorphic_on": type,
    }

class Engineer(Employee):
    __tablename__ = "engineer"
    id = Column(Integer, ForeignKey("employee.id"), primary_key=True)
    favorite_employee_id = Column(Integer, ForeignKey("employee.id"))

    favorite_employee = relationship(Employee, foreign_keys=favorite_employee_id)

    __mapper_args__ = {
        "polymorphic_identity": "engineer",
        "inherit_condition": id == Employee.id,
    }
```

### 处理简单的命名冲突

在映射过程中出现命名冲突的情况下，根据需要覆盖任何 `classname_for_table()`、`name_for_scalar_relationship()` 和 `name_for_collection_relationship()`。例如，如果 automap 尝试将一个多对一关系命名为现有列的名称，可以有条件地选择替代约定。给定一个模式：

```py
CREATE  TABLE  table_a  (
  id  INTEGER  PRIMARY  KEY
);

CREATE  TABLE  table_b  (
  id  INTEGER  PRIMARY  KEY,
  table_a  INTEGER,
  FOREIGN  KEY(table_a)  REFERENCES  table_a(id)
);
```

上述模式将首先将 `table_a` 表自动映射为名为 `table_a` 的类；然后将在 `table_b` 类上自动映射一个与此相关类相同名称的关系，例如 `table_a`。这个关系名称与映射列 `table_b.table_a` 冲突，并且在映射时会发出错误。

通过使用下划线，我们可以解决这个冲突：

```py
def name_for_scalar_relationship(base, local_cls, referred_cls, constraint):
    name = referred_cls.__name__.lower()
    local_table = local_cls.__table__
    if name in local_table.columns:
        newname = name + "_"
        warnings.warn("Already detected name %s present.  using %s" % (name, newname))
        return newname
    return name

Base.prepare(
    autoload_with=engine,
    name_for_scalar_relationship=name_for_scalar_relationship,
)
```

或者，我们可以在列的一侧更改名称。可以使用在 显式命名声明性映射列 中描述的技术修改映射的列，通过将列显式分配给一个新名称：

```py
Base = automap_base()

class TableB(Base):
    __tablename__ = "table_b"
    _table_a = Column("table_a", ForeignKey("table_a.id"))

Base.prepare(autoload_with=engine)
```

## 使用具有显式声明的 Automap

正如之前所指出的，automap 不依赖于反射，并且可以利用 `Table` 对象集合中的任何对象在 `MetaData` 集合中。由此可见，automap 也可以用于生成缺失的关系，只要有一个完全定义了表元数据的完整模型：

```py
from sqlalchemy.ext.automap import automap_base
from sqlalchemy import Column, Integer, String, ForeignKey

Base = automap_base()

class User(Base):
    __tablename__ = "user"

    id = Column(Integer, primary_key=True)
    name = Column(String)

class Address(Base):
    __tablename__ = "address"

    id = Column(Integer, primary_key=True)
    email = Column(String)
    user_id = Column(ForeignKey("user.id"))

# produce relationships
Base.prepare()

# mapping is complete, with "address_collection" and
# "user" relationships
a1 = Address(email="u1")
a2 = Address(email="u2")
u1 = User(address_collection=[a1, a2])
assert a1.user is u1
```

在上面的例子中，给定了大部分完整的 `User` 和 `Address` 映射，我们在 `Address.user_id` 上定义的 `ForeignKey` 允许在映射类上生成一个双向关系对 `Address.user` 和 `User.address_collection`。

请注意，当子类化`AutomapBase`时，需要调用`AutomapBase.prepare()`方法；如果未调用，则我们声明的类处于未映射状态。

## 拦截列定义

`MetaData`和`Table`对象支持一个事件钩子`DDLEvents.column_reflect()`，可用于在构建`Column`对象之前拦截有关数据库列的反射信息。例如，如果我们想要使用命名约定来映射列，例如`"attr_<columnname>"`，则可以应用该事件如下：

```py
@event.listens_for(Base.metadata, "column_reflect")
def column_reflect(inspector, table, column_info):
    # set column.key = "attr_<lower_case_name>"
    column_info["key"] = "attr_%s" % column_info["name"].lower()

# run reflection
Base.prepare(autoload_with=engine)
```

版本 1.4.0b2 中的新内容：`DDLEvents.column_reflect()`事件可以应用于一个`MetaData`对象。

另请参阅

`DDLEvents.column_reflect()`

从反射表自动命名方案 - 在 ORM 映射文档中

## API 参考

| 对象名称 | 描述 |
| --- | --- |
| automap_base([declarative_base], **kw) | 生成一个声明式自动映射基类。 |
| AutomapBase | 用于“自动映射”模式的基类。 |
| classname_for_table(base, tablename, table) | 给定表名，返回应该使用的类名。 |
| generate_relationship(base, direction, return_fn, attrname, ..., **kw) | 代表两个映射类生成一个`relationship()`或者`backref()`。 |
| name_for_collection_relationship(base, local_cls, referred_cls, constraint) | 返回用于从一个类引用另一个类的属性名称，用于集合引用。 |
| name_for_scalar_relationship(base, local_cls, referred_cls, constraint) | 返回用于从一个类引用另一个类的属性名称，用于标量对象引用。 |

```py
function sqlalchemy.ext.automap.automap_base(declarative_base: Type[Any] | None = None, **kw: Any) → Any
```

生成一个声明式自动映射基类。

此函数生成一个新的基类，该基类是由 `AutomapBase` 类和由 `declarative_base()` 产生的声明性基类的产品。

除了 `declarative_base` 外的所有参数都是直接传递给 `declarative_base()` 函数的关键字参数。

参数：

+   `declarative_base` – 由 `declarative_base()` 产生的现有类。当传递此参数时，函数不再调用 `declarative_base()` 自身，并且所有其他关键字参数都将被忽略。

+   `**kw` – 关键字参数会传递给 `declarative_base()`。

```py
class sqlalchemy.ext.automap.AutomapBase
```

用于“automap”模式的基类。

`AutomapBase` 类可以与由 `declarative_base()` 函数产生的“声明性基类”类进行比较。在实践中，`AutomapBase` 类始终与实际的声明性基类一起使用作为混入。

一个新的可子类化的 `AutomapBase` 通常是使用 `automap_base()` 函数实例化的。

**成员**

by_module, classes, metadata, prepare()

另请参阅

Automap

```py
attribute by_module: ClassVar[ByModuleProperties]
```

包含点分隔的模块名称的层次结构，链接到类的 `Properties` 实例。

这个集合是 `AutomapBase.classes` 集合的一种替代方法，当利用 `AutomapBase.prepare.modulename_for_table` 参数时，该参数将为生成的类应用不同的 `__module__` 属性。

automap 生成类的默认 `__module__` 是 `sqlalchemy.ext.automap`；要使用 `AutomapBase.by_module` 访问此命名空间，看起来像这样：

```py
User = Base.by_module.sqlalchemy.ext.automap.User
```

如果一个类的 `__module__` 是 `mymodule.account`，访问此命名空间看起来像这样：

```py
MyClass = Base.by_module.mymodule.account.MyClass
```

2.0 版中的新功能。

另请参阅

从多个模式生成映射

```py
attribute classes: ClassVar[Properties[Type[Any]]]
```

包含类的 `Properties` 实例。

此对象的行为类似于表上的 `.c` 集合。类以其给定的名称存在，例如：

```py
Base = automap_base()
Base.prepare(autoload_with=some_engine)

User, Address = Base.classes.User, Base.classes.Address
```

对于与 `Properties` 方法名重叠的类名，例如 `items()`，也支持使用 getitem 形式：

```py
Item = Base.classes["items"]
```

```py
attribute metadata: ClassVar[MetaData]
```

指的是将用于新 `Table` 对象的 `MetaData` 集合。

另请参见

访问表和元数据

```py
classmethod prepare(autoload_with: Engine | None = None, engine: Any | None = None, reflect: bool = False, schema: str | None = None, classname_for_table: PythonNameForTableType | None = None, modulename_for_table: PythonNameForTableType | None = None, collection_class: Any | None = None, name_for_scalar_relationship: NameForScalarRelationshipType | None = None, name_for_collection_relationship: NameForCollectionRelationshipType | None = None, generate_relationship: GenerateRelationshipType | None = None, reflection_options: Dict[_KT, _VT] | immutabledict[_KT, _VT] = {}) → None
```

从 `MetaData` 中提取映射类和关系，并执行映射。

有关完整文档和示例，请参见 基本使用。

参数：

+   `autoload_with` – 使用与其执行模式反射的 `Engine` 或 `Connection`；当指定时，`MetaData.reflect()` 方法将在此方法的范围内调用。

+   `engine` –

    已弃用；使用 `AutomapBase.autoload_with`。用于指示在反映表时使用的 `Engine` 或 `Connection`，如果 `AutomapBase.reflect` 为 True。

    自版本 1.4 起已弃用：`AutomapBase.prepare.engine` 参数已弃用，并将在未来版本中删除。请使用 `AutomapBase.prepare.autoload_with` 参数。

+   `reflect` –

    已弃用；使用 `AutomapBase.autoload_with`。指示是否应调用 `MetaData.reflect()`。

    自版本 1.4 起已弃用：`AutomapBase.prepare.reflect` 参数已弃用，并将在未来版本中删除。当传递了 `AutomapBase.prepare.autoload_with` 时启用反射。

+   `classname_for_table` – 一个可调用的函数，将根据表名生成新类名。默认为 `classname_for_table()`。

+   `modulename_for_table` –

    可调用函数，用于为内部生成的类生成有效的`__module__`，以允许在单个自动映射基类中具有相同名称的多个类，这些类将位于不同的“模块”中。

    默认为`None`，表示`__module__`不会被显式设置；Python 运行时将为这些类使用值`sqlalchemy.ext.automap`。

    在为生成的类分配`__module__`时，可以基于点分隔的模块名称使用`AutomapBase.by_module`集合访问它们。使用此钩子分配了显式`__module_`的类**不会**放入`AutomapBase.classes`集合中，而只会放入`AutomapBase.by_module`中。

    2.0 版中的新功能。

    另请参见

    从多个模式生成映射

+   `name_for_scalar_relationship` – 可调用函数，用于为标量关系生成关系名称。默认为`name_for_scalar_relationship()`。

+   `name_for_collection_relationship` – 可调用函数，用于为面向集合的关系生成关系名称。默认为`name_for_collection_relationship()`。

+   `generate_relationship` – 可调用函数，用于实际生成`relationship()`和`backref()`构造。默认为`generate_relationship()`。

+   `collection_class` – 当创建代表集合的新`relationship()`对象时将使用的 Python 集合类。默认为`list`。

+   `schema` –

    反映表时要反映的模式名称，使用`AutomapBase.prepare.autoload_with`参数。该名称传递给`MetaData.reflect()`的`MetaData.reflect.schema`参数。当省略时，将使用数据库连接使用的默认模式。

    注意

    `AutomapBase.prepare.schema` 参数支持一次反射单个模式。要包含来自多个模式的表，请多次调用 `AutomapBase.prepare()`。

    有关多模式自动映射的概述，包括使用附加命名约定解决表名冲突的方法，请参阅从多个模式生成映射 部分。

    新版本 2.0 中：`AutomapBase.prepare()` 可以直接调用任意次数，并跟踪已处理的表，以避免再次处理它们。

+   `reflection_options` –

    当存在时，此选项字典将传递给 `MetaData.reflect()` 以提供通用的反射特定选项，如 `only` 和/或特定于方言的选项，如 `oracle_resolve_synonyms`。

    新版本 1.4 中。

```py
function sqlalchemy.ext.automap.classname_for_table(base: Type[Any], tablename: str, table: Table) → str
```

返回应使用的类名，给定表的名称。

默认实现为：

```py
return str(tablename)
```

可以使用 `AutomapBase.prepare.classname_for_table` 参数指定替代实现。

参数：

+   `base` – 进行准备的 `AutomapBase` 类。

+   `tablename` – `Table` 的字符串名称。

+   `table` – `Table` 对象本身。

返回：

一个字符串类名。

注意

在 Python 2 中，用于类名的字符串 **必须** 是非 Unicode 对象，例如 `str()` 对象。`Table` 的 `.name` 属性通常是 Python unicode 子类，因此应在考虑任何非 ASCII 字符后，应用 `str()` 函数到此名称。

```py
function sqlalchemy.ext.automap.name_for_scalar_relationship(base: Type[Any], local_cls: Type[Any], referred_cls: Type[Any], constraint: ForeignKeyConstraint) → str
```

返回应用于从一个类到另一个类的引用的属性名称，用于标量对象引用。

默认实现为：

```py
return referred_cls.__name__.lower()
```

可以使用 `AutomapBase.prepare.name_for_scalar_relationship` 参数指定替代实现。

参数：

+   `base` – 进行准备的 `AutomapBase` 类。

+   `local_cls` – 映射到本地方的类。

+   `referred_cls` – 映射到引用方的类。

+   `constraint` – 正在检查以生成此关系的`ForeignKeyConstraint`。

```py
function sqlalchemy.ext.automap.name_for_collection_relationship(base: Type[Any], local_cls: Type[Any], referred_cls: Type[Any], constraint: ForeignKeyConstraint) → str
```

返回应用于从一个类到另一个类的引用的属性名称，用于集合引用。

默认实现如下：

```py
return referred_cls.__name__.lower() + "_collection"
```

可以使用`AutomapBase.prepare.name_for_collection_relationship`参数指定替代实现。

参数：

+   `base` – 执行准备工作的`AutomapBase`类。

+   `local_cls` – 要映射到本地方的类。

+   `referred_cls` – 要映射到引用方的类。

+   `constraint` – 正在检查以生成此关系的`ForeignKeyConstraint`。

```py
function sqlalchemy.ext.automap.generate_relationship(base: Type[Any], direction: RelationshipDirection, return_fn: Callable[..., Relationship[Any]] | Callable[..., ORMBackrefArgument], attrname: str, local_cls: Type[Any], referred_cls: Type[Any], **kw: Any) → Relationship[Any] | ORMBackrefArgument
```

代表两个映射类生成一个`relationship()`或`backref()`。

可以使用`AutomapBase.prepare.generate_relationship`参数指定此函数的替代实现。

此函数的默认实现如下：

```py
if return_fn is backref:
    return return_fn(attrname, **kw)
elif return_fn is relationship:
    return return_fn(referred_cls, **kw)
else:
    raise TypeError("Unknown relationship function: %s" % return_fn)
```

参数：

+   `base` – 执行准备工作的`AutomapBase`类。

+   `direction` – 指示关系的“方向”; 这将是`ONETOMANY`、`MANYTOONE`、`MANYTOMANY`之一。

+   `return_fn` – 默认用于创建关系的函数。这将是`relationship()`或`backref()`之一。`backref()`函数的结果将用于在第二步生成新的`relationship()`，因此如果使用自定义关系函数，则用户定义的实现必须正确区分这两个函数。

+   `attrname` – 正在分配此关系的属性名称。如果`generate_relationship.return_fn`的值是`backref()`函数，则此名称是分配给反向引用的名称。

+   `local_cls` – 此关系或反向引用将在本地存在的“本地”类。

+   `referred_cls` – 此关系或反向引用所指向的“引用”类。

+   `**kw` – 所有附加的关键字参数都将传递给该函数。

返回值：

`relationship()` 或 `backref()` 构造，由 `generate_relationship.return_fn` 参数所指定。
