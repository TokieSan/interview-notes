# Troubleshooting Scenario Bank

Practise narrating each one: hypothesis → the evidence that would confirm or kill it → next step. Method is in [debugging-method.md](debugging-method.md); command-level playbooks are in [linux-performance-playbook.md](linux-performance-playbook.md). > Section IDs like `C1.6` or `C3.12` are cross-references inside the core systems notes: C1 = [Linux internals](../04-linux-os/linux-internals.md), C2 = [networking](../05-networking/networking-core.md), C3 = [distributed systems](../07-system-design/distributed-systems.md), C4 = [reliability & SLOs](../07-system-design/reliability-slos.md), C5 = [debugging method](../06-troubleshooting/debugging-method.md), C6 = [numbers](../07-system-design/capacity-numbers.md), C7 = [coding patterns](../01-coding/sre-practical-problems.md).

# Scenario prompts to practise out loud (8 min each)

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

# Worked scenarios

**"The website is slow." Walk me through it.**
Clarify: slow how — latency, errors, total outage? All pages or one endpoint? Started when, correlates with a deploy? → Check dashboards: is it latency or error rate? Which layer — frontend, app, DB? → Bisect: is the LB healthy, are app servers saturated (CPU/mem/threads), is a DB query slow (locks, missing index, replica lag), is a downstream dependency timing out? → Check recent changes (deploy/config). → Mitigate (roll back / scale / failover), then root-cause.

**"Latency just spiked on one service."**
Correlate with deploys, traffic, dependency health. Check the four golden signals (latency, traffic, errors, saturation). Is it GC pauses / major page faults / swap / a slow downstream / connection-pool exhaustion / a noisy neighbor? Restart storms and unbounded retry loops are classic latency/instability root causes — name them.

**"A server keeps running out of memory / getting OOM-killed."**
`dmesg`/`journalctl` for the OOM kill → which process → is it a genuine leak (RES grows unbounded) or under-provisioned for load? → heap profile / `/proc/<pid>/status` → fix the leak or right-size limits / add backpressure.

**"Disk is full on a production host."**
`df -h` to confirm → `du -sh /*` to find the culprit → usually runaway logs or a stuck temp file. Gotcha: a deleted-but-still-open file holds space until the process closes the fd — `lsof | grep deleted`. Mitigate (rotate/compress/truncate), then fix log rotation / add an alert at 80%.

**"Requests intermittently fail with 503s."**
Some backends unhealthy? LB health checks failing? Connection-pool/thread-pool exhausted? Rate limiter tripping? Downstream dependency flapping? Check error distribution across hosts (one bad host vs systemic). Drain the bad host, then root-cause.

**"DNS resolution is failing for a service."**
`dig`/`nslookup` from the affected host → resolver reachable? record exists/correct? TTL/caching stale? recent DNS change? → test from a known-good host to isolate local vs global.

---

# On-call scenarios with commands

**"A production server is slow / high CPU":**

- `top`/`htop` → which process, %CPU, load average. `pidstat 1`, `ps -eo pid,ppid,cmd,%cpu,%mem --sort=-%cpu`.
- User CPU vs system CPU vs I/O wait (`top` breakdown). High `%wa` → I/O bound, not CPU bound.
- Per-thread: `top -H`, `perf top` to find the hot function. Runaway loop? Bad deploy? Cron job?

**"Load average high but CPU & memory look normal":**

- Classic answer: processes stuck in **uninterruptible sleep (D state)** waiting on I/O — they count toward load. `ps -eo state,pid,cmd | grep "^D"`.
- Check disk: `iostat -x 1` (look at `%util`, `await`), `iotop`. Could be a slow/failing disk or NFS mount hanging.

**"Disk is full / app can't write logs":**

- `df -h` (which mount), `du -sh /path/*` to find the offender, `du -x` to stay on one filesystem.
- Gotcha: **a deleted file still held open by a process** keeps its space until the fd closes — `lsof | grep deleted`. Also **inode exhaustion**: `df -i` (many tiny files, disk "full" with free bytes).
- Fix: rotate/truncate logs (`logrotate`, `truncate -s 0`), not just `rm` on an open file.

**"Out of memory / OOM":**

- `free -h`, `vmstat 1` (watch `si`/`so` swap activity), check `/var/log/messages` or `dmesg` for the **OOM killer**. `smem`, `/proc/<pid>/status` (VmRSS). Distinguish RSS vs cached; Linux "uses" free RAM as page cache (that's fine).

**"Service isn't responding / can't connect":**

- Is the process up? `systemctl status`, `ps`. Is it listening? `ss -tlnp` / `netstat -tlnp`. Right port/interface?
- Reachability: `ping` (L3), `traceroute`/`mtr` (path), `telnet host port` / `nc -zv` (L4), `curl -v` (L7).
- DNS: `dig`/`nslookup`. Firewall: `iptables -L -n` / `nftables`, security groups. `/etc/services` maps names↔ports.
- App logs (`journalctl -u svc -f`, `/var/log/…`), file-descriptor limits (`ulimit -n`, `lsof | wc -l` — "too many open files" is a classic prod outage).

**Network-specific (ties to §6 + the two-datacenter latency question):**

- `mtr host` = continuous traceroute+ping, shows **which hop** loses/adds latency. `ping` for loss%/RTT jitter.
- `tcpdump -i eth0 host X` / Wireshark for retransmits, dup ACKs (→ loss), RST.
- Interface errors/drops: `ip -s link`, `ethtool -S eth0`, `netstat -s` (TCP retransmit counters).
- Intermittent inter-DC latency → suspect: congestion/microbursts on a shared link, a flapping route/BGP, an overloaded middlebox, MTU/PMTUD blackhole, or one bad ECMP path. Bisect by hop with `mtr`, correlate with traffic graphs, check both directions (asymmetric routing).

**"None of the external commands run" (e.g. can't fork, disk full, fd exhausted):**

- Use **shell built-ins** only (`echo`, `cd`, `read`, `type`, globbing) since `/bin/*` won't exec. Signals the interviewer you understand shell vs external binaries.

**General framing to say out loud:** reproduce → check monitoring/dashboards first → USE method across resources → form one hypothesis at a time → least-invasive diagnosis before mutating → mitigate (restore service) _then_ root-cause → write the postmortem. Mention **blameless postmortems, SLOs/error budgets, runbooks** — it signals SRE maturity.

# "SSH in and fix a flaky service"

**Q5. You SSH into a box; a service is up but not responding. First five things you check?**

**Prompt:** "Here's a scenario. You SSH into a box. A service is running — the process is up — but clients are getting errors. No alerts fired. What do you do? Think out loud."
> I'd triage like an incident, hypothesis-driven. First, is it actually listening — `ss -tlnp` on the expected port; a process can be "up" but not bound. Then logs — `journalctl -u <svc> --since "15 min ago"` and the app log for errors/restarts. Resource pressure — `top`, `free -m`, and critically `df -h`, because a full disk silently breaks writes and logging. Is it stuck — `strace -p <pid>` briefly to see if it's blocked in a syscall, or a `D`-state in `top` meaning stuck on I/O; `lsof -p <pid>` to check for FD exhaustion. Then connectivity — `curl localhost:<port>` to isolate app-vs-network, `tcpdump` to confirm requests even arrive. I state a hypothesis before each step and I don't mutate anything without knowing how I'd roll it back.

**Follow-up:** "Say `top` shows the process at 100% CPU but throughput is zero. What's your read?"
> That smells like a **busy-wait or spin** — a tight retry/poll loop burning CPU without making progress, or a livelock. I'd confirm with `perf top -p <pid>` to see the hot function, or `strace`/`py-spy` to see if it's spinning in user space vs hammering a failing syscall (e.g., retrying a connection in a tight loop with no backoff). The fix is usually adding backoff/blocking instead of polling. If you've debugged this class for real (an unbounded retry loop, a reconnect storm), say so — it proves you've owned it.

**Follow-up:** "You mentioned `strace` has overhead. How much, and what would you reach for instead on a hot prod process?"
> `strace` uses `ptrace`, which stops the process on every syscall enter and exit — two context switches per call — so on a syscall-heavy process it can be a 100×-plus slowdown. Brendan Gregg's "strace wow much syscall" post is the canonical warning. On a hot production process I'd use `perf trace` or an eBPF tool (bpftrace) instead, which hook in-kernel and sample/aggregate with far lower overhead, so I'm not adding a latency cliff to the thing I'm debugging.
