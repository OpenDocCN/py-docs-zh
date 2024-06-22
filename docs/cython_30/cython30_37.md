# 实现缓冲协议

> 原文： [`docs.cython.org/en/latest/src/userguide/buffer.html`](http://docs.cython.org/en/latest/src/userguide/buffer.html)

Cython 对象可以通过实现“缓冲协议”将内存缓冲区暴露给 Python 代码。本章介绍如何实现协议并使用 NumPy 中扩展类型管理的内存。

## 矩阵类

以下 Cython / C ++代码实现了一个浮点矩阵，其中列数在构造时固定，但行可以动态添加。

```py
# distutils: language = c++

# matrix.pyx

from libcpp.vector cimport vector

cdef class Matrix:
    cdef unsigned ncols
    cdef vector[float] v

    def __cinit__(self, unsigned ncols):
        self.ncols = ncols

    def add_row(self):
        """Adds a row, initially zero-filled."""
        self.v.resize(self.v.size() + self.ncols)

```

没有方法可以用矩阵的内容做任何有效的工作。我们可以为此实现自定义`__getitem__`，`__setitem__`等，但我们将使用缓冲协议将矩阵的数据暴露给 Python，这样我们就可以使用 NumPy 来完成有用的工作。

实现缓冲协议需要添加两个方法，`__getbuffer__`和`__releasebuffer__`，Cython 专门处理。

```py
# distutils: language = c++

from cpython cimport Py_buffer
from libcpp.vector cimport vector

cdef class Matrix:
    cdef Py_ssize_t ncols
    cdef Py_ssize_t shape[2]
    cdef Py_ssize_t strides[2]
    cdef vector[float] v

    def __cinit__(self, Py_ssize_t ncols):
        self.ncols = ncols

    def add_row(self):
        """Adds a row, initially zero-filled."""
        self.v.resize(self.v.size() + self.ncols)

    def __getbuffer__(self, Py_buffer *buffer, int flags):
        cdef Py_ssize_t itemsize = sizeof(self.v[0])

        self.shape[0] = self.v.size() / self.ncols
        self.shape[1] = self.ncols

        # Stride 1 is the distance, in bytes, between two items in a row;
        # this is the distance between two adjacent items in the vector.
        # Stride 0 is the distance between the first elements of adjacent rows.
        self.strides[1] = <Py_ssize_t>(  <char *>&(self.v[1])
                                       - <char *>&(self.v[0]))
        self.strides[0] = self.ncols * self.strides[1]

        buffer.buf = <char *>&(self.v[0])
        buffer.format = 'f'                     # float
        buffer.internal = NULL                  # see References
        buffer.itemsize = itemsize
        buffer.len = self.v.size() * itemsize   # product(shape) * itemsize
        buffer.ndim = 2
        buffer.obj = self
        buffer.readonly = 0
        buffer.shape = self.shape
        buffer.strides = self.strides
        buffer.suboffsets = NULL                # for pointer arrays only

    def __releasebuffer__(self, Py_buffer *buffer):
        pass

```

方法`Matrix.__getbuffer__`填充由 Python C-API 定义的称为`Py_buffer`的描述符结构。它包含指向内存中实际缓冲区的指针，以及有关数组形状和步幅的元数据（从一个元素或行到下一个元素或行的步长）。它的`shape`和`strides`成员是必须指向类型和大小的数组`Py_ssize_t[ndim]`的指针。只要任何缓冲区查看数据，这些数组就必须保持活动状态，因此我们将它们作为成员存储在`Matrix`对象上。

代码尚未完成，但我们已经可以编译它并测试基本功能。

```py
>>> from matrix import Matrix
>>> import numpy as np
>>> m = Matrix(10)
>>> np.asarray(m)
array([], shape=(0, 10), dtype=float32)
>>> m.add_row()
>>> a = np.asarray(m)
>>> a[:] = 1
>>> m.add_row()
>>> a = np.asarray(m)
>>> a
array([[ 1.,  1.,  1.,  1.,  1.,  1.,  1.,  1.,  1.,  1.],
       [ 0.,  0.,  0.,  0.,  0.,  0.,  0.,  0.,  0.,  0.]], dtype=float32)

```

现在我们可以将`Matrix`视为 NumPy `ndarray`，并使用标准的 NumPy 操作修改其内容。

## 记忆安全和参考计数

到目前为止实施的`Matrix`类是不安全的。 `add_row`操作可以移动底层缓冲区，这会使数据上的任何 NumPy（或其他）视图无效。如果您尝试在`add_row`调用后访问值，您将获得过时的值或段错误。

这就是`__releasebuffer__`的用武之地。我们可以为每个矩阵添加一个引用计数，并在视图存在时锁定它以进行变异。

```py
# distutils: language = c++

from cpython cimport Py_buffer
from libcpp.vector cimport vector

cdef class Matrix:

    cdef int view_count

    cdef Py_ssize_t ncols
    cdef vector[float] v
    # ...

    def __cinit__(self, Py_ssize_t ncols):
        self.ncols = ncols
        self.view_count = 0

    def add_row(self):
        if self.view_count > 0:
            raise ValueError("can't add row while being viewed")
        self.v.resize(self.v.size() + self.ncols)

    def __getbuffer__(self, Py_buffer *buffer, int flags):
        # ... as before

        self.view_count += 1

    def __releasebuffer__(self, Py_buffer *buffer):
        self.view_count -= 1

```

## 标志

我们在代码中跳过了一些输入验证。 `__getbuffer__`的`flags`参数来自`np.asarray`（和其他客户端），是一个描述所请求数组类型的布尔标志的 OR。严格地说，如果标志包含`PyBUF_ND`，`PyBUF_SIMPLE`或`PyBUF_F_CONTIGUOUS`，`__getbuffer__`必须提高`BufferError`。这些宏可以是`cpython.buffer`的`cimport` .d。

（矢量矩阵结构实际上符合`PyBUF_ND`，但这会阻止`__getbuffer__`填充步幅。单行矩阵是 F-连续的，但是更大的矩阵不是。）

## 参考文献

这里使用的缓冲接口在  [**PEP 3118** ](https://www.python.org/dev/peps/pep-3118)中列出，修改缓冲液方案。

有关使用 C 语言的教程，请参阅 Jake Vanderplas 的博客 [Python 缓冲协议简介](https://jakevdp.github.io/blog/2014/05/05/introduction-to-the-python-buffer-protocol/)。

参考文档可用于 [Python 3](https://docs.python.org/3/c-api/buffer.html) 和 [Python 2](https://docs.python.org/2.7/c-api/buffer.html) 。 Py2 文档还描述了一个不再使用的旧缓冲区协议;自 Python 2.6 起，  [**PEP 3118** ](https://www.python.org/dev/peps/pep-3118)协议已经实现，旧协议仅与遗留代码相关。