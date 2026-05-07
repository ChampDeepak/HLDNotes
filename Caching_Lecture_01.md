# 📚 Caching - Detailed Lecture Notes (Caching 1)

## 📋 Table of Contents
- [🔁 Quick Recap from Previous Class](#-quick-recap-from-previous-class-sharding--consistent-hashing)
  - [What is Sharding?](#what-is-sharding)
  - [Why is the entire data of one user kept on one machine?](#why-is-the-entire-data-of-one-user-kept-on-one-machine)
  - [Request Flow (Recap)](#request-flow-recap)
- [📊 Sharding Algorithms](#-sharding-algorithms)
  - [1️⃣ Simple Hash-Based / Round Robin](#1️⃣-simple-hash-based--round-robin-using-mod)
  - [2️⃣ Consistent Hashing](#2️⃣-consistent-hashing)
- [🩺 Health Check vs Heartbeat](#-health-check-vs-heartbeat)
- [📖 Teacher's Real Story: Snapdeal](#-teachers-real-story-snapdeal-why-consistent-hashing-matters)
- [🏗️ Stateful vs Stateless Servers](#️-stateful-vs-stateless-servers)
- [⏱️ What is Latency?](#️-what-is-latency)
- [🍵 The Tea Analogy (Why Caching = Layers)](#-the-tea-analogy-why-caching--layers)
- [🧠 More Memory Layering Examples](#-more-memory-layering-examples)
- [🌐 Client-Side Caching](#-client-side-caching-where-we-can-cache)
- [🖥️ Server-Side Caching](#️-server-side-caching-where-caches-can-live)
- [🌎 Global Cache vs Local Cache](#-global-cache-vs-local-cache)
- [🔒 Why Cache Should Be "Dumb"](#-why-cache-should-be-dumb)
- [🔑 Cache Storage Format](#-cache-storage-format)
- [⚠️ Two Major Challenges with Caching](#️-two-major-challenges-with-caching)
- [❌ Caching is Useless / Harmful When...](#-caching-is-useless--harmful-when)
- [🎯 Summary: All Possible Cache Locations](#-summary-all-possible-cache-locations)
- [📌 Key Teacher Quotes & Principles to Remember](#-key-teacher-quotes--principles-to-remember)
- [🔮 What's Coming Next (Teaser)](#-whats-coming-next-teaser)
- [📚 Recommended Resources](#-recommended-resources-teachers-suggestions)

---

## 🔁 Quick Recap from Previous Class (Sharding & Consistent Hashing)

Before jumping into caching, the teacher quickly reviews concepts from the previous class because caching builds on the same architectural foundation.

### What is Sharding?
- Sharding is a database architecture pattern that splits a single, large dataset into smaller, faster, and more manageable pieces called shards. Each shard resides on a separate server, enabling horizontal scaling to boost performance, improve reliability, and handle high-volume traffic that exceeds one machine's capacity.
- This distribution is called **sharding**.
- You cannot divide data randomly — you need a **parameter** to decide which data goes where.
- That parameter is called the **sharding key**.

**Examples of sharding keys:**
- **Delicious / Social media app** → User ID
- **E-commerce** → Product ID or Product Type
- **Netflix** → Could be something else entirely (depends on use case)

### Why is the entire data of one user kept on one machine?
- If a user's data is spread across multiple servers, the load balancer would have to:
  1. Reach out to multiple servers
  2. Collect data from each
  3. Merge it
  4. Then return it
- This **increases latency** drastically.
- So we keep all of one user's data inside **one single server**.

### Request Flow (Recap)
```
Client → DNS → Load Balancer IP → Load Balancer → Application Server → Database
```
- DNS stores IPs of multiple load balancers and gives one at random to the client.
- Load balancer must know which server holds the data of a specific user.

---

## 📊 Sharding Algorithms

### 1️⃣ Simple Hash-Based / Round Robin (using `mod`)
- Formula: `serverIndex = userID % numberOfServers`
- ✅ **Pros:** Very fast, easy to implement
- ❌ **Cons:**
  - When you **add a new server**, almost ALL data must be redistributed across all servers.
  - There's no fixed pattern — data flows in every direction.
  - This causes huge **network costs and downtime**.

> **🔑 Important point from teacher:** When should you add a new server? Only when existing servers reach **80–90% capacity**. Don't wait till they go out of memory — scale BEFORE that.

> **🔑 Key principle:** When a new server is added, it should:
> - Reduce load on **all existing servers uniformly**
> - Data flow should be **only in one direction** (existing → new), NOT across all servers.

### 2️⃣ Consistent Hashing
The improved approach. The problem with mod-based hashing is that the count of servers is **inside the function** — when count changes, function output changes.

**Solution:** Use a hash function that does **NOT include the count of servers**.

#### How it works:
1. Use a hash function that returns values in a **very large range** (e.g., 0 to 10^18).
2. Why a large range? → To avoid **collisions** (two different inputs getting the same hash).
   - Probability of collision = 1 / 10^18 → practically zero.
3. Take each **server ID** → pass through hash function → get values like `x`, `y`, `z`.
4. Place these on a **sorted array** (visualized as a circular ring).
5. For an incoming request:
   - Take the user ID
   - Pass through the **same hash function**
   - Find the **next greater element** in the sorted array (using binary search)
   - That server handles the request
6. The arrangement is **cyclic** — if no greater element exists, wrap around to the first one.

> **🔑 Important note:** The "ring" is just a mental model. Actual implementation is a **sorted array with binary search** inside the load balancer.

#### Distribution
- We don't aim for **perfectly equal** distribution (e.g., exact 25% on each of 4 servers — that's impossible without lots of rules).
- We aim for **roughly uniform** (e.g., 20%, 30%, 18%, 32% — that's acceptable).
- When servers near max capacity → add a new one.

#### Problem with Consistent Hashing
- If a server **dies**, its entire load goes to the **next server** in the ring.
- That next server may now get overloaded → it dies too → **cascading failure**.

#### Solution: Virtual Nodes
- Instead of placing each server once on the ring, place it **multiple times** at different hash positions (using server_id + suffix combinations).
- This means a server's load gets distributed across **many small ranges** on the ring.
- When that server dies, its load is **spread among many other servers**, not just one.
- The more virtual nodes per server, the more uniform the redistribution.

---

## 🩺 Health Check vs Heartbeat

How does the load balancer know if a server is alive?

| Aspect | Health Check(Are u ok?) | Heartbeat (I'm alive)|
|--------|--------------|-----------|
| Direction | Load balancer → Server (pings periodically) | Server → Load balancer (informs it's alive) |
| Performance | Almost the same | Almost the same |
| Design quality | ✅ **Better** | ❌ Violates separation of concerns |

> **🔑 Why is Health Check better? (SOLID Principles)**
>
> The **Single Responsibility Principle** says each component should have one job:
> - The **server's job** is only to take a request, process it, and return a response.
> - The server should be **agnostic** of load balancers.
> - With heartbeat, the server now also has to know about load balancers and keep informing them — extra responsibility.
> - With health check, the load balancer (whose job IS to track server status) does the work.

> **💡 Real-world note:** Most load balancers implement health check, but some implement heartbeat for specific use cases.

---

## 📖 Teacher's Real Story: Snapdeal (Why Consistent Hashing Matters)

The teacher worked at Snapdeal where:
- Originally only 2 engineers were left after a mass exodus.
- They had to upgrade search servers every 6 months (vertical scaling).
- Then **Aamir Khan campaign** brought aggressive traffic.
- Servers started filling up earlier than 6 months → kept upgrading vertically.
- Eventually CTO said: "Servers are too costly — find a technical solution."
- They first implemented **simple hash-based sharding** in 2 weeks (gave them 6 months runway).
- Then implemented **consistent hashing** properly.

> **🔑 Important Real-World Insight:**
> Even with uniform user distribution, **load may NOT be uniform**. Some users (e.g., book sellers on Snapdeal, celebrities/influencers on social media) generate **100x more data** than average users.
>
> **Solution:** Combine horizontal scaling with **vertical scaling** for those special heavy users — give them dedicated, high-config servers.

---

## 🏗️ Stateful vs Stateless Servers

### Stateful Servers
- Store data themselves.
- Can't randomly distribute requests — load balancer **must know** which server holds which user's data.
- This is why we need consistent hashing.

### Stateless Servers
- Don't store data — they only process.
- Database is separate.
- Load balancer can use **simpler algorithms** (random, round-robin) because all servers are identical.

### Modern Architecture
```
Client → DNS → Load Balancer (LB1) → Application Server (Stateless) →
        → DB Load Balancer (LB2) → Database Server (Sharded)
```
- Most database systems provide **internal load balancing**.
- Often, the application server itself is intelligent enough to reach the right database directly (avoiding extra network hop).

---

## ⏱️ What is Latency?

> **Latency** = the time taken for the client to get the response after sending a request.

**Goal of architecture:** Minimize latency.

### Where does latency come from?
1. **Network calls** — every hop adds time:
   - Client → DNS
   - Client → Load Balancer
   - Load Balancer → App Server
   - App Server → Database
2. **Processing time** at load balancer (e.g., consistent hashing computation)
3. **Application code execution** (your DSA helps here — better algorithms = lower latency)
4. **Database query execution** (joins, subqueries, etc.)

> **🔑 Key insight:** If servers are stateless, **don't use consistent hashing** — it's `O(log n)` and adds CPU work. Use a simpler O(1) algorithm.

---

## 🍵 The Tea Analogy (Why Caching = Layers)

Teacher's brilliant analogy:

You want to make tea. Where do you get sugar?
- ❌ Not from the sugar mill
- ❌ Not from the supermarket every time
- ✅ From a **small jar in the kitchen** (fast access, small amount)
- When that jar is empty → refill from a **larger jar in storeroom**
- When that's empty → go to **supermarket** (slow, but huge supply)

**Lesson:** Memory always works in **layers**:
- Smaller, faster, closer storage at the top
- Larger, slower, farther storage at the bottom

---

## 🧠 More Memory Layering Examples

### Human Brain
| Layer | Capacity | Speed |
|-------|----------|-------|
| Working memory | 3-4 items | Instant |
| Short-term memory | A few days | Fast |
| Long-term memory | Lifetime | Slower retrieval |

> **🔑 Important point from teacher:** REM sleep is critical to convert short-term → long-term memory. Sleep is the "downtime" your brain needs.

### CPU Memory Hierarchy
```
Registers → L1 Cache → L2 Cache → L3 Cache → RAM → Hard Disk
(Fastest, smallest)              (Slowest, largest)
```

> **🔑 Universal principle:** Wherever there is memory, it is **always organized in layers** — small & fast at top, large & slow at bottom.

---

## 🌐 Client-Side Caching (Where We Can Cache)

### 1. Browser Storage Types

| Storage | Size | Scope | Use Case |
|---------|------|-------|----------|
| **Cookies** | 4-5 KB | Per HTTP request | Auth tokens, small session info |
| **Session Storage** | ~5 MB | Single tab only | Form state, temporary data |
| **Local Storage** | ~5-10 MB | Whole browser, all tabs | User preferences (dark mode, etc.) |
| **IndexedDB** | ~10 GB | Per browser | Large data — images, videos, browsing history (NoSQL document store) |

> **🔑 Important:** Cookies must stay tiny because they travel with **every HTTP request** — large cookies = high latency.

> **🔑 Note:** When you "clear browsing data" and it shows "2 GB of images" — that's stored in **IndexedDB**.

### 2. DNS Caching
- After the first DNS lookup, the browser **caches the IP address**.
- Next time you visit the same site, no DNS call needed.
- You can view your DNS cache at: `chrome://net-internals/#dns`

### 3. CDN (Content Delivery Network) — The Big One

**Problem:** If Netflix servers are in USA and you're in Bangalore, streaming a movie frame-by-frame across continents = high latency. Speed of light is the upper limit, and networks fluctuate below that.

**Solution:** Store **static content** in servers physically close to users.

#### What is a CDN?
- A network of servers spread globally.
- Stores **static content** (images, videos, fonts, JS, CSS, some HTML).
- Users get content from the nearest CDN server, not the origin.

#### Major CDN Providers
- **Akamai** (largest)
- **Cloudflare**
- **Cloudfront** (Amazon)

#### What You Can Cache on CDN
- ✅ Images
- ✅ Videos
- ✅ Fonts
- ✅ JavaScript files
- ✅ CSS
- ✅ Static HTML snippets
- ❌ Anything that changes frequently (would serve stale data)

#### Cache Hit vs Cache Miss
- **Cache Hit:** Data is found in the cache → returned directly (fast)
- **Cache Miss:** Data is NOT in cache → must fetch from origin server (slow)

> **🔑 Mind-blowing stat:** By end of 2025, **70% of all internet traffic** is served by CDNs — meaning users hit application servers only 30% of the time!

#### CDN Cache Expiry
- Each cached item has an **expiry date** (e.g., images on Amazon may expire in 2045).
- After expiry, CDN re-fetches from origin and re-caches.

#### Live Demo Insight (Amazon)
When opened amazon.com and inspected Network tab:
- Images, fonts, JS, CSS → all show `CDN cache hit`
- The CDN server location was shown as `BLR50P4` → Bangalore-based CDN
- HTML doc → was a cache miss (dynamic content, fetched from origin)

#### Front-end Optimization Note
Earlier, the same image was sent to all devices and resized in browser. Now, **multiple versions** of the same image/video are stored, and the right one is sent based on device + network.

---

## 🖥️ Server-Side Caching (Where Caches Can Live)

When a request reaches the server side, we can place caches at multiple points:

```
Client → [Load Balancer / Reverse Proxy] → [App Server] → [Cache] → [Database (with internal buffer pool)]
              ↑ cache here              ↑ cache here    ↑ cache here    ↑ cache here
```

### 1. At the Load Balancer (Reverse Proxy)
- Some load balancers (like **Nginx, HAProxy**) can also act as a **reverse proxy** and cache API responses.
- If a request is very common and the response doesn't change → serve directly from LB.
- Called "reverse proxy" because LB acts as a **proxy of the actual server** to the user.

> **💡 Note:** Load balancer = redirects requests. Reverse proxy = caches responses. The same software can do both, OR you may use only one.

### 2. Inside the Database (Buffer Pool)
- Databases like **MySQL, PostgreSQL** have an internal **buffer pool**.
- They store responses of frequent queries.
- Even if you write a query in two different ways (with JOINs vs subqueries), the DB parses both into the same internal form.
- If that result already exists in buffer pool → returned instantly (no disk read).
- This is **automatic** — you don't program it.

### 3. Between App Server and Database
- A separate cache layer (e.g., **Redis, Aerospike** — in-memory key-value stores).
- Flow:
  1. App server checks cache first
  2. If found → return
  3. If not → query DB → store result in cache → return

#### Single Cache vs Distributed Cache (both are GLOBAL caches)
| Type | Pros | Cons |
|------|------|------|
| **Single Cache Server** | Lower latency (no internal LB) | Limited storage |
| **Distributed Cache** | Can store huge data | Higher latency (extra LB hop) |

### 4. Inside the Application Server (Local Cache)
- Use the **RAM** or **hard disk** of the same app server.
- ⚡ Lightning fast — **zero network hops!**

> **🔑 Important clarification:** A DP table or HashMap built inside a function is **NOT a cache**. The moment the function ends, that data is gone. A cache must **persist across requests**.

---

## 🌎 Global Cache vs Local Cache

### Global Cache
- Accessible by **all** application servers.
- Same behavior for every app server.
- Can be single-instance OR distributed.
- Example: Redis cluster sitting separately, all app servers query it.

### Local Cache
- Each application server has its **own** cache.
- Other app servers cannot access it.
- **Inherently distributed** (data is spread across servers).
- Implemented in the app server's RAM or disk.

> **🔑 Key insight from teacher:** Local cache only works in **stateful** architectures, where each app server is responsible for specific users (so it knows which data to cache).

### Hybrid Approach
- Each app server can have its own **dedicated cache server** (separate machine, e.g., its own Redis instance).
- ✅ Pros: Best of both worlds — low latency + large storage
- ❌ Cons: Cost (1000 app servers = 1000 cache servers)

> **💡 Practical reality:** Generally not done. Either go with global cache, or maintain cache inside the app server's RAM/disk.

---

## 🔒 Why Cache Should Be "Dumb"

The teacher made a strong point: **Cache should NEVER reach out to the database directly.**

### Why? (Cricket Analogy 🏏)
- In cricket, the **player** is smart — the bat, ball, pitch are dumb.
- If everything had its own brain, the system would be **chaos**.
- One central "brain" controls everything → predictable behavior.

### In Our Architecture
- The **application server** is the smart brain.
- It understands:
  - The complex schema (e.g., Reddit: questions → answers → comments → nested comments → votes)
  - Business logic
  - Where indexes exist
  - How to write the right SQL query
- If the cache also had to do this, you'd be duplicating business logic in two places.

> **🔑 Golden rule:** Most components in your architecture should be **dumb**. There should only be a few smart components. The application server is the brain. Cache, database, load balancers — all should be optimized at their own level but NOT take business decisions.

### What each component does:
- **App server:** Smart — filters, sorts, transforms, decides what to return
- **Database:** Dumb — executes the query and returns whatever's asked
- **Cache:** Dumb — stores key-value pairs, returns when key is hit

---

## 🔑 Cache Storage Format

- A cache is essentially a **hash map** — key-value pairs.
- The key can be:
  - A user ID
  - A question ID
  - A combination of multiple unique IDs/strings (very long composite keys)
- The value is whatever data the app server computed and wants to cache.

---

## ⚠️ Two Major Challenges with Caching

### 1. Stale Data Problem
- The **single source of truth** is the database — that's where writes happen.
- If you write to DB but cache still has the old value → cache serves **stale data**.
- **Solutions:**
  - On every DB write, also delete/update the cache entry.
  - But this is a **two-phase commit** → adds latency to writes.

#### When is stale data acceptable?
| Scenario | Need |
|----------|------|
| Bank balance, stock prices | **Strong / Immediate consistency** |
| Like counts, view counts, social feeds | **Eventual consistency** (5 min, 30 min, 1 hour delay is fine) |

> **🔑 Important point:** Cache will be discussed in detail under **Cache Invalidation** in the next class.

### 2. Storage Limitation
- Caches **cannot store everything** — limited memory.
- When full → need to **remove (evict)** something to make space for new data.
- This is solved by **Cache Eviction** policies (LRU, LFU, FIFO, etc.) — covered in next class and DSA.

---

## ❌ Caching is Useless / Harmful When...

- **Write-heavy systems** — cache adds extra latency to writes (you have to update cache too).
- For pure write workloads, cache is just an obstacle.
- ✅ Caches are great for **read-heavy** systems.

> **🔑 Real-world insight:** Even within the same system, some parts should be cached, others shouldn't. Decision is **case-by-case**, not blanket.

---

## 🎯 Summary: All Possible Cache Locations

| # | Location | Type | Notes |
|---|----------|------|-------|
| 1 | Browser (cookies, local/session storage, IndexedDB) | Client-side | DNS, preferences, tokens |
| 2 | CDN | Client-side | Static assets — biggest impact (70% of traffic) |
| 3 | Load Balancer / Reverse Proxy | Server-side | Common API responses |
| 4 | Inside Application Server (RAM / Hard Disk) | Server-side, Local | Lightning fast, no network hop |
| 5 | Separate cache server (Redis, Aerospike) — Single | Server-side, Global | Low latency, small data |
| 6 | Distributed cache cluster | Server-side, Global | High capacity, more latency |
| 7 | Database Buffer Pool (MySQL, Postgres) | Server-side | Built-in, automatic |

---

## 📌 Key Teacher Quotes & Principles to Remember

1. **"You should add a new server when existing ones are at 80–90% capacity, not when they're crashing."**
2. **"When adding a new server, data flow should be uni-directional — from existing servers to the new one only."**
3. **"Memory always and always works in layers."**
4. **"Most components should be dumb. Only a few should be smart."**
5. **"Cache is a hurdle for writing — it adds extra latency."**
6. **"For social media, even with uniform user distribution, load is NOT uniform — celebrities/influencers need vertical scaling."**
7. **"By end of 2025, 70% of internet traffic is served by CDNs."**
8. **"Speed of light is the upper limit — that's why CDNs put servers near users."**

---

## 🔮 What's Coming Next (Teaser)

In the **next class**, we will dive deep into the two challenges:

### A. Cache Invalidation
- How to ensure cached data is fresh
- Different invalidation strategies (write-through, write-back, write-around)
- Two-phase commits and their trade-offs

### B. Cache Eviction
- What to remove when cache is full
- Strategies: **LRU, LFU, FIFO, etc.**
- (Already covered partially in DSA)

After that → **case studies** where we design real systems using cache.

---

## 📚 Recommended Resources (Teacher's Suggestions)

- ✅ **Arpit Bhaiyani** — Excellent system design YouTube series
- ✅ **Tech blogs from Amazon, Netflix engineering teams**
- ✅ Notes provided after each class (read them, they have videos and extra material)
- ⚠️ **Avoid GeeksforGeeks** for deep understanding — articles aren't validated, and many have been found to be incorrect. OK for last-minute prep / exam revision (like GATE), but not for building solid concepts.

---

> **💡 Final Mindset:**
> Caching is fundamentally about **reducing latency** by avoiding repeated expensive work. The art is knowing **where** to cache, **what** to cache, and **how long** to cache it. Don't over-cache — sometimes it makes things worse.
