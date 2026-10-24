# Linux / Systems Q&A Drill

**Q: Process vs thread at the OS level?**
A: Process = address space + resources (FDs, mappings, PID) with ≥1 thread. Thread = schedulable context (own stack/registers/PC) that shares the address space, FDs, and heap with sibling threads. Both made by `clone(2)` on Linux — they differ by sharing flags (`fork` = new COW address space; `pthread_create` → `clone` with `CLONE_VM|CLONE_FILES|...`). Threads: cheap switch, easy shared memory, but one crash kills the process and you must synchronize. Processes: isolation, but heavier and need IPC. Same-process thread switch keeps page tables/TLB warm; process switch flips the address space and can flush the TLB.

**Q: Virtual memory & page tables?**
A: Each process gets a private virtual address space; the MMU translates virtual→physical via multi-level page tables (4–5 levels on x86-64); the TLB caches translations, a miss walks the table. Page fault = no valid physical mapping: minor (page in memory, just needs a PTE — first touch of `mmap`/COW) vs major (read from disk/swap — expensive). Demand paging loads lazily; copy-on-write shares pages read-only after `fork`. Huge pages cut TLB pressure. Inspect: `/proc/<pid>/maps`, `smaps`, `vmstat` (si/so), `/proc/meminfo`.

**Q: mmap vs read/write?**
A: `read`/`write` copy between the kernel page cache and your buffer — a syscall *and* a copy per call. `mmap` maps the file into the address space so you access it as memory; the kernel pages it in on demand via faults, no explicit copy, shareable across processes. Use mmap for large random-access files, sharing, memory-mapped DBs/indexes. Prefer plain I/O for small/streaming sequential reads (read-ahead handles it) or simpler error handling — mmap can `SIGBUS` if the file shrinks under you and has less predictable latency.

**Q: strace vs perf?**
A: `strace` traces syscalls + signals via `ptrace` — *what the program asks the kernel to do*. Find hangs (blocked in a syscall?), opened files, failing syscalls, per-call latency (`-T`), summary (`-c`). Caveat: ptrace stops the process on *every* syscall → 100×+ overhead, never on a hot prod process (use `perf trace`/eBPF). `perf` = sampling profiler / PMU interface: `perf top` (live hot functions), `record`/`report`, `stat` (cycles, instructions, cache misses, branch mispredicts, IPC) — low overhead, sees kernel + user. Split: **strace = syscall/kernel boundary; perf = where CPU time goes.**

**Q: Diagnose high CPU on a live box?**
A: Use a method (USE — Utilization, Saturation, Errors). First 60s: `uptime`, `top`, `vmstat 1`, `mpstat -P ALL 1`, `iostat -xz 1`, `free -m`, `dmesg | tail`. For CPU: user vs system time? one thread or many (`pidstat -t 1`)? → `perf top -p <pid>` to find the hot function → flame graph. Distinguish on-CPU (perf) vs off-CPU/blocked (D-state processes, run-queue saturation, waiting on locks/IO).

**Q: Diagnose high memory?**
A: `free`, `/proc/<pid>/status` (VmRSS), `smem`/`smaps` for shared vs private, `vmstat` si/so for swap thrash, `dmesg` for the OOM killer. Growing RSS → leak/cache/fragmentation (Python: `tracemalloc`; C++: ASan/valgrind/heaptrack).

**Q: Debug a flaky service?**
A: `systemctl status` / `journalctl -u <svc>`, check the listener (`ss -tlnp`), open FDs (`lsof -p` — FD leak?), connectivity (`ping`/`curl`/`tcpdump`), config/permissions. Then narrate a fix + verification + rollback plan. (Treat it as an incident you're triaging out loud.)

**Q: `|` vs `>`?**
A: Pipe = stdout of one process → stdin of another (in-kernel buffer between two concurrently running processes). Redirect = process stdout → file (truncate; `>>` appends). `2>` stderr, `2>&1` merges stderr into stdout, `<` reads stdin from a file. Crisp: **pipe = process→process; redirect = process→file.**

**Q: Hard link vs symbolic link? What's an inode?**
A: Inode = on-disk metadata for a file (type, perms, owner, timestamps, size, link count, data-block pointers) — everything except the name. Directory entries map names→inode numbers. Hard link = another dir entry to the *same* inode; data persists until link count hits 0 *and* no process holds it open; can't cross filesystems or link dirs. Symlink = a tiny separate file whose contents are a path to the target; can cross filesystems and point to dirs, but dangles if the target moves. That's why `rm` is "decrement link count," and deleting a file a process still has open frees space only on close.

**Q: kill -15 vs kill -9?**
A: `-15` = SIGTERM, the polite *catchable* terminate — the process can flush buffers, close sockets, finish in-flight work, then exit (graceful shutdown). `-9` = SIGKILL, uncatchable/unblockable, kernel kills immediately with no cleanup → risk of corrupt state. Best practice: SIGTERM, wait, escalate to SIGKILL only if needed (exactly what systemd does). A process in uninterruptible sleep (D state) won't respond even to SIGKILL until the I/O completes.

**Q: Name common bash commands (grouped)?**
A: Files: `ls -la`, `cd`, `cp`, `mv`, `rm`, `find . -name`, `ln -s`. Inspect: `cat`, `less`, `head`/`tail -f`, `wc -l`. Text: `grep -r`, `sed`, `awk`, `cut`, `sort`, `uniq -c`, `tr`. Processes: `ps aux`, `top`, `kill`, `nohup`, `&`. Perms: `chmod`, `chown`. Disk: `df -h`, `du -sh`. Net: `ss -tlnp`, `curl`, `dig`. Fluency one-liner: `grep ERROR app.log | awk '{print $5}' | sort | uniq -c | sort -rn`.

**Q: TCP internals?**
A: 3-way handshake (SYN/SYN-ACK/ACK); reliability via sequence numbers + ACKs + retransmission; flow control (receive window) vs congestion control (slow start, congestion avoidance, fast retransmit/recovery; Cubic/BBR); Nagle + delayed ACK and why `TCP_NODELAY` matters for latency-sensitive trading; `TIME_WAIT`. TCP vs UDP: ordered/reliable/stream vs unordered/unreliable/datagram and lower latency.

**Q: Why multicast for market data?**
A: One-to-many via group addresses + IGMP; exchanges fan out market data over UDP multicast because it scales to many subscribers without per-receiver state. Reliability is layered on top (sequence numbers + gap-fill/retransmit requests) since UDP doesn't guarantee delivery. Real plants run dual feeds with A/B arbitration.

**Q: Design packet routing between a hub and node servers?**
A: Structure it: addressing/topology → transport choice (TCP for reliability vs UDP/multicast for fan-out latency) → failure handling (heartbeats, timeouts, retries, idempotency) → ordering/dedup (sequence numbers) → backpressure → redundancy (dual feeds / A-B arbitration).

---

## OS trivia that keeps coming up

- **Process vs thread:** process = own address space; thread = shares address space within a process (shares heap/globals/fds, has own stack + registers). Threads cheaper to create/switch; a crash in one thread can take the process down.
- **Virtual memory / paging:** each process sees a virtual address space; MMU maps pages → physical frames via page tables; TLB caches translations. Page fault → OS loads page (or SIGSEGV if invalid). Demand paging, copy-on-write (fork).
- **Context switch:** save/restore registers + PC + memory maps; costs cache/TLB pollution. Kernel vs user mode; syscalls trap into kernel.
- **Scheduling:** preemptive; policies (round-robin, priority, CFS on Linux). Time slices/quanta.
- **fork/exec:** `fork` duplicates process (COW); `exec` replaces image. Zombie = exited but not reaped (`wait`); orphan = parent died, reparented to init.
- **Memory segments:** text (code), data (initialized globals), BSS (uninit globals), heap (grows up, `malloc`), stack (grows down, locals/frames). Stack overflow vs heap exhaustion.
- **IPC:** pipes, named pipes (FIFO), shared memory (fastest, needs sync), message queues, sockets, signals.
- **Syscalls to know:** `read/write`, `mmap`, `open/close`, `epoll/select/poll`, `futex` (backs mutexes).
- **Cache hierarchy:** L1/L2/L3 → RAM → disk; orders-of-magnitude latency jumps. Cache line ~64B → false sharing (two threads write different vars on same line → ping-pong). Fix: pad/align (`alignas(64)`).
- **User vs kernel space;** **interrupts vs polling;** **blocking vs non-blocking I/O.**

**Rough latency numbers (know the order of magnitude):** L1 ~1ns, L2 ~4ns, main memory ~100ns, SSD read ~16µs, network round trip within datacenter ~0.5ms, disk seek ~10ms.

## Escalation chains

### Chain 4A — SSH & port forwarding

**Prompt:** "You're going to be SSHing into boxes constantly here. Walk me through what actually happens when you run `ssh user@host` — from keystroke to shell."
> TCP connect to port 22, then the SSH **transport layer** negotiates: protocol version, then a key exchange (Diffie-Hellman style) that produces a shared session secret without sending it over the wire, plus server host-key verification against my `known_hosts` to prevent MITM. That session key seeds **symmetric** encryption for everything after. Then **authentication** — usually public key: the server has my public key in `authorized_keys`, challenges me, and I prove I hold the private key without transmitting it. Finally the **connection layer** multiplexes logical channels over the one encrypted pipe — an interactive shell, but also port forwards or `scp`, all over the same TCP connection.

**Follow-up:** "You've written SSH port-forwarding scripts. Explain local vs remote forwarding — `-L` vs `-R` — and when you'd use each."
> `-L` (local) forwards a port on *my* machine through the SSH server to a destination it can reach — `ssh -L 8080:db.internal:5432 host` lets me hit `localhost:8080` and reach a database only the bastion can see. `-R` (remote) is the reverse: it opens a port on the *remote* server that tunnels back to something *I* can reach — useful to expose a local dev service to a remote box, or to get back into a machine behind NAT. Mnemonic: `-L` = "pull a remote service to me," `-R` = "push a local service out there." `-D` gives a dynamic SOCKS proxy.

### Chain 4B — the live debug ("fix the flaky service")

**Prompt:** "Here's a scenario. You SSH into a box. A service is running — the process is up — but clients are getting errors. No alerts fired. What do you do? Think out loud."
> I'd triage like an incident, hypothesis-driven. First, is it actually listening — `ss -tlnp` on the expected port; a process can be "up" but not bound. Then logs — `journalctl -u <svc> --since "15 min ago"` and the app log for errors/restarts. Resource pressure — `top`, `free -m`, and critically `df -h`, because a full disk silently breaks writes and logging. Is it stuck — `strace -p <pid>` briefly to see if it's blocked in a syscall, or a `D`-state in `top` meaning stuck on I/O; `lsof -p <pid>` to check for FD exhaustion. Then connectivity — `curl localhost:<port>` to isolate app-vs-network, `tcpdump` to confirm requests even arrive. I state a hypothesis before each step and I don't mutate anything without knowing how I'd roll it back.

**Follow-up:** "Say `top` shows the process at 100% CPU but throughput is zero. What's your read?"
> That smells like a **busy-wait or spin** — a tight retry/poll loop burning CPU without making progress, or a livelock. I'd confirm with `perf top -p <pid>` to see the hot function, or `strace`/`py-spy` to see if it's spinning in user space vs hammering a failing syscall (e.g., retrying a connection in a tight loop with no backoff). The fix is usually adding backoff/blocking instead of polling. If you've debugged this class for real (an unbounded retry loop, a reconnect storm), say so — it proves you've owned it.

**Follow-up:** "You mentioned `strace` has overhead. How much, and what would you reach for instead on a hot prod process?"
> `strace` uses `ptrace`, which stops the process on every syscall enter and exit — two context switches per call — so on a syscall-heavy process it can be a 100×-plus slowdown. Brendan Gregg's "strace wow much syscall" post is the canonical warning. On a hot production process I'd use `perf trace` or an eBPF tool (bpftrace) instead, which hook in-kernel and sample/aggregate with far lower overhead, so I'm not adding a latency cliff to the thing I'm debugging.

### Chain 4C — fundamentals rapid round (expect several in a row)

**Prompt:** "Quick ones. `|` vs `>`?"
> Pipe connects one process's stdout to another's stdin — process to process, in-kernel buffer. `>` redirects stdout to a file, truncating (`>>` appends) — process to file. Pipe = process→process, redirect = process→file.

**Prompt:** "Hard link vs symlink."
> A hard link is another directory entry pointing to the same inode — same inode number, shares the data, can't cross filesystems, and the data lives until the link count hits zero and nothing has it open. A symlink is a separate little file whose contents are a path to the target — can cross filesystems and point at directories, but dangles if the target moves.

**Prompt:** "`kill -15` vs `-9`."
> `-15` is SIGTERM — catchable, the polite request that lets the process flush and shut down gracefully; it's the default. `-9` is SIGKILL — uncatchable, the kernel reaps it immediately with no cleanup, risking corrupt state. I send TERM, wait, escalate to KILL only if it won't die — which is exactly what systemd does on stop. A process stuck in `D` state won't even respond to KILL until its I/O completes.

**Prompt:** "What's an inode?"
> The on-disk metadata for a file — type, permissions, owner, timestamps, size, link count, and the pointers to the data blocks. Everything *except* the name; the name lives in the directory entry that maps a name to an inode number. That indirection is why hard links and atomic renames work.

**Follow-up:** "Process vs thread — and which has the cheaper context switch?"
> Both are Linux tasks made by `clone`; the difference is what they share. Threads share the address space and FDs; a process has its own. Switching between threads of the same process is cheaper because the page tables and TLB stay warm — switching processes flips the address space and can flush the TLB. The trade is isolation: a process contains faults, threads don't.

---
