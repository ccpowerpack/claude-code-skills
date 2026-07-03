---
name: test-first
description: Enforced test-driven development — write the failing test, run it to prove it fails, then implement until green. Use when the user asks to build a feature "with TDD", "test-first", or wants high-confidence implementation of well-specified behavior.
---

# Test First

You are implementing with strict TDD. The discipline exists for one reason: **a test you never saw fail proves nothing.** A green suite where every test was born green is indistinguishable from a suite of tautologies.

## The loop

For each behavior, in order:

### 1. Red (gate: you must observe the failure)

- Write ONE test for the next smallest behavior. Name it after the behavior, not the method: `test_expired_token_is_rejected`, not `test_validate_2`.
- Run it. **Paste the failure output.**
- Verify it fails for the RIGHT reason — an assertion failure on the behavior, not an import error, syntax error, or missing fixture. `ModuleNotFoundError` is not a red test; it's a broken test. Fix the scaffolding first and re-run until the failure is the assertion you wrote.
- If the test passes before you've implemented anything, the test is vacuous. Rewrite it.

### 2. Green (gate: passing output for this test AND the suite)

- Write the minimum implementation that makes this test pass. Resist implementing ahead of the tests — if you catch yourself writing a branch no test forces, delete it or write the test first.
- Run the test. Paste the passing output.
- Run the full relevant suite — you may not proceed with other tests newly broken.

### 3. Refactor

- With green tests as a safety net: remove duplication, improve names, extract structure that has earned its existence.
- Run the suite again after refactoring. Refactor step ends green or gets reverted.

Repeat until the feature's behaviors are covered.

## Choosing the next test

Before starting, list the planned test cases as a checklist and keep it updated so the user sees progress. Order them to maximize learning per test:

| Behavior class | When to test it | Example test name |
|---|---|---|
| Happy path, simplest input | First — forces the API shape into existence | `test_first_request_passes_through` |
| Core invariant | Second — the reason the feature exists | `test_duplicate_key_returns_cached_response` |
| Boundary: empty / zero / absent | After the invariant holds | `test_request_without_key_is_not_deduplicated` |
| Boundary: max / overflow / unicode | Same pass as other boundaries | `test_key_longer_than_255_chars_is_rejected` |
| Error path: invalid input | After boundaries — needs the validation seam | `test_malformed_key_returns_400_not_500` |
| Error path: dependency failure | Late — needs the fake/mock seam to exist | `test_store_outage_fails_open_not_closed` |
| Concurrency / interaction | Last — hardest to write, needs stable internals | `test_concurrent_same_key_executes_handler_once` |
| Time-dependent behavior | Last, with a fake clock (see below) | `test_key_expires_after_ttl` |

Skip a row only if it genuinely doesn't apply — say so in the checklist rather than silently omitting it. If mid-loop you discover a behavior the plan missed, add it to the checklist visibly before writing its test — the checklist is a living contract, not a prologue.

## Prove-the-red commands per ecosystem

Run ONE test fast; the tight loop dies if every red costs a full-suite run.

| Ecosystem | Run one test | Notes |
|---|---|---|
| pytest | `pytest tests/test_idem.py::test_key_expires_after_ttl -x -q` | `-k "expires"` for substring match; `-x` stops at first failure |
| jest | `npx jest idem.test.ts -t "expires after ttl"` | `-t` matches the test title; add `--watch` for the loop |
| vitest | `npx vitest run idem.test.ts -t "expires after ttl"` | `vitest` without `run` starts watch mode — ideal for red-green |
| go test | `go test ./middleware/ -run 'TestKeyExpiresAfterTTL$' -v -count=1` | `-count=1` defeats the test cache — cached PASS is not evidence |
| cargo | `cargo test key_expires_after_ttl -- --exact --nocapture` | substring match by default; `--exact` pins one test |
| JUnit (Maven) | `mvn test -Dtest='IdempotencyTest#keyExpiresAfterTtl' -q` | Gradle: `./gradlew test --tests 'IdempotencyTest.keyExpiresAfterTtl'` |

Whatever the ecosystem: the red run and the green run must be the SAME command, pasted both times. If the runner has a test cache (go, gradle, turbo/nx-wrapped jest), disable it for the red/green pair — a cache hit reporting PASS is not an observation.

## Worked example — idempotency-key middleware, three cycles

Feature: HTTP middleware that deduplicates POSTs carrying an `Idempotency-Key` header.

**Cycle 1 — happy path.** Test: first request with a key passes through to the handler.

```
$ pytest tests/test_idem.py::test_first_request_passes_through -x -q
E   AttributeError: module 'app.middleware' has no attribute 'IdempotencyMiddleware'
```

Wrong-reason failure — scaffolding, not behavior. Create the empty class, re-run:

```
E   assert handler.call_count == 1
E    +  where 0 = handler.call_count
1 failed in 0.04s
```

Right reason. Implement pass-through. `1 passed in 0.05s`. Suite green.

**Cycle 2 — the invariant.** Test: second request with the same key returns the stored response and does NOT call the handler again.

```
$ pytest tests/test_idem.py::test_duplicate_key_returns_cached_response -x -q
E   assert handler.call_count == 1
E    +  where 2 = handler.call_count
1 failed in 0.05s
```

Implement: store `(status, body)` by key after the first call; short-circuit on hit. `2 passed in 0.06s`.

**Cycle 3 — boundary.** Test: request without the header is never deduplicated.

```
E   assert handler.call_count == 2
E    +  where 1 = handler.call_count
```

Interesting red: the naive implementation cached under key `None`, so two keyless requests collided. The test caught a real bug before it shipped. Fix: bypass the store entirely when the header is absent. `3 passed in 0.07s`.

**Refactor.** Extract `ResponseStore` from the middleware body — it has earned its existence: cycle 2 created it, cycle 3 gave it a second rule. Re-run: `3 passed in 0.07s`.

## Async and time-dependent code — no sleeps

`sleep(0.5)` in a test is a flake with a countdown timer. Two rules:

1. **Time is an injected dependency.** The code under test reads a clock you control; the test advances it deterministically.
2. **Wait for events, not durations.** If the test must wait for async work, wait on a completion signal (future, channel, `waitFor` on an observable condition) — never a fixed delay.

| Ecosystem | Fake clock | Async sync point |
|---|---|---|
| Python | `freezegun.freeze_time()` / `time-machine` | `pytest-asyncio`: `await` the coroutine directly; drive the loop, don't sleep |
| JS/TS (jest) | `jest.useFakeTimers()` + `jest.advanceTimersByTime(60_000)` | `await Promise.resolve()` to flush microtasks; `jest.runAllTimersAsync()` |
| JS/TS (vitest) | `vi.useFakeTimers()` + `vi.advanceTimersByTime()` | `await vi.runAllTimersAsync()` |
| Go | inject a `Clock` interface (`jonboulle/clockwork`, `benbjohnson/clock`) | channels / `sync.WaitGroup`; `t.Deadline()`-bounded selects, never `time.Sleep` |
| Rust (tokio) | `#[tokio::test(start_paused = true)]` + `tokio::time::advance(Duration::from_secs(60)).await` | `await` the task handle; paused time auto-advances past sleeps |
| JVM | `java.time.Clock` injected; `Clock.fixed(...)` in tests | Awaitility `await().until(...)` polls a condition, not a duration |

For the idempotency example: `test_key_expires_after_ttl` injects a fake clock, stores at T, advances the clock by `ttl + 1s`, asserts the handler runs again. Zero wall-clock waiting, zero flake surface.

If the codebase has existing sync utilities or a clock abstraction, use those — matching house convention beats importing a new library.

Where real time genuinely cannot be faked (integration test against a real broker, subprocess startup), poll a condition with a deadline — `await().atMost(5, SECONDS).until(queueIsEmpty)` — never a fixed delay. The deadline is a failure bound, not an expected duration: the test passes the instant the condition holds.

## Test quality bar

- Each test asserts one behavior; multiple asserts are fine if they describe one outcome.
- No sleeping/polling-by-duration anywhere — see the table above.
- Use real objects where cheap; mock only at process/network boundaries.
- The test must read as documentation: given / when / then visible in its structure.
- Test names form a spec when listed: `pytest --collect-only -q` output should read like the feature's behavior list.

## Anti-patterns (hard no)

- Writing the implementation first "to understand the problem", then backfilling tests.
- Batch-writing 10 tests then implementing everything at once — that's test-after with extra steps.
- Marking a test skipped/todo to get to green.
- Weakening an assertion to make it pass. If the assertion was wrong, say so explicitly and justify the change.
- Claiming red without pasted failure output, or pasting a red that failed on imports/fixtures rather than the assertion.
- `sleep()` in any test, including "just 100ms to be safe".

## Output contract

Finish with: the checklist of behaviors covered (each linked to its test name), final full-suite run output pasted, and any behavior you deliberately left untested (with reason).
