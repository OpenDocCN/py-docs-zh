# `.\simupy\simupy\systems\__init__.py`

```
# 导入 NumPy 库，用于数值计算
import numpy as np
# 导入 SimulationMixin 类，用于模拟仿真的块图形式
from simupy.block_diagram import SimulationMixin
# 导入警告模块，用于输出警告信息
import warnings

# 当状态维度大于零时，要求 DynamicalSystem 必须有 state_equation_function 的消息
need_state_equation_function_msg = ("if dim_state > 0, DynamicalSystem must"
                                    + " have a state_equation_function")
# 当状态维度等于零时，要求 DynamicalSystem 必须有 output_equation_function 的消息
need_output_equation_function_msg = ("if dim_state == 0, DynamicalSystem must"
                                     + " have an output_equation_function")
# 零维输出时的消息，一个 DynamicalSystem 必须提供输出
zero_dim_output_msg = "A DynamicalSystem must provide an output"

# 定义一个函数 full_state_output，作为提供完整状态输出的 drop-in output_equation_function
def full_state_output(t, x, *args):
    """
    A drop-in ``output_equation_function`` for stateful systems that provide
    output the full state directly.
    """
    return x


# 定义一个类 DynamicalSystem，继承自 SimulationMixin 类
class DynamicalSystem(SimulationMixin):
    """
    A dynamical system which models systems of the form::

        xdot(t) = state_equation_function(t,x,u)
        y(t) = output_equation_function(t,x)

    or::

        y(t) = output_equation_function(t,u)

    These could also represent discrete-time systems, in which case xdot(t)
    represents x[k+1].

    This can also model discontinuous systems. Discontinuities must occur on
    zero-crossings of the ``event_equation_function``, which take the same
    arguments as ``output_equation_function``, depending on ``dim_state``.
    At the zero-crossing, ``update_equation_function`` is called with the same
    arguments. If ``dim_state`` > 0, the return value of
    ``update_equation_function`` is used as the state of the system immediately
    after the discontinuity.
    """
    def __init__(self, state_equation_function=None,
                 output_equation_function=None, event_equation_function=None,
                 update_equation_function=None, dim_state=0, dim_input=0,
                 dim_output=0, num_events=0, dt=0, initial_condition=None):
        """
        Parameters
        ----------
        state_equation_function : callable, optional
            系统状态方程的导数（或更新方程）。如果 ``dim_state`` 为零，则不需要。
        output_equation_function : callable, optional
            系统的输出方程。系统必须有一个 ``output_equation_function``。如果未设置，则使用完整状态输出。
        event_equation_function : callable, optional
            决定不连续性发生时刻的函数。
        update_equation_function : callable, optional
            在发生不连续性时调用的函数。
        dim_state : int, optional
            系统状态的维度。可选，默认为 0。
        dim_input : int, optional
            系统输入的维度。可选，默认为 0。
        dim_output : int, optional
            系统输出的维度。可选，默认为 dim_state。
        num_events : int, optional
            系统事件函数的维度。可选，默认为 0。
        dt : float, optional
            系统的采样率。可选，默认为 0，表示连续时间系统。
        initial_condition : array_like of numerical values, optional
            系统的初始条件，可以是数值数组或矩阵。默认为与状态维度相同的零向量。
        """
        self.dim_state = dim_state
        self.dim_input = dim_input
        self.dim_output = dim_output or dim_state
        self.num_events = num_events

        self.state_equation_function = state_equation_function

        self.output_equation_function = (
            full_state_output
            if output_equation_function is None and self.dim_state > 0
            else output_equation_function
        )

        self.initial_condition = initial_condition

        if ((num_events != 0) and ((event_equation_function is None) or
        (update_equation_function is None))):
            raise ValueError("Cannot provide event_equation_function or " + 
                             "update_Equation_function with num_events == 0")

        self.event_equation_function = event_equation_function
        
        # TODO: 进行一些防御性检查和/或包装更新函数，以便消耗通道编号
        self.update_equation_function = update_equation_function

        self.dt = dt

        self.validate()

    @property
    def dt(self):
        """
        返回系统的采样周期。
        """
        return self._dt

    @dt.setter
    # 定义一个方法，用于设置仿真步长 dt
    def dt(self, dt):
        # 如果 dt 小于等于 0，则将 self._dt 设置为 0 并返回
        if dt <= 0:
            self._dt = 0
            return
        # 如果已经有事件存在，抛出数值错误，说明不能同时设置 dt > 0 并使用非零 num_events 的事件 API
        if self.num_events != 0:
            raise ValueError("Cannot set dt > 0 and use event API " +
                             "with non-zero num_events")
        # 设置 num_events 为 1，并将 self._dt 设置为给定的 dt
        self.num_events = 1
        self._dt = dt
        # 设置事件方程的函数为一个 lambda 表达式，该函数根据时间 t 和参数 args 计算 sin 函数的值
        self.event_equation_function = lambda t, *args: np.atleast_1d(np.sin(np.pi*t/self.dt))
        # 保留当前状态方程和输出方程的引用
        self._state_equation_function = self.state_equation_function
        self._output_equation_function = self.output_equation_function
        # 设置状态方程的函数为一个 lambda 表达式，返回一个全零数组，数组长度为 dim_state
        self.state_equation_function = \
            lambda *args: np.zeros(self.dim_state)
        # 如果 dim_state 大于 0，则更新方程函数为原始状态方程函数
        if self.dim_state:
            self.update_equation_function = (
                lambda *args, event_channels=0: self._state_equation_function(*args)
            )
        else:
            # 如果 dim_state 为 0，则初始化 _prev_output 为 0，并定义更新方程函数为 _update_equation_function
            self._prev_output = 0
            def _update_equation_function(*args, event_channels=0):
                self._prev_output = self._output_equation_function(*args)

            self.update_equation_function = _update_equation_function
            # 输出方程函数为一个 lambda 表达式，返回 _prev_output 的值
            self.output_equation_function = lambda *args: self._prev_output


    @property
    # 定义一个属性方法，用于获取初始条件
    def initial_condition(self):
        # 如果 _initial_condition 为 None，则将其设置为一个全零数组，长度为 dim_state
        if self._initial_condition is None:
            self._initial_condition = np.zeros(self.dim_state)
        return self._initial_condition

    @initial_condition.setter
    # 定义初始条件的 setter 方法
    def initial_condition(self, initial_condition):
        # 如果 initial_condition 不为 None
        if initial_condition is not None:
            # 如果 initial_condition 是一个 numpy 数组，则获取其大小；否则获取其长度
            if isinstance(initial_condition, np.ndarray):
                size = initial_condition.size
            else:
                size = len(initial_condition)
            # 确保 initial_condition 的大小等于 dim_state
            assert size == self.dim_state
            # 将 initial_condition 转换为 numpy 数组，并按浮点数类型存储，reshape 成一维数组
            self._initial_condition = np.array(initial_condition,
                dtype=np.float_).reshape(-1)
        else:
            # 如果 initial_condition 为 None，则将 _initial_condition 设置为 None
            self._initial_condition = None

    # 定义一个方法，用于准备进行积分
    def prepare_to_integrate(self, t0, state_or_input=None):
        # 如果 dim_state 为 0 且 num_events 不为 0
        if not self.dim_state and self.num_events:
            # 如果 dim_input 不为 0，则调用 update_equation_function 方法，传入 t0 和 state_or_input 参数
            if self.dim_input:
                self.update_equation_function(t0, state_or_input)
            else:
                # 如果 dim_input 为 0，则调用 update_equation_function 方法，只传入 t0 参数
                self.update_equation_function(t0,)
        # 无论如何，如果 dim_state 或 dim_input 不为 0，则返回 output_equation_function 方法的结果，传入 t0 和 state_or_input 参数
        if self.dim_state or self.dim_input:
            return self.output_equation_function(t0, state_or_input)
        else:
            # 如果 dim_state 和 dim_input 均为 0，则返回 output_equation_function 方法的结果，只传入 t0 参数
            return self.output_equation_function(t0)
    # 定义一个对象方法用于验证对象的状态
    def validate(self):
        # 如果输出维度为0，则抛出值错误异常，消息为 zero_dim_output_msg
        if self.dim_output == 0:
            raise ValueError(zero_dim_output_msg)

        # 如果状态维度大于0，并且未设置状态方程函数，则抛出值错误异常，消息为 need_state_equation_function_msg
        if (self.dim_state > 0
                and getattr(self, 'state_equation_function', None) is None):
            raise ValueError(need_state_equation_function_msg)

        # 如果状态维度为0，并且未设置输出方程函数，则抛出值错误异常，消息为 need_output_equation_function_msg
        if (self.dim_state == 0
                and getattr(self, 'output_equation_function', None) is None):
            raise ValueError(need_output_equation_function_msg)
def SystemFromCallable(incallable, dim_input, dim_output, dt=0):
    """
    Construct a memoryless system from a callable.

    Parameters
    ----------
    incallable : callable
        Function to use as the output_equation_function. Should have signature
        (t, u) if dim_input > 0 or (t) if dim_input = 0.
    dim_input : int
        Dimension of input.
    dim_output : int
        Dimension of output.
    """
    # 使用给定的可调用函数构建一个无记忆的系统
    system = DynamicalSystem(output_equation_function=incallable,
                             dim_input=dim_input, dim_output=dim_output, dt=dt)
    return system


class SwitchedSystem(DynamicalSystem):
    """
    Provides a useful pattern for discontinuous systems where the state and
    output equations change depending on the value of a function of the state
    and/or input (``event_variable_equation_function``). Most of the usefulness
    comes from constructing the ``event_equation_function`` with a Bernstein
    basis polynomial with roots at the boundaries. This class also provides
    logic for outputting the correct state and output equation based on the
    ``event_variable_equation_function`` value.
    """
    def validate(self):
        # 调用父类的 validate 方法
        super().validate()

        if (self.dim_state > 0
                and np.any(np.equal(self.state_equations_functions, None))):
            raise ValueError(need_state_equation_function_msg)

        if (self.dim_state == 0
                and np.any(np.equal(self.output_equations_functions, None))):
            raise ValueError(need_output_equation_function_msg)

        if self.event_variable_equation_function is None:
            raise ValueError("A SwitchedSystem requires " +
                             "event_variable_equation_function")

    @property
    def event_bounds(self):
        return self._event_bounds

    @event_bounds.setter
    def event_bounds(self, event_bounds):
        if event_bounds is None:
            raise ValueError("A SwitchedSystem requires event_bounds")
        self._event_bounds = np.array(event_bounds).reshape(1, -1)
        self.n_conditions = self._event_bounds.size + 1
        if self.n_conditions == 2:
            self.event_bounds_range = 1
        else:
            self.event_bounds_range = np.diff(self.event_bounds[0, [0, -1]])

    def output_equation_function(self, *args):
        return self.output_equations_functions[self.condition_idx](*args)

    def state_equation_function(self, *args):
        return self.state_equations_functions[self.condition_idx](*args)

    def event_equation_function(self, *args):
        event_var = self.event_variable_equation_function(*args)
        return np.prod(
            (self.event_bounds_range-self.event_bounds)*event_var -
            self.event_bounds*(self.event_bounds_range - event_var),
            axis=1
        )
    # 更新方程式函数，用于根据事件变量更新条件索引
    def update_equation_function(self, *args):
        # 计算事件变量的值
        event_var = self.event_variable_equation_function(*args)
        
        # 如果条件索引为None，则确定其值
        if self.condition_idx is None: 
            # 构建条件矩阵，找到满足条件的索引
            self.condition_idx = np.where(np.all(np.r_[
                    np.c_[[[True]], event_var >= self.event_bounds],
                    np.c_[event_var <= self.event_bounds, [[True]]]
                    ], axis=0))[0][0]
            return
        
        # 计算事件变量与事件边界之间的平方距离
        sq_dist = (event_var - self.event_bounds)**2
        
        # 找到距离最小的索引，判断是否需要更新条件索引
        crossed_root_idx = np.where(sq_dist == np.min(sq_dist))[1][0]
        if crossed_root_idx == self.condition_idx:
            self.condition_idx += 1
        elif crossed_root_idx == self.condition_idx - 1:
            self.condition_idx -= 1
        else:
            # 如果未跨越相邻边界，则发出警告
            warnings.warn("SwitchedSystem did not cross a neighboring " +
                          "boundary. This may indicate an integration " +
                          "error. Continuing without updating " +
                          "condition_idx", UserWarning)
        
        # 调用状态更新方程式函数，并返回结果
        return self.state_update_equation_function(*args)

    # 准备进行积分操作
    def prepare_to_integrate(self):
        # 如果状态维度存在
        if self.dim_state:
            # 计算事件变量的值，并确定条件索引
            event_var = self.event_variable_equation_function(0, 
                self.initial_condition)
            self.condition_idx = np.where(np.all(np.r_[
                    np.c_[[[True]], event_var >= self.event_bounds],
                    np.c_[event_var <= self.event_bounds, [[True]]]
                    ], axis=0))[0][0]
        else:
            # 否则，将条件索引设为None
            self.condition_idx = None
# 定义一个类 LTISystem，继承自 DynamicalSystem 类
class LTISystem(DynamicalSystem):
    """
    A linear, time-invariant system.
    """
    def __init__(self, *args, initial_condition=None, dt=0):
        """
        Construct an LTI system with the following input formats:

        1. state matrix A, input matrix B, output matrix C for systems with
           state::

              dx_dt = Ax + Bu
              y = Hx

        2. state matrix A, input matrix B for systems with state, assume full
           state output::

              dx_dt = Ax + Bu
              y = Ix

        3. gain matrix K for systems without state::

              y = Kx


        The matrices should be numeric arrays of consistent shape. The class
        provides ``A``, ``B``, ``C`` and ``F``, ``G``, ``H`` aliases for the
        matrices of systems with state, as well as a ``K`` alias for the gain
        matrix. The ``data`` alias provides the matrices as a tuple.
        """

        # Check the number of arguments passed to the constructor
        if len(args) not in (1, 2, 3):
            raise ValueError("LTI system expects 1, 2, or 3 args")

        # Initialize instance variables related to event handling
        self.num_events = 0
        self.event_equation_function = None
        self.update_equation_function = None

        # TODO: setup jacobian functions
        # Case 1: Only a gain matrix is provided
        if len(args) == 1:
            self.gain_matrix = gain_matrix = np.array(args[0])
            self.dim_input = (self.gain_matrix.shape[1]
                              if len(gain_matrix.shape) > 1
                              else 1)
            self.dim_output = self.gain_matrix.shape[0]
            self.dim_state = 0  # No state dimension in this case
            self.initial_condition = None  # No initial condition provided
            self.state_equation_function = None  # No state equation function defined
            # Output equation function directly uses the gain matrix
            self.output_equation_function = \
                lambda t, x: (gain_matrix@x).reshape(-1)
            self.dt = dt
            return

        # Case 2: State matrix A and input matrix B are provided
        if len(args) == 2:
            state_matrix, input_matrix = args
            # Assume full state output, so create an identity matrix for output
            output_matrix = np.eye(
                getattr(state_matrix, 'shape', len(state_matrix))[0]
            )

        # Case 3: State matrix A, input matrix B, and output matrix C are provided
        elif len(args) == 3:
            state_matrix, input_matrix, output_matrix = args

        # Reshape input matrix if it's 1D to ensure it's a column vector
        if len(input_matrix.shape) == 1:
            input_matrix = input_matrix.reshape(-1, 1)

        # Convert matrices to numpy arrays
        state_matrix = np.array(state_matrix)
        input_matrix = np.array(input_matrix)
        output_matrix = np.array(output_matrix)

        # Assign dimensions based on matrix shapes
        self.dim_state = state_matrix.shape[0]
        self.dim_input = input_matrix.shape[1]
        self.dim_output = output_matrix.shape[0]

        # Assign matrices to instance variables
        self.state_matrix = state_matrix
        self.input_matrix = input_matrix
        self.output_matrix = output_matrix

        # Assign initial condition and define state equation function
        self.initial_condition = initial_condition
        if self.dim_input:
            # State equation function considering input u as zeros if not provided
            self.state_equation_function = \
                (lambda t, x, u=np.zeros(self.dim_input): \
                    (state_matrix@x + input_matrix@u))
        else:
            # State equation function with no input
            self.state_equation_function = lambda t, x, u=np.zeros(0): state_matrix@x

        # Output equation function directly uses the output matrix
        self.output_equation_function = \
            lambda t, x: (output_matrix@x)

        # Assign time step
        self.dt = dt

        # Validate the consistency of the LTI system matrices
        self.validate()
    # 调用父类的 validate 方法，确保基本的验证通过
    def validate(self):
        super().validate()
        # 如果存在状态维度信息
        if self.dim_state:
            # 断言状态矩阵的列数等于状态维度
            assert self.state_matrix.shape[1] == self.dim_state
            # 断言输入矩阵的行数等于状态维度
            assert self.input_matrix.shape[0] == self.dim_state
            # 断言输出矩阵的列数等于状态维度
            assert self.output_matrix.shape[1] == self.dim_state

    # 数据属性，根据是否存在状态维度返回不同的值
    @property
    def data(self):
        if self.dim_state:
            # 返回状态矩阵、输入矩阵和输出矩阵
            return self.state_matrix, self.input_matrix, self.output_matrix
        else:
            # 返回增益矩阵
            return self.gain_matrix

    # 属性 A 返回状态矩阵
    @property
    def A(self):
        return self.state_matrix

    # 属性 F 返回状态矩阵
    @property
    def F(self):
        return self.state_matrix

    # 属性 B 返回输入矩阵
    @property
    def B(self):
        return self.input_matrix

    # 属性 G 返回输入矩阵
    @property
    def G(self):
        return self.input_matrix

    # 属性 C 返回输出矩阵
    @property
    def C(self):
        return self.output_matrix

    # 属性 H 返回输出矩阵
    @property
    def H(self):
        return self.output_matrix

    # 属性 K 返回增益矩阵
    @property
    def K(self):
        return self.gain_matrix
```