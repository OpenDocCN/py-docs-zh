# ORM 内部

> 原文：[`docs.sqlalchemy.org/en/20/orm/internals.html`](https://docs.sqlalchemy.org/en/20/orm/internals.html)

重要的 ORM 构造，其他部分未涵盖，列在此处。

| 对象名称 | 描述 |
| --- | --- |
| AttributeEventToken | 在属性事件链中传播的标记。 |
| AttributeState | 提供相应于特定映射对象上的特定属性的检查接口。 |
| CascadeOptions | 跟踪发送给 `relationship.cascade` 的选项。 |
| ClassManager | 在类级别跟踪状态信息。 |
| ColumnProperty | 描述与表列或其他列表达式对应的对象属性。 |
| Composite | 用于`CompositeProperty`类的与声明兼容的前端。 |
| CompositeProperty | 定义“复合”映射属性，将一组列表示为一个属性。 |
| IdentityMap |  |
| InspectionAttr | 应用于所有 ORM 对象和属性的基类，这些对象和属性与可以由`inspect()`函数返回的内容有关。 |
| InspectionAttrExtensionType | 指示 `InspectionAttr` 所属的扩展类型的符号。 |
| InspectionAttrInfo | 向 `InspectionAttr` 添加 `.info` 属性。 |
| InstanceState | 在实例级别跟踪状态信息。 |
| InstrumentedAttribute | 用于代表`MapperProperty`对象的描述符对象的基类，以代表`MapperProperty`。实际的 `MapperProperty` 可通过 `QueryableAttribute.property` 属性访问。 |
| LoaderCallableStatus | 枚举类型。 |
| Mapped | 在映射类上表示 ORM 映射属性。 |
| MappedColumn | 将单个`Column`映射到类上。 |
| MappedSQLExpression | `ColumnProperty` 类的声明性前端。 |
| MapperProperty | 由 `Mapper` 映射的特定类属性的表示。 |
| merge_frozen_result(session, statement, frozen_result[, load]) | 将 `FrozenResult` 合并回 `Session`，返回一个带有 persistent 对象的新 `Result` 对象。 |
| merge_result(query, iterator[, load]) | 将结果合并到给定 `Query` 对象的会话中。 |
| NotExtension | 一个枚举。 |
| PropComparator | 为 ORM 映射属性定义 SQL 操作。 |
| QueryableAttribute | descriptor 对象的基类，代表 `MapperProperty` 对象的属性事件。可通过 `QueryableAttribute.property` 属性访问实际的 `MapperProperty`。 |
| QueryContext |  |
| Relationship | 描述一个对象属性，该属性保存与相关数据库表对应的单个项目或项目列表。 |
| RelationshipDirection | 枚举，指示 `RelationshipProperty` 的‘方向’。 |
| RelationshipProperty | 描述一个对象属性，该属性保存与相关数据库表对应的单个项目或项目列表。 |
| SQLORMExpression | 一个可用于指示任何 ORM 级别属性或对象的类型，以在 SQL 表达式构造的上下文中代替之。 |
| Synonym | `SynonymProperty` 类的声明性前端。 |
| SynonymProperty | 将属性名称表示为另一个属性的同义词，即该属性将镜像另一个属性的值和表达行为。 |
| UOWTransaction |  |

```py
class sqlalchemy.orm.AttributeState
```

为特定映射对象上的特定属性提供相应的检查接口。

`AttributeState`对象通过特定`InstanceState`的`InstanceState.attrs`集合访问：

**成员**

history, load_history(), loaded_value, value

```py
from sqlalchemy import inspect

insp = inspect(some_mapped_object)
attr_state = insp.attrs.some_attribute
```

```py
attribute history
```

返回此属性的当前**预刷新**更改历史记录，通过`History`接口。

如果属性的值未加载，则此方法**不会**发出加载器可调用。

注意

属性历史系统会**每次刷新**基础上跟踪更改。每次刷新`Session`时，每个属性的历史记录都会被重置为空。`Session`默认情况下会在每次调用`Query`时自动刷新。有关如何控制此行为的选项，请参见刷新。

另请参阅

`AttributeState.load_history()` - 如果值未在本地存在，则使用加载器可调用检索历史。

`get_history()` - 底层函数

```py
method load_history() → History
```

返回此属性的当前**预刷新**更改历史记录，通过`History`接口。

如果属性的值未加载，则此方法**会**发出加载器可调用。

注意

属性历史系统会**每次刷新**基础上跟踪更改。每次刷新`Session`时，每个属性的历史记录都会被重置为空。`Session`默认情况下会在每次调用`Query`时自动刷新。有关如何控制此行为的选项，请参见刷新。

另请参阅

`AttributeState.history`

`get_history()` - 底层函数

```py
attribute loaded_value
```

从数据库加载的当前属性值。

如果值尚未加载，或者在对象的字典中不存在，则返回 NO_VALUE。

```py
attribute value
```

返回此属性的值。

此操作相当于直接访问对象的属性或通过 `getattr()` 访问，并在需要时触发任何挂起的加载器可调用。

```py
class sqlalchemy.orm.CascadeOptions
```

跟踪发送到 `relationship.cascade` 的选项。

**类签名**

类 `sqlalchemy.orm.CascadeOptions` (`builtins.frozenset`, `typing.Generic`)

```py
class sqlalchemy.orm.ClassManager
```

在类级别跟踪状态信息。

**成员**

deferred_scalar_loader, expired_attribute_loader, has_parent(), manage(), state_getter(), unregister()

**类签名**

类 `sqlalchemy.orm.ClassManager` (`sqlalchemy.util.langhelpers.HasMemoized`, `builtins.dict`, `typing.Generic`, `sqlalchemy.event.registry.EventTarget`)

```py
attribute deferred_scalar_loader
```

从版本 1.4 开始已弃用：ClassManager.deferred_scalar_loader 属性现在命名为 expired_attribute_loader

```py
attribute expired_attribute_loader: _ExpiredAttributeLoaderProto
```

以前称为 deferred_scalar_loader

```py
method has_parent(state: InstanceState[_O], key: str, optimistic: bool = False) → bool
```

待办事项

```py
method manage()
```

将此实例标记为其类的管理器。

```py
method state_getter()
```

返回一个 (实例) -> InstanceState 可调用。

如果找不到实例的 InstanceState，"state getter" 可调用应引发 KeyError 或 AttributeError。

```py
method unregister() → None
```

删除此 ClassManager 建立的所有检测工具。

```py
class sqlalchemy.orm.ColumnProperty
```

描述对应于表列或其他列表达式的对象属性。

公共构造函数是 `column_property()` 函数。

**成员**

expressions, operate(), reverse_operate(), columns_to_assign, declarative_scan(), do_init(), expression, instrument_class(), mapper_property_to_assign, merge()

**类签名**

类`sqlalchemy.orm.ColumnProperty`（`sqlalchemy.orm._MapsColumns`，`sqlalchemy.orm.StrategizedProperty`，`sqlalchemy.orm._IntrospectsAnnotations`，`sqlalchemy.log.Identified`）

```py
class Comparator
```

为`ColumnProperty`属性生成布尔值、比较和其他操作符。

请参阅`PropComparator`的文档以获取简要概述。

另请参见

`PropComparator`

`ColumnOperators`

重新定义和创建新操作符

`TypeEngine.comparator_factory`

**类签名**

类`sqlalchemy.orm.ColumnProperty.Comparator`（`sqlalchemy.util.langhelpers.MemoizedSlots`，`sqlalchemy.orm.PropComparator`）

```py
attribute expressions: Sequence[NamedColumn[Any]]
```

由此引用的列的完整序列

属性，根据正在进行的任何别名调整。

版本 1.3.17 中的新功能。

另请参见

将类映射到多个表 - 用法示例

```py
method operate(op: OperatorType, *other: Any, **kwargs: Any) → ColumnElement[Any]
```

对参数执行操作。

这是操作的最低级别，默认情况下引发`NotImplementedError`。

在子类上重写这个方法可以使通用行为应用于所有操作。例如，重写`ColumnOperators`以将`func.lower()`应用于左右两侧：

```py
class MyComparator(ColumnOperators):
    def operate(self, op, other, **kwargs):
        return op(func.lower(self), func.lower(other), **kwargs)
```

参数：

+   `op` – 操作符可调用。

+   `*other` – 操作的‘other’一侧。对于大多数操作，将是单个标量。

+   `**kwargs` – 修饰符。这些可以通过特殊操作符（如`ColumnOperators.contains()`）传递。

```py
method reverse_operate(op: OperatorType, other: Any, **kwargs: Any) → ColumnElement[Any]
```

对参数执行反向操作。

用法与`operate()`相同。

```py
attribute columns_to_assign
```

```py
method declarative_scan(decl_scan: _ClassScanMapperConfig, registry: _RegistryType, cls: Type[Any], originating_module: str | None, key: str, mapped_container: Type[Mapped[Any]] | None, annotation: _AnnotationScanType | None, extracted_mapped_annotation: _AnnotationScanType | None, is_dataclass_field: bool) → None
```

在早期声明扫描时执行类特定的初始化。

版本 2.0 中的新功能。

```py
method do_init() → None
```

在映射创建后执行子类特定的初始化步骤。

这是由`MapperProperty`对象的 init()方法调用的模板方法。

```py
attribute expression
```

返回此 ColumnProperty 的主列或表达式。

例如：

```py
class File(Base):
    # ...

    name = Column(String(64))
    extension = Column(String(8))
    filename = column_property(name + '.' + extension)
    path = column_property('C:/' + filename.expression)
```

另请参见

在映射时从列属性组合

```py
method instrument_class(mapper: Mapper[Any]) → None
```

Mapper 调用的钩子，用于初始化由此 MapperProperty 管理的类属性的仪器化。

���里的`MapperProperty`通常会调用属性模块以设置`InstrumentedAttribute`。

这一步是设置`InstrumentedAttribute`的两个步骤中的第一步，并且在映射器设置过程的早期调用。

第二步通常是`init_class_attribute`步骤，通过`StrategizedProperty`通过`post_instrument_class()`钩子调用。此步骤为`InstrumentedAttribute`分配附加状态（特别是“impl”），该状态在`MapperProperty`确定需要执行什么类型的持久性管理后确定（例如标量，对象，集合等）。

```py
attribute mapper_property_to_assign
```

```py
method merge(session: Session, source_state: InstanceState[Any], source_dict: _InstanceDict, dest_state: InstanceState[Any], dest_dict: _InstanceDict, load: bool, _recursive: Dict[Any, object], _resolve_conflict_map: Dict[_IdentityKeyType[Any], object]) → None
```

将此`MapperProperty`表示的属性从源对象合并到目标对象。

```py
class sqlalchemy.orm.Composite
```

与`CompositeProperty`类兼容的声明性前端。

公共构造函数是`composite()`函数。

在 2.0 版本中更改：将`Composite`添加为`CompositeProperty`的声明兼容子类。

另请参见

复合列类型

**类签名**

类`sqlalchemy.orm.Composite`（`sqlalchemy.orm.descriptor_props.CompositeProperty`，`sqlalchemy.orm.base._DeclarativeMapped`）

```py
class sqlalchemy.orm.CompositeProperty
```

定义一个“复合”映射属性，表示一组列作为一个属性。

`CompositeProperty`是使用`composite()`函数构建的。

另请参见

复合列类型

**成员**

create_row_processor()，columns_to_assign，declarative_scan()，do_init()，get_history()，instrument_class()，mapper_property_to_assign

**类签名**

类`sqlalchemy.orm.CompositeProperty`（`sqlalchemy.orm._MapsColumns`，`sqlalchemy.orm._IntrospectsAnnotations`，`sqlalchemy.orm.descriptor_props.DescriptorProperty`）

```py
class Comparator
```

为`Composite`属性生成布尔值，比较和其他运算符。

请参见 Redefining Comparison Operations for Composites 中的示例，以了解用法概述，以及`PropComparator`的文档。

请参见

`PropComparator`

`ColumnOperators`

重新定义和创建新的操作符

`TypeEngine.comparator_factory`

**类签名**

类`sqlalchemy.orm.CompositeProperty.Comparator`（`sqlalchemy.orm.PropComparator`）

```py
class CompositeBundle
```

**类签名**

类`sqlalchemy.orm.CompositeProperty.CompositeBundle`（`sqlalchemy.orm.Bundle`）

```py
method create_row_processor(query: Select[Any], procs: Sequence[Callable[[Row[Any]], Any]], labels: Sequence[str]) → Callable[[Row[Any]], Any]
```

为此`Bundle`生成“行处理”函数。

可以被子类重写以在提取结果时提供自定义行为。该方法在查询执行时传递了语句对象和一组“行处理”函数；当给定一个结果行时，这些处理函数将返回单个属性值，然后可以将其调整为任何类型的返回数据结构。

下面的示例说明了将通常的`Row`返回结构替换为直接的 Python 字典：

```py
from sqlalchemy.orm import Bundle

class DictBundle(Bundle):
    def create_row_processor(self, query, procs, labels):
        'Override create_row_processor to return values as
        dictionaries'

        def proc(row):
            return dict(
                zip(labels, (proc(row) for proc in procs))
            )
        return proc
```

上述`Bundle`的结果将返回字典值：

```py
bn = DictBundle('mybundle', MyClass.data1, MyClass.data2)
for row in session.execute(select(bn)).where(bn.c.data1 == 'd1'):
    print(row.mybundle['data1'], row.mybundle['data2'])
```

```py
attribute columns_to_assign
```

```py
method declarative_scan(decl_scan: _ClassScanMapperConfig, registry: _RegistryType, cls: Type[Any], originating_module: str | None, key: str, mapped_container: Type[Mapped[Any]] | None, annotation: _AnnotationScanType | None, extracted_mapped_annotation: _AnnotationScanType | None, is_dataclass_field: bool) → None
```

在早期的声明扫描时执行类特定的初始化。

自 2.0 版开始。

```py
method do_init() → None
```

在`Composite`与其父 Mapper 关联之后发生的初始化。

```py
method get_history(state: InstanceState[Any], dict_: _InstanceDict, passive: PassiveFlag = symbol('PASSIVE_OFF')) → History
```

为使用`attributes.get_history()`的用户代码提供。

```py
method instrument_class(mapper: Mapper[Any]) → None
```

由 Mapper 调用的钩子，用于启动由此 MapperProperty 管理的类属性的检测。

这里的 MapperProperty 通常会调用 attributes 模块来设置一个 InstrumentedAttribute。

这一步是设置 InstrumentedAttribute 的两个步骤中的第一个步骤，并且在 Mapper 设置过程中的早期阶段调用。

第二步通常是 init_class_attribute 步骤，通过 post_instrument_class()钩子从 StrategizedProperty 调用。此步骤为 InstrumentedAttribute 分配了附加状态（特别是“impl”），该状态在 MapperProperty 确定需要执行什么类型的持久性管理之后确定。

```py
attribute mapper_property_to_assign
```

```py
class sqlalchemy.orm.AttributeEventToken
```

在一系列属性事件中传播的令牌。

用作事件源的指示器，还提供了一种控制在一系列属性操作中传播的方法。

处理事件时，`Event`对象作为`initiator`参数发送，例如处理`AttributeEvents.append()`、`AttributeEvents.set()`和`AttributeEvents.remove()`等事件。

`Event`对象当前由反向引用事件处理程序解释，并用于控制操作在两个相互依赖属性之间的传播。

从版本 2.0 开始更改：将名称从`AttributeEvent`更改为`AttributeEventToken`。

属性实现：

当前事件发起者的`AttributeImpl`。

属性操作：

符号`OP_APPEND`、`OP_REMOVE`、`OP_REPLACE`或`OP_BULK_REPLACE`，指示源操作。

```py
class sqlalchemy.orm.IdentityMap
```

**成员**

check_modified()

```py
method check_modified() → bool
```

如果存在任何已标记为“修改”的 InstanceState，则返回 True。

```py
class sqlalchemy.orm.InspectionAttr
```

应用于所有与可以由`inspect()`函数返回的对象相关的 ORM 对象和属性的基类。

此处定义的属性允许使用简单的布尔检查来测试有关返回对象的基本事实。

**成员**

extension_type, is_aliased_class, is_attribute, is_bundle, is_clause_element, is_instance, is_mapper, is_property, is_selectable

虽然这里的布尔检查基本上与使用 Python 的 isinstance()函数相同，但这里的标志可以在不需要导入所有这些类的情况下使用，并且 SQLAlchemy 类系统可以更改，同时保持这里的标志不变以实现向前兼容性。

```py
attribute extension_type: InspectionAttrExtensionType = 'not_extension'
```

扩展类型，如果有的话。默认为`NotExtension.NOT_EXTENSION`。

另请参阅

`HybridExtensionType`

`AssociationProxyExtensionType`

```py
attribute is_aliased_class = False
```

如果此对象是`AliasedClass`的实例，则返回 True。

```py
attribute is_attribute = False
```

如果此对象是 Python 的描述符的实例，则返回 True。

这可以指代许多类型之一。通常是一个`QueryableAttribute`，它代表一个`MapperProperty`处理属性事件。但也可以是一个扩展类型，如`AssociationProxy`或`hybrid_property`。`InspectionAttr.extension_type`将指代一个标识特定子类型的常量。

另请参阅

`Mapper.all_orm_descriptors`

```py
attribute is_bundle = False
```

如果此对象是`Bundle`的实例，则返回 True。

```py
attribute is_clause_element = False
```

如果此对象是`ClauseElement`的实例，则返回 True。

```py
attribute is_instance = False
```

如果此对象是`InstanceState`的实例，则返回 True。

```py
attribute is_mapper = False
```

如果此对象是`Mapper`的实例，则返回 True。

```py
attribute is_property = False
```

如果此对象是`MapperProperty`的实例，则返回 True。

```py
attribute is_selectable = False
```

如果此对象是`Selectable`的实例，则返回 True。

```py
class sqlalchemy.orm.InspectionAttrInfo
```

将`.info`属性添加到`InspectionAttr`。

`InspectionAttr`与`InspectionAttrInfo`之间的理由是前者兼容作为指定`__slots__`的类的 mixin；这本质上是一种实现工件。

**成员**

info

**类签名**

类 `sqlalchemy.orm.InspectionAttrInfo`（`sqlalchemy.orm.base.InspectionAttr`）

```py
attribute info
```

与对象关联的信息字典，允许将用户定义的数据与此 `InspectionAttr` 关联。

字典在首次访问时生成。或者，它可以作为构造函数参数指定给 `column_property()`、`relationship()` 或 `composite()` 函数。

另请参阅

`QueryableAttribute.info`

`SchemaItem.info`

```py
class sqlalchemy.orm.InstanceState
```

在实例级别跟踪状态信息。

`InstanceState` 是 SQLAlchemy ORM 中用于跟踪对象状态的关键对象；它在对象实例化时创建，通常是作为 SQLAlchemy 应用于类的 `__init__()` 方法的 instrumentation 的结果。

`InstanceState` 也是一个半公开对象，可用于运行时检查映射实例的状态，包括其在特定 `Session` 中的当前状态以及有关各个属性的数据的详细信息。获取 `InstanceState` 对象的公共 API 是使用 `inspect()` 系统：

```py
>>> from sqlalchemy import inspect
>>> insp = inspect(some_mapped_object)
>>> insp.attrs.nickname.history
History(added=['new nickname'], unchanged=(), deleted=['nickname'])
```

另请参阅

映射实例的检查

**成员**

async_session, attrs, callables, deleted, detached, dict, expired_attributes, has_identity, identity, identity_key, is_instance, mapper, object, pending, persistent, session, transient, unloaded, unloaded_expirable, unmodified, unmodified_intersection(), was_deleted

**类签名**

类 `sqlalchemy.orm.InstanceState` (`sqlalchemy.orm.base.InspectionAttrInfo`, `typing.Generic`)

```py
attribute async_session
```

返回此实例的拥有 `AsyncSession`，如果没有可用，则返回 `None`。

仅当此 ORM 对象使用 `sqlalchemy.ext.asyncio` API 时，此属性才不为 `None`。返回的 `AsyncSession` 对象将是一个代理，用于表示此 `InstanceState` 的 `InstanceState.session` 属性将返回的 `Session` 对象。

版本 1.4.18 中的新功能。

另请参阅

异步 I/O（asyncio）

```py
attribute attrs
```

返回一个表示映射对象上每个属性的命名空间，包括其当前值和历史记录。

返回的对象是 `AttributeState` 的实例。此对象允许检查属性内的当前数据以及自上次刷新以来的属性历史记录。

```py
attribute callables: Dict[str, Callable[[InstanceState[_O], PassiveFlag], Any]] = {}
```

可以关联每个状态加载器可调用的命名空间。

在 SQLAlchemy 1.0 中，这仅用于通过查询选项设置的延迟加载器/延迟加载器。

以前，可调用函数还用于通过在此字典中存储与 InstanceState 本身的链接来指示过期属性。现在，这个角色由 expired_attributes 集合处理。

```py
attribute deleted
```

如果对象已被删除，则返回`True`。

处于删除状态的对象保证不在其父`Session`的`Session.identity_map` 中；但是如果会话的事务被回滚，对象将被恢复到持久状态和标识映射。

注意

`InstanceState.deleted` 属性指的是对象在“持久”状态和“分离”状态之间发生的特定状态；一旦对象被分离，`InstanceState.deleted` 属性**不再返回 True**；为了检测状态是否已删除，无论对象是否与`Session`相关联，都可以使用`InstanceState.was_deleted` 访问器。

另请参见

对象状态简介

```py
attribute detached
```

如果对象是分离，则返回`True`。

另请参见

对象状态简介

```py
attribute dict
```

返回对象使用的实例字典。

在正常情况下，这通常与映射对象的`__dict__`属性同义，除非已配置了替代的仪器系统。

如果实际对象已经被垃圾回收，此访问器将返回一个空字典。

```py
attribute expired_attributes: Set[str]
```

由管理器的延迟标量加载器加载的‘过期’键集合，假设没有挂起的更改。

还请参见在刷新操作发生时与此集合相交的`unmodified`集合。

```py
attribute has_identity
```

如果此对象具有标识键，则返回`True`。

这应始终具有与表达式 `state.persistent` 或 `state.detached` 相同的值。

```py
attribute identity
```

返回映射对象的映射标识。这是 ORM 持久化的主键标识，始终可以直接传递给`Query.get()`。

如果对象没有主键标识，则返回`None`。

注意

对象在刷新之前是瞬态或挂起的情况下，**没有**映射的标识，即使其属性包括主键值。

```py
attribute identity_key
```

返回映射对象的标识键。

这是用于在`Session.identity_map`映射中定位对象的键。它包含由`identity`返回的标识。

```py
attribute is_instance: bool = True
```

如果此对象是`InstanceState`的实例，则返回`True`。

```py
attribute mapper
```

返回用于此映射对象的`Mapper`。

```py
attribute object
```

返回由此`InstanceState`表示的映射对象。

如果对象已被垃圾收集，则返回`None`。

```py
attribute pending
```

如果对象是挂起的，则返回`True`。

另请参阅

对象状态简介

```py
attribute persistent
```

如果对象是持久的，则返回`True`。

处于持久状态的对象保证位于其父`Session`的`Session.identity_map`中。

另请参阅

对象状态简介

```py
attribute session
```

返回此实例的拥有`Session`，如果没有可用的则返回`None`。

注意，此处的结果在某些情况下可能与`obj in session`的结果*不同*；已删除的对象将报告为不在`session`中，但是如果事务仍在进行中，则此属性仍将指向该会话。通常情况下，只有在事务完成时，对象才会完全分离。

另请参阅

`InstanceState.async_session`

```py
attribute transient
```

如果对象是瞬时的，则返回`True`。

另请参阅

对象状态简介

```py
attribute unloaded
```

返回不具有加载值的键的集合。

这包括已过期的属性和任何未填充或未修改的属性。

```py
attribute unloaded_expirable
```

与`InstanceState.unloaded`同义。

自版本 2.0 起已弃用：`InstanceState.unloaded_expirable`属性已弃用。请使用`InstanceState.unloaded`。

此属性在某个时候添加为实现特定的细节，并且应被视为私有。

```py
attribute unmodified
```

返回没有未提交更改的键的集合。

```py
method unmodified_intersection(keys: Iterable[str]) → Set[str]
```

返回 self.unmodified.intersection(keys)。

```py
attribute was_deleted
```

如果此对象处于“已删除”状态或先前处于“已删除”状态，并且未恢复为持久状态，则返回 True。

该标志一旦对象在刷新时被删除就会返回 True。当对象被从会话中显式地删除或通过事务提交并进入“分离”状态时，此标志将继续报告 True。

请参阅

`InstanceState.deleted` - 指的是“已删除”状态

`was_deleted()` - 独立函数

对象状态简介

```py
class sqlalchemy.orm.InstrumentedAttribute
```

用于代表`MapperProperty`对象拦截属性事件的描述符对象的基类。实际的`MapperProperty`可通过`QueryableAttribute.property`属性访问。

请参阅

`InstrumentedAttribute`

`MapperProperty`

`Mapper.all_orm_descriptors`

`Mapper.attrs`

**类签名**

class `sqlalchemy.orm.InstrumentedAttribute` (`sqlalchemy.orm.QueryableAttribute`)

```py
class sqlalchemy.orm.LoaderCallableStatus
```

一个枚举。

**成员**

ATTR_EMPTY, ATTR_WAS_SET, NEVER_SET, NO_VALUE, PASSIVE_CLASS_MISMATCH, PASSIVE_NO_RESULT

**类签名**

class `sqlalchemy.orm.LoaderCallableStatus` (`enum.Enum`)

```py
attribute ATTR_EMPTY = 3
```

用于内部表示属性没有可调用。

```py
attribute ATTR_WAS_SET = 2
```

由加载器可调用返回的符号，表示检索到的值或值已分配给目标对象上的属性。

```py
attribute NEVER_SET = 4
```

与 NO_VALUE 同义

从 1.4 版本开始更改：NEVER_SET 已与 NO_VALUE 合并

```py
attribute NO_VALUE = 4
```

符号，可放置为属性的“前一个”值，表示修改属性时未加载任何值，并且标志指示我们不加载它。

```py
attribute PASSIVE_CLASS_MISMATCH = 1
```

表示对象在给定的主键标识下本地存在，但它不是请求的类。因此，返回值为 None，不应发出任何 SQL。

```py
attribute PASSIVE_NO_RESULT = 0
```

当值无法确定时，由加载器可调用或其他属性/历史检索操作返回的符号，基于加载器可调用标志。

```py
class sqlalchemy.orm.Mapped
```

在映射类上表示 ORM 映射属性。

该类表示任何将由 ORM `Mapper`类检测的类属性的完整描述符接口。为类型检查器（如 pylance 和 mypy）提供适当的信息，以便正确对 ORM 映射属性进行类型化。

`Mapped`最突出的用途是在声明式映射形式的`Mapper`配置中，当显式使用时，它驱动 ORM 属性（如`mapped_class()`和`relationship()`）的配置。

另请参见

使用声明式基类

使用`mapped_column()`声明式表

提示

`Mapped`类表示由`Mapper`类直接处理的属性。它不包括其他作为扩展提供的 Python 描述符类，包括混合属性和关联代理。虽然这些系统仍然使用 ORM 特定的超类和结构，但当它们在类上被访问时，它们不会被`Mapper`所检测，而是在访问时提供自己的功能。

版本 1.4 中的新功能。

**类签名**

类`sqlalchemy.orm.Mapped`（`sqlalchemy.orm.base.SQLORMExpression`，`sqlalchemy.orm.base.ORMDescriptor`，`sqlalchemy.orm.base._MappedAnnotationBase`，`sqlalchemy.sql.roles.DDLConstraintColumnRole`)

```py
class sqlalchemy.orm.MappedColumn
```

在类上映射单个`Column`。

`MappedColumn`是`ColumnProperty`类的一个特化，面向声明式配置。

要构建`MappedColumn`对象，请使用`mapped_column()`构造函数。

版本 2.0 中的新功能。

**类签名**

class `sqlalchemy.orm.MappedColumn` (`sqlalchemy.orm._IntrospectsAnnotations`, `sqlalchemy.orm._MapsColumns`, `sqlalchemy.orm.base._DeclarativeMapped`)

```py
class sqlalchemy.orm.MapperProperty
```

表示由`Mapper`映射的特定类属性。

`MapperProperty`最常见的出现是映射为`ColumnProperty`实例的映射`Column`，以及由`relationship()`生成的对另一个类的引用，表示为`Relationship`实例。

**成员**

cascade_iterator(), class_attribute, comparator, create_row_processor(), do_init(), doc, info, init(), instrument_class(), is_property, key, merge(), parent, post_instrument_class(), set_parent(), setup()

**类签名**

class `sqlalchemy.orm.MapperProperty` (`sqlalchemy.sql.cache_key.HasCacheKey`, `sqlalchemy.orm._DCAttributeOptions`, `sqlalchemy.orm.base._MappedAttribute`, `sqlalchemy.orm.base.InspectionAttrInfo`, `sqlalchemy.util.langhelpers.MemoizedSlots`)

```py
method cascade_iterator(type_: str, state: InstanceState[Any], dict_: _InstanceDict, visited_states: Set[InstanceState[Any]], halt_on: Callable[[InstanceState[Any]], bool] | None = None) → Iterator[Tuple[object, Mapper[Any], InstanceState[Any], _InstanceDict]]
```

迭代与特定“cascade”相关的给定实例的实例，从这个 MapperProperty 开始。

返回一个迭代器 3 元组（实例，映射器，状态）。

注意，在调用 cascade_iterator 之前，首先检查此 MapperProperty 上的“cascade”集合是否适用于给定类型。

这个方法通常只适用于关系（Relationship）。

```py
attribute class_attribute
```

返回与此`MapperProperty`对应的类绑定描述符。

这基本上是一个`getattr()`调用：

```py
return getattr(self.parent.class_, self.key)
```

即，如果此`MapperProperty`被命名为`addresses`，并且将其映射到的类是`User`，则此序列是可能的：

```py
>>> from sqlalchemy import inspect
>>> mapper = inspect(User)
>>> addresses_property = mapper.attrs.addresses
>>> addresses_property.class_attribute is User.addresses
True
>>> User.addresses.property is addresses_property
True
```

```py
attribute comparator: PropComparator[_T]
```

实现此映射属性的 SQL 表达式构造的`PropComparator`实例。

```py
method create_row_processor(context: ORMCompileState, query_entity: _MapperEntity, path: AbstractEntityRegistry, mapper: Mapper[Any], result: Result[Any], adapter: ORMAdapter | None, populators: _PopulatorDict) → None
```

生成行处理函数并附加到给定的填充器列表。

```py
method do_init() → None
```

执行子类特定的初始化后映射器创建步骤。

这是由`MapperProperty`对象的`init()`方法调用的模板方法。

```py
attribute doc: str | None
```

可选的文档字符串

```py
attribute info: _InfoType
```

与对象关联的信息字典，允许将用户定义的数据与此`InspectionAttr`相关联。

第一次访问时生成字典。或者，它可以作为构造函数参数指定给`column_property()`、`relationship()`或`composite()`函数。

另请参阅

`QueryableAttribute.info`

`SchemaItem.info`

```py
method init() → None
```

在创建所有映射器后调用，以组装映射器之间的关系并执行其他后映射器创建初始化步骤。

```py
method instrument_class(mapper: Mapper[Any]) → None
```

由映射器调用到属性的挂钩，以启动由此`MapperProperty`管理的类属性的仪器化。

此处的`MapperProperty`通常会调用属性模块以设置`InstrumentedAttribute`。

这一步是设置`InstrumentedAttribute`的两个步骤中的第一个步骤，并在映射器设置过程中的早期阶段调用。

第二步通常是`init_class_attribute`步骤，通过`post_instrument_class()`挂钩从`StrategizedProperty`调用。此步骤分配了额外的状态给`InstrumentedAttribute`（具体为“impl”），该状态在`MapperProperty`确定其需要执行的持久性管理类型（例如标量、对象、集合等）后确定。

```py
attribute is_property = True
```

InspectionAttr 接口的一部分；说明此对象是一个映射器属性。

```py
attribute key: str
```

类属性的名称

```py
method merge(session: Session, source_state: InstanceState[Any], source_dict: _InstanceDict, dest_state: InstanceState[Any], dest_dict: _InstanceDict, load: bool, _recursive: Dict[Any, object], _resolve_conflict_map: Dict[_IdentityKeyType[Any], object]) → None
```

将此`MapperProperty`表示的属性从源对象合并到目标对象。

```py
attribute parent: Mapper[Any]
```

管理此属性的`Mapper`。

```py
method post_instrument_class(mapper: Mapper[Any]) → None
```

在`init()`完成后需要进行的仪器化调整。

给定的`Mapper`是调用操作的`Mapper`，这可能不是相同的`Mapper`作为继承场景中的`self.parent`的`Mapper`；然而，`Mapper`将始终至少是`self.parent`的子映射器。

此方法通常由 StrategizedProperty 使用，后者将其委派给 LoaderStrategy.init_class_attribute() 以在类绑定的 InstrumentedAttribute 上执行最终设置。

```py
method set_parent(parent: Mapper[Any], init: bool) → None
```

设置引用此 MapperProperty 的父映射器。

某些子类重写此方法以在首次了解映射器时执行额外的设置。

```py
method setup(context: ORMCompileState, query_entity: _MapperEntity, path: AbstractEntityRegistry, adapter: ORMAdapter | None, **kwargs: Any) → None
```

由 Query 调用，用于构造 SQL 语句。

与目标映射器关联的每个 MapperProperty 处理查询上下文引用的语句，根据需要添加列和/或条件。

```py
class sqlalchemy.orm.MappedSQLExpression
```

`ColumnProperty` 类的声明式前端。

公共构造函数是 `column_property()` 函数。

在 2.0 版本中更改：将 `MappedSQLExpression` 添加为 `ColumnProperty` 的声明式兼容子类。

另请参见

`MappedColumn`

**类签名**

类 `sqlalchemy.orm.MappedSQLExpression` (`sqlalchemy.orm.properties.ColumnProperty`, `sqlalchemy.orm.base._DeclarativeMapped`)

```py
class sqlalchemy.orm.InspectionAttrExtensionType
```

表示 `InspectionAttr` 所属扩展类型的符号。

**类签名**

类 `sqlalchemy.orm.InspectionAttrExtensionType` (`enum.Enum`)

```py
class sqlalchemy.orm.NotExtension
```

一个枚举。

**成员**

NOT_EXTENSION

**类签名**

类 `sqlalchemy.orm.NotExtension` (`sqlalchemy.orm.base.InspectionAttrExtensionType`)

```py
attribute NOT_EXTENSION = 'not_extension'
```

表示 `InspectionAttr` 不是 sqlalchemy.ext 的一部分的符号。

被赋给 `InspectionAttr.extension_type` 属性。

```py
function sqlalchemy.orm.merge_result(query: Query[Any], iterator: FrozenResult | Iterable[Sequence[Any]] | Iterable[object], load: bool = True) → FrozenResult | Iterable[Any]
```

将结果合并到给定的 `Query` 对象的会话中。

自 2.0 版本起弃用：`merge_result()`函数在 SQLAlchemy 1.x 系列中被视为遗留函数，并在 2.0 版中成为遗留结构。该函数以及`Query`上的方法被`merge_frozen_result()`函数取代。（有关 SQLAlchemy 2.0 的背景，请参见：SQLAlchemy 2.0 - 主要迁移指南）

有关此函数的顶层文档，请参见`Query.merge_result()`。

```py
function sqlalchemy.orm.merge_frozen_result(session, statement, frozen_result, load=True)
```

将`FrozenResult`合并回`Session`，返回一个新的`Result`对象，其中包含持久化对象。

有关示例，请参见重新执行语句部分。

另请参见

重新执行语句

`Result.freeze()`

`FrozenResult`

```py
class sqlalchemy.orm.PropComparator
```

定义 ORM 映射属性的 SQL 操作。

SQLAlchemy 允许在核心和 ORM 级别重新定义运算符。`PropComparator`是 ORM 级别操作重新定义的基类，包括`ColumnProperty`、`Relationship`和`Composite`的操作。

可以创建`PropComparator`的用户定义子类。可以重写内置的 Python 比较和数学运算符方法，如`ColumnOperators.__eq__()`，`ColumnOperators.__lt__()`和`ColumnOperators.__add__()`，以提供新的操作行为。定制的`PropComparator`通过`comparator_factory`参数传递给`MapperProperty`实例。在每种情况下，应使用适当的`PropComparator`子类：

```py
# definition of custom PropComparator subclasses

from sqlalchemy.orm.properties import \
                        ColumnProperty,\
                        Composite,\
                        Relationship

class MyColumnComparator(ColumnProperty.Comparator):
    def __eq__(self, other):
        return self.__clause_element__() == other

class MyRelationshipComparator(Relationship.Comparator):
    def any(self, expression):
        "define the 'any' operation"
        # ...

class MyCompositeComparator(Composite.Comparator):
    def __gt__(self, other):
        "redefine the 'greater than' operation"

        return sql.and_(*[a>b for a, b in
                          zip(self.__clause_element__().clauses,
                              other.__composite_values__())])

# application of custom PropComparator subclasses

from sqlalchemy.orm import column_property, relationship, composite
from sqlalchemy import Column, String

class SomeMappedClass(Base):
    some_column = column_property(Column("some_column", String),
                        comparator_factory=MyColumnComparator)

    some_relationship = relationship(SomeOtherClass,
                        comparator_factory=MyRelationshipComparator)

    some_composite = composite(
            Column("a", String), Column("b", String),
            comparator_factory=MyCompositeComparator
        )
```

请注意，对于列级操作符的重新定义，通常更简单的方法是在核心级别定义操作符，使用`TypeEngine.comparator_factory`属性。有关更多详细信息，请参阅重新定义和创建新操作符。

另请参阅

`比较器`

`比较器`

`比较器`

`ColumnOperators`

重新定义和创建新操作符

`TypeEngine.comparator_factory`

**成员**

__eq__(), __le__(), __lt__(), __ne__(), adapt_to_entity(), adapter, all_(), and_(), any(), any_(), asc(), between(), bitwise_and(), bitwise_lshift(), bitwise_not(), bitwise_or(), bitwise_rshift(), bitwise_xor(), bool_op(), collate(), concat(), contains(), desc(), distinct(), endswith(), has(), icontains(), iendswith(), ilike(), in_(), is_(), is_distinct_from(), is_not(), is_not_distinct_from(), isnot(), isnot_distinct_from(), istartswith(), like(), match(), not_ilike(), not_in(), not_like(), notilike(), notin_(), notlike(), nulls_first(), nulls_last(), nullsfirst(), nullslast(), of_type(), op(), operate(), property, regexp_match(), regexp_replace(), reverse_operate(), startswith(), timetuple

**类签名**

类 `sqlalchemy.orm.PropComparator` (`sqlalchemy.orm.base.SQLORMOperations`, `typing.Generic`, `sqlalchemy.sql.expression.ColumnOperators`)

```py
method __eq__(other: Any) → ColumnOperators
```

*从* `ColumnOperators` *的* `sqlalchemy.sql.expression.ColumnOperators.__eq__` *方法继承*

实现 `==` 运算符。

在列上下文中，生成子句 `a = b`。如果目标是 `None`，则生成 `a IS NULL`。

```py
method __le__(other: Any) → ColumnOperators
```

*从* `ColumnOperators` *的* `sqlalchemy.sql.expression.ColumnOperators.__le__` *方法继承*

实现 `<=` 运算符。

在列上下文中，生成子句 `a <= b`。

```py
method __lt__(other: Any) → ColumnOperators
```

*从* `ColumnOperators` *的* `sqlalchemy.sql.expression.ColumnOperators.__lt__` *方法继承*

实现 `<` 运算符。

在列上下文中，生成子句 `a < b`。

```py
method __ne__(other: Any) → ColumnOperators
```

*从* `ColumnOperators` *的* `sqlalchemy.sql.expression.ColumnOperators.__ne__` *方法继承*

实现 `!=` 运算符。

在列上下文中，生成子句 `a != b`。如果目标是 `None`，则生成 `a IS NOT NULL`。

```py
method adapt_to_entity(adapt_to_entity: AliasedInsp[Any]) → PropComparator[_T_co]
```

返回此 `PropComparator` 的副本，将使用给定的`AliasedInsp` 来生成相应的表达式。

```py
attribute adapter
```

生成一个可调用对象，以使列表达式适合此比较器的别名版本。

```py
method all_() → ColumnOperators
```

*从* `ColumnOperators` *的* `ColumnOperators.all_()` *方法继承*

对父对象产生一个 `all_()` 子句。

参见 `all_()` 的文档以获取示例。

注意

请务必不要将新的`ColumnOperators.all_()`方法与此方法的**传统**版本混淆，后者是专用于`ARRAY`的方法，采用不同的调用风格，`Comparator.all()`方法。

```py
method and_(*criteria: _ColumnExpressionArgument[bool]) → PropComparator[bool]
```

向由此关系属性表示的 ON 子句添加额外的条件。

例如：

```py
stmt = select(User).join(
    User.addresses.and_(Address.email_address != 'foo')
)

stmt = select(User).options(
    joinedload(User.addresses.and_(Address.email_address != 'foo'))
)
```

版本 1.4 中的新功能。

请参阅

将 Relationship 与自定义 ON 条件结合

向加载器选项添加条件

`with_loader_criteria()`

```py
method any(criterion: _ColumnExpressionArgument[bool] | None = None, **kwargs: Any) → ColumnElement[bool]
```

返回一个 SQL 表达式，如果此元素引用满足给定条件的成员，则表示为真。

`any()`的常规实现是`Comparator.any()`。

参数：

+   `criterion` – 针对成员类表或属性制定的可选 ClauseElement。

+   `**kwargs` – 键/值对应于成员类属性名称，这些属性将通过等式与相应的值进行比较。

```py
method any_() → ColumnOperators
```

*继承自* `ColumnOperators.any_()` *方法的* `ColumnOperators`

对父对象生成一个`any_()`子句。

请参阅 `any_()` 的文档以获取示例。

注意

请务必不要将新的`ColumnOperators.any_()`方法与此方法的**传统**版本混淆，后者是专用于`ARRAY`的方法，采用不同的调用风格，`Comparator.any()`方法。

```py
method asc() → ColumnOperators
```

*继承自* `ColumnOperators.asc()` *方法的* `ColumnOperators`

对父对象生成一个`asc()`子句。

```py
method between(cleft: Any, cright: Any, symmetric: bool = False) → ColumnOperators
```

*继承自* `ColumnOperators.between()` *方法的* `ColumnOperators`

针对父对象生成一个`between()`子句，给定下限和上限范围。

```py
method bitwise_and(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_and()` *方法的* `ColumnOperators`

执行按位与操作，通常通过`&`运算符。

新功能在版本 2.0.2 中。

另请参阅

按位运算符

```py
method bitwise_lshift(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_lshift()` *方法的* `ColumnOperators`

执行按位左移操作，通常通过`<<`运算符。

新功能在版本 2.0.2 中。

另请参阅

按位运算符

```py
method bitwise_not() → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_not()` *方法的* `ColumnOperators`

执行按位非操作，通常通过`~`运算符。

新功能在版本 2.0.2 中。

另请参阅

按位运算符

```py
method bitwise_or(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_or()` *方法的* `ColumnOperators`

执行按位或操作，通常通过`|`运算符。

新功能在版本 2.0.2 中。

另请参阅

按位运算符

```py
method bitwise_rshift(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_rshift()` *方法的* `ColumnOperators`

执行按位右移操作，通常通过`>>`运算符。

新功能在版本 2.0.2 中。

另请参阅

按位运算符

```py
method bitwise_xor(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_xor()` *方法的* `ColumnOperators`

产生一个按位异或操作，通常通过`^`运算符实现，或者在 PostgreSQL 中使用`#`。

在 2.0.2 版中新增。

另请参阅

位运算符

```py
method bool_op(opstring: str, precedence: int = 0, python_impl: Callable[[...], Any] | None = None) → Callable[[Any], Operators]
```

*继承自* `Operators.bool_op()` *方法的* `Operators`

返回一个自定义的布尔运算符。

此方法是调用 `Operators.op()` 并传递 `Operators.op.is_comparison` 标志为 True 的简写。使用 `Operators.bool_op()` 的一个关键优势是，在使用列构造时，返回的表达式的“布尔”特性将存在于 [**PEP 484**](https://peps.python.org/pep-0484/) 目的上。

另请参阅

`Operators.op()`

```py
method collate(collation: str) → ColumnOperators
```

*继承自* `ColumnOperators.collate()` *方法的* `ColumnOperators`

生成一个针对父对象的 `collate()` 子句，给定排序规则字符串。

另请参阅

`collate()`

```py
method concat(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.concat()` *方法的* `ColumnOperators`

实现 ‘concat’ 运算符。

在列上下文中，生成子句 `a || b`，或在 MySQL 上使用 `concat()` 运算符。

```py
method contains(other: Any, **kw: Any) → ColumnOperators
```

*继承自* `ColumnOperators.contains()` *方法的* `ColumnOperators`

实现‘contains’运算符。

生成一个 LIKE 表达式，用于测试字符串值中间的匹配：

```py
column LIKE '%' || <other> || '%'
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.contains("foobar"))
```

由于该运算符使用`LIKE`，存在于<other>表达式内部的通配符字符`"%"`和`"_"`也将像通配符一样起作用。对于字面字符串值，可以将`ColumnOperators.contains.autoescape`标志设置为`True`，以对字符串值内这些字符的出现应用转义，使它们作为自身而不是通配符字符进行匹配。或者，`ColumnOperators.contains.escape`参数将建立一个给定字符作为转义字符，当目标表达式不是字面字符串时可能会有用。

参数：

+   `other` – 要比较的表达式。这通常是一个简单的字符串值，但也可以是任意的 SQL 表达式。默认情况下，LIKE 通配符字符`%`和`_`不会被转义，除非将`ColumnOperators.contains.autoescape`标志设置为 True。

+   `autoescape` –

    布尔值；当为 True 时，在 LIKE 表达式中建立一个转义字符，然后将其应用于比较值中所有的`"%"`、`"_"`和转义字符本身的出现，假定比较值是一个字面字符串而不是 SQL 表达式。

    例如：

    ```py
    somecolumn.contains("foo%bar", autoescape=True)
    ```

    将呈现为：

    ```py
    somecolumn LIKE '%' || :param || '%' ESCAPE '/'
    ```

    使用值`:param`为`"foo/%bar"`。

+   `escape` –

    一个字符，当给定时，将使用`ESCAPE`关键字将该字符建立为转义字符。然后可以将该字符放在`%`和`_`的出现之前，以允许它们作为自身而不是通配符字符。

    例如：

    ```py
    somecolumn.contains("foo/%bar", escape="^")
    ```

    将呈现为：

    ```py
    somecolumn LIKE '%' || :param || '%' ESCAPE '^'
    ```

    该参数也可以与`ColumnOperators.contains.autoescape`结合使用：

    ```py
    somecolumn.contains("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上述情况下，给定的字面参数将在传递到数据库之前转换为`"foo^%bar^^bat"`。

另请参阅

`ColumnOperators.startswith()`

`ColumnOperators.endswith()`

`ColumnOperators.like()`

```py
method desc() → ColumnOperators
```

*继承自* `ColumnOperators.desc()` *方法的* `ColumnOperators`

生成一个针对父对象的`desc()`子句。

```py
method distinct() → ColumnOperators
```

*继承自* `ColumnOperators.distinct()` *方法的* `ColumnOperators`

生成一个针对父对象的`distinct()`子句。

```py
method endswith(other: Any, escape: str | None = None, autoescape: bool = False) → ColumnOperators
```

*继承自* `ColumnOperators.endswith()` *方法的* `ColumnOperators`

实现‘endswith’运算符。

生成一个 LIKE 表达式，用于测试字符串值的末尾匹配：

```py
column LIKE '%' || <other>
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.endswith("foobar"))
```

由于该运算符使用`LIKE`，在<other>表达式中存在的通配符字符`"%"`和`"_"`也会像通配符一样起作用。 对于字面字符串值，可以将`ColumnOperators.endswith.autoescape`标志设置为`True`，以对字符串值中这些字符的出现进行转义，使它们匹配为它们自身而不是通配符字符。 或者，`ColumnOperators.endswith.escape`参数将建立一个给定字符作为转义字符，当目标表达式不是字面字符串时，这可能会有用。

参数：

+   `other` – 要比较的表达式。 这通常是一个简单的字符串值，但也可以是任意 SQL 表达式。 除非将`ColumnOperators.endswith.autoescape`标志设置为 True，否则不会默认转义 LIKE 通配符`%`和`_`。

+   `autoescape` –

    布尔值；当为 True 时，在 LIKE 表达式中建立一个转义字符，然后将其应用于比较值中所有的`"%"`、`"_"`和转义字符本身的出现，假定比较值是一个文字字符串而不是一个 SQL 表达式。

    诸如：

    ```py
    somecolumn.endswith("foo%bar", autoescape=True)
    ```

    将呈现为：

    ```py
    somecolumn LIKE '%' || :param ESCAPE '/'
    ```

    值为`:param`的值为`"foo/%bar"`。

+   `escape` –

    一个字符，给定时将使用`ESCAPE`关键字将其渲染为转义字符。然后，可以在`%`和`_`的出现之前放置此字符，以允许它们作为自己而不是通配符字符。

    诸如：

    ```py
    somecolumn.endswith("foo/%bar", escape="^")
    ```

    将呈现为：

    ```py
    somecolumn LIKE '%' || :param ESCAPE '^'
    ```

    参数也可以与`ColumnOperators.endswith.autoescape`结合使用：

    ```py
    somecolumn.endswith("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上面的例子中，给定的文字参数在传递给数据库之前将被转换为`"foo^%bar^^bat"`。

请参阅

`ColumnOperators.startswith()`

`ColumnOperators.contains()`

`ColumnOperators.like()`

```py
method has(criterion: _ColumnExpressionArgument[bool] | None = None, **kwargs: Any) → ColumnElement[bool]
```

返回一个 SQL 表达式，如果此元素引用满足给定条件的成员，则表示为 true。

`has()`的通常实现是`Comparator.has()`。

参数：

+   `criterion` – 针对成员类表或属性制定的可选 ClauseElement。

+   `**kwargs` – 键/值对，对应于将通过等式与相应值进行比较的成员类属性名称。

```py
method icontains(other: Any, **kw: Any) → ColumnOperators
```

*继承自* `ColumnOperators.icontains()` *的* `ColumnOperators`

实现`icontains`运算符，例如`ColumnOperators.contains()`的不区分大小写版本。

生成一个 LIKE 表达式，测试对字符串值中间的大小写不敏感匹配：

```py
lower(column) LIKE '%' || lower(<other>) || '%'
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.icontains("foobar"))
```

由于操作符使用了 `LIKE`，所以存在于 <other> 表达式中的通配符字符 `"%"` 和 `"_"` 也将像通配符一样行为。对于文字字符串值，可以将 `ColumnOperators.icontains.autoescape` 标志设置为 `True`，以对字符串值中这些字符的出现进行转义，使它们匹配为它们自身而不是通配符字符。或者，`ColumnOperators.icontains.escape` 参数将建立一个给定的字符作为转义字符，当目标表达式不是文字字符串时可能会有用。

参数：

+   `other` - 要进行比较的表达式。通常这是一个普通的字符串值，但也可以是任意的 SQL 表达式。LIKE 通配符字符 `%` 和 `_` 默认情况下不被转义，除非 `ColumnOperators.icontains.autoescape` 标志被设置为 True。

+   `autoescape` -

    布尔值；当为 True 时，在 LIKE 表达式中建立一个转义字符，然后将其应用于比较值中所有出现的 `"%"`、`"_"` 和转义字符本身，假定比较值是一个文字字符串而不是 SQL 表达式。

    一个如下的表达式：

    ```py
    somecolumn.icontains("foo%bar", autoescape=True)
    ```

    将被渲染为：

    ```py
    lower(somecolumn) LIKE '%' || lower(:param) || '%' ESCAPE '/'
    ```

    其中参数的值为`:param`，为`"foo/%bar"`。

+   `escape` -

    一个字符，当给定时将使用 `ESCAPE` 关键字来建立该字符作为转义字符。然后，可以将该字符放在 `%` 和 `_` 的前面，以允许它们作为自身而不是通配符字符。

    一个如下的表达式：

    ```py
    somecolumn.icontains("foo/%bar", escape="^")
    ```

    将被渲染为：

    ```py
    lower(somecolumn) LIKE '%' || lower(:param) || '%' ESCAPE '^'
    ```

    此参数也可以与`ColumnOperators.contains.autoescape`结合使用：

    ```py
    somecolumn.icontains("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上述情况下，给定的文字参数在传递到数据库之前将被转换为`"foo^%bar^^bat"`。

另见

`ColumnOperators.contains()`

```py
method iendswith(other: Any, escape: str | None = None, autoescape: bool = False) → ColumnOperators
```

*继承自* `ColumnOperators.iendswith()` *方法的* `ColumnOperators`

实现`iendswith`操作符，例如`ColumnOperators.endswith()`的不区分大小写版本。

生成一个 LIKE 表达式，用于对字符串值的不区分大小写匹配进行测试：

```py
lower(column) LIKE '%' || lower(<other>)
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.iendswith("foobar"))
```

由于该操作符使用`LIKE`，在<other>表达式内部存在的通配符字符`"%"`和`"_"`也将像通配符一样起作用。对于字面字符串值，可以将`ColumnOperators.iendswith.autoescape`标志设置为`True`，以对字符串值中这些字符的出现进行转义，使它们匹配为它们自己而不是通配符字符。或者，`ColumnOperators.iendswith.escape`参数将确定一个给定字符作为转义字符，当目标表达式不是字面字符串时可能会有用。

参数：

+   `other` – 要比较的表达式。通常是一个普通字符串值，但也可以是任意 SQL 表达式。LIKE 通配符字符`%`和`_`默认情况下不会被转义，除非设置了`ColumnOperators.iendswith.autoescape`标志为 True。

+   `autoescape` –

    布尔值；当为 True 时，在 LIKE 表达式中建立一个转义字符，然后将其应用于比较值中所有出现的`"%"`、`"_"`和转义字符本身，假定比较值是一个字面字符串而不是 SQL 表达式。

    例如表达式：

    ```py
    somecolumn.iendswith("foo%bar", autoescape=True)
    ```

    将呈现为：

    ```py
    lower(somecolumn) LIKE '%' || lower(:param) ESCAPE '/'
    ```

    其中`param`的值为`"foo/%bar"`。

+   `escape` –

    一个字符，当给定时，将使用`ESCAPE`关键字来确定该字符作为转义字符。然后可以将该字符放在`%`和`_`之前，以使它们可以作为它们自己而不是通配符字符。

    例如表达式：

    ```py
    somecolumn.iendswith("foo/%bar", escape="^")
    ```

    将呈现为：

    ```py
    lower(somecolumn) LIKE '%' || lower(:param) ESCAPE '^'
    ```

    参数也可以与`ColumnOperators.iendswith.autoescape`结合使用：

    ```py
    somecolumn.endswith("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上述情况下，给定的文字参数将在传递到数据库之前转换为`"foo^%bar^^bat"`。

另请参阅

`ColumnOperators.endswith()`

```py
method ilike(other: Any, escape: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.ilike()` *方法的* `ColumnOperators`

实现 `ilike` 运算符，例如，大小写不敏感的 LIKE。

在列上下文中，生成形式为：

```py
lower(a) LIKE lower(other)
```

或在支持 ILIKE 运算符的后端上：

```py
a ILIKE other
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.ilike("%foobar%"))
```

参数：

+   `other` – 要比较的表达式

+   `escape` –

    可选的转义字符，渲染 `ESCAPE` 关键字，例如：

    ```py
    somecolumn.ilike("foo/%bar", escape="/")
    ```

另请参阅

`ColumnOperators.like()`

```py
method in_(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.in_()` *方法的* `ColumnOperators`

实现 `in` 运算符。

在列上下文中，生成子句 `column IN <other>`。

给定的参数 `other` 可能是：

+   一个字面值列表，例如：

    ```py
    stmt.where(column.in_([1, 2, 3]))
    ```

    在此调用形式中，项目列表被转换为与给定列表相同长度的一组绑定参数：

    ```py
    WHERE COL IN (?, ?, ?)
    ```

+   如果比较的对象是包含多个表达式的`tuple_()`，可以提供一个元组列表：

    ```py
    from sqlalchemy import tuple_
    stmt.where(tuple_(col1, col2).in_([(1, 10), (2, 20), (3, 30)]))
    ```

+   一个空列表，例如：

    ```py
    stmt.where(column.in_([]))
    ```

    在此调用形式中，表达式呈现一个“空集”表达式。这些表达式针对各个后端进行了定制，通常试图得到一个空的 SELECT 语句作为子查询。例如在 SQLite 上，该表达式是：

    ```py
    WHERE col IN (SELECT 1 FROM (SELECT 1) WHERE 1!=1)
    ```

    版本 1.4 中更改：在所有情况下，空的 IN 表达式现在使用执行时生成的 SELECT 子查询。

+   可以使用绑定参数，例如 `bindparam()`，如果它包含 `bindparam.expanding` 标志：

    ```py
    stmt.where(column.in_(bindparam('value', expanding=True)))
    ```

    在此调用形式中，表达式呈现一个特殊的非 SQL 占位符表达式，看起来像：

    ```py
    WHERE COL IN ([EXPANDING_value])
    ```

    这个占位符表达式在语句执行时拦截，被转换成前面所示的可变数量的绑定参数形式。如果语句执行为：

    ```py
    connection.execute(stmt, {"value": [1, 2, 3]})
    ```

    数据库将为每个值传递一个绑定参数：

    ```py
    WHERE COL IN (?, ?, ?)
    ```

    版本 1.2 中新增：“expanding” 绑定参数

    如果传递了一个空列表，则渲染一个特殊的“空列表”表达式，该表达式特定于正在使用的数据库。在 SQLite 上，这将是：

    ```py
    WHERE COL IN (SELECT 1 FROM (SELECT 1) WHERE 1!=1)
    ```

    版本 1.3 中新增：“expanding” 绑定参数现在支持空列表

+   一个`select()` 构造，通常是一个相关的标量选择：

    ```py
    stmt.where(
        column.in_(
            select(othertable.c.y).
            where(table.c.x == othertable.c.x)
        )
    )
    ```

    在此调用形式中，`ColumnOperators.in_()` 呈现如下：

    ```py
    WHERE COL IN (SELECT othertable.y
    FROM othertable WHERE othertable.x = table.x)
    ```

参数：

**other** – 一个字面量列表，一个`select()` 构造，或者一个包含设置为 True 的`bindparam()` 构造，其中包括`bindparam.expanding` 标志。

```py
method is_(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.is_()` *方法的* `ColumnOperators`

实现`IS`运算符。

通常，当与`None`的值进行比较时���会自动生成`IS`，这会解析为`NULL`。但是，在某些平台上，如果与布尔值进行比较，则可能希望显式使用`IS`。

另请参见

`ColumnOperators.is_not()`

```py
method is_distinct_from(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.is_distinct_from()` *方法的* `ColumnOperators`

实现`IS DISTINCT FROM`运算符。

在大多数平台上呈现“a IS DISTINCT FROM b”；在某些平台上，如 SQLite，可能呈现“a IS NOT b”。

```py
method is_not(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.is_not()` *方法的* `ColumnOperators`

实现`IS NOT`运算符。

通常，当与`None`的值进行比较时，会自动生成`IS NOT`，这会解析为`NULL`。但是，在某些平台上，如果与布尔值进行比较，则可能希望显式使用`IS NOT`。

从版本 1.4 开始更改：`is_not()`运算符从先前版本的`isnot()`重命名。以前的名称仍可用于向后兼容。

另请参见

`ColumnOperators.is_()` 

```py
method is_not_distinct_from(other: Any) → ColumnOperators
```

*从* `ColumnOperators.is_not_distinct_from()` *方法继承*

实现 `IS NOT DISTINCT FROM` 运算符。

在大多数平台上渲染为 “a IS NOT DISTINCT FROM b”；在一些平台上，比如 SQLite，可能会渲染为 “a IS b”。

在 1.4 版本中更改：`is_not_distinct_from()` 运算符在之前的版本中从 `isnot_distinct_from()` 重命名。 以前的名称仍然可用于向后兼容。

```py
method isnot(other: Any) → ColumnOperators
```

*从* `ColumnOperators.isnot()` *方法继承*

实现 `IS NOT` 运算符。

通常，当与 `None` 的值进行比较时，会自动生成 `IS NOT`，它解析为 `NULL`。 然而，在某些平台上，如果要与布尔值进行比较，则可能希望显式使用 `IS NOT`。

在 1.4 版本中更改：`is_not()` 运算符在之前的版本中从 `isnot()` 重命名。 以前的名称仍然可用于向后兼容。

另请参阅

`ColumnOperators.is_()`

```py
method isnot_distinct_from(other: Any) → ColumnOperators
```

*从* `ColumnOperators.isnot_distinct_from()` *方法继承*

实现 `IS NOT DISTINCT FROM` 运算符。

在大多数平台上渲染为 “a IS NOT DISTINCT FROM b”；在一些平台上，比如 SQLite，可能会渲染为 “a IS b”。

在 1.4 版本中更改：`is_not_distinct_from()` 运算符在之前的版本中从 `isnot_distinct_from()` 重命名。 以前的名称仍然可用于向后兼容。

```py
method istartswith(other: Any, escape: str | None = None, autoescape: bool = False) → ColumnOperators
```

*从* `ColumnOperators.istartswith()` *方法继承*

实现 `istartswith` 运算符，例如，`ColumnOperators.startswith()` 的不区分大小写版本。

产生一个 LIKE 表达式，用于对字符串值的开头进行不区分大小写的匹配：

```py
lower(column) LIKE lower(<other>) || '%'
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.istartswith("foobar"))
```

由于该操作符使用`LIKE`，存在于<other>表达式内部的通配符字符`"%"`和`"_"`也将像通配符一样起作用。对于字面字符串值，可以将`ColumnOperators.istartswith.autoescape`标志设置为 True，以对字符串值内这些字符的出现应用转义，使它们匹配为自身而不是通配符字符。或者，`ColumnOperators.istartswith.escape`参数将建立一个给定字符作为转义字符，当目标表达式不是字面字符串时可能会有用。

参数：

+   `other` – 待比较的表达式。通常是一个普通字符串值，但也可以是任意 SQL 表达式。默认情况下，LIKE 通配符字符`%`和`_`不会被转义，除非设置了`ColumnOperators.istartswith.autoescape`标志为 True。

+   `autoescape` –

    布尔值；当为 True 时，在 LIKE 表达式中建立一个转义字符，然后将其应用于比较值中所有的`"%"`、`"_"`和转义字符本身的出现，假定比较值为字面字符串而不是 SQL 表达式。

    诸如以下表达式：

    ```py
    somecolumn.istartswith("foo%bar", autoescape=True)
    ```

    将渲染为：

    ```py
    lower(somecolumn) LIKE lower(:param) || '%' ESCAPE '/'
    ```

    以`:param`的值为`"foo/%bar"`。

+   `escape` –

    一个字符，当给定时将使用`ESCAPE`关键字来将该字符设定为转义字符。然后可以将该字符放在`%`和`_`的前面，以允许它们作为自身而不是通配符字符。

    诸如以下表达式：

    ```py
    somecolumn.istartswith("foo/%bar", escape="^")
    ```

    将渲染为：

    ```py
    lower(somecolumn) LIKE lower(:param) || '%' ESCAPE '^'
    ```

    参数也可以与`ColumnOperators.istartswith.autoescape`结合使用：

    ```py
    somecolumn.istartswith("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上述情况下，给定的字面参数将在传递到数据库之前转换为`"foo^%bar^^bat"`。

另请参见

`ColumnOperators.startswith()`

```py
method like(other: Any, escape: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.like()` *方法的* `ColumnOperators`

实现`like`操作符。

在列上下文中，生成表达式：

```py
a LIKE other
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.like("%foobar%"))
```

参数：

+   `other` – 待比较的表达式

+   `escape` –

    可选的转义字符，渲染`ESCAPE`关键字，例如：

    ```py
    somecolumn.like("foo/%bar", escape="/")
    ```

另请参见

`ColumnOperators.ilike()`

```py
method match(other: Any, **kwargs: Any) → ColumnOperators
```

*继承自* `ColumnOperators.match()` *方法的* `ColumnOperators`

实现特定于数据库的‘match’操作符。

`ColumnOperators.match()` 尝试解析为后端��供的类似 MATCH 的函数或操作符。例如：

+   PostgreSQL - 渲染`x @@ plainto_tsquery(y)`

    > 从版本 2.0 开始更改：现在在 PostgreSQL 中使用`plainto_tsquery()`代替`to_tsquery()`；为了与其他形式兼容，请参见全文搜索。

+   MySQL - 渲染`MATCH (x) AGAINST (y IN BOOLEAN MODE)`

    另请参见

    `match` - 具有附加功能的 MySQL 特定构造。

+   Oracle - 渲染`CONTAINS(x, y)`

+   其他后端可能提供特殊实现。

+   没有任何特殊实现的后端将将操作符发出为“MATCH”。这与 SQLite 兼容，例如。

```py
method not_ilike(other: Any, escape: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.not_ilike()` *方法的* `ColumnOperators`

实现`NOT ILIKE`操作符。

这等同于使用`ColumnOperators.ilike()`进行否定，即`~x.ilike(y)`。

从版本 1.4 开始更改：`not_ilike()`操作符从先前版本的`notilike()`重命名。以前的名称仍可用于向后兼容。

另请参见

`ColumnOperators.ilike()`

```py
method not_in(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.not_in()` *方法的* `ColumnOperators`

实现`NOT IN`操作符。

这等同于使用`ColumnOperators.in_()`进行否定，即`~x.in_(y)`。

如果`other`是一个空序列，则编译器会生成一个“空 not in”表达式。 默认情况下，这将产生“1 = 1”的表达式，以在所有情况下产生 true。 可以使用`create_engine.empty_in_strategy`来更改此行为。

自版本 1.4 起更改：`not_in()`运算符从先前版本的`notin_()`重命名。 以确保向后兼容性，先前的名称仍然可用。

自版本 1.2 起更改：`ColumnOperators.in_()`和`ColumnOperators.not_in()`运算符现在默认情况下为一个空 IN 序列生成一个“静态”表达式。

另请参阅

`ColumnOperators.in_()`

```py
method not_like(other: Any, escape: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.not_like()` *方法的* `ColumnOperators`

实现`NOT LIKE`运算符。

这相当于使用`ColumnOperators.like()`进行否定，即`~x.like(y)`。

自版本 1.4 起更改：`not_like()`运算符从先前版本的`notlike()`重命名。 以确保向后兼容性，先前的名称仍然可用。

另请参阅

`ColumnOperators.like()`

```py
method notilike(other: Any, escape: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.notilike()` *方法的* `ColumnOperators`

实现`NOT ILIKE`运算符。

这相当于使用否定与`ColumnOperators.ilike()`，即`~x.ilike(y)`。

自版本 1.4 起更改：`not_ilike()`运算符从先前版本的`notilike()`重命名。 以确保向后兼容性，先前的名称仍然可用。

另请参阅

`ColumnOperators.ilike()`

```py
method notin_(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.notin_()` *方法* 的 `ColumnOperators`

实现`NOT IN`运算符。

这相当于在`ColumnOperators.in_()`中使用否定，即`~x.in_(y)`。

在`other`为空序列的情况下，编译器会生成一个“空 not in”表达式。默认情况下，这会变成表达式“1 = 1”，以在所有情况下产生 true。可以使用`create_engine.empty_in_strategy`来更改此行为。

在版本 1.4 中更改：`not_in()`运算符从先前版本的`notin_()`重命名。先前的名称仍然可用于向后兼容。

在版本 1.2 中更改：`ColumnOperators.in_()` 和 `ColumnOperators.not_in()` 运算符现在默认情况下为一个空的 IN 序列生成一个“静态”表达式。

另请参阅

`ColumnOperators.in_()`

```py
method notlike(other: Any, escape: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.notlike()` *方法* 的 `ColumnOperators`

实现`NOT LIKE`运算符。

这相当于在`ColumnOperators.like()`中使用否定，即`~x.like(y)`。

在版本 1.4 中更改：`not_like()`运算符从先前版本的`notlike()`重命名。先前的名称仍然可用于向后兼容。

另请参阅

`ColumnOperators.like()`

```py
method nulls_first() → ColumnOperators
```

*继承自* `ColumnOperators.nulls_first()` *方法* 的 `ColumnOperators`

对父对象生成一个 `nulls_first()` 子句。

1.4 版本更改：`nulls_first()` 操作符从之前的版本 `nullsfirst()` 重命名。 以前的名称仍可用于向后兼容。

```py
method nulls_last() → ColumnOperators
```

*继承自* `ColumnOperators.nullslast()` *方法的* `ColumnOperators` *类*

对父对象生成一个 `nulls_last()` 子句。

1.4 版本更改：`nulls_last()` 操作符从之前的版本 `nullslast()` 重命名。 以前的名称仍可用于向后兼容。

```py
method nullsfirst() → ColumnOperators
```

*继承自* `ColumnOperators.nullsfirst()` *方法的* `ColumnOperators` *类*

对父对象生成一个 `nulls_first()` 子句。

1.4 版本更改：`nulls_first()` 操作符从之前的版本 `nullsfirst()` 重命名。 以前的名称仍可用于向后兼容。

```py
method nullslast() → ColumnOperators
```

*继承自* `ColumnOperators.nullslast()` *方法的* `ColumnOperators` *类*

对父对象生成一个 `nulls_last()` 子句。

1.4 版本更改：`nulls_last()` 操作符从之前的版本 `nullslast()` 重命名。 以前的名称仍可用于向后兼容。

```py
method of_type(class_: _EntityType[Any]) → PropComparator[_T_co]
```

重新定义此对象，以便使用多态子类、`with_polymorphic()` 构造或 `aliased()` 构造。

返回一个新的 PropComparator，可以从中评估进一步的标准。

例如：

```py
query.join(Company.employees.of_type(Engineer)).\
   filter(Engineer.name=='foo')
```

参数：

**class_** – 表示标准为针对此特定子类的类或映射器。

另请参阅

使用关联在别名目标之间连接 - 在 ORM 查询指南中

连接到特定子类型或使用 with_polymorphic()实体

```py
method op(opstring: str, precedence: int = 0, is_comparison: bool = False, return_type: Type[TypeEngine[Any]] | TypeEngine[Any] | None = None, python_impl: Callable[..., Any] | None = None) → Callable[[Any], Operators]
```

*继承自* `Operators.op()` *方法的* `Operators`

生成一个通用的运算符函数。

例如：

```py
somecolumn.op("*")(5)
```

产生：

```py
somecolumn * 5
```

这个函数也可以用来使位运算符明确。例如：

```py
somecolumn.op('&')(0xff)
```

是`somecolumn`中值的按位与。

参数：

+   `opstring` – 一个字符串，将作为中缀运算符输出在这个元素和传递给生成函数的表达式之间。

+   `precedence` –

    数据库在 SQL 表达式中期望应用于运算符的优先级。这个整数值作为 SQL 编译器的提示，用于知道何时应该在特定操作周围渲染显式括号。较低的数字将导致在应用于具有更高优先级的另一个运算符时表达式被加括号。默认值为`0`，低于所有运算符，除了逗号（`,`）和`AS`运算符。值为 100 将高于或等于所有运算符，-100 将低于或等于所有运算符。

    另请参阅

    我正在使用 op()生成自定义运算符，但我的括号没有正确显示 - SQLAlchemy SQL 编译器如何渲染括号的详细描述

+   `is_comparison` –

    legacy; 如果为 True，则该运算符将被视为“比较”运算符，即评估为布尔真/假值的运算符，如`==`，`>`等。提供此标志是为了 ORM 关系可以在自定义连接条件中使用时建立该运算符是比较运算符。

    使用`is_comparison`参数已被使用`Operators.bool_op()`方法取代；这个更简洁的操作符会自动设置这个参数，同时也提供正确的[**PEP 484**](https://peps.python.org/pep-0484/)类型支持，因为返回的对象将表达“布尔”数据类型，即`BinaryExpression[bool]`。

+   `return_type` – 一个`TypeEngine`类或对象，将强制此运算符产生的表达式的返回类型为该类型。默认情况下，指定`Operators.op.is_comparison`的运算符将解析为`Boolean`，而那些不指定的将与��操作数的类型相同。

+   `python_impl` –

    一个可选的 Python 函数，可以以与数据库服务器上运行此操作符时相同的方式评估两个 Python 值。用于在 Python 中进行 SQL 表达式评估函数，例如用于 ORM 混合属性的函数，以及在多行更新或删除后用于匹配会话中对象的 ORM“评估器”。

    例如：

    ```py
    >>> expr = column('x').op('+', python_impl=lambda a, b: a + b)('y')
    ```

    上述表达式的操作符也将适用于非 SQL 左右对象：

    ```py
    >>> expr.operator(5, 10)
    15
    ```

    2.0 版本中的新功能。

另请参阅

`Operators.bool_op()`

重新定义和创建新的操作符

在连接条件中使用自定义操作符

```py
method operate(op: OperatorType, *other: Any, **kwargs: Any) → Operators
```

*继承自* `Operators` *方法的* `Operators.operate()`

对参数进行操作。

这是最低级别的操作，默认情况下引发`NotImplementedError`。

在子类上覆盖这个方法可以让常见的行为应用到所有操作中。例如，重写`ColumnOperators`来应用`func.lower()`到左右两侧：

```py
class MyComparator(ColumnOperators):
    def operate(self, op, other, **kwargs):
        return op(func.lower(self), func.lower(other), **kwargs)
```

参数：

+   `op` – 操作符可调用。

+   `*other` – 操作的‘其他’一侧。对于大多数操作来说，将是一个单一的标量。

+   `**kwargs` – 修饰符。这些可以由特殊操作符传递，如`ColumnOperators.contains()`。

```py
attribute property
```

返回与此`PropComparator`关联的`MapperProperty`。

这里的返回值通常是`ColumnProperty`或`Relationship`的实例。

```py
method regexp_match(pattern: Any, flags: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators` *方法的* `ColumnOperators.regexp_match()`

实现了数据库特定的‘regexp match’操作符。

例如：

```py
stmt = select(table.c.some_column).where(
    table.c.some_column.regexp_match('^(b|c)')
)
```

`ColumnOperators.regexp_match()`尝试解析为后端提供的类似 REGEXP 的函数或操作符，但是可用的特定正则表达式语法和标志**不是后端无关的**。

示例包括：

+   PostgreSQL - 在否定时渲染`x ~ y`或`x !~ y`。

+   Oracle - 渲染`REGEXP_LIKE(x, y)`。

+   SQLite - 使用 SQLite 的`REGEXP`占位符运算符，并调用 Python 的`re.match()`内置函数。

+   其他后端可能提供特殊的实现。

+   没有任何特殊实现的后端将作为“REGEXP”或“NOT REGEXP”发出。例如，这与 SQLite 和 MySQL 兼容。

目前为 Oracle、PostgreSQL、MySQL 和 MariaDB 实现了正则表达式支持。SQLite 部分支持。第三方方言之间的支持可能有所不同。

参数：

+   `pattern` – 正则表达式模式字符串或列子句。

+   `flags` – 任何要应用的正则表达式字符串标志，仅作为普通的 Python 字符串传递。这些标志是特定于后端的。一些后端，如 PostgreSQL 和 MariaDB，可能会将标志作为模式的一部分来指定。在 PostgreSQL 中使用忽略大小写标志‘i’时，将使用忽略大小写的正则表达式匹配运算符`~*`或`!~*`。

1.4 版中的新功能。

从版本 1.4.48 更改为：2.0.18 请注意，由于实现错误，“flags”参数先前接受了 SQL 表达式对象，例如列表达式，除了普通的 Python 字符串。这种实现与缓存一起使用时无法正常工作，并已被移除；应该仅传递字符串作为“flags”参数，因为这些标志在 SQL 表达式中被呈现为文字内联值。

另请参见

`ColumnOperators.regexp_replace()`

```py
method regexp_replace(pattern: Any, replacement: Any, flags: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.regexp_replace()` *的* `ColumnOperators` *方法*

实现了一个特定于数据库的‘regexp replace’运算符。

例如：

```py
stmt = select(
    table.c.some_column.regexp_replace(
        'b(..)',
        'XY',
        flags='g'
    )
)
```

`ColumnOperators.regexp_replace()` 试图解析为由后端提供的类似 REGEXP_REPLACE 的函数，通常会发出函数`REGEXP_REPLACE()`。然而，特定的正则表达式语法和可用的标志**不是后端通用的**。

目前为 Oracle、PostgreSQL、MySQL 8 或更高版本和 MariaDB 实现了正则表达式替换支持。第三方方言之间的支持可能有所不同。

参数：

+   `pattern` – 正则表达式模式字符串或列子句。

+   `pattern` – 替换字符串或列子句。

+   `flags` – 任何要应用的正则表达式字符串标志，仅作为普通的 Python 字符串传递。这些标志是特定于后端的。一些后端，如 PostgreSQL 和 MariaDB，可能会将标志作为模式的一部分来指定。

1.4 版中的新功能。

从版本 1.4.48 改变，: 2.0.18 请注意，由于实现错误，之前“flags”参数接受了 SQL 表达式对象，例如列表达式，而不仅仅是普通的 Python 字符串。这种实现在缓存方面无法正常工作，已被移除；应该只传递字符串作为“flags”参数，因为这些标志会作为 SQL 表达式中的文字内联值呈现。

另请参见

`ColumnOperators.regexp_match()`

```py
method reverse_operate(op: OperatorType, other: Any, **kwargs: Any) → Operators
```

*继承自* `Operators` *的* `Operators.reverse_operate()` *方法。

对参数进行反向操作。

使用方法与 `operate()` 相同。

```py
method startswith(other: Any, escape: str | None = None, autoescape: bool = False) → ColumnOperators
```

*继承自* `ColumnOperators` *的* `ColumnOperators.startswith()` *方法*。

实现 `startswith` 操作符。

产生一个 LIKE 表达式，用于测试字符串值的开头是否匹配：

```py
column LIKE <other> || '%'
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.startswith("foobar"))
```

由于操作符使用 `LIKE`，所以在 <other> 表达式中存在的通配符字符 `"%"` 和 `"_"` 也将像通配符一样运行。对于字面字符串值，可以将 `ColumnOperators.startswith.autoescape` 标志设置为 `True`，以便对字符串值中这些字符的出现进行转义，使它们匹配为它们自身而不是通配符字符。或者，`ColumnOperators.startswith.escape` 参数将建立一个给定的字符作为转义字符，当目标表达式不是字面字符串时，这可能会有用。

参数:

+   `other` – 要比较的表达式。通常是一个普通字符串值，但也可以是任意的 SQL 表达式。LIKE 通配符 `%` 和 `_` 默认情况下不会被转义，除非 `ColumnOperators.startswith.autoescape` 标志设置为 True。

+   `autoescape` –

    布尔值；当为 True 时，在 LIKE 表达式中建立转义字符，然后将其应用于比较值内所有的 `"%"`、`"_"` 和转义字符本身的出现，假定该比较值为文本字符串而不是 SQL 表达式。

    一个表达式如下：

    ```py
    somecolumn.startswith("foo%bar", autoescape=True)
    ```

    将呈现为：

    ```py
    somecolumn LIKE :param || '%' ESCAPE '/'
    ```

    具有值 `:param` 的情况下，为 `"foo/%bar"`。

+   `escape` –

    给定的字符，当使用时会带有 `ESCAPE` 关键字来将该字符设定为转义字符。然后可以将该字符放在 `%` 和 `_` 的前面，以使它们可以被视为自身而不是通配符字符。

    一个表达式如下：

    ```py
    somecolumn.startswith("foo/%bar", escape="^")
    ```

    将呈现为：

    ```py
    somecolumn LIKE :param || '%' ESCAPE '^'
    ```

    参数也可以与 `ColumnOperators.startswith.autoescape` 结合使用：

    ```py
    somecolumn.startswith("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上面的情况下，给定的文本参数将在传递到数据库之前转换为 `"foo^%bar^^bat"`。

另请参阅

`ColumnOperators.endswith()`

`ColumnOperators.contains()`

`ColumnOperators.like()`

```py
attribute timetuple: Literal[None] = None
```

*继承自* `ColumnOperators.timetuple` *的属性* `ColumnOperators`

Hack，允许在左侧比较日期时间对象。

```py
class sqlalchemy.orm.Relationship
```

描述一个对象属性，该属性包含与相关数据库表对应的单个项目或项目列表。

公共构造函数是 `relationship()` 函数。

另请参阅

关系配置

2.0 版更改：将 `Relationship` 添加为 `RelationshipProperty` 的声明兼容子类。

**类签名**

类`sqlalchemy.orm.Relationship`（`sqlalchemy.orm.RelationshipProperty`，`sqlalchemy.orm.base._DeclarativeMapped`）

```py
class sqlalchemy.orm.RelationshipDirection
```

枚举指示 `RelationshipProperty` 的‘方向’。

`RelationshipDirection` 可从 `RelationshipProperty` 的 `Relationship.direction` 属性访问。

**成员**

MANYTOMANY, MANYTOONE, ONETOMANY

**类签名**

类 `sqlalchemy.orm.RelationshipDirection` (`enum.Enum`)

```py
attribute MANYTOMANY = 3
```

指示 `relationship()` 的多对多方向。

此符号通常由内部使用，但可能在某些 API 功能中公开。

```py
attribute MANYTOONE = 2
```

指示 `relationship()` 的多对一方向。

此符号通常由内部使用，但可能在某些 API 功能中公开。

```py
attribute ONETOMANY = 1
```

指示 `relationship()` 的一对多方向。

此符号通常由内部使用，但可能在某些 API 功能中公开。

```py
class sqlalchemy.orm.RelationshipProperty
```

描述持有单个项目或与相关数据库表对应的项目列表的对象属性。

公共构造函数是 `relationship()` 函数。

另请参阅

关系配置

**成员**

__eq__(), __init__(), __ne__(), adapt_to_entity(), and_(), any(), contains(), entity, has(), in_(), mapper, of_type(), cascade, cascade_iterator(), declarative_scan(), do_init(), entity, instrument_class(), mapper, merge()

**类签名**

类`sqlalchemy.orm.RelationshipProperty` (`sqlalchemy.orm._IntrospectsAnnotations`, `sqlalchemy.orm.StrategizedProperty`, `sqlalchemy.log.Identified`)

```py
class Comparator
```

为`RelationshipProperty`属性生成布尔值、比较和其他操作符。

请参阅`PropComparator`的文档，了解 ORM 级别操作符定义的简要概述。

另请参见

`PropComparator`

`Comparator`

`ColumnOperators`

重新定义和创建新操作符

`TypeEngine.comparator_factory`

**类签名**

类`sqlalchemy.orm.RelationshipProperty.Comparator` (`sqlalchemy.util.langhelpers.MemoizedSlots`, `sqlalchemy.orm.PropComparator`)

```py
method __eq__(other: Any) → ColumnElement[bool]
```

实现`==`运算符。

在多对一的上下文中，例如：

```py
MyClass.some_prop == <some object>
```

这通常会生成一个子句，例如：

```py
mytable.related_id == <some id>
```

其中`<some id>`是给定对象的主键。

`==`运算符为非多对一比较提供了部分功能：

+   不支持与集合进行比较。请使用`Comparator.contains()`。

+   与标量一对多相比，将生成一个子句，比较父级中的目标列与给定目标。

+   与标量多对多相比，关联表的别名也将被渲染，形成一个自然连接，作为查询主体的一部分。这对于超出简单 AND 比较的查询不起作用，例如使用 OR 的查询。请使用显式连接、外连接或`Comparator.has()`进行更全面的非多对一标量成员测试。

+   在一个一对多或多对多的上下文中与`None`进行比较会产生一个 NOT EXISTS 子句。

```py
method __init__(prop: RelationshipProperty[_PT], parentmapper: _InternalEntityType[Any], adapt_to_entity: AliasedInsp[Any] | None = None, of_type: _EntityType[_PT] | None = None, extra_criteria: Tuple[ColumnElement[bool], ...] = ())
```

`Comparator`的构造是 ORM 属性机制的内部实现。

```py
method __ne__(other: Any) → ColumnElement[bool]
```

实现`!=`运算符。

在多对一的上下文中，例如：

```py
MyClass.some_prop != <some object>
```

这通常会生成一个子句，例如：

```py
mytable.related_id != <some id>
```

其中`<some id>`是给定对象的主键。

`!=`运算符为非多对一比较提供了部分功能：

+   不支持对集合的比较。使用 `Comparator.contains()` 结合 `not_()`。

+   与标量一对多相比，将生成一个在父项中比较目标列与给定目标的子句。

+   与标量多对多相比，关联表的别名也将被呈现，形成查询主体的一部分的自然连接。这不适用于超出简单 AND 比较的查询，例如使用 OR 的查询。使用显式联接、外联接或 `Comparator.has()` 结合 `not_()` 进行更全面的非一对多标量成员测试。

+   在一对多或多对多的情况下与 `None` 比较会产生 EXISTS 子句。

```py
method adapt_to_entity(adapt_to_entity: AliasedInsp[Any]) → RelationshipProperty.Comparator[Any]
```

返回此 PropComparator 的副本，该副本将使用给定的`AliasedInsp` 来生成相应的表达式。

```py
method and_(*criteria: _ColumnExpressionArgument[bool]) → PropComparator[Any]
```

添加 AND 条件。

请参见`PropComparator.and_()` 以获取示例。

自 1.4 版本新增。

```py
method any(criterion: _ColumnExpressionArgument[bool] | None = None, **kwargs: Any) → ColumnElement[bool]
```

生成一个根据特定标准测试集合的表达式，使用 EXISTS。

例如：

```py
session.query(MyClass).filter(
    MyClass.somereference.any(SomeRelated.x==2)
)
```

将生成类似于以下的查询：

```py
SELECT * FROM my_table WHERE
EXISTS (SELECT 1 FROM related WHERE related.my_id=my_table.id
AND related.x=2)
```

因为 `Comparator.any()` 使用相关子查询，所以与大型目标表相比，其性能不如使用联接好。

`Comparator.any()` 特别适用于测试空集合：

```py
session.query(MyClass).filter(
    ~MyClass.somereference.any()
)
```

将生成：

```py
SELECT * FROM my_table WHERE
NOT (EXISTS (SELECT 1 FROM related WHERE
related.my_id=my_table.id))
```

`Comparator.any()` 仅适用于集合，即具有 `uselist=True` 的`relationship()`。对于标量引用，请使用 `Comparator.has()`。

```py
method contains(other: _ColumnExpressionArgument[Any], **kwargs: Any) → ColumnElement[bool]
```

返回一个简单的表达式，测试集合是否包含特定项。

`Comparator.contains()` 仅适用于集合，即实现一对多或多对多关系且 `uselist=True` 的`relationship()`。

在简单的一对多上下文中使用时，例如表达式：

```py
MyClass.contains(other)
```

生成的子句类似于：

```py
mytable.id == <some id>
```

其中 `<some id>` 是指 `other` 上的外键属性的值，该属性引用其父对象的主键。因此，`Comparator.contains()` 在与简单的一对多操作一起使用时非常有用。

对于多对多操作，`Comparator.contains()` 的行为有更多注意事项。关联表将呈现在语句中，生成一个“隐式”联接，即，在 WHERE 子句中包括多个表：

```py
query(MyClass).filter(MyClass.contains(other))
```

生成的查询类似于：

```py
SELECT * FROM my_table, my_association_table AS
my_association_table_1 WHERE
my_table.id = my_association_table_1.parent_id
AND my_association_table_1.child_id = <some id>
```

其中`<some id>`将是`other`的主键。从上面可以明显看出，当在超出简单 AND 连接的查询中使用多个由 OR 连接的`Comparator.contains()`表达式时，`Comparator.contains()`将**不会**与多对多集合一起工作。在这种情况下，需要使用子查询或显式“外连接”。查看`Comparator.any()`以获取使用 EXISTS 的性能较差的替代方案，或者参考`Query.outerjoin()`以及 Joins 以获取有关构建外连接的更多详细信息。

kwargs 可能会被此运算符忽略，但对于 API 符合性是必需的。

```py
attribute entity: _InternalEntityType[_PT]
```

被此`Comparator`引用的目标实体。

这是一个`Mapper`或`AliasedInsp`对象。

这是`relationship()`的“目标”或“远程”端。

```py
method has(criterion: _ColumnExpressionArgument[bool] | None = None, **kwargs: Any) → ColumnElement[bool]
```

生成一个表达式，使用 EXISTS 针对特定标准测试标量引用。

像这样的表达式：

```py
session.query(MyClass).filter(
    MyClass.somereference.has(SomeRelated.x==2)
)
```

将生成一个查询如下：

```py
SELECT * FROM my_table WHERE
EXISTS (SELECT 1 FROM related WHERE
related.id==my_table.related_id AND related.x=2)
```

因为`Comparator.has()`使用相关子查询，所以当与大型目标表进行比较时，其性能不如使用连接。

`Comparator.has()`仅适用于标量引用，即具有`uselist=False`的`relationship()`。对于集合引用，请使用`Comparator.any()`。

```py
method in_(other: Any) → NoReturn
```

生成一个 IN 子句 - 目前尚未为基于`relationship()`的属性实现此功能。

```py
attribute mapper: Mapper[_PT]
```

被此`Comparator`引用的目标`Mapper`。

这是`relationship()`的“目标”或“远程”端。

```py
method of_type(class_: _EntityType[Any]) → PropComparator[_PT]
```

重新定义此对象以多态子类的术语。

查看`PropComparator.of_type()`的示例。

```py
attribute cascade
```

返回此`RelationshipProperty`的当前级联设置。

```py
method cascade_iterator(type_: str, state: InstanceState[Any], dict_: _InstanceDict, visited_states: Set[InstanceState[Any]], halt_on: Callable[[InstanceState[Any]], bool] | None = None) → Iterator[Tuple[Any, Mapper[Any], InstanceState[Any], _InstanceDict]]
```

遍历与特定‘cascade’相关联的给定实例的实例，从此 MapperProperty 开始。

返回一个迭代器三元组（实例，映射器，状态）。

请注意，在调用 cascade_iterator 之前，将首先检查此 MapperProperty 上的‘cascade’集合是否具有给定类型。

此方法通常仅适用于 Relationship。

```py
method declarative_scan(decl_scan: _ClassScanMapperConfig, registry: _RegistryType, cls: Type[Any], originating_module: str | None, key: str, mapped_container: Type[Mapped[Any]] | None, annotation: _AnnotationScanType | None, extracted_mapped_annotation: _AnnotationScanType | None, is_dataclass_field: bool) → None
```

在早期声明扫描时执行类特定的初始化。

版本 2.0 中的新功能。

```py
method do_init() → None
```

执行子类特定的初始化后映射器创建步骤。

这是由`MapperProperty`对象的 init()方法调用的模板方法。

```py
attribute entity
```

返回目标映射实体，这是由此`RelationshipProperty`引用的类或别名类的 inspect()。

```py
method instrument_class(mapper: Mapper[Any]) → None
```

由 Mapper 调用的钩子，用于启动由此 MapperProperty 管理的类属性的工具化。

这里的 MapperProperty 通常会调用属性模块来设置 InstrumentedAttribute。

这一步是设置`InstrumentedAttribute`的两个步骤中的第一个步骤，并在映射器设置过程中早期调用。

第二步通常是通过 StrategizedProperty 通过 post_instrument_class()钩子调用的 init_class_attribute 步骤。此步骤为 InstrumentedAttribute 分配了附加状态（特别是“impl”），该状态在 MapperProperty 确定需要执行何种持久性管理后确定（例如标量、对象、集合等）。

```py
attribute mapper
```

返回此`RelationshipProperty`的目标`Mapper`。

```py
method merge(session: Session, source_state: InstanceState[Any], source_dict: _InstanceDict, dest_state: InstanceState[Any], dest_dict: _InstanceDict, load: bool, _recursive: Dict[Any, object], _resolve_conflict_map: Dict[_IdentityKeyType[Any], object]) → None
```

将此`MapperProperty`表示的属性从源对象合并到目标对象。

```py
class sqlalchemy.orm.SQLORMExpression
```

一种可用于指示任何 ORM 级别属性或对象的类型，用于 SQL 表达式构建的上下文中。

`SQLORMExpression`从核心`SQLColumnExpression`扩展，添加了额外的 ORM 特定的 SQL 方法，例如`PropComparator.of_type()`，并且是`InstrumentedAttribute`的基础之一。它可以在[**PEP 484**](https://peps.python.org/pep-0484/)类型提示中用于指示应该作为 ORM 级别属性表达式行为的参数或返回值。

版本 2.0.0b4 中的新功能。

**类签名**

类`sqlalchemy.orm.SQLORMExpression`（`sqlalchemy.orm.base.SQLORMOperations`，`sqlalchemy.sql.expression.SQLColumnExpression`，`sqlalchemy.util.langhelpers.TypingOnly`)

```py
class sqlalchemy.orm.Synonym
```

`SynonymProperty`类的声明性前端。

公共构造函数是 `synonym()` 函数。

2.0 版中的变更：将 `Synonym` 添加为与 `SynonymProperty` 兼容的声明式子类。

另请参阅

同义词 - 同义词概述

**类签名**

类 `sqlalchemy.orm.Synonym` (`sqlalchemy.orm.descriptor_props.SynonymProperty`, `sqlalchemy.orm.base._DeclarativeMapped`)

```py
class sqlalchemy.orm.SynonymProperty
```

将属性名标记为映射属性的同义词，即属性将反映另一个属性的值和表达行为。

`同义词` 是使用 `synonym()` 函数构建的。

另请参阅

同义词 - 同义词概述

**成员**

doc, info, key, parent, set_parent(), uses_objects

**类签名**

类 `sqlalchemy.orm.SynonymProperty` (`sqlalchemy.orm.descriptor_props.DescriptorProperty`)

```py
attribute doc: str | None
```

*继承自* `DescriptorProperty.doc` *属性的* `DescriptorProperty`

可选的文档字符串

```py
attribute info: _InfoType
```

*继承自* `MapperProperty.info` *属性的* `MapperProperty`

与对象关联的信息字典，允许将用户定义的数据与此 `InspectionAttr` 关联。

字典在首次访问时生成。或者，它可以作为 `column_property()`, `relationship()`, 或 `composite()` 函数的构造函数参数指定。

另请参阅

`QueryableAttribute.info`

`SchemaItem.info`

```py
attribute key: str
```

*继承自* `MapperProperty.key` *属性的* `MapperProperty`

类属性的名称

```py
attribute parent: Mapper[Any]
```

*继承自* `MapperProperty.parent` *属性的* `MapperProperty`

管理此属性的`Mapper`。

```py
method set_parent(parent: Mapper[Any], init: bool) → None
```

设置引用此 MapperProperty 的父 Mapper。

一些子类会重写此方法，在首次了解 Mapper 时执行额外的设置。

```py
attribute uses_objects
```

```py
class sqlalchemy.orm.QueryContext
```

```py
class default_load_options
```

**类签名**

类`sqlalchemy.orm.QueryContext.default_load_options` (`sqlalchemy.sql.expression.Options`)

```py
class sqlalchemy.orm.QueryableAttribute
```

用于代表`MapperProperty`对象拦截属性事件的描述符对象的基类。实际的`MapperProperty`可通过`QueryableAttribute.property`属性访问。

另请参阅

`InstrumentedAttribute`

`MapperProperty`

`Mapper.all_orm_descriptors`

`Mapper.attrs`

**成员**

adapt_to_entity(), and_(), expression, info, is_attribute, of_type(), operate(), parent, reverse_operate()

**类签名**

类`sqlalchemy.orm.QueryableAttribute` (`sqlalchemy.orm.base._DeclarativeMapped`, `sqlalchemy.orm.base.SQLORMExpression`, `sqlalchemy.orm.base.InspectionAttr`, `sqlalchemy.orm.PropComparator`, `sqlalchemy.sql.roles.JoinTargetRole`, `sqlalchemy.sql.roles.OnClauseRole`, `sqlalchemy.sql.expression.Immutable`, `sqlalchemy.sql.cache_key.SlotsMemoizedHasCacheKey`, `sqlalchemy.util.langhelpers.MemoizedSlots`, `sqlalchemy.event.registry.EventTarget`)

```py
method adapt_to_entity(adapt_to_entity: AliasedInsp[Any]) → Self
```

返回此 PropComparator 的副本，该副本将使用给定的`AliasedInsp`来生成相应的表达式。

```py
method and_(*clauses: _ColumnExpressionArgument[bool]) → QueryableAttribute[bool]
```

向由此关系属性表示的 ON 子句添加附加条件。

例如：

```py
stmt = select(User).join(
    User.addresses.and_(Address.email_address != 'foo')
)

stmt = select(User).options(
    joinedload(User.addresses.and_(Address.email_address != 'foo'))
)
```

1.4 版中的新功能。

另请参阅

将关联与自定义 ON 条件组合

向加载器选项添加条件

`with_loader_criteria()`

```py
attribute expression: ColumnElement[_T_co]
```

由此`QueryableAttribute`表示的 SQL 表达式对象。

通常情况下，这将是一个`ColumnElement`子类的实例，代表着一个列表达式。

```py
attribute info
```

返回底层 SQL 元素的‘info’字典。

此处的行为如下：

+   如果属性是一个列映射属性，即`ColumnProperty`，它直接映射到模式级`Column`对象，那么此属性将返回与核心级`Column`对象关联的`SchemaItem.info`字典。

+   如果属性是一个`ColumnProperty`，但映射到除`Column`之外的任何其他类型的 SQL 表达式，该属性将直接指向与`ColumnProperty`关联的`MapperProperty.info`字典，假设 SQL 表达式本身没有自己的`.info`属性（这应该是情况，除非用户定义的 SQL 构造已定义了一个）。

+   如果属性指的是任何其他类型的`MapperProperty`，包括`Relationship`，那么该属性将指向与该`MapperProperty`相关联的`MapperProperty.info`字典。

+   要无条件访问`MapperProperty.info`字典的`MapperProperty`，包括与`Column`直接关联的`ColumnProperty`，可以使用`QueryableAttribute.property`属性引用属性，如`MyClass.someattribute.property.info`。

另请参阅

`SchemaItem.info`

`MapperProperty.info`

```py
attribute is_attribute = True
```

如果此对象是 Python 的描述符，则为 True。

这可以指代许多类型。通常是一个处理`MapperProperty`属性事件的`QueryableAttribute`。但也可以是一个扩展类型，如`AssociationProxy`或`hybrid_property`。`InspectionAttr.extension_type`将引用一个常量，用于标识特定的子类型。

另请参阅

`Mapper.all_orm_descriptors`

```py
method of_type(entity: _EntityType[Any]) → QueryableAttribute[_T]
```

重新定义此对象以多态子类，`with_polymorphic()`构造或`aliased()`构造。

返回一个新的`PropComparator`，可以进一步评估标准。

例如：

```py
query.join(Company.employees.of_type(Engineer)).\
   filter(Engineer.name=='foo')
```

参数：

**class_** – 一个表示标准将针对特定子类的类或映射器。

另请参阅

使用关系在别名目标之间进行连接 - 在 ORM 查询指南中

连接到特定子类型或`with_polymorphic()`实体

```py
method operate(op: OperatorType, *other: Any, **kwargs: Any) → ColumnElement[Any]
```

对参数进行操作。

这是操作的最低级别，默认情况下引发`NotImplementedError`。

在子类上覆盖此操作可以使通用行为应用于所有操作。例如，覆盖`ColumnOperators`以将`func.lower()`应用于左侧和右侧：

```py
class MyComparator(ColumnOperators):
    def operate(self, op, other, **kwargs):
        return op(func.lower(self), func.lower(other), **kwargs)
```

参数：

+   `op` – 操作符可调用。

+   `*other` – 操作的‘另一’方。对于大多数操作来说，将是一个单一标量。

+   `**kwargs` – 修饰符。这些可能由特殊操作符（如`ColumnOperators.contains()`）传递。

```py
attribute parent: _InternalEntityType[Any]
```

返回表示父实体的检查实例。

这将是`Mapper`或`AliasedInsp`的实例，取决于此属性所关联的父实体的性质。

```py
method reverse_operate(op: OperatorType, other: Any, **kwargs: Any) → ColumnElement[Any]
```

对参数执行反向操作。

使用方式与`operate()`相同。

```py
class sqlalchemy.orm.UOWTransaction
```

```py
method filter_states_for_dep(dep, states)
```

将给定的 InstanceState 列表过滤为与给定 DependencyProcessor 相关的实例。

```py
method finalize_flush_changes() → None
```

在成功的 flush()后，将已处理的对象标记为干净/已删除。

在 execute()方法成功执行且事务已提交后，在 flush()方法内调用此方法。

```py
method get_attribute_history(state, key, passive=symbol('PASSIVE_NO_INITIALIZE'))
```

作为 attributes.get_state_history()的门面，包括结果的缓存。

```py
method is_deleted(state)
```

如果给定状态在此 uowtransaction 中标记为已删除，则返回`True`。

```py
method remove_state_actions(state)
```

从 uowtransaction 中移除状态的待处理操作。

**成员**

filter_states_for_dep(), finalize_flush_changes(), get_attribute_history(), is_deleted(), remove_state_actions(), was_already_deleted()

```py
method was_already_deleted(state)
```

如果给定状态已过期且先前已被删除，则返回`True`。
