# 核心异常

> 原文：[`docs.sqlalchemy.org/en/20/core/exceptions.html`](https://docs.sqlalchemy.org/en/20/core/exceptions.html)

与 SQLAlchemy 一起使用的异常。

基础异常类是 `SQLAlchemyError`。作为 DBAPI 异常的结果引发的异常都是 `DBAPIError` 的子类。

```py
exception sqlalchemy.exc.AmbiguousForeignKeysError
```

当在联接过程中无法定位两个可选择项之间的多个匹配外键时引发。

**类签名**

类 `sqlalchemy.exc.AmbiguousForeignKeysError` (`sqlalchemy.exc.ArgumentError`)

```py
exception sqlalchemy.exc.ArgumentError
```

当提供了无效或冲突的函数参数时引发。

此错误通常对应于构建时状态错误。

**类签名**

类 `sqlalchemy.exc.ArgumentError` (`sqlalchemy.exc.SQLAlchemyError`)

```py
exception sqlalchemy.exc.AwaitRequired
```

如果在需要异步操作但没有等待时调用了异步 greenlet spawn，则会引发错误。

**类签名**

类 `sqlalchemy.exc.AwaitRequired` (`sqlalchemy.exc.InvalidRequestError`)

```py
exception sqlalchemy.exc.Base20DeprecationWarning
```

用于特定于 SQLAlchemy 2.0 中已弃用或传统的 API 的使用。

另请参阅

在 SQLAlchemy 2.0 中 <some function> 将不再 <something>。

SQLAlchemy 2.0 弃用模式

**类签名**

类 `sqlalchemy.exc.Base20DeprecationWarning` (`sqlalchemy.exc.SADeprecationWarning`)

```py
attribute deprecated_since: str | None = '1.4'
```

表明开始引发此废弃警告的版本

```py
exception sqlalchemy.exc.CircularDependencyError
```

当检测到循环依赖时，通过拓扑排序解决。

出现此错误的两种情况如下：

+   在会话刷新操作中，如果两个对象相互依赖，它们不能仅通过 INSERT 或 DELETE 语句进行插入或删除；需要使用 UPDATE 来后关联或先取消关联其中一个外键约束值。在 指向自身的行 / 相互依赖的行 中描述的 `post_update` 标志可以解决这种循环。

+   在 `MetaData.sorted_tables` 操作中，两个 `ForeignKey` 或 `ForeignKeyConstraint` 对象相互引用。对其中一个或两个应用 `use_alter=True` 标志，参见 通过 ALTER 创建/删除外键约束。

**类签名**

类 `sqlalchemy.exc.CircularDependencyError` (`sqlalchemy.exc.SQLAlchemyError`)

```py
method __init__(message: str, cycles: Any, edges: Any, msg: str | None = None, code: str | None = None)
```

```py
exception sqlalchemy.exc.CompileError
```

在 SQL 编译期间发生错误时引发

**类签名**

类 `sqlalchemy.exc.CompileError` (`sqlalchemy.exc.SQLAlchemyError`)

```py
exception sqlalchemy.exc.ConstraintColumnNotFoundError
```

当约束引用不存在于被约束表中的字符串列名称时引发。

在版本 2.0 中新增。

**类签名**

类 `sqlalchemy.exc.ConstraintColumnNotFoundError` (`sqlalchemy.exc.ArgumentError`)

```py
exception sqlalchemy.exc.DBAPIError
```

当数据库操作执行失败时引发。

封装由 DB-API 底层数据库操作引发的异常。可能的情况下，SQLAlchemy 的 `DBAPIError` 匹配 DB-API 标准异常类型的特定驱动程序实现被封装为相应的子类型。DB-API 的 `Error` 类型映射到 SQLAlchemy 中的 `DBAPIError`，否则名称相同。请注意，不能保证不同的 DB-API 实现将为任何给定的错误条件引发相同的异常类型。

`DBAPIError` 包含 `StatementError.statement` 和 `StatementError.params` 属性，提供有关出现问题的语句的上下文信息，典型情况下，错误是在发出 SQL 语句的上下文中引发的。

封装的异常对象在 `StatementError.orig` 属性中可用。其类型和属性是特定于 DB-API 实现的。

**类签名**

类 `sqlalchemy.exc.DBAPIError` (`sqlalchemy.exc.StatementError`)

```py
method __init__(statement: str | None, params: _AnyExecuteParams | None, orig: BaseException, hide_parameters: bool = False, connection_invalidated: bool = False, code: str | None = None, ismulti: bool | None = None)
```

```py
exception sqlalchemy.exc.DataError
```

包装一个 DB-API DataError。

**类签名**

类 `sqlalchemy.exc.DataError` (`sqlalchemy.exc.DatabaseError`)

```py
exception sqlalchemy.exc.DatabaseError
```

包装一个 DB-API DatabaseError。

**类签名**

类 `sqlalchemy.exc.DatabaseError` (`sqlalchemy.exc.DBAPIError`)

```py
exception sqlalchemy.exc.DisconnectionError
```

在原始 DB-API 连接上检测到断开连接。

此错误由连接池内部引发并消耗。它可以由 `PoolEvents.checkout()` 事件引发，以便主机池强制重试；在池放弃并引发关于连接尝试的 `InvalidRequestError` 之前，该异常将被连续捕获三次。

**类签名**

类 `sqlalchemy.exc.DisconnectionError` (`sqlalchemy.exc.SQLAlchemyError`)

| 对象名称 | 描述 |
| --- | --- |
| DontWrapMixin | 一个混合类，当应用于用户定义的异常类时，在执行语句过程中发出错误时不会被包装在 `StatementError` 内。 |
| HasDescriptionCode | 添加 ‘code’ 作为属性和 ‘_code_str’ 作为方法的助手 |

```py
class sqlalchemy.exc.DontWrapMixin
```

一个混合类，当应用于用户定义的异常类时，在执行语句过程中发出错误时不会被包装在 `StatementError` 内。

例如：

```py
from sqlalchemy.exc import DontWrapMixin

class MyCustomException(Exception, DontWrapMixin):
    pass

class MySpecialType(TypeDecorator):
    impl = String

    def process_bind_param(self, value, dialect):
        if value == 'invalid':
            raise MyCustomException("invalid!")
```

```py
exception sqlalchemy.exc.DuplicateColumnError
```

正在向表添加列，该列将替换另一列，但没有适当的参数允许此操作。

版本 2.0.0b4 中新增。

**类签名**

类 `sqlalchemy.exc.DuplicateColumnError` (`sqlalchemy.exc.ArgumentError`)

```py
class sqlalchemy.exc.HasDescriptionCode
```

添加 ‘code’ 作为属性和 ‘_code_str’ 作为方法的助手

```py
exception sqlalchemy.exc.IdentifierError
```

当模式名称超出最大字符限制时引发

**类签名**

类 `sqlalchemy.exc.IdentifierError` (`sqlalchemy.exc.SQLAlchemyError`)

```py
exception sqlalchemy.exc.IllegalStateChangeError
```

追踪状态的对象遇到了某种类型的非法状态改变。

版本 2.0 中新增。

**类签名**

类`sqlalchemy.exc.IllegalStateChangeError`（`sqlalchemy.exc.InvalidRequestError`）

```py
exception sqlalchemy.exc.IntegrityError
```

包装了一个 DB-API IntegrityError。

**类签名**

类`sqlalchemy.exc.IntegrityError`（`sqlalchemy.exc.DatabaseError`）

```py
exception sqlalchemy.exc.InterfaceError
```

包装了一个 DB-API InterfaceError。

**类签名**

类`sqlalchemy.exc.InterfaceError`（`sqlalchemy.exc.DBAPIError`）

```py
exception sqlalchemy.exc.InternalError
```

包装了一个 DB-API InternalError。

**类签名**

类`sqlalchemy.exc.InternalError`（`sqlalchemy.exc.DatabaseError`)

```py
exception sqlalchemy.exc.InvalidRequestError
```

SQLAlchemy 被要求执行一些它无法执行的操作。

此错误通常对应于运行时状态错误。

**类签名**

类`sqlalchemy.exc.InvalidRequestError`（`sqlalchemy.exc.SQLAlchemyError`）

```py
exception sqlalchemy.exc.InvalidatePoolError
```

当连接池应该使所有陈旧连接无效时引发。

一个`DisconnectionError`的子类，表示在连接上遇到的断开情况可能意味着整个池应该无效，因为数据库已重新启动。

此异常将以与`DisconnectionError`相同的方式处理，允许在放弃之前尝试三次重新连接。

1.2 版本中的新功能。

**类签名**

类`sqlalchemy.exc.InvalidatePoolError`（`sqlalchemy.exc.DisconnectionError`）

```py
exception sqlalchemy.exc.LegacyAPIWarning
```

表示处于“遗留”状态的 API，长期弃用。

**类签名**

类`sqlalchemy.exc.LegacyAPIWarning`（`sqlalchemy.exc.Base20DeprecationWarning`）

```py
exception sqlalchemy.exc.MissingGreenlet
```

如果在不在 greenlet spawn 上下文中调用 async greenlet await_ 时引发的错误。

**类签名**

类`sqlalchemy.exc.MissingGreenlet`（`sqlalchemy.exc.InvalidRequestError`）

```py
exception sqlalchemy.exc.MovedIn20Warning
```

用于指示仅移动的 API 的 RemovedIn20Warning 子类型。

**类签名**

class `sqlalchemy.exc.MovedIn20Warning` (`sqlalchemy.exc.Base20DeprecationWarning`)

```py
exception sqlalchemy.exc.MultipleResultsFound
```

需要单个数据库结果，但找到多个结果。

从版本 1.4 开始更改：此异常现在是 Core 中`sqlalchemy.exc`模块的一部分，已从 ORM 中移动。该符号仍然可以从`sqlalchemy.orm.exc`中导入。

**类签名**

class `sqlalchemy.exc.MultipleResultsFound` (`sqlalchemy.exc.InvalidRequestError`)

```py
exception sqlalchemy.exc.NoForeignKeysError
```

在连接期间无法找到两个可选择对象之间的外键时引发。

**类签名**

class `sqlalchemy.exc.NoForeignKeysError` (`sqlalchemy.exc.ArgumentError`)

```py
exception sqlalchemy.exc.NoInspectionAvailable
```

传递给`sqlalchemy.inspection.inspect()`的主题未产生检查的上下文。

**类签名**

class `sqlalchemy.exc.NoInspectionAvailable` (`sqlalchemy.exc.InvalidRequestError`)

```py
exception sqlalchemy.exc.NoReferenceError
```

由`ForeignKey`引发，表示无法解析引用。

**类签名**

class `sqlalchemy.exc.NoReferenceError` (`sqlalchemy.exc.InvalidRequestError`)

```py
exception sqlalchemy.exc.NoReferencedColumnError
```

当所引用的`Column`无法找到时，由`ForeignKey`引发。

**类签名**

class `sqlalchemy.exc.NoReferencedColumnError` (`sqlalchemy.exc.NoReferenceError`)

```py
method __init__(message: str, tname: str, cname: str)
```

```py
exception sqlalchemy.exc.NoReferencedTableError
```

当所引用的`Table`无法找到时，由`ForeignKey`引发。

**类签名**

class `sqlalchemy.exc.NoReferencedTableError` (`sqlalchemy.exc.NoReferenceError`)

```py
method __init__(message: str, tname: str)
```

```py
exception sqlalchemy.exc.NoResultFound
```

需要数据库结果，但未找到任何结果。

从版本 1.4 开始更改：此异常现在是 Core 中`sqlalchemy.exc`模块的一部分，已从 ORM 中移动。该符号仍然可以从`sqlalchemy.orm.exc`中导入。

**类签名**

class `sqlalchemy.exc.NoResultFound` (`sqlalchemy.exc.InvalidRequestError`)

```py
exception sqlalchemy.exc.NoSuchColumnError
```

请求了`Row`中不存在的列。

**类签名**

类 `sqlalchemy.exc.NoSuchColumnError` (`sqlalchemy.exc.InvalidRequestError`, `builtins.KeyError`)

```py
exception sqlalchemy.exc.NoSuchModuleError
```

当找不到特定名称的动态加载模块（通常是数据库方言）时引发。

**类签名**

类 `sqlalchemy.exc.NoSuchModuleError` (`sqlalchemy.exc.ArgumentError`)

```py
exception sqlalchemy.exc.NoSuchTableError
```

表不存在或对连接不可见。

**类签名**

类 `sqlalchemy.exc.NoSuchTableError` (`sqlalchemy.exc.InvalidRequestError`)

```py
exception sqlalchemy.exc.NotSupportedError
```

封装了一个 DB-API NotSupportedError。

**类签名**

类 `sqlalchemy.exc.NotSupportedError` (`sqlalchemy.exc.DatabaseError`)

```py
exception sqlalchemy.exc.ObjectNotExecutableError
```

当传递给 `.execute()` 的对象无法作为 SQL 执行时引发。

**类签名**

类 `sqlalchemy.exc.ObjectNotExecutableError` (`sqlalchemy.exc.ArgumentError`)

```py
method __init__(target: Any)
```

```py
exception sqlalchemy.exc.OperationalError
```

封装了一个 DB-API OperationalError。

**类签名**

类 `sqlalchemy.exc.OperationalError` (`sqlalchemy.exc.DatabaseError`)

```py
exception sqlalchemy.exc.PendingRollbackError
```

事务失败，需要在继续之前回滚。

自版本 1.4 新增。

**类签名**

类 `sqlalchemy.exc.PendingRollbackError` (`sqlalchemy.exc.InvalidRequestError`)

```py
exception sqlalchemy.exc.ProgrammingError
```

封装了一个 DB-API ProgrammingError。

**类签名**

类 `sqlalchemy.exc.ProgrammingError` (`sqlalchemy.exc.DatabaseError`)

```py
exception sqlalchemy.exc.ResourceClosedError
```

从已关闭状态的连接、游标或其他对象请求了操作。

**类签名**

类 `sqlalchemy.exc.ResourceClosedError` (`sqlalchemy.exc.InvalidRequestError`)

```py
exception sqlalchemy.exc.SADeprecationWarning
```

用于废弃的 API 的使用。

**类签名**

类`sqlalchemy.exc.SADeprecationWarning`（`sqlalchemy.exc.HasDescriptionCode`，`builtins.DeprecationWarning`）

```py
attribute deprecated_since: str | None = None
```

表明开始引发此弃用警告的版本

```py
exception sqlalchemy.exc.SAPendingDeprecationWarning
```

与`SADeprecationWarning`类似的警告，此警告在现代版本的 SQLAlchemy 中不使用。

**类签名**

类`sqlalchemy.exc.SAPendingDeprecationWarning`（`builtins.PendingDeprecationWarning`）

```py
attribute deprecated_since: str | None = None
```

表明开始引发此弃用警告的版本

```py
exception sqlalchemy.exc.SATestSuiteWarning
```

在测试期间检测到的非致命条件的警告

目前不在 SAWarning 之外，以便我们可以解决像 Alembic 这样的工具在警告方面做错事的问题。

**类签名**

类`sqlalchemy.exc.SATestSuiteWarning`（`builtins.Warning`）

```py
exception sqlalchemy.exc.SAWarning
```

在运行时发出。

**类签名**

类`sqlalchemy.exc.SAWarning`（`sqlalchemy.exc.HasDescriptionCode`，`builtins.RuntimeWarning`）

```py
exception sqlalchemy.exc.SQLAlchemyError
```

通用错误类。

**类签名**

类`sqlalchemy.exc.SQLAlchemyError`（`sqlalchemy.exc.HasDescriptionCode`，`builtins.Exception`）

```py
exception sqlalchemy.exc.StatementError
```

在执行 SQL 语句期间发生错误。

`StatementError`封装了执行过程中引发的异常，并具有`statement`和`params`属性，提供有关出现问题的语句的具体情况的上下文。

封装的异常对象可在`orig`属性中找到。

**类签名**

类`sqlalchemy.exc.StatementError`（`sqlalchemy.exc.SQLAlchemyError`）

```py
method __init__(message: str, statement: str | None, params: _AnyExecuteParams | None, orig: BaseException | None, hide_parameters: bool = False, code: str | None = None, ismulti: bool | None = None)
```

```py
attribute ismulti: bool | None = None
```

传递给`repr_params()`的多个参数。`None`是有意义的。

```py
attribute orig: BaseException | None = None
```

被抛出的原始异常。

```py
attribute params: _AnyExecuteParams | None = None
```

在发生此异常时使用的参数列表。

```py
attribute statement: str | None = None
```

在发生此异常时调用的字符串 SQL 语句。

```py
exception sqlalchemy.exc.TimeoutError
```

在连接池在获取连接时超时时引发。

**类签名**

类 `sqlalchemy.exc.TimeoutError` (`sqlalchemy.exc.SQLAlchemyError`)

```py
exception sqlalchemy.exc.UnboundExecutionError
```

尝试执行 SQL 但没有数据库连接。

**类签名**

类 `sqlalchemy.exc.UnboundExecutionError` (`sqlalchemy.exc.InvalidRequestError`)

```py
exception sqlalchemy.exc.UnreflectableTableError
```

表存在，但由于某种原因无法反映。

自 1.2 版本新增。

**类签名**

类 `sqlalchemy.exc.UnreflectableTableError` (`sqlalchemy.exc.InvalidRequestError`)

```py
exception sqlalchemy.exc.UnsupportedCompilationError
```

当操作不受给定编译器支持时引发。

另请参见

如何将 SQL 表达式渲染为字符串，可能包含内联的绑定参数？

编译器 StrSQLCompiler 无法渲染类型为 <element type> 的元素

**类签名**

类 `sqlalchemy.exc.UnsupportedCompilationError` (`sqlalchemy.exc.CompileError`)

```py
method __init__(compiler: Compiled | TypeCompiler, element_type: Type[ClauseElement], message: str | None = None)
```
