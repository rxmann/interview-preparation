# System Design: CAP Theorem

> Part of the [System Design](index.md) series.

---

A foundation for understanding distributed system tradeoffs. Every distributed system must trade off between three properties:

- **C**onsistency — every read returns the most recent write
- **A**vailability — every request receives a response (not guaranteed to be the latest)
- **P**artition tolerance — the system continues operating when network partitions (nodes can't talk to each other) occur

**The theorem:** in the presence of a network partition, you must choose between consistency and availability. You cannot have all three simultaneously.

**Partition tolerance is not optional** in real distributed systems — networks do fail. So the real choice is: **CP or AP?**

| | CP (Consistent + Partition tolerant) | AP (Available + Partition tolerant) |
|---|---|---|
| On partition | Returns error or waits | Returns possibly stale data |
| Examples | HBase, Zookeeper, MongoDB (strong mode) | Cassandra, DynamoDB, CouchDB |
| When to use | Financial data, inventory counts | User feeds, analytics, caches |

---

## Eventual Consistency

The AP choice. Writes propagate to all nodes eventually. Reads may return stale data briefly. Most large-scale systems (social feeds, product recommendations) accept this because the user experience cost of returning slightly stale data is lower than the cost of a failed request.

**Common patterns under eventual consistency:**

- **Read-your-writes** — after a user writes, route their reads to the same node (or the primary) so they always see their own changes, even if replicas haven't caught up yet.
- **Monotonic reads** — ensure a user never sees older data on a second read than they saw on the first. Achieved by routing a user's reads to the same replica for a session.
- **Bounded staleness** — accept stale reads but only within a defined time window (e.g., "data is at most 5 seconds old").

---

## PACELC (Extension to CAP)

CAP only speaks to behavior during a partition. PACELC asks: **even when the network is healthy**, what's the tradeoff?

> **P**artition → choose **A** or **C**; **E**lse → choose **L** (latency) or **C** (consistency)

- **PA/EL** (e.g., Cassandra, DynamoDB) — available during partitions, low latency otherwise. The common choice for high-scale reads.
- **PC/EC** (e.g., HBase, Zookeeper) — consistent during partitions, consistent otherwise. Pays a latency cost.
- **PA/EC** (e.g., MongoDB) — available during partitions, but chooses consistency over latency in normal operation.

**Interview note:** "For the payments service, I want CP — I'd rather return an error than charge the wrong amount. For the activity feed, AP is fine — a few seconds of lag on a notification is acceptable."
