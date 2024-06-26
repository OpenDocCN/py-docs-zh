# 扩展类型（又名.cdef 类）

> 原文： [`docs.cython.org/en/latest/src/tutorial/cdef_classes.html`](http://docs.cython.org/en/latest/src/tutorial/cdef_classes.html)

为了支持面向对象的编程，Cython 支持编写与 Python 完全相同的普通 Python 类：

```py
class MathFunction(object):
    def __init__(self, name, operator):
        self.name = name
        self.operator = operator

    def __call__(self, *operands):
        return self.operator(*operands)

```

然而，基于 Python 所谓的“内置类型”，Cython 支持第二种类：*扩展类型*，由于用于声明的关键字，有时也称为“cdef 类”。与 Python 类相比，它们受到一定限制，但通常比通用 Python 类更高效，速度更快。主要区别在于它们使用 C 结构来存储它们的字段和方法而不是 Python 字典。这允许他们在他们的字段中存储任意 C 类型，而不需要 Python 包装器，并且可以直接在 C 级访问字段和方法，而无需通过 Python 字典查找。

普通的 Python 类可以从 cdef 类继承，但不能从其他方面继承。 Cython 需要知道完整的继承层次结构，以便布局它们的 C 结构，并将其限制为单继承。另一方面，普通的 Python 类可以从 Cython 代码和纯 Python 代码中继承任意数量的 Python 类和扩展类型。

到目前为止，我们的集成示例并不是非常有用，因为它只集成了一个硬编码函数。为了解决这个问题，我们将使用 cdef 类来表示浮点数上的函数：

```py
cdef class Function:
    cpdef double evaluate(self, double x) except *:
        return 0

```

指令 cpdef 提供了两种版本的方法;一个快速使用从 Cython 和一个较慢的使用从 Python。然后：

```py
from libc.math cimport sin

cdef class Function:
    cpdef double evaluate(self, double x) except *:
        return 0

cdef class SinOfSquareFunction(Function):
    cpdef double evaluate(self, double x) except *:
        return sin(x ** 2)

```

这比为 cdef 方法提供 python 包装稍微多一点：与 cdef 方法不同，cpdef 方法可以被 Python 子类中的方法和实例属性完全覆盖。与 cdef 方法相比，它增加了一点调用开销。

为了使类定义对其他模块可见，从而允许在实现它们的模块之外进行有效的 C 级使用和继承，我们在`sin_of_square.pxd`文件中定义它们：

```py
cdef class Function:
    cpdef double evaluate(self, double x) except *

cdef class SinOfSquareFunction(Function):
    cpdef double evaluate(self, double x) except *

```

使用它，我们现在可以更改我们的集成示例：

```py
from sin_of_square cimport Function, SinOfSquareFunction

def integrate(Function f, double a, double b, int N):
    cdef int i
    cdef double s, dx
    if f is None:
        raise ValueError("f cannot be None")
    s = 0
    dx = (b - a) / N
    for i in range(N):
        s += f.evaluate(a + i * dx)
    return s * dx

print(integrate(SinOfSquareFunction(), 0, 1, 10000))

```

这几乎与前面的代码一样快，但是由于可以更改集成功能，因此它更加灵活。我们甚至可以传入 Python 空间中定义的新函数：

```py
>>> import integrate
>>> class MyPolynomial(integrate.Function):
...     def evaluate(self, x):
...         return 2*x*x + 3*x - 10
...
>>> integrate(MyPolynomial(), 0, 1, 10000)
-7.8335833300000077

```

这比原始的仅使用 Python 的集成代码慢大约 20 倍，但仍然快 10 倍。这显示了当整个循环从 Python 代码移动到 Cython 模块时，加速可以很容易地变大。

关于`evaluate`新实施的一些注意事项：

> *   此处的快速方法调度仅起作用，因为`Function`中声明了`evaluate`。如果在`SinOfSquareFunction`中引入`evaluate`，代码仍然可以工作，但 Cython 会使用较慢的 Python 方法调度机制。
> *   以同样的方式，如果参数`f`没有被输入，但只是作为 Python 对象传递，那么将使用较慢的 Python 调度。
> *   由于参数是打字的，我们需要检查它是否是`None`。在 Python 中，当查找`evaluate`方法时，这会导致`AttributeError`，但 Cython 会尝试访问`None`的（不兼容的）内部结构，就像它是`Function`一样，导致崩溃或数据损坏。

有一个 *编译器指令* `nonecheck`，它会以降低速度为代价启用此检查。以下是编译器指令用于动态打开或关闭`nonecheck`的方法：

```py
# cython: nonecheck=True
#        ^^^ Turns on nonecheck globally

import cython

cdef class MyClass:
    pass

# Turn off nonecheck locally for the function
@cython.nonecheck(False)
def func():
    cdef MyClass obj = None
    try:
        # Turn nonecheck on again for a block
        with cython.nonecheck(True):
            print(obj.myfunc())  # Raises exception
    except AttributeError:
        pass
    print(obj.myfunc())  # Hope for a crash!

```

cdef 类中的属性与常规类中的属性的行为不同：

> *   所有属性必须在编译时预先声明
> *   属性默认只能从用 Cython 访问（类型化访问）
> *   属性可以声明为暴露的 Python 空间的动态属性

```py
from sin_of_square cimport Function

cdef class WaveFunction(Function):

    # Not available in Python-space:
    cdef double offset

    # Available in Python-space:
    cdef public double freq

    # Available in Python-space, but only for reading:
    cdef readonly double scale

    # Available in Python-space:
    @property
    def period(self):
        return 1.0 / self.freq

    @period.setter
    def period(self, value):
        self.freq = 1.0 / value

```