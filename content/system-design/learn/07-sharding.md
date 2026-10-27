# Partitioning & Sharding

Partitioning divides large datasets into smaller subsets called shards.

## Sharding Strategies
1. **Range-based Partitioning:** Assigns contiguous key ranges to specific nodes. Easy range queries, but prone to hotspotting.
2. **Hash-based Partitioning:** Applies a hash function to the key (e.g., MD5, MurmurHash3) to distribute uniformly.
