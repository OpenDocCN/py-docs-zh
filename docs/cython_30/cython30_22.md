# 附录：在 Windows 上安装 MinGW

> 原文： [`docs.cython.org/en/latest/src/tutorial/appendix.html`](http://docs.cython.org/en/latest/src/tutorial/appendix.html)

> 1.  从 [`www.mingw.org/wiki/HOWTO_Install_the_MinGW_GCC_Compiler_Suite`](http://www.mingw.org/wiki/HOWTO_Install_the_MinGW_GCC_Compiler_Suite) 下载 MinGW 安装程序。 （截至撰写本文时，下载链接有点难以找到;它位于左侧菜单中的“关于”下）。您需要名为“Automated MinGW Installer”（当前版本为 5.1.4）的文件。
>     
>     
> 2.  运行它并安装 MinGW。 Cython 只需要基本的包，尽管你可能也想要至少抓住 C ++编译器。
>     
>     
> 3.  您需要设置 Windows 的“PATH”环境变量，以便包括例如“c：\ mingw \ bin”（如果您将 MinGW 安装到“c：\ mingw”）。以下 Web 页面描述了 Windows XP 中的过程（Vista 过程类似）： [`support.microsoft.com/kb/310519`](https://support.microsoft.com/kb/310519)
>     
>     
> 4.  最后，告诉 Python 使用 MinGW 作为默认编译器（否则它将尝试 Visual C）。如果将 Python 安装到“c：\ Python27”，则创建一个名为“c：\ Python27 \ Lib \ distutils \ distutils.cfg”的文件，其中包含：
>     
>     
>     
>     ```py
>     [build]
>     compiler = mingw32
>     
>     ```

[[WinInst]](#wininst) wiki 页面包含有关此过程的更新信息。欢迎任何有助于使 Windows 安装过程更顺畅的贡献;一个不幸的事实是，没有一个普通的 Cython 开发人员可以方便地访问 Windows。

<colgroup><col class="label"><col></colgroup>
| [[WinInst]](#id1) | [`github.com/cython/cython/wiki/CythonExtensionsOnWindows`](https://github.com/cython/cython/wiki/CythonExtensionsOnWindows) |