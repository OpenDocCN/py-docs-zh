# `.\simupy\examples\simple_saturation.py`

```
# 导入必要的库
import numpy as np
import sympy as sp
import matplotlib.pyplot as plt
from simupy.systems.symbolic import DynamicalSystem, dynamicsymbols
from simupy import block_diagram
from simupy.array import Array, r_
from simupy.discontinuities import SwitchedOutput

# 设置默认的积分器选项和事件查找选项
int_opts = block_diagram.DEFAULT_INTEGRATOR_OPTIONS.copy()
find_opts = block_diagram.DEFAULT_EVENT_FIND_OPTIONS.copy()

# 设置积分器选项的相对误差和绝对误差
int_opts['rtol'] = 1E-12
int_opts['atol'] = 1E-15
int_opts['nsteps'] = 1000

# 设置事件查找选项的绝对容差和最大迭代次数
find_opts['xtol'] = 1E-12
find_opts['maxiter'] = int(1E3)

# 创建一个简单的饱和块演示示例

# 创建一个简单的振荡器以生成正弦波
x = Array([dynamicsymbols('x')])  # 创建一个输出符号的占位符数组
tvar = dynamicsymbols._t  # 使用此符号作为时间符号
sin = DynamicalSystem(Array([sp.cos(tvar)]), x)  # 定义振荡器系统，以余弦波形为输入

# 设置饱和块的下限和上限
llim = -0.75
ulim = 0.75
saturation_limit = r_[llim, ulim]
saturation_output = r_['0,2', llim, x[0], ulim]
sat = SwitchedOutput(x[0], saturation_limit, output_equations=saturation_output, input_=x)

# 创建饱和块的块图，并将振荡器与饱和块连接
sat_bd = BlockDiagram(sin, sat)
sat_bd.connect(sin, sat)

# 对饱和块进行仿真，并绘制仿真结果
sat_res = sat_bd.simulate(2*np.pi, integrator_options=int_opts, event_find_options=find_opts)
plt.figure()
plt.plot(sat_res.t, sat_res.y[:, 0])
plt.plot(sat_res.t, sat_res.y[:, 1])
plt.xlabel('$t$, s')
plt.ylabel('$x(t)$')
plt.title('simple saturation demonstration')
plt.tight_layout()
plt.show()

# 设置死区块的阈值
eps = 0.25
deadband_limit = r_[-eps, eps]
deadband_output = r_['0,2', x[0]+eps, 0, x[0]-eps]
ded = SwitchedOutput(x[0], deadband_limit, output_equations=deadband_output, input_=x)

# 创建死区块的块图，并将振荡器与死区块连接
ded_bd = BlockDiagram(sin, ded)
ded_bd.connect(sin, ded)

# 对死区块进行仿真，并绘制仿真结果
ded_res = ded_bd.simulate(2*np.pi, integrator_options=int_opts, event_find_options=find_opts)
plt.figure()
plt.plot(ded_res.t, ded_res.y[:, 0])
plt.plot(ded_res.t, ded_res.y[:, 1])
plt.xlabel('$t$, s')
plt.ylabel('$x(t)$')
plt.title('simple deadband demonstration')
plt.tight_layout()
plt.show()
```