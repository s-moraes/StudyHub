# Data Replication Strategies

Replication means keeping a copy of the same data on multiple nodes connected via network.

## Topologies
- **Single-Leader:** All writes go to one leader node, which streams changes to followers.
- **Multi-Leader:** Multiple nodes accept writes; changes are reconciled asynchronously.
- **Leaderless:** Clients write directly to several replicas (Quorum writes: W + R > N).
