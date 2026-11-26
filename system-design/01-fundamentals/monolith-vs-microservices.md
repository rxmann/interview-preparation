# System Design: Monolith vs Microservices

> Part of the [System Design](../index.md) series.

---

| Aspect | Monolith | Microservices |
|---|---|---|
| Definition | Self-contained features from UI to backend; one unified unit | App split into small independent services communicating over APIs, queues, and DB updates |
| Deployment | Deploy the entire codebase | Deploy one service independently |
| Scaling | Duplicate the entire application | Scale only the service under load |
| Fault isolation | A bug or regression can break the whole system | A bug in one service doesn't take down the whole app |
| Tech stack | Single language, single DB | Multiple languages; usually one DB per service |
| Code | Less duplication, faster to build initially | Logic may be duplicated across services |
| Responsibility | Too much in one unit as it grows | Less responsibility per service; easier to test |
| Development | Coupling slows teams down over time | Parallel development across teams is easier |

**When to stay monolith:** early stage, small team, domain boundaries aren't clear yet. A poorly split microservices system is worse than a monolith — you get the distributed systems complexity without the benefits.

**Interview note:** "I'd start with a monolith and extract services when a clear boundary emerges — a team owns it, it has distinct scaling needs, or it needs a different tech stack."

---

→ For distributed transactions across services, see [05-messaging/saga.md](../05-messaging/saga.md)

→ For service discovery, see [06-distributed-systems/service-discovery.md](../06-distributed-systems/service-discovery.md)
