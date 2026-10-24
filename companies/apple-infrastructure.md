# Apple — Software Engineer, Infrastructure

## How Apple is different

Three structural facts, each with a direct consequence:

**1. Hiring is team-driven, not centrally calibrated.** You applied to a specific team; that team runs the loop and makes the decision. Interviewers pick their own questions, usually from their actual work, and evaluate on their own judgement. There is no hiring committee reading a standardised form.
→ *Consequence:* **domain fit beats generic excellence.** Preparing "Apple questions" is less useful than preparing "SWE Developer Foundations questions." Your recruiter is a genuinely useful source here — see [questions-to-ask](../08-behavioral/questions-to-ask.md).

**2. Privacy is an engineering constraint, not a compliance section.** Access controls and disclosure classifications are enforced granularly inside Apple, and this shows up directly in design rounds. The most-cited Apple design failure is treating privacy as a closing paragraph.
→ *Consequence:* privacy enters your design **during requirements**, alongside latency and availability. Not at minute 40.

**3. Interviewers deliberately withhold feedback.** Multiple candidate reports describe rounds run without encouragement, specifically to see whether you keep making progress in the absence of signal, and whether you can **defend a decision under pressure** — repeated as the single most common reason for rejection. One candidate was asked point-blank why they switched libraries instead of contributing a fix upstream.
→ *Consequence:* when you make a choice, immediately state the alternative you rejected and why. Pre-empt the probe. Do this in every round.

## A typical four-session infrastructure loop

(Software Engineer, Infrastructure — 4 × 45 min, often split across two days, Webex + CoderPad.)

| # | Topic | Prep |
|---|---|---|
| A1 | Distributed-systems fundamentals (coding) + how you make decisions (design with a privacy/confidentiality frame) | [cpp-concurrency-and-implementations](../02-cpp/cpp-concurrency-and-implementations.md), [privacy-first-design](../07-system-design/privacy-first-design.md), [frameworks § decisions](../08-behavioral/frameworks.md) |
| A2 | OS & systems programming + behavioural: how you define scope, communicate and direct | [linux-internals](../04-linux-os/linux-internals.md), below, [frameworks § scope](../08-behavioral/frameworks.md) |
| A3 | Coding — complete a partial implementation; graded on steering and pivoting | below |
| A4 | Distributed-systems reliability: production outage, debugging distributed systems, competing demands, measuring success | [reliability-slos](../07-system-design/reliability-slos.md), [debugging-method](../06-troubleshooting/debugging-method.md), [frameworks § outage story](../08-behavioral/frameworks.md) |

**Every session pairs a technical axis with a judgement axis.** Nailing the technical half and treating the judgement half as small talk is how you get a "solid but not senior" write-up. If the team builds developer infrastructure, read [build-systems-design](../07-system-design/build-systems-design.md) — its vocabulary changes what "distributed systems" means in the loop.

## A1 — coding a distributed-systems primitive

Expect a distributed-systems *primitive*, not a LeetCode puzzle. High-probability candidates, all in `C7.2`:

- **Rate limiter** — token bucket, then "make it work across 20 servers." (The distributed extension is the real question. Naive `GET`-then-`SET` in Redis has a read-modify-write race; you need an atomic Lua script or `INCR` with expiry. Say the race out loud.)
- **Thread-safe cache with TTL and eviction** — then "what if two threads miss simultaneously?" (Single-flight / request coalescing.)
- **Bounded producer-consumer queue** — then shutdown semantics and lost wakeups.
- **Consistent hashing ring** with virtual nodes — then "a node dies, what moves?"
- **Idempotency layer** — dedup store with TTL; what's the correctness boundary of that TTL?
- **Vector clock / version vector merge** — detect concurrent updates, don't silently LWW.
- **Retry with exponential backoff and full jitter** — then "why jitter?" (Thundering herd — C3.12.)

**Discipline:** concurrency correctness is the graded axis, not cleverness. State your invariants. Name what the lock protects. Say what happens if the process dies at each line that mutates shared state. If you use a mutex, say why not a lock-free structure — and mean it.

## A2 — systems programming (not sysadmin) depth

Everything in `C1`, but with a **systems-programming** slant rather than a sysadmin one. Apple infrastructure teams work in C/C++/Go/Rust close to the metal, so expect:

- **Memory:** virtual memory, `mmap`, allocators (why does a general-purpose allocator fragment? when would you write a pool/arena allocator?), alignment, cache lines, **false sharing**, NUMA.
- **Concurrency primitives:** mutex vs spinlock (and *when* a spinlock is right — short critical section, no contention, no preemption risk), condition variables and lost wakeups, atomics and the C++ memory model (`relaxed`/`acquire`/`release`/`seq_cst`), why double-checked locking was broken before C++11, RCU as a concept.
- **Processes:** fork/exec, signals and **async-signal-safety** (why you can't `malloc` in a signal handler), `pthread_atfork` hazards.
- **I/O models:** blocking, non-blocking, `select`/`poll`/`epoll` (level vs edge triggered — and why edge-triggered requires draining to `EAGAIN`), `io_uring`, zero-copy (`sendfile`, `splice`).
- **Containers vs VMs**, mechanically (C1.24) — this is a common senior-level probe, along with "what happens to your isolation model under memory pressure."
- **Build & linking** (Developer Foundations is a build/tooling domain — this is likely their bread and butter): compilation units, static vs dynamic linking, symbol resolution order, `LD_PRELOAD`, ODR violations, why incremental builds break, hermetic/reproducible builds, content-addressed caching, and **why a build cache needs the full input closure as the key** (compiler version, flags, env, headers — miss one and you get silent wrong-cache hits).
- **Debugging tooling:** gdb/lldb, core dumps, ASan/TSan/UBSan and what each catches, `perf` and flamegraphs, `strace`/`ltrace`, eBPF at a conceptual level.

**C++ specifics they may probe:** RAII and why it's the answer to almost everything; move semantics and when a move is actually a copy; `shared_ptr` cost (atomic refcount — a real contention source) vs `unique_ptr`; virtual dispatch cost and when devirtualisation happens; undefined behaviour you've actually hit; and how you'd find a data race in a codebase you didn't write (TSan, then narrowing by instrumentation).

## A3 — complete-the-implementation + pivoting

### 3.1 What this round actually is

*"A part to be completed, or a coding problem they have."* This is the round least like anything at Google, and the least practised by most candidates. You are handed **existing code** — possibly their real code — with a gap, and asked to fill it. Sometimes it's a bug to find. The graded axes are:

1. **Can you read code you didn't write, fast?**
2. **Do you respect the existing conventions and abstractions**, or bulldoze them?
3. **Do you find the edge cases the original author missed?**
4. **When the interviewer changes the requirement mid-problem, do you adapt cleanly or restart?**

### 3.2 The protocol for the first four minutes — rehearse it

Do **not** start typing. Do this, out loud:

1. **"Let me read this first."** Read the whole thing before touching anything. Silence for 60–90 seconds is correct here; announce that you're doing it so it doesn't read as a stall.
2. **Restate the contract.** "So this function takes X, and is supposed to return Y, and the invariant seems to be Z — is that right?"
3. **Name the conventions.** "I see you're using error codes rather than exceptions, and ownership is passed via unique_ptr — I'll match that."
4. **Ask the three questions:** What's the expected input scale? Is this called concurrently? What's the failure behaviour — throw, return an error, or best-effort?
5. **Say your plan before writing it.** "I'll handle the empty case first, then the main path, then I want to come back to the concurrency question."

### 3.3 Handling the pivot (this is explicitly what they're testing)

Mid-problem the interviewer will change something: add a constraint, scale it up 1000×, make it concurrent, remove an assumption. The correct response:

> "Okay — that changes [specific thing]. My current approach assumed [assumption], which no longer holds because [reason]. I see two options: I can extend what's here by [X], which keeps the existing structure but costs [Y]; or I can restructure around [Z], which is cleaner but throws away what we've written. Given we have 20 minutes, I'd lean toward the first — want me to go that way?"

**Three things that response does:** names the invalidated assumption (shows you understood your own design), offers options with costs (shows judgement), and checks in on time budget (shows awareness). Do not silently rewrite. Do not defend the old approach out of sunk cost.

### 3.4 Drills — set these up before the loop

Real practice for this is different from LeetCode. Do these:

1. **Take an unfamiliar open-source file** (300–600 lines, C++ or Go) and give yourself 5 minutes to explain what it does and where you'd add a feature. Repeat 6–8 times with different codebases. This is the actual skill.
2. **Deliberately half-finish your own C7.2 implementations**, leave them for 48 hours, then complete them cold.
3. **Bug-hunt drills:** take a correct implementation, introduce one subtle bug (off-by-one, missed lock, unhandled error path, iterator invalidation), leave it a day, find it. Time yourself.
4. **Constraint-flip drills:** solve a problem, then have the constraint changed (single→multi threaded, in-memory→doesn't fit, exact→approximate) and adapt without restarting.

### 3.5 The failure modes here

| Failure | Fix |
|---|---|
| Typing before reading | Announce the read. 90 seconds of silence is professional |
| Ignoring existing style | Match error handling, naming, ownership conventions explicitly and say you're doing it |
| Restarting on every pivot | Extend first, restructure only if extension is genuinely worse, and say which you chose and why |
| Not finding the edge cases | Before declaring done: empty, single element, max size, concurrent, failure mid-operation, unicode/encoding if strings |
| Silent work | Apple withholds feedback deliberately; if you go quiet you give them nothing to score |
| Losing time to a language dispute | Ask at minute zero: "any language preference?" |

### Technique notes

- **Narrate the pivot.** When a constraint changes, say the words: *"That breaks my assumption that inputs fit in memory — I'm switching from the in-memory map to a spill-to-disk approach, which costs me X."* Silent pivots read as flailing.
- **State assumptions as assumptions when you make them**, so that when they get invalidated you have something to point back at. This is what makes a pivot look like engineering instead of correction.
- **Name the trade you're taking, every time.** "I'm choosing the O(n log n) approach because the constant factor on the O(n) one requires a hash and I don't trust the distribution here."
- **When handed partial code, read it out loud first.** Say what you think it does and what the contract is *before* editing. Getting corrected at that point is cheap; getting corrected after twenty minutes of building on a misread is not.
- **Ask what's fixed.** With a partial implementation, whether you may change the existing signatures is a real question and asking it scores.

## A4 — measuring success at Apple

Use C4.9 — four levels: user-visible SLIs, guardrail metrics that must not regress, operational metrics (pages, time-to-mitigate, toil), and the counterfactual (how would we know it was us — staged rollout, holdback, A/B). Then: **what would make us roll this back, and by when do we decide.**

Add the Apple-specific fifth: **cost and device impact**. On a consumer-device company, "success" includes battery, network chatter, and storage on the client. A design that improves server-side p99 by shipping more data to devices is not a success at Apple. Say that explicitly — it's a strong signal of understanding their constraints.

## Apple-only prep checklist

- Privacy toolkit — memorise the trade-off column, not just the names
- "How I make decisions" statement — written and rehearsed
- "How I define scope" statement — written and rehearsed, with two examples
- Complete-the-code drills — at least 6 unfamiliar-codebase reps
- Outage narrative — timed at 4 minutes and at 90 seconds
- "Why Apple" — 45 seconds, something only you would say
- Two "disagreed and then committed" stories

## One-page card

**Everywhere:** privacy in requirements, not the conclusion. State the alternative you rejected *before* they ask. Keep talking — they withhold feedback on purpose. Ask about language preference at minute zero.

**A1** — Distributed primitive, thread-safe, invariants stated. Then design with the question *"what would we prefer never to be able to read?"* Have the decision-making method ready.

**A2** — Systems programming depth (memory model, concurrency, linking, build systems), not sysadmin depth. Then: scope = goal → must/severable/explicitly-out → smallest real signal → checkpoint before sunk cost. Written first, verbal second.

**A3** — Read for 90 seconds before typing, and say you're doing it. Restate the contract. Match their conventions. On a pivot: name the invalidated assumption, offer two options with costs, check the time budget.

**A4** — Outage told like a system design: system → symptom → detection → blast radius → mitigation → causes (plural) → permanent fix → postmortem → detection gap. Competing demands = error budgets + toil cap, agreed in advance. Success = SLI + guardrails + operational + counterfactual + device/cost impact.

## Typical housekeeping

No AI assistance during the process. No screenshots, photos, or video. Don't discuss a previous employer's confidential material — keep internal system names and specific capacity numbers vague.
