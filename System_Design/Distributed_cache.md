# Distributed Cache - Complete System Design Guide

**A Comprehensive Reference for Understanding and Designing Distributed Cache Systems**

-----

## Table of Contents

1. [Introduction & Motivation](#1-introduction--motivation)
1. [The Core Problem](#2-the-core-problem)
1. [Requirements & Capacity Planning](#3-requirements--capacity-planning)
1. [High-Level Architecture](#4-high-level-architecture)
1. [Data Partitioning & Consistent Hashing (DETAILED)](#5-data-partitioning--consistent-hashing)
1. [Eviction Policies](#6-eviction-policies)
1. [Cache Strategies & Patterns](#7-cache-strategies--patterns)
1. [High Availability & Replication](#8-high-availability--replication)
1. [Advanced Problems & Solutions](#9-advanced-problems--solutions)
1. [Real-World Case Studies](#10-real-world-case-studies)
1. [Interview Framework](#11-interview-framework)

-----

## 1. Introduction & Motivation

### 1.1 Why Do We Need Distributed Cache?

**The Database Bottleneck Problem:**

When building applications at scale, the database becomes the primary bottleneck:

```
Without Cache - Every Request Hits Database:
┌──────┐      ┌──────────┐      ┌──────────┐
│ User │─────▶│  Server  │─────▶│ Database │
└──────┘      └──────────┘      └──────────┘
              
Latency: 50-100ms per request
Database Load: 100% of traffic
Scalability: Limited by database capacity
Cost: Expensive database scaling
```

```
With Cache - Most Requests Hit Cache:
┌──────┐      ┌──────────┐      ┌─────────┐
│ User │─────▶│  Server  │─────▶│  Cache  │ ← 90% requests (< 1ms)
└──────┘      └──────────┘      └────┬────┘
                                     │
                              Cache Miss (10%)
                                     │
                                     ▼
                              ┌──────────┐
                              │ Database │
                              └──────────┘

Latency: 1ms for cache hits
Database Load: 10% of traffic
Scalability: Horizontally scalable cache
Cost: RAM cheaper than database scaling
```

### 1.2 Key Benefits

|Benefit                |Impact                     |Example                         |
|-----------------------|---------------------------|--------------------------------|
|**Reduced Latency**    |50-100ms → <1ms            |User profile loads 50x faster   |
|**Database Offloading**|80-95% traffic reduction   |Database handles 10x more users |
|**Horizontal Scaling** |Add cache nodes easily     |Instagram: 100s of cache nodes  |
|**Cost Efficiency**    |RAM cheaper than DB scaling|$1000 cache vs $10000 DB upgrade|

-----

## 2. The Core Problem

### 2.1 Understanding Database Architecture

**How Databases Work:**

```
Database Server:
┌───────────────────────────────────────┐
│  Connection Pool (limited slots)      │
│  ┌───┐ ┌───┐ ┌───┐  ... ┌───┐       │
│  │ 1 │ │ 2 │ │ 3 │      │100│       │
│  └───┘ └───┘ └───┘      └───┘       │
├───────────────────────────────────────┤
│  Query Parser & Optimizer             │
├───────────────────────────────────────┤
│  Buffer Pool (DB's Internal Cache)    │
│  - Frequently accessed pages in RAM   │
│  - Still requires query processing    │
├───────────────────────────────────────┤
│  Disk Storage (Persistent Data)       │
│  - Read: 5-10ms per query            │
│  - Write: 10-50ms                     │
└───────────────────────────────────────┘
```

**Cost of Database Query (even if data in DB cache):**

```
Total Latency Breakdown:
├─ Network round trip: 1-2ms
├─ Connection acquisition: 1ms
├─ Query parsing: 1-2ms
├─ Query optimization: 2-5ms
├─ Buffer pool lookup: 1-2ms
├─ Result serialization: 1-2ms
├─ Network return: 1-2ms
└─ TOTAL: 10-15ms minimum

Compare to Redis Cache:
├─ Network round trip: 1-2ms
├─ Hash table lookup: 0.1ms
├─ Network return: 1-2ms
└─ TOTAL: <5ms (3x faster!)
```

### 2.2 Why Not Cache in Application Server?

**Problem: Each Server Has Its Own Cache**

```
Load Balancer
     │
     ├──────────┬──────────┬──────────┐
     │          │          │          │
┌────▼────┐ ┌──▼──────┐ ┌─▼───────┐ ┌▼─────────┐
│Server 1 │ │Server 2 │ │Server 3 │ │Server N  │
│ ┌─────┐ │ │ ┌─────┐ │ │ ┌─────┐ │ │ ┌─────┐  │
│ │Cache│ │ │ │Cache│ │ │ │Cache│ │ │ │Cache│  │
│ └─────┘ │ │ └─────┘ │ │ └─────┘ │ │ └─────┘  │
└─────────┘ └─────────┘ └─────────┘ └──────────┘
     │          │          │          │
     └──────────┴──────────┴──────────┘
                   │
            ┌──────▼──────┐
            │  Database   │
            └─────────────┘
```

**Issues with Local Caching:**

**Problem 1: Cache Duplication & Inefficiency**

```
Request Flow:
1. User A → Load Balancer → Server 1
   - Fetches "user:123" from DB
   - Caches in Server 1

2. User B → Load Balancer → Server 2  
   - Needs same "user:123"
   - Cache MISS (not in Server 2's cache)
   - Fetches from DB again!

3. User C → Load Balancer → Server 3
   - Cache MISS again
   - Fetches from DB third time!

Result: Same data cached 3 times, DB queried 3 times!
```

**Problem 2: Data Inconsistency**

```
1. User updates profile via Server 1
   - Server 1 updates DB
   - Server 1 updates its cache

2. User's friend requests profile via Server 2
   - Server 2 still has OLD cached data
   - Shows stale information!

Result: Users see different data depending on which server they hit!
```

### 2.3 The Solution: Distributed Cache

**Shared Cache Architecture:**

```
┌──────────┬──────────┬──────────┬──────────┐
│Server 1  │Server 2  │Server 3  │Server N  │
└────┬─────┴────┬─────┴────┬─────┴────┬─────┘
     │          │          │          │
     └──────────┴────┬─────┴──────────┘
                     │
         ┌───────────▼──────────┐
         │  Distributed Cache   │
         │  (Redis Cluster)     │
         │  ┌─────────────────┐ │
         │  │  Shared Memory  │ │
         │  │  All servers    │ │
         │  │  access same    │ │
         │  │  cached data    │ │
         │  └─────────────────┘ │
         └──────────┬───────────┘
                    │
             ┌──────▼──────┐
             │  Database   │
             └─────────────┘

Benefits:
✓ No duplication - data cached once
✓ Consistency - all servers see same data
✓ Efficiency - single DB fetch for popular data
```

-----

## 3. Requirements & Capacity Planning

### 3.1 Functional Requirements

**Core Operations:**

```
Basic Operations:
├─ get(key) → value              // Retrieve cached value
├─ put(key, value, ttl?)         // Store value with optional TTL
├─ delete(key)                   // Remove from cache
└─ exists(key) → boolean         // Check if key exists

Advanced Operations:
├─ increment(key, delta)         // Atomic counter operations
├─ decrement(key, delta)         // Useful for rate limiting
├─ expire(key, ttl)              // Set/update expiration
├─ multi_get([key1, key2, ...])  // Batch retrieval
└─ multi_set({k1:v1, k2:v2})    // Batch insertion
```

### 3.2 Non-Functional Requirements

|Requirement      |Target                   |Justification                                 |
|-----------------|-------------------------|----------------------------------------------|
|**Latency (p99)**|< 1ms                    |User experience - anything >10ms is noticeable|
|**Throughput**   |100K-1M QPS per node     |Handle high traffic applications              |
|**Availability** |99.9%+                   |Minimize downtime impact                      |
|**Consistency**  |Eventual                 |Trade strong consistency for performance      |
|**Durability**   |Optional                 |Cache is ephemeral, DB is source of truth     |
|**Scalability**  |Linear horizontal scaling|Add nodes to increase capacity                |

### 3.3 Capacity Estimation Example

**Scenario: Building Instagram-like Application**

**Assumptions:**

```
Active Users: 500 million
Daily Active Users (DAU): 200 million
Requests per user per day: 100
Average cache object size: 1 KB
Cache hit ratio target: 90%
Unique cacheable objects: 50 million
```

**Traffic Calculation:**

```
Total requests per day = 200M users × 100 requests
                       = 20 billion requests/day

Requests per second (QPS) = 20B / 86,400 seconds
                          ≈ 231,000 QPS

With 90% cache hit ratio:
├─ Cache handles: 231K × 0.9 = 208K QPS
└─ Database handles: 231K × 0.1 = 23K QPS
```

**Storage Calculation:**

```
Cache storage needed = 50M objects × 1 KB
                     = 50 GB

With 2x buffer for growth:
Required cache capacity = 100 GB
```

**Node Calculation:**

```
Assumptions per cache node:
├─ RAM: 64 GB
├─ Max QPS: 100,000
└─ Network: 1 Gbps

For QPS requirement (208K QPS):
Nodes needed = 208K / 100K = 2.08 ≈ 3 nodes

For storage requirement (100 GB):
Nodes needed = 100 GB / 64 GB = 1.56 ≈ 2 nodes

For redundancy (3x replication):
Nodes needed = 3 × 3 = 9 nodes

Final: 9 nodes (with replication)
Or: 3 nodes (without replication, higher risk)
```

-----

## 4. High-Level Architecture

### 4.1 Complete System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         CLIENT LAYER                            │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │  Web App │  │ Mobile   │  │  API     │  │ Service  │       │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘       │
└───────┼─────────────┼─────────────┼─────────────┼──────────────┘
        │             │             │             │
┌───────┴─────────────┴─────────────┴─────────────┴──────────────┐
│                    LOAD BALANCER / API GATEWAY                  │
│                    (Nginx, AWS ELB, etc.)                       │
└───────┬─────────────┬─────────────┬─────────────┬──────────────┘
        │             │             │             │
┌───────┴─────────────┴─────────────┴─────────────┴──────────────┐
│                    APPLICATION SERVER LAYER                     │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Application Server 1                                    │  │
│  │  ┌────────────────────────────────────────────────────┐  │  │
│  │  │  Cache Client Library                              │  │  │
│  │  │  - Connection pooling                              │  │  │
│  │  │  - Consistent hashing (routing logic)              │  │  │
│  │  │  - Failover handling                               │  │  │
│  │  │  - Request coalescing                              │  │  │
│  │  └────────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  [Similar structure for App Server 2, 3, ... N]                │
└─────────────────────┬───────────────────────────────────────────┘
                      │
┌─────────────────────┴───────────────────────────────────────────┐
│                   DISTRIBUTED CACHE LAYER                       │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │ Cache Node 1 │  │ Cache Node 2 │  │ Cache Node 3 │  ...    │
│  │  (Master)    │  │  (Master)    │  │  (Master)    │         │
│  │              │  │              │  │              │         │
│  │ Keys:        │  │ Keys:        │  │ Keys:        │         │
│  │ user:1       │  │ user:2       │  │ user:3       │         │
│  │ post:100     │  │ post:200     │  │ post:300     │         │
│  │ ...          │  │ ...          │  │ ...          │         │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘         │
│         │                 │                 │                  │
│         │ Replication     │ Replication     │ Replication      │
│         ▼                 ▼                 ▼                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │  Replica 1A  │  │  Replica 2A  │  │  Replica 3A  │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
│                                                                  │
└─────────────────────┬────────────────────────────────────────────┘
                      │
                      │ On Cache Miss
                      │
┌─────────────────────▼───────────────────────────────────────────┐
│                      DATABASE LAYER                             │
│  ┌──────────────┐                    ┌──────────────┐          │
│  │    Master    │───replication─────▶│   Replica    │          │
│  │   Database   │                    │   Database   │          │
│  └──────────────┘                    └──────────────┘          │
│                                                                  │
│  (PostgreSQL, MySQL, MongoDB, etc.)                             │
└──────────────────────────────────────────────────────────────────┘
```

### 4.2 Request Flow - Cache Hit

```
Step-by-Step: User requests "user:12345" profile

1. Client Request
   └─▶ GET /api/users/12345

2. Load Balancer
   └─▶ Routes to Application Server 2

3. Application Server 2
   ├─ Cache client library initializes
   ├─ Computes: hash("user:12345") = 0x7A3F92B1
   ├─ Consistent hashing determines: Node 2
   └─▶ Sends GET request to Cache Node 2

4. Cache Node 2
   ├─ Hash table lookup: O(1)
   ├─ Key found: "user:12345"
   ├─ Return value
   └─▶ Response: {id: 12345, name: "John", ...}

5. Application Server 2
   ├─ Receives cached data
   ├─ Serializes to JSON
   └─▶ Returns to client

Total Time: ~2-3ms
├─ Network (app → cache): 1ms
├─ Cache lookup: 0.1ms
└─ Network (cache → app): 1ms
```

### 4.3 Request Flow - Cache Miss

```
Step-by-Step: User requests "user:99999" (not in cache)

1. Client Request
   └─▶ GET /api/users/99999

2. Application Server
   └─▶ Check Cache Node (via consistent hashing)

3. Cache Node
   ├─ Hash table lookup
   ├─ Key NOT found
   └─▶ Return NULL

4. Application Server (Cache Miss Handling)
   ├─ Detected cache miss
   ├─ Query database
   │  └─▶ SELECT * FROM users WHERE id = 99999
   ├─ Database returns data
   ├─ Write to cache
   │  └─▶ SET user:99999 = {data} TTL=3600
   └─▶ Return to client

Total Time: ~15-20ms
├─ Network (app → cache): 1ms
├─ Cache lookup: 0.1ms  
├─ Network (cache → app): 1ms
├─ Database query: 10ms
├─ Cache write: 1ms
└─ Network (app → client): 2ms
```

-----

## 5. Data Partitioning & Consistent Hashing

### 5.1 The Partitioning Problem

**Why We Need Partitioning:**

With distributed cache, data must be distributed across multiple nodes:

- 100 GB total data
- 4 nodes with 25 GB each
- **Question: Which node stores which key?**

### 5.2 Naive Approach: Modulo Hashing ❌

**Algorithm:**

```python
def get_node_id(key, num_nodes):
    hash_value = hash(key)  # e.g., hash("user:123") = 9876543210
    node_id = hash_value % num_nodes
    return node_id

Example with 4 nodes:
- hash("user:1") % 4 = 1 → Node 1
- hash("user:2") % 4 = 2 → Node 2  
- hash("user:3") % 4 = 3 → Node 3
- hash("user:4") % 4 = 0 → Node 0
```

**This works… until you add/remove nodes!**

**The Disaster Scenario:**

```
Initial State (4 nodes):
┌─────────────────┬──────────┐
│ Key             │ Node     │
├─────────────────┼──────────┤
│ user:1 (hash=5) │ 5 % 4 = 1│
│ user:2 (hash=6) │ 6 % 4 = 2│
│ user:3 (hash=7) │ 7 % 4 = 3│
│ user:4 (hash=8) │ 8 % 4 = 0│
│ user:5 (hash=9) │ 9 % 4 = 1│
└─────────────────┴──────────┘

Add Node 5 (now 5 nodes total):
┌─────────────────┬──────────┬────────────┐
│ Key             │ Old Node │ New Node   │
├─────────────────┼──────────┼────────────┤
│ user:1 (hash=5) │ 5 % 4 = 1│ 5 % 5 = 0  │ ✗ MOVED!
│ user:2 (hash=6) │ 6 % 4 = 2│ 6 % 5 = 1  │ ✗ MOVED!
│ user:3 (hash=7) │ 7 % 4 = 3│ 7 % 5 = 2  │ ✗ MOVED!
│ user:4 (hash=8) │ 8 % 4 = 0│ 8 % 5 = 3  │ ✗ MOVED!
│ user:5 (hash=9) │ 9 % 4 = 1│ 9 % 5 = 4  │ ✗ MOVED!
└─────────────────┴──────────┴────────────┘

Impact: 100% of keys remapped to different nodes!
Result: MASSIVE cache miss storm!
```

**Why This is Catastrophic:**

```
Before adding node: 90% cache hit rate
After adding node: 
├─ All keys in wrong nodes (100% miss rate)
├─ All requests hit database
├─ Database overloaded
├─ System crawls to a halt
└─ Cache needs hours to warm up again

For a system serving 200K QPS:
├─ 200K requests × 10ms DB latency = death
└─ Database cannot handle this load
```

### 5.3 Solution: Consistent Hashing ✓

**Core Concept:**

Imagine a hash space as a **ring** (circular number line):

```
Hash Space: 0 to 2^32 - 1 (4,294,967,295)

Visualized as a Ring:

                    0 / 2^32
                       ●
                  ╱         ╲
             ╱                 ╲
        ╱                         ╲
    ●                               ●
2^31-1                              1
    ╲                               ╱
        ╲                         ╱
             ╲                 ╱
                  ╲         ╱
                       ●
                     2^30
```

**Step 1: Place Nodes on the Ring**

Hash each node’s identifier and place it on the ring:

```python
# Hash each node ID to position on ring
hash("Node_A") = 100       → Position 100 on ring
hash("Node_B") = 1000      → Position 1000 on ring  
hash("Node_C") = 2500      → Position 2500 on ring
hash("Node_D") = 3000      → Position 3000 on ring
```

```
Visual Representation:

                  Node_B (1000)
                       ●
                  ╱         ╲
             ╱                 ╲
        ╱                         ╲
    ●                               ●
Node_A (100)                   Node_C (2500)
    ╲                               ╱
        ╲                         ╱
             ╲                 ╱
                  ╲         ╱
                       ●
                 Node_D (3000)
```

**Step 2: Place Keys on the Ring**

Hash each key and place it on the ring:

```python
hash("user:1") = 500      → Position 500
hash("user:2") = 1500     → Position 1500
hash("user:3") = 2800     → Position 2800
hash("user:4") = 3200     → Position 3200
hash("user:5") = 50       → Position 50
```

```
Ring with Nodes and Keys:

                  Node_B (1000)
                       ●
              user:2 (1500) ●
            ╱                   ╲
       ╱                           ╲
  ● user:1 (500)                     ● Node_C (2500)
●                                      ● user:3 (2800)
Node_A (100)                             
● user:5 (50)                         ╱
    ╲                               ╱
        ╲                         ╱
         ● user:4 (3200)       ╱
          ╲       ╱
               ●
         Node_D (3000)
```

**Step 3: Key Ownership Rule**

**Each key belongs to the FIRST NODE encountered when moving CLOCKWISE on the ring**

```
Key Assignment:
├─ user:5 (50)   → clockwise → Node_A (100)
├─ user:1 (500)  → clockwise → Node_B (1000)
├─ user:2 (1500) → clockwise → Node_C (2500)
├─ user:3 (2800) → clockwise → Node_D (3000)
└─ user:4 (3200) → clockwise → Node_A (100) [wraps around]

Distribution:
├─ Node_A: user:5, user:4
├─ Node_B: user:1
├─ Node_C: user:2
└─ Node_D: user:3
```

### 5.4 Adding a Node - The Magic ✨

**Scenario: Add Node_E at position 1500**

```
Before Adding Node_E:

                  Node_B (1000)
                       ●
              user:2 (1500) ●
            ╱                   ╲
       ╱                           ╲
  ● user:1 (500)                     ● Node_C (2500)

user:2 belongs to Node_C (first node clockwise from 1500)
```

```
After Adding Node_E at 1500:

                  Node_B (1000)
                       ●
                  Node_E (1500) ● [NEW!]
              user:2 (1500) ●
            ╱                   ╲
       ╱                           ╲
  ● user:1 (500)                     ● Node_C (2500)

Now user:2 belongs to Node_E (first node clockwise)
```

**Impact Analysis:**

```
┌──────────┬─────────────┬──────────────┬──────────┐
│ Key      │ Position    │ Before       │ After    │
├──────────┼─────────────┼──────────────┼──────────┤
│ user:5   │ 50          │ Node_A (100) │ Node_A   │ ✓ Same
│ user:1   │ 500         │ Node_B (1000)│ Node_B   │ ✓ Same
│ user:2   │ 1500        │ Node_C (2500)│ Node_E   │ ✗ Moved
│ user:3   │ 2800        │ Node_D (3000)│ Node_D   │ ✓ Same
│ user:4   │ 3200        │ Node_A (100) │ Node_A   │ ✓ Same
└──────────┴─────────────┴──────────────┴──────────┘

Only 1 out of 5 keys moved! (20%)

With N nodes, only ~1/N keys move when adding a node!
```

**Why This is Amazing:**

```
Modulo Hashing (N=4 → N=5):
└─ 100% keys remapped ✗

Consistent Hashing (N=4 → N=5):
└─ ~20% keys remapped ✓

For 1 million cached keys:
├─ Modulo: 1,000,000 cache misses
└─ Consistent: 200,000 cache misses (80% still cached!)
```

### 5.5 Removing a Node

**Scenario: Node_B (1000) fails and is removed**

```
Before (with Node_B):
user:1 (500) → Node_B (1000)

After (Node_B removed):
user:1 (500) → clockwise → Node_E (1500) [next node]
```

**Only keys that belonged to the removed node get reassigned!**

```
Impact:
├─ Keys on Node_A: Unchanged
├─ Keys on Node_B: Move to Node_E (next node clockwise)
├─ Keys on Node_C: Unchanged  
└─ Keys on Node_D: Unchanged

Only ~1/N keys affected!
```

### 5.6 Virtual Nodes - Solving Load Imbalance

**Problem with Basic Consistent Hashing:**

```
Uneven Distribution Example:

                    Node_B (500)
                         ●
                    ╱         ╲
               ╱                 ╲ 
          ╱                         ╲
     ●                                   ●
Node_A (100)                        Node_C (2900)
     ╲                                   ╱
          ╲                         ╱
               ╲                 ╱
                    ╲         ╱
                         ●
                    Node_D (3000)

Key Distribution:
├─ Node_A: keys in range [3000, 100] → 200 units
├─ Node_B: keys in range [100, 500] → 400 units  
├─ Node_C: keys in range [500, 2900] → 2400 units ← OVERLOADED!
└─ Node_D: keys in range [2900, 3000] → 100 units

Node_C handles 12x more keys than Node_D!
```

**Solution: Virtual Nodes**

Each physical node gets multiple positions on the ring:

```python
# Instead of one position per node:
Node_A → hash("Node_A") = 100

# Use multiple virtual nodes per physical node:
Node_A → hash("Node_A#1") = 100
Node_A → hash("Node_A#2") = 800
Node_A → hash("Node_A#3") = 1500
Node_A → hash("Node_A#4") = 2200
```

**With Virtual Nodes (150 vnodes per physical node):**

```
                    Much more even distribution!

              ● ● ● ● ●  ← Multiple Node_A vnodes
           ●                 ● ● ← Node_B vnodes
        ●                         ●
    ●     ● ● ← Node_C vnodes        ● ●
●                                         ●
    ●                                 ●
        ●   ● ● ← Node_D vnodes   ●
            ●                   ●
                ● ● ● ●

Each node has ~150 positions on ring
Much more uniform key distribution!
```

**Load Distribution Comparison:**

```
Without Virtual Nodes (4 physical nodes):
├─ Node_A: 15% of keys
├─ Node_B: 35% of keys
├─ Node_C: 45% of keys  ← OVERLOADED
└─ Node_D: 5% of keys   ← UNDERUTILIZED

With Virtual Nodes (150 vnodes per physical node):
├─ Node_A: 24.8% of keys
├─ Node_B: 25.3% of keys
├─ Node_C: 25.1% of keys
└─ Node_D: 24.8% of keys

Standard deviation: 5% → 0.2% (much better!)
```

### 5.7 Implementation Pseudocode

```python
class ConsistentHash:
    def __init__(self, num_virtual_nodes=150):
        self.num_virtual_nodes = num_virtual_nodes
        self.ring = {}  # position -> node_id
        self.sorted_positions = []  # sorted list of positions
        
    def add_node(self, node_id):
        """Add a physical node with virtual nodes"""
        for i in range(self.num_virtual_nodes):
            virtual_key = f"{node_id}#{i}"
            position = self._hash(virtual_key)
            self.ring[position] = node_id
        
        # Keep positions sorted for binary search
        self.sorted_positions = sorted(self.ring.keys())
    
    def remove_node(self, node_id):
        """Remove a node and its virtual nodes"""
        for i in range(self.num_virtual_nodes):
            virtual_key = f"{node_id}#{i}"
            position = self._hash(virtual_key)
            del self.ring[position]
        
        self.sorted_positions = sorted(self.ring.keys())
    
    def get_node(self, key):
        """Find which node should store this key"""
        if not self.ring:
            return None
        
        # Hash the key to get its position
        position = self._hash(key)
        
        # Find first node clockwise (binary search)
        # If no node found clockwise, wrap to first node
        idx = bisect.bisect_right(self.sorted_positions, position)
        if idx == len(self.sorted_positions):
            idx = 0
        
        node_position = self.sorted_positions[idx]
        return self.ring[node_position]
    
    def _hash(self, key):
        """Hash function (e.g., MD5, SHA-1)"""
        return int(hashlib.md5(key.encode()).hexdigest(), 16) % (2**32)

# Usage Example
ch = ConsistentHash(num_virtual_nodes=150)

# Add nodes
ch.add_node("cache_node_1")
ch.add_node("cache_node_2")
ch.add_node("cache_node_3")

# Find which node stores a key
node = ch.get_node("user:12345")  # Returns "cache_node_2"

# Add a new node - only ~1/4 keys will move
ch.add_node("cache_node_4")
```

-----

## 6. Eviction Policies

### 6.1 The Eviction Problem

**Scenario:**

```
Cache is full (10,000 entries, max capacity reached)
New request to cache: user:99999

Question: Which existing entry do we remove?
```

### 6.2 LRU (Least Recently Used) - Most Popular ⭐

**Concept:** Remove the entry that hasn’t been accessed in the longest time.

**Intuition:** If data hasn’t been used recently, it’s less likely to be used soon.

**Data Structure:**

```
HashMap + Doubly Linked List

HashMap: For O(1) lookup
├─ Key → Node pointer in linked list

Doubly Linked List: For O(1) insertion/deletion
├─ Head: Most recently accessed
└─ Tail: Least recently accessed

Combined Structure:

┌─────────────────────────────────────────────┐
│              HashMap                         │
│  ┌──────────────────────────────────────┐  │
│  │ user:1 → ●                           │  │
│  │ user:2 → ●                           │  │
│  │ user:3 → ●                           │  │
│  └─────│────────│────────│──────────────┘  │
└────────│────────│────────│───────────────────┘
         │        │        │
         ▼        ▼        ▼
    ┌─────────┐ ┌─────────┐ ┌─────────┐
HEAD│ user:3  │◄│ user:1  │◄│ user:2  │TAIL
    │(recent) │►│         │►│  (LRU)  │
    └─────────┘ ┌─────────┘ └─────────┘
         ↑
    Most recently
     accessed
```

**Operations:**

```
GET(key):
1. Lookup in HashMap: O(1)
2. If found:
   - Move node to HEAD of list: O(1)
   - Return value
3. If not found:
   - Return null

PUT(key, value):
1. If key exists:
   - Update value
   - Move to HEAD
2. If key doesn't exist:
   - If cache full:
     - Remove TAIL node (LRU)
     - Remove from HashMap
   - Create new node
   - Add to HEAD
   - Add to HashMap

Both operations: O(1) time complexity!
```

**Visual Example:**

```
Initial State (capacity = 3):
HEAD → [user:3] ↔ [user:1] ↔ [user:2] ← TAIL

Access user:1:
├─ Move user:1 to HEAD
HEAD → [user:1] ↔ [user:3] ↔ [user:2] ← TAIL

Insert user:4 (cache full, evict LRU):
├─ Remove user:2 (tail)
├─ Insert user:4 at head
HEAD → [user:4] ↔ [user:1] ↔ [user:3] ← TAIL
```

**Implementation:**

```python
class Node:
    def __init__(self, key, value):
        self.key = key
        self.value = value
        self.prev = None
        self.next = None

class LRUCache:
    def __init__(self, capacity):
        self.capacity = capacity
        self.cache = {}  # key -> Node
        
        # Dummy head and tail for easier manipulation
        self.head = Node(0, 0)
        self.tail = Node(0, 0)
        self.head.next = self.tail
        self.tail.prev = self.head
    
    def get(self, key):
        if key in self.cache:
            node = self.cache[key]
            self._remove(node)
            self._add_to_head(node)
            return node.value
        return -1
    
    def put(self, key, value):
        if key in self.cache:
            self._remove(self.cache[key])
        
        node = Node(key, value)
        self._add_to_head(node)
        self.cache[key] = node
        
        if len(self.cache) > self.capacity:
            # Remove LRU (tail)
            lru = self.tail.prev
            self._remove(lru)
            del self.cache[lru.key]
    
    def _remove(self, node):
        """Remove node from linked list"""
        node.prev.next = node.next
        node.next.prev = node.prev
    
    def _add_to_head(self, node):
        """Add node right after head"""
        node.next = self.head.next
        node.prev = self.head
        self.head.next.prev = node
        self.head.next = node
```

**Pros:**

- O(1) operations
- Good for temporal locality (recently accessed items)
- Simple to understand and implement

**Cons:**

- Can be fooled by one-time scans
- Doesn’t consider access frequency

### 6.3 LFU (Least Frequently Used)

**Concept:** Remove the entry accessed the fewest number of times.

**Data Structure:**

```
HashMap + Frequency List

frequency_map: frequency → list of keys with that frequency
key_frequency: key → current frequency
key_value: key → cached value

Example:
┌──────────────────────────────────────────┐
│ key_frequency                             │
│  user:1 → 5  (accessed 5 times)          │
│  user:2 → 2  (accessed 2 times)          │
│  user:3 → 8  (accessed 8 times)          │
└──────────────────────────────────────────┘

┌──────────────────────────────────────────┐
│ frequency_map                             │
│  2 → [user:2]                            │
│  5 → [user:1]                            │
│  8 → [user:3]                            │
└──────────────────────────────────────────┘

When evicting: Remove from lowest frequency
```

**Pros:**

- Keeps “hot” data (frequently accessed)
- Better for frequency-based patterns

**Cons:**

- Complex implementation
- Doesn’t adapt to changing patterns (old popular items stick around)
- New items easily evicted

### 6.4 Other Eviction Policies

**FIFO (First In First Out):**

```
Remove oldest inserted item (like a queue)
Simple but doesn't consider usage patterns
```

**Random:**

```
Evict random entry
Surprisingly effective in some cases!
Low overhead
```

**TTL-Based:**

```
Remove expired entries first
Then fall back to LRU/LFU
Used in combination with other policies
```

### 6.5 Comparison Table

|Policy    |Time Complexity       |Best For           |Weakness               |
|----------|----------------------|-------------------|-----------------------|
|**LRU**   |O(1) get/put          |Temporal locality  |Scan pollution         |
|**LFU**   |O(1) get, O(log n) put|Frequency matters  |Old items never evicted|
|**FIFO**  |O(1)                  |Simple workloads   |Ignores usage          |
|**Random**|O(1)                  |Low overhead needed|Unpredictable          |

-----

## 7. Cache Strategies & Patterns

### 7.1 Cache-Aside (Lazy Loading) ⭐ Most Common

**Pattern:**

```
READ Flow:
┌──────────────────────────────────────────┐
│ 1. Application checks cache              │
└────────────────┬─────────────────────────┘
                 │
        ┌────────┴─────────┐
        │                  │
    Cache HIT          Cache MISS
        │                  │
        ▼                  ▼
   Return data      ┌─────────────────┐
                    │ 2. Read from DB │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │ 3. Write cache  │
                    └────────┬────────┘
                             │
                             ▼
                       Return data

WRITE Flow:
┌──────────────────────────────────────────┐
│ 1. Write to database                     │
└────────────────┬─────────────────────────┘
                 │
                 ▼
┌──────────────────────────────────────────┐
│ 2. Invalidate (delete) cache entry       │
└──────────────────────────────────────────┘
```

**Code Example:**

```python
def get_user(user_id):
    cache_key = f"user:{user_id}"
    
    # Try cache first
    user = cache.get(cache_key)
    
    if user is None:  # Cache miss
        # Fetch from database
        user = database.query(
            f"SELECT * FROM users WHERE id = {user_id}"
        )
        
        if user:
            # Store in cache for 1 hour
            cache.set(cache_key, user, ttl=3600)
    
    return user

def update_user(user_id, data):
    # Update database first
    database.update(
        f"UPDATE users SET name='{data.name}' WHERE id={user_id}"
    )
    
    # Invalidate cache (delete, don't update!)
    cache_key = f"user:{user_id}"
    cache.delete(cache_key)
    
    # Next read will fetch fresh data from DB
```

**Why Delete Instead of Update? (Critical!)**

```
WRONG Approach (Update Cache):
──────────────────────────────
Thread A:                    Thread B:
1. Update DB                 
   user = "Alice"            
                             2. Read DB (gets old "Bob")
3. Update cache              
   cache = "Alice"           
                             4. Update cache
                                cache = "Bob" ← STALE!

Result: Cache has wrong data!

CORRECT Approach (Delete Cache):
─────────────────────────────────
Thread A:                    Thread B:
1. Update DB
   user = "Alice"
2. DELETE cache
                             3. Read cache (MISS)
                             4. Read DB (gets "Alice")
                             5. Write cache = "Alice" ✓

Result: Next read gets fresh data from DB!
```

**Pros:**

- Only caches requested data
- Resilient (cache failure → slower, but works)
- Simple to implement

**Cons:**

- Cache miss penalty (3 round trips)
- Thundering herd on cache expiration

### 7.2 Write-Through

**Pattern:**

```
WRITE Flow:
┌──────────────────────────────────────────┐
│ 1. Application writes to cache           │
└────────────────┬─────────────────────────┘
                 │
                 ▼
┌──────────────────────────────────────────┐
│ 2. Cache synchronously writes to DB      │
└────────────────┬─────────────────────────┘
                 │
                 ▼
┌──────────────────────────────────────────┐
│ 3. Return success after DB write         │
└──────────────────────────────────────────┘

All writes go through cache
Cache and DB always in sync
```

**Code Example:**

```python
def update_user(user_id, data):
    cache_key = f"user:{user_id}"
    
    # Write to cache
    cache.set(cache_key, data, ttl=3600)
    
    # Cache writes to database (synchronous)
    database.update(
        f"UPDATE users SET name='{data.name}' WHERE id={user_id}"
    )
    
    return True  # Both cache and DB updated
```

**Pros:**

- Cache always consistent with DB
- No stale data

**Cons:**

- Higher write latency (cache + DB)
- Writes data that might never be read
- More complex failure scenarios

### 7.3 Write-Behind (Write-Back)

**Pattern:**

```
WRITE Flow:
┌──────────────────────────────────────────┐
│ 1. Application writes to cache           │
└────────────────┬─────────────────────────┘
                 │
                 ▼
┌──────────────────────────────────────────┐
│ 2. Cache returns success immediately     │
└────────────────┬─────────────────────────┘
                 │
                 ▼
┌──────────────────────────────────────────┐
│ 3. Cache asynchronously writes to DB     │
│    (batched, delayed)                    │
└──────────────────────────────────────────┘
```

**Async Write Process:**

```
Cache maintains write queue:

┌─────────────────────────────────────┐
│ Write Queue (in cache memory)      │
│  ┌─────────────────────────────┐   │
│  │ user:1 → UPDATE at 10:00:01 │   │
│  │ user:2 → UPDATE at 10:00:02 │   │
│  │ user:3 → UPDATE at 10:00:03 │   │
│  │ ...                         │   │
│  └─────────────────────────────┘   │
└─────────────────┬───────────────────┘
                  │
         Every 5 seconds or 1000 writes
                  │
                  ▼
        ┌─────────────────┐
        │ Batch write to  │
        │    Database     │
        └─────────────────┘
```

**Pros:**

- Fastest writes (return immediately)
- Can batch DB writes (better throughput)
- Reduces DB load

**Cons:**

- Risk of data loss if cache crashes
- Complex to implement correctly
- Eventual consistency

### 7.4 Comparison

|Pattern          |Read Latency         |Write Latency|Consistency|Data Loss Risk|Use Case                     |
|-----------------|---------------------|-------------|-----------|--------------|-----------------------------|
|**Cache-Aside**  |Medium (miss penalty)|Fast         |Eventual   |None          |General purpose (most common)|
|**Write-Through**|Fast                 |Slow         |Strong     |None          |Read-heavy, need consistency |
|**Write-Behind** |Fast                 |Fastest      |Weak       |High          |Write-heavy, async OK        |

-----

## 8. High Availability & Replication

### 8.1 Replication Strategies

**Master-Slave (Master-Replica) Replication:**

```
Architecture:

        ┌─────────────┐
        │   Master    │ ← All writes go here
        │  (Node 1)   │
        └──────┬──────┘
               │
    ┌──────────┼──────────┐
    │          │          │
    │    Async Replication│
    ▼          ▼          ▼
┌─────────┐┌─────────┐┌─────────┐
│Replica 1││Replica 2││Replica 3│ ← Reads can go here
└─────────┘└─────────┘└─────────┘

Write Flow:
1. Client → Master
2. Master updates data
3. Master returns success
4. Master replicates to replicas (async)

Read Flow (can be distributed):
├─ Option 1: Read from master (consistent but higher load)
└─ Option 2: Read from replicas (lower load but possible lag)
```

**Replication Lag:**

```
Time: 10:00:00 - Write to master
Time: 10:00:00.100 - Replicated to Replica 1
Time: 10:00:00.150 - Replicated to Replica 2
Time: 10:00:00.200 - Replicated to Replica 3

User writes at 10:00:00, reads from Replica 2 at 10:00:00.120
└─ Might see old data! (eventual consistency)
```

**Pros:**

- Scales read capacity
- Failover available
- Simple to understand

**Cons:**

- Replication lag (eventual consistency)
- Master is single point of failure for writes
- Replica promotion complexity

### 8.2 Failure Scenarios

**Master Failure:**

```
Before Failure:
┌─────────┐
│ Master  │ ✓
└────┬────┘
     │
  ┌──┴──┐
  ▼     ▼
┌───┐ ┌───┐
│R1 │ │R2 │
└───┘ └───┘

Master Crashes:
┌─────────┐
│ Master  │ ✗ DEAD
└─────────┘

  ┌───┐ ┌───┐
  │R1 │ │R2 │
  └───┘ └───┘

Failover Process:
1. Detection (heartbeat timeout ~3-5 sec)
2. Election/Selection of new master
3. Promote R1 to master
4. Update client routing
5. Resume operations

After Failover:
┌─────────┐
│R1 (now  │ ✓ NEW MASTER
│ Master) │
└────┬────┘
     │
     ▼
   ┌───┐
   │R2 │
   └───┘
```

**Replica Failure:**

```
Before:
┌─────────┐
│ Master  │
└────┬────┘
     │
  ┌──┴──┬──┐
  ▼     ▼  ▼
┌───┐ ┌───┐┌───┐
│R1 │ │R2 ││R3 │
└───┘ └───┘└───┘

R2 Fails:
┌─────────┐
│ Master  │
└────┬────┘
     │
  ┌──┴──┬──┐
  ▼     ✗  ▼
┌───┐ ┌───┐┌───┐
│R1 │ │R2 ││R3 │
└───┘ └───┘└───┘

Impact:
├─ Reads from R2 fail → redirect to R1 or R3
├─ Master continues writing
├─ Lower read capacity temporarily
└─ Auto-recover when R2 comes back
```

### 8.3 Redis Sentinel (High Availability)

**Redis Sentinel Architecture:**

```
┌─────────────────────────────────────────────────┐
│            Sentinel Cluster                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐     │
│  │Sentinel 1│  │Sentinel 2│  │Sentinel 3│     │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘     │
└───────┼─────────────┼─────────────┼────────────┘
        │             │             │
    Monitor      Monitor        Monitor
        │             │             │
        ▼             ▼             ▼
    ┌───────────────────────────────────┐
    │   Redis Cluster                   │
    │  ┌────────┐  ┌────────┐  ┌────┐  │
    │  │Master  │  │Replica1│  │Rep2│  │
    │  └────────┘  └────────┘  └────┘  │
    └───────────────────────────────────┘

Sentinel Responsibilities:
├─ Monitor: Check master/replica health
├─ Notify: Alert on failures
├─ Failover: Automatic promotion
└─ Configuration: Update clients on topology changes
```

**Automatic Failover Process:**

```
1. Detection (Sentinels vote):
   ├─ Sentinel 1: Master down? YES
   ├─ Sentinel 2: Master down? YES
   └─ Sentinel 3: Master down? YES
   └─▶ Quorum reached (2/3) → Master is down

2. Replica Selection:
   ├─ Choose replica with least lag
   ├─ Choose replica with highest priority
   └─▶ Replica 1 selected

3. Promotion:
   ├─ Sentinel issues SLAVEOF NO ONE to Replica 1
   └─▶ Replica 1 becomes new master

4. Reconfigure:
   ├─ Point Replica 2 to new master
   └─ Update client configuration

5. Client Update:
   └─▶ Clients query Sentinels for new master address

Total failover time: ~10-30 seconds
```

-----

## 9. Advanced Problems & Solutions

### 9.1 Hot Key Problem

**Problem:**

```
Celebrity posts tweet → 10 million followers request
Same key: "tweet:12345"
All requests hit ONE cache node

┌──────────┐
│ Node 1   │ ← 1M QPS (OVERLOADED!)
└──────────┘

┌──────────┐
│ Node 2   │ ← 100K QPS (normal)
└──────────┘

┌──────────┐
│ Node 3   │ ← 100K QPS (normal)
└──────────┘

Result: Node 1 crashes, system degraded
```

**Solution 1: Local Caching (L1 Cache)**

```
Application Server:
┌─────────────────────────────────┐
│  Local In-Memory Cache          │
│  (Small, 100MB)                 │
│  ┌──────────────────────────┐   │
│  │ tweet:12345 (cached)     │   │
│  │ TTL: 10 seconds          │   │
│  └──────────────────────────┘   │
└───────────────┬─────────────────┘
                │
      Only cache miss hits
         distributed cache
                │
                ▼
        ┌──────────────┐
        │ Redis Cache  │
        └──────────────┘

With 100 app servers:
├─ 1M requests total
├─ 10K requests per app server
├─ With 10-sec local cache: 1 request per 10 sec to Redis
└─ Redis sees: 100 req/sec instead of 1M req/sec!
```

**Solution 2: Replicate Hot Keys**

```
Detect hot key → Replicate to multiple nodes

Normal:
tweet:12345 → Node 1 only

Hot key detected:
tweet:12345#1 → Node 1
tweet:12345#2 → Node 2
tweet:12345#3 → Node 3

Load balancer distributes reads:
├─ 333K QPS → Node 1
├─ 333K QPS → Node 2
└─ 333K QPS → Node 3
```

**Solution 3: Request Coalescing**

```
Within application server:

100 simultaneous requests for "tweet:12345"
└─▶ Only 1 request goes to cache
└─▶ Other 99 wait
└─▶ Share result with all 100

Pseudocode:
pending_requests = {}

def get_with_coalescing(key):
    if key in pending_requests:
        # Wait for existing request
        return pending_requests[key].wait()
    
    # First request for this key
    future = create_future()
    pending_requests[key] = future
    
    # Fetch from cache
    value = cache.get(key)
    
    # Notify all waiters
    future.set_result(value)
    del pending_requests[key]
    
    return value
```

### 9.2 Thundering Herd (Cache Stampede)

**Problem:**

```
Popular cache entry expires

Time: 10:00:00
└─ Cache entry "timeline:popular_user" expires

Time: 10:00:00.001
├─ Request 1 checks cache → MISS
├─ Request 2 checks cache → MISS
├─ Request 3 checks cache → MISS
├─ ... (1000 concurrent requests)
└─ Request 1000 checks cache → MISS

All 1000 requests hit database simultaneously!
Database: 💥 OVERLOADED
```

**Visual:**

```
Cache Entry Lifetime:
│
│ ████████████ Cached (90% hit rate)
│              ↓ Expires
│              ▼
│              ● ← Thundering herd (1000 DB queries)
│              ↓
│ ████████████ Cached again
│
└────────────────────────────────────▶ Time
     1 min      Expire    1 min
```

**Solution 1: Probabilistic Early Expiration**

```python
import random
import time

def get_with_early_expiration(key, ttl=3600, beta=1.0):
    value, expiry_time = cache.get_with_metadata(key)
    
    if value is None:
        # Cache miss - fetch and cache
        value = fetch_from_db(key)
        cache.set(key, value, ttl=ttl)
        return value
    
    # Calculate time until expiration
    time_to_expiry = expiry_time - time.time()
    
    # Probabilistically refresh before expiration
    # Keys accessed more frequently refresh earlier
    if random.random() < beta * (time_to_expiry / ttl):
        # Refresh cache asynchronously
        async_refresh(key, ttl)
    
    return value

# With this approach:
# High-traffic keys refresh ~30 sec before expiration
# Low-traffic keys expire normally
# Spreads out the refresh load
```

**Solution 2: Locking (Cache Stampede Prevention)**

```python
def get_with_lock(key, ttl=3600):
    value = cache.get(key)
    
    if value is not None:
        return value
    
    # Try to acquire lock
    lock_key = f"lock:{key}"
    lock_acquired = cache.set(
        lock_key, 
        "1", 
        ttl=10,  # 10-second lock
        nx=True  # Only set if not exists
    )
    
    if lock_acquired:
        # This request gets to refresh cache
        try:
            value = fetch_from_db(key)
            cache.set(key, value, ttl=ttl)
            return value
        finally:
            cache.delete(lock_key)
    else:
        # Another request is refreshing
        # Wait a bit and retry
        time.sleep(0.1)
        return get_with_lock(key, ttl)  # Retry
```

**Solution 3: Stale-While-Revalidate**

```python
def get_with_stale_cache(key, ttl=3600, grace_ttl=60):
    """
    Return stale data while refreshing in background
    """
    value, expiry_time = cache.get_with_metadata(key)
    
    if value is None:
        # True cache miss
        value = fetch_from_db(key)
        cache.set(key, value, ttl=ttl)
        return value
    
    now = time.time()
    
    if now < expiry_time:
        # Still fresh
        return value
    elif now < (expiry_time + grace_ttl):
        # Stale but within grace period
        # Return stale value immediately
        # Refresh asynchronously
        async_refresh(key, ttl)
        return value  # Stale but acceptable
    else:
        # Too stale - must refresh synchronously
        value = fetch_from_db(key)
        cache.set(key, value, ttl=ttl)
        return value
```

### 9.3 Celebrity Timeline Problem (DETAILED)

**The Challenge:**

```
Regular User (500 followers):
└─ Posts tweet → Update 500 follower timelines
   └─ Time: 1 second ✓
   └─ Cost: 500 cache writes ✓

Celebrity (600M followers):
└─ Posts tweet → Update 600M follower timelines?
   └─ Time: 7 days ✗
   └─ Cost: 600M cache writes ✗
   └─ Waste: 590M followers offline ✗
```

**Solution: Hybrid Fan-out Strategy**

```
User Classification:
├─ Small (< 10K followers) → Fan-out on Write (Push)
├─ Medium (10K - 1M) → Partial fan-out
└─ Large (> 1M followers) → Fan-out on Read (Pull)
```

**Implementation:**

```
┌─────────────────────────────────────────────┐
│ Tweet Posted                                │
└────────────────┬────────────────────────────┘
                 │
                 ▼
        ┌────────────────┐
        │ Check follower │
        │     count      │
        └────────┬───────┘
                 │
     ┌───────────┴──────────┐
     │                      │
< 10K followers      > 1M followers
     │                      │
     ▼                      ▼
┌─────────────┐      ┌─────────────┐
│ Fan-out     │      │ Just store  │
│ immediately │      │ in DB       │
│             │      │ (no fanout) │
│ For each    │      └─────────────┘
│ follower:   │
│ - Add tweet │
│   to their  │
│   timeline  │
│   cache     │
└─────────────┘
```

**Timeline Read for Regular User:**

```
User follows: 100 regular friends + Cristiano (celebrity)

1. Request timeline
   └─▶ GET /timeline

2. Check timeline cache
   ├─ Cache key: "timeline:user123"
   ├─ Contains: Tweets from 100 regular friends ✓
   └─ Missing: Cristiano's latest tweets ✗

3. Backend detects celebrity followees
   └─▶ Query: Which celebrities does user123 follow?
   └─▶ Result: [Cristiano, Messi, ...]

4. Fetch celebrity tweets
   ├─ Query DB/celebrity cache:
   │  SELECT * FROM tweets 
   │  WHERE user_id IN (Cristiano, Messi)
   │  AND timestamp > (last 24 hours)
   └─▶ Result: Latest celebrity tweets

5. Merge timelines
   ├─ Regular friends' tweets (from cache)
   ├─ + Celebrity tweets (from DB)
   └─▶ Sort by timestamp

6. Cache merged result (SHORT TTL = 60 seconds)
   └─▶ SET timeline:user123:merged = {merged_data} EX 60

7. Return to user
   └─▶ User sees complete timeline
```

**Why Short TTL (60 seconds)?**

```
Without caching merged timeline:
├─ Every refresh queries DB for celebrity tweets
├─ User refreshes 10x per hour
├─ 10M users × 10 refreshes = 100M DB queries/hour ✗

With 60-second cache:
├─ First request: Query DB
├─ Next 60 seconds: Cache hit
├─ User's 10 refreshes → Only 1 DB query
├─ 10M users × 1 query = 10M DB queries/hour ✓
└─ 10x reduction in DB load!

Staleness trade-off:
├─ Celebrity tweets up to 60 seconds old
└─ Acceptable for social media!
```

**Complete Flow Diagram:**

```
Cristiano Posts Tweet (2:00 PM)
        │
        ▼
┌──────────────────┐
│ Save to Database │
└────────┬─────────┘
         │
         │ (NO fan-out to 600M followers)
         ▼
    ┌────────┐
    │  Done  │ (Milliseconds!)
    └────────┘


User Opens Timeline (2:05 PM)
        │
        ▼
┌──────────────────────┐
│ Check timeline cache │
└────────┬─────────────┘
         │
    ┌────┴─────┐
    │          │
 Cache HIT  Cache MISS
(< 60 sec)  (expired)
    │          │
    │          ▼
    │    ┌──────────────────────┐
    │    │ Fetch regular        │
    │    │ friends' tweets      │
    │    │ (cached, fanned-out) │
    │    └──────────┬───────────┘
    │               │
    │               ▼
    │    ┌──────────────────────┐
    │    │ Identify celebrities │
    │    │ user follows         │
    │    └──────────┬───────────┘
    │               │
    │               ▼
    │    ┌──────────────────────┐
    │    │ Fetch celebrity      │
    │    │ tweets from DB       │
    │    │ (Cristiano's 2:00    │
    │    │  tweet included!)    │
    │    └──────────┬───────────┘
    │               │
    │               ▼
    │    ┌──────────────────────┐
    │    │ Merge both sources   │
    │    └──────────┬───────────┘
    │               │
    │               ▼
    │    ┌──────────────────────┐
    │    │ Cache merged result  │
    │    │ TTL = 60 seconds     │
    │    └──────────┬───────────┘
    │               │
    └───────────────┴──────────┐
                                │
                                ▼
                    ┌──────────────────────┐
                    │ Return complete      │
                    │ timeline to user     │
                    └──────────────────────┘
```

**Key Insight:**

```
The cost of serving is proportional to ACTIVE users, not TOTAL followers!

If Cristiano has 600M followers but only 10M active:
├─ Fan-out approach: 600M writes (wasteful!)
└─ Pull approach: 10M reads (efficient!)

Most followers are offline → No need to pre-compute their timelines!
```

-----

## 10. Real-World Case Studies

### 10.1 Facebook’s Memcache

**Scale:**

- Trillions of items
- Billions of requests per second
- Petabytes of data
- Hundreds of data centers

**Architecture:**

```
┌─────────────────────────────────────────────┐
│          Regional Clusters                  │
│                                             │
│  ┌─────────────────────────────────────┐   │
│  │  Frontend Cluster                   │   │
│  │  (Close to web servers)             │   │
│  │  - Ultra-low latency                │   │
│  │  - Serves most reads                │   │
│  └───────────────┬─────────────────────┘   │
│                  │                          │
│  ┌───────────────▼─────────────────────┐   │
│  │  Storage Cluster                    │   │
│  │  (Persistent backing store)         │   │
│  │  - Larger capacity                  │   │
│  │  - Connected to databases           │   │
│  └─────────────────────────────────────┘   │
└─────────────────────────────────────────────┘

Cross-region replication for global consistency
```

**Optimizations:**

```
1. Protocol Optimizations:
   ├─ UDP for get() → Lower latency, acceptable packet loss
   └─ TCP for set() → Reliability for writes

2. Lease Tokens (Prevent stale sets):
   ├─ Cache miss → returns lease token
   ├─ Client fetches from DB
   ├─ Client sets cache with lease token
   └─ Token invalidated if key updated elsewhere
       (prevents race condition)

3. Gutter Pools:
   ├─ Temporary backup cache when main cache fails
   ├─ Prevents thundering herd to database
   └─ Short TTL (expunges quickly when main cache recovers)

4. Regional Invalidation:
   ├─ Master region updates DB
   ├─ Broadcasts invalidation to all regions
   └─ Eventual consistency across globe
```

### 10.2 Twitter’s Caching Strategy

**Timeline Generation Challenge:**

```
User timeline requires:
├─ Tweets from 1000s of followees
├─ Merge and sort by time
├─ Apply ranking algorithm
└─ Filter blocked/muted users

Too expensive to compute on every request!
```

**Solution: Hybrid Approach**

```
┌────────────────────────────────────┐
│ User Classification                │
├────────────────────────────────────┤
│ < 10K followers: Fan-out on Write  │
│ > 10K followers: Fan-out on Read   │
└────────────────────────────────────┘

Regular User Posts:
└─▶ Push to all followers' timelines immediately

Celebrity Posts:
└─▶ Store tweet only
└─▶ Pull when followers request timeline

Result: Balanced load, acceptable latency
```

**Cache Architecture:**

```
┌─────────────────────────────────────┐
│ L1: In-memory cache in web servers │
│ (Small, fast, application-level)   │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│ L2: Redis/Memcached cluster         │
│ (Distributed, shared across servers)│
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│ L3: Database with query cache       │
└─────────────────────────────────────┘

Multi-tier caching maximizes hit rate!
```

### 10.3 Netflix’s EVCache

**Requirements:**

- 99.9% availability
- Multi-region replication
- Billions of requests per day
- Handle cloud failures gracefully

**Architecture:**

```
┌─────────────────────────────────────┐
│ Replication Group (3 AWS zones)     │
│                                     │
│  Zone A      Zone B      Zone C    │
│  ┌────┐     ┌────┐     ┌────┐     │
│  │Node│     │Node│     │Node│     │
│  └────┘     └────┘     └────┘     │
│                                     │
│  All nodes have same data           │
│  Client reads from closest zone     │
│  Client writes to all zones         │
└─────────────────────────────────────┘

Failure Handling:
├─ Zone failure → Read from other zones
├─ Node failure → Read from replicas
└─ Network partition → Serve stale data (configurable)
```

**Key Features:**

```
1. Tunable Consistency:
   └─ Can choose: Read from 1, 2, or all 3 zones
   └─ Trade-off: Consistency vs. Latency

2. Automatic Rebalancing:
   └─ Detects hot nodes
   └─ Migrates keys to balance load

3. Chunking:
   └─ Large values split across multiple keys
   └─ Enables parallel fetch

4. Fast Warm-up:
   └─ Replays recent write log on restart
   └─ Cache operational in minutes, not hours
```

-----

## 11. Interview Framework

### 11.1 How to Approach “Design a Distributed Cache”

**Step 1: Requirements Clarification (3-5 minutes)**

```
Questions to Ask:

Functional:
├─ What operations? (get, set, delete, or advanced?)
├─ What data types? (strings, objects, complex structures?)
├─ Need persistence or pure in-memory?
└─ Any special features? (TTL, atomic operations, pub/sub?)

Non-Functional:
├─ Scale: How much data? QPS?
├─ Latency: What's acceptable?
├─ Availability: Can we tolerate downtime?
├─ Consistency: Strong or eventual?
└─ Geographic distribution needed?

Example Answer:
"Let me clarify the requirements:
- We need get/set/delete operations
- Handling 1M QPS
- Sub-millisecond latency (p99)
- 100GB total data
- Eventual consistency is acceptable
- High availability (99.9%+)
Is that correct?"
```

**Step 2: Capacity Estimation (2-3 minutes)**

```
Calculate:
├─ QPS (requests per second)
├─ Storage (total cache size)
├─ Bandwidth (network throughput)
└─ Number of nodes needed

Example:
"With 1M QPS and each node handling 100K QPS:
- Need at least 10 nodes for throughput
- With 100GB data and 64GB per node:
- Need 2 nodes for storage
- With 3x replication: 6 nodes minimum
- Adding buffer: recommend 12 nodes"
```

**Step 3: High-Level Design (5-7 minutes)**

```
Draw architecture:

1. Client Layer
   └─ Cache client library (consistent hashing)

2. Cache Cluster
   ├─ Multiple nodes
   ├─ Data partitioning
   └─ Replication

3. Persistence Layer
   └─ Database (source of truth)

Explain request flow:
├─ Cache hit path
└─ Cache miss path
```

**Step 4: Deep Dive (15-20 minutes)**

Pick 2-3 areas based on interviewer interest:

```
Common Deep Dive Topics:

A) Data Partitioning:
   ├─ Why consistent hashing?
   ├─ Virtual nodes
   └─ Handling node addition/removal

B) Eviction Policy:
   ├─ LRU implementation
   ├─ Data structures (HashMap + LinkedList)
   └─ O(1) operations

C) High Availability:
   ├─ Replication strategy
   ├─ Failure detection
   └─ Failover mechanism

D) Cache Strategy:
   ├─ Cache-aside pattern
   ├─ Write-through vs write-behind
   └─ Invalidation strategy

E) Advanced Problems:
   ├─ Hot keys
   ├─ Thundering herd
   └─ Cache stampede
```

**Step 5: Bottlenecks & Trade-offs (5 minutes)**

```
Discuss:

Bottlenecks:
├─ Network bandwidth
├─ Single hot key
├─ Thundering herd
└─ Replication lag

Solutions:
├─ Local caching (L1 cache)
├─ Request coalescing
├─ Stale-while-revalidate
└─ Asynchronous replication

Trade-offs:
├─ Consistency vs Availability
├─ Memory vs Performance
├─ Complexity vs Simplicity
└─ Cost vs Latency
```

### 11.2 Common Follow-up Questions

**Q1: “How would you handle cache invalidation across multiple data centers?”**

```
Answer Framework:

1. Identify the challenge:
   "Cache invalidation across DCs is hard because:
   - Network latency between DCs (50-200ms)
   - Potential network partitions
   - Need eventual consistency"

2. Propose solution:
   "I'd use a publish-subscribe invalidation system:
   
   ┌──────────┐         ┌──────────┐
   │  DC-US   │         │  DC-EU   │
   │ ┌──────┐ │         │ ┌──────┐ │
   │ │Cache │ │         │ │Cache │ │
   │ └───┬──┘ │         │ └───┬──┘ │
   │     │    │         │     │    │
   └─────┼────┘         └─────┼────┘
         │                    │
         └────────┬───────────┘
                  │
           ┌──────▼──────┐
           │ Message Bus │
           │  (Kafka)    │
           └─────────────┘
   
   - Master DC writes update + publishes invalidation
   - Other DCs subscribe and invalidate locally
   - Accept temporary inconsistency (100-500ms)
   - Use versioning to handle out-of-order messages"

3. Discuss trade-offs:
   "Trade-off: Consistency vs availability
   - Could use 2PC for strong consistency, but high latency
   - Choosing eventual consistency for better availability"
```

**Q2: “Your cache hit rate is only 60%. How do you improve it?”**

```
Answer Framework:

1. Diagnose the problem:
   "Let me investigate why hit rate is low:
   
   Potential causes:
   ├─ Cache size too small → frequent evictions
   ├─ TTL too short → entries expire too quickly
   ├─ Working set > cache capacity
   ├─ Access pattern doesn't fit eviction policy
   └─ One-time scans polluting cache"

2. Propose solutions:

   "Short-term fixes:
   ├─ Increase cache size (if budget allows)
   ├─ Tune TTL values based on access patterns
   ├─ Implement admission policy (don't cache one-time access)
   └─ Use LRU-K instead of LRU (resist scans)
   
   Long-term optimizations:
   ├─ Analyze access patterns (hot vs cold data)
   ├─ Tiered caching (hot data in L1, warm in L2)
   ├─ Predictive prefetching
   └─ Shard cache by access pattern"

3. Metrics to track:
   "Monitor:
   ├─ Hit rate by key prefix (identify patterns)
   ├─ Eviction rate (too high = cache too small)
   ├─ Average TTL at eviction (evicting too soon?)
   └─ Distribution of access counts"
```

**Q3: “How do you handle a 10x traffic spike?”**

```
Answer Framework:

1. Immediate response:
   "For sudden 10x spike:
   
   ├─ Auto-scaling: Add cache nodes
   │  └─ With consistent hashing, only ~10% keys move
   │  └─ New nodes warm up gradually
   ├─ Rate limiting: Protect downstream systems
   ├─ Graceful degradation: Serve stale data if needed
   └─ Circuit breakers: Prevent cascade failures"

2. Cache-specific handling:
   "Cache layer response:
   ├─ Increase connection pool size
   ├─ Enable local caching (L1) in app servers
   ├─ Batch operations (multi-get instead of multiple gets)
   └─ Compress large values"

3. Database protection:
   "Most critical: Prevent DB overload
   ├─ Cache miss rate might increase temporarily
   ├─ Use request coalescing (dedup simultaneous requests)
   ├─ Implement queue/backpressure before DB
   └─ Serve stale cache with warning if DB struggling"

4. Post-spike analysis:
   "After spike subsides:
   ├─ Review cache hit rate during spike
   ├─ Identify any hot keys
   ├─ Adjust capacity planning
   └─ Improve monitoring/alerting"
```

### 11.3 Red Flags to Avoid

**❌ Don’t Say:**

```
1. "We can just use Redis"
   ↳ Shows lack of depth, doesn't explain design decisions

2. "Cache everything"
   ↳ Ignores cost, capacity, and eviction strategy

3. "We'll use strong consistency"
   ↳ Misunderstands cache purpose (eventual consistency is often OK)

4. "Let's implement our own cache from scratch"
   ↳ Reinventing the wheel, production systems use proven solutions

5. "Cache never goes down"
   ↳ Ignores failure scenarios
```

**✅ Do Say:**

```
1. "Let's use a proven solution like Redis, and here's why:
   - Battle-tested at scale
   - Rich data structures
   - Active community
   But we need to design proper partitioning, replication, and failover"

2. "We should cache based on access patterns:
   - Pareto principle: 20% of data gets 80% of requests
   - Let's identify hot data and prioritize
   - Set appropriate TTLs based on update frequency"

3. "For this use case, eventual consistency is acceptable because:
   - Read-heavy workload
   - Stale data won't cause critical issues
   - We can use cache-aside pattern
   Trade-off: Better availability and latency vs perfect consistency"

4. "We'll use Redis/Memcached as building blocks, and design:
   - Custom consistent hashing for partitioning
   - Application-level retry logic
   - Monitoring and alerting
   - Graceful degradation strategies"

5. "We must plan for cache failures:
   - Replication for availability
   - Circuit breakers to protect database
   - Fallback strategies
   - Regular failure testing (chaos engineering)"
```

-----

## Summary Checklist

Before your interview, ensure you can explain:

**Core Concepts:**

- [ ] Why distributed cache is needed (latency, throughput, cost)
- [ ] The problem with local caching (duplication, inconsistency)
- [ ] Database bottlenecks (connection overhead, query processing)

**Partitioning:**

- [ ] Why modulo hashing fails (massive remapping)
- [ ] How consistent hashing works (ring, clockwise rule)
- [ ] Virtual nodes (load balancing)
- [ ] Adding/removing nodes (minimal impact)

**Eviction:**

- [ ] LRU implementation (HashMap + LinkedList)
- [ ] O(1) operations (get, put)
- [ ] When to use LRU vs LFU vs others

**Cache Patterns:**

- [ ] Cache-aside (most common, read and write flows)
- [ ] Why delete cache instead of update (race conditions)
- [ ] Write-through vs write-behind trade-offs

**High Availability:**

- [ ] Master-replica replication
- [ ] Failure detection and failover
- [ ] Replication lag and eventual consistency

**Advanced Problems:**

- [ ] Hot keys (local cache, replication, coalescing)
- [ ] Thundering herd (early expiration, locking, stale-while-revalidate)
- [ ] Celebrity timeline (hybrid fan-out strategy)

**Interview Skills:**

- [ ] Requirements clarification questions
- [ ] Capacity estimation calculations
- [ ] Drawing clear architecture diagrams
- [ ] Explaining trade-offs
- [ ] Handling follow-up questions

-----

## Next Steps

**Practice:**

1. Draw the architecture from memory
1. Explain consistent hashing to someone else
1. Implement LRU cache in your preferred language
1. Walk through cache-aside pattern step-by-step

**Additional Topics to Explore:**

- Redis internals (data structures, persistence)
- Memcached vs Redis comparison
- CDN (Content Delivery Network) as distributed cache
- Browser caching strategies
- Database query caching

**Related System Design Problems:**

- Rate Limiter (uses distributed cache)
- URL Shortener (heavy caching layer)
- News Feed (timeline generation with cache)
- Notification System (cache for user preferences)

-----

**Good luck with your interviews!** Remember: The goal is to demonstrate your thinking process, not to recite a perfect solution. Ask clarifying questions, explain your reasoning, and discuss trade-offs.