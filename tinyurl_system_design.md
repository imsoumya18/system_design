# 🔗 TinyURL — System Design Reference

> **Purpose:** Quick revision guide for Amazon & FAANG system design interviews  
> **Difficulty:** Medium | **Category:** URL Shortener  
> **Amazon specified:** ✅ | **Interview round:** System Design — Scalability & Operational Performance

---

## 📋 Table of Contents

1. [Requirements](#1-requirements)
2. [Capacity Estimation](#2-capacity-estimation)
3. [API Design](#3-api-design)
4. [High-Level Design](#4-high-level-design)
5. [Deep Dive — Key Components](#5-deep-dive--key-components)
6. [Failure Handling & Resilience](#6-failure-handling--resilience)
7. [Scaling to 10×](#7-scaling-to-10)
8. [Operational Excellence](#8-operational-excellence)
9. [Interview Cheat Sheet](#9-interview-cheat-sheet)

---

## 1. Requirements

### ✅ Functional Requirements

| # | Requirement | Notes |
|---|---|---|
| 1 | Long URL → Short URL creation | Write operation |
| 2 | Short URL → Long URL redirection | Read operation, HTTP 301 |
| 3 | URL expiry after 5 years | Configurable TTL |
| 4 | No custom aliases | Out of scope for now |
| 5 | No analytics | Out of scope for now |

### ⚡ Non-Functional Requirements

| Requirement | Target |
|---|---|
| Availability | 99.99% |
| Redirect latency | < 100ms p99 |
| Read:Write ratio | 100:1 (read-heavy) |
| Scalability | Horizontal, auto-scaling |
| Durability | No data loss on URL mappings |

> **🎯 Interview tip:** Always ask about expiry, custom aliases, analytics, and availability SLA before designing. These answers shape your entire architecture.

---

## 2. Capacity Estimation

### Traffic

```
Writes:  100M URLs/day  ÷  86,400 sec  ≈  1,200 writes/sec
Reads:   100:1 ratio    →  120,000 reads/sec
```

### Storage

```
Per URL record:
  short_code        →   7 bytes
  long_url          →  ~200 bytes
  created_at        →   8 bytes
  expires_at (TTL)  →   8 bytes
  padding           →  ~277 bytes
  Total             ≈  500 bytes/record

Total URLs: 100M/day × 365 days × 5 years = ~180 billion URLs
Total storage: 180B × 500 bytes            ≈  90 TB
```

### Short Code Space

```
Base62 alphabet: [a-z, A-Z, 0-9] = 62 characters
7 characters   : 62^7 ≈ 3.5 trillion unique codes

3.5 trillion >> 180 billion  ✅  7 chars is sufficient
```

### Cache Sizing

```
Top 20% of URLs serve ~80% of traffic (Pareto principle)
20% of 180B records × 500 bytes ≈ 18 TB of hot cache
Use Redis Cluster to distribute across nodes
```

> **🎯 Interview tip:** Always anchor your design decisions to these numbers. "I need caching because 120K reads/sec will kill the DB" is a strong signal.

---

## 3. API Design

### Write — Create Short URL

```http
POST /urls
Content-Type: application/json

{
  "long_url": "https://www.example.com/very/long/path?query=param",
  "ttl_seconds": 157680000   // optional, default: 5 years
}
```

```http
HTTP/1.1 201 Created

{
  "short_url": "https://tiny.url/aB3xK9z",
  "short_code": "aB3xK9z",
  "expires_at": "2031-04-28T00:00:00Z"
}
```

### Read — Redirect

```http
GET /aB3xK9z
```

```http
HTTP/1.1 301 Moved Permanently
Location: https://www.example.com/very/long/path?query=param
```

> **💡 Why 301 and not 302?**  
> `301 Permanent` → Browser caches the redirect. Repeat visits never hit our servers.  
> `302 Temporary` → Every visit hits our servers. Use only if analytics per-click is needed.  
> At 120K reads/sec, 301 is a **significant load reduction**.

---

## 4. High-Level Design

```
                    ┌─────────────────────────────────────────────────────┐
                    │                   WRITE PATH                        │
                    │  POST /urls                                         │
                    │  1. Get next ID from token range (local counter)    │
                    │  2. Convert ID → Base62 (7-char short code)         │
                    │  3. Store mapping in DynamoDB with TTL              │
                    │  4. Return short URL to client                      │
                    └─────────────────────────────────────────────────────┘

                    ┌─────────────────────────────────────────────────────┐
                    │                    READ PATH                        │
                    │  GET /{short_code}                                  │
                    │  1. Hash short_code → Redis node (consistent hash)  │
                    │  2. Cache HIT  → return 301 redirect (~1ms)         │
                    │  3. Cache MISS → query DynamoDB → populate cache    │
                    │               → return 301 redirect (~5-10ms)       │
                    └─────────────────────────────────────────────────────┘
```

### System Diagram

```
                                         ┌───────────────────┐
                                         │  ID Gen Service   │
                                         │  ┌─────────────┐  │
                                         │  │Token ranges │  │
                                         │  │per server   │  │
                                         │  │e.g. 1–1000  │  │
                                         │  └─────────────┘  │
                                         │  active + standby │
                                         └────────┬──────────┘
                                                  │ on write
                                                  │ (once per 1000 reqs)
                                                  ▼
┌────────┐    ┌─────────────┐    ┌───────────────────────────┐
│        │    │             │    │       API Servers          │
│ Client │───▶│    Load     │───▶│       (stateless)          │
│        │    │  Balancer   │    │                           │
└────────┘    └─────────────┘    └────────┬──────────┬───────┘
                                          │          │
                               POST /urls │          │ GET /{code}
                                          │          │
                                          ▼          ▼
                              ┌──────────────┐  ┌────────────────────────┐
                              │   DynamoDB   │  │     Redis Cluster      │
                              │              │  │  (consistent hashing)  │
                              │  PK: code    │◀─│                        │
                              │  long_url    │  │  Primary + Replica×N   │
                              │  created_at  │  │  LRU eviction          │
                              │  TTL: 5yr    │  │  ~18TB hot URLs        │
                              │              │  └────────────────────────┘
                              │  Native TTL  │          │ cache miss
                              │  auto-expire │◀─────────┘
                              └──────────────┘
```

---

## 5. Deep Dive — Key Components

### 5.1 Short Code Generation — Base62 with Token Ranges

**Why not hash the long URL?**

| Approach | Problem |
|---|---|
| MD5/SHA256 → take 7 chars | Collision risk — different URLs → same code |
| Hash long URL | Same URL → same code (but collision handling is complex) |
| Auto-increment + Base62 | ✅ No collisions, simple, fast |

**How token ranges work:**

```
ID Gen Service maintains a global counter in persistent storage.

Server 1 requests range → gets [1, 1000]       → works locally
Server 2 requests range → gets [1001, 2000]     → works locally
Server 3 requests range → gets [2001, 3000]     → works locally

Each server converts its local ID to Base62:
  ID 1000 → Base62 → "G8"   (short code)
  ID 1001 → Base62 → "G9"   (short code)

Server hits counter only once per 1,000 URL creations.
If server crashes mid-range → at most 1,000 IDs wasted (acceptable).
```

**ID Gen Service availability:** Run as **active + standby** pair. Standby syncs state continuously. Failover in seconds.

---

### 5.2 Database — DynamoDB

**Why DynamoDB over MySQL/PostgreSQL?**

| Factor | DynamoDB | PostgreSQL |
|---|---|---|
| Access pattern | Key lookup only ✅ | Supports complex queries |
| Horizontal scaling | Native ✅ | Hard |
| Write throughput | Millions/sec ✅ | ~50K QPS max |
| Managed TTL | Native ✅ | Requires custom job |
| Latency | Single-digit ms ✅ | Varies |

**Schema:**

```
Table: url_mappings
┌─────────────┬───────────────────────────────────────────────────────┐
│ short_code  │  "aB3xK9z"                    ← Partition Key (PK)   │
│ long_url    │  "https://example.com/..."                            │
│ created_at  │  1714262400                   ← Unix timestamp        │
│ ttl         │  1872028800                   ← Unix timestamp (5yr)  │
└─────────────┴───────────────────────────────────────────────────────┘
```

**TTL (Auto-expiry):**
- DynamoDB's native TTL attribute scans for expired items
- Items stop being returned **immediately** after TTL passes
- Physical deletion happens lazily within **48 hours** in background
- Zero application code needed for cleanup ✅

---

### 5.3 Cache — Redis Cluster

**Why cache?** At 120,000 reads/sec, hitting DynamoDB directly would cost ~$50K/month and add unnecessary latency. Redis brings p99 latency from ~10ms → ~1ms.

**Consistent hashing for cache routing:**

```
                    short_code "aB3xK9z"
                           │
                    hash("aB3xK9z")
                           │
                    consistent hash ring
                    ┌──────┴──────────────────┐
                    │                         │
               Redis Node 1            Redis Node 2
              (owns range 0-33%)      (owns range 33-66%)
                                             │
                                      "aB3xK9z" → Node 2
```

**Why consistent hashing?** Ensures the same short_code always routes to the same Redis node deterministically. If a node is added/removed, only ~1/N keys are remapped — no cache miss storm.

**Cache configuration:**

```
Eviction policy : LRU (Least Recently Used)
TTL             : 5 years (match DynamoDB TTL)
Topology        : 1 Primary + 2 Replicas per shard
Write policy    : Cache-aside (lazy population on cache miss)
Target hit rate : > 99%
```

**Cache-aside pattern (read path):**

```python
def get_long_url(short_code):
    # 1. Check cache
    long_url = redis.get(short_code)
    if long_url:
        return redirect(long_url, 301)   # cache HIT

    # 2. Cache miss → query DB
    long_url = dynamodb.get(short_code)
    if not long_url:
        return 404

    # 3. Populate cache
    redis.set(short_code, long_url, ttl=FIVE_YEARS)
    return redirect(long_url, 301)       # cache MISS (slower)
```

> **⚠️ Why NOT write-through?** On URL creation (write path), we skip the cache. The URL is unlikely to be accessed immediately. Cache-aside means we only cache what's actually read — more efficient use of cache memory.

---

## 6. Failure Handling & Resilience

### 6.1 ID Gen Service Failure

```
Scenario: ID Gen Service goes down mid-flight

Impact:
  • Servers continue using their current token range locally
  • No new ranges can be allocated until recovery
  • At most 1,000 IDs wasted per server (acceptable)
  • New URL creation degrades after servers exhaust their range

Mitigation:
  • Active + standby ID Gen Service instances
  • Standby promotes automatically on health check failure
  • Recovery time: < 30 seconds
```

### 6.2 Cache Node Failure

```
Scenario: One Redis node goes down

Impact (without consistent hashing):
  • Modulo hashing breaks → massive cache miss storm → DB overload

Impact (with consistent hashing):
  • Only keys owned by the failed node remap to neighbours
  • Temporary spike in DB reads for ~1/N of traffic
  • DB read replicas absorb the spike

Mitigation:
  • Consistent hashing minimises key remapping
  • Redis Sentinel / Cluster handles automatic failover
  • Circuit breaker on DB call prevents cascade
```

### 6.3 DynamoDB Failure

```
Scenario: DynamoDB becomes slow or unavailable

Mitigation:
  • Cache absorbs 99%+ of read traffic — most users unaffected
  • Circuit breaker trips after failure threshold
  • Fast-fail on write path with appropriate error message
  • DynamoDB is multi-AZ by default — regional failure is rare
```

### 6.4 Summary Table

| Component | Failure mode | Mitigation |
|---|---|---|
| Load balancer | Single point of failure | Active-active pair |
| API servers | Instance failure | Stateless — LB routes around |
| ID Gen Service | Counter unavailable | Active + standby |
| Redis node | Cache miss storm | Consistent hashing + read replicas |
| DynamoDB | Read/write unavailable | Cache absorbs reads, circuit breaker |

---

## 7. Scaling to 10×

**Scenario:** Traffic grows to 1.2M redirects/sec

### Identify the bottleneck first

```
Layer           | Current capacity   | At 10× load      | Bottleneck?
─────────────── │ ─────────────────  │ ────────────────  │ ──────────
API servers     | ~10K RPS each      | Add more (easy)   | No — stateless
Redis cache     | ~500K ops/sec each | Add shards        | No — distributable
DynamoDB writes | Auto-scales        | 12K writes/sec    | No — managed
DynamoDB reads  | Depends on miss    | ← HERE            | Potential ⚠️
```

### The math

```
Cache miss rate assumption: 1%
At 1.2M reads/sec with 1% miss rate:
  → 12,000 DB reads/sec from cache misses alone

Without mitigation: DynamoDB read throughput pressured
```

### Strategy (in order of priority)

```
Step 1: Increase Redis cache size
        More memory → higher hit rate → fewer DB reads
        1% miss → 0.3% miss = 4× reduction in DB pressure
        Cost: relatively cheap (RAM is cheap vs DB capacity)

Step 2: Increase Redis replica count
        More replicas → distribute read traffic further
        Each replica serves a portion of the 120K → 1.2M reads/sec

Step 3: Add DynamoDB read capacity / read replicas
        Last resort — handles whatever cache misses remain
        DynamoDB auto-scales on-demand → no manual provisioning

Step 4: Scale ID Gen Service token ranges
        Increase range size 1K → 10K per allocation
        Counter service hit rate drops 10× automatically
```

> **🎯 Interview tip:** Always fix the cheapest layer first. Scaling cache is cheaper than scaling DB. Reducing miss rate is cheaper than adding replicas.

---

## 8. Operational Excellence

### SLIs (What to measure)

| Metric | Target | Alert threshold |
|---|---|---|
| p99 redirect latency | < 100ms | > 50ms |
| Cache hit rate | > 99% | < 98% |
| Error rate (5xx) | < 0.1% | > 0.01% |
| URL creation success rate | > 99.9% | < 99.5% |
| DynamoDB read throughput | — | > 80% capacity |

### Monitoring checklist

```
✅ p99 latency per endpoint
✅ Cache hit/miss ratio per Redis shard
✅ DynamoDB consumed read/write capacity
✅ ID Gen Service range allocation rate
✅ Error rate by error type (404, 5xx)
✅ Distributed traces on slow requests (> 50ms)
```

### Runbooks

**Cache miss storm detected:**
1. Check if a Redis node is down → check Redis Sentinel status
2. If node down → verify automatic failover completed
3. Monitor DB read throughput — should return to baseline within 5 min
4. If DB under pressure → temporarily increase cache TTL on hot keys

**URL creation failing:**
1. Check ID Gen Service health → primary and standby
2. Check DynamoDB write capacity — auto-scaling may lag by 1-2 min
3. If ID Gen both down → servers burn through local ranges — ~16 min buffer at 1,200 writes/sec with range 1,000

---

## 9. Interview Cheat Sheet

### The 6-step Amazon framework (timing)

```
Step 1: Clarify requirements        3–5 min   ← Never skip
Step 2: Capacity estimation         2–3 min   ← Anchor everything to numbers
Step 3: API design                  2–3 min   ← Define contract before boxes
Step 4: High-level design           5–8 min   ← Draw while narrating
Step 5: Deep dive (scale + ops)    10–15 min  ← The meat of the interview
Step 6: Operational excellence      3–5 min   ← Amazon differentiator
                                   ──────────
                                   ~35–40 min total
```

### Key decisions & justifications

| Decision | Justification |
|---|---|
| DynamoDB over PostgreSQL | Key-only access pattern, horizontal write scaling, native TTL |
| Base62 over MD5 hash | No collision risk, predictable length, easy to decode |
| Token ranges over central counter | Reduces network calls, survives brief ID Gen outage |
| Cache-aside over write-through | Only cache what's actually read — memory efficient |
| 301 over 302 redirect | Browser caches it — reduces load at scale |
| Consistent hashing on cache | Same key always hits same node, minimal remapping on topology change |
| DynamoDB TTL over cleanup job | Zero operational overhead, platform-managed |

### Numbers to remember

```
Writes:         1,200 / sec
Reads:        120,000 / sec
Storage:           90 TB (5 years)
Short code:     7 chars, Base62, 3.5T unique codes
Cache target: > 99% hit rate
Redirect SLA: < 100ms p99
```

### Amazon LP moments in this design

| LP | Where it shows |
|---|---|
| **Dive Deep** | Quantifying exactly what happens at 10× load — 12K DB reads/sec at 1% miss |
| **Bias for Action** | Choosing DynamoDB TTL over custom cleanup job — use the platform |
| **Frugality** | Scale cache before scaling DB — cheaper solution first |
| **Have Backbone** | Recommending 301 (not 302) even though it removes analytics capability |
| **Operational Excellence** | Proactively defining SLIs and runbooks without being asked |

---

## 🔁 Quick Revision — 60 second version

```
TinyURL in one breath:

Client → LB → Stateless API servers
                │
                ├─── WRITE: Get ID from token range → Base62 → store in DynamoDB (TTL=5yr)
                │
                └─── READ:  Check Redis (consistent hash) → HIT: 301 redirect
                                                          → MISS: DynamoDB → populate cache → 301

Scale: 1,200 writes/sec | 120,000 reads/sec | 90TB | 7-char Base62 | 99%+ cache hit
Failure: ID Gen = active+standby | Cache = consistent hashing | DB = cache absorbs 99% reads
Expiry: DynamoDB native TTL — zero ops overhead
10×: Increase cache size first → lower miss rate → add read replicas last
```

---

*Last updated: April 2026 | Prepared for Amazon SDE System Design Interview*
