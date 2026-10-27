# Design a Chat System

Deliver messages in real time with presence, ordering, and offline delivery.

---

## 1. Real-Time Transport Protocol

Standard HTTP is request-response initiated by the client, making it inefficient for instant message delivery.

* **Polling / Long Polling:** Easy fallback, but high header overhead and connection churn.
* **WebSockets:** Full-duplex, persistent TCP connection between client and Chat Gateway. Ideal for bidirectional low-latency messaging.

```text
[Sender] ===(WebSocket)===> [Chat Gateway 1] ---> [Message Broker / KV Store]
                                                        |
[Recipient] <==(WebSocket)== [Chat Gateway 2] <---------+
```

---

## 2. Message Ordering & IDs

Timestamps from client devices cannot be trusted due to clock skew. Even server timestamps across distributed nodes can drift.
* Use a **per-conversation monotonic sequence number** or a **Snowflake-like k-sortable ID** scoped to the `channel_id` so clients always render messages in deterministic order.

---

## 3. Presence & Heartbeats

To avoid network storms when millions of users briefly lose mobile signal:
* Clients send a lightweight heartbeat frame every **30 seconds**.
* The Presence Service marks a user `OFFLINE` only if no heartbeat arrives within a grace window (e.g., **60–90 seconds**).
