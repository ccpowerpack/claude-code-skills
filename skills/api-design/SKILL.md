---
name: api-design
description: Design or review an HTTP/RPC API surface before any handler code exists — resource model, error contract, pagination, versioning, batch semantics. Use when asked to "design an API", "add an endpoint", "review this API", "what should this route return", or before implementing any new public interface. Refuses to write handlers until the contract survives an alternatives comparison and every endpoint has its failure responses designed.
---

# API Design

The failure mode this skill defends against: the first resource shape that comes to mind gets implemented, happy paths only, storage schema leaking straight into JSON. It works in the demo, then every consumer hard-codes the accidents, and the shape is frozen forever. An API is the one artifact where "refactor later" is not available — so the design work happens **before** handler code, gated on written evidence.

## Phase 1 — Resource model and alternatives (gate)

You may not write handler code, route registrations, or OpenAPI stubs until you have written down:

1. **The resource model**: nouns, their identifiers, their relationships, which are addressable (`GET`-able by id) vs. embedded.
2. **At least one alternative shape** you considered and rejected, with the concrete reason. "I considered nesting `POST /users/{id}/invites` vs. top-level `POST /invites` with a `user_id` field; rejected nesting because invites will later be creatable in bulk across users." One sentence of hand-waving does not pass; the rejection must name a future request pattern or evolution pressure.
3. **The error contract** (Phase 2 template, filled) — before endpoints, because endpoints inherit it.

The written resource model looks like this — short, but it must exist before code:

```
RESOURCE MODEL — invitations API
invite      id: inv_*   addressable (GET /invites/{id})
            belongs-to: workspace (ws_*), created-by: user (usr_*)
            lifecycle: pending → accepted | revoked | expired   (one-way; no un-accept)
            embedded in responses: inviter {id, display_name} — NOT the full user

REJECTED ALTERNATIVE: POST /workspaces/{id}/invites (nested creation).
Rejected because admin tooling needs cross-workspace bulk invite next quarter;
top-level POST /invites with workspace_id in the body serves both.
```

The lifecycle line matters most: it dictates the 409 table, and it is the part naive designs skip. If the caller gave you a database table and asked for "CRUD on this," that is the trigger to slow down, not speed up: the table's shape is the storage's business, not the client's.

## Phase 2 — Error contract (gate: every endpoint, not just 200s)

Define once per API, then instantiate per endpoint. Template:

```json
{
  "error": {
    "code": "invite_already_accepted",     // stable, machine-readable, snake_case, NEVER renamed
    "message": "Invite inv_8XkQ was accepted on 2026-06-30.",  // human, may change freely
    "retryable": false,                    // client's single dispatch bit
    "details": { "invite_id": "inv_8XkQ", "accepted_at": "2026-06-30T14:02:11Z" }
  }
}
```

Rules:
- `code` is the contract. Clients switch on it. HTTP status is the coarse routing layer (4xx vs 5xx), `code` is the fine one. Never make clients parse `message`.
- `retryable` must be explicit. A 503 with `retryable: false` (e.g., dependency permanently gone) and a 409 with `retryable: true` (optimistic-lock conflict, retry with fresh state) both exist.
- `details` carries the identity of the failing input. An error a client can't correlate to its own request is a log line, not an API response.

Filled examples that must exist for every mutating endpoint before you proceed:

| Situation | Status | `code` | `retryable` |
|---|---|---|---|
| Body fails validation | 400 | `validation_failed` (+ per-field `details.errors[]`) | false |
| Resource id unknown | 404 | `invite_not_found` | false |
| State machine violation | 409 | `invite_already_accepted` | false |
| Version/etag conflict | 409 | `conflict_stale_version` | true (re-read, retry) |
| Rate limited | 429 | `rate_limited` (+ `details.retry_after_s`) | true |
| Downstream timeout | 504 | `upstream_timeout` | true |

An endpoint whose design lists only its 200 response is incomplete. Write the failure table first — it usually changes the success shape (e.g., 409 needs an `accepted_at` to report, so you must store it).

## Decision table — the contested choices

Do not re-litigate these from scratch; pick from the table and record which and why.

| Choice | Option A | Option B | Pick A when | Pick B when |
|---|---|---|---|---|
| Update semantics | `PUT` (full replace; client sends complete resource; omitted field = deleted) | `PATCH` (partial; omitted = unchanged; use JSON Merge Patch RFC 7396 or explicit field list) | Resource is small and clients always hold the whole thing | Resource has server-managed or large fields; concurrent editors touch different fields |
| Pagination | Offset (`?page=3&per_page=50`) | Cursor (`?cursor=eyJpZCI6...` opaque token) | Data is small/static, users need "jump to page N", total count is cheap | Data mutates under the reader (feeds, logs), tables are large (OFFSET is O(n) scan), you ever want to change sort keys without breaking clients |
| Unsafe retries | Nothing (client prays) | `Idempotency-Key` header: server stores key→response for 24h, replays the stored response on retry | Never — every non-idempotent mutation reachable over a network needs this | Payments, order creation, anything where duplicate execution costs money. Return `409 idempotency_key_reused` if same key arrives with a different body |
| Batch endpoints | Atomic (all-or-nothing, single transaction) | Partial success: `207`-style body with per-item `{index, status, error?}` | Items are interdependent (order + its lines) | Items are independent (bulk invite 500 users); one bad email must not fail 499 good ones. Design the per-item error shape NOW — it reuses the Phase 2 contract |
| Versioning | URL path `/v2/` | Additive evolution + tolerant readers, version only as last resort | You are knowingly breaking the contract and will run both versions (real cost: double the surface, double the bugs, migration deadline enforcement) | Default. Most "v2 needs" are actually "add an optional field" |
| Long-running ops | Synchronous (hold the connection until done) | `202 Accepted` + operation resource: `POST /exports` → `202 {"operation_id":"op_3fk"}`, then `GET /operations/op_3fk` → `{status:"running"|"succeeded"|"failed", result_url?, error?}` | p99 completes well under client/gateway timeouts (< ~10s) and will stay there | Work can exceed timeouts, needs progress reporting, or the client may disconnect. Retrofitting async onto a sync endpoint is a breaking change — decide now |
| Embedding related data | Client makes N follow-up `GET`s | `?expand=customer,items` opt-in expansion of referenced resources | Relations are rarely needed or huge | Common read path needs them (dashboard lists); opt-in keeps the default payload small and the expansion set is a documented, bounded list — not arbitrary graph traversal |

**Surface-consistency checklist** (a good decision applied inconsistently is a new bug class — verify across ALL endpoints before Phase 3):

- [ ] One casing convention for field names (pick `snake_case` or `camelCase`, never mix)
- [ ] Timestamps: RFC 3339 UTC strings everywhere (`2026-07-03T09:14:00Z`), never epoch-in-some-fields
- [ ] Ids: prefixed opaque strings (`inv_8XkQ`) — greppable in logs, type-confusable integers are not
- [ ] Money: integer minor units + currency code (`{"amount": 4999, "currency": "EUR"}`), never floats
- [ ] Every list endpoint paginates from day one with the same envelope (`items` + `next_cursor`) — retrofitting pagination onto a bare-array response is breaking
- [ ] Null vs absent means the same thing on every endpoint (document which)

## Phase 3 — Worked example

Naive design that arrives constantly:

```
GET /api/messages?user=42          → 200 [ {id, user, txt, ts, read: true}, ... ]
POST /api/messages/mark_read       → 200 {ok: true}   body: {ids: [1,2,3]}
```

Three concrete problems under load and evolution:

1. **Unbounded list.** No pagination. At 200k messages this response is 40MB; adding `?page=` later breaks every client that assumed completeness. Offset pagination would also skip/duplicate rows as new messages arrive mid-scroll — this is a feed, cursor is forced.
2. **`read: true` is a boolean that will become an enum.** The roadmap already has "delivered" receipts. Shipping `read: bool` means v2 pain; shipping `status: "read"` (`sent|delivered|read`) costs nothing today. Any boolean encoding a lifecycle stage gets this treatment.
3. **`mark_read` is an RPC verb with all-or-nothing ambiguity.** If id 2 belongs to another user, does the batch fail? Partially apply? `{ok: true}` can't say. And it's not idempotent-safe by declaration even though it's idempotent in effect.

Revised:

```
GET  /api/messages?cursor=<opaque>&limit=50
     → 200 { "items": [ {id, sender_id, body, sent_at, status: "read"} ],
             "next_cursor": "eyJ..." | null }
POST /api/messages/read-receipts        body: { "message_ids": ["msg_1","msg_2"] }
     → 200 { "results": [ {"id":"msg_1","status":"ok"},
                          {"id":"msg_2","status":"error",
                           "error":{"code":"message_not_owned","retryable":false}} ] }
```

The receipt is now a resource (creatable, later listable), partial failure is expressible, and the lifecycle field can grow.

## Phase 4 — Changing an EXISTING API (compatibility gate)

Before merging any change to a shipped endpoint, classify every diff line against this checklist:

**Additive-safe (ship freely):**
- [ ] New optional request field with a default preserving old behavior
- [ ] New response field (clients must be tolerant readers — see below)
- [ ] New endpoint, new enum value in a field you *documented as open* ("clients must handle unknown values")
- [ ] Loosening validation (accepting more than before)

**Breaking (requires version or migration plan — no exceptions):**
- [ ] Removing/renaming any field or endpoint, or changing a field's type (`"42"` → `42` breaks typed clients)
- [ ] Making an optional field required, tightening validation, changing a default
- [ ] New enum value in a *closed* enum clients switch exhaustively on
- [ ] Changing error `code` strings, status codes, or pagination token format semantics
- [ ] Reordering meaning: same field, new units/timezone/null-semantics

**Tolerant-reader rules to state in your API docs (they buy you the additive column):**
ignore unknown response fields; treat enums as open unless marked closed; never depend on field order or absence-vs-null distinction unless documented.

## Output contract

Deliver, in order: (1) resource model + rejected alternative with reason, (2) error contract table for every endpoint, (3) endpoint specs with success AND failure bodies, (4) decisions made from the decision table with one-line rationale each, (5) if modifying an existing API, the compatibility classification of every change. Only then: implementation.

## Anti-patterns (hard no)

- Writing a handler, route, or OpenAPI file before the Phase 1/2 gates have written artifacts.
- Designing only 200 responses; "we'll add error handling later" — the errors ARE the contract.
- Booleans for anything with a plausible third state (`read`, `active`, `verified` → lifecycle enums).
- `SELECT * → JSON`: exposing column names, internal ids, or nullable-because-legacy fields as API shape.
- Bumping the version as the first response to any change request. Version bumps are for planned breakage with a migration path, not for avoiding five minutes of additive design.
- Batch endpoints whose partial-failure behavior is undocumented ("returns 200" tells the client nothing about item 37).
