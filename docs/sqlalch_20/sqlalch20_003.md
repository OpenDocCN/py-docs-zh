# 建立连接 - Engine

> 原文：[`docs.sqlalchemy.org/en/20/tutorial/engine.html`](https://docs.sqlalchemy.org/en/20/tutorial/engine.html)

**欢迎 ORM 和 Core 读者！**

每一个连接到数据库的 SQLAlchemy 应用程序都需要使用一个 `Engine`。这个简短的部分适用于所有人。

任何 SQLAlchemy 应用程序的起点是一个称为`Engine` 的对象。这个对象充当连接到特定数据库的连接的中心来源，提供了一个工厂以及一个称为连接池的保持空间，用于这些数据库连接。该引擎通常是一个全局对象，仅为特定数据库服务器创建一次，并且使用 URL 字符串进行配置，该字符串将描述它应该如何连接到数据库主机或后端。

为了本教程，我们将使用内存中的 SQLite 数据库。这是一个方便的测试方法，无需设置实际的预先存在的数据库。通过使用`create_engine()` 函数创建 `Engine`：

```py
>>> from sqlalchemy import create_engine
>>> engine = create_engine("sqlite+pysqlite:///:memory:", echo=True)
```

`create_engine` 的主要参数是一个字符串 URL，上面传递的字符串是 `"sqlite+pysqlite:///:memory:"`。这个字符串向 `Engine` 指示了三个重要的事实：

1.  我们正在与什么样的数据库通信？上面的 `sqlite` 部分连接了 SQLAlchemy 到一个称为方言的对象。

1.  我们正在使用什么 DBAPI？Python DBAPI 是 SQLAlchemy 用来与特定数据库交互的第三方驱动程序。在这种情况下，我们使用的是 `pysqlite`，在现代 Python 中，它是 SQLite 的[sqlite3](https://docs.python.org/library/sqlite3.html) 标准库接口。如果省略，则 SQLAlchemy 将使用特定数据库的默认 DBAPI。

1.  我们如何定位数据库？在这种情况下，我们的 URL 包含短语 `/:memory:`，这是对 `sqlite3` 模块的一个指示，表明我们将使用一个**仅存在于内存中**的数据库。这种类型的数据库非常适合用于实验，因为它不需要任何服务器，也不需要创建新文件。

我们还指定了一个参数`create_engine.echo`，它将指示`Engine`将其发出的所有 SQL 记录到一个 Python 日志记录器，该记录器将写入标准输出。此标志是一种更正式设置 Python 日志记录的简便方式，并且对于脚本中的实验很有用。许多 SQL 示例将包括此 SQL 日志输出，在点击 `[SQL]` 链接后，将显示完整的 SQL 交互。
