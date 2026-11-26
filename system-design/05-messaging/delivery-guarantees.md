# System Design: Delivery Guarantees & Ordering

> Part of the [System Design](../index.md) series.

---

## Delivery Guarantees

**At-most-once** — message is delivered zero or one times. Can be lost. Fastest. Use for metrics or logs where occasional loss is acceptable.

**At-least-once** — message is delivered one or more times. Never lost, but may be delivered twice on retry. The consumer must be **idempotent** to handle duplicates safely. The common default.

**Exactly-once** — delivered exactly once, no duplicates, no loss. Hardest to achieve. Kafka supports this with transactions + idempotent producers. SQS does not natively — you emulate it with idempotency keys.

---

## Ordering

Messages within a **partition** are strictly ordered. Messages across partitions are not. To guarantee ordering for a logical entity (e.g., all events for a given user), use the user ID as the **partition key** — all messages for that user land in the same partition and are processed in order.
