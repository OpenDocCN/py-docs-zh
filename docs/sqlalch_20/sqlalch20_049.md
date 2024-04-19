# 使用会话

> 原文：[`docs.sqlalchemy.org/en/20/orm/session.html`](https://docs.sqlalchemy.org/en/20/orm/session.html)

在 ORM 映射类配置 中描述的声明基础和 ORM 映射函数是 ORM 的主要配置接口。一旦配置了映射，持久性操作的主要使用接口是 `Session`。

+   会话基础知识

    +   会话是做什么的？

    +   使用会话的基础知识

        +   打开和关闭会话

        +   构建开始/提交/回滚块

        +   使用 `sessionmaker`

        +   查询

        +   添加新的或现有的项目

        +   删除

        +   刷新

        +   按主键获取

        +   过期/刷新

        +   使用任意 WHERE 子句进行 UPDATE 和 DELETE

        +   自动开始

        +   提交

        +   回滚

        +   关闭

    +   会话常见问题

        +   何时创建 `sessionmaker`？

        +   何时构建 `Session`，何时提交，何时关闭？

        +   会话是缓存吗？

        +   如何获取特定对象的 `Session`？

        +   会话是线程安全的吗？`AsyncSession` 在并发任务中安全共享吗？

+   状态管理

    +   对象状态简介

        +   获取对象的当前状态

    +   会话属性

    +   会话引用行为

    +   合并

        +   合并提示

    +   清除

    +   刷新/过期

        +   实际加载了什么

        +   何时使对象过期或刷新

+   级联操作

    +   保存更新

        +   双向关系中保存更新级联的行为

    +   删除

        +   使用删除级联与多对多关系

        +   使用 ORM 关系的外键 ON DELETE 级联

        +   使用外键 ON DELETE 与多对多关系

    +   删除孤儿

    +   合并

    +   刷新过期

    +   清除

    +   关于删除 - 删除从集合和标量关系引用的对象的注意事项

+   事务和连接管理

    +   管理事务

        +   使用 SAVEPOINT

        +   会话级别与引擎级别的事务控制

        +   显式开始

        +   启用两阶段提交

        +   设置事务隔离级别 / DBAPI 自动提交

        +   通过事件跟踪事务状态

    +   将会话加入外部事务（例如用于测试套件）

+   其他持久化技术

    +   将 SQL 插入/更新表达式嵌入到刷新中

    +   在会话中使用 SQL 表达式

    +   强制将具有默认值的列设置为 NULL

    +   获取服务器生成的默认值

        +   情况 1：非主键，支持 RETURNING 或等效

        +   情况 2：表包含与 RETURNING 不兼容的触发器生成值

        +   情况 3：不支持或不需要非主键、RETURNING 或等效项

        +   情况 4：支持主键、RETURNING 或等效项

        +   情况 5：不支持主键、RETURNING 或等效项

        +   关于急切获取用于 INSERT 或 UPDATE 的客户端调用的 SQL 表达式的注意事项

    +   使用 INSERT、UPDATE 和 ON CONFLICT（即 upsert）返回 ORM 对象

        +   使用 PostgreSQL ON CONFLICT 和 RETURNING 返回 upserted ORM 对象

    +   分区策略（例如每个会话多个数据库后端）

        +   简单垂直分区

        +   为多引擎会话协调事务

        +   自定义垂直分区

        +   水平分区

    +   批量操作

+   上下文/线程本地会话

    +   隐式方法访问

    +   线程本地作用域

    +   在 Web 应用程序中使用线程本地作用域

    +   使用自定义创建的作用域

    +   上下文会话 API

        +   `scoped_session`

        +   `ScopedRegistry`

        +   `ThreadLocalRegistry`

        +   `QueryPropertyDescriptor`

+   使用事件跟踪查询、对象和会话变更

    +   执行事件

        +   基本查询拦截

        +   添加全局 WHERE / ON 条件

        +   重新执行语句

    +   持久性事件

        +   `before_flush()`

        +   `after_flush()`

        +   `after_flush_postexec()`

        +   Mapper 级刷新事件

    +   对象生命周期事件

        +   瞬态

        +   Transient to Pending

        +   Pending to Persistent

        +   Pending to Transient

        +   Loaded as Persistent

        +   Persistent to Transient

        +   Persistent to Deleted

        +   Deleted to Detached

        +   Persistent to Detached

        +   Detached to Persistent

        +   Deleted to Persistent

    +   事务事件

    +   属性更改事件

+   会话 API

    +   Session and sessionmaker()

        +   `sessionmaker`

        +   `ORMExecuteState`

        +   `Session`

        +   `SessionTransaction`

        +   `SessionTransactionOrigin`

    +   Session Utilities

        +   `close_all_sessions()`

        +   `make_transient()`

        +   `make_transient_to_detached()`

        +   `object_session()`

        +   `was_deleted()`

    +   属性和状态管理工具

        +   `object_state()`

        +   `del_attribute()`

        +   `get_attribute()`

        +   `get_history()`

        +   `init_collection()`

        +   `flag_modified()`

        +   `flag_dirty()`

        +   `instance_state()`

        +   `is_instrumented()`

        +   `set_attribute()`

        +   `set_committed_value()`

        +   `History`
