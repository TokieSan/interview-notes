# C++ Fundamentals — the "why" behind everything

For every topic be able to (1) explain it plainly, (2) say *why it exists or how the compiler implements it*, (3) write a 5-line demo from memory. That triad is exactly what systems / trading-firm interviewers probe.

## A. The topic map (what to be sharp on)

From recruiter guidance + documented HRT questions:

1. Memory model: stack vs heap, object layout, cache-aware design
2. Object lifetime & storage duration (static, automatic, dynamic, thread); `const`/`constexpr`
3. RAII & smart pointers (`unique_ptr`, `shared_ptr`, `weak_ptr`)
4. Move semantics, rvalue references, copy elision
5. Virtual functions, vtables, vptr, overhead, `final`/`override`
6. References vs pointers; `inline`; the "rule of 0/3/5"
7. Atomics, memory ordering, lock-free (conceptual)
8. Undefined behavior & common pitfalls
9. STL containers — internals & complexity
10. "How does a function work under the hood" (calling convention, stack frame, ABI)

---

## B. Topic-by-topic — explanation, *why*, and sources

### 0. Notes

Rvalues, Lvalues, move semantics, forward semantics
- tldr: https://agop.me/post/perfect-forwarding-cpp-11.html
- https://drewcampbell92.medium.com/understanding-move-semantics-and-perfect-forwarding-987cf4dc7e27
- https://drewcampbell92.medium.com/understanding-move-semantics-and-perfect-forwarding-part-2-6b8266b6cfa4
- https://drewcampbell92.medium.com/understanding-move-semantics-and-perfect-forwarding-part-3-65575d523ff8

### 1. Memory model: stack vs heap, layout, cache

- **Stack**: automatic storage. Allocation is a single pointer bump of the stack pointer; deallocation is automatic at scope exit (LIFO). Extremely cheap, cache-hot, but bounded (typically a few MB) → deep recursion / huge locals = stack overflow.
- **Heap**: dynamic storage via `new`/`malloc`. Flexible lifetime and size, but allocation involves an allocator (free-list/arena management), can fragment, and pointers chase across cache lines.
- **Cache-aware design**: a modern L1 hit is ~1ns; a main-memory miss is ~100ns — a ~100× gap. So **data layout dominates performance**: contiguous arrays (`std::vector`) beat node-based structures (`std::list`, `std::map`) for traversal because of spatial locality and hardware prefetching. This is the "mechanical sympathy" point HRT loves — tie it to "struct of arrays vs array of structs" and avoiding pointer chasing on hot paths.
- **Why it matters at HRT:** in low-latency code, the win is usually not a better Big-O — it's fewer cache misses and branch mispredictions.

Sources: [cppreference — storage duration](https://en.cppreference.com/w/cpp/language/storage_duration) · [What Every Programmer Should Know About Memory (Drepper)](https://akkadia.org/drepper/cpumemory.pdf) · "latency numbers every programmer should know" (Dean/Norvig figures).

### 2. Object lifetime & storage duration (a *documented* HRT question)

> Real HRT question (C++ SWE intern, NY): *"Describe object lifetime with regards to static and constant variables — at function scope and global scope."*

Four storage durations:
- **automatic** — locals; live from declaration to end of enclosing block.
- **static** — lifetime = whole program. Two flavors:
  - *Global / namespace-scope* statics: constructed before `main` (within a translation unit, in definition order; **across** TUs the order is unspecified → the *static initialization order fiasco*). Destroyed in reverse after `main`.
  - *Function-local* statics: constructed **on first control-flow pass** through the declaration, and (since C++11) that init is **thread-safe** ("magic statics"). Destroyed at program exit in reverse order of construction. This is the canonical safe-singleton idiom.
- **dynamic** — `new`/`delete`; you own the lifetime.
- **thread** — `thread_local`; one instance per thread.

`const` vs `constexpr`: `const` = "I won't modify this through this name" (runtime or compile-time). `constexpr` = "usable in a constant expression / evaluable at compile time." A namespace-scope `constexpr`/`const` integral with a constant initializer can avoid the init-order fiasco because it's a compile-time constant.

Sources: [cppreference — object lifetime](https://en.cppreference.com/w/cpp/language/lifetime) · [storage duration](https://en.cppreference.com/w/cpp/language/storage_duration) · [static init order fiasco (isocpp FAQ)](https://isocpp.org/wiki/faq/ctors#static-init-order).

### 3. RAII & smart pointers

- **RAII** = Resource Acquisition Is Initialization: bind a resource's lifetime to an object's lifetime, so the destructor releases it deterministically. This is *the* C++ idiom and the reason exceptions are safe — stack unwinding runs destructors. Frame your whole memory-safety story around RAII.
- **`std::unique_ptr<T>`** — sole ownership, **zero overhead** vs a raw pointer (same size, no refcount) for the default deleter. Move-only. Default tool.
- **`std::shared_ptr<T>`** — shared ownership via an atomic reference count in a separate **control block**. Costs: extra allocation (use `make_shared` to fuse object + control block into one allocation), atomic inc/dec on copy, and `sizeof` is two pointers (object + control block).
- **`std::weak_ptr<T>`** — non-owning observer; breaks `shared_ptr` **reference cycles**. `lock()` to get a `shared_ptr` if still alive.
- **Why prefer `make_unique`/`make_shared`:** exception safety (no leak if an arg evaluation throws between `new` and the smart-pointer ctor) and, for `make_shared`, one allocation instead of two.

Sources: [unique_ptr](https://en.cppreference.com/w/cpp/memory/unique_ptr) · [shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr) · [C++ Core Guidelines — Resource management (R)](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#S-resource).

### 4. Move semantics, rvalue references, copy elision

- **Problem it solves:** copying owns-a-buffer objects (vector, string) is O(n) and wasteful when the source is a temporary about to die. Moving *transfers ownership* of the internal buffer — O(1) — leaving the source in a valid-but-unspecified state.
- **rvalue reference `T&&`** binds to temporaries/expiring values. A move ctor/assignment takes `T&&`, steals the pointer, nulls the source.
- **`std::move`** is just a cast to `T&&` — it doesn't move anything by itself; it marks "you may pillage this."
- **When it matters:** returning/holding large containers, `emplace_back`, transferring `unique_ptr`. Also **perfect forwarding** with `T&&` template params + `std::forward`.
- **Copy elision / RVO:** the compiler can construct the return value directly in the caller's slot, skipping even the move. Since **C++17, NRVO-eligible and prvalue returns are guaranteed elision** in the prvalue case — so `return MyType(...)` does *not* call a move ctor at all.
- **Rule of 0/3/5:** prefer **Rule of 0** (let RAII members manage everything, declare no special members). If you declare one of {dtor, copy ctor, copy assign}, you likely need all three (Rule of 3); add move ctor + move assign for Rule of 5. Declaring a dtor suppresses implicit move generation — a classic gotcha.

Sources: [move ctor](https://en.cppreference.com/w/cpp/language/move_constructor) · [std::move](https://en.cppreference.com/w/cpp/utility/move) · [copy elision](https://en.cppreference.com/w/cpp/language/copy_elision) · [rule of 3/5/0](https://en.cppreference.com/w/cpp/language/rule_of_three) · Meyers, *Effective Modern C++*, Items 23–30.

### 5. Virtual functions, vtables, vptr

- **What:** dynamic dispatch — call resolved by the object's *runtime* type, not the static type of the pointer/reference.
- **How compilers usually implement it (this is the "why" they want):** each polymorphic class has one **vtable** (array of function pointers, one per virtual function). Each object of such a class carries a hidden **vptr** pointing to its class's vtable. A virtual call = load vptr → index into vtable → indirect call. So overhead is:
  - **per object:** +1 pointer (the vptr) → larger objects, worse cache density.
  - **per call:** an extra indirection + a likely **branch misprediction / can't inline** → the real cost on hot paths.
- **Other facts to have ready:** constructors are never virtual; destructors of polymorphic base classes **should** be virtual (else `delete base_ptr` is UB / leaks the derived part). `final` can let the compiler **devirtualize** and inline. Calling a virtual function *inside a constructor/destructor* dispatches to the type currently being constructed (not the most-derived) — a classic trap.

Sources: [cppreference — virtual functions](https://en.cppreference.com/w/cpp/language/virtual) · [object slicing / `final`](https://en.cppreference.com/w/cpp/language/final) · Lippman, *Inside the C++ Object Model* (for layout).

### 6. References vs pointers, `inline` (documented HRT topics)

> Real HRT questions: *"C++ topics like inline and references…"* and *"classic C++ coding questions regarding pointers, references."*

- **Reference vs pointer:** a reference must be initialized, can't be rebound, can't be null (without UB), no arithmetic; a pointer is nullable, rebindable, supports arithmetic, has its own storage. Use references for "always-present alias," pointers for "optional/rebindable/owning-via-smart-pointer." Under the hood references are *usually* implemented as pointers but the standard doesn't require storage.
- **`inline`:** the modern meaning is **not** "please inline this for speed" (that's a non-binding hint the optimizer ignores/applies on its own). Its real, guaranteed semantic is an **ODR exemption**: an `inline` function/variable may be defined in multiple translation units (e.g. in a header) and the linker merges them into one. `inline` variables (C++17) solve the header-only global problem.

Sources: [reference declaration](https://en.cppreference.com/w/cpp/language/reference) · [inline specifier](https://en.cppreference.com/w/cpp/language/inline).

### 7. Atomics, memory ordering, lock-free (conceptual)

- **`std::atomic<T>`** gives indivisible operations with no data race; for small types it's lock-free (check `is_lock_free()`), implemented via CPU instructions like `LOCK`-prefixed ops / `cmpxchg` / LL-SC.
- **Memory ordering** (the part they may push on): `memory_order_seq_cst` (default; single global order, simplest, most expensive) → `acquire`/`release` (pair to establish happens-before for a handoff, e.g. publish-a-pointer-then-set-a-flag) → `relaxed` (atomicity only, no ordering — e.g. a stats counter).
- **Lock-free idea:** algorithms that guarantee *system-wide* progress without mutexes, typically built on **CAS (compare-and-swap)** retry loops. Name the hazards: the **ABA problem** (value goes A→B→A and CAS wrongly succeeds; fixed with tagged pointers/hazard pointers) and reclamation safety. You don't need to implement a lock-free queue cold — but be able to *sketch a CAS loop* and explain acquire/release.

Sources: [std::atomic](https://en.cppreference.com/w/cpp/atomic/atomic) · [memory_order](https://en.cppreference.com/w/cpp/atomic/memory_order) · [cppreference — memory model](https://en.cppreference.com/w/cpp/language/memory_model).

### 8. Undefined behavior & common pitfalls

Have a crisp definition: **UB = the standard imposes no requirements**; the compiler may assume it never happens and optimize accordingly, so symptoms can be "works on my machine then breaks under -O2." Greatest-hits to recognize on sight:
- use-after-free / dangling reference (returning reference/pointer to a local)
- reading uninitialized values; out-of-bounds (`v[i]` past end; `[]` doesn't check, `.at()` does)
- signed integer overflow (UB; unsigned wraps, defined)
- data races (concurrent non-atomic access with ≥1 write)
- modifying an object twice without an intervening sequence point pre-C++17 (`i = i++`)
- invalidated iterators (e.g., `push_back` reallocating a vector mid-loop)
- null/`this` deref; violating strict aliasing; calling a pure virtual during construction.

Sources: [cppreference — UB](https://en.cppreference.com/w/cpp/language/ub) · [LLVM blog: "What Every C Programmer Should Know About UB"](https://blog.llvm.org/2011/05/what-every-c-programmer-should-know.html).

### 9. STL containers — internals & complexity (documented: "OS and STL")

Know the **mechanism**, not just the table:
- `vector` — contiguous; amortized O(1) push_back via **geometric growth** (typically ×1.5 or ×2). `reserve` to avoid reallocations; reallocation invalidates all iterators/pointers.
- `deque` — segmented array of fixed blocks; O(1) push front/back; not contiguous.
- `list` — doubly linked; O(1) splice; terrible locality.
- `map`/`set` — red-black tree, O(log n), ordered, node-based (pointer chasing).
- `unordered_map`/`set` — hash table (buckets + chaining in libstdc++), avg O(1), worst O(n); references stable across rehash but iterators not.
- Iterator categories and which ops invalidate them is a favorite follow-up.

Sources: [cppreference — containers library](https://en.cppreference.com/w/cpp/container).

### 10. "How does a function work under the hood?" (a *documented* HRT question)

> Real HRT question (SWE intern, Canada): *"How does a function work under the hood?"*

Walk the stack frame story: caller places args per the **calling convention/ABI** (System V AMD64: first 6 integer args in `rdi, rsi, rdx, rcx, r8, r9`, rest on stack; return in `rax`), executes `call` (pushes return address). Callee sets up a frame (push `rbp`, `mov rbp, rsp`, subtract for locals), does its work, places the return value, tears down the frame, and `ret` pops the return address. Mention the **red zone**, stack growing downward, who saves which registers (caller- vs callee-saved), and that `inline`/optimization can elide the whole frame. Connecting this to "that's why deep recursion overflows the stack" and "that's why a vtable call is just an extra indirect `call`" scores big.

Sources: [System V AMD64 ABI (PDF)](https://gitlab.com/x86-psABIs/x86-64-ABI) · [cppreference — functions](https://en.cppreference.com/w/cpp/language/functions).

---

## Practice C++ questions (trading firms)

Questions and topics to drill — each until you can answer cold *and* go two follow-ups deep (they "probe anything you say"):

1. **Object lifetime of static and const variables**, at function scope and global scope. *(C++ SWE intern, NY)*
2. **`inline` and references**, plus OS and STL. *(C++ Graduate SWE, Singapore — phone screen)*
3. **Pointers and references** classic coding questions.
4. **"How does a function work under the hood?"** *(SWE intern, Canada)*
5. **"Basics, basics, basics… how the OS works and how C++ code is interpreted by the compiler."** *(C++ SWE intern, NY)*
6. **Low-level concepts: OS, compilers, C++, memory** — no coding, pure discussion. *(SWE intern, France)*
7. **Tic-tac-toe-like / Lines-and-Squares game implementation** in C++ (implementation-heavy, debug fast). *(multiple)*

---

## Live-coding warm-ups (write these from memory, fast)

These mirror the implementation-heavy style HRT uses ("tic-tac-toe-like game," "lines and squares," "balanced tree traversal"):

1. **Tic-tac-toe / generalized N-in-a-row engine** — board state, `make_move`, `check_winner`, input validation. Practice clean encapsulation in a class with a clear interface; they grade idiomatic, readable code and *fast debugging*.
2. **LRU cache** — `unordered_map<Key, list<...>::iterator>` + `list` for O(1) get/put. Classic systems-flavored data structure.
3. **A small `unique_ptr`-like class** — templated, move-only, with `operator*`/`operator->`, correct Rule-of-5. Shows you internalized ownership.
4. **Balanced-BST traversal / rebuild** — one report walked from "traverse a BBST" to "rebuild it" to subtree-walk complexity reasoning. Know in-order traversal and how to rebuild a balanced tree from sorted data in O(n).
5. **Object pool / arena allocator (sketch)** — even verbally, shows cache/allocation awareness valued in trading tech.

---

## Tips

- **Lead with C++ when offered the choice** — it's your edge and the firm's depth bar. Say you're also comfortable in Python so they know they can pivot.
- **Always give the "why."** "Vtable" alone is a B-grade answer; "vtable, plus the per-object vptr cost and the per-call mispredict that blocks inlining" is the A-grade answer that matches HRT's stated fundamentals bar.
- **Anchor to a standard version** ("as of C++17, this elision is guaranteed"). It protects you if the interviewer's knowledge is dated and signals modern fluency.
- **Narrate while coding**; check the interviewer is following. Communication is scored as heavily as the code.
- **Find your own bugs out loud** — compile-in-your-head, trace a small input. Fast, calm debugging is a documented differentiator.
- **Don't over-engineer.** "Avoiding real work" (over-complex solutions) is a named failure mode — write the direct, correct thing first, optimize if asked.
- If you genuinely don't know something, say so and reason from fundamentals — composure under not-knowing is explicitly valued.
