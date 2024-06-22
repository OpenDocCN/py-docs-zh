# pxd 文件

> 原文： [`docs.cython.org/en/latest/src/tutorial/pxd_files.html`](http://docs.cython.org/en/latest/src/tutorial/pxd_files.html)

除了`.pyx`源文件之外，Cython 还使用`.pxd`文件，它们的工作方式类似于 C 头文件 - 它们包含 Cython 声明（有时是代码部分），仅供 Cython 模块使用。使用`cimport`关键字将`pxd`文件导入`pyx`模块。

`pxd`文件有很多用例：

> 1.  它们可用于共享外部 C 声明。
>     
>     
> 2.  它们可以包含非常适合 C 编译器内联的函数。这些功能应标记为`inline`，例如：
>     
>     
>     
>     ```py
>     cdef inline int int_min(int a, int b):
>         return b if b &lt; a else a
>     
>     ```
>     
>     
> 3.  当附带同名的`pyx`文件时，它们为 Cython 模块提供了一个 Cython 接口，以便其他 Cython 模块可以使用比 Python 更高效的协议与之通信。

在我们的集成示例中，我们可能会将其分解为`pxd`文件，如下所示：

> 1.  添加`cmath.pxd`功能，定义 C `math.h`头文件中可用的 C 功能，如`sin`。然后人们只需在`integrate.pyx`中做`from cmath cimport sin`。
>     
>     
> 2.  添加`integrate.pxd`，以便用 Cython 编写的其他模块可以定义要集成的快速自定义函数。
>     
>     
>     
>     ```py
>     cdef class Function:
>         cpdef evaluate(self, double x)
>     cpdef integrate(Function f, double a,
>                     double b, int N)
>     
>     ```
>     
>     
>     
>     请注意，如果您的 cdef 类具有属性，则必须在类声明`pxd`文件（如果使用）中声明属性，而不是`pyx`文件。编译器会告诉你这个。