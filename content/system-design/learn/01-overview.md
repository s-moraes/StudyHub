# Distributed Systems Overview

A distributed system is a collection of autonomous computing entities that communicate over a network to achieve a common goal.

## Key Properties
- **Concurrency:** Components execute tasks simultaneously.
- **No Global Clock:** Ordering events relies on logical clocks (such as Lamport timestamps or Vector clocks).
- **Independent Failures:** Individual nodes can fail while the rest of the system continues to operate.

## Why Distribute Systems?
1. **Scalability:** Horizontal scaling handles increased compute and storage load.
2. **Reliability:** Redundant nodes prevent single points of failure.
3. **Geo-Distribution:** Placing nodes close to end-users reduces network latency.
