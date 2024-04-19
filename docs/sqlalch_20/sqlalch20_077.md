# SQL 语句和表达式 API

> 原文：[`docs.sqlalchemy.org/en/20/core/expression_api.html`](https://docs.sqlalchemy.org/en/20/core/expression_api.html)

本节介绍了 SQL 表达式语言的 API 参考。要了解简介，请从 SQLAlchemy 统一教程中的数据操作开始。

+   列元素和表达式

    +   列元素基础构造函数

        +   `and_()`

        +   `bindparam()`

        +   `bitwise_not()`

        +   `case()`

        +   `cast()`

        +   `column()`

        +   `custom_op`

        +   `distinct()`

        +   `extract()`

        +   `false()`

        +   `func`

        +   `lambda_stmt()`

        +   `literal()`

        +   `literal_column()`

        +   `not_()`

        +   `null()`

        +   `or_()`

        +   `outparam()`

        +   `text()`

        +   `true()`

        +   `try_cast()`

        +   `tuple_()`

        +   `type_coerce()`

        +   `quoted_name`

    +   列元素修饰符构造函数

        +   `all_()`

        +   `any_()`

        +   `asc()`

        +   `between()`

        +   `collate()`

        +   `desc()`

        +   `funcfilter()`

        +   `label()`

        +   `nulls_first()`

        +   `nullsfirst()`

        +   `nulls_last()`

        +   `nullslast()`

        +   `over()`

        +   `within_group()`

    +   列元素类文档

        +   `BinaryExpression`

        +   `BindParameter`

        +   `Case`

        +   `Cast`

        +   `ClauseList`

        +   `ColumnClause`

        +   `ColumnCollection`

        +   `ColumnElement`

        +   `ColumnExpressionArgument`

        +   `ColumnOperators`

        +   `Extract`

        +   `False_`

        +   `FunctionFilter`

        +   `Label`

        +   `Null`

        +   `Operators`

        +   `Over`

        +   `SQLColumnExpression`

        +   `TextClause`

        +   `TryCast`

        +   `Tuple`

        +   `WithinGroup`

        +   `WrapsColumnExpression`

        +   `True_`

        +   `TypeCoerce`

        +   `UnaryExpression`

    +   列元素类型工具

        +   `NotNullable()`

        +   `Nullable()`

+   运算符参考

    +   比较运算符

    +   IN 比较

        +   IN 对值列表的比较

        +   空 IN 表达式

        +   NOT IN

        +   元组 IN 表达式

        +   子查询 IN

    +   身份比较

    +   字符串比较

    +   字符串包含

    +   字符串匹配

    +   字符串修改

    +   算术运算符

    +   位运算操作符

    +   使用连接词和否定词

    +   连接操作符

+   SELECT 和相关构造

    +   可选择的基础构造函数

        +   `except_()`

        +   `except_all()`

        +   `exists()`

        +   `intersect()`

        +   `intersect_all()`

        +   `select()`

        +   `table()`

        +   `union()`

        +   `union_all()`

        +   `values()`

    +   可选择的修改构造函数

        +   `alias()`

        +   `cte()`

        +   `join()`

        +   `lateral()`

        +   `outerjoin()`

        +   `tablesample()`

    +   可选择类文档

        +   `Alias`

        +   `AliasedReturnsRows`

        +   `CompoundSelect`

        +   `CTE`

        +   `Executable`

        +   `Exists`

        +   `FromClause`

        +   `GenerativeSelect`

        +   `HasCTE`

        +   `HasPrefixes`

        +   `HasSuffixes`

        +   `Join`

        +   `Lateral`

        +   `ReturnsRows`

        +   `ScalarSelect`

        +   `Select`

        +   `Selectable`

        +   `SelectBase`

        +   `Subquery`

        +   `TableClause`

        +   `TableSample`

        +   `TableValuedAlias`

        +   `TextualSelect`

        +   `Values`

        +   `ScalarValues`

    +   标签样式常量

        +   `SelectLabelStyle`

+   插入、更新、删除

    +   DML 基础构造函数

        +   `delete()`

        +   `insert()`

        +   `update()`

    +   DML 类文档构造函数

        +   `Delete`

        +   `Insert`

        +   `Update`

        +   `UpdateBase`

        +   `ValuesBase`

+   SQL 和通用函数

    +   函数 API

        +   `AnsiFunction`

        +   `Function`

        +   `FunctionElement`

        +   `GenericFunction`

        +   `register_function()`

    +   已选中的“已知”函数

        +   `aggregate_strings`

        +   `array_agg`

        +   `char_length`

        +   `coalesce`

        +   `concat`

        +   `count`

        +   `cube`

        +   `cume_dist`

        +   `current_date`

        +   `current_time`

        +   `current_timestamp`

        +   `current_user`

        +   `dense_rank`

        +   `grouping_sets`

        +   `localtime`

        +   `localtimestamp`

        +   `max`

        +   `min`

        +   `mode`

        +   `next_value`

        +   `now`

        +   `percent_rank`

        +   `percentile_cont`

        +   `percentile_disc`

        +   `random`

        +   `rank`

        +   `rollup`

        +   `session_user`

        +   `sum`

        +   `sysdate`

        +   `user`

+   自定义 SQL 构造和编译扩展

    +   简介

    +   特定于方言的编译规则

    +   编译自定义表达式构造的子元素

        +   SQL 和 DDL 编译器之间的交叉编译

    +   更改现有构造的默认编译

    +   更改类型的编译

    +   子类化指南

    +   为自定义构造启用缓存支持

    +   更多示例

        +   “UTC 时间戳” 函数

        +   “GREATEST” 函数

        +   “false” 表达式

    +   `compiles()`

    +   `deregister()`

+   表达式序列化扩展

    +   `Deserializer()`

    +   `Serializer()`

    +   `dumps()`

    +   `loads()`

+   SQL 表达式语言基础构造

    +   `CacheKey`

        +   `CacheKey.bindparams`

        +   `CacheKey.key`

        +   `CacheKey.to_offline_string()`

    +   `ClauseElement`

        +   `ClauseElement.compare()`

        +   `ClauseElement.compile()`

        +   `ClauseElement.get_children()`

        +   `ClauseElement.inherit_cache`

        +   `ClauseElement.params()`

        +   `ClauseElement.self_group()`

        +   `ClauseElement.unique_params()`

    +   `DialectKWArgs`

        +   `DialectKWArgs.argument_for()`

        +   `DialectKWArgs.dialect_kwargs`

        +   `DialectKWArgs.dialect_options`

        +   `DialectKWArgs.kwargs`

    +   `HasCacheKey`

        +   `HasCacheKey.inherit_cache`

    +   `LambdaElement`

    +   `StatementLambdaElement`

        +   `StatementLambdaElement.add_criteria()`

        +   `StatementLambdaElement.is_delete`

        +   `StatementLambdaElement.is_dml`

        +   `StatementLambdaElement.is_insert`

        +   `StatementLambdaElement.is_select`

        +   `StatementLambdaElement.is_text`

        +   `StatementLambdaElement.is_update`

        +   `StatementLambdaElement.spoil()`

+   Visitor and Traversal Utilities

    +   `ExternalTraversal`

        +   `ExternalTraversal.chain()`

        +   `ExternalTraversal.iterate()`

        +   `ExternalTraversal.traverse()`

        +   `ExternalTraversal.visitor_iterator`

    +   `InternalTraversal`

        +   `InternalTraversal.dp_annotations_key`

        +   `InternalTraversal.dp_anon_name`

        +   `InternalTraversal.dp_boolean`

        +   `InternalTraversal.dp_clauseelement`

        +   `InternalTraversal.dp_clauseelement_list`

        +   `InternalTraversal.dp_clauseelement_tuple`

        +   `InternalTraversal.dp_clauseelement_tuples`

        +   `InternalTraversal.dp_dialect_options`

        +   `InternalTraversal.dp_dml_multi_values`

        +   `InternalTraversal.dp_dml_ordered_values`

        +   `InternalTraversal.dp_dml_values`

        +   `InternalTraversal.dp_fromclause_canonical_column_collection`

        +   `InternalTraversal.dp_fromclause_ordered_set`

        +   `InternalTraversal.dp_has_cache_key`

        +   `InternalTraversal.dp_has_cache_key_list`

        +   `InternalTraversal.dp_has_cache_key_tuples`

        +   `InternalTraversal.dp_ignore`

        +   `InternalTraversal.dp_inspectable`

        +   `InternalTraversal.dp_inspectable_list`

        +   `InternalTraversal.dp_multi`

        +   `InternalTraversal.dp_multi_list`

        +   `InternalTraversal.dp_named_ddl_element`

        +   `InternalTraversal.dp_operator`

        +   `InternalTraversal.dp_plain_dict`

        +   `InternalTraversal.dp_plain_obj`

        +   `InternalTraversal.dp_prefix_sequence`

        +   `InternalTraversal.dp_propagate_attrs`

        +   `InternalTraversal.dp_statement_hint_list`

        +   `InternalTraversal.dp_string`

        +   `InternalTraversal.dp_string_clauseelement_dict`

        +   `InternalTraversal.dp_string_list`

        +   `InternalTraversal.dp_string_multi_dict`

        +   `InternalTraversal.dp_table_hint_list`

        +   `InternalTraversal.dp_type`

        +   `InternalTraversal.dp_unknown_structure`

    +   `Visitable`

    +   `anon_map`

    +   `cloned_traverse()`

    +   `iterate()`

    +   `replacement_traverse()`

    +   `traverse()`

    +   `traverse_using()`
