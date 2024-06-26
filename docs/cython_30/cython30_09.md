# 调用 C 库函数

> 原文： [`docs.cython.org/en/latest/src/tutorial/external.html`](http://docs.cython.org/en/latest/src/tutorial/external.html)

本教程简要介绍了从 Cython 代码调用 C 库函数时需要了解的内容。有关使用外部 C 库，包装它们和处理错误的更长更全面的教程，请参阅 使用 C 库 。

为简单起见，让我们从标准 C 库中的函数开始。这不会为您的代码添加任何依赖项，并且它还具有 Cython 已经为您定义了许多此类函数的额外优势。所以你可以直接使用它们。

例如，假设您需要一种低级方法来解析`char*`值中的数字。您可以使用`stdlib.h`头文件定义的`atoi()`功能。这可以按如下方式完成：

```py
from libc.stdlib cimport atoi

cdef parse_charptr_to_py_int(char* s):
    assert s is not NULL, "byte string value is NULL"
    return atoi(s)  # note: atoi() has no error detection!

```

您可以在 Cython 的源代码包 [Cython / Includes /](https://github.com/cython/cython/tree/master/Cython/Includes) 中找到这些标准 cimport 文件的完整列表。它们存储在`.pxd`文件中，这是提供可以在模块之间共享的可重用 Cython 声明的标准方法（参见 在 Cython 模块之间共享声明 ）。

Cython 还为 CPython 的 C-API 提供了一整套声明。例如，要在 C 编译时测试您的代码正在编译的 CPython 版本，您可以这样做：

```py
from cpython.version cimport PY_VERSION_HEX

# Python version >= 3.2 final ?
print(PY_VERSION_HEX >= 0x030200F0)

```

Cython 还提供 C 数学库的声明：

```py
from libc.math cimport sin

cdef double f(double x):
    return sin(x * x)

```

## 动态链接

libc 数学库的特殊之处在于它在某些类 Unix 系统（如 Linux）上没有默认链接。除了导入声明之外，还必须将构建系统配置为链接到共享库`m`。对于 distutils，将它添加到`Extension()`设置的`libraries`参数就足够了：

```py
from distutils.core import setup
from distutils.extension import Extension
from Cython.Build import cythonize

ext_modules = [
    Extension("demo",
              sources=["demo.pyx"],
              libraries=["m"]  # Unix-like specific
              )
]

setup(name="Demos",
      ext_modules=cythonize(ext_modules))

```

## 外部声明

如果要访问 Cython 未提供即用型声明的 C 代码，则必须自行声明。例如，上面的`sin()`函数定义如下：

```py
cdef extern from "math.h":
    double sin(double x)

```

这声明了`sin()`函数，使其可用于 Cython 代码并指示 Cython 生成包含`math.h`头文件的 C 代码。 C 编译器将在编译时在`math.h`中看到原始声明，但 Cython 不解析“math.h”并需要单独的定义。

就像数学库中的`sin()`函数一样，只要 Cython 生成的模块与共享库或静态库正确链接，就可以声明并调用任何 C 库。

请注意，通过将其声明为`cpdef`，可以轻松地从 Cython 模块导出外部 C 函数。这会为它生成一个 Python 包装器并将其添加到模块 dict 中。这是一个 Cython 模块，可以直接访问 Python 代码的 C `sin()`函数：

```py
"""
>>> sin(0)
0.0
"""

cdef extern from "math.h":
    cpdef double sin(double x)

```

当此声明出现在属于 Cython 模块的`.pxd`文件中时（即具有相同名称的 在 Cython 模块之间共享声明 ），会得到相同的结果。这允许 C 声明在其他 Cython 模块中重用，同时仍然在此特定模块中提供自动生成的 Python 包装器。

## 命名参数

C 和 Cython 都支持没有参数名称的签名声明，如下所示：

```py
cdef extern from "string.h":
    char* strstr(const char*, const char*)

```

但是，这会阻止 Cython 代码使用关键字参数调用它。因此，最好像这样编写声明：

```py
cdef extern from "string.h":
    char* strstr(const char *haystack, const char *needle)

```

您现在可以清楚地说明两个参数中的哪一个在您的调用中执行了哪些操作，从而避免了任何歧义并且通常使您的代码更具可读性：

```py
cdef extern from "string.h":
    char* strstr(const char *haystack, const char *needle)

cdef char* data = "hfvcakdfagbcffvschvxcdfgccbcfhvgcsnfxjh"

cdef char* pos = strstr(needle='akd', haystack=data)
print(pos is not NULL)

```

请注意，稍后更改现有参数名称是向后不兼容的 API 修改，就像 Python 代码一样。因此，如果您为外部 C 或 C ++函数提供自己的声明，那么通常需要额外的工作来选择其参数的名称。