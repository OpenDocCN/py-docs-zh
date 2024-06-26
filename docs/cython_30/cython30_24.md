# 语言基础

> 原文： [`docs.cython.org/en/latest/src/userguide/language_basics.html`](http://docs.cython.org/en/latest/src/userguide/language_basics.html)

## 声明数据类型

作为一种动态语言，Python 鼓励一种编程风格，即在方法和属性方面考虑类和对象，而不是它们适合类层次结构。

这可以使 Python 成为一种非常轻松和舒适的快速开发语言，但需要付出代价 - 管理数据类型的“繁文缛节”会被转储到解释器上。在运行时，解释器在搜索命名空间，获取属性以及解析参数和关键字元组方面做了大量工作。与“早期绑定”语言（如 C ++）相比，这种运行时“后期绑定”是 Python 相对缓慢的主要原因。

然而，使用 Cython，通过使用“早期绑定”编程技术可以获得显着的加速。

注意

打字不是必需品

为参数和变量提供静态类型可以方便地加速代码，但这不是必需的。优化需要的地点和时间。实际上，在键入不允许优化但 Cython 仍然需要检查某个对象的类型是否与声明的类型匹配的情况下，键入 *可以减慢* 你的代码。

## C 变量和类型定义

`cdef` 语句用于声明 C 变量，无论是本地变量还是模块级变量：

```py
cdef int i, j, k
cdef float f, g[42], *h

```

和 `struct` ， `union` 或 `enum` 类型：

```py
cdef struct Grail:
    int age
    float volume

cdef union Food:
    char *spam
    float *eggs

cdef enum CheeseType:
    cheddar, edam,
    camembert

cdef enum CheeseState:
    hard = 1
    soft = 2
    runny = 3

```

另见 struct，union 和 enum 声明的样式

Note

结构可以声明为`cdef packed struct`，其效果与 C 指令`#pragma pack(1)`相同。

将枚举声明为`cpdef`将创建  [**PEP 435** ](https://www.python.org/dev/peps/pep-0435)风格的 Python 包装器：

```py
cpdef enum CheeseState:
    hard = 1
    soft = 2
    runny = 3

```

目前没有用于定义常量的特殊语法，但您可以使用匿名 `enum` 声明来实现此目的，例如：

```py
cdef enum:
    tons_of_spam = 3

```

Note

单词`struct`，`union`和`enum`仅在定义类型时使用，而不是在引用时使用。例如，要声明一个指向`Grail`的变量，您将编写：

```py
cdef Grail *gp

```

并不是：

```py
cdef struct Grail *gp # WRONG

```

还有一个`ctypedef`语句用于为类型命名，例如：

```py
ctypedef unsigned long ULong

ctypedef int* IntPtr

```

也可以用 `cdef` 声明函数，使它们成为 c 函数。

```py
cdef int eggs(unsigned long l, float f):
    ...

```

您可以在 Python 函数与 C 函数 中阅读更多相关信息。

您可以使用 `cdef` 声明类，使它们成为 扩展类型 。那些将具有非常接近 python 类的行为，但速度更快，因为它们在内部使用`struct`来存储属性。

这是一个简单的例子：

```py
from __future__ import print_function

cdef class Shrubbery:
    cdef int width, height

    def __init__(self, w, h):
        self.width = w
        self.height = h

    def describe(self):
        print("This shrubbery is", self.width,
              "by", self.height, "cubits.")

```

您可以在 扩展类型 中阅读更多相关信息。

### 类型

Cython 使用 C 类型的常规 C 语法，包括指针。它提供所有标准 C 类型，即`char`，`short`，`int`，`long`，`long long`以及它们的`unsigned`版本，例如`unsigned int`。特殊`bint`类型用于 C 布尔值（`int`，其中 0 /非 0 值为 False / True）和`Py_ssize_t`用于（容器）的（签名）大小。

通过将`*`附加到它们指向的基本类型，例如，在 C 中构造指针类型。 `int**`指向指向 C int 的指针。数组使用普通的 C 数组语法，例如， `int[10]`，并且在编译时必须知道堆栈分配的数组的大小。 Cython 不支持 C99 的可变长度数组。请注意，Cython 使用数组访问进行指针解除引用，因为`*x`不是有效的 Python 语法，而`x[0]`是。

此外，Python 类型`list`，`dict`，`tuple`等可用于静态类型，以及任何用户定义的 扩展类型 。例如：

```py
cdef list foo = []

```

这需要类的 *精确* 匹配，它不允许子类。这允许 Cython 通过访问内置类的内部来优化代码。对于这种类型的输入，Cython 在内部使用`PyObject*`类型的 C 变量。 Python 类型 int，long 和 float 不能用于静态类型，而是分别解释为 C `int`，`long`和`float`，因为使用这些 Python 类型的静态类型变量没有任何优势。

Cython 提供了一个加速和类型化的 Python 元组，即`ctuple`。 `ctuple`由任何有效的 C 类型组装而成。例如：

```py
cdef (double, int) bar

```

它们编译成 C 结构，可以用作 Python 元组的有效替代品。

虽然这些 C 类型可以非常快，但它们具有 C 语义。具体来说，整数类型溢出，C `float`类型只有 32 位精度（而不是 Python 浮动包装的 64 位 C `double`，而且通常是你想要的）。如果要使用这些数字 Python 类型，只需省略类型声明并将它们作为对象。

也可以声明 扩展类型 （用`cdef class`声明）。这确实允许子类。此类型主要用于访问`cdef`方法和扩展类型的属性。 C 代码使用一个变量，它是指向特定类型结构的指针，类似于`struct MyExtensionTypeObject*`。

### 分组多个 C 声明

如果您有一系列声明都以 `cdef` 开头，您可以将它们分组为 `cdef` 块，如下所示：

```py
from __future__ import print_function

cdef:
    struct Spam:
        int tons

    int i
    float a
    Spam *p

    void f(Spam *s):
        print(s.tons, "Tons of spam")

```

## Python 函数与 C 函数

Cython 中有两种函数定义：

Python 函数是使用 def 语句定义的，就像在 Python 中一样。它们将 Python 对象作为参数并返回 Python 对象。

C 函数使用新的 `cdef` 语句定义。它们将 Python 对象或 C 值作为参数，并且可以返回 Python 对象或 C 值。

在 Cython 模块中，Python 函数和 C 函数可以自由地相互调用，但只能通过解释的 Python 代码从模块外部调用 Python 函数。因此，您希望从 Cython 模块“导出”的任何函数都必须使用 def 声明为 Python 函数。还有一种称为 `cpdef` 的混合功能。可以从任何地方调用 `cpdef` ，但在从其他 Cython 代码调用时使用更快的 C 调用约定。 `cpdef` 也可以在子类或实例属性上被 Python 方法覆盖，即使从 Cython 调用也是如此。如果发生这种情况，大多数性能提升当然都会丢失，即使它没有，与调用 `cdef` 相比，从 Cython 调用 `cpdef` 方法的开销很小。 ] 方法。

可以使用常规 C 声明语法将任一类型的函数的参数声明为具有 C 数据类型。例如，：

```py
def spam(int i, char *s):
    ...

cdef int eggs(unsigned long l, float f):
    ...

```

也可以使用`ctuples`：

```py
cdef (int, float) chips((long, long, double) t):
    ...

```

当 Python 函数的参数声明为具有 C 数据类型时，它将作为 Python 对象传入，并在可能的情况下自动转换为 C 值。换句话说，上面`spam`的定义等同于写作：

```py
def spam(python_i, python_s):
    cdef int i = python_i
    cdef char* s = python_s
    ...

```

目前，只能对数字类型，字符串类型和结构进行自动转换（以递归方式组合任何这些类型）;尝试将任何其他类型用于 Python 函数的参数将导致编译时错误。如果要在调用后使用指针，必须小心使用字符串以确保引用。可以从 Python 映射中获取结构，如果要在函数返回后使用它们，则必须再次使用字符串属性。

另一方面，C 函数可以具有任何类型的参数，因为它们是使用普通的 C 函数调用直接传递的。

使用 `cdef` 与 Python 对象返回类型声明的函数（如 Python 函数）将在执行离开函数体时返回`None`值而没有显式返回值。这与 C / C ++形成对比，后者使返回值未定义。在非 Python 对象返回类型的情况下，返回等效于零的值，例如，对于`int`为 0，对于`bint`为`False`，对于指针类型为`NULL`。

可以在 早期结合速度 中找到这些不同方法类型的优缺点的更完整比较。

### Python 对象作为参数和返回值

如果没有为参数或返回值指定类型，则假定它是 Python 对象。 （请注意，这与 C 约定不同，它默认为 int。）例如，下面定义了一个 C 函数，它将两个 Python 对象作为参数并返回一个 Python 对象：

```py
cdef spamobjs(x, y):
    ...

```

根据标准 Python / C API 规则自动执行这些对象的引用计数（即，借用的引用被视为参数并返回新引用）。

> 警告
> 
> 这仅适用于 Cython 代码。在 C 中实现的其他 Python 包（如 NumPy）可能不遵循这些约定。

name 对象也可用于显式声明某些内容为 Python 对象。如果声明的名称将被视为类型的名称，这可能很有用，例如：

```py
cdef ftang(object int):
    ...

```

声明一个名为 int 的参数，它是一个 Python 对象。您还可以使用 object 作为函数的显式返回类型，例如：

```py
cdef object ftang(object int):
    ...

```

为了清楚起见，始终明确 C 函数中的对象参数可能是个好主意。

要创建借用引用，请将参数类型指定为`PyObject*`。 Cython 不会执行自动`Py_INCREF`或`Py_DECREF`，例如：

将显示：

```py
Initial refcount: 2
Inside owned_reference: 3
Inside borrowed_reference: 2

```

### 可选参数

与 C 不同，可以在`cdef`和`cpdef`函数中使用可选参数。但是，无论是在`.pyx`文件还是相应的`.pxd`文件中声明它们，都存在差异。

为避免重复（以及潜在的未来不一致），默认参数值在声明中（在`.pxd`文件中）不可见，但仅在实现中（在`.pyx`文件中）。

在`.pyx`文件中，签名与 Python 本身的签名相同：

```py
from __future__ import print_function

cdef class A:
    cdef foo(self):
        print("A")

cdef class B(A):
    cdef foo(self, x=None):
        print("B", x)

cdef class C(B):
    cpdef foo(self, x=True, int k=3):
        print("C", x, k)

```

在`.pxd`文件中，签名与此示例不同：`cdef foo(x=*)`。这是因为调用函数的程序只需知道 C 中可能的签名，但不需要知道默认参数的值：

```py
cdef class A:
    cdef foo(self)

cdef class B(A):
    cdef foo(self, x=*)

cdef class C(B):
    cpdef foo(self, x=*, int k=*)

```

Note

子类化时参数的数量可能会增加，但 arg 类型和顺序必须相同，如上例所示。

当使用没有默认值的可选 arg 覆盖可选 arg 时，可能会有轻微的性能损失。

### 仅关键字参数

与在 Python 3 中一样，`def`函数可以在`"*"`参数之后和`"**"`参数之前列出仅限关键字的参数（如果有）：

```py
def f(a, b, *args, c, d = 42, e, **kwds):
    ...

# We cannot call f with less verbosity than this.
foo = f(4, "bar", c=68, e=1.0)

```

如上所示，`c`，`d`和`e`参数不能作为位置参数传递，必须作为关键字参数传递。此外，`c`和`e`是**必需**关键字参数，因为它们没有默认值。

没有参数名称的单个`"*"`可用于终止位置参数列表：

```py
def g(a, b, *, c, d):
    ...

# We cannot call g with less verbosity than this.
foo = g(4.0, "something", c=68, d="other")

```

如上所示，签名只需要两个位置参数，并且有两个必需的关键字参数。

### 功能指针

在`struct`中声明的函数会自动转换为函数指针。

有关使用带有功能指针的错误返回值，请参见 错误返回值 底部的注释。

### 错误返回值

如果你没有做任何特殊的事情，用 `cdef` 声明的不返回 Python 对象的函数无法向其调用者报告 Python 异常。如果在此类函数中检测到异常，则会打印警告消息并忽略该异常。

如果你想要一个不返回 Python 对象的 C 函数能够将异常传播给它的调用者，你需要为它声明一个异常值。这是一个例子：

```py
cdef int spam() except -1:
    ...

```

使用此声明，每当垃圾邮件内发生异常时，它将立即返回值`-1`。此外，每当对垃圾邮件的调用返回`-1`时，将假定已发生异常并将进行传播。

声明函数的异常值时，不应显式或隐式返回该值。特别是，如果异常返回值是`False`值，那么您应该确保该函数永远不会通过隐式或空返回终止。

如果所有可能的返回值都是合法的，并且您不能完全为信号错误保留一个，则可以使用另一种形式的异常值声明：

```py
cdef int spam() except? -1:
    ...

```

“？”表示值`-1`仅表示可能的错误。在这种情况下，如果返回异常值，Cython 会生成对 [`PyErr_Occurred()`](https://docs.python.org/3/c-api/exceptions.html#c.PyErr_Occurred "(in Python v3.7)") 的调用，以确保它确实是一个错误。

还有第三种形式的例外价值声明：

```py
cdef int spam() except *:
    ...

```

这种形式导致 Cython 在每次调用垃圾邮件后生成对 [`PyErr_Occurred()`](https://docs.python.org/3/c-api/exceptions.html#c.PyErr_Occurred "(in Python v3.7)") 的调用，无论它返回什么值。如果您有一个返回 void 的函数需要传播错误，则必须使用此表单，因为没有任何要测试的返回值。否则，此表格几乎没有用处。

可以使用以下命令声明可能引发异常的外部 C ++函数：

```py
cdef int spam() except +

```

有关详细信息，请参阅 在 Cython 中使用 C ++。

有些事情需要注意：

*   异常值只能为返回整数，枚举，浮点或指针类型的函数声明，并且值必须是常量表达式。 Void 函数只能使用`except *`表单。

*   异常值规范是函数签名的一部分。如果您将指向函数的指针作为参数传递或将其指定给变量，则声明的参数或变量类型必须具有相同的异常值规范（或缺少该规范）。以下是带有异常值的指针到函数声明的示例：

    ```py
    int (*grail)(int, char*) except -1

    ```

*   您不需要（也不应该）为返回 Python 对象的函数声明异常值。请记住，没有声明返回类型的函数会隐式返回 Python 对象。 （通过返回 NULL 隐式传播此类函数的异常。）

### 检查非 Cython 函数的返回值

重要的是要理解，当返回指定的值时，except 子句不会引发错误。例如，你不能写像：

```py
cdef extern FILE *fopen(char *filename, char *mode) except NULL # WRONG!

```

并且如果对`fopen()`的调用返回`NULL`，则期望自动引发异常。 except 子句不起作用;它的唯一目的是传播已经引发的 Python 异常，可以通过 Cython 函数或调用 Python / C API 例程的 C 函数来传播。要从诸如`fopen()`的非 Python 感知函数中获取异常，您必须检查返回值并自行引发它，例如：

```py
from libc.stdio cimport FILE, fopen
from libc.stdlib cimport malloc, free
from cpython.exc cimport PyErr_SetFromErrnoWithFilenameObject

def open_file():
    cdef FILE* p
    p = fopen("spam.txt", "r")
    if p is NULL:
        PyErr_SetFromErrnoWithFilenameObject(OSError, "spam.txt")
    ...

def allocating_memory(number=10):
    cdef double *my_array = <double *> malloc(number * sizeof(double))
    if not my_array:  # same as 'is NULL' above
        raise MemoryError()
    ...
    free(my_array)

```

### 覆盖扩展类型

`cpdef`方法可以覆盖`cdef`方法：

```py
from __future__ import print_function

cdef class A:
    cdef foo(self):
        print("A")

cdef class B(A):
    cdef foo(self, x=None):
        print("B", x)

cdef class C(B):
    cpdef foo(self, x=True, int k=3):
        print("C", x, k)

```

当使用 Python 类继承扩展类型时，`def`方法可以覆盖`cpdef`方法但不能覆盖`cdef`方法：

```py
from __future__ import print_function

cdef class A:
    cdef foo(self):
        print("A")

cdef class B(A):
    cpdef foo(self):
        print("B")

class C(B):  # NOTE: not cdef class
    def foo(self):
        print("C")

```

如果上面的`C`是扩展类型（`cdef class`），这将无法正常工作。在这种情况下，Cython 编译器将发出警告。

## 自动类型转换

在大多数情况下，当在需要 C 值的上下文中使用 Python 对象时，将对基本数字和字符串类型执行自动转换，反之亦然。下表总结了转换的可能性。

<colgroup><col width="42%"> <col width="30%"> <col width="27%"></colgroup> 
| C 类型 | 从 Python 类型 | 到 Python 类型 |
| --- | --- | --- |
| [unsigned] char，[unsigned] short，int，long | int，long | INT |
| unsigned int，unsigned long，[unsigned] long long | int, long | 长 |
| 浮，双，长双 | int，long，float | 浮动 |
| 字符* | STR /字节 | str / bytes [[3]](#id13) |
| C 数组 | 迭代 | 清单 [[5]](#id15) |
| 结构，联合 |  | 字典 [[4]](#id14) |

<colgroup><col class="label"><col></colgroup>
| [[3]](#id10) | 对于 Python 2.x，转换为/来自 str，对于 Python 3.x，转换为字节。 |

<colgroup><col class="label"><col></colgroup>
| [[4]](#id12) | 从 C union 类型到 Python dict 的转换将为每个 union 字段添加一个值。但是，Cython 0.23 及更高版本将拒绝自动转换具有不安全类型组合的联合。一个例子是`int`和`char*`的并集，在这种情况下，指针值可能是也可能不是有效指针。 |

<colgroup><col class="label"><col></colgroup>
| [[5]](#id11) | 除了 signed / unsigned char []。如果在编译时未知 C 数组的长度，并且使用 C 数组的切片，则转换将失败。 |

### 在 C 语境中使用 Python 字符串时的注意事项

在期望`char*`的上下文中使用 Python 字符串时需要小心。在这种情况下，使用指向 Python 字符串内容的指针，只有 Python 字符串存在时才有效。因此，只要需要 C 字符串，就需要确保保留对原始 Python 字符串的引用。如果您不能保证 Python 字符串的存活时间足够长，则需要复制 C 字符串。

Cython 检测并防止这种错误。例如，如果您尝试以下内容：

```py
cdef char *s
s = pystring1 + pystring2

```

然后 Cython 将产生错误消息`Obtaining char* from temporary Python value`。原因是连接两个 Python 字符串会产生一个新的 Python 字符串对象，该对象仅由 Cython 生成的临时内部变量引用。语句完成后，临时变量将被删除，Python 字符串被释放，`s`悬空。由于此代码无法工作，因此 Cython 拒绝编译它。

解决方案是将串联的结果赋给 Python 变量，然后从中获取`char*`，即：

```py
cdef char *s
p = pystring1 + pystring2
s = p

```

然后，您有责任在必要时保留参考 p。

请记住，用于检测此类错误的规则仅是启发式。有时 Cython 会不必要地抱怨，有时它会无法检测到存在的问题。最终，您需要了解问题并小心您的工作。

### 类型转换

C 使用`"("`和`")"`，Cython 使用`"&lt;"`和`"&gt;"`。例如：

```py
cdef char *p
cdef float *q
p = <char*>q

```

将 C 值转换为 Python 对象类型或反之时，Cython 将尝试强制。简单示例是类似`&lt;int&gt;pyobj`的转换，它将 Python 编号转换为普通的 C `int`值，或者`&lt;bytes&gt;charptr`，它将 C `char*`字符串复制到新的 Python 字节对象中。

> Note
> 
> Cython 不会阻止冗余演员，但会发出警告。

要获取某些 Python 对象的地址，请使用强制转换为`&lt;void*&gt;`或`&lt;PyObject*&gt;`等指针类型。您还可以使用`&lt;object&gt;`或更具体的内置或扩展类型（例如`&lt;MyExtType&gt;ptr`）将 C 指针强制转换回 Python 对象引用。这将使对象的引用计数增加 1，即转换返回拥有的引用。这是一个例子：

```py
from cpython.ref cimport PyObject

cdef extern from *:
    ctypedef Py_ssize_t Py_intptr_t

python_string = "foo"

cdef void* ptr = <void*>python_string
cdef Py_intptr_t adress_in_c = <Py_intptr_t>ptr
address_from_void = adress_in_c        # address_from_void is a python int

cdef PyObject* ptr2 = <PyObject*>python_string
cdef Py_intptr_t address_in_c2 = <Py_intptr_t>ptr2
address_from_PyObject = address_in_c2  # address_from_PyObject is a python int

assert address_from_void == address_from_PyObject == id(python_string)

print(<object>ptr)                     # Prints "foo"
print(<object>ptr2)                    # prints "foo"

```

`&lt;...&gt;`的优先级是`&lt;type&gt;a.b.c`被解释为`&lt;type&gt;(a.b.c)`。

转换为`&lt;object&gt;`会创建一个自有引用。 Cython 将自动执行`Py_INCREF`和`Py_DECREF`操作。转换为`&lt;PyObject *&gt;`会创建一个借用的引用，保持引用不变。

### 检查类型转换

像`&lt;MyExtensionType&gt;x`这样的强制转换会将 x 转换为类`MyExtensionType`而不进行任何检查。

要检查强制转换，请使用如下语法：`&lt;MyExtensionType?&gt;x`。在这种情况下，如果`x`不是`MyExtensionType`的实例，Cython 将应用运行时检查，该检查会引发`TypeError`。这将测试内置类型的确切类，但允许 扩展类型 的子类。

## 陈述和表达

控件结构和表达式大部分都遵循 Python 语法。当应用于 Python 对象时，它们具有与 Python 相同的语义（除非另有说明）。大多数 Python 运算符也可以应用于 C 值，具有明显的语义。

如果在表达式中混合使用 Python 对象和 C 值，则会在 Python 对象和 C 数字或字符串类型之间自动执行转换。

为所有 Python 对象自动维护引用计数，并自动检查所有 Python 操作是否有错误，并采取适当的操作。

### C 和 Cython 表达式之间的差异

C 表达式和 Cython 表达式之间在语法和语义上存在一些差异，特别是在 C 语言结构领域，它们在 Python 中没有直接的等价物。

*   整数文字被视为 C 常量，并将被截断为 C 编译器认为合适的任何大小。获取 Python 整数（任意精度）立即转换为对象（例如`&lt;object&gt;100000000000000000000`）。 `L`，`LL`和`U`后缀与 C 中的含义相同。

*   Cython 中没有`-&gt;`运算符。而不是`p-&gt;x`，使用`p.x`

*   在 Cython 中没有一元`*`运算符。而不是`*p`，使用`p[0]`

*   有一个`&`运算符，其语义与 C 中相同。

*   空 C 指针称为`NULL`，而不是`0`（`NULL`是保留字）。

*   类型转换被写为`&lt;type&gt;value`，例如：

    ```py
    cdef char* p, float* q
    p = &lt;char*&gt;q

    ```

### 范围规则

Cython 完全静态地确定变量是属于本地范围，模块范围还是内置范围。与 Python 一样，分配给未以其他方式声明隐式声明的变量将其声明为驻留在分配范围内的变量。变量的类型取决于类型推断，但全局模块范围除外，它始终是 Python 对象。

### 内置函数

Cython 将对大多数内置函数的调用编译为对相应 Python / C API 例程的直接调用，使得它们特别快。

仅优化使用这些名称的直接函数调用。如果你使用其中一个名称假设它是 Python 对象，比如将其分配给 Python 变量，然后调用它，那么调用将作为 Python 函数调用。

<colgroup><col width="42%"> <col width="18%"> <col width="39%"></colgroup> 
| 功能和参数 | 返回类型 | Python / C API 等效 |
| --- | --- | --- |
| ABS（OBJ） | 对象，双，...... | PyNumber_Absolute，fabs，fabsf，...... |
| 可调用（OBJ） | 宾特 | PyObject_Callable |
| delattr（obj，名字） | 没有 | PyObject_DelAttr |
| exec（代码，[glob，[loc]]） | 宾语 |  |
| DIR（OBJ） | 名单 | PyObject_Dir |
| divmod（a，b） | 元组 | PyNumber_Divmod |
| getattr（obj，name，[default]）（注 1） | object | PyObject_GetAttr |
| hasattr（obj，name） | bint | PyObject_HasAttr |
| 哈希（OBJ） | int / long | PyObject_Hash |
| 实习生（OBJ） | object | PY * _InternFromString |
| isinstance（obj，type） | bint | PyObject_IsInstance |
| issubclass（obj，type） | bint | PyObject_IsSubclass |
| iter（obj，[sentinel]） | object | PyObject_GetIter |
| LEN（OBJ） | Py_ssize_t | PyObject_Length |
| pow（x，y，[z]） | object | PyNumber_Power |
| 重装（OBJ） | object | PyImport_ReloadModule |
| 再版（OBJ） | object | PyObject_Repr |
| setattr（obj，name） | 空虚 | PyObject_SetAttr |

注 1：Pyrex 最初提供了一个函数`getattr3(obj, name, default)()`，对应于 Python 内置 [`getattr()`](https://docs.python.org/3/library/functions.html#getattr "(in Python v3.7)") 的三参数形式。 Cython 仍然支持这个功能，但不赞成使用普通的内置，而 Cython 可以在两种形式中进行优化。

### 运算符优先级

请记住，Python 和 C 之间的运算符优先级存在一些差异，并且 Cython 使用 Python 优先级，而不是 C 优先级。

### 整数 for 循环

Cython 识别通常的 Python for-in-range 整数循环模式：

```py
for i in range(n):
    ...

```

如果`i`被声明为 `cdef` 整数类型，它会将其优化为纯 C 循环。需要此限制，否则由于目标体系结构上的潜在整数溢出，生成的代码将不正确。如果您担心循环未正确转换，请使用 cython 命令行（`-a`）的 annotate 功能轻松查看生成的 C 代码。见 自动量程转换

为了向后兼容 Pyrex，Cython 还支持更详细的 for 循环形式，您可以在遗留代码中找到它：

```py
for i from 0 <= i < n:
    ...

```

要么：

```py
for i from 0 <= i < n by s:
    ...

```

其中`s`是一个整数步长。

Note

不推荐使用此语法，不应在新代码中使用。请使用普通的 Python for 循环。

有关 for-from 循环的一些注意事项：

*   目标表达式必须是普通变量名称。
*   下限和上限之间的名称必须与目标名称相同。
*   迭代的方向由关系决定。如果它们都来自集合{`&lt;`，`&lt;=`}则它是向上的;如果他们都来自集合{`&gt;`，`&gt;=`}那么它是向下的。 （不允许任何其他组合。）

与其他 Python 循环语句一样，break 和 continue 可以在 body 中使用，循环可以有 else 子句。

## Cython 文件类型

Cython 中有三种文件类型：

*   实现文件，带有`.py`或`.pyx`后缀。
*   定义文件，带有`.pxd`后缀。
*   包含文件，带有`.pxi`后缀。

### 实现文件

顾名思义，实现文件包含函数，类，扩展类型等的实现。此文件几乎支持所有 python 语法。大多数情况下，`.py`文件可以在不更改任何代码的情况下重命名为`.pyx`文件，Cython 将保留 python 行为。

Cython 可以编译`.py`和`.pyx`文件。如果只想使用 Python 语法，则文件名不重要，Cython 不会根据使用的后缀更改生成的代码。但是，如果想要使用 Cython 语法，则需要使用`.pyx`文件。

除了 Python 语法之外，用户还可以利用 Cython 语法（例如`cdef`）来使用 C 变量，可以将函数声明为`cdef`或`cpdef`，并可以使用 `cimport`导入 C 定义。在本页和 Cython 文档的其余部分中可以找到许多其他可用于实现文件的 Cython 功能。

如果相应的定义文件也定义了该类型，则对某些 扩展类型 的实现部分有一些限制。

Note

编译`.pyx`文件时，Cython 首先检查相应的`.pxd`文件是否存在并首先处理它。它就像一个 Cython `.pyx`文件的头文件。您可以放入其他 Cython 模块将使用的内部函数。这允许不同的 Cython 模块在没有 Python 开销的情况下使用彼此的函数和类。要了解更多有关如何操作的信息，可以看 pxd 文件 。

### 定义文件

定义文件用于声明各种事物。

可以进行任何 C 声明，它也可以是 C / C ++文件中实现的 C 变量或函数的声明。这可以通过`cdef extern from`完成。有时，`.pxd`文件用作 C / C ++头文件到 Cython 可以理解的语法的转换。这允许 C / C ++变量和函数直接用于 `cimport` 的实现文件中。您可以在 与外部 C 代码 和 的接口中使用 Cython 中的 C ++阅读更多相关信息。

它还可以包含扩展类型的定义部分和外部库的函数声明。

它不能包含任何 C 或 Python 函数的实现，也不能包含任何 Python 类定义或任何可执行语句。当想要访问 `cdef` 属性和方法，或从本模块中定义的 `cdef` 类继承时，需要它。

Note

您不需要（也不应该）在声明文件 `public` 中声明任何内容，以使其可用于其他 Cython 模块;它只是存在于定义文件中。如果要为外部 C 代码提供某些内容，则只需要公开声明。

### include 语句和包含文件

Warning

历史上，`include`语句用于共享声明。使用 在 Cython 模块 之间共享声明。

Cython 源文件可以使用 include 语句包含来自其他文件的材料，例如：

```py
include "spamstuff.pxi"

```

指定文件的内容在文本上包含在该点。包含的文件可以包含在 include 语句出现的上下文中有效的任何完整语句或声明，包括其他 include 语句。包含文件的内容应该以缩进级别零开始，并且将被视为缩进到包含该文件的 include 语句的级别。但是，include 语句不能在模块范围之外使用，例如在函数或类主体内部。

Note

还有其他机制可用于将 Cython 代码拆分为单独的部分，在许多情况下可能更合适。参见 在 Cython 模块 之间共享声明。

## 条件编译

某些功能可用于 Cython 源文件中的条件编译和编译时常量。

### 编译时定义

可以使用 DEF 语句定义编译时常量：

```py
DEF FavouriteFood = u"spam"
DEF ArraySize = 42
DEF OtherArraySize = 2 * ArraySize + 17

```

`DEF`的右侧必须是有效的编译时表达式。这些表达式由使用`DEF`语句定义的文字值和名称组成，使用任何 Python 表达式语法进行组合。

预定义了以下编译时名称，对应于 [`os.uname()`](https://docs.python.org/3/library/os.html#os.uname "(in Python v3.7)") 返回的值。

> UNAME_SYSNAME，UNAME_NODENAME，UNAME_RELEASE，UNAME_VERSION，UNAME_MACHINE

还提供以下内置常量和函数选择：

> None，True，False，abs，all，any，ascii，bin，bool，bytearray，bytes，chr，cmp，complex，dict，divmod，enumerate，filter，float，format，frozenset，hash，hex，int ，len，list，long，map，max，min，oct，ord，pow，range，reduce，repr，reverse，round，set，slice，sorted，str，sum，tuple，xrange，zip

请注意，在 Python 2.x 或 3.x 下编译时，其中一些内置函数可能不可用，或者两者中的行为可能不同。

使用`DEF`定义的名称可以在标识符出现的任何地方使用，并且用其编译时值替换，就好像它在那时作为文字写入源中一样。为此，编译时表达式必须求值为`int`，`long`，`float`，`bytes`或`unicode`（Py3 中的`str`）类型的 Python 值。

```py
from __future__ import print_function

DEF FavouriteFood = u"spam"
DEF ArraySize = 42
DEF OtherArraySize = 2 * ArraySize + 17

cdef int a1[ArraySize]
cdef int a2[OtherArraySize]
print("I like", FavouriteFood)

```

### 条件语句

`IF`语句可用于在编译时有条件地包含或排除代码段。它的工作方式与 C 中的`#if`预处理程序指令类似：

```py
IF UNAME_SYSNAME == "Windows":
    include "icky_definitions.pxi"
ELIF UNAME_SYSNAME == "Darwin":
    include "nice_definitions.pxi"
ELIF UNAME_SYSNAME == "Linux":
    include "penguin_definitions.pxi"
ELSE:
    include "other_definitions.pxi"

```

`ELIF`和`ELSE`子句是可选的。 `IF`语句可以出现在正常语句或声明可以出现的任何地方，并且它可以包含在该上下文中有效的任何语句或声明，包括`DEF`语句和其他`IF`语句。

`IF`和`ELIF`子句中的表达式必须是`DEF`语句的有效编译时表达式，尽管它们可以计算任何 Python 值，并且结果的真实性以通常的 Python 方式确定。