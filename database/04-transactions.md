# Database Notes: Transactions and ACID

> Part of the [Database Notes](index.md) series.

---

## 4. Transactions and ACID

### What a transaction is

A transaction groups multiple operations into a single unit that either fully completes or has no effect at all. Without transactions, a crash mid-operation (like a money transfer that debits one account but never credits the other) leaves the database in a broken state. The four properties that define correct transaction behavior are summarized as ACID.

### ACID

**Atomicity** means all or nothing. If any step in a transaction fails — a constraint violation, a crash, an explicit rollback — every change made in that transaction is undone. The database never ends up with half a transaction applied.

**Consistency** means the database moves from one valid state to another. All declared rules — constraints, foreign keys, check conditions — must hold after every commit. The database enforces the rules you declared; invariants it doesn't know about (like "total money must stay constant") are your application's responsibility.

**Isolation** means concurrent transactions don't interfere with each other. Each transaction behaves as if it's the only one running. In practice, full isolation is expensive, so databases offer tunable levels that trade correctness for throughput.

**Durability** means a committed transaction survives crashes. The mechanism is the write-ahead log (WAL): changes are written to a sequential log and flushed to disk before the commit is confirmed. This lets the database recover by replaying the log after a crash. WAL also enables streaming replication and point-in-time recovery.

### Isolation anomalies

Without proper isolation, concurrent transactions can produce wrong results in several ways. A **dirty read** is reading another transaction's uncommitted changes — if that transaction rolls back, you acted on data that never existed. A **non-repeatable read** is when you read the same row twice in one transaction and get different values because another transaction committed an update in between. A **phantom read** is when re-running a range query returns different rows because another transaction inserted or deleted matching rows. A **lost update** happens when two transactions both read a value, modify it, and write it back — one overwrites the other's change without knowing. **Write skew** is subtler: two transactions read the same data, make independent decisions, and write to different rows — each change is locally valid but together they violate an invariant (e.g., both on-call doctors decide it's safe to sign off, leaving nobody on call).

### Isolation levels

**Read Uncommitted** allows all anomalies. **Read Committed** prevents dirty reads and is the Postgres default — each statement sees only committed data. **Repeatable Read** prevents dirty reads and non-repeatable reads; in Postgres this is snapshot isolation, which also prevents phantoms. **Serializable** prevents everything including write skew; Postgres uses Serializable Snapshot Isolation (SSI), which detects dangerous dependency cycles and aborts one transaction rather than blocking.

### MVCC

Multi-Version Concurrency Control keeps multiple versions of each row so readers never block writers and writers never block readers. Only write-write conflicts need coordination. Postgres implements this by keeping old row versions in the table itself — they accumulate as dead tuples and must be cleaned up by VACUUM.

### Savepoints and 2PC

Savepoints let you mark a point inside a transaction you can roll back to without aborting the whole thing — useful when one step might fail expectedly and you want to continue. Two-Phase Commit (2PC) coordinates an atomic commit across multiple separate databases: in Phase 1 all participants vote YES or NO, in Phase 2 the coordinator sends commit or rollback to everyone. It's the only way to get true atomicity across systems, but it blocks indefinitely if the coordinator crashes after votes are cast.

---

### Gotchas

- Postgres Read Uncommitted behaves exactly like Read Committed — Postgres never shows dirty data regardless of level.
- Postgres Repeatable Read uses snapshot isolation, which prevents phantoms (stronger than the SQL standard requires) but does not prevent write skew.
- Serializable in Postgres requires application-level retry logic — aborted transactions must be detected and retried.
- 2PC participants hold locks until the coordinator recovers — a crashed coordinator can block tables for a long time.
- Setting synchronous_commit=off makes commits faster but risks losing the last few committed transactions on a hard crash — no corruption, just silent data loss.
