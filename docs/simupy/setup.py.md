# `.\simupy\setup.py`

```
# 导入必要的函数和模块
from setuptools import setup, find_packages
from codecs import open  # 用于处理文件编码的模块
from os import path  # 提供了处理文件和目录路径的工具函数

here = path.abspath(path.dirname(__file__))  # 获取当前脚本所在目录的绝对路径

# 获取版本信息
exec(open('simupy/version.py').read())

# 从 README 文件中获取长描述信息
with open(path.join(here, 'README.rst'), encoding='utf-8') as f:
    long_description = f.read()

# 替换长描述中的链接
long_description = long_description.replace(
    "https://simupy.readthedocs.io/en/latest/",
    "https://simupy.readthedocs.io/en/simupy-{}/".format(
        '.'.join(__version__.split('.')[:3])
    )
)

# 设置安装配置
setup(
    name='simupy',  # 包名
    version=__version__,  # 版本号
    description='A framework for modeling and simulating dynamical systems.',  # 简短描述
    long_description=long_description,  # 长描述
    packages=find_packages(),  # 查找所有包含 __init__.py 的包
    author='Benjamin Margolis',  # 作者名
    author_email='ben@sixpearls.com',  # 作者邮箱
    url='https://github.com/simupy/simupy',  # 项目 URL
    license="BSD 2-clause \"Simplified\" License",  # 许可证类型
    python_requires='>=3',  # 所需 Python 版本
    install_requires=['numpy>=1.11.3', 'scipy>=0.18.1'],  # 安装所需的依赖包
    extras_require={
        'symbolic': ['sympy>=1.0'],  # 针对不同需求提供的额外依赖包
        'doc': ['sphinx>=1.6.3', 'sympy>=1.0'],
        'examples': ['matplotlib>=2.0', 'sympy>=1.0'],
    },

    classifiers=[  # 包的分类标签列表
        'License :: OSI Approved :: BSD License',
        'Programming Language :: Python :: 3',
        'Intended Audience :: Education',
        'Intended Audience :: Science/Research',
        'Operating System :: OS Independent',
        'Topic :: Scientific/Engineering :: Physics',
        'Topic :: Scientific/Engineering :: Mathematics',
    ],
)
```