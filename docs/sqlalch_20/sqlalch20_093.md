# SQL 数据类型对象

> 原文：[`docs.sqlalchemy.org/en/20/core/types.html`](https://docs.sqlalchemy.org/en/20/core/types.html)

*   类型层次结构

    +   “驼峰式”数据类型

    +   “大写”数据类型

    +   特定后端的“大写”数据类型

    +   在多个后端使用“大写”和特定后端类型

    +   通用“驼峰式”类型

        +   `BigInteger`

        +   `Boolean`

        +   `Date`

        +   `DateTime`

        +   `Enum`

        +   `Double`

        +   `Float`

        +   `Integer`

        +   `Interval`

        +   `LargeBinary`

        +   `MatchType`

        +   `Numeric`

        +   `PickleType`

        +   `SchemaType`

        +   `SmallInteger`

        +   `String`

        +   `Text`

        +   `Time`

        +   `Unicode`

        +   `UnicodeText`

        +   `Uuid`

    +   SQL 标准和多厂商“大写”类型

        +   `ARRAY`

        +   `BIGINT`

        +   `BINARY`

        +   `BLOB`

        +   `BOOLEAN`

        +   `CHAR`

        +   `CLOB`

        +   `DATE`

        +   `DATETIME`

        +   `DECIMAL`

        +   `DOUBLE`

        +   `DOUBLE_PRECISION`

        +   `FLOAT`

        +   `INT`

        +   `JSON`

        +   `INTEGER`

        +   `NCHAR`

        +   `NVARCHAR`

        +   `NUMERIC`

        +   `REAL`

        +   `SMALLINT`

        +   `TEXT`

        +   `TIME`

        +   `TIMESTAMP`

        +   `UUID`

        +   `VARBINARY`

        +   `VARCHAR`

+   自定义类型

    +   覆盖类型编译

    +   增强现有类型

        +   `TypeDecorator`

    +   类型修饰器示例

        +   将编码字符串强制转换为 Unicode

        +   数值四舍五入

        +   将时区感知时间戳存储为时区无关的 UTC

        +   与后端无关的 GUID 类型

        +   编组 JSON 字符串

    +   应用 SQL 级别的绑定/结果处理

    +   重新定义和创建新操作符

    +   创建新类型

        +   `UserDefinedType`

    +   使用自定义类型和反射

+   基础类型 API

    +   `TypeEngine`

        +   `TypeEngine.Comparator`

        +   `TypeEngine.adapt()`

        +   `TypeEngine.as_generic()`

        +   `TypeEngine.bind_expression()`

        +   `TypeEngine.bind_processor()`

        +   `TypeEngine.coerce_compared_value()`

        +   `TypeEngine.column_expression()`

        +   `TypeEngine.comparator_factory`

        +   `TypeEngine.compare_values()`

        +   `TypeEngine.compile()`

        +   `TypeEngine.dialect_impl()`

        +   `TypeEngine.evaluates_none()`

        +   `TypeEngine.get_dbapi_type()`

        +   `TypeEngine.hashable`

        +   `TypeEngine.literal_processor()`

        +   `TypeEngine.python_type`

        +   `TypeEngine.render_bind_cast`

        +   `TypeEngine.render_literal_cast`

        +   `TypeEngine.result_processor()`

        +   `TypeEngine.should_evaluate_none`

        +   `TypeEngine.sort_key_function`

        +   `TypeEngine.with_variant()`

    +   `Concatenable`

        +   `Concatenable.Comparator`

        +   `Concatenable.comparator_factory`

    +   `Indexable`

        +   `Indexable.Comparator`

        +   `Indexable.comparator_factory`

    +   `NullType`

    +   `ExternalType`

        +   `ExternalType.cache_ok`

    +   `Variant`

        +   `Variant.with_variant()`
