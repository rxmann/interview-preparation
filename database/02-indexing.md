# Database Notes: Indexing

> Part of the [Database Notes](index.md) series.

---

## 2. Indexing

### The problem without an index

A database table is physically stored as a collection of pages on disk (8 KB each in Postgres). There is no inherent order to these pages — rows are placed wherever space is free. When you query `WHERE email = 'asha@example.com'`, the database has no choice but to **read every single page** of the table and check every row. This is called a **sequential scan**.

For a small table, that's fine. For a million-row users table:

- 1,000,000 rows × ~100 bytes each ≈ 100 MB of data
- That's ~12,500 pages to read just to find one row
- On cold storage (SSD), each page read costs microseconds — that adds up fast

The same problem shows up with joins. If the `orders` table has no index on `customer_id`, then for every customer in a 10,000-row `customers` table, the database runs a full scan of `orders` — that's a **nested loop** doing 10,000 × (full scan of orders). Without indexes on join columns, joins on large tables become prohibitively slow.

---

### What an index is

An index is a **separate, sorted data structure** maintained alongside the table. Think of it like the index at the back of a textbook: instead of reading every page to find "B-tree," you look it up in the index and jump straight to page 142.

```sql
CREATE INDEX idx_users_email ON users (email);
```

This creates a sorted structure mapping email values → row locations. A query on `email` now:
1. Does a quick binary-search-style lookup in the index (~3 steps for 1 million rows)
2. Jumps directly to the matching row(s) in the table

**The tradeoff:** every write (INSERT, UPDATE, DELETE) must also update the index. Indexes are a deliberate trade of write speed and storage for read speed.

---

### How B+trees cut a million rows down to ~20 steps

The default index type in Postgres, MySQL, and most SQL databases is a **B+tree**. The key insight is that it's not a binary tree (2 children per node) — it's a **wide, shallow tree where each node holds hundreds of keys**.

A single B+tree node is sized to match one disk page (~8 KB). That fits roughly **400 keys per node**. This means:

| Tree depth | Rows it can index |
|------------|-------------------|
| 1 level    | ~400 rows         |
| 2 levels   | ~160,000 rows     |
| 3 levels   | ~64,000,000 rows  |
| 4 levels   | ~25,000,000,000 rows |

**A 3-level tree indexes 64 million rows in just 3 page reads.** And the top 1–2 levels are almost always cached in memory, so a typical lookup takes 1 real disk read + a few memory reads. That's how an index turns a full scan of 1 million rows into effectively 3–4 steps.

The B+tree stores all data at the **leaf level**, with leaves linked to each other. This makes range scans efficient: for `WHERE price BETWEEN 100 AND 500`, the database finds the first leaf for 100, then walks sideways through the linked leaves until 500. No backtracking up the tree needed.

---

### Types of indexes

#### Single-column index (most common)

```sql
CREATE INDEX idx_users_email ON users (email);
```

Useful for exact lookups, ranges, and sorting on that column. The right starting point for most cases.

#### Composite (multi-column) index

```sql
CREATE INDEX idx_orders_cust_status ON orders (customer_id, status, created_at);
```

Covers queries that filter on multiple columns together. The index is sorted by `customer_id` first, then `status` within that, then `created_at`. Think of a phone book sorted by last name then first name.

**The leftmost prefix rule:** the index is only usable if the query filters on a leading prefix of the columns.

| Query | Uses index? |
|-------|-------------|
| `WHERE customer_id = 5` | ✓ Yes |
| `WHERE customer_id = 5 AND status = 'paid'` | ✓ Yes |
| `WHERE customer_id = 5 AND status = 'paid' AND created_at > '2026-01-01'` | ✓ Yes, fully |
| `WHERE status = 'paid'` | ✗ No — skipping the first column |
| `WHERE customer_id = 5 AND created_at > '2026-01-01'` | ⚠ Partial — seeks on customer_id, filters created_at |

**Column order rule:** put equality columns first, range/sort columns last.

#### Covering index (index-only scan)

If the index contains all columns a query needs, the database never touches the table at all — the index alone answers the query.

```sql
CREATE INDEX idx_orders_cust ON orders (customer_id) INCLUDE (total, status);
SELECT total, status FROM orders WHERE customer_id = 5;  -- no table access needed
```

This is the fastest possible read path. The `INCLUDE` clause adds extra columns to the leaf level without making them part of the sort key.

#### Unique index

```sql
CREATE UNIQUE INDEX idx_users_email_unique ON users (email);
```

Enforces no duplicates. This is exactly how `UNIQUE` constraints are implemented under the hood. Use it both for constraint enforcement and fast equality lookups.

#### Partial index

Indexes only rows matching a condition, making the index smaller and faster.

```sql
CREATE INDEX idx_orders_pending ON orders (created_at) WHERE status = 'pending';
```

If only 1% of orders are pending, this index is 100× smaller than a full index on `created_at`. It's also the right answer when you have a low-cardinality column where only the rare values matter (e.g., `status = 'failed'` at 0.1%).

#### Expression (functional) index

Indexes the result of an expression rather than a raw column value.

```sql
CREATE INDEX idx_users_lower_email ON users (LOWER(email));
-- Now this query can use the index:
WHERE LOWER(email) = 'asha@example.com'
```

Without this, `WHERE LOWER(email) = ...` would force a full scan because a plain index on `email` stores the original case — the index can't be used when the column is wrapped in a function.

#### Hash index

Supports only equality lookups (`=`). Slightly smaller than a B+tree for exact-match workloads, but no ranges, no sorting. Rarely chosen in practice because B+trees handle equality nearly as well while also supporting ranges.

---

### When to use what

| Situation | Index type |
|-----------|-----------|
| Simple equality or range on one column | Single-column B+tree |
| Queries filtering on multiple columns together | Composite index |
| Query needs only a few columns from a large table | Covering index with `INCLUDE` |
| Column with very few distinct values, rare value matters | Partial index |
| Queries using `LOWER()`, `DATE()`, or other functions on a column | Expression index |
| Need to prevent duplicates | Unique index |
| Only exact equality, never ranges | Hash index (rarely needed) |

**Always index:**
- Primary keys (automatic)
- Foreign key columns (not automatic in Postgres — missing FK indexes cause slow deletes and joins)
- Columns used frequently in `WHERE`, `JOIN ON`, and `ORDER BY`

---

### When indexes hurt

Indexes are not free. The key costs:

**Write amplification.** An INSERT on a table with 5 indexes does 6 write operations. UPDATEs that touch indexed columns must rewrite those index entries too. High-write tables with many indexes slow down noticeably.

**Low-cardinality columns.** An index on a `status` column with values `active/inactive` where 90% of rows are `active` is useless for `WHERE status = 'active'` — the database will do a seq scan anyway because it's cheaper to read everything than to do millions of random page jumps. Use a partial index instead.

**Small tables.** A table with a few thousand rows fits in a handful of pages. A seq scan is faster than the overhead of using an index.

**Unused indexes still cost writes.** An index nobody queries still slows down every INSERT and UPDATE. Find them:

```sql
SELECT relname, indexrelname, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0;
```

**Redundant indexes.** An index on `(customer_id)` is redundant if `(customer_id, status)` already exists — the composite index covers single-column queries on `customer_id` via the leftmost prefix rule.

---

### Selectivity: when an index is worth it

**Selectivity** is the fraction of rows a condition matches. `WHERE id = 7` is highly selective (one row). `WHERE country = 'IN'` might match 60% of rows (low selectivity).

Indexes pay off when conditions are **highly selective**. If a query returns 30% of the table, it may actually be faster to do a seq scan: sequential reads are cheap and predictable, while an index scan does one random page read per row, which adds up quickly when there are many matches.

Rule of thumb: an index scan is typically faster when a condition matches fewer than ~5–10% of rows. The database's query planner estimates this using statistics and decides automatically.

---

> **Key takeaway:** An index turns a full-table scan into a 3–4 step lookup for most datasets. The tradeoff is write overhead and storage. Index the columns your most common queries filter, join, and sort on — and remove indexes that don't earn their keep.
