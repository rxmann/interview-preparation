# System Design: Core Vocabulary

> Part of the [System Design](../index.md) series.

---

These terms appear constantly. Knowing their precise meaning — and the tradeoffs between them — is what separates a vague answer from a crisp one.

**Latency** — time for one request to complete, measured end-to-end. A p99 latency of 200ms means 99% of requests finish within 200ms. When you say "latency," always clarify percentile — average hides the tail.

**Throughput** — requests (or bytes) the system handles per second. High throughput and low latency are often in tension: batching increases throughput but adds latency per individual item.

**Scalability** — the system's ability to handle increased load by adding resources, without requiring a rewrite.

**Availability** — percentage of time the system is operational. Usually expressed in "nines":

- 99.9% = ~8.7 hours downtime/year (three nines)
- 99.99% = ~52 minutes/year (four nines)
- 99.999% = ~5 minutes/year (five nines, rare in practice)

**Reliability** — the system does the right thing consistently, even under failure. A system can be available (responds) but unreliable (returns wrong data).

**Fault tolerance** — the system continues operating when components fail. Achieved through redundancy, replication, and graceful degradation.

**Consistency** — all nodes see the same data at the same time. In distributed systems this is a spectrum: strong consistency (every read reflects the latest write) vs. eventual consistency (reads may lag, but converge).

**Durability** — once data is written and acknowledged, it survives crashes. The D in ACID.

**Idempotency** — an operation that produces the same result when applied once or many times. Critical for safe retries. `PUT /orders/123/status=paid` is idempotent; `POST /orders/pay` is not.

**Stateless vs stateful** — a stateless service holds no session data; any instance can handle any request. Stateful services tie a client to a specific node. Stateless is easier to scale horizontally. Push state into the database or a shared cache (Redis) so servers stay stateless.

**SLA / SLO / SLI**:

- SLI (Service Level Indicator): the actual measured metric (latency p99, error rate)
- SLO (Service Level Objective): the target ("p99 < 200ms")
- SLA (Service Level Agreement): the contractual promise with consequences for missing it
