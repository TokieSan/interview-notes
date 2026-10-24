# Build Systems / Developer Infrastructure Design

For infra roles on build, CI, or developer-tooling teams (written for an Apple Developer Foundations loop). There, "distributed systems" means remote execution, content-addressed storage, action caching and DAG scheduling — not consensus and CAP. Default to web-service vocabulary and you miss the whole loop.

**Non-negotiable framing for the whole loop:** this team builds a *large-scale functional build system* — the thing that compiles apps and operating systems at Apple. "Distributed systems" here means remote execution, content-addressed storage, action caching, and DAG scheduling. It does **not** mean consensus, quorum, or CAP. If you default to web-service design vocabulary you will miss the whole loop.

## Vocabulary — the Bazel Remote Execution API

Read the **Bazel Remote Execution API** spec. It's the public lingua franca for exactly what this team does, and fluency in it is the single highest-ROI thing available to you.

| Term | What it is | Why they care |
|---|---|---|
| **Digest** | `{hash, size}` — identity of a blob | Everything is keyed by content, not name |
| **CAS** | Content-Addressable Store | Dedup across all of Apple's builds; immutable, so no invalidation |
| **Action** | `{command, input root digest, env, platform}` | The unit of work; its digest is the cache key |
| **ActionCache** | Action digest → ActionResult | The hit-rate lever; also the correctness hazard |
| **ActionResult** | output digests, exit code, stderr/stdout | What a hit returns instead of re-running |
| **Hermeticity** | Action depends on *nothing* outside declared inputs | Violate it and you get a false cache hit — silent wrong binary |
| **Input root** | Merkle tree of the action's input files | Why a one-byte header change invalidates thousands of actions |
| **Remote execution vs remote caching** | Run it on the farm vs just fetch the result | Different failure modes, different SLOs |
| **Straggler** | One slow action holding the critical path | Mitigation = speculative re-execution |
| **Critical path** | Longest dependency chain in the build DAG | Sets the floor on build latency regardless of fleet size |

**The one insight to have a view on:** cache hit rate and correctness trade against each other. Loosening cache keys (ignoring env vars, timestamps, absolute paths) raises hit rate and risks false hits. A false hit ships a wrong binary and is *worse than an outage* — an outage is loud, a false hit is silent. Say this out loud at some point on Sep 2.

## Likely coding shapes for a build-infra team

Rank-ordered by fit to a build team:

1. **DAG scheduler with a worker pool.** Given actions with dependencies, execute across N workers, maximise parallelism. Follow-ups: cycle detection, failure propagation (fail-fast vs continue-on-error), priority by critical-path length, dynamic DAG (actions discovered mid-build).
2. **Content-addressed cache with eviction.** LRU or LFU over digests, size-bounded, with reference counting so you don't evict a blob an in-flight action needs. Follow-up: concurrent access.
3. **Merkle tree diff.** Given two input-root trees, find changed subtrees. This is incremental build in miniature.
4. **Bounded work queue / rate limiter.** Classic concurrency primitive, but frame answers around backpressure onto build clients.

**Drill these three, in your strongest language, and be ready to switch:**
- Topological execution with a thread pool and a dependency counter per node
- LRU with TTL and refcount pinning
- Merkle tree construction + diff

## Design: "remote build cache for a whole company"

Structure your answer:

**Requirements probe (do this, don't skip it).** Read-heavy or write-heavy? Cold-start builds or incremental? How many engineers, how many actions/build? What's the artifact size distribution (many tiny .o files, a few enormous linked binaries — this bimodality matters)?

**Then:**
- Split CAS from ActionCache. CAS is immutable, cacheable at edge, dedupes globally. ActionCache is small, mutable-ish, correctness-critical.
- Tiering: local disk cache → regional cache → origin. 
- Write path: upload-if-absent (`FindMissingBlobs` first) — most blobs already exist, so don't ship bytes you don't need to.
- Eviction: LRU on CAS is fine because it's just a perf loss. Eviction from ActionCache while its referenced blobs are gone is a *correctness* bug — dangling reference. Have an answer.
- Bimodal artifact handling: inline small blobs in the response, stream large ones.

**The confidentiality section — this is your differentiator.**

Do **not** talk about differential privacy or on-device processing. The correct frame:

- The cache is multi-tenant across all the company SWE. Unreleased OS features live in the build graph.
- **Cache-key leakage:** if I can query the ActionCache with a guessed action digest and get a hit, I've learned that someone built that exact thing. Framework names, symbol names, and feature flags leak via input-root contents.
- **Timing side channel:** hit vs miss latency reveals cache membership even without read access.
- Mitigations: per-tenant key namespacing (salt the action digest with a project/team ID) — costs you cross-team dedup, which is the entire value proposition. **State that trade-off explicitly.** Partial answer: allow global dedup on the CAS layer (content is opaque without the digest, which is itself a capability) while namespacing the ActionCache.
- Build logs and stderr as an exfiltration surface — they're in the ActionResult.
- Need-to-know vs velocity: locking down too hard means teams run local builds, hit rate collapses, and you've destroyed the product to protect it.

That last bullet is the kind of thing this team argues about internally. It shows you understand the product, not just the security checklist.

## Systems programming for build infrastructure

### Track A — event-driven I/O (SwiftNIO territory)

- `kqueue` (Darwin/BSD) vs `epoll` (Linux). Level- vs edge-triggered. Why edge-triggered requires you to drain until `EAGAIN`.
- Event loop architecture: one loop per core, thread affinity, why you never block the loop.
- Backpressure: what happens when a reader is slower than a writer. Buffer growth, watermarks, flow control. **A signature topic for event-loop people** — SwiftNIO's design is largely about not letting an unbounded buffer eat your process.
- Zero-copy: `sendfile`, `writev`/scatter-gather, why you'd rather slice a buffer than copy it.
- Allocation pressure in hot paths — why allocations per request matter at scale.
- `TCP_NODELAY`, Nagle, delayed ACK interaction.

### Track B — build-system OS (the team's actual domain)

- **Process spawning:** `fork`/`exec` vs `posix_spawn` and why the latter matters when the parent has a huge address space.
- **Sandboxing:** how you enforce hermeticity. Syscall interception, `sandbox_init` on Darwin, seccomp/namespaces on Linux, overlay/bind mounts to present only declared inputs. Why "trust the action to declare its inputs" fails.
- **Filesystem:** APFS `clonefile` and copy-on-write for cheap input staging; hardlinks vs symlinks vs clones for output materialisation; why symlinks break hermeticity.
- **File watching:** FSEvents vs `inotify`, coalescing, why watching a giant tree is expensive.
- `mmap` vs `read`/`write` for artifact serving — page cache behaviour, and when `mmap` hurts (page faults on a cold NFS-backed store).
- Linking: static vs dynamic, why link steps are the serial tail of most build DAGs.

## Outages and debugging in a build farm

"Production outage" here means **the build farm degraded and every engineer at Apple is blocked.** The blast radius is internal but total. Distinctive properties worth naming:

- Failure is loud and instantaneous — thousands of engineers notice in minutes. Contrast with a *false cache hit*, which is silent and ships a wrong binary. Rank these correctly.
- Load is bursty and correlated: everyone syncs to main in the morning, CI fires on every push, and a compiler or SDK version bump invalidates the whole cache at once — a **cold-cache thundering herd**.
- Degraded mode is available and valuable: fall back to local builds. Slow is much better than blocked. Have this in your answer.

Have a method, not a list:
- Distinguish *slow* from *wrong*. Different investigations entirely.
- For a build system specifically: is it the client, the cache, the scheduler, or the workers? The answer is usually visible from where latency accumulates — queue time vs execution time vs fetch time. Name that decomposition.
- Correlated vs independent failures. One bad worker poisoning results is a different problem from fleet-wide degradation, and the tell is the *distribution* of failures across workers, not the count.
- Bisecting on the DAG: which action started failing, and what changed in its input root.

## Success metrics

| Metric | Why | The tension |
|---|---|---|
| **Cache hit rate** | The core value lever | Rises if you loosen keys — which risks false hits |
| **p50 / p95 end-to-end build latency** | What engineers actually feel | p50 is the daily experience; p95 is what they complain about |
| **Queue wait vs execution time** | Separates capacity problems from efficiency problems | Adding workers fixes one, not the other |
| **Correctness / false-hit rate** | Catastrophic, silent | Nearly impossible to measure directly — you need periodic cache-bypass verification builds, which cost capacity |
| **Critical path length** | Fleet size can't fix a serial DAG | Structural; needs build-graph changes, not infra |

**The staff-level point:** the metric that matters most (false-hit rate) is the one you can't observe directly, so you have to spend real capacity on verification builds to sample it. Deciding how much capacity to spend on measuring correctness versus serving builds *is* the competing-demands question in this domain. Land that.
