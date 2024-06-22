# `.\simupy\simupy\matrices.py`

```
# 导入NumPy库并用别名np表示
import numpy as np
# 导入SymPy库并用别名sp表示
import sympy as sp
# 从sympy.physics.mechanics中导入dynamicsymbols模块
from sympy.physics.mechanics import dynamicsymbols
# 从simupy.systems.symbolic中导入DynamicalSystem类
from simupy.systems.symbolic import DynamicalSystem


def construct_explicit_matrix(name, n, m, symmetric=False, diagonal=0,
                              dynamic=False, **kwass):
    """
    construct a matrix of symbolic elements

    Parameters
    ----------
    name : string
        Base name for variables; each variable is name_ij, which
        admitedly only works clearly for n,m < 10
    n : int
        Number of rows
    m : int
        Number of columns
    symmetric : bool, optional
        Use to enforce a symmetric matrix (repeat symbols above/below diagonal)
    diagonal : bool, optional
        Zeros out off diagonals. Takes precedence over symmetry.
    dynamic : bool, optional
        Whether to use sympy.physics.mechanics dynamicsymbol. If False, use
        sp.symbols
    kwargs : dict
        remaining kwargs passed to symbol function

    Returns
    -------
    matrix : sympy Matrix
        The Matrix containing explicit symbolic elements
    """
    # 根据dynamic参数选择符号函数
    if dynamic:
        symbol_func = dynamicsymbols
    else:
        symbol_func = sp.symbols

    # 如果n不等于m且同时要求对角或对称，则引发错误
    if n != m and (diagonal or symmetric):
        raise ValueError("Cannot make symmetric or diagonal if n != m")

    # 如果diagonal为True，则返回对角矩阵
    if diagonal:
        return sp.diag(
            *[symbol_func(
                name+'_{}{}'.format(i+1, i+1), **kwass) for i in range(m)])
    else:
        # 构造普通的n×m矩阵
        matrix = sp.Matrix([
            [symbol_func(name+'_{}{}'.format(j+1, i+1), **kwass)
             for i in range(m)] for j in range(n)
        ])

        # 如果symmetric为True，则将矩阵设置为对称矩阵
        if symmetric:
            for i in range(1, m):
                for j in range(i):
                    matrix[i, j] = matrix[j, i]
        return matrix


def matrix_subs(*subs):
    """
    Generate an object that can be passed into sp.subs from matrices, replacing
    each element in from_matrix with the corresponding element from to_matrix

    There are three ways to use this function, depending on the input:
    1. A single matrix-level subsitution - from_matrix, to_matrix
    2. A list or tuple of (from_matrix, to_matrix) 2-tuples
    3. A dictionary of {from_matrix: to_matrix} key-value pairs
    """
    # 如果输入为两个元素且不是列表、元组或字典，则转换为单个元素列表
    if len(subs) == 2 and not isinstance(subs[0], (list, tuple, dict)):
        subs = [subs]
    # 如果输入为列表或元组，则返回元组形式的替换对
    if isinstance(subs, (list, tuple)):
        return tuple(
            (sub[0][i, j], sub[1][i, j])
            for sub in subs
            for i in range(sub[0].shape[0])
            for j in range(sub[0].shape[1]) if sub[0][i, j] != 0
        )
    # 如果输入为字典，则返回字典形式的替换对
    elif isinstance(subs, dict):
        return {
            sub[0][i, j]: sub[1][i, j]
            for sub in subs.items()
            for i in range(sub[0].shape[0])
            for j in range(sub[0].shape[1]) if sub[0][i, j] != 0
        }


def block_matrix(blocks):
    """
    Construct a matrix where the elements are specified by the block structure
    """
    # 将给定的块矩阵拼接成一个大的块矩阵，按照指定的块结构
    def block_matrix(blocks):
        # 使用 sympy 的 Matrix.col_join 方法将所有行的矩阵拼接成一个大的列
        return sp.Matrix.col_join(
            # 生成器表达式，遍历每一行的矩阵块
            *tuple(
                # 使用 sympy 的 Matrix.row_join 方法将当前行内的所有矩阵块拼接成一行
                sp.Matrix.row_join(
                    # 生成器表达式，遍历当前行内的每个矩阵块
                    *tuple(mat for mat in row)) for row in blocks
            )
        )
def system_from_matrix_DE(mat_DE, mat_var, mat_input=None, constants={}):
    """
    Construct a symbolic DynamicalSystem using matrices. See
    riccati_system example.

    Parameters
    ----------
    mat_DE : sympy Matrix
        The matrix derivative expression (right hand side)

    mat_var : sympy Matrix
        The matrix state

    mat_input : list-like of input expressions, optional
        A list-like of input expressions in the matrix differential equation

    constants : dict, optional
        Dictionary of constants substitutions.

    Returns
    -------
    sys : DynamicalSystem
        A DynamicalSystem which can be used to numerically solve the matrix
        differential equation.
    """
    # Flatten and retrieve unique variables from mat_var
    vec_var = list(set(sp.flatten(mat_var.tolist())))

    # Initialize a zero matrix for the derivative expressions
    vec_DE = sp.Matrix.zeros(len(vec_var), 1)

    # Iterate over the matrix mat_DE to fill vec_DE with derivative expressions
    iterator = np.nditer(mat_DE, flags=['multi_index', 'refs_ok'])
    for it in iterator:
        i, j = iterator.multi_index
        # Find the index of mat_var[i, j] in vec_var and assign corresponding mat_DE value
        idx = vec_var.index(mat_var[i, j])
        vec_DE[idx] = mat_DE[i, j]

    # Create a DynamicalSystem object using vec_DE, vec_var, mat_input, and constants
    sys = DynamicalSystem(vec_DE, sp.Matrix(vec_var), mat_input,
                          constants_values=constants)

    # Return the constructed DynamicalSystem object
    return sys
```