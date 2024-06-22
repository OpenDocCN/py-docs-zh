# 将 Cython 代码移植到 PyPy

> 原文： [`docs.cython.org/en/latest/src/userguide/pypy.html`](http://docs.cython.org/en/latest/src/userguide/pypy.html)

Cython 对 cpyext 有基本支持，cpyext 是 [PyPy](https://pypy.org/) 中模拟 CPython 的 C-API 的层。这是通过使生成的 C 代码在 C 编译时适应来实现的，因此生成的代码将在 CPython 和 PyPy 中编译都不变。

但是，除了 Cython 可以在内部覆盖和调整之外，cpyext C-API 仿真还涉及到 CPython 中对用户代码有明显影响的真实 C-API 的一些差异。此页面列出了主要的差异以及处理它们的方法，以便编写在 CPython 和 PyPy 中都有效的 Cython 代码。

## 参考计数

PyPy 的一般设计差异是运行时内部不使用引用计数，但始终是垃圾收集器。仅通过计算 C 空间中保存的引用，仅在 cpyext 层模拟引用计数。这意味着 PyPy 中的引用计数通常与 CPython 中的引用计数不同，因为它不计算 Python 空间中保存的任何引用。

## 对象寿命

作为不同垃圾收集特征的直接结果，对象可能会在 CPython 之外的其他点看到它们的生命周期结束。因此，当预期物体在 CPython 中死亡但在 PyPy 中可能没有时，必须特别小心。具体来说，扩展类型（`__dealloc__()`）的解除分配器方法可能会在比 CPython 更晚的时间点被调用，而是由内存变得比被死的对象更紧密地触发。

如果代码中的点在某个对象应该死亡时是已知的（例如，当它与另一个对象或某个函数的执行时间相关联时），那么值得考虑它是否可以无效并在此时手动清理而不是依赖于解除分配器。

作为副作用，这有时甚至可以导致更好的代码设计，例如，当上下文管理器可以与`with`语句一起使用时。

## 借用的引用和数据指针

PyPy 中的内存管理允许在内存中移动对象。 C-API 层只是 PyPy 对象的间接视图，通常将数据或状态复制到 C 空间，然后绑定到 C-API 对象的生命周期，而不是底层的 PyPy 对象。重要的是要理解这两个对象在 cpyext 中是不同的东西。

效果可能是当使用数据指针或借用引用，并且不再直接从 C 空间引用拥有对象时，引用或数据指针在某些时候可能变得无效，即使对象本身仍然存在。与 CPython 相反，仅仅在列表（或其他 Python 容器）中保持对对象的引用是不够的，因为它们的内容仅在 Python 空间中管理，因此仅引用 PyPy 对象。 Python 容器中的引用不会使 C-API 视图保持活动状态。 Python 类 dict 中的条目显然也不起作用。

可能发生这种情况的一个更明显的地方是访问字节字符串的`char*`缓冲区。在 PyPy 中，只有在 Cython 代码持有对字节字符串对象本身的直接引用时，这才会起作用。

另一点是当直接使用 CPython C-API 函数返回借用的引用时，例如， [`PyTuple_GET_ITEM()`](https://docs.python.org/3/c-api/tuple.html#c.PyTuple_GET_ITEM "(in Python v3.7)") 和类似的函数，但也有一些函数返回对内置模块或运行时环境的低级对象的借用引用。 PyPy 中的 GIL 只保证借用的引用在下次调用 PyPy（或其 C-API）时保持有效，但不一定更长。

当访问 Python 对象的内部或使用借用的引用时间长于下一次调用 PyPy 时，包括引用计数或释放 GIL 的任何东西，因此需要在 C 空间中另外保持对这些对象的直接拥有引用，例如，在函数中的局部变量或扩展类型的属性中。

如有疑问，请避免使用返回借用引用的 C-API 函数，或者在获取引用和 [`Py_DECREF()`时通过对](https://docs.python.org/3/c-api/refcounting.html#c.Py_DECREF "(in Python v3.7)") [`Py_INCREF()`](https://docs.python.org/3/c-api/refcounting.html#c.Py_INCREF "(in Python v3.7)") 的一对调用显式包围借用引用的使用完成后将其转换为自己的引用。

## 内置类型，插槽和字段

以下内置类型目前在 cpyext 中不以 C 级表示形式提供： [`PyComplexObject`](https://docs.python.org/3/c-api/complex.html#c.PyComplexObject "(in Python v3.7)") ， [`PyFloatObject`](https://docs.python.org/3/c-api/float.html#c.PyFloatObject "(in Python v3.7)") 和`PyBoolObject`。

内置类型的许多类型槽函数未在 cpyext 中初始化，因此不能直接使用。

类似地，几乎没有内置类型的（实现）特定结构域在 C 级暴露，例如 [`PyLongObject`](https://docs.python.org/3/c-api/long.html#c.PyLongObject "(in Python v3.7)") 的`ob_digit`字段或[的`allocated`字段。 `PyListObject`](https://docs.python.org/3/c-api/list.html#c.PyListObject "(in Python v3.7)") struct 等虽然容器的`ob_size`字段（`Py_SIZE()`宏使用）可用，但不保证准确。

最好不要访问任何这些结构域和插槽，而是使用普通的 Python 类型以及对象操作的普通 Python 协议。 Cython 会将它们映射到 CPython 和 cpyext 中 C-API 的适当用法。

## GIL 处理

目前，GIL 处理函数 [`PyGILState_Ensure()`](https://docs.python.org/3/c-api/init.html#c.PyGILState_Ensure "(in Python v3.7)") 在 PyPy 中不可重入，并且在被调用两次时死锁。这意味着试图获取 GIL“以防万一”的代码，因为它可能在有或没有 GIL 的情况下被调用，但在 PyPy 中不会按预期工作。如果 GIL 已经持有，请参见 [PyGILState_Ensure 不应该死锁。](https://bitbucket.org/pypy/pypy/issues/1778)

## 效率

简单的函数，尤其是用于 CPython 速度的宏，可能在 cpyext 中表现出截然不同的性能特征。

返回借用引用的函数已被提及为需要特别小心，但它们也会导致更多的运行时开销，因为它们经常在 PyPy 中创建弱引用，它们只返回 CPython 中的普通指针。可见的例子是 [`PyTuple_GET_ITEM()`](https://docs.python.org/3/c-api/tuple.html#c.PyTuple_GET_ITEM "(in Python v3.7)") 。

一些更高级别的功能也可能表现出完全不同的性能特征，例如， [`PyDict_Next()`](https://docs.python.org/3/c-api/dict.html#c.PyDict_Next "(in Python v3.7)") 用于 dict 迭代。虽然它是在 CPython 中迭代 dict 的最快方法，具有线性时间复杂度和低开销，但它目前在 PyPy 中具有二次运行时因为它映射到正常的 dict 迭代，它无法跟踪两个调用之间的当前位置，因此需要在每次调用时重新启动迭代。

这里的一般建议比 CPython 更适用，最好依靠 Cython 为您生成适当的 C-API 处理代码，而不是直接使用 C-API - 除非您真的知道自己在做什么。如果你发现在 PyPy 和 cpyext 中做一些比 Cython 目前做得更好的方法，最好修复 Cython 以获得每个人的好处。

## 已知问题

*   从 PyPy 1.9 开始，在极少数情况下，子类型内置类型会导致方法调用的无限递归。
*   特殊方法的 Docstrings 不会传播到 Python 空间。
*   pypy3 中的 Python 3.x 改编只是慢慢开始包含 C-API，因此可以预期更多的不兼容性。

## 错误和崩溃

PyPy 中的 cpyext 实现比经过充分测试的 C-API 及其在 CPython 中的底层本机实现要年轻得多且不太成熟。遇到崩溃时应记住这一点，因为问题可能并不总是存在于您的代码或 Cython 中。此外，PyPy 及其 cpyext 实现在 C 级别比 CPython 和 Cython 更容易调试，仅仅因为它们不是为它而设计的。