# Google — SRE (Systems Engineer)

## The loop

**Phone / virtual screen (SRE-SE):** two 45-min rounds over Google Meet — *Programming & Code Evaluation* (practical coding in a shared Google Doc, no IDE) and *Googleyness & Leadership* (behavioural + hypotheticals).

**Onsite (SRE-SE, ~2.5 h):**

| Round | Length | Old name | Prep |
|---|---|---|---|
| Debugging | 45 min | Troubleshooting | [debugging-method](../06-troubleshooting/debugging-method.md), [scenario-bank](../06-troubleshooting/scenario-bank.md) |
| Systems Internals | 45 min | Unix/Linux | [systems-internals-round](../04-linux-os/systems-internals-round.md), [linux-internals](../04-linux-os/linux-internals.md), [networking-core](../05-networking/networking-core.md) |
| Architectural Patterns | 60 min | NALSD | [nalsd-design-round](../07-system-design/nalsd-design-round.md), [capacity-numbers](../07-system-design/capacity-numbers.md), [distributed-systems](../07-system-design/distributed-systems.md), [reliability-slos](../07-system-design/reliability-slos.md) |

Typical onsite logistics: government photo ID; a Chromebook is provided and you work in blank Google Docs shared to your personal Google account (bring your phone for 2FA); **using AI tools = disqualification**. There may be no dedicated behavioural round on the onsite slate — the Googleyness signal is collected inside the technical rounds.

## What every interviewer scores

Every interviewer scores four attributes on a standard form:

| Attribute                           | What it means here                                                        | Where you win it                                     |
| ----------------------------------- | ------------------------------------------------------------------------- | ---------------------------------------------------- |
| **GCA** (General Cognitive Ability) | How you attack an ambiguous problem and adapt                             | G1 hypothesis discipline, G3 clarifying questions    |
| **RRK** (Role-Related Knowledge)    | Do you actually know production systems                                   | G2 mechanistic depth, G3 numbers                     |
| **Leadership**                      | "Emergent leadership" — stepping up when your skill is the one needed     | How you drive the conversation, not wait to be asked |
| **Googleyness**                     | Ambiguity tolerance, bias to action, collaboration, intellectual humility | Threaded through all three rounds                    |

Ratings run Poor / Mixed / Good / Excellent, then a hire recommendation. The hiring committee reads *the written record of what you said*, not a vibe. **This has a practical implication: what the interviewer types into the doc is your case file. Say things that are worth typing.** Crisp, named, quotable statements — "this is a metastable failure, so we need to shed below the original load to escape it" — survive transcription. Vague competence does not.

Google's candidate prep guidance tells you the four things they want in G3: **clarify the problem** (they call the prompts *deliberately underspecified*), **break down** the problem and go both high-level and deep, **identify and analyse tradeoffs** (they explicitly say they care that you understand implications, not that a tradeoff is "correct"), and **recovering from failure**.

### The five behaviours that move scores most

1. **Ask before solving.** Going straight to problem-solving is called out as a red flag.
2. **Think out loud continuously**, including while writing. The interviewer drops hints; you can only catch them if you're audible.
3. **State assumptions explicitly and check them.** "I'm assuming reads dominate 100:1 — is that right?"
4. **Brute force first, then iterate.** Google says this themselves. Don't stall hunting for the elegant answer.
5. **Admit what you don't know, then reason toward it.** Intellectual humility is a named value. "I haven't worked with X directly; from first principles I'd expect Y because Z — is that the shape of it?" scores better than bluffing and far better than silence.

## Why Debugging → Internals → Design, in that order

Google's own guidance for the design round asks you to cover *"recovering from failure — how do you detect and defend against failures?"* The design answer gets stronger if you explicitly reach back:

- In design, when you place a component, say what you'd monitor and what you'd check first if it broke. That is debugging-round thinking exported into design.
- In design, when you estimate a machine count, ground it in real numbers — page cache behaviour, syscall cost, TCP throughput. That is internals-round knowledge exported into design.

Interviewers do not confer, but the rubric rewards the same underlying trait in all three. Treat the afternoon as one continuous argument that you understand production, delivered in three formats.

## Googleyness, threaded through the technical rounds

There is often no dedicated behavioural round on the onsite, which means **the behavioural signal is collected inside the technical rounds**, plus likely 5 minutes at the start or end of one. Expect: *why Google, why SRE, tell me about a recent project, tell me about a time you had to pivot, a time you resolved a conflict, a time your work had positive impact on the team.*

Map these to your story bank. Google-specific framing notes:
- **Intellectual humility is a scored value.** When asked about failure, do not disguise it as a strength. Say what went wrong, what you got wrong specifically, what you changed.
- **Bias to action** — Google likes the person who mitigated first and understood second.
- **Emergent leadership** means influence without authority. Frame your best story as "nobody owned this, so I did," not "I was assigned it."
- **"Why Google / why SRE"** — have 45 seconds. Specific, not flattering. Something like: the SRE model's insistence that reliability is measured and budgeted rather than argued about, and Cloud specifically because the systems are large enough that the second-order effects — retry amplification, metastable states — are the interesting engineering. Tie it to something you actually did.

## Non-negotiables

1. **Practise coding in a bare Google Doc.** No syntax highlighting, no execution. Candidates lose points on layout and typos alone.
2. **Validate visibly.** Candidates get graded Lean Hire for *discussing* tests instead of writing them. Write the test case. Dry-run one input aloud. Every time.
3. **Restate the problem before solving.** SRE rounds frequently deliver the problem verbally with nothing pasted.
4. **Numbers, not adjectives.** "This gets expensive at scale" is a non-answer in design. "At 50k QPS × 2 KB that's 100 MB/s, ~8 Gbps with replication — one machine's NIC is the wall" is the answer.
5. **Ask clarifying questions before designing.** Google calls its design prompts *deliberately underspecified* and treats jumping straight to a solution as a red flag.

## Sample 3-week plan for the onsite

Assumes ~2.5 h/weekday, ~5 h/weekend day.

### Week 1 — Foundations, widest surface
| Day   | Focus                                                                     | File                                      |
| ----- | ------------------------------------------------------------------------- | ----------------------------------------- |
| Mon | Linux process model, fork/exec, signals, zombies, `ls -l` end-to-end      | `C1.1–C1.4`                               |
| Tue | Memory: virtual memory, page cache, OOM killer, swap, `mmap` vs `read`    | `C1.5–C1.8`                               |
| Wed | Filesystems: inodes, links, dentries, journaling, `df` vs `du` divergence | `C1.9–C1.12` |
| Thu | Networking L1: TCP handshake, states, TIME_WAIT, congestion, MTU/MSS      | `C2.1–C2.6`                               |
| Fri | Networking L2: DNS, TLS, HTTP/2 vs 3, load balancer types, proxies        | `C2.7–C2.12`                              |
| Sat | **Full USE-method pass** + 4 timed debugging scenarios out loud           | `C5` + `G1` bank                          |
| Sun | Distributed systems: CAP/PACELC, consistency models, consensus, quorums   | `C3.1–C3.8`                               |

### Week 2 — Depth + first Google rehearsals
| Day    | Focus                                                                   | File                  |
| ------ | ----------------------------------------------------------------------- | --------------------- |
| Mon | Replication, sharding, hot keys, rebalancing, idempotency, exactly-once | `C3.9–C3.14`          |
| Tue | SLI/SLO/error budgets, tail latency, queueing (Little's law, Erlang)    | `C4`                  |
| Wed | **Latency numbers + capacity math drills** — no calculator              | `C6` |
| Thu | NALSD framework, then Design a distributed log ingestion pipeline       | `G3` worked ex. 1     |
| Fri | Coding in a plain Google Doc: 3 problems, no execution, dry-run each    | `C7`                  |
| Sat | **Mock day**: G1 (45m) + G2 (45m) back to back, self-recorded           | `G1`, `G2`            |
| Sun | Review recording. Patch the 3 weakest gaps only.                        | —                     |

### Week 3 — Google-specific
| Day    | Focus                                                             | File                       |
| ------ | ----------------------------------------------------------------- | -------------------------- |
| Mon | G2 rapid-fire: 40 internals questions, ≤90s answers each          | `G2` bank                  |
| Tue | G3 worked ex. 2: Design a global metrics/monitoring pipeline      | `G3`                       |
| Wed | G1 scenarios ×5, hypothesis-tree discipline                       | `G1` bank |
| Thu | G3 worked ex. 3: Migrate a live datastore with no user impact     | `G3`                       |
| Fri | Behavioural stories pass; reliability narratives                  | [08-behavioral](../08-behavioral/README.md) |
| Sat | **Full Google mock**: G1+G2+G3 in sequence, 2h30 including breaks | all                        |
| Sun | Weak-spot repair + behavioural stories + Googleyness answers      | [08-behavioral](../08-behavioral/README.md) |

### Final 48h — Taper, do not cram
| Day | Focus |
|---|---|
| Mon | One light pass over `C6` (numbers) + `G3` framework card. Nothing new. Confirm route, ID, phone charged, personal Google account login tested. |
| Tue | 45 min max. Read the one-page cards at the end of each file. Sleep. |

*Key: `C1.1` etc. = sections of [linux-internals](../04-linux-os/linux-internals.md) (C1), [networking-core](../05-networking/networking-core.md) (C2), [distributed-systems](../07-system-design/distributed-systems.md) (C3), [reliability-slos](../07-system-design/reliability-slos.md) (C4), [debugging-method](../06-troubleshooting/debugging-method.md) (C5), [capacity-numbers](../07-system-design/capacity-numbers.md) (C6), [sre-practical-problems](../01-coding/sre-practical-problems.md) (C7). `G1/G2/G3` = the Debugging / Internals / Design rounds above.*

## Phone-screen specifics

- Coding: practical "scripting" problems are most likely (parse a log, count, aggregate, top-K) — see [sre-practical-problems](../01-coding/sre-practical-problems.md) and the [Google-style practice problems](../01-coding/problem-lists.md). ~20–35 lines of clean C++/Python per problem.
- G&L: 4–6 questions + follow-ups in 45 min, plus hypotheticals — see [08-behavioral](../08-behavioral/README.md). Interviewers are often deliberately neutral; don't read the room.
- If the interviewer is >10 min late, ping your recruiter or google-candidate-support@google.com.

## After the onsite

Feedback goes in within 2–3 days, then hiring committee, then team matching, then comp review. Expect weeks, not days. Don't read silence as signal. The questions are confidential — don't post them.

## One-page card

**Debugging (45m).** Impact → scope → timeline → *mitigate now?* → bisect the path → ranked hypotheses with killing evidence → verify cause not fix → detection gap. Say "mitigate before diagnose" in minute one.

**Internals (45m).** Mechanism → consequence → signal. Three sentences. Never lead with a command name. Expect "why" until you hit your edge; hitting it is fine, bluffing is not.

**Design (60m).** Announce the five questions: possible? better? feasible? resilient? better? Then: 8m clarify, 7m numbers, 7m one machine, 18m distribute, 12m resilience table, 8m trade-offs. Name the binding resource. Write in the doc.

**Everywhere.** Clarify before solving. Think out loud. State and check assumptions. Brute force then iterate. Admit the edge of your knowledge. Say quotable things — someone is typing.

*Sources: Google SRE Workbook ch. 12 (NALSD), Google candidate prep guidance, recruiter guidance, and 2025–2026 candidate reports (IGotAnOffer, Coditioning, Exponent, Blind).*
