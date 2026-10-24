# OS Theory Reference

Textbook-depth OS theory (processes, scheduling, synchronisation, memory, paging, file systems, I/O) for rounds that go "explain an operating system in as much detail as you can". For the interview-shaped version, start with [linux-internals.md](linux-internals.md).

> Merged from 2024–2025 Meta Production Engineer prep notes. Overlapping copies were deduplicated; the newer, denser notes in this folder take precedence where they disagree.

# Keyword map

A one-screen vocabulary list — if any term here is unfamiliar, that's a gap.

### Core Operating System Concepts

#### Memory Management Mechanisms
- **Paging**: Fixed-size memory blocks management
  - **Demand Paging**: Load pages only when needed
  - **Page Fault**: Access to unmapped page triggers load
  - **Page Replacement**: Algorithms like LRU, Clock, Aging
  - **Page Table**: Maps virtual to physical addresses
  - **TLB (Translation Lookaside Buffer)**: Page table cache
  - **Working Set**: Pages process needs in RAM
  - **Thrashing**: Excessive paging, little progress

- **Segmentation**: Variable-size memory blocks
  - **Segment Table**: Maps segments to physical memory
  - **External Fragmentation**: Wasted space between segments
  - **Internal Fragmentation**: Wasted space within pages

- **Virtual Memory Features**:
  - **Copy-on-Write**: Share pages until modification
  - **Zero-Copy**: Direct data transfer without copying
  - **Memory Mapping**: Map files directly to memory
  - **Shared Memory**: Multiple processes sharing memory
  - **Demand Zero**: Pages zeroed when first accessed
  - **Page Coloring**: Cache optimization technique

#### Process Management Mechanisms
- **Process Creation**:
  - **Fork**: Create new process from existing one
  - **Exec**: Replace process image with new program
  - **Clone**: Create process/thread with shared resources
  - **vfork**: Optimized fork for exec calls

- **Process Scheduling**:
  - **Preemption**: Interrupt running process
  - **Time Slice**: CPU time allocated to process
  - **Priority Boost**: Temporary priority increase
  - **Real-time Priority**: Higher than normal processes
  - **Processor Affinity**: CPU core preferences
  - **Load Balancing**: Distribute processes across CPUs

- **IPC (Inter-Process Communication)**:
  - **Pipes**: Unidirectional communication
  - **Signals**: Asynchronous notifications
  - **Semaphores**: Resource access control
  - **Message Queues**: Message passing system
  - **Shared Memory**: Direct memory sharing
  - **Unix Domain Sockets**: Local communication

#### Protection & Security
- **Memory Protection**:
  - **Ring Levels**: CPU privilege levels (0-3)
  - **MMU**: Memory Management Unit
  - **Page Protection**: Read/write/execute bits
  - **ASLR**: Address Space Layout Randomization
  - **NX Bit**: No-Execute protection
  - **Stack Canaries**: Buffer overflow protection

- **Process Protection**:
  - **UID/GID**: User/Group identifiers
  - **Capabilities**: Fine-grained privileges
  - **Namespaces**: Resource isolation
  - **Cgroups**: Resource control
  - **Seccomp**: System call filtering
  - **SELinux/AppArmor**: Mandatory Access Control

#### System Resources
- **CPU Resources**:
  - **Context Switch**: Save/restore process state
  - **Interrupt Handling**: Hardware event processing
  - **DMA**: Direct Memory Access
  - **NUMA**: Non-Uniform Memory Access
  - **CPU Cache**: L1/L2/L3 caching
  - **SMP**: Symmetric Multi-Processing

- **Memory Hierarchy**:
  - **Physical Memory**: RAM
  - **Swap Space**: Disk-based virtual memory
  - **Page Cache**: File system cache
  - **Buffer Cache**: Block device cache
  - **Slab Allocator**: Kernel memory management
  - **Huge Pages**: Large memory pages

### Core Process & Resource Concepts

#### Process Management
- **Process**: Instance of running program with own memory space, resources, PID
- **Thread**: Lightweight unit sharing process memory space
- **Context Switch**: Saving current process state and loading another
- **Process States**:
  - Running: Currently executing
  - Ready: Waiting for CPU
  - Blocked: Waiting for I/O/event
  - Zombie: Terminated but parent hasn't read exit status
- **Fork()**: Creates new process by duplicating parent
- **Exec()**: Replaces current process with new program
- **Nice Values**: -20 to +19, controls process priority

#### Memory Management
- **Virtual Memory**: Process's own address space abstraction
- **Page**: Fixed-size block of memory (typically 4KB)
- **Page Fault**: Access to page not in physical memory
  - Minor: Page in memory but not mapped
  - Major: Page must be loaded from disk
  - Invalid: Bad memory access
- **TLB (Translation Lookaside Buffer)**: Cache for page table entries
- **OOM Killer**: Terminates processes when memory exhausted
- **Swap Space**: Disk space used as virtual memory extension

#### File Systems
- **Inode**: Metadata structure for file info (except name)
- **Directory Entry**: Maps filenames to inodes
- **Mount Point**: Directory where filesystem is attached
- **Journal**: Log of filesystem changes for crash recovery
- **Buffer Cache**: RAM cache for disk blocks
- **Page Cache**: Cache for file contents

#### Networking
- **Socket**: Endpoint for communication
- **TCP Handshake**: SYN → SYN-ACK → ACK
- **Network Namespaces**: Isolated network stacks
- **Routing Table**: Determines packet path
- **ARP**: Maps IP to MAC addresses
- **MTU**: Maximum transmission unit size

# Processes, scheduling, synchronisation, threads

#### General Notes
- A program is a set of instructions written in a programming language that defines a specific task or functionality. A process is an instance of a program currently being executed by the operating system. It includes program code, data, and system resources
- **Multiprogramming**: When multiple programs execute at a time on a single device, it is multiprogramming. **Multitasking**: When a single resource is used to process multiple tasks then it is multitasking.
- The multitasking is more responsive than multiprogramming because we do not wait for a process indefinitely. It's just the same concept as Round Robin Scheduling.
- A thread is the smallest unit assignable to the CPU. Also, we saw that we switch between processes in multitasking, and switching involved a cost. But this cost is less in thread switching compared to process switching. We would see this in context switching (threads share the same memory space).

#### Processes

Processes are a program in execution. It has its own memory which is divided into four parts: Text, Data, Heap, and Stack.
	**Text:** Stores the compiled program code.
	**Data:** stores global and static variables
	**Heap:** Stores dynamically allocated data
	**Stack:** Stores local variables.

Processes states: 

**Process Control Block(PCB)**
- It is a data structure that stores all information about a process.
- It contains these:
	1. Process State: We already saw these
	2. PID (Process ID): self-explanatory
	3. CPU registers and Program Counter: It is basically a pointer to the next instruction to be executed.
	4. CPU scheduling information: priority and pointers to scheduling queues(will see later)
	5. Memory management information: It ensures that blocks of memory space are properly managed and allocated
	6. Accounting Information: CPU time consumed, limits, etc.
	7. I/o Status information: Devices allocated, open file tables, etc.

**Process Scheduling Queues**
This handles the order of execution of the process. We have different queues for different states. When the state of a process is changed, it is simply detached from the current queue and added to its new state's queue.
> [!Note] 
> Please note that we are not actually studying scheduling algorithms, we are just assuming that we have some given order and how the processes are executed.

**Schedulers**

1. Long-term Schedulers: These are responsible for sending a process from new to ready queue. It has to maintain a degree of multiprogramming which is simply this: Average rate of incoming = average rate of outgoing.
2. Short-term schedulers/ CPU schedulers: From the ready queue to the running queue. (I think it's clear)
3. Midterm schedulers: These are special types of schedulers with a unique functionality: when a process is paused(maybe it needs some input) it, **swaps in** the process and allocates this position to some other process and when it arrives again, **swaps out** the other process and resume the previous process.

**Interrupts and Context Switch**

An interrupt is like a signal to stop the current process and allow the higher priority process(the process causing the interrupt) to execute. But what happens to the previous process which CPU was executing? It gets saved! When an interrupt occurs, the system needs to save the current **context** of the process currently running on the CPU so that it can restore that context. It's like a bookmark we have while reading novels. For example, if you are reading a book on data structures and algorithms and suddenly you realize that watching that Avengers 10 min scene would be far more interesting, you place a bookmark and return after 2 hours to resume the same.

> The context is saved in the PCB

Also, please note that taking up a new process requires saving the context of the current process and restore the context of the incoming process. This is known as **context switch**.

**Inter-Process Communication**

Inter-Process Communication or IPC for short is used for transferring information from one process to another. Processes that are interconnected and have to use some other process information to execute need to have IPC.

We have studied PCBs. We know Process information is stored in PCBs. And we need some sort of a temporary variable to transfer information(like we do to swap two numbers). And we do have two types of mechanisms on this type of principle.

###### **IPC Different Mechanisms**

###### **Shared Memory**

Kind of a buffer between two processes. One important thing to note is that under normal circumstances, processes restrict their memory, i.e., you cannot access the memory of a process. That's why we have shared memory, where we don't have such restrictions.
However, this may raise the producer-consumer problem.
- **Producer Process**: This process generates data items and adds them to a shared buffer.
- **Consumer Process**: This process removes data items from the shared buffer and consumes them.
- **Shared Buffer**: This is a fixed-size or bounded buffer that acts as a queue where producers add items and consumers remove items.
- The main challenges in the Producer-Consumer problem are to ensure the following conditions:
	**Buffer Full and Empty Conditions**
	- The producer must not add an item to the buffer if it is already full.
	- The consumer must not remove an item from the buffer if it is empty.
	
	**Mutual Exclusion**
	- Access to the shared buffer must be mutually exclusive, meaning only one process (either a producer or a consumer) can access the buffer at any given time to avoid data inconsistencies.
- It can be usually solver with semaphores
	- **Full Semaphore**: Tracks the number of filled slots in the buffer. It is initialized to 0, indicating that the buffer is initially empty.
	- **Empty Semaphore**: Tracks the number of empty slots in the buffer. It is initialized to the size of the buffer, indicating that the entire buffer is empty at the beginning.
	- **Mutex Semaphore**: Ensures mutual exclusion by allowing only one process to access the buffer at a time. It is initialized to 1, indicating that one process can access the buffer.

###### **Message Passing** 
Message passing: Kind of a queue for sharing information(See the below diagram). The main difference between the two is that here we don't actually share the same address space which is why it can be used with different computers over a network. 

Also, memory passing uses two primitives send & recieve that are system calls.
However, issues can arise like losing messages, thus we need to guard them with an **acknowledgement** message. And if that Ack message gets lost, sender sends again.
###### Scheduling Algorithms

###### **Terminology**

**Arrival Time**: Time at which process arrives in ready queue

**Completion Time**: Time at which the process completes its execution.

**Burst Time**: Time required by the process for CPU execution

**Turn-Around Time**: Time difference between completion and arrival time. (How it's different from burst time?)(Add waiting time here as well)

**Waiting Time**: Turnaround time-burst time.

**Throughput**: Total number of processes completed per unit time.

**Non-preemptive Algorithms**: Once a process is in the ready queue we cannot pause its execution until it's completely executed.

**Preemptive Algorithms**: Opposite of non-preemptive. Period.

###### Scheduling Algorithms

**First Come First Serve**
	Whichever comes first gets executed first (non-preemptive). 
	It's kind of obvious why it's bad. If the arrival time of a process that takes a lot of time than others is less, it will slow down the operating system.

**Shortest Job First (SJS)**
	Just sort the jobs according to their burst times. And we schedule the jobs according to that order only. It is a non-preemptive algorithm but there does exist a preemptive algorithm for SJS as well.
	It main disadvantage is  idle time

**Round Robin**
	In this algorithm, we have fixed time slots and we execute the processes sequentially in the order given and switch between processes after every fixed time.

##### Process Synchronization

1. Independent Process: Execution of one process does not affect the execution of other processes.
2. Cooperative Process: The execution of one process affects the execution of other processes.

**Race Condition**
When more than one process is executing the same code or accessing the same memory or any shared variable in that condition there is a possibility that the output or the value of the shared variable is wrong so for that all the processes doing the race to say that my output is correct this condition known as a race condition. Several processes access and process the manipulations over the same data concurrently, then the outcome depends on the particular order in which the access takes place.

**Critical Section**

A critical section is a code segment that can be accessed by only one process at a time. This code segment is common in many processes and if many processes run simultaneously, we would have a hard time finding the process containing the error, if it happens.
**Any solution to the critical section must satisfy these rules.**

1. Mutual Exclusion: If a process is executing in its critical section, then no other process is allowed to execute in the critical section.
   
2. Progress: If no process is executing in the critical section and other processes are waiting outside the critical section, then only those processes that are not executing in their remainder section can participate in deciding which will enter in the critical section next, and the selection can not be postponed indefinitely. Woah, that's a bummer. Okay, it's simply that the process which does not want to execute the critical section should not be allowed to block the system for others.
  
3. Bounded Waiting: A bound must exist on the number of times that other processes are allowed to enter their critical sections after a process has made a request to enter its critical section and before that request is granted.

###### **Different Solutions to synchronization problems**

1) Disabling Interrupts

	We know we can have a race condition if we have some interrupt in a preemptive scheduling algorithm(in a single processor system. In a multiprocessor system, we can have a race condition if two processes on different processors execute the critical section). A process can simply announce that it is entering a critical section and the processor should not interrupt it. Well, it works fine, doesn't it? NO, of course not. This can lead to a lot of problems. Firstly, it is not applicable for multiprocessor systems. Secondly, we cannot give a process the freedom to block the interrupts. It can go on indefinitely inside the critical section, disabling interrupts forever.

2) Locks (or Mutex)

	Here we have a lock. We acquire the lock, run a process in the critical section and then release the lock. How it's different from Disabling Interrupts? Here only the process which wants to execute inside the critical section wait, all other processes can still interrupt.
	
	There are two types of implementations:
	
	1. Software: Peterson solution for two processes and Bakery Algorithm for multiple processes.
	2. Hardware: Generally used, because it's faster. We have test and lock instructions and much more.
	
	**Peterson's Solution**
	
	As we saw earlier, we need a solution for the critical section of code, as it can lead to anomalies. This solution should satisfy the rules we mentioned before.
	
	**Simple Psuedo code**
	**How it works**

3) Semaphores

	Semaphore is nothing but an integer variable, and this can be changed by using only these two operations:
	
	1. Wait: It is like a decrement operation.
	
	```
	wait(S){
	   while(S<=0);
	   S--;
	} 
	```
	
	1. Signal:
	
	```
	Signal(S){
	   S++
	} 
	```
	
	Semaphores can be counting or binary(0 or 1).
	
	Binary Semaphores are generally called **mutex locks** as they provide mutual exclusion.
	Counting Semaphores are used to control access to a given resource for multiple processes.
4) Monitors
	Basically, it is a doctor's room.
	1. Only one person (thread) can be in the room at a time.
	2. There's a waiting area outside the room where other people (threads) form a line.
	3. When someone leaves the room, they signal the next person in line to enter.
	4. Inside the room, there may be specific conditions that need to be met before certain actions can be taken.

###### Deadlocks
**Conditions for a deadlock**

1. Mutual Exclusion: One resource can't be shared between different processes.
2. Hold and Wait: A process is holding a resource and waiting for another resource.
3. No Preemptive process: A process only releases the resource when its work is done completely.
4. Cycle Wait: There should be a cycle wait, like P0 waiting for P1 to complete execution, P1 for P2, and P2 for P0.

But how to prevent deadlocks?
1. Deadlock Prevention or Avoidance: We have to prevent any of the four conditions mentioned above.
	1. Mutual Exclusion: Suppose the resource is a mouse. Now we know that a mouse can't be used by more than one process. So, we can do nothing about it. The same is the case for printers, keyboards, etc. So, we can see that it is very difficult to overcome this condition
	2. Hold and Wait: This can be avoided. The problem here is that we wait for a process and hold other processes, so what we can do is that we would start a process only if we have assigned every resource it needs to it initially. But this is not practically possible. First of all, you can't really have a list of all resources required at the start of the process, it is only available at the runtime. Secondly, even if you have such a list, suppose the process would work for 10 hours, and a printer is needed at the end of the process. Here starvation of resources occurs, you are occupying printer for 10 hours straight, but you don't need it for 9 hrs!
	3. No Preemptive processes: Here what we can do is that if a process needs a resource, we just snatch this particular resource from whatever process it was working for and assign it to the current process. Obviously, it is also not practical, as suppose you are using a printer and your half-page was printed, but then suddenly another process prints something else. There would be a lot of inconsistency of data.
	4. No circular wait: This can be actually avoided. You can give priorities to resources and One process can only request to those resources whose priorities are greater than the one assigned to it. For example, we have four resources, R1, R2, R3, R4 having priorities as 1,2,3,4, and four processes P1, P2, P3, and P4.
		R1->P1 (R1 is assigned to P1)
		
		R2->P2
		
		R3->P3
		
		R4->P4
		
		Now P1 can only request R2, R3, and R4. And P2 can request R3 and R4 and so on. As you would have realized, there can be no cycle here.
2. Deadlock detection and recovery
3. Ignore deadlock and reboot the system 

#### Threads

We have multiple threads running in an interleaved fashion inside a process. The foremost advantage is the increased responsiveness of the CPU. A thread is the smallest unit assignable to the CPU.

**Advantages**

1. Faster to create/ terminate
2. Multiple threads share the same address space, thus making them easier to communicate. And due to this, context switching is also easier.
3. Lightweight

**Disadvantages**

1. Difficult to test and debug
2. Can lead to Deadlocks and Race Conditions.

**Types of Threads**

1. User Level Thread: A process is creating multiple threads and the kernel is not at all aware of these threads being created. Here we don't need to call the OS and there are no system calls involved making it much faster for context switching. But we have both advantages and certain disadvantages as well. We get fast context switching, and very fast creation and termination time because of no involvement of kernel. But if even a single threads make a call that waits, then all of the other threads in the process have to wait for that call to complete. This is because the call would be made to kernel obviously, and kernel considers the thread as the whole process and would not entertain another call of the same process thread.
    
2. Kernel Level Threads: Kernel knows and manages the threads. OS kernel provides system calls to create and manage threads. Kernel Level threads require the involvement of the kernel, which means that you have to do a MOR switch(Google it), you need to go from User space to the kernel space, and then you need to do the switch. And then we have to schedule some other thread as well.

**Difference between the two**

| Feature                     | User Level Thread                                                                        | Kernel Level Threads                         |
| --------------------------- | ---------------------------------------------------------------------------------------- | -------------------------------------------- |
| Management                  | In User space                                                                            | In kernel space                              |
| Context Switching           | Fast                                                                                     | Slow                                         |
| Blocking                    | One thread might block all other threads                                                 | A thread block itself only                   |
| Multicore or Multiprocessor | Cannot take advantage of multicore system. Only concurrent execution on single processor | Take full advantage of the multicore system. |
| Creation/Termination        | Fast                                                                                     | Slow                                         |

**Mapping of User threads to Kernel threads**

1. One to One: Most common way, used in common Operating Systems like windows. Every user thread is mapped to one kernel thread. Very similar to a pure kernel thread. They are slower than many to one but we can use multicore systems efficiently here.

1. Many to One: Many users are mapped to one kernel thread. They can be seen as pure user threads. They do provide fast context switching along with fast creation and termination but we cannot utilize multicore systems and all the threads can be blocked by one thread making a system call.
    
2. Many to Many: Very rarely used. Many user threads are mapped to many kernel threads.

## Priority inversion

- Occurs when a high-priority task waits on a lock held by a low-priority task, and a medium-priority task preempts the low-priority one — so the high-priority task is effectively blocked by the medium one.
- Fixes: **priority inheritance** (the lock holder temporarily inherits the waiter's priority), **priority ceiling** protocol, or avoid blocking on lower-priority tasks altogether.

## Thread models (1:1, M:1, M:N)

- **1:1 (kernel threads)** — what Linux NPTL does: every user thread is a kernel task; true parallelism, kernel-managed scheduling, more expensive creation/switch.
- **M:1 (user-level threads)** — managed entirely in user space; fast switches but one blocking syscall blocks all, and no multi-core parallelism.
- **M:N (hybrid)** — green threads multiplexed onto a pool of kernel threads (Go's goroutines, Erlang processes); best of both, complex runtime.

## How a mutex is actually implemented (futex)

Uncontended path is a single atomic test-and-set entirely in user space; only on contention does the thread make a `futex()` syscall and sleep in the kernel. That's why uncontended locks cost ~20 ns and contended ones cost a context switch.

# Memory management (Tanenbaum-style deep dive)

## Memory Management

### No Memory Abstraction

No memory abstraction basically mean you have direct access to physical memory, its issue is that if two programs are modifying the same memory location both will most probably crash. 

To get some parallelism without memory abstraction, we cvan program with multiple threads, however for multiple unrelated programs this won't work as threads abstraction does not provide that. A way to run multiple programs can be through swapping (will be explained later)

**Relocation Problem**
Here after loading the two programs in memory, you can see that the JMP28 instructions would jump to the ADD instruction of the first program instead of the CMP as it was supposed to and the program will crash, this happens since both programs access the **physical memory**

However, anyone with common sense should think "What if we just added the base address of the second program to all program addresses" and that exactly was done in the IBM 360, and that's called **static relocation**. However, it slows down loading and require a lot of information and break with some instructions like `MOV REG1,28` 

In general, remember that two problems have to be solved to allow multiple applications to be in memory at the same time without interfering with each other: **protection** and **relocation**
### Memory Abstraction: Address Spaces

An address space is the set of addresses that a process can use to address memory. Each process has its own address space, independent of those belonging to other processes (except in some special circumstances where processes want to share their address spaces). Just as the process concept creates a kind of abstract CPU to run programs, the address space creates a kind of abstract memory for programs to live in.

We will visit some of the ways to give each program its own address space.

#### Base and Limit Registers

This is a simple version of **dynamic relocation**, it basically map each process' address space onto a different part of physical memory.

It uses two special hardware registers, usually called **base** and **limit** registers, which obviously just maps the starting base address and the length of the program.  

Every time a process references memory, either to fetch an instruction or read or write a data word, the CPU hardware automatically adds the base value to the address generated by the process before sending the address out on the memory bus (the subsystem that facilitates CPU-memory communication). Simultaneously, it checks whether the address offered is equal to or greater than the value in the limit register, in which case a fault is generated and the access is aborted.

A disadvantage of relocation using base and limit registers is the need to perform an addition and a comparison on every memory reference. Additions are slow due to carry-propagation time (unless we use special addition circuits)

#### Swapping

Here we bring in each process in its entirety, running it for a while, then putting it back on the disk. 
When swapping creates multiple holes in memory, it is possible to combine them all into one big one by moving all the processes downward as far as possible. This technique is called **memory compaction** but it is not common as it needs a lot of CPU time.

#### Managing Free Memory

When memory is assigned dynamically, the operating system must manage it. In general terms, there are two ways to keep track of memory usage: bitmaps and free lists. 

##### Bitmaps

With bitmaps, it goes like this:
The size of the allocation unit is an important design issue. The smaller the allocation unit, the larger the bitmap. However, even with an allocation unit as small as 4 bytes, 32 bits of memory will require only 1 bit of the map.

The main issue with bitmaps is when putting a process with size k, finding a consecutive free space with size k is a slow operations.

##### Linked Lists
Another way of keeping track of memory is to maintain a linked list of allocated and free memory segments, where a segment either contains a process or is an empty hole between two processes. (like the one in fig 3-6 (c) above). 

While this reduces the memory needed by the manager still the issue of finding an empty place for a process isn't resolved. Possible algorithms for this:
1. First fit: first space large enough to hold the process
2. Next fit: Similar to first fit but keeps track of suitable holes so it searches them instead of searching everything again. (was shown to be worse than first fit with simulations)
3. Best fit: Search entire list for the smallest hole that fits the process
4. Worst fit: Pick the largest hole that fits the process, the intuition behind this is to have a large hole after putting the process not to waste memory (since best fit tends to lose a lot of memory in small holes)

Some observable tricks that can be used to speed up the process are like maintaining a separate list for holes and processes, keep the list sorted by size so the first fitting process is the best fit (and we can use binary search).

With this observation another allocation algorithm, **quick fit**, can be thought that maintains a separate lists for common sizes requested. For instance, we can have a table with entries each points to the head of a list of fixed size holes, this would make finding a hole big enough fast but the problem arises when we want to swap out or terminate a process it's computationally expensive to find its neighbors to see if a merge is possible.

### Virtual Memory

The problem we want to solve is having programs that need a memory space larger than the size of RAM, swapping worked for this but since a SATA is limited to few MB/s it is still slow. 

The first intuitive usable solution was **overlays**, a way to split the program into chunks and only load the overlay manager when a program started. However, the programming of these was time-consuming and boring and easily error-prone. But why don't we just make something like that in the memory-level rather than requiring programmers to do it? That's the birth of virtual memory.

The basic idea is each program has its own address space that is split into chunks called **pages**, where each page is a contiguous range of addresses that are mapped onto the virtual memory, but not all of them need to be on the physical memory, the missing parts will just be loaded when needed. 

#### Paging

First remember that programs reference a set of memory addresses (are called **virtual addresses**), while without using virtual memory these addresses are referenced directly in the physical memory with the memory bus, when we use virtual memory the virtual addresses go the **Memory Management Unit (MMU)** that maps the virtual addresses into physical memory. 
See how we can have 32KB of physical memory but we can write 64K programs? While we can't load it entirely to run it, as long as the complete copy of the program is on the disk we can bring on the pieces we need.

The virtual address space consists of fixed units called **pages** that are mapped in the physical memory to what we would call **page frames**. 

To track which pages are physically present in the memory, we use a **present/absent bit**.

Now an important question comes to mind, what happens when a program references an unmapped address? Here, the MMU notices that and causes the CPU to trap (exception interrupt), that trap is called a **page fault**. The OS in this case picks a little-used page frame and save it in disk and then fetches the referenced page into the newly freed page frame, then restarts the trapped instruction.

But here, some questions arise, why are we using power of 2 pages? How does the MMU actually operate?
The diagram above shows how MMU converts a virtual address (8196) into a physical address (24580). The process is like using an index in a book:

1. The virtual address is split into two parts:
    - A page number (like a chapter number)
    - An offset (like a page number within that chapter)
2. The page number (2 in this case) is used to look up the corresponding physical page frame in the page table.
3. The MMU then builds the physical address by:
    - Taking the physical page frame number it looked up
    - Keeping the original offset unchanged
    - Combining these pieces to form the final physical address

#### Page Tables

As we saw above, simplest way to map virtual addresses onto physical addresses is having the virtual address is split into a virtual page number (high-order bits) and an offset (low-order bits).

Thus, the purpose of the page table is to map virtual pages onto page frames. Mathematically speaking, the page table is a function, with the virtual page number as argument and the physical frame number as result. 

A typical page table entry contain some stuff we explained as the page frame number (the most important thing) and the *present/absent* bit.

The protection bits tell us what kinds of access are permitted (read/write/execute).

The *Modified* and *Referenced* bits keep track of page usage, if a page is modified, and thus the modified bit was set, we call it in *dirty* state and needs to be written back to the disk, if it was not modified, it's in *clean* state and can be abandoned. That bit can be called **dirty bit**. 

The *Referenced* bit is set whenever a page is referenced (either read or write) to help the OS in choosing a page to evict when a fault occurs.

The *Caching Disabled* bit allows us to disable caching for the page, this comes in handy for pages that map onto device registers rather than memory, for instance in the case of having a tight loop waiting for some I/O (continuous fetching) so it makes sense not to cache every time a change occurs.
Before more intensive implementation issues, remember that a virtual memory fundamentally creates a new abstraction - the address space - that is an abstraction of physical memory (like a process for a CPU)

#### Speeding Up Paging

We have the basics (virtual memory & paging), now the implementation has two main requirements:
	1. The mapping for virtual to physical addresses must be fast
	2. If the virtual address space is large, the page table will be large

The simplest design is to have one page table consisting of an array of registers, one entry per virtual page. But this is expensive and we would need to load the full page table at each context switch, thus bad performance. 

Another simple design is to make the entire page table in main memory so the hardware needs a single register that points to the page table, but this require a memory reference to read page table entries with each instruction, thus slow.

Now how to fix that? Let's explore some of the techniques.

##### Translation Lookaside Buffer (TLB) or Associative Memory

The observation that gave birth to this technique is that most programs tend to make a large number of references to small number of pages. This observation make us ask the question "Can't we map some virtual addresses to physical addresses directly so we don't **always** need to go the page table?" and that's exactly what TLB does. 
The TLB is a hardware device inside the MMU with a small number of entries (rarely more than 256). 

Now when MMU recieves a virtual address TLB checks if virtual page number is present (compares in parallel). If a valid match is found and the access does not violate the protection bits (if does, a protection fault happens), the page frame is taken directly from the TLB. 

On the other hand, if it was a *miss*, we would purge one entry and replace it with the new looked up one.

##### Software TLB Management

Up till now, all of these TLB operations happen in hardware (except faults/ traps). However, many RISC machines do all of these in software. 

In this case, when a *miss* occurs, a TLB fault is generated and the OS handles the entry manipulation and restart the faulty instruction. 

In software TLB Management, we need to understand the different types of misses:
1. Soft Miss: When a referenced page is not in the TLB but in the memory
	This doesn't need disk I/Oso it gets done in nanoseconds
2. Hard Miss: When a referenced page is not in memory
	This takes a couple of milliseconds since it needs a page table mapping lookup, this lookup is called **page table walk**

However, note that the "soft" vs "miss" is more of a spectrum, if a page fault occurs (page wasn't in the process page table) the page may be in the memory but in another process, here we will just map the page in the page tables appropriately. This is called **a minor page fault** which is a bit flaccid. On the other hand, a **major page fault** happens when a page needs to be brought from disk. Also, maybe the address is an invalid and thus the program will be killed and a **segmentation fault occurs**.

#### Page Tables for Large Memories

Now we sped up the mapping, what about how to deal with larger virtual address spaces?

##### Multilevel Page Tables

Basically a hashtable of a hashtables, makes everything a bit faster, intuitively looking at the diagram below for 30 seconds will be enough to get it. Just remember that when the *present/absent* bits are 0, a page fault will occur and the OS will handle that.
##### Inverted Page Tables

In this design, there is one entry per page frame in real memory, rather than one entry per page of virtual address space. The entry keeps track of which (process, virtual page) is located in the page frame. 

Unlike traditional page tables, which maintain a separate table for each process, an inverted page table uses a single, global table that is shared across all processes.

However, here the virtual-to-physical translation is much harder. Since the inverted page table are entries of form *(n, p)* where process *n* references virtual page *p*.  But here we can just use TLB to speed up the process (we will only suffer with a TLB miss since we need to search the whole table in **software**). This can be further improved by having a hash table hashed on the virtual address
### Page Replacement Algorithms

Remember what happens when a page fault occurs? We pick a page to evict to load a new page into memory. If the page was modified while in memory, it needs to be rewritten to the disk, else, we just evict the page and load the new one. 

Picking random is not good ofc, what should we do?

#### The Optimal Page Replacement Algorithm

Just pick the least referenced page (in the future) and evict it duh?

However, the OS can not know when each of the pages will be referenced next. But we can run a *simulation* and track that information for the next runs, that will allow that to be used, and now we got our golden benchmarking algorithm.

#### Not Recently Used (NRU)

Remember that most computers with virtual memory have two status bits, R (referenced) and M (modified). These two bits get updated on every memory reference. 

We can use these bits by resetting R bit of all pages periodically (e.g., on each clock interrupt) and then divide pages into 4 classes:
1. Class 0: not referenced, not modified
2. Class 1: not referenced, modified
3. Class 2: referenced, not modified
4. Class 3: referenced, modified

The NRU basically chooses a random page from the lowest-numbered nonempty class. But why are classes 1 & 2 ordered like this? The reasoning is that even though removing a modified page incurs the disk write cost, it's still better than removing a page that's actively being used, as that would likely cause an immediate page fault to bring it back in. However, it is a design choice in the NRU.

#### First-In, First-Out (FIFO)

Finally something straightforward, just a fixed-length queue. But it has the problem that it can throw out a heavily used page.
#### The Second-Chance 

A simple modification of the FIFO by checking the *R* bit of the oldest page, if it is 0 it gets replaced directly, if 1, we clear the bit and put it to the end of the queue and the search continue.
The algorithm always terminate, but it can be unifficient due to these continous movements (think of a lot of pages with *R* bit set).

#### The Clock Page

Same as the previous one but arranged on a *clock* to avoid repeated scans. Programmers in the 20th century smoked a lot of crack apparently.
#### Least Recently Used (LRU)

The first good approximation to the *optimal algorithm*. The observation that pages that have been heavily used in the last few instructions will probably be heavily used again soon and the converse holds. 

LRU basically, at the page fault, throw out the page that has been unused for the longest time.

However, implementing that is hard as fuck; we need a linked list of all pages ordered by the recency of usage and update that list on every memory reference. If you ever did some LC you'll realize that this takes too much time, even if done in hardware. Nevertheless, there are ways to implement LRU with special hardware.

The simplest smart trick to implement this is to have a counter *C* that gets incremented after each instruction. Also, we should have that counter in each page table entry. Then, simply evict the page with the lowest one, the LRU page.

#### Simulating LRU (Not Frequently Used (NFU) & Aging)

LRU is realizable, but the hardware needed for it isn't commonly available. 

The NFU algorithm try to fix that by having a software counter with each page. In each clock interrupt, the counter is incremented if the *R* bit is set to track how often each page has been referenced and choose the page with minimum counter for replacement.

However, this raises the issue that if some pages were heavily used at some pass (complete scan) they will be kept even if not as heavily used.

We can solve this with another algorithm, **aging**, where we shift right the counter by 1 bit before adding the R bit, and we add the R bit to the leftmost bit instead of the rightmost one.
The aging algorithm differ from the LRU in two points:

1. **Difference in Precision**: The aging algorithm uses counters with a finite number of bits to track page usage, which limits its ability to distinguish between pages referenced far in the past. In contrast, LRU maintains exact information about the order of page references.
2. **Difference in Handling Recent References**: Aging records only 1 bit per time interval, which can lead to ambiguity when determining the exact order of references within a clock interval. For example, if two pages were referenced in the same interval but at different times, aging cannot distinguish between them.

#### The Working Set Page Replacement Algorithm

After a while of running a process, it usually has most of its needed pages and runs with few page faults, what happens here is called **demand paging**. 

Fortunately, most processes exhibit a **locality of reference**; during any phase of execution only a small fraction of pages are included. These set of pages are a process' **working set**. 

However, if a lot of page faults occur every few instructions, we say the system is **thrashing**.

In the **working set model**, paging systems try to keep track of a process' working set and load it in memory before running. This way of loading pages before running the process is also called **prepaging**. Obviously the working set changes over time.

Now how to use that working set to implement page replacement? We will use two pieces of information: approximate time the page was last used and the *R* bit.
Here remember that the hardware sets the *R, M* bits and a periodic clock interrupt clears *R* bit on every tick. Also, age is the current virtual time minus the time of last use. Then, the algorithm goes as the figure above. 

However, If the entire table is scanned without finding a candidate to evict, that means that all pages are in the working set. In that case, if one or more pages with R = 0 were found, the one with the greatest age is evicted. 

#### The WSClock Page Replacement Algorithm

The basic working set algorithm is cumbersome, since the entire page table has to be scanned at each page fault until a suitable candidate is located. WSClock basically just makes the list a clock (remember the clock page algorithm)
What happens if the hand comes all the way around and back to its starting point? There are two cases we have to consider:
1. At least one write has been scheduled.
2. No writes have been scheduled.

In the first one, just keep moving till a clean page is found.

In the second case, just claim any clean page and use it. If no clean pages exist, then the current page is chosen as the victim and written back to disk.

>**Note:** **clean page** refers to a page in memory that has not been modified since it was loaded from secondary storage (e.g., a disk)

#### Summary
All in all, the two best algorithms are aging and WSClock.

### Design Issues For Paging Systems

We knew the techniques, let's explore the issues. Learning the chess pieces' movements isn't enough to beat Karbov.

#### Local versus Global Allocation Policies

Here we basically ask the question: If multiple processes are competing, should we only do page replacement in the process' allocated pages or all memory pages?
In general, global algorithms work better, however, we can use an algorithm to manage the memory allocation to processes.

One algorithm for this is the **PFF (Page Fault Frequency)** algorithm. Just count the number of faults per second an allocate memory based on that.

Also, note that some algorithms simply push for either choices. For instance, anything that involves the working set should only be local.

#### Load Control

Whenever the combined working sets of all processes exceed the capacity of memory, thrashing can be expected. The only real solution to this is to temporarily get rid of some processes. This can basically be done with swapping some processes to disk till thrashing stops.

#### Page Size

The OS chooses the page size. We will explore arguments for both small and large page size.

**Small is better**

> On average, have of the final page will be empty, that extra space is called **internal fragmentation**.

>Also, when we think about programs with multiple sequential phases, if each of them is small, a large page size will waste a lot of space.

**Large is better**

>Small pages $\implies$ more pages per program $\implies$ larger page table.

>Small pages $\implies$ wasting TLB space

Thus, a balance should be made. For that, some operating systems use different page sizes for different system parts (large for kernel and small for user processes)

#### Separate Instruction and Data Spaces

Most computers have a single address space that holds both programs and data. If this address space is large enough, everything works fine. However, if it’s too small, it forces programmers to stand on their heads to fit everything into the address space.
A solution for this is to have separate address spaces for instructions (program text) and data, called **I-space** and **D-space** respectively.

While separate I- and D-spaces are no longer necessary for managing the main memory (due to larger address spaces), the concept is still relevant in modern systems, particularly in the context of **L1 cache**. **L1 cache** is the fastest and smallest level of cache memory, located directly on the CPU chip. Memory in the L1 cache is still scarce because it is expensive and limited in size.

#### Shared Pages

If two programs are using the same library, why would we have two copies of the library in the memory? We don't want to have identical pages in memories, program text and read-only stuff are easily sharable. However, data pages? It's complicated.

In the case of having separate I- and D- spaces, multiple processes just share the I- space. 

Sharing data however, is trickier, but possible. For instance, in UNIX, a fork system call requires the parent and child to share both program's text and data. Thus, no copying of pages is done at fork time, however, all data pages mapped into both processes as READ ONLY. This process continues until a process updates a memory word. This violation causes a trap to the OS. 

When that trap happens, a copy is made of the offending page so each process now has its own private copy. Now, and only now, both copies are set to READ/WRITE, so the issue is eliminated and that page won't be trapping. Simply, pages that are never modified are not copied. That approach is called **copy on write**. 

#### Shared Libraries

In modern systems, large libraries are used by many processes (like I/O and graphics). Statically binding (executable has all of its needed libraries) all these libraries does not make sense. 

This can be seen in C libraries compiling a C code or windows' DLL files. However, a problem arises when some relocation instructions deal with specific addresses, that would cause issues when two processes are dealing with the same shared library. Here, a copy on write would work but then what is the purpose of shared libraries?

The solution for this is simple, compile the shared libraries with a flag that tells the compiler not to produce any instructions using absolute addresses and only use relative addresses. Code that uses only relative offsets is called **position-independent code**. 

#### Mapped Files

Shared libraries were just a special case of **memory-mapped files**. The main idea is that a process can issue a system call to map a file onto a portion of its virtual address space. In most implementations, no pages are brought in at the time of the mapping, but as pages are touched, they are demand paged in one page at a time, using the disk file as the backing store.

Mapped files provide an alternative model for I/O. Instead, of doing reads and writes, the file can be accessed as a big character array in memory.

If two or more processes map onto the same file at the same time, they can communicate over shared memory. Writes done by one process to the shared memory are immediately visible when the other one reads from the part of its virtual address spaced mapped onto the file.

First thought that should come to mind now if you were actually learning and not just reading is that if memory-mapped files are available, shared libraries can use that mechanism.

#### Cleaning Policy

Paging works best when there is an abundant supply of free page frames that can be claimed as page faults occur. For this, paging systems have a background process, **paging daemon**, that sleeps most of the time and is awakened periodically to evict some pages if too few page frames are free.

One way to implement this cleaning policy is with a two-handed clock. The front hand is controlled by the paging daemon. When it points to a dirty page, that page is written back to disk and the front hand is advanced. When it points to a clean page, it is just advanced.

#### Virtual Memory Interface

Up till now, we assumed programs deal with the virtual memory. Sometimes, programmers have more control over the memory map to do some nontraditional optimizations. For instance, programmers' ability to name regions of their memory can allow sharing the memory among processes.

Sharing of pages can also be used to implement a high-performance message-passing system. Normally, when messages are passed, the data are copied from one address space to another, at considerable cost. If processes can control their page map, a message can be passed by having the sending process unmap the page(s) containing the message, and the receiving process mapping them in.
### Implementation Issues

Choices up till now were spread among theoritical algorithm choices (second change vs aging, etc). However, some practical issues will arise when you are doing the implementation.

#### OS Involvement 

There are four times when the operating system has paging-related work to do: process creation time, process execution time, page fault time, and process termination time.

##### Step 1: Process Creation Time

- The OS determines the initial size of the process’s program and data.
- It creates and initializes a page table for the process.
- Space is allocated in the swap area on disk for the process’s pages.
- Information about the page table and swap area is recorded in the process table.

##### Step 2: Process Execution Time

- The MMU is reset and the TLB is flushed for the new process.
- The new process’s page table is made current in hardware registers.
- Optionally, some pages are loaded into memory to reduce initial page faults.

##### Step 3: Page Fault Time

- The OS identifies the virtual address causing the page fault.
- It locates the needed page on disk and finds a free page frame in memory.
- The OS reads the page into the frame and updates the page table.
- The faulting instruction is retried after the page is loaded.

##### Step 4: Process Termination Time

- The OS releases the process’s page table and any pages in memory.
- Disk space occupied by the process’s pages is freed.
- Shared pages are only released when no processes are using them.

#### Page Fault Handling

Let's describe in detail what exactly happens on a page fault.

1. Hardware traps the kernel, saving the program counter on the stack. 
2. An assembly code routine is started to save the general registers and other volatile information.
3. OS find out a page fault occurred and tries to discover the needed virtual page. Usually one of the hardware registers contains this information, if not, the OS must retrieve the program counter, fetch the instruction, and figure out what it was doing when the fault hit.
4. The virtual address then is checked if it is valid and the protection is consistent with the access. If not, the process is sent a signal or killed. If everything is okay, we find a free page frame or free one with a page replacement algorithm.
5. If the frame is dirty, a context switch takes place and the frame is marked as busy.
6. We now have a clean page frame, the OS looks up the needed page's address and bring it in.
7. When the page has arrived, the page table is updated.
8. The faulting instruction is backed up to the initial state and the program counter is reset to that point.
9. The faulting process is scheduled and the OS goes back to the routine that called it.
10. The routine reloads the registers and other state information and turns to user space to continue the execution and nothing has happened.
#### Instruction Backup

After the operating system has fetched the page needed after a page fault, it must restart the instruction causing the trap. This is easier said than done, especially for multi-operand instructions like those in the Motorola 680x0. The program counter (PC) at the time of the fault may point to different parts of the instruction, making it difficult for the OS to determine where the instruction began. Additionally, some instructions use autoincrement or autodecrement addressing modes, which modify registers as a side effect. If the fault occurs mid-execution, the operating system must undo or redo these side effects to ensure the instruction can be restarted correctly.

To address this issue, some CPUs provide hardware support, such as hidden internal registers that store the program counter before the instruction executes and track which registers have been autoincremented or autodecremented. This information allows the operating system to unambiguously undo the effects of the faulting instruction and restart it correctly.

#### Locking Pages in Memory

Virtual memory and I/O interact in subtle ways, such as when a process reading data into its address space is suspended, and another process causes a page fault that might evict the I/O buffer page, leading to data corruption. 

To prevent this, pages involved in I/O can be locked (pinned) in memory, or I/O can be performed into kernel buffers first, with data later copied to user pages.

#### Backing Store

We talked about page replacement algorithms and swapping processes to disks, but where exactly in disks?

In most UNIX systems, we have a dedicated swap partition (sometimes even on a separate disk to balance I/O load).

Before a process can start, the swap area must be initialized. One way is to copy the entire process image to the swap area, so that it can be brought in as needed. The other is to load the entire process in memory and let it be paged out as needed. However, this simple model has a problem: processes can increase in size after starting. For this, we have two approaches that can be seen below.
#### Separation of Policy and Mechanism

To manage complexity, memory management systems can separate policy from mechanism, as demonstrated in Fig. 3-29, which divides the system into three parts: a low-level MMU handler, a kernel-based page fault handler, and an external pager running in user space. 
The MMU handler manages hardware-specific details, the fault handler handles paging mechanics, and the external pager determines policy, such as page replacement, by coordinating with the fault handler to load and map pages into the user’s address space. 

While this modular approach improves flexibility and reliability, it introduces overhead due to frequent user-kernel boundary crossings and message passing, which may be acceptable as systems prioritize maintainability over raw performance.

### Segmentation

Segmentation addresses the problem of managing multiple dynamic data structures in memory by allowing each to occupy its own independent address space, preventing conflicts and simplifying memory allocation. 

You may ask, didn't we already do this with virtual address spaces for each program? That's correct, however, segmentation solves a different problem: it allows within a single program for multiple dynamic data structures (like tables, stacks, or arrays) to grow and shrink independently without colliding.

Segmentation is a memory management technique that divides virtual memory into independent address spaces called segments, each starting at 0 and extending to a maximum address. Unlike one-dimensional memory, where tables must occupy contiguous space, segmentation allows each segment to grow or shrink independently, preventing one table from interfering with another. This approach simplifies the management of dynamic data structures, such as symbol tables or stacks, and enhances flexibility in memory allocation. 

Additionally, segmentation supports sharing and protection mechanisms, enabling processes to share segments like libraries while enforcing access controls based on segment types.

#### Implementation of Pure Segmentation

The implementation of segmentation differs from paging in an essential way: pages are of fixed size and segments are not.
How pure segmentation works can be seen below:
After the system has been running for a while, memory will be divided up into a number of chunks, some containing segments and some containing holes. 

This phenomenon, called **checkerboarding** or **external fragmentation**, wastes memory in the holes. It can be dealt with by compaction

#### Segmentation with Paging: MULTICS

The segmentation with paging is a technique used in the **MULTICS** operating system to manage memory efficiently by dividing large segments into smaller pages. 

It provided each program with a virtual memory of up to 218 segments, each up to 65,536 words long, managed through a **segment table** that was itself paged.

Each **segment descriptor** contained details like the segment's location, size, protection bits, and a pointer to its page table.

Addresses in MULTICS included a **segment number** and an **address within the segment**, further divided into a **page number** and an **offset**.

The address translation process involved locating the segment descriptor, checking the page table, and computing the physical address by adding the offset to the page's base address.

MULTICS introduced the first TLB which stored the addresses of the 16 most recently referenced pages to speed up translation.

Programs with smaller working sets ran efficiently, while larger ones caused **TLB faults**.
#### Segmentation with Paging: The Intel x86

With the introduction of x86-64, segmentation was largely deprecated, except in legacy mode, as modern operating systems no longer relied on it.  

The x86-32 architecture used two key tables for memory management: the **Local Descriptor Table (LDT)** for program-specific segments and the **Global Descriptor Table (GDT)** for system-wide segments.  

To access a segment, a program loaded a **16-bit selector** into one of the six segment registers, which identified whether the segment was local or global and specified its entry in the LDT or GDT. 

Each **segment descriptor** contained the segment's base address, size, and other details, stored in microprogram registers for quick access.  

The x86 used a two-step process to convert a **(selector, offset)** pair into a physical address: first, it checked the segment's existence and offset validity, then added the offset to the segment's base address to form a **linear address**.  

If paging was disabled, the linear address was treated as the physical address, enabling pure segmentation.  

If paging was enabled, the linear address was mapped to a physical address using a **two-level page table system**, consisting of a **page directory** and **page tables**, each with 1024 entries.  

The x86 also included a TLB. Modern operating systems for x86 typically use a single, paged 32-bit address space, ignoring segmentation for simplicity and portability.  

### Summary

We have examined memory management. We saw that the simplest systems do not swap or page at all. Once a program is loaded into memory, it remains there in place until it finishes. Some operating systems allow only one process at a time in memory, while others support multiprogramming. This model is still common in small, embedded real-time systems. 

The next step up is swapping. When swapping is used, the system can handle more processes than it has room for in memory. Processes for which there is no room are swapped out to the disk. Free space in memory and on disk can be kept track of with a bitmap or a hole list. 

Modern computers often have some form of virtual memory. In the simplest form, each process’ address space is divided up into uniform-sized blocks called pages, which can be placed into any available page frame in memory. There are many page replacement algorithms; two of the better algorithms are aging and WSClock. 

To make paging systems work well, choosing an algorithm is not enough; attention to such issues as determining the working set, memory allocation policy, and page size is required. 

Segmentation helps in handling data structures that can change size during execution and simplifies linking and sharing. It also facilitates providing different protection for different segments. Sometimes segmentation and paging are combined to provide a two-dimensional virtual memory. The MULTICS system and the 32-bit Intel x86 support segmentation and paging. Still, it is clear that few operating system developers care deeply about segmentation (because they are married to a different memory model). Consequently, it seems to be going out of fashion fast. Today, even the 64-bit version of the x86 no longer supports real segmentation.

# Linux case study

## Linux

### Introduction

Before starting, remember the *principle of least surprise*. That's, for me, the best thing about using Linux.
On the figure above, notice how at the lowest level we have the interrupt handler which are the primary way for interacting with devices and the low-level dispatching mechanism (that happens when an interrupt occur). Note that this is usually distinct from the scheduling. The low-level code here stops the running process, saves its state in the kernel process structures, and starts the appropriate driver.

### **PROCESS MANAGEMENT**

**Process Definition:**
- Running instance of a program
- Has own address space, resources
- Represented by task_struct in kernel

**Key Process Operations:**
1. fork()
   - Creates child process
   - Child is copy of parent
   - Returns: 
     - Child PID to parent
     - 0 to child
     - -1 on error

2. exec()
   - Replaces process image
   - Loads new program
   - No return if successful

3. wait()
   - Parent waits for child
   - Returns child's exit status

4. exit()
   - Terminates process
   - Releases resources
   - Becomes zombie until parent waits

**Process States:**
1. Running: Currently executing
2. Ready: Can run, waiting for CPU
3. Blocked: Waiting for I/O/event
4. Zombie: Terminated but not waited for

**Scheduling:**
1. O(1) Scheduler
   - Constant-time operations
   - Priority-based queues
   - Range: 0-139 (lower = higher priority)

2. CFS (Completely Fair Scheduler)
   - Based on virtual runtime
   - Red-black tree implementation
   - Fair CPU distribution

**3. MEMORY MANAGEMENT**

**Memory Model:**
1. Text Segment
   - Program code
   - Read-only
   - Shared between processes

2. Data Segment
   - Initialized data
   - Global variables
   - Static variables

3. BSS Segment
   - Uninitialized data
   - Zeroed at start
   - No disk space needed

4. Stack Segment
   - Automatic variables
   - Function parameters
   - Grows downward

5. Heap
   - Dynamic memory
   - malloc()/free()
   - Grows upward

**Virtual Memory:**
- Page-based memory management
- Each process has own address space
- Demand paging
- Copy-on-write optimization

**Page Replacement:**
- PFRA (Page Frame Reclaiming Algorithm)
- Modified dual-hand clock algorithm
- Considers page access patterns
- Manages clean/dirty pages

**4. INPUT/OUTPUT SYSTEM**

**Device Types:**
1. Block Devices 
	- A block device driver is a type of device driver that manages devices which store and retrieve data in fixed-size blocks, typically used for storage devices
	- Fixed-size blocks
	- Random access
	- Example: disks, HDDs

3. Character Devices
	- A character device driver is a type of device driver in operating systems that handles devices which transmit or receive data as a stream of characters (bytes)
	- Byte stream
	- Sequential access
	- Example: terminals, keyboards, mice

**Device Drivers:**
- Kernel modules
- Hardware abstraction
- Major/minor numbers
- Loadable at runtime

**I/O Scheduling:**
- Request merging
- Elevator algorithm
- Deadline scheduling
- Fair queueing

**5. FILE SYSTEM**

**File System Structure:**
1. Superblock
   - FS metadata
   - Block counts
   - Free block info

2. Inodes
   - File metadata
   - Permissions
   - Block pointers

3. Data Blocks
   - File contents
   - Directory entries
   - Indirect blocks

**File Operations:**
1. open()
   - Creates file descriptor
   - Sets access mode
   - Returns file handle

2. read()/write()
   - Transfer data
   - Use file descriptors
   - Buffer management

3. close()
   - Releases resources
   - Flushes buffers
   - Updates metadata

**File Types:**
1. Regular Files
   - Data containers
   - Binary/text content

2. Directories
   - File collections
   - Name-to-inode mapping

3. Special Files
   - Device files
   - Named pipes
   - Sockets

**6. SECURITY**

**Access Control:**
1. User/Group
   - UID/GID
   - File ownership
   - Process ownership

2. Permissions
   - Read (r)
   - Write (w)
   - Execute (x)
   - For: owner/group/others

**Special Permissions:**
1. SUID
   - Run as owner
   - Security risk
   - Example: passwd

2. SGID
   - Run as group
   - Directory inheritance
   - Shared directories

**7. ANDROID**

**Core Components:**
1. Activities
   - UI components
   - Lifecycle management
   - State preservation

2. Services
   - Background operations
   - Long-running tasks
   - No UI

3. Broadcast Receivers
   - Event handlers
   - System/app messages
   - Background wakeup

4. Content Providers
   - Data sharing
   - Structured access
   - Permission control

**Security Model:**
1. Sandboxing
   - Process isolation
   - Unique UID per app
   - Resource restrictions

2. Permissions
   - Explicit grants
   - Runtime/install-time
   - Granular control

3. Binder IPC
   - Inter-process communication
   - Security token passing
   - Permission checking

# File systems & I/O theory

### File Systems

#### Intuition and Key Concepts

- **Files as Abstractions**: Files allow data storage and retrieval without exposing the complexities of physical storage.
- **Directories**: Organized structures for managing files, enabling hierarchical arrangements and access control.
- **Block Storage**: Files are stored in fixed-size blocks to optimize space and retrieval time.

#### Core Topics

##### File Naming

- **Uniqueness**: Names must uniquely identify files.
- **Attributes**: May include length, creation/modification dates, and permissions.
- **Case Sensitivity**: Varies across systems (e.g., UNIX is case-sensitive; Windows FAT is not).

##### File Types

- **Regular Files**: ASCII or binary data.
- **Directories**: Maintain file structures.
- **Special Files**: Character and block files represent I/O devices and disks.

##### Disk-Space Management

1. **Contiguous Allocation**: Simple but inflexible; resizing requires relocation.
2. **Linked Allocation**: Uses pointers; supports dynamic resizing but slows random access.
3. **Indexed Allocation**: Central table of pointers enables random access but adds overhead.
4. **File Allocation Table (FAT)**: Links maintained in memory for quick lookups.

##### Directory Structures

- **Single-Level**: One directory for all files, suitable for simple systems.
- **Hierarchical**: Nested directories allow complex organization.

##### Access Methods

- **Sequential**: Read/write data in order.
- **Random Access**: Access data at arbitrary locations.
- **Indexed**: Uses an index table for direct retrieval.

##### File-System Implementation

- **Virtual File Systems (VFS)**: Abstract interface supporting multiple underlying file systems.
- **Caching**: Improves performance by storing frequently accessed blocks in memory.
- **Journaling**: Logs metadata changes to ensure consistency.

#### Optimization Techniques

- **Block Size Trade-offs**: Large blocks reduce overhead but waste space; small blocks reduce waste but increase metadata.
- **Defragmentation**: Rearranges blocks to minimize access times.

---

### I/O Systems

#### Intuition and Key Concepts

- **Abstraction Layers**: Simplify hardware interaction through APIs.
- **Device Independence**: Uniform interfaces for diverse devices.
- **I/O Modes**: Blocking (simple for users), non-blocking, and asynchronous (high-performance).

#### Core Topics

##### I/O Hardware Principles

1. **Ports and Memory**: Devices interact via I/O ports or memory-mapped I/O.
2. **Direct Memory Access (DMA)**: Transfers data independently of the CPU, reducing overhead.
3. **Interrupts**: Signal the CPU upon task completion, improving efficiency.

##### I/O Software Goals

- **Uniform Naming**: Abstract device details; use logical paths.
- **Error Handling**: Managed at the lowest level possible.
- **Buffering**: Temporarily store data to handle rate mismatches.

##### I/O Software Layers

1. **Interrupt Handlers**: Respond to device signals.
2. **Device Drivers**: Abstract hardware specifics.
3. **Device-Independent Software**: Provides a consistent interface, handles naming, and buffering.
4. **User-Space Software**: Spooling and system libraries like `printf` and `scanf`.

##### Programmed I/O

- **Process**: CPU manages all data transfers.
- **Limitations**: Inefficient for large data operations.

##### Interrupt-Driven I/O

- **Mechanism**: Devices signal CPU upon readiness, allowing multitasking.
- **Applications**: Common in real-time systems.

##### DMA

- **Functionality**: A controller handles large data transfers autonomously.
- **Advantages**: Frees CPU for other tasks, enhances throughput.

#### Disk Management

1. **Disk Scheduling Algorithms**:
    - **FCFS**: Simple but inefficient.
    - **SSTF**: Reduces seek time.
    - **SCAN (Elevator Algorithm)**: Moves heads in one direction, then reverses.
2. **RAID Levels**:
    - Improve performance and redundancy.

#### Optimization Techniques

- **Buffer Caches**: Store frequent data.
- **Prefetching**: Anticipates data needs.
- **Error Correction**: Handles transient hardware faults.

---

### Key Takeaways for Interview

- Understand **abstraction layers** and their roles in both file systems and I/O.
- Familiarize with common algorithms (e.g., disk scheduling, memory allocation).
- Emphasize trade-offs: space vs. time, complexity vs. performance.
- Relate concepts to real-world challenges, e.g., distributed file systems, SSD optimizations.
