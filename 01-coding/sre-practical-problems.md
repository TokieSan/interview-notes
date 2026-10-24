# SRE / Practical Coding Problems

The "systems-flavoured utility" style of coding question: parsing, streams, windows, aggregation. Solutions are interview-length C++. > Section IDs like `C1.6` or `C3.12` are cross-references inside the core systems notes: C1 = [Linux internals](../04-linux-os/linux-internals.md), C2 = [networking](../05-networking/networking-core.md), C3 = [distributed systems](../07-system-design/distributed-systems.md), C4 = [reliability & SLOs](../07-system-design/reliability-slos.md), C5 = [debugging method](../06-troubleshooting/debugging-method.md), C6 = [numbers](../07-system-design/capacity-numbers.md), C7 = [coding patterns](../01-coding/sre-practical-problems.md).

## Which patterns actually recur in SRE / infra coding rounds

The SRE-flavoured coding question is rarely a pure algorithm puzzle. It is a **systems-flavoured utility**: filesystem traversal, stream processing, log parsing, rate limiting, interval booking, scheduling. Practise those.

### C7.1 — The patterns that actually recur
| Pattern | Canonical problems | Why it shows up |
|---|---|---|
| **Stream with bounded window** | rolling average of last N, top-K over a stream, max in last 24h | Monitoring and metrics are streams |
| **Filesystem recursion** | folder size, `deleteDirectoryTree`, implement `find`, implement `du` | Directly models the job |
| **Intervals** | room booking, merge intervals, meeting rooms, maintenance windows | Scheduling and quota |
| **Rate limiting** | token bucket, sliding window log, sliding window counter | Every infra team builds one |
| **Heap / top-K** | k largest, median of a stream (two heaps), scheduler | Prioritisation |
| **Graph traversal** | dependency ordering, cycle detection, shortest path, connectivity | Service graphs, DAGs, deployment ordering |
| **Hash-map design** | LRU/LFU cache, consistent hashing ring, sorted hash table | Caching is the SRE's default hammer |
| **Parsing** | log line parsing, `tail -f` for huge files, dedup | Daily reality |
| **Concurrency** | producer/consumer with bounded queue, thread-safe cache, worker pool | Apple A1 |

### Things to be able to write from memory

1. **Token bucket rate limiter**, thread-safe, with lazy refill (no background timer). Then extend to distributed via Redis with an atomic script — and be ready to say why the naive `GET`/`INCR` version has a race.
2. **LRU cache** — hash map + intrinsic doubly-linked list, O(1) get/put. Then: how would you make it thread-safe without a global lock? (sharded locks / striping).
3. **`tail -n K` on a huge file** — seek from the end, read backwards in blocks, count newlines. Do *not* read the whole file. Discuss the partial-line boundary case, which is where the interviewer is going.
4. **Rolling average over last N of a stream** — ring buffer, O(1). Follow-up "ignore the k highest" → a multiset/two-heap structure, and be explicit about the extra cost.
5. **Recursive directory delete** given `GetDirectoryChildren()` and `Delete()` — children before parent, handle symlinks (do not follow), handle permission failures, decide whether it's fail-fast or best-effort. The *operational safety* discussion is what's being graded.
6. **Consistent hashing ring** with virtual nodes — sorted array of hashes plus binary search.
7. **Bounded producer/consumer queue** with condition variables. Then: what happens on shutdown, and how do you avoid a lost wakeup?

### Small-utility prompts

- Implement `tail -n K` for a file too large to fit in memory.
- Implement `find` — recurse directories, match a pattern.
- Write a Bash + cron setup that runs on the last Friday of every month. *(the trick: cron has no "last Friday", so run every Friday and guard with `[ $(date -d "+7 days" +\%m) != $(date +\%m) ]`)*
- Implement a shell's background-process and pipe facilities in C. *(`fork`/`pipe`/`dup2`/`execvp`/`waitpid`)*
- Script to find duplicate files. *(hash by size first, then content — say why)*

---

# Part 1 — Parsing / log-analysis problems with full C++ solutions

These are the bread and butter of practical (SRE-flavoured) coding rounds: read text, parse, count, aggregate, rank.

---

### Q1. Count requests per HTTP status code
**Prompt:** Given an access log, count how many times each status code appears, printed in descending order of count.
```
192.168.1.1 - - [10/Oct/2024:13:55:36] "GET /home HTTP/1.1" 200 512
192.168.1.2 - - [10/Oct/2024:13:55:37] "GET /x HTTP/1.1" 404 150
```
**Clarify:** Where's the status code in the line? (here: the token after the quoted request). Malformed lines?

```cpp
int main() {
    unordered_map<int,int> counts;
    string line;
    while (getline(cin, line)) {
        // status code = token right after the closing quote of "...request..."
        size_t q = line.rfind('"');
        if (q == string::npos) continue;            // skip malformed
        istringstream rest(line.substr(q + 1));     // " 200 512"
        int code;
        if (!(rest >> code)) continue;              // skip if no int
        counts[code]++;
    }
    // sort by count desc
    vector<pair<int,int>> v(counts.begin(), counts.end());
    sort(v.begin(), v.end(),
         [](auto& a, auto& b){ return a.second > b.second; });
    for (auto& [code, c] : v) cout << code << ": " << c << "\n";
}
```
**Complexity:** O(n + k log k), n = lines, k = distinct codes. **Edge cases:** empty file, lines with no quotes, non-numeric status.
**Talking point:** "`unordered_map` for O(1) average insert; I sort only the distinct keys, not all lines."

---

### Q2. Top-K most frequent items (paths / IPs)
**Prompt:** Print the top 3 most requested URL paths.
**Clarify:** Ties? (alphabetical tiebreak?) K given or fixed?

**Approach A — full sort** (simplest, fine for small k):
```cpp
int main() {
    unordered_map<string,int> freq;
    string method, path, proto, line;
    while (getline(cin, line)) {
        // assume "GET /path HTTP/1.1" is somewhere; simplest: split on quotes
        size_t a = line.find('"'), b = line.find('"', a + 1);
        if (a == string::npos || b == string::npos) continue;
        istringstream req(line.substr(a + 1, b - a - 1)); // GET /path HTTP/1.1
        req >> method >> path;
        if (path.empty()) continue;
        freq[path]++;
    }
    vector<pair<string,int>> v(freq.begin(), freq.end());
    sort(v.begin(), v.end(), [](auto& x, auto& y){
        if (x.second != y.second) return x.second > y.second; // count desc
        return x.first < y.first;                              // tie: name asc
    });
    int K = 3;
    for (int i = 0; i < (int)v.size() && i < K; ++i)
        cout << v[i].first << ": " << v[i].second << "\n";
}
```
**Approach B — min-heap of size K** (the "scales to huge input" answer — say this):
```cpp
// keep a min-heap of the K largest; O(n log k) time, O(k) space
using PCS = pair<int,string>; // (count, path)
priority_queue<PCS, vector<PCS>, greater<PCS>> heap; // min-heap
for (auto& [path, c] : freq) {
    heap.push({c, path});
    if ((int)heap.size() > K) heap.pop();  // drop smallest
}
vector<PCS> top;
while (!heap.empty()) { top.push_back(heap.top()); heap.pop(); }
reverse(top.begin(), top.end());           // now largest first
for (auto& [c, path] : top) cout << path << ": " << c << "\n";
```
**Talking point:** "Full sort is O(n log n). If K ≪ n and n is huge, a size-K min-heap is O(n log K) and O(K) memory." This is *the* SRE answer.

---

### Q3. Aggregate file sizes by extension
**Prompt:** Lines are `filepath,size_bytes,mtime`. Sum total bytes per file extension. Ignore files with no extension.
```cpp
int main() {
    unordered_map<string,long long> totals;
    string line;
    while (getline(cin, line)) {
        // split on commas
        vector<string> f; string tok; istringstream ss(line);
        while (getline(ss, tok, ',')) f.push_back(tok);
        if (f.size() < 2) continue;                       // malformed
        const string& path = f[0];
        long long size;
        try { size = stoll(f[1]); } catch (...) { continue; }  // bad number
        size_t dot = path.rfind('.');
        size_t slash = path.find_last_of("/\\");
        // ensure the dot is in the filename, not a directory, and not leading
        if (dot == string::npos) continue;
        if (slash != string::npos && dot < slash) continue; // ".d/file"
        if (dot == 0 || (slash != string::npos && dot == slash + 1)) continue; // dotfile
        totals[path.substr(dot + 1)] += size;
    }
    for (auto& [ext, t] : totals) cout << ext << ": " << t << " bytes\n";
}
```
**Edge cases (call these out — they love it):** dotfiles (`.bashrc`), dots in directory names (`/a.b/c`), trailing dot, multiple dots (`archive.tar.gz` → `gz`), missing size field, non-numeric size.
**Talking point:** "`long long` for the sum to avoid 32-bit overflow on large totals."

---

### Q4. Nested aggregation — bytes per group per day (CSV)
**Prompt:** CSV `date,process,host,bytes`. The exchange/service name is the prefix of `process` before the first `_` (e.g. `cme_trader_2` → `cme`). Print total bytes **per service per day**, sorted by date.
```cpp
int main() {
    // date -> (service -> bytes)
    map<string, map<string,long long>> agg;   // map (not unordered) => sorted output
    string line;
    getline(cin, line);  // skip header (confirm there is one!)
    while (getline(cin, line)) {
        vector<string> f; string tok; istringstream ss(line);
        while (getline(ss, tok, ',')) f.push_back(tok);
        if (f.size() < 4) continue;
        const string& date = f[0];
        const string& proc = f[1];
        long long bytes;
        try { bytes = stoll(f[3]); } catch (...) { continue; }
        string service = proc.substr(0, proc.find('_'));  // prefix before '_'
        agg[date][service] += bytes;
    }
    for (auto& [date, m] : agg)
        for (auto& [svc, b] : m)
            cout << date << "," << svc << "," << b << "\n";
}
```
**Talking point:** "I used `std::map` instead of `unordered_map` because I want output sorted by date/service for free — that's the ordered-vs-hash tradeoff: map is O(log n) per op + sorted, unordered is O(1) avg + unordered."

---

### Q5. Compute latency percentiles (p50 / p95 / p99)
**Prompt:** Each line is a request latency in ms. Print p50, p95, p99.
```cpp
double percentile(vector<long long>& v, double p) {  // v sorted ascending
    if (v.empty()) return 0;
    // nearest-rank method; mention there are several definitions
    size_t idx = (size_t)ceil(p / 100.0 * v.size()) - 1;
    idx = min(idx, v.size() - 1);
    return (double)v[idx];
}
int main() {
    vector<long long> lat; long long x;
    while (cin >> x) lat.push_back(x);
    sort(lat.begin(), lat.end());
    cout << "p50: " << percentile(lat, 50) << "\n";
    cout << "p95: " << percentile(lat, 95) << "\n";
    cout << "p99: " << percentile(lat, 99) << "\n";
}
```
**Talking point:** "There are multiple percentile definitions (nearest-rank vs linear interpolation) — I'll state which I'm using and confirm. At Google scale you'd use a streaming/approximate sketch (t-digest, HDR histogram) rather than holding every sample." That last sentence scores big.

---

### Q6. Merge overlapping intervals (uptime windows, time ranges)
**Prompt:** Given `[start,end]` pairs, merge overlaps. Classic and very SRE (maintenance windows, contiguous outages).
```cpp
vector<pair<int,int>> merge(vector<pair<int,int>> iv) {
    if (iv.empty()) return {};
    sort(iv.begin(), iv.end());                 // by start
    vector<pair<int,int>> out{iv[0]};
    for (size_t i = 1; i < iv.size(); ++i) {
        if (iv[i].first <= out.back().second)    // overlap (use < for "touching" not merged)
            out.back().second = max(out.back().second, iv[i].second);
        else
            out.push_back(iv[i]);
    }
    return out;
}
```
**Complexity:** O(n log n). **Clarify:** do touching intervals `[1,2],[2,3]` merge? (changes `<=` vs `<`).

---

### Q7. Sliding-window: detect error spike / count events in last T seconds
**Prompt:** Timestamps (sorted, seconds) of error events. Find the max number of errors within any 60-second window.
```cpp
int maxInWindow(vector<long long>& ts, long long W) {
    int best = 0, left = 0;
    for (int right = 0; right < (int)ts.size(); ++right) {
        while (ts[right] - ts[left] >= W) ++left;   // shrink: window is [ts[left], ts[right]]
        best = max(best, right - left + 1);
    }
    return best;
}
```
**Talking point:** "Two-pointer sliding window, O(n) since each pointer only moves forward. Assumes timestamps sorted — if not, sort first, O(n log n)."

---

### Q8. Deduplicate lines, preserve first-seen order
```cpp
int main() {
    unordered_set<string> seen;
    string line;
    while (getline(cin, line))
        if (seen.insert(line).second)   // .second == true if newly inserted
            cout << line << "\n";
}
```
**Talking point:** "`insert` returns `{iterator, bool}`; the bool tells me if it was new — one lookup, no double hashing."

---

### Q9. Implement `tail -n` (last N lines of a stream)
**Prompt:** Print the last N lines without knowing the file length up front.
```cpp
int main() {
    int N = 10;
    deque<string> buf;
    string line;
    while (getline(cin, line)) {
        buf.push_back(line);
        if ((int)buf.size() > N) buf.pop_front();  // ring buffer of size N
    }
    for (auto& l : buf) cout << l << "\n";
}
```
**Talking point:** "Bounded `deque` = O(N) memory regardless of file size. For `tail -f` you'd poll/inotify for appended bytes and never exit — different problem, happy to sketch it."

---

### Q10. Parse `key=value` structured log into fields
**Prompt:** Lines like `level=ERROR svc=auth latency=231 msg="bad token"`. Extract all `level=ERROR` lines' `svc` and average `latency`.
```cpp
unordered_map<string,string> parseKV(const string& line) {
    unordered_map<string,string> kv;
    istringstream ss(line);
    string token;
    while (ss >> token) {                  // splits on whitespace (won't handle quoted spaces — note this)
        size_t eq = token.find('=');
        if (eq == string::npos) continue;
        kv[token.substr(0, eq)] = token.substr(eq + 1);
    }
    return kv;
}
```
**Edge case to flag:** quoted values with spaces (`msg="bad token"`) break naive whitespace split — mention you'd need a small state machine or regex for production.

---

### Q11. Most frequent word / word count (the "FizzBuzz" of parsing)
```cpp
int main() {
    unordered_map<string,int> freq;
    string w;
    while (cin >> w) {
        // normalize: lowercase + strip punctuation
        string clean;
        for (char c : w) if (isalnum((unsigned char)c)) clean += tolower((unsigned char)c);
        if (!clean.empty()) freq[clean]++;
    }
    string best; int bestC = 0;
    for (auto& [word, c] : freq) if (c > bestC) { bestC = c; best = word; }
    cout << best << " " << bestC << "\n";
}
```
**Gotcha they test:** `tolower`/`isalnum` need `(unsigned char)` cast — passing a negative `char` is UB. Knowing this is a real C++ signal.

---

### Q12. Anagram grouping (hash by sorted key)
```cpp
int main() {
    vector<string> words = {/* read in */};
    unordered_map<string, vector<string>> groups;
    for (auto& w : words) {
        string key = w;
        sort(key.begin(), key.end());   // canonical form
        groups[key].push_back(w);
    }
    for (auto& [k, g] : groups) {
        for (auto& s : g) cout << s << " ";
        cout << "\n";
    }
}
```

# Part 2 — Stream problems (Google-style)

| #   | Question                                                                                                                                                                                                                            | Source                                   | Match                                                                                               |
| --- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------- | --------------------------------------------------------------------------------------------------- |
| 1.1 | Parse `events` like `error_1: 123`; return the **most frequent** event + all its timestamps. Follow-up: **realtime** — maintain a sliding queue of the last N events, evict oldest, return current most-frequent after each insert. | SWE-III ML R1, Jul 2025                  | 🧩 §5.1                                                                                             |
| 1.2 | Stream of `(item, timestamp)`; **dedup items arriving within a 60s window**. Follow-up: a modified dedup rule.                                                                                                                      | Google "unnoticed bugs" report, May 2026 | 🧩 §5.2 (related: 🔗 [Logger Rate Limiter #359](https://leetcode.com/problems/logger-rate-limiter)) |
| 1.3 | **Apply Substitutions:** text with `%key%` placeholders; each replacement value may itself contain placeholders; resolve fully; **detect cycles**.                                                                                  | L4 in-person, May 2026     | 🔗 [Apply Substitutions](https://leetcode.com/problems/apply-substitutions) + §5.3 (topo sort)      |
| 1.4 | Count requests per status code / top-K paths / aggregate by field (the classic log-analysis family).                                                                                                                                | Recurring SRE pattern                    | 🟢 see Part 1 above                                                        |

### §5.1 — Most-frequent event + timestamps; realtime sliding window 🟢
**Arguably the single best practice problem for an SRE coding round — it's practical, has a clean follow-up, and tests exactly the "data-structure choice + maintain-an-aggregate-under-mutation" skill SRE rounds love.**

**Part 1 — static:**
```cpp
// events like "error_1: 123". Return the most frequent key and its timestamps.
pair<string, vector<long long>> mostFrequent(const vector<string>& events) {
    unordered_map<string, vector<long long>> m;
    string bestKey;
    size_t best = 0;
    for (const auto& e : events) {
        size_t colon = e.find(':');
        if (colon == string::npos) continue;            // skip malformed
        string key = e.substr(0, colon);
        long long ts;
        try { ts = stoll(e.substr(colon + 1)); }         // stoll ignores leading spaces
        catch (...) { continue; }
        m[key].push_back(ts);
        if (m[key].size() > best) { best = m[key].size(); bestKey = key; }
    }
    return {bestKey, m[bestKey]};
}
```
Time O(n), space O(n). Edge cases: empty input, malformed line, ties (returns first to reach the max).

**Part 2 — realtime, bounded window of N (the follow-up):**
The hard part is keeping the current max as items are evicted. Maintain a frequency map **and** a "frequency → set of keys" index so max lookup stays cheap.
```cpp
class RealtimeMostFrequent {
    int N;
    deque<string> order;                          // last N keys, arrival order
    unordered_map<string,int> freq;               // key -> count
    map<int, unordered_set<string>> byFreq;       // count -> keys at that count
    void bump(const string& k, int delta) {
        int old = freq[k];
        if (old > 0) {                            // remove from old bucket
            byFreq[old].erase(k);
            if (byFreq[old].empty()) byFreq.erase(old);
        }
        int nw = old + delta;
        if (nw == 0) freq.erase(k);
        else { freq[k] = nw; byFreq[nw].insert(k); }
    }
public:
    explicit RealtimeMostFrequent(int n) : N(n) {}
    string add(const string& key) {               // insert one event, return current most-frequent
        order.push_back(key);
        bump(key, +1);
        if ((int)order.size() > N) {              // evict oldest
            string old = order.front(); order.pop_front();
            bump(old, -1);
        }
        return byFreq.empty() ? "" : *byFreq.rbegin()->second.begin(); // any key at max count
    }
};
```
Each `add` is O(log D) (D = distinct counts). **Talking point:** "The naive approach re-scans the map on every eviction → O(N) per insert. The frequency-bucket index makes max retrieval O(log D)." That observation is the whole point of the follow-up.

### §5.2 — Stream dedup within a 60s window 🟢
**Basic (one timestamp per item — like Logger Rate Limiter):**
```cpp
class WindowDedup {
    long long W;                                  // window seconds
    unordered_map<string,long long> last;         // item -> last accepted ts
public:
    explicit WindowDedup(long long w = 60) : W(w) {}
    bool accept(const string& item, long long ts) {
        auto it = last.find(item);
        if (it == last.end() || ts - it->second >= W) {
            last[item] = ts;                      // accept + record
            return true;
        }
        return false;                             // duplicate within window
    }
};
```
**If the rule needs "any occurrence in the trailing window" (multiple timestamps):** keep a per-item deque of timestamps, evict those older than `ts - W`, dedup if the deque is non-empty after eviction.
```cpp
unordered_map<string, deque<long long>> hist;
bool acceptWindowed(const string& item, long long ts) {
    auto& dq = hist[item];
    while (!dq.empty() && ts - dq.front() >= W) dq.pop_front();  // drop expired
    bool dup = !dq.empty();
    dq.push_back(ts);
    return !dup;
}
```
**Talking point:** clarify the exact dedup rule first ("since last accept" vs "any in trailing window") — they often *change the rule* as the follow-up, which is what the reporter hit.

### §5.3 — Apply Substitutions (`%key%`, nested, cycle detection) 🟢🔗
On LeetCode ([link](https://leetcode.com/problems/apply-substitutions)). Two angles:
- **Memoized DFS:** resolve a key by recursively resolving placeholders inside its value, caching resolved strings; detect cycles with a "currently-resolving" set (gray/black coloring) → throw/return error on revisit.
- **Topological sort:** build a dependency graph (key → keys it references); if there's a cycle, it's unresolvable; otherwise resolve in reverse topo order.
```cpp
// memoized DFS with cycle detection
unordered_map<string,string> val;                 // raw values, may contain %k%
unordered_map<string,string> resolved;            // cache
unordered_set<string> onStack;                    // cycle detection
string resolve(const string& key) {
    if (resolved.count(key)) return resolved[key];
    if (onStack.count(key)) throw runtime_error("cycle at " + key);
    if (!val.count(key)) throw runtime_error("unknown key " + key);
    onStack.insert(key);
    const string& s = val[key];
    string out;
    for (size_t i = 0; i < s.size(); ) {
        if (s[i] == '%') {
            size_t j = s.find('%', i + 1);
            if (j == string::npos) { out += s.substr(i); break; }
            out += resolve(s.substr(i + 1, j - i - 1)); // recurse on inner key
            i = j + 1;
        } else out += s[i++];
    }
    onStack.erase(key);
    return resolved[key] = out;
}
```
