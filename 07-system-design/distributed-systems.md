# Distributed Systems (C3)

> Section IDs like `C1.6` or `C3.12` are cross-references inside the core systems notes: C1 = [Linux internals](../04-linux-os/linux-internals.md), C2 = [networking](../05-networking/networking-core.md), C3 = [distributed systems](../07-system-design/distributed-systems.md), C4 = [reliability & SLOs](../07-system-design/reliability-slos.md), C5 = [debugging method](../06-troubleshooting/debugging-method.md), C6 = [numbers](../07-system-design/capacity-numbers.md), C7 = [coding patterns](../01-coding/sre-practical-problems.md).

**C3.1 — CAP, stated correctly.**
CAP is about *behaviour during a partition*: when the network partitions, you either refuse requests to preserve linearizability (CP) or serve possibly-stale data (AP). It says nothing about the non-partitioned case, which is where systems spend ~all their time.
**PACELC is the version worth using:** if Partitioned, choose Availability or Consistency; Else, choose Latency or Consistency. That framing is what lets you say "Spanner is CP/EC — it pays latency for consistency via TrueTime; Dynamo is AP/EL."
*In interview:* never say "we'll go with AP." Say which operations need linearizability and which tolerate staleness, and how stale is acceptable. Mixed consistency per operation is the senior answer.

**C3.2 — The consistency ladder.**
Linearizable (single global order, real-time respecting) → sequential → causal (respects happens-before, the strongest that survives partitions) → read-your-writes / monotonic reads (session guarantees) → eventual.
*Consequence:* most user-visible weirdness is a missing *session* guarantee, not a missing global one. "I posted and it disappeared" is read-your-writes. Sticky routing or a write-timestamp token in the session fixes it cheaply. Say this instead of reaching for consensus.

**C3.3 — Consensus: Raft in one paragraph.**
Leader election by randomised timeouts + term numbers; the leader appends to its log and replicates; an entry commits once a majority persists it; followers apply committed entries in order. Safety: a candidate can only win with a log at least as up to date as the majority's, so committed entries are never lost.
*Consequence:* consensus costs you a majority round-trip per write and caps write throughput at one leader's capacity. Use it for *metadata and membership* — who is the leader, which shard lives where, config — and keep the data plane out of it. That division is the thing to say.

**C3.4 — Quorums.**
`R + W > N` gives read-your-writes overlap; `W > N/2` prevents split-brain writes. N=3, W=2, R=2 is the workhorse. Sloppy quorums with hinted handoff (Dynamo) buy availability at the cost of a window where reads can miss.

**C3.5 — Leader election pitfalls.**
Two nodes can both believe they're leader across a partition or a long GC pause. Mitigations: fencing tokens (monotonically increasing epoch numbers that downstream storage checks and rejects if stale) and leases with a bounded clock-drift assumption. **Fencing tokens are the correct answer** — a lease alone does not protect against a paused process waking up and writing.

**C3.6 — Clocks.**
Physical clocks drift and jump (NTP steps, leap seconds). Lamport clocks give a partial order; vector clocks detect concurrency but grow with participants; hybrid logical clocks (HLC) give causally-consistent timestamps that stay close to wall time. Spanner's TrueTime exposes an *interval* and waits out the uncertainty.
*Consequence:* never order events across machines by wall clock. "Last write wins by timestamp" silently loses data whenever clocks disagree, and clocks always disagree.

**C3.7 — Replication modes.**
Sync (durable on commit, latency = slowest replica, availability = weakest link) / async (fast, can lose the tail on failover) / semi-sync (ack from ≥1 replica — usually the right trade). Leader-follower vs multi-leader (needs conflict resolution) vs leaderless.
*Consequence:* your RPO is exactly your replication lag. If lag is 30s and you fail over, you lost 30s of writes. Monitor lag as an SLI, not a debug metric.

**C3.8 — Conflict resolution.**
LWW (lossy, see C3.6), version vectors + application merge, CRDTs (commutative/associative/idempotent merges — counters, sets, sequences; no coordination needed), or operational transform.
*Apple relevance:* this is the heart of the classic "design settings sync across devices" question. CRDTs plus per-device version vectors, with the merge happening on-device so the server never needs to read plaintext. See A1.

**C3.9 — Sharding.**
Range (good for scans, prone to hot ranges) vs hash (uniform, kills range scans) vs directory/lookup (flexible, adds a lookup hop and a metadata service to keep available).
**Consistent hashing** with virtual nodes: adding a node moves only `K/N` keys instead of rehashing everything. Virtual nodes are what fix the load-variance problem of naive consistent hashing.

**C3.10 — Hot keys / hot shards.**
Detection: per-key or per-shard QPS histograms, not averages. Mitigation: key salting (append a bucket suffix, fan out reads), a small local cache in front, request coalescing (single-flight), or dedicated shards for known-hot tenants.
*Consequence:* "we'll shard by user ID" fails the moment one user is Beyoncé. Say the mitigation before you're asked.

**C3.11 — Backpressure and load shedding.**
Bounded queues everywhere; when full, reject fast rather than queue. Prefer shedding by *priority* (drop batch/retry traffic before interactive). Adaptive concurrency limits (Little's-law-derived, à la Netflix concurrency-limits) beat static thread pools.
*Consequence:* an unbounded queue converts a throughput problem into an unbounded-latency problem plus an OOM. Naming "bounded queue + shed" unprompted is a strong senior signal in both loops.

**C3.12 — Retries and the amplification trap.**
Retries multiply load exactly when the system is least able to take it. Required together: exponential backoff **with jitter** (full jitter, not fixed), a **retry budget** (cap retries at e.g. 10% of requests, service-wide), and **circuit breakers** to stop calling a dead dependency.
*Consequence:* without a budget, a partial failure becomes a full outage via retry storm. This is the single most common cascading-failure mechanism and worth having a story for.

**C3.13 — Idempotency and exactly-once.**
There is no exactly-once *delivery*. There is at-least-once delivery plus idempotent processing, which yields exactly-once *effect*. Implement via client-supplied idempotency keys with a dedup store (TTL'd), or via monotonic sequence numbers per producer, or by making the operation naturally idempotent (set, not increment).
*Consequence:* the dedup store's retention window is your correctness boundary. State it.

**C3.14 — Cascading failure, and the standard defences.**
Mechanism: a dependency slows → callers' threads/connections pile up → callers slow → their callers pile up → the whole call graph is saturated, and it does not self-recover even after the original fault clears (metastable failure).
Defences: timeouts everywhere (with budgets that *decrease* down the call chain), circuit breakers, bulkheads (isolated pools per dependency), load shedding, and a documented recovery procedure that includes shedding traffic to let the system drain.
*Consequence:* the recovery insight — that you often must *reduce* load below the original level to escape a metastable state — is a genuinely senior thing to say in G1 or A4.

## One-line summary

**Distributed:** PACELC not CAP · session guarantees fix most weirdness · consensus for metadata, not data plane · fencing tokens over leases · never order by wall clock · RPO = replication lag · retries need budget + jitter + breaker · at-least-once + idempotent = exactly-once effect.
