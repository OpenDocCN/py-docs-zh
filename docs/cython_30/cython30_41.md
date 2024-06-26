# Pythran 作为 Numpy 后端

> 原文： [`docs.cython.org/en/latest/src/userguide/numpy_pythran.html`](http://docs.cython.org/en/latest/src/userguide/numpy_pythran.html)

使用标志`--np-pythran`，可以将 [Pythran](https://github.com/serge-sans-paille/pythran) numpy 实现用于与 numpy 相关的操作。使用此后端的一个优点是 Pythran 实现使用 C ++表达式模板来节省内存传输，并且可以受益于现代 CPU 的 SIMD 指令。

在某些情况下，这可以带来非常有趣的加速，从 2 到 16，具体取决于目标 CPU 架构和原始算法。

请注意，此功能是实验性的。

## 使用 distutils 的用法示例

你首先需要安装 Pythran。有关更多信息，请参见其[文档](https://pythran.readthedocs.io/)。

然后，只需在 Python 文件的顶部添加一个`cython: np_pythran=True`指令，该指令需要使用 Pythran numpy 支持进行编译。

以下是使用 distutils 的简单`setup.py`文件的示例：

```py
from distutils.core import setup
from Cython.Build import cythonize

setup(
    name = "My hello app",
    ext_modules = cythonize('hello_pythran.pyx')
)

```

然后，使用`hello_pythran.pyx`中的以下标题：

```py
# cython: np_pythran=True

```

`hello_pythran.pyx`将使用 Pythran numpy 支持编译。

请注意，可以通过在`$HOME/.pythranrc`文件中添加设置来进一步调整 Pythran。例如，这可以用于启用 [Boost.SIMD](https://github.com/NumScale/boost.simd) 支持。有关更多信息，请参见 [Pythran 用户手册](https://pythran.readthedocs.io/en/latest/MANUAL.html#customizing-your-pythranrc)。