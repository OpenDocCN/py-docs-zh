# 与外部 C 代码连接

> 原文： [`docs.cython.org/en/latest/src/userguide/external_C_code.html`](http://docs.cython.org/en/latest/src/userguide/external_C_code.html)

Cython 的主要用途之一是包装现有的 C 代码库。这是通过使用外部声明来声明要使用的库中的 C 函数和变量来实现的。

您还可以使用公共声明使 Cython 模块中定义的 C 函数和变量可用于外部 C 代码。预计对此的需求会减少，但您可能希望这样做，例如，如果您[将 Python](http://www.freenet.org.nz/python/embeddingpyrex/) 作为脚本语言嵌入到另一个应用程序中。就像 Cython 模块可以用作允许 Python 代码调用 C 代码的桥梁一样，它也可以用于允许 C 代码调用 Python 代码。

## 外部声明

默认情况下，在模块级别声明的 C 函数和变量对于模块是本地的（即它们具有 C 静态存储类）。它们也可以声明为 extern 以指定它们在别处定义，例如：

```py
cdef extern int spam_counter

cdef extern void order_spam(int tons)

```

### 引用 C 头文件

当您在上面的示例中单独使用 extern 定义时，Cython 在生成的 C 文件中包含它的声明。如果声明与其他 C 代码将看到的声明不完全匹配，则可能会导致问题。例如，如果要包装现有的 C 库，则生成的 C 代码的编译与库的其余部分完全相同，这一点很重要。

为了实现这一点，你可以告诉 Cython 声明是在 C 头文件中找到的，如下所示：

```py
cdef extern from "spam.h":

    int spam_counter

    void order_spam(int tons)

```

`cdef extern` from 子句做了三件事：

1.  它指示 Cython 在生成的 C 代码中为命名头文件放置`#include`语句。
2.  它可以防止 Cython 为关联块中的声明生成任何 C 代码。
3.  它将块中的所有声明视为以`cdef extern`开头。

重要的是要了解 Cython 本身不会读取 C 头文件，因此您仍需要提供您使用的任何声明的 Cython 版本。但是，Cython 声明并不总是必须完全匹配 C 声明，在某些情况下它们不应该或不能。特别是：

1.  省略对 C 声明的任何特定于平台的扩展，例如`__declspec()`。

2.  如果头文件声明了一个大结构而你只想使用几个成员，你只需要声明你感兴趣的成员。剩下的就不会造成任何伤害，因为 C 编译器会使用完整的头文件中的定义。

    在某些情况下，您可能不需要任何 struct 的成员，在这种情况下，您可以将 pass 放在 struct 声明的主体中，例如：

    ```py
    cdef extern from "foo.h":
        struct spam:
            pass

    ```

    注意

    你只能在`cdef extern from`块内执行此操作;其他地方的 struct 声明必须是非空的。

3.  如果头文件使用`typedef`等名称来引用数字类型的平台相关风格，则需要相应的 `ctypedef` 语句，但不需要匹配完全键入，只使用正确的一般类型（int，float 等）。例如，：

    ```py
    ctypedef int word

    ```

    无论`word`的实际大小是什么（如果头文件正确定义），它都可以正常工作。如果有的话，Python 类型的转换也将用于此类型。

4.  如果头文件使用宏来定义常量，则将它们转换为普通的外部变量声明。如果它们包含正常的`int`值，您也可以将它们声明为 `enum` 。请注意，Cython 认为 `enum` 等同于`int`，因此不要对非 int 值执行此操作。

5.  如果头文件使用宏定义函数，则将其声明为普通函数，并使用适当的参数和结果类型。

6.  出于陈旧的原因，C 使用关键字`void`来声明不带参数的函数。在 Cython 中，就像在 Python 中一样，只需声明`foo()`等函数。

还有一些技巧和窍门：

*   如果你想要包含一个 C 头，因为它是另一个头所需要的，但又不想使用它的任何声明，请在 extern-from 块中放入 pass：

    ```py
    cdef extern from "spam.h":
        pass

    ```

*   如果要包含系统标题，请在引号内放置尖括号：

    ```py
    cdef extern from "&lt;sysheader.h&gt;":
        ...

    ```

*   如果要包含一些外部声明，但又不想指定头文件（因为它已包含在您已包含的其他标头中），则可以使用`*`代替头文件名：

    ```py
    cdef extern from *:
        ...

    ```

*   如果`cdef extern from "inc.h"`块不为空且仅包含函数或变量声明（并且没有任何类型的声明），Cython 将在 Cython 生成的所有声明之后放置`#include "inc.h"`语句。这意味着包含的文件可以访问由 Cython 声明的变量，函数，结构......

### 用 C 实现函数

当您想从 Cython 模块调用 C 代码时，通常该代码将位于您链接扩展的某个外部库中。但是，您也可以直接编译 C（或 C ++）代码作为 Cython 模块的一部分。在`.pyx`文件中，您可以输入以下内容：

```py
cdef extern from "spam.c":
    void order_spam(int tons)

```

Cython 将假设函数`order_spam()`在文件`spam.c`中定义。如果您还想从另一个模块中导入此函数，则必须在`.pxd`文件中声明（而不是 extern！）：

```py
cdef void order_spam(int tons)

```

为此，`spam.c`中`order_spam()`的签名必须与 Cython 使用的签名匹配，特别是该函数必须是静态的：

```py
static void order_spam(int tons)
{
    printf("Ordered %i tons of spam!\n", tons);
}

```

### struct，union 和 enum 声明的样式

结构，联合和枚举有两种主要方式可以在 C 头文件中声明：使用标记名称或使用 typedef。基于这些的各种组合还存在一些变化。

使 Cython 声明与头文件中使用的样式匹配非常重要，这样 Cython 就可以对它生成的代码中的类型发出正确的类型引用。为了实现这一点，Cython 提供了两种不同的语法来声明 struct，union 或 enum 类型。上面介绍的样式对应于标记名称的使用。要获得其他样式，请使用 `ctypedef` 作为声明的前缀，如下图所示。

下表显示了可以在头文件中找到的各种可能的样式，以及应该从块中放入`cdef extern`的相应 Cython 声明。结构声明用作示例;这同样适用于 union 和 enum 声明。

<colgroup><col width="18%"> <col width="32%"> <col width="50%"></colgroup> 
| C 代码 | 相应的 Cython 代码的可能性 | 评论 |
| --- | --- | --- |
| 

```py
struct Foo {
  ...
};

```

 | 

```py
cdef struct Foo:
  ...

```

 | Cython 将在生成的 C 代码中引用 as `struct Foo`。 |
| 

```py
typedef struct {
  ...
} Foo;

```

 | 

```py
ctypedef struct Foo:
  ...

```

 | Cython 将生成的 C 代码中的类型简称为`Foo`。 |
| 

```py
typedef struct foo {
  ...
} Foo;

```

 | 

```py
cdef struct foo:
  ...
ctypedef foo Foo #optional

```

要么：

```py
ctypedef struct Foo:
  ...

```

 | 如果 C 标头同时使用带有 *不同* 名称的标签和 typedef，则可以在 Cython 中使用任一形式的声明（尽管如果需要转发引用类型，则必须使用第一个表单）。 |
| 

```py
typedef struct Foo {
  ...
} Foo;

```

 | 

```py
cdef struct Foo:
  ...

```

 | 如果标题使用标签和 typedef 的 *相同的* 名称，则无法为其包含 `ctypedef` - 但是，这不是必需的。 |

另请参阅 外部扩展类型 的使用。请注意，在下面的所有情况中，您将 Cython 代码中的类型简称为`Foo`，而不是`struct Foo`。

### 访问 Python / C API 例程

`cdef extern from`语句的一个特殊用途是获取对 Python / C API 中的例程的访问。例如，：

```py
cdef extern from "Python.h":

    object PyString_FromStringAndSize(char *s, Py_ssize_t len)

```

将允许您创建包含空字节的 Python 字符串。

### 特殊类型

Cython 预定义名称`Py_ssize_t`以用于 Python / C API 例程。要使扩展与 64 位系统兼容，应始终在 Python / C API 例程文档中指定的类型中使用此类型。

### Windows 调用约定

`__stdcall`和`__cdecl`调用约定说明符可以在 Cython 中使用，其语法与 Windows 上的 C 编译器使用的语法相同，例如：

```py
cdef extern int __stdcall FrobnicateWindow(long handle)

cdef void (__stdcall *callback)(void *)

```

如果使用`__stdcall`，则仅认为该功能与相同签名的其他`__stdcall`功能兼容。

### 解决命名冲突 - C 名称规范

每个 Cython 模块都有一个用于 Python 和 C 名称的模块级命名空间。如果要包装一些外部 C 函数并为 Python 用户提供相同名称的 Python 函数，这可能会很不方便。

Cython 提供了几种解决此问题的方法。最好的方法，特别是如果要包装许多 C 函数，是将外部 C 函数声明放入`.pxd`文件，从而使用 中描述的设备在 Cython 模块之间共享声明中使用不同的命名空间 。将它们写入`.pxd`文件允许它们跨模块重用，避免以正常的 Python 方式命名冲突，甚至可以轻松地在 cimport 上重命名它们。例如，如果您的`decl.pxd`文件声明了 C 函数`eject_tomato`：

```py
cdef extern from "myheader.h":
    void eject_tomato(float speed)

```

然后你可以将它导入并将其包装在`.pyx`文件中，如下所示：

```py
from decl cimport eject_tomato as c_eject_tomato

def eject_tomato(speed):
    c_eject_tomato(speed)

```

或者只是简单地输入`.pxd`文件并将其用作前缀：

```py
cimport decl

def eject_tomato(speed):
    decl.eject_tomato(speed)

```

请注意，这没有运行时查找开销，就像在 Python 中一样。 Cython 在编译时解析`.pxd`文件中的名称。

对于导入时命名空间或重命名不够的特殊情况，例如当 C 中的名称与 Python 关键字冲突时，您可以使用 C 名称规范在声明时为 C 函数提供不同的 Cython 和 C 名称。例如，假设您要包装一个名为`yield()`的外部 C 函数。如果您将其声明为：

```py
cdef extern from "myheader.h":
    void c_yield "yield" (float speed)

```

那么它的 Cython 可见名称将是`c_yield`，而它在 C 中的名称将是`yield`。然后你可以用它包装它：

```py
def call_yield(speed):
    c_yield(speed)

```

对于函数，可以为变量，结构，联合，枚举，结构和联合成员以及枚举值指定 C 名称。例如：

```py
cdef extern int one "eins", two "zwei"
cdef extern float three "drei"

cdef struct spam "SPAM":
    int i "eye"

cdef enum surprise "inquisition":
    first "alpha"
    second "beta" = 3

```

请注意，Cython 不会对您提供的字符串进行任何验证或名称修改。它会将裸文本注入未经修改的 C 代码中，因此您完全可以使用此功能。如果你想声明一个名称`xyz`并让 Cython 将文本“让 C 编译器在这里失败”注入它的 C 文件中，你可以使用 C 名声明来完成。考虑这是一个高级功能，仅适用于其他一切都失败的罕见情况。

### 包括逐字 C 代码

对于高级用例，Cython 允许您直接将 C 代码写为`cdef extern from`块的“docstring”：

```py
cdef extern from *:
    """
 /* This is C code which will be put
 * in the .c file output by Cython */
 static long square(long x) {return x * x;}
 #define assign(x, y) ((x) = (y))
 """
    long square(long x)
    void assign(long& x, long y)

```

以上内容基本上等同于将 C 代码放在文件`header.h`中并进行写入

```py
cdef extern from "header.h":
    long square(long x)
    void assign(long& x, long y)

```

也可以组合头文件和逐字 C 代码：

```py
cdef extern from "badheader.h":
    """
 /* This macro breaks stuff */
 #undef int
 """
    # Stuff from badheader.h

```

在这种情况下，C 代码`#undef int`放在 Cython 生成的 C 代码中的`#include "badheader.h"`之后。

请注意，该字符串的解析方式与 Python 中的任何其他 docstring 一样。如果要将字符转义传递到 C 代码文件，请使用原始文档字符串，即`r""" ... """`。

## 使用 C

Cython 提供了两种方法，用于从 Cython 模块生成 C 语句，供外部 C 代码 - 公共声明和 C API 声明使用。

Note

你不需要使用其中任何一个来从一个 Cython 模块声明另一个 Cython 模块 - 你应该使用 `cimport` 语句。在 Cython 模块之间共享声明。

### 公开声明

您可以通过使用 public 关键字声明它们，使 Cython 模块中定义的 C 类型，变量和函数可以被 C 代码访问，C 代码与 Cython 生成的 C 文件链接在一起：

```py
cdef public struct Bunny: # public type declaration
    int vorpalness

cdef public int spam # public variable declaration

cdef public void grail(Bunny *) # public function declaration

```

如果 Cython 模块中有任何公共声明，则会生成一个名为`modulename.h`文件的头文件，其中包含等效的 C 声明，以包含在其他 C 代码中。

一个典型的用例是从多个 C 源构建一个扩展模块，其中一个是 Cython 生成的（即`setup.py`中有`Extension("grail", sources=["grail.pyx", "grail_helper.c"])`的东西。在这种情况下，文件`grail_helper.c`只需要添加`#include "grail.h"` ]为了访问公共 Cython 变量。

更高级的用例是使用 Cython 在 Python 中嵌入 Python。在这种情况下，请确保调用 Py_Initialize（）和 Py_Finalize（）。例如，在包含`grail.h`的以下代码段中：

```py
#include <Python.h>
#include "grail.h"

int main() {
    Py_Initialize();
    initgrail();  /* Python 2.x only ! */
    Bunny b;
    grail(b);
    Py_Finalize();
}

```

然后，可以在单个程序（或库）中将此 C 代码与 Cython 生成的 C 代码一起构建。

在 Python 3.x 中，应该避免直接调用模块 init 函数。相反，使用 [inittab 机制](https://docs.python.org/3/c-api/import.html#c._inittab)将 Cython 模块链接到单个共享库或程序中。

```py
err = PyImport_AppendInittab("grail", PyInit_grail);
Py_Initialize();
grail_module = PyImport_ImportModule("grail");

```

如果 Cython 模块位于包中，则`.h`文件的名称由模块的完整点名称组成，例如，名为`foo.spam`的模块将有一个名为`foo.spam.h`的头文件。

Note

在某些操作系统（如 Linux）上，也可以先按常规方式构建 Cython 扩展，然后像动态库一样链接到生成的`.so`文件。请注意，这不是便携式的，所以应该避免。

### C API 声明

为 C 代码提供声明的另一种方法是使用 `api` 关键字声明它们。您可以将此关键字与 C 函数和扩展类型一起使用。生成一个名为`modulename_api.h`的头文件，其中包含函数和扩展类型的声明，以及一个名为`import_modulename()`的函数。

想要使用这些函数或扩展类型的 C 代码需要包含标题并调用`import_modulename()`函数。然后可以调用其他函数，并像往常一样使用扩展类型。

如果想要使用这些函数的 C 代码是多个共享库或可执行文件的一部分，则需要在使用这些函数的每个共享库中调用`import_modulename()`函数。如果在调用其中一个 api 调用时遇到分段错误（linux 上的 SIGSEGV）崩溃，这可能表明包含生成分段错误的 api 调用的共享库之前没有调用`import_modulename()`函数崩溃的 api 电话。

当您包含`modulename_api.h`时，Cython 模块中的任何公共 C 类型或扩展类型声明也可用。

```py
# delorean.pyx

cdef public struct Vehicle:
    int speed
    float power

cdef api void activate(Vehicle *v):
    if v.speed >= 88 and v.power >= 1.21:
        print("Time travel achieved")

```

```py
# marty.c
#include "delorean_api.h"

Vehicle car;

int main(int argc, char *argv[]) {
	Py_Initialize();
	import_delorean();
	car.speed = atoi(argv[1]);
	car.power = atof(argv[2]);
	activate(&car);
	Py_Finalize();
}

```

Note

在 Cython 模块中定义的任何类型（用作参数或返回类型的导出函数）都需要声明为 public，否则它们将不会包含在生成的头文件中，并且当您尝试编译时会出现错误使用标头的 C 文件。

使用 `api` 方法不需要使用声明的 C 代码以任何方式与扩展模块链接，因为 Python 导入机制用于动态建立连接。但是，只能以这种方式访问​​函数，而不是变量。另请注意，要正确设置模块导入机制，用户必须调用 Py_Initialize（）和 Py_Finalize（）;如果您在`import_modulename()`调用中遇到分段错误，则很可能没有完成。

您可以在同一功能上同时使用 `public` 和 `api` ，例如：

```py
cdef public api void belt_and_braces():
    ...

```

但是，请注意，您应该在给定的 C 文件中包含`modulename.h`或`modulename_api.h`，而不是两者，否则您可能会遇到冲突的双重定义。

如果 Cython 模块位于包中，则：

*   头文件的名称包含模块的完整虚线名称。
*   导入函数的名称包含全名，其中圆点由双下划线替换。

例如。名为`foo.spam`的模块将有一个名为`foo.spam_api.h`的 API 头文件和一个名为`import_foo__spam()`的导入函数。

### 多个公共和 API 声明

您可以将 `public` 和/或 `api` 的整个项目一次性地声明为 `cdef` 块，例，：

```py
cdef public api:
    void order_spam(int tons)
    char *get_lunch(float tomato_size)

```

这在`.pxd`文件中是有用的（参见 在 Cython 模块之间共享声明 ），以通过所有三种方法使模块的公共接口可用。

### 获取和释放 GIL

Cython 提供了获取和发布[全球解释器锁（GIL）](https://docs.python.org/dev/glossary.html#term-global-interpreter-lock)的工具。当从多线程代码调用可能阻塞的（外部 C）代码或者想要从（本机）C 线程回调中使用 Python 时，这可能很有用。释放 GIL 显然应仅针对线程安全代码或针对竞争条件和并发问题使用其他保护手段的代码。

请注意，获取 GIL 是阻塞线程同步操作，因此可能成本高昂。对于次要计算，可能不值得发布 GIL。通常，并行代码中的 I / O 操作和实质计算将从中受益。

#### 释放 GIL

您可以使用`with nogil`语句围绕一段代码释放 GIL：

```py
with nogil:
    <code to be executed with the GIL released>

```

with 语句主体中的代码不得以任何方式引发异常或操纵 Python 对象，并且在不首先重新获取 GIL 的情况下不得调用任何操作 Python 对象的内容。例如，Cython 在编译时验证这些操作，但不能查看外部 C 函数。必须正确声明它们需要或不需要 GIL（见下文）才能使 Cython 的检查生效。

#### 获取 GIL

在没有 GIL 的情况下执行的 C 代码回调的 C 函数需要在操作 Python 对象之前获取 GIL。这可以通过在函数头中指定`with gil`来完成：

```py
cdef void my_callback(void *data) with gil:
    ...

```

如果可以从另一个非 Python 线程调用回调，则必须首先通过调用 [PyEval_InitThreads（）](https://docs.python.org/dev/c-api/init.html#c.PyEval_InitThreads)来初始化 GIL。如果您已经在模块中使用 cython.parallel，那么这已经得到了解决。

GIL 也可以通过`with gil`声明获得：

```py
with gil:
    <execute this block with the GIL acquired>

```

#### 条件获取/释放 GIL

有时使用条件来决定是否使用 GIL 运行某段代码。这个代码无论如何都会运行，不同之处在于 GIL 是否会被保留或释放。条件必须是常量（在编译时）。

这对于分析，调试，性能测试和融合类型非常有用（参见 条件 GIL 获取/释放 ）：

```py
DEF FREE_GIL = True

with nogil(FREE_GIL):
    <code to be executed with the GIL released>

    with gil(False):
       <GIL is still released>

```

### 在没有 GIL 的情况下将函数声明为可调用

您可以在 C 函数头或函数类型中指定 `nogil` ，以声明在没有 GIL 的情况下调用是安全的：

```py
cdef void my_gil_free_func(int spam) nogil:
    ...

```

在 Cython 中实现这样的函数时，它不能有任何 Python 参数或 Python 对象返回类型。此外，涉及 Python 对象（包括调用 Python 函数）的任何操作必须首先明确获取 GIL，例如通过使用`with gil`块或调用已定义的函数`with gil`。这些限制由 Cython 检查，如果在`nogil`代码部分中发现任何 Python 交互，您将收到编译错误。

Note

`nogil`函数注释声明在没有 GIL 的情况下调用函数是安全的。完全允许在持有 GIL 的同时执行它。如果调用者持有 GIL，则该函数本身不释放 GIL。

声明函数`with gil`（即在输入时获取 GIL）也隐式地使其签名 `nogil` 。