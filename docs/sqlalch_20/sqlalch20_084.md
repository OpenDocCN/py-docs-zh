# 表达式序列化器扩展

> 原文：[`docs.sqlalchemy.org/en/20/core/serializer.html`](https://docs.sqlalchemy.org/en/20/core/serializer.html)

用于与 SQLAlchemy 查询结构一起使用的序列化器/反序列化器对象，允许“上下文”反序列化。

遗留功能

序列化器扩展是**遗留的**，不应用于新开发。

可以使用任何 SQLAlchemy 查询结构，无论是基于 sqlalchemy.sql.* 还是 sqlalchemy.orm.*。结构引用的映射器、表、列、会话等在序列化形式中不会被持久化，而是在反序列化时重新关联到查询结构。

警告

序列化器扩展使用 pickle 对对象进行序列化和反序列化，因此与 [python 文档](https://docs.python.org/3/library/pickle.html) 中提到的相同的安全注意事项适用。

使用方式几乎与标准 Python pickle 模块相同：

```py
from sqlalchemy.ext.serializer import loads, dumps
metadata = MetaData(bind=some_engine)
Session = scoped_session(sessionmaker())

# ... define mappers

query = Session.query(MyClass).
    filter(MyClass.somedata=='foo').order_by(MyClass.sortkey)

# pickle the query
serialized = dumps(query)

# unpickle.  Pass in metadata + scoped_session
query2 = loads(serialized, metadata, Session)

print query2.all()
```

使用原始 pickle 时适用的类似限制也适用；映射类必须本身可被 pickle 化，这意味着它们可以从模块级别的命名空间导入。

序列化器模块仅适用于查询结构。不需要：

+   用户定义类的实例。在典型情况下，这些类不包含对引擎、会话或表达式构造的引用，因此可以直接序列化。

+   完全从序列化结构加载的表元数据（即在应用程序中尚未声明的元数据）。可以使用常规的 pickle.loads()/dumps() 来完全转储任何 `MetaData` 对象，通常是在以前的某个时间点从现有数据库反射的对象。序列化器模块专门用于相反的情况，即表元数据已经存在于内存中的情况。

| 对象名称 | 描述 |
| --- | --- |
| Deserializer(file[, metadata, scoped_session, engine]) |  |
| dumps(obj[, protocol]) |  |
| loads(data[, metadata, scoped_session, engine]) |  |
| Serializer(*args, **kw) |  |

```py
function sqlalchemy.ext.serializer.Deserializer(file, metadata=None, scoped_session=None, engine=None)
```

```py
function sqlalchemy.ext.serializer.Serializer(*args, **kw)
```

```py
function sqlalchemy.ext.serializer.dumps(obj, protocol=5)
```

```py
function sqlalchemy.ext.serializer.loads(data, metadata=None, scoped_session=None, engine=None)
```
