# Unicode 和传递字符串

> 原文： [`docs.cython.org/en/latest/src/tutorial/strings.html`](http://docs.cython.org/en/latest/src/tutorial/strings.html)

与 Python 3 中的字符串语义类似，Cython 严格区分字节字符串和 unicode 字符串。最重要的是，这意味着默认情况下，字节字符串和 unicode 字符串之间没有自动转换（Python 2 在字符串操作中的作用除外）。所有编码和解码必须通过显式编码/解码步骤。为了在简单的情况下简化 Python 和 C 字符串之间的转换，模块级`c_string_type`和`c_string_encoding`指令可用于隐式插入这些编码/解码步骤。

## Cython 代码中的 Python 字符串类型

Cython 支持四种 Python 字符串类型： [`bytes`](https://docs.python.org/3/library/stdtypes.html#bytes "(in Python v3.7)") ， [`str`](https://docs.python.org/3/library/stdtypes.html#str "(in Python v3.7)") ，`unicode`和`basestring`。 [`bytes`](https://docs.python.org/3/library/stdtypes.html#bytes "(in Python v3.7)") 和`unicode`类型是普通 Python 2.x 中已知的特定类型（在 Python 3 中命名为 [`bytes`](https://docs.python.org/3/library/stdtypes.html#bytes "(in Python v3.7)") 和 [`str`](https://docs.python.org/3/library/stdtypes.html#str "(in Python v3.7)") ）。此外，Cython 还支持 [`bytearray`](https://docs.python.org/3/library/stdtypes.html#bytearray "(in Python v3.7)") 类型，其行为类似于 [`bytes`](https://docs.python.org/3/library/stdtypes.html#bytes "(in Python v3.7)") 类型，除了它是可变的。

[`str`](https://docs.python.org/3/library/stdtypes.html#str "(in Python v3.7)") 类型的特殊之处在于它是 Python 2 中的字节字符串和 Python 3 中的 Unicode 字符串（用于使用语言级别 2 编译的 Cython 代码，即默认值）。意思是，它总是与 Python 运行时自身调用的类型 [`str`](https://docs.python.org/3/library/stdtypes.html#str "(in Python v3.7)") 完全对应。因此，在 Python 2 中， [`bytes`](https://docs.python.org/3/library/stdtypes.html#bytes "(in Python v3.7)") 和 [`str`](https://docs.python.org/3/library/stdtypes.html#str "(in Python v3.7)") 都代表字节串类型，而在 Python 3 中， [`str`](https://docs.python.org/3/library/stdtypes.html#str "(in Python v3.7)") 和`unicode` ]表示 Python Unicode 字符串类型。切换是在 C 编译时进行的，用于运行 Cython 的 Python 版本是不相关的。

在使用语言级别 3 编译 Cython 代码时， [`str`](https://docs.python.org/3/library/stdtypes.html#str "(in Python v3.7)") 类型在 Cython 编译时使用完全符合 Unicode 字符串类型进行标识，即在运行时无法识别 [`bytes`](https://docs.python.org/3/library/stdtypes.html#bytes "(in Python v3.7)") 在 Python 2 中。

请注意， [`str`](https://docs.python.org/3/library/stdtypes.html#str "(in Python v3.7)") 类型与 Python 2 中的`unicode`类型不兼容，即您无法将 Unicode 字符串分配给键入的变量或参数 [`str`](https://docs.python.org/3/library/stdtypes.html#str "(in Python v3.7)") 。该尝试将在运行时导致编译时错误（如果可检测）或 [`TypeError`](https://docs.python.org/3/library/exceptions.html#TypeError "(in Python v3.7)") 异常。因此，在必须与 Python 2 兼容的代码中静态键入字符串变量时应该小心，因为此 Python 版本允许混合字节字符串和数据的 unicode 字符串，用户通常希望代码能够同时使用这两者。仅针对 Python 3 的代码可以安全地将变量和参数键入为 [`bytes`](https://docs.python.org/3/library/stdtypes.html#bytes "(in Python v3.7)") 或`unicode`。

`basestring`类型表示 [`str`](https://docs.python.org/3/library/stdtypes.html#str "(in Python v3.7)") 和`unicode`类型，即 Python 2 和 Python 3 中的所有 Python 文本字符串类型。这可用于键入通常包含 Unicode 文本的文本变量（至少在 Python 3）中，但为了向后兼容的原因，必须另外接受 Python 2 中的 [`str`](https://docs.python.org/3/library/stdtypes.html#str "(in Python v3.7)") 类型。它与 [`bytes`](https://docs.python.org/3/library/stdtypes.html#bytes "(in Python v3.7)") 类型不兼容。它的使用在普通的 Cython 代码中应该是罕见的，因为通用 [`object`](https://docs.python.org/3/library/functions.html#object "(in Python v3.7)") 类型（即无类型代码）通常足够好并且具有支持字符串子类型的分配的额外优点。在 Cython 0.20 中添加了对`basestring`类型的支持。

## 字符串文字

Cython 了解所有 Python 字符串类型前缀：

*   字节串的`b'bytes'`
*   Unicode 字符串的`u'text'`
*   `f'formatted {value}'`用于格式化的 Unicode 字符串文字，由  [**PEP 498** ](https://www.python.org/dev/peps/pep-0498)定义（在 Cython 0.24 中添加）

当使用语言级别 2 和`unicode`对象（即 Python 3 [`str`](https://docs.python.org/3/library/stdtypes.html#str "(in Python v3.7)") ）进行语言级别 3 编译时，未加前缀的字符串文字将成为 [`str`](https://docs.python.org/3/library/stdtypes.html#str "(in Python v3.7)") 对象。

## 关于 C 字符串的一般说明

在许多用例中，C 字符串（即字符串指针）速度慢且繁琐。首先，它们通常需要以某种方式进行手动内存管理，这使得它更有可能在代码中引入错误。

然后，Python 字符串对象缓存它们的长度，因此请求它（例如，验证索引访问的边界或将两个字符串连接成一个）是一种有效的恒定时间操作。相反，调用`strlen()`从 C 字符串获取此信息需要线性时间，这使得对 C 字符串的许多操作相当昂贵。

关于文本处理，Python 内置了对 Unicode 的支持，C 完全缺乏。如果您正在处理 Unicode 文本，那么使用 Python Unicode 字符串对象通常比尝试使用 C 字符串中的编码数据更好。 Cython 使这非常简单有效。

一般来说：除非您知道自己在做什么，否则请尽量避免使用 C 字符串，而应使用 Python 字符串对象。对此明显的例外是从外部 C 代码来回传递它们。此外，C ++字符串也记住它们的长度，因此它们可以在某些情况下提供 Python 字节对象的合适替代，例如，在明确定义的上下文中不需要引用计数时。

## 传递字节串

我们在名为`c_func.pyx`的文件中声明了虚拟 C 函数，我们将在本教程中重复使用它们：

```py
from libc.stdlib cimport malloc
from libc.string cimport strcpy, strlen

cdef char* hello_world = 'hello world'
cdef Py_ssize_t n = strlen(hello_world)

cdef char* c_call_returning_a_c_string():
    cdef char* c_string = <char *> malloc((n + 1) * sizeof(char))
    if not c_string:
        raise MemoryError()
    strcpy(c_string, hello_world)
    return c_string

cdef void get_a_c_string(char** c_string_ptr, Py_ssize_t *length):
    c_string_ptr[0] = <char *> malloc((n + 1) * sizeof(char))
    if not c_string_ptr[0]:
        raise MemoryError()

    strcpy(c_string_ptr[0], hello_world)
    length[0] = n

```

我们制作了相应的`c_func.pxd`以便能够实现这些功能：

```py
cdef char* c_call_returning_a_c_string()
cdef void get_a_c_string(char** c_string, Py_ssize_t *length)

```

在 C 代码和 Python 之间传递字节字符串非常容易。从 C 库接收字节字符串时，只需将其转换为 Python 变量，即可让 Cython 将其转换为 Python 字节字符串：

```py
from c_func cimport c_call_returning_a_c_string

cdef char* c_string = c_call_returning_a_c_string()
cdef bytes py_string = c_string

```

转换为 [`object`](https://docs.python.org/3/library/functions.html#object "(in Python v3.7)") 或 [`bytes`](https://docs.python.org/3/library/stdtypes.html#bytes "(in Python v3.7)") 的类型将执行相同的操作：

```py
py_string = <bytes> c_string

```

这将创建一个 Python 字节字符串对象，该对象包含原始 C 字符串的副本。它可以安全地在 Python 代码中传递，并在最后一次引用超出范围时进行垃圾收集。重要的是要记住，字符串中的空字节充当终止符，如 C 中通常所知。因此，上述内容仅适用于不包含空字节的 C 字符串。

除了不使用空字节之外，对于长字符串，上述也是非常低效的，因为 Cython 必须首先在 C 字符串上调用`strlen()`以通过计算字节直到终止空字节来找出长度。在许多情况下，用户代码已经知道长度，例如因为 C 函数返回了它。在这种情况下，通过切片 C 字符串告诉 Cython 确切的字节数会更有效。这是一个例子：

```py
from libc.stdlib cimport free
from c_func cimport get_a_c_string

def main():
    cdef char* c_string = NULL
    cdef Py_ssize_t length = 0

    # get pointer and length from a C function
    get_a_c_string(&c_string, &length)

    try:
        py_bytes_string = c_string[:length]  # Performs a copy of the data
    finally:
        free(c_string)

```

这里，不需要额外的字节计数，`c_string`中的`length`字节将被复制到 Python 字节对象中，包括任何空字节。请记住，在这种情况下，切片索引被认为是准确的，并且没有进行边界检查，因此不正确的切片索引将导致数据损坏和崩溃。

请注意，Python 字节字符串的创建可能会因异常而失败，例如由于记忆力不足。如果转换后需要`free()`字符串，则应将赋值包装在 try-finally 结构中：

```py
from libc.stdlib cimport free
from c_func cimport c_call_returning_a_c_string

cdef bytes py_string
cdef char* c_string = c_call_returning_a_c_string()
try:
    py_string = c_string
finally:
    free(c_string)

```

要将字节字符串转换回 C `char*`，请使用相反的赋值：

```py
cdef char* other_c_string = py_string  # other_c_string is a 0-terminated string.

```

这是一个非常快速的操作，之后`other_c_string`指向 Python 字符串本身的字节字符串缓冲区。它与 Python 字符串的生命周期有关。当 Python 字符串被垃圾收集时，指针变为无效。因此，只要`char*`正在使用，就必须保持对 Python 字符串的引用。通常，这只会调用一个接收指针作为参数的 C 函数。但是，当 C 函数存储指针供以后使用时，必须特别小心。除了保持对字符串对象的 Python 引用外，不需要手动内存管理。

从 Cython 0.20 开始，支持 [`bytearray`](https://docs.python.org/3/library/stdtypes.html#bytearray "(in Python v3.7)") 类型并以与 [`bytes`](https://docs.python.org/3/library/stdtypes.html#bytes "(in Python v3.7)") 类型相同的方式强制执行。但是，在 C 上下文中使用它时，在将对象缓冲区转换为 C 字符串指针后，必须特别注意不要增大或缩小对象缓冲区。这些修改可以更改内部缓冲区地址，这将使指针无效。

## 接受 Python 代码中的字符串

另一方面，从 Python 代码接收输入，初看起来可能看起来很简单，因为它只处理对象。但是，在不使 API 太窄或太不安全的情况下做到这一点可能并不完全明显。

在 API 仅处理字节字符串（即二进制数据或编码文本）的情况下，最好不要将输入参数键入 [`bytes`](https://docs.python.org/3/library/stdtypes.html#bytes "(in Python v3.7)") ，因为这会将允许的输入限制为准确该类型并排除子类型和其他类型的字节容器，例如 [`bytearray`](https://docs.python.org/3/library/stdtypes.html#bytearray "(in Python v3.7)") 对象或内存视图。

根据数据的处理方式（以及在何处），最好接收一维存储器视图，例如，

```py
def process_byte_data(unsigned char[:] data):
    length = data.shape[0]
    first_byte = data[0]
    slice_view = data[1:-1]
    # ...

```

Cython 的内存视图在 Typed Memoryviews 中有更详细的描述，但上面的例子已经显示了 1 维字节视图的大部分相关功能。它们允许有效地处理数组并接受任何可以将其自身解压缩到字节缓冲区中的内容，而无需中间复制。处理后的内容最终可以在内存视图本身（或其中的一部分）中返回，但通常最好将数据复制回平坦且简单的 [`bytes`](https://docs.python.org/3/library/stdtypes.html#bytes "(in Python v3.7)") 或 [`bytearray`](https://docs.python.org/3/library/stdtypes.html#bytearray "(in Python v3.7)") 对象，特别是当只返回一个小切片时。由于内存视图不会复制数据，因此它们会保持整个原始缓冲区的活动状态。这里的一般想法是通过接受任何类型的字节缓冲区来自由输入，但通过返回一个简单，适应良好的对象来严格控制输出。这可以简单地完成如下：

```py
def process_byte_data(unsigned char[:] data):
    # ... process the data, here, dummy processing.
    cdef bint return_all = (data[0] == 108)

    if return_all:
        return bytes(data)
    else:
        # example for returning a slice
        return bytes(data[5:7])

```

如果字节输入实际上是编码文本，并且进一步处理应该在 Unicode 级别进行，那么正确的做法是直接解码输入。这几乎只是 Python 2.x 中的一个问题，Python 代码期望它可以将带有编码文本的字节串（ [`str`](https://docs.python.org/3/library/stdtypes.html#str "(in Python v3.7)") ）传递到文本 API 中。由于这通常发生在模块 API 的多个位置，因此辅助函数几乎总是可行的，因为它允许以后轻松调整输入规范化过程。

这种输入规范化功能通常类似于以下内容：

```py
# to_unicode.pyx

from cpython.version cimport PY_MAJOR_VERSION

cdef unicode _text(s):
    if type(s) is unicode:
        # Fast path for most common case(s).
        return <unicode>s

    elif PY_MAJOR_VERSION < 3 and isinstance(s, bytes):
        # Only accept byte strings as text input in Python 2.x, not in Py3.
        return (<bytes>s).decode('ascii')

    elif isinstance(s, unicode):
        # We know from the fast path above that 's' can only be a subtype here.
        # An evil cast to <unicode> might still work in some(!) cases,
        # depending on what the further processing does.  To be safe,
        # we can always create a copy instead.
        return unicode(s)

    else:
        raise TypeError("Could not convert to unicode.")

```

然后应该像这样使用：

```py
from to_unicode cimport _text

def api_func(s):
    text_input = _text(s)
    # ...

```

类似地，如果进一步处理发生在字节级别，但是应该接受 Unicode 字符串输入，那么如果您使用内存视图，则以下可能有效：

```py
# define a global name for whatever char type is used in the module
ctypedef unsigned char char_type

cdef char_type[:] _chars(s):
    if isinstance(s, unicode):
        # encode to the specific encoding used inside of the module
        s = (<unicode>s).encode('utf8')
    return s

```

在这种情况下，您可能希望另外确保字节串输入确实使用正确的编码，例如如果需要纯 ASCII 输入数据，可以在循环中运行缓冲区并检查每个字节的最高位。这也应该在输入规范化函数中完成。

## 处理“const”

许多 C 库在其 API 中使用`const`修饰符来声明它们不会修改字符串，或者要求用户不得修改它们返回的字符串，例如：

```py
typedef const char specialChar;
int process_string(const char* s);
const unsigned char* look_up_cached_string(const unsigned char* key);

```

Cython 支持该语言中的`const`修饰符，因此您可以直接声明上述函数，如下所示：

```py
cdef extern from "someheader.h":
    ctypedef const char specialChar
    int process_string(const char* s)
    const unsigned char* look_up_cached_string(const unsigned char* key)

```

## 将字节解码为文本

如果您的代码只处理字符串中的二进制数据，那么最初提供的传递和接收 C 字符串的方法就足够了。但是，当我们处理编码文本时，最好在接收时将 C 字节字符串解码为 Python Unicode 字符串，并在出路时将 Python Unicode 字符串编码为 C 字节字符串。

使用 Python 字节字符串对象，通常只需调用`bytes.decode()`方法将其解码为 Unicode 字符串：

```py
ustring = byte_string.decode('UTF-8')

```

Cython 允许您对 C 字符串执行相同操作，只要它不包含空字节：

```py
from c_func cimport c_call_returning_a_c_string

cdef char* some_c_string = c_call_returning_a_c_string()
ustring = some_c_string.decode('UTF-8')

```

并且，对于已知长度的字符串，更有效：

```py
from c_func cimport get_a_c_string

cdef char* c_string = NULL
cdef Py_ssize_t length = 0

# get pointer and length from a C function
get_a_c_string(&c_string, &length)

ustring = c_string[:length].decode('UTF-8')

```

当字符串包含空字节时，应该使用相同的字符串，例如当它使用像 UCS-4 这样的编码时，每个字符以四个字节编码，其中大多数字符往往是 0。

同样，如果提供切片索引，则不会进行边界检查，因此不正确的索引会导致数据损坏和崩溃。但是，使用负索引是可能的，并将调用`strlen()`以确定字符串长度。显然，这仅适用于没有内部空字节的 0 终止字符串。以 UTF-8 编码的文本或 ISO-8859 编码之一通常是一个很好的候选者。如果有疑问，最好传递“明显”正确的索引，而不是依赖于数据是否符合预期。

通常的做法是在专用函数中包装字符串转换（以及一般的非平凡类型转换），因为每当从 C 接收文本时，都需要以完全相同的方式完成。这可能如下所示：

```py
from libc.stdlib cimport free

cdef unicode tounicode(char* s):
    return s.decode('UTF-8', 'strict')

cdef unicode tounicode_with_length(
        char* s, size_t length):
    return s[:length].decode('UTF-8', 'strict')

cdef unicode tounicode_with_length_and_free(
        char* s, size_t length):
    try:
        return s[:length].decode('UTF-8', 'strict')
    finally:
        free(s)

```

最有可能的是，根据要处理的字符串类型，您会更喜欢代码中较短的函数名称。不同类型的内容通常意味着在接收时处理它们的不同方式。为了使代码更具可读性并预测未来的更改，最好对不同类型的字符串使用单独的转换函数。

## 将文本编码为字节

反过来，将 Python unicode 字符串转换为 C `char*`本身非常有效，假设您实际需要的是内存管理字节字符串：

```py
py_byte_string = py_unicode_string.encode('UTF-8')
cdef char* c_string = py_byte_string

```

如前所述，这将指针指向 Python 字节字符串的字节缓冲区。尝试在不保留对 Python 字节字符串的引用的情况下执行相同操作将失败并出现编译错误：

```py
# this will not compile !
cdef char* c_string = py_unicode_string.encode('UTF-8')

```

在这里，Cython 编译器注意到代码采用指向临时字符串结果的指针，该结果将在赋值后进行垃圾回收。稍后访问无效指针将读取无效内存，并可能导致段错误。因此，Cython 将拒绝编译此代码。

## C ++字符串

包装 C ++库时，字符串通常以`std::string`类的形式出现。与 C 字符串一样，Python 字节字符串自动强制转换为 C ++字符串：

```py
# distutils: language = c++

from libcpp.string cimport string

def get_bytes():
    py_bytes_object = b'hello world'
    cdef string s = py_bytes_object

    s.append('abc')
    py_bytes_object = s
    return py_bytes_object

```

内存管理情况与 C 中的情况不同，因为创建 C ++字符串会生成字符串对象随后拥有的字符串缓冲区的独立副本。因此，可以将临时创建的 Python 对象直接转换为 C ++字符串。使用此方法的常用方法是将 Python unicode 字符串编码为 C ++字符串：

```py
cdef string cpp_string = py_unicode_string.encode('UTF-8')

```

请注意，这涉及一些开销，因为它首先将 Unicode 字符串编码为临时创建的 Python 字节对象，然后将其缓冲区复制到新的 C ++字符串中。

另一方面，Cython 0.17 及更高版本提供了高效的解码支持：

```py
# distutils: language = c++

from libcpp.string cimport string

def get_ustrings():
    cdef string s = string(b'abcdefg')

    ustring1 = s.decode('UTF-8')
    ustring2 = s[2:-2].decode('UTF-8')
    return ustring1, ustring2

```

对于 C ++字符串，解码片将始终考虑字符串的适当长度并应用 Python 切片语义（例如，为越界索引返回空字符串）。

## 自动编码和解码

Cython 0.19 附带两个新指令：`c_string_type`和`c_string_encoding`。它们可用于更改 C / C ++字符串强制执行的 Python 字符串类型。默认情况下，它们仅强制执行字节类型，并且必须明确地进行编码或解码，如上所述。

有两种用例不方便。首先，如果正在处理的所有 C 字符串（或大多数）包含文本，则从 Python unicode 对象自动编码和解码可以减少代码开销。在这种情况下，您可以将模块中的`c_string_type`指令设置为`unicode`，将`c_string_encoding`指定为 C 代码使用的编码，例如：

```py
# cython: c_string_type=unicode, c_string_encoding=utf8

cdef char* c_string = 'abcdefg'

# implicit decoding:
cdef object py_unicode_object = c_string

# explicit conversion to Python bytes:
py_bytes_object = <bytes>c_string

```

第二个用例是当所有正在处理的 C 字符串只包含 ASCII 可编码字符（例如数字）时，您希望代码在 Python 2 中使用本机遗留字符串类型，而不是始终使用 Unicode。在这种情况下，您可以将字符串类型设置为 [`str`](https://docs.python.org/3/library/stdtypes.html#str "(in Python v3.7)") ：

```py
# cython: c_string_type=str, c_string_encoding=ascii

cdef char* c_string = 'abcdefg'

# implicit decoding in Py3, bytes conversion in Py2:
cdef object py_str_object = c_string

# explicit conversion to Python bytes:
py_bytes_object = <bytes>c_string

# explicit conversion to Python unicode:
py_bytes_object = <unicode>c_string

```

另一个方向，即自动编码为 C 字符串，仅支持 ASCII 和“默认编码”，它通常是 Python 3 中的 UTF-8，通常是 Python 2 中的 ASCII。在这种情况下，CPython 通过保持一个来处理内存管理字符串的编码副本与原始 unicode 字符串一起存活。否则，将无法以任何合理的方式限制编码字符串的生命周期，从而使得从其中提取 C 字符串指针的任何尝试都是危险的尝试。以下安全地将 Unicode 字符串转换为 ASCII（将`c_string_encoding`更改为`default`以使用默认编码）：

```py
# cython: c_string_type=unicode, c_string_encoding=ascii

def func():
    ustring = u'abc'
    cdef char* s = ustring
    return s[0]    # returns u'a'

```

（此示例使用函数上下文来安全地控制 Unicode 字符串的生命周期。可以从外部修改全局 Python 变量，这使得依赖其值的生命周期变得很危险。）

## 源代码编码

当字符串文字出现在代码中时，源代码编码很重要。它确定 Cython 将在字节文字的 C 代码中存储的字节序列，以及在解析字节编码的源文件时 Cython 为 unicode 文字构建的 Unicode 代码点。在  [**PEP 263** ](https://www.python.org/dev/peps/pep-0263)之后，Cython 支持源文件编码的显式声明。例如，将以下注释放在`ISO-8859-15`（Latin-9）编码源文件的顶部（进入第一行或第二行）需要在解析器中启用`ISO-8859-15`解码：

```py
# -*- coding: ISO-8859-15 -*-

```

当没有提供明确的编码声明时，源代码被解析为 UTF-8 编码的文本，如  [**PEP 3120** ](https://www.python.org/dev/peps/pep-3120)所指定的。 [UTF-8](https://en.wikipedia.org/wiki/UTF-8) 是一种非常常见的编码，可以表示整个 Unicode 字符集，并且与有效编码的纯 ASCII 编码文本兼容。这使得它成为通常主要由 ASCII 字符组成的源代码文件的非常好的选择。

例如，将以下行放入 UTF-8 编码的源文件中将打印`5`，因为 UTF-8 对双字节序列`'\xc3\xb6'`中的字母`'ö'`进行编码：

```py
print( len(b'abcö') )

```

而以下`ISO-8859-15`编码的源文件将打印`4`，因为编码仅使用此字母的 1 个字节：

```py
# -*- coding: ISO-8859-15 -*-
print( len(b'abcö') )

```

请注意，unicode 文字`u'abcö'`在两种情况下都是正确解码的四字符 Unicode 字符串，而未加前缀的 Python [`str`](https://docs.python.org/3/library/stdtypes.html#str "(in Python v3.7)") 文字`'abcö'`将成为 Python 2 中的字节字符串（因此长度为 4）或者在上面的示例中为 5），以及 Python 3 中的 4 个字符的 Unicode 字符串。如果您不熟悉编码，则在首次阅读时可能看起来不太明显。有关详细信息，请参阅 [CEP 108](https://github.com/cython/cython/wiki/enhancements-stringliterals) 。

根据经验，最好避免使用未加前缀的非 ASCII [`str`](https://docs.python.org/3/library/stdtypes.html#str "(in Python v3.7)") 文字，并对所有文本使用 unicode 字符串文字。 Cython 还支持`__future__` import `unicode_literals`，它指示解析器将源文件中所有未加前缀的 [`str`](https://docs.python.org/3/library/stdtypes.html#str "(in Python v3.7)") 文字读取为 unicode 字符串文字，就像 Python 3 一样。

## 单字节和字符

Python C-API 使用普通的 C `char`类型来表示字节值，但它有两个特殊的整数类型用于 Unicode 代码点值，即单个 Unicode 字符： [`Py_UNICODE`](https://docs.python.org/3/c-api/unicode.html#c.Py_UNICODE "(in Python v3.7)") 和 [`Py_UCS4`](https://docs.python.org/3/c-api/unicode.html#c.Py_UCS4 "(in Python v3.7)") 。 Cython 支持第一个本地，支持 [`Py_UCS4`](https://docs.python.org/3/c-api/unicode.html#c.Py_UCS4 "(in Python v3.7)") 是 Cython 0.15 的新功能。 [`Py_UNICODE`](https://docs.python.org/3/c-api/unicode.html#c.Py_UNICODE "(in Python v3.7)") 定义为无符号 2 字节或 4 字节整数，或定义为`wchar_t`，具体取决于平台。确切类型是 CPython 解释器构建中的编译时选项，扩展模块在 C 编译时继承此定义。 [`Py_UCS4`](https://docs.python.org/3/c-api/unicode.html#c.Py_UCS4 "(in Python v3.7)") 的优点在于，无论平台如何，它都可以保证足够大，以适应任何 Unicode 代码点值。它被定义为 32 位无符号整数或长整数。

在 Cython 中，`char`类型在强制转换为 Python 对象时与 [`Py_UNICODE`](https://docs.python.org/3/c-api/unicode.html#c.Py_UNICODE "(in Python v3.7)") 和 [`Py_UCS4`](https://docs.python.org/3/c-api/unicode.html#c.Py_UCS4 "(in Python v3.7)") 类型的行为不同。类似于 Python 3 中字节类型的行为，`char`类型默认强制为 Python 整数值，因此以下打印 65 而不是`A`：

```py
# -*- coding: ASCII -*-

cdef char char_val = 'A'
assert char_val == 65   # ASCII encoded byte value of 'A'
print( char_val )

```

如果你想要一个 Python 字节字符串，你必须明确地请求它，以下将打印`A`（或 Python 3 中的`b'A'`）：

```py
print( <bytes>char_val )

```

显式强制适用于任何 C 整数类型。超出`char`或`unsigned char`范围的值将在运行时升高 [`OverflowError`](https://docs.python.org/3/library/exceptions.html#OverflowError "(in Python v3.7)") 。在分配类型变量时，强制也会自动发生，例如：

```py
cdef bytes py_byte_string
py_byte_string = char_val

```

另一方面， [`Py_UNICODE`](https://docs.python.org/3/c-api/unicode.html#c.Py_UNICODE "(in Python v3.7)") 和 [`Py_UCS4`](https://docs.python.org/3/c-api/unicode.html#c.Py_UCS4 "(in Python v3.7)") 类型很少在 Python unicode 字符串的上下文之外使用，因此它们的默认行为是强制使用 Python unicode 宾语。因此，以下将打印字符`A`，与 [`Py_UNICODE`](https://docs.python.org/3/c-api/unicode.html#c.Py_UNICODE "(in Python v3.7)") 类型的代码相同：

```py
cdef Py_UCS4 uchar_val = u'A'
assert uchar_val == 65 # character point value of u'A'
print( uchar_val )

```

同样，显式转换将允许用户覆盖此行为。以下将打印 65：

```py
cdef Py_UCS4 uchar_val = u'A'
print( <long>uchar_val )

```

请注意，转换为 C `long`（或`unsigned long`）将正常工作，因为 Unicode 字符可以具有的最大代码点值是 1114111（`0x10FFFF`）。在 32 位或更高的平台上，`int`同样出色。

## 窄 Unicode 构建

在版本 3.3 之前的 CPython 的窄版本构建中，即在`sys.maxunicode`为 65535 的情况下构建（例如所有 Windows 构建，而不是宽版本中的 1114111），仍然可以使用不适合的字符串代码点。 16 位宽 [`Py_UNICODE`](https://docs.python.org/3/c-api/unicode.html#c.Py_UNICODE "(in Python v3.7)") 类型。例如，这样的 CPython 构建将接受 unicode 文字`u'\U00012345'`。但是，在这种情况下，底层系统级编码泄漏到 Python 空间中，因此该文字的长度变为 2 而不是 1.这也显示在迭代它或索引到它时。在这个例子中，可见的子串是`u'\uD808'`和`u'\uDF45'`。它们形成了代表上述特征的所谓代理对。

有关该主题的更多信息，请阅读 [Wikipedia 关于 UTF-16 编码](https://en.wikipedia.org/wiki/UTF-16/UCS-2)的文章。

相同的属性适用于为窄 CPython 运行时环境编译的 Cython 代码。在大多数情况下，例如在搜索子字符串时，可以忽略此差异，因为文本和子字符串都将包含代理项。因此，大多数 Unicode 处理代码也可以在窄版本上正常工作。编码，解码和打印将按预期工作，因此上述文字在窄和宽 Unicode 平台上变成完全相同的字节序列。

但是，程序员应该知道单个 [`Py_UNICODE`](https://docs.python.org/3/c-api/unicode.html#c.Py_UNICODE "(in Python v3.7)") 值（或 CPython 中的单个'字符'unicode 字符串）可能不足以在窄平台上表示完整的 Unicode 字符。例如，如果在 unicode 字符串中对`u'\uD808'`和`u'\uDF45'`的独立搜索成功，则这并不一定意味着字符`u'\U00012345`是该字符串的一部分。很可能是字符串中的两个不同的字符碰巧与所讨论的字符的代理对共享代码单元。寻找子串正常工作是因为代理对中的两个代码单元使用不同的值范围，因此该对在代码点序列中始终是可识别的。

从版本 0.15 开始，Cython 扩展了对代理对的支持，因此即使在狭窄的平台上，您也可以安全地使用`in`测试从完整的 [`Py_UCS4`](https://docs.python.org/3/c-api/unicode.html#c.Py_UCS4 "(in Python v3.7)") 范围中搜索字符值：

```py
cdef Py_UCS4 uchar = 0x12345
print( uchar in some_unicode_string )

```

类似地，它可以在窄和宽 Unicode 平台上将具有高 Unicode 代码点值的一个字符串强制转换为 Py_UCS4 值：

```py
cdef Py_UCS4 uchar = u'\U00012345'
assert uchar == 0x12345

```

在 CPython 3.3 及更高版本中， [`Py_UNICODE`](https://docs.python.org/3/c-api/unicode.html#c.Py_UNICODE "(in Python v3.7)") 类型是系统特定`wchar_t`类型的别名，不再与 Unicode 字符串的内部表示相关联。相反，任何 Unicode 字符都可以在所有平台上表示，而无需使用代理项对。这意味着无论 [`Py_UNICODE`](https://docs.python.org/3/c-api/unicode.html#c.Py_UNICODE "(in Python v3.7)") 的大小如何，该版本都不再存在窄版本。有关详细信息，请参阅  [**PEP 393** ](https://www.python.org/dev/peps/pep-0393)。

只要将类型推断应用于无类型变量或者在源代码中明确使用可移植 [`Py_UCS4`](https://docs.python.org/3/c-api/unicode.html#c.Py_UCS4 "(in Python v3.7)") 类型，Cython 0.16 及更高版本内部处理此更改并对单个字符值执行正确的操作平台特异性 [`Py_UNICODE`](https://docs.python.org/3/c-api/unicode.html#c.Py_UNICODE "(in Python v3.7)") 型。 Cython 应用于 Python unicode 类型的优化将在 C 编译时自动适应  [**PEP 393** ](https://www.python.org/dev/peps/pep-0393)，像往常一样。

## 迭代

只要循环变量被适当地键入，Cython 0.13 就支持对`char*`，字节和 unicode 字符串的有效迭代。所以以下将生成预期的 C 代码：

```py
cdef char* c_string = "Hello to A C-string's world"

cdef char c
for c in c_string[:11]:
    if c == 'A':
        print("Found the letter A")

```

这同样适用于字节对象：

```py
cdef bytes bytes_string = b"hello to A bytes' world"

cdef char c
for c in bytes_string:
    if c == 'A':
        print("Found the letter A")

```

对于 unicode 对象，Cython 会自动将循环变量的类型推断为 [`Py_UCS4`](https://docs.python.org/3/c-api/unicode.html#c.Py_UCS4 "(in Python v3.7)") ：

```py
cdef unicode ustring = u'Hello world'

# NOTE: no typing required for 'uchar' !
for uchar in ustring:
    if uchar == u'A':
        print("Found the letter A")

```

自动类型推断通常会在这里产生更高效的代码。但是，请注意，某些 unicode 操作仍然需要将值作为 Python 对象，因此 Cython 最终可能会为循环内部的循环变量值生成冗余转换代码。如果这导致特定代码段的性能下降，您可以显式地将循环变量键入为 Python 对象，或者将其值赋值给循环内部的 Python 类型变量，以在运行 Python 之前强制执行一次性强制对它的操作。

`in`测试也有优化，因此以下代码将在纯 C 代码中运行（实际上使用 switch 语句）：

```py
cpdef void is_in(Py_UCS4 uchar_val):
    if uchar_val in u'abcABCxY':
        print("The character is in the string.")
    else:
        print("The character is not in the string")

```

结合上面的循环优化，这可以产生非常有效的字符切换代码，例如，在 unicode 解析器中。

## Windows 和宽字符 API

Windows 系统 API 本身以零终止 UTF-16 编码`wchar_t*`字符串的形式支持 Unicode，因此称为“宽字符串”。

默认情况下，Windows 版本的 CPython 定义 [`Py_UNICODE`](https://docs.python.org/3/c-api/unicode.html#c.Py_UNICODE "(in Python v3.7)") 作为`wchar_t`的同义词。这使得内部`unicode`表示与 UTF-16 兼容，并允许有效的零拷贝转换。这也意味着 Windows 构建总是窄 Unicode 构建与所有警告。

为了帮助与 Windows API 互操作，Cython 0.19 支持宽字符串（以`Py_UNICODE*`的形式）并隐式地将它们转换为`unicode`字符串对象。这些转换的行为与`char*`和 [`bytes`](https://docs.python.org/3/library/stdtypes.html#bytes "(in Python v3.7)") 相同，如传递字节串中所述。

除自动转换外，C 语境中出现的 unicode 文字成为 C 级宽字符串文字， [`len()`](https://docs.python.org/3/library/functions.html#len "(in Python v3.7)") 内置函数专门用于计算零终止`Py_UNICODE*`字符串或数组的长度。

以下是如何在 Windows 上调用 Unicode API 的示例：

```py
cdef extern from "Windows.h":

    ctypedef Py_UNICODE WCHAR
    ctypedef const WCHAR* LPCWSTR
    ctypedef void* HWND

    int MessageBoxW(HWND hWnd, LPCWSTR lpText, LPCWSTR lpCaption, int uType)

title = u"Windows Interop Demo - Python %d.%d.%d" % sys.version_info[:3]
MessageBoxW(NULL, u"Hello Cython \u263a", title, 0)

```

警告

强烈建议不要在 Windows 之外使用`Py_UNICODE*`字符串。 [`Py_UNICODE`](https://docs.python.org/3/c-api/unicode.html#c.Py_UNICODE "(in Python v3.7)") 本身在不同平台和 Python 版本之间不可移植。

CPython 3.3 已经转移到 unicode 字符串（  [**PEP 393** ](https://www.python.org/dev/peps/pep-0393)）的灵活内部表示，使得所有 [`Py_UNICODE`](https://docs.python.org/3/c-api/unicode.html#c.Py_UNICODE "(in Python v3.7)") 相关 API 被弃用，效率低下。

CPython 3.3 更改的一个结果是`unicode`字符串的 [`len()`](https://docs.python.org/3/library/functions.html#len "(in Python v3.7)") 总是在 *代码点*（“字符”）中测量，而 Windows API 期望 UTF-16 的数量 *代码单元*（每个代理单独计算）。要始终获得代码单元的数量，请直接调用 [`PyUnicode_GetSize()`](https://docs.python.org/3/c-api/unicode.html#c.PyUnicode_GetSize "(in Python v3.7)") 。