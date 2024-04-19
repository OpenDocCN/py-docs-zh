# 编写继承映射的 SELECT 语句

> 原文：[`docs.sqlalchemy.org/en/20/orm/queryguide/inheritance.html`](https://docs.sqlalchemy.org/en/20/orm/queryguide/inheritance.html)

关于本文档

本节利用了使用 ORM 继承 功能配置的 ORM 映射，描述在 映射类继承层次结构 中。重点将放在 连接表继承，因为这是最复杂的 ORM 查询情况。

查看此页面的 ORM 设置。

## 从基类 vs. 特定子类进行 SELECT

构建在连接继承层次结构中的类上的 SELECT 语句将针对将类映射到的表以及任何现有的超级表进行查询，并使用 JOIN 将它们链接在一起。然后，查询将返回请求类型的对象以及请求类型的任何子类型，使用每行中的 鉴别器 值来确定正确的类型。下面的查询是针对 `Employee` 的 `Manager` 子类建立的，然后返回的结果将只包含 `Manager` 类型的对象：

```py
>>> from sqlalchemy import select
>>> stmt = select(Manager).order_by(Manager.id)
>>> managers = session.scalars(stmt).all()
BEGIN  (implicit)
SELECT  manager.id,  employee.id  AS  id_1,  employee.name,  employee.type,  employee.company_id,  manager.manager_name
FROM  employee  JOIN  manager  ON  employee.id  =  manager.id  ORDER  BY  manager.id
[...]  ()
>>> print(managers)
[Manager('Mr. Krabs')]
```

当 SELECT 语句针对继承层次结构中的基类时，默认行为是仅将该类的表包括在渲染的 SQL 中，并且不使用 JOIN。与所有情况一样，鉴别器 列用于区分不同的请求子类型，然后返回任何可能的子类型的对象。返回的对象将具有对应于基表的属性填充，对应于子表的属性将以未加载状态开始，在访问时自动加载。子属性的加载可配置为以多种方式更“急切”，在本节后面讨论。

下面的示例创建了针对 `Employee` 超类的查询。这表示结果集中可能包含任何类型的对象，包括 `Manager`、`Engineer` 和 `Employee`：

```py
>>> from sqlalchemy import select
>>> stmt = select(Employee).order_by(Employee.id)
>>> objects = session.scalars(stmt).all()
BEGIN  (implicit)
SELECT  employee.id,  employee.name,  employee.type,  employee.company_id
FROM  employee  ORDER  BY  employee.id
[...]  ()
>>> print(objects)
[Manager('Mr. Krabs'), Engineer('SpongeBob'), Engineer('Squidward')]
```

在上面的例子中，未包括 `Manager` 和 `Engineer` 的附加表在 SELECT 中，这意味着返回的对象尚未包含来自那些表中表示的数据，在本例中是 `Manager` 类的 `.manager_name` 属性以及 `Engineer` 类的 `.engineer_info` 属性。这些属性起始状态为 过期，当首次使用 延迟加载 访问时，它们将自动填充：

```py
>>> mr_krabs = objects[0]
>>> print(mr_krabs.manager_name)
SELECT  manager.manager_name  AS  manager_manager_name
FROM  manager
WHERE  ?  =  manager.id
[...]  (1,)
Eugene H. Krabs
```

如果加载了大量对象，则此惰性加载行为并不理想，因为消费应用程序将需要访问特定于子类的属性，这将是 N 加一问题的一个例子，每行都会发出额外的 SQL。这些额外的 SQL 可能会影响性能，并且也与使用 asyncio 等方法不兼容。此外，在我们对`Employee`对象的查询中，由于查询仅针对基本表，我们无法添加涉及特定于子类的属性（如`Manager`或`Engineer`）的 SQL 条件。接下来的两个部分详细介绍了两种不同方式解决这两个问题的构造，`selectin_polymorphic()`加载器选项和`with_polymorphic()`实体构造。

## 使用 selectin_polymorphic()

要解决在访问子类属性时的性能问题，可以使用`selectin_polymorphic()`加载策略，以便一次性急切地加载这些附加属性。此加载选项的工作方式类似于`selectinload()`关系加载策略，对于在层次结构中加载的对象，会对每个子表发出额外的 SELECT 语句，使用`IN`来根据主键查询附加行。

`selectinload()`接受作为其参数的基本实体，该实体正在被查询，然后是该实体的子类序列，对于这些子类，应为传入的行加载其特定属性：

```py
>>> from sqlalchemy.orm import selectin_polymorphic
>>> loader_opt = selectin_polymorphic(Employee, [Manager, Engineer])
```

然后，使用`selectin_polymorphic()`构造作为加载器选项，将其传递给`Select.options()`方法的`Select`。该示例说明了如何使用`selectin_polymorphic()`来急切加载`Manager`和`Engineer`子类的本地列：

```py
>>> from sqlalchemy.orm import selectin_polymorphic
>>> loader_opt = selectin_polymorphic(Employee, [Manager, Engineer])
>>> stmt = select(Employee).order_by(Employee.id).options(loader_opt)
>>> objects = session.scalars(stmt).all()
BEGIN  (implicit)
SELECT  employee.id,  employee.name,  employee.type,  employee.company_id
FROM  employee  ORDER  BY  employee.id
[...]  ()
SELECT  manager.id  AS  manager_id,  employee.id  AS  employee_id,
employee.type  AS  employee_type,  manager.manager_name  AS  manager_manager_name
FROM  employee  JOIN  manager  ON  employee.id  =  manager.id
WHERE  employee.id  IN  (?)  ORDER  BY  employee.id
[...]  (1,)
SELECT  engineer.id  AS  engineer_id,  employee.id  AS  employee_id,
employee.type  AS  employee_type,  engineer.engineer_info  AS  engineer_engineer_info
FROM  employee  JOIN  engineer  ON  employee.id  =  engineer.id
WHERE  employee.id  IN  (?,  ?)  ORDER  BY  employee.id
[...]  (2,  3)
>>> print(objects)
[Manager('Mr. Krabs'), Engineer('SpongeBob'), Engineer('Squidward')]
```

上述示例说明了发出两个额外的 SELECT 语句，以便急切地获取附加属性，例如`Engineer.engineer_info`和`Manager.manager_name`。现在，我们可以在加载的对象上访问这些子属性，而无需发出任何额外的 SQL 语句：

```py
>>> print(objects[0].manager_name)
Eugene H. Krabs
```

提示

`selectin_polymorphic()` 加载选项尚未针对基础 `employee` 表的情况进行优化，因此在第二个和第三个“急加载”查询中不需要包含 `employee` 表；因此在上面的示例中，我们看到从 `employee` 到 `manager` 和 `engineer` 的 JOIN，即使 `employee` 的列已经加载。这与 `selectinload()` 关系策略形成对比，在这方面更为复杂，并且在不需要时可以排除 JOIN。

### 对现有急加载应用 selectin_polymorphic()

除了将 `selectin_polymorphic()` 指定为由语句加载的顶级实体的选项外，我们还可以指示在现有加载目标上应用 `selectin_polymorphic()`。由于我们的设置映射包含一个父 `Company` 实体，其具有引用 `Employee` 实体的 `Company.employees` `relationship()`，我们可以说明针对 `Company` 实体的 SELECT，它急加载所有 `Employee` 对象以及其子类型的所有属性，如下所示，通过将 `Load.selectin_polymorphic()` 作为链式加载器选项应用；在此形式中，第一个参数是从前一个加载器选项隐式获取的（在本例中为 `selectinload()`），因此我们仅指示要加载的附加目标子类：

```py
>>> from sqlalchemy.orm import selectinload
>>> stmt = select(Company).options(
...     selectinload(Company.employees).selectin_polymorphic([Manager, Engineer])
... )
>>> for company in session.scalars(stmt):
...     print(f"company: {company.name}")
...     print(f"employees: {company.employees}")
BEGIN  (implicit)
SELECT  company.id,  company.name
FROM  company
[...]  ()
SELECT  employee.company_id  AS  employee_company_id,  employee.id  AS  employee_id,
employee.name  AS  employee_name,  employee.type  AS  employee_type
FROM  employee
WHERE  employee.company_id  IN  (?)
[...]  (1,)
SELECT  manager.id  AS  manager_id,  employee.id  AS  employee_id,
employee.type  AS  employee_type,
manager.manager_name  AS  manager_manager_name
FROM  employee  JOIN  manager  ON  employee.id  =  manager.id
WHERE  employee.id  IN  (?)  ORDER  BY  employee.id
[...]  (1,)
SELECT  engineer.id  AS  engineer_id,  employee.id  AS  employee_id,
employee.type  AS  employee_type,
engineer.engineer_info  AS  engineer_engineer_info
FROM  employee  JOIN  engineer  ON  employee.id  =  engineer.id
WHERE  employee.id  IN  (?,  ?)  ORDER  BY  employee.id
[...]  (2,  3)
company: Krusty Krab
employees: [Manager('Mr. Krabs'), Engineer('SpongeBob'), Engineer('Squidward')]
```

另请参见

多态子类型的急加载 - 演示了使用 `with_polymorphic()` 而不是上述等效示例的示例  ### 将加载选项应用于由 selectin_polymorphic 加载的子类

`selectin_polymorphic()` 发出的 SELECT 语句本身是 ORM 语句，因此我们还可以添加其他加载选项（例如文档中记录的那些位于 关系加载技术） ，这些选项引用特定的子类。这些选项应该作为**同级**应用于 `selectin_polymorphic()` 选项，即在 `select.options()` 内用逗号分隔。

例如，如果我们考虑`Manager`映射器与名为`Paperwork`的实体之间有一对多关系，我们可以结合使用`selectin_polymorphic()`和`selectinload()`来急加载所有`Manager`对象上的这个集合，其中`Manager`对象的子属性也会被急加载：

```py
>>> from sqlalchemy.orm import selectin_polymorphic
>>> stmt = (
...     select(Employee)
...     .order_by(Employee.id)
...     .options(
...         selectin_polymorphic(Employee, [Manager, Engineer]),
...         selectinload(Manager.paperwork),
...     )
... )
>>> objects = session.scalars(stmt).all()
SELECT  employee.id,  employee.name,  employee.type,  employee.company_id
FROM  employee  ORDER  BY  employee.id
[...]  ()
SELECT  manager.id  AS  manager_id,  employee.id  AS  employee_id,  employee.type  AS  employee_type,  manager.manager_name  AS  manager_manager_name
FROM  employee  JOIN  manager  ON  employee.id  =  manager.id
WHERE  employee.id  IN  (?)  ORDER  BY  employee.id
[...]  (1,)
SELECT  paperwork.manager_id  AS  paperwork_manager_id,  paperwork.id  AS  paperwork_id,  paperwork.document_name  AS  paperwork_document_name
FROM  paperwork
WHERE  paperwork.manager_id  IN  (?)
[...]  (1,)
SELECT  engineer.id  AS  engineer_id,  employee.id  AS  employee_id,  employee.type  AS  employee_type,  engineer.engineer_info  AS  engineer_engineer_info
FROM  employee  JOIN  engineer  ON  employee.id  =  engineer.id
WHERE  employee.id  IN  (?,  ?)  ORDER  BY  employee.id
[...]  (2,  3)
>>> print(objects[0])
Manager('Mr. Krabs')
>>> print(objects[0].paperwork)
[Paperwork('Secret Recipes'), Paperwork('Krabby Patty Orders')]
```

#### 当选择`selectin_polymorphic`本身作为子选项时应用加载器选项

版本 2.0.21 中新增。

前面的章节介绍了`selectin_polymorphic()`和`selectinload()`作为兄弟选项使用的示例，两者都在单个调用`select.options()`内使用。 如果目标实体已经从父关系加载，例如在示例将`selectin_polymorphic()`应用于现有的急加载中，我们可以使用`Load.options()`方法应用此“兄弟”模式，该方法将子选项应用于父级，如在使用`Load.options()`指定子选项中所示。 下面我们结合这两个示例，加载`Company.employees`，同时加载`Manager`和`Engineer`类的属性，以及急加载`Manager.paperwork`属性：

```py
>>> from sqlalchemy.orm import selectinload
>>> stmt = select(Company).options(
...     selectinload(Company.employees).options(
...         selectin_polymorphic(Employee, [Manager, Engineer]),
...         selectinload(Manager.paperwork),
...     )
... )
>>> for company in session.scalars(stmt):
...     print(f"company: {company.name}")
...     for employee in company.employees:
...         if isinstance(employee, Manager):
...             print(f"manager: {employee.name} paperwork: {employee.paperwork}")
BEGIN  (implicit)
SELECT  company.id,  company.name
FROM  company
[...]  ()
SELECT  employee.company_id  AS  employee_company_id,  employee.id  AS  employee_id,  employee.name  AS  employee_name,  employee.type  AS  employee_type
FROM  employee
WHERE  employee.company_id  IN  (?)
[...]  (1,)
SELECT  manager.id  AS  manager_id,  employee.id  AS  employee_id,  employee.type  AS  employee_type,  manager.manager_name  AS  manager_manager_name
FROM  employee  JOIN  manager  ON  employee.id  =  manager.id
WHERE  employee.id  IN  (?)  ORDER  BY  employee.id
[...]  (1,)
SELECT  paperwork.manager_id  AS  paperwork_manager_id,  paperwork.id  AS  paperwork_id,  paperwork.document_name  AS  paperwork_document_name
FROM  paperwork
WHERE  paperwork.manager_id  IN  (?)
[...]  (1,)
SELECT  engineer.id  AS  engineer_id,  employee.id  AS  employee_id,  employee.type  AS  employee_type,  engineer.engineer_info  AS  engineer_engineer_info
FROM  employee  JOIN  engineer  ON  employee.id  =  engineer.id
WHERE  employee.id  IN  (?,  ?)  ORDER  BY  employee.id
[...]  (2,  3)
company: Krusty Krab
manager: Mr. Krabs paperwork: [Paperwork('Secret Recipes'), Paperwork('Krabby Patty Orders')]
```

### 配置 mappers 上的 `selectin_polymorphic()`

可以在特定的 mappers 上配置`selectin_polymorphic()`的行为，以便默认生效，通过使用`Mapper.polymorphic_load`参数，在每个子类基础上使用值`"selectin"`。 下面的示例演示了在 `Engineer` 和 `Manager` 子类中使用此参数的情况：

```py
class Employee(Base):
    __tablename__ = "employee"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    type = mapped_column(String(50))

    __mapper_args__ = {"polymorphic_identity": "employee", "polymorphic_on": type}

class Engineer(Employee):
    __tablename__ = "engineer"
    id = mapped_column(Integer, ForeignKey("employee.id"), primary_key=True)
    engineer_info = mapped_column(String(30))

    __mapper_args__ = {
        "polymorphic_load": "selectin",
        "polymorphic_identity": "engineer",
    }

class Manager(Employee):
    __tablename__ = "manager"
    id = mapped_column(Integer, ForeignKey("employee.id"), primary_key=True)
    manager_name = mapped_column(String(30))

    __mapper_args__ = {
        "polymorphic_load": "selectin",
        "polymorphic_identity": "manager",
    }
```

使用上述映射，针对`Employee`类的 SELECT 语句在发出语句时会自动假定使用`selectin_polymorphic(Employee, [Engineer, Manager])`作为加载器选项。  ## 使用`with_polymorphic()`

与仅影响对象加载的 `selectin_polymorphic()` 相比，`with_polymorphic()` 构造影响了多态结构的 SQL 查询如何呈现，通常是作为一系列左外连接到每个包含的子表的 LEFT OUTER JOIN。这个连接结构被称为 **多态可选择项**。通过同时提供几个子表的视图，`with_polymorphic()` 提供了一种一次跨越多个继承类写 SELECT 语句的方法，并能够根据各个子表添加过滤条件的能力。

`with_polymorphic()` 本质上是 `aliased()` 构造的一种特殊形式。它接受的参数形式与 `selectin_polymorphic()` 类似，即被查询的基本实体，后跟一系列该实体的子类，其特定属性应该被加载到传入的行中：

```py
>>> from sqlalchemy.orm import with_polymorphic
>>> employee_poly = with_polymorphic(Employee, [Engineer, Manager])
```

为了指示所有子类都应该成为实体的一部分，`with_polymorphic()` 还将接受字符串 `"*"`，该字符串可以替代类序列以指示所有类（请注意，这目前还不被 `selectin_polymorphic()` 支持）：

```py
>>> employee_poly = with_polymorphic(Employee, "*")
```

下面的示例演示了与前一节中演示的相同操作，一次加载所有 `Manager` 和 `Engineer` 的所有列：

```py
>>> stmt = select(employee_poly).order_by(employee_poly.id)
>>> objects = session.scalars(stmt).all()
BEGIN  (implicit)
SELECT  employee.id,  employee.name,  employee.type,  employee.company_id,
manager.id  AS  id_1,  manager.manager_name,  engineer.id  AS  id_2,  engineer.engineer_info
FROM  employee
LEFT  OUTER  JOIN  manager  ON  employee.id  =  manager.id
LEFT  OUTER  JOIN  engineer  ON  employee.id  =  engineer.id  ORDER  BY  employee.id
[...]  ()
>>> print(objects)
[Manager('Mr. Krabs'), Engineer('SpongeBob'), Engineer('Squidward')]
```

与 `selectin_polymorphic()` 一样，子类的属性已经被加载：

```py
>>> print(objects[0].manager_name)
Eugene H. Krabs
```

由于 `with_polymorphic()` 默认生成的可选择项使用 LEFT OUTER JOIN，从数据库的角度来看，查询并不像 `selectin_polymorphic()` 所采用的方法那样优化，简单的 SELECT 语句仅使用基于每个表的 JOIN 发出。

### 使用 with_polymorphic() 过滤子类属性

`with_polymorphic()` 构造使包含的子类映射器上的属性可用，通过包含允许对子类的引用的命名空间。在前一节中创建的 `employee_poly` 构造包括名为 `.Engineer` 和 `.Manager` 的属性，这些属性为 `Engineer` 和 `Manager` 提供了关于多态 SELECT 的命名空间。在下面的示例中，我们可以使用 `or_()` 构造同时针对两个类创建条件：

```py
>>> from sqlalchemy import or_
>>> employee_poly = with_polymorphic(Employee, [Engineer, Manager])
>>> stmt = (
...     select(employee_poly)
...     .where(
...         or_(
...             employee_poly.Manager.manager_name == "Eugene H. Krabs",
...             employee_poly.Engineer.engineer_info
...             == "Senior Customer Engagement Engineer",
...         )
...     )
...     .order_by(employee_poly.id)
... )
>>> objects = session.scalars(stmt).all()
SELECT  employee.id,  employee.name,  employee.type,  employee.company_id,  manager.id  AS  id_1,
manager.manager_name,  engineer.id  AS  id_2,  engineer.engineer_info
FROM  employee
LEFT  OUTER  JOIN  manager  ON  employee.id  =  manager.id
LEFT  OUTER  JOIN  engineer  ON  employee.id  =  engineer.id
WHERE  manager.manager_name  =  ?  OR  engineer.engineer_info  =  ?
ORDER  BY  employee.id
[...]  ('Eugene H. Krabs',  'Senior Customer Engagement Engineer')
>>> print(objects)
[Manager('Mr. Krabs'), Engineer('Squidward')]
```  ### 使用别名化与 with_polymorphic

`with_polymorphic()` 构造，作为 `aliased()` 的特例，也提供了 `aliased()` 的基本功能，即对多态可选择本身的“别名化”。具体来说，这意味着两个或更多个引用相同类层次结构的 `with_polymorphic()` 实体可以同时在单个语句中使用。

要在连接继承映射中使用此功能，通常我们希望传递两个参数，`with_polymorphic.aliased` 以及 `with_polymorphic.flat`。`with_polymorphic.aliased` 参数表示多态可选择应该由此构造唯一的别名引用。`with_polymorphic.flat` 参数是特定于默认的 LEFT OUTER JOIN 多态可选择，并指示语句中应使用更优化的别名化形式。

为了说明这个特性，下面的示例发出了一个选择两个单独的多态实体，`Employee` 与 `Engineer` 连接，以及 `Employee` 与 `Manager` 连接的 SELECT。由于这两个多态实体都将在其多态可选择中包含基本的 `employee` 表，必须应用别名以区分这个表在其两个不同的上下文中。这两个多态实体被视为两个独立的表，因此通常需要以某种方式相互连接，如下所示，在这里实体在 `company_id` 列上与彼此连接，并附加一些额外的限制条件针对 `Employee` / `Manager` 实体：

```py
>>> manager_employee = with_polymorphic(Employee, [Manager], aliased=True, flat=True)
>>> engineer_employee = with_polymorphic(Employee, [Engineer], aliased=True, flat=True)
>>> stmt = (
...     select(manager_employee, engineer_employee)
...     .join(
...         engineer_employee,
...         engineer_employee.company_id == manager_employee.company_id,
...     )
...     .where(
...         or_(
...             manager_employee.name == "Mr. Krabs",
...             manager_employee.Manager.manager_name == "Eugene H. Krabs",
...         )
...     )
...     .order_by(engineer_employee.name, manager_employee.name)
... )
>>> for manager, engineer in session.execute(stmt):
...     print(f"{manager} {engineer}")
SELECT
employee_1.id,  employee_1.name,  employee_1.type,  employee_1.company_id,
manager_1.id  AS  id_1,  manager_1.manager_name,
employee_2.id  AS  id_2,  employee_2.name  AS  name_1,  employee_2.type  AS  type_1,
employee_2.company_id  AS  company_id_1,  engineer_1.id  AS  id_3,  engineer_1.engineer_info
FROM  employee  AS  employee_1
LEFT  OUTER  JOIN  manager  AS  manager_1  ON  employee_1.id  =  manager_1.id
JOIN
  (employee  AS  employee_2  LEFT  OUTER  JOIN  engineer  AS  engineer_1  ON  employee_2.id  =  engineer_1.id)
ON  employee_2.company_id  =  employee_1.company_id
WHERE  employee_1.name  =  ?  OR  manager_1.manager_name  =  ?
ORDER  BY  employee_2.name,  employee_1.name
[...]  ('Mr. Krabs',  'Eugene H. Krabs')
Manager('Mr. Krabs') Manager('Mr. Krabs')
Manager('Mr. Krabs') Engineer('SpongeBob')
Manager('Mr. Krabs') Engineer('Squidward')
```

在上面的例子中，`with_polymorphic.flat` 的行为是，多态可选项保持为其各自表的 LEFT OUTER JOIN，这些表本身被赋予匿名别名。还生成了一个右嵌套 JOIN。

当省略`with_polymorphic.flat` 参数时，通常行为是每个多态可选项都被包含在子查询中，产生更加冗长的形式：

```py
>>> manager_employee = with_polymorphic(Employee, [Manager], aliased=True)
>>> engineer_employee = with_polymorphic(Employee, [Engineer], aliased=True)
>>> stmt = (
...     select(manager_employee, engineer_employee)
...     .join(
...         engineer_employee,
...         engineer_employee.company_id == manager_employee.company_id,
...     )
...     .where(
...         or_(
...             manager_employee.name == "Mr. Krabs",
...             manager_employee.Manager.manager_name == "Eugene H. Krabs",
...         )
...     )
...     .order_by(engineer_employee.name, manager_employee.name)
... )
>>> print(stmt)
SELECT  anon_1.employee_id,  anon_1.employee_name,  anon_1.employee_type,
anon_1.employee_company_id,  anon_1.manager_id,  anon_1.manager_manager_name,  anon_2.employee_id  AS  employee_id_1,
anon_2.employee_name  AS  employee_name_1,  anon_2.employee_type  AS  employee_type_1,
anon_2.employee_company_id  AS  employee_company_id_1,  anon_2.engineer_id,  anon_2.engineer_engineer_info
FROM
(SELECT  employee.id  AS  employee_id,  employee.name  AS  employee_name,  employee.type  AS  employee_type,
employee.company_id  AS  employee_company_id,
manager.id  AS  manager_id,  manager.manager_name  AS  manager_manager_name
FROM  employee  LEFT  OUTER  JOIN  manager  ON  employee.id  =  manager.id)  AS  anon_1
JOIN
(SELECT  employee.id  AS  employee_id,  employee.name  AS  employee_name,  employee.type  AS  employee_type,
employee.company_id  AS  employee_company_id,  engineer.id  AS  engineer_id,  engineer.engineer_info  AS  engineer_engineer_info
FROM  employee  LEFT  OUTER  JOIN  engineer  ON  employee.id  =  engineer.id)  AS  anon_2
ON  anon_2.employee_company_id  =  anon_1.employee_company_id
WHERE  anon_1.employee_name  =  :employee_name_2  OR  anon_1.manager_manager_name  =  :manager_manager_name_1
ORDER  BY  anon_2.employee_name,  anon_1.employee_name 
```

上述形式在历史上更容易移植到不一定支持右嵌套 JOIN 的后端，并且在使用诸如具体表继承映射以及一般情况下使用替代多态可选项时，可能也是合适的。  ### 在映射上配置 with_polymorphic()

与 `selectin_polymorphic()` 类似，`with_polymorphic()` 构造也支持一个经过映射配置的版本，可以通过两种不同的方式进行配置，要么在基类上使用 `mapper.with_polymorphic` 参数，要么以更现代的形式在每个子类上使用 `Mapper.polymorphic_load` 参数，传递值为 `"inline"`。

警告

对于加入继承映射，更倾向于在查询中明确使用 `with_polymorphic()` ，或者对于隐式的子类急加载使用 `Mapper.polymorphic_load` 以 `"selectin"` 为参数，而不是使用本节中描述的映射级别的 `mapper.with_polymorphic` 参数。该参数调用了旨在重写 SELECT 语句中的 FROM 子句的复杂启发式规则，可能会干扰更复杂语句的构建，尤其是那些涉及到引用同一映射实体的嵌套子查询的语句。

例如，我们可以使用以下方式声明我们的 `Employee` 映射，将 `Mapper.polymorphic_load` 设为 `"inline"`：

```py
class Employee(Base):
    __tablename__ = "employee"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    type = mapped_column(String(50))

    __mapper_args__ = {"polymorphic_identity": "employee", "polymorphic_on": type}

class Engineer(Employee):
    __tablename__ = "engineer"
    id = mapped_column(Integer, ForeignKey("employee.id"), primary_key=True)
    engineer_info = mapped_column(String(30))

    __mapper_args__ = {
        "polymorphic_load": "inline",
        "polymorphic_identity": "engineer",
    }

class Manager(Employee):
    __tablename__ = "manager"
    id = mapped_column(Integer, ForeignKey("employee.id"), primary_key=True)
    manager_name = mapped_column(String(30))

    __mapper_args__ = {
        "polymorphic_load": "inline",
        "polymorphic_identity": "manager",
    }
```

对于上述映射，针对 `Employee` 类的 SELECT 语句将自动假设在发出语句时使用 `with_polymorphic(Employee, [Engineer, Manager])` 作为主要实体：

```py
print(select(Employee))
SELECT  employee.id,  employee.name,  employee.type,  engineer.id  AS  id_1,
engineer.engineer_info,  manager.id  AS  id_2,  manager.manager_name
FROM  employee
LEFT  OUTER  JOIN  engineer  ON  employee.id  =  engineer.id
LEFT  OUTER  JOIN  manager  ON  employee.id  =  manager.id 
```

当使用映射器级“with polymorphic”时，查询也可以直接引用子类实体，其中它们隐式地表示多态查询中的连接表。在上面的例子中，我们可以自由地直接引用 `Manager` 和 `Engineer` 对默认的 `Employee` 实体进行查询：

```py
print(
 select(Employee).where(
 or_(Manager.manager_name == "x", Engineer.engineer_info == "y")
 )
)
SELECT  employee.id,  employee.name,  employee.type,  engineer.id  AS  id_1,
engineer.engineer_info,  manager.id  AS  id_2,  manager.manager_name
FROM  employee
LEFT  OUTER  JOIN  engineer  ON  employee.id  =  engineer.id
LEFT  OUTER  JOIN  manager  ON  employee.id  =  manager.id
WHERE  manager.manager_name  =  :manager_name_1
OR  engineer.engineer_info  =  :engineer_info_1 
```

然而，如果我们需要在单独的别名上下文中引用 `Employee` 实体或其子实体，我们将再次直接使用 `with_polymorphic()` 来定义这些别名实体，如 使用别名与 with_polymorphic 中所示。

对于对多态可选的更集中的控制，可以使用更传统的映射器级多态控制形式，即 `Mapper.with_polymorphic` 参数，配置在基类上。此参数接受与 `with_polymorphic()` 构造相当的参数，然而，在连接继承映射中的常见用法是使用普通的星号，表示所有子表都应该进行 LEFT OUTER JOIN，如下所示：

```py
class Employee(Base):
    __tablename__ = "employee"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    type = mapped_column(String(50))

    __mapper_args__ = {
        "polymorphic_identity": "employee",
        "with_polymorphic": "*",
        "polymorphic_on": type,
    }

class Engineer(Employee):
    __tablename__ = "engineer"
    id = mapped_column(Integer, ForeignKey("employee.id"), primary_key=True)
    engineer_info = mapped_column(String(30))

    __mapper_args__ = {
        "polymorphic_identity": "engineer",
    }

class Manager(Employee):
    __tablename__ = "manager"
    id = mapped_column(Integer, ForeignKey("employee.id"), primary_key=True)
    manager_name = mapped_column(String(30))

    __mapper_args__ = {
        "polymorphic_identity": "manager",
    }
```

总的来说，`with_polymorphic()` 和诸如 `Mapper.with_polymorphic` 这样的选项使用的 LEFT OUTER JOIN 格式可能从 SQL 和数据库优化器的角度来看会比较繁琐；对于连接继承映射中子类属性的一般加载，应该更倾向于使用 `selectin_polymorphic()` 方法，或者将 `Mapper.polymorphic_load` 设置为 `"selectin"` 的映射器级等效，只在需要时才在每个查询基础上使用 `with_polymorphic()`。  ## 加入到特定子类型或 with_polymorphic() 实体

由于`with_polymorphic()`实体是`aliased()`的一种特殊情况，在将多态实体视为连接的目标时，特别是在使用`relationship()`构造作为 ON 子句时，我们使用与常规别名相同的技术，如在 Using Relationship to join between aliased targets 中详细说明的，最简洁的方式是使用`PropComparator.of_type()`。在下面的示例中，我们演示了从父实体`Company`沿着一对多关系`Company.employees`进行连接，该关系在 setup 中配置为链接到`Employee`对象，使用一个`with_polymorphic()`实体作为目标：

```py
>>> employee_plus_engineer = with_polymorphic(Employee, [Engineer])
>>> stmt = (
...     select(Company.name, employee_plus_engineer.name)
...     .join(Company.employees.of_type(employee_plus_engineer))
...     .where(
...         or_(
...             employee_plus_engineer.name == "SpongeBob",
...             employee_plus_engineer.Engineer.engineer_info
...             == "Senior Customer Engagement Engineer",
...         )
...     )
... )
>>> for company_name, emp_name in session.execute(stmt):
...     print(f"{company_name} {emp_name}")
SELECT  company.name,  employee.name  AS  name_1
FROM  company  JOIN  (employee  LEFT  OUTER  JOIN  engineer  ON  employee.id  =  engineer.id)  ON  company.id  =  employee.company_id
WHERE  employee.name  =  ?  OR  engineer.engineer_info  =  ?
[...]  ('SpongeBob',  'Senior Customer Engagement Engineer')
Krusty Krab SpongeBob
Krusty Krab Squidward
```

更直接地说，`PropComparator.of_type()`也与任何类型的继承映射一起使用，以将一个`relationship()`的连接限制为特定的子类型。上面的查询可以严格按照`Engineer`目标来编写，如下所示：

```py
>>> stmt = (
...     select(Company.name, Engineer.name)
...     .join(Company.employees.of_type(Engineer))
...     .where(
...         or_(
...             Engineer.name == "SpongeBob",
...             Engineer.engineer_info == "Senior Customer Engagement Engineer",
...         )
...     )
... )
>>> for company_name, emp_name in session.execute(stmt):
...     print(f"{company_name} {emp_name}")
SELECT  company.name,  employee.name  AS  name_1
FROM  company  JOIN  (employee  JOIN  engineer  ON  employee.id  =  engineer.id)  ON  company.id  =  employee.company_id
WHERE  employee.name  =  ?  OR  engineer.engineer_info  =  ?
[...]  ('SpongeBob',  'Senior Customer Engagement Engineer')
Krusty Krab SpongeBob
Krusty Krab Squidward
```

从上面可以观察到，直接加入到`Engineer`目标，而不是使用`with_polymorphic(Employee, [Engineer])`的“多态可选择”具有一个有用的特性，即使用内连接而不是左外连接，从 SQL 优化器的角度来看，这通常更具性能。

### 多态子类型的急切加载

在前一节中用`Select.join()`方法演示的`PropComparator.of_type()`也可以等效地应用于 relationship loader options，如`selectinload()`和`joinedload()`。

作为一个基本示例，如果我们希望加载`Company`对象，并且使用`with_polymorphic()`构造来对整个层次结构的`Company.employees`的所有元素进行急切加载，我们可以编写如下代码：

```py
>>> all_employees = with_polymorphic(Employee, "*")
>>> stmt = select(Company).options(selectinload(Company.employees.of_type(all_employees)))
>>> for company in session.scalars(stmt):
...     print(f"company: {company.name}")
...     print(f"employees: {company.employees}")
SELECT  company.id,  company.name
FROM  company
[...]  ()
SELECT  employee.company_id  AS  employee_company_id,  employee.id  AS  employee_id,
employee.name  AS  employee_name,  employee.type  AS  employee_type,  manager.id  AS  manager_id,
manager.manager_name  AS  manager_manager_name,  engineer.id  AS  engineer_id,
engineer.engineer_info  AS  engineer_engineer_info
FROM  employee
LEFT  OUTER  JOIN  manager  ON  employee.id  =  manager.id
LEFT  OUTER  JOIN  engineer  ON  employee.id  =  engineer.id
WHERE  employee.company_id  IN  (?)
[...]  (1,)
company:  Krusty  Krab
employees:  [Manager('Mr. Krabs'),  Engineer('SpongeBob'),  Engineer('Squidward')] 
```

上述查询可以直接与前一节中将`selectin_polymorphic()`应用于现有的急加载中的`selectin_polymorphic()`版本进行比较。

另请参阅

将`selectin_polymorphic()`应用于现有的急加载 - 演示了使用`selectin_polymorphic()`相同等效的示例，而不是上面的例子。 ## 单一继承映射的 SELECT 语句

单一表继承设置

本节讨论单表继承，描述在单表继承中使用单个表表示层次结构中的多个类。

查看本节的 ORM 设置。

与联接继承映射相比，对于单一继承映射，构造 SELECT 语句往往更简单，因为对于全部单一继承层次结构，只有一个表。

无论继承层次结构是否全是单一继承或具有联接和单一继承的混合，单一继承的 SELECT 语句都通过添加额外的 WHERE 条件来区分针对基类和子类的查询。

例如，对于`Employee`的单一继承示例映射的查询将使用简单的表 SELECT 加载`Manager`、`Engineer`和`Employee`类型的对象：

```py
>>> stmt = select(Employee).order_by(Employee.id)
>>> for obj in session.scalars(stmt):
...     print(f"{obj}")
BEGIN  (implicit)
SELECT  employee.id,  employee.name,  employee.type
FROM  employee  ORDER  BY  employee.id
[...]  ()
Manager('Mr. Krabs')
Engineer('SpongeBob')
Engineer('Squidward')
```

当针对特定子类发出加载时，会向 SELECT 添加限制行的其他条件，例如下面对`Engineer`实体执行的 SELECT：

```py
>>> stmt = select(Engineer).order_by(Engineer.id)
>>> objects = session.scalars(stmt).all()
SELECT  employee.id,  employee.name,  employee.type,  employee.engineer_info
FROM  employee
WHERE  employee.type  IN  (?)  ORDER  BY  employee.id
[...]  ('engineer',)
>>> for obj in objects:
...     print(f"{obj}")
Engineer('SpongeBob')
Engineer('Squidward')
```

### 优化单一继承的属性加载

单一继承映射关于如何选择子类上的属性的默认行为与联接继承的行为类似，即子类特定的属性仍然默认发出第二个 SELECT。在下面的示例中，加载一个`Manager`类型的单个`Employee`，但由于请求的类是`Employee`，所以默认情况下不会出现`Manager.manager_name`属性，并且当访问时会发出额外的 SELECT：

```py
>>> mr_krabs = session.scalars(select(Employee).where(Employee.name == "Mr. Krabs")).one()
BEGIN  (implicit)
SELECT  employee.id,  employee.name,  employee.type
FROM  employee
WHERE  employee.name  =  ?
[...]  ('Mr. Krabs',)
>>> mr_krabs.manager_name
SELECT  employee.manager_name  AS  employee_manager_name
FROM  employee
WHERE  employee.id  =  ?  AND  employee.type  IN  (?)
[...]  (1,  'manager')
'Eugene H. Krabs'
```

要更改此行为，对于单一继承以及联接继承加载中使用的额外属性的急加载，同样的一般概念也适用于单一继承，包括使用`selectin_polymorphic()`选项以及`with_polymorphic()`选项，后者简单地包含了额外的列，并且从 SQL 的角度来看，对于单一继承映射更有效：

```py
>>> employees = with_polymorphic(Employee, "*")
>>> stmt = select(employees).order_by(employees.id)
>>> objects = session.scalars(stmt).all()
BEGIN  (implicit)
SELECT  employee.id,  employee.name,  employee.type,
employee.manager_name,  employee.engineer_info
FROM  employee  ORDER  BY  employee.id
[...]  ()
>>> for obj in objects:
...     print(f"{obj}")
Manager('Mr. Krabs')
Engineer('SpongeBob')
Engineer('Squidward')
>>> objects[0].manager_name
'Eugene H. Krabs'
```

由于加载单继承子类映射的开销通常很小，因此建议对于那些预计加载其特定子类属性是常见的子类，包括 `Mapper.polymorphic_load` 参数，并将其设置为 `"inline"`。下面是一个示例，说明了包含此选项的 设置：

```py
>>> class Base(DeclarativeBase):
...     pass
>>> class Employee(Base):
...     __tablename__ = "employee"
...     id: Mapped[int] = mapped_column(primary_key=True)
...     name: Mapped[str]
...     type: Mapped[str]
...
...     def __repr__(self):
...         return f"{self.__class__.__name__}({self.name!r})"
...
...     __mapper_args__ = {
...         "polymorphic_identity": "employee",
...         "polymorphic_on": "type",
...     }
>>> class Manager(Employee):
...     manager_name: Mapped[str] = mapped_column(nullable=True)
...     __mapper_args__ = {
...         "polymorphic_identity": "manager",
...         "polymorphic_load": "inline",
...     }
>>> class Engineer(Employee):
...     engineer_info: Mapped[str] = mapped_column(nullable=True)
...     __mapper_args__ = {
...         "polymorphic_identity": "engineer",
...         "polymorphic_load": "inline",
...     }
```

根据上述映射，`Manager` 和 `Engineer` 类将自动在针对 `Employee` 实体的 SELECT 语句中包含它们的列：

```py
>>> print(select(Employee))
SELECT  employee.id,  employee.name,  employee.type,
employee.manager_name,  employee.engineer_info
FROM  employee 
```

## 继承加载 API

| 对象名称 | 描述 |
| --- | --- |
| selectin_polymorphic(base_cls, classes) | 指示应对子类的所有属性进行急切加载。 |
| with_polymorphic(base, classes[, selectable, flat, ...]) | 生成一个 `AliasedClass` 构造，指定给定基类的后代映射器的列。 |

```py
function sqlalchemy.orm.with_polymorphic(base: Type[_O] | Mapper[_O], classes: Literal['*'] | Iterable[Type[Any]], selectable: Literal[False, None] | FromClause = False, flat: bool = False, polymorphic_on: ColumnElement[Any] | None = None, aliased: bool = False, innerjoin: bool = False, adapt_on_names: bool = False, _use_mapper_path: bool = False) → AliasedClass[_O]
```

生成一个 `AliasedClass` 构造，指定给定基类的后代映射器的列。

使用此方法将确保每个后代映射器的表都包含在 FROM 子句中，并允许对这些表使用 filter() 条件。结果实例还将已加载这些列，因此不需要对这些列进行“后获取”。

另请参阅

使用 with_polymorphic() - 对 `with_polymorphic()` 的全面讨论。

参数：

+   `base` – 要别名化的基类。

+   `classes` – 一个类或映射器，或类/映射器列表，它们都继承自基类。或者，它也可以是字符串 `'*'`，在这种情况下，所有下降映射的类都将添加到 FROM 子句中。

+   `aliased` – 当为 True 时，可选择的将被别名化。对于 JOIN，这意味着 JOIN 将从子查询中进行 SELECT，除非设置了 `with_polymorphic.flat` 标志为 True，这对于简单的用例是推荐的。

+   `flat` – 布尔值，将传递给`FromClause.alias()`调用，以便联接对象的别名别名联接内部的各个表，而不是创建子查询。这通常由所有现代数据库支持，关于右嵌套联接通常会产生更有效的查询。建议设置此标志，只要生成的 SQL 是功能性的。

+   `selectable` –

    将用于替代生成的 FROM 子句的表或子查询。如果任何所需类使用具体表继承，则此参数是必需的，因为 SQLAlchemy 当前无法自动生成表之间的 UNION。如果使用，`selectable`参数必须表示每个映射类映射的所有表和列的完整集。否则，未考虑的映射列将直接附加到 FROM 子句，这通常会导致不正确的结果。

    当保持其默认值`False`时，将为选择行使用分配给基本映射器的多态可选择对象。但是，也可以将其传递为`None`，这将绕过配置的多态可选择对象，并代替构造给定目标类的临时可选择对象；对于联接表继承，这将是一个包含所有目标映射器及其子类的联接。

+   `polymorphic_on` – 用作给定可选择对象的“判别器”列。如果未提供，则将使用基类映射器的`polymorphic_on`属性（如果有）。这对于默认没有多态加载行为的映射非常有用。

+   `innerjoin` – 如果为 True，则使用 INNER JOIN。只有在仅查询一个特定的子类型时才应指定此选项

+   `adapt_on_names` –

    通过`aliased.adapt_on_names`参数传递到别名对象。在给定可选择对象与现有映射的可选择对象没有直接关联的情况下，这可能会有所帮助。

    自版本 1.4.33 起新增。

```py
function sqlalchemy.orm.selectin_polymorphic(base_cls: _EntityType[Any], classes: Iterable[Type[Any]]) → _AbstractLoad
```

指示应针对特定子类的所有属性进行急加载。

这使用额外的 SELECT 与所有匹配的主键值进行 IN 比较，并且是与`mapper.polymorphic_load`参数上的`"selectin"`设置对应的每个查询的类似物。

自版本 1.2 起新增。

另请参阅

使用`selectin_polymorphic()`

## 从基类 vs. 特定子类进行 SELECT

对于联合继承层次结构中的类构建的 SELECT 语句将查询该类映射到的表，以及任何存在的超级表，使用 JOIN 将它们链接在一起。然后，该查询将返回请求类型的对象以及请求类型的任何子类型，使用每行中的鉴别器值来确定正确的类型。下面的查询是针对`Employee`的`Manager`子类建立的，然后返回的结果将仅包含`Manager`类型的对象：

```py
>>> from sqlalchemy import select
>>> stmt = select(Manager).order_by(Manager.id)
>>> managers = session.scalars(stmt).all()
BEGIN  (implicit)
SELECT  manager.id,  employee.id  AS  id_1,  employee.name,  employee.type,  employee.company_id,  manager.manager_name
FROM  employee  JOIN  manager  ON  employee.id  =  manager.id  ORDER  BY  manager.id
[...]  ()
>>> print(managers)
[Manager('Mr. Krabs')]
```

当 SELECT 语句针对层次结构中的基类时，默认行为是仅包括该类的表在渲染的 SQL 中，并且不会使用 JOIN。与所有情况一样，鉴别器列用于区分不同的请求子类型，然后结果是返回任何可能的子类型的对象。返回的对象将具有与基本表对应的属性填充，而与子表对应的属性将以未加载状态开始，在访问时自动加载。子属性的加载可配置为以各种方式更加“急切”，这将在本节后面讨论。

下面的示例创建了针对`Employee`超类的查询。这表示结果集中可能包含任何类型的对象，包括`Manager`、`Engineer`和`Employee`：

```py
>>> from sqlalchemy import select
>>> stmt = select(Employee).order_by(Employee.id)
>>> objects = session.scalars(stmt).all()
BEGIN  (implicit)
SELECT  employee.id,  employee.name,  employee.type,  employee.company_id
FROM  employee  ORDER  BY  employee.id
[...]  ()
>>> print(objects)
[Manager('Mr. Krabs'), Engineer('SpongeBob'), Engineer('Squidward')]
```

上面，并未包括`Manager`和`Engineer`的附加表在 SELECT 中，这意味着返回的对象尚不包含来自这些表的数据，例如本示例中`Manager`类的`.manager_name`属性以及`Engineer`类的`.engineer_info`属性。这些属性起始处于过期状态，并且在首次访问时将自动填充自己，使用延迟加载：

```py
>>> mr_krabs = objects[0]
>>> print(mr_krabs.manager_name)
SELECT  manager.manager_name  AS  manager_manager_name
FROM  manager
WHERE  ?  =  manager.id
[...]  (1,)
Eugene H. Krabs
```

如果已加载大量对象，则此惰性加载行为是不可取的，因为消费应用程序将需要访问特定于子类的属性，这将是一个 N 加一问题的示例，每行发出额外的 SQL。这些额外的 SQL 可能会影响性能，并且还可能与诸如使用 asyncio 等方法不兼容。此外，在我们对`Employee`对象的查询中，由于查询仅针对基本表，因此我们无法以`Manager`或`Engineer`的术语添加涉及特定于子类的属性的 SQL 条件。接下来的两个部分详细介绍了两种以不同方式解决这两个问题的构造，`selectin_polymorphic()`加载器选项和`with_polymorphic()`实体构造。

## 使用 selectin_polymorphic()

为了解决在访问子类属性时的性能问题，可以使用`selectin_polymorphic()`加载器策略，一次性预加载这些额外的属性到许多对象中。此加载器选项的工作方式类似于`selectinload()`关系加载器策略，针对加载在层次结构中的对象发出额外的 SELECT 语句，使用`IN`查询基于主键的额外行。

`selectinload()`接受作为参数被查询的基本实体，然后是该实体的一系列子类，这些子类的特定属性应加载到传入的行中：

```py
>>> from sqlalchemy.orm import selectin_polymorphic
>>> loader_opt = selectin_polymorphic(Employee, [Manager, Engineer])
```

然后，`selectin_polymorphic()`构造被用作加载器选项，将其传递给`Select.options()`方法`Select`。该示例说明了使用`selectin_polymorphic()`急切加载`Manager`和`Engineer`子类本地列的用法：

```py
>>> from sqlalchemy.orm import selectin_polymorphic
>>> loader_opt = selectin_polymorphic(Employee, [Manager, Engineer])
>>> stmt = select(Employee).order_by(Employee.id).options(loader_opt)
>>> objects = session.scalars(stmt).all()
BEGIN  (implicit)
SELECT  employee.id,  employee.name,  employee.type,  employee.company_id
FROM  employee  ORDER  BY  employee.id
[...]  ()
SELECT  manager.id  AS  manager_id,  employee.id  AS  employee_id,
employee.type  AS  employee_type,  manager.manager_name  AS  manager_manager_name
FROM  employee  JOIN  manager  ON  employee.id  =  manager.id
WHERE  employee.id  IN  (?)  ORDER  BY  employee.id
[...]  (1,)
SELECT  engineer.id  AS  engineer_id,  employee.id  AS  employee_id,
employee.type  AS  employee_type,  engineer.engineer_info  AS  engineer_engineer_info
FROM  employee  JOIN  engineer  ON  employee.id  =  engineer.id
WHERE  employee.id  IN  (?,  ?)  ORDER  BY  employee.id
[...]  (2,  3)
>>> print(objects)
[Manager('Mr. Krabs'), Engineer('SpongeBob'), Engineer('Squidward')]
```

上面的示例说明了额外发出的两个额外的 SELECT 语句，以便急切地获取额外的属性，如`Engineer.engineer_info`和`Manager.manager_name`。我们现在可以访问这些被加载的对象上的子属性，而不需要发出任何额外的 SQL 语句：

```py
>>> print(objects[0].manager_name)
Eugene H. Krabs
```

提示

`selectin_polymorphic()` 加载选项尚未针对基本的 `employee` 表进行优化，因此在第二和第三个“急加载”查询中不需要包含 `employee` 表；因此，在上面的示例中，我们看到了从 `employee` 到 `manager` 和 `engineer` 的 JOIN，即使 `employee` 的列已经加载。这与`selectinload()` 关系策略形成对比，在这方面更加复杂，并且在不需要时可以消除 JOIN。

### 将 selectin_polymorphic() 应用于现有的急加载

除了将 `selectin_polymorphic()` 指定为由语句加载的顶级实体的选项之外，我们还可以在现有加载的目标上指示 `selectin_polymorphic()`。由于我们的设置映射包括一个带有引用 `Employee` 实体的 `Company.employees` `relationship()` 的父 `Company` 实体，我们可以说明针对 `Company` 实体进行的 SELECT，该 SELECT 急切加载所有 `Employee` 对象以及其子类型上的所有属性，方法是将 `Load.selectin_polymorphic()` 应用为链接的加载选项；在此形式中，第一个参数从前一个加载选项隐式获得（在本例中为 `selectinload()`），因此我们只指示我们希望加载的附加目标子类：

```py
>>> from sqlalchemy.orm import selectinload
>>> stmt = select(Company).options(
...     selectinload(Company.employees).selectin_polymorphic([Manager, Engineer])
... )
>>> for company in session.scalars(stmt):
...     print(f"company: {company.name}")
...     print(f"employees: {company.employees}")
BEGIN  (implicit)
SELECT  company.id,  company.name
FROM  company
[...]  ()
SELECT  employee.company_id  AS  employee_company_id,  employee.id  AS  employee_id,
employee.name  AS  employee_name,  employee.type  AS  employee_type
FROM  employee
WHERE  employee.company_id  IN  (?)
[...]  (1,)
SELECT  manager.id  AS  manager_id,  employee.id  AS  employee_id,
employee.type  AS  employee_type,
manager.manager_name  AS  manager_manager_name
FROM  employee  JOIN  manager  ON  employee.id  =  manager.id
WHERE  employee.id  IN  (?)  ORDER  BY  employee.id
[...]  (1,)
SELECT  engineer.id  AS  engineer_id,  employee.id  AS  employee_id,
employee.type  AS  employee_type,
engineer.engineer_info  AS  engineer_engineer_info
FROM  employee  JOIN  engineer  ON  employee.id  =  engineer.id
WHERE  employee.id  IN  (?,  ?)  ORDER  BY  employee.id
[...]  (2,  3)
company: Krusty Krab
employees: [Manager('Mr. Krabs'), Engineer('SpongeBob'), Engineer('Squidward')]
```

另请参见

多态子类型的急加载 - 展示了使用 `with_polymorphic()` 的相同示例 ### 将加载选项应用于由 selectin_polymorphic 加载的子类

`selectin_polymorphic()` 发出的 SELECT 语句本身是 ORM 语句，因此我们还可以添加其他加载选项（例如文档中记录的关系加载技术） ，这些选项引用特定的子类。这些选项应作为**兄弟**应用于 `selectin_polymorphic()` 选项，即在 `select.options()` 中用逗号分隔。

例如，如果我们考虑 `Manager` 映射器与名为 `Paperwork` 的实体之间有一对多关系，我们可以结合使用 `selectin_polymorphic()` 和 `selectinload()` 在所有 `Manager` 对象上急加载此集合，其中 `Manager` 对象的子属性也被急加载：

```py
>>> from sqlalchemy.orm import selectin_polymorphic
>>> stmt = (
...     select(Employee)
...     .order_by(Employee.id)
...     .options(
...         selectin_polymorphic(Employee, [Manager, Engineer]),
...         selectinload(Manager.paperwork),
...     )
... )
>>> objects = session.scalars(stmt).all()
SELECT  employee.id,  employee.name,  employee.type,  employee.company_id
FROM  employee  ORDER  BY  employee.id
[...]  ()
SELECT  manager.id  AS  manager_id,  employee.id  AS  employee_id,  employee.type  AS  employee_type,  manager.manager_name  AS  manager_manager_name
FROM  employee  JOIN  manager  ON  employee.id  =  manager.id
WHERE  employee.id  IN  (?)  ORDER  BY  employee.id
[...]  (1,)
SELECT  paperwork.manager_id  AS  paperwork_manager_id,  paperwork.id  AS  paperwork_id,  paperwork.document_name  AS  paperwork_document_name
FROM  paperwork
WHERE  paperwork.manager_id  IN  (?)
[...]  (1,)
SELECT  engineer.id  AS  engineer_id,  employee.id  AS  employee_id,  employee.type  AS  employee_type,  engineer.engineer_info  AS  engineer_engineer_info
FROM  employee  JOIN  engineer  ON  employee.id  =  engineer.id
WHERE  employee.id  IN  (?,  ?)  ORDER  BY  employee.id
[...]  (2,  3)
>>> print(objects[0])
Manager('Mr. Krabs')
>>> print(objects[0].paperwork)
[Paperwork('Secret Recipes'), Paperwork('Krabby Patty Orders')]
```

#### 当 selectin_polymorphic 本身是子选项时应用加载器选项

2.0.21 版中的新功能。

前一节说明了 `selectin_polymorphic()` 和 `selectinload()` 作为兄弟选项的用法，都在单个对 `select.options()` 的调用中使用。 如果目标实体已经从父关系加载，例如在将 selectin_polymorphic() 应用于现有的急加载示例中，我们可以使用 `Load.options()` 方法应用此“兄弟”模式，该方法将子选项应用于父选项，如在使用 Load.options() 指定子选项示例中说明的。 下面我们结合两个示例，加载 `Company.employees`，还加载 `Manager` 和 `Engineer` 类的属性，以及急加载 ``Manager.paperwork`` 属性：

```py
>>> from sqlalchemy.orm import selectinload
>>> stmt = select(Company).options(
...     selectinload(Company.employees).options(
...         selectin_polymorphic(Employee, [Manager, Engineer]),
...         selectinload(Manager.paperwork),
...     )
... )
>>> for company in session.scalars(stmt):
...     print(f"company: {company.name}")
...     for employee in company.employees:
...         if isinstance(employee, Manager):
...             print(f"manager: {employee.name} paperwork: {employee.paperwork}")
BEGIN  (implicit)
SELECT  company.id,  company.name
FROM  company
[...]  ()
SELECT  employee.company_id  AS  employee_company_id,  employee.id  AS  employee_id,  employee.name  AS  employee_name,  employee.type  AS  employee_type
FROM  employee
WHERE  employee.company_id  IN  (?)
[...]  (1,)
SELECT  manager.id  AS  manager_id,  employee.id  AS  employee_id,  employee.type  AS  employee_type,  manager.manager_name  AS  manager_manager_name
FROM  employee  JOIN  manager  ON  employee.id  =  manager.id
WHERE  employee.id  IN  (?)  ORDER  BY  employee.id
[...]  (1,)
SELECT  paperwork.manager_id  AS  paperwork_manager_id,  paperwork.id  AS  paperwork_id,  paperwork.document_name  AS  paperwork_document_name
FROM  paperwork
WHERE  paperwork.manager_id  IN  (?)
[...]  (1,)
SELECT  engineer.id  AS  engineer_id,  employee.id  AS  employee_id,  employee.type  AS  employee_type,  engineer.engineer_info  AS  engineer_engineer_info
FROM  employee  JOIN  engineer  ON  employee.id  =  engineer.id
WHERE  employee.id  IN  (?,  ?)  ORDER  BY  employee.id
[...]  (2,  3)
company: Krusty Krab
manager: Mr. Krabs paperwork: [Paperwork('Secret Recipes'), Paperwork('Krabby Patty Orders')]
```

### 配置 mappers 上的 selectin_polymorphic()

可以在特定的 mappers 上配置 `selectin_polymorphic()` 的行为，以便默认情况下执行，通过使用 `Mapper.polymorphic_load` 参数，在每个子类基础上使用值 `"selectin"`。 以下示例说明了在 `Engineer` 和 `Manager` 子类中使用此参数的用法：

```py
class Employee(Base):
    __tablename__ = "employee"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    type = mapped_column(String(50))

    __mapper_args__ = {"polymorphic_identity": "employee", "polymorphic_on": type}

class Engineer(Employee):
    __tablename__ = "engineer"
    id = mapped_column(Integer, ForeignKey("employee.id"), primary_key=True)
    engineer_info = mapped_column(String(30))

    __mapper_args__ = {
        "polymorphic_load": "selectin",
        "polymorphic_identity": "engineer",
    }

class Manager(Employee):
    __tablename__ = "manager"
    id = mapped_column(Integer, ForeignKey("employee.id"), primary_key=True)
    manager_name = mapped_column(String(30))

    __mapper_args__ = {
        "polymorphic_load": "selectin",
        "polymorphic_identity": "manager",
    }
```

以上映射，针对 `Employee` 类的 SELECT 语句在发出语句时将自动假定使用 `selectin_polymorphic(Employee, [Engineer, Manager])` 作为加载器选项。

### 将 selectin_polymorphic() 应用于现有的急加载

在指定为语句加载的顶级实体选项时，除了 `selectin_polymorphic()` 之外，我们还可以在现有加载的目标上指示 `selectin_polymorphic()`。由于我们的 设置 映射包括一个父 `Company` 实体，其具有引用到 `Employee` 实体的 `Company.employees` `relationship()`，我们可以如下所示对 `Company` 实体进行选择，该选择会急切地加载所有 `Employee` 对象以及其子类型的所有属性，方法是将 `Load.selectin_polymorphic()` 应用为链接的加载器选项；在这种形式中，第一个参数是从上一个加载器选项中隐含的（在本例中是 `selectinload()`），因此我们只指示我们希望加载的附加目标子类：

```py
>>> from sqlalchemy.orm import selectinload
>>> stmt = select(Company).options(
...     selectinload(Company.employees).selectin_polymorphic([Manager, Engineer])
... )
>>> for company in session.scalars(stmt):
...     print(f"company: {company.name}")
...     print(f"employees: {company.employees}")
BEGIN  (implicit)
SELECT  company.id,  company.name
FROM  company
[...]  ()
SELECT  employee.company_id  AS  employee_company_id,  employee.id  AS  employee_id,
employee.name  AS  employee_name,  employee.type  AS  employee_type
FROM  employee
WHERE  employee.company_id  IN  (?)
[...]  (1,)
SELECT  manager.id  AS  manager_id,  employee.id  AS  employee_id,
employee.type  AS  employee_type,
manager.manager_name  AS  manager_manager_name
FROM  employee  JOIN  manager  ON  employee.id  =  manager.id
WHERE  employee.id  IN  (?)  ORDER  BY  employee.id
[...]  (1,)
SELECT  engineer.id  AS  engineer_id,  employee.id  AS  employee_id,
employee.type  AS  employee_type,
engineer.engineer_info  AS  engineer_engineer_info
FROM  employee  JOIN  engineer  ON  employee.id  =  engineer.id
WHERE  employee.id  IN  (?,  ?)  ORDER  BY  employee.id
[...]  (2,  3)
company: Krusty Krab
employees: [Manager('Mr. Krabs'), Engineer('SpongeBob'), Engineer('Squidward')]
```

另请参见

多态子类型的急切加载 - 使用 `with_polymorphic()` 演示了与上述相同的等效示例

### 将加载器选项应用于由 selectin_polymorphic 加载的子类

由 `selectin_polymorphic()` 发出的 SELECT 语句本身是 ORM 语句，因此我们也可以添加其他加载器选项（例如文档中记录的那些位于 关系加载技术），这些选项是指特定的子类。这些选项应作为 **selectin_polymorphic()** 选项的 **siblings** 应用，即在 `select.options()` 中用逗号分隔。

例如，如果我们考虑到 `Manager` 映射器有一个 一对多 关系，指向一个名为 `Paperwork` 的实体，我们可以结合使用 `selectin_polymorphic()` 和 `selectinload()` 来急切地加载所有 `Manager` 对象上的此集合，其中 `Manager` 对象的子属性也被急切地加载：

```py
>>> from sqlalchemy.orm import selectin_polymorphic
>>> stmt = (
...     select(Employee)
...     .order_by(Employee.id)
...     .options(
...         selectin_polymorphic(Employee, [Manager, Engineer]),
...         selectinload(Manager.paperwork),
...     )
... )
>>> objects = session.scalars(stmt).all()
SELECT  employee.id,  employee.name,  employee.type,  employee.company_id
FROM  employee  ORDER  BY  employee.id
[...]  ()
SELECT  manager.id  AS  manager_id,  employee.id  AS  employee_id,  employee.type  AS  employee_type,  manager.manager_name  AS  manager_manager_name
FROM  employee  JOIN  manager  ON  employee.id  =  manager.id
WHERE  employee.id  IN  (?)  ORDER  BY  employee.id
[...]  (1,)
SELECT  paperwork.manager_id  AS  paperwork_manager_id,  paperwork.id  AS  paperwork_id,  paperwork.document_name  AS  paperwork_document_name
FROM  paperwork
WHERE  paperwork.manager_id  IN  (?)
[...]  (1,)
SELECT  engineer.id  AS  engineer_id,  employee.id  AS  employee_id,  employee.type  AS  employee_type,  engineer.engineer_info  AS  engineer_engineer_info
FROM  employee  JOIN  engineer  ON  employee.id  =  engineer.id
WHERE  employee.id  IN  (?,  ?)  ORDER  BY  employee.id
[...]  (2,  3)
>>> print(objects[0])
Manager('Mr. Krabs')
>>> print(objects[0].paperwork)
[Paperwork('Secret Recipes'), Paperwork('Krabby Patty Orders')]
```

#### 当 selectin_polymorphic 本身是子选项时应用加载器选项

新版本 2.0.21 中新增。

前一节说明了`selectin_polymorphic()`和`selectinload()`作为兄弟选项使用的示例，两者都在单个调用`select.options()`中使用。如果目标实体已经从父关系中加载，就像在将 selectin_polymorphic()应用于现有的急加载的示例中一样，我们可以使用`Load.options()`方法应用这种“兄弟”模式，将子选项应用于父选项，如在使用 Load.options()指定子选项中所示。下面我们结合这两个示例来加载`Company.employees`，同时加载`Manager`和`Engineer`类的属性，以及急加载`Manager.paperwork`属性：

```py
>>> from sqlalchemy.orm import selectinload
>>> stmt = select(Company).options(
...     selectinload(Company.employees).options(
...         selectin_polymorphic(Employee, [Manager, Engineer]),
...         selectinload(Manager.paperwork),
...     )
... )
>>> for company in session.scalars(stmt):
...     print(f"company: {company.name}")
...     for employee in company.employees:
...         if isinstance(employee, Manager):
...             print(f"manager: {employee.name} paperwork: {employee.paperwork}")
BEGIN  (implicit)
SELECT  company.id,  company.name
FROM  company
[...]  ()
SELECT  employee.company_id  AS  employee_company_id,  employee.id  AS  employee_id,  employee.name  AS  employee_name,  employee.type  AS  employee_type
FROM  employee
WHERE  employee.company_id  IN  (?)
[...]  (1,)
SELECT  manager.id  AS  manager_id,  employee.id  AS  employee_id,  employee.type  AS  employee_type,  manager.manager_name  AS  manager_manager_name
FROM  employee  JOIN  manager  ON  employee.id  =  manager.id
WHERE  employee.id  IN  (?)  ORDER  BY  employee.id
[...]  (1,)
SELECT  paperwork.manager_id  AS  paperwork_manager_id,  paperwork.id  AS  paperwork_id,  paperwork.document_name  AS  paperwork_document_name
FROM  paperwork
WHERE  paperwork.manager_id  IN  (?)
[...]  (1,)
SELECT  engineer.id  AS  engineer_id,  employee.id  AS  employee_id,  employee.type  AS  employee_type,  engineer.engineer_info  AS  engineer_engineer_info
FROM  employee  JOIN  engineer  ON  employee.id  =  engineer.id
WHERE  employee.id  IN  (?,  ?)  ORDER  BY  employee.id
[...]  (2,  3)
company: Krusty Krab
manager: Mr. Krabs paperwork: [Paperwork('Secret Recipes'), Paperwork('Krabby Patty Orders')]
```  #### 当 selectin_polymorphic 本身是子选项时应用加载器选项

2.0.21 版本中的新功能。

前一节说明了`selectin_polymorphic()`和`selectinload()`作为兄弟选项使用的示例，两者都在单个调用`select.options()`中使用。如果目标实体已经从父关系中加载，就像在将 selectin_polymorphic()应用于现有的急加载的示例中一样，我们可以使用`Load.options()`方法应用这种“兄弟”模式，将子选项应用于父选项，如在使用 Load.options()指定子选项中所示。下面我们结合这两个示例来加载`Company.employees`，同时加载`Manager`和`Engineer`类的属性，以及急加载`Manager.paperwork`属性：

```py
>>> from sqlalchemy.orm import selectinload
>>> stmt = select(Company).options(
...     selectinload(Company.employees).options(
...         selectin_polymorphic(Employee, [Manager, Engineer]),
...         selectinload(Manager.paperwork),
...     )
... )
>>> for company in session.scalars(stmt):
...     print(f"company: {company.name}")
...     for employee in company.employees:
...         if isinstance(employee, Manager):
...             print(f"manager: {employee.name} paperwork: {employee.paperwork}")
BEGIN  (implicit)
SELECT  company.id,  company.name
FROM  company
[...]  ()
SELECT  employee.company_id  AS  employee_company_id,  employee.id  AS  employee_id,  employee.name  AS  employee_name,  employee.type  AS  employee_type
FROM  employee
WHERE  employee.company_id  IN  (?)
[...]  (1,)
SELECT  manager.id  AS  manager_id,  employee.id  AS  employee_id,  employee.type  AS  employee_type,  manager.manager_name  AS  manager_manager_name
FROM  employee  JOIN  manager  ON  employee.id  =  manager.id
WHERE  employee.id  IN  (?)  ORDER  BY  employee.id
[...]  (1,)
SELECT  paperwork.manager_id  AS  paperwork_manager_id,  paperwork.id  AS  paperwork_id,  paperwork.document_name  AS  paperwork_document_name
FROM  paperwork
WHERE  paperwork.manager_id  IN  (?)
[...]  (1,)
SELECT  engineer.id  AS  engineer_id,  employee.id  AS  employee_id,  employee.type  AS  employee_type,  engineer.engineer_info  AS  engineer_engineer_info
FROM  employee  JOIN  engineer  ON  employee.id  =  engineer.id
WHERE  employee.id  IN  (?,  ?)  ORDER  BY  employee.id
[...]  (2,  3)
company: Krusty Krab
manager: Mr. Krabs paperwork: [Paperwork('Secret Recipes'), Paperwork('Krabby Patty Orders')]
```

### 配置 selectin_polymorphic()在映射器上

`selectin_polymorphic()`的行为可以在特定的映射器上进行配置，以便默认情况下发生，通过使用`Mapper.polymorphic_load`参数，在每个子类上使用值`"selectin"`。下面的示例说明了在`Engineer`和`Manager`子类中使用此参数的方法：

```py
class Employee(Base):
    __tablename__ = "employee"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    type = mapped_column(String(50))

    __mapper_args__ = {"polymorphic_identity": "employee", "polymorphic_on": type}

class Engineer(Employee):
    __tablename__ = "engineer"
    id = mapped_column(Integer, ForeignKey("employee.id"), primary_key=True)
    engineer_info = mapped_column(String(30))

    __mapper_args__ = {
        "polymorphic_load": "selectin",
        "polymorphic_identity": "engineer",
    }

class Manager(Employee):
    __tablename__ = "manager"
    id = mapped_column(Integer, ForeignKey("employee.id"), primary_key=True)
    manager_name = mapped_column(String(30))

    __mapper_args__ = {
        "polymorphic_load": "selectin",
        "polymorphic_identity": "manager",
    }
```

使用上述映射，针对`Employee`类的 SELECT 语句在发出语句时将自动假定使用`selectin_polymorphic(Employee, [Engineer, Manager])`作为加载选项。

## 使用 with_polymorphic()

与仅影响对象加载的 `selectin_polymorphic()` 相比，`with_polymorphic()` 构造影响多态结构的 SQL 查询的呈现方式，通常作为一系列 LEFT OUTER JOIN 到每个包含的子表。这个连接结构称为 **多态可选择**。通过提供一次查看多个子表的视图，`with_polymorphic()` 提供了一种一次跨多个继承类编写 SELECT 语句的方法，并能够根据各个子表添加过滤条件。

`with_polymorphic()` 本质上是 `aliased()` 构造的一种特殊形式。它接受的参数形式与 `selectin_polymorphic()` 相似，即被查询的基本实体，后跟一系列这个实体的子类，用于加载其特定属性的传入行：

```py
>>> from sqlalchemy.orm import with_polymorphic
>>> employee_poly = with_polymorphic(Employee, [Engineer, Manager])
```

为了指示所有子类都应该成为实体的一部分，`with_polymorphic()` 还将接受字符串 `"*"`，它可以在类序列的位置传递，以表示所有类（请注意，`selectin_polymorphic()` 尚不支持这一点）：

```py
>>> employee_poly = with_polymorphic(Employee, "*")
```

下面的示例演示了与前一节中相同操作的示例，一次加载 `Manager` 和 `Engineer` 的所有列：

```py
>>> stmt = select(employee_poly).order_by(employee_poly.id)
>>> objects = session.scalars(stmt).all()
BEGIN  (implicit)
SELECT  employee.id,  employee.name,  employee.type,  employee.company_id,
manager.id  AS  id_1,  manager.manager_name,  engineer.id  AS  id_2,  engineer.engineer_info
FROM  employee
LEFT  OUTER  JOIN  manager  ON  employee.id  =  manager.id
LEFT  OUTER  JOIN  engineer  ON  employee.id  =  engineer.id  ORDER  BY  employee.id
[...]  ()
>>> print(objects)
[Manager('Mr. Krabs'), Engineer('SpongeBob'), Engineer('Squidward')]
```

与 `selectin_polymorphic()` 类似，子类上的属性已经加载：

```py
>>> print(objects[0].manager_name)
Eugene H. Krabs
```

由于 `with_polymorphic()` 生成的默认可选择使用 LEFT OUTER JOIN，从数据库的角度来看，查询不像 `selectin_polymorphic()` 采用的方法那样优化，后者仅使用 JOINs 在每个表上发出简单的 SELECT 语句。

### 使用 with_polymorphic() 过滤子类属性

`with_polymorphic()` 构造使包含的子类映射器上的属性可用，通过包含允许引用子类的命名空间。 上一节中创建的 `employee_poly` 构造包含名为 `.Engineer` 和 `.Manager` 的属性，这些属性在多态 SELECT 的术语中为 `Engineer` 和 `Manager` 提供命名空间。 在下面的示例中，我们可以使用 `or_()` 构造同时针对两个类创建条件：

```py
>>> from sqlalchemy import or_
>>> employee_poly = with_polymorphic(Employee, [Engineer, Manager])
>>> stmt = (
...     select(employee_poly)
...     .where(
...         or_(
...             employee_poly.Manager.manager_name == "Eugene H. Krabs",
...             employee_poly.Engineer.engineer_info
...             == "Senior Customer Engagement Engineer",
...         )
...     )
...     .order_by(employee_poly.id)
... )
>>> objects = session.scalars(stmt).all()
SELECT  employee.id,  employee.name,  employee.type,  employee.company_id,  manager.id  AS  id_1,
manager.manager_name,  engineer.id  AS  id_2,  engineer.engineer_info
FROM  employee
LEFT  OUTER  JOIN  manager  ON  employee.id  =  manager.id
LEFT  OUTER  JOIN  engineer  ON  employee.id  =  engineer.id
WHERE  manager.manager_name  =  ?  OR  engineer.engineer_info  =  ?
ORDER  BY  employee.id
[...]  ('Eugene H. Krabs',  'Senior Customer Engagement Engineer')
>>> print(objects)
[Manager('Mr. Krabs'), Engineer('Squidward')]
```  ### 使用 `with_polymorphic` 进行别名处理

`with_polymorphic()` 构造作为 `aliased()` 的一种特殊情况，还提供了 `aliased()` 的基本功能，即对多态可选择本身进行“别名”处理。 具体来说，这意味着两个或多个引用相同类层次结构的 `with_polymorphic()` 实体可以同时在单个语句中使用。

要在联合继承映射中使用此功能，通常需要传递两个参数，`with_polymorphic.aliased` 和 `with_polymorphic.flat`。 `with_polymorphic.aliased` 参数指示多态可选择应该使用本结构唯一的别名来引用。 `with_polymorphic.flat` 参数特定于默认的 LEFT OUTER JOIN 多态可选择，并指示在语句中应使用更优化的别名形式。

为了说明此功能，下面的示例发出了对两个单独的多态实体 `Employee` 与 `Engineer`，以及 `Employee` 与 `Manager` 的 SELECT。由于这两个多态实体都将在其多态可选择中包括基本的 `employee` 表，因此必须应用别名以区分此表在其两个不同的上下文中。 这两个多态实体被视为两个单独的表，因此通常需要以某种方式彼此连接，如下例所示，其中实体在 `company_id` 列上与彼此连接，并附加一些针对 `Employee` / `Manager` 实体的额外限制条件：

```py
>>> manager_employee = with_polymorphic(Employee, [Manager], aliased=True, flat=True)
>>> engineer_employee = with_polymorphic(Employee, [Engineer], aliased=True, flat=True)
>>> stmt = (
...     select(manager_employee, engineer_employee)
...     .join(
...         engineer_employee,
...         engineer_employee.company_id == manager_employee.company_id,
...     )
...     .where(
...         or_(
...             manager_employee.name == "Mr. Krabs",
...             manager_employee.Manager.manager_name == "Eugene H. Krabs",
...         )
...     )
...     .order_by(engineer_employee.name, manager_employee.name)
... )
>>> for manager, engineer in session.execute(stmt):
...     print(f"{manager} {engineer}")
SELECT
employee_1.id,  employee_1.name,  employee_1.type,  employee_1.company_id,
manager_1.id  AS  id_1,  manager_1.manager_name,
employee_2.id  AS  id_2,  employee_2.name  AS  name_1,  employee_2.type  AS  type_1,
employee_2.company_id  AS  company_id_1,  engineer_1.id  AS  id_3,  engineer_1.engineer_info
FROM  employee  AS  employee_1
LEFT  OUTER  JOIN  manager  AS  manager_1  ON  employee_1.id  =  manager_1.id
JOIN
  (employee  AS  employee_2  LEFT  OUTER  JOIN  engineer  AS  engineer_1  ON  employee_2.id  =  engineer_1.id)
ON  employee_2.company_id  =  employee_1.company_id
WHERE  employee_1.name  =  ?  OR  manager_1.manager_name  =  ?
ORDER  BY  employee_2.name,  employee_1.name
[...]  ('Mr. Krabs',  'Eugene H. Krabs')
Manager('Mr. Krabs') Manager('Mr. Krabs')
Manager('Mr. Krabs') Engineer('SpongeBob')
Manager('Mr. Krabs') Engineer('Squidward')
```

在上面的示例中，`with_polymorphic.flat`的行为是多态选择保持为它们各自表的 LEFT OUTER JOIN，这些表本身被赋予匿名别名。还生成了右嵌套 JOIN。

当省略`with_polymorphic.flat`参数时，通常的行为是将每个多态可选择包含在子查询中，产生更冗长的形式：

```py
>>> manager_employee = with_polymorphic(Employee, [Manager], aliased=True)
>>> engineer_employee = with_polymorphic(Employee, [Engineer], aliased=True)
>>> stmt = (
...     select(manager_employee, engineer_employee)
...     .join(
...         engineer_employee,
...         engineer_employee.company_id == manager_employee.company_id,
...     )
...     .where(
...         or_(
...             manager_employee.name == "Mr. Krabs",
...             manager_employee.Manager.manager_name == "Eugene H. Krabs",
...         )
...     )
...     .order_by(engineer_employee.name, manager_employee.name)
... )
>>> print(stmt)
SELECT  anon_1.employee_id,  anon_1.employee_name,  anon_1.employee_type,
anon_1.employee_company_id,  anon_1.manager_id,  anon_1.manager_manager_name,  anon_2.employee_id  AS  employee_id_1,
anon_2.employee_name  AS  employee_name_1,  anon_2.employee_type  AS  employee_type_1,
anon_2.employee_company_id  AS  employee_company_id_1,  anon_2.engineer_id,  anon_2.engineer_engineer_info
FROM
(SELECT  employee.id  AS  employee_id,  employee.name  AS  employee_name,  employee.type  AS  employee_type,
employee.company_id  AS  employee_company_id,
manager.id  AS  manager_id,  manager.manager_name  AS  manager_manager_name
FROM  employee  LEFT  OUTER  JOIN  manager  ON  employee.id  =  manager.id)  AS  anon_1
JOIN
(SELECT  employee.id  AS  employee_id,  employee.name  AS  employee_name,  employee.type  AS  employee_type,
employee.company_id  AS  employee_company_id,  engineer.id  AS  engineer_id,  engineer.engineer_info  AS  engineer_engineer_info
FROM  employee  LEFT  OUTER  JOIN  engineer  ON  employee.id  =  engineer.id)  AS  anon_2
ON  anon_2.employee_company_id  =  anon_1.employee_company_id
WHERE  anon_1.employee_name  =  :employee_name_2  OR  anon_1.manager_manager_name  =  :manager_manager_name_1
ORDER  BY  anon_2.employee_name,  anon_1.employee_name 
```

上述形式在历史上更容易移植到不一定支持右嵌套 JOIN 的后端，并且在使用`with_polymorphic()`时，“多态可选择”不是表的简单 LEFT OUTER JOIN 时，如使用具体表继承映射以及一般情况下使用替代多态可选择时，此形式也可能是合适的。### 在映射器上配置 with_polymorphic()

与`selectin_polymorphic()`相似，`with_polymorphic()`构造也支持由映射器配置的版本，可以通过两种不同的方式进行配置，要么在基类上使用`mapper.with_polymorphic`参数，要么以更现代的形式在每个子类上使用`Mapper.polymorphic_load`参数，传递值`"inline"`。

警告

对于加入继承映射，请优先在查询中显式使用`with_polymorphic()`，或者对于隐式急切子类加载使用`Mapper.polymorphic_load`与`"selectin"`，而不是使用本节中描述的映射器级`mapper.with_polymorphic`参数。此参数调用旨在重写 SELECT 语句中的 FROM 子句的复杂启发式方法，可能会干扰构造更复杂语句的构造，特别是那些引用相同映射实体的嵌套子查询。

例如，我们可以使用`Mapper.polymorphic_load`将我们的`Employee`映射声明为`"inline"`，如下所示：

```py
class Employee(Base):
    __tablename__ = "employee"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    type = mapped_column(String(50))

    __mapper_args__ = {"polymorphic_identity": "employee", "polymorphic_on": type}

class Engineer(Employee):
    __tablename__ = "engineer"
    id = mapped_column(Integer, ForeignKey("employee.id"), primary_key=True)
    engineer_info = mapped_column(String(30))

    __mapper_args__ = {
        "polymorphic_load": "inline",
        "polymorphic_identity": "engineer",
    }

class Manager(Employee):
    __tablename__ = "manager"
    id = mapped_column(Integer, ForeignKey("employee.id"), primary_key=True)
    manager_name = mapped_column(String(30))

    __mapper_args__ = {
        "polymorphic_load": "inline",
        "polymorphic_identity": "manager",
    }
```

使用上述映射，针对`Employee`类的 SELECT 语句将自动假定在发出语句时将使用`with_polymorphic(Employee, [Engineer, Manager])`作为主要实体：

```py
print(select(Employee))
SELECT  employee.id,  employee.name,  employee.type,  engineer.id  AS  id_1,
engineer.engineer_info,  manager.id  AS  id_2,  manager.manager_name
FROM  employee
LEFT  OUTER  JOIN  engineer  ON  employee.id  =  engineer.id
LEFT  OUTER  JOIN  manager  ON  employee.id  =  manager.id 
```

当使用映射器级“with polymorphic”时，查询也可以直接引用子类实体，其中它们隐式地表示多态查询中的连接表。在上面的例子中，我们可以自由地针对默认的`Employee`实体直接引用`Manager`和`Engineer`：

```py
print(
 select(Employee).where(
 or_(Manager.manager_name == "x", Engineer.engineer_info == "y")
 )
)
SELECT  employee.id,  employee.name,  employee.type,  engineer.id  AS  id_1,
engineer.engineer_info,  manager.id  AS  id_2,  manager.manager_name
FROM  employee
LEFT  OUTER  JOIN  engineer  ON  employee.id  =  engineer.id
LEFT  OUTER  JOIN  manager  ON  employee.id  =  manager.id
WHERE  manager.manager_name  =  :manager_name_1
OR  engineer.engineer_info  =  :engineer_info_1 
```

但是，如果我们需要在单独的别名上下文中引用`Employee`实体或其子实体，我们将再次直接使用`with_polymorphic()` 来定义这些别名实体，如使用 with_polymorphic 进行别名处理中所示。

为了更集中地控制多态可选项，可以使用更传统的映射器级多态控制形式，即`Mapper.with_polymorphic` 参数，配置在基类上。此参数接受的参数与`with_polymorphic()` 构造类似，但在联合继承映射中的常见用法是纯星号，表示所有子表应该 LEFT OUTER JOIN，如下所示：

```py
class Employee(Base):
    __tablename__ = "employee"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    type = mapped_column(String(50))

    __mapper_args__ = {
        "polymorphic_identity": "employee",
        "with_polymorphic": "*",
        "polymorphic_on": type,
    }

class Engineer(Employee):
    __tablename__ = "engineer"
    id = mapped_column(Integer, ForeignKey("employee.id"), primary_key=True)
    engineer_info = mapped_column(String(30))

    __mapper_args__ = {
        "polymorphic_identity": "engineer",
    }

class Manager(Employee):
    __tablename__ = "manager"
    id = mapped_column(Integer, ForeignKey("employee.id"), primary_key=True)
    manager_name = mapped_column(String(30))

    __mapper_args__ = {
        "polymorphic_identity": "manager",
    }
```

总的来说，`with_polymorphic()` 和 `Mapper.with_polymorphic` 等选项所使用的 LEFT OUTER JOIN 格式从 SQL 和数据库优化器的角度来看可能比较繁琐；对于联合继承映射中子类属性的一般加载，可能更倾向于使用 `selectin_polymorphic()` 方法，或者将其映射级别等效设置为 `Mapper.polymorphic_load` 为`"selectin"`，仅在需要时基于每个查询使用 `with_polymorphic()`。  ### 使用 with_polymorphic 过滤子类属性

`with_polymorphic()` 构造使得包含的子类映射器上的属性可用，通过包含允许引用子类的命名空间。在上一节中创建的`employee_poly`构造包括名为`.Engineer`和`.Manager`的属性，它们为多态 SELECT 提供了`Engineer`和`Manager`的命名空间。在下面的示例中，我们可以使用`or_()` 构造同时对两个类创建条件：

```py
>>> from sqlalchemy import or_
>>> employee_poly = with_polymorphic(Employee, [Engineer, Manager])
>>> stmt = (
...     select(employee_poly)
...     .where(
...         or_(
...             employee_poly.Manager.manager_name == "Eugene H. Krabs",
...             employee_poly.Engineer.engineer_info
...             == "Senior Customer Engagement Engineer",
...         )
...     )
...     .order_by(employee_poly.id)
... )
>>> objects = session.scalars(stmt).all()
SELECT  employee.id,  employee.name,  employee.type,  employee.company_id,  manager.id  AS  id_1,
manager.manager_name,  engineer.id  AS  id_2,  engineer.engineer_info
FROM  employee
LEFT  OUTER  JOIN  manager  ON  employee.id  =  manager.id
LEFT  OUTER  JOIN  engineer  ON  employee.id  =  engineer.id
WHERE  manager.manager_name  =  ?  OR  engineer.engineer_info  =  ?
ORDER  BY  employee.id
[...]  ('Eugene H. Krabs',  'Senior Customer Engagement Engineer')
>>> print(objects)
[Manager('Mr. Krabs'), Engineer('Squidward')]
```

### 使用 with_polymorphic 进行别名处理

`with_polymorphic()`构造，作为`aliased()`的一个特例，还提供了`aliased()`的基本特性，即多态可选项本身的“别名”。具体来说，这意味着两个或多个指向相同类层次结构的`with_polymorphic()`实体可以同时在单个语句中使用。

要在连接继承映射中使用此功能，通常我们希望传递两个参数，`with_polymorphic.aliased`以及`with_polymorphic.flat`。`with_polymorphic.aliased`参数指示多态可选项应该被引用为此结构唯一的别名。`with_polymorphic.flat`参数是特定于默认的 LEFT OUTER JOIN 多态可选项，并指示在语句中应该使用更优化的别名形式。

为了说明这个特性，下面的示例发出了一个选择两个单独的多态实体，`Employee`与`Engineer`连接，以及`Employee`与`Manager`连接。由于这两个多态实体都将包含基本的`employee`表在其多态可选项中，必须应用别名以便在它们的两个不同的上下文中区分此表。这两个多态实体被视为两个单独的表，并且通常需要以某种方式相互连接，如下所示，在`company_id`列上与`Employee` / `Manager`实体一起加入了一些额外的限制条件：

```py
>>> manager_employee = with_polymorphic(Employee, [Manager], aliased=True, flat=True)
>>> engineer_employee = with_polymorphic(Employee, [Engineer], aliased=True, flat=True)
>>> stmt = (
...     select(manager_employee, engineer_employee)
...     .join(
...         engineer_employee,
...         engineer_employee.company_id == manager_employee.company_id,
...     )
...     .where(
...         or_(
...             manager_employee.name == "Mr. Krabs",
...             manager_employee.Manager.manager_name == "Eugene H. Krabs",
...         )
...     )
...     .order_by(engineer_employee.name, manager_employee.name)
... )
>>> for manager, engineer in session.execute(stmt):
...     print(f"{manager} {engineer}")
SELECT
employee_1.id,  employee_1.name,  employee_1.type,  employee_1.company_id,
manager_1.id  AS  id_1,  manager_1.manager_name,
employee_2.id  AS  id_2,  employee_2.name  AS  name_1,  employee_2.type  AS  type_1,
employee_2.company_id  AS  company_id_1,  engineer_1.id  AS  id_3,  engineer_1.engineer_info
FROM  employee  AS  employee_1
LEFT  OUTER  JOIN  manager  AS  manager_1  ON  employee_1.id  =  manager_1.id
JOIN
  (employee  AS  employee_2  LEFT  OUTER  JOIN  engineer  AS  engineer_1  ON  employee_2.id  =  engineer_1.id)
ON  employee_2.company_id  =  employee_1.company_id
WHERE  employee_1.name  =  ?  OR  manager_1.manager_name  =  ?
ORDER  BY  employee_2.name,  employee_1.name
[...]  ('Mr. Krabs',  'Eugene H. Krabs')
Manager('Mr. Krabs') Manager('Mr. Krabs')
Manager('Mr. Krabs') Engineer('SpongeBob')
Manager('Mr. Krabs') Engineer('Squidward')
```

在上面的示例中，`with_polymorphic.flat`的行为是，多态的可选项仍然保持为它们各自表的 LEFT OUTER JOIN，并且它们本身被赋予匿名别名。还会产生右嵌套的 JOIN。

当省略`with_polymorphic.flat`参数时，通常行为是每个多态可选项被封装在一个子查询中，生成更详细的形式：

```py
>>> manager_employee = with_polymorphic(Employee, [Manager], aliased=True)
>>> engineer_employee = with_polymorphic(Employee, [Engineer], aliased=True)
>>> stmt = (
...     select(manager_employee, engineer_employee)
...     .join(
...         engineer_employee,
...         engineer_employee.company_id == manager_employee.company_id,
...     )
...     .where(
...         or_(
...             manager_employee.name == "Mr. Krabs",
...             manager_employee.Manager.manager_name == "Eugene H. Krabs",
...         )
...     )
...     .order_by(engineer_employee.name, manager_employee.name)
... )
>>> print(stmt)
SELECT  anon_1.employee_id,  anon_1.employee_name,  anon_1.employee_type,
anon_1.employee_company_id,  anon_1.manager_id,  anon_1.manager_manager_name,  anon_2.employee_id  AS  employee_id_1,
anon_2.employee_name  AS  employee_name_1,  anon_2.employee_type  AS  employee_type_1,
anon_2.employee_company_id  AS  employee_company_id_1,  anon_2.engineer_id,  anon_2.engineer_engineer_info
FROM
(SELECT  employee.id  AS  employee_id,  employee.name  AS  employee_name,  employee.type  AS  employee_type,
employee.company_id  AS  employee_company_id,
manager.id  AS  manager_id,  manager.manager_name  AS  manager_manager_name
FROM  employee  LEFT  OUTER  JOIN  manager  ON  employee.id  =  manager.id)  AS  anon_1
JOIN
(SELECT  employee.id  AS  employee_id,  employee.name  AS  employee_name,  employee.type  AS  employee_type,
employee.company_id  AS  employee_company_id,  engineer.id  AS  engineer_id,  engineer.engineer_info  AS  engineer_engineer_info
FROM  employee  LEFT  OUTER  JOIN  engineer  ON  employee.id  =  engineer.id)  AS  anon_2
ON  anon_2.employee_company_id  =  anon_1.employee_company_id
WHERE  anon_1.employee_name  =  :employee_name_2  OR  anon_1.manager_manager_name  =  :manager_manager_name_1
ORDER  BY  anon_2.employee_name,  anon_1.employee_name 
```

上述形式在历史上更容易移植到不一定支持右嵌套 JOIN 的后端，并且在使用`with_polymorphic()`时，“多态选择”不是简单的左外连接表时，例如使用具体表继承映射以及一般情况下使用替代多态选择时，可能更合适。

### 在映射器上配置 with_polymorphic()

与`selectin_polymorphic()`一样，`with_polymorphic()`构造也支持一个通过映射器配置的版本，可以通过两种不同的方式配置，一种是在基类上使用`mapper.with_polymorphic`参数，另一种是在每个子类上使用更现代的形式，在`Mapper.polymorphic_load`参数上传递值`"inline"`。

警告

对于加入继承映射，请优先在查询中显式使用`with_polymorphic()`，或者对于隐式急切子类加载，请使用`Mapper.polymorphic_load`与`"selectin"`，而不是使用本节中描述的映射器级别的`mapper.with_polymorphic`参数。此参数调用复杂的启发式算法，旨在重写 SELECT 语句中的 FROM 子句，可能会干扰更复杂语句的构建，特别是那些引用相同映射实体的嵌套子查询。

例如，我们可以使用`Mapper.polymorphic_load`将我们的`Employee`映射状态声明为`"inline"`，如下所示：

```py
class Employee(Base):
    __tablename__ = "employee"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    type = mapped_column(String(50))

    __mapper_args__ = {"polymorphic_identity": "employee", "polymorphic_on": type}

class Engineer(Employee):
    __tablename__ = "engineer"
    id = mapped_column(Integer, ForeignKey("employee.id"), primary_key=True)
    engineer_info = mapped_column(String(30))

    __mapper_args__ = {
        "polymorphic_load": "inline",
        "polymorphic_identity": "engineer",
    }

class Manager(Employee):
    __tablename__ = "manager"
    id = mapped_column(Integer, ForeignKey("employee.id"), primary_key=True)
    manager_name = mapped_column(String(30))

    __mapper_args__ = {
        "polymorphic_load": "inline",
        "polymorphic_identity": "manager",
    }
```

有了上述映射，针对`Employee`类的 SELECT 语句在发出语句时将自动假定使用`with_polymorphic(Employee, [Engineer, Manager])`作为主要实体：

```py
print(select(Employee))
SELECT  employee.id,  employee.name,  employee.type,  engineer.id  AS  id_1,
engineer.engineer_info,  manager.id  AS  id_2,  manager.manager_name
FROM  employee
LEFT  OUTER  JOIN  engineer  ON  employee.id  =  engineer.id
LEFT  OUTER  JOIN  manager  ON  employee.id  =  manager.id 
```

当使用映射器级别的“多态”时，查询还可以直接引用子类实体，在这里它们隐式地代表了多态查询中的联接表。在上面的示例中，我们可以自由地直接引用`Manager`和`Engineer`对默认的`Employee`实体进行查询：

```py
print(
 select(Employee).where(
 or_(Manager.manager_name == "x", Engineer.engineer_info == "y")
 )
)
SELECT  employee.id,  employee.name,  employee.type,  engineer.id  AS  id_1,
engineer.engineer_info,  manager.id  AS  id_2,  manager.manager_name
FROM  employee
LEFT  OUTER  JOIN  engineer  ON  employee.id  =  engineer.id
LEFT  OUTER  JOIN  manager  ON  employee.id  =  manager.id
WHERE  manager.manager_name  =  :manager_name_1
OR  engineer.engineer_info  =  :engineer_info_1 
```

但是，如果我们需要在单独的别名上下文中引用`Employee`实体或其子实体，我们将再次直接使用`with_polymorphic()`来定义这些别名实体，如使用别名与 with_polymorphic 中所示。

为了更加集中地控制多态可选择的内容，可以使用更为传统的映射器级别多态控制形式，即`Mapper.with_polymorphic`参数，配置在基类上。该参数接受与`with_polymorphic()`构造类似的参数，但在连接继承映射中的常见用法是使用纯*号，表示应该 LEFT OUTER JOIN 所有子表，例如：

```py
class Employee(Base):
    __tablename__ = "employee"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String(50))
    type = mapped_column(String(50))

    __mapper_args__ = {
        "polymorphic_identity": "employee",
        "with_polymorphic": "*",
        "polymorphic_on": type,
    }

class Engineer(Employee):
    __tablename__ = "engineer"
    id = mapped_column(Integer, ForeignKey("employee.id"), primary_key=True)
    engineer_info = mapped_column(String(30))

    __mapper_args__ = {
        "polymorphic_identity": "engineer",
    }

class Manager(Employee):
    __tablename__ = "manager"
    id = mapped_column(Integer, ForeignKey("employee.id"), primary_key=True)
    manager_name = mapped_column(String(30))

    __mapper_args__ = {
        "polymorphic_identity": "manager",
    }
```

总的来说，`with_polymorphic()`所使用的 LEFT OUTER JOIN 格式以及诸如`Mapper.with_polymorphic`等选项，从 SQL 和数据库优化器的角度来看可能有些繁琐；对于连接继承映射中子类属性的一般加载，应该优先考虑`selectin_polymorphic()`方法，或者在映射器级别设置`Mapper.polymorphic_load`为`"selectin"`，只在需要时在每个查询中使用`with_polymorphic()`。

## 连接到特定子类型或 with_polymorphic()实体

由于`with_polymorphic()`实体是`aliased()`的一个特殊情况，为了将多态实体视为连接的目标，特别是在使用`relationship()`构造作为 ON 子句时，我们使用与常规别名相同的技术，如 Using Relationship to join between aliased targets 中详细描述的，最简洁的方法是使用`PropComparator.of_type()`。在下面的示例中，我们说明了从父`Company`实体沿着一对多关系`Company.employees`进行连接，该关系在 setup 中配置为链接到`Employee`对象，使用`with_polymorphic()`实体作为目标：

```py
>>> employee_plus_engineer = with_polymorphic(Employee, [Engineer])
>>> stmt = (
...     select(Company.name, employee_plus_engineer.name)
...     .join(Company.employees.of_type(employee_plus_engineer))
...     .where(
...         or_(
...             employee_plus_engineer.name == "SpongeBob",
...             employee_plus_engineer.Engineer.engineer_info
...             == "Senior Customer Engagement Engineer",
...         )
...     )
... )
>>> for company_name, emp_name in session.execute(stmt):
...     print(f"{company_name} {emp_name}")
SELECT  company.name,  employee.name  AS  name_1
FROM  company  JOIN  (employee  LEFT  OUTER  JOIN  engineer  ON  employee.id  =  engineer.id)  ON  company.id  =  employee.company_id
WHERE  employee.name  =  ?  OR  engineer.engineer_info  =  ?
[...]  ('SpongeBob',  'Senior Customer Engagement Engineer')
Krusty Krab SpongeBob
Krusty Krab Squidward
```

更直接地，`PropComparator.of_type()` 也用于任何类型的继承映射，以限制 `relationship()` 中的连接到 `relationship()` 的特定子类型。上述查询可以严格按照 `Engineer` 目标编写如下：

```py
>>> stmt = (
...     select(Company.name, Engineer.name)
...     .join(Company.employees.of_type(Engineer))
...     .where(
...         or_(
...             Engineer.name == "SpongeBob",
...             Engineer.engineer_info == "Senior Customer Engagement Engineer",
...         )
...     )
... )
>>> for company_name, emp_name in session.execute(stmt):
...     print(f"{company_name} {emp_name}")
SELECT  company.name,  employee.name  AS  name_1
FROM  company  JOIN  (employee  JOIN  engineer  ON  employee.id  =  engineer.id)  ON  company.id  =  employee.company_id
WHERE  employee.name  =  ?  OR  engineer.engineer_info  =  ?
[...]  ('SpongeBob',  'Senior Customer Engagement Engineer')
Krusty Krab SpongeBob
Krusty Krab Squidward
```

如上所示，直接加入到 `Engineer` 目标，而不是 `with_polymorphic(Employee, [Engineer])` 的 “多态可选择项”，具有使用内连接而不是左外连接的有用特性，从 SQL 优化器的角度来看，通常更具性能。

### 多态子类型的急加载

使用 `PropComparator.of_type()` 方法的示例见前一节中的 `Select.join()` 方法，同样可以等效地应用于关系加载器选项，例如 `selectinload()` 和 `joinedload()`。

作为基本示例，如果我们希望加载 `Company` 对象，并且另外急加载 `Company.employees` 的所有元素，使用 `with_polymorphic()` 构造针对完整层次结构，我们可以编写如下：

```py
>>> all_employees = with_polymorphic(Employee, "*")
>>> stmt = select(Company).options(selectinload(Company.employees.of_type(all_employees)))
>>> for company in session.scalars(stmt):
...     print(f"company: {company.name}")
...     print(f"employees: {company.employees}")
SELECT  company.id,  company.name
FROM  company
[...]  ()
SELECT  employee.company_id  AS  employee_company_id,  employee.id  AS  employee_id,
employee.name  AS  employee_name,  employee.type  AS  employee_type,  manager.id  AS  manager_id,
manager.manager_name  AS  manager_manager_name,  engineer.id  AS  engineer_id,
engineer.engineer_info  AS  engineer_engineer_info
FROM  employee
LEFT  OUTER  JOIN  manager  ON  employee.id  =  manager.id
LEFT  OUTER  JOIN  engineer  ON  employee.id  =  engineer.id
WHERE  employee.company_id  IN  (?)
[...]  (1,)
company:  Krusty  Krab
employees:  [Manager('Mr. Krabs'),  Engineer('SpongeBob'),  Engineer('Squidward')] 
```

上述查询可直接与前一节中演示的 `selectin_polymorphic()` 版本进行比较 将 selectin_polymorphic() 应用于现有急加载。

另见

将 selectin_polymorphic() 应用于现有急加载 - 演示了与上述相同的等效示例，使用 `selectin_polymorphic()` 替代 ### 多态子类型的急加载

使用`PropComparator.of_type()`方法，如前一节中的`Select.join()`方法所示，也可以等效地应用于关系加载器选项，例如`selectinload()`和`joinedload()`。

作为基本示例，如果我们希望加载`Company`对象，并使用`with_polymorphic()`构造针对完整层次结构的，同时急切地加载`Company.employees`的所有元素，我们可以写成：

```py
>>> all_employees = with_polymorphic(Employee, "*")
>>> stmt = select(Company).options(selectinload(Company.employees.of_type(all_employees)))
>>> for company in session.scalars(stmt):
...     print(f"company: {company.name}")
...     print(f"employees: {company.employees}")
SELECT  company.id,  company.name
FROM  company
[...]  ()
SELECT  employee.company_id  AS  employee_company_id,  employee.id  AS  employee_id,
employee.name  AS  employee_name,  employee.type  AS  employee_type,  manager.id  AS  manager_id,
manager.manager_name  AS  manager_manager_name,  engineer.id  AS  engineer_id,
engineer.engineer_info  AS  engineer_engineer_info
FROM  employee
LEFT  OUTER  JOIN  manager  ON  employee.id  =  manager.id
LEFT  OUTER  JOIN  engineer  ON  employee.id  =  engineer.id
WHERE  employee.company_id  IN  (?)
[...]  (1,)
company:  Krusty  Krab
employees:  [Manager('Mr. Krabs'),  Engineer('SpongeBob'),  Engineer('Squidward')] 
```

上述查询可以直接与前一节中将 selectin_polymorphic()应用于现有急切加载中所示的`selectin_polymorphic()`版本进行比较。

另请参阅

将 selectin_polymorphic()应用于现有急切加载 - 演示了与上述相同的例子，但使用了`selectin_polymorphic()`代替

## 单一继承映射的 SELECT 语句

单一表继承设置

本节讨论单一表继承，描述在单一表继承中使用单个表来表示层次结构中的多个类。

查看此部分的 ORM 设置。

与连接继承映射相比，为单一继承映射构造 SELECT 语句通常更简单，因为对于全单一继承层次结构，只有一个表。

无论继承层次结构是全单一继承还是具有混合连接和单一继承，单一继承的 SELECT 语句通过使用附加的 WHERE 条件限制 SELECT 语句来区分对基类和子类的查询。

例如，针对`Employee`的单一继承示例映射的查询将使用表的简单 SELECT 来加载`Manager`、`Engineer`和`Employee`类型的对象：

```py
>>> stmt = select(Employee).order_by(Employee.id)
>>> for obj in session.scalars(stmt):
...     print(f"{obj}")
BEGIN  (implicit)
SELECT  employee.id,  employee.name,  employee.type
FROM  employee  ORDER  BY  employee.id
[...]  ()
Manager('Mr. Krabs')
Engineer('SpongeBob')
Engineer('Squidward')
```

当为特定子类发出加载时，将向 SELECT 中添加附加条件以限制行，例如在下面执行针对`Engineer`实体的 SELECT：

```py
>>> stmt = select(Engineer).order_by(Engineer.id)
>>> objects = session.scalars(stmt).all()
SELECT  employee.id,  employee.name,  employee.type,  employee.engineer_info
FROM  employee
WHERE  employee.type  IN  (?)  ORDER  BY  employee.id
[...]  ('engineer',)
>>> for obj in objects:
...     print(f"{obj}")
Engineer('SpongeBob')
Engineer('Squidward')
```

### 优化单一继承的属性加载

单继承映射关于如何 SELECT 子类上的属性的默认行为类似于连接继承的行为，即子类特定的属性默认情况下仍然会发出第二个 SELECT。在下面的示例中，加载了类型为 `Manager` 的单个 `Employee`，但是由于请求的类是 `Employee`，所以 `Manager.manager_name` 属性默认情况下不会存在，并且在访问时会发出额外的 SELECT：

```py
>>> mr_krabs = session.scalars(select(Employee).where(Employee.name == "Mr. Krabs")).one()
BEGIN  (implicit)
SELECT  employee.id,  employee.name,  employee.type
FROM  employee
WHERE  employee.name  =  ?
[...]  ('Mr. Krabs',)
>>> mr_krabs.manager_name
SELECT  employee.manager_name  AS  employee_manager_name
FROM  employee
WHERE  employee.id  =  ?  AND  employee.type  IN  (?)
[...]  (1,  'manager')
'Eugene H. Krabs'
```

要改变这种行为，对于单继承，与连接继承加载中使用的相同的一般概念也适用于急切地加载这些额外属性，包括使用 `selectin_polymorphic()` 选项以及 `with_polymorphic()` 选项，后者只是简单地包含了额外的列，并且从 SQL 视角来看对于单继承映射更为高效：

```py
>>> employees = with_polymorphic(Employee, "*")
>>> stmt = select(employees).order_by(employees.id)
>>> objects = session.scalars(stmt).all()
BEGIN  (implicit)
SELECT  employee.id,  employee.name,  employee.type,
employee.manager_name,  employee.engineer_info
FROM  employee  ORDER  BY  employee.id
[...]  ()
>>> for obj in objects:
...     print(f"{obj}")
Manager('Mr. Krabs')
Engineer('SpongeBob')
Engineer('Squidward')
>>> objects[0].manager_name
'Eugene H. Krabs'
```

由于加载单继承子类映射的开销通常很小，因此建议单继承映射在那些预计其特定子类属性加载是常见的子类中包含 `Mapper.polymorphic_load` 参数，并将其设置为 `"inline"`。下面是一个修改后的示例，说明了这个设置的一个示例：

```py
>>> class Base(DeclarativeBase):
...     pass
>>> class Employee(Base):
...     __tablename__ = "employee"
...     id: Mapped[int] = mapped_column(primary_key=True)
...     name: Mapped[str]
...     type: Mapped[str]
...
...     def __repr__(self):
...         return f"{self.__class__.__name__}({self.name!r})"
...
...     __mapper_args__ = {
...         "polymorphic_identity": "employee",
...         "polymorphic_on": "type",
...     }
>>> class Manager(Employee):
...     manager_name: Mapped[str] = mapped_column(nullable=True)
...     __mapper_args__ = {
...         "polymorphic_identity": "manager",
...         "polymorphic_load": "inline",
...     }
>>> class Engineer(Employee):
...     engineer_info: Mapped[str] = mapped_column(nullable=True)
...     __mapper_args__ = {
...         "polymorphic_identity": "engineer",
...         "polymorphic_load": "inline",
...     }
```

有了上面的映射，`Manager` 和 `Engineer` 类将自动在针对 `Employee` 实体的 SELECT 语句中包含它们的列：

```py
>>> print(select(Employee))
SELECT  employee.id,  employee.name,  employee.type,
employee.manager_name,  employee.engineer_info
FROM  employee 
```

### 优化单继承属性加载

单继承映射关于如何 SELECT 子类上的属性的默认行为类似于连接继承的行为，即子类特定的属性默认情况下仍然会发出第二个 SELECT。在下面的示例中，加载了类型为 `Manager` 的单个 `Employee`，但是由于请求的类是 `Employee`，所以 `Manager.manager_name` 属性默认情况下不会存在，并且在访问时会发出额外的 SELECT：

```py
>>> mr_krabs = session.scalars(select(Employee).where(Employee.name == "Mr. Krabs")).one()
BEGIN  (implicit)
SELECT  employee.id,  employee.name,  employee.type
FROM  employee
WHERE  employee.name  =  ?
[...]  ('Mr. Krabs',)
>>> mr_krabs.manager_name
SELECT  employee.manager_name  AS  employee_manager_name
FROM  employee
WHERE  employee.id  =  ?  AND  employee.type  IN  (?)
[...]  (1,  'manager')
'Eugene H. Krabs'
```

要改变这种行为，对于单继承，与连接继承加载中使用的相同的一般概念也适用于急切地加载这些额外属性，包括使用 `selectin_polymorphic()` 选项以及 `with_polymorphic()` 选项，后者只是简单地包含了额外的列，并且从 SQL 视角来看对于单继承映射更为高效：

```py
>>> employees = with_polymorphic(Employee, "*")
>>> stmt = select(employees).order_by(employees.id)
>>> objects = session.scalars(stmt).all()
BEGIN  (implicit)
SELECT  employee.id,  employee.name,  employee.type,
employee.manager_name,  employee.engineer_info
FROM  employee  ORDER  BY  employee.id
[...]  ()
>>> for obj in objects:
...     print(f"{obj}")
Manager('Mr. Krabs')
Engineer('SpongeBob')
Engineer('Squidward')
>>> objects[0].manager_name
'Eugene H. Krabs'
```

由于加载单继承子类映射的开销通常很小，因此建议在那些预计其特定子类属性的加载是常见的单继承映射中，将`Mapper.polymorphic_load`参数设置为`"inline"`。下面是一个示例，演示了 setup 如何修改以包含此选项：

```py
>>> class Base(DeclarativeBase):
...     pass
>>> class Employee(Base):
...     __tablename__ = "employee"
...     id: Mapped[int] = mapped_column(primary_key=True)
...     name: Mapped[str]
...     type: Mapped[str]
...
...     def __repr__(self):
...         return f"{self.__class__.__name__}({self.name!r})"
...
...     __mapper_args__ = {
...         "polymorphic_identity": "employee",
...         "polymorphic_on": "type",
...     }
>>> class Manager(Employee):
...     manager_name: Mapped[str] = mapped_column(nullable=True)
...     __mapper_args__ = {
...         "polymorphic_identity": "manager",
...         "polymorphic_load": "inline",
...     }
>>> class Engineer(Employee):
...     engineer_info: Mapped[str] = mapped_column(nullable=True)
...     __mapper_args__ = {
...         "polymorphic_identity": "engineer",
...         "polymorphic_load": "inline",
...     }
```

有了上面的映射，`Manager`和`Engineer`类的列将自动包含在针对`Employee`实体的 SELECT 语句中：

```py
>>> print(select(Employee))
SELECT  employee.id,  employee.name,  employee.type,
employee.manager_name,  employee.engineer_info
FROM  employee 
```

## 继承加载 API

| 对象名称 | 描述 |
| --- | --- |
| selectin_polymorphic(base_cls, classes) | 指示应针对特定子类的所有属性进行急切加载。 |
| with_polymorphic(base, classes[, selectable, flat, ...]) | 产生一个`AliasedClass`构造，该构造指定了给定基类的后代映射器的列。 |

```py
function sqlalchemy.orm.with_polymorphic(base: Type[_O] | Mapper[_O], classes: Literal['*'] | Iterable[Type[Any]], selectable: Literal[False, None] | FromClause = False, flat: bool = False, polymorphic_on: ColumnElement[Any] | None = None, aliased: bool = False, innerjoin: bool = False, adapt_on_names: bool = False, _use_mapper_path: bool = False) → AliasedClass[_O]
```

产生一个`AliasedClass`构造，该构造指定了给定基类的后代映射器的列。

使用这种方法将确保每个子类映射器的表都包含在 FROM 子句中，并允许对这些表使用 filter()条件。结果实例也将已经加载了那些列，因此不需要对这些列进行“后获取”。

请参阅

使用 with_polymorphic() - `with_polymorphic()`的全面讨论。

参数：

+   `base` – 要别名化的基类。

+   `classes` – 单个类或映射器，或者继承自基类的类/映射器列表。或者，它也可以是字符串`'*'`，在这种情况下，所有下降的映射类将被添加到 FROM 子句中。

+   `aliased` – 当为 True 时，可选择的将被别名。对于 JOIN，这意味着 JOIN 将从子查询中 SELECT，除非设置了`with_polymorphic.flat`标志为 True，这对于更简单的用例是推荐的。

+   `flat` – 布尔值，将被传递到 `FromClause.alias()` 调用，以便 `Join` 对象的别名将别名为加入内的各个表，而不是创建子查询。这通常受到所有现代数据库的支持，关于右嵌套连接，通常会生成更有效的查询。只要生成的 SQL 有效，建议设置此标志。

+   `selectable` –

    将用于替代生成的 FROM 子句的表或子查询。如果所需的任何类使用具体表继承，这个参数是必需的，因为 SQLAlchemy 目前无法自动在表之间生成 UNION。如果使用，`selectable` 参数必须表示每个映射类映射的所有表和列的完整集。否则，未解释的映射列将直接附加到 FROM 子句，这通常会导致结果不正确。

    当将其保留在默认值 `False` 时，分配给基本 mapper 的多态可选择将用于选择行。但是，它也可以传递为 `None`，这将绕过配置的多态可选择，而是为给定的目标类构造一个临时选择; 对于联接表继承，这将是包含所有目标映射器及其子类的联接。

+   `polymorphic_on` – 作为给定可选择对象的“鉴别器”列使用的列。如果未给出，则将使用基类的 mapper 的 polymorphic_on 属性（如果有）。这对于默认不具有多态加载行为的映射非常有用。

+   `innerjoin` – 如果为 True，将使用 INNER JOIN。只有在仅查询一个特定子类型时才应指定此选项

+   `adapt_on_names` –

    将 `aliased.adapt_on_names` 参数传递给别名对象。在给定可选择对象与现有映射可选择对象不直接相关的情况下，这可能会很有用。

    新版本 1.4.33 中新增。

```py
function sqlalchemy.orm.selectin_polymorphic(base_cls: _EntityType[Any], classes: Iterable[Type[Any]]) → _AbstractLoad
```

表示应该对子类特定的所有属性进行急切加载。

这使用了一个额外的 SELECT 与所有匹配的主键值进行 IN 操作，它是对 `mapper.polymorphic_load` 参数上的 `"selectin"` 设置的每个查询的类似。

新版本 1.2 中新增。

另请参见

使用 selectin_polymorphic()
