---
name: onboard-tour
description: Generate a "first hour in this codebase" guided tour for a new engineer — entry points, core data flow, where things live, local setup, and landmines. Use when the user asks to explain the codebase, onboard someone, "how does this repo work", or create orientation docs.
---

# Onboard Tour

Write the document you wish you'd been handed on day one: not a file inventory, but a **mental model** — how a request/event flows through the system, and where the bodies are buried. Every claim in the tour must be backed by a `file:line` or a command the reader can re-run; unverifiable folklore is worse than nothing because it decays silently.

## Investigate before writing — the archaeology cookbook

Run these in parallel. For each: the command, and what to look for in the output.

**Entry points — how does execution enter?**

```
rg -l 'func main|if __name__|createServer|listen\(|@app\.route|urlpatterns|router\.' --max-count 1
cat package.json | python3 -c 'import json,sys; d=json.load(sys.stdin); print(d.get("main"), d.get("bin"), json.dumps(d.get("scripts",{}),indent=2))'
rg -n 'ENTRYPOINT|CMD' Dockerfile* ; rg -n 'command:' docker-compose*.yml k8s/ deploy/ 2>/dev/null
rg -ln 'cron|schedule|@periodic|celery|sidekiq|consumer|subscribe' src/ app/ 2>/dev/null
```

Look for: more entry points than the README admits. A "web app" with three queue consumers and a cron binary has four front doors; the tour must name all of them even if it only traces one.

**Churn hotspots — where does work actually happen?**

```
git log --since='12 months ago' --format= --name-only | sort | uniq -c | sort -rn | head -20
```

Look for: the top 5 non-generated files. High churn = either the system's living core (trace your flow through it) or its wound (landmine). Cross-check with the landmine table below to decide which. Ignore lockfiles, snapshots, generated code in this ranking.

**Ownership — who knows what?**

```
cat CODEOWNERS .github/CODEOWNERS 2>/dev/null
git shortlog -sn --since='6 months ago' -- src/billing/ | head -5    # per-area, repeat for each top dir
git log -1 --format='%an %ci' -- <hot-file>                          # freshest expert on a specific file
```

Look for: areas where the top committer has left (name absent from recent global `git shortlog -sn --since='3 months ago'`) — those go in "Where to ask" as orphaned, and their landmines get extra weight.

**Recent-convention discovery — what does "done right" look like NOW?**

```
git log --oneline --diff-filter=A --since='4 months ago' -- 'src/**' | head   # recently ADDED files
git show --stat <commit-of-a-recent-feature>                                  # the shape of one whole feature
```

Look for: the newest merged feature that touched 5-15 files. That commit's file set IS the house pattern — handler + service + model + test + migration, whatever the shape is. Old code shows history; the newest feature shows current convention. Point at it by PR/commit hash in the tour.

**Setup reality check**

```
rg -n '^\s*(run|steps|script)' .github/workflows/*.yml ci/ 2>/dev/null | head -30
cat Makefile justfile 2>/dev/null | rg -n '^[a-zA-Z_-]+:'
```

Look for: disagreement between README setup and what CI actually runs. CI is the ground truth — it passes daily; the README was last verified whenever `git log -1 --format=%ci -- README.md` says.

## Landmine detection

Grep for the confessions, then interpret with the table:

```
rg -n 'HACK|FIXME|XXX|do not|don.t touch|careful|temporary|workaround' --stats -g '!*test*'
```

| Signal | Command | Interpretation |
|---|---|---|
| One file with 3x the churn of the next | churn command above | Either the domain core (fine — say so) or a design wound absorbing every change. Read 2-3 of its recent commit messages: repeated "fix", "again", "actually" → wound |
| High TODO/HACK density in one module | `rg -c 'TODO\|HACK\|FIXME' -g '!*test*' \| sort -t: -k2 -rn \| head` | Team knows it's bad and keeps deferring — flag as "known debt, don't refactor casually, there's a reason it's still here" |
| File touched by many authors | `git shortlog -sn -- <file> \| wc -l` vs. module average | >8 authors on one file = no owner, accidental dumping ground; changes here need broad review |
| Hot file whose tests are cold | compare churn rank of `src/x.py` vs `tests/test_x.py` | Code evolving without test evolution — treat its test suite's green as weak evidence |
| Stale docs | `git log -1 --format=%ci -- docs/ README.md` vs. src | Docs 6+ months behind a hot codebase: mark specific stale claims in the tour, don't just say "docs are old" |
| Commits with fear in the message | `git log --oneline -i --grep='revert\|hotfix\|urgent\|emergency' --since='6 months ago' -- <area>` | Clusters mark fragile areas — name them in Landmines with the commit hashes as evidence |
| `skip`/`xfail`/`.skip(` in tests | `rg -n 'skip\|xfail\|@Disabled\|it\.skip' tests/` | Skipped tests are landmines wearing test coats: something is broken and tolerated |

## Write `TOUR.md`

```markdown
# First hour in <repo>

## What this system does
<2-3 sentences, plain language, no internal jargon.>

## The one flow to understand first
<The traced core flow as a numbered walk: each step = what happens +
 file:line to read. 6-10 steps. This is the spine of the doc.>

## The map
<Directory table: only directories a newcomer will touch, each with a
 one-line "what lives here and when you'd go there".>

## Core data model
<Entities + relationships in a compact list or ASCII sketch.>

## Running it locally
<The commands that actually work, in order, with expected output.
 Flag any step you could not verify.>

## How to add a feature (the house pattern)
<Point at a recent, well-shaped feature (PR/commit hash) as the template to copy.>

## Landmines
<The honest list: fragile areas, misleading names, "don't touch X
 without talking to Y", stale docs. Each with evidence (file/commit).>

## Where to ask
<CODEOWNERS, team docs, channels — if discoverable.>
```

### Worked example — "the one flow" for a realistic app

Fictional but structurally typical: `parcelo`, a Django + Celery shipment-tracking SaaS. The most representative operation is "carrier webhook updates a shipment's status":

```markdown
## The one flow to understand first: a carrier status webhook

1. POST /webhooks/carrier/<slug>/ lands at `api/urls.py:38`, routed to
   `api/views/webhooks.py:52` (`CarrierWebhookView.post`).
2. Signature check: `api/views/webhooks.py:61` calls
   `carriers/auth.py:19` (`verify_hmac`) — per-carrier secrets from the
   `CarrierAccount` model, NOT env vars. First surprise.
3. Raw payload is persisted before any parsing: `WebhookEvent` row at
   `api/views/webhooks.py:74`. The view returns 202 here — everything
   after this point is async.
4. `tasks/ingest.py:31` (`process_webhook_event`) picked up by Celery;
   dispatches on carrier slug via the registry in
   `carriers/registry.py:12` — adding a carrier means registering here.
5. The carrier adapter normalizes the payload:
   `carriers/adapters/fedex.py:88` (`to_status_update`) maps ~40 carrier
   codes to our 6 canonical statuses (`shipments/status.py:9`).
6. `shipments/services.py:117` (`apply_status_update`) is the ONLY
   write path to `Shipment.status` — it enforces the transition table at
   `shipments/status.py:24`. Never write status directly; the model
   save() will not stop you, the service is the guard.
7. A status change emits `shipment_status_changed`
   (`shipments/signals.py:14`); the notifications app listens at
   `notifications/handlers.py:27` and enqueues customer email/SMS.
8. Idempotency: step 4 checks `WebhookEvent.processed_at`
   (`tasks/ingest.py:35`) — carriers redeliver aggressively; this check
   is why replays don't double-notify customers.
```

That is the target texture: every step names a file:line, and steps 2, 6, and 8 carry the "surprise" a newcomer would otherwise learn the hard way. Numbers-only narration ("then the service layer processes it") is inventory, not a tour — rewrite until each step says where AND what's non-obvious.

## Keep it honest — verification pass (gate)

Before delivering, split every executable claim in the tour into three bins and act:

1. **Safe to run — run it.** Read-only and local-only steps: `make deps`, `docker compose up db`, `make test-unit`, lint, the dev server against a local DB. Run them; paste the tail of real output into "Running it locally". If a step fails, the tour documents the failure and the fix, not the README's fiction.
2. **Unsafe or unavailable — mark it.** Anything needing credentials you don't have, external services, seeded data, or that mutates shared state. Tag inline: `⚠ unverified — requires staging AWS creds`. An unverified step marked honest is fine; an unverified step presented as fact is the one thing that destroys the tour's credibility.
3. **file:line claims — spot-check before shipping.** Re-open every file:line referenced in "the one flow" and confirm the symbol is still there (`rg -n 'apply_status_update' shipments/services.py`). Line numbers rot fastest; where the repo is hot, anchor to symbol names ("`services.py`, `apply_status_update`") instead of bare line numbers.

You may not deliver a TOUR.md containing a "Running it locally" section with zero actually-executed commands. If nothing at all is safe to run, say exactly that at the top of the section.

## Quality bar

- Every claim carries a `file:line` (or symbol anchor) or a command a reader can follow — no unverifiable folklore.
- Length: readable in ~15 minutes. Cut inventory, keep model.
- The landmines section has at least one entry with commit-level evidence, or an explicit "no significant landmines found" — an empty section reads as "didn't look".
- If the repo already has onboarding docs, diff reality against them and update/extend rather than duplicating — and list the stale claims you found in the old doc.

## Anti-patterns (hard no)

- A directory listing with one-word descriptions posing as "The map".
- Tracing the core flow from memory of similar frameworks instead of this repo's actual files.
- Copying README setup steps into "Running it locally" without executing the safe ones.
- Landmines without evidence ("the billing code is scary") — every landmine cites a file, commit, or command output.
- Presenting the oldest, biggest module as the pattern to copy — recent code defines convention, not incumbent code.
