# HLD — Caching (Class 2)
### Scaler Academy | High Level Design

---

## 📋 Table of Contents
1. [Quick Recap — Class 1](#1-quick-recap--class-1)
2. [Cache Eviction Policies](#2-cache-eviction-policies)
3. [Cache Invalidation vs Eviction](#3-cache-invalidation-vs-eviction)
4. [Time To Live (TTL)](#4-time-to-live-ttl)
5. [Consistency — Immediate vs Eventual](#5-consistency--immediate-vs-eventual)
6. [Cache Write Policies](#6-cache-write-policies)
   - [TTL-Based Invalidation](#61-ttl-based-invalidation)
   - [Write-Around](#62-write-around)
   - [Write-Through](#63-write-through)
   - [Write-Back](#64-write-back)
7. [Edge Computing](#7-edge-computing)
8. [Case Study — Online Judge (InterviewBit / Scaler)](#8-case-study--online-judge-interviewbit--scaler)
9. [Pareto Principle in Caching (80/20 Rule)](#9-pareto-principle-in-caching-8020-rule)
10. [Questions to Ponder](#10-questions-to-ponder)
11. [Homework](#11-homework)
12. [Upcoming Topics](#12-upcoming-topics)

---

## 1. Quick Recap — Class 1

### Caching Architecture — End to End Flow

```
Client → DNS (IP cached in browser)
       → Load Balancer (can act as Reverse Proxy / cache layer)
         → Application Servers (Local Cache: RAM / HDD)
           → Global Cache (Redis / Memcached)
             → Database (SQL / NoSQL)
                → File Storage (S3 — for large files)
```

### Two Types of Server-Side Caches

| Type | Where stored | Accessible by |
|---|---|---|
| **Local Cache** | RAM or HDD of the same application server | That single server only |
| **Global Cache** | Separate dedicated cache server (e.g., Redis) | All application servers |

> If the data to cache is small → single global cache server  
> If large → distributed global cache (data spread across multiple cache servers)

### CDN — Key Takeaways
- ~**70% of the internet's traffic is routed through CDNs**
- CDN = **edge computing for static data** (JS, CSS, images, videos)
- CDNs are **dumb servers** — no business logic lives there
- Business logic must stay in **one single processing unit** (application servers)
- When you open Amazon.com → thousands of sub-requests fire → most served by CDN, not Amazon's core servers

> ⚠️ **Why only static data in CDN?**  
> Dynamic data changes frequently. If stale data sits in CDN, you serve wrong data to users. Keeping CDN in sync with DB for dynamic data is an extremely hard consistency problem. Making CDN in sync with dynamic data will make CDN busy as now it has to handle frequent write as well as read requests and it increases its chance to go down. 

---

## 2. Cache Eviction Policies

> **Problem being solved:** Cache has **limited storage**. When it fills up, what do you delete?

| Policy | Logic | Use Case |
|---|---|---|
| **LRU** (Least Recently Used) | Delete the item used furthest in the past | 99% of real-world systems |
| **LFU** (Least Frequently Used) | Delete the item with lowest access count | Specific use cases |
| **FIFO** (First In First Out) | Delete the oldest inserted item | Simple / dumb cache |
| **LIFO** (Last In First Out) | Delete the most recently inserted item | Rare |

### 🔴 Important Faculty Point
> *"You can close your eyes and go with LRU. It works 99% of the time. LRU is the most frequently asked cache implementation in interviews — not LFU or FIFO — because it's what real systems actually use."*

### When to think about eviction policy at all?
- First, optimize: better code, better DB schema design, better data structures, better algorithms, smarter load balancing, use client machine storage.
- Only **after exhausting all other options** should you benchmark different eviction policies.
- In practice, **LRU selection is not a decision you should spend time on** during system design interviews — pick it and move on.

---

## 3. Cache Invalidation vs Eviction

> ⚠️ **This is a very easy trap to fall into. Faculty stressed this multiple times.**

| Dimension | Cache Eviction | Cache Invalidation |
|---|---|---|
| **Problem solving** | Cache is **full** → need to free space | Cached data is **stale** → need to remove/refresh it |
| **Trigger** | Storage capacity reached | Data has changed in the database |
| **Storage state** | Cache may be completely full | Cache may have plenty of free space |
| **Think of it as** | Garbage collection for space | Correctness enforcement |

> Analogy used by faculty:  
> **Medicine box at home** → You don't proactively check all medicines for expiry dates. You check the expiry date only when you actually need to use a medicine. Same logic applies to cache invalidation — don't proactively scan and delete, do it **lazily at read time**.

### Why NOT proactive deletion?
- Running a script to scan millions of cache entries for expired TTLs keeps the cache **server busy** and unavailable for serving read requests.
- Lazy deletion (check TTL only when read is requested) is far more efficient.

---

## 4. Time To Live (TTL)

### What is TTL?
Every cache entry is stored as a key-value pair with an additional TTL field:

```
Cache Entry = {
    key:   "problem_123_testcases",
    value: <data>,
    ttl:   <timestamp or duration>
}
```

When a read request hits the cache:
1. Data found? → Check TTL
2. TTL not expired → serve data ✅
3. TTL expired → **delete from cache**, query database, re-populate cache

### TTL values are use-case dependent

| Example | TTL |
|---|---|
| Amazon static assets (CSS, images) | ~20 years (effectively permanent) |
| User session data | Minutes to hours |
| Product availability | Minutes |
| News feed content | Hours |

---

## 5. Consistency — Immediate vs Eventual

### The Core Problem
```
Write Request: A = 100

DB  → A = 100  ✅ (updated)
Cache → A = 10  ❌ (stale)
```
A read from cache now returns wrong/stale data.

### When do you need Immediate Consistency?

| Use Case | Consistency Required | Why |
|---|---|---|
| **Banking / Payments** | Immediate | Money transactions — wrong data = financial loss |
| **Stock Trading** | Immediate | Millisecond decisions, billions at stake |
| **Seat booking (actual seat map)** | Immediate | User is selecting seats right now |
| **Product page inventory** | Immediate | User is about to purchase |
| **Online judge test cases** | Immediate | Wrong test cases → user loses contest marks / job opportunity |

### When is Eventual Consistency acceptable?

| Use Case | Why eventual is OK |
|---|---|
| **Search results page (e-commerce)** | Showing slightly stale product listings is acceptable |
| **Win prediction in cricket** | Approximate number, algorithm-dependent, not exact |
| **Instagram views/likes count** | Trend matters, exact number doesn't — "2.1M" not "2,153,741" |
| **Movie listing page** | Full seat info shown only when you drill down |

### 🔴 Important Faculty Point — Live LinkedIn Poll
> Faculty ran a live LinkedIn poll and showed that different students saw different poll results at the same time — a **real-time demonstration of eventual consistency**. After refreshing, the updated number appeared — showing that the data eventually converged to the correct state.

---

## 6. Cache Write Policies

Four strategies for keeping cache in sync with the database.

---

### 6.1 TTL-Based Invalidation

**Mechanism:**  
- On every cache write, attach a TTL (expiry time)
- On read: if TTL has expired → delete entry → query DB → re-populate cache asynchronously
- User does NOT wait for cache re-population (async write-back to cache)

```
Read Request
    ↓
Cache hit? → Check TTL
    TTL valid → return data ✅
    TTL expired → delete from cache → query DB → return data (re-populate async) ✅
Cache miss → query DB → return data → populate cache (async) ✅
```

**Consistency:** Eventual  
**Latency:** Low (user doesn't wait for cache update)  
**Use When:** Data staleness is acceptable for some time window

---

### 6.2 Write-Around

**Mechanism:**  
- A write-around cache is a caching strategy where data is written directly to the permanent storage (database) while completely bypassing the cache.  
- The cache is only updated when that data is subsequently read.  
- This approach is highly effective for write-intensive workloads where the written data is not immediately or frequently read back.  

**How Write-Around Cache Works:**  
- Application Writes Data: The application sends a write command
- Bypass Cache: The data goes directly to the database. The cache is not updated.
- Read Operation: When the application needs to read that same data, it checks the cache.
- Cache Miss: Since the write bypassed the cache, the first read will likely result in a "cache miss" (data not found).
- Cache Populated: The system fetches data from the database, populates the cache for future requests, and returns it to the user.
- **No TTL** on cache entries
- A **Cron Job** runs at configured intervals (e.g., every 5 overs of a cricket match)
- Cron job executes heavy computation (joins, subqueries, aggregations) against the DB
- Stores the result in cache, **overwriting without checking existing value**

```
Cron Job (runs every N minutes/hours)
    → Complex DB query (joins, aggregations)
    → Result stored in cache (overwrite)

Read Request
    → Cache hit → serve data (may be slightly stale between cron runs)
```

**Consistency:** Eventual (bounded by cron frequency)  
**Latency:** Very low for reads (heavy computation done offline)  
**Use When:** Data is result of heavy computation AND slight staleness is perfectly acceptable

**Real-world example from class:**  
> **Cricket Win Prediction** — Calculated from team lineup, player stats (recent 10-20 matches), ground stats, inning type, time of day. This requires 20-30 table joins. Running this on every ball is overkill. A cron job re-calculates it every few overs and updates the cache.  
> Faculty: *"Even if the win prediction is completely wrong, it's okay — it actually becomes a legend when we win from 3% odds (India vs Pakistan)."*

---

### 6.3 Write-Through Cache

**Mechanism:**  
- On every write → write to **DB first** (wait for ACK) → then write to **cache** (wait for ACK)
- User waits until **both** writes are confirmed (**Two-Phase Commit**)
- If either fails → retry / rollback

```
Write Request
    ↓
Write to DB → ACK from DB ✅
    ↓
Write to Cache → ACK from Cache ✅
    ↓
Return success to user
```

**Consistency:** Immediate  
**Latency:** HIGH (user waits for 2 network round trips)  
**Use When:** Immediate consistency is a hard requirement (payments, bookings, etc.)

### 🔴 Important Faculty Point — Two-Phase Commit Complexity
> Faculty used the **n-body problem analogy** from physics:  
> - 2 bodies orbiting each other (Two-body problem) — solvable but complex  
> - Add a third body → exponentially harder  
> - Add more → practically unsolvable  
>
> **Same in distributed systems**: 2-phase commit across 2 servers is hard enough. At scale with thousands of servers + replicas, maintaining atomicity across all of them is one of the **hardest problems in high-level design** → covered in the CAP Theorem class.

### Write-Through in LOCAL Cache vs GLOBAL Cache

| Scenario | Difficulty | Why |
|---|---|---|
| Write-through with **local cache** | **Very easy** | Cache is in RAM of same server. Writing to RAM = updating a variable in Java. Cannot fail unless server goes down. No network call. |
| Write-through with **global cache** | **Extremely hard** | Separate server over a network. Network can fail, server can crash, response may timeout. Two-phase commit complexity. |

> 🔑 **Key Insight**: *The hardest consistency problem in distributed systems becomes trivially easy the moment you move to a local cache.*

### Stateful Servers with Write-Through Local Cache
When using write-through local cache for immediate consistency, **application servers must become stateful**:
- The load balancer can no longer use round-robin
- Requests for the **same item/user must always route to the same server** (consistent hashing)
- If eventual consistency is acceptable → servers can remain stateless

---

### 6.4 Write-Back Cache

**Mechanism:**  
- Write request goes to **cache first** (user gets immediate ACK)
- DB is updated **eventually** via a cron job
- Cache is the **source of truth** temporarily

```
Write Request
    ↓
Write to Cache → ACK immediately to user ✅
    ↓ (async, periodic)
Cron Job: Cache → DB sync (bulk writes)
```

**Consistency:** None guaranteed (data may be lost if cache crashes before DB sync)  
**Latency:** Lowest possible for writes  
**Risk:** If cache server crashes before DB sync → data is **permanently lost**

**Real-world example from class:**  
> **Instagram Reel Views/Likes** — Millions of users watching/liking a viral reel simultaneously would cause millions of write locks on the same DB row → DB crash. Instead:  
> - Each engagement hits the cache  
> - Cache is distributed (fast, no locking)  
> - Periodically synced to DB  
> - Some view counts may be lost if cache crashes — business is OK with this  
> - Instagram shows "2.1M views" not exact number — because exact number is unknowable (eventual/approximate)

> Faculty: *"You may have liked a reel but your like doesn't appear sometimes. Your interaction was simply lost. The business is completely fine with that."*

**Use When:** 
- Data is approximate by nature (view counts, like counts)
- High write volume with low business impact if some data is lost
- Trend/order-of-magnitude accuracy is sufficient

---

### Summary — Cache Write Policies

| Policy | Consistency | Write Latency | Risk | Best For |
|---|---|---|---|---|
| **TTL** | Eventual | Low | Stale data | Most general use cases |
| **Write-Around** | Eventual (cron-bounded) | N/A (offline) | Stale between cron runs | Heavy computation, low-impact data |
| **Write-Through** | **Immediate** | High | Slow writes | Payments, bookings, critical data |
| **Write-Back** | None / Eventual | **Lowest** | Data loss if cache crashes | View counts, likes, non-critical metrics |

---

## 7. Edge Computing

- **Edge** = server kept geographically close to the client
- CDN is an example of edge computing for **static data**
- Application servers can also be placed at the edge (AWS Availability Zones / Regions)
- But maintaining application servers across every city is **operationally expensive** (physical maintenance, cost)
- That's why CDN solves the common case (static data) and application-level edge is only done at a coarser granularity (region level, not city level)

---

## 8. Case Study — Online Judge (InterviewBit / Scaler)

### System Flow
```
User writes code → Presses Submit
    ↓
Load Balancer → Application Server
    ↓
[1] Query SQL DB → get URLs of test case files (input + expected output)
    ↓
[2] Download test case files from S3 (~600MB - 2GB per problem)
    ↓
[3] Execute user code against test cases
    ↓
[4] Compare output → return result (AC / WA / TLE / MLE)
```

### The Problem
> Downloading **1GB+ of test case files from S3 on every submission** is the main bottleneck driving latency.

### Why test case files are so large
- Must distinguish O(n log n) from O(n²) implementations → needs thousands of test cases to get measurable time differences
- Multiple file types: correctness files, memory limit files, time limit files
- Per-problem estimate: ~600MB–2GB → **Total for 3000 problems ≈ 3TB**

### Why CDN is NOT the right solution here

> ⚠️ **Common misconception that the faculty explicitly addressed**

CDN = **client-facing technology**. Test case files need to go to **application servers**, not the browser.
- Using CDN would route data from a CDN node near the **client** → round trip across a continent
- In AWS, S3, RDS, EC2 are all connected via **high-speed internal fiber** → very low intra-datacenter latency
- Adding CDN between S3 and application servers would **increase** latency, not reduce it

> *"CDN should ONLY be used for serving data you want to provide to the client. Never for server-to-server communication."*

### Proposed Solution — Local Cache on Application Servers

**What to cache:** Test case files for the **~400-500 most frequently accessed problems** (Pareto principle → see Section 9)

**Where to cache:** Local hard disk or RAM of the application server

**Why local and not global:**
- A global Redis server still requires a download over network
- S3 → Redis → App server = same download problem
- S3 → App server's own HDD = no extra network hop, directly executable

**Open Design Questions (to be answered in next class):**
1. Should these servers be **stateful or stateless**?
2. If stateful — stateful with respect to **problem** or **user**?
3. What should the **eviction algorithm** be?
4. What should the **invalidation strategy** be?
5. What happens if **test cases are wrong and updated during a contest** → how do we ensure all cached copies get invalidated immediately?
6. Do we need **immediate consistency or eventual consistency** for test case updates?

---

## 9. Pareto Principle in Caching (80/20 Rule)

> *"80% of the results are driven by 20% of the inputs."*

**Applies everywhere:**
- 20% of websites → 80% of internet traffic
- 20% of users → 80% of social media content
- 20% of products → 80% of e-commerce revenue

**Applied to the Online Judge:**
- 3000 total problems
- ~400-500 problems accessed on a regular basis
- Remaining ~2500+ problems: 0 hits on most days, with spikes only during specific contests

**Impact on caching design:**
- You do **NOT** need to cache all 3TB of test data
- Cache the ~400-500 hot problems → fits in the HDD of an application server
- Cold problems (accessed during contests only) can be downloaded from S3 on demand
- CDNs themselves use this principle — they don't cache a URL until it crosses a **threshold hit count** (to avoid wasting space on zero-traffic websites)

### Real-world CDN anecdote from class
> Ad companies like Media.Net serve millions of ads. For generic ads, they're OK with the first 5-6 users having slow experience (CDN hasn't cached yet). But for high-value targeted ads (like Rolls Royce), they **hit the CDN themselves in advance** to pre-warm the cache — ensuring zero buffering for the actual target audience.

---

## 10. Questions to Ponder

> *(Questions raised by the faculty during class for student reflection)*

1. **Why do we store only static data in CDNs, and not dynamic data?**  
   *(Hint: Think about consistency + CDN being a dumb server with no business logic)*

2. **Can you give examples of systems where:**
   - Immediate consistency is mandatory
   - Eventual consistency is perfectly acceptable
   - Losing some data entirely is OK

3. **For the online judge system:**
   - Should application servers be stateful or stateless if we introduce a local cache?
   - Should statefulness be per-user or per-problem?
   - What is the right invalidation strategy when test cases are updated mid-contest?

4. **Why can't we simply do a two-phase commit on every write to keep cache and DB always in sync?**  
   *(Hint: Think n-body problem, distributed systems complexity)*

5. **If you were implementing write-through cache, would you implement it as a local cache or global cache, and why?**

6. **Win prediction for a cricket match — is eventual consistency okay here? Should it be recalculated every ball?**

7. **Instagram reel views — why doesn't Instagram show exact view counts once views cross a threshold?**

---

## 11. Homework

> **Design the complete caching system for the Online Judge (Scaler/InterviewBit)**

Given the scenario:
- ~3000 problems, ~400-500 frequently accessed
- Test case files: 600MB – 2GB per problem (stored in S3)
- Test cases may need to be updated (infrequently, but sometimes mid-contest)
- Many users submit concurrently during contests

**You need to decide and justify:**

1. **What to cache** — which data, which problems
2. **Where to cache** — local vs global cache, which layer
3. **Stateful vs stateless** servers — statefulness with respect to problem or user?
4. **Eviction policy** — which algorithm and why
5. **Invalidation strategy** — what happens when test cases are updated mid-contest? Do we need immediate consistency or eventual?

> ⚡ **Note from faculty:** A **5-step framework** for solving any caching problem will be presented in the next class. This homework is meant to be attempted before that — try to reason through it on your own first.

---

## 12. Upcoming Topics

- **5-Step Caching Framework** — a systematic approach for deciding: what to cache, where, eviction, invalidation, consistency model
- **Solution walkthrough** for the Online Judge caching design
- **CAP Theorem** — the fundamental tradeoff between Consistency, Availability, and Partition tolerance
- **Case Studies:** Hotstar (live streaming) and Netflix (recorded video streaming)
- **Leaderboard system design** (write-back cache use case, mentioned briefly)
- **Types of data storage** — SQL vs NoSQL vs S3 vs other specialized storage

---

