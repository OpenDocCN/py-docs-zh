# `.\simupy\tests\test_block_diagram.py`

```
import pytest  # 导入 pytest 模块，用于单元测试
import numpy as np  # 导入 NumPy 库，用于数值计算
import sympy as sp  # 导入 SymPy 库，用于符号计算
import numpy.testing as npt  # 导入 NumPy 测试模块，用于数值测试
from scipy import linalg, signal  # 导入 SciPy 的线性代数和信号处理模块
from simupy.systems import LTISystem, SystemFromCallable  # 导入 simupy 库中的系统模块
from simupy.systems.symbolic import dynamicsymbols  # 导入 simupy 的符号动力学模块
from simupy.discontinuities import SwitchedSystem  # 导入 simupy 的离散系统模块
from simupy.array import Array, r_  # 导入 simupy 中的数组模块
from simupy.utils import (callable_from_trajectory,  # 导入 simupy 中的工具函数
                          array_callable_from_vector_trajectory)
from simupy.matrices import construct_explicit_matrix, system_from_matrix_DE  # 导入 simupy 的矩阵模块
import simupy.block_diagram as block_diagram  # 导入 simupy 的块图模块

BlockDiagram = block_diagram.BlockDiagram  # 定义 BlockDiagram 为块图对象
block_diagram.DEFAULT_INTEGRATOR_OPTIONS['rtol'] = 1E-9  # 设置块图默认数值积分选项中的相对误差阈值

TEST_ATOL = 1E-6  # 定义测试绝对误差阈值
TEST_RTOL = 1E-6  # 定义测试相对误差阈值

RICCATI_TEST_RTOL = 2E-2  # 定义 Riccati 方程测试的相对误差阈值
RICCATI_TEST_ATOL = 0  # 定义 Riccati 方程测试的绝对误差阈值


def get_double_integrator(m=1000, b=50, d=1):
    N = 2
    sys = LTISystem(
        np.c_[[0, 1], [1, -b/m]],  # 系统状态方程的 A 矩阵
        np.r_[0, d/m],  # 系统状态方程的 B 向量
        # np.r_[0, 1],  # 系统输出方程的 C 向量，暂时注释掉
    )

    def ref_func(*args):
        if len(args) == 1:
            x = np.zeros(N)
        else:
            x = args[1]
        return np.r_[d/m, 0]-x  # 返回参考输入函数的结果
    ref = SystemFromCallable(ref_func, N, N)  # 创建基于函数的系统对象

    return sys, ref  # 返回系统对象和参考输入对象


def get_electromechanical(b=1, R=1, L=1, K=np.pi/5, M=1):
    # TODO: 确定良好的参考和/或初始条件
    # TODO: 确定 b, R, L, M 的良好默认值
    N = 3
    sys = LTISystem(
        np.c_[  # 系统状态方程的 A 矩阵
            [0, 0, 0],
            [1, -b/M, -K/L],
            [0, K/M, -R/L]
        ],
        np.r_[0, 0, 1/L],  # 系统状态方程的 B 向量
        # np.r_[1, 0, 0],  # 系统输出方程的 C 向量，暂时注释掉
    )
    sys.initial_condition = np.ones(N)  # 设置系统的初始条件向量

    def ref_func(*args):
        if len(args) == 1:
            x = np.zeros(N)
        else:
            x = args[1]
        return np.r_[0, 0, 0]-x  # 返回参考输入函数的结果
    ref = SystemFromCallable(ref_func, N, N)  # 创建基于函数的系统对象

    return sys, ref  # 返回系统对象和参考输入对象


def get_double_mechanical(m1=1, k1=1, b1=1, m2=1, k2=0, b2=0):
    # TODO: 确定 m1, k1, b1, m2 的良好默认值？
    N = 4
    sys = LTISystem(
        np.c_[  # 系统状态方程的 A 矩阵
            [0, -k1/m1, 0, k1/m2],
            [1, -b1/m1, 0, b1/m2],
            [0, k1/m1, 0, -(k1+k2)/m2],
            [0, b1/m1, 1, -(b1+b2)/m2]
        ],
        np.r_[0, 1/m1, 0, 0],  # 系统状态方程的 B 向量
        # np.r_[0, 0, 1, 0],  # 系统输出方程的 C 向量，暂时注释掉
    )
    sys.initial_condition = np.r_[
        0.5/k1 if k1 else 0,
        0,
        0.5/k2 if k2 else 0,
        0
    ]  # 设置系统的初始条件向量

    def ref_func(*args):
        if len(args) == 1:
            x = np.zeros(N)
        else:
            x = args[1]
        return np.zeros(N)-x  # 返回参考输入函数的结果
    ref = SystemFromCallable(ref_func, N, N)  # 创建基于函数的系统对象

    return sys, ref  # 返回系统对象和参考输入对象


def get_cart_pendulum(m=1, M=3, L=0.5, g=9.81, pedant=False):
    N = 4
    sys = LTISystem(
        np.c_[  # 系统状态方程的 A 矩阵
            [0, 0, 0, 0],
            [1, 0, 0, 0],
            [0, m*g/M, 0, (-1)**(pedant)*(m+M)*g/(M*L)],
            [0, 0, 1, 0]
        ],
        np.r_[0, 1/M, 0, 1/(M*L)],  # 系统状态方程的 B 向量
        # np.r_[1, 0, 1, 0]  # 系统输出方程的 C 向量，暂时注释掉
    )
    sys.initial_condition = np.r_[0, 0, np.pi/3, 0]  # 设置系统的初始条件向量
    # 定义一个函数 ref_func，接受可变数量的参数 args
    def ref_func(*args):
        # 如果参数 args 的长度为 1
        if len(args) == 1:
            # 创建一个长度为 N 的零数组，并赋值给变量 x
            x = np.zeros(N)
        else:
            # 否则将 args 中索引为 1 的参数赋给 x
            x = args[1]
        # 返回一个长度为 N 的零数组减去 x 的结果
        return np.zeros(N) - x
    
    # 使用 SystemFromCallable 函数创建一个系统对象 ref，传入 ref_func 函数、N 和 N 作为参数
    ref = SystemFromCallable(ref_func, N, N)

    # 返回创建的系统对象 sys 和 ref
    return sys, ref
@pytest.fixture(
    scope="module",
    params=[
        get_electromechanical(),  # 获取通用电机电机

        get_double_mechanical(),  # 自由端
        get_double_mechanical(k2=1, b2=1),  # 弹簧阻尼端

        get_cart_pendulum(),  # 通用小车摆动模型

        # get_double_integrator(m=1, d=1, b=4.0),  # 过阻尼
        get_double_integrator(m=1, d=1, b=2.0),  # 临界阻尼
        get_double_integrator(m=1, d=1, b=1.0),  # 欠阻尼
        get_double_integrator(m=1, d=1, b=0.0),  # 无阻尼
        get_double_integrator(m=1, d=1, b=-0.5),  # 不稳定
    ]
)
def control_systems(request):
    ct_sys, ref = request.param
    Ac, Bc, Cc = ct_sys.data
    Dc = np.zeros((Cc.shape[0], 1))

    Q = np.eye(Ac.shape[0])
    R = np.eye(Bc.shape[1] if len(Bc.shape) > 1 else 1)

    # 解决连续时域的代数Riccati方程
    Sc = linalg.solve_continuous_are(Ac, Bc.reshape(-1, 1), Q, R,)
    # 计算最优控制器增益
    Kc = linalg.solve(R, Bc.T @ Sc).reshape(1, -1)
    ct_ctr = LTISystem(Kc)

    # 计算系统特征值的绝对值并排序
    evals = np.sort(np.abs(
        linalg.eig(Ac, left=False, right=False, check_finite=False)
    ))
    # 计算离散化步长
    dT = 1/(2*evals[-1])

    # 计算仿真总时间
    Tsim = (8/np.min(evals[~np.isclose(evals, 0)])
            if np.sum(np.isclose(evals[np.nonzero(evals)], 0)) > 0
            else 8
            )

    # 将连续时域系统转换为离散时域系统
    dt_data = signal.cont2discrete((Ac, Bc.reshape(-1, 1), Cc, Dc), dT)
    Ad, Bd, Cd, Dd = dt_data[:-1]
    # 解决离散时域的代数Riccati方程
    Sd = linalg.solve_discrete_are(Ad, Bd.reshape(-1, 1), Q, R,)
    # 计算最优离散控制器增益
    Kd = linalg.solve(Bd.T @ Sd @ Bd + R, Bd.T @ Sd @ Ad)

    dt_sys = LTISystem(Ad, Bd, dt=dT)
    dt_sys.initial_condition = ct_sys.initial_condition
    dt_ctr = LTISystem(Kd, dt=dT)

    yield ct_sys, ct_ctr, dt_sys, dt_ctr, ref, Tsim


@pytest.fixture(scope="module", params=['dopri5', 'lsoda'])
def intname(request):
    if request.param == 'lsoda':
        pytest.xfail("Only support adaptive step-size and dense output.")
    yield request.param


@pytest.fixture(scope="module")
def simulation_results(control_systems, intname):
    ct_sys, ct_ctr, dt_sys, dt_ctr, ref, Tsim = control_systems

    intopts = block_diagram.DEFAULT_INTEGRATOR_OPTIONS.copy()
    intopts['name'] = intname

    if intname == 'dopri5':
        tspan = Tsim
    elif intname == 'lsoda':
        tspan = np.arange(0, Tsim, dt_sys.dt*2**-2)

    results = []
    for sys, ctr in [(dt_sys, dt_ctr), (ct_sys, ct_ctr), (ct_sys, dt_ctr)]:
        bd = BlockDiagram(sys, ref, ctr)
        bd.connect(sys, ref)
        bd.connect(ref, ctr)
        bd.connect(ctr, sys)
        results.append(bd.simulate(tspan, integrator_options=intopts))

    yield results, ct_sys, ct_ctr, dt_sys, dt_ctr, ref, Tsim, tspan, intname


@pytest.mark.xfail
def test_fixed_integration_step_equivalent(control_systems):
    """
    使用类似列表的 tspan 或浮点数的 tspan 应给出相同的结果
    """
    ct_sys, ct_ctr, dt_sys, dt_ctr, ref, Tsim = control_systems
    # 循环迭代器，依次处理 (ct_sys, ct_ctr) 和 (ct_sys, dt_ctr)
    for sys, ctr in [(ct_sys, ct_ctr), (ct_sys, dt_ctr)]:
        # 创建一个 BlockDiagram 对象，用于建立系统模块的图示，包括 sys, ref, ctr
        bd = BlockDiagram(sys, ref, ctr)
        # 连接 sys 到 ref
        bd.connect(sys, ref)
        # 连接 ref 到 ctr
        bd.connect(ref, ctr)
        # 连接 ctr 到 sys，形成闭环
        bd.connect(ctr, sys)

        # 对 BlockDiagram 进行仿真，模拟时间长度为 Tsim
        var_res = bd.simulate(Tsim)
        # 使用固定时间步长 dt_sys.dt 对 BlockDiagram 进行仿真
        fix_res = bd.simulate(np.arange(1+Tsim//dt_sys.dt)*dt_sys.dt)

        # 根据控制器类型 ctr 执行不同的断言
        if ctr == ct_ctr:
            # 获取唯一时间点 unique_t 和对应的索引 unique_t_sel
            unique_t, unique_t_sel = np.unique(var_res.t, return_index=True)
            # 从仿真结果中提取在 unique_t 时间点处的状态
            ct_res = callable_from_trajectory(unique_t, var_res.x[unique_t_sel, :])

            # 断言仿真结果 ct_res(fix_res.t) 与固定时间步长仿真结果 fix_res.x 的近似性
            npt.assert_allclose(
                ct_res(fix_res.t), fix_res.x,
                atol=TEST_ATOL, rtol=TEST_RTOL
            )
        else:
            # 在 var_res.t 和 fix_res.t 中找到相等的时间点，并选取其中的偶数索引
            var_sel = np.where(np.equal(*np.meshgrid(var_res.t, fix_res.t)))[1][::2]

            # 断言仿真结果 var_res.x[var_sel, :] 与固定时间步长仿真结果 fix_res.x 的近似性
            npt.assert_allclose(
                var_res.x[var_sel, :], fix_res.x,
                atol=TEST_ATOL, rtol=TEST_RTOL
            )

    # 创建一个新的 BlockDiagram 对象，用于建立离散系统模块的图示，包括 dt_sys, ref, dt_ctr
    bd = BlockDiagram(dt_sys, ref, dt_ctr)
    # 连接 dt_sys 到 ref
    bd.connect(dt_sys, ref)
    # 连接 ref 到 dt_ctr
    bd.connect(ref, dt_ctr)
    # 连接 dt_ctr 到 dt_sys，形成闭环
    bd.connect(dt_ctr, dt_sys)

    # 对 BlockDiagram 进行仿真，模拟时间长度为 Tsim
    var_res = bd.simulate(Tsim)
    # 使用时间点 var_res.t[var_res.t < Tsim] 对 BlockDiagram 进行仿真
    fix_res = bd.simulate(var_res.t[var_res.t < Tsim])

    # 断言仿真结果 var_res.x[var_res.t < Tsim] 与固定时间仿真结果 fix_res.x 的近似性
    npt.assert_allclose(
        var_res.x[var_res.t < Tsim], fix_res.x,
        atol=TEST_ATOL, rtol=TEST_RTOL
    )
# 检验反馈控制系统下的等效性，确保仿真结果和系统A、B在反馈K下的行为一致
def test_feedback_equivalent(simulation_results):
    # 解包仿真结果和系统参数
    results, ct_sys, ct_ctr, dt_sys, dt_ctr, ref, Tsim, tspan, intname = \
        simulation_results

    # 复制默认积分器选项，并设置积分器名称
    intopts = block_diagram.DEFAULT_INTEGRATOR_OPTIONS.copy()
    intopts['name'] = intname

    # 创建等效的离散时间系统
    dt_equiv_sys = LTISystem(dt_sys.F + dt_sys.G @ dt_ctr.K,
                             -dt_sys.G @ dt_ctr.K, dt=dt_sys.dt)
    dt_equiv_sys.initial_condition = dt_sys.initial_condition

    # 构建块图并连接参考信号和等效系统
    dt_bd = BlockDiagram(dt_equiv_sys, ref)
    dt_bd.connect(ref, dt_equiv_sys)
    # 对等效系统进行仿真
    dt_equiv_res = dt_bd.simulate(tspan, integrator_options=intopts)

    # 查找离散时间和连续时间仿真时间相等的索引
    mixed_t_discrete_t_equal_idx = np.where(
        np.equal(*np.meshgrid(dt_equiv_res.t, results[0].t))
    )[1]

    # 断言等效系统的状态变量与仿真结果的状态变量在给定的容差下相等
    npt.assert_allclose(
        dt_equiv_res.x, results[0].x,
        atol=TEST_ATOL, rtol=TEST_RTOL
    )

    # 创建等效的连续时间系统
    ct_equiv_sys = LTISystem(ct_sys.F + ct_sys.G @ ct_ctr.K,
                             -ct_sys.G @ ct_ctr.K)
    ct_equiv_sys.initial_condition = ct_sys.initial_condition

    # 构建块图并连接参考信号和等效系统
    ct_bd = BlockDiagram(ct_equiv_sys, ref)
    ct_bd.connect(ref, ct_equiv_sys)
    # 对等效系统进行仿真
    ct_equiv_res = ct_bd.simulate(tspan, integrator_options=intopts)
    # 提取唯一的仿真时间点和相应的状态变量
    unique_t, unique_t_sel = np.unique(ct_equiv_res.t, return_index=True)
    ct_res = callable_from_trajectory(
        unique_t,
        ct_equiv_res.x[unique_t_sel, :]
    )

    # 断言等效系统的响应与仿真结果的响应在给定的容差下相等
    npt.assert_allclose(
        ct_res(results[1].t), results[1].x,
        atol=TEST_ATOL, rtol=TEST_RTOL
    )


# 检验离散时间系统和连续时间系统的等效性
def test_dt_ct_equivalent(simulation_results):
    """
    CT系统在时间点t=k*dT下应与其等效的DT系统完全匹配。
    """
    # 解包仿真结果和系统参数
    results, ct_sys, ct_ctr, dt_sys, dt_ctr, ref, Tsim, tspan, intname = \
        simulation_results

    # 提取离散时间仿真时间点和索引
    dt_unique_t, dt_unique_t_idx = np.unique(
        results[0].t, return_index=True
    )
    discrete_sel = dt_unique_t_idx[
        (dt_unique_t < (Tsim*7/8)) & (dt_unique_t % dt_sys.dt == 0)
    ]

    # 查找离散时间和混合时间的相等时间点的索引
    mixed_t_discrete_t_equal_idx = np.where(
        np.equal(*np.meshgrid(results[2].t, results[0].t[discrete_sel]))
    )[1]

    # 提取混合时间中与离散时间相等的唯一时间点和相应的索引
    mixed_unique_t, mixed_unique_t_idx_rev = np.unique(
        results[2].t[mixed_t_discrete_t_equal_idx][::-1], return_index=True
    )
    mixed_unique_t_idx = (len(mixed_t_discrete_t_equal_idx)
                          - mixed_unique_t_idx_rev - 1)
    mixed_sel = mixed_t_discrete_t_equal_idx[mixed_unique_t_idx]

    # 断言混合时间和离散时间的时间点在给定的容差下相等
    npt.assert_allclose(
        results[2].t[mixed_sel], results[0].t[discrete_sel],
        atol=TEST_ATOL, rtol=TEST_RTOL
    )

    # 断言混合时间和离散时间的状态变量在给定的容差下相等
    npt.assert_allclose(
        results[2].x[mixed_sel, :], results[0].x[discrete_sel, :],
        atol=TEST_ATOL, rtol=TEST_RTOL
    )


# 检验混合时间系统的匹配性
def test_mixed_dts(simulation_results):
    """
    采样频率加倍的DT等效系统在时间点t=k*dT下应与原始DT等效系统完全匹配。
    """
    # 解包仿真结果和系统参数
    results, ct_sys, ct_ctr, dt_sys, dt_ctr, ref, Tsim, tspan, intname = \
        simulation_results
    Ac, Bc, Cc = ct_sys.data
    # 创建一个形状为 (Cc.shape[0], 1) 的全零数组 Dc
    Dc = np.zeros((Cc.shape[0], 1))

    # 提取 results[0].t 中唯一的时间点，并返回这些时间点的索引
    dt_unique_t, dt_unique_t_idx = np.unique(
        results[0].t, return_index=True
    )

    # 根据条件筛选出 diskrete_sel，这些索引是 results[0].t 中小于 Tsim*7/8
    # 且能被 dt_sys.dt 整除的时间点的索引
    discrete_sel = dt_unique_t_idx[
        (dt_unique_t < (Tsim*7/8)) & (dt_unique_t % dt_sys.dt == 0)
    ]

    # 计算离散化系统的缩放因子
    scale_factor = 1/2

    # 将连续系统 (Ac, Bc.reshape(-1, 1), Cc, Dc) 转换为离散系统
    # 使用 dt_sys.dt 乘以缩放因子作为离散化的时间步长
    Ad, Bd, Cd, Dd, dT = signal.cont2discrete(
        (Ac, Bc.reshape(-1, 1), Cc, Dc),
        dt_sys.dt * scale_factor
    )

    # 创建一个新的 LTISystem 对象 dt_sys2，使用离散化后的系统参数
    dt_sys2 = LTISystem(Ad, Bd, dt=dT)

    # 将连续系统的初始条件设置为离散系统的初始条件
    dt_sys2.initial_condition = ct_sys.initial_condition

    # 复制默认的积分器选项并设置名称为 intname
    intopts = block_diagram.DEFAULT_INTEGRATOR_OPTIONS.copy()
    intopts['name'] = intname

    # 创建一个 BlockDiagram 对象 bd，用于组合系统模块
    bd = BlockDiagram(dt_sys2, ref, dt_ctr)

    # 将系统模块连接起来
    bd.connect(dt_sys2, ref)
    bd.connect(ref, dt_ctr)
    bd.connect(dt_ctr, dt_sys2)

    # 使用积分器选项进行仿真，生成仿真结果 res
    res = bd.simulate(tspan, integrator_options=intopts)

    # 找出 res.t 和 results[0].t[discrete_sel] 中时间点相等的索引
    mixed_t_discrete_t_equal_idx = np.where(
        np.equal(*np.meshgrid(res.t, results[0].t[discrete_sel]))
    )[1]

    # 从 res.t 中提取混合时间点，并返回唯一值及其逆序索引
    mixed_unique_t, mixed_unique_t_idx_rev = np.unique(
        res.t[mixed_t_discrete_t_equal_idx][::-1], return_index=True
    )

    # 根据逆序索引计算 mixed_unique_t_idx，这是最终选择的时间点的索引
    mixed_unique_t_idx = (len(mixed_t_discrete_t_equal_idx)
                          - mixed_unique_t_idx_rev - 1)

    # 选取最终的混合时间点的索引 mixed_sel
    mixed_sel = mixed_t_discrete_t_equal_idx[mixed_unique_t_idx]

    # 使用 numpy.testing.assert_allclose 检查仿真结果的状态变量
    npt.assert_allclose(
        res.x[mixed_sel, :], results[0].x[discrete_sel, :],
        atol=TEST_ATOL, rtol=TEST_RTOL
    )
# 定义测试函数，用于测试 Riccati 代数方程
def test_riccati_algebraic_equation(control_systems):
    # 解构控制系统参数元组
    ct_sys, ct_ctr, dt_sys, dt_ctr, ref, Tsim = control_systems
    # 获取状态空间维度和输入维度
    n = ct_sys.dim_state
    m = ct_sys.dim_input

    # 构造对称动态显式矩阵 S
    S = construct_explicit_matrix('s', n, n, symmetric=True, dynamic=True)
    # 初始化 Q 和 R 矩阵
    Q = np.eye(n)
    R = np.eye(m)
    # 计算 R 的逆矩阵
    Rinv = np.linalg.inv(R)
    # 解构状态空间矩阵 A, B, C
    A, B, C = ct_sys.data

    # 计算 Riccati 微分方程的右侧
    Sdot = (Q + A.T @ S + S @ A - S @ B @ Rinv @ B.T @ S)
    # 根据矩阵微分方程创建系统
    ct_riccati_sys = system_from_matrix_DE(Sdot, S)
    # 创建系统的 BlockDiagram
    ct_riccati_bd = BlockDiagram(ct_riccati_sys)
    # 模拟系统仿真结果
    ct_riccati_res = ct_riccati_bd.simulate(2 * Tsim)

    # 提取仿真数据中唯一的时间点和其索引
    ct_sim_data_unique_t, ct_sim_data_unique_t_idx = np.unique(
        ct_riccati_res.t,
        return_index=True
    )

    # 根据向量轨迹创建可调用的数组
    mat_ct_s_callable = array_callable_from_vector_trajectory(
        np.flipud(Tsim - ct_riccati_res.t[ct_sim_data_unique_t_idx]),
        np.flipud(ct_riccati_res.x[ct_sim_data_unique_t_idx]),
        ct_riccati_sys.state,
        S
    )

    # 计算控制增益矩阵 K
    K = Rinv @ B.T @ mat_ct_s_callable(0)
    # 使用数值断言检查计算结果是否与预期值 ct_ctr.K 接近
    npt.assert_allclose(
        K, ct_ctr.K, atol=RICCATI_TEST_ATOL, rtol=RICCATI_TEST_RTOL
    )


# 定义事件测试函数
def test_events():
    # 使用弹跳球模拟事件工作
    # 复制默认的积分器选项
    int_opts = block_diagram.DEFAULT_INTEGRATOR_OPTIONS.copy()
    int_opts['rtol'] = 1E-12
    int_opts['atol'] = 1E-15
    int_opts['nsteps'] = 1000
    int_opts['max_step'] = 2 ** -3

    # 定义动力学符号变量和常数
    x = x1, x2 = Array(dynamicsymbols('x_1:3'))
    mu, g = sp.symbols('mu g')
    constants = {mu: 0.8, g: 9.81}
    ic = np.r_[10, 15]

    # 创建 SwitchedSystem 对象
    sys = SwitchedSystem(
        x1, Array([0]),
        state_equations=r_[x2, -g],
        state_update_equation=r_[sp.Abs(x1), -mu * x2],
        state=x,
        constants_values=constants,
        initial_condition=ic
    )
    # 创建 BlockDiagram 对象
    bd = BlockDiagram(sys)
    # 进行仿真，存储结果
    res = bd.simulate(5, integrator_options=int_opts)

    # 计算实际碰撞时间
    tvar = dynamicsymbols._t
    impact_eq = (x2 * tvar - g * tvar ** 2 / 2 + x1).subs(
        {x1: ic[0], x2: ic[1], g: 9.81}
    )
    t_impact = sp.solve(impact_eq, tvar)[-1]

    # 确保仿真在碰撞附近改变速度符号
    abs_diff_impact = np.abs(res.t - t_impact)
    impact_idx = np.where(abs_diff_impact == np.min(abs_diff_impact))[0]
    assert np.sign(res.x[impact_idx - 1, 1]) != np.sign(res.x[impact_idx + 1, 1])
```