# 事件和内部

> 原文：[`docs.sqlalchemy.org/en/20/orm/extending.html`](https://docs.sqlalchemy.org/en/20/orm/extending.html)

SQLAlchemy ORM 以及 Core 通常通过事件钩子进行扩展。请务必查看事件系统的使用。

+   ORM 事件

    +   会话事件

    +   映射器事件

    +   实例事件

    +   属性事件

    +   查询事件

    +   仪器事件

+   ORM 内部

    +   `AttributeState`

    +   `CascadeOptions`

    +   `ClassManager`

    +   `ColumnProperty`

    +   `Composite`

    +   `CompositeProperty`

    +   `AttributeEventToken`

    +   `IdentityMap`

    +   `InspectionAttr`

    +   `InspectionAttrInfo`

    +   `InstanceState`

    +   `InstrumentedAttribute`

    +   `LoaderCallableStatus`

    +   `Mapped`

    +   `MappedColumn`

    +   `MapperProperty`

    +   `MappedSQLExpression`

    +   `InspectionAttrExtensionType`

    +   `NotExtension`

    +   `merge_result()`

    +   `merge_frozen_result()`

    +   `PropComparator`

    +   `Relationship`

    +   `RelationshipDirection`

    +   `RelationshipProperty`

    +   `SQLORMExpression`

    +   `Synonym`

    +   `SynonymProperty`

    +   `QueryContext`

    +   `QueryableAttribute`

    +   `UOWTransaction`

+   ORM 异常

    +   `ConcurrentModificationError`

    +   `DetachedInstanceError`

    +   `FlushError`

    +   `LoaderStrategyException`

    +   `NO_STATE`

    +   `ObjectDeletedError`

    +   `ObjectDereferencedError`

    +   `StaleDataError`

    +   `UnmappedClassError`

    +   `UnmappedColumnError`

    +   `UnmappedError`

    +   `UnmappedInstanceError`
