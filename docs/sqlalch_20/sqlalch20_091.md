# 定义约束和索引

> 原文：[`docs.sqlalchemy.org/en/20/core/constraints.html`](https://docs.sqlalchemy.org/en/20/core/constraints.html)

这一部分将讨论 SQL 的约束和索引。在 SQLAlchemy 中，关键类包括`ForeignKeyConstraint`和`Index`。

## 定义外键

SQL 中的*外键*是一个表级构造，它将该表中的一个或多个列约束为仅允许存在于另一组列中的值，通常但不总是位于不同的表上。我们称被约束的列为*外键*列，它们被约束到的列为*引用*列。引用列几乎总是定义其拥有表的主键，尽管也有例外情况。外键是连接具有关系的行对的“关节”，SQLAlchemy 在其几乎每个操作的每个区域都赋予了这个概念非常深的重要性。

在 SQLAlchemy 中以及在 DDL 中，外键约束可以被定义为表子句中的附加属性，或者对于单列外键，它们可以选择地在单列的定义中指定。单列外键更常见，在列级别上是通过将`ForeignKey`对象构造为`Column`对象的参数来指定的：

```py
user_preference = Table(
    "user_preference",
    metadata_obj,
    Column("pref_id", Integer, primary_key=True),
    Column("user_id", Integer, ForeignKey("user.user_id"), nullable=False),
    Column("pref_name", String(40), nullable=False),
    Column("pref_value", String(100)),
)
```

上面，我们定义了一个新表`user_preference`，其中每一行必须包含一个存在于`user`表的`user_id`列中的值。

参数`ForeignKey`最常见的形式是一个字符串，格式为*<tablename>.<columnname>*，或者对于远程模式或者“owner”的表格是*<schemaname>.<tablename>.<columnname>*。它也可以是一个实际的`Column`对象，稍后我们将看到它是通过其`c`集合从现有的`Table`对象中访问的：

```py
ForeignKey(user.c.user_id)
```

使用字符串的优点是，在首次需要时才解析`user`和`user_preference`之间的 Python 链接，因此表对象可以轻松地分布在多个模块中并按任何顺序定义。

外键也可以在表级别使用 `ForeignKeyConstraint` 对象定义。此对象可以描述单列或多列外键。多列外键称为*复合*外键，几乎总是引用具有复合主键的表。下面我们定义一个具有复合主键的表 `invoice`：

```py
invoice = Table(
    "invoice",
    metadata_obj,
    Column("invoice_id", Integer, primary_key=True),
    Column("ref_num", Integer, primary_key=True),
    Column("description", String(60), nullable=False),
)
```

然后是一个引用 `invoice` 的复合外键的表 `invoice_item`：

```py
invoice_item = Table(
    "invoice_item",
    metadata_obj,
    Column("item_id", Integer, primary_key=True),
    Column("item_name", String(60), nullable=False),
    Column("invoice_id", Integer, nullable=False),
    Column("ref_num", Integer, nullable=False),
    ForeignKeyConstraint(
        ["invoice_id", "ref_num"], ["invoice.invoice_id", "invoice.ref_num"]
    ),
)
```

注意，`ForeignKeyConstraint` 是定义复合外键的唯一方式。虽然我们也可以将单独的 `ForeignKey` 对象放置在 `invoice_item.invoice_id` 和 `invoice_item.ref_num` 列上，但 SQLAlchemy 不会意识到这两个值应该配对在一起 - 它将是两个单独的外键约束，而不是一个引用两个列的单个复合外键。

### 通过 ALTER 创建/删除外键约束

我们在教程和其他地方看到的涉及 DDL 的外键行为表明，约束通常以“内联”的方式在 CREATE TABLE 语句中呈现，例如：

```py
CREATE  TABLE  addresses  (
  id  INTEGER  NOT  NULL,
  user_id  INTEGER,
  email_address  VARCHAR  NOT  NULL,
  PRIMARY  KEY  (id),
  CONSTRAINT  user_id_fk  FOREIGN  KEY(user_id)  REFERENCES  users  (id)
)
```

`CONSTRAINT .. FOREIGN KEY` 指令用于在 CREATE TABLE 定义中以“内联”的方式创建约束。 `MetaData.create_all()` 和 `MetaData.drop_all()` 方法默认使用所有涉及的 `Table` 对象的拓扑排序，以便按照它们的外键依赖顺序创建和删除表（此排序也可通过 `MetaData.sorted_tables` 访问器获取）。

当涉及两个或更多外键约束参与“依赖循环”时，这种方法无法工作，其中一组表彼此相互依赖，假设后端执行外键（除了 SQLite、MySQL/MyISAM 之外总是是这样的情况）。因此，这些方法将在除 SQLite 之外的所有后端上将循环中的约束拆分为单独的 ALTER 语句，SQLite 不支持大多数形式的 ALTER。给定这样的模式：

```py
node = Table(
    "node",
    metadata_obj,
    Column("node_id", Integer, primary_key=True),
    Column("primary_element", Integer, ForeignKey("element.element_id")),
)

element = Table(
    "element",
    metadata_obj,
    Column("element_id", Integer, primary_key=True),
    Column("parent_node_id", Integer),
    ForeignKeyConstraint(
        ["parent_node_id"], ["node.node_id"], name="fk_element_parent_node_id"
    ),
)
```

当我们在像 PostgreSQL 这样的后端上调用 `MetaData.create_all()` 时，这两个表之间的循环被解析，并且约束被分别创建：

```py
>>> with engine.connect() as conn:
...     metadata_obj.create_all(conn, checkfirst=False)
CREATE  TABLE  element  (
  element_id  SERIAL  NOT  NULL,
  parent_node_id  INTEGER,
  PRIMARY  KEY  (element_id)
)

CREATE  TABLE  node  (
  node_id  SERIAL  NOT  NULL,
  primary_element  INTEGER,
  PRIMARY  KEY  (node_id)
)

ALTER  TABLE  element  ADD  CONSTRAINT  fk_element_parent_node_id
  FOREIGN  KEY(parent_node_id)  REFERENCES  node  (node_id)
ALTER  TABLE  node  ADD  FOREIGN  KEY(primary_element)
  REFERENCES  element  (element_id) 
```

为了发出这些表的 DROP，相同的逻辑也适用，但是请注意，在 SQL 中，发出 DROP CONSTRAINT 需要约束具有名称。在上面的 `'node'` 表的情况下，我们还没有为此约束命名；因此系统将仅尝试发出对具有名称的约束的 DROP：

```py
>>> with engine.connect() as conn:
...     metadata_obj.drop_all(conn, checkfirst=False)
ALTER  TABLE  element  DROP  CONSTRAINT  fk_element_parent_node_id
DROP  TABLE  node
DROP  TABLE  element 
```

在无法解决循环的情况下，例如如果我们在这里没有对约束应用名称，则将收到以下错误：

```py
sqlalchemy.exc.CircularDependencyError: Can't sort tables for DROP;
an unresolvable foreign key dependency exists between tables:
element, node.  Please ensure that the ForeignKey and ForeignKeyConstraint
objects involved in the cycle have names so that they can be dropped
using DROP CONSTRAINT.
```

此错误仅适用于 DROP 情况，因为我们可以在 CREATE 情况下发出 “ADD CONSTRAINT” 而无需名称；数据库通常会自动分配一个名称。

`ForeignKeyConstraint.use_alter` 和 `ForeignKey.use_alter` 关键字参数可用于手动解决依赖循环。我们可以仅将此标志添加到 `'element'` 表中，如下所示：

```py
element = Table(
    "element",
    metadata_obj,
    Column("element_id", Integer, primary_key=True),
    Column("parent_node_id", Integer),
    ForeignKeyConstraint(
        ["parent_node_id"],
        ["node.node_id"],
        use_alter=True,
        name="fk_element_parent_node_id",
    ),
)
```

在我们的 CREATE DDL 中，我们将只看到此约束的 ALTER 语句，而不是另一个约束的 ALTER 语句：

```py
>>> with engine.connect() as conn:
...     metadata_obj.create_all(conn, checkfirst=False)
CREATE  TABLE  element  (
  element_id  SERIAL  NOT  NULL,
  parent_node_id  INTEGER,
  PRIMARY  KEY  (element_id)
)

CREATE  TABLE  node  (
  node_id  SERIAL  NOT  NULL,
  primary_element  INTEGER,
  PRIMARY  KEY  (node_id),
  FOREIGN  KEY(primary_element)  REFERENCES  element  (element_id)
)

ALTER  TABLE  element  ADD  CONSTRAINT  fk_element_parent_node_id
FOREIGN  KEY(parent_node_id)  REFERENCES  node  (node_id) 
```

当与删除操作结合使用时，`ForeignKeyConstraint.use_alter` 和 `ForeignKey.use_alter` 将要求约束具有名称，否则将生成类似以下的错误：

```py
sqlalchemy.exc.CompileError: Can't emit DROP CONSTRAINT for constraint
ForeignKeyConstraint(...); it has no name
```

另请参阅

配置约束命名约定

`sort_tables_and_constraints()`  ### ON UPDATE 和 ON DELETE

大多数数据库都支持外键值的*级联*，即当父行更新时，新值会放在子行中，或者当父行删除时，所有相应的子行都会被设置为 null 或删除。在数据定义语言中，这些是使用诸如 “ON UPDATE CASCADE”，“ON DELETE CASCADE” 和 “ON DELETE SET NULL” 之类的短语来指定的，对应于外键约束。在 “ON UPDATE” 或 “ON DELETE” 之后的短语可能还允许使用特定于正在使用的数据库的其他短语。`ForeignKey` 和 `ForeignKeyConstraint` 对象通过 `onupdate` 和 `ondelete` 关键字参数支持生成此子句。该值是任何字符串，该字符串将在适当的 “ON UPDATE” 或 “ON DELETE” 短语之后输出：

```py
child = Table(
    "child",
    metadata_obj,
    Column(
        "id",
        Integer,
        ForeignKey("parent.id", onupdate="CASCADE", ondelete="CASCADE"),
        primary_key=True,
    ),
)

composite = Table(
    "composite",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    Column("rev_id", Integer),
    Column("note_id", Integer),
    ForeignKeyConstraint(
        ["rev_id", "note_id"],
        ["revisions.id", "revisions.note_id"],
        onupdate="CASCADE",
        ondelete="SET NULL",
    ),
)
```

请注意，这些子句在与 MySQL 配合使用时需要 `InnoDB` 表。它们在其他数据库上也可能不受支持。

另请参阅

有关将 `ON DELETE CASCADE` 与 ORM `relationship()` 结构集成的背景，请参阅以下部分：

使用外键 ON DELETE cascade 处理 ORM 关系

使用外键 ON DELETE 处理多对多关系  ## 唯一约束

可以使用 `Column` 上的 `unique` 关键字在单个列上匿名创建唯一约束。显式命名的唯一约束和/或具有多个列的约束通过 `UniqueConstraint` 表级构造创建。

```py
from sqlalchemy import UniqueConstraint

metadata_obj = MetaData()
mytable = Table(
    "mytable",
    metadata_obj,
    # per-column anonymous unique constraint
    Column("col1", Integer, unique=True),
    Column("col2", Integer),
    Column("col3", Integer),
    # explicit/composite unique constraint.  'name' is optional.
    UniqueConstraint("col2", "col3", name="uix_1"),
)
```

## 检查约束

检查约束可以具有命名或未命名，可以在列或表级别使用 `CheckConstraint` 构造。检查约束的文本直接传递到数据库，因此具有有限的“数据库独立”行为。列级检查约束通常只应引用它们放置的列，而表级约束可以引用表中的任何列。

请注意，一些数据库不支持主动支持检查约束，例如旧版本的 MySQL（在 8.0.16 之前）。

```py
from sqlalchemy import CheckConstraint

metadata_obj = MetaData()
mytable = Table(
    "mytable",
    metadata_obj,
    # per-column CHECK constraint
    Column("col1", Integer, CheckConstraint("col1>5")),
    Column("col2", Integer),
    Column("col3", Integer),
    # table level CHECK constraint.  'name' is optional.
    CheckConstraint("col2 > col3 + 5", name="check1"),
)

mytable.create(engine)
CREATE  TABLE  mytable  (
  col1  INTEGER  CHECK  (col1>5),
  col2  INTEGER,
  col3  INTEGER,
  CONSTRAINT  check1  CHECK  (col2  >  col3  +  5)
) 
```

## 主键约束

任何 `Table` 对象的主键约束在本质上是隐式存在的，基于标记有 `Column.primary_key` 标志的 `Column` 对象。`PrimaryKeyConstraint` 对象提供了对此约束的显式访问，其中包括直接配置的选项：

```py
from sqlalchemy import PrimaryKeyConstraint

my_table = Table(
    "mytable",
    metadata_obj,
    Column("id", Integer),
    Column("version_id", Integer),
    Column("data", String(50)),
    PrimaryKeyConstraint("id", "version_id", name="mytable_pk"),
)
```

另请参阅

`PrimaryKeyConstraint` - 详细的 API 文档。

## 使用 Declarative ORM 扩展设置约束

`Table` 是 SQLAlchemy 核心构造，允许定义表元数据，其中可以用于 SQLAlchemy ORM 作为映射类的目标。Declarative 扩展允许自动创建 `Table` 对象，主要根据表的内容作为 `Column` 对象的映射。

要将表级约束对象（例如 `ForeignKeyConstraint`）应用于使用 Declarative 定义的表，请使用 `__table_args__` 属性，详见 表配置。

## 配置约束命名约定

关系数据库通常会为所有约束和索引分配明确的名称。在常见情况下，使用`CREATE TABLE`创建表时，约束（如 CHECK、UNIQUE 和 PRIMARY KEY 约束）会与表定义一起内联生成，如果未另行指定名称，则数据库通常会自动为这些约束分配名称。当在使用诸如`ALTER TABLE`之类的命令更改数据库中的现有数据库表时，此命令通常需要为新约束指定显式名称，以及能够指定要删除或修改的现有约束的名称。

使用`Constraint.name`参数可以明确命名约束，对于索引，可以使用`Index.name`参数。然而，在约束的情况下，此参数是可选的。还有使用`Column.unique`和`Column.index`参数的用例，这些参数创建`UniqueConstraint`和`Index`对象时未明确指定名称。

对现有表和约束进行更改的用例可以由模式迁移工具（如[Alembic](https://alembic.sqlalchemy.org/)）处理。然而，Alembic 和 SQLAlchemy 目前都不会为未指定名称的约束对象创建名称，导致可以更改现有约束的情况下，必须反向工程关系数据库用于自动分配名称的命名系统，或者必须注意确保所有约束都已命名。

与必须为所有`Constraint`和`Index`对象分配显式名称相比，可以使用事件构建自动命名方案。这种方法的优势在于，约束将获得一致的命名方案，无需在整个代码中使用显式名称参数，并且约定也适用于由`Column.unique`和`Column.index`参数生成的那些约束和索引。从 SQLAlchemy 0.9.2 开始，包含了这种基于事件的方法，并且可以使用参数`MetaData.naming_convention`进行配置。

### 配置元数据集合的命名约定

`MetaData.naming_convention`指的是一个接受`Index`类或单独的`Constraint`类作为键，Python 字符串模板作为值的字典。它还接受一系列字符串代码作为备用键，分别为外键、主键、索引、检查和唯一约束的`"fk"`、`"pk"`、`"ix"`、`"ck"`、`"uq"`。在这个字典中的字符串模板在与这个`MetaData`对象关联的约束或索引没有给出现有名称的情况下���用。（包括一个例外情况，其中现有名称可以进一步装饰）。

适用于基本情况的一个命名约定示例如下：

```py
convention = {
    "ix": "ix_%(column_0_label)s",
    "uq": "uq_%(table_name)s_%(column_0_name)s",
    "ck": "ck_%(table_name)s_%(constraint_name)s",
    "fk": "fk_%(table_name)s_%(column_0_name)s_%(referred_table_name)s",
    "pk": "pk_%(table_name)s",
}

metadata_obj = MetaData(naming_convention=convention)
```

上述约定将为目标`MetaData`集合中的所有约束建立名称。例如，当我们创建一个未命名的`UniqueConstraint`时，我们可以观察到生成的名称：

```py
>>> user_table = Table(
...     "user",
...     metadata_obj,
...     Column("id", Integer, primary_key=True),
...     Column("name", String(30), nullable=False),
...     UniqueConstraint("name"),
... )
>>> list(user_table.constraints)[1].name
'uq_user_name'
```

即使我们只使用`Column.unique`标志，这个相同的特性也会生效：

```py
>>> user_table = Table(
...     "user",
...     metadata_obj,
...     Column("id", Integer, primary_key=True),
...     Column("name", String(30), nullable=False, unique=True),
... )
>>> list(user_table.constraints)[1].name
'uq_user_name'
```

命名约定方法的一个关键优势是，名称是在 Python 构建时建立的，而不是在 DDL 发出时建立的。当使用 Alembic 的`--autogenerate`功能时，这个特性的效果是，当生成新的迁移脚本时，命名约定将是明确的：

```py
def upgrade():
    op.create_unique_constraint("uq_user_name", "user", ["name"])
```

上述的`"uq_user_name"`字符串是从`--autogenerate`在我们的元数据中定位的`UniqueConstraint`对象复制而来。

可用的令牌包括 `%(table_name)s`、`%(referred_table_name)s`、`%(column_0_name)s`、`%(column_0_label)s`、`%(column_0_key)s`、`%(referred_column_0_name)s`，以及每个令牌的多列版本，包括 `%(column_0N_name)s`、`%(column_0_N_name)s`、`%(referred_column_0_N_name)s`，它们以带有或不带有下划线的形式呈现所有列名称。关于 `MetaData.naming_convention` 的文档进一步详细说明了这些约定。

### 默认命名约定

`MetaData.naming_convention` 的默认值处理了长期以来 SQLAlchemy 的行为，即使用 `Column.index` 参数创建 `Index` 对象时分配名称的情况：

```py
>>> from sqlalchemy.sql.schema import DEFAULT_NAMING_CONVENTION
>>> DEFAULT_NAMING_CONVENTION
immutabledict({'ix': 'ix_%(column_0_label)s'})
```

### 长名称的截断

当生成的名称，特别是使用多列令牌的名称，超出目标数据库的标识符长度限制（例如，PostgreSQL 具有 63 个字符的限制）时，名称将使用基于长名称的 md5 哈希的 4 个字符后缀进行确定性截断。例如，以下命名约定将根据使用的列名生成非常长的名称：

```py
metadata_obj = MetaData(
    naming_convention={"uq": "uq_%(table_name)s_%(column_0_N_name)s"}
)

long_names = Table(
    "long_names",
    metadata_obj,
    Column("information_channel_code", Integer, key="a"),
    Column("billing_convention_name", Integer, key="b"),
    Column("product_identifier", Integer, key="c"),
    UniqueConstraint("a", "b", "c"),
)
```

在 PostgreSQL 方言中，长度超过 63 个字符的名称将被截断，如以下示例所示：

```py
CREATE  TABLE  long_names  (
  information_channel_code  INTEGER,
  billing_convention_name  INTEGER,
  product_identifier  INTEGER,
  CONSTRAINT  uq_long_names_information_channel_code_billing_conventi_a79e
  UNIQUE  (information_channel_code,  billing_convention_name,  product_identifier)
)
```

上述后缀 `a79e` 基于长名称的 md5 哈希，并且每次生成相同的值，以便为给定模式生成一致的名称。

### 创建用于命名约定的自定义令牌

还可以通过在 naming_convention 字典中指定额外的令牌和可调用对象来添加新的令牌。例如，如果我们想要使用 GUID 方案对外键约束进行命名，我们可以这样做：

```py
import uuid

def fk_guid(constraint, table):
    str_tokens = (
        [
            table.name,
        ]
        + [element.parent.name for element in constraint.elements]
        + [element.target_fullname for element in constraint.elements]
    )
    guid = uuid.uuid5(uuid.NAMESPACE_OID, "_".join(str_tokens).encode("ascii"))
    return str(guid)

convention = {
    "fk_guid": fk_guid,
    "ix": "ix_%(column_0_label)s",
    "fk": "fk_%(fk_guid)s",
}
```

在创建新的 `ForeignKeyConstraint` 时，我们将获得以下名称：

```py
>>> metadata_obj = MetaData(naming_convention=convention)

>>> user_table = Table(
...     "user",
...     metadata_obj,
...     Column("id", Integer, primary_key=True),
...     Column("version", Integer, primary_key=True),
...     Column("data", String(30)),
... )
>>> address_table = Table(
...     "address",
...     metadata_obj,
...     Column("id", Integer, primary_key=True),
...     Column("user_id", Integer),
...     Column("user_version_id", Integer),
... )
>>> fk = ForeignKeyConstraint(["user_id", "user_version_id"], ["user.id", "user.version"])
>>> address_table.append_constraint(fk)
>>> fk.name
fk_0cd51ab5-8d70-56e8-a83c-86661737766d
```

另请参阅

`MetaData.naming_convention` - 附加使用详细信息以及所有可用命名组件的列表。

[命名约束的重要性](https://alembic.sqlalchemy.org/en/latest/naming.html) - 在 Alembic 文档中。

版本 1.3.0 中的新功能：添加了诸如 `%(column_0_N_name)s` 等多列命名令牌。超出目标数据库字符限制的生成名称将被确定性地截断。

### 命名 CHECK 约束

`CheckConstraint` 对象配置针对任意 SQL 表达式，该表达式可以有任意数量的列，并且通常使用原始 SQL 字符串进行配置。因此，通常与 `CheckConstraint` 一起使用的约定是，我们希望对象已经有一个名称，然后我们使用其他约定元素增强它。一个典型的约定是 `"ck_%(table_name)s_%(constraint_name)s"`：

```py
metadata_obj = MetaData(
    naming_convention={"ck": "ck_%(table_name)s_%(constraint_name)s"}
)

Table(
    "foo",
    metadata_obj,
    Column("value", Integer),
    CheckConstraint("value > 5", name="value_gt_5"),
)
```

上述表格将产生名称 `ck_foo_value_gt_5`：

```py
CREATE  TABLE  foo  (
  value  INTEGER,
  CONSTRAINT  ck_foo_value_gt_5  CHECK  (value  >  5)
)
```

`CheckConstraint` 还支持 `%(columns_0_name)s` token；我们可以通过确保在约束表达式中使用 `Column` 或 `column()` 元素来利用它，无论是通过单独声明约束表达式还是内联到表中：

```py
metadata_obj = MetaData(naming_convention={"ck": "ck_%(table_name)s_%(column_0_name)s"})

foo = Table("foo", metadata_obj, Column("value", Integer))

CheckConstraint(foo.c.value > 5)
```

或通过内联使用 `column()`：

```py
from sqlalchemy import column

metadata_obj = MetaData(naming_convention={"ck": "ck_%(table_name)s_%(column_0_name)s"})

foo = Table(
    "foo", metadata_obj, Column("value", Integer), CheckConstraint(column("value") > 5)
)
```

两者都将产生名称 `ck_foo_value`：

```py
CREATE  TABLE  foo  (
  value  INTEGER,
  CONSTRAINT  ck_foo_value  CHECK  (value  >  5)
)
```

“column zero”的名称确定是通过扫描给定表达式中的列对象来执行的。如果表达式中有多个列，扫描将使用确定性搜索，但是表达式的结构将决定哪个列被标记为“column zero”。### 针对布尔、枚举和其他模式类型进行命名配置

`SchemaType` 类指的是诸如 `Boolean` 和 `Enum` 等类型对象，它们生成与类型相伴随的 CHECK 约束。这里约束的名称最直接是通过发送“name”参数来设置的，例如 `Boolean.name`：

```py
Table("foo", metadata_obj, Column("flag", Boolean(name="ck_foo_flag")))
```

命名约定功能也可以与这些类型结合使用，通常是通过使用包含 `%(constraint_name)s` 的约定，然后将名称应用于类型：

```py
metadata_obj = MetaData(
    naming_convention={"ck": "ck_%(table_name)s_%(constraint_name)s"}
)

Table("foo", metadata_obj, Column("flag", Boolean(name="flag_bool")))
```

上述表格将产生约束名称 `ck_foo_flag_bool`：

```py
CREATE  TABLE  foo  (
  flag  BOOL,
  CONSTRAINT  ck_foo_flag_bool  CHECK  (flag  IN  (0,  1))
)
```

`SchemaType`类使用特殊的内部符号，以便命名约定仅在 DDL 编译时确定。在 PostgreSQL 上，有一个原生的 BOOLEAN 类型，因此不需要`Boolean`的 CHECK 约束；即使为检查约束设置了命名约定，我们也可以安全地设置不带名称的`Boolean`类型。如果我们针对没有原生 BOOLEAN 类型的数据库运行，如 SQLite 或 MySQL，那么仅当我们运行时才会参考此约定 CHECK 约束。

CHECK 约束也可以使用`column_0_name`标记，与`SchemaType`很好地配合使用，因为这些约束只有一个列：

```py
metadata_obj = MetaData(naming_convention={"ck": "ck_%(table_name)s_%(column_0_name)s"})

Table("foo", metadata_obj, Column("flag", Boolean()))
```

以上模式将产生：

```py
CREATE  TABLE  foo  (
  flag  BOOL,
  CONSTRAINT  ck_foo_flag  CHECK  (flag  IN  (0,  1))
)
```

### 使用 ORM 声明式混合与命名约定

使用命名约定功能与 ORM 声明性混合一起使用时，必须为每个实际表映射的子类存在单独的约束对象。有关背景和示例，请参见使用命名约定在混合上创建索引和约束部分。

## 约束 API

| 对象名称 | 描述 |
| --- | --- |
| CheckConstraint | 表级或列级 CHECK 约束。 |
| ColumnCollectionConstraint | 代理 ColumnCollection 的约束。 |
| ColumnCollectionMixin | `Column`对象的`ColumnCollection`。 |
| Constraint | 表级 SQL 约束。 |
| conv | 标记一个字符串，指示名称已经由命名约定转换。 |
| ForeignKey | 定义两列之间的依赖关系。 |
| ForeignKeyConstraint | 表级 FOREIGN KEY 约束。 |
| HasConditionalDDL | 定义一个包括`HasConditionalDDL.ddl_if()`方法的类，允许对 DDL 进行条件渲染。 |
| PrimaryKeyConstraint | 表级主键约束。 |
| UniqueConstraint | 表级 UNIQUE 约束。 |

```py
class sqlalchemy.schema.Constraint
```

表级 SQL 约束。

`Constraint` 作为一系列约束对象的基类，可以与 `Table` 对象关联，包括 `PrimaryKeyConstraint`、`ForeignKeyConstraint`、`UniqueConstraint` 和 `CheckConstraint`。

**成员**

__init__(), argument_for(), copy(), ddl_if(), dialect_kwargs, dialect_options, info, kwargs

**类签名**

类 `sqlalchemy.schema.Constraint` (`sqlalchemy.sql.base.DialectKWArgs`, `sqlalchemy.schema.HasConditionalDDL`, `sqlalchemy.schema.SchemaItem`)

```py
method __init__(name: _ConstraintNameArgument = None, deferrable: bool | None = None, initially: str | None = None, info: _InfoType | None = None, comment: str | None = None, _create_rule: Any | None = None, _type_bound: bool = False, **dialect_kw: Any) → None
```

创建一个 SQL 约束。

参数：

+   `name` – 可选，此 `Constraint` 的数据库名称。

+   `deferrable` – 可选布尔值。如果设置，则在为此约束发出 DDL 时发出 DEFERRABLE 或 NOT DEFERRABLE。

+   `initially` – 可选字符串。如果设置，则在为此约束发出 DDL 时发出 INITIALLY <value>。</value>

+   `info` – 可选数据字典，将填充到此对象的 `SchemaItem.info` 属性中。

+   `comment` –

    可选字符串，���在创建外键约束时呈现 SQL 注释。

    > 2.0 版本中的新功能。

+   `**dialect_kw` – 附加关键字参数是特定于方言的，并以 `<dialectname>_<argname>` 的形式传递。有关单个方言的文档参数详细信息，请参阅 方言。

+   `_create_rule` – 一些数据类型内部使用，也创建约束。

+   `_type_bound` – 内部使用，表示此约束与特定数据类型相关联。

```py
classmethod argument_for(dialect_name, argument_name, default)
```

*继承自* `DialectKWArgs.argument_for()` *方法的* `DialectKWArgs`

为此类添加一种新的特定于方言的关键字参数。

例如：

```py
Index.argument_for("mydialect", "length", None)

some_index = Index('a', 'b', mydialect_length=5)
```

`DialectKWArgs.argument_for()`方法是一种为`DefaultDialect.construct_arguments`字典添加额外参数的每个参数的方式。该字典提供了由各种模式级构造所接受的参数名称列表，代表一个方言。

新方言通常应该一次性指定此字典为方言类的数据成员。临时添加参数名称的用例通常是用于使用自定义编译方案并消耗额外参数的最终用户代码。

参数：

+   `dialect_name` – 方言名称。 方言必须是可定位的，否则会引发`NoSuchModuleError`。 方言还必须包括一个现有的`DefaultDialect.construct_arguments`集合，表示它参与关键字参数验证和默认系统，否则会引发`ArgumentError`。 如果方言不包含此集合，则可以已为该方言指定任何关键字参数。SQLAlchemy 中打包的所有方言都包含此集合，但对于第三方方言，支持可能会有所不同。

+   `argument_name` – 参数的名称。

+   `default` – 参数的默认值。

```py
method copy(**kw: Any) → Self
```

自 1.4 版本弃用：`Constraint.copy()` 方法已弃用，并将在将来的版本中删除。

```py
method ddl_if(dialect: str | None = None, callable_: DDLIfCallable | None = None, state: Any | None = None) → Self
```

*继承自* `HasConditionalDDL.ddl_if()` *的方法* `HasConditionalDDL`

对此模式项目应用条件 DDL 规则。

这些规则的工作方式与`ExecutableDDLElement.execute_if()`类似，不同之处在于可以在 DDL 编译阶段检查条件，例如`CreateTable`这样的结构。 `HasConditionalDDL.ddl_if()` 目前适用于`Index`结构以及所有`Constraint`结构。

参数：

+   `dialect` – 方言的字符串名称，或字符串名称的元组，表示多个方言类型。

+   `callable_` – 使用与`ExecutableDDLElement.execute_if.callable_`中描述的相同形式构建的可调用对象。

+   `state` – 将传递给可调用对象的任意任意对象（如果存在）。

2.0 版本中的新功能。

另请参阅

控制约束和索引的 DDL 生成 - 背景和用法示例

```py
attribute dialect_kwargs
```

*继承自* `DialectKWArgs.dialect_kwargs` *属性的* `DialectKWArgs`

作为针对此结构的方言特定选项指定的关键字参数的集合。

这里的参数以原始的`<dialect>_<kwarg>`格式呈现。仅包含实际传递的参数；与包含此方言的所有选项（包括默认值）的`DialectKWArgs.dialect_options` 集合不同。

该集合也是可写的；键采用`<dialect>_<kwarg>`形式，其中的值将被组装成选项列表。

另请参阅

`DialectKWArgs.dialect_options` - 嵌套字典形式

```py
attribute dialect_options
```

*继承自* `DialectKWArgs.dialect_options` *属性的* `DialectKWArgs`

作为针对此结构的方言特定选项指定的关键字参数的集合。

这是一个两级嵌套的注册表，键为 `<dialect_name>` 和 `<argument_name>`。例如，`postgresql_where` 参数可定位为：

```py
arg = my_object.dialect_options['postgresql']['where']
```

0.9.2 版本中的新功能。

另请参阅

`DialectKWArgs.dialect_kwargs` - 平面字典形式

```py
attribute info
```

*继承自* `SchemaItem.info` *属性的* `SchemaItem`

与对象相关联的信息字典，允许将用户定义的数据与此 `SchemaItem` 关联。

当首次访问时，字典会自动生成。它也可以在某些对象的构造函数中指定，例如`Table` 和 `Column`。

```py
attribute kwargs
```

*继承自* `DialectKWArgs.kwargs` *的* `DialectKWArgs` *属性*

`DialectKWArgs.dialect_kwargs` 的同义词。

```py
class sqlalchemy.schema.ColumnCollectionMixin
```

一个`Column`对象的`ColumnCollection`。

此集合表示此对象引用的列。

```py
class sqlalchemy.schema.ColumnCollectionConstraint
```

代理列集合的约束。

**成员**

__init__(), argument_for(), columns, contains_column(), copy(), ddl_if(), dialect_kwargs, dialect_options, info, kwargs

**类签名**

class `sqlalchemy.schema.ColumnCollectionConstraint` (`sqlalchemy.schema.ColumnCollectionMixin`, `sqlalchemy.schema.Constraint`)

```py
method __init__(*columns: _DDLColumnArgument, name: _ConstraintNameArgument = None, deferrable: bool | None = None, initially: str | None = None, info: _InfoType | None = None, _autoattach: bool = True, _column_flag: bool = False, _gather_expressions: List[_DDLColumnArgument] | None = None, **dialect_kw: Any) → None
```

参数:

+   `*columns` – 一系列列名或列对象。

+   `name` – 可选项，此约束的数据库中的名称。

+   `deferrable` – 可选布尔值。如果设置，发出 DDL 时会发出 DEFERRABLE 或 NOT DEFERRABLE 用于此约束。

+   `initially` – 可选字符串。如果设置，发出 DDL 时会发出 INITIALLY <value> 用于此约束。</value>

+   `**dialect_kw` – 其他关键字参数，包括方言特定参数，都会传播到`Constraint` 超类。

```py
classmethod argument_for(dialect_name, argument_name, default)
```

*继承自* `DialectKWArgs.argument_for()` *方法的* `DialectKWArgs` 

为此类添加一种新的特定于方言的关键字参数。

例如：

```py
Index.argument_for("mydialect", "length", None)

some_index = Index('a', 'b', mydialect_length=5)
```

`DialectKWArgs.argument_for()` 方法是向 `DefaultDialect.construct_arguments` 字典添加额外参数的每个参数的方法。 此字典提供了代表方言的各种模式级构造所接受的参数名称列表。

新方言通常应一次性指定此字典为方言类的数据成员。 通常，为了使用自定义编译方案并使用附加参数的最终用户代码，使用情形是添加参数名称的即兴增加。

参数：

+   `dialect_name` – 方言的名称。 方言必须可定位，否则会引发 `NoSuchModuleError`。 方言还必须包括一个现有的 `DefaultDialect.construct_arguments` 集合，指示其参与关键字参数验证和默认系统，否则将引发 `ArgumentError`。 如果方言不包括此集合，则可以代表此方言已经指定任何关键字参数。 SQLAlchemy 中打包的所有方言都包括此集合，但是对于第三方方言，支持可能有所不同。

+   `argument_name` – 参数的名称。

+   `default` – 参数的默认值。

```py
attribute columns: ReadOnlyColumnCollection[str, Column[Any]]
```

*继承自* `ColumnCollectionMixin` *属性* `ColumnCollectionMixin.columns` *的*。

代表此约束的一组列的 `ColumnCollection`。

```py
method contains_column(col: Column[Any]) → bool
```

如果此约束包含给定的列，则返回 True。

请注意，此对象还包含一个属性 `.columns`，它是 `Column` 对象的 `ColumnCollection`。

```py
method copy(*, target_table: Table | None = None, **kw: Any) → ColumnCollectionConstraint
```

从版本 1.4 开始弃用：`ColumnCollectionConstraint.copy()` 方法已弃用，并将在将来的版本中删除。

```py
method ddl_if(dialect: str | None = None, callable_: DDLIfCallable | None = None, state: Any | None = None) → Self
```

*继承自* `HasConditionalDDL.ddl_if()` *方法的* `HasConditionalDDL`

对此模式项应用条件 DDL 规则。

这些规则的工作方式与`ExecutableDDLElement.execute_if()`可调用对象类似，额外的特性是可以在 DDL 编译阶段检查条件，例如`CreateTable`构造中的条件。`HasConditionalDDL.ddl_if()`目前也适用于`Index`构造以及所有`Constraint`构造。

参数：

+   `dialect` – 方言的字符串名称，或表示多个方言类型的字符串名称元组。

+   `callable_` – 一个可调用对象，其构造方式与`ExecutableDDLElement.execute_if.callable_`中描述的形式相同。

+   `state` – 任意对象，如果存在将传递给可调用对象。

版本 2.0 中的新功能。

另请参阅

控制约束和索引的 DDL 生成 - 背景和使用示例

```py
attribute dialect_kwargs
```

*继承自* `DialectKWArgs.dialect_kwargs` *属性的* `DialectKWArgs`

作为特定于方言的选项指定为此构造的关键字参数集合。

这里的参数以原始的`<dialect>_<kwarg>`格式呈现。仅包括实际传递的参数；与包含所有已知选项的默认值的`DialectKWArgs.dialect_options`集合不同。

该集合也是可写的；键的形式为`<dialect>_<kwarg>`，其中的值将被组装到选项列表中。

另请参阅

`DialectKWArgs.dialect_options` - 嵌套字典形式

```py
attribute dialect_options
```

*继承自* `DialectKWArgs.dialect_options` *属性的* `DialectKWArgs`

作为方言特定选项指定的关键字参数集合。

这是一个两级嵌套的注册表，键入 `<dialect_name>` 和 `<argument_name>`。例如，`postgresql_where` 参数可以定位为：

```py
arg = my_object.dialect_options['postgresql']['where']
```

新版本 0.9.2 中新增。

另见

`DialectKWArgs.dialect_kwargs` - 扁平字典形式

```py
attribute info
```

*继承自* `SchemaItem.info` *属性的* `SchemaItem`

与对象关联的信息字典，允许将用户定义的数据与此 `SchemaItem` 关联。

第一次访问时自动生成该字典。也可以在某些对象的构造函数中指定，例如 `Table` 和 `Column`。

```py
attribute kwargs
```

*继承自* `DialectKWArgs.kwargs` *属性的* `DialectKWArgs`

`DialectKWArgs.dialect_kwargs` 的同义词。

```py
class sqlalchemy.schema.CheckConstraint
```

表或列级别的检查约束。

可以包含在表或列的定义中。

**成员**

__init__(), argument_for(), columns, contains_column(), copy(), ddl_if(), dialect_kwargs, dialect_options, info, kwargs

**类签名**

类 `sqlalchemy.schema.CheckConstraint` (`sqlalchemy.schema.ColumnCollectionConstraint`)

```py
method __init__(sqltext: _TextCoercedExpressionArgument[Any], name: _ConstraintNameArgument = None, deferrable: bool | None = None, initially: str | None = None, table: Table | None = None, info: _InfoType | None = None, _create_rule: Any | None = None, _autoattach: bool = True, _type_bound: bool = False, **dialect_kw: Any) → None
```

构建一个检查约束。

参数：

+   `sqltext` –

    包含约束定义的字符串，将会原样使用，或者是一个 SQL 表达式构造。如果给定为字符串，则将对象转换为 `text()` 对象。如果文本字符串包含冒号字符，则使用反斜杠进行转义：

    ```py
    CheckConstraint(r"foo ~ E'a(?\:b|c)d")
    ```

    警告

    `CheckConstraint.sqltext` 参数可以作为 Python 字符串参数传递给 `CheckConstraint`，该参数将被视为**受信任的 SQL 文本**并按原样呈现。**不要将不受信任的输入传递给此参数**。

+   `name` – 可选的，约束在数据库中的名称。

+   `deferrable` – 可选布尔值。如果设置了，则在为此约束发出 DDL 时发出 DEFERRABLE 或 NOT DEFERRABLE。

+   `initially` – 可选字符串。如果设置了，则在为此约束发出 DDL 时发出 INITIALLY <value>。

+   `info` – 可选的数据字典，将填充到该对象的 `SchemaItem.info` 属性中。

```py
classmethod argument_for(dialect_name, argument_name, default)
```

*继承自* `DialectKWArgs.argument_for()` *方法的* `DialectKWArgs`

为此类添加一种新的方言特定关键字参数。

例如：

```py
Index.argument_for("mydialect", "length", None)

some_index = Index('a', 'b', mydialect_length=5)
```

`DialectKWArgs.argument_for()` 方法是一种逐个方式向 `DefaultDialect.construct_arguments` 字典添加额外参数的方法。该字典提供了方言代表各种模式级别构造接受的参数名称列表。

新的方言通常应该一次性指定该字典，作为方言类的数据成员。临时添加参数名称的用例通常是用于最终用户代码，该代码还使用了自定义编译方案，该方案会使用额外的参数。

参数：

+   `dialect_name` – 方言的名称。如果无法找到该方言，则会引发 `NoSuchModuleError`。该方言还必须包括一个现有的 `DefaultDialect.construct_arguments` 集合，表示它参与关键字参数的验证和默认系统，否则将引发 `ArgumentError`。如果该方言不包括此集合，则可以代表该方言指定任何关键字参数。所有 SQLAlchemy 打包的方言都包括此集合，但是对于第三方方言，支持可能有所不同。

+   `argument_name` – 参数的名称。

+   `default` – 参数的默认值。

```py
attribute columns: ReadOnlyColumnCollection[str, Column[Any]]
```

*继承自* `ColumnCollectionMixin.columns` *属性的* `ColumnCollectionMixin`

表示此约束的列集合。

```py
method contains_column(col: Column[Any]) → bool
```

*继承自* `ColumnCollectionConstraint.contains_column()` *方法的* `ColumnCollectionConstraint`

如果此约束包含给定的列，则返回 True。

注意，此对象还包含一个属性 `.columns`，它是一个 `ColumnCollection`，其中包含 `Column` 对象。

```py
method copy(*, target_table: Table | None = None, **kw: Any) → CheckConstraint
```

自版本 1.4 起已弃用：`CheckConstraint.copy()` 方法已弃用，并将在将来的版本中删除。

```py
method ddl_if(dialect: str | None = None, callable_: DDLIfCallable | None = None, state: Any | None = None) → Self
```

*继承自* `HasConditionalDDL.ddl_if()` *方法的* `HasConditionalDDL`

对此模式项应用条件 DDL 规则。

这些规则的工作方式与`ExecutableDDLElement.execute_if()`可调用对象类似，但增加了一个功能，即可以在 DDL 编译阶段检查条件，例如`CreateTable`构造。 `HasConditionalDDL.ddl_if()`目前也适用于`Index`构造以及所有`Constraint`构造。

参数：

+   `dialect` - 方言的字符串名称，或字符串名称的元组以指示多个方言类型。

+   `callable_` - 使用与`ExecutableDDLElement.execute_if.callable_`描述的相同形式构造的可调用对象。

+   `state` - 任意对象，如果存在，将传递给可调用对象。

版本 2.0 中的新功能。

另请参阅

控制约束和索引的 DDL 生成 - 背景和用法示例

```py
attribute dialect_kwargs
```

*继承自* `DialectKWArgs.dialect_kwargs` *属性的* `DialectKWArgs`

作为方言特定选项传递给此构造函数的关键字参数集合。

这里的参数以其原始的`<dialect>_<kwarg>`格式存在。只包括实际传递的参数；与包含此方言的所有选项的`DialectKWArgs.dialect_options`集合不同，后者包含了所有已知选项，包括默认值。

该集合也可写入；键的形式为`<dialect>_<kwarg>`，其中的值将被组装到选项列表中。

另请参阅

`DialectKWArgs.dialect_options` - 嵌套字典形式

```py
attribute dialect_options
```

*继承自* `DialectKWArgs.dialect_options` *属性的* `DialectKWArgs`

作为方言特定选项传递给此构造函数的关键字参数集合。

这是一个两级嵌套的注册表，键为`<dialect_name>`和`<argument_name>`。例如，`postgresql_where`参数可定位为：

```py
arg = my_object.dialect_options['postgresql']['where']
```

版本 0.9.2 中的新功能。

另请参阅

`DialectKWArgs.dialect_kwargs` - 平面字典形式

```py
attribute info
```

*继承自* `SchemaItem.info` *属性的* `SchemaItem`。

与对象相关联的信息字典，允许将用户定义的数据与此`SchemaItem`相关联。

当首次访问时，字典会自动生成。也可以在某些对象的构造函数中指定，例如`Table`和`Column`。

```py
attribute kwargs
```

*继承自* `DialectKWArgs.kwargs` *属性的* `DialectKWArgs`。

`DialectKWArgs.dialect_kwargs`的同义词。

```py
class sqlalchemy.schema.ForeignKey
```

定义了两个列之间的依赖关系。

`ForeignKey`被指定为`Column`对象的一个参数，例如：

```py
t = Table("remote_table", metadata,
    Column("remote_id", ForeignKey("main_table.id"))
)
```

注意`ForeignKey`只是一个标记对象，它定义了两个列之间的依赖关系。在所有情况下，实际的约束由`ForeignKeyConstraint`对象表示。当`ForeignKey`与一个与`Table`关联的`Column`相关联时，此对象将自动生成。相反，当`ForeignKeyConstraint`应用于一个与`Table`相关联的`Table`时，将自动生成`ForeignKey`标记，这些标记也与约束对象相关联。

请注意，您不能使用`ForeignKey`对象定义“复合”外键约束，即多个父/子列的分组之间的约束。要定义此分组，必须使用`ForeignKeyConstraint`对象，并将其应用于`Table`。相关的`ForeignKey`对象会自动生成。

与单个`Column`对象关联的`ForeignKey`对象可在该列的 foreign_keys 集合中使用。

更多外键配置示例见定义外键。

**成员**

__init__(), argument_for(), column, copy(), dialect_kwargs, dialect_options, get_referent(), info, kwargs, references(), target_fullname

**类签名**

类`sqlalchemy.schema.ForeignKey`（`sqlalchemy.sql.base.DialectKWArgs`, `sqlalchemy.schema.SchemaItem`)

```py
method __init__(column: _DDLColumnArgument, _constraint: ForeignKeyConstraint | None = None, use_alter: bool = False, name: _ConstraintNameArgument = None, onupdate: str | None = None, ondelete: str | None = None, deferrable: bool | None = None, initially: str | None = None, link_to_name: bool = False, match: str | None = None, info: _InfoType | None = None, comment: str | None = None, _unresolvable: bool = False, **dialect_kw: Any)
```

构造列级 FOREIGN KEY。

构造`ForeignKey`对象时生成与父`Table`对象的约束集合关联的`ForeignKeyConstraint`。

参数：

+   `column` – 关键关系的单个目标列。一个`Column`对象或作为字符串的列名：`tablename.columnkey`或`schema.tablename.columnkey`。除非`link_to_name`为`True`，否则`columnkey`是分配给列的`key`（默认为列名本身）。

+   `name` – 可选的字符串。如果未提供约束，则用于键的数据库内名称。

+   `onupdate` – 可选的字符串。如果设置，发出 ON UPDATE <value>的 DDL。典型值包括 CASCADE、DELETE 和 RESTRICT。</value>

+   `ondelete` – 可选的字符串。如果设置，发出 ON DELETE <value>的 DDL。典型值包括 CASCADE、DELETE 和 RESTRICT。</value>

+   `deferrable` – 可选的布尔值。如果设置，发出 DEFERRABLE 或 NOT DEFERRABLE 的 DDL。

+   `initially` – 可选的字符串。如果设置，发出 INITIALLY <value>的 DDL。</value>

+   `link_to_name` – 如果为 True，则`column`中给定的字符串名称是引用列的呈现名称，而不是其本地分配的`key`。

+   `use_alter` –

    传递给底层 `ForeignKeyConstraint` 以指示应该从 CREATE TABLE/ DROP TABLE 语句外部生成/删除约束。 有关详细信息，请参阅 `ForeignKeyConstraint.use_alter`。

    另请参见

    `ForeignKeyConstraint.use_alter`

    通过 ALTER 创建/删除外键约束

+   `match` – 可选字符串。 如果设置，当为此约束发出 DDL 时，将发出 MATCH <value>。 典型值包括 SIMPLE、PARTIAL 和 FULL。</value>

+   `info` – 可选数据字典，将填充到此对象的 `SchemaItem.info` 属性中。

+   `comment` –

    可选字符串，将在外键约束创建时渲染 SQL 注释。

    > 2.0 版中的新内容。

+   `**dialect_kw` – 额外的关键字参数是特定于方言的，并以 `<dialectname>_<argname>` 的形式传递。 这些参数最终由相应的 `ForeignKeyConstraint` 处理。 有关已记录参数的详细信息，请参阅 方言 中有关单个方言的文档。

```py
classmethod argument_for(dialect_name, argument_name, default)
```

*继承自* `DialectKWArgs.argument_for()` *方法的* `DialectKWArgs`

为此类添加一种新类型的特定于方言的关键字参数。

例如：

```py
Index.argument_for("mydialect", "length", None)

some_index = Index('a', 'b', mydialect_length=5)
```

`DialectKWArgs.argument_for()` 方法是一种逐个参数方式向 `DefaultDialect.construct_arguments` 字典添加额外参数的方法。 此字典提供了由各种基于模式的构造物代表方言的参数名列表。

新方言通常应将此字典一次性指定为方言类的数据成员。 通常，临时添加参数名称的用例是为了同时使用自定义编译方案的最终用户代码，该方案使用附加参数。

参数：

+   `dialect_name` – 方言的名称。方言必须是可定位的，否则会引发 `NoSuchModuleError`。方言还必须包括一个现有的 `DefaultDialect.construct_arguments` 集合，指示其参与关键字参数验证和默认系统，否则会引发 `ArgumentError`。如果方言不包括此集合，则已经可以代表此方言指定任何关键字参数。SQLAlchemy 中打包的所有方言都包括此集合，但对于第三方方言，支持可能有所不同。

+   `argument_name` – 参数的名称。

+   `default` – 参数的默认值。

```py
attribute column
```

返回由此 `ForeignKey` 引用的目标 `Column`。

如果没有建立目标列，则会引发异常��

```py
method copy(*, schema: str | None = None, **kw: Any) → ForeignKey
```

自版本 1.4 弃用：`ForeignKey.copy()` 方法已弃用，并将在将来的版本中移除。

```py
attribute dialect_kwargs
```

*继承自* `DialectKWArgs.dialect_kwargs` *属性* 的 *`DialectKWArgs`*

作为特定于方言的选项指定的关键字参数集合。

这里的参数以其原始的 `<dialect>_<kwarg>` 格式呈现。只包括实际传递的参数；不像 `DialectKWArgs.dialect_options` 集合，该集合包含此方言已知的所有选项，包括默认值。

该集合也是可写的；键的形式为 `<dialect>_<kwarg>`，其中的值将被组装到选项列表中。

另请参阅

`DialectKWArgs.dialect_options` - 嵌套字典形式

```py
attribute dialect_options
```

*继承自* `DialectKWArgs.dialect_options` *属性* 的 *`DialectKWArgs`*

作为特定于方言的选项指定的关键字参数集合。

这是一个两级嵌套的注册表，键入为`<dialect_name>`和`<argument_name>`。例如，`postgresql_where`参数可以定位为：

```py
arg = my_object.dialect_options['postgresql']['where']
```

版本 0.9.2 中的新功能。

另请参阅

`DialectKWArgs.dialect_kwargs` - 扁平字典形式

```py
method get_referent(table: FromClause) → Column[Any] | None
```

返回此`ForeignKey`引用的给定`Table`（或任何`FromClause`）中的`Column`。

如果此`ForeignKey`未引用给定的`Table`，则返回 None。

```py
attribute info
```

*继承自* `SchemaItem.info` *属性的* `SchemaItem`

与对象关联的信息字典，允许将用户定义的数据与此`SchemaItem`关联。

字典在首次访问时自动生成。也可以在一些对象的构造函数中指定，例如`Table`和`Column`。

```py
attribute kwargs
```

*继承自* `DialectKWArgs.kwargs` *属性的* `DialectKWArgs`

`DialectKWArgs.dialect_kwargs`的同义词。

```py
method references(table: Table) → bool
```

如果此`ForeignKey`引用给定的`Table`，则返回 True。

```py
attribute target_fullname
```

返回此`ForeignKey`的基于字符串的‘列规范’。

这通常相当于首先传递给对象构造函数的基于字符串的“tablename.colname”参数。

```py
class sqlalchemy.schema.ForeignKeyConstraint
```

表级别的外键约束。

定义单列或复合 FOREIGN KEY … REFERENCES 约束。对于简单的、单列的外键，将 `ForeignKey` 添加到 `Column` 的定义中相当于一个未命名的、单列的 `ForeignKeyConstraint`。

外键配置示例位于 定义外键 中。

**成员**

__init__(), argument_for(), column_keys, columns, contains_column(), copy(), ddl_if(), dialect_kwargs, dialect_options, elements, info, kwargs, referred_table

**类签名**

类 `sqlalchemy.schema.ForeignKeyConstraint`（`sqlalchemy.schema.ColumnCollectionConstraint`）

```py
method __init__(columns: _typing_Sequence[_DDLColumnArgument], refcolumns: _typing_Sequence[_DDLColumnArgument], name: _ConstraintNameArgument = None, onupdate: str | None = None, ondelete: str | None = None, deferrable: bool | None = None, initially: str | None = None, use_alter: bool = False, link_to_name: bool = False, match: str | None = None, table: Table | None = None, info: _InfoType | None = None, comment: str | None = None, **dialect_kw: Any) → None
```

构造一个能够处理复合键的 FOREIGN KEY。

参数：

+   `columns` – 一系列本地列名称。所命名的列必须在父表中定义并存在。除非 `link_to_name` 为 True，否则名称应与每列给定的 `key` 匹配（默认为名称）。

+   `refcolumns` – 一系列外键列名称或 Column 对象。这些列必须全部位于同一张表内。

+   `name` – 可选，键的数据库内名称。

+   `onupdate` – 可选字符串。如果设置，则在为此约束发出 DDL 时发出 ON UPDATE <value>。典型值包括 CASCADE、DELETE 和 RESTRICT。</value>

+   `ondelete` – 可选字符串。如果设置，则在为此约束发出 DDL 时发出 ON DELETE <value>。典型值包括 CASCADE、DELETE 和 RESTRICT。</value>

+   `deferrable` – 可选布尔值。如果设置，则在发出此约束的 DDL 时发出 DEFERRABLE 或 NOT DEFERRABLE。

+   `initially` – 可选字符串。如果设置，则在为此约束发出 DDL 时发出 INITIALLY <value>。</value>

+   `link_to_name` – 如果为 True，则 `column` 中给定的字符串名称是引用列的渲染名称，而不是其本地分配的 `key`。

+   `use_alter` –

    如果为 True，则不会将此约束的 DDL 作为 CREATE TABLE 定义的一部分输出。相反，在创建完整的表集合之后，通过 ALTER TABLE 语句生成它，在删除完整的表集合之前，通过 ALTER TABLE 语句将其删除。

    `ForeignKeyConstraint.use_alter` 的使用特别适用于两个或多个表在相互依赖的外键约束关系内建立的情况；然而，`MetaData.create_all()` 和 `MetaData.drop_all()` 方法将自动执行此解析，因此通常不需要该标志。

    另请参阅

    通过 ALTER 创建/删除外键约束

+   `match` – 可选字符串。如果设置，则在为此约束发出 DDL 时发出 MATCH <value>。典型值包括 SIMPLE、PARTIAL 和 FULL。</value>

+   `info` – 将填充到此对象的 `SchemaItem.info` 属性中的可选数据字典。

+   `comment` –

    可选字符串，将在创建外键约束时呈现 SQL 注释。

    > 新版本 2.0 中新增。

+   `**dialect_kw` – 附加关键字参数是方言特定的，并以 `<方言名称>_<参数名称>` 形式传递。有关文档中记录的参数的详细信息，请参阅方言中有关单个方言的文档。

```py
classmethod argument_for(dialect_name, argument_name, default)
```

*来自* `DialectKWArgs` *的* `DialectKWArgs.argument_for()` *方法继承*

为此类添加一种新的方言特定关键字参数。

例如：

```py
Index.argument_for("mydialect", "length", None)

some_index = Index('a', 'b', mydialect_length=5)
```

`DialectKWArgs.argument_for()` 方法是向`DefaultDialect.construct_arguments` 字典中各种模式级构造所接受的参数名称添加额外参数的方法。

新方言通常应一次性指定此字典作为方言类的数据成员。临时添加参数名称的用例通常是终端用户代码，该代码还使用自定义编译方案，该方案消耗附加参数。

参数：

+   `dialect_name` – 方言的名称。如果无法定位方言，将引发 `NoSuchModuleError`。方言还必须包括一个现有的 `DefaultDialect.construct_arguments` 集合，表示它参与关键字参数的验证和默认系统，否则将引发 `ArgumentError`。如果方言不包括此集合，则可以代表此方言已经指定任何关键字参数。SQLAlchemy 中打包的所有方言都包括此集合，但是对于第三方方言，支持可能有所不同。

+   `argument_name` – 参数的名称。

+   `default` – 参数的默认值。

```py
attribute column_keys
```

返回表示此 `ForeignKeyConstraint` 中本地列的字符串键列表。

此列表是发送到 `ForeignKeyConstraint` 构造函数的原始字符串参数，或者如果约束已使用 `Column` 对象进行初始化，则是每个元素的字符串 `.key`。

```py
attribute columns: ReadOnlyColumnCollection[str, Column[Any]]
```

*从* `ColumnCollectionMixin` *的* `ColumnCollectionMixin.columns` *属性继承*

表示此约束的列集合的 `ColumnCollection`。

```py
method contains_column(col: Column[Any]) → bool
```

*从* `ColumnCollectionConstraint.contains_column()` *方法继承*

如果此约束包含给定的列，则返回 True。

请注意，此对象还包含一个属性 `.columns`，它是 `Column` 对象的 `ColumnCollection`。

```py
method copy(*, schema: str | None = None, target_table: Table | None = None, **kw: Any) → ForeignKeyConstraint
```

从版本 1.4 起弃用：`ForeignKeyConstraint.copy()` 方法已弃用，并将在将来的版本中删除。

```py
method ddl_if(dialect: str | None = None, callable_: DDLIfCallable | None = None, state: Any | None = None) → Self
```

*从* `HasConditionalDDL` *的* `HasConditionalDDL.ddl_if()` *方法继承*

对此模式项应用条件 DDL 规则。

这些规则的工作方式与`ExecutableDDLElement.execute_if()`可调用对象类似，但增加了在 DDL 编译阶段检查条件的功能，例如`CreateTable`构造。`HasConditionalDDL.ddl_if()`当前还适用于`Index`构造以及所有`Constraint`构造。

参数：

+   `dialect` – 方言的字符串名称，或一个字符串名称元组，表示多个方言类型。

+   `callable_` – 使用与`ExecutableDDLElement.execute_if.callable_`描述的相同形式构造的可调用对象。

+   `state` – 如果存在，则会传递给可调用对象的任意对象。

版本 2.0 中的新功能。

另请参阅

控制约束和索引的 DDL 生成 - 背景和使用示例

```py
attribute dialect_kwargs
```

*继承自* `DialectKWArgs.dialect_kwargs` *属性的* `DialectKWArgs`

一组关键字参数，指定为此构造的方言特定选项。

这些参数在这里以其原始的`<dialect>_<kwarg>`格式呈现。只包括实际传递的参数；与包含此方言的所有选项（包括默认值）的`DialectKWArgs.dialect_options`集合不同。

该集合也是可写的；接受形式为`<dialect>_<kwarg>`的键，值将被组装到选项列表中。

另请参阅

`DialectKWArgs.dialect_options` - 嵌套字典形式

```py
attribute dialect_options
```

*继承自* `DialectKWArgs.dialect_options` *属性的* `DialectKWArgs`

一组关键字参数，指定为此构造的方言特定选项。

这是一个两级嵌套的注册表，键为`<dialect_name>`和`<argument_name>`。例如，`postgresql_where`参数可以定位为：

```py
arg = my_object.dialect_options['postgresql']['where']
```

版本 0.9.2 中的新功能。

另请参阅

`DialectKWArgs.dialect_kwargs` - 平面字典形式

```py
attribute elements: List[ForeignKey]
```

一系列`ForeignKey`对象。

每个`ForeignKey`表示一个引用列/被引用列对。

此集合预期为只读。

```py
attribute info
```

继承自`SchemaItem.info` *属性的*

与对象关联的信息字典，允许将用户定义的数据与此`SchemaItem`关联。

字典在首次访问时自动生成。它也可以在某些对象的构造函数中指定，例如`Table`和`Column`。

```py
attribute kwargs
```

继承自`DialectKWArgs.kwargs` *属性的*

`DialectKWArgs.dialect_kwargs`的同义词。

```py
attribute referred_table
```

此`ForeignKeyConstraint`引用的`Table`对象。

这是一个动态计算的属性，如果约束和/或父表尚未与包含所引用表的元数据集合关联，则可能无法使用此属性。

```py
class sqlalchemy.schema.HasConditionalDDL
```

定义一个包含`HasConditionalDDL.ddl_if()`方法的类，允许对 DDL 进行条件渲染。

目前适用于约束和索引。

**成员**

ddl_if()

版本 2.0 中的新功能。

```py
method ddl_if(dialect: str | None = None, callable_: DDLIfCallable | None = None, state: Any | None = None) → Self
```

对此模式项应用条件 DDL 规则。

这些规则的工作方式类似于 `ExecutableDDLElement.execute_if()` 可调用对象，但增加了一个特性，即在 DDL 编译阶段可以检查条件，例如 `CreateTable` 等构造中的条件。 `HasConditionalDDL.ddl_if()` 目前也适用于 `Index` 构造以及所有 `Constraint` 构造。

参数：

+   `dialect` – 方言的字符串名称，或者字符串名称的元组，表示多种方言类型。

+   `callable_` – 一个可调用对象，其构造方式与 `ExecutableDDLElement.execute_if.callable_` 中描述的相同。

+   `state` – 如果存在，则将传递给可调用对象的任意对象。

版本 2.0 中的新增内容。

另请参阅

控制约束和索引的 DDL 生成 - 背景和用法示例

```py
class sqlalchemy.schema.PrimaryKeyConstraint
```

表级别的主键约束。

`Table` 对象上自动存在 `PrimaryKeyConstraint` 对象；它被分配了一组与标记有 `Column.primary_key` 标志的列对应的 `Column` 对象：

```py
>>> my_table = Table('mytable', metadata,
...                 Column('id', Integer, primary_key=True),
...                 Column('version_id', Integer, primary_key=True),
...                 Column('data', String(50))
...     )
>>> my_table.primary_key
PrimaryKeyConstraint(
 Column('id', Integer(), table=<mytable>,
 primary_key=True, nullable=False),
 Column('version_id', Integer(), table=<mytable>,
 primary_key=True, nullable=False)
)
```

也可以通过显式使用 `PrimaryKeyConstraint` 对象来指定 `Table` 的主键；在这种用法模式下，还可以指定约束的“名称”，以及方言可能识别的其他选项：

```py
my_table = Table('mytable', metadata,
            Column('id', Integer),
            Column('version_id', Integer),
            Column('data', String(50)),
            PrimaryKeyConstraint('id', 'version_id',
                                 name='mytable_pk')
        )
```

通常不应混合两种列规范样式。如果同时存在 `PrimaryKeyConstraint` 中的列与标记为 `primary_key=True` 的列不匹配，则会发出警告；在这种情况下，列严格来自 `PrimaryKeyConstraint` 声明，并且其他标记为 `primary_key=True` 的列将被忽略。此行为旨在向后兼容以前的行为。

对于需要在 `PrimaryKeyConstraint` 上指定特定选项的用例，但仍希望使用 `primary_key=True` 标志的常规方式，可以指定一个空的 `PrimaryKeyConstraint`，该约束将根据标志从 `Table` 中获取主键列集合：

```py
my_table = Table('mytable', metadata,
            Column('id', Integer, primary_key=True),
            Column('version_id', Integer, primary_key=True),
            Column('data', String(50)),
            PrimaryKeyConstraint(name='mytable_pk',
                                 mssql_clustered=True)
        )
```

**成员**

argument_for(), columns, contains_column(), copy(), ddl_if(), dialect_kwargs, dialect_options, info, kwargs

**类签名**

class `sqlalchemy.schema.PrimaryKeyConstraint` (`sqlalchemy.schema.ColumnCollectionConstraint`)

```py
classmethod argument_for(dialect_name, argument_name, default)
```

*继承自* `DialectKWArgs.argument_for()` *方法的* `DialectKWArgs`

为此类添加一种新的方言特定关键字参数。

例如：

```py
Index.argument_for("mydialect", "length", None)

some_index = Index('a', 'b', mydialect_length=5)
```

`DialectKWArgs.argument_for()` 方法是一种逐个参数地向 `DefaultDialect.construct_arguments` 字典添加额外参数的方法。此字典提供了各种模式级别构造接受的参数名称列表，代表方言。

新的方言通常应将此字典一次性指定为方言类的数据成员。通常，对参数名称进行临时添加的用例是针对终端用户代码的，该代码还使用了消耗额外参数的自定义编译方案。

参数：

+   `dialect_name` – 方言的名称。方言必须是可定位的，否则将引发 `NoSuchModuleError`。方言还必须包含一个现有的 `DefaultDialect.construct_arguments` 集合，表明它参与关键字参数验证和默认系统，否则将引发 `ArgumentError`。如果方言不包括此集合，则可以代表此方言已经指定任何关键字参数。SQLAlchemy 中打包的所有方言都包括此集合，但是对于第三方方言，支持可能有所不同。

+   `argument_name` – 参数的名称。

+   `default` – 参数的默认值。

```py
attribute columns: ReadOnlyColumnCollection[str, Column[Any]]
```

*继承自* `ColumnCollectionMixin.columns` *属性的* `ColumnCollectionMixin`

代表此约束的一组列的 `ColumnCollection`。

```py
method contains_column(col: Column[Any]) → bool
```

*继承自* `ColumnCollectionConstraint.contains_column()` *方法的* `ColumnCollectionConstraint`

如果此约束包含给定的列，则返回 True。

请注意，此对象还包含一个名为 `.columns` 的属性，它是 `Column` 对象的 `ColumnCollection`。

```py
method copy(*, target_table: Table | None = None, **kw: Any) → ColumnCollectionConstraint
```

*继承自* `ColumnCollectionConstraint.copy()` *方法的* `ColumnCollectionConstraint`

从版本 1.4 开始已弃用：`ColumnCollectionConstraint.copy()` 方法已弃用，并将在将来的版本中删除。

```py
method ddl_if(dialect: str | None = None, callable_: DDLIfCallable | None = None, state: Any | None = None) → Self
```

*继承自* `HasConditionalDDL.ddl_if()` *方法的* `HasConditionalDDL`

对这个模式项应用一个条件 DDL 规则。

这些规则与`ExecutableDDLElement.execute_if()`可调用对象的工作方式类似，额外的特性是可以在 DDL 编译阶段检查条件，例如`CreateTable`这样的结构。 `HasConditionalDDL.ddl_if()`目前也适用于`Index`结构以及所有`Constraint`结构。

参数：

+   `dialect` – 方言的字符串名称，或者字符串名称的元组以指示多个方言类型。

+   `callable_` – 使用与`ExecutableDDLElement.execute_if.callable_`中描述的相同形式构造的可调用对象。

+   `state` – 如果存在，将传递给可调用对象的任意对象。

版本 2.0 中的新功能。

另请参阅

控制约束和索引的 DDL 生成 - 背景和使用示例

```py
attribute dialect_kwargs
```

*继承自* `DialectKWArgs.dialect_kwargs` *属性的* `DialectKWArgs`

一组作为特定于方言的选项指定为此结构的关键字参数。

这些参数以原始的 `<dialect>_<kwarg>` 格式出现在此处。仅包括实际传递的参数；不像`DialectKWArgs.dialect_options`集合，该集合包含该方言知道的所有选项，包括默认值。

该集合也是可写的；键以 `<dialect>_<kwarg>` 形式接受，其中值将被组装到选项列表中。

另请参阅

`DialectKWArgs.dialect_options` - 嵌套字典形式

```py
attribute dialect_options
```

*继承自* `DialectKWArgs.dialect_options` *属性的* `DialectKWArgs`

一组作为特定于方言的选项指定为此结构的关键字参数。

这是一个两级嵌套的注册表，键为 `<dialect_name>` 和 `<argument_name>`。例如，`postgresql_where` 参数可定位为：

```py
arg = my_object.dialect_options['postgresql']['where']
```

版本 0.9.2 中的新功能。

另请参阅

`DialectKWArgs.dialect_kwargs` - 平展的字典形式

```py
attribute info
```

*继承自* `SchemaItem.info` *属性的* `SchemaItem` *的*

与对象关联的信息字典，允许将用户定义的数据与此 `SchemaItem` 关联起来。

字典在首次访问时会自动生成。也可以在某些对象的构造函数中指定，例如 `Table` 和 `Column`。

```py
attribute kwargs
```

*继承自* `DialectKWArgs.kwargs` *属性的* `DialectKWArgs`

`DialectKWArgs.dialect_kwargs` 的同义词。

```py
class sqlalchemy.schema.UniqueConstraint
```

表级别的唯一约束。

定义单列或组合唯一约束。对于简单的单列约束，向 `Column` 定义中添加 `unique=True` 是一个等效的缩写，相当于未命名的单列 UniqueConstraint。

**成员**

__init__(), argument_for(), columns, contains_column(), copy(), ddl_if(), dialect_kwargs, dialect_options, info, kwargs

**类签名**

类 `sqlalchemy.schema.UniqueConstraint` (`sqlalchemy.schema.ColumnCollectionConstraint`)。

```py
method __init__(*columns: _DDLColumnArgument, name: _ConstraintNameArgument = None, deferrable: bool | None = None, initially: str | None = None, info: _InfoType | None = None, _autoattach: bool = True, _column_flag: bool = False, _gather_expressions: List[_DDLColumnArgument] | None = None, **dialect_kw: Any) → None
```

*继承自* `sqlalchemy.schema.ColumnCollectionConstraint.__init__` *方法的* `ColumnCollectionConstraint`

参数：

+   `*columns` – 一系列列名或 Column 对象。

+   `name` – 可选项，此约束的数据库内名称。

+   `deferrable` – 可选的布尔值。如果设置，当为此约束发出 DDL 时会发出 DEFERRABLE 或 NOT DEFERRABLE。

+   `initially` – 可选字符串。如果设置，当为此约束发出 DDL 时，会发出 INITIALLY <value>。

+   `**dialect_kw` – 其他关键字参数，包括方言特定参数，将传播到 `Constraint` 超类。

```py
classmethod argument_for(dialect_name, argument_name, default)
```

*继承自* `DialectKWArgs` *的* `DialectKWArgs.argument_for()` *方法*。

为此类添加一种新的方言特定关键字参数。

例如：

```py
Index.argument_for("mydialect", "length", None)

some_index = Index('a', 'b', mydialect_length=5)
```

`DialectKWArgs.argument_for()` 方法是逐个参数地向 `DefaultDialect.construct_arguments` 字典添加额外参数的方式。此字典提供了各种模式级别构造所接受的参数名称列表，代表方言。

新方言通常应该将此字典一次性指定为方言类的数据成员。通常情况下，对参数名称的临时添加用例是用于还使用自定义编译方案的端用户代码，该方案会消耗额外的参数。

参数：

+   `dialect_name` – 方言的名称。方言必须可定位，否则会引发 `NoSuchModuleError`。方言还必须包括现有的 `DefaultDialect.construct_arguments` 集合，指示其参与关键字参数验证和默认系统，否则会引发 `ArgumentError`。如果方言不包括此集合，则已经可以代表此方言指定任何关键字参数。SQLAlchemy 中打包的所有方言都包含此集合，但是对于第三方方言，支持可能有所不同。

+   `argument_name` – 参数的名称。

+   `default` – 参数的默认值。

```py
attribute columns: ReadOnlyColumnCollection[str, Column[Any]]
```

*继承自* `ColumnCollectionMixin` *的* `ColumnCollectionMixin.columns` *属性*。

表示此约束的列集合的 `ColumnCollection`。

```py
method contains_column(col: Column[Any]) → bool
```

*继承自* `ColumnCollectionConstraint.contains_column()` *方法的* `ColumnCollectionConstraint`

如果此约束包含给定的列，则返回 True。

请注意，此对象还包含一个属性`.columns`，它是`Column`对象的`ColumnCollection`。

```py
method copy(*, target_table: Table | None = None, **kw: Any) → ColumnCollectionConstraint
```

*继承自* `ColumnCollectionConstraint.copy()` *方法的* `ColumnCollectionConstraint`

自版本 1.4 开始已弃用：`ColumnCollectionConstraint.copy()` 方法已弃用，将在未来版本中删除。

```py
method ddl_if(dialect: str | None = None, callable_: DDLIfCallable | None = None, state: Any | None = None) → Self
```

*继承自* `HasConditionalDDL.ddl_if()` *方法的* `HasConditionalDDL`

对此模式项应用一个条件 DDL 规则。

这些规则的工作方式与`ExecutableDDLElement.execute_if()`可调用对象类似，但添加了一个特性，即可以在 DDL 编译阶段检查条件，例如`CreateTable`构造。 `HasConditionalDDL.ddl_if()` 目前也适用于`Index`构造以及所有`Constraint`构造。

参数：

+   `dialect` – 方言的字符串名称，或者一个字符串名称的元组，表示多个方言类型。

+   `callable_` – 使用与`ExecutableDDLElement.execute_if.callable_`描述的相同形式构建的可调用对象。

+   `state` – 如果存在，将传递给可调用对象的任意对象。

2.0 版中的新功能。

另请参阅

控制约束和索引的 DDL 生成 - 背景和使用示例

```py
attribute dialect_kwargs
```

*继承自* `DialectKWArgs.dialect_kwargs` *属性的* `DialectKWArgs`

一个关键字参数的集合，指定为此构造函数的特定于方言的选项。

这里的参数以其原始的 `<dialect>_<kwarg>` 格式呈现。仅包括实际传递的参数；与 `DialectKWArgs.dialect_options` 集合不同，后者包含此方言的所有已知选项，包括默认值。

该集合也是可写的；接受形式为 `<dialect>_<kwarg>` 的键，其值将被组装到选项列表中。

另请参阅

`DialectKWArgs.dialect_options` - 嵌套字典形式

```py
attribute dialect_options
```

*继承自* `DialectKWArgs.dialect_options` *属性的* `DialectKWArgs`

一个关键字参数的集合，指定为此构造函数的特定于方言的选项。

这是一个两级嵌套的注册表，键入为 `<dialect_name>` 和 `<argument_name>`。例如，`postgresql_where` 参数可以定位为：

```py
arg = my_object.dialect_options['postgresql']['where']
```

新版本中新增 0.9.2。

另请参阅

`DialectKWArgs.dialect_kwargs` - 平面字典形式

```py
attribute info
```

*继承自* `SchemaItem.info` *属性的* `SchemaItem`

与此 `SchemaItem` 关联的信息字典，允许将用户定义的数据与此对象关联。

当首次访问时，该字典将自动生成。也可以在某些对象的构造函数中指定它，例如 `Table` 和 `Column`。

```py
attribute kwargs
```

*继承自* `DialectKWArgs.kwargs` *属性的* `DialectKWArgs`

`DialectKWArgs.dialect_kwargs` 的同义词。

```py
function sqlalchemy.schema.conv(value: str, quote: bool | None = None) → Any
```

标记字符串，指示名称已经被命名约定转换。

这是一个字符串子类，指示不应再受任何进一步命名约定的影响的名称。

例如，当我们按照以下命名约定创建一个 `Constraint` 时：

```py
m = MetaData(naming_convention={
    "ck": "ck_%(table_name)s_%(constraint_name)s"
})
t = Table('t', m, Column('x', Integer),
                CheckConstraint('x > 5', name='x5'))
```

上述约束的名称将呈现为 `"ck_t_x5"`。也就是说，现有名称 `x5` 被用作命名约定中的 `constraint_name` 令牌。

在某些情况下，例如在迁移脚本中，我们可能会使用已经转换过的名称渲染上述 `CheckConstraint`。为了确保名称不被双重修改，新名称使用 `conv()` 标记应用。我们可以显式地使用如下：

```py
m = MetaData(naming_convention={
    "ck": "ck_%(table_name)s_%(constraint_name)s"
})
t = Table('t', m, Column('x', Integer),
                CheckConstraint('x > 5', name=conv('ck_t_x5')))
```

在上述情况中，`conv()` 标记指示此处约束名为最终名称，名称将呈现为 `"ck_t_x5"` 而不是 `"ck_t_ck_t_x5"`

另请参见

配置约束命名约定

## 索引

可以匿名创建索引（使用自动生成的名称 `ix_<column label>`）来为单个列使用内联 `index` 关键字，该关键字还修改了 `unique` 的使用，将唯一性应用于索引本身，而不是添加单独的 UNIQUE 约束。对于具有特定名称或涵盖多个列的索引，请使用 `Index` 构造，该构造需要一个名称。

下面我们示例了一个带有多个相关 `Index` 对象的 `Table`。DDL “创建索引”的语句会在表的创建语句之后立即发出：

```py
metadata_obj = MetaData()
mytable = Table(
    "mytable",
    metadata_obj,
    # an indexed column, with index "ix_mytable_col1"
    Column("col1", Integer, index=True),
    # a uniquely indexed column with index "ix_mytable_col2"
    Column("col2", Integer, index=True, unique=True),
    Column("col3", Integer),
    Column("col4", Integer),
    Column("col5", Integer),
    Column("col6", Integer),
)

# place an index on col3, col4
Index("idx_col34", mytable.c.col3, mytable.c.col4)

# place a unique index on col5, col6
Index("myindex", mytable.c.col5, mytable.c.col6, unique=True)

mytable.create(engine)
CREATE  TABLE  mytable  (
  col1  INTEGER,
  col2  INTEGER,
  col3  INTEGER,
  col4  INTEGER,
  col5  INTEGER,
  col6  INTEGER
)
CREATE  INDEX  ix_mytable_col1  ON  mytable  (col1)
CREATE  UNIQUE  INDEX  ix_mytable_col2  ON  mytable  (col2)
CREATE  UNIQUE  INDEX  myindex  ON  mytable  (col5,  col6)
CREATE  INDEX  idx_col34  ON  mytable  (col3,  col4) 
```

请注意，在上面的示例中，`Index` 构造是在对应的表之外创建的，直接使用 `Column` 对象。`Index` 还支持在 `Table` 内部“内联”定义，使用字符串名称标识列：

```py
metadata_obj = MetaData()
mytable = Table(
    "mytable",
    metadata_obj,
    Column("col1", Integer),
    Column("col2", Integer),
    Column("col3", Integer),
    Column("col4", Integer),
    # place an index on col1, col2
    Index("idx_col12", "col1", "col2"),
    # place a unique index on col3, col4
    Index("idx_col34", "col3", "col4", unique=True),
)
```

`Index` 对象还支持自己的 `create()` 方法：

```py
i = Index("someindex", mytable.c.col5)
i.create(engine)
CREATE  INDEX  someindex  ON  mytable  (col5) 
```

### 函数索引

`索引` 支持 SQL 和函数表达式，正如目标后端所支持的那样。要针对列使用降序值创建索引，可以使用 `ColumnElement.desc()` 修饰符：

```py
from sqlalchemy import Index

Index("someindex", mytable.c.somecol.desc())
```

或者使用支持功能性索引的后端，比如 PostgreSQL，可以使用 `lower()` 函数创建“不区分大小写”的索引：

```py
from sqlalchemy import func, Index

Index("someindex", func.lower(mytable.c.somecol))
```

## 索引 API

| 对象名称 | 描述 |
| --- | --- |
| Index | 表级别的索引。 |

```py
class sqlalchemy.schema.Index
```

表级别的索引。

定义一个复合（一个或多个列）索引。

例如：

```py
sometable = Table("sometable", metadata,
                Column("name", String(50)),
                Column("address", String(100))
            )

Index("some_index", sometable.c.name)
```

对于一个简单的单列索引，添加 `Column` 也支持 `index=True`：

```py
sometable = Table("sometable", metadata,
                Column("name", String(50), index=True)
            )
```

对于一个复合索引，可以指定多列：

```py
Index("some_index", sometable.c.name, sometable.c.address)
```

也支持功能性索引，通常通过结合绑定到表的 `Column` 对象使用 `func` 构造来实现：

```py
Index("some_index", func.lower(sometable.c.name))
```

`Index` 也可以通过内联声明或使用 `Table.append_constraint()` 与 `Table` 手动关联。当使用这种方法时，索引列的名称可以指定为字符串：

```py
Table("sometable", metadata,
                Column("name", String(50)),
                Column("address", String(100)),
                Index("some_index", "name", "address")
        )
```

为了在此形式中支持功能性或基于表达式的索引，可以使用 `text()` 构造：

```py
from sqlalchemy import text

Table("sometable", metadata,
                Column("name", String(50)),
                Column("address", String(100)),
                Index("some_index", text("lower(name)"))
        )
```

另请参阅

索引 - 关于 `Index` 的一般信息。

PostgreSQL 特定索引选项 - 用于 `Index` 构造的特定于 PostgreSQL 的选项。

MySQL / MariaDB 特定索引选项 - 用于 `Index` 构造的特定于 MySQL 的选项。

聚集索引支持 - 用于 `Index` 构造的特定于 MSSQL 的选项。

**成员**

__init__(), argument_for(), create(), ddl_if(), dialect_kwargs, dialect_options, drop(), info, kwargs

**类签名**

类 `sqlalchemy.schema.Index` (`sqlalchemy.sql.base.DialectKWArgs`, `sqlalchemy.schema.ColumnCollectionMixin`, `sqlalchemy.schema.HasConditionalDDL`, `sqlalchemy.schema.SchemaItem`)

```py
method __init__(name: str | None, *expressions: _DDLColumnArgument, unique: bool = False, quote: bool | None = None, info: _InfoType | None = None, _table: Table | None = None, _column_flag: bool = False, **dialect_kw: Any) → None
```

构造索引对象。

参数：

+   `name` – 索引的名称

+   `*expressions` – 要包含在索引中的列表达式。这些表达式通常是 `Column` 的实例，但也可以是最终引用 `Column` 的任意 SQL 表达式。

+   `unique=False` – 仅限关键字参数；如果为 True，则创建唯一索引。

+   `quote=None` – 仅限关键字参数；是否对索引名称应用引号。其工作方式与 `Column.quote` 相同。

+   `info=None` – 可选数据字典，将填充到此对象的 `SchemaItem.info` 属性中。

+   `**dialect_kw` – 除上述未提及的其他关键字参数是方言特定的，并以`<dialectname>_<argname>`的形式传递。有关文档中列出的参数的详细信息，请参阅有关单个方言的文档 方言 。

```py
classmethod argument_for(dialect_name, argument_name, default)
```

*继承自* `DialectKWArgs.argument_for()` *方法的* `DialectKWArgs`

为此类添加一种新的特定于方言的关键字参数类型。

例如：

```py
Index.argument_for("mydialect", "length", None)

some_index = Index('a', 'b', mydialect_length=5)
```

`DialectKWArgs.argument_for()` 方法是一种通过每个参数的方式向 `DefaultDialect.construct_arguments` 字典添加额外参数的方法。该字典提供了各种模式级构造接受的参数名称列表，代表方言。

新方言通常应将此字典一次性指定为方言类的数据成员。通常情况下，临时添加参数名称的用例是用于终端用户代码，该代码还使用了自定义编译方案，该方案使用了额外的参数。

参数：

+   `dialect_name` – 方言的名称。方言必须是可定位的，否则会引发 `NoSuchModuleError`。方言还必须包括一个现有的 `DefaultDialect.construct_arguments` 集合，表示它参与关键字参数验证和默认系统，否则会引发 `ArgumentError`。如果方言不包括此集合，则可以代表该方言已经指定任何关键字参数。SQLAlchemy 中打包的所有方言都包括此集合，但对于第三方方言，支持可能有所不同。

+   `argument_name` – 参数的名称。

+   `default` – 参数的默认值。

```py
method create(bind: _CreateDropBind, checkfirst: bool = False) → None
```

为这个 `Index` 发出一个 `CREATE` 语句，使用给定的 `Connection` 或 `Engine` 进行连接。

另请参见

`MetaData.create_all()`.

```py
method ddl_if(dialect: str | None = None, callable_: DDLIfCallable | None = None, state: Any | None = None) → Self
```

*继承自* `HasConditionalDDL.ddl_if()` *方法的* `HasConditionalDDL`

对此模式项应用条件 DDL 规则。

这些规则的工作方式类似于 `ExecutableDDLElement.execute_if()` 可调用对象，额外的功能是可以在 DDL 编译阶段检查条件，例如 `CreateTable` 这样的构造。`HasConditionalDDL.ddl_if()` 目前也适用于 `Index` 构造以及所有 `Constraint` 构造。

参数:

+   `dialect` – 方言的字符串名称，或者字符串名称的元组，表示多个方言类型。

+   `callable_` – 一个可调用对象，其构造方式与 `ExecutableDDLElement.execute_if.callable_` 中描述的形式相同。

+   `state` – 如果存在，将传递给可调用对象的任意对象。

版本 2.0 中的新功能。

另请参见

控制约束和索引的 DDL 生成 - 背景和使用示例

```py
attribute dialect_kwargs
```

*继承自* `DialectKWArgs.dialect_kwargs` *属性的* `DialectKWArgs`

作为特定于方言的选项指定的关键字参数集合，用于此构造。

这里的参数以原始的`<dialect>_<kwarg>`格式呈现。仅包括实际传递的参数；不像`DialectKWArgs.dialect_options`集合，该集合包含此方言已知的所有选项，包括默认值。

该集合也是可写的；接受形式为`<dialect>_<kwarg>`的键，其值将被组装到选项列表中。

另请参阅

`DialectKWArgs.dialect_options` - 嵌套字典形式

```py
attribute dialect_options
```

*继承自* `DialectKWArgs.dialect_options` *属性的* `DialectKWArgs`

作为特定于方言的选项指定的关键字参数集合。

这是一个两级嵌套的注册表，键为`<dialect_name>`和`<argument_name>`。例如，`postgresql_where`参数可以定位为：

```py
arg = my_object.dialect_options['postgresql']['where']
```

从版本 0.9.2 开始。

另请参阅

`DialectKWArgs.dialect_kwargs` - 扁平字典形式

```py
method drop(bind: _CreateDropBind, checkfirst: bool = False) → None
```

使用给定的`Connection`或`Engine`进行此`Index`的`DROP`语句。

另请参阅

`MetaData.drop_all()`。

```py
attribute info
```

*继承自* `SchemaItem.info` *属性的* `SchemaItem`

与该对象关联的信息字典，允许将用户定义的数据与此`SchemaItem`关联。

当首次访问时，字典将自动生成。它也可以在某些对象的构造函数中指定，例如`Table`和`Column`。

```py
attribute kwargs
```

*继承自* `DialectKWArgs.kwargs` *属性的* `DialectKWArgs`

`DialectKWArgs.dialect_kwargs` 的一个同义词。

## 定义外键

在 SQL 中，*外键*是一个表级构造，它限制该表中的一个或多个列只允许存在于另一组列中的值，通常但不总是位于不同的表中。我们将受到限制的列称为*外键*列，它们被约束到的列称为*引用*列。引用列几乎总是定义其所属表的主键，尽管也有例外情况。外键是连接具有彼此关系的行对的“接头部分”，在几乎每个操作中，SQLAlchemy 都将这个概念赋予了非常重要的意义。

在 SQLAlchemy 以及 DDL 中，外键约束可以作为表子句中的附加属性来定义，或者对于单列外键，它们可以选择地在单列的定义中指定。单列外键更常见，在列级别上通过将 `ForeignKey` 对象构造为 `Column` 对象的参数来指定：

```py
user_preference = Table(
    "user_preference",
    metadata_obj,
    Column("pref_id", Integer, primary_key=True),
    Column("user_id", Integer, ForeignKey("user.user_id"), nullable=False),
    Column("pref_name", String(40), nullable=False),
    Column("pref_value", String(100)),
)
```

在上面的示例中，我们为 `user_preference` 定义了一个新表，其中每行必须包含一个存在于 `user` 表的 `user_id` 列中的值。

`ForeignKey` 的参数最常见的是形式为 *<tablename>.<columnname>* 的字符串，或者对于远程架构或“拥有者”的表，形式为 *<schemaname>.<tablename>.<columnname>*。它也可以是一个实际的 `Column` 对象，稍后我们将看到，它通过其 `c` 集合从现有的 `Table` 对象中访问：

```py
ForeignKey(user.c.user_id)
```

使用字符串的优势在于，`user` 和 `user_preference` 之间的 python 链接只有在首次需要时才会解析，因此表对象可以轻松地分布在多个模块中，并且以任何顺序定义。

外键也可以在表级别定义，使用`ForeignKeyConstraint`对象。此对象可以描述单列或多列外键。多列外键被称为*复合*外键，并且几乎总是引用具有复合主键的表。下面我们定义一个具有复合主键的`invoice`表：

```py
invoice = Table(
    "invoice",
    metadata_obj,
    Column("invoice_id", Integer, primary_key=True),
    Column("ref_num", Integer, primary_key=True),
    Column("description", String(60), nullable=False),
)
```

然后一个具有引用`invoice`的复合外键的`invoice_item`表：

```py
invoice_item = Table(
    "invoice_item",
    metadata_obj,
    Column("item_id", Integer, primary_key=True),
    Column("item_name", String(60), nullable=False),
    Column("invoice_id", Integer, nullable=False),
    Column("ref_num", Integer, nullable=False),
    ForeignKeyConstraint(
        ["invoice_id", "ref_num"], ["invoice.invoice_id", "invoice.ref_num"]
    ),
)
```

需要注意的是，`ForeignKeyConstraint` 是定义复合外键的唯一方法。虽然我们也可以将单独的 `ForeignKey` 对象放置在`invoice_item.invoice_id`和`invoice_item.ref_num`列上，但 SQLAlchemy 不会意识到这两个值应该成对出现 - 它将成为两个单独的外键约束，而不是引用两列的单个复合外键。

### 通过 ALTER 创建/删除外键约束

我们在教程和其他地方看到的关于 DDL 中外键的行为表明，约束通常以“内联”的方式在 CREATE TABLE 语句中呈现，例如：

```py
CREATE  TABLE  addresses  (
  id  INTEGER  NOT  NULL,
  user_id  INTEGER,
  email_address  VARCHAR  NOT  NULL,
  PRIMARY  KEY  (id),
  CONSTRAINT  user_id_fk  FOREIGN  KEY(user_id)  REFERENCES  users  (id)
)
```

`CONSTRAINT .. FOREIGN KEY`指令用于在 CREATE TABLE 定义内“内联”创建约束。`MetaData.create_all()` 和 `MetaData.drop_all()` 方法默认使用拓扑排序对所有涉及的 `Table` 对象进行排序，以便按外键依赖关系的顺序创建和删除表（此排序也可通过 `MetaData.sorted_tables` 访问器获得）。

当涉及两个或更多外键约束的“依赖循环”时，此方法无法工作，在这种情况下，一组表彼此相互依赖，假设后端执行外键（SQLite 除外，MySQL/MyISAM 总是如此）。因此，该方法将在除 SQLite 之外的所有后端上将循环中的约束分解为单独的 ALTER 语句，因为 SQLite 不支持大多数形式的 ALTER。给定一个类似的模式：

```py
node = Table(
    "node",
    metadata_obj,
    Column("node_id", Integer, primary_key=True),
    Column("primary_element", Integer, ForeignKey("element.element_id")),
)

element = Table(
    "element",
    metadata_obj,
    Column("element_id", Integer, primary_key=True),
    Column("parent_node_id", Integer),
    ForeignKeyConstraint(
        ["parent_node_id"], ["node.node_id"], name="fk_element_parent_node_id"
    ),
)
```

当我们在后端（如 PostgreSQL 后端）调用`MetaData.create_all()`时，这两个表之间的循环被解决，约束被分别创建：

```py
>>> with engine.connect() as conn:
...     metadata_obj.create_all(conn, checkfirst=False)
CREATE  TABLE  element  (
  element_id  SERIAL  NOT  NULL,
  parent_node_id  INTEGER,
  PRIMARY  KEY  (element_id)
)

CREATE  TABLE  node  (
  node_id  SERIAL  NOT  NULL,
  primary_element  INTEGER,
  PRIMARY  KEY  (node_id)
)

ALTER  TABLE  element  ADD  CONSTRAINT  fk_element_parent_node_id
  FOREIGN  KEY(parent_node_id)  REFERENCES  node  (node_id)
ALTER  TABLE  node  ADD  FOREIGN  KEY(primary_element)
  REFERENCES  element  (element_id) 
```

为了对这些表发出 DROP，相同的逻辑适用，但是请注意，在 SQL 中，发出 DROP CONSTRAINT 需要约束具有名称。 在上述`'node'`表的情况下，我们没有命名此约束; 因此，系统将仅尝试为具有名称的约束发出 DROP:

```py
>>> with engine.connect() as conn:
...     metadata_obj.drop_all(conn, checkfirst=False)
ALTER  TABLE  element  DROP  CONSTRAINT  fk_element_parent_node_id
DROP  TABLE  node
DROP  TABLE  element 
```

在无法解析循环的情况下，例如，如果我们没有在这里为任一约束指定名称，我们将收到以下错误：

```py
sqlalchemy.exc.CircularDependencyError: Can't sort tables for DROP;
an unresolvable foreign key dependency exists between tables:
element, node.  Please ensure that the ForeignKey and ForeignKeyConstraint
objects involved in the cycle have names so that they can be dropped
using DROP CONSTRAINT.
```

此错误仅适用于 DROP 情况，因为在 CREATE 情况下我们可以不带名称发出“ADD CONSTRAINT”; 数据库通常会自动分配一个名称。

当手动解析依赖关系循环时，可以使用`ForeignKeyConstraint.use_alter` 和 `ForeignKey.use_alter` 关键字参数。 我们可以仅将此标志添加到`'element'`表中，如下所示：

```py
element = Table(
    "element",
    metadata_obj,
    Column("element_id", Integer, primary_key=True),
    Column("parent_node_id", Integer),
    ForeignKeyConstraint(
        ["parent_node_id"],
        ["node.node_id"],
        use_alter=True,
        name="fk_element_parent_node_id",
    ),
)
```

在我们的 CREATE DDL 中，我们将只看到这个约束的 ALTER 语句，而不是其他约束:

```py
>>> with engine.connect() as conn:
...     metadata_obj.create_all(conn, checkfirst=False)
CREATE  TABLE  element  (
  element_id  SERIAL  NOT  NULL,
  parent_node_id  INTEGER,
  PRIMARY  KEY  (element_id)
)

CREATE  TABLE  node  (
  node_id  SERIAL  NOT  NULL,
  primary_element  INTEGER,
  PRIMARY  KEY  (node_id),
  FOREIGN  KEY(primary_element)  REFERENCES  element  (element_id)
)

ALTER  TABLE  element  ADD  CONSTRAINT  fk_element_parent_node_id
FOREIGN  KEY(parent_node_id)  REFERENCES  node  (node_id) 
```

当与删除操作一起使用时，`ForeignKeyConstraint.use_alter` 和 `ForeignKey.use_alter` 关键字参数将要求约束具有名称，否则将生成以下错误：

```py
sqlalchemy.exc.CompileError: Can't emit DROP CONSTRAINT for constraint
ForeignKeyConstraint(...); it has no name
```

请参见

配置约束命名约定

`sort_tables_and_constraints()`  ### ON UPDATE and ON DELETE

大多数数据库支持外键值的级联，即当父行更新时，新值将放置在子行中，或者当父行删除时，所有相应的子行都将设置为 null 或删除。 在数据定义语言中，这些是使用诸如“ON UPDATE CASCADE”，“ON DELETE CASCADE”和“ON DELETE SET NULL”之类的短语来指定的，这些短语对应于外键约束。 “ON UPDATE”或“ON DELETE”后面的短语还可以允许其他与正在使用的数据库特定的短语相对应的短语。 `ForeignKey` 和 `ForeignKeyConstraint` 对象通过 `onupdate` 和 `ondelete` 关键字参数支持通过生成此子句。 值是任何字符串，将在适当的“ON UPDATE”或“ON DELETE”短语之后输出：

```py
child = Table(
    "child",
    metadata_obj,
    Column(
        "id",
        Integer,
        ForeignKey("parent.id", onupdate="CASCADE", ondelete="CASCADE"),
        primary_key=True,
    ),
)

composite = Table(
    "composite",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    Column("rev_id", Integer),
    Column("note_id", Integer),
    ForeignKeyConstraint(
        ["rev_id", "note_id"],
        ["revisions.id", "revisions.note_id"],
        onupdate="CASCADE",
        ondelete="SET NULL",
    ),
)
```

请注意，这些子句在与 MySQL 一起使用时需要`InnoDB`表。 它们在其他数据库上也可能不受支持。

请参见

关于与 ORM `relationship()` 构造的 `ON DELETE CASCADE` 集成的背景，请参见以下各节：

使用 ORM 关系中的外键 ON DELETE cascade

在多对多关系中使用外键 ON DELETE  ### 通过 ALTER 创建/删除外键约束

我们在教程和其他地方看到的涉及 DDL 的外键的行为表明，约束通常在 CREATE TABLE 语句中“内联”呈现，例如：

```py
CREATE  TABLE  addresses  (
  id  INTEGER  NOT  NULL,
  user_id  INTEGER,
  email_address  VARCHAR  NOT  NULL,
  PRIMARY  KEY  (id),
  CONSTRAINT  user_id_fk  FOREIGN  KEY(user_id)  REFERENCES  users  (id)
)
```

`CONSTRAINT .. FOREIGN KEY` 指令用于以“内联”方式在 CREATE TABLE 定义中创建约束。 `MetaData.create_all()` 和 `MetaData.drop_all()` 方法默认使用所有涉及的 `Table` 对象的拓扑排序，这样表就按照它们的外键依赖顺序创建和删除（此排序也可以通过 `MetaData.sorted_tables` 访问器获得）。

当涉及两个或更多个外键约束参与“依赖循环”时，此方法无法工作，在此循环中一组表相互依赖，假设后端强制执行外键（除 SQLite、MySQL/MyISAM 之外的情况始终如此）。因此，方法将在除不支持大多数 ALTER 形式的 SQLite 外的所有后端上将此类循环中的约束拆分为单独的 ALTER 语句。给定这样的模式：

```py
node = Table(
    "node",
    metadata_obj,
    Column("node_id", Integer, primary_key=True),
    Column("primary_element", Integer, ForeignKey("element.element_id")),
)

element = Table(
    "element",
    metadata_obj,
    Column("element_id", Integer, primary_key=True),
    Column("parent_node_id", Integer),
    ForeignKeyConstraint(
        ["parent_node_id"], ["node.node_id"], name="fk_element_parent_node_id"
    ),
)
```

当我们在诸如 PostgreSQL 后端之类的后端上调用 `MetaData.create_all()` 时，这两个表之间的循环被解决，并且约束被单独创建：

```py
>>> with engine.connect() as conn:
...     metadata_obj.create_all(conn, checkfirst=False)
CREATE  TABLE  element  (
  element_id  SERIAL  NOT  NULL,
  parent_node_id  INTEGER,
  PRIMARY  KEY  (element_id)
)

CREATE  TABLE  node  (
  node_id  SERIAL  NOT  NULL,
  primary_element  INTEGER,
  PRIMARY  KEY  (node_id)
)

ALTER  TABLE  element  ADD  CONSTRAINT  fk_element_parent_node_id
  FOREIGN  KEY(parent_node_id)  REFERENCES  node  (node_id)
ALTER  TABLE  node  ADD  FOREIGN  KEY(primary_element)
  REFERENCES  element  (element_id) 
```

为了发出这些表的 DROP 命令，相同的逻辑适用，但是请注意，SQL 中，要发出 DROP CONSTRAINT 需要约束具有名称。在上面的 `'node'` 表的情况下，我们没有为此约束命名；因此系统将尝试仅发出命名的约束的 DROP：

```py
>>> with engine.connect() as conn:
...     metadata_obj.drop_all(conn, checkfirst=False)
ALTER  TABLE  element  DROP  CONSTRAINT  fk_element_parent_node_id
DROP  TABLE  node
DROP  TABLE  element 
```

如果无法解决循环，例如我们在这里未给任何约束应用名称的情况，我们将收到以下错误：

```py
sqlalchemy.exc.CircularDependencyError: Can't sort tables for DROP;
an unresolvable foreign key dependency exists between tables:
element, node.  Please ensure that the ForeignKey and ForeignKeyConstraint
objects involved in the cycle have names so that they can be dropped
using DROP CONSTRAINT.
```

此错误仅适用于 DROP 案例，因为我们可以在 CREATE 案例中发出“ADD CONSTRAINT”而无需名称；数据库通常会自动分配一个名称。

`ForeignKeyConstraint.use_alter` 和 `ForeignKey.use_alter` 关键字参数可用于手动解决依赖循环。我们可以将此标志仅添加到 `'element'` 表中，如下所示：

```py
element = Table(
    "element",
    metadata_obj,
    Column("element_id", Integer, primary_key=True),
    Column("parent_node_id", Integer),
    ForeignKeyConstraint(
        ["parent_node_id"],
        ["node.node_id"],
        use_alter=True,
        name="fk_element_parent_node_id",
    ),
)
```

在我们的 CREATE DDL 中，我们将只看到该约束的 ALTER 语句，而不是其他的：

```py
>>> with engine.connect() as conn:
...     metadata_obj.create_all(conn, checkfirst=False)
CREATE  TABLE  element  (
  element_id  SERIAL  NOT  NULL,
  parent_node_id  INTEGER,
  PRIMARY  KEY  (element_id)
)

CREATE  TABLE  node  (
  node_id  SERIAL  NOT  NULL,
  primary_element  INTEGER,
  PRIMARY  KEY  (node_id),
  FOREIGN  KEY(primary_element)  REFERENCES  element  (element_id)
)

ALTER  TABLE  element  ADD  CONSTRAINT  fk_element_parent_node_id
FOREIGN  KEY(parent_node_id)  REFERENCES  node  (node_id) 
```

当与删除操作一起使用时，`ForeignKeyConstraint.use_alter` 和 `ForeignKey.use_alter` 需要命名约束，否则会生成以下错误：

```py
sqlalchemy.exc.CompileError: Can't emit DROP CONSTRAINT for constraint
ForeignKeyConstraint(...); it has no name
```

另见

配置约束命名约定

`sort_tables_and_constraints()`

### ON UPDATE 和 ON DELETE

大多数数据库支持外键值的*级联*，也就是当父行更新时，新值将放置在子行中，或者当父行删除时，所有相应的子行都设置为 null 或删除。在数据定义语言中，这些是使用诸如“ON UPDATE CASCADE”、“ON DELETE CASCADE”和“ON DELETE SET NULL”之类的短语指定的，对应于外键约束。在“ON UPDATE”或“ON DELETE”之后的短语可能还允许其他特定于正在使用的数据库的短语。 `ForeignKey` 和 `ForeignKeyConstraint` 对象支持通过 `onupdate` 和 `ondelete` 关键字参数生成此子句。该值是任何字符串，将在适当的“ON UPDATE”或“ON DELETE”短语之后输出：

```py
child = Table(
    "child",
    metadata_obj,
    Column(
        "id",
        Integer,
        ForeignKey("parent.id", onupdate="CASCADE", ondelete="CASCADE"),
        primary_key=True,
    ),
)

composite = Table(
    "composite",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    Column("rev_id", Integer),
    Column("note_id", Integer),
    ForeignKeyConstraint(
        ["rev_id", "note_id"],
        ["revisions.id", "revisions.note_id"],
        onupdate="CASCADE",
        ondelete="SET NULL",
    ),
)
```

请注意，当与 MySQL 一起使用时，这些子句需要 `InnoDB` 表。它们在其他数据库上也可能不受支持。

另见

关于将 `ON DELETE CASCADE` 与 ORM `relationship()` 构造集成的背景信息，请参见以下部分：

使用 ORM 关联的外键 ON DELETE cascade

在多对多关系中使用外键 ON DELETE

## 唯一约束

可以使用 `Column` 上的 `unique` 关键字匿名地在单个列上创建唯一约束。通过 `UniqueConstraint` 表级构造显式命名的唯一约束和/或具有多列的约束。

```py
from sqlalchemy import UniqueConstraint

metadata_obj = MetaData()
mytable = Table(
    "mytable",
    metadata_obj,
    # per-column anonymous unique constraint
    Column("col1", Integer, unique=True),
    Column("col2", Integer),
    Column("col3", Integer),
    # explicit/composite unique constraint.  'name' is optional.
    UniqueConstraint("col2", "col3", name="uix_1"),
)
```

## CHECK 约束

检查约束可以具有命名或未命名，并且可以在列级别或表级别上创建，使用 `CheckConstraint` 构造。检查约束的文本直接传递到数据库，因此具有有限的“数据库独立”行为。列级别的检查约束通常只应引用它们所放置的列，而表级别的约束可以引用表中的任何列。

请注意，一些数据库不支持主动支持检查约束，例如较旧版本的 MySQL（在 8.0.16 之前）。

```py
from sqlalchemy import CheckConstraint

metadata_obj = MetaData()
mytable = Table(
    "mytable",
    metadata_obj,
    # per-column CHECK constraint
    Column("col1", Integer, CheckConstraint("col1>5")),
    Column("col2", Integer),
    Column("col3", Integer),
    # table level CHECK constraint.  'name' is optional.
    CheckConstraint("col2 > col3 + 5", name="check1"),
)

mytable.create(engine)
CREATE  TABLE  mytable  (
  col1  INTEGER  CHECK  (col1>5),
  col2  INTEGER,
  col3  INTEGER,
  CONSTRAINT  check1  CHECK  (col2  >  col3  +  5)
) 
```

## 主键约束

任何 `Table` 对象的主键约束都是隐式存在的，基于标记有 `Column.primary_key` 标志的 `Column` 对象。`PrimaryKeyConstraint` 对象提供对此约束的显式访问，包括直接配置的选项：

```py
from sqlalchemy import PrimaryKeyConstraint

my_table = Table(
    "mytable",
    metadata_obj,
    Column("id", Integer),
    Column("version_id", Integer),
    Column("data", String(50)),
    PrimaryKeyConstraint("id", "version_id", name="mytable_pk"),
)
```

另请参阅

`PrimaryKeyConstraint` - 详细的 API 文档。

## 使用 Declarative ORM 扩展时设置约束

`Table` 是 SQLAlchemy 核心的构造，允许定义表元数据，这些元数据可以被 SQLAlchemy ORM 用作映射类的目标之一。Declarative 扩展允许自动创建 `Table` 对象，主要是将表的内容作为 `Column` 对象的映射。

要将诸如 `ForeignKeyConstraint` 等表级约束对象应用于使用 Declarative 定义的表，请使用 `__table_args__` 属性，详见 表配置。

## 配置约束命名约定

关系数据库通常为所有约束和索引分配显式名称。在创建表时使用`CREATE TABLE`的常见情况下，约束（如 CHECK、UNIQUE 和 PRIMARY KEY 约束）会与表定义一起内联生成，如果未另有规定，则数据库通常会自动分配名称给这些约束。在使用诸如`ALTER TABLE`之类的命令在数据库中更改现有数据库表时，此命令通常需要为新约束指定显式名称，以及能够指定要删除或修改的现有约束的名称。

可以使用`Constraint.name`参数和索引的`Index.name`参数明确地为约束命名。然而，在约束的情况下，此参数是可选的。还有使用`Column.unique`和`Column.index`参数的用例，这些参数会创建未指定显式名称的`UniqueConstraint`和`Index`对象。

通过架构迁移工具，如[Alembic](https://alembic.sqlalchemy.org/)，可以处理现有表格和约束的更改用例。但是，目前既不是 Alembic 也不是 SQLAlchemy 创建约束对象的名称，除非另有规定，否则导致能够更改现有约束的情况，这意味着必须逆向工程关系数据库用于自动分配名称的命名系统，或者必须小心确保所有约束都有名称。

与不得不为所有`Constraint`和`Index`对象分配显式名称相比，可以使用事件构建自动命名方案。这种方法的优点是，约束将获得一致的命名方案，无需在代码中的所有位置都使用显式名称参数，而且约定也会对由`Column.unique`和`Column.index`参数生成的约束和索引同样适用。从 SQLAlchemy 0.9.2 开始，包含了这种基于事件的方法，可以使用参数`MetaData.naming_convention`进行配置。

### 为 MetaData 集合配置命名约定

`MetaData.naming_convention` 指的是一个字典，接受 `Index` 类或单独的 `Constraint` 类作为键，并接受 Python 字符串模板作为值。它还接受一系列字符串代码作为替代键，分别为外键、主键、索引、检查和唯一约束的 `"fk"`、`"pk"`、`"ix"`、`"ck"`、`"uq"`。在这个字典中的字符串模板在与此 `MetaData` 对象关联的约束或索引没有给出现有名称时使用（包括一个例外情况，即可以进一步装饰现有名称的情况）。

适用于基本情况的一个示例命名约定如下：

```py
convention = {
    "ix": "ix_%(column_0_label)s",
    "uq": "uq_%(table_name)s_%(column_0_name)s",
    "ck": "ck_%(table_name)s_%(constraint_name)s",
    "fk": "fk_%(table_name)s_%(column_0_name)s_%(referred_table_name)s",
    "pk": "pk_%(table_name)s",
}

metadata_obj = MetaData(naming_convention=convention)
```

上述约定将为目标 `MetaData` 集合中的所有约束建立名称。例如，当我们创建一个未命名的 `UniqueConstraint` 时，我们可以观察到生成的名称：

```py
>>> user_table = Table(
...     "user",
...     metadata_obj,
...     Column("id", Integer, primary_key=True),
...     Column("name", String(30), nullable=False),
...     UniqueConstraint("name"),
... )
>>> list(user_table.constraints)[1].name
'uq_user_name'
```

即使只是使用 `Column.unique` 标志，这个相同的特性也会生效：

```py
>>> user_table = Table(
...     "user",
...     metadata_obj,
...     Column("id", Integer, primary_key=True),
...     Column("name", String(30), nullable=False, unique=True),
... )
>>> list(user_table.constraints)[1].name
'uq_user_name'
```

命名约定方法的一个关键优势是名称在 Python 构造时建立，而不是在 DDL 发射时建立。使用 Alembic 的 `--autogenerate` 功能时的影响是，当生成新的迁移脚本时，命名约定将是明确的：

```py
def upgrade():
    op.create_unique_constraint("uq_user_name", "user", ["name"])
```

上面的 `"uq_user_name"` 字符串是从我们的元数据中 `--autogenerate` 定位到的 `UniqueConstraint` 对象中复制的。

可用的标记包括 `%(table_name)s`、`%(referred_table_name)s`、`%(column_0_name)s`、`%(column_0_label)s`、`%(column_0_key)s`、`%(referred_column_0_name)s`，以及每个的多列版本，包括 `%(column_0N_name)s`、`%(column_0_N_name)s`、`%(referred_column_0_N_name)s`，它们以带有或不带有下划线的形式呈现所有列名称。关于 `MetaData.naming_convention` 的文档对每个约定有进一步的详细信息。

### 默认命名规则

`MetaData.naming_convention`的默认值处理了 SQLAlchemy 长期以来的行为，即为使用`Column.index`参数创建的`Index`对象分配名称：

```py
>>> from sqlalchemy.sql.schema import DEFAULT_NAMING_CONVENTION
>>> DEFAULT_NAMING_CONVENTION
immutabledict({'ix': 'ix_%(column_0_label)s'})
```

### 长名称的截断

当一个生成的名称，特别是那些使用多列令牌的名称，超出了目标数据库的标识符长度限制时（例如，PostgreSQL 的限制为 63 个字符），名称将使用基于长名称的 md5 哈希的 4 字符后缀进行确定性截断。例如，以下命名约定将基于正在使用的列名称生成非常长的名称：

```py
metadata_obj = MetaData(
    naming_convention={"uq": "uq_%(table_name)s_%(column_0_N_name)s"}
)

long_names = Table(
    "long_names",
    metadata_obj,
    Column("information_channel_code", Integer, key="a"),
    Column("billing_convention_name", Integer, key="b"),
    Column("product_identifier", Integer, key="c"),
    UniqueConstraint("a", "b", "c"),
)
```

在 PostgreSQL 方言上，长度超过 63 个字符的名称将被截断，如下例所示：

```py
CREATE  TABLE  long_names  (
  information_channel_code  INTEGER,
  billing_convention_name  INTEGER,
  product_identifier  INTEGER,
  CONSTRAINT  uq_long_names_information_channel_code_billing_conventi_a79e
  UNIQUE  (information_channel_code,  billing_convention_name,  product_identifier)
)
```

上述后缀`a79e`是基于长名称的 md5 哈希值，并且每次都会生成相同的值，以便为给定模式生成一致的名称。

### 创建用于命名约定的自定义令牌

还可以通过在 naming_convention 字典中指定一个额外的令牌和一个可调用对象来添加新的令牌。例如，如果我们想要使用 GUID 方案来命名我们的外键约束，我们可以这样做：

```py
import uuid

def fk_guid(constraint, table):
    str_tokens = (
        [
            table.name,
        ]
        + [element.parent.name for element in constraint.elements]
        + [element.target_fullname for element in constraint.elements]
    )
    guid = uuid.uuid5(uuid.NAMESPACE_OID, "_".join(str_tokens).encode("ascii"))
    return str(guid)

convention = {
    "fk_guid": fk_guid,
    "ix": "ix_%(column_0_label)s",
    "fk": "fk_%(fk_guid)s",
}
```

上面，当我们创建一个新的`ForeignKeyConstraint`时，我们将获得如下名称：

```py
>>> metadata_obj = MetaData(naming_convention=convention)

>>> user_table = Table(
...     "user",
...     metadata_obj,
...     Column("id", Integer, primary_key=True),
...     Column("version", Integer, primary_key=True),
...     Column("data", String(30)),
... )
>>> address_table = Table(
...     "address",
...     metadata_obj,
...     Column("id", Integer, primary_key=True),
...     Column("user_id", Integer),
...     Column("user_version_id", Integer),
... )
>>> fk = ForeignKeyConstraint(["user_id", "user_version_id"], ["user.id", "user.version"])
>>> address_table.append_constraint(fk)
>>> fk.name
fk_0cd51ab5-8d70-56e8-a83c-86661737766d
```

另请参阅

`MetaData.naming_convention` - 用于额外的用法详情以及所有可用命名组件的列表。

[命名约束的重要性](https://alembic.sqlalchemy.org/en/latest/naming.html) - 在 Alembic 文档中。

版本 1.3.0 中的新功能：添加了多列命名令牌，如`%(column_0_N_name)s`。生成的名称如果超出目标数据库的字符限制将被确定性截断。

### 命名 CHECK 约束

`CheckConstraint`对象配置为针对任意 SQL 表达式，该表达式可以有任意数量的列，并且通常使用原始 SQL 字符串进行配置。因此，我们通常使用的与`CheckConstraint`配合使用的约定是，我们期望对象已经有一个名称，然后我们使用其他约定元素增强它。一个典型的约定是`"ck_%(table_name)s_%(constraint_name)s"`：

```py
metadata_obj = MetaData(
    naming_convention={"ck": "ck_%(table_name)s_%(constraint_name)s"}
)

Table(
    "foo",
    metadata_obj,
    Column("value", Integer),
    CheckConstraint("value > 5", name="value_gt_5"),
)
```

上述表将生成名称`ck_foo_value_gt_5`：

```py
CREATE  TABLE  foo  (
  value  INTEGER,
  CONSTRAINT  ck_foo_value_gt_5  CHECK  (value  >  5)
)
```

`CheckConstraint` 还支持`%(columns_0_name)s`令牌；我们可以通过确保在约束的表达式中使用`Column`或`column()`元素来使用此令牌，无论是通过将约束声明为表的一部分：

```py
metadata_obj = MetaData(naming_convention={"ck": "ck_%(table_name)s_%(column_0_name)s"})

foo = Table("foo", metadata_obj, Column("value", Integer))

CheckConstraint(foo.c.value > 5)
```

或者通过使用`column()`内联：

```py
from sqlalchemy import column

metadata_obj = MetaData(naming_convention={"ck": "ck_%(table_name)s_%(column_0_name)s"})

foo = Table(
    "foo", metadata_obj, Column("value", Integer), CheckConstraint(column("value") > 5)
)
```

两者都将生成名称`ck_foo_value`：

```py
CREATE  TABLE  foo  (
  value  INTEGER,
  CONSTRAINT  ck_foo_value  CHECK  (value  >  5)
)
```

对“列零”的名称确定是通过扫描给定表达式以查找列对象进行的。如果表达式中存在多个列，则扫描会使用确定性搜索，但是表达式的结构将确定哪一列被指定为“列零”。  ### 对布尔、枚举和其他模式类型进行命名配置

`SchemaType` 类引用诸如 `Boolean` 和 `Enum` 之类的类型对象，这些对象生成伴随类型的 CHECK 约束。此处约束的名称最直接通过发送“name”参数设置，例如 `Boolean.name`：

```py
Table("foo", metadata_obj, Column("flag", Boolean(name="ck_foo_flag")))
```

命名约定功能也可以与这些类型结合使用，通常是通过使用包含`%(constraint_name)s`的约定，然后将名称应用于类型：

```py
metadata_obj = MetaData(
    naming_convention={"ck": "ck_%(table_name)s_%(constraint_name)s"}
)

Table("foo", metadata_obj, Column("flag", Boolean(name="flag_bool")))
```

上述表将产生约束名称`ck_foo_flag_bool`：

```py
CREATE  TABLE  foo  (
  flag  BOOL,
  CONSTRAINT  ck_foo_flag_bool  CHECK  (flag  IN  (0,  1))
)
```

`SchemaType` 类使用特殊的内部符号，以便命名约定仅在 DDL 编译时确定。在 PostgreSQL 上，有一个原生的 BOOLEAN 类型，因此不需要 `Boolean` 的 CHECK 约束；我们可以安全地设置 `Boolean` 类型而不需要名称，即使对于检查约束已经设置了命名约定。如果我们在没有原生 BOOLEAN 类型的数据库上运行，例如 SQLite 或 MySQL，则仅会查阅此约定以获取 CHECK 约束。

CHECK 约束也可以使用`column_0_name`令牌，与`SchemaType`非常匹配，因为这些约束只有一列：

```py
metadata_obj = MetaData(naming_convention={"ck": "ck_%(table_name)s_%(column_0_name)s"})

Table("foo", metadata_obj, Column("flag", Boolean()))
```

上述模式将产生：

```py
CREATE  TABLE  foo  (
  flag  BOOL,
  CONSTRAINT  ck_foo_flag  CHECK  (flag  IN  (0,  1))
)
```

### 使用 ORM 声明性混合物时使用命名约定

当使用命名约定功能与 ORM 声明式 Mixins 时，每个实际表映射子类必须存在单独的约束对象。有关背景和示例，请参阅使用命名约定在 Mixins 上创建索引和约束的部分。

### 配置 MetaData 集合的命名约定

`MetaData.naming_convention`指的是一个字典，接受`Index`类或个别`Constraint`类作为键，以及 Python 字符串模板作为值。它还接受一系列字符串代码作为替代键，分别为外键、主键、索引、检查和唯一约束的 `"fk"`、`"pk"`、`"ix"`、`"ck"`、`"uq"`。这个字典中的字符串模板在与这个`MetaData`对象相关联的约束或索引没有给出现有名称时使用（包括一个现有名称可以进一步修饰的例外情况）。

适用于基本情况的示例命名约定如下：

```py
convention = {
    "ix": "ix_%(column_0_label)s",
    "uq": "uq_%(table_name)s_%(column_0_name)s",
    "ck": "ck_%(table_name)s_%(constraint_name)s",
    "fk": "fk_%(table_name)s_%(column_0_name)s_%(referred_table_name)s",
    "pk": "pk_%(table_name)s",
}

metadata_obj = MetaData(naming_convention=convention)
```

上述约定将为目标`MetaData`集合中的所有约束建立名称。例如，当我们创建一个未命名的`UniqueConstraint`时，我们可以观察到生成的名称。

```py
>>> user_table = Table(
...     "user",
...     metadata_obj,
...     Column("id", Integer, primary_key=True),
...     Column("name", String(30), nullable=False),
...     UniqueConstraint("name"),
... )
>>> list(user_table.constraints)[1].name
'uq_user_name'
```

即使我们只是使用`Column.unique`标志，同样的特性也会生效：

```py
>>> user_table = Table(
...     "user",
...     metadata_obj,
...     Column("id", Integer, primary_key=True),
...     Column("name", String(30), nullable=False, unique=True),
... )
>>> list(user_table.constraints)[1].name
'uq_user_name'
```

命名约定方法的一个关键优势是名称在 Python 构造时建立，而不是在 DDL 发射时。当使用 Alembic 的 `--autogenerate` 特性时，这个效果是命名约定在生成新的迁移脚本时将是明确的：

```py
def upgrade():
    op.create_unique_constraint("uq_user_name", "user", ["name"])
```

上述`"uq_user_name"`字符串是从我们的元数据中`--autogenerate`定位到的`UniqueConstraint`对象中复制的。

可用的标记包括`%(table_name)s`、`%(referred_table_name)s`、`%(column_0_name)s`、`%(column_0_label)s`、`%(column_0_key)s`、`%(referred_column_0_name)s`，以及每个的多列版本，包括`%(column_0N_name)s`、`%(column_0_N_name)s`、`%(referred_column_0_N_name)s`，它们用下划线分隔或不分隔所有列名。有关`MetaData.naming_convention`的文档还详细介绍了每个约定。

### 默认命名约定

`MetaData.naming_convention`的默认值处理了 SQLAlchemy 的长期行为，即为使用`Column.index`参数创建的`Index`对象分配名称：

```py
>>> from sqlalchemy.sql.schema import DEFAULT_NAMING_CONVENTION
>>> DEFAULT_NAMING_CONVENTION
immutabledict({'ix': 'ix_%(column_0_label)s'})
```

### 截断长名称

当生成的名称特别是那些使用多列令牌的名称，超出目标数据库的标识符长度限制时（例如，PostgreSQL 的限制为 63 个字符），名称将使用基于长名称的 md5 哈希的 4 字符后缀进行确定性截断。例如，给定以下命名约定，根据使用的列名称，将生成非常长的名称：

```py
metadata_obj = MetaData(
    naming_convention={"uq": "uq_%(table_name)s_%(column_0_N_name)s"}
)

long_names = Table(
    "long_names",
    metadata_obj,
    Column("information_channel_code", Integer, key="a"),
    Column("billing_convention_name", Integer, key="b"),
    Column("product_identifier", Integer, key="c"),
    UniqueConstraint("a", "b", "c"),
)
```

在 PostgreSQL 方言中，名称长度超过 63 个字符的将被截断，如以下示例所示：

```py
CREATE  TABLE  long_names  (
  information_channel_code  INTEGER,
  billing_convention_name  INTEGER,
  product_identifier  INTEGER,
  CONSTRAINT  uq_long_names_information_channel_code_billing_conventi_a79e
  UNIQUE  (information_channel_code,  billing_convention_name,  product_identifier)
)
```

上述后缀`a79e`是基于长名称的 md5 哈希值，并且每次生成相同的值，以便为给定模式生成一致的名称。

### 创建自定义令牌以用于命名约定

也可以通过在 naming_convention 字典中指定额外的令牌和可调用对象来添加新令牌。例如，如果我们想要使用 GUID 方案为外键约束命名，我们可以这样做：

```py
import uuid

def fk_guid(constraint, table):
    str_tokens = (
        [
            table.name,
        ]
        + [element.parent.name for element in constraint.elements]
        + [element.target_fullname for element in constraint.elements]
    )
    guid = uuid.uuid5(uuid.NAMESPACE_OID, "_".join(str_tokens).encode("ascii"))
    return str(guid)

convention = {
    "fk_guid": fk_guid,
    "ix": "ix_%(column_0_label)s",
    "fk": "fk_%(fk_guid)s",
}
```

上面，当我们创建一个新的`ForeignKeyConstraint`时，我们将得到以下名称：

```py
>>> metadata_obj = MetaData(naming_convention=convention)

>>> user_table = Table(
...     "user",
...     metadata_obj,
...     Column("id", Integer, primary_key=True),
...     Column("version", Integer, primary_key=True),
...     Column("data", String(30)),
... )
>>> address_table = Table(
...     "address",
...     metadata_obj,
...     Column("id", Integer, primary_key=True),
...     Column("user_id", Integer),
...     Column("user_version_id", Integer),
... )
>>> fk = ForeignKeyConstraint(["user_id", "user_version_id"], ["user.id", "user.version"])
>>> address_table.append_constraint(fk)
>>> fk.name
fk_0cd51ab5-8d70-56e8-a83c-86661737766d
```

另请参阅

`MetaData.naming_convention` - 有关更多使用详细信息以及所有可用命名组件的列表。

[命名约束的重要性](https://alembic.sqlalchemy.org/en/latest/naming.html) - Alembic 文档中的内容。

从版本 1.3.0 开始：添加了多列命名令牌，例如`%(column_0_N_name)s`。生成的名称如果超过目标数据库的字符限制，将被确定性地截断。

### 命名 CHECK 约束

`CheckConstraint`对象针对任意 SQL 表达式进行配置，该表达式可以有任意数量的列，而且通常使用原始 SQL 字符串进行配置。因此，与`CheckConstraint`一起使用的一种常见约定是，我们期望对象已经具有名称，然后我们使用其他约定元素来增强它。一个典型的约定是`"ck_%(table_name)s_%(constraint_name)s"`：

```py
metadata_obj = MetaData(
    naming_convention={"ck": "ck_%(table_name)s_%(constraint_name)s"}
)

Table(
    "foo",
    metadata_obj,
    Column("value", Integer),
    CheckConstraint("value > 5", name="value_gt_5"),
)
```

上述表将生成名称`ck_foo_value_gt_5`：

```py
CREATE  TABLE  foo  (
  value  INTEGER,
  CONSTRAINT  ck_foo_value_gt_5  CHECK  (value  >  5)
)
```

`CheckConstraint`还支持`%(columns_0_name)s`令牌；我们可以通过确保在约束表达式中使用`Column`或`column()`元素来利用这一点，无论是通过单独声明约束还是通过在表内：

```py
metadata_obj = MetaData(naming_convention={"ck": "ck_%(table_name)s_%(column_0_name)s"})

foo = Table("foo", metadata_obj, Column("value", Integer))

CheckConstraint(foo.c.value > 5)
```

或通过使用`column()`内联：

```py
from sqlalchemy import column

metadata_obj = MetaData(naming_convention={"ck": "ck_%(table_name)s_%(column_0_name)s"})

foo = Table(
    "foo", metadata_obj, Column("value", Integer), CheckConstraint(column("value") > 5)
)
```

两者都会生成名称为`ck_foo_value`的内容：

```py
CREATE  TABLE  foo  (
  value  INTEGER,
  CONSTRAINT  ck_foo_value  CHECK  (value  >  5)
)
```

“列零”的名称确定是通过扫描给定表达式中的列对象执行的。如果表达式中存在多个列，则扫描将使用确定性搜索，但表达式的结构将确定哪一列被标记为“列零”。

### 针对布尔型、枚举型和其他模式类型进行命名配置

`SchemaType`类引用诸如`Boolean`和`Enum`之类的类型对象，这些对象生成伴随类型的 CHECK 约束。此处约束的名称最直接通过发送“name”参数设置，例如`Boolean.name`：

```py
Table("foo", metadata_obj, Column("flag", Boolean(name="ck_foo_flag")))
```

命名约定功能也可以与这些类型结合使用，通常是使用包含`%(constraint_name)s`的约定，然后将名称应用于类型：

```py
metadata_obj = MetaData(
    naming_convention={"ck": "ck_%(table_name)s_%(constraint_name)s"}
)

Table("foo", metadata_obj, Column("flag", Boolean(name="flag_bool")))
```

上述表将产生约束名为`ck_foo_flag_bool`：

```py
CREATE  TABLE  foo  (
  flag  BOOL,
  CONSTRAINT  ck_foo_flag_bool  CHECK  (flag  IN  (0,  1))
)
```

`SchemaType`类使用特殊的内部符号，以便命名约定仅在 DDL 编译时确定。在 PostgreSQL 上，有一种原生的 BOOLEAN 类型，因此不需要`Boolean`的 CHECK 约束；即使有检查约定，我们也可以安全地设置一个不带名称的`Boolean`类型。只有在没有原生 BOOLEAN 类型的数据库（如 SQLite 或 MySQL）上运行时，才会咨询此约定以进行 CHECK 约束。

CHECK 约束也可以使用`column_0_name`令牌，这与`SchemaType`非常匹配，因为这些约束只有一个列：

```py
metadata_obj = MetaData(naming_convention={"ck": "ck_%(table_name)s_%(column_0_name)s"})

Table("foo", metadata_obj, Column("flag", Boolean()))
```

上述模式将产生：

```py
CREATE  TABLE  foo  (
  flag  BOOL,
  CONSTRAINT  ck_foo_flag  CHECK  (flag  IN  (0,  1))
)
```

### 使用 ORM 声明性 Mixin 配置命名约定

在使用命名约定功能与 ORM 声明性混合时，每个实际表映射的子类必须存在单独的约束对象。有关背景和示例，请参见使用命名约定在混合上创建索引和约束部分。

## 约束 API

| 对象名称 | 描述 |
| --- | --- |
| 检查约束 | 表级或列级检查约束。 |
| 列集合约束 | 代理列集合的约束。 |
| ColumnCollectionMixin | 一个`ColumnCollection`对象的`Column`集合。 |
| 约束 | 表级 SQL 约束。 |
| conv | 标记一个字符串，指示名称已经通过命名约定转换。 |
| 外键 | 定义两列之间的依赖关系。 |
| 外键约束 | 表级外键约束。 |
| 具有条件 DDL | 定义一个包括`HasConditionalDDL.ddl_if()`方法的类，允许对 DDL 进行条件渲染。 |
| 主键约束 | 表级主键约束。 |
| 唯一约束 | 表级唯一约束。 |

```py
class sqlalchemy.schema.Constraint
```

表级 SQL 约束。

`Constraint`作为可以与`Table`对象关联的一系列约束对象的基类，包括`PrimaryKeyConstraint`、`ForeignKeyConstraint`、`UniqueConstraint`和`CheckConstraint`。

**成员**

__init__(), argument_for(), copy(), ddl_if(), dialect_kwargs, dialect_options, info, kwargs

**类签名**

类 `sqlalchemy.schema.Constraint` (`sqlalchemy.sql.base.DialectKWArgs`, `sqlalchemy.schema.HasConditionalDDL`, `sqlalchemy.schema.SchemaItem`)

```py
method __init__(name: _ConstraintNameArgument = None, deferrable: bool | None = None, initially: str | None = None, info: _InfoType | None = None, comment: str | None = None, _create_rule: Any | None = None, _type_bound: bool = False, **dialect_kw: Any) → None
```

创建一个 SQL 约束。

参数：

+   `name` – 可选的，这个 `Constraint` 在数据库中的名称。

+   `deferrable` – 可选布尔值。如果设置，当为这个约束发出 DDL 时，会发出 DEFERRABLE 或 NOT DEFERRABLE。

+   `initially` – 可选字符串。如果设置，当为这个约束发出 DDL 时，会发出 INITIALLY <value>。

+   `info` – 可选的数据字典，将被填充到此对象的 `SchemaItem.info` 属性中。

+   `comment` –

    在外键约束创建时渲染 SQL 注释的可选字符串。

    > 新版本 2.0 中新增。

+   `**dialect_kw` – 附加的关键字参数是方言特定的，并以 `<dialectname>_<argname>` 的形式传递。有关每个方言的文档参数的详细信息，请参见 Dialects 中的文档。

+   `_create_rule` – 由一些也创建约束的数据类型内部使用。

+   `_type_bound` – 用于内部指示此约束与特定数据类型相关联。

```py
classmethod argument_for(dialect_name, argument_name, default)
```

*继承自* `DialectKWArgs.argument_for()` *方法的* `DialectKWArgs`

为这个类添加一种新的方言特定关键字参数。

例如：

```py
Index.argument_for("mydialect", "length", None)

some_index = Index('a', 'b', mydialect_length=5)
```

`DialectKWArgs.argument_for()` 方法是一种通过每个参数方式向 `DefaultDialect.construct_arguments` 字典添加额外参数的方法。该字典提供了接受方言各种架构级别构造的参数名称列表。

新方言通常应该一次性将此字典指定为方言类的数据成员。通常，对于使用自定义编译方案并消耗额外参数的端用户代码，额外添加参数名的用例是使用这些参数。

参数：

+   `dialect_name` – 方言的名称。方言必须是可定位的，否则会引发`NoSuchModuleError`异常。方言还必须包括一个现有的`DefaultDialect.construct_arguments`集合，表明它参与关键字参数验证和默认系统，否则会引发`ArgumentError`异常。如果方言不包含此集合，则可以代表该方言已经指定任何关键字参数。SQLAlchemy 中打包的所有方言都包含此集合，但对于第三方方言，支持可能有所不同。

+   `argument_name` – 参数的名称。

+   `default` – 参数的默认值。

```py
method copy(**kw: Any) → Self
```

从 1.4 版开始已弃用：`Constraint.copy()`方法已弃用，并将在未来的版本中移除。

```py
method ddl_if(dialect: str | None = None, callable_: DDLIfCallable | None = None, state: Any | None = None) → Self
```

*继承自* `HasConditionalDDL.ddl_if()` *方法的* `HasConditionalDDL`

将条件 DDL 规则应用于此模式项。

这些规则的工作方式类似于`ExecutableDDLElement.execute_if()`可调用对象，但增加了一个特性，即可以在 DDL 编译阶段为`CreateTable`等结构检查条件。`HasConditionalDDL.ddl_if()`目前适用于`Index`结构以及所有`Constraint`结构。

参数：

+   `dialect` – 方言的字符串名称，或者表示多个方言类型的字符串名称元组。

+   `callable_` – 使用与`ExecutableDDLElement.execute_if.callable_`描述的相同形式构造的可调用对象。

+   `state` – 如果存在，则将传递给可调用对象的任意对象。

2.0 版中的新功能。

另请参阅

控制约束和索引的 DDL 生成 - 背景和使用示例

```py
attribute dialect_kwargs
```

*继承自* `DialectKWArgs.dialect_kwargs` *属性的* `DialectKWArgs`

作为方言特定选项指定为关键字参数的集合。

这些参数以原始的 `<dialect>_<kwarg>` 格式呈现在这里。仅包括实际传递的参数；不像`DialectKWArgs.dialect_options` 集合那样，它包含了该方言已知的所有选项，包括默认值。

该集合也是可写的；键的形式为 `<dialect>_<kwarg>`，其中的值将被组装到选项列表中。

另请参阅

`DialectKWArgs.dialect_options` - 嵌套字典形式

```py
attribute dialect_options
```

*继承自* `DialectKWArgs.dialect_options` *属性的* `DialectKWArgs`

作为方言特定选项指定为关键字参数的集合。

这是一个两级嵌套的注册表，以 `<dialect_name>` 和 `<argument_name>` 为键。例如，`postgresql_where` 参数可以定位为：

```py
arg = my_object.dialect_options['postgresql']['where']
```

在版本 0.9.2 中新增。

另请参阅

`DialectKWArgs.dialect_kwargs` - 扁平字典形式

```py
attribute info
```

*继承自* `SchemaItem.info` *属性的* `SchemaItem`

与对象相关联的信息字典，允许将用户定义的数据与此`SchemaItem` 关联起来。

字典在首次访问时会自动生成。它也可以在某些对象的构造函数中指定，比如`Table` 和 `Column`。

```py
attribute kwargs
```

*继承自* `DialectKWArgs.kwargs` *属性的* `DialectKWArgs`

一个与`DialectKWArgs.dialect_kwargs` 同义的词语。

```py
class sqlalchemy.schema.ColumnCollectionMixin
```

一个`Column`对象的`ColumnCollection`。

此集合代表此对象引用的列。

```py
class sqlalchemy.schema.ColumnCollectionConstraint
```

代理列集合的约束。

**成员**

__init__(), argument_for(), columns, contains_column(), copy(), ddl_if(), dialect_kwargs, dialect_options, info, kwargs

**类签名**

class `sqlalchemy.schema.ColumnCollectionConstraint` (`sqlalchemy.schema.ColumnCollectionMixin`, `sqlalchemy.schema.Constraint`)

```py
method __init__(*columns: _DDLColumnArgument, name: _ConstraintNameArgument = None, deferrable: bool | None = None, initially: str | None = None, info: _InfoType | None = None, _autoattach: bool = True, _column_flag: bool = False, _gather_expressions: List[_DDLColumnArgument] | None = None, **dialect_kw: Any) → None
```

参数：

+   `*columns` – 一系列列名或列对象。

+   `name` – 可选，此约束在数据库中的名称。

+   `deferrable` – 可选布尔值。如果设置，当为此约束发���DDL 时，发出 DEFERRABLE 或 NOT DEFERRABLE。

+   `initially` – 可选字符串。如果设置，当为此约束发出 DDL 时，发出 INITIALLY <value>。</value>

+   `**dialect_kw` – 其他关键字参数，包括方言特定参数，将传播到`Constraint`超类。

```py
classmethod argument_for(dialect_name, argument_name, default)
```

*继承自* `DialectKWArgs` *的* `DialectKWArgs.argument_for()` *方法*

为这个类添加一种新的方言特定关键字参数。

例如：

```py
Index.argument_for("mydialect", "length", None)

some_index = Index('a', 'b', mydialect_length=5)
```

`DialectKWArgs.argument_for()` 方法是一种逐个参数地向`DefaultDialect.construct_arguments` 字典添加额外参数的方式。该字典提供了一个由方言代表接受各种模式级构造的参数名称列表。

新方言通常应一次性将此字典指定为方言类的数据成员。临时添加参数名称的用例通常是针对同时使用自定义编译方案的最终用户代码，该编译方案使用额外的参数。

参数：

+   `dialect_name` – 方言的名称。方言必须是可定位的，否则会引发 `NoSuchModuleError`。方言还必须包括一个现有的 `DefaultDialect.construct_arguments` 集合，表明它参与关键字参数的验证和默认系统，否则会引发 `ArgumentError`。如果方言不包括此集合，则可以代表此方言已经指定任何关键字参数。SQLAlchemy 中打包的所有方言都包括此集合，但是对于第三方方言，支持可能会有所不同。

+   `argument_name` – 参数的名称。

+   `default` – 参数的默认值。

```py
attribute columns: ReadOnlyColumnCollection[str, Column[Any]]
```

*继承自* `ColumnCollectionMixin.columns` *属性，来自于* `ColumnCollectionMixin`

一个 `ColumnCollection` 表示这个约束的列集合。

```py
method contains_column(col: Column[Any]) → bool
```

如果此约束包含给定的列，则返回 True。

注意，此对象还包含一个属性 `.columns`，它是一个 `ColumnCollection`，包含了 `Column` 对象。

```py
method copy(*, target_table: Table | None = None, **kw: Any) → ColumnCollectionConstraint
```

从版本 1.4 开始不推荐使用：`ColumnCollectionConstraint.copy()` 方法已弃用，将在未来的版本中删除。

```py
method ddl_if(dialect: str | None = None, callable_: DDLIfCallable | None = None, state: Any | None = None) → Self
```

*继承自* `HasConditionalDDL.ddl_if()` *方法，来自于* `HasConditionalDDL`

为此模式项应用条件性 DDL 规则。

这些规则的工作方式类似于 `ExecutableDDLElement.execute_if()` 可调用对象，额外的功能是可以在 DDL 编译阶段检查条件，例如 `CreateTable` 这样的结构。 `HasConditionalDDL.ddl_if()` 目前也适用于 `Index` 结构以及所有 `Constraint` 结构。

参数：

+   `dialect` – 方言的字符串名称，或指示多个方言类型的字符串名称的元组。

+   `callable_` – 一个可调用对象，其构造方式与 `ExecutableDDLElement.execute_if.callable_` 中描述的形式相同。

+   `state` – 如果存在，将传递给可调用对象的任意对象。

新版本 2.0 中新增。

另请参阅

控制约束和索引的 DDL 生成 - 背景和用法示例

```py
attribute dialect_kwargs
```

*从* `DialectKWArgs.dialect_kwargs` *属性继承*

作为此结构的方言特定选项指定的关键字参数集合。

这里的参数以其原始 `<dialect>_<kwarg>` 格式呈现。只包括实际传递的参数；与 `DialectKWArgs.dialect_options` 集合不同，后者包含此方言知道的所有选项，包括默认值。

此集合也是可写的；接受形式为 `<dialect>_<kwarg>` 的键，其值将组装到选项列表中。

另请参阅

`DialectKWArgs.dialect_options` - 嵌套字典形式

```py
attribute dialect_options
```

*从* `DialectKWArgs.dialect_options` *属性继承*

作为此结构的方言特定选项指定的关键字参数集合。

这是一个两级嵌套的注册表，键入为 `<dialect_name>` 和 `<argument_name>`。例如，`postgresql_where` 参数可以定位为：

```py
arg = my_object.dialect_options['postgresql']['where']
```

新版本 0.9.2 中新增。

另请参阅

`DialectKWArgs.dialect_kwargs` - 平面字典形式

```py
attribute info
```

*继承自* `SchemaItem.info` *属性的* `SchemaItem`

与对象关联的信息字典，允许将用户定义的数据与此 `SchemaItem` 关联。

字典在首次访问时会自动生成。也可以在某些对象的构造函数中指定，例如 `Table` 和 `Column`。

```py
attribute kwargs
```

*继承自* `DialectKWArgs.kwargs` *属性的* `DialectKWArgs`

`DialectKWArgs.dialect_kwargs` 的同义词。

```py
class sqlalchemy.schema.CheckConstraint
```

表或列级别的 CHECK 约束。

可以包含在表或列的定义中。

**成员**

__init__(), argument_for(), columns, contains_column(), copy(), ddl_if(), dialect_kwargs, dialect_options, info, kwargs

**类签名**

类 `sqlalchemy.schema.CheckConstraint` (`sqlalchemy.schema.ColumnCollectionConstraint`)

```py
method __init__(sqltext: _TextCoercedExpressionArgument[Any], name: _ConstraintNameArgument = None, deferrable: bool | None = None, initially: str | None = None, table: Table | None = None, info: _InfoType | None = None, _create_rule: Any | None = None, _autoattach: bool = True, _type_bound: bool = False, **dialect_kw: Any) → None
```

构造一个 CHECK 约束。

参数：

+   `sqltext` –

    包含约束定义的字符串，将直接使用，或者是一个 SQL 表达式构造。如果给定为字符串，则对象将转换为 `text()` 对象。如果文本字符串包含冒号字符，请使用反斜杠进行转义：

    ```py
    CheckConstraint(r"foo ~ E'a(?\:b|c)d")
    ```

    警告

    `CheckConstraint` 的`CheckConstraint.sqltext` 参数可以作为 Python 字符串参数传递，该字符串参数将被视为**可信任的 SQL 文本**并按照给定的方式呈现。**不要将不受信任的输入传递给此参数**。

+   `name` – 可选，约束的数据库中名称。

+   `deferrable` – 可选布尔值。如果设置，则在为此约束发出 DDL 时发出 DEFERRABLE 或 NOT DEFERRABLE。

+   `initially` – 可选字符串。如果设置，则在为此约束发出 DDL 时发出 INITIALLY <value>。</value>

+   `info` – 可选数据字典，将填充到此对象的`SchemaItem.info`属性中。

```py
classmethod argument_for(dialect_name, argument_name, default)
```

*继承自* `DialectKWArgs` *类的* `DialectKWArgs.argument_for()` *方法*

为此类添加一种新的方言特定关键字参数。

例如：

```py
Index.argument_for("mydialect", "length", None)

some_index = Index('a', 'b', mydialect_length=5)
```

`DialectKWArgs.argument_for()` 方法是向`DefaultDialect.construct_arguments`字典添加额外参数的一种逐参数方式。此字典为代表方言的各种模式级构造接受的参数名称提供了列表。

新方言通常应一次性将此字典指定为方言类的数据成员。通常情况下，用于添加参数名称的临时用例是终端用户代码，该代码还使用了自定义编译方案，该方案消耗了额外的参数。

参数：

+   `dialect_name` – 方言的名称。方言必须是可定位的，否则将引发`NoSuchModuleError`。方言还必须包括一个现有的`DefaultDialect.construct_arguments`集合，指示它参与关键字参数验证和默认系统，否则将引发`ArgumentError`。如果方言不包括此集合，则可以代表此方言已经指定任何关键字参数。SQLAlchemy 中打包的所有方言都包括此集合，但是对于第三方方言，支持可能有所不同。

+   `argument_name` – 参数的名称。

+   `default` – 参数的默认值。

```py
attribute columns: ReadOnlyColumnCollection[str, Column[Any]]
```

*继承自* `ColumnCollectionMixin` *的* `ColumnCollectionMixin.columns` *属性*

表示此约束的列集合。

```py
method contains_column(col: Column[Any]) → bool
```

*继承自* `ColumnCollectionConstraint` *的* `ColumnCollectionConstraint.contains_column()` *方法*

如果此约束包含给定列，则返回 True。

请注意，此对象还包含一个属性 `.columns`，它是 `Column` 对象的 `ColumnCollection`。

```py
method copy(*, target_table: Table | None = None, **kw: Any) → CheckConstraint
```

自版本 1.4 弃用：`CheckConstraint.copy()` 方法已弃用，将在未来的版本中移除。

```py
method ddl_if(dialect: str | None = None, callable_: DDLIfCallable | None = None, state: Any | None = None) → Self
```

*继承自* `HasConditionalDDL` *的* `HasConditionalDDL.ddl_if()` *方法*

将条件 DDL 规则应用于此模式项。

这些规则的工作方式与 `ExecutableDDLElement.execute_if()` 可调用对象类似，额外的功能是可以在 DDL 编译阶段检查标准，例如 `CreateTable` 构造。 `HasConditionalDDL.ddl_if()` 目前也适用于 `Index` 构造以及所有 `Constraint` 构造。

参数：

+   `dialect` – 方言的字符串名称，或字符串名称的元组，以指示多个方言类型。

+   `callable_` – 使用与`ExecutableDDLElement.execute_if.callable_`中描述的相同形式构建的可调用对象。

+   `state` – 将传递给可调用对象的任意对象（如果存在）。

自版本 2.0 新增。

另请参阅

控制约束和索引的 DDL 生成 - 背景和用法示例

```py
attribute dialect_kwargs
```

*继承自* `DialectKWArgs.dialect_kwargs` *属性的* `DialectKWArgs`

作为方言特定选项指定为此结构的关键字参数的集合。

这里的参数以其原始 `<dialect>_<kwarg>` 格式呈现。仅包含实际传递的参数；不像 `DialectKWArgs.dialect_options` 集合，该集合包含此方言已知的所有选项，包括默认值。

该集合也是可写的；接受的键的形式为 `<dialect>_<kwarg>`，其中值将被组装成选项列表。

另请参阅

`DialectKWArgs.dialect_options` - 嵌套字典形式

```py
attribute dialect_options
```

*继承自* `DialectKWArgs.dialect_options` *属性的* `DialectKWArgs`

作为方言特定选项指定为此结构的关键字参数的集合。

这是一个两级嵌套的注册表，键入 `<dialect_name>` 和 `<argument_name>`。例如，`postgresql_where` 参数可以定位为：

```py
arg = my_object.dialect_options['postgresql']['where']
```

版本 0.9.2 中新增。

另请参阅

`DialectKWArgs.dialect_kwargs` - 扁平字典形式

```py
attribute info
```

*继承自* `SchemaItem.info` *属性的* `SchemaItem`

与对象关联的信息字典，允许将用户定义的数据与此 `SchemaItem` 关联。

第一次访问时，字典会自动生成。它也可以在某些对象的构造函数中指定，例如 `Table` 和 `Column`。

```py
attribute kwargs
```

*继承自* `DialectKWArgs.kwargs` *属性的* `DialectKWArgs`

`DialectKWArgs.dialect_kwargs` 的同义词。

```py
class sqlalchemy.schema.ForeignKey
```

定义两列之间的依赖关系。

`ForeignKey` 被指定为 `Column` 对象的参数，例如：

```py
t = Table("remote_table", metadata,
    Column("remote_id", ForeignKey("main_table.id"))
)
```

请注意，`ForeignKey` 仅是一个标记对象，定义了两列之间的依赖关系。在所有情况下，实际约束都由 `ForeignKeyConstraint` 对象表示。当`ForeignKey`与一个 `Column` 相关联，而这个列又与一个 `Table` 相关联时，此对象将自动生成。反之，当 `ForeignKeyConstraint` 应用于一个 `Table` 时，`ForeignKey` 标记将自动在每个相关联的 `Column` 上存在，这些列也与约束对象相关联。

请注意，您不能使用 `ForeignKey` 对象定义“复合”外键约束，即多个父/子列的分组约束。要定义此分组，必须使用 `ForeignKeyConstraint` 对象，并应用于 `Table`。相关联的`ForeignKey`对象将自动创建。

与单个 `Column` 对象相关联的 `ForeignKey` 对象可在该列的 foreign_keys 集合中找到。

关于外键配置的更多示例在定义外键中。

**成员**

__init__(), argument_for(), column, copy(), dialect_kwargs, dialect_options, get_referent(), info, kwargs, references(), target_fullname

**类签名**

类 `sqlalchemy.schema.ForeignKey` (`sqlalchemy.sql.base.DialectKWArgs`, `sqlalchemy.schema.SchemaItem`)

```py
method __init__(column: _DDLColumnArgument, _constraint: ForeignKeyConstraint | None = None, use_alter: bool = False, name: _ConstraintNameArgument = None, onupdate: str | None = None, ondelete: str | None = None, deferrable: bool | None = None, initially: str | None = None, link_to_name: bool = False, match: str | None = None, info: _InfoType | None = None, comment: str | None = None, _unresolvable: bool = False, **dialect_kw: Any)
```

构建一个列级的外键。

构造时生成 `ForeignKey` 对象，与父 `Table` 对象的约束集合关联的 `ForeignKeyConstraint`。

参数：

+   `column` – 键关系的单个目标列。`Column` 对象或列名称字符串：`tablename.columnkey` 或 `schema.tablename.columnkey`。`columnkey` 是分配给列的 `key`（默认为列名称本身），除非 `link_to_name` 为 `True`，此时使用列的呈现名称。

+   `name` – 可选字符串。如果未提供约束，则键的数据库名称。

+   `onupdate` – 可选字符串。如果设置，则在为此约束发出 DDL 时发出 ON UPDATE <value>。典型值包括 CASCADE、DELETE 和 RESTRICT。</value>

+   `ondelete` – 可选字符串。如果设置，则在为此约束发出 DDL 时发出 ON DELETE <value>。典型值包括 CASCADE、DELETE 和 RESTRICT。</value>

+   `deferrable` – 可选布尔值。如果设置，则在为此约束发出 DDL 时发出 DEFERRABLE 或 NOT DEFERRABLE。

+   `initially` – 可选字符串。如果设置，则在为此约束发出 DDL 时发出 INITIALLY <value>。</value>

+   `link_to_name` – 如果为 True，则 `column` 中给定的字符串名称是引用列的呈现名称，而不是其本地分配的 `key`。

+   `use_alter` –

    传递给底层 `ForeignKeyConstraint`，以指示约束应从 CREATE TABLE/ DROP TABLE 语句外部生成/删除。有关详细描述，请参阅 `ForeignKeyConstraint.use_alter`。

    另请参阅

    `ForeignKeyConstraint.use_alter`

    通过 ALTER 创建/删除外键约束

+   `match` – 可选字符串。如果设置，则在为此约束发出 DDL 时发出 MATCH <value>。典型值包括 SIMPLE、PARTIAL 和 FULL。</value>

+   `info` – 可选数据字典，将填充到此对象的 `SchemaItem.info` 属性中。

+   `comment` –

    可选字符串，将在创建外键约束时呈现 SQL 注释。

    > 版本 2.0 中的新内容。

+   `**dialect_kw` – 附加的关键字参数是方言特定的，并以 `<dialectname>_<argname>` 的形式传递。这些参数最终由相应的 `ForeignKeyConstraint` 处理。有关文档化参数的详细信息，请参阅 Dialects 中关于各个方言的文档。

```py
classmethod argument_for(dialect_name, argument_name, default)
```

*从* `DialectKWArgs` *的* `DialectKWArgs.argument_for()` *方法继承*

为这个类添加一种新的方言特定关键字参数。

例如：

```py
Index.argument_for("mydialect", "length", None)

some_index = Index('a', 'b', mydialect_length=5)
```

`DialectKWArgs.argument_for()` 方法是向 `DefaultDialect.construct_arguments` 字典添加额外参数的逐个参数方式。该字典提供了在方言代表各种模式级构造时代表方言接受的参数名称的列表。

新方言通常应将此字典一次性指定为方言类的数据成员。临时添加参数名称的用例通常用于同时使用自定义编译方案的最终用户代码，该编译方案使用额外的参数。

参数：

+   `dialect_name` – 方言的名称。方言必须是可定位的，否则会引发 `NoSuchModuleError`。方言还必须包括一个现有的 `DefaultDialect.construct_arguments` 集合，表示它参与关键字参数验证和默认系统，否则会引发 `ArgumentError`。如果方言不包括此集合，则已经可以代表该方言指定任何关键字参数。所有打包在 SQLAlchemy 中的方言都包括此集合，但对于第三方方言，支持可能有所不同。

+   `argument_name` – 参数的名称。

+   `default` – 参数的默认值。

```py
attribute column
```

返回由此 `ForeignKey` 引用的目标 `Column`。

如果没有建立目标列，则会引发异常。

```py
method copy(*, schema: str | None = None, **kw: Any) → ForeignKey
```

自版本 1.4 起弃用：`ForeignKey.copy()` 方法已弃用，并将在未来版本中移除。

```py
attribute dialect_kwargs
```

*继承自* `DialectKWArgs.dialect_kwargs` *属性的* `DialectKWArgs`

这是作为方言特定选项指定的关键字参数的集合。

这些参数以原始的 `<dialect>_<kwarg>` 格式显示在此处。仅包括实际传递的参数；与 `DialectKWArgs.dialect_options` 不同，后者包含此方言已知的所有选项，包括默认值。

该集合也是可写的；接受形式为 `<dialect>_<kwarg>` 的键，其中值将被组装成选项列表。

另请参阅

`DialectKWArgs.dialect_options` - 嵌套字典形式

```py
attribute dialect_options
```

*继承自* `DialectKWArgs.dialect_options` *属性的* `DialectKWArgs`

这是作为方言特定选项指定的关键字参数的集合。

这是一个两级嵌套的注册表，以 `<dialect_name>` 和 `<argument_name>` 为键。例如，`postgresql_where` 参数可以定位为：

```py
arg = my_object.dialect_options['postgresql']['where']
```

新版本为 0.9.2。

另请参阅

`DialectKWArgs.dialect_kwargs` - 平面字典形式

```py
method get_referent(table: FromClause) → Column[Any] | None
```

返回由此 `ForeignKey` 引用的给定 `Table`（或任何 `FromClause`）中的 `Column`。

如果此 `ForeignKey` 未引用给定的 `Table`，则返回 None。

```py
attribute info
```

*继承自* `SchemaItem.info` *属性的* `SchemaItem`

与对象关联的信息字典，允许将用户定义的数据与此 `SchemaItem` 关联。

第一次访问时，字典会自动生成。它也可以在某些对象的构造函数中指定，例如`Table`和`Column`。

```py
attribute kwargs
```

*从* `DialectKWArgs.kwargs` *属性继承* `DialectKWArgs`

`DialectKWArgs.dialect_kwargs`的同义词。

```py
method references(table: Table) → bool
```

如果给定的`Table`被此`ForeignKey`引用，则返回 True。

```py
attribute target_fullname
```

为此`ForeignKey`返回基于字符串的“列规范”。

这通常是传递给对象构造函数的基于字符串的“tablename.colname”参数的等效值。

```py
class sqlalchemy.schema.ForeignKeyConstraint
```

表级外键约束。

定义单列或复合外键引用约束。对于简单的、单列外键，向`Column`的定义中添加一个`ForeignKey`是一个简写等效于未命名的、单列`ForeignKeyConstraint`。

外键配置示例在定义外键中。

**成员**

__init__(), argument_for(), column_keys, columns, contains_column(), copy(), ddl_if(), dialect_kwargs, dialect_options, elements, info, kwargs, referred_table

**类签名**

类`sqlalchemy.schema.ForeignKeyConstraint`（`sqlalchemy.schema.ColumnCollectionConstraint`）

```py
method __init__(columns: _typing_Sequence[_DDLColumnArgument], refcolumns: _typing_Sequence[_DDLColumnArgument], name: _ConstraintNameArgument = None, onupdate: str | None = None, ondelete: str | None = None, deferrable: bool | None = None, initially: str | None = None, use_alter: bool = False, link_to_name: bool = False, match: str | None = None, table: Table | None = None, info: _InfoType | None = None, comment: str | None = None, **dialect_kw: Any) → None
```

构造一个支持复合的外键。

参数：

+   `columns` – 本地列名称的序列。这些命名列必须在父表中定义并存在。除非 `link_to_name` 为 True，否则名称应与每个列（默认为名称）给定的 `key` 匹配。

+   `refcolumns` – 外键列名称或 Column 对象的序列。这些列必须全部位于同一个表中。

+   `name` – 可选项，键的数据库内名称。

+   `onupdate` – 可选字符串。如果设置，则在为此约束发出 DDL 时发出 ON UPDATE <value>。典型值包括 CASCADE、DELETE 和 RESTRICT。

+   `ondelete` – 可选字符串。如果设置，则在为此约束发出 DDL 时发出 ON DELETE <value>。典型值包括 CASCADE、DELETE 和 RESTRICT。

+   `deferrable` – 可选布尔值。如果设置，则在发出 DDL 时发出 DEFERRABLE 或 NOT DEFERRABLE。

+   `initially` – 可选字符串。如果设置，则在为此约束发出 DDL 时发出 INITIALLY <value>。

+   `link_to_name` – 如果为 True，则 `column` 中给定的字符串名称是引用列的渲染名称，而不是其在本地分配的 `key`。

+   `use_alter` –

    如果为 True，则不会将此约束的 DDL 作为 CREATE TABLE 定义的一部分发出。相反，在完整的表集合创建之后，通过 ALTER TABLE 语句生成它，并在删除完整的表集合之前通过 ALTER TABLE 语句将其删除。

    使用 `ForeignKeyConstraint.use_alter` 特别适用于两个或多个表在相互依赖的外键约束关系中建立的情况；但是，`MetaData.create_all()` 和 `MetaData.drop_all()` 方法将自动执行此解析，因此通常不需要此标志。

    另请参阅

    通过 ALTER 创建/删除外键约束

+   `match` – 可选字符串。如果设置，则在为此约束发出 DDL 时发出 MATCH <value>。典型值包括 SIMPLE、PARTIAL 和 FULL。

+   `info` – 将填充到此对象的 `SchemaItem.info` 属性中的可选数据字典。

+   `comment` –

    在外键约束创建时渲染一个 SQL 注释的可选字符串。

    > 在 2.0 版中新增。

+   `**dialect_kw` – 额外的关键字参数是方言特定的，并以 `<方言名称>_<参数名称>` 的形式传递。有关文档化参数的详细信息，请参阅方言中有关各个方言的文档。

```py
classmethod argument_for(dialect_name, argument_name, default)
```

*继承自* `DialectKWArgs` *的* `DialectKWArgs.argument_for()` *方法*

为此类添加一种新的特定于方言的关键字参数。

例如：

```py
Index.argument_for("mydialect", "length", None)

some_index = Index('a', 'b', mydialect_length=5)
```

`DialectKWArgs.argument_for()`方法是向`DefaultDialect.construct_arguments`字典添加额外参数的每个参数的方法。此字典提供了各种模式级别构造函数可接受的参数名称列表。

新的方言通常应将此字典作为方言类的数据成员一次性指定。通常情况下，用于临时添加参数名称的用例是为了终端用户代码，该代码还使用自定义编译方案，该方案消耗附加参数。

参数：

+   `dialect_name` – 方言的名称。方言必须是可定位的，否则会引发`NoSuchModuleError`。方言还必须包括一个现有的`DefaultDialect.construct_arguments`集合，指示它参与关键字参数验证和默认系统，否则会引发`ArgumentError`。如果方言不包含此集合，则可以代表该方言指定任何关键字参数。SQLAlchemy 内置的所有方言都包含此集合，但对于第三方方言，支持可能有所不同。

+   `argument_name` – 参数的名称。

+   `default` – 参数的默认值。

```py
attribute column_keys
```

返回一个字符串键列表，表示此`ForeignKeyConstraint`中的本地列。

此列表是发送到`ForeignKeyConstraint`构造函数的原始字符串参数，或者如果约束已使用`Column`对象初始化，则是每个元素的字符串`.key`。

```py
attribute columns: ReadOnlyColumnCollection[str, Column[Any]]
```

*继承自* `ColumnCollectionMixin.columns` *属性*

代表此约束的`ColumnCollection`表示列的集合。

```py
method contains_column(col: Column[Any]) → bool
```

*继承自* `ColumnCollectionConstraint.contains_column()` *方法的* `ColumnCollectionConstraint`

如果此约束包含给定列，则返回 True。

请注意，这个对象还包含一个属性`.columns`，它是一个`ColumnCollection`对象的集合，其中包含`Column`对象。

```py
method copy(*, schema: str | None = None, target_table: Table | None = None, **kw: Any) → ForeignKeyConstraint
```

自版本 1.4 弃用：`ForeignKeyConstraint.copy()`方法已弃用，并将在将来的版本中移除。

```py
method ddl_if(dialect: str | None = None, callable_: DDLIfCallable | None = None, state: Any | None = None) → Self
```

*继承自* `HasConditionalDDL.ddl_if()` *方法的* `HasConditionalDDL`

将条件 DDL 规则应用于此模式项。

这些规则的工作方式类似于`ExecutableDDLElement.execute_if()`可调用对象，只是增加了一个特性，即在 DDL 编译阶段可以检查条件，例如`CreateTable`构造。`HasConditionalDDL.ddl_if()`当前还适用于`Index`构造以及所有`Constraint`构造。

参数：

+   `dialect` – 方言的字符串名称，或者一个字符串名称的元组，表示多个方言类型。

+   `callable_` – 使用与`ExecutableDDLElement.execute_if.callable_`描述的相同形式构造的可调用对象。

+   `state` – 任意的对象，如果存在，将传递给可调用对象。

从版本 2.0 开始新添加。

另请参阅

控制约束和索引的 DDL 生成 - 背景和使用示例

```py
attribute dialect_kwargs
```

*继承自* `DialectKWArgs.dialect_kwargs` *属性的* `DialectKWArgs`

此结构的特定方言选项的关键字参数集合。

这里的参数以其原始`<dialect>_<kwarg>`格式呈现。仅包含实际传递的参数；与`DialectKWArgs.dialect_options`集合不同，后者包含此方言已知的所有选项，包括默认值。

该集合也是可写的；键采用`<dialect>_<kwarg>`的形式，其中值将被组装成选项列表。

另请参阅

`DialectKWArgs.dialect_options` - 嵌套字典形式

```py
attribute dialect_options
```

*继承自* `DialectKWArgs.dialect_options` *属性的* `DialectKWArgs`

此结构的特定方言选项的关键字参数集合。

这是一个两级嵌套注册表，键入`<dialect_name>`和`<argument_name>`。例如，`postgresql_where`参数可以定位为：

```py
arg = my_object.dialect_options['postgresql']['where']
```

从版本 0.9.2 开始。

另请参阅

`DialectKWArgs.dialect_kwargs` - 平面字典形式

```py
attribute elements: List[ForeignKey]
```

一系列`ForeignKey`对象。

每个`ForeignKey`代表单个引用列/被引用列对。

此集合旨在为只读。

```py
attribute info
```

*继承自* `SchemaItem.info` *属性的* `SchemaItem`

与对象关联的信息字典，允许将用户定义的数据与此`SchemaItem`相关联。

字典在首次访问时自动生成。它也可以在某些对象的构造函数中指定，例如`Table`和`Column`。

```py
attribute kwargs
```

*继承自* `DialectKWArgs.kwargs` *属性，属于* `DialectKWArgs`

`DialectKWArgs.dialect_kwargs` 的同义词。

```py
attribute referred_table
```

此 `ForeignKeyConstraint` 引用的 `Table` 对象。

这是一个动态计算的属性，如果约束和/或父表尚未与包含所引用表的元数据集关联，则可能不可用。

```py
class sqlalchemy.schema.HasConditionalDDL
```

定义一个包含 `HasConditionalDDL.ddl_if()` 方法的类，允许对 DDL 进行条件渲染。

目前适用于约束和索引。

**成员**

ddl_if()

版本 2.0 中新增。

```py
method ddl_if(dialect: str | None = None, callable_: DDLIfCallable | None = None, state: Any | None = None) → Self
```

将一个条件 DDL 规则应用到此模式项。

这些规则的工作方式与 `ExecutableDDLElement.execute_if()` 可调用对象类似，但增加了一个功能，即可以在 DDL 编译阶段检查条件，例如 `CreateTable`。 `HasConditionalDDL.ddl_if()` 目前也适用于 `Index` 构造以及所有 `Constraint` 构造。

参数：

+   `dialect` – 方言的字符串名称，或字符串名称的元组，表示多个方言类型。

+   `callable_` – 使用与 `ExecutableDDLElement.execute_if.callable_` 描述的形式构建的可调用对象。

+   `state` – 如果存在，将传递给可调用对象的任意对象。

版本 2.0 中新增。

另请参阅

控制约束和索引的 DDL 生成 - 背景和使用示例

```py
class sqlalchemy.schema.PrimaryKeyConstraint
```

表级主键约束。

`PrimaryKeyConstraint` 对象会自动出现在任何 `Table` 对象上；它被分配一组与标记为 `Column.primary_key` 标志相对应的 `Column` 对象：

```py
>>> my_table = Table('mytable', metadata,
...                 Column('id', Integer, primary_key=True),
...                 Column('version_id', Integer, primary_key=True),
...                 Column('data', String(50))
...     )
>>> my_table.primary_key
PrimaryKeyConstraint(
 Column('id', Integer(), table=<mytable>,
 primary_key=True, nullable=False),
 Column('version_id', Integer(), table=<mytable>,
 primary_key=True, nullable=False)
)
```

`Table` 的主键也可以通过显式使用 `PrimaryKeyConstraint` 对象来指定；在这种用法模式下，“约束”的“名称”也可以指定，以及方言可能识别的其他选项：

```py
my_table = Table('mytable', metadata,
            Column('id', Integer),
            Column('version_id', Integer),
            Column('data', String(50)),
            PrimaryKeyConstraint('id', 'version_id',
                                 name='mytable_pk')
        )
```

两种列规范样式通常不应混合使用。如果 `PrimaryKeyConstraint` 中存在的列与标记为 `primary_key=True` 的列不匹配，则会发出警告，如果两者都存在；在这种情况下，列严格来自 `PrimaryKeyConstraint` 声明，并且其他标记为 `primary_key=True` 的列将被忽略。这种行为旨在与先前的行为向后兼容。

对于需要在 `PrimaryKeyConstraint` 上指定特定选项的用例，但仍然希望使用 `primary_key=True` 标志的常规样式的情况，可以指定一个空的 `PrimaryKeyConstraint`，它将从基于标志的 `Table` 中采用主键列集合：

```py
my_table = Table('mytable', metadata,
            Column('id', Integer, primary_key=True),
            Column('version_id', Integer, primary_key=True),
            Column('data', String(50)),
            PrimaryKeyConstraint(name='mytable_pk',
                                 mssql_clustered=True)
        )
```

**成员**

argument_for(), columns, contains_column(), copy(), ddl_if(), dialect_kwargs, dialect_options, info, kwargs

**类签名**

类 `sqlalchemy.schema.PrimaryKeyConstraint`（`sqlalchemy.schema.ColumnCollectionConstraint`）

```py
classmethod argument_for(dialect_name, argument_name, default)
```

*继承自* `DialectKWArgs.argument_for()` *方法的* `DialectKWArgs`

为此类添加一种新的方言特定的关键字参数。

例如：

```py
Index.argument_for("mydialect", "length", None)

some_index = Index('a', 'b', mydialect_length=5)
```

`DialectKWArgs.argument_for()` 方法是一种逐个参数地向 `DefaultDialect.construct_arguments` 字典添加额外参数的方式。此字典提供了由各种模式级构造接受的参数名称列表，代表方言。

新的方言通常应一次性指定此字典作为方言类的数据成员。临时添加参数名的用例通常是为了使用自定义编译方案的终端用户代码，该编译方案消耗额外的参数。

参数：

+   `dialect_name` – 方言的名称。 方言必须是可定位的，否则会引发 `NoSuchModuleError`。方言还必须包含一个现有的 `DefaultDialect.construct_arguments` 集合，表示它参与关键字参数验证和默认系统，否则会引发 `ArgumentError`。 如果方言不包含此集合，则可以为此方言代表已经指定任何关键字参数。SQLAlchemy 中打包的所有方言都包括此集合，但对于第三方方言，支持可能有所不同。

+   `argument_name` – 参数的名称。

+   `default` – 参数的默认值。

```py
attribute columns: ReadOnlyColumnCollection[str, Column[Any]]
```

*继承自* `ColumnCollectionMixin.columns` *属性的* `ColumnCollectionMixin`

表示此约束的列集合的 `ColumnCollection`。

```py
method contains_column(col: Column[Any]) → bool
```

*继承自* `ColumnCollectionConstraint.contains_column()` *方法的* `ColumnCollectionConstraint`

如果此约束包含给定列，则返回 True。

请注意，此对象还包含一个名为`.columns`的属性，它是`Column`对象的`ColumnCollection`。

```py
method copy(*, target_table: Table | None = None, **kw: Any) → ColumnCollectionConstraint
```

*继承自* `ColumnCollectionConstraint.copy()` *方法的* `ColumnCollectionConstraint`

自版本 1.4 起弃用：`ColumnCollectionConstraint.copy()`方法已弃用，并将在将来的版本中移除。

```py
method ddl_if(dialect: str | None = None, callable_: DDLIfCallable | None = None, state: Any | None = None) → Self
```

*继承自* `HasConditionalDDL.ddl_if()` *方法的* `HasConditionalDDL`

对此模式项应用条件 DDL 规则。

这些规则的工作方式类似于`ExecutableDDLElement.execute_if()`可调用对象，额外的特性是可以在 DDL 编译阶段检查条件，例如`CreateTable`这样的构造函数。`HasConditionalDDL.ddl_if()`目前也适用于`Index`构造以及所有`Constraint`构造。

参数：

+   `dialect` – 方言的字符串名称，或表示多个方言类型的字符串名称元组。

+   `callable_` – 使用与`ExecutableDDLElement.execute_if.callable_`中描述的相同形式构造的可调用对象。

+   `state` – 如果存在，将传递给可调用对象的任意对象。

从版本 2.0 开始新增。

另请参阅

控制约束和索引的 DDL 生成 - 背景和使用示例

```py
attribute dialect_kwargs
```

*继承自* `DialectKWArgs.dialect_kwargs` *属性的* `DialectKWArgs`

作为方言特定选项指定的关键字参数集合，用于此构造函数。

这里的参数以其原始`<dialect>_<kwarg>`格式呈现。仅包括实际传递的参数；不像`DialectKWArgs.dialect_options`集合，该集合包含此方言已知的所有选项，包括默认值。

该集合也是可写的；接受形式为`<dialect>_<kwarg>`的键，其中值将被组装到选项列表中。

另请参阅

`DialectKWArgs.dialect_options` - 嵌套字典形式

```py
attribute dialect_options
```

*继承自* `DialectKWArgs.dialect_options` *属性的* `DialectKWArgs`

作为方言特定选项指定给此结构的关键字参数集合。

这是一个两级嵌套的注册表，键为`<dialect_name>`和`<argument_name>`。例如，`postgresql_where`参数可定位为：

```py
arg = my_object.dialect_options['postgresql']['where']
```

新版本中新增的功能为 0.9.2。

另请参阅

`DialectKWArgs.dialect_kwargs` - 平面字典形式

```py
attribute info
```

*继承自* `SchemaItem.info` *属性的* `SchemaItem`

与对象关联的信息字典，允许将用户定义的数据与此`SchemaItem`相关联。

字典在首次访问时会自动生成。它也可以在某些对象的构造函数中指定，例如`Table`和`Column`。

```py
attribute kwargs
```

*继承自* `DialectKWArgs.kwargs` *属性的* `DialectKWArgs`

`DialectKWArgs.dialect_kwargs` 的同义词。

```py
class sqlalchemy.schema.UniqueConstraint
```

表级别的唯一约束。

定义单列或复合唯一约束。对于简单的单列约束，将`unique=True`添加到`Column`定义中相当于未命名的单列 UniqueConstraint 的简写等效形式。

**成员**

__init__(), argument_for(), columns, contains_column(), copy(), ddl_if(), dialect_kwargs, dialect_options, info, kwargs

**类签名**

类`sqlalchemy.schema.UniqueConstraint` (`sqlalchemy.schema.ColumnCollectionConstraint`)

```py
method __init__(*columns: _DDLColumnArgument, name: _ConstraintNameArgument = None, deferrable: bool | None = None, initially: str | None = None, info: _InfoType | None = None, _autoattach: bool = True, _column_flag: bool = False, _gather_expressions: List[_DDLColumnArgument] | None = None, **dialect_kw: Any) → None
```

*继承自* `sqlalchemy.schema.ColumnCollectionConstraint.__init__` *方法的* `ColumnCollectionConstraint`

参数：

+   `*columns` – 列名或列对象的序列。

+   `name` – 可选，此约束的数据库中的名称。

+   `deferrable` – 可选布尔值。 如果设置，为此约束发出 DEFERRABLE 或 NOT DEFERRABLE。

+   `initially` – 可选字符串。 如果设置，发出 INITIALLY <value>when 为此约束发出 DDL。</value>

+   `**dialect_kw` – 其他关键字参数，包括方言特定参数，将传递给`Constraint`超类。

```py
classmethod argument_for(dialect_name, argument_name, default)
```

*继承自* `DialectKWArgs.argument_for()` *方法的* `DialectKWArgs`

为这个类添加一种新的方言特定关键字参数。

例如：

```py
Index.argument_for("mydialect", "length", None)

some_index = Index('a', 'b', mydialect_length=5)
```

`DialectKWArgs.argument_for()` 方法是逐个添加额外参数到`DefaultDialect.construct_arguments` 字典的方法。 这个字典提供了由各种模式级构造接受的代表方言的参数名称的列表。

新的方言通常应该一次性指定这个字典作为方言类的数据成员。 临时添加参数名称的用例通常是为了使用自定义编译方案并消耗额外参数的最终用户代码。

参数：

+   `dialect_name` – 方言的名称。方言必须是可定位的，否则会引发 `NoSuchModuleError`。方言还必须包括一个现有的 `DefaultDialect.construct_arguments` 集合，表示它参与关键字参数验证和默认系统，否则会引发 `ArgumentError`。如果方言不包括此集合，则可以代表此方言已经指定任何关键字参数。SQLAlchemy 内置的所有方言都包括此集合，但对于第三方方言，支持可能有所不同。

+   `argument_name` – 参数的名称。

+   `default` – 参数的默认值。

```py
attribute columns: ReadOnlyColumnCollection[str, Column[Any]]
```

*从* `ColumnCollectionMixin` *的* `.columns` *属性继承*

代表此约束的一组列的 `ColumnCollection`。

```py
method contains_column(col: Column[Any]) → bool
```

*从* `ColumnCollectionConstraint` *的* `ColumnCollectionConstraint.contains_column()` *方法继承*

如果此约束包含给定列，则返回 True。

请注意，此对象还包含一个名为`.columns`的属性，它是 `Column` 对象的 `ColumnCollection`。

```py
method copy(*, target_table: Table | None = None, **kw: Any) → ColumnCollectionConstraint
```

*从* `ColumnCollectionConstraint` *的* `ColumnCollectionConstraint.copy()` *方法继承*

自版本 1.4 起已弃用：`ColumnCollectionConstraint.copy()` 方法已弃用，并将在将来的版本中删除。

```py
method ddl_if(dialect: str | None = None, callable_: DDLIfCallable | None = None, state: Any | None = None) → Self
```

*从* `HasConditionalDDL` *的* `HasConditionalDDL.ddl_if()` *方法继承*

对此模式项目应用条件 DDL 规则。

这些规则的工作方式类似于`ExecutableDDLElement.execute_if()`可调用对象，额外的特性是在 DDL 编译阶段可以检查条件，例如`CreateTable`构造中。`HasConditionalDDL.ddl_if()` 目前也适用于`Index`构造以及所有`Constraint`构造。

参数：

+   `dialect` – 方言的字符串名称，或者字符串名称的元组，表示多个方言类型。

+   `callable_` – 一个可调用对象，其构造方式与`ExecutableDDLElement.execute_if.callable_`中描述的形式相同。

+   `state` – 任意对象，如果存在将传递给可调用对象。

新版本 2.0。

另请参阅

控制约束和索引的 DDL 生成 - 背景和用法示例

```py
attribute dialect_kwargs
```

*继承自* `DialectKWArgs.dialect_kwargs` *属性的* `DialectKWArgs`

作为方言特定选项指定给此构造的关键字参数集合。

这里的参数以其原始的`<dialect>_<kwarg>`格式呈现。仅包括实际传递的参数；不像`DialectKWArgs.dialect_options`集合，该集合包含此方言已��的所有选项，包括默认值。

该集合也是可写的；键的形式为`<dialect>_<kwarg>`，其中值将被组装到选项列表中。

另请参阅

`DialectKWArgs.dialect_options` - 嵌套字典形式

```py
attribute dialect_options
```

*继承自* `DialectKWArgs.dialect_options` *属性的* `DialectKWArgs`

作为方言特定选项指定给此构造的关键字参数集合。

这是一个两级嵌套的注册表，键为`<dialect_name>`和`<argument_name>`。例如，`postgresql_where` 参数可以定位为：

```py
arg = my_object.dialect_options['postgresql']['where']
```

新版本 0.9.2。

另请参阅

`DialectKWArgs.dialect_kwargs` - 平坦的字典形式

```py
attribute info
```

*继承自* `SchemaItem.info` *属性的* `SchemaItem`

与对象关联的信息字典，允许将用户定义的数据与此 `SchemaItem` 关联起来。

字典在首次访问时会自动生成。它也可以在某些对象的构造函数中指定，例如 `Table` 和 `Column`。

```py
attribute kwargs
```

*继承自* `DialectKWArgs.kwargs` *属性的* `DialectKWArgs`

`DialectKWArgs.dialect_kwargs` 的同义词。

```py
function sqlalchemy.schema.conv(value: str, quote: bool | None = None) → Any
```

标记一个字符串，指示名称已经通过命名约定转换。

这是一个字符串子类，表示不应再受任何其他命名约定影响的名称。

例如 当我们使用以下命名约定创建 `Constraint` 时：

```py
m = MetaData(naming_convention={
    "ck": "ck_%(table_name)s_%(constraint_name)s"
})
t = Table('t', m, Column('x', Integer),
                CheckConstraint('x > 5', name='x5'))
```

上述约束的名称将呈现为 `"ck_t_x5"`。即，现有名称 `x5` 被用作命名约定中的 `constraint_name` 令牌。

在某些情况下，例如在迁移脚本中，我们可能会渲染上述 `CheckConstraint` 的名称已经被转换了。为了确保名称不会被双重修改，新名称使用 `conv()` 标记应用。我们可以明确使用如下：

```py
m = MetaData(naming_convention={
    "ck": "ck_%(table_name)s_%(constraint_name)s"
})
t = Table('t', m, Column('x', Integer),
                CheckConstraint('x > 5', name=conv('ck_t_x5')))
```

在上面的例子中，`conv()` 标记表示此处的约束名称是最终的，名称将呈现为 `"ck_t_x5"` 而不是 `"ck_t_ck_t_x5"`

另请参阅

配置约束命名约定

## 索引

索引可以匿名创建（使用自动生成的名称 `ix_<column label>`）为单列使用内联 `index` 关键字，在 `Column` 上，该关键字也修改了 `unique` 的用法，将唯一性应用于索引本身，而不是添加一个单独的 UNIQUE 约束。对于具有特定名称或涵盖多个列的索引，请使用 `Index` 结构，该结构需要一个名称。

下面我们展示了一个具有多个关联 `Index` 对象的 `Table`。DDL 为“CREATE INDEX”在表的创建语句之后发布：

```py
metadata_obj = MetaData()
mytable = Table(
    "mytable",
    metadata_obj,
    # an indexed column, with index "ix_mytable_col1"
    Column("col1", Integer, index=True),
    # a uniquely indexed column with index "ix_mytable_col2"
    Column("col2", Integer, index=True, unique=True),
    Column("col3", Integer),
    Column("col4", Integer),
    Column("col5", Integer),
    Column("col6", Integer),
)

# place an index on col3, col4
Index("idx_col34", mytable.c.col3, mytable.c.col4)

# place a unique index on col5, col6
Index("myindex", mytable.c.col5, mytable.c.col6, unique=True)

mytable.create(engine)
CREATE  TABLE  mytable  (
  col1  INTEGER,
  col2  INTEGER,
  col3  INTEGER,
  col4  INTEGER,
  col5  INTEGER,
  col6  INTEGER
)
CREATE  INDEX  ix_mytable_col1  ON  mytable  (col1)
CREATE  UNIQUE  INDEX  ix_mytable_col2  ON  mytable  (col2)
CREATE  UNIQUE  INDEX  myindex  ON  mytable  (col5,  col6)
CREATE  INDEX  idx_col34  ON  mytable  (col3,  col4) 
```

注意，在上面的示例中，`Index` 结构是外部创建的，与其对应的表使用 `Column` 对象直接创建。`Index` 也支持在 `Table` 内部“内联”定义，使用字符串名称来标识列：

```py
metadata_obj = MetaData()
mytable = Table(
    "mytable",
    metadata_obj,
    Column("col1", Integer),
    Column("col2", Integer),
    Column("col3", Integer),
    Column("col4", Integer),
    # place an index on col1, col2
    Index("idx_col12", "col1", "col2"),
    # place a unique index on col3, col4
    Index("idx_col34", "col3", "col4", unique=True),
)
```

`Index` 对象也支持其自己的 `create()` 方法：

```py
i = Index("someindex", mytable.c.col5)
i.create(engine)
CREATE  INDEX  someindex  ON  mytable  (col5) 
```

### 函数索引

`Index` 支持 SQL 和函数表达式，与目标后端支持的一样。要针对列使用降序值创建索引，可以使用 `ColumnElement.desc()` 修改器：

```py
from sqlalchemy import Index

Index("someindex", mytable.c.somecol.desc())
```

或者使用支持函数索引的后端，比如 PostgreSQL，可以使用 `lower()` 函数创建“不区分大小写”的索引：

```py
from sqlalchemy import func, Index

Index("someindex", func.lower(mytable.c.somecol))
```  ### 函数索引

`Index` 支持 SQL 和函数表达式，与目标后端支持的一样。要针对列使用降序值创建索引，可以使用 `ColumnElement.desc()` 修改器：

```py
from sqlalchemy import Index

Index("someindex", mytable.c.somecol.desc())
```

或者使用支持函数索引的后端，比如 PostgreSQL，可以使用 `lower()` 函数创建“不区分大小写”的索引：

```py
from sqlalchemy import func, Index

Index("someindex", func.lower(mytable.c.somecol))
```

## 索引 API

| 对象名称 | 描述 |
| --- | --- |
| 索引 | 一个表级索引。 |

```py
class sqlalchemy.schema.Index
```

一个表级索引。

定义一个复合（一个或多个列）索引。

例如：

```py
sometable = Table("sometable", metadata,
                Column("name", String(50)),
                Column("address", String(100))
            )

Index("some_index", sometable.c.name)
```

对于一个简单的、单列索引，添加 `Column` 也支持 `index=True`：

```py
sometable = Table("sometable", metadata,
                Column("name", String(50), index=True)
            )
```

对于复合索引，可以指定多列：

```py
Index("some_index", sometable.c.name, sometable.c.address)
```

功能性索引也得到支持，通常通过与绑定到表的`Column`对象一起使用`func`构造来实现：

```py
Index("some_index", func.lower(sometable.c.name))
```

`Index`也可以手动与`Table`关联，可以通过内联声明或使用`Table.append_constraint()`来实现。当使用此方法时，可以将索引列的名称指定为字符串：

```py
Table("sometable", metadata,
                Column("name", String(50)),
                Column("address", String(100)),
                Index("some_index", "name", "address")
        )
```

要支持此形式中的功能性或基于表达式的索引，可以使用`text()`构造：

```py
from sqlalchemy import text

Table("sometable", metadata,
                Column("name", String(50)),
                Column("address", String(100)),
                Index("some_index", text("lower(name)"))
        )
```

另请参阅

索引 - 有关`Index`的一般信息。

PostgreSQL 特定索引选项 - 适用于`Index`构造的 PostgreSQL 特定选项。

MySQL / MariaDB 特定索引选项 - MySQL 特定选项适用于`Index`构造。

集群索引支持 - MSSQL 特定选项适用于`Index`构造。

**成员**

__init__(), argument_for(), create(), ddl_if(), dialect_kwargs, dialect_options, drop(), info, kwargs

**类签名**

类`sqlalchemy.schema.Index` (`sqlalchemy.sql.base.DialectKWArgs`, `sqlalchemy.schema.ColumnCollectionMixin`, `sqlalchemy.schema.HasConditionalDDL`, `sqlalchemy.schema.SchemaItem`)

```py
method __init__(name: str | None, *expressions: _DDLColumnArgument, unique: bool = False, quote: bool | None = None, info: _InfoType | None = None, _table: Table | None = None, _column_flag: bool = False, **dialect_kw: Any) → None
```

构造一个索引对象。

参数：

+   `name` – 索引的名称

+   `*expressions` – 要包含在索引中的列表达式。这些表达式通常是`Column`的实例，但也可以是最终指向`Column`的任意 SQL 表达式。

+   `unique=False` – 仅限关键字参数；如果为 True，则创建一个唯一索引。

+   `quote=None` – 仅限关键字参数；是否对索引的名称应用引号。工作方式与`Column.quote`相同。

+   `info=None` – 可选数据字典，将填充到此对象的`SchemaItem.info` 属性中。

+   `**dialect_kw` – 上述未提及的额外关键字参数是方言特定的，并以`<dialectname>_<argname>`的形式传递。有关个别方言的文档参数的详细信息，请参阅方言的文档。

```py
classmethod argument_for(dialect_name, argument_name, default)
```

*继承自* `DialectKWArgs.argument_for()` *方法的* `DialectKWArgs`

为这个类添加一种新的方言特定的关键字参数。

例如：

```py
Index.argument_for("mydialect", "length", None)

some_index = Index('a', 'b', mydialect_length=5)
```

`DialectKWArgs.argument_for()` 方法是一种逐个添加额外参数到`DefaultDialect.construct_arguments` 字典的方式。该字典提供了在方言代表构造级别构造上接受的各种参数名称的列表。

新方言通常应将此字典一次性指定为方言类的数据成员。通常，对于需要临时添加参数名的用例，是用于终端用户代码，该代码还使用了自定义的编译方案，其中包含了额外的参数。

参数：

+   `dialect_name` - 方言的名称。方言必须是可定位的，否则会引发 `NoSuchModuleError`。方言还必须包括一个现有的 `DefaultDialect.construct_arguments` 集合，指示它参与关键字参数的验证和默认系统，否则会引发 `ArgumentError`。如果方言不包括此集合，则已为此方言指定任何关键字参数都是可以的。SQLAlchemy 中打包的所有方言都包含此集合，但对于第三方方言，支持可能会有所不同。

+   `argument_name` - 参数的名称。

+   `default` - 参数的默认值。

```py
method create(bind: _CreateDropBind, checkfirst: bool = False) → None
```

对于此 `Index`，使用给定的 `Connection` 或 `Engine` 发出 `CREATE` 语句以进行连接。

参见

`MetaData.create_all()`。

```py
method ddl_if(dialect: str | None = None, callable_: DDLIfCallable | None = None, state: Any | None = None) → Self
```

*继承自* `HasConditionalDDL.ddl_if()` *方法* `HasConditionalDDL`

将条件 DDL 规则应用于此模式项。

这些规则的工作方式类似于 `ExecutableDDLElement.execute_if()` 可调用对象，其附加功能是可以在 DDL 编译阶段检查条件，例如 `CreateTable` 构造。`HasConditionalDDL.ddl_if()` 目前也适用于 `Index` 构造以及所有 `Constraint` 构造。

参数：

+   `dialect` - 方言的字符串名称，或一组字符串名称以指示多个方言类型。

+   `callable_` - 使用与 `ExecutableDDLElement.execute_if.callable_` 中描述的相同形式构造的可调用对象。

+   `state` - 如果存在，将传递给可调用对象的任意对象。

从版本 2.0 开始新增。

参见

控制约束和索引的 DDL 生成 - 背景和用法示例

```py
attribute dialect_kwargs
```

*继承自* `DialectKWArgs.dialect_kwargs` *属性的* `DialectKWArgs`

作为方言特定选项的关键字参数集合，用于这个构造函数。

这里的参数以其原始的 `<dialect>_<kwarg>` 格式存在。只包括实际传递的参数；不像 `DialectKWArgs.dialect_options` 集合，该集合包含此方言已知的所有选项，包括默认值。

集合也是可写的；接受形式为 `<dialect>_<kwarg>` 的键，其值将被组装成选项列表。

另请参阅

`DialectKWArgs.dialect_options` - 嵌套的字典形式

```py
attribute dialect_options
```

*继承自* `DialectKWArgs.dialect_options` *属性的* `DialectKWArgs`

作为方言特定选项的关键字参数集合，用于这个构造函数。

这是一个两级嵌套的注册表，以 `<dialect_name>` 和 `<argument_name>` 为键。例如，`postgresql_where` 参数可以定位为：

```py
arg = my_object.dialect_options['postgresql']['where']
```

版本 0.9.2 中的新功能。

另请参阅

`DialectKWArgs.dialect_kwargs` - 平坦的字典形式

```py
method drop(bind: _CreateDropBind, checkfirst: bool = False) → None
```

使用给定的 `Connection` 或 `Engine` 进行连接性，为此 `Index` 发出 `DROP` 语句。

另请参阅

`MetaData.drop_all()`.

```py
attribute info
```

*继承自* `SchemaItem.info` *属性的* `SchemaItem`

与对象关联的信息字典，允许将用户定义的数据与此 `SchemaItem` 关联。

第一次访问时，字典会自动生成。它也可以在某些对象的构造函数中指定，比如`Table` 和 `Column`。

```py
attribute kwargs
```

*继承自* `DialectKWArgs.kwargs` *属性的* `DialectKWArgs`。

`DialectKWArgs.dialect_kwargs` 的同义词。
