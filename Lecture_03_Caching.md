# Caching Case Studies — Class Notes


---

## Table of Contents

1. [Quick Recap: Cache Invalidation Strategies](#1-quick-recap-cache-invalidation-strategies)
   - 1.1 TTL (Time To Live)
   - 1.2 Write-Around
   - 1.3 Write-Through
   - 1.4 Write-Back (Write-Behind)
2. [The 5-Step Framework for Designing a Cache](#2-the-5-step-framework-for-designing-a-cache)
3. [Case Study 1 — Scaler Code Evaluation (Online Judge)](#3-case-study-1--scaler-code-evaluation-online-judge)
   - 3.1 Problem Statement
   - 3.2 Estimating File Sizes
   - 3.3 Step 1: Establish Need for Cache
   - 3.4 Step 2: Type of Cache (Global vs Local)
   - 3.5 Why Global Cache Fails Here
   - 3.6 Latency Numbers Every Engineer Must Know
   - 3.7 Step 3: Eviction Algorithm
   - 3.8 Step 4: Invalidation Strategy — The Immutability Trick
   - 3.9 Step 5: Load Balancing Strategy
   - 3.10 Summary of Decisions — Case Study 1
4. [Case Study 2 — Contest Leaderboard](#4-case-study-2--contest-leaderboard)
   - 4.1 Problem Statement
   - 4.2 Understanding the Leaderboard Data
   - 4.3 Rough DB Schema for the Contest System
   - 4.4 Step 1: Establish Need for Cache
   - 4.5 Step 2: Type of Cache (Global vs Local)
   - 4.6 Why Global Cache is Correct Here
   - 4.7 Data Storage Design — Key-Value Structure
   - 4.8 Step 3: Eviction Algorithm
   - 4.9 Step 4: Invalidation Strategy
   - 4.10 Step 5: Load Balancing
   - 4.11 Handling the Personal Rank Row
   - 4.12 Summary of Decisions — Case Study 2
5. [Key Concepts and Side Discussions](#5-key-concepts-and-side-discussions)
   - 5.1 Why Write-Through is Hard in Distributed Systems
   - 5.2 IO-Bound vs CPU-Bound Processes (Code Evaluation Context)
   - 5.3 Cache is a Concept, Not a Physical Entity
   - 5.4 "Cache" vs "Invalidation" — Don't Mix Them
   - 5.5 The Famous Quote on Cache Invalidation
6. [Important Highlights & Faculty Emphasis Points](#6-important-highlights--faculty-emphasis-points)
7. [Questions to Ponder](#7-questions-to-ponder)
8. [Next Class Preview](#8-next-class-preview)

---

## 1. Quick Recap: Cache Invalidation Strategies

> **Note from Faculty:** These names (TTL, Write-Around, Write-Through, Write-Back) are **not standardized** across the industry. Some resources define them differently. Focus on understanding the **architecture and flow**, not just the name-to-definition mapping.

### 1.1 TTL (Time To Live)

- Every cache entry is stored **along with an expiry timestamp**.
- When a **read request** comes:
  - Check if the entry exists in cache.
  - If it exists, **also check the TTL**.
  - If TTL has passed → treat it as a **cache miss**, delete the entry, fetch fresh data from DB.
  - Re-populate cache **asynchronously** (user does not wait).
- Write requests **always go directly to DB**, which is why cache data becomes stale — TTL solves this by enforcing an expiry.
- **TTL provides eventual consistency.** A very small TTL → lots of cache misses, but fresher data. A very large TTL (e.g., Amazon sets expiry to 2046) → effectively permanent storage.
- **Adding TTL is not an overhead** — it's just 4–8 extra bytes per entry.

**Example from class (IPL Win Percentage):**
- Calculating win percentage requires multiple joins, group-by, and aggregations → takes ~18 seconds.
- A ball is delivered every few seconds → by the time you compute, data is already outdated.
- So: store a TTL of 10–15 minutes. Serve stale data; recompute asynchronously. Consistency is practically unachievable here anyway.

> ⚠️ **Problem with TTL (Thundering Herd / Cache Stampede):** After TTL expires, **all** concurrent servers detect the miss simultaneously and hammer the DB with the heavy computation query. This can take down the database.

### 1.2 Write-Around

- A scheduled **cron job** runs at a fixed interval and **rewrites the entire cache** from scratch.
- The regular read/write flow does **not touch the cache at all**.
- Write requests always go to DB.
- **Why use this over TTL?** When you have extremely heavy computations (like the IPL win% or leaderboard calculations), TTL creates the **thundering herd problem** — all servers rush to DB simultaneously. Write-around solves this by computing exactly **once** via a scheduled job, not by each server on cache miss.
- Best for: data that is expensive to compute and where eventual consistency over a fixed window is acceptable.

**When to use Write-Around vs TTL:**

| Scenario | Preferred Strategy |
|---|---|
| Simple queries, moderate data | TTL |
| Extremely heavy computation (report-style queries) | Write-Around (scheduled cron) |
| Data that should never expire | TTL with very large expiry |

### 1.3 Write-Through

- **Only strategy that provides immediate consistency.**
- On every write request: update **both DB and cache simultaneously**.
- Wait for acknowledgement (success) from **both** before returning response to user.

**Why is Write-Through so hard?**

- DB and cache are on different servers in a distributed system — no shared memory, no shared locks, no shared OS.
- Atomicity (like DB transactions) **cannot be guaranteed** across network.
- If one update succeeds and the other fails → data is inconsistent.
- **Cannot revert** because reverting is another network call that may also fail, and running conflicting update operations simultaneously makes consistency worse.
- **Retry strategies** (linear backoff, exponential backoff) are used instead of rollback.
- Used only in **critical applications**: finance, ticket booking, cart/order processing.
- **Reference:** Faculty mentioned a video on **Two-Phase Commits** as required reading for understanding the full complexity.

### 1.4 Write-Back (Write-Behind)

- All **write requests go to the cache first**, bypassing the DB.
- A **background script** periodically syncs the cache data back to the DB.
- Script frequency depends on data criticality:
  - Very important data → sync every 30 seconds / 1 minute.
  - Less important data → sync every hour / day.
- **Risk:** Cache data is stored in RAM. If the server crashes before sync → **data loss**.
- **Mitigation:** Periodically create backups (in files or a secondary DB) on the same server.
- **Eviction consideration:** Before evicting an entry (LRU/LFU/etc.), the eviction algorithm must first **write the entry to DB** — it must be "smart" about eviction.

---

## 2. The 5-Step Framework for Designing a Cache

> ⭐ **Faculty emphasis: Follow these steps in this exact order every time — in interviews, system design discussions, and real-world design.**

```
Step 1: Establish the NEED for cache
Step 2: Identify the TYPE of cache (Global vs Local; Single vs Distributed)
Step 3: Choose the EVICTION algorithm
Step 4: Choose the INVALIDATION strategy
Step 5: Design the LOAD BALANCING strategy
```

**Step 1 — Establish Need**
- Cache adds cost (infra, code complexity, network overhead, extra logic).
- You must **justify with numbers** — never say "there are many users, so we need a cache."
- Do estimations: number of users × request frequency → total requests → latency impact → decision.

**Step 2 — Type of Cache**
- **Global Cache:** A separate, dedicated cache server (e.g., Redis/Memcached). All app servers share it.
- **Local Cache:** Cache stored on the app server itself (RAM or hard disk). No separate cache server.
- **Single vs Distributed:** Depends on data volume. Small data → single server. Large data → distributed.

**Step 3 — Eviction Algorithm**
- 99.9% of the time: **use LRU** (Least Recently Used).
- Eviction handles *when to delete* entries to make space.
- Completely separate concern from invalidation.

**Step 4 — Invalidation Strategy**
- The **hardest step**. Forces you to reason about:
  - What level of consistency is needed? (immediate / eventual / none)
  - How frequently will the data change?
  - How expensive is it to recompute the data?
  - What is the user experience impact?
- Choose from: TTL / Write-Around / Write-Through / Write-Back.

**Step 5 — Load Balancing**
- Applies when you have a **distributed/global cache**.
- Options: Round Robin (stateless servers) or Consistent Hashing (stateful servers).
- **If single-server cache → usually no load balancer needed.**

> ⚠️ **Faculty Warning:** Do not mix eviction and invalidation. These solve two different problems. Keep them separate in your thinking and in interviews.

---

## 3. Case Study 1 — Scaler Code Evaluation (Online Judge)

### 3.1 Problem Statement

- A student presses **Submit** on a coding problem.
- The code travels to an **application server**.
- The server queries the **DB** to get metadata: time complexity, space complexity, test case file names/URLs, user authorization.
- Using the file URL, it downloads the **test case files from S3**.
- The test cases are run and the result is returned to the user.

**The Problem:** Test case files are very large → downloading them from S3 every time introduces massive latency.

### 3.2 Estimating File Sizes

> **Faculty tip: In HLD interviews, always back your assumptions with rough calculations. Being off by a small factor is okay; being wrong by orders of magnitude is a red flag.**

Estimation for one test case file:
- 100 test cases per file (typical for hard problems).
- Array size per test case: up to 10^6 elements.
- One integer = 8 bytes.
- Total: `100 × 10^6 × 8 bytes = 800 MB ≈ 1 GB`

So: **downloading 2 files (input + output) = ~2 GB per submission.**

At ~10 Gbps fiber speed: transferring 1 GB takes `(1 × 8) / 10 = 0.8 seconds` just in transfer time, plus network roundtrip overhead → **significant latency.**

### 3.3 Step 1: Establish Need for Cache

- Files are 1 GB+ in size.
- Every submission re-downloads them from S3.
- This is clearly creating high latency.
- ✅ **Cache is needed.**

### 3.4 Step 2: Type of Cache (Global vs Local)

- Scaler has ~3,000 problems. Each has ~1 GB file → **3 TB of data**.
- For RAM: 3 TB is enormous.
- For hard disk: manageable.

**Key Insight:** The problem is not just *where* to store — it's **why global cache doesn't help here**.

### 3.5 Why Global Cache Fails Here

> ⭐ **Faculty Highlight: This is a key architectural insight — understand it deeply.**

If you use a **global cache (separate Redis/Memcached server)**:
- App server checks metadata (DB query).
- App server queries global cache for the test case file.
- Cache hit → file has to be **downloaded over the network** from the cache server to the app server.
- Cache miss → file is downloaded from S3 to cache, then again from cache to app server.

**The problem is the same:** You are still transferring GBs of data over the network. Whether it's S3 or a cache server, the **network transfer time dominates**. This is proven by latency numbers:

### 3.6 Latency Numbers Every Engineer Must Know

*(Referenced by Faculty from "Latency Numbers Every Programmer Should Know" — benchmarked by Jeff Dean at Google)*

| Operation | Approximate Latency |
|---|---|
| L1 cache read | 0.5 ns |
| L2 cache read | 7 ns |
| RAM read | 100 ns (modern: ~10 ns) |
| Disk seek | ~10 ms |
| Sending packet over network | ~150 ms |

> **Key takeaway:** The ordering of these magnitudes never changes, even if exact numbers improve. RAM → disk → network: each tier is orders of magnitude slower. For large files, **network transfer time utterly dominates** hard disk read time.

**Conclusion:** Use a **local cache** — store test case files in the **hard disk of the application server itself**.

- First request for a problem: download from S3, store on local hard disk.
- All subsequent requests: read directly from local hard disk (no network call).
- The OS itself optimizes file access with file tables, memory-mapped files, and pagination — no extra engineering needed.

### 3.7 Step 3: Eviction Algorithm

- Use **LRU** on the hard disk.
- When disk fills up, the least recently used test case file is evicted.
- During contests, all requests for the same problem go to potentially different servers → redundant files across servers are fine; eviction handles cleanup.

### 3.8 Step 4: Invalidation Strategy — The Immutability Trick

**The Challenge:**
- If test cases are updated, all servers that have the old file cached must get the new version.
- Write-Through is very hard with local caches on stateless servers (servers don't know about each other).

**The Elegant Solution: Make Files Immutable.**

- Instead of updating a file, **create a new file** with a timestamp in the filename.
  - e.g., `testcase_problemX_20260504_143000.txt`
- The **DB entry for the problem** now points to the new filename.
- Flow on next submission:
  - App server fetches metadata from DB → gets the **new filename**.
  - Checks local hard disk → old file is there, but **new filename is not found** → cache miss.
  - Downloads the new file from S3, stores it locally.
- The old file will be evicted by **LRU naturally** (it's no longer accessed).

> ⭐ **Faculty highlight:** Immutability sidesteps the entire cache invalidation problem for this use case. You are not invalidating; you are using a new identity for the new version.

**Why we don't cache problem metadata (small DB data):**
- Metadata (~1 KB) is tiny → fast to fetch from DB each time.
- Metadata can change (e.g., file titles change when test cases are updated).
- Caching it would require write-through for immediate consistency → too complex.
- DB itself caches frequently-used rows in its own internal buffer → metadata fetches are already fast.
- **No need to cache metadata.**

### 3.9 Step 5: Load Balancing Strategy

- Since files are stored locally, **all servers become stateless** (any request for any problem can go to any server).
- If a server doesn't have the file → cache miss → downloads it → now it does.
- **Use Round Robin** load balancing.
- Stateful load balancing (consistent hashing per problem) was rejected because it concentrates load → single server overload during peak.

> ⭐ **Faculty highlight:** In a large contest, if all 100,000 students are on problem 1 and you use stateful routing → all requests go to one server → that server crashes. Stateless + round-robin = balanced load.

### 3.10 Summary of Decisions — Case Study 1

| Step | Decision | Reason |
|---|---|---|
| Need Cache? | ✅ Yes | 1 GB+ file download per submission is unacceptable |
| Cache Type | Local (hard disk of app server) | Network transfer of GBs is slow regardless of cache server location |
| Eviction Algorithm | LRU | Standard; evicts least recently accessed test case files |
| Invalidation Strategy | Immutability (version via filename + DB metadata) | Files are rarely updated; new filename = automatic cache miss on update |
| Load Balancing | Round Robin (stateless servers) | Ensures balanced load; stateful routing would overload a single server |

---

## 4. Case Study 2 — Contest Leaderboard

### 4.1 Problem Statement

Design the caching architecture for a **real-time contest leaderboard** during a 3-hour coding contest with:
- 100,000 participants
- 5 problems
- Leaderboard updates with every submission

### 4.2 Understanding the Leaderboard Data

A leaderboard typically shows:
- Rank
- Username
- Score per problem (or status: solved / attempted / unsolved)
- Cumulative score
- **Personal rank row** (your own rank shown on every page, regardless of what page you're on)
- Pagination (e.g., 10 entries per page)

### 4.3 Rough DB Schema for the Contest System

*(Faculty walked through this to establish why the query is expensive)*

Tables involved:
- **Users** (user_id, username, email, profile data)
- **Problems** (problem_id, title, test case URLs, time/space complexity)
- **Contests** (contest_id, title, duration, rules)
- **Contest_Problems** (contest_id, problem_id) — many-to-many
- **User_Contests** (user_id, contest_id) — which users are in which contest
- **Submissions** (submission_id, user_id, problem_id, contest_id, status, score, timestamp)

To build the leaderboard → **join all these tables + sort by cumulative score** for all 100,000 participants. This is an extremely complex SQL query.

**Estimated time: 3–5 seconds per query** (faculty estimate based on experience, used as a lower bound).

### 4.4 Step 1: Establish Need for Cache

**Estimation:**
- 100,000 users × 5 problems × 1 submission per problem (avg) = **500,000 total submissions**
- Over 3 hours (~10,000 seconds) → **~50 submissions per second**
- Users check the leaderboard ~every 10 minutes → 20 checks per user over 3 hours
- Total leaderboard reads = 100,000 × 20 = **2 million requests**

**Decision:**
- 2 million requests for a query taking 3–5 seconds each → **will absolutely kill the database**
- ✅ **Cache is needed.**

> ⭐ **Faculty emphasis: You cannot make cache decisions based on intuition. You need numbers. "Many users = need cache" is a red flag in an interview. Always back it with estimations.**

### 4.5 Step 2: Type of Cache (Global vs Local)

**Size of leaderboard data:**
- 1 row ≈ 100 bytes (username string + a few integers for scores and rank)
- 100,000 rows × 100 bytes = **10 MB**

10 MB is tiny. A smartwatch could cache this. This is **not a network transfer problem** unlike Case Study 1.

**Why not local cache?**
- App servers have other jobs (code evaluation is CPU-bound and memory-intensive).
- Adding 10 MB per server is wasteful redundancy.
- More importantly: **invalidation becomes extremely hard.** When the leaderboard is recalculated, you'd need to push updates to all app servers. App servers don't know about each other.
- **Use Global Cache (single server).**

### 4.6 Why Global Cache is Correct Here

- Data is small (10 MB) → network transfer is negligible.
- One single cache server can serve millions of reads.
- Invalidation is easy: cron job updates one place → all app servers get fresh data on next fetch.

### 4.7 Data Storage Design — Key-Value Structure

Cache is a **key-value store** (e.g., Redis). Keys are designed hierarchically:

**Page-level keys:**
```
Key:   contest_1:page_1
Value: [{"rank": 1, "username": "abc", "q1_score": 100, "q2_score": 200, ...}, ...]
       (JSON array stored as string)
```

**User-level keys (for personal rank row):**
```
Key:   contest_1:user_153
Value: {"rank": 3437, "score": 150, "q1": 100, "q2": 50, ...}
       (single JSON object stored as string)
```

**College-level filtration keys (optional):**
```
Key:   contest_1:college_1:page_1
Value: [top 10 from that college ...]
```

> Faculty note: Data is only a few MBs total — it fits in a single server's RAM easily. You can maintain many such keys without worry.

### 4.8 Step 3: Eviction Algorithm

- During the contest: **no eviction needed** — the leaderboard data is continuously in use.
- After the contest ends: the leaderboard becomes **static** → dump it to DB, evict from cache after a reasonable window (a few hours post-contest).
- **No active eviction algorithm required during the contest.**

### 4.9 Step 4: Invalidation Strategy

**Consistency Analysis:**

- Leaderboard computation = multiple joins + sort over 100,000 rows = 3–5 seconds minimum.
- We are getting **50 submissions/second**.
- By the time you finish one computation (5 seconds), **250 more submissions have been made** → computed data is already stale.
- **Immediate consistency is practically impossible here.**

**Why not Write-Through?**
- Even if you tried, the data would be stale within milliseconds of being written due to the submission rate.
- Implementing write-through here = wasted complexity for zero benefit.

**Why not Write-Back?**
- Changing the leaderboard 50 times/second would make it unreadable — it would look like a flickering video.

**Why not just TTL?**
- TTL creates the **thundering herd problem**: at expiry, all 2 million+ requests from all app servers rush to recompute simultaneously → DB collapses.

**✅ Solution: Write-Around (Scheduled Cron Job)**
- Run a background job every **10–15 minutes** that:
  1. Runs the heavy SQL query once.
  2. Computes ranks for all users.
  3. Writes all pages and user-rank keys to the cache.
- **Exactly one computation** happens per interval, not 2 million.
- Users see stale data, but this is acceptable — even expected.

> ⭐ **UX Tip from Faculty:** Show a countdown on the leaderboard UI: "Leaderboard refreshes in 8 minutes." This sets user expectations so they don't spam-refresh.

**Is eventual consistency acceptable here?**
- ✅ Yes. The leaderboard is a "good to have" feature. Missing a few position changes doesn't hurt the core experience of solving problems.

### 4.10 Step 5: Load Balancing

- Single cache server → **no load balancer needed** in the base case.
- For high-profile contests (Amazon hiring contest, etc.): replicate the same data in 2 cache servers, use **Round Robin** across replicas.
- Since all replicas have identical data (same cron job updates all), servers are stateless → Round Robin works perfectly.

### 4.11 Handling the Personal Rank Row

- The personal rank row is different for **every single user**.
- It cannot be served from the page-level keys.
- **Solution:** Store a separate user-level key in the same cache: `contest_1:user_<user_id>`
- Same cron job computes and writes this per-user key during its run.
- Reading personal rank: app server fetches the user's key directly → O(1) lookup.

**Why not compute it client-side?**
- The client would need all 100,000 rows to compute rank locally → way too much data to send per request.
- The cache already has precomputed per-user rank keys → just use them.

### 4.12 Summary of Decisions — Case Study 2

| Step | Decision | Reason |
|---|---|---|
| Need Cache? | ✅ Yes | 2M requests × 3–5s query = DB collapse |
| Cache Type | Global Cache, Single Server | Data is 10 MB (tiny); local cache complicates invalidation |
| Eviction Algorithm | None during contest; evict post-contest | Leaderboard is always needed during contest; static after |
| Invalidation Strategy | Write-Around (scheduled cron, every 10–15 min) | Immediate consistency is impossible; TTL causes thundering herd; Write-Around is cost-efficient |
| Load Balancing | None (single server); Round Robin if replicated | Single server handles millions of reads for 10 MB data |

---

## 5. Key Concepts and Side Discussions

### 5.1 Why Write-Through is Hard in Distributed Systems

Achieving **atomicity** across DB and cache requires:
- Both operations to succeed before returning success to user.
- No rollback mechanism (rollback = another network call that may also fail).
- If two concurrent write operations have different values, the result depends on execution order → **no determinism**.
- Solution is **retry** (with linear or exponential backoff), not revert.
- True distributed atomicity requires **Two-Phase Commit (2PC)** — very complex, high latency, rarely used.

> **Faculty Resource:** Watch the video on Two-Phase Commits (linked in class notes) to understand full distributed consistency complexity.

### 5.2 IO-Bound vs CPU-Bound Processes (Code Evaluation Context)

- **IO-Bound:** CPU spends most time waiting for I/O (mouse clicks, network data, disk reads). CPU sits idle 99% of the time.
- **CPU-Bound:** CPU continuously computes. Example: running submitted code over 100 test cases = 3–5 seconds of pure CPU time.
- **Why this matters for load balancing:** If all 100,000 users submit problem 1 simultaneously and routing is stateful (all go to server A), server A is CPU-bound and crashes. Stateless routing + round robin distributes this load across all servers.
- **CPU Speed Context (Faculty example):**
  - 5 GHz CPU = 5 × 10^9 clock ticks/second → one clock tick = 0.2 ns
  - One floating point addition (64-bit) per clock tick
  - Quad core: 4 additions per 0.2 ns
  - In the time it takes a photon to travel 30 cm from screen to eye (~1 ns), the CPU can do **20 floating point additions**
  - CPU is incredibly fast; the bottleneck is almost always **memory latency**, not compute speed.

### 5.3 Cache is a Concept, Not a Physical Entity

> ⭐ **Faculty Highlight**

Cache is not tied to RAM specifically. It is the concept of storing data in a **faster layer** relative to the primary source:
- You can use RAM as cache (Redis, Memcached).
- You can use the **local hard disk as cache** (Case Study 1).
- You can use S3 as a cache relative to a remote origin.
- Your OS already implements a cache (file table, memory-mapped files, demand paging).

The key property: the cache layer is **faster to access** than the source, and stores a **subset** of the source data.

### 5.4 "Cache Eviction" vs "Cache Invalidation" — Don't Mix Them

| | Eviction | Invalidation |
|---|---|---|
| **What it solves** | Space management — when cache is full, what to delete? | Consistency — how to keep cache in sync with DB? |
| **When it runs** | When cache reaches capacity | Based on TTL / writes / scheduled jobs |
| **Examples** | LRU, LFU, FIFO, LIFO | TTL, Write-Through, Write-Around, Write-Back |
| **Concern level** | Usually straightforward (LRU 99% of the time) | Hardest design decision in caching |

> These are two completely independent problems. When solving one, do not think about the other.

### 5.5 The Famous Quote on Cache Invalidation

> **Phil Karlton** (Senior Engineer at Netscape, widely quoted in CS):
> *"There are only two hard things in Computer Science: cache invalidation and naming things."*

Faculty extended this with a colleague's version:
> *"There are actually three hard things: naming variables, concurrency issues, cache invalidation — and off-by-one errors."*

---

## 6. Important Highlights & Faculty Emphasis Points

> These are points that were repeated, stressed, or called out specifically as important during the class.

1. **⭐ Always do estimations before deciding on cache.** "There are so many users, so we need cache" is a red flag in interviews. Come with numbers and reasoning.

2. **⭐ Cache invalidation is the hardest part of caching design.** It requires understanding business context, consistency requirements, and data update frequency.

3. **⭐ Eviction ≠ Invalidation.** These solve different problems. Never conflate them in an interview or system design.

4. **⭐ The 5-step framework must be followed in order.** Skipping to load balancing before deciding invalidation leads to backtracking and confusion.

5. **⭐ Cache is a concept, not a technology.** Hard disk can be a cache. RAM can be a cache. Context determines what "faster" means.

6. **⭐ For large file caching, a global cache doesn't help.** The bottleneck is network transfer of the file, not computation. Local hard disk cache eliminates the network step.

7. **⭐ Immutability is a powerful invalidation bypass.** If you can make data immutable and use versioned identifiers (filenames with timestamps), cache invalidation complexity disappears.

8. **⭐ In Write-Around, computation happens exactly once.** This is the key difference from TTL — TTL can cause N servers to simultaneously compute the same expensive query. Write-Around centralizes computation in a single scheduled job.

9. **⭐ Stateful vs Stateless servers affect your cache strategy.** Stateful routing (consistent hashing) can concentrate load on one server during contests → use stateless (round robin) when possible.

10. **⭐ In HLD interviews, you will not be asked "design a cache" directly.** You'll design a larger system and then decide whether cache is warranted. The decision should be data-driven.

11. **⭐ Naming of strategies is not standardized.** Different textbooks/companies use different names. Understand the architecture, not just the name.

---

## 7. Questions to Ponder

*(These were questions raised during the class for student reflection)*

1. **Why do we need Write-Around if TTL works in almost all cases?**
   *(Hint: Think about what happens when TTL expires for a very expensive computation with millions of concurrent users.)*

2. **Which consistency model does TTL-based invalidation generally provide?**
   *(Eventual consistency. What are the tradeoffs of a very small vs very large TTL?)*

3. **What is the tradeoff when choosing a very low TTL?**
   *(High cache miss rate → more fresh data, but more DB load. Is there a sweet spot?)*

4. **If we are using a local cache for test cases, and the test case file is updated, how do we ensure that all app servers get the updated version without implementing write-through?**
   *(The immutability trick — new filename → automatic cache miss.)*

5. **In the leaderboard case, can we ever achieve consistent data? Why or why not?**
   *(No — leaderboard takes 3–5 seconds to compute; 50 submissions happen per second → data is stale before computation finishes.)*

6. **Why does Write-Back require the eviction algorithm to be "smart"?**
   *(Before evicting an entry, the algorithm must first flush it to DB to prevent data loss.)*

7. **In a large scale contest, why would routing all problem-1 requests to the same server be a disaster?**
   *(Code evaluation is CPU-bound. One server with 4 cores can evaluate 4 codes at a time. 10,000+ concurrent submissions would crash it.)*

8. **Why is Write-Through practically impossible when using local caches across stateless servers?**
   *(Each server has its own local cache. Writing to all of them requires each server to know about all others — they are supposed to be agnostic.)*

---

## 8. Next Class Preview

- **Next topic:** Caching Case Study — **Social Media Newsfeed**
- After completing the Social Media Newsfeed case study, the instructor plans a **lab session** for hands-on implementation of the caching concepts.
- Future classes will also cover: **Capacity Estimation in HLD Interviews** — you are expected to derive request rates and scale from first principles, not ask the interviewer for numbers.

---

