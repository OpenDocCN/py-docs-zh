# `.\simupy\examples\bouncing_ball.py`

```
import numpy as np  # 导入NumPy库，用于数值计算
import matplotlib.pyplot as plt  # 导入matplotlib库，用于绘图
import sympy as sp  # 导入SymPy库，用于符号计算
from sympy.physics.mechanics import dynamicsymbols  # 导入动力学符号计算模块
from simupy.discontinuities import SwitchedSystem  # 导入SwitchedSystem类，用于建立切换系统模型
from simupy import block_diagram  # 导入模块用于搭建模块图
from simupy.array import Array, r_  # 导入Array类和r_函数，用于数组操作

BlockDiagram = block_diagram.BlockDiagram  # 定义BlockDiagram为block_diagram模块中的BlockDiagram类
int_opts = block_diagram.DEFAULT_INTEGRATOR_OPTIONS.copy()  # 复制默认的积分器选项
find_opts = block_diagram.DEFAULT_EVENT_FIND_OPTIONS.copy()  # 复制默认的事件查找选项

"""
This example shows how to use a SwitchedSystem to model a bouncing ball. The
event detection accurately finds the point of impact, and the simulation
is generally accurate when the ball has sufficient energy. However, due to
numerical error, the simulation does show the ball chattering after all
the energy should have been dissipated.
"""

int_opts['rtol'] = 1E-12  # 设置相对误差容限
int_opts['atol'] = 1E-15  # 设置绝对误差容限
int_opts['nsteps'] = 1000  # 设置最大步数
int_opts['max_step'] = 2**-3  # 设置最大步长

find_opts['xtol'] = 1E-12  # 设置事件查找的容限
find_opts['maxiter'] = int(1E3)  # 设置最大迭代次数

x = x1, x2 = Array(dynamicsymbols('x_1:3'))  # 定义动力学符号数组x1, x2
mu, g = sp.symbols('mu g')  # 定义符号变量mu, g
constants = {mu: 0.8, g: 9.81}  # 定义常数字典
ic = np.r_[10, 15]  # 初始化条件数组
sys = SwitchedSystem(
    x1, Array([0]),
    state_equations=r_[x2, -g],
    state_update_equation=r_[sp.Abs(x1), -mu*x2],
    state=x,
    constants_values=constants,
    initial_condition=ic
)
bd = BlockDiagram(sys)  # 创建模块图对象
res = bd.simulate(
    20.36, integrator_options=int_opts, event_find_options=find_opts
)

expr_subs = constants.copy()  # 复制常数字典
expr_subs[x1] = ic[0]  # 替换符号变量x1的值为初始条件的第一个元素
expr_subs[x2] = ic[1]  # 替换符号变量x2的值为初始条件的第二个元素
v1 = sp.sqrt(x2**2 + 2*g*x1).evalf(subs=expr_subs)  # 计算速度v1
tstar = ((x2 + v1*(1 + mu)/(1-mu))/g).evalf(subs=expr_subs)  # 计算停止时间tstar

tvar = dynamicsymbols._t  # 获取时间符号变量
impact_eq = (x2*tvar - g*tvar**2/2 + x1).subs(expr_subs)  # 计算冲击方程
t_impact = sp.solve(impact_eq, tvar)[-1]  # 解决冲击方程得到冲击时间t_impact

# tstar is where the ball should come to a rest, however due to numerical
# error, it continues to chatter.
t_sel = (res.t < tstar*1.01)  # 选择时间窗口，用于绘制图像

plt.figure()  # 创建新的图形
plt.subplot(2, 1, 1)  # 创建第一个子图
plt.title('bouncing ball')  # 设置标题
plt.subplot(2, 1, 1)  # 创建第一个子图
plt.plot(res.t[t_sel], res.x[t_sel, 0])  # 绘制球的位置随时间变化的曲线
plt.plot(2*[t_impact], [0, np.max(res.x[t_sel, 0])])  # 标记冲击时间的垂直线
plt.ylabel('ball position, m')  # 设置y轴标签
plt.subplot(2, 1, 2)  # 创建第二个子图
plt.plot(res.t[t_sel], res.x[t_sel, 1])  # 绘制球的速度随时间变化的曲线
plt.ylabel('ball velocity, m/s')  # 设置y轴标签
plt.xlabel('time, s')  # 设置x轴标签
plt.tight_layout()  # 调整布局以防重叠
plt.show()  # 显示图形

plt.figure()  # 创建新的图形
plt.subplot(2, 1, 1)  # 创建第一个子图
plt.title('bouncing ball chatter')  # 设置标题
t_sel = (res.t > 20) & (res.t < tstar*1.03)  # 选择时间窗口，用于绘制图像
plt.subplot(2, 1, 1)  # 创建第一个子图
plt.plot(res.t[t_sel], res.x[t_sel, 0])  # 绘制球的位置随时间变化的曲线
plt.ylabel('ball position, m')  # 设置y轴标签
plt.subplot(2, 1, 2)  # 创建第二个子图
plt.plot(res.t[t_sel], res.x[t_sel, 1])  # 绘制球的速度随时间变化的曲线
plt.ylabel('ball velocity, m/s')  # 设置y轴标签
plt.xlabel('time, s')  # 设置x轴标签
plt.tight_layout()  # 调整布局以防重叠
plt.show()  # 显示图形
```