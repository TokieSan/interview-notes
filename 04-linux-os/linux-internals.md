# Linux & OS Internals (C1)

Mechanism first, then the operational consequence. This is the densest, highest-yield file for any systems / SRE / production-engineering round.

Answer format throughout: **the mechanism, then the operational consequence.** Google's rubric explicitly rewards mechanistic understanding over command recall; Apple's rewards the decision the mechanism forces. Every answer below is written so the second half is reusable in either register.

## C1 — Linux & OS internals
*Feeds: G2 primarily, A2, and grounds every capacity estimate in G3.*

### Processes, threads, scheduling

**C1.1 — Process vs thread, precisely.**
Linux has one primitive: the *task* (`task_struct`). `clone()` decides what is shared. A thread is a task sharing `CLONE_VM | CLONE_FILES | CLONE_FS | CLONE_SIGHAND` with its siblings; a process is a task sharing almost nothing. `fork()` is `clone()` with nothing shared. Threads within a process share the address space, fd table, and signal dispositions but each has its own stack, registers, and kernel stack.
*Consequence:* a context switch between threads of the same process skips the page-table swap and (on non-PCID hardware) the TLB flush — that's why the "threads are cheaper" claim is true, and it is true for a specific, nameable reason. Say the reason.

**C1.2 — What `fork()` actually copies.**
Nothing eagerly. Page tables are duplicated with all writable pages marked read-only in both parent and child; the first write traps a page fault and the kernel copies that page. Copy-on-write. `vfork()` doesn't even duplicate page tables and suspends the parent — legacy optimisation for immediate `exec()`.
*Consequence:* a 40 GB JVM can `fork()` in milliseconds, but a `fork()`-then-write-heavy child can quietly double RSS. This is the classic Redis `BGSAVE` memory spike, and it's a good thing to name unprompted.

**C1.3 — `fork()` + `exec()` in sequence.**
`fork()` returns twice (0 in child, child PID in parent). Child calls `execve()`: kernel tears down the address space, maps the new binary's segments, maps the dynamic linker (`ld.so`) if dynamically linked, resets signal handlers to default (but keeps ignored ones ignored), preserves the fd table except entries marked `FD_CLOEXEC`. Control transfers to `ld.so`, which resolves the needed shared objects and jumps to `_start`.
*Consequence:* fd inheritance across `exec` is where "my daemon holds the deleted log file open" bugs come from. `FD_CLOEXEC` is the fix.

**C1.4 — Zombie vs orphan.**
A **zombie** has exited but its parent hasn't `wait()`ed, so the kernel retains the exit status and PID in the process table. It holds no memory or fds — just a PID slot. An **orphan** is still running but its parent died; it is reparented to `init`/`systemd` (or nearest subreaper), which reaps it.
*Consequence:* zombies are only harmful in bulk — they exhaust the PID space. `kill -9` on a zombie does nothing; kill or fix the *parent*. This connects directly to a common interview question "a system is running out of PIDs, how do you detect and stop it": check `/proc/sys/kernel/pid_max`, `ps -eo stat | grep Z | wc -l`, find the common PPID, and look for a missing `wait()` loop or a fork bomb — then containment via cgroup `pids.max`.

**C1.5 — Signals: the mechanism.**
A signal sets a bit in the task's pending mask. It is delivered when the task next returns from kernel to user mode. The kernel pushes a signal frame onto the user stack, runs the handler, and `sigreturn()` restores. `SIGKILL` (9) and `SIGSTOP` (19) are handled entirely in the kernel and cannot be caught, blocked, or ignored. Standard signals are *not queued* — two identical pending signals collapse to one; real-time signals (`SIGRTMIN`+) do queue.
*Consequence:* `kill -9` skips every cleanup path — no flushing buffers, no releasing locks, no removing pid files, no graceful connection drain. `SIGTERM` first, `SIGKILL` after a timeout, is a policy, not a preference. If a process ignores `SIGKILL`, it is in uninterruptible sleep (`D` state) blocked in a kernel call — usually stuck I/O or an NFS hang.

**C1.6 — Process states you must be able to read from `ps`.**
`R` runnable (running *or* on the runqueue), `S` interruptible sleep, `D` uninterruptible sleep (in-kernel, usually I/O), `T` stopped, `Z` zombie, `I` idle kernel thread.
*Consequence:* `D` state processes count toward Linux load average. This is why load average is not CPU utilisation on Linux — a box with load 40 and 5% CPU is an I/O problem, not a CPU problem. Excellent thing to say in G1.

**C1.7 — CFS in one paragraph.**
Completely Fair Scheduler tracks `vruntime` per task, weighted by nice value, in a red-black tree; it always runs the leftmost (lowest vruntime) task. No fixed timeslices — a target latency is divided among runnable tasks. Real-time classes (`SCHED_FIFO`, `SCHED_RR`) preempt everything in `SCHED_OTHER`. (Newer kernels ship EEVDF as CFS's successor; know the name, know it's still virtual-time-fairness with an added eligibility/deadline notion.)
*Consequence:* CPU cgroup throttling (`cfs_quota_us`) is a per-period hard cap. A container with a modest average CPU use can still be throttled hard if it bursts, causing latency spikes that look like nothing on an averaged CPU graph. `/sys/fs/cgroup/cpu.stat` → `nr_throttled` is the check.

**C1.8 — Context switch cost, concretely.**
~1–3 µs direct (register save, scheduler, page-table swap), but the real cost is indirect: cold L1/L2 and TLB after the switch, often 10s of µs of degraded execution. Thread-to-thread in-process is cheaper (no CR3 reload).
*Consequence:* this is the argument for event loops / io_uring over thread-per-connection at high connection counts, and for pinning latency-critical threads.

### Memory

**C1.9 — Virtual memory path.**
Virtual address → TLB lookup → hit: physical address. Miss: page-table walk (4- or 5-level on x86-64). Entry not present → page fault. **Minor fault**: page is in memory (page cache, or CoW, or a zero page) — just fix the mapping. **Major fault**: must read from disk.
*Consequence:* major faults are the metric that matters. `ps -o min_flt,maj_flt`, `/proc/vmstat` `pgmajfault`. A service whose latency degrades as its working set grows past RAM shows up here long before it shows on CPU.

**C1.10 — Page cache.**
Unused RAM is used to cache file-backed pages. Reads are served from it; writes land in it and are marked dirty, flushed by `pdflush`/writeback threads under `vm.dirty_ratio` / `dirty_background_ratio` thresholds.
*Consequence:* "free memory is low" is almost never a problem — check `available`, not `free`. But large dirty ratios mean a writeback storm can stall the whole box for seconds. And `fsync()` is the only thing that gets you a durability guarantee; a successful `write()` guarantees nothing but visibility to other readers.

**C1.11 — `mmap` vs `read`.**
`read()` copies kernel page cache → user buffer (one copy, one syscall per call). `mmap()` maps page cache pages into the address space — zero copy, but you pay a page fault per new page touched, and TLB pressure, and you lose explicit error handling (I/O errors arrive as `SIGBUS`, not a return code).
*Consequence:* `mmap` wins for random access over a large file that mostly fits in cache; `read()` with a decent buffer wins for sequential streaming. Kafka-style systems use `sendfile()` to skip user space entirely.

**C1.12 — OOM killer.**
When reclaim fails, the kernel scores every task by `oom_score` (roughly proportional to RSS, adjustable via `oom_score_adj`) and kills the highest. Memory cgroups have their own OOM path, killing within the cgroup.
*Consequence:* the OOM killer usually kills your biggest process, which is usually the important one. In containers the cgroup limit fires first — `dmesg` and `memory.events` → `oom_kill`. And overcommit (`vm.overcommit_memory`) means `malloc()` succeeding tells you nothing; the failure arrives later, at touch time, as a kill.

**C1.13 — Where does a memory leak show?**
RSS grows without bound; `pmap -x` or `/proc/PID/smaps` shows which mapping. Anonymous growth = heap/`malloc` leak. If RSS is flat but the process still OOMs, look at kernel-side objects (`slabtop`) or fd leaks holding buffers. glibc `malloc` returning memory to the OS is arena- and threshold-dependent — RSS not shrinking after a free burst isn't necessarily a leak.

### Filesystems

**C1.14 — Inode: what's in it and what isn't.**
In it: type, permissions, uid/gid, link count, size, timestamps (atime/mtime/ctime), and pointers to data blocks (direct, then indirect/double/triple, or extents on ext4/XFS). **Not** in it: the filename. The name lives in the directory entry, which maps name → inode number.
*Consequence:* this single fact explains hard links (multiple dirents → same inode, link count > 1), why `mv` within a filesystem is O(1), why you can delete a file another process has open (dirent removed, link count 0, but the open fd keeps the inode alive — space is not freed until last close), and why `df` and `du` diverge. That last one is the highest-value production answer in this section.

**C1.15 — Hard vs symbolic link.**
Hard link: another dirent pointing at the same inode. Cannot cross filesystems (inode numbers are per-filesystem), cannot link directories (cycle risk). Symlink: its own inode whose data is a path string; resolved at open time; can dangle, can cross filesystems.

**C1.16 — "`df` says full, `du` says half empty."**
A process holds a deleted file open. Find it: `lsof +L1` (link count < 1). Fix: restart the holder, or truncate via `/proc/PID/fd/N`. Second cause: inode exhaustion — `df -i`. Third: reserved blocks (5% root reserve by default on ext4).

**C1.17 — Journaling.**
ext4 modes: `journal` (data + metadata journalled, safest, slowest), `ordered` (default — metadata journalled, data written before the metadata commit), `writeback` (metadata only, can expose stale data after a crash).
*Consequence:* the journal bounds recovery time, not data loss. Durability of *your* write still needs `fsync()` — and on some configurations, `fsync()` on the parent directory too, if you created the file.

**C1.18 — What happens when you type `ls -l`.**
Worth rehearsing end to end, it's a recurring Google question. Shell reads the line via terminal driver in canonical mode → parses, performs expansions (see C1.19) → `fork()` → child `execve("/bin/ls")` after PATH search → kernel loads ELF, maps segments, maps `ld.so` → dynamic linking resolves `libc` → `main()` → `ls` calls `openat()`/`getdents64()` on the directory, then `statx()` per entry for the `-l` metadata, plus `getpwuid`/`getgrgid` (possibly hitting NSS → LDAP → network!) to resolve names → formats into a buffer → `write()` to fd 1 → terminal driver → `exit()` → parent `wait()`s and reaps.
*Two things to say unprompted:* (a) the per-entry `stat` is why `ls -l` on a directory with 500k entries is slow while `ls` alone is fast; (b) the NSS lookup is why `ls -l` can hang on a box with a broken LDAP/DNS — a genuinely great cross-domain observation.

**C1.19 — What happens when you run `ls *`.**
**The shell expands the glob, not `ls`.** The shell `readdir`s the current directory, matches, sorts, and passes each match as a separate argv entry. So: `ls *` in an empty dir passes the literal `*`; a filename beginning with `-` becomes an accidental flag; and past ~2 MB of combined argv+environment you get `E2BIG` — "argument list too long", which is a shell/kernel limit (`MAX_ARG_STRLEN`, `ARG_MAX`), not an `ls` limit. Fix with `find … -exec … +` or `xargs`.

### Syscalls, permissions, boot

**C1.20 — Syscall mechanism.**
User code puts the syscall number in `rax`, arguments in `rdi, rsi, rdx, r10, r8, r9`, executes `syscall` (fast path via MSR-configured entry point; the old `int 0x80` is legacy). CPU switches to ring 0, kernel dispatches through the syscall table, returns a value in `rax` — negative values in `-4095..-1` are errnos. glibc wraps this and sets `errno`.
*Consequence:* the cost is ~50–100ns baseline, materially more post-Spectre/Meltdown mitigations (KPTI page-table switching). This is why batching syscalls matters — `readv`/`writev`, `epoll` over `select`, `io_uring` over per-op syscalls.

**C1.21 — "Which syscall returns non-zero on success?"**
Plenty, and the framing is the point: most syscalls return a *resource or count*, not a status. `fork()` returns the child PID to the parent (non-zero) and 0 to the child. `read`/`write` return byte counts. `open` returns an fd (≥0, and never 0 in practice since 0 is stdin). `getpid()` can't fail. The general rule: **-1 plus `errno` signals failure; anything else is data.**

**C1.22 — Permission bits beyond rwx.**
setuid (run as file owner), setgid (run as group; on a *directory*, new files inherit the group), sticky bit (on a directory, only the owner can unlink — `/tmp`). Then: capabilities (fine-grained root, `CAP_NET_BIND_SERVICE` etc.), ACLs, and LSMs (SELinux/AppArmor) which can deny after DAC has allowed.
*Consequence:* "permission denied as root" → check capabilities, an LSM, or an immutable attribute (`chattr +i`), or a read-only mount.

**C1.23 — Boot sequence.**
Firmware (UEFI) → bootloader (GRUB) → kernel + initramfs loaded → kernel decompresses, initialises, mounts initramfs as temporary root → loads drivers needed for the real root → `pivot_root` → PID 1 (`systemd`) → units by dependency ordering → target reached.
*Consequence:* "boots to initramfs prompt" = the real root's driver or the root device UUID is wrong. Practical, not trivia.

**C1.24 — Containers, mechanically.**
Not a kernel object. A container is a process with (a) **namespaces** — pid, mnt, net, uts, ipc, user, cgroup — restricting what it can *see*, and (b) **cgroups** limiting what it can *use*, and (c) usually seccomp + capability drops + an LSM profile restricting what it can *do*. Shared kernel throughout.
*Consequence:* the isolation boundary is the syscall interface, which is large — that's the argument for VMs (or gVisor/Firecracker) when the threat model needs a hardware boundary. **A common senior-level probe: "container vs VM, and what happens to your isolation model under memory pressure."** See A2.

## One-line summary

**Linux:** one task type, `clone()` decides sharing · fork is CoW · zombie = unreaped, orphan = reparented · SIGKILL uncatchable, skips cleanup · `D` state counts in load · inode has everything but the name · df≠du → deleted-open file or inodes · shell expands the glob, not the program · major faults are the memory metric.
