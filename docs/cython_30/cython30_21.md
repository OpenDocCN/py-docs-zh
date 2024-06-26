# 相关工作

> 原文： [`docs.cython.org/en/latest/src/tutorial/related_work.html`](http://docs.cython.org/en/latest/src/tutorial/related_work.html)

Pyrex [[Pyrex]](../quickstart/overview.html#pyrex) 是 Cython 最初基于的编译器项目。 Cython 语言的许多功能和主要设计决策由 Greg Ewing 开发，作为该项目的一部分。今天，Cython 通过提供与 Python 代码和 Python 语义的更高兼容性，以及优秀的优化和与 NumPy 等科学 Python 扩展的更好集成，取代了 Pyrex 的功能。

ctypes [[ctypes]](#ctypes) 是 Python 的外部函数接口（FFI）。它提供 C 兼容的数据类型，并允许在 DLL 或共享库中调用函数。它可以用于在纯 Python 代码中包装这些库。与 Cython 相比，它具有主要优势，即可以在标准库中直接从 Python 代码中使用，而无需任何其他依赖项。主要缺点是它的性能受到 Python 调用开销的影响，因为所有操作必须首先通过 Python 代码。 Cython 作为一种编译语言，可以通过将更多功能和长时间运行的循环转换为快速 C 代码来避免大量此类开销。

SWIG [[SWIG]](#swig) 是一个包装器代码生成器。它使得在 C / C ++头文件中解析大型 API 定义变得非常容易，并为大量编程语言生成直接的包装器代码。然而，与 Cython 相反，它本身并不是一种编程语言。薄包装器很容易生成，但包装器需要提供的功能越多，使用 SWIG 实现它就越困难。另一方面，Cython 使得为 Python 语言编写非常精细的包装代码变得非常容易，并且可以根据需要在任何给定的位置使其变薄或变厚。此外，存在用于解析 C 头文件并使用它来生成 Cython 定义和模块骨架的第三方代码。

ShedSkin [[ShedSkin]](#shedskin) 是一个实验性的 Python-to-C ++编译器。它使用非常强大的整个模块类型推理引擎从（受限制的）Python 源代码生成 C ++程序。主要缺点是它不支持为本机不支持的操作调用 Python / C API，并且支持很少的标准 Python 模块。

<colgroup><col class="label"><col></colgroup>
| [[ctypes]](#id2) | [`docs.python.org/library/ctypes.html`](https://docs.python.org/library/ctypes.html) 。 |

<colgroup><col class="label"><col></colgroup>
| [[ShedSkin]](#id4) | M. Dufour，J。Coughlan，ShedSkin， [`github.com/shedskin/shedskin`](https://github.com/shedskin/shedskin) |

<colgroup><col class="label"><col></colgroup>
| [[SWIG]](#id3) | David M. Beazley 等，SWIG：一种易于使用的工具，用于将脚本语言与 C 和 C ++集成， [`www.swig.org`](http://www.swig.org) 。 |