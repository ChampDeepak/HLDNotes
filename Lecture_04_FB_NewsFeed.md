# Case Study: Facebook NewsFeed — HLD Notes

## Table of Contents
1. [Recap of Previous Class](#1-recap-of-previous-class)
2. [Quizzes & Key Discussions](#2-quizzes--key-discussions)
3. [Five Steps of Implementing a Cache](#3-five-steps-of-implementing-a-cache)
4. [Facebook Case Study — Problem Statement](#4-facebook-case-study--problem-statement)
5. [Profile Page vs NewsFeed](#5-profile-page-vs-newsfeed)
6. [Back-of-the-Envelope Calculations](#6-back-of-the-envelope-calculations)
7. [Database Schema Design](#7-database-schema-design)
8. [Data Sharding Strategy](#8-data-sharding-strategy)
9. [Profile Page — Do We Need Caching?](#9-profile-page--do-we-need-caching)
10. [NewsFeed — Do We Need Caching?](#10-newsfeed--do-we-need-caching)
11. [Overall Architecture](#11-overall-architecture)
12. [Important Points Highlighted by Faculty](#12-important-points-highlighted-by-faculty)
13. [Questions to Ponder](#13-questions-to-ponder)
14. [Homework / Assignment](#14-homework--assignment)

---

## 1. Recap of Previous Class

### Scaler Code Judge Case Study
- **Problem:** Downloading GBs of test case input/output files caused high latency.
- **Solution:** Cache files locally on the application server's hard disk.
- **Invalidation Strategy:** Files made **immutable** — every test-case edit creates a *new file*; the SQL DB simply points to the new file name.
- **Consequence:** The old file becomes orphan data, occupying disk space.
- **Eviction Policy:** **LRU (Least Recently Used)** — when disk space runs out, the least recently used (typically outdated) file is purged.

> **Highlight:** Consistency is only a problem when data *changes*. Making data immutable removes consistency from the picture entirely.

### LeaderBoard Case Study
- **Problem:** Top-N leaderboard requires complex JOINs across 5–6 tables, GROUP BY, ORDER BY, LIMIT/OFFSET. This is a "good-to-have," not "must-have" feature, and would overload DB while it handles 50 submissions/second.
- **Solution:** Use a **global distributed cache** storing pre-computed leaderboard data.
- **Storage Format:** Key = `(contest_id, page_number)`, Value = JSON string with the entire page payload.
- **Benefit:** No SQL query at read-time; front-end parses JSON and renders.

---

## 2. Quizzes & Key Discussions

### Q: Why can caching *worsen* worst-case performance?
- **Best case:** Cache hit → response served instantly, no DB call.
- **Worst case:** Cache miss OR expired TTL → still must call SQL DB → **extra network hop** added on top of the DB query.
- > **Highlight:** Caching is not free — it adds complexity, cost, extra servers, code, load balancing, and network hops.

### Q: First and most important decision when implementing a cache?
- **Answer:** Do we even *need* a cache?
- Eviction algorithm (LRU) is generally already decided. The real question is **need** + **what data to cache**.
- Reach the decision via **back-of-the-envelope calculations**:
  - Make realistic assumptions, get order-of-magnitude numbers.
  - Numbers don't need to be precise — being off by 2–5× is fine; being off by 10× or 100× is not.

### Q: Why didn't we use a global cache for Scaler Code Judge?
- The maximum latency contributor is always the **network**, not disk/cache lookups.
- A global cache would still require downloading GB-sized files across the network → same network bottleneck.
- Local cache (on application server's hard disk) eliminates that network hop.

### Q: Does a CDN require a write-through cache?
- **No.** A CDN is just a cache; the invalidation algorithm depends on the use case (TTL, write-through, or write-around).
- ~13% answered incorrectly believing CDN ⇒ write-through.

### Q: Why not use CDN for Scaler Code Judge?
- **CDNs are meant for the client** (images, CSS, JS files served to browsers).
- The test-case files are served to **servers**, not clients → CDN is the wrong tool.

### Q: What is cache invalidation?
- Checking if cached data is still consistent with the source DB. If not, remove it from the cache.
- Needed only when data **changes**. If data is immutable, no invalidation needed.

### Q: Does TTL make data immutable?
- **No.** TTL works with any cache type. But TTL alone cannot give **immediate consistency**:
  - Short TTL → frequent refetches → defeats the purpose.
  - Long TTL → risk of serving stale data.
- For immediate consistency, use **write-through cache** or an **immutability-based design**.

### Q: Why not redirect Scaler requests by Problem ID (consistent hashing)?
- In a contest, ~100k users solve the same easy problem in the first 10–15 mins → all requests hit one server → **server collapse**.
- Hence: keep app servers **stateless**, use **round-robin** load balancing, distribute identical cached data across all servers.

> **Highlight:** Even though local caches store data, the servers are *stateless* because every server holds the same data.

---

## 3. Five Steps of Implementing a Cache

1. **Establish the need** — back-of-the-envelope calculations to justify caching.
2. **Type of cache** — local vs. global; single vs. distributed.
3. **Eviction algorithm** — usually LRU.
4. **Cache invalidation** — *the hardest step*; requires domain knowledge:
   - What's the consistency requirement?
   - How often does data change?
   - What's the schema and access pattern?
5. **Load balancing algorithm** — only relevant if going with a distributed cache.

> **Highlight:** In Scaler, we discussed load balancing *before* eviction because we first realized the servers must be stateless (so distributed local cache is required).

---

## 4. Facebook Case Study — Problem Statement

### What you must design
- The **Profile Page**
- The **NewsFeed Page**

### Simplifying assumptions for this design
- No likes, no comments, no replies.
- Only fetching/serving posts (no engagement features).
- NewsFeed shows only posts by users you're connected with (no recommendations from strangers yet).
- Each post is roughly text only (URLs optional); image/video are just URLs to S3.

### Given numbers
- Facebook MAU: ~**3 billion**
- Facebook DAU: ~**2 billion**
- Avg friends per user: ~**1000**
- Avg friends from whom posts actually appear in feed: ~**50–60** (based on engagement strength)
- Facebook uses ~**2000 parameters** in its recommendation/ranking model (treated as a black box for this design).

> **Highlight:** In an HLD interview, you'll never be asked directly to "design a cache." You're asked to serve data; whether caching is needed emerges from your design.

---

## 5. Profile Page vs NewsFeed

### Profile Page (e.g., Adam's profile)
- Shows: posts made **by** Adam (his statuses) + posts made **on Adam's profile by friends**.
- Ordered chronologically by creation date.
- Unique to Facebook: friends can post on someone else's profile (not possible in Instagram/LinkedIn).

### NewsFeed
- Shows: posts made by the user + posts made by all of their connections (in their own profiles).
- **Not strictly chronological** — sorted by a pre-computed **engagement score** (recency, shock factor, relationship strength, geolocation, hashtags, tagging, etc.).
- Goal: maximize "attention" — more time on feed → more ad impressions/clicks → revenue.

> **Highlight:** ~80% of social media users are passive consumers, ~20% engage, ~1% are content creators who post regularly.

---

## 6. Back-of-the-Envelope Calculations

### Pareto's Principle (extended for social media)
- ~99% of users **never** post — they just browse/consume.
- ~1% are active creators who may make ~4–5 posts/day.

### Posts per day
```
Active creators       = 1% × 2B DAU       = 20 million
Posts per creator/day = 5
Posts per day         = 20M × 5           ≈ 100 million posts/day
```

### Memory per post
- IDs (post ID, author ID, recipient ID): a few 8-byte numbers
- Text (max ~400–500 chars): ~500 bytes
- URLs: ~100 bytes
- **Round to: ~500 bytes (or 1 KB for easy math)**

### Total data over 20 years
```
Days in 20 yrs       ≈ 20 × 400 = 8000 days (round 365 → 400 for easy math)
Total posts          ≈ 100M × 8000 = 8 × 10^11 = 800 billion posts
Storage              ≈ 800B × 500 bytes = 4 × 10^14 bytes ≈ 400 TB
```

> **Highlight:** Pursue numbers that simplify arithmetic. Approximations within a factor of 2–5 are fine. Avoid being off by 10× or 100×.

### Implication
- 400 TB cannot fit in one machine (technically possible but absurdly expensive and SQL has indexing + compute overhead).
- **Therefore: the data MUST be sharded across multiple DB servers.**

---

## 7. Database Schema Design

### Users Table
| Column | Notes |
|--------|-------|
| user_id | PK |
| name | |
| email | |
| password_hash | |
| profile_pic_url | (stored in S3, only URL here) |
| ...other profile attributes | |

> Friends are **NOT** stored as a list inside the users table — SQL data must be normalized.

### Friends Table (User-to-User Relationships)
| Column | Notes |
|--------|-------|
| user1_id | indexed |
| user2_id | indexed |

- **Redundancy intentionally introduced:** every friendship stored **twice** (once with Adam→Deepesh, once with Deepesh→Adam).
- One-time write cost; reads (which dominate) become much faster — no need to query both columns.
- > **Highlight:** Tradeoff time vs space — use extra space to reduce latency.

### Posts Table
| Column | Notes |
|--------|-------|
| post_id | PK |
| creator_id | FK → users (who wrote the post) |
| recipient_id | FK → users (whose profile it lives on) |
| text | up to ~500 chars |
| image_url | optional |
| video_url | optional |
| created_at | |
| score | pre-computed by recommendation engine (used in newsfeed sorting) |

---

## 8. Data Sharding Strategy

### Sharding Key Choices Considered

| Strategy | Verdict |
|----------|---------|
| **By region/geolocation** | ❌ Uneven load (e.g., USA shard always hot, Pakistan shard idle). |
| **By Post ID (round-robin)** | ❌ Fetching a profile requires fan-out across ALL shards. |
| **By User ID using mod-hash** | ❌ The 1% active creators would all land on a few shards → hotspots. |
| **By User ID using Consistent Hashing** ✅ | Uniform distribution AND deterministic — given a user, you know exactly which shard holds their data. |

### Handling Posts that Belong to Two Users
- A post has both `creator_id` and `recipient_id` → could belong to 2 different shards.
- **Solution: Duplicate the post row in both shards.**
  - Adam writes on Deepesh's profile → row stored in Adam's shard AND Deepesh's shard.
  - This doubles storage (now ~800 TB) but eliminates cross-shard fan-out at read time.
  - More servers solve the storage cost — horizontal scaling is cheap.

> **Highlight:** Facebook's friendship rows and post rows are **both redundant by design**. Storage is cheap; cross-shard queries are not.

---

## 9. Profile Page — Do We Need Caching?

### Read path
1. Request hits app load balancer (round-robin) → app server.
2. App server hits DB load balancer.
3. DB load balancer computes `hash(user_id)` → binary search on hash ring → identifies target shard.
4. Query 1: fetch user details by `user_id` (indexed). 
5. Query 2: fetch posts by `recipient_id = user_id` (indexed), paginated.
6. Return data.

### Verdict: **❌ NO CACHE NEEDED for profile page**
- Single shard hit.
- Both queries use indexed fields.
- Paginated (only first 10 entries per request).
- This is **exactly what SQL was built for** — lightning fast.

---

## 10. NewsFeed — Do We Need Caching?

### Read path (without cache)
- User has ~1000 friends.
- Friends are sharded by `user_id` → friends are scattered across (potentially) all DB shards.
- To build the newsfeed: query EVERY shard to get recent posts from that user's friends → **fan-out request**.
- Then aggregate, sort by score, paginate.

### Why this is bad
- 2B DAU all browsing newsfeeds → fan-out requests to ALL DB shards → massive unnecessary load.
- Even **without** any "celebrity" user, the design is already broken.
- The "celebrity problem" amplifies this further — to be covered in the Twitter case study.

### Verdict: **✅ CACHE IS NEEDED for newsfeed**

> **Critical warning from faculty:** Don't design a cache that itself fans out to all cache shards. That just moves the problem from DB to Redis without solving it.

### What still needs to be decided (next class)
- Type of cache (local vs global; single vs distributed)
- What data exactly to cache
- How to structure the cached timeline data
- Invalidation algorithm
- Cache load balancing algorithm

---

## 11. Overall Architecture

```
Client
  │
  ▼
[App Load Balancer]   ── round-robin ──>  [Stateless App Servers]
                                                │
                                                ▼
                                      [DB Load Balancer]
                                                │  consistent-hash(user_id)
                                                ▼
                              [Sharded DB Servers]
                              ┌──────┬──────┬──────┐
                              │Shard1│Shard2│ ...N │
                              └──────┴──────┴──────┘
```

- App servers: **stateless** → simple **round-robin** load balancing.
- DB servers: **sharded by user_id** using **consistent hashing**.
- All tables (Users, Friends, Posts) exist in every shard; only the *rows* differ per shard.

---

## 12. Important Points Highlighted by Faculty

1. **Always justify cache with numbers**, not vibes. Phrases like "very frequent reads" must translate to actual estimates.
2. **Network calls dominate latency.** Cache vs DB lookup differences are negligible compared to a network hop.
3. **CDNs ≠ a specific invalidation type.** Invalidation depends on the use case, not the cache type.
4. **Immutable data eliminates the need for invalidation** entirely.
5. **TTL ≠ immediate consistency.** For strong consistency, use write-through or immutability.
6. **Keep servers stateless when possible** — round robin > consistent hashing if you can avoid state.
7. **Storage tradeoff:** redundant friendship + post rows = 2× storage but massive read latency win.
8. **In an HLD interview, you're never asked to design a cache directly** — caching emerges from your data-serving design.
9. **Avoid premature optimization** — design only for the problem given; don't add complexity for unspecified future needs.
10. **Order-of-magnitude correctness** is what matters in BoE calculations — being off by 2× is fine, being off by 10× is not.
11. **Pareto extended for social media:** 80% passive, 20% engaged, 1% creators.
12. **Sharding by region or by mod-hash on user_id both fail** — consistent hashing on user_id is the right answer.
13. **The fan-out problem is the heart of the newsfeed challenge** — and your cache design must NOT replicate that fan-out.

---

## 13. Questions to Ponder

These were posed by the faculty during class — try to answer them yourself:

1. Why can caching worsen worst-case performance?
2. What is the most important decision when designing a cache?
3. How do you decide if you need a cache? (What constitutes "frequent reads"?)
4. Why didn't Scaler Code Judge use a global cache?
5. What's the point of a global cache if it adds network hops?
6. Does a CDN require a write-through cache?
7. Why shouldn't a CDN be used for Scaler Code Judge?
8. What is cache invalidation? When is it needed?
9. Does TTL make data immutable?
10. Why shouldn't Scaler requests be routed by Problem ID?
11. What are the five steps of implementing a cache?
12. **Facebook-specific:**
    - What should be the database schema?
    - How many posts are generated per day on Facebook?
    - How much data does Facebook generate over 20 years?
    - Can this data fit on a single machine? If not, how do we shard it?
    - What is the right sharding key — region, post ID, user ID?
    - Why duplicate the post and friendship rows?
    - Do we need caching for the profile page? Why or why not?
    - Do we need caching for the newsfeed? Why or why not?
    - What type of cache, what data, what invalidation, what load balancing?

---

## 14. Homework / Assignment

The faculty deferred the final NewsFeed cache design to the **next class**, but left the following for students to work on:

1. **Calculate how much data needs to be cached** for the newsfeed.
2. **Decide the cache type** — local vs global, single vs distributed.
3. **Decide what exactly is cached** — full timeline per user? Posts per user? Something else?
4. **Decide the invalidation strategy** — TTL, write-through, write-around, hybrid?
5. **Decide the cache load-balancing algorithm.**
6. The faculty also mentioned a **practical lab assignment** to be released after 2–3 more classes, where students will implement the concepts learned.

> Bring your worked-out answers (with BoE calculations) to the next class for discussion.
