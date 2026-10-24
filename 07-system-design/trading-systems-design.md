# Trading-Systems Design Prompts

Design questions typical of trading firms (Citadel Securities, HRT). Correctness, latency and auditability dominate.

# Design a distributed system for processing transactions

Structure the answer; don't ramble. Trading firm context → correctness, low latency, auditability.

**1. Clarify requirements:** throughput (txns/sec), latency target, consistency needs (financial = strong), durability (can't lose a trade), ordering guarantees, read vs write ratio.

**2. Core properties:**

- **ACID** for the money path — atomicity + durability are non-negotiable. Often via a database with transactions or an event log.
- **Idempotency** — every request carries a unique ID so retries don't double-process. Critical for exactly-once effect over an at-least-once network.
- **Ordering** — sequence numbers / a sequencer; total order for a matching engine.

**3. Architecture:**

- **Ingestion / gateway** (validate, authenticate, assign IDs) → **durable log** (Kafka / append-only journal, replicated) → **processing/matching engine** → **state store** → **downstream** (settlement, risk, reporting).
- **Write-ahead log / event sourcing:** append immutable events first (durable + audit trail), then apply to state. Replay = recovery.
- **Partitioning/sharding** by account/instrument for horizontal scale; keep related txns on one shard to preserve local ordering & avoid cross-shard txns.

**4. Consistency & correctness:**

- Single-shard = easy strong consistency. Cross-shard money movement (A→B) needs **atomic commit**: **two-phase commit** (blocking, coordinator SPOF) or **Saga** (compensating transactions, eventual consistency) — name the trade-off.
- **Consensus (Raft/Paxos)** for replicating the log/state so no committed txn is lost on node failure. Leader handles writes, followers replicate, majority quorum.

**5. Reliability (SRE lens):**

- Replication (N replicas, quorum writes), leader election, failover.
- Idempotent + retriable; dead-letter queue for poison messages.
- Backpressure on overload; rate limiting.
- **CAP:** financial systems pick **CP** (consistency over availability) on the money path. **PACELC:** even without partitions you trade latency vs consistency.
- Exactly-once = at-least-once delivery + idempotent processing + dedup.

**6. Observability:** metrics (latency p50/p99/p999, throughput, error rate), structured logs, distributed tracing, audit log, reconciliation jobs, alerting on SLO burn.

**7. Failure modes to volunteer:** duplicate processing, out-of-order delivery, split-brain, clock skew (use logical clocks / a sequencer, not wall-clock), coordinator failure, slow-consumer backpressure.

# Reliable market-data fan-out

**Q7. Design reliable market-data delivery from one source to many consumers.**
> I'd fan out over **UDP multicast** so the source sends once regardless of subscriber count — that's how exchanges do it and it keeps per-receiver state off the publisher. Since UDP can drop/reorder, I layer reliability: monotonic **sequence numbers** so receivers detect gaps, a **gap-fill/retransmit** channel or a snapshot+incremental scheme to recover, and **dual redundant feeds (A/B)** that receivers arbitrate so a loss on one is covered by the other. For control/slow paths where reliability beats latency I'd use TCP. Key trade-off I'd call out: multicast optimizes fan-out latency at the cost of building reliability yourself.

# Design packet routing between a hub and node servers

Structure it: addressing/topology → transport choice (TCP for reliability vs UDP/multicast for fan-out latency) → failure handling (heartbeats, timeouts, retries, idempotency) → ordering/dedup (sequence numbers) → backpressure → redundancy (dual feeds / A-B arbitration).
