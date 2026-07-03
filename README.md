# Claude Code Skills — Gate Pattern Collection

**15 production-grade skills that give Claude Code the discipline of a senior engineer.** Free, MIT-licensed, no strings.

Most "skills" you'll find are advice dressed up as instructions: *prefer simple designs, be careful with migrations, write good tests.* Current models read that, agree, and do what they were going to do anyway — because advice has no failure condition. There's nothing to check.

These are different. Every skill here is built on **gates**: observable preconditions the model must satisfy before it may proceed. "You may not write the fix until you've pasted the failing command's output." "Produce 2-3 genuinely different structures and argue against your own favorite before implementing." Compliance is checkable, not a matter of judgment — which is why gate-style skills behave the same on a cheap model as an expensive one, and survive the model swaps everyone's dealing with right now.

Read **[docs/GATE-PATTERN.md](docs/GATE-PATTERN.md)** for the full method and how to write your own.

## The skills

**Debugging & quality**
| Skill | Gate |
|---|---|
| `bug-hunt` | No fix until a one-sentence root cause with observed file:line evidence — includes bisect strategy, an instrumentation cookbook, and a heisenbug playbook |
| `test-first` | Runs the failing test and shows you the red *before* implementing — per-ecosystem prove-the-red commands, async-without-sleeps |
| `refactor-safe` | Pins behavior with characterization tests first; the same tests must pass unmodified after |
| `perf-pass` | No optimization without a measurement, no merge without before/after — profiler matrix + "usual suspects" table |
| `dead-code-sweep` | Deletes only with call-site evidence — detector matrix + dynamic-escape-hatch checklist |

**Design & review**
| Skill | Gate |
|---|---|
| `api-design` | No handler code until the resource model, error contract, and one alternative are written down |
| `error-handling` | Every failure source classified into a taxonomy, each class one handling policy — before writing catch blocks |
| `security-sweep` | Map entry points and trust boundaries first; every finding needs a concrete attack path, not a category name |
| `pr-narrative` | Branch diff → reviewer-ready description with a risk rubric and a suggested review path |
| `commit-craft` | Messy tree → atomic commit series, each building on its own; intent-first messages |

**Workflow & the model itself**
| Skill | Gate |
|---|---|
| `agent-handoff` | An orchestrator→executor artifact isn't done without exact paths, per-step acceptance commands, and explicit non-goals |
| `context-econ` | A written context plan before large tasks; state distilled to disk at every phase boundary |
| `dependency-audit` | A package must be proven to exist on the registry with exact spelling *before* any install runs (catches LLM-hallucinated typosquats) |
| `release-notes` | Git history → changelog grouped by audience, breaking changes first, with a sanity gate |
| `onboard-tour` | Generates a "first hour in this codebase" guide — traced core flow, landmine detection, honest setup |

## Install (30 seconds)

```bash
mkdir -p ~/.claude/skills
cp -r skills/* ~/.claude/skills/
```

Per-project instead: `cp -r skills/* your-repo/.claude/skills/`. They trigger automatically when a request matches ("why is this test flaky" → `bug-hunt`) or explicitly via `/bug-hunt`, `/api-design`, etc. Works with any model tier.

## Want the advanced set?

The **[Claude Code Power Pack](https://teasipper06.gumroad.com/l/claude-code-power-pack)** adds the skills that need the most depth — architecture-taste gates (`design-review`), migration safety (`migration-guard`), project-convention enforcement (`house-rules`), the complete **Gate Pattern Playbook**, and tested safety hooks that block secrets from being written to disk and force-pushes to protected branches. More than twice the depth of this repo.

## Contributing & license

MIT — use anywhere, including commercial work. Found a gate that leaks, or have one that transfers well between models? Open an issue or PR. If these save you time, a ⭐ helps others find them.
