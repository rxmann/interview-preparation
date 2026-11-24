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

### Gotchas

- Locking rows in inconsistent order across different code paths creates deadlocks — always lock in a fixed order (e.g., always by ascending ID).
- FK checks lock the parent row during child inserts — concurrent inserts that reference the same parent row will contend unexpectedly.
- Never run a migration without lock_timeout — a blocked ALTER can queue all traffic behind it and silently halt production.
- SKIP LOCKED gives an intentionally inconsistent view of the table — it is only appropriate for queue-style work, never for regular queries.
- Optimistic locking under high contention produces cascades of failures and retries — switch to pessimistic locking when multiple users routinely edit the same records.
