# 纯 Python 模式

> 原文： [`docs.cython.org/en/latest/src/tutorial/pure.html`](http://docs.cython.org/en/latest/src/tutorial/pure.html)

在某些情况下，需要加速 Python 代码而不会失去使用 Python 解释器运行它的能力。虽然可以使用 Cython 编译纯 Python 脚本，但通常只能获得大约 20％-50％的速度增益。

为了超越这一点，Cython 提供了语言结构，为 Python 模块添加静态类型和 cythonic 功能，使其在编译时运行得更快，同时仍允许对其进行解释。这是通过增加`.pxd`文件，通过 Python 类型注释（在 [PEP 484](https://www.python.org/dev/peps/pep-0484/) 和 [PEP 526](https://www.python.org/dev/peps/pep-0526/) 之后）和/或通过导入魔法后可用的特殊函数和装饰器来实现的`cython`模块。尽管项目通常会决定使静态类型信息易于管理的特定方式，但所有这三种方式都可以根据需要进行组合。

虽然通常不建议在`.pyx`文件中编写直接的 Cython 代码，但有正当理由这样做 - 更容易测试和调试，与纯 Python 开发人员协作等。在纯模式下，您或多或少地受限于可以在 Python 中表达（或至少模拟）的代码，以及静态类型声明。除此之外的任何事情都只能在扩展语言语法的.pyx 文件中完成，因为它取决于 Cython 编译器的功能。

## 增加.pxd

使用扩充`.pxd`可以让原始`.py`文件完全不受影响。另一方面，需要保持`.pxd`和`.py`以使它们保持同步。

虽然`.pyx`文件中的声明必须与具有相同名称的`.pxd`文件的声明完全对应（并且任何矛盾导致编译时错误，请参阅 pxd 文件 ） ，`.py`文件中的无类型定义可以通过`.pxd`中存在的更具体的类型覆盖并使用静态类型进行扩充。

如果找到与正在编译的`.py`文件同名的`.pxd`文件，将搜索 `cdef` 类和 `cdef` / `cpdef` 的功能和方法。然后，编译器将`.py`文件中的相应类/函数/方法转换为声明的类型。因此，如果有一个文件`A.py`：

```py
def myfunction(x, y=2):
    a = x - y
    return a + x * y

def _helper(a):
    return a + 1

class A:
    def __init__(self, b=0):
        self.a = 3
        self.b = b

    def foo(self, x):
        print(x + _helper(1.0))

```

并添加`A.pxd`：

```py
cpdef int myfunction(int x, int y=*)
cdef double _helper(double a)

cdef class A:
    cdef public int a, b
    cpdef foo(self, double x)

```

然后 Cython 将编译`A.py`，就像它编写如下：

```py
cpdef int myfunction(int x, int y=2):
    a = x - y
    return a + x * y

cdef double _helper(double a):
    return a + 1

cdef class A:
    cdef public int a, b
    def __init__(self, b=0):
        self.a = 3
        self.b = b

    cpdef foo(self, double x):
        print(x + _helper(1.0))

```

注意为了向`.pxd`中的定义提供 Python 包装器，即可以从 Python 访问，

*   Python 可见函数签名必须声明为 &lt;cite&gt;cpdef&lt;/cite&gt; （默认参数替换为 &lt;cite&gt;*&lt;/cite&gt; 以避免重复）：

    ```py
    cpdef int myfunction(int x, int y=*)

    ```

*   内部函数的 C 函数签名可以声明为 &lt;cite&gt;cdef&lt;/cite&gt; ：

    ```py
    cdef double _helper(double a)

    ```

*   &lt;cite&gt;cdef&lt;/cite&gt; 类（扩展类型）声明为 &lt;cite&gt;cdef 类&lt;/cite&gt;;

*   &lt;cite&gt;cdef&lt;/cite&gt; 类属性必须声明为 &lt;cite&gt;cdef public&lt;/cite&gt; 如果需要读/写 Python 访问， &lt;cite&gt;cdef readonly&lt;/cite&gt; 用于只读 Python 访问，或普通 &lt;cite&gt;cdef&lt;/cite&gt; 用于内部 C 级属性;

*   &lt;cite&gt;cdef&lt;/cite&gt; 类方法必须声明为 &lt;cite&gt;cpdef&lt;/cite&gt; 用于 Python 可见方法或 &lt;cite&gt;cdef&lt;/cite&gt; 用于内部 C 方法。

在上面的例子中， &lt;cite&gt;myfunction（）&lt;/cite&gt;中局部变量&lt;cite&gt;和&lt;/cite&gt;的类型不固定，因此是一个 Python 对象。要静态输入，可以使用 Cython 的`@cython.locals`装饰器（参见 魔法属性 和 魔法属性.pxd） 。

普通 Python（ [`def`](https://docs.python.org/3/reference/compound_stmts.html#def "(in Python v3.7)") ）函数不能在`.pxd`文件中声明。因此，目前不可能在`.pxd`文件中覆盖普通 Python 函数的类型，例如覆盖其局部变量的类型。在大多数情况下，将它们声明为 &lt;cite&gt;cpdef&lt;/cite&gt; 将按预期工作。

## 魔法属性

magic `cython`模块提供了特殊装饰器，可用于在 Python 文件中添加静态类型，同时被解释器忽略。

此选项将`cython`模块依赖项添加到原始代码，但不需要维护补充`.pxd`文件。 Cython 提供了这个模块的虚假版本 &lt;cite&gt;Cython.Shadow&lt;/cite&gt; ，当安装 Cython 时可以作为 &lt;cite&gt;cython.py&lt;/cite&gt; 使用，但是当 Cython 是 Cython 时可以被复制以供其他模块使用。未安装。

### “编译”开关

*   `compiled`是一个特殊变量，在编译器运行时设置为`True`，在解释器中设置为`False`。因此，代码

    ```py
    import cython

    if cython.compiled:
        print("Yep, I'm compiled.")
    else:
        print("Just a lowly interpreted script.")

    ```

    根据代码是作为编译扩展名（`.so` / `.pyd`）模块还是普通`.py`文件执行，将表现不同。

### 静态打字

*   `cython.declare`在当前作用域中声明一个类型变量，可用于代替`cdef type var [= value]`构造。这有两种形式，第一种作为赋值（在解释模式中创建声明时很有用）：

    ```py
    import cython

    x = cython.declare(cython.int)              # cdef int x
    y = cython.declare(cython.double, 0.57721)  # cdef double y = 0.57721

    ```

    和第二种模式作为一个简单的函数调用：

    ```py
    import cython

    cython.declare(x=cython.int, y=cython.double)  # cdef int x; cdef double y

    ```

    它还可以用于定义扩展类型 private，readonly 和 public 属性：

    ```py
    import cython

    @cython.cclass
    class A:
        cython.declare(a=cython.int, b=cython.int)
        c = cython.declare(cython.int, visibility='public')
        d = cython.declare(cython.int)  # private by default.
        e = cython.declare(cython.int, visibility='readonly')

        def __init__(self, a, b, c, d=5, e=3):
            self.a = a
            self.b = b
            self.c = c
            self.d = d
            self.e = e

    ```

*   `@cython.locals`是一个装饰器，用于指定函数体中局部变量的类型（包括参数）：

    ```py
    import cython

    @cython.locals(a=cython.long, b=cython.long, n=cython.longlong)
    def foo(a, b, x, y):
        n = a * b
        # ...

    ```

*   `@cython.returns(&lt;type&gt;)`指定函数的返回类型。

*   `@cython.exceptval(value=None, *, check=False)`指定函数的异常返回值和异常检查语义，如下所示：

    ```py
    @exceptval(-1)               # cdef int func() except -1:
    @exceptval(-1, check=False)  # cdef int func() except -1:
    @exceptval(check=True)       # cdef int func() except *:
    @exceptval(-1, check=True)   # cdef int func() except? -1:

    ```

*   Python 注释可用于声明参数类型，如以下示例所示。为避免与其他类型的注释使用冲突，可以使用指令`annotation_typing=False`禁用此功能。

    ```py
    import cython

    def func(foo: dict, bar: cython.int) -&gt; tuple:
        foo["hello world"] = 3 + bar
        return foo, 5

    ```

    对于非 Python 返回类型，这可以与`@cython.exceptval()`装饰器结合使用：

    ```py
    import cython

    @cython.exceptval(-1)
    def func(x: cython.int) -&gt; cython.int:
        if x &lt; 0:
            raise ValueError("need integer &gt;= 0")
        return x + 1

    ```

    从版本 0.27 开始，Cython 还支持 [PEP 526](https://www.python.org/dev/peps/pep-0526/) 中定义的变量注释。这允许以 Python 3.6 兼容的方式声明变量类型，如下所示：

    ```py
    import cython

    def func():
        # Cython types are evaluated as for cdef declarations
        x: cython.int               # cdef int x
        y: cython.double = 0.57721  # cdef double y = 0.57721
        z: cython.float = 0.57721   # cdef float z  = 0.57721

        # Python types shadow Cython types for compatibility reasons
        a: float = 0.54321          # cdef double a = 0.54321
        b: int = 5                  # cdef object b = 5
        c: long = 6                 # cdef object c = 6
        pass

    @cython.cclass
    class A:
        a: cython.int
        b: cython.int

        def __init__(self, b=0):
            self.a = 3
            self.b = b

    ```

    目前无法表达对象属性的可见性。

### C 类型

Cython 模块内置了许多类型。它提供所有标准 C 类型，即`char`，`short`，`int`，`long`，`longlong`以及它们的无符号版本`uchar`，`ushort`，`uint`，`ulong`， `ulonglong`。特殊的`bint`类型用于 C 布尔值，`Py_ssize_t`用于（容器）的（签名）大小。

对于每种类型，都有指针类型`p_int`，`pp_int`等，在解释模式下最多三级，在编译模式下无限深。可以使用`cython.pointer(cython.int)`构建更多指针类型，将数组构造为`cython.int[10]`。有限的尝试是模拟这些更复杂的类型，但只能通过 Python 语言完成。

Python 类型 int，long 和 bool 分别被解释为 C `int`，`long`和`bint`。此外，可以使用 Python 内置类型`list`，`dict`，`tuple`等，以及任何用户定义的类型。

键入的 C 元组可以声明为 C 类型的元组。

### 扩展类型和 cdef 函数

*   类装饰器`@cython.cclass`创建`cdef class`。
*   函数/方法装饰器`@cython.cfunc`创建 `cdef` 函数。
*   `@cython.ccall`创建 `cpdef` 函数，即 Cython 代码可以在 C 级调用的函数。
*   `@cython.locals`声明局部变量（见上文）。它还可用于声明参数的类型，即签名中使用的局部变量。
*   `@cython.inline`相当于 C `inline`修饰符。
*   `@cython.final`通过阻止将类型用作基类来终止继承链，或者通过在子类型中重写方法来终止继承链。这可以实现某些优化，例如内联方法调用。

以下是 `cdef` 功能的示例：

```py
@cython.cfunc
@cython.returns(cython.bint)
@cython.locals(a=cython.int, b=cython.int)
def c_compare(a,b):
    return a == b

```

### 进一步的 Cython 函数和声明

*   `address`用于代替`&`运算符：

    ```py
    cython.declare(x=cython.int, x_ptr=cython.p_int)
    x_ptr = cython.address(x)

    ```

*   `sizeof`模拟运算符的&lt;cite&gt;大小。它可以采用两种类型和表达方式。&lt;/cite&gt;

    ```py
    cython.declare(n=cython.longlong)
    print(cython.sizeof(cython.longlong))
    print(cython.sizeof(n))

    ```

*   `struct`可用于创建结构类型：

    ```py
    MyStruct = cython.struct(x=cython.int, y=cython.int, data=cython.double)
    a = cython.declare(MyStruct)

    ```

    相当于代码：

    ```py
    cdef struct MyStruct:
        int x
        int y
        double data

    cdef MyStruct a

    ```

*   `union`使用与`struct`完全相同的语法创建联合类型。

*   `typedef`定义给定名称下的类型：

    ```py
    T = cython.typedef(cython.p_int)   # ctypedef int* T

    ```

*   `cast`将（不安全地）重新解释表达式类型。 `cython.cast(T, t)`相当于`&lt;T&gt;t`。第一个属性必须是类型，第二个属性是要转换的表达式。指定可选关键字参数`typecheck=True`具有`&lt;T?&gt;t`的语义。

    ```py
    t1 = cython.cast(T, t)
    t2 = cython.cast(T, t, typecheck=True)

    ```

### .pxd

特殊的 &lt;cite&gt;cython&lt;/cite&gt; 模块也可以在扩充`.pxd`文件中导入和使用。例如，以下 Python 文件`dostuff.py`：

```py
def dostuff(n):
    t = 0
    for i in range(n):
        t += i
    return t

```

可以使用以下`.pxd`文件`dostuff.pxd`进行扩充：

```py
import cython

@cython.locals(t=cython.int, i=cython.int)
cpdef int dostuff(int n)

```

`cython.declare()`函数可用于在扩充`.pxd`文件中指定全局变量的类型。

## 提示与技巧

### 调用 C 函数

通常，不可能在纯 Python 模式下调用 C 函数，因为在普通（未编译）Python 中没有通用的方法来支持它。但是，在存在等效 Python 函数的情况下，可以通过将 C 函数强制与条件导入相结合来实现，如下所示：

```py
# mymodule.pxd

# declare a C function as "cpdef" to export it to the module
cdef extern from "math.h":
    cpdef double sin(double x)

```

```py
# mymodule.py

import cython

# override with Python import if not in compiled code
if not cython.compiled:
    from math import sin

# calls sin() from math.h when compiled with Cython and math.sin() in Python
print(sin(0))

```

请注意，“sin”函数将在此处显示在“mymodule”的模块命名空间中（即，将存在`mymodule.sin()`函数）。您可以根据 Python 惯例将其标记为内部名称，方法是将其重命名为`.pxd`文件中的“_sin”，如下所示：

```py
cdef extern from "math.h":
    cpdef double _sin "sin" (double x)

```

然后，您还可以将 Python 导入更改为`from math import sin as _sin`以使名称再次匹配。

### 将 C 数组用于固定大小的列表

C 数组可以自动强制转换为 Python 列表或元组。这可以被利用来在编译时用 C 数组替换 Python 代码中的固定大小的 Python 列表。一个例子：

```py
import cython

@cython.locals(counts=cython.int[10], digit=cython.int)
def count_digits(digits):
    """
 >>> digits = '01112222333334445667788899'
 >>> count_digits(map(int, digits))
 [1, 3, 4, 5, 3, 1, 2, 2, 3, 2]
 """
    counts = [0] * 10
    for digit in digits:
        assert 0 <= digit <= 9
        counts[digit] += 1
    return counts

```

在普通的 Python 中，这将使用 Python 列表来收集计数，而 Cython 将生成使用 C int 的 C 数组的 C 代码。