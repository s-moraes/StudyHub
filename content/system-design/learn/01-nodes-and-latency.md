# Nodes, Latency & Failure

In a single-machine application, function calls happen in memory within nanoseconds and rarely fail unless the entire machine crashes. When you distribute a system across multiple nodes connected by a network, everything changes.

---

## The Fallacies of Distributed Computing

Originally formulated by L. Peter Deutsch and colleagues at Sun Microsystems, these are the false assumptions engineers often make when building distributed systems:

1. **The network is reliable:** Packets get dropped, switches fail, and TLS handshakes time out.
2. **Latency is zero:** Moving data across regions takes milliseconds—orders of magnitude slower than RAM access.
3. **Bandwidth is infinite:** Serialization overhead and large payloads saturate network links quickly.
4. **The network is secure:** Every inter-service call must be authenticated and encrypted.

---

## Latency vs. Throughput

| Metric | Definition | Typical Unit |
| :--- | :--- | :--- |
| **Latency** | Time required to process a single request from start to finish. | `ms` (Milliseconds) |
| **Throughput** | Number of requests a system can process per unit of time. | `RPS` / `QPS` |

### Measuring Tail Latency (Percentiles)
Averages hide real user pain. Instead of `mean` latency, distributed systems use percentiles:
* **p50 (Median):** Half of the requests are faster than this value.
* **p95 / p99 (Tail Latency):** The slowest 5% or 1% of requests. In microservice fan-out architectures, a single slow downstream node dictates the overall response time.

```text
Client ---> [API Gateway] ---> [Service A (12ms)]
                          ---> [Service B (18ms)]
                          ---> [Service C (240ms - p99 spike)]
Total Response Time >= 240ms
```
