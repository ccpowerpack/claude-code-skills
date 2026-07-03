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
4. For intermittent bugs, "reproduce" means **a command that fails with known probability**, e.g. "fails 4/50 runs of `pytest tests/test_billing.py -x --count=50`". That is a valid gate pass; "it fails sometimes in CI" is not.

## Phase 2 — Isolate

Shrink the search space before reading code broadly. Pick the bisect axis deliberately:

### Bisect strategy decision table

| Axis | Use when | Technique | Gotchas |
|---|---|---|---|
| **Time** (which commit) | It worked before; a known-good ref exists | `git bisect start HEAD v2.3.0`, then `git bisect run ./repro.sh` — script exits 0 = good, 1–124 = bad, 125 = skip (unbuildable) | Flaky repro poisons bisect: make `repro.sh` retry N times and only exit 1 on consistent failure. Skip commits that don't build rather than guessing. |
| **Input** (which part of the data) | Bug depends on a specific payload/file/record | Delta debugging: cut the failing input in half, keep whichever half still fails, repeat to a minimal case. Property-test shrinkers (Hypothesis, fast-check) do this automatically — feed the failing case in. | Some bugs need *two* interacting parts; when neither half fails alone, bisect the *removals* instead (remove one half at a time). |
| **Layer** (which boundary) | Value is wrong at the surface; unknown where it went wrong | Probe each hand-off point and find the **first** boundary where the value is already wrong (see per-stack probes below). | Probe the raw wire/storage value, not the framework's convenient accessor — the accessor may be the bug. |

### Layer-probe techniques per stack

- **HTTP service**: hit the handler directly, bypassing infrastructure — `curl -sS http://localhost:8080/api/orders/42 | jq .total` vs. the same path through the LB/gateway. If they differ, the bug is in proxy/middleware config, not app code. Inside the app, log the parsed request at the first middleware and at the handler: if they differ, a middleware mutated it.
- **DB-backed app**: run the exact SQL with the exact parameters in `psql`/`mysql` directly. If the row is correct at rest but wrong in the app, the bug is in ORM mapping, type coercion, or caching — not the query. Turn on statement logging to get the real SQL (`SET log_statement = 'all'` per-session, or ORM echo mode).
- **Frontend**: three checkpoints — Network tab response body → store/state after the reducer/setter → props at the rendering component. First checkpoint that's wrong names the guilty layer (API, state management, or render logic). `console.log` in a selector is cheaper than reading five components.

State your current hypothesis explicitly before each probe: "If X is the cause, then probing at Y will show Z." A probe without a predicted outcome is tourism.

## Phase 3 — Instrument

- Add targeted logging/assertions at the suspected boundary — not scattershot prints.
- Prefer reading the actual runtime value over reasoning about what it "should" be.
- Remove every instrument you added before finishing (verify with `git diff`).

### Instrumentation cookbook — fastest way to see a runtime value

| Language | Quick inline | Live process, no restart |
|---|---|---|
| Python | `print(f"{order=}", file=sys.stderr, flush=True)` (`=` syntax prints name and repr) | `py-spy dump --pid <pid>` for stacks; `py-spy dump --pid <pid> --locals` for local variable values |
| Node | `console.error({ order })` — object shorthand keeps the name | `kill -USR1 <pid>` opens the inspector port; attach `chrome://inspect` or `node inspect -p <pid>` |
| Go | `fmt.Fprintf(os.Stderr, "order=%+v\n", order)` (`%+v` includes field names) | `kill -QUIT <pid>` dumps all goroutine stacks to stderr; `dlv attach <pid>` for values |
| JVM | `System.err.printf("order=%s%n", order)` — but prefer next column, redeploys are slow | `arthas` → `watch com.acme.OrderService total '{params, returnObj}'` prints live args/returns per call; `jstack <pid>` for stacks |
| Rust | `dbg!(&order)` — prints file:line, expression, and value to stderr; returns the value so it inlines anywhere | `rust-lldb -p <pid>`; for tracing-instrumented code, bump `RUST_LOG=my_crate=trace` |

Always `flush`/stderr, never buffered stdout — a crash eats buffered output and you'll chase a ghost ("the log line never printed, so it never ran" — no, it ran and the buffer died).

## Phase 3b — Heisenbugs: when instrumentation changes the behavior

If adding a print makes the bug disappear (or appear), you have a timing-sensitive bug — usually a data race, unsynchronized init, or event-ordering assumption. Naive instrumentation is now actively misleading. Do this instead:

1. **Observe from outside the process**: sampling profilers and tracers don't serialize your threads — `py-spy`, `perf`, `bpftrace`, `strace -f -e trace=futex` cost far less timing distortion than an inline `print` (which takes a lock on stderr and does sync I/O).
2. **Use the purpose-built detectors first**: `go test -race ./...`, C/C++/Rust ThreadSanitizer (`RUSTFLAGS="-Zsanitizer=thread"` on nightly, `-fsanitize=thread`), Java `-XX:+UnlockDiagnosticVMOptions` + jcstress for suspected JMM issues. A race detector finding is Phase 4 evidence on its own.
3. **Record/replay when available**: `rr record ./repro && rr replay` gives you a deterministic recording you can instrument freely — the recording's timing is frozen, so probes can't perturb it.
4. **Widen the race window instead of watching it**: if you hypothesize a race between A and B, inject a deliberate `sleep 50ms` inside the suspected window and show the failure rate jump from 4/50 to 50/50. That is hypothesis-driven and it's *evidence*: you found the window.
5. **Buffer in memory, dump at exit**: if you must log inline, append to a preallocated in-memory ring buffer and print it in a crash/exit handler — microseconds instead of milliseconds of distortion.
6. Constrain the scheduler to shift interleavings reproducibly: `taskset -c 0 ./repro` (one CPU) or `stress-ng --cpu 8` alongside (contention) — whichever makes the failure rate go *up* tells you about the required interleaving.

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
4. Re-run the original reproduction from Phase 1. For intermittent bugs, re-run at the same N: 0/50 failures after a 4/50 baseline is proof; 0/3 is not.

## Worked example — a flaky test, symptom to root cause

Symptom: `test_invoice_totals` fails ~1 in 10 CI runs with `AssertionError: 108.0 != 100.0`. Passes every time locally.

**Phase 1 — get a failing command.** Alone it always passes:

```
$ pytest tests/test_billing.py::test_invoice_totals --count=50 -q
50 passed in 4.1s
```

So it's not internally flaky — it's order-dependent. The CI runs use `pytest-randomly`. Grab the seed from a failing CI log (`Using --randomly-seed=1683021`) and replay it:

```
$ pytest tests/ -p randomly --randomly-seed=1683021 -q
....F....
FAILED tests/test_billing.py::test_invoice_totals - AssertionError: 108.0 != 100.0
```

Deterministic reproduction. Gate passed.

**Phase 2 — bisect by input (the "input" is the set of preceding tests).** The seeded order runs 212 tests before the failure. Halve the prefix:

```
$ pytest tests/test_tax.py tests/test_billing.py::test_invoice_totals -q
.F     # still fails — pollution is somewhere in test_tax.py
$ pytest tests/test_tax.py::test_eu_vat_applied tests/test_billing.py::test_invoice_totals -q
.F     # minimal pair found
```

**Phase 3 — instrument the boundary.** 108.0 = 100 × 1.08, which smells like a tax rate leaking across tests. Probe the value the billing code actually reads:

```python
# billing/rates.py, inside get_tax_rate()
print(f"{get_tax_rate.cache_info()=} {_cached_rate=}", file=sys.stderr, flush=True)
```

```
get_tax_rate.cache_info()=CacheInfo(hits=1, misses=1, ...) _cached_rate=0.08
```

`get_tax_rate` is wrapped in `functools.lru_cache`. `test_eu_vat_applied` monkeypatches the settings, calls it (cache stores 0.08), and monkeypatch teardown restores settings — but never clears the cache.

**Phase 4 — gate.**

```
ROOT CAUSE: functools.lru_cache on billing/rates.py:14 get_tax_rate() caches a
  monkeypatched value across tests; any test that computes tax after
  test_eu_vat_applied in the same process reads 0.08 instead of 0.0.
EVIDENCE: billing/rates.py:14 — cache_info() shows the hit, _cached_rate=0.08
  observed inside the failing run; failure reproduces 100% with the minimal
  pair and 0% with `get_tax_rate.cache_clear()` inserted between them.
WHY IT ESCAPED: only surfaces under randomized test order when the two tests
  share a process; local default order runs billing before tax.
```

**Phase 5.** Fix at the causal site: an autouse fixture calling `get_tax_rate.cache_clear()` (test isolation was the broken contract — the cache itself is fine in prod). Regression test: run the minimal pair as an explicit ordered test. Re-run the seeded full suite: passes. Remove the probe print, confirm with `git diff`.

## Output contract

Finish with: root cause statement, the fix diff summary, proof (test output before/after — same N for intermittent bugs), and anything suspicious you noticed but didn't fix.

## Anti-patterns (hard no)

- "This might fix it" commits.
- Adding try/catch or null-checks to silence a symptom whose cause you haven't found.
- Fixing the test instead of the code without justifying why the test was wrong.
- Declaring victory without re-running the original reproduction.
- Adding retries or sleeps to a flaky test instead of finding the shared state or race that makes it flaky.
- Leaving instrumentation in the final diff (a `dbg!`, a stray `console.error`, a lowered log level).
