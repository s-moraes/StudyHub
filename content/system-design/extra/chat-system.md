# Design a Chat System

Deliver messages in real time with presence, ordering, and offline delivery.

## Key Components
- **WebSocket Gateway:** Maintains persistent duplex TCP connections with clients.
- **Message Sync Service:** Assigns sequence numbers to guarantee message ordering across channels.
