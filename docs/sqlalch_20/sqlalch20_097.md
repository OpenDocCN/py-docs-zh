# 引擎和连接使用

> 原文：[`docs.sqlalchemy.org/en/20/core/engines_connections.html`](https://docs.sqlalchemy.org/en/20/core/engines_connections.html)

+   引擎配置

    +   支持的数据库

    +   数据库 URL

        +   转义特殊字符，如密码中的@符号

        +   程序化创建 URL

        +   特定后端的 URL

    +   引擎创建 API

        +   `create_engine()`

        +   `engine_from_config()`

        +   `create_mock_engine()`

        +   `make_url()`

        +   `create_pool_from_url()`

        +   `URL`

    +   连接池

    +   自定义 DBAPI connect()参数 / 连接时例程

        +   传递给 dbapi.connect()的特殊关键字参数

        +   控制参数如何传递给 DBAPI connect()函数

        +   连接后修改 DBAPI 连接，或连接后运行命令

        +   完全替换 DBAPI `connect()`函数

    +   配置日志记录

        +   更多关于 Echo 标志的信息

        +   设置日志名称

        +   设置每个连接/子引擎令牌

        +   隐藏参数

+   使用引擎和连接

    +   基本用法

    +   使用事务

        +   边用边提交

        +   一次性开始

        +   从引擎连接和一次性开始

        +   混合风格

    +   设置事务隔离级别，包括 DBAPI 自动提交

        +   为连接设置隔离级别或 DBAPI 自动提交

        +   为引擎设置隔离级别或 DBAPI 自动提交

        +   为单个引擎维护多个隔离级别

        +   理解 DBAPI 级别的自动提交隔离级别

    +   使用服务器端游标（即流式结果）

        +   通过 yield_per 进行固定缓冲区流式处理

        +   通过 stream_results 使用动态增长缓冲区进行流式处理

    +   模式名称的翻译

    +   SQL 编译缓存

        +   配置

        +   使用日志估算缓存性能

        +   缓存使用多少内存？

        +   禁用或使用备用字典缓存部分（或全部）语句

        +   为第三方方言进行缓存

        +   使用 Lambda 函数为语句生成提速

    +   “插入多个值”行为适用于 INSERT 语句

        +   当前支持

        +   禁用该特性

        +   批处理模式操作

        +   将 RETURNING 行与参数集相关联

        +   非批处理模式操作

        +   语句执行模型

        +   控制批处理大小

        +   日志和事件

        +   Upsert 支持

    +   引擎释放

    +   与 Driver SQL 和原始 DBAPI 连接一起工作

        +   直接调用驱动程序的 SQL 字符串

        +   直接使用 DBAPI 游标

        +   调用存储过程和用户定义函数

        +   多结果集

    +   注册新方言

        +   进程内注册方言

    +   连接 / 引擎 API

        +   `Connection`

        +   `CreateEnginePlugin`

        +   `Engine`

        +   `ExceptionContext`

        +   `NestedTransaction`

        +   `RootTransaction`

        +   `Transaction`

        +   `TwoPhaseTransaction`

    +   结果集 API

        +   `ChunkedIteratorResult`

        +   `CursorResult`

        +   `FilterResult`

        +   `FrozenResult`

        +   `IteratorResult`

        +   `MergedResult`

        +   `Result`

        +   `ScalarResult`

        +   `MappingResult`

        +   `Row`

        +   `RowMapping`

        +   `TupleResult`

+   连接池

    +   连接池配置

    +   切换池实现

    +   使用自定义连接函数

    +   构建池

    +   返回时重置

        +   对于非事务连接禁用返回时重置

        +   自定义返回时重置方案

        +   记录返回时重置事件

    +   池事件

    +   处理断开连接

        +   断开处理 - 悲观

        +   断开处理 - 乐观

        +   更多关于失效的信息

        +   支持断开情景下的新数据库错误代码

    +   使用 FIFO vs. LIFO

    +   在多进程或 os.fork() 中使用连接池

    +   直接使用池实例

    +   API 文档 - 可用的池实现

        +   `Pool`

        +   `QueuePool`

        +   `AsyncAdaptedQueuePool`

        +   `SingletonThreadPool`

        +   `AssertionPool`

        +   `NullPool`

        +   `StaticPool`

        +   `ManagesConnection`

        +   `ConnectionPoolEntry`

        +   `PoolProxiedConnection`

        +   `_ConnectionFairy`

        +   `_ConnectionRecord`

+   核心事件

    +   `事件`

        +   `Events.dispatch`

    +   连接池事件

        +   `PoolEvents`

        +   `PoolResetState`

    +   SQL 执行和连接事件

        +   `ConnectionEvents`

        +   `DialectEvents`

    +   模式事件

        +   `DDLEvents`

        +   `SchemaEventTarget`
