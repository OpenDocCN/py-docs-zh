# 扩展类型

> 原文： [`docs.cython.org/en/latest/src/userguide/extension_types.html`](http://docs.cython.org/en/latest/src/userguide/extension_types.html)

## 介绍

除了使用 Python 类语句创建普通的用户定义类之外，Cython 还允许您创建新的内置 Python 类型，称为扩展类型。使用 `cdef` 类语句定义扩展类型。这是一个例子：

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

如您所见，Cython 扩展类型定义看起来很像 Python 类定义。在其中，您使用 def 语句来定义可以从 Python 代码调用的方法。您甚至可以像在 Python 中一样定义许多特殊方法，如`__init__()`。

主要区别在于您可以使用 `cdef` 语句来定义属性。属性可以是 Python 对象（通用或特定扩展类型），或者它们可以是任何 C 数据类型。因此，您可以使用扩展类型来包装任意 C 数据结构，并为它们提供类似 Python 的接口。

## 静态属性

扩展类型的属性直接存储在对象的 C 结构中。这组属性在编译时是固定的;您无法在运行时向扩展类型实例添加属性，只需分配给它们，就像使用 Python 类实例一样。但是，您可以显式启用对动态分配的属性的支持，或者使用普通的 Python 类将扩展类型子类化，然后支持任意属性分配。参见 动态属性 。

有两种方法可以访问扩展类型的属性：通过 Python 属性查找，或通过从 Cython 代码直接访问 C 结构。 Python 代码只能通过第一种方法访问扩展类型的属性，但 Cython 代码可以使用任一方法。

默认情况下，扩展类型属性只能通过直接访问访问，而不能通过 Python 访问访问，这意味着无法从 Python 代码访问它们。要使它们可以从 Python 代码访问，您需要将它们声明为 `public` 或 `readonly` 。例如：

```py
cdef class Shrubbery:
    cdef public int width, height
    cdef readonly float depth

```

使得宽度和高度属性可以从 Python 代码中读取和写入，深度属性可读但不可写。

注意

您只能为 Python 访问公开简单的 C 类型，例如整数，浮点数和字符串。您还可以公开 Python 值属性。

Note

此外， `public` 和 `readonly` 选项仅适用于 Python 访问，而不适用于直接访问。扩展类型的所有属性始终可通过 C 级访问进行读写。

## 动态属性

默认情况下，无法在运行时向扩展类型添加属性。您有两种方法可以避免这种限制，当从 Python 代码调用方法时，这两种方法都会增加开销。特别是在调用`cpdef`方法时。

第一种方法是创建一个 Python 子类：

```py
cdef class Animal:

    cdef int number_of_legs

    def __cinit__(self, int number_of_legs):
        self.number_of_legs = number_of_legs

class ExtendableAnimal(Animal):  # Note that we use class, not cdef class
    pass

dog = ExtendableAnimal(4)
dog.has_tail = True

```

声明`__dict__`属性是启用动态属性的第二种方式：

```py
cdef class Animal:

    cdef int number_of_legs
    cdef dict __dict__

    def __cinit__(self, int number_of_legs):
        self.number_of_legs = number_of_legs

dog = Animal(4)
dog.has_tail = True

```

## 类型声明

在您可以直接访问扩展类型的属性之前，Cython 编译器必须知道您拥有该类型的实例，而不仅仅是通用 Python 对象。它已经知道这种类型的方法的`self`参数，但在其他情况下，您将不得不使用类型声明。

例如，在以下功能中：

```py
cdef widen_shrubbery(sh, extra_width): # BAD
    sh.width = sh.width + extra_width

```

因为`sh`参数没有给出类型，所以将通过 Python 属性查找访问 width 属性。如果属性已被声明为 `public` 或 `readonly` ，那么这将起作用，但效率非常低。如果属性是私有的，它根本不起作用 - 代码将编译，但是在运行时会引发属性错误。

解决方案是将`sh`声明为`Shrubbery`类型，如下所示：

```py
from my_module cimport Shrubbery

cdef widen_shrubbery(Shrubbery sh, extra_width):
    sh.width = sh.width + extra_width

```

现在，Cython 编译器知道`sh`有一个名为`width`的 C 属性，并将生成代码以直接有效地访问它。同样的考虑适用于局部变量，例如：

```py
from my_module cimport Shrubbery

cdef Shrubbery another_shrubbery(Shrubbery sh1):
    cdef Shrubbery sh2
    sh2 = Shrubbery()
    sh2.width = sh1.width
    sh2.height = sh1.height
    return sh2

```

Note

我们这里`cimport`类`Shrubbery`，这是在编译时声明类型所必需的。为了能够`cimport`扩展类型，我们将类定义分为两部分，一部分在定义文件中，另一部分在相应的实现文件中。你应该阅读 共享扩展类型 来学习这样做。

### 类型测试和铸造

假设我有一个方法`quest()`，它返回`Shrubbery`类型的对象。要访问它的宽度我可以写：

```py
cdef Shrubbery sh = quest()
print(sh.width)

```

这需要使用局部变量并在赋值时执行类型测试。如果 *知道*，`quest()`的返回值将是`Shrubbery`类型，你可以使用强制转换来写：

```py
print( (<Shrubbery>quest()).width )

```

如果`quest()`实际上不是`Shrubbery`，这可能是危险的，因为它将尝试访问宽度作为可能不存在的 C 结构成员。在 C 级别，不是提出 [`AttributeError`](https://docs.python.org/3/library/exceptions.html#AttributeError "(in Python v3.7)") ，而是返回一个无意义的结果（将该地址处的任何数据解释为 int）或者尝试访问无效内存可能导致段错误。相反，人们可以写：

```py
print( (<Shrubbery?>quest()).width )

```

在进行强制转换并允许代码继续之前执行类型检查（可能引发 [`TypeError`](https://docs.python.org/3/library/exceptions.html#TypeError "(in Python v3.7)") ）。

要显式测试对象的类型，请使用`isinstance()`内置函数。对于已知的内置或扩展类型，Cython 将这些转换为快速且安全的类型检查，忽略对象的`__class__`属性等的更改，以便在成功进行`isinstance()`测试后，代码可以依赖于预期的 C 结构扩展类型及其 `cdef` 属性和方法。

## 扩展类型和无

当您将参数或 C 变量声明为扩展类型时，Cython 将允许它采用值`None`以及其声明类型的值。这类似于 C 指针可以采用值`NULL`的方式，因此需要谨慎行事。只要您对它执行 Python 操作就没有问题，因为将应用完整的动态类型检查。但是，当您访问扩展类型的 C 属性时（如上面的 widen_shrubbery 函数），由您来确保您使用的引用不是`None` - 为了提高效率，Cython 不会检查这个。

在公开将扩展类型作为参数的 Python 函数时，您需要特别小心。如果我们想让`widen_shrubbery()`成为 Python 函数，例如，如果我们只是写道：

```py
def widen_shrubbery(Shrubbery sh, extra_width): # This is
    sh.width = sh.width + extra_width           # dangerous!

```

那么我们模块的用户可以通过传递`sh`参数的`None`来使其崩溃。

解决这个问题的一种方法是：

```py
def widen_shrubbery(Shrubbery sh, extra_width):
    if sh is None:
        raise TypeError
    sh.width = sh.width + extra_width

```

但由于预计这是一个如此频繁的要求，Cython 提供了一种更方便的方式。声明为扩展类型的 Python 函数的参数可以具有`not None`子句：

```py
def widen_shrubbery(Shrubbery sh not None, extra_width):
    sh.width = sh.width + extra_width

```

现在该函数将自动检查`sh`是否为`not None`并检查它是否具有正确的类型。

Note

`not None`子句只能用于 Python 函数（用 [`def`](https://docs.python.org/3/reference/compound_stmts.html#def "(in Python v3.7)") 定义）而不能用于 C 函数（用 `cdef` 定义）。如果需要检查 C 函数的参数是否为 None，则需要自己进行。

Note

还有一些事情：

*   保证扩展类型方法的 self 参数永远不会是`None`。
*   将值与`None`进行比较时，请记住，如果`x`是 Python 对象，`x is None`和`x is not None`非常有效，因为它们直接转换为 C 指针比较，而`x == None`和`x != None`或者简单地使用`x`作为布尔值（如在`if x: ...`中）将调用 Python 操作，因此速度要慢得多。

## 特殊方法

尽管原理类似，但许多`__xxx__()`扩展类型的特殊方法和它们的 Python 对应方之间存在很大差异。有一个 单独的页面 致力于这个主题，你应该在尝试使用扩展类型中的任何特殊方法之前仔细阅读它。

## 属性

您可以使用与普通 Python 代码中相同的语法在扩展类中声明属性：

```py
cdef class Spam:

    @property
    def cheese(self):
        # This is called when the property is read.
        ...

    @cheese.setter
    def cheese(self, value):
            # This is called when the property is written.
            ...

    @cheese.deleter
    def cheese(self):
        # This is called when the property is deleted.

```

还有一种特殊的（已弃用的）遗留语法，用于定义扩展类中的属性：

```py
cdef class Spam:

    property cheese:

        "A doc string can go here."

        def __get__(self):
            # This is called when the property is read.
            ...

        def __set__(self, value):
            # This is called when the property is written.
            ...

        def __del__(self):
            # This is called when the property is deleted.

```

`__get__()`，`__set__()`和`__del__()`方法都是可选的;如果省略它们，则在尝试相应操作时将引发异常。

这是一个完整的例子。它定义了一个属性，每次写入时都会添加到列表中，在读取列表时返回列表，并在删除列表时清空列表：

```py
# cheesy.pyx
cdef class CheeseShop:

    cdef object cheeses

    def __cinit__(self):
        self.cheeses = []

    @property
    def cheese(self):
        return "We don't have: %s" % self.cheeses

    @cheese.setter
    def cheese(self, value):
        self.cheeses.append(value)

    @cheese.deleter
    def cheese(self):
        del self.cheeses[:]

# Test input
from cheesy import CheeseShop

shop = CheeseShop()
print(shop.cheese)

shop.cheese = "camembert"
print(shop.cheese)

shop.cheese = "cheddar"
print(shop.cheese)

del shop.cheese
print(shop.cheese)

```

```py
# Test output
We don't have: []
We don't have: ['camembert']
We don't have: ['camembert', 'cheddar']
We don't have: []

```

## 子类化

扩展类型可以从内置类型或其他扩展类型继承：

```py
cdef class Parrot:
    ...

cdef class Norwegian(Parrot):
    ...

```

基本类型的完整定义必须可用于 Cython，因此如果基类型是内置类型，则它必须先前已声明为 extern 扩展类型。如果基类型在另一个 Cython 模块中定义，则必须将其声明为 extern 扩展类型或使用 `cimport` 语句导入。

扩展类型只能有一个基类（没有多重继承）。

Cython 扩展类型也可以在 Python 中进行子类化。 Python 类可以从多个扩展类型继承，前提是遵循通常的多重继承的 Python 规则（即所有基类的 C 布局必须兼容）。

有一种方法可以防止扩展类型在 Python 中被子类型化。这是通过`final`指令完成的，通常使用装饰器在扩展类型上设置：

```py
cimport cython

@cython.final
cdef class Parrot:
   def done(self): pass

```

尝试从此类型创建 Python 子类将在运行时引发 [`TypeError`](https://docs.python.org/3/library/exceptions.html#TypeError "(in Python v3.7)") 。 Cython 还将阻止在同一模块内部对最终类型进行子类型化，即创建使用最终类型的扩展类型，因为其基类型将在编译时失败。但请注意，此限制目前不会传播到其他扩展模块，因此即使是最终扩展类型仍可以通过外部代码在 C 级进行子类型化。

## C 方法

扩展类型可以有 C 方法和 Python 方法。与 C 函数一样，使用 `cdef` 或 `cpdef` 而不是 [`def`](https://docs.python.org/3/reference/compound_stmts.html#def "(in Python v3.7)") 声明 C 方法。 C 方法是“虚拟的”，可以在派生的扩展类型中重写。另外，当调用 C 方法时， `cpdef` 方法甚至可以被 python 方法覆盖。与 `cdef` 方法相比，这增加了一些他们的调用开销：

```py
# pets.pyx
cdef class Parrot:

    cdef void describe(self):
        print("This parrot is resting.")

cdef class Norwegian(Parrot):

    cdef void describe(self):
        Parrot.describe(self)
        print("Lovely plumage!")

cdef Parrot p1, p2
p1 = Parrot()
p2 = Norwegian()
print("p1:")
p1.describe()
print("p2:")
p2.describe()

```

```py
# Output
p1:
This parrot is resting.
p2:
This parrot is resting.
Lovely plumage!

```

上面的例子还说明了一个 C 方法可以使用通常的 Python 技术调用一个继承的 C 方法，即：

```py
Parrot.describe(self)

```

可以使用@staticmethod 装饰器将 &lt;cite&gt;cdef&lt;/cite&gt; 方法声明为静态。这对于构造采用非 Python 兼容类型的类特别有用：

```py
cdef class OwnedPointer:
    cdef void* ptr

    def __dealloc__(self):
        if self.ptr is not NULL:
            free(self.ptr)

    @staticmethod
    cdef create(void* ptr):
        p = OwnedPointer()
        p.ptr = ptr
        return p

```

## 前向声明扩展类型

扩展类型可以是前向声明的，如 `struct` 和 `union` 类型。这通常不是必要的，违反了 DRY 原则（不要重复自己）。

如果要向前声明具有基类的扩展类型，则必须在前向声明及其后续定义中指定基类，例如：

```py
cdef class A(B)

...

cdef class A(B):
    # attributes and methods

```

## 快速实例化

Cython 提供了两种加速扩展类型实例化的方法。第一个是直接调用`__new__()`特殊静态方法，如 Python 所知。对于扩展类型`Penguin`，您可以使用以下代码：

```py
cdef class Penguin:
    cdef object food

    def __cinit__(self, food):
        self.food = food

    def __init__(self, food):
        print("eating!")

normal_penguin = Penguin('fish')
fast_penguin = Penguin.__new__(Penguin, 'wheat')  # note: not calling __init__() !

```

请注意，通过`__new__()`的路径将 *而非* 调用类型的`__init__()`方法（再次，如 Python 所知）。因此，在上面的示例中，第一个实例化将打印`eating!`，但第二个实例化不会打印`eating!`。这只是`__cinit__()`方法比扩展类型的正常`__init__()`方法更安全和更可取的原因之一。

第二个性能改进适用于经常连续创建和删除的类型，以便它们可以从空闲列表中受益。 Cython 为此提供了装饰器`@cython.freelist(N)`，它为给定类型创建了一个静态大小的`N`实例空闲列表。例：

```py
cimport cython

@cython.freelist(8)
cdef class Penguin:
    cdef object food
    def __cinit__(self, food):
        self.food = food

penguin = Penguin('fish 1')
penguin = None
penguin = Penguin('fish 2')  # does not need to allocate memory!

```

## 从现有的 C / C ++指针实例化

想要从现有（指向 a）数据结构实例化扩展类是很常见的，通常由外部 C / C ++函数返回。

由于扩展类只能在其构造函数中接受 Python 对象作为参数，因此必须使用工厂函数。例如，

```py
from libc.stdlib cimport malloc, free

# Example C struct
ctypedef struct my_c_struct:
    int a
    int b

cdef class WrapperClass:
    """A wrapper class for a C/C++ data structure"""
    cdef my_c_struct *_ptr
    cdef bint ptr_owner

    def __cinit__(self):
        self.ptr_owner = False

    def __dealloc__(self):
        # De-allocate if not null and flag is set
        if self._ptr is not NULL and self.ptr_owner is True:
            free(self._ptr)
            self._ptr = NULL

    # Extension class properties
    @property
    def a(self):
        return self._ptr.a if self._ptr is not NULL else None

    @property
    def b(self):
        return self._ptr.b if self._ptr is not NULL else None

    @staticmethod
    cdef WrapperClass from_ptr(my_c_struct *_ptr, bint owner=False):
        """Factory function to create WrapperClass objects from
 given my_c_struct pointer.

 Setting ``owner`` flag to ``True`` causes
 the extension type to ``free`` the structure pointed to by ``_ptr``
 when the wrapper object is deallocated."""
        # Call to __new_*bypasses*_init__ constructor
        cdef WrapperClass wrapper = WrapperClass.__new__(WrapperClass)
        wrapper._ptr = _ptr
        wrapper.ptr_owner = owner
        return wrapper

    @staticmethod
    cdef WrapperClass new_struct():
        """Factory function to create WrapperClass objects with
 newly allocated my_c_struct"""
        cdef my_c_struct *_ptr = <my_c_struct *>malloc(sizeof(my_c_struct))
        if _ptr is NULL:
            raise MemoryError
        _ptr.a = 0
        _ptr.b = 0
        return WrapperClass.from_ptr(_ptr, owner=True)

```

然后，从现有的`my_c_struct`指针创建`WrapperClass`对象，可以在 Cython 代码中使用`WrapperClass.from_ptr(ptr)`。要分配新结构并同时包装它，可以使用`WrapperClass.new_struct`代替。

如果需要，可以从同一指针创建多个 Python 对象，这些指针指向相同的内存中数据，尽管在解除分配时必须小心，如上所示。此外，`ptr_owner`标志可用于控制哪个`WrapperClass`对象拥有指针并负责解除分配 - 在示例中默认设置为`False`，可以通过调用`from_ptr(ptr, owner=True)`来启用。

GIL 必须 *而不是* 在`__dealloc__`中释放，或者如果是，则使用另一个锁定，在这种情况下，或者在多次解除分配时可能发生竞争条件。

作为对象构造函数的一部分，`__cinit__`方法具有 Python 签名，这使得它无法接受`my_c_struct`指针作为参数。

尝试在 Python 签名中使用指针将导致以下错误：

```py
Cannot convert 'my_c_struct *' to Python object

```

这是因为 Cython 不能自动将指针转换为 Python 对象，这与`int`等本机类型不同。

请注意，对于本机类型，Cython 将复制该值并创建新的 Python 对象，而在上述情况下，不会复制数据，并且取消分配内存是扩展类的责任。

## 使扩展类型弱引用

默认情况下，扩展类型不支持对它们进行弱引用。您可以通过声明名为`__weakref__`的对象类型的 C 属性来启用弱引用。例如，：

```py
cdef class ExplodingAnimal:
    """This animal will self-destruct when it is
 no longer strongly referenced."""

    cdef object __weakref__

```

## 控制 CPython 中的释放和垃圾收集

Note

本节仅适用于 Python 的常用 CPython 实现。 PyPy 等其他实现的工作方式不同。

### 介绍

首先，很好理解在 CPython 中有两种方法可以触发 Python 对象的释放：CPython 对所有对象使用引用计数，并且任何引用计数为零的对象都会被立即释放。这是解除分配对象的最常用方法。例如，考虑一下

```py
>>> x = "foo"
>>> x = "bar"

```

执行第二行后，不再引用字符串`"foo"`，因此将其取消分配。这是使用`tp_dealloc`插槽完成的，可以通过实现`__dealloc__`在 Cython 中自定义。

第二种机制是循环垃圾收集器。这是为了解决诸如的循环参考循环

```py
>>> class Object:
...     pass
>>> def make_cycle():
...     x = Object()
...     y = [x]
...     x.attr = y

```

调用`make_cycle`时，会创建一个参考循环，因为`x`引用`y`，反之亦然。即使在`make_cycle`返回后无法访问`x`或`y`，两者的引用计数均为 1，因此不会立即取消分配。在常规时间，垃圾收集器运行，它会注意到参考周期（使用`tp_traverse`插槽）并将其中断。打破引用循环意味着在循环中获取一个对象并将其中的所有引用移除到其他 Python 对象（我​​们称之为 *清除* 一个对象）。清除与解除分配几乎相同，只是实际对象尚未释放。对于上例中的`x`，`x`的属性将从`x`中删除。

请注意，在参考周期中只清除一个对象就足够了，因为在清除一个对象后不再有一个周期。一旦循环中断，通常基于 refcount 的释放将实际从内存中删除对象。清除在`tp_clear`插槽中实现。正如我们刚刚解释的那样，循环中的一个对象实现`tp_clear`就足够了。

### 启用释放垃圾箱

在 CPython 中，可以创建深度递归对象。例如：

```py
>>> L = None
>>> for i in range(2**20):
...     L = [L]

```

现在假设我们删除了最后的`L`。然后`L`解除分配`L[0]`，解除分配`L[0][0]`，依此类推，直到达到`2**20`的递归深度。这种解除分配是在 C 中完成的，这种深度递归可能会溢出 C 调用堆栈，从而导致 Python 崩溃。

CPython 发明了一种称为 *垃圾桶* 的机制。它通过延迟一些解除分配来限制解除分配的递归深度。

默认情况下，Cython 扩展类型不使用垃圾桶，但可以通过将`trashcan`指令设置为`True`来启用它。例如：

```py
cimport cython
@cython.trashcan(True)
cdef class Object:
    cdef dict __dict__

```

Trashcan 用法由子类继承（除非`@cython.trashcan(False)`明确禁用）。像`list`这样的内置类型使用垃圾桶，因此它的子类默认使用垃圾桶。

### 禁用循环中断（`tp_clear`）

默认情况下，每种扩展类型都支持 CPython 的循环垃圾收集器。如果可以引用任何 Python 对象，Cython 将自动生成`tp_traverse`和`tp_clear`插槽。这通常是你想要的。

至少有一个原因可能不是你想要的：如果你需要清理`__dealloc__`特殊功能中的一些外部资源而你的对象恰好处于参考周期，垃圾收集器可能已经触发了一个`tp_clear`清除对象（参见 简介 ）。

在这种情况下，调用`__dealloc__`时，任何对象引用都会消失。现在，您的清理代码无法访问它必须清理的对象。要解决此问题，您可以使用`no_gc_clear`指令禁用特定类的清除实例：

```py
@cython.no_gc_clear
cdef class DBCursor:
    cdef DBConnection conn
    cdef DBAPI_Cursor *raw_cursor
    # ...
    def __dealloc__(self):
        DBAPI_close_cursor(self.conn.raw_conn, self.raw_cursor)

```

此示例尝试在销毁 Python 对象时通过数据库连接关闭游标。 `DBConnection`对象通过`DBCursor`的引用保持活动状态。但是如果游标恰好在引用循环中，则垃圾收集器可能会删除数据库连接引用，这使得无法清理游标。

如果使用`no_gc_clear`，则任何给定的参考循环必须包含至少一个没有 `no_gc_clear`的对象 _。否则，循环不能被破坏，这是内存泄漏。_

### 禁用循环垃圾收集

在极少数情况下，可以保证扩展类型不参与循环，但编译器将无法证明这一点。如果类永远不能引用自身，甚至间接引用它，情况就是这样。在这种情况下，您可以使用`no_gc`指令手动禁用循环收集，但要注意这样做实际上扩展类型可以参与循环可能会导致内存泄漏

```py
@cython.no_gc
cdef class UserInfo:
    cdef str name
    cdef tuple addresses

```

如果您可以确定地址仅包含对字符串的引用，则上述内容将是安全的，并且可能会产生显着的加速，具体取决于您的使用模式。

## 控制酸洗

默认情况下，Cython 将生成一个`__reduce__()`方法，以便当且仅当其每个成员都可以转换为 Python 且没有`__cinit__`方法时才允许修改扩展类型。要求此行为（即，如果无法对类进行 pickle，则在编译时抛出错误）使用`@cython.auto_pickle(True)`修饰类。也可以用`@cython.auto_pickle(False)`注释以获得在任何情况下都不生成`__reduce__`方法的旧行为。

手动实现`__reduce__`或 &lt;cite&gt;__reduce_ex__`&lt;/cite&gt; 方法也将禁用此自动生成，并可用于支持更复杂类型的酸洗。

## 公共和外部扩展类型

扩展类型可以声明为 extern 或 public。外部扩展类型声明使外部 C 代码中定义的扩展类型可用于 Cython 模块。公共扩展类型声明使得在 Cython 模块中定义的扩展类型可用于外部 C 代码。

### 外部扩展类型

外部扩展类型允许您访问 Python 核心或非 Cython 扩展模块中定义的 Python 对象的内部。

Note

在以前版本的 Pyrex 中，extern 扩展类型也用于引用另一个 Pyrex 模块中定义的扩展类型。虽然你仍然可以做到这一点，但 Cython 为此提供了更好的机制。参见 在 Cython 模块 之间共享声明。

下面是一个示例，它将让您了解内置复杂对象的 C 级成员：

```py
from __future__ import print_function

cdef extern from "complexobject.h":

    struct Py_complex:
        double real
        double imag

    ctypedef class __builtin__.complex [object PyComplexObject]:
        cdef Py_complex cval

# A function which uses the above type
def spam(complex c):
    print("Real:", c.cval.real)
    print("Imag:", c.cval.imag)

```

Note

一些重要的事情：

1.  在此示例中，已使用 `ctypedef` 类。这是因为，在 Python 头文件中，`PyComplexObject`结构声明为：

    ```py
    typedef struct {
        ...
    } PyComplexObject;

    ```

    在运行时，将导入`__builtin__.complex`的`tp_basicsize`与`sizeof(`PyComplexObject)`匹配的 Cython c 扩展模块时执行检查。如果使用一个版本的`complexobject.h`标头编译 Cython c-extension 模块但导入到具有更改标头的 Python 中，则此检查可能会失败。可以使用名称规范子句中的`check_size`来调整此检查。

2.  除了扩展类型的名称外，还指定了可以在其中找到类型对象的模块。请参阅下面的隐式导入部分。

3.  声明外部扩展类型时，不要声明任何方法。为了调用它们，不需要声明方法，因为调用是 Python 方法调用。另外，与 `struct` 和 `union` 一样，如果你的扩展类声明在块中的 `cdef` extern 内，你只需要声明您希望访问的 C 成员。

### 名称规范条款

方括号中的类声明部分是仅适用于外部或公共扩展类型的特殊功能。该条款的完整形式是：

```py
[object object_struct_name, type type_object_name, check_size cs_option]

```

哪里：

*   `object_struct_name`是为类型的 C 结构假设的名称。
*   `type_object_name`是为类型的静态声明的类型对象假定的名称。
*   `cs_option`是`warn`（默认值），`error`或`ignore`，仅用于外部扩展类型。如果`error`，在编译时找到的`sizeof(object_struct)`必须与类型的运行时`tp_basicsize`完全匹配，否则模块导入将失败并显示错误。如果`warn`或`ignore`，允许`object_struct`小于类型的`tp_basicsize`，这表示运行时类型可能是更新模块的一部分，并且外部模块的开发人员向后扩展了对象兼容的方式（仅在对象的末尾添加新字段）。如果`warn`，在这种情况下将发出警告。

条款可以按任何顺序书写。

如果扩展类型声明在块中的 `cdef` extern 内，则需要 object 子句，因为 Cython 必须能够生成与头文件中的声明兼容的代码。否则，对于 extern 扩展类型，object 子句是可选的。

对于公共扩展类型，object 和 type 子句都是必需的，因为 Cython 必须能够生成与外部 C 代码兼容的代码。

### 属性名称匹配和别名

有时，`object_struct_name`中指定的类型的 C 结构可能会使用不同的字段标签，而不是`PyTypeObject`中的标签。这在手工编码的 C 扩展中很容易发生，其中`PyTypeObject_Foo`具有 getter 方法，但名称与`PyFooObject`中的名称不匹配。例如，在 NumPy 中，python-level `dtype.itemsize`是 C struct 字段`elsize`的 getter。 Cython 支持别名字段名称，以便可以在 Cython 代码中编写`dtype.itemsize`，这些代码将被编译为 C struct 字段的直接访问，而无需通过相当于`dtype.__getattr__('itemsize')`的 C-API。

例如，我们可能有一个扩展模块`foo_extension`：

```py
cdef class Foo:
    cdef public int field0, field1, field2;

    def __init__(self, f0, f1, f2):
        self.field0 = f0
        self.field1 = f1
        self.field2 = f2

```

但是文件`foo_nominal.h`中的 C 结构：

```py
typedef struct {
     PyObject_HEAD
     int f0;
     int f1;
     int f2;
 } FooStructNominal;

```

请注意，结构使用`f0`，`f1`，`f2`，但它们是`Foo`中的`field0`，`field1`和`field2`。我们得到了这种情况，包括带有该结构的头文件，我们希望编写一个函数来对值进行求和。如果我们编写扩展模块`wrapper`：

```py
cdef extern from "foo_nominal.h":

    ctypedef class foo_extension.Foo [object FooStructNominal]:
        cdef:
            int field0
            int field1
            int feild2

def sum(Foo f):
    return f.field0 + f.field1 + f.field2

```

那么`wrapper.sum(f)`（其中`f = foo_extension.Foo(1, 2, 3)`）仍将使用相当于的 C-API：

```py
return f.__getattr__('field0') +
       f.__getattr__('field1') +
       f.__getattr__('field1')

```

而不是所需的 C 当量`return f-&gt;f0 + f-&gt;f1 + f-&gt;f2`。我们可以使用以下字段对字段进行别名：

```py
cdef extern from "foo_nominal.h":

    ctypedef class foo_extension.Foo [object FooStructNominal]:
        cdef:
            int field0 "f0"
            int field1 "f1"
            int field2 "f2"

def sum(Foo f) except -1:
    return f.field0 + f.field1 + f.field2

```

现在 Cython 将用对 CooStructNominal 字段的直接 C 访问来替换慢`__getattr__`。这在直接处理 Python 代码时很有用。即使 Python 和 C 中的字段名称不同，也不需要对 Python 进行任何更改以实现显着的加速。当然，应该确保字段是等价的。

### 隐式导入

Cython 要求您在 extern 扩展类声明中包含模块名称，例如：

```py
cdef extern class MyModule.Spam:
    ...

```

类型对象将从指定模块隐式导入，并绑定到此模块中的相应名称。换句话说，在这个例子中隐含：

```py
from MyModule import Spam

```

语句将在模块加载时执行。

模块名称可以是带点名称，用于引用包层次结构内的模块，例如：

```py
cdef extern class My.Nested.Package.Spam:
    ...

```

您还可以使用 as 子句指定用于导入类型的备用名称，例如：

```py
cdef extern class My.Nested.Package.Spam as Yummy:
   ...

```

它对应于隐式 import 语句：

```py
from My.Nested.Package import Spam as Yummy

```

### 类型名称与构造函数名称

在 Cython 模块中，扩展类型的名称有两个不同的用途。在表达式中使用时，它指的是包含类型构造函数（即其类型对象）的模块级全局变量。但是，它也可以用作 C 类型名称来声明该类型的变量，参数和返回值。

当你声明：

```py
cdef extern class MyModule.Spam:
    ...

```

Spam 这个名字兼具这两个角色。可能有其他名称可以引用构造函数，但只有 Spam 可以用作类型名称。例如，如果要显式导入 MyModule，则可以使用`MyModule.Spam()`创建 Spam 实例，但不能将`MyModule.Spam`用作类型名称。

使用 as 子句时，as 子句中指定的名称也将接管这两个角色。所以如果你宣布：

```py
cdef extern class MyModule.Spam as Yummy:
    ...

```

然后 Yummy 成为类型名称和构造函数的名称。同样，您可以通过其他方式获取构造函数，但只有 Yummy 可用作类型名称。

## 公共扩展类型

扩展类型可以声明为 public，在这种情况下会生成一个包含其对象 struct 和 type 对象声明的`.h`文件。通过将`.h`文件包含在您编写的外部 C 代码中，该代码可以访问扩展类型的属性。