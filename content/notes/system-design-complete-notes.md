---
layout: post
title:  "System Design Notes"
description: "How to approach system design"
categories: ["system-design"]
date: 2026-09-26 19:45:31 +0530
author: "Sai Kiran"
comments: false
---

# System Design Notes

A single consolidated reference covering methodology, estimation and capacity reasoning, when a single node stops being enough, how to protect the least-elastic component, wide-column stores and Bigtable, and three fully worked archetype problems.

**How the notes flow:** Part I sets the method. Part II turns requirements into numbers. Part III uses those numbers to decide when to scale out. Part IV covers protecting whatever component the scaling leaves as the new weak link. Part V covers wide-column datastores. Part VI applies the whole loop to three canonical problems. Part VII is a standalone deep dive on Google's Bigtable paper.

---

# Part I — Methodology

## 1. How to Approach System Design

System design starts with modeling a real-world domain. Clarifying requirements is central, and the process is genuinely iterative — asking questions to close gaps is part of the work.

**System design is *not* about finding all the cases.** Enumerating cases and recursing until none remain is closer to state-machine reasoning or test coverage. The case space is effectively unbounded, and trying to be complete is a trap. The actual skill is the opposite: deliberately **scoping down**, deciding what to leave out, and making **tradeoffs** under constraints that cannot all be satisfied at once. Knowing what *not* to build is a scored skill.

The heart of system design is **non-functional requirements (NFRs) and tradeoffs**, not completeness:

- Scale (users, read vs write, data volume)
- Latency vs throughput
- Consistency vs availability (CAP-style tension — you can't have everything)
- Cost, operational complexity, failure modes

### The Loop (better than "enumerate cases until none remain")

1. **Clarify requirements** (functional + non-functional)
2. **Estimate scale** (back-of-the-envelope capacity math)
3. **Define the API / data model**
4. **Sketch a high-level design**
5. **Stress-test** against bottlenecks and failure scenarios, and evolve

You stop **not** when "no cases remain" but when the design **defensibly meets the requirements you agreed on**.

### Classic tricky interview problems

1. **URL shortener (TinyURL)** — deceptively simple; trick is hashing/collision handling and read-heavy scaling.
2. **Rate limiter** — token bucket vs sliding window, distributed setting.
3. **News feed (Twitter/Instagram)** — fan-out-on-write vs fan-out-on-read; celebrity/hot-key problem.
4. **Chat system (WhatsApp/Messenger)** — real-time delivery, presence, ordering, receipts, offline queues.
5. **Distributed cache / key-value store** — consistent hashing, replication, eviction.
6. **Ride-sharing (Uber)** — geospatial indexing, matching, real-time location updates.
7. **Video streaming (YouTube/Netflix)** — CDNs, encoding pipelines, storage tiering.
8. **Notification system** — fan-out, retries, deduplication, multi-channel delivery.
9. **Web crawler / search indexing** — politeness, dedup, distributed frontier.
10. **Payment / ticket booking** — idempotency, concurrency, avoiding double-booking, exactly-once semantics.

What makes each "tricky" is usually **one specific tension** — hot keys, ordering guarantees, consistency under concurrency — not the number of cases. Most other classic problems are variations on three shapes: **read-scale, fan-out, or concurrency-correctness** (see Part VI).

## 2. Functional vs Non-Functional Requirements

### Functional requirements
Describe **what the system does** — features and behaviors. If you can phrase it as "the system shall let a user do X," it's functional.

*URL shortener example:* submit a long URL and get a short one; visiting the short URL redirects; URLs can optionally expire; user can see click analytics.

### Non-functional requirements (NFRs)
Describe **how well** the system does it — qualities and constraints. They don't add features; they constrain the design of features you already have. **These drive your architecture.**

Main categories:

- **Scale** — users, requests/sec, data volume, read/write ratio
- **Latency** — response time (p50, p99)
- **Availability** — uptime target (99.9% vs 99.99%), downtime tolerance
- **Consistency** — everyone sees same data immediately vs eventual consistency
- **Durability** — can committed data ever be lost
- **Reliability / fault tolerance** — behavior when a node/network/dependency fails
- **Security** — auth, encryption, rate limiting
- **Cost & operational complexity** — often unstated but always real

### Key insight
Functional requirements are usually the **easy, quick part** — nail them in the first two minutes and move on. The **NFRs are where the design lives.** "Build a URL shortener" is trivial functionally; what makes it a design problem is "handle 100M redirects/day at p99 < 50ms with high availability." That NFR set forces caching, replication, datastore choice, CDN placement, etc.

**Common failure mode:** spending too long enumerating features (functional) and never getting to the tradeoffs (non-functional).

**Nuance:** the boundary can blur. "Support two-factor auth" sounds functional (a feature) but is driven by a security NFR. Don't over-classify — the reason to separate them is to make sure you don't skip the NFRs.

---

# Part II — Estimation and Load

## 3. Back-of-the-Envelope Estimation

### The core mindset
Back-of-the-envelope isn't about precision — it's about getting the **order of magnitude** right so you can spot the real bottleneck (storage problem? QPS problem? bandwidth problem?). Round aggressively, use powers of 10, and always **state your assumptions out loud**. Sloppy estimation cascades into a wrongly-sized design, which is where interviews are won or lost.

You don't estimate for its own sake. You estimate to find **which dimension breaks first**, because that determines which architectural lever you pull. Numbers are useful only insofar as they cross a threshold that forces a decision. Estimation isn't a box to tick before design; it's the step that **tells you where the design has to bend.**

### The estimation chain
Everything flows in one direction — you derive each downstream quantity from users:

```
users → QPS → storage → bandwidth → memory/cache → servers
```

NFRs give you the targets (scale, latency, availability); the envelope math converts targets into numbers; the numbers reveal the first bottleneck; the bottleneck drives your first real design decision. Everything downstream (datastore, caching, sharding, replication) traces back to a number produced here.

### Inputs to clarify first (this itself is scored — it shows scoping discipline)
- **DAU** (daily active users), not total registered
- **Actions per user per day**, split into **reads and writes separately**
- **Payload size** per action/object
- **Retention period** (how long data lives)
- **Read:write ratio** — often the single most important number; most consumer systems are 10:1 to 1000:1 read-heavy

### Average QPS

`avg QPS = (DAU × actions per user per day) / 86,400`

**Mental trick:** there are ~100,000 seconds in a day (86,400 rounded up). So `DAU × actions / 10^5` gives QPS instantly in your head.

### Peak load
Traffic is never uniform, so:

`peak QPS = avg QPS × peak factor`

Two ways to reason about the peak factor:

1. **Diurnal concentration** — usage isn't spread over 24 hours; it clusters in a few active hours. If most of a day's traffic happens in ~6 hours instead of 24, that's already a ~4× concentration versus the naive average. This is the defensible way to *justify* a peak factor rather than pulling one from thin air.
2. **Bursts on top of that** — launches, viral events, cron jobs, retries during partial outages, cache stampedes. These stack on the diurnal peak.

**Interview default:** peak = **2–3× average** for a steady consumer app; explicitly note that spiky workloads (live events, flash sales, notification fan-outs) can hit **10× or more** and need separate handling (rate limiting, autoscaling, queue-based load leveling). State the factor *and why*, not just the value.

**Key subtlety:** you design *capacity* for peak, but size *cost/storage* for average. Conflating the two is a common mistake.

Always **separate read QPS from write QPS** — the read/write ratio changes everything (a 100:1 read-heavy system screams "cache + read replicas"; a write-heavy one pushes toward sharding, LSM-tree stores, and queues).

### Storage, bandwidth, cache
- **Storage** = writes/day × payload size × retention. Decides sharding and whether it fits on one machine. (Equivalently: bytes/record × records/day × retention → total, then project growth.)
- **Bandwidth** = QPS × payload. Do ingress (writes) and egress (reads) separately — egress usually dominates on read-heavy systems.
- **Cache/memory** = apply the 80/20 rule. Cache the hot ~20% of data that serves ~80% of reads; size the cache to that fraction of daily read volume.

### Numbers to have memorized

**Powers of 2 → storage:**

| Power | ≈ | Unit |
|---|---|---|
| 2^10 | 10^3 | KB |
| 2^20 | 10^6 | MB |
| 2^30 | 10^9 | GB |
| 2^40 | 10^12 | TB |
| 2^50 | 10^15 | PB |

**Availability ("N nines" → downtime):**
- 99.9% → ~8.8 hrs/year
- 99.99% → ~53 min/year
- 99.999% → ~5 min/year

**Latency anchors (order of magnitude, from Jeff Dean's classic list):**
- Main memory read ~100 ns
- Intra-datacenter round trip ~0.5 ms
- SSD read of 1 MB ~1 ms
- Disk seek ~10 ms
- Cross-continent round trip ~150 ms

These let you sanity-check whether a design can hit a latency SLA at all.

### Worked example (read-heavy feed)
Assume 100M DAU, 2 writes/user/day, 100 reads/user/day (50:1 read:write), 300-byte objects.

- **Write QPS** = 100M × 2 / 10^5 = **~2,000/s** average → peak (×3) **~6,000/s**
- **Read QPS** = 100M × 100 / 10^5 = **~100,000/s** average → peak (×3) **~300,000/s**

Immediately the shape is clear: writes are trivial, reads are the challenge → this is a caching and read-replica problem, not a write-throughput problem.

- **Storage** = 200M writes/day × 300 B = 60 GB/day → ~22 TB/year → needs sharding (and that's before media, which would dominate).
- **Read bandwidth** = 100k reads/s × 300 B ≈ 30 MB/s (×20 if a feed returns 20 objects/request → ~600 MB/s, a real constraint).

### The meta-skill and the "therefore"
After you compute, *interpret*. The numbers exist to point you at the one or two dimensions that actually stress the system, so you know what to design deeply and what to wave off.

Keep the math **fast and round** — powers of ten, clean numbers, no calculator. The interviewer checks whether you can reason from a scale number to an architectural implication. The skill is the "therefore":

> "That's ~100K write QPS at peak, which is well past a single Postgres primary, so I'll need to shard writes by user_id."

The precise figure matters far less than the therefore. Each result should trigger the question **"does any single number exceed what one machine / one component can handle?"** — the levers per dimension are listed in Section 6.

### References
- [Building Software Systems At Google and Lessons Learned — Jeff Dean (Stanford EE380, 2010)](https://www.youtube.com/watch?v=modXC5IWTJI) — Jeff Dean walks through how Google's infrastructure evolved (GFS, MapReduce, Bigtable, the indexing stack) on fleets of commodity, failure-prone machines. The part most relevant here is his "Numbers Everyone Should Know" — the order-of-magnitude latency table (L1/L2 cache, main memory, disk seek, datacenter and cross-continent round trips) reproduced in the *Latency anchors* subsection above. His core method is exactly back-of-the-envelope estimation: before building anything, do the rough math on QPS, storage, and latency to sanity-check whether a design can possibly meet its targets and to locate the dimension that will dominate. Also stresses designing for failure as the default at scale.
- [It's all a Numbers Game — the Dirty Little Secret of Scalable Systems — Martin Thompson (GOTO 2012)](https://www.youtube.com/watch?v=1KRYH75wgy4) — Thompson's argument is "mechanical sympathy": scalability and performance come from understanding the real numbers of the hardware (cache-line sizes, memory-access latency, disk and network throughput, context-switch cost) rather than from stacking abstraction layers. He shows how respecting those physical constants — keeping the working set in cache, minimizing contention and coordination, choosing data structures that match how the machine actually works — often lets a single well-designed node do what people assume needs a cluster. The takeaway that reinforces this section: measure and reason from concrete numbers, because guessing about where the bottleneck lives is how designs go wrong.

## 4. Average vs Peak Load (Deeper)

### Mean vs max of a distribution
Picture the real 24-hour traffic curve:
- **Average QPS** = area under the curve ÷ 24h.
- **Peak QPS** = the height of the curve at its tallest point.

The average is a fiction — no real system experiences it, because traffic is never flat. You **provision for the peak (the max)** but **size storage against the average (the area)**.

The peak factor decomposes into two independent things (separating them makes the estimate defensible):
- **Diurnal concentration** — the day's traffic clusters into active hours. ~80% in ~6 of 24 hours is a ~3–4× concentration over the naive average. Arithmetic you can justify out loud.
- **Bursts on top** — minute-to-minute variance within the peak window, plus event-driven spikes (launches, notification fan-outs, cron, retry storms). Stack multiplicatively on the diurnal peak.

### Most-missed subtlety — peak *duration* matters as much as peak *height*
A 10× spike lasting 5 seconds and a 10× load lasting 3 hours demand completely different responses:
- A **short burst** can be absorbed by a buffer — a queue soaks it up and the DB drains it at its own rate.
- A **sustained peak** cannot be buffered away; the queue grows without bound. You need real provisioned capacity or load-shedding.

So when you name a peak factor, also say how long it lasts — that decides whether the answer is "add a queue" or "add nodes."

## 5. The Operations Beyond QPS and Storage

QPS and storage are necessary but not sufficient. The ones that actually change designs:

- **Read:write ratio** — decides replicas-and-cache (read-heavy) vs sharding (write-heavy). The most consequential single number.
- **Fan-out** — one logical request → N physical ops. A feed read returning 20 items is 20× the object reads; a write fanning out to a celebrity's 10M followers is a write-amplification bomb. Silently multiplies every other number.
- **Working set size** — fits in RAM or not; determines memory-bound vs disk-bound.
- **Write amplification** — one logical write becomes many physical writes: index updates, WAL, LSM compaction, replication factor. A "1,000 writes/s" workload can be 5,000+ physical writes at the disk.
- **Replication factor** — total storage = data × RF; also multiplies write cost. RF=3 means 3× storage and write work.
- **IOPS / disk bandwidth** — the physical ceiling underneath write throughput; often the real limit, not CPU.
- **Network bandwidth** — bytes/s = QPS × payload (split ingress/egress; egress dominates read-heavy).
- **Cache hit ratio** — determines how much load actually *reaches* the DB. An 80% hit ratio means the DB sees 20% of read QPS — can be the difference between one node and ten.
- **Growth rate + retention** — storage isn't static; size for the dataset at your planning horizon, not today.

### The workflow that ties it all together
1. Compute **average QPS**.
2. Apply a **justified peak factor** (diurnal + burst, *with duration*).
3. **Split reads/writes**.
4. Apply **fan-out** and **write amplification**.
5. Check each against the **single-node ceilings** (Section 6).
6. The **first dimension that breaks** tells you *which* scaling lever to pull (cache, replica, queue, shard).
7. **Protect whatever component that lever leaves as the new weakest link** (Part IV).

---

# Part III — Single-Node Ceilings and When to Scale Out

## 6. What Fits on One Machine

The strongest interview move is often *not* reaching for a distributed store. A single beefy box in 2026 is shockingly capable, and it comes with transactions, joins, and zero distributed-systems tax (no consensus, no eventual consistency, no rebalancing). A single node suffices until **any one** of these dimensions exceeds what one box provides. They **break independently**, and the *first* to break determines your scaling strategy.

### The dimension-by-dimension ceilings (order of magnitude)

| Dimension | Comfortable single-node ceiling (order of magnitude) | What breaks it | Lever when it breaks |
|---|---|---|---|
| **Storage** | a few TB comfortably, stretch to tens of TB | Total dataset outgrows the box | Sharding / distributed store (or archive cold data) |
| **RAM / working set** | tens to low-hundreds of GB (high-mem boxes reach ~1 TB+) | Hot set + indexes stop fitting in RAM → hits disk → latency cliff | More RAM → caching strategy & sizing → sharding |
| **Write throughput** | ~thousands–10k writes/sec (relational primary); ~50k–100k+ for a tuned LSM engine | Sustained writes exceed what one commit log/disk can absorb | Sharding, or a different write path |
| **Read throughput** | ~100k+ ops/sec (Redis in RAM); ~tens of thousands (SQL, more with replicas) | Reads exceed one node at acceptable latency | Read replicas + cache + CDN (only shard if writes *or* data also break) |
| **IOPS** | local NVMe ~hundreds of thousands to ~1M; cloud network volumes far lower unless provisioned | Random-IO-heavy workload on network-attached storage | Faster/provisioned storage, then scale out |
| **Connections** | ~few hundred direct (Postgres = process per conn); thousands via a pooler | Connection *count*, not query load | Pooler (a PgBouncer / Little's-Law problem, **not** a sharding one) |
| **Network** | one NIC ≈ 10 Gbps (~1.25 GB/s) | Large payloads (video, big blobs) at high QPS | CDN / edge, more NICs, scale out |
| **Latency at that QPS** | — | A node hits target QPS but blows the p99 SLA | Reduce per-query cost, denormalize, in-memory paths, geo-distribution, cache, or scale out |

### The asymmetry (why the write ceiling matters most)
Reads break "gently" — replicas and cache are cheap and well-understood. Writes break "hard" — sharding, with all its cross-shard pain. **Writes can't be scaled by adding read replicas**, so the write ceiling of a single primary is usually **the** number that forces sharding, and the one worth computing most carefully.

### Concrete anchors (caveat heavily — order of magnitude)
These vary enormously with hardware, schema, and especially query complexity (a point lookup vs a 5-table join with aggregation differ by 2–3 orders of magnitude). Napkin anchors, not guarantees. The real number is the one you measure under your workload.

**Single Postgres node (good NVMe hardware):**

| Dimension | Anchor | Notes |
|---|---|---|
| Direct connections | ~hundreds (100–500) comfortable | process-per-connection, ~5–10 MB each; pooler → thousands of clients |
| Write throughput | ~10^4/s order (roughly 10k–50k simple writes/s) | bottleneck is usually WAL fsync + disk; batching helps a lot |
| Read throughput (point) | 10s of thousands to ~100k+/s if working set in RAM | collapses if disk-bound; multiply by adding replicas |
| Storage (practical) | comfortably several TB; operational pain (vacuum, backup, restore, index rebuild) mounts into tens of TB | the limit is usually operational, not raw capacity |
| Working set | whatever fits in RAM (128 GB–1 TB+ boxes common) | the single biggest performance determinant |

**Distributed store model:** aggregate ceiling ≈ *per-node ceiling × node count × efficiency*, where efficiency is sub-linear because coordination costs grow. A Cassandra node does ~10k–50k writes/s (LSM / write-optimized); clusters scale roughly linearly into millions of ops/s. DynamoDB is effectively unbounded in aggregate but gated by per-partition throughput and hot keys. Mental model: aggregate ≈ per-node × node count, minus coordination tax, with a *per-shard* hot-spot limit sitting underneath.

### The synthesis rule
**If peak QPS is in the low thousands and data is in the low TB, one primary + a read replica + a cache handles it — no sharding.**

You start seriously justifying a distributed/sharded store when:
- *sustained writes* cross roughly tens of thousands per second, **or**
- the *working set* dwarfs RAM, **or**
- the *dataset* exceeds what one box comfortably holds.

### Two nuances that separate a good answer from a great one
1. **The connections row is a trap.** Hitting a connection ceiling looks like "I need to scale out," but the fix is a pooler (PgBouncer) multiplexing thousands of clients onto a small pool — you're moving the bottleneck, not sharding the data. Don't conflate a connection problem with a throughput problem.
2. **"Reads" almost never justify sharding alone.** Reads scale with replicas and caching, both far cheaper than sharding. It's **writes and storage** that force the jump, because those are the two things replicas can't help with.

### The disciplined answer to "does this need to be distributed?"
Estimate peak write QPS and total storage first, check them against those two ceilings, and only invoke sharding if one genuinely breaks. **Reaching for Cassandra when a single Postgres primary would do is the more common interview mistake than the reverse.**

### Availability caveat
Even if one node *could* handle the load, availability requirements mean you never run one. HA demands replicas regardless. **"One node handles the load" ≠ "deploy one node."**

## 7. Sharding: Mechanism and Partitioning

### Shard-native vs shard-as-afterthought
Wide-column stores aren't *always* running sharded (you can run a single-node Cassandra just fine) — they're **partition-native**: the partition key is a first-class part of the data model, not something bolted on later.

- In Cassandra the row key *is* the partition key.
- In Bigtable/HBase the table is auto-split into tablets by row range.

The architecture assumes horizontal partitioning as the primary axis of scale, so sharding is automatic and cheap rather than a painful migration. Contrast with Postgres/MySQL, where the schema has no notion of a partition key and sharding is a bolt-on you dread. That's the real distinction: **shard-native by design vs. shard-as-afterthought.**

Two flavors worth naming:
- **Range partitioning** (Bigtable tablets) — good for range scans, risks hotspots.
- **Hash partitioning** (Cassandra consistent hashing) — even spread, no cross-partition range scans.

### Why sharding is critical for write throughput — the mechanism
A single node has one commit log, one set of CPUs, one disk subsystem, so all writes serialize through it. Sharding makes writes to different partitions **independent**, so aggregate write throughput scales roughly linearly with node count.

A single-primary relational DB is the opposite — every write funnels through one primary; replicas only help reads. **That single-primary write ceiling is *the* reason wide-column stores exist.**

## 8. Distributed Databases Have Limits Too — The Weak Component Just Moves

Distributing doesn't remove the ceiling; it raises it and shards it. Two things stay true:

1. **Each node still has a local limit.** A Cassandra or CockroachDB node has the same per-node connection, CPU, memory, and IOPS ceilings as any single box. Spreading load across N nodes multiplies the aggregate, but every individual node is still a "weak component" locally — many small weak components instead of one big one.

2. **The bottleneck often changes form.** In a sharded system the weak points become the **hot partition/shard** (one key getting disproportionate traffic), the **coordinator node**, **cross-shard transactions**, and **the slowest replica in a quorum**. Managed stores abstract connections away entirely — DynamoDB has no persistent-connection model; its limit is throughput per partition (historically ~1,000 write units / 3,000 read units per partition), so a hot key hits a *partition-throughput* wall, not a connection wall. Same principle, different name.

**Generalized rule:** always protect the least-elastic component — but first identify what it actually is. Single Postgres → connections; sharded store → usually the hot shard; system fronting a third-party API → that API's rate limit. The instinct is right; the target shifts.

---

# Part IV — Protecting the Weak (Least-Elastic) Component

## 9. Autoscaling vs Non-Autoscalable Tiers

### Why the problem exists
Stateless app pods scale horizontally almost for free. A relational database is **stateful**, and stateful components resist horizontal scaling because data must live somewhere consistent. When the autoscaler spins pods 10 → 200 under peak, each pod opens its own connection pool, and you multiply straight into the DB's connection ceiling.

**The brutal math:** 200 pods × 20-connection pool = 4,000 connections. A single Postgres instance is comfortable at a few hundred. Each connection isn't free — in Postgres it's a whole backend process with its own memory (~several MB) plus scheduling overhead. The DB falls over not because it can't do the *work*, but because it's drowning in *connections*. The scaling that saved the app tier kills the data tier.

### The connection-limit approach — necessary but not sufficient
Capping max DB connections and bounding autoscaling is the right instinct (protecting the component that can't protect itself), but alone it's blunt: once you hit the cap, new pods can't get connections or you refuse to scale, turning a DB limit into an app-tier availability problem. It's a **backstop, not a strategy**.

Better framing: **decouple "number of app instances" from "number of DB connections."** They should not be forced to grow together.

### The main tools (roughly in order of reach)

**Connection pooling middleware (reach for this first).** Put a pooler between the app tier and the DB — PgBouncer or pgcat (Postgres), ProxySQL (MySQL), or RDS Proxy (AWS). The 200 pods connect to the pooler; the pooler maintains a small, stable set of real connections to the DB (say 100), multiplexing thousands of client connections onto them. In *transaction* pooling mode a real connection is only held for a transaction's duration, so utilization is very high. Highest-leverage fix; directly breaks the pods↔connections coupling.

**Right-size the per-pod pool.** A common mistake is a fat pool per pod "just in case." Counterintuitive result (HikariCP): throughput often *increases* as you shrink the pool, because a smaller pool means less lock contention and context-switching inside the DB. Fewer connections doing more work beats many connections thrashing.

**Read/write splitting + read replicas.** Reads scale horizontally via replicas in a way writes can't. Send reads to replicas, keep writes on the primary. Offloads most connection/query pressure, at the cost of replication lag (eventual consistency on reads — call it out explicitly).

**Caching in front of the DB.** Every read served by Redis/Caffeine is a connection the DB never sees. Cache the hot 20% and a large fraction of read traffic never touches the data tier. The cheapest connection is the one you don't open.

**Queue-based load leveling for writes.** For write bursts (and anything not needing to be synchronous), put a queue between app and DB. The DB consumes at its own sustainable rate instead of being slammed at peak QPS. Converts spiky load into smooth load. Trade-off: those writes become asynchronous — the client gets an ack, not a committed result.

**Then, structural scaling of the data tier itself:**
- **Vertical scaling** — bigger DB box. Simplest, buys time, has a ceiling, doesn't help availability.
- **Sharding / partitioning** — split data across instances by key. The real horizontal-scale answer for writes, but a big commitment: cross-shard queries, rebalancing, hot-shard problems, distributed-transaction pain. Reach for it when a single primary genuinely can't hold the write volume, not before.

### The general principle
Any non-autoscalable or slower-scaling component behind an elastic tier needs a protection mechanism, because the elastic tier will find its limit for you. Three recurring patterns:

1. **A decoupling layer** so the fast tier's instance count doesn't map 1:1 to load on the slow tier (connection pooler, cache).
2. **A buffer** that absorbs bursts and lets the slow component consume at its own rate (queue).
3. **Backpressure / admission control** so that when the slow component is saturated, you shed or slow load *gracefully* at the edge rather than letting it collapse (rate limiting, circuit breakers, bounded pools that fail fast).

Applies to third-party APIs with rate limits, legacy mainframes, payment gateways, single-writer stores — anything downstream that can't match the app tier's elasticity. The connection cap is really pattern #3 applied to a database; a complete answer pairs it with #1 and #2 so the cap rarely gets hit.

**Interview phrasing that scores:** "The app tier is elastic but the database isn't, so I need to decouple their scaling — pooler and cache to reduce connection demand, a queue to level write bursts, replicas for read scale, and a hard connection cap as the safety backstop." Shows you see the *class* of problem, not just the instance.

## 10. Connection Pooling: What Are the Limits?

A pooler doesn't remove the limit — it **moves** it, from "how many connections can the DB tolerate" to "how much concurrent work can the DB actually do." That second limit is the real, harder one.

### The pooler doesn't manufacture capacity
Multiplexing 5,000 client connections onto 100 DB connections works only because at any instant most clients are idle between requests. The pooler exploits that idleness. But the 100 real connections are still a hard ceiling on **simultaneous in-flight queries**. If the workload genuinely needs 500 queries executing at the same moment, no pooling mode saves you — clients queue at the pooler and you've converted a connection error into latency. The ceiling isn't "connections," it's **peak concurrent query demand vs. DB-side pool size**.

### How to size the DB-side pool — Little's Law

`connections needed = throughput (QPS) × avg query time (s)`

Example: 2,000 queries/s, each holding a connection for 5 ms → 2,000 × 0.005 = **10 connections** to keep up. The *right* pool is usually small. Set the DB-side pool a bit above this for headroom, not at some big round number.

**Corollary:** pool size should be bounded by what the DB hardware can usefully run in parallel (~core count and disk concurrency). Past that point, more connections don't add throughput — they add context-switching, lock contention, and cache thrashing, and throughput *drops*. The upper limit is set by the database's physical parallelism; a good pool size respects it.

### Pooling mode is itself a limit

**Session pooling** — a real connection is pinned to a client for its whole session. Safe, fully compatible, but barely better than no pooling for the multiplication problem, because idle sessions still hold real connections.

**Transaction pooling** — the real connection is held only for a transaction's duration, then returned. Source of the big multiplication ratio. But it breaks anything relying on state living *across* transactions on the same physical connection: session-level `SET` variables, `WITH HOLD` cursors, `LISTEN/NOTIFY`, advisory locks held across statements, and server-side **prepared statements** (PgBouncer transaction mode historically didn't support protocol-level prepared statements well — improved recently, still a config concern). The limit: transaction pooling costs you per-session server-side features.

**Statement pooling** — most aggressive, returns the connection after every statement. Forbids multi-statement transactions entirely. Niche.

### The pooler is its own component with its own ceilings
- **Single-threaded designs** — classic PgBouncer is single-threaded, bounded by one CPU core for connection-handling work. Very high client counts can saturate that core, making the pooler the bottleneck. Fix: multiple pooler instances (with `SO_REUSEPORT`) or a multi-threaded pooler like pgcat — but then multiple poolers each have their own DB-side pool, and their pools **sum** against the DB's limit, so you must divide the budget.
- **File descriptors / memory** — every client connection is still a socket and some memory on the pooler box. Thousands is fine; hundreds of thousands needs tuning.
- **It's a new hop** — added latency (usually sub-ms, but real) and a new failure point. Wants HA itself, which brings back the "how do the standby pooler's connections count against the DB budget" question.

### The limits, as a checklist
The pooler helps up to the point where one of these binds:
- **DB-side pool ≈ QPS × query time** — if real concurrency exceeds the pool, clients queue (latency, not errors).
- **DB physical parallelism** — pool larger than ~cores + effective disk concurrency stops helping and starts hurting.
- **Feature compatibility** — transaction/statement modes forbid cross-transaction session state; caps how aggressively you can multiplex a given app.
- **Pooler's own CPU/FD/memory** — one instance has a throughput ceiling; scaling it out splits the DB budget.
- **Total across all poolers ≤ DB max connections** — the sum still has to fit under the database's hard limit.

When you're pushing all of these at once, pooling has done its job; the next lever is structural — read replicas (move read concurrency off the primary), caching (cut query volume), or sharding (multiply the write-side ceiling). **The pooler buys a large constant factor; it doesn't change the asymptote.**

---

# Part V — Wide-Column Stores

## 11. What "Wide-Column" Means in a Database

A wide-column store is a NoSQL database model that organizes data into rows and columns, but unlike a relational database, the columns aren't fixed across all rows. Each row can have its own set of columns, and different rows in the same table can hold wildly different columns.

### The mental model: a nested map, not a grid
Think of it as a **two-level (or nested) map** rather than a rigid grid:

```
Row Key → Column Family → { column : value, column : value, ... }
```

A lookup is essentially:

```
map[rowKey][columnFamily][columnName] = value
```

- Each row is identified by a **row key**.
- Columns are grouped into **column families** (defined up front).
- Within a family you can have thousands or millions of **dynamic columns** that vary row to row.

### Relational vs. wide-column

| Relational | Wide-column |
|---|---|
| Fixed schema — every row has the same columns | Columns are dynamic per row |
| Rows are the unit of storage | Data physically stored/grouped by column family |
| Sparse data wastes space (lots of NULLs) | Sparse data costs nothing — absent columns just don't exist |
| Joins, ACID | Denormalized, query-driven modeling |

### Concrete example
Storing user activity where every user tracks different metrics:

```
row: user_123
  profile: { name: "Sai", city: "..." }
  activity: { login_2026_01_01: "...", login_2026_01_02: "...", ... millions of these }

row: user_456
  profile: { name: "...", country: "..." }
  activity: { purchase_x: "...", purchase_y: "..." }
```

Neither row is forced to have the other's columns.

### Why it exists / when to reach for it
Wide-column stores are built for **massive write throughput and horizontal scaling**. The row key determines partitioning, so writes and reads for a given key hit one partition — very fast, and it scales linearly by adding nodes. The tradeoff is you design your tables around your queries (**query-first modeling**), since there are no joins and limited ad-hoc querying.

**Canonical implementations:** Apache Cassandra, HBase (both descended from Google's Bigtable paper), and ScyllaDB.

**Good fit for:** write-heavy, high-scale problems — time-series data, event logging, messaging/chat history, IoT sensor data — anywhere you have a huge volume of writes, a natural partition key, and access patterns you can define ahead of time.

### Important clarification: "wide-column" ≠ "columnar"
- **Columnar** (column-oriented analytical stores like Redshift, ClickHouse, Parquet) optimize OLAP scans by storing each column contiguously on disk for compression and aggregation.
- **Wide-column** stores are operational (OLTP-ish) key-based stores with flexible schemas.

The names sound alike but they solve different problems — a common point of confusion.

# Part VI — Worked Problems

Before drawing any boxes, remember the ordering that scores points: **API design comes first** (define the contract; every endpoint traces to a functional requirement, its performance profile to an NFR; watch for idempotency on writes). **Then the data model** (design around access patterns, not abstract normalization; the read/write ratio and consistency NFR usually settle SQL vs NoSQL). **Then the high-level design**, where every box exists because a number or an NFR demanded it — and you narrate the justifying constraint out loud. Keep the app tier **stateless** (state belongs in the datastore or cache); a server holding local session state is the most common self-inflicted wound.

## 12. URL Shortener

### 12.1 Requirements
**Functional:** submit long URL → get short URL; short URL redirects; optional custom aliases, expiration, click analytics.

**Non-functional:**
- **Read-heavy** — redirects vastly outnumber creations (~100:1)
- **Low latency** on redirect (user-facing path; p99 in tens of ms)
- **High availability** — a down shortener breaks every shared link; availability > strong consistency
- **Durability** — a created mapping must never be lost
- Short codes short and non-guessable-ish

*Consistency call:* slightly stale metadata on redirect is fine → "availability over consistency."

### 12.2 Estimation
Assume 100M new URLs/day.
- **Write QPS:** 100M / 86,400 ≈ ~1,200/sec avg, ~3,000 peak
- **Read QPS:** at 100:1 → ~120K/sec avg, ~300K peak
- **Storage:** ~500 bytes/record × 100M/day ≈ 50 GB/day → over 5 years ≈ **~90 TB**
- **Cache sizing:** 80/20 rule — hot ~20% of a day's reads is a few GB → fits in RAM easily

**Therefore:**
- 300K peak reads/sec cannot come from a disk-backed DB alone → **cache is mandatory**
- 90 TB exceeds one machine → datastore must **shard**
- 3K writes/sec is modest → write path is not the pressure point; **read path and storage volume are**

### 12.3 Core design question: generating the short code
This is the **one specific tension**. Three approaches:

**(a) Hash the long URL** (MD5/SHA, take first N chars). Simple, same URL → same code. But **collisions** require detection (rehash with salt or extend length), adding a read per write. Fiddly.

**(b) Counter + Base62 encoding.** Global incrementing counter, encode integer in Base62 (`[a-zA-Z0-9]`). No collisions. 62^7 ≈ 3.5 trillion codes at 7 chars. Problem: a **single global counter is a bottleneck and SPOF**. Fix with **range handout** — ZooKeeper or a ticket server gives each app instance a block (e.g., 1,000 IDs); each burns through its block locally, coordinating only when it needs a new block. **Cleanest answer; lead with this.**

**(c) Pre-generated key set / KGS.** Generate unique codes offline, store in an "available keys" table, hand out on demand. Removes generation from the request path. Adds machinery (never hand the same key twice, careful restart handling). Good alternative to mention.

**Signal:** name the tradeoff — *hashing trades coordination for collision handling; counters trade collision handling for coordination.* Range-based counter gets the best of both.

### 12.4 API and data model
**API:**
- `POST /urls {longUrl, customAlias?, expiry?} → {shortUrl}` — rare write
- `GET /{shortCode} → 301 redirect` — 300K/sec hot path

**301 vs 302 tradeoff:** 301 (permanent) lets browsers/intermediaries cache the redirect (slashes read load) but **loses click analytics** (cached redirects never hit your server). 302 (temporary) forces every click through you, preserving analytics at the cost of load. A direct functional-vs-NFR tradeoff — state it out loud.

**Idempotency on write:** retried `POST /urls` shouldn't create two codes. Hashing naturally maps a retry to the same code; with a counter, dedupe via a request-id or lookup-by-long-URL.

**Data model** — point lookup by key:
```
shortCode (PK) | longUrl | createdAt | expiry | ownerId | ...
```
Point-lookup + massive scale + eventual consistency OK → **key-value / wide-column** (DynamoDB, Cassandra), sharded by `shortCode`.

### 12.5 High-level design
- Client → **load balancer** → stateless **app tier**
- App tier → **ID/counter service** (range-based) on writes only
- **Cache (Redis)** on read path — absorbs 300K/sec. Redirect: check cache → hit → redirect; miss → read datastore, populate cache, redirect
- **Datastore** — sharded KV store, durable mapping
- **CDN / edge** — push hot mappings to the edge, cut latency
- **Async queue** for analytics — click events off the request path

Each box traces to a number: cache/CDN → 300K reads; sharding → 90 TB; queue → clean hot path; ID service → code-generation tension.

### 12.6 Bottleneck and failure analysis
- **Cache is now critical infrastructure** — if it dies, 300K/sec slams the DB. Need cache replication/clustering. Moving the read bottleneck into the cache didn't eliminate it; it relocated it.
- **Hot key** — a viral link concentrates traffic on one cache node/shard. Mitigate via CDN/edge (popularity makes it cache-friendly) and, if needed, replicate hot keys.
- **ID service failure** — range-based: a crashed instance just grabs a new block; the unused tail is wasted (fine — trillions of codes). No correctness impact.
- **Read-your-writes** — brief window where a new code isn't readable everywhere under eventual consistency. Usually acceptable; if not, route immediate post-create reads to the primary.

## 13. News Feed

The tension here is **fan-out**.

### 13.1 Requirements
**Functional:** post content; see a feed of posts from followees, newest-ish first; optional likes/comments, ranking, media.

**Non-functional:**
- **Massively read-heavy** — people scroll far more than they post
- **Low feed-load latency** — retention-critical path
- **High availability** — a stale feed is fine; a blank feed is not. Availability + eventual consistency > strong consistency
- **Scale asymmetry across users** — most have hundreds of followers; a few have tens of millions. *This is the whole ballgame.*

*Consistency call:* nobody notices if a post takes a few seconds to appear in followers' feeds → async fan-out is legal.

### 13.2 Estimation
Assume 500M DAU, ~2 posts/day each, ~10 feed reads/day each.
- **Writes (posts):** 1B/day ≈ ~12K/sec avg, ~30K peak
- **Reads (feed loads):** 5B/day ≈ ~60K/sec avg, ~150K peak
- Each feed load surfaces ~200 posts

**Therefore:** you cannot assemble a feed on-read by querying every followee and merging at 150K/sec (fan-in of thousands of queries per load). The question is *when* you do the expensive merge. That timing choice is the entire design.

### 13.3 Core tension: fan-out on write vs fan-out on read
**Fan-out on write (push):** on post, push the post ID into a precomputed per-user feed cache (e.g., Redis) for *every follower*.
- Reads are trivial and blazing fast (read your precomputed list)
- Cost paid at write time, *per follower*. 200 followers → 200 writes. Fine.
- **Catastrophe:** a celebrity with 50M followers → **50M writes per post** (hot-key / thundering herd). Push does not survive celebrities.

**Fan-out on read (pull):** store posts once; on feed load, fetch recent posts from all followees and merge on the fly.
- Writes cheap (one write). Celebrities a non-problem.
- Reads expensive — fan-in merge every load. At 150K loads/sec, crushes you for normal users.

**Insight:** neither pure approach works because the follower distribution is **bimodal**. Push is cheap for normal posters, lethal for celebrities; pull is the reverse.

**Answer — hybrid:**
- **Push** for normal users — precompute followers' feeds
- **Pull** for celebrity/high-fan-out accounts — do *not* fan out; at read time, merge their recent posts into the reader's precomputed feed

A feed load = "read my precomputed list (covers normal followees) + pull-and-merge posts from the few mega-accounts I follow." Naming the hybrid *and why* (bimodal distribution) is what's scored.

### 13.4 API and data model
**API:**
- `POST /posts {content, media?} → {postId}` — triggers fan-out (or not, for celebrities)
- `GET /feed?cursor=... → [posts]` — **cursor-based pagination** (offset breaks on a growing, reordering list)

**Data model** — multiple stores by access pattern:
- **Posts** — `postId (PK) | authorId | content | createdAt`. Write-once, read by ID. KV/wide-column.
- **Social graph** — follower/followee edges. Read constantly (fan-out: "who follows X?"; pull-merge: "who does Y follow?"). Often its own service.
- **Feed cache** — per-user precomputed list of **post IDs** in Redis. Store IDs, not full posts — hydrate content at read time so you don't duplicate post bodies across millions of lists.

### 13.5 High-level design
- Client → LB → stateless **feed service** (reads) and **post service** (writes)
- Post service writes the post, then drops a **fan-out job on a queue** — fan-out is async, off the write path, so posting feels instant
- **Fan-out workers** consume the queue: look up followers, push post ID into each follower's Redis list — *unless* the author is flagged high-fan-out (then skip)
- **Feed service** on read: pull precomputed list, merge in recent posts from celebrity-followees, hydrate content from cache/store, return
- **Cache** on the read path; **CDN** for media

### 13.6 Bottleneck and failure analysis
- **Celebrity post still bursty** — you skip fan-out, but every follower pulls-and-merges on next load. Absorb with heavy caching of celebrity posts (same post read by millions → perfect cache candidate; popularity that makes it dangerous makes it cacheable).
- **Fan-out lag** — a non-celebrity with millions of legit followers may take seconds to fully fan out. That's the licensed eventual-consistency window. Acceptable.
- **Feed cache eviction** — don't keep precomputed feeds for inactive users forever; evict and rebuild on next login (one-time pull). Ties to 80/20: precompute for the active minority.
- **Thundering herd on a cache miss** for a hot post — many readers miss simultaneously and stampede the store. Guard the rebuild with **per-key locking** so only one reader repopulates while others wait (cache-stampede prevention).

## 14. Ticket Booking

This one flips everything: the tension is **concurrency correctness**, not read scale. QPS is often modest; the difficulty is that two people must never buy the same seat. Tests locking, transactions, and idempotency — no hand-waving with "add a cache."

### 14.1 Requirements
**Functional:** browse events and seats; select and book seat(s); payment completes booking; booked seats become unavailable to everyone else.

**Non-functional:**
- **Correctness over availability** (the inversion). Double-booking is unacceptable. Prefer briefly showing an available seat as unavailable over selling it twice. Leans toward **strong consistency** for seat-state, even at some availability cost. Contrast with the feed's opposite call.
- **Latency** — matters for browsing; the booking transaction can tolerate a few hundred ms (correctness dominates).
- **Scale** — bursty. A hot on-sale is a **thundering herd on a tiny, contended dataset** (one venue's seats). The "flash sale" pattern: not high sustained QPS, but massive contention on a few rows at one instant.

### 14.2 Estimation
The interesting number is **contention**, not aggregate QPS. A 20,000-seat venue goes on sale; 500,000 people hit it in the first minute → ~8K requests/sec fighting over **20,000 rows**, most targeting the same "good seats."

**Therefore:** the bottleneck is **write contention on a small hot set of rows**. Every decision follows from managing that contention correctly without (a) selling a seat twice or (b) locking so aggressively the system grinds to a halt.

### 14.3 Core tension: the reservation problem
**Naive (broken):** pick seat → check "available?" → if yes, take payment → mark booked. Bug: between the *check* and the *mark*, another user also sees "available." Both pass, both pay, both get the seat — a **check-then-act race (TOCTOU)**. Payment takes seconds, making the window enormous. You cannot hold a DB lock while a user types card details (that serializes the whole venue behind one slow user).

**Real question:** how do you reserve a seat atomically, hold it briefly during payment, and release it if payment fails or the user vanishes? Three layered mechanisms:

**(a) Atomic reservation with a short-lived hold.** Split into *reserve* then *confirm*. Reservation flips the seat to `RESERVED` **atomically** with an expiry (e.g., 5–10 min) — long enough to pay, short enough not to strand inventory. Atomicity is the crux: `AVAILABLE → RESERVED` must be a single atomic op so only one of two racing requests wins.

**(b) The atomic transition — guaranteeing "only one wins."** In rough order of preference:
- **Conditional / compare-and-set update:**
  ```sql
  UPDATE seats SET status='RESERVED', holder=:u, expiry=:t
  WHERE seat_id=:s AND status='AVAILABLE'
  ```
  The DB guarantees the row update is atomic; the `WHERE status='AVAILABLE'` means exactly one concurrent request gets `rows_affected = 1` and wins; others get `0` → "seat taken." Optimistic concurrency, clean, pushes correctness down to the DB. **Lead with this.**
- **SELECT ... FOR UPDATE (pessimistic row lock)** inside a transaction — correct but holds a lock for the transaction's duration; scales worse under high contention. Mention it.
- **Distributed lock (Redis Redlock, etc.)** — only if seat state lives outside a single transactional store. More moving parts and failure modes (lock expiry vs work duration). Mention, don't lead.

**(c) Expiry / release of abandoned holds.** If payment fails or the user leaves, `RESERVED` must return to `AVAILABLE`. Two mechanisms — a **TTL** reclaimed by a background job (or the next reader), and an **explicit release** on payment failure. You need both: explicit release for the common case, TTL as the safety net for crashes/disappearances.

**Signal:** recognize that the long-running payment step is exactly why you can't just hold a lock, and that reserve-with-expiry decouples "claim the seat atomically (fast)" from "pay for it (slow)."

### 14.4 Idempotency and payment — the second correctness trap
Payment's demon is **the retry** (flaky networks, double-clicked "Pay," lost responses). Without protection you **charge twice**.

**Fix — idempotency key:** client generates a unique key per booking attempt and sends it with the payment request. Server records it; a repeat key returns the *original* result instead of charging again. Makes payment **exactly-once from the user's perspective** over an at-least-once network. Payment providers (Stripe et al.) expose exactly this; naming idempotency keys unprompted is a strong senior signal.

**Full booking flow:**
1. `reserve(seatId, userId, idempotencyKey)` → atomic CAS to `RESERVED` with expiry. One winner.
2. User pays → `confirm(reservationId, paymentToken, idempotencyKey)` → payment processed idempotently.
3. Payment success → `RESERVED → BOOKED`, permanent.
4. Failure/timeout → release `RESERVED → AVAILABLE`.

### 14.5 API and data model
**API:**
- `GET /events/{id}/seats → seat map with statuses` — browse path, cacheable but near-real-time (short TTL)
- `POST /reservations {seatIds, userId, idempotencyKey} → {reservationId, expiresAt}` — atomic claim
- `POST /reservations/{id}/confirm {paymentToken, idempotencyKey} → {bookingId}` — idempotent payment + finalize
- `DELETE /reservations/{id}` — explicit release

**Data model** — the store choice **inverts** from the previous problems:
- **Seats / inventory** — `seat_id (PK) | event_id | status (AVAILABLE/RESERVED/BOOKED) | holder | expiry`. Needs transactional, strongly-consistent updates with a conditional-write guarantee → **relational (Postgres/MySQL)**. Here you *don't* reach for eventually-consistent NoSQL; the whole problem is atomic state transitions. The consistency NFR chose the store — opposite of the URL shortener.
- **Bookings** — `booking_id | user_id | seat_ids | payment_id | created_at`
- **Idempotency keys** — `key (PK) | result | created_at` so retries return cached results

### 14.6 High-level design
- Client → LB → stateless **booking service** (reservation + confirm logic)
- **Relational DB** as source of truth for seat state, doing atomic conditional updates. **Shard by `event_id`** — an event's seats live together, different events don't contend, so hot-event contention is contained to one shard.
- **Payment service** (external provider), called idempotently
- **Reservation-expiry worker** — background job scanning for expired `RESERVED` seats and releasing them; plus lazy reclaim on read as a backstop
- **Cache** for the *browse/seat-map* path only (read-mostly, staleness-tolerant) — **not** for the reservation write path, which must go to the transactional store. Knowing which paths can be cached and which cannot is key here.
- **Queue / virtual waiting room** for the flash-sale burst — admit users in controlled batches, converting an unmanageable instantaneous herd into a manageable stream (how Ticketmaster-style systems survive on-sales)

### 14.7 Bottleneck and failure analysis
- **Hot rows under the flash sale** — thousands contend for the same "good" seats. Optimistic CAS handles correctness (losers retry a different seat); under brutal contention add the waiting room to throttle admission, and shard by event so one hot event doesn't starve others.
- **Partial failure: seat booked but payment ambiguous** — provider times out; did it charge? Idempotency keys + a **reconciliation step**: on timeout, query the provider by idempotency key to learn the true outcome, then finalize or release. Never assume; always reconcile.
- **Crash mid-reservation** — seat stuck in `RESERVED`. TTL + expiry worker returns it to `AVAILABLE`.
- **Distributed-lock hazard** (if used) — a lock expiring while the holder still works lets a second worker in. A transactional conditional update *inside the DB* is safer when feasible — guarantee and data live in the same place.
- **Read-your-writes on the seat map** — after reserving, you should see it reserved; route post-reservation reads appropriately or invalidate the cached map for that event.

## 15. The Meta-Lesson (Through-Line Across All Three)

**Each classic problem is defined by one NFR that dominates, and that NFR picks your entire toolkit.**

| Problem | Dominant NFR | Toolkit | Consistency stance |
|---|---|---|---|
| URL shortener | Read scale | Cache / CDN / sharding | Eventual consistency fine |
| News feed | Read scale + bimodal fan-out | Push/pull hybrid | Eventual consistency fine |
| Ticket booking | Consistency under concurrency | Transactional store, atomic conditional writes, idempotency | Sacrifices easy availability |

The skill isn't memorizing three diagrams. It's **diagnosing which NFR dominates** and letting that diagnosis pick the tools. This loops back to the opening point: "enumerate all the cases" is the wrong frame because these problems aren't won by completeness — they're won by **correctly identifying the one tension that matters** and reasoning hard about it.

Most other classic problems are variations on one of these three shapes: **read-scale, fan-out, or concurrency-correctness.**


---

# Part VII — Deep Dive: Google's Bigtable Paper

## 16. The Gist of Google's Bigtable Paper

The Bigtable paper (Chang et al., 2006 — *"Bigtable: A Distributed Storage System for Structured Data"*) describes the system Google built to store petabytes of data across thousands of commodity machines for products like Search, Maps, Gmail, and Analytics. It's the intellectual ancestor of Cassandra and HBase.

### The data model
Bigtable is a **sparse, distributed, persistent, multi-dimensional sorted map.** That one sentence is the whole model:

```
(row key, column key, timestamp) → value
```

Every word earns its place:
- **Sparse** — most cells don't exist; absent cells cost nothing.
- **Sorted** — rows are stored in lexicographic order by row key. This is the single most important design lever: rows that sort near each other live near each other, so range scans are cheap. You design row keys to exploit this (e.g., reversing domain names — `com.google.maps` — so related URLs cluster).
- **Multi-dimensional** — columns are grouped into **column families** (the unit of access control and the thing you declare up front), and each cell keeps multiple **timestamped versions** of its value, with automatic garbage collection of old versions.

### How rows are partitioned: tablets
The row range of a table is split into **tablets** — contiguous ranges of rows, roughly 100–200 MB each. Tablets are the unit of distribution and load balancing. As a table grows or shrinks, tablets split and merge automatically. Each tablet is served by exactly one **tablet server** at a time, which is what gives Bigtable **strong single-row consistency** (a given row lives on one server).

### The architecture — it stands on other Google systems
This layered design is a point interviewers love:
- **GFS (Google File System)** stores the actual data files and logs. Bigtable itself is stateless-ish about durability; GFS handles replication.
- **SSTable** is the on-disk file format — an immutable, sorted, persistent map from keys to values with a block index. Immutability does a lot of quiet work: no locking on reads, easy caching, and simple concurrent access.
- **Chubby**, Google's distributed lock service (Paxos-based), is the coordination backbone. It elects the single master, stores the location of the root tablet, holds schema and access-control info, and tracks which tablet servers are alive. If Bigtable loses Chubby for long enough, Bigtable becomes unavailable — a deliberate dependency.

There's one **master** (assigns tablets to servers, detects server failures, handles schema changes) and many **tablet servers**. Crucially, client data doesn't flow through the master — clients learn tablet locations and talk to tablet servers directly, so the master isn't a bottleneck.

### Finding a tablet: the three-level hierarchy
Locating which tablet server holds a given row uses a B+-tree-like hierarchy, similar to how a database finds a page:

```
Chubby file → root tablet → METADATA tablets → user tablets
```

The root tablet is never split, so the depth is fixed at three levels, which is enough to address a huge number of tablets. Clients cache these locations aggressively.

### The write and read path (LSM-tree, though the paper predates the popular name)
- A write goes first to a **commit log** in GFS (durability), then into an in-memory sorted buffer called the **memtable**.
- When the memtable gets big, it's frozen and flushed to a new immutable SSTable on GFS (a **minor compaction**).
- Reads must merge the memtable with the relevant SSTables. Over time SSTables accumulate, so background **major compactions** merge them into one, discarding deleted/expired data.

This is exactly the **log-structured merge (LSM)** pattern you'll see in Cassandra, RocksDB, HBase, and LevelDB. Optimizations the paper calls out:
- **Bloom filters** to skip SSTables that can't contain a key.
- **Block caches** and **key caches**.
- **Locality groups** — grouping frequently-co-accessed column families into separate SSTables (a nod toward columnar layout for specific access patterns).

### The consistency model — the important limitation
Bigtable guarantees **atomic single-row** reads and writes, no matter how many columns are involved. It deliberately does **not** provide cross-row transactions. That single choice is what lets it scale horizontally so cleanly, and it's the tradeoff to articulate: they gave up general multi-row transactions to get linear scalability and simple failure semantics. (Google later built Megastore and then Spanner specifically to add stronger transactional guarantees on top of this lineage.)

### The one-paragraph version for an interview
Bigtable is a distributed sorted map keyed by (row, column, timestamp), partitioned into contiguous row-range tablets that are each served by a single tablet server for strong single-row consistency. It layers on GFS for durable storage, immutable SSTables plus an in-memory memtable and commit log for an LSM-style write path, and Chubby for coordination and master election. It scales to petabytes by keeping the master off the data path and by giving up multi-row transactions in exchange for horizontal scalability.

### Follow-up worth knowing
Cassandra diverged from this blueprint by dropping the single master for a peer-to-peer gossip/consistent-hashing design — a classic follow-up question.
