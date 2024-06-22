# 用户指南

> 原文： [`docs.cython.org/en/latest/src/userguide/index.html`](http://docs.cython.org/en/latest/src/userguide/index.html)

内容：

*   语言基础知识
    *   声明数据类型
    *   C 变量和类型定义
    *   Python 函数与 C 函数
    *   自动类型转换
    *   陈述和表达
    *   Cython 文件类型
    *   条件编译
*   扩展类型
    *   简介
    *   静态属性
    *   动态属性
    *   类型声明
    *   扩展类型和无
    *   特殊方法
    *   属性
    *   子类
    *   C 方法
    *   前向声明扩展类型
    *   快速实例化
    *   来自现有 C / C ++指针的实例化
    *   使扩展类型弱引用
    *   控制 CPython 中的释放和垃圾收集
    *   控制酸洗
    *   公共和外部扩展类型
    *   公共扩展类型
*   扩展类型的特殊方法
    *   声明
    *   Docstrings
    *   初始化方法：`__cinit__()`和`__init__()`
    *   终结方法：`__dealloc__()`
    *   算术方法
    *   丰富的比较
    *   `__next__()`方法
    *   特殊方法表
*   在 Cython 模块之间共享声明
    *   定义和实施文件
    *   定义文件包含什么
    *   实施文件包含什么
    *   cimport 声明
    *   分享 C 函数
    *   分享扩展类型
*   与外部 C 代码接口
    *   外部声明
    *   使用 C 的 Cython 声明
*   源文件和编译
    *   从命令行编译
    *   基本 setup.py
    *   包中的多个 Cython 文件
    *   集成多个模块
    *   使用`pyximport` 进行编译
    *   使用`cython.inline` 进行编译
    *   与 Sage 一起编译
    *   使用 Jupyter 笔记本进行编译
    *   编译器指令
*   早期绑定速度
*   在 Cython 中使用 C ++
    *   概述
    *   一个简单的教程
    *   编制和导入
    *   高级 C ++功能
    *   RTTI 和 typeid（）
    *   在 setup.py 中指定 C ++语言
    *   警告和限制
*   融合类型（模板）
    *   快速入门
    *   声明融合类型
    *   使用融合类型
    *   选择专业化
    *   内置熔断类型
    *   熔铸函数
    *   类型检查专业化
    *   条件 GIL 获取/释放
    *   __signatures__
*   将 Cython 代码移植到 PyPy
    *   参考计数
    *   物件寿命
    *   借用参考和数据指针
    *   内置类型，插槽和字段
    *   GIL 处理
    *   效率
    *   已知问题
    *   错误和崩溃
*   限制
    *   嵌套元组参数解包
    *   检查支持
    *   堆叠帧
    *   推断文字的身份与平等
*   Cython 和派热克斯之间的差异
    *   Python 3 支持
    *   条件表达式“x if b else y”
    *   cdef inline
    *   声明转让（例如“cdef int spam = 5”）
    *   for 循环中的'by'表达式（例如“for i from 0＆lt; = i＆lt; 10 by 2”）
    *   Boolean int 类型（例如，它的行为类似于 c int，但是作为布尔值强制转换为/来自 python）
    *   可执行类主体
    *   cpdef 函数
    *   自动量程转换
    *   更友好型铸造
    *   cdef / cpdef 函数中的可选参数
    *   结构中的函数指针
    *   C ++异常处理
    *   同义词
    *   源代码编码
    *   自动`typecheck`
    *   来自 __future__ 指令
    *   纯 Python 模式
*   类型记忆视图
    *   快速入门
    *   使用记忆库视图
    *   与旧缓冲支持的比较
    *   Python 缓冲支持
    *   内存布局
    *   Memoryviews 和 GIL
    *   Memoryview 对象和 Cython 阵列
    *   Cython 阵列
    *   Python 数组模块
    *   强制对 NumPy
    *   无切片
    *   通过指针从 C 函数传递数据
*   实施缓冲协议
    *   矩阵类
    *   内存安全和引用计数
    *   旗帜
    *   参考文献
*   使用并行
    *   编制
    *   突破循环
    *   使用 OpenMP 函数
*   调试你的 Cython 程序
    *   运行调试器
    *   使用调试器
    *   便利功能
    *   配置调试器
*   Cython for NumPy 用户
    *   Cython 一览
    *   您的 Cython 环境
    *   安装
    *   手动编译
    *   第一个 Cython 程序
    *   添加类型
    *   内存视图的高效索引
    *   进一步调整索引
    *   将 NumPy 数组声明为连续
    *   使功能更清晰
    *   更通用的代码
    *   使用多线程
    *   从哪里开始？
*   Pythran 作为 Numpy 后端
    *   distutils 的用法示例

## 指数和表格

*   指数
*   模块索引
*   搜索页