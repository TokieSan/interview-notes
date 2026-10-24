# Meta — Production Engineer (PE / Network PE)

## The loop

From public candidate reports:

- **Coding** — practical, usually in your strongest language: file/log parsing, string manipulation, simulations. Typical: generate a random Minesweeper board; palindrome with extra constraints; a file-parsing question; "Dinosaur" (join two CSVs and compute speeds); Goat Latin; Battleship; split an array into two equal-sum halves; Bulb Switcher.
- **Systems** — two parts: **OS knowledge** (processes & threads, memory management, file systems, I/O) and a **troubleshooting scenario** (server not working, process eating memory, slow server — "going into many kernel details").
- **System design** — scaling basics (horizontal vs vertical, caching, load balancing) for some screens.
- **Behavioural** — Meta values; see [08-behavioral](../08-behavioral/README.md).

### Typical systems questions

- What happens when you press Ctrl+C, in terms of signals / IPC?
- How does `ls` work (inodes, globbing, etc.)?
- What exactly happens when you `tail` a file, in terms of buffers?
- Give as detailed an explanation as you can of an operating system.
- What would you do if you saw request failures from a web server?
- Internet: how does a request get to an app? How would you build one?
- `tar` details; the `w` command (who is logged in).
- Theoretical troubleshooting of a slow server, going deep into kernel details.

## Advice from someone who passed the loop

> The systems interview is split into (1) OS knowledge and (2) a troubleshooting scenario.
>
> For OS knowledge you need processes and threads, memory management, file systems, and I/O. *Modern Operating Systems* has a chapter on each; reading only the processes and memory chapters was enough in my case. If you already had a good OS course, the book's Linux case-study chapter (how the theory is implemented in Linux) may be enough on its own.
>
> For troubleshooting you get a scenario (server not working, process taking too much memory, …) and are asked what you'd do. Read the [Linux Performance Analysis in 60,000 ms](https://netflixtechblog.com/linux-performance-analysis-in-60-000-milliseconds-accc10403c55) and [Netflix at Velocity 2015](https://netflixtechblog.com/netflix-at-velocity-2015-linux-performance-tools-51964ddb81cf) write-ups, watch the [Linux performance tools talk](https://www.youtube.com/watch?v=FJW8nGV4jxY), and read the [USE method](https://www.brendangregg.com/usemethod.html).

Another fast-path suggestion (2–3 days per topic): Blind 75 minus DP/graphs/hards for coding; the [system-design-primer](https://github.com/donnemartin/system-design-primer) for design; Linux internals for systems — e.g. *The Linux Systems Interview: Understanding Linux Internals and Troubleshooting*.

## Suggested reading order in this repo

1. [os-theory-reference § keyword map](../04-linux-os/os-theory-reference.md) — quick refresh of OS vocabulary.
2. [linux-internals](../04-linux-os/linux-internals.md) — the interview-shaped version of OS knowledge.
3. [linux-performance-playbook](../06-troubleshooting/linux-performance-playbook.md) — 60-second checklist → USE → decision tree → symptom/fix tables.
4. [debugging-method](../06-troubleshooting/debugging-method.md) and [scenario-bank](../06-troubleshooting/scenario-bank.md) — practise narrating scenarios.
5. [networking-core](../05-networking/networking-core.md); for Network PE add [protocols-reference](../05-networking/protocols-reference.md) (BGP, OSPF, DHCP, ARP) and [network-debugging-commands](../05-networking/network-debugging-commands.md).
6. [linux-qa-bank](../04-linux-os/linux-qa-bank.md) — final self-quiz.
7. [os-theory-reference](../04-linux-os/os-theory-reference.md) — the deep version, for "explain an OS in as much detail as you can".
8. Coding: [sre-practical-problems](../01-coding/sre-practical-problems.md) and [large-file-processing](../01-coding/large-file-processing.md).

## Practice resources

- [SadServers](https://sadservers.com/) — live broken-Linux-box scenarios; good daily practice. Their [Linux server review](https://docs.sadservers.com/docs/troubleshooting/linux-server-review/) is a solid checklist.
- [alex/what-happens-when](https://github.com/alex/what-happens-when) — the "type a URL and press Enter" question, end to end.
- [Coditioning: Meta Production Engineer interview guide](https://www.coditioning.com/blog/8/meta-production-engineer-interview-guide-deeper-dive).
- [Applied Programming OS notes](https://applied-programming.github.io/Operating-Systems-Notes/) — good last-day OS review.
- Prepfully for PE-specific topics; interviewing.io for coding mocks.
- Meta's published values (move fast, build awesome things, be direct and respect your colleagues, focus on long-term impact, live in the future, "Meta, Metamates, me") — map your behavioural stories to them.

A daily routine that worked: one systems Pomodoro, one coding + SadServers Pomodoro.
