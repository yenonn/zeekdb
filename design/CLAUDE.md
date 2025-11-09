# ZeekDB - Key-Value Cache Database Design

## Project Overview

A key-value cache database implementation in Zig, focusing on performance, memory safety, and clean architecture.

---

## Architecture Design

### Core Components

#### 1. Storage Layer

**Purpose**: Manage the actual data structure holding key-value pairs

**Design Considerations**:

- Hash map for O(1) average-case lookups
- Memory allocation strategy (arena vs general-purpose allocator)
- Key collision handling
- Value size limits and memory budgets
- Support for different value types (strings, integers, binary data)

**Questions to Answer**:

- Should keys be limited to strings or support arbitrary bytes?
- How to handle large values (inline vs pointer storage)?
- Memory pooling strategy for frequently allocated/deallocated entries?
- Copy-on-write semantics or direct mutation?

---

#### 2. Cache Policies

**Purpose**: Eviction strategies when cache reaches capacity limits

**Eviction Strategies**:

- **LRU (Least Recently Used)**: Evict least recently accessed items
- **LFU (Least Frequently Used)**: Evict least frequently accessed items
- **TTL (Time To Live)**: Auto-expire entries after time threshold
- **FIFO**: Simple queue-based eviction
- **Random**: Random eviction (low overhead)

**Design Considerations**:

- Metadata overhead per entry (access time, frequency counter, etc.)
- Performance trade-offs (eviction speed vs accuracy)
- Pluggable policy system vs hardcoded
- Combination policies (LRU + TTL hybrid)
- Background vs on-demand expiration for TTL

**Questions to Answer**:

- Should multiple policies be supported simultaneously?
- How to efficiently track access patterns without significant overhead?
- Lazy vs eager cleanup for expired entries?

---

#### 3. API Layer

**Purpose**: Public interface for cache operations

**Core Operations**:

- `get(key)` - Retrieve value by key
- `set(key, value)` - Store or update key-value pair
- `delete(key)` - Remove entry
- `clear()` - Remove all entries
- `exists(key)` - Check key presence
- `size()` - Current entry count

**Advanced Operations**:

- `setWithTTL(key, value, ttl)` - Set with expiration
- `getMany(keys)` - Batch retrieval
- `setMany(entries)` - Batch insertion
- `keys()` - List all keys
- `values()` - List all values
- `stats()` - Cache statistics (hit rate, evictions, etc.)

**Design Considerations**:

- Error handling strategy (return error unions vs optionals)
- API thread safety guarantees
- Const vs mutable operations
- Iterator pattern for large result sets

---

#### 4. Optional Advanced Features

##### 4.1 Persistence

**Purpose**: Save cache state to disk for durability

**Approaches**:

- **Snapshot**: Periodic full dump to disk
- **AOF (Append-Only File)**: Log every write operation
- **Hybrid**: Snapshots + AOF for recovery
- **Memory-mapped files**: Direct memory-to-disk mapping

**Design Considerations**:

- Serialization format (custom binary, JSON, MessagePack)
- Crash recovery mechanism
- Performance impact on write operations
- Compaction strategy for AOF logs

---

##### 4.2 Networking Layer

**Purpose**: Remote access via network protocols

**Protocol Options**:

- **Redis Protocol (RESP)**: Compatible with Redis clients
- **HTTP/REST**: Simple, widely supported
- **Custom binary protocol**: Optimized for performance
- **gRPC**: Structured RPC framework

**Design Considerations**:

- Connection pooling and management
- Request parsing and serialization
- Authentication and authorization
- Rate limiting and DOS protection

---

##### 4.3 Concurrency & Thread Safety

**Purpose**: Support concurrent access from multiple threads

**Synchronization Strategies**:

- **Coarse-grained locking**: Single mutex for entire cache
- **Fine-grained locking**: Per-bucket or per-entry locks
- **Lock-free structures**: Atomic operations and CAS
- **MVCC (Multi-Version Concurrency Control)**: Multiple versions of data

**Design Considerations**:

- Read vs write workload characteristics
- Lock contention hotspots
- Deadlock prevention
- Performance scalability with thread count

---

##### 4.4 Monitoring & Statistics

**Purpose**: Observability and performance metrics

**Metrics to Track**:

- Hit rate / miss rate
- Eviction count and reasons
- Memory usage (current, peak, average)
- Operation latencies (p50, p95, p99)
- Key distribution and hotspots

**Design Considerations**:

- Metrics storage overhead
- Real-time vs aggregated stats
- Export format (Prometheus, JSON, etc.)

---

## Implementation Phases

### Phase 1: Basic In-Memory Store

**Goal**: Simple, working key-value store without eviction

**Tasks**:

- Design Cache struct with HashMap storage
- Implement get/set/delete operations
- Proper memory management with allocators
- Comprehensive test suite
- Handle edge cases (null keys, empty values)

**Success Criteria**:

- All basic operations work correctly
- No memory leaks (valgrind/sanitizer clean)
- Test coverage > 80%

---

### Phase 2: Add Cache Semantics

**Goal**: Implement cache behavior with size limits and eviction

**Tasks**:

- Add max size/capacity limits
- Implement LRU eviction policy
- Add TTL support with expiration
- Benchmark performance characteristics
- Stress testing with large datasets

**Success Criteria**:

- Eviction works correctly under memory pressure
- TTL entries expire as expected
- Performance remains O(1) for get/set

---

### Phase 3: Polish & Advanced Features

**Goal**: Production-ready features

**Tasks**:

- Add persistence layer (snapshot or AOF)
- Implement networking with chosen protocol
- Thread safety with appropriate synchronization
- Monitoring and statistics
- Documentation and examples

**Success Criteria**:

- Can survive crashes and recover state
- Handles concurrent access safely
- Comprehensive documentation

---

## Design Questions to Resolve

### Data Model

- [ ] What types should keys support? (strings only, arbitrary bytes, integers)
- [ ] What types should values support? (strings, binary blobs, typed data)
- [ ] Should we support nested structures or just flat key-value?
- [ ] Maximum key/value size limits?

### Memory Management

- [ ] Which allocator strategy? (GPA, Arena, custom pool)
- [ ] How to handle memory exhaustion? (fail operations vs force eviction)
- [ ] Memory overhead per entry (acceptable percentage)
- [ ] Should we support different allocators per cache instance?

### Eviction Policy

- [ ] Default eviction policy? (LRU recommended)
- [ ] Should policy be configurable at runtime or compile-time?
- [ ] Hybrid policies? (LRU + TTL combined)
- [ ] Background eviction thread vs on-demand?

### Persistence

- [ ] Persistence required for MVP or Phase 3?
- [ ] Snapshot vs AOF vs hybrid?
- [ ] Synchronous writes (slow, safe) vs async (fast, risky)?
- [ ] File format compatibility/versioning strategy?

### Networking

- [ ] Single-threaded event loop or thread pool?
- [ ] Which protocol provides best balance of compatibility and performance?
- [ ] Connection limits and resource management?
- [ ] TLS/encryption support needed?

### Performance Targets

- [ ] Expected throughput? (ops/second)
- [ ] Latency requirements? (p99 latency target)
- [ ] Max cache size to support? (MB/GB)
- [ ] Number of concurrent connections to support?

---

## Code Structure (Proposed)

```
src/
├── main.zig              # CLI/server entry point
├── root.zig              # Library public API exports
├── cache.zig             # Main Cache struct and core logic
├── storage/
│   ├── hashmap.zig       # HashMap wrapper with optimizations
│   └── entry.zig         # Cache entry metadata
├── eviction/
│   ├── lru.zig           # LRU policy implementation
│   ├── lfu.zig           # LFU policy implementation
│   ├── ttl.zig           # TTL expiration logic
│   └── policy.zig        # Eviction policy interface
├── persistence/
│   ├── snapshot.zig      # Snapshot-based persistence
│   └── aof.zig           # Append-only file logging
├── network/
│   ├── server.zig        # Network server
│   ├── protocol.zig      # Protocol parser/serializer
│   └── connection.zig    # Connection management
├── sync/
│   └── mutex.zig         # Thread synchronization utilities
└── stats/
    └── metrics.zig       # Statistics and monitoring
```

---

## References & Inspiration

- Redis: <https://redis.io/>
- Memcached: <https://memcached.org/>
- LRU Cache implementations
- Zig standard library HashMap: `std.HashMap`, `std.StringHashMap`
- Zig allocators: `std.mem.Allocator`, `std.heap.GeneralPurposeAllocator`

---

## Next Steps

1. Finalize design decisions (answer questions above)
2. Create detailed API specification
3. Begin Phase 1 implementation with basic store
4. Iterate based on testing and benchmarking
