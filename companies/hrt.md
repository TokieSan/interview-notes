# Hudson River Trading (HRT) — SWE / SRE / Systems

## 1. The single most important framing

HRT **interviews everyone as a software engineer regardless of the role**. There is no separate "SRE track" or "ops track." A SWE-SRE framing on a req means they want to see **coding depth and operational/systems thinking in the same person**. A production-engineering / SRE background is close to the profile they screen for — your job in the room is to make that legible.

The official company line, from HRT's own engineering blog, is worth internalizing because interviewers repeat it almost verbatim:

> *"Our ideal would be that a strong programmer from a competitive firm would ace our technical interview with no studying. We want engineers with strong fundamentals but stay away from obscura and 'tricks.'"*
> — Joe Smith, *Engineering and Interviewing at HRT* ([source](https://www.hudsonrivertrading.com/hrtbeat/engineering-and-interviewing-at-hrt/))

Translation: they are **not** primarily testing whether you can pull a clever LeetCode-hard insight out of thin air. They are testing whether you *actually understand what you use* — how a vtable is laid out, why a Python list resizes the way it does, what a page fault costs. The most common failure mode they cite is **"a shallow understanding of fundamentals."**

## Where systems roles sit in HRT engineering

HRT divides engineering along three dimensions. Knowing the map helps you (a) talk fluently about where you'd fit and (b) ask good questions.

**Dimension 1 — Research & Development vs Trading Tech**
- **Trading Tech** = "today." Parsing exchange data, order entry, FPGA/ASIC acceleration, clearing, reconciliation. Biggest challenge: **latency**. ~70% C++ / 30% Python.
- **R&D** = "history." Storage, data centers, CPU/GPU clusters, scheduling/orchestration, ETL, research tooling. Biggest challenge: **scale**. ~70% Python / 30% C++.

**Dimension 2 — Acceleration vs Platform**
- **Acceleration** ("embedded") = engineers attached to a specific Algo (trading) team, doing iterative, customized work.
- **Platform** = products used across *all* of Algo, longer roadmaps. Note: HRT platform engineers still interact with traders daily — they're not put behind a wall.

**Dimension 3 — Language/level ladder** (low → high):
`Verilog → Low-Level C++ → C++ → C++/Python → Python → Systems Automation/Python → TypeScript/Python`

The *Experienced Software Engineer – Systems* req sits in **[Live trading and Research | Platform | Systems Automation/Python]** — "maintains the software platform used to provision, manage, and monitor HRT's servers and network equipment… primarily Python (with some Go), plus deploying and running high-reliability services." That's why that loop is **Python + C++ + Linux**, not heavy algorithmic gymnastics. HRT encourages rotating across all three dimensions over a career, but the advice is blunt: start where you have the fastest impact.

## The funnel

Reports vary by office/year and HRT was reportedly mid-revamp as of mid-2026, so treat this as the modal path, not a guarantee. Your recruiter's confirmation email is the **source of truth** if it conflicts with anything here.

| Stage | What it is | Notes |
|---|---|---|
| **0. CV / ATS filter** | Resume screen | They filter hard. |
| **1. Online Assessment** | CodeSignal (sometimes HackerRank). **3–4 problems, ~70 min** (some custom variants run 90–150 min). | LC easy→hard, sometimes a pandas/file-IO problem. Pass bar is high (~90%+ / near-full AC). See [coding-round-playbook](../01-coding/coding-round-playbook.md). |
| **2. Phone screen(s)** | 1–2 rounds, ~45–60 min each, via Zoom + CoderPad. | Mix of live coding, fundamentals grilling, and resume/project deep-dive. |
| **3. Onsite / "Superday"** | Typically 3–5 back-to-back rounds (virtual or in person; HRT prefers some in person). | Coding + debugging, technical design discussion, systems fundamentals, team fit. |

A reported Systems phone-screen format: **~30 min Python and C++** (live coding + fundamentals) then **~15 min Linux**. HRT was reportedly revamping its process in mid-2026 — your confirmation email is the source of truth.

## What they score

Pulled from HRT's recruiting blog ([Dani Ortiz, *How to Prepare for Your SWE Interview at HRT*](https://www.hudsonrivertrading.com/hrtbeat/how-to-prepare-for-your-software-engineer-interview-at-hrt/)) and Joe Smith's pitfalls list. There are essentially **two axes**, weighted roughly equally:

### Technical axis
1. **Programming skill** — idiomatic, modern, well-encapsulated, readable, *fast to write*. Multiple candidates failed onsites purely because they couldn't implement + debug quickly enough in the slot.
2. **Systems-level knowledge** — memory, I/O, process management, "what happens under the hood." This is the bar that separates HRT from a generic SWE screen.
3. **Problem-solving** — decompose an unfamiliar problem and make incremental progress.

### Human axis (HRT says they rate this "as much as" technical ability)
4. **Communication** — articulate your reasoning *and* read whether the interviewer is following. Many interviewers won't be experts in your domain; check in, supply context.
5. **Collaboration** — take hints well, stay open to a different approach.
6. **Teachability** — apply an idea introduced earlier in the interview to a later problem. This is a *huge* positive signal.

### The 7 documented ways candidates fail (Joe's list, in frequency order)
1. **Shallow fundamentals** — using tools without intuition for *why they exist / how they're implemented*. (Their examples: how compilers implement virtual classes; the memory-growth characteristic of Python lists. Both are in your prep pack.)
2. **Communication issues** — can't articulate or can't understand others.
3. **Listening/understanding** — diving in before understanding the problem; talking over the interviewer.
4. **Avoiding real work** — over-engineering instead of just writing the practical solution. HRT expects *everyone* to code, regardless of seniority ("we administer the same technical tests to candidates at all levels, even if you were the CTO of your last company").
5. **Arrogance** — "this is finance"; conduct yourself with grace.
6. **Indifference** — show some genuine engagement/pride in your work.
7. **Cheating** — using LLMs in early stages. Later stages are designed to catch it; flagged candidates are not re-invited.

## The five behaviours that win this loop

These come straight from candidate reports + recruiter guidance + HRT's own writing. Practice them, not just facts.

1. **Think out loud, continuously.** HRT *probes everything you say.* Treat each statement as something you'll be asked to defend. Walk through reasoning before jumping to an answer. (This also lets a junior/less-expert interviewer follow — several candidates failed because the interviewer "couldn't get a word in.")
2. **Understand before you code.** Restate the problem, confirm constraints, work a tiny example on paper. The #1 onsite complaint is "dove in before understanding."
3. **Have 2–3 concrete end-to-end performance/ownership stories ready.** This firm rewards specifics over generalities. Rehearse them with numbers (throughput, latency, % improvement, scale).
4. **When pushed past your edge, say so and reason anyway.** They assess *how you think under pressure*. One candidate explicitly "bombed" an OS round (hadn't taken the course) and it "was fine" — composure + honest reasoning matters more than the fact.
5. **Code fast and debug fast.** The slots are short and dense ("very rapid follow-ups"). Practice writing correct, idiomatic code at speed and finding your own bugs without flailing.

**On hostility:** several reports describe a brusque, "distant," or even under-prepared interviewer (one candidate said the interviewer had several C++ facts wrong). Don't let it rattle you and **don't correct them combatively** — that reads as arrogance. State your reasoning, cite the standard/version ("in C++17 this is guaranteed because…"), and let it stand. Calm > right.

## Calibrating depth

| Track      | Bar HRT sets                                                                                                                                                                                                                                          | Your prep emphasis                                                                                                                           |
| ---------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| **C++**    | "A fair bit of low-level knowledge — how computers work." This is where you must be **sharpest.**                                                                                                                                         | Memory model, RAII/smart pointers, move semantics, vtables, atomics/lock-free (conceptual), UB, object lifetime. → [02-cpp](../02-cpp/README.md)                  |
| **Python** | Systems knowledge expected (e.g. virtual memory) but fewer language specifics than C++. Lean into **production performance**, not trivia — though trivia (`with`, generators, decorators, dict internals) still appears and "catches candidates out." | Profiling (py-spy/cProfile), GIL in practice, memory leaks in long-running procs, native extensions, plus the language internals. → [03-python](../03-python/README.md) |
| **Linux**  | They go **fairly deep.** "Systems automation skills" is its own competency for your role.                                                                                                                                                             | process vs thread, virtual memory/page tables, mmap, strace/perf, diagnosing live CPU/memory, links, signals, networking. → [04-linux-os](../04-linux-os/linux-deep-dive-with-sources.md)         |

## A 7–10 day plan

- **Days 1–3 — C++ fundamentals (heaviest).** [cpp-fundamentals](../02-cpp/cpp-fundamentals.md) end to end. For each topic: explain it plainly, say *why it exists / how it's implemented*, write a 5-line demo from memory.
- **Days 3–5 — Linux/systems.** [linux-deep-dive-with-sources](../04-linux-os/linux-deep-dive-with-sources.md). Open a real Linux box and actually run `strace`, `perf top`, `vmstat`, `/proc/<pid>/maps`, `ss`, `tcpdump`.
- **Days 4–6 — Python.** [python-internals-and-performance](../03-python/python-internals-and-performance.md). Re-tell two perf stories in profiling terms (which tool, what you saw, what you changed).
- **Daily — coding warm-ups.** 2–3 timed implementation problems from [worked-problems](../01-coding/worked-problems.md), alternating C++ and Python, out loud. Prioritise implementation-heavy "build a game / simulator" problems and graphs/trees over esoteric DP.
- **Final 2 days — mocks + stories.** Full 45-min mock with someone interrupting you, using the drill sheets ([C++](../02-cpp/cpp-qa-drill.md), [Python](../03-python/python-qa-drill.md), [Linux](../04-linux-os/linux-qa-drill.md)).

## Logistics & polish

- **Default to C++** for coding when given the choice if it's your strongest — but say up front you're comfortable reading/writing Python too, since they may steer there for the systems role. If shown an unfamiliar language, they grade with your familiarity in mind; just pattern-match and narrate.
- **CoderPad / Zoom:** test your link *before* the slot — one report lost interview time waiting on a bad link, and it counted against them. Square camera, look at the camera not the screen, blurred/clean background.
- **Have questions ready** (Platform vs Acceleration day-to-day; how the Systems team interacts with Algo; what "high-reliability services" they own; rotation opportunities). Indifference is a documented failure mode; curiosity is a documented positive.
- **Don't use LLMs during any live/recorded stage.** Non-negotiable for them.

## Sources

- HRT blog — engineering topology & interview philosophy: https://www.hudsonrivertrading.com/hrtbeat/engineering-and-interviewing-at-hrt/
- HRT blog — SWE interview prep: https://www.hudsonrivertrading.com/hrtbeat/how-to-prepare-for-your-software-engineer-interview-at-hrt/
- Candidate experiences: jointaro, Glassdoor, Reddit (r/csMajors, r/cscareerquestionsEU), Blind, Teamblind.
- CodeSignal OA format corroboration: EverythingQuant, programhelp, Blind threads (2025–2026).
