---
name: release-notes
description: Generate release notes / changelog from git history, grouped by audience with breaking changes first. Use when the user asks for release notes, a changelog entry, "what changed since", or is cutting a release/tag.
---

# Release Notes

Release notes are read by three different audiences with three different questions. Serve all three; never make a user read commit messages.

## Gather

1. Determine the range: last tag → HEAD (`git describe --tags --abbrev=0`), or the range the user names.
2. `git log <range> --oneline --no-merges` plus merge commits for PR titles (`--merges`) — PR titles are usually better-written than commit subjects.
3. For anything user-facing or breaking, read the actual diff — commit messages lie by omission. Especially check: config file changes, CLI flag changes, schema migrations, dependency major bumps, API signature changes.
4. Check for an existing `CHANGELOG.md` and match its established format exactly (heading style, tense, linking convention). House style beats this template.

## Classify every change

- **Breaking** — anyone must act before/when upgrading. Always first, never buried.
- **Features** — new capabilities, written as what the user can now do.
- **Fixes** — what was broken, from the sufferer's perspective ("dates before 1970 no longer crash export" — not "fix timestamp handling").
- **Operations** — new env vars/flags, migrations to run, resource/infra changes, deprecations with timelines.
- **Internal** — refactors, CI, dev-deps. One line or omitted, per house style.

Drop noise entirely: version bump commits, merge commits, "fix typo", "address review comments".

## Writing rules

- Every entry: what changed + who cares. Under ~20 words each.
- Breaking changes get a migration snippet: exact before → after.
- Link PR/issue numbers in the house format if the changelog does.
- Never invent user impact you can't verify from the diff — if unclear, mark it `<!-- TODO: confirm impact -->` rather than guessing.

## Sanity gate before finishing

- Does every schema migration in the range appear under Operations?
- Does every major-version dependency bump appear somewhere?
- Is the note count plausible vs. commit count? (40 commits → 2 notes means you dropped something; 40 commits → 40 notes means you didn't classify.)

## Output

Write into `CHANGELOG.md` above the previous release (matching its format), or print in a fenced block if no changelog exists — and offer both a full and a condensed (tweet-length) version when the release is user-facing.
