# Database Notes: Database Objects and Features

> Part of the [Database Notes](index.md) series.

---

## 9. Database Objects and Features

### Views

A view is a named, stored query. It holds no data — every time you query it, the database runs the underlying SQL. Views are always current because they're just stored queries, not stored results. They're useful for three things: abstraction (encapsulate complex joins once so consumers see a simple name), security (grant access to a view that exposes only certain columns or rows without granting table access), and stable interfaces (the underlying tables can change while the view keeps the same shape for consumers).

### Materialized views

A materialized view runs the underlying query once and stores the results physically, like a table. Reads are fast because the work is precomputed, but the data goes stale until you refresh it. You can add indexes to a materialized view, which a regular view doesn't support. Use them for expensive aggregations that power dashboards or reports where slight staleness is acceptable. Refresh can be full (simpler), concurrent (slower but doesn't block reads — requires a unique index), or on a schedule.

### Stored procedures

Stored procedures are named programs stored and executed inside the database. They reduce network round trips (many operations in one call), keep logic close to the data, and let you grant execute access without granting table access. The downsides are real: they're harder to test, version control, and debug than application code; they use vendor-specific syntax that doesn't port; and they put compute on the database tier, which is the hardest tier to scale horizontally.

### Triggers

Triggers run a function automatically when a row is inserted, updated, or deleted. Common uses are audit logging (record who changed what), maintaining denormalized data (keeping a comment_count column in sync), enforcing complex business rules that CHECK constraints can't express, and auto-setting timestamps. The danger is hidden behavior — a developer runs a simple UPDATE and doesn't realize it also modifies three other tables. Row-level triggers on bulk operations run once per row, so updating a million rows runs the trigger function a million times inside the same transaction.

### CTEs (WITH clause)

A Common Table Expression defines a named temporary result set that exists only for the duration of one query. They're primarily a readability tool — breaking a complex query into named, sequential steps that read top-to-bottom instead of nesting subqueries. Recursive CTEs reference themselves and are the standard way to query hierarchical data: org charts, category trees, and threaded comments. The recursive part runs repeatedly until an iteration produces no new rows.

In Postgres 12 and later, non-recursive CTEs referenced once are inlined into the main query so the optimizer can push filters into them. Before Postgres 12, CTEs were always materialized and acted as an optimization fence.

### Window functions

Window functions compute a value for each row using a set of related rows — without collapsing the rows the way GROUP BY does. You keep full row-level detail and get aggregate or ranking context alongside it. Key functions: ROW_NUMBER (unique sequential rank), RANK (gaps after ties), DENSE_RANK (no gaps), LAG and LEAD (value from the previous or next row), and standard aggregates like SUM and AVG used over a window. The classic interview pattern is top N per group: apply ROW_NUMBER partitioned by the group column, then filter in an outer query where the row number is at most N.

### Full-text search

Full-text search goes beyond LIKE by tokenizing text, removing stop words, stemming words to their roots (running/ran/runs → run), and building an inverted index that maps each term to the documents containing it. This supports relevance ranking and language-aware matching. Postgres tsvector and tsquery with a GIN index works well for many applications without extra infrastructure. Move to a dedicated engine like Elasticsearch or Meilisearch when you need advanced relevance tuning, typo tolerance, faceted navigation, or search at very high scale.

### JSON columns

Postgres jsonb stores JSON in a parsed binary format — faster to query than plain json and indexable with GIN. JSON columns are good for variable-shape data like product attributes, user preferences, and raw API payloads. They're bad for fields you filter, join, sort, or aggregate on regularly — the planner has poor statistics for values inside JSON, so frequent queries on JSON fields should be promoted to real typed columns.

---

### Gotchas

- Postgres REFRESH MATERIALIZED VIEW takes an exclusive lock by default, blocking all reads for the duration — use REFRESH CONCURRENTLY on live tables (requires a unique index on the view).
- A trigger that updates a single counter row on every insert forces all concurrent inserts to serialize on that one row — a common unintentional hotspot.
- In Postgres before version 12, CTEs were always materialized and prevented the planner from pushing predicates in — upgrade or explicitly mark CTEs as NOT MATERIALIZED.
- Window functions execute after the WHERE clause, so you can't filter on a window function result in the same query level — wrap in a subquery or CTE.
- Frequently querying a jsonb field without an index forces a full-table scan that extracts and checks the JSON on every row — add a GIN index or extract the field into a generated column.
