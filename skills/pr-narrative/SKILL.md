---
name: pr-narrative
description: Turn the current branch's diff into a reviewer-ready PR description — intent, change map, risk assessment, test evidence, and a suggested review path. Use when the user asks to "write a PR description", "describe this branch", or prepare a change for review.
---

# PR Narrative

A great PR description is written for a reviewer with 10 minutes and zero context. Your job: let them review confidently without reverse-engineering your intent from the diff.

## Gather (gate: do not write a word of description before this is done)

1. `git log main..HEAD --oneline` (or the repo's default branch) — the commit story.
2. `git diff main...HEAD --stat` — the change footprint.
3. Read the actual diff of the most significant files — never describe changes you haven't read.
4. Check for related context: linked issue in branch name/commits, `CHANGELOG`, recent related PRs.

If the diff is > ~1500 lines, read the core logic changes fully and skim mechanical ones (renames, generated files, lockfiles) — but say which is which in the description.

## Classify the risk (gate: pick a row before writing)

The description's depth and the evidence you demand of yourself follow from the change type, not the line count. Classify against this rubric; a PR spanning multiple rows inherits the strictest one.

| Change type | Reviewer must scrutinize | Required evidence in the description |
|---|---|---|
| **Schema/migration** | Reversibility; lock behavior on large tables; old-code-new-schema window during deploy | Migration tested against a prod-sized copy; rollback statement written out; note on whether the deploy order is code-first or migration-first and why it's safe |
| **Auth/authz-touching** | Fail-open vs fail-closed on every new branch; which routes changed effective policy; token/session lifetime changes | A table or list of routes → before/after policy; a test that proves an *unauthorized* request is rejected (positive tests alone are worthless here) |
| **Concurrency** | Lock ordering; what happens at 2× the assumed parallelism; shared mutable state introduced | Race detector / stress-run output (`go test -race`, repeated-run counts); statement of the invariant being protected |
| **Dependency bump** | Changelog of the dep between versions; transitive changes; license/supply-chain deltas on majors | Link to the dep's changelog with the relevant entries quoted; lockfile diff labeled mechanical; note whether any API the codebase uses changed |
| **Config/infra** | Blast radius per environment; ordering of rollout vs code deploy; secrets handling | Diff of rendered config (not just the template) for one real environment; rollback = revert? if not, say what it is |
| **Pure refactor** | That behavior is actually unchanged — the whole claim rests on this | Test suite green *and* a statement of coverage on the moved code; if coverage is thin, say so explicitly rather than letting "pure refactor" imply safety |

## Write the description

```markdown
## What & why
<2-4 sentences: the user-visible or system-visible outcome, and the reason
 this change exists. Lead with the problem, not the solution.>

## How
<The approach in 3-6 bullets. Name the key design decision and the
 alternative you rejected, if the diff implies one.>

## Change map
<Group files by role, not alphabetically:>
- **Core logic**: `path/to/x.py` — <what changed>
- **Tests**: ...
- **Mechanical/generated**: <renames, lockfiles — reviewable at a glance>

## Risk & rollout
- Blast radius: <what breaks if this is wrong>
- Migration/compat: <schema changes, API compat, feature flags, rollback plan>
- <"None — pure refactor with unchanged behavior, covered by existing tests" is a valid entry IF true>

## Test evidence
<What you ran and what it showed. Paste the actual command + summary line.
 "Tests pass" without evidence is not evidence.>

## Suggested review path
<Ordered list: "Start with the interface change in a.ts, then its use in b.ts,
 then skim the test updates." Save the reviewer the archaeology.>
```

## Worked example — filled template for an auth middleware change

Branch: 9 commits, 14 files, +412/−388. Replaces per-route session checks with centralized auth middleware and adds a refresh-token grace window.

```markdown
## What & why
Session validation is currently copy-pasted into 11 route handlers, and three
of them drifted: they skip the `session.revoked_at` check, so a revoked
session keeps working on those routes until natural expiry (INC-2291). This
PR centralizes validation into one middleware so a policy change is one edit,
and adds a 30s grace window for refresh-token rotation to stop the spurious
401s mobile clients hit mid-rotation.

## How
- New `middleware/require_auth.ts` runs on every route under `/api` except an
  explicit `PUBLIC_ROUTES` allowlist — deny-by-default, so a newly added
  route cannot silently ship unauthenticated (the failure mode that caused
  INC-2291's cousin, INC-2107).
- Rejected alternative: a `@requireAuth` decorator per route. Same code
  dedup, but keeps opt-in semantics — forgetting the decorator fails open.
- Grace window: a rotated-out refresh token stays valid for 30s after
  rotation, tracked in Redis (`rotated:{jti}` with TTL), not in Postgres —
  it's ephemeral by definition.
- The 11 inline checks are deleted in the same PR so there is no window with
  two enforcement points disagreeing.

## Change map
- **Core logic**: `middleware/require_auth.ts` (new, 96 lines) — validation +
  revocation + grace window; `middleware/public_routes.ts` — the allowlist
  (review this list character by character).
- **Deletions**: 11 route files — inline check removal only; no other logic
  touched in those hunks.
- **Tests**: `middleware/require_auth.test.ts` — includes negative cases:
  revoked session → 401, expired grace window → 401, unlisted route without
  token → 401.
- **Mechanical**: `routes/index.ts` import reordering.

## Risk & rollout
- Blast radius: every authenticated endpoint. A bug here is either a
  site-wide lockout (fail-closed, loud) or an auth bypass (fail-open, quiet).
  The middleware is written fail-closed: any validation error → 401.
- Effective policy change: the 3 drifted routes (`GET /api/export`,
  `POST /api/webhooks/test`, `GET /api/admin/stats`) now enforce
  `revoked_at`. Revoked sessions using them will start getting 401s —
  that is the *point*, but it is a behavior change; noted in CHANGELOG.
- Rollback: single revert, no schema change. Redis keys self-expire.

## Test evidence
    $ npm test -- --coverage middleware routes
    Test Suites: 14 passed, 14 total
    Tests:       112 passed, 112 total
    Coverage: middleware/require_auth.ts 100% lines, 96% branches
Manually verified the lockout path on staging: revoked a live session, got
401 on `/api/export` within one request (previously kept working — INC-2291).

## Suggested review path
1. `middleware/require_auth.ts` — the whole policy lives here (96 lines).
2. `middleware/public_routes.ts` — verify nothing sensitive is allowlisted.
3. One route deletion (e.g. `routes/export.ts`) to see the pattern, then
   trust the other 10 — they're mechanically identical hunks.
4. Skim the negative tests to confirm they assert 401, not just "no crash".
```

Note what makes this reviewable: the drifted-route behavior change is self-reported, the rejected decorator alternative is named with its failure mode, and the review path tells the reviewer where the 96 lines that matter live inside a 800-line diff.

## Review path heuristics

When ordering the suggested review path:

- **Interface first**: the type/trait/API-contract change is the PR's thesis; every other hunk is a consequence. A reviewer who has read the interface can predict the call-site diffs.
- **Test diff as spec**: if behavior changed, point the reviewer at the changed *assertions* before the implementation — the test diff states what the code now promises, in less space.
- **One instance of a repeated pattern**: for N mechanically identical changes, direct full attention to one and say the rest are identical — and be right about that, because the reviewer will spot-check.
- **Config and lockfiles last**: low information density; reading them first burns the reviewer's attention budget on the least surprising bytes.
- **Migrations before the code that assumes them**: reviewing model code against the *old* schema in your head produces phantom objections.

## Splitting oversized PRs

If you find yourself writing "this PR also…" more than once, propose a split. Clean cut lines, in order of preference:

1. **Mechanical vs. behavioral**: the rename/move/format PR merges in minutes with `--color-moved=dimmed-zebra` verification; the behavior PR then shows a small honest diff. Never interleave them.
2. **Refactor-then-feature**: PR 1 reshapes existing code with zero behavior change (existing tests prove it); PR 2 adds the feature the new shape enables. Each is independently green and independently revertible.
3. **Schema first**: migration + backward-compatible code in PR 1 (deployable alone), code that requires the new schema in PR 2. This matches how it must deploy anyway.
4. **Vertical slice**: one end-to-end path (one endpoint, one command) per PR, each shippable behind a flag if incomplete.

A **bad** cut leaves either half broken or meaningless: splitting an interface from its only implementation, or tests from the code they test. The test of a clean cut line: **each PR is green on its own, revertible on its own, and describable in one honest "What & why" paragraph.** If PR 1's description can only say "preparation for PR 2", the cut is in the wrong place — move code into it until it has a standalone claim (even if that claim is just "pure refactor, no behavior change").

## Honesty rules

- If tests weren't run, write "Not run" — do not imply coverage that doesn't exist.
- If part of the diff is unrelated drive-by cleanup, label it; better yet, suggest splitting it out.
- Flag anything in the diff YOU find questionable — reviewers trust descriptions that self-report weaknesses.
- Description length scales with risk, not diff size: a 20-line schema change deserves more words than a 500-line rename.

## Output

Write the description to the PR body if asked (`gh pr create/edit`), otherwise print it in a fenced block ready to paste.

## Anti-patterns (hard no)

- Describing a file's changes from its name in the diffstat without reading its hunks.
- "Tests pass" with no command and no summary line pasted.
- A risk section that says "low risk" for anything matching the schema/auth/concurrency rows of the rubric — those rows are never "low risk", they are "here is the specific evidence".
- Burying a behavior change inside a PR titled as a refactor.
- A review path that just lists the files in diff order — that's the archaeology you were supposed to save the reviewer from.
- Writing the description before the final push, then not re-checking that late commits kept it true.
