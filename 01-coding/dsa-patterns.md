# DSA Patterns & Templates

Big-O, data-structure choice, and the core patterns with a template each. For C++ STL idioms see [../02-cpp/](../02-cpp/README.md).

# Concepts

### 1. Big-O complexity — the language of the interview

Big-O describes how runtime/memory **grows** as input `n` grows. Drop constants and lower terms: `3n² + 5n + 9` → **O(n²)**.

| Class      | Name         | Example                         | Feels like            |
| ---------- | ------------ | ------------------------------- | --------------------- |
| O(1)       | constant     | hash lookup, array index        | instant               |
| O(log n)   | logarithmic  | binary search, balanced tree op | halving each step     |
| O(n)       | linear       | one scan of the input           | read everything once  |
| O(n log n) | linearithmic | good sorts, heap-based top-K    | sort-ish              |
| O(n²)      | quadratic    | nested loop over pairs          | nested for-loops      |
| O(2ⁿ)      | exponential  | naive subset enumeration        | brute-force recursion |

**How to analyze on the spot:** count nested loops over the input (each is a factor of `n`); a `sort` is `n log n`; a hash op is `O(1)` average. State **both time and space**. Space includes the data structures you allocate *and* recursion stack depth.

**The interview move:** start with the brute force, state its complexity, then improve. "Brute force is O(n²) — checking every pair. I can drop to O(n) with a hash map." This *out-loud progression* is half the grade.

---

### 2. Data structures — what to use and the C++ container that backs it

| Need                                  | Structure                | C++                   | Key ops & cost                                           |
| ------------------------------------- | ------------------------ | --------------------- | -------------------------------------------------------- |
| Indexed sequence, grow at end         | dynamic array            | `std::vector`         | index O(1), push_back amortized O(1), middle insert O(n) |
| Fast membership / dedup               | hash set                 | `std::unordered_set`  | insert/find O(1) avg                                     |
| Key→value, fast                       | hash map                 | `std::unordered_map`  | O(1) avg                                                 |
| Key→value, **sorted** / range queries | balanced BST (red-black) | `std::map`            | O(log n), ordered iteration                              |
| LIFO                                  | stack                    | `std::stack`          | push/pop/top O(1)                                        |
| FIFO                                  | queue                    | `std::queue`          | push/pop/front O(1)                                      |
| Both ends                             | double-ended queue       | `std::deque`          | push/pop both ends O(1)                                  |
| Always pull max/min                   | binary heap              | `std::priority_queue` | push/pop O(log n), top O(1)                              |
| Sorted unique keys                    | set                      | `std::set`            | O(log n)                                                 |

**The two decisions you'll defend most:**
- **`unordered_map` vs `map`:** hash (O(1), unordered) vs tree (O(log n), sorted). Pick unordered for speed; map when you need keys in order or range scans.
- **`vector` vs `deque` vs `list`:** vector is the default (cache-friendly, contiguous). Deque when you need both ends. List almost never (only O(1) splice/erase mid-sequence with an iterator, but terrible cache behavior).

#### Trees & graphs — the vocabulary
- **Binary tree:** each node ≤ 2 children. **BST:** left < node < right → search/insert O(log n) if balanced, O(n) if degenerate (a "linked list" tree).
- **Balanced trees** (red-black, AVL) guarantee O(log n) — `std::map`/`std::set` are red-black trees.
- **Heap:** complete binary tree with heap property (parent ≥ children for max-heap). Backs priority queues. Build O(n), push/pop O(log n).
- **Graph representations:** **adjacency list** (`vector<vector<int>>`, space O(V+E), good for sparse) vs **adjacency matrix** (`V×V`, O(1) edge check, O(V²) space, good for dense).
- **BFS** (queue) → shortest path in *unweighted* graphs, level-order. **DFS** (recursion/stack) → connectivity, cycle detection, topological sort, path existence.

---

### 3. Algorithms you must be able to discuss

**Sorting.** Know that the STL `sort` is **introsort** (quicksort + heapsort fallback + insertion sort for small), average O(n log n). `stable_sort` is O(n log n) and preserves equal-element order. Mergesort is stable O(n log n) but O(n) extra space. Quicksort is in-place average O(n log n) but O(n²) worst case on bad pivots. **Special-case answer (Google guide hints at this):** for small integer ranges, **counting sort / radix sort** is O(n) and beats comparison sorts; insertion sort wins on tiny or nearly-sorted inputs.

**Searching.** Linear O(n); binary search O(log n) but **requires sorted input**. Hashing gives O(1) average membership.

**Recursion.** A function calling itself on a smaller input + a base case. Each call adds a **stack frame** → recursion depth = O(depth) space, and too deep → stack overflow. Any recursion can be rewritten iteratively (explicit stack). Mention this for SRE: deep recursion on untrusted input is a reliability risk.

**Divide & conquer.** Split → solve halves → combine (mergesort, binary search). Often `T(n) = 2T(n/2) + O(n)` → O(n log n).

**Dynamic programming.** Overlapping subproblems + optimal substructure → cache results (memoization, top-down) or build a table (bottom-up). Turns exponential into polynomial. Classic: Fibonacci, coin change, longest common subsequence, edit distance.

**Greedy.** Take the locally best choice each step; works only when the problem has the "greedy-choice property" (e.g., interval scheduling: always pick the earliest-finishing interval). Faster than DP when valid, but not always correct — say "I'd verify greedy is actually optimal here."

**Two pointers / sliding window.** Convert an O(n²) pair/substring scan to O(n) by moving two indices that never backtrack.

---

### 6. Discrete math refresher (the guide says "some interviewers ask")
- **Permutations:** order matters, `n! / (n-k)!`.
- **Combinations (n-choose-k):** order doesn't, `C(n,k) = n! / (k!(n-k)!)`.
- **Probability basics:** independent events multiply; `P(A or B) = P(A)+P(B)−P(A and B)`; expected value = Σ(value × probability).
- **Powers of 2:** 2¹⁰≈10³ (1K), 2²⁰≈10⁶ (1M), 2³⁰≈10⁹ (1B), 2⁴⁰≈10¹² (1T). **Memorize these** — they power back-of-envelope sizing.

---

# Pattern templates (C++)

Know the *pattern*, not 200 problems. One representative solved problem per pattern.

### Pattern: Hash map for O(1) lookup
**Two Sum** — indices of two numbers summing to target.
```cpp
vector<int> twoSum(vector<int>& a, int target) {
    unordered_map<int,int> seen;            // value -> index
    for (int i = 0; i < (int)a.size(); ++i) {
        int need = target - a[i];
        if (seen.count(need)) return {seen[need], i};
        seen[a[i]] = i;
    }
    return {};
}
```

### Pattern: Two pointers
**Valid palindrome** (alphanumeric only, case-insensitive):
```cpp
bool isPalindrome(const string& s) {
    int i = 0, j = (int)s.size() - 1;
    while (i < j) {
        while (i < j && !isalnum((unsigned char)s[i])) ++i;
        while (i < j && !isalnum((unsigned char)s[j])) --j;
        if (tolower((unsigned char)s[i]) != tolower((unsigned char)s[j])) return false;
        ++i; --j;
    }
    return true;
}
```

### Pattern: Sliding window
**Longest substring without repeating characters:**
```cpp
int lengthOfLongest(const string& s) {
    unordered_map<char,int> last;  // char -> last index seen
    int best = 0, start = 0;
    for (int i = 0; i < (int)s.size(); ++i) {
        auto it = last.find(s[i]);
        if (it != last.end() && it->second >= start) start = it->second + 1;
        last[s[i]] = i;
        best = max(best, i - start + 1);
    }
    return best;
}
```

### Pattern: Stack
**Valid parentheses:**
```cpp
bool valid(const string& s) {
    stack<char> st;
    unordered_map<char,char> match{{')','('},{']','['},{'}','{'}};
    for (char c : s) {
        if (c=='('||c=='['||c=='{') st.push(c);
        else {
            if (st.empty() || st.top() != match[c]) return false;
            st.pop();
        }
    }
    return st.empty();
}
```

### Pattern: Linked list
**Reverse a singly linked list:**
```cpp
struct Node { int val; Node* next; };
Node* reverse(Node* head) {
    Node* prev = nullptr;
    while (head) {
        Node* nxt = head->next;
        head->next = prev;
        prev = head;
        head = nxt;
    }
    return prev;
}
```
**Detect cycle (Floyd's tortoise/hare):**
```cpp
bool hasCycle(Node* head) {
    Node *slow = head, *fast = head;
    while (fast && fast->next) {
        slow = slow->next;
        fast = fast->next->next;
        if (slow == fast) return true;
    }
    return false;
}
```

### Pattern: Tree traversal (BFS + DFS)
**Level-order (BFS):**
```cpp
struct TreeNode { int val; TreeNode *left, *right; };
vector<vector<int>> levelOrder(TreeNode* root) {
    vector<vector<int>> res;
    if (!root) return res;
    queue<TreeNode*> q; q.push(root);
    while (!q.empty()) {
        int n = q.size();
        vector<int> level;
        for (int i = 0; i < n; ++i) {
            TreeNode* cur = q.front(); q.pop();
            level.push_back(cur->val);
            if (cur->left) q.push(cur->left);
            if (cur->right) q.push(cur->right);
        }
        res.push_back(level);
    }
    return res;
}
```
**Max depth (DFS recursion):**
```cpp
int maxDepth(TreeNode* root) {
    if (!root) return 0;
    return 1 + max(maxDepth(root->left), maxDepth(root->right));
}
```

### Pattern: Graph BFS/DFS
**Number of islands (grid DFS):**
```cpp
void sink(vector<vector<char>>& g, int r, int c) {
    if (r < 0 || c < 0 || r >= (int)g.size() || c >= (int)g[0].size() || g[r][c] != '1') return;
    g[r][c] = '0';
    sink(g, r+1, c); sink(g, r-1, c); sink(g, r, c+1); sink(g, r, c-1);
}
int numIslands(vector<vector<char>>& g) {
    int count = 0;
    for (int r = 0; r < (int)g.size(); ++r)
        for (int c = 0; c < (int)g[0].size(); ++c)
            if (g[r][c] == '1') { ++count; sink(g, r, c); }
    return count;
}
```

### Pattern: Binary search
```cpp
int binarySearch(vector<int>& a, int target) {
    int lo = 0, hi = (int)a.size() - 1;
    while (lo <= hi) {
        int mid = lo + (hi - lo) / 2;     // avoid overflow vs (lo+hi)/2
        if (a[mid] == target) return mid;
        else if (a[mid] < target) lo = mid + 1;
        else hi = mid - 1;
    }
    return -1;
}
```

### Pattern: Heap / priority_queue
**Kth largest element:**
```cpp
int kthLargest(vector<int>& a, int k) {
    priority_queue<int, vector<int>, greater<int>> minHeap; // size-k min-heap
    for (int x : a) {
        minHeap.push(x);
        if ((int)minHeap.size() > k) minHeap.pop();
    }
    return minHeap.top();
}
```

### Pattern: Basic DP
**Climbing stairs / Fibonacci (bottom-up, O(1) space):**
```cpp
int climb(int n) {
    if (n <= 2) return n;
    int a = 1, b = 2;
    for (int i = 3; i <= n; ++i) { int c = a + b; a = b; b = c; }
    return b;
}
```
**Coin change (min coins):**
```cpp
int coinChange(vector<int>& coins, int amount) {
    vector<int> dp(amount + 1, amount + 1);
    dp[0] = 0;
    for (int a = 1; a <= amount; ++a)
        for (int c : coins)
            if (c <= a) dp[a] = min(dp[a], dp[a - c] + 1);
    return dp[amount] > amount ? -1 : dp[amount];
}
```

### More templates (DSU, grid DP, BFS with portals, binary search on the answer)

**Union-Find (DSU)** — for contact-tracing / connectivity / grouping:
```cpp
struct DSU {
    vector<int> p, r;
    DSU(int n): p(n), r(n,0) { iota(p.begin(), p.end(), 0); }
    int find(int x){ return p[x]==x ? x : p[x]=find(p[x]); }     // path compression
    bool unite(int a,int b){ a=find(a); b=find(b); if(a==b) return false;
        if(r[a]<r[b]) swap(a,b); p[b]=a; if(r[a]==r[b]) ++r[a]; return true; } // union by rank
};
```

**Grid path DP** — max score top-left → bottom-right (down/right):
```cpp
int maxPath(vector<vector<int>>& g){
    int m=g.size(), n=g[0].size();
    vector<vector<long long>> dp(m, vector<long long>(n, LLONG_MIN));
    for(int i=0;i<m;++i) for(int j=0;j<n;++j){
        long long best = (i==0&&j==0)?0:LLONG_MIN;
        if(i>0) best=max(best,dp[i-1][j]);
        if(j>0) best=max(best,dp[i][j-1]);
        dp[i][j]=g[i][j]+best;
    }
    return dp[m-1][n-1];
}
```

**BFS on a grid w/ teleport portals** (matrix-maze variant): standard BFS over cells; when you step on a portal cell, also enqueue its paired exit (cost +0 or +1 per the prompt). Track `visited`; mind that portals can create shortcuts — BFS still gives shortest unit-cost paths.

**Binary search on the answer** ("time intervals without obstacles", feasibility problems):
```cpp
// find smallest x in [lo,hi] with feasible(x) true (monotonic predicate)
int lo = LO, hi = HI;
while (lo < hi) { int mid = lo + (hi-lo)/2; if (feasible(mid)) hi = mid; else lo = mid+1; }
// lo is the answer
```

### Python equivalents (in case you get steered)

```python
# Detect pattern (LC 1566)
def contains_pattern(a, m, k):
    run = 0
    for i in range(len(a) - m):
        run = run + 1 if a[i] == a[i + m] else 0
        if run == (k - 1) * m: return True
    return False

# DSU
class DSU:
    def __init__(self, n): self.p = list(range(n)); self.r = [0]*n
    def find(self, x):
        while self.p[x] != x:
            self.p[x] = self.p[self.p[x]]; x = self.p[x]   # path halving (iterative: no stack risk)
        return x
    def unite(self, a, b):
        a, b = self.find(a), self.find(b)
        if a == b: return False
        if self.r[a] < self.r[b]: a, b = b, a
        self.p[b] = a
        if self.r[a] == self.r[b]: self.r[a] += 1
        return True

# Max importance of roads (LC 2285)
def maximum_importance(n, roads):
    deg = [0]*n
    for u, v in roads: deg[u]+=1; deg[v]+=1
    deg.sort()
    return sum(d*(i+1) for i, d in enumerate(deg))
```
Note Python recursion depth defaults to ~1000 — for deep tree DFS use an explicit stack or `sys.setrecursionlimit`, and mention it.

# Rapid-fire direct Q&A (drill these)

- **Reverse a string in place?** Two pointers swap from both ends, or `reverse(s.begin(), s.end())`.
- **Check if two strings are anagrams?** Sort both and compare, or count chars in a `array<int,256>`.
- **Find duplicates in an array?** `unordered_set`: if `insert(...).second == false`, it's a dup.
- **First non-repeating character?** Count pass, then scan for first with count 1.
- **Merge two sorted arrays?** Two pointers, O(n+m).
- **FizzBuzz?** Loop 1..n; `%15`→FizzBuzz, `%3`→Fizz, `%5`→Buzz, else number.
- **Count set bits in an int?** `__builtin_popcount(x)`, or `while(x){ c += x&1; x>>=1; }`.
- **Reverse an integer (watch overflow)?** Pop digits `x%10`, push into result, check overflow before `*10`.
- **Is a number prime?** Trial-divide up to `sqrt(n)`.
- **GCD?** `__gcd(a,b)` or Euclid: `while(b){ a%=b; swap(a,b); }`.
- **Detect balanced brackets?** Stack (see Part 2).
- **Find max/min/sum of vector?** `*max_element`, `*min_element`, `accumulate(v.begin(),v.end(),0LL)`.
- **Sort by custom key?** `sort(v.begin(), v.end(), [](auto&a, auto&b){ return ...; })`.
- **Remove element while iterating a map?** `it = m.erase(it);` — `erase` returns next valid iterator.
