# 使用 C 库

> 原文： [`docs.cython.org/en/latest/src/tutorial/clibraries.html`](http://docs.cython.org/en/latest/src/tutorial/clibraries.html)

除了编写快速代码之外，Cython 的一个主要用例是从 Python 代码调用外部 C 库。由于 Cython 代码编译为 C 代码本身，因此直接在代码中调用 C 函数实际上是微不足道的。下面给出了在 Cython 代码中使用（和包装）外部 C 库的完整示例，包括适当的错误处理和有关为 Python 和 Cython 代码设计合适 API 的注意事项。

想象一下，您需要一种有效的方法来将整数值存储在 FIFO 队列中。由于内存非常重要，并且值实际上来自 C 代码，因此您无法在列表或双端队列中创建和存储 Python `int`对象。所以你要注意 C 中的队列实现。

经过一些网络搜索，你会发现 C 算法库 [[CAlg]](#calg) 并决定使用它的双端队列实现。但是，为了使处理更容易，您决定将其包装在可以封装所有内存管理的 Python 扩展类型中。

<colgroup><col class="label"><col></colgroup>
| [[CAlg]](#id2) | Simon Howard，C 算法库， [`c-algorithms.sourceforge.net/`](http://c-algorithms.sourceforge.net/) |

## 定义外部声明

你可以在这里下载 CAlg [。](https://codeload.github.com/fragglet/c-algorithms/zip/master)

队列实现的 C API，在头文件`c-algorithms/src/queue.h`中定义，基本上如下所示：

```py
/* queue.h */

typedef struct _Queue Queue;
typedef void *QueueValue;

Queue *queue_new(void);
void queue_free(Queue *queue);

int queue_push_head(Queue *queue, QueueValue data);
QueueValue queue_pop_head(Queue *queue);
QueueValue queue_peek_head(Queue *queue);

int queue_push_tail(Queue *queue, QueueValue data);
QueueValue queue_pop_tail(Queue *queue);
QueueValue queue_peek_tail(Queue *queue);

int queue_is_empty(Queue *queue);

```

首先，第一步是在`.pxd`文件中重新定义 C API，例如`cqueue.pxd`：

```py
# cqueue.pxd

cdef extern from "c-algorithms/src/queue.h":
    ctypedef struct Queue:
        pass
    ctypedef void* QueueValue

    Queue* queue_new()
    void queue_free(Queue* queue)

    int queue_push_head(Queue* queue, QueueValue data)
    QueueValue  queue_pop_head(Queue* queue)
    QueueValue queue_peek_head(Queue* queue)

    int queue_push_tail(Queue* queue, QueueValue data)
    QueueValue queue_pop_tail(Queue* queue)
    QueueValue queue_peek_tail(Queue* queue)

    bint queue_is_empty(Queue* queue)

```

请注意这些声明与头文件声明几乎完全相同，因此您通常可以将它们复制过来。但是，您不需要像上面那样提供 *所有* 声明，只需要在代码或其他声明中使用那些声明，这样 Cython 就可以看到它们的足够和一致的子集。然后，考虑对它们进行一些调整，以使它们在 Cython 中更舒适。

具体来说，您应该注意为 C 函数选择好的参数名称，因为 Cython 允许您将它们作为关键字参数传递。稍后更改它们是向后不兼容的 API 修改。立即选择好的名称将使这些函数更适合使用 Cython 代码。

我们上面使用的头文件的一个值得注意的差异是第一行中`Queue`结构的声明。在这种情况下，`Queue`用作 *不透明手柄*;只有被调用的库才知道里面是什么。由于没有 Cython 代码需要知道结构的内容，我们不需要声明它的内容，所以我们只提供一个空的定义（因为我们不想声明 C 头中引用的`_Queue`类型） [[1]](#id4) 。

<colgroup><col class="label"><col></colgroup>
| [[1]](#id3) | `cdef struct Queue: pass`和`ctypedef struct Queue: pass`之间存在细微差别。前者声明一种在 C 代码中引用为`struct Queue`的类型，而后者在 C 中引用为`Queue`。这是 Cython 无法隐藏的 C 语言怪癖。大多数现代 C 库使用`ctypedef`类型的结构。 |

另一个例外是最后一行。 `queue_is_empty()`函数的整数返回值实际上是一个 C 布尔值，即关于它的唯一有趣的事情是它是非零还是零，表明队列是否为空。这最好用 Cython 的`bint`类型表示，它在 C 中使用时是普通的`int`类型，但在转换为 Python 对象时映射到 Python 的布尔值`True`和`False`。这种在`.pxd`文件中收紧声明的方法通常可以简化使用它们的代码。

最好为您使用的每个库定义一个`.pxd`文件，如果 API 很大，有时甚至为每个头文件（或功能组）定义。这简化了它们在其他项目中的重用。有时，您可能需要使用标准 C 库中的 C 函数，或者想直接从 CPython 调用 C-API 函数。对于像这样的常见需求，Cython 附带了一组标准的`.pxd`文件，这些文件以易于使用的方式提供这些声明，以适应它们在 Cython 中的使用。主要包装是`cpython`，`libc`和`libcpp`。 NumPy 库还有一个标准的`.pxd`文件`numpy`，因为它经常在 Cython 代码中使用。有关提供的`.pxd`文件的完整列表，请参阅 Cython 的`Cython/Includes/`源包。

## 编写包装类

在声明我们的 C 库的 API 之后，我们可以开始设计应该包装 C 队列的 Queue 类。它将存在于名为`queue.pyx`的文件中。 [[2]](#id6)

<colgroup><col class="label"><col></colgroup>
| [[2]](#id5) | 请注意，`.pyx`文件的名称必须与带有 C 库声明的`cqueue.pxd`文件不同，因为两者都没有描述相同的代码。具有相同名称的`.pyx`文件旁边的`.pxd`文件定义`.pyx`文件中代码的导出声明。由于`cqueue.pxd`文件包含常规 C 库的声明，因此不得有与 Cython 关联的`.pyx`文件具有相同名称的文件。 |

这是 Queue 类的第一个开始：

```py
# queue.pyx

cimport cqueue

cdef class Queue:
    cdef cqueue.Queue* _c_queue

    def __cinit__(self):
        self._c_queue = cqueue.queue_new()

```

请注意，它表示`__cinit__`而不是`__init__`。虽然`__init__`也可用，但不保证可以运行（例如，可以创建子类并忘记调用祖先的构造函数）。因为没有初始化 C 指针经常导致 Python 解释器的硬崩溃，所以在 CPython 甚至考虑调用`__init__`之前，Cython 提供`__cinit__`，*始终* 在构造时立即被调用，因此它是正确的位置初始化新实例的`cdef`字段。但是，在对象构造期间调用`__cinit__`时，`self`尚未完全构造，并且必须避免对`self`执行任何操作，而是分配给`cdef`字段。

另请注意，上述方法不带参数，但子类型可能需要接受一些参数。无参数`__cinit__()`方法是一种特殊情况，它只是不接收传递给构造函数的任何参数，因此它不会阻止子类添加参数。如果参数在`__cinit__()`的签名中使用，则它们必须与用于实例化类型的类层次结构中任何声明的`__init__`类方法相匹配。

## 内存管理

在我们继续实施其他方法之前，重要的是要了解上述实现并不安全。如果在`queue_new()`调用中出现任何问题，此代码将简单地吞下错误，因此我们稍后可能会遇到崩溃。根据`queue_new()`功能的文档，上述可能失败的唯一原因是内存不足。在这种情况下，它将返回`NULL`，而它通常会返回指向新队列的指针。

Python 的方法是提出`MemoryError` [[3]](#id8) 。因此我们可以更改 init 函数，如下所示：

```py
# queue.pyx

cimport cqueue

cdef class Queue:
    cdef cqueue.Queue* _c_queue

    def __cinit__(self):
        self._c_queue = cqueue.queue_new()
        if self._c_queue is NULL:
            raise MemoryError()

```

<colgroup><col class="label"><col></colgroup>
| [[3]](#id7) | 在`MemoryError`的特定情况下，为了引发它而创建一个新的异常实例实际上可能会失败，因为我们的内存不足。幸运的是，CPython 提供了一个 C-API 函数`PyErr_NoMemory()`，可以安全地为我们提出正确的例外。只要您编写`raise MemoryError`或`raise MemoryError()`，Cython 就会自动替换此 C-API 调用。如果您使用的是旧版本，则必须从标准软件包`cpython.exc`中导入 C-API 函数并直接调用它。 |

接下来要做的是在不再使用 Queue 实例时清理（即删除了对它的所有引用）。为此，CPython 提供了 Cython 作为特殊方法`__dealloc__()`提供的回调。在我们的例子中，我们所要做的就是释放 C Queue，但前提是我们在 init 方法中成功初始化它：

```py
def __dealloc__(self):
    if self._c_queue is not NULL:
        cqueue.queue_free(self._c_queue)

```

## 编译和链接

在这一点上，我们有一个可以测试的工作 Cython 模块。要编译它，我们需要为 distutils 配置一个`setup.py`脚本。这是编译 Cython 模块的最基本脚本：

```py
from distutils.core import setup
from distutils.extension import Extension
from Cython.Build import cythonize

setup(
    ext_modules = cythonize([Extension("queue", ["queue.pyx"])])
)

```

要构建针对外部 C 库，我们需要确保 Cython 找到必要的库。存档有两种方法。首先，我们可以告诉 distutils 在哪里找到 c-source 来自动编译`queue.c`实现。或者，我们可以构建和安装 C-Alg 作为系统库并动态链接它。如果其他应用也使用 C-Alg，后者是有用的。

### 静态链接

要自动构建 c 代码，我们需要在 &lt;cite&gt;queue.pyx&lt;/cite&gt; 中包含编译器指令：

```py
# distutils: sources = c-algorithms/src/queue.c
# distutils: include_dirs = c-algorithms/src/

cimport cqueue

cdef class Queue:
    cdef cqueue.Queue* _c_queue
    def __cinit__(self):
        self._c_queue = cqueue.queue_new()
        if self._c_queue is NULL:
            raise MemoryError()

    def __dealloc__(self):
        if self._c_queue is not NULL:
            cqueue.queue_free(self._c_queue)

```

`sources`编译器指令给出了 distutils 将编译和链接（静态）到生成的扩展模块的 C 文件的路径。通常，所有相关的头文件都应该在`include_dirs`中找到。现在我们可以使用以下方法构建项目

```py
$ python setup.py build_ext -i

```

并测试我们的构建是否成功：

```py
$ python -c 'import queue; Q = queue.Queue()'

```

### 动态链接

如果我们要打包的库已经安装在系统上，则动态链接很有用。要执行动态链接，我们首先需要构建和安装 c-alg。

要在您的系统上构建 c 算法：

```py
$ cd c-algorithms
$ sh autogen.sh
$ ./configure
$ make

```

安装 CAlg 运行：

```py
$ make install

```

之后文件`/usr/local/lib/libcalg.so`应该存在。

注意

此路径适用于 Linux 系统，在其他平台上可能有所不同，因此您需要根据系统中`libcalg.so`或`libcalg.dll`的路径调整本教程的其余部分。

在这种方法中，我们需要告诉安装脚本与外部库链接。为此，我们需要扩展安装脚本以安装更改扩展设置

```py
ext_modules = cythonize([Extension("queue", ["queue.pyx"])])

```

至

```py
ext_modules = cythonize([
    Extension("queue", ["queue.pyx"],
              libraries=["calg"])
    ])

```

现在我们应该能够使用以下方法构建项目：

```py
$ python setup.py build_ext -i

```

如果 &lt;cite&gt;libcalg&lt;/cite&gt; 未安装在“普通”位置，用户可以通过传递适当的 C 编译器标志在外部提供所需的参数，例如：

```py
CFLAGS="-I/usr/local/otherdir/calg/include"  \
LDFLAGS="-L/usr/local/otherdir/calg/lib"     \
    python setup.py build_ext -i

```

在运行模块之前，我们还需要确保 &lt;cite&gt;libcalg&lt;/cite&gt; 在 &lt;cite&gt;LD_LIBRARY_PATH&lt;/cite&gt; 环境变量中，例如通过设置：

```py
$ export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/lib

```

一旦我们第一次编译模块，我们现在可以导入它并实例化一个新的队列：

```py
$ export PYTHONPATH=.
$ python -c 'import queue; Q = queue.Queue()'

```

但是，这是我们所有 Queue 类到目前为止所能做到的，所以让它更有用。

### 映射功能

在实现此类的公共接口之前，最好先查看 Python 提供的接口，例如在`list`或`collections.deque`类中。由于我们只需要 FIFO 队列，因此足以提供方法`append()`，`peek()`和`pop()`，另外还有`extend()`方法一次添加多个值。此外，由于我们已经知道所有值都来自 C，因此最好现在只提供`cdef`方法，并为它们提供直接的 C 接口。

在 C 中，数据结构通常将数据作为`void*`存储到任何数据项类型。由于我们只想存储通常适合指针类型大小的`int`值，我们可以通过一个技巧避免额外的内存分配：我们将`int`值转换为`void*`，反之亦然，并存储值直接作为指针值。

这是`append()`方法的简单实现：

```py
cdef append(self, int value):
    cqueue.queue_push_tail(self._c_queue, <void*>value)

```

同样，适用与`__cinit__()`方法相同的错误处理注意事项，因此我们最终会使用此实现：

```py
cdef append(self, int value):
    if not cqueue.queue_push_tail(self._c_queue,
                                  <void*>value):
        raise MemoryError()

```

现在添加`extend()`方法应该是直截了当的：

```py
cdef extend(self, int* values, size_t count):
    """Append all ints to the queue.
 """
    cdef int value
    for value in values[:count]:  # Slicing pointer to limit the iteration boundaries.
        self.append(value)

```

例如，当从 C 数组读取值时，这变得很方便。

到目前为止，我们只能将数据添加到队列中。下一步是编写两个方法来获取第一个元素：`peek()`和`pop()`，它们分别提供只读和破坏性读访问。为了避免在直接将`void*`转换为`int`时出现编译器警告，我们使用的中间数据类型足以容纳`void*`。在这里，`Py_ssize_t`：

```py
cdef int peek(self):
    return <Py_ssize_t>cqueue.queue_peek_head(self._c_queue)

cdef int pop(self):
    return <Py_ssize_t>cqueue.queue_pop_head(self._c_queue)

```

通常，在 C 中，当我们将较大的整数类型转换为较小的整数类型而不检查边界时，我们冒着丢失数据的风险，并且`Py_ssize_t`可能是比`int`更大的类型。但由于我们控制了如何将值添加到队列中，我们已经知道队列中的所有值都适合`int`，因此上面的转换从`void*`到`Py_ssize_t`到`int`（返回类型） ）设计安全。

### 处理错误

现在，当队列为空时会发生什么？根据文档，函数返回`NULL`指针，该指针通常不是有效值。但由于我们只是简单地向内和外输入，我们无法区分返回值是否为`NULL`，因为队列为空或者队列中存储的值为`0`。在 Cython 代码中，我们希望第一种情况引发异常，而第二种情况应该只返回`0`。为了解决这个问题，我们需要特殊情况下这个值，并检查队列是否真的是空的：

```py
cdef int peek(self) except? -1:
    cdef int value = <Py_ssize_t>cqueue.queue_peek_head(self._c_queue)
    if value == 0:
        # this may mean that the queue is empty, or
        # that it happens to contain a 0 value
        if cqueue.queue_is_empty(self._c_queue):
            raise IndexError("Queue is empty")
    return value

```

请注意我们如何在希望常见的情况下有效地创建了通过该方法的快速路径，返回值不是`0`。如果队列为空，则只有该特定情况需要额外检查。

方法签名中的`except? -1`声明属于同一类别。如果函数是返回 Python 对象值的 Python 函数，CPython 将在内部返回`NULL`而不是 Python 对象来指示异常，该异常将立即由周围的代码传播。问题是返回类型是`int`并且任何`int`值都是有效的队列项值，因此无法向调用代码显式地发出错误信号。实际上，如果没有这样的声明，Cython 就没有明显的方法可以知道在异常上返回什么以及调用代码甚至知道这个方法 *可能* 以异常退出。

调用代码可以处理这种情况的唯一方法是在从函数返回时调用`PyErr_Occurred()`以检查是否引发了异常，如果是，则传播异常。这显然有性能损失。因此，Cython 允许您声明在异常的情况下它应隐式返回的值，以便周围的代码只需在接收到此精确值时检查异常。

我们选择使用`-1`作为异常返回值，因为我们期望它是一个不太可能被放入队列的值。 `except? -1`声明中的问号表示返回值不明确（毕竟，*可能* 可能是队列中的`-1`值）并且需要使用`PyErr_Occurred()`进行额外的异常检查在调用代码。没有它，调用此方法并接收异常返回值的 Cython 代码将默默地（有时不正确地）假定已引发异常。在任何情况下，所有其他返回值将几乎没有惩罚地通过，因此再次为“正常”值创建快速路径。

既然实现了`peek()`方法，`pop()`方法也需要适应。但是，由于它从队列中删除了一个值，因此仅在删除后测试队列是否为空 _ 是不够的。相反，我们必须在进入时测试它：_

```py
cdef int pop(self) except? -1:
    if cqueue.queue_is_empty(self._c_queue):
        raise IndexError("Queue is empty")
    return <Py_ssize_t>cqueue.queue_pop_head(self._c_queue)

```

异常传播的返回值与`peek()`完全相同。

最后，我们可以通过实现`__bool__()`特殊方法以正常的 Python 方式为 Queue 提供空白指示符（注意 Python 2 调用此方法`__nonzero__`，而 Cython 代码可以使用任一名称）：

```py
def __bool__(self):
    return not cqueue.queue_is_empty(self._c_queue)

```

请注意，此方法返回`True`或`False`，因为我们在`cqueue.pxd`中将`queue_is_empty()`函数的返回类型声明为`bint`。

### 测试结果

既然实现已经完成，您可能需要为它编写一些测试以确保它正常工作。特别是 doctests 非常适合这个目的，因为它们同时提供了一些文档。但是，要启用 doctests，您需要一个可以调用的 Python API。从 Python 代码中看不到 C 方法，因此无法从 doctests 中调用。

为类提供 Python API 的一种快速方法是将方法从`cdef`更改为`cpdef`。这将让 Cython 生成两个入口点，一个可以使用 Python 调用语义和 Python 对象作为参数从普通 Python 代码调用，另一个可以使用快速 C 语义从 C 代码调用，不需要从 Python 进行中间参数转换。类型。请注意，`cpdef`方法确保 Python 方法可以适当地覆盖它们，即使从 Cython 调用它们也是如此。与`cdef`方法相比，这增加了很小的开销。

现在我们已经为我们的类提供了 C 接口和 Python 接口，我们应该确保两个接口都是一致的。 Python 用户期望`extend()`方法接受任意迭代，而 C 用户希望有一个允许传递 C 数组和 C 内存的方法。两个签名都不兼容。

我们将通过考虑在 C 中，API 也可能想要支持其他输入类型来解决此问题，例如， `long`或`char`的数组，通常支持不同名称的 C API 函数，如`extend_ints()`，`extend_longs()`，extend_chars（）``等。这允许我们释放方法名`extend()` duck typed Python 方法，可以接受任意迭代。

以下清单显示了尽可能使用`cpdef`方法的完整实现：

```py
# queue.pyx

cimport cqueue

cdef class Queue:
    """A queue class for C integer values.

 >>> q = Queue()
 >>> q.append(5)
 >>> q.peek()
 5
 >>> q.pop()
 5
 """
    cdef cqueue.Queue* _c_queue
    def __cinit__(self):
        self._c_queue = cqueue.queue_new()
        if self._c_queue is NULL:
            raise MemoryError()

    def __dealloc__(self):
        if self._c_queue is not NULL:
            cqueue.queue_free(self._c_queue)

    cpdef append(self, int value):
        if not cqueue.queue_push_tail(self._c_queue,
                                      <void*> <Py_ssize_t> value):
            raise MemoryError()

    # The `cpdef` feature is obviously not available for the original "extend()"
    # method, as the method signature is incompatible with Python argument
    # types (Python does not have pointers).  However, we can rename
    # the C-ish "extend()" method to e.g. "extend_ints()", and write
    # a new "extend()" method that provides a suitable Python interface by
    # accepting an arbitrary Python iterable.
    cpdef extend(self, values):
        for value in values:
            self.append(value)

    cdef extend_ints(self, int* values, size_t count):
        cdef int value
        for value in values[:count]:  # Slicing pointer to limit the iteration boundaries.
            self.append(value)

    cpdef int peek(self) except? -1:
        cdef int value = <Py_ssize_t> cqueue.queue_peek_head(self._c_queue)

        if value == 0:
            # this may mean that the queue is empty,
            # or that it happens to contain a 0 value
            if cqueue.queue_is_empty(self._c_queue):
                raise IndexError("Queue is empty")
        return value

    cpdef int pop(self) except? -1:
        if cqueue.queue_is_empty(self._c_queue):
            raise IndexError("Queue is empty")
        return <Py_ssize_t> cqueue.queue_pop_head(self._c_queue)

    def __bool__(self):
        return not cqueue.queue_is_empty(self._c_queue)

```

现在我们可以使用 python 脚本测试我们的 Queue 实现，例如这里`test_queue.py`：

```py
from __future__ import print_function

import time

import queue

Q = queue.Queue()

Q.append(10)
Q.append(20)
print(Q.peek())
print(Q.pop())
print(Q.pop())
try:
    print(Q.pop())
except IndexError as e:
    print("Error message:", e)  # Prints "Queue is empty"

i = 10000

values = range(i)

start_time = time.time()

Q.extend(values)

end_time = time.time() - start_time

print("Adding {} items took {:1.3f} msecs.".format(i, 1000 * end_time))

for i in range(41):
    Q.pop()

Q.pop()
print("The answer is:")
print(Q.pop())

```

作为在作者的机器上使用 10000 个数字的快速测试表明，使用来自 Cython 代码的 C `int`值的这个队列大约是使用 Cython 代码和 Python 对象值的速度的五倍，几乎是使用它的速度的八倍。 Python 循环中的 Python 代码，仍然比使用 Python 整数的 Cython 代码中的 Python 高度优化的`collections.deque`类型快两倍。

### 回调

假设您希望为用户提供一种方法，可以将队列中的值弹出到某个用户定义的事件。为此，您希望允许它们传递一个判断何时停止的谓词函数，例如：

```py
def pop_until(self, predicate):
    while not predicate(self.peek()):
        self.pop()

```

现在，让我们假设为了参数，C 队列提供了一个将 C 回调函数作为谓词的函数。 API 可能如下所示：

```py
/* C type of a predicate function that takes a queue value and returns
 * -1 for errors
 *  0 for reject
 *  1 for accept
 */
typedef int (*predicate_func)(void* user_context, QueueValue data);

/* Pop values as long as the predicate evaluates to true for them,
 * returns -1 if the predicate failed with an error and 0 otherwise.
 */
int queue_pop_head_until(Queue *queue, predicate_func predicate,
                         void* user_context);

```

C 回调函数具有通用`void*`参数是正常的，该参数允许通过 C-API 将任何类型的上下文或状态传递到回调函数中。我们将使用它来传递我们的 Python 谓词函数。

首先，我们必须定义一个带有预期签名的回调函数，我们可以将其传递给 C-API 函数：

```py
cdef int evaluate_predicate(void* context, cqueue.QueueValue value):
    "Callback function that can be passed as predicate_func"
    try:
        # recover Python function object from void* argument
        func = <object>context
        # call function, convert result into 0/1 for True/False
        return bool(func(<int>value))
    except:
        # catch any Python errors and return error indicator
        return -1

```

主要思想是将指针（a.k.a。借用的引用）作为用户上下文参数传递给函数对象。我们将调用 C-API 函数如下：

```py
def pop_until(self, python_predicate_function):
    result = cqueue.queue_pop_head_until(
        self._c_queue, evaluate_predicate,
        <void*>python_predicate_function)
    if result == -1:
        raise RuntimeError("an error occurred")

```

通常的模式是首先将 Python 对象引用转换为`void*`以将其传递给 C-API 函数，然后将其转换回 C 谓词回调函数中的 Python 对象。对`void*`的强制转换创建了借用的引用。在转换为`&lt;object&gt;`时，Cython 递增对象的引用计数，从而将借用的引用转换回拥有的引用。在谓词函数结束时，拥有的引用再次超出范围，Cython 丢弃它。

上面代码中的错误处理有点简单。具体而言，谓词函数引发的任何异常将基本上被丢弃，并且只会导致在事实之后引发普通`RuntimeError()`。这可以通过将异常存储在通过 context 参数传递的对象中并在 C-API 函数返回`-1`以指示错误之后重新引发它来改进。