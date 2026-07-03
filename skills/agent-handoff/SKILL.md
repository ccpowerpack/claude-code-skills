---
name: agent-handoff
description: Design the artifact one agent hands to another to execute — orchestrator→executor, strong→cheap model, or session→session. Banks a strong model's judgment into a spec a cheaper/later agent can run without re-deciding anything. Use for "write a handoff / plan / spec for another agent", "break this down for a sub-agent to execute", "prepare this for a cheaper model", "hand this off", or when you're the strong model deciding and something else will do the doing. Refuses to ship a handoff missing paths, acceptance commands, non-goals, or rationale.
---

# Agent Handoff

You are writing the artifact a *different* agent will execute — often a cheaper model, a fresh session with none of your context, or a parallel worker. The cardinal rule: **a handoff is not done until an executor with zero access to your reasoning can complete it without making a single judgment call you already made.** Every place the executor has to decide is a place it will drift, and it drifts toward the locally-plausible-but-wrong.

The 2026 workflow this serves: spend the strong model's judgment once, freeze it into an artifact, let cheap models execute at scale. That arbitrage only pays if the artifact is airtight. A vague step doesn't get cheaper — it gets re-decided, badly, by something with less taste than you.

## The four gates (a handoff missing any one is not shippable)

1. **Exact file paths.** Every step names the absolute or repo-relative file(s) it touches. No "the auth module", no "the relevant handler." If you can't name the file, you haven't done the executor's discovery for them — do it now.
2. **Per-step acceptance commands.** Each step ends with a command the executor runs to know it's done, and the expected result. `pytest tests/test_auth.py::test_expired_token -q` → `1 passed`. Not "verify it works."
3. **Explicit non-goals.** What must NOT be touched, renamed, reformatted, or "improved." The executor will helpfully wander unless fenced.
4. **Decision rationale for anything re-decidable.** For every tradeoff you already resolved, state the decision AND why, so the executor doesn't reopen it. "Use the existing `Clock` interface, not `time.Now()` directly — the suite injects a fake clock; calling `time.Now()` will make tests flaky."

## Handoff artifact template (fill every field)

```
# Handoff: <one-line objective>

## Context the executor needs (and nothing more)
- Repo root: <path>
- Branch: <name>  |  Base commit: <sha>
- Preconditions to verify first: <command> → <expected>   # fail fast if env is wrong

## Non-goals (do NOT touch)
- <file/dir/behavior that must stay byte-identical>
- Do not reformat, rename, upgrade deps, or "clean up" adjacent code.

## Decisions already made (do NOT re-decide)
- <decision> — because <reason>. If this seems wrong, escalate (see below); don't override.

## Steps
### Step 1 — <imperative title>
- Files: <exact paths>
- Change: <specific edit — what goes where; for non-mechanical steps, the exact signature/shape>
- Acceptance: `<command>` → `<expected output>`

### Step 2 — ...
   (same shape; steps ordered so each acceptance passes before the next starts)

## Final acceptance (all must pass)
- `<full test/build/lint command>` → `<expected>`

## Escalation
- On any ambiguity or an acceptance that won't pass after one honest attempt:
  STOP, do not improvise. Report: step number, what you observed, the specific
  question. Bounce back — do not locally decide.
```

## Failure-mode table

| Vague artifact says | How the executor drifts | The fix in the artifact |
|---|---|---|
| "Refactor the auth module" | Rewrites files you didn't mean, changes behavior, renames symbols other code imports | Name the exact files + the exact edit; list untouched files as non-goals |
| "Add validation" | Invents rules, rejects valid input, or no-ops | Specify the rule: field, constraint, error type, and a test asserting each |
| "Make the code clean" | Reformats the whole file, churns the diff, hides the real change | Delete the step; "clean" is not an acceptance. If style matters, name the linter command |
| "Handle errors appropriately" | Swallows exceptions, logs and continues past corruption | State the policy: which errors surface, which retry, which are fatal, with a test per branch |
| "Update the tests" | Edits assertions to match new (possibly wrong) behavior | Say which tests, expected new values, and which must stay unchanged |
| No acceptance command | Declares done by inspection; ships broken | Every step gets a runnable command + expected output |

## Worked example — mediocre step rewritten as executable

**Before (re-decidable, will drift):**

> Step 3: Refactor the token check in the auth middleware to be more robust and handle expired tokens properly. Make sure it's secure.

An executor faces: which file? what's "robust"? what does "properly" return — 401? 403? refresh? what's "secure"? It will guess, and the guess becomes the contract.

**After (executable):**

> ### Step 3 — Reject expired JWTs with 401
> - Files: `src/middleware/auth.ts` (only)
> - Change: in `verifyToken`, after `jwt.verify`, catch `TokenExpiredError` specifically and `return res.status(401).json({ error: 'token_expired' })`. Do not catch generic `Error` — let malformed-token errors keep their existing 400 path.
> - Decision already made: return 401 (not auto-refresh) — refresh is Step 5; keep them separate so each is independently testable. Do not add refresh logic here.
> - Non-goal: do not touch `src/middleware/rateLimit.ts` or reorder other middleware.
> - Acceptance: `npm test -- auth.spec.ts -t "expired token"` → `1 passing`; and `npm test -- auth.spec.ts -t "malformed token"` → still `1 passing` (proves you didn't broaden the catch).

The rewrite removes every judgment: file, exact edit, the error type to catch and the one to leave alone, why 401 over refresh, what not to touch, and two acceptance commands (one proves the new behavior, one proves you didn't regress the adjacent path).

## Context budgeting — spend the executor's window on what it can't reconstruct

A cheaper/fresh executor has a smaller effective context and no memory of your session. Right-size what you pass:

- **Include** the exact edits, decisions+rationale, non-goals, and acceptance commands — these are what the executor *cannot* re-derive cheaply.
- **Exclude** your exploration transcript, dead ends, and "here's everything I read." That's your cost, already paid; forwarding it just burns the executor's window and buries the instructions.
- **Point, don't paste, for stable references**: cite `file:line` and let the executor open it, rather than inlining large code blocks that may drift from disk.
- **Front-load non-goals and decisions.** If context gets truncated, the fences and the settled tradeoffs are what keep a degraded executor from wandering.
- One artifact = one objective. If the objective needs 20 steps, it probably needs to be two handoffs with a checkpoint between them, not one the executor loses the thread of.

## Worked example — session→session handoff (fresh model, zero shared memory)

You've spent a long session diagnosing a flaky test and know the fix, but the window is nearly full. Instead of continuing, you bank a handoff for a fresh session:

```
# Handoff: fix flaky test_order_expiry (passes ~70% of runs)

## Context
- Repo root: ~/proj/orders  |  Branch: fix/expiry-flake  |  Base: 4a91c2e
- Precondition: `pytest tests/test_order.py::test_order_expiry -q` → flaky (run 5x, expect ≥1 fail)

## Decisions already made (do NOT re-decide)
- Root cause is real-time `datetime.now()` in OrderExpiry, not a test bug — proven: the
  test asserts a 30-min boundary and fails only when the run crosses a wall-clock minute.
  Do NOT "fix" this by widening the assertion tolerance; that hides it.
- Use the existing `Clock` port (src/clock.py), already injected in prod. Do NOT add a new one.

## Non-goals
- Do not touch other tests in tests/test_order.py; do not upgrade freezegun/pytest.

## Steps
### Step 1 — Inject Clock into OrderExpiry
- Files: src/order/expiry.py (only)
- Change: replace `datetime.now()` at expiry.py:42 with `self._clock.now()`; add `clock: Clock`
  ctor param defaulting to `SystemClock()`.
- Acceptance: `pytest tests/test_order.py::test_order_expiry -q` still passes → `1 passed`

### Step 2 — Pin time in the test
- Files: tests/test_order.py (only, this test)
- Change: pass a `FakeClock(fixed=...)` into OrderExpiry in this test.
- Acceptance: `for i in $(seq 20); do pytest tests/test_order.py::test_order_expiry -q || break; done`
  → 20 consecutive `1 passed`, no failure

## Escalation
- If Step 1 acceptance regresses any other test, STOP and report which test + output.
```

A fresh model with none of your debugging context executes this exactly — the root-cause decision (and the explicit "don't widen the tolerance" fence) is what stops it from re-diagnosing and picking the tempting wrong fix.

## Escalation protocol (bind the executor to this)

The executor's default on ambiguity is the single most dangerous behavior — it decides locally. The artifact must instruct, and you must design so the executor *can* comply:

- On any step where the acceptance command doesn't pass after one honest attempt, or where two readings of the instruction diverge: **STOP**. Do not "try something else."
- Report upward with: step number, exact command run, exact output observed, and one specific question ("Step 3 acceptance expects 401 but the existing test at auth.spec.ts:88 asserts 403 — which is authoritative?").
- Never resolve a contradiction between the artifact and the code by silently picking one. A contradiction is a signal the *handoff author* missed something — route it back to them.

## Output contract

Deliver the filled template: objective, context, non-goals, pre-made decisions with rationale, ordered steps (each with exact files, exact change, and an acceptance command + expected output), final acceptance, and the escalation protocol. Before shipping, self-audit against the four gates — name any step that fails one and fix it.

## Anti-patterns (hard no)

- Steps naming a module/area instead of exact files ("refactor the auth module").
- Acceptance criteria requiring judgment ("code should be clean", "handle errors appropriately", "make it robust").
- Re-litigating downstream a tradeoff the handoff already settled — or leaving it settle-able by omitting the rationale.
- Steps without a runnable acceptance command and its expected output.
- Dumping your full reasoning transcript instead of the decisions + rationale the executor actually needs.
- An escalation instruction that says "use your best judgment" — that reintroduces the exact drift the handoff exists to prevent.
