# 使用 Python 数组

> 原文： [`docs.cython.org/en/latest/src/tutorial/array.html`](http://docs.cython.org/en/latest/src/tutorial/array.html)

Python 有一个内置数组模块，支持原始类型的动态一维数组。可以从 Cython 中访问 Python 数组的底层 C 数组。同时它们是普通的 Python 对象，当使用 [`multiprocessing`](https://docs.python.org/3/library/multiprocessing.html#module-multiprocessing "(in Python v3.7)") 时，它们可以存储在列表中并在进程之间进行序列化。

与使用`malloc()`和`free()`的手动方法相比，这提供了 Python 的安全和自动内存管理，并且与 Numpy 数组相比，不需要安装依赖项，如 [`array`](https://docs.python.org/3/library/array.html#module-array "(in Python v3.7)") 模块内置于 Python 和 Cython 中。

## 内存视图的安全使用

```py
from cpython cimport array
import array
cdef array.array a = array.array('i', [1, 2, 3])
cdef int[:] ca = a

print(ca[0])

```

注意：导入会将常规 Python 数组对象带入命名空间，而 cimport 会添加可从 Cython 访问的函数。

Python 数组使用类型签名和初始值序列构造。有关可能的类型签名，请参阅[阵列模块](https://docs.python.org/library/array.html)的 Python 文档。

请注意，当将 Python 数组分配给类型为内存视图的变量时，构造内存视图会有轻微的开销。但是，从那时起，变量可以无需开销即可传递给其他函数，只要输入即可：

```py
from cpython cimport array
import array

cdef array.array a = array.array('i', [1, 2, 3])
cdef int[:] ca = a

cdef int overhead(object a):
    cdef int[:] ca = a
    return ca[0]

cdef int no_overhead(int[:] ca):
    return ca[0]

print(overhead(a))  # new memory view will be constructed, overhead
print(no_overhead(ca))  # ca is already a memory view, so no overhead

```

## 零开销，不安全访问原始 C 指针

为了避免任何开销并且能够将 C 指针传递给其他函数，可以将底层连续数组作为指针进行访问。没有类型或边界检查，因此请小心使用正确的类型和签名。

```py
from cpython cimport array
import array

cdef array.array a = array.array('i', [1, 2, 3])

# access underlying pointer:
print(a.data.as_ints[0])

from libc.string cimport memset

memset(a.data.as_voidptr, 0, len(a) * sizeof(int))

```

请注意，对数组对象的任何长度更改操作都可能使指针无效。

## 克隆，扩展数组

为了避免必须使用 Python 模块中的数组构造函数，可以创建一个与模板具有相同类型的新数组，并预分配给定数量的元素。请求时，数组初始化为零。

```py
from cpython cimport array
import array

cdef array.array int_array_template = array.array('i', [])
cdef array.array newarray

# create an array with 3 elements with same type as template
newarray = array.clone(int_array_template, 3, zero=False)

```

阵列也可以扩展和调整大小;这避免了重复的内存重新分配，如果元素将被逐个追加或删除。

```py
from cpython cimport array
import array

cdef array.array a = array.array('i', [1, 2, 3])
cdef array.array b = array.array('i', [4, 5, 6])

# extend a with b, resize as needed
array.extend(a, b)
# resize a, leaving just original three elements
array.resize(a, len(a) - len(b))

```

## API 参考

### 数据字段

```py
data.as_voidptr
data.as_chars
data.as_schars
data.as_uchars
data.as_shorts
data.as_ushorts
data.as_ints
data.as_uints
data.as_longs
data.as_ulongs
data.as_longlongs  # requires Python >=3
data.as_ulonglongs  # requires Python >=3
data.as_floats
data.as_doubles
data.as_pyunicodes

```

使用给定类型直接访问基础连续 C 数组;例如，`myarray.data.as_ints`。

### 功能

以下函数可用于阵列模块中的 Cython：

```py
int resize(array self, Py_ssize_t n) except -1

```

快速调整大小/重新分配。不适合重复的小增量;将基础数组的大小调整为所请求的数量。

```py
int resize_smart(array self, Py_ssize_t n) except -1

```

高效率的小增量;使用增长模式，提供摊销的线性时间附加。

```py
cdef inline array clone(array template, Py_ssize_t length, bint zero)

```

给定模板数组，快速创建新数组。类型与`template`相同。如果零是`True`，则新数组将用零初始化。

```py
cdef inline array copy(array self)

```

制作一个数组的副本。

```py
cdef inline int extend_buffer(array self, char* stuff, Py_ssize_t n) except -1

```

有效附加相同类型的新数据（例如相同数组类型）`n`：元素数量（不是字节数！）

```py
cdef inline int extend(array self, array other) except -1

```

使用来自另一个数组的数据扩展数组;类型必须匹配。

```py
cdef inline void zero(array self)

```

将数组的所有元素设置为零。