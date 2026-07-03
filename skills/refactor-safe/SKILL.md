---
name: refactor-safe
description: Behavior-preserving refactoring with proof. Pins current behavior in characterization tests BEFORE touching structure, then proves the SAME tests pass unmodified after. Use for "refactor this", "clean up / restructure / extract / rename this without changing behavior", "split this function", "untangle this module", or any change that claims "no behavior change". Halts and re-scopes the moment a refactor turns out to alter behavior.
---

# Refactor Safe

You are performing a refactor: changing structure while keeping behavior identical. The cardinal rule: **you may not change structure until current behavior is pinned by tests you have run and shown passing — and after the refactor, those SAME tests must pass with their assertions unmodified.** A test you had to edit to make green is proof the behavior changed. That's not automatically wrong, but it is *loud*: it must be surfaced and justified, never quietly absorbed.

The failure this defends against: refactor and behavior change smuggled into one diff, where "I improved it" hides "I changed what it does" and no one can tell which lines did which.

## Phase 1 — Pin behavior (gate: paste these tests passing before any structural edit)

Before touching the code, capture what it currently does — bugs and all. Characterization tests describe *actual* behavior, not desired behavior.

1. Identify the observable surface of the code under refactor: return values, thrown errors, side effects (writes, calls, emitted events), for representative + edge + degenerate inputs.
2. Write characterization tests against the **current** implementation. If a test surprises you (asserts something you think is a bug), keep it — you're pinning reality, not fixing it. Note the suspected bug separately.
3. Run them. **Paste the passing output.** This is the gate. Green output against the unmodified code is the baseline you will hold the refactor to.

If the code is too tangled to test at the unit level, pin it at the next boundary out (HTTP response, CLI stdout, golden file) — a coarse pin that runs beats a fine pin that doesn't exist.

**Coverage honesty (don't pin air):** a green char-test suite is worthless if it doesn't execute the lines you're about to change. Measure it: run coverage over the region under refactor (`pytest --cov=<module>`, `go test -cover`, `c8`/`nyc`, `cargo llvm-cov`) and confirm the branches you're touching are actually hit. An uncovered branch is an unpinned branch — add a case or declare it explicitly unproven before you refactor it.

## Phase 2 — Classify the refactor and pick tooling

Match the change to its mechanical risk and let tooling do the deterministic parts.

| Refactor type | Mechanical risk | Prefer |
|---|---|---|
| Rename symbol | Low — if scope-aware | LSP/IDE "rename symbol" (semantic, not text sed); never `grep`+replace across a codebase — it hits strings, comments, unrelated same-named symbols |
| Extract function/method | Medium — closed-over state, early returns, mutation | IDE "extract method" where available; otherwise extract by hand and lean hard on char-tests |
| Inline | Medium — duplicated side effects, evaluation order | IDE "inline"; verify call sites don't rely on the indirection |
| Move (file/module/package) | Medium — import cycles, visibility changes | IDE move refactor; run the build to catch broken imports |
| Change signature (params/return) | High — every caller, defaults, overloads | Codemod (`jscodeshift`, `comby`, `gofmt -r`, `ast-grep`) for callers; char-tests at each call site |
| Restructure control flow | High — the whole point is subtle | Hand work, maximal char-test coverage, small steps |

Prefer many tiny structural steps, each green, over one big leap. A green bar every few minutes localizes any regression to the last step.

**Semantic tooling per ecosystem** (use these instead of text search-replace):

| Language | Scope-aware rename | Codemod / signature change |
|---|---|---|
| TS/JS | `tsserver` rename (VS Code F2), `ts-morph` | `jscodeshift`, `ast-grep`, `comby` |
| Python | Pyright/`rope` rename, `pylsp` | `libcst` codemods, `bowler`, `ast-grep` |
| Go | `gopls` rename | `gofmt -r 'a(b) -> b.a()'`, `eg`, `ast-grep` |
| Java | IDE (IntelliJ/Eclipse) refactor | OpenRewrite recipes, `error-prone` refaster |
| Rust | `rust-analyzer` rename | `ast-grep`, hand + `cargo check` |

**Where semantic rename still lies:** symbols referenced by *string*, not by the type system — reflection, string-based DI/wiring, ORM column mappings, serialized field names (JSON/protobuf), config keys, route strings, DB migration names, and public API consumed outside this repo. The LSP renames the code and silently misses the string. After any rename, grep the *old* name across the whole tree (`grep -rn "oldName"`) to catch the string references the tool couldn't see — a hit there is a behavior change hiding in a "mechanical" rename.

## Phase 3 — Characterization-test cookbook

Cheap ways to pin behavior when hand-writing assertions is too slow:

- **Golden / snapshot files**: capture current output (JSON, rendered text, serialized struct) to a file; test asserts equality. `pytest --snapshot-update` (syrupy), Jest `toMatchSnapshot`, `cupaloy`/`goldie` in Go, `insta` in Rust. Generate against old code; freeze; the refactor must reproduce byte-for-byte.
- **Property tests old-vs-new**: keep the old implementation temporarily as `oldFn`; assert `newFn(x) == oldFn(x)` over generated inputs (Hypothesis, fast-check, quickcheck, gopter). This is the strongest pin for pure functions — it searches the input space for a divergence you didn't think of. Delete `oldFn` once green.
- **Record/replay for side effects**: capture the sequence of outbound calls (DB queries, HTTP, emitted events) with a spy/mock; assert the same sequence before and after.
- **Approval on real traffic**: for a hairy transform, run a sample of production inputs through old and new, diff the outputs.

**Checking the pin is actually strong**: if the refactor is high-stakes, run a mutation tester (`mutmut`/`cosmic-ray` for Python, StrykerJS for JS/TS, `go-mutesting`, `cargo-mutants`) against the region *before* refactoring. Surviving mutants = behavior your char-tests don't pin. Kill them or accept the gap explicitly. This turns "the tests pass" into "the tests would notice if behavior changed."

**Introducing a test seam when the code resists testing** — if behavior can't be observed because a dependency is hardwired (real clock, network, global singleton), add the *smallest* seam that makes it observable, and make that seam its own labeled, behavior-preserving step verified by the coarse boundary test above. Techniques: extract the hardwired call behind a parameter with a default equal to the current value (so existing callers are unchanged), or subclass-and-override the one method in a test double. The seam itself must not change behavior — prove it with the outer pin before you use it to write finer pins.

## Phase 4 — Refactor, then prove (gate)

1. Apply the structural change (one type at a time from Phase 2).
2. Re-run the Phase 1 tests **without touching their assertions**. Paste the passing output.
3. Diff the tests: `git diff -- <test files>` must show **no assertion changes**. If it does, you did not do a behavior-preserving refactor — go to Stop conditions.
4. Run the broader suite + build/typecheck.

The proof is two pasted test runs (before, after) plus a clean `git diff` on the test assertions.

Never batch multiple Phase 2 refactor types into one unverified leap — a rename plus an extract plus a signature change applied together means a red bar can't tell you which one broke. One type, re-run, green, next.

## Stop conditions — halt and re-scope

The moment any of these is true, stop refactoring and report; do not push through:

- A char-test can only pass if you edit its assertion → the change alters behavior. Split it: land the behavior change as its own diff with its own justification, separate from the refactor.
- You discover the pinned behavior is a bug you must fix now → that's a behavior change, not a refactor. Re-scope, tell the user, fix it in a labeled diff.
- The refactor requires changing a public contract (signature, error type, event shape) callers depend on → that's an API change; surface it.
- You cannot pin the behavior at any boundary → you cannot prove preservation; say so before touching structure rather than refactoring blind.

## Worked example — extracting a gnarly function

Target: a 60-line `computeInvoiceTotal(order)` with nested discount/tax/rounding logic. Goal: extract `applyDiscounts` and `applyTax` without changing a cent.

Phase 1 — pin with a property test against the current code, plus goldens for known cases:

```python
# test_invoice_char.py — against ORIGINAL computeInvoiceTotal
@given(order=orders())                       # hypothesis strategy
def test_total_matches_recorded(order):
    assert computeInvoiceTotal(order) == _recorded_total(order)   # golden

def test_edge_zero_and_rounding():
    assert computeInvoiceTotal(ZERO_ORDER) == Decimal("0.00")
    assert computeInvoiceTotal(THREE_ITEMS_ROUNDING) == Decimal("19.99")
```

```
$ pytest test_invoice_char.py -q
...                                                                  [100%]
3 passed in 0.42s
```

Gate passed. Now extract `applyDiscounts` / `applyTax`, keeping arithmetic and order of operations identical.

Phase 4 — same tests, assertions untouched:

```
$ pytest test_invoice_char.py -q
...                                                                  [100%]
3 passed in 0.40s
$ git diff -- test_invoice_char.py
(no output)
```

Clean diff on the tests + green = behavior preserved. The extraction is safe to land.

## Worked example — a stop condition firing mid-refactor

Target: "inline the `normalizeEmail` helper into its two call sites" — labeled a pure refactor.

Phase 1 pins the current behavior, including this surprising char-test kept from reality:

```python
def test_normalize_preserves_plus_tag():
    assert normalizeEmail("A.b+tag@Gmail.com") == "a.b+tag@gmail.com"  # lowercases, keeps +tag
```

While inlining, you notice call site 2 *also* strips `+tag` before calling — meaning inlining changes what site 2 produces (it would now keep the tag). The char-test for site 2's output can only pass if you edit its assertion.

**Stop.** This is not behavior-preserving — the two sites were never equivalent, so "inline" is a merge that changes one of them. Re-scope: report it, and split into (a) a labeled behavior-change diff deciding whether `+tag` should be stripped, with its own test, and (b) the mechanical inline, only once both sites truly agree. Pushing through by editing the assertion would have silently altered email routing.

## Output contract

Deliver: (1) the characterization tests and their passing output against the pre-refactor code; (2) the refactor diff; (3) the SAME tests passing post-refactor with `git diff` on the test files showing no assertion edits; (4) the full suite + build/typecheck green, plus confirmation the production diff touches only structure (moved/renamed/extracted code), no logic; (5) any stop-condition you hit and how you re-scoped it. If any assertion changed, lead with that and its justification.

## Commit hygiene

Land the refactor as its own commit(s), separate from any behavior change, so a reviewer (and `git bisect`) can trust that a "refactor" commit cannot be the source of a behavior regression. If characterization tests were net-new, they go in a commit *before* the refactor. A diff that mixes "moved code" with "changed logic" forces the reviewer to read every line to tell which is which — the whole point of proving preservation was to spare them that.

## Anti-patterns (hard no)

- Refactor and behavior change in the same diff without labeling which lines do which.
- Editing a test assertion mid-refactor to make the bar green.
- "The test was wrong anyway" without evidence that it asserted something the code never actually did.
- Structural edits before any behavior is pinned — "I'll add tests after" defeats the entire method.
- `grep`+`sed` rename across the tree instead of a scope-aware rename (hits strings, comments, unrelated symbols).
- Declaring the refactor safe without pasting both the before and after test runs.
