# 扩展类型的特殊方法

> 原文： [`docs.cython.org/en/latest/src/userguide/special_methods.html`](http://docs.cython.org/en/latest/src/userguide/special_methods.html)

本页介绍了 Cython 扩展类型当前支持的特殊方法。所有特殊方法的完整列表显示在底部的表格中。其中一些方法的行为与 Python 对应方式不同，或者没有直接的 Python 对应方式，需要特别提及。

注意

此页面上说的所有内容仅适用于使用`cdef class`语句定义的扩展类型。它不适用于使用 Python [`class`](https://docs.python.org/3/reference/compound_stmts.html#class "(in Python v3.7)") 语句定义的类，其中适用普通的 Python 规则。

## 声明

必须使用 [`def`](https://docs.python.org/3/reference/compound_stmts.html#def "(in Python v3.7)") 而不是 `cdef` 声明扩展类型的特殊方法。这不会影响它们的性能 - Python 使用不同的调用约定来调用这些特殊方法。

## Docstrings

目前，某些扩展类型的特殊方法并未完全支持 docstrings。您可以在源中放置 docstring 作为注释，但在运行时它不会显示在相应的`__doc__`属性中。 （这似乎是一个 Python 限制 - 在 &lt;cite&gt;PyTypeObject&lt;/cite&gt; 数据结构中没有任何地方放置这样的文档字符串。）

## 初始化方法：`__cinit__()`和`__init__()`

初始化对象有两种方法。

您应该在`__cinit__()`方法中执行对象的基本 C 级初始化，包括分配您的对象将拥有的任何 C 数据结构。您需要注意在`__cinit__()`方法中执行的操作，因为在调用对象时，该对象可能还不是完全有效的 Python 对象。因此，您应该小心地调用可能触及该对象的任何 Python 操作;特别是它的方法和任何可以被子类型覆盖的东西（因此依赖于它们已经初始化的子类型状态）。

在调用`__cinit__()`方法时，已为该对象分配了内存，并且已将其初始化为 0 或 null 的任何 C 属性。 （任何 Python 属性也已初始化为 None，但您可能不应该依赖它。）保证您的`__cinit__()`方法只能被调用一次。

如果扩展类型具有基类型，则在`__cinit__()`方法之前会自动调用基类型层次结构中的任何现有`__cinit__()`方法。您无法显式调用继承的`__cinit__()`方法，基类型可以自由选择是否实现`__cinit__()`。如果需要将修改后的参数列表传递给基类型，则必须在`__init__()`方法中执行初始化的相关部分，而不是调用继承方法的常规规则。

任何无法在`__cinit__()`方法中安全完成的初始化都应在`__init__()`方法中完成。到`__init__()`被调用时，该对象是一个完全有效的 Python 对象，所有操作都是安全的。在某些情况下，可能会多次调用`__init__()`或根本不调用`__init__()`，因此在这种情况下，您的其他方法应该设计得很稳健。

传递给构造函数的任何参数都将传递给`__cinit__()`方法和`__init__()`方法。如果你期望在 Python 中继承你的扩展类型，你可能会发现给`__cinit__()`方法 &lt;cite&gt;*&lt;/cite&gt; 和 &lt;cite&gt;**&lt;/cite&gt; 参数是有用的，这样它就可以接受并忽略额外的参数。否则，任何具有不同签名的`__init__()`的 Python 子类都必须覆盖`__new__()` [[1]](#id4) 以及`__init__()`，Python 类的编写者不会期望得做。或者，为方便起见，如果您声明`__cinit__`()`方法不带参数（除了 self），它将忽略传递给构造函数的任何额外参数，而不会抱怨签名不匹配。

Note

所有构造函数参数都将作为 Python 对象传递。这意味着不可转换的 C 类型（如指针或 C ++对象）无法从 Cython 代码传递到构造函数中。如果需要，请使用工厂函数来处理对象初始化。直接调用此函数中的`__new__()`以绕过对`__init__()`构造函数的调用通常会有所帮助。

有关示例，请参阅现有 C / C ++指针 中的 实例化。

<colgroup><col class="label"><col></colgroup>
| [[1]](#id3) | [`docs.python.org/reference/datamodel.html#object.__new__`](https://docs.python.org/reference/datamodel.html#object.__new__) |

## 定稿方法：`__dealloc__()`

`__cinit__()`方法的对应物是`__dealloc__()`方法，它应该执行`__cinit__()`方法的反转。您`__cinit__()`方法中明确分配的任何 C 数据（例如通过 malloc）都应该在`__dealloc__()`方法中释放。

您需要注意在`__dealloc__()`方法中执行的操作。在调用`__dealloc__()`方法时，对象可能已经被部分破坏，并且就 Python 而言可能不处于有效状态，因此您应该避免调用可能触及该对象的任何 Python 操作。特别是，不要调用对象的任何其他方法或做任何可能导致对象复活的事情。如果你坚持只是解除分配 C 数据是最好的。

您无需担心释放对象的 Python 属性，因为在`__dealloc__()`方法返回后，Cython 将为您完成此操作。

在对扩展类型进行子类化时，请注意，即使重写了超类的`__dealloc__()`方法，也始终会调用它。这与典型的 Python 行为形成对比，在这种行为中，除非子类显式调用超类方法，否则不会执行超类方法。

Note

扩展类型没有`__del__()`方法。

## 算术方法

算术运算符方法（如`__add__()`）的行为与 Python 对应方法不同。这些方法没有单独的“反向”版本（`__radd__()`等）。相反，如果第一个操作数不能执行操作，则调用第二个操作数的相同方法，操作数的顺序相同。

这意味着您不能依赖这些方法的第一个参数是“自”或正确的类型，并且您应该在决定做什么之前测试两个操作数的类型。如果您无法处理已经给出的类型组合，则应返回 &lt;cite&gt;NotImplemented&lt;/cite&gt; 。

这也适用于就地算术方法`__ipow__()`。它不适用于任何其他始终将 &lt;cite&gt;self&lt;/cite&gt; 作为第一个参数的原位方法（`__iadd__()`等）。

## 丰富的比较

有两种方法可以实现比较方法。根据应用程序的不同，这种方式可能更好：

*   第一种方法使用 6 Python [特殊方法](https://docs.python.org/3/reference/datamodel.html#basic-customization) `__eq__()`，`__lt__()`等。这是 Cython 0.27 以来的新功能，与普通 Python 类完全一样。

*   第二种方法使用单一特殊方法`__richcmp__()`。这在一个方法中实现了所有丰富的比较操作。签名是`def __richcmp__(self, other, int op)`。整数参数`op`表示要执行的操作，如下表所示：

    &lt;colgroup&gt;&lt;col width="42%"&gt; &lt;col width="58%"&gt;&lt;/colgroup&gt; 
    | ＆LT; | Py_LT |
    | == | Py_EQ |
    | ＆GT; | Py_GT |
    | ＆LT = | Py_LE |
    | ！= | Py_NE |
    | ＆GT; = | Py_GE |

    这些常量可以从`cpython.object`模块中导入。

## `__next__()`方法

希望实现迭代器接口的扩展类型应该定义一个名为`__next__()`的方法，而不是下一个方法。 Python 系统将自动提供调用`__next__()`的下一个方法。 *NOT* 是否明确地给你的类型一个`next()`方法，否则可能会发生坏事。

## 特殊方法表

此表列出了所有特殊方法及其参数和返回类型。在下表中，self 的参数名称用于指示参数具有方法所属的类型。表中未指定类型的其他参数是通用 Python 对象。

您不必将方法声明为采用这些参数类型。如果您声明了不同的类型，则会根据需要执行转换。

### 一般

[`docs.python.org/3/reference/datamodel.html#special-method-names`](https://docs.python.org/3/reference/datamodel.html#special-method-names)

<colgroup><col width="18%"> <col width="30%"> <col width="10%"> <col width="41%"></colgroup> 
| 名称 | 参数 | 返回类型 | 描述 |
| --- | --- | --- | --- |
| *_cinit_* | 自我，...... |  | 基本初始化（没有直接的 Python 等价物） |
| **在里面** | self, … |   | 进一步初始化 |
| *_dealloc_* | 自 |   | 基本的释放（没有直接的 Python 等价物） |
| *_cmp_* | x，y | INT | 3 路比较（仅限 Python 2） |
| *_str_* | self | 宾语 | STR（个体经营） |
| *_repr_* | self | object | 再版（个体经营） |
| *_hash_* | self | Py_hash_t | 散列函数（返回 32/64 位整数） |
| **呼叫** | self, … | object | 自（…） |
| *_iter_* | self | object | 返回序列的迭代器 |
| *_getattr_* | 自我，名字 | object | 获取属性 |
| *_getattribute_* | self, name | object | 无条件地获取属性 |
| *_setattr_* | 自我，名字，val |   | 设置属性 |
| *_delattr_* | self, name |   | 删除属性 |

### 丰富的比较运算符

[`docs.python.org/3/reference/datamodel.html#basic-customization`](https://docs.python.org/3/reference/datamodel.html#basic-customization)

您可以选择实现标准的 Python 特殊方法，如`__eq__()`或单个特殊方法`__richcmp__()`。根据应用，一种方式或另一种方式可能更好。

<colgroup><col width="18%"> <col width="30%"> <col width="10%"> <col width="43%"></colgroup> 
| Name | Parameters | Return type | Description |
| --- | --- | --- | --- |
| *_eq_* | 自我，是的 | object | 自我== y |
| *_ne_* | self, y | object | self！= y（如果没有，则回退到`__eq__`） |
| *_lt_* | self, y | object | 自我＆lt; ÿ |
| *_gt_* | self, y | object | 自我＆gt; ÿ |
| *_le_* | self, y | object | 自我＆lt; = y |
| *_ge_* | self, y | object | 自我＆gt; = y |
| *_richcmp_* | self，y，int op | object | 为上述所有内容加入了丰富的比较方法（没有直接的 Python 等价物） |

### 算术运算符

[`docs.python.org/3/reference/datamodel.html#emulating-numeric-types`](https://docs.python.org/3/reference/datamodel.html#emulating-numeric-types)

<colgroup><col width="18%"> <col width="30%"> <col width="10%"> <col width="41%"></colgroup> 
| Name | Parameters | Return type | Description |
| --- | --- | --- | --- |
| **加** | x, y | object | 二进制 &lt;cite&gt;+&lt;/cite&gt; 运算符 |
| *_sub_* | x, y | object | 二进制 &lt;cite&gt;-&lt;/cite&gt; 运算符 |
| *_mul_* | x, y | object | &lt;cite&gt;*&lt;/cite&gt; 运算符 |
| *_div_* | x, y | object | &lt;cite&gt;/&lt;/cite&gt; 运算符用于旧式划分 |
| *_floordiv_* | x, y | object | &lt;cite&gt;//&lt;/cite&gt; 运算符 |
| *_truediv_* | x, y | object | &lt;cite&gt;/&lt;/cite&gt; 运算符用于新式划分 |
| *_mod_* | x, y | object | &lt;cite&gt;％&lt;/cite&gt;运算符 |
| *_divmod_* | x, y | object | 组合 div 和 mod |
| *_pow_* | x，y，z | object | &lt;cite&gt;**&lt;/cite&gt; 运算符或 pow（x，y，z） |
| *_neg_* | self | object | 一元 &lt;cite&gt;-&lt;/cite&gt; 运算符 |
| *_pos_* | self | object | 一元 &lt;cite&gt;+&lt;/cite&gt; 运算符 |
| *_abs_* | self | object | 绝对值 |
| *_nonzero_* | self | int | 转换为布尔值 |
| **倒置** | self | object | &lt;cite&gt;〜&lt;/cite&gt;运算符 |
| *_lshift_* | x, y | object | &lt;cite&gt;＆lt;＆lt;&lt;/cite&gt; 运营商 |
| *_rshift_* | x, y | object | &lt;cite&gt;＆gt;＆gt;&lt;/cite&gt; 运营商 |
| **和** | x, y | object | &lt;cite&gt;＆amp;&lt;/cite&gt; 运营商 |
| **要么** | x, y | object | &lt;cite&gt;&#124;&lt;/cite&gt; 运营商 |
| *_xor_* | x, y | object | &lt;cite&gt;^&lt;/cite&gt; 运算符 |

### 数字转换

[`docs.python.org/3/reference/datamodel.html#emulating-numeric-types`](https://docs.python.org/3/reference/datamodel.html#emulating-numeric-types)

<colgroup><col width="18%"> <col width="30%"> <col width="10%"> <col width="41%"></colgroup> 
| Name | Parameters | Return type | Description |
| --- | --- | --- | --- |
| *_int_* | self | object | 转换为整数 |
| **长** | self | object | 转换为长整数 |
| **浮动** | self | object | 转换为浮动 |
| *_oct_* | self | object | 转换为八进制 |
| *_hex_* | self | object | 转换为十六进制 |
| __index__（仅限 2.5+） | self | object | 转换为序列索引 |

### 就地算术运算符

[`docs.python.org/3/reference/datamodel.html#emulating-numeric-types`](https://docs.python.org/3/reference/datamodel.html#emulating-numeric-types)

<colgroup><col width="18%"> <col width="30%"> <col width="10%"> <col width="41%"></colgroup> 
| Name | Parameters | Return type | Description |
| --- | --- | --- | --- |
| **我加** | 自我，x | object | &lt;cite&gt;+ =&lt;/cite&gt; 运算符 |
| *_isub_* | self, x | object | &lt;cite&gt;- =&lt;/cite&gt; 运算符 |
| *_imul_* | self, x | object | &lt;cite&gt;* =&lt;/cite&gt; 运算符 |
| *_idiv_* | self, x | object | &lt;cite&gt;/ =&lt;/cite&gt; 运算符用于旧式划分 |
| *_ifloordiv_* | self, x | object | &lt;cite&gt;// =&lt;/cite&gt; 运算符 |
| *_truediv_* | self, x | object | &lt;cite&gt;/ =&lt;/cite&gt; 运算符用于新式划分 |
| *_imod_* | self, x | object | &lt;cite&gt;％=&lt;/cite&gt; 运算符 |
| *_pow_* | x, y, z | object | &lt;cite&gt;** =&lt;/cite&gt; 运算符 |
| *_ilshift_* | self, x | object | &lt;cite&gt;＆lt;＆lt; =&lt;/cite&gt; 运算符 |
| *_irshift_* | self, x | object | &lt;cite&gt;＆gt;＆gt; =&lt;/cite&gt; 运算符 |
| **我和** | self, x | object | &lt;cite&gt;＆amp; =&lt;/cite&gt; 运算符 |
| *_ior_* | self, x | object | &lt;cite&gt;&#124; =&lt;/cite&gt; 运算符 |
| *_ixor_* | self, x | object | &lt;cite&gt;^ =&lt;/cite&gt; 运算符 |

### 序列和映射

[`docs.python.org/3/reference/datamodel.html#emulating-container-types`](https://docs.python.org/3/reference/datamodel.html#emulating-container-types)

<colgroup><col width="18%"> <col width="30%"> <col width="10%"> <col width="41%"></colgroup> 
| Name | Parameters | Return type | Description |
| --- | --- | --- | --- |
| *_len_* | self | Py_ssize_t | LEN（个体经营） |
| *_getitem_* | self, x | object | 自[X] |
| *_setitem_* | 自我，x，y |   | self [x] = y |
| *_delitem_* | self, x |   | del self [x] |
| *_getslice_* | self，Py_ssize_t i，Py_ssize_t j | object | 自[I：j]的 |
| *_setslice_* | self，Py_ssize_t i，Py_ssize_t j，x |   | 自我[i：j] = x |
| *_delslice_* | self, Py_ssize_t i, Py_ssize_t j |   | del self [i：j] |
| *_contains_* | self, x | int | x 在自我中 |

### 迭代器

[`docs.python.org/3/reference/datamodel.html#emulating-container-types`](https://docs.python.org/3/reference/datamodel.html#emulating-container-types)

<colgroup><col width="18%"> <col width="30%"> <col width="10%"> <col width="41%"></colgroup> 
| Name | Parameters | Return type | Description |
| --- | --- | --- | --- |
| **下一个** | self | object | 获取下一个项目（在 Python 中称为 next） |

### 缓冲接口[  [**PEP 3118** ](https://www.python.org/dev/peps/pep-3118)]（没有 Python 当量 - 见注 1）

<colgroup><col width="18%"> <col width="30%"> <col width="10%"> <col width="41%"></colgroup> 
| Name | Parameters | Return type | Description |
| --- | --- | --- | --- |
| *_getbuffer_* | self，Py_buffer &lt;cite&gt;* view&lt;/cite&gt; ，int flags |   |   |
| *_releasebuffer_* | self，Py_buffer &lt;cite&gt;* view&lt;/cite&gt; |   |   |

### 缓冲接口[遗留]（没有 Python 等价物 - 见注 1）

<colgroup><col width="18%"> <col width="30%"> <col width="10%"> <col width="41%"></colgroup> 
| Name | Parameters | Return type | Description |
| --- | --- | --- | --- |
| *_getreadbuffer_* | self，Py_ssize_t i，void &lt;cite&gt;** p&lt;/cite&gt; |   |   |
| *_getwritebuffer_* | self, Py_ssize_t i, void &lt;cite&gt;**p&lt;/cite&gt; |   |   |
| *_getsegcount_* | self，Py_ssize_t &lt;cite&gt;* p&lt;/cite&gt; |   |   |
| *_getcharbuffer_* | self，Py_ssize_t i，char &lt;cite&gt;** p&lt;/cite&gt; |   |   |

### 描述符对象（见注 2）

[`docs.python.org/3/reference/datamodel.html#emulating-container-types`](https://docs.python.org/3/reference/datamodel.html#emulating-container-types)

<colgroup><col width="18%"> <col width="30%"> <col width="10%"> <col width="41%"></colgroup> 
| Name | Parameters | Return type | Description |
| --- | --- | --- | --- |
| **得到** | 自我，实例，阶级 | object | 获取属性的值 |
| **组** | 自我，实例，价值 |   | 设置属性值 |
| **删除** | 自我，实例 |   | Delete attribute |

Note

（1）缓冲区接口旨在供 C 代码使用，不能直接从 Python 访问。它在 Python 2.x 的 Python / C API 参考手册的 6.6 和 10.6 节中描述。它被 Python 2.6 中新的  [**PEP 3118** ](https://www.python.org/dev/peps/pep-3118)缓冲协议取代，并且在 Python 3 中不再可用。有关新 API 的操作指南，见 实现缓冲协议 。

Note

（2）描述符对象是新式 Python 类的支持机制的一部分。请参阅 Python 文档中对描述符的讨论。另见  [**PEP 252** ](https://www.python.org/dev/peps/pep-0252)，“使类型看起来更像类”，  [**PEP 253** ](https://www.python.org/dev/peps/pep-0253)，“Subtyping Built-In Types”。