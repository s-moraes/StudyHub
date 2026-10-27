## Q1
Why do high-scale distributed systems monitor p95 and p99 latency percentiles instead of relying solely on the arithmetic mean (average)?

### Option A
Because p99 is mathematically faster to compute in SQL databases than `AVG()`.

### Option B [CORRECT]
Because averages hide high tail-latency spikes experienced by the slowest 1% to 5% of requests, which degrade fan-out architectures.

### Option C
Because network switches automatically drop packets that exceed the p50 latency threshold.

### Option D
Because horizontal scaling only improves p99 latency and never affects average latency.

### Explanation
Arithmetic averages mask outliers. If 99 requests take 10ms and 1 request takes 5,000ms, the average looks healthy (~60ms), but in a microservice architecture where a page aggregates 10 backend calls, the probability of hitting at least one p99 slow call becomes very high.

===

## Q2
In a URL Shortener service, what is the primary behavioral difference between returning an `HTTP 301` vs. an `HTTP 302` status code?

### Option A
`301` encrypts the redirect URL using TLS, whereas `302` sends it in plain text.

### Option B
`301` supports Base62 keys, whereas `302` only supports numeric IDs.

### Option C [CORRECT]
`301` instructs browsers to cache the redirect permanently (bypassing your server on repeat visits), while `302` forces requests through your server for analytics tracking.

### Option D
`302` is only valid for POST requests, while `301` is used for GET requests.

### Explanation
`301 Moved Permanently` allows browsers to resolve subsequent visits directly from local cache, reducing server load. `302 Found` ensures every click hits the shortener service so click metrics can be recorded.

===

## Q3
Which requirement is essential for an application service tier to scale horizontally behind a round-robin load balancer without sticky sessions?

### Option A [CORRECT]
The application instances must be stateless, storing session or user state externally (e.g., Redis or signed tokens).

### Option B
Each application instance must maintain a local `ConcurrentHashMap` of active user sessions in JVM heap memory.

### Option C
All instances must share the exact same physical CPU socket.

### Option D
The service must only communicate via synchronous HTTP/1.0 connections.

### Explanation
When instances are stateless, any server in the pool can handle any consecutive request from a user without losing session context.
