# Worked Problems

Full solutions to practice problems that don't have a clean LeetCode match, plus a few that do but have an interview-specific twist.

# Google — custom / unmatched questions

### §5.4 — Movies + similarity graph: top-K similar 🔵
Graph: node = movie, weighted edge = similarity. Top-K similar to movie `m` = its highest-weight neighbors.
```cpp
// adj[m] = list of (similarity, neighborMovie)
vector<int> topKSimilar(int m, unordered_map<int, vector<pair<int,int>>>& adj, int K) {
    priority_queue<pair<int,int>, vector<pair<int,int>>, greater<>> minHeap; // size-K min-heap
    for (auto& [sim, nb] : adj[m]) {
        minHeap.push({sim, nb});
        if ((int)minHeap.size() > K) minHeap.pop();   // keep K largest
    }
    vector<int> res;
    while (!minHeap.empty()) { res.push_back(minHeap.top().second); minHeap.pop(); }
    reverse(res.begin(), res.end());                  // most similar first
    return res;
}
```
O(deg·log K) time, O(K) space — matches the asked complexity. **For many repeated queries:** pre-sort each node's neighbor list once (O(E log E)), then each query is just the first K → that's the "O(log K)/O(K)" optimization they pushed toward (binary-search/precomputed structure).

### §5.5 — Remove a node; max remaining edge weight; O(1) per query 🔵
**Key insight (state this — it's the whole solution):** an edge touches only **2** nodes. So removing node `v` can only delete the global-max edge if `v` is one of its **≤2 endpoints**. Therefore:
- Precompute edge weights sorted descending.
- For query "remove `v`": answer = the largest edge weight whose **both** endpoints ≠ `v`. Since each top edge is invalidated by at most 2 nodes, you only ever look a few edges down.
```cpp
// edges sorted desc as (weight, u, w). Precompute; query is ~O(1) amortized.
long long maxAfterRemoving(int v, const vector<tuple<long long,int,int>>& sortedDesc) {
    for (auto& [wt, a, b] : sortedDesc)               // walk from largest
        if (a != v && b != v) return wt;              // first edge not incident to v
    return -1;                                        // no edges remain
}
```
For true O(1): precompute, for each node, whether it's an endpoint of the top edge; the answer is `edges[0].weight` for **all but ≤2 nodes**, and the precomputed runner-up for those. Explain that and you've nailed it.

### §5.6 — Max-area rectangle from 2D points 🔵
Same technique as LeetCode #939 (which asks **min** area) — just track max. Hash all points into a set; for every **pair** of points that could be opposite corners (different x and y), check if the other two corners exist; if so, update area.
```cpp
long long maxRectArea(vector<pair<int,int>>& pts) {
    set<pair<int,int>> S(pts.begin(), pts.end());
    long long best = 0;
    for (int i = 0; i < (int)pts.size(); ++i)
        for (int j = i + 1; j < (int)pts.size(); ++j) {
            auto [x1,y1] = pts[i]; auto [x2,y2] = pts[j];
            if (x1 == x2 || y1 == y2) continue;       // need a true diagonal
            if (S.count({x1,y2}) && S.count({x2,y1}))  // other two corners present
                best = max(best, (long long)abs(x1-x2) * abs(y1-y2));
        }
    return best;
}
```
O(n²) time, O(n) space. Axis-aligned assumption — **clarify** whether rotated rectangles count (then it's harder, group by point-pair distance/midpoint).

### §5.7 — Ad server: getAd() highest score, decrement, no repeats, cooldown 🔵
Pattern = **max-heap + cooldown queue** (same shape as Task Scheduler #621 / Reorganize String #767).
- `insertAd(content, score)` → push into a max-heap keyed by score.
- `getAd()` → pop the max, but **hold back** ads still in cooldown in a side queue; serve the best eligible one, decrement its score, and put it into the cooldown queue with a "ready-after" marker; release ads from the cooldown queue back into the heap once their gap has passed.
```cpp
struct Ad { int score; string content; bool operator<(const Ad& o) const { return score < o.score; } };
class AdServer {
    priority_queue<Ad> heap;
    queue<pair<long long, Ad>> cooldown;          // (readyTick, ad)
    long long tick = 0, gap;
public:
    explicit AdServer(long long g = 1) : gap(g) {}
    void insertAd(const string& c, int s) { heap.push({s, c}); }
    string getAd() {
        ++tick;
        while (!cooldown.empty() && cooldown.front().first <= tick) {  // release ready ads
            heap.push(cooldown.front().second); cooldown.pop();
        }
        if (heap.empty()) return "";
        Ad a = heap.top(); heap.pop();
        if (a.score > 0) cooldown.push({tick + gap, {a.score - 1, a.content}}); // decremented, on cooldown
        return a.content;
    }
};
```
Each op ~O(log n). The cooldown queue guarantees no repeat within `gap` serves (covers "no consecutive same ad" as gap ≥ 1).

### §5.8 — Order book / auction 🔵
Two heaps: **bids** = max-heap by price, **asks** = min-heap by price. On a new order, match while `bestBid ≥ bestAsk` (cross), executing at the resting price and decrementing quantities; leftover rests in its book. Operations O(log n). (Add price→level maps + FIFO queues per level for full price-time priority — mention it as the productionization step.)

### §5.9 — Task scheduling: earliest finish + min CPUs (binary search) 🔵
*(Underspecified; here's the most likely interpretation + the technique they wanted.)*
- **Part 1 — earliest finish, M machines, T tasks each duration `d`:** if tasks are independent and identical, earliest finish = `ceil(T / M) * d`. (If there are release times/dependencies, it becomes a simulation with a min-heap of machine-free-times.)
- **Part 2 — min #CPUs to hit a target finish time `L`:** **binary search on the number of CPUs.** Feasibility check `can(c)`: with `c` CPUs, max tasks completable by `L` is `c * floor(L / d)`; feasible iff `c * floor(L/d) >= T`. Binary-search the smallest `c` that's feasible.
```cpp
bool canFinish(long long cpus, long long T, long long d, long long L) {
    return cpus * (L / d) >= T;                   // each CPU does floor(L/d) tasks by time L
}
int minCpus(long long T, long long d, long long L) {
    long long lo = 1, hi = T;                     // at most T CPUs needed
    while (lo < hi) {
        long long mid = lo + (hi - lo) / 2;
        if (canFinish(mid, T, d, L)) hi = mid;
        else lo = mid + 1;
    }
    return (int)lo;
}
```
**Talking point:** "binary search on the answer" is the reusable trick here (same family as Koko #875, Split Array #410) — name the pattern out loud.

# HRT — previously asked problems

> Problem statements are paraphrased; follow the link for the exact prompt.

### C1. Add Edges to Make Degrees of All Nodes Even — *Graphs, "Very likely"* ([LC 2508](https://leetcode.com/problems/add-edges-to-make-degrees-of-all-nodes-even/))

**Idea:** By the **handshake lemma**, the count of odd-degree nodes is always even. Each added edge flips the parity of exactly 2 nodes, so with **at most 2 edges** you can fix **at most 4** odd-degree nodes. Cases: 0 odd → done; 2 odd → connect them directly, or (if already adjacent) route through a common non-neighbor; 4 odd → check if you can split into two non-adjacent pairs; >4 → impossible.

```cpp
bool isPossible(int n, vector<vector<int>>& edges) {
    vector<unordered_set<int>> adj(n + 1);
    for (auto& e : edges) { adj[e[0]].insert(e[1]); adj[e[1]].insert(e[0]); }
    vector<int> odd;
    for (int i = 1; i <= n; ++i) if (adj[i].size() & 1) odd.push_back(i);

    if (odd.empty()) return true;
    if (odd.size() == 2) {
        int a = odd[0], b = odd[1];
        if (!adj[a].count(b)) return true;                       // direct edge
        for (int c = 1; c <= n; ++c)                              // route via common non-neighbor
            if (c != a && c != b && !adj[a].count(c) && !adj[b].count(c)) return true;
        return false;
    }
    if (odd.size() == 4) {
        int a=odd[0], b=odd[1], c=odd[2], d=odd[3];
        return (!adj[a].count(b) && !adj[c].count(d))
            || (!adj[a].count(c) && !adj[b].count(d))
            || (!adj[a].count(d) && !adj[b].count(c));
    }
    return false;
}
```
**Talking point:** open by stating the handshake-lemma observation — that *is* the insight they want to hear before code.

### C2. Longest Path With Different Adjacent Characters — *Trees/DFS, Hard* ([LC 2246](https://leetcode.com/problems/longest-path-with-different-adjacent-characters/))

**Idea:** tree-diameter DP. For each node, among children whose char differs, keep the **two longest** downward chains; the best path *through* this node = `top1 + top2 + 1`. Return `top1 + 1` upward.

```cpp
int longestPath(vector<int>& parent, string s) {
    int n = parent.size();
    vector<vector<int>> ch(n);
    for (int i = 1; i < n; ++i) ch[parent[i]].push_back(i);
    int ans = 1;
    function<int(int)> dfs = [&](int u) -> int {
        int b1 = 0, b2 = 0;
        for (int v : ch[u]) {
            int len = dfs(v);
            if (s[v] == s[u]) continue;          // adjacent chars must differ
            if (len > b1) { b2 = b1; b1 = len; }
            else if (len > b2) b2 = len;
        }
        ans = max(ans, b1 + b2 + 1);
        return b1 + 1;
    };
    dfs(0);
    return ans;
}
```
**Production note to mention:** for n up to 1e5 the recursion can blow the stack — say you'd convert to an **explicit-stack iterative DFS** in real code. HRT loves that you flagged it.

### C3. Detect Pattern of Length M Repeated K+ Times — *Sliding window, Easy* ([LC 1566](https://leetcode.com/problems/detect-pattern-of-length-m-and-at-least-k-repetitions/))

```cpp
bool containsPattern(vector<int>& a, int m, int k) {
    int run = 0;
    for (int i = 0; i + m < (int)a.size(); ++i) {
        run = (a[i] == a[i + m]) ? run + 1 : 0;     // chars repeating one period later
        if (run == (k - 1) * m) return true;
    }
    return false;
}
```

### C4. Remove Comments — *Strings / state machine, Medium* ([LC 722](https://leetcode.com/problems/remove-comments/))

**Idea:** scan char-by-char with an `inBlock` flag. (Note: standard C block comments **don't nest**, so a single flag suffices; if the prompt says nesting, use a depth counter.)

```cpp
vector<string> removeComments(vector<string>& source) {
    vector<string> res; string cur; bool inBlock = false;
    for (auto& line : source) {
        int i = 0, n = line.size();
        while (i < n) {
            if (!inBlock && i+1 < n && line[i]=='/' && line[i+1]=='/') break;          // line comment
            else if (!inBlock && i+1 < n && line[i]=='/' && line[i+1]=='*') { inBlock=true; i+=2; }
            else if ( inBlock && i+1 < n && line[i]=='*' && line[i+1]=='/') { inBlock=false; i+=2; }
            else if (inBlock) ++i;
            else cur += line[i++];
        }
        if (!inBlock && !cur.empty()) { res.push_back(cur); cur.clear(); }
    }
    return res;
}
```

### C5. Maximum Total Importance of Roads — *Greedy, Medium* ([LC 2285](https://leetcode.com/problems/maximum-total-importance-of-roads/))

**Idea:** each city contributes `value × degree`. To maximize, give the **largest values to the highest-degree cities** → sort by degree, assign `1..n`.

```cpp
long long maximumImportance(int n, vector<vector<int>>& roads) {
    vector<long long> deg(n, 0);
    for (auto& r : roads) { deg[r[0]]++; deg[r[1]]++; }
    sort(deg.begin(), deg.end());
    long long total = 0;
    for (int i = 0; i < n; ++i) total += deg[i] * (i + 1LL);   // long long: avoids overflow
    return total;
}
```

### C6. Small "filler" greedy/bit problems (Easy — bank these instantly)

```cpp
// Steps to reduce to zero (LC 1342): even -> /2, odd -> -1
int numberOfSteps(int num){ int s=0; while(num){ num=(num&1)?num-1:num>>1; ++s; } return s; }
// Closed form: for num>0, steps = (bit_length-1) + popcount(num)

// Maximum 69 number (LC 1323): flip first '6' to '9'
int maximum69Number(int num){ string t=to_string(num);
    for(char&c:t) if(c=='6'){c='9';break;} return stoi(t); }

// No-zero integers (LC 1317)
vector<int> getNoZeroIntegers(int n){
    auto ok=[](int x){ return to_string(x).find('0')==string::npos; };
    for(int a=1;a<n;++a) if(ok(a)&&ok(n-a)) return {a,n-a};
    return {};
}
```

### C7. Order matching (from HRT OA guide) — *Two-pointer*

> Match buys with sells whenever `buy_price ≥ sell_price`; return total executed volume.

```cpp
long long matchVolume(vector<array<int,3>>& orders) { // {price, qty, type: 0=buy,1=sell}
    vector<pair<int,int>> buys, sells;
    for (auto& o : orders) (o[2]==0 ? buys : sells).push_back({o[0], o[1]});
    sort(buys.rbegin(), buys.rend());      // highest buy first
    sort(sells.begin(), sells.end());      // lowest sell first
    long long vol = 0; size_t i = 0, j = 0;
    while (i < buys.size() && j < sells.size() && buys[i].first >= sells[j].first) {
        int t = min(buys[i].second, sells[j].second);
        vol += t; buys[i].second -= t; sells[j].second -= t;
        if (!buys[i].second) ++i;
        if (!sells[j].second) ++j;
    }
    return vol;
}
```

### C8. Consonant-shift string (HRT OA style)

> Every k-th consonant (in order) is shifted to the next consonant letter (wrapping z→…); case preserved, non-consonants untouched. e.g. `"CodeSignal", k=3 → "CodeTignam"`.

```cpp
string shiftEveryKthConsonant(string s, int k) {
    auto isVowel = [](char c){ c = tolower(c); return c=='a'||c=='e'||c=='i'||c=='o'||c=='u'; };
    auto nextCons = [&](char c){
        do { c = (c=='z')?'a' : (c=='Z')?'A' : c+1; } while (isVowel(c));
        return c;
    };
    int cnt = 0;
    for (char& c : s)
        if (isalpha((unsigned char)c) && !isVowel(c) && ++cnt == k) { c = nextCons(c); cnt = 0; }
    return s;
}
```
**Pitfall the walkthrough flags:** the z→b wrap and upper/lower case — exactly the careless-bug class HRT punishes.

# Citadel — "test a quicksort pivot" (test-design question)

They're testing your _test design_, not sorting. Partition = rearrange so elements ≤ pivot are left, > pivot right; return pivot's final index.

**Property to verify after partition around index p:** `all(arr[i] <= arr[p] for i < p)` and `all(arr[j] >= arr[p] for j > p)`, and the multiset of elements is unchanged (it's a permutation).

**Test cases (enumerate these — coverage is the signal):**

- Empty array; single element; two elements (both orders).
- Already sorted; reverse sorted (worst case for naive pivot).
- **All duplicates** (`[5,5,5,5]`) — classic breakage; naive Lomuto degrades to O(n²), Hoare needs care.
- Pivot is min; pivot is max (partition boundary lands at an end).
- Negative numbers; mixed sign; large values (overflow in mid = `lo + (hi-lo)/2`, not `(lo+hi)/2`).
- Random large array — fuzz: compare against `std::sort`; assert it's a permutation (same multiset) and the partition property holds.
- Stability isn't expected (quicksort isn't stable) — don't test for it.

```cpp
// property-style check
void check_partition(std::vector<int> a) {
    auto original = a;                      // save multiset
    int p = partition(a, 0, a.size()-1);
    for (int i = 0; i < p; ++i) assert(a[i] <= a[p]);
    for (int j = p+1; j < (int)a.size(); ++j) assert(a[j] >= a[p]);
    std::sort(a.begin(), a.end());
    std::sort(original.begin(), original.end());
    assert(a == original);                  // still a permutation
}
```

Mention: fuzz/property-based testing (randomized inputs + invariants) beats hand-picked cases; also test the caller's index math for off-by-one.
