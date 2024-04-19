# 方言

> 原文：[`docs.sqlalchemy.org/en/20/dialects/index.html`](https://docs.sqlalchemy.org/en/20/dialects/index.html)

**方言** 是 SQLAlchemy 用来与各种类型的 DBAPI 实现和数据库通信的系统。以下各节包含每个后端使用的特定用法的参考文档和说明，以及各种 DBAPI 的说明。

所有方言都要求安装适当的 DBAPI 驱动程序。

## 包含的方言

+   PostgreSQL

+   MySQL 和 MariaDB

+   SQLite

+   Oracle

+   Microsoft SQL Server

### 包含的方言的支持级别

以下表格总结了每个包含的方言的支持级别。

**包含的方言支持的数据库版本**

| 数据库 | 在 CI 中完全测试 | 正常支持 | 尽力而为 |
| --- | --- | --- | --- |
| **Microsoft SQL Server** | 2017 | 2012+ | 2005+ |
| **MySQL / MariaDB** | 5.6, 5.7, 8.0 / 10.8, 10.9 | 5.6+ / 10+ | 5.0.2+ / 5.0.2+ |
| **Oracle** | 18c | 11+ | 9+ |
| **PostgreSQL** | 12, 13, 14, 15 | 9.6+ | 9+ |
| **SQLite** | 3.36.0 | 3.12+ | 3.7.16+ |

### 支持定义

在 CI 中完全测试

**在 CI 中完全测试** 表示已在 sqlalchemy CI 系统中进行测试并通过测试套件中的所有测试的版本。

正常支持

**正常支持** 表示大多数功能应该可以工作，但不是所有版本都在 ci 配置中进行测试，因此可能存在一些不受支持的边缘情况。我们将尝试修复影响这些版本的问题。

尽力而为

**尽力而为** 指我们尝试在其中支持基本功能，但在某些用例中可能存在不受支持的功能或错误。我们可能会接受相关问题的拉取请求以继续支持较旧的版本，这些请求会逐案审查。  ## 外部方言

目前为 SQLAlchemy 维护的外部方言项目包括：

| 数据库 | 方言 |
| --- | --- |
| Actian Data Platform、Vector、Actian X、Ingres | [sqlalchemy-ingres](https://github.com/ActianCorp/sqlalchemy-ingres) |
| Amazon Athena | [pyathena](https://github.com/laughingman7743/PyAthena/) |
| 亚马逊 Redshift (通过 psycopg2) | [sqlalchemy-redshift](https://pypi.org/project/sqlalchemy-redshift) |
| Apache Drill | [sqlalchemy-drill](https://github.com/JohnOmernik/sqlalchemy-drill) |
| Apache Druid | [pydruid](https://github.com/druid-io/pydruid) |
| Apache Hive 和 Presto | [PyHive](https://github.com/dropbox/PyHive#sqlalchemy) |
| Apache Solr | [sqlalchemy-solr](https://github.com/aadel/sqlalchemy-solr) |
| CockroachDB | [sqlalchemy-cockroachdb](https://github.com/cockroachdb/sqlalchemy-cockroachdb) |
| CrateDB | [crate-python](https://github.com/crate/crate-python) |
| EXASolution | [sqlalchemy_exasol](https://github.com/blue-yonder/sqlalchemy_exasol) |
| Elasticsearch（只读） | [elasticsearch-dbapi](https://github.com/preset-io/elasticsearch-dbapi/) |
| Firebird | [sqlalchemy-firebird](https://github.com/pauldex/sqlalchemy-firebird) |
| Firebolt | [firebolt-sqlalchemy](https://pypi.org/project/firebolt-sqlalchemy/) |
| Google BigQuery | [pybigquery](https://github.com/mxmzdlv/pybigquery/) |
| Google Sheets | [gsheets](https://github.com/betodealmeida/gsheets-db-api) |
| IBM DB2 和 Informix | [ibm-db-sa](https://pypi.org/project/ibm-db-sa/) |
| IBM Netezza Performance Server [[1]](#id5) | [nzalchemy](https://pypi.org/project/nzalchemy/) |
| Impala | [impyla](https://pypi.org/project/impyla/) |
| Microsoft Access（通过 pyodbc） | [sqlalchemy-access](https://pypi.org/project/sqlalchemy-access/) |
| Microsoft SQL Server（通过 python-tds） | [sqlalchemy-tds](https://github.com/m32/sqlalchemy-tds) |
| Microsoft SQL Server（通过 turbodbc） | [sqlalchemy-turbodbc](https://pypi.org/project/sqlalchemy-turbodbc/) |
| MonetDB [[1]](#id5) | [sqlalchemy-monetdb](https://github.com/gijzelaerr/sqlalchemy-monetdb) |
| OpenGauss | [openGauss-sqlalchemy](https://gitee.com/opengauss/openGauss-sqlalchemy) |
| Rockset | [rockset-sqlalchemy](https://pypi.org/project/rockset-sqlalchemy) |
| SAP ASE（前 Sybase 方言的分支） | [sqlalchemy-sybase](https://pypi.org/project/sqlalchemy-sybase/) |
| SAP Hana [[1]](#id5) | [sqlalchemy-hana](https://github.com/SAP/sqlalchemy-hana) |
| SAP Sybase SQL Anywhere | [sqlalchemy-sqlany](https://github.com/sqlanywhere/sqlalchemy-sqlany) |
| Snowflake | [snowflake-sqlalchemy](https://github.com/snowflakedb/snowflake-sqlalchemy) |
| Teradata Vantage | [teradatasqlalchemy](https://pypi.org/project/teradatasqlalchemy/) |

| YugabyteDB | [sqlalchemy-yugabytedb](https://pypi.org/project/sqlalchemy-yugabytedb/) |  ## 包含的方言

+   PostgreSQL

+   MySQL 和 MariaDB

+   SQLite

+   Oracle

+   Microsoft SQL Server

### 包含的方言的支持级别

下表总结了每个包含的方言的支持级别。

**包含的方言的支持数据库版本**

| 数据库 | 在 CI 中完全测试过 | 普通支持 | 尽力而为 |
| --- | --- | --- | --- |
| **Microsoft SQL Server** | 2017 | 2012+ | 2005+ |
| **MySQL / MariaDB** | 5.6、5.7、8.0 / 10.8、10.9 | 5.6+ / 10+ | 5.0.2+ / 5.0.2+ |
| **Oracle** | 18c | 11+ | 9+ |
| **PostgreSQL** | 12、13、14、15 | 9.6+ | 9+ |
| **SQLite** | 3.36.0 | 3.12+ | 3.7.16+ |

### 支持定义

在 CI 中完全测试过

**在 CI 中完全测试过** 表示经过在 sqlalchemy CI 系统中测试的版本，并通过测试套件中的所有测试。  

普通支持

**普通支持** 表示大多数功能应该正常工作，但并非所有版本都在 ci 配置中进行测试，因此可能存在一些不受支持的边缘情况。我们将尝试修复影响这些版本的问题。

尽力而为

**尽力而为** 表示我们尽力在这些版本上支持基本功能，但在某些用例中可能会存在不支持的功能或错误。可能会接受带有相关问题的拉取请求，以继续支持旧版本，这些请求会根据具体情况进行审查。

### 已包括方言的支持级别

以下表总结了每个已包含方言的支持级别。

**已包括方言的支持数据库版本**

| 数据库 | 在 CI 中进行了全面测试 | 普通支持 | 尽力而为 |
| --- | --- | --- | --- |
| **Microsoft SQL Server** | 2017 | 2012+ | 2005+ |
| **MySQL / MariaDB** | 5.6, 5.7, 8.0 / 10.8, 10.9 | 5.6+ / 10+ | 5.0.2+ / 5.0.2+ |
| **Oracle** | 18c | 11+ | 9+ |
| **PostgreSQL** | 12、13、14、15 | 9.6+ | 9+ |
| **SQLite** | 3.36.0 | 3.12+ | 3.7.16+ |

### 支持定义

在 CI 中进行了全面测试

**在 CI 中进行了全面测试** 表示已经在 sqlalchemy CI 系统中测试并通过了测试套件中的所有测试的版本。

普通支持

**普通支持** 表示大多数功能应该可以正常工作，但并非所有版本都在 ci 配置中进行了测试，因此可能存在一些不受支持的边缘情况。我们将尝试修复影响这些版本的问题。

尽力而为

**尽力而为** 表明我们尽力在这些版本上支持基本功能，但在某些用例中可能会存在不支持的功能或错误。可能会接受带有相关问题的拉取请求，以继续支持旧版本，这些请求会根据具体情况进行审查。

## 外部方言

目前由 SQLAlchemy 维护的外部方言项目包括：

| 数据库 | 方言 |
| --- | --- |
| Actian Data Platform、Vector、Actian X、Ingres | [sqlalchemy-ingres](https://github.com/ActianCorp/sqlalchemy-ingres) |
| 亚马逊 Athena | [pyathena](https://github.com/laughingman7743/PyAthena/) |
| Amazon Redshift（通过 psycopg2） | [sqlalchemy-redshift](https://pypi.org/project/sqlalchemy-redshift) |
| Apache Drill | [sqlalchemy-drill](https://github.com/JohnOmernik/sqlalchemy-drill) |
| Apache Druid | [pydruid](https://github.com/druid-io/pydruid) |
| Apache Hive 和 Presto | [PyHive](https://github.com/dropbox/PyHive#sqlalchemy) |
| Apache Solr | [sqlalchemy-solr](https://github.com/aadel/sqlalchemy-solr) |
| CockroachDB | [sqlalchemy-cockroachdb](https://github.com/cockroachdb/sqlalchemy-cockroachdb) |
| CrateDB | [crate-python](https://github.com/crate/crate-python) |
| EXASolution | [sqlalchemy_exasol](https://github.com/blue-yonder/sqlalchemy_exasol) |
| Elasticsearch（只读） | [elasticsearch-dbapi](https://github.com/preset-io/elasticsearch-dbapi/) |
| Firebird | [sqlalchemy-firebird](https://github.com/pauldex/sqlalchemy-firebird) |
| Firebolt | [firebolt-sqlalchemy](https://pypi.org/project/firebolt-sqlalchemy/) |
| Google BigQuery | [pybigquery](https://github.com/mxmzdlv/pybigquery/) |
| Google Sheets | [gsheets](https://github.com/betodealmeida/gsheets-db-api) |
| IBM DB2 and Informix | [ibm-db-sa](https://pypi.org/project/ibm-db-sa/) |
| IBM Netezza Performance Server [[1]](#id5) | [nzalchemy](https://pypi.org/project/nzalchemy/) |
| Impala | [impyla](https://pypi.org/project/impyla/) |
| Microsoft Access (via pyodbc) | [sqlalchemy-access](https://pypi.org/project/sqlalchemy-access/) |
| Microsoft SQL Server (via python-tds) | [sqlalchemy-tds](https://github.com/m32/sqlalchemy-tds) |
| Microsoft SQL Server (via turbodbc) | [sqlalchemy-turbodbc](https://pypi.org/project/sqlalchemy-turbodbc/) |
| MonetDB [[1]](#id5) | [sqlalchemy-monetdb](https://github.com/gijzelaerr/sqlalchemy-monetdb) |
| OpenGauss | [openGauss-sqlalchemy](https://gitee.com/opengauss/openGauss-sqlalchemy) |
| Rockset | [rockset-sqlalchemy](https://pypi.org/project/rockset-sqlalchemy) |
| SAP ASE (fork of former Sybase dialect) | [sqlalchemy-sybase](https://pypi.org/project/sqlalchemy-sybase/) |
| SAP Hana [[1]](#id5) | [sqlalchemy-hana](https://github.com/SAP/sqlalchemy-hana) |
| SAP Sybase SQL Anywhere | [sqlalchemy-sqlany](https://github.com/sqlanywhere/sqlalchemy-sqlany) |
| Snowflake | [snowflake-sqlalchemy](https://github.com/snowflakedb/snowflake-sqlalchemy) |
| Teradata Vantage | [teradatasqlalchemy](https://pypi.org/project/teradatasqlalchemy/) |
| YugabyteDB | [sqlalchemy-yugabytedb](https://pypi.org/project/sqlalchemy-yugabytedb/) |
