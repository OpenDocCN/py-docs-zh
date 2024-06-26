# 源文件和编译

> 原文： [`docs.cython.org/en/latest/src/userguide/source_files_and_compilation.html`](http://docs.cython.org/en/latest/src/userguide/source_files_and_compilation.html)

Cython 源文件名由模块名称后跟`.pyx`扩展名组成，例如名为 primes 的模块将具有名为`primes.pyx`的源文件。

与 Python 不同，必须编译 Cython 代码。这发生在两个阶段：

> *   A `.pyx`文件由 Cython 编译为`.c`文件。
> *   `.c`文件由 C 编译器编译为`.so`文件（或 Windows 上的`.pyd`文件）

一旦编写了`.pyx`文件，就可以通过几种方法将其转换为扩展模块。

以下小节介绍了构建扩展模块的几种方法，以及如何将指令传递给 Cython 编译器。

## 从命令行编译

有两种从命令行编译的方法。

*   `cython`命令获取`.py`或`.pyx`文件并将其编译为 C / C ++文件。
*   `cythonize`命令获取`.py`或`.pyx`文件并将其编译为 C / C ++文件。然后，它将 C / C ++文件编译为可直接从 Python 导入的扩展模块。

### 使用`cython`命令进行编译

一种方法是使用 Cython 编译器手动编译它，例如：

```py
$ cython primes.pyx

```

这将生成一个名为`primes.c`的文件，然后需要使用适合您平台的任何选项使用 C 编译器编译该文件以生成扩展模块。有关这些选项，请查看官方 Python 文档。

另一种可能更好的方法是使用 Cython 提供的 [`distutils`](https://docs.python.org/3/library/distutils.html#module-distutils "(in Python v3.7)") 扩展。这种方法的好处是它将提供平台特定的编译选项，就像一个精简的自动工具。

### 使用`cythonize`命令进行编译

使用您的选项和`.pyx`文件列表运行`cythonize`编译器命令以生成扩展模块。例如：

```py
$ cythonize -a -i yourmod.pyx

```

这将创建一个`yourmod.c`文件（或 C ++模式下的`yourmod.cpp`），对其进行编译，并将生成的扩展模块（`.so`或`.pyd`，具体取决于您的平台）放在源文件旁边以进行直接导入（`-i`建立“到位”）。 `-a`开关另外生成源代码的带注释的 html 文件。

`cythonize`命令接受多个源文件和类似`**/*.pyx`的 glob 模式作为参数，并且还了解运行多个并行构建作业的常用`-j`选项。在没有其他选项的情况下调用时，它只会将源文件转换为`.c`或`.cpp`文件。传递`-h`标志以获取支持选项的完整列表。

更简单的命令行工具`cython`仅调用源代码转换器。

在手动编译的情况下，如何编译`.c`文件将根据您的操作系统和编译器而有所不同。用于编写扩展模块的 Python 文档应该包含系统的一些详细信息。例如，在 Linux 系统上，它看起来可能类似于：

```py
$ gcc -shared -pthread -fPIC -fwrapv -O2 -Wall -fno-strict-aliasing \
      -I/usr/include/python3.5 -o yourmod.so yourmod.c

```

（`gcc`需要有包含头文件的路径和要链接的库的路径。）

编译后，将`yourmod.so`（Windows 的`yourmod.pyd`）文件写入目标目录，您的模块`yourmod`可以像任何其他 Python 模块一样导入。请注意，如果您不依赖于`cythonize`或 distutils，您将不会自动受益于 CPython 为消除歧义而生成的特定于平台的文件扩展名，例如 CPython 3.5 的常规 64 位 Linux 安装上的`yourmod.cpython-35m-x86_64-linux-gnu.so`。

## 基本 setup.py

Cython 提供的 distutils 扩展允许您将`.pyx`文件直接传递到安装文件中的`Extension`构造函数。

如果你有一个 Cython 文件要转换为编译扩展名，比如文件名`example.pyx`，关联的`setup.py`将是：

```py
from distutils.core import setup
from Cython.Build import cythonize

setup(
    ext_modules = cythonize("example.pyx")
)

```

要更全面地了解`setup.py`官方 [`distutils`](https://docs.python.org/3/library/distutils.html#module-distutils "(in Python v3.7)") 文档。要编译扩展以在当前目录中使用，请使用：

```py
$ python setup.py build_ext --inplace

```

### 配置 C-Build

如果您在非标准位置包含文件，则可以将`include_path`参数传递给`cythonize`：

```py
from distutils.core import setup
from Cython.Build import cythonize

setup(
    name="My hello app",
    ext_modules=cythonize("src/*.pyx", include_path=[...]),
)

```

通常，提供 C 级 API 的 Python 包提供了一种查找必要包含文件的方法，例如，为 NumPy：

```py
include_path = [numpy.get_include()]

```

注意

使用内存视图或使用`import numpy`导入 NumPy 并不意味着您必须添加 NumPy 包含文件的路径。只有在使用`cimport numpy`时才需要添加此路径。

尽管如此，您仍然会收到编译器中的以下警告，因为 Cython 使用的是已弃用的 Numpy API：

```py
.../include/numpy/npy_1_7_deprecated_api.h:15:2: warning: #warning "Using deprecated NumPy API, disable it by " "#defining NPY_NO_DEPRECATED_API NPY_1_7_API_VERSION" [-Wcpp]

```

目前，这只是一个警告，你可以忽略。

如果需要指定编译器选项，要链接的库或其他链接器选项，则需要手动创建`Extension`实例（请注意，glob 语法仍可用于在一行中指定多个扩展名）：

```py
from distutils.core import setup
from distutils.extension import Extension
from Cython.Build import cythonize

extensions = [
    Extension("primes", ["primes.pyx"],
        include_dirs=[...],
        libraries=[...],
        library_dirs=[...]),
    # Everything but primes.pyx is included here.
    Extension("*", ["*.pyx"],
        include_dirs=[...],
        libraries=[...],
        library_dirs=[...]),
]
setup(
    name="My hello app",
    ext_modules=cythonize(extensions),
)

```

请注意，使用 setuptools 时，您应该在 Cython 之前导入它，因为 setuptools 可能会替换 distutils 中的`Extension`类。否则，两人可能不同意这里使用的课程。

另请注意，如果使用 setuptools 而不是 distutils，则运行`python setup.py install`时的默认操作是创建一个压缩的`egg`文件，当您尝试从依赖项中使用它们时，这些文件无法与`pxd`文件一起用于`pxd`文件包。为防止这种情况，请在`setup()`的参数中包含`zip_safe=False`。

如果您的选项是静态的（例如，您不需要调用`pkg-config`之类的工具来确定它们），您也可以使用文件开头的特殊注释块直接在.pyx 或.pxd 源文件中提供它们。 ：

```py
# distutils: libraries = spam eggs
# distutils: include_dirs = /opt/food/include

```

如果您导入多个定义库的.pxd 文件，那么 Cython 会合并库列表，因此这可以按预期工作（与其他选项类似，如上面的`include_dirs`）。

如果你有一些用 Cython 包装的 C 文件，并且你想将它们编译到你的扩展中，你可以定义 distutils `sources`参数：

```py
# distutils: sources = helper.c, another_helper.c

```

请注意，这些源将添加到当前扩展模块的源列表中。在`setup.py`文件中将其拼写如下：

```py
from distutils.core import setup
from Cython.Build import cythonize
from distutils.extension import Extension

sourcefiles = ['example.pyx', 'helper.c', 'another_helper.c']

extensions = [Extension("example", sourcefiles)]

setup(
    ext_modules=cythonize(extensions)
)

```

`Extension`类有很多选项，可以在 [distutils 文档](https://docs.python.org/extending/building.html)中找到更全面的解释。要了解的一些有用选项是`include_dirs`，`libraries`和`library_dirs`，它们指定链接到外部库时在哪里可以找到`.h`和库文件。

有时这还不够，你需要更精细的自定义 distutils `Extension`。为此，您可以提供自定义函数`create_extension`，以便在 Cython 处理源，依赖项和`# distutils`指令之后但在文件实际进行 Cython 化之前创建最终的`Extension`对象。该函数有 2 个参数`template`和`kwds`，其中`template`是作为 Cython 输入的`Extension`对象，`kwds`是 [`dict`](https://docs.python.org/3/library/stdtypes.html#dict "(in Python v3.7)") ，应该使用所有关键字创建`Extension`。函数`create_extension`必须返回 2 元组`(extension, metadata)`，其中`extension`是创建的`Extension`，`metadata`是元数据，将在生成的 C 文件的顶部写为 JSON。此元数据仅用于调试目的，因此您可以在其中放置任何内容（只要它可以转换为 JSON）。默认功能（在`Cython.Build.Dependencies`中定义）是：

```py
def default_create_extension(template, kwds):
    if 'depends' in kwds:
        include_dirs = kwds.get('include_dirs', []) + ["."]
        depends = resolve_depends(kwds['depends'], include_dirs)
        kwds['depends'] = sorted(set(depends + template.depends))

    t = template.__class__
    ext = t(**kwds)
    metadata = dict(distutils=kwds, module_name=kwds['name'])
    return ext, metadata

```

如果您将字符串而不是`Extension`传递给`cythonize()`，`template`将是没有来源的`Extension`。例如，如果执行`cythonize("*.pyx")`，`template`将为`Extension(name="*.pyx", sources=[])`。

举个例子，这会将`mylib`作为库添加到每个扩展名中：

```py
from Cython.Build.Dependencies import default_create_extension

def my_create_extension(template, kwds):
    libs = kwds.get('libraries', []) + ["mylib"]
    kwds['libraries'] = libs
    return default_create_extension(template, kwds)

ext_modules = cythonize(..., create_extension=my_create_extension)

```

Note

如果你并行 Cython 化（使用`nthreads`参数），那么`create_extension`的参数必须是 pickleable。特别是，它不能是 lambda 函数。

### Cythonize 参数

函数`cythonize()`可以使用额外的参数来允许您自定义构建。

`Cython.Build.``cythonize`(_module_list_, _exclude=None_, _nthreads=0_, _aliases=None_, _quiet=False_, _force=False_, _language=None_, _exclude_failures=False_, _**options_)

将一组源模块编译为 C / C ++文件，并为它们返回 distutils Extension 对象的列表。

&lt;colgroup&gt;&lt;col class="field-name"&gt; &lt;col class="field-body"&gt;&lt;/colgroup&gt;
| 参数： | 

*   **module_list** - 作为模块列表，传递一个 glob 模式，一个 glob 模式列表或一个扩展对象列表。后者允许您通过常规 distutils 选项单独配置扩展。您还可以传递具有 glob 模式作为其源的扩展对象。然后，cythonize 将解析模式并为每个匹配的文件创建扩展的副本。
*   **排除** - 将 glob 模式作为`module_list`传递时，可以通过将某些模块名称传递给`exclude`选项来明确排除它们。
*   **nthreads** - 并行编译的并发构建数（需要`multiprocessing`模块）。
*   **别名** - 如果你想使用像`# distutils: ...`这样的编译器指令，但只能在编译时（运行`setup.py`时）知道要使用哪些值，你可以使用别名和在调用 `cythonize()` 时传递将这些别名映射到 Python 字符串的字典。例如，假设您要使用编译器指令`# distutils: include_dirs = ../static_libs/include/`，但此路径并不总是固定，您希望在运行`setup.py`时找到它。然后，您可以执行`# distutils: include_dirs = MY_HEADERS`，在`setup.py`中找到`MY_HEADERS`的值，将其作为字符串放入名为`foo`的 python 变量中，然后调用`cythonize(..., aliases={'MY_HEADERS': foo})`。
*   **安静** - 如果为 True，Cython 将不会在编译期间打印错误和警告消息。
*   **force** - 强制重新编译 Cython 模块，即使时间戳不表示需要重新编译。
*   **语言** - 要全局启用 C ++模式，可以传递`language='c++'`。否则，将根据编译器指令在每个文件级别确定。这仅影响基于文件名找到的模块。传入 `cythonize()` 的扩展实例不会更改。建议使用编译器指令`# distutils: language = c++`而不是此选项。
*   **exclude_failures** - 对于广泛的“尝试编译”模式，忽略编译失败并简单地排除失败的扩展，传递`exclude_failures=True`。请注意，这仅适用于编译`.py`文件，这些文件也可以在不编译的情况下使用。
*   **注释** - 如果`True`，将为每个编译的`.pyx`或`.py`文件生成一个 HTML 文件。与普通的 C 代码相比，HTML 文件指示了每个源代码行中的 Python 交互量。它还允许您查看为每行 Cython 代码生成的 C / C ++代码。当优化速度函数以及确定何时 释放 GIL 时，此报告非常有用：通常，`nogil`块可能只包含“白色”代码。参见 中的实例确定添加类型 或 Primes 的位置。
*   **compiler_directives** - 允许在`setup.py`中设置编译器指令，如下所示：`compiler_directives={'embedsignature': True}`。参见 编译器指令 。

 |
| --- | --- |

## 包中的多个 Cython 文件

要自动编译多个 Cython 文件而不显式列出所有这些文件，可以使用 glob 模式：

```py
setup(
    ext_modules = cythonize("package/*.pyx")
)

```

如果通过`cythonize()`传递它们，也可以在`Extension`对象中使用 glob 模式：

```py
extensions = [Extension("*", ["*.pyx"])]

setup(
    ext_modules = cythonize(extensions)
)

```

### 分发 Cython 模块

强烈建议您分发生成的`.c`文件以及 Cython 源，以便用户可以安装模块而无需使用 Cython。

还建议在您分发的版本中默认不启用 Cython 编译。即使用户安装了 Cython，他/她可能也不想仅仅使用它来安装模块。此外，安装的版本可能与您使用的版本不同，并且可能无法正确编译您的源。

这只是意味着您附带的`setup.py`文件将只是生成的 &lt;cite&gt;.c&lt;/cite&gt; 文件中的正常 distutils 文件，对于我们将要使用的基本示例：

```py
from distutils.core import setup
from distutils.extension import Extension

setup(
    ext_modules = [Extension("example", ["example.c"])]
)

```

通过更改扩展模块源的文件扩展名，可以很容易地与`cythonize()`结合使用：

```py
from distutils.core import setup
from distutils.extension import Extension

USE_CYTHON = ...   # command line option, try-import, ...

ext = '.pyx' if USE_CYTHON else '.c'

extensions = [Extension("example", ["example"+ext])]

if USE_CYTHON:
    from Cython.Build import cythonize
    extensions = cythonize(extensions)

setup(
    ext_modules = extensions
)

```

如果您有许多扩展并希望避免声明中的额外复杂性，您可以使用它们的正常 Cython 源声明它们，然后在不使用 Cython 时调用以下函数而不是`cythonize()`来调整 Extensions 中的源列表：

```py
import os.path

def no_cythonize(extensions, **_ignore):
    for extension in extensions:
        sources = []
        for sfile in extension.sources:
            path, ext = os.path.splitext(sfile)
            if ext in ('.pyx', '.py'):
                if extension.language == 'c++':
                    ext = '.cpp'
                else:
                    ext = '.c'
                sfile = path + ext
            sources.append(sfile)
        extension.sources[:] = sources
    return extensions

```

另一个选择是使 Cython 成为系统的设置依赖项，并使用 Cython 的 build_ext 模块作为构建过程的一部分运行`cythonize`：

```py
setup(
    setup_requires=[
        'cython>=0.x',
    ],
    extensions = [Extension("*", ["*.pyx"])],
    cmdclass={'build_ext': Cython.Build.build_ext},
    ...
)

```

如果要公开库的 C 级接口以便其他库从 cimport，请使用 package_data 来安装`.pxd`文件，例如：

```py
setup(
    package_data = {
        'my_package': ['*.pxd'],
        'my_package/sub_package': ['*.pxd'],
    },
    ...
)

```

如果这些`.pxd`文件包含纯粹的外部库声明，则它们不需要具有相应的`.pyx`模块。

请记住，如果使用 setuptools 而不是 distutils，则运行`python setup.py install`时的默认操作是创建一个压缩的`egg`文件，当您尝试从依赖包中使用它们时，这些文件无法与`pxd`文件一起用于`pxd`文件。为防止这种情况，请在`setup()`的参数中包含`zip_safe=False`。

## 集成多个模块

在某些情况下，将多个 Cython 模块（或其他扩展模块）链接到单个二进制文件中可能很有用，例如：将 Python 嵌入另一个应用程序时。这可以通过 CPython 的 inittab 导入机制来完成。

创建一个新的 C 文件以集成扩展模块并将其添加到它：

```py
#if PY_MAJOR_VERSION < 3
# define MODINIT(name)  init ## name
#else
# define MODINIT(name)  PyInit_ ## name
#endif

```

如果您只定位 Python 3.x，只需使用`PyInit_`作为前缀。

然后，对于每个模块，声明其模块 init 函数如下，将`some_module_name`替换为模块的名称：

```py
PyMODINIT_FUNC  MODINIT(some_module_name) (void);

```

在 C ++中，将它们声明为`extern C`。

如果您不确定模块初始化函数的名称，请参阅生成的模块源文件并查找以`PyInit_`开头的函数名称。

接下来，在使用`Py_Initialize()`从应用程序代码启动 Python 运行时之前，需要使用`PyImport_AppendInittab()` C-API 函数在运行时初始化模块，再次插入每个模块的名称：

```py
PyImport_AppendInittab("some_module_name", MODINIT(some_module_name));

```

这样可以为嵌入式扩展模块进行正常导入。

为了防止连接的二进制文件将所有模块初始化函数导出为公共符号，如果在 C 编译模块 C 文件时定义了宏`CYTHON_NO_PYINIT_EXPORT`，则 Cython 0.28 及更高版本可以隐藏这些符号。

另请查看 [cython_freeze](https://github.com/cython/cython/blob/master/bin/cython_freeze) 工具。它可以生成必要的样板代码，用于将一个或多个模块链接到单个 Python 可执行文件中。

## 用`pyximport` 编译

为了在开发期间构建 Cython 模块而不在每次更改后显式运行`setup.py`，您可以使用`pyximport`：

```py
>>> import pyximport; pyximport.install()
>>> import helloworld
Hello World

```

这允许您在 Py​​thon 尝试导入的每个`.pyx`上自动运行 Cython。只有在没有额外的 C 库且不需要特殊的构建设置的情况下，才应该将它用于简单的 Cython 构建。

也可以编译正在导入的新`.py`模块（包括标准库和已安装的软件包）。要使用此功能，只需告诉`pyximport`：

```py
>>> pyximport.install(pyimport=True)

```

在 Cython 无法编译 Python 模块的情况下，`pyximport`将回退到加载源模块。

请注意，建议不要让`pyximport`在最终用户端构建代码，因为它会挂钩到导入系统。满足最终用户的最佳方式是以[轮](https://wheel.readthedocs.io/)包装格式提供预先构建的二进制包。

### 参数

函数`pyximport.install()`可以使用几个参数来影响 Cython 或 Python 文件的编译。

`pyximport.``install`(_pyximport=True_, _pyimport=False_, _build_dir=None_, _build_in_temp=True_, _setup_args=None_, _reload_support=False_, _load_py_module_on_import_failure=False_, _inplace=False_, _language_level=None_)

pyxinstall 的主要入口点。

调用此方法在元数据路径中为单个 Python 进程安装`.pyx`导入挂钩。如果您希望在使用 Python 时安装它，请将其添加到`sitecustomize`（如上所述）。

&lt;colgroup&gt;&lt;col class="field-name"&gt; &lt;col class="field-body"&gt;&lt;/colgroup&gt;
| Parameters: | 

*   **pyximport** - 如果设置为 False，则不尝试导入`.pyx`文件。
*   **pyimport** - 您可以传递`pyimport=True`以在元路径中安装`.py`导入挂钩。但请注意，它是相当实验性的，对于某些`.py`文件和软件包根本不起作用，并且由于搜索和编译而会大大减慢您的导入速度。使用风险由您自己承担。
*   **build_dir** - 默认情况下，已编译的模块最终将位于用户主目录的`.pyxbld`目录中。传递一个不同的路径作为`build_dir`将覆盖此。
*   **build_in_temp** - 如果`False`，将在本地生成 C 文件。使用复杂的依赖项和调试变得更加容易。这主要可能会干扰同名的现有文件。
*   **setup_args** - 分布参数的字典。见`distutils.core.setup()`。
*   **reload_support** - 支持动态`reload(my_module)`，例如在 Cython 代码更改后。当无法覆盖先前加载的模块文件时，该帐户可能会出现附加文件`&lt;so_path&gt;.reloadNN`。
*   **load_py_module_on_import_failure** - 如果`.py`文件的编译成功，但后续导入由于某种原因失败，请使用普通`.py`模块而不是编译模块重试导入。请注意，这可能会导致在导入期间更改系统状态的模块出现不可预测的结果，因为第二次导入将在导入已编译模块失败后系统处于的任何状态下重新运行这些修改。
*   **就地** - 在源文件旁边安装已编译的模块（Linux 的`.so`和 Mac 的`.pyd`）。
*   **language_level** - 要使用的源语言级别：2 或 3.默认情况下，使用.py 文件的当前 Python 运行时的语言级别和`.pyx`文件的 Py2。

 |
| --- | --- |

### 依赖性处理

由于`pyximport`内部不使用`cythonize()`，因此它当前需要不同的依赖设置。可以声明您的模块依赖于多个文件（可能是`.h`和`.pxd`文件）。如果您的 Cython 模块名为`foo`，因此具有文件名`foo.pyx`，那么您应该在名为`foo.pyxdep`的同一目录中创建另一个文件。 `modname.pyxdep`文件可以是文件名列表或“globs”（如`*.pxd`或`include/*.h`）。每个文件名或 glob 必须在单独的行上。在决定是否重建模块之前，Pyximport 将检查每个文件的文件日期。为了跟踪已经处理了依赖关系的事实，Pyximport 更新了“.pyx”源文件的修改时间。未来版本可能会做更复杂的事情，比如直接通知依赖关系的 distutils。

### 限制

`pyximport`不使用`cythonize()`。因此，不可能在 Cython 文件的顶部使用编译器指令或将 Cython 代码编译为 C ++。

Pyximport 不会让您控制 Cython 文件的编译方式。通常默认值很好。如果你想用半-C，半 Cython 编写你的程序并将它们构建到一个库中，你可能会遇到问题。

Pyximport 不会隐藏导入过程生成的 Distutils / GCC 警告和错误。可以说，如果出现问题以及原因，这将为您提供更好的反馈。如果没有出现任何问题，它会给你一种温暖的模糊感觉，pyximport 确实按照预期重建你的模块。

选项`reload_support=True`提供基本模块重新加载支持。请注意，这将为每个构建生成一个新的模块文件名，从而最终将多个共享库随时间加载到内存中。 CPython 对重新加载共享库的支持有限，参见 [PEP 489](https://www.python.org/dev/peps/pep-0489/) 。

Pyximport 将您的`.c`文件和特定于平台的二进制文件放入一个单独的构建目录，通常是`$HOME/.pyxblx/`。要将其复制回包层次结构（通常在源文件旁边）以进行手动重用，可以传递选项`inplace=True`。

## 用`cython.inline` 编译

也可以用类似于 SciPy 的`weave.inline`的方式编译 Cython。例如：

```py
>>> import cython
>>> def f(a):
...     ret = cython.inline("return a+b", b=3)
...

```

未绑定的变量会自动从周围的本地和全局范围中提取，并且编译结果将被缓存以便有效地重复使用。

## 用 Sage 编译

Sage 笔记本允许通过在单元格顶部键入`%cython`并对其进行评估来透明地编辑和编译 Cython 代码。 Cython 单元格中定义的变量和函数将导入到正在运行的会话中。有关详细信息，请查看 [Sage 文档](https://www.sagemath.org/doc/)。

您可以通过指定以下指令来定制 Cython 编译器的行为。

## 使用 Jupyter 笔记本进行编译

使用 Cython 可以在笔记本单元中编译代码。为此你需要加载 Cython 魔法：

```py
%load_ext cython

```

然后，您可以通过在其上面写入`%%cython`来定义 Cython 单元格。像这样：

```py
%%cython

cdef int a = 0
for i in range(10):
    a += i
print(a)

```

请注意，每个单元格将编译为单独的扩展模块。因此，如果您在 Cython 单元格中使用包，则必须在同一单元格中导入此包。在先前的单元格中导入包是不够的。如果你不遵守，Cython 会告诉你编译时有“未定义的全局名称”。

然后将单元格的全局名称（顶级函数，类，变量和模块）加载到笔记本的全局命名空间中。所以最后，它的行为就好像你执行了一个 Python 单元格。

下面列出了 Cython 魔术的其他允许参数。您也可以通过在 IPython 或 Jupyter 笔记本中键入``%%cython?`来查看它们。

<colgroup><col width="25%"> <col width="75%"></colgroup> 
| -a，-annotate | 生成源的彩色 HTML 版本。 |
| -annotate-fullc | 生成源的彩色 HTML 版本，其中包括整个生成的 C / C ++代码。 |
| - +，-cplus | 输出 C ++而不是 C 文件。 |
| -f，-force | 强制编译新模块，即使以前编译过源也是如此。 |
| -3 | 选择 Python 3 语法 |
| -2 | 选择 Python 2 语法 |
| -c = COMPILE_ARGS，-compile-args = COMPILE_ARGS | 通过 extra_compile_args 传递给编译器的额外标志。 |
| -link-args LINK_ARGS | 通过 extra_link_args 传递给链接器的额外标志。 |
| -l LIB，-lib LIB | 添加库以链接扩展名（可以多次指定）。 |
| -L dir | 添加库目录列表的路径（可以多次指定）。 |
| - 我包括，包括 INCLUDE | 添加包含目录列表的路径（可以多次指定）。 |
| -S，-src | 添加 src 文件列表的路径（可以多次指定）。 |
| -n NAME， - name NAME | 指定 Cython 模块的名称。 |
| -pgo | 在 C 编译器中启用配置文件引导优化。编译单元格两次并在其间执行以生成运行时配置文件。 |
| -verbose | 打印调试信息，如生成的.c / .cpp 文件位置和调用的精确 gcc / g ++命令。 |

### 编译器选项

在调用`cythonize()`之前，可以在`setup.py`中设置编译器选项，如下所示：

```py
from distutils.core import setup

from Cython.Build import cythonize
from Cython.Compiler import Options

Options.docstrings = False

setup(
    name = "hello",
    ext_modules = cythonize("lib.pyx"),
)

```

以下是可用的选项：

`Cython.Compiler.Options.``docstrings` _= True_

是否在 Python 扩展中包含 docstring。如果为 False，则二进制大小将更小，但任何类或函数的`__doc__`属性将为空字符串。

`Cython.Compiler.Options.``embed_pos_in_docstring` _= False_

将源代码位置嵌入到函数和类的文档字符串中。

`Cython.Compiler.Options.``emit_code_comments` _= True_

将原始源代码逐行复制到生成的代码文件中的 C 代码注释中，以帮助理解输出。这也是覆盖率分析所必需的。

`Cython.Compiler.Options.``generate_cleanup_code` _= False_

在退出时为每个模块删除全局变量以进行垃圾回收。 0：无，1 +：实习对象，2 +：cdef 全局，3 +：类型对象主要用于降低 Valgrind 中的噪声，仅在进程退出时执行（当所有内存都将被回收时）。

`Cython.Compiler.Options.``clear_to_none` _= True_

tp_clear（）应该将对象字段设置为 None 而不是将它们清除为 NULL 吗？

`Cython.Compiler.Options.``annotate` _= False_

生成带注释的 HTML 版本的输入源文件，以进行调试和优化。这与`cythonize()`中的`annotate`参数具有相同的效果。

`Cython.Compiler.Options.``fast_fail` _= False_

这将在第一次发生错误时中止编译，而不是试图继续进行并打印更多错误消息。

`Cython.Compiler.Options.``warning_errors` _= False_

将所有警告变为错误。

`Cython.Compiler.Options.``error_on_unknown_names` _= True_

使未知名称成为错误。 Python 在运行时遇到未知名称时会引发 NameError，而此选项会使它们成为编译时错误。如果您想要完全兼容 Python，则应禁用此选项以及“cache_builtins”。

`Cython.Compiler.Options.``error_on_uninitialized` _= True_

使未初始化的局部变量引用编译时错误。 Python 在运行时引发 UnboundLocalError，而此选项使它们成为编译时错误。请注意，此选项仅影响“python 对象”类型的变量。

`Cython.Compiler.Options.``convert_range` _= True_

当`i`是 C 整数类型时，这将把`for i in range(...)`形式的语句转换为`for i from ...`，并且可以确定方向（即步骤的符号）。警告：如果范围导致 i 的分配溢出，则可能会更改语义。具体来说，如果设置了此选项，则在输入循环之前将引发错误，而如果没有此选项，则循环将执行，直到遇到溢出值。

`Cython.Compiler.Options.``cache_builtins` _= True_

在模块初始化时，仅对内置名称执行一次查找。如果在初始化期间找不到它使用的内置名称，这将阻止导入模块。默认为 True。请注意，在 Python 3.x 中构建时，一些遗留的内置函数会自动从 Python 2 名称重新映射到 Cython 的 Python 3 名称，这样即使启用此选项，它们也不会妨碍它们。

`Cython.Compiler.Options.``gcc_branch_hints` _= True_

生成分支预测提示以加快错误处理等。

`Cython.Compiler.Options.``lookup_module_cpdef` _= False_

如果 cpdef 函数为 foo，则启用此选项以允许写入`your_module.foo = ...`来覆盖定义，代价是每次调用时额外的字典查找。如果这是假的，它只生成 Python 包装器而不进行覆盖检查。

`Cython.Compiler.Options.``embed` _= None_

是否嵌入 Python 解释器，用于制作独立的可执行文件或从外部库调用。这将提供一个 C 函数，它初始化解释器并执行该模块的主体。有关具体示例，请参见[此演示](https://github.com/cython/cython/tree/master/Demos/embed)。如果为 true，则初始化函数是 C main（）函数，但此选项也可以设置为非空字符串以显式提供函数名称。默认值为 False。

`Cython.Compiler.Options.``cimport_from_pyx` _= False_

允许从没有 pxd 文件的 pyx 文件中导入。

`Cython.Compiler.Options.``buffer_max_dims` _= 8_

缓冲区的最大维数 - 设置低于 numpy 中的维数，因为切片按值传递并涉及大量复制。

`Cython.Compiler.Options.``closure_freelist_size` _= 8_

要保留在空闲列表中的函数闭包实例数（0：没有空闲列表）

## 编译器指令

编译器指令是影响 Cython 代码行为的指令。以下是当前支持的指令列表：

`binding` (True / False)

Controls whether free functions behave more like Python’s CFunctions (e.g. [`len()`](https://docs.python.org/3/library/functions.html#len "(in Python v3.7)")) or, when set to True, more like Python’s functions. When enabled, functions will bind to an instance when looked up as a class attribute (hence the name) and will emulate the attributes of Python functions, including introspections like argument names and annotations. Default is False.

`boundscheck` (True / False)

If set to False, Cython is free to assume that indexing operations ([]-operator) in the code will not cause any IndexErrors to be raised. Lists, tuples, and strings are affected only if the index can be determined to be non-negative (or if `wraparound` is False). Conditions which would normally trigger an IndexError may instead cause segfaults or data corruption if this is set to False. Default is True.

`wraparound` (True / False)

In Python, arrays and sequences can be indexed relative to the end. For example, A[-1] indexes the last value of a list. In C, negative indexing is not supported. If set to False, Cython is allowed to neither check for nor correctly handle negative indices, possibly causing segfaults or data corruption. If bounds checks are enabled (the default, see `boundschecks` above), negative indexing will usually raise an `IndexError` for indices that Cython evaluates itself. However, these cases can be difficult to recognise in user code to distinguish them from indexing or slicing that is evaluated by the underlying Python array or sequence object and thus continues to support wrap-around indices. It is therefore safest to apply this option only to code that does not process negative indices at all. Default is True.

`initializedcheck` (True / False)

If set to True, Cython checks that a memoryview is initialized whenever its elements are accessed or assigned to. Setting this to False disables these checks. Default is True.

`nonecheck` (True / False)

If set to False, Cython is free to assume that native field accesses on variables typed as an extension type, or buffer accesses on a buffer variable, never occurs when the variable is set to `None`. Otherwise a check is inserted and the appropriate exception is raised. This is off by default for performance reasons. Default is False.

`overflowcheck` (True / False)

If set to True, raise errors on overflowing C integer arithmetic operations. Incurs a modest runtime penalty, but is much faster than using Python ints. Default is False.

`overflowcheck.fold` (True / False)

If set to True, and overflowcheck is True, check the overflow bit for nested, side-effect-free arithmetic expressions once rather than at every step. Depending on the compiler, architecture, and optimization settings, this may help or hurt performance. A simple suite of benchmarks can be found in `Demos/overflow_perf.pyx`. Default is True.

`embedsignature` (True / False)

If set to True, Cython will embed a textual copy of the call signature in the docstring of all Python visible functions and classes. Tools like IPython and epydoc can thus display the signature, which cannot otherwise be retrieved after compilation. Default is False.

`cdivision` (True / False)

If set to False, Cython will adjust the remainder and quotient operators C types to match those of Python ints (which differ when the operands have opposite signs) and raise a `ZeroDivisionError` when the right operand is 0\. This has up to a 35% speed penalty. If set to True, no checks are performed. See [CEP 516](https://github.com/cython/cython/wiki/enhancements-division). Default is False.

`cdivision_warnings` (True / False)

If set to True, Cython will emit a runtime warning whenever division is performed with negative operands. See [CEP 516](https://github.com/cython/cython/wiki/enhancements-division). Default is False.

`always_allow_keywords` (True / False)

Avoid the `METH_NOARGS` and `METH_O` when constructing functions/methods which take zero or one arguments. Has no effect on special methods and functions with more than one argument. The `METH_NOARGS` and `METH_O` signatures provide faster calling conventions but disallow the use of keywords.

`profile` (True / False)

Write hooks for Python profilers into the compiled C code. Default is False.

`linetrace` (True / False)

Write line tracing hooks for Python profilers or coverage reporting into the compiled C code. This also enables profiling. Default is False. Note that the generated module will not actually use line tracing, unless you additionally pass the C macro definition `CYTHON_TRACE=1` to the C compiler (e.g. using the distutils option `define_macros`). Define `CYTHON_TRACE_NOGIL=1` to also include `nogil` functions and sections.

`infer_types` (True / False)

Infer types of untyped variables in function bodies. Default is None, indicating that only safe (semantically-unchanging) inferences are allowed. In particular, inferring *integral*types for variables*used in arithmetic expressions* is considered unsafe (due to possible overflow) and must be explicitly requested.

`language_level` (2/3/3str)

Globally set the Python language level to be used for module compilation. Default is compatibility with Python 2\. To enable Python 3 source code semantics, set this to 3 (or 3str) at the start of a module or pass the “-3” or “–3str” command line options to the compiler. The `3str` option enables Python 3 semantics but does not change the `str` type and unprefixed string literals to `unicode` when the compiled code runs in Python 2.x. Note that cimported files inherit this setting from the module being compiled, unless they explicitly set their own language level. Included source files always inherit this setting.

`c_string_type` (bytes / str / unicode)

Globally set the type of an implicit coercion from char* or std::string.

`c_string_encoding` (ascii, default, utf-8, etc.)

Globally set the encoding to use when implicitly coercing char* or std:string to a unicode object. Coercion from a unicode object to C type is only allowed when set to `ascii` or `default`, the latter being utf-8 in Python 3 and nearly-always ascii in Python 2.

`type_version_tag` (True / False)

Enables the attribute cache for extension types in CPython by setting the type flag `Py_TPFLAGS_HAVE_VERSION_TAG`. Default is True, meaning that the cache is enabled for Cython implemented types. To disable it explicitly in the rare cases where a type needs to juggle with its `tp_dict` internally without paying attention to cache consistency, this option can be set to False.

`unraisable_tracebacks` (True / False)

Whether to print tracebacks when suppressing unraisable exceptions.

`iterable_coroutine` (True / False)

[PEP 492](https://www.python.org/dev/peps/pep-0492/) specifies that async-def coroutines must not be iterable, in order to prevent accidental misuse in non-async contexts. However, this makes it difficult and inefficient to write backwards compatible code that uses async-def coroutines in Cython but needs to interact with async Python code that uses the older yield-from syntax, such as asyncio before Python 3.5\. This directive can be applied in modules or selectively as decorator on an async-def coroutine to make the affected coroutine(s) iterable and thus directly interoperable with yield-from.

### 可配置的优化

`optimize.use_switch` (True / False)

Whether to expand chained if-else statements (including statements like `if x == 1 or x == 2:`) into C switch statements. This can have performance benefits if there are lots of values but cause compiler errors if there are any duplicate values (which may not be detectable at Cython compile time for all C constants). Default is True.

`optimize.unpack_method_calls` (True / False)

Cython can generate code that optimistically checks for Python method objects at call time and unpacks the underlying function to call it directly. This can substantially speed up method calls, especially for builtins, but may also have a slight negative performance impact in some cases where the guess goes completely wrong. Disabling this option can also reduce the code size. Default is True.

### 警告

所有警告指令都采用 True / False 作为打开/关闭警告的选项。

`warn.undeclared` (default False)

Warns about any variables that are implicitly declared without a `cdef` declaration

`warn.unreachable` (default True)

Warns about code paths that are statically determined to be unreachable, e.g. returning twice unconditionally.

`warn.maybe_uninitialized` (default False)

Warns about use of variables that are conditionally uninitialized.

`warn.unused` (default False)

Warns about unused variables and declarations

`warn.unused_arg` (default False)

Warns about unused function arguments

`warn.unused_result` (default False)

Warns about unused assignment to the same name, such as `r = 2; r = 1 + 2`

`warn.multiple_declarators` (default True)

Warns about multiple variables declared on the same line with at least one pointer type. For example `cdef double* a, b` - which, as in C, declares `a` as a pointer, `b` as a value type, but could be mininterpreted as declaring two pointers.

### 如何设置指令

#### 全球

可以通过文件顶部附近的特殊标题注释设置编译器指令，如下所示：

```py
# cython: language_level=3, boundscheck=False

```

注释必须出现在任何代码之前（但可以出现在其他注释或空格之后）。

也可以使用-X 开关在命令行上传递指令：

```py
$ cython -X boundscheck=True ...

```

在命令行上传递的指令将覆盖在头注释中设置的指令。

#### 本地

对于本地块，您需要使用特殊的内置`cython`模块：

```py
#!python
cimport cython

```

然后你可以使用指令作为装饰器或在 with 语句中，如下所示：

```py
#!python
@cython.boundscheck(False) # turn off boundscheck for this function
def f():
    ...
    # turn it temporarily on again for this block
    with cython.boundscheck(True):
        ...

```

警告

这两种设置指令的方法是**而不是**，它们使用-X 选项覆盖命令行上的指令。

#### 在`setup.py` 中

通过将关键字参数传递给`cythonize`，也可以在`setup.py`文件中设置编译器指令：

```py
from distutils.core import setup
from Cython.Build import cythonize

setup(
    name="My hello app",
    ext_modules=cythonize('hello.pyx', compiler_directives={'embedsignature': True}),
)

```

这将覆盖`compiler_directives`字典中指定的默认指令。请注意，如上所述的显式每文件或本地指令优先于传递给`cythonize`的值。