# How to Interview — Behaviours That Move Scores

These apply to every round at every company in this repo. The technical content is in the topic folders; this is about delivery.

## The five behaviours that move scores most

1. **Ask before solving.** Going straight to problem-solving is called out as a red flag at Google, and "dove in before understanding" is the #1 documented onsite failure at HRT. Restate the problem, confirm constraints, work a tiny example.
2. **Think out loud, continuously** — including while writing. Interviewers drop hints; you can only catch them if you're audible. A wrong idea spoken aloud beats a right idea kept in your head: they grade *thinking*.
3. **State assumptions explicitly and check them.** "I'm assuming reads dominate 100:1 — is that right?" Stating assumptions as assumptions is also what makes a later pivot look like engineering instead of correction.
4. **Brute force first, then iterate.** State the naive approach and its complexity, then improve. Don't stall hunting for the elegant answer.
5. **Admit the edge of your knowledge, then reason toward it.** "I haven't worked with X directly; from first principles I'd expect Y because Z — is that the shape of it?" scores better than bluffing and far better than silence. Composure under not-knowing is explicitly valued.

## Say things worth writing down

At Google the hiring committee reads *the written record of what you said*, not a vibe — and most companies work similarly. Crisp, named, quotable statements survive transcription ("this is a metastable failure, so we need to shed below the original load to escape it"). Vague competence does not.

## Answer shape for knowledge questions

**Mechanism → consequence → signal.** What the kernel/library actually does → what breaks or gets slow in production because of it → how you'd observe it. If the first sentence of your answer is a command name, restart the answer. See [04-linux-os/systems-internals-round.md](04-linux-os/systems-internals-round.md).

## When the interviewer pivots

> "Okay — that changes [specific thing]. My current approach assumed [assumption], which no longer holds because [reason]. I see two options: extend what's here by [X], which keeps the structure but costs [Y]; or restructure around [Z], which is cleaner but throws away what we've written. Given the time left I'd lean toward the first — want me to go that way?"

Name the invalidated assumption, offer options with costs, check the time budget. Don't silently rewrite; don't defend the old approach out of sunk cost.

## When you make a choice, name what you rejected

Some interviewers (Apple especially) deliberately withhold feedback and probe whether you can **defend a decision under pressure** — the most commonly reported reason for rejection there. Pre-empt it: "I'm choosing X, which costs me Y; Z would be better if the workload were W."

## When the interviewer is brusque — or wrong

Don't let it rattle you, and don't correct them combatively — that reads as arrogance. State your reasoning, cite the standard or version ("in C++17 this elision is guaranteed because…"), and let it stand. Calm beats right.

## Virtual-interview setup

- Quiet room, wired internet if possible, a phone-hotspot fallback, laptop on power.
- Light in front of you, camera at eye level. Close Slack, email, notifications, other tabs.
- Water, pen and paper for scratch work (tracing a loop, back-of-envelope math).
- Restart your machine the morning of; do a 5-minute test call ~30 minutes before.
- Test the interview link (CoderPad, Meet, Zoom) beforehand — one reported candidate lost interview time to a bad link and it counted against them.
- Camera on, energy up ~20% — video flattens enthusiasm.
- "Let me think about that for a few seconds" is a perfectly good sentence.
- Treat the interviewer as a teammate: ask clarifying questions and take hints well. Applying a hint from earlier in the interview to a later problem is a strong documented positive ("teachability").

## Notes and AI policy

- **No AI tools in any live or recorded stage.** At Google it is disqualification; HRT and Apple state the same; later stages are designed to catch OA cheating.
- Coding must be your own live work — don't have pre-written solutions open. Prepare by internalising patterns.
- For behavioural rounds, glancing at a list of story *tags* is normal on video; reading scripts verbatim is not — follow-ups instantly expose a recited answer.
- Interview questions are usually confidential. Don't post the ones you were asked.

## Energy management

Back-to-back rounds are tiring, and a coding round mid-afternoon after a long gap is where people go flat. Eat properly in gaps, get away from the screen, and don't spend the gap re-reading notes. Use the first 30 seconds of each round to settle rather than sprinting.
