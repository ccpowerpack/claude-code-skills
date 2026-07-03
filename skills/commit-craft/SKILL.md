---
name: commit-craft
description: Turn a messy working tree into a reviewable series of atomic commits. Use when asked to "clean up these changes", "split this into commits", "make this reviewable", "commit this properly", when a working tree mixes unrelated changes (rename + fix + feature), or before opening a PR from a long hacking session. Refuses to commit until the independent logical changes in the tree have been listed, and requires each commit to build and pass tests on its own.
---

# Commit Craft

The failure mode: three hours of work becomes one commit titled "fix stuff" containing a rename, a bugfix, and half a feature — and the reviewer must untangle what the author already knew. A commit series is a document written for two readers: the reviewer this week and the `git blame`/`bisect` user in two years. Both need atomic commits with intent-stating subjects. The gate: **you may not run `git commit` until the change inventory exists, and no commit may knowingly break the build.**

## Phase 1 — Change inventory (gate)

Run `git status` and `git diff` (plus `git diff --stat` for shape). Then write the inventory:

```
CHANGE INVENTORY
1. Rename `UserStore` → `AccountStore` (12 files, mechanical, zero behavior change)
2. Bugfix: off-by-one in pagination cursor (store.py:214 + regression test)
3. Feature: soft-delete for accounts (new column, new method, 2 tests) — depends on #1
File overlap: store.py contains #1, #2, AND #3 → needs hunk-level splitting
```

Each numbered item = one commit. Note dependencies (they force ordering) and file overlaps (they tell you where `git add -p` is unavoidable). No inventory, no commits.

## Phase 2 — Ordering

Default order, and why reviewers benefit:

1. **Mechanical first** (renames, moves, formatting, codegen): reviewable in 10 seconds *if pure* — reviewer checks "no logic hiding here" and moves on. `git diff -M --stat` showing `similarity 100%` is the proof.
2. **Refactors that enable the change** (extract function, widen a signature): behavior-preserving, existing tests prove it.
3. **Bugfixes**: small, with their regression test in the same commit.
4. **Behavior change / feature last**: the reviewer arrives with the mechanical noise already cleared and full attention left for the only diff that needs judgment.

Mixing #1 into #4 is the classic crime: a 40-file diff where 38 files are rename fallout and 2 files hide the logic — the reviewer must diff every file to find out which two.

## Phase 3 — Splitting cookbook

**Split the working tree with `git add -p`:**

```bash
git add -p src/store.py        # y = stage hunk, n = skip, s = split hunk smaller,
                               # e = edit hunk manually (for changes on adjacent lines)
git diff --cached              # VERIFY what you staged is exactly one logical change
git commit                     # then repeat for the next logical change
```

When two logical changes touch the *same line*, `s` can't separate them — use `e` to edit the staged hunk, keeping only one change's lines (delete the `+` lines of the other change; leave context lines alone).

**Unstage a hunk you added by mistake:** `git restore --staged -p src/store.py`.

**Move a change out of an already-made commit (it's in HEAD, shouldn't be):**

```bash
git reset --soft HEAD^         # commit undone, everything staged again
# re-split with git restore --staged -p + git add -p, commit twice
```

**Amend a change into an earlier commit in the series — no interactive rebase needed.** Claude Code cannot drive `git rebase -i`'s editor; use the fixup/autosquash flow, which is fully non-interactive:

```bash
git add -p src/store.py                        # stage just the fix-that-belongs-earlier
git commit --fixup=<sha-of-target-commit>      # subject becomes "fixup! <target subject>"
GIT_SEQUENCE_EDITOR=true git rebase -i --autosquash <sha-of-target>^
# GIT_SEQUENCE_EDITOR=true accepts the generated todo list untouched → non-interactive
```

**Reorder or drop commits non-interactively** — generate the todo list yourself:

```bash
GIT_SEQUENCE_EDITOR='sed -i "s/^pick \(abc1234\)/drop \1/"' git rebase -i main
# or write the full todo: GIT_SEQUENCE_EDITOR='cp /path/to/todo.txt' git rebase -i main
```

**Reword an earlier commit's message:** `git commit --fixup=reword:<sha>` then the same autosquash rebase; or for HEAD just `git commit --amend`.

**Test a partial staging before committing it** — run the suite against *only* what's staged, since the unstaged remainder won't be in the commit:

```bash
git stash push --keep-index -m "unstaged remainder"   # working tree now == the staged change
make test                                             # this is what commit N will actually contain
git stash pop                                         # bring the rest back
```

**Untracked files** don't appear in `git add -p`; list them with `git status --porcelain | grep '^??'` and assign each to an inventory item explicitly — new files silently falling into whichever commit runs `git add .` last is how migrations end up in the rename commit.

**Safety rails:** before any rebase, `git branch backup/pre-split`; after, `git range-diff backup/pre-split...HEAD` to prove content survived; `git rebase --abort` bails out mid-flight. Never rebase commits already pushed to a shared branch.

**When not to split:** a change and the test that proves it are one commit, never two; a fix whose revert must also revert its enabler stays with the enabler. Atomic means "independently buildable and revertable," not "as small as possible" — over-splitting into commits that only make sense together is the same crime in the other direction.

## Phase 4 — Per-commit verification (gate)

Each commit must build and pass tests **on its own** — this is what makes `git bisect` usable and reverts safe. Verify the series, not just the tip:

```bash
git rebase -x 'make test' main         # runs the suite at EVERY commit, non-interactively
# or spot-check one intermediate: git stash && git checkout <sha> && make test && git checkout - && git stash pop
```

Evidence requirement: paste the test output for at least the final state; for intermediates, either paste the `rebase -x` run or explicitly claim the discipline ("each commit compiled: rebase -x 'go build ./...' passed at all 3"). A series where commit 2/3 doesn't compile is one commit wearing a costume.

## Phase 5 — Messages

Subject = the change's **intent**, imperative mood, ≤ 72 chars. The diff already shows *what* changed; the subject and body exist to say *why* and *what for*. Rubric — a subject fails if a reviewer reading it alone can't predict what kind of diff follows:

| Before | After | Why the rewrite |
|---|---|---|
| `fix bug` | `Prevent duplicate charge on webhook redelivery` | Names the user-visible failure being prevented, not the activity of fixing |
| `update store.py` | `Rename UserStore to AccountStore ahead of multi-account support` | Mechanics were the filename; intent is the rename AND its reason |
| `address review comments` | `Validate cursor tokens before decoding to avoid 500 on garbage input` | "Address comments" describes the PR process, invisible in git history two years later |
| `add retry` | `Retry receipt writes with backoff; DB failovers were dropping paid receipts` | States the incident-shaped why; "add retry" would force the blame-reader to reconstruct it |

Body (when the diff doesn't speak for itself): the why, the alternative you rejected, the limitation you accepted. Never a prose restatement of the diff ("Changed X to Y in file Z" — the reader has the diff open in the other pane).

## Worked example — one mixed tree → three commits

Tree state: `store.py` (rename fallout + cursor bugfix + soft-delete method), `api.py` + 10 others (pure rename fallout), `test_store.py` (regression test + feature tests), `migrations/0042_deleted_at.py` (feature).

```bash
git branch backup/pre-split

# Commit 1 — mechanical rename (10 clean files whole, store.py/test_store.py partially)
git add api.py handlers.py ... # the 10 pure-rename files
git add -p store.py            # y on rename hunks; n on the cursor fix and soft-delete hunks
git add -p test_store.py       # y on renamed references only
git diff --cached --stat && git commit -m "Rename UserStore to AccountStore ahead of multi-account support"

# Commit 2 — bugfix + its regression test
git add -p store.py            # y on the cursor fix hunk only
git add -p test_store.py       # y on test_cursor_last_page only
git commit -m "Fix cursor skipping the final page when total is a multiple of page size"

# Commit 3 — feature
git add store.py test_store.py migrations/0042_deleted_at.py
git commit -m "Add soft-delete for accounts via deleted_at column"

# Verify the whole series, then prove nothing was lost
git rebase -x 'python -m pytest -q' main
git range-diff backup/pre-split...HEAD   # content-equal, just resliced
```

## Output contract

Deliver: the change inventory, the final `git log --oneline` of the series, verification evidence (final-state test output + `rebase -x` result or explicit discipline claim for intermediates), and the range-diff/backup-branch name if any rebase happened.

## Anti-patterns (hard no)

- Subjects like `WIP`, `fix`, `fixes`, `update`, `address comments`, `more changes` — all fail the rubric's prediction test.
- Mixing a rename/move/format change with a logic change in one commit. If `git diff -M` can't show clean renames, the reviewer pays.
- Commit bodies that restate the diff instead of the why.
- Committing a series where an intermediate commit is known not to build ("it's fixed two commits later" breaks bisect and revert).
- Running `git commit -a` on a tree the inventory says contains more than one logical change.
- Rebasing without a backup branch, or rebasing commits already pushed to a shared branch.
