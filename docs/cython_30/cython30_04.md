# 安装 Cython

> 原文： [`docs.cython.org/en/latest/src/quickstart/install.html`](http://docs.cython.org/en/latest/src/quickstart/install.html)

许多学术性 Python 发行版，例如 Anaconda [[Anaconda]](#anaconda) ，Enthought Canopy [[Canopy]](#canopy) 和 Sage [[Sage]](#sage) ，都自带有 Cython 并且不需要设置。但请注意，如果您的发行版发布的 Cython 版本太旧，您仍然可以使用下面的说明更新 Cython。除非脚注另有说明，否则本教程中的所有内容都应与 Cython 0.11.2 及更高版本一起使用。

与大多数 Python 软件不同，Cython 需要在系统上存在 C 编译器。获取 C 编译器的细节因使用的系统而异：

> *   **Linux** 通常自带 GNU C 编译器（gcc），或通过包系统轻松获得。例如，在 Ubuntu 或 Debian 上，输入命令`sudo apt-get install build-essential`将获取您需要的所有内容。
> *   **Mac OS X** 要检索 gcc，一个选项是安装 Apple 的 XCode，可以从 Mac OS X 的安装 DVD 或 [`developer.apple.com /`](https://developer.apple.com/) 获得。
> *   **Windows** 一个流行的选择是使用开源 MinGW（Windows 的 gcc 分发版）。有关手动设置 MinGW 的说明，请参阅附录.Enthought Canopy 和 Python（x，y）捆绑 MinGW，另一个选择是使用 Microsoft 的 Visual C.然后必须使用与编译安装的 Python 相同的版本。

安装 Cython 的最简单方法是使用`pip`：

```py
pip install Cython
```

最新的 Cython 版本始终可以从 [`cython.org/`](https://cython.org/) 下载。解压缩 tarball 或 zip 文件，输入目录，然后运行：

```py
python setup.py install
```

对于一次性构建，例如对于 CI /测试。当所在平台 PyPI 并没有提供轮子包（wheel package）时。安装未编译（较慢）的 Cython 版本比编译整个源代码来安装要快得多。安装且不编译 Cython 的命令：

```py
pip install Cython --install-option="--no-cython-compile"
```

> [[python]](#id1)  [`docs.anaconda.com/anaconda/`](https://docs.anaconda.com/anaconda/) 
>  [[Canopy]](#id2)  [`www.enthought.com/product/canopy/`](https://www.enthought.com/product/canopy/) 
>  [[Sage]](#id3) W. Stein 等，Sage Mathematics Software， [`www.sagemath.org/`](https://www.sagemath.org/)

 