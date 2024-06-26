# Cython - 概述

> 原文： [`docs.cython.org/en/latest/src/quickstart/overview.html`](http://docs.cython.org/en/latest/src/quickstart/overview.html)

[[Cython]](#cython) 是一种编程语言，它使 Python 语言的 C 语言扩展与 Python 本身一样简单。它旨在成为 [[Python]](#python) 语言的超集，为其提供高级，面向对象，函数式和动态编程。它的主要特性是支持任意的静态类型声明作为语言的一部分。源代码被转换为优化过的 C / C ++代码并编译为 Python 扩展模块。这一特性使得程序可以执行非常快速并且能与外部 C 语言库的紧密集成，同时还能是程序员高能保持众所周知的 Python 语言开发效率。

主要的 Python 执行环境通常被称为 CPython，因为它是用 C 语言编写的。其他主要实现使用 Java（Jython [[Jython]](#jython) ），C＃（IronPython [[IronPython]](#ironpython) ）和 Python 本身（PyPy [[PyPy]](#pypy) ）。用 C 和 CPython 编程有助于包装许多通过 C 语言提供接口的外部库。然而，即是是需要在 C 中编写必要的胶水代码这仍然是值得的，特别是对于那些只熟悉 Python 这样的高级语言的程序员而不熟悉像 C 这样的接近底层的语言。

最初基于着名的 Pyrex [[Pyrex]](#pyrex) ，Cython 项目通过源代码编译器将 Python 代码转换为等效的 C 代码来解决这个问题。此代码在 CPython 运行时环境中执行，但是却以编译后的 C 程序那般速度执行，并且能够直接调用 C 语言库。同时，它保留了 Python 源代码的原始接口，这使得它可以直接使用 Python 语言代码。这些双重特性使 Cython 的这两个主要使用场景成为可能：使用快速二进制模块来扩展 CPython 解释器，以及将 Python 代码与外部 C 库连接。

与此同时 Cython 可以编译（大多数）常规 Python 代码，而且生成的 C 代码通常可以从 Python 和 C 类型的任意静态类型声明中获得主要（并且有时很惊人）的速度上的提升。这些允许 Cython 将 C 语义分配给代码的一部分，并将它们转换为非常高效率的 C 代码。因此，类型声明可用于两个目的：将代码段从动态 Python 语义转换为静态和快速 C 语义，还用于直接操作外部库中定义的类型。因此，Cython 将这两个世界合并为一种非常广泛适用的编程语言。

> [Cython]](#id1) | G. Ewing，R。W. Bradshaw，S。Behnel，D。S. Seljebotn 等人，Cython 编译器， [`cython.org/`](https://cython.org/) 。
> [[IronPython]](#id4) | Jim Hugunin 等人， [`archive.codeplex.com/?p=IronPython`](https://archive.codeplex.com/?p=IronPython) 。 
> [[Jython]](#id3) | J. Huginin，B。Warsaw，F.Bock，et al。，Jython：Python for the Java platform， [`www.jython.org`](http://www.jython.org) 。 
> [[PyPy]](#id5) | PyPy Group，PyPy：用 Python 编写的 Python 实现， [`pypy.org/`](https://pypy.org/) 。
> [[派热克斯]](#id6) | G. Ewing，Pyrex：Python 的 C-Extensions， [`www.cosc.canterbury.ac.nz/greg.ewing/python/Pyrex/`](https://www.cosc.canterbury.ac.nz/greg.ewing/python/Pyrex/) 
> [[Python]](#id2) | G. van Rossum 等人，Python 编程语言， [`www.python.org/`](https://www.python.org/) 。