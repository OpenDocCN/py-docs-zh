# 映射类继承层次结构

> 原文：[`docs.sqlalchemy.org/en/20/orm/inheritance.html`](https://docs.sqlalchemy.org/en/20/orm/inheritance.html)

SQLAlchemy 支持三种继承形式：

+   **单表继承** – 几种类别的类别由单个表表示；

+   **具体表继承** – 每种类别的类别都由独立的表表示；

+   **联接表继承** – 类层次结构在依赖表之间分解。每个类由其自己的表表示，该表仅包含该类本地的属性。

最常见的继承形式是单一和联接表，而具体继承则提出了更多的配置挑战。

当映射器配置在继承关系中时，SQLAlchemy 有能力以多态方式加载元素，这意味着单个查询可以返回多种类型的对象。

另请参见

为继承映射编写 SELECT 语句 - 在 ORM 查询指南 中

继承映射示例 - 联接、单一和具体继承的完整示例

## 联接表继承

在联接表继承中，沿着类层次结构的每个类都由一个不同的表表示。对类层次结构中特定子类的查询将作为 SQL JOIN 在其继承路径上的所有表之间进行。如果查询的类是基类，则查询基表，同时可以选择包含其他表或允许后续加载特定于子表的属性的选项。

在所有情况下，对于给定行要实例化的最终类由基类上定义的鉴别器列或 SQL 表达式确定，该列将生成与特定子类关联的标量值。

联接继承层次结构中的基类将配置具有指示多态鉴别器列以及可选地为基类本身配置的多态标识符的其他参数：

```py
from sqlalchemy import ForeignKey
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column

class Base(DeclarativeBase):
    pass

class Employee(Base):
    __tablename__ = "employee"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    type: Mapped[str]

    __mapper_args__ = {
        "polymorphic_identity": "employee",
        "polymorphic_on": "type",
    }

    def __repr__(self):
        return f"{self.__class__.__name__}({self.name!r})"
```

在上面的示例中，鉴别器是 `type` 列，可以使用 `Mapper.polymorphic_on` 参数进行配置。该参数接受一个面向列的表达式，可以指定为要使用的映射属性的字符串名称，也可以指定为列表达式对象，如 `Column` 或 `mapped_column()` 构造。

鉴别器列将存储指示行内表示的对象类型的值。该列可以是任何数据类型，但字符串和整数是最常见的。要为数据库中的特定行应用到该列的实际数据值是使用下面描述的 `Mapper.polymorphic_identity` 参数指定的。

尽管多态鉴别器表达式不是严格必需的，但如果需要多态加载，则需要它。在基础表上建立列是实现这一点的最简单方法，然而非常复杂的继承映射可能会使用 SQL 表达式，例如 CASE 表达式，作为多态鉴别器。

注意

目前，**整个继承层次结构只能配置一个鉴别器列或 SQL 表达式**，通常在层次结构中最基本的类上。暂时不支持“级联”多态鉴别器表达式。

我们接下来定义 `Engineer` 和 `Manager` 的 `Employee` 子类。每个类包含代表其所代表的子类的唯一属性的列。每个表还必须包含一个主键列（或列），以及对父表的外键引用：

```py
class Engineer(Employee):
    __tablename__ = "engineer"
    id: Mapped[int] = mapped_column(ForeignKey("employee.id"), primary_key=True)
    engineer_name: Mapped[str]

    __mapper_args__ = {
        "polymorphic_identity": "engineer",
    }

class Manager(Employee):
    __tablename__ = "manager"
    id: Mapped[int] = mapped_column(ForeignKey("employee.id"), primary_key=True)
    manager_name: Mapped[str]

    __mapper_args__ = {
        "polymorphic_identity": "manager",
    }
```

在上面的示例中，每个映射都在其映射器参数中指定了`Mapper.polymorphic_identity`参数。此值填充了基础映射器上建立的`Mapper.polymorphic_on`参数指定的列。每个映射类的`Mapper.polymorphic_identity`参数应在整个层次结构中是唯一的，并且每个映射类应只有一个“标识”；如上所述，“级联”标识不支持一些子类引入第二个标识的情况。

ORM 使用由`Mapper.polymorphic_identity`设置的值来确定加载行的多态时行属于哪个类。在上面的示例中，每个代表`Employee`的行将在其`type`列中具有值`'employee'`；类似地，每个`Engineer`将获得值`'engineer'`，每个`Manager`将获得值`'manager'`。无论继承映射是否为子类使用不同的连接表（如连接表继承）或所有一个表（如单表继承），这个值都应该被持久化并在查询时对 ORM 可用。`Mapper.polymorphic_identity`参数也适用于具体表继承，但实际上并没有持久化；有关详细信息，请参阅后面的具体表继承部分。

在多态设置中，最常见的是外键约束建立在与主键本身相同的列或列上，但这并非必需；也可以使与主键不同的列引用到父级的外键。从基表到子类的 JOIN 的构建方式也是可直接自定义的，但这很少是必要的。

使用连接继承映射完成后，针对`Employee`的查询将返回`Employee`、`Engineer`和`Manager`对象的组合。新保存的`Engineer`、`Manager`和`Employee`对象在这种情况下将自动填充`employee.type`列中的正确“识别器”值，如`"engineer"`、`"manager"`或`"employee"`。

### 使用连接继承的关系

使用连接表继承完全支持关系。涉及连接继承类的关系应该针对在层次结构中也对应于外键约束的类；在下面的示例中，由于`employee`表有一个回到`company`表的外键约束，因此关系被设置在`Company`和`Employee`之间：

```py
from __future__ import annotations

from sqlalchemy.orm import relationship

class Company(Base):
    __tablename__ = "company"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    employees: Mapped[List[Employee]] = relationship(back_populates="company")

class Employee(Base):
    __tablename__ = "employee"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    type: Mapped[str]
    company_id: Mapped[int] = mapped_column(ForeignKey("company.id"))
    company: Mapped[Company] = relationship(back_populates="employees")

    __mapper_args__ = {
        "polymorphic_identity": "employee",
        "polymorphic_on": "type",
    }

class Manager(Employee): ...

class Engineer(Employee): ...
```

如果外键约束在对应于子类的表上，关系应该指向该子类。在下面的示例中，有一个从`manager`到`company`的外键约束，因此关系建立在`Manager`和`Company`类之间：

```py
class Company(Base):
    __tablename__ = "company"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    managers: Mapped[List[Manager]] = relationship(back_populates="company")

class Employee(Base):
    __tablename__ = "employee"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    type: Mapped[str]

    __mapper_args__ = {
        "polymorphic_identity": "employee",
        "polymorphic_on": "type",
    }

class Manager(Employee):
    __tablename__ = "manager"
    id: Mapped[int] = mapped_column(ForeignKey("employee.id"), primary_key=True)
    manager_name: Mapped[str]

    company_id: Mapped[int] = mapped_column(ForeignKey("company.id"))
    company: Mapped[Company] = relationship(back_populates="managers")

    __mapper_args__ = {
        "polymorphic_identity": "manager",
    }

class Engineer(Employee): ...
```

在上面的例子中，`Manager`类将有一个`Manager.company`属性；`Company`将有一个`Company.managers`属性，总是对`employee`和`manager`表一起加载。

### 加载连接继承映射

请参阅编写继承映射的 SELECT 语句部分，了解关于继承加载技术的背景，包括在映射器配置时间和查询时间配置要查询的表。## 单表继承

单表继承在单个表中表示所有子类的所有属性。具有唯一于该类的属性的特定子类将在表中的列中保留它们，如果行引用了不同类型的对象，则这些列将为空。

在层次结构中查询特定子类将呈现为针对基表的 SELECT 查询，其中将包括一个 WHERE 子句，该子句限制行为具有鉴别器列或表达式中存在的特定值或值的行。

单表继承相对于联接表继承具有简单性的优势；查询要高效得多，因为只需要涉及一个表来加载每个表示类的对象。

单表继承配置看起来很像联接表继承，除了只有基类指定了`__tablename__`。还需要在基表上有一个鉴别器列，以便类可以彼此区分。

即使子类共享所有属性的基表，在使用声明性时，仍然可以在子类上指定`mapped_column`对象，指示该列仅映射到该子类；`mapped_column`将应用于相同的基本`Table`对象：

```py
class Employee(Base):
    __tablename__ = "employee"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    type: Mapped[str]

    __mapper_args__ = {
        "polymorphic_on": "type",
        "polymorphic_identity": "employee",
    }

class Manager(Employee):
    manager_data: Mapped[str] = mapped_column(nullable=True)

    __mapper_args__ = {
        "polymorphic_identity": "manager",
    }

class Engineer(Employee):
    engineer_info: Mapped[str] = mapped_column(nullable=True)

    __mapper_args__ = {
        "polymorphic_identity": "engineer",
    }
```

请注意，派生类 Manager 和 Engineer 的映射器省略了`__tablename__`，这表明它们没有自己的映射表。另外，包含了一个带有`nullable=True`的`mapped_column()`指令；由于为这些类声明的 Python 类型不包括`Optional[]`，因此该列通常被映射为`NOT NULL`，这对于该列只期望被填充为那些对应于该特定子类的行并不合适。

### 使用`use_existing_column`解决列冲突

请注意，在上一节中，`manager_name`和`engineer_info`列被“上移”以应用于`Employee.__table__`，因为它们在没有自己的表的子类上声明。当两个子类想要指定*相同*的列时，就会出现一个棘手的情况，如下所示：

```py
from datetime import datetime

class Employee(Base):
    __tablename__ = "employee"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    type: Mapped[str]

    __mapper_args__ = {
        "polymorphic_on": "type",
        "polymorphic_identity": "employee",
    }

class Engineer(Employee):
    __mapper_args__ = {
        "polymorphic_identity": "engineer",
    }
    start_date: Mapped[datetime] = mapped_column(nullable=True)

class Manager(Employee):
    __mapper_args__ = {
        "polymorphic_identity": "manager",
    }
    start_date: Mapped[datetime] = mapped_column(nullable=True)
```

上面，在`Engineer`和`Manager`上声明的`start_date`列将导致错误：

```py
sqlalchemy.exc.ArgumentError: Column 'start_date' on class Manager conflicts
with existing column 'employee.start_date'.  If using Declarative,
consider using the use_existing_column parameter of mapped_column() to
resolve conflicts.
```

上述场景对声明式映射系统存在一种模糊性，可以通过在`mapped_column()`上使用`mapped_column.use_existing_column`参数来解决，该参数指示`mapped_column()`在存在继承的超类时查找并使用已经映射的列，如果已经存在，则映射一个新列：

```py
from sqlalchemy import DateTime

class Employee(Base):
    __tablename__ = "employee"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    type: Mapped[str]

    __mapper_args__ = {
        "polymorphic_on": "type",
        "polymorphic_identity": "employee",
    }

class Engineer(Employee):
    __mapper_args__ = {
        "polymorphic_identity": "engineer",
    }

    start_date: Mapped[datetime] = mapped_column(
        nullable=True, use_existing_column=True
    )

class Manager(Employee):
    __mapper_args__ = {
        "polymorphic_identity": "manager",
    }

    start_date: Mapped[datetime] = mapped_column(
        nullable=True, use_existing_column=True
    )
```

上面的例子中，当 `Manager` 被映射时，`start_date` 列已经存在于 `Employee` 类中，已经由 `Engineer` 映射提供。`mapped_column.use_existing_column`参数指示 `mapped_column()` 应首先在 `Employee` 的映射 `Table` 中查找请求的 `Column`，如果存在，则保持该现有映射。如果不存在，`mapped_column()`将正常映射列，将其添加为由 `Employee` 超类引用的 `Table` 中的列之一。

版本 2.0.0b4 中新增：- 添加了`mapped_column.use_existing_column`，提供了一种在继承子类上有条件地映射列的 2.0 兼容方法。之前的方法结合了 `declared_attr` 和对父类 `.__table__` 的查找，仍然可以正常工作，但缺乏[**PEP 484**](https://peps.python.org/pep-0484/)的类型支持。

可以使用类似的概念来定义特定系列的列和/或其他可重复使用的混合类中的映射属性（请参阅使用混合类组合映射层次结构）：

```py
class Employee(Base):
    __tablename__ = "employee"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    type: Mapped[str]

    __mapper_args__ = {
        "polymorphic_on": type,
        "polymorphic_identity": "employee",
    }

class HasStartDate:
    start_date: Mapped[datetime] = mapped_column(
        nullable=True, use_existing_column=True
    )

class Engineer(HasStartDate, Employee):
    __mapper_args__ = {
        "polymorphic_identity": "engineer",
    }

class Manager(HasStartDate, Employee):
    __mapper_args__ = {
        "polymorphic_identity": "manager",
    }
```

### 单表继承关系

关系完全支持单表继承。配置方式与连接继承的方式相同；外键属性应该在关系的“外键”一侧的同一类上：

```py
class Company(Base):
    __tablename__ = "company"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    employees: Mapped[List[Employee]] = relationship(back_populates="company")

class Employee(Base):
    __tablename__ = "employee"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    type: Mapped[str]
    company_id: Mapped[int] = mapped_column(ForeignKey("company.id"))
    company: Mapped[Company] = relationship(back_populates="employees")

    __mapper_args__ = {
        "polymorphic_identity": "employee",
        "polymorphic_on": "type",
    }

class Manager(Employee):
    manager_data: Mapped[str] = mapped_column(nullable=True)

    __mapper_args__ = {
        "polymorphic_identity": "manager",
    }

class Engineer(Employee):
    engineer_info: Mapped[str] = mapped_column(nullable=True)

    __mapper_args__ = {
        "polymorphic_identity": "engineer",
    }
```

同样，类似于连接继承的情况，我们可以创建涉及特定子类的关系。在查询时，SELECT 语句将包括一个 WHERE 子句，将类选择限制为该子类或子类：

```py
class Company(Base):
    __tablename__ = "company"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    managers: Mapped[List[Manager]] = relationship(back_populates="company")

class Employee(Base):
    __tablename__ = "employee"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    type: Mapped[str]

    __mapper_args__ = {
        "polymorphic_identity": "employee",
        "polymorphic_on": "type",
    }

class Manager(Employee):
    manager_name: Mapped[str] = mapped_column(nullable=True)

    company_id: Mapped[int] = mapped_column(ForeignKey("company.id"))
    company: Mapped[Company] = relationship(back_populates="managers")

    __mapper_args__ = {
        "polymorphic_identity": "manager",
    }

class Engineer(Employee):
    engineer_info: Mapped[str] = mapped_column(nullable=True)

    __mapper_args__ = {
        "polymorphic_identity": "engineer",
    }
```

在上面的例子中，`Manager`类将具有`Manager.company`属性；`Company`将具有一个`Company.managers`属性，该属性始终针对`employee`加载，并附加一个 WHERE 子句，限制行为具有`type = 'manager'`的行。

### 使用`polymorphic_abstract`构建更深层次的层次结构

在 2.0 版本中新增。

在构建任何类型的继承层次结构时，映射类可以包括设置为`True`的`Mapper.polymorphic_abstract`参数，这表明该类应该被正常映射，但不希望直接实例化，并且不包括`Mapper.polymorphic_identity`。然后可以声明这个映射类的子类，这些子类本身可以包括一个`Mapper.polymorphic_identity`，因此可以正常使用。这允许一系列子类被一个通用的基类引用，该基类在层次结构中被视为“抽象”，在查询和`relationship()`声明中都是如此。这种用法与在 Declarative 中使用 __abstract__ 属性不同，后者将目标类完全取消映射，因此无法单独作为映射类使用。`Mapper.polymorphic_abstract`可以应用于层次结构中的任何类或类，包括同时应用于多个级别的类。

例如，假设`Manager`和`Principal`都被分类到一个超类`Executive`，而`Engineer`和`Sysadmin`被分类到一个超类`Technologist`。`Executive`和`Technologist`都不会被实例化，因此没有`Mapper.polymorphic_identity`。可以使用`Mapper.polymorphic_abstract`来配置这些类，如下所示：

```py
class Employee(Base):
    __tablename__ = "employee"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    type: Mapped[str]

    __mapper_args__ = {
        "polymorphic_identity": "employee",
        "polymorphic_on": "type",
    }

class Executive(Employee):
  """An executive of the company"""

    executive_background: Mapped[str] = mapped_column(nullable=True)

    __mapper_args__ = {"polymorphic_abstract": True}

class Technologist(Employee):
  """An employee who works with technology"""

    competencies: Mapped[str] = mapped_column(nullable=True)

    __mapper_args__ = {"polymorphic_abstract": True}

class Manager(Executive):
  """a manager"""

    __mapper_args__ = {"polymorphic_identity": "manager"}

class Principal(Executive):
  """a principal of the company"""

    __mapper_args__ = {"polymorphic_identity": "principal"}

class Engineer(Technologist):
  """an engineer"""

    __mapper_args__ = {"polymorphic_identity": "engineer"}

class SysAdmin(Technologist):
  """a systems administrator"""

    __mapper_args__ = {"polymorphic_identity": "sysadmin"}
```

在上面的示例中，新的类 `Technologist` 和 `Executive` 都是普通的映射类，并且还指示要添加到超类的新列 `executive_background` 和 `competencies`。然而，它们都缺少对 `Mapper.polymorphic_identity` 的设置；这是因为不希望直接实例化 `Technologist` 或 `Executive`；我们总是会有 `Manager`、`Principal`、`Engineer` 或 `SysAdmin` 中的一个。然而，我们可以查询 `Principal` 和 `Technologist` 角色，并且让它们成为`relationship()`的目标。下面的示例演示了对 `Technologist` 对象的 SELECT 语句：

```py
session.scalars(select(Technologist)).all()
SELECT  employee.id,  employee.name,  employee.type,  employee.competencies
FROM  employee
WHERE  employee.type  IN  (?,  ?)
[...]  ('engineer',  'sysadmin') 
```

`Technologist` 和 `Executive` 的抽象映射类也可以作为`relationship()`映射的目标，就像任何其他映射类一样。我们可以扩展上面的示例，包括 `Company`，其中包含单独的集合 `Company.technologists` 和 `Company.principals`：

```py
class Company(Base):
    __tablename__ = "company"
    id = Column(Integer, primary_key=True)

    executives: Mapped[List[Executive]] = relationship()
    technologists: Mapped[List[Technologist]] = relationship()

class Employee(Base):
    __tablename__ = "employee"
    id: Mapped[int] = mapped_column(primary_key=True)

    # foreign key to "company.id" is added
    company_id: Mapped[int] = mapped_column(ForeignKey("company.id"))

    # rest of mapping is the same
    name: Mapped[str]
    type: Mapped[str]

    __mapper_args__ = {
        "polymorphic_on": "type",
    }

# Executive, Technologist, Manager, Principal, Engineer, SysAdmin
# classes from previous example would follow here unchanged
```

使用上述映射，我们可以分别跨 `Company.technologists` 和 `Company.executives` 使用联接和关系加载技术：

```py
session.scalars(
    select(Company)
    .join(Company.technologists)
    .where(Technologist.competency.ilike("%java%"))
    .options(selectinload(Company.executives))
).all()
SELECT  company.id
FROM  company  JOIN  employee  ON  company.id  =  employee.company_id  AND  employee.type  IN  (?,  ?)
WHERE  lower(employee.competencies)  LIKE  lower(?)
[...]  ('engineer',  'sysadmin',  '%java%')

SELECT  employee.company_id  AS  employee_company_id,  employee.id  AS  employee_id,
employee.name  AS  employee_name,  employee.type  AS  employee_type,
employee.executive_background  AS  employee_executive_background
FROM  employee
WHERE  employee.company_id  IN  (?)  AND  employee.type  IN  (?,  ?)
[...]  (1,  'manager',  'principal') 
```

另请参阅

__abstract__ - 声明参数，允许在继承层次结构中完全取消映射声明的类，同时仍然继承自映射的超类。

### 加载单表继承映射

单表继承的加载技术与连接表继承的加载技术大部分相同，并且提供了这两种映射类型之间的高度抽象，因此很容易在它们之间切换，以及在单个继承层次结构中混合使用它们（只需从要单继承的子类中省略 `__tablename__`）。请参阅编写用于继承映射的 SELECT 语句和单表继承映射的 SELECT 语句章节，了解有关继承加载技术的文档，包括在映射器配置时间和查询时间配置要查询的类。## 具体表继承

具体继承将每个子类映射到其自己的独立表，每个表包含产生该类实例所需的所有列。具体继承配置默认以非多态方式查询；对于特定类的查询将仅查询该类的表，并且仅返回该类的实例。通过在映射器内配置特殊的 SELECT，通常会将所有表的 UNION 作为结果来启用具体类的多态加载。

警告

具体表继承比连接或单表继承**复杂得多**，在使用关系、急加载和多态加载方面**功能受限**，尤其是与其一起使用时。当以多态方式使用时，会生成**非常大的查询**，其中包含不会像简单连接那样执行得好的 UNION。强烈建议如果需要关系加载和多态加载的灵活性，尽量使用连接或单表继承。如果不需要多态加载，则每个类完全引用自己的表时可以使用普通的非继承映射。

相比于连接和单表继承在“多态”加载方面更为流畅，具体继承在这方面更为麻烦。因此，当**不需要多态加载**时，具体继承更为合适。建立涉及具体继承类的关系也更为麻烦。

要将类标记为使用具体继承，需要在`__mapper_args__`中添加`Mapper.concrete`参数。这表示对于声明式和映射来说，超类表不应被视为映射的一部分：

```py
class Employee(Base):
    __tablename__ = "employee"

    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))

class Manager(Employee):
    __tablename__ = "manager"

    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    manager_data = mapped_column(String(50))

    __mapper_args__ = {
        "concrete": True,
    }

class Engineer(Employee):
    __tablename__ = "engineer"

    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    engineer_info = mapped_column(String(50))

    __mapper_args__ = {
        "concrete": True,
    }
```

应该注意两个关键点：

+   我们必须在每个子类上**明确定义所有列**，即使是同名的列也是如此。例如此处的`Employee.name`列**不会**被复制到`Manager`或`Engineer`映射的表中。

+   虽然`Engineer`和`Manager`类与`Employee`之间有映射关系，但它们**不包括多态加载**。这意味着，如果我们查询`Employee`对象，`manager`和`engineer`表根本不会被查询。

### 具体多态加载配置

具体继承的多态加载要求针对应具有多态加载的每个基类配置一个专门的 SELECT。此 SELECT 需要能够单独访问所有映射的表，并且通常是使用 SQLAlchemy 助手`polymorphic_union()`构造的 UNION 语句。

如为继承映射编写 SELECT 语句中所讨论的，任何类型的映射继承配置都可以配置为默认从特殊可选中加载，使用`Mapper.with_polymorphic`参数。当前的公共 API 要求在首次构造`Mapper`时设置此参数。

但是，在使用 Declarative 的情况下，映射器和被映射的`Table`同时创建，一旦定义了映射的类。这意味着由于尚未定义对应于子类的`Table`对象，因此暂时无法提供`Mapper.with_polymorphic`参数。

有几种可用的策略来解决这个循环，但是 Declarative 提供了处理此问题的助手类`ConcreteBase`和`AbstractConcreteBase`。

使用`ConcreteBase`，我们可以几乎以与其他形式的继承映射相同的方式设置我们的具体映射：

```py
from sqlalchemy.ext.declarative import ConcreteBase
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    pass

class Employee(ConcreteBase, Base):
    __tablename__ = "employee"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))

    __mapper_args__ = {
        "polymorphic_identity": "employee",
        "concrete": True,
    }

class Manager(Employee):
    __tablename__ = "manager"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    manager_data = mapped_column(String(40))

    __mapper_args__ = {
        "polymorphic_identity": "manager",
        "concrete": True,
    }

class Engineer(Employee):
    __tablename__ = "engineer"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    engineer_info = mapped_column(String(40))

    __mapper_args__ = {
        "polymorphic_identity": "engineer",
        "concrete": True,
    }
```

上面，Declarative 在映射器“初始化”时为`Employee`类设置了多态可选项；这是解决其他依赖映射器的延迟配置步骤。`ConcreteBase`助手使用`polymorphic_union()`函数在设置了所有其他类后创建所有具体映射表的 UNION，并然后使用已经存在的基类映射器配置此语句。

在选择时，多态联合会产生这样的查询：

```py
session.scalars(select(Employee)).all()
SELECT
  pjoin.id,
  pjoin.name,
  pjoin.type,
  pjoin.manager_data,
  pjoin.engineer_info
FROM  (
  SELECT
  employee.id  AS  id,
  employee.name  AS  name,
  CAST(NULL  AS  VARCHAR(40))  AS  manager_data,
  CAST(NULL  AS  VARCHAR(40))  AS  engineer_info,
  'employee'  AS  type
  FROM  employee
  UNION  ALL
  SELECT
  manager.id  AS  id,
  manager.name  AS  name,
  manager.manager_data  AS  manager_data,
  CAST(NULL  AS  VARCHAR(40))  AS  engineer_info,
  'manager'  AS  type
  FROM  manager
  UNION  ALL
  SELECT
  engineer.id  AS  id,
  engineer.name  AS  name,
  CAST(NULL  AS  VARCHAR(40))  AS  manager_data,
  engineer.engineer_info  AS  engineer_info,
  'engineer'  AS  type
  FROM  engineer
)  AS  pjoin 
```

上面的 UNION 查询需要为每个子表制造“NULL”列，以适应那些不是特定子类成员的列。

另请参阅

`ConcreteBase`  ### 抽象具体类

到目前为止，所示的具体映射同时显示了子类和基类分别映射到各自的表中。在具体继承用例中，通常基类在数据库中不会被表示，只有子类。换句话说，基类是“抽象的”。

通常，当想要将两个不同的子类映射到各自的表中，并且将基类保持未映射时，这可以很容易地实现。在使用 Declarative 时，只需使用`__abstract__`指示符声明基类：

```py
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    pass

class Employee(Base):
    __abstract__ = True

class Manager(Employee):
    __tablename__ = "manager"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    manager_data = mapped_column(String(40))

class Engineer(Employee):
    __tablename__ = "engineer"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    engineer_info = mapped_column(String(40))
```

上面，我们实际上并没有使用 SQLAlchemy 的继承映射功能；我们可以正常加载和持久化`Manager`和`Engineer`的实例。然而，当我们需要**多态查询**时，情况就会发生变化，也就是说，我们希望发出`select(Employee)`并返回`Manager`和`Engineer`实例的集合。这将我们带回到具体继承的领域，我们必须针对`Employee`构建一个特殊的映射器才能实现这一点。

要修改我们的具体继承示例，以说明一个能够进行多态加载的“抽象”基类，我们将只有一个`engineer`和一个`manager`表，没有`employee`表，但是`Employee`映射器将直接映射到“多态联合”，而不是在`Mapper.with_polymorphic`参数中本地指定它。

为了帮助解决这个问题，Declarative 提供了一种名为`AbstractConcreteBase`的`ConcreteBase`类的变体，它可以自动实现这一点：

```py
from sqlalchemy.ext.declarative import AbstractConcreteBase
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    pass

class Employee(AbstractConcreteBase, Base):
    strict_attrs = True

    name = mapped_column(String(50))

class Manager(Employee):
    __tablename__ = "manager"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    manager_data = mapped_column(String(40))

    __mapper_args__ = {
        "polymorphic_identity": "manager",
        "concrete": True,
    }

class Engineer(Employee):
    __tablename__ = "engineer"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    engineer_info = mapped_column(String(40))

    __mapper_args__ = {
        "polymorphic_identity": "engineer",
        "concrete": True,
    }

Base.registry.configure()
```

上面，调用了`registry.configure()`方法，这将触发实际映射`Employee`类；在配置步骤之前，类没有映射，因为它将从中查询的子表尚未定义。此过程比`ConcreteBase`更复杂，因为必须延迟基类的整个映射，直到所有子类都已声明。使用像上面的映射，只能持久化`Manager`和`Engineer`的实例；对`Employee`类进行查询将始终产生`Manager`和`Engineer`对象。

使用上述映射，可以根据`Employee`类和在其上本地声明的任何属性生成查询，例如`Employee.name`：

```py
>>> stmt = select(Employee).where(Employee.name == "n1")
>>> print(stmt)
SELECT  pjoin.id,  pjoin.name,  pjoin.type,  pjoin.manager_data,  pjoin.engineer_info
FROM  (
  SELECT  engineer.id  AS  id,  engineer.name  AS  name,  engineer.engineer_info  AS  engineer_info,
  CAST(NULL  AS  VARCHAR(40))  AS  manager_data,  'engineer'  AS  type
  FROM  engineer
  UNION  ALL
  SELECT  manager.id  AS  id,  manager.name  AS  name,  CAST(NULL  AS  VARCHAR(40))  AS  engineer_info,
  manager.manager_data  AS  manager_data,  'manager'  AS  type
  FROM  manager
)  AS  pjoin
WHERE  pjoin.name  =  :name_1 
```

`AbstractConcreteBase.strict_attrs` 参数指示 `Employee` 类应直接映射仅属于 `Employee` 类的属性，如本例中的 `Employee.name` 属性。其他属性，如 `Manager.manager_data` 和 `Engineer.engineer_info`，仅存在于其相应的子类中。当未设置 `AbstractConcreteBase.strict_attrs` 时，所有子类属性（如 `Manager.manager_data` 和 `Engineer.engineer_info`）都将映射到基类 `Employee`。这是一种传统的使用模式，可能更方便查询，但其效果是所有子类共享整个层次结构的完整属性集；在上述示例中，不使用 `AbstractConcreteBase.strict_attrs` 将导致生成非实用的 `Engineer.manager_name` 和 `Manager.engineer_info` 属性。

新版 2.0 中：新增了 `AbstractConcreteBase.strict_attrs` 参数到 `AbstractConcreteBase` 中，以产生更清晰的映射；默认值为 False，以允许继续使用旧版 1.x 版本中的传统映射。

另请参阅

`AbstractConcreteBase`

### 经典和半经典具体多态配置

使用 `ConcreteBase` 和 `AbstractConcreteBase` 说明的声明性配置相当于另外两种使用 `polymorphic_union()` 显式的配置形式。 这些配置形式明确使用 `Table` 对象，以便首先创建“多态联合”，然后将其应用于映射。 这些示例旨在澄清 `polymorphic_union()` 函数在映射中的作用。

例如，**半经典映射**利用声明性，但分别建立 `Table` 对象：

```py
metadata_obj = Base.metadata

employees_table = Table(
    "employee",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    Column("name", String(50)),
)

managers_table = Table(
    "manager",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    Column("name", String(50)),
    Column("manager_data", String(50)),
)

engineers_table = Table(
    "engineer",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    Column("name", String(50)),
    Column("engineer_info", String(50)),
)
```

接下来，使用 `polymorphic_union()` 生成 UNION：

```py
from sqlalchemy.orm import polymorphic_union

pjoin = polymorphic_union(
    {
        "employee": employees_table,
        "manager": managers_table,
        "engineer": engineers_table,
    },
    "type",
    "pjoin",
)
```

使用上述 `Table` 对象，可以使用“半经典”样式生成映射，在此样式中，我们与 `__table__` 参数一起使用声明性；我们上面的多态联合通过 `__mapper_args__` 传递给 `Mapper.with_polymorphic` 参数：

```py
class Employee(Base):
    __table__ = employee_table
    __mapper_args__ = {
        "polymorphic_on": pjoin.c.type,
        "with_polymorphic": ("*", pjoin),
        "polymorphic_identity": "employee",
    }

class Engineer(Employee):
    __table__ = engineer_table
    __mapper_args__ = {
        "polymorphic_identity": "engineer",
        "concrete": True,
    }

class Manager(Employee):
    __table__ = manager_table
    __mapper_args__ = {
        "polymorphic_identity": "manager",
        "concrete": True,
    }
```

或者，可以完全以“经典”风格使用相同的 `Table` 对象，而不使用声明性。 构造函数与声明性提供的类似，如下所示：

```py
class Employee:
    def __init__(self, **kw):
        for k in kw:
            setattr(self, k, kw[k])

class Manager(Employee):
    pass

class Engineer(Employee):
    pass

employee_mapper = mapper_registry.map_imperatively(
    Employee,
    pjoin,
    with_polymorphic=("*", pjoin),
    polymorphic_on=pjoin.c.type,
)
manager_mapper = mapper_registry.map_imperatively(
    Manager,
    managers_table,
    inherits=employee_mapper,
    concrete=True,
    polymorphic_identity="manager",
)
engineer_mapper = mapper_registry.map_imperatively(
    Engineer,
    engineers_table,
    inherits=employee_mapper,
    concrete=True,
    polymorphic_identity="engineer",
)
```

"抽象" 示例也可以使用“半经典”或“经典”风格进行映射。 不同之处在于，我们不再将“多态联合”应用于 `Mapper.with_polymorphic` 参数，而是直接将其作为我们最基本的映射器上的映射选择。 半经典映射如下所示：

```py
from sqlalchemy.orm import polymorphic_union

pjoin = polymorphic_union(
    {
        "manager": managers_table,
        "engineer": engineers_table,
    },
    "type",
    "pjoin",
)

class Employee(Base):
    __table__ = pjoin
    __mapper_args__ = {
        "polymorphic_on": pjoin.c.type,
        "with_polymorphic": "*",
        "polymorphic_identity": "employee",
    }

class Engineer(Employee):
    __table__ = engineer_table
    __mapper_args__ = {
        "polymorphic_identity": "engineer",
        "concrete": True,
    }

class Manager(Employee):
    __table__ = manager_table
    __mapper_args__ = {
        "polymorphic_identity": "manager",
        "concrete": True,
    }
```

在上面的示例中，我们与以前一样使用 `polymorphic_union()`，只是省略了`employee`表。

另请参阅

命令式映射 - 有关命令式或“经典”映射的背景信息

### 具体继承关系

在具体继承场景中，映射关系是具有挑战性的，因为不同的类不共享一个表。如果关系只涉及特定类，例如我们之前示例中的`Company`和`Manager`之间的关系，那么不需要特殊步骤，因为这只是两个相关表。

然而，如果`Company`要与`Employee`建立一对多关系，表示集合可能包括`Engineer`和`Manager`对象，这意味着`Employee`必须具有多态加载能力，并且要关联的每个表都必须有一个外键返回到`company`表。这种配置的示例如下：

```py
from sqlalchemy.ext.declarative import ConcreteBase

class Company(Base):
    __tablename__ = "company"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    employees = relationship("Employee")

class Employee(ConcreteBase, Base):
    __tablename__ = "employee"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    company_id = mapped_column(ForeignKey("company.id"))

    __mapper_args__ = {
        "polymorphic_identity": "employee",
        "concrete": True,
    }

class Manager(Employee):
    __tablename__ = "manager"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    manager_data = mapped_column(String(40))
    company_id = mapped_column(ForeignKey("company.id"))

    __mapper_args__ = {
        "polymorphic_identity": "manager",
        "concrete": True,
    }

class Engineer(Employee):
    __tablename__ = "engineer"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    engineer_info = mapped_column(String(40))
    company_id = mapped_column(ForeignKey("company.id"))

    __mapper_args__ = {
        "polymorphic_identity": "engineer",
        "concrete": True,
    }
```

具体继承和关系的下一个复杂性涉及当我们希望`Employee`、`Manager`和`Engineer`中的一个或全部自身引用`Company`时。对于这种情况，SQLAlchemy 在 `Employee` 上放置一个与 `Company` 相关的 `relationship()` 时，在实例级别执行时不适用于 `Manager` 和 `Engineer` 类，而必须对每个类应用一个不同的 `relationship()`。为了实现三个独立关系的双向行为，这些关系作为 `Company.employees` 的相反关系，使用了 `relationship.back_populates` 参数：

```py
from sqlalchemy.ext.declarative import ConcreteBase

class Company(Base):
    __tablename__ = "company"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    employees = relationship("Employee", back_populates="company")

class Employee(ConcreteBase, Base):
    __tablename__ = "employee"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    company_id = mapped_column(ForeignKey("company.id"))
    company = relationship("Company", back_populates="employees")

    __mapper_args__ = {
        "polymorphic_identity": "employee",
        "concrete": True,
    }

class Manager(Employee):
    __tablename__ = "manager"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    manager_data = mapped_column(String(40))
    company_id = mapped_column(ForeignKey("company.id"))
    company = relationship("Company", back_populates="employees")

    __mapper_args__ = {
        "polymorphic_identity": "manager",
        "concrete": True,
    }

class Engineer(Employee):
    __tablename__ = "engineer"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    engineer_info = mapped_column(String(40))
    company_id = mapped_column(ForeignKey("company.id"))
    company = relationship("Company", back_populates="employees")

    __mapper_args__ = {
        "polymorphic_identity": "engineer",
        "concrete": True,
    }
```

上述限制与当前实现相关，包括具体继承类不共享超类的任何属性，因此需要设置不同的关系。

### 加载具体继承映射

具体继承加载选项有限；通常，如果在映射器上配置了多态加载，使用其中一个声明性具体混合类，就不能在当前 SQLAlchemy 版本中在查询时修改它。通常，`with_polymorphic()` 函数应该能够覆盖具体使用的加载样式，但由于当前限制，这还不受支持。  ## 连接表继承

在连接表继承中，类层次结构中的每个类都由一个不同的表表示。在层次结构中查询特定子类将作为 SQL JOIN 渲染其继承路径上的所有表。如果查询的类是基类，则将查询基表，同时可以选择包括其他表或允许特定于子表的属性稍后加载。

在所有情况下，给定行的最终实例化类由基类上定义的鉴别器列或 SQL 表达式确定，该列将产生与特定子类关联的标量值。

连接继承层次结构中的基类将配置具有指示多态鉴别器列的额外参数，以及可选的基类自身的多态标识符：

```py
from sqlalchemy import ForeignKey
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column

class Base(DeclarativeBase):
    pass

class Employee(Base):
    __tablename__ = "employee"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    type: Mapped[str]

    __mapper_args__ = {
        "polymorphic_identity": "employee",
        "polymorphic_on": "type",
    }

    def __repr__(self):
        return f"{self.__class__.__name__}({self.name!r})"
```

在上面的例子中，鉴别器是`type`列，可以使用`Mapper.polymorphic_on`参数进行配置。该参数接受一个基于列的表达式，可以指定为要使用的映射属性的字符串名称，也可以指定为列表达式对象，如`Column`或`mapped_column()`构造。

鉴别器列将存储一个值，该值指示行中表示的对象类型。该列可以是任何数据类型，但字符串和整数最常见。为数据库中的特定行应用于此列的实际数据值是使用`Mapper.polymorphic_identity`参数指定的，如下所述。

虽然多态鉴别器表达式不是严格必需的，但如果需要多态加载，则需要。在基表上建立一个列是实现此目的的最简单方法，但是非常复杂的继承映射可能会使用 SQL 表达式，例如 CASE 表达式，作为多态鉴别器。

注意

目前，**整个继承层次结构仅可以配置一个鉴别器列或 SQL 表达式**，通常在层次结构中最基本的类上。目前不支持“级联”多态鉴别器表达式。

我们接下来定义`Engineer`和`Manager`作为`Employee`的子类。每个子类包含代表其所代表子类的唯一属性的列。每个表还必须包含主键列（或列）以及对父表的外键引用：

```py
class Engineer(Employee):
    __tablename__ = "engineer"
    id: Mapped[int] = mapped_column(ForeignKey("employee.id"), primary_key=True)
    engineer_name: Mapped[str]

    __mapper_args__ = {
        "polymorphic_identity": "engineer",
    }

class Manager(Employee):
    __tablename__ = "manager"
    id: Mapped[int] = mapped_column(ForeignKey("employee.id"), primary_key=True)
    manager_name: Mapped[str]

    __mapper_args__ = {
        "polymorphic_identity": "manager",
    }
```

在上面的示例中，每个映射在其映射器参数中指定了`Mapper.polymorphic_identity`参数。此值填充了由基本映射器上建立的`Mapper.polymorphic_on`参数指定的列。`Mapper.polymorphic_identity`参数应该对整个层次结构中的每个映射类是唯一的，并且每个映射类只应有一个“标识”；如上所述，不支持一些子类引入第二个标识的“级联”标识。

ORM 使用`Mapper.polymorphic_identity`设置的值来确定加载行时行属于哪个类。在上面的示例中，每个代表`Employee`的行在其`type`列中将有值`'employee'`；同样，每个`Engineer`将获得值`'engineer'`，每个`Manager`将获得值`'manager'`。无论继承映射使用不同的联接表作为子类（如联合表继承）还是所有一个表作为单表继承，这个值都应该被持久化并在查询时对 ORM 可用。`Mapper.polymorphic_identity`参数也适用于具体表继承，但实际上并没有被持久化；有关详细信息，请参阅后面的具体表继承部分。

在多态设置中，最常见的是外键约束建立在与主键本身相同的列或列上，但这并非必需；一个与主键不同的列也可以通过外键指向父类。从基本表到子类构建 JOIN 的方式也是可以直接自定义的，但这很少是必要的。

完成联合继承映射后，针对`Employee`的查询将返回`Employee`、`Engineer`和`Manager`对象的组合。新保存的`Engineer`、`Manager`和`Employee`对象将自动填充`employee.type`列，此时正确的“鉴别器”值为`"engineer"`、`"manager"`或`"employee"`。

### 具有联合继承关系

与联合表继承完全支持关系。涉及联合继承类的关系应该针对与外键约束对应的层次结构中的类；在下面的示例中，由于`employee`表有一个指向`company`表的外键约束，关系被建立在`Company`和`Employee`之间：

```py
from __future__ import annotations

from sqlalchemy.orm import relationship

class Company(Base):
    __tablename__ = "company"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    employees: Mapped[List[Employee]] = relationship(back_populates="company")

class Employee(Base):
    __tablename__ = "employee"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    type: Mapped[str]
    company_id: Mapped[int] = mapped_column(ForeignKey("company.id"))
    company: Mapped[Company] = relationship(back_populates="employees")

    __mapper_args__ = {
        "polymorphic_identity": "employee",
        "polymorphic_on": "type",
    }

class Manager(Employee): ...

class Engineer(Employee): ...
```

如果外键约束在对应于子类的表上，则关系应该指向该子类。在下面的示例中，从`manager`到`company`有一个外键约束，因此建立了`Manager`和`Company`类之间的关系：

```py
class Company(Base):
    __tablename__ = "company"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    managers: Mapped[List[Manager]] = relationship(back_populates="company")

class Employee(Base):
    __tablename__ = "employee"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    type: Mapped[str]

    __mapper_args__ = {
        "polymorphic_identity": "employee",
        "polymorphic_on": "type",
    }

class Manager(Employee):
    __tablename__ = "manager"
    id: Mapped[int] = mapped_column(ForeignKey("employee.id"), primary_key=True)
    manager_name: Mapped[str]

    company_id: Mapped[int] = mapped_column(ForeignKey("company.id"))
    company: Mapped[Company] = relationship(back_populates="managers")

    __mapper_args__ = {
        "polymorphic_identity": "manager",
    }

class Engineer(Employee): ...
```

在上面，`Manager`类将具有`Manager.company`属性；`Company`将具有`Company.managers`属性，总是针对`employee`和`manager`表一起加载的连接进行加载。

### 加载连接继承映射

请参阅编写用于继承映射的 SELECT 语句部分，了解继承加载技术的背景，包括在映射器配置时间和查询时间配置要查询的表。

### 具有连接继承的关系

与连接表继承完全支持关系。涉及连接继承类的关系应该指向与外键约束对应的层次结构中的类；在下面的示例中，由于`employee`表有一个指向`company`表的外键约束，因此在`Company`和`Employee`之间建立了关系：

```py
from __future__ import annotations

from sqlalchemy.orm import relationship

class Company(Base):
    __tablename__ = "company"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    employees: Mapped[List[Employee]] = relationship(back_populates="company")

class Employee(Base):
    __tablename__ = "employee"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    type: Mapped[str]
    company_id: Mapped[int] = mapped_column(ForeignKey("company.id"))
    company: Mapped[Company] = relationship(back_populates="employees")

    __mapper_args__ = {
        "polymorphic_identity": "employee",
        "polymorphic_on": "type",
    }

class Manager(Employee): ...

class Engineer(Employee): ...
```

如果外键约束在对应于子类的表上，则关系应该指向该子类。在下面的示例中，从`manager`到`company`有一个外键约束，因此建立了`Manager`和`Company`类之间的关系：

```py
class Company(Base):
    __tablename__ = "company"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    managers: Mapped[List[Manager]] = relationship(back_populates="company")

class Employee(Base):
    __tablename__ = "employee"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    type: Mapped[str]

    __mapper_args__ = {
        "polymorphic_identity": "employee",
        "polymorphic_on": "type",
    }

class Manager(Employee):
    __tablename__ = "manager"
    id: Mapped[int] = mapped_column(ForeignKey("employee.id"), primary_key=True)
    manager_name: Mapped[str]

    company_id: Mapped[int] = mapped_column(ForeignKey("company.id"))
    company: Mapped[Company] = relationship(back_populates="managers")

    __mapper_args__ = {
        "polymorphic_identity": "manager",
    }

class Engineer(Employee): ...
```

在上面，`Manager`类将具有`Manager.company`属性；`Company`将具有`Company.managers`属性，总是针对`employee`和`manager`表一起加载的连接进行加载。

### 加载连接继承映射

请参阅编写用于继承映射的 SELECT 语句部分，了解继承加载技术的背景，包括在映射器配置时间和查询时间配置要查询的表。

## 单表继承

单表继承将所有子类的所有属性表示为单个表中的内容。具有特定类别属性的特定子类将在表中的列中保留它们，如果行引用不同类型的对象，则列中将为空。

在层次结构中查询特定子类将呈现为针对基表的 SELECT，其中将包括一个 WHERE 子句，该子句将限制行为具有鉴别器列或表达式中存在的特定值或值。

单表继承相对于连接表继承具有简单性的优势；查询效率更高，因为只需要涉及一个表来加载每个表示类的对象。

单表继承配置看起来很像连接表继承，只是基类指定了`__tablename__`。基表还需要一个鉴别器列，以便类之间可以区分开来。

即使子类共享所有属性的基本表，当使用 Declarative 时，仍然可以在子类上指定 `mapped_column` 对象，指示该列仅映射到该子类；`mapped_column` 将应用于相同的基本 `Table` 对象：

```py
class Employee(Base):
    __tablename__ = "employee"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    type: Mapped[str]

    __mapper_args__ = {
        "polymorphic_on": "type",
        "polymorphic_identity": "employee",
    }

class Manager(Employee):
    manager_data: Mapped[str] = mapped_column(nullable=True)

    __mapper_args__ = {
        "polymorphic_identity": "manager",
    }

class Engineer(Employee):
    engineer_info: Mapped[str] = mapped_column(nullable=True)

    __mapper_args__ = {
        "polymorphic_identity": "engineer",
    }
```

注意到派生类 Manager 和 Engineer 的映射器省略了 `__tablename__`，表明它们没有自己的映射表。此外，还包括一个带有 `nullable=True` 的 `mapped_column()` 指令；由于为这些类声明的 Python 类型不包括 `Optional[]`，该列通常会被映射为 `NOT NULL`，这对于只期望为对应于特定子类的那些行填充的列来说是不合适的。

### 使用 `use_existing_column` 解决列冲突

注意在前一节中，`manager_name` 和 `engineer_info` 列被“上移”应用到 `Employee.__table__`，因为它们在没有自己表的子类上声明。当两个子类想要指定*相同*列时，就会出现一个棘手的情况，如下所示：

```py
from datetime import datetime

class Employee(Base):
    __tablename__ = "employee"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    type: Mapped[str]

    __mapper_args__ = {
        "polymorphic_on": "type",
        "polymorphic_identity": "employee",
    }

class Engineer(Employee):
    __mapper_args__ = {
        "polymorphic_identity": "engineer",
    }
    start_date: Mapped[datetime] = mapped_column(nullable=True)

class Manager(Employee):
    __mapper_args__ = {
        "polymorphic_identity": "manager",
    }
    start_date: Mapped[datetime] = mapped_column(nullable=True)
```

上面，在 `Engineer` 和 `Manager` 上声明的 `start_date` 列将导致错误：

```py
sqlalchemy.exc.ArgumentError: Column 'start_date' on class Manager conflicts
with existing column 'employee.start_date'.  If using Declarative,
consider using the use_existing_column parameter of mapped_column() to
resolve conflicts.
```

上述情况对 Declarative 映射系统提出了一个模棱两可的问题，可以通过在 `mapped_column()` 上使用 `mapped_column.use_existing_column` 参数来解决，该参数指示 `mapped_column()` 查找并使用已经映射的继承超类上的列，如果已经存在，否则映射一个新列：

```py
from sqlalchemy import DateTime

class Employee(Base):
    __tablename__ = "employee"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    type: Mapped[str]

    __mapper_args__ = {
        "polymorphic_on": "type",
        "polymorphic_identity": "employee",
    }

class Engineer(Employee):
    __mapper_args__ = {
        "polymorphic_identity": "engineer",
    }

    start_date: Mapped[datetime] = mapped_column(
        nullable=True, use_existing_column=True
    )

class Manager(Employee):
    __mapper_args__ = {
        "polymorphic_identity": "manager",
    }

    start_date: Mapped[datetime] = mapped_column(
        nullable=True, use_existing_column=True
    )
```

在上面的例子中，当`Manager`被映射时，`start_date`列已经存在于`Employee`类上，已经由`Engineer`映射提供。`mapped_column.use_existing_column`参数指示`mapped_column()`应该首先在`Employee`的映射`Table`上查找请求的`Column`，如果存在，则保持该现有映射。如果不存在，`mapped_column()`将正常映射该列，将其添加为`Employee`超类引用的`Table`中的列之一。

2.0.0b4 版本中新增：- 添加了`mapped_column.use_existing_column`，提供了一种符合 2.0 版本的方式来有条件地映射继承子类上的列。之前的方法结合了`declared_attr`和对父类`.__table__`的查找仍然有效，但缺乏[**PEP 484**](https://peps.python.org/pep-0484/)类型支持。

类似的概念可以与混合类一起使用（参见使用混合类组合映射层次结构）来定义一系列特定的列和/或其他可重用混合类中的映射属性：

```py
class Employee(Base):
    __tablename__ = "employee"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    type: Mapped[str]

    __mapper_args__ = {
        "polymorphic_on": type,
        "polymorphic_identity": "employee",
    }

class HasStartDate:
    start_date: Mapped[datetime] = mapped_column(
        nullable=True, use_existing_column=True
    )

class Engineer(HasStartDate, Employee):
    __mapper_args__ = {
        "polymorphic_identity": "engineer",
    }

class Manager(HasStartDate, Employee):
    __mapper_args__ = {
        "polymorphic_identity": "manager",
    }
```

### 与单表继承的关系

与单表继承完全支持关系。配置方式与连接继承相同；外键属性应该在与关系的“外键”一侧相同的类上：

```py
class Company(Base):
    __tablename__ = "company"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    employees: Mapped[List[Employee]] = relationship(back_populates="company")

class Employee(Base):
    __tablename__ = "employee"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    type: Mapped[str]
    company_id: Mapped[int] = mapped_column(ForeignKey("company.id"))
    company: Mapped[Company] = relationship(back_populates="employees")

    __mapper_args__ = {
        "polymorphic_identity": "employee",
        "polymorphic_on": "type",
    }

class Manager(Employee):
    manager_data: Mapped[str] = mapped_column(nullable=True)

    __mapper_args__ = {
        "polymorphic_identity": "manager",
    }

class Engineer(Employee):
    engineer_info: Mapped[str] = mapped_column(nullable=True)

    __mapper_args__ = {
        "polymorphic_identity": "engineer",
    }
```

此外，与连接继承的情况类似，我们可以创建涉及特定子类的关系。在查询时，SELECT 语句将包含一个 WHERE 子句，将类选择限制为该子类或子类：

```py
class Company(Base):
    __tablename__ = "company"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    managers: Mapped[List[Manager]] = relationship(back_populates="company")

class Employee(Base):
    __tablename__ = "employee"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    type: Mapped[str]

    __mapper_args__ = {
        "polymorphic_identity": "employee",
        "polymorphic_on": "type",
    }

class Manager(Employee):
    manager_name: Mapped[str] = mapped_column(nullable=True)

    company_id: Mapped[int] = mapped_column(ForeignKey("company.id"))
    company: Mapped[Company] = relationship(back_populates="managers")

    __mapper_args__ = {
        "polymorphic_identity": "manager",
    }

class Engineer(Employee):
    engineer_info: Mapped[str] = mapped_column(nullable=True)

    __mapper_args__ = {
        "polymorphic_identity": "engineer",
    }
```

在上面的例子中，`Manager`类将具有`Manager.company`属性；`Company`将具有`Company.managers`属性，始终针对具有额外 WHERE 子句的`employee`加载，该子句将行限制为`type = 'manager'`的行。

### 使用`polymorphic_abstract`构建更深层次的层次结构

2.0 版本中新增。

在构建任何继承层次结构时，一个映射类可以设置`Mapper.polymorphic_abstract`参数为`True`，这表示该类应该被正常映射，但不希望直接实例化，并且不包含`Mapper.polymorphic_identity`。然后可以声明这个映射类的子类，这些子类可以包含`Mapper.polymorphic_identity`，因此可以被正常使用。这允许一系列子类被一个公共基类引用，该基类在层次结构中被认为是“抽象的”，无论是在查询中还是在`relationship()`声明中。这种用法与在 Declarative 中使用 __abstract__ 属性的用法不同，后者将目标类完全未映射，因此不能单独作为映射类使用。`Mapper.polymorphic_abstract`可以应用于层次结构中的任何类或类，包括同时在多个级别上应用。

举例来说，假设`Manager`和`Principal`都被分类到一个超类`Executive`下，而`Engineer`和`Sysadmin`被分类到一个超类`Technologist`下。`Executive`和`Technologist`都不会被实例化，因此没有`Mapper.polymorphic_identity`。这些类可以通过`Mapper.polymorphic_abstract`进行配置如下：

```py
class Employee(Base):
    __tablename__ = "employee"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    type: Mapped[str]

    __mapper_args__ = {
        "polymorphic_identity": "employee",
        "polymorphic_on": "type",
    }

class Executive(Employee):
  """An executive of the company"""

    executive_background: Mapped[str] = mapped_column(nullable=True)

    __mapper_args__ = {"polymorphic_abstract": True}

class Technologist(Employee):
  """An employee who works with technology"""

    competencies: Mapped[str] = mapped_column(nullable=True)

    __mapper_args__ = {"polymorphic_abstract": True}

class Manager(Executive):
  """a manager"""

    __mapper_args__ = {"polymorphic_identity": "manager"}

class Principal(Executive):
  """a principal of the company"""

    __mapper_args__ = {"polymorphic_identity": "principal"}

class Engineer(Technologist):
  """an engineer"""

    __mapper_args__ = {"polymorphic_identity": "engineer"}

class SysAdmin(Technologist):
  """a systems administrator"""

    __mapper_args__ = {"polymorphic_identity": "sysadmin"}
```

在上面的例子中，新的类`Technologist`和`Executive`都是普通的映射类，并且指示要添加到超类中的新列`executive_background`和`competencies`。然而，它们都缺少`Mapper.polymorphic_identity`的设置；这是因为不希望直接实例化`Technologist`或`Executive`；我们总是会有`Manager`、`Principal`、`Engineer`或`SysAdmin`中的一个。但是我们可以查询`Principal`和`Technologist`角色，并且让它们成为`relationship()`的目标。下面的示例演示了针对`Technologist`对象的 SELECT 语句：

```py
session.scalars(select(Technologist)).all()
SELECT  employee.id,  employee.name,  employee.type,  employee.competencies
FROM  employee
WHERE  employee.type  IN  (?,  ?)
[...]  ('engineer',  'sysadmin') 
```

`Technologist`和`Executive`抽象映射类也可以成为`relationship()`映射的目标，就像任何其他映射类一样。我们可以扩展上面的例子，包括`Company`，有单独的集合`Company.technologists`和`Company.principals`：

```py
class Company(Base):
    __tablename__ = "company"
    id = Column(Integer, primary_key=True)

    executives: Mapped[List[Executive]] = relationship()
    technologists: Mapped[List[Technologist]] = relationship()

class Employee(Base):
    __tablename__ = "employee"
    id: Mapped[int] = mapped_column(primary_key=True)

    # foreign key to "company.id" is added
    company_id: Mapped[int] = mapped_column(ForeignKey("company.id"))

    # rest of mapping is the same
    name: Mapped[str]
    type: Mapped[str]

    __mapper_args__ = {
        "polymorphic_on": "type",
    }

# Executive, Technologist, Manager, Principal, Engineer, SysAdmin
# classes from previous example would follow here unchanged
```

使用上述映射，我们可以分别跨`Company.technologists`和`Company.executives`使用连接和关系加载技术：

```py
session.scalars(
    select(Company)
    .join(Company.technologists)
    .where(Technologist.competency.ilike("%java%"))
    .options(selectinload(Company.executives))
).all()
SELECT  company.id
FROM  company  JOIN  employee  ON  company.id  =  employee.company_id  AND  employee.type  IN  (?,  ?)
WHERE  lower(employee.competencies)  LIKE  lower(?)
[...]  ('engineer',  'sysadmin',  '%java%')

SELECT  employee.company_id  AS  employee_company_id,  employee.id  AS  employee_id,
employee.name  AS  employee_name,  employee.type  AS  employee_type,
employee.executive_background  AS  employee_executive_background
FROM  employee
WHERE  employee.company_id  IN  (?)  AND  employee.type  IN  (?,  ?)
[...]  (1,  'manager',  'principal') 
```

另请参阅

__abstract__ - 声明性参数，允许声明性类在层次结构中完全取消映射，同时仍然从映射的超类扩展。

### 加载单一继承映射

单表继承的加载技术与联接表继承的加载技术基本相同，并且在这两种映射类型之间提供了高度的抽象，使得很容易在它们之间进行切换，以及在单个层次结构中混合使用它们（只需从要单继承的子类中省略 `__tablename__`）。请参阅 编写继承映射的 SELECT 语句 和 单一继承映射的 SELECT 语句 部分，了解有关继承加载技术的文档，包括在映射器配置时间和查询时间配置要查询的类。

### 使用 `use_existing_column` 解决列冲突

在上一节中注意到，`manager_name`和`engineer_info`列被“上移”，应用于`Employee.__table__`，因为它们在没有自己的表的子类上声明。当两个子类想要指定*相同*列时会出现棘手的情况，如下所示：

```py
from datetime import datetime

class Employee(Base):
    __tablename__ = "employee"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    type: Mapped[str]

    __mapper_args__ = {
        "polymorphic_on": "type",
        "polymorphic_identity": "employee",
    }

class Engineer(Employee):
    __mapper_args__ = {
        "polymorphic_identity": "engineer",
    }
    start_date: Mapped[datetime] = mapped_column(nullable=True)

class Manager(Employee):
    __mapper_args__ = {
        "polymorphic_identity": "manager",
    }
    start_date: Mapped[datetime] = mapped_column(nullable=True)
```

在上述代码中，同时在`Engineer`和`Manager`上声明的`start_date`列将导致错误：

```py
sqlalchemy.exc.ArgumentError: Column 'start_date' on class Manager conflicts
with existing column 'employee.start_date'.  If using Declarative,
consider using the use_existing_column parameter of mapped_column() to
resolve conflicts.
```

上述情景对声明性映射系统存在一种模糊性，可以通过在`mapped_column()`上使用`mapped_column.use_existing_column`参数来解决，该参数指示`mapped_column()`查找继承的超类，并使用已经存在的列，如果已经存在，则映射新列：

```py
from sqlalchemy import DateTime

class Employee(Base):
    __tablename__ = "employee"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    type: Mapped[str]

    __mapper_args__ = {
        "polymorphic_on": "type",
        "polymorphic_identity": "employee",
    }

class Engineer(Employee):
    __mapper_args__ = {
        "polymorphic_identity": "engineer",
    }

    start_date: Mapped[datetime] = mapped_column(
        nullable=True, use_existing_column=True
    )

class Manager(Employee):
    __mapper_args__ = {
        "polymorphic_identity": "manager",
    }

    start_date: Mapped[datetime] = mapped_column(
        nullable=True, use_existing_column=True
    )
```

在上文中，当 `Manager` 被映射时，`start_date` 列已经存在于 `Employee` 类中，已经由之前的 `Engineer` 映射提供。`mapped_column.use_existing_column` 参数指示给 `mapped_column()`，它应该首先查找映射到 `Employee` 的映射 `Table` 上的请求的 `Column`，如果存在，则保持该现有映射。如果不存在，`mapped_column()` 将正常映射列，将其添加为 `Employee` 超类引用的 `Table` 中的列之一。

新版本 2.0.0b4 中新增：- 添加了 `mapped_column.use_existing_column`，它提供了一个与 2.0 兼容的方法，以条件地映射继承子类上的列。先前的方法结合了 `declared_attr` 与对父类 `.__table__` 的查找，仍然有效，但缺少了 [**PEP 484**](https://peps.python.org/pep-0484/) 类型支持。

一个类似的概念可以与 mixin 类一起使用（参见 Composing Mapped Hierarchies with Mixins）来定义来自可重用 mixin 类的特定系列列和/或其他映射属性：

```py
class Employee(Base):
    __tablename__ = "employee"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    type: Mapped[str]

    __mapper_args__ = {
        "polymorphic_on": type,
        "polymorphic_identity": "employee",
    }

class HasStartDate:
    start_date: Mapped[datetime] = mapped_column(
        nullable=True, use_existing_column=True
    )

class Engineer(HasStartDate, Employee):
    __mapper_args__ = {
        "polymorphic_identity": "engineer",
    }

class Manager(HasStartDate, Employee):
    __mapper_args__ = {
        "polymorphic_identity": "manager",
    }
```

### 单表继承的关系

关系在单表继承中得到充分支持。配置方式与联接继承的方式相同；外键属性应位于与关系的“外部”一侧相同的类上：

```py
class Company(Base):
    __tablename__ = "company"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    employees: Mapped[List[Employee]] = relationship(back_populates="company")

class Employee(Base):
    __tablename__ = "employee"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    type: Mapped[str]
    company_id: Mapped[int] = mapped_column(ForeignKey("company.id"))
    company: Mapped[Company] = relationship(back_populates="employees")

    __mapper_args__ = {
        "polymorphic_identity": "employee",
        "polymorphic_on": "type",
    }

class Manager(Employee):
    manager_data: Mapped[str] = mapped_column(nullable=True)

    __mapper_args__ = {
        "polymorphic_identity": "manager",
    }

class Engineer(Employee):
    engineer_info: Mapped[str] = mapped_column(nullable=True)

    __mapper_args__ = {
        "polymorphic_identity": "engineer",
    }
```

类似于联接继承的情况，我们也可以创建涉及特定子类的关系。当查询时，SELECT 语句将包含一个 WHERE 子句，将类的选择限制为该子类或子类：

```py
class Company(Base):
    __tablename__ = "company"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    managers: Mapped[List[Manager]] = relationship(back_populates="company")

class Employee(Base):
    __tablename__ = "employee"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    type: Mapped[str]

    __mapper_args__ = {
        "polymorphic_identity": "employee",
        "polymorphic_on": "type",
    }

class Manager(Employee):
    manager_name: Mapped[str] = mapped_column(nullable=True)

    company_id: Mapped[int] = mapped_column(ForeignKey("company.id"))
    company: Mapped[Company] = relationship(back_populates="managers")

    __mapper_args__ = {
        "polymorphic_identity": "manager",
    }

class Engineer(Employee):
    engineer_info: Mapped[str] = mapped_column(nullable=True)

    __mapper_args__ = {
        "polymorphic_identity": "engineer",
    }
```

上文中，`Manager` 类将具有一个 `Manager.company` 属性；`Company` 将具有一个 `Company.managers` 属性，该属性始终加载针对具有附加 WHERE 子句的 `employee`，限制行为具有 `type = 'manager'` 的行。

### 使用 `polymorphic_abstract` 构建更深层次的层次结构

新版本 2.0 中新增。

在构建任何类型的继承层次结构时，映射类可以包含设置为`True`的`Mapper.polymorphic_abstract`参数，表示该类应该正常映射，但不期望直接实例化，并且不包括`Mapper.polymorphic_identity`。然后可以声明这个映射类的子类，这些子类本身可以包含`Mapper.polymorphic_identity`，因此可以正常使用。这允许一系列子类被一个被认为是层次结构内“抽象”的公共基类引用，无论是在查询中还是在`relationship()`声明中。这种用法与在 Declarative 中使用 __abstract__ 属性的用法不同，后者使目标类完全未映射，因此不能作为一个映射类单独使用。`Mapper.polymorphic_abstract`可以应用于层次结构中的任何类或类，包括一次在多个级别上。

举个例子，假设要将`Manager`和`Principal`都归类到一个超类`Executive`下，而`Engineer`和`Sysadmin`则归类到一个超类`Technologist`下。`Executive`和`Technologist`都不会被实例化，因此没有`Mapper.polymorphic_identity`。可以使用`Mapper.polymorphic_abstract`来配置这些类，如下所示：

```py
class Employee(Base):
    __tablename__ = "employee"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    type: Mapped[str]

    __mapper_args__ = {
        "polymorphic_identity": "employee",
        "polymorphic_on": "type",
    }

class Executive(Employee):
  """An executive of the company"""

    executive_background: Mapped[str] = mapped_column(nullable=True)

    __mapper_args__ = {"polymorphic_abstract": True}

class Technologist(Employee):
  """An employee who works with technology"""

    competencies: Mapped[str] = mapped_column(nullable=True)

    __mapper_args__ = {"polymorphic_abstract": True}

class Manager(Executive):
  """a manager"""

    __mapper_args__ = {"polymorphic_identity": "manager"}

class Principal(Executive):
  """a principal of the company"""

    __mapper_args__ = {"polymorphic_identity": "principal"}

class Engineer(Technologist):
  """an engineer"""

    __mapper_args__ = {"polymorphic_identity": "engineer"}

class SysAdmin(Technologist):
  """a systems administrator"""

    __mapper_args__ = {"polymorphic_identity": "sysadmin"}
```

在上面的示例中，新类`Technologist`和`Executive`都是普通的映射类，并指示要添加到超类中的新列`executive_background`和`competencies`。然而，它们都缺少`Mapper.polymorphic_identity`的设置；这是因为不期望直接实例化`Technologist`或`Executive`；我们总是会有`Manager`、`Principal`、`Engineer`或`SysAdmin`中的一个。然而，我们可以查询`Principal`和`Technologist`角色，并使它们成为`relationship()`的目标。下面的示例演示了用于`Technologist`对象的 SELECT 语句：

```py
session.scalars(select(Technologist)).all()
SELECT  employee.id,  employee.name,  employee.type,  employee.competencies
FROM  employee
WHERE  employee.type  IN  (?,  ?)
[...]  ('engineer',  'sysadmin') 
```

抽象映射的`Technologist`和`Executive`抽象映射类也可以成为`relationship()`映射的目标，就像任何其他映射类一样。我们可以扩展上述示例以包括`Company`，并分别添加`Company.technologists`和`Company.principals`两个集合：

```py
class Company(Base):
    __tablename__ = "company"
    id = Column(Integer, primary_key=True)

    executives: Mapped[List[Executive]] = relationship()
    technologists: Mapped[List[Technologist]] = relationship()

class Employee(Base):
    __tablename__ = "employee"
    id: Mapped[int] = mapped_column(primary_key=True)

    # foreign key to "company.id" is added
    company_id: Mapped[int] = mapped_column(ForeignKey("company.id"))

    # rest of mapping is the same
    name: Mapped[str]
    type: Mapped[str]

    __mapper_args__ = {
        "polymorphic_on": "type",
    }

# Executive, Technologist, Manager, Principal, Engineer, SysAdmin
# classes from previous example would follow here unchanged
```

使用上述映射，我们可以分别在`Company.technologists`和`Company.executives`之间使用连接和关系加载技术：

```py
session.scalars(
    select(Company)
    .join(Company.technologists)
    .where(Technologist.competency.ilike("%java%"))
    .options(selectinload(Company.executives))
).all()
SELECT  company.id
FROM  company  JOIN  employee  ON  company.id  =  employee.company_id  AND  employee.type  IN  (?,  ?)
WHERE  lower(employee.competencies)  LIKE  lower(?)
[...]  ('engineer',  'sysadmin',  '%java%')

SELECT  employee.company_id  AS  employee_company_id,  employee.id  AS  employee_id,
employee.name  AS  employee_name,  employee.type  AS  employee_type,
employee.executive_background  AS  employee_executive_background
FROM  employee
WHERE  employee.company_id  IN  (?)  AND  employee.type  IN  (?,  ?)
[...]  (1,  'manager',  'principal') 
```

另请参见

__abstract__ - 声明性参数，允许在层次结构中完全取消映射 Declarative 类，同时仍从映射的超类扩展。

### 加载单继承映射

单表继承的加载技术大部分与用于连接表继承的技术相同，并且在这两种映射类型之间提供了很高程度的抽象，因此很容易在它们之间进行切换以及在单个层次结构中混合使用它们（只需从要单继承的子类中省略`__tablename__`）。请参阅编写继承映射的 SELECT 语句和单继承映射的 SELECT 语句章节，了解有关继承加载技术的文档，包括在映射器配置时间和查询时间配置要查询的类。

## 具体表继承

具体表继承将每个子类映射到其自己的独立表格，每个表格包含产生该类实例所需的所有列。具体继承配置默认情况下进行非多态查询；对于特定类的查询只会查询该类的表格，并且只返回该类的实例。具体类的多态加载通过在映射器内配置一个特殊的 SELECT 来启用，该 SELECT 通常被生成为所有表的 UNION。

警告

具体表继承比连接或单表继承**更加复杂**，在功能上**更加受限**，特别是在使用关系、急加载和多态加载方面。当以多态方式使用时，会产生**非常庞大的查询**，其中包含的 UNION 操作不会像简单的连接那样执行良好。强烈建议如果需要灵活性的关系加载和多态加载，尽可能使用连接或单表继承。如果不需要多态加载，则可以使用普通的非继承映射，如果每个类都完全引用其自己的表格。

虽然联接和单表继承在“多态”加载方面很流畅，但在具体继承中却是一种更笨拙的事情。因此，当**不需要多态加载**时，具体继承更为适用。建立涉及具体继承类的关系也更加麻烦。

要将类建立为使用具体继承，请在`__mapper_args__`中添加`Mapper.concrete`参数。这既表示对声明式以及映射，超类表不应被视为映射的一部分：

```py
class Employee(Base):
    __tablename__ = "employee"

    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))

class Manager(Employee):
    __tablename__ = "manager"

    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    manager_data = mapped_column(String(50))

    __mapper_args__ = {
        "concrete": True,
    }

class Engineer(Employee):
    __tablename__ = "engineer"

    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    engineer_info = mapped_column(String(50))

    __mapper_args__ = {
        "concrete": True,
    }
```

有两个关键点需要注意：

+   我们必须**在每个子类上显式定义所有列**，甚至是同名的列。例如，此处的`Employee.name`列**不会**被复制到由我们映射的`Manager`或`Engineer`表中。

+   虽然`Engineer`和`Manager`类在与`Employee`的继承关系中被映射，但它们**仍然不包括多态加载**。也就是说，如果我们查询`Employee`对象，`manager`和`engineer`表根本不会被查询。

### 具体多态加载配置

具有具体继承的多态加载要求针对应该具有多态加载的每个基类配置专门的 SELECT。此 SELECT 需要能够单独访问所有映射的表，并且通常是使用 SQLAlchemy 辅助程序`polymorphic_union()`构造的 UNION 语句。

如编写继承映射的 SELECT 语句所述，任何类型的映射器继承配置都可以使用`Mapper.with_polymorphic`参数默认配置从特殊的可选项中加载。当前公共 API 要求在首次构造`Mapper`时设置此参数。

但是，在声明式的情况下，映射器和被映射的`Table`同时创建，即在定义映射类的那一刻。这意味着暂时无法提供`Mapper.with_polymorphic`参数，因为子类对应的`Table`对象尚未定义。

有一些可用的策略来解决这个循环，然而声明性提供了帮助类`ConcreteBase` 和 `AbstractConcreteBase`，它们在幕后处理这个问题。

使用`ConcreteBase`，我们几乎可以以与其他形式的继承映射相同的方式设置我们的具体映射：

```py
from sqlalchemy.ext.declarative import ConcreteBase
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    pass

class Employee(ConcreteBase, Base):
    __tablename__ = "employee"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))

    __mapper_args__ = {
        "polymorphic_identity": "employee",
        "concrete": True,
    }

class Manager(Employee):
    __tablename__ = "manager"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    manager_data = mapped_column(String(40))

    __mapper_args__ = {
        "polymorphic_identity": "manager",
        "concrete": True,
    }

class Engineer(Employee):
    __tablename__ = "engineer"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    engineer_info = mapped_column(String(40))

    __mapper_args__ = {
        "polymorphic_identity": "engineer",
        "concrete": True,
    }
```

在上述示例中，声明性在映射器“初始化”时为`Employee`类设置了多态可选择项；这是解析其他依赖映射器的映射器的后配置步骤。`ConcreteBase` 帮助程序使用`polymorphic_union()`函数在设置了其他所有类之后创建所有具体映射表的 UNION，然后使用已经存在的基类映射器配置此语句。

在选择时，多态联合产生这样的查询：

```py
session.scalars(select(Employee)).all()
SELECT
  pjoin.id,
  pjoin.name,
  pjoin.type,
  pjoin.manager_data,
  pjoin.engineer_info
FROM  (
  SELECT
  employee.id  AS  id,
  employee.name  AS  name,
  CAST(NULL  AS  VARCHAR(40))  AS  manager_data,
  CAST(NULL  AS  VARCHAR(40))  AS  engineer_info,
  'employee'  AS  type
  FROM  employee
  UNION  ALL
  SELECT
  manager.id  AS  id,
  manager.name  AS  name,
  manager.manager_data  AS  manager_data,
  CAST(NULL  AS  VARCHAR(40))  AS  engineer_info,
  'manager'  AS  type
  FROM  manager
  UNION  ALL
  SELECT
  engineer.id  AS  id,
  engineer.name  AS  name,
  CAST(NULL  AS  VARCHAR(40))  AS  manager_data,
  engineer.engineer_info  AS  engineer_info,
  'engineer'  AS  type
  FROM  engineer
)  AS  pjoin 
```

上述 UNION 查询需要为每个子表制造“NULL”列，以容纳那些不属于特定子类的列。

另请参阅

`ConcreteBase`  ### 抽象具体类

到目前为止，所示的具体映射同时显示了子类和基类映射到各自的表中。在具体继承用例中，常见的是基类在数据库中没有表示，只有子类。换句话说，基类是“抽象的”。

通常，当一个人想要将两个不同的子类映射到各自的表中，并且保留基类未映射时，这可以非常容易地实现。在使用声明性时，只需使用`__abstract__`指示器声明基类：

```py
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    pass

class Employee(Base):
    __abstract__ = True

class Manager(Employee):
    __tablename__ = "manager"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    manager_data = mapped_column(String(40))

class Engineer(Employee):
    __tablename__ = "engineer"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    engineer_info = mapped_column(String(40))
```

在上述示例中，我们实际上没有利用 SQLAlchemy 的继承映射功能；我们可以正常加载和持久化`Manager`和`Engineer`的实例。然而，当我们需要**多态地查询**时，也就是说，我们想要发出`select(Employee)`并返回`Manager`和`Engineer`实例的集合时，情况就会发生变化。这让我们重新进入具体继承的领域，我们必须针对`Employee`构建一个特殊的映射器才能实现这一点。

要修改我们的具体继承示例以说明一个能够进行多态加载的“抽象”基类，我们将只有一个 `engineer` 和一个 `manager` 表，没有 `employee` 表，但是 `Employee` 映射器将直接映射到“多态联合”，而不是在 `Mapper.with_polymorphic` 参数中本地指定它。

为了帮助实现这一点，声明性提供了一个名为 `AbstractConcreteBase` 的 `ConcreteBase` 类的变体，它可以自动实现这一点：

```py
from sqlalchemy.ext.declarative import AbstractConcreteBase
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    pass

class Employee(AbstractConcreteBase, Base):
    strict_attrs = True

    name = mapped_column(String(50))

class Manager(Employee):
    __tablename__ = "manager"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    manager_data = mapped_column(String(40))

    __mapper_args__ = {
        "polymorphic_identity": "manager",
        "concrete": True,
    }

class Engineer(Employee):
    __tablename__ = "engineer"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    engineer_info = mapped_column(String(40))

    __mapper_args__ = {
        "polymorphic_identity": "engineer",
        "concrete": True,
    }

Base.registry.configure()
```

在上面的代码中，调用了 `registry.configure()` 方法，这将触发 `Employee` 类实际上被映射；在配置步骤之前，由于尚未定义将从中查询的子表，因此该类没有映射。这个过程比 `ConcreteBase` 更复杂，因为整个基类的映射必须延迟到所有子类都声明完毕。有了上述的映射，只有 `Manager` 和 `Engineer` 的实例才能被持久化；对 `Employee` 类进行查询将始终产生 `Manager` 和 `Engineer` 对象。

使用上述的映射，可以根据 `Employee` 类和在其上局部声明的任何属性生成查询，例如 `Employee.name`：

```py
>>> stmt = select(Employee).where(Employee.name == "n1")
>>> print(stmt)
SELECT  pjoin.id,  pjoin.name,  pjoin.type,  pjoin.manager_data,  pjoin.engineer_info
FROM  (
  SELECT  engineer.id  AS  id,  engineer.name  AS  name,  engineer.engineer_info  AS  engineer_info,
  CAST(NULL  AS  VARCHAR(40))  AS  manager_data,  'engineer'  AS  type
  FROM  engineer
  UNION  ALL
  SELECT  manager.id  AS  id,  manager.name  AS  name,  CAST(NULL  AS  VARCHAR(40))  AS  engineer_info,
  manager.manager_data  AS  manager_data,  'manager'  AS  type
  FROM  manager
)  AS  pjoin
WHERE  pjoin.name  =  :name_1 
```

`AbstractConcreteBase.strict_attrs` 参数表示 `Employee` 类应直接映射仅属于 `Employee` 类本身的属性，例如 `Employee.name` 属性。其他属性如 `Manager.manager_data` 和 `Engineer.engineer_info` 仅存在于它们各自的子类中。当未设置 `AbstractConcreteBase.strict_attrs` 时，所有子类的属性如 `Manager.manager_data` 和 `Engineer.engineer_info` 都会映射到基类 `Employee` 中。这是一种遗留模式，对于查询可能更方便，但会导致所有子类共享整个层次结构的完整属性集；在上述示例中，如果不使用 `AbstractConcreteBase.strict_attrs`，将会生成无用的 `Engineer.manager_name` 和 `Manager.engineer_info` 属性。

新版本 2.0 中：增加了 `AbstractConcreteBase.strict_attrs` 参数到 `AbstractConcreteBase`，以生成更清晰的映射；默认值为 False，以允许遗留映射在 1.x 版本中继续正常工作。

另请参阅

`AbstractConcreteBase`

### 经典和半经典具体多态配置

通过`ConcreteBase`和`AbstractConcreteBase`说明的声明性配置等同于另外两种明确使用`polymorphic_union()`的配置形式。这些配置形式明确使用`Table`对象，以便首先创建“多态联合”，然后应用到映射中。这些例子旨在澄清`polymorphic_union()`函数在映射中的作用。

例如，**半经典映射**利用了声明性，但是单独建立了`Table`对象：

```py
metadata_obj = Base.metadata

employees_table = Table(
    "employee",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    Column("name", String(50)),
)

managers_table = Table(
    "manager",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    Column("name", String(50)),
    Column("manager_data", String(50)),
)

engineers_table = Table(
    "engineer",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    Column("name", String(50)),
    Column("engineer_info", String(50)),
)
```

接下来，使用`polymorphic_union()`生成 UNION：

```py
from sqlalchemy.orm import polymorphic_union

pjoin = polymorphic_union(
    {
        "employee": employees_table,
        "manager": managers_table,
        "engineer": engineers_table,
    },
    "type",
    "pjoin",
)
```

使用上述`Table`对象，可以使用“半经典”风格生成映射，在这种风格中，我们将声明性与`__table__`参数结合使用；我们上面的多态联合通过`__mapper_args__`传递给`Mapper.with_polymorphic`参数：

```py
class Employee(Base):
    __table__ = employee_table
    __mapper_args__ = {
        "polymorphic_on": pjoin.c.type,
        "with_polymorphic": ("*", pjoin),
        "polymorphic_identity": "employee",
    }

class Engineer(Employee):
    __table__ = engineer_table
    __mapper_args__ = {
        "polymorphic_identity": "engineer",
        "concrete": True,
    }

class Manager(Employee):
    __table__ = manager_table
    __mapper_args__ = {
        "polymorphic_identity": "manager",
        "concrete": True,
    }
```

或者，可以完全以“经典”风格使用相同的`Table`对象，而不使用声明性。与声明性提供的类似的构造函数如下所示：

```py
class Employee:
    def __init__(self, **kw):
        for k in kw:
            setattr(self, k, kw[k])

class Manager(Employee):
    pass

class Engineer(Employee):
    pass

employee_mapper = mapper_registry.map_imperatively(
    Employee,
    pjoin,
    with_polymorphic=("*", pjoin),
    polymorphic_on=pjoin.c.type,
)
manager_mapper = mapper_registry.map_imperatively(
    Manager,
    managers_table,
    inherits=employee_mapper,
    concrete=True,
    polymorphic_identity="manager",
)
engineer_mapper = mapper_registry.map_imperatively(
    Engineer,
    engineers_table,
    inherits=employee_mapper,
    concrete=True,
    polymorphic_identity="engineer",
)
```

“抽象”示例也可以使用“半经典”或“经典”风格进行映射。不同之处在于，我们将“多态联合”应用于`Mapper.with_polymorphic`参数的方式，而是将其直接应用于我们基本映射器上的映射可选项。半经典映射如下所示：

```py
from sqlalchemy.orm import polymorphic_union

pjoin = polymorphic_union(
    {
        "manager": managers_table,
        "engineer": engineers_table,
    },
    "type",
    "pjoin",
)

class Employee(Base):
    __table__ = pjoin
    __mapper_args__ = {
        "polymorphic_on": pjoin.c.type,
        "with_polymorphic": "*",
        "polymorphic_identity": "employee",
    }

class Engineer(Employee):
    __table__ = engineer_table
    __mapper_args__ = {
        "polymorphic_identity": "engineer",
        "concrete": True,
    }

class Manager(Employee):
    __table__ = manager_table
    __mapper_args__ = {
        "polymorphic_identity": "manager",
        "concrete": True,
    }
```

在上面的例子中，我们像以前一样使用`polymorphic_union()`，只是省略了`employee`表。

另请参阅

命令式映射 - 命令式或“经典”映射的背景信息

### 具体继承的关系

在具体继承的情况下，映射关系是具有挑战性的，因为不同的类不共享表格。如果关系仅涉及特定类，例如在我们先前的示例中`Company`和`Manager`之间的关系，那么不需要特殊步骤，因为这只是两个相关的表。

但是，如果`Company`要对`Employee`有一对多的关系，表明集合可能包含`Engineer`和`Manager`对象，那么这意味着`Employee`必须具有多态加载功能，并且要与`company`表关联的每个表都必须有一个外键。这种配置的示例如下：

```py
from sqlalchemy.ext.declarative import ConcreteBase

class Company(Base):
    __tablename__ = "company"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    employees = relationship("Employee")

class Employee(ConcreteBase, Base):
    __tablename__ = "employee"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    company_id = mapped_column(ForeignKey("company.id"))

    __mapper_args__ = {
        "polymorphic_identity": "employee",
        "concrete": True,
    }

class Manager(Employee):
    __tablename__ = "manager"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    manager_data = mapped_column(String(40))
    company_id = mapped_column(ForeignKey("company.id"))

    __mapper_args__ = {
        "polymorphic_identity": "manager",
        "concrete": True,
    }

class Engineer(Employee):
    __tablename__ = "engineer"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    engineer_info = mapped_column(String(40))
    company_id = mapped_column(ForeignKey("company.id"))

    __mapper_args__ = {
        "polymorphic_identity": "engineer",
        "concrete": True,
    }
```

具体继承和关系的下一个复杂性涉及当我们希望`Employee`、`Manager`和`Engineer`中的一个或全部自己参考`Company`时。对于这种情况，SQLAlchemy 具有特殊行为，即在`Employee`上放置到`Company`的`relationship()`在实例级别时**不适用**于`Manager`和`Engineer`类。相反，必须对每个类应用不同的`relationship()`。为了在三个单独的关系中实现与`Company.employees`相反的双向行为，使用了`relationship.back_populates`参数：

```py
from sqlalchemy.ext.declarative import ConcreteBase

class Company(Base):
    __tablename__ = "company"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    employees = relationship("Employee", back_populates="company")

class Employee(ConcreteBase, Base):
    __tablename__ = "employee"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    company_id = mapped_column(ForeignKey("company.id"))
    company = relationship("Company", back_populates="employees")

    __mapper_args__ = {
        "polymorphic_identity": "employee",
        "concrete": True,
    }

class Manager(Employee):
    __tablename__ = "manager"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    manager_data = mapped_column(String(40))
    company_id = mapped_column(ForeignKey("company.id"))
    company = relationship("Company", back_populates="employees")

    __mapper_args__ = {
        "polymorphic_identity": "manager",
        "concrete": True,
    }

class Engineer(Employee):
    __tablename__ = "engineer"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    engineer_info = mapped_column(String(40))
    company_id = mapped_column(ForeignKey("company.id"))
    company = relationship("Company", back_populates="employees")

    __mapper_args__ = {
        "polymorphic_identity": "engineer",
        "concrete": True,
    }
```

上述限制与当前的实现相关，其中具体继承类不共享超类的任何属性，因此需要设置不同的关系。

### 加载具体继承映射

具体继承的加载选项有限；一般来说，如果使用声明性具体混合类型之一在映射器上配置多态加载，那么在当前 SQLAlchemy 版本中无法在查询时修改它。通常，`with_polymorphic()`函数应该能够覆盖具体使用的加载方式，但由于当前的限制，这尚不受支持。

### 具体多态加载配置

具有具体继承的多态加载要求针对应该具有多态加载的每个基类配置一个专用的 SELECT。这个 SELECT 需要能够单独访问所有映射的表，并且通常是使用 SQLAlchemy 助手`polymorphic_union()`构造的 UNION 语句。

如为继承映射编写 SELECT 语句所讨论的，任何类型的映射器继承配置都可以使用`Mapper.with_polymorphic`参数默认配置从特殊的可选项加载。当前的公共 API 要求在首次构造`Mapper`时设置此参数。

但是，在声明式编程中，映射器和被映射的`Table`同时创建，即在定义映射类的时候。这意味着`Mapper.with_polymorphic`参数还不能提供，因为对应于子类的`Table`对象尚未定义。

有几种策略可用于解决这种循环，然而，声明式提供了处理此问题的助手类`ConcreteBase`和`AbstractConcreteBase`。

使用`ConcreteBase`，我们可以几乎与其他形式的继承映射方式相同地设置我们的具体映射：

```py
from sqlalchemy.ext.declarative import ConcreteBase
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    pass

class Employee(ConcreteBase, Base):
    __tablename__ = "employee"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))

    __mapper_args__ = {
        "polymorphic_identity": "employee",
        "concrete": True,
    }

class Manager(Employee):
    __tablename__ = "manager"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    manager_data = mapped_column(String(40))

    __mapper_args__ = {
        "polymorphic_identity": "manager",
        "concrete": True,
    }

class Engineer(Employee):
    __tablename__ = "engineer"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    engineer_info = mapped_column(String(40))

    __mapper_args__ = {
        "polymorphic_identity": "engineer",
        "concrete": True,
    }
```

在上述情况下，声明式在映射器“初始化”时为`Employee`类设置多态可选项；这是为解析其他依赖映射器而进行的映射器的后期配置步骤。`ConcreteBase`助手使用`polymorphic_union()`函数在设置完所有其他类之后创建所有具体映射表的联合，然后使用已经存在的基类映射器配置此语句。

在选择时，多态联合产生类似这样的查询：

```py
session.scalars(select(Employee)).all()
SELECT
  pjoin.id,
  pjoin.name,
  pjoin.type,
  pjoin.manager_data,
  pjoin.engineer_info
FROM  (
  SELECT
  employee.id  AS  id,
  employee.name  AS  name,
  CAST(NULL  AS  VARCHAR(40))  AS  manager_data,
  CAST(NULL  AS  VARCHAR(40))  AS  engineer_info,
  'employee'  AS  type
  FROM  employee
  UNION  ALL
  SELECT
  manager.id  AS  id,
  manager.name  AS  name,
  manager.manager_data  AS  manager_data,
  CAST(NULL  AS  VARCHAR(40))  AS  engineer_info,
  'manager'  AS  type
  FROM  manager
  UNION  ALL
  SELECT
  engineer.id  AS  id,
  engineer.name  AS  name,
  CAST(NULL  AS  VARCHAR(40))  AS  manager_data,
  engineer.engineer_info  AS  engineer_info,
  'engineer'  AS  type
  FROM  engineer
)  AS  pjoin 
```

上述的 UNION 查询需要为每个子表制造“NULL”列，以适应那些不是特定子类成员的列。

另请参见

`ConcreteBase`

### 抽象具体类

到目前为止，所示的具体映射显示了子类以及基类映射到单独的表中。在具体继承用例中，常见的情况是基类在数据库中不表示，只有子类。换句话说，基类是“抽象的”。

通常，当一个人想要将两个不同的子类映射到单独的表中，并且保留基类未映射时，这可以非常容易地实现。当使用声明式时，只需使用`__abstract__`指示符声明基类：

```py
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    pass

class Employee(Base):
    __abstract__ = True

class Manager(Employee):
    __tablename__ = "manager"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    manager_data = mapped_column(String(40))

class Engineer(Employee):
    __tablename__ = "engineer"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    engineer_info = mapped_column(String(40))
```

在上述代码中，我们实际上没有使用 SQLAlchemy 的继承映射功能；我们可以正常加载和持久化`Manager`和`Engineer`的实例。然而，当我们需要进行**多态查询**时，情况就会发生变化，也就是说，我们希望发出`select(Employee)`并返回一组`Manager`和`Engineer`实例。这将我们带回到具体继承的领域，我们必须构建一个针对`Employee`的特殊映射器才能实现这一点。

要修改我们的具体继承示例，以说明能够进行多态加载的“抽象”基类，我们将只有一个`engineer`和一个`manager`表，而没有`employee`表，但`Employee`映射器将直接映射到“多态联合”，而不是将其局部指定给`Mapper.with_polymorphic`参数。

为了帮助解决这个问题，Declarative 提供了一种名为`AbstractConcreteBase`的`ConcreteBase`类的变体，可以自动实现这一点：

```py
from sqlalchemy.ext.declarative import AbstractConcreteBase
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    pass

class Employee(AbstractConcreteBase, Base):
    strict_attrs = True

    name = mapped_column(String(50))

class Manager(Employee):
    __tablename__ = "manager"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    manager_data = mapped_column(String(40))

    __mapper_args__ = {
        "polymorphic_identity": "manager",
        "concrete": True,
    }

class Engineer(Employee):
    __tablename__ = "engineer"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    engineer_info = mapped_column(String(40))

    __mapper_args__ = {
        "polymorphic_identity": "engineer",
        "concrete": True,
    }

Base.registry.configure()
```

在上面的代码中，调用了`registry.configure()`方法，这将触发`Employee`类实际映射；在配置步骤之前，该类没有映射，因为它将从中查询的子表尚未定义。这个过程比`ConcreteBase`的过程更复杂，因为必须延迟基类的整个映射，直到所有的子类都已声明。通过像上面这样的映射，只能持久化`Manager`和`Engineer`的实例；对`Employee`类进行查询将始终生成`Manager`和`Engineer`对象。

使用上述映射，可以按照`Employee`类和在其上本地声明的任何属性生成查询，例如`Employee.name`：

```py
>>> stmt = select(Employee).where(Employee.name == "n1")
>>> print(stmt)
SELECT  pjoin.id,  pjoin.name,  pjoin.type,  pjoin.manager_data,  pjoin.engineer_info
FROM  (
  SELECT  engineer.id  AS  id,  engineer.name  AS  name,  engineer.engineer_info  AS  engineer_info,
  CAST(NULL  AS  VARCHAR(40))  AS  manager_data,  'engineer'  AS  type
  FROM  engineer
  UNION  ALL
  SELECT  manager.id  AS  id,  manager.name  AS  name,  CAST(NULL  AS  VARCHAR(40))  AS  engineer_info,
  manager.manager_data  AS  manager_data,  'manager'  AS  type
  FROM  manager
)  AS  pjoin
WHERE  pjoin.name  =  :name_1 
```

`AbstractConcreteBase.strict_attrs` 参数指示 `Employee` 类应直接映射仅属于 `Employee` 类的本地属性，即 `Employee.name` 属性。其他属性，如 `Manager.manager_data` 和 `Engineer.engineer_info`，仅存在于其对应的子类中。当未设置 `AbstractConcreteBase.strict_attrs` 时，所有子类属性，如 `Manager.manager_data` 和 `Engineer.engineer_info`，都映射到基类 `Employee`。这是一种传统的使用模式，可能更方便查询，但其效果是所有子类共享整个层次结构的完整属性集；在上述示例中，不使用 `AbstractConcreteBase.strict_attrs` 将导致生成不必要的 `Engineer.manager_name` 和 `Manager.engineer_info` 属性。

2.0 版本新增：增加了 `AbstractConcreteBase.strict_attrs` 参数到 `AbstractConcreteBase` 中，以产生更清晰的映射；默认值为 False，以允许传统映射继续像 1.x 版本中那样工作。

另请参阅

`AbstractConcreteBase`

### 经典和半经典具有多态性的具体配置

用`ConcreteBase`和`AbstractConcreteBase`说明的声明性配置等同于另外两种明确使用`polymorphic_union()`的配置形式。这些配置形式明确使用`Table`对象，以便首先创建“多态联合”，然后应用于映射。这里举例说明了这些配置形式，以阐明`polymorphic_union()`函数在映射方面的作用。

一个**半经典映射**的例子利用了声明性，但是单独建立了`Table`对象：

```py
metadata_obj = Base.metadata

employees_table = Table(
    "employee",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    Column("name", String(50)),
)

managers_table = Table(
    "manager",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    Column("name", String(50)),
    Column("manager_data", String(50)),
)

engineers_table = Table(
    "engineer",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    Column("name", String(50)),
    Column("engineer_info", String(50)),
)
```

接下来，使用`polymorphic_union()`生成 UNION：

```py
from sqlalchemy.orm import polymorphic_union

pjoin = polymorphic_union(
    {
        "employee": employees_table,
        "manager": managers_table,
        "engineer": engineers_table,
    },
    "type",
    "pjoin",
)
```

使用上述`Table`对象，可以使用“半经典”风格生成映射，其中我们与`__table__`参数一起使用声明性；我们上面的多态联合通过`__mapper_args__`传递给`Mapper.with_polymorphic`参数：

```py
class Employee(Base):
    __table__ = employee_table
    __mapper_args__ = {
        "polymorphic_on": pjoin.c.type,
        "with_polymorphic": ("*", pjoin),
        "polymorphic_identity": "employee",
    }

class Engineer(Employee):
    __table__ = engineer_table
    __mapper_args__ = {
        "polymorphic_identity": "engineer",
        "concrete": True,
    }

class Manager(Employee):
    __table__ = manager_table
    __mapper_args__ = {
        "polymorphic_identity": "manager",
        "concrete": True,
    }
```

或者，完全可以使用完全“经典”风格，而根本不使用声明性，使用与声明性提供的类似的构造函数：

```py
class Employee:
    def __init__(self, **kw):
        for k in kw:
            setattr(self, k, kw[k])

class Manager(Employee):
    pass

class Engineer(Employee):
    pass

employee_mapper = mapper_registry.map_imperatively(
    Employee,
    pjoin,
    with_polymorphic=("*", pjoin),
    polymorphic_on=pjoin.c.type,
)
manager_mapper = mapper_registry.map_imperatively(
    Manager,
    managers_table,
    inherits=employee_mapper,
    concrete=True,
    polymorphic_identity="manager",
)
engineer_mapper = mapper_registry.map_imperatively(
    Engineer,
    engineers_table,
    inherits=employee_mapper,
    concrete=True,
    polymorphic_identity="engineer",
)
```

“抽象”示例也可以使用“半经典”或“经典”风格进行映射。不同之处在于，我们不是将“多态联合”应用于`Mapper.with_polymorphic`参数，而是直接将其应用为我们最基本的映射器上的映射选择器。下面是半经典映射的示例：

```py
from sqlalchemy.orm import polymorphic_union

pjoin = polymorphic_union(
    {
        "manager": managers_table,
        "engineer": engineers_table,
    },
    "type",
    "pjoin",
)

class Employee(Base):
    __table__ = pjoin
    __mapper_args__ = {
        "polymorphic_on": pjoin.c.type,
        "with_polymorphic": "*",
        "polymorphic_identity": "employee",
    }

class Engineer(Employee):
    __table__ = engineer_table
    __mapper_args__ = {
        "polymorphic_identity": "engineer",
        "concrete": True,
    }

class Manager(Employee):
    __table__ = manager_table
    __mapper_args__ = {
        "polymorphic_identity": "manager",
        "concrete": True,
    }
```

在上面的示例中，我们像以前一样使用`polymorphic_union()`，只是省略了`employee`表。

另请参见

命令式映射 - 关于命令式或“经典”映射的背景信息

### 具体继承关系

在具体继承的情况下，映射关系是具有挑战性的，因为不同的类不共享一个表。如果关系只涉及特定的类，比如我们之前示例中的 `Company` 和 `Manager` 之间的关系，那么不需要特殊步骤，因为这只是两个相关的表。

然而，如果 `Company` 要与 `Employee` 有一对多的关系，表明集合可能包括 `Engineer` 和 `Manager` 对象，这意味着 `Employee` 必须具有多态加载能力，并且每个相关的表都必须有一个外键返回到 `company` 表。这样的配置示例如下：

```py
from sqlalchemy.ext.declarative import ConcreteBase

class Company(Base):
    __tablename__ = "company"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    employees = relationship("Employee")

class Employee(ConcreteBase, Base):
    __tablename__ = "employee"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    company_id = mapped_column(ForeignKey("company.id"))

    __mapper_args__ = {
        "polymorphic_identity": "employee",
        "concrete": True,
    }

class Manager(Employee):
    __tablename__ = "manager"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    manager_data = mapped_column(String(40))
    company_id = mapped_column(ForeignKey("company.id"))

    __mapper_args__ = {
        "polymorphic_identity": "manager",
        "concrete": True,
    }

class Engineer(Employee):
    __tablename__ = "engineer"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    engineer_info = mapped_column(String(40))
    company_id = mapped_column(ForeignKey("company.id"))

    __mapper_args__ = {
        "polymorphic_identity": "engineer",
        "concrete": True,
    }
```

具体继承和关系的下一个复杂性涉及当我们希望 `Employee`、`Manager` 和 `Engineer` 中的一个或全部自身参考 `Company` 时。对于这种情况，SQLAlchemy 在 `Employee` 上有特殊的行为，即一个链接到 `Company` 的 `relationship()` 放置在 `Employee` 上，当在实例级别执行时，**不适用**于 `Manager` 和 `Engineer` 类。相反，必须对每个类应用不同的 `relationship()`。为了实现作为 `Company.employees` 的相反的三个单独关系的双向行为，使用了 `relationship.back_populates` 参数：

```py
from sqlalchemy.ext.declarative import ConcreteBase

class Company(Base):
    __tablename__ = "company"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    employees = relationship("Employee", back_populates="company")

class Employee(ConcreteBase, Base):
    __tablename__ = "employee"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    company_id = mapped_column(ForeignKey("company.id"))
    company = relationship("Company", back_populates="employees")

    __mapper_args__ = {
        "polymorphic_identity": "employee",
        "concrete": True,
    }

class Manager(Employee):
    __tablename__ = "manager"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    manager_data = mapped_column(String(40))
    company_id = mapped_column(ForeignKey("company.id"))
    company = relationship("Company", back_populates="employees")

    __mapper_args__ = {
        "polymorphic_identity": "manager",
        "concrete": True,
    }

class Engineer(Employee):
    __tablename__ = "engineer"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    engineer_info = mapped_column(String(40))
    company_id = mapped_column(ForeignKey("company.id"))
    company = relationship("Company", back_populates="employees")

    __mapper_args__ = {
        "polymorphic_identity": "engineer",
        "concrete": True,
    }
```

上述限制与当前实现相关，包括具体继承类不共享超类的任何属性，因此需要设置不同的关系。

### 加载具体继承映射

具体继承加载选项有限；一般来说，如果在映射器上配置了多态加载，使用其中一个声明性的具体混合类，那么在当前的 SQLAlchemy 版本中它就不能在查询时进行修改。通常，`with_polymorphic()` 函数应该能够覆盖具体加载使用的样式，然而由于当前的限制，这还不被支持。
