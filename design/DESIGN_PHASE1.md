# Phase 1 Detailed Design: Basic In-Memory Key-Value Store

## Executive Summary

Phase 1 focuses on building a robust, correct, and well-tested in-memory key-value store without eviction policies. This serves as the foundation for all future features.

---

## 1. Data Model Design

### 1.1 Key Type Design

#### Option A: String-Only Keys
**Pros:**
- Simple, intuitive API
- Most common use case (80%+ of cache scenarios)
- Easy to debug and inspect
- Natural serialization for persistence later
- String comparison is well-defined

**Cons:**
- Requires conversion if users want numeric keys
- Every lookup requires string hashing
- Memory overhead for string storage

**Implementation:**
```zig
const Key = []const u8;  // Slice of bytes, treated as string
```

#### Option B: Arbitrary Byte Keys
**Pros:**
- Maximum flexibility (strings, binary data, serialized structs)
- Can use any hash-able data
- Useful for binary protocols
- No conversion overhead

**Cons:**
- More complex API (caller must manage byte lifetimes)
- Harder to debug (binary data not human-readable)
- Equality semantics less obvious

**Implementation:**
```zig
const Key = []const u8;  // Arbitrary bytes
// Require caller to ensure key lifetime
```

#### Option C: Generic Type Parameter Keys
**Pros:**
- Type-safe at compile time
- Can optimize for specific key types
- Zero conversion overhead
- Supports integers, enums, structs directly

**Cons:**
- More complex implementation
- Requires comptime programming
- Different cache instances for different key types
- Hash function must be provided for custom types

**Implementation:**
```zig
fn Cache(comptime K: type, comptime V: type) type {
    return struct {
        // Generic implementation
    };
}
```

#### Option D: Tagged Union Keys
**Pros:**
- Runtime polymorphism
- Single cache can handle multiple key types
- Safe type checking

**Cons:**
- Runtime overhead (tag checking)
- Larger memory footprint per key
- More complex hash function

**Implementation:**
```zig
const Key = union(enum) {
    string: []const u8,
    int: i64,
    bytes: []const u8,
};
```

**RECOMMENDATION:** **Option A (String-Only)** for Phase 1
- Covers 90% of use cases
- Simplest implementation
- Can migrate to Option B or C later if needed
- Consistent with Redis, Memcached patterns

---

### 1.2 Value Type Design

#### Option A: Opaque Byte Slices
**Pros:**
- Maximum flexibility - store anything
- No type restrictions
- Caller handles serialization/deserialization
- Zero-copy possible in some cases

**Cons:**
- No type safety
- Caller must track what type was stored
- Manual serialization burden

**Implementation:**
```zig
const Value = []const u8;
```

#### Option B: Tagged Union Values
**Pros:**
- Type-safe storage
- Can store multiple types in same cache
- Runtime type checking prevents errors
- Support common types (string, int, float, bool, bytes)

**Cons:**
- Limited to predefined types
- Memory overhead for tag
- More complex API

**Implementation:**
```zig
const Value = union(enum) {
    string: []const u8,
    int: i64,
    float: f64,
    bool: bool,
    bytes: []const u8,
    nil: void,
};
```

#### Option C: Generic Type Parameter Values
**Pros:**
- Compile-time type safety
- Zero overhead
- Can store any type

**Cons:**
- Single cache per value type
- Complex for mixed-type storage
- Caller manages serialization for complex types

**Implementation:**
```zig
fn Cache(comptime V: type) type { ... }
```

#### Option D: Allocator-Owned Opaque Pointers
**Pros:**
- Can store any type via `*anyopaque`
- Flexible size handling
- Can store large objects

**Cons:**
- No type safety at all
- Must track types externally
- Casting burden on caller

**Implementation:**
```zig
const Value = struct {
    ptr: *anyopaque,
    len: usize,
};
```

**RECOMMENDATION:** **Option A (Opaque Byte Slices)** for Phase 1
- Matches Redis model (bytes in, bytes out)
- Simple, predictable behavior
- Caller controls serialization strategy
- Can add Option B as wrapper layer later

---

### 1.3 Key-Value Entry Metadata

Every entry needs metadata for proper management:

#### Required Fields
```zig
const Entry = struct {
    key: []const u8,           // Owned key copy
    value: []const u8,         // Owned value copy
    key_hash: u64,             // Cached hash (avoid recomputing)
};
```

#### Optional Fields for Future Phases
```zig
    // For LRU (Phase 2):
    last_accessed: i64,        // Timestamp or tick count

    // For LFU (Phase 2):
    access_count: u64,         // Number of accesses

    // For TTL (Phase 2):
    expires_at: ?i64,          // Null = no expiration

    // For doubly-linked list (LRU):
    prev: ?*Entry,
    next: ?*Entry,
```

**RECOMMENDATION for Phase 1:**
- **Minimal Entry**: Only `key`, `value`, `key_hash`
- Keep it simple, add metadata in Phase 2
- Pre-calculate hash for performance

---

### 1.4 Size Limits

#### Maximum Key Size
**Options:**
- **No limit**: Accept any size (risk: memory exhaustion)
- **Soft limit**: Warn but accept (e.g., 512 bytes)
- **Hard limit**: Reject oversized keys (e.g., 64KB max)

**RECOMMENDATION:** Hard limit of **64 KB** (Redis uses 512 MB, but 64KB is more reasonable)

#### Maximum Value Size
**Options:**
- **No limit**: Risk of single large value consuming all memory
- **Soft limit**: Configurable warning threshold
- **Hard limit**: Reject large values (e.g., 1 MB, 10 MB, 512 MB)

**RECOMMENDATION:** Configurable hard limit, default **1 MB**
- Prevent single-value DoS
- Can be increased via config
- Return error on oversized values

#### Maximum Entry Count
**Options:**
- **No limit**: Until memory exhaustion (dangerous)
- **Hard limit**: Fixed maximum entries
- **Memory-based limit**: Total cache size in bytes

**RECOMMENDATION for Phase 1:** **No limit** (no eviction yet)
- Phase 2 will add capacity limits
- For now, trust caller to manage size
- Document memory risk clearly

---

## 2. Memory Management

### 2.1 Allocator Strategy

#### Option A: Single General Purpose Allocator (GPA)
**Pros:**
- Simple, standard approach
- Good general performance
- Built-in leak detection in debug builds
- Thread-safe

**Cons:**
- Fragmentation with many allocations/deallocations
- Slower than specialized allocators
- Higher memory overhead per allocation

**Usage:**
```zig
var gpa = std.heap.GeneralPurposeAllocator(.{}){};
defer _ = gpa.deinit();
const allocator = gpa.allocator();
```

#### Option B: Arena Allocator
**Pros:**
- Extremely fast allocation (just bump pointer)
- Zero fragmentation
- Free entire cache at once (single deinit)
- Perfect for write-heavy workloads

**Cons:**
- **Cannot free individual entries** (critical limitation!)
- Memory grows unbounded until arena is destroyed
- Not suitable for long-running caches with updates/deletes
- Wasteful for sparse caches

**Usage:**
```zig
var arena = std.heap.ArenaAllocator.init(std.heap.page_allocator);
defer arena.deinit();  // Frees everything at once
const allocator = arena.allocator();
```

#### Option C: FixedBufferAllocator (Stack-Based)
**Pros:**
- Fastest possible allocation (stack memory)
- Predictable performance
- No heap fragmentation
- Good for embedded/constrained systems

**Cons:**
- **Fixed size** - cannot grow
- Stack overflow risk if buffer too large
- Not suitable for variable workloads
- Must know max size upfront

**Usage:**
```zig
var buffer: [1024 * 1024]u8 = undefined;  // 1 MB buffer
var fba = std.heap.FixedBufferAllocator.init(&buffer);
const allocator = fba.allocator();
```

#### Option D: Hybrid Approach (Arena + GPA)
**Pros:**
- Fast allocation via arena
- Periodic compaction to reclaim memory
- Balance of speed and flexibility

**Cons:**
- Complex implementation
- Requires cache rebuild on compaction
- Compaction causes latency spikes

**Strategy:**
- Use Arena for allocations
- Periodically: create new arena, copy live entries, destroy old arena

#### Option E: Memory Pool (Custom)
**Pros:**
- Pre-allocate fixed-size chunks
- Extremely fast alloc/free (just pointer manipulation)
- Zero fragmentation for fixed-size entries
- Predictable memory usage

**Cons:**
- Complex implementation
- Wasted space if entry sizes vary
- Must handle pool exhaustion
- Requires custom allocator implementation

**Implementation sketch:**
```zig
const Pool = struct {
    chunks: [][]u8,
    free_list: std.ArrayList(*Chunk),

    fn allocChunk() !*Chunk { ... }
    fn freeChunk(chunk: *Chunk) void { ... }
};
```

#### Option F: Caller-Provided Allocator (Interface Pattern)
**Pros:**
- Maximum flexibility - caller chooses strategy
- Cache doesn't dictate memory policy
- Testable with different allocators
- Zig idiom (std library pattern)

**Cons:**
- Caller must understand allocators
- Cannot optimize internally for specific allocator

**Usage:**
```zig
const Cache = struct {
    allocator: std.mem.Allocator,  // Stored from caller

    pub fn init(allocator: std.mem.Allocator) Cache {
        return .{ .allocator = allocator };
    }
};
```

**RECOMMENDATION:** **Option F (Caller-Provided)** with **GPA as default example**
- Follows Zig conventions
- Flexible for different use cases
- Testable with std.testing.allocator
- Can use Arena for read-only caches, GPA for read-write

---

### 2.2 Memory Ownership & Copying

#### Key and Value Storage Strategy

#### Option A: Always Copy (Defensive Ownership)
**Pros:**
- Cache owns all data (clear ownership)
- Caller can free source data immediately
- No lifetime tracking needed
- Safe from use-after-free bugs

**Cons:**
- Memory overhead (duplicate data)
- CPU overhead (copying)
- Extra allocations

**Implementation:**
```zig
pub fn set(self: *Cache, key: []const u8, value: []const u8) !void {
    const key_copy = try self.allocator.dupe(u8, key);
    const value_copy = try self.allocator.dupe(u8, value);
    // Store copies
}
```

#### Option B: Borrow References (Caller Owns)
**Pros:**
- Zero-copy - very fast
- No extra allocations
- Minimal memory usage

**Cons:**
- **Dangerous**: Caller must ensure lifetime
- Use-after-free if caller frees data
- Difficult to reason about ownership
- Not safe for most use cases

**Implementation:**
```zig
pub fn set(self: *Cache, key: []const u8, value: []const u8) !void {
    // Store pointers directly - UNSAFE unless lifetime guaranteed
}
```

#### Option C: Hybrid - Optional Copy
**Pros:**
- Flexibility for performance-critical cases
- Safe default (copy), unsafe opt-in (borrow)

**Cons:**
- Complex API
- Easy to misuse
- Caller must understand ownership semantics

**Implementation:**
```zig
pub fn set(self: *Cache, key: []const u8, value: []const u8, copy: bool) !void {
    if (copy) {
        // Copy data
    } else {
        // Borrow - caller must ensure lifetime
    }
}
```

#### Option D: Move Semantics (Transfer Ownership)
**Pros:**
- Caller allocates, cache takes ownership
- No duplicate allocation
- Clear ownership transfer

**Cons:**
- Caller must use same allocator
- More complex contract
- Caller cannot use data after set

**Implementation:**
```zig
// Caller must allocate with same allocator
pub fn setOwned(self: *Cache, key: []const u8, value: []const u8) !void {
    // Cache takes ownership - caller must not free
}
```

**RECOMMENDATION:** **Option A (Always Copy)** for Phase 1
- Safest approach
- Clear ownership (cache owns everything)
- Predictable behavior
- Matches Redis/Memcached semantics

---

### 2.3 Memory Cleanup

#### On Delete Operation
```zig
pub fn delete(self: *Cache, key: []const u8) !bool {
    if (self.map.fetchRemove(key)) |kv| {
        // MUST free both key and value
        self.allocator.free(kv.key);
        self.allocator.free(kv.value);
        return true;
    }
    return false;
}
```

#### On Clear Operation
```zig
pub fn clear(self: *Cache) void {
    var it = self.map.iterator();
    while (it.next()) |entry| {
        self.allocator.free(entry.key_ptr.*);
        self.allocator.free(entry.value_ptr.*);
    }
    self.map.clearRetainingCapacity();  // or clearAndFree()
}
```

#### On Deinit
```zig
pub fn deinit(self: *Cache) void {
    self.clear();  // Free all entries
    self.map.deinit();  // Free hashmap structure
}
```

---

## 3. Storage Implementation

### 3.1 Underlying Data Structure

#### Option A: std.StringHashMap
**Pros:**
- Built-in, well-tested
- Optimized for string keys
- Handles hashing automatically
- Simple API

**Cons:**
- Less control over hashing
- May rehash on grow
- Generic implementation (not cache-optimized)

**Usage:**
```zig
const std = @import("std");
map: std.StringHashMap([]const u8),  // Key -> Value
```

#### Option B: std.HashMap with Custom Context
**Pros:**
- Full control over hashing and equality
- Can cache hash values
- Can optimize for cache access patterns

**Cons:**
- More boilerplate
- Must implement hash/equality functions

**Usage:**
```zig
const Context = struct {
    pub fn hash(self: @This(), key: []const u8) u64 {
        return std.hash.Wyhash.hash(0, key);
    }
    pub fn eql(self: @This(), a: []const u8, b: []const u8) bool {
        return std.mem.eql(u8, a, b);
    }
};

map: std.HashMap([]const u8, Entry, Context, 80),
```

#### Option C: std.AutoHashMap
**Pros:**
- Simple for basic types
- Automatic hashing

**Cons:**
- Rehashes keys every lookup (inefficient for strings)
- Not ideal for byte slices

**Usage:**
```zig
map: std.AutoHashMap([]const u8, []const u8),
```

#### Option D: Custom Hash Table
**Pros:**
- Complete control
- Can optimize specifically for cache
- Can implement cache-aware eviction integration
- Learning experience

**Cons:**
- Complex implementation
- Bug-prone (collision handling, resizing, etc.)
- Reinventing the wheel
- Not necessary for Phase 1

**RECOMMENDATION:** **Option B (std.HashMap with Custom Context)**
- Allows us to cache hash values in Entry
- Full control for future optimizations
- Still uses standard library foundation
- Can pre-compute hash on insert

---

### 3.2 Hash Function Selection

#### Option A: Wyhash (Default)
**Pros:**
- Zig's default hash
- Very fast (one of fastest non-crypto hashes)
- Good distribution
- Simple

**Cons:**
- Not cryptographically secure (doesn't matter for cache)

**Usage:**
```zig
const hash = std.hash.Wyhash.hash(0, key);
```

#### Option B: CityHash
**Pros:**
- Excellent distribution
- Fast for larger strings
- Widely used

**Cons:**
- Not in Zig std lib (would need to port)

#### Option C: xxHash
**Pros:**
- Extremely fast
- Good distribution
- Popular in databases

**Cons:**
- Not in Zig std lib

#### Option D: Custom Hash (Avoid)
**Cons:**
- Difficult to get distribution right
- Performance tuning required
- Not recommended

**RECOMMENDATION:** **Wyhash** (Option A)
- Already in std library
- Excellent performance
- Good enough for cache use case
- Can swap later if needed

---

### 3.3 Hash Collision Handling

Zig's HashMap already handles this, but understanding:

#### Chaining (std.HashMap uses this)
- Each bucket is a list of entries
- Multiple entries with same hash bucket
- Slower with many collisions

#### Open Addressing
- Find next available bucket
- Better cache locality
- More complex deletion

**For Phase 1:** Trust std.HashMap's implementation (chaining with Robin Hood hashing)

---

### 3.4 Load Factor and Resizing

#### Load Factor
- Ratio: `entries / buckets`
- std.HashMap default: 80% (0.8)
- Higher = better memory usage, worse performance
- Lower = worse memory usage, better performance

#### Resizing Strategy
- std.HashMap doubles size when load factor exceeded
- Rehashes all entries (expensive operation)

#### Optimization Opportunities (Phase 2+)
- Pre-allocate if max size known: `map.ensureTotalCapacity(n)`
- Avoid resize during hot path

**For Phase 1:** Use HashMap defaults (80% load factor)

---

## 4. API Design

### 4.1 Core Operations

#### GET Operation

**Signature Options:**

Option A: Return optional
```zig
pub fn get(self: *Cache, key: []const u8) ?[]const u8
```
- Simple, idiomatic Zig
- `null` means not found
- Caller checks with `if (cache.get(key)) |value| { ... }`

Option B: Return error union
```zig
pub fn get(self: *Cache, key: []const u8) ![]const u8
```
- Can distinguish "not found" from actual errors
- `error.KeyNotFound`
- More verbose caller code: `cache.get(key) catch |err| { ... }`

Option C: Return both value and found flag
```zig
pub fn get(self: *Cache, key: []const u8) struct { value: ?[]const u8, found: bool }
```
- Explicit found/not-found
- Distinguish null value from not found
- Verbose

**RECOMMENDATION:** **Option A (Return optional)**
- Most idiomatic Zig
- Simple caller code
- Phase 1 has no null values, so null = not found

**Behavior:**
- If key exists: return value slice
- If key doesn't exist: return null
- No side effects (read-only)

---

#### SET Operation

**Signature:**

```zig
pub fn set(self: *Cache, key: []const u8, value: []const u8) !void
```

**Return type:**
- `!void` - can fail due to allocation
- Error on allocation failure
- Error on oversized key/value

**Behavior:**
- If key exists: **UPDATE** value (free old value, store new copy)
- If key doesn't exist: **INSERT** new entry
- Copy both key and value
- Atomic operation (no partial state)

**Edge Cases:**
- Empty key: **Allow** (valid use case)
- Empty value: **Allow** (valid use case)
- Null key: **Compiler prevents** (slice cannot be null)
- Null value: **Compiler prevents**

---

#### DELETE Operation

**Signature Options:**

Option A: Return bool
```zig
pub fn delete(self: *Cache, key: []const u8) bool
```
- `true` if key existed and was deleted
- `false` if key didn't exist
- Simple, no errors

Option B: Return error union
```zig
pub fn delete(self: *Cache, key: []const u8) !bool
```
- Can fail (shouldn't in Phase 1)
- Allows future error conditions

**RECOMMENDATION:** **Option A (Return bool)** for Phase 1
- Deletion cannot fail in basic implementation
- Clear semantics

**Behavior:**
- If key exists: remove from map, free key and value, return true
- If key doesn't exist: no-op, return false

---

#### EXISTS Operation

**Signature:**
```zig
pub fn exists(self: *Cache, key: []const u8) bool
```

**Behavior:**
- Return true if key present
- Return false otherwise
- No side effects (read-only)
- Cheaper than `get()` if only checking presence

**Implementation:**
```zig
pub fn exists(self: *Cache, key: []const u8) bool {
    return self.map.contains(key);
}
```

---

#### SIZE Operation

**Signature:**
```zig
pub fn size(self: *Cache) usize
```

**Behavior:**
- Return current number of entries
- O(1) operation (HashMap tracks count)

**Implementation:**
```zig
pub fn size(self: *Cache) usize {
    return self.map.count();
}
```

---

#### CLEAR Operation

**Signature:**
```zig
pub fn clear(self: *Cache) void
```

**Behavior:**
- Remove all entries
- Free all keys and values
- Reset to empty state
- Cannot fail

**Implementation:**
```zig
pub fn clear(self: *Cache) void {
    var it = self.map.iterator();
    while (it.next()) |entry| {
        self.allocator.free(entry.key_ptr.*);
        self.allocator.free(entry.value_ptr.*);
    }
    self.map.clearAndFree();  // or clearRetainingCapacity()
}
```

**Choice:** `clearAndFree()` vs `clearRetainingCapacity()`
- `clearAndFree()`: Free bucket array (save memory)
- `clearRetainingCapacity()`: Keep buckets (faster re-fill)
- **Recommend:** `clearAndFree()` for Phase 1 (predictable memory)

---

### 4.2 Struct Definition

```zig
const std = @import("std");

pub const Cache = struct {
    allocator: std.mem.Allocator,
    map: std.StringHashMap([]const u8),
    max_key_size: usize,
    max_value_size: usize,

    pub const Error = error{
        KeyTooLarge,
        ValueTooLarge,
    } || std.mem.Allocator.Error;

    pub fn init(allocator: std.mem.Allocator) Cache {
        return .{
            .allocator = allocator,
            .map = std.StringHashMap([]const u8).init(allocator),
            .max_key_size = 64 * 1024,        // 64 KB
            .max_value_size = 1024 * 1024,    // 1 MB
        };
    }

    pub fn deinit(self: *Cache) void {
        self.clear();
        self.map.deinit();
    }

    pub fn get(self: *Cache, key: []const u8) ?[]const u8 {
        return self.map.get(key);
    }

    pub fn set(self: *Cache, key: []const u8, value: []const u8) Error!void {
        if (key.len > self.max_key_size) return error.KeyTooLarge;
        if (value.len > self.max_value_size) return error.ValueTooLarge;

        // Copy key and value
        const key_copy = try self.allocator.dupe(u8, key);
        errdefer self.allocator.free(key_copy);

        const value_copy = try self.allocator.dupe(u8, value);
        errdefer self.allocator.free(value_copy);

        // Check if key exists (need to free old value)
        if (self.map.fetchPut(key_copy, value_copy)) |old_entry| {
            self.allocator.free(old_entry.key);
            self.allocator.free(old_entry.value);
        }
    }

    pub fn delete(self: *Cache, key: []const u8) bool {
        if (self.map.fetchRemove(key)) |kv| {
            self.allocator.free(kv.key);
            self.allocator.free(kv.value);
            return true;
        }
        return false;
    }

    pub fn exists(self: *Cache, key: []const u8) bool {
        return self.map.contains(key);
    }

    pub fn size(self: *Cache) usize {
        return self.map.count();
    }

    pub fn clear(self: *Cache) void {
        var it = self.map.iterator();
        while (it.next()) |entry| {
            self.allocator.free(entry.key_ptr.*);
            self.allocator.free(entry.value_ptr.*);
        }
        self.map.clearAndFree();
    }
};
```

---

### 4.3 Advanced API Considerations (Phase 2+)

#### Batch Operations
```zig
pub fn getMany(self: *Cache, keys: []const []const u8) ![]?[]const u8;
pub fn setMany(self: *Cache, entries: []const Entry) !void;
pub fn deleteMany(self: *Cache, keys: []const []const u8) ![]bool;
```

#### Iterator Pattern
```zig
pub const Iterator = struct {
    inner: MapIterator,

    pub fn next(self: *Iterator) ?struct { key: []const u8, value: []const u8 } {
        if (self.inner.next()) |entry| {
            return .{ .key = entry.key_ptr.*, .value = entry.value_ptr.* };
        }
        return null;
    }
};

pub fn iterator(self: *Cache) Iterator {
    return .{ .inner = self.map.iterator() };
}
```

**Not needed for Phase 1**, but good to design for

---

## 5. Error Handling

### 5.1 Error Types

#### Allocation Errors
- `error.OutOfMemory` - from allocator

#### Validation Errors
- `error.KeyTooLarge` - key exceeds max_key_size
- `error.ValueTooLarge` - value exceeds max_value_size

#### Combined Error Set
```zig
pub const Error = error{
    KeyTooLarge,
    ValueTooLarge,
} || std.mem.Allocator.Error;
```

### 5.2 Error Handling Strategy

#### Fail Fast
- Validate inputs immediately
- Return errors early
- Don't attempt to recover from programmer errors

#### Atomic Operations
- Use `errdefer` to clean up on failure
- Leave cache in consistent state
- No partial updates

#### Error Documentation
- Document all possible errors in doc comments
- Explain when each error can occur

---

## 6. Testing Strategy

### 6.1 Unit Tests

#### Basic Functionality Tests
```zig
test "set and get" {
    var cache = Cache.init(std.testing.allocator);
    defer cache.deinit();

    try cache.set("key1", "value1");
    try std.testing.expectEqualStrings("value1", cache.get("key1").?);
}

test "get nonexistent key returns null" {
    var cache = Cache.init(std.testing.allocator);
    defer cache.deinit();

    try std.testing.expect(cache.get("nonexistent") == null);
}

test "set updates existing key" {
    var cache = Cache.init(std.testing.allocator);
    defer cache.deinit();

    try cache.set("key", "value1");
    try cache.set("key", "value2");
    try std.testing.expectEqualStrings("value2", cache.get("key").?);
    try std.testing.expectEqual(@as(usize, 1), cache.size());
}

test "delete existing key" {
    var cache = Cache.init(std.testing.allocator);
    defer cache.deinit();

    try cache.set("key", "value");
    try std.testing.expect(cache.delete("key"));
    try std.testing.expect(cache.get("key") == null);
}

test "delete nonexistent key returns false" {
    var cache = Cache.init(std.testing.allocator);
    defer cache.deinit();

    try std.testing.expect(!cache.delete("nonexistent"));
}

test "exists" {
    var cache = Cache.init(std.testing.allocator);
    defer cache.deinit();

    try std.testing.expect(!cache.exists("key"));
    try cache.set("key", "value");
    try std.testing.expect(cache.exists("key"));
}

test "size" {
    var cache = Cache.init(std.testing.allocator);
    defer cache.deinit();

    try std.testing.expectEqual(@as(usize, 0), cache.size());
    try cache.set("key1", "value1");
    try std.testing.expectEqual(@as(usize, 1), cache.size());
    try cache.set("key2", "value2");
    try std.testing.expectEqual(@as(usize, 2), cache.size());
}

test "clear" {
    var cache = Cache.init(std.testing.allocator);
    defer cache.deinit();

    try cache.set("key1", "value1");
    try cache.set("key2", "value2");
    cache.clear();
    try std.testing.expectEqual(@as(usize, 0), cache.size());
    try std.testing.expect(cache.get("key1") == null);
}
```

#### Edge Cases Tests
```zig
test "empty key" {
    var cache = Cache.init(std.testing.allocator);
    defer cache.deinit();

    try cache.set("", "value");
    try std.testing.expectEqualStrings("value", cache.get("").?);
}

test "empty value" {
    var cache = Cache.init(std.testing.allocator);
    defer cache.deinit();

    try cache.set("key", "");
    try std.testing.expectEqualStrings("", cache.get("key").?);
}

test "unicode keys and values" {
    var cache = Cache.init(std.testing.allocator);
    defer cache.deinit();

    try cache.set("🔑", "😀");
    try std.testing.expectEqualStrings("😀", cache.get("🔑").?);
}

test "large key rejected" {
    var cache = Cache.init(std.testing.allocator);
    defer cache.deinit();

    var large_key = try std.testing.allocator.alloc(u8, 65 * 1024); // 65 KB
    defer std.testing.allocator.free(large_key);

    try std.testing.expectError(error.KeyTooLarge, cache.set(large_key, "value"));
}

test "large value rejected" {
    var cache = Cache.init(std.testing.allocator);
    defer cache.deinit();

    var large_value = try std.testing.allocator.alloc(u8, 2 * 1024 * 1024); // 2 MB
    defer std.testing.allocator.free(large_value);

    try std.testing.expectError(error.ValueTooLarge, cache.set("key", large_value));
}

test "binary data" {
    var cache = Cache.init(std.testing.allocator);
    defer cache.deinit();

    const binary_data = [_]u8{ 0, 1, 2, 255, 254 };
    try cache.set("binary", &binary_data);
    try std.testing.expectEqualSlices(u8, &binary_data, cache.get("binary").?);
}
```

#### Memory Leak Tests
```zig
test "no memory leaks on multiple operations" {
    var cache = Cache.init(std.testing.allocator);
    defer cache.deinit();

    var i: usize = 0;
    while (i < 1000) : (i += 1) {
        var buf: [32]u8 = undefined;
        const key = try std.fmt.bufPrint(&buf, "key{}", .{i});
        try cache.set(key, "value");
    }

    cache.clear();
    // Allocator will detect leaks
}

test "no leaks on update" {
    var cache = Cache.init(std.testing.allocator);
    defer cache.deinit();

    var i: usize = 0;
    while (i < 100) : (i += 1) {
        try cache.set("key", "value");
    }
    // Should not leak 99 values
}
```

#### Stress Tests
```zig
test "many entries" {
    var cache = Cache.init(std.testing.allocator);
    defer cache.deinit();

    const count = 10000;
    var i: usize = 0;
    while (i < count) : (i += 1) {
        var key_buf: [32]u8 = undefined;
        const key = try std.fmt.bufPrint(&key_buf, "key{}", .{i});
        var value_buf: [32]u8 = undefined;
        const value = try std.fmt.bufPrint(&value_buf, "value{}", .{i});
        try cache.set(key, value);
    }

    try std.testing.expectEqual(@as(usize, count), cache.size());
}
```

### 6.2 Integration Tests

#### Realistic Usage Patterns
```zig
test "session cache simulation" {
    var cache = Cache.init(std.testing.allocator);
    defer cache.deinit();

    // Simulate session storage
    try cache.set("session:user123", "{ \"user_id\": 123, \"name\": \"Alice\" }");
    try cache.set("session:user456", "{ \"user_id\": 456, \"name\": \"Bob\" }");

    // Retrieve session
    const session = cache.get("session:user123").?;
    try std.testing.expect(std.mem.indexOf(u8, session, "Alice") != null);

    // Invalidate session
    try std.testing.expect(cache.delete("session:user123"));
}
```

### 6.3 Performance Tests

Not critical for Phase 1, but good to establish baseline:

```zig
test "set performance baseline" {
    var cache = Cache.init(std.testing.allocator);
    defer cache.deinit();

    const iterations = 100000;
    const start = std.time.nanoTimestamp();

    var i: usize = 0;
    while (i < iterations) : (i += 1) {
        var buf: [32]u8 = undefined;
        const key = try std.fmt.bufPrint(&buf, "key{}", .{i});
        try cache.set(key, "value");
    }

    const end = std.time.nanoTimestamp();
    const duration_ms = @divTrunc(end - start, 1_000_000);
    std.debug.print("\n{} SET operations in {}ms\n", .{ iterations, duration_ms });
}
```

### 6.4 Test Organization

```
src/
├── cache.zig          # Implementation + unit tests
├── cache_test.zig     # Additional integration tests
└── bench.zig          # Performance benchmarks
```

---

## 7. Documentation Requirements

### 7.1 Public API Documentation

Every public function needs doc comments:

```zig
/// Retrieves the value associated with the given key.
///
/// Returns the value as a byte slice if the key exists, or null if not found.
/// The returned slice is valid until the next cache modification operation.
///
/// Example:
/// ```zig
/// if (cache.get("user:123")) |value| {
///     std.debug.print("Found: {s}\n", .{value});
/// }
/// ```
pub fn get(self: *Cache, key: []const u8) ?[]const u8 { ... }

/// Stores a key-value pair in the cache.
///
/// If the key already exists, its value is updated. Both key and value are
/// copied into cache-owned memory.
///
/// Errors:
/// - `error.KeyTooLarge`: Key exceeds max_key_size (default 64 KB)
/// - `error.ValueTooLarge`: Value exceeds max_value_size (default 1 MB)
/// - `error.OutOfMemory`: Allocation failed
///
/// Example:
/// ```zig
/// try cache.set("user:123", "{ \"name\": \"Alice\" }");
/// ```
pub fn set(self: *Cache, key: []const u8, value: []const u8) Error!void { ... }
```

### 7.2 README Documentation

Create `README.md` with:
- Quick start example
- API overview
- Installation instructions
- Performance characteristics
- Limitations
- Future roadmap

---

## 8. Success Criteria for Phase 1

### Functional Requirements
- [ ] All core operations (get, set, delete, exists, size, clear) work correctly
- [ ] Handles edge cases (empty keys/values, unicode, binary data)
- [ ] Proper error handling for oversized keys/values
- [ ] Update existing keys correctly

### Quality Requirements
- [ ] Zero memory leaks (valgrind or Zig leak detector)
- [ ] Test coverage > 80%
- [ ] All public functions documented
- [ ] Passes all unit and integration tests

### Performance Requirements (Baseline)
- [ ] `get()` is O(1) average case
- [ ] `set()` is O(1) average case
- [ ] Can handle 10,000+ entries without issues
- [ ] No significant performance degradation under load

### Code Quality
- [ ] Follows Zig naming conventions
- [ ] Clear, readable code
- [ ] Proper error handling (no panics in library code)
- [ ] Idiomatic Zig patterns

---

## 9. Implementation Plan

### Step 1: Basic Structure (Day 1)
- Create `src/cache.zig`
- Define `Cache` struct
- Implement `init()` and `deinit()`
- First test: create and destroy cache

### Step 2: Core Operations (Day 2-3)
- Implement `set()`, `get()`, `delete()`
- Write tests for each operation
- Verify no memory leaks

### Step 3: Additional Operations (Day 4)
- Implement `exists()`, `size()`, `clear()`
- Write tests

### Step 4: Edge Cases & Validation (Day 5)
- Add size limit validation
- Handle edge cases (empty, unicode, binary)
- Write edge case tests

### Step 5: Documentation & Polish (Day 6)
- Add doc comments
- Write README
- Code cleanup
- Final testing

### Step 6: Benchmarking (Day 7)
- Create performance benchmarks
- Establish baseline metrics
- Document performance characteristics

---

## 10. Open Questions & Decisions Needed

### Questions to Answer Before Implementation

1. **Allocator choice for examples/tests:**
   - GPA (general purpose) or Arena for specific use cases?
   - **Recommendation:** Show both in examples

2. **Should we support zero-sized keys/values?**
   - Currently: allowed
   - **Alternative:** Reject with error
   - **Decision needed:** Allow (more flexible)

3. **Return value lifetime guarantees:**
   - Current: valid until next modification
   - **Alternative:** Caller must copy immediately
   - **Decision needed:** Document "valid until next modification"

4. **Configuration struct vs. hardcoded limits:**
   ```zig
   pub const Config = struct {
       max_key_size: usize = 64 * 1024,
       max_value_size: usize = 1024 * 1024,
       initial_capacity: usize = 16,
   };

   pub fn initWithConfig(allocator: std.mem.Allocator, config: Config) Cache { ... }
   ```
   - **Decision needed:** Add config struct for flexibility

5. **Thread safety for Phase 1:**
   - Current plan: **Not thread-safe** (document this)
   - Add mutex in Phase 3
   - **Decision needed:** Document as single-threaded for Phase 1

---

## 11. Risk Assessment

### Low Risk
- ✅ Using std.HashMap (well-tested, reliable)
- ✅ Simple memory model (copy everything)
- ✅ No concurrency in Phase 1

### Medium Risk
- ⚠️ Memory usage could grow unbounded (no eviction)
  - Mitigation: Document clearly, add in Phase 2
- ⚠️ Large value copying performance
  - Mitigation: Document size limits, benchmark

### High Risk
- ❌ None identified for Phase 1

---

## 12. Future Considerations (Phase 2+)

### Things NOT to implement in Phase 1
- ❌ Eviction policies (LRU, LFU, TTL)
- ❌ Capacity limits
- ❌ Persistence
- ❌ Networking
- ❌ Thread safety
- ❌ Statistics/monitoring
- ❌ Batch operations
- ❌ Iterator pattern

### Architecture Decisions That Support Phase 2
- ✅ Entry struct can be extended with metadata
- ✅ Allocator pattern supports different strategies
- ✅ Error handling extensible
- ✅ Config struct can add capacity limits

---

## Summary

**Core Decisions:**
1. **Keys:** String-only ([]const u8)
2. **Values:** Opaque byte slices ([]const u8)
3. **Memory:** Caller-provided allocator, always-copy semantics
4. **Storage:** std.HashMap with custom context
5. **Hash:** Wyhash
6. **Limits:** 64 KB keys, 1 MB values (configurable)
7. **Thread Safety:** None (Phase 1)
8. **Eviction:** None (Phase 1)

**Next Step:** Begin implementation with Step 1 (Basic Structure)