# 声明式扩展

> 原文：[`docs.sqlalchemy.org/en/20/orm/extensions/declarative/index.html`](https://docs.sqlalchemy.org/en/20/orm/extensions/declarative/index.html)

声明式映射 API 特定的扩展。

1.4 版本更改：绝大部分声明式扩展现在已整合到 SQLAlchemy ORM 中，并可从 `sqlalchemy.orm` 命名空间导入。请参阅声明式映射的文档以获取新文档。有关更改的概述，请参阅声明式现已与 ORM 整合，并带有新功能。

| 对象名称 | 描述 |
| --- | --- |
| AbstractConcreteBase | 一个用于“具体”声明式映射的辅助类。 |
| ConcreteBase | 一个用于“具体”声明式映射的辅助类。 |
| DeferredReflection | 一个用于基于延迟反射步骤构建映射的辅助类。 |

```py
class sqlalchemy.ext.declarative.AbstractConcreteBase
```

一个用于“具体”声明式映射的辅助类。

`AbstractConcreteBase` 将自动使用 `polymorphic_union()` 函数，对所有作为此类的子类映射的表执行。该函数通过 `__declare_first__()` 函数调用，这实际上是一个 `before_configured()` 事件的钩子。

`AbstractConcreteBase` 应用 `Mapper` 到其直接继承的类，就像对任何其他声明式映射的类一样。然而，`Mapper` 没有映射到任何特定的 `Table` 对象。相反，它直接映射到由 `polymorphic_union()` 产生的“多态”可选择的对象，并且不执行自己的持久化操作。与 `ConcreteBase` 相比，后者将其直接继承的类映射到直接存储行的实际 `Table`。

注意

`AbstractConcreteBase`延迟了基类的映射器创建，直到所有子类都已定义，因为它需要创建一个针对包含所有子类表的可选择项的映射。为了实现这一点，它等待**映射器配置事件**发生，然后扫描所有配置的子类，并设置一个将一次性查询所有子类的映射。

虽然此事件通常会自动调用，但在`AbstractConcreteBase`的情况下，如果第一个操作是针对此基类的查询，则可能需要在定义所有子类映射之后显式调用它。为此，一旦所有期望的类都已配置，可以调用正在使用的`registry`上的`registry.configure()`方法，该方法可在特定声明基类的关系中使用：

```py
Base.registry.configure()
```

示例：

```py
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.ext.declarative import AbstractConcreteBase

class Base(DeclarativeBase):
    pass

class Employee(AbstractConcreteBase, Base):
    pass

class Manager(Employee):
    __tablename__ = 'manager'
    employee_id = Column(Integer, primary_key=True)
    name = Column(String(50))
    manager_data = Column(String(40))

    __mapper_args__ = {
        'polymorphic_identity':'manager',
        'concrete':True
    }

Base.registry.configure()
```

抽象基类在声明时以一种特殊的方式处理；在类配置时，它的行为类似于声明式的混入或`__abstract__`基类。一旦类被配置并生成映射，它会被映射自身，但在其所有子类之后。这是在任何其他 SQLAlchemy API 功能中都找不到的非常独特的映射系统。

使用这种方法，我们可以指定将在映射的子类上发生的列和属性，就像我们通常在 Mixin 和自定义基类中所做的那样：

```py
from sqlalchemy.ext.declarative import AbstractConcreteBase

class Company(Base):
    __tablename__ = 'company'
    id = Column(Integer, primary_key=True)

class Employee(AbstractConcreteBase, Base):
    strict_attrs = True

    employee_id = Column(Integer, primary_key=True)

    @declared_attr
    def company_id(cls):
        return Column(ForeignKey('company.id'))

    @declared_attr
    def company(cls):
        return relationship("Company")

class Manager(Employee):
    __tablename__ = 'manager'

    name = Column(String(50))
    manager_data = Column(String(40))

    __mapper_args__ = {
        'polymorphic_identity':'manager',
        'concrete':True
    }

Base.registry.configure()
```

然而，当我们使用我们的映射时，`Manager`和`Employee`都将拥有一个可独立使用的`.company`属性：

```py
session.execute(
    select(Employee).filter(Employee.company.has(id=5))
)
```

参数：

**strict_attrs** –

当在基类上指定时，“严格”属性模式被启用，试图将基类上的 ORM 映射属性限制为仅当下立即存在的属性，同时仍保留“多态”加载行为。

2.0 版中新增。

另请参阅

`ConcreteBase`

具体表继承

抽象具体类

**类签名**

类`sqlalchemy.ext.declarative.AbstractConcreteBase` (`sqlalchemy.ext.declarative.extensions.ConcreteBase`)

```py
class sqlalchemy.ext.declarative.ConcreteBase
```

用于‘具体’声明映射的辅助类。

`ConcreteBase` 会自动使用 `polymorphic_union()` 函数，针对所有映射为该类的子类的表。该函数通过 `__declare_last__()` 函数调用，这实质上是 `after_configured()` 事件的钩子。

`ConcreteBase` 为类本身生成一个映射表。与 `AbstractConcreteBase` 相比，后者不会。

示例：

```py
from sqlalchemy.ext.declarative import ConcreteBase

class Employee(ConcreteBase, Base):
    __tablename__ = 'employee'
    employee_id = Column(Integer, primary_key=True)
    name = Column(String(50))
    __mapper_args__ = {
                    'polymorphic_identity':'employee',
                    'concrete':True}

class Manager(Employee):
    __tablename__ = 'manager'
    employee_id = Column(Integer, primary_key=True)
    name = Column(String(50))
    manager_data = Column(String(40))
    __mapper_args__ = {
                    'polymorphic_identity':'manager',
                    'concrete':True}
```

`polymorphic_union()` 使用的鉴别器列的默认名称为 `type`。为了适应映射的用例，其中映射表中的实际列已命名为 `type`，可以通过设置 `_concrete_discriminator_name` 属性来配置鉴别器名称：

```py
class Employee(ConcreteBase, Base):
    _concrete_discriminator_name = '_concrete_discriminator'
```

自版本 1.3.19 中新增：为 `ConcreteBase` 添加了 `_concrete_discriminator_name` 属性，以便自定义虚拟鉴别器列名称。

自版本 1.4.2 中更改：只需将 `_concrete_discriminator_name` 属性放置在最基类上即可使所有子类正确生效。如果映射列名称与鉴别器名称冲突，则现在会显示显式错误消息，而在 1.3.x 系列中会有一些警告，然后生成一个无用的查询。

另请参阅

`AbstractConcreteBase`

具体表继承

```py
class sqlalchemy.ext.declarative.DeferredReflection
```

一个用于基于延迟反射步骤构建映射的辅助类。

通常情况下，通过将一个 `Table` 对象设置为具有 autoload_with=engine 的 `__table__` 属性，可以使用反射来使用声明。一个声明性类。需要注意的是，在构建普通声明性映射的时候，`Table` 必须是完全反映的，或者至少有一个主键列，这意味着在类声明时必须可用 `Engine`。

`DeferredReflection` mixin 将映射器的构建移动到稍后的时间点，在调用首先反射到目前为止创建的所有 `Table` 对象的特定方法之后。类可以定义如下：

```py
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.ext.declarative import DeferredReflection
Base = declarative_base()

class MyClass(DeferredReflection, Base):
    __tablename__ = 'mytable'
```

在上面，`MyClass` 还没有映射。在上述方式定义了一系列类之后，可以使用 `prepare()` 反射所有表并创建映射：

```py
engine = create_engine("someengine://...")
DeferredReflection.prepare(engine)
```

`DeferredReflection` mixin 可以应用于单个类，用作声明基类本身，或用于自定义抽象类。使用抽象基类允许仅为特定准备步骤准备一部分类，这对于使用多个引擎的应用程序是必要的。例如，如果一个应用程序有两个引擎，您可能会使用两个基类，并分别准备每个基类，例如：

```py
class ReflectedOne(DeferredReflection, Base):
    __abstract__ = True

class ReflectedTwo(DeferredReflection, Base):
    __abstract__ = True

class MyClass(ReflectedOne):
    __tablename__ = 'mytable'

class MyOtherClass(ReflectedOne):
    __tablename__ = 'myothertable'

class YetAnotherClass(ReflectedTwo):
    __tablename__ = 'yetanothertable'

# ... etc.
```

在上面，`ReflectedOne` 和 `ReflectedTwo` 的类层次结构可以分别配置：

```py
ReflectedOne.prepare(engine_one)
ReflectedTwo.prepare(engine_two)
```

**成员**

prepare()

另请参阅

使用 DeferredReflection - 在 使用声明式配置表 部分。

```py
classmethod prepare(bind: Engine | Connection, **reflect_kw: Any) → None
```

反射所有当前 `DeferredReflection` 子类的所有 `Table` 对象

参数：

+   `bind` –

    `Engine` 或 `Connection` 实例

    ..versionchanged:: 2.0.16 现在也接受 `Connection`。

+   `**reflect_kw` –

    传递给 `MetaData.reflect()` 的其他关键字参数，例如 `MetaData.reflect.views`。

    新版本 2.0.16 中的内容。
