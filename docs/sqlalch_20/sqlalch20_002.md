# SQLAlchemy Unified Tutorial

> 原文：[`docs.sqlalchemy.org/en/20/tutorial/index.html`](https://docs.sqlalchemy.org/en/20/tutorial/index.html)

关于本文档

SQLAlchemy Unified Tutorial 在 SQLAlchemy 的 Core 和 ORM 组件之间集成，并作为 SQLAlchemy 整体的统一介绍。对于在 1.x 系列中使用 SQLAlchemy 的用户，在 2.0 风格下工作的用户，ORM 使用带有`select()`构造的 Core 风格查询，并且 Core 连接和 ORM 会话之间的事务语义是等效的。注意每个部分的蓝色边框样式，这将告诉您一个特定主题有多“ORM 式”！

对于已经熟悉 SQLAlchemy 的用户，特别是那些希望将现有应用程序迁移到 SQLAlchemy 2.0 系列下的 1.4 过渡阶段的用户，也应查看 SQLAlchemy 2.0 - 重大迁移指南文档。

对于新手来说，这份文档包含**大量**细节，但到最后他们将被视为**炼金术士**。

SQLAlchemy 被呈现为两个不同的 API，一个建立在另一个之上。这些 API 被称为**Core**和**ORM**。

**SQLAlchemy Core**是 SQLAlchemy 作为“数据库工具包”的基础架构。该库提供了管理与数据库的连接、与数据库查询和结果的交互以及 SQL 语句的编程构造的工具。

**主要仅 Core**的部分不会提到 ORM。在这些部分中使用的 SQLAlchemy 构造将从`sqlalchemy`命名空间导入。作为主题分类的另一个指示符，它们还将在右侧包括一个**深蓝色边框**。在使用 ORM 时，这些概念仍然存在，但在用户代码中不太明显。ORM 用户应阅读这些部分，但不要期望直接使用这些 API 来编写 ORM 中心的代码。

**SQLAlchemy ORM**建立在 Core 之上，提供了可选的**对象关系映射**功能。ORM 提供了一个额外的配置层，允许用户定义的 Python 类被**映射**到数据库表和其他构造，以及一个称为**Session**的对象持久性机制。然后，它扩展了 Core 级别的 SQL 表达语言，允许以用户定义的对象的术语来组合和调用 SQL 查询。

**主要仅 ORM**的部分应该**标题包含“ORM”短语**，以便清楚地表明这是一个与 ORM 相关的主题。在这些部分中使用的 SQLAlchemy 构造将从`sqlalchemy.orm`命名空间导入。最后，作为主题分类的另一个指示符，它们还将在左侧包括一个**浅蓝色边框**。仅 Core 的用户可以跳过这些部分。

**大多数**本教程中的部分都讨论了**与 ORM 明确使用的核心概念**。特别是 SQLAlchemy 2.0 在 ORM 中更大程度地整合了核心 API 的使用。

对于这些部分的每一个，都会有**介绍性文本**讨论 ORM 用户应该期望使用这些编程模式的程度。这些部分中的 SQLAlchemy 构造将从`sqlalchemy`命名空间中导入，同时可能会使用一些`sqlalchemy.orm`构造。作为主题分类的额外指示，这些部分还将在左侧具有较薄的浅色边框，并在右侧具有较粗的深色边框。核心和 ORM 用户应该同样熟悉这些部分的概念。

## 教程概述

教程将以应该学习的自然顺序呈现这两个概念，首先是以主要基于核心的方式，然后扩展到更多的 ORM 中心的概念。

本教程的主要部分如下：

+   建立连接 - Engine - 所有的 SQLAlchemy 应用都以一个`Engine`对象开始；这是如何创建一个的方法。

+   处理事务和 DBAPI - 此处介绍了`Engine`及其相关对象`Connection`和`Result`的使用 API。此内容是核心为中心的，但 ORM 用户至少应熟悉`Result`对象。

+   处理数据库元数据 - SQLAlchemy 的 SQL 抽象以及 ORM 都依赖于将数据库模式构造定义为 Python 对象的系统。本节介绍了如何从核心和 ORM 的角度进行操作。

+   处理数据 - 在这里我们学习如何在数据库中创建、选择、更新和删除数据。这里所谓的 CRUD 操作以 SQLAlchemy 核心的形式给出，并链接到其 ORM 对应项。在使用 SELECT 语句中详细介绍的 SELECT 操作同样适用于核心和 ORM。

+   使用 ORM 进行数据操作 - 涵盖了 ORM 的持久化框架；基本上是 ORM 为中心的插入、更新和删除的方式，以及如何处理事务。

+   处理 ORM 相关对象介绍了 `relationship()` 构造的概念，并简要概述了其用法，并提供了链接到更深入文档的链接。

+   进一步阅读 列出了一系列完全记录了本教程介绍的概念的顶级文档部分。

### 版本检查

本教程是使用称为 [doctest](https://docs.python.org/3/library/doctest.html) 的系统编写的。所有带有 `>>>` 的代码摘录实际上都作为 SQLAlchemy 的测试套件的一部分运行，并且读者被邀请使用他们自己的 Python 解释器实时处理给定的代码示例。

如果运行示例，建议读者执行快速检查以验证我们在 **版本 2.0** 的 SQLAlchemy 上：

```py
>>> import sqlalchemy
>>> sqlalchemy.__version__  
2.0.0
```

## 教程概述

本教程将按照应该学习的自然顺序呈现这两个概念，首先是基本的 Core 方法，然后扩展到更多的 ORM 方法。

本教程的主要部分如下：

+   建立连接 - Engine - 所有的 SQLAlchemy 应用程序都始于一个 `Engine` 对象；这里介绍如何创建一个。

+   处理事务和 DBAPI - 这里介绍了 `Engine` 及其相关对象 `Connection` 和 `Result` 的使用 API。这部分内容主要围绕 Core，但 ORM 用户至少要熟悉 `Result` 对象。

+   处理数据库元数据 - SQLAlchemy 的 SQL 抽象以及 ORM 都依赖于将数据库模式构造定义为 Python 对象的系统。本节介绍了如何从 Core 和 ORM 的角度来做到这一点。

+   处理数据 - 这里我们学习如何在数据库中创建、选择、更新和删除数据。这里所谓的 CRUD 操作以 SQLAlchemy Core 的术语给出，并链接到其 ORM 对应项。在 使用 SELECT 语句 中详细介绍的 SELECT 操作同样适用于 Core 和 ORM。

+   使用 ORM 进行数据操作涵盖了 ORM 的持久性框架；基本上是 ORM-centric 的插入、更新和删除方式，以及如何处理事务。

+   使用 ORM 相关对象介绍了`relationship()`构造的概念，并简要介绍了它的用法，并提供了更深入文档的链接。

+   进一步阅读列出了一系列主要的顶级文档部分，这些部分完全记录了本教程中介绍的概念。

### 版本检查

本教程是使用名为[doctest](https://docs.python.org/3/library/doctest.html)的系统编写的。 所有使用`>>>`编写的代码摘录实际上都作为 SQLAlchemy 测试套件的一部分运行，并且读者被邀请实时使用自己的 Python 解释器与给出的代码示例一起工作。

如果运行示例，则建议读者进行快速检查以验证我们使用的是**2.0 版本**的 SQLAlchemy：

```py
>>> import sqlalchemy
>>> sqlalchemy.__version__  
2.0.0
```

### 版本检查

本教程是使用名为[doctest](https://docs.python.org/3/library/doctest.html)的系统编写的。 所有使用`>>>`编写的代码摘录实际上都作为 SQLAlchemy 测试套件的一部分运行，并且读者被邀请实时使用自己的 Python 解释器与给出的代码示例一起工作。

如果运行示例，则建议读者进行快速检查以验证我们使用的是**2.0 版本**的 SQLAlchemy：

```py
>>> import sqlalchemy
>>> sqlalchemy.__version__  
2.0.0
```
