# Citadel Securities — SRE

## Format

~1 hour on CoderPad: one algorithmic problem + targeted systems/C++ probing for experienced candidates. Core languages Python & C++. They dig into **memory management, the OS underneath your code, concurrency, and performance**. Not trick questions — they want your *process*: clarify → state complexity/edge cases up front → discuss trade-offs → say when you're stuck. Weak C++ is worse than good Python; they probe language semantics hard, so commit to C++ only where you're fluent.

## Practice questions

Pulled from current candidate reports and Citadel Securities' published interview guides. The **topic distribution they actually test** (per 2026 guide): DSA → Coding/Implementation → **Networking Fundamentals** → Problem-Solving Approach → **Linux/Bash scripting**. Networking + Linux are explicitly weighted for the SRE loop — they matter as much as the LeetCode.

**Systems / SRE questions:**

- "How would you troubleshoot **network latency spiking intermittently between two data centers**?" → [scenario-bank](../06-troubleshooting/scenario-bank.md)
- "Explain the impact of **CPU cache locality** on your code's performance." → [cache locality](../02-cpp/cpp-concurrency-and-implementations.md)
- "How do you handle **resource contention in a multi-threaded C++ environment**?" → [concurrency](../02-cpp/cpp-concurrency-and-implementations.md)
- "**Design a system for high-throughput messaging** between trading components." → [trading-systems-design](../07-system-design/trading-systems-design.md)
- "**TCP vs UDP in the context of HFT**" — know why trading uses UDP multicast + app-level reliability. → [networking-core](../05-networking/networking-core.md)
- "Explain how a **`virtual` function works under the hood**." → [cpp-fundamentals](../02-cpp/cpp-fundamentals.md)
- "How would you **optimize a data structure for cache-line utilization**?" → [cache locality](../02-cpp/cpp-concurrency-and-implementations.md)
- "How would you **simulate 50% packet loss** for testing?" → `tc qdisc add dev eth0 root netem loss 50%` (Linux traffic control). Diagnose loss with `netstat -s`, `ss`, `/proc/net/snmp`, `/proc/net/sockstat`.
- On-call scenarios: "A production server is slow — find the cause"; "load average high but CPU/mem look normal" (→ I/O wait, D-state procs); "disk full, app can't write logs"; "service isn't responding, troubleshoot." → [scenario-bank](../06-troubleshooting/scenario-bank.md)

**Coding questions:**

- **Best Time to Buy/Sell Stock** — optimize the O(n²) to O(n). A classic screen question; know it cold.
- Shortest path in a graph **with a custom/extra constraint** (they modify the standard problem — don't just recite Dijkstra).
- **Running top-K over a data stream** (heap + hash map). → [problem-lists](../01-coding/problem-lists.md)
- **Thread-safe circular (ring) buffer** — common implementation question. → [ring buffer](../02-cpp/cpp-concurrency-and-implementations.md)
- Explain **trade-offs between sorting algorithms in a low-latency context.**
- Autocomplete / prefix search that must respond **sub-millisecond** (trie + design).

**Behavioral questions:**

- "Tell me about a time you made a **critical mistake in production** — how did you resolve it?"
- "Describe a time you had to **defend a technical decision against a senior stakeholder**."
- "How do you handle **ambiguity** when requirements aren't fully defined?"
- "**Why Citadel Securities rather than a traditional tech company?**"

They will **cross-examine your CV line by line** — every project, its architecture, failure modes, why you chose each data structure. Re-read your resume beforehand. Don't overstate anything.

Earlier reports also list: heavy C++ debugging, OOP, OS trivia, deadlocks, concurrent vs multi-core, packet loss across network setups, "what do you know about C++14/17 — and what else?", how `std::unordered_map` works, atomics, implement a thread pool, implement `shared_ptr`, test a quicksort pivot. All are covered in [02-cpp](../02-cpp/README.md), [04-linux-os](../04-linux-os/README.md) and [05-networking](../05-networking/networking-core.md).

## "Why Citadel Securities?"

- One crisp answer to **"why us vs a big tech company"**: scale + latency where correctness has immediate financial consequence, ownership early, tight feedback loop between your code and real outcomes.
- Have 2–3 STAR stories: a hard debugging/reliability incident, a performance optimization (with _numbers_ — latency/throughput before→after), a system you owned end-to-end. Frame around measurable impact.

- Prepare questions about the team's on-call model, what breaks most, what the SRE ↔ trading-systems boundary looks like, and the biggest reliability challenge right now.
