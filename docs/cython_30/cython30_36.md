# 键入的内存视图

> 原文： [`docs.cython.org/en/latest/src/userguide/memoryviews.html`](http://docs.cython.org/en/latest/src/userguide/memoryviews.html)

类型化的内存视图允许有效访问内存缓冲区，例如 NumPy 阵列的内存缓冲区，而不会产生任何 Python 开销。 Memoryview 类似于当前的 NumPy 阵列缓冲支持（`np.ndarray[np.float64_t, ndim=2]`），但它们具有更多功能和更清晰的语法。

Memoryview 比旧的 NumPy 阵列缓冲支持更通用，因为它们可以处理更多种类的数据源。例如，它们可以处理 C 数组和 Cython 数组类型（ Cython 数组 ）。

memoryview 可以在任何上下文中使用（函数参数，模块级，cdef 类属性等），并且几乎可以通过 [PEP 3118](https://www.python.org/dev/peps/pep-3118/) 缓冲区接口从任何暴露可写缓冲区的对象中获取。

## 快速入门

如果您习惯使用 NumPy，以下示例将使您开始使用 Cython 内存视图。

```py
from cython.view cimport array as cvarray
import numpy as np

# Memoryview on a NumPy array
narr = np.arange(27, dtype=np.dtype("i")).reshape((3, 3, 3))
cdef int [:, :, :] narr_view = narr

# Memoryview on a C array
cdef int carr[3][3][3]
cdef int [:, :, :] carr_view = carr

# Memoryview on a Cython array
cyarr = cvarray(shape=(3, 3, 3), itemsize=sizeof(int), format="i")
cdef int [:, :, :] cyarr_view = cyarr

# Show the sum of all the arrays before altering it
print("NumPy sum of the NumPy array before assignments: %s" % narr.sum())

# We can copy the values from one memoryview into another using a single
# statement, by either indexing with ... or (NumPy-style) with a colon.
carr_view[...] = narr_view
cyarr_view[:] = narr_view
# NumPy-style syntax for assigning a single value to all elements.
narr_view[:, :, :] = 3

# Just to distinguish the arrays
carr_view[0, 0, 0] = 100
cyarr_view[0, 0, 0] = 1000

# Assigning into the memoryview on the NumPy array alters the latter
print("NumPy sum of NumPy array after assignments: %s" % narr.sum())

# A function using a memoryview does not usually need the GIL
cpdef int sum3d(int[:, :, :] arr) nogil:
    cdef size_t i, j, k, I, J, K
    cdef int total = 0
    I = arr.shape[0]
    J = arr.shape[1]
    K = arr.shape[2]
    for i in range(I):
        for j in range(J):
            for k in range(K):
                total += arr[i, j, k]
    return total

# A function accepting a memoryview knows how to use a NumPy array,
# a C array, a Cython array...
print("Memoryview sum of NumPy array is %s" % sum3d(narr))
print("Memoryview sum of C array is %s" % sum3d(carr))
print("Memoryview sum of Cython array is %s" % sum3d(cyarr))
# ... and of course, a memoryview.
print("Memoryview sum of C memoryview is %s" % sum3d(carr_view))

```

此代码应提供以下输出：

```py
NumPy sum of the NumPy array before assignments: 351
NumPy sum of NumPy array after assignments: 81
Memoryview sum of NumPy array is 81
Memoryview sum of C array is 451
Memoryview sum of Cython array is 1351
Memoryview sum of C memoryview is 451

```

## 使用记忆库视图

### 语法

内存视图以与 NumPy 类似的方式使用 Python 切片语法。

要在一维 int 缓冲区上创建完整视图：

```py
cdef int[:] view1D = exporting_object

```

完整的 3D 视图：

```py
cdef int[:,:,:] view3D = exporting_object

```

它们也可以方便地作为函数参数：

```py
def process_3d_buffer(int[:,:,:] view not None):
    ...

```

参数的`not None`声明自动拒绝 None 值作为输入，否则将允许。默认情况下允许 None 的原因是它可以方便地用于返回参数：

```py
import numpy as np

def process_buffer(int[:,:] input_view not None,
                   int[:,:] output_view=None):

   if output_view is None:
       # Creating a default view, e.g.
       output_view = np.empty_like(input_view)

   # process 'input_view' into 'output_view'
   return output_view

```

Cython 将自动拒绝不兼容的缓冲区，例如将三维缓冲区传递给需要二维缓冲区的函数会产生`ValueError`。

### 索引

在 Cython 中，内存视图上的索引访问会自动转换为内存地址。以下代码请求 C `int`类型的项目和索引的二维内存视图：

```py
cdef int[:,:] buf = exporting_object

print(buf[1,2])

```

负指数也起作用，从相应维度的末尾开始计算：

```py
print(buf[-1,-2])

```

以下函数循环遍历 2D 数组的每个维度，并为每个项目添加 1：

```py
import numpy as np

def add_one(int[:,:] buf):
    for x in range(buf.shape[0]):
        for y in range(buf.shape[1]):
            buf[x, y] += 1

# exporting_object must be a Python object
# implementing the buffer interface, e.g. a numpy array.
exporting_object = np.zeros((10, 20), dtype=np.intc)

add_one(exporting_object)

```

索引和切片可以在有或没有 GIL 的情况下完成。它基本上像 NumPy 一样工作。如果为每个维度指定了索引，您将获得基本类型的元素（例如 &lt;cite&gt;int&lt;/cite&gt; ）。否则，您将获得一个新视图。省略号表示您为每个未指定的维度获取连续切片：

```py
import numpy as np

exporting_object = np.arange(0, 15 * 10 * 20, dtype=np.intc).reshape((15, 10, 20))

cdef int[:, :, :] my_view = exporting_object

# These are all equivalent
my_view[10]
my_view[10, :, :]
my_view[10, ...]

```

### 复制

内存视图可以复制到位：

```py
import numpy as np

cdef int[:, :, :] to_view, from_view
to_view = np.empty((20, 15, 30), dtype=np.intc)
from_view = np.ones((20, 15, 30), dtype=np.intc)

# copy the elements in from_view to to_view
to_view[...] = from_view
# or
to_view[:] = from_view
# or
to_view[:, :, :] = from_view

```

也可以使用`copy()`和`copy_fortran()`方法复制它们;见 C 和 Fortran 连续拷贝 。

### 转置

在大多数情况下（见下文），内存视图的转置方式与 NumPy 切片可以转置的方式相同：

```py
import numpy as np

array = np.arange(20, dtype=np.intc).reshape((2, 10))

cdef int[:, ::1] c_contig = array
cdef int[::1, :] f_contig = c_contig.T

```

这为数据提供了一个新的转置视图。

转置要求存储器视图的所有维度都具有直接存取存储器布局（即，没有通过指针的间接指令）。有关详细信息，请参阅 指定更一般的存储器布局 。

### Newaxis

对于 NumPy，可以通过使用`None`索引数组来引入新轴

```py
cdef double[:] myslice = np.linspace(0, 10, num=50)

# 2D array with shape (1, 50)
myslice[None] # or
myslice[None, :]

# 2D array with shape (50, 1)
myslice[:, None]

# 3D array with shape (1, 10, 1)
myslice[None, 10:-20:2, None]

```

可以将新的轴索引与所有其他形式的索引和切片混合。另见[示例](https://docs.scipy.org/doc/numpy/reference/arrays.indexing.html)。

### 只读视图

从 Cython 0.28 开始，memoryview 项类型可以声明为`const`以支持只读缓冲区作为输入：

```py
import numpy as np

cdef const double[:] myslice   # const item type => read-only view

a = np.linspace(0, 10, num=50)
a.setflags(write=False)
myslice = a

```

使用带有二进制 Python 字符串的非 const 内存视图会产生运行时错误。您可以使用`const`内存视图解决此问题：

```py
cdef bint is_y_in(const unsigned char[:] string_view):
    cdef int i
    for i in range(string_view.shape[0]):
        if string_view[i] == b'y':
            return True
    return False

print(is_y_in(b'hello world'))   # False
print(is_y_in(b'hello Cython'))  # True

```

注意，这不是 *要求* 输入缓冲区是只读的：

```py
a = np.linspace(0, 10, num=50)
myslice = a   # read-only view of a writable buffer

```

`const`视图仍然接受可写缓冲区，但非 const，可写视图不接受只读缓冲区：

```py
cdef double[:] myslice   # a normal read/write memory view

a = np.linspace(0, 10, num=50)
a.setflags(write=False)
myslice = a   # ERROR: requesting writable memory view from read-only buffer!

```

## 与旧缓冲支持的比较

您可能更喜欢使用旧视图的内存视图，因为：

*   语法更清晰
*   Memoryview 通常不需要 GIL（参见 Memoryviews 和 GIL）
*   记忆视图要快得多

例如，这是上面`sum3d`函数的旧语法等价物：

```py
cpdef int old_sum3d(object[int, ndim=3, mode='strided'] arr):
    cdef int I, J, K, total = 0
    I = arr.shape[0]
    J = arr.shape[1]
    K = arr.shape[2]
    for i in range(I):
        for j in range(J):
            for k in range(K):
                total += arr[i, j, k]
    return total

```

请注意，我们不能像上面`sum3d`的 memoryview 版本那样使用`nogil`作为函数的缓冲版本，因为缓冲区对象是 Python 对象。但是，即使我们不将`nogil`与 memoryview 一起使用，它也要快得多。这是导入两个版本后 IPython 会话的输出：

```py
In [2]: import numpy as np

In [3]: arr = np.zeros((40, 40, 40), dtype=int)

In [4]: timeit -r15 old_sum3d(arr)
1000 loops, best of 15: 298 us per loop

In [5]: timeit -r15 sum3d(arr)
1000 loops, best of 15: 219 us per loop

```

## Python 缓冲支持

Cython 内存视图几乎支持导出 Python [新样式缓冲区](https://docs.python.org/3/c-api/buffer.html)接口的所有对象。这是 [PEP 3118](https://www.python.org/dev/peps/pep-3118/) 中描述的缓冲接口。 NumPy 数组支持此接口， Cython 数组 也是如此。 “几乎所有”是因为 Python 缓冲区接口允许数据数组中的 *元素* 本身成为指针; Cython 的内存视图还不支持这一点。

## 存储器布局

缓冲区接口允许对象以各种方式识别底层内存。除了数据元素的指针外，Cython 内存视图支持所有 Python 新型缓冲区布局。如果内存必须是外部例程的特定格式或代码优化，则了解或指定内存布局可能很有用。

### 背景

概念如下：有数据访问和数据打包。数据访问意味着直接（无指针）或间接（指针）。数据打包意味着您的数据在内存中可能是连续的或不连续的，并且可以使用 *步幅* 来识别连续索引需要为每个维度进行的内存跳转。

NumPy 数组提供了一个良好的跨步直接数据访问模型，因此我们将使用它们来更新 C 和 Fortran 连续数组的概念以及数据步幅。

### 简要概述 C，Fortran 和跨步存储器布局

最简单的数据布局可能是 C 连续数组。这是 NumPy 和 Cython 数组中的默认布局。 C 连续意味着阵列数据在存储器中是连续的（见下文），并且阵列的第一维中的相邻元素在存储器中是最远的，而最后维中的相邻元素最接近在一起。例如，在 NumPy 中：

```py
In [2]: arr = np.array([['0', '1', '2'], ['3', '4', '5']], dtype='S1')

```

这里，`arr[0, 0]`和`arr[0, 1]`在存储器中相隔一个字节，而`arr[0, 0]`和`arr[1, 0]`相距 3 个字节。这引出了我们 *步幅* 的想法。数组的每个轴都有一个步长，这是从该轴上的一个元素到下一个元素所需的字节数。在上面的例子中，轴 0 和 1 的步幅显然是：

```py
In [3]: arr.strides
Out[4]: (3, 1)

```

对于 3D C 连续数组：

```py
In [5]: c_contig = np.arange(24, dtype=np.int8).reshape((2,3,4))
In [6] c_contig.strides
Out[6]: (12, 4, 1)

```

Fortran 连续数组具有相反的内存排序，第一个轴上的元素在内存中最接近：

```py
In [7]: f_contig = np.array(c_contig, order='F')
In [8]: np.all(f_contig == c_contig)
Out[8]: True
In [9]: f_contig.strides
Out[9]: (1, 2, 6)

```

连续数组是单个连续存储器块包含数组元素的所有数据的数组，因此存储器块长度是数组中元素数量和元素大小（以字节为单位）的乘积。在上面的示例中，内存块长度为 2 * 3 * 4 * 1 个字节，其中 1 是 int8 的长度。

数组可以是连续的而不是 C 或 Fortran 顺序：

```py
In [10]: c_contig.transpose((1, 0, 2)).strides
Out[10]: (4, 12, 1)

```

切片 NumPy 数组很容易使它不连续：

```py
In [11]: sliced = c_contig[:,1,:]
In [12]: sliced.strides
Out[12]: (12, 1)
In [13]: sliced.flags
Out[13]:
C_CONTIGUOUS : False
F_CONTIGUOUS : False
OWNDATA : False
WRITEABLE : True
ALIGNED : True
UPDATEIFCOPY : False

```

### 内存视图布局的默认行为

正如您在 中看到的那样，指定更一般的内存布局 ，您可以为内存视图的任何维度指定内存布局。对于未指定布局的任何维度，假定数据访问是直接的，并假设数据打包是跨越的。例如，这将是内存视图的假设，如：

```py
int [:, :, :] my_memoryview = obj

```

### C 和 Fortran 连续的内存视图

您可以使用定义中的`::1`步骤语法为内存视图指定 C 和 Fortran 连续布局。例如，如果您确定您的内存视图将位于 3D C 连续布局之上，您可以编写：

```py
cdef int[:, :, ::1] c_contiguous = c_contig

```

其中`c_contig`可以是 C 连续的 NumPy 数组。第 3 位的`::1`意味着该第 3 维中的元素将是存储器中的一个元素。如果您知道将拥有 3D Fortran 连续数组：

```py
cdef int[::1, :, :] f_contiguous = f_contig

```

例如，如果传递非连续缓冲区

```py
# This array is C contiguous
c_contig = np.arange(24).reshape((2,3,4))
cdef int[:, :, ::1] c_contiguous = c_contig

# But this isn't
c_contiguous = np.array(c_contig, order='F')

```

你会在运行时得到`ValueError`：

```py
/Users/mb312/dev_trees/minimal-cython/mincy.pyx in init mincy (mincy.c:17267)()
    69
    70 # But this isn't
---> 71 c_contiguous = np.array(c_contig, order='F')
    72
    73 # Show the sum of all the arrays before altering it

/Users/mb312/dev_trees/minimal-cython/stringsource in View.MemoryView.memoryview_cwrapper (mincy.c:9995)()

/Users/mb312/dev_trees/minimal-cython/stringsource in View.MemoryView.memoryview.__cinit__ (mincy.c:6799)()

ValueError: ndarray is not C-contiguous

```

因此，切片类型规范中的 &lt;cite&gt;:: 1&lt;/cite&gt; 指示数据在哪个维度上是连续的。它只能用于指定完整的 C 或 Fortran 连续性。

### C 和 Fortran 连续副本

可以使用`.copy()`和`.copy_fortran()`方法使 C 或 Fortran 连续复制：

```py
# This view is C contiguous
cdef int[:, :, ::1] c_contiguous = myview.copy()

# This view is Fortran contiguous
cdef int[::1, :] f_contiguous_slice = myview.copy_fortran()

```

### 指定更一般的内存布局

可以使用先前看到的`::1`切片语法或使用`cython.view`中的任何常量指定数据布局。如果在任何维度中都没有给出说明符，则假定数据访问是直接的，并假设数据打包是跨步的。如果你不知道维度是直接的还是间接的（因为你可能从某个库得到一个带缓冲接口的对象），那么你可以指定&lt;cite&gt;泛型&lt;/cite&gt;标志，在这种情况下它将在运行时确定。

标志如下：

*   通用 - 跨步，直接或间接
*   跨步 - 跨步（直接）（这是默认值）
*   间接 - 跨步和间接的
*   连续的 - 连续的和直接的
*   indirect_contiguous - 指针列表是连续的

它们可以像这样使用：

```py
from cython cimport view

# direct access in both dimensions, strided in the first dimension, contiguous in the last
cdef int[:, ::view.contiguous] a

# contiguous list of pointers to contiguous lists of ints
cdef int[::view.indirect_contiguous, ::1] b

# direct or indirect in the first dimension, direct in the second dimension
# strided in both dimensions
cdef int[::view.generic, :] c

```

只有间接维度后面的第一个，最后一个或维度可以指定为连续的：

```py
from cython cimport view

# VALID
cdef int[::view.indirect, ::1, :] a
cdef int[::view.indirect, :, ::1] b
cdef int[::view.indirect_contiguous, ::1, :] c

```

```py
# INVALID
cdef int[::view.contiguous, ::view.indirect, :] d
cdef int[::1, ::view.indirect, :] e

```

&lt;cite&gt;连续&lt;/cite&gt;标志和 &lt;cite&gt;:: 1&lt;/cite&gt; 说明符之间的区别在于前者仅指定一个维度的邻接，而后者指定所有后续（Fortran）或前一个（C）的邻接。 ）尺寸：

```py
cdef int[:, ::1] c_contig = ...

# VALID
cdef int[:, ::view.contiguous] myslice = c_contig[::2]

# INVALID
cdef int[:, ::1] myslice = c_contig[::2]

```

前一种情况是有效的，因为最后一个维度仍然是连续的，但是第一个维度不再“跟随”最后一个维度（意思是，它已经跨越了，但它不再是 C 或 Fortran 连续），因为它被切成了。

## Memoryviews 和 GIL

正如您将从 快速入门 部分中看到的那样，内存视图通常不需要 GIL：

```py
cpdef int sum3d(int[:, :, :] arr) nogil:
    ...

```

特别是，您不需要 GIL 进行内存视图索引，切片或转置。 Memoryviews 需要 GIL 用于复制方法（ C 和 Fortran 连续拷贝 ），或者当 dtype 是对象并且读取或写入对象元素时。

## Memoryview 对象和 Cython 阵列

这些类型化的内存视图可以转换为 Python 内存视图对象（ &lt;cite&gt;cython.view.memoryview&lt;/cite&gt; ）。这些 Python 对象可以像原始内存视图一样进行索引，可切片和转换。它们也可以随时转换回 Cython 空间的内存视图。

它们具有以下属性：

> *   `shape`：每个维度的大小，作为元组。
> *   `strides`：沿每个维度跨步，以字节为单位。
> *   `suboffsets`
> *   `ndim`：维数。
> *   `size`：视图中的项目总数（形状的乘积）。
> *   `itemsize`：视图中项目的大小（以字节为单位）。
> *   `nbytes`：等于`size`乘以`itemsize`。
> *   `base`

当然还有前面提到的`T`属性（ Transposing）。这些属性与 [NumPy](https://docs.scipy.org/doc/numpy/reference/arrays.ndarray.html#memory-layout) 具有相同的语义。例如，要检索原始对象：

```py
import numpy
cimport numpy as cnp

cdef cnp.int32_t[:] a = numpy.arange(10, dtype=numpy.int32)
a = a[::2]

print(a)
print(numpy.asarray(a))
print(a.base)

# this prints:
#    <MemoryView of 'ndarray' object>
#    [0 2 4 6 8]
#    [0 1 2 3 4 5 6 7 8 9]

```

请注意，此示例返回从中获取视图的原始对象，并在此期间重新查看视图。

## Cython 数组

每当复制 Cython 内存视图时（使用任何&lt;cite&gt;副本&lt;/cite&gt;或 &lt;cite&gt;copy_fortran&lt;/cite&gt; 方法），您将获得新创建的`cython.view.array`对象的新内存视图片段。此数组也可以手动使用，并自动分配数据块。稍后可以将其分配给 C 或 Fortran 连续切片（或跨步切片）。它可以像：

```py
from cython cimport view

my_array = view.array(shape=(10, 2), itemsize=sizeof(int), format="i")
cdef int[:, :] my_slice = my_array

```

它还需要一个可选参数&lt;cite&gt;模式&lt;/cite&gt;（'c'或'fortran'）和一个布尔 &lt;cite&gt;allocate_buffer&lt;/cite&gt; ，它指示缓冲区是否应该在超出范围时分配和释放：

```py
cdef view.array my_array = view.array(..., mode="fortran", allocate_buffer=False)
my_array.data = <char *> my_data_pointer

# define a function that can deallocate the data (if needed)
my_array.callback_free_data = free

```

您还可以将指针转换为数组，或将 C 数组转换为数组：

```py
cdef view.array my_array = <int[:10, :2]> my_data_pointer
cdef view.array my_array = <int[:, :]> my_c_array

```

当然，您也可以立即将 cython.view.array 分配给类型化的 memoryview 切片。可以将 C 数组直接分配给 memoryview 切片：

```py
cdef int[:, ::1] myslice = my_2d_c_array

```

这些数组可以像 Python 内存对象一样从 Python 空间进行索引和切片，并且具有与 memoryview 对象相同的属性。

## Python 数组模块

`cython.view.array`的替代方案是 Python 标准库中的`array`模块。在 Python 3 中，`array.array`类型本身支持缓冲区接口，因此内存视图无需额外设置即可在其上工作。

但是，从 Cython 0.17 开始，可以在 Python 2 中使用这些数组作为缓冲提供程序。这是通过显式 cimporting `cpython.array`模块完成的，如下所示：

```py
cimport cpython.array

def sum_array(int[:] view):
    """
 >>> from array import array
 >>> sum_array( array('i', [1,2,3]) )
 6
 """
    cdef int total
    for i in range(view.shape[0]):
        total += view[i]
    return total

```

请注意，cimport 还为数组类型启用旧的缓冲区语法。因此，以下也有效：

```py
from cpython cimport array

def sum_array(array.array[int] arr):  # using old buffer syntax
    ...

```

## 强制到 NumPy

Memoryview（和数组）对象可以强制转换为 NumPy ndarray，而无需复制数据。你可以，例如做：

```py
cimport numpy as np
import numpy as np

numpy_array = np.asarray(<np.int32_t[:10, :10]> my_pointer)

```

当然，您不限于使用 NumPy 的类型（例如此处的`np.int32_t`），您可以使用任何可用的类型。

## 无切片

尽管 memoryview 切片不是对象，但它们可以设置为 None，并且可以检查它们是否为 None：

```py
def func(double[:] myarray = None):
    print(myarray is None)

```

如果函数需要实内存视图作为输入，则最好在签名中直接拒绝 None 输入，这在 Cython 0.17 及更高版本中受支持，如下所示：

```py
def func(double[:] myarray not None):
    ...

```

与扩展类的对象属性不同，memoryview 切片不会初始化为 None。

## 通过指针从 C 函数传递数据

由于在 C 中使用指针是无处不在的，这里我们给出一个快速示例，说明如何调用其参数包含指针的 C 函数。假设您想要使用 NumPy 管理数组（分配和释放）（它也可以是 Python 数组，或者任何支持缓冲区接口的数组），但是您希望使用`C_func_file.c`中实现的外部 C 函数对此数组执行计算]：

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
```

 | 

```py
#include "C_func_file.h"

void multiply_by_10_in_C(double arr[], unsigned int n)
{
    unsigned int i;
    for (i = 0; i &lt; n; i++) {
        arr[i] *= 10;
    }
}

```

 |

此文件附带一个名为`C_func_file.h`的头文件，其中包含：

| 

```py
1
2
3
4
5
6
```

 | 

```py
#ifndef C_FUNC_FILE_H
#define C_FUNC_FILE_H

void multiply_by_10_in_C(double arr[], unsigned int n);

#endif

```

 |

其中`arr`指向数组，`n`是其大小。

您可以通过以下方式在 Cython 文件中调用该函数：

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
24
25
26
27
28
```

 | 

```py
cdef extern from "C_func_file.c":
    # C is include here so that it doesn't need to be compiled externally
    pass

cdef extern from "C_func_file.h":
    void multiply_by_10_in_C(double *, unsigned int)

import numpy as np

def multiply_by_10(arr): # 'arr' is a one-dimensional numpy array

    if not arr.flags['C_CONTIGUOUS']:
        arr = np.ascontiguousarray(arr) # Makes a contiguous copy of the numpy array.

    cdef double[::1] arr_memview = arr

    multiply_by_10_in_C(&arr_memview[0], arr_memview.shape[0])

    return arr

a = np.ones(5, dtype=np.double)
print(multiply_by_10(a))

b = np.ones(10, dtype=np.double)
b = b[::2]  # b is not contiguous.

print(multiply_by_10(b))  # but our function still works as expected.

```

 |

Several things to note:

*   `::1`请求 C 连续视图，如果缓冲区不是 C 连续则失败。参见 C 和 Fortran 连续记忆视图 。
*   `&arr_memview[0]`可以理解为'存储器视图的第一个元素的地址'。对于连续数组，这相当于平坦内存缓冲区的起始地址。
*   `arr_memview.shape[0]`可能被`arr_memview.size`，`arr.shape[0]`或`arr.size`取代。但`arr_memview.shape[0]`更有效，因为它不需要任何 Python 交互。
*   如果传递的数组是连续的，`multiply_by_10`将就地执行计算，如果`arr`不连续，它将返回一个新的 numpy 数组。
*   如果您使用的是 Python 数组而不是 numpy 数组，则无需检查数据是否连续存储，因为总是如此。参见 使用 Python 数组 。

这样，您可以调用类似于普通 Python 函数的 C 函数，并将所有内存管理和清理留给 NumPy 数组和 Python 的对象处理。有关如何在 C 文件中编译和调用函数的详细信息，请参阅 使用 C 库 。