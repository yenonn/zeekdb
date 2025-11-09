# Distributed Sharding Design: Multi-Node Cluster Memory Distribution

## Executive Summary

This document explores strategies for distributing cached data across multiple nodes in a cluster, addressing partitioning, routing, replication, consistency, and failure handling.

---

## 1. Core Sharding Strategies

### 1.1 Hash-Based Sharding (Modulo)

**Concept:** Hash the key, take modulo of node count

```
node_id = hash(key) % num_nodes
```

**Example:**
- 3 nodes: Node0, Node1, Node2
- Key "user:123" → hash = 0x7A3F → 31295 % 3 = 1 → **Node1**
- Key "user:456" → hash = 0x1B8E → 7054 % 3 = 2 → **Node2**

**Pros:**
- ✅ Simple to implement
- ✅ Uniform distribution (with good hash function)
- ✅ Deterministic routing (no lookup table needed)
- ✅ Fast computation

**Cons:**
- ❌ **Massive reshuffling on node add/remove**
  - Add node: `3 → 4 nodes` means `hash % 3 → hash % 4`
  - ~75% of keys move to different nodes!
- ❌ Not suitable for dynamic clusters
- ❌ Complete cache invalidation on topology change

**Best For:**
- Static cluster size
- Rare topology changes
- Can afford full cache warm-up on resize

**Implementation Considerations:**
```zig
const Cluster = struct {
    nodes: []Node,

    fn getNode(self: *Cluster, key: []const u8) *Node {
        const hash = std.hash.Wyhash.hash(0, key);
        const node_idx = hash % self.nodes.len;
        return &self.nodes[node_idx];
    }
};
```

---

### 1.2 Consistent Hashing (Ring-Based)

**Concept:** Map both nodes and keys onto a hash ring (0 to 2^64-1), route key to next node clockwise

**Hash Ring Visualization:**
```
         0 / 2^64
           |
    Node2  |  Node0
        \  |  /
         \ | /
          \|/
    -------+-------
          /|\
         / | \
        /  |  \
    Node1  |  Node3
           |
```

**Algorithm:**
1. Hash each node's identifier → position on ring
2. Hash each key → position on ring
3. Walk clockwise to find first node
4. Route request to that node

**Example:**
- Node0 hashed to position: 0x1000000000000000
- Node1 hashed to position: 0x5000000000000000
- Node2 hashed to position: 0x9000000000000000
- Node3 hashed to position: 0xD000000000000000

- Key "user:123" hashes to 0x3000000000000000
  - Walk clockwise → next node is Node1 (0x5000...)
  - **Route to Node1**

**Pros:**
- ✅ **Minimal data movement on topology change**
  - Add node: Only `1/N` of keys move (N = total nodes)
  - Remove node: Only that node's keys move
- ✅ Scales well with dynamic membership
- ✅ Industry-proven (DynamoDB, Cassandra, Riak)

**Cons:**
- ❌ More complex implementation
- ❌ Can have uneven distribution with few nodes
- ❌ Requires sorted data structure (binary search on ring)

**Optimization: Virtual Nodes (vnodes)**

Each physical node maps to multiple virtual nodes on ring:

```
Physical Node0 → vnodes: Node0-v1, Node0-v2, ..., Node0-v100
Physical Node1 → vnodes: Node1-v1, Node1-v2, ..., Node1-v100
```

**Benefits:**
- Better load distribution
- Smoother rebalancing
- Handles heterogeneous node capacities (more vnodes for bigger machines)

**Typical vnode count:** 100-500 per physical node

**Implementation Sketch:**
```zig
const Ring = struct {
    // Sorted by hash position
    vnodes: std.ArrayList(VNode),

    const VNode = struct {
        position: u64,      // Hash position on ring
        node_id: u32,       // Physical node ID
    };

    fn init(allocator: std.mem.Allocator) Ring {
        return .{ .vnodes = std.ArrayList(VNode).init(allocator) };
    }

    fn addNode(self: *Ring, node_id: u32, vnode_count: u32) !void {
        var i: u32 = 0;
        while (i < vnode_count) : (i += 1) {
            // Hash: node_id + vnode_index
            const vnode_key = try std.fmt.allocPrint(
                self.allocator,
                "node{}-v{}",
                .{node_id, i}
            );
            defer self.allocator.free(vnode_key);

            const position = std.hash.Wyhash.hash(0, vnode_key);
            try self.vnodes.append(.{
                .position = position,
                .node_id = node_id
            });
        }

        // Sort by position for binary search
        std.sort.sort(VNode, self.vnodes.items, {}, vNodeLessThan);
    }

    fn getNode(self: *Ring, key: []const u8) u32 {
        const key_hash = std.hash.Wyhash.hash(0, key);

        // Binary search for first vnode with position >= key_hash
        var left: usize = 0;
        var right: usize = self.vnodes.items.len;

        while (left < right) {
            const mid = left + (right - left) / 2;
            if (self.vnodes.items[mid].position < key_hash) {
                left = mid + 1;
            } else {
                right = mid;
            }
        }

        // Wrap around if necessary
        const idx = if (left >= self.vnodes.items.len) 0 else left;
        return self.vnodes.items[idx].node_id;
    }

    fn vNodeLessThan(context: void, a: VNode, b: VNode) bool {
        _ = context;
        return a.position < b.position;
    }
};
```

---

### 1.3 Range-Based Sharding

**Concept:** Partition key space into ranges, assign ranges to nodes

**Example:**
```
Node0: keys starting with A-F
Node1: keys starting with G-M
Node2: keys starting with N-S
Node3: keys starting with T-Z
```

Or numeric ranges:
```
Node0: user IDs 0-999999
Node1: user IDs 1000000-1999999
Node2: user IDs 2000000-2999999
```

**Pros:**
- ✅ Range queries are efficient (all data on one node)
- ✅ Predictable data locality
- ✅ Easy to reason about

**Cons:**
- ❌ **Hotspots** if access pattern is uneven
  - If most users start with 'A', Node0 is overloaded
- ❌ Requires knowledge of key distribution
- ❌ Manual rebalancing needed
- ❌ Not suitable for random key-value cache

**Best For:**
- Time-series data (by timestamp ranges)
- Known sequential access patterns
- Data warehousing, not caching

**Not Recommended for ZeekDB** (cache workload is typically random)

---

### 1.4 Directory-Based Sharding (Centralized Mapping)

**Concept:** Maintain a mapping table: `key/prefix → node`

**Example Directory:**
```
user:* → Node0
session:* → Node1
product:* → Node2
```

Or hash-to-node table:
```
Hash range 0x0000-0x3FFF → Node0
Hash range 0x4000-0x7FFF → Node1
Hash range 0x8000-0xBFFF → Node2
Hash range 0xC000-0xFFFF → Node3
```

**Pros:**
- ✅ Flexible assignment
- ✅ Can optimize based on access patterns
- ✅ Easy to rebalance (update directory)

**Cons:**
- ❌ **Single point of failure** (directory)
- ❌ Extra network hop (lookup in directory)
- ❌ Latency overhead
- ❌ Directory must be highly available (replicated)

**Optimization: Client-Side Caching**
- Cache directory locally
- Subscribe to directory updates
- Reduces lookup overhead

**Best For:**
- Complex routing rules
- Need to colocate related data
- Can afford directory infrastructure

**Implementation:**
```zig
const Directory = struct {
    // Map hash range to node
    ranges: []struct { start: u64, end: u64, node_id: u32 },

    fn getNode(self: *Directory, key: []const u8) u32 {
        const hash = std.hash.Wyhash.hash(0, key);

        for (self.ranges) |range| {
            if (hash >= range.start and hash <= range.end) {
                return range.node_id;
            }
        }

        unreachable; // Should never happen if ranges cover full space
    }
};
```

---

### 1.5 Jump Consistent Hash

**Concept:** Algorithmic alternative to ring-based consistent hashing, uses mathematical formula

**Algorithm:**
```
fn jumpConsistentHash(key: u64, num_buckets: i32) i32 {
    var b: i64 = -1;
    var j: i64 = 0;

    while (j < num_buckets) {
        b = j;
        key = key * 2862933555777941757 + 1;
        j = @floatToInt(i64, @intToFloat(f64, b + 1) *
            (@intToFloat(f64, @as(i64, 1) << 31) /
             @intToFloat(f64, (key >> 33) + 1)));
    }

    return @intCast(i32, b);
}
```

**Pros:**
- ✅ No memory overhead (pure computation)
- ✅ Fast (O(ln n) loop iterations)
- ✅ Minimal key migration on resize
- ✅ Perfect distribution

**Cons:**
- ❌ **Can only add nodes at end** (num_buckets++)
- ❌ Cannot remove arbitrary nodes (only last one)
- ❌ Less flexible than ring-based
- ❌ Requires numerical node IDs (0, 1, 2, ...)

**Best For:**
- Append-only cluster growth
- Static node IDs
- Performance-critical routing

**Comparison:**
| Strategy | Memory | Routing Speed | Add Node Cost | Remove Node Cost | Flexibility |
|----------|--------|---------------|---------------|------------------|-------------|
| Modulo | O(1) | O(1) | O(N) rehash | O(N) rehash | Low |
| Consistent Hash Ring | O(V*N) | O(log V*N) | O(1/N) rehash | O(1/N) rehash | High |
| Jump Hash | O(1) | O(log N) | O(1/N) rehash | O(1) (last only) | Medium |
| Directory | O(R) | O(R) or O(log R) | O(1) | O(1) | Very High |

*V = vnodes per node, N = nodes, R = ranges*

---

## 2. Data Partitioning Strategies

### 2.1 No Replication (Sharding Only)

**Approach:** Each key exists on exactly one node

**Pros:**
- ✅ Maximum memory efficiency
- ✅ Simple implementation
- ✅ No consistency issues

**Cons:**
- ❌ **No fault tolerance** - node failure = data loss
- ❌ No redundancy
- ❌ Cannot survive network partitions

**Use Case:** Can tolerate cache misses, source of truth elsewhere (database)

---

### 2.2 Master-Slave Replication

**Approach:** Primary node owns data, replicas copy from master

**Write Flow:**
1. Client writes to master
2. Master acknowledges immediately
3. Master replicates to slaves asynchronously

**Read Flow:**
- Strong consistency: Read from master only
- Eventual consistency: Read from any replica

**Pros:**
- ✅ Read scalability (distribute reads)
- ✅ Fault tolerance (promote slave on master failure)
- ✅ Simple consistency model

**Cons:**
- ❌ Single master bottleneck for writes
- ❌ Replication lag (eventual consistency)
- ❌ Failover complexity

**Replication Factor:**
- **RF=2**: 1 master + 1 replica (survives 1 failure)
- **RF=3**: 1 master + 2 replicas (survives 2 failures)

**Implementation:**
```zig
const ReplicaSet = struct {
    primary: *Node,
    replicas: []*Node,

    fn write(self: *ReplicaSet, key: []const u8, value: []const u8) !void {
        // Write to primary
        try self.primary.set(key, value);

        // Async replicate to slaves
        for (self.replicas) |replica| {
            replica.asyncSet(key, value) catch |err| {
                // Log error, replica will catch up later
                std.log.err("Replication failed: {}", .{err});
            };
        }
    }

    fn read(self: *ReplicaSet, key: []const u8, allow_stale: bool) ?[]const u8 {
        if (allow_stale and self.replicas.len > 0) {
            // Load balance across replicas
            const idx = @mod(std.crypto.random.int(usize), self.replicas.len);
            return self.replicas[idx].get(key);
        }

        // Strong consistency: read from primary
        return self.primary.get(key);
    }
};
```

---

### 2.3 Multi-Master Replication (Leaderless)

**Approach:** All nodes are equal, client writes to multiple nodes

**Quorum-Based Writes/Reads:**
- **W** = write quorum (must acknowledge)
- **R** = read quorum (must respond)
- **N** = replication factor

**Consistency Guarantee:** If `W + R > N`, reads see latest write

**Example (N=3, W=2, R=2):**
```
Write "key1=value1":
- Client sends to Node0, Node1, Node2
- Waits for 2 ACKs (W=2)
- Once 2 nodes confirm, write is successful

Read "key1":
- Client queries Node0, Node1, Node2
- Waits for 2 responses (R=2)
- Takes latest value (by timestamp/version)
```

**Tunable Consistency:**
| W | R | Consistency | Use Case |
|---|---|-------------|----------|
| 1 | 1 | Eventual | High performance, low consistency |
| 2 | 2 | Strong (N=3) | Balanced |
| N | 1 | Strong write | Ensure durability |
| 1 | N | Strong read | Ensure latest value |

**Pros:**
- ✅ No single point of failure
- ✅ High availability
- ✅ Tunable consistency vs performance
- ✅ Survives network partitions

**Cons:**
- ❌ More complex
- ❌ Requires conflict resolution
- ❌ Higher write latency (wait for quorum)
- ❌ Network overhead

**Conflict Resolution Strategies:**

#### Last-Write-Wins (LWW)
- Attach timestamp to each write
- On conflict, newest timestamp wins
- Requires clock synchronization (dangerous!)

#### Vector Clocks
- Track causality with version vectors
- Can detect concurrent writes
- Application-level conflict resolution

#### CRDTs (Conflict-Free Replicated Data Types)
- Mathematically provable convergence
- Complex implementation

**Implementation Sketch:**
```zig
const QuorumCache = struct {
    nodes: []*Node,
    replication_factor: usize,
    write_quorum: usize,
    read_quorum: usize,

    fn set(self: *QuorumCache, key: []const u8, value: []const u8) !void {
        // Determine replica nodes (using consistent hashing)
        const replicas = self.getReplicaNodes(key);

        // Parallel write to all replicas
        var acks: usize = 0;
        var errors: usize = 0;

        for (replicas) |node| {
            node.set(key, value) catch {
                errors += 1;
                continue;
            };
            acks += 1;

            if (acks >= self.write_quorum) {
                return; // Success!
            }
        }

        return error.QuorumNotReached;
    }

    fn get(self: *QuorumCache, key: []const u8) ![]const u8 {
        const replicas = self.getReplicaNodes(key);

        var responses: []*Response = ...;
        var count: usize = 0;

        for (replicas) |node| {
            if (node.get(key)) |value| {
                responses[count] = ...;
                count += 1;

                if (count >= self.read_quorum) {
                    break;
                }
            }
        }

        if (count < self.read_quorum) {
            return error.QuorumNotReached;
        }

        // Resolve conflicts (take latest by timestamp)
        return self.resolveConflicts(responses[0..count]);
    }

    fn getReplicaNodes(self: *QuorumCache, key: []const u8) []*Node {
        // Use consistent hash to get primary
        const primary_idx = self.ring.getNode(key);

        // Walk ring to get N consecutive nodes
        var replicas: []*Node = undefined;
        var i: usize = 0;
        while (i < self.replication_factor) : (i += 1) {
            const idx = (primary_idx + i) % self.nodes.len;
            replicas[i] = self.nodes[idx];
        }

        return replicas;
    }
};
```

---

### 2.4 Hierarchical Sharding (Cluster of Clusters)

**Concept:** Organize nodes into groups (shards), then shard within groups

**Example:**
```
Cluster
├── Shard 0 (users A-M)
│   ├── Node 0-0 (primary)
│   ├── Node 0-1 (replica)
│   └── Node 0-2 (replica)
├── Shard 1 (users N-Z)
│   ├── Node 1-0 (primary)
│   ├── Node 1-1 (replica)
│   └── Node 1-2 (replica)
```

**Pros:**
- ✅ Scales to very large clusters
- ✅ Combines partitioning + replication
- ✅ Reduces coordination overhead

**Cons:**
- ❌ Complex topology
- ❌ Cross-shard operations expensive
- ❌ Harder to rebalance

**Best For:**
- Very large deployments (hundreds of nodes)
- Clear data partitioning strategy

---

## 3. Node Discovery and Membership

### 3.1 Static Configuration

**Approach:** Hardcode node list in config file

```toml
[cluster]
nodes = [
    "node0:7000",
    "node1:7000",
    "node2:7000"
]
```

**Pros:**
- ✅ Simple
- ✅ No external dependencies
- ✅ Predictable

**Cons:**
- ❌ Manual updates on topology change
- ❌ Not dynamic
- ❌ Requires restart on change

**Best For:** Small, stable clusters

---

### 3.2 Gossip Protocol (Epidemic)

**Approach:** Nodes periodically exchange membership information

**Algorithm:**
1. Every T seconds, each node picks random peer
2. Exchange membership state (who's alive, who's dead)
3. Update local membership view
4. Detect failures via heartbeat timeout

**Example (SWIM Protocol):**
- Node0 pings Node1
- If no response after timeout, ask Node2 to ping Node1 (indirect probe)
- If still no response, mark Node1 as suspected
- After grace period, declare Node1 dead
- Gossip this information to cluster

**Pros:**
- ✅ Fully decentralized
- ✅ Scales to large clusters
- ✅ Robust to network partitions
- ✅ Automatic failure detection

**Cons:**
- ❌ Eventually consistent membership
- ❌ False positives (network hiccup → marked dead)
- ❌ More complex implementation

**Implementation Libraries:**
- Hashicorp Serf (Go) - could inspire Zig implementation
- SWIM paper (https://www.cs.cornell.edu/projects/Quicksilver/public_pdfs/SWIM.pdf)

---

### 3.3 Centralized Coordinator (Raft/Paxos)

**Approach:** Use consensus protocol for membership

**Components:**
- Leader election
- Log replication
- Membership changes as log entries

**Pros:**
- ✅ Strong consistency on membership
- ✅ No split-brain scenarios
- ✅ Well-studied algorithms

**Cons:**
- ❌ Complex implementation
- ❌ Leader is single point of contention
- ❌ Requires quorum for operations

**Existing Solutions:**
- etcd (Raft-based, Go)
- Consul (Raft-based, Go)
- ZooKeeper (Zab/Paxos-like, Java)

**Best For:**
- Need strong consistency guarantees
- Can use external coordinator

---

### 3.4 Hybrid: Service Discovery + Gossip

**Approach:**
- Use external service (etcd/Consul) for bootstrapping
- Use gossip for heartbeats and failure detection

**Pros:**
- ✅ Best of both worlds
- ✅ Fast convergence
- ✅ Reliable bootstrap

---

## 4. Rebalancing and Migration

### 4.1 Trigger Conditions

**When to Rebalance:**
- Node added to cluster
- Node removed (planned shutdown)
- Node failed (unplanned)
- Load imbalance detected
- Manual trigger (operator command)

---

### 4.2 Migration Strategies

#### Strategy A: Stop-the-World Migration
1. Pause all writes
2. Migrate data
3. Update routing
4. Resume writes

**Pros:** Simple, consistent
**Cons:** Downtime

#### Strategy B: Online Migration (Dual-Write)
1. Start migrating in background
2. New writes go to both old and new node
3. Once migration complete, switch routing
4. Stop dual-write, remove old copy

**Phases:**
```
Phase 1: Normal operation
  Write(key) → Node0

Phase 2: Dual-write during migration
  Write(key) → Node0 AND Node1 (new owner)
  Read(key) → Node0 (authoritative) OR Node1 (if not in Node0)

Phase 3: Switch over
  Write(key) → Node1

Phase 4: Cleanup
  Delete from Node0
```

**Pros:**
- ✅ Zero downtime
- ✅ No data loss

**Cons:**
- ❌ Complex state machine
- ❌ Extra write overhead during migration
- ❌ Requires coordination

#### Strategy C: Lazy Migration (Redirect on Access)
1. Update routing immediately
2. Keys remain on old node
3. On access, migrate key lazily
4. Eventually all keys migrated

**Pros:**
- ✅ Fast rebalancing
- ✅ No upfront migration cost

**Cons:**
- ❌ Read latency during migration (redirect hop)
- ❌ Data fragmented temporarily

---

### 4.3 Migration Rate Limiting

**Throttle migration to avoid:**
- CPU saturation
- Network saturation
- Impacting live traffic

**Strategies:**
- Token bucket rate limiter
- Adaptive rate (slow down if latency increases)
- Schedule during off-peak hours

---

## 5. Failure Handling

### 5.1 Node Failure Detection

**Heartbeat-Based:**
- Periodic pings between nodes
- Mark as failed after N missed heartbeats
- Tunable timeout (trade-off: false positives vs detection speed)

**Phi Accrual Failure Detector:**
- Statistical approach (Cassandra uses this)
- Calculate "suspicion level" based on heartbeat intervals
- Adaptive to network conditions

---

### 5.2 Failure Recovery

#### Replica Promotion (Master-Slave)
1. Detect master failure
2. Elect new master from replicas
3. Update routing to new master
4. Replicate to new replica to restore RF

#### Quorum Continues (Leaderless)
- With RF=3, W=2, R=2:
  - 1 node failure: still have quorum (2 nodes)
  - Continue serving requests
- Eventually restore failed node or add new node

---

### 5.3 Split-Brain Prevention

**Problem:** Network partition isolates nodes, each side thinks other is dead

**Prevention:**
- Quorum-based decisions (majority must agree)
- Fencing (only one side can operate)
- External arbitrator (tie-breaker service)

---

## 6. Recommended Architecture for ZeekDB

### Phase 1-2: Single Node (Baseline)
- No clustering
- Build solid single-node cache

### Phase 3: Basic Clustering
**Sharding:** Consistent hashing with virtual nodes
**Replication:** Master-slave (RF=2 or 3)
**Discovery:** Static config or simple gossip
**No rebalancing yet** (manual operations)

### Phase 4: Production Clustering
**Sharding:** Consistent hashing (proven)
**Replication:** Quorum-based (tunable consistency)
**Discovery:** Gossip protocol (SWIM)
**Rebalancing:** Online migration with dual-write
**Monitoring:** Cluster health, rebalancing progress

---

## 7. Implementation Roadmap

### Minimal Viable Cluster (MVC)

#### Components Needed:

1. **Node Communication Layer**
   ```zig
   const NodeClient = struct {
       fn connect(address: []const u8, port: u16) !NodeClient;
       fn get(key: []const u8) !?[]const u8;
       fn set(key: []const u8, value: []const u8) !void;
       fn ping() !void;
   };
   ```

2. **Consistent Hash Ring**
   ```zig
   const Ring = struct {
       fn addNode(node_id: u32, vnodes: u32) !void;
       fn removeNode(node_id: u32) !void;
       fn getNode(key: []const u8) u32;
       fn getReplicaNodes(key: []const u8, count: usize) []u32;
   };
   ```

3. **Cluster Manager**
   ```zig
   const Cluster = struct {
       local_cache: Cache,
       ring: Ring,
       nodes: std.AutoHashMap(u32, NodeClient),

       fn get(key: []const u8) !?[]const u8;
       fn set(key: []const u8, value: []const u8) !void;
       fn delete(key: []const u8) !bool;
   };
   ```

4. **Simple Replication**
   ```zig
   fn set(self: *Cluster, key: []const u8, value: []const u8) !void {
       const replicas = self.ring.getReplicaNodes(key, self.replication_factor);

       for (replicas) |node_id| {
           if (node_id == self.local_node_id) {
               try self.local_cache.set(key, value);
           } else {
               const client = self.nodes.get(node_id).?;
               try client.set(key, value);
           }
       }
   }
   ```

---

## 8. Trade-Offs Summary

| Aspect | Simple Approach | Advanced Approach | Recommendation |
|--------|-----------------|-------------------|----------------|
| **Sharding** | Modulo hash | Consistent hash + vnodes | Consistent hash |
| **Replication** | No replication | Quorum replication | Master-slave → Quorum |
| **Discovery** | Static config | Gossip protocol | Static → Gossip |
| **Consistency** | Single master | Tunable quorum | Start strong, allow tuning |
| **Rebalancing** | Manual | Automatic online | Manual → Automatic |

---

## 9. Open Questions

1. **Should we support multi-datacenter deployment?**
   - Adds latency considerations
   - Requires rack/zone awareness

2. **How to handle network partitions?**
   - AP (availability) vs CP (consistency) trade-off
   - Recommend: AP for cache (availability over consistency)

3. **Should clients be cluster-aware?**
   - Smart client (routes directly) vs Proxy pattern
   - Smart client is faster but more complex

4. **Monitoring and observability?**
   - Cluster health metrics
   - Rebalancing progress
   - Network partition detection

---

## 10. References

- **Amazon Dynamo Paper**: https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf
- **Consistent Hashing**: https://en.wikipedia.org/wiki/Consistent_hashing
- **SWIM Protocol**: https://www.cs.cornell.edu/projects/Quicksilver/public_pdfs/SWIM.pdf
- **Cassandra Architecture**: https://cassandra.apache.org/doc/latest/architecture/
- **Redis Cluster**: https://redis.io/docs/management/scaling/

---

## Summary

**For ZeekDB, recommended phased approach:**

1. **Phase 1-2**: Single-node cache (current focus)
2. **Phase 3**: Consistent hashing + master-slave replication + static config
3. **Phase 4**: Quorum replication + gossip discovery + online rebalancing

**Key Decisions:**
- ✅ Consistent hashing (not modulo)
- ✅ Virtual nodes (100-500 per physical node)
- ✅ Start with master-slave, migrate to quorum
- ✅ Tunable consistency (W, R, N parameters)
- ✅ Gossip for membership (eventual consistency acceptable)
- ✅ Online rebalancing with dual-write

This provides a clear path from simple to sophisticated clustering while maintaining correctness and availability.
