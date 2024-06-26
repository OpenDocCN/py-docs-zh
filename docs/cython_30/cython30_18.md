# 使用 NumPy

> 原文： [`docs.cython.org/en/latest/src/tutorial/numpy.html`](http://docs.cython.org/en/latest/src/tutorial/numpy.html)

注意

Cython 0.16 引入了类型化的内存视图作为此处描述的 NumPy 集成的后续版本。它们比下面的缓冲区语法更容易使用，开销更少，并且可以在不需要 GIL 的情况下传递。它们应该优先于本页面中提供的语法。有关 NumPy 用户 ，请参阅 Cython。

您可以使用 Cython 中的 NumPy 与常规 Python 中的 NumPy 完全相同，但这样做会导致潜在的高速加速，因为 Cython 支持快速访问 NumPy 数组。让我们看一下如何使用一个简单的例子。

下面的代码使用过滤器对图像进行 2D 离散卷积（我相信你可以做得更好！，让它用于演示目的）。它既是有效的 Python 又是有效的 Cython 代码。我将它称为 Python 版本的`convolve_py.py`和 Cython 版本的`convolve1.pyx` - Cython 使用“.pyx”作为其文件后缀。

```py
import numpy as np

def naive_convolve(f, g):
    # f is an image and is indexed by (v, w)
    # g is a filter kernel and is indexed by (s, t),
    #   it needs odd dimensions
    # h is the output image and is indexed by (x, y),
    #   it is not cropped
    if g.shape[0] % 2 != 1 or g.shape[1] % 2 != 1:
        raise ValueError("Only odd dimensions on filter supported")
    # smid and tmid are number of pixels between the center pixel
    # and the edge, ie for a 5x5 filter they will be 2.
    #
    # The output size is calculated by adding smid, tmid to each
    # side of the dimensions of the input image.
    vmax = f.shape[0]
    wmax = f.shape[1]
    smax = g.shape[0]
    tmax = g.shape[1]
    smid = smax // 2
    tmid = tmax // 2
    xmax = vmax + 2 * smid
    ymax = wmax + 2 * tmid
    # Allocate result image.
    h = np.zeros([xmax, ymax], dtype=f.dtype)
    # Do convolution
    for x in range(xmax):
        for y in range(ymax):
            # Calculate pixel value for h at (x,y). Sum one component
            # for each pixel (s, t) of the filter g.
            s_from = max(smid - x, -smid)
            s_to = min((xmax - x) - smid, smid + 1)
            t_from = max(tmid - y, -tmid)
            t_to = min((ymax - y) - tmid, tmid + 1)
            value = 0
            for s in range(s_from, s_to):
                for t in range(t_from, t_to):
                    v = x - smid + s
                    w = y - tmid + t
                    value += g[smid - s, tmid - t] * f[v, w]
            h[x, y] = value
    return h

```

这应编译为生成`yourmod.so`（对于 Linux 系统，在 Windows 系统上，它将是`yourmod.pyd`）。我们运行 Python 会话来测试 Python 版本（从`.py` -file 导入）和编译的 Cython 模块。

```py
In [1]: import numpy as np
In [2]: import convolve_py
In [3]: convolve_py.naive_convolve(np.array([[1, 1, 1]], dtype=np.int),
...     np.array([[1],[2],[1]], dtype=np.int))
Out [3]:
array([[1, 1, 1],
 [2, 2, 2],
 [1, 1, 1]])
In [4]: import convolve1
In [4]: convolve1.naive_convolve(np.array([[1, 1, 1]], dtype=np.int),
...     np.array([[1],[2],[1]], dtype=np.int))
Out [4]:
array([[1, 1, 1],
 [2, 2, 2],
 [1, 1, 1]])
In [11]: N = 100
In [12]: f = np.arange(N*N, dtype=np.int).reshape((N,N))
In [13]: g = np.arange(81, dtype=np.int).reshape((9, 9))
In [19]: %timeit -n2 -r3 convolve_py.naive_convolve(f, g)
2 loops, best of 3: 1.86 s per loop
In [20]: %timeit -n2 -r3 convolve1.naive_convolve(f, g)
2 loops, best of 3: 1.41 s per loop

```

还没有那么大的差别;因为 C 代码仍然完全符合 Python 解释器的作用（例如，意味着为每个使用的数字分配了一个新对象）。查看生成的 html 文件，看看即使是最简单的语句，您需要快速得到什么。我们需要给 Cython 更多信息;我们需要添加类型。

## 添加类型

要添加类型，我们使用自定义 Cython 语法，因此我们现在正在破坏 Python 源兼容性。考虑一下这段代码（_ 阅读评论！_）：

```py
# tag: numpy_old
# You can ignore the previous line.
# It's for internal testing of the cython documentation.

import numpy as np

# "cimport" is used to import special compile-time information
# about the numpy module (this is stored in a file numpy.pxd which is
# currently part of the Cython distribution).
cimport numpy as np

# We now need to fix a datatype for our arrays. I've used the variable
# DTYPE for this, which is assigned to the usual NumPy runtime
# type info object.
DTYPE = np.int

# "ctypedef" assigns a corresponding compile-time type to DTYPE_t. For
# every type in the numpy module there's a corresponding compile-time
# type with a _t-suffix.
ctypedef np.int_t DTYPE_t

# "def" can type its arguments but not have a return type. The type of the
# arguments for a "def" function is checked at run-time when entering the
# function.
#
# The arrays f, g and h is typed as "np.ndarray" instances. The only effect
# this has is to a) insert checks that the function arguments really are
# NumPy arrays, and b) make some attribute access like f.shape[0] much
# more efficient. (In this example this doesn't matter though.)
def naive_convolve(np.ndarray f, np.ndarray g):
    if g.shape[0] % 2 != 1 or g.shape[1] % 2 != 1:
        raise ValueError("Only odd dimensions on filter supported")
    assert f.dtype == DTYPE and g.dtype == DTYPE

    # The "cdef" keyword is also used within functions to type variables. It
    # can only be used at the top indentation level (there are non-trivial
    # problems with allowing them in other places, though we'd love to see
    # good and thought out proposals for it).
    #
    # For the indices, the "int" type is used. This corresponds to a C int,
    # other C types (like "unsigned int") could have been used instead.
    # Purists could use "Py_ssize_t" which is the proper Python type for
    # array indices.
    cdef int vmax = f.shape[0]
    cdef int wmax = f.shape[1]
    cdef int smax = g.shape[0]
    cdef int tmax = g.shape[1]
    cdef int smid = smax // 2
    cdef int tmid = tmax // 2
    cdef int xmax = vmax + 2 * smid
    cdef int ymax = wmax + 2 * tmid
    cdef np.ndarray h = np.zeros([xmax, ymax], dtype=DTYPE)
    cdef int x, y, s, t, v, w

    # It is very important to type ALL your variables. You do not get any
    # warnings if not, only much slower code (they are implicitly typed as
    # Python objects).
    cdef int s_from, s_to, t_from, t_to

    # For the value variable, we want to use the same data type as is
    # stored in the array, so we use "DTYPE_t" as defined above.
    # NB! An important side-effect of this is that if "value" overflows its
    # datatype size, it will simply wrap around like in C, rather than raise
    # an error like in Python.
    cdef DTYPE_t value
    for x in range(xmax):
        for y in range(ymax):
            s_from = max(smid - x, -smid)
            s_to = min((xmax - x) - smid, smid + 1)
            t_from = max(tmid - y, -tmid)
            t_to = min((ymax - y) - tmid, tmid + 1)
            value = 0
            for s in range(s_from, s_to):
                for t in range(t_from, t_to):
                    v = x - smid + s
                    w = y - tmid + t
                    value += g[smid - s, tmid - t] * f[v, w]
            h[x, y] = value
    return h

```

在构建完这个并继续我的（非正式）基准测试后，我得到：

```py
In [21]: import convolve2
In [22]: %timeit -n2 -r3 convolve2.naive_convolve(f, g)
2 loops, best of 3: 828 ms per loop

```

## 高效索引

仍有瓶颈杀戮性能，那就是数组查找和分配。 `[]` -operator 仍然使用完整的 Python 操作 - 我们想要做的是直接以 C 速度访问数据缓冲区。

我们需要做的是输入`ndarray`对象的内容。我们使用特殊的“缓冲区”语法来执行此操作，必须告诉数据类型（第一个参数）和维度数量（“ndim”仅限关键字参数，如果未提供，则假定为一维）。

这些是必要的变化：

```py
...
def naive_convolve(np.ndarray[DTYPE_t, ndim=2] f, np.ndarray[DTYPE_t, ndim=2] g):
...
cdef np.ndarray[DTYPE_t, ndim=2] h = ...

```

用法：

```py
In [18]: import convolve3
In [19]: %timeit -n3 -r100 convolve3.naive_convolve(f, g)
3 loops, best of 100: 11.6 ms per loop

```

请注意这种变化的重要性。

_Gotcha_ ：这种有效的索引仅影响某些索引操作，即那些具有完全`ndim`数量的类型化整数索引的索引操作。因此，如果没有输入`v`，则查找`f[v, w]`不会被优化。另一方面，这意味着您可以继续使用 Python 对象进行复杂的动态切片等，就像未键入数组时一样。

## 进一步调整索引

数组查找仍然受到两个因素的影响：

1.  进行边界检查。

2.  检查负指数并正确处理。上面的代码是明确编码的，因此它不使用负索引，并且（希望）总是在边界内访问。我们可以添加一个装饰器来禁用边界检查：

    ```py
    ...
    cimport cython
    @cython.boundscheck(False) # turn off bounds-checking for entire function
    @cython.wraparound(False)  # turn off negative index wrapping for entire function
    def naive_convolve(np.ndarray[DTYPE_t, ndim=2] f, np.ndarray[DTYPE_t, ndim=2] g):
    ...

    ```

现在没有执行边界检查（并且，作为一个副作用，如果你'碰巧'访问越界，你将在最好的情况下崩溃你的程序，在最坏的情况下会损坏数据）。可以通过多种方式切换边界检查模式，有关详细信息，请参阅 编译器指令 。

此外，我们已禁用检查以包装负指数（例如 g [-1]给出最后一个值）。与禁用边界检查一样，如果我们尝试实际使用带有此禁用的负索引，则会发生错误。

函数调用开销现在开始发挥作用，因此我们将后两个示例与较大的 N 进行比较：

```py
In [11]: %timeit -n3 -r100 convolve4.naive_convolve(f, g)
3 loops, best of 100: 5.97 ms per loop
In [12]: N = 1000
In [13]: f = np.arange(N*N, dtype=np.int).reshape((N,N))
In [14]: g = np.arange(81, dtype=np.int).reshape((9, 9))
In [17]: %timeit -n1 -r10 convolve3.naive_convolve(f, g)
1 loops, best of 10: 1.16 s per loop
In [18]: %timeit -n1 -r10 convolve4.naive_convolve(f, g)
1 loops, best of 10: 597 ms per loop

```

（这也是一个混合基准，因为结果数组是在函数调用中分配的。）

警告

速度需要一些成本。特别是将类型对象（如我们的示例代码中的`f`，`g`和`h`）设置为`None`会很危险。将这些对象设置为`None`是完全合法的，但您可以使用它们检查它们是否为 None。所有其他用途（属性查找或索引）都可能会破坏或损坏数据（而不是像在 Python 中那样引发异常）。

实际规则有点复杂但主要信息很明确：不要使用类型化对象而不知道它们没有设置为 None。

## 更通用的代码

有可能做到：

```py
def naive_convolve(object[DTYPE_t, ndim=2] f, ...):

```

即使用 [`object`](https://docs.python.org/3/library/functions.html#object "(in Python v3.7)") 而不是`np.ndarray`。在 Python 3.0 下，这可以允许您的算法使用任何支持缓冲区接口的库;并支持例如如果有人对 Python 2.x 感兴趣，可以轻松添加 Python Imaging Library。

但是这有一些速度损失（如果类型设置为`np.ndarray`，则编译时假设编译时间更多，特别是假设数据以纯步进模式而不是间接模式存储）。