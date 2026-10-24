# C++ Idioms, Debugging & Modern Features

# STL idioms the coding round grades

### 4. C++ idioms & STL — the fluency they're grading

#### Iterating
```cpp
for (auto& x : v) { ... }              // range-for, & to avoid copies
for (auto& [key, val] : myMap) { ... } // structured bindings (C++17)
for (auto it = m.begin(); it != m.end(); ) {
    if (drop(*it)) it = m.erase(it);   // erase returns next valid iterator
    else ++it;
}
```

#### The string toolkit
```cpp
string s = "a,b,c";
s.substr(2, 3);          // from index 2, length 3
s.find(',');             // first index of ',' or string::npos
s.rfind('.');            // last index
stoi(s); stoll(s); stod(s);  // parse to int / long long / double (throw on bad input)
to_string(42);
transform(s.begin(), s.end(), s.begin(), ::tolower);  // lowercase whole string
```
**Split a string on a delimiter (you WILL need this):**
```cpp
vector<string> split(const string& s, char delim) {
    vector<string> out; string tok; istringstream ss(s);
    while (getline(ss, tok, delim)) out.push_back(tok);
    return out;
}
```

#### `<algorithm>` greatest hits
```cpp
sort(v.begin(), v.end());
sort(v.begin(), v.end(), greater<int>());           // descending
sort(v.begin(), v.end(), [](auto&a, auto&b){ return a.second > b.second; }); // custom
auto it = find(v.begin(), v.end(), target);
int c = count(v.begin(), v.end(), x);
long long total = accumulate(v.begin(), v.end(), 0LL);  // 0LL => long long accumulator!
*max_element(v.begin(), v.end());
reverse(v.begin(), v.end());
v.erase(unique(v.begin(), v.end()), v.end());        // dedup *sorted* vector
bool any = binary_search(v.begin(), v.end(), x);     // sorted only
```

#### Lambdas (capture rules — they ask)
```cpp
int threshold = 5;
auto f = [threshold](int x){ return x > threshold; };   // capture by value (copy)
auto g = [&threshold](int x){ threshold += x; };        // capture by reference
auto h = [&](int x){ ... };                              // capture all by ref
```
**Pitfall:** capturing a local by reference and using the lambda after the local dies = dangling = UB.

#### References & const-correctness
```cpp
void readonly(const vector<int>& v);  // no copy, can't modify — default for big inputs
void mutate(vector<int>& v);          // modifies caller's vector
```

#### Memory & smart pointers (verbal-level is enough for this round)
- `new`/`delete` = manual heap. Every `new` needs exactly one `delete` or it **leaks**.
- `std::unique_ptr<T>` — sole owner, auto-frees, zero overhead. `std::shared_ptr<T>` — reference-counted, freed when last owner dies. **Prefer smart pointers; avoid raw `new`/`delete` in modern C++.**
- Know the bug classes: **memory leak** (lost allocation), **dangling pointer / use-after-free**, **double free**, **buffer overflow** (write past array end). These are SRE's daily reality — naming them confidently is a strong signal.

---

### 5. OOP quick reference (in case they ask design)
- **Encapsulation:** data + methods bundled, internals hidden behind an interface.
- **Inheritance:** `class Derived : public Base`. Reuse + specialize.
- **Polymorphism:** `virtual` methods dispatched at runtime via vtable; a `Base*` can point to a `Derived` and call the right override. **A base class with virtual methods needs a `virtual` destructor** (else deleting through a base pointer leaks the derived part).
- **Abstraction:** pure virtual (`= 0`) → abstract interface.
- **Composition over inheritance:** prefer "has-a" to deep "is-a" hierarchies.

---

# Heavy C++ debugging

Interviewers drop broken/UB code and ask "what's wrong, how would you find it." Have a systematic verbal checklist.

**Classic bug categories to name on sight:**

- **UB from uninitialized reads** — reading a local/member before assignment; value is indeterminate.
- **Dangling references/pointers** — returning reference to local; pointer to freed memory; iterator invalidation after `push_back`/`erase` (vector reallocation invalidates all iterators/pointers/refs; `erase` invalidates from the erased point on).
- **Use-after-move** — object used after `std::move`; moved-from is valid-but-unspecified.
- **Double free / ownership confusion** — two owners `delete` the same raw pointer.
- **Off-by-one / signed-unsigned mismatch** — `for (size_t i = n-1; i >= 0; --i)` never terminates (`size_t` wraps). Comparing `int` to `.size()` (unsigned) triggers surprising promotions.
- **Slicing** — assigning derived to base by value drops the derived part.
- **Missing virtual destructor** — `delete base_ptr` to a derived object → UB / partial destruction.
- **Data races** — shared mutable state without synchronization.
- **Lifetime of temporaries** — binding a `const&` extends life, but a raw pointer into a temporary dangles.

**Tooling to say out loud (SRE cares about this):**

- **Compiler first:** `-Wall -Wextra -Werror`, `-fsanitize=address` (ASan: heap/stack overflow, use-after-free), `-fsanitize=undefined` (UBSan), `-fsanitize=thread` (TSan: data races). ASan+UBSan combine; TSan is separate.
- **valgrind / memcheck** for leaks & invalid access (slower than ASan but no recompile of deps needed).
- **gdb:** breakpoints, `watch` (hardware watchpoints for "who mutated this"), `bt`, `info locals`, core-dump post-mortem (`gdb ./bin core`), `catch throw`.
- **Reproduce → isolate → bisect.** Minimal repro; `git bisect` for regressions; add asserts/logging; rubber-duck the invariant that's violated.
- Static analysis: clang-tidy, cppcheck.

**Debugging narration in the room:** state the invariant you _expect_, find where it breaks, form a hypothesis, test it. Don't randomly mutate code.

# OOP (C++ flavoured)

- **Four pillars:** encapsulation, abstraction, inheritance, polymorphism. Be ready to give a concrete example of each, not just definitions.
- **Virtual functions / vtables:** each polymorphic class has a vtable (array of function pointers); each object has a hidden vptr. Virtual call = deref vptr → index vtable → call. Cost: one extra indirection, defeats inlining. Non-virtual = resolved at compile time.
- **Virtual destructor rule:** any class meant to be deleted through a base pointer needs a `virtual ~Base()`. Otherwise UB.
- **Pure virtual** (`= 0`) → abstract class, can't instantiate. Interface = all-pure-virtual class.
- **Override / final:** use `override` (compiler checks you actually override); `final` stops further overriding/inheritance.
- **Rule of 0 / 3 / 5:**
    - _Rule of 3:_ if you define destructor, copy ctor, or copy assignment, you likely need all three.
    - _Rule of 5:_ add move ctor + move assignment for efficiency.
    - _Rule of 0:_ prefer designing so you need _none_ — let members (smart pointers, containers) manage resources.
- **Composition vs inheritance:** prefer composition ("has-a") unless a true "is-a" substitutable relationship (Liskov).
- **Static vs dynamic dispatch;** **overloading (compile-time) vs overriding (run-time).**
- **Diamond problem / virtual inheritance:** multiple inheritance can duplicate a base; `virtual` inheritance shares one base subobject. Know it exists; rarely asked to implement.
- **RAII** (the single most important C++ idiom): resource lifetime tied to object lifetime; acquire in ctor, release in dtor. Underpins locks, files, smart pointers.

# C++11 / 14 / 17 / 20 — and "what else?"

They ask open-endedly to gauge depth. Have layered answers ready — lead with what you actually use.

**C++11 (the big one):** move semantics & rvalue references, `auto`, range-for, lambdas, `nullptr`, `std::unique_ptr`/`shared_ptr`/`weak_ptr`, `std::thread`/`mutex`/`atomic`/`condition_variable`, the **memory model**, `constexpr`, variadic templates, `std::tuple`, `enum class`, uniform init `{}`, `override`/`final`, `= default`/`= delete`, `std::function`, `std::array`, `unordered_map/set`.

**C++14:** generic lambdas (`auto` params), lambda init-capture (`[x = std::move(y)]`), `std::make_unique` (11 had `make_shared` only), variable templates, relaxed `constexpr`, `[[deprecated]]`, binary literals, digit separators.

**C++17:** structured bindings (`auto [a,b] = pair`), `if constexpr`, `std::optional`, `std::variant`, `std::any`, `std::string_view`, `std::filesystem`, parallel STL algorithms (`std::execution::par`), fold expressions, guaranteed copy elision, inline variables, `[[nodiscard]]`/`[[maybe_unused]]`, `std::from_chars` (fast parsing).

**"What else?" ladder:** if pushed further, go to **C++20** (concepts, ranges, coroutines, `std::span`, `<=>` spaceship, modules, `std::jthread`, `constexpr` everything) → **C++23** (`std::expected`, `mdspan`, `print`). Then pivot to _how you use them for performance_: move to avoid copies, `string_view`/`span` to avoid allocations, `emplace` over `insert`, reserve capacity, `constexpr` to shift work to compile time.

# How `std::unordered_map` works

- **Hash table with separate chaining.** Buckets array; each bucket is a linked list of nodes. Key hashed → `hash(key) % bucket_count` → bucket → walk list comparing with `operator==`.
- **Complexity:** average **O(1)** insert/find/erase; **worst case O(n)** if everything collides (or adversarial hashing).
- **Load factor** = size / bucket_count. Exceeding `max_load_factor` (default 1.0) triggers **rehash** → more buckets, all elements re-placed. Rehash **invalidates iterators** but **NOT references/pointers to elements** (nodes are stable — standard mandates node-based).
- **Requirements:** `std::hash<Key>` (or custom hasher) + `operator==` (or custom `KeyEqual`).
- vs `std::map`: map is a balanced BST (red-black), **ordered**, **O(log n)**, no hashing. Use map when you need ordering/range queries; unordered_map when you need raw lookup speed.
- **Gotchas:** `operator[]` **default-inserts** if key absent (use `.find()`/`.contains()` to avoid); custom types need a hash specialization.

```cpp
#include <unordered_map>
#include <string>
#include <iostream>

int main() {
    std::unordered_map<std::string, int> counts;
    counts["apple"] = 3;
    counts["banana"] = 5;
    counts["apple"] += 1;          // now 4

    // safe lookup without inserting
    if (auto it = counts.find("cherry"); it != counts.end())
        std::cout << it->second;
    else
        std::cout << "not found\n";

    counts.emplace("date", 7);     // constructs in place, no-op if exists
    counts.reserve(100);           // avoid rehashes if size known

    for (const auto& [key, val] : counts)   // C++17 structured binding
        std::cout << key << ": " << val << "\n";
}
```

Custom key hashing:

```cpp
struct Point { int x, y; bool operator==(const Point&) const = default; };
struct PointHash {
    size_t operator()(const Point& p) const {
        return std::hash<int>{}(p.x) ^ (std::hash<int>{}(p.y) << 1);
    }
};
std::unordered_map<Point, int, PointHash> m;
```
