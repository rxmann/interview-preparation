# Database Notes: Postgres-Specific Topics

> Part of the [Database Notes](index.md) series.

---

## 12. Postgres-Specific Topics

### Index types beyond B-tree

The default B-tree covers equality, ranges, and sorting. Postgres has three other important index types for specialized data.

**GIN** (Generalized Inverted Index) maps each element inside a value to the rows containing it — like an inverted index in a search engine. Use it for JSONB containment queries, array overlap and containment, full-text search, and trigram-based LIKE/ILIKE with the pg_trgm extension. GIN indexes are fast to query but slow to build and expensive to update — not suitable for write-heavy columns.

**GiST** (Generalized Search Tree) stores a summary (like a bounding box) at each internal node, allowing searches to skip subtrees that can't match. Use it for geometric and geographic data (PostGIS), range type overlap and containment, nearest-neighbor searches, and exclusion constraints (preventing overlapping bookings). GiST is more flexible than GIN but slower for full-text and trigram use cases.

**BRIN** (Block Range Index) stores only the minimum and maximum value per range of physical pages. It's tiny (kilobytes vs gigabytes for a B-tree on the same data) and almost free to maintain. It only works when column values correlate with physical row order — append-only timestamps and sequential IDs are the ideal case. On randomly ordered data it's useless. Best for huge time-series or log tables where you query by time range.

### VACUUM and dead tuples

Postgres MVCC never overwrites rows in place. Every UPDATE writes a new row version and marks the old one dead. Every DELETE marks the row dead. Dead tuples stay in the table, consuming space and slowing scans, until VACUUM removes them. VACUUM also updates the visibility map (which enables index-only scans) and freezes old row versions to prevent transaction ID wraparound.

Plain VACUUM runs concurrently — it doesn't block reads or writes. VACUUM FULL rewrites the entire table into a new compact file and returns space to the OS, but it takes an exclusive lock for its full duration. Never run VACUUM FULL on a live production table; use pg_repack instead, which does the same thing with only brief locks.

### Autovacuum

Autovacuum runs VACUUM and ANALYZE automatically in the background when enough rows change. The default threshold is 20% of the table's row count — meaning a 500-million-row table waits for 100 million dead tuples before vacuuming. That's too late. Tune scale factors per table for high-churn tables. Never disable autovacuum; if it's causing visible load, tune it to run more often in smaller batches, not less often.

### Transaction ID wraparound

Postgres transaction IDs are 32-bit integers. If old row versions aren't frozen by VACUUM, eventually a transaction ID wraps around and those rows appear to be "in the future" — they become invisible, which is effectively data loss. Autovacuum prevents this by freezing old row versions before they reach the danger zone. If VACUUM falls too far behind, Postgres will refuse new write transactions entirely to protect data, requiring manual emergency intervention. Monitor the age of relfrozenxid per table.

### Table bloat

Bloat is space in tables and indexes occupied by dead tuples or empty pages beyond what live data needs. The main cause is VACUUM not running often enough. The hidden cause is long-running transactions and idle-in-transaction sessions — VACUUM cannot remove dead tuples still visible to any open snapshot. One forgotten transaction can cause the whole database to bloat. Abandoned replication slots have the same effect. Set idle_in_transaction_session_timeout to automatically kill stalled sessions.

### pg_stat_statements

This built-in extension tracks execution statistics per normalized query (literals replaced with placeholders). It shows total time, mean time, call count, cache hits, and disk reads per query. The key insight is to optimize by total time — a 5ms query called 2 million times a day consumes far more DB resources than a 2-second query run twice. Reset statistics after deploys to get a clean baseline for the new code.

### Table partitioning

Declarative partitioning splits one logical table into physical child tables on the same server. Range partitioning by time is the most common use case. Benefits: partition pruning lets queries skip irrelevant child tables entirely; dropping old data is an instant DROP TABLE instead of a slow DELETE across billions of rows; indexes on recent partitions fit in memory. Unique constraints must include the partition key because uniqueness is enforced per partition, not globally. Too many partitions (thousands) hurts planning time. Use pg_partman to automate partition creation on a schedule.

### Soft delete

Soft delete marks rows as deleted with a flag or timestamp instead of removing them. It enables undo, preserves history for audit, and maintains referential integrity (old orders can still reference a deleted product). The risks: every query must add a filter for deleted rows — forgetting it in one place causes data leaks or wrong counts; unique constraints break because a deleted user's email still occupies the index; tables grow without bound; and GDPR right-to-erasure requires actual deletion of personal data. Use partial unique indexes (unique where deleted_at IS NULL) and a background purge job that hard-deletes rows past the retention window.

### Row-Level Security (RLS)

RLS attaches policies to a table that the database automatically applies to every query on it. In a multi-tenant application, this enforces tenant isolation at the database level — a buggy endpoint that forgets the tenant filter still can't leak data because the database adds the filter itself. The app sets the current tenant via a session variable, and the policy checks it on every row access. By default, superusers and table owners bypass RLS. If the application role owns the table, you must explicitly enable FORCE ROW LEVEL SECURITY. In PgBouncer transaction mode, always use SET LOCAL so the tenant variable resets at transaction end and doesn't leak to the next connection.

---

### Gotchas

- GIN indexes are large and expensive to update — use them on columns that are queried heavily but written infrequently.
- VACUUM FULL holds an exclusive lock for its full duration — use pg_repack for live tables to avoid an outage.
- A single idle-in-transaction session can hold back VACUUM for the entire database; bloat accumulates until it's killed.
- Table partitioning is not sharding — all partitions live on one server and share its resources; use Citus or application-level sharding for multi-node scaling.
- RLS policies are silently bypassed by superuser and table owner connections — always test with the actual application role.
