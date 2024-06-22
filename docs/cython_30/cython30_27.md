# 在 Cython 模块之间共享声明

> 原文： [`docs.cython.org/en/latest/src/userguide/sharing_declarations.html`](http://docs.cython.org/en/latest/src/userguide/sharing_declarations.html)

本节描述如何在一个 Cython 模块中使 C 声明，函数和扩展类型可用于另一个 Cython 模块。这些工具以 Python 导入机制为模型，可以被认为是它的编译时版本。

## 定义和实施文件

Cython 模块可以分为两部分：带有`.pxd`后缀的定义文件，包含可供其他 Cython 模块使用的 C 声明，以及带有`.pyx`后缀的实现文件，其中包含其他所有内容。当模块想要使用在另一个模块的定义文件中声明的内容时，它会使用 `cimport` 语句导入它。

仅包含 extern 声明的`.pxd`文件不需要与实际的`.pyx`文件或 Python 模块相对应。这可以使它成为一个放置常见声明的便利位置，例如来自 外部库 的函数声明，这些函数需要在多个模块中使用。

## 什么定义文件包含

定义文件可以包含：

*   任何类型的 C 类型声明。
*   extern C 函数或变量声明。
*   模块中定义的 C 函数声明。
*   扩展类型的定义部分（见下文）。

它不能包含任何 C 或 Python 函数的实现，也不能包含任何 Python 类定义或任何可执行语句。当想要访问 `cdef` 属性和方法，或从本模块中定义的 `cdef` 类继承时，需要它。

注意

您不需要（也不应该）在声明文件 public 中声明任何内容，以使其可供其他 Cython 模块使用;它只是存在于定义文件中。如果要为外部 C 代码提供某些内容，则只需要公开声明。

## 实现文件包含什么

实现文件可以包含任何类型的 Cython 语句，但是如果相应的定义文件也定义了该类型，则对扩展类型的实现部分有一些限制（见下文）。如果这个模块不需要 `cimport` ，那么这是唯一需要的文件。

## cimport 声明

`cimport` 语句用于定义或实现文件中，以访问在另一个定义文件中声明的名称。它的语法与普通的 Python import 语句完全相同：

```py
cimport module [, module...]

from module cimport name [as name] [, name [as name] ...]

```

这是一个例子。 `dishes.pxd`是导出 C 数据类型的定义文件。 `restaurant.pyx`是一个导入并使用它的实现文件。

`dishes.pxd`：

```py
cdef enum otherstuff:
    sausage, eggs, lettuce

cdef struct spamdish:
    int oz_of_spam
    otherstuff filler

```

`restaurant.pyx`：

```py
from __future__ import print_function
cimport dishes
from dishes cimport spamdish

cdef void prepare(spamdish *d):
    d.oz_of_spam = 42
    d.filler = dishes.sausage

def serve():
    cdef spamdish d
    prepare(&d)
    print(f'{d.oz_of_spam} oz spam, filler no. {d.filler}')

```

重要的是要理解 `cimport` 语句只能用于导入 C 数据类型，C 函数和变量以及扩展类型。它不能用于导入任何 Python 对象，并且（有一个例外）它并不意味着在运行时导入任何 Python。如果要引用已经 cimported 的模块中的任何 Python 名称，则还必须为其包含常规 import 语句。

例外情况是，当您使用 `cimport` 导入扩展类型时，其类型对象将在运行时导入，并由您导入它的名称提供。使用 `cimport` 导入扩展类型将在下面详细介绍。

如果`.pxd`文件发生变化，则可能需要重新编译 `cimport` 的所有模块。 `Cython.Build.cythonize`实用程序可以为您解决此问题。

### 定义文件的搜索路径

当您 `cimport` 一个名为`modulename`的模块时，Cython 编译器会搜索名为`modulename.pxd`的文件。它沿着包含文件的路径搜索此文件（由`-I`命令行选项或`cythonize()`的`include_path`选项指定）以及`sys.path`。

使用`package_data`在`setup.py`脚本中安装`.pxd`文件允许其他软件包将模块中的项目作为依赖项进行处理。

此外，每当编译文件`modulename.pyx`时，首先沿着包含路径搜索相应的定义文件`modulename.pxd`（但不是`sys.path`），如果找到，则在处理`.pyx`文件之前对其进行处理。

### 使用 cimport 解决命名冲突

`cimport` 机制提供了一种简洁明了的方法来解决使用相同名称的 Python 函数包装外部 C 函数的问题。您需要做的就是将外部 C 声明放入假想模块的`.pxd`文件中，将 `cimport` 放入该模块。然后，您可以通过使用模块名称对它们进行限定来引用 C 函数。这是一个例子：

`c_lunch.pxd`：

```py
cdef extern from "lunch.h":
    void eject_tomato(float)

```

`lunch.pyx`：

```py
cimport c_lunch

def eject_tomato(float speed):
    c_lunch.eject_tomato(speed)

```

您不需要任何`c_lunch.pyx`文件，因为`c_lunch.pxd`中定义的唯一内容是外部 C 实体。在运行时不会有任何实际的`c_lunch`模块，但这无关紧要; `c_lunch.pxd`文件完成了在编译时提供额外命名空间的工作。

## 共享 C 函数

可以通过 `cimport` 在`.pxd`文件中为它们添加标题，使模块顶层定义的 C 函数可用，例如：

`volume.pxd`：

```py
cdef float cube(float)

```

`volume.pyx`：

```py
cdef float cube(float x):
    return x * x * x

```

`spammery.pyx`：

```py
from __future__ import print_function

from volume cimport cube

def menu(description, size):
    print(description, ":", cube(size),
          "cubic metres of spam")

menu("Entree", 1)
menu("Main course", 3)
menu("Dessert", 2)

```

Note

当模块以这种方式导出 C 函数时，对象将出现在函数名称下的模块字典中。但是，您不能使用 Python 中的此对象，也不能使用普通的 import 语句从 Cython 中使用它;你必须使用 `cimport` 。

## 共享扩展类型

可以通过 `cimport` 将扩展类型分为两部分，一部分在定义文件中，另一部分在相应的实现文件中。

扩展类型的定义部分只能声明 C 属性和 C 方法，而不能声明 Python 方法，并且它必须声明所有类型的 C 属性和 C 方法。

实现部分必须实现定义部分中声明的所有 C 方法，并且可能不会添加任何其他 C 属性。它也可以定义 Python 方法。

以下是定义和导出扩展类型的模块示例，以及使用它的另一个模块：

`shrubbing.pxd`：

```py
cdef class Shrubbery:
    cdef int width
    cdef int length

```

`shrubbing.pyx`：

```py
cdef class Shrubbery:
    def __cinit__(self, int w, int l):
        self.width = w
        self.length = l

def standard_shrubbery():
    return Shrubbery(3, 7)

```

`landscaping.pyx`：

```py
cimport shrubbing
import shrubbing

def main():
    cdef shrubbing.Shrubbery sh
    sh = shrubbing.standard_shrubbery()
    print("Shrubbery size is", sh.width, 'x', sh.length)

```

然后，人们需要编译这两个模块，例如运用

`setup.py`：

```py
from distutils.core import setup
from Cython.Build import cythonize

setup(ext_modules=cythonize(["landscaping.pyx", "shrubbing.pyx"]))

```

有关此示例的一些注意事项：

*   `Shrubbing.pxd`和`Shrubbing.pyx`中都有 `cdef` 类灌木声明。编译 Shrubbing 模块时，这两个声明合并为一个。
*   在 Landscaping.pyx 中， `cimport` Shrubbing 声明允许我们将灌木类型称为`Shrubbing.Shrubbery`。但它在运行时没有绑定 Landscaping 模块命名空间中的名称 Shrubbing，因此要访问`Shrubbing.standard_shrubbery()`我们还需要`import Shrubbing`。
*   如果您使用 setuptools 而不是 distutils，则需要注意，运行`python setup.py install`时的默认操作是创建一个压缩的`egg`文件，当您尝试从依赖包中使用它们时，这些文件无法与`pxd`文件一起用于`pxd`文件。为防止这种情况，请在`setup()`的参数中包含`zip_safe=False`。