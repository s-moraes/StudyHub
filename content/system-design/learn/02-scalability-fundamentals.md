# Scalability Fundamentals

Scalability is a system's ability to handle growing amounts of load gracefully by adding resources.

---

## 1. Vertical Scaling (Scale Up)
Adding more CPU cores, RAM, or faster NVMe storage to an existing server.

* **Pros:** Zero code changes required, no distributed state coordination, strong consistency out of the box.
* **Cons:** Hard physical hardware limits, single point of failure (SPOF), and exponential cost curves at high tiers.

---

## 2. Horizontal Scaling (Scale Out)
Adding more commodity instances (nodes/containers) behind a Load Balancer.

* **Pros:** Near-infinite elasticity, fault tolerance through redundancy, cost-effective commodity hardware.
* **Cons:** Requires **stateless application tiers**, distributed caching, and database partitioning/replication.

---

## Keeping the Application Tier Stateless

To scale horizontally without sticky sessions, never store user session state inside a specific node's local memory:

```java
// Anti-pattern for Horizontal Scaling: Local in-memory session map
public class LocalSessionStore {
    private final Map<String, UserSession> sessions = new ConcurrentHashMap<>();
}
```

Instead, offload session state to a shared, low-latency store (such as Redis) or use signed, stateless tokens (JWT) so any instance in the pool can serve any request.
