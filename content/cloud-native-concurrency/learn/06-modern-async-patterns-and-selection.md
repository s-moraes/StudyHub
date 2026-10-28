# Reactive Streams & Architecture Decision Matrix

Building resilient, high-throughput cloud applications requires shifting from synchronous, blocking paradigms to asynchronous, non-blocking execution models. However, adopting asynchronous patterns introduces new architectural trade-offs that dictate how components interact and scale under heavy load.

---

## Modern Asynchronous Programming Patterns

### 1. The Reactive Streams Specification

When asynchronous services produce data faster than downstream consumers can process it, memory buffers overflow, causing `OutOfMemoryError` crashes or dropped messages.

The **Reactive Streams** initiative (formalized in Java 9 via `java.util.concurrent.Flow`) establishes a standardized contract for non-blocking stream processing with **asynchronous backpressure**.

```text
[ Publisher ]  <----- request(N) -----  [ Subscriber ]
[ (Producer) ]  ---- onNext(item) --->   [ (Consumer) ]
```

The Reactive Streams contract defines four core interfaces:
1. `Publisher<T>`: Emits a sequence of elements to registered Subscribers.
2. `Subscriber<T>`: Receives notifications (`onSubscribe`, `onNext`, `onError`, `onComplete`).
3. `Subscription`: The bridge connecting Publisher and Subscriber. Allows the consumer to signal capacity via `subscription.request(n)`.
4. `Processor<T, R>`: A pipeline stage acting as both Subscriber and Publisher.

```java
// Spring WebFlux & Project Reactor example
@RestController
@RequestMapping("/api/telemetry")
public class TelemetryController {

    private final TelemetryService service;

    public TelemetryController(TelemetryService service) {
        this.service = service;
    }

    @GetMapping(produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<TelemetryData> streamTelemetry() {
        return service.getRealTimeMetrics()
            .filter(metric -> metric.severity() > 3)
            .limitRate(100) // Backpressure: request 100 items at a time
            .onErrorResume(e -> Flux.empty());
    }
}
```

---

## 2. Asynchronous Messaging & Decoupling

While Reactive Streams optimizes intra-process and service-to-service streaming, **Asynchronous Messaging** using message brokers (such as **Apache Kafka**, **RabbitMQ**, or **AWS SQS**) decouples distributed systems at the network boundary.

```text
[ Order Service ] ---> [ Kafka Topic: orders ] ---> [ Inventory Worker 1 ]
                                              ---> [ Inventory Worker 2 ]
```

### Strategic Benefits of Asynchronous Queues
- **Temporal Decoupling:** The producer can send messages even if consumers are temporarily down or undergoing maintenance.
- **Traffic Smoothing (Peak Levelling):** Surges in user traffic sit safely in durable queues rather than overwhelming downstream databases.
- **Parallel Work Distribution:** Partitioned topics enable parallel worker consumer groups to scale horizontally on demand.

---

## Choosing the Right Tool for the Job

Selecting the appropriate concurrency toolkit or coordination mechanism depends on four critical architectural dimensions:

```text
                     [ ARCHITECTURAL DIMENSIONS ]
                                  |
    +---------------+-------------+-------------+---------------+
    |               |                           |               |
[ Complexity ] [ Scalability ]         [ Coordination ] [ Style ]
```

### 1. Complexity vs Productivity
- **Akka / Lagom:** Offers comprehensive actor primitives and state clustering, but requires a steep learning curve and team paradigm shift.
- **Vert.x:** Low framework overhead and simple non-blocking primitives, ideal for fast on-ramps to reactive programming.
- **Standard Spring Boot + Redis:** Familiar MVC programming style with minimal cognitive overhead.

### 2. Coordination Needs
- **Simple distributed locks & caches:** **Redis** with Redisson or **Hazelcast** provides sufficient distributed locking primitives with minimal operational weight.
- **Strict consensus, leader election, and config trees:** **Apache ZooKeeper** or **etcd** are purpose-built for linearizable cluster consensus.
- **Multi-region service routing and discovery:** **HashiCorp Consul** offers an integrated service mesh and discovery catalog out of the box.

### 3. Programming Style
- **Synchronous with Thread Offloading:** Classic Spring MVC or Java 21 Virtual Threads (`Project Loom`) preserves straightforward imperative code.
- **Event-Driven & Non-Blocking:** Reactive Streams (Project Reactor / Vert.x) maximizes resource efficiency for I/O-bound microservices at the cost of stack trace readability.

---

## Architectural Decision Matrix

| Requirement | Recommended Solution | Alternative | Key Trade-off |
| :--- | :--- | :--- | :--- |
| **Microservice User Sessions** | Redis (Spring Session) | ZooKeeper | Redis provides native TTL; ZooKeeper is overkill for ephemeral sessions |
| **Audit Trails & Event Ledgers** | CQRS + Event Sourcing (Axon) | Kafka Event Log | Higher initial domain complexity vs ultimate auditability |
| **High Contention Database Rows** | Pessimistic Locking (`FOR UPDATE`)| Redis Distributed Mutex | Database handles serialization; risks longer transaction latency |
| **Low Contention Database Rows** | Optimistic Locking (`@Version`) | Retry mechanisms | Zero lock overhead; fails fast with retry cost |
| **Ultra-High Throughput Gateway** | Eclipse Vert.x / Spring WebFlux| Akka HTTP | Non-blocking event loop beats thread-per-request under 50k+ sockets |
| **Cluster Leader Election** | Apache ZooKeeper (Curator) | etcd / Consul | Zab/Raft consensus guarantees single leader without split-brain |
