# `.\simupy\simupy\utils\symbolic.py`

```
import numpy as np  # 导入NumPy库，用于数值计算
import sympy as sp  # 导入SymPy库，用于符号计算
from sympy.utilities.lambdify import implemented_function  # 导入SymPy的函数，用于实现特定函数
from sympy.physics.mechanics import dynamicsymbols  # 导入SymPy的dynamicsymbols模块，用于处理动态符号
from simupy.array import r_, Array  # 从simupy.array模块导入r_和Array类

DEFAULT_LAMBDIFY_MODULES = ({'ImmutableMatrix': np.matrix, "atan2": np.arctan2}, "numpy", {"Mod": np.mod, "atan2": np.arctan2})

def process_vector_args(args):
    """
    A helper function to process vector arguments so callables can take
    vectors or individual components. Essentially unravels the arguments.
    """
    new_args = []
    for arg in args:
        if hasattr(arg, 'shape') and len(arg.shape) > 0:
            shape = arg.shape
            if (min(shape) != 1 and len(shape) == 2) or len(shape) > 2:
                raise AttributeError("Arguments should only contain vectors")
            for i in range(max(shape)):
                if len(shape) == 1:
                    new_args.append(arg[i])
                elif shape[0] == 1:
                    new_args.append(arg[0, i])
                elif shape[1] == 1:
                    new_args.append(arg[i, 0])
        elif isinstance(arg, (list, tuple)):
            for element in arg:
                if isinstance(element, (list, tuple)):
                    raise AttributeError("Arguments should not be nested " +
                                         "lists/tuples")
                new_args.append(element)
        else:  # hope it's atomic!
            new_args.append(arg)

    return tuple(new_args)

def lambdify_with_vector_args(args, expr, modules=DEFAULT_LAMBDIFY_MODULES):
    """
    A wrapper around sympy's lambdify where process_vector_args is used so
    generated callable can take arguments as either vector or individual
    components

    Parameters
    ----------
    args : list-like of sympy symbols
        Input arguments to the expression to call
    expr : sympy expression
        Expression to turn into a callable for numeric evaluation
    modules : list
        See lambdify documentation; passed directly as modules keyword.

    """
    new_args = process_vector_args(args)

    if sp.__version__ < '1.1' and hasattr(expr, '__len__'):
        expr = sp.Matrix(expr)

    f = sp.lambdify(new_args, expr, modules=modules)

    def lambda_function_with_vector_args(*func_args):
        new_func_args = process_vector_args(func_args)
        return np.array(f(*new_func_args))
    lambda_function_with_vector_args.__doc__ = f.__doc__
    return lambda_function_with_vector_args

def grad(f, basis, for_numerical=True):
    """
    Compute the symbolic gradient of a vector-valued function with respect to a
    basis.

    Parameters
    ----------
    f : 1D array_like of sympy Expressions
        The vector-valued function to compute the gradient of.
    basis : 1D array_like of sympy symbols
        The basis symbols to compute the gradient with respect to.
    for_numerical : bool, optional
        A placeholder for the option of numerically computing the gradient.

    Returns
    -------
    """
    # 如果 f 是一个可迭代对象（例如列表或元组），将其转换为 sympy 的矩阵对象
    if hasattr(f, '__len__'):  # as of version 1.1.1, Array isn't supported
        f = sp.Matrix(f)

    # 构造一个新的与 f 类型相同的对象，其中每个元素是一个列表推导式
    return f.__class__([
        [
            # 如果 for_numerical 为 False 或者 f[x] 不包含基向量 basis[y] 的符号，
            # 则计算 f[x] 对 basis[y] 的偏导数；否则，返回 0
            sp.diff(f[x], basis[y])
            if not for_numerical or not f[x].has(sp.sign(basis[y])) else 0
            for y in range(len(basis))  # 对基向量列表 basis 中的每个向量进行迭代
        ] for x in range(len(f))  # 对 f 中的每个元素进行迭代
    ])
# 输入参数system是一个DynamicalSystem对象，用于增广输入。
# input_是一个可选参数，用于指定要增广的输入符号数组。
# update_outputs是一个布尔值，如果为True并且系统提供完整的状态输出，则还会将增广的输入添加到输出中。

# 复制输入的系统对象，以便进行修改而不影响原始系统。
augmented_system = system.copy()

# 如果input_为空列表，则默认为增广所有的输入符号。
if input_ == []:
    input_ = system.input

# 将增广后的输入符号添加到系统的状态中。
augmented_system.state = r_[system.state, input_]

# 为每个输入符号生成其导数符号，并将其作为新的输入。
augmented_system.input = Array([
    dynamicsymbols(str(input_var.func) + 'prime')
    for input_var in input_
])

# 更新系统的状态方程，将增广后的输入符号也加入到状态方程中。
augmented_system.state_equation = r_[
    system.state_equation, augmented_system.input]

# 如果update_outputs为True，并且系统的输出方程等于状态方程，则将输出方程更新为增广后的状态。
if update_outputs and system.output_equation == system.state:
    augmented_system.output_equation = augmented_system.state

# 返回增广后的系统对象。
return augmented_system
```