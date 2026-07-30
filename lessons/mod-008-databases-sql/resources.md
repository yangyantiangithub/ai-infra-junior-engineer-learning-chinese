# Module 008 — 数据库与 SQL 资源

## 官方文档

- **PostgreSQL 文档** — [postgresql.org/docs](https://www.postgresql.org/docs/)。最受欢迎的关系型数据库；文档是最佳参考。
- **SQLite 文档** — [sqlite.org/docs.html](https://www.sqlite.org/docs.html)。本地开发+测试的标准选择。
- **MySQL 文档** — [dev.mysql.com/doc](https://dev.mysql.com/doc/)。在遗留系统中仍很常见。
- **MongoDB 文档** — [mongodb.com/docs](https://www.mongodb.com/docs/)。标准的文档存储。
- **Redis 文档** — [redis.io/docs](https://redis.io/docs/)。内存键值+数据结构存储。

## 书籍

- **《SQL Cookbook（第 2 版）》**—— Anthony Molinaro 和 Robert de Graaf。基于菜谱；便于查阅。
- **《The Art of PostgreSQL》**—— Dimitri Fontaine。现代 PostgreSQL 惯用法。
- **《Designing Data-Intensive Applications》**—— Martin Kleppmann。关于数据系统最好的一本书。必读。
- **《Database Internals》**—— Alex Petrov。深入探讨数据库工作原理。

## 交互式学习

- **SQLZoo** — [sqlzoo.net](https://sqlzoo.net/)。交互式 SQL 教程。
- **Mode SQL Tutorial** — [mode.com/sql-tutorial](https://mode.com/sql-tutorial/)。实用，面向分析。
- **PostgreSQL Tutorial** — [postgresqltutorial.com](https://www.postgresqltutorial.com/)。覆盖基础到高级的免费教程。

## ML 特定数据工具

- **DuckDB 文档** — [duckdb.org/docs](https://duckdb.org/docs/)。嵌入式分析数据库。越来越多用于 ML 中的快速本地数据探索。
- **ClickHouse 文档** — [clickhouse.com/docs](https://clickhouse.com/docs)。大规模分析工作负载的列式存储。
- **Snowflake 文档** — [docs.snowflake.com](https://docs.snowflake.com/)。最常用的云数据仓库。
- **BigQuery 文档** — [cloud.google.com/bigquery/docs](https://cloud.google.com/bigquery/docs)。GCP 的数据仓库。

## ORM 和数据库库（Python）

- **SQLAlchemy** — [docs.sqlalchemy.org](https://docs.sqlalchemy.org/)。标准的 Python ORM + Core。阅读这两个层。
- **psycopg** — [www.psycopg.org/psycopg3](https://www.psycopg.org/psycopg3/docs/)。PostgreSQL 驱动。v3 原生支持 async。
- **asyncpg** — [magicstack.github.io/asyncpg](https://magicstack.github.io/asyncpg/)。快速的异步 PostgreSQL 驱动。
- **alembic** — [alembic.sqlalchemy.org](https://alembic.sqlalchemy.org/)。SQLAlchemy 的迁移工具。
- **DBOS** — [dbos.dev](https://www.dbos.dev/)。较新，由 Postgres 支持的持久执行。

## SQL 风格 + 性能

- **Use The Index, Luke!** — [use-the-index-luke.com](https://use-the-index-luke.com/)。关于数据库索引和查询性能的最佳免费参考。
- **SQL Style Guide** — [sqlstyle.guide](https://www.sqlstyle.guide/)。一致的约定。
- **EXPLAIN ANALYZE 演练** — 练习阅读 PostgreSQL 中的查询计划。

## Vector Database（ML 特定）

- **pgvector** — [github.com/pgvector/pgvector](https://github.com/pgvector/pgvector)。用于向量搜索的 Postgres 扩展。通常是正确的选择。
- **Chroma** — [docs.trychroma.com](https://docs.trychroma.com/)。轻量级 vector DB。
- **Pinecone** — [docs.pinecone.io](https://docs.pinecone.io/)。托管式 vector DB。
- **Weaviate / Qdrant / Milvus** — 替代的向量存储。

## 常见错误

- N+1 查询（在循环中加载相关行）。
- 存储 JSON 数据块然后在其内部查询（应使用适当的关系，或使用带索引的 JSONB）。
- 外键上没有索引。
- 在 SQL 字符串中信任用户输入（SQL 注入）。
- 全程使用 ORM 而不查看生成的 SQL。

## 本课程中的交叉引用

- 模块 001（Python）用于数据库客户端代码。
- 工程师方向的 `mod-105-data-pipelines` 用于生产数据管道。
- ML 平台方向用于特征存储（一种专门的数据库模式）。
