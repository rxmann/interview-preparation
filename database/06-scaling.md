# Database Notes: Scaling and Distribution

> Part of the [Database Notes](index.md) series.

---

## 6. Scaling and Distribution

### Vertical vs horizontal scaling

Vertical scaling means moving to a bigger machine — more CPU, more RAM, faster storage. It requires no application changes, keeps everything simple, and a single well-tuned machine can serve most applications for a long time. Its limits are the hardware ceiling, the cost curve at the high end, and the fact that one machine is a single point of failure.

Horizontal scaling means adding more machines and distributing the work across them. It offers near-unlimited capacity and fault tolerance, but introduces distributed systems complexity: replication lag, consistency trade-offs, cross-node queries, and distributed transactions. The right scaling path is to exhaust the cheap options first — fix slow queries, add indexes, add connection pooling, scale vertically, add caching, then read replicas — and only shard when writes or storage truly exceed what one primary can handle.

Reads and writes scale differently. Read traffic scales easily with replicas and caches. Write traffic is harder because all writes must go through one leader in the simplest architectures.

### Replication

**Leader-follower** is the most common model: one leader accepts all writes and streams changes to followers, which replicate asynchronously. Reads can be spread across followers. It's simple and supported by every major database (Postgres streaming replication, RDS read replicas). The downside is that write throughput is limited to one machine and leader failure requires failover.

**Multi-leader** allows multiple nodes to accept writes, typically one per region. Writes are fast locally, and each region keeps working during network partitions. The hard problem is write conflicts — the same record modified in two regions concurrently needs resolution (last-write-wins discards data, CRDTs merge mathematically, or you route each record's writes to a home region).

**Leaderless** (Dynamo-style) has the client write to W of N replicas and read from R replicas. If W + R > N, the read set must overlap the write set, ensuring at least one response has the latest value. No single point of failure, but eventual consistency and conflict resolution are required.

**Synchronous vs asynchronous**: synchronous replication means the leader waits for a follower to confirm the write before telling the client it succeeded — zero data loss on failover, but every commit pays a network round trip. Asynchronous is faster but a follower might be behind, and if the leader dies, those unshipped writes are lost.

### Replication lag

Lag is the delay between a write committing on the leader and appearing on a follower. It causes read-your-writes violations (you save your profile, reload the page, and see the old data because the read hit a lagging replica) and monotonic reads violations (refreshing twice shows newer data then older). Fix read-your-writes by routing the user's reads to the leader for a window after their write, or by tracking which log position the write landed at and only reading from replicas that have caught up.

### Sharding

Sharding splits data across multiple machines by a shard key. Hash sharding distributes evenly but makes range queries hit every shard. Range sharding keeps ranges together but risks hotspots (all new writes hitting the latest shard). Directory sharding uses a lookup service to map keys to shards — fully flexible but needs the lookup to be highly available.

A good shard key has high cardinality, distributes load evenly, and aligns with the dominant query pattern so most queries touch one shard. Cross-shard joins and transactions are painful and should be designed away.

### Consistent hashing and failover

Consistent hashing places both nodes and keys on a ring, so adding or removing a node moves only about 1/N of the keys instead of remapping everything. Virtual nodes give each physical node multiple ring positions for smoother distribution.

Failover is the process of promoting a replica to leader after the primary dies: detect failure, choose the most up-to-date replica, promote it, reconfigure clients to point at the new leader, and reintegrate the old leader as a follower when it returns. The risks are split brain (two nodes both believing they're leader, causing conflicting writes) and data loss with async replication.

---

### Gotchas

- Sharding sacrifices joins, cross-shard transactions, and global uniqueness — treat it as a last resort, not a first architecture choice.
- Read-your-writes is a common gotcha with replicas — always route writes and their immediate reads to the leader or track LSN position.
- Split brain is prevented by using consensus (Raft/Paxos) for leader election and fencing tokens on storage to reject writes from stale leaders.
- Global unique constraints (unique email across all shards) cannot be enforced by the database — you need a separate lookup table or service.
- With many application instances each holding a connection pool, total connections to the database multiply quickly — use an external pooler like PgBouncer.
