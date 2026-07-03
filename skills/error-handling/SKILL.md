---
name: error-handling
description: Design an error strategy for a module or service instead of sprinkling try/catch. Use when adding error handling, wrapping a flaky dependency, "make this robust", "handle failures here", reviewing code with catch blocks, or designing retries/timeouts. Refuses to write handlers until every failure source in the diff is classified into a taxonomy with one designated pattern per class.
---

# Error Handling

The failure mode: error handling grows call-site by call-site — a try/catch here because a test flaked, a retry there because prod paged — until the module has five inconsistent policies and no strategy. Errors are a design surface, same as the API. The gate: **you may not write a catch/rescue/match-on-Err block until every failure source in the diff is classified, and each class has exactly one designated handling pattern.**

## Phase 1 — Inventory and classify (gate)

Walk the diff (or module). List every operation that can fail: I/O, network calls, parsing, arithmetic on external data, lock acquisition, subprocess exits. Classify each into exactly one of four classes and write the table down before any handler code:

```
FAILURE INVENTORY — payment_worker.py
| Source                          | Class                | Policy (from taxonomy) |
| json.loads(webhook_body)        | invalid-input        | reject-with-identity   |
| stripe.Charge.create(...)       | retryable-transient  | retry-then-surface     |
| stripe: card_declined           | permanent-external   | surface, no retry      |
| db.save(receipt)                | retryable-transient  | retry-then-crash       |
| KeyError on our own config dict | bug                  | crash loudly           |
```

If two sources in the same class get different treatment, either the classification is wrong or you're inventing policy at the call site — both are gate failures.

## Taxonomy — per-class policy

| Class | Definition | Retry? | Surface to caller/user? | Page someone? | Process outcome |
|---|---|---|---|---|---|
| **invalid-input** | Caller/external party sent garbage; re-sending the same input fails identically | Never | Yes — with the identity of the bad input and what rule it broke | No (it's their bug) | Reject the request, keep serving |
| **retryable-transient** | The operation might succeed if repeated: timeouts, 429/503, lock contention, connection reset | Yes — backoff + jitter, bounded (cookbook below) | Only after budget exhausted, as "temporarily unavailable, retryable" | If exhaustion rate breaches threshold | Continue |
| **permanent-external** | Dependency answered definitively no: 401 bad creds, card declined, quota hard-cap, 404 on required resource | Never (retrying a decline is how you get fined) | Yes — translated to domain terms | Creds/config failures: yes, that's ops | Continue, or refuse startup if it's a boot dependency |
| **bug** | Our invariant broke: impossible enum value, negative balance, missing "guaranteed" key | Never | Generic 500/crash — never leak internals | Yes | Crash (or fail the unit of work) loudly and early — a bug handled "gracefully" runs on corrupted state |

The classes are about **what the correct reaction is**, not where the error came from. The same Stripe SDK exception type may map to two classes (429 → transient, card_declined → permanent); classify by semantics, not by exception type alone.

## Per-ecosystem idiom table

| Ecosystem | Idiom | Use when |
|---|---|---|
| **Go** | Wrap with context at each hop: `fmt.Errorf("charging invoice %s: %w", inv.ID, err)`. Branch with `errors.Is(err, ErrNotFound)` / `errors.As(err, &apiErr)`. Define sentinel errors / typed errors at package boundaries | Always. Naked `return err` from a non-trivial function loses the trail; `err.Error()` string matching is a firing offense |
| **Rust** | `thiserror` for library/module error enums (callers match on variants); `anyhow` + `.context("charging invoice inv_8XkQ")` in application/binary code | `thiserror` when callers must branch per-variant; `anyhow` when errors only travel up to be reported. The boundary between them IS your module boundary |
| **Python** | One module-level hierarchy: `class PaymentError(Exception)` → `CardDeclined`, `GatewayTimeout(PaymentError, RetryableError)`. Raise with `raise CardDeclined(...) from e` to keep the chain | Design the hierarchy in Phase 1 so callers can `except PaymentError` at the boundary instead of `except Exception`. Never `except:` bare; never swallow `KeyboardInterrupt`/`SystemExit` |
| **TypeScript** | Discriminated-union results for *expected domain outcomes*: `type ChargeResult = {ok:true; receipt:Receipt} | {ok:false; reason:'card_declined'|'insufficient_funds'}`. `throw` only for bug-class and infra failures | Unions force exhaustive `switch` handling at compile time — use them where "failure" is a normal business outcome. Exceptions for the truly exceptional; `catch (e: unknown)` and narrow, never `catch (e: any)` |

## Phase 2 — Boundary placement

Handlers live at boundaries; interior code propagates. Deciding *where* each class's designated pattern executes is half the design:

| Layer | Role | What it may do with errors |
|---|---|---|
| **Edge** (HTTP handler, queue consumer, CLI `main`, cron entry) | The only place errors are *finally* handled | Translate class → response/exit code/nack; log ONCE with full context; decide retry-by-redelivery vs reject |
| **Interior** (services, domain logic) | Propagate, adding context per hop | Wrap with identity (`invoice inv_8XkQ`); convert dependency errors into the module's own error types at the module boundary; never log-and-swallow |
| **Adapter** (DB client wrapper, HTTP client wrapper) | Classify | Map raw errors (`OperationalError`, status 429) into taxonomy classes; own the retry loop, because retry policy is a property of the dependency, not of every caller |

Two rules fall out of this table. First, **log at the edge, wrap in the interior** — if every layer logs, one failure produces five log entries and the on-call engineer can't tell one incident from five. A catch block that logs and rethrows without adding new context is noise; either add the identity this layer knows (which invoice, which attempt, which shard) or don't touch the error. Second, **retry in exactly one layer** — the adapter that owns the idempotency mechanism — because stacked retry loops multiply (see cookbook).

Timeouts are part of placement: every network call gets an explicit deadline (no library default is trustworthy — many are infinite), and the timeout value must be *shorter* at inner layers than outer ones (inner 2s, edge 10s), otherwise the edge gives up while the interior is still retrying and the work is wasted or, worse, completes unobserved. A timeout is the canonical producer of the retryable-transient class — but remember it means "unknown outcome," not "didn't happen."

## Retry cookbook (retryable-transient class only)

- **Formula**: `delay = min(cap, base * 2^attempt) * random(0.5, 1.0)` — full-ish jitter. Typical: `base=200ms, cap=30s, attempts=4`. Without jitter, every client that failed together retries together (thundering herd).
- **Idempotency precondition — non-negotiable**: retry only operations that are idempotent by nature (GET, PUT-with-full-body) or made so (POST with an `Idempotency-Key`, INSERT with a unique constraint + ON CONFLICT). A timeout does NOT mean the operation didn't happen; retrying a non-idempotent charge after a timeout is how customers get billed twice.
- **Retry budget**: cap retries as a fraction of total traffic (e.g., retries ≤ 10% of requests, circuit-break past that). Per-request retries multiply across layers: client×3 → gateway×3 → service×3 = 27 attempts hitting a struggling database.
- **When retrying makes it worse**: the dependency is overloaded (your retries are the load); the error is permanent-external misclassified as transient; you're inside a caller that also retries (pick ONE layer to own retries — usually the outermost that has the idempotency key).

Reference shape (the parameters above, in ~10 lines — adapt, don't import a framework for this):

```python
def with_retry(op, retry_on, no_retry=(), attempts=4, base=0.2, cap=30.0):
    for i in range(attempts):
        try:
            return op()
        except no_retry:
            raise                                   # permanent-external: out immediately
        except retry_on as e:
            if i == attempts - 1:
                raise RetryExhausted(attempts=attempts) from e
            time.sleep(min(cap, base * 2**i) * random.uniform(0.5, 1.0))
```

Note what is deliberately absent: no retry on bare `Exception` (bug-class must escape on attempt one, not after 4 sleeps), and no logging inside the loop — the edge logs once, with `attempts` attached via the exception chain.

**Edge translation table** — the taxonomy class fixes the external representation; write it once per service and stop deciding per-endpoint:

| Class | HTTP edge | Queue consumer edge | CLI edge |
|---|---|---|---|
| invalid-input | 400/422 + machine-readable code + field details | Reject to DLQ immediately (redelivery cannot fix garbage) | Exit 2, usage-style message to stderr |
| retryable-transient (budget exhausted) | 503 + `Retry-After` | Nack for redelivery with delay; DLQ after N deliveries | Exit 75 (`EX_TEMPFAIL`), message says "retry may succeed" |
| permanent-external | 4xx/502 mapped to domain code (`card_declined` → 402) | DLQ with the classified reason attached | Exit 1 with the dependency's verdict |
| bug | 500, generic body, full detail to logs/tracing only | Nack once (maybe it was corruption in transit), then DLQ + alert | Exit 70 (`EX_SOFTWARE`), stack trace to stderr |

The queue column encodes the least-obvious rule: **invalid-input must not be retried by redelivery** — a poison message nacked in a loop burns the consumer forever. Classification is what tells the consumer to DLQ it on the first attempt.

## Error message content checklist

Every surfaced error and every log line at the edge carries, checkably:

- [ ] **Identity of the failing input**: the id, key, row number, or a safe prefix of the payload — never just the operation name
- [ ] **The expectation that broke**: "expected ISO-8601 date, got '2026-13-01'" — rule + observed value
- [ ] **The taxonomy class's dispatch info**: retryable or not; for user-facing errors, what the user can do about it
- [ ] **Correlation**: request/trace id so the edge log line joins to the interior wrap chain
- [ ] **Nothing secret**: no tokens, no full card numbers, no raw payload dumps of PII — identity means *reference*, not contents

## Phase 3 — Worked example

Before — one function, three failure modes, zero strategy:

```python
def process_webhook(body: bytes):
    try:
        event = json.loads(body)                     # invalid-input
        charge = stripe.Charge.create(               # transient OR permanent
            amount=event["amount"], idempotency_key=event["id"])
        db.save_receipt(charge)                      # transient
    except Exception as e:
        logger.error(f"webhook failed: {e}")         # which webhook? retriable?
        return {"ok": False}                         # caller can't tell garbage from outage
```

Every real problem is here: catch-log-continue means a DB blip silently drops a paid charge's receipt (money moved, no record — corrupted state); the caller gets the same `{"ok": False}` for a malformed body (never retry) and a Stripe timeout (must retry); the log line has no event id.

After — one pattern per class, boundary translation at the top:

```python
def process_webhook(body: bytes) -> WebhookResult:
    try:
        event = parse_event(body)          # raises InvalidWebhook(payload_prefix=body[:64])
    except InvalidWebhook as e:
        return WebhookResult.reject(e)     # invalid-input: 400, no retry, identity attached

    charge = with_retry(                   # transient: backoff+jitter, budget=4
        lambda: stripe.Charge.create(amount=event.amount,
                                     idempotency_key=event.id),  # retry precondition
        retry_on=(RateLimitError, APIConnectionError),
        no_retry=(CardError,))             # permanent-external falls through
    # CardError propagates to the boundary handler → domain error, no retry, not paged

    with_retry(lambda: db.save_receipt(charge), retry_on=(OperationalError,),
               on_exhaust=raise_)          # transient→bug promotion: charge exists,
                                           # receipt doesn't → crash; the queue redelivers
                                           # and idempotency_key makes redelivery safe
    return WebhookResult.ok(event.id)
```

Note the last line's design move: "retries exhausted after money moved" is *promoted to crash* because continuing would run with corrupted state, and crash+redeliver is safe **only because** the idempotency key exists. That coupling is the strategy.

## Output contract

Deliver: (1) the failure-inventory table, (2) the per-class pattern chosen (with ecosystem idiom named), (3) the retry parameters and their idempotency justification, (4) the code, (5) one sentence per catch block saying which class it handles. A catch block you can't assign to a class gets deleted.

## Anti-patterns (hard no)

- Catch-log-continue past a failure that leaves state half-written. Log-and-continue is only legal when the operation is provably complete-or-untouched.
- Retrying any operation without stating why the retry is safe (idempotent by nature, or key/constraint shown).
- `catch (e) { log("operation failed"); rethrow e; }` — log-and-rethrow without adding context (which invoice? which attempt?) guarantees the same error is logged N times with no new information. Add context or don't touch it.
- Error messages/log lines without the failing input's identity. "Invalid date format" is useless; "invalid date '2026-13-01' in row 4823 of upload_7Kx.csv" is actionable.
- One generic `AppError`/`CustomException` for everything — callers are forced to string-match messages, which is the same as no error types.
- Designing handlers before the inventory table exists.
