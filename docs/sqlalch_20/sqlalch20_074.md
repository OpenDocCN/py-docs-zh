# 替代类仪器化

> 原文：[`docs.sqlalchemy.org/en/20/orm/extensions/instrumentation.html`](https://docs.sqlalchemy.org/en/20/orm/extensions/instrumentation.html)

可扩展的类仪器化。

`sqlalchemy.ext.instrumentation`包提供了 ORM 内的类仪器化的替代系统。类仪器化是指 ORM 如何将属性放在类上，以维护数据并跟踪对该数据的更改，以及安装在类上的事件钩子。

注意

该扩展包是为了与其他已经执行自己仪器化的对象管理包集成而提供的。它不适用于一般用途。

查看仪器扩展如何使用的示例，请参见示例属性仪器化。

## API 参考

| 对象名称 | 描述 |
| --- | --- |
| ExtendedInstrumentationRegistry | 使用额外的簿记扩展`InstrumentationFactory`，以适应多种类型的类管理器。 |
| instrumentation_finders | 一个可扩展的返回仪器化实现的可调用序列 |
| INSTRUMENTATION_MANAGER | 属性，当存在于映射类上时，选择自定义仪器化。 |
| InstrumentationFactory | 用于新的 ClassManager 实例的工厂。 |
| InstrumentationManager | 用户定义的类仪器化扩展。 |

```py
sqlalchemy.ext.instrumentation.INSTRUMENTATION_MANAGER = '__sa_instrumentation_manager__'
```

属性，当存在于映射类上时，选择自定义仪器化。

允许类指定稍微或完全不同的技术来跟踪对映射属性和集合所做的更改。

在给定对象继承层次结构中只允许一个仪器化实现。

此属性的值必须是可调用的，并将传递一个类对象。可调用对象必须返回以下之一：

> +   一个`InstrumentationManager`的实例或子类
> +   
> +   实现所有或部分 InstrumentationManager 的对象（待办事项）
> +   
> +   实现上述所有或部分的可调用对象字典（待办事项）
> +   
> +   一个`ClassManager`的实例或子类

一旦导入了`sqlalchemy.ext.instrumentation`模块，此属性将被 SQLAlchemy 仪器解析所使用。 如果在全局仪器查找器列表中安装了自定义查找器，则它们可能会选择是否尊重此属性。

```py
class sqlalchemy.orm.instrumentation.InstrumentationFactory
```

用于新的 ClassManager 实例的工厂。

**类签名**

类`sqlalchemy.orm.instrumentation.InstrumentationFactory` (`sqlalchemy.event.registry.EventTarget`)

```py
class sqlalchemy.ext.instrumentation.InstrumentationManager
```

用户定义的类仪器扩展。

`InstrumentationManager` 可以被子类化以改变类仪器化的进行方式。 此类存在的目的是与其他对象管理框架集成，这些框架希望完全修改 ORM 的仪器方法，并且不适用于常规使用。 有关截取类仪器化事件，请参阅`InstrumentationEvents`。

**成员**

dict_getter(), get_instance_dict(), initialize_instance_dict(), install_descriptor(), install_member(), install_state(), instrument_attribute(), instrument_collection_class(), manage(), manager_getter(), post_configure_attribute(), remove_state(), state_getter(), uninstall_descriptor(), uninstall_member(), unregister()

此类的 API 应被视为半稳定的，并且可能会随着新版本略微更改。

```py
method dict_getter(class_)
```

```py
method get_instance_dict(class_, instance)
```

```py
method initialize_instance_dict(class_, instance)
```

```py
method install_descriptor(class_, key, inst)
```

```py
method install_member(class_, key, implementation)
```

```py
method install_state(class_, instance, state)
```

```py
method instrument_attribute(class_, key, inst)
```

```py
method instrument_collection_class(class_, key, collection_class)
```

```py
method manage(class_, manager)
```

```py
method manager_getter(class_)
```

```py
method post_configure_attribute(class_, key, inst)
```

```py
method remove_state(class_, instance)
```

```py
method state_getter(class_)
```

```py
method uninstall_descriptor(class_, key)
```

```py
method uninstall_member(class_, key)
```

```py
method unregister(class_, manager)
```

```py
sqlalchemy.ext.instrumentation.instrumentation_finders = [<function find_native_user_instrumentation_hook>]
```

一个可扩展的返回仪器实现的可调用序列

当一个类被注册时，每个可调用对象都将传递一个类对象。如果返回 None，则会查阅序列中的下一个查找器。否则，返回值必须是一个遵循与 sqlalchemy.ext.instrumentation.INSTRUMENTATION_MANAGER 相同指南的仪器工厂。

默认情况下，唯一的查找器是 find_native_user_instrumentation_hook，它搜索 INSTRUMENTATION_MANAGER。如果所有查找器都返回 None，则使用标准的 ClassManager 仪器。

```py
class sqlalchemy.ext.instrumentation.ExtendedInstrumentationRegistry
```

用额外的记录扩展了 `InstrumentationFactory`，以适应多种类型的类管理器。

**类签名**

类 `sqlalchemy.ext.instrumentation.ExtendedInstrumentationRegistry` (`sqlalchemy.orm.instrumentation.InstrumentationFactory`)

## API 参考

| 对象名称 | 描述 |
| --- | --- |
| 扩展仪器注册表 | 用额外的记录扩展了 `InstrumentationFactory`，以适应多种类型的类管理器。 |
| instrumentation_finders | 一个可扩展的序列，其中包含返回仪器实现的可调用对象。 |
| INSTRUMENTATION_MANAGER | 属性，在映射类上出现时选择自定义仪器。 |
| 仪器工厂 | 用于创建新的 ClassManager 实例的工厂。 |
| 仪器管理器 | 用户定义的类仪器扩展。 |

```py
sqlalchemy.ext.instrumentation.INSTRUMENTATION_MANAGER = '__sa_instrumentation_manager__'
```

属性，在映射类上出现时选择自定义仪器。

允许一个类指定一种稍微或完全不同的技术来跟踪对映射属性和集合所做的更改。

在给定对象继承层次结构中只允许有一个仪器实现。

此属性的值必须是一个可调用对象，并将传递一个类对象。可调用对象必须返回以下之一：

> +   `InstrumentationManager` 或其子类的实例
> +   
> +   实现了所有或部分 InstrumentationManager 的对象（待办）
> +   
> +   一个可调用对象的字典，实现了上述所有或部分功能（待办）
> +   
> +   `ClassManager` 或其子类的实例

一旦导入 `sqlalchemy.ext.instrumentation` 模块，此属性将由 SQLAlchemy 仪器化解析所使用。如果在全局 `instrumentation_finders` 列表中安装了自定义查找器，则它们可能会选择是否尊重此属性。

```py
class sqlalchemy.orm.instrumentation.InstrumentationFactory
```

生成新的 `ClassManager` 实例的工厂。

**类签名**

类 `sqlalchemy.orm.instrumentation.InstrumentationFactory` (`sqlalchemy.event.registry.EventTarget`)

```py
class sqlalchemy.ext.instrumentation.InstrumentationManager
```

用户自定义类仪器化扩展。

可以通过子类化 `InstrumentationManager` 来更改类仪器化的方式。此类存在的目的是为了与其他希望完全修改 ORM 的仪器化方法的对象管理框架集成，并不适用于常规使用。要拦截类仪器化事件，请参阅 `InstrumentationEvents`。

**成员**

dict_getter(), get_instance_dict(), initialize_instance_dict(), install_descriptor(), install_member(), install_state(), instrument_attribute(), instrument_collection_class(), manage(), manager_getter(), post_configure_attribute(), remove_state(), state_getter(), uninstall_descriptor(), uninstall_member(), unregister()

该类的 API 应被视为半稳定，可能会在新版本中略微更改。

```py
method dict_getter(class_)
```

```py
method get_instance_dict(class_, instance)
```

```py
method initialize_instance_dict(class_, instance)
```

```py
method install_descriptor(class_, key, inst)
```

```py
method install_member(class_, key, implementation)
```

```py
method install_state(class_, instance, state)
```

```py
method instrument_attribute(class_, key, inst)
```

```py
method instrument_collection_class(class_, key, collection_class)
```

```py
method manage(class_, manager)
```

```py
method manager_getter(class_)
```

```py
method post_configure_attribute(class_, key, inst)
```

```py
method remove_state(class_, instance)
```

```py
method state_getter(class_)
```

```py
method uninstall_descriptor(class_, key)
```

```py
method uninstall_member(class_, key)
```

```py
method unregister(class_, manager)
```

```py
sqlalchemy.ext.instrumentation.instrumentation_finders = [<function find_native_user_instrumentation_hook>]
```

一个可扩展的可调用序列，返回仪器化实现。

当一个类被注册时，每个可调用对象都将传递一个类对象。如果返回 None，则会查阅序列中的下一个查找器。否则，返回值必须是一个遵循与 sqlalchemy.ext.instrumentation.INSTRUMENTATION_MANAGER 相同指南的检测工厂。

默认情况下，唯一的查找器是 find_native_user_instrumentation_hook，它搜索 INSTRUMENTATION_MANAGER。如果所有查找器都返回 None，则使用标准的 ClassManager 仪器化。

```py
class sqlalchemy.ext.instrumentation.ExtendedInstrumentationRegistry
```

通过额外的簿记扩展`InstrumentationFactory`，以适应多种类型的类管理器。

**类签名**

类`sqlalchemy.ext.instrumentation.ExtendedInstrumentationRegistry`（`sqlalchemy.orm.instrumentation.InstrumentationFactory`）
