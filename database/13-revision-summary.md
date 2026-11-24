# Database Notes: Revision Summary

> Part of the [Database Notes](index.md) series.

---

## 13. Revision Summary

A one-liner per concept for quick review. No explanations — just the rules and facts.

---

### Fundamentals

- Relational databases give schema enforcement, joins, and ACID; choose them when relationships matter or access patterns are unknown.
- Normalization removes redundancy so each fact lives in exactly one place, eliminating update, insert, and delete anomalies.
- 1NF: atomic values, one type per column, primary key on every table, no repeating groups.
- 2NF: every non-key column depends on the whole composite key, not part of it.
- 3NF: no non-key column depends on another non-key column (no transitive dependencies).
- BCNF: every determinant must be a candidate key — stricter than 3NF, same goal.
- Denormalization is intentional redundancy for read speed — always pair it with a sync strategy.
- Use surrogate PKs plus a UNIQUE constraint on the natural key; index all foreign key columns.
- UUIDv7 gives distributed generation with sequential index-friendliness — better than random UUIDv4.
- OLTP: many small row-level transactions; OLAP: large scans and aggregations — keep them on separate systems.
- Row stores: fast whole-record reads (OLTP); column stores: fast column-only scans with compression (OLAP).

---

### Indexing

- An index is a separate sorted structure mapping column values to row locations — trades write overhead for fast reads.
- A B+tree with ~400 keys per node reaches any row in 3–4 steps regardless of table size.
- Composite indexes follow the leftmost prefix rule — equality columns first, range/sort column last.
- A covering index (with INCLUDE) lets the database answer entirely from the index without touching the table.
- Partial indexes cover only rows matching a condition — much smaller and faster for selective subsets.
- Expression indexes are needed when queries use functions on columns (LOWER, DATE, etc.).
- Low-cardinality columns (status, boolean) rarely benefit from an index — the planner seq-scans them anyway.
- Unused indexes still slow every write — find and remove them.

---

### Query Optimization

- The query planner is cost-based and uses column statistics; bad estimates cause bad plans.
- EXPLAIN ANALYZE shows real vs estimated row counts — a large gap is the number-one cause of bad plans.
- Sequential scan reads every page; index scan fetches rows via the index; index-only scan never touches the table; bitmap scan batches index lookups for medium selectivity.
- Nested loop is good for small outer + indexed inner; hash join for large unsorted inputs (equality only); merge join for pre-sorted inputs.
- N+1: fetching a list then looping with one query per item — fix with JOIN or a single batched IN query.
- OFFSET degrades linearly — use keyset pagination (cursor on the last seen value) for deep pages.
- Functions on columns (LOWER, DATE) prevent index use — rewrite the predicate or add an expression index.
- NOT IN with a nullable subquery silently returns zero rows — use NOT EXISTS.

---

### Transactions

- ACID: Atomicity (all or nothing), Consistency (valid state to valid state), Isolation (concurrent transactions don't interfere), Durability (committed data survives crashes via WAL).
- Dirty read: uncommitted data visible. Non-repeatable read: same row returns different values. Phantom read: re-running a query returns different rows. Lost update: two transactions overwrite each other. Write skew: two transactions read shared data, write different rows, violate an invariant together.
- Read Committed (Postgres default) prevents dirty reads. Repeatable Read (snapshot isolation) also prevents phantoms. Serializable (SSI) prevents write skew — requires application retry on abort.
- MVCC keeps multiple row versions so readers never block writers — Postgres pays with dead tuples requiring VACUUM.
- WAL: changes written to a sequential log before data pages; flush at commit = durability; also enables replication and PITR.
- 2PC: atomic commit across multiple systems — blocks if coordinator crashes after votes are cast.
- Postgres Read Uncommitted behaves like Read Committed; Postgres Repeatable Read does not prevent write skew.

---

### Concurrency

- Ordinary SELECTs in Postgres take no row locks — they read from their MVCC snapshot.
- Exclusive row locks are taken automatically by UPDATE and DELETE and held until commit.
- ACCESS EXCLUSIVE table lock (DDL) blocks even SELECTs — always set lock_timeout on migration sessions.
- Pessimistic locking (FOR UPDATE): good for high-contention short operations; holds locks until commit.
- Optimistic locking (version column): good for low-contention long sessions; retries on version conflict.
- An atomic single-statement update (SET col = col - 1 WHERE col > 0) beats both when logic fits in SQL.
- Deadlock: a lock cycle — the DB kills one victim; always retry; always acquire locks in a consistent order.
- SKIP LOCKED: workers each grab unlocked rows with no blocking — the right way to build a job queue.
- Idempotency: idempotency keys, unique constraints, upserts, and absolute writes make retries safe.

---

### Scaling

- Fix queries and indexes first; connection pool; scale vertically; add cache; add replicas; shard last.
- Leader-follower: one writer, replicas read-only — simplest and most common.
- Multi-leader: multiple writers (one per region) — fast local writes, write conflicts need resolution.
- Leaderless (W+R>N quorum): no single point of failure, eventual consistency, needs conflict handling.
- Async replication: fast, but lag can cause read-your-writes violations and data loss on failover.
- Sync replication: zero data loss, but every commit waits for the network round trip.
- Good shard key: high cardinality, even distribution, aligned with main query pattern.
- Consistent hashing: adding/removing a node moves only 1/N of keys, not everything.
- Split brain: prevent with consensus for leader election and fencing tokens on storage.

---

### Distributed Theory

- CAP: during a network partition, choose Consistency or Availability — Partition tolerance is mandatory.
- PACELC extends CAP: even without a partition, strong consistency costs latency.
- Linearizable: every read returns the latest write, real-time order — required for locks and balances.
- Causal consistency: cause before effect, concurrent ops can differ — strongest model still available during partitions.
- Eventual consistency: replicas converge eventually — highest availability, needs conflict resolution.
- Consensus (Raft/Paxos): majority quorum, 2f+1 nodes tolerate f failures — why clusters use odd counts.
- 2PC: atomic across systems but blocks if coordinator crashes; Sagas: eventual with compensations.
- Outbox pattern: write event in same DB transaction as business change, relay publishes it — at-least-once.
- CDC: reads changes from the transaction log — captures everything including deletes, in commit order.
- Saga compensations are new business actions, not rollbacks — the original transaction really committed.

---

### Caching

- Cache-aside: app checks cache, misses go to DB, writes delete the key — most common pattern.
- Write-through: writes go cache then DB synchronously — cache always current but slower writes.
- Write-back: writes go to cache only, flush async — fastest writes, risk of data loss on cache failure.
- Cache stampede: hot key expires, all requests miss at once — fix with single-flight lock or stale-while-revalidate.
- Cache penetration: requests for missing data bypass cache — fix with null caching or bloom filter.
- Cache avalanche: many keys expire together — fix with TTL jitter and HA cache.
- Delete the cache key on write, never update it in place — racing updates leave stale values.
- TTL is a safety net, not a correctness guarantee — combine with explicit invalidation.
- LRU evicts least recently used (general default); LFU evicts least frequently used (stable popularity patterns).
- Redis is the default: rich data types, persistence, replication; Memcached is a simple pure string cache.

---

### Database Objects

- Views: stored queries, no data, always current — use for abstraction, security, and stable interfaces.
- Materialized views: stored results, stale until refreshed — use for expensive aggregations with tolerable staleness.
- REFRESH CONCURRENTLY avoids blocking reads but requires a unique index on the materialized view.
- Stored procedures: fewer round trips, logic near data — but hard to test, vendor-locked, DB CPU is hard to scale.
- Triggers: automatic side effects on write — powerful but hidden; row-level triggers on bulk ops run once per row.
- CTEs: named query steps for readability; recursive CTEs traverse hierarchies; inline by default in Postgres 12+.
- Window functions: ROW_NUMBER, RANK, LAG, running SUM — compute over a window without collapsing rows.
- Top-N per group: ROW_NUMBER() OVER (PARTITION BY col ORDER BY ...) then filter rn <= N in a subquery.
- Full-text search: Postgres tsvector+tsquery+GIN works for many apps; use Elasticsearch for scale and relevance tuning.
- jsonb: use for variable-shape data; promote frequently filtered fields to real columns with proper indexes.

---

### Operations and Security

- Connection pools cap DB connections and queue requests — throughput peaks at ~2× CPU cores, not at max_connections.
- PgBouncer transaction mode breaks SET, temp tables, and LISTEN — always use SET LOCAL.
- Databases belong in private subnets; security groups restrict access to the app tier's SG only.
- Monitor total query time (calls × mean), lock waits, replication lag, connections, cache hit ratio, vacuum health, and disk space.
- Backups need regular restore tests — a backup never tested is a hope.
- Expand-and-contract for zero-downtime migrations; CREATE INDEX CONCURRENTLY; set lock_timeout.
- Parameterized queries are the only real defense against SQL injection — ORMs help but raw SQL escape hatches are still vulnerable.
- Encryption at rest: protects hardware theft; in transit (verify-full TLS): protects the wire; application-level: hides data from DBAs.
- App role: SELECT/INSERT/UPDATE/DELETE only — never superuser; store credentials in a secrets manager.

---

### NoSQL

- NoSQL is query-first: design data structures around access patterns, not entities.
- Key-value: get/put/delete by key — fastest, simplest, no ad hoc queries.
- Document: JSON records with indexable nested fields — flexible schema, weak joins.
- Wide-column (Cassandra): partition key + clustering columns — massive write throughput, rigid query patterns.
- Graph (Neo4j): nodes + edges as first-class data — O(1) relationship traversal, weak bulk analytics.
- Time-series: append-mostly timestamped data — 10× compression, automatic downsampling, retention policies.
- DynamoDB partition key: determines node; sort key: range queries within partition; GSI: alternate access, eventual, anytime; LSI: alternate sort, strong reads, creation-time only, 10 GB limit.
- Single-table design: co-locate related entity types in one table for single-Query fetches.
- DynamoDB Scan reads everything — always design for GetItem or Query.

---

### Postgres

- GIN: inverted index for JSONB, arrays, full-text, trigrams — fast queries, expensive updates.
- GiST: summary tree for spatial data, ranges, nearest-neighbor, exclusion constraints.
- BRIN: min/max per block range — tiny and cheap, only works when values correlate with physical order.
- VACUUM: removes dead tuples, updates visibility map, freezes XIDs — runs concurrently, never blocks.
- VACUUM FULL: compacts and returns space to OS — takes exclusive lock, use pg_repack in production.
- Autovacuum default scale factor (20%) is too coarse for large tables — tune per table.
- XID wraparound: 32-bit IDs; unfroozen old rows become invisible — monitor age(relfrozenxid), never fall behind.
- Bloat cause: long-running transactions and idle-in-transaction sessions hold back VACUUM.
- pg_stat_statements: optimize by total time (calls × mean), not slowest single execution.
- Table partitioning: range/list/hash on same server — partition pruning, instant DROP for retention, not sharding.
- Soft delete: needs partial unique indexes and a purge job for GDPR compliance.
- RLS: enforces row access in DB; bypassed by superuser and table owner — test with app role, use FORCE ROW LEVEL SECURITY.
