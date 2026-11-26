# System Design: Service Discovery

> Part of the [System Design](../index.md) series.

---

In a microservices system, service instances come and go (auto-scaling, deploys, crashes). Service discovery lets services find each other without hardcoded IPs.

**Client-side discovery** — the client queries a **service registry** (e.g., Consul, Eureka) to get the list of healthy instances, then picks one and calls it directly. The client owns the load balancing logic.

**Server-side discovery** — the client calls a load balancer; the load balancer queries the registry and routes the request. The client doesn't need to know about the registry.
