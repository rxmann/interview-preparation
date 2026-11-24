# Database Notes: Concepts, Internals, and Design

> Learning-oriented notes. Each topic explains **what** the concept is, **why** it exists (the problem it solves), **how** it works underneath, and **what tradeoffs** come with it. Examples are mostly PostgreSQL, with MySQL/InnoDB, DynamoDB, Redis, and others where relevant.

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

#### First Normal Form (1NF)

1NF establishes the foundational rules of a relational table. A table is in 1NF when:

- Row order carries no hidden meaning.
- Every column enforces a **single, strict data type**.
- Every table has a **primary key** that uniquely identifies each row.
- Each row-column intersection holds a **single atomic value** — no lists or repeating column groups.

**The 4 ways to break 1NF:**

**1. Using row order to convey information**

Ordering the Beatles by height without a height column loses data because relational databases return rows in arbitrary order.

Fix: add an explicit column (`height_cm INT`).

**2. Mixing data types within a column**

Storing both `180` (integer) and `"180 cm"` or `"unknown"` (strings) in a `height_cm` column.

Fix: enforce a strict single type per column (integers only, NULLs for unknown).

**3. No primary key**

Without a PK on `Beatle`, the database allows duplicate contradictory rows for the same person.

Fix: define a primary key on every table.

**4. Repeating groups on a single row**

In a game inventory, either lumping items into a text string (`"arrows,shields,coins"`) or creating wide columns like `item_1, item_2, item_3` makes querying impossible (e.g., "who has more than 10 coins?").

```
Violates 1NF:   player_inventory(player, item_1, item_2, item_3, item_4)
Also violates:  player_inventory(player, items = "arrows,shields,coins")

Fixed:          player_inventory(player, item_type, quantity)
                PK: (player, item_type)
```

The original simpler framing: each cell holds one value, and there are no repeated columns like `phone1, phone2, phone3`.

```
Violates 1NF:   users(id, name, phones = "98100, 98200")
Fixed:          users(id, name)
                user_phones(user_id, phone)
```

Why: you can't index, constrain, or query individual values hidden inside a list.

---

#### Second Normal Form (2NF)

2NF governs how non-key attributes relate to a **composite primary key**.

**Core rule:** every non-key attribute must be functionally dependent on the **entire** primary key, not just part of it.

**Applies to tables with a composite key.** Tables with a single-column PK are automatically in 2NF once they satisfy 1NF.

**How partial dependency breaks 2NF:**

Take a valid 1NF `player_inventory(player, item_type, quantity, player_rating)` with PK `(player, item_type)`.

- `quantity` depends on the whole key `(player + item_type)` ✓
- `player_rating` depends only on `player` (part of the key) ✗ — **partial dependency**

Because `player_rating` is repeated across every item row for the same player, three anomalies appear:

| Anomaly | What goes wrong |
|---------|-----------------|
| **Deletion** | Deleting a player's last item also erases their `player_rating` |
| **Update** | Updating a player's rating in one row but not others leaves contradictory values |
| **Insertion** | A new player with no items yet cannot have a `player_rating` stored at all |

Fix: split into two tables.

```
players(player_id, player_rating)              PK: player_id
player_inventory(player_id, item_type, quantity)  PK: (player_id, item_type)
```

The classic order-items example:

```
order_items(order_id, product_id, quantity, product_name)
Key: (order_id, product_id)

quantity      depends on (order_id, product_id)   ✓
product_name  depends on product_id only          ✗ partial dependency
```

Fix: move `product_name` into a `products` table.

---

#### Third Normal Form (3NF)

3NF eliminates indirect dependencies **among non-key columns**.

**Core rule:** no non-key attribute may depend on another non-key attribute (no transitive dependencies).

Classic rule: every non-key attribute must depend on **the key** (1NF), **the whole key** (2NF), and **nothing but the key** (3NF).

**How transitive dependency breaks 3NF:**

In `players(player_id, player_skill_level, player_rating)`:

- Both `player_skill_level` and `player_rating` depend on `player_id` → passes 2NF.
- But `player_rating` is **determined by** `player_skill_level` (1–3 = Beginner, 4–6 = Intermediate, 7–9 = Advanced) → `player_id → player_skill_level → player_rating`.

Anomaly: if a player's skill level changes from 3 to 4, forgetting to also update `player_rating` leaves a contradictory row (skill 4 but still "Beginner").

Fix: extract the lookup into its own table.

```
players(player_id, player_skill_level)
skill_ratings(player_skill_level, player_rating)   PK: player_skill_level
```

The classic employee-department example:

```
employees(emp_id, name, dept_id, dept_name)
emp_id → dept_id → dept_name      ✗ dept_name depends on emp_id only *through* dept_id
```

Fix: `departments(dept_id, dept_name)`. Renaming a department now touches one row.

---

#### Boyce-Codd Normal Form (BCNF)

BCNF is a slightly stricter version of 3NF. It drops the qualifier "non-key" from the 3NF rule:

**Core rule:** for every functional dependency `X → Y` in the table, X must be a **candidate key** (a superkey).

In plain terms: every attribute (including key attributes) must depend on the key, the whole key, and nothing but the key.

**How 3NF can be satisfied while BCNF is broken:**

A table breaks 3NF-but-not-BCNF only when it has multiple overlapping candidate keys where a non-key attribute (or part of one candidate key) determines part of another candidate key.

Example: each instructor teaches exactly one course, and a student takes a course from exactly one instructor.

```
enrollments(student, course, instructor)
Candidate keys: (student, course), (student, instructor)
Dependency: instructor → course       ← instructor is not a key
```

This passes 3NF (instructor is part of a candidate key), but breaks BCNF (instructor is not a *superkey*). Problem: if the instructor switches courses, many rows need updating.

Fix: `instructors(instructor, course)` and `enrollments(student, instructor)`.

**Practical takeaway:** encountering a table that satisfies 3NF without also satisfying BCNF is exceedingly rare. By designing every table so that every attribute depends on "the key, the whole key, and nothing but the key," you will satisfy both 3NF and BCNF 99% of the time.

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

