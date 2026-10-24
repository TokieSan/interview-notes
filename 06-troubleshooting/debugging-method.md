# Structured Debugging (C5)

> Section IDs like `C1.6` or `C3.12` are cross-references inside the core systems notes: C1 = [Linux internals](../04-linux-os/linux-internals.md), C2 = [networking](../05-networking/networking-core.md), C3 = [distributed systems](../07-system-design/distributed-systems.md), C4 = [reliability & SLOs](../07-system-design/reliability-slos.md), C5 = [debugging method](../06-troubleshooting/debugging-method.md), C6 = [numbers](../07-system-design/capacity-numbers.md), C7 = [coding patterns](../01-coding/sre-practical-problems.md).

Google's rubric here is explicit: they want a **logical and structured approach**, and the round is a *dialogue* — the interviewer feeds you partial signals as you ask for them. Answers that dump commands score badly; answers that state a hypothesis and name the evidence that would kill it score well.

### C5.1 — The loop, and say it out loud at the start

1. **Characterise.** What exactly is broken, for whom, since when? Scope: all users or some? All regions? All request types? Did it start suddenly or degrade?
2. **Bound the blast radius.** This is a *reliability* step, not a diagnostic one, and doing it before root-causing is the senior move. Can we mitigate now — roll back, shed load, fail over — and diagnose after?
3. **Timeline.** What changed? Deploys, config pushes, flag flips, cert rotations, dependency releases, traffic events, expiry dates. Correlate onset against the change log.
4. **Bisect the request path.** Client → DNS → LB → service → dependencies → datastore. At each hop: does the request arrive, and does it leave? Halving the search space per step is the whole game.
5. **Hypotheses, ranked.** State 2–4. For each: what evidence confirms it, what evidence kills it, and how expensive is the check. Do the cheap discriminating checks first.
6. **Verify the fix, then verify the *cause*.** A fix that works is not a diagnosis. Ask "why did this only start now?" — if you can't answer, you haven't finished.
7. **Prevent.** Detection gap (why didn't monitoring catch it?), and recurrence.

### C5.2 — The opening questions that buy you the most

- What's the SLI impact — error ratio, latency, or availability? Which one moved first?
- Is it *all* users or a *subset*? A subset means routing, sharding, a canary, a region, or a client version.
- Does it fail fast or slow? Fast → rejection, DNS, auth, connection refused. Slow → saturation, retries, timeouts, lock contention.
- Does it reproduce from a different network path / by IP instead of name / with a different client?
- What changed in the last 24h, including things nobody counts as changes (certs, TTL expiries, quota resets, cron boundaries, month rollover).

### C5.3 — USE + RED as a checklist per box

**USE** per resource: Utilisation, Saturation, Errors. CPU (`top`, `mpstat`, and check *steal*), memory (`free -m`, `available`, `vmstat` si/so, `pgmajfault`), disk (`iostat -x`, `%util` and `await`), network (`sar -n DEV`, `ip -s link` for drops/errors), and the ones people forget: **fd limits** (`/proc/PID/limits`, `lsof | wc -l`), **PID limits**, **conntrack table** (`nf_conntrack_count` vs `max` — silent drops when full), **ephemeral ports**, **thread pool depth**.
**RED** per service: Rate, Errors, Duration — plus queue depth and dependency latency.

### C5.4 — The 60-second first-look sequence

`uptime` (load vs core count) → `dmesg -T | tail` (OOM kills, link flaps, I/O errors, conntrack full) → `vmstat 1 5` (r, b, si/so, us/sy/wa/st) → `mpstat -P ALL 1` (one hot core = single-threaded bottleneck or interrupt affinity) → `iostat -xz 1` → `free -m` → `ss -s` and `ss -lnt` (queues) → `top` sorted by RSS then by CPU.
*Say what each one rules out as you go.* That narration is the thing being graded.

### C5.5 — Signature table (memorise the left column, derive the right)

| Symptom | First hypotheses |
|---|---|
| High load, low CPU | `D`-state processes — I/O or NFS hang (C1.6) |
| High `sy` time | Syscall storm, lock contention in kernel, interrupt flood |
| High `st` (steal) | Noisy neighbour / hypervisor oversubscription |
| Latency fine at p50, terrible at p99 | Queueing, GC pauses, tail amplification (C4.3), hot shard (C3.10) |
| Errors only for *some* users | Sharding, canary, region, client version, hot key |
| Works by IP, fails by name | DNS (C2.7) |
| Small requests OK, large hang | PMTU black hole (C2.5) |
| Connection refused | Nothing listening, or accept queue overflow (C2.2) |
| Connection times out | Firewall dropping (not rejecting), routing, or SYN backlog |
| Intermittent TLS failures | Clock skew, partial chain deployment, SNI mismatch, expiring cert |
| Disk full but `du` disagrees | Deleted-but-open file, or inode exhaustion (C1.16) |
| Memory climbs then process dies | Leak, or cgroup limit + OOM (C1.12) |
| Everything slow after a partial recovery | Retry storm / metastable failure (C3.12, C3.14) |
| Fails at exactly the same time daily | Cron, log rotation, batch job, token refresh, cert renewal |

### C5.6 — Distributed-specific debugging (this is Apple's A4 axis)

Single-box tools stop working when the fault is *emergent*. Reach for: distributed tracing (find the span where latency is *introduced*, not just where it's observed), correlation IDs propagated through every hop, exemplar-linked metrics, structured logs joined on request ID, and **comparing a healthy shard against a sick one** — differential diagnosis beats absolute thresholds.
*Say this:* in a distributed system the hard part is not finding the slow component, it's establishing whether the component is slow *because* of the fault or *in reaction to* it. Retry amplification and queueing make the loudest signal usually the victim, not the cause. Work backwards along the causality chain, not toward the biggest number.

## One-line summary

**Debug:** characterise → bound blast radius → timeline → bisect the path → ranked hypotheses with killing evidence → verify cause not just fix → close the detection gap.

---

# Running a live debugging round

The format below is Google's SRE "Debugging" round (formerly Troubleshooting), but Meta PE, HRT and Citadel run the same shape: a vague symptom, deliberately incomplete information, and an interviewer who feeds you signals as you ask for them.

### 1.1 Format

You get a scenario — a service is broken — with **deliberately incomplete information**. The interviewer plays the environment: you ask for a signal, they give it to you. It is a dialogue, and pacing is graded. Google's stated examples are App Engine-flavoured production outages. The canonical published one is the *Shakespeare service* from the SRE book: a black-box probe reports search failures for a specific query for the past five minutes — what do you do?

### 1.2 The opening 90 seconds (rehearse this until it's automatic)

> "Before I start guessing, let me establish scope and see if we can mitigate.
> First — **what's the user impact and how is it measured?** Is this an availability drop, an error-rate rise, or latency? Which moved first?
> Second — **is it total or partial?** All users, all regions, all request types, or a subset?
> Third — **when did it start, and what changed around then?** Deploys, config pushes, flag flips, cert rotations, dependency releases, traffic events.
> Fourth — **is there a fast mitigation available right now** — rollback, drain a region, shed load — that I should do before I understand the cause? I'd rather stop the bleeding and diagnose in parallel.
> Then I'll bisect the request path and work with ranked hypotheses."

Saying "mitigate before diagnose" out loud, at minute one, is the single strongest opening available to you. It's what separates an SRE from a debugger, and it is exactly what the rubric means by production judgement.

### 1.3 The bisection ladder (your default structure)

Client → DNS → LB/edge → service instances → dependencies → datastore → underlying infra.
At each hop ask two questions only: **does the request arrive here, and does it leave here?** That halves the space each time. Say which hop you're testing and what result would eliminate which half.

### 1.4 Hypothesis discipline — the format that scores

Never list causes. Always state them in this frame:

> "Three hypotheses, ranked. **One**, the accept queue is overflowing on the backends — cheap to check with `ss -lnt`, and if `Recv-Q` is at the backlog limit that confirms it and rules out the app being slow. **Two**, a dependency is timing out and we're seeing retry amplification — I'd check the dependency's own latency SLI and our retry rate; if retries are flat this is dead. **Three**, it's a bad canary — I'd check whether errors correlate with build version. That's the cheapest of the three, so actually let me do it first."

Note the last move: reordering by cost after ranking by likelihood. Do that visibly at least once.

### 1.5 Scenario bank — work these out loud, 8 min each

**Networking / connectivity**
1. Packets reach some parts of the network and not others.
2. You can't SSH into a remote machine. Walk the layers.
3. Small requests succeed, large ones hang. *(PMTU — C2.5)*
4. Service works by IP, fails by hostname, intermittently, only from one cluster.
5. Latency to one region jumped 40 ms overnight with no deploy.
6. TLS handshakes fail for ~5% of clients.

**Host / kernel**
7. Load average is 60 on a 16-core box but CPU is 4% idle-heavy. *(D-state, C1.6)*
8. A system is running out of PIDs. Detect and stop it. *(C1.4)*
9. `df` says the disk is full, `du` says it's half empty. *(C1.16)*
10. A process is killed nightly at 02:00 with no logs. *(OOM + cron/log rotation)*
11. One CPU core is pinned at 100%, the rest idle. *(single-threaded hot path, or IRQ affinity)*
12. Requests are slow only on newly-started instances. *(cold cache, JIT warmup, connection pool ramp, DNS TTL)*

**Distributed / service-level**
13. Debug a slow web server, any tool available. *(deliberately open)*
14. p50 is fine, p99 is 40× worse, and it's got steadily worse over a week. *(hot shard / GC / tail amplification)*
15. After a dependency recovered, the service stayed broken. *(metastable failure — C3.14)*
16. Error rate is 100% for exactly 3% of users. *(shard, hot key, client version, feature flag)*
17. A batch job that took 20 min now takes 5 hours; nothing changed in the code.
18. Black-box probe fails for one specific query while everything else is green. *(the Shakespeare scenario — think index shard, not global outage)*

### 1.6 The five G1 failure modes

| Failure                   | What it looks like                        | Fix                                                                            |
| ------------------------- | ----------------------------------------- | ------------------------------------------------------------------------------ |
| Command dumping           | "I'd run top, then iostat, then netstat…" | Name the *hypothesis* each command tests                                       |
| Tunnel vision             | Chasing hypothesis 1 for 30 minutes       | Timebox each check; announce when you're abandoning a line                     |
| Never mitigating          | Perfect root cause, users down for 45 min | Bring up mitigation in minute one and again mid-round                          |
| Stopping at the fix       | "Restarting it fixed it"                  | Always ask *why now?* and *why didn't monitoring catch it?*                    |
| Not using the interviewer | Monologue                                 | Ask for signals explicitly: "what does the dependency's error rate look like?" |

### 1.7 Closing the round

Reserve two minutes. "To summarise: root cause was X; we mitigated with Y in Z minutes; the detection gap was that our SLI measured the backend not the client, so I'd add a probe at the edge; and the recurrence fix is A, owned before the next release." That paragraph, delivered unprompted, is a Googleyness data point as much as a technical one.
