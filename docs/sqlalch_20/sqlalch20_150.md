# SQLAlchemy 0.5 中有什么新功能？

> 原文：[`docs.sqlalchemy.org/en/20/changelog/migration_05.html`](https://docs.sqlalchemy.org/en/20/changelog/migration_05.html)

关于本文档

本文档描述了 SQLAlchemy 版本 0.4（最后发布于 2008 年 10 月 12 日）与 SQLAlchemy 版本 0.5（最后发布于 2010 年 1 月 16 日）之间的变化。

文档日期：2009 年 8 月 4 日

本指南记录了影响用户从 SQLAlchemy 0.4 系列迁移到 0.5 系列的 API 更改。对于那些从[Essential SQLAlchemy](https://oreilly.com/catalog/9780596516147/)开始工作的人也是推荐的，该书只涵盖了 0.4 版本，甚至在其中有一些旧的 0.3 版本的内容。请注意，SQLAlchemy 0.5 删除了在整个 0.4 系列中已弃用的许多行为，并且还弃用了更多与 0.4 特定的行为。

## 主要文档更改

文档的一些部分已经完全重写，可以作为新 ORM 功能的介绍。特别是`Query`和`Session`对象在 API 和行为上有一些明显的区别，这些区别从根本上改变了许多基本操作的方式，特别是构建高度定制的 ORM 查询和处理过时的会话状态、提交和回滚。

+   [ORM 教程](https://www.sqlalchemy.org/docs/05/ormtutorial.html)

+   [会话文档](https://www.sqlalchemy.org/docs/05/session.html)

## 弃用来源

另一个信息源记录在一系列单元测试中，展示了一些常见`Query`模式的最新用法；此文件可在[source:sqlalchemy/trunk/test/orm/test_deprecations.py]中查看。

## 要求更改

+   需要 Python 2.4 或更高版本。SQLAlchemy 0.4 系列是最后一个支持 Python 2.3 的版本。

## 对象关系映射

+   **Query 中的列级表达式。** - 如[教程](https://www.sqlalchemy.org/docs/05/ormtutorial.html)中所述，`Query`具有创建特定 SELECT 语句的能力，而不仅仅是针对完整行的语句：

    ```py
    session.query(User.name, func.count(Address.id).label("numaddresses")).join(
        Address
    ).group_by(User.name)
    ```

    任何多列/实体查询返回的元组都是*命名*元组：

    ```py
    for row in (
        session.query(User.name, func.count(Address.id).label("numaddresses"))
        .join(Address)
        .group_by(User.name)
    ):
        print("name", row.name, "number", row.numaddresses)
    ```

    `Query`具有`statement`访问器，以及一个`subquery()`方法，允许`Query`用于创建更复杂的组合：

    ```py
    subq = (
        session.query(Keyword.id.label("keyword_id"))
        .filter(Keyword.name.in_(["beans", "carrots"]))
        .subquery()
    )
    recipes = session.query(Recipe).filter(
        exists()
        .where(Recipe.id == recipe_keywords.c.recipe_id)
        .where(recipe_keywords.c.keyword_id == subq.c.keyword_id)
    )
    ```

+   **建议使用显式 ORM 别名进行别名连接** - `aliased()`函数生成一个类的“别名”，允许在 ORM 查询中与别名进行细粒度控制。虽然仍然可以使用表级别的别名（即`table.alias()`），但 ORM 级别的别名保留了 ORM 映射对象的语义，这对于继承映射、选项和其他场景非常重要。例如：

    ```py
    Friend = aliased(Person)
    session.query(Person, Friend).join((Friend, Person.friends)).all()
    ```

+   **query.join()功能大大增强。** - 您现在可以通过多种方式指定连接的目标和 ON 子句。可以仅提供目标类，SQLA 将尝试通过相同的外键形式连接到它，就像`table.join(someothertable)`一样。还可以提供目标和显式的 ON 条件，其中 ON 条件可以是`relation()`名称，实际类描述符或 SQL 表达式。或者也可以像以前那样只提供`relation()`名称或类描述符。请参阅 ORM 教程，其中有几个示例。

+   **建议使用声明性用于不需要（且不喜欢）表和映射器之间抽象的应用程序** - [/docs/05/reference/ext/declarative.html 声明性]模块用于将`Table`、`mapper()`和用户定义的类对象的表达结合在一起，强烈建议使用它，因为它简化了应用程序配置，确保了“每个类一个映射器”的模式，并允许对不同的`mapper()`调用提供完整的配置范围。将`mapper()`和`Table`的使用分开现在被称为“经典 SQLAlchemy 使用方式”，当然可以与声明性混合使用。

+   **已从类中删除了`.c.`属性**（即`MyClass.c.somecolumn`）。与 0.4 版本一样，类级别的属性可用作查询元素，即`Class.c.propname`现在被`Class.propname`所取代，并且`c`属性仍然保留在`Table`对象上，其中它们指示存在于表上的`Column`对象的命名空间。

    要获取映射类的表（如果您之前没有保留它）：

    ```py
    table = class_mapper(someclass).mapped_table
    ```

    迭代遍历列：

    ```py
    for col in table.c:
        print(col)
    ```

    使用特定列进行操作：

    ```py
    table.c.somecolumn
    ```

    类绑定描述符支持完整的 Column 运算符集，以及文档化的与关系有关的运算符，如`has()`、`any()`、`contains()`等。

    删除`.c.`的原因是，在 0.5 版本中，类绑定描述符可能具有不同的含义，以及关于类映射的信息，与普通的`Column`对象不同-并且存在一些情况，您会特别想要使用其中之一。通常，使用类绑定描述符会调用一组映射/多态感知的转换，而使用表绑定列则不会。在 0.4 版本中，这些转换适用于所有表达式，但是 0.5 版本完全区分列和映射描述符，仅将转换应用于后者。因此，在许多情况下，特别是在处理连接的表继承配置以及使用`query(<columns>)`时，`Class.propname`和`table.c.colname`不可互换。

    例如，`session.query(users.c.id, users.c.name)`与`session.query(User.id, User.name)`是不同的；在后一种情况下，`Query`知道正在使用的映射器，并且可以使用进一步的映射器特定操作，如`query.join(<propname>)`，`query.with_parent()`等，但在前一种情况下不行。此外，在多态继承场景中，类绑定描述符指的是多态可选择使用的列，而不一定是直接对应描述符的表列。例如，一组类通过连接表继承与`person`表相关联，每个表的`person_id`列都将其`Class.person_id`属性映射到`person`中的`person_id`列，而不是其子类表。版本 0.4 会自动将此行为映射到表绑定的`Column`对象上。在 0.5 中，已移除了此自动转换，因此实际上*可以*使用表绑定列来覆盖多态查询时发生的转换；这使得`Query`能够在连接表或具体表继承设置中创建优化的选择，以及可移植的子查询等。

+   **会话现在与事务自动同步。** 会话现在默认情况下自动与事务同步，包括自动刷新和自动过期。除非使用`autocommit`选项禁用，否则始终存在事务。当所有三个标志都设置为默认值时，会话在回滚后能够优雅地恢复，并且很难将过时数据导入会话中。详细信息请参阅新的会话文档。

+   **隐式排序已移除**。这将影响依赖于 SA 的“隐式排序”行为的 ORM 用户，该行为规定所有没有`order_by()`的 Query 对象将按照主映射表的“id”或“oid”列进行排序，并且所有延迟/急切加载的集合都应用类似的排序。在 0.5 中，必须显式配置`mapper()`和`relation()`对象上的自动排序（如果需要），或者在使用`Query`时。

    要将 0.4 映射转换为 0.5，使其排序行为与 0.4 或之前的版本极为相似，请在`mapper()`和`relation()`上使用`order_by`设置：

    ```py
    mapper(
        User,
        users,
        properties={"addresses": relation(Address, order_by=addresses.c.id)},
        order_by=users.c.id,
    )
    ```

    要在 backref 上设置排序，请使用`backref()`函数：

    ```py
    "keywords": relation(
        Keyword,
        secondary=item_keywords,
        order_by=keywords.c.name,
        backref=backref("items", order_by=items.c.id),
    )
    ```

    使用声明式？为了帮助满足新的`order_by`要求，现在可以使用稍后在 Python 中评估的字符串来设置`order_by`和相关内容（这仅适用于声明式，而不是普通的映射器）：

    ```py
    class MyClass(MyDeclarativeBase):
        ...
        "addresses": relation("Address", order_by="Address.id")
    ```

    通常在加载基于列表的项目集合的`relation()`上设置`order_by`是一个好主意，因为否则无法影响排序。除此之外，最佳实践是使用`Query.order_by()`来控制加载的主要实体的排序。

+   **Session 现在是 autoflush=True/autoexpire=True/autocommit=False。** - 要设置它，只需调用`sessionmaker()`而不带任何参数。现在`transactional=True`的名称是`autocommit=False`。刷新发生在每次查询时（可通过`autoflush=False`禁用），在每次`commit()`之前（一如既往），以及在每次`begin_nested()`之前（因此回滚到 SAVEPOINT 是有意义的）。所有对象在每次`commit()`和每次`rollback()`后都会过期。回滚后，待定对象被清除，删除的对象移回持久状态。这些默认设置非常好地协同工作，实际上不再需要像`clear()`这样的旧技术（也已重命名为`expunge_all()`）。

    P.S.: 在`rollback()`后，会话现在是可重用的。标量和集合属性的更改、添加和删除都会被回滚。

+   **session.add()取代了 session.save()、session.update()、session.save_or_update()。** - `session.add(someitem)`和`session.add_all([list of items])`方法取代了`save()`、`update()`和`save_or_update()`。这些方法将在整个 0.5 版本中继续被弃用。

+   **backref 配置更简洁。** - `backref()`函数现在在未明确声明时使用前向`relation()`的`primaryjoin`和`secondaryjoin`参数。在两个方向上分别指定`primaryjoin`/`secondaryjoin`不再必要。

+   **简化的多态选项。** - ORM 的“多态加载”行为已经简化。在 0.4 版本中，mapper()有一个名为`polymorphic_fetch`的参数，可以配置为`select`或`deferred`。此选项已被移除；现在映射器将仅推迟未包含在 SELECT 语句中的任何列。实际使用的 SELECT 语句由`with_polymorphic`映射器参数控制（在 0.4 中也有，替代了`select_table`），以及`Query`上的`with_polymorphic()`方法（同样在 0.4 中）。

    对继承类的延迟加载进行了改进，现在映射器在所有情况下都会生成“优化”版本的 SELECT 语句；也就是说，如果类 B 继承自 A，并且类 B 上的几个属性已过期，刷新操作将只包括 B 的表在 SELECT 语句中，不会 JOIN 到 A。

+   `Session`上的`execute()`方法将普通字符串转换为`text()`构造，以便所有绑定参数都可以指定为“:bindname”而无需显式调用`text()`。如果需要“原始”SQL，请使用`session.connection().execute("raw text")`。

+   `session.Query().iterate_instances()`已重命名为`instances()`。旧的返回列表而不是迭代器的`instances()`方法已不复存在。如果你依赖于该行为，应该使用`list(your_query.instances())`。

## 扩展 ORM

在 0.5 版本中，我们将继续提供更多修改和扩展 ORM 的方法。以下是摘要：

+   **MapperExtension.** - 这是经典的扩展类，仍然存在。很少需要的方法是`create_instance()`和`populate_instance()`。要控制从数据库加载对象时的初始化，使用`reconstruct_instance()`方法，或者更容易地使用文档中描述的`@reconstructor`装饰器。

+   **SessionExtension.** - 这是一个易于使用的会话事件扩展类。特别是，它提供了`before_flush()`、`after_flush()`和`after_flush_postexec()`方法。在许多情况下，推荐使用这种用法，而不是`MapperExtension.before_XXX`，因为在`before_flush()`中，您可以自由修改会话的刷新计划，这是无法从`MapperExtension`中完成的。

+   **AttributeExtension.** - 这个类现在是公共 API 的一部分，允许拦截属性上的用户事件，包括属性设置和删除操作，以及集合追加和删除。它还允许修改要设置或追加的值。文档中描述的`@validates`装饰器提供了一种快速的方式，将任何映射属性标记为特定类方法“验证”。

+   **Attribute Instrumentation Customization.** - 提供了一个 API，用于雄心勃勃地完全替换 SQLAlchemy 的属性检测，或者仅在某些情况下进行增强。这个 API 是为 Trellis 工具包而制作的，但作为公共 API 可用。在分发的`/examples/custom_attributes`目录中提供了一些示例。

## 模式/类型

+   **String with no length no longer generates TEXT, it generates VARCHAR** - 当未指定长度时，`String`类型不再神奇地转换为`Text`类型。这只在发出 CREATE TABLE 时才会生效，因为它将发出没有长度参数的`VARCHAR`，这在许多（但不是所有）数据库上是无效的。要创建 TEXT（或 CLOB，即无界字符串）列，请使用`Text`类型。

+   **PickleType() with mutable=True requires an __eq__() method** - 当`PickleType`类型的`mutable=True`时，需要比较值。比较`pickle.dumps()`的方法效率低下且不可靠。如果传入对象没有实现`__eq__()`，并且也不是`None`，则使用`dumps()`进行比较，但会发出警告。对于实现`__eq__()`的类型，包括所有字典、列表等，比较将使用`==`，默认情况下是可靠的。

+   **TypeEngine/TypeDecorator 的 `convert_bind_param()` 和 `convert_result_value()` 方法已移除。** - 不幸的是，O’Reilly 书籍在 0.3 之后弃用了这些方法，但仍然对其进行了文档记录。对于一个子类化 `TypeEngine` 的用户定义类型，应该使用 `bind_processor()` 和 `result_processor()` 方法进行绑定/结果处理。任何用户定义类型，无论是扩展 `TypeEngine` 还是 `TypeDecorator`，只要使用旧的 0.3 风格，都可以通过以下适配器轻松地调整为新风格：

    ```py
    class AdaptOldConvertMethods(object):
      """A mixin which adapts 0.3-style convert_bind_param and
     convert_result_value methods

     """

        def bind_processor(self, dialect):
            def convert(value):
                return self.convert_bind_param(value, dialect)

            return convert

        def result_processor(self, dialect):
            def convert(value):
                return self.convert_result_value(value, dialect)

            return convert

        def convert_result_value(self, value, dialect):
            return value

        def convert_bind_param(self, value, dialect):
            return value
    ```

    要使用上述混合项：

    ```py
    class MyType(AdaptOldConvertMethods, TypeEngine): ...
    ```

+   `Column` 和 `Table` 上的 `quote` 标志以及 `Table` 上的 `quote_schema` 标志现在控制引用方式，包括正面和负面。默认值为 `None`，表示让常规的引用规则生效。当为 `True` 时，强制引用。当为 `False` 时，强制不引用。

+   现在可以更方便地使用 `Column(..., server_default='val')` 指定列 `DEFAULT` 值的 DDL，废弃了 `Column(..., PassiveDefault('val'))`。`default=` 现在仅用于 Python 初始化的默认值，并且可以与 `server_default` 共存。新的 `server_default=FetchedValue()` 取代了标记列受外部触发器影响的 `PassiveDefault('')` 习惯用法，没有 DDL 的副作用。

+   SQLite 的 `DateTime`、`Time` 和 `Date` 类型现在**仅接受 datetime 对象，而不接受字符串**作为绑定参数输入。如果想要创建自己的“混合”类型，它接受字符串并将结果返回为日期对象（可以是任何格式），则创建一个基于 `String` 的 `TypeDecorator`。如果只想要基于字符串的日期，只需使用 `String`。

+   此外，当与 SQLite 一起使用时，`DateTime` 和 `Time` 类型现在以与 `str(datetime)` 相同的方式表示 Python `datetime.datetime` 对象的 “微秒” 字段，即作为小数秒，而不是微秒的计数。也就是说：

    ```py
    dt = datetime.datetime(2008, 6, 27, 12, 0, 0, 125)  # 125 usec

    # old way
    "2008-06-27 12:00:00.125"

    # new way
    "2008-06-27 12:00:00.000125"
    ```

    因此，如果现有的基于文件的 SQLite 数据库打算在 0.4 和 0.5 之间使用，您必须将 datetime 列升级为存储新格式（注意：请测试此功能，我相信它是正确的）：

    ```py
    UPDATE  mytable  SET  somedatecol  =
      substr(somedatecol,  0,  19)  ||  '.'  ||  substr((substr(somedatecol,  21,  -1)  /  1000000),  3,  -1);
    ```

    或者，可以按以下方式启用“传统”模式：

    ```py
    from sqlalchemy.databases.sqlite import DateTimeMixin

    DateTimeMixin.__legacy_microseconds__ = True
    ```

## 连接池默认不再是线程本地的

0.4 版本不幸默认设置为 `pool_threadlocal=True`，导致在单个线程中使用多个 Sessions 时出现意外行为。此标志在 0.5 版本中默认关闭。要重新启用 0.4 版本的行为，请在 `create_engine()` 中指定 `pool_threadlocal=True`，或者通过 `strategy="threadlocal"` 使用 “threadlocal” 策略。

## *args 被接受，*args 不再被接受

`method(\*args)` 和 `method([args])` 的策略是，如果方法接受一个表示固定结构的可变长度项集合，则采用 `\*args`。如果方法接受一个数据驱动的可变长度项集合，则采用 `[args]`。

+   各种 Query.options() 函数 `eagerload()`, `eagerload_all()`, `lazyload()`, `contains_eager()`, `defer()`, `undefer()` 现在都接受可变长度的 `\*keys` 作为参数，这允许使用描述符来制定路径，例如：

    ```py
    query.options(eagerload_all(User.orders, Order.items, Item.keywords))
    ```

    为了向后兼容性，仍然接受单个数组参数。

+   类似地，`Query.join()` 和 `Query.outerjoin()` 方法现在接受可变长度的 *args，为了向后兼容性，现在只接受单个数组：

    ```py
    query.join("orders", "items")
    query.join(User.orders, Order.items)
    ```

+   列上的 `in_()` 方法和类似方法现在只接受列表参数。不再接受 `\*args`。

## 已移除

+   **entity_name** - 这个特性一直存在问题，很少被使用。0.5 版本更深入地揭示了 `entity_name` 的问题，导致其被移除。如果需要为单个类使用不同的映射，将类拆分为单独的子类并分别映射它们。一个示例在 [wiki:UsageRecipes/EntityName] 中。有关背景的更多信息请参阅 https://groups.google.c om/group/sqlalchemy/browse_thread/thread/9e23a0641a88b96d? hl=en 。

+   **get()/load() 清理**

    `load()` 方法已被移除。它的功能有点随意，基本上是从 Hibernate 复制过来的，在那里也不是一个特别有意义的方法。

    要获得等效功能：

    ```py
    x = session.query(SomeClass).populate_existing().get(7)
    ```

    `Session.get(cls, id)` 和 `Session.load(cls, id)` 已被移除。`Session.get()` 与 `session.query(cls).get(id)` 是多余的。

    `MapperExtension.get()` 也已被移除（`MapperExtension.load()` 也是）。要覆盖 `Query.get()` 的功能，使用子类：

    ```py
    class MyQuery(Query):
        def get(self, ident): ...

    session = sessionmaker(query_cls=MyQuery)()

    ad1 = session.query(Address).get(1)
    ```

+   `sqlalchemy.orm.relation()`

    下列已弃用的关键字参数已被移除：

    foreignkey, association, private, attributeext, is_backref

    特别地，`attributeext` 被替换为 `extension` - `AttributeExtension` 类现在在公共 API 中。

+   `session.Query()`

    下列已弃用的函数已被移除：

    list, scalar, count_by, select_whereclause, get_by, select_by, join_by, selectfirst, selectone, select, execute, select_statement, select_text, join_to, join_via, selectfirst_by, selectone_by, apply_max, apply_min, apply_avg, apply_sum

    另外，`join()`、`outerjoin()`、`add_entity()` 和 `add_column()` 的 `id` 关键字参数已被移除。要将 `Query` 中的表别名定位到结果列，使用 `aliased` 构造：

    ```py
    from sqlalchemy.orm import aliased

    address_alias = aliased(Address)
    print(session.query(User, address_alias).join((address_alias, User.addresses)).all())
    ```

+   `sqlalchemy.orm.Mapper`

    +   instances()

    +   get_session() - 这个方法并不是很显著，但会将延迟加载与特定会话关联起来，即使父对象完全分离，当使用 `scoped_session()` 或旧的 `SessionContextExt` 等扩展时。有些应用程序可能依赖于这种行为，但现在可能不再按预期工作；但更好的编程实践是始终确保对象在会话中存在，如果需要从它们的属性访问数据库。

+   `mapper(MyClass, mytable)`

    映射类不再使用“c”类属性进行仪器化; 例如`MyClass.c`

+   `sqlalchemy.orm.collections`

    _prepare_instrumentation 别名 prepare_instrumentation 已被移除。

+   `sqlalchemy.orm`

    移除了`EXT_PASS`别名`EXT_CONTINUE`。

+   `sqlalchemy.engine`

    从`DefaultDialect.preexecute_sequences`到`.preexecute_pk_sequences`的别名已被移除。

    已移除了弃用的 engine_descriptors()函数。

+   `sqlalchemy.ext.activemapper`

    模块已移除。

+   `sqlalchemy.ext.assignmapper`

    模块已移除。

+   `sqlalchemy.ext.associationproxy`

    代理的`.append(item, \**kw)`上的关键字参数传递已被移除，现在只是`.append(item)`

+   `sqlalchemy.ext.selectresults`，`sqlalchemy.mods.selectresults`

    模块已移除。

+   `sqlalchemy.ext.declarative`

    `declared_synonym()`已移除。

+   `sqlalchemy.ext.sessioncontext`

    模块已移除。

+   `sqlalchemy.log`

    `SADeprecationWarning`别名`sqlalchemy.exc.SADeprecationWarning`已被移除。

+   `sqlalchemy.exc`

    `exc.AssertionError`已移除，使用被 Python 内置的同名替换。

+   `sqlalchemy.databases.mysql`

    已弃用的`get_version_info`方言方法已被移除。

## 重命名或移动

+   `sqlalchemy.exceptions`现在是`sqlalchemy.exc`

    该模块在 0.6 之前仍可使用旧名称导入。

+   `FlushError`，`ConcurrentModificationError`，`UnmappedColumnError` -> sqlalchemy.orm.exc

    这些异常已移至 orm 包。导入`sqlalchemy.orm`将在 0.6 之前为兼容性安装 sqlalchemy.exc 的别名。

+   `sqlalchemy.logging` -> `sqlalchemy.log`

    此内部模块已重命名。在使用 py2app 等扫描导入的工具打包 SA 时，不再需要特殊处理。

+   `session.Query().iterate_instances()` -> `session.Query().instances()`.

## 已弃用

+   `Session.save()`，`Session.update()`，`Session.save_or_update()`

    所有三个被`Session.add()`替换

+   `sqlalchemy.PassiveDefault`

    使用`Column(server_default=...)`在底层转换为`sqlalchemy.DefaultClause()`。

+   `session.Query().iterate_instances()`。已重命名为`instances()`.

## 主要文档更改

文档的一些部分已被完全重写，可以作为新 ORM 功能的介绍。特别是`Query`和`Session`对象在 API 和行为上有一些明显的差异，这些差异从根本上改变了许多基本操作的方式，特别是构建高度定制的 ORM 查询和处理过时的会话状态、提交和回滚。

+   [ORM 教程](https://www.sqlalchemy.org/docs/05/ormtutorial.html)

+   [会话文档](https://www.sqlalchemy.org/docs/05/session.html)

## 弃用来源

另一个信息来源在一系列单元测试中记录了一些常见`Query`模式的最新用法；此文件可在[source:sqlalchemy/trunk/test/orm/test_deprecations.py]中查看。

## 要求更改

+   Python 2.4 或更高版本是必需的。SQLAlchemy 0.4 系列是最后一个支持 Python 2.3 的版本。

## 对象关系映射

+   **Query 中的列级表达式。** - 如 [教程](https://www.sqlalchemy.org/docs/05/ormtutorial.html) 中详细说明的，`Query` 具有创建特定 SELECT 语句的能力，而不仅仅是针对完整行的语句：

    ```py
    session.query(User.name, func.count(Address.id).label("numaddresses")).join(
        Address
    ).group_by(User.name)
    ```

    任何多列/实体查询返回的元组都是*命名*元组：

    ```py
    for row in (
        session.query(User.name, func.count(Address.id).label("numaddresses"))
        .join(Address)
        .group_by(User.name)
    ):
        print("name", row.name, "number", row.numaddresses)
    ```

    `Query` 有一个 `statement` 访问器，以及一个 `subquery()` 方法，允许 `Query` 用于创建更复杂的组合：

    ```py
    subq = (
        session.query(Keyword.id.label("keyword_id"))
        .filter(Keyword.name.in_(["beans", "carrots"]))
        .subquery()
    )
    recipes = session.query(Recipe).filter(
        exists()
        .where(Recipe.id == recipe_keywords.c.recipe_id)
        .where(recipe_keywords.c.keyword_id == subq.c.keyword_id)
    )
    ```

+   **推荐使用显式 ORM 别名进行别名连接** - `aliased()` 函数生成一个类的“别名”，允许与 ORM 查询一起对别名进行精细控制。虽然仍然可以使用表级别的别名（即 `table.alias()`），但 ORM 级别的别名保留了 ORM 映射对象的语义，这对于继承映射、选项和其他场景非常重要。例如：

    ```py
    Friend = aliased(Person)
    session.query(Person, Friend).join((Friend, Person.friends)).all()
    ```

+   **query.join() 大大增强。** - 现在可以以多种方式指定连接的目标和 ON 子句。可以仅提供目标类，SQLA 将尝试通过外键以与 `table.join(someothertable)` 相同的方式与其形成连接。也可以提供目标和显式的 ON 条件，其中 ON 条件可以是 `relation()` 名称、实际类描述符或 SQL 表达式。或者仍然可以使用旧的仅 `relation()` 名称或类描述符的方式。请参阅 ORM 教程，其中有几个示例。

+   **推荐为不需要（也不偏好）表和映射器之间抽象的应用程序使用声明式** - [/docs/05/reference/ext/declarative.html 声明式] 模块，用于将 `Table`、`mapper()` 和用户定义的类对象的表达式结合在一起，强烈推荐，因为它简化了应用程序配置，确保了“每个类一个映射器”的模式，并允许对不同的 `mapper()` 调用可用的完整配置范围。现在将单独使用 `mapper()` 和 `Table` 称为“经典 SQLAlchemy 使用”，当然可以与声明式自由混合使用。

+   **已从类中删除 .c. 属性**（即 `MyClass.c.somecolumn`）。与 0.4 版本一样，类级别的属性可用作查询元素，即 `Class.c.propname` 现在被 `Class.propname` 取代，`c` 属性仍然保留在表对象上，其中它们指示表上存在的 `Column` 对象的命名空间。

    要获取映射类的表（如果您之前没有保留它）：

    ```py
    table = class_mapper(someclass).mapped_table
    ```

    通过列进行迭代：

    ```py
    for col in table.c:
        print(col)
    ```

    使用特定列：

    ```py
    table.c.somecolumn
    ```

    类绑定描述符支持完整的列运算符以及文档化的关系导向运算符，如 `has()`、`any()`、`contains()` 等。

    移除`.c.`的原因是在 0.5 中，类绑定的描述符可能具有不同的含义，以及关于类映射的信息，与普通的`Column`对象不同-并且存在一些情况下，您可能希望明确使用其中之一。通常，使用类绑定的描述符会调用一组映射/多态感知的转换，而使用表绑定的列则不会。在 0.4 中，这些转换适用于所有表达式，但是 0.5 完全区分列和映射描述符，仅对后者应用转换。因此，在许多情况下，特别是在处理连接表继承配置以及使用`query(<columns>)`时，`Class.propname`和`table.c.colname`是不可互换的。

    例如，`session.query(users.c.id, users.c.name)`与`session.query(User.id, User.name)`是不同的；在后一种情况下，`Query`知道正在使用的映射器，并且可以使用进一步的映射器特定操作，如`query.join(<propname>)`，`query.with_parent()`等，但在前一种情况下则不行。此外，在多态继承场景中，类绑定的描述符指的是多态可选择的列，而不一定是直接对应描述符的表列。例如，一组通过连接表继承到`person`表的类，每个表的`person_id`列都将其`Class.person_id`属性映射到`person`中的`person_id`列，而不是其子类表。版本 0.4 会自动将此行为映射到表绑定的`Column`对象上。在 0.5 中，这种自动转换已被移除，因此实际上*可以*使用表绑定的列来覆盖多态查询时发生的转换；这使得`Query`能够在连接表或具体表继承设置中创建优化的选择，以及可移植的子查询等。

+   **会话现在与事务自动同步。** 会话现在默认自动与事务同步，包括自动刷新和自动过期。除非使用`autocommit`选项禁用，否则始终存在事务。当所有三个标志都设置为默认值时，会话在回滚后能够优雅地恢复，并且很难将过时数据输入会话。有关详细信息，请参阅新的会话文档。

+   **隐式排序已移除**。这将影响依赖 SA 的“隐式排序”行为的 ORM 用户，该行为规定所有没有`order_by()`的 Query 对象将按照主映射表的“id”或“oid”列进行排序，并且所有延迟/急切加载的集合都会应用类似的排序。在 0.5 版本中，必须在`mapper()`和`relation()`对象上明确配置自动排序（如果需要），或者在使用`Query`时进行配置。

    要将 0.4 映射转换为 0.5，使其排序行为与 0.4 或之前的版本极为相似，请在`mapper()`和`relation()`上使用`order_by`设置：

    ```py
    mapper(
        User,
        users,
        properties={"addresses": relation(Address, order_by=addresses.c.id)},
        order_by=users.c.id,
    )
    ```

    要在`backref`上设置排序，使用`backref()`函数：

    ```py
    "keywords": relation(
        Keyword,
        secondary=item_keywords,
        order_by=keywords.c.name,
        backref=backref("items", order_by=items.c.id),
    )
    ```

    使用声明式？为了帮助新的`order_by`要求，现在可以使用稍后在 Python 中评估的字符串来设置`order_by`和相关内容（这仅适用于声明式，而不是普通映射器）：

    ```py
    class MyClass(MyDeclarativeBase):
        ...
        "addresses": relation("Address", order_by="Address.id")
    ```

    通常在加载基于列表的项目集合的`relation()`上设置`order_by`是个好主意，因为否则无法影响排序。除此之外，最佳实践是使用`Query.order_by()`来控制加载的主要实体的排序。

+   **会话现在是 autoflush=True/autoexpire=True/autocommit=False。** - 要设置它，只需调用`sessionmaker()`而不带参数。现在`transactional=True`的名称是`autocommit=False`。刷新发生在每个查询发出时（使用`autoflush=False`禁用），在每个`commit()`之前（一如既往），以及在每个`begin_nested()`之前（因此回滚到 SAVEPOINT 是有意义的）。在每个`commit()`和每个`rollback()`之后，所有对象都会过期。回滚后，待处理对象被清除，删除的对象移回持久状态。这些默认设置非常好地配合在一起，实际上不再需要像`clear()`这样的旧技术（也更名为`expunge_all()`）。

    注：在`rollback()`后，会话现在是可重用的。标量和集合属性的更改、添加和删除都会被回滚。

+   **session.add()取代 session.save()、session.update()、session.save_or_update()。** - `session.add(someitem)`和`session.add_all([list of items])`方法取代了`save()`、`update()`和`save_or_update()`。这些方法将在整个 0.5 版本中继续被弃用。

+   **backref 配置更简洁。** - 当前`backref()`函数在未明确说明时，现在使用前向`relation()`的`primaryjoin`和`secondaryjoin`参数。不再需要在两个方向分别指定`primaryjoin`/`secondaryjoin`。

+   **简化多态选项。** - ORM 的“多态加载”行为已经简化。在 0.4 版本中，`mapper()`有一个名为`polymorphic_fetch`的参数，可以配置为`select`或`deferred`。此选项已被移除；现在映射器将仅推迟未在 SELECT 语句中出现的任何列。实际使用的 SELECT 语句由`with_polymorphic`映射器参数控制（在 0.4 中也有，替换了`select_table`），以及`Query`上的`with_polymorphic()`方法（在 0.4 中也有）。

    对于继承类的延迟加载的改进是，映射器现在在所有情况下都生成“优化”版本的 SELECT 语句；也就是说，如果类 B 从 A 继承，并且在类 B 上已经过期了几个属性，则刷新操作将仅在 SELECT 语句中包含 B 的表，并且不会 JOIN 到 A。

+   `Session` 上的 `execute()` 方法将普通字符串转换为 `text()` 结构，以便可以将绑定参数全部指定为“:bindname”而不需要显式调用 `text()`。如果需要“原始”SQL，请使用 `session.connection().execute("raw text")`。

+   `session.Query().iterate_instances()` 已重命名为 `instances()`。旧的 `instances()` 方法不再返回列表而是返回迭代器。如果您依赖该行为，则应使用 `list(your_query.instances())`。

## 扩展 ORM

在 0.5 版本中，我们正在以更多的方式修改和扩展 ORM。以下是摘要：

+   **MapperExtension.** - 这是经典的扩展类，仍然存在。很少需要的方法是 `create_instance()` 和 `populate_instance()`。要控制从数据库加载对象时的初始化，请使用 `reconstruct_instance()` 方法，或者更容易地使用文档中描述的 `@reconstructor` 装饰器。

+   **SessionExtension.** - 这是一个易于使用的会话事件扩展类。特别是，它提供了 `before_flush()`、`after_flush()` 和 `after_flush_postexec()` 方法。在许多情况下，推荐使用这种用法而不是 `MapperExtension.before_XXX`，因为在 `before_flush()` 中，您可以自由地修改会话的刷新计划，而这在 `MapperExtension` 中无法做到。

+   **AttributeExtension.** - 此类现在是公共 API 的一部分，并允许拦截属性上的用户事件，包括属性设置和删除操作以及集合附加和删除操作。它还允许修改要设置或附加的值。在文档中描述的 `@validates` 装饰器提供了一种快速的方式，可以将任何映射属性标记为特定类方法“验证”的方法。

+   **属性仪器定制。** - 为了完全替换 SQLAlchemy 的属性仪器，或者仅在某些情况下对其进行增强，提供了一个 API。此 API 是为了 Trellis 工具包而制作的，但作为公共 API 提供。在分发目录中的 `/examples/custom_attributes` 中提供了一些示例。

## 架构/类型

+   **没有长度的字符串不再生成 TEXT，而是生成 VARCHAR** - 当未指定长度时，`String` 类型不再神奇地转换为 `Text` 类型。这仅在发出 CREATE TABLE 时才会生效，因为它将发出不带长度参数的 `VARCHAR`，这在许多（但不是所有）数据库上都是无效的。要创建 TEXT（或 CLOB，即无界限的字符串）列，请使用 `Text` 类型。

+   **PickleType() with mutable=True 需要一个 __eq__()方法** - `PickleType`类型在 mutable=True 时需要比较值。使用`pickle.dumps()`进行比较的方法效率低下且不可靠。如果传入对象没有实现`__eq__()`并且也不是`None`，则使用`dumps()`比较，但会发出警告。对于实现`__eq__()`的类型，包括所有字典、列表等，比较将使用`==`，现在默认情况下是可靠的。

+   **TypeEngine/TypeDecorator 的 convert_bind_param()和 convert_result_value()方法已移除。** - O’Reilly 书籍不幸地记录了这些方法，尽管它们在 0.3 版本后已被弃用。对于一个继承`TypeEngine`的用户定义类型，应该使用`bind_processor()`和`result_processor()`方法进行绑定/结果处理。任何用户定义的类型，无论是扩展`TypeEngine`还是`TypeDecorator`，只要使用旧的 0.3 风格，都可以通过以下适配器轻松地适应新风格：

    ```py
    class AdaptOldConvertMethods(object):
      """A mixin which adapts 0.3-style convert_bind_param and
     convert_result_value methods

     """

        def bind_processor(self, dialect):
            def convert(value):
                return self.convert_bind_param(value, dialect)

            return convert

        def result_processor(self, dialect):
            def convert(value):
                return self.convert_result_value(value, dialect)

            return convert

        def convert_result_value(self, value, dialect):
            return value

        def convert_bind_param(self, value, dialect):
            return value
    ```

    要使用上述 mixin：

    ```py
    class MyType(AdaptOldConvertMethods, TypeEngine): ...
    ```

+   `Column`和`Table`上的`quote`标志以及`Table`上的`quote_schema`标志现在控制引号的正面和负面。默认值为`None`，表示让常规引号规则生效。当为`True`时，强制引号。当为`False`时，强制不引号。

+   列`DEFAULT`值 DDL 现在可以更方便地使用`Column(..., server_default='val')`来指定，废弃了`Column(..., PassiveDefault('val'))`。`default=`现在专门用于 Python 发起的默认值，并且可以与 server_default 共存。一个新的`server_default=FetchedValue()`取代了`PassiveDefault('')`的习惯用法，用于标记受外部触发器影响的列，并且没有 DDL 副作用。

+   SQLite 的`DateTime`、`Time`和`Date`类型现在**只接受 datetime 对象，而不是字符串**作为绑定参数输入。如果您想创建自己的“混合”类型，接受字符串并将结果返回为日期对象（以您喜欢的任何格式），请创建一个基于`String`的`TypeDecorator`。如果您只想要基于字符串的日期，只需使用`String`。

+   此外，当与 SQLite 一起使用时，`DateTime`和`Time`类型现在以与`str(datetime)`相同的方式表示 Python `datetime.datetime`对象的“微秒”字段 - 作为分数秒，而不是微秒计数。也就是说：

    ```py
    dt = datetime.datetime(2008, 6, 27, 12, 0, 0, 125)  # 125 usec

    # old way
    "2008-06-27 12:00:00.125"

    # new way
    "2008-06-27 12:00:00.000125"
    ```

    因此，如果现有的基于文件的 SQLite 数据库打算在 0.4 和 0.5 之间使用，您必须将 datetime 列升级为存储新格式（注意：请测试这一点，我相当确定是正确的）：

    ```py
    UPDATE  mytable  SET  somedatecol  =
      substr(somedatecol,  0,  19)  ||  '.'  ||  substr((substr(somedatecol,  21,  -1)  /  1000000),  3,  -1);
    ```

    或者，按如下方式启用“legacy”模式：

    ```py
    from sqlalchemy.databases.sqlite import DateTimeMixin

    DateTimeMixin.__legacy_microseconds__ = True
    ```

## 连接池默认不再是线程本地的。

0.4 版本的默认设置`pool_threadlocal=True`导致意外行为，例如在单个线程中使用多个会话时。在 0.5 中，此标志已关闭。要重新启用 0.4 的行为，请在`create_engine()`中指定`pool_threadlocal=True`，或者通过`strategy="threadlocal"`使用“threadlocal”策略。

## *args 被接受，*args 不再被接受

`method(\*args)` vs. `method([args])` 的策略是，如果方法接受代表固定结构的可变长度项目集合，则使用`\*args`。如果方法接受数据驱动的可变长度项目集合，则使用`[args]`。

+   各种 Query.options()函数`eagerload()`，`eagerload_all()`，`lazyload()`，`contains_eager()`，`defer()`，`undefer()`现在都接受可变长度的`\*keys`作为参数，这允许使用描述符制定路径，例如：

    ```py
    query.options(eagerload_all(User.orders, Order.items, Item.keywords))
    ```

    为了向后兼容，仍然接受单个数组参数。

+   类似地，`Query.join()` 和 `Query.outerjoin()` 方法接受可变长度的*args，向后兼容仍然接受单个数组：

    ```py
    query.join("orders", "items")
    query.join(User.orders, Order.items)
    ```

+   `in_()` 方法现在只接受列表参数，不再接受`\*args`。

## 已移除

+   **entity_name** - 这个特性一直存在问题，很少被使用。0.5 更深入地揭示了`entity_name`存在的问题，因此被移除。如果需要为单个类使用不同的映射，将类拆分为单独的子类并分别映射它们。一个示例在[wiki:UsageRecipes/EntityName]中。有关原因的更多信息请参见 https://groups.google.c om/group/sqlalchemy/browse_thread/thread/9e23a0641a88b96d? hl=en 。

+   **get()/load() 清理**

    `load()` 方法已被移除。其功能有点随意，基本上是从 Hibernate 复制过来的，在那里也不是一个特别有意义的方法。

    要获得等效功能：

    ```py
    x = session.query(SomeClass).populate_existing().get(7)
    ```

    `Session.get(cls, id)` 和 `Session.load(cls, id)` 已被移除。`Session.get()` 与 `session.query(cls).get(id)` 是冗余的。

    `MapperExtension.get()` 也被移除（`MapperExtension.load()`也是）。要重写`Query.get()`的功能，使用一个子类：

    ```py
    class MyQuery(Query):
        def get(self, ident): ...

    session = sessionmaker(query_cls=MyQuery)()

    ad1 = session.query(Address).get(1)
    ```

+   `sqlalchemy.orm.relation()`

    下列已弃用的关键字参数已被移除：

    外键，关联，私有，属性扩展，is_backref

    特别是，`attributeext` 被替换为 `extension` - `AttributeExtension` 类现在在公共 API 中。

+   `session.Query()`

    下列已弃用的函数已被移除：

    列表，标量，count_by，select_whereclause，get_by，select_by，join_by，selectfirst，selectone，select，execute，select_statement，select_text，join_to，join_via，selectfirst_by，selectone_by，apply_max，apply_min，apply_avg，apply_sum

    另外，已移除了`join()`、`outerjoin()`、`add_entity()`和`add_column()`中的`id`关键字参数。要将`Query`中的目标表别名指向结果列，请使用`aliased`构造：

    ```py
    from sqlalchemy.orm import aliased

    address_alias = aliased(Address)
    print(session.query(User, address_alias).join((address_alias, User.addresses)).all())
    ```

+   `sqlalchemy.orm.Mapper`

    +   instances()

    +   get_session() - 此方法可能不太显著，但其效果是将延迟加载与特定会话关联，即使父对象完全分离，当使用`scoped_session()`或旧的`SessionContextExt`等扩展时。一些依赖于此行为的应用程序可能不再按预期工作；但���好的编程实践是始终确保对象存在于会话中，如果需要从其属性访问数据库。

+   `mapper(MyClass, mytable)`

    映射类不再使用“c”类属性进行检测；例如`MyClass.c`

+   `sqlalchemy.orm.collections`

    `_prepare_instrumentation`别名已移除。

+   `sqlalchemy.orm`

    已移除`EXT_PASS`别名的`EXT_CONTINUE`。

+   `sqlalchemy.engine`

    从`DefaultDialect.preexecute_sequences`到`.preexecute_pk_sequences`的别名已移除。

    已移除不推荐使用的`engine_descriptors()`函数。

+   `sqlalchemy.ext.activemapper`

    模块已移除。

+   `sqlalchemy.ext.assignmapper`

    模块已移除。

+   `sqlalchemy.ext.associationproxy`

    在代理的`.append(item, \**kw)`上的关键字参数传递已移除，现在简单地是`.append(item)`

+   `sqlalchemy.ext.selectresults`、`sqlalchemy.mods.selectresults`

    模块已移除。

+   `sqlalchemy.ext.declarative`

    `declared_synonym()`已移除。

+   `sqlalchemy.ext.sessioncontext`

    模块已移除。

+   `sqlalchemy.log`

    `SADeprecationWarning`别名到`sqlalchemy.exc.SADeprecationWarning`的移除。

+   `sqlalchemy.exc`

    `exc.AssertionError`已移除，使用替代名称相同的 Python 内置函数。

+   `sqlalchemy.databases.mysql`

    已移除不推荐使用的`get_version_info`方言方法。

## 已更名或移动

+   `sqlalchemy.exceptions`现在是`sqlalchemy.exc`

    该模块仍可在 0.6 版本之前使用旧名称导入。

+   `FlushError`、`ConcurrentModificationError`、`UnmappedColumnError` -> sqlalchemy.orm.exc

    这些异常已移至 orm 包。导入‘sqlalchemy.orm’将在 0.6 版本之前为兼容性安装 sqlalchemy.exc 的别名。

+   `sqlalchemy.logging` -> `sqlalchemy.log`

    此内部模块已更名。在打包 SA 与 py2app 等扫描导入工具时不再需要特殊处理。

+   `session.Query().iterate_instances()` -> `session.Query().instances()`.

## 已弃用

+   `Session.save()`、`Session.update()`、`Session.save_or_update()`

    以上三个被`Session.add()`替代

+   `sqlalchemy.PassiveDefault`

    使用`Column(server_default=...)`在底层转换为 sqlalchemy.DefaultClause()。

+   `session.Query().iterate_instances()`。已更名为`instances()`.
