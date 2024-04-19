# 安装

> 原文：[`docs.sqlalchemy.org/en/20/faq/installation.html`](https://docs.sqlalchemy.org/en/20/faq/installation.html)

+   当我尝试使用 asyncio 时，出现了关于未安装 greenlet 的错误

## 当我尝试使用 asyncio 时，出现了关于未安装 greenlet 的错误

对于不提供[预构建二进制轮](https://pypi.org/project/greenlet/#files)的 CPU 架构，默认情况下不会安装 `greenlet` 依赖项。特别是，**这包括 Apple M1**。要安装包括 `greenlet` 的内容，请将 `asyncio` [setuptools 额外内容](https://packaging.python.org/en/latest/tutorials/installing-packages/#installing-setuptools-extras)添加到 `pip install` 命令中：

```py
pip install sqlalchemy[asyncio]
```

欲了解更多背景信息，请参阅 Asyncio 平台安装说明（包括 Apple M1）。

另请参阅

Asyncio 平台安装说明（包括 Apple M1）  ## 当我尝试使用 asyncio 时，出现了关于未安装 greenlet 的错误

对于不提供[预构建二进制轮](https://pypi.org/project/greenlet/#files)的 CPU 架构，默认情况下不会安装 `greenlet` 依赖项。特别是，**这包括 Apple M1**。要安装包括 `greenlet` 的内容，请将 `asyncio` [setuptools 额外内容](https://packaging.python.org/en/latest/tutorials/installing-packages/#installing-setuptools-extras)添加到 `pip install` 命令中：

```py
pip install sqlalchemy[asyncio]
```

欲了解更多背景信息，请参阅 Asyncio 平台安装说明（包括 Apple M1）。

另请参阅

Asyncio 平台安装说明（包括 Apple M1）
