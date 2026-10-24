# Linux Performance Playbook

Command-level playbooks: the 60-second checklist, USE per resource, a decision tree, and symptom → fix tables. The reasoning layer on top is in [debugging-method.md](debugging-method.md).

> Merged from 2024–2025 Meta Production Engineer prep notes. Overlapping copies were deduplicated; the newer, denser notes in this folder take precedence where they disagree.

# Part 1 — The 60-second checklist, command by command

(After Brendan Gregg's *Linux Performance Analysis in 60,000 Milliseconds*.)

### Quick Investigation Plan
Run these commands in sequence. Each provides specific insights into system performance:

#### 1. uptime
```bash
$ uptime
```
**What to look for:**
- Load averages for 1min, 5min, and 15min
- High numbers indicate high demand
- Compare values to number of CPUs
- If 1min is lower than 15min, you might have missed the issue
- If 1min is much higher, system load is increasing

#### 2. dmesg | tail
```bash
$ dmesg | tail
```
**What to look for:**
- Out of memory (OOM) killer messages
- TCP dropping packets
- Disk failures
- Hardware errors
- Any other system-level errors

#### 3. vmstat 1
```bash
$ vmstat 1
```
**Key columns:**
- r: Number of processes waiting for CPU
  - If > CPU count = CPU saturation
- free: Free memory (KB)
  - Too many digits = enough memory
- si/so: Swap in/out
  - Non-zero = memory problems
- us/sy/id/wa/st: CPU usage breakdown
  - us = user time
  - sy = system time (>20% needs investigation)
  - id = idle time
  - wa = waiting for I/O (high = disk bottleneck)
  - st = stolen time (virtualization)

#### 4. mpstat -P ALL 1
```bash
$ mpstat -P ALL 1
```
**What to look for:**
- CPU imbalance across cores
- Single hot CPU = possible single-threaded bottleneck
- Compare user/system time per CPU
- High system time on specific CPUs

#### 5. pidstat 1
```bash
$ pidstat 1
```
**What to look for:**
- Which processes are using CPU
- %CPU > 100 means using multiple cores
- High system time in specific processes
- Pattern of CPU usage over time

#### 6. iostat -xz 1
```bash
$ iostat -xz 1
```
**Key metrics:**
- r/s, w/s: Reads/writes per second
- rkB/s, wkB/s: Read/write throughput
- await: Average I/O time (includes queue time)
  - Higher than expected = device saturation
- avgqu-sz: Average queue size
  - >1 might indicate saturation
- %util: Device utilization
  - >60% may indicate problems
  - 100% usually means saturation

#### 7. free -m
```bash
$ free -m
```
**What to look for:**
- Buffers: Block device I/O cache
- Cached: File system page cache
- Should not be near-zero
- "-/+ buffers/cache" shows true memory usage
- Low free memory with high cache = probably fine

#### 8. sar -n DEV 1
```bash
$ sar -n DEV 1
```
**Key metrics:**
- rxkB/s: Receive throughput
- txkB/s: Transmit throughput
- Compare against network interface limits
- %ifutil: Interface utilization (if working)

#### 9. sar -n TCP,ETCP 1
```bash
$ sar -n TCP,ETCP 1
```
**Key metrics:**
- active/s: Locally initiated connections/sec
- passive/s: Remote initiated connections/sec
- retrans/s: Retransmits/sec
  - High retrans = network/server issues
- Look for TCP connection patterns

#### 10. top
```bash
$ top
```
**What to look for:**
- Real-time view of system activity
- CPU consumption by process
- Memory consumption by process
- Process count and states
- Overall system state

### Investigation Strategy
1. Start with high-level metrics (uptime, dmesg)
2. Look for obvious errors first
3. Check saturation metrics (queues, wait times)
4. Examine resource utilization
5. Focus on the intersection of symptoms
6. Use elimination to narrow down issues
7. Deep dive into specific resources showing problems

### Common Problem Patterns

#### CPU Issues:
- High load averages + high 'r' queue = CPU saturation
- High system time = kernel/driver issues
- Single hot CPU = single-threaded bottleneck

#### Memory Issues:
- Non-zero si/so = memory pressure
- High cache usage = normal
- Low cache + high disk I/O = possible memory shortage

#### Disk Issues:
- High await times + high utilization = disk saturation
- High avgqu-sz = too many concurrent requests
- High system time + high disk I/O = possible filesystem issues

#### Network Issues:
- High retrans/s = network reliability problems
- High throughput + normal retrans = normal load
- Many new connections + high CPU = connection handling bottleneck

# Part 2 — The USE method, per resource

### Core Principle
For every system resource, analyze three key metrics:
- **Utilization**: Average time the resource was busy
- **Saturation**: Extra work that can't be serviced (usually queued)
- **Errors**: Count of error events

### Resource Checklist with Commands

#### 1. CPU Resources

##### Utilization Metrics
Command: `mpstat -P ALL 1`
```
CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
all  23.49    0.00   4.26    0.12    0.00    0.12    0.00    0.00    0.00   72.01
```
- Watch %usr + %sys for overall CPU utilization
- >70% sustained utilization needs investigation
- Per-CPU imbalance may indicate threading issues

Command: `top`
```
%Cpu(s): 23.1 us,  4.2 sy,  0.0 ni, 72.4 id,  0.1 wa,  0.0 hi,  0.2 si,  0.0 st
```
- us (user) + sy (system) shows CPU utilization
- wa (I/O wait) indicates disk bottlenecks

##### Saturation Metrics
Command: `vmstat 1`
```
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 4  0      0 810012  65508 483504    0    0     0     0  157  175 25  4 71  0  0
```
- 'r' column > CPU count indicates CPU saturation
- High 'cs' (context switches) may indicate scheduling issues

##### Error Metrics
Command: `dmesg | grep -i error`
- Look for CPU-related errors, thermal throttling
- Hardware-related CPU errors

#### 2. Memory Resources

##### Utilization Metrics
Command: `free -m`
```
              total        used        free      shared  buff/cache   available
Mem:          16384        4096        8192          0        4096        8192
```
- Watch available memory versus total
- buff/cache should not be near-zero

Command: `vmstat 1`
```
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
```
- swpd shows amount of used swap
- free shows available memory

##### Saturation Metrics
Command: `vmstat 1`
- 'si' and 'so' (swap in/out) should be zero
- Non-zero values indicate memory pressure

Command: `sar -B 1`
```
pgpgin/s pgpgout/s   fault/s  majflt/s
   0.00     0.00    105.93      0.00
```
- majflt/s shows major page faults requiring I/O
- High rates indicate memory saturation

##### Error Metrics
Command: `dmesg | grep -i "out of memory"`
- Shows OOM killer activity
- Any occurrences indicate severe memory pressure

#### 3. Storage Devices

##### Utilization Metrics
Command: `iostat -xz 1`
```
Device   r/s   w/s   rkB/s   wkB/s  avgqu-sz  await  svctm  %util
sda     0.00  0.00    0.00    0.00      0.00   0.00   0.00   0.00
```
- %util shows device utilization
- >60% utilization may indicate problems
- avgqu-sz > 1 suggests saturation

##### Saturation Metrics
Command: `iostat -xz 1`
- await > svctm indicates queueing
- avgqu-sz shows average queue length

##### Error Metrics
Command: `smartctl -A /dev/sda`
- Shows disk error counters and health
- Watch for increasing error counts

#### 4. Network Interfaces

##### Utilization Metrics
Command: `sar -n DEV 1`
```
Interface   rxpck/s   txpck/s   rxkB/s   txkB/s
eth0         42.00     42.00   10.00    10.00
```
- rxkB/s and txkB/s show bandwidth usage
- Compare against interface capacity

##### Saturation Metrics
Command: `netstat -s`
- Look for dropped packets
- Retransmitted segments indicate network issues

Command: `sar -n EDEV 1`
```
Interface   rxerr/s   txerr/s   coll/s  rxdrop/s  txdrop/s
eth0          0.00      0.00      0.00     0.00      0.00
```
- Any non-zero values need investigation

##### Error Metrics
Command: `sar -n EDEV 1`
- Watch for rxerr/s and txerr/s
- coll/s shows collision rate

#### 5. Additional System-Wide Checks

##### System-Wide Error Check
Command: `dmesg | tail`
- Shows recent system messages
- Look for errors and warnings

##### Process Issues
Command: `pidstat 1`
```
UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
0         123    0.00    0.00    0.00    0.00    0.00     0  process1
```
- Identifies processes consuming CPU
- Shows per-process CPU utilization patterns

##### I/O Wait Analysis
Command: `iostat -xz 1`
```
avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           23.21    0.00    4.52    0.04    0.00   72.23
```
- %iowait shows time spent waiting for I/O
- High values indicate disk bottlenecks

### Interpretation Guidelines

#### Immediate Action Needed
- Any non-zero error counts
- Saturation metrics > 0
- Utilization consistently >80%
- Increasing queue lengths
- Non-zero swap activity

#### Warning Signs
- Utilization >70%
- Occasional queue length spikes
- Increasing error rates
- High but sporadic I/O wait

#### Normal Conditions
- Zero error counts
- No saturation
- Utilization <70%
- No swap activity
- Low queue lengths

### Best Practices
1. Run commands with 1-second intervals for real-time analysis
2. Compare multiple time windows
3. Create baselines during normal operation
4. Document unreadable metrics
5. Consider time-of-day patterns

### Remember
- Check errors first (fastest to interpret)
- Then check saturation (queueing)
- Then analyze utilization
- Unknown metrics are known unknowns
- Document metrics you couldn't measure

# Part 3 — USE decision tree (what to run next)

### Initial Assessment
First, collect high-level system metrics to identify potential problem areas:

```bash
# Quick system overview
uptime                 # Load averages
dmesg | tail          # Recent system messages
vmstat 1              # Virtual memory statistics
free -m               # Memory usage
```

### 1. CPU Investigation Path
#### Check Utilization
```bash
mpstat -P ALL 1
pidstat 1
top
```

##### If High User Time (>70%)
- **Symptoms**: High %usr in mpstat, high load average
- **Next Steps**:
  1. Identify CPU-intensive processes:
     ```bash
     top -b -n 1 | head -n 20
     ps aux --sort=-%cpu | head -n 10
     ```
  2. Profile specific process:
     ```bash
     perf record -p <PID> -g
     perf report
     ```
  
  ├── If Single Process Dominant
  │   ├── Check if process is misbehaving:
  │   │   ```bash
  │   │   strace -p <PID>
  │   │   ```
  │   └── Consider process-specific optimization
  │
  └── If Multiple Processes
      ├── Check for resource contention:
      │   ```bash
      │   pidstat -w 1    # Context switching
      │   ```
      └── Consider workload distribution

##### If High System Time (>20%)
- **Symptoms**: High %sys in mpstat
- **Next Steps**:
  ```bash
  # Check system calls
  strace -c -p <PID>
  
  # Check kernel functions
  perf top -g
  ```
  
  ├── If I/O Related
  │   └── Follow I/O investigation path
  │
  └── If Network Related
      └── Follow Network investigation path

#### Check Saturation
```bash
vmstat 1              # Check 'r' column
pidstat -w 1          # Context switching
```

##### If High Run Queue ('r' > CPU count)
1. Check scheduling latency:
   ```bash
   perf sched record -- sleep 10
   perf sched latency
   ```

2. Examine process priorities:
   ```bash
   ps -el | grep <priority>
   ```

#### Check Errors
```bash
dmesg | grep -i error
perf record -g -a sleep 10  # Look for throttling
```

### 2. Memory Investigation Path
#### Check Utilization
```bash
free -m
vmstat 1
slabtop                # Kernel memory usage
```

##### If Low Available Memory
- **Symptoms**: Low 'free' and 'available' in free -m
- **Next Steps**:
  1. Check per-process memory usage:
     ```bash
     ps aux --sort=-%mem | head
     ```
  
  2. Analyze memory mappings:
     ```bash
     pmap -x <PID>
     ```

  ├── If Process Memory Leak Suspected
  │   ├── Track RSS growth:
  │   │   ```bash
  │   │   watch -n 1 'ps -o pid,rss,vsz,comm -p <PID>'
  │   │   ```
  │   └── Use memory profiling tools
  │
  └── If System-wide Memory Pressure
      ├── Check page cache usage:
      │   ```bash
      │   cat /proc/meminfo | grep -i active
      │   ```
      └── Consider memory limits/cgroups

#### Check Saturation
```bash
vmstat 1              # Check si/so columns
sar -B 1              # Page faults
```

##### If Swapping Detected
1. Check swap usage per process:
   ```bash
   for file in /proc/*/status ; do awk '/VmSwap|Name/{printf $2 " " $3}END{ print ""}' $file; done | sort -k 2 -n -r | head
   ```

2. Monitor swap activity:
   ```bash
   sar -W 1
   ```

#### Check Errors
```bash
dmesg | grep -i "out of memory"
dmesg | grep -i "memory"
```

### 3. I/O Investigation Path
#### Check Utilization
```bash
iostat -xz 1
```

##### If High Utilization (%util > 80%)
- **Next Steps**:
  1. Identify processes causing I/O:
     ```bash
     iotop -o
     pidstat -d 1
     ```
  
  2. Check I/O patterns:
     ```bash
     biosnoop      # Requires bcc-tools
     ```

  ├── If Random I/O Pattern
  │   └── Consider I/O scheduling optimization
  │
  └── If Sequential I/O
      └── Check for read-ahead settings

#### Check Saturation
```bash
iostat -xz 1         # Check await and avgqu-sz
```

##### If High Queue Depth (avgqu-sz > 1)
1. Check for blocked processes:
   ```bash
   ps aux | grep D
   ```

2. Analyze block device queue:
   ```bash
   cat /sys/block/<device>/queue/scheduler
   ```

#### Check Errors
```bash
smartctl -a /dev/sda
dmesg | grep -i "i/o error"
```

### 4. Network Investigation Path
#### Check Utilization
```bash
sar -n DEV 1
nicstat 1
```

##### If High Bandwidth Usage
1. Identify processes:
   ```bash
   iftop -P
   nethogs
   ```

2. Analyze connections:
   ```bash
   ss -tap
   netstat -anp
   ```

#### Check Saturation
```bash
netstat -s
sar -n TCP,ETCP 1
```

##### If Retransmits Detected
1. Check network errors:
   ```bash
   netstat -s | grep -i retransmitted
   ```

2. Monitor packet drops:
   ```bash
   sar -n EDEV 1
   ```

#### Check Errors
```bash
dmesg | grep -i eth
ethtool -S eth0 | grep -i error
```

### Resolution Strategies

#### For CPU Issues
1. Process Optimization
   - Profile and optimize code
   - Adjust nice values
   - Consider threading improvements

2. Load Distribution
   - Load balancing
   - Process affinity
   - Workload scheduling

#### For Memory Issues
1. Memory Management
   - Implement/adjust memory limits
   - Optimize application memory usage
   - Configure swap appropriately

2. Cache Optimization
   - Tune page cache
   - Adjust kernel memory parameters
   - Implement caching layers

#### For I/O Issues
1. Storage Optimization
   - Adjust I/O scheduler
   - Implement proper buffering
   - Consider SSD/faster storage

2. Access Patterns
   - Optimize read/write patterns
   - Implement proper caching
   - Consider async I/O

#### For Network Issues
1. Network Configuration
   - TCP tuning
   - Buffer size adjustment
   - QoS implementation

2. Application Level
   - Connection pooling
   - Protocol optimization
   - Caching strategies

# Part 4 — Symptom → diagnose → fix

#### CPU Problems

**High Load Average**
- Diagnose: `uptime` or `top`
  - Look for load averages > number of CPU cores
- Solution:
  - Identify high CPU processes: `top -c` or `htop`
  - Use `ps aux --sort=-%cpu` to find CPU-hungry processes
  - Consider process nice levels or limiting with cgroups

**High System Time**
- Diagnose: `top` (look at %sy column)
  - Normal is 5-10%, concerning if >20%
- Solution: 
  - `perf top` to identify kernel functions consuming CPU
  - Check for excessive syscalls: `strace -c -p PID`
  - Optimize application to reduce system calls

**Context Switch Storms**
- Diagnose:
  - `vmstat 1` (look at 'cs' column)
  - `pidstat -w 1` for per-process context switches
- Solution:
  - Reduce thread count
  - Use thread pooling
  - Implement work batching

**Single Core Saturation**
- Diagnose:
  - `mpstat -P ALL 1`
  - `top` press '1' to show per-core usage
- Solution:
  - Enable process/thread affinity with `taskset`
  - Distribute load across cores
  - Check if application supports parallel processing

**Priority Inversions**
- Diagnose:
  - `ps -eo pid,ni,pri,comm`
  - `trace` for priority inheritance issues
- Solution:
  - Adjust process priorities with `nice`/`renice`
  - Enable priority inheritance in mutex locks
  - Review application threading design

#### Memory Problems

**Page Thrashing**
- Diagnose:
  - `vmstat 1` (check si/so columns)
  - `sar -B 1` for page statistics
- Solution:
  - Reduce working set size
  - Increase RAM
  - Adjust swappiness: `sysctl vm.swappiness`

**High Swap Usage**
- Diagnose:
  - `free -h`
  - `swapon -s`
  - `vmstat 1`
- Solution:
  - Add more RAM
  - Identify memory-hungry processes: `smem -k`
  - Tune kernel swapping behavior: `vm.swappiness`

**Memory Leaks**
- Diagnose:
  - `ps aux --sort=-%mem`
  - `pmap -x PID`
  - `valgrind --leak-check=full`
- Solution:
  - Use memory profilers (e.g., Valgrind)
  - Monitor with `top`/`htop` over time
  - Fix application code memory management

**Cache Misses**
- Diagnose:
  - `perf stat -e cache-misses,cache-references -p PID`
  - `perf record -e cache-misses -a`
- Solution:
  - Optimize data structures for cache locality
  - Use cache-aligned structures
  - Profile and optimize memory access patterns

**OOM Kills**
- Diagnose:
  - Check kernel logs: `dmesg | grep -i "out of memory"`
  - Monitor `/var/log/messages`
- Solution:
  - Adjust OOM killer settings
  - Set process memory limits
  - Monitor and restrict memory usage with cgroups

#### I/O Problems

**High Wait Times**
- Diagnose:
  - `iostat -x 1`
  - `iotop`
  - Check %iowait in `top`
- Solution:
  - Use faster storage
  - Optimize I/O patterns
  - Use I/O schedulers appropriately

**Queue Saturation**
- Diagnose:
  - `iostat -x` (check avgqu-sz)
  - `cat /sys/block/device/queue/nr_requests`
- Solution:
  - Adjust queue depths
  - Use better I/O scheduler
  - Consider SSD/faster storage

**Random I/O Patterns**
- Diagnose:
  - `biosnoop` (from bcc-tools)
  - `blktrace`
- Solution:
  - Use readahead where appropriate
  - Implement better buffering
  - Consider switching to SSDs

**Buffer Overflow**
- Diagnose:
  - Check kernel logs: `dmesg`
  - Monitor process limits: `ulimit -a`
- Solution:
  - Increase buffer sizes
  - Implement flow control
  - Use appropriate buffer management

**Journal Bottlenecks**
- Diagnose:
  - `iostat -x`
  - `blktrace` on journal device
- Solution:
  - Move journal to separate device
  - Adjust journal size
  - Consider data=writeback mode

#### Network Problems

**Packet Drops**
- Diagnose:
  - `netstat -s | grep -i "drop"`
  - `tcpdump`
  - `sar -n DEV 1`
- Solution:
  - Increase buffer sizes
  - Check NIC settings
  - Monitor for network congestion

**High Latency**
- Diagnose:
  - `ping`
  - `mtr`
  - `traceroute`
- Solution:
  - Check network configuration
  - Monitor bandwidth usage
  - Consider QoS implementation

**Buffer Bloat**
- Diagnose:
  - `ping` during file transfers
  - `tc -s qdisc show`
- Solution:
  - Implement active queue management (e.g., fq_codel)
  - Adjust txqueuelen
  - Enable ECN

**Connection Tracking Overflow**
- Diagnose:
  - `cat /proc/sys/net/netfilter/nf_conntrack_count`
  - `dmesg | grep conntrack`
- Solution:
  - Increase nf_conntrack_max
  - Enable connection tracking helpers
  - Optimize connection timeout values

**ARP/Routing Issues**
- Diagnose:
  - `ip route show`
  - `arp -an`
  - `traceroute`
- Solution:
  - Check routing tables
  - Verify ARP cache
  - Monitor for duplicate IP addresses

For most of these issues, it's good practice to:
1. Establish baseline metrics during normal operation
2. Use monitoring tools (like Prometheus, Nagios, etc.)
3. Set up alerting for early detection
4. Document troubleshooting procedures
5. Maintain system logs for historical analysis

# Part 5 — Methodologies & tool map (Netflix "Linux Performance Tools" talk)

Notes on Brendan Gregg's Velocity 2015 talk: [video](https://www.youtube.com/watch?v=FJW8nGV4jxY) · [write-up](https://netflixtechblog.com/netflix-at-velocity-2015-linux-performance-tools-51964ddb81cf).

### Methodologies

#### System is Slow

We can keep chasing multiple tails (if no clear methodology) 
	-> top (CPU usage) 
	-> iotop or iostat (I/O usage) 
	-> dstat or sar-n DEV 1 (compare multiple things in the same time)
	-> vmstat -1 or dstat --vmstat (same as above)
	-> strace -p `pgreb process_name`

This is called **Street Light Anti-Method** (keep running familiar tools till u find an obv issue).

#### High Latency

Let's try the USE method
	-> top
	-> vmstat 1
	-> mpstat 1
	-> iostat -x 1
	-> sar -n DEV 1

#### App is taking forever

	-> vmstat 1
	-> mpstat 1
	-> pidstat 1

Here assume we can find a lot of user time and system time by the process, now we check why is this
	-> iostat -x 1 
Answer is no, are we using network devices?
	sar -n DEV 1
Answer is no. 

Now we have three observations, we have a lot of system time and CPU usage but the disks and network are idle.

Are we swapping?
	-> vmstat 1
No.

Now, I think we should check the system calls, for this we can use strace

	strace -tp `pgrep lab003` 2>&1 | head -100
	
Here we can see it is calling read with 0 bytes 

#### CPU Consumer

	-> top
Found that CPU is busy, but the process that is using it is not available
	->pstat 1
same thing
	-> iostat -x 1
not disk io
	-> sar -n DEV 1
not network io
	-> mpstat 1
it is not paging or swapping
	-> vmstat 1
cpus are busy
	-> perf record -F 99 -a -g -- sleep 10 
	-> perf report -n --stdio
82% of time is in checksum command, so that loop is the cause  
#### Problem Statement Method

1. What makes you _think_ there is a performance problem?
2. Has this system _ever_ performed well?
3. What has _changed_ recently? (Software? Hardware? Load?)
4. Can the performance degradation be expressed in terms of _latency_ or run time?
5. Does the problem affect _other_ people or applications (or is it just you)?
6. What is the _environment_? Software, hardware, instance types? Versions? Configuration?

#### Workload Characterization Method

1. Who is causing the load? PID, UID, Ip addr, ...
2. Why is the load called? code path, stack trace
3. What is the load? IOPS, tput, type, r/w
4. How is the load changing over time?

#### The USE Method

#### Off-CPU Method

#### CPU Profile Method

Flame Graph 

#### RTFM Method

### Tools

#### Observability

Watch activity

1. **uptime**
	 print load averages (1 min - 5 min - 15 min)
	 Load > # of CPUs => CPU saturation
2. **top/ htop**
	Can miss short-lived processes (atop won't)
3. **ps**
	Process status listing
	Custom Fields 
4. **vmstat**
	Virtual memory statistics and more 
	first column (r) is runnable tasks
	CPU columns are splitted to user time and system time (what kernels do: syscalls, interrupts, ..)
5. **iostat**
	Block I/O (disk) stats
	How often an application had errors
	
	To identify bottlenecks using `iostat`:
	
	- **High `%iowait`**: Indicates a disk I/O bottleneck. Check disk utilization (`%util`) and response times (`await`, `svctm`).
	- **High `%user` or `%system`**: Indicates a CPU bottleneck. Check if the CPU is fully utilized.
	- **High Disk Utilization (`%util`)**: Indicates that the disk is heavily used and may be a bottleneck.
	- **Long I/O Response Times (`await`, `svctm`)**: Indicates slow disk performance.
1. mpstat
	Multi-processor statistics, per-CPU
	Looking for unbalanced workloads, hot CPUs
7. free
	Main memory usage
	buffers: block device I/O cache
	cached: virtual pages caching
8. iptables
	1. List route table
---
8. strace
	System call tracer
	-ttt time (us) since epoch; -T syscall time
	Has a massive overhead
9. tcpdump
	Trace of the network packets for port analysis
10. netstat
	1. multi-tool: -i (interface stats), -r (route table), default (list conns)
	2. netstat -p: shows process details
	3. per second interval with -c
11. nicstat
	1. Network interface stats, iostat-like output
12. pidstat
	1. Process stats by thread, disk I/O
13. swapon -s
	1. Swap usage
14. lsof
	1. File descriptor usage, current active network connections
	2. help understand the environment
15. sar
	1. System activity reporter  
---
16. ss
	1. More socket statistics
17. iptraf
	1. histogram of network package sizes 
18. iotop
	1. block device i/o by process
19. slabtop
	1. For memory issues related to kernel, it gets kernel slab allocator memory usage
20. pcstat
	1. Shows page cache residency by file
	2. useful for database performance analysis
21. perf_events (perf command) (important)
	1. cpu profiling, PMC profiling, static & dynamic tracing
	2. 
	3. 
	4. 
	5. 
22. titop
	1. has IPC by process, %MISS, %BUS
	2. how effective the CPU subsystem with some process
	3. needs PMC enables
23. rdmsr
24. /dev/proc
#### Benchmarking

Load test. Error prone, you benchmark A, actually measure B, conclude you measure C.

Common Mistakes include:
1. Testing wrong target: eg, FS cache instead of disk
2. Choosing wrong target: eg, disk instead of FS cache .. doesn't resemble real world usage
3. Invalid results: eg, bugs

Active Benchmarking Method
1. run benchmark for hours
2. while running, analyze and confirm performance limiter using observability tools
	1. Disk benchmark: run iostat, ...
	2. CPU benchmark: run pidstat, perf, flame graphs, ...
3. Answer the question: why isn't the result 10x

Tools:
1. lmbench
	1. cpu, memory, kernel micro-benchmarks
	2. eg, memory latency by stride size:
	   lat_mem_rd 100m 128 > out.latencies
2. fio
	1. FS or disk I/O micro-benchmarks
	3. results include basic latency distribution
3. pchar
	1. traceroute with bandwidth per hop

#### Tuning

Change.

General interface: sysctl, /sys

Specific Method:
1. Question
2. Hypothesis
3. Prediction
4. Test
5. Analysis

Any observational or benchmarking tests you can try before tuning?

#### Static

Check Configuration.

- CPU types & flags
- CPU frequency scaling config
- Storage devices
- File system capacity
- File system and volume configuration
- Route table
- State of hardware
- etc.

Basically what can be checked on a system without load

Examples:
1. CPU Types & Flags `cat /proc/cpuinfo`
2. CPU Frequency Scaling `cat /sys/.../cpu0/cpufreq/scaling_governer`
3. Storage devices: `cat /proc/scsi/scsi` & `lsscsi`
4. Routing Table `netstat -rn` & `ip route get 1.1.1.1`
5. System Messages: `dmesg`
6. Network Interface Config: `ifconfig -a` & `ip link`
7. File System Capacity: `df -h`
8. Volume Config: `mdadm --misc -D /dev/md0`
9. Storage Device Info: `smartctl`
10. NUMA Config: `numactl -s` & `numactl -H`
11. PCI Info: `lspci`
12. Installed Kernel Modules: `lsmod`
13. Root Crontab Config: `crontab -l`
14. Services: `service --status-all`
15. DNS records: `dig RECORDTYPE example.com` or `nslookup domain.com`

#### Tracing

- kprobes: dynamic kernel tracing
- uprobes: dynamic user-level tracing
- opensnoop: trace open() syscalls showing filenames
- tpoint: who is creating disk I/O and of what type
-  sysdig (replacement for strace or perf trace) currently only syscalls and user-level processing
- Present: ftrace & perf
---
### Profiling

#### CPU Profiling

Record stacks at a timer interval.

perf_events is good for this.

Flame graph is OP

# Part 6 — The storage & I/O stack

## Linux Storage & I/O System Concepts

### 1. Storage Stack Overview

#### Architecture
The Linux storage stack is organized in layers:
```
Application Layer
    ↓
VFS (Virtual File System)
    ↓
File System Layer (ext4, xfs, etc.)
    ↓
Page Cache
    ↓
Block I/O Layer
    ↓
I/O Scheduler
    ↓
Device Driver
    ↓
Hardware
```

#### Key Components

##### Page Cache
- Central component for file system performance
- Caches file data and metadata in memory
- Uses LRU (Least Recently Used) algorithm with variations
- Managed through `pdflush` or newer `flusher` threads

Failure Modes:
- Memory pressure causing thrashing
- Dirty page writeback storms
- Cache coherency issues in distributed systems

Debugging:
```bash
# Check page cache statistics
$ cat /proc/meminfo | grep -i page
# Monitor page cache hit rates
$ pcstat /path/to/file
# Check dirty page parameters
$ sysctl -a | grep vm.dirty
```

Performance Implications:
- Write-back vs Write-through caching
- Read-ahead and prefetching behavior
- Memory pressure vs I/O performance tradeoffs

##### Block I/O Layer

Core Components:
- bio: Basic I/O unit
- Request Queue: Manages I/O requests
- Elevator: I/O scheduling algorithm

Debugging Tools:
```bash
# Monitor block I/O statistics
$ iostat -xz 1
# Trace block I/O operations
$ blktrace /dev/sda
# Check queue parameters
$ cat /sys/block/sda/queue/scheduler
```

Common Interview Questions:
1. "How would you investigate high I/O latency?"
Strong Answer: "I would:
- First check iostat for device utilization and queue depths
- Use blktrace to analyze I/O patterns
- Check for page cache effectiveness with pcstat
- Monitor system calls with strace
- Look for signs of filesystem fragmentation
- Consider if write barriers or journaling are causing delays"

2. "Explain how write barriers work and their performance impact"
Strong Answer: "Write barriers ensure storage consistency by:
- Forcing pending writes to disk
- Maintaining write ordering for journaling
- Impact includes increased latency but better data integrity
- Can be especially impactful in virtualized environments
- Modern filesystems use techniques like group commits to minimize impact"

### 2. File System Internals

#### Key Structures

##### Inodes
- Store metadata about files
- Fixed size per filesystem
- Contains:
  - File permissions
  - Owner/group IDs
  - Size and timestamps
  - Block pointers

Debugging:
```bash
# Check inode usage
$ df -i
# Find files by inode
$ find . -inum <inode_number>
# Show detailed inode information
$ debugfs -R 'stat <inode_number>' /dev/sda1
```

##### Journal
Purpose:
- Maintains filesystem consistency
- Prevents corruption after crashes
- Provides atomic updates

Modes:
1. Data + Metadata journaling
2. Ordered journaling
3. Writeback journaling

Performance Implications:
- Journal size affects recovery time
- Write amplification from journaling
- Memory usage for journal buffer
### 3. I/O Scheduling

#### Scheduler Types

##### CFQ (Completely Fair Queuing)
- Time-sliced I/O scheduling
- Per-process I/O queues
- Suitable for desktop workloads

##### Deadline
- Prevents request starvation
- Separate read/write queues
- Enforces latency guarantees

##### Noop
- Simple FIFO queueing
- Minimal overhead
- Good for SSDs

Performance Analysis:
```bash
# Check current scheduler
$ cat /sys/block/sda/queue/scheduler
# Monitor I/O latency distribution
$ iostat -xz 1
# Analyze I/O patterns
$ iotop -o
```

#### Resource Management

##### Control Groups (cgroups)
- I/O bandwidth control
- Per-group accounting
- Priority management

Configuration:
```bash
# Set I/O weight for group
$ echo 100 > /sys/fs/cgroup/blkio/group1/blkio.weight
# Monitor group I/O stats
$ cat /sys/fs/cgroup/blkio/group1/blkio.throttle.io_service_bytes
```

Real-world Applications:
- Container I/O isolation
- Service quality guarantees
- Resource fair sharing

### 4. Direct I/O and Async I/O

#### Direct I/O
- Bypasses page cache
- Requires aligned buffers
- Used for:
  - Database engines
  - Streaming workloads
  - Custom caching layers

Implementation:
```c
fd = open("file", O_DIRECT);
// Buffer must be aligned
posix_memalign(&buf, 512, size);
```

#### Async I/O
- Non-blocking I/O operations
- Event notification based
- Supports:
  - I/O polling
  - Completion queues
  - Batch processing

Linux AIO Implementation:
```c
io_context_t ctx;
io_setup(128, &ctx);
struct iocb iocb;
io_prep_pread(&iocb, fd, buf, len, offset);
io_submit(ctx, 1, &iocbs);
```

Performance Considerations:
- CPU usage vs I/O throughput
- Memory bandwidth impact
- Interrupt handling overhead

# Part 7 — Talking about performance in interviews

## System Performance Engineering Interview Guide

### Key Terms & Their Implications

#### Resource Constraints
1. **CPU-bound**
   - What it means: Process/application limited by CPU processing power
   - Shows expertise when you mention: "Profile revealed high CPU utilization but low I/O wait times"
   - Advanced follow-up: "Investigated scheduler run queue length to confirm CPU saturation"

2. **I/O-bound**
   - What it means: Process limited by disk or network operations
   - Shows expertise when you mention: "High iowait times in vmstat indicated disk bottleneck"
   - Advanced follow-up: "Used iostat to confirm high device utilization and queue lengths"

3. **Memory-bound**
   - What it means: Process limited by available RAM
   - Shows expertise when you mention: "Observed increased page faults and swap activity"
   - Advanced follow-up: "Monitored PSI (Pressure Stall Information) metrics for memory contention"

#### Performance Metrics

4. **Latency**
   - What it means: Time taken to complete a single operation
   - Shows expertise when you mention: "P99 latency spike indicated tail latency issues"
   - Advanced follow-up: "Used histogram analysis to identify latency distribution patterns"

5. **Throughput**
   - What it means: Operations completed per unit time
   - Shows expertise when you mention: "Optimized for throughput by increasing batch size"
   - Advanced follow-up: "Balanced throughput against latency requirements"

6. **Saturation**
   - What it means: Degree to which a resource can't handle additional work
   - Shows expertise when you mention: "Identified saturation through queue length analysis"
   - Advanced follow-up: "Used Little's Law to analyze relationship between throughput and queue length"

#### System Analysis Terms

7. **Resource Contention**
   - What it means: Multiple processes competing for same resource
   - Shows expertise when you mention: "Used perf to identify lock contention patterns"
   - Advanced follow-up: "Implemented resource isolation using cgroups"

8. **Noisy Neighbor**
   - What it means: Other workloads interfering with service performance
   - Shows expertise when you mention: "Identified interference through CPU steal time analysis"
   - Advanced follow-up: "Implemented workload isolation strategies"

9. **Hot Spots**
   - What it means: Specific components causing disproportionate load
   - Shows expertise when you mention: "Profiling revealed uneven shard distribution"
   - Advanced follow-up: "Implemented consistent hashing to better distribute load"

### Interview Questions & Strong Answers

#### Q1: "How would you debug a slow service?"
Strong answer components:
- "First, establish baseline metrics using USE methodology"
- "Check for error conditions in dmesg and application logs"
- "Analyze saturation metrics across CPU, memory, and I/O"
- "Use profiling tools to identify specific bottlenecks"
- "Consider both vertical and horizontal scaling solutions"

#### Q2: "What metrics would you monitor in production?"
Strong answer components:
- "Resource utilization metrics using RED methodology (Rate, Errors, Duration)"
- "Saturation metrics to predict impending issues"
- "Error rates and error budget consumption"
- "SLI/SLO compliance metrics"
- "Business impact metrics tied to user experience"

#### Q3: "How would you investigate a memory leak?"
Strong answer components:
- "Monitor RSS and VSZ growth patterns"
- "Use pmap for process memory mapping analysis"
- "Employ heap profiling tools"
- "Analyze GC patterns and frequency"
- "Correlate with application-specific metrics"

#### Q4: "Describe a performance issue you've solved"
Structure your answer with:
- "Used systematic approach starting with USE method"
- "Identified bottleneck through metric analysis"
- "Confirmed root cause through targeted profiling"
- "Implemented solution with clear success metrics"
- "Established monitoring to prevent recurrence"

### Advanced Concepts to Mention

#### Performance Analysis
- **Flame Graphs**: "Used flame graphs to visualize CPU call stacks"
- **Heat Maps**: "Applied latency heat maps to identify patterns"
- **Profiling Overhead**: "Considered observer effect in profiling strategy"

#### System Internals
- **Page Cache**: "Optimized file I/O patterns for page cache efficiency"
- **TCP Tuning**: "Adjusted TCP parameters for workload characteristics"
- **Scheduler Behavior**: "Analyzed CFS behavior under load"

#### Architecture Patterns
- **Back Pressure**: "Implemented back pressure to prevent cascade failures"
- **Circuit Breaking**: "Added circuit breakers for fault isolation"
- **Load Shedding**: "Designed graceful degradation mechanisms"

### Red Flags to Avoid
1. Don't just say "high CPU usage" - specify user vs. system time
2. Avoid suggesting "add more RAM" without evidence
3. Don't mention tools without explaining why they're relevant
4. Avoid suggesting horizontal scaling without analyzing bottlenecks

### Bonus Points
1. Mention conducting chaos engineering experiments
2. Discuss implementing observability from ground up
3. Talk about capacity planning methodologies
4. Describe automated performance testing pipelines
5. Share experience with distributed tracing
