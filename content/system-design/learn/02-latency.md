# Network Latency & Fallacies

Peter Deutsch described 8 fallacies commonly assumed when building distributed software.

## The Eight Fallacies
1. The network is reliable.
2. Latency is zero.
3. Bandwidth is infinite.
4. The network is secure.
5. Topology doesn't change.
6. There is one administrator.
7. Transport cost is zero.
8. The network is homogeneous.

Handling these realities requires timeouts, retries with exponential backoff, circuit breakers, and idempotency keys.
