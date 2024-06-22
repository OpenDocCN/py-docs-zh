# `.\simupy\simupy\utils\__init__.py`

```
# 导入 NumPy 库，用于科学计算
import numpy as np
# 从 SciPy 库中导入 interpolate 模块，用于插值运算
from scipy import interpolate

# 构建一个可调用的函数，根据给定的时间轨迹和曲线集合，使用样条插值生成立方 B 样条插值函数
def callable_from_trajectory(t, curves, k=3):
    """
    Use scipy.interpolate.make_interp_spline to build cubic b-spline
    interpolating functions over a set of curves.

    Parameters
    ----------
    t : 1D array_like
        Array of m time indices of trajectory
    curves : 2D array_like
        Array of m x n vector samples at the time indices. First dimension
        indexes time, second dimension indexes vector components

    Returns
    -------
    interpolated_callable : callable
        Callable which interpolates the given curve/trajectories
    """
    # 使用 make_interp_spline 函数创建立方 B 样条插值函数
    bspline = interpolate.make_interp_spline(
        y=curves, x=t, k=k)

    return bspline

# 构建一个插值样条函数，从给定的轨迹中构建一个插值样条，处理可能由事件引起的不连续性
def trajectory_interpolant(res, for_="output", cols=slice(None), k=3, bc_type="natural"):
    """
    Construct an interpolating spline from a trajectory that handles potential
    discontinuities from events.

    Parameters
    ----------
    res : `SimulationResult` object
        The `SimulationResult` object that will be interpolated
    for_ : {"state" or "output"}
        Indicate whether to interpolate the state or output of the SimulationResult.

    Returns
    -------
    interpolated_callable : callable
        Callable which interpolates the given curve/trajectories
    """
    # 根据不同的 for_ 值选择要插值的状态或输出
    if for_=="state":
        vals = res.x[:, cols]
    elif for_=="output":
        vals = res.y[:, cols]
    else:
        raise ValueError("Unsupported `for_` value")

    # 找到事件发生的索引位置
    event_idxs = np.unique(np.where(np.diff(np.sign(res.e), axis=0)!=0)[0])+1
    if event_idxs.size > 0 and event_idxs[0] == 1:
        event_idxs = event_idxs[1:]
    prev_idx = 0
    interps = []
    extra_val = int((k+1)//2)
    event_idx = 0
    # 根据事件位置分段构建插值样条函数
    for event_idx in event_idxs:
        interps.append(
            interpolate.make_interp_spline(
                res.t[prev_idx:event_idx],
                vals[prev_idx:event_idx, :],
                k=k, bc_type=bc_type
            )
        )
        prev_idx = event_idx
    if prev_idx < res.t.shape[0]:
        interps.append(
            interpolate.make_interp_spline(
                res.t[event_idx:],
                vals[event_idx:, :],
                k=k, bc_type=bc_type
            )
        )

    ccl = []
    # 构建连接系数，处理不同插值段之间的连接
    for interp in interps:
        if interp is not interps[0]:
            ccl.append(interp.c[[0]*extra_val, :])

        ccl.append(interp.c)

        if interp is not interps[-1]:
            ccl.append(interp.c[[-1]*extra_val, :])

    cc = np.concatenate(ccl)
    tt = np.concatenate([interp.t for interp in interps])
    # 使用快速构建方法构建 BSpline 对象
    return interpolate.BSpline.construct_fast(tt, cc, k)

def trajectory_linear_combination(res1, res2, coeff1=1, coeff2=1, for_="output", cols=slice(None), k=3, bc_type="natural"):
    # 根据 for_ 值选择要线性组合的状态或输出的形状
    if for_=="state":
        shape1 = res1.x.shape[1]
        shape2 = res2.x.shape[1]
    elif for_=="output":
        shape1 = res1.y.shape[1]
        shape2 = res2.y.shape[1]
    # 如果`for_`的值不在支持的列表中，抛出数值错误异常
    else:
        raise ValueError("Unsupported `for_` value")

    # 将res1和res2的时间点合并并排序，转换为NumPy数组
    eval_t_list = list(set(res1.t) | set(res2.t))
    eval_t_list.sort()
    eval_t = np.array(eval_t_list)

    # 使用给定的参数构建两个插值对象
    interp1 = trajectory_interpolant(res1, for_, cols, k, bc_type)
    interp2 = trajectory_interpolant(res2, for_, cols, k, bc_type)

    # 对于每个评估时间点t，如果interp1中没有，则插入插值
    for t in eval_t:
        if t not in interp1.t:
            interp1 = interpolate.insert(t, interp1)
        # 对于每个评估时间点t，如果interp2中没有，则插入插值
        if t not in interp2.t:
            interp2 = interpolate.insert(t, interp2)

    # 检查插值对象的节点是否相同
    if not np.all(interp1.t == interp2.t):
        raise ValueError("Expected to construct the same knot structure for each interpolant")

    # 检查插值对象的节点是否完全匹配评估时间点
    if not np.all(np.unique(interp1.t) == eval_t):
        raise ValueError("Expected the knots to lie on eval_t")

    # 使用加权系数coeff1和coeff2，基于插值对象构建快速BSpline插值
    return interpolate.BSpline.construct_fast(interp1.t,
                                              interp1.c*coeff1 + interp2.c*coeff2,
                                              k)
def trajectory_norm(interp, p=2):
    # 获取插值对象的唯一时间点
    eval_t = np.unique(interp.t)
    # 如果 p 为无穷大，返回插值函数在所有时间点上的最大值
    if p == np.inf:
        return np.max(np.abs(interp(eval_t)), axis=0)
    # 如果 p 为负无穷，返回插值函数在所有时间点上的最小值
    if p == -np.inf:
        return np.min(np.abs(interp(eval_t)), axis=0)
    # 如果 p 是整数，构造积分被插值函数绝对值的 BSpline
    if isinstance(p, int):
        integrand = interpolate.BSpline.construct_fast(interp.t, np.abs(interp.c)**p,
                                                       interp.k)
        # 返回积分函数在最后一个时间点的值的 p 次方根除以时间点首尾差值
        return (integrand.antiderivative()(eval_t[-1])**(1/p))/np.diff(eval_t[[0,-1]])
    # 如果 p 的类型不符合预期，抛出异常
    raise ValueError("unexpected value for p")


def isclose(res1, res2, p=np.Inf, atol=1E-8, rtol=1E-5, mode='numpy', for_="output",
            cols=slice(None), k=3, bc_type="natural"):
    """
    Compare two trajectories

    Parameters
    ---------- 
    mode : {'numpy' or 'pep485'}
        比较模式，可以是 'numpy' 或 'pep485'
    """
    # 计算两个轨迹之间的线性组合差异
    traj_diff = trajectory_linear_combination(res1, res2, coeff1=1, coeff2=-1,
                                              for_=for_, cols=cols, k=k, bc_type=bc_type)
    # 计算差异轨迹的范数
    diff_norm = trajectory_norm(traj_diff, p)
    # 计算第一个结果轨迹的范数
    res1_norm = trajectory_norm(trajectory_interpolant(res1, for_=for_, cols=cols, k=k,
                                                       bc_type=bc_type), p)
    # 计算第二个结果轨迹的范数
    res2_norm = trajectory_norm(trajectory_interpolant(res2, for_=for_, cols=cols, k=k,
                                                       bc_type=bc_type), p)
    # 根据比较模式进行比较
    if mode=='numpy':
        return (diff_norm <= (atol + rtol*res2_norm))
    elif mode=='pep485':
        return (diff_norm <= np.clip(
                    rtol*np.max(np.stack([res1_norm, res2_norm]), axis=0),
                    a_min=atol, a_max=None)
                )


def discrete_callable_from_trajectory(t, curves):
    """
    Build a callable that interpolates a discrete-time curve by returning the
    value of the previous time-step.

    Parameters
    ----------
    t : 1D array_like
        Array of m time indices of trajectory
    curves : 2D array_like
        Array of m x n vector samples at the time indices. First dimension
        indexes time, second dimension indexes vector components

    Returns
    -------
    nearest_neighbor_callable : callable
        Callable which interpolates the given discrete-time curve/trajectories
    """
    # 复制时间和曲线数据
    local_time = np.array(t).copy()
    local_curves = np.array(curves).reshape(local_time.shape[0], -1).copy()
    # 定义最近邻可调用函数
    def nearest_neighbor_callable(t, *args):
        return local_curves[
            np.argmax((local_time.reshape(1,-1)>=np.array([t]).reshape(-1,1)),
                axis=1), :]

    return nearest_neighbor_callable



def array_callable_from_vector_trajectory(tt, x, unraveled, raveled):
    """
    Convert a trajectory into an interpolating callable that returns a 2D
    array. The unraveled, raveled pair map how the array is filled in. See
    riccati_system example.

    Parameters
    ----------
    tt : 1D array_like
        Array of m time indices of trajectory
    x : array_like
        Vector samples at the time indices. Should be consistent with unraveled and raveled.
    unraveled : array_like
        Indices of the unraveled output
    raveled : array_like
        Indices of the raveled output
    """
    # 在此省略剩余代码，因为未提供完整内容
    xx : 2D array_like
        Array of m x n vector samples at the time indices. First dimension
        indexes time, second dimension indexes vector components
    unraveled : 1D array_like
        Array of n unique keys matching xx.
    raveled : 2D array_like
        Array where the elements are the keys from unraveled. The mapping
        between unraveled and raveled is used to specify how the output array
        is filled in.

    Returns
    -------
    matrix_callable : callable
        The callable interpolating the trajectory with the specified shape.
    """
    # 获取输入矩阵 xx 的形状信息
    xn, xm = x.shape

    # 根据时间向量 tt 和样本矩阵 x 创建可调用对象 vector_callable
    vector_callable = callable_from_trajectory(tt, x)

    # 如果 unraveled 具有 'shape' 属性且为多维数组，则将其扁平化为列表
    if hasattr(unraveled, 'shape') and len(unraveled.shape) > 1:
        unraveled = np.array(unraveled).flatten().tolist()

    # 定义内部函数 array_callable，用于生成插值后的数组
    def array_callable(t):
        # 调用 vector_callable 获取插值向量结果
        vector_result = vector_callable(t)

        # 初始化结果数组 array_result，根据输入 t 的类型决定维度
        as_array = False
        if isinstance(t, (list, tuple, np.ndarray)) and len(t) > 1:
            array_result = np.zeros((len(t),)+raveled.shape)
            as_array = True
        else:
            array_result = np.zeros(raveled.shape)

        # 迭代 raveled 中的元素，使用 np.nditer 实现迭代器
        iterator = np.nditer(raveled, flags=['multi_index', 'refs_ok'])
        for it in iterator:
            # 获取当前迭代器的多维索引
            iterator.multi_index
            # 在 unraveled 中查找 raveled 当前元素的索引
            idx = unraveled.index(raveled[iterator.multi_index])
            # 根据结果数组类型，将 vector_result 的对应元素填入 array_result
            if as_array:
                array_result.__setitem__(
                    (slice(None), *iterator.multi_index),
                    vector_result[..., idx]
                )
            else:
                array_result[tuple(iterator.multi_index)] = vector_result[idx]
        return array_result

    # 返回内部函数 array_callable 作为结果
    return array_callable
```