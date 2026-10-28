# Cloud Database Concurrency & Locking

In cloud-native environments, dozens or hundreds of microservice replicas connect to shared relational or distributed databases simultaneously. Managing concurrent access to shared resources without corrupting data or causing crippling bottlenecks requires a disciplined strategy.

---

## The Concurrency Dilemma in the Cloud

When multiple transactions attempt to read and mutate the same records concurrently, systems risk data integrity violations:

- **Lost Updates:** Transaction 1 overwrites changes made by Transaction 2 without reading them.
- **Dirty Reads:** Reading uncommitted changes that roll back later.
- **Non-Repeatable Reads:** Rereading the same row yields different values within the same transaction.
- **Phantom Reads:** Range queries return new rows committed by other transactions mid-flight.

While relational databases provide ACID isolation levels, distributed cloud environments face a delicate balance between strict consistency and throughput.

---

## In-JVM Locking vs Cloud Reality

Java provides mature in-memory synchronization primitives:

```java
// Local JVM lock using ReentrantLock
public class AccountService {
    private final ReentrantLock lock = new ReentrantLock();

    public void debit(String accountId, BigDecimal amount) {
        lock.lock();
        try {
            // Modify in-memory state
        } finally {
            lock.unlock();
        }
    }
}
```

### The Limitation of In-JVM Locks
Java's `synchronized` keyword and `ReentrantLock` only synchronize threads running **within the exact same JVM process**. In a Kubernetes cluster where 10 instances of `AccountService` run on separate pods, an in-JVM lock cannot prevent Pod A and Pod B from mutating the exact same database row at the same time.

For multi-instance synchronization, concurrency controls must be delegated to the database engine or a distributed lock manager.

---

## Optimistic vs Pessimistic Locking

Java frameworks like JPA and Hibernate provide two primary concurrency strategies:

```text
+-----------------------+---------------------------------------------+
| Optimistic Locking    | Assumes conflicts are rare.                 |
|                       | Checks version column at commit time.       |
+-----------------------+---------------------------------------------+
| Pessimistic Locking   | Assumes conflicts are frequent.             |
|                       | Holds database row locks (SELECT FOR UPDATE)|
+-----------------------+---------------------------------------------+
```

### 1. Optimistic Locking
Optimistic locking assumes transactions rarely collide. It uses a version counter or timestamp column on the entity table:

```java
@Entity
@Table(name = "products")
public class Product {

    @Id
    private Long id;

    private String name;
    private int stock;

    @Version
    private Long version;
}
```

When Hibernate generates the SQL update, it appends the current version to the `WHERE` clause:

```sql
UPDATE products 
SET stock = 45, version = version + 1 
WHERE id = 101 AND version = 3;
```

If another transaction already updated version `3` to `4`, the statement affects `0` rows, and Hibernate immediately throws an `OptimisticLockException`. The application can then catch this exception, reload the fresh data, and retry the operation.

### 2. Pessimistic Locking
Pessimistic locking acts proactively by acquiring database-level exclusive locks upon reading the data:

```java
public interface ProductRepository extends JpaRepository<Product, Long> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT p FROM Product p WHERE p.id = :id")
    Optional<Product> findByIdForUpdate(@Param("id") Long id);
}
```

This translates directly to a `SELECT ... FOR UPDATE` SQL query. Any other transaction attempting to read or write this record must wait until the locking transaction commits or rolls back.

### Choosing the Right Rhythm

| Dimension | Optimistic Locking | Pessimistic Locking |
| :--- | :--- | :--- |
| **Conflict Frequency** | Low to moderate contention | High contention / Hotspots |
| **Lock Duration** | Zero lock time during transaction | Locks held for entire transaction |
| **Throughput** | High read/write throughput | Reduced throughput due to queueing |
| **Deadlock Risk** | None (fails fast with exception) | High if locking order is inconsistent |
| **Failure Cost** | Client must retry transaction | Blocked connection threads |

---

## Caching the Craze: Distributed In-Memory Grids

Constantly hammering the database to handle concurrent reads creates severe performance bottlenecks. Frequently read data is offloaded to a distributed in-memory data grid such as **Hazelcast** or **Apache Ignite**.

```java
// Configuring a distributed map in Hazelcast
Config config = new Config();
HazelcastInstance hz = Hazelcast.newHazelcastInstance(config);

IMap<String, ProductCacheDto> productMap = hz.getMap("products");

// Atomic read-and-update across cluster nodes
productMap.lock("prod-101");
try {
    ProductCacheDto product = productMap.get("prod-101");
    product.decrementInventory(1);
    productMap.put("prod-101", product);
} finally {
    productMap.unlock("prod-101");
}
```

### Benefits of Distributed Caching
- **Offloaded Database I/O:** 90%+ of repetitive read queries terminate in memory, slashing latency to sub-milliseconds.
- **Distributed Locks & Mutexes:** Distributed grids provide cluster-aware locks (`IMap.lock()`) that synchronize actions across distinct JVM containers.
- **Scalable Topologies:** Caches partition data automatically across cluster nodes, scaling memory linearly as instances scale.
