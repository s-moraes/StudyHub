# Scalability Fundamentals

Scalability is the characteristic of a system that describes its capability to perform and adapt well under an increased workload.

![Scalability Architecture](../assets/scalability-diagram.webp)

## Horizontal vs Vertical Scaling
- **Vertical Scaling (Scale-Up):** Adding more power (CPU, RAM, storage) to an existing machine. Fast initially, but hits hardware ceilings and creates single points of failure.
- **Horizontal Scaling (Scale-Out):** Adding more instances or nodes to the pool. Preferred for elastic cloud architectures.

### Core Load Distribution Pattern
```java
public class RoundRobinDispatcher {
    private final List<String> serverNodes;
    private final AtomicInteger pointer = new AtomicInteger(0);

    public RoundRobinDispatcher(List<String> serverNodes) {
        this.serverNodes = serverNodes;
    }

    public String selectNode() {
        int index = pointer.getAndIncrement();
        return serverNodes.get(Math.abs(index % serverNodes.size()));
    }
}
```

## Statelessness
Keeping application servers stateless by offloading session data to distributed caches (e.g., Redis) or databases enables trivial auto-scaling.
