# Database Notes: Query Optimization

> Part of the [Database Notes](index.md) series.

---

## 3. Query Optimization

### How a query executes

When you send SQL to the database, it goes through three stages: parse (check syntax and resolve names), plan (the query planner generates candidate execution strategies and picks the cheapest one based on statistics), and execute (run the chosen plan). The planner doesn't know the true cost — it estimates using column statistics stored from the last ANALYZE run. Most bad performance comes from bad estimates, not bad execution.

### EXPLAIN and reading a plan

EXPLAIN shows the plan the planner chose with estimated costs and row counts. EXPLAIN ANALYZE actually runs the query and shows real timings and real row counts alongside the estimates. The most important thing to look at is the gap between estimated rows and actual rows — a large divergence means the planner made wrong assumptions and likely picked the wrong strategy. Read plans bottom-up: the most indented nodes execute first and feed results up to their parent.

### Scan types

The database has four ways to read rows from a table. A **sequential scan** reads every page in the table — no index needed, cheap per-page, but slow when you only want a few rows from millions. An **index scan** traverses the index to find matching entries, then fetches each row from the table individually — good when very few rows match. An **index-only scan** answers entirely from the index without touching the table at all — the fastest read path when the index contains all needed columns. A **bitmap scan** sits between the two: it scans the index to build an in-memory bitmap of matching pages, then reads those pages in physical order — better than an index scan when medium numbers of rows match, because it avoids random I/O.

### Join algorithms

**Nested loop**: for each row on the outer side, look up matching rows on the inner side. Great when the outer side is small and the inner side has an index. Terrible when both sides are large. **Hash join**: build a hash table from the smaller side, then probe it with the larger side. Great for large unsorted inputs, but only works for equality conditions and needs memory for the hash table. **Merge join**: both inputs must be sorted on the join key, then walk through them together. Excellent when inputs arrive pre-sorted from an index, and handles large data well.

### N+1 problem

The N+1 problem is when code runs one query to get a list of N items, then runs one more query per item to fetch related data — 1 + N queries total. Each query has network and planning overhead, so 50 items means 51 round trips instead of 1. Fix it with a JOIN or by loading all related records in a single batched query using IN.

### Pagination

OFFSET/LIMIT seems simple but degrades linearly — to show page 500, the database must generate and discard the first 10,000 rows. Keyset pagination (cursor-based) avoids this by filtering on the last seen value instead of skipping rows, making every page equally fast.

### Statistics and SELECT *

The planner uses per-column statistics (value distributions, distinct counts, null fractions) to estimate how many rows a condition will match. These go stale after bulk loads. Correlated columns (city + state together) fool the planner because it assumes independence. Run ANALYZE after large data changes.

Avoid SELECT * — it fetches columns you don't need (extra I/O), prevents covering index use, and breaks code when columns are added or reordered.

---

### Gotchas

- Estimated vs actual row divergence is the number-one cause of bad plans — look for it in every EXPLAIN ANALYZE output.
- Wrapping a column in a function (LOWER, DATE, EXTRACT) prevents the index on that column from being used — create an expression index on the function result instead.
- LIKE '%term' with a leading wildcard cannot use a standard B-tree index — the index is sorted by the start of the string, not the middle.
- NOT IN with a subquery that can return NULL will silently return zero rows — use NOT EXISTS instead.
- OFFSET 10000 does not skip rows — the database must generate all 10,000 rows and throw them away; it gets slower the deeper you page.
