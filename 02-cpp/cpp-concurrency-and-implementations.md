# C++ Concurrency & From-Scratch Implementations

Whiteboard-ready implementations that are actually asked (thread pool, ring buffer, `shared_ptr`, LRU) plus the concurrency theory they test around them.

# `std::atomic` and memory ordering

- `std::atomic<T>` → operations are **indivisible**; no torn reads/writes, no data race. For trivially-copyable T. `is_lock_free()` tells you if it's HW-backed vs mutex-backed (integers/pointers usually lock-free).
- Common ops: `load`, `store`, `exchange`, `fetch_add`/`fetch_sub`/`fetch_or`, `compare_exchange_weak/strong` (CAS — foundation of lock-free algorithms).
- **`compare_exchange_weak` vs `strong`:** weak may fail spuriously (cheaper on some archs) → use in a loop; strong doesn't → use for one-shot.
- **Memory ordering (the money question):**
    - **`relaxed`** — atomicity only, no ordering/synchronization. Fine for counters where order doesn't matter. No extra CPU fence, only limits compiler reordering.
    - **`acquire`** (loads) / **`release`** (stores) — a release store _synchronizes-with_ an acquire load of the same variable: everything sequenced-before the release becomes visible to the reader after the acquire. This is how you publish data safely (the classic flag pattern). Lighter than seq_cst — on x86 often no extra instructions.
    - **`acq_rel`** — for read-modify-write (both acquire and release).
    - **`seq_cst`** (default) — release/acquire **plus** a single total order all threads agree on. Needed for multi-producer/multi-consumer where everyone must see all ops in the same order. Requires a full memory fence on multicore → most expensive.
- **`atomic` ≠ mutex-free correctness for compound logic:** atomic makes one op safe, not a sequence. `if (a.load()) a.store(...)` is still a race. Use CAS loops or a lock.
- **`volatile` is NOT for threading** — it prevents compiler caching for memory-mapped I/O; gives no atomicity or ordering. Common trap question.

**Publish pattern (know this cold):**

```cpp
std::atomic<bool> ready{false};
int data = 0;
// producer
data = 42;
ready.store(true, std::memory_order_release);
// consumer
while (!ready.load(std::memory_order_acquire)) {}
assert(data == 42);   // guaranteed: release/acquire synchronizes-with
```

# Implement a thread pool

Whiteboard-ready. Workers pull tasks from a queue guarded by a mutex + condition variable.

```cpp
#include <vector>
#include <queue>
#include <thread>
#include <mutex>
#include <condition_variable>
#include <functional>
#include <atomic>

class ThreadPool {
public:
    explicit ThreadPool(size_t n) {
        for (size_t i = 0; i < n; ++i)
            workers_.emplace_back([this] { worker_loop(); });
    }

    ~ThreadPool() {
        {
            std::unique_lock<std::mutex> lock(mtx_);
            stop_ = true;
        }
        cv_.notify_all();
        for (auto& t : workers_) t.join();
    }

    // enqueue a message-send task
    void submit(std::function<void()> task) {
        {
            std::unique_lock<std::mutex> lock(mtx_);
            tasks_.push(std::move(task));
        }
        cv_.notify_one();
    }

private:
    void worker_loop() {
        for (;;) {
            std::function<void()> task;
            {
                std::unique_lock<std::mutex> lock(mtx_);
                cv_.wait(lock, [this] { return stop_ || !tasks_.empty(); });
                if (stop_ && tasks_.empty()) return;
                task = std::move(tasks_.front());
                tasks_.pop();
            }
            task();   // e.g. send_message(msg) — runs outside the lock
        }
    }

    std::vector<std::thread> workers_;
    std::queue<std::function<void()>> tasks_;
    std::mutex mtx_;
    std::condition_variable cv_;
    bool stop_ = false;
};
```

**Points to say aloud:**

- `cv_.wait(lock, pred)` handles **spurious wakeups** (the predicate loop) — interviewers love this.
- Run `task()` **outside** the lock so sends happen in parallel; only the queue is serialized.
- Graceful shutdown: set `stop_`, `notify_all`, drain remaining tasks (or don't, depending on semantics), `join`.
- Extensions they may push on: return a `std::future` (`submit` returns `std::packaged_task`'s future); bounded queue + backpressure; per-worker queues + **work stealing** to cut contention; ordering guarantees for message sends.

# Implement a thread-safe ring buffer

Common implementation question, adjacent to the thread pool above. Bounded FIFO over a fixed array; producer/consumer. Simple mutex+CV version first (correct & clear), mention lock-free as the follow-up.

```cpp
#include <vector>
#include <mutex>
#include <condition_variable>

template <typename T>
class RingBuffer {
public:
    explicit RingBuffer(size_t cap) : buf_(cap) {}   // capacity fixed

    void push(T item) {                              // blocks if full
        std::unique_lock<std::mutex> lk(m_);
        not_full_.wait(lk, [this] { return count_ < buf_.size(); });
        buf_[tail_] = std::move(item);
        tail_ = (tail_ + 1) % buf_.size();
        ++count_;
        not_empty_.notify_one();
    }

    T pop() {                                        // blocks if empty
        std::unique_lock<std::mutex> lk(m_);
        not_empty_.wait(lk, [this] { return count_ > 0; });
        T item = std::move(buf_[head_]);
        head_ = (head_ + 1) % buf_.size();
        --count_;
        not_full_.notify_one();
        return item;
    }

private:
    std::vector<T> buf_;
    size_t head_ = 0, tail_ = 0, count_ = 0;
    std::mutex m_;
    std::condition_variable not_full_, not_empty_;
};
```

**Talking points:**

- `head_`/`tail_` wrap with `% capacity`; `count_` disambiguates full vs empty (the alternative is "leave one slot empty" so `head==tail` means empty and `(tail+1)==head` means full — mention you know both).
- Two CVs (`not_full_`, `not_empty_`) so producers and consumers wake the right waiters. Predicate loop handles spurious wakeups.
- **Single-producer/single-consumer (SPSC) lock-free** version: two `std::atomic<size_t>` indices, producer writes `tail` with `release`, consumer reads it with `acquire` (and vice versa) — no mutex, wait-free. This is the HFT-relevant answer; offer it as the optimization. MPMC lock-free is much harder (ABA problem, needs CAS loops) — acknowledge the jump in difficulty rather than hand-waving.
- Complexity: O(1) push/pop, O(capacity) space, zero allocation after construction (cache-friendly, predictable latency — why trading systems love ring buffers).

# Implement `shared_ptr`

Core idea: **object pointer + separate control block** holding a reference count; last owner destroys the object. Interviewers may ask for thread-safe refcount (atomic) and `weak_ptr` awareness.

```cpp
#include <atomic>
#include <utility>

template <typename T>
class SharedPtr {
    struct ControlBlock {
        std::atomic<long> count;
        explicit ControlBlock(long c) : count(c) {}
    };

    T* ptr_ = nullptr;
    ControlBlock* ctrl_ = nullptr;

    void release() {
        if (ctrl_ && ctrl_->count.fetch_sub(1, std::memory_order_acq_rel) == 1) {
            delete ptr_;
            delete ctrl_;
        }
    }

public:
    SharedPtr() = default;

    explicit SharedPtr(T* p) : ptr_(p), ctrl_(p ? new ControlBlock(1) : nullptr) {}

    // copy: share ownership, bump count
    SharedPtr(const SharedPtr& o) : ptr_(o.ptr_), ctrl_(o.ctrl_) {
        if (ctrl_) ctrl_->count.fetch_add(1, std::memory_order_relaxed);
    }

    // move: steal, no count change
    SharedPtr(SharedPtr&& o) noexcept : ptr_(o.ptr_), ctrl_(o.ctrl_) {
        o.ptr_ = nullptr; o.ctrl_ = nullptr;
    }

    SharedPtr& operator=(SharedPtr o) noexcept {   // copy-and-swap: handles copy, move & self-assign
        std::swap(ptr_, o.ptr_);
        std::swap(ctrl_, o.ctrl_);
        return *this;
    }

    ~SharedPtr() { release(); }

    T& operator*()  const { return *ptr_; }
    T* operator->() const { return ptr_; }
    T* get()        const { return ptr_; }
    long use_count() const { return ctrl_ ? ctrl_->count.load() : 0; }
    explicit operator bool() const { return ptr_ != nullptr; }
};
```

**Talking points:**

- **Why a separate control block?** So multiple `shared_ptr`s share one count, and `weak_ptr` can outlive the object (control block holds a _second_ (weak) count and survives until weak count also hits 0).
- **Atomic refcount** → copying/destroying across threads is safe. But the _pointee_ isn't automatically thread-safe. `fetch_sub` uses acq_rel so the delete happens-after all prior releases.
- **`make_shared`** allocates object + control block in **one** allocation (faster, better locality; downside: object memory lives until weak count = 0 too).
- **copy-and-swap** `operator=` (take by value) elegantly covers copy, move, and self-assignment.
- **Cycles:** two `shared_ptr`s pointing at each other never free → memory leak → break with `weak_ptr`.
- Real `shared_ptr` also supports custom deleters and aliasing constructor — mention if pushed.

# Implement an LRU cache (and make it thread-safe)

**Prompt:** "Let's write something. Implement an LRU cache — `get` and `put`, both O(1), fixed capacity. C++ is fine. Talk me through it before you type."

> **Restate first (don't dive in):** "So: a fixed-capacity key→value cache; `get` returns the value and marks it most-recently-used; `put` inserts/updates and, if over capacity, evicts the least-recently-used. Both O(1). The standard structure is a hash map plus a doubly linked list — the map gives O(1) lookup, the list maintains recency order with O(1) move-to-front and O(1) tail eviction. The map stores iterators into the list so I can splice in O(1). Sound right?"

```cpp
#include <list>
#include <unordered_map>

class LRUCache {
    int cap_;
    std::list<std::pair<int,int>> items_;                 // front = MRU, back = LRU; {key,value}
    std::unordered_map<int, std::list<std::pair<int,int>>::iterator> idx_;
public:
    explicit LRUCache(int capacity) : cap_(capacity) {}

    int get(int key) {
        auto it = idx_.find(key);
        if (it == idx_.end()) return -1;                  // miss
        items_.splice(items_.begin(), items_, it->second); // O(1) move to front
        return it->second->second;
    }

    void put(int key, int value) {
        auto it = idx_.find(key);
        if (it != idx_.end()) {                           // update + promote
            it->second->second = value;
            items_.splice(items_.begin(), items_, it->second);
            return;
        }
        if ((int)items_.size() == cap_) {                 // evict LRU
            idx_.erase(items_.back().first);
            items_.pop_back();
        }
        items_.push_front({key, value});
        idx_[key] = items_.begin();
    }
};
```

> **Narrate while typing:** "`splice` is the key trick — it relinks the node in O(1) without invalidating iterators, so the map entries stay valid. `list` iterators are stable across other inserts/erases, which is exactly why I'm storing them in the map." **Then test out loud:** cap 2 → put(1,1), put(2,2), get(1)→1, put(3,3) evicts 2, get(2)→-1. Walk that trace verbally.

**Follow-up:** "Now make it thread-safe — multiple threads hitting `get`/`put`."
> Simplest correct thing first: wrap both methods with a `std::mutex` via `std::lock_guard` — note `get` mutates recency, so it's not a read-only operation and a shared/reader lock alone is wrong. That serializes everything, which is fine if it's not the bottleneck. If it becomes contended, I'd shard the cache into N independent LRUs keyed by `hash(key) % N`, each with its own lock — that cuts contention roughly N-fold without a lock-free rewrite. I'd only go lock-free if profiling proved the sharded mutex is still the hot spot, because lock-free LRU is genuinely hard to get right.

**Follow-up 2:** "What if the cache has to be shared across *processes*, not threads?"
> Then I can't share heap pointers — I'd put the data in shared memory (`shm_open` + `mmap`) with a process-shared mutex (`pthread_mutexattr_setpshared`), or more pragmatically reach for an existing out-of-process cache like Redis/memcached rather than reinventing inter-process LRU. Which way depends on latency budget vs. operational simplicity.

# Cache locality & cache-line optimisation

They ask "explain the impact of cache locality" and "optimize a data structure for cache-line utilization." Have concrete answers.

- **Why it matters:** L1 ~1ns vs main memory ~100ns → a cache miss is ~100× slower. On low-latency trading paths, memory layout often dominates algorithmic constants.
- **Cache line = 64 bytes.** Memory is fetched a whole line at a time. Access one byte → you pay for 64.
- **Spatial locality:** keep data you access together, together. `std::vector` (contiguous) beats `std::list`/`std::map` (node-per-element, pointer-chasing) for iteration — often even when Big-O favors the node structure, because of cache misses + prefetching.
- **Temporal locality:** reuse recently-touched data while it's still hot in cache.
- **Struct-of-Arrays (SoA) vs Array-of-Structs (AoS):** if you only touch one field across many elements, SoA packs those fields contiguously → fewer lines loaded. AoS wastes bandwidth on fields you don't read. Classic cache optimization to name.
- **False sharing:** two threads writing different variables that share a cache line → the line ping-pongs between cores' caches (cache-coherence traffic), killing performance despite no logical contention. Fix: `alignas(64)` / padding to put hot per-thread data on separate lines.
- **Other levers:** `reserve()` to avoid reallocation; avoid pointer indirection; hot/cold splitting (keep rarely-used fields out of the hot struct); prefer flat hash maps (open addressing, e.g. `absl::flat_hash_map`) over `std::unordered_map` (node-based, chases pointers) for cache-friendliness; access memory sequentially so the hardware prefetcher can work.
- **How to measure:** `perf stat` (cache-misses, cache-references), `perf record`, `cachegrind`, LLC-miss counters. Say you'd measure, not guess.

# Deadlocks

- **Definition:** set of threads each waiting on a resource held by another, none can proceed.
- **Coffman conditions (all four required):** (1) **mutual exclusion**, (2) **hold-and-wait**, (3) **no preemption**, (4) **circular wait**. Break any one → no deadlock.
- **Prevention/avoidance:**
    - **Lock ordering** — always acquire locks in a global consistent order (breaks circular wait). Most practical answer.
    - `std::scoped_lock`/`std::lock(...)` — acquires multiple locks deadlock-free (all-or-nothing).
    - `std::lock_guard` / `std::unique_lock` (RAII) so locks always release.
    - Try-lock with backoff / timeout (breaks hold-and-wait).
    - Avoid nested locks; minimize critical sections; use lock-free structures where feasible.
    - Banker's algorithm = avoidance (mostly academic).
- **Livelock** (threads keep reacting, no progress) and **starvation** (a thread never gets the resource) — mention as related failure modes.
- **How to debug one live:** `gdb` attach → `thread apply all bt` → look for two threads blocked on each other's mutexes. TSan and helgrind detect lock-order inversions.

# Concurrency vs parallelism

- **Concurrency = dealing with many things at once (structure).** Tasks make progress in overlapping time periods; can happen on a **single core** via time-slicing/interleaving. About _composition of independent tasks._
- **Parallelism = doing many things at once (execution).** Truly simultaneous execution, requires **multiple cores/hardware.**
- Rob Pike's line: "Concurrency is about structure, parallelism is about execution." Concurrent design _enables_ parallelism but doesn't require it.
- Single-core concurrent system: I/O-bound work (async, event loop) still wins big because you overlap waiting.
- Multi-core parallel system: CPU-bound work scales with cores — until bounded by **Amdahl's law** (speedup capped by the serial fraction: `1 / (s + (1-s)/N)`).
- Contention, synchronization overhead, and false sharing can make more cores _slower_. Mention this — it signals real experience.
