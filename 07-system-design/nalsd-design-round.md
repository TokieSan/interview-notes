# Non-Abstract Large System Design (NALSD)

Google's SRE design round ("Architectural Patterns", formerly NALSD): capacity planning + component isolation + graceful degradation, with real arithmetic. The method transfers to any infra design round. > Section IDs like `C1.6` or `C3.12` are cross-references inside the core systems notes: C1 = [Linux internals](../04-linux-os/linux-internals.md), C2 = [networking](../05-networking/networking-core.md), C3 = [distributed systems](../07-system-design/distributed-systems.md), C4 = [reliability & SLOs](../07-system-design/reliability-slos.md), C5 = [debugging method](../06-troubleshooting/debugging-method.md), C6 = [numbers](../07-system-design/capacity-numbers.md), C7 = [coding patterns](../01-coding/sre-practical-problems.md).

### 3.1 What "non-abstract" means

Not "design Twitter." Google's own framing: NALSD **combines capacity planning, component isolation, and graceful degradation**, and you iterate designs until you reach one that survives real constraints. You will be doing arithmetic. Recruiters state it explicitly: **you will almost never write code**, beyond pseudocode for a data payload.

### 3.2 The five-question loop (this is the actual Google framework — use it explicitly)

**Phase 1 — basic design**
1. **Is it possible?** Ignoring resource limits, what design satisfies the requirements at all?
2. **Can we do better?** Faster, smaller, more efficient? O(N) → O(log N)?

**Phase 2 — scale it up**
3. **Is it feasible?** Given real money, hardware, and network constraints, can this design actually be built? *(This is where the arithmetic lives.)*
4. **Is it resilient?** What happens when a machine, rack, or region dies? What's the degraded mode?
5. **Can we do better?** — again, now with scale constraints in hand.

Announce the loop at the start: *"I'll start with a single-machine design to establish the mechanics and get concrete numbers, then scale it and check feasibility and resilience at each step."* You have now told the interviewer you know the round's name in their own vocabulary. That is worth a lot.

### 3.3 The 60-minute shape

| Minutes | Phase | Deliverable |
|---|---|---|
| 0–8 | **Clarify.** Requirements, users, scale, SLOs, read/write ratio, consistency needs, retention, what's explicitly out of scope | A written requirements list in the doc |
| 8–15 | **Estimate.** QPS, bandwidth, storage, working set — C6.3 | Numbers on the page |
| 15–22 | **One machine.** Does it fit on one box? What breaks first? | Named binding resource |
| 22–40 | **Distribute.** Sharding, replication, coordination, the data path | Component diagram + data flow |
| 40–52 | **Resilience & ops.** Failure domains, degraded modes, monitoring, SLIs, rollout | Failure table |
| 52–60 | **Trade-offs & summary.** What you'd change with 10× scale, what you deliberately didn't build | Explicit trade-off statement |

**Write in the doc as you go.** The interviewer is transcribing; help them. Numbered requirements, a table of numbers, a component list. Do not talk for 60 minutes into an empty document.

### 3.4 Clarifying questions that always apply
- Who are the users, how many, and what's the growth assumption?
- What's the read:write ratio? What's the payload size distribution — mean *and* tail?
- What's the availability target, and what's the latency SLO, at which percentile?
- What consistency does each operation need? Can any read be stale, and by how long?
- Retention: how long do we keep the data, and is deletion a hard requirement (GDPR)?
- What's explicitly out of scope? Auth? Billing? Multi-region?
- Is there existing infrastructure I should assume — a cluster manager, a blob store, a pub/sub?

### 3.5 Worked example 1 — Distributed log ingestion pipeline
*A good first drill because the domain (log ingestion) is familiar to most infra engineers and exercises every part of the method.*

**Requirements to establish:** 10 M hosts? No — say 50k hosts emitting logs, average 2 KB/event, 200 events/host/second, retained 7 days, consumers must read within 5s of write (freshness SLO), no event loss above 10⁻⁶, and out-of-order tolerance of 1 minute.

**Numbers:**
- Ingest rate = 50,000 × 200 = **10 M events/s**. At 2 KB → **20 GB/s** ingest.
- That's 160 Gbps — immediately, no single machine. Binding resource named in minute nine.
- 7-day retention = 20 GB/s × 604,800 s = **12 PB**, ×3 replication = **36 PB**.
- If a storage node holds 100 TB usable → **360 nodes** on storage alone.
- Network per ingest node at 25 Gbps ≈ 3 GB/s usable → 20 GB/s ÷ 3 = **~7 nodes** minimum for ingest bandwidth, but you'll need far more for fan-out to consumers and replication traffic (×3 write amplification → ~21+, then ×2 for headroom).
- **Binding resource: storage, then network.** Say this.

**Design progression:**
1. *One machine:* append to a local file, index by offset. Establishes the mechanics — sequential writes are cheap (~1 GB/s NVMe), reads are offset seeks. Breaks at ~1 GB/s and ~10 TB.
2. *Partition:* shard by a key. **Which key?** Hashing by host gives uniformity but destroys per-service locality for consumers; hashing by (service, host) with virtual nodes gives both. Name the hot-shard risk (C3.10) — one pathological service can dominate — and the mitigation: per-partition rate limits and dynamic split.
3. *Replicate:* leader per partition, follower replication, ack on W=2 of N=3 for a bounded RPO. Consensus (Raft) for **partition leadership and metadata only**, never for the data path (C3.3). This division is the senior signal.
4. *Consumers:* offset-based, consumer-managed checkpoints, at-least-once delivery plus idempotent consumers (C3.13). Explain why exactly-once delivery isn't a thing.
5. *Backpressure:* bounded queues at the producer with local spill-to-disk; when the buffer fills, shed low-priority log levels first (C3.11). **Never block the application being logged from.** That's a real operational principle and worth stating flatly.

**Resilience table** (make one of these in every design):

| Failure | Detection | Impact | Mitigation | Degraded mode |
|---|---|---|---|---|
| Ingest node dies | Health check + partition lag | Producers on that partition stall | Producer failover to another partition | Ordering guarantee weakens to per-producer |
| Partition leader dies | Raft election timeout | ~5s write stall on that partition | New leader from ISR | Bounded write unavailability |
| Rack loses power | Rack-level metric | Up to 1/3 replicas | Replicas placed across failure domains | None if placement is correct |
| Consumer falls behind | Lag SLI | Reads stale beyond freshness SLO | Scale consumers; shed retention | Serve stale, alert |
| Disk fills | Capacity SLI, projected | Writes fail | Enforce retention aggressively; reject low-priority first | Drop debug-level logs |
| Retry storm from producers | Retry rate SLI | Amplified load | Retry budget + jitter + breaker | Shed |

**SLIs to declare:** ingest success ratio, end-to-end freshness (p99 write→readable), consumer lag, and durability (measured by an injected canary event, not by hope).

### 3.6 Worked example 2 — Global metrics/monitoring pipeline
Establish: 1 M monitored targets, 500 time series each, 10s scrape interval, 13-month retention for downsampled data. → 50 M series, 5 M samples/s. At ~2 bytes/sample compressed (Gorilla-style delta-of-delta + XOR — *name the compression, it's the difference between a plausible and an impressive answer*), that's 10 MB/s raw and ~315 TB/year before downsampling.
Key discussions: push vs pull (pull gives you a free liveness signal and no unknown senders; push handles NAT and short-lived jobs — real systems do both), cardinality explosion as the dominant failure mode and how you defend (label limits, cardinality budgets per team, enforcement at ingest), hot/warm/cold tiering, and query fan-out with per-query resource limits so one bad query can't take out the read path.

### 3.7 Worked example 3 — Migrate a live datastore with no user impact
*A classic SRE design prompt, directly relevant to Apple's A1.*
The shape: **dual write → backfill → dual read with comparison → flip reads → stop dual write → decommission.** At each stage: how do you verify correctness (shadow reads with diffing, sampled), what's the rollback (and how fast), what breaks if you're halfway when something fails, and how do you handle writes that arrive during backfill (versioning, or backfill-then-catch-up-from-a-changelog). Emphasise **reversibility at every step** and the fact that the comparison phase runs long enough to cover a full weekly traffic cycle.

### 3.8 Other prompts to run cold
- Design a distributed rate limiter for a global API.
- Design a caching server.
- Design a thumbnail/image service.
- Design a system to copy a file to 10,000 remote servers. *(the good answer is a distribution tree or BitTorrent-style swarm, not 10,000 unicast pushes; do the bandwidth math to prove it)*
- Design a distributed cron/scheduler with exactly-once semantics.
- Design a chat system for 10 M users.
- Design a build/CI system for a monorepo. *(overlaps Apple's Developer Foundations domain — good dual-use practice)*
- Design a service that detects when a machine in the fleet is unhealthy.

### 3.9 The G3 failure modes

| Failure | Fix |
|---|---|
| Abstract boxes, no numbers | Force yourself to produce a QPS figure by minute 12 |
| Designing before clarifying | 8 minutes of questions, minimum, written down |
| Never naming the binding resource | Derive machine count from CPU, memory, disk, *and* network; say which one wins |
| No failure discussion until prompted | Build the resilience table unprompted around minute 40 |
| Defending a choice as "correct" | Google explicitly wants coherence, not correctness. Say "I'm choosing X, which costs me Y; the alternative Z would be better if the workload were W" |
| Running out of time mid-design | Timebox visibly: "I've got 20 minutes left, so I'll go deep on the storage layer and stay high-level on the query path unless you'd prefer otherwise" |

---

# Worked example — distributed rate limiter for 1M QPS

- **Requirements:** limit each API key to N req/min, globally, low added latency, fail-open or fail-closed?
- **Numbers:** 1M QPS peak; if 1M distinct keys, counters fit in memory (each ~tens of bytes → ~tens of MB).
- **Design:** sliding-window or token-bucket counters in a sharded in-memory store (Redis-like), keyed by API key, sharded by hash(key) so each shard owns a slice; app tier checks+increments atomically. Local per-node cache + periodic sync to reduce hot-key contention (approximate but cheap).
- **Bottlenecks:** hot keys → shard further / local pre-aggregation; store node failure → replicate, decide fail-open vs fail-closed; atomicity → atomic increment / Lua script.
- **Reliability:** what happens if the limiter store is down? (Probably fail-open for availability, with alerting.)

# Other prompts to drill

Design: a distributed log/ingestion system, a URL shortener at scale, a metrics/monitoring pipeline, a distributed cache, a CDN, a key-value store, a notification fan-out system, "store and serve N PB of logs with M-day retention."

# Practice design prompts

- **Design an Event Calendar module like Google Meet** (HLD + LLD: data model, APIs, scheduling/conflict detection, scalability) — *generic summary*.
- **Design a database recommendation reasoning** (DynamoDB vs Cassandra — defend on latency, partitioning, consistency, ops overhead, *not* internals) Read white papers: Aurora, DynamoDB, Kafka — focus on *problem → tradeoff → why this design*.
- **Ad serving system** (scale the ad-server problem in [worked-problems](../01-coding/worked-problems.md)) · **Order book** · **LRU/LFU at scale** · **Distributed rate limiter / stream dedup** (see the worked rate-limiter example above).
- Depth expected: defend *why* a component, name tradeoffs, size with back-of-envelope math.
