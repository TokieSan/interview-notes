# Python — Internals & Production Performance

Systems roles probe Python two ways: production performance (profiling, leaks, the GIL) and language internals (`dict`, `list` growth, generators, decorators, `with`). Prepare both.

## A. The topic map

**Production performance (lead here):**
1. Profiling in production — `py-spy`, `cProfile`, when to use which
2. Diagnosing memory leaks in long-running processes
3. The GIL in practice (and the 3.13+ free-threaded story)
4. Pushing work into native extensions / C++ when Python isn't enough

**Language internals (don't get caught out):**
5. `with` / context managers
6. Generators and `yield`
7. Decorators
8. How CPython implements `dict`
9. `list` memory growth
10. Reference counting + cyclic GC; mutable default args; `is` vs `==`

---

## B. Topic-by-topic — explanation, *why*, sources

### 1. Profiling in production: py-spy vs cProfile

- **`cProfile`** — built-in **deterministic** profiler: instruments every call, gives exact call counts and cumulative/total time. Cost: meaningful overhead, runs *in-process*, and you generally have to start the process under it. Great for **local/dev deep dives**; visualize with `snakeviz` or pull stats with `pstats`.
- **`py-spy`** — third-party **sampling** profiler written in Rust. It attaches to an **already-running** process by PID (`py-spy top --pid <PID>`, `py-spy dump --pid <PID>`, `py-spy record -o out.svg --pid <PID>`) and reads stacks from *outside* the process, so it adds **near-zero overhead** and needs **no code changes** — which is exactly why it's the production tool. It produces flame graphs; `--gil` filters to samples actually holding the GIL, `--idle` includes threads blocked on I/O.
- **The answer they want:** *"`py-spy` for live/production triage because it's zero-instrumentation and low-overhead on a running PID; `cProfile` for a deterministic local deep-dive once I've localized the hotspot. Sampling tells me where wall-clock goes; deterministic tells me exact call counts."*

Sources: [cProfile docs](https://docs.python.org/3/library/profile.html) · [py-spy (GitHub)](https://github.com/benfred/py-spy) · [Brendan Gregg — flame graphs](https://www.brendangregg.com/flamegraphs.html).

### 2. Memory leaks / growth in long-running processes

In CPython, "leak" usually means **unintended retention**, not C-style lost pointers (refcounting frees most things). Causes to name:
- **Lingering references** — module-level caches/dicts/lists that only grow, closures capturing big objects, `lru_cache` without `maxsize`, accumulating in a global registry, exception tracebacks holding frames.
- **Reference cycles** with objects that the cyclic GC handles — but if a `__del__` finalizer or C-extension complicates collection, they linger.
- **C-extension leaks** — native code that forgets `Py_DECREF`; these *are* true leaks and won't show in Python-level tools.
- **Fragmentation** — freed memory not returned to the OS (`pymalloc` arenas).

Toolkit: `tracemalloc` (built-in — snapshot + diff allocations by line), `objgraph` (what's referencing what, growth over time), `gc` module (`gc.get_objects()`, `gc.collect()`, `gc.set_debug`), `py-spy dump` for stacks, and OS-level `RSS` via `/proc/<pid>/status` or `ps`. **Method:** take a baseline `tracemalloc` snapshot, run the workload, diff, follow the top growers' referrer chains.

Sources: [tracemalloc](https://docs.python.org/3/library/tracemalloc.html) · [gc module](https://docs.python.org/3/library/gc.html) · [objgraph](https://mg.pov.lt/objgraph/).

### 3. The GIL in practice (know the *current* story — this is 2026)

- **What:** the Global Interpreter Lock is a mutex that lets only **one thread execute Python bytecode at a time**, protecting CPython internals (notably refcounts). It existed because it made the interpreter and C-extension authoring simpler.
- **Practical consequence:** threads do **not** parallelize **CPU-bound** pure-Python work — you'll see one core pinned while others idle. Threads *are* useful for **I/O-bound** work because the GIL is released during blocking I/O (and many C extensions, e.g. NumPy, release it during heavy compute).
- **How engineers work around it (give the practitioner answer):**
  - **`multiprocessing`** / `ProcessPoolExecutor` — true parallelism across processes, at the cost of IPC/serialization overhead.
  - **`asyncio`** — single-thread concurrency for high-I/O fan-out without thread overhead.
  - **Release the GIL in native code** — NumPy/Cython/C extensions drop the GIL during number-crunching so other threads run.
  - **Push hot loops out of Python** entirely (next section).
- **The 2024–2026 update (have this ready, it signals you're current):** PEP 703 made the GIL **optional**. Python **3.13** (Oct 2024) shipped an experimental **free-threaded build** (`python3.13t`) where the GIL can be disabled — true multi-core threading — at the cost of some single-thread overhead and the burden of real thread-safety (object-level locking replaces the one global lock). **3.14** improved the single-thread penalty; ecosystem/C-extension support is still catching up. You can check at runtime with `sys._is_gil_enabled()`. Bottom line you can say: *"Historically I'd reach for multiprocessing or push work native; going forward free-threaded CPython is changing the calculus, but it's not yet a drop-in for production with C-extension dependencies."*

Sources: [Python wiki — GIL](https://wiki.python.org/moin/GlobalInterpreterLock) · [PEP 703 (making the GIL optional)](https://peps.python.org/pep-0703/) · [Python 3.13 free-threading HOWTO](https://docs.python.org/3/howto/free-threading-python.html).

### 4. Native extensions / dropping to C++ when Python isn't enough

When profiling shows a CPU-bound hot loop in interpreted bytecode, options in rough order: **vectorize with NumPy** (push the loop into C); **Cython** (annotate types, compile to C); **C/C++ extension** via the CPython C-API, **pybind11**, or **Cython**; **ctypes/cffi** to call an existing shared lib; **PyO3** for Rust. The systems-engineer framing: *measure first* (`py-spy`/`cProfile`), confirm it's CPU-bound and in Python land (not I/O or a C lib already), then move the narrowest hot path native and **release the GIL** in that section so threads scale. Tie this to a real story if you have one (e.g., a parsing/serialization hot path).

Sources: [Extending Python with C/C++](https://docs.python.org/3/extending/extending.html) · [pybind11](https://pybind11.readthedocs.io/) · [Cython](https://cython.readthedocs.io/).

### 5. `with` / context managers (a *documented* HRT question)

> Real HRT question (SWE, Singapore): *"How does the `with` keyword work?"*

`with` runs a **context manager**: an object implementing `__enter__` and `__exit__`. On entry Python calls `__enter__()` (its return binds to the `as` name); on exit — **normal or exceptional** — it calls `__exit__(exc_type, exc_val, exc_tb)`, which is where cleanup runs. If `__exit__` returns truthy it *suppresses* the exception. This is Python's RAII-equivalent: deterministic cleanup of files, locks, sockets, transactions, regardless of how the block exits. You can write one as a class or, more compactly, with `@contextlib.contextmanager` on a generator (code before `yield` = enter, after = exit). Good cross-link to your C++ answer: *"`with` is to Python what RAII is to C++ — the difference is C++ ties cleanup to scope/destructors automatically, Python ties it to the explicit `with` block."*

Sources: [with statement / context managers](https://docs.python.org/3/reference/compound_stmts.html#the-with-statement) · [contextlib](https://docs.python.org/3/library/contextlib.html).

### 6. Generators and `yield` (a *documented* HRT topic)

> Real HRT question (SRE, US): *"What is a `yield` statement used for? What are a generator and a decorator?"*

A function containing `yield` is a **generator function**; calling it returns a **generator object** (an iterator) without running the body. Each `next()` runs until the next `yield`, produces a value, and **suspends** — local state and the instruction pointer are frozen on the frame and resumed on the next `next()`. Benefits: **lazy / streaming** evaluation and **O(1) memory** over huge or infinite sequences (you don't materialize a list). `yield from` delegates to a sub-iterator. This is the idiomatic way to stream large data through a pipeline — directly relevant to data/systems tooling.

Sources: [yield expressions](https://docs.python.org/3/reference/expressions.html#yield-expressions) · [generator glossary](https://docs.python.org/3/glossary.html#term-generator).

### 7. Decorators (documented)

A decorator is a callable that takes a function (or class) and returns a replacement, applied with `@deco` sugar = `f = deco(f)`. Used for cross-cutting concerns without touching the wrapped body: logging, timing, caching (`functools.lru_cache`), retries, auth, registration. Be ready to **write one live**, including `*args, **kwargs` passthrough, `functools.wraps` to preserve metadata, and a **parameterized** decorator (a function returning a decorator — the third nesting level catches people out):

```python
import functools, time

def timed(label):                      # decorator factory (takes args)
    def deco(fn):
        @functools.wraps(fn)           # preserves __name__/__doc__
        def wrapper(*args, **kwargs):
            t0 = time.perf_counter()
            try:
                return fn(*args, **kwargs)
            finally:
                print(f"{label}: {time.perf_counter()-t0:.4f}s")
        return wrapper
    return deco

@timed("work")
def work(n): ...
```

Sources: [PEP 318 (decorators)](https://peps.python.org/pep-0318/) · [functools.wraps](https://docs.python.org/3/library/functools.html#functools.wraps).

### 8. How CPython implements `dict` (a *documented* HRT question)

> Real HRT question (Systems SWE, NY): *"Explain how Python implements dictionaries."*

An **open-addressing hash table** (not chaining). Key points to hit:
- A sparse **index array** maps `hash(key) mod size` to a slot; **collisions** are resolved by a **probing sequence** (a perturbation scheme mixing in high hash bits), not linked lists.
- Average **O(1)** lookup/insert/delete; worst case O(n) under pathological collisions.
- It **resizes/rehashes** (grows) when the load factor gets too high (roughly 2/3 full) to keep probe chains short.
- Since **Python 3.6/3.7** dicts are **insertion-ordered** and use a **compact "split" layout**: a dense `entries` array holds (hash, key, value) in insertion order, and the sparse array stores *indices* into it — this cut dict memory ~20–25% and gave ordering for free.
- `hash` + `__eq__` contract: equal objects must have equal hashes; keys must be hashable (immutable).

Sources: [CPython `dictobject.c`](https://github.com/python/cpython/blob/main/Objects/dictobject.c) · [Raymond Hettinger — "Modern Dictionaries" talk](https://www.youtube.com/watch?v=p33CVV29OG8) · [data model — hashing](https://docs.python.org/3/reference/datamodel.html#object.__hash__).

### 9. `list` memory growth (HRT's own blog literally names this)

> HRT's blog cites this exact thing as a fundamentals test: *"If you use Python lists, do you know the memory growth characteristic as it expands?"*

A CPython `list` is a **dynamic array of `PyObject*` pointers** (not the objects themselves) plus a stored length and capacity. When `append` exceeds capacity it **over-allocates** geometrically so growth is amortized O(1); the CPython growth formula is roughly `new = old + (old >> 3) + 6` then rounded — i.e. **~1.125× plus a constant**, *not* 2× (a common wrong answer). Consequences: appends are amortized O(1); `insert(0, ...)`/`pop(0)` are O(n) (shift everything) → use `collections.deque` for a queue; indexing is O(1); and each element is a pointer to a heap object, so a list of ints is far less cache-friendly than a NumPy array or C++ `vector<int>`.

Sources: [CPython `listobject.c` — `list_resize`](https://github.com/python/cpython/blob/main/Objects/listobject.c) · [time complexity wiki](https://wiki.python.org/moin/TimeComplexity).

### 10. Memory management: refcounting + cyclic GC, and classic gotchas

- CPython frees objects primarily by **reference counting** (immediate, deterministic), plus a **generational cyclic garbage collector** to catch reference *cycles* refcounting can't. Three generations; objects surviving collections get promoted; `gc` is tunable/disable-able.
- **`is` vs `==`**: identity vs equality. Don't use `is` for value comparison (works for small-int caching/`None` by accident, breaks generally). Compare to `None` with `is None`.
- **Mutable default argument** trap: `def f(x, acc=[])` — the list is created **once** at def time and shared across calls. Use `acc=None` then `acc = acc or []`.
- **Late-binding closures** in loops capture the variable, not its value (`lambda: i` in a loop).

Sources: [data model](https://docs.python.org/3/reference/datamodel.html) · [gc module](https://docs.python.org/3/library/gc.html).

---

## C. Previously asked (from real HRT reports)

1. **"How does the `with` keyword work?"** *(SWE, Singapore)*
2. **"What is a `yield` statement used for? What are a generator and a decorator?"** + Linux Qs *(SRE, US)*
3. **"Explain how Python implements dictionaries."** *(Systems SWE, NY)*
4. **Array sort using a lambda function / sort with a custom key.** *(SWE, NY — Glassdoor)*
5. **Pandas data-manipulation OA** — groupby, pivot, merge, missing values, aggregation. *(SWE intern, US; also the "pandas q" in the file-IO prescreen report)*
6. **Resume/project deep-dive** — "describe a project you've done"; deep investigation/debugging experience. *(Data Eng, NY; SWE via recruiter)*
7. **"Design a concurrent framework (e.g. Python's asyncio)."** *(SWE intern, Singapore — verbalize design)*
8. **SSH into a remote box and debug a flaky Python service** ("debugging round"). *(Blind/Reddit, Systems/Python loop)* — see `HRT_03` for the Linux side.

---

## D. Extrapolated Q&A with model answers

**Q1. A long-running Python service's memory grows over days. How do you find the leak?**
> First confirm it from the OS — watch RSS over time via `/proc/<pid>/status` or `ps`. Then attribute it inside Python: take a `tracemalloc` baseline snapshot, let it run, snapshot again, and diff to see which lines/types are growing. I follow the top growers' referrer chains with `objgraph` to find what's holding them — usually an ever-growing module-level cache, an unbounded `lru_cache`, a closure capturing big state, or a registry that never evicts. If Python-level tools show nothing but RSS still climbs, I suspect a C-extension missing a `Py_DECREF` or arena fragmentation, and I'd `py-spy dump` to see what it's doing. Fix is usually bounding the cache or breaking the retained reference.

**Q2. You have a CPU-bound Python job pinned to one core despite using threads. Why, and what do you do?**
> That's the GIL — only one thread runs Python bytecode at a time, so CPU-bound threads serialize. Options: move to `multiprocessing`/`ProcessPoolExecutor` for true parallelism (accepting IPC/serialization cost), or push the hot loop into native code — NumPy, Cython, or a C/C++ extension — and release the GIL in that section so threads actually run in parallel. If it were I/O-bound instead, threads or `asyncio` would already help because the GIL is released during blocking I/O. And as of 3.13+ there's the free-threaded build that removes the GIL, though I'd verify C-extension compatibility before relying on it in production.

**Q3. How would you profile a slow request path in production without taking the service down?**
> `py-spy`, because it attaches to the running PID from outside the process and samples stacks with near-zero overhead and no code change — `py-spy top` for a live view, `py-spy record` for a flame graph. Sampling shows me where wall-clock actually goes. Once I've localized the hot function I can reproduce locally under `cProfile` for exact call counts, or add targeted timing. I specifically avoid `cProfile` in prod because its per-call instrumentation overhead distorts a latency-sensitive service.

**Q4. (Live) Write a decorator that retries a function up to N times with backoff.**
> *(Write it: factory → `deco(fn)` → `wrapper(*a, **k)` loop with `functools.wraps`, catch exception, sleep growing delay, re-raise on final attempt. Narrate the three nesting levels.)*

**Q5. What's the difference between a list and a generator for processing a 50 GB file?**
> A list would try to materialize everything in memory — 50 GB of objects, which won't fit. A generator yields one record at a time, holding O(1) memory, so I can stream the file through a transform/filter pipeline lazily. I'd read line-by-line (the file object is itself an iterator) and chain generator expressions. It trades random access for memory, which is the right trade for a streaming job.

**Q6. Why are Python lists slow for numeric work compared to NumPy or a C++ vector?**
> A Python list stores pointers to individually heap-allocated boxed objects, so iterating chases pointers across the heap with poor cache locality, plus per-element interpreter overhead. NumPy (and a C++ `vector<int>`) stores unboxed values contiguously, so it's cache-friendly and vectorizable. That's why the fix for a slow numeric loop is usually "vectorize into NumPy," which runs the loop in C with the GIL released.

**Q7. `sorted(data, key=lambda x: ...)` — how does sorting with a key work, and what's the complexity?**
> `sorted`/`list.sort` use **Timsort**, a stable, adaptive merge/insertion hybrid — O(n log n) worst case, O(n) on already-ordered runs. The `key` function is called **once per element** to produce a decorated sort key (the "decorate–sort–undecorate" / Schwartzian pattern, done internally), which is why a `key` is faster than an old-style comparator that's called O(n log n) times. Stability means equal keys keep input order, which lets you sort by multiple fields in passes.

**Q8. Explain `__enter__`/`__exit__` and write a context manager for a timed lock.**
> *(Explain per B-5, then either a class with both dunder methods or `@contextlib.contextmanager`: acquire before `yield` in a `try`, release in `finally`. Note `__exit__` runs even on exception and can suppress it by returning truthy.)*

---

## E. Live-coding warm-ups (Python)

- A **decorator** (timing/retry/cache) with `functools.wraps` and a parameterized variant.
- A **context manager** both ways (class and `@contextmanager`).
- A **generator pipeline** over a large file (filter → map → aggregate).
- **Pandas mini-drill** (for OA insurance): `groupby().agg()`, `pivot_table`, `merge`, `fillna`/`dropna`, sorting with a key. The intern OA reports were explicitly pandas-heavy.
- `sorted` with a custom `key` / multi-key sort; `collections.deque`, `defaultdict`, `Counter`, `heapq` fluency.

---

## F. Segment tips

- **Lead with the practitioner framing:** when Python comes up, steer toward *"here's how I profiled and fixed X"* — production experience is the differentiator. Have a tool name attached to every story (`py-spy`, `cProfile`, `tracemalloc`, `perf`).
- **But don't get caught on trivia.** Have crisp 30-second answers for `with`, generators, decorators, dict internals, list growth, GIL — these are *documented* HRT questions, and the firm's own blog uses list-growth as a litmus test.
- **Know the *current* GIL story** (3.13 free-threaded, PEP 703). It's an easy way to read as up-to-date for a 2026 interview.
- **Bridge to C++** when natural ("`with` ≈ RAII", "list-of-pointers vs contiguous vector"). Cross-language fluency is exactly what the SWE-everywhere model rewards.
- For the **SSH/debugging round** (some Python/Systems loops do this), see `HRT_03` — practice fixing a broken service on a real box, narrating each diagnostic step.
- Write **idiomatic, readable** Python (comprehensions, context managers, `pathlib`, f-strings) — "idiomatic, modern syntax" is on their scorecard.
