# `.\simupy\simupy\discontinuities.py`

```
import sympy as sp
import numpy as np
from simupy.systems.symbolic import (DynamicalSystem, MemorylessSystem,
                                     dynamicsymbols, find_dynamicsymbols)
from simupy.systems import SwitchedSystem as SwitchedSystemBase
from simupy.array import r_

class DiscontinuousSystem(DynamicalSystem):
    """
    A continuous-time dynamical system with a discontinuity. Must provide the
    following attributes in addition to those of DynamicalSystem:

    ``event_equation_function`` - A function called at each integration time-
    step and stored in simulation results. Takes input and state, if stateful.
    A zero-crossing of this output triggers the discontinuity.

    ``update_equation_function`` - A function that is called when the
    discontinuity occurs. This is generally used to change what
    ``state_equation_function``, ``output_equation_function``, and
    ``event_equation_function`` compute based on the occurrence of the
    discontinuity. If stateful, returns the state immediately after the
    discontinuity.
    """

    # 定义事件方程函数，抛出未实现错误，需子类实现
    def event_equation_function(self, *args, **kwargs):
        raise NotImplementedError

    # 定义更新方程函数，抛出未实现错误，需子类实现
    def update_equation_function(self, *args, **kwargs):
        raise NotImplementedError

    # 定义属性 dt 的 getter 方法
    @property
    def dt(self):
        return 0

    # 定义属性 dt 的 setter 方法，用于验证 dt 是否为零
    @dt.setter
    def dt(self, dt):
        if dt != 0:
            raise ValueError("Discontinuous systems only make sense for " +
                             "continuous time systems")

# SwitchedSystem 类继承自 SwitchedSystemBase 和 DiscontinuousSystem
class SwitchedSystem(SwitchedSystemBase, DiscontinuousSystem):
    pass
    def __init__(self, event_variable_equation, event_bounds_expressions,
                 state_equations=None, output_equations=None,
                 state_update_equation=None, **kwargs):
        """
        SwitchedSystem constructor, used to create switched systems from
        symbolic expressions. The parameters below are in addition to
        parameters from the ``systems.symbolic.DynamicalSystems`` constructor.

        Parameters
        ----------
        event_variable_equation : sympy Expression
            表示事件变量方程的符号表达式
        event_bounds_expressions : list-like of sympy Expressions or floats
            按顺序排列的列表，定义事件边界（相对于event_variable_equation）
        state_equations : array_like of sympy Expressions, optional
            系统的状态方程。第一个维度索引事件状态，并应比事件边界的数量多一。
            应根据边界进行索引（即，当event_variable_equation低于第一个event_bounds值时使用第一个表达式）。
            第二维度是系统的状态维度。如果是1维的，则每个条件使用单个方程。
        output_equations : array_like of sympy Expressions, optional
            系统的输出方程。第一个维度索引事件状态，并应比事件边界的数量多一。
            应根据边界进行索引（即，当event_variable_equation低于第一个event_bounds值时使用第一个表达式）。
            第二维度是系统的输出维度。如果是1维的，则每个条件使用单个方程。
        state_update_equation : sympy Expression
            表示状态更新方程的符号表达式
        """
        # 调用父类DiscontinuousSystem的构造函数，并传入所有额外的关键字参数
        DiscontinuousSystem.__init__(self, **kwargs)
        # 设置对象的属性值
        self.event_variable_equation = event_variable_equation
        self.event_bounds_expressions = event_bounds_expressions
        self.output_equations = output_equations
        self.state_equations = state_equations
        self.state_update_equation = state_update_equation
        self.condition_idx = None  # 初始化条件索引为None
        self.validate(True)  # 调用validate方法验证对象的合法性

    def prepare_to_integrate(self):
        # TODO: refactor the setters so I can call an update instead
        # 将当前对象的属性重新设置为它们自身，以便调用更新而不是设置方法
        self.event_variable_equation = self.event_variable_equation
        self.event_bounds_expressions = self.event_bounds_expressions
        self.output_equations = self.output_equations
        self.state_equations = self.state_equations
        self.state_update_equation = self.state_update_equation

        # 调用父类的prepare_to_integrate方法
        super().prepare_to_integrate()

    def validate(self, from_self=False):
        # 如果from_self为True，则调用父类的validate方法验证对象自身
        if from_self:
            super().validate()
    def state_equations(self):
        # 返回当前对象的状态方程
        return self._state_equations

    @state_equations.setter
    def state_equations(self, state_equations):
        # 如果状态方程为 None，则将 _state_equations 设为 None 并返回
        if state_equations is None:  # or other checks?
            self._state_equations = None
            return
        # 如果对象有 event_bounds 属性
        if hasattr(self, 'event_bounds'):
            # 如果 state_equations 是一维数组或第一维度长度为 1
            if (len(state_equations.shape) == 1 or
                    state_equations.shape[0] == 1):
                # 使用 r_.__getitem__ 函数重新组织 state_equations
                state_equations = r_.__getitem__(
                    ('0,2', *tuple(self.n_conditions*(state_equations,)))
                )
            # 检查条件数量是否与 event_bounds 的列数匹配
            n_conditions_test = self.event_bounds.shape[1]+1
            assert state_equations.shape[0] == n_conditions_test
        # 将 state_equations 赋给 _state_equations
        self._state_equations = state_equations
        # 记录条件数量
        self.n_conditions = state_equations.shape[0]
        # 初始化状态方程函数列表
        self.state_equations_functions = np.empty(self.n_conditions, object)

        # 为每个条件索引生成状态方程函数
        for cond_idx in range(self.n_conditions):
            self.state_equation = state_equations[cond_idx, :]
            self.state_equations_functions[cond_idx] = \
                self.state_equation_function
        # 定义状态方程函数
        self.state_equation_function = (
            lambda *args:
            SwitchedSystemBase.state_equation_function(self, *args)
        )

    @property
    def output_equations(self):
        # 返回当前对象的输出方程
        return self._output_equations

    @output_equations.setter
    def output_equations(self, output_equations):
        # 如果输出方程为 None，则根据条件初始化输出方程
        if output_equations is None:  # or other checks?
            if self.dim_state > 0 and hasattr(self, 'n_conditions'):
                output_equations = r_.__getitem__(
                    ('0,2', *tuple(self.n_conditions*(self.state,)))
                )
            else:
                self._output_equations = None
                return
        # 如果对象有 event_bounds 属性
        if hasattr(self, 'event_bounds'):
            # 如果 output_equations 是一维数组或第一维度长度为 1
            if (len(output_equations.shape) == 1 or
                    output_equations.shape[0] == 1):
                # 使用 r_.__getitem__ 函数重新组织 output_equations
                output_equations = r_.__getitem__(
                    ('0,2', *tuple(self.n_conditions*(output_equations,)))
                )
            # 检查条件数量是否与 event_bounds 的列数匹配
            n_conditions_test = self.event_bounds.shape[1]+1
            assert output_equations.shape[0] == n_conditions_test
        # 将 output_equations 赋给 _output_equations
        self._output_equations = output_equations
        # 记录条件数量
        self.n_conditions = output_equations.shape[0]
        # 初始化输出方程函数列表
        self.output_equations_functions = np.empty(self.n_conditions, object)

        # 为每个条件索引生成输出方程函数
        for cond_idx in range(self.n_conditions):
            self.output_equation = output_equations[cond_idx, :]
            self.output_equations_functions[cond_idx] = \
                self.output_equation_function
        # 定义输出方程函数
        self.output_equation_function = (
            lambda *args:
            SwitchedSystemBase.output_equation_function(self, *args)
        )

    @property
    def state_update_equation(self):
        # 返回当前对象的状态更新方程
        return self._state_update_equation

    @state_update_equation.setter
    # 定义状态更新方程的方法，接受一个状态更新方程作为参数
    def state_update_equation(self, state_update_equation):
        # 如果传入的状态更新方程为 None，则根据当前对象的状态和输入情况进行选择
        if state_update_equation is None:
            # 如果状态维度大于 0，则选择当前状态作为更新方程
            if self.dim_state > 0:
                state_update_equation = self.state
            # 否则选择输入作为更新方程
            else:
                state_update_equation = self.input
        
        # 确保状态更新方程中的符号仅包含在常量值和时间符号的集合中
        assert state_update_equation.atoms(sp.Symbol) <= set(
            self.constants_values.keys()) | set([dynamicsymbols._t])
        
        # 将传入的状态更新方程赋值给对象的内部变量
        self._state_update_equation = state_update_equation
        
        # 如果存在状态维度
        if self.dim_state:
            # 确保状态更新方程中的动态符号仅包含在当前状态和输入变量的集合中
            assert find_dynamicsymbols(state_update_equation) <= \
                set(self.state) | set(self.input)
            
            # 使用代码生成器生成状态更新方程的函数
            self.state_update_equation_function = self.code_generator(
                ([dynamicsymbols._t] + 
                 sp.flatten(self.state) + sp.flatten(self.input)),
                self._state_update_equation.subs(self.constants_values),
                **self.code_generator_args
            )
        else:
            # 确保状态更新方程中的动态符号仅包含在输入变量的集合中
            assert find_dynamicsymbols(state_update_equation) <= \
                set(self.input)
            
            # 使用代码生成器生成状态更新方程的函数
            self.state_update_equation_function = self.code_generator(
                [dynamicsymbols._t] + sp.flatten(self.input),
                self._state_update_equation.subs(self.constants_values),
                **self.code_generator_args
            )

    # 定义事件变量方程的属性方法
    @property
    def event_variable_equation(self):
        return self._event_variable_equation

    # 事件变量方程的 setter 方法，接受一个事件变量方程作为参数
    @event_variable_equation.setter
    def event_variable_equation(self, event_variable_equation):
        # 确保事件变量方程中的符号仅包含在常量值和时间符号的集合中
        assert event_variable_equation.atoms(sp.Symbol) <= set(
            self.constants_values.keys()) | set([dynamicsymbols._t])
        
        # 将传入的事件变量方程赋值给对象的内部变量
        self._event_variable_equation = event_variable_equation
        
        # 如果存在状态维度
        if self.dim_state:
            # 确保事件变量方程中的动态符号仅包含在当前状态的集合中
            assert find_dynamicsymbols(event_variable_equation) <= \
                set(self.state)
            
            # 使用代码生成器生成事件变量方程的函数
            self.event_variable_equation_function = self.code_generator(
                [dynamicsymbols._t] + sp.flatten(self.state),
                self._event_variable_equation.subs(self.constants_values),
                **self.code_generator_args
            )
        else:
            # 确保事件变量方程中的动态符号仅包含在输入变量的集合中
            assert find_dynamicsymbols(event_variable_equation) <= \
                set(self.input)
            
            # 使用代码生成器生成事件变量方程的函数
            self.event_variable_equation_function = self.code_generator(
                [dynamicsymbols._t] + sp.flatten(self.input),
                self._event_variable_equation.subs(self.constants_values),
                **self.code_generator_args
            )

    # 定义事件边界表达式的属性方法
    @property
    def event_bounds_expressions(self):
        return self._event_bounds_expressions

    # 事件边界表达式的 setter 方法，接受一个事件边界表达式作为参数
    @event_bounds_expressions.setter
    # 定义一个方法用于设置事件边界表达式
    def event_bounds_expressions(self, event_bounds_exp):
        # 如果对象有属性 'output_equations'，则验证事件边界表达式的长度是否与输出方程的行数匹配
        if hasattr(self, 'output_equations'):
            assert len(event_bounds_exp) + 1 == self.output_equations.shape[0]
        
        # 如果对象有属性 'output_equations_functions'，则验证事件边界表达式的长度是否与输出方程函数的大小匹配
        if hasattr(self, 'output_equations_functions'):
            assert len(event_bounds_exp) + 1 == self.output_equations_functions.size
        
        # 如果存在 'state_equations' 属性，则验证事件边界表达式的长度是否与状态方程的行数匹配
        if getattr(self, 'state_equations', None) is not None:
            assert len(event_bounds_exp) + 1 == self.state_equations.shape[0]
        
        # 如果存在 'state_equations_functions' 属性，则验证事件边界表达式的长度是否与状态方程函数的大小匹配
        if getattr(self, 'state_equations_functions', None) is not None:
            assert len(event_bounds_exp) + 1 == self.state_equations_functions.size
        
        # 将输入的事件边界表达式保存到对象的 '_event_bounds_expressions' 属性中
        self._event_bounds_expressions = event_bounds_exp
        
        # 使用常量值替换常量，并将事件边界表达式转换为 numpy 数组，存储在 'event_bounds' 属性中
        self.event_bounds = np.array(
            [sp.N(bound, subs=self.constants_values)  # 对每个边界应用数值替换
             for bound in event_bounds_exp],  # 遍历每个事件边界表达式
            dtype=np.float_  # 设置数组类型为浮点数
        )
# 定义一个类 `MemorylessDiscontinuousSystem`，继承自 `DiscontinuousSystem` 和 `MemorylessSystem`
class MemorylessDiscontinuousSystem(DiscontinuousSystem, MemorylessSystem):
    pass
# 定义一个类 `SwitchedOutput`，继承自 `SwitchedSystem` 和 `MemorylessDiscontinuousSystem`
class SwitchedOutput(SwitchedSystem, MemorylessDiscontinuousSystem):
    """
    一个无记忆间断系统，方便构建开关输出。
    """
```