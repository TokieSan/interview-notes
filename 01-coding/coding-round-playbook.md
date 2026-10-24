# Coding Round Playbook

How to run any live coding round: the protocol, the plain-doc reality, the checklists, and OA tactics. Problems live in the sibling files.

## 1. The protocol (say it out loud, every problem)

**What they grade (the rubric in plain terms):**
1. **Do you clarify before coding?** Open prompts are deliberate. Ask about input format, size, malformed data, duplicates, output ordering, empty input.
2. **Is the code clean and idiomatic C++?** STL fluency, sensible names, no spaghetti. They are explicitly testing "you understand your language usage well."
3. **Does it actually work?** Compile it in your head, then *dry-run a sample input out loud*. Find your own bugs before they do.
4. **Edge cases.** Empty input, one element, all-same, malformed line, integer overflow, off-by-one.
5. **Complexity.** State time + space for your solution unprompted. Know if it can be better.
6. **Production mindset (SRE flavour).** "What if the file is 500GB?" → stream line-by-line, don't load into memory. "What if a line is corrupt?" → skip + count, don't crash.

**The protocol — say this out loud, every problem:**
```
1. Restate: "So I'm parsing a log where each line is <format>, and I need <output>. Correct?"
2. Clarify:  size? malformed lines? ties? ordering? case sensitivity? empty?
3. Approach:  "I'll use a hash map to count, then a heap for top-K. O(n log k) time."
4. Code:      write it, narrate as you go.
5. Test:      dry-run a 3-line example, trace variables.
6. Optimize:  "If memory is tight I'd stream; if I need top-K I'd swap full sort for a heap."
```

**Golden rule:** never go silent. A wrong idea spoken aloud beats a right idea in your head — they grade *thinking*.

**C++ scaffold to open with (memorize this):**
```cpp
#include <bits/stdc++.h>   // single header; fine for interviews
using namespace std;       // fine for interviews; note you'd avoid in prod headers

int main() {
    ios_base::sync_with_stdio(false);  // optional speed line; mention you know it
    cin.tie(nullptr);
    // ...
    return 0;
}
```

## 2. Coding in a plain doc (Google-style, no IDE)

**The critical mechanical fact: you code in a plain Google Doc.**
- **No IDE. No compiler. No autocomplete. No syntax highlighting. No auto-indent. No error squiggles.**
- You type C++ as plain text while the interviewer watches live.
- This is the #1 thing people underestimate. Indentation is manual. Typos don't get flagged. You have to compile and find bugs *in your head*.

- **Write at least 3–4 of the [practical problems](sre-practical-problems.md) in an actual plain Google Doc or a no-frills text editor** — not VS Code. Get used to: no autocomplete on `unordered_map`, manually indenting, no red underlines.
- Build muscle memory for your scaffold so you don't waste time:
  ```cpp
  #include <bits/stdc++.h>
  using namespace std;
  int main() {
      // ...
      return 0;
  }
  ```
- Practice **narrating while typing** — silence in a plain doc is worse than in an IDE because the interviewer only has your doc + face to read.
- Keep formatting clean even without auto-indent: consistent 4-space indents, blank lines between logical blocks. Readability is graded.

### Doc-coding discipline

- Write the **signature and a one-line contract first**, then the body. It gives the interviewer something to react to before you've spent 10 minutes.
- Use short, unambiguous variable names — you have no autocomplete and no highlighting.
- Leave a blank line between logical blocks; the doc has no folding.
- **After you finish: write `# Test:` and at least two cases, including one edge case, and trace one by hand out loud.** Often the Lean-Hire/Hire boundary.
- State complexity in time *and* space, and state what you'd change if the input didn't fit in memory.

### Language choice

Pick one language for the whole loop and say why. C++ is a legitimate choice everywhere, but in a bare doc it costs keystrokes and typo surface; Python is faster to write and interviewers rarely care. Some companies (and individual interviewers) have strong preferences — ask at minute zero: *"I'm most fluent in C++ but happy to use Python — do you have a preference?"* That avoids losing flow to a mid-round language switch.

## 3. Edge-case checklist (your safety net)

Run every solution through this mentally:
1. **Empty input** — zero lines, empty string, empty vector.
2. **Single element.**
3. **All identical / all distinct.**
4. **Boundaries** — first/last element, index 0 and `size()-1`.
5. **Malformed data** — missing fields, non-numeric where number expected, wrong delimiter → *skip + count, never crash* (SRE mindset).
6. **Overflow** — sums/products in `long long`; check before multiplying.
7. **Negative numbers / zero** where you assumed positive.
8. **Ties** — what's the tiebreak rule for "top K"?
9. **Ordering** — does output need to be sorted? By what?
10. **Huge input** — does it fit in memory, or must I stream line-by-line?

## 4. The "production-ready" framing that wins SRE points

Whenever you finish, add one of these:
- "For a 500GB log I'd stream line-by-line — O(1) memory beyond the aggregate — rather than load it all."
- "Malformed lines get skipped and counted, with the count logged, so one bad line doesn't take down the job."
- "For top-K at scale I'd use a bounded heap, not a full sort — O(n log K), O(K) memory."
- "Percentiles at scale → approximate sketch (t-digest/HDR histogram), not every sample in RAM."
- "This recursion could blow the stack on adversarial input; I'd convert to an explicit stack / set a depth bound."

These sentences are the difference between "good coder" and "thinks like an SRE."

## 5. Final 60-second checklist before you submit

1. ☐ Did I dry-run a real example out loud?
2. ☐ Empty input handled?
3. ☐ Single element / single line?
4. ☐ Malformed line — skip, not crash?
5. ☐ Integer overflow (use `long long` for sums)?
6. ☐ Off-by-one in loops / `substr` bounds?
7. ☐ Stated time + space complexity?
8. ☐ Named variables sensibly?
9. ☐ Did I mention the "what if input is huge → stream it" angle?

## 6. Online assessments (CodeSignal / HackerRank style)

From 2025–2026 candidate reports for trading-firm OAs (HRT especially); the tactics generalise.

From 2025–2026 candidate reports (Blind, Glassdoor, EverythingQuant, programhelp, Reddit):

- **Platform:** CodeSignal most common (sometimes HackerRank). Standardized scoring on CodeSignal is **/600**.
- **Questions:** **3–4 problems**, difficulty **LC easy → hard**, usually ascending. Q1–Q3 are typically quick; **Q4 is the filter** (one report: a niche file-IO problem; another: an old unsolved-online MSFT-style DP).
- **Time:** commonly **~70 min** (4 in 70 is the modal report); some custom/role variants run **90–150 min** (Algo Dev take-home ~2.5 hrs with 3 questions; a 3-hr data variant exists). Don't assume — read your invite.
- **Pass bar is brutal:** multiple reports say **~90–95%+ / near-full AC** is needed; one low-latency C++ role reportedly required **all** questions solved. A 3/4 with an un-optimized Q4 frequently does **not** advance.
- **Language:** Python and C++ are the usual choices; some roles **restrict to C++**. A few OAs are **pandas/data-science** flavored (groupby, merge, pivot, missing values) — see `HRT_02 §E`.
- **Possible wildcards:** a **pandas** question, a **file-IO** question, **SQL** (for data/quant variants), **number-systems** (binary/hex), and **floating-point/bit-manipulation**.

**OA tactics:**
1. **Solve in order, bank the easy points fast.** Speed is scored implicitly (timeouts cost you). Don't perfect Q1 while Q4 sits untouched — but note some variants gate Q4 behind completing Q3.
2. **Test your own edge cases** — HRT's blog and CodeSignal both warn sample tests aren't comprehensive. Empty input, single element, duplicates, overflow (use `long long`), off-by-one, negative numbers.
3. **Watch complexity** — the "I solved it but it timed out" failure is extremely common in reports. Estimate N and target the right Big-O *before* coding.
4. **Don't over-style** — for the OA, correctness + speed beat maintainability (per HRT's own prep blog).
5. **No LLMs.** Later rounds are designed to catch OA cheating; flagged candidates aren't re-invited.

## 7. Live coding & debugging rounds — tips that generalise

- **Lead with C++**, but state up front you can read/write Python — they may hand you an unfamiliar language fragment to gauge adaptability and grade with your familiarity in mind.
- **Restate the problem and confirm constraints before coding.** "Dove in before understanding" is the #1 documented onsite failure. Work a 30-second example out loud.
- **State complexity before and after.** Estimate N → pick target Big-O → verify your solution hits it. "Solved but timed out" is the most common OA death.
- **Test your own code unprompted** — pick edge cases and trace them. Self-debugging speed is explicitly graded.
- **Narrate continuously; take hints gracefully.** Applying a hint from an earlier question to a later one ("teachability") is a documented strong positive.
- **Don't over-engineer.** Write the direct correct solution first; optimize when asked. "Avoiding real work via overly complex solutions" is a named failure mode.
- **`long long` for sums/products, `size_t` care in loops, `.at()` mentally for bounds** — small correctness habits that prevent the careless bugs HRT punishes.
- If you're algorithmically strong, your risk isn't the algorithm, it's **rushing into a misunderstood problem** or a **careless implementation bug**. Slow down for 30 seconds at the start; it's the highest-ROI thing you can do.
