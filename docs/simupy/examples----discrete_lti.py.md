# `.\simupy\examples\discrete_lti.py`

```
# 导入所需模块
import numpy as np
from scipy import signal, linalg
from simupy.systems import LTISystem
from simupy.block_diagram import BlockDiagram
import matplotlib.pyplot as plt

# 设置使用的模型
use_model = 1

# 如果使用模型为 0，则设置模型参数
if use_model == 0:
    m = 1
    M = 3
    L = 0.5
    g = 9.81
    pedant = False
    Ac = np.c_[  # A
        [0, 0, 0, 0],
        [1, 0, 0, 0],
        [0, m*g/M, 0, (-1)**(pedant)*(m+M)*g/(M*L)],
        [0, 0, 1, 0]
    ]
    Bc = np.r_[0, 1/M, 0, 1/(M*L)].reshape(-1, 1)
    Cc = np.eye(4)
    Dc = np.zeros((4, 1))
    ic = np.r_[0, 0, np.pi/3, 0]
# 如果使用模型为 1，则设置模型参数
elif use_model == 1:
    m = 1
    d = 1
    b = 1
    k = 1
    Ac = np.c_[[0, -k/m], [1, -b/m]]
    Bc = np.r_[0, d/m].reshape(-1, 1)
    Cc = np.eye(2)
    Dc = np.zeros((2, 1))
    ic = np.r_[1, 0]

# 创建连续时间系统对象，设置初始条件
ct_sys = LTISystem(Ac, Bc, Cc)
ct_sys.initial_condition = ic

# 获取系统的维度
n, m = Bc.shape

# 计算系统的特征值，并计算采样周期
evals = np.sort(np.abs(np.real(
    linalg.eig(Ac, left=False, right=False, check_finite=False)
)))
dT = 1/(2*evals[-1])
Tsim = (8/np.min(evals[~np.isclose(evals[np.nonzero(evals)], 0)])
        if np.sum(np.isclose(evals[np.nonzero(evals)], 0)) > 0
        else 8
        )

# 连续时间系统转为离散时间系统
Ad, Bd, Cd, Dd, dT = signal.cont2discrete((Ac, Bc, Cc, Dc), dT)
dt_sys = LTISystem(Ad, Bd, Cd, dt=dT)
dt_sys.initial_condition = ic

# 设置权重矩阵 Q 和 R
Q = np.eye(Ac.shape[0])
R = np.eye(Bc.shape[1] if len(Bc.shape) > 1 else 1)

# 求解连续时间系统的最优控制器
Sc = linalg.solve_continuous_are(Ac, Bc, Q, R,)
Kc = linalg.solve(R, Bc.T @ Sc).reshape(1, -1)
ct_ctr = LTISystem(-Kc)

# 求解离散时间系统的最优控制器
Sd = linalg.solve_discrete_are(Ad, Bd, Q, R,)
Kd = linalg.solve(Bd.T @ Sd @ Bd + R, Bd.T @ Sd @ Ad)
dt_ctr = LTISystem(-Kd, dt=dT)

# 创建连续时间系统和离散时间控制器的闭环图
dtct_bd = BlockDiagram(ct_sys, dt_ctr)
dtct_bd.connect(ct_sys, dt_ctr)
dtct_bd.connect(dt_ctr, ct_sys)
dtct_res = dtct_bd.simulate(Tsim)

# 创建离散时间系统和离散时间控制器的闭环图
dtdt_bd = BlockDiagram(dt_sys, dt_ctr)
dtdt_bd.connect(dt_sys, dt_ctr)
dtdt_bd.connect(dt_ctr, dt_sys)
dtdt_res = dtdt_bd.simulate(Tsim)

# 绘制图形
plt.figure()
for st in range(n):
    plt.subplot(n+m, 1, st+1)
    plt.plot(dtct_res.t, dtct_res.y[:, st], '+-')
    plt.plot(dtdt_res.t, dtdt_res.y[:, st], 'x-')
    plt.ylabel('$x_{}(t)$'.format(st+1))
    plt.grid(True)
for st in range(m):
    plt.subplot(n+m, 1, st+n+1)
    plt.plot(dtct_res.t, dtct_res.y[:, st+n], '+-')
    plt.plot(dtdt_res.t, dtdt_res.y[:, st+n], 'x-')
    plt.ylabel('$u(t)$')
    plt.grid(True)
plt.xlabel('$t$, s')

# 设置图形标题和图例
plt.subplot(n+m, 1, 1)
plt.title('Equality of discrete-time equivalent and original\n' +
          'continuous-time system subject to same control input')
plt.legend(['continuous-time system', 'discrete-time equivalent'])
# plt.show()

# 控制系统和整体系统的等效性
# 创建一个控制系统的模块图对象，连接系统和控制器
ctct_bd = BlockDiagram(ct_sys, ct_ctr)
# 在模块图中连接系统和控制器
ctct_bd.connect(ct_sys, ct_ctr)
# 在模块图中再次连接控制器和系统，形成闭环
ctct_bd.connect(ct_ctr, ct_sys)
# 对模块图进行仿真，返回仿真结果
ctct_res = ctct_bd.simulate(Tsim)

# 创建连续时间下的等效系统对象，使用状态空间表达式 (Ac - Bc @ Kc)
cteq_sys = LTISystem(Ac - Bc @ Kc, np.zeros((n, 0)))
# 设置等效系统的初始条件
cteq_sys.initial_condition = ic
# 对等效系统进行仿真，返回仿真结果
cteq_res = cteq_sys.simulate(Tsim)

# 绘制新的图形窗口
plt.figure()
# 绘制状态变量的图像
for st in range(n):
    plt.subplot(n+m, 1, st+1)
    # 绘制连续时间模块图仿真结果中的状态变量
    plt.plot(ctct_res.t, ctct_res.y[:, st], '+')
    # 绘制等效系统仿真结果中的状态变量
    plt.plot(cteq_res.t, cteq_res.y[:, st], 'x')
    # 设置纵轴标签
    plt.ylabel('$x_{}(t)$'.format(st+1))
    # 显示网格线
    plt.grid(True)
# 绘制输入变量的图像
for st in range(m):
    plt.subplot(n+m, 1, st+n+1)
    # 绘制连续时间模块图仿真结果中的输入变量
    plt.plot(ctct_res.t, ctct_res.y[:, st+n], '+')
    # 绘制等效系统仿真结果中的控制输入变量
    plt.plot(cteq_res.t, -(Kc@cteq_res.y.T).T, 'x')
    # 设置纵轴标签
    plt.ylabel('$u(t)$')
    # 显示网格线
    plt.grid(True)
# 设置横轴标签
plt.xlabel('$t$, s')
# 在第一个子图中设置标题，说明系统在反馈控制下的等效闭环连续时间行为
plt.subplot(n+m, 1, 1)
plt.title('Equality of system under feedback control and\n' +
          'equivalent closed-loop, continuous time')
# 设置图例，解释连续时间系统的状态方程和等效闭环系统的状态方程
plt.legend([r'$\dot{x}(t) = A\ x(t) + B\ u(t)$; $u(t) = K\ x(t)$',
            r'$\dot{x}(t) = (A - B\ K)\ x(t)$'])

# 创建离散时间下的等效系统对象，使用状态空间表达式 (Ad - Bd @ Kd)，并指定采样周期 dT
dteq_sys = LTISystem(Ad - Bd @ Kd, np.zeros((n, 0)), dt=dT)
# 设置离散时间等效系统的初始条件
dteq_sys.initial_condition = ic
# 对离散时间等效系统进行仿真，返回仿真结果
dteq_res = dteq_sys.simulate(Tsim)

# 绘制新的图形窗口
plt.figure()
# 绘制状态变量的图像
for st in range(n):
    plt.subplot(n+m, 1, st+1)
    # 绘制连续时间模块图仿真结果中的状态变量
    plt.plot(dtdt_res.t, dtdt_res.y[:, st])
    # 绘制离散时间等效系统仿真结果中的状态变量
    plt.plot(dteq_res.t, dteq_res.y[:, st])
    # 显示网格线
    plt.grid(True)
    # 设置纵轴标签
    plt.ylabel('$x_{}(t)$'.format(st+1))
# 绘制输入变量的图像
for st in range(m):
    plt.subplot(n+m, 1, st+n+1)
    # 绘制连续时间模块图仿真结果中的输入变量
    plt.plot(dtdt_res.t, dtdt_res.y[:, st+n])
    # 绘制离散时间等效系统仿真结果中的控制输入变量
    plt.plot(dteq_res.t, -(Kd@dteq_res.y.T).T)
    # 显示网格线
    plt.grid(True)
    # 设置纵轴标签
    plt.ylabel('$u(t)$')
# 设置横轴标签
plt.xlabel('$t$, s')
# 在第一个子图中设置标题，说明系统在反馈控制下的等效闭环离散时间行为
plt.subplot(n+m, 1, 1)
plt.title('Equality of system under feedback control and\n' +
          'equivalent closed-loop, discrete time')
# 设置图例，解释离散时间系统的状态方程和等效闭环系统的状态方程
plt.legend([r'$x[k+1] = A\ x[k] + B\ u[k]$; $u[k] = K\ x[k]$',
            r'$x[k+1] = (A - B\ K)\ x[k]$'])
# 显示图形
plt.show()
```