# `.\simupy\tests\test_numeric_switchedsystem.py`

```
# 导入pytest模块，用于编写和运行测试
import pytest
# 导入numpy库，并使用别名np
import numpy as np
# 导入numpy.testing模块，并使用别名npt
import numpy.testing as npt
# 导入simupy.systems模块中的特定函数和类
from simupy.systems import (SwitchedSystem, need_state_equation_function_msg,
                            need_output_equation_function_msg,
                            zero_dim_output_msg, full_state_output)

# 定义最大条件数目为4
max_n_condition = 4
# 定义边界的最小值为-1
bounds_min = -1
# 定义边界的最大值为1
bounds_max = 1

# 定义状态方程函数，返回与x相同形状的全1数组
def state_equation_function(t, x, u):
    return np.ones(x.shape)

# 定义输出方程函数，返回与u相同形状的全1数组
def output_equation_function(t, u):
    return np.ones(u.shape)

# 定义事件变量方程函数，返回输入x
def event_variable_equation_function(t, x):
    return x

# 定义pytest的fixture，用于生成测试用例的数据参数化
@pytest.fixture(scope="module", params=np.arange(2, max_n_condition))
def switch_fixture(request):
    yield (((bounds_max - bounds_min)*np.sort(np.random.rand(request.param-1))
           + bounds_min),
           # 生成数组，每个元素为一个lambda函数，返回与x相同形状的cnd*1数组
           np.array([lambda t, x, u: cnd*np.ones(x.shape)
                     for cnd in range(request.param)]),
           # 生成数组，每个元素为一个lambda函数，返回与u相同形状的cnd*1数组
           np.array([lambda t, u: cnd*np.ones(u.shape)
                     for cnd in range(request.param)])
           )

# 定义测试函数，测试dim_output为0的情况
def test_dim_output_0(switch_fixture):
    # 使用pytest的raises方法，断言抛出ValueError异常并匹配zero_dim_output_msg
    with pytest.raises(ValueError, match=zero_dim_output_msg):
        SwitchedSystem(
            event_variable_equation_function=event_variable_equation_function,
            event_bounds=switch_fixture[0],
            output_equations_functions=switch_fixture[2]
        )
    # 实例化SwitchedSystem对象，指定dim_output为1
    SwitchedSystem(
        dim_output=1,
        event_variable_equation_function=event_variable_equation_function,
        event_bounds=switch_fixture[0],
        output_equations_functions=switch_fixture[2],
    )

# 定义测试函数，测试state_equations_functions关键字参数
def test_state_equations_functions_kwarg(switch_fixture):
    # 使用pytest的raises方法，断言抛出ValueError异常并匹配need_state_equation_function_msg
    with pytest.raises(ValueError, match=need_state_equation_function_msg):
        SwitchedSystem(
            dim_state=1,
            event_variable_equation_function=event_variable_equation_function,
            event_bounds=switch_fixture[0],
            output_equations_functions=switch_fixture[2]
        )
    # 使用pytest的raises方法，断言抛出匹配字符串"broadcast"的ValueError异常
    with pytest.raises(ValueError, match="broadcast"):
        SwitchedSystem(
            dim_state=1,
            event_variable_equation_function=event_variable_equation_function,
            event_bounds=switch_fixture[0],
            # 生成数组，每个元素为一个lambda函数，返回与x相同形状的cnd*1数组
            state_equations_functions=np.array([
                     lambda t, x, u: cnd*np.ones(x.shape)
                     for cnd in range(switch_fixture[0].size+2)
             ]),
            output_equations_functions=switch_fixture[2]
        )
    # 实例化SwitchedSystem对象，验证state_equations_functions和output_equations_functions参数
    sys = SwitchedSystem(
        dim_state=1,
        event_variable_equation_function=event_variable_equation_function,
        event_bounds=switch_fixture[0],
        state_equations_functions=switch_fixture[1],
        output_equations_functions=switch_fixture[2]
    )
    # 使用numpy.testing的assert_array_equal方法，断言两个数组相等
    npt.assert_array_equal(sys.state_equations_functions, switch_fixture[1])
    # 实例化SwitchedSystem对象，验证state_equations_functions和output_equations_functions参数
    sys = SwitchedSystem(
        dim_state=1,
        event_variable_equation_function=event_variable_equation_function,
        event_bounds=switch_fixture[0],
        state_equations_functions=state_equation_function,
        output_equations_functions=switch_fixture[2]
    )
    # 使用 NumPy 测试工具库进行数组相等性断言，检查 sys 对象的 state_equations_functions 属性是否与 state_equation_function 相等。
    npt.assert_array_equal(
        sys.state_equations_functions,
        state_equation_function
    )
def test_output_equations_functions_kwarg(switch_fixture):
    # 使用 pytest 的断言检查是否会抛出 ValueError 异常，并匹配指定的错误信息
    with pytest.raises(ValueError, match=need_output_equation_function_msg):
        # 创建 SwitchedSystem 对象，设定 dim_output=1，事件变量方程函数为 event_variable_equation_function，
        # 事件边界为 switch_fixture[0]
        SwitchedSystem(
            dim_output=1,
            event_variable_equation_function=event_variable_equation_function,
            event_bounds=switch_fixture[0],
        )
    # 使用 pytest 的断言检查是否会抛出 ValueError 异常，并匹配包含 "broadcast" 的错误信息
    with pytest.raises(ValueError, match="broadcast"):
        # 创建 SwitchedSystem 对象，设定 dim_output=1，事件变量方程函数为 event_variable_equation_function，
        # 事件边界为 switch_fixture[0]，输出方程函数为一个包含 lambda 表达式的 NumPy 数组
        SwitchedSystem(
            dim_output=1,
            event_variable_equation_function=event_variable_equation_function,
            event_bounds=switch_fixture[0],
            output_equations_functions=np.array([
                     lambda t, u: cnd*np.ones(u.shape)
                     for cnd in range(switch_fixture[0].size+2)
             ]),
        )
    # 创建 SwitchedSystem 对象，设定 dim_state=1，事件变量方程函数为 event_variable_equation_function，
    # 事件边界为 switch_fixture[0]，状态方程函数为 switch_fixture[1]
    sys = SwitchedSystem(
        dim_state=1,
        event_variable_equation_function=event_variable_equation_function,
        event_bounds=switch_fixture[0],
        state_equations_functions=switch_fixture[1],
    )
    # 使用 numpy.testing.assert_array_equal 检查 sys.output_equations_functions 是否与 full_state_output 数组相等
    npt.assert_array_equal(
        sys.output_equations_functions,
        full_state_output
    )

    # 创建 SwitchedSystem 对象，设定 dim_output=1，事件变量方程函数为 event_variable_equation_function，
    # 事件边界为 switch_fixture[0]，输出方程函数为 switch_fixture[2]
    sys = SwitchedSystem(
        dim_output=1,
        event_variable_equation_function=event_variable_equation_function,
        event_bounds=switch_fixture[0],
        output_equations_functions=switch_fixture[2]
    )
    # 使用 numpy.testing.assert_array_equal 检查 sys.output_equations_functions 是否与 switch_fixture[2] 数组相等
    npt.assert_array_equal(sys.output_equations_functions, switch_fixture[2])

    # 创建 SwitchedSystem 对象，设定 dim_output=1，事件变量方程函数为 event_variable_equation_function，
    # 事件边界为 switch_fixture[0]，输出方程函数为 output_equation_function
    sys = SwitchedSystem(
        dim_output=1,
        event_variable_equation_function=event_variable_equation_function,
        event_bounds=switch_fixture[0],
        output_equations_functions=output_equation_function
    )
    # 使用 numpy.testing.assert_array_equal 检查 sys.output_equations_functions 是否与 output_equation_function 数组相等
    npt.assert_array_equal(
        sys.output_equations_functions,
        output_equation_function
    )


def test_event_equation_function(switch_fixture):
    # 创建 SwitchedSystem 对象，设定 dim_output=1，事件变量方程函数为 event_variable_equation_function，
    # 事件边界为 switch_fixture[0]，状态方程函数为 switch_fixture[1]，输出方程函数为 switch_fixture[2]
    sys = SwitchedSystem(
        dim_output=1,
        event_variable_equation_function=event_variable_equation_function,
        event_bounds=switch_fixture[0],
        state_equations_functions=switch_fixture[1],
        output_equations_functions=switch_fixture[2],
    )

    # 使用断言检查 sys.state_update_equation_function 是否等于 full_state_output
    assert sys.state_update_equation_function == full_state_output

    # 遍历 bounds_min 到 bounds_max 之间的值，并检查事件方程的输出是否不全为零
    for x in np.linspace(bounds_min, bounds_max):
        if not np.any(np.isclose(x, switch_fixture[0])):
            assert ~np.any(np.isclose(
                sys.event_equation_function(x, x),
                0
            ))

    # 遍历 switch_fixture[0] 中的每个值，并使用 numpy.testing.assert_allclose 检查事件方程在给定索引和值时是否接近零
    for zero in switch_fixture[0]:
        npt.assert_allclose(
            sys.event_equation_function(len(switch_fixture[0]), zero),
            0
        )


def test_update_equation_function(switch_fixture):
    # 创建 SwitchedSystem 对象，设定 dim_output=1，事件变量方程函数为 event_variable_equation_function，
    # 事件边界为 switch_fixture[0]，状态方程函数为 switch_fixture[1]，输出方程函数为 switch_fixture[2]
    sys = SwitchedSystem(
        dim_output=1,
        event_variable_equation_function=event_variable_equation_function,
        event_bounds=switch_fixture[0],
        state_equations_functions=switch_fixture[1],
        output_equations_functions=switch_fixture[2],
    )

    # 使用断言检查 sys 是否没有 'condition_idx' 属性
    assert not hasattr(sys, 'condition_idx')
    # 调用 sys.prepare_to_integrate() 方法，准备进行积分
    sys.prepare_to_integrate()

    # 使用断言检查 sys.condition_idx 是否为 None
    assert sys.condition_idx is None
    # 调用 sys.update_equation_function 方法，传入随机生成的值和 bounds_min 进行更新方程的计算
    sys.update_equation_function(np.random.rand(1), bounds_min)
    # 断言系统的条件索引是否为0，用于检查系统状态
    assert sys.condition_idx == 0
    
    # 遍历 switch_fixture[0] 中的元素及其索引
    for cnd_idx, zero in enumerate(switch_fixture[0]):
        # 使用随机生成的一个数调用 sys 的更新方程函数，zero 是一个参数
        sys.update_equation_function(np.random.rand(1), zero)
        # 断言系统的条件索引是否与当前索引 cnd_idx + 1 相等，用于检查系统状态是否更新正确
        assert sys.condition_idx == cnd_idx + 1
    
    # 如果 switch_fixture[0] 中的元素数量大于1
    if len(switch_fixture[0]) > 1:
        # 在执行下面代码块期间，预期会引发 UserWarning 警告
        with pytest.warns(UserWarning):
            # 使用随机生成的一个数调用 sys 的更新方程函数，bounds_min 是一个参数
            sys.update_equation_function(np.random.rand(1), bounds_min)
```