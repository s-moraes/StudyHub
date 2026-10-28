# Distributed Coordination & Consensus Primitives

In distributed systems, microservices cannot rely on a single shared CPU or centralized operating system kernel to maintain order. Tasks such as **leader election**, **distributed mutual exclusion (locks)**, and **cluster configuration synchronization** require consensus protocols to guarantee consistency across network partitions.

---

## The Distributed Coordination Problem

When 5 instances of a microservice run simultaneously, problems arise:
- **Leader Election:** Who runs the periodic billing cron job so it doesn't execute 5 times?
- **Distributed Locking:** Who gets exclusive write access to a shared object without stepping on other nodes?
- **Configuration Management:** How do all nodes immediately receive updated feature flags without restarting?

To solve this, cloud architectures rely on dedicated coordination engines built on formal consensus algorithms (Zab, Raft, or Paxos).

---

## 1. Apache ZooKeeper: The Proven Coordinator

**Apache ZooKeeper** coordinates distributed processes using a shared hierarchical namespace structured like a filesystem, consisting of **znodes**.

```text
                  / (Root)
                 /        \
          /services      /locks
          /        \        |
      /orders-1  /orders-2  +--- /lock-node-00001 (Ephemeral Sequential)
      (Ephemeral)           +--- /lock-node-00002
```

### Key Primitives in ZooKeeper
- **Znodes:** Data nodes that can be persistent (remain until deleted) or ephemeral (deleted automatically when client session disconnects).
- **Sequential Flags:** Appends an increasing integer (`000001`, `000002`), enabling FIFO queuing and fair distributed locks.
- **Watches:** Clients attach one-time triggers on znodes. ZooKeeper sends asynchronous push notifications when data changes, eliminating polling.
- **Apache Curator (Java Client):** High-level Java client that wraps ZooKeeper recipes into clean abstractions:

```java
// Distributed Lock using Apache Curator and ZooKeeper in Java
CuratorFramework client = CuratorFrameworkFactory.newClient(
    "zookeeper.internal:2181", 
    new ExponentialBackoffRetry(1000, 3)
);
client.start();

InterProcessMutex lock = new InterProcessMutex(client, "/locks/invoice-job");

if (lock.acquire(10, TimeUnit.SECONDS)) {
    try {
        // Execute critical section: only 1 instance runs this
        processDailyInvoices();
    } finally {
        lock.release();
    }
}
```

---

## 2. etcd: The Kubernetes Cloud Backbone

**etcd** is an open-source, strongly consistent distributed key-value store built by CoreOS. It uses the **Raft consensus algorithm** to achieve linearizable writes and fault tolerance.

### Architectural Features of etcd
- **Raft Consensus:** Easier to understand and implement than classical Paxos, ensuring automatic leader election and log replication.
- **gRPC and HTTP/2 Protocol:** Native streaming APIs for high-efficiency network transport.
- **Leases & Keep-Alives:** Keys are bound to leases with TTLs. If the client fails to renew its lease, etcd automatically expires and removes the key.
- **Watchers:** Long-lived HTTP/2 streams notify listeners immediately when a key or prefix is updated.

Because of its rock-solid reliability and efficiency, etcd acts as the primary data store and source of truth for all **Kubernetes** cluster state (Pods, Secrets, ConfigMaps).

---

## 3. HashiCorp Consul: Service Mesh & Discovery

While ZooKeeper and etcd focus heavily on low-level primitives, **HashiCorp Consul** provides a full-featured distributed service mesh and coordination platform.

### Core Features of Consul
- **Service Discovery & Health Checking:** Services register via agent APIs. Consul monitors application health and routes traffic only to healthy nodes via DNS or HTTP endpoints.
- **Distributed Key-Value Store:** Key-value storage supporting sessions, distributed locking, and watches.
- **Multi-Datacenter Federation:** Built from the ground up to synchronize service registries across multiple public cloud regions.
- **Rich Operator Experience:** Provides a built-in web UI, CLI, and RESTful APIs out of the box.

---

## Coordination Engines Comparison

| Feature | Apache ZooKeeper | etcd | HashiCorp Consul |
| :--- | :--- | :--- | :--- |
| **Consensus Protocol** | Zab (ZooKeeper Atomic Broadcast) | Raft | Raft + Gossip (Serf) |
| **Data Model** | Hierarchical tree (znodes) | Flat Key-Value with revisions | Key-Value + Service Catalog |
| **Protocol / API** | Custom TCP (Jute binary) | gRPC / HTTP/2 (JSON/Protobuf) | HTTP / gRPC / DNS |
| **Ecosystem Fit** | Hadoop, Kafka, Solr | Kubernetes, Cloud Native | Multi-cloud, Microservice Meshes |
| **Operational Weight**| High (requires JVM tuning) | Moderate (single Go binary) | Low to Moderate (single Go binary) |
