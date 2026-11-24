# Database Notes: Concepts, Internals, and Design

> Learning-oriented notes. Each topic explains **what** the concept is, **why** it exists (the problem it solves), **how** it works underneath, and **what tradeoffs** come with it. Examples are mostly PostgreSQL, with MySQL/InnoDB, DynamoDB, Redis, and others where relevant.

---

## Table of Contents

1. [Fundamentals](#1-fundamentals)
2. [Indexing and Internals](#2-indexing-and-internals)
3. [Query Optimization](#3-query-optimization)
4. [Transactions and ACID](#4-transactions-and-acid)
5. [Concurrency and Locking](#5-concurrency-and-locking)
6. [Scaling and Distribution](#6-scaling-and-distribution)
7. [Distributed Data Theory](#7-distributed-data-theory)
8. [Caching](#8-caching)
9. [Database Objects and Features](#9-database-objects-and-features)
10. [Operations and Security](#10-operations-and-security)
11. [NoSQL Types and When to Use Them](#11-nosql-types-and-when-to-use-them)
12. [Additional Postgres Topics](#12-additional-postgres-topics)
13. [Revision Summary](#13-revision-summary)

---

## 1. Fundamentals

### 1.1 Relational vs Non-Relational Databases

#### The relational model

A relational database (RDBMS) stores data in **tables** (relations). Each table has a fixed set of typed **columns**, and each **row** is one record. Tables relate to each other through **keys**: an `orders` row stores a `customer_id` that points to a row in `customers`. You query with **SQL**, a declarative language: you describe _what_ you want, and the database's query planner decides _how_ to get it.

The relational model's core strengths come from three ideas:

1. **Schema enforcement.** The database rejects data that doesn't fit (wrong type, missing required value, broken reference). Bad data is stopped at the door rather than discovered later.
2. **Joins.** Because data is split into normalized tables, you can combine it in any way at query time. You don't need to know every future question when you design the schema.
3. **Transactions.** Multi-row, multi-table changes can be made atomically with ACID guarantees (Section 4).

Examples: PostgreSQL, MySQL, MariaDB, SQL Server, Oracle, SQLite, and distributed SQL systems such as CockroachDB, Google Spanner, and Amazon Aurora.

#### The non-relational (NoSQL) family

"NoSQL" is an umbrella term for databases that drop some relational assumptions, usually to gain **horizontal scalability, flexible schemas, or a data model that fits a specific problem better**. It covers very different systems (Section 11):

- **Key-value:** a giant distributed hash map (Redis, DynamoDB).
- **Document:** JSON-like documents with nested structure (MongoDB).
- **Wide-column:** rows partitioned by key with many sorted columns (Cassandra).
- **Graph:** nodes and edges (Neo4j).
- **Time-series:** timestamped measurements (InfluxDB, TimescaleDB).

Most NoSQL systems historically gave up joins and multi-record transactions so that data could be split across many machines without coordination. Many have since added limited transactions (MongoDB multi-document transactions, DynamoDB `TransactWriteItems`), so the line has blurred.

#### Comparison

| Aspect        | Relational                                                   | Non-Relational                                                            |
| ------------- | ------------------------------------------------------------ | ------------------------------------------------------------------------- |
| Data model    | Tables with rows and columns                                 | Key-value, document, wide-column, graph                                   |
| Schema        | **Schema-on-write**: structure enforced when data is written | **Schema-on-read**: structure interpreted by the application when reading |
| Relationships | Foreign keys and joins                                       | Usually embedded or duplicated (denormalized)                             |
| Transactions  | Full ACID across any rows/tables                             | Often limited to one item, one partition, or small batches                |
| Querying      | Flexible ad hoc SQL                                          | Efficient only for access patterns you designed for                       |
| Scaling       | Vertical first; horizontal requires replicas/sharding        | Built for horizontal partitioning                                         |
| Consistency   | Strong by default                                            | Often tunable or eventual                                                 |

#### How to choose

Choose a **relational database** when:

- Data has many relationships (users, orders, products, payments).
- You need correctness guarantees spanning multiple entities (money transfer, inventory reservation).
- Access patterns are not fully known yet, or analysts need ad hoc queries.
- Data size fits on one strong server or a primary plus replicas (this covers the vast majority of applications; a single Postgres instance can handle terabytes and thousands of transactions per second).

Choose a **NoSQL database** when a specific requirement pushes you there:

- **Scale beyond one machine** with predictable, simple access (e.g., billions of key lookups per day).
- **Very low, consistent latency** at any scale (DynamoDB, Redis).
- **Data shape** fits another model better: deeply nested documents, highly connected graphs, append-only time series.
- **Flexible or rapidly changing schemas** where enforcing structure upfront is a burden.
- **High write throughput** (Cassandra's LSM-based storage).

Real systems often use both (**polyglot persistence**): Postgres as the system of record, Redis for caching and rate limiting, Elasticsearch for search, a warehouse for analytics.

> **Key takeaway:** Choose based on access patterns, consistency needs, and scale, not trends. Relational is the safe default because it keeps your options open; NoSQL is an optimization for known patterns.

---

### 1.2 Normalization

#### Why normalize?

Storing the same fact in multiple places causes **anomalies**. Consider one denormalized table:

| order_id | customer_id | customer_name | customer_city | product | price |
| -------- | ----------- | ------------- | ------------- | ------- | ----- |
| 1        | 10          | Asha          | Pune          | Pen     | 20    |
| 2        | 10          | Asha          | Pune          | Book    | 150   |
| 3        | 11          | Ravi          | Delhi         | Pen     | 20    |

- **Update anomaly:** Asha moves to Mumbai. You must update every one of her rows; miss one and the data contradicts itself.
- **Insert anomaly:** You can't record a new customer who hasn't ordered yet, because there's no order row to hold them.
- **Delete anomaly:** Deleting order 3 also deletes the only record that Ravi exists.

Normalization is the process of splitting tables so that **each fact lives in exactly one place**. It's based on **functional dependencies**: `A → B` means "if you know A, B is determined." For example, `customer_id → customer_name`.

#### The normal forms

**First Normal Form (1NF): atomic values, no repeating groups**

Each cell holds one value, and there are no repeated columns like `phone1, phone2, phone3`.

```
Violates 1NF:   users(id, name, phones = "98100, 98200")
Fixed:          users(id, name)
                user_phones(user_id, phone)
```

Why: you can't index, constrain, or query individual values hidden inside a list.

**Second Normal Form (2NF): no partial dependencies**

Applies to tables with a **composite key**. Every non-key column must depend on the _whole_ key, not part of it.

```
order_items(order_id, product_id, quantity, product_name)
Key: (order_id, product_id)

quantity      depends on (order_id, product_id)   ✓
product_name  depends on product_id only          ✗ partial dependency
```

Fix: move `product_name` into a `products` table. Otherwise the product name repeats on every order line and can drift out of sync.

**Third Normal Form (3NF): no transitive dependencies**

A non-key column must not depend on another non-key column.

```
employees(emp_id, name, dept_id, dept_name)
emp_id → dept_id → dept_name      ✗ dept_name depends on emp_id only *through* dept_id
```

Fix: `departments(dept_id, dept_name)`. Renaming a department now touches one row.

A classic summary: every non-key attribute must depend on **the key** (1NF), **the whole key** (2NF), and **nothing but the key** (3NF).

**Boyce-Codd Normal Form (BCNF): every determinant is a candidate key**

3NF allows one edge case: a dependency where the determining column is part of a key but isn't itself a key. BCNF closes it: for every `X → Y`, X must be a superkey.

Example: each instructor teaches exactly one course, and a student takes a course from one instructor.

```
enrollments(student, course, instructor)
Key: (student, course)
Dependency: instructor → course       ← instructor is not a key
```

Problem: if the instructor switches courses, many rows need updating. Fix: `instructors(instructor, course)` and `enrollments(student, instructor)`.

Beyond BCNF there are 4NF (multi-valued dependencies) and 5NF (join dependencies). They are rarely needed in practice; in production, **3NF/BCNF is the normal target**.

#### Benefits and costs

| Benefits of normalization                    | Costs                              |
| -------------------------------------------- | ---------------------------------- |
| No redundancy → no anomalies                 | More tables → more joins for reads |
| Smaller tables, less storage                 | Reads can be slower on large data  |
| Writes touch one place                       | Queries are more complex to write  |
| Easier to enforce integrity with constraints |                                    |

---

### 1.3 Denormalization

Denormalization means **intentionally reintroducing redundancy** to make reads faster or simpler. It's done _after_ normalizing, for specific, measured reasons.

**Common denormalization techniques:**

| Technique              | Example                                      | Why                                                                                              |
| ---------------------- | -------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| Duplicate a column     | Store `customer_name` on `orders`            | Avoid a join on the hot "list orders" page                                                       |
| Precomputed aggregates | `posts.comment_count`, `orders.total_amount` | Avoid `COUNT(*)` / `SUM()` on every read                                                         |
| Summary tables         | `daily_sales(date, region, revenue)`         | Dashboards query small precomputed tables                                                        |
| Materialized views     | See Section 9                                | DB maintains a stored query result                                                               |
| Embedding (NoSQL)      | Order document contains its line items       | One read fetches everything                                                                      |
| Historical snapshot    | Store `price_at_purchase` on order lines     | Not really redundancy: the product's price may change later, but the order must keep the old one |

**The cost: keeping copies in sync.** Every duplicated value needs an update mechanism:

- Update both places in the **same transaction** (simplest, strongest).
- **Triggers** that maintain the copy automatically.
- **Asynchronous** updates via events/CDC (scalable, but temporarily inconsistent).
- **Periodic rebuilds** (for summary tables where slight staleness is fine).

**When denormalization makes sense:** read-heavy workloads, analytics (OLAP star schemas are deliberately denormalized), NoSQL stores without joins, and hot paths where a join has been proven to be the bottleneck.

> **Key takeaway:** Normalize to get correctness; denormalize selectively to get speed, and always have a plan for keeping duplicated data consistent.

---

### 1.4 Keys

Keys are how rows are identified and how tables reference each other.

| Key type             | Definition                                                                       | Example                                   |
| -------------------- | -------------------------------------------------------------------------------- | ----------------------------------------- |
| **Super key**        | Any set of columns that uniquely identifies a row                                | `(id)`, `(id, email)`, `(email)`          |
| **Candidate key**    | A _minimal_ super key: remove any column and it stops being unique               | `(id)`, `(email)`                         |
| **Primary key (PK)** | The candidate key chosen as the main identifier. Unique, NOT NULL, one per table | `id`                                      |
| **Alternate key**    | Candidate keys not chosen as PK; enforce with `UNIQUE`                           | `email`                                   |
| **Foreign key (FK)** | Column(s) referencing a PK/unique key in another (or the same) table             | `orders.customer_id → customers.id`       |
| **Composite key**    | A key made of two or more columns                                                | `(order_id, product_id)` in `order_items` |

#### Foreign keys and referential integrity

A foreign key guarantees you can't have an order pointing at a customer that doesn't exist. You also decide what happens when the referenced row changes:

| Action                   | On delete of parent                                                   |
| ------------------------ | --------------------------------------------------------------------- |
| `RESTRICT` / `NO ACTION` | Block the delete if children exist (default)                          |
| `CASCADE`                | Delete the children too (e.g., delete order → delete its order_items) |
| `SET NULL`               | Set the FK column to NULL (e.g., employee's manager leaves)           |
| `SET DEFAULT`            | Set the FK to its default                                             |

Practical notes:

- Postgres does **not** automatically index foreign key columns. Without an index on `orders.customer_id`, deleting a customer forces a full scan of orders to check for children, and joins on it are slower. Index your FKs.
- Some high-scale systems drop FK constraints (enforcing integrity in the app) because FKs add write overhead and complicate sharding. This is a deliberate tradeoff, not a default.

#### Surrogate vs natural keys

A **natural key** has real-world meaning (email, national ID, ISBN, country code). A **surrogate key** is a meaningless system-generated identifier (auto-increment integer, UUID).

|           | Surrogate key                                                               | Natural key                                                   |
| --------- | --------------------------------------------------------------------------- | ------------------------------------------------------------- |
| Stability | Never changes                                                               | Can change (people change emails, regulations change formats) |
| Size      | Small (4–16 bytes), fast joins                                              | Can be long strings                                           |
| Meaning   | None, so no business logic leaks into it                                    | Meaningful, readable                                          |
| Risk      | Duplicates of the real-world entity unless you also add a UNIQUE constraint | Changes cascade to every referencing table                    |
| Privacy   | Safe to expose (especially UUIDs)                                           | Exposing SSNs or emails in URLs is a leak                     |

**Common practice:** a surrogate primary key, plus a `UNIQUE` constraint on the natural key to prevent duplicates. Natural keys are fine when they are truly immutable and standard (ISO country codes, currency codes).

#### Auto-increment vs UUID

|                     | Auto-increment (`BIGSERIAL`, `IDENTITY`)           | UUID v4 (random)                                                        | UUID v7 / ULID (time-ordered)   |
| ------------------- | -------------------------------------------------- | ----------------------------------------------------------------------- | ------------------------------- |
| Size                | 8 bytes                                            | 16 bytes                                                                | 16 bytes                        |
| Generated by        | Database (central)                                 | Anyone (app, client)                                                    | Anyone                          |
| Index locality      | Excellent: always appends to the end of the B-tree | Poor: random inserts across the whole index → page splits, cache misses | Good: roughly sequential        |
| Guessable           | Yes (`/orders/1001` → try 1002), leaks volume      | No                                                                      | Partially reveals creation time |
| Distributed systems | Needs coordination                                 | Great                                                                   | Great                           |

UUIDv7 gives you the distributed generation of UUIDs with the index friendliness of sequential IDs, which is why it's increasingly recommended.

---

### 1.5 Constraints

Constraints are rules the database enforces on every write, regardless of which application, script, or person makes the change. Application validation can be bypassed (a bug, a manual SQL fix, a second service); constraints cannot.

| Constraint           | Purpose                                   | Example                                                 |
| -------------------- | ----------------------------------------- | ------------------------------------------------------- |
| `NOT NULL`           | Value required                            | `email TEXT NOT NULL`                                   |
| `UNIQUE`             | No duplicates                             | `UNIQUE (email)`                                        |
| `PRIMARY KEY`        | Unique + not null identifier              | `id BIGINT PRIMARY KEY`                                 |
| `FOREIGN KEY`        | Value must exist in referenced table      | `REFERENCES customers(id)`                              |
| `CHECK`              | Arbitrary condition on the row            | `CHECK (quantity > 0)`, `CHECK (end_date > start_date)` |
| `DEFAULT`            | Value when none provided                  | `created_at TIMESTAMPTZ DEFAULT now()`                  |
| `EXCLUDE` (Postgres) | No two rows may "conflict" by an operator | No overlapping bookings for the same room               |

```sql
CREATE TABLE bookings (
  room_id INT,
  during  TSTZRANGE,
  EXCLUDE USING gist (room_id WITH =, during WITH &&)   -- && = overlaps
);
```

**Uniqueness and concurrency:** A UNIQUE constraint is the most reliable way to prevent duplicates under concurrency. Checking "does this email exist?" in the app and then inserting has a race: two requests can both see "no" and both insert. The constraint makes one of them fail.

**NULL semantics to remember:** `NULL` means "unknown." `NULL = NULL` is not true (it's unknown), so use `IS NULL`. In UNIQUE constraints, multiple NULLs are normally allowed (Postgres 15+ offers `NULLS NOT DISTINCT` to change this).

**Deferrable constraints:** `DEFERRABLE INITIALLY DEFERRED` checks a constraint at COMMIT instead of after each statement. Useful when rows reference each other circularly within one transaction.

---

### 1.6 Cardinality and ER Modeling

**Cardinality** describes how many rows on one side of a relationship relate to rows on the other.

| Relationship           | Example            | Implementation                                                                                                                                                          |
| ---------------------- | ------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **One-to-one (1:1)**   | User ↔ UserProfile | FK with a UNIQUE constraint on the child, or the same PK in both tables. Often merged into one table unless the second part is optional, large, or sensitive.           |
| **One-to-many (1:N)**  | Customer → Orders  | FK on the "many" side (`orders.customer_id`).                                                                                                                           |
| **Many-to-many (M:N)** | Students ↔ Courses | A **junction (associative) table**: `enrollments(student_id, course_id)` with a composite PK. The junction table can carry its own attributes (`enrolled_at`, `grade`). |

**Optionality** (participation) is the other half: must every order have a customer (mandatory, `NOT NULL` FK), or can an employee have no manager (optional, nullable FK)?

**Self-referencing relationships:** `employees.manager_id → employees.id` models hierarchies (queried with recursive CTEs, Section 9).

#### ER modeling process

An **Entity-Relationship (ER) diagram** is the blueprint before you write DDL.

1. **Identify entities**: the nouns in the requirements (Customer, Order, Product).
2. **Identify attributes** of each entity, and which ones identify it.
3. **Identify relationships** and their cardinality/optionality (a Customer _places_ many Orders).
4. **Resolve M:N relationships** into junction tables.
5. **Choose keys** (surrogate vs natural).
6. **Normalize** to 3NF/BCNF.
7. **Add constraints** to express business rules.
8. **Review against access patterns** and denormalize where needed.

Crow's foot notation is the most common diagram style: `||` means exactly one, `o<` means zero or many, `|<` means one or many.

---

### 1.7 OLTP vs OLAP

Databases serve two very different workloads.

**OLTP (Online Transaction Processing)** runs the application: placing orders, updating profiles, transferring money. Each query touches a few rows by key, and there are many concurrent users.

**OLAP (Online Analytical Processing)** answers business questions: revenue by region per month, cohort retention. Each query scans and aggregates huge amounts of data, and there are few concurrent users.

|                        | OLTP                                   | OLAP                                                     |
| ---------------------- | -------------------------------------- | -------------------------------------------------------- |
| Typical query          | `SELECT * FROM orders WHERE id = 123`  | `SELECT region, SUM(revenue) FROM sales GROUP BY region` |
| Rows touched per query | Few                                    | Millions to billions                                     |
| Operations             | Inserts, updates, deletes, point reads | Large scans and aggregations                             |
| Latency target         | Milliseconds                           | Seconds to minutes is acceptable                         |
| Concurrency            | Thousands of users                     | Tens of analysts/dashboards                              |
| Schema                 | Normalized                             | Denormalized **star/snowflake schema**                   |
| Data freshness         | Real-time                              | Often loaded periodically (ETL/ELT)                      |
| Systems                | Postgres, MySQL, DynamoDB              | Snowflake, BigQuery, Redshift, ClickHouse, DuckDB        |

**Star schema:** a central **fact table** (events or measurements, e.g. each sale with quantity and amount) surrounded by **dimension tables** (date, product, customer, store). A snowflake schema normalizes the dimensions further.

**Why separate them?** A heavy analytics query on the production OLTP database competes for CPU, memory, and I/O, and can hold back cleanup (Section 12). Common setups: run analytics on a read replica, or stream data via ETL/CDC into a warehouse. **HTAP** systems (TiDB, SingleStore) attempt to serve both.

---

### 1.8 Row Stores vs Column Stores

This is about **how data is physically laid out on disk**, and it's the main reason OLTP and OLAP databases differ.

```
Table: (id, name, country, amount)

Row store (pages hold whole rows):
[1, Asha, IN, 100] [2, Ravi, IN, 250] [3, Mei, SG, 75] ...

Column store (each column stored separately):
id:      [1, 2, 3, ...]
name:    [Asha, Ravi, Mei, ...]
country: [IN, IN, SG, ...]
amount:  [100, 250, 75, ...]
```

**Row stores** (Postgres, MySQL) are ideal when you read or write whole records: fetching one order, inserting a new row, updating one customer. All of a row's data sits together in one page.

**Column stores** (Redshift, BigQuery, ClickHouse, Parquet files) are ideal for analytics:

- **Read only needed columns.** `SELECT AVG(amount) FROM sales` reads only the `amount` column. If the table has 50 columns, that's roughly 1/50 of the I/O.
- **Excellent compression.** Values in one column are similar (same type, often repeating), so techniques like run-length encoding (`IN, IN, IN, IN` → `IN×4`) and dictionary encoding compress data 5–10× or more.
- **Vectorized execution.** CPUs process arrays of same-typed values very efficiently (SIMD).

**Downsides of column stores:** inserting or updating a single row means touching many separate column files, so they handle small, frequent writes poorly. They typically batch writes.

> **Key takeaway:** Row store = fast access to whole records (OLTP). Column store = fast scans over a few columns of many rows (OLAP).

---

## 2. Indexing and Internals

### 2.1 How Data Is Stored (Background)

To understand indexes, you first need to know how a table sits on disk.

- **Pages (blocks):** Databases read and write data in fixed-size pages, not individual rows. Postgres pages are 8 KB; InnoDB pages are 16 KB. Reading one row means reading at least its whole page.
- **Heap (Postgres):** A Postgres table is a "heap": an unordered collection of pages. New rows go wherever there is free space. Each row is addressed by a **TID / ctid** `(page number, slot in page)`.
- **Clustered storage (InnoDB, SQL Server):** The table _is_ a B+tree ordered by the primary key. Rows are stored in the leaf pages in PK order.
- **Buffer pool / shared buffers:** An in-memory cache of pages. Reading from memory takes about 100 nanoseconds; reading from SSD takes tens to hundreds of microseconds; spinning disk takes milliseconds. A database's performance depends heavily on how many page reads it needs and how many are served from memory.

**Without an index**, finding `WHERE email = 'a@x.com'` requires a **sequential scan**: read every page of the table and check every row. For a 10 GB table that's over a million pages. With an index, it's a handful of page reads.

### 2.2 What an Index Is

An index is a **separate, sorted data structure** that maps column values to row locations, like the index at the back of a book that maps a term to page numbers.

```sql
CREATE INDEX idx_users_email ON users (email);
```

- Reads that filter, join, or sort on `email` can use the index.
- Every write to `users` must also update the index.
- The index takes its own disk space and memory.

---

### 2.3 B-tree vs B+tree

Both are **balanced, multi-way search trees** designed for disk. Unlike a binary tree (2 children per node), each node holds hundreds of keys and has hundreds of children, because a node is sized to fit one page. This makes the tree very **wide and shallow**, which means few page reads per lookup.

#### B-tree

- Every node (internal and leaf) stores **keys together with their data/row pointers**.
- A search can stop at an internal node if it finds the key there.
- Range scans require moving up and down the tree (in-order traversal).

#### B+tree

- **Internal nodes store only keys** (used for routing/navigation).
- **All data/row pointers live in the leaf nodes.**
- **Leaves are linked** to their neighbors in a (doubly) linked list.

```
                      [ 40 | 80 ]                  ← root (routing keys only)
                  /        |        \
         [10|20|30]   [50|60|70]   [90|100]        ← internal / leaves
             ↓             ↓            ↓
  leaf: 10→20→30  ⇄  40→50→60→70  ⇄  80→90→100     ← all keys + row pointers, linked
```

#### Why databases use B+trees

1. **Higher fanout.** Internal nodes don't hold data, so more keys fit per page. With ~400 keys per node, a 3-level tree indexes ~400³ = 64 million keys, and a 4-level tree about 25 billion. Any row is reachable in 3–4 page reads, and the top levels are almost always cached in memory.
2. **Efficient range scans.** For `WHERE price BETWEEN 100 AND 500`, find the first leaf for 100, then follow leaf links sideways until 500. No re-traversal of the tree.
3. **Ordered results for free.** Leaves are in key order, so `ORDER BY` on an indexed column can avoid a sort.
4. **Predictable performance.** Every lookup goes to leaf level, so all lookups take the same number of steps.

|                        | B-tree                            | B+tree                                |
| ---------------------- | --------------------------------- | ------------------------------------- |
| Data stored in         | All nodes                         | Leaves only                           |
| Internal node contents | Keys + data                       | Keys only                             |
| Fanout / tree height   | Lower fanout, taller              | Higher fanout, shorter                |
| Range queries          | Tree traversal                    | Linked-leaf scan                      |
| Point lookup           | Can finish early at internal node | Always goes to leaf (consistent cost) |

When people say "B-tree index" in Postgres or MySQL, the actual structure is a B+tree variant.

**Complexity:** search, insert, and delete are O(log n). Inserts into a full page cause a **page split**: the page is divided in two and a new key is pushed up to the parent. Random-order inserts (UUIDv4) cause many splits and half-empty pages; sequential inserts mostly append to the rightmost page.

---

### 2.4 LSM Trees (an Alternative to B-trees)

Not every database uses B-trees for primary storage. **Log-Structured Merge trees** power Cassandra, RocksDB, LevelDB, ScyllaDB, HBase, and the storage layers of many modern systems.

**How an LSM tree works:**

1. Writes go to an in-memory sorted structure (**memtable**) and an append-only log (for durability).
2. When the memtable fills up, it's flushed to disk as an immutable sorted file (**SSTable**).
3. Background **compaction** merges SSTables, discarding overwritten and deleted values.
4. Reads check the memtable, then SSTables from newest to oldest. **Bloom filters** skip files that definitely don't contain the key.

|                     | B-tree                             | LSM tree                                                     |
| ------------------- | ---------------------------------- | ------------------------------------------------------------ |
| Writes              | In-place page updates (random I/O) | Sequential appends (very fast)                               |
| Reads               | Fast and predictable               | May check several files (slower, mitigated by bloom filters) |
| Write amplification | Moderate                           | Compaction rewrites data multiple times                      |
| Space               | Some fragmentation                 | Better compression, but temporary duplicates                 |
| Best for            | Read-heavy, mixed OLTP             | Write-heavy workloads (logs, IoT, messaging)                 |

---

### 2.5 Types of Indexes (Concepts)

#### Clustered vs non-clustered

- **Clustered index:** the table rows themselves are stored in index order. There can be only one (data can only be physically sorted one way). In InnoDB, the primary key is the clustered index.
- **Non-clustered (secondary) index:** a separate structure whose leaves point to rows. In Postgres, all indexes are secondary and point to heap TIDs. In InnoDB, secondary indexes store the **primary key value**, so a lookup goes secondary index → PK → clustered index (two tree traversals). This is one reason a large PK (like a long string) makes every InnoDB secondary index bigger.

Postgres has a `CLUSTER` command that physically reorders a table once by an index, but it doesn't maintain that order for new writes.

#### Composite (multi-column) indexes

```sql
CREATE INDEX idx_orders_cust_status_date ON orders (customer_id, status, created_at);
```

The index is sorted by `customer_id`, then by `status` within each customer, then by `created_at`. Think of a phone book sorted by last name, then first name.

**Leftmost prefix rule:** the index is usable when the query constrains a leading prefix of its columns.

| Query filter                                                        | Uses index?                                                                           |
| ------------------------------------------------------------------- | ------------------------------------------------------------------------------------- |
| `customer_id = 5`                                                   | Yes                                                                                   |
| `customer_id = 5 AND status = 'paid'`                               | Yes                                                                                   |
| `customer_id = 5 AND status = 'paid' AND created_at > '2026-01-01'` | Yes, fully                                                                            |
| `customer_id = 5 ORDER BY status`                                   | Yes, including the sort                                                               |
| `status = 'paid'`                                                   | Generally no (like finding everyone named "Ravi" in a phone book sorted by last name) |
| `customer_id = 5 AND created_at > ...`                              | Partially: seeks on customer_id, then filters created_at                              |

**Column order guideline:** put columns used with **equality** first, then the column used for **ranges or sorting** last. A range condition on an earlier column stops the index from using later columns efficiently.

#### Covering indexes and index-only scans

If an index contains every column a query needs, the database can answer from the index alone without visiting the table:

```sql
CREATE INDEX idx_orders_cust ON orders (customer_id) INCLUDE (total, status);
SELECT total, status FROM orders WHERE customer_id = 5;   -- index-only scan
```

`INCLUDE` adds columns to the leaves without making them part of the sort key. In Postgres, index-only scans also depend on the **visibility map** being up to date (maintained by VACUUM, Section 12).

#### Other variants

- **Unique index:** enforces uniqueness (that's how UNIQUE constraints are implemented).
- **Partial index:** indexes only rows matching a condition. Smaller and faster.
  ```sql
  CREATE INDEX idx_orders_pending ON orders (created_at) WHERE status = 'pending';
  ```
- **Expression (functional) index:** indexes the result of an expression.
  ```sql
  CREATE INDEX idx_users_lower_email ON users (LOWER(email));
  ```
- **Hash index:** O(1) equality lookups only; no ranges or sorting.
- Postgres-specific types (GIN, GiST, BRIN) are covered in Section 12.

#### Selectivity and cardinality

- **Cardinality** = number of distinct values in a column. `email` is high; `is_active` is low.
- **Selectivity** = fraction of rows a condition matches. `WHERE id = 7` is highly selective (one row); `WHERE country = 'IN'` might match 60% of rows (low selectivity).

Indexes pay off when a condition is **highly selective**. If a query matches a large fraction of the table, it's cheaper to read the table sequentially than to perform a separate random page read for each matching row.

---

### 2.6 When Indexes Hurt

Indexes are not free. Each one is a tradeoff.

**1. Write amplification.** An INSERT into a table with 8 indexes performs 9 structure updates. UPDATEs that change indexed columns must update those indexes, and in Postgres, an update usually creates a new row version which requires index entries in _every_ index unless it qualifies as a **HOT update** (Heap-Only Tuple: no indexed column changed and there's free space on the same page). Write-heavy tables with many indexes become slow and generate more WAL.

**2. Low-cardinality columns.** An index on `status` with three values, where `active` is 90% of rows, is rarely useful for `WHERE status = 'active'`; the planner picks a seq scan anyway. Exception: querying the **rare** value (e.g., `status = 'failed'` at 0.1%). A **partial index** on just that value is ideal.

**3. Storage and memory pressure.** Indexes can collectively exceed the table size. They compete for the buffer cache; if hot indexes don't fit in memory, everything slows down.

**4. Small tables.** A table that fits in a few pages is read faster sequentially.

**5. Unused and redundant indexes.** An index on `(a)` is redundant if `(a, b)` exists (the composite serves `a`-only queries). Unused indexes still cost every write. Find them:

```sql
SELECT relname, indexrelname, idx_scan, pg_size_pretty(pg_relation_size(indexrelid))
FROM pg_stat_user_indexes WHERE idx_scan = 0;
```

**6. Planner confusion.** Many overlapping indexes give the planner more choices, and occasionally it picks a worse one.

**7. Index maintenance.** Indexes bloat over time and may need `REINDEX`. Building an index on a large table locks writes unless you use `CREATE INDEX CONCURRENTLY` (slower, but doesn't block writes).

#### What to index (rules of thumb)

- Primary keys (automatic) and **foreign keys** (not automatic in Postgres).
- Columns in frequent `WHERE`, `JOIN`, and `ORDER BY` clauses with good selectivity.
- Composite indexes shaped to your most frequent queries.
- Verify with `EXPLAIN ANALYZE`; don't guess.

> **Key takeaway:** An index trades write speed, storage, and memory for read speed. Index for your actual query patterns, measure, and remove indexes that don't earn their keep.

---

## 3. Query Optimization

### 3.1 How a Query Is Executed

Understanding the pipeline explains why plans look the way they do:

1. **Parse:** check SQL syntax, build a parse tree.
2. **Analyze/Rewrite:** resolve table and column names, expand views, apply rules.
3. **Plan/Optimize:** generate candidate execution plans (which indexes, which join order, which join algorithm) and **estimate the cost** of each using table statistics. Pick the cheapest.
4. **Execute:** run the plan as a tree of operators. Each node pulls rows from its children (the "iterator" or "volcano" model).

The optimizer is **cost-based**: it doesn't know the true cost, only estimates. Most bad plans come from bad estimates, usually caused by stale or insufficient statistics.

---

### 3.2 EXPLAIN and EXPLAIN ANALYZE

| Command                      | What it does                                                                               |
| ---------------------------- | ------------------------------------------------------------------------------------------ |
| `EXPLAIN query`              | Shows the planner's chosen plan with **estimated** costs and rows. Does not run the query. |
| `EXPLAIN ANALYZE query`      | **Executes** the query and shows actual times, actual rows, and loops next to estimates.   |
| `EXPLAIN (ANALYZE, BUFFERS)` | Adds how many pages came from cache (`shared hit`) vs disk (`read`).                       |

`EXPLAIN ANALYZE` really runs the statement, so for writes wrap it:

```sql
BEGIN;
EXPLAIN ANALYZE DELETE FROM logs WHERE created_at < '2025-01-01';
ROLLBACK;
```

#### Reading a plan

```
Hash Join  (cost=12.50..245.30 rows=1200 width=48) (actual time=0.31..4.20 rows=1180 loops=1)
  Hash Cond: (o.customer_id = c.id)
  ->  Seq Scan on orders o  (cost=0.00..180.00 rows=10000 width=24) (actual time=0.01..1.80 rows=10000 loops=1)
        Filter: (status = 'paid')
        Rows Removed by Filter: 2000
  ->  Hash  (cost=10.00..10.00 rows=200 width=24) (actual time=0.20..0.20 rows=200 loops=1)
        ->  Seq Scan on customers c  (cost=0.00..10.00 rows=200 width=24) (actual ...)
Planning Time: 0.2 ms
Execution Time: 4.5 ms
```

What each part means:

- **Tree structure:** Each `->` is a child node. Execution flows from the **most indented nodes upward**. Here both seq scans run, the customers result is hashed, then the join combines them.
- **`cost=12.50..245.30`:** startup cost (before the first row can be returned) and total cost, in arbitrary planner units (based on settings like `seq_page_cost = 1`, `random_page_cost = 4`). Compare costs between plans, not with milliseconds.
- **`rows=1200`:** the planner's **estimate** of rows output. **`actual ... rows=1180`:** what really happened.
- **`width`:** estimated average row size in bytes.
- **`loops`:** how many times the node ran. In a nested loop, the inner node may run thousands of times; total time = time per loop × loops.
- **`Rows Removed by Filter`:** rows read and then discarded, which is a signal that an index might help.

#### What to look for

1. **Estimated vs actual rows differ massively** (e.g., estimated 10, actual 500,000). This is the #1 cause of bad plans: the planner chose a strategy for a small result and got a huge one. Fix statistics (`ANALYZE`, higher statistics target, extended stats).
2. **Seq Scan on a large table** where you expected an index, with many rows removed by filter.
3. **Nested Loop with a large outer side** and a high `loops` count.
4. **Sorts spilling to disk:** `Sort Method: external merge Disk: 102400kB`. Increase `work_mem` or add an index that provides the order.
5. **The node where most of the time goes.** Optimize the biggest cost first.

Tools such as explain.depesz.com and explain.dalibo.com visualize plans.

---

### 3.3 Scan Types

| Scan                                     | How it works                                                                                             | Chosen when                                                            |
| ---------------------------------------- | -------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| **Sequential Scan**                      | Reads every page of the table in order                                                                   | Small tables, or a large fraction of rows match, or no usable index    |
| **Index Scan**                           | Traverses the index, then fetches each matching row from the heap (random I/O)                           | Few rows match (high selectivity), or order from the index is needed   |
| **Index Only Scan**                      | Answers entirely from the index, no heap access                                                          | All selected columns are in the index and pages are marked all-visible |
| **Bitmap Index Scan + Bitmap Heap Scan** | Scans index(es) to build an in-memory bitmap of matching pages, then reads those pages in physical order | Medium selectivity; can also combine multiple indexes with AND/OR      |

**Why is a seq scan sometimes faster than an index scan?** Sequential reads are cheap (the disk reads ahead, and many rows come per page). An index scan does one random page fetch per matching row. If 30% of rows match, you might fetch nearly every page anyway, in random order, plus the index pages. The planner models this with `random_page_cost` vs `seq_page_cost`. On SSDs, lowering `random_page_cost` (e.g., to 1.1) makes the planner more willing to use indexes.

---

### 3.4 Join Algorithms

**Nested Loop Join**

```
for each row r in outer:
    for each row s in inner matching r:   ← ideally an index lookup
        emit (r, s)
```

- Cost ≈ outer rows × cost of each inner lookup.
- Excellent when the outer side is small and the inner side has an index on the join key (e.g., 10 orders, look up each customer by PK).
- Terrible when both sides are large and the inner has no index (O(n × m)).
- The only algorithm that handles non-equality join conditions in general (`a.x < b.y`).

**Hash Join**

1. **Build phase:** read the smaller input and build a hash table keyed on the join column.
2. **Probe phase:** stream the larger input, looking up each row in the hash table.

- Cost ≈ O(n + m). Great for large, unsorted inputs.
- **Equality joins only.**
- Needs memory for the hash table (`work_mem`); if it doesn't fit, it spills to disk in batches.

**Merge Join**

1. Both inputs sorted on the join key (via sort or ordered index scans).
2. Walk through both in step, like merging two sorted lists.

- Cost ≈ O(n + m) after sorting, sorting being O(n log n) if needed.
- Excellent when both inputs are already sorted (indexes), and for very large inputs.

| Algorithm   | Best when                       | Weakness                       |
| ----------- | ------------------------------- | ------------------------------ |
| Nested Loop | Small outer + indexed inner     | Large inputs without index     |
| Hash Join   | Large unsorted inputs, equality | Memory, equality only          |
| Merge Join  | Pre-sorted large inputs         | Sorting cost if not pre-sorted |

**Join order** matters too: with many tables, the planner searches possible orders (Postgres switches to a genetic algorithm beyond `geqo_threshold`, 12 tables by default).

---

### 3.5 The N+1 Query Problem

The N+1 problem occurs when code runs one query to fetch a list, then one additional query per item.

```python
posts = Post.objects.all()[:50]          # 1 query
for post in posts:
    print(post.author.name)              # 50 more queries (lazy loading)
```

**Why it's bad:** each query has fixed overhead: network round trip, parsing, planning, and result transfer. 51 queries at 2 ms each = 102 ms, versus one query at ~3 ms. At scale this also multiplies DB load. It's hard to spot because each individual query is fast; the problem only shows as many near-identical queries in logs or APM traces.

**Why it happens:** ORMs **lazy-load** relationships by default: accessing `post.author` silently triggers a query.

**Fixes:**

```sql
-- 1. JOIN: one query
SELECT p.*, u.name FROM posts p JOIN users u ON u.id = p.author_id LIMIT 50;

-- 2. Batch load: two queries total
SELECT * FROM posts LIMIT 50;
SELECT * FROM users WHERE id IN (3, 7, 12, ...);   -- all author_ids at once
```

ORM eager-loading equivalents:

| ORM                | JOIN-based                 | Separate batched query                      |
| ------------------ | -------------------------- | ------------------------------------------- |
| Django             | `select_related('author')` | `prefetch_related('comments')`              |
| SQLAlchemy         | `joinedload()`             | `selectinload()`                            |
| Rails ActiveRecord | `eager_load`               | `includes` / `preload`                      |
| Prisma             |                            | `include: { author: true }`                 |
| GraphQL            |                            | DataLoader (batches and caches per request) |

JOINs are best for to-one relationships; batched queries are often better for to-many relationships (a JOIN would duplicate the parent row for every child).

**Detection:** query count per request in APM tools, Django Debug Toolbar, Bullet gem (Rails), `pg_stat_statements` showing a simple query with an enormous call count.

---

### 3.6 Pagination: OFFSET vs Keyset

#### OFFSET/LIMIT

```sql
SELECT * FROM posts ORDER BY created_at DESC LIMIT 20 OFFSET 10000;
```

**How it works internally:** the database must produce the first 10,020 rows in order and then throw away 10,000 of them. Page 1 is fast; page 500 is slow. Cost grows linearly with the offset.

**Consistency problem:** if a new post is inserted while a user is paging, everything shifts by one, so the user sees a duplicate at the top of the next page. Deletions cause skipped rows.

#### Keyset pagination (cursor/seek method)

Remember the last row's sort key and ask for rows _after_ it:

```sql
-- First page
SELECT * FROM posts ORDER BY created_at DESC, id DESC LIMIT 20;

-- Next page: pass the last row's (created_at, id) as the cursor
SELECT * FROM posts
WHERE (created_at, id) < ('2026-11-20 10:15:00', 98231)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

With an index on `(created_at DESC, id DESC)`, the database seeks directly to the cursor position and reads 20 rows. Page 1 and page 10,000 cost the same.

**Why include `id`?** `created_at` may not be unique; two posts with the same timestamp could be skipped or duplicated. A unique tiebreaker makes the ordering total. The row-value comparison `(a, b) < (x, y)` handles this concisely.

|                               | OFFSET                             | Keyset                                                |
| ----------------------------- | ---------------------------------- | ----------------------------------------------------- |
| Performance on deep pages     | Degrades linearly                  | Constant                                              |
| Stable under inserts/deletes  | No                                 | Yes                                                   |
| Jump to arbitrary page number | Yes                                | No (next/previous only)                               |
| Total page count              | Easy (but `COUNT(*)` is expensive) | Not natural                                           |
| Implementation                | Trivial                            | Needs an indexed, unique sort key and cursor encoding |

**Use keyset** for feeds, infinite scroll, public APIs, and batch processing of large tables. **Use offset** for small datasets or admin tables where users jump to page numbers. APIs usually encode the cursor as an opaque base64 token.

---

### 3.7 Why Avoid SELECT *

1. **Extra I/O and network transfer:** fetching 40 columns when you need 3, including large `TEXT`/`JSONB` values (stored out-of-line in Postgres's TOAST tables, so reading them costs extra page fetches).
2. **Prevents index-only scans:** a covering index can't cover "every column."
3. **Fragile code:** adding or reordering a column can break code that depends on position, and changes API payloads unintentionally.
4. **Memory:** larger rows mean larger sorts and hash tables, more likely to spill to disk.
5. **Security:** may expose sensitive columns you didn't intend to return.

`SELECT *` is fine for ad hoc exploration and inside `EXISTS (SELECT * ...)`, where the columns are never read.

---

### 3.8 Statistics and the Query Planner

The planner estimates how many rows each step produces, and those estimates come from **statistics** gathered by `ANALYZE` (run automatically by autovacuum).

For each column, Postgres stores (view them in `pg_stats`):

- **null_frac:** fraction of NULLs.
- **n_distinct:** number of distinct values.
- **most_common_vals / most_common_freqs:** the most frequent values and how often they appear.
- **histogram_bounds:** bucket boundaries describing the distribution of the remaining values.
- **correlation:** how closely physical row order matches column order (affects index scan cost).

Plus table-level `reltuples` (row count) and `relpages`.

**Example:** `WHERE country = 'IN'`. If `IN` is in most_common_vals with frequency 0.6, the planner estimates 60% of rows and picks a seq scan. For `country = 'IS'` (Iceland, rare), it estimates few rows and picks an index scan. Same query shape, different plans.

**Where estimates go wrong:**

- **Stale statistics** after a bulk load or delete: run `ANALYZE` after large data changes.
- **Correlated columns:** the planner assumes columns are independent. `city = 'Mumbai' AND state = 'Maharashtra'` gets estimated as P(city) × P(state), far too low. Fix with extended statistics:
  ```sql
  CREATE STATISTICS stats_city_state (dependencies) ON city, state FROM addresses;
  ANALYZE addresses;
  ```
- **Skewed data** with too few histogram buckets: raise per-column statistics (`ALTER TABLE t ALTER COLUMN c SET STATISTICS 1000`).
- **Expressions and functions:** the planner may use generic guesses; expression indexes come with their own statistics.
- **Parameterized queries / prepared statements:** a generic plan may be chosen that fits typical values but not skewed ones.

---

### 3.9 Slow Query Log and Finding Problem Queries

You can't optimize what you can't see.

**MySQL:**

```ini
slow_query_log = ON
long_query_time = 1          # seconds
log_queries_not_using_indexes = ON
```

Analyze with `pt-query-digest` (Percona Toolkit), which groups similar queries and ranks them.

**PostgreSQL:**

- `log_min_duration_statement = 500` logs any statement over 500 ms.
- `auto_explain` extension logs the **execution plan** of slow queries automatically (crucial, since the plan at the moment of slowness may differ from what you see later).
- `pg_stat_statements` aggregates stats per normalized query (Section 12).
- AWS RDS **Performance Insights** shows DB load broken down by query and wait event.

**Optimization workflow:**

1. Rank queries by **total time** (calls × mean time), not by single slowest execution. A 3 ms query called 2 million times a day may matter more than a 4 s nightly report.
2. Reproduce with `EXPLAIN (ANALYZE, BUFFERS)` using realistic parameters.
3. Identify the expensive node and the cause (missing index, bad estimate, anti-pattern).
4. Fix: add or adjust an index, rewrite the query, update statistics, or change the design (caching, denormalization).
5. Measure again, and watch write impact if you added indexes.

---

### 3.10 Common Query Anti-Patterns

**1. Functions on indexed columns (non-sargable predicates)**

"Sargable" (Search ARGument ABLE) means the condition can use an index seek. Wrapping the column in a function hides it from a plain index, because the index stores `email`, not `LOWER(email)`.

```sql
-- Not sargable
WHERE LOWER(email) = 'a@x.com'
WHERE DATE(created_at) = '2026-11-23'
WHERE EXTRACT(YEAR FROM created_at) = 2026
WHERE price * 1.18 > 1000

-- Sargable rewrites
CREATE INDEX ON users (LOWER(email));                     -- expression index
WHERE created_at >= '2026-11-23' AND created_at < '2026-11-24'
WHERE created_at >= '2026-01-01' AND created_at < '2027-01-01'
WHERE price > 1000 / 1.18
```

**2. Implicit type conversion**

```sql
-- phone is VARCHAR, compared with a number
WHERE phone = 9876543210
```

The database must convert types to compare. In MySQL, comparing a string column to a number converts every row's value to a number, so the index can't be used. In Postgres, type mismatches often cause an error or a cast that disables the index. Similar issues: comparing columns of different types in joins (`INT` vs `BIGINT` is fine; `VARCHAR` vs `INT` isn't), or mismatched collations. **Fix:** match types exactly in literals, parameters, and join columns.

**3. Leading wildcards**

`WHERE name LIKE '%son'` can't use a B-tree (it's sorted by the beginning of the string). `LIKE 'son%'` can. Fix for substring search: `pg_trgm` trigram GIN index, or full-text search.

**4. OR across different columns**

`WHERE email = ? OR phone = ?` can't use a single index. Postgres may combine two indexes with a BitmapOr; otherwise rewrite as `UNION` of two indexed queries.

**5. NOT IN with NULLs**

```sql
SELECT * FROM users WHERE id NOT IN (SELECT user_id FROM banned);
```

If any `banned.user_id` is NULL, this returns **zero rows**, because `x NOT IN (..., NULL)` evaluates to unknown. It also often plans poorly. Use `NOT EXISTS` instead.

**6. Correlated subqueries executed per row**

```sql
SELECT o.*, (SELECT name FROM customers c WHERE c.id = o.customer_id) FROM orders o;
```

Can run once per row. A JOIN usually plans better (modern planners sometimes rewrite this automatically, but not always).

**7. COUNT(\*) on huge tables for UI badges**

Due to MVCC, Postgres can't keep a cheap exact row count; it must check row visibility. For large tables use estimates (`SELECT reltuples FROM pg_class WHERE relname = 'orders'`) or a maintained counter table.

**8. Large IN lists and unbounded queries**

Thousands of literal IN values cost parsing and planning. Use `= ANY($1::int[])`, a temp table join, or batching. Queries without `LIMIT` that can grow unbounded become problems as data grows.

**9. Over-fetching and work in the wrong place**

Pulling 100,000 rows to the application to filter or aggregate in code instead of letting the DB do it with `WHERE`/`GROUP BY`.

**10. Long transactions around slow work**

Holding a transaction open while calling an external API keeps locks held and stops VACUUM from cleaning up dead rows (Section 12).

---

## 4. Transactions and ACID

### 4.1 What a Transaction Is

A transaction groups several operations into one logical unit of work that either **completes entirely or has no effect at all**.

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;   -- Alice
UPDATE accounts SET balance = balance + 100 WHERE id = 2;   -- Bob
COMMIT;      -- or ROLLBACK to undo everything
```

Without transactions, a crash between the two updates would destroy $100. Without isolation, another session might read the accounts mid-transfer. Transactions solve both problems, and **ACID** names the four guarantees involved.

Note: outside an explicit `BEGIN`, most databases run each statement in its own transaction (**autocommit**).

---

### 4.2 ACID Properties in Depth

#### Atomicity: all or nothing

If any part of the transaction fails (constraint violation, crash, explicit ROLLBACK), **all** of its changes are undone. The database is never left with half a transaction applied.

- **Example:** in the transfer above, if the server crashes after the debit but before the credit, on restart the debit is rolled back. Alice's $100 is still in her account.
- **How it's implemented:** undo logs (InnoDB, Oracle) record how to reverse each change; Postgres simply never marks the transaction committed, so its new row versions remain invisible forever (and are later cleaned by VACUUM).
- **Note:** atomicity isn't about concurrency. It's about the ability to abort.

#### Consistency: valid state to valid state

A transaction takes the database from one valid state to another, where "valid" means all declared rules hold: constraints, foreign keys, unique indexes, check conditions, and cascades.

- **Example:** with `CHECK (balance >= 0)`, transferring $100 from an account holding $50 fails, and the whole transaction is rolled back.
- **Important nuance:** the database only enforces the rules you declare. Business invariants it doesn't know about ("total money across all accounts must stay constant") are the application's responsibility; the application uses atomicity and isolation to maintain them. Some authors argue the "C" belongs to the application rather than the database.
- Don't confuse this with **consistency in CAP** (Section 7), which means something different: all replicas returning the latest value.

#### Isolation: concurrent transactions don't interfere

Each transaction behaves as though it runs alone, even when many run concurrently. The _ideal_ is **serializability**: the result equals some sequential ordering of the transactions.

- **Example:** while the transfer runs, a reporting query summing all balances should not see Alice already debited but Bob not yet credited (it would report $100 missing).
- In practice, full isolation is expensive, so databases offer weaker **isolation levels** that allow certain anomalies (Section 4.3).

#### Durability: committed data survives

Once COMMIT returns success, the change survives crashes, power loss, and restarts.

- **Example:** the bank app shows "transfer complete." Seconds later the server loses power. After restart, the transfer is still there.
- **How it's implemented:** the **write-ahead log (WAL)** is flushed (`fsync`) to persistent storage before COMMIT returns (Section 4.5). Data pages can be written later.
- **Limits:** durability on one machine doesn't protect against disk destruction; for that you need replication and backups. Settings like Postgres `synchronous_commit = off` trade a small window of durability (recent commits may be lost on crash, though without corruption) for speed.

---

### 4.3 Isolation Levels and Anomalies

#### The anomalies (what can go wrong concurrently)

**Dirty read:** reading another transaction's uncommitted data.

```
T1: UPDATE accounts SET balance = 0 WHERE id = 1;     (not committed)
T2: SELECT balance FROM accounts WHERE id = 1;  → 0
T1: ROLLBACK;
→ T2 acted on a value that never officially existed.
```

**Non-repeatable read (fuzzy read):** reading the same row twice in one transaction and getting different values because another transaction committed an update in between.

```
T1: SELECT balance FROM accounts WHERE id = 1;  → 500
T2: UPDATE accounts SET balance = 300 WHERE id = 1; COMMIT;
T1: SELECT balance FROM accounts WHERE id = 1;  → 300
```

**Phantom read:** re-running a query with a condition returns a _different set of rows_ because another transaction inserted or deleted rows that match.

```
T1: SELECT COUNT(*) FROM orders WHERE amount > 1000;  → 5
T2: INSERT INTO orders (amount) VALUES (2000); COMMIT;
T1: SELECT COUNT(*) FROM orders WHERE amount > 1000;  → 6
```

**Lost update:** two transactions read a value, modify it, and write it back; one overwrites the other's change.

```
T1: SELECT stock → 10          T2: SELECT stock → 10
T1: UPDATE stock = 9           T2: UPDATE stock = 9
→ Two items sold, stock only decreased by one.
```

**Write skew:** two transactions read the _same_ data, make a decision based on it, and update _different_ rows. Each transaction alone is valid, but together they violate an invariant.

```
Rule: at least one doctor must be on call. Alice and Bob are both on call.
T1 (Alice): SELECT COUNT(*) WHERE on_call → 2, so it's safe to leave
T2 (Bob):   SELECT COUNT(*) WHERE on_call → 2, so it's safe to leave
T1: UPDATE doctors SET on_call = false WHERE name = 'Alice'; COMMIT;
T2: UPDATE doctors SET on_call = false WHERE name = 'Bob';   COMMIT;
→ Nobody is on call.
```

Other examples: double-booking a meeting room (both check "room free?" then each inserts a booking), two users claiming the same username via separate checks, overdrawing a combined account limit.

#### The four standard levels

| Isolation level  | Dirty read    | Non-repeatable read | Phantom read            | Lost update              | Write skew    |
| ---------------- | ------------- | ------------------- | ----------------------- | ------------------------ | ------------- |
| Read Uncommitted | Possible      | Possible            | Possible                | Possible                 | Possible      |
| Read Committed   | **Prevented** | Possible            | Possible                | Possible                 | Possible      |
| Repeatable Read  | Prevented     | **Prevented**       | Possible (per standard) | Implementation-dependent | Possible      |
| Serializable     | Prevented     | Prevented           | **Prevented**           | Prevented                | **Prevented** |

```sql
BEGIN ISOLATION LEVEL SERIALIZABLE;
-- or
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
```

#### How real databases implement them

The SQL standard describes levels by which anomalies they forbid, but real implementations differ:

**PostgreSQL**

- **Read Uncommitted** behaves exactly like Read Committed (Postgres never shows dirty data).
- **Read Committed (default):** each _statement_ sees a fresh snapshot of committed data. Two SELECTs in one transaction can see different data.
- **Repeatable Read:** the whole _transaction_ uses one snapshot taken at its first query. This is **snapshot isolation**. It also prevents phantoms (stronger than the standard requires) and detects lost updates: if you update a row that another transaction modified and committed after your snapshot, you get `could not serialize access due to concurrent update`. It does **not** prevent write skew.
- **Serializable:** implemented with **Serializable Snapshot Isolation (SSI)**. Transactions run on snapshots without extra blocking, while Postgres tracks read/write dependencies. If a pattern could produce a non-serializable outcome (like write skew), one transaction is aborted with a serialization error. **The application must retry** these transactions.

**MySQL InnoDB**

- **Repeatable Read is the default.** Plain SELECTs read from a consistent snapshot. Locking reads (`SELECT ... FOR UPDATE`) and writes use **next-key locks** (row lock + lock on the gap before it) to block inserts into scanned ranges, reducing phantoms.
- Lost updates _can_ occur with plain read-then-write in InnoDB RR, because writes operate on the latest committed row rather than failing.
- **Serializable** converts plain SELECTs into shared-locking reads (more blocking).

**Choosing a level**

- **Read Committed** works for most web applications combined with targeted techniques: atomic updates (`SET stock = stock - 1`), `SELECT ... FOR UPDATE`, optimistic locking, and unique constraints.
- **Serializable** is the cleanest way to guarantee correctness for complex invariants, at the cost of more aborts under contention; design for retries.
- Higher isolation = fewer anomalies but more locking or aborts and lower throughput.

---

### 4.4 MVCC and Snapshot Isolation

#### The problem MVCC solves

With pure locking, a reader takes a shared lock and blocks writers, and a writer's exclusive lock blocks readers. Under load, long reports would block updates and vice versa.

**Multi-Version Concurrency Control** keeps **multiple versions of each row**, so readers look at the version that was current for their snapshot while writers create new versions. Result:

- **Readers never block writers, and writers never block readers.**
- Writers updating the **same row** still conflict and wait for each other (row lock).

#### How Postgres implements MVCC

Every row version (tuple) carries hidden system columns:

- **`xmin`:** the transaction ID (XID) that created this version.
- **`xmax`:** the XID that deleted or replaced it (0 if still live).

Operations:

- **INSERT:** create a tuple with `xmin = current XID`.
- **DELETE:** set `xmax = current XID` on the existing tuple (the data stays on disk).
- **UPDATE:** effectively DELETE + INSERT: mark the old tuple's `xmax` and write a brand-new tuple.

A **snapshot** records which transactions had committed at the moment it was taken. A tuple is visible to a snapshot if its `xmin` committed before the snapshot and its `xmax` is either unset or not yet committed as of the snapshot.

```
Row id=1, balance:
  version A: balance=500, xmin=100, xmax=205
  version B: balance=300, xmin=205, xmax=0

A transaction whose snapshot predates XID 205's commit sees 500.
A newer transaction sees 300.
```

**Consequences:**

- Old versions (**dead tuples**) accumulate and must be removed by **VACUUM** (Section 12). This is the main operational cost of Postgres's design.
- Updates write a whole new row, so frequent updates of wide rows generate a lot of I/O (HOT updates help).
- Long-running transactions keep old snapshots alive, which prevents cleanup and causes bloat.

#### How InnoDB and Oracle implement MVCC

They update rows **in place** and store the previous versions in an **undo log**. A reader needing an older version follows the undo chain to reconstruct it. Purge threads clean undo records no longer needed by any snapshot. Same visible behavior, different cleanup mechanics.

#### Snapshot isolation

A transaction reads from a consistent snapshot as of its start. Writes are checked at commit: if two concurrent transactions modify the **same row**, the second to commit fails ("first committer wins").

- **Prevents:** dirty reads, non-repeatable reads, phantoms (for reads), lost updates (via write-conflict detection).
- **Doesn't prevent:** write skew, because the conflicting transactions write _different_ rows. Neither sees the other's write in its snapshot, and there's no row-level conflict to detect.

---

### 4.5 WAL, Redo/Undo Logs, Crash Recovery, and Checkpoints

#### Write-Ahead Logging (WAL)

**The rule:** before a change to a data page is written to disk, a log record describing the change must be written to durable storage first.

**Why not just write the data pages directly at commit?**

- A transaction may modify pages scattered across the disk (table, several indexes). Writing them all on every commit means slow random I/O.
- A page write can be interrupted halfway (a "torn page"), corrupting it.

**With WAL:**

1. The transaction modifies pages **in memory** (the buffer pool) and appends compact change records to the WAL buffer.
2. At **COMMIT**, the WAL is flushed to disk with `fsync`. This is a **sequential append**, which is fast. Once it's on disk, the commit is durable.
3. The modified ("dirty") data pages are written to the data files **later**, in the background, possibly combining many changes into one write.

If the system crashes, the data files may be out of date, but the WAL contains everything needed to bring them current.

**WAL also enables:**

- **Replication:** stream WAL records to replicas, which replay them (Postgres physical streaming replication).
- **Point-in-time recovery (PITR):** restore a base backup and replay WAL up to any moment (e.g., just before an accidental `DROP TABLE`).
- **Logical decoding / CDC:** decode WAL into row-level change events (Section 7).

MySQL's equivalent of WAL for the storage engine is the InnoDB **redo log**; it separately has the **binlog** used for replication.

#### Redo and undo logs

| Log          | Contains                                                  | Used for                                                                      | Guarantees                                |
| ------------ | --------------------------------------------------------- | ----------------------------------------------------------------------------- | ----------------------------------------- |
| **Redo log** | How to re-apply a change ("page 42, offset 100: write X") | Replaying committed changes not yet written to data files                     | **Durability**                            |
| **Undo log** | How to reverse a change (the old value)                   | Rolling back aborted transactions; building old row versions for MVCC readers | **Atomicity** (and MVCC in InnoDB/Oracle) |

Postgres's WAL is essentially a redo log. It has no undo log, because old row versions remain in the table itself and uncommitted versions are simply never visible.

#### Crash recovery

The classic algorithm (ARIES, used in various forms by most databases):

1. **Analysis:** read the log from the last checkpoint to determine which transactions were active and which pages may be dirty.
2. **Redo:** replay log records from the checkpoint forward, restoring the database to its exact state at the moment of the crash (including changes from uncommitted transactions).
3. **Undo:** roll back transactions that never committed.

Postgres's recovery is simpler: replay WAL from the last checkpoint (redo); transactions without a commit record are treated as aborted, and their row versions stay invisible.

#### Checkpoints

If the database only ever appended to the WAL, recovery would need to replay the entire history, and the WAL would grow forever. A **checkpoint**:

1. Flushes all dirty pages in memory to the data files.
2. Writes a checkpoint record into the WAL.

After a checkpoint, recovery only needs to start from that point, and older WAL segments can be recycled (unless needed by replicas, archiving, or replication slots).

**Tradeoffs:**

- **Frequent checkpoints:** faster recovery, less WAL kept, but more I/O. In Postgres, the first change to each page after a checkpoint writes the entire page to WAL (**full-page writes**, which protect against torn pages), so frequent checkpoints also increase WAL volume.
- **Infrequent checkpoints:** less I/O overhead, but longer recovery after a crash.

Postgres controls this with `checkpoint_timeout`, `max_wal_size`, and spreads checkpoint I/O over time with `checkpoint_completion_target` to avoid I/O spikes.

---

### 4.6 Savepoints

A savepoint marks a point inside a transaction that you can roll back to **without aborting the whole transaction**.

```sql
BEGIN;
INSERT INTO orders (id, customer_id) VALUES (1, 10);

SAVEPOINT before_coupon;
UPDATE coupons SET used = true WHERE code = 'SAVE10';   -- fails: coupon already used
ROLLBACK TO SAVEPOINT before_coupon;                    -- undo just the coupon step

INSERT INTO order_items (order_id, product_id) VALUES (1, 55);
COMMIT;                                                 -- order + items committed
```

**Uses:**

- Handling an expected failure in one step while keeping the rest of the work.
- **Nested transactions in ORMs:** Django's nested `atomic()` blocks and SQLAlchemy's `begin_nested()` are implemented with savepoints.
- In Postgres, any error inside a transaction aborts it entirely ("current transaction is aborted, commands ignored until end of transaction block"). Savepoints are the only way to recover and continue.

**Caution:** savepoints have overhead; creating thousands in one transaction (e.g., one per row in a loop) can hurt performance.

---

### 4.7 Two-Phase Commit (2PC)

**The problem:** a single transaction must update two separate databases (or a database and a message broker), and both must commit or neither. Each system can only guarantee atomicity for itself.

**The protocol** (with one coordinator and multiple participants):

**Phase 1: Prepare (voting)**

1. Coordinator sends PREPARE to every participant.
2. Each participant does all the work, writes it durably to its log, holds its locks, and votes **YES** ("I promise I can commit, even after a crash") or **NO**.

**Phase 2: Commit (decision)** 3. If all voted YES, the coordinator durably records COMMIT and tells everyone to commit. 4. If any voted NO (or timed out), it tells everyone to roll back. 5. Participants acknowledge.

```
Coordinator          DB-A              DB-B
    | --PREPARE------->|                 |
    | --PREPARE------------------------->|
    |<------YES--------|                 |
    |<-----------------------YES---------|
    | --COMMIT-------->|                 |
    | --COMMIT-------------------------->|
```

Postgres supports participants with `PREPARE TRANSACTION 'id'` and `COMMIT PREPARED 'id'`; XA is the standard interface for this in Java and MySQL.

**Problems:**

- **Blocking:** once a participant votes YES, it can't decide alone. If the coordinator crashes before sending the decision, participants are stuck holding locks until it recovers ("in-doubt" transactions). This is the fundamental weakness.
- **Latency:** at least two network round trips plus multiple fsyncs per transaction.
- **Availability:** the transaction can only succeed if every participant and the coordinator are up. More participants means lower overall availability.
- **Coupling:** all participants must support the protocol (most message brokers and many services don't).

Consensus protocols (Paxos/Raft) can make the coordinator fault-tolerant (as Google Spanner does), but in microservice architectures 2PC is generally avoided in favor of **Sagas** and the **Outbox pattern** (Section 7).

---

## 5. Concurrency and Locking

### 5.1 Why Locks Exist

MVCC handles read/write conflicts, but **write/write conflicts** still need coordination: two transactions can't both modify the same row at the same time. Locks also implement stronger guarantees (preventing lost updates and write skew) and protect schema changes.

### 5.2 Shared and Exclusive Locks

| Lock mode                      | Taken for                                          | Allows others to take |
| ------------------------------ | -------------------------------------------------- | --------------------- |
| **Shared (S)** / read lock     | Reading data that must not change while you use it | Other shared locks    |
| **Exclusive (X)** / write lock | Modifying data                                     | Nothing               |

**Compatibility matrix:**

|               | S held    | X held |
| ------------- | --------- | ------ |
| **Request S** | ✓ granted | ✗ wait |
| **Request X** | ✗ wait    | ✗ wait |

This gives "many readers or one writer." With MVCC, ordinary SELECTs in Postgres and InnoDB take **no row locks at all**; shared row locks appear only with explicit locking reads (`FOR SHARE`). Writes always take exclusive row locks, held until the transaction ends.

**Two-Phase Locking (2PL):** a classic way to achieve serializability with locks. Transactions acquire locks during a "growing" phase and release them only at commit or abort (strict 2PL). It's correct but causes heavy blocking and deadlocks, which is why MVCC/SSI became popular.

**Intention locks** (InnoDB, SQL Server): before locking a row, a transaction places an "intention" lock on the table (IS/IX), so a later request to lock the entire table can quickly see that rows inside are locked without scanning every row lock.

---

### 5.3 Row-Level vs Table-Level Locks

|             | Row-level locks                                  | Table-level locks                                                  |
| ----------- | ------------------------------------------------ | ------------------------------------------------------------------ |
| Granularity | Individual rows                                  | Whole table                                                        |
| Concurrency | High: different rows can be modified in parallel | Low: one writer blocks everyone                                    |
| Overhead    | More locks to track                              | Very little                                                        |
| Taken by    | UPDATE, DELETE, SELECT FOR UPDATE                | DDL (ALTER, DROP, TRUNCATE), explicit LOCK TABLE, some maintenance |

**Postgres table lock modes (simplified)**, from weakest to strongest:

- `ACCESS SHARE`: taken by every SELECT. Only conflicts with ACCESS EXCLUSIVE.
- `ROW EXCLUSIVE`: taken by INSERT/UPDATE/DELETE. Lets other reads and writes proceed.
- `SHARE`: taken by `CREATE INDEX` (non-concurrent). Blocks writes.
- `ACCESS EXCLUSIVE`: taken by most `ALTER TABLE`, `DROP`, `TRUNCATE`, `VACUUM FULL`. **Blocks everything, including SELECTs.**

**The migration lock-queue trap:** an `ALTER TABLE` needs ACCESS EXCLUSIVE. If a long-running SELECT holds ACCESS SHARE, the ALTER waits. Every new query arriving afterward queues _behind the ALTER_, even simple SELECTs, so a quick migration can freeze the application. Mitigation:

```sql
SET lock_timeout = '3s';     -- give up rather than block everything; retry later
ALTER TABLE orders ADD COLUMN notes TEXT;
```

**Lock escalation** (SQL Server, not Postgres): when a transaction holds very many row locks, the engine converts them into one table lock to save memory, which suddenly reduces concurrency.

---

### 5.4 Optimistic vs Pessimistic Locking

Both prevent **lost updates** in read-modify-write cycles, but with opposite philosophies.

#### Pessimistic locking: "assume conflict, lock first"

Lock the data when reading it, so nobody else can change it until you're done.

```sql
BEGIN;
SELECT stock FROM products WHERE id = 1 FOR UPDATE;   -- row locked; others wanting it wait
-- application checks stock > 0
UPDATE products SET stock = stock - 1 WHERE id = 1;
COMMIT;                                               -- lock released
```

- **Pros:** conflicts are impossible; no retries needed; straightforward reasoning.
- **Cons:** waiting reduces throughput; locks held during slow work hurt everyone; deadlock risk; doesn't work across a user's "think time" (you can't hold a DB lock while someone edits a form for 10 minutes).
- **Best for:** high contention on short operations: inventory decrement, seat reservation, balance updates, job claiming.

#### Optimistic locking: "assume no conflict, verify at write"

Read without locking, then make the write conditional on the data being unchanged. Usually done with a **version column**.

```sql
-- Read
SELECT id, title, body, version FROM articles WHERE id = 7;    -- version = 3

-- User edits for several minutes...

-- Write only if nobody else changed it
UPDATE articles
SET title = 'New title', body = '...', version = version + 1
WHERE id = 7 AND version = 3;
```

- If the update affects **1 row**, success.
- If it affects **0 rows**, someone else updated the article (version is now 4). The application reloads, then retries, merges, or shows "this record was modified by someone else."

Alternatives to an integer version: an `updated_at` timestamp (risky if two updates share a timestamp), a hash of the row, Postgres's system column `xmin`, or HTTP **ETags** with `If-Match` headers in REST APIs. ORMs support this natively (JPA `@Version`, Rails `lock_version`).

- **Pros:** no locks held, no blocking, no deadlocks, works across long user sessions and stateless HTTP requests.
- **Cons:** under high contention, many attempts fail and retry repeatedly (wasted work, possible starvation); the app must handle conflicts.
- **Best for:** low contention: editing documents, profiles, settings, admin records.

|                              | Pessimistic                 | Optimistic                        |
| ---------------------------- | --------------------------- | --------------------------------- |
| Mechanism                    | Lock on read (`FOR UPDATE`) | Version check on write            |
| Conflict handling            | Others wait                 | Loser retries or reports conflict |
| Contention suited for        | High                        | Low                               |
| Holds DB locks               | Yes, until commit           | No                                |
| Works across user think-time | No                          | Yes                               |
| Deadlock risk                | Yes                         | No                                |

#### Often the best option: avoid read-modify-write altogether

```sql
UPDATE products SET stock = stock - 1 WHERE id = 1 AND stock > 0;
-- check rows affected: 0 means out of stock
```

A single atomic statement locks the row only for the instant of the update and cannot lose updates. Use it whenever the logic can be expressed in SQL.

---

### 5.5 Deadlocks

#### What a deadlock is

A deadlock is a cycle of transactions each waiting for a lock held by another, so none can ever proceed.

```
T1: UPDATE accounts SET ... WHERE id = 1;   -- T1 locks row 1
T2: UPDATE accounts SET ... WHERE id = 2;   -- T2 locks row 2
T1: UPDATE accounts SET ... WHERE id = 2;   -- T1 waits for T2
T2: UPDATE accounts SET ... WHERE id = 1;   -- T2 waits for T1  → deadlock
```

This is a real scenario: Alice sends money to Bob while Bob sends money to Alice, and each transfer locks the sender first.

Four conditions are required (Coffman conditions): mutual exclusion, hold and wait, no preemption, and circular wait. Breaking any one prevents deadlocks; in databases, the practical one to break is **circular wait**.

#### Common causes

- **Inconsistent lock ordering** across code paths (the example above).
- **Long transactions** holding locks while doing other work.
- **Missing indexes:** an UPDATE without a usable index scans and may lock many more rows (InnoDB locks rows it scans), increasing overlap.
- **Lock upgrades:** two transactions hold shared locks on a row, then both try to upgrade to exclusive.
- **Foreign keys and triggers** taking locks you didn't expect (inserting a child row takes a lock on the parent row).
- **Gap locks** in InnoDB under Repeatable Read, where two transactions lock overlapping ranges.

#### Detection and resolution

Databases detect deadlocks automatically by maintaining a **wait-for graph** (nodes = transactions, edges = "waits for"). A cycle means deadlock. The database picks a **victim** (often the transaction that's cheapest to roll back), aborts it with an error, and lets the others continue.

- **Postgres:** a transaction that has waited longer than `deadlock_timeout` (default 1 s) triggers a deadlock check. Error: `ERROR: deadlock detected` (SQLSTATE `40P01`). The log shows both queries.
- **MySQL:** checks immediately (`innodb_deadlock_detect`), error 1213. `SHOW ENGINE INNODB STATUS` shows the latest deadlock.

Some systems use **lock timeouts** instead of detection, which is simpler but can't distinguish a deadlock from a merely slow transaction.

#### Prevention strategies

1. **Acquire locks in a consistent global order.** For the transfer, always lock the lower account ID first:
   ```sql
   SELECT * FROM accounts WHERE id IN (1, 2) ORDER BY id FOR UPDATE;
   ```
2. **Keep transactions short.** Do validation, external calls, and computation before BEGIN or after COMMIT.
3. **Lock everything you need up front** at the start of the transaction rather than incrementally.
4. **Index the columns used in UPDATE/DELETE WHERE clauses** so only the target rows get locked.
5. **Use lower isolation levels** where acceptable (fewer gap locks in InnoDB Read Committed).
6. **Use `NOWAIT` or `lock_timeout`** to fail fast rather than queue.
7. **Batch updates in a deterministic order** (sorted by PK) in bulk jobs.
8. **Always implement retry with backoff.** Deadlocks can be minimized but never fully eliminated in a busy system; treat deadlock and serialization errors as retryable.

---

### 5.6 SELECT FOR UPDATE, NOWAIT, SKIP LOCKED

**`SELECT ... FOR UPDATE`** reads rows and takes exclusive row locks on them until the transaction ends. Other transactions trying to update, delete, or lock those rows wait. Plain SELECTs are not blocked (MVCC).

It's the standard tool for:

- **Preventing lost updates** in read-check-write logic.
- **Preventing write skew**, by locking the rows that the decision depends on. In the on-call example: `SELECT * FROM doctors WHERE on_call FOR UPDATE` forces the second transaction to wait and then see the updated state. (When the decision depends on rows that _don't exist yet_, such as checking for no conflicting bookings, locking can't help directly; use a unique/exclusion constraint, lock a parent row like the room, or Serializable.)

**Lock strength variants (Postgres):**

- `FOR UPDATE`: strongest; blocks other locks of any kind on the row.
- `FOR NO KEY UPDATE`: like FOR UPDATE but allows concurrent inserts of child rows referencing it (taken by normal UPDATEs that don't change the key).
- `FOR SHARE`: shared lock; others can also share-lock, but nobody can modify.
- `FOR KEY SHARE`: weakest; only blocks deletes and key changes (taken by FK checks).

**`NOWAIT`:** instead of waiting for a locked row, fail immediately with an error.

```sql
SELECT * FROM seats WHERE id = 12 FOR UPDATE NOWAIT;
-- ERROR: could not obtain lock on row → tell the user "seat is being booked, try another"
```

**`SKIP LOCKED`:** silently skip rows that other transactions have locked, returning only unlocked rows. This turns a table into an efficient **job queue**:

```sql
BEGIN;
SELECT id, payload FROM jobs
WHERE status = 'pending'
ORDER BY created_at
LIMIT 10
FOR UPDATE SKIP LOCKED;

-- process jobs, then
UPDATE jobs SET status = 'done' WHERE id IN (...);
COMMIT;
```

Ten workers running this concurrently each grab a different batch with no blocking and no double-processing. If a worker crashes, its transaction rolls back, releasing the locks, and the jobs become available again. Libraries like Que, Oban, Solid Queue, and graphile-worker use this pattern. (SKIP LOCKED gives an inconsistent view of the data by design, so it's for queue-like work, not general queries.)

---

### 5.7 Idempotency

An operation is **idempotent** if performing it multiple times has the same effect as performing it once.

**Why it matters:** in distributed systems, you often can't tell whether a request failed or just its response was lost. Clients retry, users double-click, message queues deliver at least once, and load balancers retry timeouts. Without idempotency, a retry can charge a card twice or create duplicate orders.

| Operation                                         | Idempotent?                                  |
| ------------------------------------------------- | -------------------------------------------- |
| `UPDATE users SET status = 'active' WHERE id = 5` | Yes (absolute value)                         |
| `DELETE FROM sessions WHERE id = 9`               | Yes                                          |
| `UPDATE accounts SET balance = balance + 100`     | **No** (relative change)                     |
| `INSERT INTO orders ...` with generated ID        | **No**                                       |
| HTTP GET, PUT, DELETE                             | Yes by definition (if implemented correctly) |
| HTTP POST                                         | No by default                                |

**Techniques:**

**1. Idempotency keys.** The client generates a unique key per logical operation (UUID) and sends it with the request. The server records it atomically with the result.

```sql
CREATE TABLE idempotency_keys (
  key          TEXT PRIMARY KEY,
  request_hash TEXT,
  response     JSONB,
  created_at   TIMESTAMPTZ DEFAULT now()
);
```

Flow: in one transaction, insert the key (the PRIMARY KEY rejects duplicates) and perform the operation; store the response. On a repeat request with the same key, return the stored response without redoing the work. Stripe's API popularized this with the `Idempotency-Key` header. Handle the in-progress case (first request still running) and expire old keys.

**2. Unique constraints on natural identifiers.** `UNIQUE (order_id)` on `payments` means a second payment for the same order fails. `UNIQUE (message_id)` on a processed-messages table deduplicates consumer messages.

**3. Upserts.**

```sql
INSERT INTO user_settings (user_id, theme) VALUES (5, 'dark')
ON CONFLICT (user_id) DO UPDATE SET theme = EXCLUDED.theme;

INSERT INTO processed_events (event_id) VALUES ('evt_123')
ON CONFLICT DO NOTHING;   -- 0 rows inserted → already processed, skip
```

**4. Absolute instead of relative writes**, and **state-machine conditions:** `UPDATE orders SET status = 'shipped' WHERE id = 7 AND status = 'paid'` only succeeds once.

Idempotency is the foundation for safe retries, at-least-once messaging (outbox, CDC, Kafka consumers), and Saga compensations (Section 7).

---

## 6. Scaling and Distribution

### 6.1 Vertical vs Horizontal Scaling

**Vertical scaling (scale up):** move to a bigger machine with more CPU cores, RAM, and faster storage.

- No application changes, no distributed-systems complexity, full ACID and joins remain trivial.
- Limits: there's a largest machine you can buy, cost grows faster than capacity at the high end, and resizing often needs a restart or failover.
- One machine is still a single point of failure (mitigated with a standby replica).

**Horizontal scaling (scale out):** add more machines and spread the work across them.

- Near-unlimited capacity, commodity hardware, fault tolerance (one node failing isn't fatal).
- Costs: replication lag, consistency problems, distributed transactions, cross-node queries, and much more operational complexity.

|                 | Vertical                | Horizontal             |
| --------------- | ----------------------- | ---------------------- |
| Method          | Bigger server           | More servers           |
| App changes     | None                    | Often significant      |
| Upper limit     | Hardware ceiling        | Practically unlimited  |
| Fault tolerance | Single point of failure | Survives node failures |
| Consistency     | Simple                  | Harder (Section 7)     |

**Reads vs writes scale differently.** Reads scale horizontally fairly easily with replicas and caches. Writes are harder: every write must reach the leader (in leader-based replication), so scaling writes beyond one machine requires **sharding**.

**Typical scaling path for a relational database (do the cheap things first):**

1. Fix slow queries and add the right indexes.
2. Add connection pooling.
3. Scale vertically.
4. Add caching for hot reads.
5. Add read replicas for read traffic.
6. Partition large tables; archive old data.
7. Move specific workloads to specialized stores (search, analytics, queues).
8. Shard only when write volume or data size exceeds one primary.

---

### 6.2 Replication

Replication keeps copies of the same data on multiple nodes. Reasons:

- **High availability:** if one node dies, another takes over.
- **Read scaling:** spread read queries across replicas.
- **Latency:** place replicas near users in different regions.
- **Disaster recovery and isolation:** run backups or analytics on a replica without loading the primary.

#### Leader-follower (primary-replica, master-slave)

```
         writes
Client ─────────► Leader ──WAL stream──► Follower 1   ◄── reads
                         └─────────────► Follower 2   ◄── reads
```

- **One leader accepts all writes.** It records changes in its log (WAL/binlog) and ships them to followers, which apply them in the same order.
- Reads can go to any node (with lag caveats).
- **Pros:** simple, no write conflicts (one source of truth), supported by every major RDBMS (Postgres streaming replication, MySQL replication, RDS read replicas, Aurora replicas).
- **Cons:** write throughput limited to one machine; leader failure requires **failover** (Section 6.6); async followers can serve stale data.

**Physical vs logical replication:** physical replication ships low-level WAL (byte-for-byte copy of the entire cluster, same major version). Logical replication ships row-level changes (`INSERT row X into table Y`), allowing selective tables, different versions (useful for zero-downtime upgrades), and different schemas or indexes on the target.

#### Multi-leader (multi-master)

Several nodes accept writes, typically **one leader per data center or region**, replicating to each other asynchronously.

- **Pros:** writes are fast locally in each region; each region keeps working if inter-region links fail.
- **Cons:** **write conflicts**: two regions modify the same record concurrently. Conflict resolution strategies:
  - **Last-write-wins (LWW):** keep the write with the latest timestamp. Simple, but silently discards data and depends on clock accuracy.
  - **Merge application logic:** custom code decides (e.g., union of shopping cart items).
  - **CRDTs (Conflict-free Replicated Data Types):** data structures (counters, sets, text) mathematically designed to merge without conflicts.
  - **Avoid conflicts:** route each record's writes to a single "home" region.
- Examples: DynamoDB Global Tables (LWW), CouchDB, multi-region Cassandra, offline-first mobile apps (each device is effectively a leader), collaborative editing.

#### Leaderless (Dynamo-style)

There is no leader. The client (or a coordinator node) sends each write to **several replicas** and each read to several replicas.

- **Quorum:** with N replicas per key, a write must be acknowledged by W nodes and a read must hear from R nodes. If **W + R > N**, the set of nodes read overlaps with the set written, so at least one response contains the latest value (the client picks the newest using version numbers).
  - N=3, W=2, R=2: tolerates one node down for both reads and writes.
  - N=3, W=3, R=1: fast reads, but writes fail if any node is down.
  - N=3, W=1, R=1: fastest and most available, but weak consistency.
- **Repairing stale replicas:** **read repair** (on read, update replicas that returned old values), **hinted handoff** (a neighbor holds writes for a down node and forwards them later), **anti-entropy** (background comparison using Merkle trees).
- **Pros:** no failover needed, highly available, tolerant of slow nodes.
- **Cons:** eventual consistency, concurrent writes need conflict resolution, quorums aren't a full linearizability guarantee in edge cases.
- Examples: Cassandra, ScyllaDB, Riak, the original Amazon Dynamo.

|                    | Leader-follower                    | Multi-leader                  | Leaderless                              |
| ------------------ | ---------------------------------- | ----------------------------- | --------------------------------------- |
| Who accepts writes | One leader                         | Several leaders               | Any replica (quorum)                    |
| Write conflicts    | None                               | Yes                           | Yes                                     |
| Failover needed    | Yes                                | Per-leader                    | No                                      |
| Consistency        | Strong on leader, lag on followers | Eventual                      | Tunable/eventual                        |
| Typical use        | Most RDBMS setups                  | Multi-region, offline clients | High-availability, write-heavy at scale |

#### Synchronous vs asynchronous replication

**Synchronous:** the leader waits for the follower to confirm it has received (or applied) the change before telling the client the commit succeeded.

- **Pro:** zero data loss if the leader dies; the follower is guaranteed up to date.
- **Con:** every commit pays the network round trip; if the synchronous follower becomes slow or unreachable, writes stall.

**Asynchronous:** the leader commits locally and replies immediately; followers catch up afterward.

- **Pro:** fast commits; follower problems don't affect writes.
- **Con:** if the leader dies, writes not yet shipped are **lost** on failover; followers serve stale reads.

**Semi-synchronous / quorum commit:** wait for at least one (or k of n) followers, while the rest are async. Postgres: `synchronous_standby_names = 'ANY 1 (replica1, replica2)'`. AWS RDS Multi-AZ uses a synchronous standby for durability; RDS read replicas are asynchronous. Aurora writes to 4 out of 6 storage copies across three availability zones before acknowledging.

---

### 6.3 Replication Lag and Read-Your-Writes

**Replication lag** is the delay between a write committing on the leader and becoming visible on a follower. Usually milliseconds, but it can grow to seconds or minutes under heavy writes, long-running queries on the replica, network trouble, or large transactions.

This lag creates user-visible anomalies:

**1. Read-your-writes (read-after-write) violation**
A user updates their profile picture (write → leader), the page reloads and reads from a replica that hasn't received the change, and the old picture appears. The user thinks the update failed and tries again.

**2. Monotonic reads violation**
A user refreshes twice; the first read hits a caught-up replica and shows a new comment, the second hits a lagging replica where the comment hasn't arrived. Data appears to go backward in time.

**3. Consistent prefix violation**
In partitioned systems, a user sees an answer before the question it responds to, because different partitions replicate at different speeds.

**Solutions:**

| Technique                                      | How it works                                                                                                                                                                      |
| ---------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Read from leader after own writes**          | For data the user can edit (own profile), or for a short window (e.g., 10–60 seconds) after any write, route that user's reads to the leader                                      |
| **Track replication position**                 | After a write, remember the leader's log position (Postgres LSN, MySQL GTID). Only read from a replica that has replayed at least that position; otherwise use the leader or wait |
| **Sticky routing**                             | Pin each user/session to one replica, so reads never go backward (fixes monotonic reads)                                                                                          |
| **Client-side optimism**                       | Update the UI from the client's own data after a successful write                                                                                                                 |
| **Synchronous replication** for critical paths | Slower, but no lag                                                                                                                                                                |
| **Remove lagging replicas**                    | Health checks take replicas out of the read pool when lag exceeds a threshold                                                                                                     |

**Monitoring lag in Postgres:**

```sql
-- On the primary
SELECT client_addr, state, write_lag, flush_lag, replay_lag FROM pg_stat_replication;

-- On a replica
SELECT now() - pg_last_xact_replay_timestamp() AS replica_lag;
```

On RDS, monitor the `ReplicaLag` CloudWatch metric.

---

### 6.4 Sharding and Partitioning

**Partitioning** splits a large dataset into smaller pieces. **Sharding** (horizontal partitioning across servers) places those pieces on different machines so that each machine stores and serves only a fraction of the data. This is the main way to scale **writes** and **storage** beyond one machine.

Terminology:

- **Horizontal partitioning:** split rows (users 1–1M on shard A, 1M–2M on shard B).
- **Vertical partitioning:** split columns or tables (user profile table on one DB, user activity on another), or split by feature/service.
- In Postgres, "partitioning" usually means splitting one table into child tables on the same server (Section 12); "sharding" means multiple servers.

#### Partitioning strategies

**Range partitioning**
Each shard holds a contiguous range of the key.

```
Shard 1: user_id 1 – 1,000,000
Shard 2: user_id 1,000,001 – 2,000,000
or by date: Shard per month
```

- **Pros:** range queries on the key are efficient (they hit one or few shards); easy to understand; easy to archive old ranges.
- **Cons:** **hotspots**. With a timestamp or auto-increment key, _all new writes go to the latest shard_, while older shards sit idle. Uneven data distribution if ranges aren't balanced.

**Hash partitioning**
Apply a hash function to the key and assign by the result: `shard = hash(user_id) % N`.

- **Pros:** spreads keys and load evenly, even for sequential keys.
- **Cons:** range queries must ask **every shard** (scatter-gather), because adjacent keys land on different shards. **Changing N remaps almost every key** (going from 4 to 5 shards moves ~80% of data). Consistent hashing fixes this (Section 6.5).

**Directory (lookup) partitioning**
A lookup table/service maps each key (or tenant) to its shard.

```
tenant_id → shard
acme      → shard-3
globex    → shard-1
bigcorp   → shard-7 (dedicated)
```

- **Pros:** fully flexible: move a single tenant, give large customers dedicated shards, rebalance without changing a formula.
- **Cons:** an extra lookup on every request (usually cached); the directory must be highly available or it becomes a single point of failure.

**Geographic partitioning:** data placed by user region, often for latency and data residency laws (EU data stays in the EU).

#### Choosing a shard key

The shard key is the most consequential decision, because changing it later means moving all the data.

A good shard key:

1. **High cardinality:** many distinct values, so data can be split finely. (`country` is a poor key: only ~200 values, and a few countries dominate.)
2. **Even distribution** of both data size and traffic.
3. **Aligned with access patterns:** the most frequent queries should include the shard key, so they go to a **single shard**. Queries without it must fan out to all shards.
4. **Keeps related data together:** data that's joined or updated together in one transaction should share a shard. For a multi-tenant SaaS, `tenant_id` puts each customer's entire dataset on one shard, so joins and transactions within a tenant still work.
5. **Not monotonically increasing** with range partitioning (timestamps, sequences).

Examples: `tenant_id` for B2B SaaS, `user_id` for social or consumer apps, `device_id` for IoT (often combined with time).

#### Hotspots

A hotspot is a shard that receives a disproportionate share of traffic, which limits the whole system to that shard's capacity.

Causes: a celebrity account, a viral post, a single giant tenant, a monotonic key, a bad hash.

Mitigations:

- **Key salting / suffixing:** append a random number (0–9) to a hot key so its writes spread over 10 partitions; reads then query all 10 and merge.
- **Caching** hot reads.
- **Splitting** the hot shard, or giving a huge tenant a dedicated shard (directory-based).
- **Choosing a compound key** that distributes better (e.g., `(user_id, month)`).

#### Resharding (rebalancing)

Data grows and shards fill up, so you must move data to new shards, ideally online.

Approaches:

- **Consistent hashing:** adding a node moves only a small fraction of keys.
- **Many logical shards, few physical nodes:** create e.g. 1,024 logical shards upfront and map them to 8 servers. To scale, move whole logical shards to new servers; the key-to-logical-shard mapping never changes. (Used by Vitess, Citus, Elasticsearch with fixed shard counts, and many in-house systems.)
- **Dynamic splitting:** split a shard in two when it grows too big (HBase, MongoDB chunks, DynamoDB partitions).
- **Migration process:** copy data to the new location, keep it in sync (dual-writes or CDC), verify, switch reads and writes, then delete old data.

#### The costs of sharding

- **Cross-shard queries:** joins and aggregations across shards need scatter-gather in the application or a coordinator.
- **Cross-shard transactions:** need 2PC or Sagas, or must be designed away.
- **Global uniqueness:** auto-increment IDs per shard collide; use UUIDs, Snowflake-style IDs (timestamp + machine ID + sequence), or ID ranges per shard.
- **Unique constraints across shards** (e.g., unique email) need a separate global lookup.
- **Operational load:** backups, schema migrations, and monitoring across many databases.
- **Rebalancing** is complex and risky.

Tools that handle sharding for you: Vitess (MySQL), Citus (Postgres), and distributed SQL databases (CockroachDB, YugabyteDB, Spanner, TiDB). NoSQL databases (DynamoDB, Cassandra, MongoDB) shard automatically by partition key.

> **Key takeaway:** Sharding scales writes and storage, but it gives up easy joins, transactions, and simplicity. Exhaust cheaper options first, and when you do shard, pick a key that matches your dominant access pattern.

---

### 6.5 Consistent Hashing and Virtual Nodes

#### The problem with modulo hashing

With `node = hash(key) % N`, the mapping depends on N. Going from 4 to 5 nodes changes the result for about 80% of keys, so almost all data must move (or, for a cache, almost every key misses at once).

#### How consistent hashing works

1. Imagine a **ring** representing the hash space (e.g., 0 to 2³² − 1, wrapping around).
2. Hash each **node** (by name or IP) to a position on the ring.
3. Hash each **key** to a position on the ring.
4. A key belongs to the **first node found moving clockwise** from the key's position.

```
                0
          Node A •
        /          \
  key k1 •           • Node B
       |               |
  Node D •           • key k2
        \          /
          • Node C
```

**Adding a node** E between A and B: only keys between A and E (previously owned by B) move to E. Everything else stays put.
**Removing a node:** only its keys move, to the next node clockwise.

On average, adding or removing one node relocates only about **1/N of the keys**.

#### Virtual nodes (vnodes)

With only a few nodes placed by hashing, positions on the ring are uneven: one node might own a huge arc and get far more keys. Also, when a node fails, its entire load falls on the single next node.

**Solution:** each physical node is placed on the ring many times (e.g., 100–256 virtual nodes, by hashing `"nodeA#1"`, `"nodeA#2"`, ...).

- **Smoother distribution:** many small arcs average out, so each physical node gets roughly equal share.
- **Spread failure load:** when a node fails, its many small arcs are absorbed by many different nodes instead of one neighbor.
- **Heterogeneous hardware:** a machine with twice the capacity gets twice the vnodes.
- **Faster rebalancing:** a new node takes small pieces from many existing nodes in parallel.

**Replication on the ring:** to store N copies, a key is placed on the next N _distinct physical_ nodes clockwise.

**Used by:** Cassandra, DynamoDB (internally), Riak, Memcached client libraries (ketama), load balancers, CDNs. Alternative approaches with similar goals include rendezvous (highest random weight) hashing and jump consistent hash.

---

### 6.6 Failover and High Availability

**High availability (HA)** means the system keeps working despite component failures. For databases, the central mechanism is **failover**: when the leader fails, a replica is promoted to become the new leader.

#### Availability numbers

| Availability           | Downtime per year | Downtime per month |
| ---------------------- | ----------------- | ------------------ |
| 99%                    | ~3.65 days        | ~7.3 hours         |
| 99.9% ("three nines")  | ~8.8 hours        | ~43 minutes        |
| 99.99% ("four nines")  | ~53 minutes       | ~4.3 minutes       |
| 99.999% ("five nines") | ~5.3 minutes      | ~26 seconds        |

Two other key metrics:

- **RPO (Recovery Point Objective):** how much data loss is acceptable (e.g., "at most 5 seconds of writes"). Synchronous replication gives RPO ≈ 0.
- **RTO (Recovery Time Objective):** how long recovery may take (e.g., "back online in 2 minutes").

#### The failover process

1. **Detect the failure.** Usually via heartbeats and timeouts. Choosing the timeout is hard: too short causes unnecessary failovers during brief network blips or load spikes; too long means longer outages.
2. **Choose a new leader.** Typically the replica with the most up-to-date data (smallest lag), to minimize lost writes. Ideally decided by a consensus system so that all nodes agree.
3. **Promote it** to accept writes.
4. **Reconfigure clients** to send writes to the new leader: update DNS, move a virtual IP, or update a proxy/connection router (PgBouncer, HAProxy, RDS endpoint).
5. **Reintegrate the old leader** as a follower when it returns. It may have writes the new leader never received, which must be discarded (Postgres `pg_rewind`).

**Manual vs automatic:** automatic failover gives faster recovery, but a false-positive detection can cause more harm than the original issue. Many teams automate failover while carefully tuning detection.

#### Failover risks

**Split brain:** two nodes both believe they're the leader and accept writes, causing divergent, conflicting data. Happens when the old leader isn't actually dead (it was only partitioned from the monitor) and keeps serving clients.

Prevention:

- **Consensus/quorum:** a leader must hold a lease obtained from a majority (via etcd, ZooKeeper, or Consul running Raft/Paxos). A partitioned minority can't elect or keep a leader.
- **Fencing:** forcibly stop the old leader from writing: cut its power (STONITH, "Shoot The Other Node In The Head"), revoke its storage access, or use **fencing tokens** (monotonically increasing numbers checked by the storage layer, which rejects writes from stale leaders).
- **Leases with expiry:** the old leader steps down when it can't renew its lease.

**Data loss with async replication:** the promoted replica may lack the last few writes. Those writes are gone, which can also cause problems if other systems (caches, external services) already saw them (e.g., reused auto-increment IDs).

**Cascading failures:** the new leader receives all the load and may fall over too, especially if replicas were also serving reads.

#### HA tools and managed offerings

- **Patroni** (Postgres): automatic failover coordinated by etcd/Consul/ZooKeeper; widely used on Kubernetes and VMs.
- **pg_auto_failover**, **repmgr**: other Postgres HA managers.
- **MySQL:** Group Replication / InnoDB Cluster, Orchestrator.
- **AWS RDS Multi-AZ:** a synchronous standby in another availability zone; automatic failover by flipping the DNS endpoint, typically within 1–2 minutes. The standby doesn't serve reads (in the classic setup).
- **Amazon Aurora:** compute separated from a shared, replicated storage layer; replicas can be promoted quickly (usually under a minute) because they share the same storage.

HA also includes **backups and point-in-time recovery** (Section 10), because replication faithfully copies mistakes: an accidental `DELETE` is replicated everywhere in milliseconds.

---

## 7. Distributed Data Theory

### 7.1 CAP Theorem

The CAP theorem (Eric Brewer, proven by Gilbert and Lynch in 2002) concerns distributed data stores and three properties:

- **Consistency (C):** every read receives the most recent write or an error. Formally, **linearizability**: the system behaves as if there were a single copy of the data, and every operation takes effect atomically at one instant.
- **Availability (A):** every request to a non-failing node receives a (non-error) response, without a guarantee that it reflects the latest write.
- **Partition tolerance (P):** the system continues operating despite messages between nodes being lost or delayed (network partition).

**The theorem:** when a network partition occurs, a system must choose between consistency and availability.

#### Why the choice is forced

Two replicas, X and Y, lose contact with each other. A client writes `balance = 50` to X. Another client reads the balance from Y.

- If Y **answers** with the old value, the system is available but not consistent (**AP**).
- If Y **refuses** or waits until it can reach X, the system is consistent but not available (**CP**).

There's no third option, because Y cannot know about the write.

#### "Pick two" is misleading

Network partitions are a fact of life in any distributed system (switches fail, cables get cut, GC pauses make nodes unresponsive), so P isn't really optional. The real question is: **during a partition, does the system sacrifice C or A?** When there's no partition, a system can provide both.

| Category | Behavior during partition                                                                                                         | Examples                                                                          |
| -------- | --------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------- |
| **CP**   | Some requests fail or block to avoid returning stale/conflicting data. Typically, the minority side of a partition stops serving. | ZooKeeper, etcd, Consul, HBase, Spanner, MongoDB with majority read/write concern |
| **AP**   | All nodes keep responding, possibly with stale data; divergent writes are reconciled later                                        | Cassandra, DynamoDB (default eventually consistent reads), Riak, CouchDB, DNS     |

Many systems are **tunable** per operation: Cassandra with `QUORUM` reads and writes behaves more like CP; with `ONE` it's AP. DynamoDB offers strongly consistent reads within a region at higher cost.

**Choosing in practice:**

- Use **CP** where incorrect data is worse than downtime: account balances, inventory counts, leader election, distributed locks, configuration.
- Use **AP** where availability matters more and temporary staleness is acceptable: social feeds, likes, product catalogs, shopping carts, metrics.

A single-node database isn't covered by CAP, since there's nothing to partition.

---

### 7.2 PACELC

CAP only describes behavior during partitions, which are relatively rare. PACELC (Daniel Abadi) adds the everyday tradeoff:

> **If** there is a **P**artition, choose between **A**vailability and **C**onsistency; **E**lse (normal operation), choose between **L**atency and **C**onsistency.

**Why there's a latency/consistency tradeoff without failures:** to guarantee every read sees the latest write, a write must be confirmed by multiple replicas (often across data centers) before it's acknowledged, or reads must contact multiple replicas or the leader. That coordination costs network round trips. Serving from the nearest replica immediately is faster but may return stale data.

| System                              | PACELC                   | Explanation                                                                                                              |
| ----------------------------------- | ------------------------ | ------------------------------------------------------------------------------------------------------------------------ |
| Cassandra, Riak, DynamoDB (default) | **PA/EL**                | Favor availability during partitions and low latency otherwise                                                           |
| Spanner, CockroachDB, VoltDB, HBase | **PC/EC**                | Always favor consistency; pay latency for it                                                                             |
| MongoDB (typical config)            | **PA/EC**                | Consistent reads from the primary normally; may elect a new primary and roll back unreplicated writes in some partitions |
| Postgres with async replicas        | **EL** for replica reads | Reads on replicas are fast but may be stale                                                                              |

PACELC is more useful than CAP for real design discussions, because the latency cost of consistency is paid on _every_ request, not just during rare partitions.

---

### 7.3 Consistency Models

A consistency model is a contract describing what values a read may return in a replicated system. From strongest to weakest:

#### Strong consistency (linearizability)

Every read returns the value of the most recent completed write, and all clients observe operations in the same real-time order. The system looks like a single copy.

- **Example:** after transferring money, any balance read anywhere reflects the transfer.
- **Implemented by:** single leader with synchronous replication plus reads from the leader, consensus protocols (Raft/Paxos: etcd, ZooKeeper), Spanner (uses TrueTime, GPS- and atomic-clock-bounded timestamps).
- **Cost:** latency (coordination on writes and often reads) and reduced availability during partitions.
- **Needed for:** locks, leader election, uniqueness guarantees (usernames), balances, inventory.

(Note: **serializability** is a transaction isolation guarantee; **linearizability** is a recency guarantee for single objects. "Strict serializability" combines both. They're different concepts that are often confused.)

#### Sequential consistency

All clients see all operations in the same order, and each client's operations appear in the order it issued them, but that order doesn't have to match real time.

#### Causal consistency

Operations that are **causally related** are seen by everyone in the same order; **concurrent** (unrelated) operations may be seen in different orders by different clients.

- **Example:** Alice posts "Should we go hiking?", Bob replies "Yes!". No one should see Bob's reply without Alice's post. But two unrelated posts by strangers can appear in different orders for different users.
- Causality is tracked with **vector clocks** or similar version metadata.
- It's the **strongest model that remains available during partitions**, which makes it attractive for geo-distributed systems.

#### Session guarantees (client-centric)

Practical guarantees from a single client's perspective, often added on top of eventual consistency:

- **Read-your-writes:** you always see your own updates.
- **Monotonic reads:** you never see older data after having seen newer data.
- **Monotonic writes:** your writes are applied in the order you made them.
- **Writes-follow-reads:** a write made after reading something is ordered after that thing.

#### Eventual consistency

If no new writes occur, all replicas **eventually converge** to the same value. No guarantee about _when_ or what you'll read in the meantime; reads can return stale data or temporarily inconsistent results.

- **Example:** DNS changes take time to propagate; a "like" count differs briefly between data centers.
- **Requires** a conflict-resolution strategy for concurrent writes (last-write-wins, version vectors with merge, CRDTs).
- **Pros:** highest availability and lowest latency.
- **Cons:** application complexity: the code must tolerate stale reads and conflicts.

| Model        | Guarantee                         | Availability during partition | Latency     |
| ------------ | --------------------------------- | ----------------------------- | ----------- |
| Linearizable | Latest value, real-time order     | Low                           | High        |
| Sequential   | Same order for all, not real-time | Low                           | Medium–high |
| Causal       | Cause before effect               | High                          | Low         |
| Eventual     | Converges eventually              | Highest                       | Lowest      |

---

### 7.4 Consensus (Background)

Many distributed databases rely on **consensus algorithms** to get nodes to agree on a value despite failures: who the leader is, what order log entries are in, whether a transaction committed.

- **Raft** (easier to understand, used by etcd, Consul, CockroachDB, TiKV, MongoDB's replication protocol is Raft-like) and **Paxos** (older, used by Spanner, Chubby).
- **How Raft works in brief:** nodes elect a leader using terms (election rounds) and majority votes. The leader appends client commands to its log and replicates them; an entry is **committed** once a **majority** of nodes have stored it. If the leader fails, a new election chooses a node with an up-to-date log.
- **Majority quorum:** with 2f + 1 nodes, the system tolerates f failures (3 nodes tolerate 1, 5 tolerate 2). That's why clusters use odd numbers of nodes.
- Consensus is what prevents split brain in failover systems and enables strongly consistent distributed databases.

---

### 7.5 Distributed Transactions

When one business operation spans multiple databases or services (common with microservices, where each service owns its database), a local ACID transaction no longer covers it.

```
Place order:
  Order Service      → create order        (orders DB)
  Inventory Service  → reserve items       (inventory DB)
  Payment Service    → charge customer     (payments DB / external provider)
  Shipping Service   → schedule shipment
```

If payment fails after inventory is reserved, the reservation must be released. Options:

1. **Two-Phase Commit** (Section 4.7): true atomicity, but blocking, slow, and often unsupported by services and brokers.
2. **Sagas:** a sequence of local transactions with compensations; eventual consistency.
3. **Redesign service boundaries** so data that must change atomically lives in one service and database. Frequently the best answer.

---

### 7.6 Saga Pattern

A **saga** breaks a distributed transaction into a sequence of **local transactions**. Each step commits in its own service. If a step fails, the saga runs **compensating transactions** to semantically undo the steps already completed, in reverse order.

```
Happy path:
T1 Create order (PENDING) → T2 Reserve inventory → T3 Charge payment → T4 Mark order CONFIRMED

Failure at T3 (card declined):
C2 Release inventory → C1 Cancel order (CANCELLED)
```

**Compensation is not rollback.** T1 really committed and others may have seen it. A compensation is a new business action that offsets it: issuing a refund rather than "un-charging," marking an order cancelled rather than deleting it.

#### Two coordination styles

**Choreography (event-driven):** each service listens for events and decides what to do next. No central controller.

```
OrderService publishes OrderCreated
  → InventoryService reserves, publishes InventoryReserved
    → PaymentService charges, publishes PaymentFailed
      → InventoryService releases; OrderService cancels
```

- **Pros:** loose coupling, no single point of control, simple for short flows.
- **Cons:** the overall flow is implicit and spread across services, hard to understand, test, and debug; risk of cyclic dependencies; hard to answer "what state is order 123 in?"

**Orchestration (central coordinator):** an orchestrator tells each service what to do and tracks the saga's state.

```
OrderSagaOrchestrator:
  call Inventory.reserve → ok
  call Payment.charge    → failed
  call Inventory.release
  call Orders.cancel
```

- **Pros:** the flow is explicit in one place; easy monitoring, retries, and timeouts; easier to add steps.
- **Cons:** the orchestrator is another component to build and operate; risk of putting too much business logic in it.
- **Tools:** Temporal, AWS Step Functions, Camunda, Netflix Conductor.

#### Design concerns

- **Lack of isolation:** other transactions can see intermediate states (an order that's PENDING, inventory that's reserved but not yet paid). Countermeasures: **semantic locks** (status fields like `PENDING` that other operations respect), commutative updates, re-reading values before acting, ordering steps so the riskiest or most likely to fail come first.
- **Idempotency:** steps and compensations may be retried (network failures, redelivered messages), so each must be idempotent.
- **Compensations must not fail permanently:** retry them until they succeed; alert humans if they can't.
- **Non-compensatable actions:** some actions can't be undone (sending an email, shipping a package). Put them last ("pivot transaction" then "retriable transactions"), or compensate with a follow-up action (send a correction).
- **Observability:** track saga state with a correlation ID.

|                  | 2PC                                      | Saga                                  |
| ---------------- | ---------------------------------------- | ------------------------------------- |
| Consistency      | Atomic (strong)                          | Eventual                              |
| Isolation        | Yes (locks held)                         | No (intermediate states visible)      |
| Blocking         | Yes                                      | No                                    |
| Failure handling | Automatic rollback                       | Explicit compensations                |
| Scalability      | Poor                                     | Good                                  |
| Suited for       | Few databases in one trusted environment | Microservices, long-running workflows |

---

### 7.7 The Dual-Write Problem and the Outbox Pattern

#### The problem

A service needs to update its database **and** publish an event (to Kafka, SQS, RabbitMQ) so that other services react.

```python
db.save(order)               # 1
kafka.publish(OrderCreated)  # 2
```

These are two separate systems with no shared transaction:

- Crash after step 1: order saved, event never sent → downstream services never learn about the order.
- Publish first, then the DB write fails: event announces an order that doesn't exist.
- Even wrapping the publish inside the DB transaction doesn't help: the publish can succeed and then the commit can fail.

#### The Outbox pattern

Write the event into an **outbox table in the same database, in the same local transaction** as the business change. Atomicity is now guaranteed by the database. A separate process then delivers outbox rows to the message broker.

```sql
BEGIN;
INSERT INTO orders (id, customer_id, total, status)
VALUES ('o-123', 'c-9', 499.00, 'CREATED');

INSERT INTO outbox (id, aggregate_type, aggregate_id, event_type, payload, created_at)
VALUES ('evt-77', 'Order', 'o-123', 'OrderCreated', '{"orderId":"o-123","total":499}', now());
COMMIT;
```

**Delivering outbox events (the relay), two approaches:**

1. **Polling publisher:** a worker periodically selects unsent rows (`FOR UPDATE SKIP LOCKED`), publishes them, and marks them sent or deletes them. Simple; adds some latency and polling load.
2. **Transaction log tailing (CDC):** a CDC tool (Debezium) reads the outbox inserts from the WAL and publishes them. Low latency, no polling, preserves commit order.

**Delivery semantics:** if the relay publishes and then crashes before marking the row as sent, it will publish again after restart. So the outbox provides **at-least-once delivery**, and **consumers must be idempotent** (track processed event IDs, use upserts). The complementary **inbox pattern** on the consumer side records received message IDs in the consumer's DB within the same transaction as its processing.

**Ordering:** publish events for the same aggregate in order (e.g., use `aggregate_id` as the Kafka partition key).

**Housekeeping:** delete or archive processed outbox rows so the table doesn't grow forever.

---

### 7.8 Change Data Capture (CDC)

CDC captures **every row-level change** (INSERT, UPDATE, DELETE) committed in a database and delivers it as a stream of events to other systems.

#### How it works

Instead of querying tables, CDC reads the database's own **transaction log**, which already records every change in commit order:

- **Postgres:** logical decoding of the WAL via a replication slot (output plugins like `pgoutput`).
- **MySQL:** the row-based **binlog**.
- **MongoDB:** change streams (oplog).
- **DynamoDB:** DynamoDB Streams.
- **SQL Server:** built-in CDC tables.

The most common tool is **Debezium**, usually running on Kafka Connect, which emits one event per change, including before and after values, the operation type, and transaction metadata. AWS DMS and Fivetran also offer CDC.

```json
{
  "op": "u",
  "before": { "id": 42, "status": "pending" },
  "after": { "id": 42, "status": "shipped" },
  "source": { "table": "orders", "lsn": 123456789 }
}
```

#### Use cases

- **Keeping derived systems in sync:** search indexes (Elasticsearch), caches (invalidate or update Redis entries), data warehouses and lakes (near-real-time analytics instead of nightly ETL).
- **Event-driven microservices:** other services react to changes without the source service writing extra code.
- **Outbox relay** (Section 7.7).
- **Audit trails** and history tables.
- **Zero-downtime migrations:** replicate data from an old database to a new one, then switch over.

#### CDC vs alternatives

| Approach                                    | Problems                                                                                                                                             |
| ------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Polling** (`WHERE updated_at > last_run`) | Misses hard deletes, misses changes that don't update the timestamp, adds query load, higher latency, can miss rows committed out of timestamp order |
| **Dual writes from the app**                | Inconsistency on partial failure (Section 7.7)                                                                                                       |
| **Triggers writing to a change table**      | Adds overhead to every write in the source transaction                                                                                               |
| **CDC from the log**                        | Captures everything including deletes, low latency, minimal overhead, ordered by commit                                                              |

#### Operational cautions

- **Replication slots retain WAL** until the consumer confirms it. If the CDC consumer stops, WAL accumulates on the primary and can **fill the disk**. Monitor slot lag and alert.
- Schema changes must be handled downstream (schema registry, versioned events).
- Exposing raw table changes couples consumers to your internal schema; the outbox pattern publishes deliberate, stable events instead.

---

## 8. Caching

### 8.1 Why Cache?

A cache stores copies of frequently accessed data in a faster layer (usually memory) so that future requests are served without repeating expensive work.

- **Latency:** a Redis lookup takes well under a millisecond in the same network; a complex DB query may take tens or hundreds of milliseconds.
- **Load reduction:** if 95% of reads hit the cache, the database handles only 5% of read traffic, letting a smaller database serve far more users.
- **Cost:** memory is expensive per GB, but cheaper than scaling the database for read throughput.

**Caching layers:** browser cache → CDN → application in-process cache (e.g., a local LRU map) → distributed cache (Redis, Memcached) → database buffer pool.

**What to cache:** data that's read often and changes rarely, results of expensive computations or aggregations, session data, rendered fragments, API responses from slow third parties.

**What not to cache (or cache carefully):** data that must always be exactly current (account balance during a payment decision), data that changes constantly, and data rarely read twice.

**Key metric:** **hit ratio** = hits / (hits + misses). A low hit ratio means the cache adds latency and complexity without much benefit.

**The fundamental cost:** a cache is a second copy of the data, which can become **stale**. Every caching decision is a tradeoff between freshness, performance, and complexity.

---

### 8.2 Caching Patterns

#### Cache-aside (lazy loading)

The **application** manages the cache explicitly. The cache doesn't know about the database.

**Read:**

1. Look up the key in the cache.
2. **Hit:** return it.
3. **Miss:** query the database, store the result in the cache (with a TTL), return it.

**Write:** update the database, then **delete** (invalidate) the cache key. The next read repopulates it.

```python
def get_user(user_id):
    key = f"user:{user_id}"
    cached = redis.get(key)
    if cached:
        return json.loads(cached)
    user = db.query("SELECT * FROM users WHERE id = %s", user_id)
    redis.set(key, json.dumps(user), ex=3600)   # 1 hour TTL
    return user

def update_user(user_id, data):
    db.execute("UPDATE users SET ... WHERE id = %s", user_id)
    redis.delete(f"user:{user_id}")
```

- **Pros:** simple and widely used; only data that's actually requested gets cached; if the cache goes down, the app still works (slower) by reading the DB.
- **Cons:** the first request for each key is slow (cold cache); data can be stale until the TTL expires or the key is invalidated; cache logic is spread through application code.

**Why delete instead of update on write?** If two requests update the same record concurrently, their cache writes may arrive in the opposite order to their DB writes, leaving the cache permanently holding the older value. Deleting is safer: the next read loads the current DB value. (A small race remains: a reader can load an old value from the DB just before the update and write it into the cache just after the delete. Short TTLs bound how long that lasts; "delayed double delete" or versioned values reduce it further.)

#### Read-through

Like cache-aside, but the **cache itself** loads data from the database on a miss, using a configured loader. The application only talks to the cache.

- **Pros:** simpler application code; loading logic is centralized.
- **Cons:** needs a cache library or provider that supports loaders (e.g., Caffeine/Guava in Java, DAX for DynamoDB, NCache).

#### Write-through

Every write goes to the **cache**, which **synchronously** writes it to the database before acknowledging.

- **Pros:** the cache is always up to date for written data; reads after writes are fast and consistent.
- **Cons:** higher write latency (two writes per operation); the cache fills with data that may never be read (usually combined with TTLs); still needs a strategy for writes that bypass the cache.
- Often paired with read-through.

#### Write-back (write-behind)

Writes go to the **cache only** and are acknowledged immediately. The cache flushes changes to the database **asynchronously**, often in batches.

- **Pros:** very fast writes; absorbs write bursts; batching and coalescing reduce DB load (100 updates to a counter become one DB write).
- **Cons:** **risk of data loss** if the cache fails before flushing; the database is temporarily behind; more complex.
- **Use for:** counters, view counts, analytics events, and other data where losing a few seconds of writes is acceptable.

#### Write-around

Writes go **directly to the database**, bypassing the cache; data enters the cache only when read (combined with cache-aside).

- **Pros:** the cache isn't polluted with data written once and rarely read (logs, bulk imports).
- **Cons:** a read right after a write is a miss.

| Pattern       | Who loads cache | Write path                          | Consistency                              | Write speed | Data loss risk |
| ------------- | --------------- | ----------------------------------- | ---------------------------------------- | ----------- | -------------- |
| Cache-aside   | App             | DB, then delete cache               | Eventually (stale until invalidated/TTL) | Normal      | None           |
| Read-through  | Cache           | (usually with write-through/around) | Same as above                            | Normal      | None           |
| Write-through | Cache           | Cache + DB synchronously            | Strong for cached writes                 | Slower      | None           |
| Write-back    | Cache           | Cache now, DB later                 | DB lags                                  | Fastest     | **Yes**        |
| Write-around  | App on read     | DB only                             | Fresh on next miss                       | Normal      | None           |

---

### 8.3 Cache Invalidation

Invalidation means removing or updating cached data when the source changes. It's famously hard because the cache and the database are separate systems with no shared transaction, so updates can interleave in many ways.

**Strategies:**

1. **TTL (time-based expiration):** every key expires after a set time. Simple, self-healing (even if invalidation logic has bugs, staleness is bounded). Choose the TTL by how stale the data can acceptably be: seconds for prices, hours for user profiles, days for static reference data.
2. **Explicit invalidation on write:** the code that changes data deletes the related keys. Precise and fast, but you must remember every key affected by each write (a product update may affect `product:42`, `category:7:products`, `homepage:featured`).
3. **Event-driven invalidation:** changes publish events (or are captured via CDC), and a consumer invalidates keys. Works across services and catches writes from any source (including manual SQL fixes).
4. **Versioned keys:** embed a version in the key (`product:42:v17`) or a global namespace version (`v3:product:42`). Bumping the version makes old keys unreachable; they then expire naturally. Good for invalidating large groups at once.
5. **Write-through:** the cache is updated as part of every write.

**Best practice:** combine explicit or event-driven invalidation (for freshness) with a TTL (as a safety net).

---

### 8.4 TTL and Eviction

**TTL (Time To Live)** controls **freshness**: how long a value may live before it's considered outdated. **Eviction** controls **space**: which keys to remove when the cache's memory is full, regardless of TTL.

**Choosing TTLs:**

- Shorter TTL → fresher data, lower hit ratio, more DB load.
- Longer TTL → better hit ratio, staler data.
- **Add random jitter** (e.g., 3600 ± 300 seconds) so keys created together don't all expire together (Section 8.5).

**Eviction policies:**

| Policy                          | Evicts                                    | Strengths                                                       | Weaknesses                                                                                        |
| ------------------------------- | ----------------------------------------- | --------------------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| **LRU** (Least Recently Used)   | The key not accessed for the longest time | Good general-purpose choice; adapts to changing access patterns | A one-time scan of many keys can flush out genuinely popular ones                                 |
| **LFU** (Least Frequently Used) | The key accessed the fewest times         | Keeps consistently popular items; resistant to one-off scans    | Slow to adapt when popularity changes (old favorites linger); needs counters that decay over time |
| **FIFO**                        | Oldest inserted                           | Very simple                                                     | Ignores how data is used                                                                          |
| **Random**                      | A random key                              | Cheap, surprisingly OK                                          | Unpredictable                                                                                     |
| **TTL-based**                   | Keys closest to expiry                    | Respects freshness intent                                       | Only works for keys with TTLs                                                                     |

**LRU implementation:** a hash map (O(1) lookup) plus a doubly linked list ordered by recency. On access, move the node to the front; on eviction, remove from the tail. All operations O(1). (A classic coding interview question.)

Redis approximates LRU and LFU by sampling a few keys rather than tracking exact order, which saves memory.

**Redis `maxmemory-policy` options:**

- `noeviction`: return errors on writes when memory is full (good when Redis is a primary store and data loss is unacceptable).
- `allkeys-lru` / `allkeys-lfu`: evict any key (typical for a pure cache).
- `volatile-lru` / `volatile-lfu` / `volatile-ttl`: evict only keys that have a TTL set.
- `allkeys-random` / `volatile-random`.

---

### 8.5 Cache Stampede, Penetration, and Avalanche

These three failure modes all end with the database being overwhelmed because the cache stopped protecting it.

#### Cache stampede (thundering herd, dogpile)

**What happens:** a single **hot key** (e.g., the homepage feed, read 10,000 times/second) expires. In the moment before it's rebuilt, thousands of concurrent requests all miss, and all of them run the same expensive query against the database at once. The DB slows down, the rebuild takes longer, more requests pile up, and it can cascade into an outage.

**Solutions:**

- **Mutex / lock on rebuild (single-flight):** the first request to miss acquires a short lock (`SET lock:key 1 NX EX 10` in Redis) and rebuilds the value; other requests wait briefly and re-check the cache, or receive the stale value. Only one query hits the DB.
- **Serve stale while revalidating:** store the value with a "soft" expiry inside it and a longer real TTL. When the soft expiry passes, one request refreshes it in the background while others continue receiving the slightly stale value.
- **Probabilistic early expiration (XFetch):** each request has a small, increasing chance of refreshing the key _before_ it expires, with the probability rising as expiry approaches. The refresh happens early, for one request, before a crowd forms.
- **Background refresh for known hot keys:** a scheduled job keeps them warm; they never expire under load.
- **Request coalescing** at the application or proxy level.

#### Cache penetration

**What happens:** requests ask for data that **doesn't exist** in the database (e.g., `user:-1` or random IDs, often from bots or attackers). The cache never stores anything for these keys, so every request goes straight to the database. An attacker can bypass the cache entirely.

**Solutions:**

- **Cache negative results:** store a "not found" placeholder (`user:999999 → NULL`) with a short TTL (e.g., 1–5 minutes) so repeats are answered by the cache.
- **Bloom filter:** a compact probabilistic set of all valid keys checked before the cache and DB. If it says "definitely not present," reject immediately. False positives are possible (occasionally a missing key passes through), false negatives are not. Must be updated as keys are added.
- **Input validation:** reject malformed IDs (negative, wrong format) at the API layer.
- **Rate limiting** per client or IP.

#### Cache avalanche

**What happens:** **many keys expire at the same moment** (e.g., all loaded during a cache warm-up with identical TTLs), or the **cache cluster goes down** entirely. A huge share of traffic suddenly hits the database.

**Solutions:**

- **TTL jitter:** randomize expiration times so expirations are spread out.
- **High availability for the cache:** replicas with automatic failover (Redis Sentinel, Redis Cluster, ElastiCache Multi-AZ).
- **Multi-level caching:** an in-process local cache in front of Redis absorbs traffic if Redis is slow or unavailable.
- **Protect the database:** circuit breakers, rate limiting, request queuing, and graceful degradation (serve defaults or partial pages instead of hammering the DB).
- **Gradual warm-up** after a cache restart (pre-load hot keys before taking traffic).

| Problem     | Trigger                                  | Scope                      | Key fixes                                                 |
| ----------- | ---------------------------------------- | -------------------------- | --------------------------------------------------------- |
| Stampede    | One hot key expires                      | One key, many requests     | Lock/single-flight, stale-while-revalidate, early refresh |
| Penetration | Keys that don't exist                    | Requests for missing data  | Cache nulls, Bloom filter, validation                     |
| Avalanche   | Many keys expire together, or cache down | Large portion of the cache | TTL jitter, HA cache, circuit breakers                    |

---

### 8.6 Redis vs Memcached

Both are in-memory key-value stores used as caches, but Redis is far more feature-rich.

|                                      | Redis                                                                                                                    | Memcached                                                             |
| ------------------------------------ | ------------------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------- |
| Data types                           | Strings, hashes, lists, sets, sorted sets, bitmaps, HyperLogLog, streams, geospatial, JSON (module)                      | Strings/byte blobs only                                               |
| Persistence                          | Optional: **RDB** snapshots (point-in-time dumps) and **AOF** (append-only log of every write, fsync configurable)       | None: data lost on restart                                            |
| Replication and HA                   | Primary-replica replication, **Sentinel** (automatic failover), **Redis Cluster** (sharding across 16,384 hash slots)    | None built in; clients shard keys across servers (consistent hashing) |
| Threading                            | Commands execute on a single main thread (atomic, no locks needed); network I/O can be multi-threaded in recent versions | Fully multi-threaded                                                  |
| Atomic operations                    | Many: INCR, transactions (MULTI/EXEC), Lua scripts, functions                                                            | Basic: incr/decr, CAS                                                 |
| Other features                       | Pub/sub, streams with consumer groups, key expiration per key, keyspace notifications                                    | Simple LRU cache                                                      |
| Max value size                       | 512 MB                                                                                                                   | 1 MB default                                                          |
| Memory efficiency for simple strings | Slightly more overhead                                                                                                   | Very efficient (slab allocator)                                       |

**Choose Redis** when you need data structures (leaderboards, queues, rate limiters), persistence, replication/failover, atomic multi-step operations, or pub/sub. It's the default choice today.

**Choose Memcached** for a simple, very large, multi-threaded cache of strings or serialized objects where you need nothing else.

Note: Redis changed its license in 2024; the Linux Foundation fork **Valkey** is now offered by several cloud providers (e.g., AWS ElastiCache for Valkey) and is compatible with Redis.

---

### 8.7 Redis Data Structures and Use Cases

| Structure             | Description                                                       | Key commands                                                             | Use cases                                                                                                                                              |
| --------------------- | ----------------------------------------------------------------- | ------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **String**            | Binary-safe value up to 512 MB; can be treated as a number        | `GET`, `SET` (with `EX`, `NX`), `INCR`, `INCRBY`                         | Caching objects (serialized JSON), counters, rate-limit counters, distributed locks                                                                    |
| **Hash**              | Map of field → value under one key                                | `HSET`, `HGET`, `HGETALL`, `HINCRBY`                                     | Objects with fields (user profile), updating one field without rewriting the whole object                                                              |
| **List**              | Ordered linked list                                               | `LPUSH`, `RPUSH`, `LPOP`, `BRPOP` (blocking), `LRANGE`, `LTRIM`          | Simple queues, recent activity ("last 50 notifications" with LTRIM)                                                                                    |
| **Set**               | Unordered unique strings                                          | `SADD`, `SISMEMBER`, `SINTER`, `SUNION`                                  | Tags, unique visitors per day, mutual friends, deduplication                                                                                           |
| **Sorted Set (ZSET)** | Unique members each with a numeric score, ordered by score        | `ZADD`, `ZINCRBY`, `ZRANGE ... REV`, `ZRANGEBYSCORE`, `ZREMRANGEBYSCORE` | **Leaderboards**, priority queues, delayed job scheduling (score = run time), **sliding-window rate limiting** (score = timestamp), time-ordered feeds |
| **Bitmap**            | String treated as an array of bits                                | `SETBIT`, `GETBIT`, `BITCOUNT`                                           | Daily active users (bit per user ID), feature flags, attendance (very memory efficient)                                                                |
| **HyperLogLog**       | Probabilistic cardinality estimator, ~12 KB per key, ~0.81% error | `PFADD`, `PFCOUNT`, `PFMERGE`                                            | Counting unique visitors/searches at massive scale                                                                                                     |
| **Stream**            | Append-only log with IDs, consumer groups, acknowledgments        | `XADD`, `XREADGROUP`, `XACK`                                             | Event logs, reliable message queues (lightweight alternative to Kafka)                                                                                 |
| **Geospatial**        | Members with coordinates (stored in a sorted set)                 | `GEOADD`, `GEOSEARCH`                                                    | "Find drivers within 2 km"                                                                                                                             |
| **Pub/Sub**           | Fire-and-forget messaging (not stored)                            | `PUBLISH`, `SUBSCRIBE`                                                   | Real-time notifications, chat fan-out                                                                                                                  |

**Example: fixed-window rate limiter (100 requests/minute per user)**

```
key = ratelimit:{user_id}:{current_minute}
count = INCR key
if count == 1: EXPIRE key 60
if count > 100: reject
```

**Example: leaderboard**

```
ZINCRBY leaderboard 50 "player:42"        # add points
ZREVRANGE leaderboard 0 9 WITHSCORES      # top 10
ZREVRANK leaderboard "player:42"          # player's rank
```

**Example: distributed lock**

```
SET lock:order:123 <unique-token> NX PX 30000     # acquire only if absent, auto-expire in 30s
# ... do work ...
# release only if we still own it (check token and delete atomically with a Lua script)
```

Single-instance Redis locks are fine for efficiency (avoiding duplicate work) but not fully safe for correctness under failover or long pauses; for correctness-critical locking, add fencing tokens or use a consensus system (etcd/ZooKeeper) or database locks.

---

## 9. Database Objects and Features

### 9.1 Views vs Materialized Views

#### Views

A **view** is a **named, stored query**. It holds no data; each time you query it, the database runs the underlying query (merging it into your query and optimizing the whole thing).

```sql
CREATE VIEW active_customers AS
SELECT id, name, email, country
FROM customers
WHERE deleted_at IS NULL AND status = 'active';

SELECT * FROM active_customers WHERE country = 'IN';
```

**Why use views:**

- **Abstraction and reuse:** encapsulate complex joins and logic once; consumers query a simple name.
- **Security:** grant access to a view that exposes only certain columns or rows, without granting access to the underlying table (e.g., hide salary columns). Postgres `security_barrier` views prevent information leaks through clever predicates.
- **Stable interface:** the underlying tables can change while the view keeps the same shape for consumers (helpful in migrations).

**Characteristics:** always up to date; performance equals the underlying query (no speedup); simple views can be updatable (INSERT/UPDATE pass through to the base table), complex ones can use `INSTEAD OF` triggers.

#### Materialized views

A **materialized view** runs the query once and **stores the result physically**, like a table. Reads are fast because the work is precomputed, but the data is a snapshot that becomes **stale** until refreshed.

```sql
CREATE MATERIALIZED VIEW monthly_revenue AS
SELECT date_trunc('month', created_at) AS month,
       region,
       SUM(total) AS revenue,
       COUNT(*)   AS orders
FROM orders
GROUP BY 1, 2;

CREATE UNIQUE INDEX ON monthly_revenue (month, region);   -- indexable like a table
```

**Use for:** expensive aggregations behind dashboards, reports, leaderboards, search summaries, data joined from multiple large tables that's read far more often than it changes and can tolerate some staleness.

|                  | View                     | Materialized View     |
| ---------------- | ------------------------ | --------------------- |
| Stores data      | No                       | Yes                   |
| Freshness        | Always current           | Stale until refreshed |
| Read performance | Same as underlying query | Fast (precomputed)    |
| Write overhead   | None                     | Refresh cost          |
| Indexes          | No                       | Yes                   |
| Disk usage       | None                     | Yes                   |

#### Refresh strategies

| Strategy                       | How                                                                                                                              | Tradeoff                                                                                                                        |
| ------------------------------ | -------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| **Full refresh**               | `REFRESH MATERIALIZED VIEW monthly_revenue;`                                                                                     | Recomputes everything; in Postgres it takes an exclusive lock, blocking reads during refresh                                    |
| **Concurrent refresh**         | `REFRESH MATERIALIZED VIEW CONCURRENTLY monthly_revenue;`                                                                        | Reads continue during refresh; requires a UNIQUE index; slower, because it computes the new result and applies only differences |
| **Scheduled**                  | cron, `pg_cron`, Airflow (e.g., every 15 minutes)                                                                                | Predictable staleness bound                                                                                                     |
| **On demand / event-driven**   | Refresh after an ETL load completes, or when a change event arrives                                                              | Fresh right after updates; refresh may be frequent                                                                              |
| **Incremental (fast refresh)** | Apply only changes since the last refresh (Oracle materialized view logs; SQL Server indexed views are maintained automatically) | Efficient, but limited query types; not native in Postgres (needs extensions like `pg_ivm`, or manual approaches)               |
| **Manual summary tables**      | Regular tables maintained by triggers or application code on each write                                                          | Always fresh; adds write cost and complexity                                                                                    |

In analytics warehouses (Snowflake, BigQuery, Redshift), materialized views are often maintained automatically and incrementally.

---

### 9.2 Stored Procedures and Functions

A **stored procedure** is a named program stored and executed inside the database, written in a procedural extension of SQL (PL/pgSQL in Postgres, T-SQL in SQL Server, PL/SQL in Oracle). It can contain variables, loops, conditionals, error handling, and multiple SQL statements.

```sql
CREATE OR REPLACE PROCEDURE transfer(from_id INT, to_id INT, amount NUMERIC)
LANGUAGE plpgsql AS $$
BEGIN
  UPDATE accounts SET balance = balance - amount WHERE id = from_id;
  IF (SELECT balance FROM accounts WHERE id = from_id) < 0 THEN
    RAISE EXCEPTION 'Insufficient funds';
  END IF;
  UPDATE accounts SET balance = balance + amount WHERE id = to_id;
END;
$$;

CALL transfer(1, 2, 100);
```

In Postgres, **functions** (`CREATE FUNCTION`) return a value and can be used inside queries; **procedures** (`CREATE PROCEDURE`, Postgres 11+) are invoked with `CALL` and can control transactions (COMMIT within them).

**Advantages:**

- **Fewer network round trips:** many operations in one call, which matters for chatty logic or high-latency connections.
- **Logic close to the data:** process large datasets without shipping them to the app.
- **Security:** applications can be given permission to execute procedures but not to touch tables directly (`SECURITY DEFINER` runs with the owner's privileges).
- **Consistency:** the same logic regardless of which application calls it.

**Disadvantages:**

- **Harder to version, test, review, and debug** than application code; weaker tooling.
- **Scaling:** the database is usually the hardest tier to scale; putting computation there consumes its CPU.
- **Vendor lock-in:** PL/pgSQL and T-SQL don't port across databases.
- **Split business logic** between the app and the DB makes the system harder to understand.
- **Deployment coupling:** logic changes require database migrations.

**When to use them:** data-intensive batch operations, operations that must be atomic and performant close to the data, enforcing critical rules for multiple applications sharing a database, and generating values used in constraints or indexes. Modern application architectures usually keep most business logic in the application tier.

---

### 9.3 Triggers

A **trigger** automatically runs a function when a specified event occurs on a table.

- **Timing:** `BEFORE` (can modify or reject the row before it's written), `AFTER` (sees the final row; used for side effects), `INSTEAD OF` (on views).
- **Event:** `INSERT`, `UPDATE` (optionally `UPDATE OF column`), `DELETE`, `TRUNCATE`.
- **Granularity:** `FOR EACH ROW` (once per affected row) or `FOR EACH STATEMENT` (once per statement).
- **Conditional:** `WHEN (OLD.status IS DISTINCT FROM NEW.status)`.

```sql
-- Keep updated_at current automatically
CREATE FUNCTION set_updated_at() RETURNS trigger AS $$
BEGIN
  NEW.updated_at := now();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_orders_updated_at
BEFORE UPDATE ON orders
FOR EACH ROW EXECUTE FUNCTION set_updated_at();
```

**Common uses:**

- **Audit logging:** record who changed what and when into a history table (the trigger has access to `OLD` and `NEW` row values).
- **Timestamps** (`updated_at`).
- **Maintaining denormalized data:** counters (`posts.comment_count`), summary tables, search vectors.
- **Complex integrity rules** that CHECK constraints can't express (rules involving other tables).
- **Soft-delete archiving** (copy a deleted row to an archive table).

**Downsides:**

- **Hidden behavior:** a developer runs a simple UPDATE and doesn't realize three other tables change. Surprising side effects make debugging hard.
- **Performance:** row-level triggers execute for every row; a bulk update of a million rows runs the function a million times, inside the same transaction.
- **Cascading triggers:** triggers firing other triggers can create long chains or loops.
- **Contention:** a trigger that updates a single counter row on every insert turns concurrent inserts into a serialized hot spot.
- **Testing and portability** challenges, similar to stored procedures.

Use triggers for narrow, well-documented, data-level concerns, and keep them simple.

---

### 9.4 Common Table Expressions (CTEs)

A CTE (`WITH` clause) defines a **named temporary result set** that exists only for the duration of one query. Its main purpose is readability: breaking a complex query into logical steps.

```sql
WITH customer_spend AS (
  SELECT customer_id, SUM(total) AS spent
  FROM orders
  WHERE created_at >= now() - interval '90 days'
  GROUP BY customer_id
),
top_customers AS (
  SELECT customer_id, spent
  FROM customer_spend
  WHERE spent > 10000
)
SELECT c.name, c.email, t.spent
FROM top_customers t
JOIN customers c ON c.id = t.customer_id
ORDER BY t.spent DESC;
```

Compared with nested subqueries, CTEs read top to bottom like steps, can be referenced multiple times, and are easier to debug (you can run each step alone).

#### Recursive CTEs

A recursive CTE references itself, allowing queries over **hierarchical or graph data** of unknown depth.

```sql
-- Everyone who reports (directly or indirectly) to employee 1
WITH RECURSIVE reports AS (
  SELECT id, name, manager_id, 1 AS depth       -- anchor: the starting row(s)
  FROM employees
  WHERE id = 1

  UNION ALL

  SELECT e.id, e.name, e.manager_id, r.depth + 1  -- recursive step: join to previous results
  FROM employees e
  JOIN reports r ON e.manager_id = r.id
)
SELECT * FROM reports;
```

How it executes: run the anchor; then repeatedly run the recursive part against the rows produced in the previous iteration; stop when an iteration produces no new rows.

**Uses:** org charts, category trees, threaded comments, bill of materials (parts containing parts), graph traversal (friends of friends), generating series of dates. **Caution:** cyclic data can loop forever; guard with a depth limit or track visited nodes (Postgres 14+ has a `CYCLE` clause).

#### CTE performance

- **Postgres 12+:** non-recursive CTEs referenced once are **inlined** into the main query, so the optimizer can push filters into them. Before version 12, CTEs were always materialized and acted as an "optimization fence," which sometimes made them slower than subqueries.
- You can control this explicitly: `WITH x AS MATERIALIZED (...)` (compute once, useful when referenced multiple times or to force a plan) or `NOT MATERIALIZED`.
- CTEs can contain data-modifying statements: `WITH deleted AS (DELETE FROM ... RETURNING *) INSERT INTO archive SELECT * FROM deleted;` moves rows in one atomic statement.

---

### 9.5 Window Functions

A window function computes a value for each row using a set of related rows (the "window"), **without collapsing rows** the way `GROUP BY` does. You keep the row-level detail and get aggregate or ranking context alongside it.

```sql
function() OVER (
  PARTITION BY ...    -- divide rows into groups (like GROUP BY, but rows stay separate)
  ORDER BY ...        -- order within each partition
  ROWS/RANGE ...      -- optional frame: which rows around the current one to include
)
```

**GROUP BY vs window function:**

```sql
-- GROUP BY: one row per department
SELECT department, AVG(salary) FROM employees GROUP BY department;

-- Window: every employee row, plus their department's average
SELECT name, department, salary,
       AVG(salary) OVER (PARTITION BY department) AS dept_avg,
       salary - AVG(salary) OVER (PARTITION BY department) AS diff_from_avg
FROM employees;
```

#### Categories of window functions

**Ranking:**

| Function                        | Behavior with ties (salaries 100, 90, 90, 80)        |
| ------------------------------- | ---------------------------------------------------- |
| `ROW_NUMBER()`                  | 1, 2, 3, 4 (unique, arbitrary among ties)            |
| `RANK()`                        | 1, 2, 2, 4 (gaps after ties)                         |
| `DENSE_RANK()`                  | 1, 2, 2, 3 (no gaps)                                 |
| `NTILE(4)`                      | Splits rows into 4 roughly equal buckets (quartiles) |
| `PERCENT_RANK()`, `CUME_DIST()` | Relative position as a fraction                      |

**Offset (value from another row):**

- `LAG(col, n)`: value from n rows before; `LEAD(col, n)`: n rows after.
- `FIRST_VALUE()`, `LAST_VALUE()`, `NTH_VALUE()`.

**Aggregates as windows:** `SUM`, `AVG`, `COUNT`, `MIN`, `MAX` with `OVER`.

#### Classic examples

**Top N per group** (most common interview pattern):

```sql
SELECT * FROM (
  SELECT name, department, salary,
         ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS rn
  FROM employees
) ranked
WHERE rn <= 3;      -- top 3 earners per department
```

(Window functions can't be used directly in WHERE, because they're computed after filtering; hence the subquery.)

**Latest row per entity / deduplication:**

```sql
SELECT * FROM (
  SELECT *, ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at DESC) AS rn
  FROM logins
) t WHERE rn = 1;
```

(Postgres also offers `SELECT DISTINCT ON (user_id) ... ORDER BY user_id, created_at DESC` for this.)

**Running total:**

```sql
SELECT order_date, amount,
       SUM(amount) OVER (ORDER BY order_date) AS running_total
FROM daily_sales;
```

**Change from the previous period:**

```sql
SELECT month, revenue,
       revenue - LAG(revenue) OVER (ORDER BY month) AS change,
       ROUND(100.0 * (revenue - LAG(revenue) OVER (ORDER BY month))
             / LAG(revenue) OVER (ORDER BY month), 1) AS pct_change
FROM monthly_revenue;
```

**Moving average (window frame):**

```sql
SELECT day, visits,
       AVG(visits) OVER (ORDER BY day ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS avg_7d
FROM traffic;
```

**Frames:** `ROWS` counts physical rows; `RANGE` uses value ranges (rows with equal ORDER BY values are treated as peers). The default frame when ORDER BY is present is `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`, which is why `SUM() OVER (ORDER BY x)` gives a running total, and why `LAST_VALUE` often surprises people (the frame ends at the current row).

**Performance:** window functions usually require sorting by the PARTITION BY and ORDER BY columns; an index on those columns can avoid the sort.

---

### 9.6 Full-Text Search

#### Why LIKE isn't enough

`WHERE body LIKE '%database index%'`:

- Can't use a regular index (leading wildcard), so it scans every row.
- Matches only the exact substring: "indexing databases" won't match.
- No understanding of language (plurals, verb forms), no stop words, no relevance ranking.

#### How full-text search works

1. **Tokenize:** split text into words.
2. **Normalize:** lowercase, remove stop words ("the," "and," "is"), and **stem** words to their root ("running," "runs," "ran" → "run"), using language-specific dictionaries.
3. **Index:** build an **inverted index** mapping each term to the documents (and positions) containing it.
4. **Query:** parse the search into terms with operators (AND, OR, NOT, phrase), look them up in the inverted index, and **rank** results by relevance (term frequency, proximity, field weights).

#### Full-text search in Postgres

```sql
-- tsvector: normalized document; tsquery: normalized search
SELECT to_tsvector('english', 'The quick brown foxes jumped');
-- 'brown':3 'fox':4 'jump':5 'quick':2

ALTER TABLE articles ADD COLUMN search tsvector
  GENERATED ALWAYS AS (
    setweight(to_tsvector('english', coalesce(title, '')), 'A') ||
    setweight(to_tsvector('english', coalesce(body,  '')), 'B')
  ) STORED;

CREATE INDEX idx_articles_search ON articles USING GIN (search);

SELECT id, title, ts_rank(search, query) AS rank
FROM articles, websearch_to_tsquery('english', 'database indexing -mysql') AS query
WHERE search @@ query
ORDER BY rank DESC
LIMIT 20;
```

- `@@` is the match operator.
- `setweight` makes title matches rank higher than body matches.
- `websearch_to_tsquery` accepts search-engine-style input (quotes for phrases, `-` to exclude).
- `ts_headline` generates highlighted snippets.
- For typo tolerance and fuzzy matching, add the `pg_trgm` extension (trigram similarity).

#### When to use a dedicated search engine

Postgres FTS is good for many applications: no extra infrastructure, transactional consistency with your data. Move to **Elasticsearch, OpenSearch, Meilisearch, Typesense, or Solr** when you need: advanced relevance tuning, typo tolerance and synonyms at scale, faceted navigation (filter counts by category), autocomplete, multilingual analysis, aggregations over search results, or very large data and query volumes. The usual architecture keeps the database as the source of truth and syncs the search index via CDC or events, accepting slight indexing delay.

---

### 9.7 JSON Columns

Relational databases now support semi-structured JSON data inside columns, combining flexible schemas with relational guarantees.

#### Postgres `json` vs `jsonb`

|                                                 | `json`                                   | `jsonb`                                      |
| ----------------------------------------------- | ---------------------------------------- | -------------------------------------------- |
| Storage                                         | Exact text copy                          | Decomposed binary format                     |
| Preserves whitespace, key order, duplicate keys | Yes                                      | No (last duplicate key wins, keys reordered) |
| Insert speed                                    | Slightly faster (no parsing into binary) | Slightly slower                              |
| Query speed                                     | Slower (reparsed each time)              | Faster                                       |
| Indexing (GIN)                                  | No                                       | **Yes**                                      |
| Containment/existence operators                 | Limited                                  | Full                                         |

**Use `jsonb`** in almost every case.

#### Querying

```sql
CREATE TABLE products (
  id         BIGSERIAL PRIMARY KEY,
  name       TEXT NOT NULL,
  price      NUMERIC NOT NULL,
  attributes JSONB
);

INSERT INTO products (name, price, attributes)
VALUES ('Laptop', 75000, '{"brand": "Acme", "ram_gb": 16, "colors": ["silver", "black"], "specs": {"cpu": "M3"}}');

SELECT attributes -> 'specs'           FROM products;   -- JSON object
SELECT attributes ->> 'brand'          FROM products;   -- text: Acme
SELECT attributes #>> '{specs,cpu}'    FROM products;   -- nested path as text: M3

SELECT * FROM products WHERE attributes @> '{"brand": "Acme"}';      -- containment
SELECT * FROM products WHERE attributes ? 'ram_gb';                  -- key exists
SELECT * FROM products WHERE (attributes ->> 'ram_gb')::int >= 16;   -- cast and compare
SELECT * FROM products WHERE attributes @? '$.colors[*] ? (@ == "black")';  -- SQL/JSON path
```

#### Indexing

```sql
-- GIN index on the whole document: supports @>, ?, ?|, ?&
CREATE INDEX idx_products_attrs ON products USING GIN (attributes);

-- Smaller and faster for containment only
CREATE INDEX idx_products_attrs_path ON products USING GIN (attributes jsonb_path_ops);

-- B-tree expression index on one frequently filtered field
CREATE INDEX idx_products_brand ON products ((attributes ->> 'brand'));
```

#### When JSON columns are a good fit

- Attributes that vary by record type (product specs differ between laptops and shoes).
- User preferences and settings.
- Storing raw payloads from external APIs or webhooks.
- Event or audit data with varying shapes.
- Rapid prototyping, before a schema has settled.

#### When they're a bad fit

- Fields you **filter, join, sort, or aggregate on frequently**: promote them to real columns (with proper types, constraints, and statistics). The planner has little statistical knowledge about values inside JSON, which leads to poor estimates.
- Data needing **constraints or foreign keys** (you can add CHECK constraints on JSON, but it's awkward).
- **Large documents updated frequently:** updating one key rewrites the entire value (and in Postgres, the whole row version), which is expensive for big documents.
- Using JSON to avoid schema design altogether: you end up with an undocumented schema enforced nowhere.

A good hybrid: core, frequently queried fields as columns; variable, rarely queried details in a `jsonb` column. Generated columns can extract a JSON field into an indexed real column automatically.

---

## 10. Operations and Security

### 10.1 Connection Pooling

#### Why connections are expensive

Opening a database connection involves:

1. A TCP handshake (and DNS lookup).
2. A TLS handshake if encrypted (several more round trips plus cryptography).
3. Authentication (password hashing like SCRAM, or IAM token validation).
4. Server-side setup: **in Postgres, the server forks a new OS process for every connection**, each using several MB of memory (more as it caches catalog data and query plans).

This can take tens of milliseconds, which is often longer than the query itself.

#### Why too many connections hurt

- **Memory:** 2,000 connections × ~10 MB = 20 GB of RAM unavailable for caching data.
- **CPU contention:** far more active processes than CPU cores means constant context switching.
- **Internal contention:** each active connection competes for shared locks and structures (e.g., computing snapshots gets more expensive with more connections in Postgres).
- **Hard limits:** `max_connections` (Postgres default 100). Exceeding it returns errors like `FATAL: sorry, too many clients already`.

**Throughput doesn't increase with more connections.** Database throughput peaks at a fairly small number of active connections (roughly a few times the CPU core count) and then **declines** as contention grows. More connections just mean more waiting inside the database instead of in a queue outside it.

#### What a pool does

A connection pool keeps a set of open connections and lends them out:

1. The application requests a connection from the pool.
2. The pool hands over an idle one (or opens a new one up to a maximum, or makes the request wait).
3. The application runs its queries and returns the connection.
4. The connection stays open for the next request.

Benefits: no connection setup cost per request, a cap on concurrent database load, a queue that smooths traffic spikes, and protection of the database from overload.

#### Where pooling happens

**Application-side pools:** built into drivers and frameworks: HikariCP (Java), SQLAlchemy's pool (Python), `pg-pool` (Node.js), ActiveRecord pool (Rails). Each application instance has its own pool.

**The problem:** with 50 application instances × a pool of 20 = 1,000 database connections, even if each instance is mostly idle. Autoscaling multiplies this. Serverless platforms (AWS Lambda) are worse: each concurrent function invocation is a separate container with its own connection, so a traffic spike creates thousands of connections.

**External poolers** sit between all application instances and the database, multiplexing many client connections onto a few server connections:

- **PgBouncer:** a lightweight, single-process pooler for Postgres. Can accept thousands of client connections while using, say, 50 real database connections.
- **RDS Proxy:** AWS's managed proxy for RDS/Aurora (Postgres and MySQL). Benefits: managed and highly available; **especially useful for Lambda**; maintains connections during failover and redirects to the new primary, reducing failover disruption (AWS cites up to ~66% faster failover); integrates with **IAM authentication** and **Secrets Manager** so apps don't handle DB passwords.
- Others: Pgpool-II, Odyssey, PgCat, Supavisor (Postgres); ProxySQL (MySQL).

#### PgBouncer pooling modes

| Mode            | A server connection is assigned to a client... | Compatibility                                                                                                                                                                   | Efficiency                                                   |
| --------------- | ---------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------ |
| **Session**     | For the client's entire session                | Everything works                                                                                                                                                                | Lowest (no better than direct connections for long sessions) |
| **Transaction** | Only for the duration of a transaction         | Breaks session-level state: `SET` (use `SET LOCAL`), session advisory locks, `LISTEN/NOTIFY`, temp tables across transactions; prepared statements supported in PgBouncer 1.21+ | **High: the most common choice**                             |
| **Statement**   | Only for a single statement                    | Multi-statement transactions not allowed                                                                                                                                        | Highest, rarely used                                         |

In transaction mode, consecutive transactions from the same client may run on different server connections, so anything that relies on session state breaks.

#### Pool sizing

A widely cited starting formula (from the PostgreSQL wiki / HikariCP):

```
pool size ≈ (number of CPU cores × 2) + effective number of disks (spindles)
```

For a 16-core database server, roughly 30–40 active connections is a reasonable starting point for the **total** across all clients. Measure and tune from there; workloads that spend time waiting on I/O or locks may need somewhat more.

Also configure: connection timeout (how long a request waits for a free connection), idle timeout, max lifetime (recycle connections periodically), and validation/health checks. Set `idle_in_transaction_session_timeout` in Postgres so abandoned transactions don't hold connections and locks.

---

### 10.2 Network Security: Private Subnets, Security Groups, Bastion Hosts

The principle: **the database should never be directly reachable from the internet.** Only the systems that need it should be able to connect, and humans should go through controlled, audited paths.

#### VPC and subnets (AWS terminology; similar concepts exist in GCP and Azure)

- **VPC (Virtual Private Cloud):** your isolated network in the cloud.
- **Public subnet:** has a route to an **Internet Gateway**. Resources with public IPs here are reachable from the internet (subject to firewall rules). Typical residents: load balancers, NAT gateways, bastion hosts.
- **Private subnet:** **no route from the internet.** Resources can't be reached from outside, even if misconfigured. Outbound internet access, when needed, goes through a NAT gateway. Typical residents: application servers, **databases**, caches.
- For databases, use subnets in **multiple availability zones** (an RDS DB subnet group requires at least two) for high availability, and set `Publicly accessible = No`.

```
              Internet
                 │
        ┌────────▼────────┐  Public subnet
        │  Load balancer  │  Bastion host (optional)
        └────────┬────────┘
                 │
        ┌────────▼────────┐  Private subnet (app tier)
        │   App servers   │
        └────────┬────────┘
                 │ port 5432 only from app SG
        ┌────────▼────────┐  Private subnet (data tier)
        │  Database (RDS) │
        └─────────────────┘
```

#### Security groups

A **security group** is a virtual firewall attached to an instance or network interface.

- **Stateful:** if inbound traffic is allowed, the response is automatically allowed out.
- **Allow rules only** (default deny everything else).
- Rules can reference **other security groups** instead of IP addresses.

**Database security group example:**

| Direction               | Protocol/Port | Source           | Purpose                          |
| ----------------------- | ------------- | ---------------- | -------------------------------- |
| Inbound                 | TCP 5432      | `sg-app-servers` | Application access               |
| Inbound                 | TCP 5432      | `sg-bastion`     | Admin access through the bastion |
| (No rule for 0.0.0.0/0) |               |                  | Never open to the world          |

Referencing the app servers' security group means any instance in that group (including new ones from autoscaling) can connect, and nothing else can, without maintaining IP lists.

**Network ACLs (NACLs)** are a second, **stateless** layer at the subnet level, with both allow and deny rules (return traffic must be explicitly allowed). Useful for broad subnet-level blocks; security groups do most of the work.

#### Bastion host (jump box)

Engineers sometimes need direct database access (debugging, migrations, data fixes). Since the DB is in a private subnet, a **bastion host** in a public subnet acts as the single, hardened entry point:

```bash
# SSH tunnel: local port 5433 → bastion → database:5432
ssh -i key.pem -N -L 5433:mydb.xxxx.rds.amazonaws.com:5432 ec2-user@bastion-public-ip
psql -h localhost -p 5433 -U admin mydb
```

**Hardening a bastion:** allow SSH only from known IP ranges (office/VPN), use key-based authentication only (no passwords), minimal installed software, regular patching, session logging and auditing, MFA, and shut it down when not needed.

**Modern alternatives that avoid open SSH ports entirely:**

- **AWS Systems Manager Session Manager:** connect to an instance (and port-forward to the DB) using IAM permissions, with no inbound ports open and full session logging.
- **VPN** (AWS Client VPN, WireGuard, Tailscale) into the VPC.
- **EC2 Instance Connect Endpoint**, or identity-aware proxies (Teleport, Google IAP).

---

### 10.3 Monitoring

Monitoring catches problems before users notice and provides data for diagnosis.

#### What to monitor

**Query performance**

- Top queries by total time, mean time, and call count (`pg_stat_statements`).
- Slow query logs with plans (`auto_explain`).
- Sudden changes in a query's latency (often a plan change after statistics updates or data growth).

**Locks and blocking**

```sql
-- Who is blocking whom
SELECT pid,
       pg_blocking_pids(pid) AS blocked_by,
       state,
       wait_event_type,
       now() - xact_start AS xact_age,
       left(query, 80) AS query
FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pid)) > 0;

-- Long-running or idle-in-transaction sessions (hold locks, block vacuum)
SELECT pid, state, now() - xact_start AS age, left(query, 80)
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY xact_start
LIMIT 10;
```

Watch the deadlock count (`pg_stat_database.deadlocks`) and lock wait times.

**Replication**

- Replication lag (`pg_stat_replication.replay_lag`, RDS `ReplicaLag`).
- Replication slot WAL retention (an inactive slot can fill the disk):
  ```sql
  SELECT slot_name, active,
         pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained_wal
  FROM pg_replication_slots;
  ```

**Connections**

- Active vs idle vs idle-in-transaction connections compared with `max_connections`.
- Pool wait times and pool exhaustion on the application side.

**Cache efficiency**

- Buffer cache hit ratio: for OLTP, usually above ~99%. A drop suggests the working set no longer fits in memory.
  ```sql
  SELECT sum(blks_hit) * 100.0 / nullif(sum(blks_hit) + sum(blks_read), 0) AS hit_pct
  FROM pg_stat_database;
  ```

**Maintenance health**

- Dead tuples and last vacuum/analyze per table (`pg_stat_user_tables`).
- Table and index bloat.
- Transaction ID age (wraparound risk): `SELECT datname, age(datfrozenxid) FROM pg_database;`

**System resources**

- CPU, memory, disk I/O (IOPS, throughput, latency), **free storage space**, network.
- On AWS: CloudWatch metrics, **Performance Insights** (database load by wait event, query, host), Enhanced Monitoring (OS-level).

**Tools:** Datadog, New Relic, Grafana with Prometheus and `postgres_exporter`, pganalyze, pgwatch, Percona Monitoring and Management.

#### Alerting suggestions

- Free disk space below ~20%.
- Replication lag above an acceptable threshold (e.g., 30 seconds).
- Connections above ~80% of the maximum.
- Transactions running longer than a few minutes; idle-in-transaction sessions.
- Deadlock rate increase; error rate spikes.
- Transaction ID age approaching the autovacuum freeze limits.
- p95/p99 query latency regressions.

---

### 10.4 Backups and Point-in-Time Recovery (Additional Topic)

Replication protects against hardware failure but **not** against human error, bugs, or ransomware, because the replica faithfully copies the bad change. Backups are essential.

| Backup type                                           | Description                                | Pros                                              | Cons                                                          |
| ----------------------------------------------------- | ------------------------------------------ | ------------------------------------------------- | ------------------------------------------------------------- |
| **Logical** (`pg_dump`, `mysqldump`)                  | Exports SQL statements or data             | Portable across versions, selective (one table)   | Slow for large databases; restore requires rebuilding indexes |
| **Physical** (`pg_basebackup`, file/volume snapshots) | Copies data files                          | Fast for large databases                          | Same major version; whole cluster                             |
| **Continuous archiving + PITR**                       | Base backup plus continuously archived WAL | Restore to any second within the retention window | More storage and setup                                        |

**Point-in-time recovery:** restore the latest base backup before the incident, then replay WAL up to a target time, e.g., one second before someone ran `DELETE FROM orders` without a WHERE clause. Tools: pgBackRest, WAL-G, Barman. AWS RDS provides automated backups with PITR (retention up to 35 days) and manual snapshots.

**Best practices:**

- Follow the **3-2-1 rule:** 3 copies, 2 different media, 1 offsite (e.g., a different region or account).
- **Test restores regularly.** A backup you've never restored is a hope, not a backup. Measure how long a restore takes, which determines your real RTO.
- Encrypt backups and restrict access to them.
- Define RPO and RTO and design backup frequency around them.

---

### 10.5 Zero-Downtime Schema Migrations (Additional Topic)

Schema changes on large, busy tables can lock them and cause outages. Safe practices:

- **Expand and contract** (parallel change): add the new structure, deploy code that writes to both old and new, backfill existing data in batches, switch reads to the new structure, stop writing the old one, and finally drop it in a later release. Each step is backward compatible.
- **Adding a column:** fast in Postgres when nullable or with a constant default (Postgres 11+ doesn't rewrite the table). Volatile defaults (like `random()`) still rewrite it.
- **Adding NOT NULL:** add a `CHECK (col IS NOT NULL) NOT VALID` constraint, then `VALIDATE CONSTRAINT` (which doesn't block writes), then set NOT NULL.
- **Indexes:** `CREATE INDEX CONCURRENTLY` (and `DROP INDEX CONCURRENTLY`) to avoid blocking writes.
- **Foreign keys:** add with `NOT VALID`, then validate separately.
- **Renaming columns:** don't rename in place while old code is running; add a new column and migrate.
- **Changing column types** often rewrites the whole table; use a new column and backfill instead.
- **Backfills:** update in small batches (e.g., 1,000–10,000 rows) with pauses, to avoid long locks, replication lag, and WAL spikes.
- Always set `lock_timeout` (and often `statement_timeout`) for migration sessions.
- Tools: gh-ost and pt-online-schema-change (MySQL), pgroll and Reshape (Postgres), strong_migrations (Rails) to catch unsafe migrations.

---

### 10.6 SQL Injection

SQL injection happens when **untrusted input is concatenated into a SQL string**, letting an attacker change the query's structure. It remains one of the most common and damaging web vulnerabilities (OWASP Top 10, under "Injection").

```python
# VULNERABLE
email = request.args["email"]
query = f"SELECT * FROM users WHERE email = '{email}'"
```

Attacker input and resulting query:

| Input                                              | Resulting SQL                                 | Effect                                             |
| -------------------------------------------------- | --------------------------------------------- | -------------------------------------------------- |
| `' OR '1'='1`                                      | `... WHERE email = '' OR '1'='1'`             | Returns every user (authentication bypass)         |
| `'; DROP TABLE users; --`                          | `... WHERE email = ''; DROP TABLE users; --'` | Destroys data (if multiple statements are allowed) |
| `' UNION SELECT card_number, cvv FROM payments --` | Appends another query's results               | Exfiltrates other tables                           |

**Blind SQL injection:** even when results aren't displayed, attackers infer data from true/false behavior or timing (`AND pg_sleep(5)`).

#### Defenses

**1. Parameterized queries / prepared statements (the primary defense).**
The SQL structure is sent separately from the data; the database treats parameters strictly as values, never as code.

```python
# Python (psycopg)
cur.execute("SELECT * FROM users WHERE email = %s", (email,))
```

```javascript
// Node.js (pg)
await client.query("SELECT * FROM users WHERE email = $1", [email]);
```

```java
// Java (JDBC)
PreparedStatement ps = conn.prepareStatement("SELECT * FROM users WHERE email = ?");
ps.setString(1, email);
```

**2. ORMs and query builders** parameterize automatically, but **raw SQL escape hatches** (`.raw()`, `.extra()`, string-built `WHERE` fragments) are still vulnerable if you concatenate input.

**3. Allow-lists for identifiers.** Parameters only work for values, not for table names, column names, or `ASC/DESC`. For dynamic sorting:

```python
ALLOWED_SORT = {"name", "created_at", "price"}
column = sort if sort in ALLOWED_SORT else "created_at"
direction = "DESC" if order == "desc" else "ASC"
```

**4. Least privilege.** If the app's database user can't `DROP` tables or read the `payments` table, a successful injection does far less damage (Section 10.8).

**5. Defense in depth:** input validation (types, formats, lengths), web application firewalls, hiding detailed database error messages from users, and monitoring for suspicious queries. These help but **don't replace parameterization**.

**Second-order injection:** data stored safely earlier (e.g., a username containing SQL) is later concatenated unsafely into another query. Parameterize every query, not just ones handling direct user input.

---

### 10.7 Encryption

#### Encryption at rest

Protects data stored on disk, including data files, backups, snapshots, replicas, WAL, and logs, against stolen disks, improperly disposed hardware, or unauthorized access to storage.

- **Storage/volume encryption:** the storage layer encrypts everything transparently. On **AWS RDS**, encryption uses **KMS** keys (AWS-managed or customer-managed for control over rotation and access policies). Encryption must be enabled **when the instance is created**; to encrypt an existing unencrypted database, take a snapshot, copy it with encryption enabled, and restore from the copy. Read replicas and snapshots of an encrypted instance are encrypted too.
- **Transparent Data Encryption (TDE):** database-level encryption of data files (SQL Server, Oracle, MySQL Enterprise).
- **Limitation:** at-rest encryption doesn't protect against someone who can **query** the database (an attacker with stolen credentials or via SQL injection sees plaintext).

#### Encryption in transit

Protects data moving between the application and the database (and between replicas) from eavesdropping and tampering, using **TLS**.

- Postgres connection setting `sslmode`:
  - `disable`, `allow`, `prefer`: may not encrypt.
  - `require`: encrypts but doesn't verify the server's identity (vulnerable to man-in-the-middle).
  - `verify-ca`: verifies the certificate was signed by a trusted CA.
  - **`verify-full`:** also verifies the hostname matches the certificate. Recommended.
- Enforce TLS on the server side (RDS Postgres: parameter `rds.force_ssl = 1`; MySQL: `require_secure_transport = ON`).
- Even traffic inside a VPC should be encrypted (zero-trust principle; compliance requirements like PCI DSS).

#### Application/column-level encryption

Encrypt particularly sensitive fields (national IDs, bank details, health data) **in the application** before storing, so the database only holds ciphertext. Even DBAs, backups, and SQL injection reveal nothing useful without the keys.

- Use envelope encryption with a KMS: a data key encrypts the data, and the KMS master key encrypts the data key.
- **Tradeoffs:** you can't search, sort, or index encrypted values normally. Workarounds: store a **keyed hash (HMAC)** alongside for exact-match lookups, deterministic encryption (leaks equality), or tokenization services. Key management and rotation become your responsibility.
- Postgres `pgcrypto` provides in-database functions, but the key then passes through the database (visible in logs or `pg_stat_statements` if not careful).

#### Passwords are hashed, not encrypted

Store user passwords with slow, salted, one-way hashes: **Argon2id**, **bcrypt**, or scrypt. Never store them in plaintext or with reversible encryption, and never with fast hashes like MD5 or SHA-256 alone.

---

### 10.8 Least-Privilege Access

Every user, application, and service should have **only the permissions it needs, and nothing more**. This limits the blast radius of bugs, compromised credentials, and SQL injection.

#### Role design

| Role        | Permissions                                                              | Used by                                      |
| ----------- | ------------------------------------------------------------------------ | -------------------------------------------- |
| `app_rw`    | SELECT, INSERT, UPDATE, DELETE on application tables; USAGE on sequences | The application at runtime                   |
| `app_ro`    | SELECT only                                                              | Read-only services, reporting, read replicas |
| `analytics` | SELECT on specific tables or views (possibly with masked columns)        | BI tools, analysts                           |
| `migrator`  | DDL (CREATE, ALTER, DROP) on the schema                                  | CI/CD migration jobs only                    |
| `admin`     | Broad privileges                                                         | Rare break-glass use, audited                |

```sql
CREATE ROLE app_rw LOGIN PASSWORD '...';
GRANT CONNECT ON DATABASE shop TO app_rw;
GRANT USAGE ON SCHEMA app TO app_rw;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA app TO app_rw;
GRANT USAGE ON ALL SEQUENCES IN SCHEMA app TO app_rw;
ALTER DEFAULT PRIVILEGES IN SCHEMA app
  GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO app_rw;   -- future tables too

REVOKE CREATE ON SCHEMA public FROM PUBLIC;   -- default in Postgres 15+
```

#### Practices

- **Never** let applications connect as the superuser or the RDS master user.
- **Separate credentials per service**, so one compromised service doesn't expose others, and audit logs show which service did what.
- **Column- and row-level restrictions:** grant on specific columns (`GRANT SELECT (id, name) ON customers TO support`), use views to hide sensitive fields, and use Row-Level Security for tenant isolation (Section 12).
- **Secrets management:** store credentials in AWS Secrets Manager, HashiCorp Vault, or similar, with **automatic rotation**. Never hard-code them or commit them to repositories.
- **IAM database authentication** (RDS): applications obtain short-lived (15-minute) tokens using their IAM role instead of long-lived passwords.
- **Audit logging:** `pgaudit` (Postgres) or database activity streams to record who accessed or changed what, required for compliance (SOC 2, PCI DSS, HIPAA).
- **Regular access reviews:** remove unused roles and permissions; require approval for production data access.
- **Restrict network access** in addition to credentials (Section 10.2).

---

## 11. NoSQL Types and When to Use Them

### 11.1 The NoSQL Mindset: Model by Access Pattern

Relational modeling is **entity-first**: model the data's structure, normalize it, and write whatever queries you need later.

NoSQL modeling is **query-first**: list every access pattern the application needs ("get a user's last 20 orders," "get all items in an order"), then design the keys and structure so that each pattern is served by a single, efficient lookup. Data is often duplicated so each query reads from one place. The schema is effectively shaped by the queries, which makes reads very fast and scalable but makes new, unplanned queries hard.

### 11.2 Key-Value Stores

**Model:** a key maps to a value. The database usually treats the value as opaque (or offers limited structure). Operations are `GET key`, `PUT key value`, `DELETE key`.

**How they scale:** keys are hashed and distributed across nodes; every operation touches one node, so throughput scales almost linearly.

**Examples:** Redis and Memcached (in memory), DynamoDB (persistent, managed; also supports a document model), Riak, etcd (for configuration and coordination).

**Use cases:** caching, session storage, shopping carts, user preferences, feature flags, rate limiting, leaderboards (Redis), distributed locks.

**Limitations:** you can only look things up by key. Queries by value ("all users in Pune") require building and maintaining your own secondary lookup structures. No joins.

---

### 11.3 Document Databases

**Model:** each record is a **document** (JSON/BSON) with nested objects and arrays. Documents in one collection don't need identical fields. The database understands the document's structure, so you can query and index fields inside it.

```json
{
  "_id": "order_123",
  "customer": { "id": "c_9", "name": "Asha" },
  "status": "shipped",
  "items": [
    { "sku": "PEN-01", "qty": 2, "price": 20 },
    { "sku": "BOOK-7", "qty": 1, "price": 150 }
  ],
  "createdAt": "2026-11-20T10:15:00Z"
}
```

**Examples:** MongoDB, Couchbase, Firestore, Amazon DocumentDB, CouchDB.

**Modeling: embedding vs referencing**

- **Embed** data that's read together and belongs to the parent: order items inside the order, addresses inside a user. One read gets everything; updates to the document are atomic.
- **Reference** (store an ID) when the related data is large, shared by many documents, updated independently, or **unbounded** (a user's comments could grow forever; MongoDB documents are limited to 16 MB).

**MongoDB features:** rich query language, secondary and compound indexes, aggregation pipeline, multi-document ACID transactions (since 4.0), replica sets (automatic failover), and automatic sharding.

**Use cases:** content management, product catalogs with varying attributes, user profiles, mobile app backends, event data, rapid development where schemas evolve.

**Limitations:** joins (`$lookup`) exist but are less efficient than relational joins; duplicated (denormalized) data must be kept in sync; without discipline, "schemaless" turns into inconsistent data (use schema validation).

---

### 11.4 Wide-Column Stores

**Model:** data is organized into tables of rows, but each row is identified by a **partition key** that determines which node stores it, and within a partition, rows are sorted by **clustering columns**. Rows can have different sets of columns. Conceptually: a distributed, sorted map of `partition key → (clustering key → columns)`.

```sql
-- Cassandra (CQL)
CREATE TABLE messages_by_conversation (
  conversation_id UUID,
  sent_at         TIMEUUID,
  sender_id       UUID,
  body            TEXT,
  PRIMARY KEY ((conversation_id), sent_at)      -- partition key, clustering column
) WITH CLUSTERING ORDER BY (sent_at DESC);

-- Efficient: one partition, already sorted
SELECT * FROM messages_by_conversation WHERE conversation_id = ? LIMIT 50;
```

**Examples:** Apache Cassandra, ScyllaDB, HBase, Google Bigtable.

**Cassandra characteristics:**

- **Leaderless** replication with consistent hashing; every node is equal, no single point of failure, multi-datacenter support built in.
- **Tunable consistency** per query: `ONE`, `QUORUM`, `LOCAL_QUORUM`, `ALL`.
- **LSM-tree storage** (Section 2.4): extremely high write throughput.
- **One table per query pattern:** the same data is often written to several tables, each keyed for a different query (`messages_by_conversation`, `messages_by_user`).
- **Deletes create tombstones** (markers) that are removed only during compaction after a grace period; many deletes can slow reads.
- Avoid unbounded partitions (e.g., all events for a device forever); bucket by time: `PRIMARY KEY ((device_id, day), ts)`.

**Use cases:** messaging, IoT sensor data, activity feeds, logging, time-ordered event data, any workload with massive writes and known query patterns.

**Limitations:** no joins, limited ad hoc querying (queries generally must specify the partition key), secondary indexes are limited, aggregation is weak, eventual consistency requires care.

---

### 11.5 Graph Databases

**Model:** **nodes** (entities) and **edges** (relationships), both with properties. Relationships are first-class and stored as direct pointers between nodes ("index-free adjacency"), so following a relationship is a constant-time hop, rather than an index lookup or join.

```cypher
// Neo4j Cypher: friends of friends whom Asha doesn't already know
MATCH (a:Person {name: 'Asha'})-[:FRIEND]->(:Person)-[:FRIEND]->(fof:Person)
WHERE NOT (a)-[:FRIEND]->(fof) AND fof <> a
RETURN fof.name, count(*) AS mutual_friends
ORDER BY mutual_friends DESC;
```

The equivalent SQL needs self-joins for each hop, and the cost grows quickly with depth; a graph database's cost depends on the part of the graph explored, not the total data size.

**Examples:** Neo4j, Amazon Neptune, ArangoDB (multi-model), JanusGraph, Memgraph.

**Use cases:** social networks, recommendation engines ("people who bought X also bought Y"), fraud detection (rings of accounts sharing devices, addresses, cards), knowledge graphs, network and IT infrastructure dependency mapping, access control (who can reach which resource through which groups), route finding.

**Limitations:** less suited to bulk aggregations over all records or simple high-volume CRUD; horizontal scaling (sharding a graph) is difficult because relationships cross partitions; smaller ecosystem.

---

### 11.6 Time-Series Databases

**Model:** data points of the form `(timestamp, series identifiers/tags, values)`, arriving continuously and mostly appended, rarely updated. Queries are almost always time-range aggregations.

**Optimizations:**

- Storage partitioned by time chunks (fast range queries; old chunks dropped cheaply).
- Specialized compression (delta encoding of timestamps, Gorilla compression of floats) achieving 10× or more.
- **Downsampling / continuous aggregates:** automatically roll up raw per-second data into per-minute or per-hour summaries.
- **Retention policies:** automatically delete raw data older than N days.
- Time-oriented functions: `time_bucket`, gap filling, interpolation, moving averages.

**Examples:** TimescaleDB (a Postgres extension, so you keep SQL and joins), InfluxDB, Prometheus (metrics monitoring, pull-based), QuestDB, Amazon Timestream, ClickHouse (often used for this as well).

**Use cases:** infrastructure and application metrics, IoT sensor readings, financial ticks, energy usage, application analytics.

---

### 11.7 Summary: Choosing a NoSQL Type

| Type        | Data model                                | Strengths                                             | Weaknesses                     | Typical use cases                     |
| ----------- | ----------------------------------------- | ----------------------------------------------------- | ------------------------------ | ------------------------------------- |
| Key-value   | Key → value                               | Simplicity, speed, scale                              | Key-only access                | Cache, sessions, carts, rate limiting |
| Document    | Nested JSON documents                     | Flexible schema, natural object mapping, rich queries | Weak joins, duplication        | Catalogs, CMS, profiles, app backends |
| Wide-column | Partition key + sorted clustering columns | Huge write throughput, multi-DC availability          | Rigid query patterns, no joins | Messaging, IoT, feeds, logs           |
| Graph       | Nodes and edges                           | Deep relationship traversal                           | Bulk analytics, sharding       | Social, recommendations, fraud        |
| Time-series | Timestamped measurements                  | Compression, time-range aggregations, retention       | General-purpose use            | Metrics, monitoring, sensors          |

---

### 11.8 DynamoDB in Depth

DynamoDB is AWS's fully managed, serverless NoSQL database (key-value and document). It offers consistent single-digit-millisecond latency at virtually any scale, with no servers to manage, automatic partitioning and replication across three availability zones, and built-in encryption and backups.

#### Core concepts

- **Table:** a collection of items.
- **Item:** a record (like a row); a set of attributes. Maximum size **400 KB** (store larger objects in S3 and keep a pointer).
- **Attribute:** a name-value pair. Types: string, number, binary, boolean, null, list, map, and sets. Items in the same table can have different attributes; only the key attributes are required.

#### Primary keys

**Simple primary key (partition key only)**

- The **partition key (PK)** value is hashed to choose the physical partition where the item lives.
- Must uniquely identify each item.
- Access: get an item by its exact key.
- Example: `Users` table with PK = `user_id`.

**Composite primary key (partition key + sort key)**

- Items with the same partition key form an **item collection**, stored together on the same partition and **sorted by the sort key (SK)**.
- The combination (PK, SK) must be unique.
- Enables efficient **range queries within a partition**: `=`, `<`, `<=`, `>`, `>=`, `BETWEEN`, `begins_with` on the sort key.

```
PK               SK                         Attributes
CUSTOMER#c9      PROFILE                    name=Asha, email=...
CUSTOMER#c9      ORDER#2026-10-02#o81       total=499, status=delivered
CUSTOMER#c9      ORDER#2026-11-20#o95       total=150, status=shipped
CUSTOMER#c9      ADDRESS#home               city=Pune, ...
```

Query: `PK = "CUSTOMER#c9" AND begins_with(SK, "ORDER#")` returns all of the customer's orders sorted by date. Use `ScanIndexForward = false` for newest first, plus `Limit` for "last N orders."

#### Read and write operations

| Operation                                 | Description                                                                                                        | Efficiency                                             |
| ----------------------------------------- | ------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------ |
| `GetItem`                                 | Fetch one item by its full primary key                                                                             | Most efficient                                         |
| `BatchGetItem`                            | Up to 100 items by key in one call                                                                                 | Efficient                                              |
| `Query`                                   | All items in **one partition key**, optionally filtered by a sort key condition; sorted; paginated (1 MB per page) | Efficient, the main workhorse                          |
| `Scan`                                    | Reads **every item** in the table or index                                                                         | Expensive and slow on large tables; avoid in hot paths |
| `PutItem` / `UpdateItem` / `DeleteItem`   | Single-item writes                                                                                                 | Efficient                                              |
| `BatchWriteItem`                          | Up to 25 puts/deletes (not atomic as a group)                                                                      | Efficient                                              |
| `TransactWriteItems` / `TransactGetItems` | ACID transactions across up to 100 items, possibly in multiple tables                                              | Costs **twice** the capacity                           |

**Filter expressions** are applied **after** items are read, so they reduce the data returned but **not** the capacity consumed. Design keys so queries don't need large filters.

**Condition expressions** make writes conditional, which enables **optimistic locking** and uniqueness:

```
UpdateItem
  Key: { PK: "PRODUCT#1", SK: "STOCK" }
  UpdateExpression: "SET stock = stock - :one, version = version + :one"
  ConditionExpression: "stock > :zero AND version = :v"
→ fails with ConditionalCheckFailedException if the condition doesn't hold
```

`attribute_not_exists(PK)` on a PutItem prevents overwriting an existing item (insert-if-absent).

**Read consistency:**

- **Eventually consistent reads** (default): may return slightly stale data (typically consistent within about a second); cost half as much.
- **Strongly consistent reads:** reflect all prior successful writes; available on tables and LSIs, **not** on GSIs.

#### Secondary indexes

Because you can only query efficiently by the primary key, secondary indexes provide **alternate keys** for other access patterns. DynamoDB maintains them automatically (copying items into the index).

**Global Secondary Index (GSI)**

- A **completely different partition key** (and optionally sort key) from the base table. "Global" because queries on it span all of the table's partitions.
- Can be **added or deleted at any time**.
- **Eventually consistent reads only** (updates propagate asynchronously from the table).
- Has **its own provisioned throughput** (in provisioned mode), separate from the table.
- Key values don't need to be unique.
- **Sparse indexes:** only items that have the GSI's key attributes appear in the index. Useful for indexing a subset, e.g., only items with an `open_ticket_status` attribute.
- **Projections:** choose which attributes are copied into the index (`KEYS_ONLY`, `INCLUDE` specific attributes, or `ALL`). Smaller projections cost less storage and write capacity; but if a query needs a non-projected attribute, you must fetch it from the table separately.
- **Back-pressure warning:** if a GSI lacks write capacity, writes to the **base table are throttled**. Give GSIs adequate capacity and well-distributed keys.
- Default limit: 20 GSIs per table.

**Local Secondary Index (LSI)**

- **Same partition key** as the table, **different sort key**. "Local" because it's scoped to items within the same partition key.
- Must be **created when the table is created**; can't be added later.
- Supports **strongly consistent reads**.
- **Shares** the table's throughput.
- Imposes a **10 GB limit per partition key value** (the item collection, including LSI data).
- Limit: 5 per table.
- Example: table PK = `customer_id`, SK = `order_date`; LSI SK = `order_total` lets you query a customer's orders sorted by amount.

|                  | GSI                      | LSI                            |
| ---------------- | ------------------------ | ------------------------------ |
| Partition key    | Any attribute            | Same as table                  |
| Sort key         | Any attribute (optional) | Different attribute (required) |
| Created          | Anytime                  | Table creation only            |
| Read consistency | Eventual only            | Eventual or strong             |
| Throughput       | Separate                 | Shared with table              |
| Size limit       | None                     | 10 GB per partition key value  |
| Max per table    | 20 (default, adjustable) | 5                              |

In practice, **GSIs are far more commonly used**; choose an LSI only when you need strongly consistent reads on an alternate sort order and are sure the 10 GB limit won't be a problem.

#### Single-table design

Because DynamoDB has no joins, fetching related entities from multiple tables means multiple round trips. **Single-table design** stores multiple entity types in **one table**, using generic key names (`PK`, `SK`, `GSI1PK`, `GSI1SK`) and **key overloading** (the meaning of the key depends on the entity type), so that related items share a partition and can be retrieved in **one Query**.

**Example: e-commerce**

| Entity     | PK               | SK                     | GSI1PK             | GSI1SK      |
| ---------- | ---------------- | ---------------------- | ------------------ | ----------- |
| Customer   | `CUSTOMER#c9`    | `PROFILE`              | `EMAIL#asha@x.com` | `PROFILE`   |
| Order      | `CUSTOMER#c9`    | `ORDER#2026-11-20#o95` | `ORDER#o95`        | `ORDER#o95` |
| Order item | `ORDER#o95`      | `ITEM#PEN-01`          | `PRODUCT#PEN-01`   | `ORDER#o95` |
| Product    | `PRODUCT#PEN-01` | `METADATA`             |                    |             |

Access patterns served:

- Customer profile + all their orders → `Query PK = CUSTOMER#c9` (one request returns the profile item and order items).
- Customer's orders in a date range → `Query PK = CUSTOMER#c9 AND SK BETWEEN ORDER#2026-01 AND ORDER#2026-12`.
- All items in an order → `Query PK = ORDER#o95`.
- Customer by email → `Query GSI1 where GSI1PK = EMAIL#asha@x.com`.
- Orders containing a product → `Query GSI1 where GSI1PK = PRODUCT#PEN-01`.

**Design process:**

1. Understand the domain and draw an ER diagram.
2. **List every access pattern** with its parameters and expected frequency.
3. Design primary keys and GSIs so each pattern is a `GetItem` or `Query` (never a `Scan`).
4. Use sort key prefixes and hierarchical values (`COUNTRY#IN#STATE#MH#CITY#Pune`) for flexible `begins_with` queries.
5. Validate with sample data before building.

**Pros:**

- Fewer requests (related data fetched together, "pre-joined").
- Consistent performance at any scale.
- Fewer tables to configure, monitor, and provision (historically a big cost advantage in provisioned mode).

**Cons:**

- Steep learning curve; the table is hard to read for newcomers.
- **Inflexible:** new access patterns may require new GSIs or data migrations (backfilling attributes into existing items).
- Analytics and ad hoc queries are difficult; export to S3 and query with Athena, or stream into a warehouse.
- All entities share the table's settings (streams, backups, capacity).

**Multi-table design** is reasonable when access patterns are simple, still changing, or entities are rarely fetched together. Many teams use single-table design for tightly related entities within one service rather than for the entire application.

#### Capacity modes and units

**Capacity units:**

- **1 RCU (Read Capacity Unit)** = one strongly consistent read per second of an item up to **4 KB**, or **two** eventually consistent reads per second. Transactional reads cost 2 RCUs per 4 KB.
- **1 WCU (Write Capacity Unit)** = one write per second of an item up to **1 KB**. Transactional writes cost 2 WCUs per 1 KB.
- Sizes round up: reading a 9 KB item strongly consistently costs 3 RCUs; writing a 2.5 KB item costs 3 WCUs. **Updating any attribute costs WCUs based on the full item size**, which is a reason to keep frequently updated items small.
- Each GSI write consumes WCUs for the index too.

**On-demand mode:**

- Pay per request (read request units and write request units). No capacity planning.
- Scales instantly to accommodate traffic, including sudden spikes (it accommodates up to double the previous peak instantly, and adapts beyond that over time).
- Best for unpredictable, spiky, or new workloads, dev/test environments, and low-traffic tables.

**Provisioned mode:**

- You specify RCUs and WCUs; you pay for provisioned capacity per hour whether used or not.
- **Auto scaling** adjusts capacity based on utilization targets, but reacts over minutes, so sudden spikes can be throttled. **Burst capacity** (unused capacity saved for up to 5 minutes) absorbs short spikes.
- **Reserved capacity** offers further discounts for long-term commitments.
- Cheaper for steady, predictable traffic.
- Switching between modes is allowed (with limits on how often).

Exceeding capacity results in **throttling** (`ProvisionedThroughputExceededException`); AWS SDKs retry with exponential backoff automatically.

#### Partitions and hot partitions

DynamoDB stores data in **partitions** (physical storage units on SSDs). Each partition holds up to about **10 GB** and supports up to about **3,000 RCUs and 1,000 WCUs per second**. DynamoDB splits partitions automatically as data or throughput grows.

A **hot partition** happens when a disproportionate share of traffic targets one partition key value (or a few). Even if the table's total capacity is high, requests to that one key are limited by a single partition's throughput, causing throttling.

**Common causes:**

- A low-cardinality partition key (`status`, `country`, `is_active`).
- A date as the partition key (`2026-11-23`): every write today hits the same partition.
- A celebrity user, a viral item, or one giant tenant.
- A GSI keyed on a low-cardinality attribute (which then throttles base table writes).

**Solutions:**

- **High-cardinality partition keys** (user ID, order ID, device ID) that spread requests evenly.
- **Write sharding:** add a suffix to spread a hot logical key across multiple partition key values.
  - Random suffix: `DATE#2026-11-23#7` (0–9). Writes spread evenly; to read the whole day, query all 10 suffixes in parallel and merge.
  - Calculated suffix: `hash(order_id) % 10`, so a specific item can still be located directly.
- **Caching hot reads** with **DAX (DynamoDB Accelerator)**, an in-memory write-through cache with microsecond read latency and a DynamoDB-compatible API, or with ElastiCache.
- **Composite keys** combining dimensions (`tenant#region`).
- **Adaptive capacity:** DynamoDB automatically gives hot partitions more of the table's unused throughput and can isolate frequently accessed items onto their own partitions ("split for heat"). It reduces throttling but doesn't fix a fundamentally bad key, since a single partition key value's throughput still has an upper bound.
- **CloudWatch Contributor Insights** shows the most accessed and most throttled keys.

#### Other important DynamoDB features

- **DynamoDB Streams:** a time-ordered log of item-level changes (retained 24 hours), commonly consumed by **Lambda** for CDC: updating search indexes, aggregates, sending notifications, replicating data. Kinesis Data Streams integration is also available.
- **TTL:** set an attribute with an expiry timestamp (epoch seconds); DynamoDB deletes expired items automatically at no write cost (typically within a few days of expiry, so filter out expired items in queries if precision matters). Deletions appear in Streams. Great for sessions, temporary tokens, and old logs.
- **Global Tables:** multi-region, multi-active replication; each region accepts reads and writes, with last-writer-wins conflict resolution. Used for low-latency global apps and disaster recovery.
- **Backups:** on-demand backups and **point-in-time recovery** (restore to any second in the last 35 days).
- **Export to S3 / import from S3** for analytics and migrations without consuming table capacity.
- **PartiQL:** a SQL-compatible query language over DynamoDB (doesn't change the underlying access rules; a query without a key condition is still a scan).
- **Security:** IAM policies down to the item and attribute level (fine-grained access control using leading keys), encryption at rest by default, VPC endpoints.

#### When DynamoDB is (and isn't) a good fit

**Good fit:** known, stable access patterns; high scale and need for consistent low latency; serverless architectures (pairs naturally with Lambda, no connection pooling issues since it's an HTTP API); key-value lookups, user sessions, shopping carts, gaming state, IoT, metadata stores.

**Poor fit:** heavy ad hoc querying, complex reporting and aggregations, many-to-many relationships queried in many ways, workloads requiring complex multi-entity transactions, or teams still discovering access patterns.

---

## 12. Additional Postgres Topics

### 12.1 Postgres Index Types

Postgres has several index access methods, each suited to different data and operators.

#### B-tree (default)

```sql
CREATE INDEX idx_orders_created ON orders (created_at);
```

- Supports `=`, `<`, `<=`, `>`, `>=`, `BETWEEN`, `IN`, `IS NULL`, sorting (`ORDER BY`), and prefix matching (`LIKE 'abc%'` with the right collation or `text_pattern_ops`).
- The right choice for the vast majority of indexes. Supports unique indexes, multicolumn indexes, `INCLUDE` columns, and deduplication of repeated values (Postgres 13+).

#### Hash

- Equality (`=`) only. Can be smaller than B-tree for long values like URLs. Crash-safe since Postgres 10. Rarely chosen, since B-trees handle equality nearly as well and support much more.

#### GIN (Generalized Inverted Index)

**Concept:** an inverted index, like the index at the back of a book. For each **element** contained in a value, it stores the list of rows containing that element. Ideal when **one column value contains many elements** and you search for rows containing particular elements.

**Use cases:**

- **JSONB:** containment (`@>`), key existence (`?`, `?|`, `?&`).
  ```sql
  CREATE INDEX idx_events_data ON events USING GIN (data);
  SELECT * FROM events WHERE data @> '{"type": "purchase", "country": "IN"}';
  ```
- **Full-text search:** each lexeme in a `tsvector` maps to the rows containing it.
  ```sql
  CREATE INDEX idx_docs_fts ON docs USING GIN (to_tsvector('english', body));
  ```
- **Arrays:** `tags @> ARRAY['postgres']`, overlap `&&`.
- **Trigram search** with `pg_trgm`: makes `LIKE '%term%'`, `ILIKE`, and similarity searches fast.
  ```sql
  CREATE EXTENSION pg_trgm;
  CREATE INDEX idx_users_name_trgm ON users USING GIN (name gin_trgm_ops);
  SELECT * FROM users WHERE name ILIKE '%sharma%';
  ```

**Tradeoffs:** very fast lookups; **slower to build and update** (inserting one row with many elements updates many index entries). The `fastupdate` option buffers new entries in a pending list and merges them in bulk (faster writes, slightly slower searches until merged). GIN indexes can be large.

#### GiST (Generalized Search Tree)

**Concept:** a balanced tree framework where each internal node stores a **summary** (such as a bounding box or range) that covers all entries beneath it. Searches descend only into subtrees whose summary could match. It's "generalized" because different data types plug in their own logic.

**Use cases:**

- **Geometric and geographic data** (PostGIS): "points within this polygon," "shapes that intersect."
- **Range types:** overlap (`&&`), containment (`@>`) for `tstzrange`, `daterange`, `int4range`.
- **Nearest-neighbor (KNN) searches:** `ORDER BY location <-> point LIMIT 10` returns the closest items efficiently.
- **Exclusion constraints:** prevent overlapping bookings (Section 1.5).
- Full-text search (smaller and faster to update than GIN, but slower to query and can produce false positives that need rechecking).

```sql
CREATE INDEX idx_stores_location ON stores USING GIST (location);
SELECT name FROM stores ORDER BY location <-> ST_MakePoint(73.85, 18.52) LIMIT 5;
```

**SP-GiST** (space-partitioned GiST) supports non-balanced structures like quadtrees, k-d trees, and radix tries; good for data that partitions naturally (points, IP addresses, phone-number prefixes).

#### BRIN (Block Range Index)

**Concept:** instead of indexing every row, BRIN stores a **summary (typically min and max) for each range of physical table blocks** (default 128 pages per range). A query checks the summaries and skips block ranges that can't contain matching values.

```
Block range   min(created_at)        max(created_at)
0–127         2026-01-01 00:00       2026-01-03 14:20
128–255       2026-01-03 14:20       2026-01-06 02:11
...
Query: created_at in [2026-01-04, 2026-01-05] → read only block range 128–255
```

**Requirements:** works well only when the column's values **correlate with physical row order**, which is naturally true for append-only data: timestamps of logs, events, IoT readings, and sequential IDs. If rows are randomly ordered, every range spans everything and BRIN is useless.

**Tradeoffs:**

- **Tiny:** often kilobytes to a few megabytes, where a B-tree on a billion-row table might be tens of gigabytes.
- Very cheap to maintain on inserts.
- **Lossy:** it identifies candidate block ranges, which must then be scanned and rechecked, so it's slower than a B-tree for selecting a few specific rows. Best for range queries returning many rows on huge tables.
- New ranges may need summarizing (autovacuum handles it, or `autosummarize = on`).

```sql
CREATE INDEX idx_logs_created_brin ON logs USING BRIN (created_at);
```

#### Summary

| Index   | Best for                                                 | Key operators             | Size   | Write cost |
| ------- | -------------------------------------------------------- | ------------------------- | ------ | ---------- |
| B-tree  | General equality, ranges, sorting                        | `= < > BETWEEN LIKE 'x%'` | Medium | Moderate   |
| Hash    | Equality only                                            | `=`                       | Medium | Moderate   |
| GIN     | Multi-element values: JSONB, arrays, full-text, trigrams | `@> ? && @@`              | Large  | High       |
| GiST    | Spatial, ranges, nearest neighbor, exclusion constraints | `&& @> <->`               | Medium | Moderate   |
| SP-GiST | Naturally partitionable data (points, IPs, prefixes)     | Similar to GiST           | Medium | Moderate   |
| BRIN    | Huge, naturally ordered tables (time-series, logs)       | Ranges                    | Tiny   | Very low   |

---

### 12.2 VACUUM, ANALYZE, Table Bloat, and pg_stat_statements

#### Why VACUUM exists

Recall MVCC (Section 4.4): Postgres never overwrites a row in place. Each UPDATE creates a new row version and marks the old one dead; each DELETE marks the row dead. Dead tuples stay in the table and its indexes until they're cleaned up. Without cleanup:

- Tables and indexes keep growing (**bloat**).
- Scans read more pages full of dead data.
- Index-only scans become ineffective (pages aren't marked all-visible).
- Eventually, **transaction ID wraparound** threatens the database.

**VACUUM** is the garbage collector.

#### What VACUUM does

1. **Removes dead tuples** that are no longer visible to _any_ running transaction, and the index entries pointing to them.
2. **Marks the space as free** in the free space map, so future inserts and updates can reuse it. (Plain VACUUM usually doesn't shrink the file on disk, except for empty pages at the very end of the table.)
3. **Updates the visibility map**, marking pages where all tuples are visible to everyone, which enables index-only scans and lets future vacuums skip those pages.
4. **Freezes old tuples** to prevent transaction ID wraparound (below).

Plain `VACUUM` runs **concurrently** with normal reads and writes (it takes a lock that doesn't block DML).

#### VACUUM FULL

- Rewrites the entire table into a new, compact file and rebuilds all indexes, **returning space to the operating system**.
- Takes an **ACCESS EXCLUSIVE lock** for its entire duration: the table can't be read or written. On a large table, that can mean a long outage.
- Needs extra disk space for the new copy.
- In production, prefer **`pg_repack`** (or `pg_squeeze`), which rebuilds the table online with only brief locks.

#### ANALYZE

Collects statistics about column value distributions (Section 3.8) used by the query planner. Needed after significant data changes. `VACUUM ANALYZE` does both. ANALYZE samples rows, so it's fairly cheap even on large tables.

#### Autovacuum

A background process that automatically runs VACUUM and ANALYZE on tables when enough rows have changed.

A table is vacuumed when:

```
dead tuples > autovacuum_vacuum_threshold + autovacuum_vacuum_scale_factor × table rows
                     (default 50)                 (default 0.2 = 20%)
```

Similar thresholds (default 10%) trigger ANALYZE. Postgres 13+ also triggers vacuum based on inserts.

**The problem with defaults on large tables:** for a 500-million-row table, 20% means waiting for 100 million dead tuples before vacuuming. By then the table is heavily bloated and the vacuum takes a long time. **Tune per table:**

```sql
ALTER TABLE orders SET (
  autovacuum_vacuum_scale_factor = 0.01,     -- vacuum after ~1% changes
  autovacuum_analyze_scale_factor = 0.005
);
```

Other settings: `autovacuum_max_workers`, `autovacuum_naptime`, and cost limits (`autovacuum_vacuum_cost_limit`) that throttle vacuum's I/O. On fast storage, raising the cost limit lets autovacuum keep up.

**Never disable autovacuum.** If it seems to cause load, tune it to run more often (smaller, faster batches), not less.

#### Transaction ID wraparound

- Postgres transaction IDs are **32-bit**, about 4.2 billion values, compared using modular arithmetic: roughly 2 billion IDs are "in the past" and 2 billion "in the future" relative to the current one.
- If an old tuple's `xmin` falls more than ~2 billion transactions behind, it would suddenly appear to be "in the future" and become invisible, which is effectively data loss.
- **Freezing** prevents this: VACUUM marks sufficiently old tuples as frozen, meaning "visible to everyone, forever," so their XID no longer matters.
- Autovacuum launches **anti-wraparound vacuums** automatically when a table's oldest unfrozen XID age passes `autovacuum_freeze_max_age` (default 200 million). These can't be skipped.
- If freezing falls too far behind, Postgres eventually **refuses new write transactions** to protect data, requiring manual intervention. This is a well-known cause of serious outages.
- Monitor: `SELECT relname, age(relfrozenxid) FROM pg_class WHERE relkind = 'r' ORDER BY 2 DESC LIMIT 10;`

#### Table bloat

**Bloat** = space in tables and indexes occupied by dead tuples or empty space beyond what's needed for live data. A table with 10 GB of live data might occupy 40 GB.

**Causes:**

- Autovacuum not running often enough or throttled too much.
- **Long-running transactions and idle-in-transaction sessions:** VACUUM can only remove dead tuples older than the oldest snapshot still in use. One forgotten open transaction (or a long analytics query) prevents cleanup across the entire database. Also: abandoned replication slots, old prepared transactions, and `hot_standby_feedback` on replicas running long queries.
- High-churn update patterns (queue tables, counters, frequently updated status columns).
- Massive deletes (e.g., deleting old data instead of dropping partitions).

**Effects:** more disk usage, more I/O per query, larger backups, poorer cache efficiency, slower queries.

**Prevention and fixes:**

- Tune autovacuum per table; monitor `n_dead_tup` and `last_autovacuum` in `pg_stat_user_tables`.
- Kill or prevent long transactions: `idle_in_transaction_session_timeout`, `statement_timeout`, and monitoring of `pg_stat_activity`.
- Enable **HOT updates** by not indexing frequently updated columns and leaving free space in pages (`fillfactor = 80–90` on update-heavy tables).
- Use **partitioning** for time-based data and drop old partitions instead of deleting rows.
- Reclaim space with `pg_repack`; rebuild bloated indexes with `REINDEX INDEX CONCURRENTLY`.
- Estimate bloat with the `pgstattuple` extension or community bloat queries.

#### pg_stat_statements

An extension (bundled with Postgres; enabled by default on many managed services) that tracks execution statistics for every distinct normalized query. Literal values are replaced with placeholders, so `WHERE id = 5` and `WHERE id = 9` count as the same query.

**Setup:**

```sql
-- postgresql.conf: shared_preload_libraries = 'pg_stat_statements'  (restart needed)
CREATE EXTENSION pg_stat_statements;
```

**Key columns:** `query`, `calls`, `total_exec_time`, `mean_exec_time`, `max_exec_time`, `stddev_exec_time`, `rows`, `shared_blks_hit`, `shared_blks_read` (cache misses), `temp_blks_written` (spilling to disk).

```sql
-- Queries consuming the most total time
SELECT left(query, 100) AS query,
       calls,
       round(total_exec_time::numeric, 0) AS total_ms,
       round(mean_exec_time::numeric, 2)  AS mean_ms,
       round(100 * total_exec_time / sum(total_exec_time) OVER (), 1) AS pct_total,
       rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- Queries reading the most from disk (cache misses)
SELECT left(query, 100), shared_blks_read, calls
FROM pg_stat_statements ORDER BY shared_blks_read DESC LIMIT 10;

-- Reset statistics (e.g., after a deploy, to measure fresh)
SELECT pg_stat_statements_reset();
```

**How to use it:**

- Optimize by **total time** first: a 5 ms query called 1 million times per day (83 minutes of DB time) matters more than a 2-second query called twice.
- High `calls` with low mean time often reveals **N+1 patterns**.
- High `stddev` indicates plan instability or parameter-dependent performance.
- High `temp_blks_written` points to sorts or hashes exceeding `work_mem`.
- Compare snapshots over time to detect regressions after deploys.

---

### 12.3 Table Partitioning in Postgres

**Declarative partitioning** (Postgres 10+, much improved in 11–14) splits one large logical table into smaller physical **partitions**. Applications query the parent table normally; Postgres routes inserts to the correct partition and reads only the relevant partitions.

#### Partitioning methods

**Range partitioning** (most common: by time)

```sql
CREATE TABLE events (
  id          BIGINT GENERATED ALWAYS AS IDENTITY,
  tenant_id   INT NOT NULL,
  created_at  TIMESTAMPTZ NOT NULL,
  payload     JSONB,
  PRIMARY KEY (id, created_at)              -- must include the partition key
) PARTITION BY RANGE (created_at);

CREATE TABLE events_2026_10 PARTITION OF events
  FOR VALUES FROM ('2026-10-01') TO ('2026-11-01');
CREATE TABLE events_2026_11 PARTITION OF events
  FOR VALUES FROM ('2026-11-01') TO ('2026-12-01');

CREATE TABLE events_default PARTITION OF events DEFAULT;   -- catches rows matching no partition
```

Range bounds are inclusive at the lower end and exclusive at the upper end.

**List partitioning** (by discrete values)

```sql
CREATE TABLE customers (...) PARTITION BY LIST (region);
CREATE TABLE customers_apac PARTITION OF customers FOR VALUES IN ('IN', 'SG', 'JP');
CREATE TABLE customers_eu   PARTITION OF customers FOR VALUES IN ('DE', 'FR', 'NL');
```

**Hash partitioning** (even distribution when there's no natural range)

```sql
CREATE TABLE sessions (...) PARTITION BY HASH (user_id);
CREATE TABLE sessions_p0 PARTITION OF sessions FOR VALUES WITH (MODULUS 4, REMAINDER 0);
-- ... p1, p2, p3
```

**Sub-partitioning:** partitions can themselves be partitioned (e.g., by month, then by hash of tenant).

#### Benefits

1. **Partition pruning:** a query with a condition on the partition key skips irrelevant partitions entirely, at planning time or even at execution time (for parameters and joins).
   ```sql
   EXPLAIN SELECT count(*) FROM events WHERE created_at >= '2026-11-01';
   -- scans only events_2026_11 (and later partitions)
   ```
2. **Fast data retention:** removing a month of data is `DROP TABLE events_2025_01;` or `ALTER TABLE events DETACH PARTITION events_2025_01 CONCURRENTLY;`. It's instantaneous, creates no dead tuples or bloat, and generates almost no WAL, whereas `DELETE ... WHERE created_at < ...` on billions of rows is slow, I/O heavy, and leaves massive bloat.
3. **Smaller indexes per partition:** the hot (recent) partition's indexes fit in memory.
4. **Maintenance per partition:** VACUUM, REINDEX, and backups operate on smaller pieces; old partitions become effectively read-only and rarely need vacuuming.
5. **Tiered storage:** older partitions can be moved to cheaper tablespaces, or detached and archived (e.g., exported to S3).
6. **Bulk loading:** load data into a standalone table, index it, then `ATTACH PARTITION`.

#### Caveats

- **Queries without the partition key scan every partition** (each with its own index lookups), which can be slower than an unpartitioned table. Choose a partition key that most queries filter on.
- **Primary keys and unique constraints must include the partition key** columns, because uniqueness is enforced per partition. Global uniqueness on another column alone (e.g., `id`) isn't enforced across partitions.
- **Too many partitions hurt planning time and memory.** Hundreds are fine; many thousands become problematic. Choose sensible granularity (monthly or daily depending on volume).
- **Partitions must exist before data arrives** (or rows go to the default partition, or inserts fail). Automate creation with **`pg_partman`** or a scheduled job.
- Updating a row's partition key moves it between partitions (supported since Postgres 11, internally a delete + insert).
- **Converting an existing large table** into a partitioned one requires data migration (create a partitioned table, copy or attach data, and swap), which needs planning.
- **Partitioning isn't sharding:** all partitions live on the same server, sharing its CPU, memory, and disk. For multi-server scaling, use Citus or application-level sharding.

**When to partition:** very large tables (commonly hundreds of GB or more, or hundreds of millions of rows), time-series or event data with retention requirements, tables where queries naturally target a subset (recent data, one region), or when vacuum and index maintenance on one giant table has become painful. Don't partition small tables "just in case."

---

### 12.4 Soft Delete vs Hard Delete

#### Hard delete

The row is physically removed with `DELETE`.

- **Pros:** simple queries; storage stays lean; unique constraints behave naturally; satisfies privacy requirements to erase data.
- **Cons:** no undo; history lost unless captured elsewhere; cascading deletes can remove more than intended.

#### Soft delete

The row stays, marked as deleted with a flag or timestamp:

```sql
ALTER TABLE users ADD COLUMN deleted_at TIMESTAMPTZ;

UPDATE users SET deleted_at = now() WHERE id = 42;    -- "delete"
UPDATE users SET deleted_at = NULL  WHERE id = 42;    -- restore
```

A timestamp (`deleted_at`) is usually better than a boolean (`is_deleted`), since it records when the deletion happened and supports time-based purging.

**Pros:**

- Easy recovery from mistakes ("undo delete," trash bins).
- Preserves history for audit, analytics, and dispute resolution.
- Referential integrity: related records (an old order referencing a deleted product) still point to an existing row.

**Cons and pitfalls:**

- **Every query must exclude deleted rows** (`WHERE deleted_at IS NULL`). Forgetting this in one place causes bugs or data leaks (deleted users appearing in search, deleted items counted in reports). This is the biggest risk.
- **Unique constraints break:** a deleted user's email still occupies the unique index, so the person can't sign up again.
- **Cascading is manual:** soft-deleting a customer doesn't soft-delete their orders automatically.
- **Tables keep growing**, and indexes include deleted rows unless designed otherwise.
- **Privacy compliance:** regulations such as GDPR's "right to erasure" and India's DPDP Act require actually removing personal data on request. Soft deletion alone may not comply.

#### Making soft delete work well

```sql
-- Uniqueness only among active rows
CREATE UNIQUE INDEX uq_users_email_active ON users (email) WHERE deleted_at IS NULL;

-- Indexes focused on active rows
CREATE INDEX idx_users_active_created ON users (created_at) WHERE deleted_at IS NULL;

-- A view for application queries
CREATE VIEW active_users AS SELECT * FROM users WHERE deleted_at IS NULL;
```

- ORM default scopes (Rails `default_scope` or the `paranoia`/`discard` gems, Django custom managers, Hibernate `@SQLRestriction`), used carefully.
- Row-Level Security policies can hide soft-deleted rows automatically.
- **Purge job:** hard-delete or anonymize soft-deleted rows after a retention period (e.g., 30 or 90 days).

#### Alternative: hard delete with an archive

Delete the row from the main table and copy it to an archive or history table (via trigger, application code, or CDC):

```sql
WITH moved AS (DELETE FROM orders WHERE id = 42 RETURNING *)
INSERT INTO orders_archive SELECT *, now() AS archived_at FROM moved;
```

The main table stays clean with no query filters needed, while history is retained separately with its own retention rules. Other alternatives: temporal tables (system-versioned history), event sourcing.

|                    | Soft delete          | Hard delete  | Hard delete + archive          |
| ------------------ | -------------------- | ------------ | ------------------------------ |
| Recovery           | Easy                 | Backups only | From archive                   |
| Query complexity   | Every query filters  | Simple       | Simple                         |
| Unique constraints | Need partial indexes | Natural      | Natural                        |
| Table size         | Grows                | Lean         | Lean (archive grows)           |
| Privacy compliance | Needs purge          | Good         | Archive needs retention policy |

**Guideline:** use soft delete where users expect "undo" or where business records must be retained (orders, invoices, accounts), with a purge policy for personal data. Use hard delete for truly disposable data (sessions, tokens, temporary records). Consider the archive approach for large, high-traffic tables.

---

### 12.5 Row-Level Security (RLS)

**Row-Level Security** lets the database restrict **which rows** each user or session can see or modify, using **policies** attached to a table. The database automatically adds the policy conditions to every query on that table, so filtering doesn't depend on the application remembering to add a `WHERE` clause.

#### Why it matters

In a multi-tenant SaaS application where all tenants share tables, every query must include `WHERE tenant_id = ?`. One missing filter (in a new endpoint, a report, an ORM edge case) leaks one customer's data to another, which is a severe security incident. RLS provides **defense in depth**: even a buggy query can only see the current tenant's rows.

#### Basic setup

```sql
-- 1. Enable RLS on the table
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;

-- 2. Define a policy
CREATE POLICY tenant_isolation ON documents
  USING (tenant_id = current_setting('app.current_tenant')::int)
  WITH CHECK (tenant_id = current_setting('app.current_tenant')::int);

-- 3. The application sets the tenant for each transaction
BEGIN;
SET LOCAL app.current_tenant = '42';
SELECT * FROM documents;                 -- returns only tenant 42's rows
INSERT INTO documents (tenant_id, title) VALUES (7, 'x');   -- rejected by WITH CHECK
COMMIT;
```

- **`USING`**: a condition determining which **existing** rows are visible for SELECT and which rows can be targeted by UPDATE and DELETE.
- **`WITH CHECK`**: a condition that **new or modified** rows must satisfy for INSERT and UPDATE (prevents writing rows into another tenant).
- Policies can be specific to commands (`FOR SELECT`, `FOR INSERT`, `FOR UPDATE`, `FOR DELETE`, or `FOR ALL`) and to roles (`TO app_user`).
- Multiple **permissive** policies combine with OR; **restrictive** policies (`AS RESTRICTIVE`) combine with AND, useful for adding rules that must always hold.
- **Default deny:** once RLS is enabled, if no policy applies to a user, they see no rows.

#### Other examples

```sql
-- Users see only their own rows (user identity from a session setting)
CREATE POLICY own_rows ON notes
  USING (owner_id = current_setting('app.user_id')::uuid);

-- Managers can read their team's rows
CREATE POLICY team_read ON timesheets FOR SELECT
  USING (team_id IN (SELECT team_id FROM team_managers
                     WHERE manager_id = current_setting('app.user_id')::uuid));

-- Hide soft-deleted rows from everyone
CREATE POLICY hide_deleted ON orders AS RESTRICTIVE
  USING (deleted_at IS NULL);
```

Supabase builds its entire security model on RLS, using JWT claims (`auth.uid()`) inside policies, which allows clients to query the database through an API safely.

#### Important gotchas

- **Superusers and roles with `BYPASSRLS` ignore all policies.** The application must connect as a normal role.
- **Table owners bypass RLS by default.** If the application connects as the table owner, policies don't apply unless you run `ALTER TABLE documents FORCE ROW LEVEL SECURITY;`. A common mistake is testing RLS as the owner (or superuser) and concluding it works, or not realizing it's being bypassed.
- **Connection pooling:** with PgBouncer in transaction mode, a plain `SET` would leak to the next client using the same connection. Always use **`SET LOCAL`** (or `set_config('app.current_tenant', '42', true)`) inside the transaction so the setting resets at transaction end.
- **Performance:** the policy condition is added to every query; complex policies (especially with subqueries) can slow queries down. Index the policy columns (`tenant_id`), and wrap function calls so they're evaluated once (`(SELECT current_setting(...))`).
- **Unset settings:** `current_setting('app.current_tenant')` raises an error if not set; `current_setting('app.current_tenant', true)` returns NULL, which matches no rows (failing safely).
- **Side channels:** unique constraint violations or foreign key errors can reveal the existence of rows the user can't see. Views run with the owner's permissions unless created with `security_invoker = true` (Postgres 15+), which can bypass RLS unexpectedly.
- **Migrations and admin tasks** typically need a role that bypasses RLS; keep that role separate from the application's role.

**When to use RLS:** multi-tenant applications sharing tables, applications exposing the database to clients directly (Supabase, PostgREST), and regulated data where access must be enforced centrally. It complements, rather than replaces, authorization logic in the application.

---

## 13. Revision Summary

A condensed recap of the main ideas from each section, for quick review after studying the details above.

**Fundamentals**

- Relational databases give schema enforcement, joins, and ACID transactions; NoSQL databases trade some of those for horizontal scale, flexible schemas, or specialized data models. Choose based on access patterns, consistency needs, and scale.
- Normalization (1NF → 3NF/BCNF) removes redundancy and prevents update, insert, and delete anomalies. Denormalization deliberately adds redundancy for read speed and requires a sync strategy.
- Use surrogate primary keys plus UNIQUE constraints on natural keys; index foreign keys; prefer time-ordered IDs (UUIDv7) over random UUIDs for index locality.
- Constraints enforce integrity no matter which client writes; UNIQUE constraints are the reliable way to prevent duplicates under concurrency.
- OLTP = many small transactions on row stores; OLAP = large scans and aggregations on column stores.

**Indexing**

- Databases read fixed-size pages; performance depends on how many pages are read and whether they're cached.
- B+trees keep data only in linked leaves, giving high fanout (3–4 levels for billions of rows) and efficient range scans. LSM trees favor writes.
- Composite indexes follow the leftmost-prefix rule; put equality columns first and range/sort columns last. Covering indexes enable index-only scans.
- Indexes cost write performance, storage, and memory; they don't help low-selectivity queries or small tables.

**Query optimization**

- The planner is cost-based and relies on statistics; most bad plans come from bad row estimates.
- Read plans bottom-up; compare estimated vs actual rows; find the most expensive node.
- Nested loop (small outer + indexed inner), hash join (large unsorted, equality), merge join (pre-sorted inputs).
- Fix N+1 with joins or batched IN queries; use keyset pagination for deep pages; keep predicates sargable and types matched.

**Transactions**

- ACID: atomicity (undo), consistency (constraints), isolation (concurrency control), durability (WAL fsync).
- Anomalies: dirty read, non-repeatable read, phantom, lost update, write skew. Postgres defaults to Read Committed; its Repeatable Read is snapshot isolation (no write-skew protection); Serializable (SSI) needs retries.
- MVCC keeps row versions so readers and writers don't block each other; Postgres pays for it with VACUUM.
- WAL makes commits durable with sequential writes, and powers replication and PITR; checkpoints bound recovery time.
- 2PC gives atomic commits across systems but blocks if the coordinator fails.

**Concurrency**

- Writes take exclusive row locks; DDL takes strong table locks (use `lock_timeout` for migrations).
- Pessimistic locking (`FOR UPDATE`) for high contention; optimistic locking (version column) for low contention; atomic single-statement updates when possible.
- Prevent deadlocks with consistent lock ordering and short transactions; always retry deadlock and serialization errors.
- `FOR UPDATE SKIP LOCKED` builds safe concurrent job queues. Idempotency keys, unique constraints, and upserts make retries safe.

**Scaling**

- Scale cheaply first (queries, indexes, pooling, vertical, caching, replicas); shard last.
- Leader-follower is simplest; multi-leader and leaderless need conflict resolution; quorum W + R > N.
- Async replication causes lag; handle read-your-writes by reading from the leader or tracking LSNs.
- Shard keys need high cardinality, even distribution, and alignment with the dominant query. Consistent hashing with virtual nodes limits data movement when nodes change.
- Failover requires failure detection, promotion, and client redirection; prevent split brain with consensus and fencing.

**Distributed theory**

- CAP: during a partition, choose consistency or availability. PACELC: otherwise, choose latency or consistency.
- Consistency models range from linearizable to causal to eventual; stronger costs latency and availability.
- Sagas replace distributed transactions with local transactions and compensations; the outbox pattern solves dual writes; CDC streams changes from the transaction log.

**Caching**

- Cache-aside with delete-on-write plus TTL is the default pattern; write-through favors freshness; write-back favors write speed at the risk of loss.
- Stampede → single-flight/early refresh; penetration → cache nulls/Bloom filters; avalanche → TTL jitter/HA cache.
- Redis offers rich data structures, persistence, and HA; Memcached is a simple multi-threaded cache.

**Database objects**

- Views are stored queries; materialized views store results and need refresh strategies.
- Stored procedures and triggers put logic in the database: powerful but hidden and harder to maintain.
- CTEs structure complex queries; recursive CTEs traverse hierarchies; window functions compute rankings, running totals, and comparisons without collapsing rows.
- Postgres full-text search uses tsvector/tsquery with GIN indexes; JSONB stores semi-structured data with GIN indexing, but frequently queried fields belong in columns.

**Operations and security**

- Connection pooling (PgBouncer transaction mode, RDS Proxy) is essential because connections are expensive and throughput peaks at a modest number of active connections.
- Databases belong in private subnets, reachable only from application security groups; humans connect via bastion, SSM, or VPN.
- Monitor slow queries, locks, replication lag, connections, cache hit ratio, vacuum health, and disk space. Back up with PITR and test restores. Use expand-and-contract migrations.
- Parameterized queries prevent SQL injection; encrypt at rest and in transit (`verify-full`); grant least privilege with separate roles and managed secrets.

**NoSQL**

- Model by access pattern. Key-value for lookups, document for nested flexible records, wide-column for massive writes, graph for relationship traversal, time-series for metrics.
- DynamoDB: partition key decides placement; sort key enables range queries within a partition; GSIs add alternate keys (eventual consistency, anytime); LSIs add alternate sort keys (strong reads, creation time only, 10 GB limit). Single-table design pre-joins data; on-demand vs provisioned capacity; avoid hot partitions with high-cardinality keys and write sharding.

**Postgres extras**

- B-tree for general use; GIN for JSONB, arrays, full-text, trigrams; GiST for spatial, ranges, nearest-neighbor; BRIN for huge naturally ordered tables.
- VACUUM removes dead tuples, maintains the visibility map, and prevents XID wraparound; long transactions cause bloat; tune autovacuum per table; use `pg_stat_statements` to find costly queries by total time.
- Partitioning enables pruning and instant retention by dropping partitions, but unique keys must include the partition key.
- Soft delete needs partial unique indexes, filtering everywhere, and a purge policy; hard delete plus archive is often cleaner.
- RLS enforces row access in the database; remember owner/superuser bypass, `FORCE ROW LEVEL SECURITY`, and `SET LOCAL` with pooling.
