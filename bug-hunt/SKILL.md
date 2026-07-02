---
name: bug-hunt
description: Systematic root-cause debugging loop — reproduce, isolate, instrument, fix, prove. Use when investigating a bug, crash, flaky test, regression, or any "why is this broken / not working / behaving weirdly" request. Refuses to patch symptoms without a confirmed root cause.
---

# Bug Hunt

You are running a disciplined debugging loop. The cardinal rule: **you may not write a fix until you can state the root cause in one sentence and point to the line(s) that prove it.**

## Phase 1 — Reproduce (gate: do not proceed without this)

1. Restate the expected vs. actual behavior in one line each.
2. Find or build the smallest command that demonstrates the bug: a failing test, a curl call, a REPL snippet. Run it. Capture the exact output.
3. If you cannot reproduce it:
   - Check environment deltas (versions, env vars, data state, timezone, locale).
   - Ask for the exact input/logs from when it failed — do NOT guess-fix.
   - A bug you can't reproduce is a bug you can't prove you fixed.

## Phase 2 — Isolate

Shrink the search space before reading code broadly:

- **Bisect by time**: `git log` — did this ever work? `git bisect` if a known-good commit exists.
- **Bisect by input**: cut the failing input in half until minimal.
- **Bisect by layer**: is the bad value produced, transformed, or displayed wrong? Check the data at each boundary (DB → service → API → UI) and find the first boundary where it's already wrong.

State your current hypothesis explicitly before each probe: "If X is the cause, then adding a log at Y will show Z."

## Phase 3 — Instrument

- Add targeted logging/assertions at the suspected boundary — not scattershot prints.
- Prefer reading the actual runtime value over reasoning about what it "should" be.
- Remove every instrument you added before finishing (verify with `git diff`).

## Phase 4 — Root cause statement (gate)

Before touching the fix, write:

```
ROOT CAUSE: <one sentence>
EVIDENCE: <file:line + the observed value/behavior that confirms it>
WHY IT ESCAPED: <why tests/review didn't catch it>
```

If you can't fill in EVIDENCE with something you actually observed, return to Phase 3.

## Phase 5 — Fix and prove

1. Write a regression test that fails for the root-cause reason (run it, show the failure).
2. Apply the minimal fix at the causal site — not a defensive band-aid downstream.
3. Run the regression test (now passes) plus the surrounding test suite.
4. Re-run the original reproduction from Phase 1.

## Output contract

Finish with: root cause statement, the fix diff summary, proof (test output before/after), and anything suspicious you noticed but didn't fix.

## Anti-patterns (hard no)

- "This might fix it" commits.
- Adding try/catch or null-checks to silence a symptom whose cause you haven't found.
- Fixing the test instead of the code without justifying why the test was wrong.
- Declaring victory without re-running the original reproduction.
