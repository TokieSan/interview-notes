# Problem Lists

Curated LeetCode lists, grouped by pattern, for Google, HRT and Citadel Securities.

## How to use these lists

Prioritise: practical/parsing problems first (see [sre-practical-problems.md](sre-practical-problems.md)) → the high-frequency core list → company-specific breadth. Track your own progress in a copy; the lists below are deliberately un-ticked.

## Topics worth a dedicated pass

- SCCs (Tarjan’s algorithm, Kosaraju’s algorithm) https://cp-algorithms.com/graph/strongly-connected-components.html
- Euler paths/circuits https://cp-algorithms.com/graph/euler_path.html 
	- I think this is [Hierholzer's algorithm](https://www.wikiwand.com/en/Eulerian_path#/Hierholzer.27s_algorithm). Before applying it, you need to check that a Euler circuit actually exists for the given graph. Here is another good [resource](http://iampandiyan.blogspot.in/2013/10/c-program-to-find-euler-path-or-euler.html). Take a look at this [problem](https://codeforces.com/contest/508/problem/D) which asks you to construct a Eulerian circuit. My code for the same can be found [here](https://codeforces.com/contest/508/submission/12133138).
- topological sorting: https://cp-algorithms.com/graph/topological-sort.html
- DSUs
- Flood fill/ graph algos
- next greater, next smaller
- knapsack dp (dp atcoder is a good full review)
- tries
- streaming ideas (like problems with streaming for big files rather than bounded input)

# Google — practice problems, by pattern

Most problems below are typical of **SWE L3/L4 algorithmic loops**, not the SRE-SE "Programming & Code Evaluation" track specifically. That matters for calibration:
- The SRE-SE coding round is officially **practical coding** ("simple algorithm/data-structure implementation… clean, working, efficient"). The **parsing/stream problems in Tier 1** are the closest match to what SRE candidates actually see.
- But Google's SRE coding **can lean fully algorithmic**, and the question bank overlaps. So Tiers 2–3 are worth drilling — they build the pattern-recognition the round rewards.
- **Verdict:** prioritize Tier 1 (your flavor) → then the highest-frequency Tier-3 patterns → then breadth in Tier 2.

### Trees / DFS
- **BST, longest path where all nodes share the same value** + DFS follow-up 🔗 [Longest Univalue Path #687](https://leetcode.com/problems/longest-univalue-path)
- **Binary Tree Maximum Path Sum** — recurring. 🔗 [#124](https://leetcode.com/problems/binary-tree-maximum-path-sum)
- **Lowest Common Ancestor** — recurring. 🔗 [#236](https://leetcode.com/problems/lowest-common-ancestor-of-a-binary-tree)
- **Diameter of Binary Tree** — recurring. 🔗 [#543](https://leetcode.com/problems/diameter-of-binary-tree)

### Graphs (heavy at Google — drill hard)
- **Topological sort** (course-schedule style) 🔗 [Course Schedule #207](https://leetcode.com/problems/course-schedule), [Course Schedule II #210](https://leetcode.com/problems/course-schedule-ii)
- **Dijkstra / shortest path** 🔗 [Network Delay Time #743](https://leetcode.com/problems/network-delay-time)
- **Number of Islands** + variants — recurring. 🔗 [#200](https://leetcode.com/problems/number-of-islands)
- **"Find all lakes in a grid; twist: land inside the lake"** (enclosed water) 🔗 [Number of Closed Islands #1254](https://leetcode.com/problems/number-of-closed-islands) (+ [Flood Fill #733](https://leetcode.com/problems/flood-fill))
- **Number of Provinces / connected components** — recurring. 🔗 [#547](https://leetcode.com/problems/number-of-provinces)
- **Remove a node from a weighted graph; return max remaining edge weight; O(1) per query** 🧩 §5.5
- **Movies + similarity graph; top-K similar movies; optimize toward O(log K)** 🧩 §5.4

### Intervals / greedy / sweep
- **Greedy + interval** problem 🔗 [Merge Intervals #56](https://leetcode.com/problems/merge-intervals)
- **Overlapping intervals, line-sweep + priority queue** 🔗 [Meeting Rooms II #253](https://leetcode.com/problems/meeting-rooms-ii)

### Arrays / strings / two-pointer / sliding window
- **Kadane's (max subarray)** + hashmap follow-up 🔗 [Maximum Subarray #53](https://leetcode.com/problems/maximum-subarray)
- **Subarray Sum Equals K** — recurring. 🔗 [#560](https://leetcode.com/problems/subarray-sum-equals-k)
- **Array divisible into groups of 5 consecutive increasing numbers (bool)**, handle duplicates, use each element once 🔗 [Divide Array in Sets of K Consecutive Numbers #1296](https://leetcode.com/problems/divide-array-in-sets-of-k-consecutive-numbers) (≈ [Hand of Straights #846](https://leetcode.com/problems/hand-of-straights))
- **LC-Hard sliding window** (with precomputation follow-up) 🔗 [Sliding Window Maximum #239](https://leetcode.com/problems/sliding-window-maximum), [Minimum Window Substring #76](https://leetcode.com/problems/minimum-window-substring)
- **Number of Visible People in a Queue** (monotonic stack) + custom visibility twist 🔗 [#1944](https://leetcode.com/problems/number-of-visible-people-in-a-queue)

### Geometry
- **Max-area rectangle from a set of 2D points** (efficient) 🔗 [Minimum Area Rectangle #939](https://leetcode.com/problems/minimum-area-rectangle) — *same hash-the-point-set technique; track **max** area instead of min.* (§5.6 note)

### Design / implementation-heavy
- **LRU / LFU cache** variants (e.g. "evict only even scores", "score++ on access") 🔗 [LRU #146](https://leetcode.com/problems/lru-cache), [LFU #460](https://leetcode.com/problems/lfu-cache)
- **insertAd(content, score) / getAd():** return highest-score ad, decrement after serving, no consecutive repeats, cooldown follow-up, ~O(1) 🧩 §5.7 (pattern ≈ [Task Scheduler #621](https://leetcode.com/problems/task-scheduler) / [Reorganize String #767](https://leetcode.com/problems/reorganize-string))
- **Order book / auction:** insert bids, execute lowest ask 🧩 §5.8

### Scheduling
- **Tasks (equal duration) across M machines → earliest finish; then min #CPUs for that finish (binary search)** 🧩 §5.9

### Math / DP / misc
- **0/1 knapsack / subset-sum DP** (memo vs tabulation) 🔗 [Partition Equal Subset Sum #416](https://leetcode.com/problems/partition-equal-subset-sum)
- **Graph + DP** medium-hard combo — *generic summary*. 🔗 [Word Break #139](https://leetcode.com/problems/word-break), [Longest Increasing Subsequence #300](https://leetcode.com/problems/longest-increasing-subsequence)

# Google — high-frequency core list

**Arrays / hashing**
[Two Sum #1](https://leetcode.com/problems/two-sum) · [Best Time to Buy/Sell Stock #121](https://leetcode.com/problems/best-time-to-buy-and-sell-stock) · [Longest Consecutive Sequence #128](https://leetcode.com/problems/longest-consecutive-sequence) · [Subarray Sum Equals K #560](https://leetcode.com/problems/subarray-sum-equals-k) · [Majority Element #169](https://leetcode.com/problems/majority-element) · [Group Anagrams #49](https://leetcode.com/problems/group-anagrams) · [Contains Duplicate #217](https://leetcode.com/problems/contains-duplicate)

**Two pointers / sliding window**
[3Sum #15](https://leetcode.com/problems/3sum) · [Container With Most Water #11](https://leetcode.com/problems/container-with-most-water) · [Trapping Rain Water #42](https://leetcode.com/problems/trapping-rain-water) · [Longest Substring Without Repeating Characters #3](https://leetcode.com/problems/longest-substring-without-repeating-characters) · [Longest Repeating Character Replacement #424](https://leetcode.com/problems/longest-repeating-character-replacement) · [Minimum Window Substring #76](https://leetcode.com/problems/minimum-window-substring) · [Sliding Window Maximum #239](https://leetcode.com/problems/sliding-window-maximum) · [Valid Palindrome #125](https://leetcode.com/problems/valid-palindrome)

**Strings**
[Longest Common Prefix #14](https://leetcode.com/problems/longest-common-prefix) · [Roman to Integer #13](https://leetcode.com/problems/roman-to-integer) · [Longest Palindromic Substring #5](https://leetcode.com/problems/longest-palindromic-substring) · [Decode String #394](https://leetcode.com/problems/decode-string) · [String to Integer (atoi) #8](https://leetcode.com/problems/string-to-integer-atoi) · [Zigzag Conversion #6](https://leetcode.com/problems/zigzag-conversion)

**Stack / monotonic**
[Valid Parentheses #20](https://leetcode.com/problems/valid-parentheses) · [Min Stack #155](https://leetcode.com/problems/min-stack) · [Largest Rectangle in Histogram #84](https://leetcode.com/problems/largest-rectangle-in-histogram) · [Maximal Rectangle #85](https://leetcode.com/problems/maximal-rectangle) · [Daily Temperatures #739](https://leetcode.com/problems/daily-temperatures) · [Basic Calculator #224](https://leetcode.com/problems/basic-calculator)

**Linked list**
[Add Two Numbers #2](https://leetcode.com/problems/add-two-numbers) · [Reverse Linked List #206](https://leetcode.com/problems/reverse-linked-list) · [Merge Two Sorted Lists #21](https://leetcode.com/problems/merge-two-sorted-lists) · [Merge k Sorted Lists #23](https://leetcode.com/problems/merge-k-sorted-lists) · [Reorder List #143](https://leetcode.com/problems/reorder-list) · [Reverse Nodes in k-Group #25](https://leetcode.com/problems/reverse-nodes-in-k-group)

**Trees**
[Diameter of Binary Tree #543](https://leetcode.com/problems/diameter-of-binary-tree) · [Lowest Common Ancestor #236](https://leetcode.com/problems/lowest-common-ancestor-of-a-binary-tree) · [Binary Tree Maximum Path Sum #124](https://leetcode.com/problems/binary-tree-maximum-path-sum) · [Same Tree #100](https://leetcode.com/problems/same-tree) · [Longest Univalue Path #687](https://leetcode.com/problems/longest-univalue-path)

**Graphs**
[Number of Islands #200](https://leetcode.com/problems/number-of-islands) · [Course Schedule #207](https://leetcode.com/problems/course-schedule) · [Course Schedule II #210](https://leetcode.com/problems/course-schedule-ii) · [Number of Provinces #547](https://leetcode.com/problems/number-of-provinces) · [Pacific Atlantic Water Flow #417](https://leetcode.com/problems/pacific-atlantic-water-flow) · [Network Delay Time #743](https://leetcode.com/problems/network-delay-time) · [Apply Substitutions](https://leetcode.com/problems/apply-substitutions)

**Binary search**
[Search in Rotated Sorted Array #33](https://leetcode.com/problems/search-in-rotated-sorted-array) · [Find First and Last Position #34](https://leetcode.com/problems/find-first-and-last-position-of-element-in-sorted-array) · [Find Peak Element #162](https://leetcode.com/problems/find-peak-element) · [Koko Eating Bananas #875](https://leetcode.com/problems/koko-eating-bananas) · [Median of Two Sorted Arrays #4](https://leetcode.com/problems/median-of-two-sorted-arrays) · [Split Array Largest Sum #410](https://leetcode.com/problems/split-array-largest-sum)

**Intervals**
[Merge Intervals #56](https://leetcode.com/problems/merge-intervals) · [Meeting Rooms II #253](https://leetcode.com/problems/meeting-rooms-ii)

**Heap / design**
[Kth Largest Element in an Array #215](https://leetcode.com/problems/kth-largest-element-in-an-array) · [Top K Frequent Words #692](https://leetcode.com/problems/top-k-frequent-words) · [Find Median from Data Stream #295](https://leetcode.com/problems/find-median-from-data-stream) · [LRU Cache #146](https://leetcode.com/problems/lru-cache) · [LFU Cache #460](https://leetcode.com/problems/lfu-cache) · [Logger Rate Limiter #359](https://leetcode.com/problems/logger-rate-limiter) · [Insert Delete GetRandom O(1) #380](https://leetcode.com/problems/insert-delete-getrandom-o1)

**Matrix / simulation**
[Rotate Image #48](https://leetcode.com/problems/rotate-image) · [Spiral Matrix #54](https://leetcode.com/problems/spiral-matrix) · [Set Matrix Zeroes #73](https://leetcode.com/problems/set-matrix-zeroes) · [Valid Sudoku #36](https://leetcode.com/problems/valid-sudoku)

**DP**
[Climbing Stairs #70](https://leetcode.com/problems/climbing-stairs) · [House Robber #198](https://leetcode.com/problems/house-robber) · [Coin Change #322](https://leetcode.com/problems/coin-change) · [Word Break #139](https://leetcode.com/problems/word-break) · [Longest Increasing Subsequence #300](https://leetcode.com/problems/longest-increasing-subsequence) · [Maximal Square #221](https://leetcode.com/problems/maximal-square) · [Edit Distance #72](https://leetcode.com/problems/edit-distance) · [Unique Paths #62](https://leetcode.com/problems/unique-paths) · [Jump Game #55](https://leetcode.com/problems/jump-game)

> The phone-screen report flagged these as the **closest to actual asked**: LRU Cache, Sliding Window Maximum, Median of Two Sorted Arrays, Russian Doll Envelopes.

# HRT — question distribution

Ranked by HRT frequency (from the jointaro question bank + reports):

| Theme | Likelihood | Representative problems |
|---|---|---|
| **Graphs** | **Very likely** | Add Edges to Make Degrees Even; contact-tracing; packet routing; connected components |
| **Trees / DFS** | Likely | Longest Path w/ Different Adjacent Chars; balanced-BST traverse & rebuild; K-d tree internals |
| **Grid / DP** | Likely | Max Score from Grid Operations; max path sum; 2048 simulator |
| **Implementation / simulation** | Likely (live rounds) | Tic-tac-toe / "Lines & Squares"; 2048; matrix maze w/ teleport portals |
| **Arrays / sliding window / two-pointer** | Likely | Detect Pattern length M repeated K; order matching (buy/sell) |
| **Strings** | Medium | Remove Comments; consonant-shift; substring/fragment matching |
| **Binary search** | Medium | "Time intervals without obstacles"; binary-search-on-answer |
| **Union-Find** | Medium | Connectivity / grouping |
| **Greedy** | Medium | Max Total Importance of Roads; Max 69 Number; No-Zero integers |
| **Bit manipulation / number systems / floating point** | Lower | Steps to Reduce to Zero; binary/hex; FP representation |

# Citadel — patterns and practice problems

**Coding questions:**

- **Best Time to Buy/Sell Stock** — optimize the O(n²) to O(n). A classic screen question; know it cold.
- Shortest path in a graph **with a custom/extra constraint** (they modify the standard problem — don't just recite Dijkstra).
- **Running top-K over a data stream** (heap + hash map). (see §17)
- **Thread-safe circular (ring) buffer** — common implementation question. (see §16)
- Explain **trade-offs between sorting algorithms in a low-latency context.**
- Autocomplete / prefix search that must respond **sub-millisecond** (trie + design).

Beyond your grind, these patterns show up most for Citadel/Citadel Securities (skew: **graphs/trees, arrays, heaps, intervals**; often "optimize O(n²)→O(n)"):

- **Best Time to Buy/Sell Stock** (I, II, III, with cooldown/fee) — know the O(n) one-pass DP cold.
- **Arrays / two-pointer / sliding window:** longest substring w/o repeats, max subarray (Kadane), min window substring, trapping rain water, product except self.
- **Hash-map counting:** two-sum, group anagrams, subarray sum equals K (prefix sums + map), top-K frequent.
- **Heaps / priority_queue:** merge k sorted lists, kth largest, median from data stream, task scheduler, top-K.
- **Intervals:** merge intervals, insert interval, meeting rooms I/II, non-overlapping intervals.
- **Graphs/trees (their favorite):** BFS/DFS, number of islands, course schedule (topo sort / cycle detect), clone graph, word ladder, LCA, serialize/deserialize tree, level-order, validate BST, Dijkstra/union-find basics.
- **Binary search** (and on answer space): search rotated array, median of two sorted arrays, koko eating bananas / capacity-to-ship style.
- **Stacks:** valid parentheses, next greater element (monotonic stack), min stack, largest rectangle in histogram.
- **DP:** coin change, LIS, edit distance, house robber, word break, unique paths.
- **Design:** LRU cache (hashmap + doubly linked list), rate limiter, hit counter — bridges into their systems questions.
- **Low-latency spin:** be ready to discuss cache locality, avoiding allocations, and complexity of _your_ solution — they optimize the correct answer further.

**In-room process (they grade this):** restate problem + assumptions → brute force + complexity → optimize with trade-offs → edge cases _before_ coding → clean code → test/walk through → note complexity. Think out loud; ask clarifying questions; if stuck, say so and reason forward.

### Citadel-tagged problem list (from frequency lists)

Also worth practising: implement a thread pool for sending messages, a `shared_ptr` class, test a quicksort pivot, and an order book (see [github.com/olibre/Cpp14-OrderBook](https://github.com/olibre/Cpp14-OrderBook)).

- https://leetcode.com/problems/number-of-orders-in-the-backlog/description/ XX 

- https://leetcode.com/problems/sliding-window-median/description/
- https://leetcode.com/problems/recover-binary-search-tree/description/
- Design a random generator given a list of probabilities https://leetcode.com/problems/random-pick-with-weight/description/
- https://leetcode.com/problems/random-pick-index/description/
- https://leetcode.com/problems/minimum-number-of-operations-to-make-string-sorted/description/
- Implement a thread pool for sending messages.
- https://leetcode.com/problems/subarray-sum-equals-k/
- https://leetcode.com/problems/length-of-longest-fibonacci-subsequence/
- Test a quicksort pivot operation
- Implement C++ shared pointer class (https://medium.com/analytics-vidhya/c-shared-ptr-and-how-to-write-your-own-d0d385c118ad)
- https://github.com/olibre/Cpp14-OrderBook

| **Problem Title**                                                | **Link**                                                                                                                                                                |
| ---------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **348. Design Tic-Tac-Toe**                                      | [348. Design Tic-Tac-Toe](https://leetcode.com/problems/design-tic-tac-toe/)                                                                                            |
| **718. Maximum Length of Repeated Subarray**                     | [718. Maximum Length of Repeated Subarray](https://leetcode.com/problems/maximum-length-of-repeated-subarray/)                                                          |
| **767. Reorganize String**                                       | [767. Reorganize String](https://leetcode.com/problems/reorganize-string/)                                                                                              |
| **782. Transform to Chessboard**                                 | [782. Transform to Chessboard](https://leetcode.com/problems/transform-to-chessboard/)                                                                                  |
| **875. Koko Eating Bananas**                                     | [875. Koko Eating Bananas](https://leetcode.com/problems/koko-eating-bananas/)                                                                                          |
| **981. Time Based Key-Value Store**                              | [981. Time Based Key-Value Store](https://leetcode.com/problems/time-based-key-value-store/)                                                                            |
| **1048. Longest String Chain**                                   | [1048. Longest String Chain](https://leetcode.com/problems/longest-string-chain/)                                                                                       |
| **1155. Number of Dice Rolls With Target Sum**                   | [1155. Number of Dice Rolls With Target Sum](https://leetcode.com/problems/number-of-dice-rolls-with-target-sum/)                                                       |
| **1670. Design Front Middle Back Queue**                         | [1670. Design Front Middle Back Queue](https://leetcode.com/problems/design-front-middle-back-queue/)                                                                   |
| **2044. Count Number of Maximum Bitwise-OR Subsets**             | [2044. Count Number of Maximum Bitwise-OR Subsets](https://leetcode.com/problems/count-number-of-maximum-bitwise-or-subsets/)                                           |
| **2661. First Completely Painted Row or Column**                 | [2661. First Completely Painted Row or Column](https://leetcode.com/problems/first-completely-painted-row-or-column/)                                                   |
| **2958. Length of Longest Subarray With at Most K Frequency**    | [2958. Length of Longest Subarray With at Most K Frequency](https://leetcode.com/problems/length-of-longest-subarray-with-at-most-k-frequency/)                         |
| **69. Sqrt(x)**                                                  | [69. Sqrt(x)](https://leetcode.com/problems/sqrtx/)                                                                                                                     |
| **297. Serialize and Deserialize Binary Tree**                   | [297. Serialize and Deserialize Binary Tree](https://leetcode.com/problems/serialize-and-deserialize-binary-tree/)                                                      |
| **412. Fizz Buzz**                                               | [412. Fizz Buzz](https://leetcode.com/problems/fizz-buzz/)                                                                                                              |
| **460. LFU Cache**                                               | [460. LFU Cache](https://leetcode.com/problems/lfu-cache/)                                                                                                              |
| **622. Design Circular Queue**                                   | [622. Design Circular Queue](https://leetcode.com/problems/design-circular-queue/)                                                                                      |
| **765. Couples Holding Hands**                                   | [765. Couples Holding Hands](https://leetcode.com/problems/couples-holding-hands/)                                                                                      |
| **1004. Max Consecutive Ones III**                               | [1004. Max Consecutive Ones III](https://leetcode.com/problems/max-consecutive-ones-iii/)                                                                               |
| **1606. Find Servers That Handled Most Number of Requests**      | [1606. Find Servers That Handled Most Number of Requests](https://leetcode.com/problems/find-servers-that-handled-most-number-of-requests/)                             |
| **2598. Smallest Missing Non-negative Integer After Operations** | [2598. Smallest Missing Non-negative Integer After Operations](https://leetcode.com/problems/smallest-missing-non-negative-integer-after-operations/)                   |
| **49. Group Anagrams**                                           | [49. Group Anagrams](https://leetcode.com/problems/group-anagrams/)                                                                                                     |
| **200. Number of Islands**                                       | [200. Number of Islands](https://leetcode.com/problems/number-of-islands/)                                                                                              |
| **210. Course Schedule II**                                      | [210. Course Schedule II](https://leetcode.com/problems/course-schedule-ii/)                                                                                            |
| **1515. Best Position for a Service Centre**                     | [1515. Best Position for a Service Centre](https://leetcode.com/problems/best-position-for-a-service-centre/)                                                           |
| **2050. Parallel Courses III**                                   | [2050. Parallel Courses III](https://leetcode.com/problems/parallel-courses-iii/)                                                                                       |
| **1268. Search Suggestions System**                              | [1268. Search Suggestions System](https://leetcode.com/problems/search-suggestions-system/)                                                                             |
| **1632. Rank Transform of a Matrix**                             | [1632. Rank Transform of a Matrix](https://leetcode.com/problems/rank-transform-of-a-matrix/)                                                                           |
| **3186. Maximum Total Damage With Spell Casting**                | [3186. Maximum Total Damage With Spell Casting](https://leetcode.com/problems/maximum-total-damage-with-spell-casting/)                                                 |
| **1. Two Sum**                                                   | [1. Two Sum](https://leetcode.com/problems/two-sum/)                                                                                                                    |
| **5. Longest Palindromic Substring**                             | [5. Longest Palindromic Substring](https://leetcode.com/problems/longest-palindromic-substring/)                                                                        |
| **17. Letter Combinations of a Phone Number**                    | [17. Letter Combinations of a Phone Number](https://leetcode.com/problems/letter-combinations-of-a-phone-number/)                                                       |
| **31. Next Permutation**                                         | [31. Next Permutation](https://leetcode.com/problems/next-permutation/)                                                                                                 |
| **39. Combination Sum**                                          | [39. Combination Sum](https://leetcode.com/problems/combination-sum/)                                                                                                   |
| **42. Trapping Rain Water**                                      | [42. Trapping Rain Water](https://leetcode.com/problems/trapping-rain-water/)                                                                                           |
| **46. Permutations**                                             | [46. Permutations](https://leetcode.com/problems/permutations/)                                                                                                         |
| **50. Pow(x, n)**                                                | [50. Pow(x, n)](https://leetcode.com/problems/powx-n/)                                                                                                                  |
| **53. Maximum Subarray**                                         | [53. Maximum Subarray](https://leetcode.com/problems/maximum-subarray/)                                                                                                 |
| **70. Climbing Stairs**                                          | [70. Climbing Stairs](https://leetcode.com/problems/climbing-stairs/)                                                                                                   |
| **79. Word Search**                                              | [79. Word Search](https://leetcode.com/problems/word-search/)                                                                                                           |
| **98. Validate Binary Search Tree**                              | [98. Validate Binary Search Tree](https://leetcode.com/problems/validate-binary-search-tree/)                                                                           |
| **123. Best Time to Buy and Sell Stock III**                     | [123. Best Time to Buy and Sell Stock III](https://leetcode.com/problems/best-time-to-buy-and-sell-stock-iii/)                                                          |
| **239. Sliding Window Maximum**                                  | [239. Sliding Window Maximum](https://leetcode.com/problems/sliding-window-maximum/)                                                                                    |
| **2484. Count Palindromic Subsequences**                         | [2484. Count Palindromic Subsequences](https://leetcode.com/problems/count-palindromic-subsequences/)                                                                   |
| **647. Palindromic Substrings**                                  | [647. Palindromic Substrings](https://leetcode.com/problems/palindromic-substrings/)                                                                                    |
| **121. Best Time to Buy and Sell Stock**                         | [121. Best Time to Buy and Sell Stock](https://leetcode.com/problems/best-time-to-buy-and-sell-stock/)                                                                  |
| **2702. Minimum Operations to Make Numbers Non-negative**        | [2702. Minimum Operations to Make Numbers Non-negative](https://www.google.com/search?q=https://leetcode.com/problems/minimum-operations-to-make-numbers-non-negative/) |
| **56. Merge Intervals**                                          | [56. Merge Intervals](https://leetcode.com/problems/merge-intervals/)                                                                                                   |
| **146. LRU Cache**                                               | [146. LRU Cache](https://leetcode.com/problems/lru-cache/)                                                                                                              |
| **1801. Number of Orders in the Backlog**                        | [1801. Number of Orders in the Backlog](https://leetcode.com/problems/number-of-orders-in-the-backlog/)                                                                 |
| **295. Find Median from Data Stream**                            | [295. Find Median from Data Stream](https://leetcode.com/problems/find-median-from-data-stream/)                                                                        |
| **399. Evaluate Division**                                       | [399. Evaluate Division](https://leetcode.com/problems/evaluate-division/)                                                                                              |
| **287. Find the Duplicate Number**                               | [287. Find the Duplicate Number](https://leetcode.com/problems/find-the-duplicate-number/)                                                                              |
| **380. Insert Delete GetRandom O(1)**                            | [380. Insert Delete GetRandom O(1)](https://leetcode.com/problems/insert-delete-getrandom-o1/)                                                                          |
| **759. Employee Free Time**                                      | [759. Employee Free Time](https://leetcode.com/problems/employee-free-time/)                                                                                            |
| **1229. Meeting Scheduler**                                      | [1229. Meeting Scheduler](https://leetcode.com/problems/meeting-scheduler/)                                                                                             |
| **127. Word Ladder**                                             | [127. Word Ladder](https://leetcode.com/problems/word-ladder/)                                                                                                          |
| **149. Max Points on a Line**                                    | [149. Max Points on a Line](https://leetcode.com/problems/max-points-on-a-line/)                                                                                        |
| **150. Evaluate Reverse Polish Notation**                        | [150. Evaluate Reverse Polish Notation](https://leetcode.com/problems/evaluate-reverse-polish-notation/)                                                                |
| **151. Reverse Words in a String**                               | [151. Reverse Words in a String](https://leetcode.com/problems/reverse-words-in-a-string/)                                                                              |
| **155. Min Stack**                                               | [155. Min Stack](https://leetcode.com/problems/min-stack/)                                                                                                              |
| **188. Best Time to Buy and Sell Stock IV**                      | [188. Best Time to Buy and Sell Stock IV](https://leetcode.com/problems/best-time-to-buy-and-sell-stock-iv/)                                                            |
| **208. Implement Trie (Prefix Tree)**                            | [208. Implement Trie (Prefix Tree)](https://leetcode.com/problems/implement-trie-prefix-tree/)                                                                          |
| **221. Maximal Square**                                          | [221. Maximal Square](https://leetcode.com/problems/maximal-square/)                                                                                                    |
| **279. Perfect Squares**                                         | [279. Perfect Squares](https://leetcode.com/problems/perfect-squares/)                                                                                                  |
| **346. Moving Average from Data Stream**                         | [346. Moving Average from Data Stream](https://leetcode.com/problems/moving-average-from-data-stream/)                                                                  |
| **2361. Minimum Costs Using the Train Line**                     | [2361. Minimum Costs Using the Train Line](https://leetcode.com/problems/minimum-costs-using-the-train-line/)                                                           |
| **23. Merge k Sorted Lists**                                     | [23. Merge k Sorted Lists](https://leetcode.com/problems/merge-k-sorted-lists/)                                                                                         |
| **34. Find First and Last Position of Element in Sorted Array**  | [34. Find First and Last Position of Element in Sorted Array](https://leetcode.com/problems/find-first-and-last-position-of-element-in-sorted-array/)                   |
| **37. Sudoku Solver**                                            | [37. Sudoku Solver](https://leetcode.com/problems/sudoku-solver/)                                                                                                       |
| **103. Binary Tree Zigzag Level Order Traversal**                | [103. Binary Tree Zigzag Level Order Traversal](https://leetcode.com/problems/binary-tree-zigzag-level-order-traversal/)                                                |
| **124. Binary Tree Maximum Path Sum**                            | [124. Binary Tree Maximum Path Sum](https://leetcode.com/problems/binary-tree-maximum-path-sum/)                                                                        |
| **126. Word Ladder II**                                          | [126. Word Ladder II](https://leetcode.com/problems/word-ladder-ii/)                                                                                                    |
| **204. Count Primes**                                            | [204. Count Primes](https://leetcode.com/problems/count-primes/)                                                                                                        |
| **207. Course Schedule**                                         | [207. Course Schedule](https://leetcode.com/problems/course-schedule/)                                                                                                  |
| **253. Meeting Rooms II**                                        | [253. Meeting Rooms II](https://leetcode.com/problems/meeting-rooms-ii/)                                                                                                |
| **256. Paint House**                                             | [256. Paint House](https://leetcode.com/problems/paint-house/)                                                                                                          |
| **332. Reconstruct Itinerary**                                   | [332. Reconstruct Itinerary](https://leetcode.com/problems/reconstruct-itinerary/)                                                                                      |
| **465. Optimal Account Balancing**                               | [465. Optimal Account Balancing](https://leetcode.com/problems/optimal-account-balancing/)                                                                              |
| **631. Design Excel Sum Formula**                                | [631. Design Excel Sum Formula](https://leetcode.com/problems/design-excel-sum-formula/)                                                                                |
| **1203. Sort Items by Groups Respecting Dependencies**           | [1203. Sort Items by Groups Respecting Dependencies](https://leetcode.com/problems/sort-items-by-groups-respecting-dependencies/)                                       |
