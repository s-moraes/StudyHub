# Managing State in Microservices

Microservices represent an architectural style where independently deployable services collaborate to deliver complex business capabilities. However, managing mutable state across this distributed landscape is one of the most challenging engineering hurdles. Without deliberate state boundaries, distributed systems quickly devolve into tightly coupled, fragile distributed monoliths.

---

## The Stateless Serenity

The foundational rule of scalable cloud-native design is to keep compute services as **stateless** as possible. A stateless service does not retain client context or session state between requests. Any instance of the service can process any incoming request with identical results.

```text
[ Client ] ---> [ Load Balancer ]
                      |
        +-------------+-------------+
        |                           |
        v                           v
[ Order Service A ]         [ Order Service B ]
   (Stateless)                 (Stateless)
        \                           /
         +------------+------------+
                      v
             [ Redis / Database ]
               (State Layer)
```

### Key Benefits of Stateless Design

- **Effortless Horizontal Scaling:** New pods or instances can spin up or shut down dynamically in response to traffic spikes without migrating in-memory session data.
- **Resilience and Auto-Recovery:** When a service instance crashes, the load balancer routes subsequent requests to healthy instances with zero data loss or session drops.
- **Simplified Deployments:** Rolling upgrades, blue-green deployments, and canary releases occur smoothly without needing session draining or sticky sessions.

---

## Distributed Session Sherpas: Redis and ZooKeeper

When an application genuinely requires state across multiple requests (such as user authentication tokens, active shopping carts, or ephemeral wizard progress), holding state in JVM heap memory (`HttpSession`) breaks horizontal scaling.

If request 1 lands on Node A and request 2 lands on Node B, sticky sessions (routing requests from a client to the same node) create severe traffic hotspots and data loss when nodes restart.

The solution is to offload session tracking to **distributed state sherpas**:

```java
// Spring Session data configuration using Redis
@Configuration
@EnableRedisHttpSession(maxInactiveIntervalInSeconds = 1800)
public class SessionConfig {

    @Bean
    public LettuceConnectionFactory connectionFactory() {
        return new LettuceConnectionFactory(
            new RedisStandaloneConfiguration("redis-cluster", 6379)
        );
    }
}
```

### Redis for Distributed Session Storage
- **High Throughput & In-Memory Speed:** Sub-millisecond read and write latency for session keys.
- **Automatic TTL Expiration:** Native Time-To-Live (TTL) automatically purges inactive sessions without background cleaner threads.
- **Clustering & Replication:** High availability across availability zones via primary-replica replication and Redis Sentinel/Cluster.

### ZooKeeper for Distributed State Tracking
- **Ephemeral Nodes & Watches:** Used when session state reflects dynamic node participation, active cluster membership, or distributed lease ownership.
- **Strong Consistency:** Guarantees linearizable writes through the Zab consensus algorithm.

---

## CQRS and Event Sourcing: The Stateful Waltz

For complex domains where state transitions must be audited, replayed, or queried across various dimensions, traditional CRUD databases fall short. Combining **Command Query Responsibility Segregation (CQRS)** with **Event Sourcing (ES)** provides a battle-tested architecture.

```text
  [ Command API ]                   [ Query API ]
         |                                ^
         v                                |
  [ Aggregate Root ]             [ Materialized View ]
         |                         (Elastic / Read DB)
         v                                ^
  [ Event Store ] ------------------------+
   (Append-Only)         Project / Replay
```

### 1. Command Query Responsibility Segregation (CQRS)
CQRS splits the data model into two separate paths:
- **Command Model:** Optimized for business validation, transactional integrity, and state-mutating commands (e.g., `PlaceOrderCommand`).
- **Query Model:** Optimized for fast, denormalized, non-blocking reads (e.g., `GetCustomerOrdersQuery`).

### 2. Event Sourcing
Instead of storing only the *current* state of an entity (e.g., `status = "SHIPPED"`), Event Sourcing captures every state modification as an immutable, append-only event sequence (`OrderCreatedEvent`, `OrderPaidEvent`, `OrderShippedEvent`).

The current state is reconstructed by replaying past events sequentially:

$$\text{Current State} = \sum_{i=1}^{n} \text{Event}_i$$

### Java Implementation with Axon Framework
The Axon Framework simplifies CQRS and Event Sourcing on the JVM by providing abstractions for aggregate roots, command handlers, and event sourcing handlers:

```java
@Aggregate
public class OrderAggregate {

    @AggregateIdentifier
    private String orderId;
    private boolean completed;

    // Default constructor for Axon reconstruction
    protected OrderAggregate() {}

    @CommandHandler
    public OrderAggregate(CreateOrderCommand cmd) {
        // Enforce invariants before applying event
        if (cmd.getAmount() <= 0) {
            throw new IllegalArgumentException("Amount must be positive");
        }
        AggregateLifecycle.apply(new OrderCreatedEvent(cmd.getOrderId(), cmd.getAmount()));
    }

    @EventSourcingHandler
    public void on(OrderCreatedEvent event) {
        this.orderId = event.getOrderId();
        this.completed = false;
    }
}
```

---

## Architectural Comparison Matrix

| Strategy | Primary Use Case | Scalability | Complexity | Failure Recovery |
| :--- | :--- | :--- | :--- | :--- |
| **Stateless Services** | REST APIs, business services, compute | Very High | Low | Instant failover across nodes |
| **Distributed Sessions** | Auth sessions, shopping carts, wizards | High | Moderate | Redis/ZooKeeper replica failover |
| **CQRS + Event Sourcing** | Financial ledgers, audits, complex domains | High | High | Deterministic state replay from log |
