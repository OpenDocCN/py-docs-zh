# `.\simupy\simupy\codegen.py`

```
# 导入 sympy 库，并从 sympy.physics.mechanics 中导入动力学符号
import sympy as sp
from sympy.physics.mechanics import dynamicsymbols
# 从 sympy.printing.pycode 中导入 NumPyPrinter 类
from sympy.printing.pycode import NumPyPrinter
# 从 sympy.utilities.lambdify 中导入 _EvaluatorPrinter 类
from sympy.utilities.lambdify import _EvaluatorPrinter
# 从 sympy.codegen.ast 中导入 AssignmentBase, Symbol, Element, Variable 类
from sympy.codegen.ast import AssignmentBase, Symbol, Element, Variable
# 从 sympy.core.function 中导入 AppliedUndef 类
from sympy.core.function import AppliedUndef
# 从 sympy.core.compatibility 中导入 iterable, PY3, string_types
import keyword
import re

# 将 _t 作为时间动力学符号 t
t = dynamicsymbols._t

def staticfy_expressions(expr):
    """
    Converts dynamic symbols (i.e., symbols which are implied functions of time)
    to static symbols in expr. Many codegen operations need dynamic symbols
    converted to static.
    """
    # 找到表达式中的动态符号
    dyn_syms = list(sp.physics.mechanics.functions.find_dynamicsymbols(expr))
    # 存储动态符号到静态符号的映射
    dynamic_to_static = {}
    for sym in dyn_syms:
        # 如果符号只有一个参数且为时间符号 t，则转换为静态符号
        if (len(sym.args) == 1) and sym.args[0] == t:
            dynamic_to_static[sym] = sp.symbols(sym.__class__.__name__)
    
    # 如果没有动态符号需要转换，则返回原始表达式
    if len(dynamic_to_static) == 0:
        return expr
    
    # 使用静态符号替换动态符号生成新的表达式
    new_expr = expr.subs(dynamic_to_static)
    
    # 如果原始表达式是函数且具有 shape 属性，则将 shape 属性赋予新表达式
    if (sp.Function in expr.__class__.__mro__) and hasattr(expr, 'shape'):
        new_expr.shape = expr.shape
    
    return new_expr

def process_funcs_to_print(funcs_to_print):
    """
    Prepares a set of sympy expressions to be printed with CSE. The `dict`\s
    are modified in-place.

    funcs_to_print is an iterable of dicts with keys:
        'extra_assignments', a dict of exta assignments before CSE
        'input_args', an Array of the input arguments
        'sym_expr', the symbolic expression to print with CSE
    """

    # 准备存储代码打印参数的空列表
    code_print_params = []
    # 遍历 funcs_to_print 列表中的每个字典元素
    for funcs_to_print_dict in funcs_to_print:
        # 创建一个空字典 extras_dict 来存储静态化后的额外赋值表达式
        extras_dict = {}  # funcs_to_print_dict['extra_assignments'].copy()

        # 遍历 funcs_to_print_dict 字典中额外赋值的项目
        for key, val in funcs_to_print_dict['extra_assignments'].items():
            # 将键和值静态化处理，并存入 extras_dict 中
            static_key = staticfy_expressions(key)
            static_val = staticfy_expressions(val)
            extras_dict[static_key] = static_val

        # 更新 funcs_to_print_dict 中的 extra_assignments 为静态化后的 extras_dict
        funcs_to_print_dict['extra_assignments'] = extras_dict

        # 如果 funcs_to_print_dict 中有输入参数，则将其静态化处理
        if funcs_to_print_dict['input_args']:
            funcs_to_print_dict['input_args'] = staticfy_expressions(funcs_to_print_dict['input_args'])

        # 将 funcs_to_print_dict 中的 sym_expr 静态化处理
        funcs_to_print_dict['sym_expr'] = staticfy_expressions(funcs_to_print_dict['sym_expr'])

        # 计算 funcs_to_print_dict['sym_expr'] 的总大小
        tot_size = 1
        if hasattr(funcs_to_print_dict['sym_expr'], 'shape'):
            raveled_shape = funcs_to_print_dict['sym_expr'].shape
            # 计算总大小
            for dim in raveled_shape:
                tot_size = tot_size * dim
            # 将 sym_expr 重塑为一维数组并转换为列表形式
            funcs_to_print_dict['sym_expr'] = funcs_to_print_dict['sym_expr'].reshape(tot_size).tolist()

        # 对 sym_expr 进行常数子表达式消除 (Common Subexpression Elimination, CSE)
        cse_out = sp.cse(funcs_to_print_dict['sym_expr'], order='none')
        cse_subs = {k: v for k, v in cse_out[0]}
        # 更新 extra_assignments 字典以包含 CSE 的结果
        funcs_to_print_dict['extra_assignments'].update(cse_subs)

        # 如果 tot_size 大于 1，则重新形状化 sym_expr 为原始形状
        if tot_size > 1:
            funcs_to_print_dict['sym_expr'] = sp.Array(cse_out[1]).reshape(*raveled_shape)
        else:
            funcs_to_print_dict['sym_expr'] = cse_out[1][0]

        # 将 funcs_to_print_dict 的 num_name、input_args、sym_expr 和 extra_assignments 添加到 code_print_params 列表中
        code_print_params.append((funcs_to_print_dict['num_name'], funcs_to_print_dict['input_args'], funcs_to_print_dict['sym_expr'], funcs_to_print_dict['extra_assignments']))

    # 返回 code_print_params 列表作为结果
    return code_print_params
class Assignment(AssignmentBase):
    """
    Represents variable assignment for code generation, but can also use Array
    on the LHS and non-defensively allows a function on RHS with indexed LHS.

    Parameters
    ==========

    lhs : Expr
        Sympy object representing the lhs of the expression. These should be
        singular objects, such as one would use in writing code. Notable types
        include Symbol, MatrixSymbol, MatrixElement, and Indexed. Types that
        subclass these types are also supported.

    rhs : Expr
        Sympy object representing the rhs of the expression. This can be any
        type, provided its shape corresponds to that of the lhs. For example,
        a Matrix type can be assigned to MatrixSymbol, but not to Symbol, as
        the dimensions will not align.
    """

    @classmethod
    def _check_args(cls, lhs, rhs):
        """ Check arguments to __new__ and raise exception if any problems found.

        Derived classes may wish to override this.
        """
        from sympy.matrices.expressions.matexpr import (
            MatrixElement, MatrixSymbol)
        from sympy.tensor.indexed import Indexed

        # Tuple of things that can be on the lhs of an assignment
        assignable = (Symbol, MatrixSymbol, MatrixElement, Indexed, Element, Variable, sp.Array)
        # 检查 lhs 是否是可赋值类型的实例
        if not isinstance(lhs, assignable):
            raise TypeError("Cannot assign to lhs of type %s." % type(lhs))

        # Indexed 类型实现了 shape 属性，但延迟定义，会导致赋值验证时的问题
        lhs_is_mat = hasattr(lhs, 'shape') and not isinstance(lhs, Indexed)
        rhs_is_mat = hasattr(rhs, 'shape') and not isinstance(rhs, Indexed)

        # 如果 lhs 和 rhs 结构相同，则赋值合法
        if lhs_is_mat and not isinstance(rhs, AppliedUndef):
            # 如果 rhs 不是矩阵，则抛出异常
            if not rhs_is_mat:
                raise ValueError("Cannot assign a scalar to a matrix.")
            # 如果 lhs 和 rhs 的形状不一致，则抛出异常
            elif lhs.shape != rhs.shape:
                raise ValueError("Dimensions of lhs and rhs don't align.")
        # 如果 rhs 是矩阵而 lhs 不是，则抛出异常
        elif rhs_is_mat and not lhs_is_mat:
            raise ValueError("Cannot assign a matrix to a scalar.")
    
    # 赋值操作符为 :=
    op = ':='


class ArrayNumPyPrinter(NumPyPrinter):
    def _print_Max(self, expr):
        # 格式化输出 numpy.maximum 函数调用
        return '{0}({1})'.format(self._module_format('numpy.maximum'), ','.join(self._print(i) for i in expr.args))
    # 定义一个方法用于打印 Piecewise 函数
    def _print_Piecewise(self, expr):
        """Piecewise function printer

        from sympy version:
        # If [default_value, True] is a (expr, cond) sequence in a Piecewise object
        #     it will behave the same as passing the 'default' kwarg to select()
        #     *as long as* it is the last element in expr.args.
        # If this is not the case, it may be triggered prematurely.

        which is just not implemented at all...
        """
        # 如果 Piecewise 函数的最后一个参数的条件为 True
        if expr.args[-1].cond == True:
            # 取出除了最后一个参数之外的所有参数作为表达式列表
            expr_args = expr.args[:-1]
            # 将最后一个参数的表达式作为默认值
            default_val = expr.args[-1].expr
        else:
            # 否则将所有参数作为表达式列表
            expr_args = expr.args
            # 默认值设为 NaN
            default_val = sp.NaN
        # 生成表达式字符串
        exprs = '[{0}]'.format(','.join(self._print(arg.expr) for arg in expr_args))
        # 生成条件字符串
        conds = '[{0}]'.format(','.join(self._print(arg.cond) for arg in expr_args))
        # 返回格式化后的字符串，表示使用 numpy.select 来处理 Piecewise 函数
        return '{0}({1}, {2}, default={3})'.format(
            self._module_format('numpy.select'), conds, exprs,
            self._print(default_val))
        
    # 定义一个方法用于打印 Heaviside 函数
    def _print_Heaviside(self, expr):
        # 返回 Heaviside 函数的字符串表示形式
        return '(({0}>0.0)+0.5*({0}==0.0))'.format(','.join(self._print(i) for i in expr.args))

    # 定义一个方法用于打印 ImmutableDenseNDimArray 对象
    def _print_ImmutableDenseNDimArray(self, expr):
        # 返回 ImmutableDenseNDimArray 对象的 numpy.array() 形式的字符串表示
        return "numpy.array(%s)" % (self._print(expr.tolist()),)
    # 定义一个方法 _print_Assignment，接受一个表达式参数 expr
    def _print_Assignment(self, expr):
        # 如果表达式的左侧是一个 sympy 数组（sp.Array）
        if isinstance(expr.lhs, sp.Array):
            # 将左侧和右侧分别赋值给 lhs 和 rhs
            lhs = expr.lhs
            rhs = expr.rhs

            # 如果右侧是一个 Piecewise 对象
            if isinstance(expr.rhs, sp.Piecewise):
                # 修改 Piecewise，使每个表达式都变成 Assignment，并继续打印操作
                expressions = []
                conditions = []
                for (e, c) in rhs.args:
                    expressions.append(Assignment(lhs, e))
                    conditions.append(c)
                temp = sp.Piecewise(*zip(expressions, conditions))
                return self._print(temp)

            # 打印左侧的代码，将其展平并输出
            lhs_code = self._print(sp.flatten(lhs.tolist()))
            
            # 打印右侧的代码，将其展开并添加 '.ravel()' 后缀
            rhs_code = self._print(rhs) + '.ravel()'
            
            # 返回赋值语句，格式为 "lhs_code = rhs_code"
            return self._get_statement("%s = %s" % (lhs_code, rhs_code))
            
            # 下面的代码段看起来像是作者的一些备忘或尝试，但目前被注释掉了

        else:
            # 如果右侧是 Piecewise 对象
            if expr.rhs.func == sp.Piecewise:
                # 打印左侧和右侧，返回赋值语句
                lhs_code = self._print(expr.lhs)
                rhs_code = self._print(expr.rhs)
                return self._get_statement("%s = %s" % (lhs_code, rhs_code))
            
            # 对于其他情况，调用父类的方法进行处理
            return super()._print_Assignment(expr)
# 定义 ModuleNumPyPrinter 类，继承自 ArrayNumPyPrinter 类
class ModuleNumPyPrinter(ArrayNumPyPrinter):
    # 初始化方法，接受多个关键字参数
    def __init__(self, for_class=False, **kwargs):
        # 弹出 function_modules 关键字参数并赋值给 new_modules
        new_modules = kwargs.pop('function_modules', {})
        # 弹出 function_names 关键字参数并赋值给 new_func_names
        new_func_names = kwargs.pop('function_names', {})
        # 弹出 constant_modules 关键字参数并赋值给 new_constant_modules
        new_constant_modules = kwargs.pop('constant_modules', {})
        # 弹出 constant_names 关键字参数并赋值给 new_constant_names
        new_constant_names = kwargs.pop('constant_names', {})

        # 调用父类 ArrayNumPyPrinter 的初始化方法，并传入剩余的关键字参数
        super().__init__(**kwargs)

        # 设置 for_class 属性，表示是否为类方法
        self.for_class = for_class  # --> prepend self to args, assume unknown functions are in self
        # TODO: also assume constants are in self, but need to distinguish from inputs and intermediate computations

        # 初始化已知的函数模块字典，初始包含 None: 'numpy'
        self.known_func_modules = {None: 'numpy'}
        # 更新已知的函数模块字典，使用传入的 new_modules
        self.known_func_modules.update(new_modules)

        # 初始化已知的函数名字典，使用传入的 new_func_names
        self.known_func_names = new_func_names

        # 初始化已知的常量模块字典，使用传入的 new_constant_modules
        self.known_constant_modules = new_constant_modules
        # 初始化已知的常量名字典，使用传入的 new_constant_names
        self.known_constant_names = new_constant_names

        # 初始化不支持的函数集合
        self.functions_not_supported = set()
        # 初始化不支持的常量集合
        self.constants_not_supported = set()

    # 定义 _traverse_matrix_indices 方法，用于遍历矩阵的索引
    def _traverse_matrix_indices(self, mat):
        # 获取矩阵的行数和列数
        rows, cols = mat.shape
        # 返回生成器，生成所有的矩阵索引
        return ((i, j) for i in range(rows) for j in range(cols))
        
    # 定义 _print_Symbol 方法，用于打印符号表达式
    def _print_Symbol(self, expr):
        # 调用父类 ArrayNumPyPrinter 的 _print_Symbol 方法
        super_ret = super()._print_Symbol(expr)
        # 如果符号在已知的常量模块字典中
        if expr in self.known_constant_modules:
            # 获取符号对应的模块
            module = self.known_constant_modules[expr]
            # 如果符号在已知的常量名字典中
            if expr in self.known_constant_names:
                const_name = self.known_constant_names[expr]
            else:
                const_name = super_ret
            # 如果模块不为空
            if module != '':
                # 返回模块格式化后的常量名称
                return self._module_format('.'.join([module, const_name]))
            else:
                # 返回常量名称
                return const_name
        # 返回父类返回的结果
        return super_ret
 
    # 定义 _print_Function 方法，用于打印函数表达式
    def _print_Function(self, expr):
        # 获取函数对象
        func = expr.func
        # 如果函数只有一个参数且为动态符号 _t
        if (len(expr.args) == 1) and (expr.args[0] == dynamicsymbols._t):
            # 返回函数名称
            return func.__name__
        # 如果函数不在已知函数集合中
        elif func not in self.known_functions:
            # 如果函数在已知函数模块字典中
            if func in self.known_func_modules:
                module = self.known_func_modules[func]
            # 如果为类方法
            elif self.for_class:
                module = 'self'
            else:
                module = ''
            # 如果函数在已知函数名字典中
            if func in self.known_func_names:
                func_name = self.known_func_names[func]
            else:
                func_name = func.__name__
            # 如果模块不为空
            if module != '':
                # 格式化输出模块和函数名及参数
                func_text = self._module_format('.'.join([module, func_name]))
            else:
                func_text = func_name
            # 将表达式的参数打印为字符串，使用逗号分隔
            args = ','.join(self._print(i) for i in expr.args)
            # 返回格式化后的函数调用字符串
            return "{}({})".format(func_text, args)
        else:
            # 调用父类的 _print_Function 方法
            return super()._print_Function(expr)

    # 定义 _print_Assignment 方法，用于打印赋值表达式
    def _print_Assignment(self, expr):
        # 如果表达式的左侧具有 shape 属性（即为矩阵）
        if hasattr(expr.lhs, 'shape'):
            # 遍历左侧每个元素
            for lhs_element in expr.lhs:
                # 将左侧元素添加到已知常量模块字典中，模块为空字符串
                self.known_constant_modules[lhs_element] = ''
        else:
            # 将左侧表达式添加到已知常量模块字典中，模块为空字符串
            self.known_constant_modules[expr.lhs] = ''
        # 调用父类的 _print_Assignment 方法，并返回其结果
        return super()._print_Assignment(expr)
    # 定义一个特殊方法 __delattr__，用于删除对象的属性
    def __delattr__(self, attr):
        # 如果要删除的属性是 '_not_supported'
        if attr == '_not_supported':
            # 将 self._not_supported 中的元素加入 self.functions_not_supported 中
            self.functions_not_supported = self.functions_not_supported.union(self._not_supported)
        # 调用父类的 __delattr__ 方法来删除属性
        return super().__delattr__(attr)
class ModulePrinter(_EvaluatorPrinter):
    _safe_star_args = re.compile(r'^\*{0,2}')

    # 如果是 Python 3，检查标识符是否安全
    @classmethod
    def _is_safe_ident(cls, ident):
        return isinstance(ident, string_types) and cls._safe_star_args.sub('',ident).isidentifier() \
                and not keyword.iskeyword(ident)

    # 如果不是 Python 3，检查标识符是否安全
    @classmethod
    def _is_safe_ident(cls, ident):
        return isinstance(ident, string_types) and cls._safe_ident_re.match(cls._safe_star_args.sub('',ident)) \
                and not (keyword.iskeyword(ident) or ident == 'None')

    # 定义代码打印函数，生成打印的代码行
    def codeprint(self, func_arg_expr_s, extra_pre_lines='', extra_post_lines=''):
        # 获取打印机对象
        printer = self._exprrepr.__self__
        lines = []
        # 将所有已知的函数模块和常量模块导入
        for module in set(printer.known_func_modules.values()) | set(printer.known_constant_modules.values()):
            # 排除空模块和特定情况下的self模块
            if module != '' and not (printer.for_class and module == 'self'):
                lines.append("import {}".format(module))
        # 添加额外的前置行
        if extra_pre_lines != '':
            lines.append(extra_pre_lines)
        lines.append("\n")
        # 遍历函数参数表达式
        for func_arg_expr in func_arg_expr_s:
            if len(func_arg_expr)==4:
                # 执行字典赋值？可能在此处不需要额外的操作，如果打印机在赋值时已处理
                pass
            # 执行打印操作并获取函数对象
            lines.append(self.doprint(*func_arg_expr))
            func = sp.Function(func_arg_expr[0])
            # 如果函数不在已知函数模块中，则添加到已知函数模块中
            if func not in printer.known_func_modules:
                printer.known_func_modules[func] = ''
            
            # 遍历表达式中的未应用函数符号
            for func in func_arg_expr[2].atoms(AppliedUndef):
                # 如果函数不在已知函数模块中，且参数个数不为1或不等于dynamicsymbols._t，则引发异常
                if func.func not in printer.known_func_modules:
                    if (len(func.args) == 1) and (func.args[0] == dynamicsymbols._t):
                        pass
                    else:
                        raise ValueError("Unknown function {} in ModulePrinter".format(func.__class__.__name__))
            # 遍历表达式中的符号常量
            for const in func_arg_expr[2].atoms(sp.Symbol):
                # 如果常量不在已知常量模块中且不在参数列表中，则引发异常
                if const not in printer.known_constant_modules and const not in func_arg_expr[1]:
                    if const == dynamicsymbols._t:
                        pass
                    else:
                        raise ValueError("Unknown symbol {} in ModulePrinter".format(str(const)))

        # 添加额外的后置行
        if extra_post_lines != '':
            lines.append(extra_post_lines)

        # 如果存在不支持的函数，则引发异常
        if len(printer.functions_not_supported):
            raise ValueError("Unknown functions in ModulePrinter", printer.functions_not_supported)
        
        # 返回所有生成的代码行
        return "\n".join(lines)
    # 定义一个方法，用于生成函数的字符串表示形式
    def doprint(self, funcname, args, expr, extra_assignments=None):
        """
        Returns the function definition code as a string.
        
        extra_assignments is a dict of extra assignment definitions or None
        """
        
        # 导入 sympy 模块中的 Dummy 类
        from sympy import Dummy
        
        # 如果额外赋值项 extra_assignments 为 None，则设为空字典
        if extra_assignments is None:
            extra_assignments = {}

        # 初始化函数体列表
        funcbody = []

        # 如果 args 不可迭代，则转为单元素列表
        if not iterable(args):
            args = [args]

        # 对参数和表达式进行预处理，获取参数字符串列表和处理后的表达式
        argstrs, expr = self._preprocess(args, expr)

        # 生成参数解包和最终参数列表
        funcargs = []   # 存储最终的函数参数
        unpackings = [] # 存储参数解包的代码

        # 遍历参数字符串列表
        for argstr in argstrs:
            if iterable(argstr):  # 如果参数可迭代（表示需要解包）
                funcargs.append(self._argrepr(Dummy()))  # 添加一个虚拟参数表示
                unpackings.extend(self._print_unpacking(argstr, funcargs[-1]))  # 执行解包操作
            else:
                funcargs.append(argstr)  # 直接添加参数字符串

        # 如果表达式所属类的标志为真（即表达式需要 self 参数）
        if self._exprrepr.__self__.for_class:
            funcargs.insert(0, 'self')  # 在参数列表最前面加入 self

        # 构建函数签名，格式为 def 函数名(参数列表)
        funcsig = 'def {}({}):'.format(funcname, ', '.join(funcargs))

        # 在解包之前，对输入参数进行包装处理
        funcbody.extend(self._print_funcargwrapping(funcargs))

        # 将解包操作添加到函数体中
        funcbody.extend(unpackings)
        
        # 遍历额外赋值项字典，对每一个键值对执行表达式表示并添加到函数体中
        for lhs, rhs in extra_assignments.items():
            funcbody.append(self._exprrepr(Assignment(lhs, rhs)))

        # 返回表达式的字符串表示形式作为函数的返回值
        funcbody.append('return ({})'.format(self._exprrepr(expr)))

        # 构建完整的函数代码行列表
        funclines = [funcsig]  # 添加函数签名作为第一行
        funclines.extend('    ' + line for line in funcbody)  # 缩进添加函数体的每一行

        # 将所有函数代码行连接成一个字符串并返回
        return '\n'.join(funclines) + '\n'
```