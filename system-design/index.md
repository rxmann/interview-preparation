# System Design

> Revision notes and vocabulary for system design interviews. Each file covers one topic in depth. Start with vocabulary and scaling, then follow the architecture flow.

---

## Topics

| # | File | Description |
|---|---|---|
| — | [Vocabulary](01-vocabulary.md) | Latency, throughput, availability, consistency, idempotency, SLA/SLO/SLI |
| 1 | [Scaling](02-scaling.md) | Vertical vs horizontal, tradeoff table |
| 2 | [Load Balancer + Reverse Proxy](03-load-balancer.md) | L4/L7, algorithms, health checks, SSL termination, reverse proxy |
| 3 | [Consistent Hashing](04-consistent-hashing.md) | Hash ring, virtual nodes, when to use |
| 4 | [CAP Theorem](05-cap-theorem.md) | CP vs AP, partition tolerance, eventual consistency |
| 5 | [SPOF & Redundancy](06-spof.md) | Common SPOFs, active-passive, active-active, cascading failures |
| 6 | [Message Queue](07-message-queue.md) | Async processing, notifier, heartbeat, what queues fix |
| 7 | [Microservices](08-microservices.md) | Monolith vs microservices, 2PC vs Saga, distributed transactions |
| 8 | [Sharding](09-sharding.md) | Partitioning, shard key, consistent hashing, master-slave |
| 9 | [Caching](10-caching.md) | Layers, eviction, stampede, consistency |
| 10 | [CDN](11-cdn.md) | Edge caching, latency, cache miss flow |
| 11 | [DB Write Scaling](12-db-write-scaling.md) | Write coalescing, WAL, LSM-tree, compaction |

---

## Architecture Flow (Baseline)

A typical production web system today:

```mermaid
flowchart LR
    Client([Client]) -->|HTTPS| DNS[DNS]
    DNS --> CDN[CDN]
    CDN -->|cache miss| LB[Load Balancer]
    LB --> S1[App Server 1]
    LB --> S2[App Server 2]
    S1 & S2 --> Cache[(Redis Cache)]
    S1 & S2 --> Primary[(Primary DB)]
    Primary --> Replica[(Read Replica)]
    S1 & S2 -->|async events| Queue[Message Queue]
    Queue --> Worker[Background Worker]
```

Each hop in this diagram is a topic worth understanding deeply:

- **DNS** → routes to the nearest datacenter or CDN edge
- **CDN** → serves static assets and cached responses at the edge
- **Load Balancer** → distributes traffic, health checks, SSL termination
- **App Servers** → stateless, horizontally scalable
- **Cache (Redis)** → absorbs read traffic, reduces DB load
- **Primary DB + Read Replica** → writes go to primary, reads can fan out to replicas
- **Message Queue** → decouples slow/async work from the request path
- **Background Worker** → processes jobs asynchronously (email, media processing, analytics)

---

## HLD Checklist (Interview Walkthrough)

When walking through any system design question, hit these in order:

1. **Clarify requirements** — scale (how many users, QPS, data volume), latency targets, consistency needs, read/write ratio
2. **Start with the simplest architecture** — single server + DB, then add complexity only as requirements demand
3. **Identify bottlenecks** — where will it fall over first? (usually: DB reads, DB writes, single node limits)
4. **Scale reads** — caching, read replicas, CDN for static assets
5. **Scale writes** — sharding, write-behind caching, message queues to absorb bursts
6. **Eliminate SPOFs** — redundancy at every layer (LB, DB, app servers, DNS)
7. **Handle failure modes** — what happens if the cache goes down? if the primary DB fails? if a downstream service is slow?
8. **Observability** — logging, metrics, tracing, alerting (you can't fix what you can't see)
