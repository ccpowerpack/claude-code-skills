---
name: release-notes
description: Generate release notes / changelog from git history, grouped by audience with breaking changes first. Use when the user asks for release notes, a changelog entry, "what changed since", or is cutting a release/tag.
---

# Release Notes

Release notes are read by three different audiences with three different questions: users ask "what can I do now / what will break", operators ask "what must I do to upgrade", contributors ask "what moved". Serve all three; never make a user read commit messages.

## Gather

1. Determine the range: last tag → HEAD (`git describe --tags --abbrev=0`), or the range the user names.
2. `git log <range> --oneline --no-merges` plus merge commits for PR titles (`git log <range> --merges --format='%s'`) — PR titles are usually better-written than commit subjects.
3. For anything user-facing or breaking, read the actual diff — commit messages lie by omission. Especially check: config file changes (`git diff <range> -- '*.yml' '*.toml' '*example*'`), CLI flag changes, schema migrations (`git diff <range> --stat -- '*migrations*'`), dependency major bumps (`git diff <range> -- package.json go.mod Cargo.toml requirements*.txt`), API signature changes.
4. Check for an existing `CHANGELOG.md` and match its established format exactly (heading style, tense, linking convention). House style beats this template.

## Classify every change

- **Breaking** — anyone must act before/when upgrading. Always first, never buried.
- **Features** — new capabilities, written as what the user can now do.
- **Fixes** — what was broken, from the sufferer's perspective ("dates before 1970 no longer crash export" — not "fix timestamp handling").
- **Operations** — new env vars/flags, migrations to run, resource/infra changes, deprecations with timelines.
- **Internal** — refactors, CI, dev-deps. One line or omitted, per house style.

Drop noise entirely: version bump commits, merge commits, "fix typo", "address review comments".

### Decision table — the cases that get misfiled

| Change in range | Correct classification | Why |
|---|---|---|
| Revert of a change that never shipped in a release | **Drop entirely** | Net effect on a released→released diff is zero; noting it confuses readers |
| Revert of a change that DID ship | Fix or Breaking | The revert is itself a behavior change for users on the last release |
| Dependency major bump (e.g. postgres driver v2→v3) | **Operations**, even if "internal" | Majors change runtime requirements, connection behavior, transitive licenses — operators must know |
| Dependency patch/minor bump | Internal, usually omitted | Unless it fixes a CVE — then Operations with the CVE id |
| Perf improvement users will feel (p99 latency, startup time) | **Features/Fixes** with the number | "Cold start 4.1s → 1.3s" is user-facing; measured impact or it doesn't get a claim |
| Perf refactor with no measured user-visible change | Internal | No measurement → no user-facing claim |
| New feature behind a default-off flag | Features, marked experimental/opt-in | Announce it, but say how to enable and that default behavior is unchanged |
| Removing a default-off experimental flag's feature | Internal or short note | Nobody was on it by default; one line prevents surprise for the few opt-ins |
| Deprecation (still works, warns) | **Operations** with the removal timeline | Deprecations without a stated removal version are just noise |
| Behavior change to undocumented-but-observable behavior | **Breaking** | Hyrum's Law: if users can observe it, someone depends on it. When unsure, err toward Breaking |
| Migration that runs automatically on startup | Operations regardless | Operators need to know about lock time / table rewrites even if "no action needed" |
| Security fix | Fixes or Operations, per house disclosure policy | Check whether the project withholds details until advisory publication — don't front-run it |

## Writing rules

- Every entry: what changed + who cares. Under ~20 words each.
- Breaking changes get a migration snippet: exact before → after (template below).
- Link PR/issue numbers in the house format if the changelog does.
- Never invent user impact you can't verify from the diff — if unclear, mark it `<!-- TODO: confirm impact -->` rather than guessing.

### Breaking-change migration snippet template

```markdown
#### <one-line summary of what breaks>
**Who is affected:** <precise population — "anyone setting X", "all deployments using Y">
**Before / after:**
    <exact old syntax>       →      <exact new syntax>
**To upgrade:** <the command(s) or edit, copy-pasteable>
**If you do nothing:** <the exact failure they will see>
```

Filled example — renamed config key:

```markdown
#### `server.timeout` renamed to `server.request_timeout_seconds`
**Who is affected:** anyone with `server.timeout` in `config.yml` (it was the default-off knob, so most who set it).
**Before / after:**
    server:                       server:
      timeout: 30          →        request_timeout_seconds: 30
**To upgrade:** `sed -i 's/^\(\s*\)timeout:/\1request_timeout_seconds:/' config.yml` — or edit the one key by hand.
**If you do nothing:** startup fails with `unknown config key: server.timeout (renamed in v3.0, see release notes)`.
```

"If you do nothing" is the line most templates skip and the one operators most want — verify it against the actual code path (does it hard-fail, warn, or silently ignore?).

## Audience splitting — same change, two lines

A single change often needs one sentence for users and a different one for operators. Don't average them into one mushy line; write both, in their sections.

Change: export jobs moved from in-process to a worker queue.

- Under **Features**: "Large CSV exports no longer time out — exports over 100k rows now run in the background and email a download link."
- Under **Operations**: "New required worker process: run `bin/worker export` alongside the web process; new env var `EXPORT_QUEUE_URL`. Without a worker, exports queue forever."

Rule of thumb: if the diff touches both behavior and deployment topology (new process, new env var, new port, migration), it appears twice. Once is a bug.

The two lines answer different questions and use different vocabulary: the user line names the visible symptom and benefit in product terms; the operator line names processes, env vars, and the failure mode of inaction. If you can swap the two lines between sections without anyone noticing, you wrote the mushy average — rewrite.

## Worked example — 12 commits → finished notes

Input (`git log v2.3.0..HEAD --oneline --no-merges`):

```
9f3c2a1 feat: add CSV export for audit log
7b21e90 fix: retry webhook delivery on 429 with backoff
3d0a114 chore: bump eslint to 9.x
c88f7e2 feat!: rename server.timeout config to request_timeout_seconds
5a91bc3 fix typo in README
e77d240 perf: cache tenant lookup, p99 /api/v1/events 480ms -> 95ms
1f6b8aa Revert "feat: inline editor beta" (never released)
b30291c chore: bump pg driver 8.x -> 9.x
44e0d17 feat: add --json flag to `acme status`
a1c9f02 db: add index on events(tenant_id, created_at)
0d4e551 address review comments
8c72b3f fix: audit log export includes soft-deleted rows
```

Output:

```markdown
## v3.0.0 — 2026-07-03

### Breaking
- **`server.timeout` renamed to `server.request_timeout_seconds`** (#412) — <full migration snippet as above>

### Features
- Audit log can now be exported as CSV from Settings → Audit (#398)
- `acme status --json` emits machine-readable output for scripting (#405)
- `/api/v1/events` p99 latency reduced 480ms → 95ms via tenant-lookup caching (#401)

### Fixes
- Webhook deliveries hitting rate limits are now retried with backoff instead of being dropped (#396)
- Audit log CSV export no longer includes soft-deleted rows (#417)

### Operations
- Postgres driver upgraded 8.x → 9.x: minimum supported PostgreSQL is now 13; connection-string `ssl` params parse per pg9 rules (#408)
- New migration adds an index on `events(tenant_id, created_at)` — CONCURRENTLY, but budget time on tables >10M rows (#410)
```

Accounting for all 12: 8 became notes, 4 dropped with reasons — typo fix (noise), review-comments commit (noise), eslint bump (dev-dep, Internal omitted per house style), revert of unreleased feature (net zero). The perf commit made Features only because the commit carried measured numbers; the pg bump landed in Operations despite its `chore:` prefix — the classification table overrides the commit-type prefix every time.

## Sanity gate before finishing

- Does every schema migration in the range appear under Operations? (`git diff <range> --stat -- '*migrations*'` — count must match.)
- Does every major-version dependency bump appear somewhere?
- Every commit accounted for: became a note, merged into one, or dropped with a stated reason. Is the note count plausible vs. commit count? (40 commits → 2 notes means you dropped something; 40 commits → 40 notes means you didn't classify.)
- Does every Breaking entry have a filled migration snippet including "if you do nothing"?

## Anti-patterns (hard no)

- Classifying by commit-message prefix alone — `chore:` hiding a driver major bump is the canonical failure.
- Burying a breaking change under Features because it "also adds capability".
- Copying commit subjects verbatim into any section — subjects describe the diff, notes describe the impact.
- Announcing a revert of something that never shipped.
- Perf claims without numbers from the range's commits, benchmarks, or diff.

## Output

Write into `CHANGELOG.md` above the previous release (matching its format), or print in a fenced block if no changelog exists — and offer both a full and a condensed (tweet-length) version when the release is user-facing.

The condensed version is not a compressed list: lead with the single biggest user-visible change, then the breaking-change count with a link to the full notes ("v3.0: audit-log CSV export and 5x faster event queries. One breaking config rename — migration guide: <link>").
