# Distributed Cache with Consistent Hashing - Summary

## What is Consistent Hashing?

A technique to distribute data across multiple servers where adding/removing a server only affects **~1/N keys** (not all keys).

### The Hash Ring Concept
- Servers and keys are placed on a virtual circle (0 to 2³²)
- Each key belongs to the **first server found clockwise** from its position
- When servers change, only neighboring keys move

```
        0°
         |
    S3 ──●── S1
        /   \
       /     \
      ●───────●
     270°    90°
              |
             S2
            180°
```

---

## Real-World Example: Amazon DynamoDB

Amazon uses consistent hashing for shopping carts:
- Each customer's cart always routes to the same server
- Server crash → only ~25% customers affected (not 100%)
- Black Friday scaling → add servers without disrupting everyone

### Flow
```
Customer: "john@email.com" adds item to cart

1. hash("john@email.com") = 0x5A2B
2. Find server on ring: 0x5A2B → Server C
3. John's cart ALWAYS goes to Server C
```

---

## How The Ring Is Maintained

| Component | Responsibility |
|-----------|----------------|
| **Coordinator/Router** | Stores ring config, routes requests, monitors health |
| **Data Structure** | Sorted map: position → server |
| **Health Check** | Heartbeat every few seconds, remove unresponsive nodes |

### Storage Options
- Centralized coordinator
- Gossip protocol (Cassandra, DynamoDB)
- ZooKeeper/etcd (Kafka)
- Client-side (Redis Cluster, ioredis)

---

## Redis Cluster & ioredis

### Hash Slots
Redis uses 16,384 slots instead of raw consistent hashing:
```
Slot = CRC16(key) % 16384
```

### Slot Distribution
```
Node 1: Slots 0-5460
Node 2: Slots 5461-10922
Node 3: Slots 10923-16383
```

### ioredis Routing Process
1. Calculates slot from key: `CRC16("user:123") % 16384 = 7842`
2. Looks up slot → node mapping (cached locally)
3. Sends command directly to correct node
4. Refreshes mapping on `MOVED`/`ASK` errors

### Code Example
```javascript
const Redis = require('ioredis');

const cluster = new Redis.Cluster([
  { host: '192.168.1.10', port: 6379 },
  { host: '192.168.1.11', port: 6379 },
  { host: '192.168.1.12', port: 6379 }
]);

// ioredis automatically routes to correct node
await cluster.set('user:123', 'data');
```

---

## Key Migration When Node Fails/Removed

| Scenario | Method | Downtime |
|----------|--------|----------|
| **Node crash** | Replica promoted (already has keys) | ~1-2 seconds |
| **Planned removal** | Migrate slots first, then remove | Zero |
| **Add new node** | Reshard slots to new node | Zero |

### Automatic Failover (Node Crash)
```
BEFORE: Master + Replica
  Node 2 (Master) ←──replicates──→ Node 2R (Replica)

NODE 2 CRASHES!

AFTER: Replica promoted
  Node 2R (now Master) - serves all requests
  
No keys lost - replica had a copy!
```

### Migration Commands
```bash
# Check cluster health
redis-cli --cluster check 192.168.1.10:6379

# Reshard (move slots between nodes)
redis-cli --cluster reshard 192.168.1.10:6379

# Add new node
redis-cli --cluster add-node new_host:6379 existing_host:6379

# Remove node (after emptying slots)
redis-cli --cluster del-node host:6379 <node-id>

# Rebalance slots evenly
redis-cli --cluster rebalance 192.168.1.10:6379
```

---

## LRU Eviction Policy

**LRU (Least Recently Used)** decides which keys to delete when memory is full.

### How It Works With Consistent Hashing
1. **Consistent Hashing**: Routes key to correct node
2. **LRU**: If node memory full, evict oldest unused key

```
Node memory full (1GB/1GB)

New key arrives → Need space → Delete least recently used key

user:001 (used 10 min ago) ← DELETED (LRU)
user:002 (used 5 min ago)
user:003 (used 1 min ago)
user:004 (used just now)
user:005 (NEW) ← ADDED
```

### Redis Eviction Policies
| Policy | Description |
|--------|-------------|
| `noeviction` | Return error when full |
| `allkeys-lru` | Delete LRU from all keys |
| `volatile-lru` | Delete LRU from keys with TTL |
| `allkeys-lfu` | Delete least frequently used |
| `volatile-ttl` | Delete keys closest to expiry |

### Configuration
```bash
# redis.conf
maxmemory 1gb
maxmemory-policy allkeys-lru
```

---

## Summary Table

| Concept | Question It Answers |
|---------|---------------------|
| **Consistent Hashing** | Which node stores this key? |
| **Hash Slots** | Redis implementation (16384 slots) |
| **Ring Maintenance** | How nodes join/leave gracefully |
| **Replication** | How to survive node crashes |
| **LRU** | Which key to delete when memory full |

---

## Key Takeaways

1. **Consistent hashing** minimizes data movement when cluster changes
2. **ioredis** handles routing automatically using cached slot mapping
3. **Replicas** provide instant failover without data loss
4. **Live migration** allows zero-downtime scaling
5. **LRU** manages memory within each node independently

---

## One-Liner Definitions

- **Consistent Hashing**: Maps servers and keys to a ring so each key goes to the nearest server clockwise — when servers change, only neighboring keys move.

- **LRU**: When cache memory is full, delete the key that hasn't been used for the longest time.

- **Redis Cluster**: Distributed Redis using 16384 hash slots, with automatic failover via replicas.

---

# Advanced Concepts

---

## Virtual Nodes

### The Problem: Uneven Distribution

Imagine a clock (hash ring) with only 3 servers placed at random positions:

```
              12 o'clock
                  │
      Server A ───●───────────────── 2 o'clock
     (at 11)      │                      │
                  │                      │
 9 o'clock ───────┼──────────────────────● Server B (at 3)
                  │                      
                  │                      
                  ●─────────────────── 6 o'clock
              Server C (at 5)

Keys go to NEXT server clockwise:
- Keys from 11 to 3  → Server B  (33%)
- Keys from 3 to 5   → Server C  (17%)
- Keys from 5 to 11  → Server A  (50%)  ← UNFAIR!
```

**Server A handles 50% of all keys!** That's not balanced.

### The Solution: Virtual Nodes

Instead of putting each server at **1 position**, put it at **multiple positions**:

```
Server A gets 3 positions: A1, A2, A3
Server B gets 3 positions: B1, B2, B3
Server C gets 3 positions: C1, C2, C3

Now the ring has 9 points spread evenly:

              12 o'clock
                  │
             A1 ──●
        B2 ──●    │    ●── C1
 9 o'clock ──┼────┼────┼── 3 o'clock
        C3 ──●    │    ●── A2
             B1 ──●
              6 o'clock

Now each server handles ~33% of keys!
```

### Real Example with Keys

```
WITHOUT Virtual Nodes (3 positions):
─────────────────────────────────────
Ring: [A at 100, B at 400, C at 800]

key "user:1" → hash = 150 → next server = B (at 400)
key "user:2" → hash = 250 → next server = B (at 400)
key "user:3" → hash = 350 → next server = B (at 400)
key "user:4" → hash = 450 → next server = C (at 800)
key "user:5" → hash = 500 → next server = C (at 800)

Server A: 0 keys
Server B: 3 keys  ← Overloaded!
Server C: 2 keys


WITH Virtual Nodes (9 positions):
─────────────────────────────────────
Ring: [A1=100, B1=200, C1=300, A2=400, B2=500, C2=600, A3=700, B3=800, C3=900]

key "user:1" → hash = 150 → next = B1 (200) → Server B
key "user:2" → hash = 250 → next = C1 (300) → Server C
key "user:3" → hash = 350 → next = A2 (400) → Server A
key "user:4" → hash = 450 → next = B2 (500) → Server B
key "user:5" → hash = 550 → next = C2 (600) → Server C

Server A: 1 key
Server B: 2 keys
Server C: 2 keys  ← Much more balanced!
```

### Code Example

```javascript
class HashRing {
  constructor(servers, virtualNodesPerServer = 100) {
    this.ring = new Map();
    
    for (const server of servers) {
      // Add 100 virtual nodes for each physical server
      for (let i = 0; i < virtualNodesPerServer; i++) {
        const virtualNodeKey = `${server}-v${i}`;  // "ServerA-v0", "ServerA-v1"
        const position = hash(virtualNodeKey);      // Random position on ring
        this.ring.set(position, server);            // Maps back to physical server
      }
    }
  }
}

// Usage
const ring = new HashRing(['ServerA', 'ServerB', 'ServerC'], 100);
// Ring now has 300 positions (100 each), evenly distributed
```

### Analogy

Think of a pizza with 3 people:
- **Without virtual nodes**: Cut into 3 unequal slices (one person gets half!)
- **With virtual nodes**: Cut into 30 tiny slices, give 10 to each person (equal!)

### Trade-off

| Benefit | Cost |
|---------|------|
| Even distribution | More memory to store ring |
| Better load balancing | Slightly slower lookups |
| Flexible weighting (stronger server = more virtual nodes) | More complex implementation |

---

## MOVED vs ASK Redirects

Both are Redis redirect responses, but used in different situations:

### MOVED (Permanent)

```
Scenario: Slot 7842 was migrated from Node A to Node B (migration complete)

Client → Node A: GET user:123 (slot 7842)
Node A → Client: MOVED 7842 192.168.1.11:6379

Meaning: "This slot permanently lives on Node B now"
Action:  Client updates its slot map cache, always goes to B for this slot
```

### ASK (Temporary)

```
Scenario: Slot 7842 is CURRENTLY being migrated from Node A to Node B

Client → Node A: GET user:123 (slot 7842)
Node A → Client: ASK 7842 192.168.1.11:6379

Meaning: "This specific key has moved, but migration isn't complete"
Action:  Client sends ASKING command + GET to Node B (just this once)
         Does NOT update slot map (slot still officially belongs to A)
```

### Summary

| Response | When | Client Action |
|----------|------|---------------|
| `MOVED` | Migration complete | Update cache permanently |
| `ASK` | Migration in progress | Redirect once, don't update cache |

---

## Split Brain Problem

### What is Split Brain?

Network partition causes nodes to lose contact, but both think they're the master:

```
BEFORE: Normal cluster
  Master A ←──→ Master B ←──→ Master C

NETWORK PARTITION! (C gets isolated)

Partition 1: Master A ←→ Master B
Partition 2: Master C (alone)
```

### The Danger

- Client 1 writes to Master C: `SET balance 100`
- Client 2 writes to Master A: `SET balance 200`
- When partition heals → Data inconsistency!

### How Redis Prevents This

**Minimum replicas to write** (`min-replicas-to-write`):

```bash
# redis.conf
min-replicas-to-write 1
min-replicas-max-lag 10
```

Master C is isolated → Cannot see any replicas → REFUSES writes → Returns error to client

**Cluster Quorum**: Redis Cluster requires **majority of masters** to agree for failover:
- 3 masters: need 2 to agree
- If C is alone, it can't get majority, so it stops accepting writes

---

## Hot Key Problem

### The Problem

```
Key "trending:homepage" → hash = 5500 → Always goes to Node 2

100,000 requests/second → ALL hit Node 2
Node 2 is overloaded! Other nodes are idle.
```

### Solutions

**Solution 1: Local Cache (Application Level)**
```javascript
const localCache = new Map();

async function getTrending() {
  if (localCache.has('trending')) {
    return localCache.get('trending');  // No Redis call!
  }
  const data = await redis.get('trending:homepage');
  localCache.set('trending', data);
  setTimeout(() => localCache.delete('trending'), 1000);  // 1s local TTL
  return data;
}
```

**Solution 2: Key Sharding (Multiple Copies)**
```javascript
// Spread across multiple keys
const shardId = Math.floor(Math.random() * 10);
const key = `trending:homepage:${shardId}`;  // trending:homepage:0 to :9

// Each shard goes to different node → load spread across nodes
await redis.get(key);
```

**Solution 3: Read from Replicas**
```
Reads go to replicas, not just master:
Client → Replica 2A, 2B, 2C (read trending)
Master 2 only handles writes
```

---

## LRU vs LFU

### LRU (Least Recently Used)

Evicts the key that **hasn't been accessed for the longest time**.

```
Key A: last used 1 day ago (but used 1000 times before)
Key B: last used 1 sec ago (only 5 times total)

LRU evicts: KEY A (oldest access time)
```

### LFU (Least Frequently Used)

Evicts the key with the **lowest access count**.

```
Key A: 1000 accesses total
Key B: 5 accesses total

LFU evicts: KEY B (lowest frequency)
```

### When to Use Which?

| Use Case | Best Policy |
|----------|-------------|
| General caching | LRU (recent = relevant) |
| CDN / static content | LFU (popular items stay) |
| Session data | LRU (inactive sessions expire) |
| Trending content | LFU (viral content stays cached) |

---

## Replication Lag & Data Loss

### The Problem

```
Timeline:
WRITE to Master → CRASH! → Replica promoted
SET x=100         Master dies   x=100 LOST!
                  (replication hadn't completed)
```

### Solutions

**Solution 1: WAIT Command (Synchronous Replication)**
```javascript
await redis.set('critical:data', value);
await redis.wait(1, 1000);  // Wait for 1 replica, timeout 1000ms

// Now write is on master + at least 1 replica
```

**Solution 2: min-replicas-to-write**
```bash
# redis.conf
min-replicas-to-write 1    # Require 1 replica acknowledgment
min-replicas-max-lag 10    # Replica must be within 10 seconds
```

### Trade-off

| Approach | Consistency | Performance |
|----------|-------------|-------------|
| Async replication (default) | Eventual | Fast |
| WAIT command | Strong | Slower |
| min-replicas-to-write | Strong | May reject writes if replicas down |

---

## Slot Migration Flow (Detailed)

### Scenario

Slot 7842 migrating: Node A → Node B
Key "user:123" has ALREADY moved to Node B
Client's cached map still says: Slot 7842 → Node A

### What Happens

```
1. Client sends GET user:123 to Node A (cached mapping)
2. Node A responds: ASK 7842 NodeB (key already moved)
3. Client sends ASKING + GET user:123 to Node B
4. Node B returns data
5. Client does NOT update cache yet (migration in progress)

Later, migration completes:

6. Client sends GET user:123 to Node A
7. Node A responds: MOVED 7842 NodeB (permanent)
8. Client UPDATES cache: slot 7842 → Node B
9. Future requests go directly to Node B
```

---

## Quick Reference Card

| Concept | One-Liner |
|---------|-----------|
| **Virtual Nodes** | Multiple ring positions per server for even distribution |
| **MOVED** | Permanent redirect, update your cache |
| **ASK** | Temporary redirect during migration, don't update cache |
| **Split Brain** | Network partition causing multiple masters; prevent with quorum |
| **Hot Key** | Single key overloading one node; solve with local cache or key sharding |
| **LRU** | Evict least recently accessed |
| **LFU** | Evict least frequently accessed |
| **Replication Lag** | Async replication can lose writes; use WAIT for critical data |

