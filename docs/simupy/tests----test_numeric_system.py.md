# `.\simupy\tests\test_numeric_system.py`

```
# 导入 pytest 库，用于编写和运行测试用例
import pytest
# 导入 numpy 库，并使用 np 别名
import numpy as np
# 导入 numpy.testing 模块，并使用 npt 别名
import numpy.testing as npt
# 从 simupy.systems 模块中导入指定内容：DynamicalSystem 类、need_state_equation_function_msg、need_output_equation_function_msg、zero_dim_output_msg
from simupy.systems import (DynamicalSystem, need_state_equation_function_msg,
                            need_output_equation_function_msg,
                            zero_dim_output_msg)

# 定义常量 N，值为 4
N = 4

# 定义一个函数，返回一个形状与输入 x 相同的全为 1 的 numpy 数组
def ones_equation_function(t, x):
    return np.ones(x.shape)

# 定义测试函数，验证在状态维度为 0 时是否会抛出 ValueError 异常，并匹配 zero_dim_output_msg 提示信息
def test_dim_output_0():
    with pytest.raises(ValueError, match=zero_dim_output_msg):
        DynamicalSystem(dim_state=0)

# 定义测试函数，验证在状态方程函数未提供时是否会抛出 ValueError 异常，并匹配 need_state_equation_function_msg 提示信息
def test_state_equation_function_kwarg():
    with pytest.raises(ValueError, match=need_state_equation_function_msg):
        DynamicalSystem(dim_state=N)
    # 生成一个长度为 N+1 的随机数组
    args = np.random.rand(N+1)
    # 创建一个 DynamicalSystem 对象，指定状态维度为 N，并提供状态方程函数为 ones_equation_function
    sys = DynamicalSystem(dim_state=N,
                          state_equation_function=ones_equation_function)
    # 断言状态方程函数的输出与全为 1 的数组形状相同
    npt.assert_allclose(
        sys.state_equation_function(args[0], args[1:]),
        np.ones(N)
    )

# 定义测试函数，验证在输出方程函数未提供时是否会抛出 ValueError 异常，并匹配 need_output_equation_function_msg 提示信息
def test_output_equation_function_kwarg():
    with pytest.raises(ValueError, match=need_output_equation_function_msg):
        DynamicalSystem(dim_output=N)

    args = np.random.rand(N+1)

    # 创建一个 DynamicalSystem 对象，指定状态维度为 N，并提供状态方程函数为 ones_equation_function
    sys = DynamicalSystem(dim_state=N,
                          state_equation_function=ones_equation_function)
    # 断言输出方程函数的输出与输入参数的后 N 个元素相同
    npt.assert_allclose(
        sys.output_equation_function(args[0], args[1:]),
        args[1:]
    )

    # 创建一个 DynamicalSystem 对象，指定状态维度为 1，并提供状态方程函数与输出方程函数均为 ones_equation_function
    sys = DynamicalSystem(dim_state=1,
                          state_equation_function=ones_equation_function,
                          output_equation_function=ones_equation_function)
    # 断言输出方程函数的输出与全为 1 的数组形状相同
    npt.assert_allclose(
        sys.output_equation_function(args[0], args[1:]),
        np.ones(N)
    )

# 定义测试函数，验证初始条件是否正确设置
def test_initial_condition_kwarg():
    # 创建一个 DynamicalSystem 对象，指定状态维度为 N，并提供状态方程函数为 ones_equation_function
    sys = DynamicalSystem(dim_state=N,
                          state_equation_function=ones_equation_function)
    # 断言初始条件为全为 0 的数组
    npt.assert_allclose(sys.initial_condition, np.zeros(N))
    # 创建一个 DynamicalSystem 对象，指定状态维度为 N，并提供状态方程函数为 ones_equation_function，初始条件为全为 1 的数组
    sys = DynamicalSystem(dim_state=N,
                          initial_condition=np.ones(N),
                          state_equation_function=ones_equation_function)
    # 断言初始条件为全为 1 的数组
    npt.assert_allclose(sys.initial_condition, np.ones(N))
```