# `.\simupy\simupy\block_diagram.py`

```
# 导入必要的库 numpy 和 warnings
import numpy as np
import warnings

# 从 simupy.utils 中导入 callable_from_trajectory 函数
from simupy.utils import callable_from_trajectory

# 从 scipy.integrate 中导入 ode 类
from scipy.integrate import ode

# 从 scipy.optimize 中导入 brentq 函数
from scipy.optimize import brentq

# 默认的积分器类为 ode
DEFAULT_INTEGRATOR_CLASS = ode

# 默认的积分器选项字典，包括积分器名称和数值参数设置
DEFAULT_INTEGRATOR_OPTIONS = {
    "name": "dopri5",
    "rtol": 1e-6,
    "atol": 1e-12,
    "nsteps": 500,
    "max_step": 0.0,
}

# 默认的事件查找器设定为 brentq 函数
DEFAULT_EVENT_FINDER = brentq

# 默认的事件查找器选项字典，包括容差和最大迭代次数设置
DEFAULT_EVENT_FIND_OPTIONS = {
    "xtol": 2e-12,
    "rtol": 8.8817841970012523e-16,
    "maxiter": 100,
}

# 当模拟结果出现 NaN 时的警告消息模板
nan_warning_message = (
    "Simulation encountered NaN outputs and quit during"
    + " {}. This may have been intentional! NaN outputs at "
    + "time t={}, state x={}, output y={}"
)

# SimulationResult 类定义，用于收集模拟结果轨迹
class SimulationResult(object):
    """
    A simple class to collect simulation result trajectories.

    Attributes
    ----------
    t : array of times
    x : array of states
    y : array of outputs
    e : array of events
    """

    # 最大允许分配的内存大小
    max_allocation = 2 ** 7

    def to_file(self, file, compress=True):
        """
        Save attributes of ``SimulationResult`` object to binary file in NumPy ``.npz`` 
        format so that the object can be re-created.

        Parameters
        ----------
        file : file, str, or pathlib.Path
            File or filename to which the data is saved.  If file is a file-object,
            then the filename is unchanged.  If file is a string or Path, a ``.npz``
            extension will be appended to the file name if it does not already
            have one.
        compress : bool, optional
            Whether to compress the archive of attributes.  Optional, 
            default is True.

        See Also
        --------
        from_file
        """
        # 准备保存的数据字典
        save_dict = dict(t=self.t, x=self.x, y=self.y, e=self.e)
        # 根据 compress 参数决定是否压缩保存数据
        if compress:
            np.savez_compressed(file, **save_dict)
        else:
            np.savez(file, **save_dict)

    @classmethod
    def from_file(cls, file):
        """
        Create a ``SimulationResult`` object from a NumPy archive containing the
        necessary attributes.

        Parameters
        ----------
        file : file-like object, string, or pathlib.Path
            The file to read. File-like objects must support the
            ``seek()`` and ``read()`` methods. Pickled files require that the
            file-like object support the ``readline()`` method as well.

        See Also
        --------
        to_file
        """

        # 创建一个空的 SimulationResult 对象
        res = cls(0, 0, 0, np.array([0.,0.]))
        # 使用 np.load 加载文件数据
        with np.load(file) as data:
            res.t = data["t"]
            res.x = data["x"]
            res.y = data["y"]
            res.e = data["e"]
        return res
    # 初始化函数，设置对象的初始状态
    def __init__(self, dim_states, dim_outputs, num_events, tspan, initial_size=0):
        # 如果初始大小为0，则使用时间跨度的大小作为初始大小
        if initial_size == 0:
            initial_size = tspan.size
        # 初始化时间数组，空的numpy数组
        self.t = np.empty(initial_size)
        # 初始化状态数组，空的numpy二维数组
        self.x = np.empty((initial_size, dim_states))
        # 初始化输出数组，空的numpy二维数组
        self.y = np.empty((initial_size, dim_outputs))
        # 初始化事件数组，空的numpy二维数组
        self.e = np.empty((initial_size, num_events))
        # 初始化结果索引
        self.res_idx = 0
        # 设置时间跨度
        self.tspan = tspan
        # 设置初始时间
        self.t0 = tspan[0]
        # 设置结束时间
        self.tF = tspan[-1]

    # 分配空间函数，用于在需要时扩展数组容量
    def allocate_space(self, t):
        # 计算需要增加的行数，确保足够的空间存储数据
        more_rows = int((self.tF - t) * self.t.size / (t - self.t0)) + 1
        # 限制增加的行数不超过设定的最大值
        more_rows = max(min(more_rows, self.max_allocation), 1)

        # 在时间、状态、输出和事件数组末尾增加新的空行
        self.t = np.r_[self.t, np.empty(more_rows)]
        self.x = np.r_[self.x, np.empty((more_rows, self.x.shape[1]))]
        self.y = np.r_[self.y, np.empty((more_rows, self.y.shape[1]))]
        self.e = np.r_[self.e, np.empty((more_rows, self.e.shape[1]))]

    # 新结果函数，用于添加新的时间点、状态、输出和事件数据
    def new_result(self, t, x, y, e=None):
        # 如果结果索引超过当前时间数组大小，则扩展数组空间
        if self.res_idx >= self.t.size:
            self.allocate_space(t)
        # 将传入的数据存储到相应的时间、状态、输出和事件数组中
        self.t[self.res_idx] = t
        self.x[self.res_idx, :] = x
        self.y[self.res_idx, :] = y.squeeze()  # 去除y的多余维度
        if e is not None:
            self.e[self.res_idx, :] = e
        else:
            self.e[self.res_idx, :] = np.zeros(self.e.shape[1])  # 如果没有事件数据，则填充零向量
        self.res_idx += 1  # 更新结果索引

    # 最后结果函数，用于获取最后n个时间点、状态、输出和事件数据
    def last_result(self, n=1, copy=False):
        # 将n限制在1到结果索引之间
        n = np.clip(n, 1, self.res_idx)
        if copy:
            # 如果复制标志为True，则返回复制的数据副本
            return (
                np.copy(self.t[self.res_idx - n]),   # 时间数据副本
                np.copy(self.x[self.res_idx - n, :]),  # 状态数据副本
                np.copy(self.y[self.res_idx - n, :]),  # 输出数据副本
                np.copy(self.e[self.res_idx - n, :]),  # 事件数据副本
            )
        else:
            # 否则直接返回数据引用
            return (
                self.t[self.res_idx - n],   # 时间数据引用
                self.x[self.res_idx - n, :],  # 状态数据引用
                self.y[self.res_idx - n, :],  # 输出数据引用
                self.e[self.res_idx - n, :],  # 事件数据引用
            )
class SimulationMixin:
    def computation_step(self, t, state, output=None, do_events=False):
        """
        callable to compute system outputs and state derivatives
        """
        # 计算系统的状态方程，完整系统的情况下为 x[t_k'] = f(t_k, x[t_k], u[t_k])
        output = (
            output if output is not None else self.output_equation_function(t, state)
        )
        # 计算状态方程的导数
        dxdt = self.state_equation_function(t, state, output)

        if do_events:
            if self.num_events:
                if isinstance(self, BlockDiagram):
                    # 如果是 BlockDiagram 类型，计算事件方程
                    events = self.event_equation_function(t, state, output)
                else:
                    # 否则只计算基础的事件方程
                    events = self.event_equation_function(t, state, )
            else:
                # 如果没有事件，返回空数组
                events = np.zeros(0)

            return dxdt, output, events

        return dxdt, output

    def simulate(
        self,
        tspan,
        integrator_class=DEFAULT_INTEGRATOR_CLASS,
        integrator_options=DEFAULT_INTEGRATOR_OPTIONS,
        event_finder=DEFAULT_EVENT_FINDER,
        event_find_options=DEFAULT_EVENT_FIND_OPTIONS,
    ):
        """
        Simulates the system over a given time span using specified integrators and event finders.
        """
        # 实现系统仿真过程的方法
        pass


class BlockDiagram(SimulationMixin):
    """
    A block diagram of dynamical systems with their connections which can be
    numerically simulated.
    """

    def __init__(self, *systems):
        """
        Initialize a BlockDiagram, with an optional list of systems to start
        the diagram.
        """
        # 初始化 BlockDiagram 类的对象
        self.systems = np.array([], dtype=object)
        self.connections = np.array([], dtype=np.bool_).reshape((0, 0))

        self.dts = np.array([], dtype=np.float_)

        self.events = np.array([], dtype=np.int_)
        self.states = np.array([], dtype=np.int_)
        self.inputs = np.array([], dtype=np.int_)
        self.outputs = np.array([], dtype=np.int_)

        self.cum_inputs = np.array([0], dtype=np.int_)
        self.cum_outputs = np.array([0], dtype=np.int_)
        self.cum_states = np.array([0], dtype=np.int_)
        self.cum_events = np.array([0], dtype=np.int_)

        self.inputs = np.array([], dtype=np.bool_).reshape((0, 0))
        self.dim_input = 0

        for sys in systems:
            self.add_system(sys)

    @property
    def initial_condition(self):
        """
        Returns the initial conditions as a concatenated array of all subsystems' initial conditions.
        """
        x0 = np.zeros(self.dim_state)  # TODO: pre-allocate?
        for sysidx in range(len(self.systems)):
            sys = self.systems[sysidx]
            state_start = self.cum_states[sysidx]
            state_end = self.cum_states[sysidx + 1]
            x0[state_start:state_end] = sys.initial_condition
        return x0

    @property
    def dim_state(self):
        """
        Returns the total dimension of the state vector by summing up all subsystems' state dimensions.
        """
        return self.cum_states[-1]

    @property
    def num_events(self):
        """
        Returns the total number of events across all subsystems.
        """
        return self.cum_events[-1]

    @property
    def dim_output(self):
        """
        Returns the total dimension of the output vector by summing up all subsystems' output dimensions.
        """
        # TODO: allow internal outputs to be "closed"? For now, no
        return self.cum_outputs[-1]

    # @property
    # def dt(self):
    #     return self._dt
    # 准备系统集成，计算输出以供完整系统使用，y[t_k]=h(t_k,x[t_k])
    def prepare_to_integrate(self, t, state):
        # 初始化输出数组为全零数组，长度为系统输出维度
        output = np.zeros(self.dim_output)

        # 确定具有状态的系统在系统列表中的索引
        self._stateful_idx = np.where((self.states > 0))[0]
        # 确定无状态的系统在系统列表中的索引，并反向排列
        self._stateless_idx = np.where((self.states == 0))[0][::-1]
        # 确定无状态且有事件的系统在系统列表中的索引
        self._stateless_event_idx = np.where((self.states == 0) & self.events)[0]
        # 确定有状态且有事件的系统在系统列表中的索引
        self._stateful_event_idx = np.where((self.states > 0) & self.events)[0]

        # 遍历具有状态的系统
        for sysidx in self._stateful_idx:
            # 获取当前系统对象
            sys = self.systems[sysidx]
            # 计算当前系统在输出数组中的起始和结束位置
            output_start = self.cum_outputs[sysidx]
            output_end = self.cum_outputs[sysidx + 1]
            # 计算当前系统在状态向量中的起始和结束位置
            state_start = self.cum_states[sysidx]
            state_end = self.cum_states[sysidx + 1]

            # 提取当前系统的状态值
            state_values = state[state_start:state_end]
            # 调用系统的准备集成方法，计算输出并转换为一维数组
            output[output_start:output_end] = sys.prepare_to_integrate(
                t, state_values
            ).reshape(-1)

        # 遍历无状态的系统
        # 计算输出为无记忆系统，y[t_k]=h(t_k,u[t_k])
        for sysidx in self._stateless_idx:
            # 获取当前系统对象
            sys = self.systems[sysidx]
            # 计算当前系统在输出数组中的起始和结束位置
            output_start = self.cum_outputs[sysidx]
            output_end = self.cum_outputs[sysidx + 1]
            # 计算当前系统在输入向量中的起始和结束位置
            input_start = self.cum_inputs[sysidx]
            input_end = self.cum_inputs[sysidx + 1]

            # 初始化当前系统的输入值数组
            input_values = np.zeros(sys.dim_input)

            # 获取连接矩阵中非零元素的行列索引
            input_index, output_index = np.where(
                self.connections[:, input_start:input_end].T
            )
            # 将对应输出值赋给当前系统的输入值数组
            input_values[input_index] = output[output_index]

            # 获取输入矩阵中作为系统输入的行列索引
            input_index, as_sys_input_index = np.where(
                self.inputs[:, input_start:input_end].T
            )

            # 如果存在作为系统输入的索引
            if as_sys_input_index.size:
                # TODO: 待实现的功能
                input_values[input_index] = 0.0  # input_[as_sys_input_index]

            # 如果当前系统有输入维度
            if sys.dim_input:
                # 调用系统的准备集成方法，计算输出并更新到输出数组中
                output[output_start:output_end] = sys.prepare_to_integrate(
                    t, input_values
                )
            else:
                # 调用系统的准备集成方法，计算输出并更新到输出数组中
                output[output_start:output_end] = sys.prepare_to_integrate(t)
        
        # 返回计算得到的输出数组
        return output
    def create_input(self, to_system_input, channels=[], inputs=[]):
        """
        Create or use input channels to use block diagram as a subsystem.

        Parameters
        ----------
        channels : list-like
            Selector index of the input channels to connect.
        to_system_input : dynamical system
            The system (already added to BlockDiagram) to which inputs will be
            connected. Note that any previous input connections will be
            over-written.
        inputs : list-like, optional
            Selector index of the inputs to connect. If not specified or of
            length 0, will connect all of the inputs.
        """
        # 将 channels 转换为 NumPy 数组
        channels = np.asarray(channels)
        # 如果 channels 为空，则抛出数值错误
        if len(channels) == 0:
            raise ValueError("Cannot create input without specifying channel")
        # 如果 channels 的最小值小于 0，则抛出数值错误
        if np.min(channels) < 0:
            raise ValueError("Cannot create input channel < 0")

        # 如果 inputs 为空，则连接所有系统输入
        if len(inputs) == 0:
            inputs = np.arange(to_system_input.dim_input)
        else:
            # 将 inputs 转换为 NumPy 数组
            inputs = np.asarray(inputs)
        # 将 inputs 调整为与 to_system_input 相关的索引
        inputs = inputs + self.cum_inputs[np.where(self.systems == to_system_input)]

        # 检查 channels 和 inputs 的长度，确保能够广播连接
        if len(channels) != len(inputs) and len(channels) != 1:
            raise ValueError("Cannot broadcast channels to inputs")

        # 如果 channels 中的最大值超过当前系统输入维度的最大索引
        if np.max(channels) > self.dim_input - 1:
            # 扩展 self.inputs 数组以容纳新的 channels
            self.inputs = np.pad(
                self.inputs,
                ((0, np.max(channels) - self.dim_input + 1), (0, 0)),
                "constant",
                constant_values=0,
            )
            # 更新系统输入维度为 channels 的最大索引加一
            self.dim_input = np.max(channels) + 1

        # 将 self.inputs 和 self.connections 中的指定 inputs 索引位置设置为 False
        self.inputs[:, inputs] = False
        self.connections[:, inputs] = False

        # 将指定 channels 和 inputs 索引位置设置为 True，表示连接输入通道
        self.inputs[channels, inputs] = True
    def connect(self, from_system_output, to_system_input, outputs=[], inputs=[]):
        """
        Connect systems in the block diagram.

        Parameters
        ----------
        from_system_output : dynamical system
            The system (already added to BlockDiagram) from which outputs will
            be connected. Note that the outputs of a system can be connected to
            multiple inputs.
        to_system_input : dynamical system
            The system (already added to BlockDiagram) to which inputs will be
            connected. Note that any previous input connections will be
            over-written.
        outputs : list-like, optional
            Selector index of the outputs to connect. If not specified or of
            length 0, will connect all of the outputs.
        inputs : list-like, optional
            Selector index of the inputs to connect. If not specified or of
            length 0, will connect all of the inputs.
        """
        # 如果未指定连接的输出，将所有输出连接起来
        if len(outputs) == 0:
            outputs = np.arange(from_system_output.dim_output)
        else:
            outputs = np.asarray(outputs)
        # 将输出索引与系统对应的输出累计值相加，得到真实的输出索引
        outputs = (
            outputs + self.cum_outputs[np.where(self.systems == from_system_output)]
        )

        # 如果未指定连接的输入，将所有输入连接起来
        if len(inputs) == 0:
            inputs = np.arange(to_system_input.dim_input)
        else:
            inputs = np.asarray(inputs)
        # 将输入索引与系统对应的输入累计值相加，得到真实的输入索引
        inputs = inputs + self.cum_inputs[np.where(self.systems == to_system_input)]

        # TODO: Check that this can be broadcast correctly
        # TODO: 检查这个操作是否能正确广播

        # 将输入矩阵中对应的输入位置设为False
        self.inputs[:, inputs] = False
        # 将连接矩阵中对应的输入位置设为False
        self.connections[:, inputs] = False

        # 将连接矩阵中对应的输出和输入位置设为True，表示连接
        self.connections[outputs, inputs] = True
    def add_system(self, system):
        """
        Add a system to the block diagram

        Parameters
        ----------
        system : dynamical system
            System to add to BlockDiagram
        """
        # 将系统添加到系统数组中
        self.systems = np.append(self.systems, system)
        # 将系统状态维度添加到状态数组中
        self.states = np.append(
            self.states,
            system.dim_state,
        )
        # 计算累积状态维度并添加到累积状态数组中
        self.cum_states = np.append(
            self.cum_states, self.cum_states[-1] + system.dim_state
        )
        # 计算累积输入维度并添加到累积输入数组中
        self.cum_inputs = np.append(
            self.cum_inputs, self.cum_inputs[-1] + system.dim_input
        )
        # 计算累积输出维度并添加到累积输出数组中
        self.cum_outputs = np.append(
            self.cum_outputs, self.cum_outputs[-1] + system.dim_output
        )

        # 添加系统事件数到事件数组中
        self.events = np.append(
            self.events,
            system.num_events,
        )
        # 计算累积事件数并添加到累积事件数组中
        self.cum_events = np.append(
            self.cum_events, self.cum_events[-1] + system.num_events
        )

        # 将系统的时间步长（如果定义了）添加到时间步长数组中
        self.dts = np.append(self.dts, getattr(system, "dt", 0))
        # 如果时间步长数组中存在非零值，则将最小非零值作为整体时间步长
        if np.any(self.dts != 0.0):
            self.dt = np.min(self.dts[self.dts != 0.0])
        else:
            self.dt = 0.0

        # 扩展连接矩阵以适应新增系统的输出和输入
        self.connections = np.pad(
            self.connections,
            ((0, system.dim_output), (0, system.dim_input)),
            "constant",
            constant_values=0,
        )
        # 扩展输入矩阵以适应新增系统的输入
        self.inputs = np.pad(
            self.inputs, ((0, 0), (0, system.dim_input)), "constant", constant_values=0
        )

    def output_equation_function(
        self,
        t,
        state_or_input,
        ):
            # 如果 dim_state 为真，则 state_or_input 是状态值，创建一个全零输出数组
            if self.dim_state:
                state = state_or_input
                output = np.zeros(self.dim_output)
            else:
                # 否则，state_or_input 直接作为输出
                output = state_or_input

        # 计算完整系统的输出，即 y[t_k]=h(t_k,x[t_k])
        for sysidx in self._stateful_idx:
            sys = self.systems[sysidx]
            output_start = self.cum_outputs[sysidx]
            output_end = self.cum_outputs[sysidx + 1]
            state_start = self.cum_states[sysidx]
            state_end = self.cum_states[sysidx + 1]

            # 获取当前系统的状态值
            state_values = state[state_start:state_end]
            # 计算系统的输出并填充到相应位置
            output[output_start:output_end] = sys.output_equation_function(
                t, state_values
            ).reshape(-1)

        # 计算无记忆系统的输出，即 y[t_k]=h(t_k,u[t_k])
        for sysidx in self._stateless_idx:
            sys = self.systems[sysidx]
            output_start = self.cum_outputs[sysidx]
            output_end = self.cum_outputs[sysidx + 1]
            input_start = self.cum_inputs[sysidx]
            input_end = self.cum_inputs[sysidx + 1]

            # 创建全零输入数组
            input_values = np.zeros(sys.dim_input)

            # 获取连接矩阵中的非零元素的索引
            input_index, output_index = np.where(
                self.connections[:, input_start:input_end].T
            )
            # 将相应的输出值赋给输入数组
            input_values[input_index] = output[output_index]

            # 获取输入作为系统输入的索引
            input_index, as_sys_input_index = np.where(
                self.inputs[:, input_start:input_end].T
            )

            # 如果有系统作为输入，则将其赋值为零
            if as_sys_input_index.size:
                input_values[input_index] = 0.0  # input_[as_sys_input_index]

            # 根据系统是否有输入，计算系统的输出并填充到相应位置
            if sys.dim_input:
                output[output_start:output_end] = sys.output_equation_function(
                    t, input_values
                ).reshape(-1)
            else:
                output[output_start:output_end] = sys.output_equation_function(
                    t
                ).reshape(-1)

        # 返回计算得到的输出
        return output
    # 定义状态方程函数，包括时间参数t，系统状态state，输入input_和输出output参数
    def state_equation_function(self, t, state, input_=None, output=None):
        # 如果输出output参数为None，则调用输出方程函数获取输出
        output = (
            output
            if output is not None
            else self.output_equation_function(
                t,
                state,
            )
        )

        # 初始化状态变量的变化率为零向量
        dxdt = np.zeros(self.dim_state)

        # 对每一个有状态的系统进行遍历
        for sysidx in self._stateful_idx:
            sys = self.systems[sysidx]

            # 获取当前系统的状态起始和结束索引
            state_start = self.cum_states[sysidx]
            state_end = self.cum_states[sysidx + 1]
            state_values = state[state_start:state_end]

            # 获取当前系统的输入起始和结束索引
            input_start = self.cum_inputs[sysidx]
            input_end = self.cum_inputs[sysidx + 1]

            # 初始化当前系统的输入值为零向量
            input_values = np.zeros(sys.dim_input)

            # 获取连接矩阵中非零元素所在的行列索引，用于提取对应的输出值作为输入值
            input_index, output_index = np.where(
                self.connections[:, input_start:input_end].T
            )
            input_values[input_index] = output[output_index]

            # 获取输入矩阵中非零元素所在的行列索引，用于提取输入input_的值
            input_index, as_sys_input_index = np.where(
                self.inputs[:, input_start:input_end].T
            )

            # 如果输入input_中有值对应当前系统的输入，则更新输入值
            if as_sys_input_index.size:
                input_values[input_index] = input_[as_sys_input_index]

            # 如果当前系统有输入，调用当前系统的状态方程函数计算状态变化率
            if sys.dim_input:
                dxdt[state_start:state_end] = sys.state_equation_function(
                    t, state_values, input_values
                ).reshape(-1)
            # 如果当前系统没有输入，则只调用当前系统的状态方程函数计算状态变化率
            else:
                dxdt[state_start:state_end] = sys.state_equation_function(
                    t, state_values
                ).reshape(-1)

        # 返回状态变化率向量
        return dxdt
    # 定义事件方程函数，用于计算系统事件
    def event_equation_function(self, t, state_or_input, output=None):
        # 如果系统具有状态空间，则从参数中获取状态
        if self.dim_state:
            state = state_or_input
        # 如果未提供输出，则调用输出方程函数计算输出
        output = (
            output if output is not None else self.output_equation_function(t, state)
        )
        # 初始化事件数组，长度为系统定义的事件数目
        events = np.zeros(self.num_events)

        # 计算有状态系统的事件
        for sysidx in self._stateful_event_idx:
            sys = self.systems[sysidx]

            # 获取当前系统的事件起始和结束索引
            event_start = self.cum_events[sysidx]
            event_end = self.cum_events[sysidx + 1]

            # 获取当前系统的状态起始和结束索引，以及相应的状态值
            state_start = self.cum_states[sysidx]
            state_end = self.cum_states[sysidx + 1]
            state_values = state[state_start:state_end]

            # 调用系统对象的事件方程函数计算事件，并将结果重塑为一维数组
            events[event_start:event_end] = sys.event_equation_function(
                t, state_values
            ).reshape(-1)

        # 计算无状态系统的事件
        for sysidx in self._stateless_event_idx:
            sys = self.systems[sysidx]

            # 获取当前系统的事件起始和结束索引
            event_start = self.cum_events[sysidx]
            event_end = self.cum_events[sysidx + 1]

            # 获取当前系统的输入起始和结束索引
            input_start = self.cum_inputs[sysidx]
            input_end = self.cum_inputs[sysidx + 1]

            # 初始化输入值数组，长度为当前系统的输入维度
            input_values = np.zeros(sys.dim_input)

            # 使用连接矩阵获取当前系统的输入和对应的输出
            input_index, output_index = np.where(
                self.connections[:, input_start:input_end].T
            )
            input_values[input_index] = output[output_index]

            # 如果系统具有输入，则调用系统对象的事件方程函数计算事件；否则，仅调用事件方程函数
            if sys.dim_input:
                events[event_start:event_end] = sys.event_equation_function(
                    t, input_values
                ).reshape(-1)
            else:
                events[event_start:event_end] = sys.event_equation_function(t).reshape(
                    -1
                )

        # 返回计算得到的事件数组
        return events
        ):
            # 复制当前状态，以便进行状态更新
            next_state = state.copy()
            # 如果未提供输出，则使用输出方程计算输出
            output = (
                output if output is not None else self.output_equation_function(t, state)
            )
            # 找出哪些系统的事件通道已经跨过阈值
            sys_indices = np.argmax(event_channels[:, None] < self.cum_events, axis=1) - 1
            # 获取唯一的系统索引
            unique_sys_indices = np.unique(sys_indices)
            # 遍历每个唯一的系统索引
            for sysidx in unique_sys_indices:
                # 计算每个系统的事件通道
                sys_event_channels = (
                    event_channels[sys_indices == sysidx] - self.cum_events[sysidx]
                )
                # 获取当前系统对象
                sys = self.systems[sysidx]
                # 获取当前系统的输出起始和结束索引
                output_start = self.cum_outputs[sysidx]
                output_end = self.cum_outputs[sysidx + 1]
                # 获取当前系统的输入起始和结束索引
                input_start = self.cum_inputs[sysidx]
                input_end = self.cum_inputs[sysidx + 1]
                # 根据连接矩阵获取当前系统的输入值
                input_values = output[
                    np.where(self.connections[:, input_start:input_end].T)[1]
                ]
                # 获取当前系统的状态起始和结束索引
                state_start = self.cum_states[sysidx]
                state_end = self.cum_states[sysidx + 1]
                # 获取当前系统的状态值
                state_values = state[state_start:state_end]
                # 根据系统的状态和输入维度调用更新方程
                if sys.dim_state and sys.dim_input:
                    update_return_value = sys.update_equation_function(
                        t,
                        state_values,
                        input_values,
                        event_channels=sys_event_channels,
                    )
                elif sys.dim_state:
                    update_return_value = sys.update_equation_function(
                        t,
                        state_values,
                        event_channels=sys_event_channels,
                    )
                elif sys.dim_input:
                    update_return_value = sys.update_equation_function(
                        t,
                        input_values,
                        event_channels=sys_event_channels,
                    )
                else:
                    update_return_value = sys.update_equation_function(
                        t,
                        event_channels=sys_event_channels,
                    )
                # 如果系统具有状态，更新下一个状态
                if sys.dim_state:
                    next_state[state_start:state_end] = update_return_value.reshape(-1)
                    # 计算当前系统的输出并更新到输出数组中
                    output[output_start:output_end] = sys.output_equation_function(
                        t, update_return_value
                    ).squeeze()
                elif sys.dim_input:
                    # 如果系统具有输入但无状态，计算当前系统的输出并更新到输出数组中
                    output[output_start:output_end] = sys.output_equation_function(
                        t, input_values
                    ).squeeze()
                else:
                    # 如果系统既无状态也无输入，计算当前系统的输出并更新到输出数组中
                    output[output_start:output_end] = sys.output_equation_function(
                        t
                    ).squeeze()
            # 返回更新后的状态数组
            return next_state
```