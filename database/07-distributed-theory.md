# Database Notes: Distributed Data Theory

> Part of the [Database Notes](index.md) series.

---

## 7. Distributed Data Theory

### CAP theorem

CAP says a distributed system can guarantee at most two of: Consistency (every read returns the latest write), Availability (every non-failing node responds), and Partition tolerance (the system keeps working when nodes can't communicate). In practice, network partitions happen in any real distributed system, so partition tolerance is not optional. The real choice is: when a partition occurs, do you sacrifice consistency or availability?

CP systems (ZooKeeper, etcd, HBase, Spanner) refuse to serve stale data during a partition — some requests fail or block. AP systems (Cassandra, DynamoDB with default settings, DNS) keep responding but may return stale data, reconciling differences later.

### PACELC

CAP only describes behavior during partitions, which are relatively rare. PACELC extends it: even during normal operation, there is a tradeoff between Latency and Consistency. Getting strong consistency always requires coordination — multiple nodes must agree before a write is confirmed, costing network round trips. Serving from the nearest replica immediately is faster but potentially stale. This latency/consistency tradeoff happens on every request, not just during failures, which makes PACELC more useful for real design conversations.

### Consistency models

**Linearizability** is the strongest — the system looks like a single copy of the data, and all operations take effect at one instant in real time. Required for distributed locks, leader election, and account balances. Expensive because every read and write needs coordination.

**Causal consistency** preserves cause-and-effect order: if Alice posts a question and Bob replies, no one should see the reply before the question. Concurrent unrelated operations can be seen in different orders. It's the strongest model that remains available during partitions, making it attractive for geo-distributed systems.

**Eventual consistency** only guarantees that replicas converge eventually if no new writes occur. No timing guarantees, reads can return stale data. Highest availability and lowest latency, but requires conflict resolution strategies.

### Consensus

Consensus algorithms (Raft, Paxos) get a cluster of nodes to agree on a single value despite failures — used for leader election, log ordering, and distributed locks. A majority quorum of 2f+1 nodes tolerates f failures: a 3-node cluster tolerates 1 failure, a 5-node cluster tolerates 2. This is why clusters use odd node counts. Consensus is what prevents split brain in failover setups.

### Distributed transactions and Sagas

When one business operation spans multiple databases or services, a local ACID transaction doesn't cover it. Two-Phase Commit (2PC) gives true atomicity across systems but blocks if the coordinator crashes — participants hold locks until it recovers. Sagas break the distributed transaction into a sequence of local transactions, each committing independently. If a step fails, compensating transactions undo the already-committed steps in reverse. Compensations are not rollbacks — the original transaction really committed; a compensation is a new business action like issuing a refund.

Sagas can be choreographed (each service listens for events and decides what to do — decentralized but hard to follow) or orchestrated (a central coordinator tells each service what to do — explicit and observable but adds another component).

### Outbox pattern

The dual-write problem: you need to update the database and publish a message to a queue atomically. If you write to the DB then crash before publishing, the event is lost. If you publish first then the DB write fails, downstream services act on a phantom event. The outbox pattern solves this by writing the event into an outbox table in the same database transaction as the business change. A separate relay process reads the outbox and publishes to the message broker. Because the relay may retry on failure, outbox delivers at-least-once — consumers must be idempotent.

### Change Data Capture (CDC)

CDC reads every row-level change directly from the database's transaction log rather than querying the tables. This captures everything including hard deletes, in commit order, with minimal overhead. Used for syncing search indexes, caches, and warehouses without polling, and for event-driven services that react to data changes without the source service writing extra code. The common tool is Debezium reading Postgres WAL or MySQL binlog.

---

### Gotchas

- CAP's "pick two" framing is misleading — P is mandatory in any real distributed system; the choice is always C vs A during a partition.
- Saga compensations are new business actions, not rollbacks — the original steps really committed and other services may have already acted on them.
- Outbox delivers at-least-once, not exactly-once — if the relay publishes and crashes before marking the message sent, it will publish again after restart; consumers must deduplicate.
- An inactive CDC replication slot causes Postgres to retain WAL forever — if the consumer stops, WAL accumulates on disk until it fills up; monitor and alert on slot lag.
- Write skew cannot be prevented by snapshot isolation — two transactions can each see a consistent picture and together violate an invariant; only Serializable isolation or explicit locking prevents it.
