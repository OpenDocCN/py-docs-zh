# `.\simupy\tests\test_symbolic_system.py`

```
# 导入 pytest 库，用于单元测试
import pytest
# 导入 numpy 库，并使用别名 np
import numpy as np
# 导入 sympy 库，并使用别名 sp
import sympy as sp
# 导入 numpy.testing 模块中的 npt 别名，用于测试 numpy 数组
import numpy.testing as npt
# 从 simupy.systems.symbolic 模块导入 DynamicalSystem 和 dynamicsymbols
from simupy.systems.symbolic import DynamicalSystem, dynamicsymbols
# 从 simupy.systems 模块导入 need_state_equation_function_msg 和 zero_dim_output_msg
from simupy.systems import (need_state_equation_function_msg,
                            zero_dim_output_msg)
# 从 simupy.array 模块导入 Array 和 r_
from simupy.array import Array, r_

# 定义动力系统的状态变量 x，包括 x1 和 x2
x = x1, x2 = Array(dynamicsymbols('x1:3'))
# 定义符号常量 mu
mu = sp.symbols('mu')
# 定义状态方程，使用 r_ 函数将 x2 和 (-x1+mu*(1-x1**2)*x2) 组合成数组
state_equation = r_[x2, -x1+mu*(1-x1**2)*x2]
# 定义输出方程，使用 r_ 函数将 x1**2 + x2**2 和 sp.atan2(x2, x1) 组合成数组
output_equation = r_[x1**2 + x2**2, sp.atan2(x2, x1)]
# 定义常量字典，将 mu 映射到值 5
constants = {mu: 5}

# 定义测试函数 test_dim_output_0，测试输出维度是否为零的情况
def test_dim_output_0():
    # 使用 pytest 来断言是否会引发 ValueError 异常，并匹配 zero_dim_output_msg 消息
    with pytest.raises(ValueError, match=zero_dim_output_msg):
        DynamicalSystem(input_=x, constants_values=constants)

# 定义测试函数 test_state_equation_kwarg，测试状态方程是否为关键字参数的情况
def test_state_equation_kwarg():
    # 使用 pytest 来断言是否会引发 ValueError 异常，并匹配 need_state_equation_function_msg 消息
    with pytest.raises(ValueError, match=need_state_equation_function_msg):
        DynamicalSystem(state=x, constants_values=constants)
    # 创建动力系统对象 sys，传入状态变量 x、状态方程 state_equation 和常量字典 constants
    sys = DynamicalSystem(state=x,
                          state_equation=state_equation,
                          constants_values=constants)
    # 生成随机数组 args，长度为状态变量 x 的长度加一
    args = np.random.rand(len(x)+1)
    # 使用 npt.assert_allclose 来断言状态方程函数的计算结果是否与预期值一致
    npt.assert_allclose(
        sys.state_equation_function(args[0], args[1:]).squeeze(),
        np.r_[args[2], -args[1]+constants[mu]*(1-args[1]**2)*args[2]]
    )

# 定义测试函数 test_output_equation_function_kwarg，测试输出方程是否为关键字参数的情况
def test_output_equation_function_kwarg():
    # 使用 pytest 来断言是否会引发 ValueError 异常，并匹配 zero_dim_output_msg 消息
    with pytest.raises(ValueError, match=zero_dim_output_msg):
        DynamicalSystem(input_=x)
    # 生成随机数组 args，长度为状态变量 x 的长度加一
    args = np.random.rand(len(x)+1)
    # 创建动力系统对象 sys，传入状态变量 x、状态方程 state_equation 和常量字典 constants
    sys = DynamicalSystem(state=x,
                          state_equation=state_equation,
                          constants_values=constants)
    # 使用 npt.assert_allclose 来断言输出方程函数的计算结果是否与预期值一致
    npt.assert_allclose(
        sys.output_equation_function(args[0], args[1:]).squeeze(),
        args[1:]
    )
    # 创建动力系统对象 sys，传入状态变量 x、状态方程 state_equation、输出方程 output_equation 和常量字典 constants
    sys = DynamicalSystem(state=x,
                          state_equation=state_equation,
                          output_equation=output_equation,
                          constants_values=constants)
    # 使用 npt.assert_allclose 来断言输出方程函数的计算结果是否与预期值一致
    npt.assert_allclose(
        sys.output_equation_function(args[0], args[1:]).squeeze(),
        np.r_[args[1]**2 + args[2]**2, np.arctan2(args[2], args[1])]
    )
```