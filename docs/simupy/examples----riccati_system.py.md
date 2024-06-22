# `.\simupy\examples\riccati_system.py`

```
# 导入需要的库
import numpy as np
import matplotlib.pyplot as plt
from simupy.systems import SystemFromCallable, LTISystem
from simupy.utils import array_callable_from_vector_trajectory
from simupy.block_diagram import BlockDiagram
from simupy.matrices import (construct_explicit_matrix, matrix_subs,
                             system_from_matrix_DE)

# 定义符号矩阵
As = construct_explicit_matrix('a', 2, 2, dynamic=False)
Bs = construct_explicit_matrix('b', 2, 1, dynamic=False)
Qs = construct_explicit_matrix('q', 2, 2, symmetric=True, dynamic=False)
Rs = construct_explicit_matrix('r', 1, 1, symmetric=True, dynamic=False)
Ss = construct_explicit_matrix('s', 2, 2, symmetric=True, dynamic=True)
Gs = construct_explicit_matrix('g', 2, 1, dynamic=True)
rxs = construct_explicit_matrix('rx', 2, 1, dynamic=True)

# 定义数值矩阵
An = np.mat("0,1;-2,-3")
Bn = np.mat("0;1")
Qn = np.mat("1,0;0,0")
Rn = np.mat("0.02")

# 构造矩阵微分方程
Ssdot = (Qs+As.T*Ss+Ss*As-Ss*Bs*Rs.inv()*Bs.T*Ss)
Gsdot = As.T*Gs - Ss*Bs*Rs.inv()*Bs.T*Gs - Qs*rxs
SGdot = Ssdot.row_join(Gsdot)
SG = Ss.row_join(Gs)
SG_subs = dict(matrix_subs((As, An), (Bs, Bn), (Qs, Qn), (Rs, Rn)))

# 从矩阵微分方程和参考值构造系统
SG_sys = system_from_matrix_DE(SGdot, SG,  rxs, SG_subs)

# 定义参考输入的控制器
def ref_input_ctr(t, *args):
    return np.r_[2*(tF-t), 0]
ref_input_ctr_sys = SystemFromCallable(ref_input_ctr, 0, 2)

# 模拟带参考输入的矩阵微分方程
tF = 20
RiccatiBD = BlockDiagram(SG_sys, ref_input_ctr_sys)
RiccatiBD.connect(ref_input_ctr_sys, SG_sys)
sg_sim_res = RiccatiBD.simulate(tF)

# 创建可调用函数以插值模拟结果
sim_data_unique_t, sim_data_unique_t_idx = np.unique(
    sg_sim_res.t,
    return_index=True
)
mat_sg_result = array_callable_from_vector_trajectory(
    np.flipud(tF-sg_sim_res.t[sim_data_unique_t_idx]),
    np.flipud(sg_sim_res.x[sim_data_unique_t_idx]),
    SG_sys.state,
    SG
)
vec_sg_result = array_callable_from_vector_trajectory(
    np.flipud(tF-sg_sim_res.t[sim_data_unique_t_idx]),
    np.flipud(sg_sim_res.x[sim_data_unique_t_idx]),
    SG_sys.state,
    SG_sys.state
)

# 画出 S 组件
plt.figure()
plt.plot()
plt.plot(sg_sim_res.t, mat_sg_result(sg_sim_res.t)[:, [0, 0, 1], [0, 1, 1]])
plt.legend(['$s_{11}$', '$s_{12}$', '$s_{22}$'])
plt.title('unforced component of solution to Riccatti differential equation')
plt.xlabel('$t$, s')
plt.ylabel('$s_{ij}(t)$')
# 展示图形界面
plt.show()

# 绘制 G 分量
plt.figure()
plt.plot(sg_sim_res.t, mat_sg_result(sg_sim_res.t)[:, :, -1])
plt.title('forced component of solution to Riccatti differential equation')
plt.legend(['$g_1$', '$g_2$'])
plt.xlabel('$t$, s')
plt.ylabel('$g_{i}(t)$')
plt.show()

# 从 Riccati 微分代数方程的解构造控制器
tracking_controller = SystemFromCallable(
    lambda t, xx: -Rn**-1@Bn.T@(mat_sg_result(t)[:, :-1]@xx
                                + mat_sg_result(t)[:, -1]),
    2, 1
)
sys = LTISystem(An, Bn)  # 植物系统
sys.initial_condition = np.zeros((2, 1))

# 模拟控制器和植物
control_BD = BlockDiagram(sys, tracking_controller)
control_BD.connect(sys, tracking_controller)
control_BD.connect(tracking_controller, sys)
control_res = control_BD.simulate(tF)

# 绘制状态和参考值
plt.figure()
plt.plot(control_res.t, control_res.x, control_res.t, 2*control_res.t)
plt.title('reference and state vs. time')
plt.xlabel('$t$, s')
plt.legend([r'$x_1$', r'$x_2$', r'$r_1$'])
plt.ylabel('$x_{i}(t)$')
plt.show()

# 绘制控制输入
plt.figure()
plt.plot(control_res.t, control_res.y[:, -1])
plt.title('optimal control law vs. time')
plt.xlabel('$t$, s')
plt.ylabel('$u(t)$')
plt.show()
```