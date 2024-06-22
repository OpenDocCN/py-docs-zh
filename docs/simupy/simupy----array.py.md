# `.\simupy\simupy\array.py`

```
# 从 sympy.tensor.array 导入 Array 类
from sympy.tensor.array import Array
# 从 sympy 导入 ImmutableDenseMatrix 并起别名为 Matrix
from sympy import ImmutableDenseMatrix as Matrix
# 从 numpy.lib.index_tricks 导入 RClass、CClass、AxisConcatenator 三个类
from numpy.lib.index_tricks import RClass, CClass, AxisConcatenator

# 定义一个 Mix-in 类 SymAxisConcatenatorMixin，用于将 numpy 的 AxisConcatenator 类转换为 sympy 的 N-D 数组使用
class SymAxisConcatenatorMixin:
    """
    A mix-in to convert numpy AxisConcatenator classes to use with sympy N-D
    arrays.
    """

    # 定义静态方法 concatenate，使用 sympy 的 Array 对象封装 numpy 的 AxisConcatenator.concatenate 方法
    concatenate = staticmethod(
        lambda *args, **kwargs: Array(
            AxisConcatenator.concatenate(*args, **kwargs)
        )
    )
    # 定义静态方法 makemat，使用 sympy 的 ImmutableDenseMatrix 封装 Matrix 类
    makemat = staticmethod(Matrix)

    # 定义 _retval 方法，根据条件选择返回 Matrix 或 Array 对象的实例化结果
    def _retval(self, res):  # support numpy < 1.13
        if self.matrix:
            cls = Matrix
        else:
            cls = Array
        return cls(super()._retval(res))

    # 重载 __getitem__ 方法，根据 key 的类型选择相应的处理方式
    def __getitem__(self, key):
        return super().__getitem__(tuple(
            k if isinstance(k, str) else
            Array(k) if hasattr(k, '__len__')
            else Array([k])
            for k in key
        ))


# 定义 SymRClass 类，继承自 SymAxisConcatenatorMixin 和 numpy 的 RClass 类
class SymRClass(SymAxisConcatenatorMixin, RClass):
    pass


# 定义 SymCClass 类，继承自 SymAxisConcatenatorMixin 和 numpy 的 CClass 类
class SymCClass(SymAxisConcatenatorMixin, CClass):
    pass


# 创建 SymRClass 类的实例 r_
r_ = SymRClass()
# 创建 SymCClass 类的实例 c_
c_ = SymCClass()


def empty_array():
    """
    Construct an empty array, which is often needed as a place-holder
    """
    # 创建一个包含单个元素 0 的 Array 对象 a
    a = Array([0])
    # 设置 a 的形状为空元组
    a._shape = tuple()
    # 设置 a 的秩为 0
    a._rank = 0
    # 设置 a 的循环大小为 0
    a._loop_size = 0
    # 设置 a 的数组为一个空列表
    a._array = []
    # 定义 a 的字符串表示为一个空数组的字符串
    a.__str__ = lambda *args, **kwargs: "[]"
    # 返回构造好的 Array 对象 a
    return a
```