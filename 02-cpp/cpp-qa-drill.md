# C++ Q&A Drill

Cover the answer, say yours out loud against a timer, then compare.

## Rapid Q&A (say these out loud)

**Q: Stack vs heap?**
A: Stack = automatic storage, allocation is a single pointer bump, freed at scope exit (LIFO), cache-hot but bounded (~few MB) → deep recursion / huge locals overflow it. Heap = dynamic via `new`/`malloc`, flexible lifetime and size, but allocator overhead, fragmentation, and pointers chase across cache lines.

**Q: Why does data layout beat Big-O on hot paths?**
A: L1 hit ~1ns, main-memory miss ~100ns — 100× gap. Contiguous arrays (`vector`) beat node-based structures (`list`/`map`) for traversal because of spatial locality + hardware prefetch. In low-latency code the win is usually fewer cache misses and branch mispredicts, not better complexity. (Struct-of-arrays vs array-of-structs, avoid pointer chasing.)

**Q: Four storage durations?**
A: automatic (locals, scope-bound), static (whole program), dynamic (`new`/`delete`, you own it), thread (`thread_local`, one per thread).

**Q: Object lifetime of static vars — function scope vs global scope?**
A: Global/namespace statics: constructed before `main` (definition order within a TU; unspecified *across* TUs → static init order fiasco), destroyed in reverse after `main`. Function-local statics: constructed on first control-flow pass through the declaration, thread-safe since C++11 ("magic statics"), destroyed at exit in reverse construction order — the safe-singleton idiom.

**Q: const vs constexpr?**
A: `const` = "I won't modify through this name" (runtime *or* compile-time). `constexpr` = "evaluable at compile time / usable in a constant expression." A `constexpr` integral with a constant initializer dodges the init-order fiasco.

**Q: What is RAII?**
A: Resource Acquisition Is Initialization — tie a resource's lifetime to an object's lifetime so the destructor releases it deterministically. It's *why* exceptions are safe: stack unwinding runs destructors.

**Q: unique_ptr vs shared_ptr vs weak_ptr?**
A: `unique_ptr` = sole ownership, zero overhead vs a raw pointer, move-only — default choice. `shared_ptr` = shared ownership via an atomic refcount in a control block; costs an extra allocation, atomic inc/dec on copy, two pointers wide. `weak_ptr` = non-owning observer, breaks `shared_ptr` cycles, `lock()` to get a `shared_ptr` if still alive.

**Q: Why make_shared / make_unique?**
A: Exception safety (no leak if an arg throws between `new` and the smart-ptr ctor); `make_shared` fuses object + control block into one allocation instead of two.

**Q: What problem do move semantics solve?**
A: Copying a buffer-owning object (vector/string) is O(n) and wasteful when the source is a temporary about to die. Move transfers ownership of the internal buffer — O(1) — leaving the source valid-but-unspecified.

**Q: What is std::move?**
A: Just a cast to `T&&`. It moves nothing itself; it marks an object as "you may pillage this." The actual move happens in the move ctor/assignment that takes the `T&&`.

**Q: Copy elision / RVO?**
A: Compiler constructs the return value directly in the caller's slot, skipping even the move. Since C++17, prvalue returns are *guaranteed* elision — `return MyType(...)` calls no move ctor at all.

**Q: Rule of 0/3/5?**
A: Prefer Rule of 0 — let RAII members manage everything, declare no special members. If you declare one of {dtor, copy ctor, copy assign} you probably need all three (Rule of 3); add move ctor + move assign for Rule of 5. Gotcha: declaring a dtor suppresses implicit move generation.

**Q: How are virtual functions implemented?**
A: Each polymorphic class has one vtable (array of function pointers). Each object carries a hidden vptr to its class's vtable. A virtual call = load vptr → index into vtable → indirect call. Overhead: +1 pointer per object (worse cache density), and a per-call indirection that can't inline + possible branch mispredict — the real hot-path cost.

**Q: Why must a polymorphic base's destructor be virtual?**
A: Otherwise `delete base_ptr` is UB and destroys only the base part. Ctors are never virtual. `final` lets the compiler devirtualize and inline.

**Q: Trap with virtual calls inside a ctor/dtor?**
A: They dispatch to the type *currently being constructed*, not the most-derived type.

**Q: Reference vs pointer?**
A: Reference must be initialized, can't be rebound, can't be null without UB, no arithmetic. Pointer is nullable, rebindable, has arithmetic and its own storage. Use references for an always-present alias, pointers for optional/rebindable. Usually implemented as pointers under the hood, but the standard doesn't require storage.

**Q: What does `inline` actually mean?**
A: Not "please inline for speed" — that's a non-binding hint the optimizer ignores or applies on its own. Its real guaranteed meaning is an ODR exemption: the function/variable can be defined in multiple TUs (e.g. in a header) and the linker merges them into one. `inline` variables (C++17) solve header-only globals.

**Q: std::atomic and memory ordering?**
A: `atomic<T>` = indivisible ops, no data race; lock-free for small types (`is_lock_free()`), built on CPU instructions (`LOCK`-prefix / `cmpxchg`). Orderings: `seq_cst` (default, single global order, simplest, most expensive) → `acquire`/`release` (pair to establish happens-before for a handoff — publish pointer then set flag) → `relaxed` (atomicity only, no ordering — e.g. a stats counter).

**Q: Lock-free / CAS / ABA?**
A: Lock-free = guarantees system-wide progress without mutexes, usually a compare-and-swap retry loop. ABA = value goes A→B→A and CAS wrongly succeeds; fix with tagged pointers / hazard pointers.

**Q: What is undefined behavior?**
A: The standard imposes no requirements; the compiler may assume it never happens and optimize accordingly — so it can work at `-O0` then break at `-O2`. Greatest hits: use-after-free / dangling ref, uninitialized reads, out-of-bounds, signed overflow, data races, iterator invalidation (`push_back` reallocating mid-loop), `i = i++` pre-C++17.

**Q: vector internals?**
A: Contiguous; amortized O(1) `push_back` via geometric growth (×1.5 or ×2); `reserve` avoids reallocations; reallocation invalidates all iterators and pointers.

**Q: map vs unordered_map?**
A: `map` = red-black tree, O(log n), ordered, node-based (pointer chasing). `unordered_map` = hash table (buckets + chaining), avg O(1) / worst O(n); references stay stable across rehash, iterators don't.

**Q: How does a function work under the hood?**
A: Caller puts args per the ABI (System V AMD64: first 6 integer args in `rdi, rsi, rdx, rcx, r8, r9`, rest on stack; return in `rax`), `call` pushes the return address. Callee sets up a frame (`push rbp; mov rbp,rsp; sub rsp` for locals), runs, places the return value, tears down, `ret` pops the return address. Stack grows downward; caller- vs callee-saved registers; red zone; inlining can elide the whole frame. That's why deep recursion overflows the stack, and why a vtable call is just an extra indirect `call`.

---

## Longer model answers

> Answers written the way you'd *say* them — concise, then ready to go deeper. C++ is primary; a couple cross over to Python where HRT might pivot.

**Q1. What's the difference between `unique_ptr` and `shared_ptr`, and what does each cost?**
> `unique_ptr` is sole ownership and is essentially free — same size as a raw pointer, no reference count, move-only. `shared_ptr` is shared ownership with an atomic reference count in a separate control block, so it's two pointers wide, the copies pay atomic increments/decrements, and naïvely you get two allocations — one for the object, one for the control block — which `make_shared` fuses into one. I default to `unique_ptr` and only reach for `shared_ptr` when ownership is genuinely shared, and `weak_ptr` to break cycles. *(Follow-up ready: what happens on the last `shared_ptr` destruction → refcount hits 0 → object destroyed; weak count governs the control block.)*

**Q2. Explain move semantics and when it actually matters.**
> A move transfers ownership of an object's internal resources instead of deep-copying them — O(1) vs O(n) for something like a vector or string. The move constructor takes an rvalue reference, steals the pointer, and leaves the source valid-but-empty. `std::move` is just a cast to `T&&` that says "this is safe to pillage." It matters whenever you'd otherwise copy a temporary that's about to die — returning big containers, `emplace_back`, handing off a `unique_ptr`. Though in many return cases the compiler skips even the move via copy elision, which C++17 made mandatory for prvalues. *(Follow-up: Rule of 5; declaring a destructor suppresses the implicit move.)*

**Q3. How are virtual functions implemented and what do they cost?**
> Typically with a vtable per class and a hidden vptr per object. A virtual call loads the vptr, indexes the vtable, and does an indirect call. The costs are an extra pointer per object — which hurts cache density — and per call an indirection the CPU can mispredict and the compiler usually can't inline. On a hot path that indirection is the expensive part, not the memory. If a class or method is `final`, the compiler can devirtualize and inline. And the base destructor must be virtual or `delete` through a base pointer is undefined. *(Follow-up: calling a virtual in a constructor dispatches to the type under construction.)*

**Q4. What is RAII and why does it make C++ exception-safe?**
> RAII ties a resource's lifetime to an object's lifetime: acquire in the constructor, release in the destructor. Because destructors run during stack unwinding when an exception propagates, any RAII-held resource — memory, a lock, a file handle — is released automatically without an explicit `finally`. That's why `lock_guard`, `unique_ptr`, and `fstream` are safe by construction. My rule is Rule of 0: let RAII members do the work and declare no special functions unless I'm writing the resource wrapper itself.

**Q5. Define undefined behavior and give three examples that bite in production.**
> UB is behavior the standard places no constraints on — and crucially the optimizer is allowed to assume it never occurs, so a latent UB bug can be invisible at `-O0` and explode at `-O2`. Three that bite: a dangling reference returned from a function (use-after-free), signed integer overflow (the compiler assumes it can't happen and may delete your overflow check), and a data race — two threads touching the same non-atomic object with at least one write. *(Follow-up: iterator invalidation after `push_back`.)*

**Q6. `std::vector` push_back is "amortized O(1)" — explain.**
> When it runs out of capacity it allocates a bigger buffer — geometrically, usually 1.5× or 2× — and moves the elements over. That copy is O(n), but it happens rarely enough that the *amortized* cost per push is constant: the total work over n pushes is proportional to n because the geometric series sums to a constant multiple of the final size. The catch is reallocation invalidates all iterators and pointers, so I `reserve()` when I know the size, especially on hot paths.

**Q7. What does `inline` actually do?**
> The guaranteed semantic is an ODR exemption — an `inline` function or variable can be defined in multiple translation units (e.g. in a header) and the linker collapses them to one definition. It is *not* a binding request to inline-expand for speed; that decision belongs to the optimizer regardless of the keyword. The C++17 `inline` variable is how you put a single global definition in a header.

**Q8. Walk me through what happens when one function calls another (the stack).**
> *(Use the section B-10 walkthrough: ABI arg passing in registers, `call` pushes return address, callee sets up the frame, return value in `rax`, frame torn down, `ret`. Tie it to stack overflow and to the cost of an indirect/virtual call.)*

**Q9. `std::map` vs `std::unordered_map` — which and why?**
> `map` is a balanced binary tree, O(log n), keeps keys ordered, node-based so it chases pointers. `unordered_map` is a hash table, average O(1) but worst-case O(n) with adversarial collisions, unordered, and rehashes as it grows. I pick `unordered_map` for raw lookup throughput, `map` when I need ordered iteration or range queries. For small fixed key sets a flat sorted vector often beats both because of locality.

**Q10. (Cross-over) Compare a C++ `std::vector<int>` to a Python `list` in memory.**
> A `std::vector<int>` is a contiguous block of raw ints — n×4 bytes, cache-friendly. A CPython `list` is a contiguous array of *pointers* to boxed `PyObject` integers, so you pay a pointer plus a heap-allocated object per element and lose locality. That's the core reason numerical Python work goes to NumPy, which stores unboxed contiguous data like the C++ vector. *(Good bridge if they pivot you from C++ to Python.)*

---

## "Gotcha" questions asked while you code

**Q: `map` vs `unordered_map`?** Ordered: red-black tree, O(log n) ops, keys sorted, stable iteration order. Unordered: hash table, O(1) average / O(n) worst (collisions), no ordering. Use unordered for pure speed; ordered when you need sorted traversal or range queries.

**Q: `vector` vs `list`?** Vector: contiguous, O(1) random access + cache-friendly, O(n) middle insert. List: doubly-linked, O(1) insert/erase given iterator, no random access, poor cache locality. **Default to vector** almost always.

**Q: When does `vector` reallocate, and what breaks?** When size exceeds capacity it allocates a bigger buffer (typically 2×) and moves elements — this **invalidates all iterators/pointers/references** into it. `reserve(n)` up front avoids repeated reallocs.

**Q: Pass by value vs reference vs const reference?** Const ref (`const string&`) for read-only large objects — no copy. Non-const ref to mutate caller's object. Value for small/primitive types or when you need a copy anyway.

**Q: What's `emplace_back` vs `push_back`?** `emplace_back` constructs in place from args (can avoid a temporary + move); `push_back` takes an already-constructed object. Minor perf win.

**Q: `nullptr` vs `NULL`?** `nullptr` is type-safe (`std::nullptr_t`); `NULL` is just `0` and can pick the wrong overload. Always `nullptr`.

**Q: What is a dangling pointer / use-after-free?** Pointer to memory that's been freed or gone out of scope. Reading it is UB. Common SRE-relevant bug class.

**Q: Stack vs heap memory?** Stack: automatic, fast, LIFO, freed on scope exit, limited size. Heap: `new`/`malloc`, manual or smart-pointer lifetime, larger, slower, fragmentable.

**Q: `auto`?** Compiler deduces the type. Great for verbose iterator types (`for (auto& [k,v] : m)`). Use `auto&` to avoid copies.

**Q: Why `lo + (hi-lo)/2`?** Avoids integer overflow that `(lo+hi)/2` can cause for large indices.

**Q: How do you read input efficiently?** `getline(cin, line)` for lines; `cin >> x` for tokens. `ios_base::sync_with_stdio(false); cin.tie(nullptr);` speeds up heavy I/O.

**Q: String building in a loop — efficiency?** `+=` on `std::string` amortizes; avoid building with `str = str + x` in a hot loop (can be quadratic). Consider `ostringstream` or `reserve`.

## Escalation chains — how a senior interviewer drills down

Each opener is followed by "why?" until you hit your edge. Practise surviving three follow-ups deep.

### Chain 1A — RAII → ownership → the under-the-hood

**Prompt:** "You use a lot of C++17/20. If I just say the word RAII — what is it and why does C++ lean on it so hard?"
> RAII ties a resource's lifetime to an object's lifetime: acquire in the constructor, release in the destructor. C++ leans on it because there's no GC and exceptions exist — when an exception unwinds the stack, destructors run automatically, so any RAII-held resource (memory, a lock, a file descriptor, a socket) is released without an explicit `finally`. `lock_guard`, `unique_ptr`, `fstream` are all RAII. My default is Rule of 0 — let RAII members manage everything and declare no special functions unless I'm writing the wrapper itself.

**Follow-up:** "Okay, `unique_ptr` vs `shared_ptr` — what do they actually cost me?"
> `unique_ptr` is sole ownership and is basically free: same size as a raw pointer, no refcount, move-only. `shared_ptr` carries an atomic reference count in a separate control block, so it's two pointers wide, copies pay atomic inc/dec, and naïvely you get two allocations — `make_shared` fuses object and control block into one. I default to `unique_ptr` and only reach for `shared_ptr` when ownership is genuinely shared, plus `weak_ptr` to break cycles.

**Follow-up:** "That atomic refcount — why does it *have* to be atomic? What breaks if it isn't?"
> Because copies/destructions of a `shared_ptr` can happen on different threads concurrently. If the count were a plain `int`, two threads decrementing at once is a data race — read-modify-write can interleave, you lose a decrement, and you either double-free or leak. Atomics make the inc/dec indivisible. Note it's only the *control block* count that's atomic — the managed object itself isn't protected, so concurrent writes to the pointee still need my own synchronization.

**Follow-up:** "Right. So if I'm in a single-threaded hot loop, is `shared_ptr` free then?"
> No — the atomic ops still emit the more expensive instructions (and inhibit some reordering) even with one thread, and you still pay the extra pointer and cache footprint. If it's single-threaded and I just need shared observation without ownership, I'd pass a raw pointer or reference, or use `unique_ptr` + borrowing. `shared_ptr` is for shared *ownership*, not "I'm passing a pointer around."

### Chain 1B — virtuals → vtable → cache

**Prompt:** "Virtual functions — how are they implemented, and what do they cost on a hot path?"
> Typically a vtable per class — an array of function pointers — and a hidden vptr per object. A virtual call loads the vptr, indexes the vtable, indirect-calls. Costs: +1 pointer per object, which hurts cache density, and per call an indirection the CPU can mispredict and the compiler usually can't inline. On a hot path that lost inlining is the real cost. If the class or method is `final`, the compiler can devirtualize and inline.

**Follow-up:** "Give me a gotcha with virtuals that's bitten you or that you watch for."
> Two: a polymorphic base class needs a **virtual destructor** or `delete base_ptr` is undefined and slices the derived part. And calling a virtual function **inside a constructor or destructor** dispatches to the type currently being constructed, not the most-derived type — so a virtual call in a base constructor won't reach the override you expect.

### Chain 1C — move semantics

**Prompt:** "Explain move semantics like I haven't touched C++ in a year, then tell me when it actually matters."
> A move transfers ownership of an object's internals instead of deep-copying — O(1) vs O(n) for something like a vector or string. The move ctor takes an rvalue reference, steals the pointer, leaves the source valid-but-empty. `std::move` is just a cast to `T&&` saying "safe to pillage." It matters whenever you'd otherwise copy a temporary about to die — returning big containers, `emplace_back`, handing off a `unique_ptr`. Though for returns the compiler often skips even the move via copy elision, which C++17 made mandatory for prvalues.

**Follow-up:** "If I declare a destructor on a class, what happens to its move operations?"
> Declaring a destructor (or any copy operation) **suppresses** the implicit generation of the move constructor and move assignment — so the class silently falls back to copying. That's the Rule of 5 trap: if I write one special member I should think about all five. I'd `= default` the moves explicitly if I want them.

## Cold rapid-fire (≤30s each)

**C++:** What's `std::move` actually do? · Why is `vector::push_back` amortized O(1)? · What does `inline` *really* mean (hint: not "go faster")? · `std::map` vs `unordered_map` and why? · What's undefined behavior and give one that changes between `-O0` and `-O2`? · What's the size of an empty class, and why not zero? · Reference vs pointer — when each? · What's `constexpr` vs `const`?
