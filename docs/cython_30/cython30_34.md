# Limitations

> 原文： [`docs.cython.org/en/latest/src/userguide/limitations.html`](http://docs.cython.org/en/latest/src/userguide/limitations.html)

此页面用于列出 Cython 中的错误，这些错误使得编译代码的语义与 Python 中的语义不同。大多数缺失的功能已在 Cython 0.15 中修复。未来的 Cython 版本计划提供完整的 Python 语言兼容性。目前，问题跟踪器可以提供我们知道并希望修复的偏差的概述。

[`github.com/cython/cython/labels/Python%20Semantics`](https://github.com/cython/cython/labels/Python%20Semantics)

以下是我们可能无法解决的差异列表。大多数这些事情更多地落入实现细节而不是语义，我们可能决定不修复（或需要一个-pedantic 标志来获取）。

## 嵌套元组参数解包

```py
def f((a,b), c):
    pass

```

这在 Python 3 中被删除了。

## 检查支持

虽然很有可能在 Cython 自己的函数类型中模拟函数的接口，并且最近的 Cython 版本在这里看到了一些改进，但“inspect”模块并没有将 Cython 实现的函数视为“函数”，因为它测试了对象类型显式而不是比较抽象接口或抽象基类。这对使用 inspect 来检查函数对象的代码有负面影响，但是需要对 Python 本身进行更改。

## 堆栈帧

目前，我们生成假追踪作为异常传播的一部分，但不填写本地并且无法填写 co_code。为了完全兼容，我们必须在函数调用时生成这些堆栈帧对象（具有潜在的性能损失）。我们可以选择启用此功能进行调试。

## 推断文字的身份与平等

```py
a = 1.0          # a inferred to be C type 'double'
b = c = None     # b and c inferred to be type 'object'
if some_runtime_expression:
    b = a        # creates a new Python float object
    c = a        # creates a new Python float object
print(b is c)     # most likely not the same object

```