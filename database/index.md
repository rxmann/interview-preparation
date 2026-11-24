# Database Notes: Index

> Learning-oriented notes. Each topic explains **what** the concept is, **why** it exists (the problem it solves), **how** it works underneath, and **what tradeoffs** come with it. Examples are mostly PostgreSQL, with MySQL/InnoDB, DynamoDB, Redis, and others where relevant.

---

## Table of Contents

| # | Topic | File |
|---|-------|------|
| 1 | [Fundamentals](01-fundamentals.md) — relational vs NoSQL, normalization, keys, constraints, ER modeling, OLTP vs OLAP, row vs column stores |
| 2 | [Indexing and Internals](02-indexing.md) — pages, B+trees, LSM trees, index types, selectivity, when indexes hurt |
| 3 | [Query Optimization](03-query-optimization.md) — EXPLAIN, scan types, join algorithms, N+1, pagination, anti-patterns |
| 4 | [Transactions and ACID](04-transactions.md) — ACID deep dive, isolation levels, MVCC, WAL, savepoints, 2PC |
| 5 | [Concurrency and Locking](05-concurrency.md) — shared/exclusive locks, row vs table locks, optimistic vs pessimistic, deadlocks, FOR UPDATE, idempotency |
| 6 | [Scaling and Distribution](06-scaling.md) — vertical vs horizontal, replication types, replication lag, sharding, consistent hashing, failover |
| 7 | [Distributed Data Theory](07-distributed-theory.md) — CAP, PACELC, consistency models, consensus, Sagas, Outbox pattern, CDC |
| 8 | [Caching](08-caching.md) — why cache, cache-aside/read-through/write-through/write-back, invalidation, TTL, eviction, Redis vs Memcached, Redis data structures |
| 9 | [Database Objects and Features](09-database-objects.md) — views, materialized views, stored procedures, triggers, CTEs, window functions, full-text search, JSON columns |
| 10 | [Operations and Security](10-operations-security.md) — connection pooling, network security, monitoring, backups/PITR, zero-downtime migrations, SQL injection, encryption, least privilege |
| 11 | [NoSQL Types and When to Use Them](11-nosql.md) — key-value, document, wide-column, graph, time-series, DynamoDB in depth |
| 12 | [Additional Postgres Topics](12-postgres.md) — index types (GIN, GiST, BRIN), VACUUM, autovacuum, bloat, pg_stat_statements, table partitioning, soft delete, RLS |
| 13 | [Revision Summary](13-revision-summary.md) — condensed recap of all sections |

---

> **Note:** The original single-file version is preserved at [database.md](database.md).
