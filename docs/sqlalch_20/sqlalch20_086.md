# 访问者和遍历实用程序

> 原文：[`docs.sqlalchemy.org/en/20/core/visitors.html`](https://docs.sqlalchemy.org/en/20/core/visitors.html)

`sqlalchemy.sql.visitors` 模块由用于通用地 **遍历** 核心 SQL 表达式结构的类和函数组成。这与 Python 的 `ast` 模块类似，因为它提供了一个程序可以操作 SQL 表达式每个组件的系统。它通常用于定位各种类型的元素，如 `Table` 或 `BindParameter` 对象，以及更改结构状态，如使用其他 FROM 子句替换某些 FROM 子句。

注意

`sqlalchemy.sql.visitors` 模块是一个内部 API，不是完全公开的。它可能会发生变化，而且对于不考虑 SQLAlchemy 内部工作方式的使用模式可能无法正常运行。

`sqlalchemy.sql.visitors` 模块是 SQLAlchemy 的 **内部** 部分，通常不会由调用应用程序代码使用。但是，在某些边缘情况下会使用它，例如构建缓存例程以及使用 自定义 SQL 构造和编译扩展 构建自定义 SQL 表达式时。

访问者/遍历接口和库函数。

| 对象名称 | 描述 |
| --- | --- |
| anon_map | `cache_anon_map` 的别名 |
| cloned_traverse(obj, opts, visitors) | 克隆给定的表达式结构，允许访问者修改可变对象。 |
| ExternalTraversal | 用于可以使用 `traverse()` 函数进行外部遍历的访问者对象的基类。 |
| InternalTraversal | 定义用于内部遍历的访问者符号。 |
| iterate(obj[, opts]) | 遍历给定的表达式结构，返回一个迭代器。 |
| replacement_traverse(obj, opts, replace) | 克隆给定的表达式结构，允许使用给定的替换函数进行元素替换。 |
| traverse(obj, opts, visitors) | 使用默认迭代器遍历给定的表达式结构并访问。 |
| traverse_using(iterator, obj, visitors) | 使用给定的对象迭代器访问给定的表达式结构。 |
| Visitable | 可访问对象的基类。 |

```py
class sqlalchemy.sql.visitors.ExternalTraversal
```

用于使用`traverse()`函数进行外部遍历的访问者对象的基类。

直接使用`traverse()`函数通常更可取。

**成员**

chain(), iterate(), traverse(), visitor_iterator

**类签名**

类`sqlalchemy.sql.visitors.ExternalTraversal`（`sqlalchemy.util.langhelpers.MemoizedSlots`）

```py
method chain(visitor: ExternalTraversal) → _ExtT
```

在此 ExternalTraversal 上“链接”一个额外的 ExternalTraversal

连接的访问者将在此后接收所有访问事件。

```py
method iterate(obj: ExternallyTraversible | None) → Iterator[ExternallyTraversible]
```

遍历给定的表达式结构，返回所有元素的迭代器。

```py
method traverse(obj: ExternallyTraversible | None) → ExternallyTraversible | None
```

遍历并访问给定的表达式结构。

```py
attribute visitor_iterator
```

通过此访问者和每个“链接”访问者进行迭代。

```py
class sqlalchemy.sql.visitors.InternalTraversal
```

定义用于内部遍历的访问者符号。

`InternalTraversal`类有两种用法。一种是它可以作为一个实现该类各种访问方法的对象的超类。另一种是`InternalTraversal`自身的符号被用在`_traverse_internals`集合中。例如，`Case`对象将`_traverse_internals`定义为

```py
class Case(ColumnElement[_T]):
    _traverse_internals = [
        ("value", InternalTraversal.dp_clauseelement),
        ("whens", InternalTraversal.dp_clauseelement_tuples),
        ("else_", InternalTraversal.dp_clauseelement),
    ]
```

在上面，`Case`类将其内部状态表示为名为`value`、`whens`和`else_`的属性。它们各自链接到一个`InternalTraversal`方法，该方法指示每个属性引用的数据结构类型。

使用`_traverse_internals`结构，`InternalTraversible`类型的对象将自动实现以下方法：

+   `HasTraverseInternals.get_children()`

+   `HasTraverseInternals._copy_internals()`

+   `HasCacheKey._gen_cache_key()`

子类还可以直接实现这些方法，特别是`HasTraverseInternals._copy_internals()`方法，当需要特殊步骤时。

版本 1.4 中的新功能。

**成员**

dp_annotations_key, dp_anon_name, dp_boolean, dp_clauseelement, dp_clauseelement_list, dp_clauseelement_tuple, dp_clauseelement_tuples, dp_dialect_options, dp_dml_multi_values, dp_dml_ordered_values, dp_dml_values, dp_fromclause_canonical_column_collection, dp_fromclause_ordered_set, dp_has_cache_key, dp_has_cache_key_list, dp_has_cache_key_tuples, dp_ignore, dp_inspectable, dp_inspectable_list, dp_multi, dp_multi_list, dp_named_ddl_element, dp_operator, dp_plain_dict, dp_plain_obj, dp_prefix_sequence, dp_propagate_attrs, dp_statement_hint_list, dp_string, dp_string_clauseelement_dict, dp_string_list, dp_string_multi_dict, dp_table_hint_list, dp_type, dp_unknown_structure

**类签名**

类 `sqlalchemy.sql.visitors.InternalTraversal` (`enum.Enum`)。

```py
attribute dp_annotations_key = 'AK'
```

访问 _annotations_cache_key 元素。

这是有关修改其角色的 ClauseElement 的其他信息的字典。在比较或缓存对象时应包括此信息，但是生成此键相对昂贵。在创建此键之前，访问者应首先检查“_annotations”字典是否为非 None。

```py
attribute dp_anon_name = 'AN'
```

访问可能“匿名化”的字符串值。

字符串值被视为缓存键生成的重要因素。

```py
attribute dp_boolean = 'B'
```

访问布尔值。

布尔值被视为缓存键生成的重要因素。

```py
attribute dp_clauseelement = 'CE'
```

访问 `ClauseElement` 对象。

```py
attribute dp_clauseelement_list = 'CL'
```

访问包含 `ClauseElement` 对象的列表。

```py
attribute dp_clauseelement_tuple = 'CT'
```

访问包含 `ClauseElement` 对象的元组。

```py
attribute dp_clauseelement_tuples = 'CTS'
```

访问包含 `ClauseElement` 对象的元组列表。

```py
attribute dp_dialect_options = 'DO'
```

访问方言选项结构。

```py
attribute dp_dml_multi_values = 'DML_MV'
```

访问 `Insert` 对象的字典的值（值为多个）。

```py
attribute dp_dml_ordered_values = 'DML_OV'
```

访问 `Update` 对象的有序元组列表的值。

```py
attribute dp_dml_values = 'DML_V'
```

访问 `ValuesBase`（例如 Insert 或 Update）对象的字典的值。

```py
attribute dp_fromclause_canonical_column_collection = 'FC'
```

访问 `FromClause` 对象的 `columns` 属性的上下文中。

列集合是“规范的”，这意味着它是 `ColumnClause` 对象的最初定义位置。目前这意味着正在访问的对象只能是 `TableClause` 或 `Table` 对象。

```py
attribute dp_fromclause_ordered_set = 'CO'
```

访问 `FromClause` 对象的有序集合。

```py
attribute dp_has_cache_key = 'HC'
```

访问 `HasCacheKey` 对象。

```py
attribute dp_has_cache_key_list = 'HL'
```

访问包含 `HasCacheKey` 对象的列表。

```py
attribute dp_has_cache_key_tuples = 'HT'
```

访问包含 `HasCacheKey` 对象的元组列表。

```py
attribute dp_ignore = 'IG'
```

指定应完全忽略的对象。

这目前适用于函数调用参数缓存，其中一些参数不应被视为缓存键的一部分。

```py
attribute dp_inspectable = 'IS'
```

访问可检查对象，其返回值是`HasCacheKey`对象。

```py
attribute dp_inspectable_list = 'IL'
```

访问可检查对象的列表，在检查后是`HasCacheKey`对象。

```py
attribute dp_multi = 'M'
```

访问可能是`HasCacheKey`或可能是普通可哈希对象的对象。

```py
attribute dp_multi_list = 'MT'
```

访问包含可能是`HasCacheKey`或可能是普通可哈希对象的元组。

```py
attribute dp_named_ddl_element = 'DD'
```

访问简单的命名 DDL 元素。

此方法使用的当前对象是`Sequence`。

该对象仅在缓存键生成中被认为是重要的，就其名称而言，但不涉及其它方面。

```py
attribute dp_operator = 'O'
```

访问一个运算符。

运算符是`sqlalchemy.sql.operators`模块中的函数。

运算符值被认为在缓存键生成中是重要的。

```py
attribute dp_plain_dict = 'PD'
```

访问具有字符串键的字典。

字典的键应该是字符串，值应该是不可变的和可哈希的。 字典被认为在缓存键生成中是重要的。

```py
attribute dp_plain_obj = 'PO'
```

访问普通的 Python 对象。

值应该是不可变的和可哈希的，例如整数。 值被认为在缓存键生成中是重要的。

```py
attribute dp_prefix_sequence = 'PS'
```

访问由`HasPrefixes`或`HasSuffixes`表示的序列。

```py
attribute dp_propagate_attrs = 'PA'
```

访问传播属性字典。 这是硬编码到我们目前关心的特定元素。

```py
attribute dp_statement_hint_list = 'SH'
```

访问`Select`对象的`_statement_hints`集合。

```py
attribute dp_string = 'S'
```

访问普通的字符串值。

例如，表名和列名，绑定参数键，特殊关键字如“UNION”，“UNION ALL”。

字符串值被认为在缓存键生成中是重要的。

```py
attribute dp_string_clauseelement_dict = 'CD'
```

访问具有字符串键到`ClauseElement`对象的字典。

```py
attribute dp_string_list = 'SL'
```

访问字符串列表。

```py
attribute dp_string_multi_dict = 'MD'
```

访问具有字符串键和值的字典，值可能是普通的不可变/可哈希的对象，也可能是`HasCacheKey`对象。

```py
attribute dp_table_hint_list = 'TH'
```

访问`Select`对象的`_hints`集合。

```py
attribute dp_type = 'T'
```

访问`TypeEngine`对象。

类型对象被认为对缓存键生成很重要。

```py
attribute dp_unknown_structure = 'UK'
```

访问一个未知的结构。

```py
class sqlalchemy.sql.visitors.Visitable
```

用于可访问对象的基类。

`Visitable` 用于实现 SQL 编译器分发函数。其他形式的遍历，例如用于缓存键生成的遍历，是使用 `HasTraverseInternals` 接口单独实现的。

在版本 2.0 中发生了变化：1.4 系列中的 `Visitable` 类被命名为 `Traversible`；该名称在 2.0 中改回了 `Visitable`，这是 1.4 之前的名称。

在 1.4 和 2.0 版本中，这两个名称仍然可导入。

```py
attribute sqlalchemy.sql.visitors.anon_map
```

`cache_anon_map` 的别名

```py
function sqlalchemy.sql.visitors.cloned_traverse(obj: ExternallyTraversible | None, opts: Mapping[str, Any], visitors: Mapping[str, Callable[[Any], None]]) → ExternallyTraversible | None
```

克隆给定的表达式结构，允许访问者修改可变对象。

遍历用法与 `traverse()` 相同。`visitors` 字典中的访问者函数也可以在遍历过程中修改给定结构的内部。

`cloned_traverse()` 函数**不会**提供属于`Immutable`接口的对象给访问方法（这主要包括 `ColumnClause`、`Column`、`TableClause` 和 `Table` 对象）。由于此遍历仅旨在允许对象的原地突变，因此跳过`Immutable`对象。仍然在每个对象上调用 `Immutable._clone()` 方法，以允许对象根据其子内部的克隆替换自身为不同的对象（例如，一个克隆其子查询以返回一个新的 `ColumnClause` 的 `ColumnClause`）。

在版本 2.0 中发生了变化：`cloned_traverse()` 函数省略了属于`Immutable`接口的对象。

除了用于实现迭代的 `ClauseElement.get_children()` 函数外，`cloned_traverse()` 和 `replacement_traverse()` 函数使用的中心 API 特性是 `ClauseElement._copy_internals()` 方法。要正确支持克隆和替换遍历的 `ClauseElement` 结构，它需要能够将克隆函数传递给其内部成员，以便对其进行复制。

另请参阅

`traverse()`

`replacement_traverse()`

```py
function sqlalchemy.sql.visitors.iterate(obj: ExternallyTraversible | None, opts: Mapping[str, Any] = {}) → Iterator[ExternallyTraversible]
```

遍历给定的表达式结构，返回一个迭代器。

遍历配置为广度优先。

`iterate()` 函数使用的中心 API 特性是 `ClauseElement.get_children()` 方法，用于 `ClauseElement` 对象。该方法应返回与特定 `ClauseElement` 对象关联的所有 `ClauseElement` 对象。例如，`Case` 结构将在其 “whens” 和 “else_” 成员变量中引用一系列 `ColumnElement` 对象。

参数：

+   `obj` – 要遍历的 `ClauseElement` 结构

+   `opts` – 迭代选项的字典。在现代用法中，此字典通常为空。

```py
function sqlalchemy.sql.visitors.replacement_traverse(obj: ExternallyTraversible | None, opts: Mapping[str, Any], replace: _TraverseTransformCallableType[Any]) → ExternallyTraversible | None
```

克隆给定的表达式结构，允许通过给定的替换函数进行元素替换。

此函数与`cloned_traverse()`函数非常相似，不同之处在于，该函数不是被传递一个访问者字典，而是所有元素都无条件地传递给给定的替换函数。然后，替换函数可以选择返回一个完全新的对象，该对象将替换给定的对象。如果返回`None`，则保留对象在原位。

`cloned_traverse()`和`replacement_traverse()`之间的使用差异在于，在前一种情况下，已克隆的对象被传递给访问者函数，然后访问者函数可以操作对象的内部状态。在后一种情况下，访问者函数应该只返回一个完全不同的对象，或者什么也不做。

`replacement_traverse()`的用例是在 SQL 结构内部用不同的 FROM 子句替换一个 FROM 子句，这是 ORM 中常见的用例。

```py
function sqlalchemy.sql.visitors.traverse(obj: ExternallyTraversible | None, opts: Mapping[str, Any], visitors: Mapping[str, Callable[[Any], None]]) → ExternallyTraversible | None
```

使用默认迭代器遍历和访问给定的表达式结构。

> 例如：
> 
> ```py
> from sqlalchemy.sql import visitors
> 
> stmt = select(some_table).where(some_table.c.foo == 'bar')
> 
> def visit_bindparam(bind_param):
>     print("found bound value: %s" % bind_param.value)
> 
> visitors.traverse(stmt, {}, {"bindparam": visit_bindparam})
> ```

对象的迭代使用`iterate()`函数，该函数使用堆栈进行广度优先遍历。

参数：

+   `obj` – 要遍历的`ClauseElement`结构

+   `opts` – 迭代选项的字典。在现代用法中，该字典通常为空。

+   `visitors` – 访问函数的字典。该字典应该有字符串作为键，每个键对应于特定类型的 SQL 表达式对象的`__visit_name__`，并且可调用的函数作为值，每个值代表该类型对象的访问函数。

```py
function sqlalchemy.sql.visitors.traverse_using(iterator: Iterable[ExternallyTraversible], obj: ExternallyTraversible | None, visitors: Mapping[str, Callable[[Any], None]]) → ExternallyTraversible | None
```

使用给定的对象迭代器访问给定的表达式结构。

`traverse_using()`通常在内部作为`traverse()`函数的结果而调用。

参数：

+   `iterator` – 一个可迭代或序列，它将生成`ClauseElement`结构；假定该迭代器是`iterate()`函数的产品。

+   `obj` – 作为`iterate()`函数目标使用的`ClauseElement`。

+   `visitors` – 访问函数的字典。有关此字典的详细信息，请参见`traverse()`。

另请参阅

`traverse()`
