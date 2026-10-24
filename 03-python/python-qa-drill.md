# Python Q&A Drill

**Q: py-spy vs cProfile?**
A: `cProfile` = built-in deterministic profiler, instruments every call, exact counts/timings, meaningful overhead, in-process — for local/dev deep dives. `py-spy` = sampling profiler (Rust) that attaches to an already-running PID from *outside*, near-zero overhead, no code changes — the production triage tool. Sampling tells you where wall-clock goes; deterministic gives exact call counts.

**Q: What causes "memory leaks" in long-running Python?**
A: Usually unintended *retention*, not lost pointers — refcounting frees most things. Lingering refs (ever-growing module-level caches/dicts, `lru_cache` with no `maxsize`, global registries, tracebacks holding frames), reference cycles complicated by `__del__` or C-extensions, true C-extension leaks (missing `Py_DECREF`), and `pymalloc` arena fragmentation. Tools: `tracemalloc` (snapshot + diff by line), `objgraph` (referrer chains), `gc`, `py-spy dump`, RSS via `/proc/<pid>/status`.

**Q: What is the GIL and its practical consequence?**
A: A mutex letting only one thread execute Python bytecode at a time, protecting CPython internals (notably refcounts). Consequence: threads don't parallelize CPU-bound pure-Python work (one core pinned, others idle), but they *do* help I/O-bound work because the GIL is released during blocking I/O and inside many C extensions (NumPy).

**Q: How do you work around the GIL?**
A: `multiprocessing`/`ProcessPoolExecutor` (true parallelism, IPC/serialization cost), `asyncio` (single-thread I/O concurrency, no thread overhead), release the GIL in native code (NumPy/Cython), or push the hot loop out of Python entirely.

**Q: The current (2024–2026) GIL story?**
A: PEP 703 made the GIL optional. Python 3.13 (Oct 2024) shipped an experimental free-threaded build (`python3.13t`) where the GIL can be disabled — true multi-core threading — at the cost of single-thread overhead and a real thread-safety burden (object-level locking replaces the one global lock). 3.14 cut the single-thread penalty; C-extension ecosystem still catching up. Check at runtime with `sys._is_gil_enabled()`. Not yet a drop-in for prod with C-ext deps.

**Q: When and how do you drop to native code?**
A: Measure first (`py-spy`/`cProfile`), confirm it's CPU-bound *and* in Python land (not I/O or an existing C lib). Then move the narrowest hot path native — vectorize with NumPy, Cython, a C/C++ extension via pybind11, ctypes/cffi for an existing lib, or PyO3 for Rust — and *release the GIL* in that section so threads scale.

**Q: How does `with` work?**
A: It runs a context manager — an object with `__enter__` and `__exit__`. On entry `__enter__()` runs (return binds to the `as` name); on exit, normal or exceptional, `__exit__(exc_type, exc_val, tb)` runs cleanup. If `__exit__` returns truthy it suppresses the exception. It's Python's RAII equivalent, but tied to the explicit block rather than scope/destructors. Write one as a class or with `@contextlib.contextmanager` on a generator (before `yield` = enter, after = exit).

**Q: What's a generator / `yield`?**
A: A function containing `yield` is a generator function; calling it returns a generator (iterator) without running the body. Each `next()` runs to the next `yield`, produces a value, and suspends — local state + instruction pointer frozen on the frame, resumed next call. Benefits: lazy/streaming evaluation and O(1) memory over huge or infinite sequences. `yield from` delegates to a sub-iterator.

**Q: What's a decorator?**
A: A callable that takes a function/class and returns a replacement; `@deco` is sugar for `f = deco(f)`. For cross-cutting concerns (logging, timing, caching, retries, auth, registration) without touching the wrapped body. Be ready to write one live: `*args, **kwargs` passthrough, `functools.wraps` to preserve metadata, and a *parameterized* one (a function returning a decorator — the third nesting level trips people up).

**Q: How does CPython implement dict?**
A: Open-addressing hash table (not chaining). A sparse index array maps `hash(key) mod size` to a slot; collisions resolved by a perturbation probing sequence. Avg O(1), worst O(n). Resizes when ~2/3 full. Since 3.6/3.7 it's insertion-ordered with a compact "split" layout — a dense `entries` array holds (hash, key, value) in insertion order and the sparse array stores indices into it; cut dict memory ~20–25% and gave ordering for free. Contract: equal objects must hash equal; keys must be hashable.

**Q: list memory growth characteristic?**
A: A list is a dynamic array of `PyObject*` pointers (not the objects) plus length and capacity. On append past capacity it over-allocates geometrically: roughly `new = old + (old >> 3) + 6` → about **1.125× plus a constant, NOT 2×** (common wrong answer). So append is amortized O(1); `insert(0)`/`pop(0)` are O(n) → use `deque` for a queue; indexing O(1); each element is a pointer to a heap object, so far less cache-friendly than NumPy or `vector<int>`.

**Q: How does Python manage memory?**
A: Primarily reference counting (immediate, deterministic) plus a generational cyclic GC (3 generations) to catch reference cycles refcounting can't.

**Q: is vs ==?**
A: `is` = identity, `==` = value equality. Don't use `is` for values (works for small-int caching/`None` by accident, breaks generally). Compare to None with `is None`.

**Q: Mutable default argument trap?**
A: `def f(x, acc=[])` — the list is created once at def time and shared across calls. Use `acc=None` then `acc = acc or []`.

**Q: Sort with a custom key / lambda?**
A: `sorted(xs, key=lambda x: x.field)` or `xs.sort(key=...)`. Stable; key computed once per element; `reverse=True` for descending.

---

## Escalation chains

### Chain 2A — the GIL, in practice (lead with the practitioner answer)

**Prompt:** "Tell me about the GIL — not the textbook line, but how it's actually affected something you've shipped or debugged."
> The GIL means only one thread runs Python bytecode at a time, so CPU-bound pure-Python work doesn't parallelize across threads — you'll see one core pinned while the rest idle. Where it *doesn't* hurt is I/O-bound work, because the GIL is released during blocking I/O, and during heavy native compute like NumPy. In practice when I hit a CPU-bound Python bottleneck I either move to `multiprocessing`/a process pool for true parallelism, accepting IPC cost, or push the hot path into native code and release the GIL there. First, though, I profile to confirm it's actually CPU-bound in Python land and not I/O or already in a C library.

**Follow-up:** "And the 3.13 free-threading stuff — is the GIL just gone now?"
> Not by default. PEP 703 made it optional: 3.13 ships an experimental free-threaded build, `python3.13t`, where the GIL is disabled and threads can run Python in parallel — at the cost of some single-thread overhead and the burden of real thread-safety, since it's now object-level locking instead of one global lock. 3.14 improved the single-thread penalty. I wouldn't move a production service with C-extension dependencies onto it yet without checking the compatibility tracker — you can verify at runtime with `sys._is_gil_enabled()`.

### Chain 2B — profiling

**Prompt:** "A Python service in prod is slow and you can't take it down. How do you find where the time's going?"
> `py-spy`, because it attaches to the running PID from outside the process and samples stacks with near-zero overhead and no code change — `py-spy top` for a live view, `py-spy record` for a flame graph. Sampling tells me where wall-clock actually goes. Once I've localized the hot function I can reproduce locally under `cProfile` for exact call counts. I specifically avoid `cProfile` in prod because its per-call instrumentation distorts a latency-sensitive service.

**Follow-up:** "How does py-spy read another process's stack without stopping it? That seems like magic."
> It reads the target process's memory directly — on Linux via `process_vm_readv` / `/proc/<pid>/mem` — and walks CPython's interpreter data structures (the thread-state and frame objects) to reconstruct the Python stack, knowing the interpreter's memory layout for that version. It's sampling, so it peeks periodically rather than tracing every call, which is why the overhead on the target is negligible. That's also why it needs `ptrace`-level permissions.

### Chain 2C — quick internals (don't get caught out)

**Prompt:** "Fast ones. How does a Python `dict` work? And what's the memory growth of a `list`?"
> `dict` is an open-addressing hash table — collisions resolved by a probe sequence, not chaining — average O(1), resizes when it's ~2/3 full, and since 3.6/3.7 it's insertion-ordered via a compact layout where a dense entries array holds the items and a sparse array holds indices into it. `list` is a dynamic array of `PyObject` pointers; on append past capacity it over-allocates geometrically — roughly old + old/8 + a constant, so about 1.125×, not 2× — giving amortized O(1) appends but O(n) inserts/pops at the front, which is why I use `deque` for a queue.

---
