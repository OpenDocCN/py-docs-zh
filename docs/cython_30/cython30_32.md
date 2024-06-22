# 融合类型（模板）

> 原文： [`docs.cython.org/en/latest/src/userguide/fusedtypes.html`](http://docs.cython.org/en/latest/src/userguide/fusedtypes.html)

融合类型允许您有一个可以引用多种类型的类型定义。这允许您编写一个静态类型的 cython 算法，该算法可以对多种类型的值进行操作。因此，融合类型允许[泛型编程](https://en.wikipedia.org/wiki/Generic_programming)，类似于 C ++中的模板或 Java / C＃等语言中的泛型。

注意

目前不支持融合类型作为扩展类型的属性。只能使用融合类型声明变量和函数/方法参数。

## 快速入门

```py
from __future__ import print_function

ctypedef fused char_or_float:
    char
    float

cpdef char_or_float plus_one(char_or_float var):
    return var + 1

def show_me():
    cdef:
        char a = 127
        float b = 127
    print('char', plus_one(a))
    print('float', plus_one(b))

```

这给出了：

```py
>>> show_me()
char -128
float 128.0

```

`plus_one(a)`将“融合型”`char_or_float`“专门化”为`char`，而`plus_one(b)`将`char_or_float`专门化为`float`。

## 声明熔断类型

融合类型可以声明如下：

```py
cimport cython

ctypedef fused my_fused_type:
    cython.int
    cython.double

```

这声明了一个名为`my_fused_type`的新类型，它可以是和`int` _ 或 _ a `double`。或者，声明可以写成：

```py
my_fused_type = cython.fused_type(cython.int, cython.float)

```

只有名称可用于组成类型，但它们可以是任何（非融合）类型，包括 typedef。即可以写：

```py
ctypedef double my_double
my_fused_type = cython.fused_type(cython.int, my_double)

```

## 使用融合类型

融合类型可用于声明函数或方法的参数：

```py
cdef cfunc(my_fused_type arg):
    return arg + 1

```

如果在参数列表中多次使用相同的融合类型，则融合类型的每个特化必须相同：

```py
cdef cfunc(my_fused_type arg1, my_fused_type arg2):
    return cython.typeof(arg1) == cython.typeof(arg2)

```

在这种情况下，两个参数的类型都是 int 或 double（根据前面的示例）。但是，因为这些参数使用相同的融合类型`my_fused_type`，所以`arg1`和`arg2`都专用于相同类型。因此，对于每个可能的有效调用，此函数都返回 True。但是你可以混合融合类型：

```py
def func(A x, B y):
    ...

```

其中`A`和`B`是不同的融合类型。这将为`A`和`B`中包含的所有类型组合生成专门的代码路径。

### 融合类型和数组

请注意，仅数字类型的特化可能不是非常有用，因为通常可以依赖于类型的提升。但是，对于内存的数组，指针和类型化视图，情况并非如此。的确，有人可能写道：

```py
def myfunc(A[:, :] x):
    ...

# and

cdef otherfunc(A *x):
    ...

```

请注意，在 Cython 0.20.x 及更早版本中，当类型签名中的多个内存视图使用融合类型时，编译器会生成所有类型组合的完整交叉积。

```py
def myfunc(A[:] a, A[:] b):
    # a and b had independent item types in Cython 0.20.x and earlier.
    ...

```

这对于大多数用户来说是出乎意料的，不太可能是期望的，并且与其他结构化类型声明（例如融合类型的 C 数组）不一致，这些声明被认为是相同的类型。因此在 Cython 0.21 中进行了更改，以便对融合类型的所有内存视图使用相同的类型。为了获得原始行为，只需在不同的名称下声明相同的融合类型，然后在声明中使用它们：

```py
ctypedef fused A:
    int
    long

ctypedef fused B:
    int
    long

def myfunc(A[:] a, B[:] b):
    # a and b are independent types here and may have different item types
    ...

```

要在较旧的 Cython 版本（0.21 之前版本）中仅获得相同类型，可以使用`ctypedef`：

```py
ctypedef A[:] A_1d

def myfunc(A_1d a, A_1d b):
    # a and b have identical item types here, also in older Cython versions
    ...

```

## 选择专业化

您可以通过两种方式选择特化（具有特定或专用（即非融合）参数类型的函数实例）：通过索引或通过调用。

### 索引

您可以使用类型索引函数以获得某些特化，即：

```py
cfunccython.p_double

# From Cython space
funcfloat, double

# From Python space
funccython.float, cython.double

```

如果使用融合类型作为基类型，这将意味着基类型是融合类型，因此基类型需要专门化：

```py
cdef myfunc(A *x):
    ...

# Specialize using int, not int *
myfuncint

```

### 调用

也可以使用参数调用融合函数，其中自动计算调度：

```py
cfunc(p1, p2)
func(myfloat, mydouble)

```

对于从 Cython 调用的`cdef`或`cpdef`函数，这意味着在编译时计算出特化。对于`def`函数，在运行时对参数进行类型检查，并执行尽力而为的方法来确定需要哪种特化。这意味着如果没有找到特化，这可能会导致运行时`TypeError`。如果函数的类型未知，则`cpdef`函数的处理方式与`def`函数的处理方式相同（例如，如果它是外部的，并且没有 cimport）。

自动调度规则通常如下所示，按优先顺序排列：

*   试着找到完全匹配
*   选择最大的相应数值类型（最大浮点数，最大复数，最大 int）

## 内置熔断类型

为方便起见，有一些内置的融合类型，它们是：

```py
cython.integral # short, int, long
cython.floating # float, double
cython.numeric  # short, int, long, float, double, float complex, double complex

```

## 铸造熔断函数

融合的`cdef`和`cpdef`函数可以转换或分配给 C 函数指针，如下所示：

```py
cdef myfunc(cython.floating, cython.integral):
    ...

# assign directly
cdef object (*funcp)(float, int)
funcp = myfunc
funcp(f, i)

# alternatively, cast it
(<object (*)(float, int)> myfunc)(f, i)

# This is also valid
funcp = myfunc[float, int]
funcp(f, i)

```

## 类型检查专业化

可以基于融合参数的特化来做出决定。修剪错误条件以避免无效代码。可以检查`is`，`is not`和`==`和`!=`以查看融合类型是否等于某个其他非融合类型（检查专业化），或使用`in`和 COD5 判断专门化是否是另一组类型（指定为融合类型）的一部分。例如：

```py
ctypedef fused bunch_of_types:
    ...

ctypedef fused string_t:
    cython.p_char
    bytes
    unicode

cdef cython.integral myfunc(cython.integral i, bunch_of_types s):
    cdef int *int_pointer
    cdef long *long_pointer

    # Only one of these branches will be compiled for each specialization!
    if cython.integral is int:
        int_pointer = &i
    else:
        long_pointer = &i

    if bunch_of_types in string_t:
        print("s is a string!")

```

## 条件 GIL 获取/释放

获取和释放 GIL 可以通过编译时已知的条件来控制（参见 [条件获取/释放 GIL）。

当与融合类型结合使用时，这是最有用的。融合类型函数可能必须处理 cython 本机类​​型（例如 cython.int 或 cython.double）和 python 类型（例如对象或字节）。条件获取/释放 GIL 提供了一种运行相同代码的方法，无论是发布 GIL（对于 cython 本机类​​型）还是持有 GIL（对于 python 类型）：

```py
cimport cython

ctypedef fused double_or_object:
    cython.double
    object

def increment(double_or_object x):
    with nogil(double_or_object is cython.double):
        # Same code handles both cython.double (GIL is released)
        # and python object (GIL is not released).
        x = x + 1
    return x

```

## __signatures__

最后，来自`def`或`cpdef`函数的函数对象具有 __signatures__ 属性，该属性将签名字符串映射到实际的专用函数。这可能对检查有用。列出的签名字符串也可以用作融合函数的索引，但索引格式可能会在 Cython 版本之间发生变化：

```py
specialized_function = fused_function["MyExtensionClass|int|float"]

```

通常最好像这样索引，但是：

```py
specialized_function = fused_function[MyExtensionClass, int, float]

```

虽然后者将从 Python 空间中选择`int`和`float`的最大类型，因为它们不是类型标识符，而是内置类型。但是，通过`cython.int`和`cython.float`可以解决这个问题。

对于来自 python 空间的 memoryview 索引，我们可以执行以下操作：

```py
ctypedef fused my_fused_type:
    int[:, ::1]
    float[:, ::1]

def func(my_fused_type array):
    ...

my_fused_type[cython.int[:, ::1]](myarray)

```

使用例如同样的方法也是如此。 `cython.numeric[:, :]`。