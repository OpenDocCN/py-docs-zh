# 元数据 / 模式

> 原文：[`docs.sqlalchemy.org/en/20/faq/metadata_schema.html`](https://docs.sqlalchemy.org/en/20/faq/metadata_schema.html)

+   当我说 `table.drop()` / `metadata.drop_all()` 时，我的程序挂起了

+   SQLAlchemy 支持 ALTER TABLE、CREATE VIEW、CREATE TRIGGER、Schema 升级功能吗？

+   我如何按照它们的依赖关系对 Table 对象进行排序？

+   我如何将 CREATE TABLE/ DROP TABLE 输出作为字符串获取？

+   我如何子类化 Table/Column 以提供某些行为/配置？

## 当我说 `table.drop()` / `metadata.drop_all()` 时，我的程序挂起了

这通常对应两种情况：1. 使用 PostgreSQL，它对表锁非常严格，2. 您仍然打开了一个包含对表的锁定的连接，并且与用于 DROP 语句的连接不同。这是模式的最简化版本：

```py
connection = engine.connect()
result = connection.execute(mytable.select())

mytable.drop(engine)
```

上述，连接池连接仍然被检出；此外，上述结果对象还保持对此连接的链接。如果使用“隐式执行”，结果将保持此连接打开，直到结果对象关闭或所有行都被耗尽。

调用 `mytable.drop(engine)` 试图在从 `Engine` 获取的第二连接上发出 DROP TABLE 操作，这将会被锁定。

解决方法是在发出 DROP TABLE 前关闭所有连接：

```py
connection = engine.connect()
result = connection.execute(mytable.select())

# fully read result sets
result.fetchall()

# close connections
connection.close()

# now locks are removed
mytable.drop(engine)
```

## SQLAlchemy 支持 ALTER TABLE、CREATE VIEW、CREATE TRIGGER、Schema 升级功能吗？

SQLAlchemy 直接不支持通用 ALTER。对于特殊的 ad-hoc 基础 DDL，可以使用 `DDL` 和相关构造。有关此主题的讨论，请参阅 自定义 DDL。

更全面的选择是使用模式迁移工具，例如 Alembic 或 SQLAlchemy-Migrate；有关此问题的讨论，请参阅 通过迁移修改数据库对象。

## 我如何按照它们的依赖关系对 Table 对象进行排序？

这可以通过 `MetaData.sorted_tables` 函数获得：

```py
metadata_obj = MetaData()
# ... add Table objects to metadata
ti = metadata_obj.sorted_tables
for t in ti:
    print(t)
```

## 如何将 CREATE TABLE/ DROP TABLE 输出作为字符串获取？

现代的 SQLAlchemy 具有表示 DDL 操作的从句构造。这些可以像任何其他 SQL 表达式一样渲染为字符串：

```py
from sqlalchemy.schema import CreateTable

print(CreateTable(mytable))
```

要获得特定于某个引擎的字符串：

```py
print(CreateTable(mytable).compile(engine))
```

还有一种特殊形式的`Engine`可通过`create_mock_engine()`访问，允许将整个元数据创建序列转储为字符串，使用以下方法：

```py
from sqlalchemy import create_mock_engine

def dump(sql, *multiparams, **params):
    print(sql.compile(dialect=engine.dialect))

engine = create_mock_engine("postgresql+psycopg2://", dump)
metadata_obj.create_all(engine, checkfirst=False)
```

[Alembic](https://alembic.sqlalchemy.org) 工具还支持一种“离线”SQL 生成模式，将数据库迁移呈现为 SQL 脚本。

## 如何对表/列进行子类化以提供特定的行为/配置？

`Table` 和 `Column` 不是直接子类化的良好目标。但是，可以使用创建函数来获取构造时的行为，并使用附加事件来处理与模式对象之间的链接，例如约束约定或命名约定。可以在 [命名约定](https://www.sqlalchemy.org/trac/wiki/UsageRecipes/NamingConventions) 中看到许多这些技术的示例。

## 当我说`table.drop()` / `metadata.drop_all()`时，我的程序挂起了。

这通常对应于两个条件：1. 使用 PostgreSQL，它对表锁非常严格，2. 你有一个仍然打开的连接，其中包含对表的锁，并且与用于 DROP 语句的连接不同。以下是该模式的最简版本：

```py
connection = engine.connect()
result = connection.execute(mytable.select())

mytable.drop(engine)
```

上面，连接池连接仍然被检出；此外，上述结果对象还维护对此连接的链接。如果使用“隐式执行”，结果将保持此连接打开，直到关闭结果对象或耗尽所有行。

对`mytable.drop(engine)`的调用尝试在从`Engine`获取的第二个连接上发出 DROP TABLE，这会导致锁定。

解决方案是在发出 DROP TABLE 前关闭所有连接：

```py
connection = engine.connect()
result = connection.execute(mytable.select())

# fully read result sets
result.fetchall()

# close connections
connection.close()

# now locks are removed
mytable.drop(engine)
```

## SQLAlchemy 支持 ALTER TABLE、CREATE VIEW、CREATE TRIGGER、Schema 升级功能吗？

SQLAlchemy 并不直接支持一般的 ALTER。对于特殊的按需 DDL，可以使用`DDL`和相关构造。有关此主题的讨论，请参阅 自定义 DDL。

更全面的选项是使用模式迁移工具，例如 Alembic 或 SQLAlchemy-Migrate；请参阅 通过迁移更改数据库对象 以讨论此问题。

## 如何按其依赖顺序对 Table 对象进行排序？

可通过`MetaData.sorted_tables`函数进行访问：

```py
metadata_obj = MetaData()
# ... add Table objects to metadata
ti = metadata_obj.sorted_tables
for t in ti:
    print(t)
```

## 如何将 CREATE TABLE/DROP TABLE 输出为字符串？

现代的 SQLAlchemy 有表示 DDL 操作的子句构造。这些可以像任何其他 SQL 表达式一样渲染为字符串：

```py
from sqlalchemy.schema import CreateTable

print(CreateTable(mytable))
```

要获取特定引擎的字符串：

```py
print(CreateTable(mytable).compile(engine))
```

还有一种特殊形式的 `Engine` 可以通过 `create_mock_engine()` 获得，它允许将整个元数据创建序列转储为字符串，使用以下方法：

```py
from sqlalchemy import create_mock_engine

def dump(sql, *multiparams, **params):
    print(sql.compile(dialect=engine.dialect))

engine = create_mock_engine("postgresql+psycopg2://", dump)
metadata_obj.create_all(engine, checkfirst=False)
```

[Alembic](https://alembic.sqlalchemy.org) 工具还支持一种“离线”SQL 生成模式，将数据库迁移呈现为 SQL 脚本。

## 我如何子类化 Table/Column 以提供某些行为/配置？

`Table` 和 `Column` 不适合直接进行子类化。但是，可以使用创建函数来获得在构造时的行为，以及使用附加事件来处理模式对象之间的链接行为，比如约束惯例或命名惯例。许多这些技术的示例可以在 [命名约定](https://www.sqlalchemy.org/trac/wiki/UsageRecipes/NamingConventions) 中看到。
