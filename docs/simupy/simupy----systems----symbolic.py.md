# `.\simupy\simupy\systems\symbolic.py`

```
# 导入 sympy 库，并使用别名 sp
import sympy as sp
# 从 sympy.physics.mechanics 模块中导入 dynamicsymbols 函数
from sympy.physics.mechanics import dynamicsymbols
# 从 sympy.physics.mechanics.functions 模块中导入 find_dynamicsymbols 函数
from sympy.physics.mechanics.functions import find_dynamicsymbols
# 从 simupy.utils.symbolic 模块中导入 lambdify_with_vector_args, grad, DEFAULT_LAMBDIFY_MODULES 函数和变量
from simupy.utils.symbolic import (lambdify_with_vector_args, grad, 
    DEFAULT_LAMBDIFY_MODULES)
# 从 simupy.array 模块中导入 Array 类和 empty_array 函数
from simupy.array import Array, empty_array

# 从 simupy.systems 模块中导入 DynamicalSystem 类，并使用 DynamicalSystemBase 作为基类
from simupy.systems import DynamicalSystem as DynamicalSystemBase

# 设置默认的代码生成器为 lambdify_with_vector_args 函数
DEFAULT_CODE_GENERATOR = lambdify_with_vector_args
# 设置默认的代码生成器参数为一个包含 DEFAULT_LAMBDIFY_MODULES 的字典
DEFAULT_CODE_GENERATOR_ARGS = {
    'modules': DEFAULT_LAMBDIFY_MODULES
}

# 定义 DynamicalSystem 类，继承自 DynamicalSystemBase
class DynamicalSystem(DynamicalSystemBase):
    # 定义 DynamicalSystem 类的构造函数，用于基于符号表达式创建系统

    def __init__(self, state_equation=None, state=None, input_=None,
                 output_equation=None, constants_values={}, dt=0,
                 initial_condition=None, code_generator=None,
                 code_generator_args={}):
        """
        DynamicalSystem constructor, used to create systems from symbolic
        expressions.

        Parameters
        ----------
        state_equation : array_like of sympy Expressions, optional
            状态变量导数的向量值表达式。
        state : array_like of sympy symbols, optional
            表示状态变量组件的符号向量，按照所需的顺序，与 state_equation 匹配。
        input_ : array_like of sympy symbols, optional
            表示输入变量组件的符号向量，按照所需的顺序。state_equation 可能依赖于系统输入。
            如果系统没有状态，output_equation 可能依赖于系统输入。
        output_equation : array_like of sympy Expressions
            系统输出的向量值表达式。
        constants_values : dict
            常量替换的字典。
        dt : float
            系统的采样率。对于连续时间系统，请使用 0。
        initial_condition : array_like of numerical values, optional
            作为系统初始条件的数组或矩阵。默认为与状态相同维度的零向量。
        code_generator : callable, optional
            用作代码生成器的函数。
        code_generator_args : dict, optional
            传递给代码生成器的关键字参数字典。

        By default, the code generator uses a wrapper for ``sympy.lambdify``.
        You can change it by passing the system initialization arguments
        ``code_generator`` (the function) and additional keyword arguments to
        the generator in a dictionary ``code_generator_args``. You can change
        the defaults for future systems by changing the module values. See the
        readme or docs for an example.
        
        默认情况下，代码生成器使用 ``sympy.lambdify`` 的包装器。
        通过传递系统初始化参数 ``code_generator``（函数）和额外的关键字参数字典 ``code_generator_args``，
        您可以更改它。您可以通过更改模块值来更改将来系统的默认值。请参阅自述文件或文档获取示例。

        """
        # 设置常量值字典
        self.constants_values = constants_values
        # 设置状态向量
        self.state = state
        # 设置输入向量
        self.input = input_

        # 设置代码生成器，默认使用 DEFAULT_CODE_GENERATOR
        self.code_generator = code_generator or DEFAULT_CODE_GENERATOR

        # 设置代码生成器参数，默认使用 DEFAULT_CODE_GENERATOR_ARGS，然后更新为传入的参数
        code_gen_args_to_set = DEFAULT_CODE_GENERATOR_ARGS.copy()
        code_gen_args_to_set.update(code_generator_args)
        self.code_generator_args = code_gen_args_to_set

        # 设置状态方程
        self.state_equation = state_equation
        # 设置输出方程
        self.output_equation = output_equation

        # 设置初始条件
        self.initial_condition = initial_condition
        # 设置采样率
        self.dt = dt

        # 执行验证方法
        self.validate()

    @property
    # 定义状态属性的 getter 方法
    def state(self):
        return self._state

    @state.setter
    # 定义一个方法 `state`，用于设置对象的状态属性
    def state(self, state):
        # 如果状态为 None，则调用 empty_array() 函数创建一个空数组
        if state is None:  # or other checks?
            state = empty_array()
        # 如果状态是 SymPy 表达式，则封装成包含单个表达式的数组
        if isinstance(state, sp.Expr):
            state = Array([state])
        # 设置对象的状态维度为状态数组的长度
        self.dim_state = len(state)
        # 将状态数组赋值给对象的状态属性 `_state`
        self._state = state

    # 定义一个属性 `input`，用于获取对象的输入属性
    @property
    def input(self):
        # 返回对象的输入属性 `_inputs`
        return self._inputs

    # `input` 属性的 setter 方法，用于设置对象的输入属性
    @input.setter
    def input(self, input_):
        # 如果输入为 None，则调用 empty_array() 函数创建一个空数组
        if input_ is None:  # or other checks?
            input_ = empty_array()
        # 如果输入是 SymPy 表达式，则封装成包含单个表达式的数组
        if isinstance(input_, sp.Expr):  # check it's a single dynamicsymbol?
            input_ = Array([input_])
        # 设置对象的输入维度为输入数组的长度
        self.dim_input = len(input_)
        # 将输入数组赋值给对象的输入属性 `_inputs`
        self._inputs = input_

    # 定义一个属性 `state_equation`，用于获取对象的状态方程属性
    @property
    def state_equation(self):
        # 返回对象的状态方程属性 `_state_equation`
        return self._state_equation

    # `state_equation` 属性的 setter 方法，用于设置对象的状态方程属性
    @state_equation.setter
    def state_equation(self, state_equation):
        # 如果状态方程为 None，则调用 empty_array() 函数创建一个空数组
        if state_equation is None:  # or other checks?
            state_equation = empty_array()
        else:
            # 断言状态方程的长度与对象状态数组的长度相同
            assert len(state_equation) == len(self.state)
            # 断言状态方程中的动态符号在对象的状态和输入集合中
            assert find_dynamicsymbols(state_equation) <= (
                    set(self.state) | set(self.input)
                )
            # 断言状态方程中的符号在对象的常量值集合和时间符号集合中
            assert state_equation.atoms(sp.Symbol) <= (
                    set(self.constants_values.keys())
                    | set([dynamicsymbols._t])
                )

        # 将状态方程赋值给对象的状态方程属性 `_state_equation`
        self._state_equation = state_equation
        # 更新状态方程的相关函数
        self.update_state_equation_function()

        # 计算状态方程对状态的雅可比矩阵，并更新相应的函数
        self.state_jacobian_equation = grad(self.state_equation, self.state)
        self.update_state_jacobian_function()

        # 计算状态方程对输入的雅可比矩阵，并更新相应的函数
        self.input_jacobian_equation = grad(self.state_equation, self.input)
        self.update_input_jacobian_function()

    # 定义一个属性 `output_equation`，用于获取对象的输出方程属性
    @property
    def output_equation(self):
        # 返回对象的输出方程属性 `_output_equation`
        return self._output_equation

    # `output_equation` 属性的 setter 方法，用于设置对象的输出方程属性
    @output_equation.setter
    def output_equation(self, output_equation):
        # 如果输出方程是 SymPy 表达式，则封装成包含单个表达式的数组
        if isinstance(output_equation, sp.Expr):
            output_equation = Array([output_equation])
        # 如果输出方程为 None 且对象的状态维度为 0，则调用 empty_array() 函数创建一个空数组
        if output_equation is None and self.dim_state == 0:
            output_equation = empty_array()
        else:
            # 如果输出方程为 None，则将对象的状态数组赋值给输出方程
            if output_equation is None:
                output_equation = self.state

            # 断言输出方程中的符号在对象的常量值集合和时间符号集合中
            assert output_equation.atoms(sp.Symbol) <= (
                    set(self.constants_values.keys())
                    | set([dynamicsymbols._t])
                   )

            # 如果对象有状态，则断言输出方程中的动态符号在对象的状态集合中
            if self.dim_state:
                assert find_dynamicsymbols(output_equation) <= set(self.state)
            else:
                # 否则断言输出方程中的动态符号在对象的输入集合中
                assert find_dynamicsymbols(output_equation) <= set(self.input)

        # 设置对象的输出维度为输出方程数组的长度
        self.dim_output = len(output_equation)

        # 将输出方程赋值给对象的输出方程属性 `_output_equation`
        self._output_equation = output_equation
        # 更新输出方程的相关函数
        self.update_output_equation_function()

    # 定义一个方法 `update_state_equation_function`，用于更新状态方程的函数表示
    def update_state_equation_function(self):
        # 如果对象没有状态或状态方程为空数组，则返回
        if not self.dim_state or self.state_equation == empty_array():
            return
        # 使用代码生成器生成状态方程的函数表示
        self.state_equation_function = self.code_generator(
            [dynamicsymbols._t] + sp.flatten(self.state) +
            sp.flatten(self.input),
            self.state_equation.subs(self.constants_values),
            **self.code_generator_args
        )
    # 更新状态雅可比矩阵函数
    def update_state_jacobian_function(self):
        # 如果状态维度为空或状态方程为空数组，则返回
        if not self.dim_state or self.state_equation == empty_array():
            return
        # 使用代码生成器生成状态雅可比矩阵函数
        self.state_jacobian_equation_function = self.code_generator(
            [dynamicsymbols._t] + sp.flatten(self.state) +
            sp.flatten(self.input),
            self.state_jacobian_equation.subs(self.constants_values),
            **self.code_generator_args
        )

    # 更新输入雅可比矩阵函数
    def update_input_jacobian_function(self):
        # TODO: 无状态系统应该有输入/输出雅可比矩阵
        # 如果状态维度为空或状态方程为空数组，则返回
        if not self.dim_state or self.state_equation == empty_array():
            return
        # 使用代码生成器生成输入雅可比矩阵函数
        self.input_jacobian_equation_function = self.code_generator(
            [dynamicsymbols._t] + sp.flatten(self.state) +
            sp.flatten(self.input),
            self.input_jacobian_equation.subs(self.constants_values),
            **self.code_generator_args
        )

    # 更新输出方程函数
    def update_output_equation_function(self):
        # 如果输出维度为空或输出方程为空数组，则返回
        if not self.dim_output or self.output_equation == empty_array():
            return
        # 如果状态维度存在，则生成输出方程函数
        if self.dim_state:
            self.output_equation_function = self.code_generator(
                [dynamicsymbols._t] + sp.flatten(self.state),
                self.output_equation.subs(self.constants_values),
                **self.code_generator_args
            )
        else:  # 如果状态维度不存在，则根据输入生成输出方程函数
            self.output_equation_function = self.code_generator(
                [dynamicsymbols._t] + sp.flatten(self.input),
                self.output_equation.subs(self.constants_values),
                **self.code_generator_args
            )

    # 准备进行积分
    def prepare_to_integrate(self, t0, state_or_input=None):
        # 更新输出方程函数
        self.update_output_equation_function()
        # 更新状态方程函数
        self.update_state_equation_function()
        # 如果状态维度为空且事件数不为零，则更新方程函数
        if not self.dim_state and self.num_events:
            self.update_equation_function(t0, state_or_input)
        # 如果状态维度或输入维度存在，则返回输出方程函数值
        if self.dim_state or self.dim_input:
            return self.output_equation_function(t0, state_or_input)
        else:  # 否则，只返回输出方程函数值
            return self.output_equation_function(t0)

    # 创建当前对象的副本
    def copy(self):
        # 创建当前对象的副本
        copy = self.__class__(
            state_equation=self.state_equation,
            state=self.state,
            input_=self.input,
            output_equation=self.output_equation,
            constants_values=self.constants_values,
            dt=self.dt
        )
        # 将输出方程函数和状态方程函数复制给副本对象
        copy.output_equation_function = self.output_equation_function
        copy.state_equation_function = self.state_equation_function
        return copy

    # 求解系统的平衡点
    def equilibrium_points(self, input_=None):
        # 求解状态方程关于状态变量的根，返回字典形式的平衡点
        return sp.solve(self.state_equation, self.state, dict=True)
class MemorylessSystem(DynamicalSystem):
    """
    A system with no state.

    With no input, can represent a signal (function of time only). For example,
    a stochastic signal could interpolate points and use prepare_to_integrate
    to re-seed the data.
    """

    def __init__(self, input_=None, output_equation=None, **kwargs):
        """
        DynamicalSystem constructor

        Parameters
        ----------
        input_ : array_like of sympy symbols
            Vector of symbols representing the components of the input, in the
            desired order. The output may depend on the system input.
        output_equation : array_like of sympy Expressions
            Vector valued expression for the output of the system.
        """
        # 调用父类构造函数初始化系统，传入输入符号和输出方程
        super().__init__(
              input_=input_, output_equation=output_equation, **kwargs)

    @property
    def state(self):
        # 返回当前状态属性
        return self._state

    @state.setter
    def state(self, state):
        # 如果状态为None，则使用空数组作为状态
        if state is None:  # or other checks?
            state = empty_array()
        else:
            # 如果状态不为空，则抛出异常，因为MemorylessSystem不应有状态
            raise ValueError("Memoryless system should not have state or " +
                             "state_equation")
        # 设置状态维度为状态数组的长度
        self.dim_state = len(state)
        # 将状态数组赋给私有属性_state
        self._state = state
```