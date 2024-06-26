# 在 Cython 中使用 C ++

> 原文： [`docs.cython.org/en/latest/src/userguide/wrapping_CPlusPlus.html`](http://docs.cython.org/en/latest/src/userguide/wrapping_CPlusPlus.html)

## 概述

Cython 对大多数 C ++语言都有本机支持。特别：

*   可以使用`new`和`del`关键字动态分配 C ++对象。
*   C ++对象可以进行堆栈分配。
*   可以使用 new 关键字`cppclass`声明 C ++类。
*   支持模板化的类和函数。
*   支持重载功能。
*   支持重载 C ++运算符（例如 operator +，operator []，...）。

### 程序概述

包装 C ++文件的一般过程现在可以描述如下：

*   在`setup.py`脚本中指定 C ++语言，或在源文件中指定本地。
*   使用`cdef extern from`块和（如果存在）C ++命名空间名称创建一个或多个`.pxd`文件。在这些块中：
    *   将类声明为`cdef cppclass`块
    *   声明公共名称（变量，方法和构造函数）
*   `cimport`它们位于一个或多个扩展模块（`.pyx`文件）中。

## 一个简单的教程

### 示例 C ++ API

这是一个很小的 C ++ API，我们将在本文档中将其作为示例。我们假设它将位于名为`Rectangle.h`的头文件中：

```py
#ifndef RECTANGLE_H
#define RECTANGLE_H

namespace shapes {
    class Rectangle {
        public:
            int x0, y0, x1, y1;
            Rectangle();
            Rectangle(int x0, int y0, int x1, int y1);
            ~Rectangle();
            int getArea();
            void getSize(int* width, int* height);
            void move(int dx, int dy);
    };
}

#endif

```

以及名为`Rectangle.cpp`的文件中的实现：

```py
#include <iostream>
#include "Rectangle.h"

namespace shapes {

    // Default constructor
    Rectangle::Rectangle () {}

    // Overloaded constructor
    Rectangle::Rectangle (int x0, int y0, int x1, int y1) {
        this->x0 = x0;
        this->y0 = y0;
        this->x1 = x1;
        this->y1 = y1;
    }

    // Destructor
    Rectangle::~Rectangle () {}

    // Return the area of the rectangle
    int Rectangle::getArea () {
        return (this->x1 - this->x0) * (this->y1 - this->y0);
    }

    // Get the size of the rectangle.
    // Put the size in the pointer args
    void Rectangle::getSize (int *width, int *height) {
        (*width) = x1 - x0;
        (*height) = y1 - y0;
    }

    // Move the rectangle by dx dy
    void Rectangle::move (int dx, int dy) {
        this->x0 += dx;
        this->y0 += dy;
        this->x1 += dx;
        this->y1 += dy;
    }
}

```

这非常愚蠢，但应该足以证明所涉及的步骤。

### 声明 C ++类接口

包装 C ++类的过程与包装普通 C 结构的过程非常类似，只需添加几个。让我们从创建基本`cdef extern from`块开始：

```py
cdef extern from "Rectangle.h" namespace "shapes":

```

这将使得 Rectangle 的 C ++类 def 可用。请注意名称空间声明。命名空间仅用于创建对象的完全限定名称，并且可以嵌套（例如`"outer::inner"`）或甚至引用类（例如`"namespace::MyClass`在 MyClass 上声明静态成员）。

#### 使用 cdef cppclass 声明类

现在，让我们从块中将 Rectangle 类添加到此 extern - 只需从 Rectangle.h 复制类名并调整 Cython 语法，现在它变为：

```py
cdef extern from "Rectangle.h" namespace "shapes":
    cdef cppclass Rectangle:

```

#### 添加公共属性

我们现在需要声明在 Cython 上使用的属性和方法。我们将这些声明放在一个名为`Rectangle.pxd`的文件中。您可以将其视为可由 Cython 读取的头文件：

```py
cdef extern from "Rectangle.cpp":
    pass

# Declare the class with cdef
cdef extern from "Rectangle.h" namespace "shapes":
    cdef cppclass Rectangle:
        Rectangle() except +
        Rectangle(int, int, int, int) except +
        int x0, y0, x1, y1
        int getArea()
        void getSize(int* width, int* height)
        void move(int, int)

```

请注意，构造函数声明为“except +”。如果 C ++代码或初始内存分配由于失败而引发异常，这将使 Cython 安全地引发适当的 Python 异常（见下文）。如果没有此声明，Cython 将不会处理源自构造函数的 C ++异常。

我们使用这些线：

```py
cdef extern from "Rectangle.cpp":
    pass

```

包含来自`Rectangle.cpp`的 C ++代码。也可以指定`Rectangle.cpp`为源的 distutils。为此，您可以在`.pyx`（不是`.pxd`）文件的顶部添加此指令：

```py
# distutils: sources = Rectangle.cpp

```

请注意，使用`cdef extern from`时，指定的路径相对于当前文件，但如果使用 distutils 指令，则路径相对于`setup.py`。如果要在运行`setup.py`时发现源的路径，可以使用`cythonize()`功能的`aliases`参数。

#### 用包装的 C ++类声明一个 var

我们将创建一个名为`rect.pyx`的`.pyx`文件来构建我们的包装器。我们使用的是`Rectangle`以外的名称，但如果您希望为包装器提供与 C ++类相同的名称，请参阅 解决命名冲突 的部分。

在其中，我们使用 cdef 使用 C ++ `new`语句声明类的 var：

```py
# distutils: language = c++

from Rectangle cimport Rectangle

def main():
    rec_ptr = new Rectangle(1, 2, 3, 4)  # Instantiate a Rectangle object on the heap
    try:
        rec_area = rec_ptr.getArea()
    finally:
        del rec_ptr  # delete heap allocated object

    cdef Rectangle rec_stack  # Instantiate a Rectangle object on the stack

```

这条线：

```py
# distutils: language = c++

```

是向 Cython 表明这个`.pyx`文件必须编译为 C ++。

只要它具有“默认”构造函数，也可以声明堆栈分配的对象：

```py
cdef extern from "Foo.h":
    cdef cppclass Foo:
        Foo()

def func():
    cdef Foo foo
    ...

```

请注意，与 C ++一样，如果类只有一个构造函数并且它是一个无效的，那么就没有必要声明它。

### 创建 Cython 包装类

此时，我们已经将我们的 pyx 文件的命名空间暴露给 C ++ Rectangle 类型的接口。现在，我们需要从外部 Python 代码访问它（这是我们的全部观点）。

常见的编程实践是创建一个 Cython 扩展类型，它将 C ++实例作为属性保存并创建一堆转发方法。所以我们可以将 Python 扩展类型实现为：

```py
# distutils: language = c++

from Rectangle cimport Rectangle

# Create a Cython extension type which holds a C++ instance
# as an attribute and create a bunch of forwarding methods
# Python extension type.
cdef class PyRectangle:
    cdef Rectangle c_rect  # Hold a C++ instance which we're wrapping

    def __cinit__(self, int x0, int y0, int x1, int y1):
        self.c_rect = Rectangle(x0, y0, x1, y1)

    def get_area(self):
        return self.c_rect.getArea()

    def get_size(self):
        cdef int width, height
        self.c_rect.getSize(&width, &height)
        return width, height

    def move(self, dx, dy):
        self.c_rect.move(dx, dy)

```

我们终于得到它了。从 Python 的角度来看，这种扩展类型的外观和感觉就像本机定义的 Rectangle 类一样。需要注意的是，如果要提供属性访问权限，可以实现一些属性：

```py
# distutils: language = c++

from Rectangle cimport Rectangle

cdef class PyRectangle:
    cdef Rectangle c_rect

    def __cinit__(self, int x0, int y0, int x1, int y1):
        self.c_rect = Rectangle(x0, y0, x1, y1)

    def get_area(self):
        return self.c_rect.getArea()

    def get_size(self):
        cdef int width, height
        self.c_rect.getSize(&width, &height)
        return width, height

    def move(self, dx, dy):
        self.c_rect.move(dx, dy)

    # Attribute access
    @property
    def x0(self):
        return self.c_rect.x0
    @x0.setter
    def x0(self, x0):
        self.c_rect.x0 = x0

    # Attribute access
    @property
    def x1(self):
        return self.c_rect.x1
    @x1.setter
    def x1(self, x1):
        self.c_rect.x1 = x1

    # Attribute access
    @property
    def y0(self):
        return self.c_rect.y0
    @y0.setter
    def y0(self, y0):
        self.c_rect.y0 = y0

    # Attribute access
    @property
    def y1(self):
        return self.c_rect.y1
    @y1.setter
    def y1(self, y1):
        self.c_rect.y1 = y1

```

Cython 使用 nullary 构造函数初始化 cdef 类的 C ++类属性。如果要包装的类没有构造函数，则必须存储指向包装类的指针并手动分配和取消分配它。一个方便和安全的地方是 &lt;cite&gt;__cinit__&lt;/cite&gt; 和 &lt;cite&gt;__dealloc__&lt;/cite&gt; 方法，保证在创建和删除 Python 实例时只调用一次。

```py
# distutils: language = c++

from Rectangle cimport Rectangle

cdef class PyRectangle:
    cdef Rectangle*c_rect  # hold a pointer to the C++ instance which we're wrapping

    def __cinit__(self, int x0, int y0, int x1, int y1):
        self.c_rect = new Rectangle(x0, y0, x1, y1)

    def __dealloc__(self):
        del self.c_rect

```

## 编译和导入

要编译 Cython 模块，必须有一个`setup.py`文件：

```py
from distutils.core import setup

from Cython.Build import cythonize

setup(ext_modules=cythonize("rect.pyx"))

```

运行`$ python setup.py build_ext --inplace`

要测试它，打开 Python 解释器：

```py
>>> import rect
>>> x0, y0, x1, y1 = 1, 2, 3, 4
>>> rect_obj = rect.PyRectangle(x0, y0, x1, y1)
>>> print(dir(rect_obj))
['__class__', '__delattr__', '__dir__', '__doc__', '__eq__', '__format__', '__ge__',
 '__getattribute__', '__gt__', '__hash__', '__init__', '__init_subclass__', '__le__',
 '__lt__', '__ne__', '__new__', '__reduce__', '__reduce_ex__', '__repr__', '__setattr__',
 '__setstate__', '__sizeof__', '__str__', '__subclasshook__', 'get_area', 'get_size', 'move']

```

## 高级 C ++特性

我们在这里描述了上面教程中没有讨论过的所有 C ++特性。

### 重载

重载非常简单。只需使用不同的参数声明方法并使用其中任何一个：

```py
cdef extern from "Foo.h":
    cdef cppclass Foo:
        Foo(int)
        Foo(bool)
        Foo(int, bool)
        Foo(int, int)

```

### 重载运算符

Cython 使用 C ++命名来重载运算符：

```py
cdef extern from "foo.h":
    cdef cppclass Foo:
        Foo()
        Foo operator+(Foo)
        Foo operator-(Foo)
        int operator*(Foo)
        int operator/(int)
        int operator*(int, Foo) # allows 1*Foo()
    # nonmember operators can also be specified outside the class
    double operator/(double, Foo)

cdef Foo foo = new Foo()

foo2 = foo + foo
foo2 = foo - foo

x = foo * foo2
x = foo / 1

x = foo[0] * foo2
x = foo[0] / 1
x = 1*foo[0]

cdef double y
y = 2.0/foo[0]

```

请注意，如果有一个 *指针* 到 C ++对象，则必须进行解除引用以避免执行指针算术而不是对对象本身进行算术运算：

```py
cdef Foo* foo_ptr = new Foo()
foo = foo_ptr[0] + foo_ptr[0]
x = foo_ptr[0] / 2

del foo_ptr

```

### 嵌套类声明

C ++允许嵌套类声明。类声明也可以嵌套在 Cython 中：

```py
# distutils: language = c++

cdef extern from "<vector>" namespace "std":
    cdef cppclass vector[T]:
        cppclass iterator:
            T operator*()
            iterator operator++()
            bint operator==(iterator)
            bint operator!=(iterator)
        vector()
        void push_back(T&)
        T& operator[](int)
        T& at(int)
        iterator begin()
        iterator end()

cdef vector[int].iterator iter  #iter is declared as being of type vector<int>::iterator

```

请注意，嵌套类使用`cppclass`声明但没有`cdef`，因为它已经是`cdef`声明部分的一部分。

### C ++运算符与 Python 语法不兼容

Cython 尝试使其语法尽可能接近标准 Python。因此，某些 C ++运算符（如 preincrement `++foo`或解除引用运算符`*foo`）不能与 C ++使用相同的语法。 Cython 提供了在特殊模块`cython.operator`中替换这些运算符的函数。提供的功能是：

*   用于解除引用的`cython.operator.dereference`。 `dereference(foo)`将生成 C ++代码`*(foo)`
*   `cython.operator.preincrement`用于预增量。 `preincrement(foo)`将生成 C ++代码`++(foo)`。类似地`predecrement`，`postincrement`和`postdecrement`。
*   逗号运算符的`cython.operator.comma`。 `comma(a, b)`将生成 C ++代码`((a), (b))`。

这些功能需要被引导。当然，可以使用`from ... cimport ... as`来获得更短且更易读的功能。例如：`from cython.operator cimport dereference as deref`。

为了完整起见，还值得一提的是`cython.operator.address`，它也可以写成`&foo`。

### 模板

Cython 使用括号语法进行模板化。包装 C ++向量的一个简单示例：

```py
# distutils: language = c++

# import dereference and increment operators
from cython.operator cimport dereference as deref, preincrement as inc

cdef extern from "<vector>" namespace "std":
    cdef cppclass vector[T]:
        cppclass iterator:
            T operator*()
            iterator operator++()
            bint operator==(iterator)
            bint operator!=(iterator)
        vector()
        void push_back(T&)
        T& operator[](int)
        T& at(int)
        iterator begin()
        iterator end()

cdef vector[int] *v = new vector[int]()
cdef int i
for i in range(10):
    v.push_back(i)

cdef vector[int].iterator it = v.begin()
while it != v.end():
    print(deref(it))
    inc(it)

del v

```

可以将多个模板参数定义为列表，例如`[T, U, V]`或`[int, bool, char]`。可以通过写`[T, U, V=*]`来指示可选的模板参数。如果 Cython 需要显式引用不完整模板实例化的默认模板参数的类型，它将写入`MyClass&lt;T, U&gt;::V`，因此如果类为其模板参数提供 typedef，则最好在此处使用该名称。

模板函数的定义与类模板类似，模板参数列表位于函数名称后面：

```py
# distutils: language = c++

cdef extern from "<algorithm>" namespace "std":
    T maxT

print(maxlong)
print(max(1.5, 2.5))  # simple template argument deduction

```

### 标准库

C ++标准库的大多数容器已在位于 [/ Cython / Includes / libcpp](https://github.com/cython/cython/tree/master/Cython/Includes/libcpp) 中的 pxd 文件中声明。这些容器是：deque，list，map，pair，queue，set，stack，vector。

例如：

```py
# distutils: language = c++

from libcpp.vector cimport vector

cdef vector[int] vect
cdef int i, x

for i in range(10):
    vect.push_back(i)

for i in range(10):
    print(vect[i])

for x in vect:
    print(x)

```

[/ Cython / Includes / libcpp](https://github.com/cython/cython/tree/master/Cython/Includes/libcpp) 中的 pxd 文件也是如何声明 C ++类的好例子。

STL 容器强制执行相应的 Python 内置类型。转换是通过赋值给类型变量（包括类型化函数参数）或通过显式强制转换来触发的，例如：

```py
# distutils: language = c++

from libcpp.string cimport string
from libcpp.vector cimport vector

py_bytes_object = b'The knights who say ni'
py_unicode_object = u'Those who hear them seldom live to tell the tale.'

cdef string s = py_bytes_object
print(s)  # b'The knights who say ni'

cdef string cpp_string = <string> py_unicode_object.encode('utf-8')
print(cpp_string)  # b'Those who hear them seldom live to tell the tale.'

cdef vector[int] vect = range(1, 10, 2)
print(vect)  # [1, 3, 5, 7, 9]

cdef vector[string] cpp_strings = b'It is a good shrubbery'.split()
print(cpp_strings[1])   # b'is'

```

可以使用以下强制措施：

<colgroup><col width="35%"> <col width="31%"> <col width="33%"></colgroup> 
| Python type =＆gt; | *C ++类型 * | =＆GT; Python 类型 |
| --- | --- | --- |
| 字节 | 的 std :: string | bytes |
| 迭代 | 的 std ::矢量 | 名单 |
| iterable | 的 std ::名单 | list |
| iterable | 的 std ::设为 | 组 |
| 可迭代的（len 2） | 的 std ::对 | 元组（len 2） |

所有转换都会创建一个新容器并将数据复制到其中。容器中的物品自动转换成相应的类型，包括递归地转换容器内的容器，例如容器。字符串映射的 C ++向量。

通过`for .. in`语法（包括列表推导）支持对 stl 容器（或实际上任何具有`begin()`和`end()`方法的类返回支持递增，解除引用和比较的对象的迭代）的迭代。例如，人们可以写：

```py
# distutils: language = c++

from libcpp.vector cimport vector

def main():
    cdef vector[int] v = [4, 6, 5, 10, 3]

    cdef int value
    for value in v:
        print(value)

    return [x*x for x in v if x % 2 == 0]

```

如果未指定循环目标变量，则 类型推断 使用类型`*container.begin()`的赋值。

注意

支持切片 stl 容器，你可以做`for x in my_vector[:5]: ...`，但与指针切片不同，它将创建一个临时的 Python 对象并迭代它。因此使迭代非常缓慢。出于性能原因，您可能希望避免切片 C ++容器。

### 使用默认构造函数简化包装

如果您的扩展类型使用默认构造函数（不传递任何参数）实例化包装的 C ++类，您可以通过将生命周期处理直接绑定到 Python 包装器对象的生命周期来简化生命周期处理。您可以声明一个实例，而不是指针属性：

```py
# distutils: language = c++

from libcpp.vector cimport vector

cdef class VectorStack:
    cdef vector[int] v

    def push(self, x):
        self.v.push_back(x)

    def pop(self):
        if self.v.empty():
            raise IndexError()
        x = self.v.back()
        self.v.pop_back()
        return x

```

Cython 将自动生成在创建 Python 对象时实例化 C ++对象实例的代码，并在 Python 对象被垃圾回收时删除它。

### 例外情况

Cython 不能抛出 C ++异常，或者用 try-except 语句捕获它们，但是可以将函数声明为可能引发 C ++异常并将其转换为 Python 异常。例如，

```py
cdef extern from "some_file.h":
    cdef int foo() except +

```

这会将 try 和 C ++错误转换为适当的 Python 异常。根据下表执行转换（从 C ++标识符中省略`std::`前缀）：

<colgroup><col width="52%"> <col width="48%"></colgroup> 
| C ++ | 蟒蛇 |
| --- | --- |
| `bad_alloc` | `MemoryError` |
| `bad_cast` | `TypeError` |
| `bad_typeid` | `TypeError` |
| `domain_error` | `ValueError` |
| `invalid_argument` | `ValueError` |
| `ios_base::failure` | `IOError` |
| `out_of_range` | `IndexError` |
| `overflow_error` | `OverflowError` |
| `range_error` | `ArithmeticError` |
| `underflow_error` | `ArithmeticError` |
| （所有其他人） | `RuntimeError` |

保留`what()`消息（如果有）。请注意，C ++ `ios_base_failure`可以表示 EOF，但是没有足够的信息供 Cython 识别，因此请注意 IO 流上的异常掩码。

```py
cdef int bar() except +MemoryError

```

这将捕获任何 C ++错误并在其位置引发 Python MemoryError。 （任何 Python 异常在这里都有效。）

```py
cdef int raise_py_error()
cdef int something_dangerous() except +raise_py_error

```

如果 something_dangerous 引发了 C ++异常，那么将调用 raise_py_error，这允许自定义 C ++到 Python 错误“翻译”。如果 raise_py_error 实际上没有引发异常，则会引发 RuntimeError。

还有一种特殊的形式：

```py
cdef int raise_py_or_cpp() except +*

```

对于那些可能引发 Python 或 C ++异常的函数。

### 静态成员方法

如果 Rectangle 类具有静态成员：

```py
namespace shapes {
    class Rectangle {
    ...
    public:
        static void do_something();

    };
}

```

你可以使用 Python @staticmethod 装饰器声明它，即：

```py
cdef extern from "Rectangle.h" namespace "shapes":
    cdef cppclass Rectangle:
        ...
        @staticmethod
        void do_something()

```

### 声明/使用参考文献

Cython 支持使用标准`Type&`语法声明左值引用。但请注意，没有必要将 extern 函数的参数声明为引用（const 或其他），因为它对调用者的语法没有影响。

### `auto`关键字

虽然 Cython 没有`auto`关键字，但是没有用`cdef`显式输入的 Cython 局部变量是从 *右侧的类型中推断出所有* 的分配（参见`infer_types` 编译指令 ）。在处理返回复杂，嵌套，模板化类型的函数时，这尤其方便，例如：

```py
cdef vector[int] v = ...
it = v.begin()

```

（当然，对于支持迭代协议的对象，`for .. in`语法是首选。）

## RTTI 和 typeid（）

Cython 支持`typeid(...)`运算符。

> 来自 cython.operator cimport typeid

`typeid(...)`运算符返回`const type_info &`类型的对象。

如果要将 type_info 值存储在 C 变量中，则需要将其存储为指针而不是引用：

```py
from libcpp.typeinfo cimport type_info
cdef const type_info* info = &typeid(MyClass)

```

如果将无效类型传递给`typeid`，它将抛出`std::bad_typeid`异常，该异常在 Python 中转换为`TypeError`异常。

`libcpp.typeindex`中提供了另一个与 C ++ 11 兼容的 RTTI 相关类`std::type_index`。

## 在 setup.py 中指定 C ++语言

可以在`setup.py`文件中声明它们，而不是在源文件中指定语言和源：

```py
from distutils.core import setup
from Cython.Build import cythonize

setup(ext_modules = cythonize(
           "rect.pyx",                 # our Cython source
           sources=["Rectangle.cpp"],  # additional source file(s)
           language="c++",             # generate C++ code
      ))

```

Cython 将生成并编译`rect.cpp`文件（来自`rect.pyx`），然后它将编译`Rectangle.cpp`（`Rectangle`类的实现）并将两个目标文件一起链接到 Linux 上的`rect.so`或`rect.pyd`在 Windows 上，然后可以使用`import rect`在 Python 中导入（如果忘记链接`Rectangle.o`，则在 Python 中导入库时将丢失符号）。

请注意，`language`选项对传递给`cythonize()`的用户提供的 Extension 对象没有影响。它仅用于按文件名找到的模块（如上例所示）。

Cython 版本中最大 0.21 的`cythonize()`功能无法识别`language`选项，需要将其指定为描述扩展名的`Extension`选项，然后由`cythonize()`处理，如下所示：

```py
from distutils.core import setup, Extension
from Cython.Build import cythonize

setup(ext_modules = cythonize(Extension(
           "rect",                                # the extension name
           sources=["rect.pyx", "Rectangle.cpp"], # the Cython source and
                                                  # additional C++ source files
           language="c++",                        # generate and compile C++ code
      )))

```

选项也可以直接从源文件传递，这通常是可取的（并覆盖任何全局选项）。从版本 0.17 开始，Cython 还允许以这种方式将外部源文件传递到`cythonize()`命令。这是一个简化的 setup.py 文件：

```py
from distutils.core import setup
from Cython.Build import cythonize

setup(
    name = "rectangleapp",
    ext_modules = cythonize('*.pyx'),
)

```

在.pyx 源文件中，将其写入第一个注释块，在任何源代码之前，以 C ++模式编译它并将其静态链接到`Rectangle.cpp`代码文件：

```py
# distutils: language = c++
# distutils: sources = Rectangle.cpp

```

Note

使用 distutils 指令时，路径相对于 distutils 运行的工作目录（通常是`setup.py`所在的项目根目录）。

要手动编译（例如使用`make`），可以使用`cython`命令行实用程序生成 C ++ `.cpp`文件，然后将其编译为 python 扩展。使用`--cplus`选项打开`cython`命令的 C ++模式。

## 警告和限制

### 访问仅限 C 的函数

每当生成 C ++代码时，Cython 都会生成函数的声明和调用，假设这些函数是 C ++（即，未声明为`extern "C" {...}`。如果 C 函数具有 C ++入口点，这是可以的，但如果它们只是 C，那么你将遇到障碍。如果你有一个 C ++ Cython 模块需要调用 pure-C 函数，你需要编写一个小的 C ++ shim 模块：

*   包括 extern“C”块中所需的 C 头
*   包含 C ++中的最小转发函数，每个函数都调用相应的 pure-C 函数

### C ++左值

C ++允许返回引用的函数为 left-values。 Cython 目前不支持此功能。 `cython.operator.dereference(foo)`也不被视为左值。