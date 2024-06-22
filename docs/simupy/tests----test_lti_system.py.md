# `.\simupy\tests\test_lti_system.py`

```
# 导入所需的库
import numpy as np                      # 导入NumPy库，用于数值计算
import numpy.testing as npt             # 导入NumPy的测试模块，用于测试数组
from simupy.systems import LTISystem    # 从simupy.systems模块中导入LTISystem类
import pytest                           # 导入pytest库，用于编写和运行测试用例

# 定义元素的最小值和最大值
elem_min = -1
elem_max = +1

# 定义最大的n、m、p值
max_n = 4
max_m = 4
max_p = 4

# 测试函数：测试LTISystem类的K矩阵设置
def test_k():
    # 遍历不同的m和p值
    for m in range(1, max_m+1):
        for p in range(1, max_p+1):
            # 创建随机的K矩阵和输入向量u
            K = elem_min + (elem_max-elem_min)*np.random.rand(p, m)
            u = elem_min + (elem_max-elem_min)*np.random.rand(m)
            
            # 创建LTISystem对象
            sys = LTISystem(K)
            
            # 断言LTISystem对象的状态空间维度为0，输入维度为m，输出维度为p
            assert sys.dim_state == 0
            assert sys.dim_input == m
            assert sys.dim_output == p
            
            # 使用npt.assert_allclose函数断言输出方程的计算结果与预期的K@u相近
            npt.assert_allclose(
                sys.output_equation_function(p*max_m + m, u),
                K @ u
            )

# 测试函数：测试LTISystem类的A和B矩阵设置
def test_ab():
    # 遍历不同的n和m值
    for n in range(1, max_n+1):
        for m in range(1, max_m+1):
            # 创建随机的A和B矩阵，以及状态向量x和输入向量u
            A = elem_min + (elem_max-elem_min)*np.random.rand(n, n)
            B = elem_min + (elem_max-elem_min)*np.random.rand(n, m)
            x = elem_min + (elem_max-elem_min)*np.random.rand(n)
            u = elem_min + (elem_max-elem_min)*np.random.rand(m)
            
            # 如果n不等于m，验证LTISystem初始化时是否会引发AssertionError
            if n != m:
                with pytest.raises(AssertionError):
                    LTISystem(np.random.rand(n, m), B)
                with pytest.raises(AssertionError):
                    LTISystem(A, np.random.rand(m, n))
            
            # 创建LTISystem对象
            sys = LTISystem(A, B)
            
            # 断言LTISystem对象的状态空间维度为n，输入维度为m，输出维度也为n
            assert sys.dim_state == n
            assert sys.dim_output == n
            assert sys.dim_input == m
            
            # 使用npt.assert_allclose函数断言状态方程的计算结果与预期的A@x + B@u相近
            npt.assert_allclose(
                sys.state_equation_function(n*max_m + m, x, u),
                A @ x + B @ u
            )
            
            # 使用npt.assert_allclose函数断言输出方程的计算结果与预期的x相近
            npt.assert_allclose(
                sys.output_equation_function(n*max_m + m, x),
                x
            )

# 测试函数：测试LTISystem类的A、B和C矩阵设置
def test_abc():
    # 遍历不同的n、m和p值
    for n in range(1, max_n+1):
        for m in range(1, max_m+1):
            for p in range(1, max_p+1):
                # 创建随机的A、B和C矩阵，以及状态向量x和输入向量u
                A = elem_min + (elem_max-elem_min)*np.random.rand(n, n)
                B = elem_min + (elem_max-elem_min)*np.random.rand(n, m)
                C = elem_min + (elem_max-elem_min)*np.random.rand(p, n)
                x = elem_min + (elem_max-elem_min)*np.random.rand(n)
                u = elem_min + (elem_max-elem_min)*np.random.rand(m)
                
                # 如果p不等于n，验证LTISystem初始化时是否会引发AssertionError
                if p != n:
                    with pytest.raises(AssertionError):
                        LTISystem(A, B, np.random.rand(n, p))
                
                # 创建LTISystem对象
                sys = LTISystem(A, B, C)
                
                # 断言LTISystem对象的状态空间维度为n，输入维度为m，输出维度为p
                assert sys.dim_state == n
                assert sys.dim_output == p
                assert sys.dim_input == m
                
                # 使用npt.assert_allclose函数断言状态方程的计算结果与预期的A@x + B@u相近
                npt.assert_allclose(
                    sys.state_equation_function(n*max_m + m, x, u),
                    A @ x + B @ u
                )
                
                # 使用npt.assert_allclose函数断言输出方程的计算结果与预期的C@x相近
                npt.assert_allclose(
                    sys.output_equation_function(n*max_m + m, x),
                    C @ x
                )
```