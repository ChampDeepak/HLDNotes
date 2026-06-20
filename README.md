# High-Level System Design — Unified Lecture Notes (Lec09–Lec16)

These notes synthesize the **lecture transcripts** with the matching **written notes** (local PDFs and web pages) into one explanation per lecture — built for *applying* the concepts to real system-design problems, not just reviewing terminology. Each file is self-contained but cross-links to the others where a topic spans lectures.

## The Arc
The eight lectures move from **building a database** → **search systems** → **messaging** → **architecture & delivery at scale**, all taught through one repeatable interview method:

> **understand the problem → functional requirements (MVP) → non-functional requirements → scale estimation → design** — then choose the right **database, sharding key, cache, and communication**.

## Lectures

| # | Lecture | One-line summary |
|---|---|---|
| 09 | [SQL Internals, Sharding for ACID, and Building an LSM-Tree Database](./Lec09.md) | Why SQL still wins (ACID), why ACID needs a single machine (so we shard), how SQL stores rows on disk (pages, WAL, B+ trees), and a build-your-own write-optimized **LSM key-value store**. |
| 10 | [Making the LSM Tree Read-Fast — Sparse Index, Bloom Filter, Tombstones](./Lec10.md) | Fixing LSM reads (sparse index → 1 disk seek/file, Bloom filter → skip absent files) and deletes (tombstones); the full read/write algorithm behind Cassandra. |
| 11 | [How to Attack a System-Design Problem — Typeahead, Scale Estimation, Async](./Lec11.md) | The HLD method (clarify → FR → NFR → scale), back-of-the-envelope estimation for Google Typeahead, microservices, and **sync vs async** + messaging queues. |
| 12 | [Designing Typeahead — Trie → Augmented Trie → Key-Value, with Batching & Sampling](./Lec12.md) | The Typeahead solution: trie pitfalls, precomputed top-k, pivoting to a key-value store for uniform sharding, and **batching/sampling** to tame a read+write-heavy system. |
| 13 | [Designing a Messaging App (Part 1) — Sharding, Consistency, Idempotency, Ordering](./Lec13.md) | WhatsApp-scale design: shard by `user_id`, the **receiver-shard-first** consistency illusion, **idempotency** via client `message_id`, distributed IDs (UUID/Snowflake), and message-chain ordering. |
| 14 | [Messaging App (Part 2) — Idempotent Payments, Cassandra, Local Caches, WebSockets](./Lec14.md) | Choosing a write-optimized **wide-column DB (Cassandra)**, a **local write-through cache**, **WebSockets/MQTT** for live delivery, payment idempotency, and IRCTC requirements (per-API consistency). |
| 15 | [Monolith vs Microservices, API Gateway, Inter-Service Communication — and IRCTC](./Lec15.md) | When a monolith breaks, what microservices fix (and their distributed tax), the **API gateway**, **REST → RPC → gRPC/protobuf**, and IRCTC as microservices (write-around caches, derived data). |
| 16 | [IRCTC Booking, Kafka & Messaging Queues, and Video Streaming](./Lec16.md) | Completing IRCTC booking (seat-overlap problem), deriving **Kafka** from first principles (log → topics/partitions/offsets), and **video streaming** (chunking, adaptive bitrate, HLS, CDNs). |

## Recurring Concepts (where they're introduced)
- **Sharding & sharding keys** — [Lec09](./Lec09.md) (5 properties, ACID-needs-one-machine), applied in [Lec12](./Lec12.md) (typeahead), [Lec13](./Lec13.md) (messaging), [Lec16](./Lec16.md) (IRCTC).
- **LSM tree / Cassandra** — [Lec09](./Lec09.md)–[Lec10](./Lec10.md) (built), [Lec14](./Lec14.md) (chosen for messaging).
- **Consistency (PACELC), caching, sync/async** — [Lec11](./Lec11.md) onward; **write-through vs write-around**, **TTL vs cron** in [Lec14](./Lec14.md)–[Lec16](./Lec16.md).
- **Idempotency & distributed IDs** — [Lec13](./Lec13.md)–[Lec14](./Lec14.md).
- **Microservices, API gateway, RPC/gRPC, Kafka** — [Lec11](./Lec11.md), [Lec15](./Lec15.md), [Lec16](./Lec16.md).
- **Scale estimation** (back-of-the-envelope, Pareto, peak load) — [Lec11](./Lec11.md), [Lec13](./Lec13.md), [Lec15](./Lec15.md)–[Lec16](./Lec16.md).

## How to Read
Each lecture follows the same template: **Overview → topic sections (intuition-first, with examples) → Try It Yourself (application exercises) → Homework / Next Lecture Preview**. Two callouts recur:
- 🔑 **Key Point** — something the faculty explicitly flagged as important.
- 🤔 **Think About It** — a question the faculty posed to build intuition.

**Diagrams** are authored as **Mermaid** (they render natively on GitHub and in most Markdown viewers). *Why not embed the source diagrams?* The written-notes PDFs are full-page document scans — each page is a single image with no separately-extractable diagram, and no image-cropping tooling was available in this environment — so embedding them as-is would have meant pasting whole document pages (tiny text included) into otherwise-clean notes. Per the task's diagram rule, clearer Mermaid reproductions were used instead. (If you'd prefer the original source pages embedded anyway, that's a quick follow-up.)

## Sources & Attribution
Synthesized from the lecture transcripts (Lec9–Lec16) plus the written notes mapped in `index.txt`:

| Topic | Source used | Lectures |
|---|---|---|
| SQL vs NoSQL + Sharding Key | local PDF (Progy Agarwal's notes) | [Lec09](./Lec09.md) |
| Types of NoSQL Databases | local PDF | [Lec09](./Lec09.md), referenced in [Lec14](./Lec14.md) |
| NoSQL Internals — LSM Tree | [hld-notes.vercel.app](https://hld-notes.vercel.app/%5BSST-2028%5D%20NoSQL%20Internals%20-%20LSM%20Tree.html) | [Lec10](./Lec10.md) |
| LSM Tree + Bloom Filter + Sparse Index | local PDF | [Lec10](./Lec10.md) |
| Case Study: Google Search Typeahead | [hld-notes.vercel.app](https://hld-notes.vercel.app/%5BSST-2028%5D%20Case%20Study_%20Google%20Search%20Typeahead.html) | [Lec11](./Lec11.md)–[Lec12](./Lec12.md) |
| Case Study: Typeahead - 2 | [hld-notes.vercel.app](https://hld-notes.vercel.app/%5BSST-2028%5D%20Case%20Study_%20Typeahead%20-2.html) | [Lec12](./Lec12.md) |
| Case Study: Messaging Apps (Messenger/WhatsApp/Slack) | local PDF | [Lec13](./Lec13.md)–[Lec14](./Lec14.md) |
| E-commerce, Microservices, IRCTC, Video Streaming | transcript only (no matching written note) | [Lec15](./Lec15.md)–[Lec16](./Lec16.md) |

> **Recommended/examinable readings flagged in class:** the **Cassandra paper** (Lakshman & Malik) — *mandatory/examinable* ([Lec14](./Lec14.md)); the **Kafka paper** — *strongly recommended, not examinable* ([Lec16](./Lec16.md)).
