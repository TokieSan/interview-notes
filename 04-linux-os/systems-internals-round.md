# The Systems-Internals Round

How a rapid-fire Linux internals round runs (Google SRE "Systems Internals", formerly "Unix/Linux"; similar rounds exist at HRT, Citadel, Meta PE) and a 40-question bank mapped to [linux-internals.md](linux-internals.md) and [../05-networking/networking-core.md](../05-networking/networking-core.md). Recruiter-stated syllabus: kernels, libraries, system calls, memory management, permissions, file systems, client-server protocols, and the shell.

### 2.1 Format

Rapid-fire and adaptive. An initial question, then **"why" follow-ups until you hit the edge of your knowledge** — that's the design of the round, and hitting the edge is expected. What's graded is whether you *knew* you'd hit it and reasoned forward honestly. Some candidates report being asked to write small utilities (implement `tail` for a huge file, implement `find`) rather than answer purely verbally.

### 2.2 The answer shape — use this every single time

> **Mechanism** (what the kernel/shell/libc actually does) → **Consequence** (what breaks or gets slow in production) → **Signal** (how you'd observe it).

Three sentences. "An inode holds file metadata and block pointers but not the filename, which lives in the directory entry. *That's why* you can delete a file another process has open — the dirent goes but the inode survives until the last fd closes, so the space isn't reclaimed. *You see it as* `df` and `du` disagreeing, and you find it with `lsof +L1`." That answer is worth typing into the feedback doc. "An inode stores metadata" is not.

### 2.3 Rapid-fire bank — target ≤90 seconds each, using the three-sentence shape

**Processes & signals** — C1.1–C1.8
1. Process vs thread in Linux, mechanically.
2. What does `fork()` copy?
3. What happens between `fork()` and the new program running?
4. What is a zombie process? How do you get rid of one?
5. What's the default signal from `kill`? *(SIGTERM/15)* Difference from `kill -9`?
6. What is a signal and how does the kernel deliver it?
7. What are the process states, and which count toward load average?
8. How does the scheduler decide what runs next?
9. A process ignores SIGKILL. Explain.
10. What's the cost of a context switch, and where does it actually come from?

**Memory** — C1.9–C1.13
11. Walk a virtual address to physical.
12. Minor vs major page fault. Which one do you alert on?
13. What is the swap area? Should you enable swap on a server?
14. What is the page cache and why is "free memory is low" usually fine?
15. `mmap` vs `read` — when does each win?
16. How does the OOM killer choose? How is it different inside a container?
17. How would you tell a memory leak from fragmentation from a cache growing?
18. How does a garbage collector work? *(cover mark-sweep, generational hypothesis, and the operational point: GC pauses show up as p99 latency, not as CPU)*

**Filesystems** — C1.14–C1.17
19. What's an inode and what does it store — and not store?
20. Hard link vs soft link. Why can't you hard-link a directory?
21. `df` full, `du` not. Explain and resolve.
22. What does journaling protect you from, and what doesn't it protect you from?
23. Does a successful `write()` mean the data is on disk?
24. What is the function of inodes when a filesystem runs out of them?

**Syscalls, shell, permissions** — C1.18–C1.24
25. What happens on a kernel level when you type `ls -l`? *(the full walk, C1.18)*
26. What happens if you run `ls *`? *(the shell expands it, C1.19)*
27. How does the shell parse and execute `rm foo*`?
28. Which syscall returns non-zero on success? *(C1.21)*
29. How does a syscall actually transition to the kernel, and what does it cost?
30. Explain setuid, setgid, and the sticky bit.
31. You're root and you get "permission denied". Why?
32. How do you load/unload a kernel module?
33. Walk the boot sequence.
34. What is a container, mechanically? Namespaces vs cgroups vs seccomp.
35. Static vs dynamic linking — and what `LD_PRELOAD` lets you do.

**Client-server protocols** — C2 (the recruiter listed this under Systems Internals, so expect it here as well as G1)
36. TCP vs UDP. When would you actually choose UDP?
37. Walk the TCP handshake and the two server-side queues.
38. What is TIME_WAIT and whose problem is it?
39. Walk a DNS resolution end to end.
40. What happens when you type a URL and press enter? *(the full stack — practise a 3-minute version and a 30-second version)*

### 2.4 Small-utility prompts (be able to write these in a Doc)
- Implement `tail -n K` for a file too large to fit in memory.
- Implement `find` — recurse directories, match a pattern.
- Write a Bash + cron setup that runs on the last Friday of every month. *(the trick: cron has no "last Friday", so run every Friday and guard with `[ $(date -d "+7 days" +\%m) != $(date +\%m) ]`)*
- Implement a shell's background-process and pipe facilities in C. *(`fork`/`pipe`/`dup2`/`execvp`/`waitpid`)*
- Script to find duplicate files. *(hash by size first, then content — say why)*

### 2.5 The G2 failure mode

The trap is **fluency without mechanism**. You use these systems daily, so the commands come out fast — and fast command recall reads as *superficial* to this rubric. Deliberately slow down and lead with the kernel-level "why". If you catch yourself saying a command name in the first sentence, restart the answer.
