# Reliability, SLOs & Measuring Success (C4)

> Section IDs like `C1.6` or `C3.12` are cross-references inside the core systems notes: C1 = [Linux internals](../04-linux-os/linux-internals.md), C2 = [networking](../05-networking/networking-core.md), C3 = [distributed systems](../07-system-design/distributed-systems.md), C4 = [reliability & SLOs](../07-system-design/reliability-slos.md), C5 = [debugging method](../06-troubleshooting/debugging-method.md), C6 = [numbers](../07-system-design/capacity-numbers.md), C7 = [coding patterns](../01-coding/sre-practical-problems.md).

**C4.1 — SLI / SLO / SLA / error budget.**
SLI = a measured ratio of good events to valid events. SLO = the target for that ratio over a window. SLA = the contractual consequence. Error budget = `1 − SLO`, spent by unreliability.
*Framing that lands:* the error budget converts reliability from an argument into an arithmetic. Budget remaining → ship. Budget exhausted → freeze features and spend on reliability. It resolves the dev/ops tension structurally rather than politically. **This is the answer to Apple's "competing demands" prompt** (A4).

**C4.2 — Picking good SLIs.**
Four golden signals: latency, traffic, errors, saturation. USE for resources: Utilisation, Saturation, Errors. RED for services: Rate, Errors, Duration.
*Rules:* measure as close to the user as possible (client-side or LB, not the backend); define "valid events" carefully (exclude bot traffic, include timeouts as errors); express latency SLIs as a *threshold ratio* — "99% of requests under 300ms" — not as "p99 < 300ms", because the former composes and aggregates correctly and the latter doesn't.

**C4.3 — Why averages lie, precisely.**
p50 tells you about the median user; nobody is the median user across ten requests. With 10 sequential backend calls each at p99 = 1s, the chance *no* call hits the tail is 0.99¹⁰ ≈ 90% — so ~10% of user requests hit a 1s+ call. Tail latency is amplified by fan-out.
*Mitigations:* hedged requests (send a second after p95, take the first response — costs ~5% extra load), tied requests, request cancellation, and reducing fan-out width.

**C4.4 — Little's Law and why it's the most useful formula you have.**
`L = λW` — concurrency = arrival rate × latency. So `required threads = QPS × latency`.
*Use it constantly:* 5,000 QPS at 40 ms mean latency needs 200 concurrent in-flight requests. If your pool is 100, you are the bottleneck and the queue grows without bound. Also inverts: if latency doubles under load and QPS is fixed, concurrency doubled — where did it go?

**C4.5 — Utilisation and the queueing wall.**
Waiting time scales roughly as `1/(1−ρ)`. At 50% utilisation, waiting ≈ service time. At 90%, ≈ 9×. At 99%, ≈ 99×.
*Consequence:* "we're only at 80% CPU" is not headroom, it's a cliff edge. This single relationship justifies most capacity decisions in G3; say it with the number.

**C4.6 — Toil, and the SRE bargain.**
Toil = manual, repetitive, automatable, tactical, no enduring value, scales linearly with service size. Google's convention caps it at ~50% of an SRE's time.
*Consequence:* if you're asked "how do you balance reliability work against feature work", the answer is error budgets (C4.1) plus a toil cap, not willpower.

**C4.7 — Blameless postmortems.**
Triggered by defined criteria (user-visible downtime over threshold, data loss, manual intervention required). Contents: timeline, impact quantified in SLI terms, root cause(s) — plural, always plural — what went well, what went badly, where we got lucky, and action items with owners and dates.
*Consequence:* blamelessness is not politeness; it's an information-extraction strategy. If people are punished, you stop hearing about near-misses, and near-misses are your cheapest data. **Apple asks "what process change prevented recurrence?" — have that answer**.

**C4.8 — Deployment safety.**
Canary (small % of traffic, compare SLIs against control, automatic rollback on regression), blue/green (instant switch, instant rollback, 2× resource cost), feature flags (decouple deploy from release, allow per-user rollout, but flag debt is real), progressive rollout by cell/zone.
*Consequence:* the metric that matters is not deploy frequency but **time to rollback**. If rollback takes an hour, your canary window is theatre.

**C4.9 — Measuring success at design time** — the direct answer to A4's prompt.
Four levels, and say them in this order: (1) **user-visible SLIs** — did the thing users care about get better, expressed as a ratio; (2) **guardrail metrics** — what must *not* regress (latency, cost per request, error rate on adjacent paths); (3) **operational metrics** — pages per week, time-to-mitigate, toil hours reclaimed; (4) **the counterfactual** — how would we know this was the cause? A/B, staged rollout by cell, or a holdback. Then: what would make us roll this back, and what's the decision deadline.

## "Competing demands" — reliability vs velocity

This is the reliability-vs-velocity question, and the correct answer is structural, not diplomatic:

> "I don't think it's resolvable by argument, which is why error budgets exist. You agree an SLO, the budget is one minus that, and unreliability spends it. Budget remaining means ship features. Budget exhausted means the next sprint is reliability work — and crucially that's agreed in advance, when nobody's under pressure, so it isn't a negotiation during an incident. It also protects the other direction: if we're at 99.99% against a 99.9% target, we're over-investing in reliability and should be shipping faster or reducing cost.
> The second half is toil. If more than about half of operational time is manual repetitive work, the team can't do reliability engineering at all, only reliability operations, and it degrades from there. So I'd track toil as a first-class metric alongside the budget."

## Debugging distributed systems — the design-level answer

They said "broader level, in terms of design." So they don't want a command list; they want your **methodology and what you'd build to make it possible**. Use C5.6 and add the design layer:

> "The hard part in a distributed system isn't finding the slow component, it's telling apart the component that's slow *because* of the fault from the one that's slow *in reaction* to it. Retries and queueing mean the loudest signal is usually the victim.
> So the design question is: what do I need to have built *in advance* to answer that? Three things. **Distributed tracing** with a correlation ID propagated through every hop, so I can find where latency is introduced rather than where it's observed. **Differential comparison** — I want shards, cells, or regions I can compare against each other, because in a system with no healthy control, every threshold is a guess. And **causality-preserving telemetry**: exemplars linking a metric spike to specific traces, and logs joined on the request ID.
> Then operationally: bound the blast radius before root-causing, and work backwards along the causal chain rather than toward the biggest number."

Then name specific mechanisms: retry amplification (C3.12), metastable failure and why you sometimes have to shed *below* the original load to escape it (C3.14), tail amplification with fan-out (C4.3), hot shards (C3.10), and clock-skew-induced ordering bugs (C3.6).

## Prompts to run cold

- Walk me through a production outage you owned, end to end.
- A service's p99 degrades over a week with no deploys. Design-level: how do you find it?
- How do you decide what to alert on, and what *not* to alert on? *(Alert on symptoms, not causes; every page must be actionable and novel; anything else is a ticket or a dashboard.)*
- Your service depends on six others. How do you set your SLO? *(Multiply their availabilities as a floor, then decide where you need to be resilient to them rather than dependent on them.)*
- How would you design a system so that a single bad deploy can't take down all regions?
- You have three weeks and two competing asks: a reliability fix and a launch-blocking feature. Decide.
- How do you measure whether an infrastructure/tooling change was worth it? *(Especially relevant to Developer Foundations: developer-facing infra success = build times, flake rate, time-to-first-green, and adoption — not uptime alone.)*

## One-line summary

**Reliability:** SLI = good/valid · error budget makes it arithmetic · L = λW · queueing wall at 1/(1−ρ) · tail amplifies with fan-out · measure at the user.
