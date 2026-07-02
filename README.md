# Claude Code Skills — Free Pack

Two production-grade skills that give Claude Code hard process gates. Free, MIT-licensed, no strings.

## What's here

- **`bug-hunt`** — a root-cause debugging loop. Claude may not write a fix until it states a one-sentence root cause with observed file:line evidence. No reproduction → no fix.
- **`release-notes`** — git history → changelog grouped by audience (users / operators / developers), breaking changes always first, with a sanity gate against dropped migrations and major bumps.

## Install

```bash
mkdir -p ~/.claude/skills
cp -r bug-hunt release-notes ~/.claude/skills/
```

They trigger automatically ("why is this test flaky" → bug-hunt) or via `/bug-hunt`, `/release-notes`.

## The pattern behind these

Prompts that say "be careful" don't change agent behavior. **Gates do**: phases where the model must paste observed evidence (a failing command's output, a root-cause statement with file:line) before it is allowed to proceed. Both skills here are built on that pattern.

## Want the full set?

The **[Claude Code Power Pack](https://teasipper06.gumroad.com/l/claude-code-power-pack)** has 8 skills + 2 tested safety hooks:

`test-first` (TDD with proof-of-red) · `pr-narrative` · `migration-guard` (lock/backfill analysis) · `dead-code-sweep` · `perf-pass` (measurement-first) · `onboard-tour` · plus hooks that block secrets from being written to disk and force-pushes to protected branches.

$24, lifetime updates, 30-day refund → https://teasipper06.gumroad.com/l/claude-code-power-pack
