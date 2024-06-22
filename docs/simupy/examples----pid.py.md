# `.\simupy\examples\pid.py`

```
# 导入必要的库
import numpy as np
from scipy import signal, linalg
from simupy.systems import LTISystem, SystemFromCallable
from simupy.block_diagram import BlockDiagram
import matplotlib.pyplot as plt

# 构造二阶系统状态和输入矩阵
m = 1
d = 1
b = 1
k = 1
Ac = np.c_[[0, -k/m], [1, -b/m]]
Bc = np.r_[0, d/m].reshape(-1, 1)

# 增广状态和输入矩阵以添加积分误差状态
A_aug = np.hstack((np.zeros((3,1)), np.vstack((np.r_[1, 0], Ac)))
B_aug = np.hstack(( np.vstack((0, Bc)), -np.eye(3,1)))

# 构造系统
aug_sys = LTISystem(A_aug, B_aug,)

# 构造 PID 系统
Kc = 1
tau_I = 1
tau_D = 1
K = -np.r_[Kc/tau_I, Kc, Kc*tau_D].reshape((1,3))
pid = LTISystem(K)

# 构造参考信号
ref = SystemFromCallable(lambda *args: np.ones(1), 0, 1)

# 创建模块图
BD = BlockDiagram(aug_sys, pid, ref)
BD.connect(aug_sys, pid) # PID 需要反馈
BD.connect(pid, aug_sys, inputs=[0]) # PID 输出到系统控制输入
BD.connect(ref, aug_sys, inputs=[1]) # 参考输出到系统命令输入

res = BD.simulate(10) # 模拟

# 绘图
plt.figure()
plt.plot(res.t, res.y[:, 0], label=r'$\int x$')
plt.plot(res.t, res.y[:, 1], label='$x$')
plt.plot(res.t, res.y[:, 2], label=r'$\dot{x}$')
plt.plot(res.t, res.y[:, 3], label='$u$')
plt.plot(res.t, res.y[:, 4], label='$x_c$')
plt.legend()
plt.show()
```