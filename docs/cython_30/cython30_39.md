# 调试你的 Cython 程序

> 原文： [`docs.cython.org/en/latest/src/userguide/debugging.html`](http://docs.cython.org/en/latest/src/userguide/debugging.html)

Cython 附带了 GNU Debugger 的扩展，可以帮助用户调试 Cython 代码。要使用此功能，您需要安装 gdb 7.2 或更高版本，使用 Python 支持（链接到 Python 2.6 或更高版本）构建。调试器支持 2.6 及更高版本的调试对象。对于 Python 3，代码应该使用 Python 3 构建，调试器应该使用 Python 2 运行（或者至少它应该能够找到 Python 2 Cython 安装）。请注意，在最新版本的 Ubuntu 中，安装`apt-get`的`gdb`配置了 Python 3.在这样的系统上，可以通过下载`gdb`源然后运行来获得`gdb`的正确配置。 ：

```py
./configure --with-python=python2
make
sudo make install

```

调试器将需要 Cython 编译器可以导出的调试信息。这可以通过将`gdb_debug=True`传递给`cythonize()`在设置脚本中实现：

```py
from distutils.core import setup
from distutils.extension import Extension

extensions = [Extension('source', ['source.pyx'])]

setup(..., ext_modules=cythonize(extensions, gdb_debug=True))

```

对于开发，将`--inplace`标志传递给`setup.py`脚本通常很有帮助，这使得 distutils“就地”构建项目，即不在单独的&lt;cite&gt;构建&lt;/cite&gt;目录中。

直接从命令行调用 Cython 时，可以使用`--gdb`标志写入调试信息：

```py
cython --gdb myfile.pyx

```

## 运行调试器

要运行 Cython 调试器并让它导入 Cython 导出的调试信息，请在构建目录中运行`cygdb`：

```py
$ python setup.py build_ext --inplace
$ cygdb
GNU gdb (GDB) 7.2
...
(gdb)

```

使用 Cython 调试器时，最好使用使用调试符号编译的解释器构建和运行代码（即使用`--with-pydebug`配置或使用`-g` CFLAG 编译）。如果您的包由管理器管理器安装和管理，则可能需要单独安装调试支持。如果使用 NumPy，那么你还需要安装 numpy 调试，否则你会看到多播的[导入错误。例如。对于 ubuntu：](https://bugzilla.redhat.com/show_bug.cgi?id=1030830)

```py
$ sudo apt-get install python-dbg python-numpy-dbg
$ python-dbg setup.py build_ext --inplace

```

然后你还需要用`python-dbg`运行你的脚本。确保在使用调试符号构建包时，如果先前已编译了 cython 扩展，则重新编译它们。如果您的软件包受版本控制，您可能需要在构建之前执行`git clean -fxd`或`hg purge --all`。

您还可以将其他参数传递给 gdb：

```py
$ cygdb /path/to/build/directory/ GDBARGS

```

即：

```py
$ cygdb . -- --args python-dbg mainscript.py

```

要告诉 cygdb 不要导入任何调试信息，请提供`--`作为第一个参数：

```py
$ cygdb --

```

## 使用调试器

Cython 调试器附带了一组支持断点，堆栈检查，源代码列表，步进，单步执行等命令。大多数这些命令与它们各自的 gdb 命令类似。

`cy break breakpoints...`

打破 Python，Cython 或 C 函数。首先，它将查找具有该名称的 Cython 函数，如果 cygdb 不知道具有该名称的函数（或方法），它将设置（待定）C 断点。 `-p`选项可用于指定 Python 断点。

可以为函数或方法名设置断点，也可以完全“限定”断点，这意味着给出函数的整个“路径”：

```py
(gdb) cy break cython_function_or_method
(gdb) cy break packagename.cython_module.cython_function
(gdb) cy break packagename.cython_module.ClassName.cython_method
(gdb) cy break c_function

```

您还可以打破 Cython 行号：

```py
(gdb) cy break :14
(gdb) cy break cython_module:14
(gdb) cy break packagename.cython_module:14

```

Python 断点当前支持模块的名称（不是整个包路径）和函数或方法：

```py
(gdb) cy break -p python_module.python_function_or_method
(gdb) cy break -p python_function_or_method

```

注意

Python 断点仅适用于 Python 构建，其中可以从调试器中读取 Python 帧信息。要确保这一点，请使用 Python 调试版本或使用调试支持编译的非剥离版本。

`cy step`

逐步完成 Python，Cython 或 C 代码。直接从 Cython 代码调用的 Python，Cython 和 C 函数被认为是相关的，并将被引入。

`cy next`

跨越 Python，Cython 或 C 代码。

`cy run`

运行程序。默认解释器是用于构建扩展的解释器，或者在“不导入调试信息”选项生效的情况下运行解释器`cygdb`。可以使用 gdb 的`file`命令覆盖解释器。

`cy cont`

继续该计划。

`cy up`

`cy down`

在堆栈中上下移动到被认为是相关的框架。

`cy finish`

执行直到满足向上相关的帧或停止执行。

`cy bt`

`cy backtrace`

打印所有相关的框架的回溯。 `-a`选项使其打印完整的回溯（所有 C 帧）。

`cy select`

按`cy backtrace`列出的编号选择堆栈帧。引入此命令是因为`cy backtrace`打印反向堆栈跟踪，因此帧编号与 gdb 的`bt`不同。

`cy print varname`

打印本地或全局 Cython，Python 或 C 变量（取决于上下文）。变量也可能被取消引用：

```py
(gdb) cy print x
x = 1
(gdb) cy print *x
*x = (PyObject) {
    _ob_next = 0x93efd8,
    _ob_prev = 0x93ef88,
    ob_refcnt = 65,
    ob_type = 0x83a3e0
}

```

`cy set cython_variable = value`

将 Cython 堆栈上的 Cython 变量设置为 value。

`cy list`

列出当前行周围的源代码。

`cy locals`

`cy globals`

打印所有本地和全局变量及其值。

`cy import FILE...`

从作为参数提供的文件中导入调试信息。导入调试信息的最简单方法是使用 cygdb 命令行工具。

`cy exec code`

在当前的 Python 或 Cython 框架中执行代码。这就像 Python 的交互式解释器一样。

对于 Python 帧，它使用 Python 框架中的全局变量和局部变量，对于 Cython 帧，它使用 Cython 模块上使用的全局变量的 dict 和一个填充了本地 Cython 变量的新 dict。

Note

`cy exec`修改状态并在调试对象中执行代码，因此可能存在危险。

例：

```py
(gdb) cy exec x + 1
2
(gdb) cy exec import sys; print sys.version_info
(2, 6, 5, 'final', 0)
(gdb) cy exec
>global foo
>
>foo = 'something'
>end

```

## 便利功能

以下函数是 gdb 函数，这意味着它们可以在 gdb 表达式中使用。

`cy_cname`(_varname_)

返回 Cython 变量的 C 变量名。对于全局变量，这可能实际上不是有效的。

`cy_cvalue`(_varname_)

返回 Cython 变量的值。

`cy_eval`(_expression_)

在最近的 Python 或 Cython 框架中评估 Python 代码，并将表达式的结果作为 gdb 值返回。如果成功则提供新引用，出错时为 NULL。

`cy_lineno`()

返回所选 Cython 帧中的当前行号。

Example:

```py
(gdb) print $cy_cname("x")
$1 = "__pyx_v_x"
(gdb) watch $cy_cvalue("x")
Hardware watchpoint 13: $cy_cvalue("x")
(gdb) cy set my_cython_variable = $cy_eval("{'spam': 'ham'}")
(gdb) print $cy_lineno()
$2 = 12

```

## 配置调试器

调试器的一些方面可以使用 gdb 参数进行配置。例如，可以禁用颜色，可以配置终端背景颜色和断点自动完成。

`cy_complete_unqualified`

告诉 Cython 调试器`cy break`是否还应该完成普通函数名称，即不以其模块名称为前缀。例如。如果你有一个名为`spam`的函数，在模块`M`中，它会告诉你是仅完成`M.spam`还是只完成`spam`。

默认值为 true。

`cy_colorize_code`

告诉调试器是否着色源代码。默认值为 true。

`cy_terminal_background_color`

告诉调试器有关终端背景颜色的信息，这会影响源代码着色。默认值为“dark”，另一个有效选项为“light”。

这是这些参数的使用方式：

```py
(gdb) set cy_complete_unqualified off
(gdb) set cy_terminal_background_color light
(gdb) show cy_colorize_code

```