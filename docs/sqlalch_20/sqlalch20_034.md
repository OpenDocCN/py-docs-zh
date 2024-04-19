# 邻接列表关系

> 原文：[`docs.sqlalchemy.org/en/20/orm/self_referential.html`](https://docs.sqlalchemy.org/en/20/orm/self_referential.html)

**邻接列表**模式是一种常见的关系模式，其中表包含对自身的外键引用，换句话说是**自引用关系**。这是在平面表中表示层次数据的最常见方法。其他方法包括**嵌套集**，有时称为“修改的先序”，以及**材料路径**。尽管在 SQL 查询中评估其流畅性时修改的先序具有吸引力，但邻接列表模型可能是满足大多数层次存储需求的最合适模式，原因是并发性、减少的复杂性，以及修改的先序对于能够完全加载子树到应用程序空间的应用程序几乎没有优势。

另请参阅

此部分详细说明了自引用关系的单表版本。有关使用第二个表作为关联表的自引用关系，请参阅自引用多对多关系部分。

在本示例中，我们将使用一个名为`Node`的单个映射类，表示树结构：

```py
class Node(Base):
    __tablename__ = "node"
    id = mapped_column(Integer, primary_key=True)
    parent_id = mapped_column(Integer, ForeignKey("node.id"))
    data = mapped_column(String(50))
    children = relationship("Node")
```

使用此结构，可以构建如下的图形：

```py
root --+---> child1
       +---> child2 --+--> subchild1
       |              +--> subchild2
       +---> child3
```

可以用数据表示为：

```py
id       parent_id     data
---      -------       ----
1        NULL          root
2        1             child1
3        1             child2
4        3             subchild1
5        3             subchild2
6        1             child3
```

这里的`relationship()`配置与“正常”的一对多关系的工作方式相同，唯一的例外是，“方向”，即关系是一对多还是多对一，默认假定为一对多。要将关系建立为多对一，需要添加一个额外的指令，称为`relationship.remote_side`，它是一个`Column`或一组`Column`对象，指示应该被视为“远程”的对象：

```py
class Node(Base):
    __tablename__ = "node"
    id = mapped_column(Integer, primary_key=True)
    parent_id = mapped_column(Integer, ForeignKey("node.id"))
    data = mapped_column(String(50))
    parent = relationship("Node", remote_side=[id])
```

在上述情况下，`id`列被应用为`parent` `relationship()`的`relationship.remote_side`，从而将`parent_id`建立为“本地”端，并且关系随后表现为多对一。

一如既往，两个方向可以结合成一个双向关系，使用两个由`relationship.back_populates`链接的`relationship()`构造。

```py
class Node(Base):
    __tablename__ = "node"
    id = mapped_column(Integer, primary_key=True)
    parent_id = mapped_column(Integer, ForeignKey("node.id"))
    data = mapped_column(String(50))
    children = relationship("Node", back_populates="parent")
    parent = relationship("Node", back_populates="children", remote_side=[id])
```

另请参阅

邻接列表 - 更新为 SQLAlchemy 2.0 的工作示例

## 复合邻接列表

邻接列表关系的一个子类别是在连接条件的“本地”和“远程”两侧都存在特定列的罕见情况。下面是`Folder`类的一个示例；使用复合主键，`account_id`列指向自身，以指示位于与父文件夹相同帐户内的子文件夹；而`folder_id`则指向该帐户内的特定文件夹：

```py
class Folder(Base):
    __tablename__ = "folder"
    __table_args__ = (
        ForeignKeyConstraint(
            ["account_id", "parent_id"], ["folder.account_id", "folder.folder_id"]
        ),
    )

    account_id = mapped_column(Integer, primary_key=True)
    folder_id = mapped_column(Integer, primary_key=True)
    parent_id = mapped_column(Integer)
    name = mapped_column(String)

    parent_folder = relationship(
        "Folder", back_populates="child_folders", remote_side=[account_id, folder_id]
    )

    child_folders = relationship("Folder", back_populates="parent_folder")
```

在上述示例中，我们将`account_id`传递到`relationship.remote_side`列表中。`relationship()`识别到这里的`account_id`列在两侧都存在，并将“远程”列与它识别为唯一存在于“远程”侧的`folder_id`列对齐。

## 自引用查询策略

查询自引用结构的方式与任何其他查询相同：

```py
# get all nodes named 'child2'
session.scalars(select(Node).where(Node.data == "child2"))
```

但是，当尝试沿着树的一个级别从一个外键连接到下一个级别时，需要额外小心。在 SQL 中，从表连接到自身的连接需要至少对表达式的一侧进行“别名”，以便可以明确引用它。

请回想一下 ORM 教程中的选择 ORM 别名，`aliased()`结构通常用于提供 ORM 实体的“别名”。使用此技术从`Node`连接到自身的连接如下所示：

```py
from sqlalchemy.orm import aliased

nodealias = aliased(Node)
session.scalars(
    select(Node)
    .where(Node.data == "subchild1")
    .join(Node.parent.of_type(nodealias))
    .where(nodealias.data == "child2")
).all()
SELECT  node.id  AS  node_id,
  node.parent_id  AS  node_parent_id,
  node.data  AS  node_data
FROM  node  JOIN  node  AS  node_1
  ON  node.parent_id  =  node_1.id
WHERE  node.data  =  ?
  AND  node_1.data  =  ?
['subchild1',  'child2'] 
```  ## 配置自引用的急切加载

在正常查询操作期间，通过从父表到子表的连接或外连接来发生关系的急切加载，以便可以从单个 SQL 语句或所有子集合的第二个语句中填充父对象及其直接子集合或引用。SQLAlchemy 的连接和子查询急切加载在连接到相关项时在所有情况下使用别名表，因此与自引用连接兼容。然而，要使用自引用关系进行急切加载，SQLAlchemy 需要告知应该连接和/或查询多少级深度；否则，急切加载将根本不会发生。此深度设置通过`relationships.join_depth`进行配置：

```py
class Node(Base):
    __tablename__ = "node"
    id = mapped_column(Integer, primary_key=True)
    parent_id = mapped_column(Integer, ForeignKey("node.id"))
    data = mapped_column(String(50))
    children = relationship("Node", lazy="joined", join_depth=2)

session.scalars(select(Node)).all()
SELECT  node_1.id  AS  node_1_id,
  node_1.parent_id  AS  node_1_parent_id,
  node_1.data  AS  node_1_data,
  node_2.id  AS  node_2_id,
  node_2.parent_id  AS  node_2_parent_id,
  node_2.data  AS  node_2_data,
  node.id  AS  node_id,
  node.parent_id  AS  node_parent_id,
  node.data  AS  node_data
FROM  node
  LEFT  OUTER  JOIN  node  AS  node_2
  ON  node.id  =  node_2.parent_id
  LEFT  OUTER  JOIN  node  AS  node_1
  ON  node_2.id  =  node_1.parent_id
[] 
```

## 复合邻接列表

邻接列表关系的一个子类别是在连接条件的“本地”和“远程”两侧都存在特定列的罕见情况。下面是`Folder`类的一个示例；使用复合主键，`account_id`列指向自身，以指示位于与父文件夹相同帐户内的子文件夹；而`folder_id`则指向该帐户内的特定文件夹：

```py
class Folder(Base):
    __tablename__ = "folder"
    __table_args__ = (
        ForeignKeyConstraint(
            ["account_id", "parent_id"], ["folder.account_id", "folder.folder_id"]
        ),
    )

    account_id = mapped_column(Integer, primary_key=True)
    folder_id = mapped_column(Integer, primary_key=True)
    parent_id = mapped_column(Integer)
    name = mapped_column(String)

    parent_folder = relationship(
        "Folder", back_populates="child_folders", remote_side=[account_id, folder_id]
    )

    child_folders = relationship("Folder", back_populates="parent_folder")
```

在上面的例子中，我们将`account_id`传递到`relationship.remote_side`列表中。`relationship()`识别到这里的`account_id`列在两侧均存在，并且将“远程”列与它识别为唯一存在于“远程”一侧的`folder_id`列对齐。

## 自引用查询策略

自引用结构的查询与任何其他查询相同：

```py
# get all nodes named 'child2'
session.scalars(select(Node).where(Node.data == "child2"))
```

但是，在尝试从树的一级到下一级进行连接时需要特别注意。在 SQL 中，从表连接到自身需要至少一个表达式的一侧被“别名”，以便可以明确地引用它。

请回想一下在 ORM 教程中选择 ORM 别名，`aliased()`构造通常用于提供 ORM 实体的“别名”。使用这种技术从`Node`到自身的连接看起来像：

```py
from sqlalchemy.orm import aliased

nodealias = aliased(Node)
session.scalars(
    select(Node)
    .where(Node.data == "subchild1")
    .join(Node.parent.of_type(nodealias))
    .where(nodealias.data == "child2")
).all()
SELECT  node.id  AS  node_id,
  node.parent_id  AS  node_parent_id,
  node.data  AS  node_data
FROM  node  JOIN  node  AS  node_1
  ON  node.parent_id  =  node_1.id
WHERE  node.data  =  ?
  AND  node_1.data  =  ?
['subchild1',  'child2'] 
```

## 配置自引用关系的急切加载

通过在正常查询操作期间从父表到子表使用连接或外连接来进行关系的急切加载，以便可以从单个 SQL 语句或所有直接子集合的第二个语句中填充父表及其直接子集合或引用。SQLAlchemy 的连接和子查询急切加载在加入相关项时始终使用别名表，因此与自引用连接兼容。然而，要想使用自引用关系的急切加载，需要告诉 SQLAlchemy 应该加入和/或查询多少级深度；否则，急切加载将根本不会发生。此深度设置通过`relationships.join_depth`进行配置：

```py
class Node(Base):
    __tablename__ = "node"
    id = mapped_column(Integer, primary_key=True)
    parent_id = mapped_column(Integer, ForeignKey("node.id"))
    data = mapped_column(String(50))
    children = relationship("Node", lazy="joined", join_depth=2)

session.scalars(select(Node)).all()
SELECT  node_1.id  AS  node_1_id,
  node_1.parent_id  AS  node_1_parent_id,
  node_1.data  AS  node_1_data,
  node_2.id  AS  node_2_id,
  node_2.parent_id  AS  node_2_parent_id,
  node_2.data  AS  node_2_data,
  node.id  AS  node_id,
  node.parent_id  AS  node_parent_id,
  node.data  AS  node_data
FROM  node
  LEFT  OUTER  JOIN  node  AS  node_2
  ON  node.id  =  node_2.parent_id
  LEFT  OUTER  JOIN  node  AS  node_1
  ON  node_2.id  =  node_1.parent_id
[] 
```
