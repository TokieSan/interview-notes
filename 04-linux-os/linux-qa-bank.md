# Linux / OS Q&A Bank

A long list of question → short model answer. Good for a final self-quiz: read the question, answer out loud, then check.

> Merged from 2024–2025 Meta Production Engineer prep notes. Overlapping copies were deduplicated; the newer, denser notes in this folder take precedence where they disagree.

# OS & Linux internals

### Process Management

1. **What is a process in Linux?**
   A process is an instance of a running program that consists of the program code, its current activity, and a set of system resources. Each process has its own memory space, system resources, and security context.

2. **Explain the fork() system call.**
   fork() creates a new process by duplicating the calling process. The new process (child) is an exact copy of the parent process, including all memory segments, but has a unique process ID. After fork(), both processes execute the next instruction following the fork() call. The parent receives the child's PID as the return value, while the child receives 0.

3. **What happens during a context switch?**
   A context switch occurs when the kernel switches from one process to another. The kernel saves the current process's state (CPU registers, program counter, stack pointer) to memory, loads the saved state of the next process, and gives control to that process. Context switches are expensive operations due to the overhead of saving/restoring state and cold CPU caches.

4. **What are the different states a process can be in?**
   - Running: Currently executing on CPU
   - Ready/Runnable: Ready to run, waiting for CPU time
   - Sleeping/Waiting: Waiting for I/O or some event
   - Zombie: Process has terminated but parent hasn't read its exit status
   - Stopped: Process has been stopped, usually by receiving a signal

5. **Explain Copy-on-Write (CoW).**
   Copy-on-Write is an optimization technique used during fork(). Instead of copying all memory pages immediately, the parent and child initially share the same physical pages marked as read-only. When either process tries to modify a page, a page fault occurs, and only then is the page copied. This significantly reduces memory usage and improves fork() performance.

### Memory Management

6. **What is virtual memory and how does it work?**
   Virtual memory is an abstraction that provides each process with its own continuous address space, isolating processes from each other and from physical memory details. The MMU (Memory Management Unit) translates virtual addresses to physical addresses using page tables.

7. **Explain the concept of page faults.**
   A page fault occurs when a program tries to access a page in its virtual address space that isn't currently mapped to physical memory. There are three types:
   - Minor: The page exists in memory but isn't mapped
   - Major: The page needs to be loaded from disk
   - Invalid: The program tried to access an invalid address

8. **What is thrashing and how can it be prevented?**
   Thrashing occurs when a system spends more time paging than executing processes. It happens when physical memory is overcommitted, causing constant page faults. Prevention methods include:
   - Adding more RAM
   - Reducing working set sizes
   - Implementing better page replacement algorithms
   - Using swap prefetching
   - Implementing proper process scheduling

9. **Explain the buddy memory allocation system.**
   The buddy system is a memory allocation algorithm used in the Linux kernel. It works by dividing memory into fixed-size blocks that are powers of 2. When memory is requested, it finds the smallest block that can accommodate the request. If a block is too large, it's split into two "buddies". When memory is freed, if both buddies are free, they're merged back together.

10. **What is the difference between swapping and paging?**
    Swapping moves entire processes between main memory and disk, while paging moves individual pages. Paging is more flexible and efficient as it allows partial process residence in memory. Modern Linux systems primarily use paging.

### File Systems

11. **What is an inode and what information does it store?**
    An inode (index node) is a data structure that stores metadata about a file, including:
    - File permissions
    - Owner and group IDs
    - File size
    - Timestamps (access, modification, change)
    - Number of hard links
    - Block addresses for file data
    - File type
    It doesn't store the filename, which is stored in directory entries.

12. **Explain journaling in file systems.**
    Journaling is a technique that maintains a log (journal) of changes before committing them to the main file system. If a system crash occurs during writes, the journal can be replayed to maintain consistency. Different journaling modes exist:
    - Data + Metadata journaling (highest safety, lowest performance)
    - Metadata-only journaling (good balance)
    - Writeback mode (highest performance, lowest safety)

### Process Scheduling

13. **Explain the Linux scheduling classes and their priorities.**
    Linux uses multiple scheduling classes:
    - SCHED_FIFO: First-in-first-out real-time scheduler
    - SCHED_RR: Round-robin real-time scheduler
    - SCHED_OTHER (or SCHED_NORMAL): Default time-sharing scheduler
    - SCHED_BATCH: For CPU-intensive batch processes
    - SCHED_IDLE: For extremely low priority tasks
    Priorities range from 0-99 for real-time tasks and 100-139 for normal tasks (mapped to nice values -20 to +19).

14. **What is the Completely Fair Scheduler (CFS)?**
    CFS is Linux's default scheduler that aims to provide fair CPU time to all processes. It:
    - Uses a red-black tree to track runnable processes
    - Assigns CPU time proportionally based on nice values
    - Maintains "virtual runtime" for each process
    - Uses scheduling latency target and minimum granularity
    - Provides group scheduling via cgroups

15. **How does load balancing work in multi-core systems?**
    Linux performs load balancing across CPU cores through:
    - Per-CPU run queues
    - Periodic load balancing at different scheduling domains
    - CPU affinity considerations
    - NUMA awareness
    - Power management considerations
    - Cache locality optimization

16. **What is the purpose of nice values?**
    Nice values (-20 to +19) determine process scheduling priority:
    - Lower values mean higher priority
    - Default is 0
    - Only root can assign negative values
    - Affects CPU time allocation in CFS
    - Translates to time slice calculations

17. **Explain real-time scheduling in Linux.**
    Real-time scheduling provides deterministic response times:
    - SCHED_FIFO: Runs until blocked or yields
    - SCHED_RR: Similar to FIFO but with time slices
    - Preempts normal processes
    - Uses priority levels 0-99
    - Requires special privileges to use
    - Can cause system lockup if misused

### System Calls and Kernel Interfaces

18. **What happens during a system call?**
    System call execution involves:
    - User space prepares arguments
    - Triggers software interrupt (e.g., int 0x80 or syscall)
    - CPU switches to kernel mode
    - Kernel validates arguments
    - Executes system call handler
    - Returns result to user space
    - CPU returns to user mode

19. **Explain the difference between user space and kernel space.**
    - User space:
      - Restricted access to hardware
      - Limited privileges
      - Process isolation
      - Can be swapped out
    - Kernel space:
      - Full hardware access
      - Highest privileges
      - Shared among all processes
      - Always resident in memory

20. **What is the role of the vDSO (virtual dynamic shared object)?**
    vDSO optimizes system calls by:
    - Providing user space mapping of kernel routines
    - Avoiding context switches for common operations
    - Implementing fast system calls (e.g., gettimeofday)
    - Improving performance for frequently used calls
    - Supporting CPU-specific optimizations

21. **Explain the clone() system call.**
    clone() is a flexible version of fork() that allows:
    - Sharing specific resources between parent and child
    - Creating threads (when sharing address space)
    - Controlling what is shared (memory, file descriptors, signals)
    - Implementing threading libraries
    - Setting child termination signals

22. **What is the purpose of futex (Fast Userspace muTEX)?**
    Futex provides fast user-space synchronization:
    - Atomic operations in user space
    - Kernel intervention only for contention
    - Basic building block for pthread mutexes
    - Supports both shared and private operations
    - Efficient for uncontended cases

### Inter-Process Communication (IPC)

23. **Compare different IPC mechanisms in Linux.**
    Linux provides several IPC mechanisms:
    - Pipes: Unidirectional communication between related processes
    - FIFOs: Named pipes for unrelated processes
    - Message Queues: Structured message passing
    - Shared Memory: Fastest but requires synchronization
    - Semaphores: For synchronization and mutual exclusion
    - Unix Domain Sockets: For local communication
    Each has specific use cases and trade-offs.

24. **How does shared memory work in Linux?**
    Shared memory implementation includes:
    - Creating/attaching via shmget()/shmat()
    - Page table mappings to same physical memory
    - Persistence until explicitly removed
    - Need for external synchronization
    - Support for POSIX and System V interfaces

25. **Explain the purpose and working of semaphores.**
    Semaphores provide synchronization through:
    - Atomic P (wait) and V (signal) operations
    - Counter for resource availability
    - Queuing of waiting processes
    - Support for both binary and counting types
    - System V and POSIX implementations

### Network Stack

26. **Describe the Linux network stack layers.**
    The network stack consists of:
    - Socket Interface (SOCK_STREAM, SOCK_DGRAM, etc.)
    - Transport Layer (TCP/UDP implementation)
    - Network Layer (IP routing, firewalling)
    - Link Layer (Device drivers, queuing)
    - Hardware Layer (NIC interaction)

27. **What is the sk_buff structure?**
    sk_buff is the fundamental network buffer structure:
    - Contains packet data and metadata
    - Manages data between layers
    - Supports zero-copy operations
    - Handles protocol headers
    - Maintains network buffer queues

28. **Explain TCP connection states in Linux.**
    TCP states include:
    - LISTEN: Waiting for connection
    - SYN_SENT: Client initiated connection
    - SYN_RECEIVED: Server received SYN
    - ESTABLISHED: Connection established
    - FIN_WAIT, CLOSE_WAIT: Connection termination
    - TIME_WAIT: Waiting for late packets

### Memory Management (Advanced)

29. **How does Transparent Huge Pages (THP) work?**
    THP automatically uses larger page sizes:
    - Default 2MB instead of 4KB pages
    - Reduces TLB misses
    - Supports automatic promotion/demotion
    - Can be configured per-application
    - May cause latency spikes during compaction

30. **Explain the NUMA memory system.**
    NUMA (Non-Uniform Memory Access):
    - Different memory access times based on location
    - Memory node affinity to CPUs
    - Local vs remote access considerations
    - Automatic and manual policy control
    - Supports memory migration

### File Systems (Advanced)

31. **How does ext4 handle delayed allocation?**
    Delayed allocation in ext4:
    - Postpones block allocation until write-out
    - Improves block allocation decisions
    - Reduces fragmentation
    - Increases performance
    - Can lead to larger data loss on crashes

32. **Explain the page cache and its interaction with file systems.**
    Page cache:
    - Caches file contents in memory
    - Implements read-ahead
    - Handles write-back
    - Supports mmap operations
    - Uses LRU-like eviction

### Security and Access Control

33. **How does capabilities system work in Linux?**
    Capabilities break down root privileges:
    - Fine-grained access control
    - Inheritable vs effective capabilities
    - Per-thread capability sets
    - Filesystem capability attributes
    - Reduced risk compared to full root

34. **Explain seccomp (Secure Computing Mode).**
    seccomp restricts system calls:
    - Filters based on rules
    - Uses BPF programs
    - Supports whitelist/blacklist
    - Per-thread restrictions
    - Used in container security

### Performance and Debugging

35. **How does ftrace work?**
    ftrace is a kernel tracing framework:
    - Traces function calls
    - Measures latencies
    - Supports dynamic tracing
    - Provides various trace types
    - Minimal overhead design

### Advanced Topics

36. **Explain how eBPF works in Linux.**
    Extended Berkeley Packet Filter (eBPF):
    - In-kernel virtual machine
    - Safe program execution
    - Dynamic tracing and monitoring
    - Network packet filtering
    - Performance analysis capabilities
    - Security enforcement

37. **How does the Out-of-Memory (OOM) killer work?**
    The OOM killer:
    - Activates when memory is exhausted
    - Scores processes based on memory usage
    - Considers process priority
    - Kills highest-scored process
    - Logs decision in kernel messages
    - Can be tuned via /proc

38. **Explain CPU frequency scaling in Linux.**
    CPU frequency scaling involves:
    - Multiple operating frequencies
    - Different governors (performance, powersave, ondemand)
    - Thermal and power constraints
    - ACPI interface
    - User-space control options

39. **How does the virtual file system (VFS) layer work?**
    VFS provides a common interface:
    - Abstract interface for all filesystems
    - Common file operations
    - Inode and dentry caching
    - Unified mount system
    - Support for network filesystems

40. **Explain kernel preemption.**
    Kernel preemption allows:
    - Interrupting kernel code
    - Better real-time response
    - Reduced latency
    - Configurable preemption models
    - Priority inheritance support

### Debugging and Troubleshooting

41. **What tools are available for kernel debugging?**
    Kernel debugging tools include:
    - kgdb: Remote kernel debugging
    - printk: Kernel messages
    - ftrace: Function tracing
    - perf: Performance analysis
    - systemtap: Dynamic instrumentation
    - crash: Post-mortem analysis

42. **How to handle kernel panics?**
    Kernel panic handling involves:
    - Capturing panic information
    - Analyzing crash dumps
    - Using kdump/kexec
    - Reading oops messages
    - Understanding call traces

43. **Explain kernel profiling techniques.**
    Kernel profiling uses:
    - perf events
    - Hardware performance counters
    - Sampling-based profiling
    - Function tracers
    - Timeline recording

### Modern Features

44. **How do cgroups work?**
    Control Groups (cgroups):
    - Resource limitation
    - Prioritization
    - Accounting
    - Process control
    - Hierarchical organization
    - Multiple subsystems (cpu, memory, io, etc.)

45. **Explain namespace isolation.**
    Namespaces isolate system resources:
    - PID namespaces
    - Network namespaces
    - Mount namespaces
    - UTS namespaces
    - User namespaces
    - IPC namespaces

# Networking fundamentals

1. **What happens when you type a URL in a browser and press Enter?**
   - DNS lookup to resolve domain name to IP
   - TCP three-way handshake
   - TLS/SSL handshake if HTTPS
   - HTTP request sent to server
   - Server processes request and sends response
   - Browser renders the content

2. **Explain the TCP 3-way handshake process.**
   - Client sends SYN packet with initial sequence number
   - Server responds with SYN-ACK packet with acknowledgment number and its own sequence number
   - Client sends ACK packet confirming the connection

3. **What is the difference between TCP and UDP?**
   - TCP: Connection-oriented, reliable, ordered delivery, flow control
   - UDP: Connectionless, unreliable, no ordering, no flow control, faster

4. **What is a subnet mask and what is its purpose?**
   - Determines which portion of an IP address identifies the network and which identifies the host
   - Example: 255.255.255.0 (or /24) means first 24 bits are network portion

5. **Explain the difference between public and private IP addresses.**
   - Public IPs are globally unique and routable on the internet
   - Private IPs (10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16) are for internal networks

6. **What is ARP and what is its purpose?**
   - Address Resolution Protocol
   - Maps IP addresses to MAC addresses on a local network
   - Broadcasts "Who has this IP?" to get MAC address response

7. **What is the purpose of ICMP?**
   - Internet Control Message Protocol
   - Used for network diagnostics and error reporting
   - Examples: ping, traceroute

8. **Explain the difference between a switch and a router.**
   - Switch: Layer 2 device, forwards frames based on MAC addresses within a LAN
   - Router: Layer 3 device, routes packets between different networks based on IP addresses

9. **What is NAT and why is it used?**
   - Network Address Translation
   - Maps private IP addresses to public IPs
   - Helps conserve IPv4 addresses and adds security

10. **What is a VLAN and what problem does it solve?**
    - Virtual LAN
    - Logically segments a network regardless of physical topology
    - Improves security and reduces broadcast domain size

# Linux command line

### Linux Command Line

21. **How do you check system resource usage?**
    - `top` or `htop`: Process activity
    - `free`: Memory usage
    - `df`: Disk usage
    - `iostat`: I/O statistics

22. **What is the purpose of the /proc filesystem?**
    - Virtual filesystem providing process and system information
    - Contains runtime system information
    - Used by many monitoring tools

23. **How do you find the process ID using most CPU?**
    ```bash
    ps aux --sort=-%cpu | head -n 2
    # or
    top -b -n 1 | head -n 8
    ```

24. **How do you check open network connections?**
    ```bash
    netstat -tuln
    # or
    ss -tuln
    ```

25. **What is the difference between soft and hard links?**
    - Soft links: Point to file path, can cross filesystems
    - Hard links: Point to same inode, must be on same filesystem

26. **How do you check disk I/O statistics?**
    ```bash
    iostat -dx
    # or
    iotop
    ```

27. **What does the sticky bit do?**
    - Prevents deletion of files in directory by non-owners
    - Common on /tmp directory
    - Set with chmod +t

28. **How do you find files modified in last 24 hours?**
    ```bash
    find /path -mtime -1
    ```

29. **What is the purpose of /etc/fstab?**
    - Configures filesystem mounts
    - Loaded during boot
    - Defines mount points, options, and fsck order

30. **How do you check system logs?**
    ```bash
    journalctl
    # or
    tail -f /var/log/syslog
    ```

# Linux troubleshooting

### Linux Troubleshooting

31. **What steps would you take to troubleshoot high CPU usage?**
    1. Check load average with `uptime`
    2. Use `top/htop` to identify processes
    3. Check process threads with `ps -eLf`
    4. Use `perf` for detailed analysis

32. **How would you investigate a disk space issue?**
    1. `df -h` to check filesystem usage
    2. `du -sh /*` to find large directories
    3. `lsof | grep deleted` for held deleted files
    4. Check for large log files

33. **What would you check if a system is running slowly?**
    1. Load average (uptime)
    2. Memory usage (free)
    3. Disk I/O (iostat)
    4. Network activity (netstat)
    5. Process activity (top)

34. **How would you troubleshoot a network connectivity issue?**
    1. Check physical connectivity
    2. Verify IP configuration
    3. Test DNS resolution
    4. Check routing table
    5. Use ping/traceroute

35. **What would you do if a filesystem is mounted read-only?**
    1. Check dmesg for errors
    2. Run fsck on unmounted filesystem
    3. Check for hardware errors
    4. Verify mount options

36. **How would you investigate high memory usage?**
    1. Check free memory with `free`
    2. Use `top` to identify processes
    3. Check swap usage
    4. Examine process memory maps

37. **What steps would you take if a service won't start?**
    1. Check service status
    2. Examine service logs
    3. Verify configuration
    4. Check dependencies
    5. Test manually

38. **How would you troubleshoot SSH connection issues?**
    1. Verify network connectivity
    2. Check SSH service status
    3. Examine auth logs
    4. Test SSH configuration
    5. Verify permissions

39. **What would you check if system boot is slow?**
    1. Check systemd boot analysis
    2. Examine boot logs
    3. Check service dependencies
    4. Look for failed services

40. **How would you investigate a process eating memory?**
    1. Use `top` to identify process
    2. Check `/proc/<pid>/status`
    3. Examine memory maps
    4. Use `pmap` for detailed analysis

# System administration

### System Administration

41. **What is systemd and what are its main features?**
    - Init system and service manager
    - Parallel service startup
    - Dependencies management
    - Service supervision
    - Logging (journald)

42. **Explain file permissions in Linux.**
    - Read (4), Write (2), Execute (1)
    - User, Group, Others
    - Special permissions: SUID, SGID, Sticky bit

43. **What is SELinux?**
    - Security Enhanced Linux
    - Mandatory Access Control system
    - Provides additional security layer
    - Uses security contexts

44. **What is RAID and what are common RAID levels?**
    - Redundant Array of Independent Disks
    - RAID 0: Striping
    - RAID 1: Mirroring
    - RAID 5: Striping with parity
    - RAID 10: Mirror of stripes

45. **What is LVM and what are its benefits?**
    - Logical Volume Management
    - Flexible disk space management
    - Easy resizing of filesystems
    - Snapshots capability

46. **Explain the boot process in Linux.**
    1. BIOS/UEFI
    2. Bootloader (GRUB)
    3. Kernel load
    4. Init system
    5. Services startup

47. **What is load average and what does it indicate?**
    - Average system load over 1, 5, 15 minutes
    - Number of processes in running/waiting state
    - Relative to number of CPU cores

48. **What is swap space and when is it used?**
    - Virtual memory space on disk
    - Used when physical memory is full
    - Can indicate memory pressure

49. **What is a chroot jail?**
    - Changed root environment
    - Isolates process from main filesystem
    - Basic form of containerization

50. **What is the difference between SSL and TLS?**
    - SSL: Older protocol
    - TLS: Newer, more secure version
    - Both provide encryption and authentication

# Short-answer questions worth revisiting

- Explain the types of context switching
	- Process Switching: Switching between processes  
	- Thread Switching: Switching between threads of the same process

- How does filesystems allocate memory?
	- Contiguous allocation: gives a file n number of blocks of memory that can fit the file. Can lead to internal and external fragmentation  
	- Linked allocation: Non contiguous blocks and the directory contains the point to the start and end. Each block has the pointer the next block. Some overhead needed since in order to get a block thats in the middle of the chain we gotta go through each block beforehan  
	- Indexed allocation: Similar to Linked however theres a central block that is connected to all the other blocks
- explain what happens when we run ls \*.txt in extreme detail
	1. Shell Expansion (Globbing):
		- When you type `ls *.txt`, the shell (e.g., bash) first processes the command before `ls` is even invoked
		- The `*` is a wildcard character that triggers filename globbing
		- The shell expands `*.txt` by:
		  - Scanning the current directory
		  - Looking for all files that match the pattern: any characters (*) followed by literal ".txt"
		  - Creating a list of matching filenames
		  - Replacing the `*.txt` pattern with the actual matching filenames in alphabetical order
	
	2. Directory Access:
		- The shell accesses the current directory's inode through the filesystem
		- It reads the directory entries using the `getdents` system call
		- Each directory entry contains:
		  - Inode number
		  - Record length
		  - File name
		  - File type
	
	3. Pattern Matching:
		- For each directory entry:
		  - The shell checks if the filename matches the pattern `*.txt`
		  - It uses the following rules:
		    - `*` matches zero or more characters
		    - `.txt` must match exactly at the end
		  - The matching is case-sensitive on Unix-like systems
	
	4. Command Execution:
		- After expansion, if files are found, the shell executes `ls` with the expanded list as arguments
		- For example, if you have `file1.txt` and `file2.txt`, the shell effectively runs:
		  `ls file1.txt file2.txt`
		- If no files match, most shells pass the literal `*.txt` to ls as an argument
	
	5. The `ls` Command Processing:
		- `ls` receives the list of filenames as arguments
		- For each filename:
		  - Calls `stat` system call to get file information
		  - Retrieves:
		    - File permissions
		    - Number of links
		    - Owner and group
		    - File size
		    - Last modification time
		  - Formats this information based on options (default is just filenames)
	
	6. Special Cases:
		- Hidden files (starting with '.') are not included unless specifically requested
		- If no matching files exist:
		  - Bash: Passes literal pattern to ls
		  - Some shells: Show error message
		- If the pattern matches too many files:
		  - Can hit argument length limits
		  - May need to use `find` instead
		
	7. Error Handling:
		- Shell checks for permissions to read directory
		- `ls` checks for permissions to access each file
		- Various error conditions might be handled:
		  - Directory not readable
		  - File not accessible
		  - Too many arguments
