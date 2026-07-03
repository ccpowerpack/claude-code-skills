# The Gate Pattern

*Why "be careful" does nothing, and what to write instead.*

## The problem with advice

Every team that adopts coding agents eventually writes guidance: a CLAUDE.md, a conventions doc, a "please always X" section. And every team makes the same discovery — the model reads it, agrees with it, and then does what it was going to do anyway.

This isn't disobedience. Advice like *"prefer simple designs"* or *"be careful with migrations"* has no failure condition. The model can believe it complied — there is nothing to check. Careful compared to what? Simple by whose measure? Under ambiguity, the model's prior wins, and the model's prior is exactly what you were trying to override.

The 2026 version of this problem is sharper, because the models are better. Current agents execute well. What they still get wrong is **judgment**: the first structurally-plausible design gets built; abstractions appear because they *could* exist; locally-reasonable choices compound into a globally bad shape; the project's own documented conventions get read and then ignored. You can't fix a judgment problem by adding more judgment-shaped words ("be thoughtful!"). You fix it by converting judgment into something checkable.

## What a gate is

A gate is an **observable precondition**: evidence that must exist before the workflow may proceed.

| Advice (degrades) | Gate (survives) |
|---|---|
| Reproduce bugs before fixing them | You may not write a fix until you have pasted the failing command's actual output |
| Consider alternatives before building | Produce 2-3 genuinely different structures and write the strongest honest case AGAINST your preferred one |
| Follow the project conventions | Extract the constraints that bind THIS task, cite file+section for each, and walk the list against the final diff |
| Write tests | Run the new test BEFORE implementing and paste the failure — a test you never saw fail proves nothing |
| Be careful with schema changes | No verdict on a migration until each statement is checked against the lock table for the target database and version |

Three properties make the right column work:

1. **Compliance is observable.** Either the failing output was pasted or it wasn't. No judgment call, no vibes.
2. **The gate blocks, it doesn't suggest.** "You may not proceed until" creates a state the model must exit by producing evidence — the path of least resistance now runs *through* the discipline instead of around it.
3. **It's model-independent.** A weaker model can't fake its way past an evidence requirement any more than a stronger one can. This is why gate-style skills survive model swaps and downgrades — the judgment is banked in the artifact, not in the model reading it.

That last property matters more every month. Strong models come and go from subscription tiers; teams run orchestrator/executor splits where a cheap model does the typing. Advice inherits the reading model's judgment. Gates carry their own.

## Anatomy of a gate-style skill

Every skill in this collection follows the same skeleton:

```
1. Frontmatter — name + a description that says WHEN it triggers
2. One paragraph — the failure mode this skill exists to prevent
3. Phased workflow — each phase has a concrete deliverable;
   the load-bearing phases are gates: "(gate: do not proceed without X)"
4. Reference density — the tables/checklists/matrices the model
   (and you) would otherwise have to reconstruct from memory
5. Output contract — what "done" must contain
6. Anti-patterns — hard-no behaviors, specific enough to adjudicate
```

The **reference density** layer is not decoration. A gate says *"check each statement against lock behavior"* — the skill must then supply the lock-behavior table, or the model will hallucinate one. Gates create demands for facts; a good skill pre-pays them.

The **anti-patterns** section is the second half of every gate. Gates say what must happen; anti-patterns say what must not, phrased concretely enough that a violation is identifiable in a diff: not "don't be sloppy" but *"no catch block may log-and-rethrow without adding context"*.

## Writing your own

The recipe, distilled from every skill here:

1. **Catch the model skipping a step.** Not hypothetically — watch a real session go wrong. The step it skipped is your gate candidate.
2. **Ask: what evidence would prove the step happened?** A pasted command output, a written statement with a file:line citation, a filled table. If you can't name the evidence, you have advice, not a gate — keep digging.
3. **Write the block, not the suggestion.** "You may not X until Y" — and make Y the evidence from step 2.
4. **Pre-pay the facts.** Whatever the gate forces the model to check, supply the reference table it needs to check it correctly.
5. **Test the skill by watching where it leaks.** Run it on a real task with a *cheaper* model than you normally use. Wherever the cheap model slides past the discipline, the gate's evidence requirement is too soft. Tighten and repeat. When it holds on the cheap model, it will hold on everything.

Step 5 is the whole trick, and it's the same trick as TDD: you haven't written a gate until you've seen it stop something.

## What's in this repo

Free, MIT-licensed, no strings: production-grade gate-style skills covering debugging, testing, performance, code hygiene, security review, dependency vetting, API design, agent handoffs, and more. Install:

```bash
mkdir -p ~/.claude/skills
cp -r skills/* ~/.claude/skills/
```

The paid [Power Pack](https://teasipper06.gumroad.com/l/claude-code-power-pack) adds the advanced set — architecture-taste gates, migration safety, project-convention enforcement, the full Gate Pattern Playbook, and tested safety hooks — at more than twice this repo's depth.
