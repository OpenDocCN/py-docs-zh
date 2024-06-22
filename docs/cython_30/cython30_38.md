> 原文： [`docs.cython.org/en/latest/src/userguide/parallelism.html`](http://docs.cython.org/en/latest/src/userguide/parallelism.html)

# 使用并行性

Cython 通过 `cython.parallel` 模块支持原生并行性。要使用这种并行性，必须释放 GIL（参见 释放 GIL）。它目前支持 OpenMP，但稍后可能会支持更多后端。

注意

由于 OpenMP 限制，此模块中的功能仅可用于主线程或并行区域。

`cython.parallel.``prange`(_[start,] stop[, step][, nogil=False][, schedule=None[, chunksize=None]][, num_threads=None]_)

此函数可用于并行循环。 OpenMP 自动启动线程池并根据使用的计划分配工作。

线程局部性和减少量是自动推断的变量。

如果分配给 prange 块中的变量，它将变为 lastprivate，这意味着变量将包含上次迭代的值。如果对变量使用 inplace 运算符，它将变为减少，这意味着变量的线程局部副本的值将随操作符一起减少，并在循环后分配给原始变量。索引变量始终是 lastprivate。与块并行分配的变量在块之后将是私有的并且不可用，因为没有顺序最后值的概念。

&lt;colgroup&gt;&lt;col class="field-name"&gt; &lt;col class="field-body"&gt;&lt;/colgroup&gt;
| 参数： | 

*   **start** - 指示循环开始的索引（与范围中的起始参数相同）。
*   **stop** - 指示何时停止循环的索引（与范围中的停止参数相同）。
*   **步** - 给出序列步骤的整数（与范围中的步骤参数相同）。它不能为 0。
*   **nogil** - 此功能只能与发布的 GIL 一起使用。如果`nogil`为真，则循环将包裹在 nogil 部分中。
*   **schedule** –

    `schedule`传递给 OpenMP，可以是以下之一：

    static:

    If a chunksize is provided, iterations are distributed to all threads ahead of time in blocks of the given chunksize. If no chunksize is given, the iteration space is divided into chunks that are approximately equal in size, and at most one chunk is assigned to each thread in advance.

    当调度开销很重要并且可以将问题简化为已知具有大致相同运行时的相同大小的块时，这是最合适的。

    dynamic:

    The iterations are distributed to threads as they request them, with a default chunk size of 1.

    当每个块的运行时间不同并且事先不知道时，这是合适的，因此使用更多数量的较小块来保持所有线程忙。

    guided:

    As with dynamic scheduling, the iterations are distributed to threads as they request them, but with decreasing chunk size. The size of each chunk is proportional to the number of unassigned iterations divided by the number of participating threads, decreasing to 1 (or the chunksize if provided).

    这比纯动态调度有一个优势，当事实证明最后一个块比预期花费更多时间或者其他方式被错误调度时，所以大多数线程开始运行空闲而最后一个块只由较少数量的线程处理。

    runtime:

    The schedule and chunk size are taken from the runtime scheduling variable, which can be set through the `openmp.omp_set_schedule()` function call, or the OMP_SCHEDULE environment variable. Note that this essentially disables any static compile time optimisations of the scheduling code itself and may therefore show a slightly worse performance than when the same scheduling policy is statically configured at compile time. The default schedule is implementation defined. For more information consult the OpenMP specification [[1]](#id2).

*   **num_threads** - `num_threads`参数表示团队应该包含多少个线程。如果没有给出，OpenMP 将决定使用多少线程。通常，这是计算机上可用的核心数。但是，这可以通过`omp_set_num_threads()`功能或`OMP_NUM_THREADS`环境变量来控制。
*   **chunksize** - `chunksize`参数指示用于在线程之间划分迭代的块大小。这仅对`static`，`dynamic`和`guided`调度有效，并且是可选的。根据日程安排，它提供的负载平衡，调度开销和错误共享的数量（如果有的话），不同的组块可以提供显着不同的性能结果。

 |
| --- | --- |

减少的例子：

```py
from cython.parallel import prange

cdef int i
cdef int n = 30
cdef int sum = 0

for i in prange(n, nogil=True):
    sum += i

print(sum)

```

带有类型化内存视图的示例（例如 NumPy 数组）：

```py
from cython.parallel import prange

def func(double[:] x, double alpha):
    cdef Py_ssize_t i

    for i in prange(x.shape[0]):
        x[i] = alpha * x[i]

```

`cython.parallel.``parallel`(_num_threads=None_)

该指令可用作`with`语句的一部分，以并行执行代码序列。这对于设置 prange 使用的线程局部缓冲区非常有用。包含的 prange 将是一个不平行的工作共享循环，因此在并行部分中分配给的任何变量对于 prange 也是私有的。在并行块之后，并行块中的私有变量不可用。

线程局部缓冲区的示例：

```py
from cython.parallel import parallel, prange
from libc.stdlib cimport abort, malloc, free

cdef Py_ssize_t idx, i, n = 100
cdef int * local_buf
cdef size_t size = 10

with nogil, parallel():
    local_buf = &lt;int *&gt; malloc(sizeof(int) * size)
    if local_buf is NULL:
        abort()

    # populate our local buffer in a sequential loop
    for i in xrange(size):
        local_buf[i] = i * 2

    # share the work using the thread-local buffer(s)
    for i in prange(n, schedule='guided'):
        func(local_buf)

    free(local_buf)

```

稍后可能会在并行块中支持部分，以在线程之间分配工作的代码部分。

`cython.parallel.``threadid`()

返回线程的 id。对于 n 个线程，id 的范围从 0 到 n-1。

## 编译

要实际使用 OpenMP 支持，您需要告诉 C 或 C ++编译器启用 OpenMP。对于 gcc，这可以在 setup.py 中完成如下：

```py
from distutils.core import setup
from distutils.extension import Extension
from Cython.Build import cythonize

ext_modules = [
    Extension(
        "hello",
        ["hello.pyx"],
        extra_compile_args=['-fopenmp'],
        extra_link_args=['-fopenmp'],
    )
]

setup(
    name='hello-parallel-world',
    ext_modules=cythonize(ext_modules),
)

```

对于 Microsoft Visual C ++编译器，请使用`'/openmp'`而不是`'-fopenmp'`。

## 打破循环

并行和 prange 块支持语句中断，继续并以 nogil 模式返回。此外，在这些块中使用`with gil`块是有效的，并且具有从它们传播的异常。但是，因为块使用 OpenMP，所以不能只留下它们，因此现有的过程是最好的努力。对于 prange（），这意味着在任何线程中的任何后续迭代的第一次中断，返回或异常之后跳过循环体。如果可能返回多个不同的值，则返回哪个值是未定义的，因为迭代没有特定的顺序：

```py
from cython.parallel import prange

cdef int func(Py_ssize_t n):
    cdef Py_ssize_t i

    for i in prange(n, nogil=True):
        if i == 8:
            with gil:
                raise Exception()
        elif i == 4:
            break
        elif i == 2:
            return i

```

在上面的例子中，未定义是否应该引发异常，它是否会简单地中断或者它是否会返回 2。

## 使用 OpenMP 函数

可以通过 cimporting `openmp`使用 OpenMP 函数：

```py
# tag: openmp
# You can ignore the previous line.
# It's for internal testing of the Cython documentation.

from cython.parallel cimport parallel
cimport openmp

cdef int num_threads

openmp.omp_set_dynamic(1)
with nogil, parallel():
    num_threads = openmp.omp_get_num_threads()
    # ...

```

参考

<colgroup><col class="label"><col></colgroup>
| [[1]](#id1) | [`www.openmp.org/mp-documents/spec30.pdf`](https://www.openmp.org/mp-documents/spec30.pdf) |