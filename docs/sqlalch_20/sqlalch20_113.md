# 常见问题

> 原文：[`docs.sqlalchemy.org/en/20/faq/index.html`](https://docs.sqlalchemy.org/en/20/faq/index.html)

常见问题部分是一系列常见问题的不断增长的集合，涵盖了众多已知问题。

+   安装

    +   当我尝试使用 asyncio 时，为什么会出现关于未安装 greenlet 的错误？

+   连接 / 引擎

    +   如何配置日志记录？

    +   如何池化数据库连接？我的连接是否被池化？

    +   如何将自定义连接参数传递给我的数据库 API？

    +   “MySQL 服务器已断开连接”

    +   “命令不同步；你现在无法运行此命令” / “此结果对象不返回行。它已被自动关闭”

    +   如何自动“重试”语句执行？

    +   为什么 SQLAlchemy 发出了这么多 ROLLBACKs？

    +   我正在使用 SQLite 数据库的多个连接（通常用于测试事务操作），但我的测试程序无法工作！

    +   在使用 Engine 时，如何获取原始的 DBAPI 连接？

    +   如何在 Python 多进程或 os.fork() 中使用引擎 / 连接 / 会话？

+   元数据 / 模式

    +   当我使用 `table.drop()` / `metadata.drop_all()` 时，我的程序挂起了

    +   SQLAlchemy 是否支持 ALTER TABLE、CREATE VIEW、CREATE TRIGGER、模式升级功能？

    +   如何按依赖顺序对 Table 对象进行排序？

    +   如何将 CREATE TABLE / DROP TABLE 输出作为字符串获取？

    +   如何派生 Table/Column 以提供某些行为/配置？

+   SQL 表达式

    +   如何将 SQL 表达式呈现为字符串，可能包含内联的绑定参数？

    +   当将 SQL 语句字符串化时，为什么百分号会被加倍？

    +   我正在使用 `op()` 生成自定义运算符，但我的括号不正确

+   ORM 配置

    +   如何映射没有主键的表？

    +   如何配置一个列，该列是 Python 保留字或类似的？

    +   如何在给定映射类的情况下获取所有列、关系、映射属性等的列表？

    +   我收到关于“隐式组合列 X 在属性 Y 下”的警告或错误

    +   我使用声明性，并使用 `and_()` 或 `or_()` 设置 primaryjoin/secondaryjoin，并且收到有关外键的错误消息。

    +   为什么推荐 `LIMIT` 结合 `ORDER BY`（尤其是与 `subqueryload()` 一起）？

+   性能

    +   为什么我升级到 1.4 和/或 2.x 后应用程序变慢？

    +   我如何对基于 SQLAlchemy 的应用程序进行性能分析？

    +   我正在使用 ORM 插入 400,000 行，但速度非常慢！

+   会话 / 查询

    +   我正在使用我的会话重新加载数据，但它没有看到我在其他地方提交的更改

    +   “由于 flush 期间的前一个异常，此会话的事务已回滚。”（或类似的）

    +   如何制作一个查询，始终向每个查询添加特定的过滤器？

    +   我的查询没有返回与 query.count() 告诉我的相同数量的对象 - 为什么？

    +   我已经创建了一个对外连接的映射，虽然查询返回了行，但没有返回对象。为什么？

    +   我使用 `joinedload()` 或 `lazy=False` 创建了一个 JOIN/OUTER JOIN，但是当我尝试添加 WHERE、ORDER BY、LIMIT 等条件时，SQLAlchemy 并没有构造正确的查询（依赖于 (OUTER) JOIN）。

    +   查询没有 `__len__()`，为什么？

    +   如何在 ORM 查询中使用文本 SQL？

    +   我调用 `Session.delete(myobject)`，但它没有从父集合中删除！

    +   为什么在加载对象时我的 `__init__()` 没有被调用？

    +   我如何在 SA 的 ORM 中使用 ON DELETE CASCADE？

    +   我将我的实例的“foo_id”属性设置为“7”，但“foo”属性仍然是`None` - 难道它不应该加载 id 为 #7 的 Foo 吗？

    +   如何遍历与给定对象相关的所有对象？

    +   是否有一种方法可以自动地只拥有唯一的关键词（或其他类型的对象），而不必查询关键词并获得包含该关键词的行的引用？

    +   为什么 post_update 除了第一个 UPDATE 外还会发出 UPDATE？

+   第三方集成问题

    +   我遇到了与“`numpy.int64`”、“`numpy.bool_`”等相关的错误。

    +   SQL 表达式中期望 WHERE/HAVING 角色，实际得到了 True
