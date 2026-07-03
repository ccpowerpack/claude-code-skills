---
name: context-econ
description: Token and context-window economy for long agentic sessions. Use before starting any large multi-file task, when a session is burning usage limits, when responses show context drift ("you already read that file", "we decided this an hour ago"), or on request like "plan the context for this", "we're going to be at this a while", "stop re-reading everything". Enforces a written context plan up front and state-to-disk at every phase boundary.
---

# Context Economy

The failure mode: a long session spends its window on exploration garbage — full-file reads that needed 30 lines, verbose test logs pasted whole, the same file re-read after every edit — until compaction eats the early decisions and the agent starts contradicting its own plan. Context is a budget, and like any budget the fix is not frugality vibes, it is a written plan and durable checkpoints. Conversation memory is a cache; disk is the source of truth.

## Phase 1 — Context plan (gate: before any large task)

Before the first file read of a task expected to exceed ~10 tool calls, write the plan down (in the response, or in the plan file if one will exist anyway):

```
CONTEXT PLAN
Read fully:    src/billing/invoice.py (I will edit it), src/billing/types.py (small, load-bearing)
Read targeted: src/api/routes.py — only the 2 handlers that call invoice code (grep first, read ±20 lines)
Search only:   usages of `InvoiceState` across repo — grep -rn, read nothing unless a hit surprises me
Summarize:     the 400-line migration history — one grep for the latest schema, ignore the rest
Do not read:   tests/fixtures/*.json (large, content irrelevant to the change)
State file:    NOTES.md — decisions land here at each phase boundary
```

The classification test for each file: *will a specific line of it change my edit?* If yes → targeted read of that region. If you'll edit it → full read. If you only need to know *whether/where* something exists → search, don't read. No classification, no reads.

## Cost-behavior table — what actually burns the window

| Behavior | Real cost | Mitigation |
|---|---|---|
| Full read of a 1,500-line file to answer one question | ~15–20k tokens, mostly noise that also degrades attention | `grep -n` for the symbol first, then `Read` with offset/limit around the hit (±30 lines) |
| Re-reading a file after your own edit "to verify" | Full file cost again, per edit, per file | Don't. The edit tool errored if it failed. Re-read only after *external* changes (formatter, codegen, another agent) |
| Pasting whole test/build logs | A failing `pytest -v` run can be 10k+ tokens of pass noise | `pytest -x -q 2>&1 | tail -30`, or `... | grep -A 10 "FAILED\|Error"`. You need the failure, not the celebration |
| Broad exploration in the main thread | Every dead-end file read stays in context forever | Delegate to a subagent (below) — its exploration dies with it, only the conclusion returns |
| Repeated identical tool calls (same grep, same read) | Full duplicate cost; also a drift symptom | If you notice yourself repeating a lookup, that fact belongs in the state file |
| Long back-and-forth design debate left only in conversation | Fine until compaction, then the *losing* option sometimes survives summarization better than the winner | Decisions go to disk the moment they're made (Phase 2) |

## Session structure — the standing pattern

Long tasks run in this shape; each arrow is a boundary with an artifact:

```
EXPLORE            → DISTILL              → COMPACT/CLEAR      → EXECUTE
read/search/spike    write findings to      drop the raw         work FROM the file;
freely, tolerate     PLAN.md/NOTES.md:      exploration; the     reload it as the
dead ends — but      decisions, file:line   file survives,       first act after any
delegate the big     targets, landmines,    the transcript       compaction; update
sweeps (Phase 3)     rejected options+why   doesn't need to      it at each subtask
```

Why this order works: exploration is the phase with the worst signal-to-token ratio (most of what you read turns out irrelevant), so it is exactly the content you want to pay for once, compress, and discard. Execution, by contrast, needs almost no history — it needs the distilled plan and the files being edited. Separating them means the execution phase starts with a nearly empty window instead of inheriting 80k tokens of dead reads.

The pattern also nests: within execution, a subtask that needs its own exploration (say, an unexpected test failure) gets a mini explore→distill cycle — the finding goes into NOTES.md's Landmines section, not just the transcript.

## Phase 2 — Distill at phase boundaries (gate)

At every phase boundary — exploration done, plan approved, subtask finished — you may not start the next phase until durable state is written to a file. Not "I'll remember"; a file. Minimum contents:

```markdown
# NOTES.md (or the task's plan file)
## Decided
- Cursor pagination, opaque token = base64(id+sort_key). Offset rejected: feed mutates under reader.
## Done
- invoice.py: extracted `compute_tax()` — tests green (`pytest tests/billing -q` → 41 passed)
## Next
- routes.py handlers still call old signature (2 call sites: routes.py:88, routes.py:214)
## Landmines
- types.py `Money` is frozen dataclass — don't add mutable fields
```

Why a file and not memory: compaction summarizes; summaries drop the "why" and occasionally invert a decision. A file survives compaction, survives `/clear`, survives handing the session to a different model tomorrow. **Write for the reader who has zero conversation history**, because after compaction, that reader is you.

Concrete triggers to compact/clear (do it at the boundary, not mid-task):
- Exploration phase is over and its findings are distilled to the file → compact; the raw reads are now dead weight.
- Context warnings appear, or you're about to start a mechanically repetitive phase (apply the same edit to 12 files) that needs no history.
- You catch yourself re-establishing facts already established → distill immediately, then compact.

## Phase 3 — Delegate exploration, keep conclusions

Fan out to subagents any subtask whose *working set is large but whose answer is small*:

| Delegate | Keep in main thread |
|---|---|
| "Find every call site of X and report file:line + calling convention" | The 3 call sites you will actually edit |
| "Read these 5 candidate modules, report which one owns retry logic" | Editing that one module |
| "Run the full test suite, report only failures with tracebacks" | The specific failing test you're fixing |
| "Survey how this repo does pagination, one paragraph + one example path" | Applying the pattern |

Rule of thumb: if the subtask would add >5 file reads to your context and you need <20 lines of its output, it's a delegation. The subagent's dead-end reads cost *its* window, not yours. Give it a sharp output contract or it will paste files back at you and defeat the purpose. Template:

```
Find every place that constructs a Stripe client in this repo (search for
"stripe.Client", "NewStripe", env var STRIPE_KEY). OUTPUT CONTRACT: a list of
file:line + one sentence each on how the client is configured there. Do NOT
paste function bodies. If more than 10 sites, group by package and report counts.
```

Two things make that template work: the search hints (the subagent doesn't share your context, so seed it with the naming conventions you already know) and the overflow rule (caps the worst-case size of what comes back). What does NOT delegate well: the edit itself (you'd need to verify it, re-paying the cost), and judgment calls that depend on conversation history the subagent doesn't have.

**Drift symptoms — when you see one, stop and run the Phase 2 distill immediately:**

- [ ] You (or the user) reference a decision and the reasoning behind it is no longer in context
- [ ] You're about to re-run a search/read you have a feeling you already did
- [ ] Your current edit contradicts something the plan file says (either the plan rotted or you drifted — resolve on disk, not in prose)
- [ ] The user says "you already did that" or "we decided against that"

## Tool-output hygiene — quick reference

The window pays for tool output at the same rate as reasoning. Standing substitutions:

```bash
pytest tests/ -x -q 2>&1 | tail -30            # not: pytest -v (pass noise)
go test ./... 2>&1 | grep -v "^ok" | head -40  # failures only
npm test -- --silent 2>&1 | tail -40           # suppress per-test chatter
git diff --stat                                # shape first; full diff only for files you'll discuss
git log --oneline -15                          # not: git log (full bodies)
grep -rn "symbol" src/ | head -20              # cap unknown-cardinality searches
find . -name "*.test.ts" | wc -l               # when you need the count, get the count
make build 2>&1 | grep -E "error|warning" || echo BUILD_OK   # errors or one token
```

The pattern behind all of these: decide what question the command answers *before* running it, then shape the output to exactly that answer. A command whose output you will scroll past is a command you ran wrong.

## Worked example — same task, two ways

Task: rename `Account.plan` to `Account.tier` across a mid-size repo (models, 3 services, tests, one serializer).

**Naive session:** read `models/account.py` in full (400 lines, ~5k tok). Read all of `services/` one file at a time hunting for usages (6 files × ~3k = 18k). Edit each, re-read each after editing (another 18k). Run `pytest -v`, paste full output (9k). Two failures; re-read the two test files in full (6k). Fix, re-run full verbose suite (9k). **≈65k tokens**, and by the end the window also contains four stale copies of edited files — prime drift material.

**Planned session:** `grep -rn "\.plan\b\|'plan'\|\"plan\"" --include="*.py" | grep -i account` → 14 hits, ~1k tok. Context plan: full-read only `models/account.py` lines 1–80 (the class, found via grep) — 1k. Targeted read ±10 lines around each remaining hit — 14 × ~300 = 4k. Edit all sites, no re-reads. `pytest -x -q 2>&1 | tail -20` — 0.5k. One failure; read that test's failing function only — 0.5k. Fix, re-run tail'd — 0.5k. Write 6-line NOTES entry. **≈8k tokens.** Same rename, same green suite, 8× cheaper — and the delta compounds: the naive session hits compaction by mid-afternoon, the planned one doesn't.

## Multi-session and handoff

The same discipline covers the harder cases for free:

- **Resuming tomorrow**: session starts by reading NOTES.md, not by re-exploring. If resuming requires re-exploration, yesterday's distill gate was failed — note that and fix the file first.
- **Handing to another agent/model**: the state file plus the context plan IS the handoff artifact. An orchestrator that hands a subtask to an executor without a written state file is asking the executor to re-derive intent from a diff — intent loss is the default outcome, not the edge case.
- **CLAUDE.md**: facts that keep getting re-established across *sessions* (build command quirks, "tests must run from repo root", directory map) graduate from NOTES.md to CLAUDE.md. NOTES.md is per-task; CLAUDE.md is per-repo. Promoting a fact costs one line once; re-deriving it costs a grep-and-read every session forever.

## Output contract

For any large task: the context plan (before reads), the state file updated at each boundary, and a final entry in it listing what was done + verification evidence. If you compacted, the post-compaction turn must begin by reloading the state file — cite it, don't reconstruct from memory.

## Anti-patterns (hard no)

- Reading any file in full when a grep + targeted read answers the question.
- Re-reading a file immediately after your own successful edit to "double-check".
- Pasting full verbose test/build output when `tail`/`grep` would carry the failure.
- Keeping exploration in the main context when a subagent could return a 10-line conclusion.
- Relying on the model to "remember" a decision across compaction instead of writing it to the state file — if it matters after compaction, it goes on disk *before* compaction.
- Compacting mid-task with undistilled state (compaction is for boundaries you prepared, not a panic button).
