# Notes

## Sloppy Quorum and Hinted Handoff

### Sloppy Quorum

In a high available database, data is replicated on multiple nodes, which makes up a replication group of `N`. A write operation must be forwarded to these nodes and only after we receive `N` responses can the operation returns success. However, this causes a single point of failure - one node's shutdown disables all incoming writes. To solve this, we relax our requirement to some number `W` < `N` such that we return if we receive `W` responses. To achieve data consistency, we now have to read from `R` nodes for an entry where `W + R > N`. 

For a write request failing to receive `W` responses, we may simply return an error to the client. Another option is to forward the write to other reachable nodes and this strategy is called `Sloppy Quorum`. 

### Hinted handoff

When a nodes receives a write request, it stores the data in a temporary storage, attaching a hint specifying the its original node. Once that node is back online, these entries are handed over to it. 

### Pros and Cons

- Pros: increased write availability - as long as any `W` nodes are available, the database can accept writes.
- Cons: read staled data even if `W + R > N` - the last value may have been temporarily written t some nodes outside of `N` 

## Partitioning and Data Rebalancing in KV Store

In a distributed key–value store, data must be spread across multiple storage nodes, and several partitioning strategies exist. The simplest one hashes each key and takes the result modulo the number of nodes:

```
node_id = hash(key) % node_count
```

This approach is flawed in that when the number of nodes changes, data rebalancing is time consuming - a large number of entries need to be relocated to a different node. Two better options are used in productivity:

1. Hash the key and divide value space into ranges, each of which is assigned to one partition. Each partition is stored on a storage node, so it's a many-to-one mapping. Besides, the storage nodes typically sort the entries by `hash(key)` rather than by original key.

   ```
   partition_id = find_partition(hash(key))
   node_it = check_table[partition_id]
   ```

2. Keep the key intact, and divide the key space into ranges. Each partition covers a range and mapping to a storage node.

The first strategy is employed by `Cassandra, DynamoDB, Riak`. When rebalancing data across nodes, the partition serves as the smallest unit of transfer. Because each partition corresponds to a distinct hash range and entries are stored in hash order, we can use binary search to quickly locate the segments that need to be moved in a storage node.

Unfortunately, this approach has its downside. As data are arranged in hash value order, range scan over key is impossible. Queries like `where key >= xx and key <= xxx` would require sequential scan.

In comparison, the second option support range scan because entries are ordered by key in a node. It is used by `TiKV` and `CockroachDB`. However, it introduces the hotspot problem - unlike hash distribution, which uniformly separate the key space, range distribution may cause keys concentrate in certain key ranges. To address this, the system must dynamically split and merge partitions, which increases complexity.