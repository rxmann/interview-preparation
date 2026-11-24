# Database Notes: NoSQL Types and When to Use Them

> Part of the [Database Notes](index.md) series.

---

## 11. NoSQL Types and When to Use Them

### The NoSQL mindset

Relational modeling is entity-first: design the data's structure, normalize it, write whatever queries you need later. NoSQL modeling is query-first: list every access pattern the application needs, then design keys and data structures so that each pattern is a single efficient lookup. Data is often duplicated across multiple structures — one for each query pattern. This makes known reads extremely fast but makes new unplanned queries difficult or impossible without schema changes.

### Key-value stores

The simplest model: a key maps to an opaque value. Operations are get, put, and delete. Scales easily by hashing keys across nodes — each operation touches exactly one node. Examples: Redis, DynamoDB, Memcached, etcd. Use for caching, sessions, rate limiting, feature flags, and leaderboards. The hard limit is that you can only look up data by key — querying by value requires building your own secondary structure.

### Document databases

Each record is a JSON-like document with nested objects and arrays. The database understands the document's structure, so you can query and index fields inside it. The core design decision is embed vs reference: embed data that's always read together and belongs to one parent (order items inside an order); reference data that's large, shared across many documents, or could grow unboundedly (a user's full comment history). Examples: MongoDB, Firestore, Couchbase. Use for product catalogs, CMS, user profiles, and app backends where schemas evolve. Joins exist but are weak — denormalization is the norm, and duplicated data must be kept in sync.

### Wide-column stores

Data is organized by partition key (which node stores it) and clustering columns (sorted order within that partition). The same data is often written to multiple tables, each keyed for a different query pattern. Examples: Cassandra, HBase, Bigtable. Use for messaging, IoT sensor data, activity feeds, and logs — anything with massive write throughput and predictable query patterns. The constraint is tight: queries generally must specify the partition key; ad hoc queries and range scans across partitions are expensive or impossible.

### Graph databases

Nodes and edges are both first-class with their own properties. Relationships are stored as direct pointers between nodes rather than index lookups, so traversing a relationship is O(1) regardless of data size. SQL would need a self-join for each hop, and cost grows with depth. Examples: Neo4j, Amazon Neptune. Use for social networks, recommendations, fraud ring detection, and access control hierarchies. Weak for bulk aggregations over all records, and sharding a graph is hard because relationships cross partition boundaries.

### Time-series databases

Optimized for (timestamp, tags, values) data that arrives continuously and is almost never updated. Features include specialized compression (delta-encoding timestamps, Gorilla compression for floats, often 10× smaller than a row store), automatic downsampling (roll raw per-second data into per-minute summaries), and retention policies (automatically delete data older than N days). Examples: TimescaleDB, InfluxDB, Prometheus, QuestDB. Use for infrastructure metrics, IoT sensors, financial ticks, and application analytics.

### DynamoDB in depth

DynamoDB is AWS's fully managed key-value and document store. The partition key determines which physical node stores the item; the sort key enables range queries within a partition. All items with the same partition key are co-located and sorted by sort key, so a single Query can fetch a customer's profile and all their orders together.

A GSI (global secondary index) provides an alternate partition and sort key for a different access pattern. It's eventually consistent, can be added at any time, and has its own write capacity budget. An LSI (local secondary index) must be defined at table creation, shares the table's throughput, supports strong reads, but imposes a 10 GB limit per partition key value.

Single-table design stores multiple entity types in one table using generic key names and prefixed values, so related data shares a partition and can be fetched in one Query call. It's efficient but hard to understand for newcomers and inflexible when new access patterns emerge.

Hot partitions — where most traffic targets one partition key — are the most common DynamoDB performance failure. Causes include low-cardinality keys, date-based keys, or celebrity items.

---

### Gotchas

- DynamoDB Scan reads every item in the table regardless of filters — always design access patterns as GetItem or Query, never Scan on hot paths.
- A GSI with insufficient write capacity throttles writes to the base table, not just to the index — size GSIs carefully and monitor their consumed capacity.
- Cassandra tombstones accumulate with every delete and slow down reads on affected partitions — heavy delete workloads cause compaction and read amplification problems.
- Document databases without schema validation produce inconsistently shaped documents across the collection — enforce structure in the application or use the database's built-in schema validation.
- Wide-column tables require a new table (and data duplication) to serve a new access pattern — query flexibility is sacrificed at design time, not discovered at runtime.
