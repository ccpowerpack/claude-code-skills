---
name: dead-code-sweep
description: Find and safely remove dead code with call-site evidence — unused functions, unreachable branches, orphaned files, stale feature flags. Use when the user asks to clean up, find unused code, or reduce codebase size.
---

# Dead Code Sweep

Deleting code is the highest-leverage refactor — but only a deletion you can prove safe. The standard of proof: **evidence of zero references, not absence of obvious ones.** Static tools give you candidates; only the dynamic-escape-hatch checks turn a candidate into a verdict.

## Phase 1 — Candidate discovery

Run multiple detectors; each finds what the others miss. Every static tool has blind spots — treat their output as leads, never verdicts.

| Language | Tool + command | What it misses |
|---|---|---|
| TypeScript | `npx knip` (unused files, exports, deps) | Symbols reached via string-keyed dispatch; anything in `knip.json` ignore lists — read the config before trusting output |
| TypeScript | `npx ts-prune \| grep -v '(used in module)'` | Type-only usage in some configs; re-exports through barrels can hide real consumers |
| Python | `vulture src/ --min-confidence 80` | `getattr`/`globals()` dispatch, Django/pytest convention magic, celery task names; below 80 confidence it drowns you in decorators |
| Python | `coverage run -m pytest && coverage report --show-missing` | Coverage proves "tests don't reach it", not "production doesn't" — corroborating signal only |
| Go | `go run golang.org/x/tools/cmd/deadcode@latest ./...` | Code reached via reflection (`reflect.Value.Call`), plugin loading, build tags excluding files from analysis |
| Go | `staticcheck ./...` (U1000) | Exported identifiers — it assumes external packages may use them |
| Rust | `cargo +nightly udeps` (unused deps); `cargo build` warnings (`dead_code` lint) | `#[allow(dead_code)]` suppressions, `pub` items in library crates, macro-generated references |
| Ruby | `bundle exec debride --rails .` | `send`/`public_send` dispatch, Rails callbacks/validations named as symbols, metaprogrammed methods |

Exclude generated and vendored code from BOTH sides of the evidence: as candidates (regenerated code isn't yours to delete) and as references (a hit in `vendor/`, `node_modules/`, or a generated client does not exonerate a candidate — the generator will re-emit it either way). Set up `--glob '!vendor/**' --glob '!**/generated/**'` once and use it in every search below.

Plus tool-agnostic sweeps:

- Grep for exported symbols with no importers: `grep -rn "SymbolName" --include='*.ts' . | grep -v "src/module/file.ts"` — zero hits outside the definition file is a lead.
- Symbols referenced ONLY by their own tests: production refs 0, test refs >0 — the tests are life support, not usage. Candidate includes the tests.
- Orphaned files: no inbound imports and not an entry point (check bundler entries, `package.json` `main`/`exports`/`bin`, `setup.py`/`pyproject.toml` entry points, `Dockerfile` CMD).
- Stale feature flags: flags 100% on/off in every environment's config (see the flag workflow below).
- `git log -1 --format='%ci %h %s' -- <file>` on candidates — long-untouched strengthens the case, but is never proof by itself.

## Phase 2 — Prove each candidate dead (gate)

Static references aren't the whole story. For each candidate, run every applicable escape-hatch check and record the exact patterns you searched. `rg` shown; substitute `grep -rn` if unavailable.

**Dynamic escape-hatch checklist:**

- [ ] **String dispatch / reflection**: `rg '"symbol_name"|\x27symbol_name\x27' --type-add 'all:*' -t all` — the symbol as a quoted string anywhere. Also partial: `rg 'symbol_na' -l` catches string concatenation building the name.
- [ ] **getattr / send-style access**: `rg 'getattr\(|globals\(\)\[|__import__|importlib' <suspect dirs>` (Python); `rg '\.send\(|public_send|const_get|constantize' ` (Ruby); `rg 'reflect\.' ` (Go).
- [ ] **DI containers / registries**: `rg 'register|provide|bind|container' -l | head`, then check those files for the symbol. Spring: `rg '@Component|@Bean'` + XML configs. Anything registered by class name string is invisible to import analysis.
- [ ] **Route tables**: `rg 'symbol_name' $(rg -l 'router\.|urlpatterns|routes\.|@app\.route|Get\(|Post\(')` — handlers referenced in route registration, sometimes by string.
- [ ] **Serializers / codecs**: `rg 'symbol_name' $(rg -l 'serializer|marshal|json:|yaml:|protobuf')` — field names and type registries survive as strings and struct tags.
- [ ] **Templates**: `rg 'symbol_name' --glob '*.html' --glob '*.erb' --glob '*.jinja*' --glob '*.tmpl' --glob '*.vue'` — template engines resolve names at render time.
- [ ] **Config / CI / scripts**: `rg 'symbol_name' --glob '*.yml' --glob '*.yaml' --glob '*.json' --glob '*.toml' --glob 'Makefile' --glob '*.sh'` — cron entries, CI jobs, ops scripts.
- [ ] **Public API surface**: is this exported from a published package? Check `package.json` `exports`, `__all__`, `pub` in a lib crate, Go exported identifiers in a module others import. "Unused internally" ≠ dead. Verdict becomes ASK or a deprecation cycle, not DELETE.
- [ ] **Convention-based loading**: Rails autoload paths, pytest `conftest.py` fixtures (used by parameter name!), Django signals/receivers, framework lifecycle hooks.

Record for each candidate:

```
CANDIDATE: <symbol/file>
STATIC REFS: 0 (searched: <exact patterns/commands>)
DYNAMIC RISK: <none | detail>
VERDICT: DELETE | KEEP (<reason>) | ASK (<what only the team knows>)
```

Verdict semantics — be strict about the boundary:

- **DELETE** requires every applicable checklist item run and clean, with the searched patterns recorded. "I grepped the name" is one item, not the protocol.
- **KEEP** is for candidates the evidence exonerates: a hit in a template, a registry entry, a published export. Record the exonerating reference so the next sweep doesn't re-litigate it.
- **ASK** is for questions the repo cannot answer: "is any customer still on API v1?", "does the data-science team import this module out-of-repo?", "is this kept deliberately for the audited DR runbook?". Formulate the ASK as the exact question plus who can answer it — an ASK without a named audience is a stall.

Two compressed contrast records, to calibrate against the DELETE below:

```
CANDIDATE: src/api/handlers/export_v1.py::ExportV1Handler
STATIC REFS: 0 in src/; 1 in config: routes.yml:88 ("/v1/export": export_v1)
DYNAMIC RISK: live route — access logs needed to know if traffic exists
VERDICT: ASK (does /v1/export still receive traffic? → whoever owns the API gateway dashboards)
```

```
CANDIDATE: src/notifications/templates/renewal_reminder.py::build_context
STATIC REFS: 0 in *.py; 2 in templates/renewal_reminder.html (via {{ context.* }})
DYNAMIC RISK: template engine resolves at render time
VERDICT: KEEP (consumed by renewal_reminder.html:14,29 — exonerated)
```

## Worked example — one candidate, full protocol

Candidate from `vulture`: `src/billing/legacy_proration.py::compute_proration_v1` (confidence 90).

```
$ rg 'compute_proration_v1' -l
src/billing/legacy_proration.py          # definition only
$ rg '"compute_proration_v1"|\x27compute_proration_v1\x27'
(no results)                              # no string dispatch
$ rg 'compute_proration' --glob '*.yml' --glob '*.json' --glob '*.toml'
(no results)                              # no config/CI refs
$ rg 'proration' src/billing/tasks.py src/api/routes.py
src/billing/tasks.py:41: from billing.proration import compute_proration_v2
$ rg 'getattr' src/billing/
src/billing/dispatch.py:17: fn = getattr(handlers, f"compute_{kind}", None)
```

That `getattr` is the scary one: it builds `compute_<kind>` dynamically. Chase it:

```
$ rg 'kind\s*=' src/billing/dispatch.py
src/billing/dispatch.py:12: kind = PRORATION_KINDS[event.type]
$ rg 'PRORATION_KINDS' -A4 src/billing/dispatch.py
src/billing/dispatch.py:6: PRORATION_KINDS = {"upgrade": "proration_v2", "downgrade": "proration_v2"}
```

Both values resolve to `compute_proration_v2`. `git log -1 --format='%ci %s' -- src/billing/legacy_proration.py` → `2024-11-03 keep v1 for rollback window`. The rollback window is long past.

```
CANDIDATE: src/billing/legacy_proration.py::compute_proration_v1
STATIC REFS: 0 (searched: rg symbol, quoted-string, config globs, route/task files)
DYNAMIC RISK: none — getattr dispatch in dispatch.py enumerated; all kinds map to v2
VERDICT: DELETE (plus tests/billing/test_legacy_proration.py, which only tests v1)
```

## Feature-flag retirement sub-workflow

Stale flags are dead code with extra steps — the flag check, the dead branch, and the flag definition all go.

1. **Find fully-rolled flags**: dump flag states across ALL environments (`rg '<flag_name>' config/ deploy/ .env*`; or the flag service's API/export). A flag is retirable only if it has one value in every environment including prod, and no scheduled experiment references it.
   Search every casing the flag lives under — `new_checkout` in code is often `NEW_CHECKOUT` in env files and `new-checkout` in the flag dashboard: `rg -i 'new[_-]?checkout'`.
2. **Confirm age**: `git log -S '<flag_name>' --format='%ci %s' | tail -3` — when was it introduced, when did it last change? A flag at 100% for months is retirable; at 100% since last Tuesday is not.
3. **Kill the dead branch first, keep the live one inline**: replace `if flags.enabled("new_checkout"): new() else: old()` with the winning call, delete the losing function via the Phase 2 protocol (it may have other callers!).
4. **Remove the flag definition** and its config entries in every environment file.
5. **One commit per flag**: branch removal + dead function + flag definition + its tests. Commit message names the flag and its final state ("retire new_checkout, 100% on since 2026-02").

### Escalation for unresolvable ASKs — the tombstone

When an ASK has no one who can answer (orphaned area, departed owner), don't delete on faith and don't keep on fear. Instrument instead: add a one-line log/metric at the candidate's entry (`log.warning("TOMBSTONE: export_v1 called — see TICKET-123")`), ship it, and set a calendar date one full business cycle out (include month-end/quarter-end if the domain has them). Zero hits by the date converts ASK → DELETE with production evidence — the strongest evidence there is. A tombstone without a scheduled follow-up is just new dead code; file the ticket in the same commit.

## Phase 3 — Delete in reviewable batches

- One commit per logical group (a flag + its branches; a module + its tests).
- Delete the tests OF dead code along with it; never delete tests of live code to make something "unused".
- Commit message carries the evidence, not just the act: `remove compute_proration_v1 (dead since PRORATION_KINDS moved to v2, 2024-11)` — the reviewer should not have to redo your Phase 2.
- After each batch: full build + test suite + linter. A batch that breaks anything gets reverted, not patched around.
- Re-run the Phase 1 detectors after the sweep — deletions often orphan a second layer (helpers only the dead code called). One follow-up pass, then stop; don't loop forever.

## Anti-patterns (hard no)

- Commenting out instead of deleting — git remembers; that's what it's for.
- Deleting anything with a `DYNAMIC RISK` note without resolving it.
- Trusting a single detector's output as a verdict — every tool in the matrix has a documented blind spot.
- "While I'm here" refactors mixed into deletion commits — deletions must stay trivially reviewable.
- Retiring a feature flag by deleting only the `if` — leaving the losing branch's function alive as new dead code.

## Output contract

Summary table: what was deleted (LOC), what was kept and why, what needs a human decision (the ASK list with the specific question for each). Plus the verification evidence: build/test/linter output after the final batch, and the evidence record for every DELETE.
