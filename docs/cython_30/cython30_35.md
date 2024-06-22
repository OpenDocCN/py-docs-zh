# Cython 和 Pyrex 之间的区别

> 原文： [`docs.cython.org/en/latest/src/userguide/pyrex_differences.html`](http://docs.cython.org/en/latest/src/userguide/pyrex_differences.html)

警告

Cython 和 Pyrex 都是移动目标。已经到了这一点，两个项目之间所有差异的明确列表将很难列出和跟踪，但希望这个高级列表能够了解存在的差异。应该注意的是，两个项目都努力相互兼容，但 Cython 的目标是尽可能接近并完成 Python 的合理性。

## Python 3 支持

Cython 创建了`.c`文件，可以在 Python 2.x 和 Python 3.x 中构建和使用。实际上，使用 Cython 编译模块很可能是将代码移植到 Python 3 的简单方法。

Cython 还支持 Python 3.0 和后来的主要 Python 版本附带的各种语法添加。如果它们不与现有的 Python 2.x 语法或语义冲突，它们通常只是被编译器接受。其他一切都取决于编译器指令`language_level=3`（参见 编译器指令 ）。

### List / Set / Dict 理解

Cython 支持 Python 3 为列表，集和 dicts 定义的不同理解：

```py
[expr(x) for x in A]             # list
{expr(x) for x in A}             # set
{key(x) : value(x) for x in A}   # dict

```

如果`A`是列表，元组或字典，则优化循环。您也可以使用 [`for`](https://docs.python.org/3/reference/compound_stmts.html#for "(in Python v3.7)") ... [`from`](https://docs.python.org/3/reference/simple_stmts.html#from "(in Python v3.7)") 语法，但通常最好使用通常的 [`for`](https://docs.python.org/3/reference/compound_stmts.html#for "(in Python v3.7)") ... [`in`](https://docs.python.org/3/reference/expressions.html#in "(in Python v3.7)") `range(...)`语法与 C 运行变量（例如`cdef int i`）。

注意

见 自动量程转换

请注意，Cython 还支持从 Python 2.4 开始的集合文字。

### 仅关键字参数

Python 函数可以在`*`参数之后和`**`参数之前列出仅限关键字的参数，例如：

```py
def f(a, b, *args, c, d = 42, e, **kwds):
    ...

```

这里`c`，`d`和`e`不能作为位置参数传递，必须作为关键字参数传递。此外，`c`和`e`是必需的关键字参数，因为它们没有默认值。

如果省略`*`之后的参数名，则该函数将不接受任何额外的位置参数，例如：

```py
def g(a, b, *, c, d):
    ...

```

采用两个位置参数，并有两个必需的关键字参数。

## 条件表达式“x if b else y”

[`www.python.org/dev/peps/pep-0308/`](https://www.python.org/dev/peps/pep-0308/) 中描述的条件表达式：

```py
X if C else Y

```

仅评估`X`和`Y`中的一个（取决于 C 的值）。

## cdef inline

模块级函数现在可以内联声明， `inline` 关键字传递给 C 编译器。这些可以和宏一样快：

```py
cdef inline int something_fast(int a, int b):
    return a*a + b

```

请注意，类级 `cdef` 函数是通过虚函数表处理的，因此几乎在所有情况下编译器都无法内联它们。

## 声明作业（例如“cdef int spam = 5”）

在 Pyrex 中，必须写：

```py
cdef int i, j, k
i = 2
j = 5
k = 7

```

现在，有了 cython，人们可以写：

```py
cdef int i = 2, j = 5, k = 7

```

右边的表达可以任意复杂，例如：

```py
cdef int n = python_call(foo(x,y), a + b + c) - 32

```

## for 循环中的'by'表达式（例如“for i from 0＆lt; = i＆lt; 10 by 2”）

```py
for i from 0 <= i < 10 by 2:
    print i

```

收益率：

```py
0
2
4
6
8

```

Note

不鼓励使用此语法，因为它对于普通的 Python [`for`](https://docs.python.org/3/reference/compound_stmts.html#for "(in Python v3.7)") 循环来说是多余的。参见 自动量程转换 。

## 布尔 int 类型（例如，它的行为类似于 c int，但是作为布尔值强制转换为/来自 python）

在 C 中，int 用于真值。在 python 中，任何对象都可以用作真值（使用`__nonzero__()`方法），但规范选择是两个布尔对象`True`和`False`。 `bint`（用于“boolean int”）类型被编译为 C int，但是作为布尔值强制进出 Python。比较的返回类型和几个内置函数也是`bint`。这减少了在`bool()`中包装物品的需要。例如，人们可以写：

```py
def is_equal(x):
    return x == y

```

它将在 Pyrex 中返回`1`或`0`，但在 Cython 中返回`True`或`False`。可以声明函数的变量和返回值为`bint`类型。例如：

```py
cdef int i = x
cdef bint b = x

```

第一次转换将通过`x.__int__()`进行，而第二次转换将通过`x.__bool__()`（a.k.a。`__nonzero__()`）进行，并对已知的内置类型进行适当的优化。

## 可执行类主体

包括工作 [`classmethod()`](https://docs.python.org/3/library/functions.html#classmethod "(in Python v3.7)") ：

```py
cdef class Blah:
    def some_method(self):
        print self
    some_method = classmethod(some_method)
    a = 2*3
    print "hi", a

```

## cpdef 函数

Cython 在通常的 [`def`](https://docs.python.org/3/reference/compound_stmts.html#def "(in Python v3.7)") 和 `cdef` 之上添加了第三种功能类型。如果声明了一个函数 `cpdef` ，它可以被扩展和普通的 python 子类调用和覆盖。您基本上可以将 `cpdef` 方法视为 `cdef` 方法+一些额外的方法。 （这就是它至少实现的方式。）首先，它创建了一个 [`def`](https://docs.python.org/3/reference/compound_stmts.html#def "(in Python v3.7)") 方法，除了调用底层 `cdef` 方法之外什么都不做（并且如果需要，可以进行参数解包/强制） ）。在 `cdef` 方法的顶部添加了一些代码以查看它是否被覆盖，类似于以下伪代码：

```py
if hasattr(type(self), '__dict__'):
    foo = self.foo
    if foo is not wrapper_foo:
        return foo(args)
[cdef method body]

```

要检测类型是否具有字典，它只检查`tp_dictoffset`插槽，对于扩展类型，它是`NULL`（默认情况下），但对于实例类，则为非空。如果字典存在，它会执行单个属性查找，并且可以（通过比较指针）判断返回的结果是否实际上是新函数。如果且仅当它是一个新函数时，则参数被打包到元组和方法中。这一切都非常快。设置了一个标志，因此如果直接在类上调用方法，则不会发生此查找，例如：

```py
cdef class A:
    cpdef foo(self):
        pass

x = A()
x.foo()  # will check to see if overridden
A.foo(x) # will call A's implementation whether overridden or not

```

有关说明和使用提示，请参阅 早期绑定速度 。

## 自动量程转换

当`i`是任何 cdef'd 整数类型时，这将把`for i in range(...)`形式的语句转换为`for i from ...`，并且可以确定方向（即步骤的符号）。

Warning

如果范围导致赋值给`i`溢出，这可能会改变语义。具体来说，如果设置了此选项，则在输入循环之前将引发错误，而如果没有此选项，循环将执行，直到遇到溢出值。如果这会影响你，请更改`Cython/Compiler/Options.py`（最终会有更好的方法来设置它）。

## 更友好的类型铸造

在 Pyrex 中，如果有一个类型`&lt;int&gt;x`，其中`x`是一个 Python 对象，那么将获得`x`的内存地址。同样，如果一个类型`&lt;object&gt;i`，其中`i`是一个 C int，那么将在内存中的`i`位置获得一个“对象”。这会导致混乱的结果和段错误。

在 Cython `&lt;type&gt;x`中，如果其中一个类型是 python 对象，则会尝试强制执行（如将`x`赋值给类型类型的变量时）。它不会阻止一个没有转换的地方（虽然它会发出警告）。如果一个人真的想要这个地址，首先要转换为`void *`。

如在 Pyrex 中`&lt;MyExtensionType&gt;x`将`x`转换为类型`MyExtensionType`而不进行任何类型检查。 Cython 支持使用类型检查进行转换的语法`&lt;MyExtensionType?&gt;`（即如果`x`不是`MyExtensionType`的子类，它将抛出错误。

## cdef / cpdef 函数中的可选参数

Cython 现在支持 `cdef` 和 `cpdef` 功能的可选参数。

`.pyx`文件中的语法保留在 Python 中，但是通过写`cdef foo(x=*)`在`.pxd`文件中声明了这些函数。子类化时参数的数量可能会增加，但参数类型和顺序必须保持不变。在某些情况下，如果没有任何可选项的 cdef / cpdef 函数被一个具有默认参数值的函数覆盖，则会有轻微的性能损失。

例如，可以有`.pxd`文件：

```py
cdef class A:
    cdef foo(self)

cdef class B(A):
    cdef foo(self, x=*)

cdef class C(B):
    cpdef foo(self, x=*, int k=*)

```

使用相应的`.pyx`文件：

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

Note

这也证明了 `cpdef` 功能如何覆盖 `cdef` 功能。

## 结构中的函数指针

为方便起见， `struct` 中声明的函数会自动转换为函数指针。

## C ++异常处理

`cdef` 函数现在可以声明为：

```py
cdef int foo(...) except +
cdef int foo(...) except +TypeError
cdef int foo(...) except +python_error_raising_function

```

在这种情况下，捕获 C ++错误时将引发 Python 异常。有关详细信息，请参阅 在 Cython 中使用 C ++。

## 同义词

`cdef import from`与`cdef extern from`的含义相同

## 源代码编码

Cython 支持  [**PEP 3120** ](https://www.python.org/dev/peps/pep-3120)和  [**PEP 263** ](https://www.python.org/dev/peps/pep-0263)，即你可以开始你的 Cython 带有编码注释的源文件，通常用 UTF-8 编写源代码。这会影响字节字符串的编码以及将`u'abcd'`等 unicode 字符串文字转换为 unicode 对象。

## 自动`typecheck`

而不是像 [Pyrex 文档](https://www.cosc.canterbury.ac.nz/greg.ewing/python/Pyrex/version/Doc/Manual/special_methods.html)中所解释的那样引入新关键字`typecheck`，只要 [`isinstance()`](https://docs.python.org/3/library/functions.html#isinstance "(in Python v3.7)") 与扩展类型一起使用，Cython 就会发出（非欺骗性和更快速）的类型检查第二个参数。

## 来自 *_future_* 指令

Cython 支持多种`from *_future_* import ...`指令，即`absolute_import`，`unicode_literals`，`print_function`和`division`。

始终启用语句。

## 纯 Python 模式

Cython 支持编译`.py`文件，并使用装饰器和其他有效的 Python 语法接受类型注释。这允许将相同的源解释为直接 Python，或者编译以获得优化结果。有关详细信息，请参阅 纯 Python 模式 。