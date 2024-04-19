# ORM 示例

> 原文：[`docs.sqlalchemy.org/en/20/orm/examples.html`](https://docs.sqlalchemy.org/en/20/orm/examples.html)

SQLAlchemy 发行版包含各种代码示例，展示了一组选择的模式，有些是典型的，有些则不太典型。所有示例都可以运行，并且可以在发行版的`/examples`目录中找到。这里可以找到所有示例的描述和源代码。

其他 SQLAlchemy 示例，一些是用户贡献的，可以在[`www.sqlalchemy.org/trac/wiki/UsageRecipes`](https://www.sqlalchemy.org/trac/wiki/UsageRecipes)的 wiki 上找到。

## 映射配方

### 邻接列表

一个使用邻接列表模型映射的字典-字典结构的示例。

例如：

```py
node = TreeNode('rootnode')
node.append('node1')
node.append('node3')
session.add(node)
session.commit()

dump_tree(node)
```

文件列表：

+   adjacency_list.py  ### 关联

展示了“关联对象”模式的使用示例，其中一个中间类在两个关联在多对多模式中的类之间进行关联。

文件列表：

+   proxied_association.py - 与 basic_association 相同的示例，增加了对`sqlalchemy.ext.associationproxy`的使用，以使对`OrderItem`的显式引用变为可选。

+   basic_association.py - 演示了“Order”和一组“Item”对象之间的多对多关系，通过名为“OrderItem”的关联对象将每个购买价格关联起来。

+   dict_of_sets_with_default.py - 一个高级的关联代理示例，演示了关联代理的嵌套，以生成多级 Python 集合，本例中是一个具有字符串键和整数集合值的字典，隐藏了底层映射类。  ### Asyncio 集成

展示 SQLAlchemy 的 asyncio 引擎特性的示例。

文件列表：

+   async_orm.py - 演示了`sqlalchemy.ext.asyncio.AsyncSession`对象用于异步 ORM 使用。

+   async_orm_writeonly.py - 演示了使用**只写关系**来更简单地处理异步 ORM 集合。

+   gather_orm_statements.py - 演示了如何使用`asyncio.gather()`在许多 asyncio 数据库连接上并发运行多个语句，将 ORM 结果合并到单个`AsyncSession`中。

+   basic.py - 演示了 asyncio 引擎/连接接口。

+   greenlet_orm.py - 演示了使用 sqlalchemy.ext.asyncio.AsyncSession 对象进行异步 ORM 使用，包括可选的 run_sync() 方法。  ### 有向图

有向图结构的持久性示例。图被存储为一组边，每条边都引用节点表中的“下级”和“上级”节点。演示了基本的持久性和查询下级和上级邻居的方法：

```py
n2 = Node(2)
n5 = Node(5)
n2.add_neighbor(n5)
print(n2.higher_neighbors())
```

文件清单：

+   directed_graph.py  ### 作为字典的动态关系

演示如何在“动态”关系之上放置类似字典的外观，以便字典操作（假设简单字符串键）可以在不一次加载完整集合的情况下操作大集合。

文件清单：

+   dynamic_dict.py  ### 通用关联

演示了将多种类型的父对象与特定子对象关联的各种方法。

所有示例都使用声明性扩展和声明性混合类。每个示例最后都呈现相同的用例 - 两个类，`Customer` 和 `Supplier`，都是 `HasAddresses` 混合类的子类，该混合类确保父类提供一个包含 `Address` 对象的 `addresses` 集合。

discriminator_on_association.py 和 generic_fk.py 脚本是在 2007 年博客文章 [使用 SQLAlchemy 实现多态关联](https://techspot.zzzeek.org/2007/05/29/polymorphic-associations-with-sqlalchemy/) 中提出的配方的现代化版本。

文件清单：

+   table_per_association.py - 演示了一个提供通用关联的混合类，通过为每个父类生成单独的关联表来实现。关联的对象本身存储在所有父类共享的单个表中。

+   table_per_related.py - 演示了一种通用关联，将关联对象持久化在各自的表中，每个表都生成用于代表特定父类持久化这些对象。

+   discriminator_on_association.py - 演示了一个提供通用关联的混合类，使用单个目标表和单个关联表，所有父表都引用该关联表。关联表包含一个“鉴别器”列，确定每个特定行与哪种类型的父对象关联。

+   generic_fk.py - 展示了所谓的“通用外键”，类似于流行框架（如 Django、ROR 等）的方式。这种方法绕过了标准的引用完整性实践，因为“外键”列实际上不被约束以引用任何特定的表；相反，在应用程序逻辑中使用逻辑来确定引用的是哪个表。### 物化路径

使用 SQLAlchemy ORM 展示了用于分层数据的“物化路径”模式。

文件列表：

+   materialized_paths.py - 说明了“物化路径”模式。### 嵌套集

使用 SQLAlchemy ORM 展示了一种实现用于分层数据的“嵌套集”模式的简单方法。

文件列表：

+   nested_sets.py - Celko 的“嵌套集”树结构。### 性能

一个用于各种 SQLAlchemy 使用情况的性能分析套件。

每个套件都专注于特定的用例，具有特定的性能配置文件和相关影响：

+   批量插入

+   单个插入，有或没有事务

+   获取大量行

+   运行大量的短查询

所有套件包括一系列使用模式，既展示了核心使用，也展示了 ORM 使用，并且通常按性能从最差到最好的顺序排序，根据 SQLAlchemy 提供的功能量的多少相反排序，从最多到最少（这两件事通常完美对应）。

一个命令行工具在包级别被呈现，允许单独的套件运行：

```py
$ python -m examples.performance --help
usage: python -m examples.performance [-h] [--test TEST] [--dburl DBURL]
                                      [--num NUM] [--profile] [--dump]
                                      [--echo]

                                      {bulk_inserts,large_resultsets,single_inserts}

positional arguments:
  {bulk_inserts,large_resultsets,single_inserts}
                        suite to run

optional arguments:
  -h, --help            show this help message and exit
  --test TEST           run specific test name
  --dburl DBURL         database URL, default sqlite:///profile.db
  --num NUM             Number of iterations/items/etc for tests;
                        default is module-specific
  --profile             run profiling and dump call counts
  --dump                dump full call profile (implies --profile)
  --echo                Echo SQL output
```

一个示例运行如下：

```py
$ python -m examples.performance bulk_inserts
```

或者使用选项：

```py
$ python -m examples.performance bulk_inserts \
    --dburl mysql+mysqldb://scott:tiger@localhost/test \
    --profile --num 1000
```

另请参阅

如何对 SQLAlchemy 驱动的应用程序进行性能分析？

#### 文件列表

文件列表：

+   bulk_updates.py - 这一系列的测试将说明不同的方法来批量更新大量行（正在建设中！目前只有一个测试）

+   large_resultsets.py - 在这一系列的测试中，我们关注的是加载大量非常小而简单的行的时间。

+   bulk_inserts.py - 这一系列的测试说明了不同的方法来批量插入大量行。

+   short_selects.py - 这一系列的测试说明了通过主键选择单个记录的不同方法

+   single_inserts.py - 在这一系列的测试中，我们关注的是一种在独立事务内插入一行数据的方法，然后返回到基本上“关闭”的状态。这类似于启动数据库连接的 API 调用，插入行，提交并关闭。

+   __main__.py - 允许将 examples/performance 包作为脚本运行。

#### 使用时间运行所有测试

这是默认形式的运行：

```py
$ python -m examples.performance single_inserts
Tests to run: test_orm_commit, test_bulk_save,
              test_bulk_insert_dictionaries, test_core,
              test_core_query_caching, test_dbapi_raw_w_connect,
              test_dbapi_raw_w_pool

test_orm_commit : Individual INSERT/COMMIT pairs via the
    ORM (10000 iterations); total time 13.690218 sec
test_bulk_save : Individual INSERT/COMMIT pairs using
    the "bulk" API  (10000 iterations); total time 11.290371 sec
test_bulk_insert_dictionaries : Individual INSERT/COMMIT pairs using
    the "bulk" API with dictionaries (10000 iterations);
    total time 10.814626 sec
test_core : Individual INSERT/COMMIT pairs using Core.
    (10000 iterations); total time 9.665620 sec
test_core_query_caching : Individual INSERT/COMMIT pairs using Core
    with query caching (10000 iterations); total time 9.209010 sec
test_dbapi_raw_w_connect : Individual INSERT/COMMIT pairs w/ DBAPI +
    connection each time (10000 iterations); total time 9.551103 sec
test_dbapi_raw_w_pool : Individual INSERT/COMMIT pairs w/ DBAPI +
    connection pool (10000 iterations); total time 8.001813 sec
```

#### 为单个测试生成配置文件

可以为所有测试或更常见的是单个测试生成 Python 配置文件输出：

```py
$ python -m examples.performance single_inserts --test test_core --num 1000 --dump
Tests to run: test_core
test_core : Individual INSERT/COMMIT pairs using Core. (1000 iterations); total fn calls 186109
         186109 function calls (186102 primitive calls) in 1.089 seconds

   Ordered by: internal time, call count

   ncalls  tottime  percall  cumtime  percall filename:lineno(function)
     1000    0.634    0.001    0.634    0.001 {method 'commit' of 'sqlite3.Connection' objects}
     1000    0.154    0.000    0.154    0.000 {method 'execute' of 'sqlite3.Cursor' objects}
     1000    0.021    0.000    0.074    0.000 /Users/classic/dev/sqlalchemy/lib/sqlalchemy/sql/compiler.py:1950(_get_colparams)
     1000    0.015    0.000    0.034    0.000 /Users/classic/dev/sqlalchemy/lib/sqlalchemy/engine/default.py:503(_init_compiled)
        1    0.012    0.012    1.091    1.091 examples/performance/single_inserts.py:79(test_core)

    ...
```

#### 编写您自己的套件

分析套件系统是可扩展的，并且可以应用于您自己的一组测试。这是一个有价值的技术，可用于决定一些性能关键的程序集的正确方法。例如，如果我们想要分析几种加载之间的差异，我们可以创建一个名为`test_loads.py`的文件，内容如下：

```py
from examples.performance import Profiler
from sqlalchemy import Integer, Column, create_engine, ForeignKey
from sqlalchemy.orm import relationship, joinedload, subqueryload, Session
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()
engine = None
session = None

class Parent(Base):
    __tablename__ = 'parent'
    id = Column(Integer, primary_key=True)
    children = relationship("Child")

class Child(Base):
    __tablename__ = 'child'
    id = Column(Integer, primary_key=True)
    parent_id = Column(Integer, ForeignKey('parent.id'))

# Init with name of file, default number of items
Profiler.init("test_loads", 1000)

@Profiler.setup_once
def setup_once(dburl, echo, num):
    "setup once.  create an engine, insert fixture data"
    global engine
    engine = create_engine(dburl, echo=echo)
    Base.metadata.drop_all(engine)
    Base.metadata.create_all(engine)
    sess = Session(engine)
    sess.add_all([
        Parent(children=[Child() for j in range(100)])
        for i in range(num)
    ])
    sess.commit()

@Profiler.setup
def setup(dburl, echo, num):
    "setup per test.  create a new Session."
    global session
    session = Session(engine)
    # pre-connect so this part isn't profiled (if we choose)
    session.connection()

@Profiler.profile
def test_lazyload(n):
    "load everything, no eager loading."

    for parent in session.query(Parent):
        parent.children

@Profiler.profile
def test_joinedload(n):
    "load everything, joined eager loading."

    for parent in session.query(Parent).options(joinedload("children")):
        parent.children

@Profiler.profile
def test_subqueryload(n):
    "load everything, subquery eager loading."

    for parent in session.query(Parent).options(subqueryload("children")):
        parent.children

if __name__ == '__main__':
    Profiler.main()
```

我们可以直接运行我们的新脚本：

```py
$ python test_loads.py  --dburl postgresql+psycopg2://scott:tiger@localhost/test
Running setup once...
Tests to run: test_lazyload, test_joinedload, test_subqueryload
test_lazyload : load everything, no eager loading. (1000 iterations); total time 11.971159 sec
test_joinedload : load everything, joined eager loading. (1000 iterations); total time 2.754592 sec
test_subqueryload : load everything, subquery eager loading. (1000 iterations); total time 2.977696 sec
```  ### 太空入侵者

使用 SQLite 作为状态机的太空入侵者游戏。

最初于 2012 年开发。适应在 Python 3 中工作。

在文本控制台中使用 ASCII 艺术运行。

![../_images/space_invaders.jpg](img/df5d8744e7ec946672bdde78d1db980c.png)

要运行：

```py
python -m examples.space_invaders.space_invaders
```

在运行时，观察日志中的 SQL 输出：

```py
tail -f space_invaders.log
```

祝您愉快！

文件列表：

+   space_invaders.py  ### 对象版本控制

#### 使用历史表进行版本控制

说明了一个扩展，它为实体创建版本表并为每个更改存储记录。给定的扩展生成一个匿名的“history”类，表示目标对象的历史版本。

与使用时间行进行版本控制示例相比，该示例将更新写入为同一表中的新行，而不使用单独的历史表。

通过一个名为`test_versioning.py`的单元测试模块演示了用法，该模块可以像其他模块一样运行，内部使用`unittest`：

```py
python -m examples.versioned_history.test_versioning
```

示例用法的片段，使用声明性：

```py
from history_meta import Versioned, versioned_session

class Base(DeclarativeBase):
    pass

class SomeClass(Versioned, Base):
    __tablename__ = 'sometable'

    id = Column(Integer, primary_key=True)
    name = Column(String(50))

    def __eq__(self, other):
        assert type(other) is SomeClass and other.id == self.id

Session = sessionmaker(bind=engine)
versioned_session(Session)

sess = Session()
sc = SomeClass(name='sc1')
sess.add(sc)
sess.commit()

sc.name = 'sc1modified'
sess.commit()

assert sc.version == 2

SomeClassHistory = SomeClass.__history_mapper__.class_

assert sess.query(SomeClassHistory).\
            filter(SomeClassHistory.version == 1).\
            all() \
            == [SomeClassHistory(version=1, name='sc1')]
```

`Versioned`混合类设计用于与声明性结合使用。要将该扩展与经典映射器一起使用，可以应用`_history_mapper`函数：

```py
from history_meta import _history_mapper

m = mapper(SomeClass, sometable)
_history_mapper(m)

SomeHistoryClass = SomeClass.__history_mapper__.class_
```

版本控制示例还与 ORM 乐观并发特性集成，文档化在配置版本计数器中。要启用此功能，请将标志`Versioned.use_mapper_versioning`设置为 True：

```py
class SomeClass(Versioned, Base):
    __tablename__ = 'sometable'

    use_mapper_versioning = True

    id = Column(Integer, primary_key=True)
    name = Column(String(50))

    def __eq__(self, other):
        assert type(other) is SomeClass and other.id == self.id
```

如果两个具有相同版本标识符的`SomeClass`实例被同时更新并发送到数据库进行并发 UPDATE，如果数据库隔离级别允许两个 UPDATE 语句继续，其中一个将失败，因为它不再针对最后已知的版本标识符。

文件列表：

+   test_versioning.py - 展示`history_meta.py`模块函数用法的单元测试。

+   history_meta.py - 版本控制混合类和其他实用程序。  #### 使用时间行进行版本控制

几个示例说明了拦截首先被解释为对行的 UPDATE 的更改的技术，并将其转换为对新行的 INSERT，将先前的行保留为历史版本。

与带有历史表的版本化示例进行比较，该示例将历史行写入单独的历史表。

文件列表：

+   versioned_rows.py - 展示了拦截对象更改的方法，将对单个行的 UPDATE 语句转换为 INSERT 语句，以便插入具有新数据的新行，保持旧行不变。

+   versioned_rows_w_versionid.py - 展示了拦截对象更改的方法，将对单个行的 UPDATE 语句转换为 INSERT 语句，以便插入具有新数据的新行，保持旧行不变。

+   versioned_map.py - 围绕“垂直表”结构的概念构建的 versioned_rows 示例的变体，类似于垂直属性映射示例中所示的那些。

+   versioned_update_old_row.py - 展示了`versioned_rows.py`中相同的 UPDATE 转换为 INSERT 技术，但还发出了一个 UPDATE 命令来影响**旧**行的时间戳更改。 还包括一个`SessionEvents.do_orm_execute()`钩子，以限制查询仅针对最新版本。  ### 垂直属性映射

展示了“垂直表”映射。

“垂直表”是指将对象的各个属性存储为表中的不同行的技术。 “垂直表”技术用于持久化可以具有各种属性集的对象，但牺牲了简单的查询控制和简洁性。 它通常在内容/文档管理系统中找到，以灵活地表示用户创建的结构。

给出了两种方法。在第二种方法中，每行引用一个“数据类型”，其中包含有关属性中存储的信息类型的信息，例如整数、字符串或日期。

例子：

```py
shrew = Animal(u'shrew')
shrew[u'cuteness'] = 5
shrew[u'weasel-like'] = False
shrew[u'poisonous'] = True

session.add(shrew)
session.flush()

q = (session.query(Animal).
     filter(Animal.facts.any(
       and_(AnimalFact.key == u'weasel-like',
            AnimalFact.value == True))))
print('weasel-like animals', q.all())
```

文件列表：

+   dictlike-polymorphic.py - 将多态值的垂直表映射为字典。

+   dictlike.py - 将垂直表映射为字典。  ## 继承映射配方

### 基本继承映射

单表、联接表和具体表继承的工作示例，如映射类继承层次结构中所述。

文件列表：

+   joined.py - 联接表（每个子类一个表）继承示例。

+   concrete.py - 具体表（基于类的表）继承示例。

+   single.py - 单表（基于层次结构的表）继承示例。

## 特殊 API

### 属性仪器化

示例说明了对 SQLAlchemy 属性管理系统的修改。

文件列表：

+   listen_for_events.py - 展示了如何将事件附加到所有被检测的属性，并监听更改事件。

+   active_column_defaults.py - 说明了如何使用`AttributeEvents.init_scalar()`事件，配合核心列默认值，以提供 ORM 对象，当访问未设置的属性时自动产生默认值。

+   custom_management.py - 说明了如何使用`sqlalchemy.ext.instrumentation`扩展包自定义类仪器化。### 水平分片

SQLAlchemy 分片 API 的基本示例。分片是指在多个数据库之间水平扩展数据。

“分片”映射的基本组件包括：

+   多个`Engine`实例，每个都分配了一个“分片 id”。这些`Engine`实例可能引用不同的数据库，或者同一数据库中的不同模式/帐户，或者它们甚至可以仅通过会导致它们在使用时访问不同模式或表的选项进行区分。

+   一个函数，它可以根据要保存的实例返回单个分片 id；这称为“shard_chooser”。

+   一个可以返回适用于特定实例标识符的分片 id 列表的函数；这称为“id_chooser”。如果返回所有分片 id，则将搜索所有分片。

+   一个函数，它可以根据特定查询返回要尝试的分片 id 列表（“query_chooser”）。如果返回所有分片 id，则将查询所有分片并将结果连接在一起。

在这些示例中，使用不同类型的分片对相同的基本示例进行操作，该示例根据每个大陆的天气数据进行处理。我们提供了示例的 shard_chooser、id_chooser 和 query_chooser 函数。query_chooser 说明了检查 SQL 表达式元素以尝试确定所请求的单个分片。

创建通用分片例程是组织多个数据库实例的问题的一种雄心勃勃的方法。对于更简明的替代方案，“不同实体”方法是一种以明确方式将对象分配给不同表（以及可能的数据库节点）的简单方法 - 在[EntityName](https://www.sqlalchemy.org/trac/wiki/UsageRecipes/EntityName)的维基页面中有描述。

文件列表：

+   separate_databases.py - 演示了使用不同的 SQLite 数据库进行分片。

+   separate_tables.py - 演示了使用单个 SQLite 数据库进行分片，但是会使用命名约定来创建多个表。

+   separate_schema_translates.py - 演示了在使用具有多个模式的单个数据库进行分片时，可以为每个分片使用不同的“schema_translates_map”。

+   asyncio.py - 演示了与 asyncio 一起使用的分片 API。

## 扩展 ORM

### ORM 查询事件

演示了如何使用 [dogpile.cache](https://dogpilecache.sqlalchemy.org/) 功能嵌入 ORM 查询，允许完全控制缓存以及从长期缓存中提取“延迟加载”的属性。

示例包括演示`with_loader_criteria()` 选项以及`SessionEvents.do_orm_execute()` 钩子的用法。

从 SQLAlchemy 1.4 开始，`Query` 构造与 `Select` 构造合并在一起，因此这两个对象基本上是相同的。

文件列表：

+   temporal_range.py - 演示了将应用于选定实体的自定义每个查询条件。

+   ### Dogpile 缓存

filter_public.py - 演示了应用于特定类型实体的全局条件。

在这个演示中，以下技术被说明：

+   使用 `SessionEvents.do_orm_execute()` 事件挂钩

+   绕过 `Session.execute()` 的基本技术，从自定义缓存源中获取数据，而不是从数据库中获取。

+   利用 dogpile.cache 进行基本缓存，使用“区域”允许对固定配置集合进行全局控制。

+   使用自定义的 `UserDefinedOption` 对象配置语句对象中的选项。

另请参阅

重新执行语句 - 包含此处提出的技术的一般示例。

例如：

```py
# query for Person objects, specifying cache
stmt = select(Person).options(FromCache("default"))

# specify that each Person's "addresses" collection comes from
# cache too
stmt = stmt.options(RelationshipCache(Person.addresses, "default"))

# execute and results
result = session.execute(stmt)

print(result.scalars().all())
```

要运行，必须安装 SQLAlchemy 和 dogpile.cache，或者将它们安装到当前的 PYTHONPATH。演示将创建一个本地目录用于数据文件，插入初始数据，然后运行。第二次运行演示将利用已经存在的缓存文件，并且仅会发出一条 SQL 语句来查询两个表 - 但是显示的结果将利用数十个懒加载，所有懒加载都从缓存中获取。

演示脚本本身按复杂性顺序作为 Python 模块运行，以便相对导入起作用。

```py
python -m examples.dogpile_caching.helloworld

python -m examples.dogpile_caching.relationship_caching

python -m examples.dogpile_caching.advanced

python -m examples.dogpile_caching.local_session_caching
```

文件列表：

+   environment.py - 确定数据/缓存文件路径和配置，如果需要，引导装置数据。

+   caching_query.py - 表示允许使用 SQLAlchemy 进行 Dogpile 缓存的函数和类。引入了一个称为 FromCache 的查询选项。

+   model.py - 数据模型，表示具有多个 Address 对象的 Person，每个对象都有 PostalCode、City、Country。

+   fixture_data.py - 安装一些示例数据。这里有一些美国/加拿大几个城市的少量邮政编码。然后，安装 100 个 Person 记录，每个记录都有一个随机选择的邮政编码。

+   helloworld.py - 演示如何加载一些数据，并缓存结果。

+   relationship_caching.py - 演示如何在关联端点上添加缓存选项，以便懒加载从缓存中加载。

+   advanced.py - 演示如何使用 Query 结合 FromCache 选项，包括前端加载、缓存失效和集合缓存。

+   local_session_caching.py - 此示例创建了一个新的 dogpile.cache 后端，将数据持久化在当前会话的字典中。移除会话后，缓存消失。

## 映射配方

### 邻接表

使用邻接表模型映射的字典-字典结构的示例。

例如：

```py
node = TreeNode('rootnode')
node.append('node1')
node.append('node3')
session.add(node)
session.commit()

dump_tree(node)
```

文件列表：

+   adjacency_list.py  ### 关联

描述了“关联对象”模式的使用示例，其中一个中介类在两个以多对多模式关联的类之间进行关系中介。

文件清单：

+   proxied_association.py - 与 basic_association 相同的示例，添加了对 `sqlalchemy.ext.associationproxy` 的使用，以使对 `OrderItem` 的显式引用是可选的。

+   basic_association.py - 演示了“订单”和一组“商品”对象之间的多对多关系，通过名为“OrderItem”的关联对象为每个关联购买价格。

+   dict_of_sets_with_default.py - 一个高级关联代理示例，演示了关联代理的嵌套以生成多级 Python 集合，本例中是一个具有字符串键和整数集合值的字典，它隐藏了底层映射类。  ### Asyncio 集成

描述了 SQLAlchemy 的 asyncio 引擎功能的示例。

文件清单：

+   async_orm.py - 演示了使用 `sqlalchemy.ext.asyncio.AsyncSession` 对象进行异步 ORM 使用。

+   async_orm_writeonly.py - 展示了使用**只写关系**来更简单地处理 asyncio 下的 ORM 集合。

+   gather_orm_statements.py - 演示了如何使用 `asyncio.gather()` 在许多 asyncio 数据库连接上同时运行许多语句，将 ORM 结果合并为单个 `AsyncSession`。

+   basic.py - 展示了 asyncio 引擎/连接接口。

+   greenlet_orm.py - 展示了使用 sqlalchemy.ext.asyncio.AsyncSession 对象进行异步 ORM 使用的示例，包括可选的 run_sync() 方法。  ### 有向图

有向图结构的持久性示例。 图以一组边的形式存储，每个边都引用节点表中的“下限”和“上限”节点。 演示了基本的持久性和查询“下限”和“上限”邻居的方法：

```py
n2 = Node(2)
n5 = Node(5)
n2.add_neighbor(n5)
print(n2.higher_neighbors())
```

文件清单：

+   directed_graph.py  ### 动态关系作为字典

展示了如何在“动态”关系之上放置类似字典的外观，以便字典操作（假设简单字符串键）可以在一次加载完整集合的情况下操作大型集合。

文件清单：

+   dynamic_dict.py  ### 通用关联

展示了将多种类型的父对象与特定子对象关联的各种方法。

所有示例都使用了声明性扩展和声明性混合。每个示例最终呈现相同的用例 - 两个类，`Customer`和`Supplier`，都是`HasAddresses`混合类的子类，该混合类确保父类提供一个包含`Address`对象的`addresses`集合。

discriminator_on_association.py 和 generic_fk.py 脚本是 2007 年博客文章[使用 SQLAlchemy 进行多态关联](https://techspot.zzzeek.org/2007/05/29/polymorphic-associations-with-sqlalchemy/)中提出的配方的现代化版本。

文件列表：

+   table_per_association.py - 展示了一个通过为每个父类单独生成关联表来提供通用关联的混合类。关联的对象本身存储在所有父类共享的单个表中。

+   table_per_related.py - 展示了一种通用关联，通过为每个父类生成单独的关联表来持久化关联对象，每个关联表都是为了代表特定父类而生成的。

+   discriminator_on_association.py - 展示了一个提供通用关联的混合类，使用单个目标表和单个关联表，所有父表都引用该关联表。关联表包含一个“鉴别器”列，用于确定每个关联表中的行与哪种类型的父对象相关联。

+   generic_fk.py - 展示了所谓的“通用外键”，类似于流行框架（如 Django、ROR 等）的做法。这种方法绕过了标准的参照完整性实践，因为“外键”列实际上并没有约束到任何特定的表；相反，应用程序逻辑用于确定引用的是哪个表。  ### 材料化路径

展示了使用 SQLAlchemy ORM 实现“材料化路径”模式的方法。

文件列表：

+   materialized_paths.py - 展示了“材料化路径”模式。  ### 嵌套集

展示了使用 SQLAlchemy ORM 实现“嵌套集”模式的基本方法。

文件列表：

+   nested_sets.py - Celko 的“嵌套集”树结构。  ### 性能

用于各种 SQLAlchemy 用例的性能分析套件。

每个套件专注于具有特定性能配置文件和相关影响的特定用例：

+   批量插入

+   单个插入，有或者没有事务

+   获取大量行

+   运行大量短查询

所有套件都包括一系列使用模式，说明了核心和 ORM 使用，并且通常按性能从最差到最佳的顺序排序，基于 SQLAlchemy 提供的功能数量，从最大到最小（这两个方面通常完美对应）。

一个命令行工具在包级别被呈现，它允许运行个别套件：

```py
$ python -m examples.performance --help
usage: python -m examples.performance [-h] [--test TEST] [--dburl DBURL]
                                      [--num NUM] [--profile] [--dump]
                                      [--echo]

                                      {bulk_inserts,large_resultsets,single_inserts}

positional arguments:
  {bulk_inserts,large_resultsets,single_inserts}
                        suite to run

optional arguments:
  -h, --help            show this help message and exit
  --test TEST           run specific test name
  --dburl DBURL         database URL, default sqlite:///profile.db
  --num NUM             Number of iterations/items/etc for tests;
                        default is module-specific
  --profile             run profiling and dump call counts
  --dump                dump full call profile (implies --profile)
  --echo                Echo SQL output
```

示例运行如下：

```py
$ python -m examples.performance bulk_inserts
```

或使用选项：

```py
$ python -m examples.performance bulk_inserts \
    --dburl mysql+mysqldb://scott:tiger@localhost/test \
    --profile --num 1000
```

另请参阅

我如何分析使用 SQLAlchemy 的应用程序？

#### 文件列表

文件列表：

+   bulk_updates.py - 这一系列的测试将演示不同的方式来批量更新大量行（正在建设中！目前只有一个测试）

+   large_resultsets.py - 在这一系列的测试中，我们正在研究加载大量非常小而简单行所需的时间。

+   bulk_inserts.py - 这一系列的测试演示了不同的方式来批量插入大量行。

+   short_selects.py - 这一系列的测试演示了不同的方式来通过主键选择单个记录

+   single_inserts.py - 在这一系列的测试中，我们正在研究一种在独立事务中插入一行数据的方法，然后返回到基本上是“关闭”的状态。这类似于一个启动数据库连接、插入行、提交并关闭的 API 调用。

+   __main__.py - 允许 examples/performance 包被当作脚本运行。

#### 运行所有测试并计时

这是默认的运行形式：

```py
$ python -m examples.performance single_inserts
Tests to run: test_orm_commit, test_bulk_save,
              test_bulk_insert_dictionaries, test_core,
              test_core_query_caching, test_dbapi_raw_w_connect,
              test_dbapi_raw_w_pool

test_orm_commit : Individual INSERT/COMMIT pairs via the
    ORM (10000 iterations); total time 13.690218 sec
test_bulk_save : Individual INSERT/COMMIT pairs using
    the "bulk" API  (10000 iterations); total time 11.290371 sec
test_bulk_insert_dictionaries : Individual INSERT/COMMIT pairs using
    the "bulk" API with dictionaries (10000 iterations);
    total time 10.814626 sec
test_core : Individual INSERT/COMMIT pairs using Core.
    (10000 iterations); total time 9.665620 sec
test_core_query_caching : Individual INSERT/COMMIT pairs using Core
    with query caching (10000 iterations); total time 9.209010 sec
test_dbapi_raw_w_connect : Individual INSERT/COMMIT pairs w/ DBAPI +
    connection each time (10000 iterations); total time 9.551103 sec
test_dbapi_raw_w_pool : Individual INSERT/COMMIT pairs w/ DBAPI +
    connection pool (10000 iterations); total time 8.001813 sec
```

#### 为个别测试转储配置文件

Python 分析配置文件可以为所有测试或更常见的是为个别测试转储：

```py
$ python -m examples.performance single_inserts --test test_core --num 1000 --dump
Tests to run: test_core
test_core : Individual INSERT/COMMIT pairs using Core. (1000 iterations); total fn calls 186109
         186109 function calls (186102 primitive calls) in 1.089 seconds

   Ordered by: internal time, call count

   ncalls  tottime  percall  cumtime  percall filename:lineno(function)
     1000    0.634    0.001    0.634    0.001 {method 'commit' of 'sqlite3.Connection' objects}
     1000    0.154    0.000    0.154    0.000 {method 'execute' of 'sqlite3.Cursor' objects}
     1000    0.021    0.000    0.074    0.000 /Users/classic/dev/sqlalchemy/lib/sqlalchemy/sql/compiler.py:1950(_get_colparams)
     1000    0.015    0.000    0.034    0.000 /Users/classic/dev/sqlalchemy/lib/sqlalchemy/engine/default.py:503(_init_compiled)
        1    0.012    0.012    1.091    1.091 examples/performance/single_inserts.py:79(test_core)

    ...
```

#### 编写您自己的套件

分析套件系统是可扩展的，并且可以应用于您自己的一组测试。这是一个在决定某些性能关键程度的一组例程的正确方法时使用的宝贵技术。例如，如果我们想要分析几种加载之间的差异，我们可以创建一个名为`test_loads.py`的文件，内容如下：

```py
from examples.performance import Profiler
from sqlalchemy import Integer, Column, create_engine, ForeignKey
from sqlalchemy.orm import relationship, joinedload, subqueryload, Session
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()
engine = None
session = None

class Parent(Base):
    __tablename__ = 'parent'
    id = Column(Integer, primary_key=True)
    children = relationship("Child")

class Child(Base):
    __tablename__ = 'child'
    id = Column(Integer, primary_key=True)
    parent_id = Column(Integer, ForeignKey('parent.id'))

# Init with name of file, default number of items
Profiler.init("test_loads", 1000)

@Profiler.setup_once
def setup_once(dburl, echo, num):
    "setup once.  create an engine, insert fixture data"
    global engine
    engine = create_engine(dburl, echo=echo)
    Base.metadata.drop_all(engine)
    Base.metadata.create_all(engine)
    sess = Session(engine)
    sess.add_all([
        Parent(children=[Child() for j in range(100)])
        for i in range(num)
    ])
    sess.commit()

@Profiler.setup
def setup(dburl, echo, num):
    "setup per test.  create a new Session."
    global session
    session = Session(engine)
    # pre-connect so this part isn't profiled (if we choose)
    session.connection()

@Profiler.profile
def test_lazyload(n):
    "load everything, no eager loading."

    for parent in session.query(Parent):
        parent.children

@Profiler.profile
def test_joinedload(n):
    "load everything, joined eager loading."

    for parent in session.query(Parent).options(joinedload("children")):
        parent.children

@Profiler.profile
def test_subqueryload(n):
    "load everything, subquery eager loading."

    for parent in session.query(Parent).options(subqueryload("children")):
        parent.children

if __name__ == '__main__':
    Profiler.main()
```

我们可以直接运行我们的新脚本：

```py
$ python test_loads.py  --dburl postgresql+psycopg2://scott:tiger@localhost/test
Running setup once...
Tests to run: test_lazyload, test_joinedload, test_subqueryload
test_lazyload : load everything, no eager loading. (1000 iterations); total time 11.971159 sec
test_joinedload : load everything, joined eager loading. (1000 iterations); total time 2.754592 sec
test_subqueryload : load everything, subquery eager loading. (1000 iterations); total time 2.977696 sec
```  ### 太空入侵

使用 SQLite 作为状态机的太空入侵游戏。

最初于 2012 年开发。适应 Python 3 中运行。

使用 ASCII 艺术在文本控制台中运行。

![../_images/space_invaders.jpg](img/df5d8744e7ec946672bdde78d1db980c.png)

要运行：

```py
python -m examples.space_invaders.space_invaders
```

在运行时，请观察日志中的 SQL 输出：

```py
tail -f space_invaders.log
```

祝您愉快！

文件列表：

+   space_invaders.py  ### 对象版本控制

#### 使用历史表进行版本控制

演示了一个扩展，它为实体创建版本表并为每个更改存储记录。给定的扩展生成一个匿名的“历史”类，该类表示目标对象的历史版本。

与在相同表中将更新写为新行的使用时间行进行版本控制示例进行比较，而不使用单独的历史表。

通过一个单元测试模块 `test_versioning.py` 展示了用法，可以像运行任何其他模块一样运行，内部使用 `unittest`：

```py
python -m examples.versioned_history.test_versioning
```

一个使用声明性的示例用法片段：

```py
from history_meta import Versioned, versioned_session

class Base(DeclarativeBase):
    pass

class SomeClass(Versioned, Base):
    __tablename__ = 'sometable'

    id = Column(Integer, primary_key=True)
    name = Column(String(50))

    def __eq__(self, other):
        assert type(other) is SomeClass and other.id == self.id

Session = sessionmaker(bind=engine)
versioned_session(Session)

sess = Session()
sc = SomeClass(name='sc1')
sess.add(sc)
sess.commit()

sc.name = 'sc1modified'
sess.commit()

assert sc.version == 2

SomeClassHistory = SomeClass.__history_mapper__.class_

assert sess.query(SomeClassHistory).\
            filter(SomeClassHistory.version == 1).\
            all() \
            == [SomeClassHistory(version=1, name='sc1')]
```

`Versioned` 混合类设计用于与声明性一起使用。要将扩展与经典映射器一起使用，可以应用 `_history_mapper` 函数：

```py
from history_meta import _history_mapper

m = mapper(SomeClass, sometable)
_history_mapper(m)

SomeHistoryClass = SomeClass.__history_mapper__.class_
```

版本控制示例还与文档中记录的 ORM 乐观并发特性集成在一起 配置版本计数器。要启用此功能，请将标志 `Versioned.use_mapper_versioning` 设置为 True：

```py
class SomeClass(Versioned, Base):
    __tablename__ = 'sometable'

    use_mapper_versioning = True

    id = Column(Integer, primary_key=True)
    name = Column(String(50))

    def __eq__(self, other):
        assert type(other) is SomeClass and other.id == self.id
```

如果两个具有相同版本标识符的 `SomeClass` 实例被同时更新并发送到数据库以进行并发 UPDATE，如果数据库隔离级别允许两个 UPDATE 语句继续进行，则其中一个将失败，因为它不再针对最后已知的版本标识符。

文件列表：

+   test_versioning.py - 演示 `history_meta.py` 模块函数的用法的单元测试。

+   history_meta.py - 版本混合类和其他实用程序。  #### 使用时间行进行版本控制

有几个示例说明了拦截更改的技术，这些更改首先被解释为对行的 UPDATE，而实际上将其转换为对新行的 INSERT，使以前的行保持不变作为历史版本。

与将历史行写入单独的历史表的使用历史表进行版本控制示例进行比较。

文件列表：

+   versioned_rows.py - 演示拦截对象更改的方法，将单行的 UPDATE 语句转换为 INSERT 语句，以便插入具有新数据的新行，保持旧行不变。

+   versioned_rows_w_versionid.py - 演示拦截对象更改的方法，将单行的 UPDATE 语句转换为 INSERT 语句，以便插入具有新数据的新行，保持旧行不变。

+   versioned_map.py - 围绕“垂直表”结构的概念构建的 versioned_rows 示例的变体，类似于 垂直属性映射 示例中所示的那些。

+   versioned_update_old_row.py - 说明了`versioned_rows.py`中相同的 UPDATE 到 INSERT 技术，但也发出了对**旧**行的 UPDATE 以影响时间戳的更改。还包括一个`SessionEvents.do_orm_execute()`钩子，以限制查询仅限于最新版本。  ### 竖直属性映射

说明了“竖直表”映射。

“竖直表”是指一种技术，其中对象的各个属性被存储为表中的不同行。使用“竖直表”技术来持久化可以具有不同属性集的对象，但会牺牲简单的查询控制和简洁性。在内容/文档管理系统中通常可以灵活表示用户创建的结构。

给出了两种方法的变体。在第二种方法中，每行引用一个“数据类型”，其中包含关于属性中存储的信息类型的信息，例如整数、字符串或日期。

示例：

```py
shrew = Animal(u'shrew')
shrew[u'cuteness'] = 5
shrew[u'weasel-like'] = False
shrew[u'poisonous'] = True

session.add(shrew)
session.flush()

q = (session.query(Animal).
     filter(Animal.facts.any(
       and_(AnimalFact.key == u'weasel-like',
            AnimalFact.value == True))))
print('weasel-like animals', q.all())
```

文件清单：

+   dictlike-polymorphic.py - 将具有多态值的竖直表映射为字典。

+   dictlike.py - 将竖直表映射为字典的示例。  ### 邻接表

以邻接表模型映射的字典嵌套结构的示例。

例如：

```py
node = TreeNode('rootnode')
node.append('node1')
node.append('node3')
session.add(node)
session.commit()

dump_tree(node)
```

文件清单：

+   adjacency_list.py

### 关联

示例说明了“关联对象”模式的使用，其中一个中间类介于两个以多对多模式关联的类之间。

文件清单：

+   proxied_association.py - 与 basic_association 相同的示例，同时添加了对`sqlalchemy.ext.associationproxy`的使用，以使对`OrderItem`的显式引用成为可选。

+   basic_association.py - 说明了“订单”和“项目”对象集合之间的多对多关系，通过称为“OrderItem”的关联对象将每个订单价格关联起来。

+   dict_of_sets_with_default.py - 一个高级关联代理示例，说明了关联代理的嵌套以生成多级 Python 集合，本例中是一个具有字符串键和整数集合作为值的字典，该字典隐藏了底层的映射类。

### Asyncio 集成

示例说明了 SQLAlchemy 的 asyncio 引擎功能。

文件清单：

+   async_orm.py - 演示了使用`sqlalchemy.ext.asyncio.AsyncSession`对象进行异步 ORM 使用。

+   async_orm_writeonly.py - 演示了在 asyncio 下使用**只写关系**来更简单地处理 ORM 集合。

+   gather_orm_statements.py - 演示了如何使用`asyncio.gather()`在许多 asyncio 数据库连接上并发运行多个语句，将 ORM 结果合并到单个`AsyncSession`中。

+   basic.py - 演示了 asyncio 引擎/连接接口。

+   greenlet_orm.py - 演示了使用 sqlalchemy.ext.asyncio.AsyncSession 对象进行异步 ORM 使用，包括可选的 run_sync()方法。

### 有向图

一个有向图结构的持久性示例。图被存储为一组边，每条边都引用节点表中的“较低”和“较高”节点。演示了基本的持久性和查询较低和较高邻居的方法：

```py
n2 = Node(2)
n5 = Node(5)
n2.add_neighbor(n5)
print(n2.higher_neighbors())
```

文件清单：

+   directed_graph.py

### 动态关系作为字典

演示了如何在“动态”关系之上放置类似于字典的外观，以便字典操作（假设简单的字符串键）可以在大型集合上进行操作，而无需一次加载整个集合。

文件清单：

+   dynamic_dict.py

### 通用关联

演示了将多种类型的父类与特定子对象关联的各种方法。

所有示例都使用声明性扩展以及声明性 mixin。每个示例最后都呈现相同的用例 - 两个类，`Customer`和`Supplier`，都是`HasAddresses` mixin 的子类，该 mixin 确保父类提供了一个包含`Address`对象的`addresses`集合。

discriminator_on_association.py 和 generic_fk.py 脚本是 2007 年博客文章[使用 SQLAlchemy 进行多态关联](https://techspot.zzzeek.org/2007/05/29/polymorphic-associations-with-sqlalchemy/)中提出的配方的现代化版本。

文件清单：

+   table_per_association.py - 通过为每个父类单独生成关联表格来提供通用关联的 mixin 示例。关联对象本身存储在所有父类之间共享的单个表中。

+   table_per_related.py - 演示了一种通用关联，它在各个表中持久化关联对象，每个表都生成来代表特定父类持久化这些对象。

+   discriminator_on_association.py - 演示了一种提供通用关联的 mixin，该关联使用单个目标表和单个关联表，所有父表都引用它。关联表包含一个“区分符”列，用于确定哪种类型的父对象与关联表中的每个特定行关联。

+   generic_fk.py - 演示了所谓的“通用外键”，类似于流行框架（如 Django，ROR 等）的方式。这种方法绕过了标准的参照完整性实践，因为“外键”列实际上并不限制引用任何特定表；相反，应用程序逻辑用于确定引用的是哪个表。

### 材料化路径

演示了使用 SQLAlchemy ORM 实现“材料化路径”模式的方法。

文件清单：

+   materialized_paths.py - 演示了“材料化路径”模式。

### 嵌套集

演示了使用 SQLAlchemy ORM 实现“嵌套集”模式的一种基本方法。

文件清单：

+   nested_sets.py - Celko 的“嵌套集”树结构。

### 性能

用于各种 SQLAlchemy 用例的性能分析套件。

每个套件都专注于特定用例，具有特定的性能概况和相关含义：

+   批量插入

+   单独插入，有或没有事务

+   获取大量行

+   运行大量短查询

所有套件都包括各种使用模式，说明了 Core 和 ORM 的使用，并且通常按性能从最差到最佳的顺序排序，根据 SQLAlchemy 提供的功能量的多少，从最大到最小排列（这两件事通常完美对应）。

一个命令行工具在包级别呈现，允许运行各个套件：

```py
$ python -m examples.performance --help
usage: python -m examples.performance [-h] [--test TEST] [--dburl DBURL]
                                      [--num NUM] [--profile] [--dump]
                                      [--echo]

                                      {bulk_inserts,large_resultsets,single_inserts}

positional arguments:
  {bulk_inserts,large_resultsets,single_inserts}
                        suite to run

optional arguments:
  -h, --help            show this help message and exit
  --test TEST           run specific test name
  --dburl DBURL         database URL, default sqlite:///profile.db
  --num NUM             Number of iterations/items/etc for tests;
                        default is module-specific
  --profile             run profiling and dump call counts
  --dump                dump full call profile (implies --profile)
  --echo                Echo SQL output
```

一个示例运行如下：

```py
$ python -m examples.performance bulk_inserts
```

或者带有选项：

```py
$ python -m examples.performance bulk_inserts \
    --dburl mysql+mysqldb://scott:tiger@localhost/test \
    --profile --num 1000
```

另请参阅

我如何分析一个由 SQLAlchemy 驱动的应用程序？

#### 文件清单

文件清单：

+   bulk_updates.py - 这一系列测试将说明不同的方式来批量更新大量行（正在施工！目前只有一个测试）

+   large_resultsets.py - 在这系列测试中，我们研究加载大量非常小而简单的行所需的时间。

+   bulk_inserts.py - 这一系列的测试展示了批量插入大量行的不同方法。

+   short_selects.py - 这一系列的测试展示了通过主键选择单个记录的不同方法。

+   single_inserts.py - 在这一系列的测试中，我们看到了一种在不同事务中插入行并且之后返回到实质上“关闭”状态的方法。这类似于启动数据库连接的 API 调用，插入行，提交并关闭的过程。

+   __main__.py - 允许 examples/performance 包作为脚本运行。

#### 使用时间运行所有测试

这是运行的默认形式：

```py
$ python -m examples.performance single_inserts
Tests to run: test_orm_commit, test_bulk_save,
              test_bulk_insert_dictionaries, test_core,
              test_core_query_caching, test_dbapi_raw_w_connect,
              test_dbapi_raw_w_pool

test_orm_commit : Individual INSERT/COMMIT pairs via the
    ORM (10000 iterations); total time 13.690218 sec
test_bulk_save : Individual INSERT/COMMIT pairs using
    the "bulk" API  (10000 iterations); total time 11.290371 sec
test_bulk_insert_dictionaries : Individual INSERT/COMMIT pairs using
    the "bulk" API with dictionaries (10000 iterations);
    total time 10.814626 sec
test_core : Individual INSERT/COMMIT pairs using Core.
    (10000 iterations); total time 9.665620 sec
test_core_query_caching : Individual INSERT/COMMIT pairs using Core
    with query caching (10000 iterations); total time 9.209010 sec
test_dbapi_raw_w_connect : Individual INSERT/COMMIT pairs w/ DBAPI +
    connection each time (10000 iterations); total time 9.551103 sec
test_dbapi_raw_w_pool : Individual INSERT/COMMIT pairs w/ DBAPI +
    connection pool (10000 iterations); total time 8.001813 sec
```

#### 为个别测试输出简要概要

可以为所有测试或更常见的是个别测试转储 Python 分析输出：

```py
$ python -m examples.performance single_inserts --test test_core --num 1000 --dump
Tests to run: test_core
test_core : Individual INSERT/COMMIT pairs using Core. (1000 iterations); total fn calls 186109
         186109 function calls (186102 primitive calls) in 1.089 seconds

   Ordered by: internal time, call count

   ncalls  tottime  percall  cumtime  percall filename:lineno(function)
     1000    0.634    0.001    0.634    0.001 {method 'commit' of 'sqlite3.Connection' objects}
     1000    0.154    0.000    0.154    0.000 {method 'execute' of 'sqlite3.Cursor' objects}
     1000    0.021    0.000    0.074    0.000 /Users/classic/dev/sqlalchemy/lib/sqlalchemy/sql/compiler.py:1950(_get_colparams)
     1000    0.015    0.000    0.034    0.000 /Users/classic/dev/sqlalchemy/lib/sqlalchemy/engine/default.py:503(_init_compiled)
        1    0.012    0.012    1.091    1.091 examples/performance/single_inserts.py:79(test_core)

    ...
```

#### 编写你自己的测试套件

分析套件系统是可扩展的，并且可以应用于你自己的一组测试。这是一种在决定某些性能关键例程的正确方法时使用的有价值技术。例如，如果我们想要分析几种加载方式之间的差异，我们可以创建一个名为 `test_loads.py` 的文件，其内容如下：

```py
from examples.performance import Profiler
from sqlalchemy import Integer, Column, create_engine, ForeignKey
from sqlalchemy.orm import relationship, joinedload, subqueryload, Session
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()
engine = None
session = None

class Parent(Base):
    __tablename__ = 'parent'
    id = Column(Integer, primary_key=True)
    children = relationship("Child")

class Child(Base):
    __tablename__ = 'child'
    id = Column(Integer, primary_key=True)
    parent_id = Column(Integer, ForeignKey('parent.id'))

# Init with name of file, default number of items
Profiler.init("test_loads", 1000)

@Profiler.setup_once
def setup_once(dburl, echo, num):
    "setup once.  create an engine, insert fixture data"
    global engine
    engine = create_engine(dburl, echo=echo)
    Base.metadata.drop_all(engine)
    Base.metadata.create_all(engine)
    sess = Session(engine)
    sess.add_all([
        Parent(children=[Child() for j in range(100)])
        for i in range(num)
    ])
    sess.commit()

@Profiler.setup
def setup(dburl, echo, num):
    "setup per test.  create a new Session."
    global session
    session = Session(engine)
    # pre-connect so this part isn't profiled (if we choose)
    session.connection()

@Profiler.profile
def test_lazyload(n):
    "load everything, no eager loading."

    for parent in session.query(Parent):
        parent.children

@Profiler.profile
def test_joinedload(n):
    "load everything, joined eager loading."

    for parent in session.query(Parent).options(joinedload("children")):
        parent.children

@Profiler.profile
def test_subqueryload(n):
    "load everything, subquery eager loading."

    for parent in session.query(Parent).options(subqueryload("children")):
        parent.children

if __name__ == '__main__':
    Profiler.main()
```

我们可以直接运行我们的新脚本：

```py
$ python test_loads.py  --dburl postgresql+psycopg2://scott:tiger@localhost/test
Running setup once...
Tests to run: test_lazyload, test_joinedload, test_subqueryload
test_lazyload : load everything, no eager loading. (1000 iterations); total time 11.971159 sec
test_joinedload : load everything, joined eager loading. (1000 iterations); total time 2.754592 sec
test_subqueryload : load everything, subquery eager loading. (1000 iterations); total time 2.977696 sec
```

#### 文件列表

文件列表：

+   bulk_updates.py - 这一系列的测试将展示不同方法批量更新大量行（正在构建！目前只有一个测试）

+   large_resultsets.py - 在这一系列的测试中，我们关注的是加载大量非常小而简单的行的时间。

+   bulk_inserts.py - 这一系列的测试展示了批量插入大量行的不同方法。

+   short_selects.py - 这一系列的测试展示了通过主键选择单个记录的不同方法。

+   single_inserts.py - 在这一系列的测试中，我们看到了一种在不同事务中插入行并且之后返回到实质上“关闭”状态的方法。这类似于启动数据库连接的 API 调用，插入行，提交并关闭的过程。

+   __main__.py - 允许 examples/performance 包作为脚本运行。

#### 使用时间运行所有测试

这是运行的默认形式：

```py
$ python -m examples.performance single_inserts
Tests to run: test_orm_commit, test_bulk_save,
              test_bulk_insert_dictionaries, test_core,
              test_core_query_caching, test_dbapi_raw_w_connect,
              test_dbapi_raw_w_pool

test_orm_commit : Individual INSERT/COMMIT pairs via the
    ORM (10000 iterations); total time 13.690218 sec
test_bulk_save : Individual INSERT/COMMIT pairs using
    the "bulk" API  (10000 iterations); total time 11.290371 sec
test_bulk_insert_dictionaries : Individual INSERT/COMMIT pairs using
    the "bulk" API with dictionaries (10000 iterations);
    total time 10.814626 sec
test_core : Individual INSERT/COMMIT pairs using Core.
    (10000 iterations); total time 9.665620 sec
test_core_query_caching : Individual INSERT/COMMIT pairs using Core
    with query caching (10000 iterations); total time 9.209010 sec
test_dbapi_raw_w_connect : Individual INSERT/COMMIT pairs w/ DBAPI +
    connection each time (10000 iterations); total time 9.551103 sec
test_dbapi_raw_w_pool : Individual INSERT/COMMIT pairs w/ DBAPI +
    connection pool (10000 iterations); total time 8.001813 sec
```

#### 为个别测试输出简要概要

可以为所有测试或更常见的是个别测试转储 Python 分析输出：

```py
$ python -m examples.performance single_inserts --test test_core --num 1000 --dump
Tests to run: test_core
test_core : Individual INSERT/COMMIT pairs using Core. (1000 iterations); total fn calls 186109
         186109 function calls (186102 primitive calls) in 1.089 seconds

   Ordered by: internal time, call count

   ncalls  tottime  percall  cumtime  percall filename:lineno(function)
     1000    0.634    0.001    0.634    0.001 {method 'commit' of 'sqlite3.Connection' objects}
     1000    0.154    0.000    0.154    0.000 {method 'execute' of 'sqlite3.Cursor' objects}
     1000    0.021    0.000    0.074    0.000 /Users/classic/dev/sqlalchemy/lib/sqlalchemy/sql/compiler.py:1950(_get_colparams)
     1000    0.015    0.000    0.034    0.000 /Users/classic/dev/sqlalchemy/lib/sqlalchemy/engine/default.py:503(_init_compiled)
        1    0.012    0.012    1.091    1.091 examples/performance/single_inserts.py:79(test_core)

    ...
```

#### 编写你自己的测试套件

性能分析套件系统是可扩展的，并且可以应用于您自己的一组测试。这是在决定某些性能关键例程的正确方法时使用的宝贵技术。例如，如果我们想要分析几种加载之间的差异，我们可以创建一个名为`test_loads.py`的文件，其中包含以下内容：

```py
from examples.performance import Profiler
from sqlalchemy import Integer, Column, create_engine, ForeignKey
from sqlalchemy.orm import relationship, joinedload, subqueryload, Session
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()
engine = None
session = None

class Parent(Base):
    __tablename__ = 'parent'
    id = Column(Integer, primary_key=True)
    children = relationship("Child")

class Child(Base):
    __tablename__ = 'child'
    id = Column(Integer, primary_key=True)
    parent_id = Column(Integer, ForeignKey('parent.id'))

# Init with name of file, default number of items
Profiler.init("test_loads", 1000)

@Profiler.setup_once
def setup_once(dburl, echo, num):
    "setup once.  create an engine, insert fixture data"
    global engine
    engine = create_engine(dburl, echo=echo)
    Base.metadata.drop_all(engine)
    Base.metadata.create_all(engine)
    sess = Session(engine)
    sess.add_all([
        Parent(children=[Child() for j in range(100)])
        for i in range(num)
    ])
    sess.commit()

@Profiler.setup
def setup(dburl, echo, num):
    "setup per test.  create a new Session."
    global session
    session = Session(engine)
    # pre-connect so this part isn't profiled (if we choose)
    session.connection()

@Profiler.profile
def test_lazyload(n):
    "load everything, no eager loading."

    for parent in session.query(Parent):
        parent.children

@Profiler.profile
def test_joinedload(n):
    "load everything, joined eager loading."

    for parent in session.query(Parent).options(joinedload("children")):
        parent.children

@Profiler.profile
def test_subqueryload(n):
    "load everything, subquery eager loading."

    for parent in session.query(Parent).options(subqueryload("children")):
        parent.children

if __name__ == '__main__':
    Profiler.main()
```

我们可以直接运行我们的新脚本：

```py
$ python test_loads.py  --dburl postgresql+psycopg2://scott:tiger@localhost/test
Running setup once...
Tests to run: test_lazyload, test_joinedload, test_subqueryload
test_lazyload : load everything, no eager loading. (1000 iterations); total time 11.971159 sec
test_joinedload : load everything, joined eager loading. (1000 iterations); total time 2.754592 sec
test_subqueryload : load everything, subquery eager loading. (1000 iterations); total time 2.977696 sec
```

### 太空侵略者

使用 SQLite 作为状态机的太空侵略者游戏。

最初开发于 2012 年。已适配为在 Python 3 中运行。

在文本控制台中使用 ASCII 艺术运行。

![../_images/space_invaders.jpg](img/df5d8744e7ec946672bdde78d1db980c.png)

运行：

```py
python -m examples.space_invaders.space_invaders
```

在运行时，观察日志中的 SQL 输出：

```py
tail -f space_invaders.log
```

尽情享受！

文件清单：

+   space_invaders.py

### 版本化对象

#### 带有历史表的版本控制

演示了一个扩展，为实体创建版本表并存储每次更改的记录。给定的扩展生成一个匿名的“历史”类，表示目标对象的历史版本。

与使用时间行进行版本控制示例进行比较，该示例将更新写入为同一表中的新行，而不使用单独的历史表。

使用通过一个单元测试模块`test_versioning.py`进行演示，可以像其他模块一样运行，内部使用`unittest`：

```py
python -m examples.versioned_history.test_versioning
```

一个示例用法片段，使用声明性：

```py
from history_meta import Versioned, versioned_session

class Base(DeclarativeBase):
    pass

class SomeClass(Versioned, Base):
    __tablename__ = 'sometable'

    id = Column(Integer, primary_key=True)
    name = Column(String(50))

    def __eq__(self, other):
        assert type(other) is SomeClass and other.id == self.id

Session = sessionmaker(bind=engine)
versioned_session(Session)

sess = Session()
sc = SomeClass(name='sc1')
sess.add(sc)
sess.commit()

sc.name = 'sc1modified'
sess.commit()

assert sc.version == 2

SomeClassHistory = SomeClass.__history_mapper__.class_

assert sess.query(SomeClassHistory).\
            filter(SomeClassHistory.version == 1).\
            all() \
            == [SomeClassHistory(version=1, name='sc1')]
```

`Versioned`混合类设计用于与声明性一起使用。要将扩展与经典映射器一起使用，可以应用`_history_mapper`函数：

```py
from history_meta import _history_mapper

m = mapper(SomeClass, sometable)
_history_mapper(m)

SomeHistoryClass = SomeClass.__history_mapper__.class_
```

版本示例还与 ORM 乐观并发功能集成，文档化在配置版本计数器。要启用此功能，请将标志`Versioned.use_mapper_versioning`设置为 True：

```py
class SomeClass(Versioned, Base):
    __tablename__ = 'sometable'

    use_mapper_versioning = True

    id = Column(Integer, primary_key=True)
    name = Column(String(50))

    def __eq__(self, other):
        assert type(other) is SomeClass and other.id == self.id
```

如果两个具有相同版本标识符的`SomeClass`实例被同时更新并发送到数据库进行并发更新，如果数据库隔离级别允许两个 UPDATE 语句继续进行，其中一个将失败，因为它不再针对最后已知的版本标识符。

文件清单：

+   test_versioning.py - 展示了`history_meta.py`模块函数的使用的单元测试。

+   history_meta.py - 带有版本控制的混合类和其他实用工具。#### 使用时间行进行版本控制

几个示例说明拦截更改的技术，这些更改首先被解释为对行的更新，而实际上将其转换为对新行的插入，保留先前的行作为历史版本。

与带有历史表的版本控制示例进行比较，该示例将历史行写入单独的历史表。

文件清单：

+   versioned_rows.py - 说明了拦截对象更改的方法，将对单个行的 UPDATE 语句转换为 INSERT 语句，以便使用新数据插入新行，保留旧行不变。

+   versioned_rows_w_versionid.py - 说明了拦截对象更改的方法，将对单个行的 UPDATE 语句转换为 INSERT 语句，以便使用新数据插入新行，保留旧行不变。

+   versioned_map.py - 围绕“垂直表”结构的概念构建的`versioned_rows`示例的变体，类似于垂直属性映射示例中所示的内容。

+   versioned_update_old_row.py - 说明了`versioned_rows.py`的相同 UPDATE 到 INSERT 技术，但还会对**旧**行进行 UPDATE 以影响时间戳的更改。还包括一个`SessionEvents.do_orm_execute()`挂钩来将查询限制为只有最新版本。  #### 使用历史表进行版本控制

展示了一个创建实体的版本表并为每个更改存储记录的扩展。给定的扩展生成一个匿名的“history”类，表示目标对象的历史版本。

与使用时间行版本化的例子相比，这些例子将更新写入相同表中的新行中，而不使用单独的历史表。

通过一个单元测试模块`test_versioning.py`来说明使用方法，可以像其他模块一样运行，内部使用`unittest`：

```py
python -m examples.versioned_history.test_versioning
```

一个示例用法片段，使用声明式：

```py
from history_meta import Versioned, versioned_session

class Base(DeclarativeBase):
    pass

class SomeClass(Versioned, Base):
    __tablename__ = 'sometable'

    id = Column(Integer, primary_key=True)
    name = Column(String(50))

    def __eq__(self, other):
        assert type(other) is SomeClass and other.id == self.id

Session = sessionmaker(bind=engine)
versioned_session(Session)

sess = Session()
sc = SomeClass(name='sc1')
sess.add(sc)
sess.commit()

sc.name = 'sc1modified'
sess.commit()

assert sc.version == 2

SomeClassHistory = SomeClass.__history_mapper__.class_

assert sess.query(SomeClassHistory).\
            filter(SomeClassHistory.version == 1).\
            all() \
            == [SomeClassHistory(version=1, name='sc1')]
```

`Versioned` mixin 设计为与声明性一起使用。要将扩展与经典映射器一起使用，可以应用`_history_mapper`函数：

```py
from history_meta import _history_mapper

m = mapper(SomeClass, sometable)
_history_mapper(m)

SomeHistoryClass = SomeClass.__history_mapper__.class_
```

版本控制示例还与在配置版本计数器中记录的 ORM 乐观并发功能集成。要启用此功能，请将标志`Versioned.use_mapper_versioning`设置为 True：

```py
class SomeClass(Versioned, Base):
    __tablename__ = 'sometable'

    use_mapper_versioning = True

    id = Column(Integer, primary_key=True)
    name = Column(String(50))

    def __eq__(self, other):
        assert type(other) is SomeClass and other.id == self.id
```

上面，如果两个具有相同版本标识符的`SomeClass`实例被同时更新并发送到数据库进行并发更新，如果数据库隔离级别允许这两个 UPDATE 语句继续进行，其中一个将失败，因为它不再是针对最后已知版本标识符的。

文件列表：

+   test_versioning.py - 展示了`history_meta.py`模块函数的用法的单元测试。

+   history_meta.py - 版本混合类和其他实用工具。

#### 使用时间行版本化

几个示例说明了拦截更改的技术，这些更改首先被解释为对行的 UPDATE，而实际上将其转换为对新行的 INSERT，将先前的行保留为历史版本。

与带有历史表的版本控制示例进行比较，该示例将历史行写入单独的历史表中。

文件列表：

+   versioned_rows.py - 展示了拦截对象更改的方法，将对单行的 UPDATE 语句转换为 INSERT 语句，以便插入具有新数据的新行，同时保留旧行不变。

+   versioned_rows_w_versionid.py - 展示了拦截对象更改的方法，将对单行的 UPDATE 语句转换为 INSERT 语句，以便插入具有新数据的新行，同时保留旧行不变。

+   versioned_map.py - 围绕“垂直表”结构概念构建的 versioned_rows 示例的变体，类似于垂直属性映射示例中所示的那些。

+   versioned_update_old_row.py - 展示了与`versioned_rows.py`相同的 UPDATE 转换为 INSERT 技术，但还发出了对**旧**行的 UPDATE 以影响时间戳的更改。还包括一个`SessionEvents.do_orm_execute()`钩子，以限制查询仅针对最新版本。

### 垂直属性映射

展示了“垂直表”映射。

“垂直表”是指将对象的各个属性存储为表中的不同行的技术。 “垂直表”技术用于持久化可以具有各种属性集的对象，但牺牲了简单的查询控制和简洁性。它通常在内容/文档管理系统中找到，以灵活地表示用户创建的结构。

给出了两种方法的变体。在第二种方法中，每行引用一个包含有关存储在属性中的信息类型的“数据类型”，例如整数、字符串或日期。

示例：

```py
shrew = Animal(u'shrew')
shrew[u'cuteness'] = 5
shrew[u'weasel-like'] = False
shrew[u'poisonous'] = True

session.add(shrew)
session.flush()

q = (session.query(Animal).
     filter(Animal.facts.any(
       and_(AnimalFact.key == u'weasel-like',
            AnimalFact.value == True))))
print('weasel-like animals', q.all())
```

文件列表：

+   dictlike-polymorphic.py - 将多态值的垂直表映射为字典。

+   dictlike.py - 将垂直表映射为字典。

## 继承映射配方

### 基本继承映射

单表、联表和具体表继承的工作示例，如映射类继承层次结构中所述。

文件列表：

+   joined.py - 联接表（每个子类一个表）继承示例。

+   concrete.py - 具体表（每类一表）继承示例。

+   single.py - 单表（按层次结构划分）继承示例。  ### 基本继承映射

作为 Mapping Class Inheritance Hierarchies 中描述的单表、连接表和具体表继承的工作示例。 

文件清单：

+   joined.py - 连接表（每子类一表）继承示例。

+   concrete.py - 具体表（每类一表）继承示例。

+   single.py - 单表（按层次结构划分）继承示例。

## 特殊 API

### 属性仪器化

示例说明了对 SQLAlchemy 属性管理系统的修改。

文件清单：

+   listen_for_events.py - 演示了如何将事件附加到所有仪器化属性并监听更改事件。

+   active_column_defaults.py - 演示了如何使用`AttributeEvents.init_scalar()`事件，结合核心列默认值来提供 ORM 对象，当访问未设置的属性时自动产生默认值。

+   custom_management.py - 演示了如何使用`sqlalchemy.ext.instrumentation`扩展包进行自定义类仪器化。  ### 水平分片

使用 SQLAlchemy 分片 API 的基本示例。分片是指将数据横向扩展到多个数据库。

“分片”映射的基本组件包括：

+   多个`Engine`实例，每个实例分配一个“分片 ID”。这些`Engine`实例可以引用不同的数据库，或者同一数据库中的不同模式/帐户，或者甚至可以仅通过选项来区分，当使用时会使它们访问不同的模式或表。

+   给定要保存的实例，可以返回单个分片 ID 的函数；这称为“shard_chooser”。

+   可以返回适用于特定实例标识符的分片 ID 列表的函数；这称为“id_chooser”。如果返回所有分片 ID，则会搜索所有分片。

+   给定特定查询（“query_chooser”），可以返回要尝试的分片 ID 列表的函数。如果返回所有分片 ID，则会查询所有分片并将结果合并在一起。

在这些示例中，针对相同的基本示例使用不同类型的分片，适应以每个大陆为基础的天气数据。我们提供了示例的 shard_chooser、id_chooser 和 query_chooser 函数。query_chooser 演示了检查 SQL 表达式元素以尝试确定请求的单个分片。

构建通用分片例程是解决将实例组织在多个数据库中的问题的一种雄心勃勃的方法。对于更直接的替代方案，“不同实体”方法是一种简单的将对象分配给不同表（以及潜在的数据库节点）的显式方法 - 在维基上描述为[EntityName](https://www.sqlalchemy.org/trac/wiki/UsageRecipes/EntityName)。

文件列表：

+   separate_databases.py - 演示使用不同的 SQLite 数据库进行分片。

+   separate_tables.py - 演示使用单个 SQLite 数据库进行分片，但将使用命名约定创建多个表。

+   separate_schema_translates.py - 演示使用具有多个模式的单个数据库进行分片，其中可以为每个分片使用不同的“schema_translates_map”。

+   asyncio.py - 演示与 asyncio 一起使用分片 API。### 属性检测

演示对 SQLAlchemy 属性管理系统的修改的示例。

文件列表：

+   listen_for_events.py - 演示如何将事件附加到所有被检测属性并监听更改事件。

+   active_column_defaults.py - 演示使用`AttributeEvents.init_scalar()`事件，结合核心列默认值，为 ORM 对象提供在访问未设置属性时自动生成默认值的功能。

+   custom_management.py - 演示自定义类检测，使用`sqlalchemy.ext.instrumentation`扩展包。

### 水平分片

使用 SQLAlchemy 分片 API 的基本示例。分片是指在多个数据库之间水平扩展数据。

“分片”映射的基本组件包括：

+   多个`Engine` 实例，每个分配一个“分片 id”。这些`Engine` 实例可以引用不同的数据库，或者同一数据库中的不同模式/帐户，或者它们甚至可以仅通过选项进行区分，这些选项将在使用时导致它们访问不同的模式或表。

+   一个函数可以返回给定要保存的实例的单个分片 id；这称为“shard_chooser”。

+   一个函数可以返回适用于特定实例标识符的分片 id 列表；这称为“id_chooser”。如果返回所有分片 id，则将搜索所有分片。

+   一个函数可以返回给定查询的尝试分片 id 列表；如果返回所有分片 id，则将查询所有分片并将结果连接在一起。

在这些示例中，针对同一基本示例使用了不同类型的分片，该示例适用于按大陆基础提供天气数据。我们提供了示例 shard_chooser、id_chooser 和 query_chooser 函数。query_chooser 展示了对 SQL 表达式元素的检查，以尝试确定请求的单个分片。

构建通用分片例程是组织实例在多个数据库中的一种雄心勃勃的方法。对于更直接的替代方案，“不同实体”方法是一种将对象分配给不同表（和潜在的数据库节点）的简单方法 - 在维基上描述为[EntityName](https://www.sqlalchemy.org/trac/wiki/UsageRecipes/EntityName)。

文件列表：

+   separate_databases.py - 展示了使用不同的 SQLite 数据库进行分片的示例。

+   separate_tables.py - 展示了使用单个 SQLite 数据库进行分片的示例，但是会使用多个表并遵循命名约定。

+   separate_schema_translates.py - 展示了使用具有多个模式的单个数据库进行分片的示例，其中每个分片可以使用不同的“schema_translates_map”。

+   asyncio.py - 展示了与 asyncio 一起使用的分片 API。

## 扩展 ORM

### ORM 查询事件

展示了增强 ORM SELECT 行为的示例，这些示例由 `Session.execute()` 与 2.0 风格 的 `select()` 以及 1.x 风格 的 `Query` 对象一起使用。

示例包括演示`with_loader_criteria()`选项以及`SessionEvents.do_orm_execute()`钩子。

截至 SQLAlchemy 1.4 版本，`Query` 构造与`Select` 构造合并在一起，因此这两个对象大部分相同。

文件列表：

+   temporal_range.py - 演示将应用于选定实体的自定义每个查询条件。

+   filter_public.py - 演示应用于特定类型实体的全局条件的基本技术。### Dogpile Caching

演示如何嵌入[dogpile.cache](https://dogpilecache.sqlalchemy.org/)功能与 ORM 查询，允许完全控制缓存以及从长期缓存中获取“延迟加载”属性的能力。

在这个演示中，演示了以下技术：

+   使用`SessionEvents.do_orm_execute()`事件钩子

+   绕过`Session.execute()`以从自定义缓存源而不是数据库中提取的基本技术。

+   使用 dogpile.cache 进行基本缓存，使用“regions”允许全局控制一组固定配置。

+   使用自定义`UserDefinedOption`对象配置语句对象中的选项。

另请参阅

重新执行语句 - 包括这里介绍的技术的一个通用示例。

例如：

```py
# query for Person objects, specifying cache
stmt = select(Person).options(FromCache("default"))

# specify that each Person's "addresses" collection comes from
# cache too
stmt = stmt.options(RelationshipCache(Person.addresses, "default"))

# execute and results
result = session.execute(stmt)

print(result.scalars().all())
```

要运行，必须安装 SQLAlchemy 和 dogpile.cache，或者在当前 PYTHONPATH 上。演示将为数据文件创建一个本地目录，插入初始数据，然后运行。第二次运行演示将利用已经存在的缓存文件，并且只会发出一条 SQL 语句针对两个表 - 但显示的结果将利用数十个从缓存中获取的延迟加载。

演示脚本本身，按复杂性顺序作为 Python 模块运行，以便相对导入正常工作：

```py
python -m examples.dogpile_caching.helloworld

python -m examples.dogpile_caching.relationship_caching

python -m examples.dogpile_caching.advanced

python -m examples.dogpile_caching.local_session_caching
```

文件列表：

+   environment.py - 建立数据/缓存文件路径和配置，必要时引导固定数据。

+   caching_query.py - 代表允许在 SQLAlchemy 中使用 Dogpile 缓存的函数和类。引入一个名为 FromCache 的查询选项。

+   model.py - 数据模型，表示具有多个地址对象的人员，每个地址对象都有邮政编码、城市和国家。

+   fixture_data.py - 安装一些示例数据。这里有一些美国/加拿大城市的少数邮政编码。然后，安装了 100 个人员记录，每个记录都有一个随机选择的邮政编码。

+   helloworld.py - 演示了如何加载一些数据，并缓存结果。

+   relationship_caching.py - 展示了如何在关系终点添加缓存选项，以便惰性加载从缓存中加载。

+   advanced.py - 演示了 Query 与 FromCache 选项的使用，包括前端加载、缓存失效和集合缓存。

+   local_session_caching.py - 这个例子创建了一个新的 dogpile.cache 后端，它将数据持久化存储在当前会话的字典中。删除会话，缓存就消失了。### ORM 查询事件

说明如何使用`Session.execute()`与 2.0 样式的`select()`一起增强 ORM SELECT 行为的示例，以及 1.x 样式的`Query`对象。

示例包括演示`with_loader_criteria()`选项以及`SessionEvents.do_orm_execute()`钩子。

从 SQLAlchemy 1.4 开始，`Query`构造与`Select`构造统一，因此这两个对象大部分相同。

文件列表：

+   temporal_range.py - 展示了将应用于选定实体的自定义每个查询条件。

+   filter_public.py - 展示了应用于特定类型实体的全局条件。

### Dogpile 缓存

说明如何在 ORM 查询中嵌入[dogpile.cache](https://dogpilecache.sqlalchemy.org/)功能，允许完全的缓存控制，以及从长期缓存中拉取“惰性加载”属性的能力。

这个演示展示了以下技术：

+   使用`SessionEvents.do_orm_execute()`事件挂钩

+   绕过`Session.execute()`以从自定义缓存源而不是数据库中提取的基本技术。

+   基本的缓存与 dogpile.cache，使用允许全局控制一组固定配置的“区域”。

+   使用自定义`UserDefinedOption`对象配置语句对象中的选项。

另请参阅

重新执行语句 - 包括这里介绍的技术的一般示例。

例如：

```py
# query for Person objects, specifying cache
stmt = select(Person).options(FromCache("default"))

# specify that each Person's "addresses" collection comes from
# cache too
stmt = stmt.options(RelationshipCache(Person.addresses, "default"))

# execute and results
result = session.execute(stmt)

print(result.scalars().all())
```

要运行，必须安装 SQLAlchemy 和 dogpile.cache，或者在当前 PYTHONPATH 上安装。演示将为数据文件创建一个本地目录，插入初始数据，并运行。再次运行演示将利用已经存在的缓存文件，并且只会发出一条针对两个表的 SQL 语句 - 但显示的结果将利用几十个懒加载，所有这些懒加载都从缓存中拉取。

演示脚本本身，按复杂度顺序运行为 Python 模块，以便相对导入正常工作：

```py
python -m examples.dogpile_caching.helloworld

python -m examples.dogpile_caching.relationship_caching

python -m examples.dogpile_caching.advanced

python -m examples.dogpile_caching.local_session_caching
```

文件清单：

+   environment.py - 确定数据/缓存文件路径和配置，如有必要，引导安装装置数据。

+   caching_query.py - 表示使用 SQLAlchemy 可以与 Dogpile 缓存一起使用的函数和类。介绍了一个名为 FromCache 的查询选项。

+   model.py - 数据模型，表示具有多个 Address 对象的 Person，每个 Address 对象都有 PostalCode、City、Country。

+   fixture_data.py - 安装一些示例数据。这里有一些美国/加拿大几个城市的一些邮政编码。然后，安装了 100 个 Person 记录，每个记录都有一个随机选择的邮政编码。

+   helloworld.py - 演示如何加载一些数据，并缓存结果。

+   relationship_caching.py - 演示如何在关系端点上添加缓存选项，以便延迟加载从缓存加载。

+   advanced.py - 演示 Query 与 FromCache 选项的使用，包括前端加载、缓存失效和集合缓存。

+   local_session_caching.py - 此示例创建一个新的 dogpile.cache 后端，该后端将数据持久化在当前会话的字典中。删除会话，缓存消失。
