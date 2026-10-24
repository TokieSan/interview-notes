# SRE / Infrastructure / Systems Interview Notes

Prep notes for **SRE, production-engineering, infrastructure and systems-SWE interviews**, organised by topic. They cover interview loops at Google (SRE-SE), Apple (Infrastructure), Hudson River Trading, Citadel Securities and Meta (Production Engineer), and are built on candidate reports, company prep guides and recruiter guidance.

The emphasis is on **mechanism → consequence**: not "what does `kill -9` do" but why that matters in production and how you'd notice it.

## Where to start

| If you're interviewing for… | Read first |
|---|---|
| **Any role** | [how-to-interview.md](how-to-interview.md) — delivery behaviours that move scores everywhere |
| **Google SRE** | [companies/google-sre.md](companies/google-sre.md) |
| **Apple infrastructure** | [companies/apple-infrastructure.md](companies/apple-infrastructure.md) |
| **HRT / trading-firm systems roles** | [companies/hrt.md](companies/hrt.md), [companies/citadel-securities.md](companies/citadel-securities.md) |
| **Meta Production Engineer** | [companies/meta-production-engineer.md](companies/meta-production-engineer.md) |
| **Generic SRE / infra loop** | The four "start here" files below, then the topic folders |

**The four highest-yield files:** [04-linux-os/linux-internals.md](04-linux-os/linux-internals.md) · [06-troubleshooting/debugging-method.md](06-troubleshooting/debugging-method.md) · [07-system-design/nalsd-design-round.md](07-system-design/nalsd-design-round.md) · [01-coding/sre-practical-problems.md](01-coding/sre-practical-problems.md)

## Topics

| Folder | Contents |
|---|---|
| [01-coding](01-coding/README.md) | Round protocol and checklists, SRE-style parsing/stream problems, DSA patterns, worked problems, curated problem lists, huge-file processing |
| [02-cpp](02-cpp/README.md) | Fundamentals with the "why", Q&A drills, concurrency, implementing thread pool / ring buffer / `shared_ptr` / LRU |
| [03-python](03-python/README.md) | Production performance (profiling, leaks, GIL) and internals that get asked |
| [04-linux-os](04-linux-os/README.md) | Linux internals, the internals-round question bank, drills, OS theory, command reference |
| [05-networking](05-networking/README.md) | Core networking for debugging, protocol reference, debugging commands |
| [06-troubleshooting](06-troubleshooting/README.md) | Structured debugging method, scenario bank, Linux performance playbook (USE, 60-second checklist) |
| [07-system-design](07-system-design/README.md) | NALSD with real numbers, distributed systems, reliability/SLOs, capacity numbers, privacy-first, build-systems and trading-systems design |
| [08-behavioral](08-behavioral/README.md) | Frameworks, question clusters, story template, an LLM story-scoring rubric, questions to ask |
| [companies](companies/) | Per-company format, what's graded, practice questions, and pointers into the topics |

## Cross-reference key

The core systems notes use section IDs so ideas can point at each other:

| ID | File |
|---|---|
| C1.x | [04-linux-os/linux-internals.md](04-linux-os/linux-internals.md) |
| C2.x | [05-networking/networking-core.md](05-networking/networking-core.md) |
| C3.x | [07-system-design/distributed-systems.md](07-system-design/distributed-systems.md) |
| C4.x | [07-system-design/reliability-slos.md](07-system-design/reliability-slos.md) |
| C5.x | [06-troubleshooting/debugging-method.md](06-troubleshooting/debugging-method.md) |
| C6.x | [07-system-design/capacity-numbers.md](07-system-design/capacity-numbers.md) |
| C7.x | [01-coding/sre-practical-problems.md](01-coding/sre-practical-problems.md) |

`G1/G2/G3` refer to Google's onsite Debugging / Systems Internals / Architectural Patterns rounds, and `A1–A4` to Apple's four infrastructure sessions (see the company pages).

## A few caveats

- **Practice questions are generic and representative**, drawn from public sources (Blind, Glassdoor, LeetCode Discuss, IGotAnOffer, Coditioning, company blogs) and company prep guidance — not leaked question banks. Formats change; your recruiter's confirmation email is the source of truth.
- Some files intentionally overlap (e.g. a concise drill sheet and a deep dive on the same topic). The README in each folder says which to read first.
- The deep reference files in 04/05/06 (OS theory, protocol reference, performance playbook) are older and more verbose. The newer, denser files take precedence where they disagree.
- Treat model answers as something to understand and re-say in your own words, not to memorise. Interviewers probe until you hit your edge.
