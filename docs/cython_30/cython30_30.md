# 早期绑定速度

> 原文： [`docs.cython.org/en/latest/src/userguide/early_binding_for_speed.html`](http://docs.cython.org/en/latest/src/userguide/early_binding_for_speed.html)

作为一种动态语言，Python 鼓励一种编程风格，即在方法和属性方面考虑类和对象，而不是它们适合类层次结构。

这可以使 Python 成为一种非常轻松和舒适的快速开发语言，但需要付出代价 - 管理数据类型的“繁文缛节”会被转储到解释器上。在运行时，解释器在搜索命名空间，获取属性以及解析参数和关键字元组方面做了大量工作。与“早期绑定”语言（如 C ++）相比，这种运行时“后期绑定”是 Python 相对缓慢的主要原因。

然而，使用 Cython，通过使用“早期绑定”编程技术可以获得显着的加速。

例如，考虑以下（愚蠢）代码示例：

```py
cdef class Rectangle:
    cdef int x0, y0
    cdef int x1, y1

    def __init__(self, int x0, int y0, int x1, int y1):
        self.x0 = x0
        self.y0 = y0
        self.x1 = x1
        self.y1 = y1

    def area(self):
        area = (self.x1 - self.x0) * (self.y1 - self.y0)
        if area < 0:
            area = -area
        return area

def rectArea(x0, y0, x1, y1):
    rect = Rectangle(x0, y0, x1, y1)
    return rect.area()

```

在`rectArea()`方法中，对`rect.area()`和`area()`方法的调用包含大量的 Python 开销。

但是，在 Cython 中，在 Cython 代码中发生调用的情况下，可以消除大量的开销。例如：

```py
cdef class Rectangle:
    cdef int x0, y0
    cdef int x1, y1

    def __init__(self, int x0, int y0, int x1, int y1):
        self.x0 = x0
        self.y0 = y0
        self.x1 = x1
        self.y1 = y1

    cdef int _area(self):
        area = (self.x1 - self.x0) * (self.y1 - self.y0)
        if area < 0:
            area = -area
        return area

    def area(self):
        return self._area()

def rectArea(x0, y0, x1, y1):
    cdef Rectangle rect = Rectangle(x0, y0, x1, y1)
    return rect._area()

```

这里，在 Rectangle 扩展类中，我们定义了两种不同的区域计算方法，即高效的`_area()` C 方法，以及 Python 可调用的`area()`方法，它作为`_area()`的薄包装器。还要注意函数`rectArea()`我们如何'早期绑定'，通过声明显式赋予 Rectangle 类型的局部变量`rect`。通过使用此声明，我们获得了访问更有效的 C-callable `_area()`方法的能力，而不仅仅是动态分配给`rect`。

但是 Cython 通过允许我们声明双访问方法 - 可以在 C 级别高效调用的方法，但也可以以 Python 访问开销为代价从纯 Python 代码访问，从而再次为我们提供了更多的简单性。考虑以下代码：

```py
cdef class Rectangle:
    cdef int x0, y0
    cdef int x1, y1

    def __init__(self, int x0, int y0, int x1, int y1):
        self.x0 = x0
        self.y0 = y0
        self.x1 = x1
        self.y1 = y1

    cpdef int area(self):
        area = (self.x1 - self.x0) * (self.y1 - self.y0)
        if area < 0:
            area = -area
        return area

def rectArea(x0, y0, x1, y1):
    cdef Rectangle rect = Rectangle(x0, y0, x1, y1)
    return rect.area()

```

在这里，我们只有一个区域方法，声明为 `cpdef` ，使其可以作为 C 函数有效地调用，但仍然可以从纯 Python（或后期绑定的 Cython）代码访问。

如果在 Cython 代码中，我们有一个已经'早期绑定'的变量（即，显式声明为 Rectangle 类型，或者转换为 Rectangle 类型），那么调用其 area 方法将使用高效的 C 代码路径并跳过 Python 开销。但是如果在 Cython 或常规 Python 代码中我们有一个存储 Rectangle 对象的常规对象变量，那么调用 area 方法将需要：

*   区域方法的属性查找
*   打包参数的元组和关键字的 dict（在这种情况下都是空的）
*   使用 Python API 调用该方法

并且在区域方法本身内：

*   解析元组和关键字
*   执行计算代码
*   将结果转换为 python 对象并返回它

因此，在 Cython 中，通过在声明和转换变量中使用强类型来实现大规模优化是可能的。对于使用方法调用的紧密循环，以及这些方法是纯 C 的情况，差异可能很大。