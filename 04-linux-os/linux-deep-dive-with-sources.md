# Linux / Systems Deep Dive (with sources)

Topic-by-topic explanations with man-page and Brendan Gregg links, plus trading-firm-style questions and model answers. Overlaps [linux-internals.md](linux-internals.md) on purpose — this one has the sources and the hands-on warm-ups.

## A. The topic map

**Recruiter-flagged deep-dive list:**
1. Process vs thread at the OS level
2. Virtual memory & page tables
3. `mmap` vs standard file I/O
4. `strace` / `perf` in practice
5. Diagnosing high CPU / high memory on a live system

**Documented HRT questions (shell & OS fundamentals):**
6. `|` vs `>` (pipe vs redirect)
7. Hard link vs symbolic link; inodes
8. `kill -15` vs `kill -9` (signals)
9. Common bash commands
10. Networking: TCP internals, multicast, packet routing, reliability

---

## B. Topic-by-topic — explanation, *why*, sources

### 1. Process vs thread (OS level)

- A **process** is an address space + resources (file descriptors, memory mappings, PID) with ≥1 thread. A **thread** is a schedulable execution context (its own stack, registers, program counter) that **shares the address space, FDs, and heap** with the other threads of its process.
- On Linux both are created by **`clone(2)`**; what differs is the sharing flags. `fork(2)` = new address space (copy-on-write of the parent's pages); a thread (`pthread_create` → `clone` with `CLONE_VM|CLONE_FILES|...`) shares them. So Linux's "everything is a task" model is the *why*: threads and processes are points on a sharing spectrum.
- **Trade-offs:** threads = cheap context switch, easy shared-memory communication, but a crash/UB in one thread takes down the process and you must synchronize shared state (locks, atomics, races). Processes = isolation and fault containment, but heavier and need IPC (pipes, shared memory, sockets).
- **Context switch cost:** switching threads in the same process keeps the page tables / TLB warm; switching processes flips the address space and can flush the TLB → more expensive.

Sources: [clone(2)](https://man7.org/linux/man-pages/man2/clone.2.html) · [fork(2)](https://man7.org/linux/man-pages/man2/fork.2.html) · [pthreads(7)](https://man7.org/linux/man-pages/man7/pthreads.7.html).

### 2. Virtual memory & page tables

- Each process sees a private **virtual address space**; the **MMU** translates virtual → physical using **page tables** (multi-level on x86-64: 4 or 5 levels). The **TLB** caches recent translations; a TLB miss walks the page table.
- A **page fault** happens when a virtual page has no valid physical mapping: *minor* (page is in memory, just needs a PTE — e.g. first touch of an `mmap`/COW page) vs *major* (must read from disk/swap — expensive). **Demand paging** means pages are loaded lazily on first access; **copy-on-write** means `fork` shares pages read-only and copies only on write.
- **Why HRT cares:** page faults and TLB misses are latency. Big working sets that don't fit cache/TLB, swapping, and unexpected major faults are classic "why is my service slow/jittery" causes. **Huge pages** reduce TLB pressure for large mappings.
- Inspect it: `/proc/<pid>/maps` and `/proc/<pid>/smaps` (per-mapping RSS), `vmstat` (`si`/`so` = swap activity), `/proc/meminfo`.

Sources: [Understanding the Linux VM (kernel docs)](https://www.kernel.org/doc/html/latest/admin-guide/mm/) · [Gorman, *Understanding the Linux Virtual Memory Manager*](https://www.kernel.org/doc/gorman/) · `man 5 proc`.

### 3. `mmap` vs standard file I/O

- **`read`/`write`** copy data between a kernel page-cache buffer and your user buffer — a syscall **and** a copy per call.
- **`mmap(2)`** maps a file (or anonymous memory) directly into the address space, so you access file contents as memory; the kernel pages them in **on demand** via page faults, and you skip the explicit copy (and can share pages across processes).
- **Use `mmap` when:** large files with random access, sharing a region between processes, memory-mapped data structures (databases, indexes), or avoiding copy overhead on big sequential reads. **Prefer plain I/O when:** small/streaming sequential reads (read-ahead handles it well), or you want simple error handling — `mmap` faults can deliver `SIGBUS` if the file shrinks under you, and page-fault-driven access has less predictable latency.
- Anonymous `mmap` is also how big allocations / `malloc` arenas get memory from the kernel.

Sources: [mmap(2)](https://man7.org/linux/man-pages/man2/mmap.2.html).

### 4. `strace` and `perf` in practice

- **`strace`** traces **system calls** and signals via `ptrace` — *what the program asks the kernel to do*. Use it to find why a process hangs (blocked in a syscall?), what files it opens (`-e trace=open,read,write`), failing syscalls (`-e fault`/error returns), latency per call (`-T`), summary table (`-c`). **Caveat (Brendan Gregg's "strace Wow Much Syscall"):** `ptrace` stops the process on *every* syscall — overhead can be **100×+**, so never leave it on a hot production process; prefer `perf trace` / eBPF for low-overhead syscall tracing.
- **`perf`** is the kernel's sampling profiler / PMU interface: `perf top` (live hottest functions), `perf record`/`perf report` (sample then analyze), `perf stat` (hardware counters: cycles, instructions, **cache misses**, branch mispredicts, IPC). It samples on a timer/event with low overhead and sees kernel + user stacks → the tool for "where are CPU cycles going" and "is this cache-bound." Pair with **flame graphs**.
- The mental split: **`strace` = syscalls/kernel boundary behavior; `perf` = where CPU time goes.**

Sources: [strace(1)](https://man7.org/linux/man-pages/man1/strace.1.html) · [perf wiki](https://perf.wiki.kernel.org/index.php/Main_Page) · [Brendan Gregg — perf examples](https://www.brendangregg.com/perf.html) · ["strace Wow Much Syscall"](https://www.brendangregg.com/blog/2014-05-11/strace-wow-much-syscall.html).

### 5. Diagnosing high CPU / high memory live (the USE method)

Have a **method**, not a random tool dump — HRT scores structured triage. Use **Brendan Gregg's USE method**: for each resource check **Utilization, Saturation, Errors.**

- **First 60 seconds:** `uptime` (load avg), `top`/`htop`, `vmstat 1`, `mpstat -P ALL 1`, `iostat -xz 1`, `free -m`, `dmesg | tail` (OOM killer? errors?), `sar`.
- **High CPU:** is it user or system time (`top`/`mpstat`)? One thread or many? `pidstat -t 1` per-thread → `perf top -p <pid>` to find the hot function → flame graph. Distinguish on-CPU (`perf`) vs off-CPU/blocked (waiting on locks/IO — look at run-queue saturation, `D`-state processes).
- **High memory:** `free`, `/proc/<pid>/status` (VmRSS), `smem`/`smaps` for shared vs private, check for swap thrash (`vmstat` si/so), `dmesg` for the OOM killer. Growing RSS → leak/cache/fragmentation (Python: `tracemalloc`; C++: ASan/valgrind/heaptrack).
- **The flaky-service flavor:** `systemctl status`/`journalctl -u <svc>`, check the listener (`ss -tlnp`), open FDs (`lsof -p`, FD leak?), connectivity (`ping`, `curl`, `tcpdump`), config/permissions, then narrate a fix + verification + rollback plan.

Sources: [USE method (Gregg)](https://www.brendangregg.com/usemethod.html) · [Linux Performance Analysis in 60s (Netflix)](https://netflixtechblog.com/linux-performance-analysis-in-60-000-milliseconds-accc10403c55) · [Linux performance tools map](https://www.brendangregg.com/linuxperf.html).

### 6. `|` vs `>` (a *documented* HRT question)

> Real HRT question (SRE, US): *"In the Linux shell, what's the difference between `|` and `>`?"*

- **`|` (pipe):** connects the **stdout of one process to the stdin of another** — an in-kernel buffer between two concurrently running processes (`a | b`). Inter-process data flow.
- **`>` (redirect):** sends a process's **stdout to a file** (truncating; `>>` appends). Process-to-file. `2>` redirects stderr, `2>&1` merges stderr into stdout, `<` reads stdin from a file.
- The crisp distinction: **pipe = process→process; redirect = process→file (or file→process).**

Sources: [Bash manual — redirections](https://www.gnu.org/software/bash/manual/bash.html#Redirections).

### 7. Hard link vs symbolic link; inode (documented)

> Real HRT questions: *"difference between a hard link and a symbolic link?"* and *"What do you know about an inode?"*

- An **inode** is the on-disk metadata structure for a file: type, permissions, owner, timestamps, size, link count, and pointers to data blocks — **everything except the name**. Directory entries map **names → inode numbers**, so a filename is just a pointer to an inode.
- **Hard link:** another directory entry pointing to the **same inode** (same inode number, `ln target link`). The file's data persists until the inode's **link count** hits 0 *and* no process holds it open. Can't span filesystems; can't (normally) link directories.
- **Symbolic (soft) link:** a tiny **separate file** whose contents are a **path** to the target (`ln -s target link`, own inode). Can span filesystems and point to directories, but **dangles** if the target is removed/moved.
- Tie-in: this is *why* `unlink`/`rm` is "decrement link count," and why deleting a file a process still has open frees space only on close (the inode lives until link count *and* open count are 0).

Sources: [inode(7)](https://man7.org/linux/man-pages/man7/inode.7.html) · [ln(1)](https://man7.org/linux/man-pages/man1/ln.1.html) · [symlink(7)](https://man7.org/linux/man-pages/man7/symlink.7.html).

### 8. `kill -15` vs `kill -9` (documented)

> Real HRT question (SRE, US): *"difference between `kill -15` and `kill -9`?"*

- **`-15` = `SIGTERM`** (the default `kill`): a **polite, catchable** request to terminate. The process can install a handler to flush buffers, close sockets, finish in-flight work, then exit — **graceful shutdown**.
- **`-9` = `SIGKILL`**: **uncatchable, unblockable**; the kernel kills the process immediately with no cleanup → risk of corrupt state, lost data, orphaned resources. Last resort.
- Best practice: send `SIGTERM`, wait, escalate to `SIGKILL` only if it won't die (this is exactly what `systemd` does on stop). Bonus: a process stuck in **uninterruptible sleep (`D` state)** won't respond even to `SIGKILL` until the I/O completes.

Sources: [signal(7)](https://man7.org/linux/man-pages/man7/signal.7.html) · [kill(1)](https://man7.org/linux/man-pages/man1/kill.1.html).

### 9. Common bash commands (documented)

> Real HRT question (Systems Eng): *"Name some common bash commands."*

Don't just list — group by purpose and mention a flag that shows depth: navigation/files (`ls -la`, `cd`, `pwd`, `cp`, `mv`, `rm`, `find . -name`, `ln -s`); inspection (`cat`, `less`, `head`/`tail -f`, `wc -l`); text processing (`grep -r`, `sed`, `awk`, `cut`, `sort`, `uniq -c`, `tr`); processes (`ps aux`, `top`/`htop`, `kill`, `jobs`, `nohup`, `&`); perms (`chmod`, `chown`); disk (`df -h`, `du -sh`); net (`ss -tlnp`, `curl`, `dig`); plus pipes/redirects and `xargs`. A one-liner like `grep ERROR app.log | awk '{print $5}' | sort | uniq -c | sort -rn` demonstrates real fluency.

Sources: [GNU coreutils](https://www.gnu.org/software/coreutils/manual/coreutils.html).

### 10. Networking: TCP, multicast, routing (deeper-loop topics)

Reports show networking goes deep for some Systems/onsite rounds (TCP internals, multicast, "design packet routing between a hub and node servers," "networking protocols and reliability").

- **TCP internals:** 3-way handshake (SYN/SYN-ACK/ACK), reliability via sequence numbers + ACKs + retransmission, **flow control** (receive window) vs **congestion control** (slow start, congestion avoidance, fast retransmit/recovery; algorithms like Cubic/BBR), Nagle's algorithm + delayed ACK (and why `TCP_NODELAY` matters for latency-sensitive trading traffic), `TIME_WAIT`. TCP vs UDP: ordered/reliable/stream vs unordered/unreliable/datagram-and-lower-latency.
- **Multicast:** one-to-many via group addresses + IGMP; **exchanges fan out market data over UDP multicast** because it scales to many subscribers without per-receiver state — directly relevant to HRT's world. Reliability is layered on top (sequence numbers + gap-fill/retransmit request), since UDP doesn't guarantee delivery.
- **Routing/reliability design Q:** structure it — addressing/topology, transport choice (TCP for reliability vs UDP/multicast for fan-out latency), failure handling (heartbeats, timeouts, retries, idempotency), ordering/dedup (sequence numbers), backpressure, and redundancy (dual feeds / A-B arbitration, which is how real market-data plants work).
- Inspect: `ss`, `tcpdump`/`wireshark`, `ip route`, `netstat`, `ethtool`.

Sources: [TCP/IP Illustrated Vol.1 (Stevens)](https://en.wikipedia.org/wiki/TCP/IP_Illustrated) · [Beej's Guide to Network Programming](https://beej.us/guide/bgnet/) · [tcpdump(1)](https://www.tcpdump.org/manpages/tcpdump.1.html) · [ss(8)](https://man7.org/linux/man-pages/man8/ss.8.html).

---

## C. Previously asked (from real HRT reports)

1. `|` vs `>`; hard vs symbolic link; `kill -15` vs `-9`; Python `yield`/generator/decorator. *(SRE, US — friendly round)*
2. **"What do you know about an inode?"** (+ 6 more Linux/systems Qs). *(Systems Eng, NY)*
3. **"Name some common bash commands."** *(Systems Eng)*
4. **SSH into a cloud box and debug a flaky service**, narrate steps. *(Blind, Systems/Python loop)*
5. **TCP internals, multicast, process debugging, quick scripting**; "ssh in and fix a flaky service while explaining." *(Reddit, Systems Eng)*
6. **"Design a system to route network packets between one hub and multiple node servers."** *(SWE final round, US)*
7. **Networking protocols and reliability**; **OS** round; **K-d tree / balanced-tree internals**. *(SWE intern, US — got offer; very dense, rapid follow-ups)*
8. **"Design a task-switching algorithm"; caching; graph theory.** *(SWE intern, Singapore)*
9. Low-level discussion: **OS, compilers, C++, memory** (no coding). *(SWE intern, France)*
10. **OS / virtual memory / process management** under-the-hood questions across many reports.

---

## D. Extrapolated Q&A with model answers

**Q1. Process vs thread — when would you pick each?**
> Both are Linux tasks created by `clone`; the difference is what they share. A process has its own address space and FDs; threads share them. I pick **threads** when I need cheap communication through shared memory and lightweight context switches, accepting that I must synchronize and that one thread's crash kills the process. I pick **processes** when I want fault isolation and security boundaries, accepting IPC overhead and a heavier context switch that can flush the TLB. For a high-reliability service I lean toward process isolation for the supervisor and threads inside a worker for throughput.

**Q2. A service is using 100% of one CPU. Walk me through diagnosing it.**
> I'd go top-down with the USE method. `top`/`mpstat` to confirm it's one core and whether the time is user or system. If system-heavy, it's syscalls/contention — `strace -c` (briefly) or `perf trace` to see which syscalls dominate. If user-heavy, `pidstat -t` to find the hot thread, then `perf top -p <pid>` or a `perf record` flame graph to find the hot function. I distinguish a genuinely CPU-bound hot loop from a busy-wait/spinlock — a tight retry loop shows up as high CPU but no useful work. Then I'd fix the hot path, re-profile to confirm, and watch for regressions.

**Q3. RSS keeps climbing on a long-running service. How do you tell a leak from normal caching?**
> Watch the shape: a leak grows unbounded and monotonically; a cache plateaus at its bound. I confirm with `/proc/<pid>/smaps` for private vs shared and `vmstat` for swap thrash, check `dmesg` for the OOM killer. To attribute it I use language tools — `tracemalloc` snapshots+diff for Python, heaptrack/ASan/valgrind for C++ — and follow the referrer chain. If Python tools are clean but RSS still grows, I suspect a C-extension missing a decref or allocator fragmentation, where freed memory isn't returned to the OS.

**Q4. Why is `mmap` sometimes faster than `read`, and when is it not?**
> `mmap` maps the file into my address space so I touch its pages directly; the kernel pages them in on demand via faults, skipping the explicit copy from page cache into a user buffer and the per-call syscall overhead — great for large files with random access and for sharing a region between processes. It's *not* better for small streaming sequential reads, where read-ahead already hides latency and plain `read` is simpler; and `mmap` access has page-fault-driven latency variance plus `SIGBUS` risk if the file is truncated underneath me.

**Q5. You SSH into a box; a service is up but not responding. First five things you check?**
> (1) `systemctl status` / `journalctl -u <svc>` for crashes, restarts, errors. (2) Is it actually listening — `ss -tlnp` on the expected port. (3) Resource pressure — `top`, `free`, `df -h` (a full disk silently breaks tons of services). (4) Is it stuck — `strace -p <pid>` to see if it's blocked in a syscall, or `D`-state in `top` indicating stuck I/O; FD exhaustion via `lsof -p`. (5) Connectivity/config — `curl localhost:<port>`, `tcpdump` to see if requests even arrive, then config/permissions. I'd state a hypothesis, test it, and have a rollback plan before changing anything.

**Q6. Explain a page fault and why it can cause latency jitter in a low-latency service.**
> A page fault fires when I touch a virtual page with no valid physical mapping. A *minor* fault just wires up a PTE (e.g. first touch of allocated memory) — cheap; a *major* fault reads from disk/swap — milliseconds, catastrophic for a microsecond-scale path. In trading systems you avoid this by pre-faulting/pre-touching memory, locking pages with `mlock`, using huge pages to cut TLB misses, and keeping the working set resident so the kernel never swaps you out mid-burst.

**Q7. Design reliable market-data delivery from one source to many consumers.**
> I'd fan out over **UDP multicast** so the source sends once regardless of subscriber count — that's how exchanges do it and it keeps per-receiver state off the publisher. Since UDP can drop/reorder, I layer reliability: monotonic **sequence numbers** so receivers detect gaps, a **gap-fill/retransmit** channel or a snapshot+incremental scheme to recover, and **dual redundant feeds (A/B)** that receivers arbitrate so a loss on one is covered by the other. For control/slow paths where reliability beats latency I'd use TCP. Key trade-off I'd call out: multicast optimizes fan-out latency at the cost of building reliability yourself.

---

## E. Hands-on warm-ups (run these for real)

- `strace -T -e trace=openat,read,write <cmd>` and read the output; `strace -c` summary.
- `perf top`, `perf stat <cmd>` (read IPC, cache-misses, branch-misses), `perf record`/`report`.
- `cat /proc/<pid>/maps` and `smaps`; `vmstat 1` while running a memory hog; watch `si`/`so`.
- `ss -tlnp`, `lsof -p <pid>`, `tcpdump -i lo port 8080`.
- Make a service "flaky" (wrong perms, full disk via a loopback file, port already bound, FD leak) and practice diagnosing it cold while narrating.
- A log-analysis one-liner: `journalctl -u svc --since "1h ago" | grep -i error | awk '...' | sort | uniq -c | sort -rn`.

---

## F. Segment tips

- **Bring a method, not just commands.** Open with USE/RED-style structure ("I'd check utilization, then saturation, then errors for CPU, mem, disk, net"). That framing reads as senior.
- **Narrate like an incident responder:** hypothesis → test → observe → next step, with a rollback/verification plan. That's the "operational thinking" the SWE-SRE framing wants.
- **Always give the *why*** — not "`-9` kills it" but "`-9` is uncatchable so it skips cleanup and risks corrupt state; I escalate to it only after `-15`."
- **Know `strace`'s overhead caveat** — saying "I wouldn't leave `strace` on a hot prod process because `ptrace` traps every syscall; I'd use `perf`/eBPF" is a standout senior signal.
- **Tie everything to latency** where you can — page faults, context switches, cache misses, TCP_NODELAY. This is a trading firm; latency is the religion.
- For the **SSH live-debug round**, production muscle memory is the asset — but *talk through* every command and what each output rules in/out. Silence reads as flailing.
