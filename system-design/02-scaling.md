# System Design: Scaling

> Part of the [System Design](index.md) series.

---

**The problem:** a single server has a ceiling — CPU, memory, disk I/O, network. As traffic grows, you need a strategy to handle more load.

## Vertical Scaling (scale up)

Buy a bigger machine — more CPU, more RAM, faster disk.

**When to use:** early stage, before complexity of distributed systems is justified. Easy to implement, no architecture changes needed.

**Ceiling:** hardware limits. The largest AWS instance today is around 448 vCPUs / 24 TB RAM. Beyond that, you must go horizontal. Also a single point of failure — if that machine goes down, everything goes down.

**Interview note:** "We'll start vertical, then move horizontal as we approach limits."

---

## Horizontal Scaling (scale out)

Add more machines and distribute load across them.

**When to use:** when vertical scaling hits its limit, when you need fault tolerance, when traffic is bursty and you want to auto-scale.

**Requires:**

- A **load balancer** to distribute requests
- **Stateless application servers** (session data in a shared store like Redis, not in memory)
- A strategy for **data consistency** across nodes (more complex — this is where distributed systems problems start)

|  | Horizontal | Vertical |
|---|---|---|
| Load balancer | Required | Not needed |
| Fault tolerance | High (redundant nodes) | Single point of failure |
| Data consistency | Hard (distributed state) | Easy (single machine, ACID) |
| Communication overhead | Network calls between nodes | Inter-process (fast) |
| Elasticity | Scales up/down dynamically | Fixed to machine size |
| Cost curve | Linear (add nodes as needed) | Steep jump at each tier |
