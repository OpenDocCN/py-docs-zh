# `.\simupy\tests\test_utils.py`

```
# 导入NumPy库，通常用np来引用
import numpy as np
# 从simupy.utils中导入isclose函数
from simupy.utils import isclose

# 定义测试函数test_isclose
def test_isclose():
    # 创建一个包含50个元素的等间距数组，范围从0到π/2
    t1 = np.linspace(0, np.pi/2, num=50)
    # 创建一个包含30个元素的等间距数组，范围从0到π/2
    t2 = np.linspace(0, np.pi/2, num=30)
    
    # 断言：使用isclose函数比较t1、sin(t1)与t2、sin(t2)的近似性
    assert isclose(t1, np.sin(t1), t2, np.sin(t2))
    # 断言：使用isclose函数比较t1、sin(t1)与t1、sin(t1)的近似性
    assert isclose(t1, np.sin(t1), t1, np.sin(t1))
    # 断言：使用isclose函数比较t1、sin(t1)与t1、cos(t1)的近似性，预期结果为False
    assert not isclose(t1, np.sin(t1), t1, np.cos(t1))

# 如果作为主程序运行，则执行test_isclose函数
if __name__ == '__main__':
    test_isclose()
```