# Caveats

> 原文： [`docs.cython.org/en/latest/src/tutorial/caveats.html`](http://docs.cython.org/en/latest/src/tutorial/caveats.html)

由于 Cython 混合了 C 语言和 Python 语义，因此有些事情可能会有点令人惊讶或不直观。对于 Python 用户来说，工作总是让 Cython 更自然，因此这个列表将来可能会发生变化。

> *   给定两个类型`int`变量`a`和`b`，`a % b`与第二个参数（遵循 Python 语义）具有相同的符号，而不是与第一个符号相同（如在 C）。通过启用 cdivision 指令（Cython 0.12 之前的版本始终遵循 C 语义），可以在某种速度增益下获得 C 行为。
> *   无条件类型需要小心。`cdef unsigned n = 10; print(range(-n, n))`将打印一个空列表，因为`-n`在传递给`range`函数之前回绕到一个大的正整数。
> *   Python 的`float`类型实际上包含了 C `double`值，而 Python 2.x 中的`int`类型包含了 C `long`值。