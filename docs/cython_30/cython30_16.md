# 内存分配

> 原文： [`docs.cython.org/en/latest/src/tutorial/memory_allocation.html`](http://docs.cython.org/en/latest/src/tutorial/memory_allocation.html)

动态内存分配在 Python 中大多不是问题。一切都是对象，引用计数系统和垃圾收集器在不再使用时自动将内存返回给系统。

当谈到更多低级数据缓冲区时，Cython 通过 NumPy，内存视图或 Python 的 stdlib 数组类型特别支持简单类型的（多维）数组。它们是全功能的，垃圾收集的，比 C 中的裸指针更容易使用，同时仍然保持速度和静态类型的好处。参见 使用 Python 数组 和 类型记忆视图 。

但是，在某些情况下，这些对象仍然会产生不可接受的开销，这可能会导致在 C 中进行手动内存管理。

简单的 C 值和结构（例如局部变量`cdef double x`）通常在堆栈上分配并通过值传递，但对于更大和更复杂的对象（例如动态大小的双精度列表），必须手动请求内存并释放。 C 为此提供函数`malloc()`，`realloc()`和`free()`，可以从`clibc.stdlib`导入到 cython 中。他们的签名是：

```py
void* malloc(size_t size)
void* realloc(void* ptr, size_t size)
void free(void* ptr)

```

malloc 使用的一个非常简单的示例如下：

| 

```py
 1
 2
 3
 4
 5
 6
 7
 8
 9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
```

 | 

```py
import random
from libc.stdlib cimport malloc, free

def random_noise(int number=1):
    cdef int i
    # allocate number * sizeof(double) bytes of memory
    cdef double *my_array = &lt;double *&gt; malloc(number * sizeof(double))
    if not my_array:
        raise MemoryError()

    try:
        ran = random.normalvariate
        for i in range(number):
            my_array[i] = ran(0, 1)

        # ... let's just assume we do some more heavy C calculations here to make up
        # for the work that it takes to pack the C double values into Python float
        # objects below, right after throwing away the existing objects above.

        return [x for x in my_array[:number]]
    finally:
        # return the previously allocated memory to the system
        free(my_array)

```

 |

请注意，用于在 Python 堆上分配内存的 C-API 函数通常比上面的低级 C 函数更受欢迎，因为它们提供的内存实际上是在 Python 的内部内存管理系统中考虑的。它们还对较小的内存块进行了特殊优化，通过避免代价高昂的操作系统调用来加速其分配。

可以在`cpython.mem`标准声明文件中找到 C-API 函数：

```py
from cpython.mem cimport PyMem_Malloc, PyMem_Realloc, PyMem_Free

```

它们的接口和用法与相应的低级 C 函数相同。

需要记住的一件重要事情是，`malloc()`或 [`PyMem_Malloc()`](https://docs.python.org/3/c-api/memory.html#c.PyMem_Malloc "(in Python v3.7)") *获得的内存块必须手动释放*，并相应调用`free()`或 [`PyMem_Free()`](https://docs.python.org/3/c-api/memory.html#c.PyMem_Free "(in Python v3.7)") 当它们不再使用时（*必须* 总是使用匹配类型的自由函数）。否则，在 python 进程退出之前，它们不会被回收。这称为内存泄漏。

如果一块内存需要比`try..finally`块可以管理的更长的生命周期，另一个有用的习惯是将其生命周期与 Python 对象联系起来以利用 Python 运行时的内存管理，例如：

```py
from cpython.mem cimport PyMem_Malloc, PyMem_Realloc, PyMem_Free

cdef class SomeMemory:

    cdef double* data

    def __cinit__(self, size_t number):
        # allocate some memory (uninitialised, may contain arbitrary data)
        self.data = <double*> PyMem_Malloc(number * sizeof(double))
        if not self.data:
            raise MemoryError()

    def resize(self, size_t new_number):
        # Allocates new_number * sizeof(double) bytes,
        # preserving the current content and making a best-effort to
        # re-use the original data location.
        mem = <double*> PyMem_Realloc(self.data, new_number * sizeof(double))
        if not mem:
            raise MemoryError()
        # Only overwrite the pointer if the memory was really reallocated.
        # On error (mem is NULL), the originally memory has not been freed.
        self.data = mem

    def __dealloc__(self):
        PyMem_Free(self.data)  # no-op if self.data is NULL

```