# Database Notes: Concurrency and Locking

> Part of the [Database Notes](index.md) series.

---

## 5. Concurrency and Locking

### Why locks exist

MVCC handles read/write conflicts elegantly — readers see a consistent snapshot without blocking writers. But write/write conflicts still need coordination. If two transactions both try to update the same row, one must wait for the other to finish. Locks are the mechanism that enforces this ordering.

### Lock types

A **shared lock** (read lock) can be held by many transactions simultaneously. An **exclusive lock** (write lock) can only be held by one transaction and blocks everyone else. With MVCC, ordinary SELECT statements in Postgres take no row locks at all — they just read from their snapshot. Row locks only appear when you explicitly ask for them (FOR UPDATE, FOR SHARE) or when you write (UPDATE, DELETE take exclusive row locks automatically until commit).

### Pessimistic vs optimistic locking

**Pessimistic locking** (SELECT FOR UPDATE) locks the row at read time so no one else can modify it until your transaction commits. It guarantees no conflicts but holds locks for the duration of your transaction. It's the right choice for high-contention short operations like inventory decrement or seat reservation. It's wrong for anything involving user think-time — you can't hold a database lock for the 3 minutes a user spends filling out a form.

**Optimistic locking** uses a version column. You read the row and note the version, do your work, then update with a condition that checks the version is still the same. If someone else changed it, the update affects zero rows and you retry. No locks held, works across HTTP sessions, great for low-contention edits. Under high contention it wastes a lot of work because many transactions keep failing and retrying.

When the logic fits in a single SQL statement, neither approach is needed — an atomic update like "SET stock = stock - 1 WHERE id = 1 AND stock > 0" does the read-check-write atomically in one step.

### Deadlocks

A deadlock is a cycle: T1 holds a lock that T2 wants, while T2 holds a lock that T1 wants. Neither can proceed. The database detects the cycle and aborts one of the transactions as the victim. The application must catch the deadlock error and retry. The main prevention strategy is to always acquire locks in a consistent global order — if every transaction locks account A before account B, the cycle can never form.

### SKIP LOCKED for job queues

SELECT FOR UPDATE SKIP LOCKED skips any row that another transaction has locked rather than waiting for it. Multiple workers can each grab a batch of pending jobs with no blocking and no double-processing. If a worker crashes, its locks are released and the jobs become available again.

### Idempotency

An idempotent operation produces the same result whether run once or many times. Distributed systems, retries, and at-least-once message delivery all require idempotent operations. Use idempotency keys (store the result of the first execution and return it on replay), unique constraints (reject duplicate inserts at the DB level), upserts, and absolute writes (SET status = 'active') instead of relative ones (SET count = count + 1).

---

### Race conditions

A race condition happens when the correctness of a result depends on the timing of concurrent operations. In databases, most race conditions are write/write or read-then-write conflicts.

**Lost update** — two transactions read the same value, both compute a new value based on it, and both write back. The second write overwrites the first without incorporating its change.

```
T1: read stock = 10 → compute 10 - 1 = 9 → write stock = 9
T2: read stock = 10 → compute 10 - 1 = 9 → write stock = 9  ← T1's decrement is lost
result: stock = 9 (should be 8)
```

Fix: use `SELECT FOR UPDATE` (pessimistic), a version-checked update (optimistic), or an atomic SQL update: `UPDATE items SET stock = stock - 1 WHERE id = 1 AND stock > 0`.

---

**Dirty read** — one transaction reads a value that another transaction has written but not yet committed. If the writer rolls back, the reader acted on data that never officially existed.

Fix: set isolation level to at least READ COMMITTED (Postgres default). Never use READ UNCOMMITTED.

---

**Non-repeatable read** — within one transaction, the same row is read twice but returns different values because another transaction committed a change in between.

Fix: REPEATABLE READ isolation — Postgres snapshots the database at the first read, so re-reads return the same values.

---

**Phantom read** — a transaction re-runs a range query (WHERE amount > 100) and gets back different rows because another transaction inserted a matching row in between.

Fix: SERIALIZABLE isolation — the database ensures the execution is equivalent to some serial order. Performance cost is higher but it eliminates all anomalies.

---

**Double charge / duplicate operation** — a user clicks "pay" twice, a retry fires before the first request completes, or a message queue delivers an event more than once. The same operation executes multiple times.

Fix: idempotency keys — record the first result keyed to a unique request ID. On repeat, return the stored result instead of executing again.

---

**Summary: isolation level vs anomaly prevented**

| Isolation level | Dirty read | Non-repeatable read | Phantom read |
|---|---|---|---|
| READ UNCOMMITTED | possible | possible | possible |
| READ COMMITTED | prevented | possible | possible |
| REPEATABLE READ | prevented | prevented | possible |
| SERIALIZABLE | prevented | prevented | prevented |

Postgres default is READ COMMITTED. Bump to SERIALIZABLE only when you need full correctness guarantees and can tolerate retries on serialization failures.

---

### Gotchas

- Locking rows in inconsistent order across different code paths creates deadlocks — always lock in a fixed order (e.g., always by ascending ID).
- FK checks lock the parent row during child inserts — concurrent inserts that reference the same parent row will contend unexpectedly.
- Never run a migration without lock_timeout — a blocked ALTER can queue all traffic behind it and silently halt production.
- SKIP LOCKED gives an intentionally inconsistent view of the table — it is only appropriate for queue-style work, never for regular queries.
- Optimistic locking under high contention produces cascades of failures and retries — switch to pessimistic locking when multiple users routinely edit the same records.
