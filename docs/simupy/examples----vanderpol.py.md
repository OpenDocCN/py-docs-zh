# `.\simupy\examples\vanderpol.py`

```
# 导入NumPy库，用于数值计算
import numpy as np
# 导入SymPy库，用于符号计算
import sympy as sp
# 导入Matplotlib库，用于绘图
import matplotlib.pyplot as plt
# 导入simupy库中的符号动力系统相关模块
from simupy.systems.symbolic import DynamicalSystem, dynamicsymbols
# 导入simupy库中的块图模块和默认积分器选项
from simupy.block_diagram import BlockDiagram, DEFAULT_INTEGRATOR_OPTIONS
# 导入simupy库中的数组操作模块Array和r_
from simupy.array import Array, r_

# 设置默认积分器选项中的步数
DEFAULT_INTEGRATOR_OPTIONS['nsteps'] = 1000

# 定义符号变量x1, x2，并将其包装在数组Array中
x = x1, x2 = Array(dynamicsymbols('x1:3'))

# 定义符号变量mu
mu = sp.symbols('mu')

# 定义系统的状态方程，使用r_将表达式组成向量
state_equation = r_[x2, -x1 + mu * (1 - x1**2) * x2]
# 定义系统的输出方程，使用r_将表达式组成向量
output_equation = r_[x1**2 + x2**2, sp.atan2(x2, x1)]

# 创建动力系统对象DynamicalSystem，传入状态方程、状态变量、输出方程和常数值
sys = DynamicalSystem(
    state_equation,
    x,
    output_equation=output_equation,
    constants_values={mu: 5}
)

# 设置系统的初始条件为[1, 1]
sys.initial_condition = np.array([1, 1]).T

# 创建块图对象BD，传入动力系统对象sys
BD = BlockDiagram(sys)
# 对系统进行30秒的仿真
res = BD.simulate(30)

# 绘制系统状态随时间变化的图像
plt.figure()
plt.plot(res.t, res.x)
plt.legend([sp.latex(s, mode='inline') for s in sys.state])
plt.ylabel('$x_i(t)$')
plt.xlabel('$t$, s')
plt.title('system state vs time')
plt.tight_layout()
plt.show()

# 绘制系统的相图
plt.figure()
plt.plot(*res.x.T)
plt.xlabel('$x_1(t)$')
plt.ylabel('$x_2(t)$')
plt.title('phase portrait of system')
plt.tight_layout()
plt.show()

# 绘制系统输出随时间变化的图像
plt.figure()
plt.plot(res.t, res.y)
plt.legend([r'$\left| \mathbf{x}(t) \right|$', r'$\angle \mathbf{x} (t)$'])
plt.xlabel('$t$, s')
plt.title('system outputs vs time')
plt.tight_layout()
plt.show()
```