# System Design: Saga & Distributed Transactions

> Part of the [System Design](../index.md) series.

---

A distributed transaction spans multiple services — you can't use a single DB transaction across service boundaries.

## 2PC vs. SAGA

| | 2PC (Two-Phase Commit) | SAGA |
|---|---|---|
| Consistency model | Strong atomicity | Eventual consistency |
| Coordinator | A single coordinator manages all services | Choreographer (events) or Orchestrator (central) |
| Blocking | Client waits for all services to confirm | Each service commits locally |
| Failure | A failure can leave the transaction open across services | Rolls back previous steps via compensating transactions |
| Performance | O(slowest service) — synchronous | Async — faster overall |
| Complexity | Simpler logic, hard to implement correctly at scale | More complex rollback logic |

## 2PC Flow

```mermaid
sequenceDiagram
    participant C as Coordinator
    participant M1 as Service 1
    participant M2 as Service 2
    C->>M1: Prepare
    C->>M2: Prepare
    M1-->>C: Ready
    M2-->>C: Ready
    C->>M1: Commit / Rollback
    C->>M2: Commit / Rollback
```

## SAGA Flow

**Choreography (event-driven):** each service listens for events and publishes its own. No central coordinator. Harder to trace end-to-end.

```mermaid
flowchart TD
    S1["Service 1"] -->|event| E(["Event Bus"])
    E --> S2["Service 2"]
    S2 -. "compensating event on failure" .-> S1
```

**Orchestration:** a central orchestrator tells each service what to do and handles rollback explicitly. Easier to understand but the orchestrator becomes a potential SPOF.

```mermaid
flowchart LR
    O["Orchestrator"] --> S1["Service 1"]
    O --> S2["Service 2"]
    O -. "rollback" .-> S1
    O -. "rollback" .-> S2
```

**Interview note:** "For the checkout flow — reserve inventory, process payment, create order — I'd use a SAGA with orchestration. If payment fails, the orchestrator fires a compensating transaction to release the inventory reservation."
